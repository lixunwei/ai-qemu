# KVM vGIC 设备后端与中断直通深度分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中 KVM vGIC 后端实现，包括 GICv2/GICv3/ITS 的 KVM 设备创建、MMIO 委托、中断注入路径、以及迁移状态保存/恢复。KVM vGIC 将中断控制器的模拟卸载到内核，极大提高了中断处理性能。

**关键源文件：**
- `hw/intc/arm_gic_kvm.c` — KVM GICv2 后端（~610行）
- `hw/intc/arm_gicv3_kvm.c` — KVM GICv3 后端（~975行）
- `hw/intc/arm_gicv3_its_kvm.c` — KVM ITS 后端（~265行）
- `hw/arm/virt.c` — 机器模型 GIC 创建与连线（~4300行）
- `target/arm/kvm.c` — KVM IRQ 注入接口

---

## 2. KVM vGIC 架构

### 2.1 TCG vs KVM 中断路径对比

```
TCG 模式:                           KVM 模式:
┌──────┐                            ┌──────┐
│ 设备 │                            │ 设备 │
└──┬───┘                            └──┬───┘
   │ qemu_set_irq()                    │ qemu_set_irq()
   ↓                                   ↓
┌──────────┐                        ┌──────────┐
│ QEMU GIC │                        │ KVM 转发 │
│ 完整模拟 │                        │ kvm_arm_ │
│ dist/    │                        │ set_irq()│
│ redist/  │                        └──┬───────┘
│ cpuif    │                           │ KVM_IRQ_LINE ioctl
└──┬───────┘                           ↓
   │ qemu_set_irq()                ┌──────────┐
   ↓                               │ KVM 内核 │
┌──────┐                           │ vGIC     │
│ vCPU │                           │ GICD/GICR│
│ 模拟 │                           │ /ICC     │
└──────┘                           └──┬───────┘
                                      │ 直接注入
                                      ↓
                                   ┌──────┐
                                   │ vCPU │
                                   │ 硬件 │
                                   └──────┘
```

### 2.2 QOM 类型选择

```c
// hw/arm/virt.c:1133-1137
if (gic_version == VIRT_GIC_VERSION_2) {
    gictype = gic_class_name();
    // KVM 时: "kvm-arm-gic"     (arm_gic_kvm.c)
    // TCG 时: "arm_gic"         (arm_gic.c)
} else {
    gictype = gicv3_class_name();
    // KVM 时: "kvm-arm-gicv3"   (arm_gicv3_kvm.c)
    // TCG 时: "arm-gicv3"       (arm_gicv3.c)
}
```

---

## 3. KVM GICv2 后端

### 3.1 设备创建

```c
// arm_gic_kvm.c:490-585 — kvm_arm_gic_realize()
// 步骤:
// 1. 调用父类 realize（GIC common 初始化）
// 2. 检查限制:
//    - 不支持安全扩展 :504-508
//    - 不支持虚拟化扩展 :510-514
// 3. 检查迁移支持，必要时添加 migration_blocker :516-522
// 4. 初始化 IRQ 和 MMIO: gic_init_irqs_and_mmio() :524
// 5. 设置 GSI 映射: kvm_irqchip_set_qemuirq_gsi() :526-529
// 6. 创建 KVM 设备:
//    kvm_create_device(KVM_DEV_TYPE_ARM_VGIC_V2) :533
// 7. 配置中断数量: KVM_DEV_ARM_VGIC_GRP_NR_IRQS :538-542
// 8. 初始化: KVM_DEV_ARM_VGIC_CTRL_INIT :544-546
// 9. 注册 MMIO 到 KVM:
//    kvm_arm_register_device() 委托 GICD/GICC MMIO :558-573
```

### 3.2 中断注入

```c
// arm_gic_kvm.c:44-73 — kvm_arm_gic_set_irq()
// 将 QEMU IRQ 编号转换为 KVM 格式:
//   外部中断 [0..N-1]: KVM_ARM_IRQ_TYPE_SPI, irq + GIC_INTERNAL
//   内部中断 [N..]: KVM_ARM_IRQ_TYPE_PPI, (cpu, irq%32)
// 最终调用 kvm_arm_set_irq() → KVM_IRQ_LINE ioctl

// arm_gic_kvm.c:75-82 — kvm_arm_gicv2_set_irq()
// QEMU 设备 GPIO 回调，转发到 kvm_arm_gic_set_irq()
```

### 3.3 状态保存/恢复（迁移）

