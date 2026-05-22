# 25 - Block Layer I/O 子系统深度分析 — BlockDriverState、协程 I/O、请求追踪与限流

> **基于 QEMU 11.0.50 源码**，深入分析 QEMU Block Layer 完整实现：
> BlockDriverState 节点图、BlockDriver 驱动 vtable、BlockBackend 设备接口、
> BdrvChild 父子关系与权限系统、协程 I/O 读写路径、请求追踪与串行化、
> Copy-on-Read、I/O 限流、Block Job 基础设施与 QMP 接口。

---

## 目录

1. [BlockDriverState：块设备节点核心结构](#1-blockdriverstate块设备节点核心结构)
2. [BlockDriver：驱动 vtable](#2-blockdriver驱动-vtable)
3. [BlockBackend：设备前端接口](#3-blockbackend设备前端接口)
4. [BdrvChild：父子关系与 ChildRole](#4-bdrvchild父子关系与-childrole)
5. [节点图与典型链路](#5-节点图与典型链路)
6. [权限系统](#6-权限系统)
7. [BlockLimits：I/O 约束](#7-blocklimitsio-约束)
8. [bdrv_open：打开路径](#8-bdrv_open打开路径)
9. [I/O 入口：BlockBackend 层](#9-io-入口blockbackend-层)
10. [核心协程 I/O 路径](#10-核心协程-io-路径)
11. [请求追踪与串行化](#11-请求追踪与串行化)
12. [Copy-on-Read](#12-copy-on-read)
13. [写入路径与 Flush](#13-写入路径与-flush)
14. [协程包装器模式](#14-协程包装器模式)
15. [I/O 限流](#15-io-限流)
16. [Drain 机制](#16-drain-机制)
17. [Block Job 基础设施](#17-block-job-基础设施)
18. [数据流全景图](#18-数据流全景图)

---

## 1. BlockDriverState：块设备节点核心结构

```c
// block_int-common.h:1104-1292
struct BlockDriverState {
    /* 打开标志与属性 */
    int open_flags;                        // 打开标志
    bool encrypted;                        // 加密介质
    bool sg;                               // SCSI generic 设备
    bool probed;                           // 格式是探测的而非指定的
    bool force_share;                      // 强制共享权限
    bool implicit;                         // 自动插入的过滤节点

    /* 驱动与后端 */
    BlockDriver *drv;                      // 驱动 vtable（NULL=无介质）
    void *opaque;                          // 驱动私有数据

    /* AIO 上下文 */
    AioContext *aio_context;               // 事件循环
    QLIST_HEAD(, BdrvAioNotifier) aio_notifiers;

    /* 文件名与 backing */
    char filename[PATH_MAX];               // 文件名
    char backing_file[PATH_MAX];           // 镜像头中的 backing 文件
    char auto_backing_file[PATH_MAX];      // 自动解析的 backing 文件名
    char backing_format[16];               // backing 格式
    QDict *full_open_options;              // 完整打开选项
    char exact_filename[PATH_MAX];         // 精确文件名

    /* I/O 限制 */
    BlockLimits bl;                        // I/O 对齐与传输限制

    /* 支持的标志 */
    unsigned int supported_read_flags;     // 读标志
    unsigned int supported_write_flags;    // 写标志
    unsigned int supported_zero_flags;     // 写零标志
    unsigned int supported_truncate_flags; // 截断标志

    /* 节点标识与引用 */
    char node_name[32];                    // 节点名称
    QTAILQ_ENTRY(BlockDriverState) node_list;
    QTAILQ_ENTRY(BlockDriverState) bs_list;
    int refcnt;                            // 引用计数

    /* 操作阻止器 */
    QLIST_HEAD(, BdrvOpBlocker) op_blockers[BLOCK_OP_TYPE_MAX];

    /* 继承关系 */
    BlockDriverState *inherits_from;       // 继承默认选项的父节点

    /* 子节点图 */
    QLIST_HEAD(, BdrvChild) children;      // 所有子节点
    BdrvChild *backing;                    // backing 子节点（COW）
    BdrvChild *file;                       // file 子节点（协议）
    QLIST_HEAD(, BdrvChild) parents;       // 所有父节点

    /* 选项 */
    QDict *options;
    QDict *explicit_options;
    BlockdevDetectZeroesOptions detect_zeroes;

    /* 容量 */
    int64_t total_sectors;                 // 磁盘大小（扇区）

    /* 脏位图 */
    QemuMutex dirty_bitmap_mutex;
    QLIST_HEAD(, BdrvDirtyBitmap) dirty_bitmaps;

    /* I/O 追踪 */
    int copy_on_read;                      // CoR 引用计数（原子）
    unsigned int in_flight;                // 进行中请求数（原子）
    unsigned int serialising_in_flight;    // 串行化请求数

    int enable_write_cache;                // 写缓存
    int quiesce_counter;                   // 静默计数器
    unsigned int write_gen;                // 写入代

    /* 请求追踪 */
    QemuMutex reqs_lock;
    QLIST_HEAD(, BdrvTrackedRequest) tracked_requests;
    CoQueue flush_queue;                   // flush 串行化队列
    bool active_flush_req;                 // 是否有 flush 进行中
    unsigned int flushed_gen;              // 已 flush 的代

    /* 块状态缓存 */
    CoMutex bsc_modify_lock;
    BdrvBlockStatusCache *block_status_cache;
};
```

---

## 2. BlockDriver：驱动 vtable

```c
// block_int-common.h:94-800（关键回调摘录）
struct BlockDriver {
    const char *format_name;               // "qcow2"、"raw" 等
    int instance_size;                     // opaque 大小
    const char *protocol_name;             // 协议名（"file"、"nbd"、"http"）

    /* 生命周期 */
    int (*bdrv_open)(BDS *bs, QDict *options, int flags, Error **errp);
    void (*bdrv_close)(BDS *bs);
    int coroutine_fn (*bdrv_co_create)(BlockdevCreateOptions *, Error **);
    int coroutine_fn (*bdrv_co_create_opts)(BlockDriver *, const char *,
                                            QemuOpts *, Error **);

    /* 读写 I/O — 核心回调 */
    int coroutine_fn (*bdrv_co_preadv_part)(BDS *, int64_t offset,
        int64_t bytes, QEMUIOVector *qiov, size_t qiov_offset, BdrvRequestFlags);
    int coroutine_fn (*bdrv_co_pwritev_part)(BDS *, int64_t offset,
        int64_t bytes, QEMUIOVector *qiov, size_t qiov_offset, BdrvRequestFlags);

    /* 旧版回调（兼容） */
    int coroutine_fn (*bdrv_co_preadv)(BDS *, int64_t, int64_t,
                                       QEMUIOVector *, BdrvRequestFlags);
    int coroutine_fn (*bdrv_co_pwritev)(BDS *, int64_t, int64_t,
                                        QEMUIOVector *, BdrvRequestFlags);

    /* Flush */
    int coroutine_fn (*bdrv_co_flush_to_os)(BDS *);    // 刷到驱动缓存
    int coroutine_fn (*bdrv_co_flush_to_disk)(BDS *);  // 刷到物理磁盘

    /* Discard / Truncate */
    int coroutine_fn (*bdrv_co_pdiscard)(BDS *, int64_t, int64_t);
    int coroutine_fn (*bdrv_co_truncate)(BDS *, int64_t, bool exact,
                                         PreallocMode, BdrvRequestFlags, Error **);

    /* 块状态查询 */
    int coroutine_fn (*bdrv_co_block_status)(BDS *, bool, int64_t, int64_t,
                                              int64_t *, int64_t *, BDS **);

    /* 快照 */
    int (*bdrv_snapshot_create)(BDS *, QEMUSnapshotInfo *);
    int (*bdrv_snapshot_goto)(BDS *, const char *snapshot_id);
    int (*bdrv_snapshot_delete)(BDS *, const char *, const char *, Error **);
    int (*bdrv_snapshot_list)(BDS *, QEMUSnapshotInfo **);

    /* 权限 */
    int (*bdrv_check_perm)(BDS *, uint64_t perm, uint64_t shared, Error **);
    void (*bdrv_set_perm)(BDS *, uint64_t perm, uint64_t shared);
    void (*bdrv_abort_perm_update)(BDS *);
    void (*bdrv_child_perm)(BDS *, BdrvChild *, BdrvChildRole,
                            BlockReopenQueue *, uint64_t, uint64_t,
                            uint64_t *, uint64_t *);
};
```

---

## 3. BlockBackend：设备前端接口

```c
// block-backend.c:43-95
struct BlockBackend {
    char *name;                            // 名称（如 "ide0-hd0"）
    int refcnt;                            // 引用计数
    BdrvChild *root;                       // 根子节点 → 格式 BDS
    AioContext *ctx;                        // AIO 上下文（原子访问）

    DriveInfo *legacy_dinfo;               // 兼容 drive_new() 信息
    BlockBackendPublic public;             // 公共字段（含 throttle）

    DeviceState *dev;                      // 关联设备模型
    const BlockDevOps *dev_ops;            // 设备操作回调
    void *dev_opaque;                      // 设备上下文

    BlockBackendRootState root_state;      // BDS 移除后保存的状态
    bool enable_write_cache;               // 写缓存开关

    BlockAcctStats stats;                  // I/O 统计

    BlockdevOnError on_read_error;         // 读错误策略
    BlockdevOnError on_write_error;        // 写错误策略

    uint64_t perm;                         // 请求的权限
    uint64_t shared_perm;                  // 共享权限
    bool disable_perm;                     // 禁用权限检查

    int quiesce_counter;                   // 静默计数器（原子）
    CoQueue queued_requests;               // 排队请求

    unsigned int in_flight;                // 进行中请求数（原子）
};
```

---

## 4. BdrvChild：父子关系与 ChildRole

### 4.1 BdrvChild 结构

```c
// block_int-common.h:1049-1084
struct BdrvChild {
    BlockDriverState *bs;                  // 子 BDS 节点
    char *name;                            // 子节点名称
    const BdrvChildClass *klass;           // 类方法（attach/detach/drain 回调）
    BdrvChildRole role;                    // 角色位掩码
    void *opaque;                          // 父上下文

    uint64_t perm;                         // 授予的权限
    uint64_t shared_perm;                  // 允许其他用户的权限

    bool frozen;                           // 冻结（不可替换/分离）
    bool quiesced_parent;                  // 父已被此子 drain

    QLIST_ENTRY(BdrvChild) next;           // BDS.children 链表
    QLIST_ENTRY(BdrvChild) next_parent;    // BDS.parents 链表
};
```

### 4.2 BdrvChildRole 角色位

```c
// block-common.h:476-523
enum BdrvChildRoleBits {
    BDRV_CHILD_DATA       = (1 << 0),   // 存储数据
    BDRV_CHILD_METADATA   = (1 << 1),   // 存储元数据
    BDRV_CHILD_FILTERED   = (1 << 2),   // 过滤节点（透传）
    BDRV_CHILD_COW        = (1 << 3),   // COW backing 节点
    BDRV_CHILD_PRIMARY    = (1 << 4),   // 主子节点

    BDRV_CHILD_IMAGE      = BDRV_CHILD_DATA     // 常用组合：
                          | BDRV_CHILD_METADATA  //   数据+元数据+主
                          | BDRV_CHILD_PRIMARY,
};
```

---

## 5. 节点图与典型链路

```
设备（virtio-blk / IDE / SCSI）
  │
  ▼
BlockBackend (blk)
  │ root (BdrvChild, role=IMAGE)
  ▼
┌─────────────────────────────┐
│ 格式 BDS (qcow2)            │
│   drv = &bdrv_qcow2         │
│   opaque = BDRVQcow2State   │
├─────────────────────────────┤
│ file (BdrvChild, DATA|META) │──→ 协议 BDS (file-posix)
│ backing (BdrvChild, COW)    │──→ backing BDS (qcow2/raw)
└─────────────────────────────┘          │ backing → ...
                                         └──→ 可递归形成 COW 链
```

**关键源码**：
- `BDS.children/backing/file/parents`：`block_int-common.h:1216-1220`
- `child_of_bds` 角色定义：`block.c:1533-1547`

---

## 6. 权限系统

### 6.1 权限常量

```c
// block-common.h:383-420
BLK_PERM_CONSISTENT_READ    = 0x01   // 一致读
BLK_PERM_WRITE              = 0x02   // 可见内容变更
BLK_PERM_WRITE_UNCHANGED    = 0x04   // 不改变可见内容的写入
BLK_PERM_RESIZE             = 0x08   // 改变大小
BLK_PERM_ALL                = 0x0f   // 所有权限
```

### 6.2 权限传播

```
BlockBackend.perm/shared_perm
  │ 通过 BdrvChild 向下传播
  ▼
格式 BDS
  │ drv->bdrv_child_perm() 计算子节点需要的权限
  ▼
协议 BDS
  │ drv->bdrv_check_perm() 检查是否冲突
  │ drv->bdrv_set_perm() 应用权限
  ▼
文件锁（file-posix 用 fcntl/OFD 锁实现）
```

- 权限检查/设置：`block.c:2408-2428`
- 父冲突检查：`block.c:2224-2281`
- 子权限转发：`block.c:2283-2299`
- 累计传播到子树：`block.c:2514-2555`

---

## 7. BlockLimits：I/O 约束

```c
// block_int-common.h:808-919
struct BlockLimits {
    uint32_t request_alignment;          // 请求对齐（字节）
    int32_t max_pdiscard;                // 最大 discard 大小
    uint32_t pdiscard_alignment;         // discard 对齐
    int32_t max_pwrite_zeroes;           // 最大写零大小
    uint32_t pwrite_zeroes_alignment;    // 写零对齐
    int32_t opt_transfer;                // 最优传输大小
    int32_t max_transfer;                // 最大传输大小
    int32_t max_hw_transfer;             // 硬件最大传输
    int32_t max_hw_iov;                  // 硬件最大 iov 数
    size_t min_mem_alignment;            // 最小内存对齐
    size_t opt_mem_alignment;            // 最优内存对齐
    int max_iov;                         // 最大 iov 数
    bool has_variable_length;            // 可变长度设备
    // zoned 设备字段...
};
```

Block Layer 会根据 `BlockLimits` 自动拆分超大请求、对齐非对齐请求。

---

## 8. bdrv_open：打开路径

```c
// block.c:1877-2019 — bdrv_open_common()
bdrv_open_common(BDS *bs, BlockdevRef *file, QDict *options, int flags, Error **errp)
{
    // 1. 创建运行时选项，解析 QDict
    // 2. 查找驱动（bdrv_find_format）
    // 3. 设置标志：force_share、只读、copy-on-read、detect-zeroes
    // 4. 设置 filename / exact_filename
    // 5. 调用 bdrv_open_driver()
}

// block.c:1659-1747 — bdrv_open_driver()
bdrv_open_driver(BDS *bs, BlockDriver *drv, ...)
{
    // 1. 分配 bs->opaque = g_malloc0(drv->instance_size)
    // 2. bs->drv = drv
    // 3. 调用 drv->bdrv_open(bs, options, flags, errp)
    // 4. 设置 supported_*_flags
    // 5. 失败时清理 opaque 和子节点
}
```

---

## 9. I/O 入口：BlockBackend 层

### 9.1 读取入口

```c
// block-backend.c:1364-1401
// blk_co_pread() → blk_co_preadv() → blk_co_do_preadv_part()

// block-backend.c:1330-1362
blk_co_do_preadv_part(BlockBackend *blk, int64_t offset, int64_t bytes,
                       QEMUIOVector *qiov, size_t qiov_offset, flags)
{
    blk_wait_while_drained(blk);         // 等待 drain 结束
    bs = blk_bs(blk);
    blk_check_byte_request(blk, ...);    // 边界检查
    bdrv_inc_in_flight(bs);              // ① in_flight++

    // ② I/O 限流
    if (blk->public.throttle_group_member.throttle_state) {
        throttle_group_co_io_limits_intercept(..., THROTTLE_READ);
    }

    // ③ 下发到 BDS 层
    ret = bdrv_co_preadv_part(blk->root, offset, bytes, qiov, ...);
    bdrv_dec_in_flight(bs);              // ④ in_flight--
    return ret;
}
```

### 9.2 写入入口

```c
// block-backend.c:1404-1455
blk_co_do_pwritev_part(BlockBackend *blk, ...)
{
    // 类似读取路径
    // 额外：如果 !enable_write_cache → 设置 BDRV_REQ_FUA 标志
    // 限流 THROTTLE_WRITE
    // 调用 bdrv_co_pwritev_part(blk->root, ...)
}
```

---

## 10. 核心协程 I/O 路径

### 10.1 驱动分发

```c
// io.c:974-1041
bdrv_driver_preadv(BDS *bs, int64_t offset, int64_t bytes,
                   QEMUIOVector *qiov, size_t qiov_offset, int flags)
{
    // 优先调用 drv->bdrv_co_preadv_part（新接口）
    if (drv->bdrv_co_preadv_part) {
        return drv->bdrv_co_preadv_part(bs, offset, bytes, qiov,
                                        qiov_offset, flags);
    }
    // 回退到 drv->bdrv_co_preadv（旧接口）
    // 回退到 drv->bdrv_co_readv（更旧的扇区接口）
}
```

### 10.2 对齐读取

```c
// io.c:1329-1429
bdrv_aligned_preadv(BdrvChild *child, BdrvTrackedRequest *req,
                    int64_t offset, int64_t bytes, int64_t align,
                    QEMUIOVector *qiov, size_t qiov_offset, int flags)
{
    // 1. 验证对齐
    assert(is_power_of_2(align));
    assert((offset & (align - 1)) == 0);

    // 2. Copy-on-Read 处理
    if (flags & BDRV_REQ_COPY_ON_READ) {
        bdrv_make_request_serialising(req, bdrv_get_cluster_size(bs));
        // 检查是否已分配
        ret = bdrv_co_is_allocated(child->bs, offset, bytes, &pnum);
        if (!ret) {
            // 未分配 → 执行 CoR
            bdrv_co_do_copy_on_readv(child, ...);
        }
    } else {
        bdrv_wait_serialising_requests(req);
    }

    // 3. 分块读取（按 max_transfer 拆分）
    while (bytes_remaining) {
        num = MIN(bytes_remaining, max_transfer);
        ret = bdrv_driver_preadv(bs, offset + bytes - bytes_remaining,
                                 num, qiov, qiov_offset + progress, 0);
        bytes_remaining -= num;
    }
}
```

### 10.3 完整读取流程

```
blk_co_preadv()
  → blk_co_do_preadv_part()
    → throttle_group_co_io_limits_intercept()    // 限流
    → bdrv_co_preadv_part(blk->root, ...)
      → tracked_request_begin()                   // 请求追踪
      → bdrv_pad_request()                        // 对齐填充
      → bdrv_aligned_preadv()
        → bdrv_wait_serialising_requests()        // 等待冲突请求
        → bdrv_driver_preadv()                    // 驱动分发
          → drv->bdrv_co_preadv_part()           // 格式驱动
            → bdrv_co_preadv_part(bs->file, ...) // 递归到协议层
              → drv->bdrv_co_preadv_part()       // 协议驱动（实际 I/O）
      → tracked_request_end()                     // 请求完成
```

---

## 11. 请求追踪与串行化

### 11.1 BdrvTrackedRequest

```c
// block_int-common.h:63-91
enum BdrvTrackedRequestType {
    BDRV_TRACKED_READ,
    BDRV_TRACKED_WRITE,
    BDRV_TRACKED_DISCARD,
    BDRV_TRACKED_TRUNCATE,
};

typedef struct BdrvTrackedRequest {
    BlockDriverState *bs;
    int64_t offset;                  // 请求起始偏移
    int64_t bytes;                   // 请求长度
    enum BdrvTrackedRequestType type;

    bool serialising;                // 是否串行化
    int64_t overlap_offset;          // 串行化范围起始
    int64_t overlap_bytes;           // 串行化范围长度

    QLIST_ENTRY(BdrvTrackedRequest) list;
    Coroutine *co;                   // 所有者协程（死锁检测）
    CoQueue wait_queue;              // 等待此请求的协程队列
    struct BdrvTrackedRequest *waiting_for;  // 正在等待的请求
} BdrvTrackedRequest;
```

### 11.2 追踪与串行化流程

```c
// io.c:604-628 — 开始追踪
tracked_request_begin(req, bs, offset, bytes, type)
// 将请求添加到 bs->tracked_requests

// io.c:583-599 — 结束追踪
tracked_request_end(req)
// 从列表移除，唤醒 wait_queue 中的等待者

// io.c:694-710 — 设置串行化
tracked_request_set_serialising(req, overlap_offset, overlap_bytes)
// serialising = true, bs->serialising_in_flight++

// io.c:647-678 — 查找冲突
bdrv_find_conflicting_request(req)
// 遍历 tracked_requests，查找重叠的串行化请求

// io.c:681-691 — 等待串行化
bdrv_wait_serialising_requests_locked(req)
// 如果有冲突 → req->waiting_for = conflict → CoQueue 等待
```

**用途**：Copy-on-Read 和写入需要串行化同一集群的并发访问，防止数据不一致。

---

## 12. Copy-on-Read

```c
// io.c:1166-1322
bdrv_co_do_copy_on_readv(BdrvChild *child, int64_t offset, int64_t bytes,
                          QEMUIOVector *qiov, size_t qiov_offset, int flags)
{
    // 1. 对齐到簇/子簇边界
    align_offset = QEMU_ALIGN_DOWN(offset, align);
    align_bytes  = QEMU_ALIGN_UP(offset + bytes, align) - align_offset;

    // 2. 检查分配状态
    ret = bdrv_co_is_allocated(bs, align_offset, align_bytes, &pnum);

    // 3. 如果未分配 → 分配 bounce buffer 从 backing 读取
    bounce_buffer = qemu_blockalign(bs, pnum);
    ret = bdrv_driver_preadv(bs, ..., bounce_buffer_qiov, ...);

    // 4. 写入当前层（CoW 写入）
    if (!skip_write) {
        ret = bdrv_driver_pwritev(bs, ..., bounce_buffer_qiov, ...);
    }

    // 5. 复制到 Guest 缓冲区
    qemu_iovec_from_buf(qiov, qiov_offset, bounce_buffer + skip_bytes, bytes);
}
```

---

## 13. 写入路径与 Flush

### 13.1 驱动写入

```c
// io.c:1043-1127
bdrv_driver_pwritev(BDS *bs, int64_t offset, int64_t bytes,
                    QEMUIOVector *qiov, size_t qiov_offset, int flags)
{
    // 1. 分发到驱动
    if (drv->bdrv_co_pwritev_part) {
        ret = drv->bdrv_co_pwritev_part(bs, offset, bytes, qiov,
                                        qiov_offset, flags & supported);
    }

    // 2. FUA 模拟：如果驱动不支持 FUA 但上层要求
    if (has_fua && !(bs->supported_write_flags & BDRV_REQ_FUA)) {
        ret = bdrv_co_flush(bs);    // 用 flush 模拟 FUA
    }
}
```

### 13.2 Flush 链

```c
// blk_co_do_flush() → bdrv_co_flush(blk_bs(blk))
//   → drv->bdrv_co_flush_to_os()    // 格式层缓存刷出
//   → bdrv_co_flush(bs->file->bs)   // 递归到协议层
//     → drv->bdrv_co_flush_to_disk() // 物理刷盘（fsync）
```

### 13.3 写缓存控制

- `blk->enable_write_cache = false` → 写入自动加 `BDRV_REQ_FUA`
- `blk->enable_write_cache = true` → 依赖 Guest 显式 flush

---

## 14. 协程包装器模式

QEMU Block Layer 的所有 I/O 都是协程函数。同步调用者通过自动生成的包装器转换：

```c
// block-common.h:31-55 — 标记宏
// co_wrapper_mixed: 混合模式（协程内直接调用，非协程创建新协程）
// co_wrapper: 仅在非协程环境使用

// scripts/block-coroutine-wrapper.py:152-203 — 生成器
// 生成的包装器：
// 1. 检查是否已在协程中
//    - 是 → 直接调用协程函数
//    - 否 → 创建协程 + bdrv_poll_co() 轮询等待
```

**示例**：
```c
// 声明：
int co_wrapper_mixed blk_pread(BlockBackend *blk, ...);

// 自动生成：
int blk_pread(BlockBackend *blk, ...) {
    if (qemu_in_coroutine()) {
        return blk_co_pread(blk, ...);    // 直接调用
    }
    // 创建协程运行 blk_co_pread，然后 poll 等待
    Coroutine *co = qemu_coroutine_create(blk_pread_entry, ...);
    bdrv_poll_co(co);
}
```

---

## 15. I/O 限流

### 15.1 限流拦截

```c
// throttle-groups.c:357-389
throttle_group_co_io_limits_intercept(ThrottleGroupMember *tgm,
                                      int64_t bytes, ThrottleDirection direction)
{
    // 1. 获取组锁
    qemu_mutex_lock(&tg->lock);

    // 2. 获取下一个令牌（轮转调度）
    token = next_throttle_token(tgm, direction);

    // 3. 检查是否需要等待
    must_wait = throttle_group_schedule_timer(token, direction);

    // 4. 等待（协程挂起）
    if (must_wait || tgm->pending_reqs[direction]) {
        tgm->pending_reqs[direction]++;
        qemu_co_queue_wait(&tgm->throttled_reqs[direction], &tg->lock);
        tgm->pending_reqs[direction]--;
    }

    // 5. 记账
    throttle_account(tgm->throttle_state, direction, bytes);

    // 6. 调度下一个请求
    schedule_next_request(tgm, direction);

    qemu_mutex_unlock(&tg->lock);
}
```

### 15.2 限流位置

限流在 `blk_co_do_preadv_part()` 和 `blk_co_do_pwritev_part()` 中调用，位于 BlockBackend 层，在实际 I/O 下发之前。

---

## 16. Drain 机制

```c
// io.c:349-427
bdrv_do_drained_begin(BDS *bs, BdrvChild *parent, ...)
{
    // 1. quiesce_counter++ → 阻止新请求
    // 2. 通知所有父节点 drain
    // 3. bdrv_drain_poll() 等待 in_flight == 0
}

bdrv_do_drained_end(BDS *bs, BdrvChild *parent, ...)
{
    // 1. quiesce_counter-- → 恢复请求
    // 2. 通知父节点 drain 结束
    // 3. 唤醒排队请求
}

// io.c:508-534 — 全局 drain
bdrv_drain_all_begin()
{
    // 对所有 BDS 执行 drain
    // AIO_WAIT_WHILE_UNLOCKED(NULL, bdrv_drain_all_poll())
}
```

**用途**：图变更（热插拔、快照、迁移）前必须 drain 所有进行中 I/O。

---

## 17. Block Job 基础设施

### 17.1 Job 核心结构

```c
// job.h:44-187
typedef struct Job {
    char *id;                         // 任务 ID
    const JobDriver *driver;          // 驱动 vtable
    Coroutine *co;                    // 执行协程
    bool auto_finalize;               // 自动 finalize
    bool auto_dismiss;                // 自动 dismiss
    BlockCompletionFunc *cb;          // 完成回调
    ProgressMeter progress;           // 进度指示器
    AioContext *aio_context;          // AIO 上下文
    int refcnt;                       // 引用计数
    JobStatus status;                 // 当前状态
    // ...
} Job;
```

### 17.2 状态机

```c
// job.c:58-84 — JobSTT 状态转移表
// job.c:207-235 — 状态/动作映射

// 状态流：
// CREATED → RUNNING → WAITING → PENDING → CONCLUDING → NULL
//                  ↘ PAUSED ↗
//          ↘ ABORTING → CONCLUDING → NULL
```

### 17.3 Block Job 类型

| 类型 | 用途 |
|------|------|
| **mirror** | 实时磁盘镜像（同步到目标） |
| **commit** | 合并快照链 |
| **stream** | 从 backing 流式复制数据 |
| **backup** | 增量/全量备份 |

---

## 18. 数据流全景图

### 18.1 读取全路径

```
Guest 发起磁盘读取
  │
  ▼
virtio-blk / IDE / SCSI 设备
  │ blk_co_preadv()
  ▼
BlockBackend (blk)
  │ ① in_flight++
  │ ② throttle_group_co_io_limits_intercept() — 限流
  │ ③ bdrv_co_preadv_part(blk->root, ...)
  ▼
BdrvChild (root) → 格式 BDS (qcow2)
  │ tracked_request_begin() — 请求追踪
  │ bdrv_pad_request() — 对齐
  │ bdrv_aligned_preadv()
  │   ├── Copy-on-Read? → bdrv_co_do_copy_on_readv()
  │   └── bdrv_driver_preadv()
  │         → drv->bdrv_co_preadv_part() — qcow2 解密/解压/L1L2 查找
  │           → bdrv_co_preadv_part(bs->file, ...)
  ▼
BdrvChild (file) → 协议 BDS (file-posix)
  │ drv->bdrv_co_preadv_part()
  │   → preadv() / io_submit() / io_uring — 实际系统调用
  ▼
返回数据 → 逐层返回 → Guest
```

### 18.2 节点图拓扑示例

```
┌──────────────┐     ┌──────────────┐
│ BlockBackend │     │ BlockBackend │
│  "virtio0"   │     │  "ide0-hd0"  │
└──────┬───────┘     └──────┬───────┘
       │ root               │ root
       ▼                    ▼
  ┌─────────┐         ┌─────────┐
  │ qcow2   │         │ raw     │
  │ BDS     │         │ BDS     │
  └─┬───┬───┘         └────┬────┘
    │   │ backing           │ file
    │   ▼                   ▼
    │ ┌─────────┐     ┌──────────┐
    │ │ qcow2   │     │file-posix│
    │ │ backing │     │ BDS      │
    │ └────┬────┘     └──────────┘
    │ file │
    ▼      ▼
  ┌──────────┐  ┌──────────┐
  │file-posix│  │file-posix│
  │ BDS      │  │ BDS      │
  └──────────┘  └──────────┘
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/block/block_int-common.h` | ~1300 | BdrvTrackedRequest (63-91)、BlockDriver (94-800)、BlockLimits (808-919)、BdrvChild (1049-1084)、BlockDriverState (1104-1292) |
| `include/block/block-common.h` | ~600 | co_wrapper 宏 (31-76)、BLK_PERM_* (383-420)、BdrvChildRoleBits (476-523) |
| `block/block-backend.c` | ~2400 | BlockBackend (43-95)、blk_co_do_preadv_part (1330-1362)、blk_co_do_pwritev_part (1404-1455)、blk_co_flush (1863-1872) |
| `block/io.c` | ~3500 | bdrv_driver_preadv (974-1041)、bdrv_driver_pwritev (1043-1127)、bdrv_co_do_copy_on_readv (1166-1322)、bdrv_aligned_preadv (1329-1429)、tracked_request_begin/end (583-628)、drain (315-534) |
| `block.c` | ~8000 | bdrv_open_driver (1659-1747)、bdrv_open_common (1877-2019)、child_of_bds (1533-1547)、权限系统 (2224-2555) |
| `block/throttle-groups.c` | ~700 | throttle_group_co_io_limits_intercept (357-389) |
| `include/qemu/job.h` | ~400 | Job 结构 (44-187) |
| `job.c` | ~550 | JobSTT 状态转移表 (58-84)、状态/动作映射 (207-235) |
| `scripts/block-coroutine-wrapper.py` | ~210 | 协程包装器生成器 (152-203) |
