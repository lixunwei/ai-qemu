# GICv3 中断控制器深度分析

> 基于 QEMU 11.0.50 源码，深度分析 ARM GICv3 通用中断控制器的完整实现：
> 设备模型、Distributor/Redistributor/CPU Interface、中断注入路径、优先级与抢占、
> 虚拟中断、SGI、KVM 支持，以及与 ARM CPU 的集成机制。

---

## 目录

1. [GIC 架构概述](#1-gic-架构概述)
2. [GICv3 设备模型与类型注册](#2-gicv3-设备模型与类型注册)
3. [GICv3State 全局状态结构](#3-gicv3state-全局状态结构)
4. [GICv3CPUState 每 CPU 状态结构](#4-gicv3cpustate-每-cpu-状态结构)
5. [中断类型与编号空间](#5-中断类型与编号空间)
6. [Distributor (GICD) 寄存器实现](#6-distributor-gicd-寄存器实现)
7. [GICD 位图管理与 NSACR 安全控制](#7-gicd-位图管理与-nsacr-安全控制)
8. [GICD_IROUTER 亲和路由](#8-gicd_irouter-亲和路由)
9. [Redistributor (GICR) 寄存器实现](#9-redistributor-gicr-寄存器实现)
10. [LPI 支持 (Locality-specific Peripheral Interrupt)](#10-lpi-支持)
11. [CPU Interface (ICC) 系统寄存器](#11-cpu-interface-icc-系统寄存器)
12. [icv_access() 虚拟化重定向](#12-icv_access-虚拟化重定向)
13. [中断优先级与抢占模型](#13-中断优先级与抢占模型)
14. [irqbetter() 优先级比较](#14-irqbetter-优先级比较)
15. [icc_hppi_can_preempt() 抢占判断](#15-icc_hppi_can_preempt-抢占判断)
16. [icc_highest_active_prio() 运行优先级](#16-icc_highest_active_prio-运行优先级)
17. [gicv3_cpuif_update() 物理中断信号](#17-gicv3_cpuif_update-物理中断信号)
18. [Group 0/1 与 IRQ/FIQ 映射](#18-group-01-与-irqfiq-映射)
19. [完整中断注入路径：设备→GIC→CPU](#19-完整中断注入路径设备gic-cpu)
20. [gicv3_set_irq() 入口](#20-gicv3_set_irq-入口)
21. [gicv3_update() 全局重计算](#21-gicv3_update-全局重计算)
22. [arm_cpu_set_irq() CPU 端接收](#22-arm_cpu_set_irq-cpu-端接收)
23. [arm_cpu_has_work() 工作检查](#23-arm_cpu_has_work-工作检查)
24. [arm_cpu_exec_interrupt() 中断调度](#24-arm_cpu_exec_interrupt-中断调度)
25. [arm_excp_unmasked() 屏蔽检查](#25-arm_excp_unmasked-屏蔽检查)
26. [异常入口：IRQ/FIQ 向量计算](#26-异常入口irqfiq-向量计算)
27. [中断确认：ICC_IAR1_EL1 读取](#27-中断确认icc_iar1_el1-读取)
28. [icc_activate_irq() 状态转换](#28-icc_activate_irq-状态转换)
29. [中断结束：ICC_EOIR1_EL1 写入](#29-中断结束icc_eoir1_el1-写入)
30. [icc_drop_prio() 优先级下降](#30-icc_drop_prio-优先级下降)
31. [Split EOI 模式与 ICC_DIR](#31-split-eoi-模式与-icc_dir)
32. [虚拟中断 (EL2)：ICH 接口](#32-虚拟中断-el2ich-接口)
33. [gicv3_cpuif_virt_irq_fiq_update() 虚拟信号](#33-gicv3_cpuif_virt_irq_fiq_update-虚拟信号)
34. [SGI 软件生成中断](#34-sgi-软件生成中断)
35. [GIC KVM 支持](#35-gic-kvm-支持)
36. [KVM 设备创建与初始化](#36-kvm-设备创建与初始化)
37. [KVM 状态同步](#37-kvm-状态同步)
38. [Timer→GIC 中断连接示例](#38-timergic-中断连接示例)
39. [关键数据结构总览表](#39-关键数据结构总览表)
40. [端到端中断流程图](#40-端到端中断流程图)
41. [GICv2 vs GICv3 对比](#41-gicv2-vs-gicv3-对比)

---

## 1. GIC 架构概述

ARM GIC (Generic Interrupt Controller) 是 ARM 系统中统一的中断管理硬件。GICv3 是当前主流版本，支持：

- **Distributor (GICD)**：全局中断管理，处理 SPI (Shared Peripheral Interrupt)
- **Redistributor (GICR)**：每 CPU 本地中断管理，处理 SGI/PPI/LPI
- **CPU Interface (ICC)**：通过系统寄存器（非 MMIO）与 CPU 交互
- **虚拟化接口 (ICH/ICV)**：支持 EL2 下的虚拟中断注入

QEMU 中 GICv3 的实现分布在 `hw/intc/` 目录下多个文件中。

---

## 2. GICv3 设备模型与类型注册

```c
// arm_gicv3.c:465-478
static const TypeInfo arm_gicv3_info = {
    .name = TYPE_ARM_GICV3,             // "arm-gicv3"
    .parent = TYPE_ARM_GICV3_COMMON,    // 公共基类
    .instance_size = sizeof(GICv3State),
    .class_init = arm_gicv3_class_init,
    .class_size = sizeof(ARMGICv3Class),
};
type_init(arm_gicv3_register_types)
```

**设备属性**（`arm_gicv3_common.c:604-622`）：

| 属性 | 类型 | 说明 |
|------|------|------|
| `num-cpu` | uint32 | CPU 数量 |
| `num-irq` | uint32 | 中断数量 |
| `revision` | uint32 | GIC 版本（3 或 4） |
| `has-lpi` | bool | LPI 支持 |
| `has-nmi` | bool | NMI 支持（FEAT_GICv3_NMI） |
| `has-security-extensions` | bool | 安全扩展 |
| `redist-region-count` | uint32[] | Redistributor 区域分布 |

---

## 3. GICv3State 全局状态结构

```c
// include/hw/intc/arm_gicv3_common.h:225-270
struct GICv3State {
    SysBusDevice parent_obj;

    MemoryRegion iomem_dist;          // Distributor MMIO 区域
    GICv3RedistRegion *redist_regions; // Redistributor 区域数组

    uint32_t num_cpu;
    uint32_t num_irq;
    bool security_extn;
    bool nmi_support;
    int dev_fd;                       // KVM 设备 fd

    /* Distributor 状态 */
    uint32_t gicd_ctlr;              // GICD_CTLR 控制寄存器
    GIC_DECLARE_BITMAP(group);        // GICD_IGROUPR — 中断分组
    GIC_DECLARE_BITMAP(grpmod);       // GICD_IGRPMODR — 分组修改
    GIC_DECLARE_BITMAP(enabled);      // GICD_ISENABLER — 使能位
    GIC_DECLARE_BITMAP(pending);      // GICD_ISPENDR — 挂起位
    GIC_DECLARE_BITMAP(active);       // GICD_ISACTIVER — 激活位
    GIC_DECLARE_BITMAP(level);        // 当前电平
    GIC_DECLARE_BITMAP(edge_trigger); // GICD_ICFGR — 边沿/电平触发
    GIC_DECLARE_BITMAP(nmi);          // GICD_INMIR — NMI 标记
    uint8_t gicd_ipriority[GICV3_MAXIRQ]; // 优先级数组
    uint64_t gicd_irouter[GICV3_MAXIRQ];  // 路由寄存器
    GICv3CPUState *gicd_irouter_target[GICV3_MAXIRQ]; // 缓存的目标 CPU
    ...
};
```

**关键设计**：每个 SPI 中断都有独立的 group/enabled/pending/active/priority/router 位，通过位图高效管理。

---

## 4. GICv3CPUState 每 CPU 状态结构

```c
// include/hw/intc/arm_gicv3_common.h:127-213
struct GICv3CPUState {
    GICv3State *gic;
    CPUState *cpu;

    /* GIC→CPU 输出线 */
    qemu_irq parent_irq;      // 物理 IRQ
    qemu_irq parent_fiq;      // 物理 FIQ
    qemu_irq parent_virq;     // 虚拟 IRQ
    qemu_irq parent_vfiq;     // 虚拟 FIQ
    qemu_irq parent_nmi;      // 物理 NMI
    qemu_irq parent_vnmi;     // 虚拟 NMI

    /* Redistributor 寄存器 */
    uint32_t gicr_ctlr;
    uint64_t gicr_typer;
    uint32_t gicr_waker;
    uint32_t gicr_igroupr0;       // SGI/PPI 分组
    uint32_t gicr_ienabler0;      // SGI/PPI 使能
    uint32_t gicr_ipendr0;        // SGI/PPI 挂起
    uint32_t gicr_iactiver0;      // SGI/PPI 激活
    uint8_t gicr_ipriorityr[GIC_INTERNAL]; // SGI/PPI 优先级

    /* CPU Interface (ICC) 寄存器 */
    uint64_t icc_sre_el1;
    uint64_t icc_ctlr_el1[2];    // [S] 和 [NS]
    uint64_t icc_pmr_el1;         // Priority Mask
    uint64_t icc_bpr[3];          // Binary Point (G0/G1/G1NS)
    uint64_t icc_apr[3][4];       // Active Priority Registers
    uint64_t icc_igrpen[3];       // Group Enable

    /* 虚拟化控制 (ICH) */
    uint64_t ich_apr[3][4];
    uint64_t ich_hcr_el2;         // Hypervisor Control
    uint64_t ich_lr_el2[GICV3_LR_MAX]; // List Registers
    uint64_t ich_vmcr_el2;        // Virtual Machine Control

    /* 缓存：最高优先级挂起中断 */
    PendingIrq hppi;              // 物理 HPPI
    PendingIrq hpplpi;            // LPI HPPI
    PendingIrq hppvlpi;           // 虚拟 LPI HPPI

    int num_list_regs;            // List Register 数量
    int pribits;                  // 物理优先级位数
    int prebits;                  // 物理抢占位数
};
```

**`PendingIrq` 结构**（`arm_gicv3_common.h:120-125`）：

```c
typedef struct {
    int irq;        // 中断号
    uint8_t prio;   // 优先级
    int grp;        // 分组 (G0/G1/G1NS)
    bool nmi;       // NMI 标记
} PendingIrq;
```

---

## 5. 中断类型与编号空间

| 类型 | 编号范围 | 管理者 | 说明 |
|------|----------|--------|------|
| SGI | 0-15 | Redistributor | 软件生成中断 |
| PPI | 16-31 | Redistributor | 每 CPU 私有外设中断 |
| SPI | 32-1019 | Distributor | 共享外设中断 |
| LPI | 8192+ | Redistributor | 消息信号中断 (MSI) |

**GIC_INTERNAL = 32**：SGI + PPI 的总数，由 Redistributor 管理。

---

## 6. Distributor (GICD) 寄存器实现

`hw/intc/arm_gicv3_dist.c` 实现了所有 GICD 寄存器的读写。

**读写分发**（`arm_gicv3_dist.c:301-760`）：

```c
// arm_gicv3_dist.c:373-400 (gicd_readl 片段)
static bool gicd_readl(GICv3State *s, hwaddr offset,
                       uint64_t *data, MemTxAttrs attrs)
{
    switch (offset) {
    case GICD_CTLR:
        // NS 视图只能看到 ARE_S, EN_GRP1NS, RWP 位
        if (!attrs.secure && !(s->gicd_ctlr & GICD_CTLR_DS)) {
            *data = s->gicd_ctlr & (GICD_CTLR_ARE_S |
                                     GICD_CTLR_EN_GRP1NS |
                                     GICD_CTLR_RWP);
        } else {
            *data = s->gicd_ctlr;
        }
        return true;
    case GICD_ISENABLER ... GICD_ISENABLER + 0x7f:
        // 读使能位图
        ...
    }
}
```

**字节/字/双字/四字访问**（`arm_gicv3_dist.c:301-760`）：GICD 支持多种访问宽度，不同寄存器有不同的最小访问粒度。

---

## 7. GICD 位图管理与 NSACR 安全控制

```c
// arm_gicv3_dist.c:92-194 — 位图操作辅助函数

// 写 set-bitmap（ISENABLER/ISPENDR 等）
static void gicd_write_set_bitmap_reg(GICv3State *s, MemTxAttrs attrs,
                                       uint32_t *bmp, maskfn *maskfn,
                                       int offset, uint32_t val)
{
    int irq = offset * 8;
    // 跳过 SGI/PPI（< GIC_INTERNAL）和超范围中断
    if (irq < GIC_INTERNAL || irq >= s->num_irq) return;
    // NSACR 安全过滤：NS 不能操作安全中断位
    val &= mask_group_and_nsacr(s, attrs, maskfn, irq);
    *gic_bmp_ptr32(bmp, irq) |= val;  // 写 1 设置，写 0 忽略
    gicv3_update(s, irq, 32);         // 触发重计算
}

// PENDING 读取特殊处理：电平触发中断 = latch OR 电平
static uint32_t gicd_read_bitmap_reg(...)
{
    val = *gic_bmp_ptr32(bmp, irq);
    if (bmp == s->pending) {
        uint32_t edge = *gic_bmp_ptr32(s->edge_trigger, irq);
        uint32_t level = *gic_bmp_ptr32(s->level, irq);
        val |= (~edge & level);  // 电平触发 = 锁存 | 当前电平
    }
    val &= mask_group_and_nsacr(s, attrs, maskfn, irq);
    return val;
}
```

---

## 8. GICD_IROUTER 亲和路由

```c
// arm_gicv3_dist.c:264-286
static void gicd_write_irouter(GICv3State *s, MemTxAttrs attrs,
                                int irq, uint64_t val)
{
    // NSACR 安全检查
    ...
    s->gicd_irouter[irq] = val;
    gicv3_cache_target_cpustate(s, irq);  // 缓存目标 CPU 指针
    gicv3_update(s, irq, 1);             // 重新计算
}
```

QEMU 始终启用 affinity routing（`GICD_CTLR.ARE_S = 1`），因此 `GICD_ITARGETSR` 始终 RAZ/WI。路由通过 `GICD_IROUTER` 的 Aff3/Aff2/Aff1/Aff0 字段匹配 CPU。

---

## 9. Redistributor (GICR) 寄存器实现

`hw/intc/arm_gicv3_redist.c:322-709` 实现 GICR 寄存器。

**关键寄存器**：

| 寄存器 | 功能 | 实现位置 |
|--------|------|----------|
| GICR_TYPER | CPU 类型标识 | `arm_gicv3_redist.c:384+` |
| GICR_WAKER | 唤醒/睡眠控制 | `arm_gicv3_redist.c:369-526` |
| GICR_IGROUPR0 | SGI/PPI 分组 | `GICv3CPUState.gicr_igroupr0` |
| GICR_ISENABLER0 | SGI/PPI 使能 | `GICv3CPUState.gicr_ienabler0` |
| GICR_ISPENDR0 | SGI/PPI 挂起 | `GICv3CPUState.gicr_ipendr0` |
| GICR_IPRIORITYR | SGI/PPI 优先级 | `GICv3CPUState.gicr_ipriorityr[32]` |

---

## 10. LPI 支持

```c
// arm_gicv3_redist.c:99-170
static void update_for_one_lpi(GICv3CPUState *cs, int lpi, ...)
{
    // 从客户机内存读取 LPI 配置表和 Pending 表
    // 检查使能、优先级、分组
    // 更新 cs->hpplpi（最高优先级挂起 LPI）
}

static void update_for_all_lpis(GICv3CPUState *cs)
{
    // 扫描所有 LPI，找到最高优先级挂起的
    // 结果存入 cs->hpplpi
}
```

**GICR_PROPBASER**：指向客户机内存中的 LPI 配置表（每 LPI 一字节：优先级 + 使能 + 分组）。
**GICR_PENDBASER**：指向客户机内存中的 LPI Pending 位表。

---

## 11. CPU Interface (ICC) 系统寄存器

`hw/intc/arm_gicv3_cpuif.c` 实现所有 ICC/ICH 系统寄存器。GICv3 的 CPU Interface 通过 **AArch64 系统寄存器**（而非 MMIO）访问。

**注册方式**：通过 `ARMCPRegInfo` 数组注册到 CPU，使得 `MSR/MRS ICC_*_EL1` 指令自动路由到对应的读写函数。

**核心寄存器**：

| 寄存器 | 功能 | 实现 |
|--------|------|------|
| ICC_PMR_EL1 | Priority Mask | `icc_pmr_read/write` |
| ICC_IAR0_EL1 | Group 0 确认 | `icc_iar0_read` |
| ICC_IAR1_EL1 | Group 1 确认 | `icc_iar1_read` |
| ICC_EOIR0_EL1 | Group 0 结束 | `icc_eoir_write` |
| ICC_EOIR1_EL1 | Group 1 结束 | `icc_eoir_write` |
| ICC_HPPIR0/1_EL1 | 最高优先级挂起 | `icc_hppir0/1_read` |
| ICC_BPR0/1_EL1 | Binary Point | `icc_bpr_read/write` |
| ICC_CTLR_EL1 | 控制寄存器 | `icc_ctlr_el1_read/write` |
| ICC_IGRPEN0/1_EL1 | 分组使能 | `icc_igrpen_read/write` |
| ICC_SGI1R_EL1 | SGI 生成 | `icc_sgi1r_write` |
| ICC_DIR_EL1 | Deactivate | `icc_dir_write` |

---

## 12. icv_access() 虚拟化重定向

```c
// arm_gicv3_cpuif.c:85-106
static bool icv_access(CPUARMState *env, int hcr_flags)
{
    // 判断 ICC 寄存器访问是否应重定向到 ICV（虚拟接口）
    // 条件：NS EL1 且 HCR_EL2 中指定的 IMO/FMO 位被置位
    uint64_t hcr_el2 = arm_hcr_el2_eff(env);
    bool flagmatch = hcr_el2 & hcr_flags & (HCR_IMO | HCR_FMO);
    return flagmatch && arm_current_el(env) == 1
        && !arm_is_secure_below_el3(env);
}
```

**规则**：
- Group 0 寄存器（ICC_*0_EL1）：检查 `HCR_EL2.FMO`
- Group 1 寄存器（ICC_*1_EL1）：检查 `HCR_EL2.IMO`
- 通用寄存器（CTLR/DIR/PMR/RPR）：检查 `HCR_IMO | HCR_FMO`

当重定向时，物理 ICC 访问变为虚拟 ICV 访问，操作 List Registers 而非真实 GIC 状态。

---

## 13. 中断优先级与抢占模型

GICv3 使用 **8 位优先级**（0x00 最高，0xFF 最低/空闲），通过 Binary Point Register 将优先级分为：
- **Group Priority**：用于抢占判断
- **Subpriority**：同组内排序，不触发抢占

```
优先级值: [7:BPR+1] = Group Priority, [BPR:0] = Subpriority
```

---

## 14. irqbetter() 优先级比较

```c
// arm_gicv3.c:24-53
static bool irqbetter(GICv3CPUState *cs, int irq, uint8_t prio, bool nmi)
{
    // 优先级更高（值更小）者胜出
    if (prio != cs->hppi.prio) {
        return prio < cs->hppi.prio;
    }
    // 同优先级，NMI 优先
    if (nmi != cs->hppi.nmi) {
        return nmi;
    }
    // 同优先级同 NMI，中断号小的胜出（IMPDEF 选择）
    if (irq <= cs->hppi.irq) {
        return true;
    }
    return false;
}
```

---

## 15. icc_hppi_can_preempt() 抢占判断

```c
// arm_gicv3_cpuif.c:993-1044
static bool icc_hppi_can_preempt(GICv3CPUState *cs)
{
    // 1. 检查有无有效的挂起中断
    if (icc_no_enabled_hppi(cs)) return false;

    // 2. NMI 特殊处理：检查 PMR 是否允许
    if (cs->hppi.nmi) {
        // G1NS NMI 需要 PMR >= 0x80
        ...
    } else if (cs->hppi.prio >= cs->icc_pmr_el1) {
        return false;  // Priority Mask 屏蔽
    }

    // 3. 获取当前运行优先级
    rprio = icc_highest_active_prio(cs);
    if (rprio == 0xff) return true;  // 无活动中断，可抢占

    // 4. Group Priority 比较（屏蔽 subpriority）
    mask = icc_gprio_mask(cs, cs->hppi.grp);
    if ((cs->hppi.prio & mask) < (rprio & mask)) {
        return true;  // 更高组优先级
    }

    // 5. NMI 可以抢占同优先级非 NMI
    if (cs->hppi.nmi && (cs->hppi.prio & mask) == (rprio & mask)) {
        if (!(cs->icc_apr[cs->hppi.grp][0] & ICC_AP1R_EL1_NMI)) {
            return true;
        }
    }
    return false;
}
```

---

## 16. icc_highest_active_prio() 运行优先级

```c
// arm_gicv3_cpuif.c:910-945
static int icc_highest_active_prio(GICv3CPUState *cs)
{
    // NMI 支持：检查 APR 中的 NMI 位
    if (cs->nmi_support) {
        if (cs->icc_apr[GICV3_G1][0] & ICC_AP1R_EL1_NMI)
            return 0;  // NMI 活动 = 最高优先级
        if (cs->icc_apr[GICV3_G1NS][0] & ICC_AP1R_EL1_NMI)
            return (cs->gic->gicd_ctlr & GICD_CTLR_DS) ? 0 : 0x80;
    }

    // 扫描所有 APR，找最低设置位
    for (i = 0; i < icc_num_aprs(cs); i++) {
        uint32_t apr = cs->icc_apr[GICV3_G0][i] |
            cs->icc_apr[GICV3_G1][i] | cs->icc_apr[GICV3_G1NS][i];
        if (apr) {
            return (i * 32 + ctz32(apr)) << (icc_min_bpr(cs) + 1);
        }
    }
    return 0xff;  // 空闲优先级
}
```

**APR (Active Priority Register)**：位图，每位代表一个优先级组。`ctz32` 找到最低设置位 = 最高活动优先级。

---

## 17. gicv3_cpuif_update() 物理中断信号

```c
// arm_gicv3_cpuif.c:1046-1102
void gicv3_cpuif_update(GICv3CPUState *cs)
{
    int irqlevel = 0, fiqlevel = 0, nmilevel = 0;

    // G1 发送给无安全扩展的 CPU 时，降级为 G0
    if (cs->hppi.grp == GICV3_G1 && !arm_feature(env, ARM_FEATURE_EL3))
        cs->hppi.grp = GICV3_G0;

    if (icc_hppi_can_preempt(cs)) {
        bool isfiq;
        switch (cs->hppi.grp) {
        case GICV3_G0:   isfiq = true; break;         // G0→FIQ
        case GICV3_G1:   isfiq = !arm_is_secure(env)  // G1S→IRQ(NS) 或 FIQ(S@EL3)
                                 || (cur_el==3 && aa64); break;
        case GICV3_G1NS: isfiq = arm_is_secure(env); break; // G1NS→FIQ(S)
        }
        if (isfiq)           fiqlevel = 1;
        else if (cs->hppi.nmi) nmilevel = 1;
        else                 irqlevel = 1;
    }

    // 驱动 CPU 输入线
    qemu_set_irq(cs->parent_fiq, fiqlevel);   // → ARM_CPU_FIQ
    qemu_set_irq(cs->parent_irq, irqlevel);   // → ARM_CPU_IRQ
    qemu_set_irq(cs->parent_nmi, nmilevel);   // → ARM_CPU_NMI
}
```

---

## 18. Group 0/1 与 IRQ/FIQ 映射

| 中断组 | CPU 安全状态 | 信号 |
|--------|-------------|------|
| Group 0 | 任意 | **FIQ** |
| Group 1 Secure | Secure EL1/EL3(AA64) | IRQ |
| Group 1 Secure | Non-Secure | **FIQ**（安全中断） |
| Group 1 Non-Secure | Secure | **FIQ**（跨安全域） |
| Group 1 Non-Secure | Non-Secure | IRQ |

**核心规则**：Group 0 总是 FIQ；跨安全域的 Group 1 是 FIQ；同域的 Group 1 是 IRQ。

---

## 19. 完整中断注入路径：设备→GIC→CPU

```
设备（如 Timer）
  │ qemu_set_irq()
  ▼
gicv3_set_irq()              [arm_gicv3.c:373-400]
  ├── SPI → gicv3_dist_set_irq()   → 更新 Distributor 位图
  └── PPI → gicv3_redist_set_irq() → 更新 Redistributor 位图
           │
           ▼
    gicv3_update() / gicv3_redist_update()  [arm_gicv3.c:325-333]
           │  重新计算每 CPU 的 hppi
           ▼
    gicv3_cpuif_update()      [arm_gicv3_cpuif.c:1046-1102]
           │  Group→FIQ/IRQ/NMI 映射
           ▼
    qemu_set_irq(cs->parent_irq/fiq/nmi)
           │
           ▼
    arm_cpu_set_irq()          [target/arm/cpu.c:781-833]
           │  设置 CPU_INTERRUPT_HARD/FIQ/NMI 位
           ▼
    cpu_interrupt(cs, mask)    → 唤醒 vCPU 线程
```

---

## 20. gicv3_set_irq() 入口

```c
// arm_gicv3.c:373-400
static void gicv3_set_irq(void *opaque, int irq, int level)
{
    GICv3State *s = opaque;

    if (irq < (s->num_irq - GIC_INTERNAL)) {
        // 外部中断 (SPI) → Distributor
        gicv3_dist_set_irq(s, irq + GIC_INTERNAL, level);
    } else {
        // 每 CPU 中断 (PPI) → Redistributor
        int cpu = (irq - (s->num_irq - GIC_INTERNAL)) / GIC_INTERNAL;
        int pirq = irq % GIC_INTERNAL;
        assert(pirq >= GIC_NR_SGIS);  // SGI 不能通过这个路径
        gicv3_redist_set_irq(&s->cpu[cpu], pirq, level);
    }
}
```

---

## 21. gicv3_update() 全局重计算

```c
// arm_gicv3.c:257-333
static void gicv3_update_noirqset(GICv3State *s, int start, int len)
{
    // 遍历 [start, start+len) 范围的中断
    for (i = start; i < start + len; i++) {
        // 计算 pending 位图（每 32 个一批）
        pend = gicd_int_pending(s, i & ~0x1f);
        if (!(pend & (1 << (i & 0x1f)))) continue;

        // 获取目标 CPU（通过 IROUTER 缓存）
        cs = s->gicd_irouter_target[i];
        if (!cs) continue;  // 无目标

        // 获取优先级，比较是否更好
        nmi = gicv3_get_priority(cs, false, i, &prio);
        if (irqbetter(cs, i, prio, nmi)) {
            cs->hppi.irq = i;
            cs->hppi.prio = prio;
            cs->hppi.nmi = nmi;
            cs->seenbetter = true;
        }
    }

    // 如果之前的最优中断在范围内但没有更好的，需全量重算
    for (i = 0; i < s->num_cpu; i++) {
        if (!cs->seenbetter && cs->hppi.irq >= start && ...) {
            gicv3_full_update_noirqset(s);
            break;
        }
    }
}
```

**优化策略**：只重算变化范围内的中断，仅当之前的最优被移除时才触发全量重算（`gicv3_full_update_noirqset`）。

---

## 22. arm_cpu_set_irq() CPU 端接收

```c
// target/arm/cpu.c:781-833
static void arm_cpu_set_irq(void *opaque, int irq, int level)
{
    ARMCPU *cpu = opaque;
    static const int mask[] = {
        [ARM_CPU_IRQ]   = CPU_INTERRUPT_HARD,
        [ARM_CPU_FIQ]   = CPU_INTERRUPT_FIQ,
        [ARM_CPU_VIRQ]  = CPU_INTERRUPT_VIRQ,
        [ARM_CPU_VFIQ]  = CPU_INTERRUPT_VFIQ,
        [ARM_CPU_NMI]   = CPU_INTERRUPT_NMI,
        [ARM_CPU_VINMI] = CPU_INTERRUPT_VINMI,
    };

    // 记录到 irq_line_state
    if (level) env->irq_line_state |= mask[irq];
    else       env->irq_line_state &= ~mask[irq];

    switch (irq) {
    case ARM_CPU_VIRQ:  arm_cpu_update_virq(cpu); break;  // 合并 HCR.VI
    case ARM_CPU_VFIQ:  arm_cpu_update_vfiq(cpu); break;  // 合并 HCR.VF
    case ARM_CPU_VINMI: arm_cpu_update_vinmi(cpu); break;
    case ARM_CPU_IRQ:
    case ARM_CPU_FIQ:
    case ARM_CPU_NMI:
        if (level) cpu_interrupt(cs, mask[irq]);
        else       cpu_reset_interrupt(cs, mask[irq]);
        break;
    }
}
```

**VIRQ/VFIQ 特殊处理**：虚拟中断需要合并 `HCR_EL2.VI/VF` 位，因此通过 `arm_cpu_update_virq/vfiq()` 间接处理。

---

## 23. arm_cpu_has_work() 工作检查

```c
// target/arm/cpu.c:143-159
static bool arm_cpu_has_work(CPUState *cs)
{
    ARMCPU *cpu = ARM_CPU(cs);
    return (cpu->power_state != PSCI_OFF)
        && cpu_test_interrupt(cs,
               CPU_INTERRUPT_FIQ | CPU_INTERRUPT_HARD
               | CPU_INTERRUPT_NMI | CPU_INTERRUPT_VINMI | CPU_INTERRUPT_VFNMI
               | CPU_INTERRUPT_VFIQ | CPU_INTERRUPT_VIRQ | CPU_INTERRUPT_VSERR
               | CPU_INTERRUPT_EXITTB);
}
```

当 `cpu_has_work()` 返回 true 时，WFI/WFE 状态的 CPU 会被唤醒。

---

## 24. arm_cpu_exec_interrupt() 中断调度

```c
// target/arm/cpu-irq.c:171-274
bool arm_cpu_exec_interrupt(CPUState *cs, int interrupt_request)
{
    uint32_t cur_el = arm_current_el(env);
    bool secure = arm_is_secure(env);
    uint64_t hcr_el2 = arm_hcr_el2_eff(env);

    // 优先级顺序（IMPLEMENTATION DEFINED）：
    // NMI > VINMI > VFNMI > FIQ > IRQ > VIRQ > VFIQ > VSERR

    // 1. NMI 最高优先级
    if (interrupt_request & CPU_INTERRUPT_NMI) {
        excp_idx = EXCP_NMI;
        target_el = arm_phys_excp_target_el(cs, excp_idx, cur_el, secure);
        if (arm_excp_unmasked(cs, excp_idx, target_el, ...))
            goto found;
    }

    // 2. FIQ
    if (interrupt_request & CPU_INTERRUPT_FIQ) {
        excp_idx = EXCP_FIQ;
        target_el = arm_phys_excp_target_el(...);
        if (arm_excp_unmasked(...)) goto found;
    }

    // 3. IRQ
    if (interrupt_request & CPU_INTERRUPT_HARD) {
        excp_idx = EXCP_IRQ;
        ...
    }

    // 4. VIRQ (target_el 固定为 1)
    if (interrupt_request & CPU_INTERRUPT_VIRQ) {
        excp_idx = EXCP_VIRQ;
        target_el = 1;
        ...
    }

    // 5. VFIQ, VSERR...
    ...

found:
    cs->exception_index = excp_idx;
    env->exception.target_el = target_el;
    cs->cc->tcg_ops->do_interrupt(cs);  // → arm_cpu_do_interrupt()
    return true;
}
```

**NMI 降级**：如果 SCTLR.NMI 未使能，NMI 降级为普通 IRQ（`cpu-irq.c:208-222`）。

---

## 25. arm_excp_unmasked() 屏蔽检查

```c
// target/arm/cpu-irq.c:15-169
static inline bool arm_excp_unmasked(CPUState *cs, unsigned int excp_idx,
                                      unsigned int target_el,
                                      unsigned int cur_el, bool secure,
                                      uint64_t hcr_el2)
{
    // 1. 目标 EL < 当前 EL → 不能接收
    if (cur_el > target_el) return false;

    // 2. ALLINT 屏蔽检查（NMI 相关）
    if (SCTLR_NMI && cur_el == target_el) {
        allIntMask = PSTATE.ALLINT || (SCTLR.SPINTMASK && PSTATE.SP);
    }

    // 3. 根据异常类型检查 PSTATE 屏蔽位
    switch (excp_idx) {
    case EXCP_NMI:  pstate_unmasked = !allIntMask; break;
    case EXCP_FIQ:  pstate_unmasked = !(PSTATE.F); break;
    case EXCP_IRQ:  pstate_unmasked = !(PSTATE.I); break;
    case EXCP_VIRQ: // 需要 HCR_EL2.IMO 且非 TGE
                    if (!(hcr_el2 & HCR_IMO) || (hcr_el2 & HCR_TGE))
                        return false;
                    pstate_unmasked = !(PSTATE.I); break;
    case EXCP_VFIQ: // 需要 HCR_EL2.FMO 且非 TGE
                    ...
    }

    // 4. 升级到更高 EL 时忽略屏蔽位
    if (target_el > cur_el) unmasked = true;

    return unmasked || pstate_unmasked;
}
```

---

## 26. 异常入口：IRQ/FIQ 向量计算

```c
// target/arm/helper.c:9200-9428 (arm_cpu_do_interrupt_aarch64)

// 向量基址
vaddr addr = env->cp15.vbar_el[new_el];

// 来源 EL < 目标 EL：
if (cur_el < new_el) {
    if (lower_is_aa64) addr += 0x400;  // Lower EL, AArch64
    else               addr += 0x600;  // Lower EL, AArch32
}
// 同 EL：
else {
    if (PSTATE.SP)     addr += 0x200;  // SPx
    // else SP0: +0x000
}

// 异常类型偏移：
switch (exception_index) {
case EXCP_IRQ/VIRQ/NMI/VINMI:  addr += 0x80;  break;
case EXCP_FIQ/VFIQ/VFNMI:      addr += 0x100; break;
case EXCP_VSERR:                addr += 0x180; break;
}

// 保存状态
env->elr_el[new_el] = env->pc;           // 返回地址
env->banked_spsr[idx] = old_mode;         // 保存 PSTATE
pstate_write(env, PSTATE_DAIF | new_mode); // 设置新 PSTATE（屏蔽所有异常）
env->pc = addr;                            // 跳转到向量
```

**AArch64 异常向量表布局**（每个 EL 独立，以 `VBAR_ELn` 为基址）：

| 偏移 | SP0 同步 | SP0 IRQ | SP0 FIQ | SP0 SError |
|------|----------|---------|---------|------------|
| 0x000 | +0x000 | +0x080 | +0x100 | +0x180 |

| 偏移 | SPx 同步 | SPx IRQ | SPx FIQ | SPx SError |
|------|----------|---------|---------|------------|
| 0x200 | +0x200 | +0x280 | +0x300 | +0x380 |

| 偏移 | Lower AArch64 同步 | Lower AArch64 IRQ | ... |
|------|---------|---------|-----|
| 0x400 | +0x400 | +0x480 | +0x500, +0x580 |

| 偏移 | Lower AArch32 同步 | Lower AArch32 IRQ | ... |
|------|---------|---------|-----|
| 0x600 | +0x600 | +0x680 | +0x700, +0x780 |

---

## 27. 中断确认：ICC_IAR1_EL1 读取

```c
// arm_gicv3_cpuif.c:1285-1311
static uint64_t icc_iar1_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    // 虚拟化重定向
    if (icv_access(env, HCR_IMO))
        return icv_iar_read(env, ri);  // → 虚拟 IAR

    // 抢占检查
    if (!icc_hppi_can_preempt(cs))
        intid = INTID_SPURIOUS;        // 1023 = 无中断
    else
        intid = icc_hppir1_value(cs, env); // 获取 HPPI 中断号

    if (!gicv3_intid_is_special(intid)) {
        if (cs->hppi.nmi && SCTLR.NMI)
            intid = INTID_NMI;         // NMI 返回特殊 ID
        else
            icc_activate_irq(cs, intid); // 转为 Active 状态
    }
    return intid;
}
```

---

## 28. icc_activate_irq() 状态转换

```c
// arm_gicv3_cpuif.c:1158-1187
static void icc_activate_irq(GICv3CPUState *cs, int irq)
{
    // 计算 APR 位位置
    uint32_t mask = icc_gprio_mask(cs, cs->hppi.grp);
    int prio = cs->hppi.prio & mask;
    int aprbit = prio >> (8 - cs->prebits);

    // 设置 Active Priority 位
    if (cs->hppi.nmi)
        cs->icc_apr[cs->hppi.grp][0] |= ICC_AP1R_EL1_NMI;
    else
        cs->icc_apr[cs->hppi.grp][aprbit/32] |= (1U << (aprbit%32));

    // 更新中断状态位
    if (irq < GIC_INTERNAL) {
        // SGI/PPI：设置 active，清除 pending
        cs->gicr_iactiver0 = deposit32(cs->gicr_iactiver0, irq, 1, 1);
        cs->gicr_ipendr0 = deposit32(cs->gicr_ipendr0, irq, 1, 0);
        gicv3_redist_update(cs);
    } else if (irq < GICV3_LPI_INTID_START) {
        // SPI：同样设置 active，清除 pending
        gicv3_gicd_active_set(cs->gic, irq);
        gicv3_gicd_pending_clear(cs->gic, irq);
        gicv3_update(cs->gic, irq, 1);
    } else {
        // LPI：清除 pending bit in 内存
        gicv3_redist_lpi_pending(cs, irq, 0);
    }
}
```

**状态转换**：Pending → Active（对于电平触发，如果电平仍高则保持 Pending+Active）。

---

## 29. 中断结束：ICC_EOIR1_EL1 写入

```c
// arm_gicv3_cpuif.c:1645-1714
static void icc_eoir_write(CPUARMState *env, const ARMCPRegInfo *ri,
                            uint64_t value)
{
    int irq = value & 0xffffff;
    bool is_eoir0 = ri->crm == 8;  // EOIR0 vs EOIR1

    // 虚拟化重定向
    if (icv_access(env, is_eoir0 ? HCR_FMO : HCR_IMO)) {
        icv_eoir_write(env, ri, value);
        return;
    }

    // 验证中断号有效性
    if (irq >= cs->gic->num_irq && ...) return;

    // 确认分组匹配
    grp = icc_highest_active_group(cs);
    switch (grp) {
    case GICV3_G0:
        if (!is_eoir0) return;                    // G0 必须用 EOIR0
        if (!DS && !arm_is_secure(env)) return;   // NS 不能 EOI 安全中断
        break;
    case GICV3_G1NS:
        if (is_eoir0) return;                     // G1 必须用 EOIR1
        break;
    }

    // 优先级下降
    icc_drop_prio(cs, grp);

    // 非 Split EOI 模式：同时去激活
    if (!icc_eoi_split(env, cs)) {
        icc_deactivate_irq(cs, irq);
    }
}
```

---

## 30. icc_drop_prio() 优先级下降

```c
// arm_gicv3_cpuif.c:1340-1379
static void icc_drop_prio(GICv3CPUState *cs, int grp)
{
    for (i = 0; i < icc_num_aprs(cs); i++) {
        uint64_t *papr = &cs->icc_apr[grp][i];
        if (!*papr) continue;

        // NMI 位特殊处理
        if (i == 0 && cs->nmi_support && (*papr & ICC_AP1R_EL1_NMI)) {
            *papr &= ~ICC_AP1R_EL1_NMI;
            break;
        }

        // 清除最低设置位 = 最高活动优先级
        *papr &= *papr - 1;
        break;
    }
    gicv3_cpuif_update(cs);  // 运行优先级改变，重新评估
}
```

---

## 31. Split EOI 模式与 ICC_DIR

```c
// arm_gicv3_cpuif.c:1381-1394
static bool icc_eoi_split(CPUARMState *env, GICv3CPUState *cs)
{
    if (arm_is_el3_or_mon(env))
        return cs->icc_ctlr_el3 & ICC_CTLR_EL3_EOIMODE_EL3;
    if (arm_is_secure_below_el3(env))
        return cs->icc_ctlr_el1[GICV3_S] & ICC_CTLR_EL1_EOIMODE;
    else
        return cs->icc_ctlr_el1[GICV3_NS] & ICC_CTLR_EL1_EOIMODE;
}
```

**Split EOI**：当 `ICC_CTLR.EOImode = 1` 时，EOIR 只做优先级下降，去激活需要显式写 `ICC_DIR_EL1`。用于 Hypervisor 拦截中断完成。

---

## 32. 虚拟中断 (EL2)：ICH 接口

**List Registers (ICH_LR_EL2)**（`arm_gicv3_common.h:175`）：
- 每个 LR 描述一个虚拟中断：中断号、优先级、分组、状态（Invalid/Pending/Active/P+A）
- 最多 `GICV3_LR_MAX` 个 LR（通常 16）

**ICH_HCR_EL2**：控制虚拟 CPU Interface
- `EN`：使能虚拟 CPU Interface
- `UIE`：Underflow Interrupt Enable（活动 LR < 阈值时触发维护中断）

**ICH_VMCR_EL2**：虚拟 Machine Control（虚拟 PMR、BPR、CTLR 镜像）

---

## 33. gicv3_cpuif_virt_irq_fiq_update() 虚拟信号

```c
// arm_gicv3_cpuif.c:471-524
void gicv3_cpuif_virt_irq_fiq_update(GICv3CPUState *cs)
{
    int irqlevel = 0, fiqlevel = 0, nmilevel = 0;

    idx = hppvi_index(cs);  // 找最高优先级虚拟中断

    if (idx == HPPVI_INDEX_VLPI) {
        // 虚拟 LPI
        if (icv_hppvlpi_can_preempt(cs)) {
            if (cs->hppvlpi.grp == GICV3_G0) fiqlevel = 1;
            else irqlevel = 1;
        }
    } else if (idx >= 0) {
        uint64_t lr = cs->ich_lr_el2[idx];
        if (icv_hppi_can_preempt(cs, lr)) {
            if (lr & ICH_LR_EL2_GROUP) {
                if (lr & ICH_LR_EL2_NMI) nmilevel = 1;
                else irqlevel = 1;
            } else {
                fiqlevel = 1;  // G0→vFIQ
            }
        }
    }

    qemu_set_irq(cs->parent_vfiq, fiqlevel);
    qemu_set_irq(cs->parent_virq, irqlevel);
    qemu_set_irq(cs->parent_vnmi, nmilevel);
}
```

**维护中断**（`arm_gicv3_cpuif.c:526-558`）：

```c
static void gicv3_cpuif_virt_update(GICv3CPUState *cs)
{
    gicv3_cpuif_virt_irq_fiq_update(cs);

    // 维护中断（LR 为空/Underflow 时通知 Hypervisor）
    if ((cs->ich_hcr_el2 & ICH_HCR_EL2_EN) &&
        maintenance_interrupt_state(cs) != 0) {
        maintlevel = 1;
    }
    qemu_set_irq(cpu->gicv3_maintenance_interrupt, maintlevel);
}
```

---

## 34. SGI 软件生成中断

Guest 写入 `ICC_SGI1R_EL1` 触发跨 CPU 中断：

1. 解析 Aff3/Aff2/Aff1 亲和字段确定目标 CPU
2. 如果 `IRM = 1`，发送给所有其他 CPU
3. 调用 `gicv3_redist_send_sgi()` 设置目标 CPU 的 GICR 挂起位
4. 触发目标 CPU 的 `gicv3_redist_update()` → `gicv3_cpuif_update()`

---

## 35. GIC KVM 支持

`hw/intc/arm_gicv3_kvm.c` 实现 KVM 内核态 GIC。

**KVM 访问辅助函数**（`arm_gicv3_kvm.c:87-119`）：

```c
// Distributor 寄存器访问
static inline void kvm_gicd_access(GICv3State *s, int offset,
                                    uint32_t *val, bool write)
{
    kvm_device_access(s->dev_fd, KVM_DEV_ARM_VGIC_GRP_DIST_REGS,
                      KVM_VGIC_ATTR(offset, 0), val, write, &error_abort);
}

// Redistributor 寄存器访问
static inline void kvm_gicr_access(GICv3State *s, int offset, int cpu,
                                    uint32_t *val, bool write)
{
    kvm_device_access(s->dev_fd, KVM_DEV_ARM_VGIC_GRP_REDIST_REGS,
                      KVM_VGIC_ATTR(offset, s->cpu[cpu].gicr_typer),
                      val, write, &error_abort);
}

// CPU 系统寄存器访问
static inline void kvm_gicc_access(GICv3State *s, uint64_t reg, int cpu,
                                    uint64_t *val, bool write)
{
    kvm_device_access(s->dev_fd, KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS,
                      KVM_VGIC_ATTR(reg, s->cpu[cpu].gicr_typer),
                      val, write, &error_abort);
}
```

---

## 36. KVM 设备创建与初始化

```c
// arm_gicv3_kvm.c:786-835
static void kvm_arm_gicv3_realize(DeviceState *dev, Error **errp)
{
    // 验证 revision == 3
    // 验证不支持安全扩展和 NMI（内核 GIC 限制）

    gicv3_init_irqs_and_mmio(s, kvm_arm_gicv3_set_irq, NULL);

    // 注册 ICC 系统寄存器到每个 CPU
    for (i = 0; i < s->num_cpu; i++) {
        define_arm_cp_regs(cpu, gicv3_cpuif_reginfo);
    }

    // 创建 KVM VGIC 设备
    s->dev_fd = kvm_create_device(kvm_state,
                                   KVM_DEV_TYPE_ARM_VGIC_V3, false);

    // 设置 Distributor 和 Redistributor 地址
    // 设置 LR 数量等能力
}
```

---

## 37. KVM 状态同步

**保存状态**（`kvm_arm_gicv3_get`，`arm_gicv3_kvm.c:502-657`）：
- 从内核读取 GICD/GICR/ICC 全部寄存器状态到 QEMU 的 GICv3State

**恢复状态**（`kvm_arm_gicv3_put`，`arm_gicv3_kvm.c:317-500`）：
- 将 QEMU 的 GICv3State 写入内核 VGIC

用于：迁移前保存 / 迁移后恢复 / 调试时检查。

---

## 38. Timer→GIC 中断连接示例

ARM Generic Timer 是最典型的中断源。连接方式：

```
// Board 代码中（如 hw/arm/virt.c）：

// 1. Timer 输出连接到 GIC PPI 输入
// 物理 Timer → PPI 30 (ARCH_TIMER_NS_EL1_IRQ)
// 虚拟 Timer → PPI 27 (ARCH_TIMER_VIRT_IRQ)
// Hypervisor Timer → PPI 26

// 2. GIC IRQ/FIQ/VIRQ/VFIQ 输出连接回 CPU
sysbus_connect_irq(gicbusdev, i,
                    qdev_get_gpio_in(cpudev, ARM_CPU_IRQ));
sysbus_connect_irq(gicbusdev, i + num_cpu,
                    qdev_get_gpio_in(cpudev, ARM_CPU_FIQ));
sysbus_connect_irq(gicbusdev, i + 2*num_cpu,
                    qdev_get_gpio_in(cpudev, ARM_CPU_VIRQ));
sysbus_connect_irq(gicbusdev, i + 3*num_cpu,
                    qdev_get_gpio_in(cpudev, ARM_CPU_VFIQ));
```

**Timer 中断触发路径**：
```
gt_timer_fired()  → qemu_set_irq(cpu->gt_timer_outputs[timeridx])
  → gicv3_set_irq() (PPI 路径)
    → gicv3_redist_set_irq()
      → gicv3_cpuif_update()
        → qemu_set_irq(cs->parent_irq)
          → arm_cpu_set_irq()
            → cpu_interrupt(cs, CPU_INTERRUPT_HARD)
```

---

## 39. 关键数据结构总览表

| 结构/类型 | 文件 | 行号 | 说明 |
|-----------|------|------|------|
| `GICv3State` | arm_gicv3_common.h | 225-270 | 全局 GIC 状态 |
| `GICv3CPUState` | arm_gicv3_common.h | 127-213 | 每 CPU GIC 状态 |
| `PendingIrq` | arm_gicv3_common.h | 120-125 | 挂起中断描述 |
| `TYPE_ARM_GICV3` | arm_gicv3.c | 465-478 | QOM 类型注册 |
| `irqbetter()` | arm_gicv3.c | 24-53 | 优先级比较 |
| `gicv3_set_irq()` | arm_gicv3.c | 373-400 | 中断输入入口 |
| `gicv3_update()` | arm_gicv3.c | 325-333 | 全局重计算 |
| `gicv3_update_noirqset()` | arm_gicv3.c | 257-323 | 范围重计算 |
| `gicv3_cpuif_update()` | arm_gicv3_cpuif.c | 1046-1102 | 物理中断信号 |
| `gicv3_cpuif_virt_irq_fiq_update()` | arm_gicv3_cpuif.c | 471-524 | 虚拟中断信号 |
| `icv_access()` | arm_gicv3_cpuif.c | 85-106 | 虚拟化重定向 |
| `icc_hppi_can_preempt()` | arm_gicv3_cpuif.c | 993-1044 | 抢占判断 |
| `icc_highest_active_prio()` | arm_gicv3_cpuif.c | 910-945 | 运行优先级 |
| `icc_iar1_read()` | arm_gicv3_cpuif.c | 1285-1311 | 中断确认 |
| `icc_activate_irq()` | arm_gicv3_cpuif.c | 1158-1187 | 激活中断 |
| `icc_eoir_write()` | arm_gicv3_cpuif.c | 1645-1714 | 中断结束 |
| `icc_drop_prio()` | arm_gicv3_cpuif.c | 1340-1379 | 优先级下降 |
| `arm_cpu_set_irq()` | cpu.c | 781-833 | CPU 端接收 |
| `arm_cpu_has_work()` | cpu.c | 143-159 | 工作检查 |
| `arm_cpu_exec_interrupt()` | cpu-irq.c | 171-274 | 中断调度 |
| `arm_excp_unmasked()` | cpu-irq.c | 15-169 | 屏蔽检查 |
| `arm_cpu_do_interrupt_aarch64()` | helper.c | 9200-9428 | 异常入口 |
| `arm_cpu_do_interrupt()` | helper.c | 9469-9530 | 异常分发 |

---

## 40. 端到端中断流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          设备 (Timer/UART/...)                          │
│                         qemu_set_irq(irq, level)                       │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    gicv3_set_irq() [arm_gicv3.c:373]                   │
│  ┌──────────────────────┐    ┌──────────────────────────────┐          │
│  │ SPI (irq 32-1019)    │    │ PPI/SGI (irq 0-31, per-CPU) │          │
│  │ gicv3_dist_set_irq() │    │ gicv3_redist_set_irq()      │          │
│  │ 更新 GICD 位图        │    │ 更新 GICR 位图               │          │
│  └──────────┬───────────┘    └──────────────┬───────────────┘          │
│             │                               │                          │
│             ▼                               ▼                          │
│      gicv3_update()                  gicv3_redist_update()             │
│      [arm_gicv3.c:325]              重计算每 CPU hppi                   │
│      重计算 SPI→CPU 映射                                                │
│             │                               │                          │
│             └───────────────┬───────────────┘                          │
│                             ▼                                          │
│              gicv3_cpuif_update() [cpuif.c:1046]                       │
│              icc_hppi_can_preempt() → Group→FIQ/IRQ/NMI                │
│              qemu_set_irq(parent_fiq/irq/nmi)                          │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│               arm_cpu_set_irq() [cpu.c:781]                            │
│               env->irq_line_state |= mask[irq]                         │
│               cpu_interrupt(cs, CPU_INTERRUPT_HARD/FIQ/NMI)             │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │ vCPU 线程被唤醒
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│          arm_cpu_exec_interrupt() [cpu-irq.c:171]                      │
│          优先级：NMI > FIQ > IRQ > VIRQ > VFIQ > VSERR                 │
│          arm_excp_unmasked(): PSTATE.I/F/ALLINT 检查                    │
│          arm_phys_excp_target_el(): SCR/HCR 路由                        │
│                             │                                          │
│                             ▼                                          │
│          arm_cpu_do_interrupt() [helper.c:9469]                        │
│            → arm_cpu_do_interrupt_aarch64() [helper.c:9200]            │
│            向量计算: VBAR_ELn + EL偏移 + 类型偏移                       │
│            保存: SPSR_ELn = PSTATE, ELR_ELn = PC                       │
│            设置: PSTATE.DAIF, 切换到目标 EL                              │
│            跳转: PC = vector_address                                    │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │ Guest 进入中断处理程序
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     Guest 中断处理流程                                   │
│                                                                         │
│  1. MRS X0, ICC_IAR1_EL1  →  icc_iar1_read() [cpuif.c:1285]           │
│     返回中断号, 状态: Pending→Active                                     │
│     APR 更新: 运行优先级提高                                              │
│                                                                         │
│  2. 处理中断 (读设备寄存器, 清中断源等)                                    │
│                                                                         │
│  3. MSR ICC_EOIR1_EL1, X0  →  icc_eoir_write() [cpuif.c:1645]         │
│     优先级下降: icc_drop_prio()                                          │
│     去激活: icc_deactivate_irq() (非 Split EOI)                          │
│     状态: Active→Inactive                                                │
│                                                                         │
│  4. ERET  →  返回被中断的上下文                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 41. GICv2 vs GICv3 对比

| 特性 | GICv2 | GICv3 |
|------|-------|-------|
| 最大 CPU 数 | 8 | 理论无限（亲和路由） |
| CPU Interface | MMIO (GICC) | 系统寄存器 (ICC_*_EL1) |
| 路由方式 | GICD_ITARGETSR (8-bit 位图) | GICD_IROUTER (亲和字段) |
| 中断分组 | Group 0/1 | Group 0/1S/1NS |
| LPI 支持 | ❌ | ✅ (MSI) |
| NMI 支持 | ❌ | ✅ (FEAT_GICv3_NMI) |
| 虚拟化 | GICH (MMIO) | ICH (系统寄存器) |
| QEMU 实现 | arm_gic.c | arm_gicv3*.c (6 个文件) |
| KVM 支持 | arm_gic_kvm.c | arm_gicv3_kvm.c |

---

**适合读者**：需要理解 ARM 中断控制器架构、QEMU GIC 设备模型、中断注入完整路径的开发者。

**关键源文件**：
- `hw/intc/arm_gicv3.c`（~480行）— 核心逻辑与中断入口
- `hw/intc/arm_gicv3_cpuif.c`（~2500行）— CPU Interface/虚拟化接口
- `hw/intc/arm_gicv3_dist.c`（~760行）— Distributor 寄存器
- `hw/intc/arm_gicv3_redist.c`（~900行）— Redistributor 寄存器
- `hw/intc/arm_gicv3_kvm.c`（~950行）— KVM 支持
- `include/hw/intc/arm_gicv3_common.h`（~300行）— 核心数据结构
- `target/arm/cpu.c`（~2500行）— CPU 中断接收
- `target/arm/cpu-irq.c`（~280行）— 中断调度
- `target/arm/helper.c`（~10200行）— 异常入口
