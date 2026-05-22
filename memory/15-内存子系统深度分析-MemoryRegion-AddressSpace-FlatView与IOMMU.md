# 内存子系统深度分析 — MemoryRegion、AddressSpace、FlatView 与 IOMMU

> 基于 QEMU 11.0.50 源码分析，聚焦内存子系统全栈：
> MemoryRegion 树形层次、MemoryRegionOps 设备 MMIO 回调、AddressSpace 地址空间抽象、
> FlatView 拓扑展平与地址解析、MemoryListener 拓扑变更通知、IOMMU 翻译路径、
> TCG/KVM 两种加速器的内存访问路径、RAMBlock 物理内存后端、DMA 映射与 Bounce Buffer。

---

## 目录

1. [内存子系统全景](#1-内存子系统全景)
2. [MemoryRegion 核心](#2-memoryregion-核心)
3. [MemoryRegionOps 设备回调](#3-memoryregionops-设备回调)
4. [AddressSpace 地址空间](#4-addressspace-地址空间)
5. [FlatView 拓扑展平](#5-flatview-拓扑展平)
6. [地址解析全路径](#6-地址解析全路径)
7. [MemoryListener 拓扑变更通知](#7-memorylistener-拓扑变更通知)
8. [IOMMU 翻译](#8-iommu-翻译)
9. [TCG 内存访问路径](#9-tcg-内存访问路径)
10. [KVM 内存路径](#10-kvm-内存路径)
11. [RAMBlock 与 ROM](#11-ramblock-与-rom)
12. [总结](#12-总结)

---

## 1. 内存子系统全景

QEMU 内存子系统是连接 Guest 物理地址与 Host 物理内存/设备 MMIO 的核心基础设施：

```
Guest 物理地址 (GPA)
     │
     ▼
┌──────────────────────────────────────────────────────┐
│  AddressSpace                                         │
│  include/system/memory.h:1157-1186                    │
│  ├── root: MemoryRegion*（根 MR）                     │
│  ├── current_map: FlatView*（展平视图，RCU 保护）      │
│  └── listeners: MemoryListener 链表                   │
│                                                       │
│  ┌─ FlatView ────────────────────────────────────┐    │
│  │  memory.h:1194-1202                           │    │
│  │  ├── ranges[]: FlatRange 有序数组              │    │
│  │  ├── dispatch: AddressSpaceDispatch*           │    │
│  │  └── root: MemoryRegion*                       │    │
│  └───────────────────────────────────────────────┘    │
│                                                       │
│  地址解析:                                            │
│  flatview_do_translate() → address_space_translate_internal() │
│  → MemoryRegionSection → MR.ops->read/write           │
│                   或 → RAMBlock.host + offset (直接访问)│
└──────────────────────────────────────────────────────┘

MemoryRegion 树形层次:
     system_memory (container, 0~2^64)
     ├── ram (RAM, 0~2GB, terminates=true)
     │   └── ram_block → host 内存
     ├── pci_io (alias → io, offset=0)
     ├── gic_dist (MMIO, 0x8000000~0x10000)
     │   └── ops = gicv3_dist_ops
     ├── uart (MMIO, 0x9000000~0x1000)
     │   └── ops = pl011_ops
     └── pcie_mmio (container)
         ├── bar0 (MMIO, overlap, priority=1)
         └── bar1 (RAM, overlap, priority=0)
```

---

## 2. MemoryRegion 核心

### 结构体定义

```c
// include/system/memory.h:820-867
struct MemoryRegion {
    Object parent_obj;

    // === 缓存行对齐的热字段 ===
    bool romd_mode;              // ROM 设备读模式
    bool ram;                    // 是否为 RAM
    bool subpage;                // 子页面区域
    bool readonly;               // RAM 只读标志
    bool nonvolatile;            // 非易失性
    bool rom_device;             // ROM 设备
    bool flush_coalesced_mmio;   // 刷新合并 MMIO
    bool lockless_io;            // 无锁 IO
    bool unmergeable;            // 不可合并
    uint8_t dirty_log_mask;      // 脏页跟踪掩码
    bool is_iommu;               // 是否为 IOMMU 区域
    RAMBlock *ram_block;         // RAM 后端（仅 RAM/ROM）
    Object *owner;               // 所有者对象
    DeviceState *dev;            // 所有者设备（重入检查热路径）

    // === 核心配置 ===
    const MemoryRegionOps *ops;  // MMIO 读写回调
    void *opaque;                // 回调私有数据
    MemoryRegion *container;     // 父容器
    int mapped_via_alias;        // 通过 alias 映射
    Int128 size;                 // 区域大小
    hwaddr addr;                 // 在容器中的偏移
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;              // 对齐要求
    bool terminates;             // 终止解析（有 ops 或 ram_block）
    bool ram_device;             // RAM 设备
    bool enabled;                // 是否启用

    // === 树结构 ===
    MemoryRegion *alias;         // alias 目标
    hwaddr alias_offset;         // alias 偏移
    int32_t priority;            // 重叠优先级
    QTAILQ_HEAD(, MemoryRegion) subregions;       // 子区域链表
    QTAILQ_ENTRY(MemoryRegion) subregions_link;   // 兄弟链接

    // === 其他 ===
    QTAILQ_HEAD(, CoalescedMemoryRange) coalesced; // 合并范围
    const char *name;            // 名称（调试）
    unsigned ioeventfd_nb;       // ioeventfd 数量
    MemoryRegionIoeventfd *ioeventfds;
    RamDiscardManager *rdm;      // RAM 丢弃管理
    bool disable_reentrancy_guard;
};
```

### MR 类型与初始化

| 类型 | 初始化函数 | 位置 | 关键字段 |
|------|-----------|------|----------|
| **MMIO** | `memory_region_init_io()` | memory.c:1572-1579 | `ops`, `opaque`, `terminates=true` |
| **RAM** | `memory_region_init_ram()` | memory.c:3661-3671 | `ram=true`, `ram_block`, `terminates=true` |
| **ROM** | `memory_region_init_rom()` | memory.c:3685-3695 | `ram=true`, `readonly=true` |
| **ROM 设备** | `memory_region_init_rom_device()` | memory.c:3698-3713 | `rom_device=true`, 有 `ops` 和 `ram_block` |
| **Alias** | `memory_region_init_alias()` | memory.c:1685-1692 | `alias`, `alias_offset` |
| **Container** | `memory_region_init()` | 基础初始化 | 无 `ops`/`ram_block`，`terminates=false` |
| **IOMMU** | `memory_region_init_iommu()` | memory.c:1694-1700 | `is_iommu=true` |

### MR 树操作

```c
// 添加子区域
// memory.c:2609-2614
void memory_region_add_subregion(MemoryRegion *mr, hwaddr offset,
                                  MemoryRegion *subregion) {
    subregion->priority = 0;
    memory_region_add_subregion_common(mr, offset, subregion);
}

// 添加重叠子区域（带优先级）
// memory.c:2617-2624
void memory_region_add_subregion_overlap(MemoryRegion *mr, hwaddr offset,
                                          MemoryRegion *subregion, int priority) {
    subregion->priority = priority;
    memory_region_add_subregion_common(mr, offset, subregion);
}

// 启用/禁用
// memory.c:2648-2657
void memory_region_set_enabled(MemoryRegion *mr, bool enabled) {
    memory_region_transaction_begin();
    mr->enabled = enabled;
    memory_region_update_pending = true;
    memory_region_transaction_commit();    // 触发拓扑更新
}

// 修改地址
// memory.c:2689-2695
void memory_region_set_address(MemoryRegion *mr, hwaddr addr) {
    if (addr != mr->addr) {
        mr->addr = addr;
        memory_region_readd_subregion(mr);  // 重新插入以保持排序
    }
}
```

---

## 3. MemoryRegionOps 设备回调

```c
// include/system/memory.h:300-360
struct MemoryRegionOps {
    // 基本回调
    uint64_t (*read)(void *opaque, hwaddr addr, unsigned size);
    void (*write)(void *opaque, hwaddr addr, uint64_t data, unsigned size);

    // 带属性的高级回调（返回 MemTxResult）
    MemTxResult (*read_with_attrs)(void *opaque, hwaddr addr,
                                    uint64_t *data, unsigned size, MemTxAttrs attrs);
    MemTxResult (*write_with_attrs)(void *opaque, hwaddr addr,
                                     uint64_t data, unsigned size, MemTxAttrs attrs);

    enum device_endian endianness;

    // Guest 可见约束
    struct {
        unsigned min_access_size;     // 最小访问大小
        unsigned max_access_size;     // 最大访问大小
        bool unaligned;               // 是否允许非对齐
        bool (*accepts)(void *opaque, hwaddr addr,
                        unsigned size, bool is_write, MemTxAttrs attrs);
    } valid;

    // 内部实现约束
    struct {
        unsigned min_access_size;     // 实现的最小大小
        unsigned max_access_size;     // 实现的最大大小
        bool unaligned;               // 实现是否支持非对齐
    } impl;
};
```

### MMIO 访问调度

```c
// memory.c:430-514
// 读访问：
static MemTxResult memory_region_read_accessor(MemoryRegion *mr,
    hwaddr addr, uint64_t *value, unsigned size, ...) {
    tmp = mr->ops->read(mr->opaque, addr, size);     // :440
    return MEMTX_OK;
}

// 带属性的读：
static MemTxResult memory_region_read_with_attrs_accessor(...) {
    r = mr->ops->read_with_attrs(mr->opaque, addr, &tmp, size, attrs); // :463
}

// 写访问：
static MemTxResult memory_region_write_accessor(...) {
    mr->ops->write(mr->opaque, addr, tmp, size);     // :492
}

// access_with_adjusted_size() 处理大小拆分和对齐：
// memory.c:516+ — 将大访问拆分为 impl.min/max 范围内的多次小访问
```

---

## 4. AddressSpace 地址空间

```c
// include/system/memory.h:1157-1186
struct AddressSpace {
    struct rcu_head rcu;
    char *name;
    MemoryRegion *root;                      // 根 MR

    struct FlatView *current_map;            // RCU 保护的展平视图

    int ioeventfd_nb;
    int ioeventfd_notifiers;
    struct MemoryRegionIoeventfd *ioeventfds;
    QTAILQ_HEAD(, MemoryListener) listeners; // 监听器链表
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;

    size_t max_bounce_buffer_size;           // DMA bounce buffer 上限
    size_t bounce_buffer_size;               // 当前已分配的 bounce buffer
    QemuMutex map_client_list_lock;
    QLIST_HEAD(, AddressSpaceMapClient) map_client_list;
};
```

### 初始化

```c
// memory.c:3163-3175
void address_space_init(AddressSpace *as, MemoryRegion *root, const char *name) {
    memory_region_ref(root);
    as->root = root;
    as->current_map = NULL;
    as->ioeventfd_nb = 0;
    QTAILQ_INIT(&as->listeners);
    QTAILQ_INSERT_TAIL(&address_spaces, as, address_spaces_link);
    as->max_bounce_buffer_size = DEFAULT_MAX_BOUNCE_BUFFER_SIZE;
    // ...
    address_space_update_topology(as);       // 首次展平
}
```

### 读写 API

```c
// physmem.c:3419-3458
MemTxResult address_space_read_full(AddressSpace *as, hwaddr addr,
                                     MemTxAttrs attrs, void *buf, hwaddr len) {
    RCU_READ_LOCK_GUARD();                           // :3426
    fv = address_space_to_flatview(as);              // :3427 — RCU 读取
    result = flatview_read(fv, addr, attrs, buf, len); // :3428
    return result;
}

MemTxResult address_space_write(AddressSpace *as, hwaddr addr,
                                 MemTxAttrs attrs, const void *buf, hwaddr len) {
    RCU_READ_LOCK_GUARD();
    fv = address_space_to_flatview(as);
    result = flatview_write(fv, addr, attrs, buf, len); // :3444
}
```

### DMA 映射

```c
// physmem.c:3705-3767
void *address_space_map(AddressSpace *as, hwaddr addr, hwaddr *plen,
                         bool is_write, MemTxAttrs attrs) {
    fv = address_space_to_flatview(as);
    mr = flatview_translate(fv, addr, &xlat, &l, is_write, attrs);  // :3725

    if (!memory_access_is_direct(mr, is_write, attrs)) {
        // MMIO：分配 bounce buffer
        BounceBuffer *bounce = g_malloc0(l + sizeof(BounceBuffer));  // :3746
        if (!is_write) {
            flatview_read(fv, addr, attrs, bounce->buffer, l);       // :3754
        }
        return bounce->buffer;
    }

    // RAM：直接返回 host 指针
    return qemu_ram_ptr_length(mr->ram_block, xlat, plen, ...);      // :3766
}
```

---

## 5. FlatView 拓扑展平

### 数据结构

```c
// memory.h:1194-1202
struct FlatView {
    struct rcu_head rcu;
    unsigned ref;                    // 引用计数
    FlatRange *ranges;               // 有序 FlatRange 数组
    unsigned nr;                     // 当前 range 数量
    unsigned nr_allocated;           // 已分配容量
    struct AddressSpaceDispatch *dispatch; // 快速查找结构
    MemoryRegion *root;              // 对应的根 MR
};

// memory.c:222-231
struct FlatRange {
    MemoryRegion *mr;                // 终止 MR
    hwaddr offset_in_region;         // MR 内偏移
    AddrRange addr;                  // 绝对地址范围
    uint8_t dirty_log_mask;
    bool romd_mode;
    bool readonly;
    bool nonvolatile;
    bool unmergeable;
};
```

### render_memory_region — 递归展平算法

```c
// memory.c:596-686
static void render_memory_region(FlatView *view, MemoryRegion *mr,
                                  Int128 base, AddrRange clip,
                                  bool readonly, bool nonvolatile,
                                  bool unmergeable) {
    // 1. 未启用 → 跳过
    if (!mr->enabled) return;                            // :612

    // 2. 计算绝对地址，继承 readonly/nonvolatile 属性
    int128_addto(&base, int128_make64(mr->addr));        // :616
    readonly |= mr->readonly;                             // :617
    tmp = addrrange_make(base, mr->size);

    // 3. 裁剪：不在 clip 范围内 → 跳过
    if (!addrrange_intersects(tmp, clip)) return;         // :623
    clip = addrrange_intersection(tmp, clip);             // :627

    // 4. Alias → 递归展平目标 MR
    if (mr->alias) {
        render_memory_region(view, mr->alias, base - alias_offset,
                             clip, readonly, nonvolatile, unmergeable);
        return;                                           // :629-634
    }

    // 5. 递归处理子区域（按优先级排序）
    QTAILQ_FOREACH(subregion, &mr->subregions, subregions_link) {
        render_memory_region(view, subregion, base, clip, ...); // :638-641
    }

    // 6. 非终止 MR（container）→ 不生成 FlatRange
    if (!mr->terminates) return;                          // :643

    // 7. 在已有 ranges 的间隙中插入当前 MR 的 FlatRange
    for (i = 0; i < view->nr && remain > 0; ++i) {
        if (base < ranges[i].start) {
            // 间隙：插入 FlatRange                       // :663-669
            flatview_insert(view, i, &fr);
        }
        // 跳过已覆盖区域
    }
    // 尾部剩余
    if (remain > 0) {
        flatview_insert(view, i, &fr);                    // :681-684
    }
}
```

### generate_memory_topology — 生成完整拓扑

```c
// memory.c:748-772
static FlatView *generate_memory_topology(MemoryRegion *mr) {
    FlatView *view = flatview_new(mr);                    // :753

    render_memory_region(view, mr, 0, {0, 2^64},
                         false, false, false);            // :756-758

    flatview_simplify(view);                              // :760 — 合并相邻同 MR 范围

    // 构建快速查找结构
    view->dispatch = address_space_dispatch_new(view);    // :762
    for (i = 0; i < view->nr; i++) {
        flatview_add_to_dispatch(view, &mrs);             // :766
    }
    address_space_dispatch_compact(view->dispatch);       // :768

    g_hash_table_replace(flat_views, mr, view);           // :769
    return view;
}
```

**优先级规则**：子区域按 `priority` 降序、地址升序排列。高优先级子区域的 FlatRange 先插入，低优先级的只能填充间隙。

---

## 6. 地址解析全路径

```
address_space_read/write(as, addr, ...)
  │
  ├── address_space_to_flatview(as)          ← RCU 读取 current_map
  │
  ├── flatview_read/write(fv, addr, ...)
  │     │
  │     ├── flatview_do_translate()           ← physmem.c:493-528
  │     │     │
  │     │     ├── address_space_translate_internal()  ← physmem.c:366-398
  │     │     │   ├── address_space_lookup_region(d, addr)  ← 二分/页表查找
  │     │     │   ├── xlat = addr - section.oas + section.oir  ← 计算 MR 内偏移
  │     │     │   └── return MemoryRegionSection
  │     │     │
  │     │     └── [如果 MR 是 IOMMU] → address_space_translate_iommu()
  │     │
  │     ├── [RAM] → memcpy(host + xlat, buf)    ← 直接内存访问
  │     │
  │     └── [MMIO] → access_with_adjusted_size()
  │                   → memory_region_read/write_accessor()
  │                     → mr->ops->read/write(opaque, xlat, size)
```

### flatview_do_translate

```c
// physmem.c:493-528
static MemoryRegionSection flatview_do_translate(FlatView *fv, hwaddr addr,
    hwaddr *xlat, hwaddr *plen_out, hwaddr *page_mask_out,
    bool is_write, bool is_mmio, AddressSpace **target_as, MemTxAttrs attrs) {

    // 1. 在 dispatch 中查找 section
    section = address_space_translate_internal(
        flatview_to_dispatch(fv), addr, xlat, plen_out, is_mmio);  // :511-513

    // 2. 检查 IOMMU
    iommu_mr = memory_region_get_iommu(section->mr);
    if (unlikely(iommu_mr)) {
        return address_space_translate_iommu(iommu_mr, xlat, ...); // :517-520
    }

    return *section;
}
```

### address_space_translate_internal

```c
// physmem.c:366-398
static MemoryRegionSection *
address_space_translate_internal(AddressSpaceDispatch *d, hwaddr addr,
                                  hwaddr *xlat, hwaddr *plen, bool resolve_subpage) {
    section = address_space_lookup_region(d, addr, resolve_subpage); // :373
    addr -= section->offset_within_address_space;                    // :375
    *xlat = addr + section->offset_within_region;                    // :378

    // RAM 区域：限制访问长度到 section 边界
    if (memory_region_is_ram(mr)) {
        *plen = min(section.size - addr, *plen);                     // :393-395
    }
    return section;
}
```

### MemoryRegionSection

```c
// memory.h:105-114
struct MemoryRegionSection {
    Int128 size;                     // section 大小
    MemoryRegion *mr;                // 对应的 MR
    FlatView *fv;                    // 所属 FlatView
    hwaddr offset_within_region;     // MR 内起始偏移
    hwaddr offset_within_address_space; // 地址空间中的绝对地址
    bool readonly;
    bool nonvolatile;
    bool unmergeable;
};
```

---

## 7. MemoryListener 拓扑变更通知

### MemoryListener 结构

```c
// memory.h:889-950+ (摘要)
struct MemoryListener {
    void (*begin)(MemoryListener *listener);
    void (*commit)(MemoryListener *listener);
    void (*region_add)(MemoryListener *listener, MemoryRegionSection *section);
    void (*region_del)(MemoryListener *listener, MemoryRegionSection *section);
    void (*region_nop)(MemoryListener *listener, MemoryRegionSection *section);
    void (*log_start)(MemoryListener *, MemoryRegionSection *, int old, int new);
    void (*log_stop)(MemoryListener *, MemoryRegionSection *, int old, int new);
    void (*log_sync)(MemoryListener *, MemoryRegionSection *);
    void (*log_sync_global)(MemoryListener *);
    void (*log_clear)(MemoryListener *, MemoryRegionSection *);
    void (*eventfd_add)(MemoryListener *, MemoryRegionSection *, ...);
    void (*eventfd_del)(MemoryListener *, MemoryRegionSection *, ...);

    unsigned priority;               // 回调优先级
    const char *name;
    AddressSpace *address_space;
    QTAILQ_ENTRY(MemoryListener) link;
    QTAILQ_ENTRY(MemoryListener) link_as;
};
```

### 注册

```c
// memory.c:3101-3138
void memory_listener_register(MemoryListener *listener, AddressSpace *as) {
    listener->address_space = as;
    // 按优先级插入全局链表 memory_listeners
    QTAILQ_INSERT_...(&memory_listeners, listener, link);     // :3109-3118
    // 按优先级插入 AS 链表
    QTAILQ_INSERT_...(&as->listeners, listener, link_as);     // :3121-3131
    // 对已有 section 回放 region_add
    listener_add_address_space(listener, as);                  // :3133
}
```

### 拓扑变更提交流程

```c
// memory.c:1144-1172
void memory_region_transaction_commit(void) {
    --memory_region_transaction_depth;
    if (depth == 0 && memory_region_update_pending) {
        // 1. 重新生成所有 FlatView
        flatviews_reset();                                     // :1154

        // 2. 通知 begin
        MEMORY_LISTENER_CALL_GLOBAL(begin, Forward);           // :1156

        // 3. 对每个 AddressSpace 执行 diff
        QTAILQ_FOREACH(as, &address_spaces, ...) {
            address_space_set_flatview(as);                    // :1159
            // ↳ address_space_update_topology_pass(old, new, false) → region_del
            // ↳ address_space_update_topology_pass(old, new, true)  → region_add/nop
            address_space_update_ioeventfds(as);               // :1160
        }

        // 4. 通知 commit
        MEMORY_LISTENER_CALL_GLOBAL(commit, Forward);          // :1164
    }
}
```

### 拓扑 Diff 算法

```c
// memory.c:971-1038
static void address_space_update_topology_pass(AddressSpace *as,
    const FlatView *old_view, const FlatView *new_view, bool adding) {
    // 双指针扫描 old_view 和 new_view（地址有序数组）
    while (iold < old_view->nr || inew < new_view->nr) {
        if (旧有但新无 || 属性变化) {
            if (!adding) → region_del(frold)              // 第一遍：删除旧的
        } else if (新旧相同) {
            if (adding) → region_nop(frnew)               // 第二遍：通知未变
                        → 处理 dirty_log 变化（log_start/stop）
        } else {
            if (adding) → region_add(frnew)               // 第二遍：添加新的
        }
    }
}
```

**两遍策略**：先 `adding=false` 删除旧区域，再 `adding=true` 添加新区域。确保监听器先看到删除再看到添加，避免冲突。

---

## 8. IOMMU 翻译

### IOMMUMemoryRegion

```c
// memory.h:869-874
struct IOMMUMemoryRegion {
    MemoryRegion parent_obj;                      // 继承 MemoryRegion
    QLIST_HEAD(, IOMMUNotifier) iommu_notify;     // 通知器链表
    IOMMUNotifierFlag iommu_notify_flags;          // 活跃通知标志
};
```

### IOMMUMemoryRegionClass

```c
// memory.h:401-543 (摘要)
struct IOMMUMemoryRegionClass {
    MemoryRegionClass parent_class;

    // 核心翻译回调
    IOMMUTLBEntry (*translate)(IOMMUMemoryRegion *iommu, hwaddr addr,
                                IOMMUAccessFlags flag, int iommu_idx); // :436

    uint64_t (*get_min_page_size)(IOMMUMemoryRegion *iommu);           // :448
    int (*notify_flag_changed)(IOMMUMemoryRegion *, IOMMUNotifierFlag old,
                                IOMMUNotifierFlag new, Error **errp);  // :468
    void (*replay)(IOMMUMemoryRegion *, IOMMUNotifier *);              // :490
    int (*get_attr)(IOMMUMemoryRegion *, enum IOMMUMemoryRegionAttr, void *); // :512
    int (*attrs_to_index)(IOMMUMemoryRegion *, MemTxAttrs attrs);      // :529
    int (*num_indexes)(IOMMUMemoryRegion *);                            // :542
};
```

### IOMMUTLBEntry

```c
// memory.h:147-154
struct IOMMUTLBEntry {
    AddressSpace    *target_as;      // 目标地址空间
    hwaddr           iova;           // IO 虚拟地址
    hwaddr           translated_addr;// 翻译后物理地址
    hwaddr           addr_mask;      // 页掩码（0xfff = 4K）
    IOMMUAccessFlags perm;           // 权限
    uint32_t         pasid;          // Process Address Space ID
};
```

### IOMMU 翻译路径

```c
// physmem.c:422-471
static MemoryRegionSection address_space_translate_iommu(
    IOMMUMemoryRegion *iommu_mr, hwaddr *xlat, hwaddr *plen_out, ...) {

    do {
        hwaddr addr = *xlat;
        IOMMUMemoryRegionClass *imrc = memory_region_get_iommu_class_nocheck(iommu_mr);

        // IOMMU 索引选择
        if (imrc->attrs_to_index)
            iommu_idx = imrc->attrs_to_index(iommu_mr, attrs);  // :440-441

        // 调用 IOMMU translate 回调
        iotlb = imrc->translate(iommu_mr, addr, is_write ? IOMMU_WO : IOMMU_RO,
                                 iommu_idx);                      // :444-445

        // 权限检查
        if (!(iotlb.perm & (1 << is_write)))
            goto unassigned;                                      // :447

        // 计算翻译后地址
        addr = (iotlb.translated_addr & ~iotlb.addr_mask)
             | (addr & iotlb.addr_mask);                          // :451-452

        // 在目标 AS 中继续解析
        section = address_space_translate_internal(
            address_space_to_dispatch(iotlb.target_as), addr, xlat, ...); // :457-459

        // 多级 IOMMU：检查结果是否又是 IOMMU
        iommu_mr = memory_region_get_iommu(section->mr);
    } while (unlikely(iommu_mr));                                 // :462

    return *section;
}
```

### IOMMU 通知器

```c
// memory.h:186-215
typedef enum {
    IOMMU_NOTIFIER_NONE = 0,
    IOMMU_NOTIFIER_UNMAP = 0x1,       // 缓存失效通知
    IOMMU_NOTIFIER_MAP = 0x2,         // 新映射通知
    IOMMU_NOTIFIER_DEVIOTLB_UNMAP = 0x04, // 设备 IOTLB 失效
} IOMMUNotifierFlag;

struct IOMMUNotifier {
    IOMMUNotify notify;               // 回调函数
    IOMMUNotifierFlag notifier_flags;  // 关注的事件类型
    hwaddr start;                     // 通知地址范围起始
    hwaddr end;                       // 通知地址范围结束
    int iommu_idx;                    // IOMMU 索引
    void *opaque;
    QLIST_ENTRY(IOMMUNotifier) node;
};
```

---

## 9. TCG 内存访问路径

### 整体流程

```
Guest LDR X0, [X1]
  │
  ├── TCG 翻译 → tcg_gen_qemu_ld_i64()
  │     ↓
  ├── 后端生成 TLB 快路径代码
  │     │
  │     ├── TLB 命中 → LDR host_addr（直接访问）
  │     │
  │     └── TLB 未命中 → helper_ld*_mmu()
  │           │
  │           ├── tlb_fill() → cpu_tlb_fill()
  │           │   ├── target/arm: arm_cpu_tlb_fill()
  │           │   │   └── get_phys_addr() → MMU 页表遍历
  │           │   │       → tlb_set_page() 填充 TLB
  │           │   └── 重试 TLB 查找
  │           │
  │           ├── [RAM] → memcpy 从 host 地址
  │           │
  │           └── [MMIO] → io_readx()
  │                 ├── flatview_do_translate()
  │                 └── memory_region_dispatch_read()
  │                     → mr->ops->read(opaque, xlat, size)
```

### TLB 与 MemoryRegion 的连接

TCG 的 `CPUTLBEntry` 中的 `addend` 字段直接指向 `RAMBlock->host - GPA` 偏移。TLB 命中时，`host_addr = guest_addr + addend`，直接访问 host 内存，完全绕过 MemoryRegion 层。

TLB 未命中时，`tlb_fill()` 调用目标架构的 `cpu_tlb_fill()`，后者执行 Guest MMU 页表遍历，再通过 `address_space_translate()` 找到对应的 MemoryRegion，设置 TLB 条目。

---

## 10. KVM 内存路径

### KVM 内存监听器

```c
// kvm-all.c:2090-2126
void kvm_memory_listener_register(KVMState *s, KVMMemoryListener *kml,
                                   AddressSpace *as, int as_id, const char *name) {
    kml->listener.region_add = kvm_region_add;        // :2102
    kml->listener.region_del = kvm_region_del;        // :2103
    kml->listener.commit = kvm_region_commit;         // :2104
    kml->listener.log_start = kvm_log_start;          // :2105
    kml->listener.log_stop = kvm_log_stop;            // :2106
    kml->listener.priority = MEMORY_LISTENER_PRIORITY_ACCEL;

    // 脏页同步策略选择
    if (s->kvm_dirty_ring_size) {
        kml->listener.log_sync_global = kvm_log_sync_global; // :2111
    } else {
        kml->listener.log_sync = kvm_log_sync;               // :2113
        kml->listener.log_clear = kvm_log_clear;             // :2114
    }

    memory_listener_register(&kml->listener, as);             // :2117
}
```

### KVM 事务提交

```c
// kvm-all.c:1864-1957
// region_add/del 只是将 section 加入事务队列
static void kvm_region_add(MemoryListener *listener, MemoryRegionSection *section) {
    update = g_new0(KVMMemoryUpdate, 1);
    update->section = *section;
    QSIMPLEQ_INSERT_TAIL(&kml->transaction_add, update, next);  // :1873
}

// commit 时批量执行
static void kvm_region_commit(MemoryListener *listener) {
    // 检测 add/del 重叠 → 需要 ioctl 互斥
    if (range_overlaps_range(&r1, &r2)) {
        need_inhibit = true;                                     // :1918
    }

    kvm_slots_lock();                                            // :1928
    if (need_inhibit) accel_ioctl_inhibit_begin();               // :1930

    // 先删除所有旧 slot
    while (!EMPTY(&kml->transaction_del)) {
        kvm_set_phys_mem(kml, &section, false);                  // :1938
    }
    // 再添加所有新 slot
    while (!EMPTY(&kml->transaction_add)) {
        kvm_set_phys_mem(kml, &section, true);                   // :1948
    }

    if (need_inhibit) accel_ioctl_inhibit_end();
    kvm_slots_unlock();
}
```

### KVM_SET_USER_MEMORY_REGION

```c
// kvm-all.c:371-410
static int kvm_set_user_memory_region(KVMMemoryListener *kml, KVMSlot *slot, bool new) {
    struct kvm_userspace_memory_region2 mem = {};
    mem.slot = slot->slot | (kml->as_id << 16);       // slot ID + AS ID
    mem.guest_phys_addr = slot->start_addr;            // GPA
    mem.userspace_addr = (unsigned long)slot->ram;     // HVA
    mem.flags = slot->flags;                           // KVM_MEM_READONLY 等
    mem.memory_size = slot->memory_size;

    // 调用 KVM ioctl
    if (kvm_guest_memfd_supported) {
        ret = kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION2, &mem); // :400
    } else {
        ret = kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem);  // :402
    }
}
```

---

## 11. RAMBlock 与 ROM

### RAMBlock 结构

```c
// include/system/ramblock.h:25-92
struct RAMBlock {
    struct rcu_head rcu;
    struct MemoryRegion *mr;         // 关联的 MR
    uint8_t *host;                   // Host 虚拟地址（mmap 结果）
    uint8_t *colo_cache;             // COLO 缓存
    ram_addr_t offset;               // 全局 RAM 空间偏移
    ram_addr_t used_length;          // 当前使用长度
    ram_addr_t max_length;           // 最大可扩展长度
    void (*resized)(...);            // 扩展回调
    uint32_t flags;                  // RAM_PREALLOC/RAM_SHARED/RAM_GUEST_MEMFD 等
    char idstr[256];                 // 迁移标识
    QLIST_ENTRY(RAMBlock) next;      // 全局 RAMBlock 链表
    int fd;                          // 后端文件描述符
    int guest_memfd;                 // 机密计算 guest memfd
    size_t page_size;                // 页大小

    // 迁移相关
    unsigned long *bmap;             // 脏页位图
    unsigned long *file_bmap;        // 文件中的页面位图
    off_t bitmap_offset;             // 文件中的位图偏移
    uint64_t pages_offset;           // 文件中的页面偏移
    unsigned long *receivedmap;      // 已接收页面位图
    unsigned long *clear_bmap;       // 待清除脏页位图
    uint8_t clear_bmap_shift;        // 清除粒度
    ram_addr_t postcopy_length;      // postcopy 长度
};
```

### RAM 初始化

```c
// memory.c:1594-1604
bool memory_region_init_ram_flags_nomigrate(MemoryRegion *mr, Object *owner,
    const char *name, uint64_t size, uint32_t ram_flags, Error **errp) {
    memory_region_init(mr, owner, name, size);                   // :1600
    mr->ram = true;                                               // 隐含在 set_ram_block 中
    RAMBlock *rb = qemu_ram_alloc(size, ram_flags, mr, errp);    // :1601
    return memory_region_set_ram_block(mr, rb);                   // :1602
    // set_ram_block 设置: terminates=true, destructor=ram, ram_block=rb
}

// memory.c:3661-3671
bool memory_region_init_ram(MemoryRegion *mr, ...) {
    memory_region_init_ram_flags_nomigrate(mr, ...);             // :3665
    memory_region_register_ram(mr, owner);                        // :3669 — 注册迁移
}
```

### ROM

```c
// memory.c:3685-3695
bool memory_region_init_rom(MemoryRegion *mr, ...) {
    memory_region_init_ram_flags_nomigrate(mr, ...);             // :3689
    mr->readonly = true;                                          // :3693 — 唯一区别
    memory_region_register_ram(mr, owner);
}

// memory.c:3698-3713 — ROM 设备（读 RAM，写走 MMIO ops）
bool memory_region_init_rom_device(MemoryRegion *mr, ..., const MemoryRegionOps *ops, ...) {
    memory_region_init_io(mr, owner, ops, opaque, name, size);   // :3706
    RAMBlock *rb = qemu_ram_alloc(size, 0, mr, errp);            // :3707
    mr->rom_device = true;                                        // :3709
}
```

---

## 12. 总结

### 内存子系统分层架构

```
┌─────────────────────────────────────────────────┐
│  设备层                                          │
│  MemoryRegionOps.read/write → UART/GIC/PCIe     │
├─────────────────────────────────────────────────┤
│  MemoryRegion 树层                               │
│  容器嵌套 + Alias + 优先级 + 动态增删            │
├─────────────────────────────────────────────────┤
│  FlatView 层                                     │
│  树 → 有序数组 → AddressSpaceDispatch            │
│  RCU 无锁读，事务性更新                          │
├─────────────────────────────────────────────────┤
│  地址解析层                                      │
│  flatview_do_translate → IOMMU → MR Section      │
├─────────────────────────────────────────────────┤
│  加速器绑定层                                    │
│  TCG: CPUTLBEntry.addend → host 直接访问         │
│  KVM: KVM_SET_USER_MEMORY_REGION → EPT/Stage2   │
├─────────────────────────────────────────────────┤
│  物理后端层                                      │
│  RAMBlock.host → mmap 匿名/文件                  │
│  脏页跟踪 → 迁移                                │
└─────────────────────────────────────────────────┘
```

### 设计亮点

1. **MR 树形抽象**：通过 Container/Alias/Overlap 三种组合方式，用树形结构表达任意复杂的地址映射拓扑。设备只需 `memory_region_init_io()` + `add_subregion()` 即可将寄存器映射到地址空间。

2. **FlatView 展平 + RCU**：MR 树动态变化时，通过 `render_memory_region()` 递归展平生成新的 FlatView，用 RCU 原子替换。读者（CPU 线程）无锁访问旧视图，写者（主线程）在事务提交时切换。

3. **MemoryListener 观察者模式**：KVM/TCG/vhost 等加速器通过 MemoryListener 监听拓扑变化，自动同步自己的内存映射（KVM memslot、TCG TLB flush、vhost 内存表）。

4. **IOMMU 级联翻译**：`address_space_translate_iommu()` 支持多级 IOMMU 级联（do-while 循环），每级 IOMMU 翻译后检查结果是否又是 IOMMU，直到到达终止 MR。

5. **KVM 事务批量提交**：`kvm_region_commit()` 收集一个事务内的所有 add/del，检测重叠后在 kvm_slots_lock 保护下批量执行 ioctl，确保原子性。

6. **TCG TLB 直通**：TCG 的 TLB 命中路径完全绕过 MemoryRegion 层，通过 `addend` 直接计算 host 地址。只有 TLB 未命中时才走完整的 address_space_translate 路径。

---

**关键源文件**：
- `include/system/memory.h` — MemoryRegion/AddressSpace/FlatView/IOMMU/MemoryListener 定义
- `system/memory.c` — MR 操作、FlatView 展平、拓扑变更提交、Listener 管理
- `system/physmem.c` — 地址解析、RAM 分配、DMA 映射、读写 API
- `accel/kvm/kvm-all.c` — KVM 内存监听器、memslot 管理
- `include/system/ramblock.h` — RAMBlock 结构
- `accel/tcg/cputlb.c` — TCG TLB 管理
