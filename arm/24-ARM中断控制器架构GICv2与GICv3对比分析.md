# ARM 中断控制器架构总览：GICv2 与 GICv3 对比分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中 ARM GIC（Generic Interrupt Controller）的两代实现——GICv2 和 GICv3 的架构差异、寄存器模型、中断流转路径、以及机器模型如何选择 GIC 版本。通过源码级对比揭示两代 GIC 的设计演进和 QEMU 实现策略。

**关键源文件：**
- `hw/intc/arm_gic.c` — GICv2 主实现（~2200行）
- `hw/intc/arm_gic_common.c` — GICv2 公共/迁移（~300行）
- `include/hw/intc/arm_gic_common.h` — GICv2 状态结构
- `hw/intc/arm_gicv3.c` — GICv3 主实现（~480行）
- `hw/intc/arm_gicv3_dist.c` — GICv3 分发器（~820行）
- `hw/intc/arm_gicv3_redist.c` — GICv3 重分发器（~1200行）
- `hw/intc/arm_gicv3_cpuif.c` — GICv3 CPU 接口（~2700行）
- `hw/arm/virt.c` — 机器模型 GIC 选择（~4300行）

---

## 2. 架构对比总览

### 2.1 核心差异

| 特性 | GICv2 | GICv3 |
|------|-------|-------|
| 最大 CPU 数 | **8** (`GIC_NCPU`) | **无硬限制**（MPIDR 亲和性） |
| CPU 接口 | **MMIO** (GICC_*) | **系统寄存器** (ICC_*_ELx) |
| 重分发器 | **无** | **有** (GICR，per-CPU) |
| 中断路由 | GICD_ITARGETSR (8位位图) | GICD_IROUTER (亲和性) |
| LPI 支持 | **无** | 有 (INTID 8192+) |
| ITS 支持 | **无** | 有 (MSI 翻译) |
| MSI 支持 | GICv2m (外挂) | ITS (原生) |
| 安全扩展 | 可选 | 始终存在（可 DS=1 禁用） |
| SGI 源追踪 | IAR 返回源 CPU ID | 无源 CPU ID |
| 虚拟化接口 | GICH MMIO | ICH 系统寄存器 |
| NMI 支持 | **无** | 有 (FEAT_NMI) |
| 亲和性路由 | 无 (ARE=0) | 始终启用 (ARE=1) |

### 2.2 架构图对比

```
GICv2:                              GICv3:
┌─────────┐                         ┌─────────┐
│  设备    │                         │  设备    │
└───┬─────┘                         └───┬─────┘
    │ GPIO                              │ GPIO
    ↓                                   ↓
┌─────────────┐                     ┌─────────────┐
│   GICD      │                     │   GICD      │
│  (分发器)   │                     │  (分发器)   │
│ ITARGETSR   │                     │  IROUTER    │
│ 8位CPU位图  │                     │ 亲和性路由  │
└──┬──┬──┬──┘                       └──┬──────────┘
   │  │  │                              │
   ↓  ↓  ↓                         ┌───┼───┐
┌────┐┌────┐┌────┐                 ↓   ↓   ↓
│GICC││GICC││GICC│              ┌────┐┌────┐┌────┐
│ 0  ││ 1  ││ N  │              │GICR││GICR││GICR│
│MMIO││MMIO││MMIO│              │ 0  ││ 1  ││ N  │
└──┬─┘└──┬─┘└──┬─┘              └─┬──┘└─┬──┘└─┬──┘
   ↓     ↓     ↓                  ↓     ↓     ↓
  CPU0  CPU1  CPUN              ┌────┐┌────┐┌────┐
                                │ICC ││ICC ││ICC │
                                │SysR││SysR││SysR│
                                └─┬──┘└─┬──┘└─┬──┘
                                  ↓     ↓     ↓
                                 CPU0  CPU1  CPUN
```

---

## 3. QOM 类型层次

### 3.1 GICv2 QOM 树

```c
// include/hw/intc/arm_gic_common.h:154-166
TYPE_ARM_GIC_COMMON = "arm_gic_common"  (SysBusDevice 子类)
  └→ TYPE_ARM_GIC = "arm_gic"           (TCG 实现)
  └→ TYPE_KVM_ARM_GIC = "kvm-arm-gic"   (KVM 实现)

// include/hw/intc/arm_gic.h:78-90
// ARMGICClass 包含 parent_reset、post_load 等虚方法
```

### 3.2 GICv3 QOM 树

