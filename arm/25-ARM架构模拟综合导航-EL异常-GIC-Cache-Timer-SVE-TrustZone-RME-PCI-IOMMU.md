# ARM 架构模拟综合导航

> 基于 QEMU 11.0.50 源码分析，汇总 `arm/08-24` 共 17 篇文档的主题边界、阅读顺序与交叉关系。
> 本文定位为**导航页**：先建地图，再按问题跳到单篇深读。

---

## 1. ARM 子系统全景图

```text
                         ARM 子系统全景图

   EL/模式/安全态
   EL0-EL3 + AArch32/AArch64 + Secure/NS/Realm
                │
                ▼
      异常入口/返回与系统寄存器
   PSTATE/CPSR ─ VBAR/ERET ─ CP15/ARMCPRegInfo
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
    MMU       GIC       Cache/内存序
  页表/权限   IRQ/FIQ    DC/IC/Barrier/Atomic
      │         │         │
      └────┬────┴────┬────┘
           ▼         ▼
      Timer/PMU/Debug/SVE
           │
           ▼
   TrustZone / TZ-MPC/PPC / RME
           │
           ▼
   VirtIO / PCIe / SMMUv3 / IOMMU
```

可把这 17 篇文档看成 4 条主线：

1. **CPU/异常主线**：08 → 09 → 10 → 11  
2. **执行语义主线**：12 → 13 → 14 → 15 → 16 → 18  
3. **安全隔离主线**：17 → 22 → 23  
4. **平台设备主线**：19 → 20 → 21，并回连 11/17

---

## 2. 文档地图

| # | 文档 | 主题簇 | 核心内容 | 适合解答的问题 |
|---|---|---|---|---|
| **08** | [ARM64 EL状态管理与指令执行](08-ARM64-EL状态管理与指令执行深度分析.md) | EL/异常 | EL、PSTATE、异常进入返回 | EL 切换后 CPU 发生了什么？ |
| **09** | [AArch32异常处理与模式切换](09-AArch32异常处理与模式切换深度分析.md) | EL/异常 | 模式切换、banked register、异常向量 | 32 位异常为何更复杂？ |
| **10** | [CP15系统寄存器与MMU页表管理](10-CP15系统寄存器与MMU页表管理深度分析.md) | 系统寄存器/MMU | CP15、ARMCPRegInfo、页表 | 系统寄存器和 MMU 怎么连起来？ |
| **11** | [GICv3中断控制器](11-GICv3中断控制器深度分析.md) | 中断控制器 | GICD/GICR/ICC、虚拟中断 | 中断如何从设备送到 CPU？ |
| **12** | [ARM Cache管理与维护操作](12-ARM-Cache管理与维护操作深度分析.md) | Cache | CCSIDR/CSSELR/CTR、DC/IC | Cache 指令在 QEMU 里怎么体现？ |
| **13** | [ARM Generic Timer](13-ARM-Generic-Timer深度分析.md) | 定时器 | 计数器、频率、物理/虚拟定时器 | 定时器中断为何到这个 EL？ |
| **14** | [ARM PMUv3性能监控单元](14-ARM-PMUv3性能监控单元深度分析.md) | PMU | 事件、计数、溢出、EL 过滤 | PMU 如何计数并触发中断？ |
| **15** | [ARM调试架构](15-ARM调试架构深度分析.md) | 调试 | MDSCR/DBGBCR/DBGWCR、GDB | 断点/观察点/单步怎么工作？ |
| **16** | [ARM SVE/SME可伸缩向量矩阵扩展](16-ARM-SVE-SME可伸缩向量矩阵扩展深度分析.md) | SVE/SME | VL、陷阱、EL 切换影响 | 为何换 EL 后向量行为变了？ |
| **17** | [ARM TrustZone安全扩展](17-ARM-TrustZone安全扩展深度分析.md) | TrustZone | SCR_EL3、安全状态、中断路由 | Secure/NS 世界如何切换？ |
| **18** | [ARM内存模型与内存序](18-ARM内存模型与内存序深度分析.md) | 内存模型 | Barrier、Exclusive、LSE、MTE | ARM 的顺序保证在哪里？ |
| **19** | [VirtIO设备模型与传输层](19-VirtIO设备模型与传输层深度分析.md) | 设备模型 | virtqueue、通知、DMA | VirtIO 请求如何流动？ |
| **20** | [PCI/PCIe子系统](20-PCI-PCIe子系统深度分析.md) | 设备模型 | RC、BAR、配置空间、中断 | PCIe 设备如何接入 ARM 平台？ |
| **21** | [ARM SMMUv3与IOMMU框架](21-ARM-SMMUv3与IOMMU框架深度分析.md) | 设备模型 | DMA 翻译、Stream Table、IOMMU | 设备为何看到不同地址空间？ |
| **22** | [ARM TrustZone安全组件模拟](22-ARM-TrustZone安全组件模拟深度分析.md) | TrustZone | TZ-MPC/TZ-PPC/安全组件 | 外设访问为何被安全策略拦住？ |
| **23** | [ARM RME Realm管理扩展](23-ARM-RME-Realm管理扩展深度分析.md) | RME | 四世界、GPT、GPC | Realm 与 Secure/NS 如何区分？ |
| **24** | [ARM中断控制器架构 GICv2 与 GICv3 对比](24-ARM中断控制器架构GICv2与GICv3对比分析.md) | 中断控制器 | GICv2/v3 差异与迁移 | 为什么新平台倾向 GICv3？ |