```c
// arm_gic_kvm.c:88-104 — KVM 设备访问辅助函数
kvm_gicd_access(s, offset, cpu, &val, write)
  → kvm_device_access(dev_fd, KVM_DEV_ARM_VGIC_GRP_DIST_REGS, ...)
kvm_gicc_access(s, offset, cpu, &val, write)
  → kvm_device_access(dev_fd, KVM_DEV_ARM_VGIC_GRP_CPU_REGS, ...)

// arm_gic_kvm.c:288-474 — kvm_arm_gic_put/get()
// put (保存): 读 KVM GIC 寄存器 → 写入 QEMU GICState
// get (恢复): 读 QEMU GICState → 写 KVM GIC 寄存器
// 保存的寄存器:
//   GICD: CTLR, TYPER, IGROUPR, ISENABLER, ISPENDR, ISACTIVER,
//         IPRIORITYR, ITARGETSR, ICFGR
//   GICC: CTLR, PMR, BPR, ABPR, APR, AHPPIR
```

---

## 4. KVM GICv3 后端

### 4.1 设备创建

```c
// arm_gicv3_kvm.c:786-950 — kvm_arm_gicv3_realize()
// 步骤:
// 1. 调用父类 realize :794-798
// 2. 检查限制:
//    - 仅支持 revision=3 :800-803
//    - 不支持安全扩展 :805-809
//    - 不支持 NMI :811-814
//    - 不支持非零 first-cpu-idx :816-820
// 3. 初始化 IRQ 和 MMIO :822
// 4. 注册 ICC 系统寄存器 (gicv3_cpuif_reginfo) :824-828
// 5. 创建 KVM 设备:
//    kvm_create_device(KVM_DEV_TYPE_ARM_VGIC_V3) :831
// 6. 配置中断数量和重分发器区域 :837+
// 7. 注册 MMIO:
//    kvm_arm_register_device() GICD + GICR 区域 :883-900
```

### 4.2 五组状态访问接口

```c
// arm_gicv3_kvm.c:87-119

// 1. 分发器寄存器
kvm_gicd_access(s, offset, &val, write)
  → KVM_DEV_ARM_VGIC_GRP_DIST_REGS :90-92

// 2. 重分发器寄存器
kvm_gicr_access(s, offset, cpu, &val, write)
  → KVM_DEV_ARM_VGIC_GRP_REDIST_REGS :98-100

// 3. ICC 系统寄存器（CPU 接口）
kvm_gicc_access(s, reg, cpu, &val, write)
  → KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS :106-108

// 4. 中断电平状态
kvm_gic_line_level_access(s, irq, cpu, &val, write)
  → KVM_DEV_ARM_VGIC_GRP_LEVEL_INFO :114-118

// 5. CTRL 组（初始化/地址配置）
//    在 realize 时使用
```

### 4.3 状态保存/恢复

```c
// arm_gicv3_kvm.c:317-620 — kvm_arm_gicv3_put/get()

// GICD 保存/恢复:
//   GICD_CTLR, GICD_STATUSR
//   per-IRQ: IGROUPR, IGRPMODR, ISENABLER, ISPENDR,
//            ISACTIVER, IPRIORITYR, ICFGR, IROUTER

// GICR 保存/恢复 (per-CPU):
//   GICR_CTLR, GICR_STATUSR, GICR_WAKER
//   SGI/PPI: IGROUPR0, IGRPMODR0, ISENABLER0, ISPENDR0,
//            ISACTIVER0, IPRIORITYR, ICFGR0/1
//   LPI: PROPBASER, PENDBASER

// ICC 保存/恢复 (per-CPU):
//   ICC_SRE_EL1, ICC_CTLR_EL1, ICC_IGRPEN0/1_EL1
//   ICC_PMR_EL1, ICC_BPR0/1_EL1
//   ICC_AP0R/AP1R (active priority)

// 中断电平:
//   KVM_DEV_ARM_VGIC_GRP_LEVEL_INFO — 保存边沿中断电平状态
```

---

## 5. KVM ITS 后端

### 5.1 设备创建

```c
// arm_gicv3_its_kvm.c:92-128 — kvm_arm_its_realize()
// 步骤:
// 1. kvm_create_device(KVM_DEV_TYPE_ARM_VGIC_ITS) :96
// 2. KVM_DEV_ARM_VGIC_CTRL_INIT 初始化 :103-104
// 3. kvm_arm_register_device() 注册 ITS MMIO :107-108
// 4. 添加到 GICv3: gicv3_add_its() :110
// 5. 检查迁移支持 (GITS_CTLR 属性) :114-123
// 6. 启用: kvm_msi_via_irqfd_allowed = true :127
```

