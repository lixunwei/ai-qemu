# GIC 规范验证补充：ARM IHI 0048B.b / IHI 0069H.b 对照勘误

## 1. 概述

本文作为现有 QEMU GIC 分析文档的**规范核对补充件**，目的不是重写既有分析，而是将既有结论与 ARM 官方 GIC 规范逐条对照，标记：

1. 哪些结论已经与规范一致，可继续保留；
2. 哪些表述存在“实现上大体正确，但规范术语或边界条件不够准确”的问题；
3. 哪些段落建议补充规范章节引用，以便后续读者区分：
   - **QEMU 当前实现行为**
   - **ARM 架构规范定义**
   - **两者之间的简化、别名或实现特定限制**

本补充文档重点覆盖以下两个规范版本：

- **GICv2**：ARM Generic Interrupt Controller Architecture Specification, **IHI 0048B.b**
- **GICv3/v4**：ARM Generic Interrupt Controller Architecture Specification, **IHI 0069H.b**

本文采用“**已验证正确 / 需修正 / 建议补引**”三层结构，尽量保持和既有文档的阅读路径一致，并将容易混淆的架构术语（如 *acknowledgment*、*priority drop*、*deactivation*、*running priority*）统一回 ARM 规范原词。

---

## 2. 验证范围

本次核对的对象为下列 4 份既有分析文档：

| 序号 | 文档 | 大小 | 主题 |
|---|---|---:|---|
| 1 | `darren/arm64/24-GICv3完整中断生命周期深度分析.md` | 23KB | GICv3 中断生成、路由、应答、EOI、抢占 |
| 2 | `darren/arm64/25-GICv3-ITS中断翻译服务与LPI深度分析.md` | 20KB | ITS、LPI、命令队列、翻译表、集合与路由 |
| 3 | `darren/arm64/26-GICv3寄存器模拟与状态机深度分析.md` | 20KB | GICD/GICR/ICC/ICH 寄存器模型与状态机 |
| 4 | `darren/arm/24-ARM中断控制器架构GICv2与GICv3对比分析.md` | 14KB | GICv2 与 GICv3 的架构与虚拟化差异 |

### 2.1 验证方法说明

本补充件遵循以下原则：

- **优先以 ARM 规范原文定义为准**；
- 若既有文档描述的是 **QEMU 源码行为**，则判断其是否：
  - 与架构规范直接一致；
  - 是规范允许的某种实现；
  - 或仅是 QEMU 当前实现特例，不能上升为架构定论；
- 若既有文档使用了简写、口语化术语、或将两个规范步骤压缩为一步，则标记为“需修正”。

### 2.2 规范引用格式

后续建议在正文中统一采用如下引用格式：

- `IHI 0069H.b §4.1.2`
- `IHI 0069H.b §12.2.19`
- `IHI 0048B.b §4.4.4`

如需同时指出寄存器名与章节，可写为：

- `ICC_PMR_EL1（IHI 0069H.b §12.2.19）`
- `GICC_IAR（IHI 0048B.b §4.4.4）`

如是对比引用，可写为：

- `GICv2 使用 GICC_IAR（IHI 0048B.b §4.4.4），GICv3 使用 ICC_IAR0_EL1/ICC_IAR1_EL1（IHI 0069H.b §12.2.13 / §12.2.14）`

---

## 3. GICv3 中断生命周期验证（对应 `arm64/24`）

该文档整体质量较高，对 QEMU 中 SPI / PPI / SGI 的生命周期、优先级判定、EOI 模式和路由过程均有较完整覆盖。主要问题不在主线流程，而在**规范术语拆分不够细**，尤其是：

- 将 *acknowledgment* 简化为“读 IAR”；
- 将 *priority drop* 与 *deactivation* 合并描述；
- 将 LPI 套入普通 interrupt state 机，造成状态边界不够严谨。

### 3.1 已验证正确项

