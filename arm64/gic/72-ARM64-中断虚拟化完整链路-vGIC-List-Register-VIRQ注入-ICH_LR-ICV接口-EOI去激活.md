# ARM64 中断虚拟化完整链路分析：vGIC List Register → VIRQ 注入

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/72 |
| 分析对象 | GICv3 虚拟化接口完整数据路径 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM IHI 0069H §8 (Virtual CPU Interface), ARM DDI 0487 §G8 |
| 关键数据结构 | ICH_LR_EL2, ICH_VMCR_EL2, ICH_HCR_EL2 |
| 核心结论 | **QEMU 完整实现 GICv3 虚拟化接口，包括 List Register 管理、优先级抢占、HW bit 物理-虚拟联动、EOI/DIR 去激活** |

---

## 1. 中断虚拟化架构概述

### 1.1 物理 vs 虚拟 CPU Interface

```
┌─────────────────────────────────────────────────────────────┐
│                      EL2 (Hypervisor)                        │
│                                                             │
│  ICC_* 寄存器 (物理)     ICH_* 寄存器 (Hypervisor 控制)       │
│  ├── ICC_IAR1_EL1        ├── ICH_LR<n>_EL2 (List Registers) │
│  ├── ICC_EOIR1_EL1       ├── ICH_HCR_EL2 (控制)             │
│  ├── ICC_PMR_EL1         ├── ICH_VMCR_EL2 (虚拟机控制)       │
│  └── ICC_IGRPEN1_EL1    └── ICH_VTR_EL2 (类型)             │
├─────────────────────────────────────────────────────────────┤
│                      EL1 (Guest OS)                          │
│                                                             │
│  ICV_* 寄存器 (虚拟, trap 到 EL2 或 QEMU 直接模拟)           │
│  ├── ICV_IAR1_EL1 → Guest 读取中断 ID                       │
│  ├── ICV_EOIR1_EL1 → Guest EOI                              │
│  ├── ICV_PMR_EL1 → Guest 优先级 mask                        │
│  └── ICV_IGRPEN1_EL1 → Guest 中断组使能                     │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 关键设计原则

ARM GICv3 虚拟化的核心思想：

1. **Hypervisor 填写 List Register**：将要注入 Guest 的中断信息写入 ICH_LR<n>_EL2
2. **硬件自动呈现虚拟中断**：Guest 读 ICV_IAR 时，自动从 List Register 中选择最高优先级
3. **透明性**：Guest 不知道自己在使用虚拟接口（ICV vs ICC 由 HCR_EL2.IMO/FMO 控制）

---

## 2. List Register (ICH_LR_EL2) 结构

### 2.1 位域定义

```c
// hw/intc/gicv3_internal.h:239
#define ICH_LR_EL2_VINTID_SHIFT     0     // [31:0] 虚拟中断 ID
#define ICH_LR_EL2_VINTID_LENGTH    32
#define ICH_LR_EL2_PINTID_SHIFT     32    // [41:32] 物理中断 ID (HW=1 时有效)
#define ICH_LR_EL2_PINTID_LENGTH    10
#define ICH_LR_EL2_EOI              (1ULL << 41)  // EOI 维护中断使能
#define ICH_LR_EL2_PRIORITY_SHIFT   48    // [55:48] 优先级
#define ICH_LR_EL2_PRIORITY_LENGTH  8
#define ICH_LR_EL2_NMI             (1ULL << 59)  // NMI 标记 (FEAT_GICv3_NMI)
#define ICH_LR_EL2_GROUP           (1ULL << 60)  // 0=Group0(FIQ), 1=Group1(IRQ)
#define ICH_LR_EL2_HW              (1ULL << 61)  // 硬件中断映射
#define ICH_LR_EL2_STATE_SHIFT     62    // [63:62] 状态机
#define ICH_LR_EL2_STATE_LENGTH    2
```

### 2.2 List Register 状态机

```
             ICH_LR_EL2_STATE:
             00 = Invalid (空闲)
             01 = Pending (等待 Guest ACK)
             10 = Active (Guest 已 ACK，正在处理)
             11 = Active+Pending (重新 pending)

                    ┌──────────┐
         写入 LR    │ Invalid  │  EOI/Deactivate
        ─────────→  │   (00)   │  ←──────────────
        │           └─────┬────┘              │
        │                 │ 设置 Pending       │
        │                 ▼                   │
        │           ┌──────────┐              │
        │           │ Pending  │              │
        │           │   (01)   │              │
        │           └─────┬────┘              │
        │                 │ IAR 读取           │
        │                 ▼                   │
        │           ┌──────────┐              │
        │           │  Active  │──────────────┘
        │           │   (10)   │
        │           └──────────┘
