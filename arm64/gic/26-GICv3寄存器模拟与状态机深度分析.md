# GICv3 寄存器模拟与状态机深度分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中 GICv3 所有关键寄存器的模拟实现，包括分发器（GICD）、重分发器（GICR）、CPU 接口（ICC 系统寄存器）和 Hypervisor 接口（ICH）四个层面的寄存器读写处理、安全访问控制、副作用触发、以及迁移状态保存。

**关键源文件：**
- `hw/intc/arm_gicv3_dist.c` — GICD 寄存器 MMIO 读写（~820行）
- `hw/intc/arm_gicv3_redist.c` — GICR 寄存器 MMIO 读写（~1200行）
- `hw/intc/arm_gicv3_cpuif.c` — ICC/ICV/ICH 系统寄存器（~2700行）
- `hw/intc/arm_gicv3_common.c` — VMSTATE 迁移定义（~400行）
- `include/hw/intc/arm_gicv3_common.h` — 状态结构体定义
- `hw/intc/gicv3_internal.h` — 内部宏和常量

---

## 2. GICD 分发器寄存器

### 2.1 GICD 寄存器分派

```c
// arm_gicv3_dist.c:373-820
// 四个主要 MMIO 处理函数：
//   gicd_readl()  / gicd_writel()  — 32位寄存器
//   gicd_readq()  / gicd_writeq()  — 64位寄存器（IROUTER）
// 通过 offset 范围分派到具体寄存器
```

### 2.2 GICD_CTLR — 控制寄存器

```c
// 读: arm_gicv3_dist.c:382-402
// NS 视图仅可见: ARE_S(别名) | EN_GRP1NS | RWP
// Secure 视图: 完整寄存器

// 写: arm_gicv3_dist.c:622-660
// DS=0 + Secure: 可写 DS | EN_GRP0 | EN_GRP1S | EN_GRP1NS
// DS=0 + NS:     仅可写 EN_GRP1NS（别名 bit[1]）
// DS=1:          可写 EN_GRP0 | EN_GRP1NS
// ARE_S/ARE_NS:  RAO/WI（QEMU 始终启用亲和性路由）
// 写 DS=1:       单向转换，清除 EN_GRP1S | ARE_NS
// 副作用:        gicv3_full_update(s)
```

**GICD_CTLR 位域：**

| 位 | 名称 | DS=0 Secure 写 | DS=0 NS 写 | DS=1 写 |
|----|------|---------------|-----------|--------|
| 0 | EN_GRP0 | 可写 | RES0 | 可写 |
| 1 | EN_GRP1NS | 可写 | 可写 | 可写 |
| 2 | EN_GRP1S | 可写 | RES0 | N/A |
| 4 | ARE_NS | RAO/WI | RAO/WI | N/A |
| 5 | ARE_S | RAO/WI | N/A | RAO/WI |
| 6 | DS | 可写(单向→1) | RES0 | RAO/WI |
| 31 | RWP | RO | RO | RO |

### 2.3 GICD_TYPER — 类型寄存器（只读）

```c
// arm_gicv3_dist.c:403-432
// 动态计算：
*data = (1 << 25)              // A3V: 支持 Aff3 非零
      | (1 << 24)              // No1N: 不支持 1-of-N
      | (dvis << 18)           // DVIS: GICv4 支持直接 vLPI
      | (sec_extn << 10)       // SecurityExtn: DS=0 时为 1
      | (nmi << NMI_SHIFT)     // NMI: FEAT_NMI 支持
      | (lpi << LPIS_SHIFT)    // LPIS: LPI 支持
      | (0xf << 19)            // IDbits=15: 16位 INTID
      | itlinesnumber;          // ITLinesNumber = (num_irq/32)-1
// CPUNumber=0: ARE 始终为 1 时无意义
```

### 2.4 GICD_IGROUPR / GICD_IGRPMODR — 组配置

