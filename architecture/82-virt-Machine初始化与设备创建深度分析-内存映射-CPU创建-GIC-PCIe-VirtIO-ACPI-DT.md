# virt Machine 初始化与设备创建深度分析

> 文档编号：82  
> 分析目标：QEMU ARM virt 机型的完整初始化流程、内存映射、设备创建顺序  
> 源码版本：QEMU 11.0.50  
> 核心文件：hw/arm/virt.c (4265行)、hw/arm/virt-acpi-build.c (1523行)、include/hw/arm/virt.h (241行)

---

## 一、virt 机型概述

`virt` 是 QEMU ARM/AArch64 的**主力虚拟机型**，不对应任何真实硬件，专为虚拟化设计。
特点：
- 支持 TCG / KVM / HVF 加速
- GICv2 / GICv3 / GICv4 可选
- PCIe / VirtIO-MMIO 双通道
- TrustZone / 虚拟化扩展可选
- ACPI / DT 双启动协议
- 支持最多 512 个 vCPU

---

## 二、物理地址空间布局（Memory Map）

### 2.1 基本内存映射（base_memmap）

```
地址范围                    设备                    大小
─────────────────────────────────────────────────────────
0x0000_0000 - 0x07FF_FFFF   Flash (pflash×2)        128 MB
0x0800_0000 - 0x0801_FFFF   CPU 外设基址            128 KB
  ├─ 0x0800_0000            GIC Distributor          64 KB
  ├─ 0x0801_0000            GIC CPU Interface        64 KB
  ├─ 0x0802_0000            GICv2M                    4 KB
  ├─ 0x0803_0000            GIC Hypervisor            64 KB
  ├─ 0x0804_0000            GIC vCPU                  64 KB
  ├─ 0x0808_0000            GIC ITS                  128 KB
  └─ 0x080A_0000            GIC Redistributor       ~15 MB
0x0900_0000                 UART0 (PL011)             4 KB
0x0901_0000                 RTC (PL031)               4 KB
0x0902_0000                 fw_cfg                    24 B
0x0903_0000                 GPIO (PL061)              4 KB
0x0904_0000                 UART1                     4 KB
0x0905_0000                 SMMUv3                   128 KB
0x0907_0000                 PCDIMM ACPI               4 KB
0x0908_0000                 ACPI GED                  ...
0x090A_0000                 PVTime                    64 KB
0x0A00_0000 - 0x0A00_3FFF   VirtIO MMIO ×32        各 512 B
0x0C00_0000 - 0x0DFF_FFFF   Platform Bus             32 MB
0x0E00_0000 - 0x0EFF_FFFF   Secure Memory            16 MB
0x1000_0000 - 0x3EFE_FFFF   PCIe MMIO               ~750 MB
0x3EFF_0000 - 0x3EFF_FFFF   PCIe PIO                 64 KB
0x3F00_0000 - 0x3FFF_FFFF   PCIe ECAM                16 MB
0x4000_0000 (1 GiB)         RAM 起始               可配置
```

### 2.2 高地址扩展（extended_memmap，浮动在 RAM 之上）

```
动态计算基址               设备                    大小
─────────────────────────────────────────────────────────
(对齐到 64 MB)             High GIC Redist2          64 MB
(对齐到 1 MB)              CXL Host                   1 MB
(对齐到 256 MB)            High PCIe ECAM            256 MB
(对齐到 512 GB)            High PCIe MMIO            512 GB
```

### 2.3 IRQ 分配

```c
static const int a15irqmap[] = {
    [VIRT_UART0]        = 1,     // SPI 1
    [VIRT_RTC]          = 2,     // SPI 2
    [VIRT_PCIE]         = 3,     // SPI 3-6 (4个PCIe INTx)
    [VIRT_GPIO]         = 7,     // SPI 7
    [VIRT_UART1]        = 8,     // SPI 8
    [VIRT_ACPI_GED]     = 9,     // SPI 9
    [VIRT_MMIO]         = 16,    // SPI 16-47 (32个VirtIO)
    [VIRT_GIC_V2M]      = 48,    // SPI 48+ (GICv2m MSI)
    [VIRT_SMMU]         = 74,    // SPI 74+
    [VIRT_PLATFORM_BUS] = 112,   // SPI 112+
};
```

---

## 三、machvirt_init() 初始化主流程

`machvirt_init()` 是 virt 机型的核心入口（hw/arm/virt.c:2600），按以下顺序执行：

