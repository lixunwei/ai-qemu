# ARM64 GICv3 中断控制器深度分析：Distributor/Redistributor/CPU Interface、中断路由与优先级

> 基于 QEMU 11.0.50 源码分析，涵盖 ARM GICv3 通用中断控制器完整子系统：
> GICv3State 核心状态（num_cpu/num_irq/gicd_ctlr/GIC_DECLARE_BITMAP group/enabled/pending/active 8 位图、
> gicd_ipriority[MAXIRQ]/gicd_irouter[MAXIRQ]/gicd_irouter_target[MAXIRQ] 路由缓存、
> GICv3CPUState 每 CPU 状态 gicr_*/icc_*/ich_*/hppi/hpplpi/hppvlpi）、
> Distributor GICD（gicv3_dist_read/write MMIO：GICD_CTLR S/NS 视图分离、
> GICD_TYPER ITLinesNumber/SecurityExtn/LPIS/NMI、GICD_IGROUPR/IGRPMODR 三组分配、
> GICD_ISENABLER/ICENABLER/ISPENDR/ICPENDR/ISACTIVER/ICACTIVER 位图操作、
> GICD_IPRIORITYR 优先级、GICD_IROUTER 亲和性路由→gicv3_cache_target_cpustate）、
> Redistributor GICR（gicv3_redist_read/write：GICR_CTLR EnableLPIs、GICR_TYPER/WAKER、
> SGI/PPI 32 位寄存器 GICR_IENABLER0/IPENDR0/IACTIVER0/IPRIORITYR[32]、
> GICR_VPROPBASER/VPENDBASER vLPI 表）、
> CPU Interface ICC 系统寄存器（icc_pmr_read/write PMR 优先级掩码、
> icc_iar0/1_read IAR 中断确认→icc_activate_irq、icc_eoir_write EOI 结束→icc_drop_prio+icc_deactivate_irq、
> icc_hppir0/1_read HPPIR 最高优先级挂起、ICC_BPR0/1 二进制点→icc_gprio_mask 组优先级、
> ICC_CTLR_EL1/3 控制、ICC_SRE 系统寄存器使能、icc_sgi0r/1r_write SGI 生成→icc_generate_sgi、
> ICC_AP0R/1R 活跃优先级、ICC_RPR 运行优先级）、
> 优先级与抢占（irqbetter 最高优先级选择 prio<hppi.prio||同优先级取低 IRQ 号、
> icc_gprio_mask BPR→组优先级掩码、icc_hppi_can_preempt PMR 过滤+运行优先级比较+NMI 特殊处理、
> gicv3_redist_update_noirqset SGI/PPI 扫描→hppi、gicv3_update_noirqset SPI 分发→per-CPU hppi）、
> 中断信号传递（gicv3_cpuif_update：G0→FIQ/G1→视安全状态 FIQ 或 IRQ/G1NS→Secure 时 FIQ 否则 IRQ、
> qemu_set_irq parent_fiq/parent_irq/parent_nmi→CPU 异常输入）、
> 安全模型（gicv3_use_ns_bank→arm_is_secure_below_el3、
> GICD_CTLR DS 位简化安全/Group0 FIQ→EL3/Group1S Secure IRQ/Group1NS NS IRQ、
> gicd_int_pending EN_GRP0/GRP1S/GRP1NS 组使能过滤）、
> 虚拟接口（ICH_HCR_EL2 使能/ICH_VMCR_EL2 虚拟 PMR+BPR+VENG0/1、
> ICH_LR_EL2 列表寄存器 vINTID/pINTID/Priority/NMI/Group/HW/State 四状态、
> hppvi_index 最高优先级虚拟中断选择、gicv3_cpuif_virt_update vIRQ/vFIQ 信号+维护中断、
> icv_iar_read/icv_eoir_write 虚拟 IAR/EOI）、
> 中断生命周期（gicv3_set_irq GPIO→SPI:gicv3_dist_set_irq/PPI:gicv3_redist_set_irq→
> pending 位设置→gicv3_update→gicv3_cpuif_update→qemu_set_irq→CPU 异常→
> ICC_IAR1 读→icc_activate_irq→处理→ICC_EOIR1 写→deactivate）。

---

## 目录

