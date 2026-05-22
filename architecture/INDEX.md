# 架构 (Architecture)

QEMU 全局架构、对象模型、执行循环、线程模型等核心设计文档。

| # | 文档 | 主题 |
|---|------|------|
| 00 | [全局架构概览](00-全局架构概览.md) | QEMU 整体架构设计、模块划分 |
| 01 | [QOM对象模型深度分析](01-QOM对象模型深度分析.md) | QOM 类型系统、属性、继承 |
| 02 | [模拟执行循环与MMIO分发](02-模拟执行循环与MMIO分发深度分析.md) | CPU 执行循环、MMIO 地址分发 |
| 03 | [Machine建立流程](03-Machine建立流程深度分析.md) | Machine 初始化、设备树构建 |
| 04 | [线程模型与锁机制](04-线程模型与锁机制深度分析.md) | BQL、iothread、锁层次 |
| 05 | [事件循环与IO模型](05-事件循环与IO模型深度分析.md) | AioContext、fd handler、BH |
| 22 | [QOM对象模型(进阶)](22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md) | TypeInfo、ObjectClass、Property、接口继承 |
| 23 | [主事件循环与协程](23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md) | AioContext、Coroutine、IOThread |
| 27 | [架构子系统综合导航](27-QEMU架构子系统综合导航-QOM-内存-TCG-VirtIO-事件循环-Block-PCI.md) | 全部子系统交叉索引 |
