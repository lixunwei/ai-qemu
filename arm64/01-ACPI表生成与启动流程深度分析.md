# ACPI 表生成与启动流程深度分析

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 关键 commit：`46cd2c1050`（ARM64 TPM PPI 支持）、`f168046842`（QLIST_FOREACH 初始化修复）  
> 参考文档：ACPI 6.5 规范、ARM BBR/SBBR 标准、`docs/specs/fw_cfg.rst`  

---

## 目录

- [第一部分：启动架构概览](#第一部分启动架构概览)
  - [1. ARM64 virt 启动模式](#1-arm64-virt-启动模式)
  - [2. 启动内存布局](#2-启动内存布局)
  - [3. 完整启动时间线](#3-完整启动时间线)
- [第二部分：固件加载与 fw_cfg](#第二部分固件加载与-fw_cfg)
  - [4. UEFI 固件加载（pflash）](#4-uefi-固件加载pflash)
  - [5. fw_cfg 固件配置设备](#5-fw_cfg-固件配置设备)
- [第三部分：直接内核启动](#第三部分直接内核启动)
  - [6. 内核加载流程](#6-内核加载流程)
  - [7. DTB 放置与传递](#7-dtb-放置与传递)
- [第四部分：设备树（FDT）生成](#第四部分设备树fdt生成)
  - [8. FDT 创建入口](#8-fdt-创建入口)
  - [9. FDT 节点完整清单](#9-fdt-节点完整清单)
  - [10. GIC 设备树描述](#10-gic-设备树描述)
  - [11. PCIe 设备树描述](#11-pcie-设备树描述)
  - [12. PSCI 电源管理](#12-psci-电源管理)
  - [13. FDT 终结与加载](#13-fdt-终结与加载)
- [第五部分：ACPI 表生成](#第五部分acpi-表生成)
  - [14. ACPI 构建入口与表顺序](#14-acpi-构建入口与表顺序)
  - [15. MADT — GIC 描述表](#15-madt--gic-描述表)
  - [16. GTDT — 通用定时器描述表](#16-gtdt--通用定时器描述表)
  - [17. FADT — 固定 ACPI 描述表](#17-fadt--固定-acpi-描述表)
  - [18. DSDT — AML 设备命名空间](#18-dsdt--aml-设备命名空间)
  - [19. IORT — I/O 重映射表](#19-iort--io-重映射表)
  - [20. 其他 ACPI 表](#20-其他-acpi-表)
  - [21. ACPI 表链接与终结](#21-acpi-表链接与终结)
- [第六部分：AML 构建与热插拔](#第六部分aml-构建与热插拔)
  - [22. AML 构建 API](#22-aml-构建-api)
  - [23. GED 通用事件设备](#23-ged-通用事件设备)
- [第七部分：安全启动与 TrustZone](#第七部分安全启动与-trustzone)
  - [24. TrustZone 安全世界启动](#24-trustzone-安全世界启动)
- [附录](#附录)
  - [A. FDT 与 ACPI 的关系](#a-fdt-与-acpi-的关系)
  - [B. 关键源文件索引](#b-关键源文件索引)
  - [C. 完整启动流程示意图](#c-完整启动流程示意图)

---

# 第一部分：启动架构概览

## 1. ARM64 virt 启动模式

QEMU ARM64 virt 机器支持两种主要启动模式：

```
┌──────────────────────────────────────────────────────┐
│              模式 1: UEFI 启动                        │
│  pflash (CFI01) ─→ EDK2/AAVMF ─→ GRUB ─→ Linux     │
│  固件从 0x0 开始执行, 通过 fw_cfg 获取 ACPI 表        │
├──────────────────────────────────────────────────────┤
│              模式 2: 直接内核启动                      │
│  -kernel vmlinuz -initrd initrd -dtb virt.dtb        │
│  QEMU 内置 bootloader 设置 X0=DTB, PC=kernel_entry   │
└──────────────────────────────────────────────────────┘
```

**模式选择逻辑**（`boot.c:1207-1212`）：
- 有 `-kernel` 且无 firmware → 直接内核启动
- 有 firmware（pflash/bios）→ UEFI 启动
- 有 `-kernel` 且有 firmware → 通过 fw_cfg 将内核传递给固件

## 2. 启动内存布局

基于 `virt.c:161-208` 的 `base_memmap[]`：

```
地址空间布局 (AArch64 virt machine):

0x0000_0000 ┌──────────────────────┐
            │  Flash 0 (安全固件)   │  64 MiB — UEFI 代码
0x0400_0000 ├──────────────────────┤
            │  Flash 1 (变量存储)   │  64 MiB — UEFI NVRAM
0x0800_0000 ├──────────────────────┤
            │  外设空间             │
            │  GIC Dist  @0x0800_0000
            │  GIC Redist @0x080A_0000
            │  UART      @0x0900_0000
            │  RTC       @0x0901_0000
            │  fw_cfg    @0x0902_0000
            │  GPIO      @0x0903_0000
            │  Secure UART @0x0904_0000
            │  MMIO Virtio @0x0A00_0000 (×32)
            │  Platform Bus @0x0C00_0000
0x1000_0000 ├──────────────────────┤
            │  PCIe MMIO           │  256 MiB
0x2000_0000 ├──────────────────────┤
            │  PCIe PIO            │  64 KiB
0x3EFF_0000 ├──────────────────────┤
            │  PCIe ECAM           │  16 MiB
0x4000_0000 ├──────────────────────┤  ← 1 GiB
            │  RAM                 │  (可变大小)
            │  0x4008_0000 内核加载 │  (kernel offset = 0x80000)
            │  ... initrd ...      │
            │  ... DTB ...         │
            └──────────────────────┘

高地址 (>4 GiB, 需要 highmem):
0x80_0000_0000  PCIe ECAM (高位)
0x80_1000_0000  PCIe MMIO (高位, 512 GiB)
```

## 3. 完整启动时间线

从 `qemu_init()` 到第一条客户机指令：

```
qemu_init()
 └─→ machine_run_board_init()
      └─→ machvirt_init()                          [virt.c:2600]
           │
           ├── 1. 计算内存映射与 GIC 版本               [virt.c:2640-2664]
           ├── 2. 创建安全世界内存（若启用）              [virt.c:2649-2664]
           ├── 3. virt_firmware_init()                 [virt.c:2680]
           │       ├── 创建 pflash_cfi01 ×2             [virt.c:1571-1602]
           │       └── 加载 -bios 到 flash0             [virt.c:1685-1733]
           │
           ├── 4. 创建 CPU ×N                          [virt.c:2737-2840]
           │       ├── ARMCPU 实例化与属性设置
           │       ├── CPU topology 配置
           │       └── CPU FDT 节点 (/cpus/cpu@N)       [virt.c:598-873]
           │
           ├── 5. 创建内存 & 设备                       [virt.c:2843-2939]
           │       ├── RAM (VIRT_MEM @ 1GiB)            [virt.c:2850-2851]
           │       ├── create_fdt()                     [virt.c:364-423]
           │       ├── create_gic()                     [virt.c:1122-1293]
           │       ├── create_uart() ×2                 [virt.c:1295-1344]
           │       ├── create_rtc()                     [virt.c:1347-1369]
           │       ├── create_pcie()                    [virt.c:1912-2043]
           │       ├── create_virtio_devices() ×32      [virt.c:1504-1568]
           │       ├── create_ged / create_gpio          [virt.c:2912-2919]
           │       └── fw_cfg_init_mem_dma()            [virt.c:1735-1755]
           │
           ├── 6. 设置 boot_info & arm_load_kernel()    [virt.c:2952-2959]
           │       ├── 选择启动路径（firmware vs direct）
           │       ├── 加载内核/initrd（若适用）
           │       └── 设置 PSCI / 辅助 CPU 启动方式
           │
           └── 7. 注册 machine_done 回调

qemu_machine_creation_done()
 └─→ virt_machine_done()                            [virt.c:2163-2200]
      ├── 添加 platform-bus FDT 节点
      ├── arm_load_dtb() — DTB 写入客户机内存          [boot.c:467-653]
      ├── virt_acpi_setup() — 构建 ACPI 表            [virt-acpi-build.c:1469-1523]
      └── virt_build_smbios(vms) — SMBIOS 数据         [virt.c:2130-2160]

qemu_main_loop() → cpu_exec()
 └─→ do_cpu_reset()                                 [boot.c:655-740]
      ├── 设置 PC = firmware_entry (0x0) 或 kernel_entry
      ├── 设置 X0 = DTB 地址（直接内核启动）
      └── 执行第一条客户机指令！
```

---

# 第二部分：固件加载与 fw_cfg

## 4. UEFI 固件加载（pflash）

### 4.1 pflash 设备创建

ARM virt 使用两个 CFI01 并行闪存设备模拟 UEFI 固件存储（`virt.c:1571-1602`）：

| 设备 | 地址 | 大小 | 用途 |
|------|------|------|------|
| flash0 | 0x0000_0000 | 64 MiB | UEFI 代码（只读） |
| flash1 | 0x0400_0000 | 64 MiB | UEFI 变量存储（可读写） |

```c
// virt.c:1571-1602
static PFlashCFI01 *virt_flash_create1(VirtMachineState *vms, ...) {
    // TYPE_PFLASH_CFI01, sector-length = 256KiB
    // 安全模式下 flash0 标记为 secure=true
}
```

### 4.2 固件加载逻辑

`virt_firmware_init()`（`virt.c:1685-1733`）：

```
if (-bios 指定) {
    将 BIOS 文件加载到 flash0 内存区域
    flash0 设为只读
} else if (pflash drive 0 挂载) {
    直接使用 pflash 块设备
}
// -bios 和 -drive if=pflash,unit=0 不能同时使用
```

### 4.3 Flash 地址映射

```c
// virt.c:1619-1639
// flash0 映射到地址 0x0（安全固件入口点）
// flash1 映射到紧随 flash0 之后
// 安全模式下 flash0 仅安全世界可访问
```

CPU 复位后 PC = 0x0，直接从 flash0 开始执行 UEFI 固件。

## 5. fw_cfg 固件配置设备

### 5.1 设备创建

`fw_cfg_init_mem_dma()`（`virt.c:1735-1755`）在 MMIO 地址 `VIRT_FW_CFG`（0x0902_0000）创建 fw_cfg 设备：

```c
// 设备支持两种访问模式：
// 1. Selector/Data 端口（传统模式）
// 2. DMA 接口（高效批量传输）
```

### 5.2 fw_cfg 暴露的数据

fw_cfg 是 QEMU 向固件传递运行时配置的标准通道：

| 条目路径 | 内容 | 用途 |
|----------|------|------|
| `etc/acpi/tables` | ACPI 表数据 blob | UEFI 解析 ACPI 表 |
| `etc/acpi/rsdp` | RSDP 结构 | ACPI 根表指针 |
| `etc/table-loader` | Bios linker/loader 命令 | 指导固件分配和链接 ACPI 表 |
| `etc/smbios/smbios-tables` | SMBIOS 数据 | 系统管理 BIOS |
| — | kernel/initrd/cmdline | 直接内核启动数据 |

**fw_cfg 条目常量**（`aml-build.h:11-15`）：
```c
#define ACPI_BUILD_TABLE_FILE    "etc/acpi/tables"
#define ACPI_BUILD_RSDP_FILE     "etc/acpi/rsdp"
#define ACPI_BUILD_LOADER_FILE   "etc/table-loader"
```

### 5.3 DMA 接口

fw_cfg DMA 允许固件高效地批量读写数据（`fw_cfg.c:334-466`）：

```
DMA 操作流程:
1. 固件写入 DMA 控制寄存器（地址、长度、方向）
2. QEMU 执行数据传输
3. 固件等待完成标志
```

### 5.4 Bios Linker/Loader

QEMU 使用 bios linker/loader 机制指导固件正确放置和链接 ACPI 表：

```
命令类型:
- ALLOCATE: 在客户机内存中分配空间放置表
- ADD_POINTER: 在表 A 中写入指向表 B 的指针
- ADD_CHECKSUM: 计算并写入校验和
```

---

# 第三部分：直接内核启动

## 6. 内核加载流程

`arm_load_kernel()`（`boot.c:1173-1297`）是直接启动的核心：

```c
// boot.c:1207-1212 — 路径选择
if (info->kernel_filename && !info->firmware_loaded) {
    // 直接内核启动路径
    arm_setup_direct_kernel_boot(info);
} else {
    // 固件路径（UEFI）
    // 内核通过 fw_cfg 传递给固件
}
```

### 6.1 内核格式支持

QEMU 支持多种内核格式（`boot.c:892-1108`）：

| 格式 | 检测方式 | 加载偏移 |
|------|----------|----------|
| ELF | ELF magic `\x7fELF` | 按 ELF 段加载 |
| uImage | U-Boot header magic | 按 header 指定 |
| AArch64 Image | ARM64 kernel header magic | RAM + 0x80000 |
| 原始二进制 | 最后尝试 | RAM + 0x80000（AArch64）或 0x10000（AArch32） |

### 6.2 加载地址

```c
// boot.c:40-43
#define KERNEL_ARGS_ADDR  0x100      // DTB/参数地址偏移
#define KERNEL_LOAD_ADDR  0x00010000 // AArch32 内核偏移
#define KERNEL64_LOAD_ADDR 0x00080000 // AArch64 内核偏移
```

实际加载地址 = `RAM_base + offset`：
- AArch64 内核：`0x4000_0000 + 0x80000 = 0x4008_0000`
- initrd：内核之后，4KiB 对齐
- DTB：initrd 之后，2MiB 对齐

### 6.3 内置 Bootloader

对于 AArch64 直接启动，QEMU 注入一小段 bootloader 代码（`boot.c:69-80`）：

```asm
; AArch64 bootloader stub
MOV   X4, X0                ; 保存内核入口地址
LDR   X0, dtb_addr          ; X0 = DTB 地址（内核约定）
MOV   X1, #0                ; X1 = 0
MOV   X2, #0                ; X2 = 0
MOV   X3, #0                ; X3 = 0
BR    X4                    ; 跳转到内核入口
```

## 7. DTB 放置与传递

### 7.1 DTB 写入客户机内存

`arm_load_dtb()`（`boot.c:467-653`）：

```c
// 1. 获取 DTB 数据（机器生成或用户提供的 -dtb 文件）
// 2. 修改 /chosen 节点：bootargs、initrd 地址
// 3. 添加 /memory 节点（RAM 范围）
// 4. 添加 PSCI 节点
// 5. 写入客户机内存作为 ROM blob
rom_add_blob_fixed("dtb", fdt, fdt_size, addr);  // boot.c:636-645
```

### 7.2 DTB 地址传递

AArch64 Linux 内核启动约定：
- **X0** = DTB 物理地址
- **X1** = 0（保留）
- **X2** = 0（保留）
- **X3** = 0（保留）
- **PC** = 内核入口点

由 QEMU bootloader stub 在 `do_cpu_reset()`（`boot.c:655-740`）中设置。

---

# 第四部分：设备树（FDT）生成

## 8. FDT 创建入口

FDT 在机器初始化阶段创建（`virt.c:364-423`），在 `machvirt_init()` 中调用（`virt.c:2737-2738`）：

```c
// virt.c:364-423
static void create_fdt(VirtMachineState *vms) {
    MachineState *ms = MACHINE(vms);

    ms->fdt = create_device_tree(&vms->fdt_size);

    // 根节点属性
    qemu_fdt_setprop_string(fdt, "/", "compatible", "linux,dummy-virt");
    qemu_fdt_setprop_cell(fdt, "/", "interrupt-parent", phandle);
    qemu_fdt_setprop_cell(fdt, "/", "#address-cells", 2);
    qemu_fdt_setprop_cell(fdt, "/", "#size-cells", 2);

    // 创建基本节点
    qemu_fdt_add_subnode(fdt, "/chosen");
    qemu_fdt_add_subnode(fdt, "/aliases");

    // 安全模式下创建 /secure-chosen
    if (vms->secure) {
        qemu_fdt_add_subnode(fdt, "/secure-chosen");
    }

    // 固定时钟节点 /apb-pclk
    qemu_fdt_add_subnode(fdt, "/apb-pclk");
    qemu_fdt_setprop_string(fdt, "/apb-pclk", "compatible", "fixed-clock");
    qemu_fdt_setprop_cell(fdt, "/apb-pclk", "clock-frequency", 24000000);
}
```

## 9. FDT 节点完整清单

以下是 ARM virt 机器创建的所有 FDT 节点：

### 9.1 核心节点

| 节点路径 | 创建函数 | 位置 | 说明 |
|----------|----------|------|------|
| `/` | `create_fdt()` | `virt.c:377-395` | 根节点，compatible="linux,dummy-virt" |
| `/chosen` | `create_fdt()` | `virt.c:400` | 启动参数 |
| `/aliases` | `create_fdt()` | `virt.c:401` | 设备别名 |
| `/apb-pclk` | `create_fdt()` | `virt.c:407-421` | 24MHz 固定时钟 |
| `/memory@...` | `arm_load_dtb()` | `boot.c:530-579` | RAM 描述 |

### 9.2 CPU 节点

| 节点路径 | 创建位置 | 说明 |
|----------|----------|------|
| `/cpus` | `virt.c:598-630` | CPU 容器，#address-cells=1 |
| `/cpus/cpu@N` | `virt.c:632-780` | 每个 CPU 核心 |
| `/cpus/cpu-map` | `virt.c:830-873` | CPU 拓扑结构 |

CPU 节点属性：
```
/cpus/cpu@0 {
    device_type = "cpu";
    compatible = "arm,cortex-a72" | "arm,cortex-a57" | ...;
    reg = <cpu_id>;
    enable-method = "psci";         // PSCI 启用时
    // AArch32: enable-method = "spin-table"; spin-table-addr = ...
};
```

### 9.3 中断与定时器

| 节点 | 创建函数 | 位置 | compatible |
|------|----------|------|------------|
| `/intc` | `fdt_add_gic_node()` | `virt.c:915-990` | arm,gic-v3 / arm,cortex-a15-gic |
| `/intc/its@...` | `fdt_add_its_gic_node()` | `virt.c:876-894` | arm,gic-v3-its |
| `/intc/v2m@...` | `fdt_add_v2m_gic_node()` | `virt.c:896-913` | arm,gic-v2m-frame |
| `/timer` | （create_timer_fdt） | `virt.c:447-512` | arm,armv8-timer |
| `/pmu` | （create_pmu_fdt） | `virt.c:993-1018` | arm,armv8-pmuv3 |

### 9.4 设备节点

| 节点 | 创建函数 | 位置 | compatible |
|------|----------|------|------------|
| `/pl011@9000000` | `create_uart()` | `virt.c:1295-1344` | arm,pl011 |
| `/pl031@9010000` | `create_rtc()` | `virt.c:1347-1369` | arm,pl031 |
| `/flash@0` | — | `virt.c:1641-1683` | cfi-flash |
| `/fw-cfg@9020000` | — | `virt.c:1735-1754` | qemu,fw-cfg-mmio |
| `/virtio_mmio@a000000..` | `create_virtio_devices()` | `virt.c:1504-1568` | virtio,mmio |
| `/pcie@10000000` | `create_pcie()` | `virt.c:1912-2043` | pci-host-ecam-generic |
| `/gpio-keys` | — | `virt.c:1434-1450` | gpio-keys |

### 9.5 安全世界节点（TrustZone 启用时）

| 节点 | 说明 |
|------|------|
| `/secure-chosen` | 安全世界启动参数 |
| `/secflash@...` | 安全闪存 |
| `/pl011@9040000` | 安全 UART |
| 安全 GPIO | 安全中断控制 |

## 10. GIC 设备树描述

`fdt_add_gic_node()`（`virt.c:915-990`）根据 GIC 版本生成不同描述：

### 10.1 GICv3/v4

```dts
/intc {
    compatible = "arm,gic-v3";
    #interrupt-cells = <3>;
    interrupt-controller;
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;
    reg = <GICD_BASE GICD_SIZE>,    // Distributor
          <GICR_BASE GICR_SIZE>;    // Redistributor(s)
    // 可能有第二个 Redistributor region（大量 CPU 时）
};

/intc/its@... {                      // virt.c:876-894
    compatible = "arm,gic-v3-its";
    msi-controller;
    #msi-cells = <1>;
    reg = <ITS_BASE ITS_SIZE>;
};
```

### 10.2 GICv2

```dts
/intc {
    compatible = "arm,cortex-a15-gic";
    #interrupt-cells = <3>;
    interrupt-controller;
    reg = <GICD_BASE GICD_SIZE>,     // Distributor
          <GICC_BASE GICC_SIZE>,     // CPU Interface
          <GICH_BASE GICH_SIZE>,     // Hypervisor Interface
          <GICV_BASE GICV_SIZE>;     // Virtual CPU Interface
};
```

### 10.3 中断描述格式

`#interrupt-cells = 3` 表示每个中断由 3 个 cell 描述：

```
<type irq_num flags>
  type:  0 = SPI（共享外设中断），1 = PPI（私有外设中断）
  irq_num: 中断号（SPI 从 0 开始，实际 IRQ = irq_num + 32）
  flags: 触发方式（1=上升沿，4=高电平，等）
```

## 11. PCIe 设备树描述

`create_pcie()`（`virt.c:1912-2043`）创建 PCIe 主桥 FDT 节点：

```dts
/pcie@10000000 {
    compatible = "pci-host-ecam-generic";
    device_type = "pci";
    #address-cells = <3>;
    #size-cells = <2>;
    linux,pci-domain = <0>;

    bus-range = <0 0xFF>;                   // virt.c:1995-1996
    reg = <ECAM_BASE ECAM_SIZE>;            // virt.c:2004-2005

    ranges = <                              // virt.c:2007-2022
        0x01000000  PIO_BASE   PIO_SIZE     // I/O 空间
        0x02000000  MMIO_BASE  MMIO_SIZE    // 32位 MMIO
        0x43000000  HIGH_MMIO  HIGH_SIZE    // 64位 prefetchable MMIO
    >;

    #interrupt-cells = <1>;
    interrupt-map-mask = <0x1800 0 0 7>;    // virt.c:1757-1792
    interrupt-map = <                       // create_pcie_irq_map()
        // dev/fn → GIC SPI 映射（轮转分配 4 个 SPI）
    >;

    msi-map = <0 &its_phandle 0 0x10000>;  // virt.c:1999-2002
    iommu-map = <0 &smmu 0 0x10000>;        // virt.c:2027-2041（若有 SMMU）
};
```

**interrupt-map** 通过 `create_pcie_irq_map()`（`virt.c:1757-1792`）生成，将 PCIe INTX 信号映射到 GIC SPI 中断，使用轮转分配 4 个 SPI（SPI 3-6）。

## 12. PSCI 电源管理

### 12.1 PSCI FDT 节点

`fdt_add_psci_node()`（`boot.c:366-442`）：

```dts
/psci {
    compatible = "arm,psci-1.0", "arm,psci-0.2", "arm,psci";
    method = "hvc";        // 或 "smc"，取决于 conduit 配置

    // PSCI 函数 ID（标准定义）
    cpu_suspend = <0xC4000001>;    // boot.c:438-441
    cpu_off     = <0x84000002>;
    cpu_on      = <0xC4000003>;    // AArch64: SMC64 编码
    migrate     = <0xC4000005>;
};
```

### 12.2 CPU 启动方式

PSCI `cpu_on` 是辅助 CPU 的标准启动方式：

```
主 CPU (CPU0):
  → 直接从固件/内核入口启动

辅助 CPU (CPU1..N):
  → 初始状态: halted
  → 主 CPU 通过 PSCI cpu_on(target_cpu, entry_point) 启动
  → QEMU 拦截 PSCI HVC/SMC 调用
  → 设置目标 CPU 的 PC 和参数，唤醒 vCPU 线程
```

### 12.3 PSCI conduit 选择

`machvirt_init()`（`virt.c:2666-2682`）：
- 无安全固件 → PSCI 由 QEMU 模拟（conduit = HVC 或 SMC）
- 有安全固件 → PSCI 由固件（如 TF-A）处理，QEMU 不干预

## 13. FDT 终结与加载

### 13.1 终结流程

`virt_machine_done()`（`virt.c:2163-2200`）在所有设备创建完成后执行：

```c
static void virt_machine_done(Notifier *notifier, void *data) {
    // 1. 添加 platform-bus 动态设备的 FDT 节点
    // 2. 调用 arm_load_dtb() 将 FDT 写入客户机内存
    arm_load_dtb(info->dtb_start, info, info->dtb_limit, ...);

    // 3. 构建 ACPI 表（若启用）
    virt_acpi_setup(vms);

    // 4. 设置 SMBIOS 数据
}
```

### 13.2 DTB 写入

`arm_load_dtb()`（`boot.c:636-645`）：
```c
rom_add_blob_fixed_as("dtb", fdt, fdt_totalsize(fdt), addr, as);
```

DTB 作为 ROM blob 写入客户机物理内存的指定地址，由 bootloader stub 将地址传入 X0 寄存器。

---

# 第五部分：ACPI 表生成

## 14. ACPI 构建入口与表顺序

### 14.1 入口函数

`virt_acpi_setup()`（`virt-acpi-build.c:1469-1523`）：

```c
void virt_acpi_setup(VirtMachineState *vms) {
    // 仅在有 fw_cfg 且 ACPI 启用时构建
    if (!vms->fw_cfg || !virt_is_acpi_enabled(vms)) return;

    // 构建所有 ACPI 表
    virt_acpi_build(&build, vms);

    // 通过 fw_cfg 暴露给固件
    acpi_add_rom_blob(vms->fw_cfg, tables, ACPI_BUILD_TABLE_FILE, 0);
    acpi_add_rom_blob(vms->fw_cfg, rsdp, ACPI_BUILD_RSDP_FILE, 0);
    acpi_add_rom_blob(vms->fw_cfg, linker, ACPI_BUILD_LOADER_FILE, 0);
}
```

### 14.2 表构建顺序

`virt_acpi_build()`（`virt-acpi-build.c:1258-1417`）按以下顺序构建 17 类 ACPI 表：

| 序号 | 表 | 位置 | 说明 |
|------|-----|------|------|
| 1 | **DSDT** | `:1277-1280` | 差异化系统描述表（AML 设备） |
| 2 | **FADT** | `:1281-1284` | 固定 ACPI 描述表（引用 DSDT） |
| 3 | **MADT** | `:1285-1287` | 多 APIC 描述表（GIC 拓扑） |
| 4 | **PPTT** | `:1288-1292` | 处理器拓扑表（可选） |
| 5 | **GTDT** | `:1294-1296` | 通用定时器描述表 |
| 6 | **MCFG** | `:1297-1305` | PCIe ECAM 配置空间 |
| 7 | **SPCR** | `:1307-1311` | 串口控制台重定向 |
| 8 | **DBG2** | `:1313-1314` | 调试设备表 |
| 9 | **HEST** | `:1316-1339` | 硬件错误源表（RAS） |
| 10 | **SRAT** | `:1341-1345` | 系统资源亲和性表（NUMA） |
| 11 | **SLIT** | `:1346-1349` | 系统局部性信息表（NUMA） |
| 12 | **HMAT** | `:1350-1354` | 异构内存属性表（NUMA） |
| 13 | **CEDT** | `:1357-1360` | CXL 早期发现表 |
| 14 | **NVDIMM** | `:1362-1365` | NVDIMM ACPI 表 |
| 15 | **IORT** | `:1368-1369` | I/O 重映射表（SMMU/ITS） |
| 16 | **TPM2** | `:1371-1376` | TPM 2.0 表 |
| 17 | **VIOT** | `:1379-1383` | Virtio IOMMU 拓扑表 |

最后构建：
- **XSDT**（`:1385-1388`）— 扩展系统描述表（指向上述所有表）
- **RSDP**（`:1390-1399`）— 根系统描述指针（指向 XSDT）

## 15. MADT — GIC 描述表

`build_madt()`（`virt-acpi-build.c:991-1096`）为 ARM 平台生成 GIC 相关的 MADT 结构：

```
MADT 结构组成:

┌─────────────────────────────────────┐
│  MADT Header (Type 1)               │
├─────────────────────────────────────┤
│  GICC ×N (每 CPU 一个)               │  GIC CPU Interface 描述
│    - CPU Interface Number            │
│    - MPIDR                           │
│    - Flags (enabled/online-capable)  │
│    - PMU Interrupt                   │
├─────────────────────────────────────┤
│  GICD ×1                             │  GIC Distributor
│    - Physical Base Address           │
│    - GIC Version (3)                 │
├─────────────────────────────────────┤
│  GICR ×1-2                           │  GIC Redistributor
│    - Discovery Range Base/Length     │
│    - 可能有第二个 range（大量 CPU）    │
├─────────────────────────────────────┤
│  GIC ITS ×1 (可选)                    │  MSI 控制器
│    - Physical Base Address           │
│    - GIC ITS ID                      │
├─────────────────────────────────────┤
│  GICv2m ×1 (可选, GICv2 场景)         │  v2 MSI Frame
│    - SPI Base/Count                  │
│    - Frame Physical Base             │
└─────────────────────────────────────┘
```

## 16. GTDT — 通用定时器描述表

`build_gtdt()`（`virt-acpi-build.c:863-916`）描述 ARM 通用定时器的中断配置：

| 定时器 | 中断类型 | 标志 |
|--------|----------|------|
| Secure EL1 Physical Timer | PPI 13 | Level, Active Low |
| Non-secure EL1 Physical Timer | PPI 14 | Level, Active Low |
| Virtual Timer | PPI 11 | Level, Active Low |
| Non-secure EL2 Physical Timer | PPI 10 | Level, Active Low |
| Virtual EL2 Timer（可选） | PPI 12 | Level, Active Low |

## 17. FADT — 固定 ACPI 描述表

`build_fadt_rev6()`（`virt-acpi-build.c:1100-1127`）：

ARM FADT 的关键特性：

```c
// Hardware Reduced ACPI（ARM 必需）
fadt->flags |= ACPI_FADT_F_HW_REDUCED_ACPI;   // 硬件缩减模式
fadt->flags |= ACPI_FADT_F_LOW_POWER_S0;       // 低功耗 S0 idle

// ARM Boot Architecture
fadt->arm_boot_arch = ACPI_FADT_ARM_PSCI_COMPLIANT;    // PSCI 兼容
if (conduit == HVC) {
    fadt->arm_boot_arch |= ACPI_FADT_ARM_PSCI_USE_HVC; // HVC conduit
}

// DSDT 指针（通过 bios-linker 在客户机侧修补）
fadt->x_dsdt = 0;  // 由 linker 填充
```

**Hardware Reduced ACPI**：ARM 平台不使用传统 x86 的 PM1/PM2 寄存器，所有硬件事件通过 GED（Generic Event Device）处理。

## 18. DSDT — AML 设备命名空间

`build_dsdt()`（`virt-acpi-build.c:1130-1226`）构建 ACPI 设备命名空间：

```
DSDT 设备树结构:

\_SB (System Bus)
 ├── CPU0..N          — 处理器设备
 ├── COM0             — PL011 UART
 │    └── _CRS: Memory32Fixed + Interrupt
 ├── FLS0/FLS1        — Flash 设备
 ├── FWCF             — fw_cfg 设备
 ├── VR00..VR31       — Virtio MMIO 设备
 │    └── _CRS: Memory32Fixed + Interrupt
 ├── PCI0             — PCIe 主桥
 │    ├── _CRS: BusRange + MMIO/PIO/ECAM ranges
 │    ├── _PRT: PCI 中断路由表
 │    └── _DSM: PCIe 特定方法
 ├── GED0             — 通用事件设备（热插拔）
 │    ├── _CRS: GED 中断
 │    └── _EVT: 事件分发方法
 ├── PWRB             — 电源按钮
 ├── \_GPE            — 通用目的事件（ACPI GED）
 └── (TPM、Error Device 等可选设备)
```

### 18.1 UART AML 示例

`virt-acpi-build.c:85-100`：

```c
// 构建 PL011 UART 的 ACPI 资源
Aml *crs = aml_resource_template();
aml_append(crs, aml_memory32_fixed(base, size, AML_READ_WRITE));
aml_append(crs, aml_interrupt(AML_CONSUMER, AML_LEVEL, AML_ACTIVE_HIGH,
                              AML_EXCLUSIVE, &irq, 1));

Aml *dev = aml_device("COM0");
aml_append(dev, aml_name_decl("_HID", aml_string("ARMH0011")));
aml_append(dev, aml_name_decl("_CRS", crs));
```

### 18.2 PCIe AML

`acpi_dsdt_add_pci()` / `acpi_dsdt_add_gpex()`（`virt-acpi-build.c:144-188`）：
- 描述 PCIe 配置空间、MMIO/PIO 窗口
- `_PRT` 方法定义中断路由
- 支持热插拔通知

## 19. IORT — I/O 重映射表

`build_iort()`（`virt-acpi-build.c:545-758`）描述 SMMU 和 ITS 的 I/O 地址重映射拓扑：

```
IORT 节点结构:

┌────────────────────┐
│  ITS Group Node    │  MSI 控制器组
│    ITS ID = 0      │
├────────────────────┤
│  Named Component   │  设备组件（如 virtio-mmio）
│    → maps to ITS   │
├────────────────────┤
│  Root Complex      │  PCIe 根复合体
│    → maps to SMMU  │  (或直接到 ITS)
│    → maps to ITS   │
├────────────────────┤
│  SMMUv3 (可选)     │  IOMMU
│    → maps to ITS   │
├────────────────────┤
│  RMR Nodes (可选)  │  保留内存区域（加速 SMMUv3）
└────────────────────┘
```

## 20. 其他 ACPI 表

### 20.1 MCFG（PCIe 配置空间）

```
MCFG 描述 PCIe ECAM 配置空间:
  Base Address = 0x3EFF_0000 (低位) 或 0x80_0000_0000 (高位)
  Start Bus = 0, End Bus = 255
  PCI Segment Group = 0
```

### 20.2 SPCR（串口控制台重定向）

`build_spcr()`（`virt-acpi-build.c:760-797`）：

```
SPCR 描述 PL011 UART 作为系统控制台:
  Interface Type = ARM PL011 UART
  Base Address = 0x0900_0000
  Interrupt Type = GIC SPI
  Interrupt = SPI 1
  Baud Rate = 115200
  Flow Control = None
```

### 20.3 PPTT（处理器拓扑）

当 CPU 拓扑启用时（`!vmc->no_cpu_topology`），PPTT 描述处理器层次：

```
PPTT:
  Package 0
    Cluster 0
      Core 0 (ACPI Processor UID, MPIDR)
      Core 1
    Cluster 1
      Core 2
      Core 3
```

### 20.4 NUMA 相关表

| 表 | 条件 | 内容 |
|-----|------|------|
| SRAT | NUMA 启用 | CPU-to-Node 和 Memory-to-Node 亲和性 |
| SLIT | NUMA 启用 | 节点间距离矩阵 |
| HMAT | NUMA + hmat | 异构内存延迟/带宽属性 |

### 20.5 VIOT（Virtio IOMMU 拓扑）

当使用 virtio-iommu 时，VIOT 表描述 virtio-iommu 设备与受保护设备之间的关系（`virt-acpi-build.c:1379-1383`）。

## 21. ACPI 表链接与终结

### 21.1 表间链接

```
RSDP                            [virt-acpi-build.c:1390-1399]
  └─→ XSDT                     [virt-acpi-build.c:1385-1388]
       ├─→ FADT ──→ DSDT       (FADT 内含 X_DSDT 指针)
       ├─→ MADT
       ├─→ GTDT
       ├─→ MCFG
       ├─→ SPCR
       ├─→ IORT
       ├─→ PPTT
       ├─→ SRAT / SLIT / HMAT
       └─→ ... (其他表)
```

### 21.2 Bios Linker/Loader 机制

ACPI 表在 QEMU 侧构建时使用占位符，通过 bios-linker 命令在客户机侧（由固件执行）完成最终链接：

```c
// aml-build.c:1809-1869 — build_rsdp()
// 1. ALLOCATE: 在客户机 FSEG 内存分配 RSDP 空间
bios_linker_loader_alloc(linker, ACPI_BUILD_RSDP_FILE, ..., true);

// 2. ADD_POINTER: 在 RSDP 中写入 XSDT 地址
bios_linker_loader_add_pointer(linker, ACPI_BUILD_RSDP_FILE,
                               xsdt_tbl_offset, ...);

// 3. ADD_CHECKSUM: 计算并写入校验和
bios_linker_loader_add_checksum(linker, ACPI_BUILD_RSDP_FILE, ...);
```

### 21.3 fw_cfg 发布

构建完成后通过 `acpi_add_rom_blob()`（`virt-acpi-build.c:1490-1513`）将表暴露给固件：

```
fw_cfg entries:
  "etc/acpi/tables"    → 所有 ACPI 表数据（连续 blob）
  "etc/acpi/rsdp"      → RSDP 结构
  "etc/table-loader"   → Bios linker/loader 命令序列
```

UEFI 固件（EDK2）读取 `etc/table-loader`，按命令执行内存分配、指针修补和校验和计算，最终将 ACPI 表安装到客户机内存中。

---

# 第六部分：AML 构建与热插拔

## 22. AML 构建 API

QEMU 使用程序化 API 构建 AML（ACPI Machine Language）字节码，无需 ASL 编译器（`aml-build.c:546-1237`）：

### 22.1 核心构建函数

| 函数 | 用途 | 对应 ASL |
|------|------|----------|
| `aml_scope(name)` | 创建作用域 | `Scope(\_SB) {}` |
| `aml_device(name)` | 创建设备 | `Device(COM0) {}` |
| `aml_method(name, argc, ...)` | 创建方法 | `Method(_STA, 0) {}` |
| `aml_name_decl(name, val)` | 名称声明 | `Name(_HID, "ARMH0011")` |
| `aml_resource_template()` | 资源模板 | `ResourceTemplate() {}` |
| `aml_memory32_fixed(base, len, rw)` | 固定内存资源 | `Memory32Fixed(...)` |
| `aml_interrupt(...)` | 中断资源 | `Interrupt(...)` |
| `aml_operation_region(name, ...)` | 操作区域 | `OperationRegion(...)` |
| `aml_field(region, ...)` | 字段定义 | `Field(REG, ...) {}` |
| `aml_if(predicate)` | 条件语句 | `If(cond) {}` |
| `aml_notify(obj, val)` | 通知 | `Notify(DEV, 0x80)` |
| `aml_store(val, target)` | 赋值 | `Store(val, target)` |
| `aml_return(val)` | 返回 | `Return(val)` |

### 22.2 构建示例：GPIO 电源按钮

`virt-acpi-build.c:190-218`：

```c
// ASL 等价:
// Device(\_SB.GPIO) {
//     Name(_AEI, ResourceTemplate() {
//         GpioInt(Edge, ActiveHigh, ...) { 3 }
//     })
//     Method(_E03) {   // GPIO pin 3 事件处理
//         Notify(\_SB.PWRB, 0x80)  // 通知电源按钮
//     }
// }

Aml *dev = aml_device("GPIO");
Aml *aei = aml_resource_template();
aml_append(aei, aml_gpio_int(..., 3));          // GPIO pin 3
aml_append(dev, aml_name_decl("_AEI", aei));

Aml *method = aml_method("_E03", 0, AML_NOTSERIALIZED);
aml_append(method, aml_notify(aml_name("\\._SB.PWRB"), aml_int(0x80)));
aml_append(dev, method);
```

## 23. GED 通用事件设备

GED（Generic Event Device）是 ARM 平台的标准热插拔事件机制，替代 x86 的 GPE 块。

### 23.1 GED 支持的事件类型

`generic_event_device.c:28-35`：

| 事件 | 位 | 触发场景 |
|------|-----|----------|
| Memory Hot-plug | bit 0 | 内存热添加/移除 |
| Power Button | bit 1 | ACPI 电源按钮 |
| NVDIMM | bit 2 | NVDIMM 热插拔 |
| CPU Hot-plug | bit 3 | vCPU 热添加/移除 |
| PCI Hot-plug | bit 4 | PCIe 设备热插拔 |
| Hardware Error | bit 5 | RAS 错误通知 |

### 23.2 GED AML 结构

`build_ged_aml()`（`generic_event_device.c:47-167`）：

```
Device(\_SB.GED0) {
    Name(_HID, "ACPI0013")            // GED 硬件 ID

    // 操作区域映射到 GED MMIO 寄存器
    OperationRegion(EREG, SystemMemory, GED_BASE, GED_SIZE)
    Field(EREG, DWordAcc, NoLock, WriteAsZeros) {
        ESEL, 32                       // 事件选择器寄存器
    }

    // 中断资源
    Name(_CRS, ResourceTemplate() {
        Interrupt(...) { GED_IRQ }
    })

    // 事件分发方法
    Method(_EVT, 1, Serialized) {
        Local0 = ESEL                  // 读取事件选择器
        If (Local0 & MEM_BIT) {
            // 调用内存热插拔扫描
            Notify(\_SB.MEM0, 0x80)
        }
        If (Local0 & CPU_BIT) {
            // 调用 CPU 热插拔扫描
            \_SB.CPUS.CSCN()
        }
        If (Local0 & PCI_BIT) {
            // 通知 PCI 总线扫描
            Notify(\_SB.PCI0, 0x02)
        }
    }
}
```

### 23.3 事件注入路径

```
Host 侧事件触发:
  acpi_ged_send_event()                [generic_event_device.c:~320+]
    → 写入事件选择器寄存器
    → 触发 GED IRQ（SPI 9）

Guest 侧处理:
  GED IRQ → ACPI _EVT 方法
    → 读取选择器
    → 分发到对应设备的通知
    → OS 执行热插拔响应
```

---

# 第七部分：安全启动与 TrustZone

## 24. TrustZone 安全世界启动

### 24.1 安全模式配置

ARM virt 机器支持 TrustZone 安全扩展（`virt.c:2965-2977`）：

```c
// 默认禁用，需要显式启用
// -machine virt,secure=on
vms->secure = true;  // 启用 EL3 安全世界
```

**限制条件**（`virt.c:2709-2714`）：
- 仅 TCG 和 qtest 加速器支持安全模式
- KVM 不支持（宿主无法模拟 EL3）

### 24.2 安全世界资源

启用安全模式后创建的额外资源（`virt.c:2649-2664`）：

| 资源 | 说明 |
|------|------|
| 安全 RAM | 独立的安全内存区域 |
| 安全 UART | `/pl011@9040000`（仅安全世界可访问） |
| 安全 Flash | flash0 标记为 secure-only |
| 安全 GPIO | 安全中断控制 |
| `/secure-chosen` | 安全世界的 FDT chosen 节点 |

### 24.3 PSCI 与安全固件

当安全固件（如 ARM Trusted Firmware / TF-A）加载时（`virt.c:2666-2682`）：
- QEMU 禁用内置 PSCI 模拟
- PSCI 调用由固件处理（固件运行在 EL3）
- 启动链：BL1 (ROM) → BL2 (Trusted Boot) → BL31 (Runtime) → BL33 (UEFI)

---

# 附录

## A. FDT 与 ACPI 的关系

ARM virt 机器同时支持 FDT 和 ACPI，但用途不同：

| 方面 | FDT | ACPI |
|------|-----|------|
| 始终创建 | ✓ | 仅 UEFI 模式 |
| 热插拔 | 不支持 | GED 支持 |
| 标准 | 嵌入式 Linux 传统 | SBBR/BBR 服务器标准 |
| 复杂度 | 简单 | 完整（含 AML 字节码） |
| NUMA | 不支持 | SRAT/SLIT/HMAT |
| 优先级 | UEFI 模式下 ACPI 优先 | — |

**判断逻辑**（`virt.c:2912-2919`）：
```c
if (aarch64 && firmware_loaded && virt_is_acpi_enabled(vms)) {
    // 使用 ACPI GED 替代 GPIO
    create_acpi_ged(vms);
} else {
    // 使用 GPIO + FDT
    create_gpio_devices(vms);
}
```

即使在 ACPI 模式下，FDT 仍然被创建（用于固件早期启动阶段），但操作系统优先使用 ACPI 表。

## B. 关键源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `hw/arm/virt.c` | 4,265 | ARM virt 机器定义（设备创建、FDT、启动） |
| `hw/arm/virt-acpi-build.c` | 1,523 | ARM virt ACPI 表构建 |
| `hw/arm/boot.c` | 1,310 | ARM 启动加载（内核/DTB/bootloader） |
| `hw/acpi/aml-build.c` | 2,847 | AML 构建 API + RSDP/XSDT 构建 |
| `hw/acpi/generic_event_device.c` | 632 | GED 热插拔事件设备 |
| `hw/nvram/fw_cfg.c` | — | fw_cfg 固件配置设备 |
| `include/hw/acpi/aml-build.h` | — | AML 构建 API 声明 + fw_cfg 常量 |
| `include/hw/arm/virt.h` | — | ARM virt 机器头文件 |

## C. 完整启动流程示意图

```
═══════════════════════════════════════════════════════════════
                    UEFI 启动路径
═══════════════════════════════════════════════════════════════

QEMU 侧:
  1. machvirt_init()
     → 创建 pflash ×2, CPU, GIC, UART, PCIe, virtio...
     → create_fdt() 创建设备树
  2. virt_machine_done()
     → virt_acpi_setup() 构建 ACPI 表 → fw_cfg
     → arm_load_dtb() 写入 DTB

CPU Reset:
  3. PC = 0x0 (flash0 起始地址)
     → 执行 EDK2 UEFI 固件

EDK2 固件:
  4. SEC Phase  — 安全初始化
  5. PEI Phase  — 预 EFI 初始化（内存发现）
  6. DXE Phase  — 驱动执行
     → 读取 fw_cfg "etc/table-loader"
     → 执行 bios-linker 命令分配/链接 ACPI 表
     → 安装 ACPI 表到 EFI Configuration Table
  7. BDS Phase  — 启动设备选择
     → GRUB / Linux 直接启动
  8. OS Loader  — 内核接管
     → 通过 EFI_ACPI_TABLE_GUID 获取 RSDP
     → 解析 ACPI 表发现硬件

═══════════════════════════════════════════════════════════════
                   直接内核启动路径
═══════════════════════════════════════════════════════════════

QEMU 侧:
  1. machvirt_init()
     → 创建设备，create_fdt()
  2. arm_load_kernel()
     → 加载 vmlinuz 到 RAM+0x80000
     → 加载 initrd 到内核之后
     → arm_load_dtb() 写入 DTB
     → 注入 bootloader stub

CPU Reset:
  3. do_cpu_reset()
     → PC = bootloader stub 地址
     → X0 = DTB 物理地址

Bootloader Stub:
  4. X0 = DTB addr
     X1 = X2 = X3 = 0
     BR kernel_entry

Linux Kernel:
  5. 从 X0 获取 DTB 指针
     → 解析设备树发现硬件
     → 初始化 GIC, timer, UART...
     → mount initrd, 启动 init
```

---

> **相关文档**：
> - `darren/architecture/00-全局架构概览.md` — QEMU 全局架构与 virt 机器概览
> - `darren/arm64/00-ARM64-CPU-GICv3-TCG深度分析.md` — ARM64 CPU 模型与 GIC 详解
> - `darren/device-model/00-设备模型与virtio深度分析.md` — 设备模型框架与 virt 设备拓扑
> - `darren/memory/00-内存子系统深度分析.md` — 内存子系统（与 IORT/SMMU 相关）
> - ACPI 6.5 规范 — 表格式定义
> - ARM SBBR（Server Base Boot Requirements）— ARM 服务器启动标准