```c
// arm_gicv3_dist.c:455-468,664-679,748-765
// IGROUPR: bit=0 → 非 G1NS, bit=1 → G1NS
// IGRPMODR: bit=1 → G1S（仅当 IGROUPR=0 时）
// 组判定: G0 = !group & !grpmod
//         G1S = !group & grpmod
//         G1NS = group

// NS 访问限制（DS=0 时）:
// IGROUPR: Secure 中断位 RAZ/WI
// IGRPMODR: NS 完全 RAZ/WI
// 写后副作用: gicv3_update(s, irq, 32)
```

### 2.5 GICD_ISENABLER / GICD_ICENABLER — 使能

```c
// arm_gicv3_dist.c:115-160,680-687
// ISENABLER 写: 位为 1 → 设置 enabled 位（set-only）
// ICENABLER 写: 位为 1 → 清除 enabled 位（clear-only）
// NS: Secure 中断位 RAZ/WI
// 副作用: gicv3_update(s, irq, 32)
```

### 2.6 GICD_ISPENDR / GICD_ICPENDR — Pending

```c
// arm_gicv3_dist.c:168-193,688-695
// 读: 返回 (pending_latch | (~edge_trigger & level))
// ISPENDR 写: 设置 pending 锁存（不影响电平输入）
// ICPENDR 写: 清除 pending 锁存
// NS: Secure 中断位 RAZ/WI
// 副作用: gicv3_update(s, irq, 32)
```

### 2.7 GICD_ISACTIVER / GICD_ICACTIVER — Active

```c
// arm_gicv3_dist.c:488-495,696-703
// ISACTIVER 写: 设置 active 位
// ICACTIVER 写: 清除 active 位
// NS: Secure 中断位 RAZ/WI
// 副作用: gicv3_update(s, irq, 32)
```

### 2.8 GICD_IPRIORITYR — 优先级

```c
// arm_gicv3_dist.c:196-241
// 每中断 8 位优先级

// 读（NS, DS=0）:
//   Secure 中断: 返回 0（RAZ）
//   NS 中断: 返回 (prio << 1) & 0xff  // 左移1位，高半空间映射到全空间

// 写（NS, DS=0）:
//   Secure 中断: 忽略（WI）
//   NS 中断: 存储 0x80 | (value >> 1)  // 压缩到 [0x80-0xFF]

// Secure 或 DS=1: 直接读写，无变换
```

**优先级空间映射：**

```
Secure 视图:  [0x00 ────────── 0x7F] [0x80 ────────── 0xFF]
              ← Secure 专属 →        ← NS 可见 →

NS 视图:      [0x00 ──────────────────────────────── 0xFF]
              ← 映射到 Secure 视图的 [0x80-0xFF] →
```

### 2.9 GICD_ICFGR — 触发配置

```c
// arm_gicv3_dist.c:512-547,721-747
// 每中断 2 位: bit[1] = 0 电平触发, 1 边沿触发; bit[0] 保留
// SGI (0-15): 固定边沿，不可修改
// 写后副作用: gicv3_update(s, irq, 32)
```

### 2.10 GICD_IROUTER — 亲和性路由

```c
// arm_gicv3_dist.c:243-286,585-597,800-813
// 64 位: Aff3[39:32]:Aff2[23:16]:Aff1[15:8]:Aff0[7:0] + IRM[31]
// IRM=1: 路由到任意 CPU（QEMU 选第一个匹配）
// 写后: gicv3_cache_target_cpustate(s, irq) 更新缓存
// 缓存: s->gicd_irouter_target[irq] = 匹配的 GICv3CPUState*
```

---

## 3. GICR 重分发器寄存器

### 3.1 GICR 寄存器分派

```c
// arm_gicv3_redist.c:322-620
// 两个 MMIO frame:
//   Frame 0 (RD_base): GICR_CTLR, TYPER, WAKER, PROPBASER, PENDBASER
//   Frame 1 (SGI_base): IGROUPR0, ISENABLER0, ISPENDR0, IPRIORITYR 等
// gicr_readl/writel + gicr_readll/writell（64位）
```

### 3.2 GICR_CTLR — 控制

