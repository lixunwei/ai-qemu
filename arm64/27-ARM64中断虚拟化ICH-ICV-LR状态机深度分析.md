# ARM64 中断虚拟化深度分析：ICH/ICV/LR 状态机与维护中断

## 1. 概述

本文深度分析 QEMU 11.0.50 中 ARM GICv3 中断虚拟化的完整实现，包括 Hypervisor 控制接口（ICH）、虚拟 CPU 接口（ICV）、List Register 状态机、维护中断机制，以及 GICv2 虚拟接口和 GICv4 直接注入。中断虚拟化是 ARM 虚拟化扩展的核心，允许 Hypervisor（EL2）高效地向虚拟机（EL1）注入虚拟中断。

**关键源文件：**
- `hw/intc/arm_gicv3_cpuif.c` — ICV/ICH 寄存器模拟（~2700行）
- `hw/intc/gicv3_internal.h` — ICH_LR/ICH_HCR/ICH_VMCR 位域定义
- `hw/intc/arm_gic.c` — GICv2 GICH/GICV 虚拟接口（~2200行）
- `hw/intc/gic_internal.h` — GICH_LR 位域定义
- `hw/intc/arm_gicv3_redist.c` — vLPI/GICv4 直接注入支持

---

## 2. 中断虚拟化架构总览

### 2.1 三层结构

```
                    物理中断                虚拟中断
                    ┌──────┐               ┌──────┐
                    │ GICD │               │(无)  │  虚拟中断不经过物理分发器
                    └──┬───┘               └──────┘
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │  GICR   │   │  GICR   │   │  GICR   │   物理重分发器
   └────┬────┘   └────┬────┘   └────┬────┘
        │              │              │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │  ICC    │   │  ICC    │   │  ICC    │   物理 CPU 接口
   │(EL2/EL3)│   │(EL2/EL3)│   │(EL2/EL3)│   Hypervisor 直接操作
   └────┬────┘   └────┬────┘   └────┬────┘
        │              │              │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │  ICH    │   │  ICH    │   │  ICH    │   虚拟化控制
   │(EL2)    │   │(EL2)    │   │(EL2)    │   List Register 管理
   └────┬────┘   └────┬────┘   └────┬────┘
        │              │              │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │  ICV    │   │  ICV    │   │  ICV    │   虚拟 CPU 接口
   │(NS EL1) │   │(NS EL1) │   │(NS EL1) │   Guest 透明访问
   └─────────┘   └─────────┘   └─────────┘
```

### 2.2 ICV 重定向机制

```c
// arm_gicv3_cpuif.c:85-106 — icv_access()
// 判断条件:
//   1. 当前在 NS EL1（非安全 EL1）
//   2. HCR_EL2.IMO 或 FMO 已置位
// 满足条件时，ICC_* 访问透明重定向到 ICV_*

// 四类寄存器重定向规则:
// - HCR_EL2.FMO=1: ICV_*0_* (G0 寄存器，如 ICV_IAR0, ICV_EOIR0)
// - HCR_EL2.IMO=1: ICV_*1_* (G1 寄存器，如 ICV_IAR1, ICV_EOIR1)
// - IMO||FMO: ICV_CTLR, ICV_DIR, ICV_PMR, ICV_RPR
```

**Guest 完全无感知** — Guest OS 认为自己在操作物理 ICC 寄存器，实际上所有读写都被重定向到了虚拟寄存器。

---

## 3. List Register（LR）格式与状态机

### 3.1 GICv3 LR 格式（ICH_LR<n>_EL2，64位）

```c
// gicv3_internal.h:239-262
// 位域布局:
// [63:62] State     — 状态: 00=Invalid, 01=Pending, 10=Active, 11=PendingActive
// [61]    HW        — 硬件位: 1=关联物理中断（EOI 时自动 deactivate 物理中断）
// [60]    Group     — 组: 0=G0(FIQ), 1=G1(IRQ)
// [59]    NMI       — NMI 标志（GICv3.1+）
// [55:48] Priority  — 虚拟优先级
// [41]    EOI       — EOI 通知标志（与 pINTID 高位共用）
// [41:32] pINTID    — 物理中断 ID（HW=1 时有效）
// [31:0]  vINTID    — 虚拟中断 ID

#define ICH_LR_EL2_STATE_INVALID        0   // 空闲
#define ICH_LR_EL2_STATE_PENDING        1   // 待处理
#define ICH_LR_EL2_STATE_ACTIVE         2   // 已激活
#define ICH_LR_EL2_STATE_ACTIVE_PENDING 3   // 激活且有新 pending
```

