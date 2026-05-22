# 20 - PCI/PCIe 子系统深度分析 — 设备模型、配置空间、BAR 映射、MSI/MSI-X 中断与 ECAM

> **基于 QEMU 11.0.50 源码**，深入分析 PCI/PCIe 子系统的完整实现：
> 从核心数据结构（PCIDevice/PCIBus/PCIHostState）到配置空间读写机制、
> BAR 地址映射、MSI/MSI-X 中断投递、ECAM 访问路径，以及 ARM virt 机器的 GPEX 集成。

---

## 目录

1. [PCI 设备模型核心：PCIDevice 与 PCIDeviceClass](#1-pci-设备模型核心pcdevice-与-pcideviceclass)
2. [PCI 总线模型：PCIBus 与 PCIHostState](#2-pci-总线模型pcibus-与-pcihoststate)
3. [配置空间管理：分配、掩码与读写机制](#3-配置空间管理分配掩码与读写机制)
4. [配置空间访问路径：Host Bridge → Device 调度](#4-配置空间访问路径host-bridge--device-调度)
5. [BAR 管理：注册、地址计算与动态映射](#5-bar-管理注册地址计算与动态映射)
6. [PCIe 能力（Capability）与 PCIExpressDevice](#6-pcie-能力capability与-pciexpressdevice)
7. [MSI 中断：能力注册、消息构造与投递](#7-msi-中断能力注册消息构造与投递)
8. [MSI-X 中断：Table/PBA MMIO 与向量管理](#8-msi-x-中断tablepba-mmio-与向量管理)
9. [MSI/MSI-X → GIC ITS 投递路径](#9-msimsi-x--gic-its-投递路径)
10. [INTx 传统中断：路由、Swizzle 与电平追踪](#10-intx-传统中断路由swizzle-与电平追踪)
11. [ECAM 与 GPEX：ARM virt PCIe 主桥](#11-ecam-与-gpexarm-virt-pcie-主桥)
12. [virt 机器 PCIe 集成：地址窗口、IRQ 与设备树](#12-virt-机器-pcie-集成地址窗口irq-与设备树)
13. [设备生命周期：注册、实现与热插拔](#13-设备生命周期注册实现与热插拔)

---

## 1. PCI 设备模型核心：PCIDevice 与 PCIDeviceClass

### 1.1 PCIDeviceClass — 设备类方法表

```c
// pci_device.h:26-44
struct PCIDeviceClass {
    DeviceClass parent_class;
    void (*realize)(PCIDevice *dev, Error **errp);  // 设备实现回调
    PCIUnregisterFunc *exit;                         // 设备销毁回调
    PCIConfigReadFunc *config_read;                  // 配置空间读（可覆盖）
    PCIConfigWriteFunc *config_write;                // 配置空间写（可覆盖）
    uint16_t vendor_id;                              // PCI Vendor ID
    uint16_t device_id;                              // PCI Device ID
    uint8_t revision;                                // Revision ID
    uint16_t class_id;                               // Class Code
    uint16_t subsystem_vendor_id;                    // 仅 Type 0
    uint16_t subsystem_id;                           // 仅 Type 0
    const char *romfile;                             // ROM 文件路径
    bool sriov_vf_user_creatable;
};
```

**设计要点**：
- `config_read`/`config_write` 可被具体设备覆盖，默认使用 `pci_default_read_config`/`pci_default_write_config`
- `vendor_id`/`device_id` 等在类定义时静态赋值，设备实现时写入配置空间

### 1.2 PCIDevice — 设备实例结构

```c
// pci_device.h:60-190
struct PCIDevice {
    DeviceState qdev;                    // QOM 基类
    bool partially_hotplugged;
    bool enabled;

    /* 配置空间（4 个掩码数组 + 1 个已用位图） */
    uint8_t *config;                     // 实际配置数据（256B 或 4KB）
    uint8_t *cmask;                      // Check Mask：load 时校验用
    uint8_t *wmask;                      // Write Mask：哪些位可写
    uint8_t *w1cmask;                    // Write-1-to-Clear Mask
    uint8_t *used;                       // Capability 空间占用位图

    /* 设备标识 */
    int32_t devfn;                       // device:3 | function:3 编码
    PCIReqIDCache requester_id_cache;    // 缓存 Requester ID
    char name[64];

    /* BAR（Base Address Register）区域 */
    PCIIORegion io_regions[PCI_NUM_REGIONS]; // 最多 7 个（6 BAR + 1 ROM）

    /* DMA 主设备地址空间 */
    AddressSpace bus_master_as;          // MSI 写入也经此空间
    bool is_master;
    MemoryRegion bus_master_container_region;
    MemoryRegion bus_master_enable_region;

    /* 中断状态 */
    uint8_t irq_state;                   // 4 个 INTx pin 的电平位图
    uint32_t cap_present;                // 能力标志位

    /* MSI */
    uint8_t msi_cap;                     // MSI 能力在配置空间的偏移
    MSITriggerFunc *msi_trigger;         // MSI 发送回调（默认 pci_msi_trigger）
    MSIPrepareMessageFunc *msi_prepare_message;

    /* MSI-X */
    uint8_t msix_cap;                    // MSI-X 能力偏移
    int msix_entries_nr;                 // MSI-X 向量数
    uint8_t *msix_table;                 // MSI-X Table（内存中副本）
    uint8_t *msix_pba;                   // Pending Bit Array
    unsigned *msix_entry_used;           // 引用计数
    bool msix_function_masked;
    MemoryRegion msix_table_mmio;        // Table MMIO 区域
    MemoryRegion msix_pba_mmio;          // PBA MMIO 区域
    MSIxPrepareMessageFunc *msix_prepare_message;

    /* PCIe */
    PCIExpressDevice exp;                // PCIe 扩展能力

    /* ROM */
    char *romfile;
    uint32_t romsize;
    bool has_rom;
    MemoryRegion rom;
    int32_t rom_bar;

    /* INTx 路由通知 */
    PCIINTxRoutingNotifier intx_routing_notifier;

    /* MSI-X KVM irqfd 通知 */
    MSIVectorUseNotifier msix_vector_use_notifier;
    MSIVectorReleaseNotifier msix_vector_release_notifier;
    MSIVectorPollNotifier msix_vector_poll_notifier;
};
```

**关键设计**：
- **5 个配置空间数组**：`config`（数据）、`cmask`（校验掩码）、`wmask`（写掩码）、`w1cmask`（W1C 掩码）、`used`（能力空间占用）
- **MSI/MSI-X 双栈**：MSI 通过配置空间寄存器传递 address/data，MSI-X 通过独立 MMIO Table
- **bus_master_as**：DMA 和 MSI 写入的统一地址空间，可能经过 IOMMU 翻译

---

## 2. PCI 总线模型：PCIBus 与 PCIHostState

### 2.1 PCIBus — 总线实例

```c
// pci_bus.h:33-59
struct PCIBus {
    BusState qbus;                           // QOM 总线基类
    enum PCIBusFlags flags;                  // PCI_BUS_IS_ROOT / EXTENDED_CONFIG_SPACE / CXL
    const PCIIOMMUOps *iommu_ops;            // IOMMU 操作（可选）
    void *iommu_opaque;
    uint8_t devfn_min;                       // 最小 devfn（通常 0）
    uint32_t slot_reserved_mask;

    /* IRQ 路由 */
    pci_set_irq_fn set_irq;                 // 最终 IRQ 触发回调
    pci_map_irq_fn map_irq;                 // INTx pin → IRQ 映射
    pci_route_irq_fn route_intx_to_irq;     // PCIe INTx 路由
    void *irq_opaque;
    int nirq;                                // IRQ 数（通常 4 = INTA/B/C/D）
    int *irq_count;                          // 每个 IRQ 的活跃设备计数

    /* 设备表 */
    PCIDevice *devices[PCI_SLOT_MAX * PCI_FUNC_MAX]; // 最多 256 个 devfn

    /* 拓扑 */
    PCIDevice *parent_dev;                   // 上游桥设备（root bus 为 NULL）
    MemoryRegion *address_space_mem;         // MMIO 地址空间
    MemoryRegion *address_space_io;          // IO 端口地址空间

    QLIST_HEAD(, PCIBus) child;              // 下级总线链表
    QLIST_ENTRY(PCIBus) sibling;
};
```

**IRQ 路由三回调**：
- `map_irq`：将设备的 INTx pin 映射为总线级 IRQ 编号（含 Swizzle）
- `set_irq`：将总线级 IRQ 编号转发到中断控制器（如 GIC SPI）
- `route_intx_to_irq`：PCIe 路由查询，返回 `PCIINTxRoute`

### 2.2 PCIHostState — 主桥状态

```c
// pci_host.h:39-51
struct PCIHostState {
    SysBusDevice busdev;       // SysBus 设备基类
    MemoryRegion conf_mem;     // CF8 配置地址端口（legacy x86）
    MemoryRegion data_mem;     // CFC 配置数据端口（legacy x86）
    MemoryRegion mmcfg;        // ECAM MMIO 区域
    uint32_t config_reg;       // 当前配置地址寄存器值
    bool mig_enabled;
    PCIBus *bus;               // 根总线指针
    bool bypass_iommu;         // 是否绕过 IOMMU
};
```

### 2.3 根总线创建

```c
// pci.c:756-769
PCIBus *pci_register_root_bus(DeviceState *parent, const char *name,
                              pci_set_irq_fn set_irq, pci_map_irq_fn map_irq,
                              void *irq_opaque,
                              MemoryRegion *mem, MemoryRegion *io,
                              uint8_t devfn_min, int nirq,
                              const char *typename)
```

调用链：
```
pci_register_root_bus()
  → pci_root_bus_new()                    [pci.c:713-722]
    → qbus_new(typename, parent, name)    // 创建 QOM 总线
    → pci_root_bus_internal_init()        [pci.c:677-692]
       设置 address_space_mem/io、PCI_BUS_IS_ROOT
  → pci_bus_irqs(bus, set_irq, ...)       [pci.c:731-738]
  → pci_bus_map_irqs(bus, map_irq)        [pci.c:741-743]
```

---

## 3. 配置空间管理：分配、掩码与读写机制

### 3.1 配置空间分配

```c
// pci.c:1190-1199
static void pci_config_alloc(PCIDevice *pci_dev)
{
    int config_size = pci_config_size(pci_dev);  // 256B(PCI) 或 4KB(PCIe)
    pci_dev->config = g_malloc0(config_size);
    pci_dev->cmask = g_malloc0(config_size);
    pci_dev->wmask = g_malloc0(config_size);
    pci_dev->w1cmask = g_malloc0(config_size);
    pci_dev->used = g_malloc0(config_size);
}
```

### 3.2 掩码初始化

设备注册时（`do_pci_register_device`，pci.c:1399-1425）初始化三层掩码：

```c
// pci.c:1041-1078
pci_init_cmask()    // Check Mask：标记 Vendor/Device ID、Status、Class 等只读字段
pci_init_wmask()    // Write Mask：PCI_COMMAND (IO|MEM|MASTER|INTX_DISABLE) + 能力空间全可写
pci_init_w1cmask()  // W1C Mask：Status 寄存器中的错误位（Parity、Target Abort 等）
```

**PCI_COMMAND 可写位**：
```c
// pci.c:1059-1061
PCI_COMMAND_IO | PCI_COMMAND_MEMORY | PCI_COMMAND_MASTER | PCI_COMMAND_INTX_DISABLE
```

### 3.3 默认读/写实现

**读**（pci.c:1786-1799）：
```c
uint32_t pci_default_read_config(PCIDevice *d, uint32_t address, int len)
{
    // PCIe 下行端口特殊处理：同步链路状态
    if (pci_is_express_downstream_port(d) &&
        ranges_overlap(address, len, d->exp.exp_cap + PCI_EXP_LNKSTA, 2)) {
        pcie_sync_bridge_lnk(d);
    }
    memcpy(&val, d->config + address, len);  // 直接从 config 数组读
    return le32_to_cpu(val);
}
```

**写**（pci.c:1801-1836）：
```c
void pci_default_write_config(PCIDevice *d, uint32_t addr, uint32_t val_in, int l)
{
    for (i = 0; i < l; val >>= 8, ++i) {
        uint8_t wmask = d->wmask[addr + i];
        uint8_t w1cmask = d->w1cmask[addr + i];
        // 写掩码逻辑：保留不可写位，应用可写位
        d->config[addr + i] = (d->config[addr + i] & ~wmask) | (val & wmask);
        // W1C 逻辑：写 1 清零
        d->config[addr + i] &= ~(val & w1cmask);
    }

    // 触发 BAR 重映射（如果写了 BAR/ROM/COMMAND/PM 相关区域）
    if (ranges_overlap(addr, l, PCI_BASE_ADDRESS_0, 24) ||
        ranges_overlap(addr, l, PCI_ROM_ADDRESS, 4) ||
        range_covers_byte(addr, l, PCI_COMMAND)) {
        pci_update_mappings(d);                       // [pci.c:1723]
    }

    // 更新 INTx 禁用状态 + Bus Master
    if (ranges_overlap(addr, l, PCI_COMMAND, 2)) {
        pci_update_irq_disabled(d, was_irq_disabled);
        pci_set_master(d, ...);
    }

    // 级联到 MSI/MSI-X/SR-IOV 写处理
    msi_write_config(d, addr, val_in, l);
    msix_write_config(d, addr, val_in, l);
    pcie_sriov_config_write(d, addr, val_in, l);
}
```

**核心要点**：每次配置空间写入都可能触发 BAR 重映射、中断状态更新、MSI/MSI-X 配置变更。

---

## 4. 配置空间访问路径：Host Bridge → Device 调度

### 4.1 ECAM (PCIe) 访问路径

ECAM 地址编码（pcie_host.h:60-79）：
```
bit 27:20 = Bus Number     (8 位，最多 256 个总线)
bit 19:15 = Device Number  (5 位)
bit 14:12 = Function       (3 位)
bit 11:0  = Config Offset  (12 位，4KB 配置空间)
```

```
Guest MMIO 写入 ECAM 地址
  → pcie_mmcfg_data_write()                  [pcie_host.c:35-50]
    → pcie_dev_find_by_mmcfg_addr()          // 从地址解码 bus/devfn
      → pci_find_device(bus, bus_num, devfn)
    → pci_host_config_write_common()         [pci_host.c:76-97]
      → pci_dev->config_write(dev, addr, val, len)
        → pci_default_write_config()         // 或设备自定义
```

### 4.2 Legacy CF8/CFC 访问路径

```
IO 端口写 0xCF8 (Config Address)
  → pci_host_config_write()     [pci_host.c:158-168]
    → 存储到 PCIHostState.config_reg

IO 端口写 0xCFC (Config Data)
  → pci_host_data_write()       [pci_host.c:170-180]
    → pci_data_write()          [pci_host.c:126-140]
      → pci_dev_find_by_addr()
      → pci_host_config_write_common()
```

### 4.3 通用配置写入核心

```c
// pci_host.c:76-97
void pci_host_config_write_common(PCIDevice *pci_dev, uint32_t addr,
                                  uint32_t limit, uint32_t val, uint32_t len)
{
    pci_adjust_config_limit(pci_get_bus(pci_dev), &limit);
    if (limit <= addr) return;

    // 热插拔设备必须 function 0 存在才暴露其他 function
    if ((pci_dev->qdev.hotplugged && !pci_get_function_0(pci_dev)) ||
        !pci_dev->enabled || is_pci_dev_ejected(pci_dev)) {
        return;
    }

    pci_dev->config_write(pci_dev, addr, val, MIN(len, limit - addr));
}
```

---

## 5. BAR 管理：注册、地址计算与动态映射

### 5.1 BAR 注册

```c
// pci.c:1497-1551
void pci_register_bar(PCIDevice *pci_dev, int region_num,
                      uint8_t type, MemoryRegion *memory)
{
    PCIIORegion *r = &pci_dev->io_regions[region_num];
    r->size = memory_region_size(memory);
    r->type = type;
    r->memory = memory;

    // 根据类型选择地址空间
    r->address_space = type & PCI_BASE_ADDRESS_SPACE_IO
                        ? pci_get_bus(pci_dev)->address_space_io
                        : pci_get_bus(pci_dev)->address_space_mem;

    if (pci_is_vf(pci_dev)) {
        // SR-IOV VF 直接映射
        r->addr = pci_bar_address(pci_dev, region_num, r->type, r->size);
        if (r->addr != PCI_BAR_UNMAPPED)
            memory_region_add_subregion_overlap(r->address_space, r->addr, r->memory, 1);
    } else {
        r->addr = PCI_BAR_UNMAPPED;  // 初始未映射

        // 设置 BAR 掩码：size 对齐
        wmask = ~(size - 1);
        if (region_num == PCI_ROM_SLOT)
            wmask |= PCI_ROM_ADDRESS_ENABLE;

        // 64 位 BAR 设置 quad 掩码
        if (!(r->type & PCI_BASE_ADDRESS_SPACE_IO) &&
            r->type & PCI_BASE_ADDRESS_MEM_TYPE_64) {
            pci_set_quad(pci_dev->wmask + addr, wmask);
        } else {
            pci_set_long(pci_dev->wmask + addr, wmask & 0xffffffff);
        }
    }
}
```

**BAR 类型标志**：
| 标志 | 含义 |
|------|------|
| `PCI_BASE_ADDRESS_SPACE_IO` | IO 端口 BAR |
| `PCI_BASE_ADDRESS_MEM_TYPE_64` | 64 位 MMIO（占 2 个连续 BAR 槽位） |
| `PCI_BASE_ADDRESS_MEM_PREFETCH` | 可预取内存 |

### 5.2 BAR 地址计算

```c
// pci.c:1658-1721
pcibus_t pci_bar_address(PCIDevice *d, int reg, uint8_t type, pcibus_t size)
{
    uint16_t cmd = pci_get_word(d->config + PCI_COMMAND);

    if (type & PCI_BASE_ADDRESS_SPACE_IO) {
        if (!(cmd & PCI_COMMAND_IO)) return PCI_BAR_UNMAPPED;  // IO 未使能
        new_addr = pci_config_get_bar_addr(d, reg, type, size);
        // 校验 32 位 IO 不能回绕
    } else {
        if (!(cmd & PCI_COMMAND_MEMORY)) return PCI_BAR_UNMAPPED;  // MEM 未使能
        new_addr = pci_config_get_bar_addr(d, reg, type, size);
        // ROM BAR 需要额外检查 ENABLE 位
        // 校验不回绕、不超 32 位（非 64 位 BAR）
    }
    return new_addr;
}
```

### 5.3 BAR 动态重映射

```c
// pci.c:1723-1765
static void pci_update_mappings(PCIDevice *d)
{
    for (i = 0; i < PCI_NUM_REGIONS; i++) {
        r = &d->io_regions[i];
        if (!r->size) continue;

        new_addr = pci_bar_address(d, i, r->type, r->size);
        if (!d->enabled || pci_pm_state(d))
            new_addr = PCI_BAR_UNMAPPED;

        if (new_addr == r->addr) continue;  // 地址未变

        // 解除旧映射
        if (r->addr != PCI_BAR_UNMAPPED)
            memory_region_del_subregion(r->address_space, r->memory);

        r->addr = new_addr;

        // 建立新映射
        if (r->addr != PCI_BAR_UNMAPPED)
            memory_region_add_subregion_overlap(r->address_space, r->addr, r->memory, 1);
    }
}
```

**触发条件**（来自 `pci_default_write_config`，pci.c:1819-1824）：
- 写入 `PCI_BASE_ADDRESS_0`..`PCI_BASE_ADDRESS_5` 范围
- 写入 `PCI_ROM_ADDRESS` 或 `PCI_ROM_ADDRESS1`
- 写入 `PCI_COMMAND`（IO/Memory Enable 位变化）
- PM 状态变化（D0 ↔ D3）

---

## 6. PCIe 能力（Capability）与 PCIExpressDevice

### 6.1 PCIExpressDevice 结构

```c
// pcie.h:58-85
struct PCIExpressDevice {
    uint8_t exp_cap;           // PCIe 能力在配置空间的偏移
    bool hpev_notified;        // 热插拔事件逻辑与
    uint16_t aer_cap;          // AER 能力偏移
    PCIEAERLog aer_log;        // AER 错误日志
    uint16_t ats_cap;          // ATS 能力
    uint16_t pasid_cap;        // PASID 能力
    uint16_t pri_cap;          // PRI 能力
    uint16_t acs_cap;          // ACS 能力
    uint16_t sriov_cap;        // SR-IOV 能力
    PCIESriovPF sriov_pf;
    PCIESriovVF sriov_vf;
};
```

### 6.2 PCIe 能力初始化

```c
// pcie.c:223-257
int pcie_cap_init(PCIDevice *dev, uint8_t offset,
                  uint8_t type, uint8_t port, Error **errp)
{
    // 注册 PCIe Capability（v2，64 字节）
    pos = pci_add_capability(dev, PCI_CAP_ID_EXP, offset, PCI_EXP_VER2_SIZEOF, errp);
    dev->exp.exp_cap = pos;

    pcie_cap_v1_fill(dev, port, type, PCI_EXP_FLAGS_VER2);  // 填充类型/端口
    pcie_cap_fill_slot_lnk(dev);                             // 填充链路速度/宽度

    // v2 特有：Device Capabilities 2
    pci_set_long(exp_cap + PCI_EXP_DEVCAP2,
                 PCI_EXP_DEVCAP2_EFF | PCI_EXP_DEVCAP2_EETLPP);
}
```

**PCIe 设备类型**（`type` 参数）：
| 类型 | 说明 |
|------|------|
| `PCI_EXP_TYPE_ENDPOINT` | 端点设备 |
| `PCI_EXP_TYPE_ROOT_PORT` | 根端口 |
| `PCI_EXP_TYPE_UPSTREAM` | 上游端口 |
| `PCI_EXP_TYPE_DOWNSTREAM` | 下游端口 |
| `PCI_EXP_TYPE_PCI_BRIDGE` | PCI-to-PCIe 桥 |

---

## 7. MSI 中断：能力注册、消息构造与投递

### 7.1 MSI 能力初始化

```c
// msi.c:193-255
int msi_init(PCIDevice *dev, uint8_t offset,
             unsigned int nr_vectors, bool msi64bit,
             bool msi_per_vector_mask, Error **errp)
{
    // 向量数必须是 2 的幂，最大 32
    vectors_order = ctz32(nr_vectors);

    // 构造 MSI FLAGS
    flags = vectors_order << ctz32(PCI_MSI_FLAGS_QMASK);
    if (msi64bit) flags |= PCI_MSI_FLAGS_64BIT;
    if (msi_per_vector_mask) flags |= PCI_MSI_FLAGS_MASKBIT;

    // 注册 PCI Capability
    config_offset = pci_add_capability(dev, PCI_CAP_ID_MSI, offset, cap_size, errp);
    dev->msi_cap = config_offset;
    dev->cap_present |= QEMU_PCI_CAP_MSI;

    // 设置可写掩码：Address、Data、Enable
    pci_set_long(dev->wmask + msi_address_lo_off(dev), PCI_MSI_ADDRESS_LO_MASK);
    pci_set_word(dev->wmask + msi_data_off(dev, msi64bit), 0xffff);

    dev->msi_prepare_message = msi_prepare_message;  // 消息构造函数
}
```

### 7.2 MSI 消息构造

```c
// msi.c:140-163
static MSIMessage msi_prepare_message(PCIDevice *dev, unsigned int vector)
{
    MSIMessage msg;
    // 从配置空间读取 Address（32/64 位）
    if (msi64bit)
        msg.address = pci_get_quad(dev->config + msi_address_lo_off(dev));
    else
        msg.address = pci_get_long(dev->config + msi_address_lo_off(dev));

    // Data 字段：低 N 位用于向量编号
    msg.data = pci_get_word(dev->config + msi_data_off(dev, msi64bit));
    if (nr_vectors > 1) {
        msg.data &= ~(nr_vectors - 1);  // 清除低位
        msg.data |= vector;              // 填入向量号
    }
    return msg;
}
```

### 7.3 MSI 投递流程

```
设备调用 msi_notify(dev, vector)           [msi.c:353-376]
  ├── 检查向量是否被 Mask（有则设 Pending 位）
  ├── msg = msi_get_message(dev, vector)    // → msi_prepare_message
  └── msi_send_message(dev, msg)            [msi.c:378-381]
      └── dev->msi_trigger(dev, msg)        // 默认 pci_msi_trigger
          → pci_msi_trigger()               [pci.c:433-451]
            └── address_space_stl_le(&dev->bus_master_as,
                                     msg.address, msg.data, attrs, NULL)
```

**关键**：MSI 写入通过设备的 `bus_master_as` 地址空间，这意味着：
1. 如果启用了 IOMMU，MSI 地址会经过 IOMMU 翻译
2. 最终写入落在 GIC ITS 的 `GITS_TRANSLATER` 寄存器上

### 7.4 MSI 配置空间写处理

```c
// msi.c:384-431
void msi_write_config(PCIDevice *dev, uint32_t addr, uint32_t val, int len)
```

由 `pci_default_write_config()` 级联调用（pci.c:1833），处理：
- Guest 写入 MSI Address/Data/Mask 寄存器
- MSI Enable 位变化时清除 INTx（PCI 规范 6.8.3.3）
- Mask 位清除时重播 Pending 向量

---

## 8. MSI-X 中断：Table/PBA MMIO 与向量管理

### 8.1 MSI-X 初始化

```c
// msix.c:321-392
int msix_init(PCIDevice *dev, uint32_t nentries,
              MemoryRegion *table_bar, uint8_t table_bar_nr,
              unsigned table_offset,
              MemoryRegion *pba_bar, uint8_t pba_bar_nr,
              unsigned pba_offset, uint8_t cap_pos, Error **errp)
{
    table_size = nentries * PCI_MSIX_ENTRY_SIZE;     // 每条目 16 字节
    pba_size = QEMU_ALIGN_UP(nentries, 64) / 8;     // 每向量 1 位

    // 注册 MSI-X Capability
    cap = pci_add_capability(dev, PCI_CAP_ID_MSIX, cap_pos, MSIX_CAP_LENGTH, errp);
    dev->msix_cap = cap;
    dev->cap_present |= QEMU_PCI_CAP_MSIX;

    // Table Size 和 BAR 指示符
    pci_set_word(config + PCI_MSIX_FLAGS, nentries - 1);
    pci_set_long(config + PCI_MSIX_TABLE, table_offset | table_bar_nr);
    pci_set_long(config + PCI_MSIX_PBA, pba_offset | pba_bar_nr);

    // 分配内存中的 Table 和 PBA 副本
    dev->msix_table = g_malloc0(table_size);
    dev->msix_pba = g_malloc0(pba_size);
    dev->msix_entry_used = g_malloc0(nentries * sizeof(*dev->msix_entry_used));

    // 初始全部 Mask
    msix_mask_all(dev, nentries);

    // 创建 MMIO 区域并挂载到 BAR
    memory_region_init_io(&dev->msix_table_mmio, OBJECT(dev),
                          &msix_table_mmio_ops, dev, "msix-table", table_size);
    memory_region_add_subregion(table_bar, table_offset, &dev->msix_table_mmio);

    memory_region_init_io(&dev->msix_pba_mmio, OBJECT(dev),
                          &msix_pba_mmio_ops, dev, "msix-pba", pba_size);
    memory_region_add_subregion(pba_bar, pba_offset, &dev->msix_pba_mmio);
}
```

### 8.2 MSI-X Table 条目布局

每个 MSI-X 条目 16 字节（`PCI_MSIX_ENTRY_SIZE`）：

| 偏移 | 大小 | 字段 |
|------|------|------|
| 0x00 | 4B | Lower Address |
| 0x04 | 4B | Upper Address |
| 0x08 | 4B | Data |
| 0x0C | 4B | Vector Control（bit 0 = Mask） |

### 8.3 MSI-X Table MMIO 访问

```c
// msix.c:212-233
static uint64_t msix_table_mmio_read(void *opaque, hwaddr addr, unsigned size)
{
    return pci_get_long(dev->msix_table + addr);  // 直接从内存副本读
}

static void msix_table_mmio_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    int vector = addr / PCI_MSIX_ENTRY_SIZE;
    was_masked = msix_is_masked(dev, vector);
    pci_set_long(dev->msix_table + addr, val);    // 写入内存副本
    msix_handle_mask_update(dev, vector, was_masked); // 处理 Mask 变化→可能重播 Pending
}
```

### 8.4 MSI-X 投递

```c
// msix.c:540-558
void msix_notify(PCIDevice *dev, unsigned vector)
{
    if (!dev->msix_entry_used[vector]) return;  // 未使用的向量不投递
    if (msix_is_masked(dev, vector)) {
        msix_set_pending(dev, vector);           // 设置 PBA 位
        return;
    }
    msg = msix_get_message(dev, vector);         // 从 Table 读 address/data
    msi_send_message(dev, msg);                  // → pci_msi_trigger → 内存写
}
```

### 8.5 MSI-X 消息构造

```c
// msix.c:37-45
static MSIMessage msix_prepare_message(PCIDevice *dev, unsigned vector)
{
    uint8_t *table_entry = dev->msix_table + vector * PCI_MSIX_ENTRY_SIZE;
    msg.address = pci_get_quad(table_entry + PCI_MSIX_ENTRY_LOWER_ADDR);
    msg.data = pci_get_long(table_entry + PCI_MSIX_ENTRY_DATA);
    return msg;
}
```

**MSI vs MSI-X 对比**：
| 特性 | MSI | MSI-X |
|------|-----|-------|
| 向量数 | 最多 32 | 最多 2048 |
| 存储位置 | 配置空间寄存器 | 独立 MMIO Table（BAR 内） |
| 地址/数据 | 所有向量共享 Address，Data 低位区分 | 每向量独立 Address + Data |
| 掩码 | 可选（per-vector mask） | 必有（per-vector + function mask） |
| Pending | 配置空间位图 | 独立 PBA MMIO |

---

## 9. MSI/MSI-X → GIC ITS 投递路径

### 9.1 完整投递调用链

```
设备 → msix_notify(dev, vector)  或  msi_notify(dev, vector)
  → msi_send_message(dev, msg)                     [msi.c:378-381]
    → dev->msi_trigger(dev, msg)
      → pci_msi_trigger(dev, msg)                   [pci.c:433-451]
        → address_space_stl_le(&dev->bus_master_as,
                               msg.address, msg.data, attrs, NULL)
```

**MSI 地址解码**：
- ARM GIC ITS 的 `GITS_TRANSLATER` 寄存器映射在特定物理地址
- MSI 地址指向该寄存器，数据包含 EventID
- ITS 收到写入后执行中断翻译：DeviceID + EventID → LPI

### 9.2 GIC ITS 处理

```c
// arm_gicv3_its.c:514-525
static ItsCmdResult do_process_its_cmd(GICv3ITSState *s,
                                       uint32_t devid, uint32_t eventid,
                                       ItsCmdType cmd)
{
    // Device Table → ITT → ITE 查找
    cmdres = lookup_ite(s, __func__, devid, eventid, &ite, &dte);

    // 物理中断：ITE → CTE → Redistributor → LPI
    // arm_gicv3_its.c:465-477
    process_its_cmd_phys():
      lookup_cte(s, ite->icid, &cte)
      gicv3_redist_process_lpi(&cpu[cte.rdbase], ite->intid, irqlevel)

    // 虚拟中断：ITE → VTE → vLPI
    // arm_gicv3_its.c:479-503
    process_its_cmd_virt():
      lookup_vte(s, ite->vpeid, &vte)
      gicv3_redist_process_vlpi(&cpu[vte.rdbase], ite->intid, ...)
}
```

### 9.3 VirtIO PCIe 中断示例

```
virtio_pci_notify()                          [virtio-pci.c:73-85]
  └── msix_notify(dev, vector)               // VirtIO 默认使用 MSI-X
      → msix_get_message → msi_send_message
        → pci_msi_trigger → address_space_stl_le
          → [IOMMU 翻译（如有）] → GIC ITS GITS_TRANSLATER 写入
            → ITS 中断翻译 → LPI → target CPU
```

---

## 10. INTx 传统中断：路由、Swizzle 与电平追踪

### 10.1 INTx 电平追踪

```c
// pci.c:370-379
static inline int pci_irq_state(PCIDevice *d, int irq_num)
{
    return (d->irq_state >> irq_num) & 0x1;  // 4 位位图，每 pin 1 位
}
```

### 10.2 INTx 路由（Swizzle + 向上传播）

```c
// pci.c:389-405
static void pci_change_irq_level(PCIDevice *pci_dev, int irq_num, int change)
{
    PCIBus *bus;
    for (;;) {
        bus = pci_get_bus(pci_dev);
        irq_num = bus->map_irq(pci_dev, irq_num);  // Swizzle 映射
        if (bus->set_irq)  // 到达有 set_irq 的总线（通常是根总线）
            break;
        pci_dev = bus->parent_dev;                  // 继续向上级桥传播
    }
    pci_bus_change_irq_level(bus, irq_num, change);
}
```

**Swizzle 规则**（GPEX，gpex.c:83-88）：
```c
static int gpex_swizzle_map_irq_fn(PCIDevice *pci_dev, int pin)
{
    PCIBus *bus = pci_device_root_bus(pci_dev);
    return (PCI_SLOT(pci_dev->devfn) + pin) % bus->nirq;
    // (slot + pin) mod 4 → 实现跨设备的 INTx 分散
}
```

### 10.3 总线级 IRQ 到 GIC 的最终投递

```c
// pci.c:381-387
static void pci_bus_change_irq_level(PCIBus *bus, int irq_num, int change)
{
    bus->irq_count[irq_num] += change;
    bus->set_irq(bus->irq_opaque, irq_num,
                 bus->irq_count[irq_num] != 0);  // OR 语义：任一设备活跃则触发
}
```

---

## 11. ECAM 与 GPEX：ARM virt PCIe 主桥

### 11.1 GPEX（Generic PCIe Express）Host Bridge

```c
// gpex.h:53-70
struct GPEXHost {
    PCIExpressHost parent_obj;            // PCIe Host Bridge 基类
    GPEXRootState gpex_root;              // 嵌入的根设备
    MemoryRegion io_ioport;               // PCI IO 端口空间
    MemoryRegion io_mmio;                 // PCI MMIO 空间
    MemoryRegion io_ioport_window;        // IO 窗口（含 unassigned 背景）
    MemoryRegion io_mmio_window;          // MMIO 窗口
    GPEXIrq *irq;                         // INTx IRQ 数组
    uint8_t num_irqs;                     // IRQ 数量（4）
    bool allow_unmapped_accesses;         // 未映射访问返回 -1
    struct GPEXConfig gpex_cfg;           // 地址空间配置
};
```

### 11.2 GPEX 实现流程

```c
// gpex.c:90-158
static void gpex_host_realize(DeviceState *dev, Error **errp)
{
    // 1. 初始化 ECAM MMIO
    pcie_host_mmcfg_init(pex, PCIE_MMCFG_SIZE_MAX);  // 256MB 最大
    sysbus_init_mmio(sbd, &pex->mmio);                // SysBus MMIO #0 = ECAM

    // 2. 初始化 PCI MMIO 和 IO 空间
    memory_region_init(&s->io_mmio, OBJECT(s), "gpex_mmio", UINT64_MAX);
    memory_region_init(&s->io_ioport, OBJECT(s), "gpex_ioport", 64 * 1024);

    // 3. 可选：创建 "未映射访问" 窗口（读返回 -1，写忽略）
    if (s->allow_unmapped_accesses) {
        // 创建 window 容器，将实际空间作为子区域
        memory_region_init_io(&s->io_mmio_window, ..., &unassigned_io_ops, ...);
        memory_region_add_subregion(&s->io_mmio_window, 0, &s->io_mmio);
    }

    // 4. 初始化 IRQ 输出（连接到 GIC）
    for (i = 0; i < s->num_irqs; i++)
        sysbus_init_irq(sbd, &s->irq[i].irq);

    // 5. 创建根总线 "pcie.0"
    pci->bus = pci_register_root_bus(dev, "pcie.0",
                                     gpex_set_irq, gpex_swizzle_map_irq_fn,
                                     s, &s->io_mmio, &s->io_ioport,
                                     0, s->num_irqs, TYPE_PCIE_BUS);

    // 6. 设置 INTx 路由回调
    pci_bus_set_route_irq_fn(pci->bus, gpex_route_intx_pin_to_irq);

    // 7. 实现嵌入的根设备
    qdev_realize(DEVICE(&s->gpex_root), BUS(pci->bus), &error_fatal);
}
```

### 11.3 ECAM 地址解码

```c
// pcie_host.c:76-83
static void pcie_host_init(Object *obj)
{
    e->base_addr = PCIE_BASE_ADDR_UNMAPPED;
    memory_region_init_io(&e->mmio, OBJECT(e), &pcie_mmcfg_ops, e,
                          "pcie-mmcfg-mmio", PCIE_MMCFG_SIZE_MAX);
}
```

ECAM 读写处理（pcie_host.c:35-68）：
```
Guest 访问 ECAM 地址 = base + (bus << 20) | (devfn << 12) | offset
  → pcie_mmcfg_data_write/read()
    → pcie_dev_find_by_mmcfg_addr(bus, mmcfg_addr)
      → pci_find_device(bus, PCIE_MMCFG_BUS(addr), PCIE_MMCFG_DEVFN(addr))
    → pci_host_config_write/read_common(dev, offset, limit, val, len)
```

---

## 12. virt 机器 PCIe 集成：地址窗口、IRQ 与设备树

### 12.1 create_pcie 主流程

```c
// virt.c:1935-2043
static void create_pcie(VirtMachineState *vms)
{
    // 1. 创建 GPEX Host Bridge
    dev = qdev_new(TYPE_GPEX_HOST);
    sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);

    // 2. 映射 ECAM 窗口（alias 截取特定大小）
    ecam_alias = g_new0(MemoryRegion, 1);
    memory_region_init_alias(ecam_alias, ..., "pcie-ecam", ecam_reg, 0, size_ecam);
    memory_region_add_subregion(get_system_memory(), base_ecam, ecam_alias);

    // 3. 映射 MMIO 窗口（1:1 映射到系统地址空间）
    mmio_alias = g_new0(MemoryRegion, 1);
    memory_region_init_alias(mmio_alias, ..., "pcie-mmio", mmio_reg, base_mmio, size_mmio);
    memory_region_add_subregion(get_system_memory(), base_mmio, mmio_alias);

    // 4. 可选：映射高位 MMIO（> 4GB）
    if (vms->highmem_mmio) { ... }

    // 5. 映射 IO 端口窗口
    sysbus_mmio_map(SYS_BUS_DEVICE(dev), 2, base_pio);

    // 6. 连接 4 个 INTx 到 GIC SPI
    for (i = 0; i < PCI_NUM_PINS; i++) {
        sysbus_connect_irq(SYS_BUS_DEVICE(dev), i,
                           qdev_get_gpio_in(vms->gic, irq + i));
        gpex_set_irq_num(GPEX_HOST(dev), i, irq + i);
    }

    // 7. IOMMU bypass 设置
    pci->bypass_iommu = vms->default_bus_bypass_iommu;

    // 8. 设备树生成
    //   compatible = "pci-host-ecam-generic"
    //   reg = <base_ecam, size_ecam>
    //   ranges = IO/MMIO/MMIO64 窗口
    //   msi-map = <0, its_phandle, 0, 0x10000>
    //   interrupt-map = INTx Swizzle 映射到 GIC SPI

    // 9. 可选：创建 SMMU 并设置 iommu-map
    if (vms->iommu == VIRT_IOMMU_SMMUV3) {
        create_smmu(vms, vms->bus);
        qemu_fdt_setprop_cells(fdt, nodename, "iommu-map",
                               0x0, vms->iommu_phandle, 0x0, 0x10000);
    }
}
```

### 12.2 virt PCIe 地址空间布局

| 区域 | 地址范围（典型） | 大小 | 说明 |
|------|-----------------|------|------|
| ECAM | `0x40_1000_0000` | 256MB | 配置空间（bus×device×func×4KB） |
| IO Ports | `0x3eff_0000` | 64KB | PCI IO 端口映射 |
| MMIO (32-bit) | `0x1000_0000` | 512MB | 32 位 MMIO 窗口 |
| MMIO (64-bit) | `0x80_0000_0000` | 512GB | 高位 MMIO 窗口 |

### 12.3 设备树 interrupt-map

```c
// virt.c:1757-1792 — create_pcie_irq_map()
```

生成 INTx Swizzle 映射表，将 PCIe 设备的 INTA/B/C/D 映射到 GIC SPI 线。每个 slot 的 4 个 pin 按 `(slot + pin) % 4` 旋转分配到 4 条 SPI 中断线。

---

## 13. 设备生命周期：注册、实现与热插拔

### 13.1 设备实现流程

```c
// pci.c:2247-2310 — pci_qdev_realize()
static void pci_qdev_realize(DeviceState *qdev, Error **errp)
{
    // 1. 验证 acpi-index 唯一性
    // 2. 自动设置 QEMU_PCI_CAP_EXPRESS（如果实现了 INTERFACE_PCIE_DEVICE）
    if (object_class_dynamic_cast(klass, INTERFACE_PCIE_DEVICE) &&
       !object_class_dynamic_cast(klass, INTERFACE_CONVENTIONAL_PCI_DEVICE))
        pci_dev->cap_present |= QEMU_PCI_CAP_EXPRESS;

    // 3. 注册设备到总线（分配 devfn、配置空间、初始化标准头）
    pci_dev = do_pci_register_device(pci_dev, typename, pci_dev->devfn, errp);

    // 4. 调用设备特定的 realize 回调
    if (pc->realize)
        pc->realize(pci_dev, &local_err);

    // 5. ROM 加载
    // 6. MSI 触发器设置
}
```

### 13.2 do_pci_register_device 内部

```c
// pci.c:1390-1430（关键部分）
// 1. 创建 bus_master 地址空间
memory_region_init(&pci_dev->bus_master_container_region, ...);
address_space_init(&pci_dev->bus_master_as, ...);

// 2. 分配配置空间
pci_config_alloc(pci_dev);  // 5 个数组

// 3. 填充标准头
pci_config_set_vendor_id(pci_dev->config, pc->vendor_id);
pci_config_set_device_id(pci_dev->config, pc->device_id);
pci_config_set_revision(pci_dev->config, pc->revision);
pci_config_set_class(pci_dev->config, pc->class_id);

// 4. 初始化三层掩码
pci_init_cmask(pci_dev);    // 只读字段校验
pci_init_wmask(pci_dev);    // 可写位
pci_init_w1cmask(pci_dev);  // W1C 位
```

### 13.3 QOM 类型层次

```
Object
 └── DeviceState
      └── PCIDevice (TYPE_PCI_DEVICE, 抽象)
           ├── 实现 INTERFACE_PCIE_DEVICE → PCIe 设备
           ├── 实现 INTERFACE_CONVENTIONAL_PCI_DEVICE → 传统 PCI
           └── 具体设备类型
               ├── virtio-pci
               ├── e1000e
               ├── nvme
               ├── vfio-pci
               └── ...
```

---

## 总结

### 关键数据流图

```
┌──────────────┐    ECAM MMIO     ┌──────────────────┐
│  Guest CPU   │ ───────────────→ │ pcie_mmcfg_ops   │
│              │    (配置空间)     │   ↓ Bus/DevFn    │
└──────────────┘                  │   ↓ Offset       │
                                  └──────────────────┘
                                          │
                                          ▼
                              ┌────────────────────────┐
                              │ pci_host_config_write/  │
                              │ read_common()           │
                              │   ↓                     │
                              │ dev->config_write()     │
                              └────────────────────────┘
                                          │
                      ┌───────────────────┼──────────────────┐
                      ▼                   ▼                  ▼
              ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
              │ BAR 重映射    │   │ MSI 配置更新  │   │ MSI-X 配置   │
              │ pci_update_  │   │ msi_write_   │   │ msix_write_  │
              │ mappings()   │   │ config()     │   │ config()     │
              └──────────────┘   └──────────────┘   └──────────────┘

┌──────────────┐                ┌──────────────────────┐
│  设备中断    │ → msix_notify  │  msi_send_message    │
│  (如 VirtIO) │   /msi_notify  │   → pci_msi_trigger  │
└──────────────┘                │   → address_space_stl │
                                │   → [IOMMU]           │
                                │   → GIC ITS           │
                                │   → LPI → CPU         │
                                └──────────────────────┘
```

### 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/hw/pci/pci_device.h` | ~205 | PCIDeviceClass (26-44)、PCIDevice (60-190) |
| `include/hw/pci/pci_bus.h` | ~70 | PCIBus (33-59)、总线标志 |
| `include/hw/pci/pci_host.h` | ~60 | PCIHostState (39-51) |
| `include/hw/pci/pci.h` | ~350 | PCIINTxRoute (236-243)、pci_register_bar (253-254) |
| `include/hw/pci/pcie.h` | ~100 | PCIExpressDevice (58-85)、pcie_cap_init (90-91) |
| `include/hw/pci/pcie_host.h` | ~80 | ECAM 地址编码宏 (60-79) |
| `include/hw/pci/msi.h` | ~45 | MSIMessage 结构、MSI API |
| `include/hw/pci/msix.h` | ~50 | MSI-X API |
| `include/hw/pci-host/gpex.h` | ~72 | GPEXHost (53-70)、GPEXConfig (41-50) |
| `hw/pci/pci.c` | ~3500 | 配置空间管理、BAR、IRQ、设备生命周期 |
| `hw/pci/pci_host.c` | ~200 | 主桥配置空间调度 |
| `hw/pci/pcie.c` | ~260 | PCIe 能力初始化 |
| `hw/pci/pcie_host.c` | ~110 | ECAM MMIO 读写 |
| `hw/pci/msi.c` | ~440 | MSI 能力、消息构造、投递 |
| `hw/pci/msix.c` | ~640 | MSI-X Table/PBA MMIO、向量管理 |
| `hw/pci-host/gpex.c` | ~260 | GPEX Host Bridge 实现 |
| `hw/arm/virt.c` | ~2043 | create_pcie (1935-2043)、PCIe DT 生成 |
| `hw/intc/arm_gicv3_its.c` | ~525 | ITS 中断翻译 |
