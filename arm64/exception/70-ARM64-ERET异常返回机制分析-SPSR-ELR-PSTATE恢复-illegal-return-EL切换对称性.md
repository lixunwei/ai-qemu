# ARM64 ERET 异常返回与 EL 状态恢复机制分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/70 |
| 分析对象 | ERET 指令执行流程与 EL2→EL1 状态恢复 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §D1.10 (Exception return) |
| 关键数据 | SPSR_ELx, ELR_ELx, PSTATE, VBAR_ELx |
| 核心结论 | **ERET 实现完整，illegal return 处理符合规范** |

---

## 1. 概述

ERET（Exception Return）是 ARM64 中从高特权 EL 返回低特权 EL 的唯一指令。
它与异常入口 `arm_cpu_do_interrupt_aarch64()` 构成对称的 EL 切换对：

```
异常入口 (arm_cpu_do_interrupt):        异常返回 (ERET):
  SPSR_ELx ← PSTATE                      PSTATE ← SPSR_ELx
  ELR_ELx  ← PC                          PC     ← ELR_ELx
  ESR_ELx  ← syndrome                    (ESR_ELx 不变)
  PSTATE   ← {DAIF=1111, SP=1, ...}      恢复完整 PSTATE
  PC       ← VBAR_ELx + offset           回到被中断地址
  EL       ← new_el (上升)               EL     ← SPSR.M (下降)
```

---

## 2. 异常入口：arm_cpu_do_interrupt_aarch64()

### 2.1 代码位置

`target/arm/helper.c:9198`

### 2.2 执行步骤

```c
static void arm_cpu_do_interrupt_aarch64(CPUState *cs)
{
    unsigned int new_el = env->exception.target_el;  // 目标 EL
    vaddr addr = env->cp15.vbar_el[new_el];          // 向量基址
    unsigned int cur_el = arm_current_el(env);       // 当前 EL

    // ① SVE/SME 向量长度可能在 EL 切换时变化
    aarch64_sve_change_el(env, cur_el, new_el, is_a64(env));

    // ② 计算向量偏移
    if (cur_el < new_el) {
        // 从低 EL 来：根据低 EL 是 AArch64/AArch32 选择偏移
        if (is_aa64) addr += 0x400;  // Lower EL using AArch64
        else         addr += 0x600;  // Lower EL using AArch32
    } else {
        // 同 EL 来：根据 SP 选择
        if (pstate_read(env) & PSTATE_SP) addr += 0x200;  // SPx
        // else: SP0 偏移为 0x000
    }

    // ③ 根据异常类型加偏移
    // Synchronous: +0x000, IRQ: +0x080, FIQ: +0x100, SError: +0x180

    // ④ 保存旧状态
    old_mode = pstate_read(env);
    aarch64_save_sp(env, cur_el);
    env->elr_el[new_el] = env->pc;                  // 保存返回地址
    env->banked_spsr[index] = old_mode;              // 保存 PSTATE

    // ⑤ 写 ESR_ELx（同步异常）
    env->cp15.esr_el[new_el] = env->exception.syndrome;

    // ⑥ 设置新 PSTATE
    new_mode = aarch64_pstate_mode(new_el, true);    // M=ELx, SP=1
    new_mode |= PSTATE_DAIF;                         // mask 所有中断
    // PAN、TCO、SSBS、ALLINT 根据 SCTLR 配置设置
    pstate_write(env, PSTATE_DAIF | new_mode);
    env->aarch64 = true;
    aarch64_restore_sp(env, new_el);

    // ⑦ 跳转到向量
    env->pc = addr;
}
```

### 2.3 VBAR 向量表布局