### 3.1 阶段一：内存映射与 GIC 版本确定

```
machvirt_init()
│
├─ 1. virt_set_memmap(pa_bits)     — 计算物理地址空间布局
│     ├─ 复制 base_memmap
│     ├─ 计算 device_memory 区域
│     └─ virt_set_high_memmap()   — 放置高地址设备
│
├─ 2. finalize_gic_version()       — 确定 GICv2/v3/v4
│     └─ 根据加速器能力 + 用户配置决定
│
├─ 3. finalize_msi_controller()    — 确定 ITS/v2m
│
└─ 4. 创建 secure_sysmem          — 安全内存视图（如果启用 TrustZone）
```

### 3.2 阶段二：固件与 PSCI 配置

```
├─ 5. virt_firmware_init()         — 加载 pflash/BIOS 固件
│     ├─ pflash_cfi01 映射
│     └─ -bios 或 -drive if=pflash 加载
│
├─ 6. PSCI conduit 选择
│     ├─ secure + firmware → DISABLED
│     ├─ virt (EL2)       → SMC
│     └─ 默认             → HVC
│
└─ 7. virt_max_cpus 计算          — GIC 版本决定 CPU 上限
```

### 3.3 阶段三：CPU 创建（关键步骤）

```
├─ 8. create_fdt()                 — 初始化设备树框架
│
├─ 9. CPU 创建循环 (n = 0..smp_cpus-1)
│     ├─ object_new(cpu_type)      — 创建 CPU 对象
│     ├─ mp-affinity 设置          — MPIDR 编码
│     ├─ has_el3 = vms->secure     — 安全扩展控制
│     ├─ has_el2 = vms->virt       — 虚拟化扩展控制
│     ├─ memory 链接               — 主内存地址空间
│     ├─ secure-memory 链接        — 安全内存地址空间
│     ├─ MTE tag-memory 设置       — MTE 标签内存
│     └─ qdev_realize()            — 实现 CPU（触发 reset）
│
├─ 10. fdt_add_timer_nodes()       — Generic Timer DT 节点
│
└─ 11. fdt_add_cpu_nodes()         — CPU DT 节点 + PSCI enable-method
```

### 3.4 阶段四：设备创建

```
├─ 12. RAM 映射                     — memory_region_add_subregion(VIRT_MEM)
│
├─ 13. create_gic()                 — GIC 控制器
│     ├─ qdev_new(gictype)
│     ├─ 配置 revision/num-cpu/num-irq
│     ├─ Redistributor 区域设置
│     ├─ ITS / v2m 创建
│     ├─ CPU 中断线连接 (IRQ/FIQ/VIRQ/VFIQ)
│     └─ FDT 节点生成
│
├─ 14. fdt_add_pmu_nodes()          — PMU DT 节点
│
├─ 15. create_uart() ×2             — PL011 UART
│     ├─ UART1（如果有第二串口或安全模式）
│     └─ UART0（主控制台，stdout-path）
│
├─ 16. create_secure_ram()          — 安全 RAM（16 MB）
│
├─ 17. create_rtc()                 — PL031 RTC
│
├─ 18. create_pcie()                — PCIe 主桥（GPEX）
│     ├─ ECAM 配置空间映射
│     ├─ MMIO 窗口（低 + 高）
│     ├─ PIO 空间
│     ├─ INTx 中断（SPI 3-6）
│     └─ SMMU 或 virtio-iommu（如果启用）
│
├─ 19. create_acpi_ged() / create_gpio_devices()
│     └─ ACPI 模式：GED 事件设备
│     └─ 非 ACPI：GPIO 按键 + 电源按钮
│
├─ 20. create_virtio_devices()      — 32 个 VirtIO MMIO Transport
│     └─ 每个 512 字节，SPI 16-47
│
├─ 21. create_fw_cfg()              — QEMU fw_cfg 设备
│
├─ 22. create_platform_bus()        — 平台总线（动态 sysbus 设备）
│
├─ 23. NVDIMM 初始化               — 如果启用持久内存
│
└─ 24. arm_load_kernel()            — 加载内核 + PSCI 配置 + DTB
```

### 3.5 阶段五：machine_done 回调

```
virt_machine_done() — 在所有设备创建后触发
│
├─ CXL 初始化完成
├─ platform_bus DT 节点添加
├─ arm_load_dtb()              — 最终 DTB 加载到 RAM
├─ virt_acpi_setup()           — ACPI 表生成
└─ virt_build_smbios()         — SMBIOS 表
```