```c
// include/hw/intc/arm_gicv3_common.h:87-140
TYPE_ARM_GICV3_COMMON = "arm-gicv3-common"  (SysBusDevice 子类)
  └→ TYPE_ARM_GICV3 = "arm-gicv3"           (TCG 实现)
  └→ TYPE_KVM_ARM_GICV3 = "kvm-arm-gicv3"   (KVM 实现)

// ITS 作为独立设备:
TYPE_ARM_GICV3_ITS = "arm-gicv3-its"
TYPE_KVM_ARM_GICV3_ITS = "arm-gicv3-its-kvm"
```

---

## 4. 状态结构对比

### 4.1 GICv2 — GICState

```c
// arm_gic_common.h:65-151
struct GICState {
    // 输出线（每 CPU）
    qemu_irq parent_irq[8];       // IRQ 输出
    qemu_irq parent_fiq[8];       // FIQ 输出
    qemu_irq parent_virq[8];      // 虚拟 IRQ
    qemu_irq parent_vfiq[8];      // 虚拟 FIQ

    // 控制
    uint32_t ctlr;                 // GICD_CTLR
    uint32_t cpu_ctlr[16];        // GICC_CTLR（含 vCPU）

    // 中断状态（统一）
    gic_irq_state irq_state[1020]; // 每中断状态（pending/active/level/group/...）
    uint8_t irq_target[1020];      // GICD_ITARGETSR: 8位 CPU 位图
    uint8_t priority1[32][8];      // SGI/PPI 优先级（per-CPU）
    uint8_t priority2[988];        // SPI 优先级（全局）

    // SGI 源追踪
    uint8_t sgi_pending[16][8];    // SGI pending 位图[SGI号][目标CPU]

    // CPU 接口状态
    uint16_t priority_mask[16];    // PMR
    uint16_t running_priority[16]; // 运行优先级
    uint16_t current_pending[16];  // 当前最高 pending
    uint8_t bpr[16];               // BPR
    uint8_t abpr[16];              // ABPR（NS 别名）
    uint32_t apr[4][8];            // APR

    // 虚拟化接口
    uint32_t h_hcr[8];            // GICH_HCR
    uint32_t h_lr[64][8];         // GICH_LR（MMIO）

    // MMIO 区域
    MemoryRegion iomem;            // GICD
    MemoryRegion cpuiomem[9];      // GICC（MMIO）
    MemoryRegion vifaceiomem[9];   // GICH（MMIO）
    MemoryRegion vcpuiomem;        // GICV（虚拟 CPU 接口 MMIO）
};
```

### 4.2 GICv3 — GICv3State + GICv3CPUState

```c
// arm_gicv3_common.h
// GICv3State:  分发器状态（位图，非 per-IRQ 结构）
// GICv3CPUState: per-CPU 状态（GICR + ICC + ICH 合一）
// 对比 GICv2: 无 MemoryRegion cpuiomem（CPU 接口通过系统寄存器）
// 新增: LPI 状态(propbaser/pendbaser)、NMI、GICv4
```

### 4.3 关键结构差异

| 特性 | GICv2 | GICv3 |
|------|-------|-------|
| 中断状态存储 | `gic_irq_state` 结构数组 | 独立位图（pending/active/enabled/...） |
| 路由存储 | `irq_target[irq]` 8位位图 | `gicd_irouter[irq]` 64位亲和性 + 缓存 |
| SGI pending | `sgi_pending[sgi][cpu]` 源位图 | `gicr_ipendr0` 无源追踪 |
| CPU 接口 | MMIO MemoryRegion | ARMCPRegInfo 系统寄存器 |
| 优先级 | 分 SGI/PPI + SPI 两个数组 | 分 GICR (0-31) + GICD (32+) |
| 虚拟化 | GICH MMIO (h_hcr/h_lr) | ICH 系统寄存器 (ich_hcr/ich_lr) |

---

## 5. 中断流转对比

### 5.1 GICv2 中断流程

```c
// arm_gic.c:381-420 — gic_set_irq()
// 与 GICv3 结构相同: SPI→分发器, PPI→per-CPU
// 但路由使用 GICD_ITARGETSR 8 位位图

// arm_gic.c:162-230 — gic_update_internal()
// 遍历所有 CPU:
//   gic_get_best_irq() 找最高优先级
//   比较 priority_mask 和 running_priority
//   Group 0 + FIQ_EN → FIQ; 否则 → IRQ
//   直接 qemu_set_irq()

// arm_gic.c:599-645 — gic_acknowledge_irq()
// 关键差异: SGI 返回 (源CPU << 10 | INTID)
//   gic_clear_pending_sgi() 清除特定源 CPU 的 pending
```

### 5.2 GICv3 中断流程

