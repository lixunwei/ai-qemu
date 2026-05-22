# ARM64 内存模型与屏障语义规范验证

> QEMU 11.0.50 vs ARM DDI 0487 M.b Chapter B2 — 内存排序 / DMB / DSB / ISB / Acquire-Release / Exclusive / LSE 对照
>
> 说明：用户给出的“`B2.4 DMB / B2.5 DSB / B2.6 ISB / B2.7 Acquire-Release / B2.8 Exclusives / B2.10 LSE`”更接近旧版材料的组织方式；**在 DDI 0487 M.b 实际目录里**，屏障语义位于 `§B2.6`，内存类型位于 `§B2.10`，排他同步位于 `§B2.12`。下文按 **M.b 实际章节号** 引用规范，同时保持用户要求的主题划分。

## 1. 验证概述

本次核对的 QEMU 关键路径如下：

- 屏障翻译：`target/arm/tcg/translate-a64.c:2243-2281`
- Acquire/Release：`target/arm/tcg/translate-a64.c:3526-3572`, `4164-4190`, `4237-4290`
- Exclusive：`target/arm/tcg/translate-a64.c:3230-3400`, `3402-3478`
- Exclusive 状态：`target/arm/cpu.h:704-713`, `target/arm/internals.h:691-698`
- 128-bit/atomic fallback：`accel/tcg/ldst_atomicity.c.inc:80-92`, `132-173`, `accel/tcg/cpu-exec.c:549-597`
- TCG 屏障/原子：`include/tcg/tcg-mo.h:28-46`, `tcg/tcg-op.c:289-305`, `tcg/aarch64/tcg-target.c.inc:1584-1593`, `tcg/tcg-op-ldst.c:951-1008`
- host 原子语义：`include/qemu/atomic.h:36-38`, `140-199`, `accel/tcg/atomic_template.h:186-205`, `345-363`
- MTTCG / RR 执行模型：`accel/tcg/tcg-accel-ops.c:65-69`, `accel/tcg/tcg-accel-ops-mttcg.c:106-109`, `accel/tcg/tcg-accel-ops-rr.c:285-298`, `accel/tcg/internal-common.h:25-31`
- shareability / cacheability 简化：`target/arm/ptw.c:1884`, `1965-1967`

### 1.1 结论先行

先给结论：**QEMU 11.0.50 的 ARM64 内存模型实现是“功能正确优先”的并发功能模型，不是对 ARM 弱内存模型的 litmus-accurate 仿真器。**

- **✅ 基本功能正确**：
  - DMB/DSB/ISB 指令都能生成“不会比规范更弱”的同步效果；
  - LDAR/STLR、LDAXR/STLXR、CAS/CASP、LDADD/LDCLR/SWP 等都具备正确的基本原子功能；
  - 128-bit LSE128（`LDCLRP/LDSETP/SWPP`）与 `CASP` 都有实现；
  - 异常返回会清除 exclusive 状态，`CLREX` 也会清除。

- **⚠️ 主要偏差是“过度有序”**：
  - DSB 与 DMB 在翻译层被等价处理；
  - LDAPR 被实现为比 RCpc 更强的 LDAR/LDAQ；
  - LSE Acquire/Release 变体被统一成 full barrier RMW；
  - RR 模式、serial fallback、host seq-cst atomics 共同导致 guest 看到的顺序通常 **强于 ARM 真实硬件**。

- **⚠️ 其次是“exclusive monitor 近似化”**：
  - QEMU 不建模架构定义的 local/global monitor 状态机；
  - `LDXR/STXR` 依靠“地址+旧值+CMPXCHG”近似；
  - 这会漏掉真实硬件的 reservation granule、monitor clear、ABA/同值写回等行为。

- **❌ 少量可见规范偏差**：
  - `trans_LDAPR()` 额外要求 `aa64_lse`：`target/arm/tcg/translate-a64.c:4170-4172`，而 ARM ARM 将 LDAPR 归属于 `FEAT_LRCPC`，不是 `FEAT_LSE`；
  - QEMU 的 exclusive reservation 只记录精确地址，不符合 `§B2.12.3` 允许的 4-512 words reservation granule 范围；
  - DSB 独有的 completion / TLBI-IC maintenance scope 语义没有被单独建模。

### 1.2 差异分类原则

本文使用三种判定：

- **✅ 正确实现**：guest 可观察语义与规范一致，或 QEMU 的实现正好匹配规范允许的某一实现点。
- **⚠️ 简化 / 偏强 / 近似**：通常不会比规范更弱，但不真实、过度有序、或忽略 implementation-defined 细节。
- **❌ 错误 / 缺失 / 门控不符**：存在明确的规范不匹配，或某些特性组合下会与 ARM ARM 定义冲突。

### 1.3 最关键的总体判断

> **QEMU 不是“不足有序”的 ARM64 模拟器；它的主要问题恰恰相反：多数情况下是“比 ARM 规范更强、更同步、更少失败”。**

这意味着：

1. 对大多数 OS / 驱动 / 锁实现，QEMU **更容易跑通**；
2. 对依赖真实弱序行为的 litmus test、lock-free 算法重试特征、exclusive failure 频率分析，QEMU **不够真实**；
3. DSB/ISB/TLBI 等序列在 QEMU 上通常表现为**同步 helper + TB 断裂**，而不是硬件流水线/缓存层级/共享域的真实时间行为。

---

## 2. ARM 内存模型基本性质

### 2.1 规范基线

ARM ARM `§B2.2` 先定义了 single-copy atomicity / multi-copy atomicity。知识库摘录显示：

- `§B2.2.1`：单寄存器对齐 load/store 是 single-copy atomic；pair load/store 通常是“两个单独的 single-copy atomic 元素”；成功的 64-bit `LDXP/STXP`（两条 64-bit）可以形成整个位置的 single-copy atomic 更新（知识库 `arm_architecture_reference_manual.md:10539+`）。
- `§B2.2.4`：ARM 内存模型是 **Other-multi-copy atomic**，定义为“某个写一旦被另一观察者观察到，则所有对该位置进行 coherent 访问的观察者都将观察到它”（知识库 `769236+`）；同时 ARM 明确说 **Normal / Device memory 都不要求天然 multi-copy atomic**（`10655-10665`）。
- `§B2.3.10.2` / `§B2.3.10.3` 又用 external completion / global completion 的形式，把“外部可见性”“全局完成”写成全序关系（知识库 `13139+`, `13260+`）。

### 2.2 QEMU 的基本模型

#### 2.2.1 单 vCPU 程序顺序