| 标记 | 结论 | 规范核对结果 | 依据 |
|---|---|---|---|
| ✓ | Interrupt states: Inactive / Pending / Active / Active-and-pending | 与规范一致 | `IHI 0069H.b §4.1.2` |
| ✓ | SPI / SGI / PPI 可以进入 Active-and-pending；LPI 不可以 | 与规范一致 | `IHI 0069H.b §4.1.2` |
| ✓ | 通过 `ICC_IAR0_EL1` / `ICC_IAR1_EL1` 进行 acknowledgment 后，边沿中断表现为 pending→active，电平中断可表现为 pending→active-and-pending | 与规范一致 | `IHI 0069H.b §4.1_p0`, `§4.1.2` |
| ✓ | `ICC_EOIR0_EL1` / `ICC_EOIR1_EL1` 执行 priority drop；在 `EOImode=1` 时，deactivation 需经 `ICC_DIR_EL1` 单独完成 | 与规范一致 | `IHI 0069H.b §4.1`, `§4.1_p0` |
| ✓ | `ICC_EOIR0_EL1` 用于 Group 0；`ICC_EOIR1_EL1` 用于 Group 1 | 与规范一致 | `IHI 0069H.b §4.1` |
| ✓ | SPI 通过 `GICD_IROUTER<n>` 路由，SGI 通过 affinity target list 路由 | 与规范一致 | `IHI 0069H.b §2.3.1`, `§4.4`, `§4.5` |

### 3.2 需修正项

| 标记 | 现有表述 | 问题 | 建议修正 | 依据 |
|---|---|---|---|---|
| ✗ | “EOI = Active→Inactive” | 该说法只在 `EOImode=0` 时可近似成立；在架构术语中 EOI 首先是 **priority drop**，并不必然等同去激活 | 改为：“EOI 对应 priority drop；当 `EOImode=0` 时可同时完成 deactivation；当 `EOImode=1` 时需通过 `ICC_DIR_EL1` 去激活” | `IHI 0069H.b §4.1`, `§4.1_p0` |
| ✗ | 将 LPI 也纳入 Redistributor 的 active / active-pending 状态图 | LPI 在规范上没有与 SPI/PPI/SGI 完全相同的 active / active-pending 状态语义 | 单独画出 LPI 的简化状态说明，不再套用普通 interrupt state 图 | `IHI 0069H.b §4.1.2` |
| ✗ | 抢占只取决于“当前最高 pending” | 规范要求还要检查 group enable、priority mask、running priority，以及二进制点对 group priority 的影响 | 补充“最高 pending 只是候选，真正 preemption 需同时满足使能与优先级条件” | `IHI 0069H.b §4.1_p0`, `§4.1`, `§4.1.2` |

### 3.3 建议补充的规范引用

建议在 `arm64/24` 中补入以下章节引用，避免读者只见 QEMU 行为、看不到架构分层：

| 主题 | 建议引用 |
|---|---|
| 生命周期主步骤：Generate / Distribute / Deliver / Activate / Priority drop / Deactivation | `IHI 0069H.b §4.1` |
| `ICC_HPPIR0_EL1` / `ICC_HPPIR1_EL1`、`ICC_RPR_EL1`、running priority | `IHI 0069H.b §4.1_p0` |
| 状态机转移 A1/A2/B1/B2/C/D/E1/E2 | `IHI 0069H.b §4.1.2` |
| Interrupt grouping：Group 0 / Secure Group 1 / Non-secure Group 1 | `IHI 0069H.b §4.6` |
| SPI / SGI affinity routing：`GICD_IROUTER`、`ICC_SGI*R_EL1`、`IRM`、targetlist | `IHI 0069H.b §2.3.1` |
| PPI / SGI / SPI 的定义与触发类型约束 | `IHI 0069H.b §4.3`, `§4.4`, `§4.5` |

### 3.4 术语修正规范

| 建议术语 | 不建议写法 | 说明 |
|---|---|---|
| acknowledgment | “IAR 读取” | 读 IAR 是实现动作，但规范定义的是“interrupt acknowledgment” |
| priority drop | “EOI 清中断” | priority drop 与 deactivation 在规范上是两个步骤 |
| deactivation | “EOI 完成态清除” | 在 `EOImode=1` 下必须和 priority drop 分离 |
| running priority | “当前 active 最高优先级” | 需明确是“已 active 且尚未 priority drop 的最高优先级” |
| active and pending | “A+P” | 文档可口语说明，但正式结论建议回到规范原词 |
| Non-secure Group 1 / Secure Group 1 | “G1NS / G1S” | 在规范引用处建议写全称，缩写可在首次定义后再使用 |

### 3.5 建议补入的生命周期图（修正版）

