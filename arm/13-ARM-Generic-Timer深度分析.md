# ARM Generic Timer 深度分析

> 基于 QEMU 11.0.50 源码，分析 ARM Generic Timer 的完整实现  
> 包括定时器设备模型、系统寄存器、计数器实现、中断连接、访问控制

---

## 目录

1. [概述](#1-概述)
2. [定时器类型枚举](#2-定时器类型枚举)
3. [CPUARMState 中的定时器状态](#3-cpuarmstate-中的定时器状态)
4. [ARMGenericTimer 结构](#4-armgenerictimer-结构)
5. [定时器 QOM 集成](#5-定时器-qom-集成)
6. [计数器实现](#6-计数器实现)
7. [计数频率与周期](#7-计数频率与周期)
8. [系统寄存器总览](#8-系统寄存器总览)
9. [CNTFRQ_EL0 频率寄存器](#9-cntfrq_el0-频率寄存器)
10. [CNTKCTL_EL1 访问控制](#10-cntkctl_el1-访问控制)
11. [物理定时器寄存器 CNTP_*](#11-物理定时器寄存器-cntp_)
12. [虚拟定时器寄存器 CNTV_*](#12-虚拟定时器寄存器-cntv_)
13. [Hypervisor 定时器 CNTHP/CNTHV](#13-hypervisor-定时器-cnthpcnthv)
14. [安全定时器 CNTPS](#14-安全定时器-cntps)
15. [CNTVOFF_EL2 虚拟偏移](#15-cntvoff_el2-虚拟偏移)
16. [CNTHCTL_EL2 Hypervisor 控制](#16-cnthctl_el2-hypervisor-控制)
17. [CTL 寄存器位定义](#17-ctl-寄存器位定义)
18. [gt_ctl_write() 控制写入](#18-gt_ctl_write-控制写入)
19. [gt_recalc_timer() 定时器重算](#19-gt_recalc_timer-定时器重算)
20. [gt_update_irq() 中断输出](#20-gt_update_irq-中断输出)
21. [CVAL 与 TVAL 写入](#21-cval-与-tval-写入)
22. [虚拟计数器偏移计算](#22-虚拟计数器偏移计算)
23. [访问控制函数体系](#23-访问控制函数体系)
24. [gt_cntfrq_access() 频率寄存器访问](#24-gt_cntfrq_access-频率寄存器访问)
25. [gt_counter_access() 计数器访问](#25-gt_counter_access-计数器访问)
26. [gt_ptimer_access() 物理定时器访问](#26-gt_ptimer_access-物理定时器访问)
27. [Timer→GIC PPI 连接](#27-timergic-ppi-连接)
28. [virt 机器定时器接线](#28-virt-机器定时器接线)
29. [设备树定时器描述](#29-设备树定时器描述)
30. [RME/Realm 定时器掩码](#30-rmerealm-定时器掩码)
31. [KVM 定时器处理](#31-kvm-定时器处理)
32. [ECV 扩展支持](#32-ecv-扩展支持)
33. [定时器与 EL 状态关系](#33-定时器与-el-状态关系)
34. [完整定时器触发流程](#34-完整定时器触发流程)
35. [定时器类型对比表](#35-定时器类型对比表)
36. [关键数据结构汇总](#36-关键数据结构汇总)
37. [总结](#37-总结)

---

## 1. 概述

ARM Generic Timer 是 ARM 架构中的标准定时器框架，为所有异常级别 (EL0-EL3) 提供独立的计时和比较定时器。在 QEMU 中，Generic Timer **不作为独立设备**存在，而是嵌入在 CPU 模型内部，使用 QEMU 虚拟时钟驱动计数器，通过 GPIO 输出线连接到 GIC 的 PPI 中断。

**关键特征**：
- 7 种定时器类型（物理/虚拟/Hypervisor/安全等）
- 每种定时器有 CTL/CVAL/TVAL 三个控制/比较寄存器
- 全局共享物理计数器，虚拟计数器通过 CNTVOFF 偏移
- 定时器输出连接 GIC PPI 线（PPI 14/11/10/13 等）

---

## 2. 定时器类型枚举

```c
// gtimer.h:12-21
enum {
    GTIMER_PHYS     = 0, /* CNTP_* ; EL1 physical timer */
    GTIMER_VIRT     = 1, /* CNTV_* ; EL1 virtual timer */
    GTIMER_HYP      = 2, /* CNTHP_* ; EL2 physical timer */
    GTIMER_SEC      = 3, /* CNTPS_* ; EL3 physical timer */
    GTIMER_HYPVIRT  = 4, /* CNTHV_* ; EL2 virtual timer ; only if FEAT_VHE */
    GTIMER_S_EL2_PHYS = 5, /* CNTHPS_* ; only if FEAT_SEL2 */
    GTIMER_S_EL2_VIRT = 6, /* CNTHVS_* ; only if FEAT_SEL2 */
#define NUM_GTIMERS   7
};
```

每种定时器对应特定的异常级别和安全状态：

| 索引 | 名称 | 寄存器前缀 | 用途 | 依赖 FEAT |
|------|------|-----------|------|-----------|
| 0 | GTIMER_PHYS | CNTP_* | EL1 物理定时器 | 基本 |
| 1 | GTIMER_VIRT | CNTV_* | EL1 虚拟定时器 | 基本 |
| 2 | GTIMER_HYP | CNTHP_* | EL2 物理定时器 | EL2 |
| 3 | GTIMER_SEC | CNTPS_* | 安全 EL1 物理定时器 | EL3 |
| 4 | GTIMER_HYPVIRT | CNTHV_* | EL2 虚拟定时器 | VHE |
| 5 | GTIMER_S_EL2_PHYS | CNTHPS_* | 安全 EL2 物理定时器 | SEL2 |
| 6 | GTIMER_S_EL2_VIRT | CNTHVS_* | 安全 EL2 虚拟定时器 | SEL2 |

---

## 3. CPUARMState 中的定时器状态

```c
// cpu.h:515-520
uint64_t c14_cntfrq;    /* Counter Frequency register */
uint64_t c14_cntkctl;   /* Timer Control register (CNTKCTL_EL1) */
uint64_t cnthctl_el2;   /* Counter/Timer Hyp Control register */
uint64_t cntvoff_el2;   /* Counter Virtual Offset register */
uint64_t cntpoff_el2;   /* Counter Physical Offset register */
ARMGenericTimer c14_timer[NUM_GTIMERS];
```

这些字段位于 `env->cp15` 结构中，是 CP15 系统寄存器空间的一部分。`c14_timer[]` 数组为每种定时器类型维护独立的状态。

---

## 4. ARMGenericTimer 结构

每个定时器实例包含三个关键字段：

- **ctl** (uint32_t)：控制寄存器，包含 ENABLE、IMASK、ISTATUS 位
- **cval** (uint64_t)：比较值寄存器 (CompareValue)
- TVAL 不单独存储，是 CVAL 的 32 位下溢计数视图

---

## 5. 定时器 QOM 集成

定时器在 CPU 实例初始化时创建，作为 CPU 的一部分而非独立设备：

```c
// cpu.c - arm_cpu_instance_init / arm_cpu_realizefn
qdev_init_gpio_out(DEVICE(cpu), cpu->gt_timer_outputs, NUM_GTIMERS);
cpu->gt_timer[i] = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_*_cb, cpu);
```

**关键点**：
- `gt_timer_outputs[]` — 每种定时器类型一个 GPIO 输出 qemu_irq，连接到 GIC PPI
- `gt_timer[]` — 每种定时器类型一个 QEMUTimer，用于调度下次到期
- 使用 `QEMU_CLOCK_VIRTUAL` 虚拟时钟驱动

---

## 6. 计数器实现

```c
// helper.c:1339-1344
uint64_t gt_get_countervalue(CPUARMState *env)
{
    ARMCPU *cpu = env_archcpu(env);
    return qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) / gt_cntfrq_period_ns(cpu);
}
```

**计数器值 = QEMU 虚拟时钟纳秒数 / 每计数周期纳秒数**

这是全局物理计数器，所有定时器共享。虚拟计数器通过减去 CNTVOFF_EL2 偏移获得。

---

## 7. 计数频率与周期

- `gt_cntfrq_hz` — CPU 属性，默认 62.5MHz (ARM 推荐频率)
- `gt_cntfrq_period_ns(cpu)` — 将频率转换为每计数的纳秒数
- `arm_gt_cntfrq_reset()` — 复位时将 `CNTFRQ` 设置为 `cpu->gt_cntfrq_hz`
- `CNTFRQ` 是纯软件可读写寄存器，写入不会实际改变定时器频率

---

## 8. 系统寄存器总览

```c
// helper.c:2025-2223
static const ARMCPRegInfo generic_timer_cp_reginfo[] = {
    // CNTFRQ_EL0, CNTKCTL_EL1
    // CNTP_CTL/TVAL/CVAL (物理定时器)
    // CNTV_CTL/TVAL/CVAL (虚拟定时器)
    // CNTPCT_EL0, CNTVCT_EL0 (计数器)
    // CNTPS_* (安全定时器)
    ...
};
```

这个大数组定义了所有 Generic Timer 系统寄存器，包括 AArch32 和 AArch64 编码。

---

## 9. CNTFRQ_EL0 频率寄存器

```c
// helper.c:2036-2041
{ .name = "CNTFRQ_EL0", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 3, .crn = 14, .crm = 0, .opc2 = 0,
  .access = PL1_RW | PL0_R, .accessfn = gt_cntfrq_access,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_cntfrq),
  .resetfn = arm_gt_cntfrq_reset,
},
```

- EL0 只读，EL1+ 可读写
- 写入不影响实际计数频率（纯软件约定值）
- 复位值由 `cpu->gt_cntfrq_hz` 决定
- 只有最高异常级别可以写入（`gt_cntfrq_access` 检查）

---

## 10. CNTKCTL_EL1 访问控制

```c
// helper.c:2043-2050
{ .name = "CNTKCTL_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .opc1 = 0, .crn = 14, .crm = 1, .opc2 = 0,
  .access = PL1_RW,
  .vhe_redir_to_el2 = ENCODE_AA64_CP_REG(3, 4, 14, 1, 0),
  .vhe_redir_to_el01 = ENCODE_AA64_CP_REG(3, 5, 14, 1, 0),
  .fieldoffset = offsetof(CPUARMState, cp15.c14_cntkctl),
},
```

**CNTKCTL_EL1 位字段**：
- bit[0]: `EL0PCTEN` — EL0 可访问物理计数器
- bit[1]: `EL0VCTEN` — EL0 可访问虚拟计数器
- bit[8]: `EL0PTEN` — EL0 可访问物理定时器
- bit[9]: `EL0VTEN` — EL0 可访问虚拟定时器

支持 VHE 重定向：当 HCR_EL2.{E2H,TGE}=11 时，CNTKCTL_EL1 重定向到 CNTHCTL_EL2。

---

## 11. 物理定时器寄存器 CNTP_*

```c
// helper.c:2052-2079
{ .name = "CNTP_CTL", ... .accessfn = gt_ptimer_access,
  .fieldoffset = ... cp15.c14_timer[GTIMER_PHYS].ctl,
  .readfn = gt_phys_redir_ctl_read,
  .writefn = gt_phys_redir_ctl_write, },
```

- **CNTP_CTL_EL0** — 物理定时器控制（ENABLE/IMASK/ISTATUS）
- **CNTP_CVAL_EL0** — 物理定时器比较值（64 位绝对值）
- **CNTP_TVAL_EL0** — 物理定时器倒计数（32 位有符号，CVAL = counter + TVAL）
- 区分 Secure/NonSecure 版本（`ARM_CP_SECSTATE_S/NS`）
- Secure 态访问 `GTIMER_SEC`，NonSecure 态访问 `GTIMER_PHYS`
- 支持 `gt_phys_redir` 重定向（VHE 模式下重定向到 Hypervisor 定时器）

---

## 12. 虚拟定时器寄存器 CNTV_*

```c
// helper.c:2080-2097
{ .name = "CNTV_CTL_EL0", ... .accessfn = gt_vtimer_access,
  .fieldoffset = ... cp15.c14_timer[GTIMER_VIRT].ctl,
  .readfn = gt_virt_redir_ctl_read,
  .writefn = gt_virt_redir_ctl_write, },
```

虚拟定时器与物理定时器结构相同，但比较时使用虚拟计数器值：

**虚拟计数器 = 物理计数器 - CNTVOFF_EL2**

这允许 Hypervisor 为每个虚拟机设置不同的时间基准。

---

## 13. Hypervisor 定时器 CNTHP/CNTHV

- **CNTHP_*_EL2** — EL2 物理定时器（`GTIMER_HYP`），用于 Hypervisor 自身定时
- **CNTHV_*_EL2** — EL2 虚拟定时器（`GTIMER_HYPVIRT`），仅 FEAT_VHE 可用
- 这些寄存器只在 EL2 可访问

---

## 14. 安全定时器 CNTPS

```c
// helper.c: generic_timer_cp_reginfo[]
{ .name = "CNTPS_TVAL_EL1", ... .accessfn = gt_stimer_access, ... },
{ .name = "CNTPS_CTL_EL1", ... .accessfn = gt_stimer_access, ... },
{ .name = "CNTPS_CVAL_EL1", ... .accessfn = gt_stimer_access, ... },
```

- 对应 `GTIMER_SEC`，为安全世界 EL1 提供定时器
- 通过 `gt_stimer_access()` 检查 SCR_EL3.NS 状态

---

## 15. CNTVOFF_EL2 虚拟偏移

```c
// helper.c:1784-1792
// CNTVOFF_EL2 write path
```

- 由 Hypervisor (EL2) 设置
- 虚拟计数器读取时：`CNTVCT_EL0 = CNTPCT_EL0 - CNTVOFF_EL2`
- 虚拟定时器比较时也应用此偏移
- 允许 Hypervisor 为每个 Guest VM 提供独立的时间视图

---

## 16. CNTHCTL_EL2 Hypervisor 控制

```c
// helper.c:1741-1782
// CNTHCTL_EL2 write path
```

**CNTHCTL_EL2 关键位**：
- `EL1PCTEN` — 允许 EL1 访问物理计数器
- `EL1PCEN` — 允许 EL1 访问物理定时器
- `EL0VCTEN/EL0PCTEN` — VHE 模式下的 EL0 访问控制
- `CNTVMASK/CNTPMASK` — RME/Realm 模式下的定时器输出掩码

---

## 17. CTL 寄存器位定义

每种定时器的 CTL 寄存器共享相同的位布局：

| 位 | 名称 | 描述 |
|----|------|------|
| [0] | ENABLE | 定时器使能。1=启用比较，0=禁用 |
| [1] | IMASK | 中断掩码。1=屏蔽中断输出，0=允许 |
| [2] | ISTATUS | 中断状态（只读）。1=计数器≥CVAL，0=未到期 |

**中断输出逻辑**：`IRQ = ISTATUS && !IMASK`（CTL 的 bit[2:1] == 0b10）

---

## 18. gt_ctl_write() 控制写入

```c
// helper.c:1589-1609
static void gt_ctl_write(CPUARMState *env, const ARMCPRegInfo *ri,
                         int timeridx, uint64_t value)
{
    ARMCPU *cpu = env_archcpu(env);
    uint32_t oldval = env->cp15.c14_timer[timeridx].ctl;

    env->cp15.c14_timer[timeridx].ctl = deposit64(oldval, 0, 2, value);
    if ((oldval ^ value) & 1) {
        /* Enable toggled */
        gt_recalc_timer(cpu, timeridx);
    } else if ((oldval ^ value) & 2) {
        /* IMASK toggled: don't need to recalculate,
         * just set the interrupt line based on ISTATUS */
        gt_update_irq(cpu, timeridx);
    }
}
```

**写入行为**：
- 只有 bit[1:0]（ENABLE 和 IMASK）可写，bit[2]（ISTATUS）只读
- ENABLE 切换 → 完整重算定时器（`gt_recalc_timer`）
- 仅 IMASK 切换 → 只更新中断输出线（`gt_update_irq`），无需重算

---

## 19. gt_recalc_timer() 定时器重算

```c
// helper.c:1466-1526
static void gt_recalc_timer(ARMCPU *cpu, int timeridx)
{
    ARMGenericTimer *gt = &cpu->env.cp15.c14_timer[timeridx];

    if (gt->ctl & 1) {
        /* Timer enabled */
        uint64_t offset = gt_indirect_access_timer_offset(&cpu->env, timeridx);
        uint64_t count = gt_get_countervalue(&cpu->env);
        int istatus = count - offset >= gt->cval;  // 无符号比较

        gt->ctl = deposit32(gt->ctl, 2, 1, istatus);

        if (istatus) {
            // 已到期：下次翻转是 count 回绕时
            nexttick = (offset > count) ? offset : UINT64_MAX;
        } else {
            // 未到期：下次触发在 count == cval + offset
            if (uadd64_overflow(gt->cval, offset, &nexttick))
                nexttick = UINT64_MAX;
        }

        // 设置 QEMUTimer
        if (nexttick > INT64_MAX / gt_cntfrq_period_ns(cpu)) {
            timer_mod_ns(cpu->gt_timer[timeridx], INT64_MAX);
        } else {
            timer_mod(cpu->gt_timer[timeridx], nexttick);
        }
    } else {
        /* Timer disabled: ISTATUS=0, cancel timer */
        gt->ctl &= ~4;
        timer_del(cpu->gt_timer[timeridx]);
    }
    gt_update_irq(cpu, timeridx);
}
```

**核心算法**：
1. 读取当前计数器值和偏移
2. 无符号 64 位比较：`count - offset >= cval`
3. 设置 ISTATUS 位
4. 计算下次状态变化时间点
5. 通过 `timer_mod()` 调度 QEMUTimer
6. 更新中断输出线

**溢出处理**：当 `nexttick` 超过 QEMUTimer 的 `INT64_MAX` 限制时，设置为最大值，到期后再次重算。

---

## 20. gt_update_irq() 中断输出

```c
// helper.c:1346-1366
static void gt_update_irq(ARMCPU *cpu, int timeridx)
{
    CPUARMState *env = &cpu->env;
    uint64_t cnthctl = env->cp15.cnthctl_el2;
    ARMSecuritySpace ss = arm_security_space(env);
    /* ISTATUS && !IMASK */
    int irqstate = (env->cp15.c14_timer[timeridx].ctl & 6) == 4;

    /* RME: CNTHCTL_EL2.CNT[VP]MASK overrides IMASK */
    if ((ss == ARMSS_Root || ss == ARMSS_Realm) &&
        ((timeridx == GTIMER_VIRT && (cnthctl & R_CNTHCTL_CNTVMASK_MASK)) ||
         (timeridx == GTIMER_PHYS && (cnthctl & R_CNTHCTL_CNTPMASK_MASK)))) {
        irqstate = 0;
    }

    qemu_set_irq(cpu->gt_timer_outputs[timeridx], irqstate);
}
```

**中断条件**：`CTL[2:1] == 0b10`，即 ISTATUS=1 且 IMASK=0。

**RME 掩码**：在 Root/Realm 安全空间中，CNTHCTL_EL2 的 CNTVMASK/CNTPMASK 位可以额外覆盖中断输出，用于 Realm Management Extension 场景。

---

## 21. CVAL 与 TVAL 写入

```c
// helper.c:1548-1578 (概略)
// CVAL write: 直接写入 gt->cval, 然后 gt_recalc_timer
// TVAL write: 计算 cval = (counter - offset) + sextract64(value, 0, 32)
//             然后写入 gt->cval, gt_recalc_timer
```

- **CVAL**：直接设置 64 位绝对比较值
- **TVAL**：设置相对倒计数值，自动转换为 CVAL = 当前计数 + TVAL
- TVAL 读取：返回 `cval - (counter - offset)` 的低 32 位有符号值

---

## 22. 虚拟计数器偏移计算

虚拟定时器使用偏移后的计数器进行比较：

```
offset = gt_indirect_access_timer_offset(env, timeridx)
```

对于 `GTIMER_VIRT`，偏移为 `CNTVOFF_EL2`。  
对于 `GTIMER_PHYS`（带 FEAT_ECV），偏移为 `CNTPOFF_EL2`。  
其他定时器偏移为 0。

比较条件：`(counter - offset) >= cval`

---

## 23. 访问控制函数体系

Generic Timer 使用分层访问控制，每个寄存器组有独立的 `accessfn`：

| 函数 | 检查内容 | 保护目标 |
|------|---------|---------|
| `gt_cntfrq_access()` | CNTKCTL/CNTHCTL EL0 位 | CNTFRQ 读取 |
| `gt_counter_access()` | CNTKCTL/CNTHCTL + E2H/TGE | 计数器读取 |
| `gt_ptimer_access()` | CNTKCTL.EL0PTEN + CNTHCTL | 物理定时器 |
| `gt_vtimer_access()` | CNTKCTL.EL0VTEN + CNTHCTL | 虚拟定时器 |
| `gt_stimer_access()` | SCR_EL3.NS 检查 | 安全定时器 |
| `gt_sel2timer_access()` | SEL2 功能检查 | 安全 EL2 定时器 |

---

## 24. gt_cntfrq_access() 频率寄存器访问

```c
// helper.c:1115-1155
static CPAccessResult gt_cntfrq_access(CPUARMState *env,
                                       const ARMCPRegInfo *ri, bool isread)
{
    int el = arm_current_el(env);
    switch (el) {
    case 0:
        // 检查 CNTKCTL/CNTHCTL 的 EL0PCTEN/EL0VCTEN
        if (!extract32(cntkctl, 0, 2))
            return CP_ACCESS_TRAP_EL1;
        break;
    case 1:
        // AArch32 Secure EL1 写入 UNDEF
        break;
    }
    // 只有最高异常级别可写
    if (!isread && el < arm_highest_el(env))
        return CP_ACCESS_UNDEFINED;
    return CP_ACCESS_OK;
}
```

**关键规则**：
- EL0：需要 CNTKCTL 的 PL0PCTEN 或 PL0VCTEN 至少一个为 1
- 写入：只有最高实现的异常级别才能写（通常 EL3）
- AArch32 Secure EL1 写入是 UNDEF（不是 trap）

---

## 25. gt_counter_access() 计数器访问

```c
// helper.c:1157-1240
static CPAccessResult gt_counter_access(CPUARMState *env, int timeridx,
                                        bool isread)
```

这个函数检查对 CNTPCT_EL0/CNTVCT_EL0 计数器的访问权限，考虑：

1. **EL0**：检查 CNTKCTL 或 CNTHCTL（VHE 模式）的对应位
2. **EL1**：检查 CNTHCTL_EL2 的陷阱位（EL1PCTEN 等）
3. **E2H+TGE**：特殊 VHE 路径，使用 CNTHCTL 而非 CNTKCTL

---

## 26. gt_ptimer_access() 物理定时器访问

物理/虚拟定时器的访问控制：
- EL0：检查 CNTKCTL_EL1.EL0PTEN/EL0VTEN
- EL1：检查 CNTHCTL_EL2 的陷阱位
- 支持 VHE 重定向和 FEAT_NV 嵌套虚拟化

---

## 27. Timer→GIC PPI 连接

定时器通过 CPU 的 GPIO 输出线连接到 GIC 的 PPI (Private Peripheral Interrupt)：

```
CPU gt_timer_outputs[GTIMER_*] → GIC PPI input
```

标准 PPI 编号（ARM 架构定义）：

| 定时器 | PPI 号 | 中断 ID (16+PPI) | 常量 |
|--------|--------|-----------------|------|
| GTIMER_SEC (安全物理) | 13 | 29 | ARCH_TIMER_S_EL1_IRQ |
| GTIMER_PHYS (非安全物理) | 14 | 30 | ARCH_TIMER_NS_EL1_IRQ |
| GTIMER_VIRT (虚拟) | 11 | 27 | ARCH_TIMER_VIRT_IRQ |
| GTIMER_HYP (Hypervisor) | 10 | 26 | ARCH_TIMER_NS_EL2_IRQ |

---

## 28. virt 机器定时器接线

```c
// hw/arm/virt.c
// timer_irq[] mapping:
// GTIMER_PHYS  -> ARCH_TIMER_NS_EL1_IRQ
// GTIMER_VIRT  -> ARCH_TIMER_VIRT_IRQ
// GTIMER_HYP   -> ARCH_TIMER_NS_EL2_IRQ
// GTIMER_SEC   -> ARCH_TIMER_S_EL1_IRQ
// ...

qdev_connect_gpio_out(cpudev, irq,
    qdev_get_gpio_in(vms->gic, intidbase + timer_irq[irq]));
```

virt 机器在创建 CPU 和 GIC 后，将每个 CPU 的定时器输出线连接到 GIC 对应的 PPI 输入。`intidbase` 是每个 CPU 的中断偏移量。

---

## 29. 设备树定时器描述

virt 机器生成的设备树中包含 `/timer` 节点，描述定时器中断：

```
timer {
    compatible = "arm,armv8-timer";
    interrupts = <GIC_PPI 13 ...>,  /* Secure Physical */
                 <GIC_PPI 14 ...>,  /* Non-Secure Physical */
                 <GIC_PPI 11 ...>,  /* Virtual */
                 <GIC_PPI 10 ...>;  /* Hypervisor */
    always-on;
};
```

---

## 30. RME/Realm 定时器掩码

```c
// helper.c:1354-1362 (gt_update_irq)
if ((ss == ARMSS_Root || ss == ARMSS_Realm) &&
    ((timeridx == GTIMER_VIRT && (cnthctl & R_CNTHCTL_CNTVMASK_MASK)) ||
     (timeridx == GTIMER_PHYS && (cnthctl & R_CNTHCTL_CNTPMASK_MASK)))) {
    irqstate = 0;
}
```

FEAT_RME 引入的 CNTHCTL_EL2.CNTVMASK/CNTPMASK 位，在 Root/Realm 安全空间中可以强制掩码定时器中断输出，覆盖定时器自身的 IMASK 设置。

---

## 31. KVM 定时器处理

在 KVM 加速模式下：
- 定时器由 KVM 内核模块直接处理，使用主机硬件定时器
- QEMU 侧仍然暴露定时器 GPIO 线，但实际计数/比较由 KVM 完成
- `QEMU_CLOCK_VIRTUAL` 定时器仅用于 TCG 模式
- KVM 通过 vGIC 直接注入定时器 PPI 中断

---

## 32. ECV 扩展支持

```c
// helper.c:2230-2281
// ECV (Enhanced Counter Virtualization) 相关寄存器:
// CNTVCTSS, CNTPCTSS — Self-Synchronized 计数器（保证无需屏障）
// CNTPOFF_EL2 — 物理计数器偏移（类似 CNTVOFF 但用于物理计数器）
```

FEAT_ECV 允许 Hypervisor 为物理计数器也设置偏移 (`CNTPOFF_EL2`)，以及提供自同步的计数器读取变体。

---

## 33. 定时器与 EL 状态关系

| EL | 可用定时器 | 访问控制来源 |
|----|-----------|-------------|
| EL0 | CNTP_*/CNTV_* (受控) | CNTKCTL_EL1 / CNTHCTL_EL2(VHE) |
| EL1 | CNTP_*/CNTV_* | CNTHCTL_EL2 可陷阱 |
| EL2 | CNTHP_*/CNTHV_* + 所有 EL1 定时器 | 无陷阱 |
| EL3 | CNTPS_* + 所有定时器 | 无陷阱 |

---

## 34. 完整定时器触发流程

```
1. gt_ctl_write() / gt_cval_write()
   → gt_recalc_timer()
     → count = gt_get_countervalue()     // QEMU虚拟时钟/频率
     → istatus = (count - offset >= cval) // 无符号64位比较
     → timer_mod(gt_timer[idx], nexttick) // 设置QEMUTimer到期

2. QEMUTimer 到期回调 (arm_gt_*_cb):
   → gt_recalc_timer()                   // 重新计算ISTATUS
     → gt_update_irq()
       → irqstate = (ctl & 6) == 4       // ISTATUS=1 && IMASK=0
       → qemu_set_irq(gt_timer_outputs[idx], irqstate)

3. GIC 接收 PPI:
   → gicv3_set_irq() (PPI 路径)
   → gicv3_redist_set_irq()
   → gicv3_update()
   → gicv3_cpuif_update()
   → arm_cpu_set_irq()
   → cpu_interrupt(CPU_INTERRUPT_FIQ/IRQ)

4. CPU 执行循环:
   → arm_cpu_exec_interrupt()
   → 进入异常向量 (IRQ/FIQ entry)
   → Guest OS 定时器中断处理
   → 读 IAR 确认 → 写 EOIR 结束
```

---

## 35. 定时器类型对比表

| 特性 | 物理定时器 | 虚拟定时器 | Hypervisor 定时器 | 安全定时器 |
|------|-----------|-----------|-----------------|-----------|
| 计数器 | CNTPCT_EL0 | CNTVCT_EL0 | CNTPCT_EL0 | CNTPCT_EL0 |
| 偏移 | 无(或CNTPOFF) | CNTVOFF_EL2 | 无 | 无 |
| 最低访问EL | EL0(受控) | EL0(受控) | EL2 | EL1(Secure) |
| PPI | 14(NS)/13(S) | 11 | 10 | 13 |
| VHE 重定向 | → CNTHP | → CNTHV | - | - |

---

## 36. 关键数据结构汇总

| 结构/字段 | 位置 | 用途 |
|----------|------|------|
| `ARMGenericTimer` | cpu.h | 每定时器 ctl/cval |
| `c14_timer[NUM_GTIMERS]` | cpu.h:520 | 7 种定时器状态数组 |
| `c14_cntfrq` | cpu.h:515 | 计数频率寄存器 |
| `c14_cntkctl` | cpu.h:516 | EL1 访问控制 |
| `cnthctl_el2` | cpu.h:517 | EL2 访问控制 |
| `cntvoff_el2` | cpu.h:518 | 虚拟计数器偏移 |
| `cntpoff_el2` | cpu.h:519 | 物理计数器偏移(ECV) |
| `gt_timer_outputs[]` | ARMCPU | GPIO 输出到 GIC |
| `gt_timer[]` | ARMCPU | QEMUTimer 实例 |

---

## 37. 总结

QEMU 的 ARM Generic Timer 实现有以下特点：

1. **集成在 CPU 模型中**：不是独立设备，定时器状态是 CPUARMState 的一部分
2. **虚拟时钟驱动**：使用 `QEMU_CLOCK_VIRTUAL` 提供确定性计时
3. **完整的访问控制**：实现了 CNTKCTL/CNTHCTL 所有陷阱位
4. **7 种定时器**：覆盖 EL0-EL3 + VHE + SEL2 全部场景
5. **GPIO 连接 GIC**：通过标准 QEMU GPIO/IRQ 机制连接 PPI
6. **VHE 重定向**：CNTP→CNTHP、CNTV→CNTHV 透明重定向
7. **RME 支持**：CNTVMASK/CNTPMASK 在 Root/Realm 空间的额外掩码
8. **ECV 支持**：物理计数器偏移 CNTPOFF_EL2 和自同步计数器