QEMU TCG 前端按 guest 程序顺序翻译和发射内存访问；单个 vCPU 的生成代码本身并不主动模拟 ARM 的投机重排。对于同一 vCPU：

- 普通 load/store 以翻译顺序进入 TCG IR；
- barrier / atomic 又通过 `tcg_gen_mb()` 与 atomic helpers 强制附加 host fence。

因此，**QEMU 单 vCPU 自然满足“不会弱于程序顺序”的保守模型**。这对 guest 来说通常 **强于** ARM 对 Normal memory 允许的无依赖重排。

#### 2.2.2 coherence

QEMU 对 RAM 最终落到 host memory + host atomic primitives：

- `qatomic_cmpxchg__nocheck`、`qatomic_fetch_*` 全部是 `__ATOMIC_SEQ_CST`：`include/qemu/atomic.h:152-180`
- 原子 RMW helper 注释明确写着“**These helpers are, as a whole, full barriers**”：`accel/tcg/atomic_template.h:186-205`, `345-363`

所以对于同一 guest RAM location，QEMU 依赖 host 内存系统给出一个稳定的 per-location order。**这通常足以实现 guest 所需的 coherence / SC-per-location。**

#### 2.2.3 multi-copy atomicity

QEMU 没有单独实现“某个写先只对一部分 PE 可见”的传播层；guest RAM 在一个进程地址空间内，由 host cache-coherent memory 承载。因此：

- QEMU **不会主动模拟** ARM 硬件允许的非 multi-copy atomic 传播窗口；
- 在大多数 host 上，写一旦提交到共享内存，就表现得比 ARM 允许模型更接近“全局同时可见”。

结论：

- **判定：⚠️ 过度有序 / 过度一致**
- **原因**：规范只要求 Other-multi-copy atomic；QEMU 通常给出接近“共享内存立即一致”的更强效果。
- **影响**：一些真实硬件上可见的传播延迟 / IRIW 类弱序现象，在 QEMU 中更难复现。

#### 2.2.4 shareability / cacheability 抽象缺失

QEMU 自己就在页表遍历代码里写明：

- `target/arm/ptw.c:1884`：`/* TODO: This code does not support shareability levels. */`
- `target/arm/ptw.c:1965-1967`：`QEMU ignores shareability and cacheability attributes`

这意味着 **ARM 内存模型里围绕 Inner Shareable / Outer Shareable / Non-shareable 的大量区别，在 QEMU 中根本没有被完整建模**。这直接影响后文对 DMB/DSB 域参数的评估：QEMU 并非分别实现了 `NSH/ISH/OSH/SY`，而是基本把 guest 内存都放进一个统一的 host coherence 世界里。

### 2.3 Relaxed ordering：QEMU 是否模拟了 ARM 的弱序？

严格说，**没有完整模拟**。

ARM 允许：

- 对 Normal memory 的无依赖 load / store 进行较大程度乱序；
- RCpc/RCsc、barrier、dependency 只是在这套弱序基础上施加约束。

而 QEMU：

- 不引入硬件级 speculative reordering；
- 在 system mode 下，`tcg_gen_mb()` **无条件认为 parallel=true**，即使单 CPU 也会保留 barrier：`tcg/tcg-op.c:291-305`
- RR 模式中所有 vCPU 轮流串行执行：`accel/tcg/tcg-accel-ops-rr.c:285-298`
- MTTCG 中 atomic/helper 大量使用 seq-cst / full barrier。

所以 QEMU 更像：

- **RR 模式**：近似全局串行功能模型；
- **MTTCG 模式**：host-dependent，但整体仍显著偏强序；
- **不是** ARM litmus 工具里的 relaxed formal machine。

### 2.4 本节结论

| 维度 | ARM 规范 | QEMU 11.0.50 | 判定 |
|---|---|---|---|
| single-copy atomicity | `§B2.2.1` 对不同访问大小有细分要求 | 通过 `MemOp` / atomic helper / CAS 大体落实 | ✅ / ⚠️（pair/monitor 细节见后文） |
| multi-copy atomicity | 仅要求 Other-multi-copy atomic | 通常表现为更强、更接近统一共享内存可见性 | ⚠️ 过度有序 |
| coherence | 每个位置存在一致观察顺序 | host memory + seq-cst atomics 基本满足 | ✅ |
| shareability | NSH/ISH/OSH/SY 有实义 | `ptw.c` 明说未完整支持 shareability | ⚠️ 简化 |
| relaxed ordering | ARM 允许大量无依赖重排 | QEMU 不主动模拟真实弱序传播/投机 | ⚠️ 明显偏强 |

---

## 3. DMB 数据内存屏障

### 3.1 规范要求

`§B2.6.2 DataMemoryBarrier` 明确指出：

- DMB 只保证 **barrier 前后的受影响访问之间存在相对顺序**；
- **不保证 completion**；
- 由 DMB 选项决定受影响的访问类型与 shareability domain（知识库 `14049+`）。

规范原文关键句：

- “`The DMB instruction does not ensure the completion ...`”
- 只影响 memory accesses 与 data/unified cache maintenance，不影响其他普通指令顺序。

### 3.2 QEMU 翻译

QEMU 在 `target/arm/tcg/translate-a64.c:2243-2260`：

- `trans_DSB_DMB()` 把 DSB 和 DMB 共用一个翻译入口；
- 对 `a->types` 做三分：
  - `Reads` → `TCG_BAR_SC | TCG_MO_LD_LD | TCG_MO_LD_ST`
  - `Writes` → `TCG_BAR_SC | TCG_MO_ST_ST`
  - `All` → `TCG_BAR_SC | TCG_MO_ALL`
- 然后统一 `tcg_gen_mb(bar)`。

这与 ARM 中 DMB 的 access-type 语义大致对应：

- `LD` 类覆盖 `rr/rw`
- `ST` 类覆盖 `ww`
- `SY/ALL` 覆盖全部

### 3.3 QEMU 没有区分 shareability domain

`trans_DSB_DMB()` 只看 `a->types`，**完全不使用 domain 选项**；而 AArch64 backend 的 `tcg_out_mb()` 又固定发射：

- `MO_ALL` → `DMB_ISH | DMB_LD | DMB_ST`
- `MO_ST_ST` → `DMB_ISH | DMB_ST`
- `MO_LD_LD` / `MO_LD_ST` → `DMB_ISH | DMB_LD`

见 `tcg/aarch64/tcg-target.c.inc:1584-1593`。

也就是说：

- guest `DMB NSHLD/NSHST/NSH/SY/OSH/ISH` 在 QEMU AArch64 host backend 上最终都被压扁为 **ISH 域** 的某种 `DMB`；
- **`SY/OSH/NSH/ISH` 的域差异被抹平**。