```

---

## 3. 虚拟中断注入完整路径

### 3.1 路径一：Hypervisor 写 List Register

```
Hypervisor 决定注入虚拟中断
          │
          ▼
MSR ICH_LR<n>_EL2, Xd   (设置 State=Pending, vINTID, Priority, Group)
          │
          ▼
ich_lr_write() [arm_gicv3_cpuif.c:2830]
  │
  ├── 存储到 cs->ich_lr_el2[regno]
  ├── 强制 RES0: 优先级低位清零 (vpribits < 8)
  ├── 强制 RES0: NMI 位 (无 FEAT_GICv3_NMI 时)
  │
  └── gicv3_cpuif_virt_update(cs)  ← 触发虚拟中断信号更新
          │
          ▼
gicv3_cpuif_virt_irq_fiq_update(cs) [L471]
  │
  ├── hppvi_index(cs)  ← 找最高优先级 pending 虚拟中断
  │     │
  │     ├── 遍历所有 List Register
  │     ├── 过滤: State == Pending && Group Enable
  │     ├── 选择最高优先级 (最低数值)
  │     └── 与 vLPI (hppvlpi) 比较
  │
  ├── icv_hppi_can_preempt(cs, lr)  ← 优先级/抢占检查
  │     │
  │     ├── ICH_HCR_EL2.En == 1? (虚拟接口使能)
  │     ├── Priority < VPMR? (未被优先级 mask)
  │     ├── 能否抢占当前活跃中断? (group priority 比较)
  │     └── NMI 特殊处理 (可抢占非 NMI)
  │
  └── 输出信号到 CPU:
        if (Group0)     → qemu_set_irq(cs->parent_vfiq, 1)
        if (Group1)     → qemu_set_irq(cs->parent_virq, 1)
        if (Group1+NMI) → qemu_set_irq(cs->parent_vnmi, 1)
```

### 3.2 路径二：HCR_EL2.VI/VF 直接注入

```
Hypervisor 设置 HCR_EL2.VI = 1 (不经过 GIC List Register)
          │
          ▼
arm_cpu_update_virq(cpu) [cpu-irq.c:277]
  │
  ├── new_state = (HCR_VI && !HCRX_VINMI) || irq_line_state
  └── cpu_interrupt(cs, CPU_INTERRUPT_VIRQ)
          │
          ▼
arm_cpu_exec_interrupt() → EXCP_VIRQ → target_el=1
```

### 3.3 从 GIC 物理中断信号到虚拟中断

```
Device 产生中断
     │
     ▼
gicv3_dist_set_irq() → gicv3_update() → gicv3_cpuif_update()
     │
     ▼
gicv3_cpuif_update(cs) [L1046]:
  │
  ├── 物理路径: cs->hppi (highest priority pending interrupt)
  │   └── qemu_set_irq(cs->parent_irq/fiq, level)
  │       └── arm_cpu_set_irq(ARM_CPU_IRQ/FIQ)
  │           └── cpu_interrupt(cs, CPU_INTERRUPT_HARD/FIQ)
  │
  │ (物理中断路由到 EL2 因为 HCR_EL2.IMO=1)
  │ (Hypervisor 在中断 handler 中:)
  │
  ▼
Hypervisor 读 ICC_IAR1_EL1 → 获取物理 INTID
Hypervisor 写 ICH_LR<n>_EL2 → 设置 HW=1, pINTID, vINTID, State=Pending
     │
     ▼