```
VBAR_ELx + 0x000: Synchronous (Current EL, SP_EL0)
VBAR_ELx + 0x080: IRQ         (Current EL, SP_EL0)
VBAR_ELx + 0x100: FIQ         (Current EL, SP_EL0)
VBAR_ELx + 0x180: SError      (Current EL, SP_EL0)

VBAR_ELx + 0x200: Synchronous (Current EL, SP_ELx)
VBAR_ELx + 0x280: IRQ         (Current EL, SP_ELx)
VBAR_ELx + 0x300: FIQ         (Current EL, SP_ELx)
VBAR_ELx + 0x380: SError      (Current EL, SP_ELx)

VBAR_ELx + 0x400: Synchronous (Lower EL, AArch64)
VBAR_ELx + 0x480: IRQ         (Lower EL, AArch64)
VBAR_ELx + 0x500: FIQ         (Lower EL, AArch64)
VBAR_ELx + 0x580: SError      (Lower EL, AArch64)

VBAR_ELx + 0x600: Synchronous (Lower EL, AArch32)
VBAR_ELx + 0x680: IRQ         (Lower EL, AArch32)
VBAR_ELx + 0x700: FIQ         (Lower EL, AArch32)
VBAR_ELx + 0x780: SError      (Lower EL, AArch32)
```

### 2.4 PSTATE 在异常入口的变化

| 字段 | 入口时设置 | 来源 |
|------|-----------|------|
| M[3:0] | new_el, SP=1 | 固定：SPx 模式 |
| DAIF | 0b1111 | 全部 mask |
| PAN | 保留旧值 或 设1 | SCTLR.SPAN==0 时强制设1 |
| TCO | 1 | FEAT_MTE: tag check 关闭 |
| SSBS | SCTLR.DSSBS | 按 SCTLR 配置 |
| ALLINT | 按 SCTLR.SPINTMASK | NMI 相关 |
| BTYPE | 0 | 清除 |
| SS | 0 | 清除 |
| IL | 0 | 清除 |
| nRW | 0 | AArch64 |

---

## 3. ERET 异常返回

### 3.1 翻译阶段

```c
// target/arm/tcg/translate-a64.c:1951
static bool trans_ERET(DisasContext *s, arg_ERET *a)
{
    if (s->current_el == 0) {
        return false;           // EL0 不能执行 ERET
    }
    if (s->trap_eret) {
        // FEAT_NV: EL1 的 ERET 被 trap 到 EL2
        gen_exception_insn_el(s, 0, EXCP_UDEF, syn_erettrap(0), 2);
        return true;
    }

    // 加载 ELR_ELx 作为目标 PC
    dst = tcg_temp_new_i64();
    tcg_gen_ld_i64(dst, tcg_env, offsetof(CPUARMState, elr_el[s->current_el]));

    translator_io_start(&s->base);      // IO barrier
    gen_helper_exception_return(tcg_env, dst);
    s->base.is_jmp = DISAS_EXIT;        // 强制退出 TB
    return true;
}
```

### 3.2 ERETA（带 PAC 认证的 ERET）

```c
// translate-a64.c:1978
static bool trans_ERETA(DisasContext *s, arg_reta *a)
{
    if (!dc_isar_feature(aa64_pauth, s)) return false;
    if (s->current_el == 0) return false;

    if (s->trap_eret) {
        gen_exception_insn_el(s, 0, EXCP_UDEF,
            syn_erettrap(a->m ? 3 : 2), 2);  // RETAA=2, RETAB=3
        return true;
    }

    dst = ELR_ELx;
    dst = auth_branch_target(s, dst, SP, !a->m);  // PAC 验证

    gen_helper_exception_return(tcg_env, dst);
    s->base.is_jmp = DISAS_EXIT;
    return true;
}
```

### 3.3 Runtime Helper: HELPER(exception_return)

