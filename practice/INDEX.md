# QEMU 实践总结

本目录包含 QEMU 的监测、调试和使用实践指南。

---

## 文档列表

| # | 文件 | 主题 | 行数 |
|---|------|------|------|
| 102 | [监测与追踪系统](102-QEMU监测与追踪系统深度分析-Tracing框架-QMP监控-Stats指标-事件通知-性能集成.md) | Tracing 6后端, QMP查询, Stats框架, 事件通知, HMP 71命令 | ~500 |
| 103 | [调试系统](103-QEMU调试系统深度分析-GDB-Stub-TCG调试-HMP命令-CoreDump-ARM64调试架构.md) | GDB Stub RSP, TCG日志30+分类, Core Dump, ARM64调试寄存器 | ~480 |
| 104 | [基础用法](104-QEMU基础用法实践指南-命令行-CPU-内存-存储-网络-显示-固件-Monitor.md) | 编译安装, CPU/内存/存储/网络/显示配置, Monitor, 常用场景 | ~300 |
| 105 | [高级用法](105-QEMU高级用法实践指南-KVM调优-VirtIO-VFIO直通-迁移-快照-安全-自动化.md) | KVM优化, VirtIO多队列, VFIO直通, 热迁移, 安全, 自动化 | ~450 |
| 106 | [GDB远程调试完全指南](106-QEMU-GDB远程调试完全指南-RSP命令-ARM64寄存器-断点观察点-monitor透传-反向调试.md) | RSP协议40+命令, ARM64寄存器组, 断点观察点, monitor透传, 反向调试, 物理内存模式 | ~830 |
| 110 | [Linux内核调试工作流](110-QEMU-GDB调试Linux内核完整工作流-编译配置-Rootfs制作-启动参数-断点调试-模块调试-自动化.md) | ARM64内核编译, Rootfs制作(3种), QEMU启动配置, GDB内核调试, 模块调试, 7个调试场景, 自动化脚本 | ~450 |
| 111 | [性能分析与Profiling](111-QEMU性能分析与Profiling实践指南-perf集成-TCG统计-ftrace-sync-profile-PMU-调优.md) | perfmap/jitdump, sync-profile, trace events, Guest perf/ftrace, PMU仿真, 代码缓存调优, 火焰图 | ~430 |
| 112 | [开发者指南:添加设备与指令](112-QEMU开发者指南-添加设备与指令-QOM模型-SysBus-PCI-decode-TCG翻译-测试.md) | QOM设备模型, SysBus/PCI完整示例, ARM64指令添加, decode文件, Helper, 构建系统, QTest | ~650 |

---

## 推荐阅读顺序

1. **新手入门**: 104 (基础用法) → 103 (调试) → 106 (GDB详细命令) → 110 (内核调试)
2. **性能调优**: 105 (高级用法) → 102 (监测)
3. **问题排查**: 103 (调试) → 106 (GDB命令) → 110 (实战调试) → 102 (追踪) → 105 §14 (检查清单)

## 与源码分析文档的关系

| 实践主题 | 对应源码分析 |
|----------|--------------|
| KVM 调优 | architecture/86, 98, 99, 100 |
| VirtIO | device-model/10, 21, 26 |
| 网络 | device-model/06-网络, network/ |
| 迁移 | architecture/89 |
| TCG 调试 | tcg/*, arm64/tcg/* |
| GDB | debug/ |
| UI/Display | architecture/101 |
