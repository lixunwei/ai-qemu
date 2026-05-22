# ARM64 MMU/TLB/内存管理

MMU 系统寄存器、页表遍历 (PTW)、Stage-1/2 翻译、TLB 管理、
内存属性 (MAIR/TCR)、Cache 一致性。

| # | 文档 | 主题 |
|---|------|------|
| 11 | [MMU-TLB深度分析](11-MMU-TLB深度分析.md) | MMU/TLB 整体架构 |
| 30 | [MMU系统寄存器与页表遍历](30-ARM64-MMU系统寄存器与页表遍历深度分析.md) | TCR/TTBR/MAIR、4级页表 |
| 38 | [内存管理(页表/TLB/Stage2/属性)](38-ARM64内存管理深度分析-页表遍历-TLB-Stage2翻译与属性合并.md) | 页表遍历、TLB、Stage-2、属性合并 |
| 49 | [页表遍历PTW(Stage1/2/权限/Fault)](49-ARM64-页表遍历PTW深度分析-Stage1-Stage2翻译-权限检查-Fault处理-安全属性传播.md) | PTW 完整流程、Fault 处理 |
| 51 | [内存属性与缓存一致性](51-ARM64-内存属性与缓存一致性深度分析-MAIR-TCR属性编码-Device-Normal类型-缓存维护指令.md) | Device/Normal 类型、DC/IC 指令 |
| 55 | [Hypervisor虚拟化(Stage1/2 MMU)](55-ARM64-Hypervisor虚拟化深度分析-NonSecure-EL2-Secure-EL2-Stage1-Stage2-MMU翻译.md) | EL2 Stage-1/2 翻译 |
| 58 | [MMU地址翻译规范验证](58-ARM64-MMU地址翻译规范验证-DDI0487-D8-翻译体制-页表格式-权限检查-Fault类型对照.md) | DDI 0487 D8 对照验证 |
| 66 | [Cache一致性与TLB广播规范验证](66-ARM64-Cache一致性与TLB广播规范验证-DDI0487-B2-D8-DC-IC-TLBI-多核一致性-PoC-PoU对照.md) | DC/IC/TLBI、PoC/PoU 验证 |