```c
// target/arm/tcg/helper-a64.c:622
void HELPER(exception_return)(CPUARMState *env, uint64_t new_pc)
{
    int cur_el = arm_current_el(env);
    unsigned int spsr_idx = aarch64_banked_spsr_index(cur_el);
    uint64_t spsr = env->banked_spsr[spsr_idx];  // 读取 SPSR_ELx
    int new_el;
    bool return_to_aa64 = (spsr & PSTATE_nRW) == 0;

    aarch64_save_sp(env, cur_el);     // 保存当前 SP 到 SP_ELx
    arm_clear_exclusive(env);          // 清除 exclusive monitor

    // 单步调试处理
    if (arm_generate_debug_exceptions(env)) {
        spsr &= ~PSTATE_SS;
    }

    // ① 从 SPSR 提取目标 EL
    new_el = el_from_spsr(spsr);
    if (new_el == -1) goto illegal_return;

    // ② 合法性检查
    if (new_el > cur_el) goto illegal_return;           // 不能返回到更高 EL
    if (new_el == 2 && !arm_is_el2_enabled(env))        // EL2 未实现
        goto illegal_return;
    if (new_el != 0 && arm_el_is_aa64(env, new_el) != return_to_aa64)
        goto illegal_return;                             // 执行状态不匹配
    if (!return_to_aa64 && !cpu_isar_feature(aa64_aa32, cpu))
        goto illegal_return;                             // CPU 不支持 AArch32
    if (new_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE))
        goto illegal_return;                             // TGE=1 时 EL1 不可用

    // FEAT_RME: EL3 返回到无效安全状态
    if (cur_el == 3 && new_el < 3 &&
        (scr_el3 & (SCR_NS | SCR_NSE)) == SCR_NSE)
        goto illegal_return;

    // FEAT_GCS: EXLOCK 检查
    if (new_el == cur_el && GCSCR.EXLOCKEN && !PSTATE.EXLOCK)
        goto illegal_return;

    // ③ 调用 pre-EL-change hook
    arm_call_pre_el_change_hook(cpu);

    // ④ 执行返回
    if (!return_to_aa64) {
        // 返回到 AArch32
        env->aarch64 = false;
        cpsr_write_from_spsr_elx(env, spsr);
        aarch64_sync_64_to_32(env);
        env->regs[15] = new_pc & ~(thumb ? 0x1 : 0x3);
    } else {
        // 返回到 AArch64
        env->aarch64 = true;
        spsr &= aarch64_pstate_valid_mask(&cpu->isar);  // 过滤无效位
        pstate_write(env, spsr);                          // 恢复 PSTATE
        aarch64_restore_sp(env, new_el);                  // 恢复 SP

        // TBI 地址处理
        tbii = EX_TBFLAG_A64(env->hflags, TBII);
        if ((tbii >> extract64(new_pc, 55, 1)) & 1) {
            new_pc = sextract64(new_pc, 0, 56);  // 符号扩展
        }
        env->pc = new_pc;
    }

    // ⑤ SVE 向量长度切换
    aarch64_sve_change_el(env, cur_el, new_el, return_to_aa64);

    // ⑥ 调用 post-EL-change hook
    arm_call_el_change_hook(cpu);
    return;

illegal_return:
    // 非法返回处理（见 §4）
    ...
}
```

---

## 4. Illegal Return 处理

### 4.1 ARM 规范定义

DDI 0487 §D1.10.1 定义了 illegal exception return 的行为：

> "An illegal exception return is one where the combination of the target
> exception level, register width, or Security state cannot be valid."

发生 illegal return 时：
- **不改变异常级别**（留在当前 EL）
- 恢复 NZCV 和 DAIF（从 SPSR）
- 恢复 ALLINT（从 SPSR，如果实现）
- 设置 `PSTATE.IL = 1`（Illegal 标志）
- 恢复 PC（从 ELR_ELx）
- 不改变 SP 选择

### 4.2 QEMU 实现

```c
// helper-a64.c:766
illegal_return:
    env->pstate |= PSTATE_IL;          // 设置 IL 位
    env->pc = new_pc;                   // 仍然恢复 PC

    // 只恢复 NZCV + DAIF + ALLINT
    spsr &= PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT;
    spsr |= pstate_read(env) & ~(PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT);
    pstate_write(env, spsr);

    if (!arm_singlestep_active(env)) {
        env->pstate &= ~PSTATE_SS;
    }
    helper_rebuild_hflags_a64(env, cur_el);  // 保持当前 EL
```

### 4.3 触发 Illegal Return 的条件

| 条件 | QEMU 检查 | 规范章节 |
|------|:---------:|----------|
| SPSR.M 无效 | ✅ el_from_spsr 返回 -1 | §D1.10.1 |
| 返回到更高 EL | ✅ new_el > cur_el | §D1.10.1 |
| EL2 未实现 | ✅ !arm_is_el2_enabled | §D1.10.1 |
| 执行状态不匹配 | ✅ arm_el_is_aa64 != return_to_aa64 | §D1.10.1 |
| CPU 不支持 AArch32 | ✅ !aa64_aa32 | §D1.10.1 |
| HCR_EL2.TGE && EL1 | ✅ | §D1.10.1 |
| RME 无效安全状态 | ✅ SCR_NSE 检查 | FEAT_RME |
| GCS EXLOCK | ✅ GCSCR.EXLOCKEN | FEAT_GCS |