### 3.2 GICv2 LR 格式（GICH_LR<n>，32位）

```c
// gic_internal.h:111-142
// 位域布局:
// [31]    HW         — 硬件关联
// [30]    Grp1       — 组1标志
// [29:28] State      — 状态（同 GICv3 四状态）
// [27:23] Priority   — 5 位优先级
// [19]    EOI        — EOI 维护中断通知
// [12:10] CPUID      — SGI 源 CPU（HW=0 时有效）
// [9:0]   VirtualID  — 虚拟中断 ID
```

### 3.3 LR 状态机

```
                Hypervisor 写入 LR
                      │
                      ↓
            ┌───────────────────┐
            │  Invalid (00)     │
            │  LR 空闲          │
            └───────┬───────────┘
                    │ Hypervisor 设置 State=01
                    ↓
            ┌───────────────────┐
    ┌──────→│  Pending (01)     │←─────── 新中断到达（Pending+Active→PendingActive）
    │       │  等待 Guest 应答   │
    │       └───────┬───────────┘
    │               │ Guest 读 ICV_IAR
    │               │ icv_activate_irq() :765-786
    │               ↓
    │       ┌───────────────────┐
    │       │  Active (10)      │
    │       │  Guest 正在处理    │
    │       └───────┬───────────┘
    │               │ Guest 写 ICV_EOIR
    │               │ icv_eoir_write() :1584-1645
    │               ↓
    │       ┌───────────────────┐
    │       │  HW=1?            │
    │       │  EOI split?       │
    │       └───┬───────┬───────┘
    │           │No     │Yes(EOI split)
    │           ↓       ↓
    │      Invalid    仅 drop prio
    │      (00)       等待 ICV_DIR deactivate
    │                      │
    └──────────────────────┘ icv_dir_write() :1551-1582
```

### 3.4 状态转换实现

```c
// icv_activate_irq() — Pending → Active
// arm_gicv3_cpuif.c:765-786
cs->ich_lr_el2[idx] &= ~ICH_LR_EL2_STATE_PENDING_BIT;  // 清 Pending
cs->ich_lr_el2[idx] |= ICH_LR_EL2_STATE_ACTIVE_BIT;    // 设 Active
// 同时在 ich_apr[] 中记录 active 优先级

// icv_eoir_write() — priority drop + optional deactivate
// arm_gicv3_cpuif.c:1584-1645
dropprio = icv_drop_prio(cs, &nmi);  // 降低运行优先级
if (!icv_eoi_split(env, cs)) {
    icv_deactivate_irq(cs, idx);      // 非 split: 立即 deactivate
}
// split 模式: 仅 drop priority, 等待 ICV_DIR 才 deactivate
```

---

## 4. ICV 寄存器模拟详解

### 4.1 ICV_IAR — 虚拟中断应答

```c
// arm_gicv3_cpuif.c:800-842 — icv_iar_read()
// 流程:
// 1. hppvi_index() 找最高优先级 pending LR（或 vLPI）
// 2. 检查组匹配 + 可抢占
// 3. vLPI: icv_activate_vlpi()（清 pending table，设 APR）
//    LR: icv_activate_irq()（状态 P→A，设 APR）
// 4. 返回 vINTID
// 5. gicv3_cpuif_virt_update() 重新评估 vIRQ/vFIQ

// 特殊: NMI 中断返回 INTID_NMI (0x1FFFFFFD)
```

### 4.2 ICV_EOIR — 虚拟 EOI

```c
// arm_gicv3_cpuif.c:1584-1645 — icv_eoir_write()
// 流程:
// 1. icv_drop_prio() 从 ich_apr[] 清除最高 active 位
// 2. icv_find_active() 在 LR 中查找匹配的 active 中断
// 3. 检查组匹配 + 优先级匹配
// 4. 非 split 模式: icv_deactivate_irq() 清除 LR active 位
//    split 模式: 仅 drop priority，保留 LR active
// 5. HW=1 的 LR: deactivate 时同时 deactivate 物理中断
// 6. EOI 后 LR 变空 → 可能触发维护中断
```

### 4.3 ICV_PMR/BPR/CTLR/IGRPEN

