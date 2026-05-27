# QEMU 11.0.50 ARM64 源码深度分析文档库

> 基于 QEMU 11.0.50 源码，结合 ARM Architecture Reference Manual (DDI 0487)、
> GIC 规范 (IHI 0069H) 等官方文档，对 QEMU ARM64 模拟器进行全面深度分析。
> 所有文档使用中文撰写。

---

## 目录结构

| 目录 | 文档数 | 主题 |
|------|--------|------|
| [architecture/](architecture/INDEX.md) | 26 | 全局架构、QOM、线程模型、事件循环、KVM、UI |
| [tcg/](tcg/INDEX.md) | 15 | TCG 核心翻译引擎 (IR/优化/后端/MTTCG/TLB/Plugin) |
| [arm64/](arm64/INDEX.md) | 18+51 | ARM64 模拟 (通用 + 6 个子目录) |
| ├─ [arm64/gic/](arm64/gic/INDEX.md) | 14 | GIC 中断控制器 |
| ├─ [arm64/mmu/](arm64/mmu/INDEX.md) | 9 | MMU/TLB/页表遍历 |
| ├─ [arm64/security/](arm64/security/INDEX.md) | 10 | 安全特性 (TrustZone/PAC/BTI/MTE/RME) |
| ├─ [arm64/exception/](arm64/exception/INDEX.md) | 12 | 异常处理/EL 切换/Trap |
| ├─ [arm64/tcg/](arm64/tcg/INDEX.md) | 8 | ARM64 TCG 翻译细节 |
| └─ [arm64/spec-verify/](arm64/spec-verify/INDEX.md) | 7 | ARM 规范验证对照 |
| [arm/](arm/INDEX.md) | 18 | ARM 通用 (含 AArch32) |
| [device-model/](device-model/INDEX.md) | 21 | 设备模型 (VirtIO/Block/PCI/VFIO/DMA/Timer) |
| [memory/](memory/INDEX.md) | 6 | 内存子系统 (MemoryRegion/FlatView/RAMBlock) |
| [debug/](debug/INDEX.md) | 4 | GDB 调试子系统 |
| [monitor/](monitor/INDEX.md) | 1 | Monitor/QMP/QAPI |
| [network/](network/INDEX.md) | 2 | 网络子系统 |
| [practice/](practice/INDEX.md) | 5 | 实践指南 (监测/调试/用法/GDB) |
| **总计** | **~168** | |

---

## 快速导航

### 按学习路线

**路线 A: QEMU 架构入门**
1. architecture/00 (全局概览) → architecture/01 (QOM) → architecture/02 (执行循环)
2. architecture/04 (线程模型) → architecture/05 (事件循环)
3. architecture/22 (QOM 深入) → architecture/97 (QOM 增补: Resettable/Visitor)

**路线 B: TCG 翻译引擎**
1. tcg/00 (概览) → tcg/02 (IR 设计) → tcg/13 (前端翻译) → tcg/78 (IR+优化 Pass)
2. tcg/03/14 (后端代码生成) → tcg/04/12 (MTTCG) → tcg/05 (Softmmu TLB)
3. arm64/tcg/42 (ARM64 前后端) → arm64/tcg/44 (执行循环) → arm64/tcg/85 (浮点/SVE)

**路线 C: ARM64 异常与 EL 切换**
1. arm64/exception/35 (EL 切换) → arm64/exception/52 (完整异常流程)
2. arm64/exception/41 (TCG hflags) → arm64/exception/70 (ERET 返回)
3. arm64/exception/69 (SMC/HVC/Trap) → arm64/exception/74 (FGT)

**路线 D: GIC 中断**
1. arm64/gic/03 (GICv3 架构) → arm64/gic/50 (Dist/Redist/CPU IF)
2. arm64/gic/24 (完整生命周期) → arm64/gic/27 (中断虚拟化 ICH/ICV)
3. arm64/gic/71 (IRQ/FIQ 路由) → arm64/gic/72 (vGIC 注入链路)
4. arm64/gic/28 (KVM vGIC) → arm64/gic/25 (ITS/LPI)

