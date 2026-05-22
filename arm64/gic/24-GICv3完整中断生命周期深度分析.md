# GICv3 完整中断生命周期深度分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中 GICv3 中断控制器从设备触发到 CPU 处理完成的完整生命周期。覆盖 SPI/PPI/SGI 三种中断类型的端到端流程、中断状态机（Inactive→Pending→Active→Active+Pending）、优先级与抢占机制、EOI 模式（优先级下降与去激活分离）、以及电平/边沿触发行为。

**关键源文件：**
- `hw/intc/arm_gicv3.c` — GIC 顶层调度（set_irq、update、full_update）
- `hw/intc/arm_gicv3_cpuif.c` — CPU 接口（IAR、EOIR、SGI 生成、优先级判定）
- `hw/intc/arm_gicv3_dist.c` — 分发器（SPI 配置、IROUTER、pending 计算）
- `hw/intc/arm_gicv3_redist.c` — 重分发器（PPI、SGI、LPI 处理）
- `hw/intc/arm_gicv3_common.c` — 公共初始化（GPIO 连线、IRQ/FIQ 输出）
- `include/hw/intc/arm_gicv3_common.h` — 数据结构定义

---

## 2. GICv3 架构与数据结构

### 2.1 分层架构

```
设备 → [GPIO/qemu_set_irq] → GICv3 顶层 (arm_gicv3.c)
                                  |
                     ┌────────────┼────────────┐
                     ↓            ↓            ↓
                   GICD        GICR[0]      GICR[n]
                (分发器)     (重分发器0)   (重分发器n)
                SPI 管理    SGI/PPI/LPI    SGI/PPI/LPI
                     |            |            |
                     └────────┬───┘            |
                              ↓                ↓
                          CPU接口[0]       CPU接口[n]
                        优先级/抢占       优先级/抢占
                              |                |
                              ↓                ↓
                         FIQ/IRQ/NMI      FIQ/IRQ/NMI
```

### 2.2 核心数据结构

```c
// arm_gicv3_common.h:253-275 — GICv3State（分发器）
typedef struct GICv3State {
    uint32_t num_irq;           // SPI 数量
    uint32_t num_cpu;           // CPU 数量
    GIC_DECLARE_BITMAP(pending);    // 每中断 pending 位
    GIC_DECLARE_BITMAP(active);     // 每中断 active 位
    GIC_DECLARE_BITMAP(enabled);    // 每中断 enable 位
    GIC_DECLARE_BITMAP(group);      // IGROUPR：0=G0, 1=G1NS
    GIC_DECLARE_BITMAP(grpmod);     // IGRPMODR：G1S 标记
    GIC_DECLARE_BITMAP(level);      // 电平输入
    GIC_DECLARE_BITMAP(edge_trigger); // 0=电平, 1=边沿
    uint8_t gicd_ipriority[...];    // 优先级
    uint64_t gicd_irouter[...];     // 亲和性路由
    GICv3CPUState *gicd_irouter_target[...]; // 路由缓存
    // ...
} GICv3State;

// arm_gicv3_common.h:137-155 — GICv3CPUState（重分发器+CPU接口）
typedef struct GICv3CPUState {
    uint32_t gicr_ipendr0;      // SGI/PPI pending
    uint32_t gicr_iactiver0;    // SGI/PPI active
    uint32_t level;             // PPI 电平输入
    uint32_t edge_trigger;      // 边沿/电平配置
    PendingIrq hppi;            // 当前最高优先级 pending 中断
    PendingIrq hpplpi;          // 当前最高优先级 LPI
    uint8_t icc_pmr_el1;        // 优先级屏蔽寄存器
    uint32_t icc_apr[3][4];     // 活动优先级寄存器 [G0/G1/G1NS][0-3]
    uint64_t gicr_propbaser;    // LPI 配置表基址
    uint64_t gicr_pendbaser;    // LPI pending 表基址
    // ...
} GICv3CPUState;
```

---

## 3. GPIO 连线与中断输入