```c
// arm_gicv3_cpuif.c:560-760
// 所有 ICV 寄存器操作 ICH_VMCR_EL2 中的虚拟控制位:
//   ICV_PMR  → ICH_VMCR_EL2.VPMR[31:24]
//   ICV_BPR0 → ICH_VMCR_EL2.VBPR0[23:21]
//   ICV_BPR1 → ICH_VMCR_EL2.VBPR1[20:18]
//   ICV_CTLR → ICH_VMCR_EL2.VCBPR[4] + VEOIM[9]
//   ICV_IGRPEN0 → ICH_VMCR_EL2.VENG0[0]
//   ICV_IGRPEN1 → ICH_VMCR_EL2.VENG1[1]
// 每次写入后调用 gicv3_cpuif_virt_update() 重新评估
```

### 4.4 ICV_HPPIR — 最高优先级 Pending

```c
// arm_gicv3_cpuif.c:740-762 — icv_hppir_read()
// 调用 hppvi_index() 找最高优先级 pending LR
// 检查组匹配 → 返回 vINTID 或 1023(spurious)
```

---

## 5. ICH 虚拟化控制寄存器

### 5.1 ICH_HCR_EL2 — Hypervisor 控制

```c
// gicv3_internal.h:222-237
// 关键控制位:
// [0]  En      — 虚拟 CPU 接口使能
// [1]  UIE     — Underflow 中断使能（<2 valid LR）
// [2]  LRENPIE — List Register 进入空 Pending 中断使能
// [3]  NPIE    — No Pending 中断使能
// [4]  VGRP0EIE— 虚拟 Group 0 使能中断
// [5]  VGRP0DIE— 虚拟 Group 0 禁用中断
// [6]  VGRP1EIE— 虚拟 Group 1 使能中断
// [7]  VGRP1DIE— 虚拟 Group 1 禁用中断
// [10] TC      — Trap CTLR
// [11] TALL0   — Trap ALL Group 0
// [12] TALL1   — Trap ALL Group 1
// [13] TSEI    — Trap SEI
// [14] TDIR    — Trap DIR
// [31:27] EOIcount — EOI 计数（维护中断用）
```

### 5.2 ICH_VMCR_EL2 — 虚拟机控制

```c
// gicv3_internal.h:202-220
// 保存虚拟 CPU 接口的所有配置:
// [0]     VENG0  — 虚拟 Group 0 使能
// [1]     VENG1  — 虚拟 Group 1 使能
// [3]     VFIQEN — 虚拟 FIQ 使能
// [4]     VCBPR  — 虚拟 CBPR
// [9]     VEOIM  — 虚拟 EOI 模式（split）
// [20:18] VBPR1  — 虚拟 BPR1
// [23:21] VBPR0  — 虚拟 BPR0
// [31:24] VPMR   — 虚拟 PMR
```

### 5.3 ICH_AP0R/AP1R — 虚拟 Active Priority

```c
// arm_gicv3_cpuif.c:560-587 — icv_ap_read/write()
// 直接映射到 cs->ich_apr[GICV3_G0][n] 和 cs->ich_apr[GICV3_G1NS][n]
// 用于追踪虚拟中断的嵌套/抢占关系
// 写入后触发 gicv3_cpuif_virt_update()
```

---

## 6. 虚拟中断投递与维护中断

### 6.1 虚拟中断投递

```c
// arm_gicv3_cpuif.c:471-524 — gicv3_cpuif_virt_irq_fiq_update()
// 流程:
// 1. hppvi_index() 找最高优先级 pending（LR 或 vLPI）
// 2. vLPI: icv_hppvlpi_can_preempt() 检查抢占
//    LR: icv_hppi_can_preempt() 检查抢占
// 3. 根据 Group 和 NMI 决定信号:
//    G0 → vFIQ
//    G1 → vIRQ（或 NMI → vNMI）
// 4. qemu_set_irq() 驱动 vFIQ/vIRQ/vNMI 输出线
```

### 6.2 维护中断触发

```c
// arm_gicv3_cpuif.c:434-469 — maintenance_interrupt_state()
// 维护中断 (ICH_MISR_EL2) 触发条件:
//
// EOI: LR.State==0 && LR.HW==0 && LR.EOI==1
//      → Hypervisor 需要完成软件 EOI 处理
//
// U (Underflow): valid LR < 2 && ICH_HCR.UIE
//      → Hypervisor 可能需要填充更多 LR
//
// NP (No Pending): 无 pending LR && ICH_HCR.NPIE
//      → Hypervisor 知道所有虚拟中断已处理
//
// LRENP: EOIcount != 0 && ICH_HCR.LRENPIE
//      → Guest EOI 了不在 LR 中的中断
//
// VGRP0E/D, VGRP1E/D: 虚拟组使能/禁用变化
//      → Hypervisor 跟踪 Guest 的组使能状态
```

