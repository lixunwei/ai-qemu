# QEMU RAM 管理与脏页追踪深度分析

> QEMU 版本：11.0.50  
> 分析范围：RAMBlock 管理、内存分配（mmap/huge pages）、内存后端（QOM）、脏页追踪（VGA/CODE/MIGRATION）、KVM dirty log/ring、TCG 脏页机制、迁移 RAM 保存  
> 前置阅读：[00-内存子系统深度分析.md](00-内存子系统深度分析.md)（MemoryRegion/AddressSpace/FlatView/MMIO 分发）

---

## 目录

- [第一部分：RAMBlock 管理](#第一部分ramblock-管理)
  - [1. RAMBlock 结构体](#1-ramblock-结构体)
  - [2. RAMList 全局链表](#2-ramlist-全局链表)
  - [3. RAMBlock 注册流程](#3-ramblock-注册流程)
  - [4. Guest RAM 分配路径](#4-guest-ram-分配路径)
  - [5. mmap 内存映射](#5-mmap-内存映射)
- [第二部分：大页与内存后端](#第二部分大页与内存后端)
  - [6. Huge Pages 支持](#6-huge-pages-支持)
  - [7. 内存后端 QOM 对象](#7-内存后端-qom-对象)
  - [8. RAM 预分配](#8-ram-预分配)
- [第三部分：脏页追踪机制](#第三部分脏页追踪机制)
  - [9. 脏页位图架构](#9-脏页位图架构)
  - [10. memory_region_set_dirty 核心 API](#10-memory_region_set_dirty-核心-api)
  - [11. TCG 脏页追踪](#11-tcg-脏页追踪)
  - [12. KVM Dirty Log](#12-kvm-dirty-log)
  - [13. KVM Dirty Ring](#13-kvm-dirty-ring)
  - [14. MemoryListener 脏页回调](#14-memorylistener-脏页回调)
- [第四部分：迁移 RAM 保存](#第四部分迁移-ram-保存)
  - [15. 迁移位图同步](#15-迁移位图同步)
  - [16. Precopy 脏页迭代](#16-precopy-脏页迭代)
  - [17. Clear Bitmap 优化](#17-clear-bitmap-优化)
- [第五部分：NUMA 与内存热插拔](#第五部分numa-与内存热插拔)
  - [18. NUMA 拓扑](#18-numa-拓扑)
  - [19. 内存热插拔 (PC-DIMM/NVDIMM)](#19-内存热插拔-pc-dimmnvdimm)
  - [20. virtio-mem 动态内存](#20-virtio-mem-动态内存)
  - [21. ARM virt 内存布局](#21-arm-virt-内存布局)
- [附录](#附录)

---

## 第一部分：RAMBlock 管理

### 1. RAMBlock 结构体

每个 Guest RAM 区域对应一个 `RAMBlock`，是物理内存管理的最小单元。

**定义**：`ramblock.h:25-92`

```c
struct RAMBlock {
    struct rcu_head rcu;
    struct MemoryRegion *mr;       // 关联的 MemoryRegion
    uint8_t *host;                 // 宿主虚拟地址（mmap 返回值）
    uint8_t *colo_cache;           // COLO 缓存（容错）
    ram_addr_t offset;             // 在全局 RAM 地址空间中的偏移
    ram_addr_t used_length;        // 当前使用长度
    ram_addr_t max_length;         // 最大长度（支持 resize）
    void (*resized)(const char*, uint64_t length, void *host);
    uint32_t flags;                // RAM_SHARED, RAM_PMEM, RAM_NORESERVE 等
    char idstr[256];               // 标识名（迁移用）
    QLIST_ENTRY(RAMBlock) next;    // 链表节点
    int fd;                        // 后备文件描述符（-1 表示匿名）
    uint64_t fd_offset;            // 文件中的偏移
    int guest_memfd;               // KVM guest_memfd（机密计算）
    size_t page_size;              // 页大小（4K/2M/1G）

    /* 迁移相关位图 */
    unsigned long *bmap;           // 脏页位图（per-RAMBlock）
    unsigned long *file_bmap;      // mapped-ram 迁移：文件中存在的页
    unsigned long *receivedmap;    // 目标端：已接收页位图
    unsigned long *clear_bmap;     // 延迟清除位图
    uint8_t clear_bmap_shift;      // 清除粒度（一个 bit 覆盖多少页）
    ram_addr_t postcopy_length;    // postcopy 长度
};
```

**关键 flags**（`ramblock.h` 及 `memory.h`）：

| Flag | 含义 |
|------|------|
| `RAM_SHARED` | MAP_SHARED 映射（与其他进程共享） |
| `RAM_PMEM` | 持久化内存（pmem） |
| `RAM_NORESERVE` | MAP_NORESERVE（不预留 swap） |
| `RAM_NAMED_FILE` | 有命名后备文件 |
| `RAM_READONLY` | 只读内存 |
| `RAM_GUEST_MEMFD` | KVM guest_memfd（机密 VM） |

### 2. RAMList 全局链表

所有 RAMBlock 组织在全局 `ram_list` 中。

**定义**：`ramlist.h:43-53`

```c
typedef struct RAMList {
    QemuMutex mutex;                          // 保护写操作
    RAMBlock *mru_block;                      // 最近使用的 block（加速查找）
    QLIST_HEAD(, RAMBlock) blocks;            // RAMBlock 链表（RCU 保护）
    DirtyMemoryBlocks *dirty_memory[DIRTY_MEMORY_NUM]; // 3 个脏页位图
    unsigned int num_dirty_blocks;            // 脏页位图块数
    uint32_t version;                         // 版本号（每次修改递增）
    QLIST_HEAD(, RAMBlockNotifier) ramblock_notifiers; // 通知链
} RAMList;

extern RAMList ram_list;                      // 全局单例
```

**脏页位图分块设计**（`ramlist.h:12-41`）：

```
DirtyMemoryBlocks（RCU 保护）
  ├── blocks[0] → unsigned long 位图（覆盖 256K×8 = 2M 页 = 8GB）
  ├── blocks[1] → unsigned long 位图
  └── ...
```

分块允许在 RCU 保护下动态增长：新增 RAMBlock 时分配新的 `DirtyMemoryBlocks` 数组，旧指针不变，读线程安全。

### 3. RAMBlock 注册流程

**函数**：`ram_block_add()`（`physmem.c:2149-2289`）

```
ram_block_add(new_block)
  │
  ├── 1. 获取 ramlist 锁
  │
  ├── 2. 分配偏移
  │     └── find_ram_offset(max_length)
  │           遍历现有 blocks，找到不重叠的空隙
  │
  ├── 3. 分配宿主内存（如果 host == NULL）
  │     ├── Xen: xen_ram_alloc()
  │     └── 普通: qemu_anon_ram_alloc(max_length, &align, shared, noreserve)
  │           └── mmap(MAP_ANONYMOUS | MAP_PRIVATE/MAP_SHARED)
  │
  ├── 4. 可选：分配 guest_memfd（RAM_GUEST_MEMFD 标志）
  │     └── kvm_create_guest_memfd(size, flags)
  │
  ├── 5. 启用 KSM 合并
  │     └── memory_try_enable_merging(host, max_length)
  │
  ├── 6. 扩展脏页位图
  │     └── dirty_memory_extend(old_ram_size, new_ram_size)
  │
  ├── 7. 插入链表
  │     └── QLIST_INSERT_AFTER/BEFORE/HEAD
  │
  ├── 8. 更新版本号
  │     └── ram_list.version++
  │
  └── 9. 释放锁，通知监听器
        └── ram_block_notify_add(host, size, max_size)
```

### 4. Guest RAM 分配路径

从高层 API 到底层 mmap 的完整调用链：

```
memory_region_init_ram(mr, owner, name, size)    [memory.c:1594]
  └── memory_region_init_ram_flags_nomigrate(mr, ..., ram_flags)
        └── qemu_ram_alloc(size, ram_flags, mr)  [physmem.c:2554]
              └── qemu_ram_alloc_internal(size, size, NULL, mr, ram_flags)
                    │                             [physmem.c:2460]
                    ├── new_block = g_malloc0(sizeof(RAMBlock))
                    ├── new_block->mr = mr
                    ├── new_block->used_length = size
                    ├── new_block->max_length = max_size
                    ├── new_block->flags = ram_flags
                    ├── new_block->fd = -1
                    └── ram_block_add(new_block)
                          └── qemu_anon_ram_alloc(size, ...)
                                └── qemu_ram_mmap(-1, size, align, flags, 0)
                                      └── mmap(NULL, size, PROT_RW, flags, -1, 0)
```

**文件后备 RAM**（`-mem-path` 或 `memory-backend-file`）：

```
qemu_ram_alloc_from_file(size, mr, flags, mem_path, offset)
  └── file_ram_alloc(block, size, fd, offset)    [physmem.c:1708-1782]
        ├── qemu_fd_getpagesize(fd)  → 检测 huge page 大小
        ├── block->page_size = page_size
        └── qemu_ram_mmap(fd, size, align, map_flags, offset)
```

### 5. mmap 内存映射

**核心函数**：`qemu_ram_mmap()`（`mmap-alloc.c:184-295`）

```
qemu_ram_mmap(fd, size, align, qemu_map_flags, offset)
  │
  ├── 计算对齐：guardsize = MAX(align, getpagesize())
  │
  ├── 第一步：保留地址空间（过大以保证对齐）
  │     └── mmap(NULL, total, PROT_NONE, MAP_ANONYMOUS|MAP_PRIVATE, -1, 0)
  │
  ├── 第二步：在对齐地址上实际映射
  │     └── mmap_activate(ptr, size, fd, qemu_map_flags, offset)
  │           ├── prot = PROT_READ | PROT_WRITE
  │           ├── shared? → MAP_SHARED : MAP_PRIVATE
  │           ├── fd < 0? → MAP_ANONYMOUS
  │           ├── noreserve? → MAP_NORESERVE
  │           ├── sync? → MAP_SYNC | MAP_SHARED_VALIDATE (DAX/pmem)
  │           └── mmap(ptr, size, prot, flags | MAP_FIXED, fd, offset)
  │
  └── 返回对齐后的指针
```

**`MAP_SHARED` vs `MAP_PRIVATE`**：

| 映射类型 | 使用场景 | 特点 |
|---------|---------|------|
| `MAP_PRIVATE` | 默认 Guest RAM | 写时复制，进程独占 |
| `MAP_SHARED` | vhost-user、迁移、多进程 | 跨进程可见，需要后备文件/memfd |

---

## 第二部分：大页与内存后端

### 6. Huge Pages 支持

QEMU 支持通过 hugetlbfs 或 memfd 使用大页。

#### 大页检测

**函数**：`qemu_fd_getpagesize()`（`mmap-alloc.c:60-82`）

```c
size_t qemu_fd_getpagesize(int fd) {
    struct statfs fs;
    if (fstatfs(fd, &fs) == 0) {
        switch (fs.f_type) {
        case HUGETLBFS_MAGIC:
            return fs.f_bsize;        // hugetlbfs 报告实际大页大小
        ...
        }
    }
    return qemu_real_host_page_size(); // 默认 4KB
}
```

#### 使用方式

**方式一：`-mem-path`**（传统）

```bash
# 挂载 hugetlbfs
mount -t hugetlbfs -o pagesize=2M nodev /dev/hugepages

# QEMU 使用
qemu-system-aarch64 -m 4G -mem-path /dev/hugepages ...
```

**方式二：`memory-backend-file`**（现代，推荐）

```bash
-object memory-backend-file,id=mem0,size=4G,mem-path=/dev/hugepages,prealloc=on
-machine memory-backend=mem0
```

**方式三：`memory-backend-memfd`**（无需挂载）

```bash
-object memory-backend-memfd,id=mem0,size=4G,hugetlb=on,hugetlbsize=2M
-machine memory-backend=mem0
```

### 7. 内存后端 QOM 对象

QEMU 使用 QOM 对象 (`HostMemoryBackend`) 管理 Guest RAM 分配策略。

```
HostMemoryBackend (TYPE_MEMORY_BACKEND)           [hostmem.c]
  ├── HostMemoryBackendRam                        [hostmem-ram.c]
  │     匿名 mmap，最简单
  │
  ├── HostMemoryBackendFile                       [hostmem-file.c]
  │     文件后备：mem-path, align, offset, readonly
  │     支持 hugetlbfs / DAX / pmem
  │
  └── HostMemoryBackendMemfd                      [hostmem-memfd.c]
        memfd_create() 匿名文件
        支持 hugetlb + seal（密封，防止 resize）
```

#### HostMemoryBackendFile 分配（`hostmem-file.c:40-95`）

```
file_backend_memory_alloc(backend)
  │
  ├── 构建 ram_flags:
  │     ├── share=on  → RAM_SHARED
  │     ├── pmem=on   → RAM_PMEM
  │     ├── readonly  → RAM_READONLY_FD
  │     └── rom       → RAM_READONLY
  │
  └── memory_region_init_ram_from_file(&backend->mr, ...)
        └── qemu_ram_alloc_from_file(size, mr, ram_flags, mem-path, offset)
              └── file_ram_alloc() → mmap(fd, ...)
```

#### HostMemoryBackendMemfd 分配（`hostmem-memfd.c:33-66`）

```
memfd_backend_memory_alloc(backend)
  │
  ├── fd = qemu_memfd_create(name, size + guard_size,
  │         hugetlb=m->hugetlb, hugetlbsize=m->hugetlbsize, ...)
  │
  └── memory_region_init_ram_from_fd(&backend->mr, ..., fd, ...)
```

#### 内存后端公共属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `size` | uint64 | 内存大小 |
| `host-nodes` | bitmap | NUMA 节点绑定 |
| `policy` | enum | NUMA 策略（default/preferred/bind/interleave） |
| `prealloc` | bool | 是否预分配 |
| `share` | bool | MAP_SHARED 映射 |
| `dump` | bool | 是否包含在 coredump 中 |
| `merge` | bool | KSM 合并 |

### 8. RAM 预分配

预分配确保页面在 Guest 访问前已分配物理页，避免运行时缺页。

**函数**：`qemu_prealloc_mem()`（`oslib-posix.c:607-682`）

```
qemu_prealloc_mem(fd, area, sz, max_threads, threadfn, errp)
  │
  ├── 优先尝试: MADV_POPULATE_WRITE         [oslib-posix.c:423-441]
  │     └── madvise(area, sz, MADV_POPULATE_WRITE)
  │           内核预填充，不需要用户空间触碰
  │           Linux 5.14+ 支持
  │
  └── 回退: 多线程 touch                     [oslib-posix.c:443-570]
        ├── 安装 SIGBUS 处理器（huge pages 可能 SIGBUS）
        ├── 创建 max_threads 个线程
        │     └── do_touch_pages()
        │           对每页执行 *(volatile char *)addr = 0
        │           触发缺页分配
        └── 等待所有线程完成
```

---

## 第三部分：脏页追踪机制

### 9. 脏页位图架构

QEMU 维护三组独立的脏页位图，分别服务不同客户端：

**定义**：`ram_addr.h:28-31`

```c
#define DIRTY_MEMORY_VGA       0   // VGA 显存更新追踪
#define DIRTY_MEMORY_CODE      1   // 自修改代码检测（TCG 用）
#define DIRTY_MEMORY_MIGRATION 2   // 实时迁移脏页追踪
#define DIRTY_MEMORY_NUM       3   // 总数
```

**全局位图结构**：

```
ram_list.dirty_memory[3]          ramlist.h:48
  │
  ├── [VGA]       → DirtyMemoryBlocks
  │                    └── blocks[N] → unsigned long 位图
  │
  ├── [CODE]      → DirtyMemoryBlocks
  │                    └── blocks[N] → unsigned long 位图
  │
  └── [MIGRATION] → DirtyMemoryBlocks
                       └── blocks[N] → unsigned long 位图

每个 blocks[i] 覆盖 DIRTY_MEMORY_BLOCK_SIZE = 256K×8 = 2M 页 = 8GB
每个 bit 对应一个 TARGET_PAGE（通常 4KB）
```

**per-RAMBlock 迁移位图**：除了全局位图，迁移还在每个 `RAMBlock` 上维护：
- `bmap` — 本轮脏页位图（`ramblock.h:47`）
- `clear_bmap` — 延迟清除位图（`ramblock.h:80`）
- `receivedmap` — 目标端已接收位图（`ramblock.h:62`）

### 10. memory_region_set_dirty 核心 API

设备/CPU 修改 RAM 后调用此函数标记脏页。

**函数**：`memory_region_set_dirty()`（`memory.c:2188-2195`）

```
memory_region_set_dirty(mr, addr, size)
  │
  ├── 计算全局 RAM 地址
  │     ram_addr = memory_region_get_ram_addr(mr) + addr
  │
  ├── 获取该 MR 的 dirty log mask
  │     mask = memory_region_get_dirty_log_mask(mr)
  │
  └── physical_memory_set_dirty_range(ram_addr, size, mask)
        │                              [physmem.c:1011-1060]
        │
        ├── 将 ram_addr 转换为页号
        │     page = start >> TARGET_PAGE_BITS
        │
        └── 遍历每个 dirty_memory client:
              ├── MIGRATION: bitmap_set_atomic(blocks[idx], offset, count)
              ├── VGA:       bitmap_set_atomic(...)
              └── CODE:      bitmap_set_atomic(...)
```

**原子操作**：使用 `bitmap_set_atomic()` 保证多线程安全（TCG 多线程或 IOThread 并发标记）。

### 11. TCG 脏页追踪

TCG 翻译的 store 指令需要追踪脏页，通过 TLB 中的 `TLB_NOTDIRTY` 标志实现。

#### 工作流程

```
1. TLB 填充时检查页面清洁状态
   tlb_set_page_full()                   [cputlb.c:1024-1183]
     └── if physical_memory_is_clean(ram_addr, DIRTY_MEMORY_CODE)
           → 设置 TLB 条目: iotlb |= TLB_NOTDIRTY
           → 写该页会触发 slow path

2. Guest 执行 store 指令 → TLB 命中但有 TLB_NOTDIRTY
   → 进入 notdirty_write()              [cputlb.c:1336-1357]

3. notdirty_write() 处理:
   │
   ├── 检查 DIRTY_MEMORY_CODE
   │     if !physical_memory_get_dirty_flag(ram_addr, DIRTY_MEMORY_CODE)
   │       → tb_invalidate_phys_page_fast() 失效该页的翻译块
   │
   ├── 标记脏页
   │     physical_memory_set_dirty_range(ram_addr, size, DIRTY_CLIENTS_NOCODE)
   │     // 标记 VGA + MIGRATION，不重复标记 CODE
   │
   └── 更新 TLB：移除 TLB_NOTDIRTY
         tlb_set_dirty(cpu, vaddr)        [cputlb.c:943-972]
         // 后续写入同一页直接走 fast path
```

**DIRTY_CLIENTS_NOCODE**（`physmem.h:14-28`）：= VGA | MIGRATION（不含 CODE），因为 CODE 在标记后已处理（TB 失效）。

### 12. KVM Dirty Log

KVM 在内核中维护脏页位图，QEMU 通过 ioctl 获取。

#### 启动/停止

```
kvm_log_start(listener, section)         [kvm-all.c:850-882]
  └── kvm_section_update_flags()
        └── 更新 KVMSlot.flags |= KVM_MEM_LOG_DIRTY_PAGES
              └── kvm_set_user_memory_region()
                    └── ioctl(KVM_SET_USER_MEMORY_REGION2)

kvm_log_stop(listener, section)
  └── 移除 KVM_MEM_LOG_DIRTY_PAGES 标志
```

#### 获取脏页

```
kvm_slot_get_dirty_log(s, slot)          [kvm-all.c:934-952]
  │
  ├── d.dirty_bitmap = slot->dirty_bmap  // 用户空间位图
  ├── d.slot = slot_id | (as_id << 16)
  └── kvm_vm_ioctl(s, KVM_GET_DIRTY_LOG, &d)
        // 内核将脏页位图复制到用户空间
        // 并清除内核侧位图

kvm_physical_sync_dirty_bitmap()         [kvm-all.c:1179-1181]
  ├── kvm_slot_get_dirty_log()
  └── kvm_slot_sync_dirty_pages()
        // 将 slot->dirty_bmap 合并到 ram_list.dirty_memory[MIGRATION]
```

#### KVM_CLEAR_DIRTY_LOG（精细清除）

```
kvm_log_clear_one_slot()                 [kvm-all.c:1192-1296]
  └── ioctl(KVM_CLEAR_DIRTY_LOG, &d)
        // 仅清除指定范围，不影响其他区域
        // 减少 TLB flush 开销
```

### 13. KVM Dirty Ring

Dirty Ring 是 KVM 的新一代脏页追踪机制，比 dirty log 更高效。

#### 与 Dirty Log 的对比

| 特性 | Dirty Log | Dirty Ring |
|------|-----------|------------|
| 数据结构 | 全局位图 | Per-vCPU 环形缓冲区 |
| 获取方式 | ioctl（需要暂停 vCPU） | 用户空间直接读取 |
| 粒度 | 页级位图 | 精确到每次写入（GFN+slot） |
| 开销 | 全量 bitmap 复制 | 增量消费 |
| 内核版本 | 长期支持 | Linux 5.11+ |

#### 初始化

```
kvm_dirty_ring_init(s)                   [kvm-all.c:1801-1861]
  │
  ├── 协商能力
  │     kvm_check_extension(KVM_CAP_DIRTY_LOG_RING)
  │     或 KVM_CAP_DIRTY_LOG_RING_ACQ_REL（有获取-释放语义）
  │
  ├── 启用
  │     kvm_vm_enable_cap(KVM_CAP_DIRTY_LOG_RING, ring_size)
  │
  └── 启动 reaper 线程
        kvm_dirty_ring_reaper_init(s)
```

#### Dirty Ring 结构

```
Per-vCPU 环形缓冲区:
  cpu->kvm_dirty_gfns = mmap(vcpu_fd, ring_offset)

  struct kvm_dirty_gfn {
      __u32 flags;    // KVM_DIRTY_GFN_F_DIRTY → COLLECTED
      __u32 slot;     // slot_id | (as_id << 16)
      __u64 offset;   // 页内偏移（GFN × PAGE_SIZE）
  };

  ┌─────────────────────────────────────────┐
  │ [0] dirty  [1] dirty  [2] collected ... │
  │         ↑ fetch_index                   │
  └─────────────────────────────────────────┘
```

#### Reap（收割）流程

```
kvm_dirty_ring_reap_one(s, cpu)          [kvm-all.c:1010-1044]
  │
  └── while (dirty_gfns[fetch] is DIRTY):
        ├── kvm_dirty_ring_mark_page(s, as_id, slot_id, offset)
        │     // 在 slot->dirty_bmap 中标记对应页
        ├── dirty_gfn_set_collected(cur)
        │     // 标记为已收集，释放 ring 槽位
        └── fetch++, count++

kvm_dirty_ring_reaper_thread(s)          [kvm-all.c:1757-1790]
  │
  └── while (true):
        ├── sleep(1)                     // 每秒一次
        ├── 如果 dirtylimit_in_service() → 继续 sleep
        └── bql_lock()
              kvm_dirty_ring_reap(s, NULL)  // 收割所有 vCPU
              bql_unlock()
```

#### Dirty Ring Flush（迁移同步时）

```
kvm_dirty_ring_flush()                   [kvm-all.c:1134-1150]
  │
  ├── 踢出所有 vCPU（强制退出 KVM_RUN）
  │     CPU_FOREACH(cpu) → kvm_cpu_kick(cpu)
  │
  ├── 等待所有 vCPU 退出
  │
  └── 收割所有 ring
        kvm_dirty_ring_reap(s, NULL)
```

### 14. MemoryListener 脏页回调

MemoryListener 将脏页追踪与地址空间拓扑变更联系起来。

**相关回调**（`memory.h:883-1013`）：

```c
struct MemoryListener {
    ...
    void (*log_start)(MemoryListener *listener,
                      MemoryRegionSection *section, int old, int new);
    void (*log_stop)(MemoryListener *listener,
                     MemoryRegionSection *section, int old, int new);
    void (*log_sync)(MemoryListener *listener,
                     MemoryRegionSection *section);
    void (*log_sync_global)(MemoryListener *listener,
                            bool last_stage);
    ...
};
```

**KVM 注册**（`kvm-all.c:2102-2115`）：

```c
kml->listener.log_start = kvm_log_start;
kml->listener.log_stop = kvm_log_stop;
kml->listener.log_sync = kvm_log_sync;
kml->listener.log_sync_global = kvm_log_sync_global;
```

**脏页同步流程**：

```
memory_global_dirty_log_sync(last_stage)
  │
  └── MEMORY_LISTENER_CALL_GLOBAL(log_sync_global, ...)
        │
        ├── KVM (dirty log):
        │     kvm_log_sync_global()      [kvm-all.c:1982-1999]
        │       └── 遍历所有 slot → kvm_physical_sync_dirty_bitmap()
        │
        └── KVM (dirty ring):
              kvm_log_sync_global()
                ├── kvm_dirty_ring_flush()     // 先刷新 ring
                └── 遍历所有 slot → sync slot bitmap
```

---

## 第四部分：迁移 RAM 保存

### 15. 迁移位图同步

每轮迁移迭代前，同步 KVM/TCG 的脏页到 per-RAMBlock 位图。

**函数**：`migration_bitmap_sync()`（`ram.c:1134-1172`）

```
migration_bitmap_sync(rs, last_stage)
  │
  ├── 1. 全局脏页同步
  │     memory_global_dirty_log_sync(last_stage)
  │     → KVM: kvm_log_sync_global()
  │     → 从内核获取脏页 → ram_list.dirty_memory[MIGRATION]
  │
  ├── 2. 同步到 per-RAMBlock 位图
  │     RAMBLOCK_FOREACH_NOT_IGNORED(block):
  │       ramblock_sync_dirty_bitmap(rs, block)
  │       → 从 ram_list.dirty_memory[MIGRATION] 复制到 block->bmap
  │       → 统计新脏页数：rs->num_dirty_pages_period
  │
  ├── 3. 清理全局位图
  │     memory_global_after_dirty_log_sync()
  │
  └── 4. 速率控制（每秒一次）
        migration_trigger_throttle(rs)
        migration_update_rates(rs, end_time)
```

### 16. Precopy 脏页迭代

Precopy 在 VM 运行时反复扫描脏页并发送。

**函数**：`ram_save_iterate()`（`ram.c:3255-3356`）

```
ram_save_iterate(f, opaque)
  │
  ├── 1. 同步位图
  │     migration_bitmap_sync_precopy(false)
  │
  ├── 2. 迭代发送脏页
  │     while (还有脏页 && 未超时 && 未达速率限制):
  │       │
  │       └── ram_find_and_save_block(rs)     [ram.c:2303-2365]
  │             │
  │             ├── 从上次位置继续扫描 block->bmap
  │             │     find_dirty_block() → find_next_bit(bmap, ...)
  │             │
  │             ├── 找到脏页 → 发送
  │             │     ram_save_host_page()
  │             │       └── ram_save_target_page()
  │             │             └── save_normal_page() 或 save_zero_page()
  │             │
  │             └── 更新扫描位置
  │                   clear bit in bmap
  │
  └── 3. 完成标志
        ram_counters 更新
```

**收敛条件**：
- 每轮结束时检查脏页率，如果脏页产生速度 > 发送速度，触发 throttle（限制 vCPU 执行时间）
- 最终轮（`ram_save_complete`）暂停 VM 后发送剩余脏页

### 17. Clear Bitmap 优化

为减少 `KVM_CLEAR_DIRTY_LOG` 的开销，QEMU 使用 `clear_bmap` 延迟清除。

**机制**（`ramblock.h:64-81`）：

```
clear_bmap: 每个 bit 覆盖 2^clear_bmap_shift 个页
  （粗粒度追踪"哪些区域需要向 KVM 发送清除请求"）

流程:
  1. migration_bitmap_sync() 获取 KVM 脏页
  2. clear_bmap_set(rb, page, npages) 标记需要清除的区域
  3. 下次同步前: clear_bmap_test_and_clear()
       → 仅对标记的区域调用 KVM_CLEAR_DIRTY_LOG
       → 避免对整个 slot 做清除
```

---

## 第五部分：NUMA 与内存热插拔

### 18. NUMA 拓扑

QEMU 通过 `-numa` 选项配置 Guest 的 NUMA 拓扑。

**核心数据结构**（`numa.h:38-52`）：

```c
typedef struct NodeInfo {
    uint64_t node_mem;              // 节点内存大小
    HostMemoryBackend *node_memdev; // 关联的内存后端
    bool present;                   // 节点是否存在
    bool has_cpu;                   // 是否有 CPU
    uint16_t initiator;             // 发起者节点
    uint8_t distance[MAX_NODES];    // 到其他节点的距离
} NodeInfo;

typedef struct NumaState {
    int num_nodes;                  // 节点总数
    NodeInfo nodes[MAX_NODES];      // 节点信息数组
    ...
} NumaState;
```

**解析流程**（`numa.c:63-170`）：

```
parse_numa_node(ms, node_opts, errp)
  │
  ├── 获取 node id（自动分配或显式指定）
  │
  ├── 处理 memdev=（推荐方式）
  │     object_resolve_path_type(memdev_id, TYPE_MEMORY_BACKEND, NULL)
  │     → node->node_memdev = backend
  │     → node->node_mem = backend->size
  │
  ├── 或处理 mem=（传统方式，与 memdev 互斥）
  │
  └── node->present = true
```

**初始化**（`numa_init_memdev_container`）：

```bash
# 典型用法
-object memory-backend-ram,id=mem0,size=2G
-object memory-backend-ram,id=mem1,size=2G
-numa node,nodeid=0,memdev=mem0,cpus=0-3
-numa node,nodeid=1,memdev=mem1,cpus=4-7
-numa dist,src=0,dst=1,val=20
```

### 19. 内存热插拔 (PC-DIMM/NVDIMM)

QEMU 通过 PC-DIMM 设备模拟内存热插拔。

#### PC-DIMM Realize

**函数**：`pc_dimm_realize()`（`pc-dimm.c:183-218`）

```
pc_dimm_realize(dev, errp)
  │
  ├── 验证 NUMA 节点
  │
  ├── 验证 hostmem 后端未被其他设备使用
  │     host_memory_backend_is_mapped(dimm->hostmem)
  │
  ├── 调用父类 realize
  │
  └── 标记后端已映射
        host_memory_backend_set_mapped(dimm->hostmem, true)
```

#### PC-DIMM Plug

**函数**：`pc_dimm_plug()`（`pc-dimm.c:75-86`）

```
pc_dimm_plug(dimm, machine)
  │
  ├── 将 DIMM 的 MemoryRegion 添加到设备内存区域
  │     memory_region_add_subregion(&machine->device_memory->mr, addr, mr)
  │
  ├── 注册 vmstate（迁移时保存/恢复）
  │
  └── 更新 DIMM 总大小（非 NVDIMM）
        machine->device_memory->dimm_size += size
```

#### NVDIMM

**文件**：`nvdimm.c:117-205`

NVDIMM 在 PC-DIMM 基础上增加：
- 持久化语义（`RAM_PMEM` 标志）
- NVDIMM 专用 MemoryRegion 别名（`nvdimm->nvdimm_mr`）
- ACPI NFIT 表生成

```bash
# 使用 NVDIMM
-object memory-backend-file,id=nv0,size=1G,mem-path=/dev/dax0.0,pmem=on,align=2M
-device nvdimm,memdev=nv0,id=nvdimm0,label-size=128K
```

### 20. virtio-mem 动态内存

virtio-mem 允许在运行时动态增减 Guest 内存，无需模拟物理 DIMM 插拔。

**结构体**（`virtio-mem.h:43-126`）：

```c
struct VirtIOMEM {
    VirtIODevice parent;
    MemoryRegion *mr;              // 底层 RAM 区域
    HostMemoryBackend *memdev;     // 内存后端
    unsigned long *bitmap;         // 页块状态位图（1=plugged）
    uint64_t addr;                 // 在 Guest 物理地址中的位置
    uint64_t size;                 // 当前已 plug 的大小
    uint64_t requested_size;       // 目标大小（Guest 驱动据此 plug/unplug）
    uint64_t block_size;           // 最小 plug/unplug 粒度
    uint32_t node;                 // NUMA 节点
    bool dynamic_memslots;         // 动态 memslot（减少 KVM 开销）
    QLIST_HEAD(, RamDiscardListener) rdl_list; // 丢弃监听器
};
```

#### 工作原理

```
1. 初始化时创建完整的 MemoryRegion，但不 plug 任何页

2. 通过 QMP 设置 requested_size:
     { "execute": "qom-set",
       "arguments": { "path": "/machine/peripheral/vm0",
                      "property": "requested-size", "value": 2147483648 } }

3. Guest 驱动收到配置变更通知
     → 驱动逐块 plug/unplug 到目标大小

4. plug 操作:
     virtio_mem_set_block_state(vmem, gpa, size, true)
       ├── bitmap_set(vmem->bitmap, ...)
       ├── vmem->size += size
       └── 通知 RamDiscardListener（VFIO 需要更新 IOMMU 映射）

5. unplug 操作:
     virtio_mem_set_block_state(vmem, gpa, size, false)
       ├── bitmap_clear(vmem->bitmap, ...)
       ├── ram_block_discard_range(rb, offset, size)
       │     // madvise(MADV_DONTNEED) 释放物理页
       └── vmem->size -= size
```

#### RamDiscardManager 接口

virtio-mem 实现 `RamDiscardManager` 接口，让 VFIO 等组件知道哪些页面有效：

```c
// virtio-mem.c class_init 中注册的方法:
rdmc->get_min_granularity   → block_size
rdmc->is_populated          → 检查 bitmap
rdmc->replay_populated      → 遍历已 plug 区域回调
rdmc->replay_discarded      → 遍历已 unplug 区域回调
rdmc->register_listener     → 注册变更通知
```

### 21. ARM virt 内存布局

ARM virt 机器的内存布局由 `virt_set_memmap()` 动态计算。

**函数**：`virt.c:2272-2335`

```
virt_set_memmap(vms, pa_bits)
  │
  ├── 基础内存映射（base_memmap 固定区域）
  │     VIRT_FLASH    : 0x0000_0000 (64MB)
  │     VIRT_GIC      : 0x0800_0000
  │     VIRT_UART     : 0x0900_0000
  │     VIRT_MMIO     : 0x0A00_0000
  │     VIRT_PCIE_MMIO: 0x1000_0000 (256MB)
  │     VIRT_MEM      : 0x4000_0000 (1GB 起始)  ← Guest RAM 起点
  │
  ├── RAM 区域
  │     VIRT_MEM.base = 1GiB
  │     VIRT_MEM.size = machine->ram_size
  │
  ├── 设备内存区域（热插拔用）
  │     device_memory_base = ROUND_UP(VIRT_MEM.base + ram_size, 1GiB)
  │     device_memory_size = 可配置
  │
  ├── 高地址 PCIE MMIO
  │     virt_set_high_memmap(...)
  │
  └── 初始化设备内存容器
        machine_memory_devices_init(ms, device_memory_base, device_memory_size)
```

**典型 4GB Guest 布局**：

```
0x0000_0000 ┌──────────────┐
            │ Flash (64MB) │
0x0800_0000 ├──────────────┤
            │ GIC/UART/... │
0x1000_0000 ├──────────────┤
            │ PCIE MMIO    │
0x4000_0000 ├──────────────┤  ← VIRT_MEM.base
            │              │
            │  Guest RAM   │
            │   (4 GB)     │
            │              │
0x1_4000_0000├─────────────┤  ← device_memory_base (GiB 对齐)
            │ Device Memory│  ← DIMM 热插拔区域
            │              │
0x1_8000_0000├─────────────┤
            │ High PCIE    │
            │ MMIO/ECAM    │
            └──────────────┘
```

---

## 附录

### A. 脏页追踪完整流程示例

**场景**：KVM 加速下，Guest 修改 RAM 页面，迁移发送该脏页

```
1. Guest vCPU 写入地址 0x8000_1000（RAM 页）
2. KVM 在 EPT/SPT 中检测到写入
     → 将 GFN 记录到 dirty ring（或 dirty bitmap）
3. QEMU reaper 线程定期收割:
     kvm_dirty_ring_reap_one() → 标记 slot->dirty_bmap
4. 迁移线程请求同步:
     migration_bitmap_sync()
       → memory_global_dirty_log_sync()
       → kvm_log_sync_global()
            → kvm_dirty_ring_flush()     // 踢 vCPU + reap
            → kvm_slot_sync_dirty_pages() // slot bmap → ram_list 全局位图
       → ramblock_sync_dirty_bitmap()    // 全局位图 → block->bmap
5. 迁移线程发送:
     ram_find_and_save_block()
       → find_next_bit(block->bmap, ...)  // 找到脏页
       → ram_save_host_page()             // 发送页面数据
       → clear bit in bmap               // 标记已发送
```

### B. 关键源文件索引

| 文件 | 内容 | 行数级 |
|------|------|--------|
| include/system/ramblock.h | RAMBlock 结构体定义 | ~290 行 |
| include/system/ramlist.h | RAMList、DirtyMemoryBlocks | ~87 行 |
| include/system/ram_addr.h | DIRTY_MEMORY_* 常量 | ~33 行 |
| system/physmem.c | ram_block_add、脏页位图操作 | ~3800 行 |
| system/memory.c | memory_region_set_dirty | ~3200 行 |
| accel/kvm/kvm-all.c | KVM dirty log/ring | ~3600 行 |
| accel/tcg/cputlb.c | TCG TLB_NOTDIRTY、notdirty_write | ~2800 行 |
| migration/ram.c | 迁移 RAM 保存、位图同步 | ~4500 行 |
| util/mmap-alloc.c | qemu_ram_mmap | ~300 行 |
| util/oslib-posix.c | qemu_prealloc_mem | ~700 行 |
| backends/hostmem.c | HostMemoryBackend 基类 | ~500 行 |
| backends/hostmem-file.c | 文件后备内存 | ~200 行 |
| backends/hostmem-memfd.c | memfd 内存 | ~150 行 |
| hw/core/numa.c | NUMA 拓扑解析 | ~600 行 |
| hw/mem/pc-dimm.c | PC-DIMM 热插拔 | ~300 行 |
| hw/virtio/virtio-mem.c | virtio-mem 动态内存 | ~1300 行 |
| hw/arm/virt.c | ARM virt 内存布局 | ~3500 行 |

### C. 关键设计总结

| 设计 | 说明 |
|------|------|
| **三客户端脏页位图** | VGA/CODE/MIGRATION 独立追踪，互不干扰 |
| **RCU 保护分块位图** | dirty_memory 可在 RCU 下无锁增长 |
| **per-RAMBlock 迁移位图** | bmap/clear_bmap/receivedmap 分离关注点 |
| **Dirty Ring > Dirty Log** | 增量消费、无需暂停 vCPU、更低延迟 |
| **内存后端 QOM** | 解耦分配策略与使用，支持 NUMA/huge pages/共享 |
| **两级 mmap** | 先 PROT_NONE 保留、再 MAP_FIXED 映射，保证对齐 |
| **virtio-mem 动态内存** | 比 DIMM 更灵活，支持页粒度 plug/unplug |

### D. 推荐深入方向

1. **Postcopy 迁移** — userfaultfd 按需页面传输
2. **VFIO 与脏页追踪** — IOMMU dirty tracking、vfio_listener 集成
3. **机密计算 (Confidential Computing)** — guest_memfd、RamBlockAttributes、TDX/SEV 内存加密
4. **内存气球 (Balloon)** — virtio-balloon 动态调整 Guest 内存