### 3.1 设备到 GIC 的连接

```c
// arm_gicv3_common.c:314-370 — gicv3_init_irqs_and_mmio()
// SPI 输入：num_irq - GIC_INTERNAL 个 GPIO
// PPI 输入：每 CPU GIC_INTERNAL (32) 个 GPIO
qdev_init_gpio_in_named(DEVICE(s), gicv3_set_irq, NULL,
                        s->num_irq - GIC_INTERNAL +     // SPI
                        s->num_cpu * GIC_INTERNAL);      // PPI

// CPU 输出线：每 CPU 4 个输出（IRQ、FIQ、VIRQ、VFIQ + NMI 等）
for (i = 0; i < s->num_cpu; i++) {
    sysbus_init_irq(SYS_BUS_DEVICE(s), &s->cpu[i].parent_irq);
    sysbus_init_irq(SYS_BUS_DEVICE(s), &s->cpu[i].parent_fiq);
    sysbus_init_irq(SYS_BUS_DEVICE(s), &s->cpu[i].parent_virq);
    sysbus_init_irq(SYS_BUS_DEVICE(s), &s->cpu[i].parent_vfiq);
    sysbus_init_irq(SYS_BUS_DEVICE(s), &s->cpu[i].parent_nmi);
}
```

### 3.2 中断分派入口

```c
// arm_gicv3.c:372-400 — gicv3_set_irq()
static void gicv3_set_irq(void *opaque, int irq, int level)
{
    GICv3State *s = opaque;
    if (irq < (s->num_irq - GIC_INTERNAL)) {
        // SPI → 分发器
        gicv3_dist_set_irq(s, irq + GIC_INTERNAL, level);
    } else {
        // PPI → 对应 CPU 的重分发器
        int cpu = irq / GIC_INTERNAL;
        irq %= GIC_INTERNAL;
        assert(irq >= GIC_NR_SGIS);  // SGI 不应通过此路径
        gicv3_redist_set_irq(&s->cpu[cpu], irq, level);
    }
}
```

---

## 4. SPI 生命周期（端到端）

### 4.1 阶段 1：设备触发 → GICD 接收

```
设备调用 qemu_set_irq(spi_line, 1)
  → gicv3_set_irq()
    → gicv3_dist_set_irq(s, irq + GIC_INTERNAL, level)
```

分发器更新电平和 pending 位：
- 边沿触发：0→1 跳变时设置 pending 锁存
- 电平触发：level 位直接跟踪输入

### 4.2 阶段 2：GICD pending 计算

```c
// arm_gicv3.c:55-99 — gicd_int_pending()
static uint32_t gicd_int_pending(GICv3State *s, int irq)
{
    // 有效 pending = (pending 锁存 | (非边沿 & 电平输入))
    pend = pending | (~edge_trigger & level);
    pend &= enable;       // 必须使能
    pend &= ~active;      // 已 active 的不算（避免重复，A+P 另行处理）

    // 按组使能过滤
    grpmask = 0;
    if (GICD_CTLR & EN_GRP1NS) grpmask |= group;      // G1NS 使能
    if (GICD_CTLR & EN_GRP1S)  grpmask |= (~group & grpmod);  // G1S
    if (GICD_CTLR & EN_GRP0)   grpmask |= (~group & ~grpmod); // G0
    pend &= grpmask;
    return pend;
}
```

### 4.3 阶段 3：路由到目标 CPU

```c
// arm_gicv3.c:257-323 — gicv3_update_noirqset()
// 遍历 pending SPI，通过 gicd_irouter_target[] 缓存找到目标 CPU
for (i = start; i < start + len; i++) {
    cs = s->gicd_irouter_target[i];   // GICD_IROUTER 路由缓存
    if (!cs) continue;                // 无目标 CPU → 保持 pending
    nmi = gicv3_get_priority(cs, false, i, &prio);
    if (irqbetter(cs, i, prio, nmi)) {
        cs->hppi.irq = i;            // 更新此 CPU 的最高优先级中断
        cs->hppi.prio = prio;
        cs->hppi.nmi = nmi;
    }
}

// arm_gicv3.c:325-333 — gicv3_update()
void gicv3_update(GICv3State *s, int start, int len)
{
    gicv3_update_noirqset(s, start, len);
    for (i = 0; i < s->num_cpu; i++) {
        gicv3_cpuif_update(&s->cpu[i]);  // 通知每个 CPU 接口
    }
}
```