结合 `ptw.c:1884`, `1965-1967` 可知，这不是偶然遗漏，而是 QEMU 整体就没有完整 shareability model。

### 3.4 对 Normal / Device memory 的效果区别是否建模？

ARM 在 `§B2.10 Memory types and attributes` 中强调：

- shareable Normal memory 与 Non-shareable Normal memory、Device memory 的可见性/排序/外围到达次序不同；
- 对 non-cacheable Normal memory，DMB 还会在某个 implementation-defined peripheral block 上引入 barrier-ordered-before（知识库 `14953+`）。

QEMU 的 DMB 路径并不读取 memory type：

- `TCGBar` 只描述 `rr/rw/wr/ww + acq/rel/sc`：`include/tcg/tcg-mo.h:28-46`
- `MemOp` 只描述大小、符号、端序、对齐、atomicity：`include/exec/memop.h:17-126`
- **没有任何“Device vs Normal 的 barrier 差异位”**

因此：

- **QEMU 不会基于 guest memory type 改变 DMB 语义**；
- 对 MMIO/device，真实硬件存在的“arrival / completion / peripheral ordering”细粒度行为，在 QEMU 里大多被 memory-region callback 的同步执行替代。

### 3.5 DMB 是否错误？

不是“更弱”，而是“更粗”。

- 对普通 guest 软件，QEMU 的 DMB 通常 **不比规范更弱**；
- 但它也 **不区分域、不区分 memory type、不表现传播层级**；
- 因而它是 **功能上正确、微观时序上不真实** 的实现。

### 3.6 DMB 结论

| 项目 | 规范 | QEMU 实现 | 判定 | 影响 |
|---|---|---|---|---|
| LD/ST/ALL scope | `§B2.6.2` 明确定义 | `translate-a64.c:2248-2257` 正确映射 rr/rw/ww/all | ✅ | 基本 guest 正确 |
| shareability domain (`NSH/ISH/OSH/SY`) | 架构有区别 | backend 固定 `DMB_ISH*`，无域模型 | ⚠️ 简化 | 时序不真实 |
| DMB 不保证 completion | 规范明确如此 | QEMU 也未单独建 completion | ✅ | 与 DMB 本义一致 |
| Device vs Normal 差异 | 规范有区别 | QEMU barrier 不读取 memory type | ⚠️ 简化 | 外设排序细节不真实 |

---

## 4. DSB 数据同步屏障

### 4.1 规范要求

`§B2.6.9 DataSynchronizationBarrier` 明确：

- DSB 要求 **barrier 前的受影响 memory accesses 在 DSB 完成前已经完成**；
- 它比 DMB 更强；
- 同选项下，DSB 也包含 DMB 产生的 ordering；
- `§B2.6.9.1` 进一步把 `DSB NSH/ISH/OSH` 与 TLBI / TLBIP / IC maintenance scope 绑定起来（知识库 `14181+`, `14272+`）。

所以，DSB 独有的关键点是：

1. **completion**，不只是 relative order；
2. **maintenance scope**，尤其与 TLBI/IC 交互。

### 4.2 QEMU 实现

QEMU 在 `translate-a64.c:2243-2245` 直接写了注释：

```c
/* We handle DSB and DMB the same way */
```

随后与 DMB 共用完全相同的 `TCGBar` 生成逻辑。

这意味着：

- QEMU **没有单独的 DSB IR 语义**；
- QEMU **没有为 DSB completion / maintenance scope 增加任何特殊处理**；
- 在 TCG 层，DSB ≈ DMB。

### 4.3 少了哪些 DSB 独有保证？

#### 4.3.1 completion 语义缺失

ARM：DSB 要求 prior accesses complete before DSB completes。  
QEMU：只发 barrier，不建“访问完成点/全局完成点”状态机。

不过 QEMU 大多数 RAM 访问本来就是同步执行，helper 也是同步调用，因此很多情况下 guest **感觉不到** 这点缺失；但这并不等于 QEMU 真正实现了 DSB completion 的架构语义。

#### 4.3.2 TLBI / IC / cache maintenance scope 未专门建模

ARM `§B2.6.9.1` 明确把 `DSB NSH / ISH / OSH` 与维护指令适用范围关联。  
QEMU 的 barrier path：

- 不区分 `NSH/ISH/OSH`；
- 也不在 `trans_DSB_DMB()` 里与 TLBI/IC helper 协同。

QEMU 对 TLBI / TB flush 常通过别的同步机制实现（helper 立即更新软件 TLB、退出 TB、全局 flush 等），所以 **许多实际 guest 序列仍然能跑对**；但“为什么跑对”往往是因为 **helper 本身同步完成**，不是因为 DSB 被精确实现。

#### 4.3.3 Device / maintenance side effect completion 没有硬件层次

真实硬件的 DSB 还约束缓存层次、TLB 维护、部分系统寄存器副作用在共享域上的完成。  
QEMU 没有 cache hierarchy / interconnect / outer-shareable fabric 的真实模型，因此这些 completion 差异被“软件直接完成”替代。

### 4.4 DSB 是“过弱”还是“过强”？

这是最容易误判的点。

- **从实现形式看**：DSB 被降格成 DMB，像是“过弱”；
- **从 guest 实际可观察结果看**：由于 QEMU 访存与 helper 通常同步完成，很多场景又表现得**比真实硬件更同步**。

所以最准确的判断是：

> **QEMU 对 DSB 的问题不是“典型弱化”，而是“没有单独建模 DSB completion，转而依赖整体功能执行已经足够同步”。**

这会导致：

- 对普通内存访问：通常看起来没问题；
- 对维护指令/共享域层次：细节不真实；
- 对精确 barrier litmus：DSB 与 DMB 可观察差异被压缩。

### 4.5 DSB 结论

| 项目 | 规范 | QEMU 实现 | 判定 | 影响 |
|---|---|---|---|---|
| DSB 比 DMB 更强 | `§B2.6.9` 明确 | `translate-a64.c:2245` 明说两者同实现 | ⚠️ 简化 |
| prior accesses complete | 规范要求 | 未单独建模完成点 | ⚠️ |
| DSB 维护域 (`NSH/ISH/OSH`) | `§B2.6.9.1` 明确 | 无域区别 | ⚠️ |
| TLBI/IC completion 语义 | 规范要求 | 靠 helper 同步副作用近似 | ⚠️ |
| 对常规 guest 功能正确性 | 多数场景可满足 | 通常“足够同步” | ✅ / ⚠️ |

---

## 5. ISB 指令同步屏障

### 5.1 规范要求

`§B2.6.1 InstructionSynchronizationBarrier` 要求：