---

## 四、关键设备创建详解

### 4.1 GIC 创建 (create_gic)

```c
// hw/arm/virt.c:1122
vms->gic = qdev_new(gictype);  // "arm-gic" 或 "arm-gicv3"
qdev_prop_set_uint32(vms->gic, "revision", 2/3/4);
qdev_prop_set_uint32(vms->gic, "num-cpu", smp_cpus);
qdev_prop_set_uint32(vms->gic, "num-irq", NUM_IRQS + 32);

// CPU 中断线连接
for (irq = 0; irq < ARRAY_SIZE(timer_irq); irq++) {
    qdev_connect_gpio_out(cpudev, irq,
        qdev_get_gpio_in(vms->gic, ppibase + timer_irq[irq]));
}
// 连接 IRQ/FIQ/VIRQ/VFIQ 到每个 CPU
sysbus_connect_irq(gicbusdev, i,          // IRQ
    qdev_get_gpio_in(cpudev, ARM_CPU_IRQ));
sysbus_connect_irq(gicbusdev, i + smp_cpus, // FIQ
    qdev_get_gpio_in(cpudev, ARM_CPU_FIQ));
```

### 4.2 UART 创建 (create_uart)

```c
// hw/arm/virt.c:1295
DeviceState *dev = qdev_new(TYPE_PL011);
qdev_prop_set_chr(dev, "chardev", chr);  // 连接字符设备后端
sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
memory_region_add_subregion(mem, base, sysbus_mmio_get_region(...));
sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0,
    qdev_get_gpio_in(vms->gic, irq));
// FDT: /pl011@9000000, compatible="arm,pl011"
// chosen: stdout-path="/pl011@9000000"
```

### 4.3 PCIe 创建 (create_pcie)

```c
// hw/arm/virt.c:1912
dev = qdev_new(TYPE_GPEX_HOST);  // Generic PCIe Host Bridge
// 映射 3 个 MMIO 区域：
// [0] ECAM 配置空间 → 0x3f000000 (16 MB)
// [1] MMIO 窗口     → 0x10000000 (750 MB) + 高地址 (512 GB)
// [2] PIO 窗口      → 0x3eff0000 (64 KB)

// INTx 中断映射：4 个 SPI (3-6)，round-robin 分配给 PCI 设备
// SMMU 可选：create_smmu() 或 create_virtio_iommu()
```

### 4.4 VirtIO MMIO Transport

```c
// hw/arm/virt.c:1537
for (i = 0; i < NUM_VIRTIO_TRANSPORTS; i++) {  // 默认 32 个
    hwaddr base = 0x0a000000 + i * 0x200;      // 每个 512 字节
    int irq = 16 + i;                           // SPI 16-47
    sysbus_create_simple("virtio-mmio", base,
        qdev_get_gpio_in(vms->gic, irq));
}
```

---

## 五、机型属性与配置选项

### 5.1 通过 QOM 属性配置

```c
// virt_machine_class_init()
对象属性                    类型        默认值    说明
──────────────────────────────────────────────────────────
acpi                      OnOffAuto   auto      ACPI 启用
secure                    bool        false     TrustZone
virtualization            bool        false     EL2 虚拟化
highmem                   bool        true      高地址空间
compact-highmem           bool        true      紧凑高地址布局
highmem-redists           bool        true      高地址 GIC redist
highmem-ecam              bool        true      高地址 ECAM
highmem-mmio              bool        true      高地址 PCIe MMIO
highmem-mmio-size         uint64      512 GiB   高地址 MMIO 大小
gic-version               string      "host"    GIC 版本
iommu                     string      ""        IOMMU 类型
its                       bool        true      ITS 启用
mte                       bool        false     MTE 支持
dtb-kaslr-seed            bool        true      KASLR 种子
```

### 5.2 版本兼容性宏

```c
// 每个 QEMU 版本定义特定的机型变体
DEFINE_VIRT_MACHINE(10, 1)  // virt-10.1
DEFINE_VIRT_MACHINE(10, 0)  // virt-10.0
// ... 回溯到 virt-2.6
// 每个版本可覆盖特定 compat 属性
```

---

## 六、固件加载方式

### 6.1 三种固件模式

