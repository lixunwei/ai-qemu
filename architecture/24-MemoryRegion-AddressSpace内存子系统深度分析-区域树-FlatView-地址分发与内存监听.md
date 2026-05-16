# 24 - MemoryRegion/AddressSpace 内存子系统深度分析 — 区域树、FlatView、地址分发与内存监听

> **基于 QEMU 11.0.50 源码**，深入分析 QEMU 内存子系统的完整实现：
> MemoryRegion 区域树、MemoryRegionOps MMIO 回调、AddressSpace 地址空间、
> FlatView 扁平化、PhysPageEntry 基数树分发、MemoryListener 变更通知、
> RAMBlock 宿主内存管理、IOMMU 地址翻译以及脏页追踪。

---

## 目录

1. [MemoryRegion：内存区域核心结构](#1-memoryregion内存区域核心结构)
2. [MemoryRegionOps：MMIO 回调接口](#2-memoryregionopsmmio-回调接口)
3. [MemoryRegion 初始化函数族](#3-memoryregion-初始化函数族)
4. [AddressSpace：地址空间](#4-addressspace地址空间)
5. [FlatView：扁平化视图](#5-flatview扁平化视图)
6. [FlatView 生成与拓扑更新](#6-flatview-生成与拓扑更新)
7. [PhysPageEntry：基数树地址分发](#7-physpageentry基数树地址分发)
8. [地址翻译路径](#8-地址翻译路径)
9. [address_space_read/write：内存访问入口](#9-address_space_readwrite内存访问入口)
10. [MemoryListener：变更通知机制](#10-memorylistener变更通知机制)
11. [子区域管理与优先级](#11-子区域管理与优先级)
12. [RAMBlock：宿主内存管理](#12-ramblock宿主内存管理)
13. [IOMMUMemoryRegion：IOMMU 地址翻译](#13-ioммуmemoryregioniommu-地址翻译)
14. [脏页追踪](#14-脏页追踪)
15. [全局地址空间与系统内存](#15-全局地址空间与系统内存)
16. [数据流全景图](#16-数据流全景图)

---

## 1. MemoryRegion：内存区域核心结构

```c
// memory.h:820-867
struct MemoryRegion {
    Object parent_obj;                   // QOM 对象基类

    /* 缓存行对齐的热字段 */
    bool romd_mode;                      // ROM 设备可执行模式
    bool ram;                            // 是否 RAM 后端
    bool subpage;                        // 子页面标记
    bool readonly;                       // 只读（RAM 区域）
    bool nonvolatile;                    // 非易失性
    bool rom_device;                     // ROM 设备
    bool flush_coalesced_mmio;           // 刷新合并 MMIO
    bool lockless_io;                    // 无锁 I/O
    bool unmergeable;                    // 不可合并
    uint8_t dirty_log_mask;              // 脏页日志掩码
    bool is_iommu;                       // IOMMU 区域标记
    RAMBlock *ram_block;                 // RAM 块后端

    Object *owner;                       // 所有者对象
    DeviceState *dev;                    // 所有者设备（重入检查用）

    const MemoryRegionOps *ops;          // MMIO 读写回调
    void *opaque;                        // 回调上下文

    MemoryRegion *container;             // 父容器
    int mapped_via_alias;                // 通过别名映射

    Int128 size;                         // 区域大小
    hwaddr addr;                         // 在父容器中的偏移
    void (*destructor)(MemoryRegion *mr); // 析构函数
    uint64_t align;                      // 对齐要求

    bool terminates;                     // 是否终止分发（有 ops/ram）
    bool ram_device;                     // RAM 设备（device-mapped）
    bool enabled;                        // 是否启用

    MemoryRegion *alias;                 // 别名目标
    hwaddr alias_offset;                 // 别名偏移
    int32_t priority;                    // 重叠优先级

    QTAILQ_HEAD(, MemoryRegion) subregions;      // 子区域链表
    QTAILQ_ENTRY(MemoryRegion) subregions_link;  // 兄弟链表节点

    const char *name;                    // 区域名称
    unsigned ioeventfd_nb;               // ioeventfd 数量
    MemoryRegionIoeventfd *ioeventfds;   // ioeventfd 数组
    RamDiscardManager *rdm;              // RAM 丢弃管理器
    bool disable_reentrancy_guard;       // 禁用重入保护
};
```

**MemoryRegion 类型**：

| 类型 | `ram` | `ops` | `terminates` | `alias` | 说明 |
|------|-------|-------|-------------|---------|------|
| Container | ✗ | ✗ | ✗ | ✗ | 纯分组，无自身存储 |
| RAM | ✓ | ✗ | ✓ | ✗ | 宿主内存后端 |
| MMIO | ✗ | ✓ | ✓ | ✗ | 设备 I/O 回调 |
| Alias | ✗ | ✗ | ✗ | ✓ | 指向另一区域的窗口 |
| ROM | ✓ | ✗ | ✓ | ✗ | 只读 RAM |
| ROM Device | ✓ | ✓ | ✓ | ✗ | 可切换读模式 |

---

## 2. MemoryRegionOps：MMIO 回调接口

```c
// memory.h:300-360
struct MemoryRegionOps {
    // 基础回调（不带属性）
    uint64_t (*read)(void *opaque, hwaddr addr, unsigned size);
    void (*write)(void *opaque, hwaddr addr, uint64_t data, unsigned size);

    // 带属性回调（优先使用，可返回错误）
    MemTxResult (*read_with_attrs)(void *opaque, hwaddr addr,
                                   uint64_t *data, unsigned size,
                                   MemTxAttrs attrs);
    MemTxResult (*write_with_attrs)(void *opaque, hwaddr addr,
                                    uint64_t data, unsigned size,
                                    MemTxAttrs attrs);

    enum device_endian endianness;        // 字节序

    struct {                              // Guest 可见约束
        unsigned min_access_size;         // 最小访问大小
        unsigned max_access_size;         // 最大访问大小
        bool unaligned;                   // 允许非对齐
        bool (*accepts)(...);             // 访问验证回调
    } valid;

    struct {                              // 内部实现约束
        unsigned min_access_size;         // 实际最小实现
        unsigned max_access_size;         // 实际最大实现
        bool unaligned;                   // 实现支持非对齐
    } impl;
};
```

**valid vs impl**：
- `valid`：Guest 发起超出范围的访问将触发 Machine Check
- `impl`：QEMU 自动将大/小/非对齐访问拆分/合并为 impl 范围内的访问

---

## 3. MemoryRegion 初始化函数族

```c
// memory.c:1246-1253 — 容器（无后端存储）
void memory_region_init(MemoryRegion *mr, Object *owner,
                        const char *name, uint64_t size)
{
    object_initialize(mr, sizeof(*mr), TYPE_MEMORY_REGION);
    memory_region_do_init(mr, owner, name, size);
}

// memory.c:1572-1579 — MMIO（设备 I/O 回调）
void memory_region_init_io(MemoryRegion *mr, Object *owner,
                           const MemoryRegionOps *ops, void *opaque,
                           const char *name, uint64_t size)
// 设置 mr->ops, mr->opaque, mr->terminates = true

// memory.c:3661-3671 — RAM（宿主内存后端）
void memory_region_init_ram(MemoryRegion *mr, Object *owner,
                            const char *name, uint64_t size, Error **errp)
// 分配 RAMBlock，mr->ram = true, mr->terminates = true

// memory.c:1685-1692 — 别名（另一区域的窗口）
void memory_region_init_alias(MemoryRegion *mr, Object *owner,
                              const char *name, MemoryRegion *orig,
                              hwaddr offset, uint64_t size)
// mr->alias = orig, mr->alias_offset = offset

// memory.c:3685-3696 — ROM（只读 RAM）
void memory_region_init_rom(MemoryRegion *mr, Object *owner,
                            const char *name, uint64_t size, Error **errp)
// mr->ram = true, mr->readonly = true
```

---

## 4. AddressSpace：地址空间

```c
// memory.h:1157-1186
struct AddressSpace {
    struct rcu_head rcu;
    char *name;                          // 名称
    MemoryRegion *root;                  // 根 MemoryRegion

    struct FlatView *current_map;        // 当前扁平视图（RCU 访问）

    int ioeventfd_nb;                    // ioeventfd 数量
    struct MemoryRegionIoeventfd *ioeventfds;

    QTAILQ_HEAD(, MemoryListener) listeners;           // 监听器列表
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;    // 全局链表

    size_t max_bounce_buffer_size;       // DMA bounce buffer 上限
    size_t bounce_buffer_size;           // 当前 bounce buffer 用量
    QemuMutex map_client_list_lock;
    QLIST_HEAD(, AddressSpaceMapClient) map_client_list;
};
```

### 初始化

```c
// memory.c:3163-3178
void address_space_init(AddressSpace *as, MemoryRegion *root, const char *name)
{
    memory_region_ref(root);             // 增加根区域引用
    as->root = root;
    as->current_map = NULL;              // 稍后由 update_topology 设置
    QTAILQ_INIT(&as->listeners);
    QTAILQ_INSERT_TAIL(&address_spaces, as, address_spaces_link);
    as->name = g_strdup(name);
    address_space_update_topology(as);   // 生成初始 FlatView
    address_space_update_ioeventfds(as);
}
```

---

## 5. FlatView：扁平化视图

MemoryRegion 构成的树形结构需要"扁平化"为一维有序范围数组，用于高效地址分发。

### 5.1 FlatRange — 单个扁平范围

```c
// memory.c:222-231
struct FlatRange {
    MemoryRegion *mr;              // 所属区域
    hwaddr offset_in_region;       // 区域内偏移
    AddrRange addr;                // 绝对地址范围
    uint8_t dirty_log_mask;        // 脏页日志
    bool romd_mode;                // ROM 设备模式
    bool readonly;                 // 只读
    bool nonvolatile;              // 非易失
    bool unmergeable;              // 不可合并
};
```

### 5.2 FlatView 结构

```c
// memory.h:1194-1202
struct FlatView {
    struct rcu_head rcu;
    unsigned ref;                           // 引用计数
    FlatRange *ranges;                      // 有序范围数组
    unsigned nr;                            // 范围数量
    unsigned nr_allocated;                  // 已分配数量
    struct AddressSpaceDispatch *dispatch;  // 基数树分发表
    MemoryRegion *root;                     // 根区域
};
```

### 5.3 MemoryRegionSection — 区域片段

```c
// memory.h:105-114
struct MemoryRegionSection {
    Int128 size;                            // 片段大小
    MemoryRegion *mr;                       // 所属区域
    FlatView *fv;                           // 所属 FlatView
    hwaddr offset_within_region;            // 区域内偏移
    hwaddr offset_within_address_space;     // 地址空间内偏移
    bool readonly;
    bool nonvolatile;
    bool unmergeable;
};
```

---

## 6. FlatView 生成与拓扑更新

### 6.1 生成流程

```c
// memory.c:747-772
static FlatView *generate_memory_topology(MemoryRegion *mr)
{
    // 1. 创建新 FlatView
    view = flatview_new(mr);

    // 2. 递归渲染 MR 树为扁平范围
    render_memory_region(view, mr, int128_zero(),
                         addrrange_make(int128_zero(), int128_2_64()),
                         false, false, false);

    // 3. 合并相邻可合并范围
    flatview_simplify(view);

    // 4. 构建基数树分发表
    view->dispatch = address_space_dispatch_new(view);
    for (i = 0; i < view->nr; i++) {
        MemoryRegionSection mrs = section_from_flat_range(&view->ranges[i], view);
        flatview_add_to_dispatch(view, &mrs);
    }
    address_space_dispatch_compact(view->dispatch);

    return view;
}
```

### 6.2 render_memory_region — 递归渲染

```c
// memory.c:596-686
static void render_memory_region(FlatView *view, MemoryRegion *mr, ...)
{
    // 递归处理子区域和别名
    // 高优先级区域覆盖低优先级
    // 产出一组不重叠的 FlatRange
}
```

### 6.3 事务提交触发更新

```c
// memory.c:1138-1171
void memory_region_transaction_begin(void)
{
    ++memory_region_transaction_depth;
}

void memory_region_transaction_commit(void)
{
    --memory_region_transaction_depth;
    if (!memory_region_transaction_depth) {
        if (memory_region_update_pending) {
            // ① 重新生成所有 FlatView
            flatviews_reset();

            // ② 通知所有 Listener：begin
            MEMORY_LISTENER_CALL_GLOBAL(begin, Forward);

            // ③ 逐个 AddressSpace 更新 FlatView + ioeventfd
            QTAILQ_FOREACH(as, &address_spaces, address_spaces_link) {
                address_space_set_flatview(as);
                address_space_update_ioeventfds(as);
            }

            // ④ 通知所有 Listener：commit
            MEMORY_LISTENER_CALL_GLOBAL(commit, Forward);
        }
    }
}
```

**关键设计**：事务机制确保多个 MR 变更作为一个原子操作应用，避免中间状态被观察到。

---

## 7. PhysPageEntry：基数树地址分发

```c
// physmem.c:108-147
struct PhysPageEntry {
    uint32_t skip : 6;     // 跳过位数（0 = 叶节点）
    uint32_t ptr : 26;     // 索引到 sections[]（叶）或 nodes[]（内部节点）
};

// L2 页表参数
#define ADDR_SPACE_BITS 64
#define P_L2_BITS 9
#define P_L2_SIZE (1 << P_L2_BITS)    // 512 个条目/节点
#define P_L2_LEVELS (((64 - TARGET_PAGE_BITS - 1) / 9) + 1)

typedef PhysPageEntry Node[P_L2_SIZE]; // 512-entry 页表节点

struct PhysPageMap {
    unsigned sections_nb;            // section 数量
    unsigned nodes_nb;               // 节点数量
    Node *nodes;                     // 节点数组
    MemoryRegionSection *sections;   // section 数组
};

struct AddressSpaceDispatch {
    MemoryRegionSection *mru_section;  // MRU 缓存
    PhysPageEntry phys_map;            // 根页表条目
    PhysPageMap map;                   // 页表和 section 存储
};
```

**工作原理**：多级基数树将 64 位地址空间划分为页面大小的块，每个叶节点指向一个 `MemoryRegionSection`。`skip` 字段允许跳过空白层级，实现稀疏地址空间的高效表示。

---

## 8. 地址翻译路径

### 8.1 内部分发

```c
// physmem.c:364-398
static MemoryRegionSection *
address_space_translate_internal(AddressSpaceDispatch *d, hwaddr addr,
                                hwaddr *xlat, hwaddr *plen, bool resolve_subpage)
{
    // 1. 在基数树中查找
    section = address_space_lookup_region(d, addr, resolve_subpage);

    // 2. 计算地址空间内偏移
    addr -= section->offset_within_address_space;

    // 3. 计算区域内偏移
    *xlat = addr + section->offset_within_region;

    // 4. 对 RAM 区域限制长度到区域边界
    if (memory_region_is_ram(mr)) {
        diff = int128_sub(section->size, int128_make64(addr));
        *plen = int128_get64(int128_min(diff, int128_make64(*plen)));
    }
    return section;
}
```

### 8.2 IOMMU 感知的翻译

```c
// physmem.c:493-527
static MemoryRegionSection flatview_do_translate(FlatView *fv,
                                                 hwaddr addr, ...)
{
    // 1. 内部基数树查找
    section = address_space_translate_internal(
            flatview_to_dispatch(fv), addr, xlat, plen_out, is_mmio);

    // 2. 检查是否 IOMMU 区域
    iommu_mr = memory_region_get_iommu(section->mr);
    if (unlikely(iommu_mr)) {
        // IOMMU 二次翻译
        return address_space_translate_iommu(iommu_mr, xlat, ...);
    }

    return *section;
}
```

### 8.3 顶层翻译

```c
// physmem.c:568-589
MemoryRegion *flatview_translate(FlatView *fv, hwaddr addr,
                                 hwaddr *xlat, hwaddr *plen, ...)
{
    section = flatview_do_translate(fv, addr, xlat, plen, ...);
    return section.mr;
}
```

---

## 9. address_space_read/write：内存访问入口

### 9.1 写入路径

```c
// physmem.c:3311-3326
static MemTxResult flatview_write(FlatView *fv, hwaddr addr, MemTxAttrs attrs,
                                  const void *buf, hwaddr len)
{
    while (len > 0) {
        // 1. 翻译地址 → MemoryRegion + 偏移
        section = flatview_do_translate(fv, addr, &addr1, &l, ...);

        if (memory_access_is_direct(mr, true, attrs)) {
            // 2a. 直接访问：memcpy 到宿主内存
            ptr = qemu_map_ram_ptr(mr->ram_block, addr1);
            memcpy(ptr, buf, l);
            invalidate_and_set_dirty(mr, addr1, l);
        } else {
            // 2b. MMIO：调用 ops->write 回调
            memory_region_dispatch_write(mr, addr1, ...);
        }
    }
}
```

### 9.2 读取路径

```c
// physmem.c:3402-3417
static MemTxResult flatview_read(FlatView *fv, hwaddr addr, ...)
{
    // 类似写入路径
    // 直接访问：memcpy 从宿主内存
    // MMIO：调用 ops->read 回调
}
```

### 9.3 DMA 映射

```c
// physmem.c:3705-3807
void *address_space_map(AddressSpace *as, hwaddr addr, hwaddr *plen,
                        bool is_write, MemTxAttrs attrs)
{
    // 1. 翻译地址
    // 2. 如果是 RAM → 直接返回宿主指针
    // 3. 如果是 MMIO → 分配 bounce buffer
    return ptr;
}

void address_space_unmap(AddressSpace *as, void *buffer, hwaddr len,
                         bool is_write, hwaddr access_len)
{
    // 如果使用了 bounce buffer：
    //   写模式 → flush bounce buffer → address_space_write
    //   释放 bounce buffer
}
```

---

## 10. MemoryListener：变更通知机制

### 10.1 MemoryListener 结构

```c
// memory.h:889-1128（部分）
struct MemoryListener {
    void (*begin)(MemoryListener *listener);              // 事务开始
    void (*commit)(MemoryListener *listener);             // 事务提交

    void (*region_add)(MemoryListener *listener,
                       MemoryRegionSection *section);     // 新区域
    void (*region_del)(MemoryListener *listener,
                       MemoryRegionSection *section);     // 删除区域
    void (*region_nop)(MemoryListener *listener,
                       MemoryRegionSection *section);     // 未变化区域

    void (*log_start)(MemoryListener *listener,
                      MemoryRegionSection *section, ...); // 脏页日志开始
    void (*log_stop)(MemoryListener *listener,
                     MemoryRegionSection *section, ...);  // 脏页日志停止
    void (*log_sync)(MemoryListener *listener,
                     MemoryRegionSection *section);       // 脏页同步
    void (*log_sync_global)(MemoryListener *listener);    // 全局脏页同步
    void (*log_clear)(MemoryListener *listener,
                      MemoryRegionSection *section);      // 清除脏位图

    void (*log_global_start)(MemoryListener *listener);   // 全局日志开始
    void (*log_global_stop)(MemoryListener *listener);    // 全局日志停止

    void (*eventfd_add)(MemoryListener *listener,
                        MemoryRegionSection *section, ...); // ioeventfd 添加
    void (*eventfd_del)(MemoryListener *listener,
                        MemoryRegionSection *section, ...); // ioeventfd 删除

    void (*coalesced_io_add)(MemoryListener *listener, ...);
    void (*coalesced_io_del)(MemoryListener *listener, ...);

    unsigned priority;                    // 优先级
    const char *name;                     // 名称
};
```

### 10.2 监听器优先级

```c
// memory.h:879-881
#define MEMORY_LISTENER_PRIORITY_MIN            0
#define MEMORY_LISTENER_PRIORITY_ACCEL          10
```

### 10.3 注册与通知流程

```c
// 注册：memory.c:3101-3138
void memory_listener_register(MemoryListener *listener, AddressSpace *as)
// 按优先级插入 as->listeners，然后通知所有现有区域

// 通知触发：memory_region_transaction_commit()
// → MEMORY_LISTENER_CALL_GLOBAL(begin)
// → address_space_set_flatview() 对比新旧 FlatView
//   → 新增区域 → listener->region_add()
//   → 删除区域 → listener->region_del()
//   → 未变区域 → listener->region_nop()
// → MEMORY_LISTENER_CALL_GLOBAL(commit)
```

### 10.4 KVM 内存监听器

KVM 通过 MemoryListener 将 QEMU 的内存映射同步到内核 KVM memslot：

```c
// kvm-all.c — KVM MemoryListener 回调
// region_add  → kvm_set_user_memory_region() 创建 memslot
// region_del  → kvm_set_user_memory_region() 删除 memslot
// log_start   → 设置 KVM_MEM_LOG_DIRTY_PAGES
// log_stop    → 清除脏页日志标志
// log_sync    → KVM_GET_DIRTY_LOG / dirty ring 同步
```

---

## 11. 子区域管理与优先级

### 11.1 添加子区域

```c
// memory.c:2609-2624
void memory_region_add_subregion(MemoryRegion *mr, hwaddr offset,
                                 MemoryRegion *subregion)
{
    subregion->priority = 0;   // 默认优先级
    memory_region_add_subregion_common(mr, offset, subregion);
}

void memory_region_add_subregion_overlap(MemoryRegion *mr, hwaddr offset,
                                         MemoryRegion *subregion, int priority)
{
    subregion->priority = priority;  // 显式优先级
    memory_region_add_subregion_common(mr, offset, subregion);
}
```

### 11.2 优先级排序

```c
// memory.c:2571-2592
static void memory_region_update_container_subregions(MemoryRegion *subregion)
{
    MemoryRegion *mr = subregion->container;

    memory_region_transaction_begin();

    // 按优先级降序插入（高优先级在前）
    QTAILQ_FOREACH(other, &mr->subregions, subregions_link) {
        if (subregion->priority >= other->priority) {
            QTAILQ_INSERT_BEFORE(other, subregion, subregions_link);
            goto done;
        }
    }
    QTAILQ_INSERT_TAIL(&mr->subregions, subregion, subregions_link);

done:
    memory_region_update_pending |= mr->enabled && subregion->enabled;
    memory_region_transaction_commit();
}
```

**优先级规则**：在 FlatView 渲染时，高优先级区域覆盖低优先级区域。重叠区域的高优先级部分"遮挡"低优先级部分。

---

## 12. RAMBlock：宿主内存管理

```c
// ramblock.h:25-92
struct RAMBlock {
    struct rcu_head rcu;
    struct MemoryRegion *mr;           // 关联的 MemoryRegion
    uint8_t *host;                     // 宿主虚拟地址（mmap）

    ram_addr_t offset;                 // 全局 RAM 偏移
    ram_addr_t used_length;            // 已用长度
    ram_addr_t max_length;             // 最大长度（支持热扩展）
    uint32_t flags;                    // 标志位

    char idstr[256];                   // 迁移标识符

    int fd;                            // 后端文件描述符（-1 = 匿名）
    uint64_t fd_offset;                // 文件偏移
    int guest_memfd;                   // 机密 VM guest memfd
    RamBlockAttributes *attributes;    // 属性（机密 VM）
    size_t page_size;                  // 页面大小

    /* 迁移用位图 */
    unsigned long *bmap;               // 脏页位图
    unsigned long *file_bmap;          // 文件存在位图
    unsigned long *receivedmap;        // 已接收位图
    unsigned long *clear_bmap;         // 待清除脏位图
    uint8_t clear_bmap_shift;          // 清除位图粒度

    ram_addr_t postcopy_length;        // postcopy 长度
};
```

**内存分配**：RAMBlock 通过 `qemu_ram_alloc_internal()` → `mmap()` 分配宿主内存。支持匿名映射、文件后端（hugepages）和 memfd。

---

## 13. IOMMUMemoryRegion：IOMMU 地址翻译

```c
// memory.h:869-874
struct IOMMUMemoryRegion {
    MemoryRegion parent_obj;
    QLIST_HEAD(, IOMMUNotifier) iommu_notify;    // IOMMU 通知器列表
    IOMMUNotifierFlag iommu_notify_flags;
};
```

### IOMMUMemoryRegionClass

```c
// memory.h:401-543
// 关键虚函数：
// translate(iommu_mr, addr, flag, iommu_idx) → IOMMUTLBEntry
// get_min_page_size(iommu_mr) → 最小页面大小
// notify_flag_changed(iommu_mr, old_flags, new_flags)
// replay(iommu_mr, notifier) → 重放现有映射
// get_attr(iommu_mr, attr, data)
// attrs_to_index(attrs) → IOMMU 索引
// num_indexes(iommu_mr) → 索引数量
```

### IOMMU 翻译流程

```c
// physmem.c:422-471
// address_space_translate_iommu()
// 1. 调用 iommu_mr->translate(addr) 获取 IOMMUTLBEntry
// 2. 如果有 target_as → 递归翻译
// 3. 支持多级 IOMMU 嵌套
```

**典型实现**：ARM SMMUv3（`hw/arm/smmuv3.c`）、Intel VT-d（`hw/i386/intel_iommu.c`）。

---

## 14. 脏页追踪

### 14.1 设置脏页

```c
// physmem.c — 脏页标记
// cpu_physical_memory_set_dirty_range(start, length, dirty_log_mask)
// 在 RAM 写入后调用，标记脏页

// physmem.c:3124-3148
static inline void invalidate_and_set_dirty(MemoryRegion *mr,
                                            hwaddr addr, hwaddr length)
{
    // 使 TB 缓存失效（自修改代码检测）
    // 标记脏页（供迁移和 VGA 使用）
}
```

### 14.2 脏页同步

```c
// KVM 路径：
// kvm_slot_sync_dirty_pages()
//   → KVM_GET_DIRTY_LOG ioctl
//   → cpu_physical_memory_set_dirty_lebitmap()

// Dirty Ring 路径：
// kvm_dirty_ring_mark_page()
//   → 直接标记特定页面为脏
```

### 14.3 用途

- **Live Migration**：追踪脏页进行增量传输
- **VGA 显示**：追踪帧缓冲区变化进行重绘
- **VFIO**：追踪设备 DMA 脏页

---

## 15. 全局地址空间与系统内存

```c
// physmem.c:100-104
static MemoryRegion *system_memory;    // 全局系统内存根区域
static MemoryRegion *system_io;        // 全局 I/O 端口根区域

AddressSpace address_space_io;         // I/O 端口地址空间
AddressSpace address_space_memory;     // 物理内存地址空间
```

### 初始化

```c
// physmem.c:3101-3112
static void memory_map_init(void)
{
    system_memory = g_malloc(sizeof(*system_memory));
    memory_region_init(system_memory, NULL, "system", UINT64_MAX);
    address_space_init(&address_space_memory, system_memory, "memory");

    system_io = g_malloc(sizeof(*system_io));
    memory_region_init_io(system_io, NULL, &unassigned_io_ops, NULL, "io",
                          65536);
    address_space_init(&address_space_io, system_io, "I/O");
}
```

**典型内存布局**（ARM virt 机器）：

```
system_memory (container, 0 ~ UINT64_MAX)
  ├── ram (RAM, 0x4000_0000 ~ 0x4000_0000 + ram_size)
  ├── flash0 (ROM, 0x0000_0000 ~ 0x0400_0000)
  ├── gic_dist (MMIO, 0x0800_0000 ~ 0x0801_0000)
  ├── gic_redist (MMIO, 0x080A_0000 ~ ...)
  ├── uart0 (MMIO, 0x0900_0000 ~ 0x0900_1000)
  ├── pcie_mmio (alias → pcie_mmio_window)
  └── pcie_ecam (MMIO, 0x4010_0000 ~ ...)
```

---

## 16. 数据流全景图

### 16.1 地址翻译全景

```
Guest 内存访问（GPA）
  │
  ▼
AddressSpace (address_space_memory)
  │
  ▼
FlatView (current_map，RCU 读取)
  │
  ▼
AddressSpaceDispatch (基数树)
  │
  ├──→ PhysPageEntry.skip > 0：内部节点 → 递归下降
  └──→ PhysPageEntry.skip == 0：叶节点 → MemoryRegionSection
         │
         ├── RAM → qemu_map_ram_ptr() → 宿主内存 memcpy
         ├── MMIO → mr->ops->read/write() → 设备回调
         └── IOMMU → address_space_translate_iommu()
                       → iommu_mr->translate() → 二次翻译
```

### 16.2 拓扑更新流程

```
memory_region_add_subregion()
  │
  ▼
memory_region_transaction_begin()
  ├── 设置 memory_region_update_pending = true
  └── memory_region_transaction_commit()
        │
        ▼
      flatviews_reset()
        │ 对每个 AddressSpace 根 MR：
        │   generate_memory_topology(mr)
        │     ├── render_memory_region() 递归渲染
        │     ├── flatview_simplify() 合并
        │     └── 构建 dispatch 基数树
        ▼
      MEMORY_LISTENER_CALL_GLOBAL(begin)
        │
      对比新旧 FlatView:
        ├── 新增 section → listener->region_add()
        │     └── KVM: kvm_set_user_memory_region() → 创建 memslot
        ├── 删除 section → listener->region_del()
        │     └── KVM: kvm_set_user_memory_region() → 删除 memslot
        └── 不变 section → listener->region_nop()
        │
      MEMORY_LISTENER_CALL_GLOBAL(commit)
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/system/memory.h` | ~2900 | MemoryRegionSection (105-114)、MemoryRegionOps (300-360)、IOMMUMemoryRegionClass (401-543)、MemoryRegion (820-867)、IOMMUMemoryRegion (869-874)、MemoryListener (889-1128)、AddressSpace (1157-1186)、FlatView (1194-1202) |
| `system/memory.c` | ~3700 | FlatRange (222-231)、render_memory_region (596-686)、generate_memory_topology (747-772)、memory_region_transaction_begin/commit (1138-1171)、memory_region_init (1246-1253)、memory_region_init_io (1572)、memory_region_init_alias (1685)、memory_region_init_iommu (1694)、memory_region_add_subregion (2609-2624)、memory_region_set_enabled (2648-2695)、listener_add_address_space (3000-3099)、memory_listener_register (3101-3138)、address_space_init (3163-3178)、memory_region_init_ram (3661) |
| `system/physmem.c` | ~3900 | system_memory/system_io (100-101)、address_space_memory/io (103-104)、PhysPageEntry (110-115)、PhysPageMap (129-138)、AddressSpaceDispatch (140-147)、address_space_translate_internal (364-398)、flatview_do_translate (493-527)、flatview_translate (568-589)、memory_map_init (3101-3112)、flatview_write (3311-3326)、flatview_read (3402-3417)、address_space_map (3705-3767)、address_space_unmap (3769-3807) |
| `include/system/ramblock.h` | ~130 | RAMBlock (25-92) |