```c
// arm_gicv3_redist.c:488-505
// ENABLE_LPIS 位: 写 1 使能 LPI 处理
//   副作用: gicv3_redist_update_lpi() + gicv3_redist_update()
// RWP (bit 3): 只读，表示待处理操作
// UWP (bit 4): 只读
```

### 3.3 GICR_TYPER — 类型（只读，64位）

```c
// arm_gicv3_redist.c:347-372
// 包含: Affinity (64位), PPInum, VLPIS, Last, DPGS 等
// Aff3:Aff2:Aff1:Aff0 从 CPU 的 MPIDR 派生
// Last bit: 标识此 CPU 是否是最后一个重分发器
```

### 3.4 GICR_WAKER — 电源状态

```c
// arm_gicv3_redist.c:509-526
// ProcessorSleep (bit 1): 写 1 = 进入睡眠
// ChildrenAsleep (bit 2): 只读，QEMU 中镜像 ProcessorSleep
// QEMU 简化: 不真正实现电源管理，仅追踪状态
```

### 3.5 GICR SGI/PPI 寄存器

```c
// 与 GICD 对应，但仅 32 位（覆盖 INTID 0-31）:
// GICR_IGROUPR0:     组配置
// GICR_IGRPMODR0:    组修饰符
// GICR_ISENABLER0:   使能
// GICR_ISPENDR0:     Pending（读=pending|level, 写=设置锁存）
// GICR_IACTIVER0:    Active
// GICR_IPRIORITYR:   优先级 (32 字节, 每中断 1 字节)
// GICR_ICFGR0/1:     触发配置

// NS 访问限制: 同 GICD（Secure 中断 RAZ/WI）
// 写后副作用: gicv3_redist_update(cs)
```

### 3.6 GICR_PROPBASER / GICR_PENDBASER — LPI 表

```c
// arm_gicv3_redist.c:527-544,633-644,657-688
// PROPBASER: LPI 配置表物理基址 + IDbits + Shareability + Cacheability
// PENDBASER: LPI pending 表物理基址
// 仅在 ENABLE_LPIS=0 时可写
// 这两个寄存器指向 Guest 内存中的 LPI 配置
```

---

## 4. ICC CPU 接口系统寄存器

### 4.1 寄存器定义表

```c
// arm_gicv3_cpuif.c:2463+ — gicv3_cpuif_reginfo[]
// ARMCPRegInfo 数组定义所有 ICC_/ICV_/ICH_ 系统寄存器
// 包含: opc0/opc1/crn/crm/opc2 编码、访问权限、读写函数
```

### 4.2 ICC_PMR_EL1 — 优先级屏蔽

```c
// arm_gicv3_cpuif.c:1131-1156
// 读: NS 访问时 PMR < 0x80 返回 0; PMR >= 0x80 返回 (pmr<<1)&0xff
// 写: NS 访问时 存储 0x80|(value>>1); PMR < 0x80 时 NS 不能修改
// ICV 重定向: icv_access(HCR_FMO|HCR_IMO) → icv_pmr_read/write
// 副作用: gicv3_cpuif_update(cs)
```

### 4.3 ICC_IAR0_EL1 / ICC_IAR1_EL1 — 中断应答

```c
// arm_gicv3_cpuif.c:1262-1311
// ICC_IAR0: Group 0 应答（accessfn = gicv3_fiq_access）
// ICC_IAR1: Group 1 应答（accessfn = gicv3_irq_access）
// 只读寄存器
// 流程: icc_hppi_can_preempt() → icc_hppir_value() → icc_activate_irq()
// 返回值: INTID (0-8191) 或 INTID_SPURIOUS (1023) 或 INTID_NMI
// ICV 重定向: icv_access → icv_iar_read()
```

### 4.4 ICC_EOIR0_EL1 / ICC_EOIR1_EL1 — 中断完成

```c
// arm_gicv3_cpuif.c:1645-1714
// 只写寄存器
// 流程: 组匹配检查 → icc_drop_prio() → icc_deactivate_irq() (如 EOImode=0)
// ICV 重定向: icv_eoir_write()
```

### 4.5 ICC_HPPIR0_EL1 / ICC_HPPIR1_EL1 — 最高优先级 Pending

