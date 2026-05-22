# ARM64 SVE/SME 可扩展向量规范验证（DDI 0487 M.b C8/C9 对照 QEMU）

## 1. 概述

本文目标是把既有分析文档《`15-SVE-SME可扩展向量扩展深度分析.md`》与 ARM Architecture Reference Manual DDI 0487 M.b 中与 **SVE/SME 架构模型**直接相关的章节重新做一次交叉验证，重点不是逐条指令编码，而是：

- **SVE (§C8.1)**：向量长度（VL）、谓词寄存器、FFR、gather/scatter、first-fault / no-fault 语义。
- **SME (§C9.1)**：Streaming SVE 模式、ZA 矩阵存储、tile 视图、SVL 与 VL 的关系。
- **系统级控制（Part D + C5）**：`ZCR_EL1/EL2/EL3`、`SMCR_EL1/EL2/EL3`、`SVCR`、`CPACR_EL1.ZEN/SMEN`、`CPTR_EL2.TZ/TSM`、`CPTR_EL3.EZ/ESM`。

本文使用的证据分三类：

1. **规范侧**：
   - DDI 0487 §C8.1（About the SVE instructions）
   - DDI 0487 §C9.1（About the SME instructions）
   - DDI 0487 §C5.2.27（SVCR）
   - DDI 0487 §D22.2 / §D22.3（SME traps and state validity）
   - DDI 0487 §D24.2.35 / §D24.2.37 / §D24.2.38 / §D24.2.183 / §D24.2.184 / §D24.2.225 / §D24.2.226
2. **QEMU 源码侧**：
   - `target/arm/cpu.h`
   - `target/arm/helper.c`
   - `target/arm/cpu64.c`
   - `target/arm/internals.h`
   - `target/arm/tcg/translate-a64.c`
   - `target/arm/tcg/translate-a64.h`
   - `target/arm/tcg/translate-sve.c`
   - `target/arm/tcg/sve_helper.c`
   - `target/arm/tcg/translate-sme.c`
   - `target/arm/tcg/sme_helper.c`
   - `target/arm/tcg/hflags.c`
3. **待核对既有文档**：
   - `darren/arm64/15-SVE-SME可扩展向量扩展深度分析.md`

一个总的结论先给出：**文档 15 对 QEMU 的寄存器布局、ZCR/SMCR 约束关系、SM 切换时状态重置、ZA 的基本布局判断大体正确；但在“流模式下 SVE 指令由谁负责 trap”以及“陷入控制与架构状态有效性相互独立”这两点上，系统级语义写得不够完整，且部分源码路径已经变为 `target/arm/tcg/*`。**

---

## 2. SVE 向量长度模型验证（§C8.1）

### 2.1 架构规则：VL 是实现定义，但程序应 VL-agnostic

按照 DDI 0487 §C8.1 以及 `ZCR_ELx` 的系统寄存器定义（§D24.2.225/226 等），SVE 的核心不是“固定 128/256/512 位向量”，而是：

- **VL（Vector Length）取值范围为 128–2048 bits**，步长 128 bits。
- 软件请求值通过 `ZCR_ELx.LEN` 表示，含义是 **(LEN+1) × 128 bits**。
- 实现可以只支持一个子集，但**必须支持所有不大于自身最大 VL 的 2 的幂长度**。
- 真正生效的是 **Effective Non-streaming SVE vector length**，它要同时受：
  - 当前 EL 自己的 `ZCR_ELx`
  - 更高 EL 的 `ZCR_ELy`
  - 硬件实际支持长度集合
  的共同约束。

这也是 SVE 的“向量长度无关（VLA / VL-agnostic）编程模型”基础：**程序不应假设 VL 是固定常数，而应通过谓词、元素计数和按 VL 比例扩展的循环来写代码。**

### 2.2 QEMU 的 SVE 长度存储与规格上限一致

QEMU 在 `target/arm/cpu.h:168-177` 定义：

```c
#define ARM_MAX_VQ 16

typedef struct ARMVectorReg {
    uint64_t d[2 * ARM_MAX_VQ] QEMU_ALIGNED(16);
} ARMVectorReg;
```

这里：

- `VQ = VL / 128`。
- `ARM_MAX_VQ = 16`，对应最大 VL=2048 bits。
- 每个 `ARMVectorReg` 可容纳 `2 * 16 = 32` 个 64-bit 字，即 256 字节，即 2048 位。

这与 DDI 0487 对 SVE 架构最大 VL 的定义**完全一致**。因此文档 15 关于 “QEMU 用固定最大 2048-bit 物理存储承载可变 VL 逻辑视图” 的理解是对的。

### 2.3 QEMU 用 `sve_vq.map` 表示“实现支持的长度集合”

在 `target/arm/cpu64.c:63-261`，QEMU 用 `cpu->sve_vq.map`、`cpu->sve_vq.supported`、`cpu->sve_max_vq` 来处理 SVE 长度能力与用户配置；在 `target/arm/internals.h:1828-1830` 中又定义了：

```c
static inline int arm_max_vq(ARMCPU *cpu)
{
    return MAX(cpu->sve_max_vq, cpu->sme_max_vq);
}
```

这说明 QEMU 把“CPU 支持哪些 VL”（`supported` / `map`）与“当前 EL 真正看到的有效 VL”（由 `ZCR_ELx` 再约束）分开建模；这与 DDI 0487 §D24.2.225/226 的系统寄存器语义一致。

### 2.4 QEMU 的有效 VL 计算与 `ZCR_ELx` 分层约束一致

`target/arm/helper.c:4695-4736`：