1. [架构概述](#1-架构概述)
2. [GICv3State 与 GICv3CPUState](#2-gicv3state-与-gicv3cpustate)
3. [QOM 设备模型与 MMIO](#3-qom-设备模型与-mmio)
4. [Distributor (GICD)](#4-distributor-gicd)
5. [Redistributor (GICR)](#5-redistributor-gicr)
6. [中断状态位图与组管理](#6-中断状态位图与组管理)
7. [亲和性路由 (ARE)](#7-亲和性路由-are)
8. [CPU Interface ICC 系统寄存器](#8-cpu-interface-icc-系统寄存器)
9. [优先级与抢占机制](#9-优先级与抢占机制)
10. [中断信号传递](#10-中断信号传递)
11. [安全模型与组路由](#11-安全模型与组路由)
12. [虚拟接口 (GICv3 Virtualization)](#12-虚拟接口-gicv3-virtualization)
13. [SGI 生成](#13-sgi-生成)
14. [中断完整生命周期](#14-中断完整生命周期)
15. [关键数据流总结](#15-关键数据流总结)

---

## 1. 架构概述

GICv3（Generic Interrupt Controller v3）是 ARM 架构的标准中断控制器，由三大组件构成：

```
                    ┌─────────────────────────────────────┐
  外设 SPI ───────→ │         Distributor (GICD)           │
                    │  路由 SPI 到目标 CPU（IROUTER）      │
                    └───────────┬─────────────────────────┘
                                │ per-CPU
                    ┌───────────▼─────────────────────────┐
  PPI/SGI ────────→ │       Redistributor (GICR)          │
  LPI ────────────→ │  管理 per-CPU 中断（SGI/PPI/LPI）   │
                    └───────────┬─────────────────────────┘
                                │ hppi (最高优先级挂起)
                    ┌───────────▼─────────────────────────┐
                    │       CPU Interface (ICC)            │
                    │  优先级过滤、抢占、确认/结束          │
                    │  → IRQ/FIQ/NMI 信号 → CPU           │
                    └─────────────────────────────────────┘
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `arm_gicv3_common.h` | 127-213 | GICv3CPUState 结构 |
| `arm_gicv3_common.h` | 225-281 | GICv3State 结构 |
| `arm_gicv3_common.c` | 314-370 | gicv3_init_irqs_and_mmio |
| `arm_gicv3_common.c` | 372-492 | arm_gicv3_common_realize |
| `arm_gicv3.c` | 24-53 | irqbetter 优先级比较 |
| `arm_gicv3.c` | 55-99 | gicd_int_pending SPI 挂起计算 |
| `arm_gicv3.c` | 175-370 | gicv3_redist/full_update |
| `arm_gicv3.c` | 372-399 | gicv3_set_irq 外部中断输入 |
| `arm_gicv3_dist.c` | 381-825 | gicd_readl/writel |
| `arm_gicv3_redist.c` | 322-647 | gicr_readl/writel |
| `arm_gicv3_cpuif.c` | 947-1101 | 优先级/抢占/cpuif_update |
| `arm_gicv3_cpuif.c` | 1262-1310 | icc_iar0/1_read |
| `arm_gicv3_cpuif.c` | 1645-1713 | icc_eoir_write |
| `arm_gicv3_cpuif.c` | 179-558 | 虚拟接口 |
| `gicv3_internal.h` | 202-262 | ICH_* 寄存器位定义 |

---

## 2. GICv3State 与 GICv3CPUState

### GICv3State（全局状态）

```c
// include/hw/intc/arm_gicv3_common.h:225-281
struct GICv3State {
    SysBusDevice parent_obj;

    MemoryRegion iomem_dist;               // 230: GICD MMIO 区域
    GICv3RedistRegion *redist_regions;     // 231: GICR 区域数组
    uint32_t nb_redist_regions;            // 233: GICR 区域数

    uint32_t num_cpu;                      // 236: CPU 数量
    uint32_t num_irq;                      // 237: 中断总数
    bool security_extn;                    // 242: 安全扩展支持
    bool nmi_support;                      // 241: NMI 支持

    // === Distributor 状态 ===
    uint32_t gicd_ctlr;                    // 258: 控制寄存器
    GIC_DECLARE_BITMAP(group);             // 260: IGROUPR 组分配
    GIC_DECLARE_BITMAP(grpmod);            // 261: IGRPMODR 组修改
    GIC_DECLARE_BITMAP(enabled);           // 262: ISENABLER 使能
    GIC_DECLARE_BITMAP(pending);           // 263: ISPENDR 挂起
    GIC_DECLARE_BITMAP(active);            // 264: ISACTIVER 活跃
    GIC_DECLARE_BITMAP(level);             // 265: 当前电平
    GIC_DECLARE_BITMAP(edge_trigger);      // 266: 边沿/电平触发
    GIC_DECLARE_BITMAP(nmi);               // 267: NMI 标记
    uint8_t gicd_ipriority[GICV3_MAXIRQ];  // 268: 优先级
    uint64_t gicd_irouter[GICV3_MAXIRQ];   // 269: 路由寄存器
    GICv3CPUState *gicd_irouter_target[GICV3_MAXIRQ]; // 273: 路由缓存

    GICv3CPUState *cpu;                    // 276: per-CPU 状态数组
};
```

### GICv3CPUState（每 CPU 状态）

```c
// include/hw/intc/arm_gicv3_common.h:127-213
struct GICv3CPUState {
    GICv3State *gic;                       // 128: 反向引用
    CPUState *cpu;                         // 129: CPU 对象
    qemu_irq parent_irq;                   // 130: IRQ 输出
    qemu_irq parent_fiq;                   // 131: FIQ 输出
    qemu_irq parent_virq;                  // 132: vIRQ 输出
    qemu_irq parent_vfiq;                  // 133: vFIQ 输出
    qemu_irq parent_nmi;                   // 134: NMI 输出
    qemu_irq parent_vnmi;                  // 135: vNMI 输出

    // === Redistributor ===
    uint32_t level;                        // 138: 当前 IRQ 电平
    uint32_t gicr_ctlr;                    // 140
    uint64_t gicr_typer;                   // 141
    uint32_t gicr_waker;                   // 143
    uint64_t gicr_propbaser;               // 144: LPI 配置表基址
    uint64_t gicr_pendbaser;               // 145: LPI 挂起表基址
    uint32_t gicr_igroupr0;                // 147: SGI/PPI 组
    uint32_t gicr_ienabler0;               // 148: SGI/PPI 使能
    uint32_t gicr_ipendr0;                 // 149: SGI/PPI 挂起
    uint32_t gicr_iactiver0;               // 150: SGI/PPI 活跃
    uint8_t gicr_ipriorityr[GIC_INTERNAL]; // 155: SGI/PPI 优先级 (32 个)

    // === CPU Interface ===
    uint64_t icc_sre_el1;                  // 161: SRE 使能
    uint64_t icc_ctlr_el1[2];             // 162: 控制（S/NS 银行）
    uint64_t icc_pmr_el1;                  // 163: 优先级掩码
    uint64_t icc_bpr[3];                   // 164: 二进制点 [G0/G1/G1NS]
    uint64_t icc_apr[3][4];                // 165: 活跃优先级寄存器
    uint64_t icc_igrpen[3];                // 166: 组使能 [G0/G1/G1NS]
    uint64_t icc_ctlr_el3;                 // 167: EL3 控制

    // === Virtualization ===
    uint64_t ich_apr[3][4];                // 173: 虚拟活跃优先级
    uint64_t ich_hcr_el2;                  // 174: 虚拟控制
    uint64_t ich_lr_el2[GICV3_LR_MAX];    // 175: 列表寄存器
    uint64_t ich_vmcr_el2;                 // 176: 虚拟机控制
    int num_list_regs;                     // 183: 列表寄存器数

    // === 缓存 ===
    PendingIrq hppi;                       // 193: 最高优先级物理挂起
    PendingIrq hpplpi;                     // 199: 最高优先级 LPI
    PendingIrq hppvlpi;                    // 202: 最高优先级 vLPI
};
```

---

## 3. QOM 设备模型与 MMIO

```c
// arm_gicv3_common.h:311-314
#define TYPE_ARM_GICV3_COMMON "arm-gicv3-common"

// arm_gicv3_common.c:314-370 — gicv3_init_irqs_and_mmio
//   GPIO 输入: [0..N-1] SPI + [N..N+31*num_cpu] per-CPU PPI
//   输出 IRQ 线: parent_irq/fiq/virq/vfiq/nmi/vnmi × num_cpu
//   MMIO: iomem_dist (0x10000) + redist_regions (per-CPU × 0x20000)

// arm_gicv3_common.c:372-492 — arm_gicv3_common_realize
//   验证 num_irq (32-1020), num_cpu
//   分配 cpu[] 数组, 初始化每 CPU gicr_typer（亲和性编码）
//   设置 pribits, prebits, num_list_regs
```

---

## 4. Distributor (GICD)

### MMIO 读（gicd_readl）

```c
// arm_gicv3_dist.c:381-597
static bool gicd_readl(GICv3State *s, hwaddr offset,
                       uint64_t *data, MemTxAttrs attrs)
{
    GICD_CTLR (382-402):     // S/NS 视图分离
        NS && !DS → 仅暴露 ARE_S, EN_GRP1NS, RWP
        Secure → 完整 gicd_ctlr

    GICD_TYPER (403-432):    // 中断线数、安全扩展、LPI、NMI
        ITLinesNumber = (num_irq/32) - 1
        SecurityExtn = !(gicd_ctlr & DS)

    GICD_ISENABLER (472-478): gicd_read_bitmap_reg(s->enabled, ...)
    GICD_ICENABLER (479-482): 同上
    GICD_ISPENDR (483-488):   gicd_read_bitmap_reg(s->pending, ...)
    GICD_IPRIORITYR (496-507): gicd_read_ipriorityr()
    GICD_IROUTER (585-597):    gicd_read_irouter()
}
```

### MMIO 写（gicd_writel）

```c
// arm_gicv3_dist.c:614-825
static bool gicd_writel(GICv3State *s, hwaddr offset, ...)
{
    GICD_CTLR (622-659):
        DS 位: 禁用安全→合并组
        EN_GRP0/GRP1NS/GRP1S: 控制各组中断是否转发
        → gicv3_full_update() 刷新

    GICD_IGROUPR (664-678):
        NS 访问: value |= ~(old_group & ~old_grpmod)
        → 设置每 IRQ 的安全组

    GICD_ISENABLER (680-690):
        gicd_write_set_bitmap_reg(s->enabled, ...)
        → gicv3_update() 通知

    GICD_IROUTER (800-813):
        设置 gicd_irouter[irq]
        → gicv3_cache_target_cpustate() 更新路由缓存
        → gicv3_update() 通知
}
```

---

## 5. Redistributor (GICR)

```c
// arm_gicv3_redist.c:322-482 — gicr_readl
GICR_CTLR (351):      cs->gicr_ctlr
GICR_TYPER (355-365):  cs->gicr_typer (亲和性、Last、LPI 能力)
GICR_WAKER (366-371):  cs->gicr_waker (ProcessorSleep 位)
GICR_ISENABLER0 (391): gicr_read_bitmap_reg(cs->gicr_ienabler0)
GICR_IPENDR0 (395):    gicr_read_bitmap_reg(电平/边沿合并)
GICR_IPRIORITYR (409):  gicr_read_ipriorityr()
GICR_ICFGR0/1 (425):   边沿/电平配置

// arm_gicv3_redist.c:484-647 — gicr_writel
GICR_CTLR (488-505):     EnableLPIs → 使能 LPI
GICR_WAKER (509-526):    ProcessorSleep / ChildrenAsleep
GICR_ISENABLER0 (546):   gicr_write_set_bitmap_reg → gicv3_redist_update
GICR_ICENABLER0 (549):   gicr_write_clear_bitmap_reg
GICR_ISPENDR0 (552):     设置挂起位
GICR_IPRIORITYR (564):   设置 SGI/PPI 优先级
```

---

## 6. 中断状态位图与组管理

### 位图操作宏

```c
// arm_gicv3_common.h:283-309
GICV3_BITMAP_ACCESSORS(group)     // gicv3_gicd_group_set/test/clear
GICV3_BITMAP_ACCESSORS(grpmod)    // gicv3_gicd_grpmod_set/test/clear
GICV3_BITMAP_ACCESSORS(enabled)   // ...
GICV3_BITMAP_ACCESSORS(pending)
GICV3_BITMAP_ACCESSORS(active)
GICV3_BITMAP_ACCESSORS(level)
GICV3_BITMAP_ACCESSORS(edge_trigger)
GICV3_BITMAP_ACCESSORS(nmi)
```

### SPI 挂起资格计算

```c
// arm_gicv3.c:55-99 — gicd_int_pending
//
// 一个 SPI 被认为 "挂起" 需要同时满足:
//   1. PENDING 锁存 OR (电平触发 AND 输入=1)
//   2. ENABLE 位为 1
//   3. 不处于 ACTIVE 状态
//   4. 所属组的 GICD_CTLR 使能位为 1

pend = pending | (~edge_trigger & level);  // 78: 合并边沿/电平
pend &= enable;                            // 79: 使能过滤
pend &= ~active;                           // 80: 排除活跃

// 组使能过滤 (82-96):
if (EN_GRP1NS) grpmask |= group;              // Group1NS
if (EN_GRP1S)  grpmask |= (~group & grpmod);  // Group1S
if (EN_GRP0)   grpmask |= (~group & ~grpmod); // Group0
pend &= grpmask;
```

### 三组分配规则

| group 位 | grpmod 位 | 组 | 含义 |
|---------|---------|------|------|
| 0 | 0 | Group 0 | 安全 FIQ |
| 0 | 1 | Group 1 Secure | 安全 IRQ |
| 1 | X | Group 1 Non-Secure | 非安全 IRQ |

---

## 7. 亲和性路由 (ARE)

```c
// arm_gicv3_dist.c:243-286 — gicd_read/write_irouter
//   GICD_IROUTER[n]: Aff3[39:32] . Aff2[23:16] . Aff1[15:8] . Aff0[7:0]
//   中断模式: bit[31] = 1 → 1-of-N (任意 CPU)

// gicv3_internal.h:827-847 — gicv3_cache_target_cpustate
static inline void gicv3_cache_target_cpustate(GICv3State *s, int irq)
{
    // IROUTER 亲和性值 → 匹配 gicr_typer 的 CPU
    // 缓存到 gicd_irouter_target[irq] 供快速查找
}

// arm_gicv3.c:271-296 — SPI 路由使用缓存
for (i = start; i < start + len; i++) {
    cs = s->gicd_irouter_target[i];   // 283: O(1) 路由查找
    if (!cs) continue;                 // 未路由到有效 CPU
    nmi = gicv3_get_priority(cs, false, i, &prio);
    if (irqbetter(cs, i, prio, nmi)) {
        cs->hppi = {irq=i, prio, nmi}; // 更新 per-CPU hppi
    }
}
```

---

## 8. CPU Interface ICC 系统寄存器

### ICC_PMR_EL1（优先级掩码）

```c
// arm_gicv3_cpuif.c:1104-1156
// 读: 返回 icc_pmr_el1（NS 访问时高位截断）
// 写: 设置 PMR → gicv3_cpuif_update()
// 效果: 优先级 >= PMR 的中断被屏蔽
```

### ICC_IAR0/1_EL1（中断确认）

```c
// arm_gicv3_cpuif.c:1262-1310
static uint64_t icc_iar0_read(env, ri)      // Group 0 确认
{
    if (icv_access(env, HCR_FMO))            // 1267: 虚拟化重定向
        return icv_iar_read();
    if (!icc_hppi_can_preempt(cs))           // 1271: 不能抢占
        intid = INTID_SPURIOUS;              // → 返回 1023
    else
        intid = icc_hppir0_value();          // 1274: 最高 G0 中断
    if (!gicv3_intid_is_special(intid))
        icc_activate_irq(cs, intid);         // 1278: 标记活跃
    return intid;
}

static uint64_t icc_iar1_read(env, ri)      // Group 1 确认
{
    // 类似，但选择 Group 1(S/NS) 中断
    // NMI: 返回 INTID_NMI (1026) 而非实际 INTID   // 1302-1303
    intid = icc_hppir1_value();              // 1298
    icc_activate_irq(cs, intid);             // 1305
}
```

### ICC_EOIR0/1_EL1（中断结束）

```c
// arm_gicv3_cpuif.c:1645-1713
static void icc_eoir_write(env, ri, value)
{
    irq = value & 0xffffff;                  // 1650: 中断号
    is_eoir0 = (ri->crm == 8);              // 1652: 区分 EOIR0/EOIR1

    grp = icc_highest_active_group(cs);      // 1675: 当前最高活跃组
    // 验证 EOIR 写入者的安全状态匹配       // 1676-1706

    icc_drop_prio(cs, grp);                  // 1708: 降低运行优先级
    if (!icc_eoi_split(env, cs))
        icc_deactivate_irq(cs, irq);         // 1712: 非分离模式→立即去活
}
```

### 其他 ICC 寄存器

| 寄存器 | 行号 | 功能 |
|--------|------|------|
| ICC_HPPIR0/1 | 1716-1741 | 最高优先级挂起中断号（只读） |
| ICC_BPR0/1 | 1744-1825 | 二进制点（优先级分组） |
| ICC_AP0R/1R | 1827-1914 | 活跃优先级位图 |
| ICC_RPR | 1999-2039 | 当前运行优先级 |
| ICC_CTLR_EL1 | 2197-2241 | 控制（CBPR, EOImode, PRIbits） |
| ICC_CTLR_EL3 | 2244-2298 | EL3 控制（跨安全域） |
| ICC_SRE | 2598-2649 | 系统寄存器使能（SRE=1 启用 ICC） |

---

## 9. 优先级与抢占机制

### 优先级比较

```c
// arm_gicv3.c:24-53 — irqbetter
static bool irqbetter(cs, irq, prio, nmi)
{
    if (prio != cs->hppi.prio)
        return prio < cs->hppi.prio;   // 34: 低值 = 高优先级
    if (nmi != cs->hppi.nmi)
        return nmi;                     // 42: NMI 优先
    return irq <= cs->hppi.irq;         // 49: 同优先级取低号
}
```

### 组优先级掩码

```c
// arm_gicv3_cpuif.c:947-981 — icc_gprio_mask
// BPR (Binary Point Register) 决定组优先级位数:
//   BPR=0: 组优先级 = [7:1], 子优先级 = [0]
//   BPR=3: 组优先级 = [7:4], 子优先级 = [3:0]
//   BPR=7: 组优先级 = [7],   子优先级 = [6:0]
return ~0U << (bpr + 1);  // 981: 掩码
```

### 抢占判定

```c
// arm_gicv3_cpuif.c:993-1043 — icc_hppi_can_preempt
static bool icc_hppi_can_preempt(cs)
{
    // 1. 无使能的挂起中断 → false                     // 1003
    // 2. NMI: PMR < 0x80(NS) 或 =0x80(S) → false     // 1007-1016
    // 3. 非 NMI: prio >= PMR → false                  // 1017-1019
    // 4. 无活跃中断(rprio=0xff) → true                // 1022-1025
    // 5. (挂起 prio & mask) < (运行 prio & mask)      // 1033
    //    组优先级严格更高 → 抢占
    // 6. NMI 同组优先级但无 NMI 活跃 → 抢占           // 1037-1041
}
```

### 更新流程

```c
// arm_gicv3.c:178-241 — gicv3_redist_update_noirqset
//   扫描 32 个 SGI/PPI → 找最高优先级 → 更新 cs->hppi
//   检查 LPI (hpplpi) 是否更高优先级
//   若无更好结果且之前最优在本范围 → full_update

// arm_gicv3.c:257-323 — gicv3_update_noirqset
//   扫描 SPI 范围 [start, start+len)
//   通过 gicd_irouter_target[i] 路由到对应 CPU
//   更新每 CPU 的 hppi

// arm_gicv3.c:325-332 — gicv3_update
//   gicv3_update_noirqset() + 每 CPU gicv3_cpuif_update()
```

---

## 10. 中断信号传递

```c
// arm_gicv3_cpuif.c:1046-1101
void gicv3_cpuif_update(GICv3CPUState *cs)
{
    int irqlevel = 0, fiqlevel = 0, nmilevel = 0;

    // G1S → G0 降级（无 EL3 的 CPU）                 // 1060-1065
    if (cs->hppi.grp == GICV3_G1 && !ARM_FEATURE_EL3)
        cs->hppi.grp = GICV3_G0;

    if (icc_hppi_can_preempt(cs)) {                    // 1067
        switch (cs->hppi.grp) {
        case GICV3_G0:   isfiq = true;                 // 1074-1075
        case GICV3_G1:   isfiq = !secure || (EL3&&AA64);// 1077-1079
        case GICV3_G1NS: isfiq = arm_is_secure(env);   // 1081-1082
        }

        if (isfiq)      fiqlevel = 1;                  // 1088-1089
        else if (nmi)    nmilevel = 1;                  // 1090-1091
        else             irqlevel = 1;                  // 1093
    }

    qemu_set_irq(cs->parent_fiq, fiqlevel);            // 1099
    qemu_set_irq(cs->parent_irq, irqlevel);            // 1100
    qemu_set_irq(cs->parent_nmi, nmilevel);            // 1101
}
```

### IRQ/FIQ 路由规则

| 中断组 | CPU 安全状态 | 信号 | 说明 |
|--------|------------|------|------|
| Group 0 | 任意 | **FIQ** | 安全固件中断，始终 FIQ |
| Group 1S | Secure | **IRQ** | 安全 OS 中断 |
| Group 1S | Non-Secure | **FIQ** | 需陷入 EL3 路由 |
| Group 1NS | Non-Secure | **IRQ** | 正常中断 |
| Group 1NS | Secure | **FIQ** | 需陷入更高 EL |

---

## 11. 安全模型与组路由

```c
// arm_gicv3_cpuif.c:40-48 — gicv3_use_ns_bank
// ICC 寄存器按安全状态银行:
//   icc_ctlr_el1[2]  — [GICV3_S] / [GICV3_NS]
//   icc_bpr[3]       — [G0] / [G1] / [G1NS]
//   icc_igrpen[3]    — [G0] / [G1] / [G1NS]

// GICD_CTLR.DS = 1: 禁用安全扩展
//   → 所有中断视为单组
//   → NS 看到完整 GICD_CTLR
//   → grpmod 位无效

// GICD_CTLR.DS = 0: 安全扩展启用
//   EN_GRP0:   控制 Group 0 转发
//   EN_GRP1S:  控制 Group 1S 转发
//   EN_GRP1NS: 控制 Group 1NS 转发
//   NS 视图仅可见 EN_GRP1NS (作为 bit[1])
```

---

## 12. 虚拟接口 (GICv3 Virtualization)

### ICH 寄存器

```c
// gicv3_internal.h:202-262
ICH_VMCR_EL2:                              // 202-220
    VENG0/VENG1: 虚拟组 0/1 使能
    VBPR0/VBPR1: 虚拟二进制点
    VPMR: 虚拟优先级掩码

ICH_HCR_EL2:                              // 222-237
    EN: 虚拟 CPU 接口使能
    UIE/NPIE/LRENPIE: 维护中断条件
    TALL0/TALL1: 陷入 ICC_* 访问
    EOICOUNT: EOI 计数

ICH_LR_EL2[n]:                            // 239-262
    [31:0]  vINTID: 虚拟中断号
    [41:32] pINTID: 物理中断号（HW=1 时）
    [41]    EOI: EOI 维护中断标记
    [55:48] Priority: 虚拟优先级
    [59]    NMI: 不可屏蔽标记
    [60]    Group: 虚拟组 (0/1)
    [61]    HW: 硬件映射标记
    [63:62] State: Invalid/Pending/Active/Active+Pending
```

### 虚拟中断选择

```c
// arm_gicv3_cpuif.c:179-256 — hppvi_index
// 扫描所有列表寄存器，找 Pending 状态 + 组使能 + 最低优先级值
// 同时比较 vLPI (hppvlpi)
// 返回: LR 索引 / HPPVI_INDEX_VLPI / -1

// arm_gicv3_cpuif.c:471-523 — gicv3_cpuif_virt_irq_fiq_update
// 虚拟信号: G0 → vFIQ, G1 → vIRQ (或 NMI → vNMI)
qemu_set_irq(cs->parent_vfiq, fiqlevel);   // 521
qemu_set_irq(cs->parent_virq, irqlevel);   // 522
qemu_set_irq(cs->parent_vnmi, nmilevel);   // 523

// arm_gicv3_cpuif.c:526-558 — gicv3_cpuif_virt_update
// 维护中断: maintenance_interrupt_state()
//   EOI/NP/U/LRENP/VGRP0E/VGRP1E/VGRP0D/VGRP1D
// → qemu_set_irq(gicv3_maintenance_interrupt, maintlevel)
```

---

## 13. SGI 生成

```c
// arm_gicv3_cpuif.c:2095-2115
static void icc_sgi0r_write(env, ri, value)
{
    icc_generate_sgi(env, cs, value, GICV3_G0, ns);  // 2102
}

static void icc_sgi1r_write(env, ri, value)
{
    grp = ns ? GICV3_G1NS : GICV3_G1;                // 2113
    icc_generate_sgi(env, cs, value, grp, ns);        // 2114
}

// ICC_SGI1R_EL1 格式:
//   [15:0]  TargetList: 目标 CPU 位图（Aff0 匹配）
//   [23:16] Aff1
//   [39:32] Aff2
//   [47:40] INTID: SGI 号 (0-15)
//   [55:48] Aff3
//   [40]    IRM: 1=除自己外所有 CPU
```

---

## 14. 中断完整生命周期

```
1. 外设触发 SPI
   → gicv3_set_irq(opaque, irq, level=1)              [arm_gicv3.c:373]

2. Distributor 记录
   → gicv3_dist_set_irq(s, irq+GIC_INTERNAL, level)
   → pending 位图 set / level 位图更新
   → gicv3_update(s, irq, 1)                           [arm_gicv3.c:325]

3. 路由到目标 CPU
   → gicd_int_pending() 计算挂起 & 使能 & 组使能       [arm_gicv3.c:55]
   → gicd_irouter_target[irq] → GICv3CPUState *cs      [arm_gicv3.c:283]
   → irqbetter() → 更新 cs->hppi                       [arm_gicv3.c:291]

4. CPU Interface 评估
   → gicv3_cpuif_update(cs)                             [arm_gicv3_cpuif.c:1046]
   → icc_hppi_can_preempt(): PMR/运行优先级检查
   → G0→FIQ / G1→IRQ 信号选择
   → qemu_set_irq(parent_fiq/irq, 1)                   [arm_gicv3_cpuif.c:1099]

5. CPU 响应异常
   → ARM CPU 检测 IRQ/FIQ 线拉高
   → arm_cpu_exec_interrupt → arm_cpu_do_interrupt_aarch64
   → 跳转到 VBAR_EL1 + 0x280 (lower EL, IRQ)

6. 软件读 ICC_IAR1_EL1
   → icc_iar1_read()                                    [arm_gicv3_cpuif.c:1285]
   → 返回 INTID
   → icc_activate_irq(): pending→active, 更新 APR

7. 中断处理...

8. 软件写 ICC_EOIR1_EL1
   → icc_eoir_write()                                   [arm_gicv3_cpuif.c:1645]
   → icc_drop_prio(): 恢复运行优先级
   → icc_deactivate_irq(): active 位清除
   → gicv3_update() → 检查下一个挂起中断
```

---

## 15. 关键数据流总结

```
                     gicv3_set_irq()
                          │
        ┌─────────────────┼─────────────────┐
        ▼                                    ▼
  gicv3_dist_set_irq()             gicv3_redist_set_irq()
  (SPI: IRQ 32+)                   (SGI/PPI: IRQ 0-31)
        │                                    │
        ▼                                    ▼
  s->pending[irq] set              cs->gicr_ipendr0 set
        │                                    │
        ▼                                    ▼
  gicv3_update()                   gicv3_redist_update()
  ├─ gicd_int_pending()            ├─ gicr_int_pending()
  ├─ IROUTER → target CPU          ├─ 扫描 32 个 SGI/PPI
  └─ irqbetter → hppi             └─ irqbetter → hppi
        │                                    │
        └────────────┬───────────────────────┘
                     ▼
           gicv3_cpuif_update()
           ├─ icc_hppi_can_preempt()
           │  ├─ PMR 检查
           │  ├─ 运行优先级比较
           │  └─ 组优先级掩码
           ├─ G0/G1/G1NS → FIQ/IRQ 选择
           └─ qemu_set_irq(parent_fiq/irq)
                     │
                     ▼
              CPU 异常处理
              ├─ ICC_IAR1 读 → activate
              ├─ ISR 执行
              └─ ICC_EOIR1 写 → deactivate
```

---

## 交叉参考

- [48-ARM64-安全扩展TrustZone深度分析](48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md) — SCR_EL3.IRQ/FIQ 路由、安全状态判定
- [05-ARM64-中断注入与 Machine 创建深度分析](../qemu-system/05-ARM64-中断注入与Machine创建深度分析-irq级联-GIC初始化-virt平台构建-设备树生成.md) — GIC 与 virt 平台集成
- [04-QEMU-执行循环与GICv3分析](../qemu-system/04-QEMU-执行循环与GICv3分析-MainLoop-AioContext-中断检查-GICv3状态机.md) — 执行循环中的中断检查

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