---

## 3. EL / 异常

对应：[08](08-ARM64-EL状态管理与指令执行深度分析.md)、[09](09-AArch32异常处理与模式切换深度分析.md)

- **关键概念**：EL0-EL3、AArch32 模式、PSTATE/CPSR、SPSR、VBAR、ERET
- **阅读要点**：08 是 64 位总览，09 是 32 位兼容层与 banked register 补完
- **适用场景**：异常跳转错位、特权级行为差异、模式切换问题

## 4. 系统寄存器 / MMU

对应：[10](10-CP15系统寄存器与MMU页表管理深度分析.md)

- **关键概念**：CP15、ARMCPRegInfo、SCTLR/TTBR 类控制、页表遍历、权限检查
- **阅读要点**：把“寄存器访问控制”和“地址翻译规则”放在一起理解
- **适用场景**：系统寄存器 trap、页表属性、权限 fault

## 5. 中断控制器

对应：[11](11-GICv3中断控制器深度分析.md)、[24](24-ARM中断控制器架构GICv2与GICv3对比分析.md)

- **关键概念**：Distributor、Redistributor、CPU Interface、SGI/PPI/SPI/LPI、虚拟中断
- **阅读要点**：11 看完整路径，24 看版本差异与软件接口演进
- **适用场景**：中断送达失败、优先级异常、GICv2→v3 迁移

## 6. Cache

对应：[12](12-ARM-Cache管理与维护操作深度分析.md)

- **关键概念**：CCSIDR、CSSELR、CTR、DC/IC 指令、架构可见 cache 维护
- **阅读要点**：重点看“架构效果”，不是模拟真实硬件 cache 时序
- **适用场景**：cache 维护、自修改代码、cache/TLB 初始化路径

## 7. 定时器

对应：[13](13-ARM-Generic-Timer深度分析.md)

- **关键概念**：系统计数器、频率寄存器、物理/虚拟/安全/Hypervisor 定时器
- **阅读要点**：这是把 EL、GIC、异常路由串起来的最佳应用篇
- **适用场景**：定时器不触发、虚拟定时偏移、中断路由分析

## 8. PMU

对应：[14](14-ARM-PMUv3性能监控单元深度分析.md)

- **关键概念**：事件选择、计数器、溢出中断、EL 过滤
- **阅读要点**：PMU 不是纯性能工具，它也是架构状态机
- **适用场景**：perf 计数差异、溢出中断、特权级过滤

## 9. 调试

对应：[15](15-ARM调试架构深度分析.md)

- **关键概念**：MDSCR、DBGBCR、DBGWCR、OS Lock、gdbstub 接入
- **阅读要点**：把调试事件看成“特殊原因触发的异常”最容易理解
- **适用场景**：断点/观察点/单步、GDB 调试路径

## 10. SVE / SME

对应：[16](16-ARM-SVE-SME可伸缩向量矩阵扩展深度分析.md)

- **关键概念**：VL、寄存器可见性、陷阱控制、EL 切换影响
- **阅读要点**：理解“特性可用性 ≠ 单纯的 CPU 支持位”
- **适用场景**：向量异常、VL 变化、用户态/内核态行为差异

## 11. TrustZone

对应：[17](17-ARM-TrustZone安全扩展深度分析.md)、[22](22-ARM-TrustZone安全组件模拟深度分析.md)

- **关键概念**：Secure/Non-secure、SCR_EL3、中断路由、TZ-MPC/TZ-PPC
- **阅读要点**：17 偏 CPU 安全态，22 偏总线/外设安全组件
- **适用场景**：世界切换、安全中断、外设访问隔离

## 12. 内存模型

对应：[18](18-ARM内存模型与内存序深度分析.md)

- **关键概念**：DMB/DSB/ISB、Exclusive、LSE、MTE
- **阅读要点**：这篇讲的是顺序保证，不是页表遍历本身
- **适用场景**：并发 bug、lock-free、屏障与原子语义

