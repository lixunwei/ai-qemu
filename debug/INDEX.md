# 调试子系统 (Debug)

GDB Stub、硬件断点、Watchpoint、单步执行、ARM 调试架构规范验证。

| # | 文档 | 主题 |
|---|------|------|
| 00 | [GDB调试子系统](00-GDB调试子系统深度分析.md) | GDB Stub 整体架构 |
| 18 | [Debug/Breakpoint/Watchpoint](18-Debug-Breakpoint-Watchpoint调试子系统深度分析-硬件断点-单步与GDB-Stub.md) | 硬件断点、Watchpoint、单步 |
| 53 | [ARM64调试架构(MDSCR/断点/观察点)](53-ARM64-调试架构深度分析-MDSCR-DBGBCR-DBGWCR-断点观察点-单步执行.md) | ARM64 MDSCR、DBGBCR/DBGWCR |
| 61 | [ARM64调试架构规范验证(DDI0487)](61-ARM64-调试架构规范验证-DDI0487-D2-H-断点观察点-单步-Debug异常路由-MDSCR对照.md) | DDI 0487 D2/H 对照验证 |
| 109 | [GDB Stub源码级完整调试会话Walkthrough](109-GDB-Stub源码级完整调试会话Walkthrough-启动连接-断点命中-寄存器读取-单步分离-调用链.md) | 7阶段源码追踪: 启动→连接→断点→继续→寄存器→单步→分离 |
| 113 | [GDB x/xp内存访问原理](113-GDB-x-xp内存访问原理-虚拟地址翻译-PhyMemMode-物理地址直读-ARM64页表遍历.md) | 虚拟/物理地址分发, PhyMemMode, cpu_memory_rw_debug, ARM64页表遍历, Monitor x vs xp |