### 4.4 阶段 4：CPU 接口抢占判定

```c
// arm_gicv3_cpuif.c:993-1044 — icc_hppi_can_preempt()
static bool icc_hppi_can_preempt(GICv3CPUState *cs)
{
    // 1. PMR 检查：hppi.prio >= icc_pmr_el1 → 被屏蔽
    if (cs->hppi.prio >= cs->icc_pmr_el1) return false;

    // 2. 运行优先级检查
    rprio = icc_highest_active_prio(cs);  // 扫描 APR 位
    if (rprio == 0xff) return true;        // 无活动中断 → 可抢占

    // 3. 组优先级比较（忽略子优先级）
    mask = icc_gprio_mask(cs, cs->hppi.grp);
    if ((cs->hppi.prio & mask) < (rprio & mask)) return true;

    // 4. NMI 特殊处理：同优先级但非 NMI active → NMI 可抢占
    if (cs->hppi.nmi && !(apr[grp][0] & NMI_BIT)) return true;

    return false;
}
```

### 4.5 阶段 5：信号 CPU

```c
// arm_gicv3_cpuif.c:1046-1102 — gicv3_cpuif_update()
// 见 arm64/23 文档详细分析
// G0 → FIQ, G1S 跨世界 → FIQ, G1NS 跨世界 → FIQ, 其他 → IRQ
qemu_set_irq(cs->parent_fiq, fiqlevel);
qemu_set_irq(cs->parent_irq, irqlevel);
qemu_set_irq(cs->parent_nmi, nmilevel);
```

### 4.6 阶段 6：CPU 应答（ICC_IAR 读取）

```c
// arm_gicv3_cpuif.c:1262-1283 — icc_iar0_read()（Group 0）
static uint64_t icc_iar0_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    if (icv_access(env, HCR_FMO)) return icv_iar_read(env, ri);

    if (!icc_hppi_can_preempt(cs)) return INTID_SPURIOUS;
    intid = icc_hppir0_value(cs, env);

    if (!gicv3_intid_is_special(intid)) {
        icc_activate_irq(cs, intid);  // Pending → Active
    }
    return intid;
}

// arm_gicv3_cpuif.c:1285-1311 — icc_iar1_read()（Group 1）
// 类似，但检查 Group 1 + NMI 返回 INTID_NMI
```

### 4.7 阶段 7：状态转换 Pending → Active

```c
// arm_gicv3_cpuif.c:1158-1187 — icc_activate_irq()
static void icc_activate_irq(GICv3CPUState *cs, int irq)
{
    // 1. 设置活动优先级寄存器（APR）位
    int prio = cs->hppi.prio & mask;
    int aprbit = prio >> (8 - cs->prebits);
    cs->icc_apr[grp][regno] |= (1U << regbit);

    // 2. 按中断类型更新状态位
    if (irq < GIC_INTERNAL) {
        // SGI/PPI → 重分发器
        cs->gicr_iactiver0 |= (1 << irq);   // 设置 active
        cs->gicr_ipendr0 &= ~(1 << irq);    // 清除 pending
        gicv3_redist_update(cs);
    } else if (irq < GICV3_LPI_INTID_START) {
        // SPI → 分发器
        gicv3_gicd_active_set(s, irq);
        gicv3_gicd_pending_clear(s, irq);
        gicv3_update(s, irq, 1);
    } else {
        // LPI → pending 表
        gicv3_redist_lpi_pending(cs, irq, 0);
    }
}
```

### 4.8 阶段 8：EOI（中断完成）

