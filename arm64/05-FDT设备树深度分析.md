# QEMU FDT（Flattened Device Tree）深度分析

> QEMU 版本：11.0.50  
> 分析范围：libfdt 集成、FDT 封装层、ARM virt 逐设备节点创建、FDT 加载与启动、ACPI 共存、多架构支持  
> 关联文档：`darren/architecture/03-Machine建立流程深度分析.md`、`darren/arm64/01-ACPI表生成与启动流程深度分析.md`

---

## 目录

1. [概述与架构总览](#1-概述与架构总览)
2. [libfdt 集成与构建](#2-libfdt-集成与构建)
3. [QEMU FDT 封装层](#3-qemu-fdt-封装层)
4. [FDT Blob 创建与生命周期](#4-fdt-blob-创建与生命周期)
5. [ARM virt 根节点与全局属性](#5-arm-virt-根节点与全局属性)
6. [时钟节点（apb-pclk）](#6-时钟节点apb-pclk)
7. [CPU 节点](#7-cpu-节点)
8. [CPU 拓扑（cpu-map）](#8-cpu-拓扑cpu-map)
9. [Cache 层次结构](#9-cache-层次结构)
10. [PSCI 节点](#10-psci-节点)
11. [GIC 中断控制器节点](#11-gic-中断控制器节点)
12. [ITS / GICv2m MSI 控制器](#12-its--gicv2m-msi-控制器)
13. [Timer 节点](#13-timer-节点)
14. [UART（PL011）节点](#14-uartpl011节点)
15. [RTC（PL031）节点](#15-rtcpl031节点)
16. [GPIO 与电源按键](#16-gpio-与电源按键)
17. [virtio-mmio 节点](#17-virtio-mmio-节点)
18. [PCIe Host Bridge 节点](#18-pcie-host-bridge-节点)
19. [SMMUv3 / virtio-IOMMU](#19-smmuv3--virtio-iommu)
20. [Flash 节点](#20-flash-节点)
21. [fw_cfg 节点](#21-fw_cfg-节点)
22. [Platform Bus 与动态 SysBus 设备](#22-platform-bus-与动态-sysbus-设备)
23. [Memory 节点与 NUMA 拓扑](#23-memory-节点与-numa-拓扑)
24. [/chosen 节点与引导参数](#24-chosen-节点与引导参数)
25. [NUMA distance-map](#25-numa-distance-map)
26. [Phandle 管理与交叉引用](#26-phandle-管理与交叉引用)
27. [FDT 加载到 Guest 内存](#27-fdt-加载到-guest-内存)
28. [FDT vs ACPI 选择机制](#28-fdt-vs-acpi-选择机制)
29. [命令行选项与调试](#29-命令行选项与调试)
30. [多架构 FDT 支持](#30-多架构-fdt-支持)
31. [virt_machine_done 终态处理](#31-virt_machine_done-终态处理)
32. [FDT 完整节点树](#32-fdt-完整节点树)

---

## 1. 概述与架构总览

FDT（Flattened Device Tree，扁平设备树）是 QEMU 向 Guest 内核描述硬件拓扑的核心机制。在 ARM/RISC-V 等架构中，FDT 取代了 x86 的 BIOS/ACPI 探测方式，由 hypervisor/固件在启动时将硬件布局以二进制 blob（DTB）形式传递给内核。

**QEMU FDT 工作流全景图**：

```
┌──────────────────────────────────────────────────────────────┐
│                    QEMU 启动流程                              │
│                                                              │
│  main() → qemu_init() → machvirt_init()                     │
│                            │                                 │
│                    ┌───────┴───────┐                         │
│                    │ create_fdt()  │ ← 创建空白 FDT blob     │
│                    └───────┬───────┘                         │
│                            │                                 │
│  ┌─────────────────────────┼─────────────────────────┐      │
│  │     逐设备创建 + 写入 FDT 节点                      │      │
│  │                                                    │      │
│  │  fdt_add_cpu_nodes()    → /cpus/cpu@N              │      │
│  │  fdt_add_gic_node()     → /intc@...                │      │
│  │  fdt_add_timer_nodes()  → /timer                   │      │
│  │  create_uart()          → /pl011@...               │      │
│  │  create_rtc()           → /pl031@...               │      │
│  │  create_virtio_devices()→ /virtio_mmio@...         │      │
│  │  create_pcie()          → /pcie@...                │      │
│  │  create_fw_cfg()        → /fw-cfg@...              │      │
│  └────────────────────────────────────────────────────┘      │
│                            │                                 │
│                    ┌───────┴───────┐                         │
│                    │virt_machine_  │ ← machine_done 回调     │
│                    │   done()      │                         │
│                    └───────┬───────┘                         │
│                            │                                 │
│              ┌─────────────┼─────────────┐                  │
│              │             │             │                   │
│    platform_bus    arm_load_dtb()   virt_acpi_setup()        │
│    _add_all_       │                     │                   │
│    fdt_nodes()     │ 修补 /memory        │ ACPI 表生成      │
│                    │ 修补 /chosen        │ (可选)            │
│                    │ 添加 PSCI           │                   │
│                    │                     │                   │
│                    ▼                     │                   │
│           rom_add_blob_fixed_as()        │                   │
│           将 DTB 写入 Guest RAM          │                   │
└──────────────────────────────────────────────────────────────┘

Guest 启动时：
  x0 = DTB 物理地址 → Linux kernel 读取 FDT → 探测设备
```

**核心源文件**：

| 文件 | 行数 | 职责 |
|------|------|------|
| `system/device_tree.c` | 679 | libfdt 封装层、FDT 创建/加载/查询 |
| `include/system/device_tree.h` | 213 | FDT API 声明 |
| `hw/arm/virt.c` | ~4,100 | ARM virt 机器的所有 FDT 节点创建 |
| `hw/arm/boot.c` | 1,310 | ARM 启动加载、DTB 修补与写入 |
| `hw/core/sysbus-fdt.c` | 217 | Platform Bus 动态设备 FDT 生成 |

---

## 2. libfdt 集成与构建

### 2.1 外部依赖

QEMU 使用 **libfdt**（来自 dtc 项目）操作 FDT blob。构建系统首先尝试系统安装的 libfdt，若不满足版本要求则回退到内置 dtc 子项目：

```c
// meson.build:2058-2084
// 要求 libfdt >= 1.5.1（需要 fdt_find_max_phandle 函数）
fdt = dependency('libfdt', version: '>= 1.5.1', ...)
// 回退：subproject('dtc')
```

### 2.2 关键 libfdt API

QEMU 使用的核心 libfdt 函数：

| API | 用途 |
|-----|------|
| `fdt_create()` | 创建空白 FDT blob |
| `fdt_open_into()` | 将 FDT 展开到指定大小的缓冲区 |
| `fdt_add_subnode()` | 添加子节点 |
| `fdt_setprop()` / `fdt_setprop_cell()` / `fdt_setprop_string()` | 设置属性 |
| `fdt_getprop()` | 读取属性 |
| `fdt_get_phandle()` | 获取节点 phandle |
| `fdt_path_offset()` | 通过路径查找节点偏移 |
| `fdt_node_offset_by_compatible()` | 按 compatible 搜索节点 |
| `fdt_nop_node()` | 将节点置为 NOP（逻辑删除） |
| `fdt_check_header()` | 验证 FDT 头部有效性 |

---

## 3. QEMU FDT 封装层

### 3.1 头文件 API

`include/system/device_tree.h` 声明了所有 QEMU FDT 封装函数：

```c
// device_tree.h:17-18 — 创建/加载
void *create_device_tree(int *sizep);
void *load_device_tree(const char *filename_path, int *sizep);

// device_tree.h:65-124 — 属性操作
int qemu_fdt_setprop(void *fdt, const char *node_path,
                     const char *property, const void *val, int size);
int qemu_fdt_setprop_cell(void *fdt, ...);    // 写 32-bit 值
int qemu_fdt_setprop_u64(void *fdt, ...);     // 写 64-bit 值
int qemu_fdt_setprop_string(void *fdt, ...);  // 写字符串
int qemu_fdt_add_subnode(void *fdt, const char *name);

// device_tree.h:120-121 — phandle 管理
uint32_t qemu_fdt_get_phandle(void *fdt, const char *path);
uint32_t qemu_fdt_alloc_phandle(void *fdt);
```

### 3.2 便利宏

```c
// device_tree.h:126-134 — 多 cell 属性设置
#define qemu_fdt_setprop_cells(fdt, node_path, property, ...)
    // 将可变参数转为 big-endian uint32_t 数组后调用 qemu_fdt_setprop()

// device_tree.h:186-193 — 大小感知的 cell 设置
#define qemu_fdt_setprop_sized_cells(fdt, node_path, property, ...)
    // 参数交替为 (cell数, 值) 对，自动处理 1-cell 或 2-cell 地址
```

### 3.3 错误处理策略

封装层采用两种错误处理风格：

```c
// device_tree.c:231-243 — "必须成功"查找
static int findnode_nofail(void *fdt, const char *node_path)
{
    int offset = fdt_path_offset(fdt, node_path);
    if (offset < 0) {
        error_report("... node not found: %s", node_path);
        exit(1);  // 致命错误，直接终止 QEMU
    }
    return offset;
}

// device_tree.c:431-446 — "可失败"读取
const void *qemu_fdt_getprop(..., Error **errp)
{
    // 返回 NULL 并设置 errp，由调用者决定如何处理
}
```

---

## 4. FDT Blob 创建与生命周期

### 4.1 创建

```c
// device_tree.c:36-75
#define FDT_MAX_SIZE  0x100000    // 1 MiB

void *create_device_tree(int *sizep)
{
    *sizep = FDT_MAX_SIZE;
    fdt = g_malloc0(FDT_MAX_SIZE);
    fdt_create(fdt, FDT_MAX_SIZE);       // 创建空白 FDT
    fdt_finish_reservemap(fdt);           // 完成保留内存映射
    fdt_begin_node(fdt, "");              // 根节点开始
    fdt_end_node(fdt);                    // 根节点结束
    fdt_finish(fdt);                      // 完成构建
    fdt_open_into(fdt, fdt, *sizep);      // 展开以便后续添加节点
    return fdt;
}
```

### 4.2 存储位置

FDT blob 指针存储在 `MachineState::fdt` 中：

```c
// include/hw/core/boards.h:398-405
struct MachineState {
    ...
    void *fdt;           // FDT blob 指针
    ...
};
```

### 4.3 生命周期

```
create_device_tree()          → 分配 1MB FDT blob
         │
machvirt_init() 中：
  create_fdt() 调用            → ms->fdt = fdt
         │
  各 create_*() 函数           → 向 fdt 添加设备节点
         │
virt_machine_done() 中：
  platform_bus_add_all_fdt_nodes() → 添加动态 sysbus 设备节点
  arm_load_dtb()               → 修补 memory/chosen/psci
                               → rom_add_blob_fixed_as() 写入 Guest RAM
         │
Guest 启动                     → 从 RAM 地址读取 DTB
```

---

## 5. ARM virt 根节点与全局属性

```c
// virt.c:364-392 — create_fdt()
static void create_fdt(VirtMachineState *vms)
{
    void *fdt = create_device_tree(&vms->fdt_size);
    ms->fdt = fdt;

    // 根节点属性
    qemu_fdt_setprop_string(fdt, "/", "compatible", "linux,dummy-virt");
    qemu_fdt_setprop_cell(fdt, "/", "#address-cells", 0x2);  // 64-bit 地址
    qemu_fdt_setprop_cell(fdt, "/", "#size-cells", 0x2);     // 64-bit 大小
    qemu_fdt_setprop_string(fdt, "/", "model", "linux,dummy-virt");
    qemu_fdt_setprop(fdt, "/", "dma-coherent", NULL, 0);     // 所有 DMA 一致
}
```

**生成的 FDT 根节点**：
```dts
/ {
    compatible = "linux,dummy-virt";
    model = "linux,dummy-virt";
    #address-cells = <0x02>;
    #size-cells = <0x02>;
    dma-coherent;
};
```

`dma-coherent` 放在根节点的原因（`virt.c:383-391` 注释）：
- 避免遗漏标记 DMA 设备
- 避免 Linux 内核对不支持 DMA 的设备产生虚假警告

---

## 6. 时钟节点（apb-pclk）

```c
// virt.c:409-421
vms->clock_phandle = qemu_fdt_alloc_phandle(fdt);
qemu_fdt_add_subnode(fdt, "/apb-pclk");
qemu_fdt_setprop_string(fdt, "/apb-pclk", "compatible", "fixed-clock");
qemu_fdt_setprop_cell(fdt, "/apb-pclk", "#clock-cells", 0x0);
qemu_fdt_setprop_cell(fdt, "/apb-pclk", "clock-frequency", 24000000);
qemu_fdt_setprop_string(fdt, "/apb-pclk", "clock-output-names", "clk24mhz");
qemu_fdt_setprop_cell(fdt, "/apb-pclk", "phandle", vms->clock_phandle);
```

**FDT 输出**：
```dts
apb-pclk {
    compatible = "fixed-clock";
    #clock-cells = <0x00>;
    clock-frequency = <0x016e3600>;    // 24 MHz
    clock-output-names = "clk24mhz";
    phandle = <0x8000>;                // 分配的 phandle
};
```

此时钟节点被 PL011 UART、PL031 RTC 等 PrimeCell 设备通过 phandle 引用。虽然 PL011 binding 文档声称时钟属性"可选"，但 Linux 内核实际会拒绝探测没有时钟属性的 PL011（`virt.c:409-412` 注释）。

---

## 7. CPU 节点

### 7.1 /cpus 容器

```c
// virt.c:598-657 — fdt_add_cpu_nodes()
// 检查是否有 CPU 使用 Aff3 来决定 #address-cells
for (cpu = 0; cpu < smp_cpus; cpu++) {
    if (arm_cpu_mp_affinity(armcpu) & ARM_AFF3_MASK) {
        addr_cells = 2;    // 64-bit MPIDR，需要 2 个 cell
        break;
    }
}
qemu_fdt_add_subnode(ms->fdt, "/cpus");
qemu_fdt_setprop_cell(ms->fdt, "/cpus", "#address-cells", addr_cells);
qemu_fdt_setprop_cell(ms->fdt, "/cpus", "#size-cells", 0x0);
```

### 7.2 每个 CPU 节点

```c
// virt.c:659-695 — 逆序创建以保证 FDT 中升序排列
for (cpu = smp_cpus - 1; cpu >= 0; cpu--) {
    char *nodename = g_strdup_printf("/cpus/cpu@%d", cpu);

    qemu_fdt_setprop_string(ms->fdt, nodename, "device_type", "cpu");
    qemu_fdt_setprop_string(ms->fdt, nodename, "compatible",
                            armcpu->dtb_compatible);
    // enable-method = "psci"（SMP 时）
    if (vms->psci_conduit != QEMU_PSCI_CONDUIT_DISABLED && smp_cpus > 1) {
        qemu_fdt_setprop_string(ms->fdt, nodename, "enable-method", "psci");
    }
    // reg = MPIDR affinity
    if (addr_cells == 2) {
        qemu_fdt_setprop_u64(ms->fdt, nodename, "reg",
                             arm_cpu_mp_affinity(armcpu));
    } else {
        qemu_fdt_setprop_cell(ms->fdt, nodename, "reg",
                              arm_cpu_mp_affinity(armcpu));
    }
    // NUMA 亲和性
    if (has_node_id) {
        qemu_fdt_setprop_cell(ms->fdt, nodename, "numa-node-id", node_id);
    }
    // 拓扑 phandle
    if (!vmc->no_cpu_topology) {
        qemu_fdt_setprop_cell(ms->fdt, nodename, "phandle",
                              qemu_fdt_alloc_phandle(ms->fdt));
    }
}
```

**FDT 输出示例（4 核）**：
```dts
cpus {
    #address-cells = <0x02>;
    #size-cells = <0x00>;

    cpu@0 {
        device_type = "cpu";
        compatible = "arm,cortex-a72";
        reg = <0x00 0x00>;
        enable-method = "psci";
        phandle = <0x01>;
    };
    cpu@1 {
        device_type = "cpu";
        compatible = "arm,cortex-a72";
        reg = <0x00 0x01>;
        enable-method = "psci";
        phandle = <0x02>;
    };
    // ... cpu@2, cpu@3
};
```

---

## 8. CPU 拓扑（cpu-map）

```c
// virt.c:830-874 — 当 no_cpu_topology 未设置时
// 创建 /cpus/cpu-map/socketN/clusterN/coreN/threadN 层次
```

**FDT 输出示例**（2 socket × 2 core × 2 thread）：
```dts
cpus {
    cpu-map {
        socket0 {
            cluster0 {
                core0 {
                    thread0 { cpu = <&cpu0>; };
                    thread1 { cpu = <&cpu1>; };
                };
                core1 {
                    thread0 { cpu = <&cpu2>; };
                    thread1 { cpu = <&cpu3>; };
                };
            };
        };
        socket1 { ... };
    };
};
```

---

## 9. Cache 层次结构

QEMU 11.0 新增了 Cache 层次描述（`virt.c:514-825`，commit `f669de700`）：

```c
// virt.c:697-825 — 添加 L1/L2/L3 cache 节点
// 每个 CPU 节点通过 next-level-cache phandle 链接到上层 cache
// Cache 节点属性：cache-size, cache-line-size, cache-sets, cache-level
```

**FDT 输出示例**：
```dts
cpu@0 {
    ...
    next-level-cache = <&l2_cache_0>;

    l1-icache {
        compatible = "cache";
        cache-level = <1>;
        cache-size = <0x10000>;      // 64KB
        cache-line-size = <64>;
        cache-sets = <256>;
    };
};

l2-cache-0 {
    compatible = "cache";
    cache-level = <2>;
    cache-unified;
    phandle = <&l2_cache_0>;
    next-level-cache = <&l3_cache>;
};
```

---

## 10. PSCI 节点

PSCI（Power State Coordination Interface）节点由 `arm_load_dtb()` 调用 `fdt_add_psci_node()` 创建：

```c
// boot.c:630 — 在 DTB 修补阶段添加
fdt_add_psci_node(fdt, cpu);
```

**FDT 输出**：
```dts
psci {
    compatible = "arm,psci-1.0", "arm,psci-0.2", "arm,psci";
    method = "hvc";            // 或 "smc"，取决于 psci-conduit
    cpu_suspend = <0xc4000001>;
    cpu_off = <0x84000002>;
    cpu_on = <0xc4000003>;
    migrate = <0xc4000005>;
};
```

`method` 的选择：
- KVM 模式：`"hvc"`（hypervisor call）
- TCG 模式：`"hvc"` 或 `"smc"`（取决于 EL3 是否存在）

---

## 11. GIC 中断控制器节点

### 11.1 GICv3

```c
// virt.c:915-961 — fdt_add_gic_node()
vms->gic_phandle = qemu_fdt_alloc_phandle(ms->fdt);
qemu_fdt_setprop_cell(ms->fdt, "/", "interrupt-parent", vms->gic_phandle);

nodename = g_strdup_printf("/intc@%" PRIx64, vms->memmap[VIRT_GIC_DIST].base);
qemu_fdt_setprop_string(ms->fdt, nodename, "compatible", "arm,gic-v3");
qemu_fdt_setprop_cell(ms->fdt, nodename, "#interrupt-cells", 3);
qemu_fdt_setprop(ms->fdt, nodename, "interrupt-controller", NULL, 0);
qemu_fdt_setprop_cell(ms->fdt, nodename, "#redistributor-regions",
                      nb_redist_regions);
// reg = [distributor, redistributor(s)]
```

**FDT 输出**：
```dts
intc@8000000 {
    compatible = "arm,gic-v3";
    #interrupt-cells = <0x03>;
    interrupt-controller;
    #address-cells = <0x02>;
    #size-cells = <0x02>;
    ranges;
    #redistributor-regions = <0x01>;
    reg = <0x00 0x08000000 0x00 0x10000    /* Distributor */
           0x00 0x080a0000 0x00 0xf60000>; /* Redistributor */
    phandle = <0x8001>;

    // 虚拟化维护中断（virt=on 时）
    interrupts = <GIC_PPI ARCH_GIC_MAINT_IRQ LEVEL_HI>;
};
```

### 11.2 GICv2

```c
// virt.c:962-986
qemu_fdt_setprop_string(ms->fdt, nodename, "compatible",
                        "arm,cortex-a15-gic");
// reg = [distributor, cpu-interface]
// virt 模式额外添加 [hyp-interface, vcpu-interface]
```

---

## 12. ITS / GICv2m MSI 控制器

### 12.1 GICv3 ITS

```c
// virt.c:876-893 — fdt_add_its_gic_node()
vms->msi_phandle = qemu_fdt_alloc_phandle(ms->fdt);
nodename = g_strdup_printf("/intc/its@%" PRIx64, vms->memmap[VIRT_GIC_ITS].base);
qemu_fdt_setprop_string(ms->fdt, nodename, "compatible", "arm,gic-v3-its");
qemu_fdt_setprop(ms->fdt, nodename, "msi-controller", NULL, 0);
qemu_fdt_setprop_cell(ms->fdt, nodename, "#msi-cells", 1);
```

**FDT 输出**：
```dts
intc@8000000 {
    its@8080000 {
        compatible = "arm,gic-v3-its";
        msi-controller;
        #msi-cells = <0x01>;
        reg = <0x00 0x08080000 0x00 0x20000>;
        phandle = <0x8002>;
    };
};
```

### 12.2 GICv2m

```c
// virt.c:896-912 — fdt_add_v2m_gic_node()
// 作为 /intc 子节点 /intc/v2m@...
qemu_fdt_setprop_string(ms->fdt, nodename, "compatible",
                        "arm,gic-v2m-frame");
```

---

## 13. Timer 节点

```c
// virt.c:447-511 — fdt_add_timer_nodes()
qemu_fdt_add_subnode(ms->fdt, "/timer");

// ARMv8 兼容字符串
const char compat[] = "arm,armv8-timer\0arm,armv7-timer";

qemu_fdt_setprop(ms->fdt, "/timer", "always-on", NULL, 0);

// 中断列表：Secure EL1 / NS EL1 / Virtual / NS EL2 [/ NS EL2 Virtual]
qemu_fdt_setprop_cells(ms->fdt, "/timer", "interrupts",
    GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_S_EL1_IRQ),  irqflags,
    GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_NS_EL1_IRQ), irqflags,
    GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_VIRT_IRQ),   irqflags,
    GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_NS_EL2_IRQ), irqflags);
```

**FDT 输出**：
```dts
timer {
    compatible = "arm,armv8-timer", "arm,armv7-timer";
    always-on;
    interrupts = <GIC_PPI 13 LEVEL_HI    /* Secure EL1 Physical */
                  GIC_PPI 14 LEVEL_HI    /* Non-secure EL1 Physical */
                  GIC_PPI 11 LEVEL_HI    /* Virtual */
                  GIC_PPI 10 LEVEL_HI>;  /* Non-secure EL2 Physical */
};
```

**中断触发方式的历史问题**（`virt.c:448-466` 注释）：
- 真实硬件：level-triggered
- KVM 4.4 之前：edge-triggered
- virt-2.8 及之前版本保持 edge-triggered 以向后兼容
- 新版本使用正确的 level-triggered

---

## 14. UART（PL011）节点

```c
// virt.c:1295-1344 — create_uart()
nodename = g_strdup_printf("/pl011@%" PRIx64, base);
const char compat[] = "arm,pl011\0arm,primecell";
const char clocknames[] = "uartclk\0apb_pclk";

qemu_fdt_setprop(ms->fdt, nodename, "compatible", compat, sizeof(compat));
qemu_fdt_setprop_sized_cells(ms->fdt, nodename, "reg", 2, base, 2, size);
qemu_fdt_setprop_cells(ms->fdt, nodename, "interrupts",
    GIC_FDT_IRQ_TYPE_SPI, irq, GIC_FDT_IRQ_FLAGS_LEVEL_HI);
qemu_fdt_setprop_cells(ms->fdt, nodename, "clocks",
    vms->clock_phandle, vms->clock_phandle);  // uartclk + apb_pclk
qemu_fdt_setprop(ms->fdt, nodename, "clock-names",
    clocknames, sizeof(clocknames));

// UART0 设置为默认控制台
if (uart == VIRT_UART0) {
    qemu_fdt_setprop_string(ms->fdt, "/chosen", "stdout-path", nodename);
    qemu_fdt_setprop_string(ms->fdt, "/aliases", "serial0", nodename);
}
```

**FDT 输出**：
```dts
pl011@9000000 {
    compatible = "arm,pl011", "arm,primecell";
    reg = <0x00 0x09000000 0x00 0x1000>;
    interrupts = <GIC_SPI 1 LEVEL_HI>;
    clocks = <&apb_pclk &apb_pclk>;
    clock-names = "uartclk", "apb_pclk";
};

chosen {
    stdout-path = "/pl011@9000000";
};

aliases {
    serial0 = "/pl011@9000000";
};
```

**Secure UART**（`virt.c:1335-1342`）：
- 标记 `status = "disabled"`, `secure-status = "okay"`
- 设置 `/secure-chosen/stdout-path`

---

## 15. RTC（PL031）节点

```c
// virt.c:1347-1368 — create_rtc()
const char compat[] = "arm,pl031\0arm,primecell";
qemu_fdt_setprop(ms->fdt, nodename, "compatible", compat, sizeof(compat));
qemu_fdt_setprop_cell(ms->fdt, nodename, "clocks", vms->clock_phandle);
qemu_fdt_setprop_string(ms->fdt, nodename, "clock-names", "apb_pclk");
```

**FDT 输出**：
```dts
pl031@9010000 {
    compatible = "arm,pl031", "arm,primecell";
    reg = <0x00 0x09010000 0x00 0x1000>;
    interrupts = <GIC_SPI 2 LEVEL_HI>;
    clocks = <&apb_pclk>;
    clock-names = "apb_pclk";
};
```

---

## 16. GPIO 与电源按键

```c
// virt.c:1453-1493 — create_gpio_devices()
// /pl061@... 节点
qemu_fdt_setprop_string(ms->fdt, nodename, "compatible",
                        "arm,pl061\0arm,primecell");
qemu_fdt_setprop_cell(ms->fdt, nodename, "#gpio-cells", 2);
qemu_fdt_setprop(ms->fdt, nodename, "gpio-controller", NULL, 0);

// virt.c:1398-1415 — /gpio-keys/poweroff
// 电源按键绑定到 GPIO pin
```

**FDT 输出**：
```dts
pl061@9030000 {
    compatible = "arm,pl061", "arm,primecell";
    reg = <0x00 0x09030000 0x00 0x1000>;
    #gpio-cells = <0x02>;
    gpio-controller;
    interrupts = <GIC_SPI 7 LEVEL_HI>;
    clocks = <&apb_pclk>;
    clock-names = "apb_pclk";
    phandle = <0x8003>;
};

gpio-keys {
    compatible = "gpio-keys";
    poweroff {
        gpios = <&gpio3 3 0>;        // 引用 GPIO phandle
        linux,code = <116>;           // KEY_POWER
        label = "GPIO Key Poweroff";
    };
};
```

---

## 17. virtio-mmio 节点

```c
// virt.c:1504-1568 — create_virtio_devices()
// 创建 32 个 virtio-mmio transport（默认值）
for (i = 0; i < vms->virtio_transports; i++) {
    sysbus_create_simple("virtio-mmio", base + i * size,
                         qdev_get_gpio_in(vms->gic, irq));
}

// 逆序写入 FDT 以保证地址升序排列
for (i = vms->virtio_transports - 1; i >= 0; i--) {
    qemu_fdt_setprop_string(ms->fdt, nodename, "compatible", "virtio,mmio");
    qemu_fdt_setprop_cells(ms->fdt, nodename, "interrupts",
        GIC_FDT_IRQ_TYPE_SPI, irq, GIC_FDT_IRQ_FLAGS_EDGE_LO_HI);
    qemu_fdt_setprop(ms->fdt, nodename, "dma-coherent", NULL, 0);
}
```

**FDT 输出**（32 个中的一个）：
```dts
virtio_mmio@a000000 {
    compatible = "virtio,mmio";
    reg = <0x00 0x0a000000 0x00 0x200>;
    interrupts = <GIC_SPI 16 EDGE_LO_HI>;
    dma-coherent;
};
```

**注意**：virtio-mmio 使用 **edge-triggered** 中断（`GIC_FDT_IRQ_FLAGS_EDGE_LO_HI`），与其他设备的 level-triggered 不同。

---

## 18. PCIe Host Bridge 节点

```c
// virt.c:1912-2042 — create_pcie()
nodename = g_strdup_printf("/pcie@%" PRIx64, base);

qemu_fdt_setprop_string(ms->fdt, nodename, "compatible",
                        "pci-host-ecam-generic");
qemu_fdt_setprop_string(ms->fdt, nodename, "device_type", "pci");
qemu_fdt_setprop_cell(ms->fdt, nodename, "#address-cells", 3);
qemu_fdt_setprop_cell(ms->fdt, nodename, "#size-cells", 2);
qemu_fdt_setprop_cell(ms->fdt, nodename, "linux,pci-domain", 0);
qemu_fdt_setprop_cells(ms->fdt, nodename, "bus-range", 0, nr_pcie_buses - 1);
qemu_fdt_setprop(ms->fdt, nodename, "dma-coherent", NULL, 0);

// MSI 映射（引用 ITS phandle）
if (vms->msi_phandle) {
    qemu_fdt_setprop_cells(ms->fdt, nodename, "msi-map",
                           0, vms->msi_phandle, 0, 0x10000);
}

// ECAM 配置空间
qemu_fdt_setprop_sized_cells(ms->fdt, nodename, "reg",
                             2, base_ecam, 2, size_ecam);

// 地址范围：IO Port + MMIO + 64-bit MMIO
qemu_fdt_setprop_sized_cells(ms->fdt, nodename, "ranges",
    1, FDT_PCI_RANGE_IOPORT,   2, 0,             2, base_pio,  2, size_pio,
    1, FDT_PCI_RANGE_MMIO,     2, base_mmio,     2, base_mmio, 2, size_mmio,
    1, FDT_PCI_RANGE_MMIO_64BIT, 2, base_mmio_hi, 2, base_mmio_hi, 2, size_mmio_hi);

// 中断映射
create_pcie_irq_map(ms, vms->gic_phandle, irq, nodename);
```

### 18.1 PCIe 中断映射

```c
// virt.c:1757-1792 — create_pcie_irq_map()
// 为 4 个 slot × 4 个 pin 创建 interrupt-map
// 使用旋转公式：irq_nr = first_irq + ((pin + slot) % 4)
// interrupt-map-mask 限制到 slot 0-3, pin 0-3
```

**FDT 输出**：
```dts
pcie@10000000 {
    compatible = "pci-host-ecam-generic";
    device_type = "pci";
    #address-cells = <0x03>;
    #size-cells = <0x02>;
    bus-range = <0x00 0xff>;
    linux,pci-domain = <0x00>;
    dma-coherent;
    msi-map = <0x00 &its 0x00 0x10000>;
    reg = <0x00 0x3f000000 0x00 0x1000000>;  /* ECAM */
    ranges = <0x01000000 ...   /* IO Port */
              0x02000000 ...   /* 32-bit MMIO */
              0x03000000 ...>; /* 64-bit MMIO */
    #interrupt-cells = <0x01>;
    interrupt-map = <...>;
    interrupt-map-mask = <0x1800 0x00 0x00 0x07>;
};
```

---

## 19. SMMUv3 / virtio-IOMMU

```c
// virt.c:1794-1841 — create_smmuv3_dt_bindings()
const char compat[] = "arm,smmu-v3";
const char irq_names[] = "eventq\0priq\0cmdq-sync\0gerror";
// 创建 /smmuv3@... 节点
// 设置 iommu-map 到 PCIe 节点

// virt.c:2027-2041 — PCIe 节点添加 iommu-map
if (vms->iommu == VIRT_IOMMU_SMMUV3) {
    qemu_fdt_setprop_cells(ms->fdt, nodename, "iommu-map",
                           0x0, vms->iommu_phandle, 0x0, 0x10000);
}
```

---

## 20. Flash 节点

```c
// virt.c:1641+ — virt_flash_fdt()
// /flash@... 节点
// compatible = "cfi-flash"
// bank-width = <4>
// 双 flash bank 分别用于 UEFI 固件和 UEFI 变量存储
```

**FDT 输出**：
```dts
flash@0 {
    compatible = "cfi-flash";
    reg = <0x00 0x00000000 0x00 0x4000000     /* 64MB bank 0 */
           0x00 0x04000000 0x00 0x4000000>;   /* 64MB bank 1 */
    bank-width = <0x04>;
};
```

---

## 21. fw_cfg 节点

```c
// virt.c:1735-1753 — create_fw_cfg()
qemu_fdt_setprop_string(ms->fdt, nodename, "compatible", "qemu,fw-cfg-mmio");
qemu_fdt_setprop_sized_cells(ms->fdt, nodename, "reg", 2, base, 2, size);
qemu_fdt_setprop(ms->fdt, nodename, "dma-coherent", NULL, 0);
```

**FDT 输出**：
```dts
fw-cfg@9020000 {
    compatible = "qemu,fw-cfg-mmio";
    reg = <0x00 0x09020000 0x00 0x18>;
    dma-coherent;
};
```

---

## 22. Platform Bus 与动态 SysBus 设备

### 22.1 Platform Bus 节点

```c
// sysbus-fdt.c:178-217 — platform_bus_add_all_fdt_nodes()
node = g_strdup_printf("/platform-bus@%"PRIx64, addr);
const char platcomp[] = "qemu,platform\0simple-bus";

qemu_fdt_setprop(fdt, node, "compatible", platcomp, sizeof(platcomp));
qemu_fdt_setprop_cells(fdt, node, "#size-cells", 1);
qemu_fdt_setprop_cells(fdt, node, "#address-cells", 1);
qemu_fdt_setprop_cells(fdt, node, "ranges", 0, addr >> 32, addr, bus_size);
qemu_fdt_setprop_phandle(fdt, node, "interrupt-parent", intc);

// 遍历所有动态 sysbus 设备
foreach_dynamic_sysbus_device(add_fdt_node, &data);
```

### 22.2 支持的动态设备绑定

```c
// sysbus-fdt.c:134-143
static const BindingEntry bindings[] = {
    TYPE_BINDING(TYPE_TPM_TIS_SYSBUS,  add_tpm_tis_fdt_node),
    TYPE_BINDING(TYPE_ARM_SMMUV3,      no_fdt_node),         // 无通用 FDT
    TYPE_BINDING(TYPE_RAMFB_DEVICE,    no_fdt_node),         // 无 FDT
    TYPE_BINDING(TYPE_UEFI_VARS_SYSBUS, add_uefi_vars_node), // UEFI 变量服务
};
```

**注意**：当前不支持 VFIO 设备的动态 FDT 生成。不在绑定表中的设备类型会导致 QEMU 报错退出。

---

## 23. Memory 节点与 NUMA 拓扑

Memory 节点**不在** `create_fdt()` 中创建，而是在 `arm_load_dtb()` 阶段动态修补：

```c
// boot.c:530-579
// 1. 先 nop 掉现有的 /memory* 节点
node_path = qemu_fdt_node_unit_path(fdt, "memory", &err);
while (node_path[n]) {
    qemu_fdt_nop_node(fdt, node_path[n]);  // 逻辑删除
}

// 2. 按 NUMA 拓扑重建
if (num_nodes > 0) {
    for (i = 0; i < num_nodes; i++) {
        mem_len = nodes[i].node_mem;
        if (!mem_len) continue;              // 跳过空 NUMA 节点
        fdt_add_memory_node(fdt, acells, mem_base, scells, mem_len, i);
        mem_base += mem_len;
    }
} else {
    // 单节点：一个 /memory@... 覆盖全部 RAM
    fdt_add_memory_node(fdt, acells, loader_start, scells, ram_size, -1);
}

// 3. NVDIMM 支持
for (m = md_list; m; m = m->next) {
    if (mi->type == MEMORY_DEVICE_INFO_KIND_NVDIMM) {
        fdt_add_pmem_node(fdt, acells, scells, di->addr, di->size, di->node);
    }
}
```

**FDT 输出**（NUMA）：
```dts
memory@40000000 {
    device_type = "memory";
    reg = <0x00 0x40000000 0x00 0x40000000>;  /* 1 GiB */
    numa-node-id = <0x00>;
};

memory@80000000 {
    device_type = "memory";
    reg = <0x00 0x80000000 0x00 0x40000000>;  /* 1 GiB */
    numa-node-id = <0x01>;
};
```

**空 NUMA 节点处理**（`boot.c:544-552` 注释）：Linux DT NUMA binding 要求不生成空 NUMA 节点的 memory 节点。Linux 会从 distance-map 获取空节点 ID。

---

## 24. /chosen 节点与引导参数

```c
// boot.c:598-628
// 在 arm_load_dtb() 阶段修补 /chosen
rc = fdt_path_offset(fdt, "/chosen");
if (rc < 0) {
    qemu_fdt_add_subnode(fdt, "/chosen");  // 确保存在
}

// bootargs（-append 参数）
if (ms->kernel_cmdline && *ms->kernel_cmdline) {
    qemu_fdt_setprop_string(fdt, "/chosen", "bootargs", ms->kernel_cmdline);
}

// initrd 地址范围
if (binfo->initrd_size) {
    qemu_fdt_setprop_sized_cells(fdt, "/chosen", "linux,initrd-start",
                                 acells, binfo->initrd_start);
    qemu_fdt_setprop_sized_cells(fdt, "/chosen", "linux,initrd-end",
                                 acells, binfo->initrd_start + binfo->initrd_size);
}
```

**FDT 输出**：
```dts
chosen {
    stdout-path = "/pl011@9000000";       // create_uart() 设置
    bootargs = "console=ttyAMA0 root=/dev/vda";  // -append 参数
    linux,initrd-start = <0x00 0x48000000>;
    linux,initrd-end = <0x00 0x49000000>;
    rng-seed = <...>;                     // 随机种子
};
```

---

## 25. NUMA distance-map

```c
// virt.c:423-444 — 在 create_fdt() 中创建
if (nb_numa_nodes > 0 && ms->numa_state->have_numa_distance) {
    // 构建 N×N×3 的距离矩阵
    // 格式：[src_node, dst_node, distance] × N²
    uint32_t *matrix = g_malloc0(size);
    for (i = 0; i < nb_numa_nodes; i++) {
        for (j = 0; j < nb_numa_nodes; j++) {
            matrix[idx + 0] = cpu_to_be32(i);
            matrix[idx + 1] = cpu_to_be32(j);
            matrix[idx + 2] = cpu_to_be32(distance[j]);
        }
    }
    qemu_fdt_add_subnode(fdt, "/distance-map");
    qemu_fdt_setprop_string(fdt, "/distance-map", "compatible",
                           "numa-distance-map-v1");
    qemu_fdt_setprop(fdt, "/distance-map", "distance-matrix", matrix, size);
}
```

**FDT 输出**：
```dts
distance-map {
    compatible = "numa-distance-map-v1";
    distance-matrix = <0 0 10    /* node0→node0 = 10 */
                       0 1 20    /* node0→node1 = 20 */
                       1 0 20    /* node1→node0 = 20 */
                       1 1 10>;  /* node1→node1 = 10 */
};
```

---

## 26. Phandle 管理与交叉引用

### 26.1 分配机制

```c
// device_tree.c — qemu_fdt_alloc_phandle()
// 调用 fdt_find_max_phandle() 找到当前最大 phandle
// 返回 max_phandle + 1
```

### 26.2 交叉引用关系

```
/ interrupt-parent ──────────────────► GIC phandle
                                        │
/cpus/cpu@N next-level-cache ──────► L2 cache phandle
                                        │
/pl011@... clocks ─────────────────► apb-pclk phandle (×2)
                                        │
/pl031@... clocks ─────────────────► apb-pclk phandle
                                        │
/gpio-keys/poweroff gpios ─────────► GPIO phandle
                                        │
/pcie@... msi-map ─────────────────► ITS phandle
                                        │
/pcie@... interrupt-map ───────────► GIC phandle
                                        │
/pcie@... iommu-map ───────────────► SMMU phandle
```

---

## 27. FDT 加载到 Guest 内存

### 27.1 加载流程

```c
// boot.c:467-653 — arm_load_dtb()
int arm_load_dtb(hwaddr addr, const struct arm_boot_info *binfo,
                 hwaddr addr_limit, AddressSpace *as, MachineState *ms,
                 ARMCPU *cpu)
{
    // 1. 获取 DTB 源
    if (binfo->dtb_filename) {
        fdt = load_device_tree(filename, &size);    // 用户提供的 DTB
    } else {
        fdt = binfo->get_dtb(binfo, &size);         // 板级生成的 DTB
    }

    // 2. 大小检查
    if (addr_limit > addr && size > (addr_limit - addr)) {
        return 0;  // DTB 太大，放不下
    }

    // 3. 验证 #address-cells / #size-cells
    acells = qemu_fdt_getprop_cell(fdt, "/", "#address-cells", ...);
    if (scells < 2 && binfo->ram_size >= 4 * GiB) {
        // 用户 DTB 不兼容大于 4GB 的 RAM
    }

    // 4. 修补 memory / chosen / PSCI / initrd
    // ... (见前文各节)

    // 5. 写入 Guest RAM 作为 ROM
    rom_add_blob_fixed_as("dtb", fdt, size, addr, as);

    // 6. 注册 reset 时重新随机化 rng-seed
    qemu_register_reset_nosnapshotload(qemu_fdt_randomize_seeds,
                                       rom_ptr_for_as(as, addr, size));
}
```

### 27.2 DTB 回调

```c
// virt.c:2119-2128
static void *machvirt_dtb(const struct arm_boot_info *binfo, int *fdt_size)
{
    *fdt_size = board->fdt_size;
    return ms->fdt;    // 返回 machvirt_init() 期间构建的 FDT
}
```

### 27.3 Guest 如何找到 DTB

ARM64 Linux 内核启动协议要求：
- **x0** 寄存器 = DTB 物理地址
- DTB 必须 8 字节对齐
- DTB 必须在 2MB 自然对齐的 512MB 范围内

QEMU 通过 `arm_setup_direct_kernel_boot()` 计算 DTB 放置地址，然后在 CPU reset 处理器中设置 x0。

---

## 28. FDT vs ACPI 选择机制

```c
// virt.c:3262-3268
static bool virt_is_acpi_enabled(VirtMachineState *vms)
{
    if (vms->acpi == ON_OFF_AUTO_OFF) {
        return false;     // 用户显式禁用 ACPI
    }
    return true;          // 默认启用 ACPI
}
```

### 28.1 共存行为

FDT 和 ACPI **可以同时存在**：
1. FDT **始终被构建和加载**（作为基础硬件描述）
2. ACPI 表**可选地生成**，通过 fw_cfg 传递给固件
3. 当 ACPI 启用时，部分 FDT 节点被跳过

受 ACPI 影响的 FDT 生成：

```c
// virt.c:732,772 — 某些设备在 ACPI 模式下跳过 FDT 节点
if (!virt_is_acpi_enabled(vms)) {
    // 生成 FDT 节点（仅 FDT-only 模式）
}
```

### 28.2 典型配置

| 场景 | FDT | ACPI | 固件 |
|------|-----|------|------|
| 直接内核启动 | ✓ 主要 | ✗ | 无 |
| UEFI 启动（默认） | ✓ 基础 | ✓ 主要 | EDK2 |
| -machine acpi=off | ✓ 主要 | ✗ | EDK2 回退到 FDT |

---

## 29. 命令行选项与调试

### 29.1 DTB 相关选项

| 选项 | 说明 | 实现 |
|------|------|------|
| `-machine dtb=file.dtb` | 使用用户提供的 DTB | `MachineState` 属性 `dtb`（`machine.c:1087`） |
| `-machine dumpdtb=out.dtb` | 导出生成的 DTB 到文件 | `handle_machine_dumpdtb()`（`machine.c:1703`） |
| `-machine acpi=off` | 禁用 ACPI，使用纯 FDT | `vms->acpi = ON_OFF_AUTO_OFF` |

### 29.2 使用外部 DTB

```c
// boot.c:480-501
if (binfo->dtb_filename) {
    fdt = load_device_tree(filename, &size);  // 加载用户 DTB
    // QEMU 仍会修补 memory、chosen、PSCI 等
}
```

**当用户提供 DTB 时**：
- 跳过 platform_bus 动态 FDT 节点生成（`virt.c:2185-2190`）
- 仍然执行 memory/chosen/PSCI 修补
- 用户负责确保 DTB 中的地址/中断与 QEMU 地址映射匹配

### 29.3 DTB 验证

```c
// device_tree.c:117-123 — 加载时验证
fdt_check_header(fdt);  // 检查 magic、版本

// boot.c:512-528 — 内容验证
if (acells == 0 || scells == 0) {
    // 无效的 #address-cells 或 #size-cells
}
if (scells < 2 && ram_size >= 4GB) {
    // DTB 不兼容大 RAM
}
```

### 29.4 调试技巧

```bash
# 导出 QEMU 生成的 DTB
qemu-system-aarch64 ... -machine virt,dumpdtb=virt.dtb

# 反编译查看
dtc -I dtb -O dts virt.dtb > virt.dts

# HMP 控制台中导出
(qemu) dumpdtb virt.dtb
```

---

## 30. 多架构 FDT 支持

### 30.1 RISC-V

RISC-V virt 机器同样使用 FDT，但有不同：

```
hw/riscv/virt.c     — RISC-V virt 机器 FDT 创建
hw/riscv/numa.c     — RISC-V NUMA FDT 支持
```

| 差异点 | ARM | RISC-V |
|--------|-----|--------|
| 中断控制器 | GICv3/v2 | PLIC/CLINT |
| 时钟 | ARM Generic Timer | SiFive CLINT |
| CPU 节点 | MPIDR affinity | hart ID |
| NUMA | 按内存区域 | 按 socket |
| 启动协议 | x0=DTB | a1=DTB（OpenSBI） |

### 30.2 PPC

PPC 使用预构建的 DTB 文件，QEMU 加载并修补：
- `hw/ppc/spapr.c`（PAPR 平台）自建 FDT
- `hw/ppc/pegasos.c` 等加载外部 DTB

### 30.3 其他架构

- **MicroBlaze**：接受 `dtb_filename`，支持回退到默认 DTB
- **OpenRISC**：保存 `ms->fdt` 用于 dumpdtb

---

## 31. virt_machine_done 终态处理

`virt_machine_done()` 是 FDT 生命周期的最终阶段，完成所有延迟操作：

```c
// virt.c:2162-2199
static void virt_machine_done(Notifier *notifier, void *data)
{
    // 1. CXL 设备连接
    cxl_hook_up_pxb_registers(...);

    // 2. Platform Bus 动态设备 FDT（仅非用户 DTB）
    if (info->dtb_filename == NULL) {
        platform_bus_add_all_fdt_nodes(ms->fdt, "/intc",
            vms->memmap[VIRT_PLATFORM_BUS].base,
            vms->memmap[VIRT_PLATFORM_BUS].size,
            vms->irqmap[VIRT_PLATFORM_BUS]);
    }

    // 3. DTB 修补并写入 Guest RAM
    if (arm_load_dtb(info->dtb_start, info, ...) < 0) {
        exit(1);
    }

    // 4. ACPI 表生成（可选）
    virt_acpi_setup(vms);

    // 5. SMBIOS 表生成
    virt_build_smbios(vms);
}
```

**执行顺序图**：
```
machvirt_init()
    │
    ├── create_fdt()              ← 创建空白 FDT + 根属性
    ├── fdt_add_cpu_nodes()       ← CPU 节点
    ├── fdt_add_gic_node()        ← GIC 节点
    ├── fdt_add_timer_nodes()     ← Timer 节点
    ├── create_uart()             ← UART 节点
    ├── create_rtc()              ← RTC 节点
    ├── create_gpio_devices()     ← GPIO 节点
    ├── create_virtio_devices()   ← virtio-mmio 节点
    ├── create_pcie()             ← PCIe 节点
    ├── create_fw_cfg()           ← fw_cfg 节点
    ├── ... 其他设备 ...
    │
    └── 注册 machine_done 回调
            │
            └── virt_machine_done()
                    │
                    ├── platform_bus_add_all_fdt_nodes()  ← 动态设备
                    ├── arm_load_dtb()                    ← 修补 + 写入 RAM
                    │       ├── nop 旧 /memory 节点
                    │       ├── 创建新 /memory@... 节点
                    │       ├── 修补 /chosen (bootargs, initrd)
                    │       ├── 添加 /psci 节点
                    │       └── rom_add_blob_fixed_as()   ← 写入 Guest RAM
                    │
                    ├── virt_acpi_setup()                  ← ACPI（可选）
                    └── virt_build_smbios()                ← SMBIOS
```

---

## 32. FDT 完整节点树

ARM virt 机器生成的完整 FDT 节点树（按地址排序）：

```
/
├── model = "linux,dummy-virt"
├── compatible = "linux,dummy-virt"
├── #address-cells = <2>
├── #size-cells = <2>
├── dma-coherent
├── interrupt-parent = <&gic>
│
├── chosen/
│   ├── stdout-path = "/pl011@9000000"
│   ├── bootargs = "..."
│   ├── linux,initrd-start = <...>
│   ├── linux,initrd-end = <...>
│   └── rng-seed = <...>
│
├── aliases/
│   ├── serial0 = "/pl011@9000000"
│   └── serial1 = "/pl011@9040000"
│
├── apb-pclk/                              fixed-clock, 24MHz
│
├── psci/                                   PSCI 接口
│
├── cpus/
│   ├── #address-cells = <2>
│   ├── cpu@0/                             cortex-a72, MPIDR=0
│   ├── cpu@1/                             cortex-a72, MPIDR=1
│   ├── ...
│   └── cpu-map/                           拓扑：socket/cluster/core/thread
│
├── distance-map/                          NUMA 距离矩阵（可选）
│
├── memory@40000000/                       RAM（由 arm_load_dtb 创建）
│
├── flash@0/                               CFI Flash
│
├── intc@8000000/                          GICv3 Distributor
│   ├── its@8080000/                       ITS (MSI controller)
│   └── (v2m@... for GICv2)
│
├── timer/                                  ARM Generic Timer
│
├── pl011@9000000/                         UART0 (控制台)
├── pl011@9040000/                         UART1 (secure, 可选)
│
├── pl031@9010000/                         RTC
│
├── fw-cfg@9020000/                        fw_cfg
│
├── pl061@9030000/                         GPIO
│
├── gpio-keys/                             电源按键
│   └── poweroff/
│
├── virtio_mmio@a000000/                   virtio #0
├── virtio_mmio@a000200/                   virtio #1
├── ...                                     (共 32 个)
├── virtio_mmio@a003e00/                   virtio #31
│
├── pcie@10000000/                         PCIe Host Bridge
│   └── (interrupt-map, ranges, msi-map)
│
├── smmuv3@.../ (可选)                     SMMUv3 IOMMU
│
└── platform-bus@c000000/                  动态 sysbus 设备
    ├── tpm_tis@.../ (可选)
    └── uefi-vars@.../ (可选)
```

---

## 附录 A：关键源文件索引

| 文件 | 行号 | 函数/结构 | 说明 |
|------|------|----------|------|
| `device_tree.c` | 36 | `FDT_MAX_SIZE` | 1 MiB 最大 FDT 大小 |
| `device_tree.c` | 38-75 | `create_device_tree()` | 创建空白 FDT blob |
| `device_tree.c` | 78-124 | `load_device_tree()` | 从文件加载 DTB |
| `device_tree.c` | 231-243 | `findnode_nofail()` | 致命查找 |
| `device_tree.c` | 354-404 | `qemu_fdt_setprop*()` | 属性设置封装 |
| `device_tree.c` | 431-446 | `qemu_fdt_getprop()` | 属性读取 |
| `device_tree.c` | 469-481 | `qemu_fdt_get_phandle()` | 获取 phandle |
| `device_tree.c` | 528-556 | `qemu_fdt_add_subnode()` | 添加子节点 |
| `device_tree.h` | 17-18 | 创建/加载声明 | API 入口 |
| `device_tree.h` | 65-124 | 封装函数声明 | setprop/getprop/phandle |
| `device_tree.h` | 126-193 | 便利宏 | cells/sized_cells |
| `virt.c` | 364-445 | `create_fdt()` | 根节点、clock、distance-map |
| `virt.c` | 447-511 | `fdt_add_timer_nodes()` | Timer 节点 |
| `virt.c` | 514-825 | `add_cache_node()`+ 辅助 | Cache 层次 |
| `virt.c` | 598-874 | `fdt_add_cpu_nodes()` | CPU/拓扑 |
| `virt.c` | 876-913 | `fdt_add_its_gic_node()`/`v2m` | ITS/v2m 节点 |
| `virt.c` | 915-990 | `fdt_add_gic_node()` | GIC 节点 |
| `virt.c` | 1295-1344 | `create_uart()` | UART (PL011) |
| `virt.c` | 1347-1368 | `create_rtc()` | RTC (PL031) |
| `virt.c` | 1398-1493 | `create_gpio_devices()` | GPIO/电源按键 |
| `virt.c` | 1504-1568 | `create_virtio_devices()` | virtio-mmio ×32 |
| `virt.c` | 1735-1753 | `create_fw_cfg()` | fw_cfg |
| `virt.c` | 1757-1792 | `create_pcie_irq_map()` | PCIe 中断映射 |
| `virt.c` | 1912-2042 | `create_pcie()` | PCIe Host Bridge |
| `virt.c` | 2119-2128 | `machvirt_dtb()` | DTB 回调 |
| `virt.c` | 2162-2199 | `virt_machine_done()` | 终态处理 |
| `virt.c` | 3262-3268 | `virt_is_acpi_enabled()` | ACPI 选择 |
| `boot.c` | 431-441 | `fdt_add_psci_node()` | PSCI 节点 |
| `boot.c` | 444-465 | `fdt_add_memory_node()` | Memory 节点 |
| `boot.c` | 467-653 | `arm_load_dtb()` | DTB 修补与写入 |
| `boot.c` | 639 | `rom_add_blob_fixed_as()` | 写入 Guest RAM |
| `sysbus-fdt.c` | 134-143 | `bindings[]` | 动态设备绑定表 |
| `sysbus-fdt.c` | 178-217 | `platform_bus_add_all_fdt_nodes()` | Platform Bus FDT |

## 附录 B：FDT 属性速查表

| 属性 | 含义 | 示例 |
|------|------|------|
| `compatible` | 设备驱动匹配字符串 | `"arm,pl011\0arm,primecell"` |
| `reg` | MMIO 地址和大小 | `<0x00 0x09000000 0x00 0x1000>` |
| `interrupts` | 中断描述（type, id, flags） | `<GIC_SPI 1 LEVEL_HI>` |
| `#address-cells` | 子节点地址 cell 数 | `<2>` |
| `#size-cells` | 子节点大小 cell 数 | `<2>` |
| `#interrupt-cells` | 中断描述 cell 数 | `<3>` |
| `interrupt-controller` | 标记为中断控制器 | （空属性） |
| `interrupt-parent` | 引用中断控制器 phandle | `<&gic>` |
| `phandle` | 节点唯一标识，供其他节点引用 | `<0x8001>` |
| `clocks` | 引用时钟源 phandle | `<&apb_pclk &apb_pclk>` |
| `clock-names` | 时钟名称 | `"uartclk\0apb_pclk"` |
| `dma-coherent` | DMA 一致性标记 | （空属性） |
| `device_type` | 设备类型 | `"cpu"`, `"memory"`, `"pci"` |
| `enable-method` | CPU 启动方式 | `"psci"` |
| `numa-node-id` | NUMA 节点归属 | `<0>` |
| `ranges` | 地址空间翻译 | PCIe IO/MMIO 映射 |
| `msi-map` | MSI 设备→控制器映射 | `<0 &its 0 0x10000>` |
| `iommu-map` | 设备→IOMMU 映射 | `<0 &smmu 0 0x10000>` |
| `status` | 节点使能状态 | `"okay"`, `"disabled"` |
| `stdout-path` | 默认控制台设备路径 | `"/pl011@9000000"` |

## 附录 C：关联文档

| 文档 | 路径 | 关联内容 |
|------|------|----------|
| Machine 建立流程 | `darren/architecture/03-Machine建立流程深度分析.md` | FDT 创建在 machvirt_init() 中的位置 |
| ACPI 表生成 | `darren/arm64/01-ACPI表生成与启动流程深度分析.md` | FDT 与 ACPI 共存机制 |
| GICv3 架构 | `darren/arm64/03-GICv3中断控制器模拟架构深度分析.md` | GIC FDT 节点的硬件模型 |
| UART 交互 | `darren/device-model/03-Chardev子系统与UART交互深度分析.md` | PL011 FDT 节点的设备实现 |
| 设备模型 | `darren/device-model/00-设备模型与virtio深度分析.md` | virtio-mmio FDT 节点的框架 |
| 内存子系统 | `darren/memory/00-内存子系统深度分析.md` | Memory FDT 节点与 MemoryRegion 关系 |