```c
// arm_gicv3_cpuif.c:1189-1260
// 只读，返回最高优先级 pending 中断 INTID
// ICC_HPPIR0: Group 0 最高优先级（跨组时返回 SPURIOUS）
// ICC_HPPIR1: Group 1 最高优先级
// 安全: NS 访问 G0 始终返回 SPURIOUS
```

### 4.6 ICC_BPR0_EL1 / ICC_BPR1_EL1 — Binary Point

```c
// arm_gicv3_cpuif.c:2494-2499
// BPR 值范围: BPR0 [0-7], BPR1 [BPR0+1 .. 7]
// 控制组优先级 vs 子优先级分割
// ICC_CTLR_EL1.CBPR=1 时: BPR1 映射为 BPR0+1
```

### 4.7 ICC_RPR_EL1 — 运行优先级

```c
// arm_gicv3_cpuif.c:2522-2527 (reginfo) → icc_rpr_read
// 只读，返回当前最高活动中断优先级
// 无活动中断时返回 0xFF（空闲）
// NMI active 时返回 0x00 或 0x80
```

### 4.8 ICC_AP0R / ICC_AP1R — 活动优先级

```c
// arm_gicv3_cpuif.c:2501-2515
// AP0R: Group 0 活动优先级位图
// AP1R: Group 1 活动优先级位图（banked S/NS）
// 每位代表一个优先级组级别
// 用于运行优先级计算和抢占判定
```

### 4.9 ICC_CTLR_EL1 / ICC_CTLR_EL3 — 控制

```c
// 关键位:
// EOImode (bit 1): 0=合并 EOI, 1=分离（优先级下降+去激活）
// CBPR (bit 0): 1=BPR1 使用 BPR0+1
// ICC_CTLR_EL3: 控制 S/NS 两套 EOImode
```

### 4.10 ICC_SGI0R_EL1 / ICC_SGI1R_EL1 / ICC_ASGI1R_EL1 — SGI 生成

```c
// arm_gicv3_cpuif.c:2528-2530+ → icc_sgi0r_write, icc_sgi1r_write, icc_asgi1r_write
// 只写，写入后调用 icc_generate_sgi()
// 编码: Aff3[55:48]:Aff2[39:32]:Aff1[23:16] + TargetList[15:0] + IRM[40] + INTID[27:24]
```

### 4.11 ICC_IGRPEN0_EL1 / ICC_IGRPEN1_EL1 — 组使能

```c
// arm_gicv3_cpuif.c:660-687
// Enable bit[0]: 使能对应组的中断投递
// 写后副作用: gicv3_cpuif_update(cs)
```

### 4.12 ICC_DIR_EL1 — 去激活

```c
// arm_gicv3_cpuif.c:2516-2521 → icc_dir_write
// 只写，EOImode=1 时用于去激活中断
// 写入 INTID → icc_deactivate_irq(cs, irq)
```

### 4.13 ICC_SRE_EL1/2/3 — 系统寄存器使能

```c
// SRE.SRE (bit 0): 系统寄存器接口使能（QEMU 始终为 1）
// SRE.DFB (bit 1): 禁用 FIQ 旁路
// SRE.DIB (bit 2): 禁用 IRQ 旁路
// QEMU: SRE 通常固定为 0x7（全部使能），因为不支持传统 MMIO 接口
```

---

## 5. ICC → ICV 虚拟接口重定向

### 5.1 重定向机制

```c
// arm_gicv3_cpuif.c:85-106 — icv_access()
// 当 NS EL1 + HCR_EL2.{IMO/FMO} 时:
//   ICC_*_EL1 访问 → 透明重定向到 ICV_*_EL1
//   Guest 无感知

// 重定向映射:
// ICC_IAR1  → icv_iar_read      (读虚拟 List Register)
// ICC_EOIR1 → icv_eoir_write    (EOI 虚拟中断)
// ICC_PMR   → icv_pmr_read/write (虚拟 PMR)
// ICC_BPR1  → icv_bpr_read/write (虚拟 BPR)
// ICC_HPPIR1→ icv_hppir_read    (虚拟最高优先级)
// ICC_RPR   → icv_rpr_read      (虚拟运行优先级)
// ICC_CTLR  → icv_ctlr_read/write (虚拟控制)
// ICC_DIR   → icv_dir_write     (虚拟去激活)
```

