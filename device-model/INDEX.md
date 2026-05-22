# 设备模型 (Device Model)

VirtIO、Block/qcow2、Chardev/UART、PCI/PCIe、VFIO、网络后端、DMA、
Timer、PMU、SMMUv3/IOMMU。

| # | 文档 | 主题 |
|---|------|------|
| 00 | [设备模型与virtio框架](00-设备模型与virtio深度分析.md) | 设备模型概览、VirtIO 框架 |
| 02 | [块层IO路径](02-块层IO路径深度分析.md) | Block I/O 请求路径 |
| 03 | [Chardev子系统与UART交互](03-Chardev子系统与UART交互深度分析.md) | Chardev/UART |
| 04 | [Block设备子系统](04-Block设备子系统深度分析.md) | Block 设备层次 |
| 05 | [VFIO设备直通与IOMMU集成](05-VFIO设备直通与IOMMU集成深度分析.md) | VFIO 直通 |
| 06a | [块层核心架构](06-块层核心架构深度分析.md) | Block 核心架构 |
| 06b | [网络后端(TAP/vhost-net/vhost-user)](06-网络后端深度分析-TAP-vhost-net-vhost-user.md) | 网络后端 |
| 07a | [DMA设备模拟架构](07-DMA设备模拟架构深度分析.md) | DMA 模拟 |
| 07b | [qcow2格式驱动与块IO后端](07-qcow2格式驱动与块IO后端深度分析.md) | qcow2 格式 |
| 08 | [ARM-Generic-Timer(计数器/7类定时器)](08-ARM-Generic-Timer深度分析-计数器-7类定时器-EL访问控制-VHE重定向与KVM集成.md) | Timer 详细实现 |
| 09 | [设备模型子系统综合导航](09-设备模型子系统综合导航-VirtIO-Block-Chardev-VFIO-网络-DMA-Timer.md) | 设备模型索引 |
| 10 | [VirtIO(Virtqueue/vhost-user/IOMMU)](10-VirtIO设备深度分析-Virtqueue实现vhost-user与IOMMU集成.md) | Virtqueue、vhost-user |
| 16 | [时钟与定时器(QEMUTimer/ptimer/Clock)](16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md) | 定时器框架 |
| 17 | [PMU性能监控单元](17-PMU性能监控单元深度分析-事件计数器-溢出中断与EL过滤.md) | PMU 事件/溢出 |
| 19 | [SMMUv3-IOMMU(Stream Table/命令队列)](19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译.md) | SMMUv3 |
| 20 | [PCI-PCIe(BAR/MSI/ECAM)](20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md) | PCI/PCIe |
| 21 | [VirtIO(VirtQueue/VRing/vhost)](21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md) | VirtQueue/VRing |
| 25 | [Block-Layer-IO(BlockDriverState/协程)](25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md) | BDS、协程 I/O |
| 26 | [VirtIO(virtio-blk/net/PCI传输)](26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md) | virtio-blk/net |
