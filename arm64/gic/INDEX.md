# ARM64 GIC 中断控制器

GICv3 中断控制器模拟：Distributor/Redistributor/CPU Interface、中断生命周期、
虚拟化 (ICH/ICV/LR)、ITS/LPI、路由与优先级。

| # | 文档 | 主题 |
|---|------|------|
| 03 | [GICv3中断控制器模拟架构](03-GICv3中断控制器模拟架构深度分析.md) | GICv3 整体模拟架构 |
| 04 | [中断注入与处理](04-中断注入与处理深度分析.md) | IRQ/FIQ 注入、CPU 接口交互 |
| 23 | [安全与非安全中断路由流转](23-ARM64安全与非安全中断路由流转深度分析.md) | Secure/Non-Secure 中断路由 |
| 24 | [GICv3完整中断生命周期](24-GICv3完整中断生命周期深度分析.md) | Pending→Active→Deactivate 全链路 |
| 25 | [GICv3-ITS中断翻译服务与LPI](25-GICv3-ITS中断翻译服务与LPI深度分析.md) | ITS 命令队列、设备表、LPI |
| 26 | [GICv3寄存器模拟与状态机](26-GICv3寄存器模拟与状态机深度分析.md) | GICD/GICR/ICC 寄存器状态机 |
| 27 | [中断虚拟化ICH-ICV-LR状态机](27-ARM64中断虚拟化ICH-ICV-LR状态机深度分析.md) | ICH/ICV 接口、List Register |
| 28 | [KVM-vGIC设备后端与中断直通](28-KVM-vGIC设备后端与中断直通深度分析.md) | KVM vGIC、中断直通 |
| 50 | [GICv3 Distributor/Redistributor/CPUInterface](50-ARM64-GICv3中断控制器深度分析-Distributor-Redistributor-CPUInterface-中断路由与优先级.md) | 三大组件、优先级、路由 |
| 56 | [GIC规范验证(IHI0048B/IHI0069H)](56-GIC规范验证补充-ARM-IHI0048B-IHI0069H-对照勘误.md) | GIC 规范对照勘误 |
| 71 | [异步异常IRQ/FIQ/SError路由](71-ARM64-异步异常IRQ-FIQ-SError路由机制-HCR_EL2-IMO-FMO-DAIF屏蔽-虚拟中断注入.md) | HCR_EL2 IMO/FMO、DAIF 屏蔽 |
| 72 | [中断虚拟化完整链路](72-ARM64-中断虚拟化完整链路-vGIC-List-Register-VIRQ注入-ICH_LR-ICV接口-EOI去激活.md) | vGIC→LR→VIRQ→ICV→EOI 全链路 |