```c
// arm_gicv3.c:372-400 — gicv3_set_irq()
// SPI→gicv3_dist_set_irq, PPI→gicv3_redist_set_irq

// arm_gicv3.c:257-333 — gicv3_update()
// 使用 gicd_irouter_target[] 缓存路由
// 最高优先级存入 hppi 结构

// arm_gicv3_cpuif.c:1262-1311 — icc_iar1_read()
// SGI 无源 CPU: 直接返回 INTID
// 通过系统寄存器访问（非 MMIO）
```

### 5.3 流程对比表

| 阶段 | GICv2 | GICv3 |
|------|-------|-------|
| 输入 | `gic_set_irq` | `gicv3_set_irq` |
| SPI 路由 | `irq_target[]` 位图 | `gicd_irouter_target[]` 缓存 |
| 优先级 | `gic_get_best_irq` 全局扫描 | `gicv3_update_noirqset` 增量更新 |
| FIQ/IRQ 决策 | `GICC_CTLR.FIQ_EN + Group 0` | `组→安全状态→信号` 映射 |
| 应答 | MMIO 读 `GICC_IAR` | 系统寄存器 `ICC_IAR1_EL1` |
| SGI 源 | IAR 返回 `(srcCPU<<10\|ID)` | IAR 仅返回 INTID |
| EOI | MMIO 写 `GICC_EOIR` | 系统寄存器 `ICC_EOIR1_EL1` |
| 虚拟接口 | MMIO (`GICV_*`) | ICV 系统寄存器重定向 |

---

## 6. 寄存器模拟对比

### 6.1 分发器寄存器

| 寄存器 | GICv2 | GICv3 |
|--------|-------|-------|
| GICD_CTLR | `gic_dist_writeb` :1191+ | `gicd_writel` :622-660 |
| GICD_TYPER | 含 CPUNumber | CPUNumber=0 (ARE=1) |
| GICD_IGROUPR | 同 | 同 + NS RAZ/WI |
| **GICD_ITARGETSR** | **8位 CPU 位图** :1024-1080 | **不存在** |
| **GICD_IROUTER** | **不存在** | **64位亲和性** :585-597 |
| GICD_IPRIORITYR | 直接读写 | NS 优先级移位 |
| GICD_ICFGR | 同 | 同 |

### 6.2 CPU 接口寄存器

| GICv2 (MMIO) | GICv3 (系统寄存器) | 差异 |
|-------------|-------------------|------|
| GICC_CTLR | ICC_CTLR_EL1 | 位域不同 |
| GICC_PMR | ICC_PMR_EL1 | GICv3 有 NS 移位 |
| GICC_BPR | ICC_BPR0/1_EL1 | GICv3 分 G0/G1 |
| GICC_IAR | ICC_IAR0/1_EL1 | GICv3 分组，SGI 无源 ID |
| GICC_EOIR | ICC_EOIR0/1_EL1 | GICv3 分组 |
| GICC_HPPIR | ICC_HPPIR0/1_EL1 | GICv3 分组 |
| GICC_APR | ICC_AP0R/AP1R | GICv3 分 G0/G1 |
| GICC_DIR | ICC_DIR_EL1 | 功能相同 |
| GICC_RPR | ICC_RPR_EL1 | 功能相同 |
| **GICV_*** | **ICV_*** | MMIO→系统寄存器重定向 |

---

## 7. 机器模型 GIC 选择

### 7.1 virt 机器 gic-version 属性

```c
// hw/arm/virt.c:2337-2380 — finalize_gic_version_do()
// gic-version 属性值:
//   "2"    → VIRT_GIC_VERSION_2
//   "3"    → VIRT_GIC_VERSION_3
//   "4"    → VIRT_GIC_VERSION_4 (GICv3 revision=4)
//   "host" → KVM 主机版本
//   "max"  → 最高支持版本
//   未指定 → NOSEL（自动选择）

// 自动选择逻辑 (NOSEL):
//   如果支持 v2 且 CPU ≤ 8 → GICv2
//   否则如果支持 v3 → GICv3
//   否则报错（>8 CPU 必须 GICv3）
```

### 7.2 GIC 创建

```c
// hw/arm/virt.c:1122-1200 — create_gic()
if (gic_version == VIRT_GIC_VERSION_2) {
    gictype = gic_class_name();     // "arm_gic" 或 "kvm-arm-gic"
} else {
    gictype = gicv3_class_name();   // "arm-gicv3" 或 "kvm-arm-gicv3"
}
// 创建设备 → 设置属性 → realize
// GICv3 额外: 创建重分发器区域、ITS
```

### 7.3 FDT/ACPI 差异

