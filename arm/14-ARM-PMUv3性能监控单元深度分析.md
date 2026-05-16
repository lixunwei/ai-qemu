# ARM PMUv3 性能监控单元深度分析

> 基于 QEMU 11.0.50 源码，分析 ARM PMUv3 性能监控单元的完整实现  
> 包括事件计数、周期计数器、溢出中断、访问控制、KVM 支持

---

## 目录

1. [概述](#1-概述)
2. [PMU 特性检测与版本](#2-pmu-特性检测与版本)
3. [CPUARMState 中的 PMU 状态](#3-cpuarmstate-中的-pmu-状态)
4. [PMU 事件定义框架](#4-pmu-事件定义框架)
5. [支持的事件列表](#5-支持的事件列表)
6. [CPU_CYCLES 事件实现](#6-cpu_cycles-事件实现)
7. [INST_RETIRED 事件实现](#7-inst_retired-事件实现)
8. [STALL 类事件](#8-stall-类事件)
9. [PMU 系统寄存器总览](#9-pmu-系统寄存器总览)
10. [PMCR_EL0 控制寄存器](#10-pmcr_el0-控制寄存器)
11. [pmcr_write() 实现](#11-pmcr_write-实现)
12. [pmcr_read() 与 HPMN](#12-pmcr_read-与-hpmn)
13. [PMCNTENSET/PMCNTENCLR 使能控制](#13-pmcntensetpmcntenclr-使能控制)
14. [PMCCNTR_EL0 周期计数器](#14-pmccntr_el0-周期计数器)
15. [pmccntr_op_start() 快照机制](#15-pmccntr_op_start-快照机制)
16. [pmccntr_op_finish() 溢出调度](#16-pmccntr_op_finish-溢出调度)
17. [pmevcntr_op_start/finish 事件计数器](#17-pmevcntr_op_startfinish-事件计数器)
18. [pmu_op_start/finish 批量操作](#18-pmu_op_startfinish-批量操作)
19. [PMEVCNTRn/PMEVTYPERn 事件计数器](#19-pmevcntrn pmevtypern-事件计数器)
20. [pmevtyper_write() 事件切换](#20-pmevtyper_write-事件切换)
21. [PMSWINC 软件递增](#21-pmswinc-软件递增)
22. [PMOVSR/PMOVSSET 溢出状态](#22-pmovsrpmovsset-溢出状态)
23. [PMINTENSET/PMINTENCLR 中断使能](#23-pmintensetpmintenclr-中断使能)
24. [pmu_update_irq() 中断更新](#24-pmu_update_irq-中断更新)
25. [PMU 中断连接](#25-pmu-中断连接)
26. [访问控制体系](#26-访问控制体系)
27. [PMUSERENR_EL0 用户态访问](#27-pmuserenr_el0-用户态访问)
28. [MDCR_EL2/EL3 陷阱控制](#28-mdcr_el2el3-陷阱控制)
29. [PMCCFILTR_EL0 周期计数过滤](#29-pmccfiltr_el0-周期计数过滤)
30. [64 位计数器 (PMUv3p5)](#30-64-位计数器-pmuv3p5)
31. [时钟分频 PMCR.D/LC](#31-时钟分频-pmcrdlc)
32. [PMU 初始化流程](#32-pmu-初始化流程)
33. [KVM PMU 支持](#33-kvm-pmu-支持)
34. [EL 状态变化处理](#34-el-状态变化处理)
35. [迁移支持](#35-迁移支持)
36. [完整 PMU 工作流程](#36-完整-pmu-工作流程)
37. [关键寄存器汇总表](#37-关键寄存器汇总表)
38. [总结](#38-总结)

---

## 1. 概述

ARM PMUv3 (Performance Monitors Extension v3) 是 ARM 架构的性能监控框架，提供硬件事件计数能力。在 QEMU 中，PMU 以**有限模拟**方式实现：

- **周期计数** (CPU_CYCLES)：映射到 QEMU 虚拟时钟
- **指令计数** (INST_RETIRED)：仅在精确 icount 模式下可用
- **SW_INCR**：软件递增事件，完全模拟
- **STALL 类事件**：始终返回 0（QEMU 无流水线概念）
- **溢出中断**：完整实现，连接到 GIC PPI

**关键源文件**：`target/arm/cpregs-pmu.c`（~1300 行，PMU 专用寄存器实现）

---

## 2. PMU 特性检测与版本

```c
// cpu-features.h:728-746
// PMU 版本从 ID_AA64DFR0_EL1.PMUVER (AArch64) 或
// ID_DFR0.PERFMON (AArch32) 字段读取

// cpu.h:2141
ARM_FEATURE_PMU  // PMU 功能标志
```

PMU 版本检测函数：
- `cpu_isar_feature(any_pmuv3p1, cpu)` — PMUv3.1 (ARMv8.1)
- `cpu_isar_feature(any_pmuv3p4, cpu)` — PMUv3.4 (ARMv8.4)
- `cpu_isar_feature(any_pmuv3p5, cpu)` — PMUv3.5 (ARMv8.5，支持 64 位计数器)

---

## 3. CPUARMState 中的 PMU 状态

```c
// cpu.h:444-449, 539-556
/* PMU 控制寄存器 */
uint32_t c9_pmcr;          // PMCR_EL0
uint32_t c9_pmcnten;       // PMCNTENSET/CLR (计数器使能位图)
uint32_t c9_pmovsr;        // PMOVSR (溢出状态位图)
uint32_t c9_pmuserenr;     // PMUSERENR_EL0
uint32_t c9_pmselr;        // PMSELR (选择寄存器)
uint32_t c9_pminten;       // PMINTENSET/CLR (中断使能位图)

/* 周期计数器 */
uint64_t c15_ccnt;          // PMCCNTR 当前值
uint64_t c15_ccnt_delta;    // 用于增量计算的基准值

/* 事件计数器 (最多 31 个) */
uint64_t c14_pmevcntr[31];       // PMEVCNTRn 值
uint64_t c14_pmevcntr_delta[31]; // 增量基准
uint64_t c14_pmevtyper[31];      // PMEVTYPERn 事件类型
uint64_t pmccfiltr_el0;          // PMCCFILTR 周期计数过滤
```

**设计要点**：使用 delta 机制避免每个时钟周期都更新计数器——只在访问时计算差值。

---

## 4. PMU 事件定义框架

```c
// cpregs-pmu.c:37-53
typedef struct pm_event {
    uint16_t number;       // PMEVTYPER.evtCount (16位事件编号)
    bool (*supported)(CPUARMState *);  // 此CPU是否支持该事件
    uint64_t (*get_count)(CPUARMState *);  // 获取当前底层计数
    int64_t (*ns_per_count)(uint64_t);     // 每count事件需要多少纳秒
} pm_event;
```

每个事件需要实现三个回调：
- `supported` — 运行时检查 CPU 是否支持此事件（用于生成 PMCEID0/1）
- `get_count` — 返回底层事件的累计计数
- `ns_per_count` — 用于预测下次溢出时间，调度定时器

---

## 5. 支持的事件列表

```c
// cpregs-pmu.c:137-170
static const pm_event pm_events[] = {
    { .number = 0x000, /* SW_INCR */
      .supported = event_always_supported,
      .get_count = swinc_get_count,       // 始终返回 0
      .ns_per_count = swinc_ns_per, },    // 返回 -1 (永不自动溢出)
    { .number = 0x008, /* INST_RETIRED */
      .supported = instructions_supported, // 仅 icount==PRECISE
      .get_count = instructions_get_count, // icount_get_raw()
      .ns_per_count = instructions_ns_per, },
    { .number = 0x011, /* CPU_CYCLES */
      .supported = event_always_supported,
      .get_count = cycles_get_count,
      .ns_per_count = cycles_ns_per, },
    { .number = 0x023, /* STALL_FRONTEND */ ... zero_event ... },
    { .number = 0x024, /* STALL_BACKEND */  ... zero_event ... },
    { .number = 0x03c, /* STALL */          ... zero_event ... },
};
```

| 事件号 | 名称 | QEMU 实现 | 条件 |
|--------|------|----------|------|
| 0x000 | SW_INCR | 软件递增，通过 PMSWINC 写入 | 始终支持 |
| 0x008 | INST_RETIRED | `icount_get_raw()` | 仅 `-icount` 精确模式 |
| 0x011 | CPU_CYCLES | 虚拟时钟/1GHz 映射 | 始终支持(system) |
| 0x023 | STALL_FRONTEND | 始终为 0 | PMUv3.1+ |
| 0x024 | STALL_BACKEND | 始终为 0 | PMUv3.1+ |
| 0x03c | STALL | 始终为 0 | PMUv3.4+ |

---

## 6. CPU_CYCLES 事件实现

```c
// cpregs-pmu.c:78-86
static uint64_t cycles_get_count(CPUARMState *env)
{
#ifndef CONFIG_USER_ONLY
    return muldiv64(qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL),
                   ARM_CPU_FREQ, NANOSECONDS_PER_SECOND);
#else
    return cpu_get_host_ticks();
#endif
}
```

```c
// cpregs-pmu.c:16
#define ARM_CPU_FREQ 1000000000 /* FIXME: 1 GHz, should be configurable */
```

**System 模式**：`cycles = 虚拟时钟纳秒 × 1GHz / 1e9 = 虚拟时钟纳秒`  
**User 模式**：使用主机 CPU 时间戳计数器  
**注意**：`ARM_CPU_FREQ` 硬编码为 1GHz，标记了 FIXME

---

## 7. INST_RETIRED 事件实现

```c
// cpregs-pmu.c:94-110
static bool instructions_supported(CPUARMState *env)
{
    return icount_enabled() == ICOUNT_PRECISE;  // 仅精确icount
}

static uint64_t instructions_get_count(CPUARMState *env)
{
    assert(icount_enabled() == ICOUNT_PRECISE);
    return (uint64_t)icount_get_raw();
}
```

只有使用 `-icount` 参数启动 QEMU 并设置精确模式时，指令计数才可用。这是因为 QEMU TCG 的翻译块执行模式下，普通运行无法精确追踪每条指令。

---

## 8. STALL 类事件

```c
// cpregs-pmu.c:125-135
static uint64_t zero_event_get_count(CPUARMState *env)
{
    return 0;  // QEMU 无流水线，永远不 stall
}

static int64_t zero_event_ns_per(uint64_t cycles)
{
    return -1;  // 永不溢出
}
```

STALL_FRONTEND/STALL_BACKEND/STALL 事件始终返回 0，因为 QEMU 没有流水线模拟。它们被注册以生成正确的 PMCEID 位，但计数值永远为 0。

---

## 9. PMU 系统寄存器总览

```c
// cpregs-pmu.c:1010-1190
static const ARMCPRegInfo v7_pm_reginfo[] = {
    // PMCNTENSET/CLR, PMOVSR/PMOVSCLR, PMSWINC
    // PMSELR, PMCCNTR, PMCCFILTR
    // PMXEVTYPER/PMXEVCNTR (间接访问)
    // PMUSERENR, PMINTENSET/CLR
    // PMOVSSET
    ...
};
```

PMU 寄存器分为三类访问权限：
- **(a)** 始终 PL1 才能访问：PMINTENSET/CLR
- **(b)** PL0 只读，PL1 读写：PMUSERENR
- **(c)** 受 PMUSERENR 控制：其他所有 PMU 寄存器

---

## 10. PMCR_EL0 控制寄存器

PMCR_EL0 是 PMU 的主控制寄存器：

| 位 | 名称 | 功能 |
|----|------|------|
| [0] | E | PMU 全局使能 |
| [1] | P | 事件计数器复位（写 1 清零所有 PMEVCNTRn）|
| [2] | C | 周期计数器复位（写 1 清零 PMCCNTR）|
| [3] | D | 时钟分频器（1=每 64 周期计数一次）|
| [4] | X | 导出使能（未实现）|
| [5] | DP | 禁用事件禁止时的周期计数 |
| [6] | LC | 长计数器使能（64 位 PMCCNTR）|
| [7] | LP | 长事件计数器使能 (PMUv3.5) |
| [15:11] | N | 事件计数器数量（只读）|

---

## 11. pmcr_write() 实现

```c
// cpregs-pmu.c:634-655
static void pmcr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                       uint64_t value)
{
    pmu_op_start(env);

    if (value & PMCRC) {
        env->cp15.c15_ccnt = 0;  // C 位：复位周期计数器
    }

    if (value & PMCRP) {
        for (i = 0; i < pmu_num_counters(env); i++) {
            env->cp15.c14_pmevcntr[i] = 0;  // P 位：复位事件计数器
        }
    }

    env->cp15.c9_pmcr &= ~PMCR_WRITABLE_MASK;
    env->cp15.c9_pmcr |= (value & PMCR_WRITABLE_MASK);

    pmu_op_finish(env);
}
```

**写入流程**：
1. `pmu_op_start()` — 将所有计数器快照到架构值
2. 处理 C/P 复位位
3. 更新可写位
4. `pmu_op_finish()` — 重新计算 delta 并调度溢出定时器

---

## 12. pmcr_read() 与 HPMN

```c
// cpregs-pmu.c:657-671
static uint64_t pmcr_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    uint64_t pmcr = env->cp15.c9_pmcr;

    // EL1/EL0 读取时，N 字段被 MDCR_EL2.HPMN 覆盖
    if (arm_current_el(env) <= 1 && arm_is_el2_enabled(env)) {
        pmcr &= ~PMCRN_MASK;
        pmcr |= (env->cp15.mdcr_el2 & MDCR_HPMN) << PMCRN_SHIFT;
    }

    return pmcr;
}
```

**HPMN 机制**：Hypervisor 可通过 MDCR_EL2.HPMN 限制 Guest 可见的计数器数量。EL1 读 PMCR.N 返回 HPMN 而非实际值，高编号计数器成为 Hypervisor 专用。

---

## 13. PMCNTENSET/PMCNTENCLR 使能控制

```c
// cpregs-pmu.c:1023-1049
// PMCNTENSET: 写入 1 的位使能对应计数器
// PMCNTENCLR: 写入 1 的位禁用对应计数器
// bit[31] = 周期计数器 PMCCNTR
// bit[0:30] = 事件计数器 PMEVCNTR0-30
```

`c9_pmcnten` 位图控制哪些计数器活跃。计数器只有在 PMCR.E=1 **且** PMCNTENSET 对应位=1 时才真正计数。

---

## 14. PMCCNTR_EL0 周期计数器

```c
// cpregs-pmu.c:1088-1095
{ .name = "PMCCNTR_EL0", ... .type = ARM_CP_IO,
  .fieldoffset = offsetof(CPUARMState, cp15.c15_ccnt),
  .readfn = pmccntr_read, .writefn = pmccntr_write, },
```

PMCCNTR 使用延迟更新 (lazy update) 机制：
- 不在每个时钟周期更新 `c15_ccnt`
- 读取时调用 `pmccntr_op_start()` 计算实际值
- 写入后调用 `pmccntr_op_finish()` 重建 delta

---

## 15. pmccntr_op_start() 快照机制

```c
// cpregs-pmu.c:479-500
static void pmccntr_op_start(CPUARMState *env)
{
    uint64_t cycles = cycles_get_count(env);

    if (pmu_counter_enabled(env, 31)) {
        uint64_t eff_cycles = cycles;
        if (pmccntr_clockdiv_enabled(env)) {
            eff_cycles /= 64;  // D 位分频
        }

        uint64_t new_pmccntr = eff_cycles - env->cp15.c15_ccnt_delta;

        // 检测溢出
        uint64_t overflow_mask = (c9_pmcr & PMCRLC) ? 1ull<<63 : 1ull<<31;
        if (env->cp15.c15_ccnt & ~new_pmccntr & overflow_mask) {
            env->cp15.c9_pmovsr |= (1ULL << 31);
            pmu_update_irq(env);
        }

        env->cp15.c15_ccnt = new_pmccntr;
    }
    env->cp15.c15_ccnt_delta = cycles;
}
```

**工作原理**：
1. 获取当前底层周期数
2. 如果计数器使能，计算 `new_ccnt = cycles - delta`
3. 检测高位翻转（溢出）：旧值最高位=1，新值最高位=0
4. 溢出时设置 PMOVSR bit[31] 并更新中断
5. 更新 `c15_ccnt` 和 `c15_ccnt_delta`

---

## 16. pmccntr_op_finish() 溢出调度

```c
// cpregs-pmu.c:508-536
static void pmccntr_op_finish(CPUARMState *env)
{
    if (pmu_counter_enabled(env, 31)) {
        uint64_t remaining_cycles = -env->cp15.c15_ccnt;
        if (!(c9_pmcr & PMCRLC))
            remaining_cycles = (uint32_t)remaining_cycles;

        int64_t overflow_in = cycles_ns_per(remaining_cycles);

        if (overflow_in > 0) {
            int64_t overflow_at;
            if (!sadd64_overflow(qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL),
                                 overflow_in, &overflow_at)) {
                timer_mod_anticipate_ns(cpu->pmu_timer, overflow_at);
            }
        }

        // 恢复 delta
        uint64_t prev_cycles = env->cp15.c15_ccnt_delta;
        if (pmccntr_clockdiv_enabled(env))
            prev_cycles /= 64;
        env->cp15.c15_ccnt_delta = prev_cycles - env->cp15.c15_ccnt;
    }
}
```

**溢出预测**：
1. 计算距溢出还需多少周期（取反 = 到 0 的距离）
2. 转换为纳秒数
3. 使用 `timer_mod_anticipate_ns()` 设置 PMU 定时器在溢出时触发

---

## 17. pmevcntr_op_start/finish 事件计数器

```c
// cpregs-pmu.c:538-590
static void pmevcntr_op_start(CPUARMState *env, uint8_t counter)
{
    uint16_t event = env->cp15.c14_pmevtyper[counter] & PMXEVTYPER_EVTCOUNT;
    uint64_t count = 0;
    if (event_supported(event)) {
        count = pm_events[event_idx].get_count(env);
    }

    if (pmu_counter_enabled(env, counter)) {
        uint64_t new = count - env->cp15.c14_pmevcntr_delta[counter];
        // 溢出检测（同 PMCCNTR 逻辑）
        if (overflow detected) {
            env->cp15.c9_pmovsr |= (1 << counter);
            pmu_update_irq(env);
        }
        env->cp15.c14_pmevcntr[counter] = new;
    }
    env->cp15.c14_pmevcntr_delta[counter] = count;
}
```

与 PMCCNTR 机制完全对称，但每个计数器根据 PMEVTYPERn 的事件类型选择不同的 `get_count()` 函数。

---

## 18. pmu_op_start/finish 批量操作

```c
// cpregs-pmu.c:592-608
void pmu_op_start(CPUARMState *env)
{
    pmccntr_op_start(env);
    for (i = 0; i < pmu_num_counters(env); i++) {
        pmevcntr_op_start(env, i);
    }
}

void pmu_op_finish(CPUARMState *env)
{
    pmccntr_op_finish(env);
    for (i = 0; i < pmu_num_counters(env); i++) {
        pmevcntr_op_finish(env, i);
    }
}
```

所有需要访问 PMU 计数器的操作（读写寄存器、PMCR 写入、EL 切换）都包裹在 `pmu_op_start/finish` 对中，确保计数器值是最新的。

---

## 19. PMEVCNTRn/PMEVTYPERn 事件计数器

- 最多 31 个事件计数器（实际数量由 CPU 模型决定，存储在 PMCR.N）
- **PMEVTYPERn**：配置事件类型（低 16 位为事件编号）+ 过滤位
- **PMEVCNTRn**：事件计数值（32 位或 64 位）
- 可通过 PMSELR + PMXEVTYPER/PMXEVCNTR 间接访问

---

## 20. pmevtyper_write() 事件切换

```c
// cpregs-pmu.c:804-838
static void pmevtyper_write(CPUARMState *env, const ARMCPRegInfo *ri,
                            uint64_t value, const uint8_t counter)
{
    pmevcntr_op_start(env, counter);

    uint16_t old_event = env->cp15.c14_pmevtyper[counter] & PMXEVTYPER_EVTCOUNT;
    uint16_t new_event = value & PMXEVTYPER_EVTCOUNT;
    if (old_event != new_event) {
        // 事件类型改变：重置 delta 基准到新事件的当前计数
        uint64_t count = pm_events[new_event_idx].get_count(env);
        env->cp15.c14_pmevcntr_delta[counter] = count;
    }

    env->cp15.c14_pmevtyper[counter] = value & PMXEVTYPER_MASK;
    pmevcntr_op_finish(env, counter);
}
```

切换事件类型时需要重置 delta 基准，否则新旧事件的计数值不可比会产生错误的溢出检测。

---

## 21. PMSWINC 软件递增

```c
// cpregs-pmu.c:673-700
static void pmswinc_write(CPUARMState *env, const ARMCPRegInfo *ri,
                          uint64_t value)
{
    for (i = 0; i < pmu_num_counters(env); i++) {
        if ((value & (1 << i)) &&           // 写入位设置
            pmu_counter_enabled(env, i) &&   // 计数器使能
            (pmevtyper[i] & EVTCOUNT) == 0x0) { // 事件类型是 SW_INCR
            pmevcntr_op_start(env, i);
            new_pmswinc = env->cp15.c14_pmevcntr[i] + 1;
            // 检测溢出
            if (overflow) {
                env->cp15.c9_pmovsr |= (1 << i);
                pmu_update_irq(env);
            }
            env->cp15.c14_pmevcntr[i] = new_pmswinc;
            pmevcntr_op_finish(env, i);
        }
    }
}
```

PMSWINC 是唯一可以无硬件事件触发的递增方式，每次写入 PMSWINC 时，对应位为 1 的 SW_INCR 类型计数器加 1。

---

## 22. PMOVSR/PMOVSSET 溢出状态

```c
// cpregs-pmu.c:788-802
static void pmovsr_write(CPUARMState *env, ...)
{
    value &= pmu_counter_mask(env);
    env->cp15.c9_pmovsr &= ~value;  // 写 1 清除
    pmu_update_irq(env);
}

static void pmovsset_write(CPUARMState *env, ...)
{
    value &= pmu_counter_mask(env);
    env->cp15.c9_pmovsr |= value;   // 写 1 设置
    pmu_update_irq(env);
}
```

- **PMOVSR** (PMOVSCLR_EL0)：写 1 清除溢出位
- **PMOVSSET**：写 1 设置溢出位（可用于测试中断）
- 两者都会触发 `pmu_update_irq()` 重新评估中断状态

---

## 23. PMINTENSET/PMINTENCLR 中断使能

```c
// cpregs-pmu.c:993-1008
static void pmintenset_write(CPUARMState *env, ...)
{
    value &= pmu_counter_mask(env);
    env->cp15.c9_pminten |= value;
    pmu_update_irq(env);
}

static void pmintenclr_write(CPUARMState *env, ...)
{
    value &= pmu_counter_mask(env);
    env->cp15.c9_pminten &= ~value;
    pmu_update_irq(env);
}
```

`c9_pminten` 位图控制哪些计数器的溢出可以产生中断。

---

## 24. pmu_update_irq() 中断更新

```c
// cpregs-pmu.c:429-434
static void pmu_update_irq(CPUARMState *env)
{
    ARMCPU *cpu = env_archcpu(env);
    qemu_set_irq(cpu->pmu_interrupt,
                  (env->cp15.c9_pmcr & PMCRE) &&
                  (env->cp15.c9_pminten & env->cp15.c9_pmovsr));
}
```

**中断条件**：`PMCR.E=1 && (PMINTENSET & PMOVSR) != 0`

三个条件缺一不可：
1. PMU 全局使能 (PMCR.E)
2. 对应计数器的中断使能 (PMINTENSET)
3. 对应计数器发生溢出 (PMOVSR)

---

## 25. PMU 中断连接

PMU 中断通过 `cpu->pmu_interrupt` qemu_irq 输出，连接到 GIC PPI。在 virt 机器中，PMU 中断通常连接到 PPI 23 (中断 ID 39)。

KVM 模式下，PMU 中断由 `kvm_arm_pmu_set_irq()` 配置。

---

## 26. 访问控制体系

```c
// cpregs-pmu.c:22-35
static CPAccessResult access_tpm(CPUARMState *env, ...)
{
    int el = arm_current_el(env);
    if (el < 2 && (mdcr_el2 & MDCR_TPM))
        return CP_ACCESS_TRAP_EL2;
    if (el < 3 && (mdcr_el3 & MDCR_TPM))
        return CP_ACCESS_TRAP_EL3;
    return CP_ACCESS_OK;
}
```

PMU 访问控制有三层：

| 层级 | 控制位 | 功能 |
|------|--------|------|
| EL0→EL1 | PMUSERENR_EL0 | 用户态访问控制 |
| EL1→EL2 | MDCR_EL2.TPM/TPMCR | Hypervisor 陷阱 |
| EL2→EL3 | MDCR_EL3.TPM | 安全监控陷阱 |

---

## 27. PMUSERENR_EL0 用户态访问

```c
// cpregs-pmu.c:231-258
static CPAccessResult do_pmreg_access(CPUARMState *env, bool is_pmcr)
{
    if (el == 0 && !(env->cp15.c9_pmuserenr & 1))
        return CP_ACCESS_TRAP_EL1;
    if (el < 2 && (mdcr_el2 & MDCR_TPM))
        return CP_ACCESS_TRAP_EL2;
    ...
}
```

**PMUSERENR 位定义**：
| 位 | 名称 | 功能 |
|----|------|------|
| [0] | EN | 通用 PMU 访问使能 |
| [1] | SW | SW_INCR 写入使能 |
| [2] | CR | 周期计数器读取使能 |
| [3] | ER | 事件计数器读取使能 |

精细化访问控制函数：
- `pmreg_access_swinc()` — 检查 bit[1] (SW)
- `pmreg_access_ccntr()` — 检查 bit[2] (CR)
- `pmreg_access_xevcntr()` — 检查 bit[3] (ER)
- `pmreg_access_selr()` — 检查 bit[3] (ER)

---

## 28. MDCR_EL2/EL3 陷阱控制

```c
// helper.c:691-697
// MDCR_EL2 PMU 相关位:
// TPM  — 陷阱 PMU 寄存器到 EL2
// TPMCR — 陷阱 PMCR 到 EL2
// HPMN — Hypervisor 性能监控数（限制 Guest 可见计数器）

// MDCR_EL3 PMU 相关位:
// TPM  — 陷阱 PMU 寄存器到 EL3
```

**MDCR_EL2 写入时需要包裹 pmu_op_start/finish**（helper.c:3304-3339），因为改变 HPMN 或使能位会影响计数器行为。

---

## 29. PMCCFILTR_EL0 周期计数过滤

PMCCFILTR 控制周期计数器在哪些异常级别和安全状态下计数：

| 位 | 名称 | 功能 |
|----|------|------|
| [31] | P | 不在 EL1 计数 |
| [30] | U | 不在 EL0 计数 |
| [29] | NSK | 不在 NonSecure EL1 计数 |
| [28] | NSU | 不在 NonSecure EL0 计数 |
| [27] | NSH | 不在 EL2 计数 |
| [26] | M | 不在安全 EL3 计数 |

QEMU 使用 `pmu_counter_enabled()` 函数检查当前 EL/安全状态是否被过滤。

---

## 30. 64 位计数器 (PMUv3p5)

```c
// cpregs-pmu.c:447-471
static bool pmevcntr_is_64_bit(CPUARMState *env, int counter)
{
    if (!cpu_isar_feature(any_pmuv3p5, env_archcpu(env)))
        return false;

    if (arm_feature(env, ARM_FEATURE_EL2)) {
        bool hlp = env->cp15.mdcr_el2 & MDCR_HLP;
        int hpmn = env->cp15.mdcr_el2 & MDCR_HPMN;
        if (counter >= hpmn)
            return hlp;  // Hypervisor 计数器宽度
    }
    return env->cp15.c9_pmcr & PMCRLP;  // Guest 计数器宽度
}
```

PMUv3.5 允许事件计数器使用 64 位宽度（默认 32 位）：
- `PMCR.LP` — Guest 计数器 64 位使能
- `MDCR_EL2.HLP` — Hypervisor 计数器 64 位使能
- 溢出检测掩码从 `1<<31` 变为 `1<<63`

---

## 31. 时钟分频 PMCR.D/LC

```c
// cpregs-pmu.c:436-445
static bool pmccntr_clockdiv_enabled(CPUARMState *env)
{
    // PMCR.D=1 且 PMCR.LC=0 时，每 64 周期计数一次
    return (env->cp15.c9_pmcr & (PMCRD | PMCRLC)) == PMCRD;
}
```

- **PMCR.D=1**：PMCCNTR 每 64 个时钟周期递增一次
- **PMCR.LC=1**：使能 64 位长计数器模式，此时 D 位无效
- D 和 LC 同时为 1 时，LC 优先（64 位模式，无分频）

---

## 32. PMU 初始化流程

```c
// cpu.c:2118-2138
// arm_cpu_realizefn:
if (PMU disabled) {
    // 清除 PMUVER/PERFMON 字段
} else {
    pmu_init(cpu);  // 初始化 PMU 定时器和中断线
}
```

`pmu_init()` 创建：
- `cpu->pmu_timer` — QEMUTimer，用于溢出中断调度
- `cpu->pmu_interrupt` — qemu_irq，连接到 GIC PPI
- 计算 `pmceid0/pmceid1`（支持的事件位图）

---

## 33. KVM PMU 支持

```c
// kvm.c
// KVM_CAP_ARM_PMU_V3 — 检查 KVM 是否支持 PMU
// KVM_ARM_VCPU_PMU_V3 — VCPU 初始化请求 PMU
// kvm_arm_pmu_init() — 初始化 KVM PMU
// kvm_arm_pmu_set_irq() — 设置 PMU 中断号
```

KVM 模式下：
- PMU 直接由宿主硬件提供，性能事件真实
- QEMU 不参与计数，只配置 KVM VCPU 的 PMU 参数
- PMU 中断通过 KVM 内核 GIC 直接注入

---

## 34. EL 状态变化处理

```c
// cpregs-pmu.c:610+
void pmu_pre_el_change(ARMCPU *cpu, void *ignored)
```

EL 切换时需要 `pmu_op_start/finish`，因为：
- 计数过滤 (PMCCFILTR/PMEVTYPER) 基于当前 EL
- 切换 EL 可能改变计数器的使能状态
- 需要在切换前快照当前值，切换后重建 delta

---

## 35. 迁移支持

PMU 寄存器是 CPUARMState 的一部分，随 CPU 状态一起迁移。`pmevtyper_rawwrite()` 特别处理迁移场景：

```c
// cpregs-pmu.c:863-884
static void pmevtyper_rawwrite(CPUARMState *env, ...)
{
    // 迁移加载时，需要重置 delta 到新事件类型的当前计数
    // 因为 pmu_op_start 已经用旧事件类型设置了 delta
    env->cp15.c14_pmevcntr_delta[counter] =
        pm_events[event_idx].get_count(env);
}
```

---

## 36. 完整 PMU 工作流程

```
1. 初始化:
   arm_cpu_realizefn → pmu_init()
   → 创建 pmu_timer + pmu_interrupt
   → 计算 pmceid0/pmceid1

2. Guest 配置:
   PMCR.E = 1                      // 全局使能
   PMEVTYPER0 = 0x011              // CPU_CYCLES
   PMCNTENSET = (1 << 0) | (1 << 31)  // 使能计数器0和PMCCNTR
   PMINTENSET = (1 << 0) | (1 << 31)  // 使能溢出中断

3. 计数进行:
   → 底层周期数通过虚拟时钟累积
   → pmccntr_op_finish 调度 pmu_timer 在预测溢出时间

4. 溢出触发:
   pmu_timer 到期回调
   → pmu_op_start()
     → pmccntr_op_start() 检测 c15_ccnt 高位翻转
     → c9_pmovsr |= (1 << 31)
     → pmu_update_irq()
       → qemu_set_irq(pmu_interrupt, 1)  // 拉高中断线

5. GIC 接收 PPI:
   → 正常中断注入流程
   → Guest ISR 读 PMOVSR 识别溢出源
   → 写 PMOVSR 清除溢出位
   → pmu_update_irq() 拉低中断线
```

---

## 37. 关键寄存器汇总表

| 寄存器 | 字段 | 访问 | 功能 |
|--------|------|------|------|
| PMCR_EL0 | c9_pmcr | PL0_RW(受控) | PMU 主控制 |
| PMCNTENSET_EL0 | c9_pmcnten | PL0_RW(受控) | 计数器使能 |
| PMCNTENCLR_EL0 | c9_pmcnten | PL0_RW(受控) | 计数器禁用 |
| PMOVSR/PMOVSCLR_EL0 | c9_pmovsr | PL0_RW(受控) | 溢出状态(清除) |
| PMOVSSET_EL0 | c9_pmovsr | PL0_RW(受控) | 溢出状态(设置) |
| PMSELR_EL0 | c9_pmselr | PL0_RW(受控) | 间接访问选择 |
| PMCCNTR_EL0 | c15_ccnt | PL0_RW(受控) | 周期计数器 |
| PMCCFILTR_EL0 | pmccfiltr_el0 | PL0_RW(受控) | 周期计数过滤 |
| PMEVCNTRn_EL0 | c14_pmevcntr[] | PL0_RW(受控) | 事件计数器 |
| PMEVTYPERn_EL0 | c14_pmevtyper[] | PL0_RW(受控) | 事件类型配置 |
| PMSWINC_EL0 | (无状态) | PL0_W(受控) | 软件递增 |
| PMUSERENR_EL0 | c9_pmuserenr | PL0_R/PL1_RW | 用户态访问控制 |
| PMINTENSET_EL1 | c9_pminten | PL1_RW | 中断使能 |
| PMINTENCLR_EL1 | c9_pminten | PL1_RW | 中断禁用 |

---

## 38. 总结

QEMU ARM PMU 实现的关键设计特点：

1. **延迟计算**：计数器不在每周期更新，通过 delta 机制在访问时计算
2. **有限事件支持**：仅 CPU_CYCLES、INST_RETIRED、SW_INCR 真正计数
3. **完整溢出中断**：支持预测性定时器调度，准确触发溢出中断
4. **三层访问控制**：PMUSERENR → MDCR_EL2.TPM → MDCR_EL3.TPM
5. **HPMN 分区**：Hypervisor 可限制 Guest 可见计数器并保留专用计数器
6. **迁移安全**：rawwrite 处理事件类型切换时的 delta 重置
7. **KVM 直通**：KVM 模式使用硬件 PMU，QEMU 仅做配置