| 模式 | 命令行 | 加载位置 | EL |
|------|--------|----------|-----|
| pflash | `-drive if=pflash,file=AAVMF_CODE.fd` | 0x0 (Flash) | EL3/EL2 |
| BIOS | `-bios AAVMF_CODE.fd` | 0x0 (Flash MR) | EL3/EL2 |
| 直接内核 | `-kernel Image` | RAM (0x40000000+) | EL2/EL1 |

### 6.2 firmware_loaded 的影响

```c
bool firmware_loaded = virt_firmware_init(vms, ...);

if (firmware_loaded) {
    // PSCI 可能由固件实现 → conduit 可能为 DISABLED
    // ACPI 表不需要 QEMU 生成（固件自带）
    // ECAM 使用高地址布局
} else {
    // QEMU 提供 PSCI
    // QEMU 生成 ACPI/DT
    // 直接内核启动模式
}
```

---

## 七、DT 与 ACPI 双启动协议

### 7.1 设备树生成

每个 `create_*` 函数同时创建硬件设备和对应的 FDT 节点：

```
/                          — 根节点
├── /psci                  — PSCI 接口
├── /cpus/cpu@N            — CPU 节点（enable-method="psci"）
├── /timer                 — Generic Timer
├── /intc                  — GIC 中断控制器
├── /pl011@9000000         — UART0
├── /pl031@9010000         — RTC
├── /fw-cfg@9020000        — fw_cfg
├── /gpio@9030000          — GPIO
├── /virtio_mmio@a000000   — VirtIO MMIO ×32
├── /pcie@10000000         — PCIe 主桥
├── /flash@0               — pflash
├── /memory@40000000       — RAM
└── /chosen                — stdout-path 等
```

### 7.2 ACPI 表生成 (virt-acpi-build.c)

```
virt_acpi_setup()
├─ DSDT  — 设备描述（UART/RTC/GPIO/PCIe/VirtIO）
├─ FADT  — 固件 ACPI 控制数据
├─ MADT  — 多核中断控制器（GIC Distributor/Redistributor/ITS）
├─ GTDT  — Generic Timer 描述
├─ MCFG  — PCIe ECAM 配置
├─ SPCR  — 串口控制台
├─ IORT  — I/O Remapping（SMMU/ITS）
├─ PPTT  — Processor Properties Topology
├─ SRAT  — 静态资源亲和性（NUMA）
├─ SLIT  — 系统局部性信息
├─ VIOT  — VirtIO IOMMU
└─ SMBIOS — 系统管理 BIOS
```

---

## 八、Secure World 配置

当 `-machine secure=on` 时：

```c
// 创建安全内存视图
secure_sysmem = g_new(MemoryRegion, 1);
memory_region_init(secure_sysmem, ..., "secure-memory", UINT64_MAX);
memory_region_add_subregion_overlap(secure_sysmem, 0, sysmem, -1);
// 安全视图包含普通内存（低优先级）+ 安全专用设备（高优先级）

// 安全 RAM: 0x0e000000 - 0x0effffff (16 MB)
create_secure_ram(vms, secure_sysmem, secure_tag_sysmem);

// 安全 UART1: 安全世界专用串口
create_uart(vms, VIRT_UART1, secure_sysmem, serial_hd(1), true);

// 安全 GPIO
create_gpio_devices(vms, VIRT_SECURE_GPIO, secure_sysmem);
```

---

## 九、CPU 创建与特性配置

```c
// hw/arm/virt.c:2740
for (n = 0; n < smp_cpus; n++) {
    cpuobj = object_new(possible_cpus->cpus[n].type);

    // MPIDR 设置
    object_property_set_int(cpuobj, "mp-affinity",
        possible_cpus->cpus[n].arch_id, NULL);

    // 特性控制
    if (!vms->secure) has_el3 = false;  // 禁用 EL3
    if (!vms->virt)   has_el2 = false;  // 禁用 EL2

    // 内存连接
    object_property_set_link(cpuobj, "memory", sysmem);
    if (secure) object_property_set_link(cpuobj, "secure-memory", secure_sysmem);

    // MTE 标签内存
    if (mte && tcg) {
        tag_sysmem = ...;  // 1/32 of address space
        object_property_set_link(cpuobj, "tag-memory", tag_sysmem);
    }

    qdev_realize(DEVICE(cpuobj), NULL, &error_fatal);
}
```

---

## 十、设备创建顺序与依赖关系