- ISB 后面的指令必须在 ISB 完成后重新从 cache/memory 获取；
- ISB 使得前面 context-changing operations（例如 system register 改写、已完成的 cache/TLB maintenance）对之后取到的指令可见；
- 后续 context-changing operations 只在 ISB 之后生效（知识库 `14037+`）。

本质上，ISB 是 **context synchronization** 与 **fetch/pipeline 刷新**。

### 5.2 QEMU 实现

`target/arm/tcg/translate-a64.c:2272-2281`：

- `trans_ISB()` 不生成专门的 mb IR；
- 只做两件事：
  - `reset_btype(s);`
  - `gen_goto_tb(s, 0, 4);`

注释写得很直接：

- 为了正确处理 self-modifying code；
- 为了立刻接收 pending interrupts；
- 因此 **需要在 ISB 后断开当前 TB**。

### 5.3 这是否等效于 ISB？

#### 5.3.1 在 QEMU 的功能模型里：大体等效

TB 断裂意味着：

- 之前的 MSR / helper / 状态更新先提交；
- 下一条指令在新的翻译/执行边界上重新取用 hflags / cpreg / TB key；
- pending interrupt 也能在主循环边界被立刻看到。

对 guest 可见效果来说，这已经抓住了 ISB 最核心的“**后续指令必须在更新后的上下文下执行**”。

#### 5.3.2 但并不是真实硬件意义上的 pipeline flush

QEMU 没有硬件 fetch queue、decode queue、branch predictor、predecode pipeline 的真实状态，因此：

- ISB 不可能体现“流水线被刷新到什么阶段”；
- 也不会表现真实硬件上 ISB 的性能代价和 speculative fetch 细节；
- 它是 **语义级 context synchronization**，不是微架构级 pipeline model。

#### 5.3.3 对 self-modifying code 的正确性

这里 QEMU 的做法是合理的：

- TB 断裂 + 代码页失效机制，确保后续指令不会继续沿用旧 TB；
- 对 guest 观察到的“修改后代码在 ISB 之后才生效”这一点，通常能满足需求。

### 5.4 ISB 结论

| 项目 | 规范 | QEMU 实现 | 判定 | 影响 |
|---|---|---|---|---|
| context synchronization | `§B2.6.1` 要求 | TB 断裂后重新执行 | ✅ |
| 后续取指在新上下文下进行 | 规范要求 | `gen_goto_tb()` 基本满足 | ✅ |
| 硬件级 pipeline flush / fetch 细节 | 真实硬件有 | QEMU 不建模 | ⚠️ |
| self-modifying code 生效边界 | ISB 后 | 通过 TB break 近似 | ✅ |

---

## 6. Load-Acquire / Store-Release

### 6.1 规范要求

`§B2.6.10 Load-Acquire, Load-AcquirePC, and Store-Release` 规定：

- `Load-Acquire` / `Store-Release` 支持 **RCsc**；
- `Load-AcquirePC + Store-Release` 支持更弱的 **RCpc**；
- `Load-Acquire` / `Load-AcquirePC` 都是“**向后约束**”——约束本条 load 与其后的访问；
- `Store-Release` 是“**向前约束**”——约束前面的访问先于本条 store 被观察到；
- `Store-ReleaseExclusive` 仅在 store 成功时才带 release 语义（知识库 `14287+`, `14300+`）。

因此 Acquire/Release 是**单向屏障**，不是双向 full fence。

### 6.2 LDAR / STLR 的 QEMU 翻译

- `trans_STLR()`：`target/arm/tcg/translate-a64.c:3526-3549`
  - 先 `tcg_gen_mb(TCG_MO_ALL | TCG_BAR_STRL)`
  - 再执行 store
- `trans_LDAR()`：`target/arm/tcg/translate-a64.c:3552-3572`
  - 先执行 load
  - 再 `tcg_gen_mb(TCG_MO_ALL | TCG_BAR_LDAQ)`

这在 IR 层面已经可见两个特点：

1. QEMU 把 LDAR/STLR 都绑定到 **`TCG_MO_ALL`**，不是只绑定最小必要的 `rr/rw` 或 `ww/wr` 子集；
2. barrier kind 虽然带了 `LDAQ/STRL`，但 AArch64 backend `tcg_out_mb()` **只看 `a0 & TCG_MO_ALL`**，完全不看 `TCG_BAR_LDAQ/STRL`：`tcg/aarch64/tcg-target.c.inc:1584-1593`。

结果：

- host AArch64 backend 上，LDAR/STLR 最终都更接近 `DMB ISH` 级别的 **full fence**；
- ARM 规范的一向 Acquire / Release distinction 在 backend 发射时被“压平”。

### 6.3 这意味着什么？

#### 6.3.1 功能上不会比规范更弱

LDAR 需要“后续访问不能跑到它前面”，QEMU 给的是 full fence；  
STLR 需要“前序访问不能拖到它后面”，QEMU 给的也是 full fence。

因此 **不会出现 guest 看到比规范更弱的排序**。

#### 6.3.2 但明显过度有序

ARM 的 one-way barrier 可允许部分另一方向的并行性；QEMU 直接给 full fence，会：

- 抹掉 RCsc acquire/release 本来比 DMB SY 更精细的弱序空间；
- 让某些 litmus 本该成立的重排序结果在 QEMU 中消失；
- 增大与真实硬件性能 / 失败窗口 / 并发行为的差距。

### 6.4 LDAPR（Load-AcquirePC）

`trans_LDAPR()`：`target/arm/tcg/translate-a64.c:4164-4190`

关键注释：

- `4181-4186` 明写：**architectural consistency requirements are weaker than full load-acquire ... but we choose to implement them as full LDAQ**。

这已经是源码级自证：

- ARM `LDAPR` 是 RCpc；
- QEMU 明知其更弱，仍故意实现成 full LDAQ。

因此：

- **语义分类：⚠️ 明显过度有序**。

### 6.5 LDAPR 的额外门控问题

`trans_LDAPR()` 还有一个独立问题：

```c
if (!dc_isar_feature(aa64_lse, s) ||
    !dc_isar_feature(aa64_rcpc_8_3, s)) {
    return false;
}
```

见 `target/arm/tcg/translate-a64.c:4170-4172`。

但 ARM ARM 把 LDAPR 定义在 **FEAT_LRCPC** 下（知识库 `4638-4648`, `14287+`），并不要求 `FEAT_LSE`。这意味着：

- QEMU 对 base-form LDAPR 的译码门槛 **比规范更严格**；
- 若存在“支持 LRCPC 但不支持 LSE”的 CPU 组合，QEMU 会把规范允许的 LDAPR 错误地当成未实现。

