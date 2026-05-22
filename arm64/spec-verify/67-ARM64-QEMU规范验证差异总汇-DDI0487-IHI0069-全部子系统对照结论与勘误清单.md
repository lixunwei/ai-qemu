# ARM64 QEMU 规范验证差异总汇

> QEMU 11.0.50 vs ARM DDI 0487 M.b / IHI 0048B / IHI 0069H — 全子系统对照结论

## 1. 概述
- 覆盖子系统：GIC、EL 状态/异常、MMU、Generic Timer、ISA、Debug、TrustZone/RME、SVE/SME、PAC/BTI/MTE、内存模型/屏障、Cache/TLB。
- 本总汇仅收录差异项，共 **58** 项：**⚠️ 50 / ❌ 8**；未列出的主线路径在源文档中普遍被评为 ✅ 或“基本一致”。
- 严重度分布：**P0=2、P1=6、P2=45、P3=5**。
- 核心结论：QEMU 在 ARM64 上的“功能路径”覆盖面已经很高，足以支撑大多数 guest OS、内核和用户态程序运行。真正的高风险缺口集中在 **PAuth_LR/PACM、外部调试、exclusive monitor 细节、LDAPR 门控** 等少数点。更常见的问题不是“功能缺失”，而是 **过度有序、过度同步、无真实 cache hierarchy、无真实 timing/randomness**。因此，QEMU 很适合做功能验证和大多数源码分析基线，但并不等价于硬件一致性验证平台。

## 2. 差异严重度分级
- **P0（阻断）**：架构特性缺失，导致新版 guest/工具链/测试根本不可运行或不可暴露。
- **P1（功能影响）**：客体可见功能、异常、调试或原子语义与规范不符，可能导致结果错误。
- **P2（行为偏差）**：功能通常正确，但模型比规范更强/更粗/更简化，容易掩盖 bug 或误导结论。
- **P3（时序/精度）**：主要影响精确时序、概率分布、竞争窗口、广播延迟、持久化窗口等。