```c
uint32_t sve_vqm1_for_el_sm(CPUARMState *env, int el, bool sm)
{
    uint64_t *cr = env->vfp.zcr_el;
    uint32_t map = cpu->sve_vq.map;
    uint32_t len = ARM_MAX_VQ - 1;
    ...
    if (el <= 1) len = MIN(len, 0xf & (uint32_t)cr[1]);
    if (el <= 2 && arm_is_el2_enabled(env)) len = MIN(len, 0xf & (uint32_t)cr[2]);
    if (arm_feature(env, ARM_FEATURE_EL3)) len = MIN(len, 0xf & (uint32_t)cr[3]);
    map &= MAKE_64BIT_MASK(0, len + 1);
    if (map != 0) {
        return 31 - clz32(map);
    }
    ...
}
```

这里有三个关键点：

- **先取本机支持集合 `map`**。
- **再对 `LEN` 做 EL1/EL2/EL3 逐级 `MIN` 约束**。
- **最终不是简单返回请求值，而是返回不超过请求上限的最高可实现长度。**

这与 DDI 0487 §D24.2.225（`ZCR_EL1`）和 §D24.2.226（`ZCR_EL2`）描述一致。比文档 15 更精确的说法是：**QEMU 先对请求上限做分层 min，再从支持位图中选出不超过该上限的最高实现值。**

### 2.5 `ZCR_ELx` 写入后的状态收缩：QEMU 与架构“不可访问部分失效”一致

`target/arm/helper.c:4738-4756` 的 `zcr_write()` 在长度缩小时调用 `aarch64_sve_narrow_vq()`；后者位于 `helper.c:10029-10053`：

- 清零 32 个 `zregs[]` 中超出新 VQ 的高位。
- 清零 17 个谓词寄存器（P0-P15 + FFR）中超出新 VQ 的高位。

QEMU 实现：

```c
memset(&env->vfp.zregs[i].d[2 * vq], 0, 16 * (ARM_MAX_VQ - vq));
...
env->vfp.pregs[i].p[j] &= pmask;
```

这与规范精神一致：当有效 VL 下降，原本高于新 VL 的部分已经不再架构可见，软件不可再依赖它们。QEMU 采用“**主动清零不可访问区**”策略，便于后续执行保持确定性。

### 2.6 EL 切换时的 VL 变化：QEMU 捕捉到了“异常前后语境不同”的问题

`target/arm/helper.c:10073-10128` 的 `aarch64_sve_change_el()` 在异常进入/返回时被调用（`helper.c:9214`、`target/arm/tcg/helper-a64.c:758`）。逻辑是：

- 比较 old EL / new EL 是否都是 AArch64。
- 若 `PSTATE.SM=1` 且发生 AArch64↔AArch32 切换，则 `ResetSVEState`。
- 否则重新计算 old/new EL 对应的有效 VQ，若新值更小则收缩。

这与 DDI 0487 对异常层级和状态切换后“可访问向量长度可能变化”的系统语义相符。文档 15 对这部分判断正确，而且抓到了 QEMU 不仅在写 `ZCR_ELx` 时收缩，也在 **EL 切换** 时收缩，这是分析中的一个亮点。

### 2.7 一个容易忽视的细节：SME-only CPU 的非流模式 VL 回退到 128 bits

`helper.c:4705-4711` 有一个很重要的 QEMU 注释：

```c
/* SME-only CPU not in streaming mode: effective VL is 128 bits */
return 0;
```

`return 0` 表示 `VQ-1 = 0`，即 VL=128 bits。这个分支说明：如果 CPU 只有 SME 能力而没有普通 SVE，且当前又不在流模式中，那么 QEMU 让普通 SVE 视角的有效 VL 回到 128 bits 基线。

文档 15 没特别强调这个边界情况。它不影响主线结论，但如果未来要补成“规范-实现全景图”，**应把 SME-only / non-streaming 这个特例补进去**。

**本节结论：** 文档 15 对 SVE 长度模型的主体描述是 **✓ 正确** 的；需要补充的只是“不是简单 min，而是 min 上限 + supported map 选顶值”以及 “SME-only 非流模式 128b 回退”这两个实现细节。

---

## 3. SVE 谓词寄存器验证

### 3.1 架构对象：P0-P15 + FFR

SVE 的第二个核心不是 Z 寄存器，而是谓词寄存器。按照 DDI 0487 §C8.1，SVE 的向量运算天然受谓词控制：

- **P0-P15**：通用谓词寄存器。
- **FFR（First Fault Register）**：记录 first-fault / no-fault 类加载之后，从哪个元素起不再“已知有效”。

谓词不是“每字节一字节的掩码”，而是**按元素粒度生效**。因此同一个物理谓词寄存器在 `.B/.H/.S/.D` 不同元素宽度下，解释方式不同；这正是 VL-agnostic 编程的另一半。

### 3.2 QEMU 对 P 寄存器和 FFR 的布局与规范一致

`target/arm/cpu.h:175-177, 677-680`：

```c
typedef struct ARMPredicateReg {
    uint64_t p[DIV_ROUND_UP(2 * ARM_MAX_VQ, 8)] QEMU_ALIGNED(16);
} ARMPredicateReg;

#define FFR_PRED_NUM 16
ARMPredicateReg pregs[17];
```

即：

- `pregs[0..15]` = P0..P15
- `pregs[16]` = FFR

这与文档 15 的描述相符，而且这种“把 FFR 当作第 17 个 predicate register 存储”的做法，确实让很多 helper 可以统一处理普通谓词和 FFR。