**路线 E: MMU 与内存**
1. memory/00 (内存子系统) → memory/11 → memory/24 (MemoryRegion 深入)
2. arm64/mmu/11 (MMU/TLB) → arm64/mmu/30 (系统寄存器) → arm64/mmu/49 (PTW)
3. arm64/mmu/55 (Hypervisor Stage-2) → arm64/mmu/51 (缓存属性)

**路线 F: 安全特性**
1. arm64/security/08 (TrustZone) → arm64/security/39 (EL3 世界切换)
2. arm64/security/75 (MTE) → arm64/security/76 (PAC) → arm64/security/77 (BTI)
3. arm64/security/17 (GCS/RME)

**路线 G: KVM 加速器**
1. architecture/86 (KVM 集成概览) → architecture/98 (内存管理)
2. architecture/99 (ARM64 VM Exit/特性) → architecture/100 (硬件对比)
3. arm64/gic/28 (KVM vGIC)

**路线 H: 设备模型**
1. device-model/00 (概览) → device-model/10/21/26 (VirtIO)
2. device-model/20 (PCI/PCIe) → device-model/05 (VFIO)
3. device-model/04/06/07/25 (Block 层) → device-model/06-网络 (网络)

**路线 I: 机器启动**
1. architecture/82/90 (virt Machine) → architecture/83 (内核加载)
2. architecture/84/92 (固件/UEFI) → arm64/81/91 (PSCI 多核)
3. architecture/94 (CPU 热插拔) → architecture/88 (Reset/Clock)

**路线 J: UI/显示**
1. architecture/101 (UI/Display 全面: VNC/GTK/Spice/输入/渲染管线)

### 按主题查找

| 想了解... | 推荐文档 |
|-----------|----------|
| QEMU 怎么翻译 ARM64 指令？ | [tcg/78](tcg/78-TCG前端IR生成与优化Pass深度分析-操作码体系-常量折叠-拷贝传播-活性分析-寄存器分配.md) |
| 中断注入全链路？ | [arm64/gic/24](arm64/gic/24-GICv3完整中断生命周期深度分析.md) |
| EL 切换发生了什么？ | [arm64/exception/35](arm64/exception/35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md) |
| TrustZone 安全世界切换？ | [arm64/security/39](arm64/security/39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md) |
| 页表遍历实现？ | [arm64/mmu/49](arm64/mmu/49-ARM64-页表遍历PTW深度分析-Stage1-Stage2翻译-权限检查-Fault处理-安全属性传播.md) |
| QEMU 与 ARM 规范的差异？ | [arm64/spec-verify/67](arm64/spec-verify/67-ARM64-QEMU规范验证差异总汇-DDI0487-IHI0069-全部子系统对照结论与勘误清单.md) |
| KVM 内存/脏页/ioeventfd？ | [architecture/98](architecture/98-KVM内存管理深度分析-MemorySlot-脏页追踪-ioeventfd-irqfd快速路径.md) |
| KVM VM Exit 处理？ | [architecture/99](architecture/99-KVM-ARM64特性与VM-Exit深度分析-退出处理-特性协商-DeviceAPI-cpreg-线程模型.md) |
| 硬件 vs QEMU 实现差异？ | [architecture/100](architecture/100-ARM64硬件虚拟化与QEMU-KVM实现对比分析-EL2-Stage2-vGIC-Timer-Trap-差异汇总.md) |
| VNC/GTK/Spice 显示？ | [architecture/101](architecture/101-UI显示子系统全面分析-VNC-GTK-Spice-输入-渲染管线-DisplayChangeListener.md) |
| QOM 对象模型？ | [architecture/22](architecture/22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md) + [architecture/97](architecture/97-QOM对象模型增补分析-Resettable三阶段重置-Visitor模式-DEFINE_PROP宏-QMP联动.md) |
| 热迁移？ | [architecture/89](architecture/89-热迁移子系统深度分析-迭代传输-RAM脏页-Postcopy-Multifd.md) |
| 确定性重放？ | [architecture/95](architecture/95-Replay确定性重放深度分析-事件日志-块设备-网络-字符设备-快照-反向调试.md) |
| icount/虚拟时间？ | [architecture/93](architecture/93-icount指令计数与确定性执行深度分析-虚拟时间-TB预算-Warp-Record-Replay.md) |
| QGA 来宾代理？ | [architecture/96](architecture/96-QGA来宾代理深度分析-通信通道-命令派发-文件系统冻结-进程执行-资源管理.md) |

