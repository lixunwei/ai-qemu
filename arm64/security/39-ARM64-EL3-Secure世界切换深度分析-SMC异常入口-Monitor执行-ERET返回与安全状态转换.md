# 39 - ARM64 EL3/Secure 世界切换深度分析 — SMC 异常入口、Monitor 执行环境、ERET 返回与安全状态转换

> **基于 QEMU 11.0.50 源码**，深入分析 ARM64 EL3（Monitor）异常级别在 QEMU 中的完整实现：
> SMC 指令翻译与陷阱决策、异常进入 EL3 的 PSTATE/SPSR/ELR 保存与向量偏移、
> EL3 执行环境（SCR_EL3、hflags、系统寄存器）、ERET 返回路径与安全状态验证、
> Secure/NonSecure 世界切换机制（TLB 刷新、地址空间切换、寄存器分组）、PSCI 固件接口。

---

## 目录

1. [SMC 指令翻译与预处理](#1-smc-指令翻译与预处理)
2. [SMC 陷阱决策表](#2-smc-陷阱决策表)
3. [异常进入 EL3](#3-异常进入-el3)
4. [向量偏移计算](#4-向量偏移计算)
5. [PSTATE 变化与状态保存](#5-pstate-变化与状态保存)
6. [EL3 系统寄存器](#6-el3-系统寄存器)
7. [SCR_EL3 写入与特性门控](#7-scr_el3-写入与特性门控)
8. [安全状态判定](#8-安全状态判定)
9. [EL3 执行环境与 hflags](#9-el3-执行环境与-hflags)
10. [ERET 从 EL3 返回](#10-eret-从-el3-返回)
11. [安全状态转换：TLB 刷新](#11-安全状态转换tlb-刷新)
12. [安全内存访问与地址空间](#12-安全内存访问与地址空间)
13. [PSCI 固件接口](#13-psci-固件接口)
14. [EL3 特有的陷阱控制](#14-el3-特有的陷阱控制)
15. [完整数据流](#15-完整数据流)

---

## 1. SMC 指令翻译与预处理

### 1.1 TCG 翻译

```c
// translate-a64.c:3193-3204
static bool trans_SMC(DisasContext *s, arg_i *a)
{
    if (s->current_el == 0) {
        unallocated_encoding(s);    // SMC 在 EL0 不可用
        return true;
    }
    gen_a64_update_pc(s, 0);
    // ① 调用 pre_smc helper 进行陷阱决策
    gen_helper_pre_smc(tcg_env, tcg_constant_i32(syn_aa64_smc(a->imm)));
    gen_ss_advance(s);
    // ② 如果 pre_smc 没有异常 → 生成 SMC 异常到 EL3
    gen_exception_insn_el(s, 4, EXCP_SMC, syn_aa64_smc(a->imm), 3);
    return true;
}
```

**关键设计**：SMC 翻译分两步：
1. `pre_smc` helper 检查是否应该 UNDEF 或陷入 EL2
2. 只有通过检查后才真正生成 `EXCP_SMC` 异常到 EL3

### 1.2 pre_smc Helper

```c
// op_helper.c:1111-1200
void HELPER(pre_smc)(CPUARMState *env, uint32_t syndrome)
{
    int cur_el = arm_current_el(env);
    bool secure = arm_is_secure(env);
    bool smd_flag = env->cp15.scr_el3 & SCR_SMD;

    // AArch64 SMD 对 Secure 和 NonSecure 都生效
    // AArch32 SMD 仅对 NonSecure 生效
    bool smd = arm_feature(env, ARM_FEATURE_AARCH64)
                   ? smd_flag : smd_flag && !secure;

    // ① 无 EL3 且无 NV 且非 PSCI-via-SMC → UNDEF
    if (!arm_feature(env, ARM_FEATURE_EL3) &&
        !(arm_hcr_el2_eff(env) & HCR_NV) &&
        cpu->psci_conduit != QEMU_PSCI_CONDUIT_SMC) {
        raise_exception(env, EXCP_UDEF, ...);
    }

    // ② EL1 + HCR_EL2.TSC → 陷入 EL2（优先于 SMD）
    if (cur_el == 1 && (arm_hcr_el2_eff(env) & HCR_TSC)) {
        raise_exception(env, EXCP_HYP_TRAP, syndrome, 2);
    }

    // ③ 非 PSCI 调用 + (SMD 或无 EL3) → UNDEF
    if (!arm_is_psci_call(cpu, EXCP_SMC) &&
        (smd || !arm_feature(env, ARM_FEATURE_EL3))) {
        raise_exception(env, EXCP_UDEF, ...);
    }

    // 通过所有检查 → 返回，后续生成 EXCP_SMC
}
```

---

## 2. SMC 陷阱决策表

### 有 EL3 且 SMD=0

| 条件 | HCR_TSC=1 且 NS EL1 | 其他情况 |
|------|---------------------|----------|
| PSCI conduit=SMC，有效调用 | 陷入 EL2 | **PSCI 调用** |
| PSCI conduit=SMC，无效调用 | 陷入 EL2 | **陷入 EL3** |
| Conduit 非 SMC | 陷入 EL2 | **陷入 EL3** |

### 有 EL3 且 SMD=1

| 条件 | HCR_TSC=1 且 NS EL1 | 其他情况 |
|------|---------------------|----------|
| PSCI conduit=SMC，有效调用 | 陷入 EL2 | **PSCI 调用** |
| PSCI conduit=SMC，无效调用 | 陷入 EL2 | **UNDEF** |
| Conduit 非 SMC | 陷入 EL2 | **UNDEF** |

### 无 EL3

| 条件 | HCR_TSC=1 且 NS EL1 | 其他情况 |
|------|---------------------|----------|
| PSCI conduit=SMC，有效调用 | 陷入 EL2 | **PSCI 调用** |
| PSCI conduit=SMC，无效调用 | 陷入 EL2 | **UNDEF** |
| Conduit 非 SMC | UNDEF 或 NV 陷入 EL2 | **UNDEF** |

---

## 3. 异常进入 EL3

```c
// helper.c:9198-9427
static void arm_cpu_do_interrupt_aarch64(CPUState *cs)
{
    unsigned int new_el = env->exception.target_el;  // = 3
    vaddr addr = env->cp15.vbar_el[new_el];          // VBAR_EL3
    uint64_t new_mode = aarch64_pstate_mode(new_el, true); // EL3h

    // SVE 状态在 EL 切换时可能需要调整
    aarch64_sve_change_el(env, cur_el, new_el, is_a64(env));

    // ... 向量偏移计算（见第 4 节）
    // ... SPSR/ELR 保存（见第 5 节）
    // ... PSTATE 设置（见第 5 节）

    // 最终设置
    pstate_write(env, PSTATE_DAIF | new_mode);  // 屏蔽所有中断
    env->aarch64 = true;                         // EL3 始终 AArch64
    aarch64_restore_sp(env, new_el);             // 切换到 SP_EL3

    arm_rebuild_hflags(env);                     // 重建 hflags
    env->pc = addr;                              // 跳转到向量
}
```

---

## 4. 向量偏移计算

```c
// helper.c:9217-9255
if (cur_el < new_el) {
    // 从低 EL 进入 EL3
    switch (new_el) {
    case 3:
        // SCR_EL3.RW 决定低 EL 使用 AArch64 还是 AArch32
        is_aa64 = arm_scr_rw_eff(env);
        break;
    }
    if (is_aa64) {
        addr += 0x400;     // 低 EL 为 AArch64：偏移 +0x400
    } else {
        addr += 0x600;     // 低 EL 为 AArch32：偏移 +0x600
    }
} else {
    // 同级 EL（EL3 → EL3）
    if (pstate_read(env) & PSTATE_SP) {
        addr += 0x200;     // 使用 SP_ELx：偏移 +0x200
    }
    // 使用 SP_EL0：偏移 +0x000
}
```

**VBAR_EL3 向量表布局**：

| 偏移 | 来源 EL | 条件 | 异常类型 |
|------|---------|------|----------|
| +0x000 | 同级 EL3，SP_EL0 | Synchronous | SMC from EL3 |
| +0x080 | 同级 EL3，SP_EL0 | IRQ | |
| +0x100 | 同级 EL3，SP_EL0 | FIQ | |
| +0x180 | 同级 EL3，SP_EL0 | SError | |
| +0x200 | 同级 EL3，SP_EL3 | Synchronous | |
| +0x280 | 同级 EL3，SP_EL3 | IRQ | |
| +0x400 | 低 EL，AArch64 | Synchronous | **SMC from EL1/EL2** |
| +0x480 | 低 EL，AArch64 | IRQ | |
| +0x600 | 低 EL，AArch32 | Synchronous | SMC from AArch32 |
| +0x680 | 低 EL，AArch32 | IRQ | |

---

## 5. PSTATE 变化与状态保存

### 5.1 状态保存

```c
// helper.c:9343-9373
if (is_a64(env)) {
    old_mode = pstate_read(env);          // 保存当前 PSTATE
    aarch64_save_sp(env, cur_el);         // 保存当前 SP 到 SP_ELn
    env->elr_el[new_el] = env->pc;        // ELR_EL3 = 返回地址
} else {
    old_mode = cpsr_read_for_spsr_elx(env); // AArch32 CPSR → SPSR 格式
    env->elr_el[new_el] = env->regs[15];   // ELR_EL3 = AArch32 PC
    aarch64_sync_32_to_64(env);             // 同步 AArch32 寄存器到 AArch64 视图
}
env->banked_spsr[aarch64_banked_spsr_index(new_el)] = old_mode;
// SPSR_EL3 = old_mode
```

### 5.2 PSTATE 设置

```c
// helper.c:9375-9417
// PAN 处理：保留旧 PAN，EL1 时可能设置为 1
new_mode |= old_mode & PSTATE_PAN;

// TCO（Tag Check Override）：进入异常时设置
if (cpu_isar_feature(aa64_mte, cpu)) {
    new_mode |= PSTATE_TCO;
}

// SSBS（Speculative Store Bypass Safe）
if (cpu_isar_feature(aa64_ssbs, cpu)) {
    if (env->cp15.sctlr_el[new_el] & SCTLR_DSSBS_64) {
        new_mode |= PSTATE_SSBS;
    }
}

// ALLINT（NMI 屏蔽）
if (cpu_isar_feature(aa64_nmi, cpu)) {
    if (!(env->cp15.sctlr_el[new_el] & SCTLR_SPINTMASK)) {
        new_mode |= PSTATE_ALLINT;
    }
}

// 最终写入
pstate_write(env, PSTATE_DAIF | new_mode);
// PSTATE.DAIF = 1111（屏蔽所有中断）
// PSTATE.EL = 3
// PSTATE.SP = 1（使用 SP_EL3）
```

### 5.3 进入 EL3 的 PSTATE 关键变化

| 字段 | 进入 EL3 后的值 | 说明 |
|------|----------------|------|
| PSTATE.EL | 3 | 最高异常级别 |
| PSTATE.SP | 1 (SP_EL3) | 使用 EL3 专用栈指针 |
| PSTATE.D | 1 | Debug 异常屏蔽 |
| PSTATE.A | 1 | SError 屏蔽 |
| PSTATE.I | 1 | IRQ 屏蔽 |
| PSTATE.F | 1 | FIQ 屏蔽 |
| PSTATE.PAN | 保留/设置 | 取决于 SCTLR_EL3.SPAN |
| PSTATE.TCO | 1 | Tag Check Override |
| PSTATE.nRW | 0 | AArch64 执行状态 |

---

## 6. EL3 系统寄存器

```c
// helper.c:4360-4410（EL3 专用寄存器定义）
{ "SCR_EL3",     opc0=3, opc1=6, crn=1, crm=1, opc2=0, PL3_RW }
{ "SDER32_EL3",  opc0=3, opc1=6, crn=1, crm=1, opc2=1, PL3_RW }
{ "TTBR0_EL3",   opc0=3, opc1=6, crn=2, crm=0, opc2=0, PL3_RW }
{ "TCR_EL3",     opc0=3, opc1=6, crn=2, crm=0, opc2=2, PL3_RW }
{ "ELR_EL3",     opc0=3, opc1=6, crn=4, crm=0, opc2=1, PL3_RW }
{ "SPSR_EL3",    opc0=3, opc1=6, crn=4, crm=0, opc2=0, PL3_RW }
{ "ESR_EL3",     opc0=3, opc1=6, crn=5, crm=2, opc2=0, PL3_RW }
{ "FAR_EL3",     opc0=3, opc1=6, crn=6, crm=0, opc2=0, PL3_RW }
{ "VBAR_EL3",    opc0=3, opc1=6, crn=12, crm=0, opc2=0, PL3_RW }
{ "CPTR_EL3",    opc0=3, opc1=6, crn=1, crm=1, opc2=2, PL3_RW }
{ "TPIDR_EL3",   opc0=3, opc1=6, crn=13, crm=0, opc2=2, PL3_RW }
```

所有 EL3 寄存器使用 `opc1=6` 编码，仅 `PL3_RW` 可访问。低异常级别访问会产生 UNDEF 或被陷阱拦截。

---

## 7. SCR_EL3 写入与特性门控

```c
// helper.c:712-836
static void scr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    uint64_t valid_mask = 0x3fff;  // 基础 v8.0 掩码

    if (arm_el_is_aa64(env, 3)) {
        value |= SCR_FW | SCR_AW;     // RES1 位

        // 无 AArch32 支持 → SCR_RW 强制为 1
        if (!aa64_aa32_el1 && !aa64_aa32_el2) {
            value |= SCR_RW;           // 低 EL 只能 AArch64
        }

        // 按 CPU 特性添加有效位：
        // FEAT_RAS     → SCR_TERR
        // FEAT_LOR     → SCR_TLOR
        // FEAT_PAuth   → SCR_API | SCR_APK
        // FEAT_SEL2    → SCR_EEL2
        // FEAT_MTE     → SCR_ATA
        // FEAT_RME     → SCR_NSE | SCR_GPF
        // FEAT_HCX     → SCR_HXEN
        // FEAT_FGT     → SCR_FGTEN
        // FEAT_GCS     → SCR_GCSEN
        // FEAT_SME     → SCR_ENTP2
        // ... 等 20+ 特性门控
    }

    // 无 EL2 → 移除 HCE/SMD 位
    if (!arm_feature(env, ARM_FEATURE_EL2)) {
        valid_mask &= ~SCR_HCE;
    }

    value &= valid_mask;
    changed = env->cp15.scr_el3 ^ value;
    env->cp15.scr_el3 = value;

    // *** 安全状态切换时刷新 TLB ***
    if (changed & (SCR_NS | SCR_NSE)) {
        tlb_flush_by_mmuidx(env_cpu(env),
            ARMMMUIdxBit_E10_0 | ARMMMUIdxBit_E10_1 |
            ARMMMUIdxBit_E10_1_PAN | ARMMMUIdxBit_E20_0 |
            ARMMMUIdxBit_E20_2 | ARMMMUIdxBit_E20_2_PAN |
            ARMMMUIdxBit_E2 | ...);  // 所有 EL3 以下的 12 种 MMU 索引
    }
}
```

### SCR_EL3 关键位

| 位 | 名称 | 作用 |
|----|------|------|
| 0 | NS | Non-Secure 位（0=Secure, 1=NonSecure） |
| 1 | IRQ | IRQ 路由到 EL3 |
| 2 | FIQ | FIQ 路由到 EL3 |
| 3 | EA | SError 路由到 EL3 |
| 7 | SMD | SMC 指令禁用 |
| 8 | HCE | HVC 使能 |
| 10 | RW | 低 EL 执行宽度（0=AArch32, 1=AArch64） |
| 11 | ST | Secure Timer 对 EL1 可见 |
| 13 | TWE | WFE 陷入 EL3 |
| 14 | TWI | WFI 陷入 EL3 |
| 21 | EEL2 | Secure EL2 使能 |
| 35 | NSE | Non-Secure Extension（RME：NSE:NS 编码四域） |

---

## 8. 安全状态判定

```c
// helper.c:10131-10161
ARMSecuritySpace arm_security_space(CPUARMState *env)
{
    if (!arm_feature(env, ARM_FEATURE_EL3)) {
        return ARMSS_NonSecure;      // 无 EL3 → 默认 NonSecure
    }

    // 当前在 EL3？
    if (is_a64(env) && extract32(env->pstate, 2, 2) == 3) {
        if (cpu_isar_feature(aa64_rme, cpu)) {
            return ARMSS_Root;       // RME → Root 域
        } else {
            return ARMSS_Secure;     // 标准 → Secure 域
        }
    }

    return arm_security_space_below_el3(env);
}

// helper.c:10163-10187
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env)
{
    if (!(env->cp15.scr_el3 & SCR_NS)) {
        return ARMSS_Secure;         // NS=0 → Secure
    } else if (env->cp15.scr_el3 & SCR_NSE) {
        return ARMSS_Realm;          // NS=1, NSE=1 → Realm
    } else {
        return ARMSS_NonSecure;      // NS=1, NSE=0 → NonSecure
    }
}
```

### ARMSecuritySpace 四域模型

```
// arm-security.h:18-23
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,  // 安全世界
    ARMSS_NonSecure  = 1,  // 非安全世界
    ARMSS_Root       = 2,  // Root 域（RME，EL3 专用）
    ARMSS_Realm      = 3,  // Realm 域（RME）
} ARMSecuritySpace;
```

**NSE:NS 编码**：

| NSE | NS | 安全域 |
|-----|-----|--------|
| 0 | 0 | Secure |
| 0 | 1 | NonSecure |
| 1 | 0 | （保留/非法） |
| 1 | 1 | Realm |

---

## 9. EL3 执行环境与 hflags

### 9.1 hflags 重建

```c
// hflags.c:506-524
static CPUARMTBFlags rebuild_hflags_internal(CPUARMState *env)
{
    int el = arm_current_el(env);               // = 3
    int fp_el = fp_exception_el(env, el);
    ARMMMUIdx mmu_idx = arm_mmu_idx_el(env, el); // = ARMMMUIdx_E3

    return rebuild_hflags_a64(env, el, fp_el, mmu_idx);
}
```

### 9.2 EL3 的 TB 标志特点

```c
// hflags.c:240-330
static CPUARMTBFlags rebuild_hflags_a64(CPUARMState *env, int el, ...)
{
    // 通用标志：
    DP_TBFLAG_ANY(flags, AARCH64_STATE, 1);
    // TBI 从 TCR_EL3 获取
    tbid = aa64_va_parameter_tbi(tcr, mmu_idx);
    // E2H 标志（EL3 不适用，为 0）
    // SVE/SME 异常级别
    // SCTLR_EL3 的对齐、大端序设置
    // PAuth 活跃状态
}
```

**EL3 与 EL1 的 hflags 差异**：

| 标志 | EL3 | EL1 | 说明 |
|------|-----|-----|------|
| MMUIDX | E3 | E10_1 | 不同翻译体制 |
| E2H | 0 | 取决于 HCR_EL2 | VHE 无关 |
| HCR_EL2 效果 | 无 | 影响陷阱/路由 | EL3 不受 HCR 控制 |
| 安全状态 | Root/Secure | 取决于 SCR_NS | EL3 始终在安全/Root 域 |
| TCR 来源 | TCR_EL3 | TCR_EL1 | 独立翻译配置 |
| TTBR | TTBR0_EL3（仅一个） | TTBR0/1_EL1 | EL3 单范围 |

### 9.3 TB 隔离

QEMU TCG 通过 hflags 中编码的 `current_el` 和 `mmu_idx` 确保不同 EL 的 Translation Block（TB）完全隔离。EL3 的 TB 使用 `ARMMMUIdx_E3`，与 EL1 的 `ARMMMUIdx_E10_1` 不同，因此不会混用。

---

## 10. ERET 从 EL3 返回

### 10.1 ERET 翻译

```c
// translate-a64.c:1951-1974
static bool trans_ERET(DisasContext *s, arg_ERET *a)
{
    if (s->current_el == 0) return false;       // EL0 不可用
    if (s->trap_eret) {                         // NV 陷阱
        gen_exception_insn_el(s, 0, EXCP_UDEF, syn_erettrap(0), 2);
        return true;
    }
    // 加载 ELR_ELn
    tcg_gen_ld_i64(dst, tcg_env, offsetof(CPUARMState, elr_el[s->current_el]));
    gen_helper_exception_return(tcg_env, dst);
    s->base.is_jmp = DISAS_EXIT;               // 必须退出 TB
    return true;
}
```

### 10.2 ERET Helper

```c
// helper-a64.c:646-785
void HELPER(exception_return)(CPUARMState *env, uint64_t new_pc)
{
    int cur_el = arm_current_el(env);  // = 3
    uint32_t spsr = env->banked_spsr[aarch64_banked_spsr_index(cur_el)];
    int new_el = el_from_spsr(spsr);   // 从 SPSR_EL3 提取目标 EL

    // ① 合法性检查
    if (new_el > cur_el) goto illegal_return;     // 不可返回到更高 EL
    if (new_el == 2 && !arm_is_el2_enabled(env))  // EL2 未使能
        goto illegal_return;

    // ② RME 安全状态检查
    if (cur_el == 3 && new_el < 3 &&
        (env->cp15.scr_el3 & (SCR_NS | SCR_NSE)) == SCR_NSE) {
        goto illegal_return;           // NSE=1,NS=0 是非法状态
    }

    // ③ 执行宽度匹配
    if (new_el != 0 && arm_el_is_aa64(env, new_el) != return_to_aa64)
        goto illegal_return;

    // ④ HCR_EL2.TGE 检查
    if (new_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE))
        goto illegal_return;

    // ⑤ 执行返回
    arm_call_pre_el_change_hook(cpu);

    if (return_to_aa64) {
        env->aarch64 = true;
        pstate_write(env, spsr);               // 恢复 PSTATE
        aarch64_restore_sp(env, new_el);       // 恢复 SP_ELn
        helper_rebuild_hflags_a64(env, new_el); // 重建 hflags
        // TBI 处理：根据新 EL 的 TBI 设置调整 PC
        env->pc = new_pc;
    } else {
        env->aarch64 = false;
        cpsr_write_from_spsr_elx(env, spsr);   // 恢复 CPSR
        aarch64_sync_64_to_32(env);            // 同步寄存器
        helper_rebuild_hflags_a32(env, new_el);
    }

    aarch64_sve_change_el(env, cur_el, new_el, return_to_aa64);
    arm_call_el_change_hook(cpu);

illegal_return:
    // 非法返回：设置 PSTATE.IL，PC = ELR，不切换 EL
    env->pstate |= PSTATE_IL;
    env->pc = new_pc;
    helper_rebuild_hflags_a64(env, cur_el);
}
```

### 10.3 ERET 安全状态转换

当 EL3 固件修改 `SCR_EL3.NS` 后执行 ERET：
1. `scr_write()` 已经刷新了所有 EL3 以下的 TLB
2. ERET 恢复 PSTATE，切换到目标 EL（如 EL1）
3. `helper_rebuild_hflags_a64()` 使用新的 `arm_mmu_idx_el()` — 此时安全状态由 `SCR_EL3.NS` 决定
4. 新 EL 运行在 Secure 或 NonSecure 世界

---

## 11. 安全状态转换：TLB 刷新

```c
// helper.c:818-835
if (changed & (SCR_NS | SCR_NSE)) {
    tlb_flush_by_mmuidx(env_cpu(env),
        ARMMMUIdxBit_E10_0      |   // EL0（EL1&0 体制）
        ARMMMUIdxBit_E10_0_GCS  |
        ARMMMUIdxBit_E20_0      |   // EL0（EL2&0 体制 VHE）
        ARMMMUIdxBit_E20_0_GCS  |
        ARMMMUIdxBit_E10_1      |   // EL1
        ARMMMUIdxBit_E10_1_PAN  |   // EL1 PAN
        ARMMMUIdxBit_E10_1_GCS  |
        ARMMMUIdxBit_E20_2      |   // EL2（VHE）
        ARMMMUIdxBit_E20_2_PAN  |
        ARMMMUIdxBit_E20_2_GCS  |
        ARMMMUIdxBit_E2         |   // EL2
        ARMMMUIdxBit_E2_GCS);
}
```

**为什么刷新所有 EL3 以下的 TLB**：
- 安全状态变化意味着低 EL 将使用不同的地址空间
- Secure 和 NonSecure 使用不同的物理地址空间
- 旧的 TLB 条目可能指向错误的地址空间
- EL3 自身的 TLB（`ARMMMUIdx_E3`）不需要刷新 — EL3 始终在 Root/Secure

---

## 12. 安全内存访问与地址空间

### 12.1 MemTxAttrs 安全标记

```c
// cpu.h:2601-2613
// MemTxAttrs.secure 控制地址空间选择
// secure = true → ARMASIdx_S → address_space_secure
// secure = false → ARMASIdx_NS → address_space_memory
```

### 12.2 页表遍历中的安全标记

```c
// ptw.c:752-756, 790-794
// 页表描述符加载时使用 attrs.secure = arm_space_is_secure(ptw->out_space)
// 然后通过 arm_addressspace(cs, attrs) 选择正确的地址空间
```

### 12.3 地址空间映射

| 安全域 | address_space | 物理 MMU 索引 |
|--------|--------------|--------------|
| Secure | address_space_secure | Phys_S |
| NonSecure | address_space_memory | Phys_NS |
| Root | — | Phys_Root |
| Realm | — | Phys_Realm |

---

## 13. PSCI 固件接口

```c
// psci.c:30-56
bool arm_is_psci_call(ARMCPU *cpu, int excp_type)
{
    // 根据 CPU 配置的 psci_conduit 判断
    switch (excp_type) {
    case EXCP_HVC:
        return cpu->psci_conduit == QEMU_PSCI_CONDUIT_HVC;
    case EXCP_SMC:
        return cpu->psci_conduit == QEMU_PSCI_CONDUIT_SMC;
    }
    return false;
}

// psci.c:58-120
void arm_handle_psci_call(ARMCPU *cpu)
{
    // 从 X0-X3（AArch64）或 R0-R3（AArch32）读取参数
    param[i] = is_a64(env) ? env->xregs[i] : env->regs[i];

    switch (param[0]) {
    case PSCI_VERSION:      ret = PSCI_VERSION_1_1; break;
    case PSCI_AFFINITY_INFO: // 查询 CPU 电源状态
    case PSCI_CPU_ON:       // 启动次级 CPU
        // 查找目标 CPU，设置入口点，启动
    case PSCI_CPU_OFF:      // 关闭当前 CPU
    case PSCI_SYSTEM_RESET: // 系统复位
    case PSCI_SYSTEM_OFF:   // 系统关机
    }
}
```

**PSCI 旁路机制**：
- 在 `pre_smc` 中 `arm_is_psci_call()` 在陷阱决策前检查
- PSCI 调用不走正常的 SMC → EL3 异常路径
- 直接由 QEMU 内部处理（模拟固件行为）
- 这使得无 EL3 固件的虚拟机也能使用 PSCI

---

## 14. EL3 特有的陷阱控制

SCR_EL3 提供多个位来控制低 EL 的指令陷入 EL3：

| SCR_EL3 位 | 作用 |
|------------|------|
| SMD | SMC 指令禁用（UNDEF 而非陷入 EL3） |
| TERR | RAS 错误记录寄存器访问陷入 EL3 |
| TLOR | LORegion 寄存器陷入 EL3 |
| API/APK | PAuth 密钥/指令陷入 EL3 |
| ATA | MTE 标签访问陷入 EL3 |
| ENSCXT | SCXTNUM 寄存器陷入 EL3 |
| HXEN | HCRX_EL2 扩展使能 |
| FGTEN | 细粒度陷阱使能 |
| PIEN | 权限间接使能 |

```c
// helper.c:310-329
// access_trap_aa32s_el1: Secure EL1 访问 Monitor 模式寄存器
// → 如果 EL3 为 AArch64 → 陷入 EL3 (CP_ACCESS_TRAP_EL3)
// → 如果 EL3 为 AArch32 → 允许访问
```

---

## 15. 完整数据流

### 15.1 SMC 进入 EL3 完整路径

```
Guest 执行 SMC #imm（EL1）
  │
  ▼
trans_SMC()                             [translate-a64.c:3193]
  ├── EL0 检查 → UNDEF
  ├── gen_helper_pre_smc()              陷阱决策
  │     ├── 无 EL3 + 非 PSCI → UNDEF
  │     ├── HCR_TSC + NS EL1 → 陷入 EL2
  │     ├── SMD + 非 PSCI → UNDEF
  │     └── 通过 → 继续
  └── gen_exception_insn_el(EXCP_SMC, 3)
  │
  ▼
arm_cpu_do_interrupt_aarch64()          [helper.c:9198]
  ├── VBAR_EL3 基地址
  ├── 向量偏移：+0x400（AArch64 低 EL）或 +0x600（AArch32）
  ├── SPSR_EL3 = 旧 PSTATE
  ├── ELR_EL3 = 返回地址
  ├── ESR_EL3 = SMC syndrome
  ├── PSTATE = DAIF=1111, EL=3, SP=1, nRW=0
  ├── aarch64_restore_sp(EL3) → SP = SP_EL3
  ├── arm_rebuild_hflags() → mmu_idx = E3
  └── PC = VBAR_EL3 + offset
  │
  ▼
EL3 固件执行（Secure Monitor）
  ├── 读取 ESR_EL3 判断异常原因
  ├── 处理安全服务请求
  ├── 修改 SCR_EL3.NS → 切换世界
  │     └── scr_write() → TLB 刷新 12 种 MMU 索引
  └── ERET 返回
  │
  ▼
HELPER(exception_return)()              [helper-a64.c:646]
  ├── SPSR_EL3 → 目标 EL + PSTATE
  ├── RME 安全状态验证
  ├── 执行宽度匹配检查
  ├── pstate_write(spsr) → 恢复 PSTATE
  ├── aarch64_restore_sp(new_el) → 恢复栈指针
  ├── helper_rebuild_hflags_a64(new_el)
  │     └── mmu_idx = arm_mmu_idx_el(new_el)
  │           → 基于 SCR_NS 确定安全状态
  ├── TBI 处理
  └── PC = ELR_EL3
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `target/arm/tcg/translate-a64.c` | ~3205 | trans_SMC (3193-3204)、trans_ERET (1951-1974) |
| `target/arm/tcg/op_helper.c` | ~1200 | HELPER(pre_smc) (1111-1200)：SMC 陷阱决策表 |
| `target/arm/helper.c` | ~10190 | scr_write (712-836)：SCR_EL3 写入/特性门控/TLB 刷新；arm_cpu_do_interrupt_aarch64 (9198-9427)：异常进入；arm_security_space (10131-10161)：安全状态判定；EL3 系统寄存器 (4360-4410) |
| `target/arm/tcg/helper-a64.c` | ~785 | HELPER(exception_return) (646-785)：ERET 完整路径 |
| `target/arm/tcg/hflags.c` | ~575 | rebuild_hflags_internal (506-524)、rebuild_hflags_a64 (240-330)：TB 标志重建 |
| `target/arm/tcg/psci.c` | ~120 | arm_is_psci_call (30-56)、arm_handle_psci_call (58-120)：PSCI 固件接口 |
| `include/hw/arm/arm-security.h` | ~35 | ARMSecuritySpace 枚举 (18-23) |
| `target/arm/cpu.h` | ~2613 | MemTxAttrs.secure 地址空间选择 (2601-2613) |
