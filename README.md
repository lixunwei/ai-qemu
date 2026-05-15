# QEMU 11.0.50 ARM64 源码深度分析 — 总索引

> 本文档集基于 QEMU 11.0.50 源码，聚焦 ARM64 (AArch64) 架构  
> 使用 AI 辅助分析，所有源码引用均标注文件名:行号及关键 git commit SHA  
> 共 **42 篇文档**，总计 **~1777KB** 中文技术文档

---

## 快速导航

| 分类 | 文档数 | 总大小 | 核心主题 |
|------|--------|--------|---------|
| [architecture/](#architecture-架构) | 6 | ~194KB | 全局架构、QOM、执行循环、Machine 建立、线程模型、事件循环与I/O模型 |
| [arm64/](#arm64-arm64-架构) | 18 | ~618KB | CPU 模型、GICv3、TCG、ACPI、FDT、中断、特殊指令、EL 状态、TrustZone、虚拟化扩展、异常入口与返回、MMU/TLB、Generic Timer、PMU、CPU 特性与 ID 寄存器、SVE/SME、PAC/BTI/MTE、GCS/RME/新扩展 |
| [device-model/](#device-model-设备模型) | 7 | ~399KB | 设备框架、virtio、块层、chardev、VFIO、网络、DMA |
| [network/](#network-网络子系统) | 1 | ~48KB | 网络核心架构、TAP/SLIRP/Socket 后端、vhost-net、virtio-net 设备模型、收发路径 |
| [memory/](#memory-内存子系统) | 2 | ~57KB | MemoryRegion、MMIO、IOMMU、RAMBlock、脏页追踪、NUMA |
| [accel/](#accel-加速器) | 7 | ~250KB | TCG 翻译引擎全貌、优化递次、IR 与前端翻译、后端代码生成与 TB 管理、MTTCG 多线程翻译、Softmmu TLB 与内存访问、TCG Plugin 系统 |
| [debug/](#debug-调试) | 1 | ~49KB | GDB 协议、断点、ARM64 寄存器映射 |

---

## architecture/ 架构

### [00-全局架构概览.md](architecture/00-全局架构概览.md)
> **19KB · 12 节**

QEMU 整体架构鸟瞰图。从 `main()` 入口到 `qemu_main_loop()`，覆盖 QOM 对象模型概述、内存模型概述、设备模型概述、加速器抽象（TCG/KVM）、事件循环与 I/O 模型、ARM virt 机器类型、构建系统与目录结构。

**适合读者**：初次接触 QEMU 源码的开发者。  
**关键源文件**：`system/main.c`、`system/vl.c`、`hw/arm/virt.c`

---

### [01-QOM对象模型深度分析.md](architecture/01-QOM对象模型深度分析.md)
> **26KB · 12 节**

QEMU 对象模型 (QOM) 的类型系统、实例化、继承、接口、属性机制。详解 `TypeImpl` 内部结构（`object.c:48-75`）、延迟类初始化（`type_initialize()`）、`__attribute__((constructor))` 自动注册、设备属性 getter/setter。

**适合读者**：需要添加新设备类型或修改 QOM 框架的开发者。  
**关键源文件**：`qom/object.c`、`include/qom/object.h`

---

### [02-模拟执行循环与MMIO分发深度分析.md](architecture/02-模拟执行循环与MMIO分发深度分析.md)
> **47KB · 24 节 + 3 附录**

QEMU 主事件循环（`qemu_main_loop`）、CPU 执行循环（`cpu_exec`）、MMIO 分发路径（TCG TLB miss → `memory_region_dispatch_read/write`）、KVM MMIO exit 处理、设备 I/O 完成回调、AioContext 与 BH 调度。

**适合读者**：调试 I/O 性能、理解 MMIO 地址翻译的开发者。  
**关键源文件**：`system/cpus.c`、`accel/tcg/cpu-exec.c`、`system/physmem.c`

---

### [03-Machine建立流程深度分析.md](architecture/03-Machine建立流程深度分析.md)
> **45KB · 35 节 + 2 附录**

ARM virt 机器从 `machvirt_init()` 到所有设备就绪的完整流程。CPU 创建与拓扑、内存布局（RAM/Flash/MMIO/PCI 地址空间）、中断控制器初始化、PCI 主桥创建、virtio 设备挂载、ACPI/FDT 生成、vCPU 线程创建与启动。

**适合读者**：需要修改机器类型或添加新外设的开发者。  
**关键源文件**：`hw/arm/virt.c`、`hw/intc/arm_gicv3*.c`、`hw/pci-host/gpex.c`

---

### [04-线程模型与锁机制深度分析.md](architecture/04-线程模型与锁机制深度分析.md)
> **27KB · 20 节 + 3 附录**

QEMU 完整多线程并发架构。BQL（Big QEMU Lock）定义与保护范围、主事件循环与 AioContext、vCPU 线程模型（MTTCG/KVM）、IOThread 独立事件循环、RCU 用户态实现（grace period/call_rcu 线程）、协程机制（ucontext 后端/两级池化/CoMutex/CoQueue）、Bottom Half 延迟回调。

**适合读者**：调试并发问题或优化 I/O 性能的开发者。  
**关键源文件**：`system/cpus.c`、`util/main-loop.c`、`util/rcu.c`、`util/qemu-coroutine.c`

---

### [05-事件循环与IO模型深度分析.md](architecture/05-事件循环与IO模型深度分析.md)
> **33KB · 27 节 + 3 附录**

AioContext 核心数据结构与生命周期、主循环（`main_loop_wait` + GMainContext 集成）、`aio_poll` 核心循环、三级 FD 监控后端（ppoll → epoll → io_uring 逐级升级）、自适应轮询机制（poll_ns/poll_max_ns 动态调整）、Bottom Half 延迟执行、定时器系统（4 种时钟类型）、EventNotifier 跨线程唤醒、IOThread 子系统、协程核心架构（ucontext/池化/CoMutex/CoQueue）、协程与事件循环集成（aio_co_schedule/aio_co_wake）、块层异步 I/O 路径（Linux AIO/io_uring/线程池）。

**适合读者**：需要深入理解 QEMU 事件驱动架构、优化 I/O 延迟或开发新异步后端的开发者。  
**关键源文件**：`util/async.c`、`util/aio-posix.c`、`util/main-loop.c`、`iothread.c`、`util/qemu-coroutine.c`、`block/linux-aio.c`

---

## arm64/ ARM64 架构

### [00-ARM64-CPU-GICv3-TCG深度分析.md](arm64/00-ARM64-CPU-GICv3-TCG深度分析.md)
> **29KB · 20 节**

ARM64 CPU 模型（ARMCPU 结构体、CPUARMState、isar feature 检测）、GICv3 中断控制器概述、TCG 翻译概述。三个子系统的协作关系与数据流。

**适合读者**：ARM64 CPU 模拟入门。  
**关键源文件**：`target/arm/cpu.h`、`target/arm/cpu64.c`、`hw/intc/arm_gicv3.c`

---

### [01-ACPI表生成与启动流程深度分析.md](arm64/01-ACPI表生成与启动流程深度分析.md)
> **41KB · 24 节**

ARM64 ACPI 表构建（DSDT/MADT/GTDT/SPCR/SRAT/PPTT/IORT 等）、UEFI 启动流程（EDK2 固件加载、DTB→ACPI 过渡）、SMBIOS 表生成、Secure/Non-secure 世界初始化。

**适合读者**：调试 ACPI 表问题或 UEFI 启动的开发者。  
**关键源文件**：`hw/arm/virt-acpi-build.c`、`hw/acpi/aml-build.c`

---

### [02-特殊指令模拟深度分析.md](arm64/02-特殊指令模拟深度分析.md)
> **36KB · 21 节 + 3 附录**

ARM64 原子/独占指令（LDXR/STXR/CAS/SWP）、内存屏障（DMB/DSB/ISB）、TLB 操作（TLBI）、Cache 维护（DC/IC）、页表遍历与 MMU 模拟。TCG 对这些指令的翻译策略与 helper 实现。

**适合读者**：分析内核同步原语或 MMU 相关问题的开发者。  
**关键源文件**：`target/arm/tcg/translate-a64.c`、`target/arm/tcg/helper-a64.c`、`target/arm/ptw.c`

---

### [03-GICv3中断控制器模拟架构深度分析.md](arm64/03-GICv3中断控制器模拟架构深度分析.md)
> **51KB · 32 节 + 3 附录**

GICv3 完整架构：Distributor（SPI 路由）、Redistributor（SGI/PPI/LPI）、CPU Interface（系统寄存器访问）、ITS（MSI/LPI 翻译）。KVM 内核加速（vGIC）与 TCG 模拟对比。

**适合读者**：调试中断路由或修改 GICv3 实现的开发者。  
**关键源文件**：`hw/intc/arm_gicv3_dist.c`、`hw/intc/arm_gicv3_redist.c`、`hw/intc/arm_gicv3_cpuif.c`

---

### [04-中断注入与处理深度分析.md](arm64/04-中断注入与处理深度分析.md)
> **57KB · 35 节 + 3 附录**

从设备中断触发到 Guest OS ISR 执行的完整路径。GICv3 中断仲裁、优先级过滤、ICC 寄存器操作、CPU 异常入口（`arm_cpu_do_interrupt`）、KVM IRQ 注入（`KVM_IRQ_LINE`）、vGIC 内核路径。

**适合读者**：调试中断延迟或中断丢失的开发者。  
**关键源文件**：`target/arm/helper.c`、`target/arm/kvm.c`、`hw/intc/arm_gicv3_kvm.c`

---

### [05-FDT设备树深度分析.md](arm64/05-FDT设备树深度分析.md)
> **49KB · 32 节 + 3 附录**

设备树 (FDT) 生成全流程：libfdt API、ARM virt 各设备节点创建（CPU/内存/GICv3/Timer/UART/PCI/Flash）、chosen 节点参数传递、动态内存布局、FDT 与 ACPI 的选择逻辑。

**适合读者**：调试 Linux 内核设备树解析或修改设备树生成的开发者。  
**关键源文件**：`hw/arm/virt.c`（`virt_machine_done()`）

---

### [06-异常级别状态管理深度分析.md](arm64/06-异常级别状态管理深度分析.md)
> **42KB · 21 节 + 3 附录**

ARM64 异常级别 (EL0-EL3) 的状态跟踪与切换。PSTATE 分拆存储、`arm_current_el()`/`arm_el_is_aa64()` 核心函数、安全状态（Root/Secure/NonSecure/Realm）× EL 交互矩阵、各 EL 执行环境对比（翻译体制/特权指令/陷入控制）。异常入口的 PSTATE→SPSR 保存与 PSTATE 修改（DAIF/PAN/TCO/SSBS/ALLINT）、ERET 返回的合法性检查与 SP 恢复、SVC/HVC/SMC 路由逻辑、WFI/WFE trap 控制、系统寄存器 EL 访问矩阵、MMU index 与 EL 映射、SVE 向量长度在 EL 切换时的收窄、EL 变化钩子（PMU/Timer/SVE）。包含系统调用/虚拟机退出/安全监控调用三种典型场景分析。

**适合读者**：分析异常/中断处理流程或 Hypervisor 切换的开发者。  
**关键源文件**：`target/arm/internals.h`、`target/arm/helper.c`、`target/arm/tcg/helper-a64.c`

---

### [07-不同EL下指令执行差异深度分析.md](arm64/07-不同EL下指令执行差异深度分析.md)
> **34KB · 25 节 + 3 附录**

CPU 在不同 EL (EL0-EL3) 下可执行指令的差异。WFI/WFE 三层 trap（SCTLR→HCR→SCR）、SVC/HVC/SMC 路由与 disable 控制、FP/SIMD/SVE/SME 多层 trap（CPACR→CPTR_EL2→CPTR_EL3）、ERET 的 EL 限制与 FGT trap、系统寄存器 CPAccessRights 位矩阵与 `cp_access_ok()` 检查、MSR/MRS 完整访问控制流程（TIDCP→查找→静态→VHE→动态→FGT）、Cache/TLBI/AT/DC ZVA 的 EL 访问控制、调试寄存器四类 trap（TDOSA/TDRA/TDA/TDCC）、TB flags 预计算优化。

**适合读者**：分析指令 trap 机制或实现 Hypervisor 细粒度控制的开发者。  
**关键源文件**：`target/arm/cpregs.h`、`target/arm/tcg/translate-a64.c`、`target/arm/tcg/op_helper.c`

---

### [08-TrustZone安全扩展与Secure-World深度分析.md](arm64/08-TrustZone安全扩展与Secure-World深度分析.md)
> **24KB · 24 节 + 3 附录**

ARM TrustZone 安全扩展的 QEMU 实现。ARMSecuritySpace 四状态模型（Secure/NonSecure/Root/Realm）、SCR_EL3 安全配置寄存器写入副作用、CPU 安全复位（EL3 启动 + RVBAR）、`arm_emulate_firmware_reset()` 固件模拟、双地址空间架构（ARMASIdx_S/NS + secure_sysmem overlay）、安全内存布局（Flash/RAM/UART/GPIO）、MMU index 安全映射、SMC 预检查与路由（SMD/TSC/PSCI）、PSCI v1.1 内置实现、世界切换机制（SCR_EL3.NS 翻转 + TLB 刷新）、寄存器 Bank 安全/非安全副本、GIC 安全分组（Group 0/1）、MPC/PPC 保护控制器。

**适合读者**：分析安全启动、TrustZone 隔离或 EL3 固件交互的开发者。  
**关键源文件**：`include/hw/arm/arm-security.h`、`target/arm/helper.c`、`target/arm/cpu.c`、`hw/arm/virt.c`

---

### [09-虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU.md](arm64/09-虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU.md)
> **37KB · 30 节 + 3 附录**

ARM64 虚拟化扩展的 QEMU 完整实现。HCR_EL2 寄存器 60+ 位域定义与写入副作用（TLB 刷新 + 虚拟中断更新）、有效 HCR 计算（`arm_hcr_el2_eff_secstate()` 含 TGE/E2H 副作用）、VHE（E2H+TGE）寄存器重定向机制（`_EL12` 别名创建）、`ELIsInHost()` 判定、HCR trap 位分类与检查函数映射、中断虚拟化（FMO/IMO/AMO 路由 + VI/VF/VSE 注入）、MMU Index 与翻译体制（E10/E20/E2/Stage2）、Stage-2 MMU 完整实现（VTTBR_EL2/VTCR_EL2、页表遍历、`get_S2prot()` 权限、FWB 属性、S2PIR 间接权限）、Stage-2 TLB 管理与故障处理（HPFAR_EL2/ESR_EL2）、异常路由与 EL2 入口、嵌套虚拟化（NV/NV1/NV2 + VNCR_EL2）、虚拟定时器、KVM 与 EL2 交互。

**适合读者**：分析 Hypervisor 行为、Stage-2 页表或虚拟化性能的开发者。  
**关键源文件**：`target/arm/cpu.h`、`target/arm/helper.c`、`target/arm/ptw.c`、`target/arm/cpu-irq.c`

### [10-异常入口与返回深度分析.md](arm64/10-异常入口与返回深度分析.md)
> **39KB · 27 节**

ARM64 异常入口与返回的 QEMU 完整实现。异常触发路径（`raise_exception()` / `raise_exception_ra()`）、目标 EL 确定（`exception_target_el()` / `arm_phys_excp_target_el()` 6 维路由表）、同步异常翻译（SVC/HVC/SMC/BRK + 系统寄存器陷入 + 内存故障 + PC/SP 对齐）、异步中断分发（`arm_cpu_exec_interrupt()` 优先级 NMI→FIQ→IRQ→VIRQ→VFIQ→VSERR）、调试异常（MDCR 路由）。异常入口总控（`arm_cpu_do_interrupt()` PSCI/Semihost 拦截 + AArch64/AArch32 分派）、向量表完整布局（16 向量 = 4来源×4类型）、ESR 综合征构建与 AArch32→AArch64 转换、FAR_ELx 写入、PSTATE→SPSR 保存（含 `pstate_read()/pstate_write()` 缓存字段拆解）、ELR 保存、SP 保存/恢复、新 PSTATE 组装（DAIF/PAN/TCO/SSBS/ALLINT）。ERET 翻译（FGT trap 优先 + PAuth 认证）、ERET Helper 完整流程（7 种非法返回条件 + SPSR→PSTATE 恢复 + TBI PC 处理）、AArch32↔AArch64 状态切换、SVE/SME 向量长度变更、FEAT_NV/NV2 SPSR 修改、FEAT_DoubleFault SError 重定向、GCS EXLOCK 集成。

**适合读者**：分析异常处理、中断路由或 EL 切换行为的开发者。  
**关键源文件**：`target/arm/helper.c`、`target/arm/tcg/helper-a64.c`、`target/arm/tcg/op_helper.c`、`target/arm/cpu-irq.c`、`target/arm/tcg/translate-a64.c`

### [11-MMU-TLB深度分析.md](arm64/11-MMU-TLB深度分析.md)
> **32KB · 32 节**

ARM64 内存管理单元完整实现分析。ARMMMUIdx 翻译体制索引全量枚举（22 种有 TLB 索引 + 5 种无 TLB 索引，含 PAN/GCS 变体）、体制选择逻辑（`arm_mmu_idx_el()` 含 VHE/PAN/AArch32 分支）、翻译禁用判定（HCR.DC/TGE/VM 联动）。LPAE 页表遍历核心函数 `get_phys_addr_lpae()` 完整 590 行解析（TCR 参数提取 `aa64_va_parameters()`、TTBR 选择、起始级别计算、描述符解析循环、Block/Table/Page 处理）。AF/Dirty Bit 硬件辅助管理（HA/HD）。Stage-1 权限检查（AP/PAN/PAN3/WXN/XN/PXN + 安全状态交叉检查）、Stage-2 权限检查（S2AP + FEAT_XNX 4 值 XN）、间接权限（FEAT_S1PIR/S2PIR 16 种编码查表）。内存属性（MAIR AttrIndx + Device 检测 + FWB 覆盖）。HCR_EL2 控制位（VM/DC/PTW/FWB/TGE）。FEAT_LPA/LPA2 52 位物理地址（TTBR/描述符 OA 扩展）。AT 指令与 PAR_EL1。故障类型全量枚举（`ARMFaultType` + `ARMMMUFaultInfo`）与分发（`arm_deliver_fault()` Stage-2 强制 EL2）。QEMU 软件 TLB 架构（直接映射 + victim cache + CPUTLBEntry 标志）、TLB 插入（`tlb_set_page_full()`）、TLBI 指令实现（IS 广播 + HCR.FB 升级 + FEAT_TLBIRANGE）、ASID/VMID 管理（全刷新策略）、多核一致性。地址空间分区（ARMASIdx 4 种）。

**适合读者**：分析 MMU 行为、页表遍历、TLB 管理或内存权限的开发者。  
**关键源文件**：`target/arm/ptw.c`、`target/arm/helper.c`、`target/arm/mmuidx.h`、`target/arm/tcg/tlb-insns.c`、`accel/tcg/cputlb.c`

### [12-Generic-Timer定时器深度分析.md](arm64/12-Generic-Timer定时器深度分析.md)
> **34KB · 31 节**

ARM64 Generic Timer 完整实现分析。7 种定时器类型全量枚举（GTIMER_PHYS/VIRT/HYP/SEC/HYPVIRT/S_EL2_PHYS/S_EL2_VIRT）。计数器系统基于 `QEMU_CLOCK_VIRTUAL`，频率可配置（ARMv8.6+ 默认 1 GHz，兼容模式 62.5 MHz）。核心重算逻辑 `gt_recalc_timer()` 完整解析：ISTATUS 计算、QEMUTimer 重编程、中断状态更新。偏移体系三层设计（raw/indirect/direct），CNTVOFF_EL2 虚拟偏移、CNTPOFF_EL2 物理偏移（FEAT_ECV）。VHE 重定向（物理→HYP、虚拟→HYPVIRT）。系统寄存器接口全量 `generic_timer_cp_reginfo[]` 数组。访问控制：CNTKCTL_EL1（EL0→EL1 陷入）、CNTHCTL_EL2（EL1→EL2 陷入、ECV 扩展陷入位）。EL3 安全定时器访问控制（SCR.ST）。GIC PPI 连线（7 条 GPIO→PPI 映射）。FDT 设备树描述（arm,armv8-timer 兼容）。KVM 定时器集成（虚拟时间同步、迁移钩子）。RME CNTVMASK/CNTPMASK 屏蔽。定时器与 WFI 唤醒机制。

**适合读者**：分析定时器中断调度、虚拟化时间管理或 KVM 定时器同步的开发者。  
**关键源文件**：`target/arm/helper.c`、`target/arm/gtimer.h`、`target/arm/cpu.c`、`target/arm/cpu.h`、`hw/arm/virt.c`、`include/hw/arm/bsa.h`

### [13-PMU性能监控单元深度分析.md](arm64/13-PMU性能监控单元深度分析.md)
> **23KB · 30 节**

ARM64 PMU 性能监控单元完整实现分析。差值快照计数模型（`pmu_op_start/finish` 围绕每次访问刷新）。支持事件：CPU_CYCLES（虚拟时钟 1GHz）、INST_RETIRED（需 icount 精确模式）、SW_INCR（软件递增）、STALL_*（恒零占位）。计数器使能五层判定：PMU 特性 → 使能位（PMCR.E/HPME） → 禁止位（HPMD/SPME/SCCD） → 事件过滤器（P/U/NSK/NSU/NSH/M） → 事件支持。溢出检测算法（高位翻转检测）。`pmu_timer` 定时器预测溢出触发中断。PMCR_EL0 完整位域（E/P/C/D/DP/LC/LP/N）。HPMN 计数器分区（Guest 组 vs EL2 组）。64 位计数器扩展（PMCR.LC/LP + MDCR.HLP）。访问控制：PMUSERENR_EL0 细粒度位（EN/SW/CR/ER）+ MDCR_EL2.TPM/TPMCR + MDCR_EL3.TPM 三层陷入。GIC PPI 23 连线。KVM vPMU 集成（PMU_V3_INIT/IRQ）。EL 变化钩子。

**适合读者**：分析性能计数器实现、PMU 中断或虚拟化 PMU 分区的开发者。  
**关键源文件**：`target/arm/cpregs-pmu.c`、`target/arm/cpu.h`、`target/arm/internals.h`、`hw/arm/virt.c`、`target/arm/kvm.c`

---

### [14-CPU特性与ID寄存器仿真深度分析.md](arm64/14-CPU特性与ID寄存器仿真深度分析.md)
> **44KB · 32 节 + 3 附录**

ARM64 CPU 特性与 ID 寄存器仿真完整分析。ARMISARegisters.idregs[] 统一存储架构、FIELD_EX64_IDREG/SET_IDREG/GET_IDREG 宏体系。全部 ID_AA64* 寄存器字段定义（PFR0/1、ISAR0/1/2、MMFR0/1/2/3/4、DFR0/1、ZFR0、SMFR0）。168 个 isar_feature_aa64_*() 特性检查函数及映射表。14 种 CPU 模型定义（A35→Neoverse-N2）及 initfn 详解。-cpu max 双路径初始化（TCG: A57+全扩展叠加, KVM: host 透传）。QOM 属性系统（sve/sme/pauth/lpa2 等开关如何修改 ID 寄存器字段）。特性终化流程（finalize_features→SVE/SME/PAuth/LPA2 验证）。Guest MRS 读取路径（ARMCPRegInfo→access_tid3→GET_IDREG）。KVM 宿主特性发现（scratch vCPU + KVM_GET_ONE_REG）。翻译时检查（dc_isar_feature→TRANS_FEAT 宏）。TB Flags 特性编码（rebuild_hflags_a64 中 SVE/SME/PAuth/BTI/MTE 状态缓存）。FEAT_LSE/PAuth/MTE/BTI 四条完整实现追踪（ID 寄存器→特性检查→翻译门控→Guest 感知）。旧 ARM_FEATURE_* 位图系统与新 isar_feature 系统的共存与迁移。

**适合读者**：分析 CPU 模型差异、特性检测机制、ID 寄存器仿真或 KVM 特性透传的开发者。  
**关键源文件**：`target/arm/cpu.h`、`target/arm/cpu-features.h`、`target/arm/cpu-sysregs.h.inc`、`target/arm/cpu64.c`、`target/arm/tcg/cpu64.c`、`target/arm/cpu.c`、`target/arm/helper.c`、`target/arm/kvm.c`、`target/arm/tcg/hflags.c`

---

### [15-SVE-SME可扩展向量扩展深度分析.md](arm64/15-SVE-SME可扩展向量扩展深度分析.md)
> **18KB · 15 节**

ARM64 SVE（Scalable Vector Extension）和 SME（Scalable Matrix Extension）完整实现分析。Z 寄存器（32×最大 2048 位）、P 谓词寄存器（16+FFR）存储布局。向量长度不可知（VLA）编程模型：ZCR_EL1/2/3 层级控制有效 VL，高 EL 可限制低 EL 的 VL。VL 缩小时 `aarch64_sve_narrow_vq()` 截断高位。三层陷入控制：CPACR.ZEN→CPTR_EL2.TZ→CPTR_EL3.EZ。SME Streaming SVE 模式（PSTATE.SM）：进入/退出清零所有 Z/P/FFR。ZA 矩阵存储（256×256 字节，Tile 行交错映射）。SVL 独立于 SVE VL（SMCR vs ZCR）。FA64 允许 Streaming 模式下执行全部 A64 指令。SVE 指令翻译基于 GVec 框架（`tcg_gen_gvec_*` + out-of-line helper）。SVE2 扩展（AES/PMULL/BitPerm/SHA3/SM4）。SME2 ZT0 寄存器（512 位）。EL 切换时 SVE 状态处理与 VL 调整。

**适合读者**：分析 SVE/SME 指令仿真、向量长度管理、Streaming 模式切换或 SIMD 性能优化的开发者。  
**关键源文件**：`target/arm/cpu.h`、`target/arm/helper.c`、`target/arm/cpu64.c`、`target/arm/tcg/translate-sve.c`、`target/arm/tcg/sve_helper.c`、`target/arm/tcg/translate-sme.c`、`target/arm/tcg/hflags.c`

### [16-PAC-BTI-MTE安全特性深度分析.md](arm64/16-PAC-BTI-MTE安全特性深度分析.md)
> **26KB · 19 节**

ARM64 三大硬件安全特性完整实现分析。**PAC（指针认证码）**：5 组 128-bit 密钥（APIA/APIB/APDA/APDB/APGA）存储、QARMA5/QARMA3/IMPDEF 三种算法选择、`pauth_addpac()` 签名与 `pauth_auth()` 认证流程、SCTLR.EnIA/EnIB/EnDA/EnDB 独立使能、HCR.API/SCR.API 两级陷入控制、认证分支指令（BRAA/BLRAA/RETAA/ERETAA）combined 路径、FEAT_FPAC/FPACCOMBINE 直接异常 vs 基础版指针破坏。**BTI（分支目标识别）**：PSTATE.BTYPE 4 值状态机、BR/BLR 分别设置 BTYPE=1/2/3、`btype_destination_ok()` 着陆点兼容性矩阵（BTI/BTI c/BTI j/BTI jc/PACIASP）、GP 保护页属性与页表第 50 位、SCTLR.BT0/BT1 强化模式、两阶段检查（翻译时静态 + 运行时页检查）、EC_BTITRAP=0x0d 异常。**MTE（内存标签扩展）**：4-bit 逻辑标签（地址 [59:56]）与 4-bit 分配标签（每 16 字节粒度）、用户模式 `page_get_target_data()` vs 系统模式独立标签地址空间、`mte_probe_int()` 逐粒度比较、SCTLR.TCF/TCF0 四种故障模式（忽略/同步/异步/非对称）、PSTATE.TCO 覆盖、IRG LFSR 随机标签生成与 GCR_EL1 排除掩码、MTEDESC 紧凑描述符、FEAT_MTE/MTE2/MTE3 三级特性。三大特性的 TB 标志集成（PAUTH_ACTIVE/BT/BTYPE/MTE_ACTIVE/ATA）。

**适合读者**：分析 ARM64 安全特性仿真、控制流/数据流完整性保护、内存安全机制实现的开发者。  
**关键源文件**：`target/arm/tcg/pauth_helper.c`、`target/arm/tcg/mte_helper.c`、`target/arm/tcg/translate-a64.c`、`target/arm/tcg/helper-a64.c`、`target/arm/tcg/hflags.c`、`target/arm/cpu.h`、`target/arm/internals.h`、`target/arm/cpu-features.h`

### [17-GCS-RME及新扩展深度分析.md](arm64/17-GCS-RME及新扩展深度分析.md)
> **22KB · 19 节**

ARM64 GCS/RME 及 ARMv9 新扩展完整实现分析。**GCS（守护控制栈）**：`GCSPR_EL[0-3]` 影子栈指针与 `GCSCR_EL[0-3]` 控制、BL/BLR 自动推入返回地址（`gen_add_gcs_record()`）、RET 弹出并比较（`gen_load_check_gcs_record()` + RVCHKEN）、GCSPUSHM/GCSPOPM/GCSPUSHX/GCSPOPCX 指令集、PSTATE.EXLOCK 异常锁定状态机、GCS 专用 MMU 索引与页表写保护、多层使能控制（GCSCR.PCRSEL→HCRX.GCSEN→SCR.GCSEN）、GCSSTR 独立陷入控制。**RME（领域管理扩展）**：ARMSecuritySpace 四安全状态（Secure/NonSecure/Root/Realm）、SCR_EL3.NS/NSE 2-bit 编码矩阵、`arm_granule_protection_check()` GPT 两级遍历（L0 block/table + L1 contiguous/granule）、GPI 值解析（16 种粒度保护标识）、4 种 GPC 故障类型（Walk/Fail/EABT/AddressSize）、Root 不可禁用保证、MFAR_EL3 故障地址记录、MECID 寄存器、Phys_Root/Phys_Realm MMU 索引。**其他新扩展**：FEAT_NMI（PSTATE.ALLINT + SCTLR.SPINTMASK + VINMI/VFNMI）、FEAT_S1PIE/S1POE（PIR_EL1/PIRE0_EL1 权限间接 16 条目索引表）、FEAT_HAFDBS（硬件 AF 自动设置 + DBM 脏位管理）、FEAT_AIE（MAIR2/AMAIR2 属性扩展）、MPAM 存根状态。

**适合读者**：分析 ARM CCA 机密计算、影子栈保护、RME 安全域隔离、ARMv9 新特性仿真实现的开发者。  
**关键源文件**：`target/arm/cpregs-gcs.c`、`target/arm/tcg/translate-a64.c`、`target/arm/ptw.c`、`target/arm/helper.c`、`target/arm/tcg/hflags.c`、`target/arm/mmuidx.h`、`include/hw/arm/arm-security.h`、`target/arm/cpu-irq.c`

---

## device-model/ 设备模型

### [00-设备模型与virtio深度分析.md](device-model/00-设备模型与virtio深度分析.md)
> **47KB · 24 节**

QEMU 设备模型框架（DeviceState/BusState/DeviceClass）、QOM 设备生命周期（realize/unrealize）、SysBus 与 PCI 总线模型、virtio 框架（VirtIODevice/VirtQueue/vring）、MMIO 与 PCI 传输层、ARM virt 设备拓扑。

**适合读者**：开发新 QEMU 设备的开发者。  
**关键源文件**：`hw/core/qdev.c`、`hw/virtio/virtio.c`、`hw/virtio/virtio-pci.c`

---

### [02-块层IO路径深度分析.md](device-model/02-块层IO路径深度分析.md)
> **28KB · 14 节 + 4 附录**

从 Guest I/O 请求到宿主文件系统的完整数据路径：BlockBackend → BlockDriverState → BDS 图 → BlockDriver 接口。读/写/Flush/Discard I/O 路径详解、协程异步 I/O 模型（blk_aio→协程桥接、AioContext 事件循环、线程池 AIO、io_uring 集成）。qcow2 格式驱动/Block 作业/限流等详见 doc 04。

**适合读者**：调试磁盘 I/O 性能或理解块层数据流的开发者。  
**关键源文件**：`block/block-backend.c`、`block/io.c`、`block/file-posix.c`、`util/thread-pool.c`

---

### [03-Chardev子系统与UART交互深度分析.md](device-model/03-Chardev子系统与UART交互深度分析.md)
> **52KB · 31 节 + 3 附录**

字符设备框架（Chardev/CharFrontend/CharBackend）、各后端实现（stdio/socket/pty/file/mux/ringbuf）、PL011 UART 与 chardev 的数据流（Guest 输出 → UART TX → chardev → 终端）。

**适合读者**：调试串口输出或开发字符设备后端的开发者。  
**关键源文件**：`chardev/char.c`、`chardev/char-socket.c`、`hw/char/pl011.c`

---

### [04-Block设备子系统深度分析.md](device-model/04-Block设备子系统深度分析.md)
> **62KB · 38 节 + 3 附录**

块设备创建与生命周期（-drive/-blockdev）、qcow2 格式内部（L1/L2 映射表、refcount、快照、L2/Refcount 缓存、压缩、加密、并行读优化）、QCOW_OFLAG_COPIED 语义与 COW 触发条件、块任务框架（mirror/commit/stream/backup）、增量备份 (dirty bitmap)、I/O 限速（Throttle）、块过滤器。新增：virtio-blk 设备仿真（VirtIOBlock 结构体、请求处理管线、请求类型、I/O 完成路径、多队列/IOThread、特性协商）。

**适合读者**：深入理解 qcow2 格式或使用高级块功能（快照/备份/迁移）的开发者。  
**关键源文件**：`block/qcow2.c`、`block/qcow2-cluster.c`、`block/mirror.c`、`block/throttle-groups.c`

---

### [05-VFIO设备直通与IOMMU集成深度分析.md](device-model/05-VFIO设备直通与IOMMU集成深度分析.md)
> **42KB · 24 节 + 3 附录**

VFIO 框架（container/group/device 模型）、PCI 设备直通（BAR 映射、中断重映射）、IOMMU 集成（SMMUv3 模拟、IOMMUFD 新框架）、DMA 映射与页表同步。

**适合读者**：配置或调试 PCI 直通的开发者。  
**关键源文件**：`hw/vfio/pci.c`、`hw/vfio/common.c`、`hw/arm/smmuv3.c`

---

### [06-网络后端深度分析-TAP-vhost-net-vhost-user.md](device-model/06-网络后端深度分析-TAP-vhost-net-vhost-user.md)
> **76KB · 37 节 + 3 附录**

网络后端全景：TAP 设备（多队列、vnet_hdr）、vhost-net 内核旁路、vhost-user 用户态旁路（DPDK 集成）、vDPA（硬件 vring）、slirp 用户态网络、socket 后端、Hub 虚拟交换、NetFilter 框架。新增：virtio-net 设备仿真（VirtIONet 结构体、RX/TX 路径、控制队列、RSS、多队列、特性协商、vhost 集成）。

**适合读者**：优化网络 I/O 或集成网络后端的开发者。  
**关键源文件**：`net/tap.c`、`hw/virtio/vhost-net.c`、`hw/virtio/vhost-user.c`

---

### [07-DMA设备模拟架构深度分析.md](device-model/07-DMA设备模拟架构深度分析.md)
> **55KB · 23 节 + 3 附录**

DMA 核心 API（`dma_memory_read/write/map`）、QEMUSGList 散列聚合、PCI DMA（bus_master_as、BME 门控）、SysBus DMA、virtio DMA（`VIRTIO_F_ACCESS_PLATFORM`）、bounce buffer、IOMMU 透明翻译、PL330/PL080 DMA 控制器、e1000/AHCI/virtio-blk DMA 实例。

**适合读者**：理解设备 DMA 路径或开发 DMA 密集型设备的开发者。  
**关键源文件**：`include/system/dma.h`、`system/dma-helpers.c`、`hw/dma/pl330.c`

---

## network/ 网络子系统

### [00-网络子系统深度分析.md](network/00-网络子系统深度分析.md)
> **48KB · 29 节**

网络子系统完整分析：核心架构（NetClientState/NetClientInfo/NICState、peer 对等连接）、数据包收发路径（TX/RX 全链路追踪）、NetQueue 队列层、NetFilter 过滤器框架、TAP 后端（fd 事件驱动、vnet_hdr）、SLIRP 用户态 NAT（DHCP/DNS/端口转发）、Socket 后端（UDP/TCP/Multicast）、vhost-net 内核加速（eventfd 转移）、virtio-net 设备模型（VirtIONet 结构、特性协商、flush_tx/receive_rcu、offloading、MAC 过滤、多队列、RSS）。

**适合读者**：网络虚拟化开发者、需要理解 QEMU 网络 I/O 路径的工程师。
**关键源文件**：`include/net/net.h`、`net/net.c`、`net/tap.c`、`net/slirp.c`、`hw/net/virtio-net.c`、`hw/net/vhost_net.c`

---

## memory/ 内存子系统

### [00-内存子系统深度分析.md](memory/00-内存子系统深度分析.md)
> **30KB · 16 节**

MemoryRegion 树、FlatView 扁平化、AddressSpace、RCU 无锁分发、MMIO dispatch、RAM 映射（mmap）、IOMMU MemoryRegion、内存事务属性、内存监听器（MemoryListener）。

**适合读者**：理解 QEMU 内存模型或调试地址空间问题的开发者。  
**关键源文件**：`system/memory.c`、`system/physmem.c`、`include/system/memory.h`

---

### [01-RAM管理与脏页追踪深度分析.md](memory/01-RAM管理与脏页追踪深度分析.md)
> **27KB · 21 节 + 4 附录**

RAMBlock 管理与 Guest RAM 分配（mmap/MAP_SHARED/MAP_PRIVATE）、Huge Pages（hugetlbfs/memfd）、内存后端 QOM（HostMemoryBackendFile/Memfd/Ram）、RAM 预分配。三客户端脏页位图（VGA/CODE/MIGRATION）、TCG TLB_NOTDIRTY 机制、KVM Dirty Log（KVM_GET_DIRTY_LOG/KVM_CLEAR_DIRTY_LOG）、KVM Dirty Ring（per-vCPU 环形缓冲区 + reaper 线程）。迁移 RAM 保存（precopy 脏页迭代、clear_bmap 优化）。NUMA 拓扑配置、PC-DIMM/NVDIMM 热插拔、virtio-mem 动态内存（RamDiscardManager）、ARM virt 内存布局。

**适合读者**：研究 VM 迁移脏页追踪、内存性能调优（大页/NUMA）或内存热插拔的开发者。  
**关键源文件**：`system/physmem.c`、`accel/kvm/kvm-all.c`、`migration/ram.c`、`include/system/ramblock.h`

---

## accel/ 加速器

### [00-TCG深度分析.md](accel/00-TCG深度分析.md)
> **65KB · 30 节**

TCG (Tiny Code Generator) 完整分析：IR 中间表示（TCGOp/TCGTemp/TCGLabel）、前端翻译（ARM64 指令 → TCG IR）、优化遍（常量折叠/死代码消除/活跃性分析）、后端代码生成（TCG IR → 宿主机器码）、Translation Block 缓存、链式执行、热路径分析。

**适合读者**：研究二进制翻译技术或修改 TCG 的开发者。  
**关键源文件**：`tcg/tcg.c`、`tcg/tcg-op.c`、`tcg/optimize.c`、`target/arm/tcg/translate-a64.c`

### [01-TCG优化递次深度分析.md](accel/01-TCG优化递次深度分析.md)
> **31KB · 29 节**

TCG 优化管线深入分析：6 阶段管线流程（optimize→reachable→pass0/1/2→regalloc）、76 个 fold 函数完整索引、常量折叠框架（do_constant_folding_2）、拷贝传播（环形链表+find_better_copy）、z_mask/o_mask/s_mask 三掩码追踪系统、fold_masks_zosa_int 核心、代数简化模式（8 种 fold_xi/ix/xx 辅助）、分支条件折叠、位域/扩展优化、内存拷贝追踪、屏障合并、进位链优化、活跃性三遍分析（DEAD_ARG/SYNC_ARG 编码）、寄存器分配约束系统、Spill/Reload 策略、ARM64 延迟标志（cpu_NF/ZF/CF/VF）。

**适合读者**：深入理解 TCG 优化机制、需要添加新优化 pass 的开发者。
**关键源文件**：`tcg/optimize.c`（3244行/76个fold函数）、`tcg/tcg.c`（活跃性+寄存器分配）

### [02-TCG-IR设计与前端翻译深度分析.md](accel/02-TCG-IR设计与前端翻译深度分析.md)
> **46KB · 32 节**

TCG IR 类型系统完整解析：TCGTemp（13字段详解）/TCGOp（8字段+柔性数组）/TCGLabel/TCGContext（~100字段分组），TCGv 类型安全句柄设计（不完整类型指针+偏移编码），126 个操作码分类索引（控制/算术/逻辑/位域/转换/TB/内存/向量），临时变量 5 种生命周期（EBB/TB/GLOBAL/FIXED/CONST），ARM64 全局寄存器映射（cpu_X[32]/cpu_pc/cpu_NF/ZF/CF/VF），NZCV 缓存编码体系（反转 ZF/bit31 NF+VF），前端翻译架构（translator_loop/DisasContext/decodetree），ARM64 指令翻译模式（数据处理/内存/分支/异常），arm_test_cc 15 条件评估，CCMP/CSEL 翻译，Helper 机制，完整翻译示例。

**适合读者**：理解 TCG IR 设计哲学、需要修改前端翻译器的开发者。
**关键源文件**：`include/tcg/tcg.h`、`tcg/tcg-op.c`、`target/arm/tcg/translate-a64.c`（10961行/156个trans_函数）

### [03-TCG后端代码生成与TB管理深度分析.md](accel/03-TCG后端代码生成与TB管理深度分析.md)
> **30KB · 30 节**

TCG 后端代码生成完整流程：tcg_gen_code() 8 阶段管线（优化→活跃性→寄存器分配→发射→后处理→重定位→icache 刷新），寄存器分配核心 tcg_reg_alloc_op()（~900行/约束满足/输入输出分配/DEAD+SYNC 处理），寄存器选择算法（两遍优先偏好策略/spill 驱逐），temp 加载/同步/保存策略（三态管理 REG/CONST/MEM），栈帧 spill 槽分配（对齐/溢出/子部分），约束系统（TCGArgConstraint 9 字段/后端定义→解析→查询→分配），进位链处理，AArch64 后端完整实现（3592行）：寄存器定义（64个/分配序/保留寄存器），指令编码体系（15+格式函数），常量加载策略（5 级递降：MOVZ→逻辑→ADR→ADRP+ADD→多条 MOVK），Prologue/Epilogue（BTI/保存恢复/guest_base），qemu_ld/st 快慢路径，TranslationBlock 结构（15 字段详解/cflags 标志），TB 完整生命周期（创建→注册→查找→链接→执行→失效→回收），TB 两级查找（jump cache + 全局哈希），TB 链接（goto_tb 修补），失效机制（6 种触发），代码缓冲区管理（region 分区/溢出/刷新），完整 ADD 指令后端之旅示例。

**适合读者**：需要理解 TCG 如何将 IR 变为机器码、需要添加新后端或优化现有后端的开发者。
**关键源文件**：`tcg/tcg.c`（~6770行）、`tcg/aarch64/tcg-target.c.inc`（3592行）、`accel/tcg/cpu-exec.c`、`accel/tcg/translate-all.c`、`accel/tcg/tb-maint.c`

### [04-MTTCG多线程翻译深度分析.md](accel/04-MTTCG多线程翻译深度分析.md)
> **28KB · 28 节**

MTTCG 多线程 TCG 执行模型完整解析：MTTCG 启用配置（TCGState/mttcg_supported/AUTO-ON-OFF决策），vCPU 线程模型（MTTCG 每 vCPU 独立线程 vs RR 单线程轮转），模式选择分发（tcg_accel_ops_init），TCGContext 线程管理（TLS tcg_ctx/tcg_ctxs数组/tcg_register_thread），代码缓冲区 Region 分区（无锁翻译），TB 哈希表并发访问（QHT 无锁读+bucket锁写），jmp_lock 跳转链保护，TB 刷新同步（exclusive context），CF_PARALLEL 标志影响范围（原子操作/内存屏障/TB缓存），TCG 内存屏障体系（TCGBar 类型/tcg_gen_mb/ARM64 DMB-DSB-ISB 映射），原子操作翻译（helper模板/CF_PARALLEL降级/LDXR-STXR），cpu_exec_step_atomic 原子单步（全局串行化），TLB 线程安全（per-vCPU/跨CPU异步刷新），ARM64 Exclusive Monitor，vCPU Kick 机制（exit_request/SIG_IPI/pthread_kill），中断投递（tcg_handle_interrupt/cpu_handle_interrupt），BQL 使用模式，Exclusive 上下文（start/end_exclusive 全局串行化），cpu_exec_start/end 执行区间管理，Safe Work 跨vCPU操作，icount 指令计数模式，EXCP_ATOMIC 完整处理流程，RR vs MTTCG 全面对比表。

**适合读者**：需要理解 QEMU 多线程执行模型、调试并发问题、优化多核性能的开发者。
**关键源文件**：`accel/tcg/tcg-accel-ops-mttcg.c`（138行）、`accel/tcg/tcg-accel-ops-rr.c`（349行）、`accel/tcg/tcg-accel-ops.c`、`cpu-common.c`、`system/cpus.c`

### [05-Softmmu-TLB与内存访问深度分析.md](accel/05-Softmmu-TLB与内存访问深度分析.md)
> **23KB · 27 节**

Softmmu TLB 核心机制完整解析：CPUTLB 数据结构层次（CPUTLBEntry/CPUTLBDescFast/CPUTLBEntryFull/CPUTLBDesc/CPUTLBCommon/CPUTLB），CPUNegativeOffsetState 负偏移缓存优化布局，TLB 标志位两级体系（快路径 TLB_INVALID/NOTDIRTY/FORCE_SLOW + 慢路径 BSWAP/WATCHPOINT/CHECK_ALIGNED/DISCARD_WRITE/MMIO），TLB 索引计算与 MMU 模式反转，动态 TLB 大小调整（100ms 窗口/扩容翻倍/缩容减半），Victim TLB（8 条目全相联/swap 回主 TLB），tlb_set_page_full 条目填充（address_space_translate_for_iotlb/标志设置/victim 驱逐），AArch64 内联快路径 prepare_host_addr（LDP-AND-ADD-LDP-CMP ~8 条指令），慢路径 helper_ld/st_mmu 入口，tlb_fill_align TLB 重填（新旧接口），mmu_lookup1 核心查找（TLB hit/victim/refill），mmu_watch_or_dirty 特殊页处理，MMIO 分发（io_prepare/memory_region_dispatch_read/write/mr->ops），跨页访问拆分，字节序处理（TLB_BSWAP），脏页追踪（TLB_NOTDIRTY/notdirty_write），Watchpoint 集成，TLB 刷新（全局/按 mmuidx/单页/跨 CPU 同步），ARM64 页表遍历（arm_cpu_tlb_fill_align/get_phys_addr），完整地址翻译链路图。

**适合读者**：需要理解 QEMU 内存虚拟化核心机制、优化 TLB 性能、调试内存访问问题的开发者。
**关键源文件**：`accel/tcg/cputlb.c`（~2800行）、`tcg/aarch64/tcg-target.c.inc`（3592行）、`include/exec/tlb-common.h`、`include/exec/tlb-flags.h`、`include/hw/core/cpu.h`、`system/memory.c`、`system/physmem.c`、`target/arm/tcg/tlb_helper.c`

### [06-TCG-Plugin系统深度分析.md](accel/06-TCG-Plugin系统深度分析.md)
> **27KB · 26 节**

TCG Plugin 动态二进制分析系统完整解析：Plugin API v6 版本体系与类型层次（qemu_plugin_id_t/回调类型/qemu_plugin_install），Plugin 加载流程（g_module_open/符号查找/版本检查/ID分配/install调用），命令行 -plugin 选项解析（多插件支持），核心状态管理（qemu_plugin_state/ctx/cb 链），回调注册机制（do_plugin_register_cb/RCU/事件掩码），TB 翻译集成（translator.c 中 4 个 hook 点：tb_start/insn_start/insn_end/tb_end），plugin_gen_inject 代码注入（TB/指令/内存三级），内存回调插桩（gen_enable/disable_mem_helper/值捕获），Scoreboard per-vCPU 数据存储（GArray/find/u64 操作），内联操作（ADD_U64/STORE_U64/3 条宿主指令无 helper 调用），条件回调（scoreboard 值检查 + 条件分支），内存回调分发（qemu_plugin_vcpu_mem_cb/REGULAR+INLINE 两种 cb 类型），vCPU 生命周期回调（init/exit/idle/resume），Syscall 回调（call/ret/filter v6 新增），Plugin 与 cputlb 集成（force_mmio 强制慢路径），卸载清理（reset_destroy/TB flush/dlclose），线程安全（plugin.lock + RCU + start_exclusive），构建系统（meson.build 符号导出），14 个官方示例插件 + 11 个测试插件分析。

**适合读者**：需要编写 QEMU 插件进行动态分析、理解插件如何与 TCG 代码生成集成的开发者。
**关键源文件**：`include/plugins/qemu-plugin.h`（~1400行）、`plugins/loader.c`（~420行）、`plugins/core.c`（~880行）、`plugins/api.c`（~740行）、`accel/tcg/plugin-gen.c`（~510行）、`accel/tcg/translator.c`

---

## debug/ 调试

### [00-GDB调试子系统深度分析.md](debug/00-GDB调试子系统深度分析.md)
> **49KB · 26 节 + 3 附录**

GDB 远程串行协议 (RSP) 完整实现分析。RSP 状态机、命令分发与完整映射表、ARM64 寄存器映射（核心/FPU/SVE/SME/MTE/pauth）、XML 目标描述系统、断点/监视点（TCG vs KVM）、单步执行、多核调试、vCont 执行控制、KVM 调试限制、System/User 模式差异。

**适合读者**：使用 GDB 调试 QEMU 虚拟机或扩展 GDB stub 的开发者。  
**关键源文件**：`gdbstub/gdbstub.c`、`gdbstub/system.c`、`target/arm/gdbstub64.c`

---

## 文档规范

### 源码引用格式
- 文件名:行号 — 如 `virt.c:173-254`
- 仅使用文件 basename，不含路径前缀

### 结构约定
- 每篇文档含目录、编号章节、附录
- 关键数据结构用代码块展示
- 架构图使用 ASCII art（非 Mermaid）
- 交叉引用使用相对路径 markdown 链接

### Git 信息
- 基准版本：QEMU 11.0.50
- 关键变更附 commit SHA
- 各文档附录包含相关文件的 git 历史

### 分析工具
- 索引：zoekt（全文搜索）+ ctags（符号索引）+ cscope（调用图）
- 索引位置：`.ai-search/` 目录
- 索引规模：11,056 文件、345,817 符号、6,765 C/C++ 文件

---

## 推荐阅读顺序

### 入门路线（从全局到局部）
1. `architecture/00-全局架构概览.md` — 建立整体认知
2. `architecture/01-QOM对象模型深度分析.md` — 理解对象系统
3. `memory/00-内存子系统深度分析.md` — 理解内存模型
4. `memory/01-RAM管理与脏页追踪深度分析.md` — RAM 分配与脏页追踪
5. `device-model/00-设备模型与virtio深度分析.md` — 理解设备框架

### ARM64 专题路线
1. `arm64/00-ARM64-CPU-GICv3-TCG深度分析.md` — CPU 模型入门
2. `arm64/06-异常级别状态管理深度分析.md` — EL 状态切换
3. `arm64/07-不同EL下指令执行差异深度分析.md` — 指令 trap 机制
4. `arm64/08-TrustZone安全扩展与Secure-World深度分析.md` — 安全世界
5. `arm64/09-虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU.md` — 虚拟化
6. `arm64/10-异常入口与返回深度分析.md` — 异常入口/返回/ERET
7. `arm64/11-MMU-TLB深度分析.md` — MMU 页表遍历与 TLB
8. `arm64/12-Generic-Timer定时器深度分析.md` — 定时器与计数器
9. `arm64/03-GICv3中断控制器模拟架构深度分析.md` — 中断系统
10. `arm64/04-中断注入与处理深度分析.md` — 中断完整路径
11. `arm64/02-特殊指令模拟深度分析.md` — 指令级细节

### I/O 路径路线
1. `architecture/02-模拟执行循环与MMIO分发深度分析.md` — 执行 + MMIO
2. `device-model/02-块层IO路径深度分析.md` — 块 I/O 深入
3. `device-model/04-Block设备子系统深度分析.md` — Block 子系统 + virtio-blk 设备
4. `device-model/06-网络后端深度分析-TAP-vhost-net-vhost-user.md` — 网络 I/O + virtio-net 设备
5. `network/00-网络子系统深度分析.md` — 网络核心架构 + 完整收发路径追踪

### 调试专题
1. `debug/00-GDB调试子系统深度分析.md` — GDB 接入分析

---

> **仓库地址**: [github.com/lixunwei/ai-qemu](https://github.com/lixunwei/ai-qemu)  
> **文档版本**: v1.0  
> **最后更新**: 2025-07
