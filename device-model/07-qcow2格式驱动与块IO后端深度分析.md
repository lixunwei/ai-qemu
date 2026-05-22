# qcow2 格式驱动与块 I/O 后端深度分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中最重要的块驱动实现——qcow2 格式驱动和 file-posix 协议驱动，以及 virtio-blk 设备如何连接到块层。qcow2 是 QEMU 的原生磁盘格式，支持 COW 快照、压缩、加密、稀疏分配等特性。file-posix 是 Linux 上最常用的协议驱动，支持 O_DIRECT、linux-aio 和 io_uring。

**关键源文件：**
- `block/qcow2.c` — qcow2 格式驱动主文件（~6400行）
- `block/qcow2.h` — qcow2 头定义和状态结构（~1000行）
- `block/qcow2-cluster.c` — L1/L2 表和簇分配（~2400行）
- `block/qcow2-refcount.c` — 引用计数管理（~3700行）
- `block/file-posix.c` — POSIX 文件协议驱动（~4100行）
- `block/raw-format.c` — raw 格式透传驱动（~700行）
- `hw/block/virtio-blk.c` — virtio-blk 设备模型（~1600行）
- `blockjob.c` / `job.c` — 块作业框架

---

## 2. qcow2 格式概览

### 2.1 磁盘布局

```
┌──────────────────────────────────────────────┐
│  QCowHeader (第一个簇)                        │
│  magic(0x514649FB) + version(2/3) + 元数据    │
├──────────────────────────────────────────────┤
│  L1 Table (l1_table_offset)                  │
│  L1 条目指向 L2 表                            │
├──────────────────────────────────────────────┤
│  Refcount Table (refcount_table_offset)      │
│  条目指向 refcount block                      │
├──────────────────────────────────────────────┤
│  Snapshot Table (snapshots_offset)           │
├──────────────────────────────────────────────┤
│  L2 Tables (按需分配)                         │
│  L2 条目指向数据簇                            │
├──────────────────────────────────────────────┤
│  Refcount Blocks (按需分配)                   │
├──────────────────────────────────────────────┤
│  Data Clusters (按需分配)                     │
│  实际 Guest 数据存储                          │
└──────────────────────────────────────────────┘
```

### 2.2 QCowHeader 结构

```c
// qcow2.h:155-183
typedef struct QCowHeader {
    uint32_t magic;               // 0x514649FB ("QFI\xfb")
    uint32_t version;             // 2 或 3
    uint64_t backing_file_offset; // backing 文件名偏移
    uint32_t backing_file_size;   // backing 文件名长度
    uint32_t cluster_bits;        // 簇大小 = 1 << cluster_bits（典型 16=64KB）
    uint64_t size;                // 虚拟磁盘大小（字节）
    uint32_t crypt_method;        // 0=无, 1=AES, 2=LUKS
    uint32_t l1_size;             // L1 表条目数
    uint64_t l1_table_offset;     // L1 表偏移
    uint64_t refcount_table_offset; // refcount 表偏移
    uint32_t refcount_table_clusters; // refcount 表大小
    uint32_t nb_snapshots;        // 快照数
    uint64_t snapshots_offset;    // 快照表偏移
    // v3 新增:
    uint64_t incompatible_features;
    uint64_t compatible_features;
    uint64_t autoclear_features;
    uint32_t refcount_order;      // refcount 位数 = 1 << order（典型 4=16位）
    uint32_t header_length;
    uint8_t compression_type;     // 0=zlib, 1=zstd
} QEMU_PACKED QCowHeader;
```

---

## 3. qcow2 两级地址翻译

### 3.1 L1/L2 查表

```
Guest 偏移量: offset
  │
  ├── L1 索引 = offset / (l2_size * cluster_size)
  │     → L1 表条目 → L2 表物理偏移
  │
  └── L2 索引 = (offset / cluster_size) % l2_entries
        → L2 表条目 → 数据簇物理偏移
        
  簇内偏移 = offset % cluster_size

典型参数（cluster_bits=16, 64KB 簇）:
  L2 条目数 = 64KB / 8 = 8192
  L2 覆盖 = 8192 × 64KB = 512MB
  L1 条目覆盖 = l1_size × 512MB
```

### 3.2 qcow2_get_host_offset