### 5.2 ICV 核心函数

```c
// arm_gicv3_cpuif.c:
// icv_iar_read       (800-841)  — 扫描 LR 找最高优先级虚拟中断
// icv_eoir_write     (843-877)  — 虚拟 EOI
// icv_activate_irq   (765-798)  — 虚拟 Pending→Active
// icv_hppir_read     (740-762)  — 虚拟 HPPIR
// ich_highest_active_virt_prio (154-177) — 虚拟运行优先级
```

---

## 6. ICH Hypervisor 接口寄存器

### 6.1 ICH_HCR_EL2 — Hypervisor 控制

```c
// 关键位:
// En (bit 0): 虚拟 CPU 接口使能
// UIE (bit 1): 低优先级中断通知
// LRENPIE (bit 2): LR 空时通知
// NPIE (bit 3): 无 pending 通知
// TC (bit 10): 陷入 EL1 ICC 访问
// TALL0 (bit 11): 陷入 EL1 Group 0 访问
// TALL1 (bit 12): 陷入 EL1 Group 1 访问
// EOIcount [31:27]: 需要维护的 EOI 计数
```

### 6.2 ICH_VTR_EL2 — 虚拟类型（只读）

```c
// ListRegs [4:0]: List Register 数量 - 1
// PREbits [28:26]: 抢占位数 - 1
// PRIbits [31:29]: 优先级位数 - 1
// IDbits [25:23]: INTID 位数 - 1
// NMI [6]: 虚拟 NMI 支持
```

### 6.3 ICH_VMCR_EL2 — 虚拟机控制

```c
// 镜像 Guest 的 ICC_CTLR/PMR/BPR:
// VENG0 (bit 0): 虚拟 Group 0 使能
// VENG1 (bit 1): 虚拟 Group 1 使能
// VCBPR (bit 4): 虚拟 CBPR
// VEOImode (bit 9): 虚拟 EOImode
// VBPR0 [23:21]: 虚拟 BPR0
// VBPR1 [20:18]: 虚拟 BPR1
// VPMR [31:24]: 虚拟 PMR
```

### 6.4 ICH_LR<n>_EL2 — List Registers

```c
// gicv3_internal.h:239-260
// 64 位/每 LR:
// vINTID [31:0]:   虚拟 INTID
// pINTID [44:32]:  物理 INTID（HW=1 时）
// Priority [55:48]: 虚拟优先级
// NMI [59]:         NMI 位
// Group [60]:       0=Group 0, 1=Group 1
// HW [61]:          1=硬件中断（EOI 同时去激活物理中断）
// State [63:62]:    00=Invalid, 01=Pending, 10=Active, 11=P+A

// State 编码:
#define ICH_LR_EL2_STATE_INVALID    0
#define ICH_LR_EL2_STATE_PENDING    1
#define ICH_LR_EL2_STATE_ACTIVE     2
#define ICH_LR_EL2_STATE_ACTIVE_PENDING 3
```

### 6.5 ICH_AP0R / ICH_AP1R — 虚拟活动优先级

```c
// 与 ICC_AP0R/AP1R 类似，但用于虚拟中断
// 维护虚拟运行优先级
// ich_highest_active_virt_prio() 扫描这些位计算虚拟 RPR
```

---

## 7. 寄存器安全访问控制总结

### 7.1 NS 访问限制矩阵