```c
// arm_gicv3_cpuif.c:1645-1714 — icc_eoir_write()
static void icc_eoir_write(CPUARMState *env, const ARMCPRegInfo *ri,
                           uint64_t value)
{
    int irq = value & 0xffffff;
    bool is_eoir0 = ri->crm == 8;  // EOIR0 vs EOIR1

    // 虚拟接口重定向
    if (icv_access(env, is_eoir0 ? HCR_FMO : HCR_IMO)) {
        icv_eoir_write(env, ri, value); return;
    }

    // 组匹配检查：EOIR0 只能 EOI G0，EOIR1 只能 EOI G1/G1NS
    grp = icc_highest_active_group(cs);
    // ... 安全状态检查 ...

    // 优先级下降（总是执行）
    icc_drop_prio(cs, grp);

    // 去激活（取决于 EOImode）
    if (!icc_eoi_split(env, cs)) {
        // EOImode=0：优先级下降 + 去激活一起
        icc_deactivate_irq(cs, irq);
    }
    // EOImode=1：仅优先级下降，需要另写 ICC_DIR_EL1 去激活
}
```

### 4.9 EOImode 分离模式

```
EOImode=0（默认）：
  写 ICC_EOIR → 优先级下降 + Active→Inactive
  一步完成，简单高效

EOImode=1（分离模式）：
  写 ICC_EOIR → 仅优先级下降（允许更低优先级中断抢占）
  写 ICC_DIR  → Active→Inactive（去激活）
  用途：允许在不去激活的情况下处理新中断（如虚拟化场景）
```

---

## 5. SGI 生命周期

### 5.1 SGI 生成

```c
// arm_gicv3_cpuif.c:2042-2093 — icc_generate_sgi()
static void icc_generate_sgi(CPUARMState *env, GICv3CPUState *cs,
                             uint64_t value, int grp, bool ns)
{
    // 从 ICC_SGI1R_EL1 解码
    uint64_t aff = Aff3:Aff2:Aff1;  // 目标亲和性
    uint32_t targetlist;              // 16位目标列表
    uint32_t irq = bits[27:24];       // SGI INTID (0-15)
    bool irm = bit[40];              // IRM=1 → 发给所有 CPU

    // G1S + DS=1 → 降级为 G0
    if (grp == GICV3_G1 && GICD_CTLR.DS) grp = GICV3_G0;

    for (each target CPU) {
        if (irm) { /* 发给除自己外所有 CPU */ }
        else     { /* 按亲和性+targetlist 匹配 */ }
        gicv3_redist_send_sgi(target, grp, irq, ns);
    }
}
```

### 5.2 重分发器 SGI 处理

```c
// arm_gicv3_redist.c:1156-1187 — gicv3_redist_send_sgi()
void gicv3_redist_send_sgi(GICv3CPUState *cs, int grp, int irq, bool ns)
{
    // 组匹配检查
    int irqgrp = gicv3_irq_group(cs->gic, cs, irq);
    if (grp == GICV3_G1 && irqgrp == GICV3_G0) grp = GICV3_G0;
    if (grp != irqgrp) return;

    // NS 安全检查（NSACR）
    if (ns && !GICD_CTLR.DS) {
        int nsaccess = gicr_ns_access(cs, irq);
        if ((G0 && nsaccess < 1) || (G1 && nsaccess < 2)) return;
    }

    // 设置 pending 位并触发更新
    cs->gicr_ipendr0 |= (1 << irq);
    gicv3_redist_update(cs);
}
```

### 5.3 SGI 与 SPI 的关键区别

| 特性 | SGI (0-15) | SPI (32-1019) |
|------|-----------|--------------|
| 路由路径 | CPU→GICR→GICR（直接） | 设备→GICD→GICR（经分发器） |
| 触发方式 | 软件写 ICC_SGIxR_EL1 | 硬件 GPIO |
| Per-CPU | 每 CPU 独立 pending | 全局共享 |
| 安全检查 | NSACR | IGROUPR/IGRPMODR |
| 应答/EOI | 与 SPI 相同 | 与 SGI 相同 |

