# QEMU DMA 设备模拟架构深度分析

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 分析范围：DMA 核心框架、AddressSpace/IOMMU 与 DMA 的交互、PCI/SysBus 设备 DMA 路径、具体设备 DMA 示例（e1000/AHCI/virtio-blk）、ARM DMA 控制器（PL330/PL080）、缓存一致性  
> 关联文档：[内存子系统深度分析](../memory/00-内存子系统深度分析.md) · [VFIO设备直通与IOMMU集成](05-VFIO设备直通与IOMMU集成深度分析.md) · [设备模型与virtio深度分析](00-设备模型与virtio深度分析.md)  
> 关键提交：`8aed841056` (arch_init.h rename) · `c77da4c0d6` (virtio-blk loadparm)

---

## 目录

1. [源码规模与文件布局](#1-源码规模与文件布局)
2. [DMA 模拟总体架构](#2-dma-模拟总体架构)
3. [DMA 核心 API](#3-dma-核心-api)
4. [QEMUSGList — 散列聚合列表](#4-qemusglist--散列聚合列表)
5. [dma_memory_read/write — DMA 内存访问](#5-dma_memory_readwrite--dma-内存访问)
6. [dma_memory_map/unmap — DMA 映射](#6-dma_memory_mapunmap--dma-映射)
7. [dma_blk_io — 块设备 DMA](#7-dma_blk_io--块设备-dma)
8. [AddressSpace 与 DMA 的关系](#8-addressspace-与-dma-的关系)
9. [Bounce Buffer 机制](#9-bounce-buffer-机制)
10. [IOMMU 在 DMA 路径中的角色](#10-iommu-在-dma-路径中的角色)
11. [PCI 设备 DMA 框架](#11-pci-设备-dma-框架)
12. [Bus Master Enable (BME) 门控](#12-bus-master-enable-bme-门控)
13. [SysBus 设备 DMA 框架](#13-sysbus-设备-dma-框架)
14. [virtio 设备 DMA 特殊性](#14-virtio-设备-dma-特殊性)
15. [设备示例：e1000 网卡 DMA](#15-设备示例e1000-网卡-dma)
16. [设备示例：AHCI/SATA DMA 引擎](#16-设备示例ahcisata-dma-引擎)
17. [设备示例：virtio-blk DMA 路径](#17-设备示例virtio-blk-dma-路径)
18. [ARM PL330 DMA 控制器](#18-arm-pl330-dma-控制器)
19. [ARM PL080 DMA 控制器](#19-arm-pl080-dma-控制器)
20. [hw/dma/ 目录全景](#20-hwdma-目录全景)
21. [ARM virt 机器 DMA 路径](#21-arm-virt-机器-dma-路径)
22. [DMA 与缓存一致性](#22-dma-与缓存一致性)
23. [端到端 DMA 流程图](#23-端到端-dma-流程图)

**附录**
- [A. 关键数据结构速查表](#附录-a-关键数据结构速查表)
- [B. 源码文件索引](#附录-b-源码文件索引)
- [C. 关联文档索引](#附录-c-关联文档索引)

---

## 1. 源码规模与文件布局

DMA 相关核心文件约 9,500 行，加上使用 DMA 的设备代码（e1000、AHCI 等）总量更大：

| 文件 | 行数 | 职责 |
|------|------|------|
| `hw/ide/ahci.c` | 1,813 | AHCI/SATA DMA 引擎 |
| `hw/net/e1000.c` | 1,767 | e1000 网卡 DMA |
| `hw/dma/pl330.c` | 1,694 | ARM PL330 DMA 控制器 |
| `hw/dma/xlnx-zdma.c` | 842 | Xilinx ZynqMP DMA |
| `hw/dma/xilinx_axidma.c` | 690 | Xilinx AXI DMA |
| `hw/dma/i8257.c` | 659 | ISA DMA 控制器 |
| `hw/dma/pl080.c` | 471 | ARM PL080/PL081 DMA |
| `hw/dma/sparc32_dma.c` | 455 | SPARC DMA |
| `hw/dma/bcm2835_dma.c` | 410 | Raspberry Pi DMA |
| `system/dma-helpers.c` | 347 | DMA 辅助函数 |
| `include/system/dma.h` | 322 | DMA 核心头文件 |

---

## 2. DMA 模拟总体架构

QEMU 中 DMA 是一个**多层抽象**的体系：

```
┌──────────────────────────────────────────────────┐
│                  设备层 (Device)                  │
│  e1000: pci_dma_read()    virtio: vring desc     │
│  AHCI:  qemu_sglist_add() PL330:  instruction    │
├──────────────────────────────────────────────────┤
│             DMA 辅助层 (dma-helpers)              │
│  dma_memory_read/write()   dma_blk_io()          │
│  dma_memory_map/unmap()    dma_buf_read/write()  │
│  QEMUSGList 管理                                  │
├──────────────────────────────────────────────────┤
│          地址空间层 (AddressSpace)                 │
│  address_space_rw()        address_space_map()    │
│  flatview_translate()                             │
├──────────────────────────────────────────────────┤
│          IOMMU 翻译层 (可选)                      │
│  IOMMUMemoryRegion.translate()                    │
│  IOVA → GPA 翻译                                 │
├──────────────────────────────────────────────────┤
│          物理内存层 (MemoryRegion)                 │
│  RAM 直接访问 / MMIO dispatch                     │
└──────────────────────────────────────────────────┘
```

核心设计原则：
- **每个设备有自己的 AddressSpace**：PCI 设备通过 `bus_master_as`，SysBus 设备通常直接使用 `address_space_memory`
- **IOMMU 透明插入**：当存在 IOMMU 时，设备的 DMA AddressSpace 自动经过 IOMMU 翻译
- **统一 API**：所有 DMA 操作最终归结为 `address_space_rw()` / `address_space_map()`

---

## 3. DMA 核心 API

### 3.1 关键类型定义

```c
// dma.h:18-21
typedef enum {
    DMA_DIRECTION_TO_DEVICE = 0,    // 内存→设备（设备读取）
    DMA_DIRECTION_FROM_DEVICE = 1,  // 设备→内存（设备写入）
} DMADirection;

// dma.h:30
typedef uint64_t dma_addr_t;        // DMA 地址类型
```

### 3.2 API 总览

```c
// dma.h:124-305 — 核心 API 声明

// 内存读写
dma_memory_read(AddressSpace *as, dma_addr_t addr, void *buf, dma_addr_t len, ...)
dma_memory_write(AddressSpace *as, dma_addr_t addr, const void *buf, dma_addr_t len, ...)
dma_memory_rw(AddressSpace *as, dma_addr_t addr, void *buf, dma_addr_t len, DMADirection dir, ...)
dma_memory_set(AddressSpace *as, dma_addr_t addr, uint8_t c, dma_addr_t len, ...)

// 内存映射
dma_memory_map(AddressSpace *as, dma_addr_t addr, dma_addr_t *len, DMADirection dir, ...)
dma_memory_unmap(AddressSpace *as, void *buffer, dma_addr_t len, DMADirection dir, dma_addr_t access_len)

// 散列聚合
qemu_sglist_init(QEMUSGList *qsg, DeviceState *dev, int alloc_hint, AddressSpace *as)
qemu_sglist_add(QEMUSGList *qsg, dma_addr_t base, dma_addr_t len)
qemu_sglist_destroy(QEMUSGList *qsg)

// 块设备 DMA
dma_blk_io(AioContext *ctx, QEMUSGList *sg, uint64_t offset, bool to_dev, ...)
dma_blk_read(BlockBackend *blk, QEMUSGList *sg, uint64_t offset, ...)
dma_blk_write(BlockBackend *blk, QEMUSGList *sg, uint64_t offset, ...)

// 缓冲区 DMA
dma_buf_read(void *ptr, int32_t len, QEMUSGList *sg, ...)
dma_buf_write(void *ptr, int32_t len, QEMUSGList *sg, ...)
```

---

## 4. QEMUSGList — 散列聚合列表

### 4.1 数据结构

```c
// dma.h:279-287
typedef struct ScatterGatherEntry {
    dma_addr_t base;                    // DMA 起始地址
    dma_addr_t len;                     // 长度
} ScatterGatherEntry;

typedef struct QEMUSGList {
    ScatterGatherEntry *sg;             // 条目数组
    int nsg;                            // 当前条目数
    int nalloc;                         // 已分配容量
    size_t size;                        // 总数据大小
    DeviceState *dev;                   // 关联设备
    AddressSpace *as;                   // DMA 地址空间
} QEMUSGList;
```

### 4.2 操作函数

```c
// dma-helpers.c:29-58
void qemu_sglist_init(QEMUSGList *qsg, DeviceState *dev,
                      int alloc_hint, AddressSpace *as) {
    qsg->sg = g_new(ScatterGatherEntry, alloc_hint);
    qsg->nsg = 0;
    qsg->nalloc = alloc_hint;
    qsg->size = 0;
    qsg->dev = dev;
    qsg->as = as;
}

void qemu_sglist_add(QEMUSGList *qsg, dma_addr_t base, dma_addr_t len) {
    // 动态扩展数组
    if (qsg->nsg == qsg->nalloc) {
        qsg->nalloc = 2 * qsg->nalloc + 1;
        qsg->sg = g_renew(ScatterGatherEntry, qsg->sg, qsg->nalloc);
    }
    qsg->sg[qsg->nsg].base = base;
    qsg->sg[qsg->nsg].len = len;
    qsg->nsg++;
    qsg->size += len;
}

void qemu_sglist_destroy(QEMUSGList *qsg) {
    g_free(qsg->sg);
    memset(qsg, 0, sizeof(*qsg));
}
```

### 4.3 使用模式

```
设备（如 AHCI）读取 guest 提供的描述符表
        │
        v
qemu_sglist_init(sg, dev, hint, pci_get_address_space(dev))
        │
        v
for each descriptor:
    qemu_sglist_add(sg, desc.base_addr, desc.byte_count)
        │
        v
dma_blk_read(blk, sg, offset, ...)  ← 按 SGList 从磁盘读到 guest 内存
        │
        v
qemu_sglist_destroy(sg)
```

---

## 5. dma_memory_read/write — DMA 内存访问

### 5.1 实现原理

```c
// dma.h:124-173 — inline 实现
static inline MemTxResult dma_memory_rw(AddressSpace *as, dma_addr_t addr,
                                        void *buf, dma_addr_t len,
                                        DMADirection dir, MemTxAttrs attrs) {
    // 直接委托给 address_space_rw()
    return address_space_rw(as, addr, attrs, buf, len,
                            dir == DMA_DIRECTION_FROM_DEVICE);
}

static inline MemTxResult dma_memory_read(AddressSpace *as, dma_addr_t addr,
                                          void *buf, dma_addr_t len,
                                          MemTxAttrs attrs) {
    return dma_memory_rw(as, addr, buf, len,
                         DMA_DIRECTION_TO_DEVICE, attrs);
}

static inline MemTxResult dma_memory_write(AddressSpace *as, dma_addr_t addr,
                                           const void *buf, dma_addr_t len,
                                           MemTxAttrs attrs) {
    return dma_memory_rw(as, addr, (void *)buf, len,
                         DMA_DIRECTION_FROM_DEVICE, attrs);
}
```

### 5.2 调用链

```
dma_memory_read(as, addr, buf, len, attrs)
    │
    └── address_space_rw(as, addr, attrs, buf, len, is_write=false)
        │
        └── flatview_rw(fv, addr, attrs, buf, len, is_write)
            │
            ├── flatview_translate(fv, addr, ...) — 查 FlatView 找到 MemoryRegion
            │   └── [若 MR 是 IOMMUMemoryRegion] → iommu_translate() → 获取 HPA
            │
            ├── [RAM] → memcpy 直接拷贝
            └── [MMIO] → memory_region_dispatch_read/write()
```

---

## 6. dma_memory_map/unmap — DMA 映射

### 6.1 映射接口

```c
// dma.h:205-237
static inline void *dma_memory_map(AddressSpace *as, dma_addr_t addr,
                                   dma_addr_t *len, DMADirection dir,
                                   MemTxAttrs attrs) {
    // 委托给 address_space_map()
    // 返回指向 guest RAM 的 host 虚拟地址指针
    return address_space_map(as, addr, (hwaddr *)len,
                             dir == DMA_DIRECTION_FROM_DEVICE, attrs);
}

static inline void dma_memory_unmap(AddressSpace *as, void *buffer,
                                    dma_addr_t len, DMADirection dir,
                                    dma_addr_t access_len) {
    address_space_unmap(as, buffer, (hwaddr)len,
                        dir == DMA_DIRECTION_FROM_DEVICE, access_len);
}
```

### 6.2 映射 vs 读写对比

| 方式 | 适用场景 | 拷贝次数 | 特点 |
|------|---------|---------|------|
| `dma_memory_read/write` | 小块数据、MMIO 区域 | 1次（buf↔guest） | 简单安全 |
| `dma_memory_map/unmap` | 大块 RAM 数据 | 0次（直接指针） | 高性能，需 unmap |

---

## 7. dma_blk_io — 块设备 DMA

### 7.1 核心实现

```c
// dma-helpers.c:214-275
// dma_blk_io() 将 SGList 转化为对 BlockBackend 的 I/O 操作

BlockAIOCB *dma_blk_io(AioContext *ctx, QEMUSGList *sg,
                       uint64_t offset, uint32_t align,
                       DMAIOFunc *io_func, void *io_func_opaque,
                       BlockCompletionFunc *cb, void *opaque,
                       DMADirection dir) {
    DMAAIOCB *dbs = ...;

    // 遍历 SGList 的每个条目：
    // 1. dma_memory_map() 获取 host 指针
    // 2. 构造 QEMUIOVector (iov)
    // 3. 调用 io_func (即 blk_aio_preadv/pwritev) 提交给块层
    // 4. 完成回调中 dma_memory_unmap()
}

// dma-helpers.c:257-275
BlockAIOCB *dma_blk_read(BlockBackend *blk, QEMUSGList *sg,
                         uint64_t offset, uint32_t align,
                         void (*cb)(void *), void *opaque) {
    return dma_blk_io(blk_get_aio_context(blk), sg, offset, align,
                      dma_blk_read_io_func, blk, cb, opaque,
                      DMA_DIRECTION_FROM_DEVICE);
}

BlockAIOCB *dma_blk_write(BlockBackend *blk, QEMUSGList *sg,
                          uint64_t offset, uint32_t align,
                          void (*cb)(void *), void *opaque) {
    return dma_blk_io(blk_get_aio_context(blk), sg, offset, align,
                      dma_blk_write_io_func, blk, cb, opaque,
                      DMA_DIRECTION_TO_DEVICE);
}
```

### 7.2 dma_buf_read/write

```c
// dma-helpers.c:278-315
// 用于在 host buffer 和 guest SGList 之间复制数据
// 常用于 IDE/SCSI 命令完成后的数据传输

dma_buf_read(void *ptr, int32_t len, QEMUSGList *sg, ...) {
    // 从 ptr 写到 sg 描述的 guest 内存区域
    dma_buf_rw(ptr, len, sg, DMA_DIRECTION_FROM_DEVICE, ...);
}

dma_buf_write(void *ptr, int32_t len, QEMUSGList *sg, ...) {
    // 从 sg 描述的 guest 内存区域读到 ptr
    dma_buf_rw(ptr, len, sg, DMA_DIRECTION_TO_DEVICE, ...);
}
```

---

## 8. AddressSpace 与 DMA 的关系

### 8.1 地址翻译路径

```c
// physmem.c:3419-3457 — address_space_rw 核心路径
MemTxResult address_space_rw(AddressSpace *as, hwaddr addr,
                             MemTxAttrs attrs, void *buf,
                             hwaddr len, bool is_write) {
    // 1. 获取当前 FlatView (RCU 保护)
    FlatView *fv = address_space_to_flatview(as);

    // 2. flatview_translate() 查找目标 MemoryRegion
    //    若遇到 IOMMUMemoryRegion → 调用 translate() 回调
    MemoryRegion *mr = flatview_translate(fv, addr, &xlat, &l, is_write, attrs);

    // 3. 根据 MR 类型执行操作
    if (memory_access_is_direct(mr, is_write)) {
        // RAM → 直接 memcpy
        ptr = qemu_map_ram_ptr(mr->ram_block, xlat);
        memcpy(ptr, buf, l);  // 或反向
    } else {
        // MMIO → dispatch
        memory_region_dispatch_read/write(mr, xlat, ...);
    }
}
```

### 8.2 IOMMU 翻译插入点

```c
// physmem.c:2753-2850
// 当 flatview_translate() 发现 MemoryRegion 是 IOMMUMemoryRegion 时：
address_space_translate_iommu(IOMMUMemoryRegion *iommu_mr,
                              hwaddr *xlat, hwaddr *plen, ...) {
    // 1. 调用 IOMMU 的 translate() 方法
    IOMMUTLBEntry iotlb = imrc->translate(iommu_mr, addr, flag, iommu_idx);

    // 2. 获得翻译结果：target_as + translated_addr + perm
    // 3. 用 translated_addr 继续在 target_as 中查找
    // 4. 递归直到找到最终 MemoryRegion
}
```

---

## 9. Bounce Buffer 机制

### 9.1 何时使用

```c
// physmem.c:3705-3766 — address_space_map()
void *address_space_map(AddressSpace *as, hwaddr addr,
                        hwaddr *plen, bool is_write, MemTxAttrs attrs) {
    mr = flatview_translate(fv, addr, &xlat, &l, is_write, attrs);

    if (memory_access_is_direct(mr, is_write)) {
        // 情况1: RAM — 直接返回 host 指针（零拷贝）
        return qemu_ram_ptr_length(mr->ram_block, xlat, plen, true);
    }

    // 情况2: MMIO 或不可直接映射 — 分配 bounce buffer
    bounce.buffer = qemu_memalign(TARGET_PAGE_SIZE, l);
    bounce.addr = addr;
    bounce.len = l;
    bounce.mr = mr;

    if (!is_write) {
        // 读方向：先将 MMIO 数据读入 bounce buffer
        flatview_read(fv, addr, attrs, bounce.buffer, l);
    }
    return bounce.buffer;
}
```

### 9.2 unmap 时回写

```c
// physmem.c:3773-3807 — address_space_unmap()
void address_space_unmap(AddressSpace *as, void *buffer, hwaddr len,
                         bool is_write, hwaddr access_len) {
    if (buffer != bounce.buffer) {
        // 直接映射的 RAM — 只需标脏页
        if (is_write) {
            invalidate_and_set_dirty(mr, ...);
        }
        return;
    }

    // Bounce buffer 路径：
    if (is_write) {
        // 将 bounce buffer 写回 MMIO
        flatview_write(fv, bounce.addr, MEMTXATTRS_UNSPECIFIED,
                       bounce.buffer, access_len);
    }
    // 释放 bounce buffer
    qemu_vfree(bounce.buffer);
}
```

### 9.3 性能影响

```
直接映射 (RAM):   设备 ←→ guest RAM（零拷贝，仅指针传递）
Bounce Buffer:    设备 ←→ bounce buf ←→ MMIO dispatch（额外拷贝+分配）

大多数 DMA 目标都是 RAM，因此 bounce buffer 路径很少触发。
主要发生在：设备试图 DMA 到另一个设备的 MMIO 空间（罕见场景）。
```

---

## 10. IOMMU 在 DMA 路径中的角色

### 10.1 IOMMUMemoryRegion

```c
// memory.h:46-49
struct IOMMUMemoryRegion {
    MemoryRegion parent_obj;
    // IOMMU 类型的特殊内存区域
    // 所有发往此区域的访问都先经过 translate() 翻译
};

// memory.h:430-542 — IOMMUMemoryRegionClass
struct IOMMUMemoryRegionClass {
    // 核心翻译回调
    IOMMUTLBEntry (*translate)(IOMMUMemoryRegion *iommu,
                               hwaddr addr, IOMMUAccessFlags flag,
                               int iommu_idx);
    // 通知器管理
    void (*notify_flag_changed)(IOMMUMemoryRegion *iommu,
                                IOMMUNotifierFlag old,
                                IOMMUNotifierFlag new, Error **);
    // 回放已有映射
    void (*replay)(IOMMUMemoryRegion *iommu, IOMMUNotifier *notifier);
    // IOTLB 支持
    int (*get_attr)(IOMMUMemoryRegion *iommu, enum IOMMUMemoryRegionAttr, void *);
    int (*attrs_to_index)(IOMMUMemoryRegion *iommu, MemTxAttrs attrs);
    int (*num_indexes)(IOMMUMemoryRegion *iommu);
};
```

### 10.2 翻译结果

```c
// memory.h:147-154
typedef struct IOMMUTLBEntry {
    AddressSpace *target_as;            // 翻译后的目标地址空间
    hwaddr iova;                        // 输入 IOVA
    hwaddr translated_addr;             // 翻译后的物理地址
    hwaddr addr_mask;                   // 地址掩码（页大小）
    IOMMUAccessFlags perm;              // 读写权限
} IOMMUTLBEntry;
```

### 10.3 DMA 路径中的 IOMMU

```
设备发起 DMA（使用 IOVA）
        │
        v
dma_memory_read(dev->dma_as, iova, ...)
        │
        v
address_space_rw(dev->dma_as, iova, ...)
        │
        v
flatview_translate()
        │
        ├── 找到 IOMMUMemoryRegion
        │
        v
address_space_translate_iommu()
        │
        v
iommu_mr->translate(iommu_mr, iova, ...)    ← SMMUv3/virtio-iommu
        │
        v
IOMMUTLBEntry { target_as=&address_space_memory, translated_addr=GPA }
        │
        v
继续在 target_as 中查找 → RAM / MMIO
```

### 10.4 IOMMU 通知器

```c
// memory.h:186-215
typedef struct IOMMUNotifier {
    void (*notify)(IOMMUNotifier *notifier, IOMMUTLBEntry *data);
    IOMMUNotifierFlag notifier_flags;   // MAP / UNMAP / DEVIOTLB_UNMAP
    hwaddr start;                       // 监听的 IOVA 范围
    hwaddr end;
    int iommu_idx;
    QLIST_ENTRY(IOMMUNotifier) node;
} IOMMUNotifier;

// memory.c:1987-2000
void memory_region_notify_iommu(IOMMUMemoryRegion *iommu_mr,
                                int iommu_idx,
                                IOMMUTLBEntry entry) {
    // 遍历所有注册的 notifier
    // 通知 VFIO / vhost 等使用者 IOMMU 映射已变化
    IOMMU_NOTIFIER_FOREACH(n, iommu_mr) {
        if (n->notifier_flags & relevant_flag) {
            n->notify(n, &entry);
        }
    }
}
```

---

## 11. PCI 设备 DMA 框架

### 11.1 PCIDevice 中的 DMA 字段

```c
// pci_device.h:60-99
struct PCIDevice {
    ...
    AddressSpace bus_master_as;         // DMA 地址空间
    MemoryRegion bus_master_enable_region; // BME 门控区域
    ...
};
```

### 11.2 PCI DMA 便利接口

```c
// pci_device.h:237-300 — static inline 便利函数

// 获取设备 DMA 地址空间
static inline AddressSpace *pci_get_address_space(PCIDevice *dev) {
    return &dev->bus_master_as;         // pci_device.h:237-241
}

// PCI DMA 读写
static inline MemTxResult pci_dma_rw(PCIDevice *dev, dma_addr_t addr,
                                     void *buf, dma_addr_t len,
                                     DMADirection dir, MemTxAttrs attrs) {
    return dma_memory_rw(pci_get_address_space(dev),
                         addr, buf, len, dir, attrs);
}

static inline MemTxResult pci_dma_read(PCIDevice *dev, dma_addr_t addr,
                                       void *buf, dma_addr_t len) {
    return pci_dma_rw(dev, addr, buf, len, DMA_DIRECTION_TO_DEVICE,
                      MEMTXATTRS_UNSPECIFIED);
}

static inline MemTxResult pci_dma_write(PCIDevice *dev, dma_addr_t addr,
                                        const void *buf, dma_addr_t len) {
    return pci_dma_rw(dev, addr, (void *)buf, len, DMA_DIRECTION_FROM_DEVICE,
                      MEMTXATTRS_UNSPECIFIED);
}
```

### 11.3 PCI DMA 地址空间初始化

```c
// pci.c:137-162
static void pci_init_bus_master(PCIDevice *pci_dev) {
    // 1. 获取设备的 IOMMU 地址空间（若有 IOMMU）
    AddressSpace *dma_as = pci_device_iommu_address_space(pci_dev);

    // 2. 创建 bus_master_enable_region（BME 门控）
    memory_region_init_alias(&pci_dev->bus_master_enable_region, ...
                             "bus master", dma_as->root, 0,
                             memory_region_size(dma_as->root));

    // 3. 初始化 bus_master_as
    address_space_init(&pci_dev->bus_master_as,
                       &pci_dev->bus_master_enable_region,
                       pci_dev->name);

    // 4. 默认禁用 Bus Master
    pci_set_master(pci_dev, false);
}
```

---

## 12. Bus Master Enable (BME) 门控

### 12.1 BME 控制逻辑

```c
// pci.c:131-135
void pci_set_master(PCIDevice *d, bool enable) {
    // 启用/禁用 bus_master_enable_region
    memory_region_set_enabled(&d->bus_master_enable_region, enable);
    d->is_master = enable;
}
```

### 12.2 工作原理

```
PCI Config Space                    DMA 路径
┌──────────────┐
│ Command Reg  │
│ Bit 2 = BME  │───── pci_set_master(dev, true/false)
└──────────────┘          │
                          v
              bus_master_enable_region
                    (MemoryRegion alias)
                          │
              ┌───────────┴───────────┐
              │ enabled=true          │ enabled=false
              │ DMA 正常访问          │ DMA 被阻断
              │ → bus_master_as       │ → 空 AddressSpace
              │ → IOMMU/sysmem       │ → 所有 DMA 返回错误
              └───────────────────────┘
```

Guest OS 通过写 PCI Command Register 的 Bus Master bit 来控制设备是否可以发起 DMA。这是 PCI 规范要求的安全机制。

---

## 13. SysBus 设备 DMA 框架

### 13.1 与 PCI 的区别

SysBus 设备没有统一的 DMA 地址空间管理接口。每个 DMA 控制器自行管理：

```c
// 典型模式（如 Xilinx AXI DMA）
// xilinx_axidma.c:561-593

typedef struct XilinxAXIDMA {
    SysBusDevice parent;
    MemoryRegion *dma_mr;               // 外部传入的 DMA 目标内存区域
    AddressSpace as;                     // 自建的 DMA 地址空间
    ...
};

static void xilinx_axidma_realize(DeviceState *dev, ...) {
    // 用传入的 dma_mr 初始化地址空间
    address_space_init(&s->as, s->dma_mr, "axidma-dma");
}

// DMA 传输时使用 s->as:
address_space_rw(&s->as, desc.buffer_address, ..., desc.length, ...);
```

### 13.2 对比

| 特性 | PCI 设备 | SysBus 设备 |
|------|---------|------------|
| DMA AS 获取 | `pci_get_address_space(dev)` | 自行 `address_space_init()` |
| IOMMU 集成 | 自动通过 `pci_device_iommu_address_space()` | 需手动配置 |
| BME 控制 | PCI 规范内建 | 无（或设备自定义） |
| 默认 AS | 经 PCI host bridge 的 AS | 通常直接 `address_space_memory` |

---

## 14. virtio 设备 DMA 特殊性

### 14.1 DMA 地址空间选择

```c
// virtio-bus.c:43-104
static void virtio_bus_device_plugged(DeviceState *d, Error **errp) {
    VirtIODevice *vdev = VIRTIO_DEVICE(d);

    // 默认使用系统内存地址空间
    vdev->dma_as = &address_space_memory;

    // 若设备声明了 VIRTIO_F_ACCESS_PLATFORM (iommu_platform)
    // 且传输层提供了 get_dma_as() 回调，则使用传输层的地址空间
    if (virtio_host_has_feature(vdev, VIRTIO_F_ACCESS_PLATFORM)) {
        VirtioBusClass *klass = VIRTIO_BUS_GET_CLASS(vdev->parent_bus);
        if (klass->get_dma_as) {
            vdev->dma_as = klass->get_dma_as(d);
            // 对于 virtio-pci: 返回 pci_get_address_space(pci_dev)
            // 这样 DMA 就经过 PCI IOMMU
        }
    }
}
```

### 14.2 Vring 描述符映射

```c
// virtio.c:1614-1659
static void virtqueue_map_desc(VirtIODevice *vdev, unsigned int *p_num_sg,
                               hwaddr *addr, struct iovec *iov, ...) {
    while (len) {
        // 使用设备的 DMA 地址空间映射 vring buffer
        iov[num_sg].iov_base = dma_memory_map(vdev->dma_as, pa, &l, dir, ...);
        iov[num_sg].iov_len = l;
        num_sg++;
        pa += l;
        len -= l;
    }
}
```

### 14.3 virtio DMA 与传统 DMA 的区别

```
=== 传统 PCI DMA (e1000) ===           === virtio DMA ===

设备硬件寄存器 → DMA 引擎               Guest driver → vring desc
    │                                       │
    v                                       v
设备发起 DMA 读写                        QEMU 读 vring desc 获取 GPA
pci_dma_read/write(dev, addr, ...)       virtqueue_map_desc(vdev, GPA, ...)
    │                                       │
    v                                       v
address_space_rw(bus_master_as, ...)      dma_memory_map(vdev->dma_as, GPA, ...)
    │                                       │
    v                                       v
[可选 IOMMU 翻译]                        [可选 IOMMU 翻译]
    │                                       │
    v                                       v
guest RAM                                guest RAM（返回 host 指针）
```

关键区别：
- **传统 DMA**：设备模拟代码主动调用 `pci_dma_read/write`
- **virtio DMA**：通过 vring 描述符间接指定 DMA 地址，QEMU 用 `dma_memory_map` 映射后构造 iov

---

## 15. 设备示例：e1000 网卡 DMA

### 15.1 TX DMA — 从 guest 读取发送数据

```c
// e1000.c:638-725 — process_tx_desc()
static void process_tx_desc(E1000State *s, struct e1000_tx_desc *dp) {
    // 1. 从 TX 描述符获取 buffer 地址和长度
    dma_addr_t addr = le64_to_cpu(dp->buffer_addr);
    uint32_t len = le16_to_cpu(dp->lower.flags.length);

    // 2. DMA 读取 guest 内存中的发送数据
    pci_dma_read(&s->parent_obj, addr, tp->data + tp->size, len);
    // 等价于 dma_memory_read(pci_get_address_space(dev), addr, ...)
}

// e1000.c:752-797 — start_xmit()
static void start_xmit(E1000State *s) {
    // 1. DMA 读取 TX 描述符环
    pci_dma_read(d, base + sizeof(desc) * s->tx.cur,
                 &desc, sizeof(desc));

    // 2. 处理每个描述符的数据
    process_tx_desc(s, &desc);

    // 3. DMA 回写描述符状态（DD bit）
    txdesc_writeback(s, base, &desc);
}

// e1000.c:727-741 — txdesc_writeback()
static void txdesc_writeback(E1000State *s, dma_addr_t base, ...) {
    // 回写 status byte（设置 DD = Descriptor Done）
    pci_dma_write(d, base + ..., &desc->upper.data, sizeof(desc->upper.data));
}
```

### 15.2 RX DMA — 向 guest 写入接收数据

```c
// e1000.c:873-1007 — e1000_receive_iov()
ssize_t e1000_receive_iov(NetClientState *nc,
                          const struct iovec *iov, int iovcnt) {
    // 1. DMA 读取 RX 描述符
    pci_dma_read(d, base, &desc, sizeof(desc));

    // 2. DMA 写入接收到的数据包到 guest 内存
    pci_dma_write(d, le64_to_cpu(desc.buffer_addr) + offset,
                  buf, size);

    // 3. DMA 回写描述符状态
    desc.status |= E1000_RXD_STAT_DD;
    pci_dma_write(d, base, &desc, sizeof(desc));

    // 4. 触发 RX 中断
    set_ics(s, 0, rxr_int);
}
```

### 15.3 e1000 DMA 完整流程

```
Guest Driver                    QEMU e1000               Host
    |                              |                       |
    +-- 设置 TX desc ring          |                       |
    |   desc[i].buffer_addr = GPA  |                       |
    |   desc[i].length = N         |                       |
    |                              |                       |
    +-- 写 TDT (Tail) 寄存器 ---->+                       |
    |                              |                       |
    |   start_xmit()               |                       |
    |   ├ pci_dma_read(desc ring)  |                       |
    |   ├ pci_dma_read(GPA, data)  |  ← 从 guest 读数据    |
    |   ├ qemu_send_packet(data) --|-->  TAP/网络后端       |
    |   └ pci_dma_write(DD bit)    |  ← 回写完成标志        |
    |                              |                       |
    |   set_ics(ICR_TXDW)          |                       |
    |   └ pci_set_irq() ---------->  中断                  |
    |                              |                       |
    +<-- Guest 收到 TX 完成中断     |                       |
```

---

## 16. 设备示例：AHCI/SATA DMA 引擎

### 16.1 PRDT 到 SGList 的转换

```c
// ahci.c:890-980 — ahci_populate_sglist()
static int ahci_populate_sglist(AHCIDevice *ad, QEMUSGList *sglist,
                                AHCICmdHdr *cmd, int offset) {
    // 1. 初始化 SGList
    qemu_sglist_init(sglist, DEVICE(ad), prdt_len,
                     pci_get_address_space(PCI_DEVICE(ad->hba)));

    // 2. 读取 PRDT (Physical Region Descriptor Table)
    for (i = 0; i < prdt_len; i++) {
        // DMA 映射 PRDT 条目
        prdt = dma_memory_map(ad->hba->as, prdt_addr, &prdt_len,
                              DMA_DIRECTION_TO_DEVICE, ...);

        // 3. 将每个 PRDT 条目添加到 SGList
        qemu_sglist_add(sglist,
                        le64_to_cpu(prdt[i].dba),      // 数据基地址
                        le32_to_cpu(prdt[i].dbc) + 1);  // 字节数

        dma_memory_unmap(ad->hba->as, prdt, ...);
    }
}
```

### 16.2 NCQ 命令 DMA 执行

```c
// ahci.c:1055-1084 — execute_ncq_command()
static void execute_ncq_command(AHCIDevice *ad, ...) {
    // 1. 构建 SGList
    ahci_populate_sglist(ad, &ncq_tfs->sglist, cmd, ...);

    // 2. 提交 DMA 块 I/O
    if (is_write) {
        // Guest → 磁盘（设备读取 guest 内存写入磁盘）
        dma_blk_write(ad->blk, &ncq_tfs->sglist, sector_offset, ...);
    } else {
        // 磁盘 → Guest（设备从磁盘读取写入 guest 内存）
        dma_blk_read(ad->blk, &ncq_tfs->sglist, sector_offset, ...);
    }
}
```

### 16.3 AHCI DMA 流程

```
Guest AHCI Driver                QEMU AHCI                    Block Layer
    |                              |                              |
    +-- 写入 Command Table         |                              |
    |   包含 CFIS + PRDT           |                              |
    |                              |                              |
    +-- 设置 Command Slot          |                              |
    |   cmd_hdr.ctba = GPA         |                              |
    |   cmd_hdr.prdtl = N          |                              |
    |                              |                              |
    +-- 写 PxCI 寄存器 ---------->+                              |
    |                              |                              |
    |   ahci_check_cmd_bh()        |                              |
    |   ├ dma_memory_map(cmd_hdr)  |                              |
    |   ├ dma_memory_map(PRDT)     |                              |
    |   ├ qemu_sglist_init()       |                              |
    |   ├ qemu_sglist_add() × N    |  ← 构建 SGList               |
    |   ├ dma_blk_read(sglist) ----|-->  blk_aio_preadv()         |
    |   |                          |      |                       |
    |   |                          |      +-- 从磁盘读数据         |
    |   |                          |      |                       |
    |   |  <-- 完成回调 ---------- |  <-- 数据写入 guest RAM       |
    |   ├ 设置 D2H FIS status      |                              |
    |   └ ahci_trigger_irq() ----> |  中断                        |
    |                              |                              |
    +<-- Guest 收到完成中断        |                              |
```

---

## 17. 设备示例：virtio-blk DMA 路径

### 17.1 请求处理

```c
// virtio-blk.c:821-901 — virtio_blk_handle_request()
static void virtio_blk_handle_request(VirtIOBlockReq *req, ...) {
    // 1. virtqueue_pop() 从 vring 获取请求描述符
    //    内部调用 virtqueue_map_desc() → dma_memory_map()
    //    将 vring buffer 映射为 host iov

    // 2. 解析请求头（out header: type, sector, ioprio）

    // 3. 构建 QEMUIOVector（从映射的 iov）

    // 4. 提交 I/O
}

// virtio-blk.c:215-265 — submit_requests()
static void submit_requests(VirtIOBlock *s, ...) {
    // 使用映射好的 iov 直接提交给块层
    if (is_write) {
        blk_aio_pwritev(s->blk, sector << BDRV_SECTOR_BITS,
                        &req->qiov, 0, virtio_blk_rw_complete, req);
    } else {
        blk_aio_preadv(s->blk, sector << BDRV_SECTOR_BITS,
                        &req->qiov, 0, virtio_blk_rw_complete, req);
    }
}
```

### 17.2 vring 映射细节

```c
// virtio.c:1614-1659 — virtqueue_map_desc()
// 将 vring 描述符中的 GPA 映射为 host 可访问的 iov

// virtio.c:850-873 — virtqueue_unmap_sg()
// I/O 完成后解映射：调用 dma_memory_unmap()
```

### 17.3 virtio-blk 完整路径

```
Guest virtio-blk Driver          QEMU virtio-blk            Block Backend
    |                              |                            |
    +-- 写入 vring desc            |                            |
    |   desc.addr = GPA (数据buf)  |                            |
    |   desc.len = N               |                            |
    |                              |                            |
    +-- kick (MMIO 写) ---------->+                            |
    |                              |                            |
    |   virtio_blk_handle_output() |                            |
    |   ├ virtqueue_pop()          |                            |
    |   │  └ virtqueue_map_desc()  |                            |
    |   │     └ dma_memory_map(    |                            |
    |   │         vdev->dma_as,    |                            |
    |   │         GPA, len)        |  ← 映射 guest buf 为 iov   |
    |   │                          |                            |
    |   ├ submit_requests()        |                            |
    |   │  └ blk_aio_preadv(iov) --|-->  qcow2/raw 读取         |
    |   │                          |      |                     |
    |   │  <-- complete callback   |  <-- 数据已在 guest RAM     |
    |   │                          |                            |
    |   ├ virtqueue_unmap_sg()     |  ← dma_memory_unmap()      |
    |   └ virtqueue_push(used)     |                            |
    |                              |                            |
    |   virtio_notify()            |                            |
    |   └ pci_set_irq() --------->  中断                       |
    |                              |                            |
    +<-- Guest 收到完成中断        |                            |
```

---

## 18. ARM PL330 DMA 控制器

### 18.1 架构概述

PL330 是 ARM 设计的可编程 DMA 控制器，具有自己的**微指令集**：

```
┌───────────────────────────────────────┐
│              PL330 DMA                │
│                                       │
│  ┌─────────┐  ┌─────────────────┐    │
│  │ Manager │  │ Channel 0..N-1  │    │
│  │ Thread  │  │  SAR / DAR /CCR │    │
│  │         │  │  State Machine  │    │
│  └────┬────┘  └────────┬────────┘    │
│       │                │              │
│       └──── 指令队列 ──┘              │
│              │                        │
│              v                        │
│    ┌─────────────────┐                │
│    │  AXI Master     │──── DMA 传输   │
│    │  (读/写总线)    │                │
│    └─────────────────┘                │
└───────────────────────────────────────┘
```

### 18.2 PL330State 数据结构

```c
// pl330.c:229-276
typedef struct PL330State {
    SysBusDevice parent_obj;

    // 通道（最多 8 个）
    PL330Chan chan[PL330_MAX_CHANNELS];
    PL330Chan manager;                  // Manager thread

    // 配置
    uint8_t num_chnls;                  // 通道数
    uint8_t num_periph_req;             // 外设请求线数
    uint8_t num_events;                 // 事件数
    uint32_t cfg[6];                    // 配置寄存器

    // FIFO 和队列
    PL330Fifo fifo;                     // 数据 FIFO
    PL330Queue read_queue;              // 读队列
    PL330Queue write_queue;             // 写队列

    // 中断和内存
    MemoryRegion iomem;                 // MMIO 区域
    qemu_irq irq[PL330_MAX_IRQS];      // 中断输出
    qemu_irq irq_abort;                // 异常中断

    // DMA 地址空间
    MemoryRegion *dma_mr;              // DMA 目标内存
    AddressSpace as;                    // DMA 地址空间
} PL330State;
```

### 18.3 PL330 通道

```c
// pl330.c:117-138
typedef struct PL330Chan {
    uint32_t src;                       // 源地址 (SAR)
    uint32_t dst;                       // 目标地址 (DAR)
    uint32_t pc;                        // 程序计数器（指令地址）
    uint32_t control;                   // 通道控制寄存器 (CCR)
    uint32_t status;                    // 通道状态
    uint32_t lc[2];                     // 循环计数器
    bool ns;                            // 非安全标志
    uint8_t request_flag;               // 外设请求标志
    uint8_t wakeup;                     // 唤醒事件
    PL330Insn insn;                     // 当前指令
    bool is_manager;                    // 是否为 manager
    PL330State *parent;                 // 父设备
} PL330Chan;
```

### 18.4 PL330 指令集

```c
// pl330.c:1063-1088 — 指令分发表
// PL330 有自己的微指令集，程序存放在 guest 内存中

static const PL330InsnDesc insn_desc[] = {
    { .opcode = 0x54, .name = "DMAADDH",  .exec = pl330_dmaaddh  },
    { .opcode = 0x5C, .name = "DMAADNH",  .exec = pl330_dmaadnh  },
    { .opcode = 0x00, .name = "DMAEND",   .exec = pl330_dmaend   },
    { .opcode = 0x35, .name = "DMAFLUSHP",.exec = pl330_dmaflushp},
    { .opcode = 0xA0, .name = "DMAGO",    .exec = pl330_dmago    },
    { .opcode = 0xFE, .name = "DMAKILL",  .exec = pl330_dmakill  },
    { .opcode = 0x04, .name = "DMALD",    .exec = pl330_dmald    },
    { .opcode = 0x25, .name = "DMALDP",   .exec = pl330_dmaldp   },
    { .opcode = 0x20, .name = "DMALP",    .exec = pl330_dmalp    },
    { .opcode = 0x28, .name = "DMALPEND", .exec = pl330_dmalpend },
    { .opcode = 0xBC, .name = "DMAMOV",   .exec = pl330_dmamov   },
    { .opcode = 0x18, .name = "DMANOP",   .exec = pl330_dmanop   },
    { .opcode = 0x12, .name = "DMARMB",   .exec = pl330_dmarmb   },
    { .opcode = 0x34, .name = "DMASEV",   .exec = pl330_dmasev   },
    { .opcode = 0x08, .name = "DMAST",    .exec = pl330_dmast    },
    { .opcode = 0x29, .name = "DMASTP",   .exec = pl330_dmastp   },
    { .opcode = 0x0C, .name = "DMASTZ",   .exec = pl330_dmastz   },
    { .opcode = 0x36, .name = "DMAWFE",   .exec = pl330_dmawfe   },
    { .opcode = 0x13, .name = "DMAWMB",   .exec = pl330_dmawmb   },
};
```

### 18.5 指令执行循环

```c
// pl330.c:1099-1277
// PL330 指令取-译-执行循环

static int pl330_chan_exec(PL330Chan *ch) {
    // 1. 从 guest 内存取指令
    //    dma_memory_read(ch->parent->as, ch->pc, &insn_buf, ...)
    uint8_t opcode;
    dma_memory_read(&ch->parent->as, ch->pc, &opcode, 1, MEMTXATTRS_UNSPECIFIED);

    // 2. 查指令表
    for (i = 0; insn_desc[i].exec; i++) {
        if ((opcode & insn_desc[i].opmask) == insn_desc[i].opcode) {
            // 3. 执行指令
            insn_desc[i].exec(ch, opcode, args, len);
            ch->pc += len;
            break;
        }
    }
}

// 单条指令示例 — DMALD (DMA Load)
// pl330.c:833-925
static void pl330_dmald(PL330Chan *ch, ...) {
    // 从 ch->src 地址读取数据到 FIFO
    // FIFO 中的数据后续由 DMAST 写入目标
    uint32_t size = (1 << ch->control_src_size);
    dma_memory_read(&ch->parent->as, ch->src, fifo_buf, size, ...);
    ch->src += size;  // 自增源地址
}

// DMAST (DMA Store)
static void pl330_dmast(PL330Chan *ch, ...) {
    // 从 FIFO 取数据写入 ch->dst 地址
    dma_memory_write(&ch->parent->as, ch->dst, fifo_buf, size, ...);
    ch->dst += size;  // 自增目标地址
}
```

### 18.6 通道状态机

```
  Stopped ──DMAGO──> Executing
     ^                   │
     │                   ├── DMALD → 从源读数据 → FIFO
     │                   ├── DMAST → 从 FIFO 写到目标
     │                   ├── DMALP/DMALPEND → 循环
     │                   ├── DMAWFE → WaitForEvent
     │                   │             │
     │                   │             └── DMASEV → 事件触发
     │                   │                 → 返回 Executing
     │                   │
     │                   ├── DMAWFP → WaitForPeriph
     │                   │
     DMAKILL/            └── DMAEND → Stopped
     DMAEND                           └── 触发中断
```

---

## 19. ARM PL080 DMA 控制器

```c
// pl080.c:81-220
// PL080/PL081 是较简单的 DMA 控制器（无微指令）
// 使用寄存器配置 DMA 传输（源地址、目标地址、传输大小、控制字）

// 核心传输逻辑：
static void pl080_run(PL080State *s) {
    while (ch->conf & PL080_CCONF_E) {  // 通道使能
        // 从源读
        dma_memory_read(&s->as, ch->src, buf, size, ...);
        // 写到目标
        dma_memory_write(&s->as, ch->dst, buf, size, ...);
        // 更新地址和计数
        ch->src += size;
        ch->dst += size;
    }
}
```

PL080 vs PL330 对比：

| 特性 | PL080 | PL330 |
|------|-------|-------|
| 编程模型 | 寄存器配置 | 微指令程序 |
| 灵活性 | 固定传输模式 | 可编程循环/条件 |
| 复杂度 | 471 行 | 1,694 行 |
| 通道数 | 8 (PL080) / 2 (PL081) | 可配置（最多 8） |

---

## 20. hw/dma/ 目录全景

```c
// hw/dma/meson.build:1-14
hw/dma/
├── bcm2835_dma.c     (410 行)  — Raspberry Pi DMA 控制器
├── i82374.c          (140 行)  — ISA DMA 控制器（薄封装 i8257）
├── i8257.c           (659 行)  — Intel 8257 ISA DMA
├── omap_dma.c                   — TI OMAP DMA
├── pl080.c           (471 行)  — ARM PrimeCell PL080/PL081
├── pl330.c           (1694 行) — ARM PrimeCell PL330
├── rc4030.c                     — MIPS RC4030 + IOMMU
├── sifive_pdma.c                — SiFive Platform DMA (RISC-V)
├── soc_dma.c                    — 通用 SoC DMA 框架
├── sparc32_dma.c     (455 行)  — SPARC STP2000 DMA + IOMMU
├── xilinx_axidma.c   (690 行)  — Xilinx AXI DMA
├── xlnx-zdma.c       (842 行)  — Xilinx ZynqMP DMA
├── xlnx-zynq-devcfg.c          — Xilinx Zynq DevCfg
├── xlnx_csu_dma.c               — Xilinx CSU DMA
└── xlnx_dpdma.c                 — Xilinx DisplayPort DMA
```

所有 DMA 控制器的共同模式：
1. 使用 `MemoryRegion *dma_mr` + `AddressSpace as` 管理 DMA 目标
2. 通过 `dma_memory_read/write()` 或 `address_space_rw()` 执行 DMA 传输
3. 通过 `qemu_irq` 向 CPU 发送 DMA 完成中断

---

## 21. ARM virt 机器 DMA 路径

### 21.1 不同设备的 DMA 路径

```
=== virtio-mmio 设备 ===                === PCIe 设备（无 IOMMU）===
vdev->dma_as = &address_space_memory    dev->bus_master_as
    │                                       │
    v                                       v
dma_memory_map(address_space_memory)    pci_dma_read(bus_master_as)
    │                                       │
    v                                       v
直接访问 guest RAM                       通过 PCI host bridge AS
（dma-coherent: 无需 cache flush）       → address_space_memory
                                         → guest RAM

=== PCIe 设备（有 SMMUv3）===
dev->bus_master_as
    │
    v
pci_dma_read(bus_master_as)
    │
    v
IOMMUMemoryRegion (SMMUv3)
    │
    v
smmuv3_translate(IOVA) → GPA
    │
    v
address_space_memory → guest RAM
```

### 21.2 dma-coherent 属性

```c
// virt.c:383-392 — 根节点
qemu_fdt_setprop(ms->fdt, "/", "dma-coherent", NULL, 0);

// virt.c:1537-1567 — virtio-mmio 节点
qemu_fdt_setprop(ms->fdt, node_path, "dma-coherent", NULL, 0);

// virt.c:1987-1998 — PCIe 节点
qemu_fdt_setprop(ms->fdt, node_path, "dma-coherent", NULL, 0);
```

### 21.3 IOMMU 配置

```c
// virt.c:1794-1848 — SMMUv3 IOMMU 映射
// 为 PCIe 设备配置 iommu-map 属性
// iommu-map = <0x0 &smmu 0x0 0x10000>
// 即所有 PCI RID 0x0000-0xFFFF 映射到 SMMUv3 StreamID 范围

// virt.c:1980-1982
pci->bypass_iommu = vms->default_bus_bypass_iommu;
// 当 bypass_iommu=true 时，PCIe 设备 DMA 不经过 SMMUv3
```

---

## 22. DMA 与缓存一致性

### 22.1 QEMU 的简化模型

QEMU 对 DMA 缓存一致性采用**完全一致性**简化：

- ARM virt 机器所有设备标记 `dma-coherent`
- QEMU **不模拟 CPU cache**（所有内存访问直接命中"RAM"）
- 因此 DMA 操作天然一致：设备写入的数据 CPU 立即可见，反之亦然
- Guest OS 中的 `dma_alloc_coherent()` / cache flush 操作在 QEMU 中是空操作

### 22.2 对比真实硬件

| 方面 | 真实硬件 | QEMU 模拟 |
|------|---------|-----------|
| CPU Cache | L1/L2/L3 层次 | 无 cache 模拟 |
| DMA 一致性 | 需要 cache flush/invalidate | 天然一致 |
| non-coherent DMA | 需要 dma_sync_* API | 不需要 |
| cache 维护指令 | 实际执行 | NOP（空操作） |
| 性能差异 | coherent vs non-coherent | 无差异 |

---

## 23. 端到端 DMA 流程图

### 23.1 PCI 设备 DMA 读（从 guest 内存读取数据到设备）

```
PCI 设备 (e.g. e1000 TX)
    │
    │  pci_dma_read(dev, IOVA, buf, len)
    │
    v
dma_memory_read(pci_get_address_space(dev), IOVA, buf, len, attrs)
    │
    v
address_space_rw(&dev->bus_master_as, IOVA, attrs, buf, len, is_write=false)
    │
    v
flatview_rw(fv, IOVA, ...)
    │
    v
flatview_translate(fv, IOVA, ...)
    │
    ├── [无 IOMMU] → 直接找到 RAM MemoryRegion
    │                  addr = IOVA (= GPA)
    │
    └── [有 IOMMU/SMMUv3] →
        address_space_translate_iommu(iommu_mr, IOVA)
            │
            v
        smmuv3_translate(IOVA) → IOMMUTLBEntry {
            target_as = &address_space_memory,
            translated_addr = GPA
        }
            │
            v
        在 target_as 中继续查找 → RAM MemoryRegion
    │
    v
memory_access_is_direct(mr) == true (RAM)
    │
    v
ptr = qemu_map_ram_ptr(mr->ram_block, offset)
memcpy(buf, ptr, len)   ← 完成 DMA 读取
```

### 23.2 DMA 控制器传输（PL330 示例）

```
Guest Driver                    PL330 DMA Controller          Memory
    │                              │                            │
    │  1. 在 guest RAM 中写入      │                            │
    │     PL330 微程序：           │                            │
    │     DMAMOV SAR, src_addr     │                            │
    │     DMAMOV DAR, dst_addr     │                            │
    │     DMALP 64                 │                            │
    │       DMALD                  │                            │
    │       DMAST                  │                            │
    │     DMALPEND                 │                            │
    │     DMAEND                   │                            │
    │                              │                            │
    │  2. 写 DBGINST 寄存器       │                            │
    │     → DMAGO channel, PC ----→│                            │
    │                              │                            │
    │                     pl330_chan_exec():                     │
    │                     ┌── DMAMOV: ch->src = src_addr        │
    │                     ├── DMAMOV: ch->dst = dst_addr        │
    │                     ├── DMALP:  lc[0] = 64                │
    │                     │                                     │
    │                     │  循环 64 次:                         │
    │                     ├── DMALD:                             │
    │                     │   dma_memory_read(as, ch->src, ...) │
    │                     │   src → FIFO                    <───┤ 读源
    │                     │   ch->src += size                    │
    │                     ├── DMAST:                             │
    │                     │   dma_memory_write(as, ch->dst, ..) │
    │                     │   FIFO → dst                    ───>│ 写目标
    │                     │   ch->dst += size                    │
    │                     ├── DMALPEND: lc[0]--, 若>0 跳回 DMALD│
    │                     │                                     │
    │                     └── DMAEND: → Stopped                 │
    │                         → DMASEV → qemu_irq               │
    │                              │                            │
    │  <──────── DMA 完成中断 ─────┤                            │
```

---

## 附录 A. 关键数据结构速查表

| 数据结构 | 定义位置 | 用途 |
|----------|----------|------|
| `DMADirection` | `dma.h:18-21` | DMA 方向（TO/FROM_DEVICE） |
| `dma_addr_t` | `dma.h:30` | DMA 地址类型（uint64_t） |
| `ScatterGatherEntry` | `dma.h:279-282` | SGList 条目 |
| `QEMUSGList` | `dma.h:37-44` | 散列聚合列表 |
| `IOMMUTLBEntry` | `memory.h:147-154` | IOMMU 翻译结果 |
| `IOMMUMemoryRegion` | `memory.h:46-49` | IOMMU 内存区域 |
| `IOMMUMemoryRegionClass` | `memory.h:430-542` | IOMMU 回调接口 |
| `IOMMUNotifier` | `memory.h:186-215` | IOMMU 映射通知器 |
| `PCIDevice.bus_master_as` | `pci_device.h:96` | PCI DMA 地址空间 |
| `PL330State` | `pl330.c:229-276` | PL330 DMA 控制器状态 |
| `PL330Chan` | `pl330.c:117-138` | PL330 通道状态 |

---

## 附录 B. 源码文件索引

| 分类 | 文件 | 行数 | 说明 |
|------|------|------|------|
| **核心框架** | `include/system/dma.h` | 322 | DMA API 定义 |
| | `system/dma-helpers.c` | 347 | DMA 辅助实现 |
| | `system/physmem.c` | ~4,000 | address_space_* 实现 |
| | `include/system/memory.h` | — | IOMMU 类型定义 |
| **PCI DMA** | `include/hw/pci/pci_device.h` | — | PCI DMA 便利接口 |
| | `hw/pci/pci.c` | — | BME 初始化 |
| **ARM DMA** | `hw/dma/pl330.c` | 1,694 | PL330 可编程 DMA |
| | `hw/dma/pl080.c` | 471 | PL080/PL081 DMA |
| | `hw/dma/bcm2835_dma.c` | 410 | Raspberry Pi DMA |
| **Xilinx DMA** | `hw/dma/xilinx_axidma.c` | 690 | AXI DMA |
| | `hw/dma/xlnx-zdma.c` | 842 | ZynqMP DMA |
| **ISA DMA** | `hw/dma/i8257.c` | 659 | Intel 8257 |
| **设备示例** | `hw/net/e1000.c` | 1,767 | e1000 DMA |
| | `hw/ide/ahci.c` | 1,813 | AHCI DMA 引擎 |
| | `hw/block/virtio-blk.c` | — | virtio-blk DMA |
| | `hw/virtio/virtio.c` | — | vring 映射 |

---

## 附录 C. 关联文档索引

| 文档 | 路径 | 关联内容 |
|------|------|----------|
| 内存子系统深度分析 | `darren/memory/00-内存子系统深度分析.md` | MemoryRegion、FlatView、地址翻译 |
| VFIO 设备直通与 IOMMU | `darren/device-model/05-VFIO设备直通与IOMMU集成深度分析.md` | SMMUv3、virtio-iommu、DMA 映射 |
| 设备模型与 virtio | `darren/device-model/00-设备模型与virtio深度分析.md` | VirtQueue、设备初始化 |
| 关键设备仿真分析 | `darren/device-model/01-关键设备仿真分析-UART-磁盘-网卡.md` | virtio-blk/virtio-net 细节 |
| 块层 IO 路径 | `darren/device-model/02-块层IO路径深度分析.md` | block backend、AIO 框架 |
| FDT 设备树 | `darren/arm64/05-FDT设备树深度分析.md` | dma-coherent、iommu-map FDT 属性 |
| 网络后端分析 | `darren/device-model/06-网络后端深度分析-TAP-vhost-net-vhost-user.md` | vhost DMA 映射 |

---

> 文档生成时间：基于 QEMU 11.0.50 源码分析  
> 索引工具：zoekt + ctags + cscope（`.ai-search/`）
