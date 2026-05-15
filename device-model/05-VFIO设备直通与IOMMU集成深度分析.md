# VFIO 设备直通与 IOMMU 集成深度分析

> QEMU 版本：11.0.50  
> 分析范围：VFIO 核心框架、PCI 直通、中断注入、DMA 映射、IOMMU 抽象、SMMUv3 模拟、IOMMUFD 后端、迁移支持  
> 关联文档：`darren/device-model/00-设备模型与virtio深度分析.md`、`darren/arm64/05-FDT设备树深度分析.md`

---

## 目录

1. [概述与架构总览](#1-概述与架构总览)
2. [VFIO 源码结构](#2-vfio-源码结构)
3. [核心数据结构](#3-核心数据结构)
4. [容器模型：Legacy vs IOMMUFD](#4-容器模型legacy-vs-iommufd)
5. [VFIO PCI QOM 类型层次](#5-vfio-pci-qom-类型层次)
6. [设备初始化流程（vfio_pci_realize）](#6-设备初始化流程vfio_pci_realize)
7. [PCI Config Space 处理](#7-pci-config-space-处理)
8. [BAR 映射与直通](#8-bar-映射与直通)
9. [ROM 处理](#9-rom-处理)
10. [中断处理：INTx](#10-中断处理intx)
11. [中断处理：MSI/MSI-X](#11-中断处理msimsi-x)
12. [KVM irqfd 直通路径](#12-kvm-irqfd-直通路径)
13. [DMA 映射机制](#13-dma-映射机制)
14. [MemoryListener 与动态映射](#14-memorylistener-与动态映射)
15. [脏页跟踪](#15-脏页跟踪)
16. [QEMU IOMMU 抽象层](#16-qemu-iommu-抽象层)
17. [ARM SMMUv3 模拟](#17-arm-smmuv3-模拟)
18. [virtio-iommu](#18-virtio-iommu)
19. [IOMMUFD 后端](#19-iommufd-后端)
20. [设备 Reset 机制](#20-设备-reset-机制)
21. [热插拔支持](#21-热插拔支持)
22. [迁移支持](#22-迁移支持)
23. [vhost + IOMMU 交互](#23-vhost--iommu-交互)
24. [端到端数据流](#24-端到端数据流)

---

## 1. 概述与架构总览

VFIO（Virtual Function I/O）是 Linux 内核提供的用户态设备直通框架。QEMU 通过 VFIO 将物理 PCIe 设备（或 SR-IOV VF、mdev 设备）直接映射给 Guest，实现接近原生的 I/O 性能。

**VFIO 直通架构全景图**：

```
┌─────────────────────────────────────────────────────────────────┐
│                         Guest VM                                │
│                                                                 │
│   Guest Driver ←→ PCIe Config/BAR/MSI-X ←→ 物理设备寄存器       │
│        │                                         ▲              │
│        │ DMA 请求                                 │ 中断         │
│        ▼                                         │              │
│   Guest 物理地址 ─────────────────────────────────┘              │
└──────────┬──────────────────────────────────────────────────────┘
           │
    ┌──────┴───────┐
    │  QEMU VFIO   │
    │              │
    │ VFIOPCIDevice│
    │  ├─ Config   │ ← 部分模拟（MSI/MSI-X/BAR sizing）
    │  ├─ BARs     │ ← mmap 快速路径 / trap 慢速路径
    │  ├─ INTx/MSI │ ← eventfd + KVM irqfd
    │  └─ DMA map  │ ← MemoryListener 跟踪 Guest 内存
    └──────┬───────┘
           │
    ┌──────┴───────────────────────────────────────┐
    │           Linux Kernel                        │
    │                                               │
    │  /dev/vfio/<group>  或  /dev/iommu (IOMMUFD)  │
    │       │                       │               │
    │  VFIO Group/Device      IOMMUFD IOAS          │
    │       │                       │               │
    │  ┌────┴───────────────────────┴────┐          │
    │  │        硬件 IOMMU               │          │
    │  │   (Intel VT-d / ARM SMMU)       │          │
    │  │   GPA → HPA 地址翻译            │          │
    │  │   DMA 隔离 & 保护               │          │
    │  └─────────────────────────────────┘          │
    └───────────────────────────────────────────────┘
```

---

## 2. VFIO 源码结构

```
hw/vfio/
├── container.c          352 行  容器基类，DMA map/unmap 分发
├── container-legacy.c  1269 行  Legacy 容器（/dev/vfio/vfio + group）
├── iommufd.c           1021 行  IOMMUFD 容器后端
├── device.c             659 行  VFIODevice 生命周期管理
├── listener.c          1306 行  MemoryListener DMA 映射跟踪
├── pci.c               4035 行  VFIO PCI 设备实现（核心）
├── pci-quirks.c               PCI 设备特定 quirk 处理
├── migration.c         1317 行  迁移状态机
├── migration-multifd.c        多 fd 迁移支持
├── region.c                   VFIORegion 辅助函数
├── helpers.c                  通用辅助函数
├── kvm-helpers.c              KVM irqfd 辅助
├── display.c                  VFIO display (GPU) 支持
├── igd.c                      Intel 集显直通
├── ccw.c                      s390 CCW 设备
├── ap.c                       s390 AP 设备
├── spapr.c / kvm-spapr.c      PPC sPAPR 支持
├── cpr*.c                     CPR（checkpoint/restore）支持
└── meson.build / Kconfig

include/hw/vfio/
├── vfio-device.h        设备核心结构
├── vfio-container.h     容器基类
├── vfio-container-legacy.h  Legacy 容器
├── vfio-migration.h     迁移接口
└── vfio-cpr.h           CPR 接口

相关文件：
├── backends/iommufd.c   623 行  IOMMUFDBackend 对象
├── include/system/iommufd.h   IOMMUFD API
├── hw/arm/smmuv3.c     2251 行  ARM SMMUv3 模拟
├── hw/virtio/virtio-iommu.c 1722 行  virtio-iommu
└── hw/virtio/vhost.c          vhost + IOMMU 交互
```

---

## 3. 核心数据结构

### 3.1 VFIODevice — 设备基础结构

```c
// vfio-device.h:52-95
typedef struct VFIODevice {
    QLIST_ENTRY(VFIODevice) next;            // 组设备链表
    QLIST_ENTRY(VFIODevice) container_next;  // 容器设备链表
    QLIST_ENTRY(VFIODevice) global_next;     // 全局设备链表
    struct VFIOGroup *group;                 // Legacy: 所属组
    VFIOContainer *bcontainer;               // 所属容器
    char *sysfsdev;                          // sysfs 路径
    char *name;                              // 设备名称
    DeviceState *dev;                        // QOM 设备
    int fd;                                  // VFIO 设备 fd
    int type;                                // PCI/CCW/AP
    bool mdev;                               // 是否 mdev 设备
    bool reset_works;                        // 设备是否支持 reset
    VFIODeviceOps *ops;                      // 设备操作回调
    VFIODeviceIOOps *io_ops;                 // I/O 操作回调
    unsigned int num_irqs;                   // IRQ 数量
    unsigned int num_initial_regions;        // Region 数量
    VFIOMigration *migration;                // 迁移状态
    IOMMUFDBackend *iommufd;                 // IOMMUFD 后端（可选）
    VFIOIOASHwpt *hwpt;                      // IOMMUFD HWPT
    struct vfio_region_info **reginfo;       // Region 信息缓存
} VFIODevice;
```

### 3.2 VFIOContainer — 容器基类

```c
// vfio-container.h:36-56
struct VFIOContainer {
    Object parent_obj;
    VFIOAddressSpace *space;          // 关联的 AddressSpace
    MemoryListener listener;          // 内存变更监听器
    uint64_t dirty_pgsizes;           // 脏页跟踪页面大小
    unsigned long pgsizes;            // 支持的页面大小
    unsigned int dma_max_mappings;    // 最大 DMA 映射数
    bool dirty_pages_supported;       // 是否支持脏页跟踪
    bool dirty_pages_started;         // 脏页跟踪是否已启动
    QLIST_HEAD(, VFIOGuestIOMMU) giommu_list;  // Guest IOMMU 列表
    QLIST_ENTRY(VFIOContainer) next;  // 容器链表
    QLIST_HEAD(, VFIODevice) device_list;      // 设备链表
    GList *iova_ranges;               // IOVA 可用范围
};
```

### 3.3 对象关系图

```
VFIOAddressSpace (per AddressSpace)
    │
    ├── VFIOContainer (Legacy 或 IOMMUFD)
    │       │
    │       ├── MemoryListener ── 跟踪 Guest 内存变更 → DMA map/unmap
    │       │
    │       ├── VFIODevice (设备1)
    │       │       ├── fd (VFIO device fd)
    │       │       ├── VFIOGroup (Legacy only)
    │       │       └── IOMMUFDBackend (IOMMUFD only)
    │       │
    │       └── VFIODevice (设备2)
    │
    └── VFIOContainer (另一个容器)

Legacy 特有：
  VFIOLegacyContainer → VFIOContainer
      ├── container_fd (/dev/vfio/vfio)
      └── VFIOGroup
            ├── groupid
            ├── fd (/dev/vfio/<group>)
            └── device_list

IOMMUFD 特有：
  VFIOIOMMUFDContainer → VFIOContainer
      ├── IOMMUFDBackend (/dev/iommu fd)
      └── ioas_id (IOAS 标识)
```

---

## 4. 容器模型：Legacy vs IOMMUFD

### 4.1 选择逻辑

```c
// device.c:462-471
bool vfio_device_attach(char *name, VFIODevice *vbasedev,
                        AddressSpace *as, Error **errp)
{
    const char *iommu_type = vbasedev->iommufd ?
                             TYPE_VFIO_IOMMU_IOMMUFD :    // 新路径
                             TYPE_VFIO_IOMMU_LEGACY;       // 传统路径
    return vfio_device_attach_by_iommu_type(iommu_type, name, vbasedev, as, errp);
}
```

### 4.2 Legacy 容器路径

```
用户空间                              内核空间
─────────                            ─────────
open("/dev/vfio/vfio")  ────────►  VFIO container fd
open("/dev/vfio/<grp>") ────────►  VFIO group fd
ioctl(SET_CONTAINER)    ────────►  group → container 绑定
ioctl(SET_IOMMU, TYPE1) ────────►  选择 IOMMU 类型
ioctl(GROUP_GET_DEVICE_FD) ─────►  获取设备 fd
ioctl(MAP_DMA)          ────────►  建立 GPA→HPA 映射
```

**代码路径**（`container-legacy.c`）：
- `vfio_group_get()` 打开 `/dev/vfio/<groupid>`（`container-legacy.c:762-814`）
- `vfio_container_connect()` 打开 `/dev/vfio/vfio`，绑定 IOMMU 类型（`container-legacy.c:609`）
- `vfio_legacy_dma_map()` 使用 `VFIO_IOMMU_MAP_DMA`（`container-legacy.c:199-229`）

### 4.3 IOMMUFD 容器路径

```
用户空间                              内核空间
─────────                            ─────────
open("/dev/iommu")      ────────►  IOMMUFD fd
open("/dev/vfio/devices/X") ────►  VFIO cdev fd
ioctl(BIND_IOMMUFD)     ────────►  设备 → IOMMUFD 绑定
ioctl(IOAS_ALLOC)       ────────►  分配 IOAS（I/O Address Space）
ioctl(IOAS_MAP)         ────────►  建立 IOVA→HPA 映射
ioctl(ATTACH_IOAS)      ────────►  设备 → IOAS 关联
```

**代码路径**（`iommufd.c` + `backends/iommufd.c`）：
- `iommufd_backend_connect()` 打开 `/dev/iommu`（`backends/iommufd.c:125`）
- `VFIO_DEVICE_BIND_IOMMUFD` 绑定设备（`hw/vfio/iommufd.c:130-173`）
- `iommufd_backend_map_dma()` 建立映射（`backends/iommufd.c:167-233`）

### 4.4 对比

| 特性 | Legacy | IOMMUFD |
|------|--------|---------|
| 内核接口 | `/dev/vfio/vfio` + group | `/dev/iommu` + cdev |
| 设备隔离粒度 | IOMMU group | 单设备（理想情况） |
| 嵌套 IOMMU | 不支持 | 支持（HWPT 嵌套） |
| 脏页跟踪 | `VFIO_IOMMU_DIRTY_PAGES` | IOMMUFD dirty API |
| 未来方向 | 维护模式 | 主推路径 |

---

## 5. VFIO PCI QOM 类型层次

```c
// pci.c:3716-3739, 3973-4035
TYPE_PCI_DEVICE
    └── TYPE_VFIO_PCI_DEVICE ("vfio-pci-device")     // 抽象基类
            ├── config_read  = vfio_pci_read_config
            ├── config_write = vfio_pci_write_config
            ├── exit         = vfio_exitfn
            │
            └── TYPE_VFIO_PCI ("vfio-pci")            // 具体可实例化
            │       ├── realize = vfio_pci_realize
            │       ├── instance_init = vfio_pci_init
            │       └── instance_finalize = vfio_pci_finalize
            │
            └── TYPE_VFIO_PCI_NOHOTPLUG ("vfio-pci-nohotplug")
                    └── hotpluggable = false
```

**主要属性**（`pci.c:3743-3820`）：

| 属性 | 类型 | 说明 |
|------|------|------|
| `host` | PCIHostDeviceAddress | 主机设备 BDF（DDDD:BB:DD.F） |
| `sysfsdev` | string | sysfs 设备路径 |
| `fd` | string | 预打开的设备 fd |
| `vf-token` | UUID | SR-IOV VF token |
| `display` | OnOffAuto | 显示直通（VGA/dma-buf） |
| `x-no-mmap` | bool | 禁用 BAR mmap |
| `x-intx-mmap-timeout-ms` | uint32 | INTx 时 mmap 超时 |
| `enable-migration` | OnOffAuto | 迁移支持 |

---

## 6. 设备初始化流程（vfio_pci_realize）

```c
// pci.c:3451-3608 — vfio_pci_realize()
```

**初始化序列图**：

```
vfio_pci_realize(pdev)
    │
    ├─1─ 解析主机设备路径
    │    ├── host= BDF → sysfsdev = "/sys/bus/pci/devices/DDDD:BB:DD.F"
    │    └── vfio_device_get_name()  (device.c:313-356)
    │
    ├─2─ 检测 mdev 设备
    │    └── vfio_device_is_mdev()
    │
    ├─3─ 连接到 IOMMU 容器
    │    └── vfio_device_attach(name, vbasedev, as)
    │         ├── Legacy: 打开 group fd → container fd → GET_DEVICE_FD
    │         └── IOMMUFD: BIND_IOMMUFD → IOAS → ATTACH
    │
    ├─4─ 获取设备信息
    │    └── vfio_pci_populate_device()  (pci.c:3021)
    │         ├── 查询 region 信息（BARs, config, VGA）
    │         └── 查询 IRQ 能力（INTx, MSI, MSI-X, ERR, REQ）
    │
    ├─5─ Config Space 设置
    │    └── vfio_pci_config_setup()  (pci.c:3293-3410)
    │         ├── 读取主机 config → 复制到 Guest config
    │         ├── 设置 emulated_config_bits（模拟区域掩码）
    │         └── BAR 发现与准备
    │
    ├─6─ 设置 vIOMMU（非 mdev）
    │    └── pci_device_set_iommu_device()
    │
    ├─7─ 添加 PCI Capabilities
    │    └── vfio_pci_add_capabilities()  (pci.c:3521)
    │         ├── MSI / MSI-X capability 设置
    │         ├── PCIe / PM / AER capability
    │         └── Quirk 处理
    │
    ├─8─ 中断设置
    │    └── vfio_pci_interrupt_setup()  (pci.c:3413-3449)
    │         └── vfio_intx_enable()（默认启用 INTx）
    │
    ├─9─ BAR 注册
    │    ├── vfio_bars_prepare()  → 获取 BAR 大小和类型
    │    └── vfio_bars_register() → 创建 MemoryRegion + mmap
    │
    └─10─ 迁移/显示/reset 设置
          ├── vfio_migration_realize()
          ├── vfio_display_probe()
          └── 注册 reset 处理器
```

---

## 7. PCI Config Space 处理

### 7.1 读取路径

```c
// pci.c:1385-1416 — vfio_pci_read_config()
static uint32_t vfio_pci_read_config(PCIDevice *pdev, uint32_t addr, int len)
{
    VFIOPCIDevice *vdev = VFIO_PCI_DEVICE(pdev);
    uint32_t emu_bits = 0, emu_val = 0, phys_val = 0;

    // 1. 检查哪些 bit 由 QEMU 模拟
    memcpy(&emu_bits, vdev->emulated_config_bits + addr, len);
    emu_bits = le32_to_cpu(emu_bits);

    // 2. 模拟部分从 QEMU PCIDevice config 读取
    if (emu_bits) {
        emu_val = pci_default_read_config(pdev, addr, len);
    }

    // 3. 物理部分从 VFIO config region 读取
    if (~emu_bits & (0xffffffffU >> (32 - len * 8))) {
        vfio_pci_config_space_read(vdev, addr, len, &phys_val);
    }

    // 4. 合并：模拟 bit 用模拟值，其余用物理值
    return (emu_val & emu_bits) | (phys_val & ~emu_bits);
}
```

### 7.2 写入路径

```c
// pci.c:1418-1491 — vfio_pci_write_config()
// 1. 写入 VFIO config region（让主机设备看到）
// 2. 调用 pci_default_write_config() 更新 QEMU 模拟状态
// 3. 处理副作用：
//    - MSI/MSI-X enable/disable 切换
//    - BAR 地址变更 → 重新映射
//    - Command register 变更（bus master, memory space）
```

### 7.3 模拟 vs 直通分类

| Config 区域 | 处理方式 | 说明 |
|-------------|---------|------|
| Vendor/Device ID | 可覆盖 | 命令行 vendor_id/device_id 属性 |
| Command Register | 混合 | QEMU 跟踪 bus master/memory space |
| BAR 地址 | 模拟 | QEMU 管理 BAR sizing |
| ROM BAR | 模拟 | 通过 VFIO ROM region 读取内容 |
| MSI Capability | 完全模拟 | QEMU 管理 MSI 向量 + eventfd |
| MSI-X Capability | 完全模拟 | QEMU 管理 MSI-X table + PBA |
| PCIe Capability | 部分模拟 | link status 等由 QEMU 管理 |
| 其他 Capability | 直通 | 透传到物理设备 |

---

## 8. BAR 映射与直通

### 8.1 BAR 发现与准备

```c
// pci.c:1912-1950 — vfio_bars_prepare()
// 从 VFIO region info 获取每个 BAR 的大小、类型、偏移
// 对比主机 config space 和 VFIO region info

// pci.c:1952-1977 — vfio_bar_register()
// 为每个 BAR 创建 MemoryRegion 并注册到 PCIDevice
```

### 8.2 快速路径（mmap）

```
Guest MMIO 访问
      │
      ├── 地址落在 BAR mmap 区域 → 直接访问物理设备寄存器
      │   （KVM EPT/Stage-2 页表直接映射，无 VM-Exit）
      │
      └── 地址落在非 mmap 区域 → trap 到 QEMU
          └── vfio_region_read/write() → ioctl 读写
```

```c
// pci.c:1967-1974
// mmap 映射尝试
ret = vfio_region_mmap(&bar->region);
if (ret) {
    // "Performance may be slow" 警告
    error_report("... Failed to mmap ... Performance may be slow");
}
```

### 8.3 Sub-page BAR 处理

```c
// pci.c:1339-1380 — vfio_sub_page_bar_update_mapping()
// 小于页面大小的 BAR 需要特殊处理：
// - 扩展到页面大小进行 mmap
// - 或者完全使用 trap 模式
```

### 8.4 数据流

```
                    ┌─────────────────────────────┐
                    │       Guest VM              │
                    │                             │
                    │  BAR 0: 0xfe000000-0xfe0ffff │
                    │       │                     │
                    └───────┼─────────────────────┘
                            │
              ┌─────────────┼─────────────────┐
              │ mmap 区域   │  非 mmap 区域    │
              ▼             ▼                  │
     ┌────────────┐  ┌────────────────┐        │
     │ KVM EPT    │  │ QEMU trap      │        │
     │ 直接映射   │  │ handler        │        │
     │ 无 VM-Exit │  │ pread/pwrite   │        │
     └─────┬──────┘  └───────┬────────┘        │
           │                 │                  │
           ▼                 ▼                  │
     ┌─────────────────────────────────┐        │
     │    物理 PCIe 设备 BAR 寄存器     │        │
     └─────────────────────────────────┘        │
```

---

## 9. ROM 处理

```c
// pci.c:1025-1111 — vfio_pci_load_rom()
// 从 VFIO ROM region 读取 Option ROM 内容
// 验证 ROM signature (0x55AA)
// 复制到 QEMU MemoryRegion

// pci.c:1131-1246 — vfio_pci_size_rom()
// 确定 ROM 大小
// 注册 ROM BAR 的 MemoryRegion
// 延迟加载：首次 Guest 访问时读取
```

---

## 10. 中断处理：INTx

### 10.1 启用流程

```c
// pci.c:322-385 — vfio_intx_enable()
static bool vfio_intx_enable(VFIOPCIDevice *vdev, Error **errp)
{
    // 1. 读取 Interrupt Pin
    pin = vfio_pci_read_config(pdev, PCI_INTERRUPT_PIN, 1);

    // 2. 创建 eventfd
    vfio_notifier_init(vdev, &vdev->intx.interrupt, "intx-interrupt", 0);

    // 3. 注册 QEMU fd handler
    qemu_set_fd_handler(fd, vfio_intx_interrupt, NULL, vdev);

    // 4. 告诉内核 VFIO 在此 eventfd 上触发中断
    vfio_device_irq_set_signaling(&vdev->vbasedev,
        VFIO_PCI_INTX_IRQ_INDEX, 0,
        VFIO_IRQ_SET_ACTION_TRIGGER, fd);

    // 5. 尝试 KVM irqfd 直通
    if (kvm_enabled()) {
        vfio_intx_enable_kvm(vdev);
    }
}
```

### 10.2 INTx 中断流

```
物理设备产生 INTx
      │
      ▼
VFIO 内核驱动
      │ eventfd_signal()
      ▼
┌─────────────────────────────────────────┐
│ 路径 A: KVM irqfd（快速路径）            │
│   eventfd → KVM irqfd → 直接注入 Guest  │
│   无 QEMU 参与                          │
│                                         │
│ 路径 B: QEMU 处理（慢速路径）            │
│   eventfd → vfio_intx_interrupt()       │
│   → pci_irq_assert() → KVM irq inject  │
└─────────────────────────────────────────┘
      │
      ▼
Guest ISR 处理
      │
      ▼
vfio_pci_intx_eoi()
      │ → pci_irq_deassert()
      │ → 重新使能 BAR mmap
      └→ vfio_device_irq_unmask() → 内核重新使能物理 INTx
```

### 10.3 INTx mmap 互斥

INTx 中断到来时禁用 BAR mmap（因为 INTx 需要 deassert），使用定时器延迟重新启用：

```c
// pci.c 中 intx.mmap_timer
// 中断到来 → vfio_mmap_set_enabled(false) → BAR 访问变为 trap
// EOI 后 → 启动定时器 → 超时后 vfio_mmap_set_enabled(true)
```

---

## 11. 中断处理：MSI/MSI-X

### 11.1 MSI 启用

```c
// pci.c:859-933 — vfio_msi_enable()
static void vfio_msi_enable(VFIOPCIDevice *vdev)
{
    vfio_disable_interrupts(vdev);       // 禁用之前的 INTx

    vdev->nr_vectors = msi_nr_vectors_allocated(pdev);

    // 批量准备 KVM MSI virq 路由
    vfio_pci_prepare_kvm_msi_virq_batch(vdev);

    // 为每个向量创建 eventfd + KVM irqfd
    for (i = 0; i < vdev->nr_vectors; i++) {
        vfio_notifier_init(...);
        qemu_set_fd_handler(fd, vfio_msi_interrupt, NULL, vector);
        vfio_pci_add_kvm_msi_virq(vdev, vector, i, false);
    }

    // 批量提交 KVM 路由
    vfio_pci_commit_kvm_msi_virq_batch(vdev);

    // 向内核 VFIO 注册所有 eventfd
    vfio_enable_vectors(vdev, false);
}
```

### 11.2 MSI-X 启用

```c
// pci.c:804-857 — vfio_msix_enable()
// 类似 MSI，但使用 MSI-X table
// vfio_msix_early_setup() 在 realize 阶段预读 MSI-X 信息
```

### 11.3 向量注册到内核

```c
// pci.c:496-556 — vfio_enable_vectors()
// 构建 VFIO_DEVICE_SET_IRQS 请求
// 将所有 eventfd 数组一次性传给内核
// 内核 VFIO 驱动配置物理设备的 MSI-X 表
```

### 11.4 MSI/MSI-X 中断流

```
物理设备 → MSI-X 中断
      │
      ▼
VFIO 内核驱动
      │ eventfd_signal(vector_fd)
      ▼
┌─────────────────────────────────────────────┐
│ 路径 A: KVM irqfd                            │
│   eventfd → KVM irqchip → 直接注入 Guest     │
│   零拷贝，无 QEMU 上下文切换                  │
│                                              │
│ 路径 B: QEMU userspace                       │
│   eventfd → vfio_msi_interrupt()             │
│   → msi_notify() / msix_notify()             │
│   → KVM inject                               │
└──────────────────────────────────────────────┘
```

---

## 12. KVM irqfd 直通路径

KVM irqfd 是 VFIO 高性能中断的关键——物理设备中断直接注入 Guest，完全绕过 QEMU：

```c
// pci.c:558-606 — vfio_pci_add_kvm_msi_virq()
// 1. 分配 KVM virq（GSI）
// 2. kvm_irqchip_add_msi_route() — 创建 MSI 路由
// 3. kvm_irqchip_add_irqfd_notifier_gsi() — 绑定 eventfd → virq

// pci.c:175-186 — INTx KVM irqfd
// kvm_irqchip_add_irqfd_notifier_gsi(kvm_state, interrupt, unmask, irq)
// interrupt: 中断 eventfd
// unmask: 重新使能 eventfd（INTx 特有）
```

**irqfd 数据路径**：
```
物理设备中断 → VFIO 内核驱动 → eventfd_signal()
                                    │
                              KVM irqfd 监听
                                    │
                              直接写入 Guest
                              LAPIC/GIC 寄存器
                                    │
                              Guest ISR 执行
```

---

## 13. DMA 映射机制

### 13.1 映射分发

```c
// container.c:77-104
int vfio_container_dma_map(VFIOContainer *bcontainer, hwaddr iova,
                           ram_addr_t size, void *vaddr, bool readonly)
{
    // 调用后端特定的 dma_map 方法
    return VFIO_IOMMU_GET_CLASS(bcontainer)->dma_map(bcontainer,
                                                     iova, size, vaddr, readonly);
}

int vfio_container_dma_unmap(VFIOContainer *bcontainer, hwaddr iova,
                             ram_addr_t size, IOMMUTLBEntry *iotlb)
{
    return VFIO_IOMMU_GET_CLASS(bcontainer)->dma_unmap(bcontainer,
                                                       iova, size, iotlb);
}
```

### 13.2 Legacy DMA 映射

```c
// container-legacy.c:199-229
// 使用 VFIO_IOMMU_MAP_DMA ioctl
struct vfio_iommu_type1_dma_map map = {
    .argsz = sizeof(map),
    .flags = VFIO_DMA_MAP_FLAG_READ | (readonly ? 0 : VFIO_DMA_MAP_FLAG_WRITE),
    .vaddr = (__u64)(uintptr_t)vaddr,  // Host 虚拟地址
    .iova  = iova,                      // Guest 物理地址（IOVA）
    .size  = size,
};
ioctl(container_fd, VFIO_IOMMU_MAP_DMA, &map);
```

### 13.3 IOMMUFD DMA 映射

```c
// hw/vfio/iommufd.c:37-57
// 使用 iommufd_backend_map_dma()
// 底层调用 IOMMU_IOAS_MAP ioctl
```

### 13.4 地址翻译

```
Guest DMA 请求: GPA (Guest Physical Address)
      │
      │ 硬件 IOMMU（VT-d / SMMU）
      ▼
IOVA → HPA 翻译（由 VFIO DMA mapping 建立）
      │
      ▼
Host Physical Address → 物理内存访问
```

---

## 14. MemoryListener 与动态映射

VFIO 使用 `MemoryListener` 监听 Guest 内存布局变更，自动维护 DMA 映射：

```c
// listener.c — VFIO MemoryListener 实现
// 当 Guest 内存布局变更时（RAM 添加/移除/映射变化）：
//   region_add  → vfio_container_dma_map()
//   region_del  → vfio_container_dma_unmap()
//
// 容器初始化时注册 listener：
//   memory_listener_register(&container->listener, as)
```

**触发场景**：
- VM 启动时 RAM 初始映射
- 热插拔内存（DIMM 添加/移除）
- balloon 膨胀/收缩
- Guest IOMMU 映射变更

---

## 15. 脏页跟踪

### 15.1 Legacy 路径

```c
// container-legacy.c:231-295
// 启动/停止脏页跟踪
vfio_legacy_set_dirty_page_tracking()
    → ioctl(VFIO_IOMMU_DIRTY_PAGES, START/STOP)

// 查询脏页位图
vfio_legacy_query_dirty_bitmap()
    → ioctl(VFIO_IOMMU_DIRTY_PAGES, GET_BITMAP)
```

### 15.2 IOMMUFD 路径

```c
// hw/vfio/iommufd.c:188+
iommufd_set_dirty_page_tracking()
iommufd_query_dirty_bitmap()
// 使用 IOMMUFD 的 dirty tracking API
```

### 15.3 脏页跟踪用途

脏页跟踪主要服务于 **VM 迁移**：
1. 启动脏页跟踪 → IOMMU 记录 DMA 写入的页面
2. 定期查询脏位图 → 传输脏页到目标机
3. 停止跟踪 → 最终一致性

---

## 16. QEMU IOMMU 抽象层

### 16.1 IOMMUMemoryRegion

```c
// include/system/memory.h:156-220
// IOMMUMemoryRegion 是 MemoryRegion 的子类
// 提供地址翻译接口

struct IOMMUMemoryRegionClass {
    // 核心翻译回调
    IOMMUTLBEntry (*translate)(IOMMUMemoryRegion *iommu,
                               hwaddr addr, bool is_write, ...);
    // 通知器注册
    int (*notify_flag_changed)(IOMMUMemoryRegion *iommu, ...);
    // 重放所有映射
    void (*replay)(IOMMUMemoryRegion *iommu, IOMMUNotifier *n);
};
```

### 16.2 在地址翻译中的集成

```c
// system/memory.c — address_space_translate()
// 当遇到 IOMMUMemoryRegion 时：
// 1. 调用 iommu->translate(addr) 获取 IOMMUTLBEntry
// 2. IOMMUTLBEntry 包含翻译后的地址和权限
// 3. 继续在翻译后的地址空间中查找
```

### 16.3 IOMMUNotifier

```c
// 外部组件（VFIO、vhost）通过 IOMMUNotifier 监听 IOMMU 映射变更
// 事件类型：
//   IOMMU_NOTIFIER_MAP    — 新映射建立
//   IOMMU_NOTIFIER_UNMAP  — 映射移除
//   IOMMU_NOTIFIER_DEVIOTLB_UNMAP — 设备 IOTLB 无效化
```

---

## 17. ARM SMMUv3 模拟

### 17.1 核心结构

```c
// include/hw/arm/smmuv3.h:28-77
typedef struct SMMUv3State {
    SMMUState parent;
    // 命令队列 / 事件队列
    SMMUQueue cmdq, evtq, priq;
    // 中断
    qemu_irq irq[4];  // eventq, priq, cmdq-sync, gerror
    // 寄存器状态
    uint32_t features;
    uint32_t sid_split;
    // ... Stream Table、Context Descriptor 相关
} SMMUv3State;
```

### 17.2 翻译流程

```
设备 DMA 请求 (StreamID + IOVA)
      │
      ▼
Stream Table Entry (STE) 查找
      │ smmu_find_ste()  (smmuv3.c:300-350)
      ▼
Context Descriptor (CD) 查找
      │ decode_cd()  (smmuv3.c:350-388)
      ▼
Stage-1 翻译（设备→虚拟）
      │
      ▼
Stage-2 翻译（虚拟→物理）
      │
      ▼
物理地址输出
```

### 17.3 命令队列

```c
// smmuv3.c:520-760 — smmuv3_cmdq_consume()
// 处理 Guest 驱动提交的 SMMU 命令：
//   CMDQ_OP_CFGI_STE     — 配置 Stream Table Entry
//   CMDQ_OP_CFGI_CD      — 配置 Context Descriptor
//   CMDQ_OP_TLBI_*       — TLB 无效化
//   CMDQ_OP_CMD_SYNC     — 命令同步
```

### 17.4 与 PCIe 集成

SMMUv3 通过 FDT `iommu-map` 属性与 PCIe 设备关联：

```dts
pcie@10000000 {
    iommu-map = <0x0 &smmu 0x0 0x10000>;
};

smmuv3@9050000 {
    compatible = "arm,smmu-v3";
    reg = <...>;
    interrupts = <...>;  // eventq, priq, cmdq-sync, gerror
};
```

---

## 18. virtio-iommu

### 18.1 概述

virtio-iommu 是一个半虚拟化 IOMMU，通过 virtio 协议与 Guest 驱动通信：

```c
// hw/virtio/virtio-iommu.c:47-69
typedef struct VirtIOIOMMUDomain {
    uint32_t id;
    GTree *mappings;            // IOVA→PA 映射表
    QLIST_HEAD(, VirtIOIOMMUEndpoint) endpoint_list;
} VirtIOIOMMUDomain;
```

### 18.2 命令处理

```c
// virtio-iommu.c:711-760+
// VIRTIO_IOMMU_T_ATTACH  — 将设备绑定到 domain
// VIRTIO_IOMMU_T_DETACH  — 解绑
// VIRTIO_IOMMU_T_MAP     — 建立 IOVA→PA 映射
// VIRTIO_IOMMU_T_UNMAP   — 移除映射
```

### 18.3 地址空间切换

```c
// virtio-iommu.c:109-133
// 当设备 attach 到 domain 时：
//   从 bypass 地址空间 → 切换到 IOMMU 地址空间
// 当设备 detach 时：
//   从 IOMMU 地址空间 → 切换回 bypass 地址空间
```

---

## 19. IOMMUFD 后端

### 19.1 IOMMUFDBackend 对象

```c
// include/system/iommufd.h:22-39
typedef struct IOMMUFDBackend {
    Object parent;
    int fd;              // /dev/iommu fd
    bool owned;          // 是否拥有 fd
    uint32_t users;      // 引用计数
} IOMMUFDBackend;
```

### 19.2 核心 API

```c
// backends/iommufd.c
iommufd_backend_connect()      // 打开 /dev/iommu
iommufd_backend_alloc_ioas()   // 分配 I/O Address Space
iommufd_backend_map_dma()      // IOVA→HVA 映射
iommufd_backend_unmap_dma()    // 解除映射
```

### 19.3 HWPT（Hardware Page Table）

IOMMUFD 支持嵌套 HWPT，用于：
- Guest 拥有自己的 IOMMU 页表（Stage-1）
- Host IOMMU 翻译 Guest 页表输出（Stage-2）
- 实现真正的嵌套虚拟化 IOMMU

```c
// include/system/iommufd.h:45-66
// vIOMMU 和 vDevice eventqueue 辅助对象
// 支持 Guest IOMMU 事件（page fault 等）传递
```

---

## 20. 设备 Reset 机制

```c
// pci.c:3641-3683 — vfio_pci_reset()
static void vfio_pci_reset(DeviceState *dev)
{
    VFIOPCIDevice *vdev = VFIO_PCI_DEVICE(pdev);

    vfio_pci_pre_reset(vdev);      // 禁用中断、清除 bus master

    // 尝试 reset 方法（优先级从高到低）：
    if (vdev->resetfn && !vdev->resetfn(vdev)) {
        goto post_reset;            // 1. 设备特定 reset
    }
    if (vdev->vbasedev.reset_works &&
        !vfio_device_reset(&vdev->vbasedev)) {
        goto post_reset;            // 2. VFIO_DEVICE_RESET（FLR）
    }
    vfio_pci_hot_reset(vdev);      // 3. PCIe Hot Reset（Bus Reset）

    vfio_pci_post_reset(vdev);     // 重新启用 INTx、清除 BAR
}
```

**Reset 能力检测**（`pci.c:2302-2333`）：
- **FLR**：PCIe Device Capabilities 的 FLR bit
- **PM Reset**：PM CSR D0→D3hot→D0 过渡
- **AF FLR**：Advanced Features Capability 的 FLR bit

---

## 21. 热插拔支持

```c
// pci.c:2741-2811 — vfio_pci_hot_reset()
// 查询热 reset 需要影响的设备组
// vfio_pci_get_pci_hot_reset_info() → 获取受影响设备列表
// vfio_pci_hot_reset_one/multi() → 执行 bus reset

// pci.c:3617-3639 — vfio_exitfn()
// 热拔出清理：
// 1. 禁用所有中断（INTx/MSI/MSI-X）
// 2. 清除 BAR 映射
// 3. 注销迁移处理器
// 4. 从 IOMMU 容器分离设备
```

`vfio-pci-nohotplug` 变体通过设置 `hotpluggable = false` 禁用热插拔。

---

## 22. 迁移支持

### 22.1 迁移状态机

```c
// migration.c:52-98 — VFIO 迁移状态
// VFIO_DEVICE_STATE_RUNNING    — 正常运行
// VFIO_DEVICE_STATE_PRE_COPY   — 预拷贝（设备继续运行）
// VFIO_DEVICE_STATE_STOP_COPY  — 停止拷贝
// VFIO_DEVICE_STATE_RESUMING   — 恢复中
// VFIO_DEVICE_STATE_P2P        — Peer-to-Peer（暂停但不拷贝）

// migration.c:133-233
vfio_migration_set_state()
    → ioctl(VFIO_DEVICE_FEATURE, VFIO_DEVICE_FEATURE_MIG_DEVICE_STATE)
```

### 22.2 迁移数据流

```
源端:                            目标端:
RUNNING                          
  │                              
  ▼ vfio_save_iterate()          
PRE_COPY ──── 设备状态数据 ───►  RESUMING
  │          （增量传输）            │
  ▼                                │
STOP_COPY ── 最终状态数据 ──►     │
  │                                │ vfio_load_state()
  ▼                                ▼
迁移完成                        RUNNING
```

### 22.3 初始化

```c
// migration.c:1012-1069 — vfio_migration_realize()
// 检查设备是否支持 MIGRATION_STOP_COPY
// 创建 VFIOMigration 状态
// 注册 savevm handlers 和 VM state notifiers
```

### 22.4 多 fd 迁移

```c
// migration-multifd.c — 多通道并行迁移
// 使用多个 fd 并行传输设备状态数据
// 属性: x-migration-multifd-transfer
```

---

## 23. vhost + IOMMU 交互

### 23.1 IOMMU 检测

```c
// hw/virtio/vhost.c:131-147
bool vhost_dev_has_iommu(struct vhost_dev *dev)
{
    // 检查 VIRTIO_F_IOMMU_PLATFORM feature
    // 如果 Guest 使能了 IOMMU，vhost 需要特殊处理
}
```

### 23.2 IOTLB 通知

```c
// hw/virtio/vhost.c:520+
// vhost_iommu_region_add() — 注册 IOMMUNotifier
// vhost_iommu_region_del() — 注销 IOMMUNotifier
//
// 当 Guest IOMMU 映射变更时：
//   IOMMUNotifier 触发 → vhost 更新内部 IOTLB
//   使用 VHOST_IOTLB_MSG 通知内核 vhost 模块
```

### 23.3 脏页同步

```c
// hw/virtio/vhost.c:223-268
// 当 IOMMU 启用时，脏页同步需要通过 IOMMU 翻译地址
// address_space_get_iotlb_entry() 获取翻译结果
```

---

## 24. 端到端数据流

### 24.1 Guest DMA 写入流程

```
Guest Driver 发起 DMA 写入
      │
      ▼
Guest 物理地址 (GPA/IOVA)
      │
      │ ┌─ 无 Guest IOMMU ─── GPA 直接作为 IOVA
      │ │
      │ └─ 有 Guest IOMMU ─── Guest IOMMU 翻译
      │         │               GVA → GPA → IOVA
      │         ▼
      │    IOMMUMemoryRegion::translate()
      │         │
      ▼         ▼
IOVA (I/O Virtual Address)
      │
      │ 硬件 IOMMU (VT-d / SMMU)
      │ 使用 VFIO DMA mapping 建立的页表
      ▼
Host Physical Address (HPA)
      │
      ▼
物理内存 / 物理设备
```

### 24.2 物理设备中断注入流程

```
物理设备产生 MSI-X 中断
      │
      ▼
VFIO 内核驱动截获中断
      │
      ▼
eventfd_signal(vector_eventfd)
      │
      ├── KVM irqfd 路径（零拷贝）
      │       │
      │       ▼
      │   KVM 直接注入 Guest vCPU
      │   （写入 virtual LAPIC/GIC）
      │       │
      │       ▼
      │   Guest ISR 执行
      │
      └── QEMU userspace 路径
              │
              ▼
          vfio_msi_interrupt()
              │
              ▼
          msix_notify() / msi_notify()
              │
              ▼
          KVM irq inject → Guest ISR
```

### 24.3 VFIO PCI 完整生命周期

```
命令行: -device vfio-pci,host=0000:01:00.0

1. QOM 对象创建
   └── vfio_pci_init()

2. PCIDevice realize
   └── vfio_pci_realize()
        ├── 解析 host BDF
        ├── 打开 VFIO group/device
        ├── 连接 IOMMU 容器
        ├── 建立 DMA 映射（全部 Guest RAM）
        ├── 配置 PCI config space
        ├── 映射 BAR（mmap 快速路径）
        ├── 设置中断（INTx → MSI/MSI-X）
        └── 注册迁移处理器

3. 运行时
   ├── BAR 访问: mmap 快速路径（无 VM-Exit）
   ├── Config 访问: 部分模拟 + 部分直通
   ├── 中断: eventfd + KVM irqfd
   └── DMA: 硬件 IOMMU 直接翻译

4. 热拔出 / VM 关闭
   └── vfio_exitfn()
        ├── 禁用中断
        ├── 解除 BAR 映射
        ├── 解除 DMA 映射
        └── 关闭 VFIO device/group fd
```

---

## 附录 A：关键源文件索引

| 文件 | 行数 | 关键函数 | 说明 |
|------|------|----------|------|
| `pci.c` | 4035 | `vfio_pci_realize()` | PCI 设备初始化入口 |
| `pci.c` | | `vfio_pci_read/write_config()` | Config space 处理 |
| `pci.c` | | `vfio_intx_enable()` | INTx 中断设置 |
| `pci.c` | | `vfio_msi_enable()` | MSI 中断设置 |
| `pci.c` | | `vfio_msix_enable()` | MSI-X 中断设置 |
| `pci.c` | | `vfio_bars_prepare/register()` | BAR 映射 |
| `pci.c` | | `vfio_pci_reset()` | 设备 Reset |
| `pci.c` | | `vfio_exitfn()` | 设备移除清理 |
| `device.c` | 659 | `vfio_device_attach()` | 设备连接容器 |
| `device.c` | | `vfio_device_prepare()` | 设备信息初始化 |
| `container.c` | 352 | `vfio_container_dma_map/unmap()` | DMA 映射分发 |
| `container-legacy.c` | 609 | `vfio_container_connect()` | Legacy 容器连接 |
| `container-legacy.c` | | `vfio_legacy_dma_map()` | Legacy DMA 映射 |
| `container-legacy.c` | | `vfio_group_get()` | Group fd 打开 |
| `iommufd.c` | 1021 | IOMMUFD 容器操作 | IOMMUFD DMA 映射 |
| `listener.c` | 1306 | MemoryListener 回调 | 动态 DMA 映射 |
| `migration.c` | 1317 | `vfio_migration_realize()` | 迁移初始化 |
| `migration.c` | | `vfio_save_iterate()` | 迁移保存 |
| `migration.c` | | `vfio_load_state()` | 迁移恢复 |
| `smmuv3.c` | 2251 | `smmuv3_cmdq_consume()` | SMMUv3 命令处理 |
| `virtio-iommu.c` | 1722 | ATTACH/MAP 命令 | virtio-iommu 命令 |
| `iommufd.c`(backends) | 623 | `iommufd_backend_connect()` | /dev/iommu 连接 |

## 附录 B：VFIO ioctl 速查表

| ioctl | 用途 | 使用位置 |
|-------|------|----------|
| `VFIO_GROUP_SET_CONTAINER` | Group 关联 Container | `container-legacy.c` |
| `VFIO_SET_IOMMU` | 选择 IOMMU 类型（TYPE1） | `container-legacy.c` |
| `VFIO_GROUP_GET_DEVICE_FD` | 获取设备 fd | `container-legacy.c` |
| `VFIO_DEVICE_GET_INFO` | 查询设备能力 | `device.c` |
| `VFIO_DEVICE_GET_REGION_INFO` | 查询 Region 信息 | `region.c` |
| `VFIO_DEVICE_SET_IRQS` | 配置中断 eventfd | `pci.c` |
| `VFIO_DEVICE_RESET` | 设备 Reset（FLR） | `pci.c` |
| `VFIO_IOMMU_MAP_DMA` | 建立 DMA 映射 | `container-legacy.c` |
| `VFIO_IOMMU_UNMAP_DMA` | 移除 DMA 映射 | `container-legacy.c` |
| `VFIO_IOMMU_DIRTY_PAGES` | 脏页跟踪控制 | `container-legacy.c` |
| `VFIO_DEVICE_BIND_IOMMUFD` | 设备绑定 IOMMUFD | `hw/vfio/iommufd.c` |
| `VFIO_DEVICE_FEATURE` | 设备功能控制（迁移等） | `migration.c` |

## 附录 C：关联文档

| 文档 | 路径 | 关联内容 |
|------|------|----------|
| 设备模型与 virtio | `darren/device-model/00-设备模型与virtio深度分析.md` | QOM 设备框架基础 |
| PCIe Host Bridge FDT | `darren/arm64/05-FDT设备树深度分析.md` | SMMUv3 FDT 节点 |
| Machine 建立流程 | `darren/architecture/03-Machine建立流程深度分析.md` | PCIe 总线创建 |
| 中断注入 | `darren/arm64/04-中断注入与处理深度分析.md` | GIC 中断路径 |
| 内存子系统 | `darren/memory/00-内存子系统深度分析.md` | MemoryRegion/IOMMU |
| 块层 IO | `darren/device-model/02-块层IO路径深度分析.md` | virtio-blk 后端 |