## 3. ❌ 缺失/错误 (P0-P1) 汇总表
| 编号 | 子系统 | 差异描述 | 规范章节 | QEMU 源码 | 严重度 | Guest 影响 |
|---|---|---|---|---|---|---|
| EL-01 | EL 状态管理 | 非法异常返回时，QEMU 仅恢复 NZCV/DAIF/ALLINT，未完整按 SPSR 恢复 PAN/SS/UAO/DIT/TCO/SSBS/BTYPE/PACM 等字段。 | DDI 0487 §D1.4.4.2 | target/arm/tcg/helper-a64.c, target/arm/helper.c | P1 / ❌ | 依赖非法返回精确语义的内核/异常测试会与真机不一致 |
| DBG-01 | Debug | Chapter H 外部调试仅有占位或极小子集；未真正建模 Debug state、EDSCR.HDE、ITR、DCC 执行环境。 | DDI 0487 Chapter H | target/arm/helper.c, gdbstub/* | P1 / ❌ | 无法把 QEMU 当作外部调试架构验证平台 |
| DBG-02 | Debug | 断点类型以地址匹配为主；context/VMID/full-context/address-mismatch 等大量匹配类型未实现或仅 LOG_UNIMP。 | DDI 0487 §D2.10 | target/arm/helper.c | P1 / ❌ | 复杂调试用例、虚拟化调试和上下文相关断点无法验证 |
| DBG-03 | Debug | OSLSR_EL1 的 OSLK 规范位为 bit[1]，但 `arm_generate_debug_exceptions()` 仍检查 bit[0]。 | DDI 0487 §D2 / OSLSR_EL1 | target/arm/helper.c | P1 / ❌ | OS Lock 相关调试异常门控可能错误 |
| PAC-01 | PAC/BTI | FEAT_PAuth_LR / PACM 未实现，`PACIASPPC/AUTIASPPC/RETAASPPC/...` 等 Armv9.4 指令与控制位缺失。 | A2.2 FEAT_PAuth_LR；C6.2.307 | target/arm/helper.c, target/arm/tcg/translate-a64.c, target/arm/cpu.h | P0 / ❌ | 使用新 PAC 指令序列的 guest 或工具链输出无法运行/暴露 |
| BTI-01 | PAC/BTI | 缺少 `PACIASPPC/PACIBSPPC` 隐式 BTI landing pad 支持，根因仍是 PAuth_LR 缺失。 | DDI 0487 BTI/PAuth_LR 相关条目 | target/arm/tcg/translate-a64.c | P0 / ❌ | 新版编译器生成的 landing pad/序言尾声不兼容 |
| MEM-01 | 内存模型/屏障 | `LDAPR` 特性门控额外要求 LSE；规范只要求 FEAT_LRCPC。 | DDI 0487 §B2 / LDAPR | target/arm/tcg/translate-a64.c | P1 / ❌ | 本可在硬件上可用的 RCpc 加载在 QEMU 上被错误禁用 |
| MEM-02 | 内存模型/屏障 | exclusive monitor 未建模 reservation granule，且未实现“监控范围内较小 store 清除保留”规则。 | DDI 0487 §B2.9-§B2.12 | target/arm/tcg/translate-a64.c | P1 / ❌ | lock-free/monitor 相关验证可能错误，出现虚假成功或遗漏失败 |

## 4. ⚠️ 简化/偏差 (P2) 汇总表
| 编号 | 子系统 | 差异描述 | 规范章节 | QEMU 源码 | 严重度 | Guest 影响 |
|---|---|---|---|---|---|---|
| GIC-01 | GIC | EOI 不能被简化为“Active→Inactive”；规范需区分 acknowledgment、priority drop、deactivation，`EOImode=1` 还需 `ICC_DIR_EL1`。 | IHI 0069H.b §4.1, §4.1_p0 | hw/intc/arm_gicv3*.c | P2 / ⚠️ | 若直接把 QEMU/文档摘要当架构结论，会误判中断生命周期 |
| GIC-02 | GIC | LPI 不应直接套用 SPI/PPI/SGI 的 active/active-pending 四态图。 | IHI 0069H.b §4.1.2 | hw/intc/arm_gicv3_its.c, hw/intc/arm_gicv3_redist.c | P2 / ⚠️ | ITS/LPI 行为分析容易被普通中断状态机误导 |
| GIC-03 | GIC | 抢占条件不只是“当前最高 pending”；还需 group enable、PMR、running priority、binary point 等共同决定。 | IHI 0069H.b §4.1, §4.1.2 | hw/intc/arm_gicv3*.c | P2 / ⚠️ | 优先级/抢占测试若只看最高 pending 会得出错误结论 |
| GIC-04 | GIC | ITS `INVALL` 语义应收敛为“对指定 collection 的缓存视图失效”，而不是泛化成“所有 Redistributor 的 LPI 全量刷新”。 | IHI 0069H.b §5.3 | hw/intc/arm_gicv3_its.c | P2 / ⚠️ | 对 ITS 命令影响范围的认知会偏大 |
| GIC-05 | GIC | `ICC_IAR* / ICC_EOIR* / ICC_PMR_EL1 / GICD_CTLR.RWP` 的寄存器副作用与适用范围比既有文档描述更窄更细。 | IHI 0069H.b §12.2.13, §12.2.19, §12.9.4 | hw/intc/arm_gicv3*.c | P2 / ⚠️ | 中断确认、屏蔽与寄存器同步语义易被过度泛化 |
| GIC-06 | GIC | GICv2“最大 CPU=8”“没有 SGI source tracking”“ARE 可套到 v2”都只是旧文档误写，不是架构事实。 | IHI 0048B.b §4.3.2, §4.4.4 | hw/intc/arm_gic.c | P2 / ⚠️ | 会把 QEMU 限制误当作 GICv2 架构上限 |
| EL-02 | EL 状态管理 | 异常入口后的 PSTATE 更新常被过度简化为“SS=0/NZCV 清零”；规范还要求 `UAO=0`，并按特性处理 `EXLOCK/PACM/UINJ`。 | DDI 0487 §D1.4.2, §D1.5.1 | target/arm/helper.c | P2 / ⚠️ | 异常入口/PSTATE 对照若直接照抄旧文会误标规范位 |
| MMU-01 | MMU | translation regime 不只是 QEMU 抽象出的 4 类；规范还区分 Non-secure/Secure/Realm/Root。 | DDI 0487 §D8.1.2 | target/arm/helper.c, target/arm/ptw.c | P2 / ⚠️ | 安全态/Realm 相关页表讨论若只按 EL 分类会失真 |
| MMU-02 | MMU | EL2 使能后，EL1&0 Stage-1 页表基址语义是 IPA，而不再是直观意义上的 PA。 | DDI 0487 §D8.2.3 | target/arm/ptw.c | P2 / ⚠️ | 二阶段翻译、页表页访问和虚拟化内存分析易漏关键前提 |
| MMU-03 | MMU | 4KB granule 在 DS=1 时可出现 level -1；Stage-2 还可能使用 concatenated tables，不能固定写成 L0~L3。 | DDI 0487 §D8.2.8-§D8.2.10 | target/arm/ptw.c | P2 / ⚠️ | 页表级数/页表格式测试若写死层级会漏边界 |
| MMU-04 | MMU | `descriptor[9:8]` 并非永远表示 SH；DS=1 时它们被重用为 `OA[51:50]`。 | DDI 0487 §D8.6.2, §D8.6.7 | target/arm/ptw.c | P2 / ⚠️ | 52-bit 输出地址和 shareability 字段图示会被画错 |
| MMU-05 | MMU | FWB 模式下 Stage-2 `MemAttr` 解释不能简化成“`MemAttr[3:2]` 直接映射 NC/WT/WB”。 | DDI 0487 §D8.6.6 | target/arm/ptw.c | P2 / ⚠️ | 内存属性合并、S1/S2 交互和 Device subtype 结论会失真 |
| MMU-06 | MMU | `ARMFaultType` 是 QEMU 内部超集，不等于架构定义的 8 类 MMU fault；“合并页大小=MAX(S1,S2)”也不是无条件成立。 | DDI 0487 §D8.15.1 | target/arm/internals.h, target/arm/ptw.c | P2 / ⚠️ | fault 分类与页大小归并若照搬内部枚举，会把实现细节当规范 |
| TIM-01 | Timer | `CNTFRQ_EL0` 复位后规范要求 UNKNOWN；QEMU 直接给确定值，且寄存器 write 不改变实际计数频率。 | DDI 0487 §D12.1.2 | target/arm/helper.c, target/arm/cpu.c | P2 / ⚠️ | 频率初始化、只读/读写语义与真机固件流程不完全等价 |
| TIM-02 | Timer | QEMU 一律提供 64-bit system counter；不模拟早期架构仅“至少 56 位”的窄宽度变体。 | DDI 0487 §D12.1.2 | target/arm/helper.c | P2 / ⚠️ | 早期最小实现仿真不精确，但多数 guest 受益于更强模型 |
| TIM-03 | Timer | QEMU 主要实现 PE 视角 sysreg timer；未单独建模完整 system-level MMIO counter/timer 组件。 | DDI 0487 §D12.1.1 | target/arm/helper.c, hw/arm/virt.c | P2 / ⚠️ | 系统级定时器组件/SoC 级验证边界有限 |
| TIM-04 | Timer | `CNTPCT/CNTVCT` 的 offset 公式依赖当前 EL、`E2H/TGE` 与 host/guest 视角，不能写成无条件 `PhysCount-offset`。 | DDI 0487 §D12.2.4 | target/arm/helper.c | P2 / ⚠️ | VHE/host EL2 场景下读计数器值若按通用公式理解会出错 |
| TIM-05 | Timer | Secure/SEL2 timer 访问控制不是“只看 `SCR_EL3.ST` / EL3 总可访问”这类单条件；还受 `EEL2`、当前 EL 等影响。 | DDI 0487 §D12.2 | target/arm/helper.c | P2 / ⚠️ | SEL2/安全态定时器门控测试容易误判 |
| TIM-06 | Timer | PPI 30/27/26/29/28/20/19 是 `virt`/BSA 板级映射，不是 Generic Timer 规范直接规定的固定架构中断号。 | DDI 0487 §D12 与板级实现边界 | hw/arm/virt.c, include/hw/arm/bsa.h | P2 / ⚠️ | 把板级连线误当作架构常量会影响移植性判断 |
| ISA-01 | ISA | `ISB` 在 QEMU 中主要靠 TB 边界完成同步，不建模真实硬件 pipeline flush/fetch 微行为。 | DDI 0487 §C6 / ISB 语义 | target/arm/tcg/translate-a64.c | P2 / ⚠️ | 功能通常正确，但不能据此验证微架构同步细节 |
| ISA-02 | ISA | `SVC` 的最终目标 EL 由异常入口逻辑决定，而不是 decode 阶段一张固定路由表即可概括。 | DDI 0487 §C3.1.5, §D1.4 | target/arm/tcg/translate-a64.c, target/arm/helper.c | P2 / ⚠️ | 异常生成/路由图若过度静态化会失去条件分支 |
| ISA-03 | ISA | “`IC IVAU` 触发 TB 失效”只在 user-only 成立；system-mode 下该路径是 NOP。 | DDI 0487 `IC IVAU` / QEMU mode distinction | target/arm/helper.c | P2 / ⚠️ | 把 user-only 现象推广到 system-mode 会高估 cache 维护效果 |
| DBG-04 | Debug | gdbstub 单步是模拟器控制循环，不等于架构 Software Step；两者可共存但语义层次不同。 | DDI 0487 §D2.11 | gdbstub/*, target/arm/helper.c | P2 / ⚠️ | 单步相关文档若混写，会把 host-side 调试误认成 guest-visible debug |
| TZ-01 | TrustZone/RME | Root 仅属于 EL3；`NS=1,NSE=1` 才是 Realm，`NS=0,NSE=1` 不是 Root。 | DDI 0487 §D1.1.2 | target/arm/helper.c | P2 / ⚠️ | RME 四域模型若按旧二域思维扩展，会画错状态图 |
| TZ-02 | TrustZone/RME | 世界切换的完成点是 EL3 修改 `SCR_EL3` 后执行 `ERET`，不是“异常一进 EL3 就已经切换”。 | DDI 0487 §D1.1.2, §D1.4.4 | target/arm/helper.c, target/arm/tcg/helper-a64.c | P2 / ⚠️ | SMC/Monitor 路径若把入口当完成点，会误解返回后低 EL 安全态 |
| TZ-03 | TrustZone/RME | Secure EL2 需要 `FEAT_SEL2 + SCR_EL3.EEL2(bit18)`；不能只凭 Secure state 判断，更不能把 EEL2 位号写错。 | DDI 0487 §D1.1.2 | target/arm/helper.c | P2 / ⚠️ | SEL2 可用性与世界/EL 布局分析容易出错 |
| TZ-04 | TrustZone/RME | QEMU 已理解 Secure/Non-secure/Realm/Root 四域 CPU 语义，但板级 address space 仍主要停留在 NS/S 两套。 | DDI 0487 RME/GPT 相关章节 | target/arm/helper.c, target/arm/ptw.c, hw/arm/virt.c | P2 / ⚠️ | 可验证 CPU/页表语义，不能等价推断完整 SoC 四域互连 |
| SVE-01 | SVE/SME | Streaming SVE mode 下，SVE 指令 trap 归属 SME trap 域（`SMEN/TSM/ESM`），不再由 `ZEN/TZ/EZ` 控制。 | DDI 0487 §D22.2 | target/arm/tcg/hflags.c, target/arm/tcg/translate-a64.c | P2 / ⚠️ | 流模式 trap 归属若判断错，会误配系统寄存器控制 |
| SVE-02 | SVE/SME | SME trap gate 与 `PSTATE.SM/PSTATE.ZA` 的状态有效性相互独立，不能把“可执行”简化成“SM/ZA 打开”。 | DDI 0487 §D22.3 | target/arm/tcg/hflags.c, target/arm/tcg/translate-sme.c | P2 / ⚠️ | SME 指令 enable/disable 与 trap 行为分析会被混淆 |
| SVE-03 | SVE/SME | ZT0 trap 路径与 SME/SME2 支持范围需要更窄的表述；当前能验证的是文中覆盖的主路径，而非“完整 SME2 一切语义”。 | DDI 0487 C8/C9 + D22 | target/arm/tcg/hflags.c, target/arm/tcg/translate-sme.c | P2 / ⚠️ | 把局部实现说成全集，会高估 QEMU 对新指令/新状态的覆盖 |
| PAC-02 | PAC/BTI | `ERETAA/ERETAB` 与 `HCR_NV` 的 trap precedence 仍留有 FIXME；嵌套虚拟化下优先级可能与规范不完全一致。 | NV + PAC trap precedence | target/arm/tcg/pauth_helper.c | P2 / ⚠️ | 主要影响 NV 场景的精确异常顺序验证 |
| MEM-03 | 内存模型/屏障 | QEMU 不主动重建 ARM relaxed memory model 的弱序传播/投机窗口，整体呈现“过度有序”。 | DDI 0487 §B2.2 | tcg backend, target/arm/tcg/* | P2 / ⚠️ | 很多硬件允许的弱序结果在 QEMU 上消失 |
| MEM-04 | 内存模型/屏障 | DMB 的 shareability domain（NSH/ISH/OSH/SY）与 memory type 语义被明显压扁。 | DDI 0487 §B2.6.2 | target/arm/ptw.c, target/arm/tcg/translate-a64.c | P2 / ⚠️ | 域传播和设备/普通内存差异无法精确验证 |
| MEM-05 | 内存模型/屏障 | DSB 未被单独实现得强于 DMB；completion 与 maintenance scope 主要靠 helper 同步副作用近似。 | DDI 0487 §B2.6.9 | target/arm/tcg/translate-a64.c | P2 / ⚠️ | TLBI/IC/DSB 完成点与真机存在差距 |
| MEM-06 | 内存模型/屏障 | `LDAR/STLR/LDAPR` 及 LSE A/L/AL 变体多数被实现成更强的 full fence/full LDAQ。 | DDI 0487 §B2.3, §B2.9 | target/arm/tcg/translate-a64.c, tcg backend | P2 / ⚠️ | Acquire/Release/RCpc 与 RCsc 的差异被抹平 |
| MEM-07 | 内存模型/屏障 | `STLR_i + SCTLR.nAA` 细节仍有 TODO，尚非完整状态机。 | DDI 0487 `SCTLR.nAA` 相关条目 | target/arm/tcg/translate-a64.c | P2 / ⚠️ | 对齐/非对齐相关边界测试可能不完整 |
| MEM-08 | 内存模型/屏障 | exclusive success 条件、ABA 干扰与伪失败频率采用“地址+旧值+cmpxchg”近似。 | DDI 0487 §B2.9-§B2.12 | target/arm/tcg/translate-a64.c | P2 / ⚠️ | 重试次数、公平性和 monitor 干扰模式与真机不同 |
| MEM-09 | 内存模型/屏障 | host 不支持大原子时的 fallback 与 RR 模式会把 guest 并发进一步串行化。 | DDI 0487 原子/并发模型 | tcg backend, accel/tcg/* | P2 / ⚠️ | 功能保守正确，但并发窗口明显偏小 |
| CTLB-01 | Cache/TLB | QEMU 没有真实 cache hierarchy；大量 `DC clean/invalidate` 指令要么 NOP，要么退化为共享 RAM 近似。 | DDI 0487 §B2 / §D8 cache maintenance | target/arm/helper.c | P2 / ⚠️ | non-coherent DMA、cache dirty/clean 相关问题会被系统性掩盖 |
| CTLB-02 | Cache/TLB | `DC IVAC` 的破坏性语义和 `DC CISW/CSW/ISW` 等 set/way 维护在 QEMU 中基本无法体现。 | DDI 0487 cache maintenance | target/arm/helper.c | P2 / ⚠️ | 无法验证 invalidate-before-clean 丢数与层级 cache 行为 |
| CTLB-03 | Cache/TLB | system-mode 下 `IC IALLU/IALLUIS` 基本 NOP，`IC IVAU` 也常为 NOP；代码页写入直接触发 TB 失效，掩盖 SMC/JIT 漏序列问题。 | DDI 0487 `IC*`, `DC*`, `DSB`, `ISB` | target/arm/helper.c, system/physmem.c, accel/tcg/tb-maint.c | P2 / ⚠️ | 自修改代码在 QEMU 上更容易“误成功” |
| CTLB-04 | Cache/TLB | TLBI 的 OS/IS、ASID-only、leaf-only 等区分被大量折叠或过度失效。 | DDI 0487 `TLBI` | target/arm/tcg/tlb-insns.c | P2 / ⚠️ | MMU/TLB bug 可能被“全刷”掩盖 |
| CTLB-05 | Cache/TLB | range TLBI 虽支持 opcode，但内部允许退化为更粗粒度 invalidate。 | DDI 0487 FEAT_TLBIRANGE | target/arm/tcg/tlb-insns.c, accel/tcg/cputlb.c | P2 / ⚠️ | 范围失效精度与性能观察值不可信 |
| CTLB-06 | Cache/TLB | PoC/PoU 基本被压平为共享 RAM + TB/TLB 软件缓存模型；`CTR_EL0.DIC/IDC` 也显式告诉 guest 不要依赖真实 cache 维护。 | DDI 0487 PoC/PoU / CTR_EL0 | target/arm/tcg/cpu64.c, target/arm/cpu.c | P2 / ⚠️ | 不能从 QEMU 结果反推真机一致性点语义 |
| CTLB-07 | Cache/TLB | `CVAP/CVADP` 只提供极弱的持久化近似，不等价于真实 PoP/PoDP。 | DDI 0487 persistence / PoP / PoDP | target/arm/helper.c, system/memory.c | P2 / ⚠️ | crash consistency / persistence 边界无法严谨验证 |

## 5. ⚠️ 时序/精度 (P3) 汇总表
| 编号 | 子系统 | 差异描述 | 规范章节 | QEMU 源码 | 严重度 | Guest 影响 |
|---|---|---|---|---|---|---|
| MTE-01 | PAC/BTI/MTE | `GCR_EL1.RRND=1` 仍走确定性 LFSR，只在种子为 0 时补随机 seed。 | DDI 0487 §D24.2.52, §D24.2.156 | target/arm/tcg/mte_helper.c | P3 / ⚠️ | 随机性、分布质量与真机不同 |
| MTE-02 | PAC/BTI/MTE | 异步 tag fault 发生时，`TFSR_ELx/TFSRE0_EL1` 立即置位；未建模 `DSB/ITFSB` 约束的可见窗口。 | DDI 0487 §D10.7.1 | target/arm/tcg/mte_helper.c, target/arm/helper.c | P3 / ⚠️ | 精确时序测试会比硬件更“立即” |
| MTE-03 | PAC/BTI/MTE | 异步 Data Abort 与 TagCheckFault 并发时的 UNKNOWN 语义未建模，QEMU 直接给出确定结果。 | DDI 0487 §D10.7.1 | target/arm/tcg/mte_helper.c | P3 / ⚠️ | 竞争相关测试结果过于确定 |
| MEM-10 | 内存模型/屏障 | RR 模式与保守 fallback 不仅偏强，而且把竞争窗口压成近似串行，失去真实统计分布。 | DDI 0487 并发/原子时序 | accel/tcg/* | P3 / ⚠️ | 重试次数、冲突概率、并发统计均不可信 |
| CTLB-08 | Cache/TLB | TLBI/DSB 依赖 `all_cpus_synced + safe work` 提供同步点，但不呈现硬件式广播延迟和域开销。 | DDI 0487 TLBI + DSB | accel/tcg/cputlb.c, cpu-common.c | P3 / ⚠️ | 只能验证“最终被刷掉”，不能验证传播时间 |

## 6. 按子系统分类视图
### 6.1 GIC (doc 56)
- 6 项：主线实现基本正确，偏差集中在术语拆分、LPI 状态机、ITS 命令作用域及寄存器副作用边界。最重要的是不要把 QEMU/旧文的工程化摘要直接上升为 GIC 架构定论。
### 6.2 EL 状态管理 (doc 57)
- 2 项：异常入口的 PSTATE 规则比“SS/NZCV”口语化描述更复杂；真正的实现偏差在非法异常返回，QEMU 采用了更保守的字段恢复模型。
### 6.3 MMU (doc 58)
- 6 项：QEMU 主 PTW 路径可信，但 regime、DS=1、FWB、fault 分类等边界条件不能被简化成固定模板。
### 6.4 Timer (doc 59)
- 6 项：QEMU 在 PE 视角上高度可用，但 system counter 复位/宽度、offset 条件、SEL2/Secure 门控和板级 PPI 仍是工程化模型。
### 6.5 ISA (doc 60)
- 3 项：decode/主语义基本正确，差异主要在 `ISB`、`SVC`、`IC IVAU` 这些容易被写成“静态规则”的路径。
### 6.6 Debug (doc 61)
- 4 项：Self-hosted debug 较完整；真正的缺口是 Chapter H external debug 与高级 breakpoint 类型，以及一个明确的 OSLSR 位号 bug。
### 6.7 TrustZone/RME (doc 62)
- 4 项：QEMU 已能表达四域 CPU 语义，但 Root/Realm 定义、世界切换完成点、SEL2 前提和板级双地址空间边界必须写清。
### 6.8 SVE/SME (doc 63)
- 3 项：VL/SVL、谓词、ZA 主实现可靠；差异集中在流模式 trap 归属、trap gate 与状态有效性的独立性，以及对 SME2 覆盖范围的收束。
### 6.9 PAC/BTI/MTE (doc 64)
- 5 项：PAC/BTI 主线成熟，但 Armv9.4 的 PAuth_LR/PACM 仍缺失；MTE 的随机性与异步 fault 时序明显工程化。
### 6.10 内存模型/屏障 (doc 65)
- 10 项：这是差异最密集的子系统之一。QEMU 整体“强于 ARM”，对功能正确友好，但对弱序、exclusive monitor、精确 barrier 语义不真实。
### 6.11 Cache/TLB (doc 66)
- 8 项：QEMU 把真实 cache/TLB hierarchy 压缩成共享 RAM + 软件缓存/TB；适合功能验证，不适合一致性、持久化和域传播验证。

## 7. QEMU 作为验证平台的适用性评估
### 7.1 可以验证的场景
- AArch64 指令主语义、异常入口主路径、sysreg trap 主路径、页表遍历主流程。
- 普通 guest OS 的启动、定时器中断、虚拟化基本功能、SVE/SME/PAC/BTI/MTE 的主功能路径。
- 页表修改后的“功能性” TLBI 生效、`DC ZVA`/MTE tag helper/大多数系统指令编码与 trap 是否可达。
- 以源码和实现逻辑为目标的架构学习、调用链分析、功能回归。

### 7.2 不能验证的场景
- Armv9.4 `FEAT_PAuth_LR/PACM` 新指令与相关 BTI landing pad。
- Chapter H 外部调试、真实 Debug state、EDSCR.HDE、ITR/DCC 执行环境。
- 真实 ARM 弱内存 litmus、reservation granule、公平性/伪失败概率。
- 真实 cache hierarchy、PoC/PoU/PoP/PoDP、non-coherent DMA、`DC IVAC` 破坏性语义、OS/IS 域传播。
- 精确 MTE 异步 fault 时序、`RRND=1` 随机性、TLBI 广播延迟。

### 7.3 需要谨慎对待的场景
- 内核并发 bug、lock-free 算法、监控 `STXR` 成败频率、依赖弱序暴露问题的测试。
- 自修改代码/JIT、cache maintenance 完整序列、ASID-only/leaf-only/range TLBI 精细验证。
- SEL2/RME/Realm/Root 场景下把“QEMU 工程抽象”直接当作“规范定义”的分析。
- Debug、Timer、GIC 等子系统中把旧文档的口语化总结当作最终架构术语。

## 8. 建议的改进优先级
1. **优先级 1（先补功能缺口）**：`FEAT_PAuth_LR/PACM`、PAuth_LR landing pad、`LDAPR` 门控修正、exclusive monitor 的 reservation granule / smaller-store clear。
2. **优先级 2（修明确 bug）**：`OSLSR_EL1` OSLK 位号检查、非法异常返回字段恢复与规范对齐。
3. **优先级 3（补架构子系统缺口）**：Chapter H external debug、更完整 breakpoint/VMID/context match 支持。
4. **优先级 4（提升验证真实性）**：把 `LDAR/STLR/LDAPR/LSE A/L/AL` 与 `DMB/DSB` 的语义进一步拆开，减少“过强模型”。
5. **优先级 5（提升 MMU/Cache/TLB 精度）**：收紧 TLBI 域/ASID/leaf-only 语义，补更多 cache maintenance/persistence 边界说明。
6. **优先级 6（精度增强）**：MTE `RRND=1`、`TFSR/TFSRE0` 同步窗口、UNKNOWN race、TLBI 广播延迟等时间/概率模型。
7. **优先级 7（系统级建模）**：Timer system-level MMIO 组件、RME/GPT/板级四域 address space 的更完整 SoC 模型。

## 9. 附录：规范引用索引
| 子系统 | 关键规范章节 |
|---|---|
| GIC | IHI 0048B.b §4.3.2, §4.4.4；IHI 0069H.b §4.1, §4.1.2, §5.3, §12.2.13, §12.2.19, §12.9.4 |
| EL/异常 | DDI 0487 §D1.4.2, §D1.4.4, §D1.5.1 |
| MMU | DDI 0487 §D8.1.2, §D8.2.3, §D8.2.8-§D8.2.10, §D8.6.2, §D8.6.6, §D8.15.1 |
| Timer | DDI 0487 §D12.1.1, §D12.1.2, §D12.2.4 |
| ISA | DDI 0487 §C3.1.5, §C6（ISB / SYS / IC IVAU 相关） |
| Debug | DDI 0487 §D2.10, §D2.11, Chapter H |
| TrustZone/RME | DDI 0487 §D1.1.2, §D1.4.4 |
| SVE/SME | DDI 0487 §C8.1, §C9.1, §D22.2, §D22.3 |
| PAC/BTI/MTE | DDI 0487 §D10.7.1, §D24.2.52, §D24.2.156；A2.2 FEAT_PAuth_LR；C6.2.307 |
| 内存模型 | DDI 0487 §B2.2, §B2.6.2, §B2.6.9, §B2.9-§B2.12 |
| Cache/TLB | DDI 0487 cache maintenance / TLBI / FEAT_TLBIRANGE / PoC / PoU / PoP / PoDP 相关条目 |

> 说明：本汇总是“差异索引”，目标是把 56-66 号文档中所有 ⚠️/❌ 类结论压缩成一份可检索清单；若需要完整上下文，应回看对应源文档。