---

## 5. el_from_spsr() — SPSR 到 EL 的映射

### 5.1 AArch64 返回（SPSR.nRW == 0）

```c
// SPSR.M[3:2] = EL 编号
// SPSR.M[0] = SP 选择（0=SP_EL0, 1=SP_ELx）
// SPSR.M[1] 必须为 0（RES0）

int el = extract32(spsr, 2, 2);  // M[3:2]

// M[1] != 0 → illegal
if (extract32(spsr, 1, 1)) return -1;

// EL0 且 M[0] != 0 → illegal（EL0 只能用 SP_EL0）
if (extract32(spsr, 0, 4) == 1) return -1;

return el;  // 0, 1, 2, 或 3
```

### 5.2 AArch32 返回（SPSR.nRW == 1）

```c
switch (spsr & CPSR_M) {
    case ARM_CPU_MODE_USR: return 0;  // User → EL0
    case ARM_CPU_MODE_HYP: return 2;  // Hyp → EL2
    case ARM_CPU_MODE_FIQ:
    case ARM_CPU_MODE_IRQ:
    case ARM_CPU_MODE_SVC:
    case ARM_CPU_MODE_ABT:
    case ARM_CPU_MODE_UND:
    case ARM_CPU_MODE_SYS: return 1;  // → EL1
    case ARM_CPU_MODE_MON:             // Mon → illegal（不能从 AA64 返回 Mon）
    default: return -1;
}
```

---

## 6. PSTATE 恢复详解

### 6.1 aarch64_pstate_valid_mask

ERET 恢复 PSTATE 前，先过滤掉 CPU 不支持的位：

```c
uint32_t valid = PSTATE_M | PSTATE_DAIF | PSTATE_IL | PSTATE_SS | PSTATE_NZCV;

if (aa64_bti)   valid |= PSTATE_BTYPE;   // BTI 分支类型
if (aa64_pan)   valid |= PSTATE_PAN;     // 特权访问禁止
if (aa64_uao)   valid |= PSTATE_UAO;     // 用户访问覆盖
if (aa64_dit)   valid |= PSTATE_DIT;     // 数据独立时序
if (aa64_ssbs)  valid |= PSTATE_SSBS;    // 推测存储旁路
if (aa64_mte)   valid |= PSTATE_TCO;     // Tag Check 覆盖
if (aa64_nmi)   valid |= PSTATE_ALLINT;  // NMI mask

spsr &= valid;  // 未实现的位被忽略
pstate_write(env, spsr);
```

### 6.2 入口保存 vs 返回恢复对称性

```
异常入口时 SPSR_ELx 保存:              ERET 时从 SPSR_ELx 恢复:
  old_mode = pstate_read(env)            spsr = env->banked_spsr[idx]
  env->banked_spsr[idx] = old_mode       spsr &= valid_mask
                                          pstate_write(env, spsr)
```

### 6.3 特殊 PSTATE 位处理

| 位 | 入口行为 | 返回行为 |
|----|----------|----------|
| DAIF | 设为 0b1111 | 从 SPSR 恢复 |
| SP | 设为 1 (SPx) | 从 SPSR.M[0] 恢复 |
| PAN | 可能强制设1 | 从 SPSR 恢复 |
| IL | 清除 | 从 SPSR 恢复（通常是0） |
| SS | 特殊处理 | 特殊处理（见下） |
| TCO | 设为 1 | 从 SPSR 恢复 |
| nRW | 设为 0 | 从 SPSR 恢复（决定返回 AA64/AA32） |

### 6.4 Single-Step (SS) 特殊逻辑

```c
// 入口：如果 debug exceptions disabled，清除 SS
if (arm_generate_debug_exceptions(env)) {
    spsr &= ~PSTATE_SS;
}

// 返回后：如果新 EL 没有 active singlestep，清除 SS
if (!arm_singlestep_active(env)) {
    env->pstate &= ~PSTATE_SS;
}
```

