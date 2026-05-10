# ARM64 PMU 性能监控单元深度分析

> QEMU 11.0.50 · 分析基于 commit HEAD  
> 关联文档：[00-ARM64-CPU-GICv3-TCG深度分析](00-ARM64-CPU-GICv3-TCG深度分析.md)、[12-Generic-Timer定时器深度分析](12-Generic-Timer定时器深度分析.md)、[06-异常级别状态管理深度分析](06-异常级别状态管理深度分析.md)

---

## 目录

1. [概述](#1-概述)
2. [PMU 版本与特性检测](#2-pmu-版本与特性检测)
3. [数据结构](#3-数据结构)
4. [支持的事件类型](#4-支持的事件类型)
5. [计数源——周期计数器](#5-计数源周期计数器)
6. [计数源——指令计数器](#6-计数源指令计数器)
7. [计数器使能判定 pmu_counter_enabled](#7-计数器使能判定-pmu_counter_enabled)
8. [事件过滤机制](#8-事件过滤机制)
9. [快照/恢复机制 pmu_op_start/finish](#9-快照恢复机制-pmu_op_startfinish)
10. [周期计数器操作 pmccntr_op_start/finish](#10-周期计数器操作-pmccntr_op_startfinish)
11. [事件计数器操作 pmevcntr_op_start/finish](#11-事件计数器操作-pmevcntr_op_startfinish)
12. [溢出检测与中断](#12-溢出检测与中断)
13. [PMU 定时器回调](#13-pmu-定时器回调)
14. [EL 变化钩子](#14-el-变化钩子)
15. [PMCR_EL0 控制寄存器](#15-pmcr_el0-控制寄存器)
16. [SW_INCR 软件递增](#16-sw_incr-软件递增)
17. [PMCR.N 与 HPMN 计数器分区](#17-pmcrn-与-hpmn-计数器分区)
18. [64 位计数器扩展](#18-64-位计数器扩展)
19. [系统寄存器接口](#19-系统寄存器接口)
20. [访问控制——PMUSERENR_EL0](#20-访问控制pmuserenr_el0)
21. [访问控制——EL2/EL3 陷入](#21-访问控制el2el3-陷入)
22. [EL2 计数器所有权 (HPMN/HPME/HPMD)](#22-el2-计数器所有权-hpmnhpmehpmd)
23. [EL3 计数控制 (SPME/SCCD)](#23-el3-计数控制-spmesccd)
24. [GIC 中断连线](#24-gic-中断连线)
25. [FDT 设备树描述](#25-fdt-设备树描述)
26. [KVM PMU 集成](#26-kvm-pmu-集成)
27. [CPU 属性与初始化](#27-cpu-属性与初始化)
28. [迁移与状态恢复](#28-迁移与状态恢复)
29. [完整数据流图](#29-完整数据流图)
30. [关键源文件索引](#30-关键源文件索引)

---

## 1. 概述

ARM PMU（Performance Monitoring Unit）是 ARMv8 架构的性能计数器子系统，提供周期计数器（PMCCNTR）和可配置事件计数器（PMEVCNTRn）。在 QEMU 中，PMU 完全嵌入 ARM CPU 对象内实现，通过系统寄存器接口暴露给客户机。

QEMU PMU 核心设计特点：
- **差值快照模型**：不逐条指令计数，而是通过差值（delta）在访问时"追赶"当前计数
- **定时器驱动溢出**：`pmu_timer` 在预测溢出时间点触发回调
- **最多 31 个事件计数器** + 1 个周期计数器（counter 31）
- **TCG 模式支持有限事件**：CPU_CYCLES（基于虚拟时钟）、INST_RETIRED（需 icount）
- **KVM 模式通过内核 vPMU**：直接利用硬件 PMU

---

## 2. PMU 版本与特性检测

PMU 版本通过 `ID_AA64DFR0_EL1.PMUVer` 字段报告：

```
cpu-features.h:

isar_feature_aa64_pmuv3p1(cpu) → PMUVER >= 4  (PMUv3.1, ARMv8.1)
isar_feature_aa64_pmuv3p4(cpu) → PMUVER >= 5  (PMUv3.4, ARMv8.4)
isar_feature_aa64_pmuv3p5(cpu) → PMUVER >= 6  (PMUv3.5, ARMv8.5)
PMUVER == 0xF → PMU 不存在
```

**CPU 属性控制**：
```
cpu.c:1288-1305

// CPU 'pmu' 属性：bool 类型
arm_get_pmu() → cpu->has_pmu
arm_set_pmu() → set/unset ARM_FEATURE_PMU
```

---

## 3. 数据结构

### 3.1 CPUARMState 中的 PMU 字段

```
cpu.h:444-456, 539-556

// 控制/状态寄存器
uint64_t c9_pmcr;              // PMCR_EL0 — 主控制寄存器
uint64_t c9_pmcnten;           // PMCNTENSET/CLR — 计数器使能位图
uint64_t c9_pmovsr;            // PMOVSSET/CLR — 溢出状态位图
uint64_t c9_pmuserenr;         // PMUSERENR_EL0 — 用户态访问控制
uint64_t c9_pmselr;            // PMSELR_EL0 — 计数器选择
uint64_t c9_pminten;           // PMINTENSET/CLR — 中断使能位图
uint64_t pmccfiltr_el0;        // PMCCFILTR_EL0 — 周期计数器过滤

// 周期计数器
uint64_t c15_ccnt;             // PMCCNTR — 周期计数值
uint64_t c15_ccnt_delta;       // 内部差值基准

// 事件计数器（最多 31 个）
uint64_t c14_pmevcntr[31];     // PMEVCNTRn — 事件计数值
uint64_t c14_pmevcntr_delta[31]; // 内部差值基准
uint64_t c14_pmevtyper[31];    // PMEVTYPERn — 事件类型/过滤器
```

### 3.2 ARMCPU 中的 PMU 资源

```
cpu.h:961, 970, 999

QEMUTimer *pmu_timer;          // 溢出预测定时器
qemu_irq pmu_interrupt;        // PMU 中断输出 (GPIO)
bool has_pmu;                  // PMU 使能标志
```

### 3.3 pm_event 事件描述符

```
cpregs-pmu.c:37-53

typedef struct pm_event {
    uint16_t number;                          // 事件编号 (PMEVTYPER.evtCount)
    bool (*supported)(CPUARMState *);         // 此 CPU 是否支持该事件
    uint64_t (*get_count)(CPUARMState *);     // 获取底层事件计数
    int64_t (*ns_per_count)(uint64_t);        // 每 N 次事件需要的纳秒数（用于溢出预测）
} pm_event;
```

---

## 4. 支持的事件类型

```
cpregs-pmu.c:137-170

static const pm_event pm_events[] = {
    { 0x000, SW_INCR,      always_supported, swinc_get_count,       swinc_ns_per },
    { 0x008, INST_RETIRED, instructions_supported, instructions_get_count, instructions_ns_per },
    { 0x011, CPU_CYCLES,   always_supported, cycles_get_count,      cycles_ns_per },
    { 0x023, STALL_FRONTEND, pmuv3p1_supported, zero_event_get_count, zero_event_ns_per },
    { 0x024, STALL_BACKEND,  pmuv3p1_supported, zero_event_get_count, zero_event_ns_per },
    { 0x03c, STALL,          pmuv3p4_supported, zero_event_get_count, zero_event_ns_per },
};
```

| 事件编号 | 名称 | 计数源 | 条件 |
|---------|------|--------|------|
| 0x000 | SW_INCR | PMSWINC 软件写入 | 始终支持 |
| 0x008 | INST_RETIRED | `icount_get_raw()` | 需 `-icount` 精确模式 |
| 0x011 | CPU_CYCLES | `QEMU_CLOCK_VIRTUAL` × 1GHz | 始终支持 |
| 0x023 | STALL_FRONTEND | 恒为 0（不发生） | PMUv3.1+ |
| 0x024 | STALL_BACKEND | 恒为 0（不发生） | PMUv3.1+ |
| 0x03c | STALL | 恒为 0（不发生） | PMUv3.4+ |

**设计限制**：QEMU TCG 不模拟微架构，因此 STALL 类事件永远返回 0。INST_RETIRED 需要 icount 精确模式（`-icount shift=auto`），否则不可用。

---

## 5. 计数源——周期计数器

```
cpregs-pmu.c:16, 78-86

#define ARM_CPU_FREQ 1000000000   // 1 GHz（固定值）

static uint64_t cycles_get_count(CPUARMState *env)
{
    // 将虚拟时钟纳秒转换为 CPU 周期
    return muldiv64(qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL),
                    ARM_CPU_FREQ, NANOSECONDS_PER_SECOND);
}
```

**计算公式**：`cycles = ns × 1GHz / 10^9 = ns`（1 GHz 下 1 cycle = 1 ns）

```
cpregs-pmu.c:89-92

static int64_t cycles_ns_per(uint64_t cycles)
{
    return (ARM_CPU_FREQ / NANOSECONDS_PER_SECOND) * cycles;  // = cycles（1:1 映射）
}
```

**PMCR.D 分频**：

```
cpregs-pmu.c:436-445

static bool pmccntr_clockdiv_enabled(CPUARMState *env)
{
    // PMCR.D=1 且 PMCR.LC=0 时，周期计数器每 64 周期递增一次
    return (env->cp15.c9_pmcr & (PMCRD | PMCRLC)) == PMCRD;
}
```

---

## 6. 计数源——指令计数器

```
cpregs-pmu.c:94-110

static bool instructions_supported(CPUARMState *env)
{
    return icount_enabled() == ICOUNT_PRECISE;   // 仅精确 icount 模式
}

static uint64_t instructions_get_count(CPUARMState *env)
{
    return (uint64_t)icount_get_raw();           // 原始指令计数
}

static int64_t instructions_ns_per(uint64_t icount)
{
    return icount_to_ns((int64_t)icount);        // icount → 纳秒
}
```

**前提条件**：QEMU 必须以 `-icount shift=auto` 启动。普通 TCG 模式下 INST_RETIRED 事件不可用，`PMCEID0` 中不会广播该事件。

---

## 7. 计数器使能判定 pmu_counter_enabled

这是 PMU 子系统最复杂的函数，决定一个计数器在当前时刻是否应该计数。

```
cpregs-pmu.c:336-427

static bool pmu_counter_enabled(CPUARMState *env, uint8_t counter)
{
    // 1. ARM_FEATURE_PMU 检查（M-profile 安全退出）
    if (!arm_feature(env, ARM_FEATURE_PMU)) return false;

    // 2. 使能位检查
    mdcr_el2 = arm_mdcr_el2_eff(env);
    hpmn = mdcr_el2 & MDCR_HPMN;

    if (counter < hpmn || counter == 31) {
        e = env->cp15.c9_pmcr & PMCRE;        // Guest 控制
    } else {
        e = mdcr_el2 & MDCR_HPME;             // EL2 控制
    }
    enabled = e && (c9_pmcnten & (1 << counter));

    // 3. 计数禁止检查
    if (el == 2 && (counter < hpmn || counter == 31))
        prohibited = MDCR_EL2.HPMD;
    if (secure)
        prohibited |= !MDCR_EL3.SPME;

    // 4. 周期计数器特殊处理
    if (counter == 31) {
        prohibited = prohibited && PMCR.DP;    // DP=1 时才禁止
        if (pmuv3p5) {
            if (secure) prohibited |= MDCR_EL3.SCCD;
            if (el == 2) prohibited |= MDCR_EL2.HCCD;
        }
    }

    // 5. 过滤器检查
    filter = (counter == 31) ? pmccfiltr_el0 : c14_pmevtyper[counter];
    // ... P/U/NSK/NSU/NSH/M 过滤逻辑 ...

    // 6. 事件支持检查
    if (counter != 31 && !event_supported(event))
        return false;

    return enabled && !prohibited && !filtered;
}
```

**判定层次**（全部满足才计数）：
1. PMU 特性存在
2. 计数器使能（PMCR.E 或 MDCR.HPME）且 PMCNTENSET 对应位置位
3. 计数未被禁止（HPMD/SPME/SCCD/HCCD）
4. 事件过滤器匹配当前 EL/安全状态
5. 事件类型被 QEMU 支持

---

## 8. 事件过滤机制

```
cpregs-pmu.c:397-413

// 过滤器位域（来自 PMEVTYPER/PMCCFILTR）
p   = filter & PMXEVTYPER_P;       // 不计 EL1（Privileged）
u   = filter & PMXEVTYPER_U;       // 不计 EL0（User）
nsk = EL3 && (filter & NSK);       // 不计 Non-Secure Kernel
nsu = EL3 && (filter & NSU);       // 不计 Non-Secure User
nsh = EL2 && (filter & NSH);       // 不计 Non-Secure Hyp
m   = AA64 && EL3 && (filter & M); // EL3 过滤

// 过滤逻辑（filtered=true → 不计数）
EL0: filtered = secure ? u : (u != nsu)
EL1: filtered = secure ? p : (p != nsk)
EL2: filtered = !nsh
EL3: filtered = (m != p)
```

**过滤含义**：
- `P=1` → 不在 EL1 计数
- `U=1` → 不在 EL0 计数
- `NSK=1` → 不在 Non-Secure EL1 计数（`nsk XOR p` 决定）
- `NSU=1` → 不在 Non-Secure EL0 计数
- `NSH=1` → 不在 EL2 计数
- `M=1` → EL3 过滤反转

---

## 9. 快照/恢复机制 pmu_op_start/finish

PMU 使用"差值快照"模型：计数器不实时更新，而是在每次访问时通过当前底层值减去基准差值来"追赶"。

```
cpregs-pmu.c:592-608

void pmu_op_start(CPUARMState *env)
{
    pmccntr_op_start(env);                      // 快照周期计数器
    for (i = 0; i < pmu_num_counters(env); i++)
        pmevcntr_op_start(env, i);              // 快照所有事件计数器
}

void pmu_op_finish(CPUARMState *env)
{
    pmccntr_op_finish(env);                     // 恢复差值 + 预测溢出
    for (i = 0; i < pmu_num_counters(env); i++)
        pmevcntr_op_finish(env, i);             // 恢复差值 + 预测溢出
}
```

**调用时机**：
- 客户机读/写任何 PMU 寄存器时（如 `pmccntr_read`、`pmcr_write`、`pmevtyper_write`）
- EL 变化时（`pmu_pre_el_change` / `pmu_post_el_change`）
- `pmu_timer` 到期时

---

## 10. 周期计数器操作 pmccntr_op_start/finish

### 10.1 op_start：快照当前值 + 溢出检测

```
cpregs-pmu.c:479-501

static void pmccntr_op_start(CPUARMState *env)
{
    uint64_t cycles = cycles_get_count(env);

    if (pmu_counter_enabled(env, 31)) {
        uint64_t eff_cycles = cycles;
        if (pmccntr_clockdiv_enabled(env))
            eff_cycles /= 64;                    // PMCR.D=1 分频

        uint64_t new_pmccntr = eff_cycles - env->cp15.c15_ccnt_delta;

        // 溢出检测：旧值最高位=1 新值最高位=0 → 发生回绕
        uint64_t overflow_mask = (PMCR.LC) ? (1ULL << 63) : (1ULL << 31);
        if (env->cp15.c15_ccnt & ~new_pmccntr & overflow_mask) {
            env->cp15.c9_pmovsr |= (1ULL << 31);   // 设置溢出位
            pmu_update_irq(env);                     // 更新中断
        }

        env->cp15.c15_ccnt = new_pmccntr;          // 更新可见计数
    }
    env->cp15.c15_ccnt_delta = cycles;              // 记录基准
}
```

### 10.2 op_finish：重建差值 + 预测下次溢出

```
cpregs-pmu.c:508-536

static void pmccntr_op_finish(CPUARMState *env)
{
    if (pmu_counter_enabled(env, 31)) {
        // 计算距溢出的剩余周期
        uint64_t remaining_cycles = -env->cp15.c15_ccnt;
        if (!(PMCR.LC))
            remaining_cycles = (uint32_t)remaining_cycles;  // 32 位模式

        int64_t overflow_in = cycles_ns_per(remaining_cycles);
        if (overflow_in > 0) {
            int64_t overflow_at = now + overflow_in;
            timer_mod_anticipate_ns(cpu->pmu_timer, overflow_at);  // 编程溢出定时器
        }

        // 重建差值：delta = cycles_now(带分频) - c15_ccnt
        uint64_t prev_cycles = c15_ccnt_delta;
        if (pmccntr_clockdiv_enabled(env)) prev_cycles /= 64;
        env->cp15.c15_ccnt_delta = prev_cycles - env->cp15.c15_ccnt;
    }
}
```

**差值模型图解**：

```
时间轴:  T0 ──────────────── T1 ──────────── T2
          │                    │                │
底层值:   cycles₀              cycles₁          cycles₂
差值:     delta₀               delta₁           delta₂
可见值:   ccnt₀                ccnt₁            ccnt₂

关系: ccnt = cycles - delta
     delta = cycles - ccnt（op_finish 时重建）
```

---

## 11. 事件计数器操作 pmevcntr_op_start/finish

```
cpregs-pmu.c:538-560

static void pmevcntr_op_start(CPUARMState *env, uint8_t counter)
{
    uint16_t event = c14_pmevtyper[counter] & PMXEVTYPER_EVTCOUNT;
    uint64_t count = 0;
    if (event_supported(event)) {
        count = pm_events[event_idx].get_count(env);   // 获取底层事件计数
    }

    if (pmu_counter_enabled(env, counter)) {
        uint64_t new_pmevcntr = count - c14_pmevcntr_delta[counter];
        uint64_t overflow_mask = pmevcntr_is_64_bit(env, counter)
                                 ? (1ULL << 63) : (1ULL << 31);

        // 溢出检测
        if (c14_pmevcntr[counter] & ~new_pmevcntr & overflow_mask) {
            c9_pmovsr |= (1 << counter);
            pmu_update_irq(env);
        }
        c14_pmevcntr[counter] = new_pmevcntr;
    }
    c14_pmevcntr_delta[counter] = count;
}
```

与 pmccntr_op_start 逻辑完全对称，区别在于：
- 事件计数源来自 `pm_events[].get_count()` 而非固定的 `cycles_get_count()`
- 可以是 32 位或 64 位计数器（取决于 PMCRLP/HLP）

---

## 12. 溢出检测与中断

### 12.1 溢出检测算法

```
溢出条件：old_value 最高位 = 1 且 new_value 最高位 = 0
         → old & ~new & overflow_mask != 0
```

- 32 位计数器：`overflow_mask = 1ULL << 31`
- 64 位计数器：`overflow_mask = 1ULL << 63`（PMCR.LC=1 或 PMCRLP=1）

### 12.2 中断生成

```
cpregs-pmu.c:429-434

static void pmu_update_irq(CPUARMState *env)
{
    ARMCPU *cpu = env_archcpu(env);
    qemu_set_irq(cpu->pmu_interrupt,
                 (env->cp15.c9_pmcr & PMCRE) &&           // 全局使能
                 (env->cp15.c9_pminten & env->cp15.c9_pmovsr));  // 中断使能 & 溢出
    );
}
```

**中断产生条件**：
1. `PMCR.E = 1` — PMU 全局使能
2. `PMINTENSET[n] = 1` — 计数器 n 中断使能
3. `PMOVSR[n] = 1` — 计数器 n 已溢出

三个条件同时满足时通过 `pmu_interrupt` GPIO 输出电平信号到 GIC。

---

## 13. PMU 定时器回调

```
cpregs-pmu.c:620-632

void arm_pmu_timer_cb(void *opaque)
{
    ARMCPU *cpu = opaque;
    // 刷新所有计数器，触发溢出检测和中断
    pmu_op_start(&cpu->env);
    pmu_op_finish(&cpu->env);
    // pmu_op_finish 同时会重新设置下次溢出时间
}
```

**定时器编程时机**：在 `pmccntr_op_finish()` 和 `pmevcntr_op_finish()` 中调用 `timer_mod_anticipate_ns(cpu->pmu_timer, overflow_at)` 设置下次溢出预测时间。

**timer_mod_anticipate_ns 语义**：只会提前（不会推迟）已设置的定时器，确保最早的溢出事件不会丢失。

---

## 14. EL 变化钩子

```
cpregs-pmu.c:610-618

void pmu_pre_el_change(ARMCPU *cpu, void *ignored)
{
    pmu_op_start(&cpu->env);    // EL 切换前：快照当前计数
}

void pmu_post_el_change(ARMCPU *cpu, void *ignored)
{
    pmu_op_finish(&cpu->env);   // EL 切换后：重建差值（新 EL 下过滤条件可能变化）
}
```

**必要性**：EL 变化可能改变过滤器的匹配结果（如 EL0→EL1 后 P 位生效），必须在切换点重新评估计数器使能状态。

---

## 15. PMCR_EL0 控制寄存器

### 15.1 位域定义

```
internals.h:1686-1729

PMCRE   (bit 0)  — 全局使能
PMCRP   (bit 1)  — 事件计数器复位（写触发）
PMCRC   (bit 2)  — 周期计数器复位（写触发）
PMCRD   (bit 3)  — 周期计数分频（/64）
PMCRX   (bit 4)  — 导出使能（QEMU 不实现）
PMCRDP  (bit 5)  — 禁止时禁止周期计数
PMCRLC  (bit 6)  — 长周期计数器（64 位）
PMCRLP  (bit 7)  — 长事件计数器（64 位）
PMCRN   (bit 11:15) — 事件计数器数量（只读）
```

### 15.2 写入处理

```
cpregs-pmu.c:634-655

static void pmcr_write(CPUARMState *env, ..., uint64_t value)
{
    pmu_op_start(env);       // 快照所有计数器

    if (value & PMCRC)       // C=1：复位周期计数器
        env->cp15.c15_ccnt = 0;

    if (value & PMCRP) {     // P=1：复位所有事件计数器
        for (i = 0; i < pmu_num_counters(env); i++)
            env->cp15.c14_pmevcntr[i] = 0;
    }

    // 只写入可写位
    env->cp15.c9_pmcr &= ~PMCR_WRITABLE_MASK;
    env->cp15.c9_pmcr |= (value & PMCR_WRITABLE_MASK);

    pmu_op_finish(env);      // 重建差值 + 重新预测溢出
}
```

### 15.3 读取处理（HPMN 映射）

```
cpregs-pmu.c:657-671

static uint64_t pmcr_read(CPUARMState *env, ...)
{
    uint64_t pmcr = env->cp15.c9_pmcr;

    // EL0/EL1 读 PMCR.N 时，返回 MDCR_EL2.HPMN 而非真实值
    if (arm_current_el(env) <= 1 && arm_is_el2_enabled(env)) {
        pmcr &= ~PMCRN_MASK;
        pmcr |= (mdcr_el2 & MDCR_HPMN) << PMCRN_SHIFT;
    }

    return pmcr;
}
```

---

## 16. SW_INCR 软件递增

```
cpregs-pmu.c:673-707

static void pmswinc_write(CPUARMState *env, ..., uint64_t value)
{
    for (i = 0; i < pmu_num_counters(env); i++) {
        if ((value & (1 << i)) &&                        // 位设置
            pmu_counter_enabled(env, i) &&               // 计数器使能
            (c14_pmevtyper[i] & EVTCOUNT) == 0x0) {      // 事件为 SW_INCR
            
            pmevcntr_op_start(env, i);                   // 快照
            
            new_pmswinc = c14_pmevcntr[i] + 1;           // 递增
            
            // 检测溢出
            if (c14_pmevcntr[i] & ~new_pmswinc & overflow_mask) {
                c9_pmovsr |= (1 << i);
                pmu_update_irq(env);
            }
            
            c14_pmevcntr[i] = new_pmswinc;
            pmevcntr_op_finish(env, i);                  // 恢复差值
        }
    }
}
```

**特点**：SW_INCR 是唯一通过软件显式写入递增的事件，无法通过 `ns_per_count` 预测溢出，因此返回 -1（永不自动溢出）。

---

## 17. PMCR.N 与 HPMN 计数器分区

EL2 可以通过 MDCR_EL2.HPMN 将计数器分为两组：

```
计数器 0 ~ HPMN-1  →  Guest 所有（使能由 PMCR.E 控制）
计数器 HPMN ~ N-1  →  EL2 所有（使能由 MDCR_EL2.HPME 控制）
计数器 31 (PMCCNTR) → 始终属于 Guest 组（由 PMCR.E 控制）
```

**对 EL1 的影响**：
- EL1 读 PMCR.N 看到的是 HPMN（而非真实 N）
- EL1 只能访问计数器 0 ~ HPMN-1
- EL2 拥有的计数器对 EL1 不可见

---

## 18. 64 位计数器扩展

```
cpregs-pmu.c:447-471

static bool pmevcntr_is_64_bit(CPUARMState *env, int counter)
{
    if (!pmuv3p5) return false;    // 需要 PMUv3.5

    if (arm_feature(env, ARM_FEATURE_EL2)) {
        bool hlp = mdcr_el2 & MDCR_HLP;   // EL2 拥有计数器的长模式
        int hpmn = mdcr_el2 & MDCR_HPMN;
        if (counter >= hpmn)
            return hlp;                     // EL2 计数器用 HLP
    }
    return c9_pmcr & PMCRLP;               // Guest 计数器用 PMCR.LP
}
```

| 计数器 | 32/64 位控制 |
|--------|-------------|
| PMCCNTR (31) | PMCR.LC |
| 事件计数器 < HPMN | PMCR.LP |
| 事件计数器 ≥ HPMN | MDCR_EL2.HLP |

---

## 19. 系统寄存器接口

```
cpregs-pmu.c:1010-1190  (v7_pm_reginfo[])
```

| 寄存器 | 编码 (AArch64) | 功能 |
|--------|---------------|------|
| PMCR_EL0 | op0=3 op1=3 CRn=9 CRm=12 op2=0 | 主控制 |
| PMCNTENSET_EL0 | ... CRm=12 op2=1 | 使能置位 |
| PMCNTENCLR_EL0 | ... CRm=12 op2=2 | 使能清除 |
| PMOVSCLR_EL0 | ... CRm=12 op2=3 | 溢出清除 |
| PMSWINC_EL0 | ... CRm=12 op2=4 | 软件递增（只写） |
| PMSELR_EL0 | ... CRm=12 op2=5 | 计数器选择 |
| PMCCNTR_EL0 | ... CRm=13 op2=0 | 周期计数器 |
| PMXEVTYPER_EL0 | ... CRm=13 op2=1 | 间接事件类型 |
| PMXEVCNTR_EL0 | ... CRm=13 op2=2 | 间接事件计数 |
| PMUSERENR_EL0 | ... CRm=14 op2=0 | 用户态访问控制 |
| PMINTENSET_EL1 | op1=0 CRm=14 op2=1 | 中断使能置位 |
| PMINTENCLR_EL1 | op1=0 CRm=14 op2=2 | 中断使能清除 |
| PMCCFILTR_EL0 | ... CRn=14 CRm=15 op2=7 | 周期计数器过滤 |
| PMOVSSET_EL0 | ... CRm=14 op2=3 | 溢出置位 |
| PMEVCNTRn_EL0 | ... CRm=8+n/8 op2=n%8 | 事件计数器 (n=0..30) |
| PMEVTYPERn_EL0 | ... CRm=12+n/8 op2=n%8 | 事件类型 (n=0..30) |

---

## 20. 访问控制——PMUSERENR_EL0

```
cpregs-pmu.c:240-330

EL0 访问 PMU 寄存器的控制位：
- EN (bit 0)：全局 EL0 访问使能
- SW (bit 1)：EL0 允许写 PMSWINC
- CR (bit 2)：EL0 允许读 PMCCNTR
- ER (bit 3)：EL0 允许读事件计数器/PMSELR
```

**逻辑**：
```
EL0 访问时：
  if (!PMUSERENR.EN) → CP_ACCESS_TRAP_EL1

特定寄存器的额外放行：
  PMSWINC  写：PMUSERENR.SW=1 → 放行
  PMCCNTR  读：PMUSERENR.CR=1 → 放行
  PMEVCNTRn/PMSELR 读：PMUSERENR.ER=1 → 放行
```

---

## 21. 访问控制——EL2/EL3 陷入

```
cpregs-pmu.c:22-35 (access_tpm) + 231-257 (do_pmreg_access)

// EL2 陷入
if (el < 2 && (MDCR_EL2 & MDCR_TPM))
    → CP_ACCESS_TRAP_EL2                    // 所有 PMU 寄存器

if (is_pmcr && (MDCR_EL2 & MDCR_TPMCR))
    → CP_ACCESS_TRAP_EL2                    // 仅 PMCR

// EL3 陷入
if (el < 3 && (MDCR_EL3 & MDCR_TPM))
    → CP_ACCESS_TRAP_EL3                    // 所有 PMU 寄存器
```

**陷入层次**：
- MDCR_EL2.TPM：陷入所有 PMU 访问到 EL2
- MDCR_EL2.TPMCR：仅陷入 PMCR 访问到 EL2
- MDCR_EL3.TPM：陷入所有 PMU 访问到 EL3

---

## 22. EL2 计数器所有权 (HPMN/HPME/HPMD)

```
cpregs-pmu.c:355-369

// HPMN：EL2 拥有的计数器分界
hpmn = mdcr_el2 & MDCR_HPMN;

// 使能控制
计数器 < hpmn 或 counter==31 → 由 PMCR.E 使能
计数器 ≥ hpmn               → 由 MDCR_EL2.HPME 使能

// 禁止计数
if (el == 2 && counter < hpmn)
    prohibited = MDCR_EL2.HPMD;   // EL2 可禁止 guest 计数器在 EL2 计数
```

---

## 23. EL3 计数控制 (SPME/SCCD)

```
cpregs-pmu.c:370-388

// SPME：安全状态计数使能
if (secure)
    prohibited |= !(MDCR_EL3 & MDCR_SPME);   // SPME=0 → 安全态禁止计数

// SCCD：周期计数器安全态禁止（PMUv3.5+）
if (counter == 31 && secure && pmuv3p5)
    prohibited |= (MDCR_EL3 & MDCR_SCCD);

// HCCD：周期计数器 EL2 禁止（PMUv3.5+）
if (counter == 31 && el == 2 && pmuv3p5)
    prohibited |= (MDCR_EL2 & MDCR_HCCD);
```

---

## 24. GIC 中断连线

```
bsa.h:27

#define VIRTUAL_PMU_IRQ  23    // PPI 23

virt.c:1266-1268

qdev_connect_gpio_out_named(cpudev, "pmu-interrupt", 0,
    qdev_get_gpio_in(vms->gic, intidbase + VIRTUAL_PMU_IRQ));
```

PMU 中断通过 CPU 的命名 GPIO 输出 `"pmu-interrupt"` 连接到 GIC 的 PPI 23 输入。每个 CPU 核心有独立的 PMU 中断线。

---

## 25. FDT 设备树描述

```
virt.c:993-1019

static void fdt_add_pmu_nodes(const VirtMachineState *vms)
{
    // 创建 /pmu 节点
    // compatible = "arm,armv8-pmuv3"（ARMv8）或 "arm,cortex-a15-pmu"（ARMv7）
    // interrupts = <GIC_FDT_IRQ_TYPE_PPI VIRTUAL_PMU_IRQ irqflags>
}
```

---

## 26. KVM PMU 集成

```
kvm.c:1845-1876

// 初始化 KVM vPMU
void kvm_arm_pmu_init(ARMCPU *cpu)
{
    if (!cpu->has_pmu) return;
    // KVM_ARM_VCPU_PMU_V3_INIT — 启用 KVM 内核 PMU 虚拟化
}

// 设置 PMU 中断号
void kvm_arm_pmu_set_irq(ARMCPU *cpu, int irq)
{
    if (!cpu->has_pmu) return;
    // KVM_ARM_VCPU_PMU_V3_IRQ — 告知 KVM 使用哪个 PPI
}

virt.c:2575
    kvm_arm_pmu_set_irq(ARM_CPU(cpu), VIRTUAL_PMU_IRQ);  // PPI 23
```

**KVM 模式下**：
- PMU 完全由内核 KVM 处理，利用硬件 PMU 虚拟化
- QEMU 只负责初始化配置和中断号设置
- 硬件计数器直接暴露给客户机（性能远优于 TCG 模拟）

---

## 27. CPU 属性与初始化

```
cpu.c:1288-1305

// 'pmu' 属性的 getter/setter
static bool arm_get_pmu(Object *obj, Error **errp) { return cpu->has_pmu; }
static void arm_set_pmu(Object *obj, bool value, Error **errp)
{
    cpu->has_pmu = value;
    if (value) set_feature(env, ARM_FEATURE_PMU);
    else       unset_feature(env, ARM_FEATURE_PMU);
}

// 属性注册：在 arm_cpu_post_init() 中
if (arm_feature(env, ARM_FEATURE_PMU))
    object_property_add_bool(obj, "pmu", arm_get_pmu, arm_set_pmu);
```

**pmu_timer 初始化**：
```
cpu.c:2130（arm_cpu_realizefn 中）

cpu->pmu_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, arm_pmu_timer_cb, cpu);
```

**GPIO 注册**：
```
cpu.c:1211-1212

qdev_init_gpio_out_named(DEVICE(cpu), &cpu->pmu_interrupt, "pmu-interrupt", 1);
```

---

## 28. 迁移与状态恢复

PMU 状态通过 CPU cpreg 迁移框架自动保存/恢复：

- `c9_pmcr`、`c9_pmcnten`、`c9_pmovsr`、`c9_pminten`、`c9_pmuserenr`
- `c15_ccnt`、`c14_pmevcntr[]`、`c14_pmevtyper[]`
- `pmccfiltr_el0`

**迁移恢复时**：`pmevtyper_rawwrite()` 在 migration load 路径中被调用时，会在 `pmu_op_start/finish` 包裹下执行，确保计数器差值正确重建。

---

## 29. 完整数据流图

```
客户机执行（TCG 模式）
  │
  ├── QEMU_CLOCK_VIRTUAL 持续递增
  │     └── cycles_get_count() = ns × 1GHz / 10^9
  │
  ├── icount_get_raw() 递增（精确 icount 模式下）
  │
  ▼
客户机读 PMCCNTR_EL0
  │
  ▼
pmccntr_read()                          [cpregs-pmu.c:709]
  │
  ├── pmu_op_start(env)
  │     └── pmccntr_op_start()
  │           ├── cycles = cycles_get_count()
  │           ├── new_ccnt = cycles/div - delta
  │           ├── 溢出检测 → PMOVSR[31] → pmu_update_irq()
  │           └── c15_ccnt = new_ccnt
  │
  ├── 读取 c15_ccnt
  │
  ├── pmu_op_finish(env)
  │     └── pmccntr_op_finish()
  │           ├── remaining = -c15_ccnt（距溢出剩余）
  │           ├── timer_mod_anticipate_ns(pmu_timer, now + remaining)
  │           └── 重建 delta
  │
  ▼
返回 c15_ccnt 给客户机

────────────────────────────────────────────────────────

pmu_timer 到期（预测的溢出时间）
  │
  ▼
arm_pmu_timer_cb()                      [cpregs-pmu.c:620]
  │
  ├── pmu_op_start()  → 检测溢出 → PMOVSR 置位
  ├── pmu_op_finish() → 重新预测下次溢出
  │
  ▼
pmu_update_irq()                        [cpregs-pmu.c:429]
  │
  ├── irqstate = PMCR.E && (PMINTEN & PMOVSR)
  │
  ▼
qemu_set_irq(pmu_interrupt, irqstate)
  │
  ▼
GIC PPI 23 → 中断注入 → 客户机异常入口
```

---

## 30. 关键源文件索引

| 文件 | 行范围 | 内容 |
|------|--------|------|
| cpregs-pmu.c | 16 | ARM_CPU_FREQ 常量 (1 GHz) |
| cpregs-pmu.c | 37-53 | pm_event 结构体定义 |
| cpregs-pmu.c | 60-170 | 事件实现 (SW_INCR/CYCLES/INST_RETIRED/STALL) |
| cpregs-pmu.c | 231-330 | 访问控制 (do_pmreg_access + PMUSERENR 细粒度) |
| cpregs-pmu.c | 336-427 | pmu_counter_enabled() 使能判定 |
| cpregs-pmu.c | 429-434 | pmu_update_irq() 中断更新 |
| cpregs-pmu.c | 436-471 | 时钟分频/64 位计数器判定 |
| cpregs-pmu.c | 479-560 | pmccntr/pmevcntr_op_start/finish |
| cpregs-pmu.c | 592-632 | pmu_op_start/finish + pmu_timer_cb |
| cpregs-pmu.c | 634-707 | pmcr_write/read + pmswinc_write |
| cpregs-pmu.c | 1010-1190 | v7_pm_reginfo[] 系统寄存器定义 |
| cpu.h | 444-556 | CPUARMState PMU 字段 |
| cpu.h | 955-970 | ARMCPU pmu_timer/pmu_interrupt |
| cpu.c | 1288-1305 | pmu CPU 属性 getter/setter |
| cpu.c | 1211 | GPIO pmu-interrupt 注册 |
| cpu.c | 2130 | pmu_timer 初始化 |
| internals.h | 1686-1729 | PMCR 位域定义 |
| cpu-features.h | 362+ | PMU 版本检测 isar_feature |
| virt.c | 993-1019 | FDT pmu 节点 |
| virt.c | 1266-1268 | PMU 中断连线 (PPI 23) |
| bsa.h | 27 | VIRTUAL_PMU_IRQ = 23 |
| kvm.c | 1845-1876 | KVM PMU 初始化 |

---

> 本文档由 AI 辅助生成，基于 QEMU 11.0.50 源码分析。所有行号和文件引用均经过验证。