```text
普通 SPI/SGI/PPI（抽象态）

Inactive
   | generate
   v
Pending -- acknowledge --> Active
   ^                        |
   | level asserted         | EOI(priority drop) + deactivate(EOImode=0)
   | or retrigger           | or DIR(EOImode=1)
   |                        v
   +------ Active and pending

说明：
- level interrupt 在被 acknowledgment 时，如输入仍保持 asserted，可进入 active and pending。
- edge interrupt 的再次触发也可导致 active and pending。
- deactivation 与 priority drop 是两个不同架构动作。
```

### 3.6 LPI 状态差异图（建议新增）

```text
LPI（不要直接套用普通 interrupt state 图）

配置表/挂起表/ITS 映射
        |
        v
    Pending for delivery
        |
        | acknowledgment + CPU interface handling
        v
    Active processing context
        |
        | EOI / completion path
        v
    Not pending

关键差异：
- 规范未要求 LPI 具备与 SPI/PPI/SGI 相同的 active and pending 状态机。
- 因此文档应避免说“LPI 在 Redistributor 中同样有 Active / Active-and-pending 位”。
```

### 3.7 对 `arm64/24` 的修订建议

1. 将“CPU 应答（ICC_IAR 读取）”一节标题改为“CPU acknowledgment（读取 `ICC_IAR0_EL1` / `ICC_IAR1_EL1`）”；
2. 将 “EOI = Active→Inactive” 改写为“EOI 首先执行 priority drop，去激活是否同步发生取决于 `EOImode`”；
3. 在抢占判定章节中补一段：
   - 最高 pending 只是候选；
   - 还要通过 `ICC_PMR_EL1`、group enable、binary point、running priority 联合过滤；
4. 在状态机图旁明确注明：“LPI 不适用此完整四态模型”。

---

## 4. GICv3 寄存器与 ITS 验证（对应 `arm64/25` + `arm64/26`）

这两份文档对 QEMU 的 ITS 翻译、LPI 投递、GICD/GICR/ICC 寄存器建模覆盖较广，整体与规范大体一致。需要修正的部分主要集中在：

- 个别 ITS 命令语义被过度简化；
- 某些系统寄存器副作用描述不完整；
- `RWP` 的适用范围未按规范收窄；
- `EOImode`、`ICC_DIR_EL1`、`ICC_PMR_EL1` 的规范行为没有完全展开。

### 4.1 已验证正确项

| 标记 | 结论 | 规范核对结果 | 依据 |
|---|---|---|---|
| ✓ | ITS translation flow：`DeviceID→DTE→ITT/ITE→ICID/pINTID→Collection table→target Redistributor` | 与规范一致 | `IHI 0069H.b §5.2` |
| ✓ | `MAPI = MAPTI with pINTID=EventID`，且有效 LPI 要求 `pINTID ≥ 0x2000` | 与规范一致 | `IHI 0069H.b §5.3` |
| ✓ | `MAPD`、`MAPC` 的作用描述与规范对齐 | 与规范一致 | `IHI 0069H.b §5.3` |
| ✓ | `ICC_BPR0_EL1` / `ICC_BPR1_EL1` 将优先级划分为 group priority + subpriority，且 `BPR1` 为 banked | 与规范一致 | `IHI 0069H.b §12.2.4`, `§12.2.5` |

### 4.2 需修正项

| 标记 | 现有表述 | 问题 | 建议修正 | 依据 |
|---|---|---|---|---|
| ✗ | `INVALL` = “所有 Redistributors 的 LPI refresh” | 过宽。规范语义是：对指定 `ICID` 的 collection 相关缓存进行失效，影响范围是该 collection 在各 Redistributor 上的缓存视图，而不是泛化成“全量刷新” | 改写为“`INVALL ICID` invalidates caching for a collection across Redistributors” | `IHI 0069H.b §5.3` |
| ✗ | `ICC_IAR0_EL1` / `ICC_IAR1_EL1` 仅被说成“中断应答寄存器” | 少了寄存器宽度、`INTID[23:0]`、special INTIDs 等规范信息 | 增补寄存器格式、特殊返回值、分组适用范围 | `IHI 0069H.b §12.2.13`, `§12.2.14` |
| ✗ | `ICC_EOIR0_EL1` / `ICC_EOIR1_EL1` 只描述为 EOI 写回 | 缺少 `EOImode` 控制的两种语义，也未补出 `ICC_DIR_EL1` | 增补“priority-drop-only” vs “priority-drop+deactivate”，并补引 `ICC_DIR_EL1` | `IHI 0069H.b §12.2.9`, `§12.2.10` |
| ✗ | `ICC_PMR_EL1` 只写到 NS remap 行为 | 漏掉“writes are self-synchronizing”这一关键行为 | 补充自同步写语义，说明软件无需额外同步即可影响中断屏蔽效果 | `IHI 0069H.b §12.2.19` |
| ✗ | `GICD_CTLR.RWP` 被描述为泛化的“所有写入完成标志” | 规范仅对特定寄存器写入集合生效 | 将 `RWP` 适用范围限定到 group enable、ARE bits、`E1NWF`、`DS`、`ICENABLER` 写入 | `IHI 0069H.b §12.9.4` |