### 3.3 QEMU 的谓词逻辑实现支持 governing / merging / zeroing 模型

DDI 0487 §C8.1 虽不是逐条说明每条指令，但 SVE 的总模型明确以 **predication** 为中心。QEMU 在 `target/arm/tcg/sve_helper.c` 与 `translate-sve.c` 中把这个模型拆成几种常见形态：

1. **governing predicate**：谁决定哪些元素是 active。
2. **merging predication**：inactive 元素保留旧值。
3. **zeroing predication**：inactive 元素写零。

证据：

- `translate-sve.c:719-772` 有 `SEL` 路径，注释直接写到 “active elements from Zn, inactive from Zm”。
- `translate-sve.c:944` 的注释明确写出“Copy Zn into Zd, storing zeros into inactive elements”。
- `sve_helper.c:153` 的 `DO_SEL`：

```c
#define DO_SEL(N, M, G)  (((N) & (G)) | ((M) & ~(G)))
```

这正是“g 选通 active/inactive 两路来源”的典型谓词选择逻辑。

因此，文档 15 虽未系统区分 governing / merging / zeroing 三个术语，但其对“谓词寄存器控制元素级生效范围”的主判断是正确的；本文只是把术语对齐到规范语言。

### 3.4 `PTRUE / PFALSE / SETFFR` 验证了谓词初始化与 FFR 复用路径

`target/arm/tcg/translate-sve.c:1652-1753` 用 `do_predset()` 统一实现 `PTRUE`、`PFALSE`、`SETFFR`：

- `PTRUE`：为相应 element size 建立 canonical true pattern。
- `PFALSE`：写空谓词。
- `SETFFR`：把 `RD` 固定解释为 `FFR_PRED_NUM=16`，相当于把 FFR 设为全有效。

这说明 QEMU **没有为 FFR 建专用存储格式**，而是复用了 predicate register 全套基础设施；这与 SVE 中“FFR 在行为上像特殊谓词寄存器”的架构理解一致。

### 3.5 `RDFFR/WRFFR` 被标记为 non-streaming only

`translate-sve.c:1749-1770`：

```c
TRANS_FEAT_NONSTREAMING(SETFFR, aa64_sve, ...)
TRANS_FEAT_NONSTREAMING(RDFFR, aa64_sve, ...)
TRANS_FEAT_NONSTREAMING(WRFFR, aa64_sve, ...)
```

这很重要：它说明在 QEMU 里，FFR 相关操作被视为**普通 SVE 非流模式**语义的一部分，而不是 SME/Streaming SVE 通用状态的一部分。这个实现和 DDI 0487 的系统级规则是兼容的：在流模式下，能否执行某些 SVE 指令取决于当前是否允许 non-streaming 指令路径以及是否启用 FA64。

文档 15 对 FFR 的存储位置描述正确，但没有强调 “**FFR 指令路径是 non-streaming SVE 语义**”，建议补充。

### 3.6 谓词测试 NZCV：QEMU 直接对应架构伪码风格

`target/arm/tcg/sve_helper.c:45-117` 中 `sve_predtest1()` / `sve_predtest()` 通过 `iter_predtest_fwd()` 计算 N/Z/C。注释写得非常直接：

- N：first active element 是否为真。
- Z：是否没有任何 active+true 元素。
- C：last active element 是否为假。

这与 ARM `PredTest` 伪码思路一致。文档 15 把 `predtest` 归纳为“扫描谓词设置 NZCV 标志”，这个判断是 **✓ 正确** 的。

**本节结论：** 文档 15 对谓词寄存器、FFR 存储和 predicate helper 的基础分析是 **✓ 正确** 的；建议增加规范术语映射（governing/merging/zeroing）以及 “FFR 属于 non-streaming SVE 指令路径” 的说明。

---

## 4. SVE 内存访问验证：Gather/Scatter、First-fault、No-fault

### 4.1 规范关注点：不是“是否有这些指令”，而是“fault 之后状态如何定义”

DDI 0487 §C8.1 把 SVE 内存访问能力概括为：

- 支持 **gather/scatter** 这种按向量元素生成地址的访问模式。
- 支持 **first-faulting** 与 **non-faulting** 负载模型。
- 通过 **FFR** 把“从哪个元素起结果不再可保证有效”显式暴露给软件。

这类设计是 SVE 区别于传统 NEON 的关键之一：它不是仅仅“向量 load/store 更宽”，而是把**矢量化访存容错语义**带进 ISA。

### 4.2 QEMU 的 gather/scatter 翻译路径是完备的

`target/arm/tcg/translate-sve.c:5824-6515` 明确是 gather/scatter 区域，且通过多个 helper 表完成组合选择：

- `gather_load_fn32[...][ff][xs][u][msz]`
- `gather_load_fn64[...][ff][xs][u][msz]`
- `gather_load_fn128[...]`
- `scatter_store_fn32[...]`
- `scatter_store_fn64[...]`
- `scatter_store_fn128[...]`

其中 `ff` 这一维直接区分普通 load 与 first-fault load；`mte` / `be` / `xs` / `u` / `msz` 等维度则对应内存标签、大小端、offset 宽度、符号扩展与访存宽度。

这说明 QEMU 在翻译层已经把 **gather/scatter 作为一级语义对象**，而不是退化成普通 load/store 的语法糖。文档 15 对这一点判断正确。

### 4.3 first-fault 与 no-fault 的核心差异，QEMU 在 helper 中显式建模

`target/arm/tcg/sve_helper.c:6525-6727` 有两个关键点：

1. `record_fault()`：
   - 一旦某元素 fault，**FFR 从该元素开始清零**。
