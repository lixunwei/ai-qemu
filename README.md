# QEMU 11.0.50 ARM64 源码深度分析文档库

> 基于 QEMU 11.0.50 源码，结合 ARM Architecture Reference Manual (DDI 0487)、
> GIC 规范 (IHI 0069H) 等官方文档，对 QEMU ARM64 模拟器进行全面深度分析。
> 所有文档使用中文撰写。

---

## 目录结构

| 目录 | 文档数 | 主题 |
|------|--------|------|
| [architecture/](architecture/INDEX.md) | 9 | 全局架构、QOM、线程模型、事件循环 |
| [tcg/](tcg/INDEX.md) | 15 | TCG 核心翻译引擎 (IR/优化/后端/MTTCG/TLB/Plugin) |
| [arm64/](arm64/INDEX.md) | 18+51 | ARM64 模拟 (通用 + 6 个子目录) |
| ├─ [arm64/gic/](arm64/gic/INDEX.md) | 12 | GIC 中断控制器 |
| ├─ [arm64/mmu/](arm64/mmu/INDEX.md) | 8 | MMU/TLB/页表遍历 |
| ├─ [arm64/security/](arm64/security/INDEX.md) | 9 | 安全特性 (TrustZone/PAC/BTI/MTE/RME) |
| ├─ [arm64/exception/](arm64/exception/INDEX.md) | 10 | 异常处理/EL 切换/Trap |
| ├─ [arm64/tcg/](arm64/tcg/INDEX.md) | 6 | ARM64 TCG 翻译细节 |
| └─ [arm64/spec-verify/](arm64/spec-verify/INDEX.md) | 6 | ARM 规范验证对照 |
| [arm/](arm/INDEX.md) | 18 | ARM 通用 (含 AArch32) |
| [device-model/](device-model/INDEX.md) | 19 | 设备模型 (VirtIO/Block/PCI/VFIO/DMA/Timer) |
| [memory/](memory/INDEX.md) | 6 | 内存子系统 (MemoryRegion/FlatView/RAMBlock) |
| [debug/](debug/INDEX.md) | 2 | GDB 调试子系统 |
| [monitor/](monitor/INDEX.md) | 1 | Monitor/QMP/QAPI |
| [network/](network/INDEX.md) | 2 | 网络子系统 |
| **总计** | **142** | |

---

## 快速导航

### 按学习路线

1. **入门**: architecture/00 → architecture/01 → architecture/02
2. **TCG 翻译**: tcg/00 → tcg/02 → tcg/03 → tcg/78
3. **ARM64 异常**: arm64/exception/35 → arm64/exception/52 → arm64/exception/70
4. **GIC 中断**: arm64/gic/03 → arm64/gic/24 → arm64/gic/27
5. **MMU 内存**: arm64/mmu/11 → arm64/mmu/30 → arm64/mmu/49
6. **安全特性**: arm64/security/08 → arm64/security/75 → arm64/security/76

### 按主题查找

- **想了解 QEMU 怎么翻译 ARM64 指令？** → [tcg/78](tcg/78-TCG前端IR生成与优化Pass深度分析-操作码体系-常量折叠-拷贝传播-活性分析-寄存器分配.md)
- **想了解中断注入全链路？** → [arm64/gic/24](arm64/gic/24-GICv3完整中断生命周期深度分析.md)
- **想了解 EL 切换发生了什么？** → [arm64/exception/35](arm64/exception/35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md)
- **想了解 TrustZone 安全世界切换？** → [arm64/security/39](arm64/security/39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md)
- **想了解页表遍历实现？** → [arm64/mmu/49](arm64/mmu/49-ARM64-页表遍历PTW深度分析-Stage1-Stage2翻译-权限检查-Fault处理-安全属性传播.md)
- **想了解 QEMU 与 ARM 规范的差异？** → [arm64/spec-verify/67](arm64/spec-verify/67-ARM64-QEMU规范验证差异总汇-DDI0487-IHI0069-全部子系统对照结论与勘误清单.md)

---

## 源码版本

- **QEMU**: 11.0.50 (commit 基于 2025 年 main 分支)
- **参考规范**: DDI 0487 M.b (ARMv9.6-A), IHI 0069H.b (GICv3/v4)