### 4.3 ITS 命令集核对结果

以下命令集合与 `IHI 0069H.b §5.3` 的主旨一致，既有文档总体可保留：

| 命令 | 核对结果 | 备注 |
|---|---|---|
| `MAPD` | ✓ | 建立 / 使能 Device table entry |
| `MAPC` | ✓ | 建立 / 使能 Collection 与目标 Redistributor 关系 |
| `MAPTI` | ✓ | 将 DeviceID + EventID 映射到指定 `pINTID` + `ICID` |
| `MAPI` | ✓ | `MAPTI` 的简化形式，`pINTID = EventID` |
| `CLEAR` | ✓ | 清除 pending 状态，但不删除映射 |
| `DISCARD` | ✓ | 删除映射，后续相关事件静默丢弃 |
| `INT` | ✓ | 触发一个映射后的事件投递 |
| `INV` | ✓ | 失效指定条目的缓存视图 |
| `INVALL` | ✓/✗ | 命令名本身核对正确，但语义解释需收窄 |
| `MOVI` | ✓ | 改变中断映射目标 collection |
| `MOVALL` | ✓ | 迁移一个 Redistributor 相关 collection 状态 |
| `SYNC` | ✓ | 排序点 / 同步语义，QEMU 同步执行可近似为 NOP，但文档应注明其架构含义 |

### 4.4 建议补充的规范引用

#### 4.4.1 Distributor / Redistributor / CPU interface

| 主题 | 建议引用 |
|---|---|
| `GICD_CTLR`：`RWP`、`DS`、`ARE_S`、`ARE_NS`、group-enable | `IHI 0069H.b §12.9.4` |
| `GICD_TYPER`：`ITLinesNumber`、`LPIS`、`A3V`、`DVIS`、`NMI`、`IDbits` | `IHI 0069H.b §12.9.38` |
| `GICD_IPRIORITYR<n>`：0-7 banked、NS/S 可见性 | `IHI 0069H.b §12.9.20` |
| `GICD_ICFGR<n>`：2-bit trigger config | `IHI 0069H.b §12.9.9` |
| `GICD_IROUTER<n>`：affinity 路由字段与 `IRM` | `IHI 0069H.b §12.9.22` |
| `GICR_CTLR` | `IHI 0069H.b §12.11.2` |
| `GICR_WAKER`：`ChildrenAsleep`、wake 行为 | `IHI 0069H.b §12.11.42` |
| `ICC_PMR_EL1`：priority filter + self-synchronizing writes | `IHI 0069H.b §12.2.19` |
| acknowledgment / EOI / binary point / SRE | `IHI 0069H.b §12.2.13`, `§12.2.14`, `§12.2.9`, `§12.2.10`, `§12.2.4`, `§12.2.5`, `§12.2.23` |

#### 4.4.2 ITS / LPI

| 主题 | 建议引用 |
|---|---|
| ITS translation overview | `IHI 0069H.b §5.2` |
| `SYNC` 的 ordering semantics | `IHI 0069H.b §5.2.9` |
| `CLEAR` 清 pending、`DISCARD` 删除映射并静默丢弃未来请求 | `IHI 0069H.b §5.3` |
| `MAPD` / `MAPC` / `MAPTI` / `MAPI` / `INT` / `INV` / `INVALL` / `MOVI` / `MOVALL` | `IHI 0069H.b §5.3` |

### 4.5 对 `arm64/25` 的专项建议

#### 4.5.1 修正 `INVALL` 的解释

建议将类似“全 Redistributor LPI 刷新”的表述，改成下面这种更贴近规范的话：

