# ARM64 TCG 翻译细节

ARM64 专属 TCG 翻译：前端/后端代码生成、softmmu TLB、执行循环、内存模型、插件/调试。

| # | 文档 | 主题 |
|---|------|------|
| 42 | [TCG前端后端代码生成](42-ARM64-TCG前端后端代码生成深度分析-IR中间表示-翻译循环-优化Pass-寄存器分配与AArch64代码发射.md) | IR 表示、翻译循环、AArch64 后端发射 |
| 43 | [TCG-softmmu-TLB](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) | TLB 数据结构、快慢路径、MMIO |
| 44 | [TCG执行循环](44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md) | cpu_exec、TB 查找/链接、MTTCG |
| 45 | [TCG内存模型与原子操作](45-ARM64-TCG内存模型与原子操作深度分析-屏障语义-MemOp标志-Exclusive-LSE原子与后端发射.md) | 屏障、Exclusive、LSE 原子 |
| 46 | [TCG插件与调试子系统](46-ARM64-TCG插件与调试子系统深度分析-PluginAPI-GDBStub-断点单步-ARM调试寄存器与Tracing.md) | Plugin API、GDB、断点/单步 |
| 73 | [TB边界与中断检查时序](73-ARM64-TCG-TB边界与中断检查时序-icount_decr机制-TB-chain-中断延迟分析.md) | icount_decr、中断延迟 |