---

## 6. PPI 生命周期

### 6.1 PPI 输入（Timer/PMU）

```c
// arm_gicv3_redist.c:1135-1154 — gicv3_redist_set_irq()
void gicv3_redist_set_irq(GICv3CPUState *cs, int irq, int level)
{
    if (level == extract32(cs->level, irq, 1)) return;  // 无变化

    cs->level = deposit32(cs->level, irq, 1, level);    // 更新电平

    if (level) {
        // 边沿触发：0→1 锁存 pending
        if (extract32(cs->edge_trigger, irq, 1)) {
            cs->gicr_ipendr0 |= (1 << irq);
        }
    }
    // 电平触发：pending 在 gicr_int_pending() 中实时计算
    gicv3_redist_update(cs);
}
```

### 6.2 典型 PPI 连接

| PPI 号 | INTID | 设备 | 用途 |
|--------|-------|------|------|
| PPI 0-3 | 16-19 | 保留 | — |
| PPI 11 | 27 | Virtual Timer | 虚拟定时器 |
| PPI 13 | 29 | Secure Physical Timer | 安全物理定时器 |
| PPI 14 | 30 | NS Physical Timer | 非安全物理定时器 |
| PPI 10 | 26 | Hypervisor Timer | Hypervisor 定时器 |
| PPI 7 | 23 | PMU | 性能监控溢出 |

---

## 7. 中断状态机

### 7.1 四种状态

```
                    ┌──────────┐
         触发       │ Inactive │ ──────────────────────────────┐
     ┌──────────→  │  (I)     │                              │
     │             └──────┬───┘                              │
     │                    │ 触发（pending 条件满足）            │
     │                    ↓                                  │
     │             ┌──────────┐    IAR 读取    ┌──────────┐  │
     │             │ Pending  │ ─────────────→ │  Active  │  │
     │             │  (P)     │                │  (A)     │  │
     │             └──────────┘                └────┬─────┘  │
     │                    │                         │        │
     │                    │  在 Active 期间         │ EOI    │
     │                    │  再次触发               │ 写入   │
     │                    │                         ↓        │
     │             ┌──────────────┐                          │
     │             │ Active +     │ ─── EOI 写入 ──→ Pending │
     │             │ Pending (AP) │                          │
     │             └──────────────┘                          │
     │                                                      │
     └──────────────────────────────────────────────────────┘
                           EOI（无新 pending）
```

### 7.2 状态转换实现

| 转换 | 触发条件 | 代码位置 |
|------|---------|---------|
| I → P | 设备触发 + enable + 组使能 | `gicv3_dist_set_irq` / `gicv3_redist_set_irq` |
| P → A | CPU 读 ICC_IAR | `icc_activate_irq` (cpuif.c:1158-1187) |
| A → I | CPU 写 ICC_EOIR (EOImode=0) 或 ICC_DIR | `icc_deactivate_irq` (cpuif.c:1436-1493) |
| A → AP | Active 期间再次触发 | pending + active 位同时为 1 |
| AP → P | CPU 写 ICC_EOIR → 清除 active | `icc_deactivate_irq` |
| AP → A | 清除 pending（电平下降或 CLEAR） | 电平变化 / GICD_ICPENDR 写入 |

### 7.3 电平 vs 边沿触发

```c
// arm_gicv3.c:78 — 电平触发的 pending 计算
pend = pending | (~edge_trigger & level);
// 电平触发：pending = 锁存 OR 当前电平
// 边沿触发：pending = 仅锁存位

// arm_gicv3.c:101-139 — gicv3_dist_set_irq()
// 边沿触发：0→1 时设置 pending 锁存
// 电平触发：仅更新 level 位，pending 在查询时实时计算
```

**关键区别：**
- 边沿触发：EOI 清除 pending 后不会重新 pending，除非再有边沿
- 电平触发：EOI 清除 pending 后，如果电平仍为高，立即重新 pending（自动进入 Active+Pending）