## 13. 设备模型

对应：[19](19-VirtIO设备模型与传输层深度分析.md)、[20](20-PCI-PCIe子系统深度分析.md)、[21](21-ARM-SMMUv3与IOMMU框架深度分析.md)

- **关键概念**：virtqueue、PCIe 枚举/BAR、SMMUv3 DMA 翻译
- **阅读要点**：19 看 I/O 路径，20 看总线接入，21 看设备侧地址空间隔离
- **适用场景**：DMA 方向错误、设备枚举、MSI/中断与 IOMMU 联动

## 14. RME

对应：[23](23-ARM-RME-Realm管理扩展深度分析.md)

- **关键概念**：Root/Secure/Realm/Non-secure、GPT、GPC
- **阅读要点**：先懂 TrustZone 双世界，再看 RME 四世界会更顺
- **适用场景**：Realm 隔离、物理粒度保护、RME 特性实现

---

## 15. 与 arm64/ 文档的对应关系

`arm/` 更偏**横向总览**，覆盖 AArch32、通用 ARM 外设、安全组件和设备模型；`arm64/` 更偏**AArch64 纵向深挖**。实用映射如下：

| arm/ 文档 | arm64/ 对应 | 说明 |
|---|---|---|
| [08](08-ARM64-EL状态管理与指令执行深度分析.md) | [35](../arm64/35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md)、[36](../arm64/36-ARM64不同EL下指令执行流变化深度分析.md)、[54](../arm64/54-ARM64-异常级别状态管理综合导航-EL切换-指令差异-安全状态-TCG翻译变化.md) | EL 状态与执行差异的强对应组 |
| [09](09-AArch32异常处理与模式切换深度分析.md) | [35](../arm64/35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md)、[52](../arm64/52-ARM64-异常处理完整流程深度分析-同步异步异常入口-VBAR向量表-PSTATE保存恢复-异常返回.md) | 无 AArch32 对等篇，可借 AArch64 异常框架对照 |
| [10](10-CP15系统寄存器与MMU页表管理深度分析.md) | [11](../arm64/11-MMU-TLB深度分析.md)、[30](../arm64/30-ARM64-MMU系统寄存器与页表遍历深度分析.md)、[47](../arm64/47-ARM64-系统寄存器与CP访问深度分析-ARMCPRegInfo框架-MRS-MSR翻译-cpregs哈希表-EL银行与访问控制.md) | 系统寄存器/MMU 在 arm64 中拆得更细 |
| [11](11-GICv3中断控制器深度分析.md) | [26](../arm64/26-GICv3寄存器模拟与状态机深度分析.md)、[27](../arm64/27-ARM64中断虚拟化ICH-ICV-LR状态机深度分析.md)、[50](../arm64/50-ARM64-GICv3中断控制器深度分析-Distributor-Redistributor-CPUInterface-中断路由与优先级.md) | GICv3 直接强对应 |
| [12](12-ARM-Cache管理与维护操作深度分析.md) | [32](../arm64/32-ARM64特殊系统寄存器与Cache-AT指令深度分析.md)、[51](../arm64/51-ARM64-内存属性与缓存一致性深度分析-MAIR-TCR属性编码-Device-Normal类型-缓存维护指令.md) | cache 查询/维护 ↔ 属性编码/一致性 |
| [13](13-ARM-Generic-Timer深度分析.md) | [12](../arm64/12-Generic-Timer定时器深度分析.md) | 基本一一对应 |
| [14](14-ARM-PMUv3性能监控单元深度分析.md) | [13](../arm64/13-PMU性能监控单元深度分析.md) | 基本一一对应 |
| [15](15-ARM调试架构深度分析.md) | [53](../arm64/53-ARM64-调试架构深度分析-MDSCR-DBGBCR-DBGWCR-断点观察点-单步执行.md) | 架构调试强对应；若看基础设施可再补 [46](../arm64/46-ARM64-TCG插件与调试子系统深度分析-PluginAPI-GDBStub-断点单步-ARM调试寄存器与Tracing.md) |
| [16](16-ARM-SVE-SME可伸缩向量矩阵扩展深度分析.md) | [15](../arm64/15-SVE-SME可扩展向量扩展深度分析.md) | 直接对应；EL 影响可补 36 |
| [17](17-ARM-TrustZone安全扩展深度分析.md) | [37](../arm64/37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md)、[48](../arm64/48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md) | TrustZone 总览 ↔ 安全状态与世界切换 |
| [18](18-ARM内存模型与内存序深度分析.md) | [45](../arm64/45-ARM64-TCG内存模型与原子操作深度分析-屏障语义-MemOp标志-Exclusive-LSE原子与后端发射.md)、[16](../arm64/16-PAC-BTI-MTE安全特性深度分析.md) | 内存序/原子主对应是 45，MTE 可补 16 |
| [19](19-VirtIO设备模型与传输层深度分析.md) | [00](../arm64/00-ARM64-CPU-GICv3-TCG深度分析.md)、[05](../arm64/05-FDT设备树深度分析.md) | 无严格对等篇，属平台接入弱对应 |
| [20](20-PCI-PCIe子系统深度分析.md) | [01](../arm64/01-ACPI表生成与启动流程深度分析.md)、[05](../arm64/05-FDT设备树深度分析.md) | arm64 中缺少纯 PCIe 专题，可从设备描述/启动接入理解 |
| [21](21-ARM-SMMUv3与IOMMU框架深度分析.md) | [09](../arm64/09-虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU.md)、[38](../arm64/38-ARM64内存管理深度分析-页表遍历-TLB-Stage2翻译与属性合并.md) | 设备侧翻译与 CPU 侧二阶段翻译形成类比 |
| [22](22-ARM-TrustZone安全组件模拟深度分析.md) | [08](../arm64/08-TrustZone安全扩展与Secure-World深度分析.md)、[37](../arm64/37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md)、[48](../arm64/48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md) | `arm/22` 更偏外围安全组件 |
| [23](23-ARM-RME-Realm管理扩展深度分析.md) | [17](../arm64/17-GCS-RME及新扩展深度分析.md) | RME 直接对应 |
| [24](24-ARM中断控制器架构GICv2与GICv3对比分析.md) | [26](../arm64/26-GICv3寄存器模拟与状态机深度分析.md)、[50](../arm64/50-ARM64-GICv3中断控制器深度分析-Distributor-Redistributor-CPUInterface-中断路由与优先级.md) | `arm/24` 提供版本演进视角 |