> `INVALL` 以 `ICID` 为粒度，使与该 collection 相关的缓存状态失效，使后续投递重新依据内存中的 collection / pending / config 信息解析。其效果跨 Redistributors 可见，但并不等同于“系统所有 LPI 全量刷新”。

#### 4.5.2 强化 `SYNC` 的架构语义

QEMU 因为同步执行 ITS 命令，容易把 `SYNC` 写成“等价 NOP”。这在**实现行为**上可以理解，但建议改为：

- `SYNC` 的**架构角色**是顺序保证点；
- 在 QEMU TCG 的同步实现里，它**可以退化为无需额外动作**；
- 但文档不应直接删除其规范上的 ordering 语义。

### 4.6 对 `arm64/26` 的专项建议

#### 4.6.1 `ICC_IAR0_EL1` / `ICC_IAR1_EL1`

建议在寄存器章节明确写出：

- 这是 **64-bit** system registers；
- 主要返回域包含 `INTID[23:0]`；
- 存在 special INTIDs（例如无可应答中断时的特殊返回值）；
- `ICC_IAR0_EL1` 与 `ICC_IAR1_EL1` 分别对应不同 interrupt group 视图。

#### 4.6.2 `ICC_EOIR0_EL1` / `ICC_EOIR1_EL1` + `ICC_DIR_EL1`

建议增加一个小表：

| 模式 | `ICC_EOIR*` 作用 | 是否还需 `ICC_DIR_EL1` |
|---|---|---|
| `EOImode=0` | priority drop + deactivate | 否 |
| `EOImode=1` | 仅 priority drop | 是 |

#### 4.6.3 `ICC_PMR_EL1`

应补一句规范级描述：

> 对 `ICC_PMR_EL1` 的写入是 self-synchronizing，因此软件可依赖其对中断可见性的即时影响，而不需要额外同步序列去确保优先级屏蔽立即生效。

#### 4.6.4 `GICD_CTLR.RWP`

建议不要写成“任意写后都可观察 `RWP`”。更准确的写法应是：

> `RWP` 只跟踪规范定义的一组写操作完成状态，主要涉及 group enable、ARE、`E1NWF`、`DS` 以及 `ICENABLER` 等影响分发状态的写入。

---

## 5. GICv2 对比验证（对应 `arm/24`）

这份对比文档的总体方向正确，尤其是：

- 能把 GICv2 的 MMIO CPU interface 与 GICv3 的 system register interface 区分开；
- 能正确指出 GICv2 虚拟化主要依赖 `GICH` / `GICV` MMIO；
- 能较好解释 `GICD_ITARGETSR` 的 CPU 目标位图模型。

但其中有三处很容易被读者误读成“架构定论”，需要修正。

### 5.1 已验证正确项

| 标记 | 结论 | 规范核对结果 | 依据 |
|---|---|---|---|
| ✓ | `GICD_ITARGETSR` 为每中断一个 8-bit CPU targets 字段 | 与规范一致 | `IHI 0048B.b §4.3.12_p0`, `§2.2.1` |
| ✓ | `GICC_IAR` 对 SGI 返回值中包含 `CPUID`，可识别请求源 CPU interface | 与规范一致 | `IHI 0048B.b §4.4.4` |
| ✓ | GICv2 虚拟化使用 `GICH` / `GICV` MMIO 寄存器模型 | 与规范一致 | `IHI 0048B.b §5.3.1`, `§5.3.8`, `§5.5.4`, `§5.5.5` |

### 5.2 需修正项

| 标记 | 现有表述 | 问题 | 建议修正 | 依据 |
|---|---|---|---|---|
| ✗ | “GICv2 最大 CPU = 8” | 对 QEMU 源码实现可成立，但不能直接当作架构上限；架构上由 `GICD_TYPER` 报告实现的 CPU interfaces | 改为“QEMU 此实现使用 8 CPU 目标位图；架构上实现数量由 `GICD_TYPER` 给出” | `IHI 0048B.b §4.3.2` |
| ✗ | “GICv2 没有 SGI source tracking” | 错误。`GICC_IAR[12:10]` 的 `CPUID` 就是 SGI 请求源接口标识 | 改为“GICv2 可在 `GICC_IAR` 中报告 SGI 请求源 CPU interface” | `IHI 0048B.b §4.4.4` |
| ✗ | “`GICD_TYPER CPUNumber=0 (ARE=1)`” | `ARE` 是 GICv3 概念，不应回填到 GICv2 说明中 | 删除 `ARE=1` 关联，只保留 `CPUNumber` 的 GICv2 语义说明 | `IHI 0048B.b §4.3.2` |