2. `sve_ldnfff1_r()`：
   - 统一处理 contiguous `LDFF1*` 与 `LDNF1*`。
   - 通过 `fault == FAULT_FIRST / FAULT_NO` 分流语义。

`record_fault()` 的注释非常关键：

- **All bits in FFR from I are cleared**。
- faulting 元素之后的结果是 **CONSTRAINED UNPREDICTABLE**。
- QEMU 选择 **MERGE** 策略，即后续元素保持原样（或者在某些路径保持未重写部分）。

这与 ARM 对 first-faulting loads 的设计精神一致：软件通过 FFR 判断“哪里开始失效”，而不是假设 fault 后所有元素都无条件清零。

### 4.4 QEMU 对 no-fault load 的处理体现了“允许返回 UNKNOWN/FAULT”

`sve_helper.c:6646-6660` 注释写得很清楚：

- `MemSingleNF`（no-fault）对 Device memory 不应该真的打总线，而应返回 `(UNKNOWN, FAULT)`。
- QEMU 因为拿不到完整设备属性，只能对 **MMIO** 做近似处理。
- 规范允许 NF load “for any reason” 失败，因此这种近似在大多数情况下是允许的。

这说明 QEMU 的重点是守住 fault-tolerant contract，而不是逐点还原所有内存类型细节。文档 15 如需补充，一句即可：**NF load 的实现重点是遵守“不像普通 load 那样同步终止向量遍历”的架构契约。**

### 4.5 FFR 更新语义：QEMU 与规范对齐

结合 `SETFFR/RDFFR/WRFFR` 和 `record_fault()` 可以得到 QEMU 的 FFR 语义闭环：

1. `SETFFR` 把 FFR 置为“全有效”。
2. `LDFF1*/LDNF1*` 在发生 fault 时，从 fault 元素起清零 FFR。
3. 软件可用 `RDFFR` 取回 FFR，再决定循环继续位置或补救策略。

这正是 DDI 0487 §C8.1 中“first-faulting + FFR”组合的核心软件接口。文档 15 对“FFR 用于 first-fault 和 no-fault 加载”这个主判断是 **✓ 正确** 的。

### 4.6 一个容易混淆的点：FFR 更新不是 scatter/store 路径的一部分

从 QEMU 实现看：

- `FFR` 更新主要出现在 `LDFF1/LDNF1` 这类 fault-tolerant load 路径。
- `scatter store` 路径主要关注 predication、MMIO/watchpoint/MTE、跨页处理，不承担 FFR 状态更新语义。

因此如果后续要扩写文档 15，建议把：

- **gather/scatter**
- **first-fault / no-fault**
- **FFR 更新**

分成三个小主题，而不是写成一个笼统“内存操作”段落。

**本节结论：** 文档 15 对 gather/scatter、first-fault、FFR 的方向判断是 **✓ 正确** 的；建议补充 QEMU 对 `(UNKNOWN, FAULT)` 的保守实现方式，以及“FFR 主要跟 fault-tolerant load 而不是 scatter/store 绑定”。

---

## 5. SME 流模式验证（§C9.1）

### 5.1 架构规则：SME 不是“又一套向量指令”，而是“流模式 + ZA 状态”

DDI 0487 §C9.1 与 §C5.2.27（`SVCR`）一起看，SME 的最关键变化不是“多了几条矩阵乘法指令”，而是引入了两类新状态：

- **PSTATE.SM / SVCR.SM**：是否进入 Streaming SVE mode。
- **PSTATE.ZA / SVCR.ZA**：ZA 存储是否有效、可访问。

进入流模式后：

- SVE 指令看到的有效向量长度不再是普通 VL，而是 **SVL（Streaming Vector Length）**。
- 某些普通 A64/SVE 指令在流模式下可能变成非法，是否继续合法受 `SMCR_ELx.FA64` 控制。
- ZA 存储与普通 Z/P/FFR 状态并不是一回事。

### 5.2 `SVCR` 的规范语义与 QEMU 一致

DDI 0487 §C5.2.27 明确：

- `SVCR.SM` 控制 Streaming SVE mode。
- `SVCR.ZA` 控制 ZA（及若实现则 ZT0）存储是否有效。
- 当 `ZA` 从 0→1 时，**ZA 存储必须清零**。
- `ZA` 改变**不应影响 SVE Z/P/FPSR**。

QEMU 的对应实现位于：

- `target/arm/cpu.h:1583-1590`
- `target/arm/helper.c:4832-4860`

```c
FIELD(SVCR, SM, 0, 1)
FIELD(SVCR, ZA, 1, 1)
...
if (change & R_SVCR_SM_MASK) {
    arm_reset_sve_state(env);
}
if (change & new & R_SVCR_ZA_MASK) {
    memset(&env->za_state, 0, sizeof(env->za_state));
}
```

可以看到 QEMU 明确区分：

- **SM 改变** → reset SVE state（Z/P/FFR/FPSR）
- **ZA 0→1** → zero ZA storage
- 两者不是同一个动作

这与 DDI 0487 §C5.2.27 完全对齐。

### 5.3 Streaming mode 切换时，QEMU 会 reset 普通 SVE 状态

`arm_reset_sve_state()`（`helper.c:4824-4830`）清零：

- `zregs[32]`
- `pregs[17]`（含 FFR）
- 并重置 `FPSR`

这意味着 QEMU 在 `SM` 位变化时，明确把 **非流模式 SVE 寄存器视图**重置掉。这与文档 15 的表述一致：

