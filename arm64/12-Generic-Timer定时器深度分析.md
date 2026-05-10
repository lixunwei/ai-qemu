# ARM64 Generic Timer 定时器深度分析

> QEMU 11.0.50 · 分析基于 commit HEAD  
> 关联文档：[00-ARM64-CPU-GICv3-TCG深度分析](00-ARM64-CPU-GICv3-TCG深度分析.md)、[03-GICv3中断控制器模拟架构深度分析](03-GICv3中断控制器模拟架构深度分析.md)、[04-中断注入与处理深度分析](04-中断注入与处理深度分析.md)、[06-异常级别状态管理深度分析](06-异常级别状态管理深度分析.md)

---

## 目录

1. [概述](#1-概述)
2. [定时器类型枚举](#2-定时器类型枚举)
3. [数据结构](#3-数据结构)
4. [计数器系统](#4-计数器系统)
5. [频率配置](#5-频率配置)
6. [定时器初始化](#6-定时器初始化)
7. [核心重算逻辑 gt_recalc_timer](#7-核心重算逻辑-gt_recalc_timer)
8. [中断状态更新 gt_update_irq](#8-中断状态更新-gt_update_irq)
9. [QEMU 主机定时器回调](#9-qemu-主机定时器回调)
10. [CTL 寄存器写入](#10-ctl-寄存器写入)
11. [CVAL/TVAL 寄存器写入](#11-cvaltval-寄存器写入)
12. [计数器偏移体系](#12-计数器偏移体系)
13. [计数器读取](#13-计数器读取)
14. [VHE 重定向](#14-vhe-重定向)
15. [系统寄存器接口](#15-系统寄存器接口)
16. [访问控制——计数器陷入](#16-访问控制计数器陷入)
17. [访问控制——定时器陷入](#17-访问控制定时器陷入)
18. [EL3 安全定时器](#18-el3-安全定时器)
19. [Secure EL2 定时器](#19-secure-el2-定时器)
20. [CNTHCTL_EL2 写入](#20-cnthctl_el2-写入)
21. [CNTVOFF_EL2 写入](#21-cntvoff_el2-写入)
22. [CNTPOFF_EL2 与 FEAT_ECV](#22-cntpoff_el2-与-feat_ecv)
23. [GIC 中断连线](#23-gic-中断连线)
24. [FDT 设备树描述](#24-fdt-设备树描述)
25. [KVM 定时器集成](#25-kvm-定时器集成)
26. [迁移与 VMState](#26-迁移与-vmstate)
27. [定时器与 WFI/WFE](#27-定时器与-wfiwfe)
28. [RME 安全状态切换](#28-rme-安全状态切换)
29. [linux-user 模式](#29-linux-user-模式)
30. [完整数据流图](#30-完整数据流图)
31. [关键源文件索引](#31-关键源文件索引)

---

## 1. 概述

ARM Generic Timer 是 ARMv8 架构中标准的定时器/计数器子系统，提供统一的系统计数器（System Counter）和每 CPU 多个定时器。在 QEMU 中，Generic Timer **不是**独立的 QOM 设备，而是嵌入在 ARM CPU 对象中实现的，通过系统寄存器（CP15/AArch64 sysreg）接口暴露给客户机。

核心设计：
- 7 种定时器类型，每种对应一个独立的 `QEMUTimer` 和 GPIO 输出
- 物理计数器基于 `QEMU_CLOCK_VIRTUAL`，保证与客户机时间同步
- 通过 CNTVOFF/CNTPOFF 偏移实现虚拟计数器
- 定时器到期产生 PPI（Per-Processor Interrupt），连接到 GIC

---

## 2. 定时器类型枚举

```
gtimer.h:12-21

enum {
    GTIMER_PHYS       = 0,  // CNTP_*  — EL1 物理定时器
    GTIMER_VIRT       = 1,  // CNTV_*  — EL1 虚拟定时器
    GTIMER_HYP        = 2,  // CNTHP_* — EL2 物理定时器
    GTIMER_SEC        = 3,  // CNTPS_* — EL3 物理定时器（安全定时器）
    GTIMER_HYPVIRT    = 4,  // CNTHV_* — EL2 虚拟定时器（需 FEAT_VHE）
    GTIMER_S_EL2_PHYS = 5,  // CNTHPS_* — 安全 EL2 物理（需 FEAT_SEL2）
    GTIMER_S_EL2_VIRT = 6,  // CNTHVS_* — 安全 EL2 虚拟（需 FEAT_SEL2）
    #define NUM_GTIMERS  7
};
```

**设计要点**：
- EL1 有两个定时器：物理（CNTP）和虚拟（CNTV）
- EL2 有两个：物理（CNTHP）和虚拟（CNTHV，VHE 后引入）
- EL3 有一个物理定时器（CNTPS）
- Secure EL2（FEAT_SEL2）引入额外两个定时器
- 总计 7 个独立定时器实例，每个有独立的 CTL/CVAL/TVAL 寄存器组

---

## 3. 数据结构

### 3.1 ARMGenericTimer 结构体

```
cpu.h:137-140

typedef struct ARMGenericTimer {
    uint64_t cval;   // CompareValue 寄存器 — 64 位比较值
    uint64_t ctl;    // Control 寄存器 — ENABLE(bit0), IMASK(bit1), ISTATUS(bit2)
} ARMGenericTimer;
```

CTL 位域：
- **bit 0 — ENABLE**：定时器使能。0 = 禁用，ISTATUS 清零
- **bit 1 — IMASK**：中断屏蔽。1 = 不触发中断输出
- **bit 2 — ISTATUS**：中断状态（只读）。1 = 计数器已达到 CVAL

### 3.2 CPU 状态中的定时器字段

```
cpu.h:515-520

uint64_t c14_cntfrq;                    // CNTFRQ — 计数器频率寄存器
uint64_t c14_cntkctl;                   // CNTKCTL_EL1 — EL1 定时器控制
uint64_t cnthctl_el2;                   // CNTHCTL_EL2 — EL2 定时器控制
uint64_t cntvoff_el2;                   // CNTVOFF_EL2 — 虚拟计数器偏移
uint64_t cntpoff_el2;                   // CNTPOFF_EL2 — 物理计数器偏移（FEAT_ECV）
ARMGenericTimer c14_timer[NUM_GTIMERS]; // 7 个定时器实例
```

### 3.3 ARMCPU 中的主机定时器和 GPIO 输出

```
cpu.h:955-966

QEMUTimer *gt_timer[NUM_GTIMERS];         // 7 个 QEMU 主机定时器
qemu_irq gt_timer_outputs[NUM_GTIMERS];   // 7 个 GPIO 中断输出线
```

**架构关系**：

```
                     ┌─────────────────────────────────────────┐
                     │              ARMCPU                     │
                     │                                        │
                     │  env.cp15.c14_timer[7]  (客户机状态)    │
                     │    ├── [0].ctl, .cval  (CNTP)          │
                     │    ├── [1].ctl, .cval  (CNTV)          │
                     │    ├── [2].ctl, .cval  (CNTHP)         │
                     │    ├── [3].ctl, .cval  (CNTPS)         │
                     │    ├── [4].ctl, .cval  (CNTHV)         │
                     │    ├── [5].ctl, .cval  (CNTHPS)        │
                     │    └── [6].ctl, .cval  (CNTHVS)        │
                     │                                        │
                     │  gt_timer[7]  (QEMU 主机定时器)         │
                     │    ├── [0] → arm_gt_ptimer_cb          │
                     │    ├── [1] → arm_gt_vtimer_cb          │
                     │    ├── [2] → arm_gt_htimer_cb          │
                     │    ├── [3] → arm_gt_stimer_cb          │
                     │    ├── [4] → arm_gt_hvtimer_cb         │
                     │    ├── [5] → arm_gt_sel2timer_cb       │
                     │    └── [6] → arm_gt_sel2vtimer_cb      │
                     │                                        │
                     │  gt_timer_outputs[7]  (GPIO → GIC PPI) │
                     │    ├── [0] → PPI 30 (CNTP)             │
                     │    ├── [1] → PPI 27 (CNTV)             │
                     │    ├── [2] → PPI 26 (CNTHP)            │
                     │    ├── [3] → PPI 29 (CNTPS)            │
                     │    ├── [4] → PPI 28 (CNTHV)            │
                     │    ├── [5] → PPI 20 (CNTHPS)           │
                     │    └── [6] → PPI 19 (CNTHVS)           │
                     └─────────────────────────────────────────┘
```

---

## 4. 计数器系统

### 4.1 物理计数器值获取

```
helper.c:1340-1344

static uint64_t gt_get_countervalue(CPUARMState *env)
{
    ARMCPU *cpu = env_archcpu(env);
    return qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) / gt_cntfrq_period_ns(cpu);
}
```

**实现细节**：
- 使用 `QEMU_CLOCK_VIRTUAL`（虚拟时钟），不受主机实时性影响
- 除以 `gt_cntfrq_period_ns(cpu)` 将纳秒转换为定时器 tick 数
- 计数器值单调递增，不可被客户机修改
- 所有定时器共享同一个物理计数器，差异仅在偏移量

### 4.2 计数器读取函数

```
helper.c:1536-1545

static uint64_t gt_cnt_read(...)        // CNTPCT 读取
{
    uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_PHYS);
    return gt_get_countervalue(env) - offset;
}

static uint64_t gt_virt_cnt_read(...)   // CNTVCT 读取
{
    uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_VIRT);
    return gt_get_countervalue(env) - offset;
}
```

**关键公式**：
- `CNTPCT = PhysCount - CNTPOFF`（仅当 FEAT_ECV 且条件满足时减 CNTPOFF）
- `CNTVCT = PhysCount - CNTVOFF`（虚拟计数器 = 物理计数器 - 虚拟偏移）

---

## 5. 频率配置

### 5.1 频率常量

```
internals.h:85-86

#define GTIMER_DEFAULT_HZ     1000000000    // 1 GHz（ARMv8.6+ 架构要求）
#define GTIMER_BACKCOMPAT_HZ  62500000      // 62.5 MHz（旧版 QEMU 兼容）
```

### 5.2 频率选择逻辑

```
cpu.c:1827-1845

if (!cpu->gt_cntfrq_hz) {
    // ARMv8.6+ 架构要求 1 GHz
    // 旧 CPU 类型或 ≤9.0 版本机器类型使用兼容值 62.5 MHz
    if (arm_feature(env, ARM_FEATURE_BACKCOMPAT_CNTFRQ) ||
        cpu->backcompat_cntfrq) {
        cpu->gt_cntfrq_hz = GTIMER_BACKCOMPAT_HZ;    // 62.5 MHz → 16ns/tick
    } else {
        cpu->gt_cntfrq_hz = GTIMER_DEFAULT_HZ;       // 1 GHz → 1ns/tick
    }
}
```

### 5.3 周期计算

```
cpu.c:1364-1380

unsigned int gt_cntfrq_period_ns(ARMCPU *cpu)
{
    // 使用整数除法截断，保证 QEMUTimer scale 的精确逆运算
    // 避免负周期导致客户机定时器"粘滞"
    return NANOSECONDS_PER_SECOND / cpu->gt_cntfrq_hz;
}
```

| 频率 | 周期 (ns) | 说明 |
|------|-----------|------|
| 1 GHz | 1 ns | ARMv8.6+ 默认，最高精度 |
| 62.5 MHz | 16 ns | 旧版 QEMU 兼容值 |

### 5.4 CNTFRQ 重置

```
helper.c:2036-2041 (generic_timer_cp_reginfo 数组)

{ .name = "CNTFRQ_EL0", ...
  .resetfn = arm_gt_cntfrq_reset,       // 复位时 c14_cntfrq ← gt_cntfrq_hz
  .fieldoffset = offsetof(CPUARMState, cp15.c14_cntfrq),
}
```

**注意**：CNTFRQ 对软件是"纯 reads-as-written"的——写入不会真正改变 QEMU 的定时器频率，只是让软件能查询到频率值。

---

## 6. 定时器初始化

```
cpu.c:1847-1864

// 在 arm_cpu_realizefn() 中创建 7 个 QEMUTimer
uint64_t scale = gt_cntfrq_period_ns(cpu);

cpu->gt_timer[GTIMER_PHYS]      = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_ptimer_cb, cpu);
cpu->gt_timer[GTIMER_VIRT]      = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_vtimer_cb, cpu);
cpu->gt_timer[GTIMER_HYP]       = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_htimer_cb, cpu);
cpu->gt_timer[GTIMER_SEC]       = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_stimer_cb, cpu);
cpu->gt_timer[GTIMER_HYPVIRT]   = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_hvtimer_cb, cpu);
cpu->gt_timer[GTIMER_S_EL2_PHYS]= timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_sel2timer_cb, cpu);
cpu->gt_timer[GTIMER_S_EL2_VIRT]= timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_sel2vtimer_cb, cpu);
```

**设计要点**：
- 所有定时器使用 `QEMU_CLOCK_VIRTUAL` — 与客户机虚拟时间同步
- `scale` 参数让 `timer_mod()` 可以直接接收 tick 数，无需手动转换
- 每个定时器都有独立的回调函数，但最终都调用 `gt_recalc_timer()`
- GPIO 输出通过 `qdev_init_gpio_out()` 注册，由 machine 代码连接到 GIC

---

## 7. 核心重算逻辑 gt_recalc_timer

这是整个定时器子系统的核心函数，负责：ISTATUS 计算、主机定时器重编程、中断信号更新。

```
helper.c:1466-1526

static void gt_recalc_timer(ARMCPU *cpu, int timeridx)
{
    ARMGenericTimer *gt = &cpu->env.cp15.c14_timer[timeridx];

    if (gt->ctl & 1) {    // ENABLE = 1
        uint64_t offset = gt_indirect_access_timer_offset(&cpu->env, timeridx);
        uint64_t count = gt_get_countervalue(&cpu->env);
        int istatus = count - offset >= gt->cval;    // 无符号 64 位比较

        gt->ctl = deposit32(gt->ctl, 2, 1, istatus); // 设置 ISTATUS

        if (istatus) {
            // 已到期：下次翻转为 count 回绕到 0 的时刻
            if (offset > count) {
                nexttick = offset;             // count == offset 时翻转
            } else {
                nexttick = UINT64_MAX;         // 需要 2^64 回绕
            }
        } else {
            // 未到期：下次触发在 count == cval + offset
            if (uadd64_overflow(gt->cval, offset, &nexttick)) {
                nexttick = UINT64_MAX;         // 溢出时设最大值
            }
        }

        // 处理 QEMUTimer 有符号 64 位限制
        if (nexttick > INT64_MAX / gt_cntfrq_period_ns(cpu)) {
            timer_mod_ns(cpu->gt_timer[timeridx], INT64_MAX);
        } else {
            timer_mod(cpu->gt_timer[timeridx], nexttick);  // nexttick 是 tick 数
        }
    } else {
        // 禁用：清除 ISTATUS，取消主机定时器
        gt->ctl &= ~4;
        timer_del(cpu->gt_timer[timeridx]);
    }
    gt_update_irq(cpu, timeridx);    // 更新中断输出
}
```

**状态转换图**：

```
                    ctl.ENABLE = 0
                  ┌─────────────────┐
                  │   DISABLED      │
                  │  ISTATUS = 0    │
                  │  timer 已取消    │
                  └──────┬──────────┘
                         │ ENABLE ← 1
                         ▼
              ┌──────────────────────┐
              │     ENABLED          │
              │  count-offset < cval │
              │  ISTATUS = 0         │──── timer_mod(cval+offset) ──┐
              │  等待到期             │                              │
              └──────────────────────┘                              │
                         ▲                                          │
                         │ count-offset < cval（回绕后）             │
                         │                                          ▼
              ┌──────────────────────┐                   定时器到期回调
              │     FIRED            │◄──────────────── gt_recalc_timer
              │  count-offset >= cval│
              │  ISTATUS = 1         │
              │  IRQ = !IMASK        │
              └──────────────────────┘
```

---

## 8. 中断状态更新 gt_update_irq

```
helper.c:1346-1366

static void gt_update_irq(ARMCPU *cpu, int timeridx)
{
    CPUARMState *env = &cpu->env;
    uint64_t cnthctl = env->cp15.cnthctl_el2;
    ARMSecuritySpace ss = arm_security_space(env);

    // 基本条件：ISTATUS=1 && IMASK=0 → (ctl & 6) == 4
    int irqstate = (env->cp15.c14_timer[timeridx].ctl & 6) == 4;

    // CNTHCTL_EL2.CNTVMASK/CNTPMASK 可覆盖 IMASK（RME/Realm 安全状态下有效）
    if ((ss == ARMSS_Root || ss == ARMSS_Realm) &&
        ((timeridx == GTIMER_VIRT && (cnthctl & R_CNTHCTL_CNTVMASK_MASK)) ||
         (timeridx == GTIMER_PHYS && (cnthctl & R_CNTHCTL_CNTPMASK_MASK)))) {
        irqstate = 0;    // Hypervisor 强制屏蔽
    }

    qemu_set_irq(cpu->gt_timer_outputs[timeridx], irqstate);
}
```

**中断产生条件三要素**：
1. `ENABLE = 1` — 定时器已启用
2. `ISTATUS = 1` — 计数器已达到比较值
3. `IMASK = 0` — 中断未被屏蔽

**额外屏蔽**：CNTHCTL_EL2.CNTVMASK/CNTPMASK（仅 Root/Realm 安全状态，FEAT_RME）

---

## 9. QEMU 主机定时器回调

```
helper.c:1976-2023

void arm_gt_ptimer_cb(void *opaque)    { gt_recalc_timer(opaque, GTIMER_PHYS); }
void arm_gt_vtimer_cb(void *opaque)    { gt_recalc_timer(opaque, GTIMER_VIRT); }
void arm_gt_htimer_cb(void *opaque)    { gt_recalc_timer(opaque, GTIMER_HYP); }
void arm_gt_stimer_cb(void *opaque)    { gt_recalc_timer(opaque, GTIMER_SEC); }
void arm_gt_hvtimer_cb(void *opaque)   { gt_recalc_timer(opaque, GTIMER_HYPVIRT); }
void arm_gt_sel2timer_cb(void *opaque) { gt_recalc_timer(opaque, GTIMER_S_EL2_PHYS); }
void arm_gt_sel2vtimer_cb(void *opaque){ gt_recalc_timer(opaque, GTIMER_S_EL2_VIRT); }
```

**执行流程**：

```
QEMU_CLOCK_VIRTUAL 到期
  → arm_gt_Xtimer_cb()
    → gt_recalc_timer(cpu, timeridx)
      → 重新计算 ISTATUS
      → 如果 ISTATUS=1：保持到期状态，设置下次回绕时刻
      → 如果 ISTATUS=0：重新编程到 cval+offset
      → gt_update_irq()
        → qemu_set_irq(gt_timer_outputs[timeridx], irqstate)
          → GIC PPI 电平设置
```

---

## 10. CTL 寄存器写入

```
helper.c:1589-1609

static void gt_ctl_write(CPUARMState *env, const ARMCPRegInfo *ri,
                         int timeridx, uint64_t value)
{
    ARMCPU *cpu = env_archcpu(env);
    uint32_t oldval = env->cp15.c14_timer[timeridx].ctl;

    // 只有 bit[1:0] 可写（ENABLE, IMASK）；bit[2] ISTATUS 只读
    env->cp15.c14_timer[timeridx].ctl = deposit64(oldval, 0, 2, value);

    if ((oldval ^ value) & 1) {
        // ENABLE 位变化 → 完整重算（重编程主机定时器 + 更新 ISTATUS + IRQ）
        gt_recalc_timer(cpu, timeridx);
    } else if ((oldval ^ value) & 2) {
        // IMASK 位变化 → 仅更新 IRQ（不需要重算 ISTATUS）
        gt_update_irq(cpu, timeridx);
    }
}
```

**优化**：IMASK 切换只调用 `gt_update_irq()`，避免不必要的 `timer_mod()`。

---

## 11. CVAL/TVAL 寄存器写入

### 11.1 CVAL 写入

```
helper.c:1548-1555

static void gt_cval_write(CPUARMState *env, const ARMCPRegInfo *ri,
                          int timeridx, uint64_t value)
{
    env->cp15.c14_timer[timeridx].cval = value;
    gt_recalc_timer(env_archcpu(env), timeridx);  // 立即重编程
}
```

### 11.2 TVAL 写入

```
helper.c:1571-1578

static void do_tval_write(CPUARMState *env, int timeridx,
                          uint64_t value, uint64_t offset)
{
    // TVAL 是 32 位有符号倒计数视图
    // CVAL = 当前计数 - 偏移 + sign_extend(TVAL, 32)
    env->cp15.c14_timer[timeridx].cval =
        gt_get_countervalue(env) - offset + sextract64(value, 0, 32);
    gt_recalc_timer(env_archcpu(env), timeridx);
}
```

### 11.3 TVAL 读取

```
helper.c:1557-1561

static uint64_t do_tval_read(CPUARMState *env, int timeridx, uint64_t offset)
{
    // TVAL = (CVAL - (count - offset)) 截断为 32 位
    return (uint32_t)(env->cp15.c14_timer[timeridx].cval -
                      (gt_get_countervalue(env) - offset));
}
```

**关键关系**：
- `TVAL` 是 `CVAL` 的倒计数视图：`TVAL = CVAL - VirtualCount`
- 写 TVAL 会转换为 CVAL 存储：`CVAL = VirtualCount + TVAL`
- TVAL 是 32 位有符号值，CVAL 是 64 位无符号值

---

## 12. 计数器偏移体系

QEMU 实现了两套偏移机制：**间接访问偏移**（用于 ISTATUS/CVAL 比较）和**直接访问偏移**（用于计数器/TVAL 读写）。

### 12.1 间接访问偏移

```
helper.c:1390-1416

static uint64_t gt_indirect_access_timer_offset(CPUARMState *env, int timeridx)
{
    switch (timeridx) {
    case GTIMER_PHYS:
        return gt_phys_raw_cnt_offset(env);     // CNTPOFF（FEAT_ECV 条件下）
    case GTIMER_VIRT:
        return env->cp15.cntvoff_el2;            // CNTVOFF_EL2
    case GTIMER_HYP:
    case GTIMER_SEC:
    case GTIMER_HYPVIRT:
    case GTIMER_S_EL2_PHYS:
    case GTIMER_S_EL2_VIRT:
        return 0;                                // 无偏移
    }
}
```

### 12.2 直接访问偏移

```
helper.c:1418-1464

uint64_t gt_direct_access_timer_offset(CPUARMState *env, int timeridx)
{
    switch (timeridx) {
    case GTIMER_PHYS:
        if (arm_current_el(env) >= 2) return 0;   // EL2+ 不减 CNTPOFF
        return gt_phys_raw_cnt_offset(env);
    case GTIMER_VIRT:
        switch (arm_current_el(env)) {
        case 2:
            if (hcr & HCR_E2H) return 0;          // VHE 模式 EL2 读 CNTVCT 不减偏移
            break;
        case 0:
            if ((hcr & (HCR_E2H|HCR_TGE)) == (HCR_E2H|HCR_TGE))
                return 0;                          // Host EL0 不减偏移
            break;
        }
        return env->cp15.cntvoff_el2;              // 其余情况减 CNTVOFF
    case GTIMER_HYP ... GTIMER_S_EL2_VIRT:
        return 0;
    }
}
```

### 12.3 物理计数器偏移（FEAT_ECV）

```
helper.c:1379-1388

static uint64_t gt_phys_raw_cnt_offset(CPUARMState *env)
{
    // 条件：SCR.ECVEN=1 && CNTHCTL.ECV=1 && EL2 启用 &&
    //       不在 Host (E2H+TGE 不同时置位)
    if ((env->cp15.scr_el3 & SCR_ECVEN) &&
        FIELD_EX64(env->cp15.cnthctl_el2, CNTHCTL, ECV) &&
        arm_is_el2_enabled(env) &&
        (arm_hcr_el2_eff(env) & (HCR_E2H | HCR_TGE)) != (HCR_E2H | HCR_TGE)) {
        return env->cp15.cntpoff_el2;
    }
    return 0;
}
```

**偏移总结表**：

| 定时器类型 | 间接偏移（CVAL 比较用） | 直接偏移（CNTPCT/CNTVCT 读取） |
|-----------|------------------------|-------------------------------|
| GTIMER_PHYS | CNTPOFF（ECV 条件下） | EL≥2 → 0；否则 CNTPOFF |
| GTIMER_VIRT | CNTVOFF_EL2 | VHE EL2 → 0；Host EL0 → 0；否则 CNTVOFF |
| GTIMER_HYP | 0 | 0 |
| GTIMER_SEC | 0 | 0 |
| GTIMER_HYPVIRT | 0 | 0 |

---

## 13. 计数器读取

### 13.1 物理计数器 CNTPCT

```
helper.c:1536-1540

static uint64_t gt_cnt_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    return gt_get_countervalue(env) - gt_direct_access_timer_offset(env, GTIMER_PHYS);
}
```

### 13.2 虚拟计数器 CNTVCT

```
helper.c:1542-1546

static uint64_t gt_virt_cnt_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    return gt_get_countervalue(env) - gt_direct_access_timer_offset(env, GTIMER_VIRT);
}
```

**虚拟计数器偏移的意义**：Hypervisor 可以通过设置 CNTVOFF_EL2 为虚拟机呈现"从某个时间点开始"的计数器视图，这对于 VM 迁移后保持客户机时间连续性至关重要。

---

## 14. VHE 重定向

VHE（Virtualization Host Extensions，HCR.E2H=1）模式下，EL2 使用 EL1 的定时器寄存器名访问时，实际操作的是 EL2 的定时器。

### 14.1 物理定时器重定向

```
helper.c:1639-1649

static int gt_phys_redir_timeridx(CPUARMState *env)
{
    switch (arm_mmu_idx(env)) {
    case ARMMMUIdx_E20_0:       // VHE Host EL0
    case ARMMMUIdx_E20_2:       // VHE Host EL2
    case ARMMMUIdx_E20_2_PAN:   // VHE Host EL2 + PAN
        return GTIMER_HYP;      // → 使用 EL2 物理定时器
    default:
        return GTIMER_PHYS;      // → 使用 EL1 物理定时器
    }
}
```

### 14.2 虚拟定时器重定向

```
helper.c:1651-1661

static int gt_virt_redir_timeridx(CPUARMState *env)
{
    switch (arm_mmu_idx(env)) {
    case ARMMMUIdx_E20_0:
    case ARMMMUIdx_E20_2:
    case ARMMMUIdx_E20_2_PAN:
        return GTIMER_HYPVIRT;   // → 使用 EL2 虚拟定时器
    default:
        return GTIMER_VIRT;      // → 使用 EL1 虚拟定时器
    }
}
```

**VHE 重定向效果**：

```
VHE 模式下（HCR.E2H=1）：
  EL2 访问 CNTP_CTL_EL0  → 实际操作 CNTHP_CTL（GTIMER_HYP）
  EL2 访问 CNTV_CTL_EL0  → 实际操作 CNTHV_CTL（GTIMER_HYPVIRT）
  EL2 访问 CNTKCTL_EL1   → 重定向到 CNTHCTL_EL2

非 VHE 模式：
  所有访问按原始寄存器名操作
```

### 14.3 CNTKCTL_EL1 的 VHE 重定向

```
helper.c:2043-2049

{ .name = "CNTKCTL_EL1", ...
  .vhe_redir_to_el2 = ENCODE_AA64_CP_REG(3, 4, 14, 1, 0),   // → CNTHCTL_EL2
  .vhe_redir_to_el01 = ENCODE_AA64_CP_REG(3, 5, 14, 1, 0),  // EL01 别名
}
```

---

## 15. 系统寄存器接口

所有 Generic Timer 寄存器通过 `generic_timer_cp_reginfo[]` 数组注册。

```
helper.c:2025-2223

static const ARMCPRegInfo generic_timer_cp_reginfo[] = {
```

### 15.1 寄存器列表

| 寄存器 | 源码位置 | 编码 | 访问权限 | 说明 |
|--------|---------|------|---------|------|
| CNTFRQ (CP15) | helper.c:2031 | CRn=14 CRm=0 op2=0 | PL1_RW, PL0_R | 频率（32位别名） |
| CNTFRQ_EL0 | helper.c:2036 | op0=3 op1=3 CRn=14 CRm=0 op2=0 | PL1_RW, PL0_R | 频率（64位） |
| CNTKCTL_EL1 | helper.c:2043 | op0=3 op1=0 CRn=14 CRm=1 op2=0 | PL1_RW | EL0 访问控制 |
| CNTP_CTL_EL0 | helper.c:2070 | op0=3 op1=3 CRn=14 CRm=2 op2=1 | PL0_RW | 物理定时器控制 |
| CNTV_CTL_EL0 | helper.c:2088 | op0=3 op1=3 CRn=14 CRm=3 op2=1 | PL0_RW | 虚拟定时器控制 |
| CNTP_TVAL_EL0 | helper.c:2112 | op0=3 op1=3 CRn=14 CRm=2 op2=0 | PL0_RW | 物理倒计数 |
| CNTV_TVAL_EL0 | helper.c:2123 | op0=3 op1=3 CRn=14 CRm=3 op2=0 | PL0_RW | 虚拟倒计数 |
| CNTPCT_EL0 | helper.c:2135 | op0=3 op1=3 CRn=14 CRm=0 op2=1 | PL0_R | 物理计数器 |
| CNTVCT_EL0 | helper.c:2145 | op0=3 op1=3 CRn=14 CRm=0 op2=2 | PL0_R | 虚拟计数器 |
| CNTP_CVAL_EL0 | helper.c:2168 | op0=3 op1=3 CRn=14 CRm=2 op2=2 | PL0_RW | 物理比较值 |
| CNTV_CVAL_EL0 | helper.c:2186 | op0=3 op1=3 CRn=14 CRm=3 op2=2 | PL0_RW | 虚拟比较值 |
| CNTPS_TVAL_EL1 | helper.c:2200 | op0=3 op1=7 CRn=14 CRm=2 op2=0 | PL1_RW | 安全物理倒计数 |
| CNTPS_CTL_EL1 | helper.c:2208 | op0=3 op1=7 CRn=14 CRm=2 op2=1 | PL1_RW | 安全物理控制 |
| CNTPS_CVAL_EL1 | helper.c:2216 | op0=3 op1=7 CRn=14 CRm=2 op2=2 | PL1_RW | 安全物理比较值 |

### 15.2 NV2 重定向偏移

部分寄存器具有 `nv2_redirect_offset` 字段，用于 FEAT_NV2（Nested Virtualization）：

```
CNTP_CTL_EL0:  nv2_redirect_offset = 0x180 | NV2_REDIR_NV1
CNTV_CTL_EL0:  nv2_redirect_offset = 0x170 | NV2_REDIR_NV1
CNTP_CVAL_EL0: nv2_redirect_offset = 0x178 | NV2_REDIR_NV1
CNTV_CVAL_EL0: nv2_redirect_offset = 0x168 | NV2_REDIR_NV1
```

这些偏移指向 VNCR 页面中的对应位置，用于嵌套虚拟化时 EL1 定时器寄存器的陷入处理。

---

## 16. 访问控制——计数器陷入

```
helper.c:1157-1193

static CPAccessResult gt_counter_access(CPUARMState *env, int timeridx, bool isread)
{
    unsigned int cur_el = arm_current_el(env);
    bool has_el2 = arm_is_el2_enabled(env);
    uint64_t hcr = arm_hcr_el2_eff(env);

    switch (cur_el) {
    case 0:
        // Host EL0 (E2H+TGE=11)：检查 CNTHCTL_EL2.EL0[PV]CTEN
        if ((hcr & (HCR_E2H|HCR_TGE)) == (HCR_E2H|HCR_TGE)) {
            return extract32(env->cp15.cnthctl_el2, timeridx, 1)
                   ? CP_ACCESS_OK : CP_ACCESS_TRAP_EL2;
        }
        // Guest EL0：检查 CNTKCTL_EL1.EL0[PV]CTEN
        if (!extract32(env->cp15.c14_cntkctl, timeridx, 1))
            return CP_ACCESS_TRAP_EL1;
        /* fall through */
    case 1:
        // 检查 CNTHCTL_EL2.EL1PCTEN（物理计数器）
        if (has_el2 && timeridx == GTIMER_PHYS) {
            // 位置取决于 E2H：E2H=1 → bit10, E2H=0 → bit0
            if (位未置位) return CP_ACCESS_TRAP_EL2;
        }
        // 检查 CNTHCTL_EL2.EL1TVCT（虚拟计数器陷入）
        if (has_el2 && timeridx == GTIMER_VIRT) {
            if (FIELD_EX64(cnthctl_el2, CNTHCTL, EL1TVCT))
                return CP_ACCESS_TRAP_EL2;
        }
        break;
    }
    return CP_ACCESS_OK;
}
```

**陷入层次**：

```
EL0 → EL1 陷入：CNTKCTL_EL1 控制
EL0 → EL2 陷入：CNTHCTL_EL2 控制（VHE Host EL0）
EL1 → EL2 陷入：CNTHCTL_EL2 控制
```

---

## 17. 访问控制——定时器陷入

```
helper.c:1195-1241

static CPAccessResult gt_timer_access(CPUARMState *env, int timeridx, bool isread)
{
    switch (cur_el) {
    case 0:
        // Host EL0 (E2H+TGE=11)：检查 CNTHCTL_EL2.EL0[PV]TEN
        if ((hcr & (HCR_E2H|HCR_TGE)) == (HCR_E2H|HCR_TGE)) {
            return extract32(cnthctl_el2, 9 - timeridx, 1)
                   ? CP_ACCESS_OK : CP_ACCESS_TRAP_EL2;
        }
        // Guest EL0：检查 CNTKCTL_EL1.EL0[PV]TEN
        if (!extract32(c14_cntkctl, 9 - timeridx, 1))
            return CP_ACCESS_TRAP_EL1;
        /* fall through */
    case 1:
        // 物理定时器：E2H=1 → EL1PTEN(bit11), E2H=0 → EL1PCEN(bit1)
        if (has_el2 && timeridx == GTIMER_PHYS) { ... }
        // 虚拟定时器：EL1TVT 陷入
        if (has_el2 && timeridx == GTIMER_VIRT) {
            if (FIELD_EX64(cnthctl_el2, CNTHCTL, EL1TVT))
                return CP_ACCESS_TRAP_EL2;
        }
    }
}
```

**CNTKCTL_EL1 与 CNTHCTL_EL2 控制位对照**：

| 控制位 | 位置 | 功能 |
|--------|------|------|
| CNTKCTL.EL0PCTEN | bit 0 | EL0 访问物理计数器 CNTPCT |
| CNTKCTL.EL0VCTEN | bit 1 | EL0 访问虚拟计数器 CNTVCT |
| CNTKCTL.EL0PTEN | bit 8 | EL0 访问物理定时器 CNTP_* |
| CNTKCTL.EL0VTEN | bit 9 | EL0 访问虚拟定时器 CNTV_* |
| CNTHCTL.EL1PCTEN | bit 0 (no-E2H) / bit 10 (E2H) | EL1 访问物理计数器 |
| CNTHCTL.EL1PCEN | bit 1 (no-E2H) | EL1 访问物理定时器 |
| CNTHCTL.EL1PTEN | bit 11 (E2H) | EL1 访问物理定时器 |
| CNTHCTL.EL1TVT | FEAT_ECV | 陷入 EL1 虚拟定时器访问到 EL2 |
| CNTHCTL.EL1TVCT | FEAT_ECV | 陷入 EL1 虚拟计数器访问到 EL2 |

---

## 18. EL3 安全定时器

```
helper.c:1269-1298

static CPAccessResult gt_stimer_access(CPUARMState *env, ...)
{
    switch (arm_current_el(env)) {
    case 1:
        if (!arm_is_secure(env)) return CP_ACCESS_UNDEFINED;   // NS-EL1：未定义
        if (arm_is_el2_enabled(env)) return CP_ACCESS_UNDEFINED; // SEL2 启用：未定义
        if (!(scr_el3 & SCR_ST)) return CP_ACCESS_TRAP_EL3;   // SCR.ST=0：陷入 EL3
        return CP_ACCESS_OK;                                    // SCR.ST=1：允许
    case 0:
    case 2:
        return CP_ACCESS_UNDEFINED;                             // EL0/EL2：未定义
    case 3:
        return CP_ACCESS_OK;                                    // EL3：始终允许
    }
}
```

**EL3 安全定时器特点**：
- 只有 EL3 可以无条件访问
- Secure EL1 需要 SCR_EL3.ST=1 才能访问，否则陷入 EL3
- EL0/EL2 访问均为 UNDEFINED
- 使用独立的 PPI 29（ARCH_TIMER_S_EL1_IRQ）

---

## 19. Secure EL2 定时器

```
helper.c:1300-1335

static CPAccessResult gt_sel2timer_access(CPUARMState *env, ...)
{
    switch (arm_current_el(env)) {
    case 0:
        return CP_ACCESS_UNDEFINED;
    case 1:
        // NS-EL1 + no FEAT_NV2：陷入 EL2（UNDEF）
        // Secure-EL1 + 无 SEL2：UNDEF
        // 检查 CNTHCTL_EL2.EL1NVPCT / EL1NVVCT
    case 2:
        // Secure EL2 或 NS-EL2：条件允许
    case 3:
        return CP_ACCESS_OK;       // EL3 始终允许
    }
}
```

需要 FEAT_SEL2 支持，使用独立 PPI：
- CNTHPS（物理）→ PPI 20（ARCH_TIMER_S_EL2_IRQ）
- CNTHVS（虚拟）→ PPI 19（ARCH_TIMER_S_EL2_VIRT_IRQ）

---

## 20. CNTHCTL_EL2 写入

```
helper.c:1741-1782

static void gt_cnthctl_write(CPUARMState *env, ..., uint64_t value)
{
    uint32_t valid_mask = EL0PCTEN | EL0VCTEN | EVNTEN | EVNTDIR | EVNTI
                        | EL0VTEN | EL0PTEN | EL1PCTEN | EL1PTEN;

    // FEAT_RME：增加 CNTVMASK/CNTPMASK
    if (cpu_isar_feature(aa64_rme, cpu))
        valid_mask |= CNTVMASK | CNTPMASK;

    // FEAT_ECV traps：增加 EL1TVT/EL1TVCT/EL1NVPCT/EL1NVVCT/EVNTIS
    if (cpu_isar_feature(aa64_ecv_traps, cpu))
        valid_mask |= EL1TVT | EL1TVCT | EL1NVPCT | EL1NVVCT | EVNTIS;

    // FEAT_ECV：增加 ECV 位
    if (cpu_isar_feature(aa64_ecv, cpu))
        valid_mask |= ECV;

    value &= valid_mask;     // 清除 RES0 位
    raw_write(env, ri, value);

    // 如果 CNTVMASK 变化 → 更新虚拟定时器 IRQ
    if ((oldval ^ value) & CNTVMASK) gt_update_irq(cpu, GTIMER_VIRT);
    // 如果 CNTPMASK 变化 → 更新物理定时器 IRQ
    if ((oldval ^ value) & CNTPMASK) gt_update_irq(cpu, GTIMER_PHYS);
}
```

---

## 21. CNTVOFF_EL2 写入

```
helper.c:1784-1792

static void gt_cntvoff_write(CPUARMState *env, ..., uint64_t value)
{
    raw_write(env, ri, value);
    gt_recalc_timer(cpu, GTIMER_VIRT);  // 偏移变化 → 重算虚拟定时器
}
```

**重要**：写入 CNTVOFF_EL2 后必须重算虚拟定时器，因为 ISTATUS 的判定条件 `count - CNTVOFF >= cval` 依赖此偏移。

---

## 22. CNTPOFF_EL2 与 FEAT_ECV

### 22.1 访问控制

```
helper.c:2253-2262

static CPAccessResult gt_cntpoff_access(CPUARMState *env, ...)
{
    // EL2 访问时，若 SCR.ECVEN=0 → 陷入 EL3
    if (arm_current_el(env) == 2 && arm_feature(env, ARM_FEATURE_EL3) &&
        !(env->cp15.scr_el3 & SCR_ECVEN))
        return CP_ACCESS_TRAP_EL3;
    return CP_ACCESS_OK;
}
```

### 22.2 写入处理

```
helper.c:2264-2272

static void gt_cntpoff_write(CPUARMState *env, ..., uint64_t value)
{
    raw_write(env, ri, value);
    gt_recalc_timer(cpu, GTIMER_PHYS);  // 物理偏移变化 → 重算物理定时器
}
```

### 22.3 FEAT_ECV 自同步寄存器

```
helper.c:2230-2251

static const ARMCPRegInfo gen_timer_ecv_cp_reginfo[] = {
    { .name = "CNTVCTSS", ... .readfn = gt_virt_cnt_read },    // 自同步虚拟计数器
    { .name = "CNTVCTSS_EL0", ... .readfn = gt_virt_cnt_read },
    { .name = "CNTPCTSS", ... .readfn = gt_cnt_read },         // 自同步物理计数器
    { .name = "CNTPCTSS_EL0", ... .readfn = gt_cnt_read },
};
```

**注意**：QEMU 中所有 sysreg 都是"自同步"的（没有乱序执行问题），因此 CNTVCTSS/CNTPCTSS 的实现与 CNTVCT/CNTPCT 完全相同。

---

## 23. GIC 中断连线

```
virt.c:1239-1253

const int timer_irq[] = {
    [GTIMER_PHYS]      = ARCH_TIMER_NS_EL1_IRQ,      // PPI 30
    [GTIMER_VIRT]      = ARCH_TIMER_VIRT_IRQ,         // PPI 27
    [GTIMER_HYP]       = ARCH_TIMER_NS_EL2_IRQ,       // PPI 26
    [GTIMER_SEC]       = ARCH_TIMER_S_EL1_IRQ,        // PPI 29
    [GTIMER_HYPVIRT]   = ARCH_TIMER_NS_EL2_VIRT_IRQ,  // PPI 28
    [GTIMER_S_EL2_PHYS]= ARCH_TIMER_S_EL2_IRQ,        // PPI 20
    [GTIMER_S_EL2_VIRT]= ARCH_TIMER_S_EL2_VIRT_IRQ,   // PPI 19
};

for (irq = 0; irq < ARRAY_SIZE(timer_irq); irq++) {
    qdev_connect_gpio_out(cpudev, irq,
                          qdev_get_gpio_in(vms->gic, intidbase + timer_irq[irq]));
}
```

### PPI 分配表

```
bsa.h:25-33

#define ARCH_TIMER_S_EL2_VIRT_IRQ   19    // Secure EL2 虚拟
#define ARCH_TIMER_S_EL2_IRQ        20    // Secure EL2 物理
#define ARCH_TIMER_NS_EL2_IRQ       26    // NS EL2 物理
#define ARCH_TIMER_VIRT_IRQ         27    // EL1 虚拟
#define ARCH_TIMER_NS_EL2_VIRT_IRQ  28    // NS EL2 虚拟
#define ARCH_TIMER_S_EL1_IRQ        29    // EL3/Secure 物理
#define ARCH_TIMER_NS_EL1_IRQ       30    // EL1 物理
```

**连接方式**：CPU GPIO output[n] → GIC PPI input[intidbase + timer_irq[n]]

每个 CPU 核心都有独立的 7 条 GPIO 输出线连接到 GIC 的对应 PPI 输入。PPI 是 per-processor 的，所以每个核心的定时器中断互相独立。

---

## 24. FDT 设备树描述

```
virt.c:447-512

static void fdt_add_timer_nodes(const VirtMachineState *vms)
{
    // 设备树兼容字符串
    const char compat[] = "arm,armv8-timer\0arm,armv7-timer";  // ARMv8
    // 或 "arm,armv7-timer"                                    // ARMv7

    qemu_fdt_setprop(ms->fdt, "/timer", "always-on", NULL, 0);

    // 中断描述（均为 PPI、level-triggered）
    // 顺序：S-EL1, NS-EL1, Virtual, NS-EL2 [, NS-EL2-Virt]
    qemu_fdt_setprop_cells(ms->fdt, "/timer", "interrupts",
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_S_EL1_IRQ), irqflags,
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_NS_EL1_IRQ), irqflags,
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_VIRT_IRQ), irqflags,
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_NS_EL2_IRQ), irqflags,
        // 可选：ARCH_TIMER_NS_EL2_VIRT_IRQ
    );
}
```

**FDT 要点**：
- `always-on` 属性表示定时器在低功耗模式下仍然运行
- 中断触发方式：level-triggered（电平触发）
- GICv2 模式下还包含 CPU 掩码（每个 PPI 对应的 CPU 集合）
- 较新 virt 机器类型（ns_el2_virt_timer_irq=true）会额外描述 NS EL2 虚拟定时器中断

---

## 25. KVM 定时器集成

### 25.1 虚拟时间同步

```
kvm.c:1162-1205

// 从 KVM 读取虚拟时间
static void kvm_arm_get_virtual_time(ARMCPU *cpu)
{
    if (cpu->kvm_vtime_dirty) return;     // 已有缓存值
    kvm_get_one_reg(CPU(cpu), KVM_REG_ARM_TIMER_CNT, &cpu->kvm_vtime);
    cpu->kvm_vtime_dirty = true;
}

// 向 KVM 写入虚拟时间
static void kvm_arm_put_virtual_time(ARMCPU *cpu)
{
    if (!cpu->kvm_vtime_dirty) return;    // 没有需要写回的
    kvm_set_one_reg(CPU(cpu), KVM_REG_ARM_TIMER_CNT, &cpu->kvm_vtime);
}
```

### 25.2 迁移钩子

```
kvm.c:1075-1098

void kvm_arm_cpu_pre_save(ARMCPU *cpu)
{
    // 保存前：将缓存的虚拟时间写入 cpreg 数组
    if (cpu->kvm_vtime_dirty) {
        *kvm_arm_get_cpreg_ptr(cpu, KVM_REG_ARM_TIMER_CNT) = cpu->kvm_vtime;
    }
}

bool kvm_arm_cpu_post_load(ARMCPU *cpu)
{
    // 加载后：从 cpreg 数组恢复虚拟时间
    cpu->kvm_vtime = *kvm_arm_get_cpreg_ptr(cpu, KVM_REG_ARM_TIMER_CNT);
    cpu->kvm_vtime_dirty = true;
}
```

### 25.3 KVM 定时器 IRQ 回同步

KVM 模式下，定时器由内核处理（in-kernel device）。`kvm_arch_post_run()` 在每次 VCPU 退出后同步定时器 IRQ 状态回 QEMU，以便 QEMU 可以正确跟踪中断状态。

### 25.4 时间调整属性

```
kvm.c:550-564

// kvm-no-adjvtime 属性（默认 false = 启用时间调整）
// 启用时，KVM 会在 VCPU 暂停时补偿虚拟计时器偏移
```

---

## 26. 迁移与 VMState

定时器状态通过 CPU 的 cpreg 迁移机制保存：

- `c14_timer[].ctl` — 每个定时器的控制寄存器
- `c14_timer[].cval` — 每个定时器的比较值
- `c14_cntfrq` — 频率寄存器
- `c14_cntkctl` — EL1 定时器控制
- `cnthctl_el2` — EL2 定时器控制
- `cntvoff_el2` — 虚拟计数器偏移
- `cntpoff_el2` — 物理计数器偏移

这些字段通过 `vmstate_arm_cpu`（machine.c:1229）中的 cpreg 框架自动序列化和反序列化。迁移后通过 `gt_recalc_timer()` 重建主机定时器状态。

---

## 27. 定时器与 WFI/WFE

WFI（Wait For Interrupt）使 CPU 进入低功耗等待状态：

```
op_helper.c: HELPER(wfi)
  → cs->halted = 1
  → cpu_loop_exit()
```

**定时器唤醒机制**：
1. CPU 执行 WFI → `cs->halted = 1`
2. QEMU 主机定时器继续运行（`timer_mod` 已编程）
3. 定时器到期 → `arm_gt_Xtimer_cb()` → `gt_recalc_timer()` → `gt_update_irq()`
4. `qemu_set_irq()` 设置 GIC PPI 电平
5. GIC 产生中断 → `cpu_check_irqs()` → 检测到挂起中断
6. CPU 解除 halt 状态 → 恢复执行

**关键点**：`QEMUTimer` 基于主机事件循环，即使 CPU halted，到期回调仍然会执行。

---

## 28. RME 安全状态切换

```
helper.c:1368-1377

void gt_rme_post_el_change(ARMCPU *cpu, void *ignored)
{
    // 切换 Root ↔ Secure/NonSecure 安全状态时，
    // CNTHCTL_EL2 的 CNTVMASK/CNTPMASK 有效值可能改变
    gt_update_irq(cpu, GTIMER_VIRT);
    gt_update_irq(cpu, GTIMER_PHYS);
}
```

**场景**：FEAT_RME 中的 CNTVMASK/CNTPMASK 仅在 Root/Realm 安全状态下生效。当 CPU 通过 EL 切换改变安全状态（如 EL3→EL1、ERET 等）时，需要重新评估中断屏蔽状态。

---

## 29. linux-user 模式

```
helper.c:2289-2299

// linux-user 模式下没有 QEMUTimer，直接使用主机时钟
static uint64_t gt_virt_cnt_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    ARMCPU *cpu = env_archcpu(env);
    return cpu_get_clock() / gt_cntfrq_period_ns(cpu);
}
```

在 linux-user 模式下：
- 只支持 CNTVCT_EL0 读取（内核 4.12+ 允许 EL0 访问）
- 其他定时器寄存器不可访问
- 计数器直接基于 `cpu_get_clock()` 而非 `QEMU_CLOCK_VIRTUAL`

---

## 30. 完整数据流图

```
客户机写入 CNTV_CVAL_EL0 = X
  │
  ▼
gt_virt_redir_cval_write()          [helper.c:1801]
  │
  ├── VHE? → gt_virt_redir_timeridx() → GTIMER_HYPVIRT 或 GTIMER_VIRT
  │
  ▼
gt_cval_write(env, ri, timeridx, X) [helper.c:1548]
  │
  ├── c14_timer[timeridx].cval = X
  │
  ▼
gt_recalc_timer(cpu, timeridx)      [helper.c:1466]
  │
  ├── offset = gt_indirect_access_timer_offset(env, timeridx)
  │     └── GTIMER_VIRT → cntvoff_el2
  │
  ├── count = gt_get_countervalue(env)
  │     └── qemu_clock_get_ns(VIRTUAL) / period_ns
  │
  ├── istatus = (count - offset) >= cval ?
  │     ├── YES: ctl.ISTATUS=1, nexttick=回绕点
  │     └── NO:  ctl.ISTATUS=0, nexttick=cval+offset
  │
  ├── timer_mod(gt_timer[timeridx], nexttick)   ← 编程 QEMU 主机定时器
  │
  ▼
gt_update_irq(cpu, timeridx)        [helper.c:1346]
  │
  ├── irqstate = (ISTATUS && !IMASK) ? 1 : 0
  ├── RME CNTVMASK/CNTPMASK 检查
  │
  ▼
qemu_set_irq(gt_timer_outputs[timeridx], irqstate)
  │
  ▼
GIC PPI (e.g., PPI 27 for CNTV)
  │
  ▼
中断注入到 CPU → 客户机异常入口
```

---

## 31. 关键源文件索引

| 文件 | 行范围 | 内容 |
|------|--------|------|
| gtimer.h | 12-21 | 定时器类型枚举 (7 种) |
| cpu.h | 137-140 | ARMGenericTimer 结构体 |
| cpu.h | 515-520 | CPU 状态中的定时器字段 |
| cpu.h | 955-966 | QEMUTimer 和 GPIO 输出数组 |
| internals.h | 85-86 | 频率常量 (1GHz / 62.5MHz) |
| cpu.c | 1364-1380 | gt_cntfrq_period_ns() 周期计算 |
| cpu.c | 1827-1864 | 定时器初始化（7 个 QEMUTimer 创建） |
| helper.c | 1157-1298 | 访问控制函数（计数器/定时器/安全定时器） |
| helper.c | 1340-1344 | gt_get_countervalue() 物理计数器 |
| helper.c | 1346-1377 | gt_update_irq() 中断更新 + RME 回调 |
| helper.c | 1379-1464 | 偏移体系（raw/indirect/direct） |
| helper.c | 1466-1526 | gt_recalc_timer() 核心重算逻辑 |
| helper.c | 1536-1609 | 计数器/TVAL/CVAL/CTL 读写 |
| helper.c | 1639-1661 | VHE 重定向（phys/virt → hyp/hypvirt） |
| helper.c | 1741-1792 | CNTHCTL/CNTVOFF 写入处理 |
| helper.c | 1976-2023 | 7 个定时器回调函数 |
| helper.c | 2025-2223 | generic_timer_cp_reginfo[] 寄存器定义 |
| helper.c | 2230-2251 | FEAT_ECV 自同步寄存器 |
| helper.c | 2253-2281 | CNTPOFF_EL2 访问/写入 |
| virt.c | 447-512 | FDT 定时器节点 |
| virt.c | 1239-1253 | GIC PPI 连线 |
| bsa.h | 25-33 | ARCH_TIMER_*_IRQ PPI 编号定义 |
| kvm.c | 1075-1205 | KVM 定时器同步（get/put virtual time） |

---

> 本文档由 AI 辅助生成，基于 QEMU 11.0.50 源码分析。所有行号和文件引用均经过验证。
