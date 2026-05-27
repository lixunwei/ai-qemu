# ARM64 模拟 (通用)

ARM64 CPU 模型、系统寄存器、ID 寄存器、调试架构等通用主题。
子主题请参见各子目录。

## 子目录

| 目录 | 主题 | 文档数 |
|------|------|--------|
| [gic/](gic/INDEX.md) | GIC 中断控制器 | 12 |
| [mmu/](mmu/INDEX.md) | MMU/TLB/页表遍历 | 8 |
| [security/](security/INDEX.md) | 安全特性 (TrustZone/PAC/BTI/MTE/RME) | 9 |
| [exception/](exception/INDEX.md) | 异常处理/EL切换/Trap | 12 |
| [tcg/](tcg/INDEX.md) | ARM64 TCG翻译细节 | 8 |
| [spec-verify/](spec-verify/INDEX.md) | 规范验证对照 | 6 |

## 本目录文档

| # | 文档 | 主题 |
|---|------|------|
| 00 | [ARM64-CPU-GICv3-TCG深度分析](00-ARM64-CPU-GICv3-TCG深度分析.md) | CPU 模型、GICv3、TCG 翻译总览 |
| 01 | [ACPI表生成与启动流程深度分析](01-ACPI表生成与启动流程深度分析.md) | ACPI 表构建、启动流程 |
| 02 | [特殊指令模拟深度分析](02-特殊指令模拟深度分析.md) | WFI/WFE/DMB/DSB 等特殊指令 |
| 05 | [FDT设备树深度分析](05-FDT设备树深度分析.md) | FDT 设备树生成与解析 |
| 09 | [虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU](09-虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU.md) | VHE、HCR_EL2、Stage-2 MMU |
| 12 | [Generic-Timer定时器深度分析](12-Generic-Timer定时器深度分析.md) | 物理/虚拟定时器、EL 控制 |
| 13 | [PMU性能监控单元深度分析](13-PMU性能监控单元深度分析.md) | PMUv3 事件计数器、溢出中断 |
| 14 | [CPU特性与ID寄存器仿真深度分析](14-CPU特性与ID寄存器仿真深度分析.md) | ID 寄存器仿真、特性发现 |
| 15 | [SVE-SME可扩展向量扩展深度分析](15-SVE-SME可扩展向量扩展深度分析.md) | SVE/SME 向量长度、谓词、ZA 矩阵 |
| 16 | [PAC-BTI-MTE安全特性深度分析](16-PAC-BTI-MTE安全特性深度分析.md) | 指针认证、分支目标、内存标签概览 |
| 32 | [ARM64特殊系统寄存器与Cache-AT指令深度分析](32-ARM64特殊系统寄存器与Cache-AT指令深度分析.md) | DC/IC/AT 指令、特殊寄存器 |
| 34 | [ARM64-ID寄存器与特性发现机制深度分析](34-ARM64-ID寄存器与特性发现机制深度分析.md) | ID_AA64* 寄存器族 |
| 47 | [ARM64-系统寄存器与CP访问深度分析-ARMCPRegInfo框架-MRS-MSR翻译-cpregs哈希表-EL银行与访问控制](47-ARM64-系统寄存器与CP访问深度分析-ARMCPRegInfo框架-MRS-MSR翻译-cpregs哈希表-EL银行与访问控制.md) | ARMCPRegInfo 框架、MRS/MSR 翻译 |
| 54 | [ARM64-异常级别状态管理综合导航-EL切换-指令差异-安全状态-TCG翻译变化](54-ARM64-异常级别状态管理综合导航-EL切换-指令差异-安全状态-TCG翻译变化.md) | EL 状态管理全部主题索引 |
| 79 | [ARM64-Semihosting指令模拟深度分析-HLT-0xF000-系统调用分发-文件描述符-控制台IO](79-ARM64-Semihosting指令模拟深度分析-HLT-0xF000-系统调用分发-文件描述符-控制台IO.md) | HLT semihosting、系统调用分发、文件/控制台 IO |
| 81 | [ARM64-PSCI电源状态协调接口与多核启动深度分析-CPU_ON-CPU_OFF-conduit-firmware_reset](81-ARM64-PSCI电源状态协调接口与多核启动深度分析-CPU_ON-CPU_OFF-conduit-firmware_reset.md) | PSCI CPU_ON/OFF、conduit、firmware reset |
| 91 | [PSCI多核启动与电源管理深度分析-CPU_ON-CPU_OFF-SYSTEM_RESET-Conduit决策](91-PSCI多核启动与电源管理深度分析-CPU_ON-CPU_OFF-SYSTEM_RESET-Conduit决策.md) | CPU_ON/OFF、SYSTEM_RESET、conduit 决策 |