| 项 | 文档 15 结论 | QEMU 结果 |
|---|---|---|
| SM 0→1 | Z/P/FFR 清零 | 是 |
| SM 1→0 | Z/P/FFR 清零 | 是 |
| ZA 是否连带清零 | 否 | 否（仅 ZA 0→1 时清零） |

所以文档 15 在这部分是 **✓ 正确** 的。

### 5.4 SVL 与 VL：QEMU 区分普通 SVE 与 Streaming SVE 的长度来源

`helper.c:4695-4704`：

- 非流模式：`cr = env->vfp.zcr_el`, `map = cpu->sve_vq.map`
- 流模式：`cr = env->vfp.smcr_el`, `map = cpu->sme_vq.map`

这说明 QEMU 明确区分：

- **VL**：普通 SVE 向量长度，由 `ZCR_ELx + sve_vq.map` 决定。
- **SVL**：Streaming SVE 向量长度，由 `SMCR_ELx + sme_vq.map` 决定。

`target/arm/tcg/hflags.c:282-299` 进一步把这个事实编码进翻译块：

- `SVL` 单独记录进 TB flag。
- 若 `PSTATE.SM=1`，则 `VL` 也被覆写成 `SVL` 用于当前翻译块。

这与 DDI 0487 §C5.2.27 和 §D24.2.183/184 对 “流模式中 VL 取 Effective Streaming SVE vector length” 的定义一致。

### 5.5 流模式下并不是“所有 SVE 指令都自然合法”

DDI 0487 §D22.2 明确指出：在 Streaming SVE mode 下，**哪些指令合法、哪些非法并产生 SME syndrome trap**，与 `PSTATE.SM` / `PSTATE.ZA` 和 trap 控制共同决定。

QEMU 侧对应在 `target/arm/tcg/translate-a64.c:1445-1589`：

- `nonstreaming_check()`：若当前 TB 标记 `sme_trap_nonstreaming` 且当前指令被标记为 `is_nonstreaming`，则抛出 `SME_ET_Streaming`。
- `sme_enabled_check_with_svcr()`：
  - 需要 `SM` 但 `pstate_sm=0` → `SME_ET_NotStreaming`
  - 需要 `ZA` 但 `pstate_za=0` → `SME_ET_InactiveZA`

这说明 QEMU 对流模式采用分层检查：先判定 SME 是否可访问，再检查 `SM` / `ZA` 是否 active，最后处理 non-streaming legality；这与 DDI 0487 §D22.2/§D22.3 一致。

### 5.6 `SMSTART / SMSTOP / MSR SVCR` 在 QEMU 中的入口清晰

`target/arm/tcg/translate-a64.c:2487-2503` 的 `trans_MSR_i_SVCR()` 是 `MSR SVCR*` 指令翻译入口；实际状态更新由 `target/arm/tcg/sme_helper.c:42-45` 的 `helper_set_svcr()` 转到 `aarch64_set_svcr()` 完成。

因此文档 15 若要补充“控制路径”，最准确的链路应写成：

```text
MSR SVCR / SMSTART / SMSTOP
  -> translate-a64.c:trans_MSR_i_SVCR()
  -> tcg/sme_helper.c:helper_set_svcr()
  -> helper.c:aarch64_set_svcr()
  -> 必要时 reset SVE state / zero ZA / rebuild hflags
```

**本节结论：** 文档 15 对 SME 流模式、SVL/VL 区分、SM 切换副作用的主体判断是 **✓ 正确** 的；本文补充了 trap 判定和翻译路径两个系统级细节。

---

## 6. SME ZA 矩阵存储验证

### 6.1 架构规则：ZA 是独立于 Z/P 的二维存储

DDI 0487 §C9.1 与 §C5.2.27 对 ZA 的理解可以归纳为：

- ZA 不是“额外几条向量寄存器”，而是**独立的大块矩阵状态**。
- 其可访问性由 `PSTATE.ZA` 决定。
- 当 `PSTATE.ZA` 从 0→1 时，已实现部分必须清零。
- 若 ZA 不可访问，访问 ZA 的 SME 指令要 trap，而**这并不等价于 SM 也必须失效**。

### 6.2 QEMU 的 ZA 存储大小与“最大 256×256 byte 方阵”一致

`target/arm/cpu.h:725-754`：

```c
struct {
    uint64_t zt0[512 / 64] QEMU_ALIGNED(16);
    ARMVectorReg za[ARM_MAX_VQ * 16];
} za_state;
```

由于：

- `ARM_MAX_VQ = 16`
- 每个 `ARMVectorReg = 256 bytes`
- `za[]` 有 `16 * 16 = 256` 行

所以最大 ZA 存储就是：

- **256 行 × 256 字节 = 64KB**

而 `cpu.h` 注释又明确说明：若 `SVL` 低于架构最大值，则来宾可见范围受限为左下角的 **SVL × SVL 方阵**。这与文档 15 写的“ZA 是一个 SVL×SVL 字节矩阵”高度一致。

### 6.3 QEMU 对 tile 映射的注释与实现，非常接近规范文字

`cpu.h:741-747` 已经把 tile 与 ZA array 的关系写得很清楚：

- 对于元素大小 `esz`，tile T 的第 N 行（horizontal slice）在 `ZA[T + N * esz]`。
- 这意味着 tile 的各行在整个 ZA 存储中是**条带式分布**，不是每个 tile 连续占一块内存。

而 `target/arm/tcg/sme_helper.c:73-103` 进一步用 `tile_vslice_index()` / `tile_vslice_offset()` 把“垂直 slice 为什么也能统一映射”为实现公式。

因此：

