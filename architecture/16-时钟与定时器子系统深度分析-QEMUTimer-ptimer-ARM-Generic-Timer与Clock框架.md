# 时钟与定时器子系统深度分析 — QEMUTimer、ptimer、ARM Generic Timer 与 Clock 框架

> 基于 QEMU 11.0.50 源码分析，聚焦时钟与定时器全栈：
> QEMUTimer 核心基础设施（4 种时钟类型、定时器链表、到期分发）、
> ptimer 周期定时器（策略标志、事务机制、单次/周期模式）、
> ARM Generic Timer（7 种定时器实例、系统寄存器访问、gt_recalc_timer 重算逻辑、
> GICv3 PPI 中断连接）、Clock 设备时钟框架（时钟树传播）、icount 指令计数模式。

---

## 目录

1. [定时器子系统全景](#1-定时器子系统全景)
2. [QEMUTimer 核心基础设施](#2-qemutimer-核心基础设施)
3. [时钟类型与时间获取](#3-时钟类型与时间获取)
4. [定时器到期分发](#4-定时器到期分发)
5. [主循环集成与虚拟时钟推进](#5-主循环集成与虚拟时钟推进)
6. [ptimer 周期定时器](#6-ptimer-周期定时器)
7. [ARM Generic Timer 架构](#7-arm-generic-timer-架构)
8. [ARM 定时器系统寄存器](#8-arm-定时器系统寄存器)
9. [gt_recalc_timer 重算逻辑](#9-gt_recalc_timer-重算逻辑)
10. [Clock 设备时钟框架](#10-clock-设备时钟框架)
11. [icount 指令计数模式](#11-icount-指令计数模式)
12. [总结](#12-总结)

---

## 1. 定时器子系统全景

```
┌───────────────────────────────────────────────────────────┐
│  Guest 视角                                               │
│  ARM Generic Timer 系统寄存器                              │
│  CNTPCT_EL0, CNTP_CTL_EL0, CNTP_CVAL_EL0, CNTVOFF_EL2   │
│                                                           │
│  gt_cnt_read()      gt_ctl_write()     gt_cval_write()    │
│       ↓                  ↓                   ↓             │
│  gt_get_countervalue() → qemu_clock_get_ns(VIRTUAL)       │
│                          gt_recalc_timer()                 │
│                               ↓                            │
│                     timer_mod(cpu->gt_timer[idx], nexttick)│
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│  QEMUTimer 核心层                                         │
│  include/qemu/timer.h + util/qemu-timer.c                 │
│                                                           │
│  ┌─────────────────────────────────────────────┐          │
│  │  QEMUTimerList (per clock type)             │          │
│  │  active_timers → timer1 → timer2 → ...      │          │
│  │                 (按 expire_time 升序)        │          │
│  └─────────────────────────────────────────────┘          │
│                                                           │
│  4 种时钟: REALTIME | VIRTUAL | HOST | VIRTUAL_RT         │
│                                                           │
│  timerlist_run_timers(): 弹出到期定时器 → 调用 cb(opaque) │
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│  主循环集成                                               │
│  GLib main loop ← timerlistgroup_deadline_ns()            │
│  aio_ctx_prepare/check → 设置 poll 超时                   │
│  qemu_clock_run_all_timers() → 遍历所有时钟类型           │
└───────────────────────────────────────────────────────────┘
```

---

## 2. QEMUTimer 核心基础设施

### QEMUTimer 结构

```c
// include/qemu/timer.h:85-93
struct QEMUTimer {
    int64_t expire_time;       // 到期时间（纳秒）
    QEMUTimerList *timer_list; // 所属定时器链表
    QEMUTimerCB *cb;           // 到期回调函数
    void *opaque;              // 回调私有数据
    QEMUTimer *next;           // 链表下一个
    int attributes;            // QEMU_TIMER_ATTR_EXTERNAL 等
    int scale;                 // 时间单位缩放（SCALE_NS/US/MS）
};
```

### QEMUTimerListGroup

```c
// include/qemu/timer.h:78-80
struct QEMUTimerListGroup {
    QEMUTimerList *tl[QEMU_CLOCK_MAX];  // 每种时钟一个链表
};

// 全局默认组
extern QEMUTimerListGroup main_loop_tlg;  // :95
```

### 定时器 API

```c
// util/qemu-timer.c:351-365
void timer_init_full(QEMUTimer *ts, QEMUTimerListGroup *timer_list_group,
                     QEMUClockType type, int scale, int attributes,
                     QEMUTimerCB *cb, void *opaque) {
    if (!timer_list_group)
        timer_list_group = &main_loop_tlg;           // :357
    ts->timer_list = timer_list_group->tl[type];     // :359
    ts->cb = cb;                                     // :360
    ts->scale = scale;                               // :362
    ts->expire_time = -1;                            // :364 — 初始未激活
}

// :460-473
void timer_mod_ns(QEMUTimer *ts, int64_t expire_time) {
    qemu_mutex_lock(&timer_list->active_timers_lock);
    timer_del_locked(timer_list, ts);                 // 先删除
    rearm = timer_mod_ns_locked(timer_list, ts, expire_time); // 按序插入
    qemu_mutex_unlock(&timer_list->active_timers_lock);
    if (rearm) timerlist_rearm(timer_list);           // 通知主循环
}

// :498-501
void timer_mod(QEMUTimer *ts, int64_t expire_time) {
    timer_mod_ns(ts, expire_time * ts->scale);        // 乘以缩放因子
}

// :447-456
void timer_del(QEMUTimer *ts) {
    timer_del_locked(timer_list, ts);                 // 设置 expire_time = -1
}
```

---

## 3. 时钟类型与时间获取

### 4 种时钟类型

```c
// include/qemu/timer.h:48-54
typedef enum {
    QEMU_CLOCK_REALTIME = 0,    // 实时时钟：VM 停止时继续运行
    QEMU_CLOCK_VIRTUAL = 1,     // 虚拟时钟：仅模拟执行时推进
    QEMU_CLOCK_HOST = 2,        // 宿主时钟：跟随 NTP/suspend
    QEMU_CLOCK_VIRTUAL_RT = 3,  // 虚拟实时：icount warp 用
    QEMU_CLOCK_MAX
} QEMUClockType;
```

### 时间获取

```c
// util/qemu-timer.c:650-663
int64_t qemu_clock_get_ns(QEMUClockType type) {
    switch (type) {
    case QEMU_CLOCK_REALTIME:
        return get_clock();                // 单调递增，clock_gettime(MONOTONIC)
    case QEMU_CLOCK_VIRTUAL:
        return cpus_get_virtual_clock();   // Guest 虚拟时间
    case QEMU_CLOCK_HOST:
        return get_clock_realtime();       // 宿主 wall clock（含 NTP）
    case QEMU_CLOCK_VIRTUAL_RT:
        return cpu_get_clock();            // icount 模式下的实时参考
    }
}
```

| 时钟类型 | 时间源 | VM 暂停时 | 典型用途 |
|---------|--------|----------|---------|
| REALTIME | `clock_gettime(MONOTONIC)` | 继续运行 | UI 更新、网络超时 |
| VIRTUAL | `cpus_get_virtual_clock()` | 停止 | **Guest 定时器**、设备模拟 |
| HOST | `clock_gettime(REALTIME)` | 继续运行 | RTC 模拟 |
| VIRTUAL_RT | `cpu_get_clock()` | 停止 | icount warp 参考时钟 |

---

## 4. 定时器到期分发

```c
// util/qemu-timer.c:518-603
bool timerlist_run_timers(QEMUTimerList *timer_list) {
    if (!qatomic_read(&timer_list->active_timers)) return false;  // :526

    current_time = qemu_clock_get_ns(timer_list->clock->type);    // :563

    qemu_mutex_lock(&timer_list->active_timers_lock);
    while ((ts = timer_list->active_timers)) {                     // :565
        if (!timer_expired_ns(ts, current_time))
            break;                                                 // :566-570

        // 从链表移除
        timer_list->active_timers = ts->next;                      // :585
        ts->next = NULL;
        ts->expire_time = -1;                                      // :587
        cb = ts->cb;
        opaque = ts->opaque;

        // 解锁后调用回调（允许回调中修改定时器）
        qemu_mutex_unlock(&timer_list->active_timers_lock);
        cb(opaque);                                                 // :593
        qemu_mutex_lock(&timer_list->active_timers_lock);

        progress = true;
    }
    qemu_mutex_unlock(&timer_list->active_timers_lock);
    return progress;
}
```

**关键设计**：回调在锁外执行（:592-593），允许回调中安全调用 `timer_mod()`/`timer_del()`。

---

## 5. 主循环集成与虚拟时钟推进

### Deadline 计算

```c
// util/qemu-timer.c:637-648
int64_t timerlistgroup_deadline_ns(QEMUTimerListGroup *tlg) {
    int64_t deadline = -1;
    for (type = 0; type < QEMU_CLOCK_MAX; type++) {
        if (qemu_clock_use_for_deadline(type)) {                   // :642
            deadline = qemu_soonest_timeout(deadline,
                timerlist_deadline_ns(tlg->tl[type]));             // :643-644
        }
    }
    return deadline;
}
```

GLib 主循环的 `aio_ctx_prepare()` 调用此函数获取最近 deadline，设置 `poll()` 超时。

### 运行所有定时器

```c
// util/qemu-timer.c:687-699
bool qemu_clock_run_all_timers(void) {
    bool progress = false;
    for (type = 0; type < QEMU_CLOCK_MAX; type++) {
        if (qemu_clock_use_for_deadline(type)) {
            progress |= qemu_clock_run_timers(type);               // :694
        }
    }
    return progress;
}
```

### 虚拟时钟推进（Warp）

```c
// util/qemu-timer.c:701-720
int64_t qemu_clock_advance_virtual_time(int64_t dest) {
    int64_t clock = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);

    while (clock < dest) {
        // 找到最近虚拟时钟定时器 deadline
        int64_t deadline = qemu_clock_deadline_ns_all(QEMU_CLOCK_VIRTUAL, ...);
        int64_t warp = qemu_soonest_timeout(dest - clock, deadline); // :709

        // 推进虚拟时钟
        qemu_virtual_clock_set_ns(clock + warp);                     // :711

        // 运行到期的虚拟时钟定时器
        qemu_clock_run_timers(QEMU_CLOCK_VIRTUAL);                  // :713
        timerlist_run_timers(aio_context->tlg.tl[QEMU_CLOCK_VIRTUAL]); // :714
        clock = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);
    }
    return clock;
}
```

**Warp 逻辑**：每次推进到最近的 deadline 或目标时间（取较小值），然后运行到期定时器。这确保定时器按正确顺序触发。

---

## 6. ptimer 周期定时器

### ptimer_state 结构

```c
// hw/core/ptimer.c:21-42
struct ptimer_state {
    uint8_t enabled;        // 0=禁用, 1=周期, 2=单次
    uint64_t limit;         // 重装值
    uint64_t delta;         // 当前计数值
    uint32_t period_frac;   // 周期小数部分（定点数）
    int64_t period;         // 周期（纳秒）
    int64_t last_event;     // 上次事件时间
    int64_t next_event;     // 下次事件时间
    uint8_t policy_mask;    // 策略标志
    QEMUTimer *timer;       // 底层 QEMUTimer（VIRTUAL 时钟）
    ptimer_cb callback;     // 到期回调
    void *callback_opaque;
    bool in_transaction;    // 事务标志
    bool need_reload;       // 延迟重装标志
};
```

### 初始化

```c
// hw/core/ptimer.c:456-478
ptimer_state *ptimer_init(ptimer_cb callback, void *callback_opaque,
                           uint8_t policy_mask) {
    s = g_new0(ptimer_state, 1);
    s->timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, ptimer_tick, s);  // :465
    s->policy_mask = policy_mask;
    s->callback = callback;
    return s;
}
```

### ptimer_tick — 到期处理

```c
// hw/core/ptimer.c:154-198
static void ptimer_tick(void *opaque) {
    ptimer_transaction_begin(s);                                   // :166

    if (s->enabled == 2) {
        // 单次模式：停止
        s->delta = 0;
        s->enabled = 0;                                           // :168-170
    } else {
        // 周期模式：重装并重新编程
        s->delta = s->limit;                                      // :188
        ptimer_reload(s, delta_adjust);                            // :190
    }

    if (trigger) {
        ptimer_trigger(s);  // 调用用户回调                        // :194
    }

    ptimer_transaction_commit(s);                                  // :197
}
```

### ptimer_reload — 重装与编程

```c
// hw/core/ptimer.c:50-152
static void ptimer_reload(ptimer_state *s, int delta_adjust) {
    // 防止超高频率定时器导致主循环忙碌
    if (s->enabled == 1 && (delta * period < 10000)
        && !icount_enabled()) {
        period = 10000 / delta;                                    // :140-143
    }

    s->last_event = s->next_event;
    s->next_event = s->last_event + delta * period;               // :147
    timer_mod(s->timer, s->next_event);                           // :151
}
```

### 策略标志

| 标志 | 含义 |
|------|------|
| `PTIMER_POLICY_WRAP_AFTER_ONE_PERIOD` | 计数到 0 后等一个周期再回绕 |
| `PTIMER_POLICY_CONTINUOUS_TRIGGER` | limit=0 时持续触发 |
| `PTIMER_POLICY_NO_IMMEDIATE_TRIGGER` | 重装时不立即触发 |
| `PTIMER_POLICY_NO_IMMEDIATE_RELOAD` | 启动时不立即重装 |
| `PTIMER_POLICY_NO_COUNTER_ROUND_DOWN` | 不向下取整 |
| `PTIMER_POLICY_TRIGGER_ONLY_ON_DECREMENT` | 仅递减到 0 时触发 |

---

## 7. ARM Generic Timer 架构

### 7 种定时器实例

```c
// target/arm/gtimer.h:12-21
enum {
    GTIMER_PHYS       = 0,  // CNTP_*  — EL1 物理定时器
    GTIMER_VIRT       = 1,  // CNTV_*  — EL1 虚拟定时器
    GTIMER_HYP        = 2,  // CNTHP_* — EL2 物理定时器
    GTIMER_SEC        = 3,  // CNTPS_* — EL3 安全物理定时器
    GTIMER_HYPVIRT    = 4,  // CNTHV_* — EL2 虚拟定时器（VHE）
    GTIMER_S_EL2_PHYS = 5,  // CNTHPS_* — S-EL2 物理（SEL2）
    GTIMER_S_EL2_VIRT = 6,  // CNTHVS_* — S-EL2 虚拟（SEL2）
#define NUM_GTIMERS   7
};
```

### CPUARMState 中的定时器状态

```c
// target/arm/cpu.h:136-140
typedef struct ARMGenericTimer {
    uint64_t cval;  // CompareValue 寄存器
    uint64_t ctl;   // Control 寄存器（ENABLE/IMASK/ISTATUS）
} ARMGenericTimer;

// 在 CPUARMState.cp15 中:
ARMGenericTimer c14_timer[NUM_GTIMERS];  // 7 个定时器实例
uint64_t c14_cntfrq;                     // CNTFRQ 计数频率
uint64_t cntvoff_el2;                    // CNTVOFF_EL2 虚拟偏移
```

### 计数器值获取

```c
// target/arm/helper.c:1339-1344
uint64_t gt_get_countervalue(CPUARMState *env) {
    ARMCPU *cpu = env_archcpu(env);
    return qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) / gt_cntfrq_period_ns(cpu);
}
// 计数值 = 虚拟时钟纳秒 / 每 tick 纳秒数
// gt_cntfrq_period_ns = NANOSECONDS_PER_SECOND / cntfrq
```

### 中断连接

```c
// target/arm/helper.c:1346-1365
static void gt_update_irq(ARMCPU *cpu, int timeridx) {
    // ISTATUS=1 且 IMASK=0 → 产生中断
    int irqstate = (env->cp15.c14_timer[timeridx].ctl & 6) == 4;  // :1352

    // CNTHCTL_EL2 mask 位覆盖（Root/Realm 安全状态）
    if ((ss == ARMSS_Root || ss == ARMSS_Realm) &&
        ((timeridx == GTIMER_VIRT && (cnthctl & CNTVMASK)) ||
         (timeridx == GTIMER_PHYS && (cnthctl & CNTPMASK)))) {
        irqstate = 0;                                              // :1361
    }

    qemu_set_irq(cpu->gt_timer_outputs[timeridx], irqstate);      // :1364
}
```

**PPI 映射**：`cpu->gt_timer_outputs[timeridx]` 连接到 GICv3 的 PPI（Private Peripheral Interrupt）：
- GTIMER_PHYS → PPI 30（非安全 EL1 物理定时器）
- GTIMER_VIRT → PPI 27（EL1 虚拟定时器）
- GTIMER_HYP → PPI 26（EL2 物理定时器）
- GTIMER_SEC → PPI 29（安全 EL1 物理定时器）

---

## 8. ARM 定时器系统寄存器

### 寄存器定义（ARMCPRegInfo）

```c
// target/arm/helper.c:2036-2216 (主要定时器寄存器)
{ .name = "CNTFRQ_EL0",   // 计数频率（只读 EL0，读写 EL1+）
  .opc0 = 3, .opc1 = 3, .crn = 14, .crm = 0, .opc2 = 0,
  .access = PL1_RW | PL0_R,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_cntfrq),     // :2039
},

{ .name = "CNTP_CTL_EL0",  // EL1 物理定时器控制
  .opc0 = 3, .opc1 = 3, .crn = 14, .crm = 2, .opc2 = 1,
  .type = ARM_CP_IO,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_timer[GTIMER_PHYS].ctl),
  .readfn = gt_phys_redir_ctl_read,
  .writefn = gt_phys_redir_ctl_write,                        // :2070-2079
},

{ .name = "CNTV_CTL_EL0",  // EL1 虚拟定时器控制
  .fieldoffset = offsetof(CPUARMState, cp15.c14_timer[GTIMER_VIRT].ctl),
  .readfn = gt_virt_redir_ctl_read,
  .writefn = gt_virt_redir_ctl_write,                        // :2088-2097
},
```

### CTL 控制位

```
bit[0]: ENABLE  — 1=定时器启用
bit[1]: IMASK   — 1=中断屏蔽
bit[2]: ISTATUS — 1=条件满足（count - offset >= cval），只读
```

### 计数器读取

```c
// target/arm/helper.c:1536-1545
static uint64_t gt_cnt_read(CPUARMState *env, const ARMCPRegInfo *ri) {
    uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_PHYS);
    return gt_get_countervalue(env) - offset;                     // :1539
}

static uint64_t gt_virt_cnt_read(CPUARMState *env, const ARMCPRegInfo *ri) {
    uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_VIRT);
    return gt_get_countervalue(env) - offset;                     // :1545
    // 虚拟计数器 = 物理计数器 - CNTVOFF_EL2
}
```

### CVAL/TVAL 写入

```c
// target/arm/helper.c:1548-1555
static void gt_cval_write(CPUARMState *env, const ARMCPRegInfo *ri,
                           int timeridx, uint64_t value) {
    env->cp15.c14_timer[timeridx].cval = value;                  // :1553
    gt_recalc_timer(env_archcpu(env), timeridx);                  // :1554
}

// :1571-1577
static void do_tval_write(CPUARMState *env, int timeridx,
                           uint64_t value, uint64_t offset) {
    // TVAL 写入转换为 CVAL: cval = count - offset + sext(tval)
    env->cp15.c14_timer[timeridx].cval =
        gt_get_countervalue(env) - offset + sextract64(value, 0, 32); // :1575-1576
    gt_recalc_timer(env_archcpu(env), timeridx);                       // :1577
}
```

### CTL 写入

```c
// target/arm/helper.c:1589-1609
static void gt_ctl_write(CPUARMState *env, const ARMCPRegInfo *ri,
                          int timeridx, uint64_t value) {
    uint32_t oldval = env->cp15.c14_timer[timeridx].ctl;
    env->cp15.c14_timer[timeridx].ctl = deposit64(oldval, 0, 2, value); // :1597

    if ((oldval ^ value) & 1) {
        // ENABLE 位翻转 → 完整重算
        gt_recalc_timer(cpu, timeridx);                            // :1600
    } else if ((oldval ^ value) & 2) {
        // IMASK 位翻转 → 只更新中断线
        gt_update_irq(cpu, timeridx);                              // :1607
    }
}
```

---

## 9. gt_recalc_timer 重算逻辑

这是 ARM Generic Timer 的核心函数，负责计算下次中断时间并编程 QEMUTimer：

```c
// target/arm/helper.c:1466-1526
static void gt_recalc_timer(ARMCPU *cpu, int timeridx) {
    ARMGenericTimer *gt = &cpu->env.cp15.c14_timer[timeridx];

    if (gt->ctl & 1) {  // 定时器启用
        uint64_t offset = gt_indirect_access_timer_offset(&cpu->env, timeridx);
        uint64_t count = gt_get_countervalue(&cpu->env);          // :1476

        // === 计算 ISTATUS ===
        int istatus = (count - offset) >= gt->cval;               // :1478
        gt->ctl = deposit32(gt->ctl, 2, 1, istatus);             // :1481

        // === 计算下次触发时间 ===
        if (istatus) {
            // 已到期：下次转换在 count 回绕过 offset 时
            if (offset > count)
                nexttick = offset;                                // :1492
            else
                nexttick = UINT64_MAX;                            // :1494
        } else {
            // 未到期：下次转换在 count == cval + offset
            if (uadd64_overflow(gt->cval, offset, &nexttick))
                nexttick = UINT64_MAX;                            // :1503-1504
        }

        // === 编程 QEMUTimer ===
        if (nexttick > INT64_MAX / gt_cntfrq_period_ns(cpu)) {
            // 超出 QEMUTimer 范围 → 设为最大值
            timer_mod_ns(cpu->gt_timer[timeridx], INT64_MAX);    // :1514
        } else {
            timer_mod(cpu->gt_timer[timeridx], nexttick);        // :1516
        }
    } else {
        // 定时器禁用：清除 ISTATUS，删除定时器
        gt->ctl &= ~4;                                           // :1521
        timer_del(cpu->gt_timer[timeridx]);                      // :1522
    }
    gt_update_irq(cpu, timeridx);                                // :1525
}
```

### 定时器偏移

```c
// target/arm/helper.c:1418-1464
uint64_t gt_direct_access_timer_offset(CPUARMState *env, int timeridx) {
    switch (timeridx) {
    case GTIMER_PHYS:
        if (arm_current_el(env) >= 2) return 0;
        return gt_phys_raw_cnt_offset(env);                       // :1434-1438

    case GTIMER_VIRT:
        // EL2 + E2H → 无偏移
        // EL0 + E2H + TGE → 无偏移
        // 其他 → CNTVOFF_EL2
        return env->cp15.cntvoff_el2;                             // :1454

    case GTIMER_HYP:
    case GTIMER_SEC:
    case GTIMER_HYPVIRT:
    case GTIMER_S_EL2_PHYS:
    case GTIMER_S_EL2_VIRT:
        return 0;                                                 // :1460
    }
}
```

### 时间流关系图

```
  QEMU_CLOCK_VIRTUAL (纳秒)
         │
         ├── / gt_cntfrq_period_ns(cpu) ──→ 原始计数值（count）
         │
         ├── GTIMER_PHYS: phys_count = count - phys_offset
         │                 ISTATUS = (phys_count >= cval)
         │
         ├── GTIMER_VIRT:  virt_count = count - cntvoff_el2
         │                 ISTATUS = (virt_count >= cval)
         │
         ├── GTIMER_HYP:   hyp_count = count（无偏移）
         │                 ISTATUS = (hyp_count >= cval)
         │
         └── GTIMER_SEC:   sec_count = count（无偏移）
                           ISTATUS = (sec_count >= cval)
```

---

## 10. Clock 设备时钟框架

### Clock 结构

```c
// include/hw/core/clock.h:71-92
struct Clock {
    Object parent_obj;

    uint64_t period;           // 周期（2^-32 纳秒定点数）
    char *canonical_path;      // 路径缓存（调试）
    ClockCallback *callback;   // 频率变化回调
    void *callback_opaque;
    unsigned int callback_events; // 关注的事件掩码

    uint32_t multiplier;       // 分频/倍频 — 乘数
    uint32_t divider;          //              除数

    // 时钟树
    Clock *source;             // 父时钟（源）
    QLIST_HEAD(, Clock) children;  // 子时钟链表
    QLIST_ENTRY(Clock) sibling;    // 兄弟链表
};
```

### 时钟事件

```c
// include/hw/core/clock.h:31-36
typedef enum ClockEvent {
    ClockUpdate = 1,     // 周期刚更新
    ClockPreUpdate = 2,  // 周期即将更新
} ClockEvent;
```

### 频率设置与传播

```c
// 设置频率
clock_set_hz(clk, 62500000);  // 设为 62.5MHz
// → period = CLOCK_PERIOD_1SEC / hz

// 连接父子时钟
clock_set_source(child_clk, parent_clk);

// 传播频率变化到所有子时钟
clock_propagate(parent_clk);
// → 遍历 children 链表
// → child.period = parent.period * child.multiplier / child.divider
// → 递归传播到孙子时钟
// → 调用 ClockPreUpdate/ClockUpdate 回调
```

### 单位转换

```c
// include/hw/core/clock.h:48-55
#define CLOCK_PERIOD_1SEC    (1000000000llu << 32)  // 1 秒的定点数表示
#define CLOCK_PERIOD_FROM_NS(ns) ((ns) * (CLOCK_PERIOD_1SEC / 1000000000llu))
#define CLOCK_PERIOD_FROM_HZ(hz) (((hz) != 0) ? CLOCK_PERIOD_1SEC / (hz) : 0u)
#define CLOCK_PERIOD_TO_HZ(per)  (((per) != 0) ? CLOCK_PERIOD_1SEC / (per) : 0u)
```

---

## 11. icount 指令计数模式

### 配置

```c
// include/exec/icount.h
typedef enum LegacyICState {
    ICOUNT_DISABLED = 0,   // 禁用（默认）
    ICOUNT_PRECISE = 1,    // 精确模式
    ICOUNT_ADAPTATIVE = 2, // 自适应模式
} LegacyICState;
```

### 虚拟时钟与指令计数的关系

在 icount 模式下：
- `QEMU_CLOCK_VIRTUAL` 不由实时时钟驱动，而由已执行的指令数驱动
- 每条指令推进固定纳秒数：`icount_time_shift` 控制精度
- CPU 空闲时，通过 warp 机制推进虚拟时钟到下一个定时器 deadline

### 对定时器的影响

```c
// util/qemu-timer.c:138-141
// icount 模式下，VIRTUAL 时钟不参与 deadline 计算
// （因为虚拟时钟由指令执行驱动，不需要 poll 超时）
bool qemu_clock_use_for_deadline(QEMUClockType type) {
    if (type == QEMU_CLOCK_VIRTUAL && icount_enabled())
        return false;
}

// hw/core/ptimer.c:140-143
// ptimer 在 icount 模式下不限制最低周期
if (s->enabled == 1 && (delta * period < 10000)
    && !icount_enabled()) {
    period = 10000 / delta;  // 非 icount 模式下限制最低 10μs
}
```

### ARM Generic Timer 与 icount

ARM Generic Timer 的计数值来自 `qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL)`（helper.c:1343），因此在 icount 模式下：
- 计数值随指令执行精确推进
- 定时器到期精确对齐到指令边界
- 适合确定性重放和精确时序调试

---

## 12. 总结

### 定时器层次架构

```
┌──────────────────────────────────────────────────┐
│  Guest 层：ARM Generic Timer 系统寄存器           │
│  CNTPCT_EL0/CNTP_CTL_EL0/CNTP_CVAL_EL0 等       │
│  7 种定时器实例 × (cval + ctl) 状态               │
│  gt_recalc_timer() 重算 → gt_update_irq() → PPI  │
├──────────────────────────────────────────────────┤
│  设备层：ptimer / 其他设备定时器                   │
│  ptimer_state: 周期/单次 + 策略 + 事务            │
│  通过 timer_mod(VIRTUAL) 编程底层定时器            │
├──────────────────────────────────────────────────┤
│  核心层：QEMUTimer + QEMUTimerList                │
│  4 种时钟类型 × 有序链表                          │
│  timer_mod_ns() 插入 → timerlist_run_timers() 分发│
├──────────────────────────────────────────────────┤
│  集成层：GLib main loop + AioContext              │
│  timerlistgroup_deadline_ns() → poll 超时          │
│  qemu_clock_advance_virtual_time() → warp         │
├──────────────────────────────────────────────────┤
│  Clock 框架：设备频率管理                          │
│  时钟树 + 分频/倍频 + 频率传播回调                │
└──────────────────────────────────────────────────┘
```

### 设计亮点

1. **4 种时钟类型**：精确分离不同时间域。Guest 定时器用 VIRTUAL（可暂停/恢复），UI/网络用 REALTIME（不受 VM 状态影响），RTC 用 HOST（跟随 NTP）。

2. **有序链表 + 锁外回调**：定时器按 expire_time 有序排列，分发时解锁后调用回调，允许回调安全修改定时器（如 ptimer_tick 中的 ptimer_reload → timer_mod）。

3. **ARM Generic Timer 直接映射 VIRTUAL 时钟**：`gt_get_countervalue()` 直接除以频率周期得到计数值。写入 CVAL 后 `gt_recalc_timer()` 立即计算下次中断并编程 QEMUTimer。

4. **ptimer 事务机制**：`ptimer_transaction_begin/commit` 确保一系列 ptimer 操作作为原子单元执行，避免中间状态触发不必要的定时器重编程。

5. **Warp 推进**：虚拟时钟不是连续推进而是跳跃到下一个 deadline。`qemu_clock_advance_virtual_time()` 逐步 warp 到目标时间，每步运行到期定时器。

6. **CNTVOFF_EL2 虚拟化偏移**：Hypervisor 通过 CNTVOFF_EL2 让每个 VM 看到不同的虚拟计数器值，实现透明的定时器虚拟化。

---

**关键源文件**：
- `include/qemu/timer.h` — QEMUTimer/QEMUClockType/QEMUTimerListGroup 定义
- `util/qemu-timer.c` — 定时器核心实现（init/mod/del/run/advance）
- `hw/core/ptimer.c` — 周期定时器实现
- `include/hw/core/clock.h` + `hw/core/clock.c` — 设备时钟框架
- `target/arm/helper.c` — ARM Generic Timer 全部实现
- `target/arm/gtimer.h` — 定时器索引枚举
- `target/arm/cpu.h` — ARMGenericTimer 结构