### 5.2 ITS 迁移

```c
// arm_gicv3_its_kvm.c:85-86 — kvm_arm_its_pre_save()
// 保存: KVM_DEV_ARM_ITS_SAVE_TABLES 命令
// 让 KVM 将 ITS 内部状态导出到 Guest 内存中的 ITS 表

// arm_gicv3_its_kvm.c:130-198 — kvm_arm_its_post_load()
// 恢复步骤:
// 1. 恢复 GITS 寄存器 (GITS_CBASER, GITS_CWRITER, GITS_CREADR,
//    GITS_BASER[0..7], GITS_CTLR) 通过 KVM_DEV_ARM_VGIC_GRP_ITS_REGS
// 2. KVM_DEV_ARM_ITS_RESTORE_TABLES 命令
//    让 KVM 从 Guest 内存重建 ITS 表
```

### 5.3 ITS 表保存/恢复策略

```
保存流程:
  QEMU → KVM_DEV_ARM_ITS_SAVE_TABLES → KVM
    KVM 将 DTEntry/CTEntry/ITEntry 序列化到 Guest 内存

恢复流程:
  QEMU → 恢复 GITS_BASER[0..7]（表基地址）
  QEMU → 恢复 GITS_CBASER/CWRITER/CREADR（命令队列）
  QEMU → KVM_DEV_ARM_ITS_RESTORE_TABLES → KVM
    KVM 从 Guest 内存反序列化重建 ITS 状态
  QEMU → 恢复 GITS_CTLR（最后，触发使能）
```

---

## 6. 机器模型 GIC 创建与连线

### 6.1 create_gic() 流程

```c
// hw/arm/virt.c:1122-1293 — create_gic()
// 1. 选择 GIC 类型 :1133-1137
// 2. 设置属性 (num-cpu, num-irq, revision, ...) :1148-1156
// 3. sysbus_realize() :1158
// 4. 映射 MMIO:
//    GICD → vms->memmap[VIRT_GIC_DIST] :1159-1162
//    GICC → vms->memmap[VIRT_GIC_CPU] (v2) :1163-1166
//    GICR → vms->memmap[VIRT_GIC_REDIST] (v3) :1168-1200
// 5. 连线 per-CPU 中断输出:
//    GIC 输出 → CPU IRQ/FIQ/VIRQ/VFIQ 输入 :1233-1284
// 6. 连线 per-CPU 维护中断:
//    maintenance_irq → GIC PPI 输入 :1268-1274
// 7. 创建 ITS (v3) :1288-1292
```

### 6.2 中断输出连线

```c
// 连线映射:
// GIC sysbus_irq[cpu*4+0] → CPU IRQ    (物理 IRQ)
// GIC sysbus_irq[cpu*4+1] → CPU FIQ    (物理 FIQ)
// GIC sysbus_irq[cpu*4+2] → CPU VIRQ   (虚拟 IRQ)
// GIC sysbus_irq[cpu*4+3] → CPU VFIQ   (虚拟 FIQ)
// GIC maintenance[cpu]     → GIC PPI    (维护中断)
```

---

## 7. KVM IRQ 注入接口

### 7.1 注入路径

```c
// 设备 GPIO → kvm_arm_gicv2/3_set_irq()
//           → kvm_arm_gic_set_irq()    :44-73
//           → kvm_arm_set_irq()        (target/arm/kvm.c:1658)
//           → kvm_set_irq() → KVM_IRQ_LINE ioctl

// KVM IRQ 编码:
// [31:24] irq_type  — SPI/PPI/SPI_VIRT/PPI_VIRT
// [23:16] vcpu_idx  — vCPU 索引
// [15:0]  irq_num   — 中断号
```

### 7.2 irqfd 机制

```c
// KVM ITS 启用 irqfd:
// arm_gicv3_its_kvm.c:127
// kvm_msi_via_irqfd_allowed = true
// 允许 eventfd 直接触发 KVM 中断注入
// 用于 VFIO 设备直通时的中断高速路径
```

---

## 8. TCG vs KVM vGIC 功能对比