[回到路径一: gicv3_cpuif_virt_update()]
```

---

## 4. Guest 中断处理流程 (ICV 接口)

### 4.1 ICC → ICV 透明重定向

```c
// arm_gicv3_cpuif.c:85
static bool icv_access(CPUARMState *env, int hcr_flags)
{
    uint64_t hcr_el2 = arm_hcr_el2_eff(env);
    bool flagmatch = hcr_el2 & hcr_flags & (HCR_IMO | HCR_FMO);
    return flagmatch && arm_current_el(env) == 1
        && !arm_is_secure_below_el3(env);
}
```

条件：EL1 + Non-Secure + (HCR_EL2.IMO 或 FMO) → ICC 访问变为 ICV

### 4.2 Guest 读 ICV_IAR (中断确认)

```c
// icv_iar_read() [L800]
static uint64_t icv_iar_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    int grp = ri->crm == 8 ? GICV3_G0 : GICV3_G1NS;
    int idx = hppvi_index(cs);        // 最高优先级 pending 虚拟中断

    if (idx >= 0) {
        uint64_t lr = cs->ich_lr_el2[idx];
        if (thisgrp == grp && icv_hppi_can_preempt(cs, lr)) {
            intid = ich_lr_vintid(lr);           // 返回虚拟 INTID
            icv_activate_irq(cs, idx, grp);      // Pending → Active
        }
    }
    gicv3_cpuif_virt_update(cs);  // 更新下一个 pending
    return intid;
}
```

### 4.3 icv_activate_irq (状态转换)

```c
// L765
static void icv_activate_irq(GICv3CPUState *cs, int idx, int grp)
{
    uint32_t mask = icv_gprio_mask(cs, grp);
    int prio = ich_lr_prio(cs->ich_lr_el2[idx]) & mask;
    int aprbit = prio >> (8 - cs->vprebits);

    // 状态: Pending → Active
    cs->ich_lr_el2[idx] &= ~ICH_LR_EL2_STATE_PENDING_BIT;
    cs->ich_lr_el2[idx] |= ICH_LR_EL2_STATE_ACTIVE_BIT;

    // 设置 Active Priority Register (用于抢占判断)
    cs->ich_apr[grp][aprbit/32] |= (1U << (aprbit % 32));
}
```

### 4.4 Guest 写 ICV_EOIR (结束中断)

```c
// icv_eoir_write() [L1584]
static void icv_eoir_write(CPUARMState *env, const ARMCPRegInfo *ri,
                           uint64_t value)
{
    int irq = value & 0xffffff;
    int grp = ri->crm == 8 ? GICV3_G0 : GICV3_G1NS;

    // 1. 优先级 Drop (降低运行优先级)
    dropprio = icv_drop_prio(cs, &nmi);

    // 2. 找到对应的 Active LR
    idx = icv_find_active(cs, irq);

    if (idx >= 0) {
        // 3. 如果 EOI mode == 0: 同时 deactivate
        if (!icv_eoi_split(env, cs)) {
            icv_deactivate_irq(cs, idx);
        }
        // 如果 EOI mode == 1: 需要 Guest 后续写 ICV_DIR 来 deactivate
    }
    gicv3_cpuif_virt_update(cs);
}
```

### 4.5 HW bit — 物理-虚拟联动去激活

```c
// icv_deactivate_irq() [L1474]
static void icv_deactivate_irq(GICv3CPUState *cs, int idx)
{
    uint64_t lr = cs->ich_lr_el2[idx];

    if (lr & ICH_LR_EL2_HW) {
        // HW=1: 同时去激活物理中断!
        int pirq = ich_lr_pintid(lr);
        if (pirq < INTID_SECURE) {
            icc_deactivate_irq(cs, pirq);  // 物理中断 Active→Invalid
        }
    }

    // 清除 Active 位: Active→Invalid 或 Active+Pending→Pending
    lr &= ~ICH_LR_EL2_STATE_ACTIVE_BIT;
    cs->ich_lr_el2[idx] = lr;
}
```

**HW bit 的意义**：
- `HW=0`：纯软件虚拟中断（如 vTimer），无物理对应
- `HW=1`：物理中断直通（如直通设备），Guest EOI 时自动去激活物理 GIC 中的中断
- 这避免了 Hypervisor 需要再次介入做物理 EOI 的开销

---

## 5. 优先级与抢占机制

### 5.1 虚拟 Priority Mask (VPMR)

```c
// icv_hppi_can_preempt() [L293]
vpmr = extract64(cs->ich_vmcr_el2, ICH_VMCR_EL2_VPMR_SHIFT, ...);
if (!is_nmi && prio >= vpmr) {
    return false;  // 被优先级 mask 屏蔽
}
```

### 5.2 虚拟 Binary Point (VBPR)

```c
// icv_gprio_mask() [L258]
// VBPR 决定 group priority 与 subpriority 的分界
bpr = read_vbpr(cs, group);
return ~0U << (bpr + 1);  // group priority mask
```

### 5.3 抢占判断

```c
// 只有 group priority 更高（数值更小）才能抢占
mask = icv_gprio_mask(cs, grp);
if ((prio & mask) < (rprio & mask)) {
    return true;  // 可以抢占
}
// NMI 可以抢占同优先级非 NMI
if ((prio & mask) == (rprio & mask) && is_nmi && !running_nmi) {
    return true;
}
```

---

## 6. Maintenance Interrupt (维护中断)

### 6.1 触发条件

```c
// maintenance_interrupt_state() [L395-418]
static uint32_t maintenance_interrupt_state(GICv3CPUState *cs)
{
    // EOI count overflow (Guest EOI 了不存在的中断)
    if (ICH_HCR.EOIcount != 0 && ICH_HCR.LRENPIE) → ICH_MISR.LRENP

    // Underflow (少于2个 LR 有中断)
    if (ICH_HCR.UIE && valid_count <= 1) → ICH_MISR.U

    // No Pending (没有 pending 中断)
    if (ICH_HCR.NPIE && no_pending) → ICH_MISR.NP

    // Group Enable/Disable 变化
    if (ICH_HCR.VGrp0EIE/VGrp1EIE/VGrp0DIE/VGrp1DIE) → 对应 MISR 位
}
```

### 6.2 Maintenance → Hypervisor 通知

```c
// gicv3_cpuif_virt_update() [L526]
static void gicv3_cpuif_virt_update(GICv3CPUState *cs)
{
    gicv3_cpuif_virt_irq_fiq_update(cs);  // vIRQ/vFIQ/vNMI 更新

    if ((cs->ich_hcr_el2 & ICH_HCR_EL2_EN) &&
        maintenance_interrupt_state(cs) != 0) {
        maintlevel = 1;
    }
    // 维护中断 → Redistributor → 物理 PPI → Hypervisor
    qemu_set_irq(cpu->gicv3_maintenance_interrupt, maintlevel);
}
```

维护中断的用途：
- **Underflow**：通知 Hypervisor 补充 List Register（LR 快用完了）
- **No Pending**：所有虚拟中断已处理完
- **EOI overflow**：检测 Guest 错误行为

---

## 7. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Hardware / Device                          │
│                     (Timer, Network, Disk, ...)                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ qemu_set_irq()
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          GIC Distributor                             │
│                     (arm_gicv3_dist.c)                               │
│                                                                     │
│  gicv3_dist_set_irq() → 路由到目标 CPU → gicv3_update()            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        GIC Redistributor                            │
│                    (arm_gicv3_redist.c)                              │
│                                                                     │
│  gicv3_redist_update() → 计算 hppi (最高优先级 pending)              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    物理 CPU Interface                                │
│                   (arm_gicv3_cpuif.c)                                │
│                                                                     │
│  gicv3_cpuif_update():                                              │
│    ├── icc_hppi_can_preempt() → 抢占检查                            │
│    ├── Group → FIQ/IRQ/NMI 映射                                     │
│    └── qemu_set_irq(parent_irq/fiq/nmi)                             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ arm_cpu_set_irq(ARM_CPU_IRQ)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        EL2 Hypervisor                                │
│                                                                     │
│  (因为 HCR_EL2.IMO=1, 物理 IRQ 路由到 EL2)                          │
│                                                                     │
│  1. 读 ICC_IAR1_EL1 → 获取 pINTID                                   │
│  2. 写 ICH_LR<n>_EL2 = {vINTID, pINTID, HW=1, Prio, State=Pending}  │
│  3. ERET → 返回 Guest                                               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ ich_lr_write() → gicv3_cpuif_virt_update()
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    虚拟 CPU Interface                                │
│                   (arm_gicv3_cpuif.c)                                │
│                                                                     │
│  gicv3_cpuif_virt_irq_fiq_update():                                 │
│    ├── hppvi_index() → 选最高优先级 LR                               │
│    ├── icv_hppi_can_preempt() → 抢占检查                            │
│    └── qemu_set_irq(parent_virq/vfiq/vnmi)                          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ arm_cpu_set_irq(ARM_CPU_VIRQ)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     CPU IRQ Processing                               │
│                      (cpu-irq.c)                                     │
│                                                                     │
│  arm_cpu_update_virq():                                             │
│    new_state = HCR.VI || irq_line_state                             │
│    → cpu_interrupt(cs, CPU_INTERRUPT_VIRQ)                          │
│                                                                     │
│  arm_cpu_exec_interrupt():                                          │
│    → EXCP_VIRQ, target_el=1                                        │
│    → arm_cpu_do_interrupt_aarch64()                                 │
│    → PC = VBAR_EL1 + 0x480 (Lower EL AArch64 IRQ)                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      EL1 Guest OS                                    │
│                                                                     │
│  IRQ Handler:                                                       │
│    1. MRS X0, ICV_IAR1_EL1 → icv_iar_read() → vINTID               │
│       (LR: Pending → Active)                                        │
│    2. 处理中断                                                       │
│    3. MSR ICV_EOIR1_EL1, X0 → icv_eoir_write()                     │
│       (LR: Active → Invalid, 如果 HW=1 则物理 deactivate)           │
│    4. ERET 返回                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. ICH_VMCR_EL2 虚拟机控制

### 8.1 关键控制位

| 位 | 名称 | 功能 |
|---|------|------|
| VENG0 | Virtual Enable Group 0 | Guest 的 Group 0 中断使能 |
| VENG1 | Virtual Enable Group 1 | Guest 的 Group 1 中断使能 |
| VCBPR | Virtual CBPR | Group 0/1 共享 BPR |
| VEOIM | Virtual EOI Mode | 0=同时 drop+deactivate, 1=分离 |
| VBPR0/VBPR1 | Virtual BPR | 虚拟 Binary Point |
| VPMR | Virtual PMR | 虚拟优先级 Mask |

### 8.2 QEMU 中的 ICV 寄存器映射

Guest 访问 ICC_* 寄存器时，QEMU 根据 `icv_access()` 透明重定向到 ICV 实现：

| Guest 寄存器 | QEMU 读函数 | QEMU 写函数 | VMCR 对应 |
|-------------|------------|------------|-----------|
| ICC_PMR_EL1 | icv_pmr_read | icv_pmr_write | VPMR |
| ICC_IGRPEN0_EL1 | icv_igrpen_read | icv_igrpen_write | VENG0 |
| ICC_IGRPEN1_EL1 | icv_igrpen_read | icv_igrpen_write | VENG1 |
| ICC_IAR0_EL1 | icv_iar_read | — | 操作 LR |
| ICC_IAR1_EL1 | icv_iar_read | — | 操作 LR |
| ICC_EOIR0_EL1 | — | icv_eoir_write | 操作 LR |
| ICC_EOIR1_EL1 | — | icv_eoir_write | 操作 LR |
| ICC_DIR_EL1 | — | icv_dir_write | 操作 LR |
| ICC_CTLR_EL1 | icv_ctlr_read | icv_ctlr_write | VCBPR, VEOIM |
| ICC_BPR0_EL1 | icv_bpr_read | icv_bpr_write | VBPR0 |
| ICC_BPR1_EL1 | icv_bpr_read | icv_bpr_write | VBPR1 |
| ICC_RPR_EL1 | icv_rpr_read | — | APR |
| ICC_HPPIR0_EL1 | icv_hppir_read | — | 查询 LR |
| ICC_HPPIR1_EL1 | icv_hppir_read | — | 查询 LR |

---

## 9. vLPI (Virtual LPI) 支持

### 9.1 vLPI 概述

vLPI (Virtual Locality-specific Peripheral Interrupts) 是 GICv4 引入的特性，允许虚拟 LPI 直接注入 Guest。

```c
// hppvi_index() 中:
if (cs->hppvlpi.prio < prio && !arm_is_secure(env)) {
    if (cs->hppvlpi.grp == GICV3_G0) {
        if (cs->ich_vmcr_el2 & ICH_VMCR_EL2_VENG0)
            return HPPVI_INDEX_VLPI;
    } else {
        if (cs->ich_vmcr_el2 & ICH_VMCR_EL2_VENG1)
            return HPPVI_INDEX_VLPI;
    }
}
```

### 9.2 vLPI vs List Register 优先级

- vLPI 和 List Register 中的中断统一比较优先级
- 最高优先级者胜出（无论来源）
- vLPI 无需占用 List Register 槽位

---

## 10. NMI 虚拟化 (FEAT_GICv3_NMI)

### 10.1 List Register 中的 NMI

```c
// ich_lr_write():
if (!cs->nmi_support) {
    value &= ~ICH_LR_EL2_NMI;  // 无 NMI 支持时强制清零
}
```

### 10.2 NMI 优先级选择

```c
// hppvi_index():
// NMI 在相同优先级时优先于普通中断
if ((thisprio < prio) || ((thisprio == prio) && (thisnmi & (!nmi)))) {
    prio = thisprio;
    nmi = thisnmi;
    idx = i;
}
```

### 10.3 虚拟 NMI 信号

```c
// gicv3_cpuif_virt_irq_fiq_update():
if (lr & ICH_LR_EL2_GROUP) {
    if (lr & ICH_LR_EL2_NMI) {
        nmilevel = 1;   // Group1 + NMI → VNMI 信号
    } else {
        irqlevel = 1;   // Group1 → VIRQ 信号
    }
} else {
    fiqlevel = 1;       // Group0 → VFIQ 信号 (NMI 不影响 G0)
}
```

---

## 11. 与规范的一致性评估

### 11.1 功能覆盖

| 规范要求 (IHI 0069H §8) | QEMU 状态 | 说明 |
|------------------------|:--------:|------|
| ICH_LR_EL2 写入 → 虚拟中断 pending | ✅ | ich_lr_write + virt_update |
| List Register 状态机 (I→P→A→I) | ✅ | 完整 4 状态 |
| 优先级/组过滤 | ✅ | VENG0/VENG1 + VPMR |
| 抢占 (VBPR group priority) | ✅ | icv_gprio_mask + can_preempt |
| ICV 透明重定向 | ✅ | icv_access() + HCR.IMO/FMO |
| HW bit 物理联动 | ✅ | icv_deactivate_irq → icc_deactivate_irq |
| EOI mode split (DIR) | ✅ | icv_eoi_split + icv_dir_write |
| Maintenance interrupt | ✅ | 7 种条件完整 |
| vLPI (GICv4) | ✅ | hppvlpi + HPPVI_INDEX_VLPI |
| FEAT_GICv3_NMI | ✅ | NMI 位 + 优先级 + VNMI 信号 |
| Active Priority Registers | ✅ | ich_apr[][], icv_drop_prio |
| INTID_SPURIOUS (1023) 返回 | ✅ | 无 pending 时返回 |

### 11.2 已知简化/差异

| 方面 | 说明 | 严重度 |
|------|------|:------:|
| List Register 数量 | QEMU 默认 4-16 个 (可配置) | ✅ 正常 |
| vLPI pending table | 内存中的 pending 表简化为软件结构 | P3 |
| Doorbell interrupt | vPE 相关的 doorbell 部分简化 | P3 |
| 并发/原子性 | 无多核 race (BQL 保护) | P2 |

### 11.3 总结

QEMU 的 GICv3 虚拟化接口实现**非常完善**：
- 完整的 List Register 状态机管理
- 正确的优先级/抢占/group 处理
- HW bit 物理-虚拟联动去激活（性能关键路径）
- 维护中断全部 7 种条件实现
- FEAT_GICv3_NMI 虚拟化支持
- vLPI 直接注入支持

这是 QEMU 中与 ARM 规范一致性最高的子系统之一。