### 6.3 维护中断递归处理

```c
// arm_gicv3_cpuif.c:526-558 — gicv3_cpuif_virt_update()
// 重要: 此函数会触发 qemu_set_irq(maintenance_interrupt)
// 维护中断连线到 GIC 作为 per-CPU PPI
// 这导致递归: virt_update → set_irq → redist_set_irq → cpuif_update
// 代码注释特别警告此递归:
//   "CAUTION: this function will call qemu_set_irq() on the
//    CPU maintenance IRQ line, which is typically wired up
//    to the GIC as a per-CPU interrupt."
```

---

## 7. 最高优先级虚拟中断选择

### 7.1 hppvi_index()

```c
// arm_gicv3_cpuif.c:179-256 — hppvi_index()
// 算法:
// 1. 检查 ICH_VMCR.VENG0/VENG1，两组都禁用则返回 -1
// 2. 遍历所有 LR:
//    - 跳过非 Pending 状态
//    - 检查组使能
//    - NMI 最高优先级
//    - 比较优先级，记录最低值
// 3. 与 vLPI 最高优先级比较:
//    - HPPVI_INDEX_VLPI 如果 vLPI 更优先
//    - 否则返回 LR index
```

### 7.2 虚拟运行优先级

```c
// arm_gicv3_cpuif.c:154-177 — ich_highest_active_virt_prio()
// 扫描 ich_apr[G0][] 和 ich_apr[G1NS][]
// NMI 特殊: ich_apr[G1NS][0] & ICV_AP1R_EL1_NMI → 优先级 0x0
// 正常: 找最低 set bit → 优先级 = (bit位置) << (vBPR+1)
// 无 active: 返回 0xff (idle)
```

---

## 8. GICv2 虚拟接口

### 8.1 GICH — Hypervisor 接口（MMIO）

```c
// arm_gic.c:1909-2020 — gic_hyp_read/gic_hyp_write
// GICH 寄存器通过 MMIO 访问（与 GICv3 的系统寄存器不同）:
//   GICH_HCR  → s->h_hcr[cpu]
//   GICH_VTR  → (num_lrs - 1) | (7 << 26) 等
//   GICH_VMCR → GICC_CTLR/PMR/BPR 虚拟配置
//   GICH_MISR → 维护中断状态
//   GICH_EISR → EOI 中断状态
//   GICH_ELRSR→ 空 LR 状态
//   GICH_APR  → Active 优先级
//   GICH_LR<n>→ List Register（32位）
```

### 8.2 GICV — 虚拟 CPU 接口（MMIO）

```c
// arm_gic.c:1845-1860 — gic_thisvcpu_read/write
// GICV 与 GICC 共用同一套读写函数
// 区别: vCPU 访问使用 cpu + GIC_NCPU 作为索引
// 这样 cpu_ctlr[cpu+8]、priority_mask[cpu+8] 等
// 存储的是虚拟 CPU 接口的状态
```

### 8.3 GICv2 虚拟中断更新

```c
// arm_gic.c:164-225 — gic_update_internal(s, true)
// virt=true 时:
//   使用 parent_virq/parent_vfiq 输出线
//   cpu_iface = cpu + GIC_NCPU（访问虚拟状态）
//   gic_get_best_virq() 扫描 LR 找最高优先级 pending
//   Group 0 + FIQ_EN → vFIQ，否则 → vIRQ
```

---

## 9. GICv4 直接注入

### 9.1 vLPI 机制

```c
// GICv4 允许 vLPI 绕过 List Register 直接注入 vCPU
// 通过 GICR_VPROPBASER 和 GICR_VPENDBASER 配置:
//   VPROPBASER: vLPI 配置表基地址（优先级、使能）
//   VPENDBASER: vLPI pending 表基地址 + vPE 标识

// gicv3_internal.h:118-170 — GICR_VPROPBASER/VPENDBASER 定义
// GICR_TYPER.DirectLPI — 表示支持直接 LPI 注入
```

### 9.2 vLPI 投递路径

