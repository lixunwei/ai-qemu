# Debug/Breakpoint/Watchpoint 调试子系统深度分析 — 硬件断点、Watchpoint、单步与 GDB Stub

> 基于 QEMU 11.0.50 源码分析，聚焦 ARM 调试子系统全栈实现：
> DBGBVR/DBGBCR 硬件断点（地址匹配、上下文匹配、链接断点）、
> DBGWVR/DBGWCR Watchpoint（BAS/MASK 地址范围、读/写/访问类型、EL/安全过滤）、
> MDSCR_EL1 调试控制（MDE 监视使能、KDE 内核调试、SS 软件单步）、
> 调试异常路由（arm_debug_target_el、MDCR_EL2.TDE、MDCR_EL3.SDD）、
> TCG 单步实现（gen_ss_advance、Active-not-pending → Active-pending 状态机）、
> BRK 指令、GDB Stub 集成、KVM 调试透传。

---

## 目录

1. [调试子系统全景](#1-调试子系统全景)
2. [CPUARMState 中的调试状态](#2-cpuarmstate-中的调试状态)
3. [硬件断点实现](#3-硬件断点实现)
4. [Watchpoint 实现](#4-watchpoint-实现)
5. [bp_wp_matches 统一匹配逻辑](#5-bp_wp_matches-统一匹配逻辑)
6. [MDSCR_EL1 与调试使能](#6-mdscr_el1-与调试使能)
7. [调试异常路由](#7-调试异常路由)
8. [软件单步（Software Step）](#8-软件单步software-step)
9. [BRK 指令与软件断点](#9-brk-指令与软件断点)
10. [调试异常分发](#10-调试异常分发)
11. [GDB Stub 集成](#11-gdb-stub-集成)
12. [KVM 调试集成](#12-kvm-调试集成)
13. [总结](#13-总结)

---

## 1. 调试子系统全景

```
┌─────────────────────────────────────────────────────────────┐
│  Guest 视角                                                 │
│  DBGBVR<n>_EL1 / DBGBCR<n>_EL1  — 硬件断点                │
│  DBGWVR<n>_EL1 / DBGWCR<n>_EL1  — 硬件 Watchpoint         │
│  MDSCR_EL1 (MDE/KDE/SS)          — 调试控制                │
│  MDCR_EL2 (TDE/TDOSA/TDA)        — 虚拟化路由              │
│  BRK #imm / BKPT #imm            — 软件断点指令            │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  TCG 层                                                     │
│                                                             │
│  hw_breakpoint_update() ─→ cpu_breakpoint_insert()          │
│  hw_watchpoint_update() ─→ cpu_watchpoint_insert()          │
│                                                             │
│  断点命中:                                                   │
│    arm_debug_check_breakpoint() → bp_wp_matches()           │
│    → arm_debug_excp_handler() → raise_exception_debug()     │
│                                                             │
│  Watchpoint 命中:                                            │
│    arm_debug_check_watchpoint() → check_watchpoints()       │
│    → arm_debug_excp_handler() → raise_exception_debug()     │
│                                                             │
│  软件单步:                                                   │
│    gen_ss_advance() → PSTATE.SS = 0                         │
│    gen_step_complete_exception() → exception_swstep()       │
│                                                             │
│  BRK #imm:                                                   │
│    trans_BRK() → gen_exception_bkpt_insn()                  │
│    → HELPER(exception_bkpt_insn) → raise_exception()       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  异常路由                                                    │
│  arm_debug_target_el():                                     │
│    HCR_EL2.TGE || MDCR_EL2.TDE → EL2                      │
│    Secure + AArch32 EL3 → EL3                               │
│    其他 → EL1                                               │
│                                                             │
│  异常综合征 (ESR_ELx.EC):                                    │
│    EC_BREAKPOINT (0x30/0x31)                                │
│    EC_WATCHPOINT (0x34/0x35)                                │
│    EC_SOFTWARESTEP (0x32/0x33)                              │
│    EC_AA64_BKPT (0x3c)                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. CPUARMState 中的调试状态

```c
// target/arm/cpu.h:529-538 (cp15 结构内)
struct {
    uint64_t dbgbvr[16];   // :529 — 断点值寄存器（最多 16 个）
    uint64_t dbgbcr[16];   // :530 — 断点控制寄存器
    uint64_t dbgwvr[16];   // :531 — Watchpoint 值寄存器
    uint64_t dbgwcr[16];   // :532 — Watchpoint 控制寄存器
    uint64_t dbgclaim;     // :533 — DBGCLAIM 位
    uint64_t mdscr_el1;    // :534 — 监视调试系统控制寄存器
    uint64_t oslsr_el1;    // :535 — OS Lock 状态
    uint64_t osdlr_el1;    // :536 — OS DoubleLock 状态
    uint64_t mdcr_el2;     // :537 — EL2 调试控制
    uint64_t mdcr_el3;     // :538 — EL3 调试控制
} cp15;
```

### 断点/Watchpoint 数量

```c
// target/arm/internals.h:1107-1141
static inline int arm_num_brps(ARMCPU *cpu) {
    // AArch64: ID_AA64DFR0_EL1.BRPs + 1
    // AArch32: DBGDIDR.BRPs + 1
    return FIELD_EX64_IDREG(&cpu->isar, ID_AA64DFR0, BRPS) + 1;  // :1110
}

static inline int arm_num_wrps(ARMCPU *cpu) {
    return FIELD_EX64_IDREG(&cpu->isar, ID_AA64DFR0, WRPS) + 1;  // :1124
}

static inline int arm_num_ctx_cmps(ARMCPU *cpu) {
    return FIELD_EX64_IDREG(&cpu->isar, ID_AA64DFR0, CTX_CMPS) + 1; // :1138
}
```

### TCG DisasContext 中的单步状态

```c
// target/arm/tcg/translate.h:125-129
struct DisasContext {
    bool ss_active;     // :128 — 软件单步是否激活
    bool pstate_ss;     // :129 — PSTATE.SS 当前值
    bool is_ldex;       // :134 — 当前指令是否 load-exclusive
};
```

---

## 3. 硬件断点实现

### DBGBVR/DBGBCR 寄存器写入

```c
// target/arm/debug_helper.c:371-400
static void dbgbvr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    ARMCPU *cpu = env_archcpu(env);
    int i = ri->crm;
    raw_write(env, ri, value);
    if (tcg_enabled())
        hw_breakpoint_update(cpu, i);          // :378-379 — 同步到 QEMU 核心
}

static void dbgbcr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    // BAS[3] = BAS[2] 副本, BAS[1] = BAS[0] 副本
    value = deposit64(value, 6, 1, extract64(value, 5, 1));  // :393
    value = deposit64(value, 8, 1, extract64(value, 7, 1));  // :394
    raw_write(env, ri, value);
    if (tcg_enabled())
        hw_breakpoint_update(cpu, i);          // :397-398
}
```

### hw_breakpoint_update — 注册到 QEMU 核心

```c
// target/arm/tcg/debug.c:652-736
void hw_breakpoint_update(ARMCPU *cpu, int n) {
    uint64_t bvr = env->cp15.dbgbvr[n];
    uint64_t bcr = env->cp15.dbgbcr[n];

    // 先移除旧断点
    if (env->cpu_breakpoint[n])
        cpu_breakpoint_remove_by_ref(CPU(cpu), env->cpu_breakpoint[n]); // :662

    if (!extract64(bcr, 0, 1))  // E 位 = 0 → 禁用
        return;

    int bt = extract64(bcr, 20, 4);  // :671

    switch (bt) {
    case 0: /* 无链接地址匹配 */
    case 1: /* 链接地址匹配 */
        // BAS 处理: 0b0000=无, 0b0011=addr, 0b1100=addr+2, 0b1111=addr
        int bas = extract64(bcr, 5, 4);
        addr = bvr & ~3ULL;
        if (bas == 0xc) addr += 2;              // :711-712 — Thumb16 偏移
        cpu_breakpoint_insert(CPU(cpu), addr, BP_CPU,
                              &env->cpu_breakpoint[n]);  // :735
        break;

    case 4: case 5: /* 地址不匹配（保留）*/
        return;

    case 2: case 8: case 10: /* 上下文/VMID 匹配（未实现）*/
        return;

    case 3: case 9: case 11: /* 链接上下文（不产生事件）*/
        return;
    }
}
```

### arm_debug_check_breakpoint — 断点命中检查

```c
// target/arm/tcg/debug.c:376-419
bool arm_debug_check_breakpoint(CPUState *cs) {
    // MDSCR_EL1.MDE=0 或不能产生调试异常 → 忽略
    if (extract32(env->cp15.mdscr_el1, 15, 1) == 0           // :387
        || !arm_generate_debug_exceptions(env))               // :388
        return false;

    // 单步 Active-pending 优先于断点
    if (arm_singlestep_active(env) && !(env->pstate & PSTATE_SS)) // :396
        return false;

    // PC 对齐错误优先
    pc = is_a64(env) ? env->pc : env->regs[15];
    if ((is_a64(env) || !env->thumb) && (pc & 3) != 0)       // :404
        return false;

    // 遍历所有断点
    for (n = 0; n < ARRAY_SIZE(env->cpu_breakpoint); n++) {
        if (bp_wp_matches(cpu, n, false))                     // :415
            return true;
    }
    return false;
}
```

---

## 4. Watchpoint 实现

### hw_watchpoint_update — 注册到 QEMU 核心

```c
// target/arm/tcg/debug.c:546-633
void hw_watchpoint_update(ARMCPU *cpu, int n) {
    vaddr wvr = env->cp15.dbgwvr[n];
    uint64_t wcr = env->cp15.dbgwcr[n];
    int flags = BP_CPU | BP_STOP_BEFORE_ACCESS;

    if (!FIELD_EX64(wcr, DBGWCR, E))  // E=0 → 禁用
        return;

    // LSC: 00=禁用, 01=读, 10=写, 11=读写
    switch (FIELD_EX64(wcr, DBGWCR, LSC)) {            // :565
    case 1: flags |= BP_MEM_READ; break;                // :570
    case 2: flags |= BP_MEM_WRITE; break;               // :573
    case 3: flags |= BP_MEM_ACCESS; break;              // :576
    }

    // MASK 字段：覆盖对齐区域（最大 2GB）
    mask = FIELD_EX64(wcr, DBGWCR, MASK);               // :585
    if (mask >= 3) {
        len = 1ULL << mask;                              // :595
        wvr &= ~(len - 1);                              // :601
    } else {
        // BAS 字段：字节地址选择
        int bas = FIELD_EX64(wcr, DBGWCR, BAS);         // :604
        basstart = ctz32(bas);                           // :626
        len = cto32(bas >> basstart);                    // :627
        wvr += basstart;                                 // :628
    }

    cpu_watchpoint_insert(CPU(cpu), wvr, len, flags,
                          &env->cpu_watchpoint[n]);      // :631-632
}
```

### check_watchpoints — Watchpoint 命中检查

```c
// target/arm/tcg/debug.c:354-374
static bool check_watchpoints(ARMCPU *cpu) {
    // MDSCR_EL1.MDE=0 或不能产生调试异常 → 忽略
    if (extract32(env->cp15.mdscr_el1, 15, 1) == 0      // :363
        || !arm_generate_debug_exceptions(env))          // :364
        return false;

    for (n = 0; n < ARRAY_SIZE(env->cpu_watchpoint); n++) {
        if (bp_wp_matches(cpu, n, true))                 // :369
            return true;
    }
    return false;
}
```

---

## 5. bp_wp_matches 统一匹配逻辑

断点和 Watchpoint 共享同一匹配函数：

```c
// target/arm/tcg/debug.c:255-351
static bool bp_wp_matches(ARMCPU *cpu, int n, bool is_wp) {
    bool is_secure = arm_is_secure(env);
    int access_el = arm_current_el(env);

    if (is_wp) {
        // Watchpoint: 检查 BP_WATCHPOINT_HIT 标志
        if (!wp || !(wp->flags & BP_WATCHPOINT_HIT))
            return false;                                // :270-271
        cr = env->cp15.dbgwcr[n];
        // LDRT/STRT 非特权访问 → access_el = 0
        if (wp->hitattrs.user) access_el = 0;           // :280
    } else {
        // 断点: 检查 PC 匹配
        uint64_t pc = is_a64(env) ? env->pc : env->regs[15];
        if (!env->cpu_breakpoint[n] || env->cpu_breakpoint[n]->pc != pc)
            return false;                                // :285-286
        cr = env->cp15.dbgbcr[n];
    }

    // === SSC (安全状态控制) ===
    pac = FIELD_EX64(cr, DBGWCR, PAC);                   // :303
    hmc = FIELD_EX64(cr, DBGWCR, HMC);                   // :304
    ssc = FIELD_EX64(cr, DBGWCR, SSC);                   // :305

    switch (ssc) {
    case 1: case 3: if (is_secure) return false; break;   // :311-314
    case 2: if (!is_secure) return false; break;          // :316-318
    }

    // === EL 控制 (PAC + HMC) ===
    switch (access_el) {
    case 3: case 2: if (!hmc) return false; break;        // :326-328
    case 1: if (!(pac & 1)) return false; break;          // :331
    case 0: if (!(pac & 2)) return false; break;          // :336
    }

    // === 链接断点检查 ===
    wt = FIELD_EX64(cr, DBGWCR, WT);                     // :344
    lbn = FIELD_EX64(cr, DBGWCR, LBN);                   // :345
    if (wt && !linked_bp_matches(cpu, lbn))               // :347
        return false;

    return true;
}
```

### linked_bp_matches — 链接断点检查

```c
// target/arm/tcg/debug.c:172-253
static bool linked_bp_matches(ARMCPU *cpu, int lbn) {
    uint64_t bcr = env->cp15.dbgbcr[lbn];

    if (!(bcr & 1)) return false;  // 链接断点禁用

    int bt = extract64(bcr, 20, 4);
    switch (bt) {
    case 3:  /* 上下文 ID 匹配 */
        // 根据当前 EL 选择 CONTEXTIDR_EL1 或 CONTEXTIDR_EL2
        contextidr = env->cp15.contextidr_el[el == 2 ? 2 : 1];
        break;
    case 7:  /* CONTEXTIDR_EL1 匹配 */
        contextidr = env->cp15.contextidr_el[1];
        break;
    case 13: /* CONTEXTIDR_EL2 匹配 */
        contextidr = env->cp15.contextidr_el[2];
        break;
    default:
        return false;  // VMID/全上下文匹配未实现
    }
    return contextidr == (uint32_t)env->cp15.dbgbvr[lbn];   // :252
}
```

---

## 6. MDSCR_EL1 与调试使能

### 寄存器定义

```c
// target/arm/debug_helper.c:192-199
{ .name = "MDSCR_EL1", .state = ARM_CP_STATE_BOTH,
  .cp = 14, .opc0 = 2, .opc1 = 0, .crn = 0, .crm = 2, .opc2 = 2,
  .access = PL1_RW, .accessfn = access_tda,
  .fgt = FGT_MDSCR_EL1,
  .fieldoffset = offsetof(CPUARMState, cp15.mdscr_el1),
  .resetvalue = 0 },
```

### MDSCR_EL1 关键位

```
bit[0]:  SS    — 软件单步使能
bit[13]: KDE   — 内核调试事件使能（same-EL 调试）
bit[15]: MDE   — 监视调试事件使能（断点/Watchpoint）
bit[27]: TDE   — [MDCR_EL2] 调试事件路由到 EL2
```

### 调试异常使能判断

```c
// target/arm/tcg/debug.c:64-92
static bool aa64_generate_debug_exceptions(CPUARMState *env) {
    int cur_el = arm_current_el(env);

    if (cur_el == 3) return false;                        // :69-71 — EL3 无调试

    // MDCR_EL3.SDD 禁用安全状态调试
    if (arm_is_secure_below_el3(env)
        && extract32(env->cp15.mdcr_el3, 16, 1))         // :74-76
        return false;

    int debug_el = arm_debug_target_el(env);

    if (cur_el == debug_el) {
        // 同 EL 调试需要 MDSCR.KDE=1 且 PSTATE.D=0
        return extract32(env->cp15.mdscr_el1, 13, 1)     // :86
            && !(env->daif & PSTATE_D);                   // :87
    }

    // 目标 EL 必须高于当前 EL
    return debug_el > cur_el;                             // :91
}

// :148-158 — 统一入口
bool arm_generate_debug_exceptions(CPUARMState *env) {
    // OS Lock 或 DoubleLock 时禁用调试
    if ((env->cp15.oslsr_el1 & 1) || (env->cp15.osdlr_el1 & 1)) // :150
        return false;
    if (is_a64(env))
        return aa64_generate_debug_exceptions(env);
    else
        return aa32_generate_debug_exceptions(env);
}
```

---

## 7. 调试异常路由

### arm_debug_target_el

```c
// target/arm/tcg/debug.c:18-41
static int arm_debug_target_el(CPUARMState *env) {
    if (arm_feature(env, ARM_FEATURE_M))
        return 1;

    bool route_to_el2 = false;
    if (arm_is_el2_enabled(env)) {
        route_to_el2 = env->cp15.hcr_el2 & HCR_TGE ||   // :29
                        env->cp15.mdcr_el2 & MDCR_TDE;   // :30
    }

    if (route_to_el2) return 2;                           // :33-34
    if (arm_feature(env, ARM_FEATURE_EL3) &&
        !arm_el_is_aa64(env, 3) && secure)
        return 3;                                         // :35-37
    return 1;                                             // :39
}
```

### raise_exception_debug

```c
// :47-61
static void raise_exception_debug(CPUARMState *env, uint32_t excp, uint32_t syndrome) {
    int debug_el = arm_debug_target_el(env);
    int cur_el = arm_current_el(env);

    assert(debug_el >= cur_el);
    // same_el 位编入 syndrome 的 EC 字段
    syndrome |= (debug_el == cur_el) << R_SYNDROME_EC_SHIFT;  // :59
    raise_exception(env, excp, syndrome, debug_el);            // :60
}
```

### 异常综合征 EC 值

```c
// target/arm/syndrome.h:68-77
EC_BREAKPOINT             = 0x30,  // 硬件断点（低 EL → 高 EL）
EC_BREAKPOINT_SAME_EL     = 0x31,  // 硬件断点（同 EL）
EC_SOFTWARESTEP           = 0x32,  // 软件单步（低 EL → 高 EL）
EC_SOFTWARESTEP_SAME_EL   = 0x33,  // 软件单步（同 EL）
EC_WATCHPOINT             = 0x34,  // Watchpoint（低 EL → 高 EL）
EC_WATCHPOINT_SAME_EL     = 0x35,  // Watchpoint（同 EL）
EC_AA32_BKPT              = 0x38,  // AArch32 BKPT 指令
EC_AA64_BKPT              = 0x3c,  // AArch64 BRK 指令
```

---

## 8. 软件单步（Software Step）

### 单步使能检查

```c
// target/arm/tcg/debug.c:164-169
bool arm_singlestep_active(CPUARMState *env) {
    return extract32(env->cp15.mdscr_el1, 0, 1)              // MDSCR.SS=1
        && arm_el_is_aa64(env, arm_debug_target_el(env))     // 目标 EL 是 AArch64
        && arm_generate_debug_exceptions(env);                // 调试异常已使能
}
```

### TB flags 传播

```c
// target/arm/tcg/hflags.c:77-89
// ss_active = arm_singlestep_active(env)
// 写入 TB flags，让翻译阶段知道需要生成单步代码
```

### gen_ss_advance — 状态推进

```c
// target/arm/tcg/translate.h:414-421
static inline void gen_ss_advance(DisasContext *s) {
    if (s->ss_active) {
        s->pstate_ss = 0;
        clear_pstate_bits(PSTATE_SS);   // :419 — Active-not-pending → Active-pending
    }
}
```

### 单步状态机

```
                    MDSCR.SS=1
                        │
      ┌─────────────────▼───────────────────┐
      │  Active-not-pending                  │
      │  PSTATE.SS = 1                       │
      │  执行一条指令                         │
      │  gen_ss_advance() → PSTATE.SS = 0    │
      └─────────────────┬───────────────────┘
                        │
      ┌─────────────────▼───────────────────┐
      │  Active-pending                      │
      │  PSTATE.SS = 0                       │
      │  gen_step_complete_exception()       │
      │  → exception_swstep()                │
      │  → Software Step 异常 (EC 0x32/33)   │
      └─────────────────┬───────────────────┘
                        │
      ┌─────────────────▼───────────────────┐
      │  异常处理（Guest 调试器）             │
      │  设置 PSTATE.SS=1 → 返回单步          │
      │  或清除 MDSCR.SS → 停止单步           │
      └─────────────────────────────────────┘
```

### gen_step_complete_exception

```c
// target/arm/tcg/translate-a64.c:511-520
static void gen_step_complete_exception(DisasContext *s) {
    // 完成一条指令后，从 Active-not-pending 转到 Active-pending
    // 然后产生单步异常
    // 选择优先 swstep 异常而非异步异常
    gen_ss_advance(s);
    gen_swstep_exception(s, 1, s->is_ldex);
}
```

### 单步异常 helper

```c
// target/arm/tcg/debug.c:541-543
void HELPER(exception_swstep)(CPUARMState *env, uint32_t syndrome) {
    raise_exception_debug(env, EXCP_UDEF, syndrome);  // Software Step
}
```

---

## 9. BRK 指令与软件断点

### BRK 指令译码

```c
// target/arm/tcg/translate-a64.c:3207-3210
static bool trans_BRK(DisasContext *s, arg_i *a) {
    gen_exception_bkpt_insn(s, syn_aa64_bkpt(a->imm));  // :3209
    return true;
}
```

### gen_exception_bkpt_insn

```c
// target/arm/tcg/translate-a64.c:504-509
static void gen_exception_bkpt_insn(DisasContext *s, uint32_t syndrome) {
    gen_a64_update_pc(s, 0);                              // :506 — 更新 PC
    gen_helper_exception_bkpt_insn(tcg_env, tcg_constant_i32(syndrome)); // :507
    s->base.is_jmp = DISAS_NORETURN;                      // :508
}
```

### HELPER(exception_bkpt_insn) — 异常产生

```c
// target/arm/tcg/debug.c:514-538
void HELPER(exception_bkpt_insn)(CPUARMState *env, uint32_t syndrome) {
    int debug_el = arm_debug_target_el(env);
    int cur_el = arm_current_el(env);

    env->exception.fsr = arm_debug_exception_fsr(env);    // :520
    env->exception.vaddress = 0;                          // :526 — FAR UNKNOWN

    // BRK 特殊：如果目标 EL < 当前 EL → 取当前 EL
    if (debug_el < cur_el)
        debug_el = cur_el;                                // :535-536

    raise_exception(env, EXCP_BKPT, syndrome, debug_el);  // :538
}
```

**BRK 特殊性**：与其他调试异常不同，BRK 总是产生异常。如果配置的调试目标 EL 低于当前 EL，异常被提升到当前 EL。

---

## 10. 调试异常分发

### arm_debug_excp_handler — 核心分发函数

```c
// target/arm/tcg/debug.c:464-508
void arm_debug_excp_handler(CPUState *cs) {
    CPUWatchpoint *wp_hit = cs->watchpoint_hit;

    if (wp_hit) {
        // === Watchpoint 命中 ===
        if (wp_hit->flags & BP_CPU) {                     // :475
            bool wnr = (wp_hit->flags & BP_WATCHPOINT_HIT_WRITE) != 0; // :476
            cs->watchpoint_hit = NULL;

            env->exception.fsr = arm_debug_exception_fsr(env); // :480
            env->exception.vaddress = wp_hit->hitaddr;         // :481
            raise_exception_debug(env, EXCP_DATA_ABORT,
                                  syn_watchpoint(0, 0, wnr));  // :482-483
        }
    } else {
        // === 断点命中 ===
        uint64_t pc = is_a64(env) ? env->pc : env->regs[15];

        // GDB 断点优先
        if (cpu_breakpoint_test(cs, pc, BP_GDB)                // :494
            || !cpu_breakpoint_test(cs, pc, BP_CPU))           // :495
            return;  // 交给 GDB 处理或不是 CPU 断点

        env->exception.fsr = arm_debug_exception_fsr(env);     // :499
        env->exception.vaddress = 0;                           // :505
        raise_exception_debug(env, EXCP_PREFETCH_ABORT,
                              syn_breakpoint(0));               // :506
    }
}
```

### arm_debug_exception_fsr

```c
// target/arm/tcg/debug.c:437-462
static uint32_t arm_debug_exception_fsr(CPUARMState *env) {
    ARMMMUFaultInfo fi = { .type = ARMFault_Debug };
    int target_el = arm_debug_target_el(env);

    // 选择 LPAE 或短描述符 FSR 格式
    if (target_el == 2 || arm_el_is_aa64(env, target_el))
        using_lpae = true;                                // :445-446

    return using_lpae ? arm_fi_to_lfsc(&fi) : arm_fi_to_sfsc(&fi);
}
```

---

## 11. GDB Stub 集成

### 软件断点

GDB 通过 `gdb_breakpoint_insert/remove()` 管理软件断点（`gdbstub/gdbstub.c:1175-1199`）。这些断点使用 `BP_GDB` 标志注册到 QEMU 核心。

### 优先级

在 `arm_debug_excp_handler` 中（:494-496），GDB 断点优先于 CPU 硬件断点检查。如果 PC 处有 GDB 断点，则返回让 GDB 处理。

### ARM GDB 寄存器

```
target/arm/gdbstub.c     — ARM32/ARM64 通用寄存器读写
target/arm/gdbstub64.c   — AArch64 特定寄存器（SVE/SME/TLS 等）
```

### 硬件断点支持

```c
// target/arm/hyp_gdbstub.c:17-170
// 为 GDB 提供硬件断点/Watchpoint 后端
// insert_hw_breakpoint() — 设置 DBGBVR/DBGBCR
// delete_hw_breakpoint() — 清除
// insert_hw_watchpoint() — 设置 DBGWVR/DBGWCR
// delete_hw_watchpoint() — 清除
```

---

## 12. KVM 调试集成

### 调试退出处理

```c
// target/arm/kvm.c:1501-1530
// KVM 调试退出分发:
case EC_SOFTWARESTEP:       // :1495 — 单步（未完全支持）
    error_report("guest single-step unsupported");
    return false;

case EC_AA64_BKPT:          // :1507 — BRK 软件断点
    if (kvm_find_sw_breakpoint(cs, env->pc))
        return true;        // GDB 软件断点命中
    break;

case EC_BREAKPOINT:         // :1512 — 硬件断点
    if (find_hw_breakpoint(cs, env->pc))
        return true;
    break;

case EC_WATCHPOINT:         // :1517 — Watchpoint
    CPUWatchpoint *wp = find_hw_watchpoint(cs, debug_exit->far);
    if (wp) {
        cs->watchpoint_hit = wp;
        return true;
    }
    break;
```

### kvm_arch_update_guest_debug

```c
// target/arm/kvm.c:1615-1624
void kvm_arch_update_guest_debug(CPUState *cs, struct kvm_guest_debug *dbg) {
    if (kvm_sw_breakpoints_active(cs))
        dbg->control |= KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_SW_BP;  // :1618

    if (kvm_arm_hw_debug_active(ARM_CPU(cs))) {
        dbg->control |= KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_HW;     // :1621
        kvm_arm_copy_hw_debug_data(&dbg->arch);                          // :1622
    }
}
```

### KVM 软件断点补丁

```c
// target/arm/kvm.c:2509-2530
// AArch64: 用 BRK #0 (0xd4200000) 替换原指令
// AArch32: 用 BKPT #0 替换
```

---

## 13. 总结

### 调试子系统层次

```
┌──────────────────────────────────────────────────────┐
│  Guest 层：调试系统寄存器                              │
│  DBGBVR/DBGBCR — 16 个硬件断点                       │
│  DBGWVR/DBGWCR — 16 个硬件 Watchpoint                │
│  MDSCR_EL1 — MDE/KDE/SS 控制                         │
│  BRK #imm — 软件断点指令                              │
├──────────────────────────────────────────────────────┤
│  寄存器写入层：debug_helper.c                          │
│  dbgbvr_write → hw_breakpoint_update                  │
│  dbgwvr_write → hw_watchpoint_update                  │
│  → cpu_breakpoint/watchpoint_insert (QEMU 核心)       │
├──────────────────────────────────────────────────────┤
│  命中检查层：tcg/debug.c                              │
│  arm_debug_check_breakpoint → bp_wp_matches           │
│  arm_debug_check_watchpoint → check_watchpoints       │
│  4 重过滤: SSC(安全) → PAC/HMC(EL) → WT(链接) → MDE │
├──────────────────────────────────────────────────────┤
│  异常产生层                                           │
│  arm_debug_excp_handler → raise_exception_debug       │
│  arm_debug_target_el → TDE/TGE 路由                   │
│  syndrome: EC_BREAKPOINT/WATCHPOINT/SOFTWARESTEP/BKPT │
├──────────────────────────────────────────────────────┤
│  外部调试层                                           │
│  GDB Stub: BP_GDB 优先于 BP_CPU                      │
│  KVM: KVM_GUESTDBG_USE_HW/SW_BP → debug exit 处理    │
└──────────────────────────────────────────────────────┘
```

### 设计亮点

1. **统一匹配函数**：`bp_wp_matches()` 同时处理断点和 Watchpoint，利用 WCR/BCR 的相同布局（SSC/HMC/PAC/WT/LBN 字段位置一致），减少代码重复。

2. **惰性断点注册**：Guest 写入 DBGBVR/DBGBCR 时，通过 `hw_breakpoint_update()` 立即同步到 QEMU 核心的 `cpu_breakpoint_insert()`。QEMU 核心在 TB 执行边界自动检查。

3. **三层使能门控**：MDSCR.MDE 全局使能 → `arm_generate_debug_exceptions()` 检查 OS Lock/MDCR 路由/PSTATE.D → `bp_wp_matches()` 检查 SSC/PAC/HMC 过滤。层层递进。

4. **单步状态机**：PSTATE.SS=1（Active-not-pending）→ `gen_ss_advance()` 清除为 0（Active-pending）→ 下条指令前产生 Software Step 异常。TCG 在每条指令翻译时检查 `ss_active`。

5. **BRK 异常不可屏蔽**：BRK 始终产生异常（与断点/Watchpoint 不同），如果目标 EL 低于当前 EL，提升到当前 EL。这保证了软件断点的可靠性。

6. **GDB 优先级**：`arm_debug_excp_handler()` 中 GDB 断点（BP_GDB）优先于 CPU 硬件断点（BP_CPU）。GDB 断点命中时直接返回给 GDB 处理，不产生 Guest 异常。

7. **KVM 透明代理**：KVM 模式下，硬件断点/Watchpoint 通过 `kvm_guest_debug_arch` 结构传递给内核。调试退出时按 EC 分发（BRK→SW_BP、BREAKPOINT→HW_BP、WATCHPOINT→HW_WP）。

---

**关键源文件**：
- `target/arm/tcg/debug.c` — **TCG 调试核心**（匹配逻辑、异常分发、断点/WP 更新、单步）
- `target/arm/debug_helper.c` — **调试寄存器定义**（MDSCR_EL1、DBGBVR/BCR/WVR/WCR、OSLAR、OSDLR）
- `target/arm/tcg/translate.h` — gen_ss_advance()、gen_swstep_exception()
- `target/arm/tcg/translate-a64.c` — trans_BRK()、gen_exception_bkpt_insn()、gen_step_complete_exception()
- `target/arm/syndrome.h` — EC 值定义（BREAKPOINT/WATCHPOINT/SOFTWARESTEP/BKPT）
- `target/arm/cpu.h` — CPUARMState 调试字段
- `target/arm/kvm.c` — KVM 调试退出处理、hw breakpoint/watchpoint 管理
- `target/arm/hyp_gdbstub.c` — GDB 硬件断点/Watchpoint 后端