---

## 全文档列表（按目录）

### architecture/ (26 篇)

| # | 文件 | 主题 |
|---|------|------|
| 00 | 全局架构概览 | QEMU 整体架构鸟瞰 |
| 01 | QOM 对象模型 | TypeInfo/ObjectClass/Property |
| 02 | 模拟执行循环 | cpu_exec/MMIO 分发 |
| 03 | Machine 建立流程 | virt machine 初始化 |
| 04 | 线程模型与锁机制 | BQL/iothread/vCPU |
| 05 | 事件循环与 IO 模型 | AioContext/fd handler |
| 22 | QOM 深度分析 | TypeInfo/Property/接口继承 |
| 23 | 主事件循环与协程 | AioContext/BH/Coroutine |
| 27 | 架构子系统综合导航 | 索引文档 |
| 82 | virt Machine 初始化 | 内存映射/GIC/PCIe/VirtIO |
| 83 | ARM 内核加载与启动 | bootloader/Image/initrd/DTB |
| 84 | 固件 UEFI 启动 | PFlash/CFI/fw_cfg/EDK2 |
| 86 | KVM 加速器集成 | ioctl/vCPU 循环/寄存器同步 |
| 88 | Reset-Clock 框架 | 三阶段复位/时钟树 |
| 89 | 热迁移子系统 | 迭代传输/RAM 脏页/Postcopy |
| 90 | virt Machine 初始化 (补) | GIC/PCIe/ACPI/FDT |
| 92 | 固件启动链 | PFlash/CFI01/EDK2/RVBAR |
| 93 | icount 指令计数 | 虚拟时间/TB 预算/Warp |
| 94 | CPU 热插拔 | 拓扑/GIC Redist/ACPI GED |
| 95 | Replay 确定性重放 | 事件日志/快照/反向调试 |
| 96 | QGA 来宾代理 | 通道/命令/文件冻结 |
| 97 | QOM 增补 | Resettable/Visitor/DEFINE_PROP |
| 98 | KVM 内存管理 | MemorySlot/脏页/ioeventfd |
| 99 | KVM ARM64 特性 | VM Exit/特性协商/Device API |
| 100 | ARM64 硬件对比 | EL2/Stage2/vGIC/Timer/差异 |
| 101 | UI/Display | VNC/GTK/Spice/输入/渲染 |

### tcg/ (15 篇)

| # | 文件 | 主题 |
|---|------|------|
| 00 | TCG 深度分析 | 概览 |
| 01 | TCG 优化递次 | 优化 Pass 层次 |
| 02 | TCG IR 设计与前端翻译 | IR 操作码/前端 |
| 03 | TCG 后端代码生成与 TB 管理 | 后端/TB |
| 04 | MTTCG 多线程翻译 | 并行执行 |
| 05 | Softmmu TLB 与内存访问 | TLB/慢路径 |
| 06 | TCG Plugin 系统 | 插件 API |
| 07 | Linux-user 用户模式翻译 | 用户态 |
| 08 | TCG 加速器综合导航 | 索引 |
| 08(b) | TCG 后端深度分析 | IR/寄存器分配/代码缓存 |
| 09 | TCG 深入: 优化/向量/TLB | 补充 |
| 12 | 多线程 TCG | MTTCG/TB 失效/屏障 |
| 13 | TCG 前端翻译 | 指令解码/IR/优化 |
| 14 | TCG 后端: AArch64 | 寄存器分配/TLB 慢路径 |
| 78 | TCG IR+优化 Pass | 常量折叠/拷贝传播/活性 |