| 寄存器 | DS=0 NS 读 | DS=0 NS 写 | 影响范围 |
|--------|-----------|-----------|---------|
| GICD_CTLR | 仅 ARE\|EN_GRP1NS\|RWP | 仅 EN_GRP1NS | 全局 |
| GICD_IGROUPR | Secure 中断位 RAZ | Secure 中断位 WI | 每32中断 |
| GICD_IGRPMODR | **完全 RAZ** | **完全 WI** | 每32中断 |
| GICD_ISENABLER | Secure 中断位 RAZ | Secure 中断位 WI | 每32中断 |
| GICD_ISPENDR | Secure 中断位 RAZ | Secure 中断位 WI | 每32中断 |
| GICD_IPRIORITYR | Secure 中断=0, NS 左移1位 | Secure WI, NS 右移+0x80 | 每中断 |
| GICD_IROUTER | Secure 中断 RAZ | Secure 中断 WI | 每中断 |
| ICC_PMR | PMR<0x80→返回0, ≥0x80→左移 | PMR<0x80→拒绝, 值右移+0x80 | 每CPU |
| ICC_IAR0 | SPURIOUS | — | 每CPU |
| GICR_IGRPMODR0 | **完全 RAZ** | **完全 WI** | 每CPU |

### 7.2 DS 位的影响

```
DS=0（默认，安全扩展启用）:
  - 三个组: G0, G1S, G1NS
  - NS 看到受限视图
  - 优先级空间分离 [0x00-0x7F] Secure, [0x80-0xFF] NS
  - GICD_CTLR 有 EN_GRP0/1S/1NS 三个使能

DS=1（安全扩展禁用）:
  - 两个组: G0, G1（无 S/NS 区分）
  - 所有访问等价（无安全过滤）
  - 优先级使用完整 8 位
  - GICD_CTLR 仅 EN_GRP0/EN_GRP1NS
  - 单向转换: DS=0→1 不可逆（除硬件 reset）
```

---

## 8. 状态位存储架构

### 8.1 分发器状态（SPI，INTID 32+）

```c
// arm_gicv3_common.h:260-274
typedef struct GICv3State {
    GIC_DECLARE_BITMAP(pending);      // pending 锁存
    GIC_DECLARE_BITMAP(active);       // active 位
    GIC_DECLARE_BITMAP(enabled);      // enable 位
    GIC_DECLARE_BITMAP(group);        // IGROUPR
    GIC_DECLARE_BITMAP(grpmod);       // IGRPMODR
    GIC_DECLARE_BITMAP(level);        // 电平输入
    GIC_DECLARE_BITMAP(edge_trigger); // 触发类型
    uint8_t gicd_ipriority[...];      // 优先级
    uint64_t gicd_irouter[...];       // 路由
    GICv3CPUState *gicd_irouter_target[...]; // 路由缓存
} GICv3State;
```

### 8.2 重分发器状态（SGI/PPI，INTID 0-31）

```c
// arm_gicv3_common.h:147-176
typedef struct GICv3CPUState {
    // SGI/PPI 状态（每 CPU）
    uint32_t gicr_ipendr0;        // SGI/PPI pending
    uint32_t gicr_iactiver0;      // SGI/PPI active
    uint32_t gicr_ienabler0;      // SGI/PPI enable
    uint32_t gicr_igroupr0;       // SGI/PPI 组
    uint32_t gicr_igrpmodr0;      // SGI/PPI 组修饰符
    uint32_t level;               // PPI 电平输入
    uint32_t edge_trigger;        // 触发类型
    uint8_t gicr_ipriorityr[32];  // SGI/PPI 优先级

    // CPU 接口状态
    uint8_t icc_pmr_el1;          // PMR
    uint64_t icc_bpr[3];          // BPR [G0/G1/G1NS]
    uint32_t icc_apr[3][4];       // APR [G0/G1/G1NS][0-3]
    uint64_t icc_ctlr_el1[2];    // CTLR [S/NS]
    uint64_t icc_ctlr_el3;       // CTLR EL3
    uint64_t icc_igrpen[3];      // 组使能

    // Hypervisor 接口
    uint64_t ich_hcr_el2;
    uint64_t ich_lr_el2[GICV3_LR_MAX];
    uint64_t ich_vmcr_el2;
    uint64_t ich_apr[3][4];       // 虚拟 APR

    // 计算缓存
    PendingIrq hppi;              // 最高优先级物理中断
    PendingIrq hpplpi;            // 最高优先级 LPI
} GICv3CPUState;
```