### 5.3 建议补充的规范引用

| 主题 | 建议引用 |
|---|---|
| Interrupt IDs：`ID0-ID1019`、SPIs 与 private interrupts | `IHI 0048B.b §2.2.1` |
| Priority drop 与 interrupt deactivation | `IHI 0048B.b §3.2.1` |
| Preemption | `IHI 0048B.b §3.3.1` |
| `GICD_TYPER` | `IHI 0048B.b §4.3.2` |
| `GICD_ITARGETSRn` | `IHI 0048B.b §4.3.12_p0` |
| `GICC_IAR`（SGI 的 `CPUID` 字段） | `IHI 0048B.b §4.4.4` |
| List registers 与 virtual interrupt handling | `IHI 0048B.b §5.2.1` |
| Maintenance interrupts | `IHI 0048B.b §5.2.5` |
| `GICH_HCR` | `IHI 0048B.b §5.3.1` |
| `GICH_LRn` | `IHI 0048B.b §5.3.8` |
| `GICV_IAR` | `IHI 0048B.b §5.5.4` |
| `GICV_EOIR` | `IHI 0048B.b §5.5.5` |

### 5.4 GICv2 / GICv3 虚拟化结论复核

以下高层结论可保留：

| 结论 | 结果 | 说明 |
|---|---|---|
| GICv2 虚拟化主要使用 `GICH` / `GICV` MMIO | ✓ | 与 IHI 0048B.b 一致 |
| GICv3 虚拟化主要使用 `ICH_*` / `ICV_*` system registers | ✓ | 与 IHI 0069H.b 的架构方向一致 |
| “GICv2 与 GICv3 的虚拟化接口模型不同” | ✓ | 可作为对比文档核心结论保留 |

---

## 6. 汇总勘误表

### 6.1 按文档汇总

| 文档 | 已验证正确 | 需修正 | 建议补引重点 |
|---|---:|---:|---|
| `arm64/24` | 6 | 3 | 生命周期主步骤、状态机、grouping、routing、running priority |
| `arm64/25` | 4 | 1~2 个 ITS 语义重点 | `INVALL`、`SYNC`、`CLEAR`、`DISCARD`、ITS command set |
| `arm64/26` | 3~4 个寄存器主结论可保留 | 4 | `ICC_IAR*`、`ICC_EOIR*`、`ICC_DIR_EL1`、`ICC_PMR_EL1`、`GICD_CTLR.RWP` |
| `arm/24` | 3 | 3 | `GICD_TYPER`、`GICC_IAR`、priority drop / deactivation、virtualization |

### 6.2 按问题类型汇总

| 类型 | 典型问题 | 处理建议 |
|---|---|---|
| 术语压缩 | 把 acknowledgment 说成“读 IAR” | 补回规范动作名称，代码动作放在括号中 |
| 生命周期合并 | 把 EOI 直接等同于 deactivation | 按 `priority drop` / `deactivation` 两步拆开 |
| 状态机泛化 | 把 LPI 套入普通 interrupt 四态 | 单列 LPI 特性，不共用完整四态图 |
| 实现特例上升为架构 | 把 QEMU 的 8 CPU 限制写成 GICv2 上限 | 明确区分“QEMU 实现”与“规范定义” |
| 寄存器副作用缺失 | `ICC_PMR_EL1` 未写 self-synchronizing | 在寄存器小节补一行“规范副作用” |
| 控制位适用范围过宽 | `RWP` 被说成任意写后都更新 | 收窄到规范列出的寄存器写入集合 |

---

## 7. 建议的更新方式

为了尽量少打断既有文档结构，建议采用“**最小侵入式修订**”：

### 7.1 对 `arm64/24`

- 在中断生命周期总图后新增一个“**规范术语对照**”小节；
- 把“EOI”章节拆成：
  1. `acknowledgment`
  2. `priority drop`
  3. `deactivation`
- 在抢占判定中增加一句：
  - “最高 pending 仅为候选，是否可 delivered / preempt 还取决于 group enable、`ICC_PMR_EL1`、binary point 和 running priority。”

### 7.2 对 `arm64/25`

