# Machine 建立流程深度分析

> QEMU 版本：11.0.50  
> 分析日期：2025年  
> 目标平台：ARM virt machine（AArch64）  
> 外设范围：UART (PL011)、硬盘 (virtio-blk)、网卡 (virtio-net)  
> 相关文档：[00-全局架构概览](00-全局架构概览.md) | [01-QOM对象模型深度分析](01-QOM对象模型深度分析.md) | [../device-model/00-设备模型与virtio深度分析](../device-model/00-设备模型与virtio深度分析.md)

---

## 目录

- [第一部分：启动入口与 Machine 选择](#第一部分启动入口与-machine-选择)
  - [§1 main() → qemu_init() 启动序列](#1-main--qemu_init-启动序列)
  - [§2 Machine 类型选择与 QOM 注册](#2-machine-类型选择与-qom-注册)
  - [§3 machvirt_init() 总体结构](#3-machvirt_init-总体结构)
- [第二部分：地址映射与内存布局](#第二部分地址映射与内存布局)
  - [§4 virt 地址映射表 (base_memmap)](#4-virt-地址映射表-base_memmap)
  - [§5 IRQ 映射表 (a15irqmap)](#5-irq-映射表-a15irqmap)
  - [§6 系统内存构建](#6-系统内存构建)
  - [§7 Flash 与固件加载](#7-flash-与固件加载)
- [第三部分：CPU 创建与初始化](#第三部分cpu-创建与初始化)
  - [§8 CPU 拓扑计算](#8-cpu-拓扑计算)
  - [§9 CPU 对象实例化循环](#9-cpu-对象实例化循环)
  - [§10 arm_cpu_realizefn() 实现链](#10-arm_cpu_realizefn-实现链)
  - [§11 vCPU 线程创建](#11-vcpu-线程创建)
- [第四部分：GIC 创建与 IRQ 连线](#第四部分gic-创建与-irq-连线)
  - [§12 create_gic() 全流程](#12-create_gic-全流程)
  - [§13 GIC MMIO 映射](#13-gic-mmio-映射)
  - [§14 GIC↔CPU IRQ 连线](#14-giccpu-irq-连线)
- [第五部分：UART (PL011) 创建](#第五部分uart-pl011-创建)
  - [§15 create_uart() 流程](#15-create_uart-流程)
  - [§16 PL011 MMIO 注册](#16-pl011-mmio-注册)
  - [§17 Chardev 后端连接](#17-chardev-后端连接)
- [第六部分：virtio-mmio 传输层与设备](#第六部分virtio-mmio-传输层与设备)
  - [§18 create_virtio_devices() 流程](#18-create_virtio_devices-流程)
  - [§19 virtio-mmio 传输层实例化](#19-virtio-mmio-传输层实例化)
  - [§20 virtio-blk 后端连接](#20-virtio-blk-后端连接)
  - [§21 virtio-net 后端连接](#21-virtio-net-后端连接)
- [第七部分：设备 QOM 实例化通用机制](#第七部分设备-qom-实例化通用机制)
  - [§22 object_new → instance_init 链](#22-object_new--instance_init-链)
  - [§23 qdev_realize → realize 链](#23-qdev_realize--realize-链)
  - [§24 sysbus_mmio_map 与地址空间注册](#24-sysbus_mmio_map-与地址空间注册)
- [第八部分：线程模型](#第八部分线程模型)
  - [§25 QEMU 进程线程全景](#25-qemu-进程线程全景)
  - [§26 主线程生命周期](#26-主线程生命周期)
  - [§27 vCPU 线程模型 (TCG/KVM)](#27-vcpu-线程模型-tcgkvm)
  - [§28 IOThread 模型](#28-iothread-模型)
  - [§29 块 I/O 线程池](#29-块-io-线程池)
  - [§30 网络后端线程模型](#30-网络后端线程模型)
  - [§31 BQL 同步机制](#31-bql-同步机制)
- [第九部分：vm_start 与 Guest 执行](#第九部分vm_start-与-guest-执行)
  - [§32 vm_start() → resume_all_vcpus()](#32-vm_start--resume_all_vcpus)
  - [§33 从初始化到第一条 Guest 指令](#33-从初始化到第一条-guest-指令)
- [第十部分：端到端流程总览](#第十部分端到端流程总览)
  - [§34 machvirt_init 设备创建顺序表](#34-machvirt_init-设备创建顺序表)
  - [§35 全流程时序图](#35-全流程时序图)
- [附录](#附录)
  - [附录A：virt 完整地址映射表](#附录avirt-完整地址映射表)
  - [附录B：关键源文件索引](#附录b关键源文件索引)

---

## 第一部分：启动入口与 Machine 选择

### §1 main() → qemu_init() 启动序列

**源文件**：`main.c:69-96`，`vl.c`

QEMU 系统模拟器的启动流程：

```
main()                              [main.c:69]
  ├─ qemu_init(argc, argv)          [vl.c]
  │    ├─ 命令行解析
  │    ├─ 加速器初始化 (TCG/KVM/HVF)
  │    ├─ qemu_create_machine()     [vl.c:3757]
  │    │    ├─ select_machine()     [vl.c:1678]
  │    │    │    └─ QOM 查找 "virt" 类型
  │    │    └─ object_new_with_class(mc)
  │    ├─ 后端创建 (chardev/blockdev/netdev)
  │    ├─ qemu_init_board()         [vl.c:2708]
  │    │    └─ machine_run_board_init()  [machine.c:1597]
  │    │         └─ mc->init(machine)
  │    │              └─ machvirt_init()  [virt.c:2600]
  │    └─ accel_setup_post()
  │
  ├─ qemu_default_main()             [main.c:44]
  │    └─ qemu_main_loop()          [runstate.c:943]
  │         └─ while (!main_loop_should_exit())
  │              main_loop_wait(false)
  └─ qemu_cleanup()
```

### §2 Machine 类型选择与 QOM 注册

**源文件**：`virt.c:115-141, 4067-4090`

ARM virt 机器通过版本化宏注册多个 QOM 类型：

```c
// virt.c:4067-4085
DEFINE_VIRT_MACHINE(11, 1)   // "virt-11.1" (最新版本)
DEFINE_VIRT_MACHINE(11, 0)   // "virt-11.0"
DEFINE_VIRT_MACHINE(10, 1)   // "virt-10.1"
// ... 向前兼容到 virt-2.6

// 宏展开结果:
// - 注册 TYPE "virt-11.1-machine"
// - 别名 "virt" 指向最新版本
// - 每个版本有独立的 class_init 设置兼容性标志
```

`select_machine()` 通过 QOM 类型名称查找（`vl.c:1678-1703`）：
- `-machine virt` → 找到最新版本的 `virt-*-machine` 类型
- `-machine virt-10.0` → 找到特定版本

### §3 machvirt_init() 总体结构

**源文件**：`virt.c:2600-2960`

`machvirt_init()` 是 ARM virt 机器的核心初始化函数，按以下阶段执行：

```
machvirt_init(machine)
  │
  │  阶段1: 配置解析 (2605-2647)
  ├─ 计算 CPU 拓扑、PA 位数
  ├─ virt_set_memmap() → 生成地址映射表
  ├─ finalize_gic_version() → 确定 GICv2/v3
  ├─ finalize_msi_controller() → 确定 MSI 控制器
  │
  │  阶段2: 安全扩展与固件 (2649-2682)
  ├─ 创建 Secure 内存视图 (if vms->secure)
  ├─ virt_firmware_init() → 加载 UEFI/bootrom
  ├─ 配置 PSCI conduit (HVC/SMC)
  │
  │  阶段3: 验证 (2684-2735)
  ├─ 检查 CPU 数量 vs GIC 容量
  ├─ 验证加速器能力 (Secure/Virt/MTE)
  │
  │  阶段4: 核心组件创建 (2737-2861)
  ├─ create_fdt()           → 设备树
  ├─ CPU 实例化循环         → N 个 vCPU
  ├─ RAM 映射              → system memory
  ├─ create_gic()           → GICv3 + IRQ 连线
  │
  │  阶段5: 外设创建 (2883-2938)
  ├─ create_uart()          → PL011 UART × 1-2
  ├─ create_rtc()           → PL031 RTC
  ├─ create_pcie()          → PCIe 主控制器
  ├─ create_gpio_devices()  → GPIO / ACPI GED
  ├─ create_virtio_devices()→ virtio-mmio × 32
  ├─ create_fw_cfg()        → fw_cfg 设备
  ├─ create_platform_bus()  → 平台总线
  │
  │  阶段6: 内核加载 (2952-2959)
  └─ arm_load_kernel()      → 加载 Guest 内核/DTB
```

---

## 第二部分：地址映射与内存布局

### §4 virt 地址映射表 (base_memmap)

**源文件**：`virt.c:161-208`

ARM virt 机器的物理地址空间静态布局：

```
  0x0000_0000 ┌────────────────────────┐
              │ Flash (128 MB)         │ UEFI 固件
  0x0800_0000 ├────────────────────────┤
              │ CPU Peripherals        │
              │ ├─ GIC Dist  (64 KB)   │ 0x0800_0000
              │ ├─ GIC CPU   (64 KB)   │ 0x0801_0000 (GICv2)
              │ ├─ GIC ITS   (128 KB)  │ 0x0808_0000
              │ └─ GIC Redist (15 MB)  │ 0x080A_0000
  0x0900_0000 ├────────────────────────┤
              │ UART0        (4 KB)    │ PL011 串口
  0x0901_0000 │ RTC          (4 KB)    │ PL031 时钟
  0x0902_0000 │ fw_cfg       (24 B)    │ 固件配置
  0x0903_0000 │ GPIO         (4 KB)    │ PL061 GPIO
  0x0904_0000 │ UART1        (4 KB)    │ 第二串口
  0x0905_0000 │ SMMU                   │ IOMMU
  0x0908_0000 │ ACPI GED               │ 电源管理
  0x090A_0000 │ PVTime       (64 KB)   │ 偷取时间
  0x0A00_0000 ├────────────────────────┤
              │ virtio-mmio × 32       │ 每个 512 B
              │ (0x0A00_0000..0x0A00_3E00)
  0x0C00_0000 ├────────────────────────┤
              │ Platform Bus (32 MB)   │ 动态设备
  0x0E00_0000 ├────────────────────────┤
              │ Secure MEM (16 MB)     │ TrustZone
  0x1000_0000 ├────────────────────────┤
              │ PCIe MMIO (~752 MB)    │ PCI 设备内存
  0x3EFF_0000 │ PCIe PIO (64 KB)       │ PCI I/O
  0x3F00_0000 │ PCIe ECAM (16 MB)      │ PCI 配置
  0x4000_0000 ├────────────────────────┤
              │ RAM (1 GB 起)          │ Guest 内存
              │ (大小由 -m 参数决定)    │
              └────────────────────────┘
```

### §5 IRQ 映射表 (a15irqmap)

**源文件**：`virt.c:243-254`

设备 SPI 中断号分配（GIC SPI 编号，实际 INTID = SPI + 32）：

| 设备 | SPI 编号 | INTID | 说明 |
|------|---------|-------|------|
| UART0 | 1 | 33 | 主串口 |
| RTC | 2 | 34 | 实时时钟 |
| PCIe | 3-6 | 35-38 | PCI 中断 A/B/C/D |
| GPIO | 7 | 39 | GPIO 中断 |
| UART1 | 8 | 40 | 第二串口 |
| ACPI GED | 9 | 41 | 电源管理事件 |
| virtio-mmio | 16-47 | 48-79 | 32 个 virtio 设备 |
| GICv2M | 48-111 | 80-143 | MSI 中断 |
| SMMU | 74-77 | 106-109 | IOMMU 中断 |
| Platform Bus | 112-143 | 144-175 | 平台设备 |

### §6 系统内存构建

**源文件**：`virt.c:2606, 2649-2660, 2850-2851`

```
get_system_memory()  → 全局 system MemoryRegion (根)
  │
  ├─ RAM 子区域 (virt.c:2850)
  │    memory_region_add_subregion(sysmem, 0x4000_0000, machine->ram)
  │
  ├─ Secure 覆盖层 (virt.c:2656-2660, if secure)
  │    secure_sysmem = MemoryRegion("secure-memory")
  │    memory_region_add_subregion_overlap(secure_sysmem, 0, sysmem, -1)
  │    // Secure 设备以更高优先级覆盖
  │
  ├─ 各设备 MMIO 子区域
  │    ├─ GIC Distributor/Redistributor
  │    ├─ UART0/UART1
  │    ├─ virtio-mmio × 32
  │    ├─ PCIe ECAM/MMIO
  │    └─ ...
  │
  └─ Flash 区域 (pflash)
       virt_flash_map() → 0x0000_0000
```

### §7 Flash 与固件加载

**源文件**：`virt.c:1604+, 2663-2664, 4013-4060`

```
virt_instance_init()              [virt.c:4013]
  └─ virt_flash_create(vms)       创建 pflash 设备对象

machvirt_init()
  └─ virt_firmware_init()         [virt.c:2663]
       ├─ 加载 -bios / -pflash 指定的固件
       ├─ virt_flash_map() 将 Flash 映射到地址空间
       └─ 设置 UEFI 入口点
```

---

## 第三部分：CPU 创建与初始化

### §8 CPU 拓扑计算

**源文件**：`virt.c:2202-2216, 3414-3449`

```c
// virt.c:3414 - 计算所有可能的 CPU ID
virt_possible_cpu_arch_ids(machine)
{
    for (n = 0; n < max_cpus; n++) {
        // MPIDR 亲和性计算
        cpus[n].arch_id = virt_cpu_mp_affinity(vms, n);
        // 填充 socket/cluster/core/thread ID
        cpus[n].props.socket_id = ...;
        cpus[n].props.cluster_id = ...;
        cpus[n].props.core_id = ...;
        cpus[n].props.thread_id = ...;
    }
}

// virt.c:2202 - MPIDR 计算
// 将线性 CPU 编号映射为层级亲和性字段:
// Aff0 (bits[7:0]) | Aff1 (bits[15:8]) | Aff2 (bits[23:16])
virt_cpu_mp_affinity(vms, cpu_index)
```

### §9 CPU 对象实例化循环

**源文件**：`virt.c:2739-2841`

```
for (n = 0; n < smp_cpus; n++) {
    // 1. 创建 CPU QOM 对象
    cpuobj = object_new(possible_cpus->cpus[n].type)     [2748]
    //   → type_initialize() → class_init
    //   → instance_init: arm_cpu_initfn()

    // 2. 设置属性
    set_int(cpuobj, "mp-affinity", arch_id)               [2749]
    cs->cpu_index = n                                      [2752]

    // 3. 特性配置
    set_bool(cpuobj, "has_el3", vms->secure)               [2760]
    set_bool(cpuobj, "has_el2", vms->virt)                 [2764]

    // 4. 内存链接
    set_link(cpuobj, "memory", sysmem)                     [2783]
    set_link(cpuobj, "secure-memory", secure_sysmem)       [2786]

    // 5. MTE tag 内存 (if enabled)
    set_link(cpuobj, "tag-memory", tag_sysmem)             [2820]

    // 6. 实现 (realize)
    qdev_realize(DEVICE(cpuobj), NULL, &error_fatal)       [2839]
    //   → arm_cpu_realizefn()
    //   → qemu_init_vcpu() → 创建 vCPU 线程
    object_unref(cpuobj)                                   [2840]
}
```

### §10 arm_cpu_realizefn() 实现链

**源文件**：`cpu.c:1740-1765, 2074-2082`

```
qdev_realize(DEVICE(cpuobj))
  └─ device_set_realized()                [qdev.c:474]
       └─ dc->realize(dev)
            └─ arm_cpu_realizefn()         [cpu.c:1740]
                 ├─ 特性验证与注册
                 ├─ MPIDR 默认值设置       [cpu.c:2074]
                 ├─ EL/安全特性配置
                 ├─ 系统寄存器注册 (cpregs)
                 ├─ GDB 寄存器注册
                 ├─ qemu_init_vcpu(cs)     ← 创建 vCPU 线程
                 │    [cpus.c:709]
                 └─ cpu_reset(cs)          ← CPU 复位
```

### §11 vCPU 线程创建

**源文件**：`cpus.c:709-730`

`qemu_init_vcpu()` 根据加速器类型选择线程创建方式：

```
qemu_init_vcpu(cs)                        [cpus.c:709]
  │
  ├─ TCG 单线程模式 (icount/RR):
  │    rr_start_vcpu_thread()              [tcg-accel-ops-rr.c:180]
  │    └─ qemu_thread_create("rr_cpu", rr_cpu_thread_fn)
  │       └─ rr_cpu_thread_fn()            [tcg-accel-ops-rr.c:199]
  │            所有 vCPU 共用一个线程，轮转执行
  │
  ├─ TCG MTTCG 模式 (默认):
  │    mttcg_start_vcpu_thread()           [tcg-accel-ops-mttcg.c:124]
  │    └─ qemu_thread_create("mttcg_cpu", mttcg_cpu_thread_fn)
  │       └─ mttcg_cpu_thread_fn()         [tcg-accel-ops-mttcg.c:65]
  │            每个 vCPU 独立线程
  │
  └─ KVM 模式:
       kvm_start_vcpu_thread()             [kvm-accel-ops.c:68]
       └─ qemu_thread_create("kvm_cpu", kvm_vcpu_thread_fn)
          └─ kvm_vcpu_thread_fn()          [kvm-accel-ops.c:31]
               每个 vCPU 独立线程 + KVM fd

  // 线程创建后，等待 cpu->created 信号
  while (!cpu->created) {
      qemu_cond_wait(&qemu_cpu_cond)       [cpus.c:713-730]
  }
```

**关键点**：vCPU 线程在 `qdev_realize()` 期间就被创建，但创建后立即阻塞等待 `vm_start()`。

---

## 第四部分：GIC 创建与 IRQ 连线

### §12 create_gic() 全流程

**源文件**：`virt.c:1122-1293`

```
create_gic(vms, sysmem)
  │
  │  1. 创建 GIC 设备 (1159)
  ├─ gicdev = qdev_new(gictype)
  │    // gictype = "arm-gicv3" 或 "arm-gic"
  │
  │  2. 配置属性 (1160-1210)
  ├─ set_uint("revision", 3)
  ├─ set_uint("num-cpu", smp_cpus)
  ├─ set_uint("num-irq", NUM_IRQS + 32)
  ├─ set_bool("has-lpi", true)       // ITS 支持
  ├─ set_bool("has-nmi", true)       // NMI 支持
  │
  │  3. 实现 (1212)
  ├─ sysbus_realize_and_unref(gicbusdev)
  │    → gicv3_realize() → 创建 GICD/GICR/ITS MMIO 区域
  │
  │  4. MMIO 映射 (1213-1225)
  ├─ sysbus_mmio_map(gic, 0, VIRT_GIC_DIST.base)   // 0x0800_0000
  ├─ sysbus_mmio_map(gic, 1, VIRT_GIC_REDIST.base)  // 0x080A_0000
  │
  │  5. CPU↔GIC IRQ 连线 (1233-1283)
  └─ for (irq = 0; irq < smp_cpus; irq++) {
       // GIC 输出 → CPU 输入
       sysbus_connect_irq(gic, i,
           qdev_get_gpio_in(cpudev, ARM_CPU_IRQ))    // IRQ
       sysbus_connect_irq(gic, i + smp_cpus,
           qdev_get_gpio_in(cpudev, ARM_CPU_FIQ))    // FIQ
       sysbus_connect_irq(gic, i + 2*smp_cpus,
           qdev_get_gpio_in(cpudev, ARM_CPU_VIRQ))   // VIRQ
       sysbus_connect_irq(gic, i + 3*smp_cpus,
           qdev_get_gpio_in(cpudev, ARM_CPU_VFIQ))   // VFIQ
     }
```

### §13 GIC MMIO 映射

**源文件**：`arm_gicv3_common.c:350-367`

GIC 在 `realize` 时创建 MMIO 区域：

```
gicv3_realize()
  └─ gicv3_init_irqs_and_mmio()        [arm_gicv3_common.c:350]
       ├─ Distributor MMIO:
       │    memory_region_init_io(&s->iomem_dist, ...)
       │    sysbus_init_mmio(sbd, &s->iomem_dist)
       │    → 映射到 0x0800_0000 (64 KB)
       │
       ├─ Redistributor MMIO (per-CPU):
       │    memory_region_init_io(&region->iomem, ...)
       │    sysbus_init_mmio(sbd, &region->iomem)
       │    → 映射到 0x080A_0000 (每 CPU 128 KB)
       │
       └─ IRQ 输出线 (per-CPU):
            sysbus_init_irq(sbd, &cs->parent_irq)
            sysbus_init_irq(sbd, &cs->parent_fiq)
            sysbus_init_irq(sbd, &cs->parent_virq)
            sysbus_init_irq(sbd, &cs->parent_vfiq)
```

### §14 GIC↔CPU IRQ 连线

连线在 `create_gic()` 中完成后的效果：

```
GICv3 Device                          ARM CPU
  ┌──────────┐                      ┌──────────┐
  │          ├─── parent_irq ──────►│ ARM_CPU_IRQ  → CPU_INTERRUPT_HARD
  │          ├─── parent_fiq ──────►│ ARM_CPU_FIQ  → CPU_INTERRUPT_FIQ
  │  per-CPU ├─── parent_virq ────►│ ARM_CPU_VIRQ → CPU_INTERRUPT_VIRQ
  │          ├─── parent_vfiq ────►│ ARM_CPU_VFIQ → CPU_INTERRUPT_VFIQ
  │          ├─── parent_nmi ─────►│ ARM_CPU_NMI  → CPU_INTERRUPT_NMI
  └──────────┘                      └──────────┘

处理函数: arm_cpu_set_irq() [cpu.c:781]
  → cpu_interrupt() / cpu_reset_interrupt()
```

---

## 第五部分：UART (PL011) 创建

### §15 create_uart() 流程

**源文件**：`virt.c:1295-1345`

```
create_uart(vms, VIRT_UART0, sysmem, serial_hd(0), false)
  │
  │  1. 创建设备 (1304)
  ├─ dev = qdev_new(TYPE_PL011)
  │
  │  2. 连接 chardev 后端 (1308)
  ├─ qdev_prop_set_chr(dev, "chardev", chr)
  │    // chr = serial_hd(0) → 对应 -serial stdio
  │
  │  3. 实现 + MMIO 映射 (1309-1313)
  ├─ sysbus_realize_and_unref(sbd)
  │    → pl011_realize()
  │         └─ qemu_chr_fe_set_handlers() 安装收发回调
  │
  │  4. 手动映射 MMIO (1311)
  ├─ memory_region_add_subregion(sysmem, base, sysbus_mmio_get_region(sbd, 0))
  │    // base = 0x0900_0000 (VIRT_UART0)
  │
  │  5. IRQ 连线 (1312)
  ├─ sysbus_connect_irq(sbd, 0, qdev_get_gpio_in(vms->gic, irq))
  │    // irq = a15irqmap[VIRT_UART0] = SPI 1 → GIC INTID 33
  │
  │  6. FDT 节点 (1314-1344)
  └─ 创建 DTB 节点 + stdout-path 别名
```

### §16 PL011 MMIO 注册

**源文件**：`pl011.c:645-655, 692-699`

```c
// pl011.c:645 - instance_init 中创建 MMIO 区域
static void pl011_init(Object *obj)
{
    SysBusDevice *sbd = SYS_BUS_DEVICE(obj);
    PL011State *s = PL011(obj);

    memory_region_init_io(&s->iomem, obj, &pl011_ops, s,
                          "pl011", 0x1000);
    sysbus_init_mmio(sbd, &s->iomem);
    sysbus_init_irq(sbd, &s->irq);
}
```

`pl011_ops` 包含读写处理函数：
- `pl011_read()` — Guest 读 UART 数据/状态寄存器
- `pl011_write()` — Guest 写 UART 数据/控制寄存器

### §17 Chardev 后端连接

**源文件**：`char-fe.c:192-214`，`pl011.c:667-668`

```
命令行: -serial stdio

后端创建链:
  vl.c 解析 -serial → serial_hd(0) 返回 Chardev 对象
  → chardev "stdio" 后端 [char-stdio.c:88-123]
  → 绑定到终端 stdin/stdout

前端连接:
  qdev_prop_set_chr(dev, "chardev", chr)
  → pl011_realize() 中:
       qemu_chr_fe_set_handlers(&s->chr, ...)  [pl011.c:667]
       // 注册接收回调: pl011_can_receive, pl011_receive

数据流:
  发送: Guest 写 UARTDR → pl011_write() → qemu_chr_fe_write() → stdout
  接收: stdin → chardev 回调 → pl011_receive() → FIFO → Guest 读 UARTDR
```

---

## 第六部分：virtio-mmio 传输层与设备

### §18 create_virtio_devices() 流程

**源文件**：`virt.c:1504-1569`

```
create_virtio_devices(vms)
  │
  └─ for (i = 0; i < NUM_VIRTIO_TRANSPORTS; i++) {    // 32 个传输层
       // 1. 计算地址和 IRQ
       hwaddr base = vms->memmap[VIRT_MMIO].base + i * 0x200
       // base = 0x0A00_0000 + i * 512
       int irq = vms->irqmap[VIRT_MMIO] + i
       // irq = SPI 16 + i → INTID 48 + i

       // 2. 创建 virtio-mmio 传输设备 (1537)
       sysbus_create_simple("virtio-mmio", base,
                            qdev_get_gpio_in(vms->gic, irq))
       // 等价于:
       //   dev = qdev_new("virtio-mmio")
       //   sysbus_realize_and_unref(dev)
       //   sysbus_mmio_map(dev, 0, base)
       //   sysbus_connect_irq(dev, 0, gic_irq)

       // 3. 创建 DT 节点 (1552-1567)
       // compatible = "virtio,mmio"
       // reg = <base 0x200>
       // interrupts = <GIC_SPI irq IRQ_TYPE_EDGE_RISING>
     }
```

**设计**：32 个 virtio-mmio 传输层在机器初始化时全部创建，但都是空的。用户通过 `-device virtio-blk-device,...` 命令行将 virtio 后端设备**插入**到空闲传输层中。

### §19 virtio-mmio 传输层实例化

**源文件**：`virtio-mmio.c:772-795`

```
virtio_mmio_realizefn(dev)
  │
  │  1. 创建 IRQ 输出
  ├─ sysbus_init_irq(sbd, &proxy->irq)
  │
  │  2. 创建 MMIO 区域
  ├─ memory_region_init_io(&proxy->iomem, ..., &virtio_mem_ops, ...)
  │    // size = 0x200 (512 bytes)
  │    // virtio_mem_ops: virtio_mmio_read / virtio_mmio_write
  │
  │  3. 注册到 SysBus
  └─ sysbus_init_mmio(sbd, &proxy->iomem)
```

MMIO 寄存器布局（virtio-mmio v2 规范）：

| 偏移 | 寄存器 | 说明 |
|------|-------|------|
| 0x000 | MagicValue | 0x74726976 ("virt") |
| 0x004 | Version | 2 |
| 0x008 | DeviceID | 设备类型 (1=net, 2=blk, ...) |
| 0x030 | QueueSel | 队列选择 |
| 0x044 | QueueReady | 队列就绪 |
| 0x050 | QueueNotify | 队列通知 (doorbell) |
| 0x060 | InterruptStatus | 中断状态 |
| 0x064 | InterruptACK | 中断确认 |
| 0x070 | Status | 设备状态 |

### §20 virtio-blk 后端连接

**源文件**：`virtio-blk.c:1723-1785`

```
命令行: -drive file=disk.qcow2,if=none,id=hd0 \
        -device virtio-blk-device,drive=hd0

后端创建链:
  -drive 解析 → drive_new()                [vl.c:657]
  → blk_new_open(filename, ...)            [block-backend.c:423]
    ├─ bdrv_open(filename, "qcow2", ...)   打开磁盘镜像
    └─ blk_new(aio_context, ...)           创建 BlockBackend

设备插入:
  -device virtio-blk-device,drive=hd0
  → virtio_blk_device_realize()            [virtio-blk.c:1723]
       ├─ conf->conf.blk 已通过属性绑定 BlockBackend
       ├─ 验证 BlockBackend 存在              [1732-1738]
       ├─ virtio_init(vdev, VIRTIO_ID_BLOCK, ...) 初始化 virtio 核心
       ├─ 创建请求 virtqueue
       └─ blk_set_aio_context() → 绑定 AIO 上下文

IOThread 集成:
  -object iothread,id=io1
  -device virtio-blk-device,drive=hd0,iothread=io1
  → virtio-blk 将 BlockBackend 的 AioContext 切换到 IOThread
    blk_set_aio_context(blk, iothread_get_aio_context(iothread))
    // [virtio-blk.c:1592-1624]
```

### §21 virtio-net 后端连接

**源文件**：`virtio-net.c:3867-3998`

```
命令行: -netdev user,id=net0 \
        -device virtio-net-device,netdev=net0

后端创建链:
  -netdev user → 创建 SLIRP NetClientState 后端
  → net_client_init() → net_init_slirp()

设备实例化:
  -device virtio-net-device,netdev=net0
  → virtio_net_device_realize()           [virtio-net.c:3867]
       ├─ virtio_init(vdev, VIRTIO_ID_NET, ...)
       ├─ 创建 TX/RX virtqueue 对 (per-queue)
       ├─ n->nic = qemu_new_nic(...)       [3991-3997]
       │    创建 NIC 前端 NetClientState
       │    绑定到 netdev=net0 后端
       └─ qemu_format_nic_info_str(...)

数据流:
  发送: Guest → virtqueue → virtio_net_flush_tx()
        → qemu_send_packet() → SLIRP/TAP 后端
  接收: SLIRP/TAP → virtio_net_receive()
        → virtqueue → Guest
```

---

## 第七部分：设备 QOM 实例化通用机制

### §22 object_new → instance_init 链

**源文件**：`object.c:496-512, 725-729`

```
object_new(typename)                        [object.c:725]
  └─ object_new_with_type(type)             [object.c:729]
       └─ object_initialize_with_type(obj)  [object.c:496]
            │
            │  1. 类型初始化 (惰性，仅首次)
            ├─ type_initialize(type)         [object.c:336]
            │    ├─ 递归初始化父类
            │    └─ type->class_init(klass)  设置虚函数表
            │
            │  2. 实例初始化 (每次创建)
            └─ object_init_with_type(obj, type)
                 ├─ 递归调用父类 instance_init
                 └─ type->instance_init(obj)
                      // 例如: arm_cpu_initfn(), pl011_init()
```

**关键**：`object_new()` 只做分配 + 初始化，**不做 realize**。设备在 `instance_init` 中创建 MMIO 区域和 IRQ 输出，但不与系统连接。

### §23 qdev_realize → realize 链

**源文件**：`qdev.c:265-278, 474-520`

```
qdev_realize(dev, bus, errp)                [qdev.c:265]
  └─ object_property_set_bool(obj, "realized", true)
       └─ device_set_realized(obj, true)    [qdev.c:474]
            │
            ├─ hotplug_handler_pre_plug() (if bus)
            │
            ├─ dc->realize(dev, errp)       设备特定 realize
            │    // 例如:
            │    // pl011_realize() → 安装 chardev 回调
            │    // virtio_mmio_realizefn() → 创建 MMIO
            │    // arm_cpu_realizefn() → 注册 cpregs + 创建 vCPU 线程
            │
            ├─ dev->realized = true         [qdev.c:520]
            │
            └─ hotplug_handler_plug() (if bus)
```

### §24 sysbus_mmio_map 与地址空间注册

```
设备 MMIO 注册流程:

1. instance_init 中:
   memory_region_init_io(&s->iomem, obj, &ops, s, name, size)
   sysbus_init_mmio(sbd, &s->iomem)
   // 仅注册到 SysBus 设备的 MMIO 列表，不映射

2. Board 代码中 (virt.c):
   方式A: sysbus_mmio_map(sbd, n, addr)
   方式B: memory_region_add_subregion(sysmem, addr,
              sysbus_mmio_get_region(sbd, n))
   // 将 MMIO 区域实际插入到系统地址空间树

3. FlatView 更新:
   address_space_update_topology()
   → 重新生成 FlatView → Guest 访问该地址时触发设备 read/write
```

---

## 第八部分：线程模型

### §25 QEMU 进程线程全景

典型的 4 核 ARM virt 虚拟机，线程布局如下：

```
QEMU 进程
  │
  ├─ 主线程 (Main Thread)
  │    ├─ 职责: 事件循环、设备模拟、管理接口
  │    ├─ 入口: main() → qemu_main_loop()
  │    └─ 循环: main_loop_wait() → GLib poll → timers → BH
  │
  ├─ vCPU 线程 × 4 (TCG MTTCG 或 KVM)
  │    ├─ 职责: Guest 代码执行
  │    ├─ TCG: mttcg_cpu_thread_fn() → cpu_exec()
  │    ├─ KVM: kvm_vcpu_thread_fn() → kvm_cpu_exec()
  │    └─ 每线程独立，通过 BQL 与主线程同步
  │
  ├─ IOThread × N (可选, 用户配置)
  │    ├─ 职责: 独立事件循环，服务设备 I/O
  │    ├─ 入口: iothread_run() → aio_poll()
  │    └─ 用于 virtio-blk/scsi 分离 I/O 处理
  │
  ├─ 块 I/O 线程池 (Thread Pool)
  │    ├─ 职责: 执行同步文件 I/O 操作
  │    ├─ 入口: worker_thread() → request_fn()
  │    ├─ 动态伸缩，按需创建
  │    └─ 完成后通过 BH 通知 AioContext
  │
  └─ 其他可选线程
       ├─ VNC/SPICE 显示线程
       ├─ 迁移线程
       └─ vhost-user 工作线程
```

### §26 主线程生命周期

**源文件**：`main.c:44-55, 69-85`，`runstate.c:943-952`

```
main()
  │
  ├─ qemu_init()         ← 持有 BQL，完成所有初始化
  │    └─ 创建 machine、CPU、设备、后端
  │
  ├─ bql_unlock()        ← 释放 BQL
  │
  ├─ qemu_main_loop()    ← 进入主事件循环
  │    └─ while (!main_loop_should_exit()) {
  │         main_loop_wait(false)    [main-loop.c:563]
  │         // GLib prepare → query → poll → check → dispatch
  │         // → 处理 fd 事件、timers、BH
  │       }
  │
  └─ qemu_cleanup()      ← 清理退出
```

### §27 vCPU 线程模型 (TCG/KVM)

**TCG MTTCG**（`tcg-accel-ops-mttcg.c:65-121`）：

```
mttcg_cpu_thread_fn(arg)
  │
  ├─ rcu_register_thread()
  ├─ tcg_register_thread()
  │
  ├─ bql_lock()
  ├─ qemu_thread_get_self(cs->thread)
  ├─ cs->thread_id = qemu_get_thread_id()
  ├─ cs->created = true                  ← 通知创建完成
  ├─ qemu_cond_signal(&qemu_cpu_cond)
  │
  └─ while (!cpu->unplug || cpu_can_run(cpu)) {
       ├─ 等待 cpu_can_run (阻塞直到 vm_start)
       ├─ tcg_cpus_exec(cpu)
       │    └─ cpu_exec(cs)              ← TCG 执行循环
       └─ qemu_wait_io_event(cpu)
     }
```

**KVM**（`kvm-accel-ops.c:31-66`）：

```
kvm_vcpu_thread_fn(arg)
  │
  ├─ rcu_register_thread()
  ├─ kvm_init_vcpu(cpu, &error_fatal)    ← KVM_CREATE_VCPU
  │
  ├─ bql_lock()
  ├─ cs->created = true
  │
  └─ while (!cpu->unplug || cpu_can_run(cpu)) {
       ├─ 等待 cpu_can_run
       ├─ kvm_cpu_exec(cpu)              ← KVM_RUN 循环
       └─ qemu_wait_io_event(cpu)
     }
```

### §28 IOThread 模型

**源文件**：`iothread.c:28-65, 174-208`

```
命令行: -object iothread,id=io1

创建流程:
  object_new_with_props(TYPE_IOTHREAD, ...)   [iothread.c:174]
  → iothread_complete()                       [iothread.c:145]
       ├─ aio_context_new()                   创建独立 AioContext
       └─ qemu_thread_create("iothread", iothread_run)

线程函数:
  iothread_run(opaque)                         [iothread.c:28]
    └─ while (!iothread->stopping) {
         aio_poll(iothread->ctx, true)         处理 AIO 事件
         // 或 g_main_loop_run() 处理 GLib 事件
       }

与 virtio-blk 集成:
  -device virtio-blk-device,drive=hd0,iothread=io1
  → virtio_blk_device_realize()
       └─ blk_set_aio_context(blk, iothread_ctx)  [virtio-blk.c:1592]
       // 将块设备 I/O 从主线程迁移到 IOThread
```

**IOThread 优势**：
- 设备 I/O 不经过主线程，不受 BQL 竞争影响
- virtio-blk 的 I/O 完成回调在 IOThread 中执行
- 显著降低延迟，提升 IOPS

### §29 块 I/O 线程池

**源文件**：`thread-pool.c:82-135, 178-217, 243-293`

```
thread_pool_submit_aio(pool, func, arg, cb, opaque)
  │
  ├─ 创建 ThreadPoolElement 请求
  │    req->func = func         同步 I/O 函数 (如 pread/pwrite)
  │    req->common.cb = cb      完成回调
  │
  ├─ 提交到 worker 队列
  │    // 如果没有空闲 worker，按需创建新线程
  │
  └─ Worker 线程执行:
       worker_thread()           [thread-pool.c:82]
         └─ req->ret = req->func(req->arg)   执行同步 I/O
              │
              └─ 完成后:
                   通知 AioContext (通过 eventfd)
                   → AioContext 的 BH 执行 cb(opaque, ret)
                   → 回调在 AioContext 所在线程运行
                     (主线程 或 IOThread)
```

### §30 网络后端线程模型

**源文件**：`slirp.c:116-140`，`tap.c:109-136`

| 后端 | 线程 | 收包机制 |
|------|------|---------|
| SLIRP (user) | 主线程 | GLib poll fd → `net_slirp_send_packet()` |
| TAP | 主线程 | `qemu_set_fd_handler(tap_fd, tap_send)` |
| vhost-net | 内核态 | 内核 vhost 直接处理，绕过 QEMU |
| vhost-user | 独立进程 | Unix socket + eventfd |

**关键**：默认网络后端不使用独立线程，收发包都在主线程事件循环中处理。高性能场景应使用 vhost-net 或 vhost-user。

### §31 BQL 同步机制

**源文件**：`cpus.c:573-586`

BQL（Big QEMU Lock）是保护设备模拟状态的全局互斥锁：

```
线程持有 BQL 的时机:

主线程:
  qemu_init() 期间 ──────── 持有 BQL
  main_loop_wait():
    ├─ 进入 poll 前 ──────── 释放 BQL
    ├─ poll 阻塞 ──────────  无 BQL
    └─ poll 返回后 ────────  重新获取 BQL
       dispatch 回调 ──────  持有 BQL

vCPU 线程 (TCG MTTCG):
  mttcg_cpu_thread_fn():
    ├─ 初始化 ────────────── 持有 BQL
    ├─ cpu_exec() 执行 TB ── 释放 BQL (大部分时间)
    ├─ MMIO 处理 ──────────  获取 BQL (BQL_LOCK_GUARD)
    └─ qemu_wait_io_event ── 释放 BQL

vCPU 线程 (KVM):
  kvm_vcpu_thread_fn():
    ├─ kvm_cpu_exec():
    │   ├─ KVM_RUN 前 ────── 释放 BQL
    │   ├─ KVM_RUN 中 ────── 无 BQL (在 Guest 中)
    │   └─ VM Exit 后 ────── 获取 BQL
    │       处理 exit ────── 持有 BQL
    └─ qemu_wait_io_event ── 释放 BQL

IOThread:
  不需要 BQL，使用自己的 AioContext 锁
```

---

## 第九部分：vm_start 与 Guest 执行

### §32 vm_start() → resume_all_vcpus()

**源文件**：`runstate.c:800-805`，`cpus.c:669-681`

```
vm_start()                              [runstate.c:800]
  ├─ vm_prepare_start(step_pending=false)
  │    └─ 设置 runstate = RUN_STATE_RUNNING
  │
  └─ resume_all_vcpus()                 [cpus.c:669]
       └─ CPU_FOREACH(cpu) {
            cpu_resume(cpu)              [cpus.c:677]
              ├─ cpu->stop = false
              ├─ cpu->stopped = false
              └─ qemu_cpu_kick(cpu)      唤醒 vCPU 线程
                   // 通过 eventfd/signal 唤醒
          }
```

### §33 从初始化到第一条 Guest 指令

```
qemu_init()
  ├─ machvirt_init()
  │    ├─ CPU 创建 + realize → vCPU 线程创建 (阻塞等待)
  │    ├─ 设备创建
  │    └─ arm_load_kernel() → 加载 Guest 镜像
  │
  ├─ vm_start()
  │    └─ resume_all_vcpus() → qemu_cpu_kick() 唤醒所有 vCPU
  │
  └─ qemu_main_loop() → 主线程进入事件循环

vCPU 线程被唤醒后:
  TCG: mttcg_cpu_thread_fn()
    └─ cpu_can_run() → true
         └─ cpu_exec(cs)
              ├─ 第一次进入: env->pc = reset_vector (UEFI 入口或内核入口)
              ├─ tb_lookup() → 缓存未命中
              ├─ tb_gen_code() → 翻译第一个 TB
              └─ cpu_loop_exec_tb() → 执行第一条 Guest 指令

  KVM: kvm_vcpu_thread_fn()
    └─ cpu_can_run() → true
         └─ kvm_cpu_exec()
              └─ ioctl(KVM_RUN) → 硬件直接执行 Guest 第一条指令
```

---

## 第十部分：端到端流程总览

### §34 machvirt_init 设备创建顺序表

| 序号 | 函数调用 | 创建的组件 | MMIO 地址 | IRQ |
|------|---------|-----------|----------|-----|
| 1 | `virt_firmware_init()` | Flash/固件 | 0x0000_0000 | - |
| 2 | CPU 循环 × N | ARM CPU + vCPU 线程 | - | - |
| 3 | `fdt_add_timer_nodes()` | Timer DT 节点 | - | PPI 13/14/11/10 |
| 4 | RAM 映射 | 系统内存 | 0x4000_0000 | - |
| 5 | `create_gic()` | GICv3 + IRQ 连线 | 0x0800_0000 | - |
| 6 | `create_uart(UART0)` | PL011 串口 | 0x0900_0000 | SPI 1 |
| 7 | `create_uart(UART1)` | PL011 串口 (可选) | 0x0904_0000 | SPI 8 |
| 8 | `create_rtc()` | PL031 RTC | 0x0901_0000 | SPI 2 |
| 9 | `create_pcie()` | PCIe 主控 | 0x1000_0000 | SPI 3-6 |
| 10 | `create_acpi_ged()` | ACPI 电源管理 | 0x0908_0000 | SPI 9 |
| 11 | `create_virtio_devices()` | virtio-mmio × 32 | 0x0A00_0000 | SPI 16-47 |
| 12 | `create_fw_cfg()` | fw_cfg | 0x0902_0000 | - |
| 13 | `create_platform_bus()` | 平台总线 | 0x0C00_0000 | SPI 112+ |

### §35 全流程时序图

```
    main()          qemu_init()       machvirt_init()    vCPU Thread     Main Loop
     │                 │                   │                │               │
  1. ├── qemu_init ───►│                   │                │               │
     │                 │                   │                │               │
  2. │                 ├── 命令行解析       │                │               │
     │                 ├── 加速器初始化     │                │               │
     │                 ├── 后端创建         │                │               │
     │                 │   (chardev/block/net)              │               │
     │                 │                   │                │               │
  3. │                 ├── machine_run_board_init ──────────►│               │
     │                 │                   │                │               │
  4. │                 │                   ├── 地址映射计算  │               │
     │                 │                   ├── 固件加载      │               │
     │                 │                   │                │               │
  5. │                 │                   ├── CPU 创建循环  │               │
     │                 │                   │   object_new()  │               │
     │                 │                   │   qdev_realize()│               │
     │                 │                   │   └─ qemu_init_vcpu()          │
     │                 │                   │      └─ 创建线程 ├──►Thread Start
     │                 │                   │         (阻塞等待 cpu->created) │
     │                 │                   │                ├── created=true │
     │                 │                   │   ◄────────────┤   (阻塞等待   │
     │                 │                   │                │    vm_start)   │
     │                 │                   │                │               │
  6. │                 │                   ├── RAM 映射      │               │
     │                 │                   ├── create_gic()  │               │
     │                 │                   │   (MMIO+IRQ连线)│               │
     │                 │                   │                │               │
  7. │                 │                   ├── create_uart() │               │
     │                 │                   │   (MMIO+chardev)│               │
     │                 │                   │                │               │
  8. │                 │                   ├── create_virtio_devices()       │
     │                 │                   │   (32×virtio-mmio)             │
     │                 │                   │                │               │
  9. │                 │                   ├── arm_load_kernel()            │
     │                 │                   │   (加载 Guest)  │               │
     │                 │                   │                │               │
     │                 │   ◄───────────────┤ (返回)         │               │
     │                 │                   │                │               │
 10. │                 ├── vm_start()       │                │               │
     │                 │   resume_all_vcpus │                │               │
     │                 │   qemu_cpu_kick ──────────────────►│               │
     │                 │                   │                ├── 解除阻塞    │
     │                 │                   │                ├── cpu_exec()  │
     │                 │                   │                │   第一条 Guest │
     │                 │                   │                │   指令执行     │
     │                 │                   │                │               │
 11. │                 ├── bql_unlock()     │                │               │
     │                 │                   │                │               │
 12. ├── qemu_main_loop ──────────────────────────────────────────────────►│
     │                 │                   │                │   main_loop_  │
     │                 │                   │                │   wait() 循环 │
     │                 │                   │                │               │
```

---

## 附录

### 附录A：virt 完整地址映射表

| 组件 | 基址 | 大小 | 说明 |
|------|------|------|------|
| Flash | 0x0000_0000 | 128 MB | UEFI 固件 |
| GIC Dist | 0x0800_0000 | 64 KB | GICv3 Distributor |
| GIC CPU | 0x0801_0000 | 64 KB | GICv2 CPU Interface |
| GIC V2M | 0x0802_0000 | 4 KB | GICv2m MSI |
| GIC HYP | 0x0803_0000 | 64 KB | GIC Hypervisor |
| GIC VCPU | 0x0804_0000 | 64 KB | GIC Virtual CPU |
| GIC ITS | 0x0808_0000 | 128 KB | Interrupt Translation Service |
| GIC Redist | 0x080A_0000 | ~15 MB | Redistributors |
| UART0 | 0x0900_0000 | 4 KB | PL011 主串口 |
| RTC | 0x0901_0000 | 4 KB | PL031 实时时钟 |
| fw_cfg | 0x0902_0000 | 24 B | 固件配置接口 |
| GPIO | 0x0903_0000 | 4 KB | PL061 GPIO |
| UART1 | 0x0904_0000 | 4 KB | PL011 第二串口 |
| SMMU | 0x0905_0000 | 128 KB | IOMMU |
| ACPI GED | 0x0908_0000 | - | 电源管理 |
| PVTime | 0x090A_0000 | 64 KB | Stolen Time |
| virtio-mmio | 0x0A00_0000 | 32×512 B | virtio 传输层 |
| Platform Bus | 0x0C00_0000 | 32 MB | 平台设备 |
| Secure MEM | 0x0E00_0000 | 16 MB | TrustZone 内存 |
| PCIe MMIO | 0x1000_0000 | ~752 MB (0x2eff0000) | PCI 设备内存窗口 |
| PCIe PIO | 0x3EFF_0000 | 64 KB | PCI I/O 端口 |
| PCIe ECAM | 0x3F00_0000 | 16 MB | PCI 配置空间 |
| RAM | 0x4000_0000 | 可变 | Guest 系统内存 |

### 附录B：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `virt.c` | ~4,100 | ARM virt 机器定义、`machvirt_init()`、设备创建函数 |
| `vl.c` | ~3,800 | QEMU 主初始化、命令行解析、`qemu_init()` |
| `main.c` | ~96 | 入口函数 `main()` |
| `runstate.c` | ~960 | `qemu_main_loop()`、`vm_start()` |
| `cpus.c` | ~740 | vCPU 管理、`qemu_init_vcpu()`、BQL |
| `machine.c` | ~1,700 | Machine 基类、`machine_run_board_init()` |
| `qdev.c` | ~1,000 | 设备 realize 机制、`qdev_realize()` |
| `object.c` | ~2,200 | QOM 核心、`object_new()`、`type_initialize()` |
| `pl011.c` | ~700 | PL011 UART 设备模拟 |
| `virtio-mmio.c` | ~800 | virtio-mmio 传输层 |
| `virtio-blk.c` | ~1,800 | virtio-blk 块设备 |
| `virtio-net.c` | ~4,000 | virtio-net 网络设备 |
| `iothread.c` | ~210 | IOThread 实现 |
| `thread-pool.c` | ~300 | 块 I/O 线程池 |
| `tcg-accel-ops-mttcg.c` | ~140 | MTTCG vCPU 线程函数 |
| `tcg-accel-ops-rr.c` | ~220 | RR (单线程) vCPU 线程函数 |
| `kvm-accel-ops.c` | ~80 | KVM vCPU 线程函数 |

---

> **交叉引用**
> - QOM 对象模型详解 → [01-QOM对象模型深度分析](01-QOM对象模型深度分析.md)
> - 全局架构概览 → [00-全局架构概览](00-全局架构概览.md)
> - 执行循环与 MMIO 分发 → [02-模拟执行循环与MMIO分发深度分析](02-模拟执行循环与MMIO分发深度分析.md)
> - 设备模型与 virtio 框架 → [../device-model/00-设备模型与virtio深度分析](../device-model/00-设备模型与virtio深度分析.md)
> - UART/磁盘/网卡仿真详解 → [../device-model/01-关键设备仿真分析-UART-磁盘-网卡](../device-model/01-关键设备仿真分析-UART-磁盘-网卡.md)
> - 块层 I/O 路径 → [../device-model/02-块层IO路径深度分析](../device-model/02-块层IO路径深度分析.md)
> - 中断注入与处理 → [../arm64/04-中断注入与处理深度分析](../arm64/04-中断注入与处理深度分析.md)