`trans_LDAPR_i()` 倒是只检查 `aa64_rcpc_8_4`：`target/arm/tcg/translate-a64.c:4237-4263`，进一步说明 base LDAPR 上的 `aa64_lse` 门控并不一致。

结论：

- **判定：❌ 门控不符规范**
- **实际影响**：对当前多数“全特性”CPU 模型影响有限，但对特性组合精确建模不正确。

### 6.6 LoadLOAcquire / StoreLORelease（LOR）

QEMU 在 `trans_STLR()` / `trans_LDAR()` 里注释：

- `StoreLORelease is the same as Store-Release for QEMU`
- `LoadLOAcquire is the same as Load-Acquire for QEMU`

见 `target/arm/tcg/translate-a64.c:3532-3537`, `3558-3560`。

如果只看这两句，似乎是偏差；但结合 `target/arm/helper.c:5173-5203`：

- QEMU 把 `LORSA_EL1/LOREA_EL1/LORN_EL1/LORC_EL1/LORID_EL1` 全部做成常量 0；
- 注释明确说这是 **trivial implementation of ARMv8.1-LOR**，表示 **zero supported Limited Ordering regions**。

而 ARM 规范 `§B2.4.3.1` 又明确说：

- 如果没有实现 LORegions，那么 `LoadLOAcquire/StoreLORelease` 就退化成普通 `Load-Acquire/Store-Release`（知识库 `13598-13629`）。

所以这里结论应分开看：

- **LOR region 本身：✅ QEMU 选择了“实现 LOR 寄存器但 0 region”的合法实现点**；
- **LO 指令最终仍被实现为 full LDAQ / STRL full fence：⚠️ 相对真实 LDAR/STLR 仍偏强序。**

### 6.7 STLR_i 与 `SCTLR.nAA`

`trans_STLR_i()` 在 `target/arm/tcg/translate-a64.c:4276` 留有：

```c
/* TODO: ARMv8.4-LSE SCTLR.nAA */
```

说明 QEMU 已知这里还有规范边界未补齐。结合 `check_ordered_align()` (`419-437`) 可知：

- 部分有序访问的对齐规则会受 `s->naa` / FEAT_LSE2 影响；
- 但 `STLR_i` 的 `nAA` 细节并未完整落地。

这属于：

- **判定：⚠️ 局部未完成 / 对齐边界简化**。

### 6.8 本节结论

| 项目 | QEMU 位置 | 结论 |
|---|---|---|
| LDAR / STLR 基本功能 | `3526-3572` | ✅ 功能正确 |
| LDAR / STLR 单向屏障精度 | `3526-3572`, `1584-1593` | ⚠️ 被实现成更强的 full fence |
| LDAPR RCpc vs LDAR RCsc 区分 | `4164-4190` | ⚠️ 未区分，直接实现成 full LDAQ |
| LDAPR 特性门控 | `4170-4172` | ❌ 额外要求 `aa64_lse`，严于规范 |
| LOR 0 region 语义 | `helper.c:5173-5203` + `§B2.4.3.1` | ✅ 合法 trivial implementation |
| `SCTLR.nAA` 细节 | `4276` | ⚠️ TODO 未完 |

---

## 7. Exclusive Monitor

### 7.1 规范要求

在 M.b 中，exclusive 相关内容位于 `§B2.12 Synchronization and semaphores`，不是用户给出的旧版 `B2.8`。

ARM 规范明确区分：

- **local monitor**：针对 non-shareable memory，本 PE 本地保留状态（知识库 `15540+`）；
- **global monitor**：针对 shareable memory，跟踪“某 PE 对某个 marked block 的排他保留”，其他 observer 对该 marked block 的成功写必须清除标记（知识库 `15650+`）；
- **reservation granule**：`§B2.12.3` 规定由 implementation-defined 参数 `a` 决定，范围 **4-512 words**（知识库 `15788+`）；
- **context switch / exception return**：`§B2.12.4` 说明 **exception return clears the local monitor**，因此上下文切换通常无需额外 `CLREX`（知识库 `15788+`）；
- **Load-Exclusive/Store-Exclusive pair** 的使用限制与成功条件有专门条款（`§B2.12.5`）。

### 7.2 QEMU 的实现核心

QEMU 在 `target/arm/tcg/translate-a64.c:3230-3240` 直接写明：

- Exclusive 是通过“记住 load 时的地址和值，再在 store 时检查它们是否仍相同”实现；
- **“This is not actually the architecturally mandated semantics”**；
- store-exclusive 通过 atomic cmpxchg 避免 MTTCG 竞态。

也就是说，QEMU 自己承认它实现的是**近似模型**。

状态只保存为三个字段：

- `exclusive_addr`：`target/arm/cpu.h:704`
- `exclusive_val`：`target/arm/cpu.h:705`
- `exclusive_high`：`target/arm/cpu.h:713`

`CLREX` / `arm_clear_exclusive()` 只把 `exclusive_addr` 置为 `-1`：

- `trans_CLREX()`：`target/arm/tcg/translate-a64.c:2237-2240`
- `arm_clear_exclusive()`：`target/arm/internals.h:695-698`

### 7.3 Load-Exclusive / Store-Exclusive 的具体行为

#### 7.3.1 Load side

`gen_load_exclusive()`：`target/arm/tcg/translate-a64.c:3241-3284`

- 执行真正的 load；
- 把读到的值保存到 `exclusive_val` / `exclusive_high`；
- 把地址保存到 `exclusive_addr`；
- 对 pair 访问：
  - 32-bit pair 使用 64-bit single-copy atomic load；
  - 64-bit pair 用 `tcg_gen_qemu_ld_i128()` 读取，但 `check_atomic_align()` / `MO_ATOM_IFALIGN_PAIR` 保持“每个 64-bit 元素原子、不是整个 128-bit 一次性读原子”的语义，符合 ARM 对 `LDXP` 的要求。

#### 7.3.2 Store side

`gen_store_exclusive()`：`target/arm/tcg/translate-a64.c:3286-3400`

它的真实判断条件是：

1. `clean_addr == exclusive_addr`
2. 当前内存值仍等于 `exclusive_val`（pair 时还要比较 `exclusive_high`）
3. 然后用 `atomic_cmpxchg_i64/i128` 尝试提交
4. 成功返回 `0`，失败返回 `1`
5. 最后清空 `exclusive_addr`

### 7.4 与真实硬件 exclusive monitor 的差异

#### 7.4.1 没有 local/global monitor 状态机