---

## 7. SP 切换机制

### 7.1 aarch64_save_sp / aarch64_restore_sp

每个 EL 有两个 SP：SP_EL0（共享）和 SP_ELx（独立）。
`PSTATE.SP` (M[0]) 决定使用哪个。

```
异常入口：
  aarch64_save_sp(env, cur_el)  → 保存当前SP到对应bank
  aarch64_restore_sp(env, new_el)  → 从new_el的SPx bank恢复
  PSTATE.SP = 1  → 使用 SP_ELx

异常返回：
  aarch64_save_sp(env, cur_el)  → 保存EL2的SP
  aarch64_restore_sp(env, new_el)  → 恢复EL1的SP
  PSTATE.SP = SPSR.M[0]  → 恢复原来的SP选择
```

### 7.2 EL2→EL1 返回的 SP 状态变化示例

```
进入 EL2 时：
  SP_EL1 bank ← 当前 xregs[31]（EL1 的 SP）
  xregs[31] ← SP_EL2 bank（EL2 的 SPx）

从 EL2 返回 EL1 时：
  SP_EL2 bank ← 当前 xregs[31]（EL2 用的 SP）
  xregs[31] ← SP_EL1 bank（恢复 EL1 的 SP）
```

---

## 8. TBI 地址处理

### 8.1 Top-Byte Ignore

ERET 返回时需要处理 TBI（Tag-Based Addressing）：

```c
tbii = EX_TBFLAG_A64(env->hflags, TBII);
if ((tbii >> extract64(new_pc, 55, 1)) & 1) {
    // TBI enabled for this address range
    if (regime_has_2_ranges(mmu_idx)) {
        new_pc = sextract64(new_pc, 0, 56);  // 符号扩展bit[55]
    } else {
        new_pc = extract64(new_pc, 0, 56);   // 零扩展
    }
}
env->pc = new_pc;
```

这确保返回地址的高字节被正确处理：
- 用户空间地址（bit[55]=0）：高位清零
- 内核空间地址（bit[55]=1）：高位符号扩展为全1

---

## 9. FEAT_NV 对 ERET 的影响

### 9.1 ERET Trap（EL1→EL2）

当 Guest OS 在 EL1 执行 ERET（试图模拟 hypervisor 返回）时：

```c
// translate-a64.c:1961
if (s->trap_eret) {
    gen_exception_insn_el(s, 0, EXCP_UDEF, syn_erettrap(0), 2);
    return true;
}
```

`trap_eret` 在 FEAT_NV/NV2 下由 `HCR_EL2.NV` 设置，
让 outer hypervisor 可以拦截 guest hypervisor 的 ERET。

### 9.2 FEAT_NV 的 SPSR 修改

异常入口时，如果 NV 使能且 cur_el==1 且 new_el==1：

```c
if ((hcr & (HCR_NV | HCR_NV1 | HCR_NV2)) == HCR_NV ||
    (hcr & (HCR_NV | HCR_NV2)) == (HCR_NV | HCR_NV2)) {
    // 修改 SPSR 报告 EL2（让 guest hypervisor 以为它在 EL2）
    old_mode = deposit64(old_mode, 2, 2, 2);  // M[3:2] = 0b10 = EL2
}
```

---

## 10. 异常入口与返回的完整对称性

### 10.1 EL1→EL2→EL1 完整流程（以 SMC trap 为例）