- 文档 15 中 “`Tile T` 第 N 行 = `ZA[T + N * esz]`” 的结论 **✓ 正确**。
- 且 QEMU 的确把这个条带化布局贯彻到了 `MOVA`、`ZERO`、load/store tile slice、outer product helpers 里。

### 6.4 `PSTATE.ZA` 的 enable/disable 语义：QEMU 与 `SVCR.ZA` 完全一致

DDI 0487 §C5.2.27：

- `ZA=0`：ZA/ZT0 无效且不可访问，相关指令 trap。
- `ZA=1`：ZA/ZT0 有效且可访问。
- `ZA` 从 0→1 时清零存储。
- 改变 `ZA` 不影响普通 SVE Z/P/FPSR。

QEMU：

- `helper.c:4853-4855` 只在 `change & new & R_SVCR_ZA_MASK` 时清零 ZA。
- 不会因为关闭 ZA 而立刻擦除整个 `za_state`；关闭时只是让它**变得不可访问**。

这与规范一致，也印证了 DDI 0487 §D22.3 所说的：**trap 控制、state validity、state contents 是分离概念。**

### 6.5 `ZERO`、`MOVA`、`ADDHA/ADDVA`、`FMOPA/SMOPA` 形成了 ZA 操作闭环

`target/arm/tcg/translate-sme.c` 提供了 `ZERO`、`MOVA`、`ADDHA/ADDVA`、`do_outprod*()` 等入口；`target/arm/tcg/sme_helper.c` 则提供 `helper_sme_zero()`、`sme_mova_*`、`sme_addha_*`、`sme_addva_*`、`sme_fmopa_*`、`sme_smopa_*` 等执行 helper。特别是 `do_outprod*()` 会取目标 tile、源 `Zn/Zm` 和 governing predicates `Pn/Pm`，再由 helper 完成按 row/col 谓词控制的 outer product accumulate。这与 DDI 0487 §C9.1 的 SME 矩阵执行模型一致。

### 6.7 对文档 15 的评价

文档 15 的 ZA 部分有两个优点：

1. 抓到了 **ZA 是 SVL×SVL 字节矩阵** 这个总体认识。
2. 抓到了 **tile 行条带映射** 这个 QEMU 实现重点。

但如果继续完善，建议再补两句：

- ZA 的“enable/disable”与 trap 控制独立；关闭 ZA 不等于清空内容，只是变为不可访问。
- `outer product` 在 QEMU 中不是抽象概念，而是有清晰的 `translate-sme.c -> sme_helper.c` 调用链。

**本节结论：** 文档 15 对 ZA 架构与 QEMU 存储模型的主体理解是 **✓ 正确** 的。

---

## 7. SVE/SME Trap 控制验证

### 7.1 `CPACR_EL1.ZEN/SMEN`、`CPTR_EL2.TZ/TSM/ ZEN/SMEN`、`CPTR_EL3.EZ/ESM`

系统级最容易写错的点，恰恰不在指令本身，而在**谁负责 trap 哪一类指令**。

从 DDI 0487：

- §D24.2.35：`CPACR_EL1` 控制 EL0/EL1 对 SVE、SME、FP 的访问。
- §D24.2.37：`CPTR_EL2` 控制 trap 到 EL2。非 E2H 格式下有 `TZ` / `TSM`，E2H 格式下则有 `ZEN` / `SMEN` 两位域。
- §D24.2.38：`CPTR_EL3` 用 `EZ` / `ESM` 控制 trap 到 EL3。

QEMU 在 `target/arm/internals.h:131-160` 把位定义写得非常清楚：

- `CPACR_EL1.ZEN[17:16]`
- `CPACR_EL1.SMEN[25:24]`
- `CPTR_EL2.TZ`, `CPTR_EL2.TSM`, `CPTR_EL2.ZEN`, `CPTR_EL2.SMEN`
- `CPTR_EL3.EZ`, `CPTR_EL3.ESM`

### 7.2 QEMU 的 SVE trap 判定路径是规范直译

`target/arm/helper.c:4598-4641` 的 `sve_exception_el()`：

1. EL0/EL1 先看 `CPACR_EL1.ZEN`
2. 再看 EL2：
   - `HCR_EL2.E2H=1` 时用 `CPTR_EL2.ZEN`
   - 否则用 `CPTR_EL2.TZ`
3. 最后看 EL3：`CPTR_EL3.EZ`（负逻辑）

`sme_exception_el()`（`helper.c:4647-4690`）则完全平行：

1. EL0/EL1 看 `CPACR_EL1.SMEN`
2. EL2：
   - E2H 用 `CPTR_EL2.SMEN`
   - 否则用 `CPTR_EL2.TSM`
3. EL3 看 `CPTR_EL3.ESM`

这两段实现几乎就是系统寄存器章节的直接落地，文档 15 在这两段函数级别的分析是 **✓ 正确** 的。

### 7.3 关键勘误：流模式下，SVE 指令是否 trap 不是由 ZEN/TZ/EZ 决定

这点是全文最重要的勘误。

DDI 0487 §D22.2 明确指出：

- **当 PE 处于 Streaming SVE mode，或 FEAT_SVE 未实现时**，
  - `CPACR_EL1.SMEN`
  - `CPTR_EL2.{SMEN,TSM}`
  - `CPTR_EL3.ESM`
  才是控制 **SVE 指令** trap 的那组位；
  - `ZEN/TZ/EZ` 此时**不负责让 SVE 指令 trap**。
- **只有在非流模式且 FEAT_SVE 实现时**，`ZEN/TZ/EZ` 才控制 SVE 指令 trap。

