# ARM64 EL 切换时 TCG 翻译变化深度分析：hflags 位域全景、TB 键与链断裂、寄存器组切换与 SCTLR/HCR 行为效应

> 基于 QEMU 11.0.50 源码分析，涵盖 ARM64 异常级别（EL）切换时 TCG 翻译器的完整行为变化：
> CPUARMTBFlags 96 位布局（flags 32 位 + flags2 64 位）、TBFLAG_ANY/A64 共 59 个位域、
> rebuild_hflags_a64 构建流程（TBI→SVE→SCTLR→PAuth→BTI→UNPRIV→FGT→NV→MTE→GCS 十大阶段）、
> arm_get_tb_cpu_state TB 键生成、EL 切换时 TB 链断裂机制、异常入口/ERET 的 hflags 重建、
> SP_EL/ELR_EL/SPSR 寄存器组切换、DisasContext 初始化与 EL 相关指令行为差异、单步调试跨 EL 处理。

---

## 目录

1. [架构概述](#1-架构概述)
2. [CPUARMTBFlags 数据结构](#2-cpuarmtbflags-数据结构)
3. [TBFLAG 位域全景](#3-tbflag-位域全景)
4. [rebuild_hflags_a64 构建流程](#4-rebuild_hflags_a64-构建流程)
5. [arm_get_tb_cpu_state — TB 键生成](#5-arm_get_tb_cpu_state--tb-键生成)
6. [异常入口的状态切换与 hflags 重建](#6-异常入口的状态切换与-hflags-重建)
7. [ERET 的状态恢复与 hflags 重建](#7-eret-的状态恢复与-hflags-重建)
8. [TB 链断裂机制](#8-tb-链断裂机制)
9. [寄存器组切换](#9-寄存器组切换)
10. [DisasContext 初始化与 EL 解码](#10-disascontext-初始化与-el-解码)
11. [SCTLR 对翻译行为的影响](#11-sctlr-对翻译行为的影响)
12. [HCR_EL2 对翻译行为的影响](#12-hcr_el2-对翻译行为的影响)
13. [EL 相关指令行为差异](#13-el-相关指令行为差异)
14. [单步调试跨 EL 处理](#14-单步调试跨-el-处理)
15. [完整 EL 切换流程总结](#15-完整-el-切换流程总结)

---

## 1. 架构概述

ARM64 的 Exception Level（EL0-EL3）切换是系统运行中最频繁的状态转换。每次 EL 切换都会改变：

- **MMU 域**（翻译表基址、权限检查规则）
- **可访问的系统寄存器集合**
- **指令行为**（陷阱、重定向、权限）
- **PSTATE 标志位**（DAIF、PAN、TCO、SS 等）

TCG 翻译器通过 **hflags**（hardware flags）机制将这些状态编码为 TB（Translation Block）的键，确保不同 EL 下的代码使用不同的 TB 缓存。

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `cpu.h` | 185-188 | `CPUARMTBFlags` 结构体 |
| `cpu.h` | 2422-2531 | TBFLAG 位域定义（59 个字段） |
| `cpu.h` | 293-318 | 寄存器组（banked_spsr/sp_el/elr_el） |
| `hflags.c` | 240-504 | `rebuild_hflags_a64()` 完整实现 |
| `hflags.c` | 506-573 | `rebuild_hflags_internal/arm_rebuild_hflags` |
| `hflags.c` | 620-695 | `arm_get_tb_cpu_state()` TB 键生成 |
| `helper.c` | 9340-9421 | 异常入口状态切换 |
| `helper-a64.c` | 715-750 | ERET 状态恢复 |
| `translate-a64.c` | 535-564 | `gen_goto_tb()` TB 链接 |
| `translate-a64.c` | 10667-10750 | `aarch64_tr_init_disas_context()` |
| `internals.h` | 565-581 | `aarch64_save_sp/restore_sp` |

---

## 2. CPUARMTBFlags 数据结构

```c
// cpu.h:185-188
typedef struct CPUARMTBFlags {
    uint32_t flags;   // 共享位域（TBFLAG_ANY），32 位
    uint64_t flags2;  // 模式特定位域（A64/A32/M32），64 位
} CPUARMTBFlags;
```

**总计 96 位**的 TB 标志空间，按以下方式分配：

```
flags (32 bits):  TBFLAG_ANY — 所有模式共享
                  bit[0]  AARCH64_STATE
                  bit[1]  SS_ACTIVE
                  bit[2]  PSTATE__SS (非缓存)
                  bit[3]  BE_DATA
                  bit[4:7] MMUIDX
                  bit[8:9] FPEXC_EL
                  bit[10] ALIGN_MEM
                  bit[11] PSTATE__IL
                  bit[12] FGT_ACTIVE
                  bit[13] FGT_SVC

flags2 (64 bits): TBFLAG_A64 — AArch64 专用 (bit 0..43)
                  或 TBFLAG_A32/M32 — AArch32 专用
```

**hflags 缓存在 `env->hflags` 中**，大部分位域在 `rebuild_hflags_*` 时计算并缓存，少数标记为 "Not cached" 的字段在每次 `arm_get_tb_cpu_state()` 时动态填充。

---

## 3. TBFLAG 位域全景

### 3.1 TBFLAG_ANY（所有模式共享，flags 低 14 位）

| 位 | 字段名 | 位宽 | 缓存 | 含义 |
|----|--------|------|------|------|
| 0 | AARCH64_STATE | 1 | ✓ | 当前为 AArch64 模式 |
| 1 | SS_ACTIVE | 1 | ✓ | 软件单步激活 |
| 2 | PSTATE__SS | 1 | ✗ | PSTATE.SS 当前值 |
| 3 | BE_DATA | 1 | ✓ | 数据大端字节序 |
| 4:7 | MMUIDX | 4 | ✓ | MMU 索引（22 种 A-profile 值） |
| 8:9 | FPEXC_EL | 2 | ✓ | FP 异常目标 EL |
| 10 | ALIGN_MEM | 1 | ✓ | 内存对齐要求（SCTLR.A） |
| 11 | PSTATE__IL | 1 | ✓ | 非法执行状态 |
| 12 | FGT_ACTIVE | 1 | ✓ | 细粒度陷阱激活 |
| 13 | FGT_SVC | 1 | ✓ | SVC 陷阱到 EL2 |

### 3.2 TBFLAG_A64（AArch64 专用，flags2 bit 0..43）

| 位 | 字段名 | 位宽 | 缓存 | 含义 |
|----|--------|------|------|------|
| 0:1 | TBII | 2 | ✓ | Top Byte Ignore for Insn（TCR.TBI） |
| 2:3 | SVEEXC_EL | 2 | ✓ | SVE 异常目标 EL |
| 4:7 | VL | 4 | ✓ | 当前向量长度 (NVL/SVL) |
| 8 | PAUTH_ACTIVE | 1 | ✓ | PAuth 指令激活 |
| 9 | BT | 1 | ✓ | BTI 启用（SCTLR.BT0/BT1） |
| 10:11 | BTYPE | 2 | ✗ | 分支类型（动态） |
| 12:13 | TBID | 2 | ✓ | Top Byte Ignore for Data |
| 14 | UNPRIV | 1 | ✓ | LDTR 使用 AccType_UNPRIV |
| 15 | ATA | 1 | ✓ | MTE Allocation Tag Access |
| 16:17 | TCMA | 2 | ✓ | Tag Check Match All |
| 18 | MTE_ACTIVE | 1 | ✓ | MTE 标签检查活动 |
| 19 | MTE0_ACTIVE | 1 | ✓ | MTE EL0 标签检查 |
| 20:21 | SMEEXC_EL | 2 | ✓ | SME 异常目标 EL |
| 22 | PSTATE_SM | 1 | ✓ | SME Streaming 模式 |
| 23 | PSTATE_ZA | 1 | ✓ | SME ZA 启用 |
| 24:27 | SVL | 4 | ✓ | SME Streaming VL |
| 28 | SME_TRAP_NONSTREAMING | 1 | ✓ | SM + !FA64 陷阱 |
| 29 | TRAP_ERET | 1 | ✓ | ERET 陷阱（FGT/NV） |
| 30 | NAA | 1 | ✓ | 非对齐原子（SCTLR.nAA） |
| 31 | ATA0 | 1 | ✓ | MTE EL0 ATA |
| 32 | NV | 1 | ✓ | 嵌套虚拟化 HCR.NV |
| 33 | NV1 | 1 | ✓ | HCR.NV1 |
| 34 | NV2 | 1 | ✓ | HCR.NV2 |
| 35 | E2H | 1 | ✓ | VHE HCR.E2H |
| 36 | NV2_MEM_BE | 1 | ✓ | NV2 内存访问大端 |
| 37 | AH | 1 | ✓ | FPCR.AH |
| 38 | NEP | 1 | ✓ | FPCR.NEP |
| 39:40 | ZT0EXC_EL | 2 | ✓ | ZT0 异常目标 EL |
| 41 | GCS_EN | 1 | ✓ | GCS 启用 |
| 42 | GCS_RVCEN | 1 | ✓ | GCS Return Verify Check 启用 |
| 43:44 | GCSSTR_EL | 2 | ✓ | GCSSTTR 异常目标 EL |

---

## 4. rebuild_hflags_a64 构建流程

`rebuild_hflags_a64()`（hflags.c:240-504）是 hflags 构建的核心，根据当前 EL、MMU 域、SCTLR、HCR、TCR 等计算所有 TB 标志位。

### 4.1 十大构建阶段

```c
// hflags.c:240-504
static CPUARMTBFlags rebuild_hflags_a64(CPUARMState *env, int el, int fp_el,
                                        ARMMMUIdx mmu_idx)
{
    // 阶段 1: 基本标志 (250)
    DP_TBFLAG_ANY(flags, AARCH64_STATE, 1);

    // 阶段 2: TBI/TBID 从 TCR 提取 (253-257)
    tbii = tbid & ~aa64_va_parameter_tbid(tcr, mmu_idx);
    DP_TBFLAG_A64(flags, TBII, tbii);
    DP_TBFLAG_A64(flags, TBID, tbid);

    // 阶段 3: E2H 标志 (259-262)
    if (hcr & HCR_E2H) DP_TBFLAG_A64(flags, E2H, 1);

    // 阶段 4: SVE/SME 向量长度与异常目标 (264-307)
    // sve_exception_el, sme_exception_el, vl, svl 计算

    // 阶段 5: SCTLR 派生 — 对齐/端序/PAuth/BTI/NAA (310-343)
    sctlr = regime_sctlr(env, stage1);
    if (aprofile_require_alignment()) DP_TBFLAG_ANY(flags, ALIGN_MEM, 1);
    if (arm_cpu_data_is_big_endian_a64()) DP_TBFLAG_ANY(flags, BE_DATA, 1);
    if (sctlr & (SCTLR_EnIA|EnIB|EnDA|EnDB)) → PAUTH_ACTIVE
    if (sctlr & SCTLR_BT0/BT1) → BT
    if (sctlr & SCTLR_nAA) → NAA

    // 阶段 6: UNPRIV（LDTR 访问类型）(345-368)
    // E10_1 → UNPRIV (除非 NV+NV1)
    // E20_2 + TGE → UNPRIV

    // 阶段 7: PSTATE__IL + FGT + TRAP_ERET (370-382)
    if (PSTATE_IL) → PSTATE__IL
    if (arm_fgt_active()) → FGT_ACTIVE, HFGITR.ERET → TRAP_ERET, fgt_svc

    // 阶段 8: NV/NV1/NV2 嵌套虚拟化 (384-400)
    if (el == 1 && (hcr & HCR_NV)) → NV, NV1, NV2, TRAP_ERET, NV2_MEM_BE

    // 阶段 9: MTE 标签检查 (402-450)
    // ATA, ATA0, TCMA, MTE_ACTIVE, MTE0_ACTIVE

    // 阶段 10: GCS + FPCR (452-501)
    // GCS_EN, GCS_RVCEN, GCSSTR_EL, AH, NEP

    return rebuild_hflags_common(env, fp_el, mmu_idx, flags);
}
```

### 4.2 hflags_common — 共享最终处理

`rebuild_hflags_common()`（hflags.c:66-85）添加 MMUIDX 和 FPEXC_EL：

```c
DP_TBFLAG_ANY(flags, MMUIDX, arm_to_core_mmu_idx(mmu_idx));
DP_TBFLAG_ANY(flags, FPEXC_EL, fp_el);
```

### 4.3 rebuild_hflags_internal — 模式分发

```c
// hflags.c:506-519
static CPUARMTBFlags rebuild_hflags_internal(CPUARMState *env)
{
    int el = arm_current_el(env);
    int fp_el = fp_exception_el(env, el);
    ARMMMUIdx mmu_idx = arm_mmu_idx_el(env, el);

    if (is_a64(env))
        return rebuild_hflags_a64(env, el, fp_el, mmu_idx);
    else if (arm_feature(env, ARM_FEATURE_M))
        return rebuild_hflags_m32(env, fp_el, mmu_idx);
    else
        return rebuild_hflags_a32(env, fp_el, mmu_idx);
}

// hflags.c:521-524
void arm_rebuild_hflags(CPUARMState *env)
{
    env->hflags = rebuild_hflags_internal(env);
}
```

### 4.4 Helper 版本（用于 EL 切换后）

```c
// hflags.c:567-573
void HELPER(rebuild_hflags_a64)(CPUARMState *env, int el)
{
    int fp_el = fp_exception_el(env, el);
    ARMMMUIdx mmu_idx = arm_mmu_idx_el(env, el);
    env->hflags = rebuild_hflags_a64(env, el, fp_el, mmu_idx);
}
```

---

## 5. arm_get_tb_cpu_state — TB 键生成

```c
// hflags.c:620-695
TCGTBCPUState arm_get_tb_cpu_state(CPUState *cs)
{
    assert_hflags_rebuild_correctly(env);    // 断言 hflags 是最新的
    flags = env->hflags;                     // 复制缓存的 hflags

    if (AARCH64_STATE) {
        pc = env->pc;
        // 动态填充 BTYPE（不缓存）
        DP_TBFLAG_A64(flags, BTYPE, env->btype);
    } else {
        pc = env->regs[15];
        // 动态填充 THUMB, CONDEXEC, VECLEN, VECSTRIDE 等
    }

    // 动态填充 PSTATE__SS（不缓存，每次 TB 查询时计算）
    if (SS_ACTIVE && (env->pstate & PSTATE_SS)) {
        DP_TBFLAG_ANY(flags, PSTATE__SS, 1);
    }

    return { .pc = pc, .flags = flags.flags, .cs_base = flags.flags2 };
}
```

**TB 键 = (PC, flags, flags2)**。任何 hflag 位变化都会导致不同的 TB 查找结果，从而确保不同 EL 下的代码使用不同的翻译缓存。

---

## 6. 异常入口的状态切换与 hflags 重建

### 6.1 完整异常入口流程

```c
// helper.c:9343-9421  arm_cpu_do_interrupt_aarch64() 核心片段

// 第1步：保存当前 SP 到 sp_el[cur_el]
aarch64_save_sp(env, arm_current_el(env));       // 9345

// 第2步：保存返回地址到 ELR_ELn
env->elr_el[new_el] = env->pc;                   // 9346

// 第3步：保存 PSTATE 到 SPSR_ELn
old_mode = pstate_read(env);                      // 9344
env->banked_spsr[aarch64_banked_spsr_index(new_el)] = old_mode;  // 9369

// 第4步：设置新的 PSTATE
// PAN: 目标 EL1/EL2 且 SCTLR.SPAN=0 → PAN=1           (9375-9393)
// TCO: FEAT_MTE → TCO=1                                  (9395-9397)
// SSBS: 从 SCTLR_ELn.DSSBS 读取                         (9399-9404)
// ALLINT: SCTLR_ELn.SPINTMASK 控制                      (9407-9412)
pstate_write(env, PSTATE_DAIF | new_mode);        // 9415（DAIF 全置位）

// 第5步：切换到 AArch64 模式
env->aarch64 = true;                              // 9416

// 第6步：恢复目标 EL 的 SP
aarch64_restore_sp(env, new_el);                  // 9417

// 第7步：重建 hflags（TCG 模式下）
if (tcg_enabled()) {
    arm_rebuild_hflags(env);                       // 9419-9421
}

// 第8步：设置新 PC（VBAR + 向量偏移）
env->pc = addr;                                    // 9423
```

### 6.2 关键观察

- **PSTATE_DAIF 全置位**：进入异常后 D/A/I/F 全部屏蔽
- **hflags 完全重建**：不是增量更新，而是从头计算所有位
- **SP 切换**：保存旧 SP → 恢复新 EL 的 SP（使用 SP_ELn，因新 PSTATE.SP=1）
- **TB 链中断**：异常入口后 PC 变为 VBAR+offset，新 hflags → 新 TB 查找

---

## 7. ERET 的状态恢复与 hflags 重建

### 7.1 翻译阶段

```c
// translate-a64.c:1951-1974  trans_ERET
if (s->current_el == 0) → UNDEF                  // ERET 仅 EL1+ 可用
if (s->trap_eret) → TRAP                          // FGT/NV 陷阱
dst = ELR_ELx                                     // 加载返回地址
gen_helper_exception_return(tcg_env, dst);         // 调用 helper
s->base.is_jmp = DISAS_EXIT;                      // 强制 TB 退出
```

### 7.2 运行时 Helper（AArch64→AArch64 返回路径）

```c
// helper-a64.c:718-750
env->aarch64 = true;                               // 721
spsr &= aarch64_pstate_valid_mask(&cpu->isar);     // 722
pstate_write(env, spsr);                            // 723（恢复 PSTATE）

if (!arm_singlestep_active(env)) {
    env->pstate &= ~PSTATE_SS;                     // 724-726（清除 SS）
}

aarch64_restore_sp(env, new_el);                    // 727（恢复目标 SP）
helper_rebuild_hflags_a64(env, new_el);             // 728（重建 hflags）

// TBI 处理：根据新 EL 的 TBII 对 PC 去除 top byte
tbii = EX_TBFLAG_A64(env->hflags, TBII);           // 737
if ((tbii >> extract64(new_pc, 55, 1)) & 1) {
    new_pc = sextract64(new_pc, 0, 56);            // 742 (双范围)
}
env->pc = new_pc;                                   // 747
```

### 7.3 ERET 关键时序

```
ERET 指令
  │
  ├── 翻译阶段：gen_helper_exception_return + DISAS_EXIT
  │
  └── 运行时 helper:
      1. aarch64_save_sp(env, cur_el)        保存当前 SP
      2. 验证目标 EL/宽度/GCS/TGE           合法性检查
      3. pstate_write(env, spsr)              恢复 PSTATE
      4. aarch64_restore_sp(env, new_el)      恢复目标 SP
      5. helper_rebuild_hflags_a64(env, new_el) 重建 hflags
      6. TBI 处理 → env->pc = new_pc          设置新 PC
      7. TB 查找使用新 hflags                  进入新 TB
```

---

## 8. TB 链断裂机制

### 8.1 gen_goto_tb — TB 链接决策

```c
// translate-a64.c:535-564
static void gen_goto_tb(DisasContext *s, unsigned tb_slot_idx, int64_t diff)
{
    if (use_goto_tb(s, s->pc_curr + diff)) {
        // 同页内跳转：可以直接链接
        tcg_gen_goto_tb(tb_slot_idx);
        tcg_gen_exit_tb(s->base.tb, tb_slot_idx);
    } else {
        // 跨页跳转：通过查找表
        gen_a64_update_pc(s, diff);
        if (s->ss_active) {
            gen_step_complete_exception(s);  // 单步：生成异常
        } else {
            tcg_gen_lookup_and_goto_ptr();   // 查找 TB 并跳转
        }
    }
}
```

### 8.2 EL 切换为什么断裂 TB 链

1. **异常入口**：`arm_cpu_do_interrupt` 设置 `CPU_INTERRUPT_EXITTB`（helper.c:9526-9528），强制退出当前 TB
2. **PC 变化**：异常入口 PC = VBAR + offset，ERET PC = ELR_ELn，都与当前 TB 地址不连续
3. **hflags 变化**：EL 变化 → MMU 索引变化 → TB 键变化 → 不可能链接到旧 TB
4. **ERET 强制退出**：trans_ERET 设置 `DISAS_EXIT`，不生成任何直接链接

### 8.3 TB 链断裂的具体触发点

| 触发源 | 机制 | 代码位置 |
|--------|------|---------|
| 同步异常（SVC/HVC/SMC） | EXCP_* → do_interrupt → EXITTB | helper.c:9526 |
| IRQ/FIQ 中断 | cpu_handle_interrupt → do_interrupt | cpu-exec.c:839-856 |
| ERET | DISAS_EXIT → 不生成 goto_tb | translate-a64.c:1972-1974 |
| 系统寄存器写入 | 写 SCTLR/HCR 等 → gen_lookup_and_goto_ptr | 各 cpreg writefn |
| PSTATE 修改（MSR DAIFSet 等） | 可能改变 hflags → TB 退出 | translate-a64.c 相关 |

---

## 9. 寄存器组切换

### 9.1 寄存器组存储

```c
// cpu.h:296-318
uint64_t banked_spsr[8];  // 8 个 SPSR 存储槽（按 EL/模式索引）
uint64_t elr_el[4];       // ELR_EL1, ELR_EL2, ELR_EL3（[0]未用）
uint64_t sp_el[4];        // SP_EL0, SP_EL1, SP_EL2, SP_EL3
```

### 9.2 SP 保存/恢复

```c
// internals.h:565-581
static inline void aarch64_save_sp(CPUARMState *env, int el)
{
    if (env->pstate & PSTATE_SP) {
        env->sp_el[el] = env->xregs[31];    // SPSel=1: 保存到 SP_ELn
    } else {
        env->sp_el[0] = env->xregs[31];     // SPSel=0: 保存到 SP_EL0
    }
}

static inline void aarch64_restore_sp(CPUARMState *env, int el)
{
    if (env->pstate & PSTATE_SP) {
        env->xregs[31] = env->sp_el[el];    // 恢复 SP_ELn
    } else {
        env->xregs[31] = env->sp_el[0];     // 恢复 SP_EL0
    }
}
```

### 9.3 异常入口的寄存器保存序列

```
异常发生时（cur_el → new_el）:
  1. aarch64_save_sp(env, cur_el)           → sp_el[cur_el] = X31
  2. env->elr_el[new_el] = env->pc          → 保存返回地址
  3. banked_spsr[idx] = pstate_read(env)     → 保存 PSTATE
  4. pstate_write(env, PSTATE_DAIF|new_mode) → 设置新 PSTATE（SP=1）
  5. aarch64_restore_sp(env, new_el)         → X31 = sp_el[new_el]
```

### 9.4 ERET 的寄存器恢复序列

```
ERET（cur_el → new_el）:
  1. aarch64_save_sp(env, cur_el)           → 保存当前 SP
  2. spsr = banked_spsr[idx]                → 读取保存的 PSTATE
  3. pstate_write(env, spsr)                → 恢复 PSTATE
  4. aarch64_restore_sp(env, new_el)        → 恢复目标 SP
  5. env->pc = env->elr_el[cur_el]          → 恢复返回地址
```

---

## 10. DisasContext 初始化与 EL 解码

### 10.1 从 hflags 提取翻译参数

```c
// translate-a64.c:10667-10750  aarch64_tr_init_disas_context()

// 核心 EL 识别
core_mmu_idx = EX_TBFLAG_ANY(tb_flags, MMUIDX);           // 10671
dc->mmu_idx = core_to_aa64_mmu_idx(core_mmu_idx);         // 10672
dc->current_el = arm_mmu_idx_to_el(dc->mmu_idx);          // 10676
dc->user = (dc->current_el == 0);                          // 10678

// 地址标签忽略
dc->tbii = EX_TBFLAG_A64(tb_flags, TBII);                 // 10673
dc->tbid = EX_TBFLAG_A64(tb_flags, TBID);                 // 10674

// SCTLR 派生
dc->align_mem = EX_TBFLAG_ANY(tb_flags, ALIGN_MEM);       // 10681
dc->pstate_il = EX_TBFLAG_ANY(tb_flags, PSTATE__IL);      // 10682
dc->naa = EX_TBFLAG_A64(tb_flags, NAA);                   // 10703

// 安全/虚拟化特性
dc->fgt_active = EX_TBFLAG_ANY(tb_flags, FGT_ACTIVE);     // 10683
dc->trap_eret = EX_TBFLAG_A64(tb_flags, TRAP_ERET);       // 10685
dc->e2h = EX_TBFLAG_A64(tb_flags, E2H);                   // 10704
dc->nv = EX_TBFLAG_A64(tb_flags, NV);                     // 10705
dc->nv2 = EX_TBFLAG_A64(tb_flags, NV2);                   // 10707

// 功能特性
dc->pauth_active = EX_TBFLAG_A64(tb_flags, PAUTH_ACTIVE); // 10692
dc->bt = EX_TBFLAG_A64(tb_flags, BT);                     // 10693
dc->unpriv = EX_TBFLAG_A64(tb_flags, UNPRIV);             // 10695

// 单步调试
dc->ss_active = EX_TBFLAG_ANY(tb_flags, SS_ACTIVE);       // 10744
dc->pstate_ss = EX_TBFLAG_ANY(tb_flags, PSTATE__SS);      // 10745
```

### 10.2 current_el 的间接推导

注意 `current_el` **不是直接从 hflags 读取的位域**，而是从 MMUIDX 间接推导：

```
MMUIDX → arm_mmu_idx_to_el() → current_el

E10_0 / E20_0 / E30_0     → EL0
E10_1 / E10_1_PAN         → EL1
E2 / E20_2 / E20_2_PAN    → EL2
E3 / E30_3 / E30_3_PAN    → EL3
```

这种设计的好处是 MMUIDX 同时编码了 EL 和 MMU regime（含 PAN、VHE 等信息）。

---

## 11. SCTLR 对翻译行为的影响

以下 SCTLR 位通过 hflags 影响 TCG 翻译：

| SCTLR 位 | hflag 字段 | 影响 | hflags.c 行号 |
|----------|-----------|------|-------------|
| SCTLR_A (对齐) | ALIGN_MEM | 所有 load/store 对齐检查 | 312-314 |
| SCTLR_EE (端序) | BE_DATA | 数据访问大/小端 | 316-318 |
| SCTLR_EnIA/IB/DA/DB | PAUTH_ACTIVE | PAuth 指令生效 vs NOP | 327-329 |
| SCTLR_BT0/BT1 | BT | BTI 分支目标识别启用 | 332-336 |
| SCTLR_nAA | NAA | 非对齐原子操作允许 | 339-342 |

**每个 EL 有自己的 SCTLR**（SCTLR_EL1/EL2/EL3），切换 EL 后使用不同的 SCTLR，因此这些 hflags 值自然不同。

---

## 12. HCR_EL2 对翻译行为的影响

HCR_EL2 通过 hflags 影响 EL1（和 EL0）的指令翻译：

| HCR 位 | hflag 字段 | 影响 | hflags.c 行号 |
|--------|-----------|------|-------------|
| HCR_E2H | E2H | VHE 模式：寄存器重定向、MMU 索引 | 259-262 |
| HCR_TGE+E2H | UNPRIV | EL0 在 Host 中时 LDTR 行为 | 355-363 |
| HCR_NV | NV + TRAP_ERET | EL1 ERET/sysreg 陷阱 | 388-390 |
| HCR_NV1 | NV1 | NV UNPRIV 语义修改 | 391-393 |
| HCR_NV2 | NV2 + NV2_MEM_BE | VNCR 内存后备寄存器 | 394-399 |
| FGT (HFGITR.ERET) | TRAP_ERET | ERET 细粒度陷阱 | 376-378 |
| FGT (SVC 陷阱) | FGT_SVC | SVC 陷阱到 EL2 | 379-381 |

---

## 13. EL 相关指令行为差异

### 13.1 仅特定 EL 可用的指令

| 指令 | 最低 EL | 检查方式 |
|------|---------|---------|
| SVC | EL0+ | 所有 EL 都可以（但可能被 FGT 陷阱） |
| HVC | EL1+ | `s->current_el == 0` → UNDEF |
| SMC | EL1+ | `s->current_el == 0` → UNDEF |
| ERET | EL1+ | `s->current_el == 0` → UNDEF |
| MSR DAIFSet/Clr | EL1+ (无 UMA) | EL0 需 SCTLR.UMA=1 |
| AT/TLBI/DC | 取决于操作 | cpreg accessfn 检查 |

### 13.2 翻译时的 EL 检查模式

```c
// translate-a64.c 中的典型模式
if (s->current_el == 0) {
    unallocated_encoding(s);   // EL0 不可用
    return true;
}

// cpreg 访问检查
static CPAccessResult access_tdra(CPUARMState *env, ...)
{
    int el = arm_current_el(env);
    // EL0/EL1 根据 MDCR_EL2/MDCR_EL3 决定是否陷阱
}
```

### 13.3 系统寄存器 EL 路由

同名寄存器在不同 EL 访问不同的底层存储：

```
SCTLR_EL1 from EL1 → env->cp15.sctlr_el[1]
SCTLR_EL1 from EL2+VHE → 重定向到 env->cp15.sctlr_el[2]
SCTLR_EL2 from EL2 → env->cp15.sctlr_el[2]
SCTLR_EL12 from EL2+VHE → env->cp15.sctlr_el[1]（原始 EL1）
```

---

## 14. 单步调试跨 EL 处理

### 14.1 单步状态机

```
SS_ACTIVE   PSTATE.SS   状态
  0           x         Inactive（不产生单步异常）
  1           0         Active-pending（即将产生异常）
  1           1         Active-not-pending（执行一条指令后进入 pending）
```

### 14.2 hflags 编码

```c
// hflags.c:620-688
// SS_ACTIVE 在 hflags 缓存中（rebuild 时设置）
// PSTATE__SS 不缓存 — 每次 arm_get_tb_cpu_state 时动态检查
if (SS_ACTIVE && (env->pstate & PSTATE_SS)) {
    DP_TBFLAG_ANY(flags, PSTATE__SS, 1);
}
```

### 14.3 跨 EL 单步行为

- **异常入口**：`cpsr_read_for_spsr_elx()` 将当前 PSTATE.SS 合并到 SPSR 中保存（helper.c:9139-9151）
- **ERET 恢复**：从 SPSR 恢复 PSTATE.SS，但如果 `!arm_singlestep_active(env)` 则清除 SS（helper-a64.c:724-726）
- **TB 终止**：active-not-pending 状态下，每执行一条指令就终止 TB 并生成 step 异常

---

## 15. 完整 EL 切换流程总结

### 异常入口（EL0→EL1 以 SVC 为例）

```
Guest 执行 SVC #imm
    │
    ├── TCG 翻译：trans_SVC → 生成 EXCP_SWI
    │   └── gen_exception_insn → s->is_jmp = DISAS_NORETURN（TB 结束）
    │
    ├── TCG 执行循环：cpu_handle_exception
    │   └── arm_cpu_do_interrupt_aarch64(cs, EXCP_SWI)
    │       │
    │       ├── aarch64_save_sp(env, 0)          SP → sp_el[0]
    │       ├── env->elr_el[1] = env->pc         保存 PC
    │       ├── banked_spsr[idx] = pstate         保存 PSTATE
    │       ├── pstate_write(PSTATE_DAIF|EL1h)    DAIF=1111, SPSel=1
    │       ├── aarch64_restore_sp(env, 1)        X31 = sp_el[1]
    │       ├── arm_rebuild_hflags(env)           重建 hflags（MMUIDX=E10_1）
    │       └── env->pc = VBAR_EL1 + 0x200       设置异常向量
    │
    └── cpu_exec_loop：TB 查找
        key = (VBAR_EL1+0x200, new_flags, new_flags2)
        └── 新 TB（EL1 上下文翻译）
```

### ERET 返回（EL1→EL0）

```
Guest 执行 ERET
    │
    ├── TCG 翻译：trans_ERET → gen_helper_exception_return + DISAS_EXIT
    │
    ├── helper exception_return():
    │   ├── aarch64_save_sp(env, 1)              SP → sp_el[1]
    │   ├── spsr = banked_spsr[idx]              读取保存的 PSTATE
    │   ├── 验证目标 EL/宽度                     合法性检查
    │   ├── pstate_write(env, spsr)              恢复 PSTATE（EL0t, DAIF 恢复）
    │   ├── aarch64_restore_sp(env, 0)           X31 = sp_el[0]
    │   ├── helper_rebuild_hflags_a64(env, 0)    重建 hflags（MMUIDX=E10_0）
    │   └── env->pc = elr_el[1] (TBI 处理后)    恢复 PC
    │
    └── DISAS_EXIT → cpu_exec_loop → TB 查找
        key = (elr_el[1], new_flags, new_flags2)
        └── 新 TB（EL0 上下文翻译）
```

### hflags 关键差异（EL0 vs EL1）

| hflag 字段 | EL0 (E10_0) | EL1 (E10_1) | 原因 |
|-----------|-------------|-------------|------|
| MMUIDX | E10_0 | E10_1 / E10_1_PAN | MMU regime 不同 |
| UNPRIV | 0 | 1 (有 LDTR) | EL1 LDTR 用 EL0 权限 |
| FPEXC_EL | 可能 ≠0 | 通常 0 | EL0 FP 可能被陷阱 |
| BT | SCTLR.BT0 | SCTLR.BT1 | BTI 不同位 |
| ATA/ATA0 | — | ATA 不同 | MTE 标签检查差异 |
| NV/NV1/NV2 | 0 | 可能 1 | 仅 EL1 受 NV 影响 |
| TRAP_ERET | 0 | 可能 1 | 仅 EL1 ERET 可被陷阱 |

---

## 交叉参考

- [40-ARM64-EL1-EL2交互深度分析](40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md) — HCR_EL2 位域、VHE、NV
- [39-ARM64-EL3-Secure世界切换深度分析](39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md) — SMC/ERET、安全状态切换
- [00-ARM64-CPU-GICv3-TCG深度分析](00-ARM64-CPU-GICv3-TCG深度分析.md) — TCG 翻译器基础

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
