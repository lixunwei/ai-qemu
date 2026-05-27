# Doc 98: KVM 内存管理深度分析

## Memory Slot · 脏页追踪 · ioeventfd/irqfd 快速路径 · MemoryListener 集成

> QEMU 11.0.50 · accel/kvm/kvm-all.c (4786行)
> 分析日期: 2025-01
> 前置文档: Doc 86 (KVM加速器集成概览)

---

## 目录

1. [概述：QEMU 内存 → KVM 内存](#1-概述qemu-内存--kvm-内存)
2. [Memory Slot 管理](#2-memory-slot-管理)
3. [kvm_set_user_memory_region — 核心映射函数](#3-kvm_set_user_memory_region--核心映射函数)
4. [MemoryListener 集成](#4-memorylistener-集成)
5. [脏页追踪：Dirty Bitmap 模式](#5-脏页追踪dirty-bitmap-模式)
6. [脏页追踪：Dirty Ring 模式](#6-脏页追踪dirty-ring-模式)
7. [两种脏页模式对比](#7-两种脏页模式对比)
8. [脏页追踪与迁移联动](#8-脏页追踪与迁移联动)
9. [ioeventfd — MMIO/PIO 快速路径](#9-ioeventfd--mmiopio-快速路径)
10. [irqfd — 中断快速路径](#10-irqfd--中断快速路径)
11. [MSI 路由](#11-msi-路由)
12. [VirtIO 全快速路径](#12-virtio-全快速路径)
13. [Memory Attributes (新特性)](#13-memory-attributes-新特性)
14. [与硬件对比：ARM64 Stage-2 页表](#14-与硬件对比arm64-stage-2-页表)

---

## 1. 概述：QEMU 内存 → KVM 内存

```
┌─────────────────────────────────────────────────────┐
│  QEMU 用户态                                         │
│                                                      │
│  AddressSpace (system_memory)                        │
│    ├─ MemoryRegion "ram" (1GB)     → mmap'd HVA     │
│    ├─ MemoryRegion "flash" (64MB)  → mmap'd HVA     │
│    └─ MemoryRegion "mmio" (设备)   → 不映射到 KVM    │
│                                                      │
│  KVMMemoryListener → 监听 flatview 变化              │
│    ↓ region_add/del                                  │
├──────────────────────────────────────────────────────┤
│  KVM 内核                                            │
│                                                      │
│  KVM_SET_USER_MEMORY_REGION                          │
│    → memslot[0]: GPA 0x4000_0000 → HVA → Stage-2    │
│    → memslot[1]: GPA 0x0000_0000 → HVA → Stage-2    │
│                                                      │
│  ARM64: 填充 Stage-2 页表                             │
│  Guest 物理地址 → Host 虚拟地址 → Host 物理地址       │
└──────────────────────────────────────────────────────┘
```

核心原则：
- **只有 RAM** 映射为 KVM memslot（有 HVA 的 MemoryRegion）
- **MMIO** 不映射 → Guest 访问触发 Stage-2 fault → VM Exit → KVM_EXIT_MMIO
- QEMU MemoryRegion 变化通过 MemoryListener **自动同步**到 KVM

---

## 2. Memory Slot 管理

### 2.1 KVMSlot 结构

```c
// include/system/kvm_int.h:22-54
typedef struct KVMSlot {
    hwaddr start_addr;           // Guest 物理起始地址 (GPA)
    ram_addr_t memory_size;      // 区域大小
    void *ram;                   // Host 虚拟地址 (HVA, mmap指针)
    int slot;                    // slot 编号
    int flags;                   // KVM_MEM_LOG_DIRTY_PAGES | READONLY | GUEST_MEMFD
    int old_flags;               // 更新前的旧 flags
    /* 脏页追踪 */
    unsigned long *dirty_bmap;   // 脏页 bitmap
    unsigned long dirty_bmap_size;
    /* guest-memfd (CoCo VM) */
    uint32_t guest_memfd;
    uint64_t guest_memfd_offset;
} KVMSlot;

typedef struct KVMMemoryListener {
    MemoryListener listener;      // 注册到 QEMU MemoryListener 框架
    KVMSlot *slots;               // slot 数组
    unsigned int nr_slots_used;   // 已用 slot 数
    unsigned int nr_slots_allocated; // 已分配 slot 数
    int as_id;                    // 地址空间 ID (0=memory, 1=SMM)
    QSIMPLEQ_HEAD(, KVMMemoryUpdate) transaction_add;  // 待添加
    QSIMPLEQ_HEAD(, KVMMemoryUpdate) transaction_del;  // 待删除
} KVMMemoryListener;
```

### 2.2 Slot 分配与查找

```c
// accel/kvm/kvm-all.c:195-307
static bool kvm_slots_grow(KVMMemoryListener *kml, unsigned int nr_slots_new) {
    // 扩展 slot 数组（realloc），上限为 s->nr_slots_max
    kml->slots = g_renew(KVMSlot, kml->slots, nr_slots_new);
}

static bool kvm_slots_double(KVMMemoryListener *kml) {
    // 翻倍扩容
    return kvm_slots_grow(kml, kml->nr_slots_allocated * 2);
}

static KVMSlot *kvm_get_free_slot(KVMMemoryListener *kml) {
    // 遍历找 memory_size == 0 的空闲 slot
    for (i = 0; i < kml->nr_slots_used; i++) {
        if (kml->slots[i].memory_size == 0) return &kml->slots[i];
    }
    // 如果满了，使用 nr_slots_used++ 或尝试 double
}

static KVMSlot *kvm_lookup_matching_slot(KVMMemoryListener *kml,
                                          hwaddr start, hwaddr size) {
    // 精确匹配 start_addr 和 memory_size
}
```

### 2.3 最大 Slot 大小

```c
// accel/kvm/kvm-all.c:112, 1594-1600
static hwaddr kvm_max_slot_size = ~0;  // 默认无限制

void kvm_set_max_memslot_size(hwaddr max_slot_size) {
    kvm_max_slot_size = max_slot_size;
}
```

当 RAM 大于 `kvm_max_slot_size` 时，`kvm_set_phys_mem()` 会将其**拆分为多个 slot**：

```c
// accel/kvm/kvm-all.c:1718-1754 (add path)
while (size > 0) {
    slot_size = MIN(kvm_max_slot_size, size);
    mem = kvm_alloc_slot(kml);
    mem->start_addr = start_addr;
    mem->memory_size = slot_size;
    mem->ram = ram;
    mem->flags = kvm_mem_flags(mr);
    kvm_set_user_memory_region(kml, mem, true);
    start_addr += slot_size;
    ram += slot_size;
    size -= slot_size;
}
```

---

## 3. kvm_set_user_memory_region — 核心映射函数

```c
// accel/kvm/kvm-all.c:371-430
static int kvm_set_user_memory_region(KVMMemoryListener *kml, KVMSlot *slot, bool new) {
    struct kvm_userspace_memory_region2 mem = {
        .slot = slot->slot | (kml->as_id << 16),   // slot编号 | 地址空间ID
        .guest_phys_addr = slot->start_addr,         // GPA
        .userspace_addr = (uintptr_t)slot->ram,      // HVA
        .flags = slot->flags,
        .memory_size = slot->memory_size,            // 0 表示删除
    };

    // guest-memfd (CoCo VM 私有内存)
    if (slot->flags & KVM_MEM_GUEST_MEMFD) {
        mem.guest_memfd = slot->guest_memfd;
        mem.guest_memfd_offset = slot->guest_memfd_offset;
    }

    // ioctl 调用
    return kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION2, &mem);
}
```

**关键语义：**
- `memory_size > 0` → 创建/更新 slot
- `memory_size = 0` → 删除 slot
- `flags` 控制只读、脏页追踪、私有内存

### Slot 删除流程

```c
// accel/kvm/kvm-all.c:1667-1715 (delete path)
mem = kvm_lookup_matching_slot(kml, start_addr, size);
mem->memory_size = 0;        // 标记为删除
mem->flags = 0;
kvm_set_user_memory_region(kml, mem, false);  // size=0 → 删除
kvm_slot_init_dirty_bitmap(mem);              // 清理 bitmap
```

---

## 4. MemoryListener 集成

### 4.1 注册

```c
// accel/kvm/kvm-all.c:2090-2126
void kvm_memory_listener_register(KVMState *s, KVMMemoryListener *kml,
                                   AddressSpace *as, int as_id) {
    kml->slots = NULL;
    kml->as_id = as_id;
    kml->listener.region_add = kvm_region_add;
    kml->listener.region_del = kvm_region_del;
    kml->listener.commit = kvm_region_commit;
    kml->listener.log_start = kvm_log_start;
    kml->listener.log_stop = kvm_log_stop;

    if (dirty_ring) {
        kml->listener.log_sync_global = kvm_log_sync_global;
    } else {
        kml->listener.log_sync = kvm_log_sync;
        kml->listener.log_clear = kvm_log_clear;
    }

    memory_listener_register(&kml->listener, as);
    // 保存 as → kml 映射
    s->as[s->nr_as].as = as;
    s->as[s->nr_as].ml = kml;
    s->nr_as++;
}
```

### 4.2 事务性更新

```c
// accel/kvm/kvm-all.c:1864-1957
static void kvm_region_add(MemoryListener *listener, MemoryRegionSection *section) {
    // 入队到 transaction_add（不立即执行）
    KVMMemoryUpdate *update = g_new0(KVMMemoryUpdate, 1);
    update->section = *section;
    QSIMPLEQ_INSERT_TAIL(&kml->transaction_add, update, next);
}

static void kvm_region_del(MemoryListener *listener, MemoryRegionSection *section) {
    // 入队到 transaction_del
    QSIMPLEQ_INSERT_TAIL(&kml->transaction_del, update, next);
}

static void kvm_region_commit(MemoryListener *listener) {
    // 原子提交：先删后加
    // 如果有重叠更新，需要 inhibit ioctl 避免竞争
    need_inhibit = !QSIMPLEQ_EMPTY(&kml->transaction_del) &&
                   !QSIMPLEQ_EMPTY(&kml->transaction_add);

    if (need_inhibit) accel_ioctl_inhibit_begin();  // 暂停 vCPU ioctl

    // 1. 处理所有删除
    QSIMPLEQ_FOREACH(update, &kml->transaction_del) {
        kvm_set_phys_mem(kml, section, false);  // 删除 slot
    }
    // 2. 处理所有添加
    QSIMPLEQ_FOREACH(update, &kml->transaction_add) {
        kvm_set_phys_mem(kml, section, true);   // 添加 slot
    }

    if (need_inhibit) accel_ioctl_inhibit_end();
}
```

### 4.3 地址空间绑定

| as_id | AddressSpace | 用途 |
|-------|-------------|------|
| 0 | `address_space_memory` | 主内存 |
| 1 | `address_space_smm` | SMM（x86）|

ARM64 通常只有 as_id=0。

---

## 5. 脏页追踪：Dirty Bitmap 模式

### 5.1 原理

```
Guest 写入 GPA → Stage-2 PTE D位置位 → KVM 记录到 dirty bitmap
                                           ↓
QEMU 调用 KVM_GET_DIRTY_LOG → 获取 bitmap → 标记 RAM dirty → 迁移发送
```

### 5.2 Bitmap 初始化

```c
// accel/kvm/kvm-all.c:901-928
static void kvm_slot_init_dirty_bitmap(KVMSlot *mem) {
    if (!(mem->flags & KVM_MEM_LOG_DIRTY_PAGES) || mem->dirty_bmap) {
        return;
    }
    // 每个 page (4KB) 1 bit
    unsigned long bmap_size = ALIGN(mem->memory_size / qemu_real_host_page_size(), 64) / 8;
    mem->dirty_bmap = g_malloc0(bmap_size);
    mem->dirty_bmap_size = bmap_size;
}
```

### 5.3 获取脏页

```c
// accel/kvm/kvm-all.c:934-952
static bool kvm_slot_get_dirty_log(KVMState *s, KVMSlot *slot) {
    struct kvm_dirty_log d = {
        .slot = slot->slot | (as_id << 16),
        .dirty_bitmap = slot->dirty_bmap,
    };
    return kvm_vm_ioctl(s, KVM_GET_DIRTY_LOG, &d) == 0;
}
```

### 5.4 同步到 QEMU RAM bitmap

```c
// accel/kvm/kvm-all.c:1163-1185
static int kvm_physical_sync_dirty_bitmap(KVMMemoryListener *kml, MemoryRegionSection *s) {
    // 对每个与 section 重叠的 slot:
    kvm_slot_get_dirty_log(kvm_state, mem);
    // 转换为 QEMU 的 dirty bitmap
    kvm_get_dirty_pages_log_range(section, mem->dirty_bmap);
    // → cpu_physical_memory_set_dirty_lebitmap()
}
```

---

## 6. 脏页追踪：Dirty Ring 模式

### 6.1 原理

```
Guest 写入 GPA → KVM 将 {slot, offset, flags} 写入 per-vCPU dirty ring
                 ↓ (ring 满时 vCPU exit: KVM_EXIT_DIRTY_RING_FULL)
QEMU reaper 线程 → 读取 ring entries → 标记 slot bitmap → KVM_RESET_DIRTY_RINGS
```

### 6.2 初始化

```c
// accel/kvm/kvm-all.c:2729-2775
static void kvm_setup_dirty_ring(KVMState *s) {
    // 优先使用 ACQ_REL 版本（更高效）
    uint64_t ring_size = s->kvm_dirty_ring_size;  // 通常 65536 entries

    if (kvm_check_extension(s, KVM_CAP_DIRTY_LOG_RING_ACQ_REL)) {
        kvm_vm_enable_cap(s, KVM_CAP_DIRTY_LOG_RING_ACQ_REL, ring_size);
    } else {
        kvm_vm_enable_cap(s, KVM_CAP_DIRTY_LOG_RING, ring_size);
    }
}
```

### 6.3 mmap Ring Pages

```c
// accel/kvm/kvm-all.c:697-709
static int map_kvm_dirty_gfns(KVMState *s, CPUState *cpu, Error **errp) {
    uint64_t ring_size = s->kvm_dirty_ring_size * sizeof(struct kvm_dirty_gfn);
    cpu->kvm_dirty_gfns = mmap(NULL, ring_size, PROT_READ | PROT_WRITE,
                               MAP_SHARED, cpu->kvm_fd, s->kvm_dirty_ring_bytes);
    // mmap 共享映射 → 零拷贝读取 dirty entries
}
```

### 6.4 Dirty Ring Entry

```c
// linux-headers/linux/kvm.h (概念)
struct kvm_dirty_gfn {
    __u32 flags;    // KVM_DIRTY_GFN_F_DIRTY / RESET / COLLECTED
    __u32 slot;     // memslot 编号
    __u64 offset;   // slot 内 page offset (以 page 为单位)
};
```

### 6.5 Reap (收割) 流程

```c
// accel/kvm/kvm-all.c:976-1044
static bool dirty_gfn_is_dirtied(struct kvm_dirty_gfn *gfn) {
    // ACQ_REL: qatomic_read(&gfn->flags) == KVM_DIRTY_GFN_F_DIRTY
    return smp_load_acquire(&gfn->flags) == KVM_DIRTY_GFN_F_DIRTY;
}

static void dirty_gfn_set_collected(struct kvm_dirty_gfn *gfn) {
    // 标记为已收集（允许 KVM 重用此 entry）
    smp_store_release(&gfn->flags, KVM_DIRTY_GFN_F_RESET);
}

static uint32_t kvm_dirty_ring_reap_one(KVMState *s, CPUState *cpu) {
    struct kvm_dirty_gfn *dirty_gfns = cpu->kvm_dirty_gfns;
    uint32_t ring_size = s->kvm_dirty_ring_size;
    uint32_t count = 0;
    uint32_t fetch = cpu->kvm_fetch_index;  // 当前读取位置

    while (true) {
        struct kvm_dirty_gfn *cur = &dirty_gfns[fetch % ring_size];
        if (!dirty_gfn_is_dirtied(cur)) break;  // 环形buffer空

        // 标记到对应 slot 的 bitmap
        KVMSlot *slot = &kml->slots[cur->slot];
        set_bit(cur->offset, slot->dirty_bmap);

        dirty_gfn_set_collected(cur);  // 归还给 KVM
        fetch++;
        count++;
    }
    cpu->kvm_fetch_index = fetch;
    return count;
}
```

### 6.6 Reaper 后台线程

```c
// accel/kvm/kvm-all.c:1757-1790
static void *kvm_dirty_ring_reaper_thread(void *data) {
    // 后台线程持续收割所有 vCPU 的 dirty ring
    while (reaper->reaper_state != KVM_DIRTY_RING_REAPER_STOP) {
        kvm_dirty_ring_reap(s, NULL);  // reap 所有 CPU
        // 通知迁移线程
        kvm_dirty_ring_reaper_notify(reaper);
        // 等待下次唤醒
        kvm_dirty_ring_reaper_wait(reaper);
    }
}
```

### 6.7 Ring 满处理

当 dirty ring 满时，KVM 触发 `KVM_EXIT_DIRTY_RING_FULL`：

```c
// accel/kvm/kvm-all.c (in kvm_cpu_exec switch)
case KVM_EXIT_DIRTY_RING_FULL:
    // 紧急收割当前 CPU 的 ring
    kvm_dirty_ring_reap_locked(s, cpu);
    // 重置 ring
    kvm_vm_ioctl(s, KVM_RESET_DIRTY_RINGS, 0);
    ret = 0;  // 继续运行
    break;
```

---

## 7. 两种脏页模式对比

| 特性 | Dirty Bitmap | Dirty Ring |
|------|-------------|------------|
| 内核接口 | `KVM_GET_DIRTY_LOG` | `KVM_CAP_DIRTY_LOG_RING` |
| 粒度 | per-slot bitmap | per-vCPU ring buffer |
| 同步方式 | 主动查询（全 slot 扫描）| 被动通知（ring entry）|
| 延迟 | 高（批量查询）| 低（实时记录）|
| 迁移适配 | `log_sync` per-slot | `log_sync_global` 全局 |
| CPU 开销 | bitmap 写入原子操作 | ring 写入，满时 exit |
| 缩放性 | O(RAM大小) 扫描 | O(脏页数) 处理 |
| 适用场景 | 小内存/低脏页率 | 大内存/高脏页率 |

---

## 8. 脏页追踪与迁移联动

### 8.1 启动脏页追踪

```c
// 迁移开始时：
migration_start()
  → ram_save_setup()
    → memory_global_dirty_log_start(GLOBAL_DIRTY_MIGRATION)
      → MemoryListener::log_start()  (每个 region)
        → kvm_log_start()
          → kvm_section_update_flags(kml, section, true)
            → slot->flags |= KVM_MEM_LOG_DIRTY_PAGES
            → kvm_set_user_memory_region(kml, slot, false)  // 更新 flags
            → kvm_slot_init_dirty_bitmap(slot)
```

### 8.2 同步脏页（迭代阶段）

```
迁移迭代:
  ram_save_iterate()
    → memory_global_dirty_log_sync()
      → MemoryListener::log_sync / log_sync_global
        → kvm_log_sync() [bitmap 模式]
          → kvm_physical_sync_dirty_bitmap()
            → KVM_GET_DIRTY_LOG → cpu_physical_memory_set_dirty_lebitmap()
        → kvm_log_sync_global() [ring 模式]
          → kvm_dirty_ring_reap(s, NULL)  // 收割所有 CPU
          → 标记到 slot bitmap → 同步到 QEMU RAM bitmap
```

### 8.3 停止脏页追踪

```c
// 迁移完成：
migration_complete()
  → memory_global_dirty_log_stop(GLOBAL_DIRTY_MIGRATION)
    → kvm_log_stop()
      → slot->flags &= ~KVM_MEM_LOG_DIRTY_PAGES
      → kvm_set_user_memory_region(kml, slot, false)
```

---

## 9. ioeventfd — MMIO/PIO 快速路径

### 9.1 原理

普通 MMIO 流程（慢）：
```
Guest 写 MMIO → VM Exit → KVM_EXIT_MMIO → QEMU dispatch → 设备处理 → KVM_RUN
```

ioeventfd 快速路径：
```
Guest 写 MMIO → KVM 内核匹配地址 → 写 eventfd → 不退出！
                                      ↓ (异步)
                              QEMU eventfd handler → 设备处理
```

### 9.2 MMIO ioeventfd

```c
// accel/kvm/kvm-all.c:1519-1551
static int kvm_set_ioeventfd_mmio(int fd, hwaddr addr, uint32_t val,
                                   bool assign, uint32_t size, bool datamatch) {
    struct kvm_ioeventfd iofd = {
        .datamatch = datamatch ? val : 0,
        .addr = addr,                        // MMIO 地址
        .len = size,                         // 匹配大小
        .flags = KVM_IOEVENTFD_FLAG_MMIO,
        .fd = fd,                            // eventfd 描述符
    };
    if (datamatch) iofd.flags |= KVM_IOEVENTFD_FLAG_DATAMATCH;
    if (!assign) iofd.flags |= KVM_IOEVENTFD_FLAG_DEASSIGN;

    return kvm_vm_ioctl(s, KVM_IOEVENTFD, &iofd);
}
```

### 9.3 PIO ioeventfd

```c
// accel/kvm/kvm-all.c:1553-1579
static int kvm_set_ioeventfd_pio(int fd, uint16_t addr, uint16_t val,
                                  bool assign, uint32_t size, bool datamatch) {
    struct kvm_ioeventfd iofd = {
        .datamatch = datamatch ? val : 0,
        .addr = addr,
        .len = size,
        .flags = 0,  // 没有 MMIO flag = PIO
        .fd = fd,
    };
    // ...
    return kvm_vm_ioctl(s, KVM_IOEVENTFD, &iofd);
}
```

### 9.4 MemoryListener 注册

```c
// accel/kvm/kvm-all.c:2017-2087
static void kvm_mem_ioeventfd_add(MemoryListener *listener, MemoryRegionSection *s,
                                   bool match_data, uint64_t data, EventNotifier *e) {
    int fd = event_notifier_get_fd(e);
    kvm_set_ioeventfd_mmio(fd, section_addr, data, true, int128_get64(s->size), match_data);
}

static void kvm_io_ioeventfd_add(MemoryListener *listener, MemoryRegionSection *s,
                                  bool match_data, uint64_t data, EventNotifier *e) {
    int fd = event_notifier_get_fd(e);
    kvm_set_ioeventfd_pio(fd, section_addr, data, true, int128_get64(s->size), match_data);
}
```

### 9.5 典型用途：VirtIO Doorbell

```
VirtIO 设备注册 ioeventfd:
  virtio_pci_ioeventfd_assign()
    → memory_region_add_eventfd(&proxy->bar, addr, 2, true, n, &vq->host_notifier)
    → 最终调用 kvm_set_ioeventfd_mmio()

Guest 写 virtqueue notify:
  Guest 写 BAR 地址 → KVM 匹配 → eventfd signal → virtio_queue_host_notifier_read()
  不产生 VM Exit！
```

---

## 10. irqfd — 中断快速路径

### 10.1 原理

普通中断注入（慢）：
```
设备产生中断 → QEMU 设置 pending → kick vCPU → vCPU exit → 注入中断 → KVM_RUN
```

irqfd 快速路径：
```
eventfd signal → KVM 内核直接注入中断 → 不需要 VM Exit/Re-enter！
```

### 10.2 核心函数

```c
// accel/kvm/kvm-all.c:2448-2492
static int kvm_irqchip_assign_irqfd(KVMState *s, EventNotifier *event,
                                     EventNotifier *resample, int virq, bool assign) {
    struct kvm_irqfd irqfd = {
        .fd = event_notifier_get_fd(event),
        .gsi = virq,                         // 全局中断号
        .flags = assign ? 0 : KVM_IRQFD_FLAG_DEASSIGN,
    };
    if (resample) {
        irqfd.flags |= KVM_IRQFD_FLAG_RESAMPLE;
        irqfd.resamplefd = event_notifier_get_fd(resample);
    }
    return kvm_vm_ioctl(s, KVM_IRQFD, &irqfd);
}
```

### 10.3 便捷包装

```c
// accel/kvm/kvm-all.c:2537-2570
int kvm_irqchip_add_irqfd_notifier_gsi(KVMState *s, EventNotifier *n,
                                         EventNotifier *rn, int virq) {
    return kvm_irqchip_assign_irqfd(s, n, rn, virq, true);
}

int kvm_irqchip_remove_irqfd_notifier_gsi(KVMState *s, EventNotifier *n, int virq) {
    return kvm_irqchip_assign_irqfd(s, n, NULL, virq, false);
}
```

---

## 11. MSI 路由

### 11.1 GSI 分配与路由表

```c
// accel/kvm/kvm-all.c:2362-2414
int kvm_irqchip_add_msi_route(KVMState *s, int vector, PCIDevice *dev) {
    // 1. 分配 GSI
    int virq = kvm_irqchip_get_virq(s);

    // 2. 填充路由条目
    struct kvm_irq_routing_entry kroute = {
        .gsi = virq,
        .type = KVM_IRQ_ROUTING_MSI,
        .u.msi = {
            .address_lo = msg.address & 0xFFFFFFFF,
            .address_hi = msg.address >> 32,
            .data = msg.data,
            .devid = pci_requester_id(dev),  // ARM: device ID
        },
    };

    // 3. 添加到路由表
    kvm_add_routing_entry(s, &kroute);

    // 4. 提交到内核
    kvm_irqchip_commit_routes(s);  // KVM_SET_GSI_ROUTING
    return virq;
}
```

### 11.2 路由提交

```c
// accel/kvm/kvm-all.c:2188-2204
void kvm_irqchip_commit_routes(KVMState *s) {
    // 将完整路由表一次性写入 KVM
    kvm_vm_ioctl(s, KVM_SET_GSI_ROUTING, s->irq_routes);
}
```

---

## 12. VirtIO 全快速路径

当 ioeventfd + irqfd 同时启用时，VirtIO 实现**零 VM Exit** 的 I/O 路径：

```
┌─────────────────────────────────────────────────────┐
│  Guest                                               │
│  1. 填写 virtqueue descriptor                        │
│  2. 写 notify MMIO → ioeventfd（不 Exit）            │
└────────────────────────┬────────────────────────────┘
                         │ eventfd signal
┌────────────────────────┼────────────────────────────┐
│  QEMU (dataplane)      ↓                            │
│  3. virtio_queue_host_notifier_read()               │
│  4. 处理 I/O 请求                                    │
│  5. 完成后 → event_notifier_set(irqfd)              │
└────────────────────────┬────────────────────────────┘
                         │ eventfd signal
┌────────────────────────┼────────────────────────────┐
│  KVM 内核              ↓                            │
│  6. irqfd 触发 → 直接注入 MSI 到 vGIC               │
│  7. Guest 收到中断（不需要 VM Exit）                 │
└─────────────────────────────────────────────────────┘
```

**性能优势：**
- 普通路径：每次 I/O = 2 次 VM Exit（notify + interrupt inject）
- 快速路径：0 次 VM Exit（全部在内核完成 eventfd → interrupt 映射）

---

## 13. Memory Attributes (新特性)

### 13.1 概述

`KVM_SET_MEMORY_ATTRIBUTES` 是为**机密计算 (CoCo)** 引入的：

```c
// accel/kvm/kvm-all.c:1602-1630
static int kvm_set_memory_attributes(hwaddr start, uint64_t size, uint64_t attr) {
    struct kvm_memory_attributes attrs = {
        .address = start,
        .size = size,
        .attributes = attr,   // KVM_MEMORY_ATTRIBUTE_PRIVATE
    };
    assert((attr & kvm_supported_memory_attributes) == attr);
    return kvm_vm_ioctl(s, KVM_SET_MEMORY_ATTRIBUTES, &attrs);
}

void kvm_set_memory_attributes_private(hwaddr start, uint64_t size) {
    kvm_set_memory_attributes(start, size, KVM_MEMORY_ATTRIBUTE_PRIVATE);
}

void kvm_set_memory_attributes_shared(hwaddr start, uint64_t size) {
    kvm_set_memory_attributes(start, size, 0);  // 清除 private
}
```

### 13.2 guest-memfd

```c
// slot 创建时 (kvm_set_phys_mem):
if (mr 有 guest_memfd) {
    slot->flags |= KVM_MEM_GUEST_MEMFD;
    slot->guest_memfd = mr->ram_block->guest_memfd;
    slot->guest_memfd_offset = ...;
    kvm_set_memory_attributes_private(slot->start_addr, slot->memory_size);
}
```

用于 AMD SEV-SNP / Intel TDX 等 CoCo 方案，Guest 内存对 Host 不可见。

### 13.3 Pre-fault Memory

```c
// accel/kvm/kvm-all.c:3071
kvm_pre_fault_memory_supported = kvm_vm_check_extension(s, KVM_CAP_PRE_FAULT_MEMORY);
```

允许在 VM 启动前预分配 Stage-2 映射，减少首次访问的 page fault 延迟。

---

## 14. 与硬件对比：ARM64 Stage-2 页表

### 14.1 KVM memslot → Stage-2 映射

| QEMU 概念 | KVM 内核操作 | ARM64 硬件 |
|-----------|-------------|-----------|
| `kvm_set_user_memory_region(GPA, HVA, size)` | 记录 memslot | — |
| Guest 首次访问 GPA | Stage-2 fault | VTTBR_EL2 指向的页表 |
| KVM 处理 fault | 填充 Stage-2 PTE | IPA → PA 映射建立 |
| dirty page tracking | Stage-2 PTE dirty bit / write protect | S2AP / DBM |

### 14.2 MMIO 映射

- MMIO 地址**不在 memslot 中** → Stage-2 页表无对应项
- Guest 访问 → Stage-2 translation fault → EL2 trap → KVM_EXIT_MMIO
- QEMU 处理后 resume → KVM re-enter guest

### 14.3 脏页追踪硬件支持

| 方案 | ARM64 硬件支持 | 描述 |
|------|---------------|------|
| Write-protect | 清除 Stage-2 S2AP write 位 | 写触发 permission fault |
| Hardware DBM | ARMv8.1 FEAT_HAFDBS | Stage-2 PTE 自动置 dirty bit |
| Dirty Ring | 基于上述机制 | KVM 记录到 ring buffer |

### 14.4 QEMU vs 真实硬件差异

| 方面 | 真实硬件 (EL2) | QEMU/KVM |
|------|---------------|----------|
| Stage-2 页表管理 | Hypervisor 直接操作 VTTBR_EL2 | KVM 内核完成 |
| TLB 维护 | TLBI VMALLE1IS 等指令 | KVM 自动 invalidate |
| 内存属性 | MAIR_EL2 / Stage-2 MemAttr | memslot flags 间接控制 |
| 大页支持 | 2MB/1GB blocks | KVM 自动合并 (THP) |
| MMIO 优化 | 无（必须 trap）| ioeventfd 避免 trap |

---

## 附录 A: ioctl 调用层次

```
QEMU                          KVM 内核
────────────────────────      ─────────────────────
kvm_vm_ioctl(KVM_SET_USER_MEMORY_REGION2)
  → 创建/更新/删除 memslot    → kvm_set_memory_region()
                               → __kvm_set_memory_region()
                               → ARM: kvm_arch_flush_shadow_memslot()

kvm_vm_ioctl(KVM_GET_DIRTY_LOG)
  → 获取脏页 bitmap            → kvm_vm_ioctl_get_dirty_log()
                               → kvm_get_dirty_log()

kvm_vm_ioctl(KVM_IOEVENTFD)
  → 注册 eventfd              → kvm_ioeventfd()
                               → kvm_assign_ioeventfd_idx()
                               → ARM: 匹配 MMIO 地址

kvm_vm_ioctl(KVM_IRQFD)
  → 注册 irqfd                → kvm_irqfd()
                               → kvm_irqfd_assign()
                               → ARM: 绑定到 vGIC GSI

kvm_vm_ioctl(KVM_SET_GSI_ROUTING)
  → MSI 路由表                 → kvm_set_irq_routing()
                               → ARM: 更新 vGIC 路由

kvm_vm_ioctl(KVM_SET_MEMORY_ATTRIBUTES)
  → 设置内存属性               → kvm_vm_set_mem_attributes()
                               → 标记 private/shared
```

---

## 附录 B: 源码文件索引

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| `accel/kvm/kvm-all.c` | 4786 | KVM 核心：memslot、dirty、ioeventfd、irqfd、运行循环 |
| `include/system/kvm_int.h` | ~200 | KVMSlot、KVMMemoryListener、KVMState 定义 |
| `include/system/kvm.h` | 610 | KVM 公共 API 声明 |
| `accel/kvm/kvm-accel-ops.c` | ~110 | vCPU 线程模型 |

---

## 附录 C: 关键数据流

```
添加 RAM:
  memory_region_init_ram() → flatview 更新
    → kvm_region_add() [入队 transaction_add]
    → kvm_region_commit()
      → kvm_set_phys_mem(add=true)
        → kvm_alloc_slot()
        → kvm_set_user_memory_region(slot, new=true)
          → KVM_SET_USER_MEMORY_REGION2 ioctl
        → KVM 填充 Stage-2 (on demand)

迁移脏页同步:
  ram_save_iterate()
    → memory_global_dirty_log_sync()
      → [bitmap] kvm_physical_sync_dirty_bitmap()
          → KVM_GET_DIRTY_LOG → slot->dirty_bmap → QEMU RAM bitmap
      → [ring] kvm_dirty_ring_reap()
          → 遍历所有 CPU dirty ring → slot->dirty_bmap → QEMU RAM bitmap
    → find_dirty_block() → ram_save_host_page() → 发送脏页
```