```c
// qcow2-cluster.c:586+ — qcow2_get_host_offset()
// 将 Guest 偏移转换为 Host 偏移:
// 1. L1 查找: l1_index → s->l1_table[l1_index]
// 2. 读 L2 表（可能从缓存）
// 3. L2 查找: l2_index → l2_entry
// 4. 判断子簇类型:
//    - QCOW2_SUBCLUSTER_NORMAL: 正常分配
//    - QCOW2_SUBCLUSTER_ZERO_PLAIN: 零填充（未分配）
//    - QCOW2_SUBCLUSTER_ZERO_ALLOC: 零填充（已分配）
//    - QCOW2_SUBCLUSTER_UNALLOCATED_PLAIN: 未分配（需读 backing）
//    - QCOW2_SUBCLUSTER_UNALLOCATED_ALLOC: 部分分配
//    - QCOW2_SUBCLUSTER_COMPRESSED: 压缩簇
// 5. 返回 host_offset = l2_entry_offset + 簇内偏移
```

---

## 4. qcow2 I/O 路径

### 4.1 读路径

```c
// qcow2.c:2472-2530 — qcow2_co_preadv_part()
// 循环处理请求（可能跨多个簇）:
while (bytes != 0) {
    // 1. 加锁查表
    qemu_co_mutex_lock(&s->lock);
    qcow2_get_host_offset(bs, offset, &cur_bytes, &host_offset, &type);
    qemu_co_mutex_unlock(&s->lock);
    
    // 2. 根据类型处理
    switch (type) {
    case ZERO/UNALLOCATED(无backing):
        qemu_iovec_memset(qiov, 0, cur_bytes);    // 填零
        break;
    case UNALLOCATED(有backing):
        // 读 backing 层（递归到 backing BDS）
        break;
    case NORMAL:
        // 读数据簇（通过 bs->file）
        break;
    case COMPRESSED:
        // 解压整个簇
        break;
    }
}
// 支持 AioTaskPool 并行多簇读取
```

### 4.2 写路径

```c
// qcow2.c:2762-2815 — qcow2_co_pwritev_part()
// 循环处理:
// 1. qcow2_alloc_host_offset() — 分配数据簇
//    - COW: 从 backing 复制部分簇数据
//    - 更新 L2 表
//    - 更新 refcount
// 2. 可能加密数据
// 3. 写入数据簇（通过 bs->file）
// 4. 更新 L2 表中的映射
```

### 4.3 簇分配

```c
// qcow2-cluster.c:1605-1743 — do_alloc_cluster_offset()
// 分配新簇:
// 1. 在文件末尾分配空间
// 2. 更新 refcount (+1)
// 3. 如果需要 COW:
//    - 读取 backing/旧簇中不需要覆盖的部分
//    - 写入新簇
// 4. 更新 L2 表条目指向新簇
```

---

## 5. qcow2 引用计数

```c
// qcow2-refcount.c — 引用计数管理
// 每个簇有一个 refcount（典型 16 位）
// refcount=0: 空闲簇
// refcount=1: 正常使用
// refcount>1: 被快照引用（COW 时需要复制）

// refcount 表结构:
// refcount_table[i] → refcount_block 偏移
// refcount_block[j] → 具体簇的 refcount 值

// 关键操作:
// qcow2_update_cluster_refcount() — 更新单个簇的 refcount
// qcow2_free_clusters() — refcount 减 1，可能释放
// qcow2_alloc_clusters() — 寻找 refcount=0 的簇
```

---

## 6. qcow2 高级特性

### 6.1 快照

```c
// bdrv_qcow2 驱动注册: qcow2.c:6323-6327
.bdrv_snapshot_create = qcow2_snapshot_create,
.bdrv_snapshot_goto   = qcow2_snapshot_goto,
.bdrv_snapshot_delete = qcow2_snapshot_delete,
.bdrv_snapshot_list   = qcow2_snapshot_list,

// 快照原理:
// 1. 复制当前 L1 表到快照区域
// 2. 所有被引用的簇 refcount +1
// 3. 后续写入触发 COW（新分配簇）
// 4. 恢复: 用快照的 L1 表替换当前 L1
```

### 6.2 加密

```c
// qcow2.c:1580-1606 — 加密初始化
// 支持 AES-128-CBC (v1) 和 LUKS (v2)
// LUKS: 每扇区独立 IV，AES-XTS 模式
// 加密/解密在读写路径中透明执行
```

### 6.3 压缩

```c
// qcow2.c:1517-1531 — 压缩类型
// 支持 zlib (默认) 和 zstd
// 压缩簇: L2 条目包含压缩偏移和大小
// 只读: 压缩簇不可就地修改，写入时分配新的非压缩簇
```

---

## 7. file-posix 协议驱动

### 7.1 驱动注册

