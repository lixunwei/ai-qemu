# PCI/PCIe 子系统深度分析

## 1. 概述

PCI/PCIe 是 QEMU 中最重要的总线子系统之一，ARM virt 机器通过 GPEX Host Bridge 提供 PCIe 根复合体。本文分析 QEMU 11.0.50 中 PCI 设备模型、配置空间、BAR 映射、中断路由、MSI-X、ECAM 和设备直通。

**关键源文件：**
- `include/hw/pci/pci_device.h` — PCIDevice 结构定义
- `include/hw/pci/pci.h` — PCI 常量和 API
- `hw/pci/pci.c` — PCI 核心逻辑
- `hw/pci/pcie.c` — PCIe 能力扩展
- `hw/pci-host/gpex.c` — Generic PCIe Host Bridge
- `hw/pci/msi.c` / `hw/pci/msix.c` — MSI/MSI-X
- `hw/pci/pcie_host.c` — ECAM 配置空间映射
- `hw/arm/virt.c` — ARM virt 的 PCIe 集成

---

## 2. PCIDevice 核心结构

```c
// pci_device.h:60-120
struct PCIDevice {
    DeviceState qdev;
    bool partially_hotplugged;
    bool enabled;

    uint8_t *config;        // PCI 配置空间（256B 或 4KB）
    uint8_t *cmask;         // 一致性检查掩码
    uint8_t *wmask;         // 写掩码（哪些位可写）
    uint8_t *w1cmask;       // Write-1-to-Clear 掩码
    uint8_t *used;          // Capability 空间分配跟踪

    int32_t devfn;          // 设备功能号（bus:dev.fn 编码）
    PCIReqIDCache requester_id_cache;  // Requester ID 缓存
    char name[64];
    PCIIORegion io_regions[PCI_NUM_REGIONS];  // BAR 区域（6 + ROM）
    AddressSpace bus_master_as;               // Bus Master 地址空间
    MemoryRegion bus_master_enable_region;

    PCIConfigReadFunc *config_read;    // 配置空间读回调
    PCIConfigWriteFunc *config_write;  // 配置空间写回调

    uint8_t irq_state;      // INTx 中断电平
    uint32_t cap_present;   // 已存在的 Capability 位图
    uint8_t pm_cap;         // PM Capability 偏移
    uint8_t msix_cap;       // MSI-X Capability 偏移
};
```

---

## 3. PCI 配置空间

### 3.1 空间大小

```c
// pci.h:189-194
#define PCI_CONFIG_SPACE_SIZE  0x100    // PCI: 256 字节
#define PCIE_CONFIG_SPACE_SIZE 0x1000   // PCIe: 4KB（扩展配置空间）
```

### 3.2 配置空间分配

```c
// pci.c:1190-1199 — pci_config_alloc()
```

分配 config/cmask/wmask/w1cmask/used 五组等大小缓冲区。PCIe 设备使用 4KB，传统 PCI 使用 256B。

### 3.3 默认读写

```c
// pci.c:1786-1836 — pci_default_read_config() / pci_default_write_config()
```

- **读**：直接从 `config[]` 缓冲区返回对应字节
- **写**：通过 `wmask` 过滤只写可写位，`w1cmask` 处理 Write-1-to-Clear 位
- 写入后触发 `pci_update_mappings()` 更新 BAR 映射

---

## 4. BAR（Base Address Register）机制

### 4.1 BAR 注册

```c
// pci.c:1497-1551 — pci_register_bar()
void pci_register_bar(PCIDevice *pci_dev, int region_num,
                      uint8_t type, MemoryRegion *memory)
{
    r = &pci_dev->io_regions[region_num];
    r->size = size;
    r->type = type;
    r->memory = memory;
    // I/O BAR → address_space_io; MMIO BAR → address_space_mem
    r->address_space = type & PCI_BASE_ADDRESS_SPACE_IO
                       ? pci_get_bus(pci_dev)->address_space_io
                       : pci_get_bus(pci_dev)->address_space_mem;

    // 设置 wmask：基于 size 对齐（size 必须是 2 的幂）
    wmask = ~(size - 1);
    pci_set_long(pci_dev->wmask + addr, wmask & 0xffffffff);

    // 64-bit BAR 跨两个寄存器
    if (type & PCI_BASE_ADDRESS_MEM_TYPE_64) {
        pci_set_quad(pci_dev->wmask + addr, wmask);
    }
}
```

### 4.2 BAR 地址计算

```c
// pci.c:1658-1721 — pci_bar_address()
```