QEMU 的 `translate-a64.c:1508-1538` 的 `sve_access_check()` 精确体现了这一点：

- 若实现了 SME：
  - `pstate_sm=1` 时，SVE 指令先走 `sme_enabled_check()` 路径；
  - 若未实现普通 SVE，但实现了 SME，也走 `sme_sm_enabled_check()` 逻辑；
- 非这些情形才落到普通 `sve_excp_el` / `syn_sve_access_trap()`。

换言之：

> **QEMU 并不是“凡 SVE 指令都查 ZEN/TZ/EZ”，而是会先根据是否处于流模式改道到 SME trap 域。**

文档 15 在 trap 控制表格上容易让人读成“ZEN 永远控制 SVE，SMEN 永远控制 SME”。这个表述对**非流模式**成立，但对**流模式**不完整，必须修正。

### 7.4 关键勘误：trap 控制与架构状态有效性是独立的

DDI 0487 §D22.3 的原意非常重要：

- 指令是否被 trap
- SME 状态（由 `PSTATE.SM` / `PSTATE.ZA` 决定）是否有效

这两件事是**独立**的。允许出现：

1. 指令 trap，状态无效
2. 指令 trap，状态仍有效
3. 指令不 trap，但状态无效（例如没进流模式、或 ZA 未启用）
4. 指令不 trap，状态有效

QEMU 的实现与此一致：

- `sme_exception_el()` 只回答“是否 trap 到 ELx”；
- `sme_enabled_check_with_svcr()` 另外再检查 `pstate_sm` / `pstate_za`；
- `aarch64_set_svcr()` 也不会因为打开/关闭 trap 控制而篡改 ZA 内容。

因此，文档 15 如果把 “能不能执行 SME 指令” 与 “SM/ZA 是否 active” 混成同一个条件集合，就是不够严谨的。更准确的结构应是：

```text
第一层：trap gate（CPACR/CPTR）
第二层：state validity（PSTATE.SM / PSTATE.ZA）
第三层：指令类别合法性（non-streaming / ZA required / ZT0 required / FA64）
```

### 7.5 `ZCR/SMCR` 层级约束也属于 trap/enable 大图的一部分

`ZCR_EL1` / `ZCR_EL2` / `ZCR_EL3` 控制非流模式下 VL 请求；
`SMCR_EL1` / `SMCR_EL2` / `SMCR_EL3` 控制流模式下 SVL 请求以及 `FA64`、`EZT0`。

QEMU 中：

- `helper.c:4759-4777` 注册 `ZCR_EL1/2/3`
- `helper.c:4907-4924` 注册 `SMCR_EL1/2/3`
- `helper.c:4868-4894` 的 `smcr_write()` 在收缩时也调用 `aarch64_sve_narrow_vq()`

这符合 DDI 0487 §D24.2.183/184/225/226 的层级结构。

### 7.6 `FA64` 与 non-streaming legality

`SMCR_EL1/EL2` 都有 `FA64` 位（§D24.2.183/184）；QEMU 对应实现：

- `cpu.h:1588-1590`：`FIELD(SMCR, FA64, 31, 1)`
- `target/arm/tcg/hflags.c:296-299`：若 `sm`，则 `SME_TRAP_NONSTREAMING = !sme_fa64(env, el)`
- `translate-a64.c:1445-1453`：nonstreaming_check 用这个 TB flag 决定是否抛 `SME_ET_Streaming`

因此 QEMU 对 FA64 的处理也是系统寄存器语义的直接映射。

**本节结论：** 文档 15 对 trap 函数实现本身是 **✓**；但对“流模式下 SVE trap 归属于 SME trap 域”以及“trap 与状态有效性独立”这两点需要明确勘误，结论为 **✗ 需修正**。

---

## 8. QEMU 实现验证汇总

把前文收束成一张实现清单：

- **长度能力与约束**：`cpu64.c:63-261`、`cpu64.c:339-395`、`internals.h:1828-1830` 负责 `sve_vq` / `sme_vq` 能力集合与最大 VQ。
- **谓词与 FFR**：`cpu.h:675-680` 定义 `pregs[17]`；`translate-sve.c:1652-1770` 和 `sve_helper.c:45-117` 完成 `PTRUE/PFALSE/SETFFR/RDFFR/WRFFR` 与 `PredTest`。
- **VL/SVL 切换**：`helper.c:4695-4756`、`helper.c:10029-10128` 处理 `ZCR/SMCR` 约束、收缩与 EL 切换。
- **流模式控制**：`cpu.h:1583-1590`、`translate-a64.c:2487-2503`、`tcg/sme_helper.c:42-45`、`helper.c:4832-4860` 处理 `SVCR.SM/ZA`、reset SVE state、zero ZA。
- **TB 语境**：`tcg/hflags.c:282-307` 设置 `SVL`、`PSTATE_SM`、`SME_TRAP_NONSTREAMING`、`PSTATE_ZA`、`ZT0EXC_EL`。
- **SVE fault-tolerant memory**：`translate-sve.c:5824-6515` 与 `sve_helper.c:6525-6727` 实现 gather/scatter、FFR 更新、NF/FF load。
- **ZA/tile/outer product**：`cpu.h:725-754`、`translate-sme.c:48-153,507-610`、`sme_helper.c:73-103,920-1210,1524-1616` 接通 ZA 存储、tile addressing 与 outer product accumulate。

**汇总结论**：QEMU 对本文关注的 VL/SVL、谓词/FFR、流模式、ZA 与 trap 语义都有可追溯实现，且与 DDI 0487 的架构模型基本一致。

---