ARM 规范里，shareable memory 的成功与否还依赖 global monitor；其他 observer 对 marked block 的写会清除标记。  
QEMU 没有显式 local/global monitor，只是：

- 记录一个地址和值；
- 依赖最终 cmpxchg 是否还能成功。

**结论：⚠️ 近似而非规范状态机。**

#### 7.4.2 reservation granule 被缩小成“精确地址”

ARM `§B2.12.3` 要求 reservation granule 是 4-512 words。  
QEMU 在 `3307-3312` 甚至显式写了 `FIXME`：

- 只记录了地址，不是整个范围；
- 假设 load/store 两边访问大小相同；
- 架构其实允许 store smaller than load，只要落在 monitor 记录范围内。

这说明 QEMU：

- **既没有实现 granule，也没有实现“子范围 store 仍可成功”的架构规则**；
- 更像“exact-address reservation”。

这与 ARM 允许的实现范围并不一致。

**判定：❌ 规范不符。**

但要补一句非常关键的工程判断：

- 对“写正确的锁代码”影响通常有限，因为大多数软件不会依赖 granule 邻接地址行为；
- 对精细的 concurrency litmus / retry 频率 / 邻址干扰测试，则是明确差异。

#### 7.4.3 ABA / 同值写回会导致“虚假成功”

真实 hardware monitor 关注的是“reservation 是否被别的 observer 破坏”，不是“内存值现在看起来是否还一样”。  
QEMU 关注的是：

- 地址还一样；
- 值还是旧值；
- cmpxchg 成功。

于是出现典型差异：

- 另一 PE 先把值从 `A` 改成 `B`，再改回 `A`；
- 真实硬件通常已经丢失 reservation，`STXR` 应失败；
- QEMU 看到当前值仍是 `A`，就可能成功。

这属于 **QEMU 独有的“过度成功”**。

**判定：⚠️ / 接近 ❌ 的近似。**

#### 7.4.4 QEMU 不建模“伪失败”

ARM 硬件允许因为实现内部原因、monitor 粒度、冲突、迁移等让 `STXR` 失败得比“值改变”更频繁。  
QEMU 除地址不等 / cmpxchg 失败外，没有更多失败来源，所以：

- **失败率通常低于真实硬件**；
- 这也是过度乐观的一种。

但规范通常允许“多失败”，不要求“少失败”，因此这更像 **微观行为差异** 而不是功能错误。

#### 7.4.5 异常返回清除 exclusive：QEMU 是对的

ARM `§B2.12.4`：exception return clears the local monitor。  
QEMU 在：

- `target/arm/tcg/helper-a64.c:622-645` 的 `HELPER(exception_return)` 里执行 `arm_clear_exclusive(env);`

因此 A-profile 的“异常返回清 monitor”是吻合规范的。用户提示里说“异常入口清除”，但 M.b 实际给的是 **exception return**。按 M.b，QEMU 这里是对的。

#### 7.4.6 CLREX：QEMU 语义够用

`CLREX` 只需清除 monitor；QEMU 用 `exclusive_addr=-1` 作为“monitor clear”哨兵值。虽然没有清 `exclusive_val/high`，但因为 `exclusive_addr` 必须先匹配，所以架构效果等价。

**判定：✅ 正确实现。**

### 7.5 LDXP/STXP 128-bit pair

QEMU 对 pair 访问做了相当细致的区分：

- `check_atomic_align()` 中注释：`size == MO_128` 时，`LDXP` 的操作“对每个 doubleword single-copy atomic，不是整个 quadword”，但仍要求 quadword 对齐：`target/arm/tcg/translate-a64.c:401-415`
- `gen_store_exclusive()` 对 64-bit pair 成功时使用 `tcg_gen_atomic_cmpxchg_i128()`：`3371-3386`

这与 ARM `§B2.2.1` 中“成功的 64-bit pair store-exclusive 可形成整个 memory location 的 single-copy atomic update”是匹配的。

**判定：✅ 基本正确。**

### 7.6 本节结论

| 项目 | QEMU 位置 | 结论 |
|---|---|---|
| CLREX 清除 monitor | `2237-2240`, `695-698` | ✅ |
| exception return 清除 monitor | `helper-a64.c:633` | ✅ |
| LDXP/STXP pair 成功时 128-bit 原子更新 | `3371-3386` | ✅ |
| local/global monitor 状态机 | 无显式状态机 | ⚠️ 近似 |
| reservation granule | `3307-3312` 明确 FIXME，仅精确地址 | ❌ |
| smaller store within monitored range | 未实现 | ❌ |
| ABA / 同值写回导致的虚假成功 | 值比较 + cmpxchg 近似 | ⚠️ |
| 伪失败频率 | 明显少于真实硬件 | ⚠️ |

---

## 8. LSE 原子指令

### 8.1 规范要求

在 M.b 中，LSE 的内存模型效果主要散落在：

- `§B2.6.10`（Acquire/Release 与 RCsc/RCpc）
- `§B2.2.1`（single-copy atomicity）
- `§B2.3.2.2` / `§B2.3.2.3`（CAS / SWP 的 intrinsic dependency）
- 指令章节 `C3/C6`（知识库表项：`LDADD*` `18075+`, `SWP*` `18246+`, `CAS*` `18283+`, `FEAT_LSE128` `7112+`）

规范上，LSE 的关键是：

- 单条原子 RMW；
- Acquire/Release 变体只提供所需方向的 ordering；
- 不同大小 / pair / 128-bit 变体有明确原子性要求。

### 8.2 QEMU 的基础实现

#### 8.2.1 64-bit 及以下 LSE RMW

`do_atomic_ld()`：`target/arm/tcg/translate-a64.c:4062-4103`

- `LDADD` → `tcg_gen_atomic_fetch_add_i64`
- `LDCLR` → `tcg_gen_atomic_fetch_and_i64`（先取反）
- `LDEOR` → `tcg_gen_atomic_fetch_xor_i64`
- `LDSET` → `tcg_gen_atomic_fetch_or_i64`
- `LDSMAX/LDSMIN/LDUMAX/LDUMIN` → 对应 fetch_max/min
- `SWP` → `tcg_gen_atomic_xchg_i64`

这些都直接走 TCG atomic primitives，因此**单条 RMW 原子性是有的**。

#### 8.2.2 CAS / CASP

- `gen_compare_and_swap()`：`3402-3418`
- `gen_compare_and_swap_pair()`：`3420-3478`
- `trans_CAS()` / `trans_CASP()`：`3599-3618`

`CAS` 直接 `tcg_gen_atomic_cmpxchg_i64`；`CASP` 则对 pair 组装后调用 `cmpxchg_i64` / `cmpxchg_i128`。