### arm64/ (18 篇通用)

| # | 文件 | 主题 |
|---|------|------|
| 00 | ARM64 CPU/GICv3/TCG | 综合入口 |
| 01 | ACPI 表生成 | DSDT/MADT/GTDT |
| 02 | 特殊指令模拟 | WFI/HVC/SMC/MRS/MSR |
| 05 | FDT 设备树 | DTB 生成/设备节点 |
| 09 | 虚拟化扩展 | VHE/HCR_EL2/Stage2 |
| 12 | Generic Timer | 定时器模拟 |
| 13 | PMU | 性能计数器 |
| 14 | CPU 特性与 ID 寄存器 | ISAR/MMFR/特性发现 |
| 15 | SVE/SME | 向量长度/ZA/流模式 |
| 16 | PAC/BTI/MTE | 安全特性概览 |
| 32 | 系统寄存器与 Cache/AT | 特殊指令 |
| 34 | ID 寄存器与特性发现 | 补充 |
| 47 | 系统寄存器 CP 访问 | ARMCPRegInfo/cpregs |
| 54 | EL 状态管理综合导航 | 索引 |
| 79 | Semihosting | HLT 0xF000/系统调用 |
| 81 | PSCI 电源协调 | CPU_ON/OFF/conduit |
| 91 | PSCI 多核启动 (补) | SYSTEM_RESET/Conduit |
| INDEX | 索引 | - |

### arm64/exception/ (12 篇)

| # | 主题 |
|---|------|
| 07 | 不同 EL 下指令执行差异 |
| 31 | EL2/EL3 系统寄存器陷阱路由 |
| 35 | EL 状态切换: 异常进入返回/PSTATE |
| 36 | 不同 EL 指令执行流变化 |
| 40 | EL1-EL2 交互: HVC/VHE/嵌套虚拟化 |
| 41 | EL 切换 TCG 翻译变化: hflags/TB 键 |
| 52 | 完整异常处理流程: VBAR/PSTATE/返回 |
| 57 | EL 状态规范验证: DDI0487 对照 |
| 68 | WFE/WFI Trap 路径 |
| 69 | SMC/HVC/MSR Trap 机制 |
| 70 | ERET 异常返回: SPSR/ELR/非法返回 |
| 74 | Fine-Grained Traps (FGT) |

### arm64/gic/ (14 篇)

| # | 主题 |
|---|------|
| 03 | GICv3 模拟架构 |
| 04 | 中断注入与处理 |
| 23 | 安全/非安全中断路由 |
| 24 | GICv3 完整中断生命周期 |
| 25 | ITS 中断翻译服务/LPI |
| 26 | GICv3 寄存器模拟与状态机 |
| 27 | 中断虚拟化 ICH/ICV/LR |
| 28 | KVM vGIC 设备后端/直通 |
| 50 | GICv3: Dist/Redist/CPU IF/路由/优先级 |
| 56 | GIC 规范验证补充 |
| 71 | 异步异常 IRQ/FIQ/SError 路由 |
| 72 | 中断虚拟化完整链路: vGIC/LR/注入 |
| INDEX | 索引 |

### arm64/mmu/ (9 篇)

| # | 主题 |
|---|------|
| 11 | MMU/TLB 深度分析 |
| 30 | MMU 系统寄存器与页表遍历 |
| 38 | 内存管理: 页表/TLB/Stage2/属性合并 |
| 49 | 页表遍历 PTW: Stage1/Stage2/权限/Fault |
| 51 | 内存属性与缓存一致性: MAIR/TCR/DC |
| 55 | Hypervisor 虚拟化: Stage1+Stage2 |
| 58 | MMU 规范验证: DDI0487 D8 对照 |
| 66 | Cache 一致性与 TLB 广播规范验证 |
| INDEX | 索引 |

