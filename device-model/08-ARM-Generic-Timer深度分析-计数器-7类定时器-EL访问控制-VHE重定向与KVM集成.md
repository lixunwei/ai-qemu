# ARM Generic Timer 深度分析：计数器、7类定时器、EL访问控制、VHE重定向与KVM集成

> 基于 QEMU 11.0.50 源码分析，涵盖 ARM 架构通用定时器（Generic Timer）的完整实现：
> 物理/虚拟计数器、7种定时器变体（PHYS/VIRT/HYP/SEC/HYPVIRT/S_EL2_PHYS/S_EL2_VIRT）、
> CTL/CVAL/TVAL 寄存器操作、EL0→EL3 四级访问控制、VHE 定时器重定向、
> 计数器偏移（CNTVOFF/CNTPOFF/ECV）、中断生成与 GIC PPI 接线、
> QEMUTimer 基础设施、virt 机器 DTB 生成，以及 KVM 定时器状态同步。

---

## 目录

1. [架构概述](#1-架构概述)
2. [数据结构](#2-数据结构)
3. [定时器变体枚举](#3-定时器变体枚举)
4. [计数器实现](#4-计数器实现)
5. [定时器频率与时间尺度](#5-定时器频率与时间尺度)
6. [CTL/CVAL/TVAL 寄存器操作](#6-ctlcvaltval-寄存器操作)
7. [VHE 定时器重定向](#7-vhe-定时器重定向)
8. [EL 访问控制体系](#8-el-访问控制体系)
9. [定时器重算与中断生成](#9-定时器重算与中断生成)
10. [QEMUTimer 基础设施](#10-qemutimer-基础设施)
11. [CPU 定时器创建与 GPIO 接线](#11-cpu-定时器创建与-gpio-接线)
12. [virt 机器 GIC 接线与 DTB 生成](#12-virt-机器-gic-接线与-dtb-生成)
13. [EL2/EL3 定时器寄存器](#13-el2el3-定时器寄存器)
14. [迁移与保存恢复](#14-迁移与保存恢复)
15. [KVM 定时器集成](#15-kvm-定时器集成)
16. [WFxT 超时定时器](#16-wfxt-超时定时器)
17. [完整数据流总结](#17-完整数据流总结)

---

## 1. 架构概述

ARM Generic Timer 是 ARMv7/v8 架构标准的定时器机制，为每个 CPU 核心提供：

- **系统计数器**（System Counter）：单调递增的物理时钟源
- **物理/虚拟计数器**（CNTPCT/CNTVCT）：物理计数器和带偏移的虚拟计数器
- **多种定时器**：每种定时器包含 CTL（控制）、CVAL（比较值）、TVAL（倒计时视图）三个寄存器
- **PPI 中断输出**：定时器到期后通过 GIC PPI（Per-Processor Interrupt）通知 CPU

在 QEMU 中，Generic Timer 不是一个独立的设备模型，而是**内嵌在 ARM CPU 对象中**的系统寄存器和 QEMUTimer 回调组合。

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `cpu.h` | 137-140 | `ARMGenericTimer` 结构体 |
| `cpu.h` | 515-520 | CPUARMState 中定时器字段 |
| `cpu.h` | 955-966 | ARMCPU 中 QEMUTimer 和 GPIO |
| `gtimer.h` | 12-20 | 7种定时器变体枚举 |
| `helper.c` | 1106-2230 | 定时器核心逻辑（~1100行） |
| `helper.c` | 4190-4230 | EL2 定时器寄存器 |
| `helper.c` | 5866-5909 | VHE/EL02 定时器寄存器 |
| `cpu.c` | 1847-1863 | QEMUTimer 创建 |
| `cpu.c` | 1206-1207 | GPIO 输出注册 |
| `virt.c` | 447-512 | DTB timer 节点 |
| `virt.c` | 1228-1253 | GIC PPI 接线 |
| `bsa.h` | 25-33 | 定时器 IRQ INTID 常量 |
| `machine.c` | 1293-1294 | 迁移 vmstate |
| `kvm.c` | 1075-1100 | KVM 定时器同步 |

---

## 2. 数据结构

### 2.1 ARMGenericTimer（单个定时器实例）

```c
// cpu.h:137-140
typedef struct ARMGenericTimer {
    uint64_t cval; /* Timer CompareValue register */
    uint64_t ctl;  /* Timer Control register */
} ARMGenericTimer;
```

每个定时器实例仅保存两个字段：
- **cval**：64位比较值，当计数器 ≥ cval 时 ISTATUS 置位
- **ctl**：控制寄存器，bit[0]=ENABLE, bit[1]=IMASK, bit[2]=ISTATUS（只读）

**TVAL 不单独存储**——它是 `cval - counter` 的 32 位截断视图，读写时动态计算。

### 2.2 CPUARMState 中的定时器字段

```c
// cpu.h:515-520
uint64_t c14_cntfrq;                    // 计数器频率寄存器
uint64_t c14_cntkctl;                   // CNTKCTL_EL1 — EL0 访问控制
uint64_t cnthctl_el2;                   // CNTHCTL_EL2 — EL1 访问控制
uint64_t cntvoff_el2;                   // CNTVOFF_EL2 — 虚拟计数器偏移
uint64_t cntpoff_el2;                   // CNTPOFF_EL2 — 物理计数器偏移(ECV)
ARMGenericTimer c14_timer[NUM_GTIMERS]; // 7个定时器实例
```

### 2.3 ARMCPU 中的运行时对象

```c
// cpu.h:955-966
QEMUTimer *gt_timer[NUM_GTIMERS];           // 7个 QEMUTimer 对象
QEMUTimer *wfxt_timer;                      // WFxT 超时定时器
qemu_irq gt_timer_outputs[NUM_GTIMERS];     // 7个 GPIO IRQ 输出
```

---

## 3. 定时器变体枚举

```c
// gtimer.h:12-20
enum {
    GTIMER_PHYS       = 0,  // CNTP_*    EL1 物理定时器
    GTIMER_VIRT       = 1,  // CNTV_*    EL1 虚拟定时器
    GTIMER_HYP        = 2,  // CNTHP_*   EL2 物理定时器
    GTIMER_SEC        = 3,  // CNTPS_*   EL3 安全物理定时器
    GTIMER_HYPVIRT    = 4,  // CNTHV_*   EL2 虚拟定时器 (FEAT_VHE)
    GTIMER_S_EL2_PHYS = 5,  // CNTHPS_*  安全EL2 物理 (FEAT_SEL2)
    GTIMER_S_EL2_VIRT = 6,  // CNTHVS_*  安全EL2 虚拟 (FEAT_SEL2)
#define NUM_GTIMERS   7
};
```

### 定时器变体对照表

| 索引 | 寄存器前缀 | 所属 EL | 特性 | 使用偏移 |
|------|-----------|---------|------|---------|
| 0 PHYS | CNTP_* | EL1 | 标准物理定时器 | CNTPOFF_EL2 (ECV) |
| 1 VIRT | CNTV_* | EL1 | 虚拟定时器 | CNTVOFF_EL2 |
| 2 HYP | CNTHP_* | EL2 | Hypervisor 物理定时器 | 无 |
| 3 SEC | CNTPS_* | EL3 | Secure 物理定时器 | 无 |
| 4 HYPVIRT | CNTHV_* | EL2 | Hypervisor 虚拟定时器 (VHE) | 无 |
| 5 S_EL2_PHYS | CNTHPS_* | S-EL2 | Secure EL2 物理 (SEL2) | 无 |
| 6 S_EL2_VIRT | CNTHVS_* | S-EL2 | Secure EL2 虚拟 (SEL2) | 无 |

---

## 4. 计数器实现

### 4.1 物理计数器读取

```c
// helper.c:1339-1343
uint64_t gt_get_countervalue(CPUARMState *env)
{
    ARMCPU *cpu = env_archcpu(env);
    return qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) / gt_cntfrq_period_ns(cpu);
}
```

**关键设计**：QEMU 不维护独立的 counter 变量，而是直接用 `QEMU_CLOCK_VIRTUAL`（虚拟时钟的纳秒值）除以每 tick 的纳秒周期，得到当前计数值。这保证了计数器与 QEMU 时钟完全同步。

### 4.2 CNTPCT_EL0 读取（物理计数器寄存器）

```c
// helper.c:1536-1539
static uint64_t gt_cnt_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_PHYS);
    return gt_get_countervalue(env) - offset;
}
```

物理计数器可能带 CNTPOFF_EL2 偏移（FEAT_ECV），仅在满足条件时生效。

### 4.3 CNTVCT_EL0 读取（虚拟计数器寄存器）

```c
// helper.c:1542-1545
static uint64_t gt_virt_cnt_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_VIRT);
    return gt_get_countervalue(env) - offset;
}
```

虚拟计数器 = 物理计数器 - CNTVOFF_EL2（由 Hypervisor 设置的偏移）。

### 4.4 直接访问偏移计算

```c
// helper.c:1418-1464
uint64_t gt_direct_access_timer_offset(CPUARMState *env, int timeridx)
{
    switch (timeridx) {
    case GTIMER_PHYS:
        if (arm_current_el(env) >= 2) return 0;       // EL2+ 不应用偏移
        return gt_phys_raw_cnt_offset(env);             // EL0/EL1 应用 CNTPOFF
    case GTIMER_VIRT:
        switch (arm_current_el(env)) {
        case 2:
            if (hcr & HCR_E2H) return 0;               // VHE Host 不偏移
            break;
        case 0:
            if ((hcr & (HCR_E2H|HCR_TGE)) == ...) return 0; // VHE Host EL0
            break;
        }
        return env->cp15.cntvoff_el2;                   // 其他情况应用 CNTVOFF
    case GTIMER_HYP/SEC/HYPVIRT/S_EL2_*:
        return 0;                                        // EL2/EL3 定时器无偏移
    }
}
```

### 4.5 间接访问偏移（用于 CompareValue 计算）

```c
// helper.c:1390-1416
static uint64_t gt_indirect_access_timer_offset(CPUARMState *env, int timeridx)
{
    switch (timeridx) {
    case GTIMER_PHYS:  return gt_phys_raw_cnt_offset(env);
    case GTIMER_VIRT:  return env->cp15.cntvoff_el2;
    default:           return 0;
    }
}
```

间接访问偏移不考虑当前 EL，始终对 PHYS 用 CNTPOFF、VIRT 用 CNTVOFF。这用于 `gt_recalc_timer()` 计算 ISTATUS 和下次触发时间。

### 4.6 FEAT_ECV 物理计数器偏移

```c
// helper.c:1379-1388
static uint64_t gt_phys_raw_cnt_offset(CPUARMState *env)
{
    if ((env->cp15.scr_el3 & SCR_ECVEN) &&              // SCR_EL3.ECVEN=1
        FIELD_EX64(env->cp15.cnthctl_el2, CNTHCTL, ECV) && // CNTHCTL.ECV=1
        arm_is_el2_enabled(env) &&                        // EL2 使能
        !((hcr & (HCR_E2H|HCR_TGE)) == (HCR_E2H|HCR_TGE))) // 非 VHE Host
    {
        return env->cp15.cntpoff_el2;
    }
    return 0;
}
```

FEAT_ECV（Enhanced Counter Virtualization）允许 Hypervisor 对物理计数器也施加偏移，需要 SCR_EL3.ECVEN 和 CNTHCTL_EL2.ECV 同时使能。

---

## 5. 定时器频率与时间尺度

### 5.1 频率常量

```c
// internals.h:85-86
#define GTIMER_DEFAULT_HZ    1000000000   // 1GHz（新机器）
#define GTIMER_BACKCOMPAT_HZ 62500000     // 62.5MHz（旧兼容）
```

### 5.2 频率选择逻辑

```c
// cpu.c:1840-1844
if (cpu->backcompat_cntfrq) {
    cpu->gt_cntfrq_hz = GTIMER_BACKCOMPAT_HZ;   // 62.5MHz
} else {
    cpu->gt_cntfrq_hz = GTIMER_DEFAULT_HZ;      // 1GHz
}
```

### 5.3 周期计算

```c
// cpu.c:1364-1390
unsigned int gt_cntfrq_period_ns(ARMCPU *cpu)
{
    // 使用整数除法截断，保证与 QEMUTimer scale 精确互逆
    return NANOSECONDS_PER_SECOND / cpu->gt_cntfrq_hz;
}
```

对于 1GHz 频率：`period_ns = 1`（每纳秒一个 tick）。
对于 62.5MHz 频率：`period_ns = 16`（每 16ns 一个 tick）。

### 5.4 CNTFRQ_EL0 重置

```c
// helper.c:1106-1111
static void arm_gt_cntfrq_reset(CPUARMState *env, const ARMCPRegInfo *ri)
{
    ARMCPU *cpu = env_archcpu(env);
    cpu->env.cp15.c14_cntfrq = cpu->gt_cntfrq_hz;
}
```

CNTFRQ_EL0 是纯软件可读写的寄存器，写入不改变实际频率，仅供 Guest OS 查询。

---

## 6. CTL/CVAL/TVAL 寄存器操作

### 6.1 CTL 写入

```c
// helper.c:1589-1609
static void gt_ctl_write(CPUARMState *env, const ARMCPRegInfo *ri,
                         int timeridx, uint64_t value)
{
    uint32_t oldval = env->cp15.c14_timer[timeridx].ctl;
    env->cp15.c14_timer[timeridx].ctl = deposit64(oldval, 0, 2, value);
    
    if ((oldval ^ value) & 1) {
        gt_recalc_timer(cpu, timeridx);   // ENABLE 翻转 → 重算
    } else if ((oldval ^ value) & 2) {
        gt_update_irq(cpu, timeridx);     // IMASK 翻转 → 更新 IRQ
    }
}
```

**CTL 位域**：
- bit[0] **ENABLE**：定时器使能
- bit[1] **IMASK**：中断屏蔽（1=屏蔽中断输出）
- bit[2] **ISTATUS**：只读，当 counter ≥ cval 时为 1

### 6.2 CVAL 写入

```c
// helper.c:1548-1555
static void gt_cval_write(CPUARMState *env, const ARMCPRegInfo *ri,
                          int timeridx, uint64_t value)
{
    env->cp15.c14_timer[timeridx].cval = value;
    gt_recalc_timer(env_archcpu(env), timeridx);
}
```

### 6.3 TVAL 读写

```c
// helper.c:1557-1561 (读)
static uint64_t do_tval_read(CPUARMState *env, int timeridx, uint64_t offset)
{
    return (uint32_t)(env->cp15.c14_timer[timeridx].cval -
                      (gt_get_countervalue(env) - offset));
}

// helper.c:1571-1577 (写)
static void do_tval_write(CPUARMState *env, int timeridx, uint64_t value,
                          uint64_t offset)
{
    env->cp15.c14_timer[timeridx].cval = gt_get_countervalue(env) - offset +
                                         sextract64(value, 0, 32);
    gt_recalc_timer(env_archcpu(env), timeridx);
}
```

TVAL 是 32 位有符号的"距离到期还有多少 tick"视图：
- **读**：`cval - (counter - offset)`，截断为 32 位
- **写**：将 TVAL 转换为绝对 CVAL = `(counter - offset) + sign_extend(value)`

---

## 7. VHE 定时器重定向

VHE（Virtualization Host Extensions）模式下，EL2 Host 使用 EL1 的定时器寄存器名称（CNTP_*/CNTV_*），但实际操作的是 EL2 的定时器（CNTHP_*/CNTHV_*）。

### 7.1 重定向索引函数

```c
// helper.c:1639-1661
static int gt_phys_redir_timeridx(CPUARMState *env)
{
    switch (arm_mmu_idx(env)) {
    case ARMMMUIdx_E20_0:       // VHE Host EL0
    case ARMMMUIdx_E20_2:       // VHE Host EL2
    case ARMMMUIdx_E20_2_PAN:   // VHE Host EL2 + PAN
        return GTIMER_HYP;      // 重定向到 EL2 物理定时器
    default:
        return GTIMER_PHYS;     // 非 VHE → EL1 物理定时器
    }
}

static int gt_virt_redir_timeridx(CPUARMState *env)
{
    switch (arm_mmu_idx(env)) {
    case ARMMMUIdx_E20_0/E20_2/E20_2_PAN:
        return GTIMER_HYPVIRT;  // 重定向到 EL2 虚拟定时器
    default:
        return GTIMER_VIRT;     // 非 VHE → EL1 虚拟定时器
    }
}
```

### 7.2 重定向 read/write 函数

以物理定时器 CTL 为例：

```c
// helper.c:1691-1703
static uint64_t gt_phys_redir_ctl_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    int timeridx = gt_phys_redir_timeridx(env);
    return env->cp15.c14_timer[timeridx].ctl;
}

static void gt_phys_redir_ctl_write(CPUARMState *env, const ARMCPRegInfo *ri,
                                    uint64_t value)
{
    int timeridx = gt_phys_redir_timeridx(env);
    gt_ctl_write(env, ri, timeridx, value);
}
```

### 7.3 重定向效果

| 条件 | CNTP_CTL_EL0 访问的实际定时器 | CNTV_CTL_EL0 访问的实际定时器 |
|------|------|------|
| 非 VHE (E2H=0) | GTIMER_PHYS (EL1 物理) | GTIMER_VIRT (EL1 虚拟) |
| VHE Host (E2H=1, EL2) | GTIMER_HYP (EL2 物理) | GTIMER_HYPVIRT (EL2 虚拟) |
| VHE Guest (E2H=1, EL0+TGE) | GTIMER_HYP (EL2 物理) | GTIMER_HYPVIRT (EL2 虚拟) |

### 7.4 EL02 编码（访问原始 EL1 定时器）

VHE 模式下，EL2 若需访问原始 EL1 定时器，使用 `_EL02` 后缀编码：

```c
// helper.c:5872-5909
{ .name = "CNTP_CTL_EL02",  opc1=5, → c14_timer[GTIMER_PHYS].ctl }
{ .name = "CNTV_CTL_EL02",  opc1=5, → c14_timer[GTIMER_VIRT].ctl }
{ .name = "CNTP_TVAL_EL02", opc1=5, → gt_phys_tval_read/write }
{ .name = "CNTV_TVAL_EL02", opc1=5, → gt_virt_tval_read/write }
{ .name = "CNTP_CVAL_EL02", opc1=5, → c14_timer[GTIMER_PHYS].cval }
{ .name = "CNTV_CVAL_EL02", opc1=5, → c14_timer[GTIMER_VIRT].cval }
```

这些 `_EL02` 寄存器绕过 VHE 重定向，直接操作 EL1 定时器实例。

---

## 8. EL 访问控制体系

Generic Timer 有精密的四级访问控制：EL0→EL1→EL2→EL3。

### 8.1 CNTFRQ_EL0 访问控制

```c
// helper.c:1115-1155 gt_cntfrq_access()
```

- EL0：需要 CNTKCTL.EL0PCTEN 或 EL0VCTEN（任一为 1 即可）
- EL1+：总是可访问
- 写入：仅最高实现 EL 可写

### 8.2 计数器访问控制（CNTPCT/CNTVCT）

```c
// helper.c:1157-1193 gt_counter_access()
switch (cur_el) {
case 0:
    // VHE Host EL0 (E2H+TGE=11)：检查 CNTHCTL.EL0[PV]CTEN
    // 普通 EL0：检查 CNTKCTL.EL0[PV]CTEN → TRAP_EL1
case 1:
    // 检查 CNTHCTL_EL2.EL1PCTEN (CNTPCT)
    // 检查 CNTHCTL_EL2.EL1TVCT (CNTVCT) → TRAP_EL2
}
```

### 8.3 定时器寄存器访问控制（CNTP_*/CNTV_*）

```c
// helper.c:1195-1241 gt_timer_access()
switch (cur_el) {
case 0:
    // VHE Host EL0 (E2H+TGE=11)：CNTHCTL.EL0[PV]TEN → TRAP_EL2
    // 普通 EL0：CNTKCTL.EL0[PV]TEN → TRAP_EL1
case 1:
    // CNTP: E2H ? CNTHCTL.EL1PTEN : CNTHCTL.EL1PCEN → TRAP_EL2
    // CNTV: CNTHCTL.EL1TVT → TRAP_EL2
}
```

### 8.4 安全定时器访问控制（CNTPS_*）

```c
// helper.c:1269-1298 gt_stimer_access()
switch (arm_current_el(env)) {
case 3: return OK;                    // EL3 总是可访问
case 1:
    if (!secure) return UNDEFINED;     // 非安全态不可见
    if (el2_enabled) return UNDEFINED; // 有 SEL2 时不可见
    if (!(SCR_EL3 & SCR_ST)) return TRAP_EL3; // SCR_EL3.ST 门控
    return OK;
case 0/2: return UNDEFINED;           // EL0/EL2 不可见
}
```

### 8.5 安全 EL2 定时器访问控制（CNTHPS_*/CNTHVS_*）

```c
// helper.c:1300-1337 gt_sel2timer_access()
switch (arm_current_el(env)) {
case 0: return UNDEFINED;
case 1:
    if (!secure) return UNDEFINED;
    if (HCR_NV) return TRAP_EL2;       // NV 嵌套虚拟化陷阱
    return UNDEFINED;
case 2:
    if (!secure) return UNDEFINED;
    return OK;                          // Secure EL2 可访问
case 3:
    if (SCR_EEL2) return OK;            // 有 SEL2 时 EL3 可访问
    return UNDEFINED;
}
```

### 8.6 访问控制总结表

| 寄存器 | EL0 控制位 | EL1→EL2 陷阱位 | EL1/2→EL3 陷阱位 |
|--------|-----------|----------------|-----------------|
| CNTPCT | CNTKCTL.EL0PCTEN | CNTHCTL.EL1PCTEN | — |
| CNTVCT | CNTKCTL.EL0VCTEN | CNTHCTL.EL1TVCT | — |
| CNTP_* | CNTKCTL.EL0PTEN | CNTHCTL.EL1PTEN/EL1PCEN | — |
| CNTV_* | CNTKCTL.EL0VTEN | CNTHCTL.EL1TVT | — |
| CNTPS_* | — | — | SCR_EL3.ST |
| CNTHPS/VS_* | — | HCR_NV (NV陷阱) | SCR_EL3.EEL2 |

---

## 9. 定时器重算与中断生成

### 9.1 gt_recalc_timer — 核心调度函数

```c
// helper.c:1466-1526
static void gt_recalc_timer(ARMCPU *cpu, int timeridx)
{
    ARMGenericTimer *gt = &cpu->env.cp15.c14_timer[timeridx];

    if (gt->ctl & 1) {  // ENABLE=1
        uint64_t offset = gt_indirect_access_timer_offset(&cpu->env, timeridx);
        uint64_t count = gt_get_countervalue(&cpu->env);
        int istatus = count - offset >= gt->cval;   // 无符号比较
        gt->ctl = deposit32(gt->ctl, 2, 1, istatus); // 更新 ISTATUS

        if (istatus) {
            // 已到期：下次触发是 counter 回绕时
            nexttick = (offset > count) ? offset : UINT64_MAX;
        } else {
            // 未到期：下次触发 = cval + offset（可能溢出）
            if (uadd64_overflow(gt->cval, offset, &nexttick))
                nexttick = UINT64_MAX;
        }

        // 调度 QEMUTimer
        if (nexttick > INT64_MAX / gt_cntfrq_period_ns(cpu)) {
            timer_mod_ns(cpu->gt_timer[timeridx], INT64_MAX);
        } else {
            timer_mod(cpu->gt_timer[timeridx], nexttick);
        }
    } else {  // ENABLE=0
        gt->ctl &= ~4;                              // 清除 ISTATUS
        timer_del(cpu->gt_timer[timeridx]);          // 取消定时器
    }
    gt_update_irq(cpu, timeridx);                    // 更新 IRQ 输出
}
```

### 9.2 gt_update_irq — IRQ 输出更新

```c
// helper.c:1346-1365
static void gt_update_irq(ARMCPU *cpu, int timeridx)
{
    int irqstate = (env->cp15.c14_timer[timeridx].ctl & 6) == 4;
    // IRQ = ISTATUS(bit2)=1 && IMASK(bit1)=0

    // RME: Root/Realm 安全域下 CNTHCTL 的 CNTVMASK/CNTPMASK 可覆盖
    if ((ss == ARMSS_Root || ss == ARMSS_Realm) &&
        ((timeridx == GTIMER_VIRT && (cnthctl & CNTVMASK)) ||
         (timeridx == GTIMER_PHYS && (cnthctl & CNTPMASK)))) {
        irqstate = 0;
    }

    qemu_set_irq(cpu->gt_timer_outputs[timeridx], irqstate);
}
```

### 9.3 EL 切换时的 IRQ 更新

```c
// helper.c:1368-1377
void gt_rme_post_el_change(ARMCPU *cpu, void *ignored)
{
    // Root↔Secure/NonSecure 切换可改变 CNTHCTL mask 有效值
    gt_update_irq(cpu, GTIMER_VIRT);
    gt_update_irq(cpu, GTIMER_PHYS);
}
```

### 9.4 定时器回调函数

```c
// helper.c:1976-2023
void arm_gt_ptimer_cb(void *opaque)  { gt_recalc_timer(cpu, GTIMER_PHYS); }
void arm_gt_vtimer_cb(void *opaque)  { gt_recalc_timer(cpu, GTIMER_VIRT); }
void arm_gt_htimer_cb(void *opaque)  { gt_recalc_timer(cpu, GTIMER_HYP); }
void arm_gt_stimer_cb(void *opaque)  { gt_recalc_timer(cpu, GTIMER_SEC); }
void arm_gt_hvtimer_cb(void *opaque) { gt_recalc_timer(cpu, GTIMER_HYPVIRT); }
void arm_gt_sel2timer_cb(void *opaque) { gt_recalc_timer(cpu, GTIMER_S_EL2_PHYS); }
void arm_gt_sel2vtimer_cb(void *opaque) { gt_recalc_timer(cpu, GTIMER_S_EL2_VIRT); }
```

每个回调仅调用 `gt_recalc_timer()`，重新计算 ISTATUS 并更新 IRQ。

---

## 10. QEMUTimer 基础设施

### 10.1 QEMUTimer 结构

```c
// include/qemu/timer.h:76-93
struct QEMUTimer {
    int64_t expire_time;    // 到期时间
    QEMUTimerList *timer_list;
    QEMUTimerCB *cb;        // 到期回调
    void *opaque;
    QEMUTimer *next;
    int attributes;
    int scale;              // 时间尺度因子（纳秒/tick）
};
```

### 10.2 关键 API

| 函数 | 用途 | ARM Timer 使用 |
|------|------|---------------|
| `timer_new(clock, scale, cb, opaque)` | 创建定时器 | cpu.c:1850-1863 |
| `timer_mod(timer, tick_count)` | 设置到期（tick 单位） | helper.c:1516 |
| `timer_mod_ns(timer, ns)` | 设置到期（纳秒单位） | helper.c:1514 |
| `timer_del(timer)` | 取消定时器 | helper.c:1522 |

ARM Generic Timer 使用 `QEMU_CLOCK_VIRTUAL`（虚拟时钟），scale 为 `gt_cntfrq_period_ns(cpu)`。

---

## 11. CPU 定时器创建与 GPIO 接线

### 11.1 QEMUTimer 创建

```c
// cpu.c:1847-1863
uint64_t scale = gt_cntfrq_period_ns(cpu);

cpu->gt_timer[GTIMER_PHYS]      = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_ptimer_cb, cpu);
cpu->gt_timer[GTIMER_VIRT]      = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_vtimer_cb, cpu);
cpu->gt_timer[GTIMER_HYP]       = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_htimer_cb, cpu);
cpu->gt_timer[GTIMER_SEC]       = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_stimer_cb, cpu);
cpu->gt_timer[GTIMER_HYPVIRT]   = timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_hvtimer_cb, cpu);
cpu->gt_timer[GTIMER_S_EL2_PHYS]= timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_sel2timer_cb, cpu);
cpu->gt_timer[GTIMER_S_EL2_VIRT]= timer_new(QEMU_CLOCK_VIRTUAL, scale, arm_gt_sel2vtimer_cb, cpu);
```

每个 CPU 核心创建 7 个 QEMUTimer，共享同一 scale。

### 11.2 GPIO 输出注册

```c
// cpu.c:1206-1207
qdev_init_gpio_out(DEVICE(cpu), cpu->gt_timer_outputs,
                   ARRAY_SIZE(cpu->gt_timer_outputs));  // 7个输出
```

---

## 12. virt 机器 GIC 接线与 DTB 生成

### 12.1 IRQ INTID 常量

```c
// include/hw/arm/bsa.h:25-33
#define ARCH_TIMER_S_EL2_VIRT_IRQ  19   // PPI 3
#define ARCH_TIMER_S_EL2_IRQ       20   // PPI 4
#define ARCH_TIMER_NS_EL2_IRQ      26   // PPI 10
#define ARCH_TIMER_VIRT_IRQ        27   // PPI 11
#define ARCH_TIMER_NS_EL2_VIRT_IRQ 28   // PPI 12
#define ARCH_TIMER_S_EL1_IRQ       29   // PPI 13
#define ARCH_TIMER_NS_EL1_IRQ      30   // PPI 14
```

### 12.2 GIC PPI 接线

```c
// virt.c:1228-1253
for (i = 0; i < smp_cpus; i++) {
    const int timer_irq[] = {
        [GTIMER_PHYS]       = ARCH_TIMER_NS_EL1_IRQ,     // PPI 14
        [GTIMER_VIRT]       = ARCH_TIMER_VIRT_IRQ,        // PPI 11
        [GTIMER_HYP]        = ARCH_TIMER_NS_EL2_IRQ,     // PPI 10
        [GTIMER_SEC]        = ARCH_TIMER_S_EL1_IRQ,       // PPI 13
        [GTIMER_HYPVIRT]    = ARCH_TIMER_NS_EL2_VIRT_IRQ, // PPI 12
        [GTIMER_S_EL2_PHYS] = ARCH_TIMER_S_EL2_IRQ,      // PPI 4
        [GTIMER_S_EL2_VIRT] = ARCH_TIMER_S_EL2_VIRT_IRQ, // PPI 3
    };
    for (irq = 0; irq < ARRAY_SIZE(timer_irq); irq++) {
        qdev_connect_gpio_out(cpudev, irq,
            qdev_get_gpio_in(vms->gic, intidbase + timer_irq[irq]));
    }
}
```

每个 CPU 的 7 个定时器输出连接到对应的 GIC PPI 输入。

### 12.3 DTB timer 节点生成

```c
// virt.c:447-512
static void fdt_add_timer_nodes(const VirtMachineState *vms)
{
    qemu_fdt_add_subnode(ms->fdt, "/timer");

    // compatible = "arm,armv8-timer" + "arm,armv7-timer"
    qemu_fdt_setprop(ms->fdt, "/timer", "compatible", ...);
    qemu_fdt_setprop(ms->fdt, "/timer", "always-on", NULL, 0);

    // interrupts 属性：4或5个 PPI（取决于是否有 NS EL2 VIRT timer）
    qemu_fdt_setprop_cells(ms->fdt, "/timer", "interrupts",
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_S_EL1_IRQ),   irqflags,
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_NS_EL1_IRQ),  irqflags,
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_VIRT_IRQ),    irqflags,
        GIC_FDT_IRQ_TYPE_PPI, INTID_TO_PPI(ARCH_TIMER_NS_EL2_IRQ),  irqflags,
        // 可选：ARCH_TIMER_NS_EL2_VIRT_IRQ
    );
}
```

DTB 的 `INTID_TO_PPI(irq)` 宏将 INTID 转换为 PPI 编号（减 16）。

---

## 13. EL2/EL3 定时器寄存器

### 13.1 EL2 定时器（CNTHCTL/CNTVOFF/CNTHP_*）

```c
// helper.c:4190-4230
{ .name = "CNTHCTL_EL2", opc1=4, crn=14, crm=1, opc2=0,
  .access = PL2_RW, .type = ARM_CP_IO, .resetvalue = 3,
  .writefn = gt_cnthctl_write,
  .fieldoffset = offsetof(CPUARMState, cp15.cnthctl_el2) },

{ .name = "CNTVOFF_EL2", opc1=4, crn=14, crm=0, opc2=3,
  .access = PL2_RW, .type = ARM_CP_IO, .resetvalue = 0,
  .writefn = gt_cntvoff_write,   // 写入时可能需要重算定时器
  .nv2_redirect_offset = 0x60,
  .fieldoffset = offsetof(CPUARMState, cp15.cntvoff_el2) },

{ .name = "CNTHP_CTL_EL2", opc1=4, crn=14, crm=2, opc2=1,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_timer[GTIMER_HYP].ctl),
  .writefn = gt_hyp_ctl_write },

{ .name = "CNTHP_CVAL_EL2", opc1=4, crn=14, crm=2, opc2=2,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_timer[GTIMER_HYP].cval),
  .writefn = gt_hyp_cval_write },

{ .name = "CNTHP_TVAL_EL2", opc1=4, crn=14, crm=2, opc2=0,
  .readfn = gt_hyp_tval_read, .writefn = gt_hyp_tval_write },
```

### 13.2 VHE 虚拟 Hypervisor 定时器（CNTHV_*）

```c
// helper.c:5866-5871
{ .name = "CNTHV_CTL_EL2", opc1=4, crn=14, crm=3, opc2=1,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_timer[GTIMER_HYPVIRT].ctl),
  .writefn = gt_hv_ctl_write },
```

### 13.3 安全定时器（CNTPS_*）

```c
// helper.c:2200-2230
{ .name = "CNTPS_TVAL_EL1", opc1=7, crn=14, crm=2, opc2=0,
  .accessfn = gt_stimer_access,
  .readfn = gt_sec_tval_read, .writefn = gt_sec_tval_write },

{ .name = "CNTPS_CTL_EL1", opc1=7, crn=14, crm=2, opc2=1,
  .accessfn = gt_stimer_access,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_timer[GTIMER_SEC].ctl) },

{ .name = "CNTPS_CVAL_EL1", opc1=7, crn=14, crm=2, opc2=2,
  .accessfn = gt_stimer_access,
  .fieldoffset = offsetof(CPUARMState, cp15.c14_timer[GTIMER_SEC].cval) },
```

---

## 14. 迁移与保存恢复

### 14.1 VMState 定义

```c
// machine.c:1293-1294
VMSTATE_TIMER_PTR(gt_timer[GTIMER_PHYS], ARMCPU),
VMSTATE_TIMER_PTR(gt_timer[GTIMER_VIRT], ARMCPU),
```

**仅 PHYS 和 VIRT 两个 QEMUTimer 被包含在主 vmstate 中**。其他定时器（HYP/SEC/HYPVIRT/S_EL2_*）的状态通过 cpreg 列表保存（c14_timer[].ctl/cval），在迁移后目标端会通过寄存器写入路径重新调度 QEMUTimer。

### 14.2 定时器相关的 cpreg 字段迁移

CPUARMState 中的以下字段在 cpreg 框架内自动迁移：
- `c14_cntfrq`：CNTFRQ_EL0
- `c14_cntkctl`：CNTKCTL_EL1
- `cnthctl_el2`：CNTHCTL_EL2
- `cntvoff_el2`：CNTVOFF_EL2
- `cntpoff_el2`：CNTPOFF_EL2
- `c14_timer[0..6].ctl/cval`：各定时器控制和比较值

---

## 15. KVM 定时器集成

### 15.1 KVM 定时器状态同步

```c
// kvm.c:1075-1081
void kvm_arm_cpu_pre_save(ARMCPU *cpu)
{
    if (cpu->kvm_vtime_dirty) {
        *kvm_arm_get_cpreg_ptr(cpu, KVM_REG_ARM_TIMER_CNT) = cpu->kvm_vtime;
    }
}
```

迁移前保存 KVM 虚拟时间计数器。

```c
// kvm.c:1083-1101
bool kvm_arm_cpu_post_load(ARMCPU *cpu)
{
    write_list_to_kvmstate(cpu, KVM_PUT_FULL_STATE);
    write_list_to_cpustate(cpu);

    if (cpu->kvm_adjvtime) {
        cpu->kvm_vtime = *kvm_arm_get_cpreg_ptr(cpu, KVM_REG_ARM_TIMER_CNT);
        cpu->kvm_vtime_dirty = true;
    }
    return true;
}
```

迁移后恢复：通过 `KVM_SET_ONE_REG` 将所有寄存器写回 KVM。

### 15.2 KVM vs TCG 定时器差异

| 方面 | TCG | KVM |
|------|-----|-----|
| 计数器源 | QEMU_CLOCK_VIRTUAL | 硬件物理计数器 |
| 定时器调度 | QEMUTimer + 回调 | KVM 内核定时器 |
| 中断注入 | qemu_set_irq → GIC 模型 | KVM 直接注入 vIRQ |
| 偏移控制 | 软件模拟 CNTVOFF | KVM 硬件 CNTVOFF |
| 接线方式 | 相同：CPU GPIO → GIC PPI | 相同（board level） |

---

## 16. WFxT 超时定时器

FEAT_WFxT（WFE/WFI with Timeout）使用独立的 `wfxt_timer`：

```c
// op_helper.c:430-469
uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_VIRT);
uint64_t cntvct = cntval - offset;

if (cpu_has_work(cs) || cntvct >= timeout) {
    return;  // 已有工作或已超时，不进入低功耗
}

// 设置超时定时器
timer_mod(cpu->wfxt_timer, nexttick);
cs->halted = 1;
cpu_loop_exit(cs);
```

WFxT 使用虚拟计数器语义计算超时，到期后唤醒 CPU。

---

## 17. 完整数据流总结

### 定时器编程到中断触发的完整路径

```
Guest 写入 CNTP_CTL_EL0 (ENABLE=1)
    │
    ├── VHE 重定向检查
    │   gt_phys_redir_timeridx() → GTIMER_PHYS 或 GTIMER_HYP
    │
    ├── gt_ctl_write()
    │   └── gt_recalc_timer(cpu, timeridx)
    │       ├── count = qemu_clock_get_ns() / period_ns
    │       ├── offset = gt_indirect_access_timer_offset()
    │       ├── istatus = (count - offset >= cval)
    │       ├── 更新 CTL.ISTATUS
    │       ├── timer_mod(gt_timer[idx], nexttick) ← 调度 QEMUTimer
    │       └── gt_update_irq()
    │           └── qemu_set_irq(gt_timer_outputs[idx], irqstate)
    │               └── GIC PPI 输入
    │
    ╰── [QEMUTimer 到期]
        └── arm_gt_ptimer_cb()
            └── gt_recalc_timer() → gt_update_irq()
                └── qemu_set_irq() → GIC → CPU_INTERRUPT_HARD
                    └── cpu_handle_interrupt() → arm_cpu_exec_interrupt()
```

### 接线拓扑

```
┌─────────────────────────────────────────────────────┐
│  ARM CPU                                             │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ PHYS(0)  │  │ VIRT(1)  │  │ HYP(2)   │  ...×7   │
│  │ ctl/cval │  │ ctl/cval │  │ ctl/cval │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │              │              │                 │
│  QEMUTimer[0]   QEMUTimer[1]  QEMUTimer[2]   ...    │
│       │              │              │                 │
│  gt_timer_outputs[0] [1]           [2]        ...    │
└───────┼──────────────┼──────────────┼────────────────┘
        │              │              │
    PPI 30         PPI 27         PPI 26    (INTID)
        │              │              │
┌───────┴──────────────┴──────────────┴────────────────┐
│                    GIC (Redistributor)                │
│                  Per-CPU PPI 处理                      │
└──────────────────────────────────────────────────────┘
```

---

## 交叉参考

- [00-ARM64-CPU-GICv3-TCG深度分析](../arm64/00-ARM64-CPU-GICv3-TCG深度分析.md) — CPU 模型与 GIC 架构
- [24-GICv3完整中断生命周期深度分析](../arm64/24-GICv3完整中断生命周期深度分析.md) — PPI 在 GIC 中的处理
- [40-ARM64-EL1-EL2交互深度分析](../arm64/40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md) — VHE 重定向机制
- [39-ARM64-EL3-Secure世界切换深度分析](../arm64/39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md) — 安全定时器 SCR_EL3.ST

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