- **I/O BAR**（1666-1680）：`addr & PCI_BASE_ADDRESS_IO_MASK`，检查不超过 64KB
- **MMIO BAR**（1682-1720）：`addr & PCI_BASE_ADDRESS_MEM_MASK`，64-bit 需要合并高 32 位

### 4.3 BAR 映射更新

```c
// pci.c:1723-1765 — pci_update_mappings()
```

Guest 修改 BAR 寄存器时，QEMU 重新计算地址并调用：
- `memory_region_del_subregion()` — 移除旧映射
- `memory_region_add_subregion_overlap()` — 添加新映射

---

## 5. PCI 总线模型

### 5.1 PCIBus 结构

```c
// pci_bus.h:33-59
struct PCIBus {
    BusState qbus;
    PCIIOMMUFunc iommu_fn;
    ...
    MemoryRegion *address_space_mem;  // MMIO 地址空间
    MemoryRegion *address_space_io;   // I/O 端口空间
};
```

### 5.2 根总线创建

```c
// pci.c:677-721 — pci_register_root_bus()
```

由 Host Bridge（如 GPEX）调用，创建 PCI 根总线并注册 IRQ 处理。

---

## 6. GPEX Host Bridge（ARM PCIe 根复合体）

### 6.1 realize 流程

```c
// gpex.c:90-158 — gpex_host_realize()
{
    // 1. 初始化 ECAM MMIO 区域
    pcie_host_mmcfg_init(pex, PCIE_MMCFG_SIZE_MAX);

    // 2. 创建 PCI MMIO 和 I/O 端口地址空间
    memory_region_init(&s->io_mmio, ..., "gpex_mmio", UINT64_MAX);
    memory_region_init(&s->io_ioport, ..., "gpex_ioport", 64 * 1024);

    // 3. 未映射地址处理（读返回 -1，写忽略）
    if (s->allow_unmapped_accesses) {
        memory_region_init_io(&s->io_mmio_window, ..., &unassigned_io_ops, ...);
        memory_region_add_subregion(&s->io_mmio_window, 0, &s->io_mmio);
    }

    // 4. 创建 PCIe 根总线
    pci->bus = pci_register_root_bus(dev, "pcie.0",
                gpex_set_irq, gpex_swizzle_map_irq_fn,
                s, &s->io_mmio, &s->io_ioport, 0,
                s->num_irqs, TYPE_PCIE_BUS);

    // 5. 设置 INTx 路由
    pci_bus_set_route_irq_fn(pci->bus, gpex_route_intx_pin_to_irq);
}
```

### 6.2 窗口结构

GPEX 提供三种地址窗口：
- **ECAM**：配置空间访问（每功能 4KB）
- **MMIO 窗口**：PCI 设备的 Memory BAR 映射
- **I/O 窗口**：PCI 设备的 I/O BAR 映射（ARM 上罕用）

---

## 7. ECAM 配置空间访问

### 7.1 ECAM 映射

```c
// pcie_host.c:27-68
```

ECAM（Enhanced Configuration Access Mechanism）将 PCI 配置空间映射为 MMIO：
- 地址编码：`base + (bus << 20) | (dev << 15) | (fn << 12) | reg`
- 每个功能占 4KB，支持完整的 PCIe 扩展配置空间
- Guest 通过普通内存访问指令读写配置寄存器

### 7.2 ARM virt ECAM 配置

```c
// virt.c:1938-1996
```

ARM virt 机器在 `create_pcie()` 中配置 ECAM 基地址和范围，支持最多 256 条总线。

---

## 8. 中断路由

### 8.1 INTx 中断

```c
// pci.c:1841-1872 — pci_irq_handler() / pci_set_irq()
```

PCI INTx 中断路由：
1. 设备调用 `pci_set_irq()` → `pci_irq_handler()`
2. 更新 `irq_state` 电平
3. 通过 `pci_bus_set_irq`→ 总线 IRQ 回调
4. **GPEX**：`gpex_set_irq()` → GIC SPI 中断

### 8.2 INTx Swizzle

```c
// pci.c:1875-1902
```

多功能设备和桥后设备的 INTx 引脚通过 swizzle 函数旋转，确保中断分散到不同 GIC 中断号。

### 8.3 ARM virt INTx 连接

```c
// virt.c:1974-1978
```

ARM virt 将 PCI INTx（INTA/B/C/D）映射到 4 个 GIC SPI 中断号。

---

## 9. MSI/MSI-X

### 9.1 MSI

```c
// msi.c:193-255 — msi_init()
// msi.c:321-376 — msi_notify()
```

MSI 通过 PCI Capability 结构实现：
- `msi_init()` — 在配置空间添加 MSI Capability
- `msi_notify()` — 构造 MSI 消息（address + data），写入 MMIO 地址触发中断