因此 CAS/CASP 也具备单条原子 RMW 特征。

#### 8.2.3 FEAT_LSE128

QEMU 明确支持：

- `aa64_lse128` feature bit：`target/arm/cpu-features.h:842-845`
- `LDCLRP` / `LDSETP` / `SWPP`：`target/arm/tcg/translate-a64.c:4157-4162`

所以对 **FEAT_LSE128**，QEMU 不是完全缺失，而是已经实现了主要 128-bit RMW 组。

### 8.3 Acquire/Release 变体在 QEMU 中被统一成 full barrier

这是 LSE 实现里最重要的偏差。

`do_atomic_ld()` 与 `do_atomic128_ld()` 的注释都写明：

```c
/* The tcg atomic primitives are all full barriers.
 * Therefore we can ignore the Acquire and Release bits of this instruction.
 */
```

见：

- `target/arm/tcg/translate-a64.c:4079-4083`
- `target/arm/tcg/translate-a64.c:4144-4151`

也就是说：

- `LDADD` / `LDADDA` / `LDADDL` / `LDADDAL` 在 QEMU 中都落到同一类 full-barrier primitive；
- `SWP/SWPA/SWPL/SWPAL` 同理；
- `LDCLRP/LDCLRPA/LDCLRPAL`、`SWPP/SWPPA/SWPPAL/SWPPL` 也同理。

再往下看 helper：

- `atomic_template.h:186-205`, `345-363` 明说整个 helper 是 full barrier；
- `include/qemu/atomic.h:152-180` 又把 cmpxchg/fetch 都绑定到 `__ATOMIC_SEQ_CST`。

因此 **QEMU 的 LSE Acquire/Release 变体基本全部塌缩成“顺序一致的 full-fence RMW”**。

### 8.4 这会导致什么差异？

#### 8.4.1 功能上通常更强，不会更弱

Acquire-only / Release-only / AcquireRelease 在 ARM 上可有细粒度差异；QEMU 全部升格为 full barrier，因此：

- 对 guest 正确性通常无害；
- 但会把真实硬件允许的更多并发行为排除掉。

#### 8.4.2 LSE litmus 会明显过度同步

例如：

- `LDADDAL` 与 `LDADDL` 在硬件上仅差 acquire；
- `SWPAL` 与 `SWPL` 在硬件上仅差 acquire；
- `CASA/CASL/CASAL/CAS` 也有不同 ordering。

QEMU 下这些差异被压缩，导致：

- litmus 结果空间过小；
- barrier/atomic 组合比真实硬件更“SC-like”。

### 8.5 CAS / CASP 的 ordering 也偏强

虽然 `gen_compare_and_swap()` 本身没有像 `do_atomic_ld()` 那样直接写注释，但其依赖的 `tcg_gen_atomic_cmpxchg_i64/i128`：

- 在 `CF_PARALLEL` 下会调用 atomic helper / helper_exit_atomic：`tcg/tcg-op-ldst.c:951-1008`
- helper 最终使用 seq-cst cmpxchg：`include/qemu/atomic.h:152-156`

所以 CAS/CASP 变体同样偏向 full barrier / seq-cst。

### 8.6 host 不支持大原子时的 serial fallback

`ldst_atomicity.c.inc` 表示：

- 若需要的 host 原子性做不到，就 `cpu_loop_exit_atomic()`：`132-173`
- 之后进入 `cpu_exec_step_atomic()`：`accel/tcg/cpu-exec.c:549-597`
- 在 exclusive context 里：
  - `start_exclusive()` 停止其他 CPU
  - 清掉 `CF_PARALLEL`
  - 只执行一条指令
  - `end_exclusive()` 恢复并行

同时 `cpu_in_serial_context()` 定义：只要不在 `CF_PARALLEL` 或处于 exclusive context，即视为 serial：`accel/tcg/internal-common.h:25-31`。

这意味着：

- 当 host 没有足够原生原子能力时，QEMU 会用“暂停其他 vCPU + 串行执行一条 guest 指令”来兜底；
- 语义上通常 **更强**，但性能和时间行为与硬件完全不同。

### 8.7 本节结论

| 项目 | QEMU 实现 | 判定 |
|---|---|---|
| LDADD/LDCLR/LDEOR/LDSET/SWP 基本原子性 | `4062-4113` | ✅ |
| CAS / CASP 基本原子性 | `3402-3478`, `3599-3618` | ✅ |
| FEAT_LSE128 (`LDCLRP/LDSETP/SWPP`) | `4157-4162`, `842-845` | ✅ |
| Acquire/Release 变体精细区分 | `4079-4083`, `4144-4147` 明说忽略 bits | ⚠️ 统一成 full barrier |
| host 无原子能力 fallback | `132-173`, `549-597` | ⚠️ 语义保守、时序不真实 |

---

## 9. QEMU 内存模型的根本特征

### 9.1 RR 模式：近似全局串行

`accel/tcg/tcg-accel-ops-rr.c:285-298` 显示 RR 模式一次只跑一个 vCPU。  
这意味着：

- 多 vCPU 不会真正并行执行普通 guest 指令；
- 大量真实硬件弱序窗口自然消失；
- barrier / atomic 更多只是“逻辑同步点”，而不是与另一个并发 vCPU 竞争中的硬件屏障。

**结论：RR 模式强于 ARM 真实并发。**

### 9.2 MTTCG：独立 host 线程 + 强原子 + serial fallback

MTTCG 中：

- `CF_PARALLEL` 在多 CPU 场景打开：`accel/tcg/tcg-accel-ops.c:67`
- `EXCP_ATOMIC` 会转 `cpu_exec_step_atomic()`：`accel/tcg/tcg-accel-ops-mttcg.c:106-109`
- unsupported atomic / 128-bit atomic 可退化到 exclusive serial context：`ldst_atomicity.c.inc:171-173`, `cpu-exec.c:555-597`

因此 MTTCG 虽然比 RR 更接近真实 SMP，但它的并发基础是：

1. host 普通共享内存；
2. seq-cst host atomics；
3. 需要时 stop-the-world 一条指令串行化。

这不是 ARM formal model，而是一个**保守正确**的功能并发实现。

### 9.3 它更像 TSO 吗？

不应简单地说“QEMU = TSO”。更准确的说法是：

- **plain accesses 的行为受 host 普通内存模型影响**；
- **barrier / atomics 大量被提升到 full barrier / seq-cst**；
- **RR 与 serial fallback 又会进一步强化顺序**。

所以 QEMU 更像：

> **host-dependent but guest-strong functional model**，整体往往强于 ARM，而不是一个固定可描述为“TSO”的抽象机。

