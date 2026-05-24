# QEMU virt Machine 初始化深度分析：内存布局 / GIC 创建 / PCIe / ACPI / FDT / 固件加载

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 分析范围：hw/arm/virt.c (4265行)、hw/arm/virt-acpi-build.c (1523行)、hw/arm/boot.c (1310行)  
> 关联文档：[Machine建立流程](03-Machine建立流程深度分析.md) · [QOM对象模型](22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md) · [GIC深度分析](../arm64/gic/)  

---

## 目录

1. [概述与架构定位](#1-概述与架构定位)
2. [VirtMachineState 核心数据结构](#2-virtmachinestate-核心数据结构)
3. [物理地址空间内存映射](#3-物理地址空间内存映射)
4. [machvirt_init() 主初始化流程](#4-machvirt_init-主初始化流程)
5. [PA 空间计算与 HighMem 布局](#5-pa-空间计算与-highmem-布局)
6. [GIC 版本确定与创建](#6-gic-版本确定与创建)
7. [CPU 创建与属性配置](#7-cpu-创建与属性配置)
8. [Secure/NonSecure 内存视图](#8-securenonsecure-内存视图)
9. [固件加载：PFlash 与 BIOS](#9-固件加载pflash-与-bios)
10. [设备创建序列](#10-设备创建序列)
11. [PCIe 主桥（GPEX）与 ECAM](#11-pcie-主桥gpex与-ecam)
12. [VirtIO MMIO Transport](#12-virtio-mmio-transport)
13. [FDT 设备树生成](#13-fdt-设备树生成)
14. [ACPI 表生成框架](#14-acpi-表生成框架)
15. [Kernel 引导与 CPU Reset](#15-kernel-引导与-cpu-reset)
16. [PSCI Conduit 决策](#16-psci-conduit-决策)
17. [MTE Tag Memory 支持](#17-mte-tag-memory-支持)
18. [Machine Class 属性与版本兼容](#18-machine-class-属性与版本兼容)
19. [virt_machine_done() 后处理](#19-virt_machine_done-后处理)
20. [与真实硬件平台的对比](#20-与真实硬件平台的对比)

---

## 1. 概述与架构定位

`mach-virt` 是 QEMU ARM64 的**参考虚拟平台**，设计原则：

- **纯设备树驱动**：所有设备通过 FDT 描述，Guest 内核无需平台特定代码
- **最小攻击面**：仅提供 Linux 可纯 DT 驱动的设备
- **类 kvmtool 方案**：不模拟真实 SoC（如 Raspberry Pi），而是抽象虚拟平台

核心文件：

| 文件 | 行数 | 职责 |
|------|------|------|
| `hw/arm/virt.c` | 4265 | Machine 初始化、设备创建、FDT 生成 |
| `hw/arm/virt-acpi-build.c` | 1523 | ACPI 表（DSDT/MADT/GTDT/IORT 等）生成 |
| `hw/arm/boot.c` | 1310 | Kernel/DTB 加载、CPU Reset 回调 |
| `include/hw/arm/virt.h` | ~200 | VirtMachineState/Class 定义、枚举 |

---

## 2. VirtMachineState 核心数据结构

```c
// include/hw/arm/virt.h:145-195
struct VirtMachineState {
    MachineState parent;
    
    /* 状态标志 */
    bool secure;                    // EL3 TrustZone 支持
    bool virt;                      // EL2 虚拟化扩展
    bool ras;                       // RAS 错误报告
    bool mte;                       // Memory Tagging Extension
    bool highmem;                   // 允许高地址 MMIO
    bool highmem_compact;           // 紧凑高地址布局
    
    /* GIC 配置 */
    VirtGICType gic_version;        // 2/3/4/host/max
    VirtMSIControllerType msi_controller;  // ITS/GICv2M/none
    DeviceState *gic;               // GIC 设备实例
    
    /* 内存映射 */
    MemMapEntry *memmap;            // 地址空间布局数组
    const int *irqmap;              // IRQ 号分配
    hwaddr highest_gpa;             // 最高有效 GPA
    
    /* PCIe */
    PCIBus *bus;                    // PCIe 根总线
    VirtIOMMUType iommu;            // SMMUv3/virtio-iommu
    
    /* 固件 */
    PFlashCFI01 *flash[2];          // NOR Flash ×2
    FWCfgState *fw_cfg;             // fw_cfg MMIO 设备
    
    /* 引导 */
    struct arm_boot_info bootinfo;  // Kernel 加载信息
    int psci_conduit;               // HVC/SMC/disabled
    
    /* ACPI */
    OnOffAuto acpi;                 // ACPI 开关
    DeviceState *acpi_dev;          // GED 设备
    char *oem_id;
    char *oem_table_id;
};
```

### VirtMachineClass 版本控制

```c
// include/hw/arm/virt.h:129-143
struct VirtMachineClass {
    MachineClass parent;
    bool no_tcg_its;               // 旧版本无 TCG ITS
    bool no_highmem_compact;       // 旧版本不支持紧凑布局
    bool no_kvm_steal_time;        // 旧版本无 steal time
    bool no_tcg_lpa2;              // 旧版本无 LPA2
    bool no_ns_el2_virt_timer_irq; // 旧版本无 NS EL2 虚拟定时器
    bool no_nested_smmu;           // 旧版本无嵌套 SMMU
    // ...
};
```

---

## 3. 物理地址空间内存映射

### 3.1 低地址固定映射 (base_memmap)

```
GPA 地址范围                    设备                    大小
────────────────────────────────────────────────────────────
0x0000_0000 - 0x07FF_FFFF      Flash (Boot ROM)       128 MB
0x0800_0000 - 0x0801_FFFF      CPU Peripherals         128 KB
  ├─ 0x0800_0000               GIC Distributor         64 KB
  ├─ 0x0801_0000               GIC CPU Interface       64 KB
  ├─ 0x0802_0000               GIC V2M                 4 KB
  ├─ 0x0803_0000               GIC HYP                 64 KB
  ├─ 0x0804_0000               GIC VCPU                64 KB
  ├─ 0x0808_0000               GIC ITS                 128 KB
  └─ 0x080A_0000               GIC Redistributor      ~15.4 MB
0x0900_0000                    UART0 (PL011)           4 KB
0x0901_0000                    RTC (PL031)             4 KB
0x0902_0000                    fw_cfg                  24 B
0x0903_0000                    GPIO (PL061)            4 KB
0x0904_0000                    UART1                   4 KB
0x0905_0000                    SMMUv3                  SMMU_IO_LEN
0x0908_0000                    ACPI GED                varies
0x0A00_0000 - 0x0A00_3DFF      VirtIO MMIO ×32        512 B each
0x0C00_0000 - 0x0DFF_FFFF      Platform Bus            32 MB
0x0E00_0000 - 0x0EFF_FFFF      Secure Memory           16 MB
0x1000_0000 - 0x3EFE_FFFF      PCIe MMIO (32-bit)     ~767 MB
0x3EFF_0000 - 0x3EFF_FFFF      PCIe PIO               64 KB
0x3F00_0000 - 0x3FFF_FFFF      PCIe ECAM              16 MB
0x4000_0000 (1 GiB)            RAM 起始               可配置
```

### 3.2 高地址浮动映射 (extended_memmap)

高地址区域基址**动态计算**，位于 RAM 之后，对齐到区域大小：

```c
// virt.c:233-241
static MemMapEntry extended_memmap[] = {
    [VIRT_HIGH_GIC_REDIST2] = { 0x0, 64 * MiB },      // 额外 redistributor
    [VIRT_CXL_HOST]         = { 0x0, 64 * KiB * 16 },  // CXL Host Bridge
    [VIRT_HIGH_PCIE_ECAM]   = { 0x0, 256 * MiB },      // 高地址 ECAM
    [VIRT_HIGH_PCIE_MMIO]   = { 0x0, 512 * GiB },      // 高地址 PCIe MMIO
};
```

浮动规则：
- 起始不低于 256 GiB（兼容性）
- 每个区域按自身大小对齐（如 512 GiB 区域对齐到 512 GiB 边界）
- 若某区域超出 PA 位数范围，自动禁用

### 3.3 IRQ 分配

```c
// virt.c:243-254
static const int a15irqmap[] = {
    [VIRT_UART0]        = 1,         // SPI 1
    [VIRT_RTC]          = 2,         // SPI 2
    [VIRT_PCIE]         = 3,         // SPI 3-6 (4 个 INTx)
    [VIRT_GPIO]         = 7,         // SPI 7
    [VIRT_UART1]        = 8,         // SPI 8
    [VIRT_ACPI_GED]     = 9,         // SPI 9
    [VIRT_MMIO]         = 16,        // SPI 16-47 (32 个 VirtIO)
    [VIRT_GIC_V2M]      = 48,        // SPI 48+ (V2M MSI)
    [VIRT_SMMU]         = 74,        // SPI 74+ (SMMU)
    [VIRT_PLATFORM_BUS] = 112,       // SPI 112+ (Platform Bus)
};
```

---

## 4. machvirt_init() 主初始化流程

`machvirt_init()` (virt.c:2600-2963) 是 Machine 的核心入口，调用序列：

```
machvirt_init()
  │
  ├── 1. virt_set_memmap(pa_bits)         // 计算地址空间布局
  ├── 2. finalize_gic_version()            // 确定 GIC 版本
  ├── 3. finalize_msi_controller()         // 确定 MSI 控制器
  │
  ├── 4. 创建 Secure MemoryRegion          // if (vms->secure)
  ├── 5. virt_firmware_init()              // Flash 映射 + BIOS 加载
  ├── 6. PSCI Conduit 决策                 // HVC/SMC/disabled
  │
  ├── 7. CPU 创建循环:
  │      for (n = 0; n < smp_cpus; n++) {
  │          object_new(cpu_type)
  │          设置 mp-affinity, has_el3, has_el2
  │          设置 memory, secure-memory, tag-memory
  │          qdev_realize()
  │      }
  │
  ├── 8. fdt_add_timer_nodes()             // 定时器 FDT
  ├── 9. fdt_add_cpu_nodes()               // CPU FDT
  ├── 10. RAM 映射到 sysmem
  │
  ├── 11. create_gic()                     // GIC 创建 + IRQ 连线
  ├── 12. virt_post_cpus_gic_realized()    // 后续 GIC 初始化
  ├── 13. fdt_add_pmu_nodes()
  │
  ├── 14. create_uart() ×2                 // PL011 UART
  ├── 15. create_secure_ram()              // if secure
  ├── 16. create_tag_ram()                 // if MTE
  │
  ├── 17. create_rtc()                     // PL031 RTC
  ├── 18. create_pcie()                    // GPEX PCIe Host Bridge
  ├── 19. create_acpi_ged() 或 create_gpio_devices()
  ├── 20. create_virtio_devices()          // 32 个 VirtIO MMIO Transport
  ├── 21. create_fw_cfg()                  // fw_cfg MMIO
  ├── 22. create_platform_bus()
  │
  ├── 23. arm_load_kernel()                // Kernel + DTB 加载
  └── 24. 注册 machine_done notifier       // ACPI/SMBIOS 后处理
```

---

## 5. PA 空间计算与 HighMem 布局

### 5.1 virt_set_memmap()

```c
// virt.c:2272-2335
static void virt_set_memmap(VirtMachineState *vms, int pa_bits) {
    // 1. 复制 base_memmap 到 extended_memmap
    for (i = 0; i < ARRAY_SIZE(base_memmap); i++)
        vms->memmap[i] = base_memmap[i];

    // 2. 计算 device_memory 区域
    device_memory_base = ROUND_UP(VIRT_MEM.base + ram_size, GiB);
    device_memory_size = maxram_size - ram_size + ram_slots * GiB;

    // 3. 高 IO 区域基址 = device_memory 之上
    base = device_memory_base + ROUND_UP(device_memory_size, GiB);
    if (base < 256 GiB) base = 256 GiB;  // 兼容性下限

    // 4. 设置高地址区域
    virt_set_high_memmap(vms, base, pa_bits);
}
```

### 5.2 PA bits 来源

- **TCG 模式**：从临时 CPU 实例获取 `arm_pamax(cpu)` → 通常 40-48 位
- **KVM 模式**：在 `kvm_type()` 中通过 KVM ioctl 提前确定 IPA 位数
- **`highmem=off`**：强制 32 位 PA（仅低 4 GiB）

---

## 6. GIC 版本确定与创建

### 6.1 GIC 版本选择逻辑

```c
// virt.c:2337-2477 — finalize_gic_version()
优先级：
1. 用户指定 gic-version=2/3/4 → 直接使用
2. gic-version=host → 从 KVM 查询支持的版本
3. gic-version=max → 选择加速器支持的最高版本 (4>3>2)
4. 默认 → 尝试 max
```

### 6.2 create_gic() 核心流程

```c
// virt.c:1122-1293
create_gic(VirtMachineState *vms, MemoryRegion *mem) {
    // 1. 选择 GIC 类型
    gictype = (v2) ? gic_class_name() : gicv3_class_name();
    
    // 2. 创建 GIC QOM 对象
    vms->gic = qdev_new(gictype);
    qdev_prop_set_uint32(vms->gic, "revision", revision);
    qdev_prop_set_uint32(vms->gic, "num-cpu", smp_cpus);
    qdev_prop_set_uint32(vms->gic, "num-irq", NUM_IRQS + 32);
    
    // 3. GICv3 特有配置
    if (v3/v4) {
        设置 redist-region-count (最多 2 个 region)
        if (tcg_its) 设置 has-lpi + sysmem
    }
    
    // 4. 地址映射
    sysbus_mmio_map(gic, 0, VIRT_GIC_DIST.base);    // Distributor
    sysbus_mmio_map(gic, 1, VIRT_GIC_REDIST.base);  // Redistributor (v3)
    //                 or  VIRT_GIC_CPU.base         // CPU Interface (v2)
    
    // 5. IRQ 连线 (per-CPU):
    for (i = 0; i < smp_cpus; i++) {
        // Timer → GIC PPI
        qdev_connect_gpio_out(cpu, GTIMER_*, gic_ppi[timer_irq]);
        // GIC → CPU IRQ/FIQ/VIRQ/VFIQ/NMI
        sysbus_connect_irq(gic, i,             cpu_irq);
        sysbus_connect_irq(gic, i+smp_cpus,    cpu_fiq);
        sysbus_connect_irq(gic, i+2*smp_cpus,  cpu_virq);
        sysbus_connect_irq(gic, i+3*smp_cpus,  cpu_vfiq);
        // GICv3 额外: NMI, VINMI
    }
    
    // 6. MSI 控制器
    if (ITS) create_its();
    else if (V2M) create_v2m();
}
```

### 6.3 Timer IRQ 映射

| 定时器 | PPI 号 | 用途 |
|--------|--------|------|
| GTIMER_PHYS | 30 (NS_EL1) | Guest EL1 物理定时器 |
| GTIMER_VIRT | 27 (VIRT) | Guest EL1 虚拟定时器 |
| GTIMER_HYP | 26 (NS_EL2) | Hypervisor EL2 物理定时器 |
| GTIMER_SEC | 29 (S_EL1) | Secure EL1 定时器 |
| GTIMER_HYPVIRT | 28 (NS_EL2_VIRT) | EL2 虚拟定时器 |

---

## 7. CPU 创建与属性配置

```c
// virt.c:2740-2841
for (n = 0; n < smp_cpus; n++) {
    cpuobj = object_new(possible_cpus->cpus[n].type);
    
    // MPIDR affinity
    object_property_set_int(cpuobj, "mp-affinity", arch_id);
    
    // 安全扩展
    if (!vms->secure) 
        set_bool(cpuobj, "has_el3", false);
    
    // 虚拟化扩展
    if (!vms->virt)
        set_bool(cpuobj, "has_el2", false);
    
    // 内存连接
    set_link(cpuobj, "memory", sysmem);
    if (secure) set_link(cpuobj, "secure-memory", secure_sysmem);
    
    // MTE: tag memory region
    if (mte && tcg) set_link(cpuobj, "tag-memory", tag_sysmem);
    if (mte && kvm) kvm_arm_enable_mte(cpuobj);
    
    qdev_realize(DEVICE(cpuobj), NULL, &error_fatal);
}
```

### MPIDR Affinity 构建

```c
// virt.c:2202-2217
virt_cpu_mp_affinity(vms, idx) {
    // GICv2: clustersz = 8 (GIC_TARGETLIST_BITS)
    // GICv3: clustersz = 16 (GICV3_TARGETLIST_BITS)
    return arm_build_mp_affinity(idx, clustersz);
}
// 结果: Aff0 = idx % clustersz, Aff1 = idx / clustersz
```

---

## 8. Secure/NonSecure 内存视图

```c
// virt.c:2649-2661
if (vms->secure) {
    secure_sysmem = g_new(MemoryRegion, 1);
    memory_region_init(secure_sysmem, "secure-memory", UINT64_MAX);
    // Secure 视图 = sysmem(低优先级) + secure 设备(高优先级)
    memory_region_add_subregion_overlap(secure_sysmem, 0, sysmem, -1);
}
```

架构含义：
- **NonSecure** 访问使用 `sysmem`
- **Secure** 访问使用 `secure_sysmem`，可看到所有 NonSecure 设备 + 额外 Secure 设备
- Secure RAM (16 MB @ 0x0E00_0000) 仅在 secure_sysmem 中可见

---

## 9. 固件加载：PFlash 与 BIOS

### 9.1 virt_firmware_init()

```c
// virt.c:1685-1733
固件加载优先级：
1. -drive if=pflash → 映射到 flash[0] (代码) / flash[1] (变量)
2. -bios → 加载到 flash[0] 的 MemoryRegion
3. 都没有 → 无固件 (直接 kernel 引导)
```

### 9.2 PFlash 配置

```c
// virt.c:1573-1640
两片 NOR Flash:
- flash[0]: 代码存储 (UEFI firmware image)
- flash[1]: 变量存储 (UEFI NVRAM)
均映射到 VIRT_FLASH 区域 (0x0 - 0x0800_0000)
sector_size = 256 KiB
```

返回值 `firmware_loaded` 影响后续决策：
- PSCI conduit 选择
- ACPI 是否启用
- HighMem ECAM 是否可用

---

## 10. 设备创建序列

`machvirt_init()` 中各设备创建的完整顺序和对应 FDT 节点：

| 顺序 | 函数 | 设备 | FDT 节点 |
|------|------|------|----------|
| 1 | create_gic() | GIC Dist + Redist/CPU | /intc |
| 2 | create_its()/create_v2m() | ITS/V2M | /intc/its / /intc/v2m |
| 3 | create_uart() | PL011 | /pl011@9000000 |
| 4 | create_rtc() | PL031 | /pl031@9010000 |
| 5 | create_pcie() | GPEX PCIe | /pcie@10000000 |
| 6 | create_acpi_ged() | GED | 无 (ACPI only) |
| 7 | create_gpio_devices() | PL061 + GPIO Keys | /pl061@9030000 |
| 8 | create_virtio_devices() | VirtIO MMIO ×32 | /virtio_mmio@a000000 |
| 9 | create_fw_cfg() | fw_cfg | /fw-cfg@9020000 |
| 10 | create_platform_bus() | Platform Bus | (动态) |

---

## 11. PCIe 主桥（GPEX）与 ECAM

### 11.1 GPEX Host Bridge

```c
// virt.c:1912-2043
create_pcie(VirtMachineState *vms) {
    dev = qdev_new(TYPE_GPEX_HOST);  // Generic PCIe Express Host
    sysbus_realize_and_unref(SYS_BUS_DEVICE(dev));
    
    // ECAM (配置空间): 16 MB → 256 bus
    memory_region_init_alias(ecam_alias, "pcie-ecam", ecam_reg, 0, size_ecam);
    memory_region_add_subregion(sysmem, base_ecam, ecam_alias);
    
    // MMIO 窗口: 1:1 映射到系统地址空间
    memory_region_init_alias(mmio_alias, "pcie-mmio", mmio_reg, base, size);
    memory_region_add_subregion(sysmem, base_mmio, mmio_alias);
    
    // High MMIO (64-bit BAR): 512 GiB
    if (highmem_mmio) {
        memory_region_init_alias(high_mmio_alias, "pcie-mmio-high", ...);
    }
    
    // PIO: 64 KB
    sysbus_mmio_map(SYS_BUS_DEVICE(dev), 2, base_pio);
    
    // INTx → GIC SPI 3-6
    for (i = 0; i < 4; i++)
        sysbus_connect_irq(dev, i, gic_spi[irq + i]);
}
```

### 11.2 IOMMU 集成

```c
// virt.c:2027-2042
switch (vms->iommu) {
    case VIRT_IOMMU_SMMUV3:
        create_smmu(vms, vms->bus);  // ARM SMMUv3
        // FDT: iommu-map = <0x0 &smmu 0x0 0x10000>
        break;
    case VIRT_IOMMU_VIRTIO:
        // 在 virt_machine_done() 中处理
        break;
}
```

---

## 12. VirtIO MMIO Transport

```c
// virt.c:1504-1569
create_virtio_devices() {
    // 创建 32 个 virtio-mmio transport
    for (i = 0; i < 32; i++) {
        base = 0x0A000000 + i * 0x200;  // 每个 512B
        irq = SPI 16 + i;
        sysbus_create_simple("virtio-mmio", base, gic_spi[irq]);
    }
    
    // FDT 节点 (逆序创建以获得正确 DTB 排列)
    for (i = 31; i >= 0; i--) {
        fdt_add_subnode("/virtio_mmio@...");
        compatible = "virtio,mmio";
        interrupts = <SPI irq EDGE_LO_HI>;
        dma-coherent;
    }
}
```

---

## 13. FDT 设备树生成

### 13.1 FDT 创建流程

```c
// virt.c:364-445 — create_fdt()
create_fdt(vms) {
    fdt = create_device_tree(&vms->fdt_size);
    
    // Root 属性
    qemu_fdt_setprop_string(fdt, "/", "compatible", "linux,dummy-virt");
    qemu_fdt_setprop_cell(fdt, "/", "#address-cells", 2);
    qemu_fdt_setprop_cell(fdt, "/", "#size-cells", 2);
    qemu_fdt_setprop_string(fdt, "/", "model", "linux,dummy-virt");
    
    // /chosen 节点
    qemu_fdt_add_subnode(fdt, "/chosen");
    
    // /memory 节点
    qemu_fdt_add_subnode(fdt, "/memory");
    qemu_fdt_setprop_string(fdt, "/memory", "device_type", "memory");
    
    // PSCI 节点
    fdt_add_psci_node();
    
    // 时钟源
    "/apb-pclk" — fixed-clock @ 24 MHz
}
```

### 13.2 CPU 节点 (fdt_add_cpu_nodes)

为每个 CPU 生成：
```
/cpus/cpu@N {
    device_type = "cpu";
    compatible = "arm,cortex-a57" / "arm,arm-v8";
    reg = <Aff2 Aff1:Aff0>;
    enable-method = "psci";
    // Cache 拓扑 (L1D/L1I/L2/L3)
}
```

### 13.3 Timer 节点

```
/timer {
    compatible = "arm,armv8-timer" / "arm,armv7-timer";
    interrupts = <
        GIC_PPI 13 IRQ_TYPE_LEVEL_LOW   // Secure EL1
        GIC_PPI 14 IRQ_TYPE_LEVEL_LOW   // NonSecure EL1
        GIC_PPI 11 IRQ_TYPE_LEVEL_LOW   // Virtual
        GIC_PPI 10 IRQ_TYPE_LEVEL_LOW   // Hypervisor
    >;
    always-on;
}
```

---

## 14. ACPI 表生成框架

### 14.1 触发时机

```c
// virt.c:2198 — 在 virt_machine_done() 中调用
virt_acpi_setup(vms);
```

仅当满足以下条件时生成 ACPI：
- `acpi=on` 或 `acpi=auto` 且固件已加载且 aarch64

### 14.2 生成的 ACPI 表

| 表 | 函数 | 内容 |
|---|------|------|
| DSDT | build_dsdt() | CPU/UART/Flash/PCI/GPIO/TPM 设备描述 |
| FADT | build_fadt_rev6() | Fixed ACPI Description, PSCI conduit |
| MADT | build_madt() | GIC 结构体：GICC/GICD/GICR/ITS |
| PPTT | build_pptt() | Processor Topology (cache 层次) |
| GTDT | build_gtdt() | Generic Timer Description |
| MCFG | build_mcfg() | PCIe ECAM 基址 |
| SPCR | build_spcr() | Serial Port Console |
| SRAT | build_srat() | NUMA affinity |
| IORT | build_iort() | I/O Remapping (SMMU/ITS) |
| VIOT | build_viot() | VirtIO IOMMU topology |

### 14.3 MADT 关键结构

```c
// virt-acpi-build.c:992-1097
build_madt() {
    // GIC Distributor Structure (Type=0xC)
    base = VIRT_GIC_DIST.base;  version = gic_version;
    
    // Per-CPU: GIC Structure (Type=0xB, 80字节)
    for each cpu:
        MPIDR, PMU interrupt, GICV/GICH (v2 only)
    
    // GICv3: GICR Structure (redistribuor range)
    // ITS: GIC ITS Structure (Type=0xF)
    // V2M: GIC MSI Frame Structure (Type=0xD)
}
```

### 14.4 FADT ARM Boot Architecture

```c
// virt-acpi-build.c:1111-1124
switch (psci_conduit) {
    case HVC:  arm_boot_arch = PSCI_COMPLIANT | PSCI_USE_HVC;
    case SMC:  arm_boot_arch = PSCI_COMPLIANT;
    case DISABLED: arm_boot_arch = 0;
}
// FADT flags: HW_REDUCED_ACPI (无传统 x86 硬件)
```

---

## 15. Kernel 引导与 CPU Reset

### 15.1 arm_load_kernel()

```c
// boot.c:1173+
arm_load_kernel(cpu, ms, info) {
    if (firmware_loaded)
        arm_setup_firmware_boot(cpu, info);  // 仅设置 reset 回调
    else
        arm_setup_direct_kernel_boot(cpu, info);  // 加载 kernel+initrd+dtb
}
```

### 15.2 Direct Kernel Boot

```c
// boot.c:892-1097
arm_setup_direct_kernel_boot() {
    // 1. 加载 kernel (ELF 或 raw Image)
    kernel_size = load_aarch64_image() 或 arm_load_elf();
    
    // 2. 加载 initrd
    initrd_size = load_ramdisk();
    
    // 3. DTB 放置在 kernel 之后对齐位置
    info->dtb_start = ROUND_UP(kernel_top, align);
    
    // 4. 写入 bootloader stub
    //    AArch64: 设置 x0=dtb_addr, 跳转到 kernel entry
    arm_write_bootloader("bootloader", as, loader_start, bootloader_aarch64, fixup);
}
```

### 15.3 do_cpu_reset() — CPU 复位回调

```c
// boot.c:655-750
do_cpu_reset(ARMCPU *cpu) {
    cpu_reset(cs);
    
    if (is_linux) {
        // 确定目标 EL: 有 EL2 → boot at EL2, 否则 EL1
        target_el = arm_feature(EL2) ? 2 : 1;
        arm_emulate_firmware_reset(cs, target_el);
        
        // Primary CPU: 设置 PC 和内核参数
        cpu_set_pc(cs, loader_start);
    } else {
        // 固件引导: 直接跳转 entry (EL3)
        cpu_set_pc(cs, info->entry);
    }
}
```

---

## 16. PSCI Conduit 决策

```c
// virt.c:2676-2682
if (secure && firmware_loaded) {
    // 固件自己实现 PSCI，QEMU 不参与
    psci_conduit = QEMU_PSCI_CONDUIT_DISABLED;
} else if (vms->virt) {
    // 有 EL2: Guest 用 SMC 调用 PSCI（EL3 处理）
    psci_conduit = QEMU_PSCI_CONDUIT_SMC;
} else {
    // 无 EL2: 用 HVC（兼容性 + KVM 要求）
    psci_conduit = QEMU_PSCI_CONDUIT_HVC;
}
```

PSCI conduit 写入：
- FDT `/psci` 节点的 `method` 属性
- ACPI FADT 的 `arm_boot_arch` 字段

---

## 17. MTE Tag Memory 支持

```c
// virt.c:2790-2837
if (vms->mte) {
    if (tcg_enabled()) {
        // 为 RAM 的每 16 字节分配 4-bit tag → RAM/32 大小
        tag_sysmem = memory_region_init("tag-memory", UINT64_MAX / 32);
        
        // 链接到每个 CPU
        object_property_set_link(cpuobj, "tag-memory", tag_sysmem);
        
        // Secure tag memory (if secure)
        secure_tag_sysmem 覆盖 tag_sysmem
    } else if (kvm_enabled()) {
        kvm_arm_enable_mte(cpuobj);  // KVM 原生 MTE 支持
    }
}

// 实际 tag RAM 分配 (virt.c:2900-2903)
create_tag_ram(tag_sysmem, VIRT_MEM.base, ram_size, "mach-virt.tag");
// 大小 = ram_size / 32 (每 16B 数据对应 4-bit tag)
```

---

## 18. Machine Class 属性与版本兼容

### 18.1 class_init 注册

```c
// virt.c:3820-3870
virt_machine_class_init() {
    mc->init = machvirt_init;
    mc->max_cpus = 512;
    mc->block_default_type = IF_VIRTIO;
    mc->no_cdrom = 1;
    mc->minimum_page_bits = 12;
    mc->default_ram_id = "mach-virt.ram";
    mc->default_nic = "virtio-net-pci";
    
    // 支持的功能
    mc->nvdimm_supported = true;
    mc->smp_props.clusters_supported = true;
    mc->auto_enable_numa_with_memhp = true;
    
    // 用户可配置属性
    property: acpi (OnOffAuto)
    property: secure (bool) — TrustZone
    property: virtualization (bool) — EL2
    property: mte (bool) — MTE
    property: highmem (bool)
    property: gic-version (str)
    property: iommu (str)
    property: ras (bool)
    property: dtb-randomness (bool)
}
```

### 18.2 版本宏

```c
// virt.c:121-127
#define DEFINE_VIRT_MACHINE_AS_LATEST(major, minor, latest)
    static const TypeInfo info = {
        .name = MACHINE_TYPE_NAME("virt-" #major "." #minor),
        .parent = TYPE_VIRT_MACHINE,
        .class_init = virt_##major##_##minor##_class_init,
    };
    mc->desc = "QEMU X.Y ARM Virtual Machine";
```

每个版本的 class_init 通过设置 `VirtMachineClass` 的 `no_xxx` 标志来保持向后兼容。

---

## 19. virt_machine_done() 后处理

在所有设备创建完成后执行（machine_init_done notifier）：

```c
// virt.c:2163-2200
virt_machine_done() {
    // 1. CXL 设备链接
    cxl_hook_up_pxb_registers();
    
    // 2. Platform Bus FDT 节点
    platform_bus_add_all_fdt_nodes(fdt, "/intc", ...);
    
    // 3. 加载 DTB 到 Guest 内存
    arm_load_dtb(info->dtb_start, info, ...);
    
    // 4. ACPI 表生成并放入 fw_cfg
    virt_acpi_setup(vms);
    
    // 5. SMBIOS 表
    virt_build_smbios(vms);
}
```

---

## 20. 与真实硬件平台的对比

| 方面 | virt Machine | 真实 ARM64 SoC (如 Kunpeng 920) |
|------|-------------|-------------------------------|
| 内存布局 | 静态表 + 动态高地址 | 固定 SoC memory map |
| GIC 发现 | QEMU 属性配置 | ACPI MADT / DT |
| PCIe | GPEX 通用桥 | SoC 特定 RC (如 HiSi Hip08) |
| 启动固件 | PFlash UEFI 可选 | SPI NOR / eMMC 固件 |
| IOMMU | SMMUv3 可选 | 集成 SMMU |
| 时钟 | 简化模型 (24 MHz apb-pclk) | 复杂 PLL/分频树 |
| 电源域 | 无建模 | SCPI/SCMI 电源管理 |
| 安全世界 | 选项: secure=on | 硬件 TrustZone + OP-TEE |
| NUMA | 软件属性配置 | 真实互联拓扑 (HCCS) |
| 中断路由 | 简单: 1 SPI/设备 | 复杂: 多 affinity + LPI |

### 关键简化

1. **无时钟树建模**：virt 仅提供一个固定频率 `apb-pclk`，真实 SoC 有几十个时钟域
2. **无电源管理**：没有 DVFS、power domain、idle state 建模
3. **PCIe 简化**：GPEX 是最简 ECAM 实现，无 ARM 特有的 SMCCC PCIe discovery
4. **PSCI 内建**：QEMU 直接在 TCG/KVM 中处理 PSCI，真实平台由 ATF (EL3 固件) 处理
5. **设备发现纯 DT/ACPI**：真实平台可能还有 SMBIOS Type 41、proprietary 方式

---

## 附录 A：关键调用链速查

```
qemu_main()
  → machine_run_board_init()
    → machvirt_init()                      [hw/arm/virt.c:2600]
      → virt_set_memmap()                  [hw/arm/virt.c:2272]
      → finalize_gic_version()             [hw/arm/virt.c:2421]
      → virt_firmware_init()               [hw/arm/virt.c:1685]
      → CPU 创建循环                        [hw/arm/virt.c:2740]
      → create_gic()                       [hw/arm/virt.c:1122]
      → create_pcie()                      [hw/arm/virt.c:1912]
      → create_virtio_devices()            [hw/arm/virt.c:1504]
      → arm_load_kernel()                  [hw/arm/boot.c:1173]
      
  → machine_init_done()
    → virt_machine_done()                  [hw/arm/virt.c:2163]
      → arm_load_dtb()                     [hw/arm/boot.c:467]
      → virt_acpi_setup()                  [hw/arm/virt-acpi-build.c:1469]
        → virt_acpi_build()                [hw/arm/virt-acpi-build.c:1257]
```

## 附录 B：Machine 属性完整列表

| 属性 | 类型 | 默认 | 说明 |
|------|------|------|------|
| acpi | OnOffAuto | auto | ACPI 表生成 |
| secure | bool | false | EL3 TrustZone |
| virtualization | bool | false | EL2 |
| mte | bool | false | Memory Tagging |
| highmem | bool | true | 允许 >4GiB 地址 |
| compact-highmem | bool | true | 紧凑高地址布局 |
| gic-version | str | "max" | GIC 版本 2/3/4/host/max |
| iommu | str | "none" | IOMMU: none/smmuv3/virtio |
| ras | bool | false | RAS 错误注入 |
| dtb-randomness | bool | true | DTB KASLR 种子 |
| highmem-mmio-size | size | 512G | 高地址 PCIe MMIO 大小 |

## 附录 C：源码文件索引

| 文件 | 行数 | 关键函数 |
|------|------|----------|
| hw/arm/virt.c | 4265 | machvirt_init, create_gic, create_pcie |
| hw/arm/virt-acpi-build.c | 1523 | build_madt, build_dsdt, build_iort |
| hw/arm/boot.c | 1310 | arm_load_kernel, do_cpu_reset |
| include/hw/arm/virt.h | ~200 | VirtMachineState, MemMapEntry enum |