### 9.2 MSI-X

```c
// msix.c:321-392 — msix_init()
// msix.c:212-285 — MSI-X Table/PBA MMIO
```

MSI-X 比 MSI 更灵活：
- **MSI-X Table**：独立的 MMIO 区域（BAR 内），每个 entry 含 address + data + control
- **PBA**（Pending Bit Array）：记录被屏蔽时挂起的中断
- 每个 virtqueue 可独立分配 MSI-X 向量

### 9.3 KVM irqfd

```c
// hw/vfio/pci.c:567-605
```

VFIO 直通设备使用 KVM irqfd 将 MSI-X 中断直接注入 Guest，无需 QEMU 用户态参与。

---

## 10. PCIe 能力扩展

### 10.1 PCIe Capability

```c
// pcie.c:223-257 — pcie_cap_init()
// pcie.c:281-314 — pcie_endpoint_cap_init()
```

PCIe Capability 结构包含：
- **Device Capabilities**：Max Payload Size、Extended Tag
- **Link Capabilities**：链路宽度、速率
- **Device Control/Status**：错误报告、Max Read Request Size

### 10.2 AER（Advanced Error Reporting）

```c
// pcie_aer.c — PCIe AER 扩展 Capability
```

AER 提供细粒度的 PCIe 错误分类和报告：
- Correctable Error（可纠正）
- Uncorrectable Error（不可纠正，Fatal/Non-Fatal）
- 错误日志记录到 AER Capability 寄存器

---

## 11. ARM virt PCIe 集成

### 11.1 create_pcie()

```c
// virt.c:1912-2043 — create_pcie()
```

完整流程：
1. 创建 GPEX Host Bridge 实例
2. 配置 ECAM 基地址和总线范围（1938-1996）
3. 映射 MMIO 窗口和 high MMIO 窗口（1950-1973, 2007-2022）
4. 分配 PIO 窗口（ARM 上仍提供兼容性支持）
5. 连接 4 个 INTx 中断到 GIC（1974-1978）
6. 生成设备树 `interrupt-map` 属性（2024-2025）

### 11.2 地址空间布局

| 区域 | 用途 |
|------|------|
| ECAM | PCIe 配置空间（每功能 4KB） |
| PCI MMIO | 32-bit MMIO 窗口（BAR 映射） |
| PCI MMIO_HIGH | 64-bit MMIO 窗口（大 BAR） |
| PCI PIO | I/O 端口窗口 |

---

## 12. VFIO 设备直通

### 12.1 原理

VFIO（Virtual Function I/O）将物理 PCI 设备直接暴露给 Guest：
- 配置空间：QEMU 拦截并转发
- BAR MMIO：直接映射到 Guest 地址空间
- 中断：MSI-X + irqfd 直接注入
- DMA：通过 IOMMU 建立安全映射

### 12.2 关键路径

```c
// hw/vfio/pci.c — vfio_pci_realize()
```

- BAR 映射：读取物理 BAR 信息，创建对应的 MemoryRegion
- MSI-X：配置 irqfd，物理中断直接触发 KVM 中断注入
- 配置空间：拦截 Guest 访问，转发到物理设备

---

## 13. 热插拔

### 13.1 SHPC（Standard Hot-Plug Controller）

```c
// hw/pci/shpc.c
```

传统 PCI 热插拔控制器，通过注意力按钮/指示灯/电源控制管理插槽。

### 13.2 Native PCIe 热插拔

```c
// pcie.c:399-520
```

PCIe 原生热插拔通过 Slot Capability：
- 插入检测：Presence Detect State Change
- 电源控制：Power Controller Control
- 事件通知：Hot-Plug Interrupt → MSI/MSI-X

---

## 14. 小结

| 方面 | 实现要点 |
|------|----------|
| **PCIDevice** | config[]/wmask[]/devfn + io_regions[7] BAR 数组 |
| **配置空间** | PCI 256B / PCIe 4KB，wmask 控制可写位 |
| **BAR** | pci_register_bar() → pci_update_mappings() 动态映射/取消映射 |
| **GPEX** | ARM PCIe 根复合体，ECAM+MMIO+PIO 三窗口 |
| **ECAM** | 每功能 4KB，(bus<<20\|dev<<15\|fn<<12\|reg) 编址 |
| **INTx** | 4 条共享线，swizzle 旋转，GPEX→GIC SPI |
| **MSI-X** | 独立 BAR 内 Table/PBA，每 queue 独立向量 |
| **VFIO** | 物理设备直通，BAR 直映 + MSI-X irqfd + IOMMU DMA |
| **热插拔** | SHPC（传统）/ Native PCIe Slot Capability |