---

## 16. 阅读路线推荐

### 路线 A：入门总览
`08 → 10 → 11 → 13 → 17 → 23`

先建立 CPU、异常、中断、定时器、安全域五个坐标。

### 路线 B：异常/中断排障
`08 → 09 → 11 → 13 → 14 → 15 → 24`

适合分析异常入口、中断送达、PMU 溢出、调试打断等运行时问题。

### 路线 C：内存/一致性
`10 → 12 → 18 → 16 → 21`

适合分析页表、cache 维护、屏障/原子、设备 DMA 地址翻译。

### 路线 D：安全与隔离
`17 → 22 → 23 → 11 → 21`

适合从 TrustZone 走到 RME，再扩展到中断与 DMA 隔离。

---

## 17. 交叉引用

### 17.1 中断为什么没有送达？
先看 [11](11-GICv3中断控制器深度分析.md)、[24](24-ARM中断控制器架构GICv2与GICv3对比分析.md)，再补 [13](13-ARM-Generic-Timer深度分析.md)、[14](14-ARM-PMUv3性能监控单元深度分析.md)、[17](17-ARM-TrustZone安全扩展深度分析.md)。

### 17.2 页表、权限或寄存器访问为什么不对？
先看 [10](10-CP15系统寄存器与MMU页表管理深度分析.md)，再补 [08](08-ARM64-EL状态管理与指令执行深度分析.md)、[12](12-ARM-Cache管理与维护操作深度分析.md)、[18](18-ARM内存模型与内存序深度分析.md)。

### 17.3 为什么不同 EL / 世界下行为不同？
先看 [08](08-ARM64-EL状态管理与指令执行深度分析.md)、[17](17-ARM-TrustZone安全扩展深度分析.md)，再补 [16](16-ARM-SVE-SME可伸缩向量矩阵扩展深度分析.md)、[15](15-ARM调试架构深度分析.md)、[14](14-ARM-PMUv3性能监控单元深度分析.md)。

### 17.4 DMA / IOMMU / 设备访问为什么异常？
先看 [19](19-VirtIO设备模型与传输层深度分析.md)、[20](20-PCI-PCIe子系统深度分析.md)、[21](21-ARM-SMMUv3与IOMMU框架深度分析.md)，再补 [11](11-GICv3中断控制器深度分析.md)、[22](22-ARM-TrustZone安全组件模拟深度分析.md)。

### 17.5 Realm / Secure / Non-secure 边界怎么建立？
建议顺序：[17](17-ARM-TrustZone安全扩展深度分析.md) → [22](22-ARM-TrustZone安全组件模拟深度分析.md) → [23](23-ARM-RME-Realm管理扩展深度分析.md)。

---

**一句话总结：** `arm/` 负责搭建 ARM 横向知识地图，`arm64/` 负责拆开 AArch64 关键细节；最佳阅读方式不是按编号线性读，而是围绕异常、中断、内存、安全、设备五条主线跳读。