- 在 ITS 命令表末尾增加“**规范边界说明**”列；
- 对 `INVALL`、`SYNC`、`DISCARD` 三条命令加粗提示；
- 在 LPI 路径图下明确说明：
  - “LPI 的规范状态模型不同于 SPI/PPI/SGI 的 active-and-pending 图示。”

### 7.3 对 `arm64/26`

- 在 `ICC_IAR0_EL1` / `ICC_IAR1_EL1` 小节加入寄存器位域摘要；
- 在 `ICC_EOIR*` 小节补一个 `EOImode` 表格；
- 在 `ICC_PMR_EL1` 小节补入 self-synchronizing 写语义；
- 在 `GICD_CTLR` 小节把 `RWP` 适用范围改成规范闭集表述。

### 7.4 对 `arm/24`

- 把“最大 CPU 数”一行改名为“QEMU 当前实现 CPU 目标位图宽度”；
- 把“SGI 源追踪：无”改成“GICv2 可在 `GICC_IAR` 返回 `CPUID`，GICv3 普通 `ICC_IAR*` 不返回等价字段”；
- 删除将 `ARE` 套用到 GICv2 的措辞。

---

## 8. 推荐插入到原文中的统一说明模板

为减少后续重复解释，建议在各文档前言或附录中加入类似模板：

> **规范一致性说明**：本文以 QEMU 当前实现为主线进行源码分析。凡涉及架构语义时，以 ARM GIC 规范为准。若正文中出现“EOI 清中断”“读 IAR 应答”等简写，应分别理解为规范中的 `priority drop` / `deactivation` 与 `acknowledgment`。对实现特定限制（例如 QEMU 的数据结构上限）不应直接上升为架构结论。

也可加入以下简表：

| 口语写法 | 规范写法 |
|---|---|
| 读 IAR | acknowledgment |
| EOI 清 active | priority drop，必要时另加 deactivation |
| 当前正在运行的最高优先级 | running priority |
| A+P | active and pending |

---

## 9. 最终结论

综合核对结果如下：

1. **4 份文档的主线分析整体可靠**，特别是对 QEMU 代码路径、寄存器分层、ITS 表结构、虚拟化接口差异的描述，绝大多数没有偏离 ARM GIC 规范；
2. 主要问题集中在**术语规范化与边界条件补充**，而不是方向性错误；
3. 其中最需要尽快修正的点有：
   - 将 `EOI` 与 `deactivation` 解耦；
   - 明确 **LPI 不适用普通 active / active-and-pending 四态图**；
   - 将抢占条件从“最高 pending”修正为“最高 pending + enable + priority + running priority 联合判定”；
   - 将 `INVALL`、`ICC_PMR_EL1`、`GICD_CTLR.RWP` 的语义改到规范精度；
   - 将 GICv2 的 8 CPU、SGI source tracking、`ARE` 相关措辞从“架构断言”修正为“规范定义或 QEMU 实现特性”。

### 9.1 建议优先级

| 优先级 | 建议处理项 |
|---|---|
| 高 | `arm64/24` 中 EOI / deactivation / LPI 状态机 / preemption 条件 |
| 高 | `arm64/26` 中 `ICC_IAR*` / `ICC_EOIR*` / `ICC_DIR_EL1` / `ICC_PMR_EL1` / `RWP` |
| 中 | `arm64/25` 中 `INVALL` / `SYNC` 的规范语义补强 |
| 中 | `arm/24` 中 GICv2 的 CPU 数、SGI source tracking、`GICD_TYPER` 表述修正 |
| 低 | 各文档术语统一为 acknowledgment / priority drop / deactivation / running priority |

### 9.2 推荐结论性表述

可在总述类文档中使用如下结论：

> 现有 QEMU GIC 分析文档在实现路径层面基本可信，与 ARM IHI 0048B.b / IHI 0069H.b 的主干定义总体一致；需修订之处主要是规范术语、寄存器副作用和少数边界条件的准确性，而非整体分析框架错误。

---

## 10. 参考规范清单

- ARM Generic Interrupt Controller Architecture Specification, **IHI 0048B.b**
- ARM Generic Interrupt Controller Architecture Specification, **IHI 0069H.b**

> 后续如在原文中增加引文，建议统一保持到“小节粒度”，例如 `IHI 0069H.b §4.1.2`，避免只写规范名不写章节，导致读者无法快速回查。