---

## 8. 优先级与抢占

### 8.1 优先级空间

```
优先级值：0x00（最高）→ 0xFF（最低/空闲）
- QEMU 实现 8 位优先级
- NS 世界看到的优先级空间：[0x80-0xFF]
- Secure 世界：[0x00-0xFF]
```

### 8.2 Binary Point 与组优先级

```c
// arm_gicv3_cpuif.c:947-982 — icc_gprio_mask()
static uint32_t icc_gprio_mask(GICv3CPUState *cs, int group)
{
    // BPR 决定优先级中"组优先级"和"子优先级"的分割点
    // 例如 BPR=3: 组优先级 = prio[7:4], 子优先级 = prio[3:0]
    // 抢占只看组优先级，不看子优先级
    int bpr = (group == GICV3_G0) ? cs->icc_bpr[0] :
              MAX(cs->icc_bpr[1], cs->icc_bpr[0] + 1);
    return ~0U << (bpr + 1);
}
```

### 8.3 活动优先级追踪（Running Priority）

```c
// arm_gicv3_cpuif.c:910-945 — icc_highest_active_prio()
static int icc_highest_active_prio(GICv3CPUState *cs)
{
    // 扫描所有 APR 位（G0 + G1 + G1NS 合并）
    // NMI 特殊处理：NMI active 时返回最高优先级 0
    if (cs->icc_apr[G1][0] & ICC_AP1R_EL1_NMI) return 0;
    if (cs->icc_apr[G1NS][0] & NMI_BIT) return DS ? 0 : 0x80;

    // 找到最低位的 APR 位 → 对应当前运行优先级
    for (i = 0; i < num_aprs; i++) {
        apr = cs->icc_apr[G0][i] | cs->icc_apr[G1][i] | cs->icc_apr[G1NS][i];
        if (apr) return (i * 32 + ctz32(apr)) << (min_bpr + 1);
    }
    return 0xff;  // 无活动中断 → 空闲优先级
}
```

### 8.4 抢占流程

```
新中断到达:
  1. PMR 检查：prio >= PMR → 被屏蔽
  2. 运行优先级：计算当前最高活动优先级 (rprio)
  3. 组优先级比较：(prio & mask) < (rprio & mask) → 可抢占
  4. NMI 特例：同组优先级但 NMI 可抢占非 NMI

抢占发生:
  → CPU 保存当前中断上下文
  → 处理新的更高优先级中断
  → EOI 新中断 → 恢复被抢占中断
```

---

## 9. 完整端到端 SPI 流程

```
1. 设备触发                                    [设备代码]
   qemu_set_irq(spi_gpio, 1)

2. GIC 入口分派                                [arm_gicv3.c:372-400]
   gicv3_set_irq()
     → gicv3_dist_set_irq(s, irq+32, 1)

3. 分发器 pending 更新                          [arm_gicv3_dist.c]
   更新 level/pending 位
   → gicv3_update(s, irq, 1)

4. 路由到目标 CPU                              [arm_gicv3.c:257-323]
   gicv3_update_noirqset()
     遍历 pending SPI
     gicd_irouter_target[irq] → 目标 CPU
     比较优先级 → 更新 cs->hppi

5. CPU 接口判定                                [arm_gicv3_cpuif.c:993-1102]
   gicv3_cpuif_update()
     icc_hppi_can_preempt()
       PMR 检查 → 运行优先级 → 组优先级比较
     确定信号类型（FIQ/IRQ/NMI）
     qemu_set_irq(parent_fiq/irq/nmi, 1)

6. CPU 中断响应                                [cpu-irq.c:171-265]
   arm_cpu_exec_interrupt()
     arm_phys_excp_target_el() → target_el
     arm_excp_unmasked() → 是否可响应
     → 异常入口

7. ISR 应答                                    [arm_gicv3_cpuif.c:1262-1311]
   CPU 读 ICC_IAR1_EL1
     icc_iar1_read()
       icc_hppi_can_preempt() → 检查
       icc_activate_irq() → Pending→Active
       返回 INTID

8. 中断处理                                    [Guest ISR]
   设备驱动处理中断

9. EOI                                         [arm_gicv3_cpuif.c:1645-1714]
   CPU 写 ICC_EOIR1_EL1
     icc_eoir_write()
       icc_drop_prio() → 降低运行优先级
       icc_deactivate_irq() → Active→Inactive（EOImode=0）

10. GIC 重新评估                               [arm_gicv3.c:325-333]
    gicv3_update() → 重新计算所有 CPU 的 hppi
    gicv3_cpuif_update() → 如有新中断 → 重复步骤 5
```

