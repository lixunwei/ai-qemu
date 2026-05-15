# ARM64 Exception Level 状态管理与指令执行变化深度分析

> **QEMU 版本**：11.0.50  
> **分析范围**：ARM64 Exception Level (EL0-EL3) 状态跟踪、EL 切换机制、安全状态管理、指令执行差异  
> **关键源文件**：`target/arm/helper.c`、`target/arm/internals.h`、`target/arm/cpu.h`、`target/arm/tcg/helper-a64.c`、`target/arm/tcg/op_helper.c`、`target/arm/tcg/translate-a64.c`、`target/arm/tcg/hflags.c`、`target/arm/mmuidx.h`

---

## 目录

1. [概述](#1-概述)
2. [PSTATE 与 EL 存储架构](#2-pstate-与-el-存储架构)
3. [arm_current_el() — 获取当前 EL](#3-arm_current_el--获取当前-el)
4. [EL 切换总览 — 异常进入与返回](#4-el-切换总览--异常进入与返回)
5. [异常进入：arm_cpu_do_interrupt](#5-异常进入arm_cpu_do_interrupt)
6. [AArch64 异常进入详细流程](#6-aarch64-异常进入详细流程)
7. [异常向量表与偏移计算](#7-异常向量表与偏移计算)
8. [PSTATE 设置与 DAIF 掩码](#8-pstate-设置与-daif-掩码)
9. [异常返回：ERET 实现](#9-异常返回eret-实现)
10. [SMC — EL1→EL3 切换](#10-smc--el1el3-切换)
11. [HVC — EL1→EL2 切换](#11-hvc--el1el2-切换)
12. [安全状态管理](#12-安全状态管理)
13. [SCR_EL3 — 安全配置寄存器](#13-scr_el3--安全配置寄存器)
14. [HCR_EL2 — 虚拟化配置寄存器](#14-hcr_el2--虚拟化配置寄存器)
15. [MMU 索引与 EL 映射](#15-mmu-索引与-el-映射)
16. [系统寄存器访问控制](#16-系统寄存器访问控制)
17. [特权指令行为差异](#17-特权指令行为差异)
18. [TB Flags 中的 EL 编码](#18-tb-flags-中的-el-编码)
19. [arm_rebuild_hflags — EL 变化后重建标志](#19-arm_rebuild_hflags--el-变化后重建标志)
20. [DisasContext 中的 EL 传播](#20-disascontext-中的-el-传播)
21. [PSTATE 特殊位对执行的影响](#21-pstate-特殊位对执行的影响)
22. [EL 切换对 SVE/SME 的影响](#22-el-切换对-svesme-的影响)
23. [arm_el_is_aa64 — EL 执行状态判断](#23-arm_el_is_aa64--el-执行状态判断)
24. [EL 切换完整时序图](#24-el-切换完整时序图)
25. [各 EL 指令执行差异对比表](#25-各-el-指令执行差异对比表)
26. [总结](#26-总结)

---

## 1. 概述

ARM64 架构定义了 4 个 Exception Level：

| EL | 用途 | 典型软件 |
|----|------|----------|
| EL0 | 用户态 | 应用程序 |
| EL1 | 内核态 | OS 内核 |
| EL2 | 虚拟化 | Hypervisor |
| EL3 | 安全监控 | Secure Monitor (ATF/TF-A) |

QEMU 中 EL 状态管理贯穿整个 CPU 模拟：
- **状态跟踪**：`CPUARMState.pstate` 位域中编码当前 EL
- **EL 切换**：通过异常进入（同步/异步）和异常返回（ERET）实现
- **指令差异**：不同 EL 可访问的系统寄存器、特权指令、地址翻译机制各不相同
- **TB 管理**：EL 变化导致 TB flags 不匹配，强制翻译块切换

---

## 2. PSTATE 与 EL 存储架构

### 2.1 CPUARMState 中的 PSTATE

**定义**：`target/arm/cpu.h:270-284`

```c
/* PSTATE isn't an architectural register for ARMv8. However, it is
 * convenient for us to assemble the underlying state into a 64 bit format
 * identical to the architectural format used for the SPSR. */
uint64_t pstate;
bool aarch64; /* True if CPU is in aarch64 state; inverse of PSTATE.nRW */
```

QEMU 将 PSTATE 的各部分分散存储以优化访问：

| 字段 | 存储位置 | 说明 |
|------|----------|------|
| NZCV | `env->CF/VF/NF/ZF` | 条件标志，分离存储避免位操作 |
| nRW (M[4]) | `env->aarch64`（取反） | AArch64/AArch32 状态 |
| DAIF | `env->daif` | 异常掩码 |
| BTYPE | `env->btype` | BTI 分支类型 |
| SM/ZA | `env->svcr` | SME 流模式/ZA 存储 |
| 其余位 | `env->pstate` | 直接存储在 pstate 字段 |

### 2.2 PSTATE 位定义

**定义**：`target/arm/cpu.h:1546-1571`

```c
#define PSTATE_SP    (1U)        /* bit 0: SP 选择 (SP_EL0 vs SP_ELx) */
#define PSTATE_M     (0xFU)      /* bit 0-3: Mode field */
#define PSTATE_nRW   (1U << 4)   /* bit 4: AArch32 = 1, AArch64 = 0 */
#define PSTATE_F     (1U << 6)   /* bit 6: FIQ mask */
#define PSTATE_I     (1U << 7)   /* bit 7: IRQ mask */
#define PSTATE_A     (1U << 8)   /* bit 8: SError mask */
#define PSTATE_D     (1U << 9)   /* bit 9: Debug mask */
#define PSTATE_SSBS  (1U << 12)  /* bit 12: Speculative Store Bypass Safe */
#define PSTATE_ALLINT (1U << 13) /* bit 13: All interrupt mask (FEAT_NMI) */
#define PSTATE_IL    (1U << 20)  /* bit 20: Illegal Execution State */
#define PSTATE_SS    (1U << 21)  /* bit 21: Software Step */
#define PSTATE_PAN   (1U << 22)  /* bit 22: Privileged Access Never */
#define PSTATE_UAO   (1U << 23)  /* bit 23: User Access Override */
#define PSTATE_DIT   (1U << 24)  /* bit 24: Data Independent Timing */
#define PSTATE_TCO   (1U << 25)  /* bit 25: Tag Check Override */
```

### 2.3 AArch64 Mode 值

**定义**：`target/arm/cpu.h:1574-1581`

```c
#define PSTATE_MODE_EL0t  0   /* EL0, 使用 SP_EL0 */
#define PSTATE_MODE_EL1t  4   /* EL1, 使用 SP_EL0 */
#define PSTATE_MODE_EL1h  5   /* EL1, 使用 SP_EL1 */
#define PSTATE_MODE_EL2t  8   /* EL2, 使用 SP_EL0 */
#define PSTATE_MODE_EL2h  9   /* EL2, 使用 SP_EL2 */
#define PSTATE_MODE_EL3t  12  /* EL3, 使用 SP_EL0 */
#define PSTATE_MODE_EL3h  13  /* EL3, 使用 SP_EL3 */
```

**编码规则**（`cpu.h:1597-1601`）：
```c
static inline unsigned int aarch64_pstate_mode(unsigned int el, bool handler)
{
    return (el << 2) | handler;  /* EL 在 bit[3:2]，SP 选择在 bit[0] */
}
```

---

## 3. arm_current_el() — 获取当前 EL

**定义**：`target/arm/internals.h:489-515`

```c
static inline int arm_current_el(CPUARMState *env)
{
    if (is_a64(env)) {
        return extract32(env->pstate, 2, 2);  /* PSTATE.M[3:2] = EL */
    }
    /* AArch32: 从 CPSR.M 模式位映射 */
    switch (env->uncached_cpsr & 0x1f) {
    case ARM_CPU_MODE_USR: return 0;
    case ARM_CPU_MODE_HYP: return 2;
    case ARM_CPU_MODE_MON: return 3;
    default:
        if (arm_is_secure(env) && !arm_el_is_aa64(env, 3)) {
            return 3;  /* AArch32 EL3: 所有安全特权模式运行在 EL3 */
        }
        return 1;
    }
}
```

**关键点**：
- AArch64：直接从 `pstate` bit[3:2] 提取，0=EL0, 1=EL1, 2=EL2, 3=EL3
- AArch32：从 CPSR 的 M[4:0] 模式位映射到 EL
- AArch32 特殊：若 EL3 为 AArch32，所有安全特权模式（SVC/FIQ/IRQ/ABT/UND/SYS）都映射到 EL3

---

## 4. EL 切换总览 — 异常进入与返回

```
┌─────────────────────────────────────────────────────┐
│                  EL 切换触发方式                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  低 EL → 高 EL（异常进入）：                           │
│    • 同步异常：SVC(→EL1), HVC(→EL2), SMC(→EL3)       │
│    • 同步陷阱：系统寄存器访问陷阱, WFI/WFE 陷阱        │
│    • 同步故障：Data Abort, Prefetch Abort, Alignment    │
│    • 异步异常：IRQ(→EL1/2), FIQ(→EL1/3), SError         │
│                                                     │
│  高 EL → 低 EL（异常返回）：                           │
│    • ERET 指令（从 SPSR_ELx + ELR_ELx 恢复）          │
│                                                     │
│  注意：EL 只能在这两种机制下切换，不能直接跳转           │
└─────────────────────────────────────────────────────┘
```

---

## 5. 异常进入：arm_cpu_do_interrupt

**定义**：`target/arm/helper.c:9469-9530`

```c
void arm_cpu_do_interrupt(CPUState *cs)
{
    unsigned int new_el = env->exception.target_el;

    /* PSCI 调用特殊处理 */
    if (tcg_enabled() && arm_is_psci_call(cpu, cs->exception_index)) {
        arm_handle_psci_call(cpu);
        return;
    }

    arm_call_pre_el_change_hook(cpu);  /* EL 变化前钩子 */

    /* 根据目标 EL 的执行状态选择处理路径 */
    if (arm_el_is_aa64(env, new_el)) {
        arm_cpu_do_interrupt_aarch64(cs);   /* AArch64 异常入口 */
    } else {
        arm_cpu_do_interrupt_aarch32(cs);   /* AArch32 异常入口 */
    }

    arm_call_el_change_hook(cpu);          /* EL 变化后钩子 */
    cpu_set_interrupt(cs, CPU_INTERRUPT_EXITTB);  /* 强制退出当前 TB */
}
```

**关键设计**：
- `env->exception.target_el` 在异常触发时就已确定目标 EL
- 目标 EL 的**执行状态**（AArch64/AArch32）决定处理路径，而非当前 EL 的状态
- `CPU_INTERRUPT_EXITTB` 确保 EL 变化后立即退出翻译块

---

## 6. AArch64 异常进入详细流程

**定义**：`target/arm/helper.c:9198-9425`

### 6.1 流程步骤

```
arm_cpu_do_interrupt_aarch64(cs)
  │
  ├── 1. 获取目标 EL 和向量基地址
  │     new_el = env->exception.target_el         :9202
  │     addr = env->cp15.vbar_el[new_el]          :9203
  │     new_mode = aarch64_pstate_mode(new_el, true)  :9204  (handler=true → SP_ELx)
  │
  ├── 2. SVE 状态切换（若 TCG）
  │     aarch64_sve_change_el(env, cur_el, new_el)  :9214
  │
  ├── 3. 计算向量偏移
  │     [详见第 7 节]                              :9217-9255
  │
  ├── 4. 保存旧状态
  │     aarch64_save_sp(env, cur_el)               :9345  (保存当前 SP)
  │     env->elr_el[new_el] = env->pc              :9346  (PC → ELR_ELx)
  │     env->banked_spsr[...] = old_mode           :9369  (PSTATE → SPSR_ELx)
  │
  ├── 5. 设置异常信息
  │     env->cp15.esr_el[new_el] = syndrome        :9320  (ESR_ELx)
  │     env->cp15.far_el[new_el] = vaddress        :9272  (FAR_ELx, 仅故障)
  │
  ├── 6. 设置新 PSTATE
  │     PAN 保持/设置                              :9375-9393
  │     TCO 设置（MTE）                            :9395-9397
  │     SSBS 设置                                  :9399-9404
  │     ALLINT 设置（NMI）                         :9407-9412
  │     pstate_write(env, PSTATE_DAIF | new_mode)  :9415  (DAIF 全掩码)
  │     env->aarch64 = true                        :9416
  │
  ├── 7. 恢复目标 EL 的 SP
  │     aarch64_restore_sp(env, new_el)            :9417
  │
  ├── 8. 重建 hflags
  │     arm_rebuild_hflags(env)                    :9420
  │
  └── 9. 跳转到异常向量
        env->pc = addr                             :9423
```

### 6.2 状态保存详解

异常进入时自动保存的状态：

| 保存内容 | 保存到 | 代码位置 |
|----------|--------|----------|
| 当前 PSTATE | SPSR_ELx | helper.c:9369 |
| 返回地址 (PC) | ELR_ELx | helper.c:9346 |
| 当前 SP | SP_ELx 寄存器组 | helper.c:9345 |
| 异常综合征 | ESR_ELx | helper.c:9320 |
| 故障地址 | FAR_ELx | helper.c:9272 |

---

## 7. 异常向量表与偏移计算

**定义**：`target/arm/helper.c:9217-9255, 9257-9338`

### 7.1 向量基地址

每个 EL 有独立的向量基地址寄存器：

```c
addr = env->cp15.vbar_el[new_el];  /* VBAR_EL1 / VBAR_EL2 / VBAR_EL3 */
```

### 7.2 偏移计算

```
异常向量偏移 = 来源偏移 + 类型偏移

来源偏移（基于 cur_el 与 new_el 的关系）：
┌────────────────────────────────────────┬────────┐
│ 条件                                    │ 偏移   │
├────────────────────────────────────────┼────────┤
│ 同 EL，使用 SP_ELx (PSTATE.SP=1)        │ +0x200 │
│ 同 EL，使用 SP_EL0 (PSTATE.SP=0)        │ +0x000 │
│ 低 EL → 高 EL，低 EL 为 AArch64         │ +0x400 │
│ 低 EL → 高 EL，低 EL 为 AArch32         │ +0x600 │
└────────────────────────────────────────┴────────┘

类型偏移（基于异常类型）：
┌────────────────┬────────┐
│ 异常类型         │ 偏移   │
├────────────────┼────────┤
│ Synchronous     │ +0x000 │
│ IRQ/NMI         │ +0x080 │
│ FIQ/VFIQ/VFNMI  │ +0x100 │
│ SError/VSError   │ +0x180 │
└────────────────┴────────┘
```

**代码实现**（helper.c:9217-9337）：

```c
/* 来源偏移 */
if (cur_el < new_el) {
    /* 低 EL → 高 EL */
    bool is_aa64;
    switch (new_el) {
    case 3: is_aa64 = arm_scr_rw_eff(env); break;  /* SCR_EL3.RW */
    case 2: is_aa64 = (hcr & HCR_RW) != 0; break;  /* HCR_EL2.RW */
    case 1: is_aa64 = is_a64(env); break;
    }
    addr += is_aa64 ? 0x400 : 0x600;
} else {
    if (pstate_read(env) & PSTATE_SP) addr += 0x200;
}

/* 类型偏移 */
switch (cs->exception_index) {
case EXCP_IRQ: case EXCP_VIRQ: case EXCP_NMI:
    addr += 0x80; break;
case EXCP_FIQ: case EXCP_VFIQ: case EXCP_VFNMI:
    addr += 0x100; break;
case EXCP_VSERR:
    addr += 0x180; break;
/* Synchronous: +0x000 (default) */
}
```

---

## 8. PSTATE 设置与 DAIF 掩码

异常进入时新 EL 的 PSTATE 设置（`helper.c:9375-9416`）：

```c
/* 基础设置：DAIF 全掩码 + handler mode (SP_ELx) */
new_mode = PSTATE_DAIF | aarch64_pstate_mode(new_el, true);

/* FEAT_PAN: 条件设置 PAN */
if (SCTLR_SPAN == 0 && (new_el == 1 || new_el == 2_E2H)) {
    new_mode |= PSTATE_PAN;
}

/* FEAT_MTE: 设置 TCO (Tag Check Override) */
new_mode |= PSTATE_TCO;

/* FEAT_SSBS: 根据 SCTLR_ELx.DSSBS 设置 */
if (SCTLR_DSSBS_64) new_mode |= PSTATE_SSBS;

/* FEAT_NMI: 根据 SCTLR_ELx.SPINTMASK 设置 ALLINT */
if (!SCTLR_SPINTMASK) new_mode |= PSTATE_ALLINT;

pstate_write(env, new_mode);
env->aarch64 = true;
```

**重要行为**：进入异常时 DAIF 全部置 1，即 **Debug/SError/IRQ/FIQ 全部屏蔽**，防止异常处理过程中被打断。

---

## 9. 异常返回：ERET 实现

**定义**：`target/arm/tcg/helper-a64.c:622-758`

### 9.1 ERET 流程

```c
void HELPER(exception_return)(CPUARMState *env, uint64_t new_pc)
{
    int cur_el = arm_current_el(env);
    uint64_t spsr = env->banked_spsr[aarch64_banked_spsr_index(cur_el)];

    aarch64_save_sp(env, cur_el);       /* 保存当前 SP */
    arm_clear_exclusive(env);            /* 清除独占监控 */

    /* 从 SPSR 提取目标 EL */
    new_el = el_from_spsr(spsr);        /* helper-a64.c:584-620 */
    if (new_el == -1) goto illegal_return;
    if (new_el > cur_el) goto illegal_return;

    /* 安全状态检查（FEAT_RME） */
    /* ... 验证 SCR_EL3.{NS,NSE} 组合合法性 ... */

    if (!return_to_aa64) {
        /* 返回 AArch32 */
        env->aarch64 = false;
        cpsr_write_from_spsr_elx(env, spsr);
        aarch64_sync_64_to_32(env);
        env->regs[15] = new_pc;
    } else {
        /* 返回 AArch64 */
        env->aarch64 = true;
        pstate_write(env, spsr);
        aarch64_restore_sp(env, new_el);
        /* TBI 处理 */
        env->pc = new_pc;               /* helper-a64.c:747 */
    }

    aarch64_sve_change_el(env, cur_el, new_el, return_to_aa64);
    arm_rebuild_hflags(env);            /* 重建 hflags */
}
```

### 9.2 el_from_spsr — 目标 EL 提取

**定义**：`target/arm/tcg/helper-a64.c:584-620`

```c
static int el_from_spsr(uint32_t spsr)
{
    if (spsr & PSTATE_nRW) {
        /* AArch32 模式 */
        switch (spsr & CPSR_M) {
        case ARM_CPU_MODE_USR: return 0;
        case ARM_CPU_MODE_HYP: return 2;
        case ARM_CPU_MODE_SVC/FIQ/IRQ/ABT/UND/SYS: return 1;
        case ARM_CPU_MODE_MON: return -1;  /* 非法：不能从 AArch64 返回到 Mon */
        }
    } else {
        /* AArch64 模式 */
        return extract32(spsr, 2, 2);  /* M[3:2] = EL */
    }
}
```

---

## 10. SMC — EL1→EL3 切换

### 10.1 SMC 指令解码

**定义**：`target/arm/tcg/translate-a64.c:3193-3205`

```c
static bool trans_SMC(DisasContext *s, arg_i *a)
{
    if (s->current_el == 0) {
        unallocated_encoding(s);  /* EL0 不能执行 SMC */
        return true;
    }
    gen_a64_update_pc(s, 0);
    gen_helper_pre_smc(tcg_env, tcg_constant_i32(syn_aa64_smc(a->imm)));
    gen_ss_advance(s);
    gen_exception_insn_el(s, 4, EXCP_SMC, syn_aa64_smc(a->imm), 3);
    return true;
}
```

### 10.2 helper_pre_smc — SMC 路由决策

**定义**：`target/arm/tcg/op_helper.c:1111-1200`

SMC 路由决策表：

```
┌───────────────────┬──────────────────────┬────────────────────┐
│ 条件               │ HCR_TSC && NS EL1    │ !HCR_TSC || !NS    │
├───────────────────┼──────────────────────┼────────────────────┤
│ EL3 存在 && !SMD   │                      │                    │
│  PSCI 有效调用      │ 陷入 EL2             │ PSCI 调用           │
│  非 PSCI           │ 陷入 EL2             │ 陷入 EL3            │
├───────────────────┼──────────────────────┼────────────────────┤
│ EL3 存在 && SMD    │                      │                    │
│  PSCI 有效调用      │ 陷入 EL2             │ PSCI 调用           │
│  非 PSCI           │ 陷入 EL2             │ UNDEF              │
├───────────────────┼──────────────────────┼────────────────────┤
│ EL3 不存在         │                      │                    │
│  PSCI 有效调用      │ 陷入 EL2             │ PSCI 调用           │
│  非 PSCI           │ UNDEF/陷入 EL2(NV)   │ UNDEF              │
└───────────────────┴──────────────────────┴────────────────────┘
```

**关键逻辑**：
```c
/* SCR_EL3.SMD 控制 SMC 是否被禁用 */
bool smd = arm_feature(env, ARM_FEATURE_AARCH64) ? smd_flag
                                                  : smd_flag && !secure;

/* HCR_EL2.TSC 在 NS EL1 优先路由到 EL2 */
if (cur_el == 1 && (arm_hcr_el2_eff(env) & HCR_TSC)) {
    raise_exception(env, EXCP_HYP_TRAP, syndrome, 2);  /* → EL2 */
}
```

---

## 11. HVC — EL1→EL2 切换

**定义**：`target/arm/tcg/translate-a64.c:3173-3191`

```c
static bool trans_HVC(DisasContext *s, arg_i *a)
{
    int target_el = s->current_el == 3 ? 3 : 2;
    if (s->current_el == 0) {
        unallocated_encoding(s);  /* EL0 不能执行 HVC */
        return true;
    }
    gen_helper_pre_hvc(tcg_env);
    gen_exception_insn_el(s, 4, EXCP_HVC, syn_aa64_hvc(a->imm), target_el);
    return true;
}
```

**HVC 路由**（`op_helper.c:1086-1107`）：
- **EL3 执行 HVC**：target_el = 3（自身处理）
- **EL1 执行 HVC**：target_el = 2
- **HCR_EL2.HCD = 1**：HVC 被禁用，UNDEF
- **SCR_EL3.HCE = 0**（AArch32 EL3）：HVC 被禁用

---

## 12. 安全状态管理

### 12.1 ARMSecuritySpace 枚举

**定义**：`include/hw/arm/arm-security.h:18-35`

```
安全空间：
┌──────────────┬──────────────┬──────────────┐
│ SCR_EL3.NS   │ SCR_EL3.NSE  │ 安全空间      │
├──────────────┼──────────────┼──────────────┤
│ 0            │ 0            │ Secure        │
│ 1            │ 0            │ Non-secure    │
│ 0            │ 1            │ (Reserved)    │
│ 1            │ 1            │ Realm (RME)   │
└──────────────┴──────────────┴──────────────┘
```

### 12.2 arm_security_space()

**定义**：`target/arm/helper.c:10131-10161`

```c
ARMSecuritySpace arm_security_space(CPUARMState *env)
{
    /* EL3 本身的安全空间 */
    if (is_a64(env) && extract32(env->pstate, 2, 2) == 3) {
        if (cpu_isar_feature(aa64_rme, ...)) return ARMSS_Root;
        else return ARMSS_Secure;
    }
    /* AArch32 Monitor mode */
    if ((env->uncached_cpsr & CPSR_M) == ARM_CPU_MODE_MON)
        return ARMSS_Secure;

    return arm_security_space_below_el3(env);
}
```

### 12.3 arm_security_space_below_el3()

**定义**：`target/arm/helper.c:10163-10187`

```c
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env)
{
    if (!(env->cp15.scr_el3 & SCR_NS)) return ARMSS_Secure;
    if (env->cp15.scr_el3 & SCR_NSE) return ARMSS_Realm;
    return ARMSS_NonSecure;
}
```

---

## 13. SCR_EL3 — 安全配置寄存器

**存储**：`target/arm/cpu.h:392`

```c
uint64_t scr_el3; /* Secure configuration register */
```

**关键控制位**：

| 位 | 名称 | 作用 |
|----|------|------|
| NS | Non-secure | 控制低于 EL3 的安全状态 |
| NSE | NS Extension | RME Realm 扩展 |
| SMD | SMC Disable | 禁用 SMC 指令 |
| RW | Register Width | 控制 EL2/EL1 是否为 AArch64 |
| HCE | HVC Enable | 控制 HVC 是否可用（AArch32） |
| IRQ/FIQ/EA | 路由控制 | 异步异常路由到 EL3 |
| SIF | Secure IF | 安全世界不执行非安全内存代码 |

---

## 14. HCR_EL2 — 虚拟化配置寄存器

**存储**：`target/arm/cpu.h:390`

```c
uint64_t hcr_el2; /* Hypervisor configuration register */
```

**关键陷阱位**：

| 位 | 名称 | 作用 |
|----|------|------|
| TSC | Trap SMC | EL1 的 SMC 陷入 EL2 |
| HCD | HVC Disable | 禁用 HVC |
| TWI | Trap WFI | EL0/EL1 的 WFI 陷入 EL2 |
| TWE | Trap WFE | EL0/EL1 的 WFE 陷入 EL2 |
| TVM | Trap Virtual Memory | 虚拟内存寄存器访问陷入 EL2 |
| RW | Register Width | EL1 为 AArch64 (=1) 或 AArch32 (=0) |
| E2H | EL2 Host | VHE 模式 |
| TGE | Trap General Exceptions | 异常路由到 EL2 |
| NV/NV2 | Nested Virtualization | 嵌套虚拟化 |

---

## 15. MMU 索引与 EL 映射

### 15.1 ARMMMUIdx 枚举

**定义**：`target/arm/mmuidx.h:142-185`

```c
/* EL0/EL1 翻译体制 */
ARMMMUIdx_E10_0       /* EL0, Stage1 via TTBR0/1_EL1 */
ARMMMUIdx_E10_1       /* EL1, Stage1 via TTBR0/1_EL1 */
ARMMMUIdx_E10_1_PAN   /* EL1 with PAN active */

/* EL2 (VHE) 翻译体制 */
ARMMMUIdx_E20_0       /* EL0, Stage1 via TTBR0/1_EL2 (VHE) */
ARMMMUIdx_E20_2       /* EL2, Stage1 via TTBR0/1_EL2 (VHE) */
ARMMMUIdx_E20_2_PAN   /* EL2 with PAN */

/* EL2 (非 VHE) */
ARMMMUIdx_E2          /* EL2, Stage1 via TTBR0_EL2 */

/* EL3 翻译体制 */
ARMMMUIdx_E3          /* EL3, Stage1 via TTBR0_EL3 */
ARMMMUIdx_E30_0       /* EL0 under EL3 (AArch32 Secure) */

/* Stage 2 */
ARMMMUIdx_Stage2_S    /* Secure Stage 2 */
ARMMMUIdx_Stage2      /* Non-secure Stage 2 */

/* 物理地址空间直接映射 */
ARMMMUIdx_Phys_S      /* Secure physical */
ARMMMUIdx_Phys_NS     /* Non-secure physical */
ARMMMUIdx_Phys_Root   /* Root physical (RME) */
ARMMMUIdx_Phys_Realm  /* Realm physical (RME) */
```

### 15.2 arm_mmu_idx_el() — EL 到 MMU 索引映射

**定义**：`target/arm/helper.c:9957-10020`

```c
ARMMMUIdx arm_mmu_idx_el(CPUARMState *env, int el)
{
    switch (el) {
    case 0:
        if (E2H && TGE) return ARMMMUIdx_E20_0;    /* VHE */
        if (Secure && !AArch64_EL3) return ARMMMUIdx_E30_0;
        return ARMMMUIdx_E10_0;
    case 1:
        if (PAN_active) return ARMMMUIdx_E10_1_PAN;
        return ARMMMUIdx_E10_1;
    case 2:
        if (E2H) return ARMMMUIdx_E20_2 [+PAN];
        return ARMMMUIdx_E2;
    case 3:
        return ARMMMUIdx_E3;
    }
}
```

**各 EL 的地址翻译体制**：

| EL | 翻译体制 | 页表基址寄存器 | Stage 2 |
|----|----------|---------------|---------|
| EL0 | EL1&0 | TTBR0_EL1 / TTBR1_EL1 | 可选（EL2 控制） |
| EL1 | EL1&0 | TTBR0_EL1 / TTBR1_EL1 | 可选 |
| EL0 (VHE) | EL2&0 | TTBR0_EL2 / TTBR1_EL2 | 无 |
| EL2 | EL2 | TTBR0_EL2 | 无 |
| EL2 (VHE) | EL2&0 | TTBR0_EL2 / TTBR1_EL2 | 无 |
| EL3 | EL3 | TTBR0_EL3 | 无 |

---

## 16. 系统寄存器访问控制

### 16.1 ARMCPRegInfo 访问级别

**定义**：`target/arm/cpregs.h:290-333`

```c
/* PL 编码（与 EL 对应） */
#define PL3_R  0x80    /* EL3 可读 */
#define PL3_W  0x40    /* EL3 可写 */
#define PL2_R  0x20    /* EL2 可读 */
#define PL2_W  0x10    /* EL2 可写 */
#define PL1_R  0x08    /* EL1 可读 */
#define PL1_W  0x04    /* EL1 可写 */
#define PL0_R  0x02    /* EL0 可读 */
#define PL0_W  0x01    /* EL0 可写 */

/* 常用组合 */
#define PL1_RW  (PL1_R | PL1_W)  /* EL1+ 可读写 */
#define PL2_RW  (PL2_R | PL2_W)  /* EL2+ 可读写 */
#define PL3_RW  (PL3_R | PL3_W)  /* EL3 可读写 */
```

### 16.2 cp_access_ok() — 访问检查

**定义**：`target/arm/cpregs.h:1122-1126`

```c
static inline bool cp_access_ok(int current_el, const ARMCPRegInfo *ri,
                                 int isread)
{
    return (ri->access >> ((current_el * 2) + isread)) & 1;
}
```

**编码方式**：`access` 字段的 bit 位直接映射 `(EL * 2 + isread)`：
- bit 0 = EL0 写
- bit 1 = EL0 读
- bit 2 = EL1 写
- bit 3 = EL1 读
- ...以此类推

### 16.3 访问结果

**定义**：`target/arm/cpregs.h:335-375`

```c
typedef enum {
    CP_ACCESS_OK = 0,              /* 允许访问 */
    CP_ACCESS_TRAP = 1,            /* 陷入当前/更高 EL */
    CP_ACCESS_TRAP_EL2 = 2,       /* 陷入 EL2 */
    CP_ACCESS_TRAP_EL3 = 3,       /* 陷入 EL3 */
    CP_ACCESS_UNDEFINED = 4,       /* UNDEF 异常 */
    /* ... 其他变体 ... */
} CPAccessResult;
```

### 16.4 寄存器示例

```c
/* VBAR_EL1: EL1+ 可读写 */
{ .name = "VBAR_EL1", .access = PL1_RW, ... }   /* helper.c:5721+ */

/* HCR_EL2: EL2+ 可读写 */
{ .name = "HCR_EL2", .access = PL2_RW, ... }    /* helper.c:4067+ */

/* SCR_EL3: EL3 可读写 */
{ .name = "SCR_EL3", .access = PL3_RW, ... }    /* helper.c:4938+ */
```

---

## 17. 特权指令行为差异

### 17.1 WFI / WFE

**翻译**：`target/arm/tcg/translate-a64.c:2030-2041`

```
EL0 执行 WFI:
  └── 若 SCTLR_EL1.nTWI = 0 → 陷入 EL1 (UNDEF)
  └── 若 HCR_EL2.TWI = 1 → 陷入 EL2
  └── 若 SCR_EL3.TWI = 1 → 陷入 EL3

EL1 执行 WFI:
  └── 若 HCR_EL2.TWI = 1 → 陷入 EL2
  └── 否则 → 执行等待中断

EL2/EL3 执行 WFI:
  └── 直接执行
```

### 17.2 Cache 操作

**访问控制**：`target/arm/helper.c:5443-5575`

| 指令 | 最低 EL | 说明 |
|------|---------|------|
| DC ZVA | EL0 | 零填充缓存行（PL0_W） |
| DC IVAC | EL1 | 使无效数据缓存（PL1_W） |
| DC CIVAC | EL0 | 清理并使无效（PL0_W） |
| DC ISW/CSW/CISW | EL1 | 按 Set/Way 操作（PL1_W） |
| IC IALLUIS/IALLU | EL1 | 指令缓存全部无效（PL1_W） |
| IC IVAU | EL0 | 指令缓存按 VA 无效（PL0_W） |

### 17.3 AT (Address Translation) 指令

**定义**：`target/arm/tcg/cpregs-at.c:49-140`

| 指令 | EL | 翻译体制 |
|------|-----|---------|
| AT S1E1R/W | EL1+ | Stage 1 EL1&0 |
| AT S1E0R/W | EL1+ | Stage 1 EL1&0 (as EL0) |
| AT S12E1R/W | EL2+ | Stage 1+2 EL1&0 |
| AT S1E2R/W | EL2+ | Stage 1 EL2 |
| AT S1E3R/W | EL3 | Stage 1 EL3 |

### 17.4 TLBI 指令

**控制**：`target/arm/tcg/tlb-insns.c:17-90`

- EL0 不能执行任何 TLBI
- EL1 TLBI 可被 HCR_EL2 陷入 EL2
- TLBI 的范围取决于 EL：EL1 只能 TLBI 当前 VMID 的条目，EL2 可以 TLBI 所有 VMID

---

## 18. TB Flags 中的 EL 编码

### 18.1 TB Flag 位布局

**定义**：`target/arm/cpu.h:2438-2449`

```c
FIELD(TBFLAG_ANY, AARCH64_STATE, 0, 1)  /* AArch64 = 1 */
FIELD(TBFLAG_ANY, SS_ACTIVE, 1, 1)      /* 单步调试激活 */
FIELD(TBFLAG_ANY, PSTATE__SS, 2, 1)     /* PSTATE.SS */
FIELD(TBFLAG_ANY, BE_DATA, 3, 1)        /* 大端数据 */
FIELD(TBFLAG_ANY, MMUIDX, 4, 4)         /* MMU 索引（隐含 EL 信息） */
FIELD(TBFLAG_ANY, FPEXC_EL, 8, 2)       /* FP 异常目标 EL */
FIELD(TBFLAG_ANY, ALIGN_MEM, 10, 1)     /* 对齐检查 */
FIELD(TBFLAG_ANY, PSTATE__IL, 11, 1)    /* 非法执行状态 */
FIELD(TBFLAG_ANY, FGT_ACTIVE, 12, 1)    /* 细粒度陷阱激活 */
```

**EL 信息编码方式**：EL 不直接作为 TB flag 位，而是**间接通过 MMUIDX 编码**。不同 EL 产生不同的 MMU 索引值，因此具有不同 EL 的 TB 自动不匹配。

---

## 19. arm_rebuild_hflags — EL 变化后重建标志

**定义**：`target/arm/tcg/hflags.c:506-524`

```c
static CPUARMTBFlags rebuild_hflags_internal(CPUARMState *env)
{
    int el = arm_current_el(env);
    int fp_el = fp_exception_el(env, el);
    ARMMMUIdx mmu_idx = arm_mmu_idx_el(env, el);

    if (is_a64(env)) {
        return rebuild_hflags_a64(env, el, fp_el, mmu_idx);
    } else if (arm_feature(env, ARM_FEATURE_M)) {
        return rebuild_hflags_m32(env, fp_el, mmu_idx);
    } else {
        return rebuild_hflags_a32(env, fp_el, mmu_idx);
    }
}

void arm_rebuild_hflags(CPUARMState *env)
{
    env->hflags = rebuild_hflags_internal(env);
}
```

**调用时机**：
- 异常进入后（`helper.c:9420`）
- 异常返回后（`helper-a64.c:714, 728`）
- 任何可能改变 EL 相关状态的操作后

**hflags 中 EL 敏感的信息**：
- MMUIDX（隐含 EL）
- AARCH64_STATE
- PAN/UAO 状态
- SCTLR 派生的对齐/端序
- FP 异常目标 EL
- FGT（Fine-Grained Trap）激活状态

---

## 20. DisasContext 中的 EL 传播

**定义**：`target/arm/tcg/translate.h:106`

```c
int current_el;  /* DisasContext 中的当前 EL */
```

**初始化**：`target/arm/tcg/translate-a64.c:10676`

```c
dc->current_el = arm_mmu_idx_to_el(dc->mmu_idx);
```

**影响翻译的方式**：
1. **指令合法性**：SMC/HVC 在 EL0 是 UNDEF
2. **系统寄存器访问**：通过 `cp_access_ok(dc->current_el, ri, isread)` 检查
3. **陷阱行为**：`access_check_cp_reg()` 根据 EL 决定是否陷入更高 EL
4. **地址翻译**：AT 指令的翻译体制取决于当前 EL

---

## 21. PSTATE 特殊位对执行的影响

### 21.1 PAN (Privileged Access Never)

- **适用 EL**：EL1 和 EL2 (VHE)
- **效果**：PAN=1 时，EL1 的 `LDTR`/`STTR` 等用户态访问指令正常，但普通 load/store 访问用户态内存会导致 Permission Fault
- **进入 EL1 异常时**：若 `SCTLR_EL1.SPAN=0`，自动设置 PAN=1

### 21.2 UAO (User Access Override)

- **适用 EL**：EL1 和 EL2
- **效果**：UAO=1 时，`LDTR`/`STTR` 的权限检查等同于普通 load/store（而非用户态权限）
- **hflags 处理**：`hflags.c:345-360`

### 21.3 SP 选择 (PSTATE.SP)

- **SP=0**：使用 SP_EL0
- **SP=1**：使用 SP_ELx（当前 EL 的专用 SP）
- **异常进入**：自动设为 handler mode (SP=1)
- **EL0**：始终使用 SP_EL0

### 21.4 SCTLR_ELx 差异

每个 EL 有独立的 SCTLR，影响：
- 端序（EE/E0E）
- 对齐检查（A）
- 写异或执行（WXN）
- MTE（TCF/TCF0/ATA/ATA0）
- 栈指针对齐检查（SA/SA0）
- SPAN/EPAN
- DSSBS（SSBS 默认值）

---

## 22. EL 切换对 SVE/SME 的影响

**处理函数**：异常进入/返回时调用 `aarch64_sve_change_el()`

```c
/* 异常进入 */
aarch64_sve_change_el(env, cur_el, new_el, is_a64(env));  /* helper.c:9214 */

/* 异常返回 */
aarch64_sve_change_el(env, cur_el, new_el, return_to_aa64); /* helper-a64.c:758 */
```

**影响**：
- SVE 向量长度可能随 EL 变化（每个 EL 有独立的 ZCR_ELx.LEN）
- EL 变化可能需要截断/扩展 Z 寄存器和 P 寄存器
- SME 流模式在 EL 变化时可能被禁用

---

## 23. arm_el_is_aa64 — EL 执行状态判断

**定义**：`target/arm/internals.h:452-483`

```c
static inline bool arm_el_is_aa64(CPUARMState *env, int el)
{
    assert(el >= 1 && el <= 3);
    bool aa64 = arm_feature(env, ARM_FEATURE_AARCH64);

    /* 最高 EL 总是最大支持宽度 */
    if (el == 3) return aa64;

    /* EL2/EL1 宽度由更高 EL 的配置决定 */
    if (arm_feature(env, ARM_FEATURE_EL3)) {
        aa64 = aa64 && arm_scr_rw_eff(env);  /* SCR_EL3.RW */
    }
    if (el == 2) return aa64;

    if (arm_is_el2_enabled(env)) {
        aa64 = aa64 && (arm_hcr_el2_eff(env) & HCR_RW);  /* HCR_EL2.RW */
    }
    return aa64;
}
```

**控制链**：

```
EL3 执行状态 = CPU 支持的最大宽度（固定 AArch64）
    │
    ├── SCR_EL3.RW ──→ EL2 执行状态
    │                     │
    │                     └── HCR_EL2.RW ──→ EL1 执行状态
    │                                           │
    │                                           └── EL0 = 跟随 EL1
    │
    └── SCR_EL3.RW=0 → EL2/EL1/EL0 全部 AArch32
```

---

## 24. EL 切换完整时序图

### 24.1 SMC: EL1 → EL3

```
Guest (EL1)                    QEMU TCG                         EL3 Handler
    │                              │                                │
    │  SMC #imm                    │                                │
    │─────────────────────────────→│                                │
    │                              │ trans_SMC()                    │
    │                              │  ├── EL0 检查                  │
    │                              │  ├── gen_helper_pre_smc()      │
    │                              │  │   ├── SMD 检查              │
    │                              │  │   ├── HCR.TSC 检查          │
    │                              │  │   └── PSCI 检查             │
    │                              │  └── gen_exception(EXCP_SMC,3) │
    │                              │                                │
    │                              │ arm_cpu_do_interrupt()         │
    │                              │  ├── pre_el_change_hook()      │
    │                              │  ├── arm_cpu_do_interrupt_aarch64()
    │                              │  │   ├── addr = VBAR_EL3 + offset
    │                              │  │   ├── save SP/ELR/SPSR      │
    │                              │  │   ├── ESR_EL3 = syndrome    │
    │                              │  │   ├── PSTATE = DAIF|EL3h    │
    │                              │  │   ├── aarch64_restore_sp(3) │
    │                              │  │   ├── arm_rebuild_hflags()  │
    │                              │  │   └── PC = addr             │
    │                              │  ├── el_change_hook()          │
    │                              │  └── EXITTB                    │
    │                              │                                │
    │                              │────────────────────────────────→│
    │                              │                     VBAR_EL3 + 0x400
```

### 24.2 ERET: EL3 → EL1

```
EL3 Handler                    QEMU TCG                         Guest (EL1)
    │                              │                                │
    │  ERET                        │                                │
    │─────────────────────────────→│                                │
    │                              │ HELPER(exception_return)()     │
    │                              │  ├── cur_el = 3               │
    │                              │  ├── spsr = SPSR_EL3           │
    │                              │  ├── save SP                   │
    │                              │  ├── new_el = el_from_spsr()   │
    │                              │  │   └── = 1 (from SPSR M bits)│
    │                              │  ├── 安全状态验证              │
    │                              │  ├── pstate_write(spsr)        │
    │                              │  ├── aarch64_restore_sp(1)     │
    │                              │  ├── arm_rebuild_hflags()      │
    │                              │  ├── SVE/SME EL change         │
    │                              │  └── PC = ELR_EL3              │
    │                              │                                │
    │                              │────────────────────────────────→│
    │                              │                     ELR_EL3 地址
```

---

## 25. 各 EL 指令执行差异对比表

| 能力 | EL0 | EL1 | EL2 | EL3 |
|------|-----|-----|-----|-----|
| **异常调用** | SVC | SVC, HVC | SVC, HVC, SMC* | SVC, HVC, SMC |
| **系统寄存器** | 极少 (CTR, TPIDR_EL0 等) | 大量 (SCTLR, TCR, TTBR, VBAR 等) | 全部 EL2 + 陷阱 EL1 | 全部 |
| **Cache 操作** | DC ZVA, DC CVAC, IC IVAU | 全部 DC/IC | 全部 + Inner Shareable | 全部 |
| **TLB 操作** | 无 | 当前 VMID | 所有 VMID | 全部 + Secure |
| **AT 指令** | 无 | S1E0/S1E1 | S1E0/S1E1/S1E2/S12E* | 全部 + S1E3 |
| **地址翻译** | TTBR0/1_EL1 | TTBR0/1_EL1 | TTBR0_EL2 (或 TTBR0/1_EL2 VHE) | TTBR0_EL3 |
| **Stage 2** | 受控 | 受控 | 不受 | 不受 |
| **安全状态** | 取决于 SCR_EL3 | 取决于 SCR_EL3 | 取决于 SCR_EL3 | Root/Secure |
| **中断掩码** | 无法直接修改 DAIF | 可修改 (受 SCTLR 控制) | 可修改 | 全部可修改 |
| **PAN** | N/A | 可用 | 可用 (VHE) | N/A |
| **UAO** | N/A | 可用 | 可用 | N/A |
| **WFI/WFE** | 可被陷入 | 可被陷入 EL2 | 直接执行 | 直接执行 |
| **断点/监视点** | 受 MDSCR 控制 | 配置 DBGBCR/DBGWCR | 可控 EL1 调试 | 可控所有调试 |

---

## 26. 总结

### EL 状态管理核心机制

1. **EL 存储**：`pstate` bit[3:2] 直接编码（AArch64），CPSR.M 模式位映射（AArch32）
2. **EL 切换**：严格通过异常进入/ERET 返回，不能直接跳转
3. **状态保存**：SPSR_ELx（PSTATE）+ ELR_ELx（返回地址）+ SP 切换
4. **安全状态**：SCR_EL3.{NS,NSE} 控制 Secure/NonSecure/Realm/Root
5. **执行状态**：SCR_EL3.RW → EL2, HCR_EL2.RW → EL1, 级联控制
6. **TB 管理**：EL 通过 MMUIDX 间接编码到 TB flags，EL 变化强制 TB 重翻译

### 指令执行变化核心规律

1. **权限递增**：EL0 → EL3，可访问的系统寄存器和特权指令递增
2. **陷阱级联**：低 EL 操作可被高 EL 通过配置位陷入（HCR_EL2、SCR_EL3）
3. **地址翻译隔离**：每个 EL 翻译体制独立，EL2 可施加 Stage 2
4. **PSTATE 掩码**：异常进入自动 DAIF 全掩码，ERET 恢复旧 PSTATE
5. **安全域隔离**：EL3 控制安全/非安全世界切换，TLB 按安全状态标记

### 关键源文件索引

| 文件 | 核心功能 |
|------|----------|
| `target/arm/cpu.h:270-284, 1546-1581` | PSTATE 定义、Mode 值 |
| `target/arm/internals.h:489-515` | arm_current_el() |
| `target/arm/internals.h:452-483` | arm_el_is_aa64() |
| `target/arm/helper.c:9198-9425` | AArch64 异常进入 |
| `target/arm/helper.c:9469-9530` | arm_cpu_do_interrupt |
| `target/arm/helper.c:9957-10020` | arm_mmu_idx_el() |
| `target/arm/helper.c:10131-10187` | arm_security_space() |
| `target/arm/tcg/helper-a64.c:584-758` | ERET 实现 |
| `target/arm/tcg/op_helper.c:1111-1200` | SMC 路由决策 |
| `target/arm/tcg/translate-a64.c:3173-3205` | HVC/SMC 解码 |
| `target/arm/tcg/hflags.c:506-524` | arm_rebuild_hflags |
| `target/arm/mmuidx.h:142-185` | ARMMMUIdx 枚举 |
| `target/arm/cpregs.h:290-375` | 系统寄存器访问控制 |
