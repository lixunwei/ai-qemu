# QEMU 11.0.50 ARM64 源码深度分析 — 总索引

> 本文档集基于 QEMU 11.0.50 源码，聚焦 ARM64 (AArch64) 架构  
> 使用 AI 辅助分析，所有源码引用均标注文件名:行号及关键 git commit SHA  
> 共 **106 篇文档**，总计 **~3012KB** 中文技术文档

---

## 快速导航

| 分类 | 文档数 | 总大小 | 核心主题 |
|------|--------|--------|---------|
| [architecture/](#architecture-架构) | 15 | ~342KB | 全局架构、QOM、执行循环、Machine 建立、线程模型、事件循环与I/O模型、块层核心架构、qcow2与块驱动、TCG后端、TCG优化与TLB、VirtIO与vhost、内存子系统、MTTCG并行执行、TCG前端翻译、主事件循环与协程 |
| [arm64/](#arm64-arm64-架构) | 37 | ~872KB | CPU 模型、GICv3、TCG、ACPI、FDT、中断、特殊指令、EL 状态、TrustZone、虚拟化扩展、异常入口与返回、MMU/TLB、Generic Timer、PMU、CPU 特性与 ID 寄存器、SVE/SME、PAC/BTI/MTE、GCS/RME/新扩展、VirtIO、PCI/PCIe、SMMUv3/IOMMU、EL 状态管理与指令执行、安全中断路由与流转、GICv3 中断生命周期、ITS/LPI、GICv3 寄存器与状态机、中断虚拟化、KVM vGIC、系统寄存器模拟、MMU 页表遍历、EL2/EL3 陷阱路由、特殊寄存器与 Cache/AT 指令、Debug/Breakpoint/Watchpoint/RAS、ID 寄存器与特性发现、EL 状态切换与 PSTATE、EL 指令执行流差异、安全状态转换与 SCR/HCR 联动 |
| [device-model/](#device-model-设备模型) | 7 | ~399KB | 设备框架、virtio、块层、chardev、VFIO、网络、DMA |
| [network/](#network-网络子系统) | 1 | ~48KB | 网络核心架构、TAP/SLIRP/Socket 后端、vhost-net、virtio-net 设备模型、收发路径 |
| [memory/](#memory-内存子系统) | 2 | ~57KB | MemoryRegion、MMIO、IOMMU、RAMBlock、脏页追踪、NUMA |
| [accel/](#accel-加速器) | 8 | ~273KB | TCG 翻译引擎全貌、优化递次、IR 与前端翻译、后端代码生成与 TB 管理、MTTCG 多线程翻译、Softmmu TLB 与内存访问、TCG Plugin 系统、Linux-user 用户模式翻译 |
| [arm/](#arm-arm-架构通用) | 14 | ~296KB | EL 状态管理、AArch32 异常、CP15/MMU、GICv3、Cache、Timer、PMU、调试、SVE/SME、TrustZone、内存模型、TZ 安全组件模拟、RME/Realm、GICv2 vs GICv3 对比 |
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

### [06-块层核心架构深度分析.md](architecture/06-块层核心架构深度分析.md)
> **10KB · 11 节**

块层三层架构（BlockBackend → 格式 BDS → 协议 BDS）、BlockDriver/BDS/BlockBackend 核心结构体、BdrvChild 父子关系与 BDS 树、I/O 分发路径（bdrv_co_preadv_part → bdrv_driver_preadv 优先级链）、协程化 I/O、驱动注册（block_init 构造器模式）、格式探测机制、块设备管理命令。

**适合读者**：需要理解 QEMU 块层整体架构、BDS 图拓扑或开发新块驱动的开发者。  
**关键源文件**：`include/block/block_int-common.h`、`block/block-backend.c`、`block/io.c`、`block.c`

### [07-qcow2格式驱动与块IO后端深度分析.md](architecture/07-qcow2格式驱动与块IO后端深度分析.md)
> **9KB · 11 节**

qcow2 磁盘格式（QCowHeader、L1/L2 两级地址翻译、引用计数、快照/加密/压缩）、读写路径（get_host_offset → 簇类型判断 → COW 分配）、file-posix 协议驱动（io_uring/linux-aio/线程池三模式、O_DIRECT、文件锁）、raw 格式透传、virtio-blk 设备（virtqueue → blk_co_preadv、IOThread dataplane）、块作业框架（mirror/commit/stream/backup、Job 状态机）。

**适合读者**：需要理解 qcow2 内部实现、优化块 I/O 性能或分析磁盘格式的开发者。  
**关键源文件**：`block/qcow2.c`、`block/qcow2.h`、`block/qcow2-cluster.c`、`block/file-posix.c`、`hw/block/virtio-blk.c`

### [08-TCG后端深度分析-IR生成寄存器分配与代码缓存.md](architecture/08-TCG后端深度分析-IR生成寄存器分配与代码缓存.md)
> **12KB · 14 节**

TCG JIT 编译器全流程：TCGTemp/TCGOp/TCGContext 数据结构、tcg_gen_* 前端 IR 生成 API、tcg_optimize() 常量折叠/传播/死代码消除、三遍活性分析（liveness_pass_0/1/2）、线性扫描寄存器分配（tcg_reg_alloc_op/mov）、all_outop[] 后端分发表、AArch64 后端实现（tcg_out_ldst/qemu_ld_direct/prologue）、TranslationBlock 结构与 cflags 编译标志、代码缓存多区域管理（per-vCPU 独占）、TB 失效与全量刷新、TB 链接（Block Chaining）消除查找开销、cpu_exec_loop 三级 TB 查找执行循环。

**适合读者**：需要深入理解 TCG JIT 编译器内部实现、优化代码生成或扩展新后端的开发者。  
**关键源文件**：`tcg/tcg.c`、`include/tcg/tcg.h`、`tcg/optimize.c`、`tcg/region.c`、`tcg/aarch64/tcg-target.c.inc`、`accel/tcg/translate-all.c`、`accel/tcg/cpu-exec.c`

### [09-TCG深入分析-优化遍向量指令与Softmmu-TLB机制.md](architecture/09-TCG深入分析-优化遍向量指令与Softmmu-TLB机制.md)
> **16KB · 12 节**

TCG 优化遍详解：OptContext/TempOptInfo 数据结构、80+ fold_* 按操作码分发（常量折叠/拷贝传播/条件简化/死代码消除）、z_mask/o_mask/s_mask 三元组比特级值范围追踪。向量指令翻译：55+ 向量操作码（V64/V128/V256）、tcg_gen_gvec_* 高级 API（自动向量 IR/标量展开/OOL helper 选择）、AArch64 后端 tcg_expand_vec_op 复杂操作展开。Softmmu TLB：CPUTLBEntry/CPUTLBDescFast 快速路径结构、6 条内联指令 TLB 查找、TLB 标志位（INVALID/NOTDIRTY/MMIO/FORCE_SLOW）、8 路全关联 Victim TLB 机制、tlb_set_page_full 填充与多粒度刷新。

**适合读者**：需要理解 TCG 优化策略、向量指令模拟机制或 Softmmu 地址翻译性能的开发者。  
**关键源文件**：`tcg/optimize.c`、`tcg/tcg-op-vec.c`、`tcg/tcg-op-gvec.c`、`tcg/aarch64/tcg-target.c.inc`、`accel/tcg/cputlb.c`、`include/exec/tlb-common.h`、`include/exec/tlb-flags.h`

### [10-VirtIO设备深度分析-Virtqueue实现vhost-user与IOMMU集成.md](architecture/10-VirtIO设备深度分析-Virtqueue实现vhost-user与IOMMU集成.md)
> **16KB · 12 节**

VirtIO 设备框架全解：VirtIODevice 三层特性协商、VRing 描述符/avail/used 环结构、Split/Packed 两种 Virtqueue 模式、virtqueue_pop/push/notify 请求处理流程、VirtIOPCIProxy PCI 传输层（Modern BAR/MSI-X/ioeventfd）、VirtIO-Net 收发路径、vhost-user 协议（43 种消息类型/Unix Socket + fd 传递/内存 mmap 共享）、vhost 数据路径优化（eventfd/irqfd 绕过 QEMU）、VirtIO IOMMU 设备（GTree 域端点管理/virtio_iommu_translate 地址翻译）、DMA 地址空间 IOMMU 集成。

**适合读者**：需要理解 VirtIO 设备模型、优化虚拟化 I/O 性能或开发 vhost-user 后端的开发者。  
**关键源文件**：`hw/virtio/virtio.c`、`include/hw/virtio/virtio.h`、`hw/virtio/virtio-pci.c`、`hw/virtio/vhost.c`、`hw/virtio/vhost-user.c`、`hw/virtio/virtio-iommu.c`

### [11-内存子系统深度分析-MemoryRegion树FlatView与RAMBlock.md](architecture/11-内存子系统深度分析-MemoryRegion树FlatView与RAMBlock.md)
> **15KB · 12 节**

QEMU 内存子系统全解：MemoryRegion 四种类型（RAM/MMIO/别名/容器）及树形组织、MemoryRegionOps 回调接口（带属性访问/大小约束/字节序）、render_memory_region 递归扁平化（优先级/重叠/别名展开）、FlatView 有序范围数组 + dispatch 基数树、AddressSpace RCU 保护与事务性更新、MemoryRegionSection 地址解析、RAMBlock 结构（mmap/脏页位图/迁移元数据）、三种脏页客户端（VGA/Code/Migration）、MemoryListener 通知机制（KVM EPT/TCG TLB/vhost）、完整 MMIO 分发路径、MemoryRegionCache 快速缓存、DMA 映射（RAM 零拷贝/MMIO bounce buffer）。

**适合读者**：需要理解 QEMU 内存管理架构、设备 MMIO 注册或内存热插拔机制的开发者。  
**关键源文件**：`include/system/memory.h`、`system/memory.c`、`system/physmem.c`、`include/system/ramblock.h`

### [12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障.md](architecture/12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障.md)
> **24KB · 12 节**

QEMU MTTCG（Multi-Threaded TCG）全解：TCGState 与 MTTCG 初始化决策（mttcg_supported/icount 互斥）、per-vCPU 线程主循环（mttcg_cpu_thread_fn）、TB 失效机制（单 TB/范围/全局 flush）、自修改代码（SMC）精确检测与回退、PageDesc 页面级 TB 追踪（自旋锁保护）、TCGBar 内存排序屏障枚举与 AArch64 DMB 映射、原子操作模板与 EXCP_ATOMIC 串行回退、独占执行上下文（start_exclusive/end_exclusive 与 pending_cpus 协调）、TB 链接的多线程安全（jmp_lock/CAS/CF_INVALID）、icount 确定性模式与 MTTCG 互斥原因。

**适合读者**：需要理解 QEMU 多线程 TCG 并行执行、TB 生命周期管理或客户机内存模型模拟的开发者。  
**关键源文件**：`accel/tcg/tcg-all.c`、`accel/tcg/tcg-accel-ops-mttcg.c`、`accel/tcg/tb-maint.c`、`accel/tcg/cpu-exec.c`、`cpu-common.c`、`include/tcg/tcg-mo.h`

### [13-TCG前端翻译深度分析-指令解码IR生成与优化Pass.md](architecture/13-TCG前端翻译深度分析-指令解码IR生成与优化Pass.md)
> **22KB · 12 节**

TCG 前端翻译管线全解：decodetree DSL 指令解码框架（scripts/decodetree.py 从 .decode 文件生成 C 解码器）、AArch64 15 个 .decode 文件（8587 行）与解码级联（disas_a64→disas_sme→disas_sve）、TranslatorOps 翻译循环（translator_loop 5 回调架构）、DisasContext 从 TB 标志恢复 60+ 字段、TCG IR 核心结构（TCGOp/TCGTemp/TCGContext）、完整 ADD_i 翻译追踪（decode→TRANS 宏→gen_rri→tcg_gen_add_i64→INDEX_op_add）、Helper 函数调用机制（DEF_HELPER→gen_helper→tcg_gen_callN）、tcg_optimize() 优化 Pass（常量折叠/复制传播/50+ 专用 fold 函数）、活性分析三阶段（liveness_pass_0/1/2 标记 DEAD_ARG/SYNC_ARG）、后端寄存器分配（tcg_reg_alloc_op 延迟分配）、Prologue 生成与代码缓冲区管理。

**适合读者**：需要理解 QEMU 翻译管线、添加新指令或新架构前端的开发者。  
**关键源文件**：`scripts/decodetree.py`、`target/arm/tcg/a64.decode`、`target/arm/tcg/translate-a64.c`、`accel/tcg/translator.c`、`tcg/tcg-op.c`、`tcg/optimize.c`、`tcg/tcg.c`、`include/tcg/tcg.h`

---

### [14-TCG后端代码生成深度分析-AArch64后端寄存器分配与TLB慢路径.md](architecture/14-TCG后端代码生成深度分析-AArch64后端寄存器分配与TLB慢路径.md)
> **22KB · 12 节**

TCG 后端代码生成管线全解：tcg_gen_code() 主循环（Op 遍历与分发）、tcg_reg_alloc_op() 寄存器分配三阶段（输入分配→输出分配→发射）、outop 表驱动架构（outop_add/outop_and 等替代传统 tcg_out_op 大 switch）、C_O1_I2(r,r,rA) 约束记法、AArch64 寄存器布局（callee-saved 优先分配 X20-X28、TMP0/TMP1/TMP2 保留、AREG0=X19、GUEST_BASE=X28）、AArch64Insn 枚举与 tcg_out_insn 类型安全宏、tcg_out_movi 7 级立即数加载策略（MOVZ→MOVN→ORR→ADR→ADRP+ADD→MOVZ+MOVK→常量池）、TLB 快路径 prepare_host_addr（LDP→AND_LSR→ADD→LDR→CMP→B.NE）、TLB 慢路径延迟发射（tcg_out_ldst_finalize）、tcg_out_qemu_ld/st_direct Host 内存访问、goto_tb 双模式 TB 链接（B 直接/LDR 间接）与原子补丁、exit_tb 返回主循环、Prologue/Epilogue 完整栈帧（STP/LDP X19-X27）、条件码映射与 CBZ/CBNZ/TBZ/TBNZ 分支优化、三种重定位类型（pc26/pc19/pc14）。

**适合读者**：需要理解 TCG 后端机器码生成、寄存器分配或添加新 Host 架构后端的开发者。  
**关键源文件**：`tcg/tcg.c`、`tcg/aarch64/tcg-target.c.inc`、`tcg/aarch64/tcg-target.h`、`include/tcg/tcg.h`

---

### [15-内存子系统深度分析-MemoryRegion-AddressSpace-FlatView与IOMMU.md](architecture/15-内存子系统深度分析-MemoryRegion-AddressSpace-FlatView与IOMMU.md)
> **31KB · 12 节**

内存子系统全栈分析：MemoryRegion 树形层次（Container/Alias/Overlap 三种组合、6种MR类型初始化）、MemoryRegionOps 设备MMIO回调（read/write/read_with_attrs 及 valid/impl 访问约束）、AddressSpace 地址空间（RCU 保护的 current_map、DMA bounce buffer 机制）、FlatView 拓扑展平（render_memory_region 递归算法、优先级间隙填充、flatview_simplify 合并）、地址解析全路径（flatview_do_translate→address_space_translate_internal→MR dispatch 或 RAM 直接访问）、MemoryListener 观察者模式（region_add/del/nop 回调、两遍 diff 算法、事务提交流程）、IOMMU 级联翻译（IOMMUMemoryRegionClass.translate 回调、多级 do-while 循环、IOMMUNotifier 失效通知）、TCG 内存访问路径（TLB addend 直通 vs tlb_fill→address_space_translate 完整路径）、KVM 内存路径（kvm_region_add/del/commit 事务批量提交、KVM_SET_USER_MEMORY_REGION2 ioctl、dirty ring 脏页同步）、RAMBlock 物理内存后端（host mmap、脏页位图、迁移支持）、ROM/ROM设备区别。

**适合读者**：需要理解 QEMU 内存映射机制、编写设备 MMIO、调试地址解析或理解 KVM/TCG 内存集成的开发者。  
**关键源文件**：`include/system/memory.h`、`system/memory.c`、`system/physmem.c`、`accel/kvm/kvm-all.c`、`include/system/ramblock.h`

### [16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md](architecture/16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md)
> **24KB · 12 节**

时钟与定时器全栈分析：QEMUTimer 核心基础设施（4 种时钟类型 REALTIME/VIRTUAL/HOST/VIRTUAL_RT、QEMUTimerList 有序链表、timer_init_full/timer_mod_ns/timer_del API）、时间获取语义（qemu_clock_get_ns 按类型分发、VM 暂停行为差异）、定时器到期分发（timerlist_run_timers 弹出循环、锁外回调设计）、主循环集成（timerlistgroup_deadline_ns → poll 超时、qemu_clock_advance_virtual_time warp 逐步推进）、ptimer 周期定时器（ptimer_state 结构、enabled 三态、ptimer_tick 单次停止/周期重装、ptimer_reload 最低周期保护、6 种策略标志、事务机制）、ARM Generic Timer 架构（7 种定时器实例 PHYS/VIRT/HYP/SEC/HYPVIRT/S_EL2_PHYS/S_EL2_VIRT、ARMGenericTimer cval+ctl 状态、gt_get_countervalue 虚拟时钟除以频率）、系统寄存器（CNTFRQ_EL0/CNTP_CTL_EL0/CNTV_CVAL_EL0 等 ARMCPRegInfo 定义、CTL ENABLE/IMASK/ISTATUS 控制位）、gt_recalc_timer 核心重算（ISTATUS 计算、nexttick 溢出处理、timer_mod 编程）、gt_update_irq 中断连接（ISTATUS∧¬IMASK → qemu_set_irq → GICv3 PPI）、定时器偏移（gt_direct_access_timer_offset、CNTVOFF_EL2 虚拟化偏移、EL 相关物理偏移）、Clock 设备时钟框架（Clock 结构体、period 定点数、时钟树 source/children、multiplier/divider 分频倍频、clock_propagate 递归传播）、icount 指令计数模式（VIRTUAL 时钟由指令驱动、deadline 排除、精确时序调试）。

**适合读者**：需要理解 QEMU 定时器机制、ARM Generic Timer 虚拟化、设备定时器开发或 icount 精确时序的开发者。  
**关键源文件**：`include/qemu/timer.h`、`util/qemu-timer.c`、`hw/core/ptimer.c`、`target/arm/helper.c`、`target/arm/gtimer.h`、`include/hw/core/clock.h`

### [17-PMU性能监控单元深度分析-事件计数器-溢出中断与EL过滤.md](architecture/17-PMU性能监控单元深度分析-事件计数器-溢出中断与EL过滤.md)
> **24KB · 12 节**

ARM PMUv3 性能监控单元全栈分析：pm_event 事件表（SW_INCR/INST_RETIRED/CPU_CYCLES/STALL_FRONTEND/STALL_BACKEND/STALL 共 6 种事件、get_count 底层计数获取、ns_per_count 溢出预测）、CPUARMState PMU 状态（c9_pmcr/c9_pmcnten/c9_pmovsr/c9_pminten 控制位图、c15_ccnt/c15_ccnt_delta 周期计数器惰性快照、c14_pmevcntr[31]/c14_pmevcntr_delta[31] 事件计数器）、惰性求值核心机制（pmccntr_op_start 快照+溢出检测、pmccntr_op_finish delta 重算+timer_mod_anticipate_ns 溢出预测、pmevcntr_op_start/finish 事件计数器同理、pmu_op_start/finish 聚合操作）、EL 切换快照（pmu_pre_el_change/pmu_post_el_change 在 EL 变化前后拆分计数）、四层使能过滤（PMCR.E 全局使能 → HPMD/SPME/SCCD/HCCD 禁止位 → P/U/NSK/NSU/NSH/M EL 过滤 → event_supported 事件支持检查）、溢出检测与中断（old & ~new & overflow_mask 回绕检测 → PMOVSR 置位 → pmu_update_irq: PMCRE ∧ PMINTEN∩PMOVSR → GICv3 PPI）、PMCR 写入（PMCRC 复位 CCNT、PMCRP 复位事件计数器、HPMN 虚拟化替换 N）、SW_INCR 软件递增（PMSWINC 写入实时递增+溢出检测，不可预测）、PMU 系统寄存器定义（v7_pm_reginfo 数组 + 动态生成 PMEVCNTR/PMEVTYPER、access_tpm 陷入控制）、pmu_timer 溢出预测定时器（arm_pmu_timer_cb → op_start+op_finish 周期唤醒）、KVM PMU 集成（KVM_ARM_VCPU_PMU_V3_INIT/IRQ 初始化、cpreg 迁移路径同步）。

**适合读者**：需要理解 ARM PMU 虚拟化实现、性能计数器惰性求值机制、EL 过滤逻辑或 PMU 中断连接的开发者。  
**关键源文件**：`target/arm/cpregs-pmu.c`、`target/arm/cpu.h`、`target/arm/internals.h`、`target/arm/kvm.c`

### [18-Debug-Breakpoint-Watchpoint调试子系统深度分析-硬件断点-单步与GDB-Stub.md](architecture/18-Debug-Breakpoint-Watchpoint调试子系统深度分析-硬件断点-单步与GDB-Stub.md)
> **25KB · 13 节**

ARM 调试子系统全栈分析：CPUARMState 调试状态（dbgbvr[16]/dbgbcr[16]/dbgwvr[16]/dbgwcr[16] 硬件断点与 Watchpoint 寄存器、mdscr_el1/mdcr_el2/mdcr_el3 调试控制、oslsr_el1/osdlr_el1 OS Lock）、硬件断点实现（dbgbvr_write→hw_breakpoint_update→cpu_breakpoint_insert、BT 类型 地址匹配/上下文匹配/链接断点、BAS 字段 Thumb16 偏移）、Watchpoint 实现（hw_watchpoint_update→cpu_watchpoint_insert、LSC 读/写/访问控制、MASK 对齐区域/BAS 字节选择、最大 2GB 范围）、bp_wp_matches 统一匹配（SSC 安全状态 → PAC/HMC EL 控制 → WT 链接断点、断点 PC 匹配 + Watchpoint HIT 标志）、linked_bp_matches 上下文 ID 匹配、MDSCR_EL1 调试控制（MDE 监视使能 bit[15]、KDE 内核调试 bit[13]、SS 软件单步 bit[0]）、调试异常路由（arm_debug_target_el: HCR_TGE/MDCR_TDE→EL2、MDCR_EL3.SDD 禁用安全调试、同 EL 需 KDE+¬PSTATE.D）、软件单步状态机（Active-not-pending PSTATE.SS=1 → gen_ss_advance 清除 → Active-pending → gen_step_complete_exception → EC_SOFTWARESTEP）、BRK 指令（trans_BRK→gen_exception_bkpt_insn→HELPER(exception_bkpt_insn)、目标 EL<当前 EL 时提升、EC_AA64_BKPT 0x3c）、调试异常分发（arm_debug_excp_handler: Watchpoint→EXCP_DATA_ABORT+syn_watchpoint, 断点→EXCP_PREFETCH_ABORT+syn_breakpoint, GDB BP_GDB 优先于 BP_CPU）、GDB Stub 集成（gdb_breakpoint_insert/remove、hyp_gdbstub.c 硬件后端）、KVM 调试（debug exit EC 分发、KVM_GUESTDBG_USE_HW/SW_BP、BRK#0 补丁）。

**适合读者**：需要理解 ARM 调试架构虚拟化、硬件断点/Watchpoint 机制、TCG 单步实现或 GDB/KVM 调试集成的开发者。  
**关键源文件**：`target/arm/tcg/debug.c`、`target/arm/debug_helper.c`、`target/arm/tcg/translate.h`、`target/arm/syndrome.h`、`target/arm/kvm.c`

### [19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译.md](architecture/19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译.md)
> **32KB · 13 节**

SMMUv3/IOMMU 全栈分析：IOMMU 核心框架（IOMMUTLBEntry 翻译结果、IOMMUNotifier MAP/UNMAP/DEVIOTLB_UNMAP 通知机制、IOMMUMemoryRegionClass 虚函数表 translate/notify_flag_changed/replay/get_attr）、SMMUv3State 设备状态（cr[3]/gbpa/strtab_base/cmdq/eventq 寄存器、4 种 IRQ、stage/accel/ril/ats/oas/ssidsize 属性）、SMMU 通用基础层（SMMUState 基类 configs 配置缓存/iotlb TLB 缓存/devices_with_notifiers 通知列表、SMMUDevice 每设备 IOMMU 上下文、smmu_get_sid() = PCI_BUILD_BDF 派生 Stream ID）、Stream Table Entry（STE 512 位 CONFIG S1/S2/Bypass/Abort、线性与两级 Stream Table 查找 smmu_find_ste、decode_ste 多层验证）、Context Descriptor（CD TTB0/TTB1/TSZ/TG/ASID/IPS、smmu_get_cd Nested 下 CD 地址 S2 翻译、decode_cd TTB Nested S2 翻译）、翻译主路径（smmuv3_translate 入口 → smmuv3_get_config 配置缓存 → smmuv3_do_translate 三类 CLASS_CD/TT/IN → smmu_translate TLB 查找 + PTW）、页表遍历（smmu_ptw 调度 S1/S2/Nested、smmu_ptw_64_s1 select_tt→逐级遍历→Nested 表地址 S2 翻译、smmu_ptw_64_s2 拼接表→S2AP 权限、combine_tlb 合并）、IOTLB 缓存（Jenkins Hash 5 字段键、逐级查找 smmu_iotlb_lookup_all_levels、256 条容量、6 种粒度失效）、命令队列（20+ 命令类型、smmuv3_cmdq_consume 主循环、smmuv3_range_inval RIL 范围失效）、事件队列（18 种事件类型、smmuv3_record_event→smmuv3_propagate_event→write_eventq→EVTQ IRQ、队列满→GERROR）、中断机制（EVTQ/PRIQ/CMD_SYNC/GERROR 四种 IRQ、GERROR toggle 协议）、IOMMU Notifier（smmuv3_notify_iova 按 ASID/VMID 过滤→memory_region_notify_iommu_one、仅支持 UNMAP 通知）、virt 机器集成（create_smmu stage=nested、primary-bus/memory/secure-memory 关联、DT iommu-map 绑定）。

**适合读者**：需要理解 ARM SMMUv3 IOMMU 虚拟化实现、DMA 地址翻译路径、嵌套翻译机制或 VFIO 直通设备 IOMMU 集成的开发者。  
**关键源文件**：`hw/arm/smmuv3.c`、`hw/arm/smmu-common.c`、`include/hw/arm/smmuv3.h`、`include/hw/arm/smmu-common.h`、`include/hw/arm/smmuv3-common.h`、`hw/arm/smmuv3-internal.h`、`include/system/memory.h`

### [20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md](architecture/20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md)
> **34KB · 13 节**

PCI/PCIe 子系统全栈分析：PCIDeviceClass 设备类方法表（realize/exit/config_read/config_write/vendor_id/device_id）、PCIDevice 设备实例（5 个配置空间数组 config/cmask/wmask/w1cmask/used、io_regions[7] BAR 区域、bus_master_as DMA 地址空间、MSI/MSI-X 双栈、PCIExpressDevice PCIe 扩展）、PCIBus 总线模型（devices[256] 设备表、set_irq/map_irq/route_intx_to_irq IRQ 三回调、address_space_mem/io 地址空间）、PCIHostState 主桥（conf_mem/data_mem/mmcfg 三种配置访问方式）、配置空间管理（pci_config_alloc 5 数组分配、pci_init_cmask/wmask/w1cmask 三层掩码、pci_default_write_config 写掩码+W1C+BAR 重映射级联）、配置空间访问路径（ECAM pcie_mmcfg_data_write→pci_host_config_write_common→config_write、Legacy CF8/CFC pci_data_write 调度）、BAR 管理（pci_register_bar 类型标志 IO/MEM64/PREFETCH、pci_bar_address 地址计算+COMMAND 使能检查、pci_update_mappings 动态 add/del_subregion）、PCIe 能力（PCIExpressDevice exp_cap/aer_cap/ats/pasid/pri/acs/sriov、pcie_cap_init v2 初始化）、MSI 中断（msi_init 能力注册 1-32 向量、msi_prepare_message 地址+数据构造、msi_notify→msi_send_message→pci_msi_trigger→address_space_stl_le 投递）、MSI-X 中断（msix_init Table/PBA BAR 内 MMIO 子区域、每条目 16 字节 addr/data/ctrl、msix_table_mmio_read/write Guest 访问、msix_notify 向量 Mask/Pending 管理）、MSI→GIC ITS 路径（bus_master_as 写入→IOMMU 翻译→GITS_TRANSLATER→do_process_its_cmd→process_its_cmd_phys→gicv3_redist_process_lpi）、INTx 传统中断（pci_change_irq_level 逐级 Swizzle 传播、gpex_swizzle_map_irq_fn (slot+pin)%4、irq_count OR 语义）、ECAM/GPEX 主桥（GPEXHost PCIExpressHost 基类、gpex_host_realize MMCFG+MMIO+IO+IRQ+根总线创建、pcie_host.h ECAM 地址编码 bus[27:20]/devfn[19:12]/offset[11:0]）、virt 机器集成（create_pcie ECAM/MMIO/IO 窗口映射、INTx→GIC SPI 连接、DT pci-host-ecam-generic/msi-map/iommu-map 生成）。

**适合读者**：需要理解 PCI/PCIe 设备虚拟化、配置空间访问机制、BAR MMIO 映射、MSI/MSI-X 中断投递路径或 ARM virt PCIe 拓扑集成的开发者。  
**关键源文件**：`include/hw/pci/pci_device.h`（~205行）、`include/hw/pci/pci_bus.h`（~70行）、`hw/pci/pci.c`（~3500行）、`hw/pci/pci_host.c`（~200行）、`hw/pci/pcie_host.c`（~110行）、`hw/pci/msi.c`（~440行）、`hw/pci/msix.c`（~640行）、`hw/pci-host/gpex.c`（~260行）、`hw/arm/virt.c`

### [21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md](architecture/21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md)
> **29KB · 13 节**

VirtIO 子系统全栈分析：VirtIODevice 核心结构（status/isr/queue_sel 状态寄存器、host_features/guest_features/backend_features Feature 三层模型、VirtQueue 数组 1024 槽位、config 设备配置空间、generation 配置代数、dma_as DMA 地址空间）、VirtioDeviceClass 虚函数表（realize/unrealize 生命周期、get_features_ex/set_features_ex Feature 协商、get/set_config 配置空间、reset/set_status 状态管理、guest_notifier_pending/mask 通知控制、get_vhost vhost 加速接口）、VRing 数据结构 Split 格式（VRingDesc 16 字节描述符 addr/len/flags/next、VRingAvail flags/idx/ring[] Available Ring、VRingUsed id/len Used Ring、VRing num/desc/avail/used GPA 映射）、Packed 格式（VRingPackedDesc id 替代 next 链、AVAIL/USED 位 + wrap 计数、VRingPackedDescEvent 事件抑制）、VirtQueue 队列状态（last_avail_idx/shadow_avail_idx 消费索引、used_idx/signalled_used 生产索引、handle_output 回调、guest_notifier/host_notifier EventNotifier）、描述符链处理（virtqueue_split_pop Available Ring 读取→描述符链遍历→间接描述符→VirtQueueElement 构造、virtqueue_packed_pop wrap 计数管理、virtqueue_fill/flush Used Ring 写回 + smp_wmb 内存屏障）、通知与中断机制（virtio_queue_notify Guest kick→ioeventfd/handle_output、virtio_notify→virtio_should_notify 抑制检查→virtio_irq ISR 设置→IOThread defer_call/virtio_notify_vector 中断注入、EVENT_IDX Split 阈值抑制、PACKED_EVENT_FLAG_DESC 精确控制、virtio_notify_config generation 递增 + ISR bit1）、Feature 协商状态机（ACKNOWLEDGE→DRIVER→FEATURES_OK→DRIVER_OK 四阶段、virtio_set_status FEATURES_OK 验证 + DRIVER_OK started 转换、virtio_reset 全状态清零 + vhost_reset_device + 队列逐一复位）、VirtioBus 总线层（VirtioBusClass notify/device_plugged/ioeventfd_assign 传输层接口、virtio_bus_device_plugged pre_plugged→get_features→device_plugged→dma_as 设置、virtio_bus_start_ioeventfd/set_host_notifier EventNotifier 管理）、VirtIO-PCI 传输层（VirtIOPCIProxy PCIDevice 基类 + 5 个 VirtIOPCIRegion common/isr/device/notify/notify_pio + modern_bar BAR 布局 + VirtIOPCIQueue 队列 PCI 状态、virtio_pci_notify MSI-X→msix_notify / Legacy→pci_set_irq 中断投递、common_read/write Feature 读写/STATUS 状态转换/队列地址设置/Q_ENABLE 激活、notify_write queue_index 解码→virtio_queue_notify Kick）、VirtIO-MMIO 传输层（寄存器布局 MAGIC_VALUE/VERSION/DEVICE_ID/FEATURES/QUEUE_SEL/QUEUE_NOTIFY/STATUS/CONFIG、平台 IRQ vs MSI-X、设备树发现 vs PCI 枚举）、vhost 加速框架（vhost_dev 后端状态 memory_listener/vqs/features/acked_features/protocol_features/vhost_ops、vhost_dev_init 后端选择→SET_OWNER→Feature 读取→内存监听注册、vhost_dev_start Feature 设置→IOMMU 监听→内存表推送→vhost_virtqueue_start 逐队列 SET_VRING_NUM/BASE/ADDR/KICK/CALL 卸载、ioeventfd→vhost 内核线程→irqfd→KVM 直接注入 完全绕过 QEMU）、vhost 后端类型（vhost-kernel ioctl 封装 kernel_ops、vhost-user Unix Socket VhostUserRequest 协议 SET_MEM_TABLE fd 传递/SET_VRING_KICK/CALL eventfd 传递 user_ops、vhost-vdpa /dev/vhost-vdpa-N 硬件加速）、vhost-net 网络加速（vhost_net_init nvqs=2 TX+RX、vhost_net_start Guest/Host Notifier 设置→vhost_net_start_one→VHOST_NET_SET_BACKEND TAP fd 绑定）。

**适合读者**：需要理解 VirtIO 设备虚拟化原理、VirtQueue 描述符链处理、中断抑制优化、PCI/MMIO 传输层实现或 vhost 数据面卸载加速的开发者。  
**关键源文件**：`include/hw/virtio/virtio.h`（~250行）、`hw/virtio/virtio.c`（~3700行）、`hw/virtio/virtio-bus.c`（~315行）、`hw/virtio/virtio-pci.c`（~2400行）、`hw/virtio/virtio-mmio.c`（~665行）、`include/hw/virtio/vhost.h`（~140行）、`hw/virtio/vhost.c`（~2300行）、`hw/virtio/vhost-user.c`（~3050行）、`hw/net/vhost_net.c`（~510行）

---

### [22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md](architecture/22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md)
> **27KB · 14 节**

QOM 对象模型全栈分析：TypeInfo 类型定义（name/parent/instance_size/instance_init/instance_post_init/instance_finalize/abstract/class_size/class_init/class_base_init/interfaces）、TypeImpl 内部表示（parent_type 解析后指针、class 懒加载字段、num_interfaces/InterfaceImpl 数组）、类型注册机制（type_init→module_init→__attribute__((constructor))→register_module_init 加入链表、module_call_init(MODULE_INIT_QOM) 遍历执行、type_register_static→type_register_internal→type_table_add 全局哈希表、type_get_or_load_by_name 模块化按需加载）、type_initialize 类初始化链（递归父类初始化→memcpy 继承虚函数表→父类接口继承→本类接口注册→class_base_init 从根到叶→class_init 覆盖虚函数）、Object 生命周期（object_new→type_initialize→g_malloc→object_initialize_with_type、memset 清零→class 绑定→object_ref 引用计数=1→object_class_property_init_all→object_init_with_type 先父后子递归→object_post_init_with_type 先父后子、object_ref/unref 原子引用计数→object_finalize 属性删除+递归 instance_finalize 先子后父+free）、ObjectClass 结构（type 回指 TypeImpl、interfaces GSList、object_cast_cache[4]/class_cast_cache[4] LRU 缓存、unparent 回调、properties 类属性哈希表）、DECLARE 宏家族（OBJECT_DECLARE_TYPE→typedef+G_DEFINE_AUTOPTR+DECLARE_OBJ_CHECKERS、OBJECT_DECLARE_SIMPLE_TYPE→DECLARE_INSTANCE_CHECKER、DECLARE_INSTANCE_CHECKER→OBJECT_CHECK 生成实例转换函数、DECLARE_CLASS_CHECKERS→GET_CLASS/CLASS 生成类转换函数、DECLARE_OBJ_CHECKERS 组合两者）、属性系统（ObjectProperty name/type/get/set/resolve/release/init/opaque/defval、实例属性 object_property_add 存 obj->properties vs 类属性 object_class_property_add 存 klass->properties、查找优先级 实例→类继承链、object_property_add_child 父子层级、object_property_add_link 对象间引用、add_str/bool/enum/uint 类型化助手）、接口机制（InterfaceInfo type 名、InterfaceClass parent_class+interface_type、type_initialize_interface 创建"type::interface"复合类型→追加 class->interfaces、INTERFACE_CHECK 转换宏、object_class_dynamic_cast 接口查找遍历 interfaces GSList）、动态类型转换（object_dynamic_cast→object_class_dynamic_cast、快速路径 name 指针比较、接口路径 interfaces 遍历+type_is_ancestor、普通继承 type_is_ancestor、object_dynamic_cast_assert 4 槽 LRU 缓存+abort 失败）、DeviceState/DeviceClass 设备模型（DeviceClass realize/unrealize/legacy_reset/vmsd/bus_type/props_/user_creatable、DeviceState realized/id/canonical_path/parent_bus/gpios/child_bus/reset、device_set_realized dc->realize→LISTENER_CALL→canonical_path→vmstate_register→子总线 realize→hotplug reset→plug）、SysBusDevice 系统总线设备（mmio[QDEV_MAX_MMIO]/pio[QDEV_MAX_PIO]、sysbus_init_mmio/init_irq/mmio_map/connect_irq）、QDev 属性助手（Property name/info/offset/defval、DEFINE_PROP_BOOL/UINT32/STRING/LINK 宏、device_class_set_props_n→qdev_class_add_property→object_class_property_add 桥接 QOM）、ARMCPU 类型链实例（arm_cpu_type_info .name=TYPE_ARM_CPU .parent=TYPE_CPU abstract=true、继承链 OBJECT→DEVICE→CPU→ARM_CPU→AARCH64_CPU、初始化顺序 type_initialize class 链→object_init_with_type 先父后子→arm_cpu_post_init 属性注册→arm_cpu_realizefn 指令集初始化）、QOM 树路径（object_get_root 全局根、object_resolve_path 绝对/相对路径解析、/machine/cpu[0] 等典型路径）。

**适合读者**：需要理解 QEMU 类型系统、对象生命周期管理、类继承与接口机制、设备模型 realize 流程或 QOM 属性系统的开发者。  
**关键源文件**：`include/qom/object.h`（~700行）、`qom/object.c`（~2800行）、`include/qemu/module.h`（~80行）、`include/hw/core/qdev.h`（~300行）、`hw/core/qdev.c`（~800行）、`include/hw/core/sysbus.h`（~90行）、`hw/core/sysbus.c`（~340行）、`target/arm/cpu.c`（~2590行）

### [23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md](architecture/23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md)
> **23KB · 12 节**

主事件循环与协程全栈分析：AioContext 异步 I/O 上下文（GSource 集成、AioHandlerList fd 处理器链表、notify_me/notifier 跨线程唤醒、BHList Bottom Half 链表、scheduled_coroutines 协程调度链表、QEMUTimerListGroup 定时器组、poll_ns/poll_max_ns 自适应轮询参数、epollfd/fdmon_ops 三种监控后端 epoll/io_uring/poll）、AioHandler fd 处理器（io_read/io_write 回调、io_poll 用户态轮询、aio_set_fd_handler 注册→fdmon_ops->update 更新 epoll）、主事件循环（qemu_main_loop while 循环→main_loop_wait 单次迭代：gpollfds 重置→poll 观察者通知→timerlistgroup_deadline_ns 定时器截止→os_host_main_loop_wait ppoll 阻塞→qemu_clock_run_all_timers）、aio_poll 核心轮询（aio_compute_timeout→try_poll_mode 用户态 busy-polling→notify_me 设置→notified 检查→fdmon_ops->wait 内核态等待→aio_bh_poll BH 分发→aio_dispatch_ready_handlers fd 回调→timerlistgroup_run_timers 定时器→aio_free_deleted_handlers 清理、自适应轮询 poll_grow/poll_shrink 动态调整）、Bottom Half 延迟回调（QEMUBH ctx/name/cb/opaque/flags、aio_bh_enqueue 原子 fetch_or BH_PENDING→QSLIST_INSERT_HEAD_ATOMIC 无锁插入→aio_notify eventfd 唤醒、aio_bh_poll QSLIST_MOVE_ATOMIC 批量取出→逐个 aio_bh_call 执行→BH_DELETED/BH_ONESHOT 清理）、定时器系统（QEMUClockType REALTIME/VIRTUAL/HOST/VIRTUAL_RT 四种时钟、QEMUTimer expire_time/timer_list/cb/opaque/next/scale、QEMUTimerList 有序链表、QEMUTimerListGroup 每种时钟一个列表、timer_new/timer_mod/timer_del 操作、timerlist_run_timers 遍历→expire_time 比较→cb 调用、AioContext.tlg 集成→影响 aio_poll 阻塞超时）、Coroutine 协程核心（CoroutineAction YIELD/TERMINATE/ENTER 三状态、Coroutine entry/entry_arg/caller/ctx/locks_held/scheduled/co_queue_wakeup、qemu_coroutine_create 对象池获取→entry 绑定、qemu_coroutine_enter 上下文切换→执行→yield/terminate 返回、ucontext 后端 makecontext/swapcontext、sigaltstack 后端、Windows Fiber 后端、对象池回收 pool_get/pool_put 避免频繁分配）、CoMutex/CoQueue 协程同步（CoMutex locked/holder/queue、co_mutex_lock CAS 快速路径→co_queue_wait 慢速路径→yield、co_mutex_unlock holder 清除→co_queue_next→aio_co_wake 唤醒、CoQueue 协程等待队列 co_queue_wait_impl→QSIMPLEQ_INSERT_TAIL→yield、co_queue_next/restart_all 唤醒）、协程与 AioContext 集成（aio_co_schedule 设置 ctx→QSLIST_INSERT_HEAD_ATOMIC→qemu_bh_schedule co_schedule_bh 跨线程调度、co_schedule_bh_cb QSLIST_MOVE_ATOMIC→逐个 qemu_aio_coroutine_enter、aio_co_wake 同线程直接 enter/跨线程 aio_co_schedule）、defer_call 批量延迟调用（DeferredCall fn/opaque、DeferCallThreadState nesting_level/deferred_call_array 每线程、defer_call nesting=0 立即执行→否则去重+追加、defer_call_begin nesting++ 嵌套支持、defer_call_end nesting→0 时遍历执行+清空、VirtIO IOThread 中断批处理 virtio_irq→defer_call→单次 event_notifier_set）、EventNotifier（rfd/wfd eventfd 或 pipe 封装、event_notifier_init→eventfd/pipe2、event_notifier_set write 触发、event_notifier_test_and_clear read 消费、aio_set_event_notifier 封装为 fd handler 集成 AioContext）、IOThread 独立事件循环线程（IOThread QemuThread/ctx/worker_context/main_loop/running/run_gcontext、iothread_run rcu_register→g_main_context_push→qemu_set_current_aio_context→while(running) aio_poll→可选 g_main_loop_run、每 IOThread 独立 AioContext 实现 I/O 并行化避免 BQL 竞争）。

**适合读者**：需要理解 QEMU 事件驱动架构、异步 I/O 上下文、Bottom Half 延迟回调、定时器系统、协程生命周期与同步原语、IOThread 多线程模型或 defer_call 批处理优化的开发者。  
**关键源文件**：`include/qemu/aio.h`（~340行）、`util/aio-posix.c`（~800行）、`util/async.c`（~730行）、`util/main-loop.c`（~610行）、`system/runstate.c`（~955行）、`include/qemu/timer.h`（~550行）、`util/qemu-timer.c`（~610行）、`include/qemu/coroutine_int.h`（~72行）、`util/qemu-coroutine.c`（~400行）、`util/coroutine-ucontext.c`（~360行）、`util/qemu-coroutine-lock.c`（~470行）、`util/defer-call.c`（~157行）、`iothread.c`（~200行）

### [24-MemoryRegion-AddressSpace内存子系统深度分析-区域树-FlatView-地址分发与内存监听.md](architecture/24-MemoryRegion-AddressSpace内存子系统深度分析-区域树-FlatView-地址分发与内存监听.md)
> **25KB · 16 节**

MemoryRegion/AddressSpace 内存子系统全栈分析：MemoryRegion 核心结构（romd_mode/ram/readonly/is_iommu 类型标志、ops/opaque MMIO 回调、container/subregions 父子树、alias/alias_offset 别名窗口、priority 重叠优先级、size/addr 地址范围、terminates 分发终止、enabled 启用控制、dirty_log_mask 脏页跟踪）、MemoryRegionOps 回调接口（read/write 基础回调、read_with_attrs/write_with_attrs 带属性变体→MemTxResult 错误返回、endianness 字节序、valid.min/max_access_size Guest 可见约束、impl.min/max_access_size 内部实现约束→自动拆分/合并）、MemoryRegion 初始化函数族（memory_region_init 容器、memory_region_init_io MMIO 设备→ops/opaque/terminates、memory_region_init_ram 宿主内存→RAMBlock mmap、memory_region_init_alias 别名窗口→alias/alias_offset、memory_region_init_rom 只读 RAM→readonly=true、memory_region_init_iommu IOMMU 翻译层）、AddressSpace 地址空间（root 根 MemoryRegion、current_map FlatView RCU 快照、listeners MemoryListener 列表、max_bounce_buffer_size DMA bounce 上限、address_space_init 初始化→memory_region_ref→update_topology→update_ioeventfds）、FlatView 扁平化（FlatRange mr/offset_in_region/addr/dirty_log_mask/romd_mode/readonly、FlatView ref/ranges/nr/dispatch/root、MemoryRegionSection size/mr/fv/offset_within_region/offset_within_address_space）、拓扑生成（generate_memory_topology flatview_new→render_memory_region 递归渲染→flatview_simplify 合并→address_space_dispatch_new 构建基数树→flatview_add_to_dispatch→address_space_dispatch_compact、render_memory_region 递归遍历子区域和别名→高优先级覆盖低优先级→产出不重叠 FlatRange）、事务机制（memory_region_transaction_begin depth++、memory_region_transaction_commit depth→0 时→flatviews_reset 重生成→MEMORY_LISTENER_CALL_GLOBAL begin→address_space_set_flatview 对比新旧→region_add/region_del/region_nop→MEMORY_LISTENER_CALL_GLOBAL commit）、PhysPageEntry 基数树分发（skip:6 跳过位+ptr:26 索引、P_L2_SIZE=512 每节点、PhysPageMap nodes/sections 数组、AddressSpaceDispatch mru_section MRU 缓存+phys_map 根节点）、地址翻译路径（address_space_translate_internal 基数树查找→section→xlat 计算→plen 边界限制、flatview_do_translate 内部翻译→IOMMU 检测→address_space_translate_iommu 二次翻译、flatview_translate 顶层入口）、内存访问入口（flatview_write 翻译→memory_access_is_direct→RAM memcpy+invalidate_and_set_dirty/MMIO memory_region_dispatch_write、flatview_read 类似→RAM memcpy/MMIO dispatch_read、address_space_map RAM 直接返回宿主指针/MMIO bounce buffer、address_space_unmap bounce flush→write→释放）、MemoryListener 变更通知（begin/commit 事务回调、region_add/region_del/region_nop 区域增删、log_start/log_stop/log_sync/log_clear 脏页日志、log_global_start/log_global_stop 全局日志、eventfd_add/eventfd_del ioeventfd、priority 优先级排序、memory_listener_register 按优先级插入→通知现有区域、KVM 监听器 kvm_set_user_memory_region 创建/删除 memslot+KVM_MEM_LOG_DIRTY_PAGES）、子区域管理（memory_region_add_subregion priority=0 默认、memory_region_add_subregion_overlap 显式优先级、memory_region_update_container_subregions 按优先级降序插入→transaction_commit 触发拓扑更新）、RAMBlock 宿主内存（host 宿主虚拟地址 mmap、offset 全局 RAM 偏移、used_length/max_length 当前/最大长度、fd 文件后端/guest_memfd 机密 VM、bmap 脏页位图、clear_bmap 延迟清除位图、postcopy_length postcopy 长度）、IOMMUMemoryRegion（IOMMUMemoryRegionClass translate/get_min_page_size/notify_flag_changed/replay 虚函数、address_space_translate_iommu 递归翻译→多级 IOMMU 嵌套）、脏页追踪（dirty_log_mask 掩码→cpu_physical_memory_set_dirty_range→KVM_GET_DIRTY_LOG/dirty ring 同步→迁移增量传输/VGA 重绘/VFIO DMA）、全局地址空间（system_memory/system_io 全局根区域、address_space_memory/address_space_io 全局地址空间、memory_map_init 初始化→system UINT64_MAX 容器+io 65536 端口）。

**适合读者**：需要理解 QEMU 内存子系统架构、MemoryRegion 类型与层次、地址翻译与分发机制、FlatView 扁平化与拓扑更新、MemoryListener 通知链、KVM memslot 同步、RAMBlock 内存管理或 IOMMU/脏页追踪的开发者。  
**关键源文件**：`include/system/memory.h`（~2900行）、`system/memory.c`（~3700行）、`system/physmem.c`（~3900行）、`include/system/ramblock.h`（~130行）

### [25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md](architecture/25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md)
> **24KB · 18 节**

Block Layer I/O 子系统全栈分析：BlockDriverState 核心结构（drv/opaque 驱动 vtable 与私有数据、aio_context 事件循环、filename/backing_file 文件链、bl BlockLimits I/O 约束、children/backing/file/parents 节点图、total_sectors 磁盘容量、dirty_bitmaps 脏位图、copy_on_read CoR 引用计数、in_flight/serialising_in_flight 请求计数、reqs_lock/tracked_requests 请求追踪、flush_queue/active_flush_req flush 串行化、write_gen/flushed_gen 写入代追踪、block_status_cache RCU 缓存）、BlockDriver 驱动 vtable（format_name/protocol_name、bdrv_open/bdrv_close 生命周期、bdrv_co_preadv_part/bdrv_co_pwritev_part 新版 I/O 回调、bdrv_co_flush_to_os/to_disk 两级 flush、bdrv_co_pdiscard/bdrv_co_truncate/bdrv_co_block_status、bdrv_snapshot_create/goto/delete/list 快照、bdrv_check_perm/bdrv_set_perm/bdrv_child_perm 权限）、BlockBackend 设备前端（name/refcnt/root BdrvChild、ctx AioContext、dev DeviceState/dev_ops 设备模型、enable_write_cache 写缓存、perm/shared_perm 权限、throttle_group_member I/O 限流、in_flight/queued_requests 请求管理）、BdrvChild 父子关系（bs/name/klass/role/opaque、perm/shared_perm 权限授予、frozen 冻结标记、BdrvChildRoleBits DATA/METADATA/FILTERED/COW/PRIMARY/IMAGE 角色位）、节点图与典型链路（BlockBackend→root→格式 BDS→file→协议 BDS→实际 I/O、backing→COW 链递归）、权限系统（BLK_PERM_CONSISTENT_READ/WRITE/WRITE_UNCHANGED/RESIZE 常量、bdrv_child_perm 计算→bdrv_check_perm 检查→bdrv_set_perm 应用→file-posix fcntl/OFD 锁、父冲突检查+累计传播到子树）、BlockLimits I/O 约束（request_alignment/max_transfer/opt_transfer/max_pdiscard/max_pwrite_zeroes/min_mem_alignment/opt_mem_alignment/max_iov→自动拆分超大请求+对齐非对齐请求）、bdrv_open 打开路径（bdrv_open_common 解析选项→查找驱动→设置标志→bdrv_open_driver 分配 opaque→drv->bdrv_open→设置 supported_flags）、I/O 入口 BlockBackend 层（blk_co_preadv→blk_co_do_preadv_part：blk_wait_while_drained→blk_check_byte_request→bdrv_inc_in_flight→throttle_group_co_io_limits_intercept 限流→bdrv_co_preadv_part 下发→bdrv_dec_in_flight、写入类似+FUA 标志控制）、核心协程 I/O 路径（bdrv_driver_preadv 驱动分发→优先 bdrv_co_preadv_part→回退旧接口、bdrv_aligned_preadv 对齐验证→CoR 串行化→按 max_transfer 分块→bdrv_driver_preadv 循环、完整读取流程 blk_co_preadv→throttle→tracked_request_begin→pad→aligned_preadv→driver_preadv→drv->bdrv_co_preadv_part→递归到协议层→tracked_request_end）、请求追踪与串行化（BdrvTrackedRequest offset/bytes/type/serialising/overlap_offset/overlap_bytes/co/wait_queue/waiting_for、tracked_request_begin/end 追踪生命周期、tracked_request_set_serialising+bdrv_find_conflicting_request+bdrv_wait_serialising_requests_locked CoR 和写入的集群级串行化）、Copy-on-Read（bdrv_co_do_copy_on_readv 簇对齐→bdrv_co_is_allocated 检查→bounce buffer 从 backing 读取→bdrv_driver_pwritev 写入当前层→复制到 Guest 缓冲区）、写入路径与 Flush（bdrv_driver_pwritev 驱动分发+FUA 模拟→bdrv_co_flush、flush 链 blk→drv->bdrv_co_flush_to_os→递归 bdrv_co_flush(bs->file)→drv->bdrv_co_flush_to_disk fsync、enable_write_cache false→自动 FUA）、协程包装器模式（co_wrapper_mixed/co_wrapper 宏→block-coroutine-wrapper.py 自动生成→协程内直接调用/非协程创建协程+bdrv_poll_co 轮询）、I/O 限流（ThrottleGroupMember/ThrottleGroup/ThrottleState、throttle_group_co_io_limits_intercept 获取锁→next_throttle_token 轮转→throttle_group_schedule_timer→qemu_co_queue_wait 协程挂起→throttle_account 记账→schedule_next_request、位于 blk_co_do_preadv_part/pwritev_part 实际 I/O 前）、Drain 机制（bdrv_do_drained_begin quiesce_counter++→通知父节点→bdrv_drain_poll 等待 in_flight==0、bdrv_do_drained_end quiesce_counter--→唤醒排队请求、bdrv_drain_all_begin/end 全局 drain→图变更前必须）、Block Job 基础设施（Job id/driver/co/auto_finalize/auto_dismiss/cb/progress/aio_context/status、JobSTT 状态转移表 CREATED→RUNNING→WAITING→PENDING→CONCLUDING→NULL+PAUSED/ABORTING 分支、mirror/commit/stream/backup 四种任务类型）。

**适合读者**：需要理解 QEMU Block Layer 架构、BDS 节点图与父子关系、块设备 I/O 协程路径、请求追踪与串行化机制、Copy-on-Read 实现、I/O 限流与 Drain、Block Job 状态机或 bdrv_open 打开路径的开发者。  
**关键源文件**：`include/block/block_int-common.h`（~1300行）、`include/block/block-common.h`（~600行）、`block/block-backend.c`（~2400行）、`block/io.c`（~3500行）、`block.c`（~8000行）、`block/throttle-groups.c`（~700行）、`include/qemu/job.h`（~400行）

### [26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md](architecture/26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md)
> **22KB · 15 节**

VirtIO 设备模型全栈分析：VirtIODevice 核心结构（status/isr/queue_sel 状态寄存器、host_features/guest_features/backend_features 三级 Feature 集、config_len/config/config_vector 配置空间、nvectors/vq VirtQueue 数组、device_id 设备类型、started/start_on_kick 启动状态、use_guest_notifier_mask 通知掩码、dma_as DMA 地址空间、device_iotlb_enabled IOTLB）、VirtIODeviceClass 回调（realize/unrealize 生命周期、get_features/set_features/validate_features Feature 协商、get_config/set_config 配置空间、set_status/reset 状态控制、queue_reset/queue_enable 队列管理、guest_notifier_pending/mask 通知、start/stop_ioeventfd、save/load 迁移、get_vhost/toggle_device_iotlb）、VirtQueue 结构（vring VRing 描述符环、used_elems Used 缓存、last_avail_idx/shadow_avail_idx 消费者索引、used_idx/used_wrap_counter Used 索引、signalled_used/signalled_used_valid 通知抑制、notification 启用标志、queue_index/inuse/vector、handle_output kick 回调、guest_notifier/host_notifier EventNotifier）、VRing 描述符环（Split 格式 VRingDesc addr/len/flags/next→VRingAvail flags/idx/ring→VRingUsed flags/idx/ring、VRing num/num_default/align/desc/avail/used GPA+caches、内存布局 desc_table→avail_ring→used_ring）、VirtQueueElement 请求元素（index/len/ndescs/out_num/in_num/in_order_filled、in_addr/out_addr GPA 数组、in_sg/out_sg iovec HVA）、Virtqueue 操作循环（virtqueue_pop 从 Avail 环取请求→遍历描述符链→address_space_map GPA→HVA、virtqueue_fill 写入 Used、virtqueue_flush 更新 used->idx+内存屏障、virtqueue_push=fill+flush 一步完成→virtio_notify 通知 Guest）、通知机制（Guest→Host Kick：无 ioeventfd→MMIO 退出→virtio_queue_notify→handle_output/有 ioeventfd→KVM eventfd→host_notifier→AioContext→handle_output、Host→Guest Notify：virtio_notify→virtio_should_notify 通知抑制→virtio_irq→主线程 virtio_notify_vector MSI-X/IOThread defer_call→irqfd 批量延迟、EVENT_IDX 通知抑制 avail_event/used_event 减少 VM Exit 和中断）、Packed Virtqueue（VRingPackedDesc addr/len/id/flags AVAIL/USED 嵌入描述符+wrap counter、VRingPackedDescEvent off_wrap/flags、virtqueue_packed_pop/fill/flush、无独立 avail/used 环 cache 友好）、VirtIO-PCI 传输层（VirtIOPCIProxy pci_dev/bar/common/isr/device/notify/notify_pio MemoryRegion、guest_features/vqs/vector_irqfd/bus、virtio_pci_realize PCI 初始化+MSI-X+BAR 注册、virtio_pci_common_read/write MMIO 配置访问、virtio_pci_ioeventfd_assign kick 路径设置）、VirtioBus 总线抽象（VirtioBusClass notify/save_config/load_config/set_guest_notifiers/device_plugged/unplugged/ioeventfd_enabled/assign/get_dma_as/iommu_enabled、VirtioBusState ioeventfd_started/grabbed）、virtio-blk 块设备（VirtIOBlock blk BlockBackend/conf/dataplane、VirtIOBlockReq sector_num/dev/vq/in/outhdr/qiov/elem、virtio_blk_device_realize virtio_init+关联 blk+virtio_add_queue、virtio_blk_handle_vq 循环 virtqueue_pop→解析请求→virtio_blk_submit_multireq 批量合并→blk_aio_preadv/pwritev）、virtio-net 网络设备（VirtIONet nic/vqs/max_queue_pairs/curr_queue_pairs 多队列、virtio_net_device_realize 队列对分配+rx/tx VirtQueue 创建、virtio_net_handle_rx 接收、virtio_net_handle_tx_timer/tx_bh timer/BH 两种发送模式）、IOThread Dataplane 集成（virtio_blk_vq_aio_context_init 每队列 AioContext 设置、blk_set_aio_context BlockBackend 绑定 IOThread、ioeventfd Guest kick 直达 IOThread→handle_output→blk_aio→I/O 完成→virtqueue_push→defer_call→irqfd MSI-X、避免 BQL 竞争）、Feature 协商（VIRTIO_F_VERSION_1/RING_PACKED/IN_ORDER/NOTIFICATION_DATA/RING_F_INDIRECT_DESC/EVENT_IDX、协商流程 host_features→Guest 读取→Guest 写 guest_features→FEATURES_OK→validate）。

**适合读者**：需要理解 VirtIO 规范在 QEMU 中的实现、VirtQueue 描述符环操作、Split/Packed 两种队列模式、Guest↔Host 双向通知机制（ioeventfd/irqfd/defer_call）、VirtIO-PCI 传输层、virtio-blk/net 设备处理流程或 IOThread dataplane 集成的开发者。  
**关键源文件**：`include/hw/virtio/virtio.h`（~450行）、`hw/virtio/virtio.c`（~4300行）、`include/hw/virtio/virtio-pci.h`（~160行）、`hw/virtio/virtio-pci.c`（~2400行）、`include/hw/virtio/virtio-bus.h`（~120行）、`hw/block/virtio-blk.c`（~1900行）、`hw/net/virtio-net.c`（~4000行）

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

### [22-ARM64异常级别状态管理与指令执行变化深度分析.md](arm64/22-ARM64异常级别状态管理与指令执行变化深度分析.md)
> **15KB · 11 节**

ARM64 异常级别（EL0-EL3）切换时 CPU 指令执行行为变化深度分析：TB Flags EL 编码（MMUIDX 间接编码 EL、rebuild_hflags_a64 全部 EL 相关标志 SVEEXC_EL/SMEEXC_EL/UNPRIV/TRAP_ERET/NV/E2H/PAUTH_ACTIVE/BT/MTE_ACTIVE）、DisasContext EL 字段（current_el/fp_excp_el/sve_excp_el/sme_excp_el/trap_eret/e2h/nv/nv1/nv2）、系统寄存器访问门控（CPAccessRights PL0-PL3 位图层级包含设计、cp_access_ok 静态检查、accessfn 动态陷阱、VHE 重定向）、每个 EL 指令可用性（EL0 用户态限制、EL1 PL1 寄存器+HCR 陷阱控制、EL2 Stage-2/VHE、EL3 SCR/RME 全权限）、异常入口完整流程（arm_cpu_do_interrupt_aarch64 VBAR 向量表 4×4 布局、SPSR/ELR 保存、ESR/FAR 综合信息、PSTATE 重构 DAIF/PAN/TCO/SSBS/ALLINT）、异常返回 ERET（helper_exception_return 7 种非法返回检测、SPSR→PSTATE 恢复、SP/TBI/SVE 调整、illegal_return PSTATE.IL 设置）、MMU Regime 与 EL 映射（6 种 regime、TGE/DC 翻译禁用、SP_EL0/SP_ELx 选择）、HCR_EL2 陷阱效应（TVM/TRVM/TSW/TACR/TTLB/TID/TSC/TWI/TWE 全表、TB Flags 缓存策略）、EL 切换状态变化对比表。

**适合读者**：需要理解 ARM64 EL 状态转换如何改变指令执行行为、TB Flags 编码、异常入口/返回完整机制的开发者。
**关键源文件**：`target/arm/tcg/hflags.c`（~500行）、`target/arm/tcg/translate-a64.c`（~11000行）、`target/arm/tcg/translate.h`（~210行）、`target/arm/tcg/helper-a64.c`（~800行）、`target/arm/helper.c`（~10188行）、`target/arm/cpregs.h`（~600行）

### [23-ARM64安全与非安全中断路由流转深度分析.md](arm64/23-ARM64安全与非安全中断路由流转深度分析.md)
> **13KB · 11 节**

ARM64 安全/非安全中断在不同异常级别和安全状态下的完整路由与流转深度分析：GICv3 三个中断组模型（Group 0/G1S/G1NS → FIQ/IRQ 信号映射规则、跨安全世界始终 FIQ）、GIC 分发器安全访问限制（mask_group_and_nsacr NS 对 G0/G1S 的 RAZ/WI 过滤、GICD_CTLR.DS 安全禁用）、CPU 接口核心路由决策（gicv3_cpuif_update 组→信号映射表、优先级抢占 icc_hppi_can_preempt、PMR 屏蔽、无 EL3 降级 G1S→G0）、中断目标 EL 确定（arm_phys_excp_target_el 六维 target_el_table 查表、SCR_EL3.IRQ/FIQ/EA + HCR_EL2.IMO/FMO/AMO + TGE 折叠）、中断屏蔽机制（arm_excp_unmasked PSTATE.DAIF、高 EL 路由不可屏蔽规则、NMI/ALLINT/SPINTMASK、E2H+TGE 例外）、CPU 中断调度优先级（arm_cpu_exec_interrupt NMI→VINMI→VFNMI→FIQ→IRQ→VIRQ→VFIQ→VSERR、NMI 降级处理）、虚拟中断机制（icv_access HCR.IMO/FMO 触发 ICC→ICV 重定向、ICH_LR List Registers 注入、HCR.VI/VF 直接注入、虚拟中断 target_el=1）、五大完整中断流转场景（NS IRQ 在 NS EL1 正常路径、Secure FIQ 在 NS EL1 跨世界、NS IRQ 在 Secure EL1、虚拟化 NS EL1 Guest + EL2 Hypervisor、Secure G1S 在 NS EL2）、优先级空间分离（NS [0x80-0xFF] 半空间、Secure 天然高优先级）、中断向量偏移（IRQ +0x80、FIQ +0x100、SError +0x180）。

**适合读者**：需要理解 ARM64 安全/非安全中断在不同 EL 和安全状态下如何路由、屏蔽、流转，以及虚拟中断注入机制的开发者。
**关键源文件**：`hw/intc/arm_gicv3_cpuif.c`（~2300行）、`hw/intc/arm_gicv3_dist.c`（~2000行）、`target/arm/cpu-irq.c`（~270行）、`target/arm/helper.c`（~10188行）

### [24-GICv3完整中断生命周期深度分析.md](arm64/24-GICv3完整中断生命周期深度分析.md)
> **18KB · 11 节**

GICv3 中断从设备触发到 CPU 处理完成的完整生命周期深度分析：GPIO 连线与中断输入（gicv3_init_irqs_and_mmio SPI/PPI GPIO + CPU 输出 IRQ/FIQ/NMI 线、gicv3_set_irq SPI→GICD / PPI→GICR 分派）、SPI 端到端（gicv3_dist_set_irq 电平/边沿 pending 更新、gicd_int_pending 四重位操作过滤 enable+active+group、gicv3_update_noirqset gicd_irouter_target 路由缓存+irqbetter 优先级比较、gicv3_cpuif_update 信号 CPU）、SGI 生命周期（icc_generate_sgi Aff3:2:1+targetlist+IRM 解码、gicv3_redist_send_sgi 组匹配+NSACR 检查+pending 设置）、PPI 生命周期（gicv3_redist_set_irq 电平追踪+边沿锁存、Timer/PMU PPI 映射表）、优先级与抢占（icc_gprio_mask BPR 组/子优先级分割、icc_highest_active_prio APR 位图扫描+NMI 特殊、icc_hppi_can_preempt PMR→运行优先级→组优先级比较链）、中断状态机四状态（Inactive→Pending→Active→Active+Pending 转换图、icc_activate_irq SGI/PPI/SPI/LPI 分支、电平 vs 边沿触发 pending 差异）、CPU 应答（icc_iar0/1_read 抢占检查+activate+返回 INTID、NMI→INTID_NMI）、EOI（icc_eoir_write 组匹配+安全检查+icc_drop_prio+icc_deactivate_irq、EOImode=0 一步 vs EOImode=1 分离 ICC_DIR）、完整 SPI 10 步流程图。

**适合读者**：需要理解 GICv3 中断从触发到完成的完整数据流、状态机转换、优先级抢占机制的开发者。
**关键源文件**：`hw/intc/arm_gicv3.c`（~480行）、`hw/intc/arm_gicv3_cpuif.c`（~2300行）、`hw/intc/arm_gicv3_dist.c`（~2000行）、`hw/intc/arm_gicv3_redist.c`（~1200行）、`hw/intc/arm_gicv3_common.c`（~400行）

### [25-GICv3-ITS中断翻译服务与LPI深度分析.md](arm64/25-GICv3-ITS中断翻译服务与LPI深度分析.md)
> **17KB · 11 节**

GICv3 ITS（中断翻译服务）和 LPI（局部性外设中断）实现深度分析：ITS 架构（QOM 类型 arm-gicv3-its、MMIO 控制+翻译区域、MSI→ITS→GICR→CPU 投递链路）、核心数据结构（DTEntry 设备表、CTEntry 集合表、ITEntry 中断翻译表、VTEntry 虚拟 PE 表）、表管理（GITS_BASER 8 寄存器配置、table_entry_addr 平坦/二级表地址计算、L1→L2 间接寻址）、命令队列（GITS_CBASER/CWRITER/CREADR 环形缓冲、process_cmdq 主循环、CMD_STALL 停滞处理）、14 种命令详解（MAPD 设备映射、MAPC 集合映射、MAPTI/MAPI 中断翻译映射、INT 软件触发、INV/INVALL 缓存失效、MOVI/MOVALL LPI 迁移、DISCARD/CLEAR 清除、VMAPTI/VMAPP/VMOVI/VINVALL GICv4 虚拟化命令）、MSI→LPI 完整 11 步流程（GITS_TRANSLATER 写入→do_process_its_cmd 三级翻译→process_its_cmd_phys 集合查找→gicv3_redist_process_lpi LPI 投递→pending 表更新→CPU 接口→应答→EOI）、LPI 配置表（GICR_PROPBASER 1字节/LPI enable+priority、NS 优先级移位）、LPI pending 表（GICR_PENDBASER 1位/LPI、gicv3_redist_update_lpi_only 全量扫描）、LPI vs SPI/PPI/SGI 对比（始终 G1NS、始终边沿、无 Active 位）、vLPI 与 GICv4（process_its_cmd_virt vPE 表查找、doorbell 中断）、KVM 直通 ITS。

**适合读者**：需要理解 MSI/MSI-X 如何通过 ITS 翻译为 LPI、ITS 命令队列和表管理、LPI 配置和 pending 机制的开发者。
**关键源文件**：`hw/intc/arm_gicv3_its.c`（~2050行）、`hw/intc/arm_gicv3_its_common.c`（~130行）、`hw/intc/arm_gicv3_redist.c`（~1200行）、`include/hw/intc/arm_gicv3_its_common.h`（~380行）

### [26-GICv3寄存器模拟与状态机深度分析.md](arm64/26-GICv3寄存器模拟与状态机深度分析.md)
> **16KB · 11 节**

GICv3 寄存器模拟与状态机深度分析：GICD 分发器寄存器（CTLR 读写、DS 单向转换、TYPER 动态计算、NS 访问过滤 RAZ/WI）、GICD_IPRIORITYR NS 优先级空间分离（NS 读 (prio<<1)&0xff、NS 写 0x80|(value>>1)、安全 [0x00-0xFF] vs 非安全 [0x80-0xFF]）、GICR 重分发器寄存器（TYPER affinity/Last/PLPIS、PROPBASER/PENDBASER LPI 配置/pending 表、WAKER 处理器睡眠协议）、ICC CPU 接口系统寄存器（gicv3_cpuif_reginfo[] 2463+ 注册表、ARMCPRegInfo opc0/opc1/crn/crm/opc2、accessfn 访问控制 gicv3_fiq/irq_access、readfn/writefn 读写回调）、ICV 虚拟化重定向（icv_access() 85-106 检查 NS EL1+HCR.IMO/FMO、透明转发到 icv_* 虚拟寄存器）、ICH 虚拟化控制寄存器（ich_hcr/vmcr/lr/ap1r、List Register 64 位 state/prio/group/vINTID）、状态位存储架构（SPI 位图 GICD、SGI/PPI 位图 GICR、per-CPU 组合打包）、VMSTATE 迁移（vmstate_gicv3_cpu 187-222 主状态、virt/sre_el1/gicv4/nmi 条件子段）、写入副作用级联（GICD_CTLR→gicv3_full_update、组/使能/pending→部分 update、GICR→per-CPU redist_update、ICC→per-CPU cpuif_update）。

**适合读者**：需要理解 GICv3 各级寄存器的 QEMU 实现细节、NS/S 隔离机制、ICV 虚拟化重定向、迁移状态保存的开发者。
**关键源文件**：`hw/intc/arm_gicv3_dist.c`（~820行）、`hw/intc/arm_gicv3_cpuif.c`（~2700行）、`hw/intc/arm_gicv3_redist.c`（~1200行）、`hw/intc/arm_gicv3_common.c`（~680行）

### [27-ARM64中断虚拟化ICH-ICV-LR状态机深度分析.md](arm64/27-ARM64中断虚拟化ICH-ICV-LR状态机深度分析.md)
> **13KB · 12 节**

ARM GICv3 中断虚拟化完整实现分析：ICV 重定向机制（icv_access() 85-106 检查 NS EL1+HCR.IMO/FMO、四类寄存器重定向规则、Guest 完全无感知）、List Register 格式与状态机（GICv3 ICH_LR 64 位 State/HW/Group/NMI/Priority/EOI/pINTID/vINTID、GICv2 GICH_LR 32 位对比、四状态 Invalid→Pending→Active→PendingActive）、ICV 寄存器模拟（icv_iar_read 800-842 虚拟应答 hppvi_index+icv_activate_irq、icv_eoir_write 1584-1645 icv_drop_prio+icv_deactivate_irq、icv_dir_write 1551-1582 split EOI deactivate、ICV_PMR/BPR/CTLR/IGRPEN 映射到 ICH_VMCR）、ICH 控制寄存器（ICH_HCR En/UIE/NPIE/LRENPIE/TC/TALL0/TALL1/TDIR/EOIcount、ICH_VMCR VENG0/VENG1/VPMR/VBPR）、虚拟中断投递（gicv3_cpuif_virt_irq_fiq_update 471-524 G0→vFIQ/G1→vIRQ/NMI→vNMI）、维护中断（6 种触发条件 EOI/U/NP/LRENP/VGRP、maintenance_interrupt_state 434-469、递归回 GIC PPI）、最高优先级选择（hppvi_index 179-256 LR+vLPI 比较、ich_highest_active_virt_prio 154-177 APR 扫描）、GICv2 虚拟接口（GICH MMIO gic_hyp_read/write 1909-2020、GICV 共用读写函数、gic_update_internal(true)）、GICv4 vLPI 直接注入（绕过 LR、VPROPBASER/VPENDBASER 配置表）、完整虚拟中断生命周期 8 步流程。

**适合读者**：需要理解 ARM 中断虚拟化实现、ICH/ICV 寄存器交互、LR 状态机、维护中断机制的开发者。
**关键源文件**：`hw/intc/arm_gicv3_cpuif.c`（~2700行）、`hw/intc/gicv3_internal.h`、`hw/intc/arm_gic.c`（~2200行）、`hw/intc/gic_internal.h`

### [28-KVM-vGIC设备后端与中断直通深度分析.md](arm64/28-KVM-vGIC设备后端与中断直通深度分析.md)
> **10KB · 11 节**

KVM vGIC 设备后端完整实现分析：KVM GICv2 后端（kvm_arm_gic_realize 490-585 KVM_DEV_TYPE_ARM_VGIC_V2 创建、GICD/GICC MMIO 委托、kvm_arm_gic_set_irq 44-73 中断注入转换、kvm_arm_gic_put/get 288-474 DIST_REGS+CPU_REGS 迁移）、KVM GICv3 后端（kvm_arm_gicv3_realize 786-950 KVM_DEV_TYPE_ARM_VGIC_V3 创建、五组状态访问 DIST/REDIST/CPU_SYSREGS/LEVEL_INFO/ITS、kvm_arm_gicv3_put/get 317-620 完整 GICD+GICR+ICC 迁移）、KVM ITS 后端（kvm_arm_its_realize 92-128 独立 KVM 设备、SAVE/RESTORE_TABLES 命令、irqfd MSI 直通）、机器模型 GIC 创建（create_gic 1122-1293 类型选择+MMIO 映射+中断连线）、KVM IRQ 注入（kvm_arm_set_irq→KVM_IRQ_LINE ioctl+irqfd 高速路径）、TCG vs KVM 功能对比（安全扩展/NMI/性能/调试差异）、9 组迁移状态总览、完整 GICv3 KVM 迁移保存/恢复流程。

**适合读者**：需要理解 KVM vGIC 工作原理、中断注入路径、KVM 迁移状态管理的开发者。
**关键源文件**：`hw/intc/arm_gic_kvm.c`（~610行）、`hw/intc/arm_gicv3_kvm.c`（~975行）、`hw/intc/arm_gicv3_its_kvm.c`（~265行）、`hw/arm/virt.c`（~4300行）

### [29-ARM64系统寄存器模拟框架深度分析.md](arm64/29-ARM64系统寄存器模拟框架深度分析.md)
> **16KB · 14 节**

ARM64 系统寄存器完整模拟框架：ARMCPRegInfo 结构体（编码/权限/回调/存储偏移/FGT/VHE重定向）、AArch64/AArch32 编码宏与哈希查找（O(1) GHashTable）、寄存器表组织（v8_cp_reginfo/el2/el3 数组）、MSR/MRS 翻译路径（handle_sys → Helper 分发 → writefn 回调）、四级访问控制（PLx 权限 + accessfn + FGT + HSTR）、关键寄存器写副作用（SCTLR→TLB刷新、HCR→虚拟中断+TLB、TCR/TTBR→条件TLB刷新、DAIF→中断掩码）、PSTATE 寄存器（NZCV/DAIF/SPSel）、迁移保存恢复（cpreg_vmstate 变长数组）、完整 MSR 执行流程图。

**适合读者**：需要理解系统寄存器访问如何被拦截、分发和处理的开发者，或需要添加新系统寄存器支持的开发者。
**关键源文件**：`target/arm/cpregs.h`（~1100行）、`target/arm/helper.c`（~8500行）、`target/arm/tcg/translate-a64.c`、`target/arm/tcg/op_helper.c`、`target/arm/machine.c`

### [30-ARM64-MMU系统寄存器与页表遍历深度分析.md](arm64/30-ARM64-MMU系统寄存器与页表遍历深度分析.md)
> **10KB · 9 节**

MMU 核心寄存器详解：TCR_EL1 完整位域（T0SZ/T1SZ/TG0/TG1/IPS/EPD/A1 等）、TTBR0/1_EL1 布局（ASID[63:48] + BADDR + CnP）、MAIR_EL1 属性槽（8×8位 Device/Normal 编码）、VTCR_EL2 Stage-2 配置（SL0 起始级别、T0SZ IPA 空间）、页表遍历完整流程（get_phys_addr_lpae 逐级查表：TCR 解码→TTBR 选择→描述符遍历→权限/属性解析）、Stage-1 + Stage-2 两级翻译路径、翻译体制选择（EL10/E2/E20/SE3）、TLBI 指令实现（IS 跨核同步、HCR_EL2.TTLB 陷阱）、QEMU TLB 简化（不跟踪 ASID、忽略 SH/IRGN/ORGN）。

**适合读者**：需要理解 QEMU 如何实现 ARM64 MMU 地址翻译或分析 TLB 性能的开发者。
**关键源文件**：`target/arm/ptw.c`（~2700行）、`target/arm/helper.c`、`target/arm/internals.h`、`target/arm/tcg/tlb-insns.c`

### [31-ARM64-EL2-EL3系统寄存器陷阱路由深度分析.md](arm64/31-ARM64-EL2-EL3系统寄存器陷阱路由深度分析.md)
> **10KB · 9 节**

完整陷阱路由框架：HCR_EL2 全部 56+ 陷阱位分类（VM 寄存器 TVM/TRVM、中断路由 FMO/IMO/AMO、指令 TWI/TWE/TSC、TLB TTLB/TTLBIS、ID TID0-5、Cache TSW/TPCP、MMU VM/DC/E2H/NV）、SCR_EL3 安全位（NS/IRQ/FIQ/EA 路由、SMD/HCE/RW/FGTEN）、CPTR_EL2/EL3 三级 FP/SIMD/SVE 陷阱链、FEAT_FGT 细粒度陷阱（HFGRTR/HFGWTR 逐寄存器位控制 vs HCR.TVM 粗粒度对比）、accessfn 实现模式、陷阱→异常流程（syndrome 生成→raise_exception→TGE 覆盖）、四级检查优先级（EL 权限→HSTR→FGT→accessfn）。

**适合读者**：需要理解虚拟化陷阱路由机制、实现新陷阱位或分析 VM Exit 路径的开发者。
**关键源文件**：`target/arm/cpu.h`（HCR/SCR 定义）、`target/arm/cpregs.h`（FGT）、`target/arm/helper.c`（accessfn）、`target/arm/tcg/op_helper.c`（异常流程）

### [32-ARM64特殊系统寄存器与Cache-AT指令深度分析.md](arm64/32-ARM64特殊系统寄存器与Cache-AT指令深度分析.md)
> **9KB · 12 节**

CPACR_EL1 三级 FP/SIMD 陷阱链（FPEN/ZEN/SMEN → CPTR_EL2 → CPTR_EL3）、CONTEXTIDR_EL1 上下文 ID 与 ASID 刷新策略（VMSA 短描述符时 TLB flush）、PAR_EL1 + AT 指令完整实现（do_ats_write 翻译流程→PAR 写入→Stage-2 fault 异常变换）、TPIDR 三寄存器（EL0 RW / EL0 RO / EL1 RW → Linux TLS/current 指针）、FPCR/FPSR 舍入模式与 softfloat 交互、Timer CNTVCT/CNTFRQ/CNTHCTL 陷阱、DC ZVA memset 实现 + HCR_EL2.TDZ/TSW/TPCP/TPU Cache 维护陷阱、QEMU 不模拟缓存但保留陷阱语义。

**适合读者**：需要理解特殊系统寄存器功能、AT 地址翻译指令实现或 Cache 维护指令虚拟化的开发者。
**关键源文件**：`target/arm/helper.c`（寄存器定义/写回调）、`target/arm/tcg/cpregs-at.c`（AT 指令）、`target/arm/tcg/vfp_helper.c`（FPCR/FPSR）

### [33-ARM64-Debug-Breakpoint-Watchpoint与RAS寄存器深度分析.md](arm64/33-ARM64-Debug-Breakpoint-Watchpoint与RAS寄存器深度分析.md)
> **11KB · 12 节**

完整调试架构实现：三级调试陷阱体系（TDOSA/TDA/TDCC × EL1→EL2→EL3）、MDSCR_EL1 全局开关（MDE/SS/TDCC）、MDCR_EL2/EL3 陷阱路由（TDE 整体重路由）、OS 锁（OSLAR/OSLSR/OSDLR）、断点动态注册（DBGBVRn/BCRn → hw_breakpoint_update → cpu_breakpoint_insert）、观察点（DBGWVRn/WCRn 支持 BAS 字节选择和 MASK 掩码最大 2GB）、统一匹配框架 bp_wp_matches()（SSC 安全过滤→PAC/HMC EL 过滤→链接断点）、调试异常路由 arm_debug_target_el()、最小 RAS（ERRIDR=0 零记录、DISR/VDISR/VSESR SError 报告注入）。

**适合读者**：需要理解 ARM 硬件调试机制、断点/观察点实现原理或 RAS 错误处理的开发者。
**关键源文件**：`target/arm/debug_helper.c`（寄存器定义/访问控制）、`target/arm/tcg/debug.c`（匹配逻辑/异常处理）、`target/arm/helper.c`（RAS/MDCR）

### [34-ARM64-ID寄存器与特性发现机制深度分析.md](arm64/34-ARM64-ID寄存器与特性发现机制深度分析.md)
> **9KB · 12 节**

ARM64 ID 寄存器完整体系：4 组 ×8 个 AArch64 ID 空间（PFR/DFR/ISAR/MMFR + ZFR/SMFR）、ARMISARegisters 统一存储与 GET_IDREG 宏、isar_feature_aa64_* 数百个内联特性检查函数与 cpu_isar_feature 宏、CPU 模型初始化（具名模型 vs `-cpu max` TCG 全特性填充）、HCR_EL2.TID0-TID5 分组陷阱控制（access_tid3 核心 ID 陷阱→Hypervisor 可伪造值）、用户态 exported_bits/fixed_bits 过滤机制（modify_arm_cp_regs）、未实现 ID 寄存器 RAZ 处理、ID_AA64PFR0.GIC 动态字段运行时填充。

**适合读者**：需要理解 ARM 特性发现机制、实现新 CPU 模型或分析 ID 寄存器虚拟化的开发者。
**关键源文件**：`target/arm/helper.c`（ID 寄存器注册 6240-6760）、`target/arm/cpu-features.h`（isar_feature 检查）、`target/arm/cpu64.c`（CPU 模型初始化）

### [35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md](arm64/35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md)
> **19KB · 12 节**

ARM64 异常级别状态切换全解：CPUARMState 中 PSTATE/ELR/SPSR/SP 的分布式存储与缓存策略、PSTATE 26+ 位域定义（NZCV/DAIF/CurrentEL/SPSel/PAN/UAO/DIT/TCO/IL/SS/ALLINT）、pstate_read()/write() 汇聚/分发实现、arm_cpu_do_interrupt_aarch64() 异常进入主流程（VBAR 基址 + 来源/类型双维度偏移 16 向量）、ESR_ELx 综合征寄存器设置（syn_aa64_svc/hvc/smc）、异常进入 PSTATE 变化（DAIF 全屏蔽/PAN-SPAN/TCO/SSBS/ALLINT）、HELPER(exception_return) 异常返回（SPSR→PSTATE/ELR→PC/6 种非法返回判定）、SCR_EL3/HCR_EL2 路由位与中断转发、AArch32↔AArch64 互操作（sync_32_to_64/64_to_32）、TCG 翻译 SVC/HVC/SMC/ERET。

**适合读者**：需要理解 ARM 异常机制、实现安全监控/虚拟化或调试 EL 切换问题的开发者。  
**关键源文件**：`target/arm/helper.c`、`target/arm/tcg/helper-a64.c`、`target/arm/cpu.h`、`target/arm/syndrome.h`、`target/arm/tcg/translate-a64.c`

### [36-ARM64不同EL下指令执行流变化深度分析.md](arm64/36-ARM64不同EL下指令执行流变化深度分析.md)
> **15KB · 12 节**

ARM64 不同异常级别（EL0-EL3）下指令执行流差异全解：rebuild_hflags_a64() EL 敏感的 TB 标志计算（E2H/UNPRIV/SVE/SME/NV/MTE/FGT）、DisasContext EL 感知翻译（current_el/mmu_idx/fp_excp_el 等）、ARMCPRegInfo 权限模型（PL0-PL3/accessfn）、access_tvm_trvm/tsw/tacr 系统寄存器陷阱路由、arm_mmu_idx_el() MMU 体制映射（E10/E20/E2/E3 七种模式）、PAN/UAO 内存访问权限控制、FP/SIMD/SVE/SME 三级陷阱控制（CPACR/CPTR_EL2/CPTR_EL3）与向量长度随 EL 变化、DAIF/ALLINT 中断屏蔽、debug_helper.c 调试寄存器三级路由（TDOSA/TDRA/TDA）、VHE 寄存器重定向表、各 EL 指令可用性与 MMU/陷阱层次对比。

**适合读者**：需要理解不同 EL 下 CPU 行为差异、实现虚拟化陷阱或调试权限问题的开发者。  
**关键源文件**：`target/arm/tcg/hflags.c`、`target/arm/tcg/translate.h`、`target/arm/cpregs.h`、`target/arm/helper.c`、`target/arm/debug_helper.c`

### [37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md](arm64/37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md)
> **21KB · 17 节**

ARM64 安全状态转换全栈分析：ARMSecuritySpace 四域安全模型（Secure/NonSecure/Root/Realm 对应 NSE:NS 编码）、SCR_EL3 安全配置寄存器位域（NS/IRQ/FIQ/EA/SMD/HCE/RW/EEL2/NSE 等 40+ 位定义与特性门控）、安全状态判定（arm_security_space EL3→Root/Secure + arm_security_space_below_el3 SCR_NS/SCR_NSE 三分支）、HCR_EL2 Hypervisor 配置位域（VM/FMO/IMO/AMO/TSC/HCD/TGE/RW/E2H/NV 等）、arm_hcr_el2_eff 有效值遮罩（安全态+无 SEL2→返回 0 完全无效、AArch32 过滤、TGE+E2H 清除虚拟化位、TGE 非 E2H 强制 FMO/IMO/AMO）、SCR_EL3 写入回调（scr_write 特性门控 valid_mask、NS/NSE 变化→全 EL3 以下 TLB 刷新 12 种 MMU 索引）、中断路由联动（arm_phys_excp_target_el 6 维查找表 target_el_table[is64][scr][rw][hcr][secure][cur_el]、SCR_IRQ/FIQ/EA 路由 EL3 优先于 HCR_IMO/FMO/AMO 路由 EL2、TGE 折叠）、中断屏蔽与安全状态（VINMI/VFIQ/VSERR 仅虚拟化激活有效、目标 EL3 不可屏蔽、VHE 模式 EL2 可屏蔽、AArch32 SCR/HCR 覆盖 CPSR）、SMC 异常路由（SCR_SMD 禁用、HCR_TSC NS EL1 陷阱到 EL2 优先、PSCI 旁路、AArch64 SMD 安全态也生效 vs AArch32 仅非安全态）、HVC 异常路由（SCR_HCE 优先于 HCR_HCD、安全态 AArch32/安全态 EL1 AArch64 UNDEF）、异常进入安全状态（arm_cpu_do_interrupt_aarch64 向量偏移 SCR_RW→+0x400/+0x600）、ERET 安全性校验（RME NSE=1+NS=0 保留非法、arm_el_is_aa64 宽度匹配、TGE EL1 返回禁止）、arm_el_is_aa64 寄存器宽度链（EL3 固定→SCR_RW 控制 EL2→HCR_RW 控制 EL1、arm_scr_rw_eff 安全态 SEL2 感知）、Secure EL2（arm_is_el2_enabled_secstate 安全态需 SCR_EEL2）、VHE 寄存器重定向（vhe_redir_to_el2/el01 SCTLR/TTBR/TCR/VBAR 等）、Stage 2 MMU（HCR_VM/DC 使能、安全态 HCR 无效时自动禁用）。

**适合读者**：需要理解 ARM64 安全世界切换、SCR_EL3/HCR_EL2 联动控制、中断路由优先级、SMC/HVC 异常路由或 Secure EL2 实现的开发者。  
**关键源文件**：`include/hw/arm/arm-security.h`（~37行）、`target/arm/cpu.h`（~2250行）、`target/arm/internals.h`（~490行）、`target/arm/helper.c`（~10190行）、`target/arm/tcg/helper-a64.c`（~730行）、`target/arm/tcg/op_helper.c`（~1200行）、`target/arm/cpu-irq.c`（~165行）

### [38-ARM64内存管理深度分析-页表遍历-TLB-Stage2翻译与属性合并.md](arm64/38-ARM64内存管理深度分析-页表遍历-TLB-Stage2翻译与属性合并.md)
> **18KB · 16 节**

ARM64 内存管理全栈分析：ARMMMUIdx 翻译体制索引（E10_0/E10_1/E20_0/E20_2/E2/E3/Stage2/Stage2_S/Phys 共 22 种索引 + Stage1_E0/E1 仅 AT 指令使用）、arm_mmu_idx_el 翻译体制选择（EL0→E10_0/E20_0/E30_0、EL1→E10_1/PAN、EL2→E2/E20_2/PAN、EL3→E3/PAN + VHE 重定向）、get_phys_addr 入口（构建 S1Translate → get_phys_addr_gpc）、ARMMMUFaultInfo 故障信息（type/level/stage2/s1ptw/s1ns/ea/dirtybit）、aa64_va_parameters TCR 解析（T0SZ/T1SZ 地址空间大小、TG0/TG1 粒度 4K/16K/64K、HA/HD 硬件访问标志管理、TTBR 选择基于 VA[55]）、get_phys_addr_lpae LPAE 逐级遍历（regime_tcr→regime_ttbr→起始级别计算→描述符读取→Table/Block/Page 分支→AF/DBM 自动管理→权限检查）、描述符格式（AttrIndx/AP/SH/AF/DBM/PXN/UXN + S2AP/MemAttr）、get_S1prot 权限检查（PAN/PAN3/EPAN 用户页特权阻止、WXN 写蕴含不可执行、安全域 SIF/Root/Realm 取指限制）、get_S2prot Stage-2 权限（S2AP[1:0] 读写 + XN 执行控制）、get_phys_addr_twostage 两阶段翻译（S1 VA→IPA → 切换 Stage2 MMU 索引 → S2 IPA→PA → prot 交集 → 属性合并 → 最小页面大小）、S1_ptw_translate（S1 描述符地址的 S2 翻译 + s1ptw 故障标记）、属性合并（无 FWB 取更强约束 + FWB S2 覆盖 S1 + 共享属性取最大值）、SoftMMU TLB（CPUTLBEntry addr_read/write/code+addend + Victim TLB 8 条目 + tlb_set_page_full 填充）、arm_cpu_tlb_fill_align TLB miss 处理（对齐检查→get_phys_addr→成功填充/probe 返回/arm_deliver_fault）、TLBI 指令模拟（VMALLE1/VAE1/ALLE1/IPAS2E1 + ASID/VMID 过滤 + alle1_tlbmask 掩码）、AT 指令（do_ats_write → get_phys_addr_for_at → PAR_EL1 构建成功/失败格式）、TBI 地址标签（aa64_va_parameter_tbi + hflags 缓存）。

**适合读者**：需要理解 ARM64 页表遍历实现、Stage-1/Stage-2 翻译交互、TLB 管理、TLBI/AT 指令模拟或内存属性合并机制的开发者。  
**关键源文件**：`target/arm/ptw.c`（~3950行）、`target/arm/mmuidx.h`（~200行）、`target/arm/internals.h`（~1500行）、`target/arm/helper.c`（~10100行）、`target/arm/tcg/tlb_helper.c`（~380行）、`target/arm/tcg/tlb-insns.c`（~740行）、`target/arm/tcg/cpregs-at.c`（~360行）、`accel/tcg/cputlb.c`（~2600行）

### [39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md](arm64/39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md)
> **20KB · 15 节**

ARM64 EL3/Secure 世界切换全栈分析：SMC 指令翻译（trans_SMC 两步处理：pre_smc 陷阱决策 + EXCP_SMC 异常生成）、pre_smc 决策表（有/无 EL3 × SMD × HCR_TSC × PSCI 三维矩阵、AArch64 SMD 安全态也生效 vs AArch32 仅非安全态、HCR_TSC 路由 EL2 优先于 SMD、NV 嵌套虚拟化陷阱）、arm_cpu_do_interrupt_aarch64 异常进入（VBAR_EL3 基地址 + 向量偏移 +0x400/+0x600 取决于 SCR_RW、SPSR_EL3/ELR_EL3/ESR_EL3 保存、PSTATE.DAIF=1111 全屏蔽 + SP_EL3 + PAN/TCO/SSBS/ALLINT 设置、arm_rebuild_hflags 切换到 E3 MMU 索引）、EL3 系统寄存器（SCR_EL3/TTBR0_EL3/TCR_EL3/ELR_EL3/SPSR_EL3/ESR_EL3/FAR_EL3/VBAR_EL3/CPTR_EL3 全部 opc1=6 PL3_RW）、SCR_EL3 写入回调（valid_mask 从 0x3fff 基础 + 20+ 特性门控扩展、SCR_RW 无 AArch32 时强制 1、NS/NSE 变化触发 12 种 MMU 索引 TLB 刷新）、安全状态判定（arm_security_space 四域：EL3 → Root/Secure、EL3 以下 → SCR_NS/NSE 三分支 Secure/NonSecure/Realm）、hflags 与 TB 隔离（rebuild_hflags_a64 编码 current_el + mmu_idx 确保 EL3 TB 不与 EL1 混用、E3 vs E10_1 翻译体制差异）、ERET 返回（el_from_spsr 提取目标 EL、RME NSE=1+NS=0 非法检查、执行宽度匹配、HCR_TGE EL1 返回禁止、pstate_write 恢复 PSTATE + aarch64_restore_sp + rebuild_hflags、TBI 地址调整、illegal_return 设置 PSTATE.IL）、安全内存访问（MemTxAttrs.secure → ARMASIdx_S/NS → 不同 address_space）、PSCI 旁路（arm_is_psci_call 在陷阱决策前检查、arm_handle_psci_call 实现 CPU_ON/OFF/RESET/VERSION）、EL3 陷阱控制（SMD/TERR/TLOR/API/APK/ATA/HXEN/FGTEN/PIEN 等 SCR_EL3 位控制低 EL 指令陷入）。

**适合读者**：需要理解 ARM64 安全世界切换实现、SMC/ERET 异常路径、EL3 Monitor 执行环境、SCR_EL3 安全配置或 PSCI 固件接口的开发者。  
**关键源文件**：`target/arm/tcg/translate-a64.c`（~3205行）、`target/arm/tcg/op_helper.c`（~1200行）、`target/arm/helper.c`（~10190行）、`target/arm/tcg/helper-a64.c`（~785行）、`target/arm/tcg/hflags.c`（~575行）、`target/arm/tcg/psci.c`（~120行）

### [40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md](arm64/40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md)
> **16KB · 15 节**

ARM64 EL1/EL2 交互全栈分析：HVC 指令翻译（trans_HVC target_el=2/3 分支、pre_hvc 决策：PSCI 旁路 > SCR_EL3.HCE > HCR_EL2.HCD > 安全态检查）、异常进入 EL2（VBAR_EL2 + 向量偏移 +0x400/+0x600、HCR_RW 决定低 EL 宽度、ESR_EL2/ELR_EL2/SPSR_EL2 保存）、HCR_EL2 位域全景（60+ 位：VM/FMO/IMO/AMO/DC/TWI/TWE/TSC/TVM/TRVM/TGE/HCD/RW/E2H/NV/NV1/NV2/FWB 分六大类）、arm_hcr_el2_eff 有效值计算（安全态无 SEL2→返回 0、TGE+E2H 清除虚拟化位、TGE 非 E2H 强制 FMO/IMO/AMO）、VHE 寄存器重定向（28 对 vhe_redir_to_el2/el01：SCTLR/TTBR0/TTBR1/TCR/VBAR/ELR/SPSR/FAR/ESR/MAIR_EL1→EL2、EL12 编码访问原始 EL1）、el_is_in_host（EL0 需 E2H+TGE、EL2 需 E2H）、VHE 对 MMU 索引影响（E20_0/E20_2 vs E10_0/E2、双范围 TTBR）、Stage-2 使能（HCR_VM→两阶段翻译、HCR_DC→默认可缓存、VHE 模式清除 VM/DC）、陷阱访问函数（access_tvm_trvm→CP_ACCESS_TRAP_EL2、影响 SCTLR/TCR/TTBR 等 EL1 寄存器）、细粒度陷阱 FGT（HFGRTR/HFGWTR/HDFGRTR/HDFGWTR/HFGITR_EL2、SCR_FGTEN 门控、HFGITR.ERET 陷阱位→TRAP_ERET hflag）、WFE/WFI 陷阱链（SCTLR.nTWE/nTWI→EL1 > HCR_TWE/TWI→EL2 > SCR_TWE/TWI→EL3 三级优先级）、ERET 从 EL2（HCR_TGE 禁止返回 EL1、hflags 从 E2/E20_2 切换到 E10_1）、EL2 hflags（E2H/UNPRIV/NV/NV1/NV2/NV2_MEM_BE/TRAP_ERET/FGT_ACTIVE 标志差异表）、嵌套虚拟化 NV/NV2（NV EL1 访问 EL2 寄存器陷入、NV2 通过 VNCR_EL2 内存后备避免 VM Exit、nv2_redirect_offset 固定偏移、ERET/SMC/AT/TLBI 陷阱路由）。

**适合读者**：需要理解 ARM64 Hypervisor 陷阱机制、VHE 寄存器重定向、HCR_EL2 位域效果、Stage-2 控制或嵌套虚拟化 NV/NV2 实现的开发者。  
**关键源文件**：`target/arm/tcg/translate-a64.c`（~3205行）、`target/arm/tcg/op_helper.c`（~1200行）、`target/arm/helper.c`（~10190行）、`target/arm/tcg/hflags.c`（~575行）、`target/arm/cpu.h`（~1755行）

### [41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md](arm64/41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md)
> **21KB · 15 节**

ARM64 EL 切换时 TCG 翻译器完整行为变化分析：CPUARMTBFlags 96 位布局（flags 32 位 + flags2 64 位）、TBFLAG_ANY 共享 14 位（AARCH64_STATE/SS_ACTIVE/PSTATE__SS/BE_DATA/MMUIDX/FPEXC_EL/ALIGN_MEM/PSTATE__IL/FGT_ACTIVE/FGT_SVC）、TBFLAG_A64 专用 45 位（TBII/SVEEXC_EL/VL/PAUTH_ACTIVE/BT/BTYPE/TBID/UNPRIV/ATA/TCMA/MTE_ACTIVE/SMEEXC_EL/PSTATE_SM/ZA/SVL/TRAP_ERET/NAA/NV/NV1/NV2/E2H/AH/NEP/GCS_EN/GCSSTR_EL）、rebuild_hflags_a64 十大构建阶段（TBI→E2H→SVE/SME→SCTLR 对齐/端序/PAuth/BTI/NAA→UNPRIV→PSTATE_IL/FGT/TRAP_ERET→NV/NV1/NV2→MTE→GCS→FPCR）、rebuild_hflags_internal 模式分发（a64/a32/m32）、arm_rebuild_hflags 缓存写入、HELPER(rebuild_hflags_a64) EL 切换调用版本、arm_get_tb_cpu_state TB 键生成（PC+flags+flags2 三元组、BTYPE/PSTATE__SS 动态填充）、异常入口完整流程（save_sp→save ELR→save SPSR→PSTATE_DAIF 全置位→PAN/TCO/SSBS/ALLINT 设置→pstate_write→restore_sp→arm_rebuild_hflags→设置 VBAR+向量偏移 PC）、ERET 恢复流程（save_sp→pstate_write(spsr)→清除 SS→restore_sp→helper_rebuild_hflags_a64→TBI 处理→设置 PC）、TB 链断裂机制（CPU_INTERRUPT_EXITTB/DISAS_EXIT/PC 变化/hflags 变化四重保障）、gen_goto_tb 链接决策（use_goto_tb 同页链接 vs lookup_and_goto_ptr）、寄存器组切换（sp_el[4]/elr_el[4]/banked_spsr[8]、aarch64_save_sp/restore_sp 按 PSTATE.SP 选择 SP_ELn vs SP_EL0）、DisasContext 初始化（MMUIDX→current_el 间接推导、80+ 行 hflag 提取）、SCTLR 影响表（A→ALIGN_MEM/EE→BE_DATA/EnIA→PAUTH_ACTIVE/BT0→BT/nAA→NAA）、HCR_EL2 影响表（E2H→VHE/TGE→UNPRIV/NV→TRAP_ERET/NV2→NV2_MEM_BE/FGT→FGT_SVC）、EL 相关指令行为差异（HVC/SMC/ERET 最低 EL 检查、系统寄存器 EL 路由/VHE 重定向）、单步调试跨 EL 处理（SS_ACTIVE/PSTATE_SS 状态机、SPSR 保存/恢复 SS 位）、EL0↔EL1 hflags 差异对比表。

**适合读者**：需要理解 ARM64 EL 切换如何影响 TCG 翻译缓存、hflags 构建细节、TB 链断裂原理或寄存器组切换实现的开发者。  
**关键源文件**：`target/arm/tcg/hflags.c`（~700行）、`target/arm/cpu.h`（~2550行）、`target/arm/tcg/translate-a64.c`（~10750行）、`target/arm/helper.c`（~10190行）、`target/arm/tcg/helper-a64.c`（~750行）、`target/arm/internals.h`（~1500行）

### [42-ARM64-TCG前端后端代码生成深度分析-IR中间表示-翻译循环-优化Pass-寄存器分配与AArch64代码发射.md](arm64/42-ARM64-TCG前端后端代码生成深度分析-IR中间表示-翻译循环-优化Pass-寄存器分配与AArch64代码发射.md)
> **28KB · 17 节**

ARM64 TCG 前端/后端完整代码生成流水线分析：TCG IR 系统（TCGTemp 五种生命周期 EBB/TB/GLOBAL/FIXED/CONST、TCGOp 链表结构、TCGContext 全局上下文 350+ 行字段）、TCGOpcode 操作码全景（tcg-opc.h 180+ 标量/向量指令：控制流/算术/逻辑/移位/位操作/比较/加载存储/进位链/类型转换/TB 控制/guest 内存/向量运算）、translator_loop 通用翻译循环（init_disas_context→tb_start→insn_start→translate_insn→tb_stop 五回调、DISAS_NEXT/TOO_MANY/NORETURN/EXIT 终止条件、tcg_op_buf_full/max_insns 限制）、ARM64 TranslatorOps（aarch64_translator_ops 回调表、aarch64_tr_translate_insn 单步/对齐/解码流程、aarch64_tr_tb_stop DISAS 分支处理：goto_tb/exit_tb/lookup_and_goto_ptr/WFI）、decodetree 解码（a64.decode→trans_* 函数→tcg_gen_* IR 生成）、tcg_gen_* 前端 API（tcg_emit_op 追加 ops 链表、mov/add/sub/ld/st/brcond/goto_tb/exit_tb/lookup_and_goto_ptr）、tb_gen_code 入口（translate-all.c:261-420 物理地址转换→TB 分配→setjmp_gen_code→溢出重试三种错误码 -1/-2/-3）、tcg_optimize 优化 Pass（OptContext/TempOptInfo 拷贝链+z/o/s_mask 追踪、copy_propagate→fold_*→常量折叠/条件折叠/死代码消除/内存拷贝追踪 IntervalTree）、liveness_pass_1 活性分析（反向遍历 ops、la_bb_end 块末状态、DEAD_ARG/SYNC_ARG 位图、纯函数无用输出删除）、寄存器分配器（AArch64 分配顺序 X20-X28 callee-saved 优先→X8-X15→X0-X7、tcg_reg_alloc 单/双寄存器分配、temp_sync 溢出到栈帧、tcg_reg_alloc_mov/call 特殊路径）、AArch64 后端代码发射（tcg_out_mov/movi/ld/st/call 指令编码、tcg_out_insn 宏编码层）、guest 内存访问翻译（prepare_host_addr TLB 内联查找：LDP mask+table→AND 索引→ADD 地址→LDR 比较值+addend→AND+CMP 页匹配→B.NE 慢路径、tcg_out_qemu_ld_direct LDRB/LDRH/LDRW/LDR 选择、慢路径 helper_ld*_mmu 调用）、TB 链接与运行时补丁（tcg_out_goto_tb B+BR 双指令、tb_target_set_jmp_target ±128MB 直接 B vs LDR+BR 间接、qatomic_set+flush_idcache 原子补丁）、Prologue/Epilogue（tcg_target_qemu_prologue STP FP/LR→保存 X19-X28→SUB SP→MOV AREG0→BR TB→tb_ret_addr→恢复→RET）、代码缓冲区管理（tcg_init→tcg_context_init+tcg_region_init、tcg_tb_alloc highwater 检查、tb_flush 全量刷新）。

**适合读者**：需要理解 QEMU TCG 翻译流水线全貌、IR 中间表示设计、优化 Pass 实现、寄存器分配策略或 AArch64 后端代码生成细节的开发者。  
**关键源文件**：`include/tcg/tcg.h`（~440行）、`include/tcg/tcg-opc.h`（~185行）、`accel/tcg/translator.c`（~250行）、`accel/tcg/translate-all.c`（~640行）、`tcg/tcg.c`（~6000行）、`tcg/optimize.c`（~3200行）、`tcg/aarch64/tcg-target.c.inc`（~3540行）、`target/arm/tcg/translate-a64.c`（~10960行）

### [45-ARM64-TCG内存模型与原子操作深度分析-屏障语义-MemOp标志-Exclusive-LSE原子与后端发射.md](arm64/45-ARM64-TCG内存模型与原子操作深度分析-屏障语义-MemOp标志-Exclusive-LSE原子与后端发射.md)
> **19KB · 17 节**

ARM64 TCG 内存模型完整子系统分析：TCGBar 屏障类型系统（TCG_MO_LD_LD/ST_LD/LD_ST/ST_ST/ALL 五种排序+TCG_BAR_LDAQ/STRL/SC 三种种类、组合含义映射 ARM DMB 域）、tcg_gen_mb 屏障 IR 生成（user-only CF_PARALLEL 门控 vs softmmu 始终生成因 IO 线程并行）、MemOp 内存操作标志完整定义（大小 MO_8-MO_1024、符号 MO_SIGN、端序 MO_BSWAP/LE/BE、对齐 MO_UNALN-MO_ALIGN+MO_ALIGN_TLB_ONLY、原子性 MO_ATOM_IFALIGN/IFALIGN_PAIR/WITHIN16/WITHIN16_PAIR/SUBALIGN/NONE 六级）、ARM64 DMB/DSB/ISB 翻译（DMB/DSB Reads→LD_LD|LD_ST Writes→ST_ST All→ALL 三种映射+DSB 等效处理、ISB→gen_goto_tb TB 断裂无屏障 IR）、Load-Acquire/Store-Release（STLR→mb(STRL)+store 前屏障、LDAR→load+mb(LDAQ) 后屏障）、Exclusive Monitor 三寄存器状态（exclusive_addr/val/high cpu.h:704-713、arm_clear_exclusive 置-1、CLREX 指令翻译）、gen_load_exclusive 独占加载（单/配对/128 位三种模式、MTE 标签检查、记录 exclusive_addr/val/high）、gen_store_exclusive 独占存储（地址比较+atomic_cmpxchg 值比较+成功/失败分支+清除监控、CAS 实现而非真 exclusive monitor）、LDXR/STXR/LDXP/STXP 翻译（lasr 标志控制 Acquire/Release 附加屏障）、LSE 原子指令翻译（do_atomic_ld 通用框架+LDADD/LDCLR/LDEOR/LDSET/SWP 到 tcg_gen_atomic_fetch_* 映射+LDSMAX/LDSMIN/LDUMAX/LDUMIN 有符号无符号极值）、CAS/CASP 比较交换（64 位 cmpxchg_i64+128 位 cmpxchg_i128）、128 位原子（FEAT_LSE128 LDCLRP/LDSETP/SWPP→i128 原语）、TCG 原子 helper 生成（atomic_template.h 宏模板生成所有大小×端序 helper、atomic_mmu_lookup TLB 交互获取 host 地址、GCC __atomic_* 原生原子操作）、AArch64 后端屏障发射（tcg_out_mb 静态数组映射 TCG_MO→DMB ISH/ISHLD/ISHST、所有屏障使用 ISH 内部共享域）、CF_PARALLEL 与 MTTCG 内存序（MTTCG 全屏障+原子 helper vs RR 非原子优化+softmmu 仍保留屏障、tcg_canonicalize_memop 原子性降级）。

**适合读者**：需要理解 QEMU TCG 内存排序模型、ARM 屏障/Acquire-Release 语义翻译、Exclusive Monitor 实现、LSE 原子指令映射或原子 helper 生成机制的开发者。
**关键源文件**：`include/tcg/tcg-mo.h`（~46行）、`include/exec/memop.h`（~130行）、`tcg/tcg-op.c`（tcg_gen_mb ~18行）、`tcg/tcg-op-ldst.c`（原子操作 ~440行）、`target/arm/tcg/translate-a64.c`（屏障+exclusive+LSE ~560行）、`target/arm/cpu.h`（exclusive 状态 ~10行）、`tcg/aarch64/tcg-target.c.inc`（tcg_out_mb ~11行）、`accel/tcg/atomic_template.h`（~312行）

### [44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md](arm64/44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md)
> **25KB · 17 节**

ARM64 TCG 执行引擎完整运行时分析：cpu_exec 主入口（sigsetjmp 异常恢复、cpu_handle_halt 检测、cpu_exec_enter/exit 钩子、SyncClocks 时钟同步初始化）、cpu_exec_loop 双层循环（外层 cpu_handle_exception 异常处理循环、内层 cpu_handle_interrupt 中断检查+TB 执行循环、get_tb_cpu_state 获取翻译键 pc/flags/cflags）、TB 查找两级缓存（tb_lookup 第一级 per-CPU jmp_cache O(1) 直接映射 PC 哈希+flags+cflags 比较、第二级 tb_htable_lookup 全局 QHT 物理地址哈希查找+qht_lookup_custom、未命中→tb_gen_code JIT 编译+回填 jmp_cache）、TB 执行 cpu_tb_exec（tcg_qemu_tb_exec 进入翻译代码、返回值 ret&~TB_EXIT_MASK=last_tb/ret&TB_EXIT_MASK=退出码、TB_EXIT_IDX0/IDX1 正常链跳转/TB_EXIT_REQUESTED 被中断、未执行 TB 恢复 guest PC synchronize_from_tb、单步调试 EXCP_DEBUG）、cpu_loop_exec_tb 退出后处理（icount 到期补充 icount_decr.u16.low+icount_extra、精确大小 TB 生成 cflags_next_tb）、TB 链接 tb_add_jump（qatomic_cmpxchg 原子占用 jmp_dest 跳转槽、tb_set_jmp_target 补丁本机跳转、jmp_list_head/next 反向链表用于失效撤销）、中断处理 cpu_handle_interrupt 优先级链（CF_NOIRQ 跳过→icount_decr.u16.high 清零→CPU_INTERRUPT_DEBUG→HALT→RESET→cpu_exec_interrupt 目标特定 IRQ/FIQ→EXITTB 断链→exit_request/icount 到期→EXCP_INTERRUPT）、异常处理 cpu_handle_exception（exception_index<0 无异常、≥EXCP_INTERRUPT 异步退出返回调用者、<EXCP_INTERRUPT 同步异常→do_interrupt 回调→继续执行）、EXCP_* 退出码全景（INTERRUPT/HLT/DEBUG/HALTED/YIELD/ATOMIC 六种异步码）、CF_* 编译标志（CF_COUNT_MASK/NO_GOTO_TB/NO_GOTO_PTR/SINGLE_STEP/MEMI_ONLY/USE_ICOUNT/INVALID/PARALLEL/NOIRQ/PCREL/BP_PAGE/CLUSTER_MASK 十二种标志）、MTTCG 多线程模型（mttcg_cpu_thread_fn 每 vCPU 独立 OS 线程、tcg_cpu_exec→cpu_exec、EXCP_ATOMIC→cpu_exec_step_atomic exclusive 单步、线程创建 CF_PARALLEL 标志）、RR 轮转单线程模型（rr_cpu_thread_fn 共享线程"ALL CPUs/TCG"、rr_kick_vcpu_timer/rr_kick_next_cpu 定时器时间片切换 TCG_KICK_PERIOD、BQL 锁定 bql_lock/unlock、icount_prepare_for_run/process_data 预算分配、CPU_NEXT 轮转、rr_start_vcpu_thread 复用线程）、exclusive 执行区域（start_exclusive 停止所有 CPU pending_cpus 计数+qemu_cpu_kick、end_exclusive 广播恢复、cpu_exec_start/cpu_exec_end 执行屏障 running 标志+has_waiter 配合、可重入 exclusive_context_count）、icount 指令计数（icount_decr.u16.low 每指令递减+u16.high 强制退出、icount_extra 溢出预算、icount_budget 本轮总预算、CF_USE_ICOUNT 影响翻译插入递减检查、align_clocks guest 太快则 nanosleep 节流）、cpu_exec_step_atomic（sigsetjmp+start_exclusive→CF_NO_GOTO_TB|~CF_PARALLEL|1 单条指令编译执行→end_exclusive 恢复）。

**适合读者**：需要理解 QEMU TCG 运行时执行引擎全貌、TB 查找/链接机制、中断异常处理流程、MTTCG/RR 线程模型或 icount 指令计数实现的开发者。
**关键源文件**：`accel/tcg/cpu-exec.c`（~1070行）、`accel/tcg/tcg-accel-ops-mttcg.c`（~137行）、`accel/tcg/tcg-accel-ops-rr.c`（~348行）、`cpu-common.c`（exclusive 部分 ~140行）、`include/exec/cpu-common.h`（EXCP_* 定义）、`include/exec/translation-block.h`（TB 结构 + CF_* 标志）

### [43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md](arm64/43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md)
> **23KB · 20 节**

ARM64 TCG 内存访问子系统完整分析：softmmu TLB 五层数据结构体系（CPUTLBEntry 快速路径 32 字节条目 addr_read/write/code+addend、CPUTLBDescFast 16 字节 mask+table 对齐加载、CPUTLBEntryFull 完整条目 xlat_offset/section/phys_addr/attrs/prot/slow_flags/ARM-extra 缓存属性、CPUTLBDesc 每 MMU 模式描述符含大页追踪/动态调整窗口/8 条目 victim cache、CPUTLB 顶层 22 个 MMU 模式分区）、TLB 索引计算与 addend 技巧（(addr>>PAGE_BITS)&mask 索引、host=guest+addend 一次加法直达 host 地址）、TLB 标志位双层系统（快速路径 TLB_INVALID_MASK/TLB_NOTDIRTY/TLB_FORCE_SLOW 三位 addr_idx 低位、慢路径 TLB_BSWAP/TLB_WATCHPOINT/TLB_CHECK_ALIGNED/TLB_DISCARD_WRITE/TLB_MMIO 五位 slow_flags）、ARMMMUIdx 22 种 MMU 索引（E10_0/E10_1/E20_0/E20_2/E2/E3 六组常规+GCS/PAN 变体、Stage2/Stage2_S 二阶段、Phys_S/NS/Root/Realm 四物理空间）、快速路径 TLB 内联查找（AArch64 后端 prepare_host_addr LDP+AND+ADD+LDR+CMP+B.NE ~6 条指令、probe_access_internal 检查与分发）、慢路径填充（arm_cpu_tlb_fill_align 对齐检查→get_phys_addr PTW→tlb_set_page_full 安装、victim cache 8 条目全关联查找+swap 策略）、ARM 页表遍历（get_phys_addr 入口→get_phys_addr_gpc GPC 检查→get_phys_addr_nogpc 分派 disabled/PMSA/VMSA-short/LPAE/twostage、get_phys_addr_lpae LPAE 四级遍历 L0-L3 描述符解析/AF 检查/DBM 处理/权限检查）、两阶段翻译 S1→S2（VA→IPA→PA 两次遍历、权限 AND 合并、combine_cacheattrs 缓存属性组合 FWB vs 传统取弱）、tlb_set_page_full TLB 安装（section 映射/标志计算/addend 计算/条目填充/victim 驱逐）、MMIO 分发（io_prepare→memory_region_dispatch_read/write→mr->ops 设备回调）、TLBI 指令（AArch32 TLBIALL/TLBIMVA/TLBIASID、AArch64 VMALLE1/ALLE1-3/VAE1-2/IPAS2E1/Range-TLBI、IS 变体跨 CPU 广播）、AT 指令（do_ats_write→get_phys_addr→PAR_EL1、ats_write64 S1E1R/W/S1E0R/W/S12E1R/W）、原子操作 atomic_mmu_lookup（TLB 查找+对齐检查+host 地址返回→host 原生原子指令）、脏页追踪 notdirty_write（TLB_NOTDIRTY 首次写通知迁移/VGA 子系统→清除标志后续直走快速路径）、TLB 动态调整（256-4096 条目范围、窗口统计自适应大小）、arm_deliver_fault 异常生成（target_el 选择、GPC/Stage2 路由、syndrome 构建 insn_abort/data_abort、raise_exception）。

**适合读者**：需要理解 QEMU softmmu 内存访问全貌、TLB 数据结构设计、快慢路径实现、ARM PTW 细节或 TLBI/AT 指令实现的开发者。
**关键源文件**：`include/exec/tlb-common.h`（~56行）、`include/hw/core/cpu.h`（TLB 部分 ~130行）、`include/exec/tlb-flags.h`（~86行）、`accel/tcg/cputlb.c`（~2600行）、`target/arm/ptw.c`（~4100行）、`target/arm/tcg/tlb_helper.c`（~380行）、`target/arm/tcg/tlb-insns.c`（~900行）、`target/arm/tcg/cpregs-at.c`（~420行）

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

### [08-ARM-Generic-Timer深度分析-计数器-7类定时器-EL访问控制-VHE重定向与KVM集成.md](device-model/08-ARM-Generic-Timer深度分析-计数器-7类定时器-EL访问控制-VHE重定向与KVM集成.md)
> **26KB · 17 节**

ARM Generic Timer 完整实现分析：ARMGenericTimer 数据结构（cval/ctl 双字段）、7种定时器变体枚举（PHYS/VIRT/HYP/SEC/HYPVIRT/S_EL2_PHYS/S_EL2_VIRT 与 NUM_GTIMERS=7）、物理/虚拟计数器实现（QEMU_CLOCK_VIRTUAL/gt_cntfrq_period_ns 纳秒→tick 转换）、CNTPCT/CNTVCT 计数器偏移（CNTVOFF_EL2 虚拟偏移、CNTPOFF_EL2 FEAT_ECV 物理偏移、直接/间接访问偏移差异）、定时器频率（1GHz 默认/62.5MHz 兼容）、CTL/CVAL/TVAL 寄存器操作（ENABLE/IMASK/ISTATUS 位域、TVAL 动态计算非独立存储）、VHE 定时器重定向（gt_phys_redir_timeridx→GTIMER_HYP、gt_virt_redir_timeridx→GTIMER_HYPVIRT、EL02 编码绕过重定向）、四级 EL 访问控制（CNTKCTL_EL1 EL0 门控、CNTHCTL_EL2 EL1 陷阱、SCR_EL3.ST 安全定时器门控、gt_sel2timer_access NV 陷阱）、定时器重算核心（gt_recalc_timer ISTATUS 计算+QEMUTimer 调度、gt_update_irq IRQ 输出+RME CNTVMASK/CNTPMASK 覆盖）、7个回调函数、QEMUTimer 基础设施、CPU 定时器创建（7×timer_new）、GPIO 输出注册、virt 机器 GIC PPI 接线（7×ARCH_TIMER_*_IRQ→PPI 3-14）、DTB /timer 节点生成、迁移 vmstate（PHYS/VIRT QEMUTimer + cpreg 自动迁移）、KVM 定时器同步（kvm_vtime/KVM_REG_ARM_TIMER_CNT pre_save/post_load）、WFxT 超时定时器。

**适合读者**：需要理解 ARM 架构定时器在 QEMU 中完整实现、EL 访问控制规则、VHE 重定向机制或调试定时器中断的开发者。  
**关键源文件**：`target/arm/helper.c`（~1100行定时器代码）、`target/arm/cpu.c`（定时器创建）、`target/arm/gtimer.h`（枚举）、`hw/arm/virt.c`（GIC 接线+DTB）、`include/hw/arm/bsa.h`（IRQ 常量）

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

### [07-Linux-user用户模式翻译深度分析.md](accel/07-Linux-user用户模式翻译深度分析.md)
> **23KB · 26 节**

Linux-user 用户模式翻译完整解析：main() 初始化流程（QOM/TCG/CPU 创建/ELF 加载/信号初始化/cpu_loop 入口），ELF 二进制加载（load_elf_image/PT_INTERP 动态链接器/create_elf_tables auxv 构造/setup_arg_pages 栈分配），Guest 地址空间模型（guest_base 偏移/reserved_va/g2h/h2g 转换/无 TLB 直接映射），AArch64 CPU 循环（cpu_exec → trapnr switch → EXCP_SWI/DATA_ABORT/UDEF/ATOMIC），系统调用翻译（do_syscall ~380 个 case/AArch64 ABI X8+X0-X5/特殊返回值 ERESTARTSYS/ESIGRETURN），内存管理（target_mmap/munmap/mprotect → host mmap + page_set_flags），页面保护管理（PageFlagsNode 区间树/page_get/set_flags/page_unprotect），信号处理架构（signal_init/host_signal_handler → queue_signal → process_pending_signals），AArch64 信号帧构造（target_rt_sigframe/FPSIMD+SVE+MTE 上下文保存），线程与 Clone（do_fork/CLONE_VM→pthread_create/TLS→TPIDR_EL0），用户模式 TCG 差异（无 softmmu TLB/guest_base ADD 直接访问/1行 vs 8行开销），Thunk 层（stat/sockaddr/ioctl 结构体转换），文件系统仿真（/proc/self/maps 等），strace 模式，GDB 调试，ARM64 特性（SVE/SME/MTE/PAC/BTI 用户模式支持），系统模式 vs 用户模式全面对比表。

**适合读者**：需要理解 QEMU 用户模式翻译原理、调试交叉编译程序、扩展系统调用支持的开发者。
**关键源文件**：`linux-user/main.c`（~1040行）、`linux-user/syscall.c`（~14500行）、`linux-user/signal.c`（~1400行）、`linux-user/mmap.c`（~1250行）、`linux-user/elfload.c`（~2000行）、`linux-user/aarch64/cpu_loop.c`（~230行）、`accel/tcg/user-exec.c`（~750行）

---

## arm/ ARM 架构通用

### [08-ARM64-EL状态管理与指令执行深度分析.md](arm/08-ARM64-EL状态管理与指令执行深度分析.md)
> **32KB · 26 节**

ARM64 Exception Level (EL0-EL3) 完整状态管理分析：PSTATE 位布局与分散存储架构、arm_current_el() EL 提取、异常进入流程（arm_cpu_do_interrupt → arm_cpu_do_interrupt_aarch64）、SPSR/ELR/ESR/FAR 状态保存、异常向量表偏移计算（来源×类型 4×4 矩阵）、PSTATE DAIF 掩码设置、ERET 异常返回（el_from_spsr → SPSR 目标 EL 提取 → AArch64/AArch32 返回路径）、SMC 路由决策（SCR_EL3.SMD/HCR_EL2.TSC/PSCI 条件表）、HVC 路由、安全状态管理（ARMSecuritySpace Secure/NonSecure/Root/Realm）、SCR_EL3/HCR_EL2 关键控制位、ARMMMUIdx EL→MMU 索引映射、系统寄存器 PL 访问控制（cp_access_ok 位编码）、特权指令行为差异（WFI/WFE/Cache/AT/TLBI 各 EL 权限）、TB flags MMUIDX 间接编码 EL、arm_rebuild_hflags EL 变化后重建、DisasContext EL 传播、PAN/UAO/SP 选择/SCTLR 差异、SVE/SME EL 变化影响、arm_el_is_aa64 执行状态级联控制（SCR.RW→HCR.RW）、EL 切换完整时序图、各 EL 指令执行差异对比表。

**适合读者**：需要理解 ARM64 异常级别切换机制、安全状态管理、指令权限控制的开发者。
**关键源文件**：`target/arm/helper.c`（~10200行）、`target/arm/internals.h`（~1600行）、`target/arm/cpu.h`（~3500行）、`target/arm/tcg/helper-a64.c`（~800行）、`target/arm/tcg/op_helper.c`（~1200行）、`target/arm/tcg/translate-a64.c`（~11000行）、`target/arm/tcg/hflags.c`（~650行）、`target/arm/mmuidx.h`（~200行）

### [09-AArch32异常处理与模式切换深度分析.md](arm/09-AArch32异常处理与模式切换深度分析.md)
> **34KB · 30 节**

AArch32 异常处理完整分析：CPSR 位布局（M/T/F/I/A/E/GE/IT/NZCV）与分散存储架构（uncached_cpsr + 缓存标志字段）、cpsr_read()/cpsr_write() 重组与拆分机制、9 种处理器模式（USR/FIQ/IRQ/SVC/MON/ABT/HYP/UND/SYS）与 EL 映射、banked 寄存器架构（banked_r13/r14/spsr[8] + usr_regs/fiq_regs[5]）、bank_number()/r14_bank_number() 索引映射（HYP LR 特殊处理）、switch_mode() 模式切换（FIQ R8-R12 交换）、arm_cpu_do_interrupt_aarch32() 异常分发（UDEF/SWI/BKPT/Abort/IRQ/FIQ/VIRQ/VFIQ/VSERR/SMC/MON_TRAP 完整分发表）、向量偏移与返回偏移、向量基地址选择（MVBAR/高向量/VBAR）、take_aarch32_exception() 核心流程（SPSR 保存/IT 清除/模式位设置/大端/DAIF/PAN/Thumb 状态/LR 保存）、HYP 异常进入（0x14 统一入口/SCR 控制屏蔽/HVBAR）、SVC 完整路径（trans_SVC → DISAS_SWI → syndrome）、Abort 路径（DFSR/IFSR/DFAR/IFAR vs ESR/FAR 对比）、IRQ/FIQ 路由（SCR.IRQ/FIQ → Monitor）、SMC/HVC 路由、异常返回机制（MOVS PC,LR / RFE / LDM{^}）、cpsr_write_eret() 实现、Thumb/IT 块处理、AArch32↔AArch64 寄存器同步（aarch64_sync_32_to_64/64_to_32 完整映射表）、SPSR AArch64 bank 索引、syndrome 编码、AArch32 vs AArch64 异常处理对比表、完整异常进入/返回时序图、各模式寄存器可见性总表。

**适合读者**：需要理解 AArch32 异常模式切换、banked 寄存器机制、CPSR 管理、AArch32/AArch64 互操作的开发者。
**关键源文件**：`target/arm/helper.c`（~10200行）、`target/arm/cpu.h`（~3500行）、`target/arm/internals.h`（~1600行）、`target/arm/tcg/translate.c`（~7000行）、`target/arm/tcg/op_helper.c`（~1200行）、`target/arm/syndrome.h`（~300行）

### [10-CP15系统寄存器与MMU页表管理深度分析.md](arm/10-CP15系统寄存器与MMU页表管理深度分析.md)
> **27KB · 31 节**

ARM CP15 协处理器寄存器框架与 MMU 页表管理完整分析：ARMCPRegInfo 寄存器描述结构（name/cp/crn/crm/opc/access/fieldoffset/readfn/writefn 全字段）、CPAccessResult 访问权限模型（OK/TRAP_EL1-3/UNDEFINED）、cp_access_ok() 位编码（EL×读写=8 位位图）、寄存器注册系统（register_cp_regs_for_features 条件注册/哈希表存储）、MRC/MCR 解码路径（do_coproc_insn/gen_helper_access_check_cp_reg）、运行时访问检查（accessfn/FGT/HSTR）、env->cp15 存储结构（union EL 索引+S/NS 别名）、A32_BANKED 宏、SCTLR 位定义（M/A/C/I/WXN/EE/TE/AFE/TRE 40+位）、sctlr_write() 回调（值比较优化+TLB 刷新）、TTBR0/TTBR1（bank_fieldoffsets/ASID 刷新）、TCR/TTBCR（T0SZ/T1SZ/TG0/TG1/IPS）、MAIR（AttrIndx→8 位属性映射）、翻译禁用判断（SCTLR.M/HCR.TGE/HCR.DC）、get_phys_addr_nogpc() 总调度（物理直通/两阶段/PMSA/VMSA 分发）、ARMv5/v6 短描述符遍历（L1/L2/Domain/DACR）、TTBR0/1 选择（TTBCR.N 分界）、LPAE/AArch64 长描述符遍历（4KB/16KB/64KB 粒度/Block/Table/Page）、Stage 2 遍历（VTTBR/VTCR/IPA→PA）、get_S1prot() 权限检查（AP/XN/PXN/WXN/PAN/PAN3）、TLBI 实现（TLBIALL/VAE/ASIDE/IPAS2E1）、翻译 Regime 概念（regime_sctlr/tcr/ttbr）、HCR_EL2 对 MMU 影响（VM/DC/TGE）、QEMU TLB 缓存（tlb_set_page_full）、关键寄存器注册表、完整页表遍历流程图、AArch32 vs AArch64 MMU 对比表。

**适合读者**：需要理解 ARM 系统寄存器框架、MMU 页表遍历实现、地址翻译机制的开发者。
**关键源文件**：`target/arm/cpregs.h`（~1250行）、`target/arm/helper.c`（~10200行）、`target/arm/ptw.c`（~4000行）、`target/arm/cpu.h`（~3500行）、`target/arm/tcg/translate.c`（~7000行）、`target/arm/tcg/tlb-insns.c`（~900行）

### [11-GICv3中断控制器深度分析.md](arm/11-GICv3中断控制器深度分析.md)
> **39KB · 41 节**

ARM GICv3 通用中断控制器完整实现分析：GICv3 设备模型与 QOM 类型注册（TYPE_ARM_GICV3/属性定义）、GICv3State 全局状态（Distributor 位图/IROUTER 路由/优先级数组）、GICv3CPUState 每 CPU 状态（ICC 寄存器/ICH 虚拟化控制/HPPI 缓存）、PendingIrq 结构、中断类型与编号空间（SGI 0-15/PPI 16-31/SPI 32-1019/LPI 8192+）、Distributor 寄存器实现（GICD_CTLR/ISENABLER/ISPENDR/IPRIORITYR/IROUTER/ICFGR）、NSACR 安全控制与位图管理、亲和路由（GICD_IROUTER→CPU 缓存）、Redistributor 实现（GICR_WAKER/SGI/PPI 管理）、LPI 支持（PROPBASER/PENDBASER/内存表扫描）、CPU Interface 系统寄存器（ICC_PMR/IAR/EOIR/HPPIR/BPR/CTLR/IGRPEN/SGI1R/DIR）、icv_access() 虚拟化重定向（HCR.IMO/FMO 检查）、优先级模型（8 位优先级/Binary Point/Group Priority/Subpriority）、irqbetter() 优先级比较、icc_hppi_can_preempt() 抢占判断（PMR/NMI/APR）、icc_highest_active_prio() 运行优先级计算、gicv3_cpuif_update() 物理信号（G0→FIQ/G1→IRQ 映射）、完整中断注入路径（设备→GIC→CPU 7 步流程）、gicv3_set_irq() SPI/PPI 分发、gicv3_update() 增量+全量重计算优化、arm_cpu_set_irq() CPU 端接收（irq_line_state/VIRQ 合并）、arm_cpu_has_work() 工作检查、arm_cpu_exec_interrupt() 调度优先级（NMI>FIQ>IRQ>VIRQ>VFIQ>VSERR）、arm_excp_unmasked() 屏蔽检查（PSTATE.I/F/ALLINT/HCR 路由）、异常向量计算（VBAR_ELn+EL 偏移+类型偏移）、ICC_IAR1 确认（icc_hppir1_value/icc_activate_irq）、ICC_EOIR1 结束（icc_drop_prio/icc_deactivate_irq）、Split EOI（EOImode/ICC_DIR）、虚拟中断 ICH 接口（List Registers/VMCR/维护中断）、gicv3_cpuif_virt_irq_fiq_update()、SGI 跨 CPU 中断生成、GIC KVM 支持（kvm_gicd/gicr/gicc_access/kvm_arm_gicv3_realize）、Timer→GIC 连接示例、端到端中断流程图、GICv2 vs GICv3 对比表。

**适合读者**：需要理解 ARM GICv3 架构、QEMU GIC 设备模型、中断注入完整路径的开发者。
**关键源文件**：`hw/intc/arm_gicv3.c`（~480行）、`hw/intc/arm_gicv3_cpuif.c`（~2500行）、`hw/intc/arm_gicv3_dist.c`（~760行）、`hw/intc/arm_gicv3_redist.c`（~900行）、`hw/intc/arm_gicv3_kvm.c`（~950行）、`include/hw/intc/arm_gicv3_common.h`（~300行）、`target/arm/cpu-irq.c`（~280行）

### [12-ARM-Cache管理与维护操作深度分析.md](arm/12-ARM-Cache管理与维护操作深度分析.md)
> **24KB · 30 节**

ARM Cache 管理在 QEMU 中的完整实现分析：QEMU 缓存模型概述（不模拟真实缓存/DC 操作全 NOP/DC ZVA 唯一例外）、Cache ID 寄存器体系（CCSIDR/CLIDR/CTR/DCZID）、make_ccsidr() 构造函数（Legacy/CCIDX 32/64 位格式）、ccsidr_read()（CSSELR 索引+Secure/NS bank）、CSSELR 缓存级别选择（bank_fieldoffsets）、CLIDR 缓存级别标识（每 3 位类型编码）、CTR_EL0（DminLine/IminLine/CWG/ERG/DIC/IDC）、DCZID_EL0 与 DC ZVA 块大小（get_dczid_bs）、ARMCPU 缓存参数（ccsidr[16]/ctr/dcz_blocksize）、CPU 模型缓存配置（A57/A72/A53/Neoverse）、arm_cpu_realizefn 缓存初始化（块大小验证）、AArch64 缓存操作注册（v8_cp_reginfo 数组/ARM_CP_NOP）、IC 操作（IALLUIS/IALLU/IVAU）、DC 操作（IVAC/ISW/CVAC/CSW/CVAU/CIVAC/CISW 全 NOP）、DC ZVA 实现（ARM_CP_DC_ZVA 特殊类型/translate 层生成 helper 调用）、HELPER(dc_zva) 详细分析（blocklen 计算/tlb_vaddr_to_host 快速路径/probe_write 慢速路径/I/O 逐字节写零）、PoC/PoU 访问控制（SCTLR.UCI/HCR.TPCP/TPU/TSW/TICAB/TOCU 陷阱）、自修改代码与 IC IVAU（JIT 双映射/W^X）、ic_ivau_write() TB 失效（CTR.IminLine/tb_invalidate_phys_range）、AArch32 CP15 缓存操作（MCR p15 c7 全表）、分支预测操作（BPIALLUIS/BPIALL NOP）、HCR_EL2 缓存陷阱控制表、SCTLR 缓存控制位（M/C/I/UCI）、QEMU TLB 交互（sctlr_write TLB 刷新）、内存类型属性简化、完整指令映射表、缓存模拟策略总结图。

**适合读者**：需要理解 QEMU 缓存处理策略、DC ZVA 实现、缓存 ID 寄存器配置的开发者。
**关键源文件**：`target/arm/helper.c`（~10200行）、`target/arm/tcg/helper-a64.c`（~1000行）、`target/arm/cpu.h`（~3500行）、`target/arm/cpu-features.h`（~1700行）、`target/arm/cpu64.c`（~1000行）

### [13-ARM-Generic-Timer深度分析.md](arm/13-ARM-Generic-Timer深度分析.md)
> **18KB · 37 节**

ARM Generic Timer 完整实现分析：定时器类型枚举（7 种：PHYS/VIRT/HYP/SEC/HYPVIRT/S_EL2_PHYS/S_EL2_VIRT）、CPUARMState 定时器状态（c14_cntfrq/c14_cntkctl/cnthctl_el2/cntvoff_el2/cntpoff_el2）、ARMGenericTimer 结构（ctl/cval）、QOM 集成（CPU 内嵌定时器/gt_timer_outputs GPIO/QEMUTimer）、gt_get_countervalue() 计数器实现（QEMU_CLOCK_VIRTUAL/频率周期换算）、generic_timer_cp_reginfo[] 系统寄存器数组、CNTFRQ_EL0 频率寄存器（纯软件值/最高 EL 可写）、CNTKCTL_EL1 访问控制（EL0PCTEN/EL0VCTEN/VHE 重定向）、CNTP_*/CNTV_* 物理/虚拟定时器（CTL/CVAL/TVAL/Secure 分离）、Hypervisor 定时器 CNTHP/CNTHV、安全定时器 CNTPS、CNTVOFF_EL2 虚拟偏移（虚拟计数器=物理-偏移）、CNTHCTL_EL2 陷阱控制、CTL 位定义（ENABLE/IMASK/ISTATUS）、gt_ctl_write() 控制写入（ENABLE 切换→重算/IMASK 切换→更新 IRQ）、gt_recalc_timer() 核心算法（无符号 64 位比较/QEMUTimer 调度/溢出处理）、gt_update_irq() 中断输出（ISTATUS&&!IMASK/RME CNTVMASK/CNTPMASK 覆盖）、CVAL/TVAL 写入转换、虚拟计数器偏移计算、访问控制函数体系（gt_cntfrq/counter/ptimer/vtimer/stimer_access）、Timer→GIC PPI 连接（PPI 14/11/10/13）、virt 机器接线（qdev_connect_gpio_out）、设备树描述、RME/Realm 掩码、KVM 定时器（硬件直通）、ECV 扩展（CNTPOFF_EL2/CNTVCTSS）、定时器与 EL 关系表、完整触发流程图、定时器类型对比表。

**适合读者**：需要理解 ARM Generic Timer 架构、定时器→GIC 中断路径、虚拟化定时器偏移的开发者。
**关键源文件**：`target/arm/helper.c`（~10200行）、`target/arm/gtimer.h`（~24行）、`target/arm/cpu.h`（~3500行）、`target/arm/cpu.c`（~3000行）、`hw/arm/virt.c`（~3000行）

### [14-ARM-PMUv3性能监控单元深度分析.md](arm/14-ARM-PMUv3性能监控单元深度分析.md)
> **21KB · 38 节**

ARM PMUv3 性能监控单元完整实现分析：PMU 版本检测（ID_AA64DFR0.PMUVER/ARM_FEATURE_PMU/pmuv3p1-p5）、CPUARMState PMU 状态（c9_pmcr/pmcnten/pmovsr/pmuserenr/pmselr/pminten/c15_ccnt/c14_pmevcntr[31]/pmevtyper[31]）、pm_event 事件定义框架（supported/get_count/ns_per_count 三回调）、支持事件列表（SW_INCR/CPU_CYCLES/INST_RETIRED/STALL*）、CPU_CYCLES 实现（虚拟时钟×1GHz 映射/ARM_CPU_FREQ）、INST_RETIRED 实现（仅 icount 精确模式/icount_get_raw）、STALL 类事件（始终零/无流水线）、v7_pm_reginfo[] 寄存器定义、PMCR_EL0 控制（E/P/C/D/X/DP/LC/LP/N）、pmcr_write() 复位逻辑（pmu_op_start/finish 包裹）、pmcr_read() HPMN 覆盖、PMCNTENSET/CLR 使能位图、PMCCNTR_EL0 延迟更新机制（delta 快照）、pmccntr_op_start() 溢出检测（高位翻转）、pmccntr_op_finish() 溢出预测调度（timer_mod_anticipate_ns）、pmevcntr_op_start/finish 事件计数器（对称设计）、pmu_op_start/finish 批量操作、PMEVCNTRn/PMEVTYPERn 配置、pmevtyper_write() 事件切换（delta 基准重置）、PMSWINC 软件递增（逐位检查+溢出）、PMOVSR/PMOVSSET 溢出状态（写 1 清/设）、PMINTENSET/CLR 中断使能、pmu_update_irq() 中断条件（PMCR.E && PMINTEN & PMOVSR）、PMU 中断连接（pmu_interrupt→GIC PPI）、访问控制体系（access_tpm/do_pmreg_access）、PMUSERENR_EL0 精细控制（EN/SW/CR/ER 4 位）、MDCR_EL2.TPM/TPMCR/HPMN 陷阱与分区、PMCCFILTR_EL0 EL 过滤（P/U/NSK/NSU/NSH/M）、64 位计数器 PMUv3p5（LP/HLP）、时钟分频 D/LC 交互、PMU 初始化流程（pmu_init/pmceid 计算）、KVM PMU 直通（KVM_ARM_VCPU_PMU_V3）、EL 状态变化处理（pmu_pre_el_change）、迁移支持（rawwrite delta 修正）、完整工作流程图。

**适合读者**：需要理解 QEMU PMU 事件计数机制、溢出中断调度、虚拟化 PMU 分区的开发者。
**关键源文件**：`target/arm/cpregs-pmu.c`（~1300行）、`target/arm/cpu.h`（~3500行）、`target/arm/cpu.c`（~3000行）、`target/arm/helper.c`（~10200行）

### [15-ARM调试架构深度分析.md](arm/15-ARM调试架构深度分析.md)
> **24KB · 45 节**

ARM 自托管调试架构完整实现分析：CPUARMState 调试寄存器（dbgbvr/dbgbcr/dbgwvr/dbgwcr[16]/mdscr_el1/oslsr_el1/osdlr_el1/mdcr_el2/mdcr_el3）、MDSCR_EL1 调试系统控制（SS 单步/KDE 内核调试/MDE 监控使能）、OSLAR/OSLSR/OSDLR OS 锁机制（oslar_write 上锁/解锁）、调试寄存器访问控制三层陷阱（access_tdosa/tdra/tda/tdcc + MDCR_EL2.TDE 总开关）、DBGBVR/DBGBCR 断点寄存器（16 对/地址匹配/BAS/SSC/HMC/PAC/BT 类型/链式）、dbgbvr/dbgbcr_write() 写入同步 TCG（hw_breakpoint_update）、DBGWVR/DBGWCR 监视点寄存器（16 对/地址+BAS/MASK 范围/LSC 读写控制）、dbgwvr/dbgwcr_write() 写入同步、define_debug_regs() 动态注册（arm_num_brps/wrps）、arm_debug_target_el() 调试目标 EL 决策（TDE/TGE→EL2/Secure→EL3/默认 EL1）、调试异常生成条件（OS 锁检查/安全策略/EL 级别/PSTATE.D）、aa64_generate_debug_exceptions()（EL3 禁止/SDD 安全禁用/同 EL 需 KDE+!D）、aa32_generate_debug_exceptions()（SDER.SUIDEN/SPD/认证开放）、bp_wp_matches() 核心匹配（SSC 安全状态/PAC-HMC 特权/链式检查）、linked_bp_matches() 链式断点、hw_breakpoint_update() TCG 绑定（地址匹配实现/上下文 UNIMP）、hw_watchpoint_update() TCG 绑定（MASK/BAS 范围计算/LSC 方向）、arm_debug_check_breakpoint()（MDE 检查/单步优先/PC 对齐优先）、arm_debug_check_watchpoint()、arm_debug_excp_handler()（WP→DATA_ABORT/BP→PREFETCH_ABORT/GDB 优先）、BRK 指令 AArch64（syn_aa64_bkpt）、BKPT 指令 AArch32（syn_aa32_bkpt）、HELPER(exception_bkpt_insn)（目标 EL < 当前 EL 时在当前 EL 处理）、软件单步（MDSCR.SS+PSTATE.SS 状态机）、arm_singlestep_active()（仅 AArch64 目标 EL）、TCG 单步翻译（单指令 TB/禁止链接/gen_step_complete_exception）、调试异常综合征表（EC 值映射）、arm_debug_exception_fsr()、DCC 通信通道（RAZ/WI 存根）、DBGAUTHSTATUS 认证开放、MDCR_EL2 Hypervisor 控制（TDE/TDA/TDOSA/TDRA/TDCC）、MDCR_EL3 安全控制（SDD/SPD）、GDB 集成（BP_GDB vs BP_CPU）、KVM 调试（KVM_EXIT_DEBUG/hw_breakpoint）、完整断点/监视点触发流程图、调试异常优先级表。

**适合读者**：需要理解 ARM 自托管调试、硬件断点/监视点、调试异常路由的开发者。
**关键源文件**：`target/arm/debug_helper.c`（~530行）、`target/arm/tcg/debug.c`（~760行）、`target/arm/cpu.h`（~3500行）、`target/arm/tcg/translate-a64.c`（~10000行）

### [16-ARM-SVE-SME可伸缩向量矩阵扩展深度分析.md](arm/16-ARM-SVE-SME可伸缩向量矩阵扩展深度分析.md)
> **15KB · 33 节**

ARM SVE/SME 完整实现分析：CPUARMState 向量状态（vfp.zregs[32]/pregs[17]/FFR/zcr_el[4]/smcr_el[4]）、ARMVectorReg 固定分配（最大 2048 位）、SVCR 寄存器（PSTATE.SM/ZA）、ZA 矩阵存储（256×256 字节/Tile 行交错映射）、ZT0 (SME2)、sve_vqm1_for_el_sm() 有效 VL 计算（ZCR_EL1/2/3 嵌套取最小值 + sve_vq.map 位图匹配）、zcr_write() VL 缩小处理（aarch64_sve_narrow_vq）、arm_cpu_sve_finalize() 属性验证（sve-max-vq/sve\<N\> 配置/2 次幂自动启用/KVM VL 发现）、sve_exception_el() 陷阱判断三层（CPACR_EL1.ZEN/CPTR_EL2.TZ/CPTR_EL3.EZ）、sme_exception_el()（CPACR_EL1.SMEN/CPTR_EL2.TSM/CPTR_EL3.ESM）、sve_access_check() 翻译检查（SM 模式分支/FP 检查）、SMCR_ELx 寄存器（FA64/EZT0）、aarch64_set_svcr() Streaming 模式切换（SM 切换→reset SVE/ZA 使能→清零矩阵）、arm_reset_sve_state()（zregs/pregs 清零/FPSR 重置）、aarch64_sve_change_el() EL 切换处理（AArch64↔AArch32+SM→reset/VL 缩小→narrow）、SVE 指令翻译（translate-sve.c/gen_helper_sve_*）、KVM SVE 直通（KVM_ARM_VCPU_SVE/寄存器保存恢复）、陷阱控制完整表、VL 生命周期流程图、ZA Tile 映射详解。

**适合读者**：需要理解 SVE 向量长度管理、SME Streaming 模式、SVE/SME 陷阱控制的开发者。
**关键源文件**：`target/arm/cpu.h`（~3500行）、`target/arm/helper.c`（~10200行）、`target/arm/tcg/translate-sve.c`（~8000行）、`target/arm/tcg/sve_helper.c`（~7000行）、`target/arm/cpu64.c`（~1000行）

### [17-ARM-TrustZone安全扩展深度分析.md](arm/17-ARM-TrustZone安全扩展深度分析.md)
> **12KB · 32 节**

ARM TrustZone 安全扩展完整实现分析：ARMSecuritySpace 枚举（Secure/NonSecure/Root/Realm 四状态）、arm_security_space()/arm_is_secure() 查询体系、SCR_EL3 关键位（NS/IRQ/FIQ/EA/HCE/SIF/RW/EEL2/NSE）、NSE:NS 组合与四种安全空间映射、Secure EL2 使能（SCR.EEL2/FEAT_SEL2）、安全/非安全寄存器分组 Banking（bank_fieldoffsets/ARM_CP_SECSTATE_S/NS/BOTH）、add_cpreg_to_hashtable_aa32() 分组注册（_S 后缀副本）、arm_phys_excp_target_el() 中断安全路由（SCR.IRQ/FIQ/EA + HCR.IMO/FMO/AMO + TGE 查表）、GICv3 Group 0/1 路由、AArch32 SMC→Monitor 模式（清除 SCR.NS）、PSCI（arm_handle_psci_call/CPU_ON/OFF/RESET）、virt 机器 PSCI conduit 选择（SMC/HVC/DISABLED）、virt 机器安全内存（secure-memory 容器/virt.secure-ram/CPU secure-memory 链接）、tz-mpc 内存保护控制器（IOMMU_IDX_S/NS/安全区域划分）、tz-ppc 外设保护控制器（端口安全配置/违规中断）、RME 寄存器（GPCCR_EL3/GPTBR_EL3/MFAR_EL3）、TF-A 安全启动流程（BL1→BL31→BL33）、Monitor 模式异常向量表、安全世界切换完整流程（NS→S/S→NS）、安全状态与地址空间映射表。

**适合读者**：需要理解 TrustZone 安全/非安全世界切换、安全中断路由、PSCI 电源管理的开发者。
**关键源文件**：`include/hw/arm/arm-security.h`（~35行）、`target/arm/cpu.h`（~3500行）、`target/arm/helper.c`（~10200行）、`hw/arm/virt.c`（~3000行）、`hw/misc/tz-mpc.c`（~400行）、`hw/misc/tz-ppc.c`（~300行）

### [18-ARM内存模型与内存序深度分析.md](arm/18-ARM内存模型与内存序深度分析.md)
> **14KB · 40 节**

ARM 内存模型完整实现分析：DMB/DSB 实现（trans_DSB_DMB→tcg_gen_mb()/TCG_BAR_SC+TCG_MO_LD_LD 等/DSB 与 DMB 等价处理）、ISB 实现（TB 结束/非内存屏障/gen_goto_tb）、SB 推测屏障（MB+TB 结束）、TCG 内存序位定义（TCG_MO_LD_LD/ST_LD/LD_ST/ST_ST/ALL + TCG_BAR_LDAQ/STRL/SC）、tcg_gen_mb() 条件生成（仅 parallel_cpus 时有效）、STLR Store-Release（屏障在存储前/TCG_BAR_STRL）、LDAR Load-Acquire（屏障在加载后/TCG_BAR_LDAQ）、LDAPR (FEAT_LRCPC)、gen_load_exclusive() 独占加载（记录 exclusive_addr/val/MTE 检查）、gen_store_exclusive() 独占存储（地址匹配+atomic_cmpxchg/Rd=0 成功/1 失败）、LDXR/STXR/LDXP/STXP/STLXR/LDAXR 变体（lasr Acquire/Release 附加）、CLREX 清除监视、CAS/CASP 比较交换（atomic_cmpxchg_i64/i128）、LSE 原子 Fetch 操作（LDADD/LDCLR/LDEOR/LDSET/SWP 等→tcg_gen_atomic_*）、内存类型与属性（arm_mmu_idx_el()/Device vs Normal/S1_attrs_are_device()）、MAIR 属性间接（attrindx×8 提取 8 位）、Stage 1/2 属性合并（combine_cacheattrs）、MMU 禁用默认属性、MTE 标签检查（gen_mte_check1/HELPER(mte_check)/对齐优先/mte_check_fail 同步/异步/DC ZVA 特殊路径）、PAC 指针认证（5 密钥/HELPER(pacia/pacib/pacda/pacdb/pacga)/SCTLR 使能位/认证失败延迟检测）、BTI 分支目标标识（btype_destination_ok()/PSTATE.BTYPE/BTI c/j/jc 兼容矩阵）、MTTCG 内存序（单线程 NOP/多线程真屏障/后端映射）、页表遍历顶层流程、监视点与屏障独立性。

**适合读者**：需要理解 ARM 内存屏障映射、独占访问机制、LSE 原子操作、MTE/PAC/BTI 安全特性的开发者。
**关键源文件**：`target/arm/tcg/translate-a64.c`（~10800行）、`include/tcg/tcg-mo.h`（~50行）、`tcg/tcg-op.c`（~3000行）、`target/arm/tcg/mte_helper.c`（~1020行）、`target/arm/tcg/pauth_helper.c`（~620行）、`target/arm/ptw.c`（~3600行）

### [19-VirtIO设备模型与传输层深度分析.md](arm/19-VirtIO设备模型与传输层深度分析.md)
> **10KB · 12 节**

VirtIO 半虚拟化 I/O 完整实现分析：VirtIODevice 核心结构（status/isr/queue_sel/host_features/guest_features/vq 数组）、VirtQueue 与 Vring 机制（Split 三环 VRingDesc/VRingAvail/VRingUsed + Packed 单环 VRingPackedDesc/VIRTIO_F_RING_PACKED 协商）、VRingMemoryRegionCaches 预映射加速、设备生命周期（virtio_init 初始化 VIRTIO_QUEUE_MAX 个队列/virtio_add_queue 注册回调/feature 协商流程）、MMIO 传输层（平坦寄存器 0x000-0x100+/Magic/Version/DeviceID/QueueSel/QueueNotify/ioeventfd 加速）、PCI 传输层（VirtIOPCIProxy/Modern Capability 结构/Legacy I/O BAR/MSI-X 每队列独立向量）、通知机制双路径（Guest→Device: ioeventfd 或 handle_output 回调; Device→Guest: ISR+IRQ 或 MSI-X 或 irqfd）、virtio-net/virtio-blk 实例、vhost 数据面卸载（vhost-net 内核/vhost-user 用户进程/ioeventfd+irqfd 旁路 QEMU）、ARM virt 集成（create_virtio_devices 创建 32 个 virtio-mmio 连接 GIC SPI）。

**适合读者**：需要理解 VirtIO 设备模型、virtqueue 数据面路径、MMIO/PCI 传输差异、vhost 卸载机制的开发者。
**关键源文件**：`include/hw/virtio/virtio.h`（~300行）、`hw/virtio/virtio.c`（~4300行）、`hw/virtio/virtio-mmio.c`（~700行）、`hw/virtio/virtio-pci.c`（~2500行）、`hw/virtio/vhost.c`（~2300行）

### [20-PCI-PCIe子系统深度分析.md](arm/20-PCI-PCIe子系统深度分析.md)
> **8KB · 14 节**

PCI/PCIe 总线子系统完整实现分析：PCIDevice 核心结构（config[]/wmask[]/w1cmask[]/devfn/io_regions[7]/irq_state/cap_present）、配置空间管理（PCI 256B/PCIe 4KB/pci_default_read/write_config/wmask 过滤）、BAR 机制（pci_register_bar 注册/I/O vs MMIO 地址空间选择/64-bit BAR 跨寄存器/pci_bar_address 计算/pci_update_mappings 动态映射）、PCIBus 总线模型（pci_register_root_bus/address_space_mem/io）、GPEX Host Bridge（gpex_host_realize/ECAM+MMIO+PIO 三窗口/pcie.0 根总线/INTx swizzle 路由）、ECAM 配置访问（bus<<20|dev<<15|fn<<12|reg 编址/每功能 4KB）、INTx 中断路由（pci_irq_handler→bus IRQ→GIC SPI/swizzle 旋转）、MSI（msi_init/msi_notify 消息构造）、MSI-X（独立 BAR Table/PBA/msix_init/每队列独立向量/KVM irqfd）、PCIe Capability（pcie_cap_init/Device/Link Capabilities）、AER 错误报告、ARM virt PCIe 集成（create_pcie/ECAM+MMIO+MMIO_HIGH+PIO 布局/4 INTx→GIC）、VFIO 设备直通（BAR 直映/MSI-X irqfd/IOMMU DMA）、热插拔（SHPC/Native PCIe Slot）。

**适合读者**：需要理解 PCI 设备模型、BAR 映射机制、MSI-X 中断、GPEX Host Bridge、VFIO 直通的开发者。
**关键源文件**：`include/hw/pci/pci_device.h`（~200行）、`hw/pci/pci.c`（~2000行）、`hw/pci-host/gpex.c`（~250行）、`hw/pci/pcie.c`（~600行）、`hw/pci/msix.c`（~500行）、`hw/pci/pcie_host.c`（~150行）

### [21-ARM-SMMUv3与IOMMU框架深度分析.md](arm/21-ARM-SMMUv3与IOMMU框架深度分析.md)
> **9KB · 13 节**

ARM SMMUv3 与 QEMU IOMMU 框架完整实现分析：通用 IOMMU 框架（IOMMUMemoryRegion/IOMMUTLBEntry/translate() 接口/address_space_translate_iommu 集成）、SMMUv3 设备模型（SMMUv3State 寄存器/队列/IRQ/MMIO 处理）、流表查找（线性/2级流表 STE 查找 smmu_find_ste/L1→L2 稀疏优化/STE 解码 Bypass/S1/S2/Nested 模式/CD 查找与解码 TTB0/TTB1/ASID/配置缓存）、页表遍历（smmuv3_translate 入口/IOTLB 缓存查找/smmu_ptw_64_s1 Stage-1 IOVA→IPA 逐级遍历/smmu_ptw_64_s2 Stage-2 IPA→PA/嵌套翻译 S1 表地址需 S2 翻译）、命令队列（CFGI 配置无效/TLBI TLB 无效/CMD_SYNC 同步/PROD-CONS 环形缓冲区）、事件队列（翻译/权限/配置错误记录/EVENTQ IRQ）、IOTLB 缓存（GHashTable 实现/ASID+VMID+IOVA 键/多粒度无效化 inv_all/iova/ipa/asid_vmid/vmid）、IRQ（GERROR/EVENTQ/CMD_SYNC/PRI 四类中断/pulse 模式）、ARM virt 集成（create_smmu 链接 GPEX/iommu-map BDF→SID）、VFIO 交互（map/unmap 通知/iommufd 嵌套翻译）。

**适合读者**：需要理解 IOMMU 地址翻译框架、SMMUv3 流表/页表遍历、TLB 管理、VFIO IOMMU 交互的开发者。
**关键源文件**：`include/system/memory.h`（~600行）、`hw/arm/smmuv3.c`（~2100行）、`hw/arm/smmu-common.c`（~850行）、`include/hw/arm/smmu-common.h`（~180行）、`hw/arm/smmuv3-internal.h`（~300行）

---

### [22-ARM-TrustZone安全组件模拟深度分析.md](arm/22-ARM-TrustZone安全组件模拟深度分析.md)
> **13KB · 11 节**

ARM TrustZone 安全组件硬件模拟深度分析：TZ-MPC 内存保护控制器（IOMMU 双索引 S/NS、BLK_LUT 块级位图安全配置、tz_mpc_cfg_ns 查找、tz_mpc_translate 翻译核心、tz_mpc_handle_block 违规捕获与 IRQ、tz_mpc_attrs_to_index 属性映射、寄存器映射 CTRL/BLK_MAX/BLK_IDX/BLK_LUT/INT_*）、TZ-PPC 外设保护控制器（tz_ppc_check 三层安全检查 nonsec_mask/cfg_nonsec/cfg_ap、代理读写转发/阻塞、per-port 独立配置）、TZ-MSC 主设备安全控制器（MSCAction 四种决策、IDAU 接口集成、cfg_nonsec 判定）、安全内存架构（virt 机器 secure_sysmem 容器+优先级覆盖、Secure RAM 映射、PSCI conduit 选择）、MMU 安全态处理（S1Translate in_space/cur_space/out_space 三空间追踪、NSTable 降级机制、Stage-2 物理索引选择、TTBR Banking）、中断安全路由（target_el_table 6 维查表、SCR_EL3 路由位、GICv3 安全组 Group0/G1S/G1NS）、世界切换（SMC→Monitor 模式/SCR.NS 清除、switch_mode 寄存器 banking R8-R14/SPSR、ERET 返回）、完整安全事务路径追踪（Secure/NS CPU 访问 MPC 保护内存全流程）。

**适合读者**：需要理解 TrustZone 硬件安全模拟、MPC/PPC/MSC 设备原理、安全内存架构、MMU 安全态追踪的开发者。
**关键源文件**：`hw/misc/tz-mpc.c`（~560行）、`hw/misc/tz-ppc.c`（~360行）、`hw/misc/tz-msc.c`（~300行）、`target/arm/ptw.c`（~2600行）、`target/arm/helper.c`（~10188行）

---

### [23-ARM-RME-Realm管理扩展深度分析.md](arm/23-ARM-RME-Realm管理扩展深度分析.md)
> **16KB · 11 节**

ARM Realm Management Extension（RME/FEAT_RME）完整实现分析：四世界安全模型（ARMSecuritySpace 枚举 Secure/NonSecure/Root/Realm、GPI 低 2 位对应枚举值）、SCR_EL3.NSE 扩展位（NSE:NS 组合选择四世界、NSE=1+NS=0 保留处理）、arm_security_space() 四世界判定（EL3+RME→Root、arm_security_space_below_el3 NSE/NS 映射）、CPU 特性（ID_AA64PFR0.RME 字段、isar_feature_aa64_rme/gpc2 检测、x-rme/x-l0gptsz QOM 属性）、RME 系统寄存器（GPCCR_EL3 全字段 GPC/PPS/PGS/L0GPTSZ/SH/SPAD/NSPAD/RLPAD/SA/NSP/NA6/NA7/NSO、GPTBR_EL3 基地址、MFAR_EL3 故障地址）、GPT 两级遍历（Level-0 Block/Table descriptor、Level-1 Contiguous/Granule descriptor、16 种 GPI 值解释 0000-1111）、四级优先级检查（配置有效性/地址空间禁用/PPS 超限/GPTBR 超限）、GPC 检查时机（get_phys_addr_gpc MMU 翻译后执行）、GPC 故障类型（GPCFOnWalk/GPCFOnOutput、GPCF 子类型 AddressSize/Walk/EABT/Fail）、FSC 编码、Realm MMU 处理（ARMMMUIdx_Phys_Realm/Root 物理索引、ptw_idx_for_stage_2 Realm 分支）、四世界架构总览（EL3=Root/三个 Lower 世界）、Realm 与 TrustZone 对比（页面级 vs 块级隔离、机密计算 vs 固件安全）。

**适合读者**：需要理解 ARM RME 四世界模型、GPT 颗粒保护表、GPC 故障机制、Realm 机密计算架构的开发者。
**关键源文件**：`include/hw/arm/arm-security.h`（~37行）、`target/arm/helper.c`（~10188行）、`target/arm/ptw.c`（~2600行）、`target/arm/internals.h`（~1100行）、`target/arm/cpu-features.h`（~1200行）

### [24-ARM中断控制器架构GICv2与GICv3对比分析.md](arm/24-ARM中断控制器架构GICv2与GICv3对比分析.md)
> **11KB · 11 节**

ARM GIC 两代架构全面对比：核心差异总览（CPU 数 8→无限、MMIO→系统寄存器、位图路由→亲和性路由、无 LPI→原生 ITS）、QOM 类型层次（arm_gic_common→arm_gic/kvm-arm-gic vs arm-gicv3-common→arm-gicv3/kvm-arm-gicv3）、状态结构对比（GICState 统一结构 vs GICv3State+GICv3CPUState 分层、irq_target[] 8 位位图 vs gicd_irouter[] 64 位亲和性、sgi_pending[][] 源追踪 vs 无源追踪、MMIO MemoryRegion vs ARMCPRegInfo 系统寄存器）、中断流转对比（gic_set_irq vs gicv3_set_irq、gic_update_internal 全局扫描 vs gicv3_update 增量更新、SGI IAR 源 CPU<<10|ID vs 仅 INTID）、分发器/CPU 接口寄存器对比（GICD_ITARGETSR vs GICD_IROUTER、GICC_* MMIO vs ICC_*_ELx 系统寄存器、GICV_* vs ICV_* 虚拟化）、机器模型选择（virt.c finalize_gic_version_do NOSEL/host/max/2/3/4、CPU≤8 默认 v2、>8 强制 v3）、GICv2m MSI 桥接（arm-gicv2m 写入→SPI 转换）、KVM 加速对比（设备类型/MMIO 拦截/状态迁移差异）、VMSTATE 迁移差异（v2 统一 vs v3 分层+条件子段）、演进趋势（MMIO→寄存器、位图→亲和性、外挂→原生 ITS、单文件→分层架构）。

**适合读者**：需要理解 GICv2/v3 架构差异、QEMU 如何选择和创建 GIC、两代实现的代码组织和状态管理差异的开发者。
**关键源文件**：`hw/intc/arm_gic.c`（~2200行）、`include/hw/intc/arm_gic_common.h`（~166行）、`hw/intc/arm_gicv2m.c`（~199行）、`hw/arm/virt.c`（~4300行）、`hw/intc/arm_gic_kvm.c`、`hw/intc/arm_gicv3_kvm.c`

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