| 特性 | TCG GIC | KVM vGIC |
|------|---------|----------|
| 安全扩展 | ✅ 支持 | ❌ 不支持 |
| 虚拟化扩展 | ✅ 支持 | ✅ KVM 内核处理 |
| NMI (FEAT_NMI) | ✅ 支持 | ❌ 不支持 |
| GICv2m | ✅ QEMU 模拟 | ✅ QEMU 模拟 |
| ITS | ✅ QEMU 模拟 | ✅ KVM ITS |
| 迁移 | ✅ VMSTATE | ✅ KVM 寄存器组 |
| 性能 | 较慢（全模拟） | 快（内核处理） |
| 调试 | 可追踪 | 不可见 |
| first-cpu-idx | ✅ 支持 | ❌ 仅从0开始 |

---

## 9. 迁移状态组总览

| 状态组 | KVM 常量 | 用途 | GICv2 | GICv3 |
|--------|----------|------|-------|-------|
| DIST_REGS | `KVM_DEV_ARM_VGIC_GRP_DIST_REGS` | GICD 寄存器 | ✅ | ✅ |
| CPU_REGS | `KVM_DEV_ARM_VGIC_GRP_CPU_REGS` | GICC 寄存器 | ✅ | — |
| REDIST_REGS | `KVM_DEV_ARM_VGIC_GRP_REDIST_REGS` | GICR 寄存器 | — | ✅ |
| CPU_SYSREGS | `KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS` | ICC 系统寄存器 | — | ✅ |
| LEVEL_INFO | `KVM_DEV_ARM_VGIC_GRP_LEVEL_INFO` | 电平中断状态 | — | ✅ |
| ITS_REGS | `KVM_DEV_ARM_VGIC_GRP_ITS_REGS` | ITS 寄存器 | — | ✅ |
| NR_IRQS | `KVM_DEV_ARM_VGIC_GRP_NR_IRQS` | 中断数量配置 | ✅ | ✅ |
| CTRL | `KVM_DEV_ARM_VGIC_GRP_CTRL` | 初始化/地址 | ✅ | ✅ |

---

## 10. 迁移流程

### 10.1 GICv3 KVM 迁移保存

```
1. QEMU 暂停 vCPU
2. kvm_arm_gicv3_get() 从 KVM 读取所有状态:
   a. 读 GICD 寄存器 (GRP_DIST_REGS)
   b. 读每个 CPU 的 GICR 寄存器 (GRP_REDIST_REGS)
   c. 读每个 CPU 的 ICC 寄存器 (GRP_CPU_SYSREGS)
   d. 读中断电平状态 (GRP_LEVEL_INFO)
3. kvm_arm_its_pre_save():
   a. KVM_DEV_ARM_ITS_SAVE_TABLES → Guest 内存
4. VMSTATE 序列化所有状态到迁移流
```

### 10.2 GICv3 KVM 迁移恢复

```
1. VMSTATE 反序列化所有状态
2. kvm_arm_gicv3_put() 写入 KVM:
   a. 写 GICD 寄存器 (GRP_DIST_REGS)
   b. 写每个 CPU 的 GICR 寄存器 (GRP_REDIST_REGS)
   c. 写每个 CPU 的 ICC 寄存器 (GRP_CPU_SYSREGS)
   d. 写中断电平状态 (GRP_LEVEL_INFO)
3. kvm_arm_its_post_load():
   a. 恢复 GITS_BASER[0..7] (GRP_ITS_REGS)
   b. 恢复 GITS_CBASER/CWRITER/CREADR
   c. KVM_DEV_ARM_ITS_RESTORE_TABLES
   d. 恢复 GITS_CTLR（使能）
4. 恢复 vCPU 运行
```

---

## 11. 小结

| 维度 | 实现 |
|------|------|
| **KVM GICv2** | kvm_create_device(V2) + GICD/GICC MMIO 委托 + 2 组状态（DIST+CPU） |
| **KVM GICv3** | kvm_create_device(V3) + GICD/GICR MMIO 委托 + 5 组状态（DIST+REDIST+SYSREG+LEVEL+ITS） |
| **KVM ITS** | 独立 KVM 设备 + SAVE/RESTORE_TABLES 命令 + irqfd MSI 直通 |
| **中断注入** | kvm_arm_set_irq() → KVM_IRQ_LINE ioctl + irqfd 高速路径 |
| **迁移策略** | 暂停→读 KVM 状态→序列化→传输→反序列化→写 KVM 状态→恢复 |
| **限制** | 无安全扩展、无 NMI、无非零 first-cpu-idx |
| **性能** | 中断处理完全在内核完成，避免 QEMU 用户态开销 |