```
                    ┌──────────────┐
                    │  memmap 计算  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  GIC 版本确定 │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼────┐      ┌─────▼─────┐     ┌──────▼──────┐
   │ FDT 创建 │      │ CPU 创建  │     │ 固件加载     │
   └────┬────┘      └─────┬─────┘     └──────┬──────┘
        │                  │                  │
        │                  │          ┌───────▼──────┐
        │                  │          │ PSCI 配置     │
        │                  │          └───────┬──────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                    ┌──────▼───────┐
                    │  GIC 创建    │ ← 依赖 CPU 数量
                    └──────┬───────┘
                           │
        ┌──────────┬───────┼───────┬──────────┐
        │          │       │       │          │
   ┌────▼───┐ ┌───▼──┐ ┌──▼──┐ ┌──▼───┐ ┌───▼────┐
   │ UART   │ │ RTC  │ │PCIe │ │GPIO/ │ │VirtIO  │
   │ ×2     │ │      │ │+SMMU│ │ GED  │ │MMIO×32 │
   └────────┘ └──────┘ └─────┘ └──────┘ └────────┘
                           │
                    ┌──────▼───────┐
                    │ fw_cfg/NVDIMM│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ arm_load_kernel│ ← PSCI + DTB 最终化
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ machine_done │ ← ACPI/SMBIOS
                    └──────────────┘
```

---

## 十一、与真实硬件对比

| 方面 | virt 机型 | 真实 ARM 服务器 |
|------|-----------|----------------|
| 内存映射 | 固定/可计算 | SoC 特定 |
| GIC | 可选 v2/v3/v4 | 固定版本 |
| PCIe | 单个 GPEX Host Bridge | 多 Root Complex |
| 启动 | pflash/直接内核 | UEFI from SPI Flash |
| UART | PL011 (SBSA) | 厂商 UART |
| Timer | 标准 Generic Timer | 同 |
| SMMU | 可选 SMMUv3 | 通常固定存在 |
| Flash | CFI (NOR) 模拟 | SPI NOR |

---

## 十二、源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `hw/arm/virt.c` | 4265 | 机型定义、设备创建、FDT 生成 |
| `hw/arm/virt-acpi-build.c` | 1523 | ACPI 表构建 |
| `include/hw/arm/virt.h` | 241 | VirtMachineState/Class 定义 |
| `hw/arm/boot.c` | 1310 | ARM 通用启动加载器 |
| `include/hw/arm/boot.h` | 240 | arm_boot_info 结构 |

### 关键函数索引

| 函数 | 行号 | 职责 |
|------|------|------|
| `machvirt_init()` | 2600 | 初始化主入口 |
| `virt_set_memmap()` | 2272 | 物理地址空间计算 |
| `finalize_gic_version()` | 2421 | GIC 版本决策 |
| `virt_firmware_init()` | 1685 | 固件/pflash 加载 |
| `create_gic()` | 1122 | GIC 创建与连接 |
| `create_uart()` | 1295 | UART 创建 |
| `create_pcie()` | 1912 | PCIe 主桥创建 |
| `create_virtio_devices()` | 1504 | VirtIO MMIO transport |
| `create_rtc()` | 1347 | RTC 创建 |
| `create_fw_cfg()` | 1735 | fw_cfg 设备 |
| `virt_machine_done()` | 2163 | 后期初始化回调 |
| `virt_machine_class_init()` | 3820 | 机型类注册 |
| `fdt_add_cpu_nodes()` | 598 | CPU DT 节点 |
| `fdt_add_timer_nodes()` | 447 | Timer DT 节点 |
| `fdt_add_gic_node()` | 915 | GIC DT 节点 |
| `fdt_add_psci_node()` | 366 | PSCI DT 节点 (boot.c) |

---

## 十三、总结

virt 机型是 QEMU ARM 生态的**核心基础设施**，其初始化流程体现了 QEMU 设备模型的典型模式：

1. **内存映射先行**：所有设备地址在创建前已确定
2. **CPU 创建驱动**：CPU 数量和特性决定 GIC、PSCI 等配置
3. **设备与 DT 同步**：每个 `create_*` 同时创建设备和 FDT 节点
4. **延迟完成**：ACPI/SMBIOS/DTB 加载推迟到 `machine_done`
5. **高度可配置**：通过 QOM 属性支持安全扩展、虚拟化、MTE、IOMMU 等组合
6. **版本兼容**：每个 QEMU 版本维护独立机型变体，确保迁移兼容