---

## 9. VMSTATE 迁移状态

### 9.1 主状态保存

```c
// arm_gicv3_common.c:187-222 — vmstate_gicv3_cpu
// 保存 GICv3CPUState 所有字段:
//   GICR: level, ctlr, statusr, waker, propbaser, pendbaser,
//          igroupr0, ienabler0, ipendr0, iactiver0, edge_trigger,
//          igrpmodr0, nsacr, ipriorityr[32]
//   ICC: ctlr_el1[2], pmr_el1, bpr[3], apr[3][4], igrpen[3], ctlr_el3
```

### 9.2 条件子段

```c
// arm_gicv3_common.c:105-222
// vmstate_gicv3_cpu_virt:  ICH 寄存器（当 virt_state_needed）
//   ich_apr[3][4], ich_hcr_el2, ich_lr_el2[], ich_vmcr_el2

// vmstate_gicv3_cpu_sre_el1: ICC_SRE_EL1（当 != 7）

// vmstate_gicv3_gicv4: GICv4 寄存器（当 revision > 3）
//   gicr_vpropbaser, gicr_vpendbaser

// vmstate_gicv3_cpu_nmi: NMI 寄存器（当 nmi_support）
//   gicr_inmir0
```

---

## 10. 寄存器写入副作用总结

| 寄存器写入 | 触发函数 | 作用 |
|-----------|---------|------|
| GICD_CTLR | `gicv3_full_update(s)` | 完全重算所有 CPU 的 hppi |
| GICD_IGROUPR/IGRPMODR | `gicv3_update(s, irq, 32)` | 重算受影响 32 个中断 |
| GICD_ISENABLER/ICENABLER | `gicv3_update(s, irq, 32)` | 同上 |
| GICD_ISPENDR/ICPENDR | `gicv3_update(s, irq, 32)` | 同上 |
| GICD_ISACTIVER/ICACTIVER | `gicv3_update(s, irq, 32)` | 同上 |
| GICD_IPRIORITYR | `gicv3_update(s, irq, 1)` | 重算单个中断 |
| GICD_IROUTER | `gicv3_update(s, irq, 1)` | 路由变更+缓存更新 |
| GICR_* | `gicv3_redist_update(cs)` | 重算此 CPU 的 SGI/PPI |
| ICC_PMR | `gicv3_cpuif_update(cs)` | 重新评估 CPU 输出线 |
| ICC_IAR0/1 | `icc_activate_irq()` | P→A + APR + 全局更新 |
| ICC_EOIR0/1 | `icc_drop_prio + deactivate` | A→I + 全局更新 |
| ICC_IGRPEN0/1 | `gicv3_cpuif_update(cs)` | 重新评估 CPU 输出线 |
| ICC_SGI*R | `icc_generate_sgi()` | 多 CPU 重分发器更新 |

---

## 11. 小结

| 寄存器层 | 文件 | 寄存器数 | 访问方式 |
|---------|------|---------|---------|
| **GICD** | arm_gicv3_dist.c | ~15 类 | MMIO (MemoryRegion) |
| **GICR** | arm_gicv3_redist.c | ~15 类 | MMIO (MemoryRegion) |
| **ICC** | arm_gicv3_cpuif.c | ~20 个 | 系统寄存器 (ARMCPRegInfo) |
| **ICH** | arm_gicv3_cpuif.c | ~8 个 | 系统寄存器 (ARMCPRegInfo) |
| **ICV** | arm_gicv3_cpuif.c | ~15 个 | ICC→ICV 透明重定向 |

**核心设计原则：**
1. **写即更新**：所有寄存器写入立即触发相关中断状态重算
2. **增量更新**：`gicv3_update(start, len)` 仅重算受影响范围
3. **安全隔离**：NS 访问通过组判定过滤，优先级通过移位隔离
4. **DS 单向门**：DS=0→1 不可逆，简化安全状态管理
5. **缓存加速**：IROUTER→target 缓存避免每次解码亲和性
6. **迁移完整性**：VMSTATE 保存所有硬件状态+条件子段（virt/GICv4/NMI）