```c
// hw/arm/virt.c:915-990 — GIC DT 节点
// GICv2: compatible = "arm,cortex-a15-gic"
//        2 个 MMIO 区域（GICD + GICC）
// GICv3: compatible = "arm,gic-v3"
//        2+ 个 MMIO 区域（GICD + GICR[s]）
//        可选 ITS 子节点

// hw/arm/virt.c:876-894 — ITS 节点（仅 GICv3）
// hw/arm/virt.c:896-912 — GICv2m 节点（仅 GICv2）
```

---

## 8. GICv2m — MSI 桥接

```c
// hw/intc/arm_gicv2m.c:1-199
// TYPE_ARM_GICV2M = "arm-gicv2m"
// 功能: 将 MSI 写入转换为 GICv2 SPI
// 原理:
//   PCIe 设备写 GICv2m MMIO 寄存器
//   GICv2m 将写入转换为 qemu_set_irq() 调用
//   触发 GICv2 SPI 输入
// 限制:
//   不如 GICv3 ITS 灵活（无设备/事件翻译）
//   SPI 范围固定
```

---

## 9. KVM 加速对比

### 9.1 GICv2 KVM

```c
// hw/intc/arm_gic_kvm.c
// TYPE_KVM_ARM_GIC
// 使用 KVM_CREATE_DEVICE 创建内核 GIC
// GICD + GICC MMIO 由内核处理
// 状态通过 KVM_DEV_ARM_VGIC_GRP_* 保存/恢复
```

### 9.2 GICv3 KVM

```c
// hw/intc/arm_gicv3_kvm.c
// TYPE_KVM_ARM_GICV3
// 使用 KVM_CREATE_DEVICE 创建内核 GICv3
// GICD + GICR MMIO 由内核处理
// ICC 系统寄存器由内核直接拦截
// 额外: ITS KVM (arm_gicv3_its_kvm.c)
```

### 9.3 KVM 差异

| 特性 | GICv2 KVM | GICv3 KVM |
|------|----------|----------|
| 设备类型 | `KVM_DEV_TYPE_ARM_VGIC_V2` | `KVM_DEV_TYPE_ARM_VGIC_V3` |
| MMIO 拦截 | GICD + GICC | GICD + GICR |
| 系统寄存器 | N/A | ICC/ICH 由 KVM |
| ITS | N/A | KVM ITS 设备 |
| 迁移 | GICD + GICC 状态 | GICD + GICR + ICC + ICH |

---

## 10. 迁移状态对比

### 10.1 GICv2 VMSTATE

```c
// arm_gic_common.c:61-129
// 保存: ctlr, cpu_ctlr, irq_state[], irq_target[], priority*,
//       running_priority, current_pending, bpr, abpr, apr, nsapr,
//       sgi_pending[][]
// 子段: virt_extn (h_hcr, h_lr, h_misr, h_apr)
```

### 10.2 GICv3 VMSTATE

```c
// arm_gicv3_common.c:187-312
// 主状态: GICR (level, ctlr, ipendr0, iactiver0, ...) +
//         ICC (ctlr_el1, pmr, bpr, apr, igrpen, ctlr_el3)
// 子段:
//   virt: ich_apr, ich_hcr, ich_lr, ich_vmcr
//   sre_el1: icc_sre_el1
//   gicv4: gicr_vpropbaser, gicr_vpendbaser
//   nmi: gicr_inmir0
// 分发器: pending/active/enabled/group/... 位图数组
```

---

## 11. 小结

| 维度 | GICv2 | GICv3 |
|------|-------|-------|
| **文件** | arm_gic.c (统一~2200行) | dist.c+redist.c+cpuif.c (~4700行) |
| **CPU 限制** | 8 | 无限制 |
| **路由** | 8 位 CPU 位图 | 亲和性路由 + 缓存 |
| **CPU 接口** | MMIO | 系统寄存器 |
| **SGI 源** | IAR 包含源 CPU | 无源追踪 |
| **LPI/ITS** | 无（GICv2m 外挂 MSI） | 原生（ITS 翻译） |
| **NMI** | 无 | FEAT_NMI |
| **虚拟化** | GICH/GICV MMIO | ICH/ICV 系统寄存器 |
| **安全隔离** | 可选 | 始终有（DS 可禁用） |
| **QEMU 选择** | CPU ≤ 8 自动 | CPU > 8 或明确指定 |

**演进趋势：**
1. **MMIO → 系统寄存器**：CPU 接口访问开销从内存映射降低到寄存器访问
2. **位图路由 → 亲和性路由**：打破 8 CPU 限制，支持大规模系统
3. **外挂 MSI → 原生 ITS**：PCIe MSI/MSI-X 集成到 GIC 架构
4. **统一 → 分层**：GICv2 单文件 vs GICv3 分发器+重分发器+CPU接口分层
5. **QEMU 默认**：≤8 CPU 倾向 GICv2（兼容性），>8 CPU 强制 GICv3