```c
// arm_gicv3_redist.c — vLPI 处理
// 1. ITS process_its_cmd_virt() 查找 vPE 表
// 2. 设置 vLPI pending 位
// 3. gicv3_redist_update_lpi_only() 扫描 vLPI pending
// 4. 更新 cs->hppvlpi（最高优先级 vLPI）
// 5. gicv3_cpuif_virt_irq_fiq_update() 投递
// 6. icv_iar_read() 中: icv_activate_vlpi() 清 pending + 设 APR
```

### 9.3 VMSTATE 迁移

```c
// arm_gicv3_common.c:150-167 — vmstate_gicv3_gicv4
// 条件: icc_ctlr_el1[GICV3_S] 的 GICv4 标志
// 保存: gicr_vpropbaser, gicr_vpendbaser
```

---

## 10. GICv2 vs GICv3 虚拟化对比

| 特性 | GICv2 | GICv3 |
|------|-------|-------|
| Hypervisor 接口 | GICH MMIO | ICH 系统寄存器 |
| 虚拟 CPU 接口 | GICV MMIO | ICV 系统寄存器重定向 |
| LR 宽度 | 32 位 | 64 位 |
| LR 数量 | 最多 64 | 最多 16（典型 4-16） |
| vINTID 范围 | 10 位 (0-1023) | 32 位 (0-2^32) |
| pINTID | 10 位 | 10 位（+EOI共用） |
| 优先级位数 | 5 位 | 8 位 |
| NMI 支持 | 无 | 有（LR.NMI） |
| vLPI/GICv4 | 无 | 有（绕过 LR） |
| SGI 源 CPU | LR.CPUID (3位) | 无 |
| 重定向机制 | GICV MMIO 硬件映射 | HCR.IMO/FMO + icv_access() |
| 维护中断 | GICH_MISR MMIO | ICH_MISR_EL2 系统寄存器 |

---

## 11. 完整虚拟中断生命周期

```
1. Hypervisor 捕获物理中断（HCR_EL2.IMO/FMO=1）
   │
2. Hypervisor 写入 ICH_LR<n>_EL2:
   │  State=Pending, vINTID=X, Priority=P
   │  可选: HW=1 + pINTID（关联物理中断）
   │
3. gicv3_cpuif_virt_update() 评估:
   │  hppvi_index() → 找到最高优先级 pending LR
   │  icv_hppi_can_preempt() → 可以抢占
   │  qemu_set_irq(vIRQ/vFIQ, 1) → 向 vCPU 发送虚拟中断
   │
4. vCPU 进入中断处理:
   │  Guest 读 ICV_IAR1 → icv_iar_read()
   │  icv_activate_irq(): LR.State = Pending → Active
   │  ich_apr[] 设置 active priority bit
   │  返回 vINTID
   │
5. Guest 中断处理中:
   │  虚拟中断可被更高优先级虚拟中断抢占
   │  LR.State 可变为 Active+Pending
   │
6. Guest 写 ICV_EOIR1 → icv_eoir_write():
   │  icv_drop_prio(): 清除 ich_apr[] 中的 active bit
   │  非 split: icv_deactivate_irq() → LR.State = Invalid
   │  split: 仅 drop priority，等待 ICV_DIR
   │
7. (split 模式) Guest 写 ICV_DIR → icv_dir_write():
   │  icv_deactivate_irq(): LR.State = Invalid
   │
8. HW=1 时: deactivate 同时触发物理中断 deactivation
   │
9. LR.State=Invalid:
   │  可能触发维护中断（EOI/Underflow/NoPending）
   │  Hypervisor 可复用此 LR 注入新中断
```

---

## 12. 小结

| 维度 | 实现 |
|------|------|
| **ICV 重定向** | icv_access() 检查 NS EL1 + HCR.IMO/FMO，透明转发到虚拟寄存器 |
| **LR 状态机** | Invalid→Pending→Active→(PendingActive)→Invalid，4 状态严格遵循 |
| **虚拟 IAR** | hppvi_index() 找最高优先级 + icv_activate_irq() 状态转换 + APR 记录 |
| **虚拟 EOI** | icv_drop_prio() + icv_deactivate_irq()，支持 split EOI 模式 |
| **维护中断** | 6 种触发条件（EOI/U/NP/LRENP/VGRP0/VGRP1），递归回 GIC 的 PPI |
| **HW 位** | 直接关联物理-虚拟中断，EOI 时联动 deactivate |
| **GICv4** | vLPI 绕过 LR，通过 VPROPBASER/VPENDBASER 配置表直接注入 |
| **代码规模** | GICv3 ICV ~400行 + ICH ~200行 + 维护 ~200行 = ~800行虚拟化代码 |