## 9. 既有文档勘误（对 `15-SVE-SME可扩展向量扩展深度分析.md`）

| 条目 | 原文核心说法 | 结论 | 说明 |
|---|---|---|---|
| 1 | QEMU 用 `zregs[32]`、`pregs[17]`、`za_state.za[]` 存储 SVE/SME 状态 | ✓ | 与 `cpu.h` 完全一致。 |
| 2 | FFR 存在于 `pregs[16]` | ✓ | `cpu.h` 明确 `FFR_PRED_NUM 16`。 |
| 3 | `ZCR_EL1/2/3` 和 `SMCR_EL1/2/3` 共同约束 VL/SVL | ✓ | `helper.c:sve_vqm1_for_el_sm()` 完整实现了该层级。 |
| 4 | VL 缩小时 QEMU 会清理高位不可访问状态 | ✓ | `aarch64_sve_narrow_vq()` 明确清零 Z/P/FFR 高位。 |
| 5 | SM 切换时 Z/P/FFR 会 reset，而 ZA 不会因此自动清零 | ✓ | `aarch64_set_svcr()` 与 `arm_reset_sve_state()` 证实该点。 |
| 6 | SVL 与 VL 可以不同 | ✓ | 流模式走 `smcr_el[] + sme_vq.map`，非流模式走 `zcr_el[] + sve_vq.map`。 |
| 7 | “SVE trap 由 ZEN/TZ/EZ 控制，SME trap 由 SMEN/TSM/ESM 控制” | ✗ | 这只对**非流模式**可成立。按 DDI 0487 §D22.2，在 **Streaming SVE mode** 下，SVE 指令也归 `SMEN/TSM/ESM` 控制。 |
| 8 | “能否执行 SME 指令”基本等同于“SM/ZA 是否打开” | ✗ | DDI 0487 §D22.3 明确 trap 控制与状态有效性独立；QEMU 也把二者分层实现。 |
| 9 | `hflags.c`、`sve_helper.c`、`sme_helper.c` 属于 `target/arm/` 顶层 | ✗ | 现源码位于 `target/arm/tcg/` 子目录，应更新路径。 |
| 10 | ZT0 trap 是“检查 `SMCR_ELx.EZT0` 的独立路径” | △ | 方向对，但文档 15 写成伪代码式描述，建议补入 `hflags.c` 的 `ZT0EXC_EL` 和 `translate-sme.c:sme2_zt0_enabled_check()` 的真实路径。 |
| 11 | no-fault / first-fault 的实现重点在 FFR 更新 | ✓ | 但还应补充 QEMU 对 MMIO / Device memory 的保守近似实现。 |
| 12 | QEMU 完整实现 SME/SME2 仿真 | △ | 作为概括可以接受，但更严谨的说法应是“本文涉及的流模式、ZA、tile 与主要 outer product/helper 路径已实现并可验证”。 |

### 9.1 建议对文档 15 做的最小修订

如果只做最小改动，建议修改四处：

1. **trap 表格后追加一句**：
   > 在 Streaming SVE mode 下，SVE 指令的 trap 控制切换到 SME trap 域（`SMEN/TSM/ESM`），不再由 `ZEN/TZ/EZ` 控制。
2. **SME 状态部分追加一句**：
   > trap gate 与 `PSTATE.SM/PSTATE.ZA` 的状态有效性相互独立，二者不是同一层判断。
3. **源文件路径统一更新为当前树形**：
   - `target/arm/tcg/hflags.c`
   - `target/arm/tcg/sve_helper.c`
   - `target/arm/tcg/sme_helper.c`
   - `target/arm/tcg/translate-sve.c`
   - `target/arm/tcg/translate-sme.c`
4. **ZA/outer product 段补入真实调用链**：
   - `translate-sme.c:do_outprod*()`
   - `sme_helper.c:sme_fmopa_* / sme_smopa_* ...`

---

## 10. 总结

基于 DDI 0487 M.b 的 §C8.1、§C9.1 以及 Part D/C5 的系统寄存器与 trap 章节，对 QEMU 当前 SVE/SME 实现做交叉验证后，可以得到以下结论：

1. **SVE 向量长度模型**：
   - QEMU 用固定 2048-bit 最大物理寄存器承载可变 VL，符合架构最大值；
   - 有效 VL 由 `ZCR_ELx` 分层约束并结合支持位图选顶值，符合规范。
2. **谓词与 FFR**：
   - `P0-P15 + FFR` 的存储布局正确；
   - governing / merging / zeroing predication 在翻译层与 helper 层都有直接映射。
3. **SVE fault-tolerant memory model**：
   - gather/scatter、first-fault、no-fault、FFR 更新语义都能在 QEMU 中找到对应实现；
   - 对 MMIO / Device memory 的近似处理符合架构允许的保守实现空间。
4. **SME 流模式与 ZA**：
   - `SVCR.SM`、`SVCR.ZA`、`SMCR_ELx`、ZA tile 存储、outer product 路径都实现清晰；
   - SVL 与 VL 的区分、SM 切换时重置普通 SVE 状态、ZA 0→1 时清零，都与规范一致。
5. **系统级 trap 控制**：
   - 文档 15 最大的问题不是函数级分析错误，而是**系统语义少了两层**：
     - Streaming mode 下，SVE 指令 trap 切入 SME trap 域；
     - trap gate 与状态有效性是独立维度。

因此，对既有文档的总体评价是：

> **主体分析可信，可继续作为 QEMU SVE/SME 入门材料；但若要作为“规范对齐文档”，必须补齐流模式下 trap 控制归属、state validity 独立性，以及当前源码实际路径。**

