# ARM64 异常处理与 EL 切换

异常进入/返回、PSTATE 管理、EL0-EL3 切换、Trap 机制 (WFE/WFI/SMC/HVC/MSR/FGT)、
ERET 返回、VHE 重定向。

| # | 文档 | 主题 |
|---|------|------|
| 07 | [不同EL下指令执行差异](07-不同EL下指令执行差异深度分析.md) | 各 EL 指令可用性差异 |
| 31 | [EL2-EL3系统寄存器陷阱路由](31-ARM64-EL2-EL3系统寄存器陷阱路由深度分析.md) | EL2/EL3 trap 路由规则 |
| 35 | [异常级别EL状态切换(PSTATE)](35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md) | 异常进入/返回、PSTATE 保存恢复 |
| 36 | [不同EL下指令执行流变化](36-ARM64不同EL下指令执行流变化深度分析.md) | EL 切换对执行流的影响 |
| 40 | [EL1-EL2交互(HVC/VHE/Stage2)](40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md) | HVC 陷入、VHE、嵌套虚拟化 |
| 41 | [EL切换TCG翻译变化(hflags/TB)](41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md) | hflags 位域、TB 键断裂 |
| 52 | [异常处理完整流程(VBAR/PSTATE)](52-ARM64-异常处理完整流程深度分析-同步异步异常入口-VBAR向量表-PSTATE保存恢复-异常返回.md) | VBAR 向量表、同步/异步异常 |
| 57 | [EL状态指令变化规范验证](57-ARM64-EL状态指令执行变化规范验证-EL1-EL3-异常入口返回-PSTATE-指令可用性-DDI0487对照.md) | DDI 0487 EL/PSTATE 对照 |
| 68 | [WFE-WFI-Trap路径](68-ARM64-WFE-WFI-Trap路径分析-HCR_EL2-TWE-SCTLR-nTWE-EL状态切换机制.md) | HCR_EL2.TWE/TWI trap 控制 |
| 69 | [SMC-HVC-MSR-Trap机制](69-ARM64-SMC-HVC-MSR-Trap机制分析-HCR_EL2-TSC-TVM-FGT-HSTR-系统寄存器访问路由.md) | TSC/TVM/FGT/HSTR trap |
| 70 | [ERET异常返回机制](70-ARM64-ERET异常返回机制分析-SPSR-ELR-PSTATE恢复-illegal-return-EL切换对称性.md) | SPSR/ELR 恢复、illegal return |
| 74 | [Fine-Grained-Traps(FGT)机制](74-ARM64-Fine-Grained-Traps-FGT机制分析-HFGRTR-HFGITR-声明式trap-REV-nXS豁免.md) | HFGRTR/HFGITR、声明式 trap |