在 x86 host 上，这种偏强更明显，因为 host 本身又是 TSO；  
在 AArch64 host 上，虽然 host memory model 不是 x86 TSO，但 QEMU 也没有去重建 guest ARM 的微观弱序，因此仍然**不是**“真实 ARM 硬件那样的弱序”。

### 9.4 “最危险”的不是过弱，而是过强

对于验证 guest 软件来说：

- **不足有序** 会让 guest 看到规范不允许的坏结果，这是致命错误；
- **过度有序** 则会让一些本应在硬件上暴露的并发 bug 在 QEMU 上“藏起来”。

QEMU 11.0.50 在本专题上更接近第二类。

---

## 10. 综合差异汇总表

| 主题 | 规范基线 | QEMU 11.0.50 | 结论 | 类型 |
|---|---|---|---|---|
| ARM relaxed memory model | ARM 允许大量无依赖重排 | QEMU 不主动模拟真实弱序传播/投机 | 偏强序 | ⚠️ 过度有序 |
| DMB LD/ST/ALL | `§B2.6.2` | `2248-2257` 正确映射访问类型 | 基本正确 | ✅ |
| DMB shareability domain | `NSH/ISH/OSH/SY` 有区别 | backend 固定 `DMB_ISH*`，`ptw.c` 忽略 shareability | 域塌缩 | ⚠️ 简化 |
| DMB completion | DMB 不要求 completion | QEMU 也未额外提供 | 一致 | ✅ |
| DSB vs DMB | DSB 更强且要求 completion | `2245` 直接同实现 | completion 未单独建模 | ⚠️ |
| DSB+TLBI/IC scope | `§B2.6.9.1` | 无专门 maintenance scope | 细节缺失 | ⚠️ |
| ISB | context synchronization + 重新取指 | `2272-2281` TB 断裂 | 语义上基本对 | ✅ / ⚠️ |
| LDAR/STLR | one-way RCsc | `3526-3572` + backend full-fence 化 | 偏强 | ⚠️ |
| LDAPR | RCpc，比 LDAR 弱 | `4181-4186` 明说按 full LDAQ 实现 | 偏强 | ⚠️ |
| LDAPR feature gate | 只要求 LRCPC | `4170-4172` 还要求 LSE | 门控过严 | ❌ |
| LOR | 若 0 region 可退化为普通 acquire/release | `5173-5203` 固定 0 region | 合法 trivial impl | ✅ |
| STLR_i + nAA | 需遵守 `SCTLR.nAA` 等细节 | `4276` 仍 TODO | 局部未完成 | ⚠️ |
| CLREX | 清 monitor | `2237-2240`, `695-698` | 正确 | ✅ |
| exception return clear exclusive | `§B2.12.4` | `helper-a64.c:633` | 正确 | ✅ |
| reservation granule | 4-512 words | `3307-3312` 仅记录地址，不记范围 | 不符规范范围 | ❌ |
| exclusive success condition | 依赖 monitor，不只是值相等 | 地址+旧值+CMPXCHG | 近似 | ⚠️ |
| ABA / same-value interference | 真实硬件可能 fail | QEMU 可能 success | 不真实 | ⚠️ |
| LDXP/STXP 64-bit pair 成功原子更新 | 规范要求 | `3371-3386` 具备 | 基本正确 | ✅ |
| LSE RMW 原子性 | 单条 atomic RMW | `4062-4113` | 正确 | ✅ |
| CAS/CASP 原子性 | 单条 atomic CAS/CASP | `3402-3478`, `3599-3618` | 正确 | ✅ |
| LSE Acquire/Release 变体 | 应区分 A/L/AL | `4079-4083`, `4144-4147` 统一 full barrier | 偏强 | ⚠️ |
| FEAT_LSE128 | 支持 128-bit atomics | `842-845`, `4157-4162` | 已支持 | ✅ |
| host 不支持大原子时 | 应保持 guest 原子性 | serial fallback + exclusive context | 语义正确但更强 | ⚠️ |
| RR 模式 | 真实硬件并非全局串行 | RR 串行执行 vCPU | 过强 | ⚠️ |

---

## 11. 对 Guest 软件的影响

### 11.1 对内核 / 驱动 / 锁实现

对 Linux 内核、自旋锁、RCU、常见驱动屏障序列而言，QEMU 的实现通常 **足够正确甚至偏保守**：

- `LDAXR/STLXR`、`LDAR/STLR`、`CAS/CASP` 都不会比规范更弱；
- DMB/DSB/ISB 序列通常能得到正确的功能效果；
- TLBI / system register / exception return 序列大多因为 helper 同步执行而“看起来很好用”。

因此：

- **普通 guest OS 往往不会在 QEMU 上因为本专题实现而直接功能错误**。

### 11.2 对弱内存 litmus / 并发验证

这是 QEMU 最大的不真实区：

1. **很多 ARM 真实硬件允许的弱序结果，在 QEMU 中消失**；
2. `LDAPR` 与 LSE A/L/AL 变体被过度同步化；
3. RR 模式甚至把 SMP 竞争窗口压成全局串行；
4. exclusive failure/success 频率与真实硬件明显不同。

所以：

- **不能把 QEMU TCG 当成 ARM memory model litmus oracle**；
- 用它跑并发 bug 复现时，若在 QEMU 上“稳定不复现”，不代表真实 ARM 硬件也不会复现。

### 11.3 对 lock-free / wait-free 算法

需要特别警惕的点：

- reservation granule 未建模；
- ABA / 同值写回下 `STXR` 可能“虚假成功”；
- 虚假失败又偏少。

这会影响：

- 重试次数统计；
- backoff 调参；
- 一些故意探测 reservation granule / exclusive fairness 的测试；
- 把 QEMU 行为误认为“真实 ARM monitor 特性”的分析。

### 11.4 最终结论

**一句话总结：**

> QEMU 11.0.50 对 ARM64 内存模型/屏障/原子操作的实现，整体是**功能上保守正确、并发上明显偏强、exclusive monitor 上近似化严重**。它足以运行大多数 guest，但不足以忠实复现 ARM DDI 0487 M.b Chapter B2 所允许的全部弱序与 monitor 细节。

如果要进一步提高真实性，优先级应是：

1. 为 `LDAPR` / LSE A/L/AL 区分 RCpc / RCsc / one-way barrier；
2. 把 DSB 与 DMB 在 IR 层分开，补维护操作 completion；
3. 为 exclusive monitor 建立“至少覆盖 reservation granule + interference clear”的模型；
4. 若目标是 memory-model 验证，再考虑引入可控的弱序/传播近似，而不是仅依赖 host 内存系统。