### arm64/security/ (10 篇)

| # | 主题 |
|---|------|
| 08 | TrustZone/Secure World |
| 17 | GCS/RME 新扩展 |
| 37 | 安全状态转换: SCR_EL3/HCR_EL2 联动 |
| 39 | EL3 Secure 世界切换: SMC/Monitor/ERET |
| 48 | TrustZone: SCR_EL3/世界切换/隔离 |
| 62 | TrustZone 规范验证: DDI0487 D1 对照 |
| 75 | MTE 内存标签: Tag 地址/IRG/故障 |
| 76 | PAC 指针认证: QARMA/签名/FPAC |
| 77 | BTI 分支目标: BTYPE/Guarded Page |
| INDEX | 索引 |

### arm64/tcg/ (8 篇)

| # | 主题 |
|---|------|
| 42 | TCG 前后端代码生成: IR/翻译/寄存器分配 |
| 43 | TCG softmmu TLB: 数据结构/快慢路径 |
| 44 | TCG 执行循环: cpu_exec/TB/中断/MTTCG |
| 45 | TCG 内存模型: 屏障/MemOp/Exclusive/LSE |
| 46 | TCG 插件与调试: Plugin/GDB/断点 |
| 73 | TCG TB 边界与中断检查时序 |
| 85 | 浮点/NEON/SVE 指令翻译 |
| 87 | Crypto 扩展翻译: AES/SHA/SM3/SM4 |

### arm64/spec-verify/ (7 篇)

| # | 主题 |
|---|------|
| 59 | Generic Timer 规范验证 |
| 60 | 指令集模拟规范验证 |
| 63 | SVE/SME 规范验证 |
| 64 | PAC/BTI/MTE 规范验证 |
| 65 | 内存模型与屏障规范验证 |
| 67 | QEMU 规范验证差异总汇 |
| INDEX | 索引 |

### arm/ (18 篇)

| # | 主题 |
|---|------|
| 08 | ARM64 EL 状态管理与指令执行 |
| 09 | AArch32 异常处理与模式切换 |
| 10 | CP15 系统寄存器与 MMU 页表 |
| 11 | GICv3 中断控制器 |
| 12 | ARM Cache 管理与维护操作 |
| 13 | ARM Generic Timer |
| 14 | ARM PMUv3 性能监控 |
| 15 | ARM 调试架构 |
| 16 | ARM SVE/SME |
| 17 | ARM TrustZone 安全扩展 |
| 18 | ARM 内存模型与内存序 |
| 19 | VirtIO 设备模型与传输层 |
| 20 | PCI/PCIe 子系统 |
| 21 | ARM SMMUv3 与 IOMMU |
| 22 | ARM TrustZone 安全组件模拟 |
| 23 | ARM RME Realm 管理扩展 |
| 24 | GICv2 与 GICv3 对比 |
| 25 | ARM 架构模拟综合导航 |

### device-model/ (21 篇)

| # | 主题 |
|---|------|
| 00 | 设备模型与 VirtIO 概览 |
| 02 | 块层 IO 路径 |
| 03 | Chardev 与 UART |
| 04 | Block 设备子系统 |
| 05 | VFIO 设备直通与 IOMMU |
| 06 | 块层核心架构 |
| 06(b) | 网络后端: TAP/vhost-net/vhost-user |
| 07 | DMA 设备模拟架构 |
| 07(b) | qcow2 格式与块 IO 后端 |
| 08 | ARM Generic Timer (设备视角) |
| 09 | 设备模型综合导航 |
| 10 | VirtIO: Virtqueue/vhost-user/IOMMU |
| 16 | 时钟与定时器子系统 |
| 17 | PMU 性能监控 |
| 19 | SMMUv3 IOMMU |
| 20 | PCI/PCIe: BAR/MSI/ECAM |
| 21 | VirtIO: VRing/通知/PCI/MMIO/vhost |
| 25 | Block Layer IO: BDS/协程/限流 |
| 26 | VirtIO: virtio-blk/net/PCI |
| INDEX | 索引 |