```
═══════════════════════════════════════════════════════════
 阶段一：EL1 执行 SMC，trap 到 EL2
═══════════════════════════════════════════════════════════

EL1 状态：
  PC = 0xFFFF_0000_1000        (SMC 指令地址)
  PSTATE = {EL1, SP1, DAIF=0000, PAN=1, ...}
  SP_EL1 = 0xFFFF_8000_0000

         │ HCR_EL2.TSC = 1
         ▼

arm_cpu_do_interrupt_aarch64():
  ① SPSR_EL2 ← 0x...（保存 EL1 的 PSTATE）
  ② ELR_EL2  ← 0xFFFF_0000_1000（SMC 地址）
  ③ ESR_EL2  ← EC=0x17, ISS=imm（SMC syndrome）
  ④ SP_EL1 bank ← SP（保存 EL1 的栈指针）
  ⑤ PSTATE ← {EL2, SP1, DAIF=1111, PAN=0, TCO=1}
  ⑥ SP ← SP_EL2 bank（恢复 EL2 的栈指针）
  ⑦ PC ← VBAR_EL2 + 0x400（Lower EL AArch64 Sync）

═══════════════════════════════════════════════════════════
 阶段二：EL2 处理 trap（hypervisor 代码）
═══════════════════════════════════════════════════════════

EL2 handler 执行：
  - 读 ESR_EL2 判断是 SMC trap
  - 读 ELR_EL2 获取 SMC 地址
  - 修改 ELR_EL2 += 4（跳过 SMC，返回到下一条指令）
  - 处理请求
  - 执行 ERET

═══════════════════════════════════════════════════════════
 阶段三：ERET 从 EL2 返回 EL1
═══════════════════════════════════════════════════════════

HELPER(exception_return)(env, ELR_EL2):
  ① spsr = SPSR_EL2  （读取保存的 PSTATE）
  ② new_el = el_from_spsr(spsr) → 1
  ③ 合法性检查通过
  ④ pstate_write(env, spsr & valid_mask)  （恢复 PSTATE）
  ⑤ aarch64_restore_sp(env, 1)  （恢复 SP_EL1）
  ⑥ env->pc = ELR_EL2  （恢复 PC）

EL1 状态恢复：
  PC = 0xFFFF_0000_1004        (SMC 下一条)
  PSTATE = {EL1, SP1, DAIF=0000, PAN=1, ...}（完全恢复）
  SP = SP_EL1                  (恢复 EL1 栈指针)
```

### 10.2 入口/返回操作对称表

| 操作 | 入口 (→EL2) | 返回 (→EL1) |
|------|-------------|-------------|
| PSTATE | → SPSR_EL2 | SPSR_EL2 → PSTATE |
| PC | → ELR_EL2 | ELR_EL2 → PC |
| SP | save_sp(EL1), restore_sp(EL2) | save_sp(EL2), restore_sp(EL1) |
| EL | 1 → 2 | 2 → 1 |
| Exclusive | 不清除 | arm_clear_exclusive() |
| SVE VL | sve_change_el(1→2) | sve_change_el(2→1) |
| DAIF | 强制 0b1111 | 恢复旧值 |
| IL | 清除 | 恢复旧值 |
| hooks | pre_el_change + el_change | pre_el_change + el_change |

---

## 11. 与规范的一致性评估

### 11.1 实现正确的部分

| 方面 | 状态 | 说明 |
|------|:----:|------|
| SPSR 保存/恢复 | ✅ | 完整保存所有有效 PSTATE 位 |
| ELR 保存/恢复 | ✅ | 正确的 PC 值 |
| SP 切换 | ✅ | save_sp/restore_sp 对称 |
| Illegal return | ✅ | 所有条件检查完整 |
| FEAT_NV ERET trap | ✅ | 翻译时检查 trap_eret |
| FEAT_GCS EXLOCK | ✅ | GCSCR.EXLOCKEN 检查 |
| FEAT_RME 安全状态 | ✅ | SCR_NSE 检查 |
| TBI 处理 | ✅ | 根据 TBII 正确扩展地址 |
| SVE VL 切换 | ✅ | aarch64_sve_change_el |
| PSTATE valid mask | ✅ | 按 CPU feature 过滤 |
| Debug SS | ✅ | 入口/返回都正确处理 |

### 11.2 已知简化

| 方面 | 说明 | 严重度 |
|------|------|:------:|
| PSTATE.EXLOCK 完整语义 | FEAT_GCS 相对较新，边界情况可能不完整 | P3 |
| ERET timing | 真实硬件可能有 pipeline flush 延迟 | P3 |
| Branch prediction | ERET 后的分支预测器状态无模拟意义 | — |

### 11.3 结论

ERET 实现是 QEMU ARM64 中**质量最高的子系统之一**：
- 所有 illegal return 条件全部覆盖
- PSTATE 过滤使用 feature-aware mask
- 支持最新特性（FEAT_NV/NV2/GCS/RME）
- EL 切换 hooks 保证了设备模型一致性
