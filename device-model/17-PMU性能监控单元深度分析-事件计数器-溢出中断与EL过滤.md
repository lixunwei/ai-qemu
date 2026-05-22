# PMU 性能监控单元深度分析 — 事件计数器、溢出中断与 EL 过滤

> 基于 QEMU 11.0.50 源码分析，聚焦 ARM PMUv3 性能监控单元全栈实现：
> pm_event 事件表（SW_INCR/INST_RETIRED/CPU_CYCLES）、惰性求值计数机制
> （pmccntr_op_start/finish、pmevcntr_op_start/finish）、
> EL/安全状态过滤（pmu_counter_enabled 的 P/U/NSK/NSU/NSH/M 位）、
> 溢出检测与 GICv3 PPI 中断、PMCR 控制寄存器、SW_INCR 软件递增、
> KVM PMU 透传、pmu_timer 溢出预测。

---

## 目录

1. [PMU 子系统全景](#1-pmu-子系统全景)
2. [CPUARMState 中的 PMU 状态](#2-cpuarmstate-中的-pmu-状态)
3. [PMU 事件定义与事件表](#3-pmu-事件定义与事件表)
4. [惰性求值计数机制](#4-惰性求值计数机制)
5. [EL/安全状态过滤](#5-el安全状态过滤)
6. [溢出检测与中断](#6-溢出检测与中断)
7. [PMCR 控制寄存器](#7-pmcr-控制寄存器)
8. [SW_INCR 软件递增事件](#8-sw_incr-软件递增事件)
9. [PMU 系统寄存器定义](#9-pmu-系统寄存器定义)
10. [pmu_timer 溢出预测定时器](#10-pmu_timer-溢出预测定时器)
11. [KVM PMU 集成](#11-kvm-pmu-集成)
12. [总结](#12-总结)

---

## 1. PMU 子系统全景

```
┌─────────────────────────────────────────────────────────────┐
│  Guest 视角：PMU 系统寄存器                                  │
│  PMCR_EL0 / PMCNTENSET_EL0 / PMCCNTR_EL0 / PMEVCNTR<n>    │
│  PMEVTYPER<n> / PMOVSSET / PMINTENSET_EL1 / PMSWINC        │
│                                                             │
│  访问时触发 readfn/writefn →                                │
│  pmu_op_start() → 快照计数 → 检测溢出                       │
│  pmu_op_finish() → 计算 delta → 预测下次溢出 → timer_mod    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  惰性求值核心                                               │
│  cpregs-pmu.c                                               │
│                                                             │
│  ┌─────────────────────────────────────────┐                │
│  │  pm_events[] 事件表                     │                │
│  │  SW_INCR(0x000) → get_count=0, ns=-1    │                │
│  │  INST_RETIRED(0x008) → icount_get_raw() │                │
│  │  CPU_CYCLES(0x011) → VIRTUAL÷ns_per_cyc │                │
│  │  STALL_FRONTEND(0x023) → always 0       │                │
│  └─────────────────────────────────────────┘                │
│                                                             │
│  pmu_counter_enabled(): PMCR.E × PMCNTEN × ¬prohibited     │
│                         × ¬filtered(P/U/NSK/NSU/NSH/M)     │
│                                                             │
│  pmu_update_irq(): PMCR.E ∧ (PMOVSR & PMINTEN) → set_irq  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  QEMUTimer 集成                                             │
│  cpu->pmu_timer (QEMU_CLOCK_VIRTUAL)                        │
│  arm_pmu_timer_cb() → pmu_op_start() + pmu_op_finish()     │
│  预测溢出时间 → timer_mod_anticipate_ns()                   │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                    GICv3 PPI 中断
                    cpu->pmu_interrupt
```

---

## 2. CPUARMState 中的 PMU 状态

```c
// target/arm/cpu.h:444-556 (cp15 结构内)
struct {
    // === 控制与使能 ===
    uint64_t c9_pmcr;          // :444 — PMCR_EL0 控制寄存器
    uint64_t c9_pmcnten;       // :445 — PMCNTENSET/CLR 计数器使能位图
    uint64_t c9_pmovsr;        // :446 — PMOVSSET/CLR 溢出状态位图
    uint64_t c9_pmuserenr;     // :447 — PMUSERENR_EL0 用户态访问使能
    uint64_t c9_pminten;       // :449 — PMINTENSET/CLR 中断使能位图
    uint64_t c9_pmselr;        // （PMSELR_EL0 计数器选择）

    // === 周期计数器 ===
    uint64_t c15_ccnt;         // :544 — PMCCNTR_EL0 当前值
    uint64_t c15_ccnt_delta;   // :552 — 上次快照时的底层计数

    // === 事件计数器（最多 31 个） ===
    uint64_t c14_pmevcntr[31];       // :553 — PMEVCNTR<n> 当前值
    uint64_t c14_pmevcntr_delta[31]; // :554 — 上次快照时的底层计数
    uint64_t c14_pmevtyper[31];      // :555 — PMEVTYPER<n> 事件类型+过滤

    // === 过滤 ===
    uint64_t pmccfiltr_el0;    // :556 — PMCCFILTR_EL0 周期计数器过滤
} cp15;
```

### ARMCPU 中的 PMU 资源

```c
// target/arm/cpu.h:998-1000
struct ARMCPU {
    bool has_pmu;              // :998 — 是否有 PMU
    qemu_irq pmu_interrupt;   // PMU 溢出中断线 → GICv3 PPI
    QEMUTimer *pmu_timer;     // 溢出预测定时器（VIRTUAL 时钟）
};
```

### 计数器数量

```c
// target/arm/internals.h:1719-1724
static inline uint32_t pmu_num_counters(CPUARMState *env) {
    ARMCPU *cpu = env_archcpu(env);
    return (cpu->isar.reset_pmcr_el0 & PMCRN_MASK) >> PMCRN_SHIFT;
}

// :1727-1730
static inline uint64_t pmu_counter_mask(CPUARMState *env) {
    return (1ULL << 31) | ((1ULL << pmu_num_counters(env)) - 1);
    // bit31 = CCNT, bit0~N-1 = 事件计数器
}
```

---

## 3. PMU 事件定义与事件表

### pm_event 结构

```c
// target/arm/cpregs-pmu.c:37-53
typedef struct pm_event {
    uint16_t number;           // PMEVTYPER.evtCount（16 位事件编号）
    bool (*supported)(CPUARMState *);  // 该事件是否受支持
    uint64_t (*get_count)(CPUARMState *);  // 获取底层事件计数
    int64_t (*ns_per_count)(uint64_t);     // 每 count 所需纳秒（溢出预测用）
    // 返回 -1 表示永不溢出
} pm_event;
```

### 事件表

```c
// target/arm/cpregs-pmu.c:137-170
static const pm_event pm_events[] = {
    { .number = 0x000, /* SW_INCR — 软件递增 */
      .get_count = swinc_get_count,      // 始终返回 0
      .ns_per_count = swinc_ns_per,      // 返回 -1（不可预测）
    },
    { .number = 0x008, /* INST_RETIRED — 架构指令退休 */
      .supported = instructions_supported, // 仅 icount=precise 时支持
      .get_count = instructions_get_count, // icount_get_raw()
      .ns_per_count = instructions_ns_per, // icount_to_ns()
    },
    { .number = 0x011, /* CPU_CYCLES — 处理器周期 */
      .get_count = cycles_get_count,       // VIRTUAL_NS × FREQ / 1e9
      .ns_per_count = cycles_ns_per,       // cycles × 1e9 / FREQ
    },
    { .number = 0x023, /* STALL_FRONTEND — 前端停顿 */
      .supported = pmuv3p1_events_supported,
      .get_count = zero_event_get_count,   // 始终 0
      .ns_per_count = zero_event_ns_per,   // -1
    },
    { .number = 0x024, /* STALL_BACKEND */ ... },
    { .number = 0x03c, /* STALL */ ... },
};
```

### 事件计数获取函数

```c
// :16 — 固定 1 GHz 频率
#define ARM_CPU_FREQ 1000000000

// :78-86 — CPU_CYCLES 获取
static uint64_t cycles_get_count(CPUARMState *env) {
    return muldiv64(qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL),
                   ARM_CPU_FREQ, NANOSECONDS_PER_SECOND);  // :81-82
    // = VIRTUAL_NS × 1GHz / 1e9 = VIRTUAL_NS（1:1 映射！）
}

// :100-103 — INST_RETIRED 获取
static uint64_t instructions_get_count(CPUARMState *env) {
    assert(icount_enabled() == ICOUNT_PRECISE);
    return (uint64_t)icount_get_raw();  // icount 精确模式下的原始指令数
}
```

**关键观察**：ARM_CPU_FREQ = 1GHz，因此 `cycles_get_count()` 的结果在数值上等于 `qemu_clock_get_ns(VIRTUAL)`。每个 Guest "CPU 周期" 对应 1 纳秒虚拟时间。

---

## 4. 惰性求值计数机制

QEMU PMU 采用**惰性求值**（lazy evaluation）：不在每个周期/指令实时递增计数器，而是在读写时快照底层计数，计算差值。

### 周期计数器 op_start/op_finish

```c
// target/arm/cpregs-pmu.c:479-536
static void pmccntr_op_start(CPUARMState *env) {
    uint64_t cycles = cycles_get_count(env);              // :481

    if (pmu_counter_enabled(env, 31)) {                   // :483
        uint64_t eff_cycles = cycles;
        if (pmccntr_clockdiv_enabled(env))
            eff_cycles /= 64;                             // :485-486

        // 新计数值 = 当前底层计数 - 上次快照时的底层计数
        uint64_t new_pmccntr = eff_cycles - env->cp15.c15_ccnt_delta; // :489

        // === 溢出检测 ===
        uint64_t overflow_mask = env->cp15.c9_pmcr & PMCRLC ?
                                 1ull << 63 : 1ull << 31;             // :491-492
        if (env->cp15.c15_ccnt & ~new_pmccntr & overflow_mask) {
            // 旧值最高位为1，新值最高位为0 → 发生了溢出
            env->cp15.c9_pmovsr |= (1ULL << 31);                     // :494
            pmu_update_irq(env);                                      // :495
        }

        env->cp15.c15_ccnt = new_pmccntr;                            // :498
    }
    env->cp15.c15_ccnt_delta = cycles;                                // :500
}

static void pmccntr_op_finish(CPUARMState *env) {
    if (pmu_counter_enabled(env, 31)) {
        // 计算距离溢出还有多少周期
        uint64_t remaining_cycles = -env->cp15.c15_ccnt;             // :513
        if (!(env->cp15.c9_pmcr & PMCRLC))
            remaining_cycles = (uint32_t)remaining_cycles;           // :515

        int64_t overflow_in = cycles_ns_per(remaining_cycles);       // :517

        if (overflow_in > 0) {
            int64_t overflow_at;
            if (!sadd64_overflow(qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL),
                                 overflow_in, &overflow_at)) {
                timer_mod_anticipate_ns(cpu->pmu_timer, overflow_at); // :525
            }
        }

        // 重算 delta 用于下次 op_start
        uint64_t prev_cycles = env->cp15.c15_ccnt_delta;
        if (pmccntr_clockdiv_enabled(env))
            prev_cycles /= 64;
        env->cp15.c15_ccnt_delta = prev_cycles - env->cp15.c15_ccnt; // :534
    }
}
```

### 事件计数器 op_start/op_finish

```c
// :538-590
static void pmevcntr_op_start(CPUARMState *env, uint8_t counter) {
    uint16_t event = env->cp15.c14_pmevtyper[counter] & PMXEVTYPER_EVTCOUNT;
    uint64_t count = 0;
    if (event_supported(event)) {
        count = pm_events[supported_event_map[event]].get_count(env); // :545
    }

    if (pmu_counter_enabled(env, counter)) {
        uint64_t new_pmevcntr = count - env->cp15.c14_pmevcntr_delta[counter]; // :549
        uint64_t overflow_mask = pmevcntr_is_64_bit(env, counter) ?
            1ULL << 63 : 1ULL << 31;                                            // :550-551

        // 溢出检测：同 CCNT 逻辑
        if (env->cp15.c14_pmevcntr[counter] & ~new_pmevcntr & overflow_mask) {
            env->cp15.c9_pmovsr |= (1 << counter);                              // :554
            pmu_update_irq(env);                                                  // :555
        }
        env->cp15.c14_pmevcntr[counter] = new_pmevcntr;                         // :557
    }
    env->cp15.c14_pmevcntr_delta[counter] = count;                              // :559
}

static void pmevcntr_op_finish(CPUARMState *env, uint8_t counter) {
    if (pmu_counter_enabled(env, counter)) {
        // 预测溢出
        uint64_t delta = -(env->cp15.c14_pmevcntr[counter] + 1);               // :568
        int64_t overflow_in = pm_events[event_idx].ns_per_count(delta);         // :574
        if (overflow_in > 0) {
            timer_mod_anticipate_ns(cpu->pmu_timer, overflow_at);               // :582
        }
        // 重算 delta
        env->cp15.c14_pmevcntr_delta[counter] -= env->cp15.c14_pmevcntr[counter]; // :587-588
    }
}
```

### 聚合操作

```c
// :592-608
void pmu_op_start(CPUARMState *env) {
    pmccntr_op_start(env);
    for (i = 0; i < pmu_num_counters(env); i++)
        pmevcntr_op_start(env, i);
}

void pmu_op_finish(CPUARMState *env) {
    pmccntr_op_finish(env);
    for (i = 0; i < pmu_num_counters(env); i++)
        pmevcntr_op_finish(env, i);
}
```

### EL 切换时的快照

```c
// :610-618
void pmu_pre_el_change(ARMCPU *cpu, void *ignored) {
    pmu_op_start(&cpu->env);   // EL 切换前：快照当前计数
}

void pmu_post_el_change(ARMCPU *cpu, void *ignored) {
    pmu_op_finish(&cpu->env);  // EL 切换后：重算 delta（新 EL 可能改变过滤）
}
```

**设计精髓**：EL 切换时调用 pmu_pre/post_el_change，确保在 EL 过滤条件变化前后正确拆分计数。

---

## 5. EL/安全状态过滤

```c
// target/arm/cpregs-pmu.c:336-427
static bool pmu_counter_enabled(CPUARMState *env, uint8_t counter) {
    // === 第一层：全局使能 ===
    if (!arm_feature(env, ARM_FEATURE_PMU)) return false;       // :351

    // PMCR.E 或 MDCR_EL2.HPME（取决于计数器归属）
    if (counter < hpmn || counter == 31)
        e = env->cp15.c9_pmcr & PMCRE;                          // :360
    else
        e = mdcr_el2 & MDCR_HPME;                               // :362

    enabled = e && (env->cp15.c9_pmcnten & (1 << counter));     // :364

    // === 第二层：禁止计数 ===
    // EL2 + 计数器 < HPMN → MDCR_EL2.HPMD 禁止
    if (el == 2 && (counter < hpmn || counter == 31))
        prohibited = mdcr_el2 & MDCR_HPMD;                      // :368

    // 安全状态 → MDCR_EL3.SPME 禁止
    if (secure) prohibited = prohibited || !(env->cp15.mdcr_el3 & MDCR_SPME); // :371

    // CCNT 特殊处理：PMCR.DP + SCCD/HCCD
    if (counter == 31) {
        prohibited = prohibited && env->cp15.c9_pmcr & PMCRDP;  // :380
        if (secure) prohibited |= (mdcr_el3 & MDCR_SCCD);       // :383
        if (el == 2) prohibited |= (mdcr_el2 & MDCR_HCCD);      // :386
    }

    // === 第三层：EL 过滤 ===
    if (counter == 31)
        filter = env->cp15.pmccfiltr_el0;                        // :392
    else
        filter = env->cp15.c14_pmevtyper[counter];               // :394

    p   = filter & PMXEVTYPER_P;     // EL1 过滤
    u   = filter & PMXEVTYPER_U;     // EL0 过滤
    nsk = EL3 && (filter & PMXEVTYPER_NSK);  // 非安全 EL1
    nsu = EL3 && (filter & PMXEVTYPER_NSU);  // 非安全 EL0
    nsh = EL2 && (filter & PMXEVTYPER_NSH);  // EL2 过滤
    m   = AA64 && EL3 && (filter & PMXEVTYPER_M);  // EL3 过滤

    // 按当前 EL 应用过滤
    if (el == 0)      filtered = secure ? u : u != nsu;          // :405-406
    else if (el == 1)  filtered = secure ? p : p != nsk;         // :407-408
    else if (el == 2)  filtered = !nsh;                          // :409-410
    else /* EL3 */     filtered = m != p;                        // :411-412

    // === 第四层：事件支持检查 ===
    if (counter != 31) {
        uint16_t event = filter & PMXEVTYPER_EVTCOUNT;
        if (!event_supported(event)) return false;               // :420-423
    }

    return enabled && !prohibited && !filtered;                  // :426
}
```

### EL 过滤逻辑表

| 当前 EL | Secure | 过滤条件 | 含义 |
|---------|--------|---------|------|
| EL0 | 是 | `U` | U=1 → 不计数 EL0 |
| EL0 | 否 | `U ⊕ NSU` | 两者异或决定是否过滤 |
| EL1 | 是 | `P` | P=1 → 不计数 EL1 |
| EL1 | 否 | `P ⊕ NSK` | 两者异或决定是否过滤 |
| EL2 | — | `¬NSH` | NSH=0 → 不计数 EL2 |
| EL3 | — | `M ⊕ P` | 两者异或决定是否过滤 |

---

## 6. 溢出检测与中断

### 溢出检测

溢出检测发生在 `pmccntr_op_start` 和 `pmevcntr_op_start` 中：

```c
// 判断逻辑（以 CCNT 为例，:493-495）
// overflow_mask = PMCR.LC ? (1<<63) : (1<<31)
if (old_value & ~new_value & overflow_mask) {
    // 旧值的最高位为 1，新值的最高位为 0
    // → 计数器从 MAX 回绕到 0 → 溢出
    env->cp15.c9_pmovsr |= (1ULL << 31);  // 设置 CCNT 溢出位
    pmu_update_irq(env);
}
```

### 中断产生

```c
// target/arm/cpregs-pmu.c:429-434
static void pmu_update_irq(CPUARMState *env) {
    ARMCPU *cpu = env_archcpu(env);
    qemu_set_irq(cpu->pmu_interrupt,
        (env->cp15.c9_pmcr & PMCRE) &&          // PMCR.E 全局使能
        (env->cp15.c9_pminten & env->cp15.c9_pmovsr)); // INTEN & OVSR 有交集
}
```

**中断条件**：`PMCR.E=1` 且 `PMINTENSET & PMOVSSET ≠ 0`

`cpu->pmu_interrupt` 连接到 GICv3 PPI 23（ARM PMU 中断标准分配）。

### PMOVSSET/PMOVSCLR 寄存器

```c
// :788-801
static void pmovsr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    value &= pmu_counter_mask(env);
    env->cp15.c9_pmovsr &= ~value;    // PMOVSCLR: 写 1 清除溢出位
    pmu_update_irq(env);
}

static void pmovsset_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    value &= pmu_counter_mask(env);
    env->cp15.c9_pmovsr |= value;     // PMOVSSET: 写 1 设置溢出位
    pmu_update_irq(env);
}
```

### PMINTENSET/PMINTENCLR

```c
// :993-1008
static void pmintenset_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    value &= pmu_counter_mask(env);
    env->cp15.c9_pminten |= value;    // 写 1 使能中断
    pmu_update_irq(env);               // 可能立即触发中断
}

static void pmintenclr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    value &= pmu_counter_mask(env);
    env->cp15.c9_pminten &= ~value;   // 写 1 清除中断使能
    pmu_update_irq(env);               // 可能取消中断
}
```

---

## 7. PMCR 控制寄存器

### PMCR 位域

```
bit[0]:  E   — 全局使能（Enable all counters）
bit[1]:  P   — 事件计数器复位（Reset event counters）
bit[2]:  C   — 周期计数器复位（Reset cycle counter）
bit[3]:  D   — 时钟分频（Clock divider, CCNT /= 64）
bit[4]:  X   — 导出使能（Export enable）
bit[5]:  DP  — 禁止时禁用 CCNT（Disable CCNT when prohibited）
bit[6]:  LC  — 长周期计数器（64 位溢出而非 32 位）
bit[10:7]: LP — 长事件计数器（PMUv3p5, 64 位）
bit[15:11]: N  — 事件计数器数量（只读）
bit[23:16]: IDCODE — 实现者 ID（只读）
bit[31:24]: IMP — 实现者代码（只读）
```

### PMCR 写入

```c
// target/arm/cpregs-pmu.c:634-655
static void pmcr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    pmu_op_start(env);                                      // :637

    if (value & PMCRC) {
        env->cp15.c15_ccnt = 0;                              // :641 — 复位 CCNT
    }

    if (value & PMCRP) {
        for (i = 0; i < pmu_num_counters(env); i++)
            env->cp15.c14_pmevcntr[i] = 0;                  // :647 — 复位所有事件计数器
    }

    env->cp15.c9_pmcr &= ~PMCR_WRITABLE_MASK;
    env->cp15.c9_pmcr |= (value & PMCR_WRITABLE_MASK);      // :651-652

    pmu_op_finish(env);                                      // :654
}
```

### PMCR 读取（HPMN 替换 N）

```c
// :657-671
static uint64_t pmcr_read(CPUARMState *env, const ARMCPRegInfo *ri) {
    uint64_t pmcr = env->cp15.c9_pmcr;

    // EL0/EL1 + EL2 存在 → 用 MDCR_EL2.HPMN 替换 N
    if (arm_current_el(env) <= 1 && arm_is_el2_enabled(env)) {
        pmcr &= ~PMCRN_MASK;
        pmcr |= (env->cp15.mdcr_el2 & MDCR_HPMN) << PMCRN_SHIFT;   // :667
    }

    return pmcr;
}
```

**虚拟化支持**：Hypervisor 通过 MDCR_EL2.HPMN 将事件计数器分为两组：
- counter 0~(HPMN-1)：Guest 可见（PMCR.E 控制）
- counter HPMN~(N-1)：Hypervisor 私有（MDCR_EL2.HPME 控制）

---

## 8. SW_INCR 软件递增事件

```c
// target/arm/cpregs-pmu.c:673-707
static void pmswinc_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    for (i = 0; i < pmu_num_counters(env); i++) {
        if ((value & (1 << i)) &&                          // :681 — 该计数器的位被设置
            pmu_counter_enabled(env, i) &&                 // :683 — 计数器已使能且未过滤
            (env->cp15.c14_pmevtyper[i] & PMXEVTYPER_EVTCOUNT) == 0x0) { // :685 — 事件是 SW_INCR
            
            pmevcntr_op_start(env, i);                     // :686

            new_pmswinc = env->cp15.c14_pmevcntr[i] + 1;  // :692

            // 溢出检测
            overflow_mask = pmevcntr_is_64_bit(env, i) ?
                1ULL << 63 : 1ULL << 31;                   // :694-695
            if (env->cp15.c14_pmevcntr[i] & ~new_pmswinc & overflow_mask) {
                env->cp15.c9_pmovsr |= (1 << i);          // :698
                pmu_update_irq(env);                        // :699
            }

            env->cp15.c14_pmevcntr[i] = new_pmswinc;      // :702
            pmevcntr_op_finish(env, i);                     // :704
        }
    }
}
```

**SW_INCR 特殊性**：
1. `swinc_get_count()` 始终返回 0 — 因为 SW_INCR 的计数由 PMSWINC 写入直接维护
2. `swinc_ns_per()` 返回 -1 — 无法预测溢出时间（不像 CPU_CYCLES 可以用时间推算）
3. 溢出检测在 pmswinc_write 内部实时执行（不能惰性求值）

---

## 9. PMU 系统寄存器定义

所有 PMU 寄存器在 `cpregs-pmu.c` 中以 `ARMCPRegInfo` 数组定义。

### 核心寄存器表

| 寄存器 | 行号 | readfn | writefn | 说明 |
|--------|------|--------|---------|------|
| PMCR_EL0 | 1212-1222 | pmcr_read | pmcr_write | 控制寄存器 |
| PMCNTENSET_EL0 | 1030-1035 | raw_read | pmcntenset_write | 计数器使能（置位） |
| PMCNTENCLR_EL0 | 1043-1049 | raw_read | pmcntenclr_write | 计数器使能（清除） |
| PMOVSCLR_EL0 | 1057-1064 | raw_read | pmovsr_write | 溢出状态（清除） |
| PMOVSSET_EL0 | 1182-1189 | raw_read | pmovsset_write | 溢出状态（置位） |
| PMSWINC_EL0 | 1070-1075 | — | pmswinc_write | 软件递增（只写） |
| PMSELR_EL0 | 1082-1087 | raw_read | pmselr_write | 计数器选择 |
| PMCCNTR_EL0 | 1088-1095 | pmccntr_read | pmccntr_write | 周期计数器 |
| PMCCFILTR_EL0 | 1102-1109 | raw_read | pmccfiltr_write | 周期计数器过滤 |
| PMXEVTYPER_EL0 | 1115-1120 | pmxevtyper_read | pmxevtyper_write | 间接事件类型 |
| PMXEVCNTR_EL0 | 1126-1131 | pmxevcntr_read | pmxevcntr_write | 间接事件计数 |
| PMINTENSET_EL1 | 1150-1157 | raw_read | pmintenset_write | 中断使能（置位） |
| PMINTENCLR_EL1 | 1164-1170 | raw_read | pmintenclr_write | 中断使能（清除） |
| PMUSERENR_EL0 | 1137-1142 | raw_read | pmuserenr_write | 用户态使能 |

### 动态生成的寄存器

```c
// :1245-1281
for (i = 0; i < pmu_num_counters(env); i++) {
    // 为每个事件计数器生成 4 个寄存器:
    //   PMEVCNTR<i>     (AArch32) — readfn=pmevcntr_readfn
    //   PMEVCNTR<i>_EL0 (AArch64) — readfn=pmevcntr_readfn
    //   PMEVTYPER<i>    (AArch32) — readfn=pmevtyper_readfn
    //   PMEVTYPER<i>_EL0(AArch64) — readfn=pmevtyper_readfn
    // 编码: crm = 8|(i>>3), opc2 = i&7
}
```

### 访问控制

```c
// :22-35
static CPAccessResult access_tpm(CPUARMState *env, const ARMCPRegInfo *ri, bool isread) {
    if (el < 2 && (mdcr_el2 & MDCR_TPM))
        return CP_ACCESS_TRAP_EL2;       // EL2 陷入（TPM 位）
    if (el < 3 && (env->cp15.mdcr_el3 & MDCR_TPM))
        return CP_ACCESS_TRAP_EL3;       // EL3 陷入
    return CP_ACCESS_OK;
}
```

---

## 10. pmu_timer 溢出预测定时器

### 定时器回调

```c
// target/arm/cpregs-pmu.c:620-632
void arm_pmu_timer_cb(void *opaque) {
    ARMCPU *cpu = opaque;

    // 更新所有计数器 → 检测溢出 → 触发中断
    pmu_op_start(&cpu->env);    // :630 — 快照 + 溢出检测
    pmu_op_finish(&cpu->env);   // :631 — 重算 delta + 重新预测
}
```

### 溢出预测机制

在 `pmccntr_op_finish` 和 `pmevcntr_op_finish` 中：

```
1. 计算距离溢出还需要多少底层事件 (remaining)
2. 通过 ns_per_count(remaining) 转换为纳秒
3. timer_mod_anticipate_ns(pmu_timer, now + overflow_in)
```

`timer_mod_anticipate_ns` 只会将定时器提前（不会推迟），确保所有计数器的溢出都被覆盖。

### 计数器读取流程

```
Guest 读 PMCCNTR_EL0
    → pmccntr_read()               // :709-716
        → pmccntr_op_start(env)    // 快照：检测溢出，更新 c15_ccnt
        → ret = env->cp15.c15_ccnt // 返回当前值
        → pmccntr_op_finish(env)   // 重算 delta，预测溢出
        → return ret
```

---

## 11. KVM PMU 集成

### PMU 初始化

```c
// target/arm/kvm.c:1845-1859
void kvm_arm_pmu_init(ARMCPU *cpu) {
    struct kvm_device_attr attr = {
        .group = KVM_ARM_VCPU_PMU_V3_CTRL,
        .attr = KVM_ARM_VCPU_PMU_V3_INIT,
    };
    if (!cpu->has_pmu) return;
    kvm_arm_set_device_attr(cpu, &attr, "PMU");  // :1855
}
```

### 中断配置

```c
// :1861-1876
void kvm_arm_pmu_set_irq(ARMCPU *cpu, int irq) {
    struct kvm_device_attr attr = {
        .group = KVM_ARM_VCPU_PMU_V3_CTRL,
        .addr = (intptr_t)&irq,
        .attr = KVM_ARM_VCPU_PMU_V3_IRQ,
    };
    kvm_arm_set_device_attr(cpu, &attr, "PMU");  // :1872
}
```

### KVM vs TCG PMU 差异

| 方面 | TCG PMU | KVM PMU |
|------|---------|---------|
| 事件源 | pm_events[] 表（6 种事件） | 硬件 PMU（完整事件集） |
| CPU_CYCLES | `VIRTUAL_NS × 1GHz / 1e9` | 真实硬件周期 |
| INST_RETIRED | 仅 icount=precise | 硬件精确计数 |
| 溢出处理 | pmu_timer 预测 + 惰性检测 | 硬件中断 → KVM_EXIT |
| 寄存器同步 | 直接访问 CPUARMState | cpreg 迁移路径（write_list_to_kvmstate） |
| EL 过滤 | pmu_counter_enabled() 软件模拟 | 硬件原生过滤 |

---

## 12. 总结

### PMU 计数流程

```
┌───────────────────────────────────────────────────────┐
│  1. 定时器到期 / 寄存器访问 / EL 切换                   │
│     触发 pmu_op_start()                                │
│                                                       │
│  2. 对每个启用的计数器:                                 │
│     new_count = event.get_count() - delta              │
│     if (old & ~new & overflow_mask) → 设置 OVSR        │
│     counter = new_count                                │
│     delta = current_underlying_count                   │
│                                                       │
│  3. pmu_update_irq():                                  │
│     if (PMCR.E && PMINTEN & PMOVSR) → qemu_set_irq    │
│                                                       │
│  4. pmu_op_finish():                                   │
│     remaining = -counter                               │
│     overflow_in = ns_per_count(remaining)              │
│     timer_mod_anticipate_ns(pmu_timer, now + overflow) │
│     delta = underlying - counter（为下次 start 准备）   │
└───────────────────────────────────────────────────────┘
```

### 设计亮点

1. **惰性求值**：不在每个周期/指令递增计数器，而是在访问时通过底层计数源（虚拟时钟/icount）的差值计算。CPU 周期零开销。

2. **溢出预测**：`ns_per_count()` 将剩余计数转换为纳秒，`timer_mod_anticipate_ns()` 编程 QEMUTimer 在溢出时唤醒。只有 SW_INCR 无法预测（返回 -1）。

3. **EL 切换快照**：`pmu_pre_el_change()` / `pmu_post_el_change()` 在 EL 切换前后调用 op_start/finish，确保过滤条件变化时正确拆分计数。

4. **四层过滤**：全局使能 → 禁止位（HPMD/SPME/SCCD/HCCD）→ EL/安全过滤（P/U/NSK/NSU/NSH/M）→ 事件支持检查。层层递进，精确控制。

5. **HPMN 虚拟化**：MDCR_EL2.HPMN 将计数器分为 Guest 和 Hypervisor 两组，PMCR.N 在 EL0/EL1 读取时被替换为 HPMN。

6. **统一 op_start/op_finish 模式**：所有寄存器读写都遵循 `op_start → 操作 → op_finish` 模式。op_start 快照+溢出检测，op_finish 重算 delta+预测溢出。代码结构一致，易于理解。

---

**关键源文件**：
- `target/arm/cpregs-pmu.c` — **PMU 全部实现**（事件表、惰性求值、寄存器定义、过滤、中断）
- `target/arm/cpu.h` — CPUARMState 中的 PMU 状态字段
- `target/arm/internals.h` — pmu_num_counters/pmu_counter_mask 辅助函数
- `target/arm/kvm.c` — KVM PMU 初始化与中断配置