---

## 10. 1-of-N SPI 与其他特性

### 10.1 1-of-N 模型

```c
// arm_gicv3_dist.c:403-418 — GICD_TYPER
// QEMU 设置 No1N=1，表示不支持 1-of-N SPI
// 即 SPI 路由严格按照 GICD_IROUTER 指定的 CPU 投递
// 不会自动选择"空闲"CPU
```

### 10.2 GICD_IROUTER 路由缓存

```c
// arm_gicv3_dist.c:264-286 — gicd_write_irouter()
// Guest 写 GICD_IROUTER[irq] 时：
// 1. 解码亲和性 Aff3:Aff2:Aff1:Aff0
// 2. 查找匹配的 GICv3CPUState
// 3. 缓存到 s->gicd_irouter_target[irq]
// 后续 gicv3_update_noirqset() 直接使用缓存
```

---

## 11. 小结

| 阶段 | 关键函数 | 源文件:行号 |
|------|---------|-----------|
| **设备触发** | `gicv3_set_irq` | arm_gicv3.c:372-400 |
| **SPI 分派** | `gicv3_dist_set_irq` | arm_gicv3_dist.c |
| **PPI 分派** | `gicv3_redist_set_irq` | arm_gicv3_redist.c:1135-1154 |
| **SGI 生成** | `icc_generate_sgi` | arm_gicv3_cpuif.c:2042-2093 |
| **SGI 接收** | `gicv3_redist_send_sgi` | arm_gicv3_redist.c:1156-1187 |
| **Pending 计算** | `gicd_int_pending` | arm_gicv3.c:55-99 |
| **路由+优先级** | `gicv3_update_noirqset` | arm_gicv3.c:257-323 |
| **全局更新** | `gicv3_update` | arm_gicv3.c:325-333 |
| **抢占判定** | `icc_hppi_can_preempt` | arm_gicv3_cpuif.c:993-1044 |
| **运行优先级** | `icc_highest_active_prio` | arm_gicv3_cpuif.c:910-945 |
| **组优先级掩码** | `icc_gprio_mask` | arm_gicv3_cpuif.c:947-982 |
| **信号 CPU** | `gicv3_cpuif_update` | arm_gicv3_cpuif.c:1046-1102 |
| **应答 G0** | `icc_iar0_read` | arm_gicv3_cpuif.c:1262-1283 |
| **应答 G1** | `icc_iar1_read` | arm_gicv3_cpuif.c:1285-1311 |
| **状态转换** | `icc_activate_irq` | arm_gicv3_cpuif.c:1158-1187 |
| **EOI** | `icc_eoir_write` | arm_gicv3_cpuif.c:1645-1714 |
| **去激活** | `icc_deactivate_irq` | arm_gicv3_cpuif.c:1436-1493 |

**核心设计原则：**
1. **分层更新**：设备→GICD/GICR→CPU接口→CPU 线，每层独立计算
2. **增量更新**：`gicv3_update(start, len)` 只重算受影响范围，优化性能
3. **路由缓存**：`gicd_irouter_target[]` 避免每次重新解码亲和性
4. **APR 位图**：活动优先级用位图追踪，`ctz32` 快速查找最高优先级
5. **统一应答/EOI**：SGI/PPI/SPI 共享同一 `icc_activate_irq`/`icc_eoir_write` 路径
