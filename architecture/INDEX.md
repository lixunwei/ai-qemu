# 架构 (Architecture)

QEMU 全局架构、对象模型、执行循环、线程模型等核心设计文档。

| # | 文档 | 主题 |
|---|------|------|
| 00 | [全局架构概览](00-全局架构概览.md) | QEMU 整体架构设计、模块划分 |
| 01 | [QOM对象模型深度分析](01-QOM对象模型深度分析.md) | QOM 类型系统、属性、继承 |
| 02 | [模拟执行循环与MMIO分发深度分析](02-模拟执行循环与MMIO分发深度分析.md) | CPU 执行循环、MMIO 地址分发 |
| 03 | [Machine建立流程深度分析](03-Machine建立流程深度分析.md) | Machine 初始化、设备树构建 |
| 04 | [线程模型与锁机制深度分析](04-线程模型与锁机制深度分析.md) | BQL、iothread、锁层次 |
| 05 | [事件循环与IO模型深度分析](05-事件循环与IO模型深度分析.md) | AioContext、fd handler、BH |
| 22 | [QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型](22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md) | TypeInfo、ObjectClass、Property、接口继承 |
| 23 | [主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread](23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md) | AioContext、Coroutine、IOThread |
| 27 | [QEMU架构子系统综合导航-QOM-内存-TCG-VirtIO-事件循环-Block-PCI](27-QEMU架构子系统综合导航-QOM-内存-TCG-VirtIO-事件循环-Block-PCI.md) | 全部子系统交叉索引 |
| 82 | [virt-Machine初始化与设备创建深度分析-内存映射-CPU创建-GIC-PCIe-VirtIO-ACPI-DT](82-virt-Machine初始化与设备创建深度分析-内存映射-CPU创建-GIC-PCIe-VirtIO-ACPI-DT.md) | virt 内存映射、CPU 创建、GIC/PCIe/VirtIO/ACPI/DT 设备创建 |
| 83 | [ARM内核加载与启动流程深度分析-bootloader桩-Image加载-initrd-DTB放置-CPU复位](83-ARM内核加载与启动流程深度分析-bootloader桩-Image加载-initrd-DTB放置-CPU复位.md) | bootloader 桩、Image/initrd/DTB 加载、CPU 复位入口 |
| 84 | [固件UEFI启动流程深度分析-PFlash-CFI-fw_cfg-UEFI变量服务-EDK2集成](84-固件UEFI启动流程深度分析-PFlash-CFI-fw_cfg-UEFI变量服务-EDK2集成.md) | PFlash、CFI、fw_cfg、UEFI 变量、EDK2 集成 |
| 86 | [KVM加速器集成深度分析-ioctl-vCPU运行循环-寄存器同步-ARM64架构钩子](86-KVM加速器集成深度分析-ioctl-vCPU运行循环-寄存器同步-ARM64架构钩子.md) | KVM ioctl 路径、vCPU 运行循环、寄存器同步 |
| 88 | [Reset-Clock框架深度分析-三阶段复位协议-时钟树模型-设备集成](88-Reset-Clock框架深度分析-三阶段复位协议-时钟树模型-设备集成.md) | 三阶段 Reset 协议、Clock 树、设备时钟集成 |
| 89 | [热迁移子系统深度分析-迭代传输-RAM脏页-Postcopy-Multifd](89-热迁移子系统深度分析-迭代传输-RAM脏页-Postcopy-Multifd.md) | RAM 脏页、迭代迁移、Postcopy、Multifd |
| 90 | [virt-Machine初始化深度分析-内存布局-GIC创建-PCIe-ACPI-FDT-固件加载](90-virt-Machine初始化深度分析-内存布局-GIC创建-PCIe-ACPI-FDT-固件加载.md) | virt 内存布局、GIC/PCIe/ACPI/FDT、固件装载 |
| 92 | [固件启动链深度分析-PFlash-CFI01-EDK2-UEFI-RVBAR-fw_cfg-安全启动](92-固件启动链深度分析-PFlash-CFI01-EDK2-UEFI-RVBAR-fw_cfg-安全启动.md) | PFlash、EDK2、UEFI、RVBAR、fw_cfg、安全启动链 |
| 93 | [icount指令计数与确定性执行深度分析-虚拟时间-TB预算-Warp-Record-Replay](93-icount指令计数与确定性执行深度分析-虚拟时间-TB预算-Warp-Record-Replay.md) | 虚拟时间、TB 预算、Warp、Record/Replay |
| 94 | [CPU热插拔深度分析-拓扑预分配-GIC-Redistributor-ACPI-GED-PSCI启动](94-CPU热插拔深度分析-拓扑预分配-GIC-Redistributor-ACPI-GED-PSCI启动.md) | CPU 拓扑预分配、GIC Redistributor、ACPI GED、PSCI 热插拔 |
| 95 | [Replay确定性重放深度分析-事件日志-块设备-网络-字符设备-快照-反向调试](95-Replay确定性重放深度分析-事件日志-块设备-网络-字符设备-快照-反向调试.md) | 事件日志、块/网络/字符设备重放、快照、反向调试 |
| 96 | [QGA来宾代理深度分析-通信通道-命令派发-文件系统冻结-进程执行-资源管理](96-QGA来宾代理深度分析-通信通道-命令派发-文件系统冻结-进程执行-资源管理.md) | QGA 通信通道、命令派发、冻结/执行/资源管理 |
| 97 | [QOM对象模型增补分析-Resettable三阶段重置-Visitor模式-DEFINE_PROP宏-QMP联动](97-QOM对象模型增补分析-Resettable三阶段重置-Visitor模式-DEFINE_PROP宏-QMP联动.md) | Resettable 重置、Visitor 模式、DEFINE_PROP、QMP 联动 |
| 98 | [KVM内存管理深度分析-MemorySlot-脏页追踪-ioeventfd-irqfd快速路径](98-KVM内存管理深度分析-MemorySlot-脏页追踪-ioeventfd-irqfd快速路径.md) | MemorySlot、脏页追踪、ioeventfd、irqfd |
| 99 | [KVM-ARM64特性与VM-Exit深度分析-退出处理-特性协商-DeviceAPI-cpreg-线程模型](99-KVM-ARM64特性与VM-Exit深度分析-退出处理-特性协商-DeviceAPI-cpreg-线程模型.md) | VM-Exit 处理、特性协商、Device API、cpreg、线程模型 |
| 100 | [ARM64硬件虚拟化与QEMU-KVM实现对比分析-EL2-Stage2-vGIC-Timer-Trap-差异汇总](100-ARM64硬件虚拟化与QEMU-KVM实现对比分析-EL2-Stage2-vGIC-Timer-Trap-差异汇总.md) | EL2、Stage-2、vGIC、Timer、Trap 实现对比 |
| 101 | [UI显示子系统全面分析-VNC-GTK-Spice-输入-渲染管线-DisplayChangeListener](101-UI显示子系统全面分析-VNC-GTK-Spice-输入-渲染管线-DisplayChangeListener.md) | VNC、GTK、Spice、输入链路、渲染管线 |
| 108 | [快照与VMState子系统深度分析-SaveVM-LoadVM-QCOW2快照-RAM保存-设备状态序列化-版本兼容](108-QEMU快照与VMState子系统深度分析-SaveVM-LoadVM-QCOW2快照-RAM保存-设备状态序列化-版本兼容.md) | VMState 框架、SaveVM/LoadVM、QCOW2 内部快照、RAM 保存、ARM64 状态、版本兼容 |
