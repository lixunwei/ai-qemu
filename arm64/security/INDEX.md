# ARM64 安全特性

TrustZone 安全扩展、RME Realm 管理、PAC 指针认证、BTI 分支目标标识、MTE 内存标签。

| # | 文档 | 主题 |
|---|------|------|
| 08 | [TrustZone安全扩展与Secure-World](08-TrustZone安全扩展与Secure-World深度分析.md) | Secure/Non-Secure 世界隔离 |
| 17 | [GCS-RME及新扩展](17-GCS-RME及新扩展深度分析.md) | Guarded Control Stack、RME |
| 37 | [安全状态转换(SCR_EL3/HCR_EL2)](37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md) | SCR_EL3/HCR_EL2 联动、安全域 |
| 39 | [EL3-Secure世界切换(SMC/ERET)](39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md) | SMC 入口、Monitor、ERET 返回 |
| 48 | [安全扩展TrustZone(SCR_EL3/隔离)](48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md) | SCR_EL3、安全状态隔离 |
| 62 | [安全架构规范验证(DDI0487 D1)](62-ARM64-安全架构TrustZone规范验证-DDI0487-D1-安全状态-SCR_EL3-世界切换-RME四域模型对照.md) | TrustZone/RME 规范对照 |
| 75 | [MTE内存标签扩展](75-ARM64-Memory-Tagging-Extension-MTE实现分析-Tag地址空间-IRG随机生成-同步异步故障.md) | Tag 地址空间、IRG、Fault |
| 76 | [PAC指针认证](76-ARM64-Pointer-Authentication-PAC实现分析-QARMA算法-签名认证-FPAC异常-Trap控制.md) | QARMA、签名/认证、FPAC |
| 77 | [BTI分支目标标识](77-ARM64-Branch-Target-Identification-BTI实现分析-BTYPE状态机-Guarded-Page-Landing-Pad.md) | BTYPE 状态机、Guarded Page |