### memory/ (6 篇)

| # | 主题 |
|---|------|
| 00 | 内存子系统概览 |
| 01 | RAM 管理与脏页追踪 |
| 02 | 内存子系统综合导航 |
| 11 | MemoryRegion 树/FlatView/RAMBlock |
| 15 | MemoryRegion/AddressSpace/FlatView/IOMMU |
| 24 | MemoryRegion/AddressSpace 深度分析 |

### debug/ (4 篇)

| # | 主题 |
|---|------|
| 00 | GDB 调试子系统 |
| 18 | Debug 断点/观察点/GDB Stub |
| 53 | ARM64 调试架构: MDSCR/DBGBCR/DBGWCR |
| 61 | 调试架构规范验证: DDI0487 D2/H |

### monitor/ (1 篇)

| # | 主题 |
|---|------|
| 00 | Monitor/QMP/QAPI: 命令/协议/Schema/Visitor |

### network/ (2 篇)

| # | 主题 |
|---|------|
| 00 | 网络子系统概览 |
| 01 | 网络综合导航: TAP/SLIRP/VirtIO/vhost |

---

## 覆盖度统计

### 已覆盖子系统 ✅

| 子系统 | 文档数 | 覆盖深度 |
|--------|--------|----------|
| QOM 对象模型 | 4 | ⭐⭐⭐⭐⭐ |
| TCG 翻译引擎 | 15+8 | ⭐⭐⭐⭐⭐ |
| ARM64 异常/EL | 12 | ⭐⭐⭐⭐⭐ |
| GIC 中断控制器 | 14 | ⭐⭐⭐⭐⭐ |
| MMU/TLB/页表 | 9 | ⭐⭐⭐⭐⭐ |
| 安全 (TZ/PAC/BTI/MTE) | 10 | ⭐⭐⭐⭐⭐ |
| KVM 加速器 | 4 | ⭐⭐⭐⭐ |
| 设备模型 (VirtIO/Block/PCI) | 21 | ⭐⭐⭐⭐ |
| 内存子系统 | 6 | ⭐⭐⭐⭐ |
| virt Machine/启动 | 6 | ⭐⭐⭐⭐ |
| 热迁移 | 1 | ⭐⭐⭐ |
| 确定性重放 | 2 | ⭐⭐⭐ |
| UI/Display | 1 | ⭐⭐⭐ |
| 调试 | 4+1 | ⭐⭐⭐⭐ |
| 网络 | 2 | ⭐⭐ |
| Monitor/QMP | 1 | ⭐⭐ |
| ARM 规范验证 | 7 | ⭐⭐⭐⭐ |
| 硬件对比分析 | 1 | ⭐⭐⭐ |

### 待覆盖 ❌

| 子系统 | 说明 |
|--------|------|
| Snapshot/VMState | 快照保存/恢复机制 |

---

### practice/ (5 篇)

| # | 主题 |
|---|------|
| 102 | 监测与追踪系统: Tracing 6后端, QMP, Stats, HMP |
| 103 | 调试系统: GDB Stub, TCG日志, Core Dump, ARM64调试 |
| 104 | 基础用法: 编译, CPU/内存/存储/网络/显示配置 |
| 105 | 高级用法: KVM调优, VirtIO, VFIO, 迁移, 安全 |
| 106 | GDB远程调试完全指南: RSP协议, ARM64寄存器, 断点, 反向调试 |

---

## 源码版本

- **QEMU**: 11.0.50 (commit 基于 2025 年 main 分支)
- **参考规范**: DDI 0487 M.b (ARMv9.6-A), IHI 0069H.b (GICv3/v4)
- **文档总数**: 168 篇 (截至 2025-07)