```c
// file-posix.c:3966-3995 — bdrv_file
BlockDriver bdrv_file = {
    .format_name   = "file",
    .protocol_name = "file",
    .bdrv_open     = raw_open,
    .bdrv_close    = raw_close,
    .bdrv_co_preadv  = raw_co_preadv,
    .bdrv_co_pwritev = raw_co_pwritev,
    .bdrv_co_flush_to_disk = raw_co_flush_to_disk,
    // ...
};
```

### 7.2 AIO 模式

```c
// file-posix.c:738-774 — AIO 模式选择
// 三种 AIO 后端:
//
// 1. io_uring (优先):
//    use_linux_io_uring = true
//    高性能异步 I/O，支持 SQE/CQE 批处理
//
// 2. linux-aio:
//    use_linux_aio = true
//    传统 io_submit/io_getevents 接口
//
// 3. 线程池 (默认):
//    在工作线程中执行同步 pread/pwrite
//    兼容性最好但性能较低

// 选择: -drive aio=io_uring / aio=native / aio=threads
```

### 7.3 O_DIRECT

```c
// file-posix.c:128-131, 739-745
// cache=none → O_DIRECT
// 要求:
//   - 读写偏移和大小对齐到 512B（或 4KB）
//   - 数据缓冲区对齐
// 块层自动处理对齐（bounce buffer）
```

### 7.4 文件锁

```c
// file-posix.c:875-980 — 文件锁
// 防止多个 QEMU 实例同时写同一磁盘
// 使用 OFD 锁（Linux）或 flock
// 锁字节偏移在文件特定位置（不影响数据）
```

---

## 8. raw 格式驱动

```c
// raw-format.c:638-680 — bdrv_raw
// 最简单的格式驱动: 直接透传到协议层
// raw_open() :469-535 — 打开底层协议
// 读写: 调整偏移后直接转发到 bs->file
// raw_probe() :537-543 — 返回 1（最低分，兜底格式）
```

---

## 9. virtio-blk 设备

### 9.1 设备与 BlockBackend 连接

```c
// hw/block/virtio-blk.c
// virtio-blk 设备通过 BlockBackend 访问块层
// 设备配置属性: drive=<block-backend-name>
// 请求处理:
//   1. Guest 写 virtqueue → virtio_blk_handle_output() :1044-1059
//   2. 解析 virtio-blk 请求头（类型/扇区号/...）
//   3. 调用 blk_co_preadv/pwritev
//   4. 完成 → 写 virtqueue used ring → 通知 Guest
```

### 9.2 IOThread/Dataplane

```c
// virtio-blk 支持 iothread 模式:
// -device virtio-blk-pci,iothread=iothread0
// 将 I/O 处理从主线程迁移到专用 IOThread
// BlockBackend 的 AioContext 设置为 IOThread 的 AioContext
// 避免主循环 BQL 竞争，提高吞吐
```

---

## 10. 块作业框架

### 10.1 Job 状态机

```c
// job.c:59-72 — JobSTT (状态转换表)
// 状态: CREATED → RUNNING → WAITING → PENDING → CONCLUDED → NULL
// 支持: PAUSED, READY (合入点), STANDBY
//
// job.c:208-220 — job_state_transition()
// 严格按 JobSTT 允许的转换进行
```

### 10.2 块作业类型

```
mirror:  实时镜像（源→目标同步，最终切换）
commit:  合入中间层（删除快照链中间节点）
stream:  流入 backing 数据（消除 backing 依赖）
backup:  全量/增量备份（源→目标复制）
create:  创建镜像（异步 qcow2_co_create）
```

### 10.3 作业与 BDS 图交互

```c
// blockjob.c:87-214
// 作业操作 BDS 图:
//   - 插入/移除过滤器节点
//   - 修改 backing 关系
//   - 在作业完成时调整拓扑
// drained section: 暂停 I/O 以安全修改 BDS 图
```

---

## 11. 小结

| 组件 | 实现 |
|------|------|
| **qcow2 格式** | 6400 行，L1/L2 两级地址翻译，refcount 引用计数，快照/加密/压缩 |
| **qcow2 读** | get_host_offset → 类型判断（normal/zero/unalloc/compressed）→ 读数据或 backing |
| **qcow2 写** | alloc_host_offset → COW → 写数据 → 更新 L2 → 更新 refcount |
| **file-posix** | 4100 行，io_uring/linux-aio/线程池三模式，O_DIRECT，文件锁 |
| **raw 格式** | 700 行，透传到协议层，probe 返回最低分兜底 |
| **virtio-blk** | virtqueue → blk_co_preadv/pwritev，支持 IOThread dataplane |
| **块作业** | mirror/commit/stream/backup，Job 状态机 6 状态严格转换 |
| **格式探测** | qcow2 magic 100 分 vs raw 兜底 1 分 |
