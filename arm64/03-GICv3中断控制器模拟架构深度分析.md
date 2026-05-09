# ARM GICv3 中断控制器模拟架构深度分析

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 关键 commit：`f35f0f1ca1`（ICC_AP1Rn NS 写入修复）、`2a3c965516`（HVF vGIC 基础设施）  
> 参考文档：ARM IHI 0069（GICv3/v4 Architecture Specification）、`docs/system/arm/virt.rst`  

---

## 目录

- [第一部分：架构总览](#第一部分架构总览)
  - [1. GICv3 硬件架构回顾](#1-gicv3-硬件架构回顾)
  - [2. QEMU GICv3 QOM 类层次](#2-qemu-gicv3-qom-类层次)
  - [3. 核心数据结构](#3-核心数据结构)
  - [4. 初始化与机器集成](#4-初始化与机器集成)
- [第二部分：Distributor（GICD）](#第二部分distributorgicd)
  - [5. GICD 寄存器框架](#5-gicd-寄存器框架)
  - [6. SPI 管理与中断位图](#6-spi-管理与中断位图)
  - [7. 亲和性路由（Affinity Routing）](#7-亲和性路由affinity-routing)
- [第三部分：Redistributor（GICR）](#第三部分redistributorgicr)
  - [8. GICR 寄存器框架](#8-gicr-寄存器框架)
  - [9. SGI / PPI 管理](#9-sgi--ppi-管理)
  - [10. LPI 配置与 Pending 表](#10-lpi-配置与-pending-表)
  - [11. Redistributor 内存布局](#11-redistributor-内存布局)
- [第四部分：CPU Interface（GICC/ICC）](#第四部分cpu-interfacegicc-icc)
  - [12. ICC 系统寄存器总表](#12-icc-系统寄存器总表)
  - [13. 中断确认（IAR）与激活](#13-中断确认iar与激活)
  - [14. 中断结束（EOI）与去激活](#14-中断结束eoi与去激活)
  - [15. 优先级与抢占逻辑](#15-优先级与抢占逻辑)
  - [16. SGI 生成路径](#16-sgi-生成路径)
  - [17. Group 0 / Group 1 安全模型](#17-group-0--group-1-安全模型)
- [第五部分：虚拟中断接口（ICH/ICV）](#第五部分虚拟中断接口ichicv)
  - [18. List Register 机制](#18-list-register-机制)
  - [19. 虚拟中断交付流程](#19-虚拟中断交付流程)
  - [20. 维护中断](#20-维护中断)
- [第六部分：ITS（Interrupt Translation Service）](#第六部分itsinterrupt-translation-service)
  - [21. ITS 架构与 QOM 层次](#21-its-架构与-qom-层次)
  - [22. ITS 表结构](#22-its-表结构)
  - [23. ITS 命令队列](#23-its-命令队列)
  - [24. LPI 翻译流程](#24-lpi-翻译流程)
  - [25. MSI/MSI-X 与 ITS 集成](#25-msimsi-x-与-its-集成)
- [第七部分：KVM vGIC 集成](#第七部分kvm-vgic-集成)
  - [26. KVM GICv3 后端架构](#26-kvm-gicv3-后端架构)
  - [27. 状态同步机制](#27-状态同步机制)
  - [28. KVM ITS](#28-kvm-its)
  - [29. 迁移（VMState）](#29-迁移vmstate)
- [第八部分：中断完整流程](#第八部分中断完整流程)
  - [30. SPI 端到端流程](#30-spi-端到端流程)
  - [31. LPI/MSI 端到端流程](#31-lpimsi-端到端流程)
  - [32. 状态重计算与级联更新](#32-状态重计算与级联更新)
- [附录](#附录)
  - [A. 关键源文件索引](#a-关键源文件索引)
  - [B. GICD/GICR/ICC 寄存器速查表](#b-gicdgicricc-寄存器速查表)
  - [C. ITS 命令速查表](#c-its-命令速查表)

---

# 第一部分：架构总览

## 1. GICv3 硬件架构回顾

ARM GICv3 是 ARMv8 架构标准中断控制器，支持最多 1020 个 SPI、16 个 SGI、16 个 PPI，以及 LPI（通过 ITS 扩展）。架构由四个主要组件构成：

```
┌─────────────────────────────────────────────────────────────────┐
│                        GICv3 架构                                │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐         ┌──────────────┐  │
│  │  Distributor  │    │    ITS       │         │  ITS (可选)   │  │
│  │   (GICD)      │    │ 翻译服务     │         │              │  │
│  │              │    │              │         │              │  │
│  │ SPI 管理     │    │ DevID+EventID│         │              │  │
│  │ 路由         │    │  → LPI+CPU   │         │              │  │
│  └──────┬───────┘    └──────┬───────┘         └──────────────┘  │
│         │                   │                                    │
│    ┌────┴────┬──────────────┴────┬──────────────┐               │
│    │         │                   │              │               │
│  ┌─┴────┐ ┌─┴────┐          ┌─┴────┐      ┌─┴────┐          │
│  │GICR 0│ │GICR 1│   ...    │GICR n│      │GICR m│          │
│  │SGI/PPI│ │SGI/PPI│         │SGI/PPI│      │SGI/PPI│          │
│  │LPI    │ │LPI    │         │LPI    │      │LPI    │          │
│  └──┬───┘ └──┬───┘          └──┬───┘      └──┬───┘          │
│     │        │                  │             │               │
│  ┌──┴───┐ ┌──┴───┐          ┌──┴───┐      ┌──┴───┐          │
│  │ICC 0 │ │ICC 1 │   ...    │ICC n │      │ICC m │          │
│  │CPU IF│ │CPU IF│          │CPU IF│      │CPU IF│          │
│  └──┬───┘ └──┬───┘          └──┴───┘      └──┴───┘          │
│     │        │                  │             │               │
│  ┌──┴───┐ ┌──┴───┐          ┌──┴───┐      ┌──┴───┐          │
│  │CPU 0 │ │CPU 1 │   ...    │CPU n │      │CPU m │          │
│  │IRQ   │ │IRQ   │          │IRQ   │      │IRQ   │          │
│  │FIQ   │ │FIQ   │          │FIQ   │      │FIQ   │          │
│  └──────┘ └──────┘          └──────┘      └──────┘          │
└─────────────────────────────────────────────────────────────────┘
```

**中断类型**：

| 类型 | ID 范围 | 来源 | 管理者 |
|------|---------|------|--------|
| SGI（软件生成中断） | 0-15 | CPU 写 ICC_SGI*R | Redistributor |
| PPI（私有外设中断） | 16-31 | 定时器/PMU 等 | Redistributor |
| SPI（共享外设中断） | 32-1019 | 外设 | Distributor |
| LPI（局部外设中断） | 8192+ | PCIe MSI 等 | ITS + Redistributor |

## 2. QEMU GICv3 QOM 类层次

```
SysBusDevice
  └── TYPE_ARM_GICV3_COMMON ("arm-gicv3-common")    [抽象基类]
        │   arm_gicv3_common.c:624-648
        │   — 公共属性（num-cpu, num-irq, revision）
        │   — VMState 迁移描述
        │   — Linux boot 钩子
        │
        ├── TYPE_ARM_GICV3 ("arm-gicv3")              [TCG 模拟实现]
        │     arm_gicv3.c:455-478
        │     — 全软件模拟 GICD/GICR/ICC
        │     — 中断状态机完整实现
        │
        ├── TYPE_KVM_ARM_GICV3 ("kvm-arm-gicv3")      [KVM 内核实现]
        │     arm_gicv3_kvm.c:786-880
        │     — 委托内核 vGIC 处理中断
        │     — QEMU 侧仅做状态同步
        │
        └── TYPE_ARM_GICV3_HVF ("arm-gicv3-hvf")      [macOS HVF 实现]
              arm_gicv3_hvf.c
              — Apple Hypervisor Framework 后端

SysBusDevice
  └── TYPE_ARM_GICV3_ITS_COMMON ("arm-gicv3-its-common")  [ITS 抽象基类]
        │   arm_gicv3_its_common.c:147-154
        │
        ├── TYPE_ARM_GICV3_ITS ("arm-gicv3-its")       [TCG ITS]
        │     arm_gicv3_its.c:2024-2030
        │
        └── TYPE_KVM_ARM_ITS ("arm-its-kvm")           [KVM ITS]
              arm_gicv3_its_kvm.c:258-264
```

## 3. 核心数据结构

### 3.1 GICv3State（Distributor 级）

`arm_gicv3_common.h:225-281`：

```c
struct GICv3State {
    SysBusDevice parent_obj;

    // Distributor 状态
    uint32_t gicd_ctlr;                    // 控制寄存器
    uint32_t gicd_statusr[2];              // 状态（安全/非安全）

    // 中断位图 — SPI 范围
    GIC_DECLARE_BITMAP(group);             // 分组（G0/G1）
    GIC_DECLARE_BITMAP(grpmod);            // 组修改
    GIC_DECLARE_BITMAP(enabled);           // 使能
    GIC_DECLARE_BITMAP(pending);           // 挂起
    GIC_DECLARE_BITMAP(active);            // 激活
    GIC_DECLARE_BITMAP(level);             // 电平状态
    GIC_DECLARE_BITMAP(edge_trigger);      // 边沿/电平触发
    GIC_DECLARE_BITMAP(nmi);               // NMI 标记

    uint8_t gicd_ipriority[GICV3_MAXIRQ]; // 优先级
    uint64_t gicd_irouter[GICV3_MAXIRQ];  // 路由目标

    // 系统配置
    uint32_t num_cpu;                      // CPU 数量
    uint32_t num_irq;                      // 中断数量（含 SPI）
    uint32_t revision;                     // GICv3 / GICv4
    bool security_extn;                    // 安全扩展
    bool nmi_support;                      // NMI 支持

    // 每 CPU 状态
    GICv3CPUState *cpu;                    // CPU 状态数组
    // Redistributor 区域
    int nb_redist_regions;
    GICv3RedistRegion *redist_regions;

    // MMIO
    MemoryRegion iomem_dist;              // Distributor MMIO
};
```

### 3.2 GICv3CPUState（每 CPU 状态）

`arm_gicv3_common.h:127-213`：

```c
struct GICv3CPUState {
    GICv3State *gic;
    CPUState *cpu;
    qemu_irq parent_irq;                  // → CPU IRQ 输入
    qemu_irq parent_fiq;                  // → CPU FIQ 输入
    qemu_irq parent_virq;                 // → CPU VIRQ 输入
    qemu_irq parent_vfiq;                 // → CPU VFIQ 输入
    qemu_irq parent_nmi;                  // → CPU NMI 输入

    // Redistributor 寄存器
    uint32_t gicr_ctlr;
    uint32_t gicr_waker;
    uint64_t gicr_propbaser;              // LPI 配置表基址
    uint64_t gicr_pendbaser;              // LPI Pending 表基址

    // SGI/PPI 状态位图
    uint32_t gicr_igroupr0;               // 分组
    uint32_t gicr_ienabler0;              // 使能
    uint32_t gicr_ipendr0;               // 挂起
    uint32_t gicr_iactiver0;             // 激活
    uint32_t gicr_icfgr0, gicr_icfgr1;  // 配置（触发模式）
    uint32_t edge_trigger;                // 边沿检测
    uint32_t level;                       // 电平状态
    uint8_t gicr_ipriorityr[32];         // SGI/PPI 优先级

    // ICC（CPU Interface）寄存器
    uint64_t icc_ctlr_el1[2];            // [NS, S]
    uint64_t icc_ctlr_el3;
    uint64_t icc_pmr_el1;                // 优先级掩码
    uint64_t icc_bpr[3];                 // 二进制点（G0, G1, G1NS）
    uint64_t icc_apr[3][4];              // 活动优先级寄存器
    uint64_t icc_igrpen[3];             // 组使能
    uint64_t icc_sre_el1;

    // 虚拟接口（Hypervisor）
    uint64_t ich_hcr_el2;                // Hypervisor 控制
    uint64_t ich_vmcr_el2;               // 虚拟机控制
    uint64_t ich_lr_el2[GICV3_LR_MAX];  // List Registers
    uint64_t ich_apr[3][4];              // 虚拟活动优先级

    // 最高优先级挂起中断缓存
    GICv3CPUHPPIState hppi;              // {irq, prio, grp, nmi}
};
```

### 3.3 位图操作宏

`arm_gicv3_common.h:55-79, 283-309`：

```c
// 声明位图
#define GIC_DECLARE_BITMAP(field) \
    uint32_t field[GICV3_BMP_SIZE]

// 访问器宏 — 自动生成 get/set/test 函数
GICV3_BITMAP_ACCESSORS(enabled)
GICV3_BITMAP_ACCESSORS(pending)
GICV3_BITMAP_ACCESSORS(active)
// ... 为每个位图生成类型安全的访问函数
```

## 4. 初始化与机器集成

### 4.1 virt 机器创建 GICv3

`virt.c:1122-1265`：

```c
static void create_gic(VirtMachineState *vms, ...) {
    // 1. 选择 GIC 类型
    if (kvm_enabled()) {
        gictype = "kvm-arm-gicv3";
    } else {
        gictype = "arm-gicv3";
    }

    // 2. 创建 GIC 设备并设置属性
    vms->gic = qdev_new(gictype);
    qdev_prop_set_uint32(vms->gic, "num-cpu", smp_cpus);
    qdev_prop_set_uint32(vms->gic, "num-irq", NUM_IRQS + 32);
    qdev_prop_set_uint32(vms->gic, "revision", 3);

    // 3. 映射 MMIO 区域
    sysbus_mmio_map(gicbusdev, 0, vms->memmap[VIRT_GIC_DIST].base);
    // Redistributor 区域
    sysbus_mmio_map(gicbusdev, 1, vms->memmap[VIRT_GIC_REDIST].base);
    // 可选的第二个 Redistributor 区域（>123 CPU 时）
    sysbus_mmio_map(gicbusdev, 2, vms->memmap[VIRT_HIGH_GIC_REDIST2].base);

    // 4. 连接每 CPU 中断线
    for (i = 0; i < smp_cpus; i++) {
        // GIC 输出 → CPU 输入
        // IRQ, FIQ, VIRQ, VFIQ, NMI 五条线
        sysbus_connect_irq(gicbusdev, i, cpu_irq[i]);
        // PPI 输入（定时器等）
        qdev_connect_gpio_out(cpudev, GTIMER_PHYS, gic_ppi[i]);
    }
}
```

### 4.2 MMIO 地址布局

| 组件 | 基地址 | 大小 | 说明 |
|------|--------|------|------|
| GICD | `0x0800_0000` | 64 KiB | Distributor |
| GICR | `0x080A_0000` | N × 128 KiB | 每 CPU Redistributor |
| ITS | `0x0808_0000` | 128 KiB | ITS（可选） |

Redistributor 帧大小（`arm_gicv3_common.h:42-47`）：
- GICv3：128 KiB（两个 64 KiB 帧：RD_base + SGI_base）
- GICv4：256 KiB（增加 VLPI 帧）

---

# 第二部分：Distributor（GICD）

## 5. GICD 寄存器框架

Distributor 实现在 `arm_gicv3_dist.c`（960 行），通过 MMIO 读写分发：

### 5.1 读写入口

```c
// arm_gicv3_dist.c:860-939
// MemoryRegionOps 定义读写入口
static uint64_t gicd_read(void *opaque, hwaddr offset, unsigned size) {
    // 根据 size 分派到 gicd_readb/w/l/q
}
static void gicd_write(void *opaque, hwaddr offset, uint64_t data, unsigned size) {
    // 根据 size 分派到 gicd_writeb/w/l/q
}
```

### 5.2 关键寄存器处理

| 寄存器 | 偏移 | 读处理 | 写处理 | 功能 |
|--------|------|--------|--------|------|
| `GICD_CTLR` | 0x0000 | `:382-402` | `:622-660` | 全局使能（EnableGrp0/1, ARE） |
| `GICD_TYPER` | 0x0004 | `:403-433` | — | 类型信息（IRQ数、CPU数、安全扩展） |
| `GICD_IIDR` | 0x0008 | `:443-448` | — | 实现者标识 |
| `GICD_IGROUPR<n>` | 0x0080+ | `:455-470` | `:664-679` | 中断分组（G0/G1） |
| `GICD_ISENABLER<n>` | 0x0100+ | `:472-479` | `:680-687` | 中断使能（置位） |
| `GICD_ICENABLER<n>` | 0x0180+ | — | `:680-687` | 中断使能（清除） |
| `GICD_ISPENDR<n>` | 0x0200+ | `:480-487` | `:688-695` | 挂起状态（置位） |
| `GICD_ICPENDR<n>` | 0x0280+ | — | `:688-695` | 挂起状态（清除） |
| `GICD_ISACTIVER<n>` | 0x0300+ | `:488-495` | `:696-703` | 激活状态（置位） |
| `GICD_ICACTIVER<n>` | 0x0380+ | — | `:696-703` | 激活状态（清除） |
| `GICD_IPRIORITYR<n>` | 0x0400+ | `:496-507` | `:704-716` | 中断优先级（每中断8位） |
| `GICD_ITARGETSR<n>` | 0x0800+ | `:508-511` | `:718-720` | **RAZ/WI**（ARE模式） |
| `GICD_ICFGR<n>` | 0x0C00+ | `:512-532` | `:721-747` | 触发配置（边沿/电平） |
| `GICD_IGRPMODR<n>` | 0x0D00+ | `:534-553` | `:748-765` | 组修改（安全扩展） |
| `GICD_NSACR<n>` | 0x0E00+ | `:554-574` | `:767-785` | 非安全访问控制 |
| `GICD_IROUTER<n>` | 0x6000+ | `:585-597` | `:800-813` | 亲和性路由目标 |

## 6. SPI 管理与中断位图

### 6.1 中断状态机

每个 SPI 都有独立的状态位：

```
      ┌─────────┐  ISPENDR/边沿   ┌──────────┐  IAR 读取   ┌────────────┐
      │ Inactive │ ──────────────→ │ Pending  │ ──────────→ │   Active   │
      │(无活动)  │                 │(等待确认) │             │(正在处理)   │
      └─────────┘                 └──────────┘             └────────────┘
           ↑                           ↑                        │
           │ EOI(Deactivate)           │                        │
           └───────────────────────────┼────────────────────────┘
                                       │ 新中断触发
                                  ┌────┴───────┐
                                  │Active&Pending│
                                  │(处理中+新挂起)│
                                  └─────────────┘
```

### 6.2 SPI 输入路径

`gicv3_dist_set_irq()`（`arm_gicv3.c` → `arm_gicv3_dist.c:941-959`）：

```c
void gicv3_dist_set_irq(GICv3State *s, int irq, int level) {
    // 1. 更新电平状态
    if (level) {
        s->level[irq/32] |= (1 << (irq % 32));
    } else {
        s->level[irq/32] &= ~(1 << (irq % 32));
    }

    // 2. 边沿触发: 上升沿 → 置 pending
    //    电平触发: 高电平持续 → pending 跟随电平
    if (edge_trigger && rising_edge) {
        s->pending[irq/32] |= (1 << (irq % 32));
    }

    // 3. 触发全局状态重计算
    gicv3_update(s, irq, 1);
}
```

### 6.3 挂起状态判定

`gicd_int_pending()`（`arm_gicv3.c:55-99`）综合考虑：

```
中断是否真正挂起 = pending 位 AND enabled 位
                  AND (group 使能 in GICD_CTLR)
                  AND (NOT 被安全策略阻止)
```

## 7. 亲和性路由（Affinity Routing）

### 7.1 GICD_IROUTER

每个 SPI 都有一个 64 位 `GICD_IROUTER<n>` 寄存器（`arm_gicv3_dist.c:585-597, 800-813`），编码目标 CPU：

```
GICD_IROUTER<n>:
  [31]     IRM   — 0: 路由到特定 CPU
                   1: 任意可用 CPU（1-of-N）
  [23:16]  Aff2  — 亲和性级别 2
  [15:8]   Aff1  — 亲和性级别 1
  [7:0]    Aff0  — 亲和性级别 0
  [39:32]  Aff3  — 亲和性级别 3
```

### 7.2 ARE 始终启用

QEMU GICv3 在复位时总是启用亲和性路由（`arm_gicv3_common.c:546-551`）：

```c
// 复位时 ARE_S 和 ARE_NS 都设为 1
s->gicd_ctlr = GICD_CTLR_ARE_S | GICD_CTLR_ARE_NS;
```

这意味着：
- `GICD_ITARGETSR` 寄存器为 RAZ/WI（`arm_gicv3_dist.c:508-511, 718-720`）
- 所有 SPI 路由通过 `GICD_IROUTER` 而非旧式目标位图

### 7.3 路由目标缓存

`arm_gicv3_common.h:270-274`：

```c
// 为每个 SPI 缓存目标 CPU 指针，避免每次路由时遍历 CPU 列表
GICv3CPUState *gicd_irouter_target[GICV3_MAXIRQ];
```

---

# 第三部分：Redistributor（GICR）

## 8. GICR 寄存器框架

Redistributor 实现在 `arm_gicv3_redist.c`（1,187 行），每 CPU 一个实例。

### 8.1 关键寄存器

| 寄存器 | 读处理 | 写处理 | 功能 |
|--------|--------|--------|------|
| `GICR_CTLR` | `:351-356` | `:488-505` | 控制（Enable LPIs） |
| `GICR_TYPER` | `:357-362` | — | 类型（Aff值、Last、LPI支持） |
| `GICR_WAKER` | `:369-371` | `:509-525` | 唤醒控制 |
| `GICR_PROPBASER` | `:372-378` | `:527-533` | LPI 属性表基址 |
| `GICR_PENDBASER` | `:378-383` | `:533-538` | LPI Pending 表基址 |
| `GICR_IGROUPR0` | `:384-390` | `:539-545` | SGI/PPI 分组 |
| `GICR_ISENABLER0` | `:391-394` | `:546-551` | SGI/PPI 使能 |
| `GICR_IPRIORITYR` | `:409-419` | `:564-572` | SGI/PPI 优先级 |
| `GICR_ICFGR0/1` | `:425-437` | `:580-599` | SGI/PPI 触发配置 |
| `GICR_VPROPBASER` | `:467-473` | `:633-639` | vLPI 属性表（GICv4） |
| `GICR_VPENDBASER` | `:473-478` | `:639-644` | vLPI Pending 表（GICv4） |

### 8.2 Redistributor 帧解码

`gicv3_redist_read/write()`（`arm_gicv3_redist.c:711-825`）根据偏移解码两个帧：

```
帧 0（RD_base）: 偏移 0x0000-0xFFFF
  — GICR_CTLR, GICR_TYPER, GICR_WAKER
  — GICR_PROPBASER, GICR_PENDBASER
  — GICR_VPROPBASER, GICR_VPENDBASER

帧 1（SGI_base）: 偏移 0x10000-0x1FFFF
  — GICR_IGROUPR0, GICR_ISENABLER0, GICR_ICENABLER0
  — GICR_ISPENDR0, GICR_ICPENDR0
  — GICR_ISACTIVER0, GICR_ICACTIVER0
  — GICR_IPRIORITYR, GICR_ICFGR0/1
```

## 9. SGI / PPI 管理

### 9.1 PPI 输入

`gicv3_redist_set_irq()`（`arm_gicv3_redist.c:1135-1154`）：

```c
void gicv3_redist_set_irq(GICv3CPUState *cs, int irq, int level) {
    // PPI 范围: 16-31
    // 更新电平、检测边沿、置 pending
    // 调用 gicv3_redist_update() 重计算
}
```

### 9.2 SGI 发送

`gicv3_redist_send_sgi()`（`arm_gicv3_redist.c:1156-1187`）：

```c
void gicv3_redist_send_sgi(GICv3CPUState *cs, int grp, int irq, bool ns) {
    // SGI 范围: 0-15
    // 直接设置目标 CPU 的 pending 位
    // 无需边沿检测（SGI 始终为边沿触发语义）
    cs->gicr_ipendr0 |= (1 << irq);
    gicv3_redist_update(cs);
}
```

## 10. LPI 配置与 Pending 表

### 10.1 表结构

LPI 配置由客户机内存中的两张表控制：

```
GICR_PROPBASER 指向的属性表（每 LPI 1 字节）:
  ┌────────────────────────────────┐
  │ Byte[LPI_ID - 8192]           │
  │  [7:2] Priority               │
  │  [1]   NMI (if supported)     │
  │  [0]   Enable                 │
  └────────────────────────────────┘

GICR_PENDBASER 指向的 Pending 表（每 LPI 1 位）:
  ┌────────────────────────────────┐
  │ Bit[LPI_ID - 8192]            │
  │  1 = 挂起                      │
  │  0 = 无挂起                    │
  └────────────────────────────────┘
```

### 10.2 LPI 扫描与更新

`update_for_one_lpi()`、`update_for_all_lpis()`（`arm_gicv3_redist.c:99-170`）：
- 从客户机内存读取属性表和 pending 表
- 检查使能、优先级，与当前 HPPI 比较
- 若找到更高优先级的 LPI，更新 `cs->hppi`

`gicv3_redist_process_lpi()`（`arm_gicv3_redist.c:897-911`）：
- ITS 触发 LPI 后调用
- 在 pending 表中设置对应位
- 触发 Redistributor 更新

## 11. Redistributor 内存布局

`GICv3RedistRegion`（`arm_gicv3_common.h:219-223`）：

```c
struct GICv3RedistRegion {
    GICv3State *gic;
    uint32_t cpuidx;              // 起始 CPU 索引
    uint32_t count;               // 本区域包含的 CPU 数
    MemoryRegion iomem;           // MMIO 区域
};
```

`gicv3_init_irqs_and_mmio()`（`arm_gicv3_common.c:314-370`）创建 MMIO 区域：

```
每 CPU 的 Redistributor 区域连续排列:
  GICR_base + 0 × REDIST_SIZE  → CPU 0
  GICR_base + 1 × REDIST_SIZE  → CPU 1
  ...
  GICR_base + N × REDIST_SIZE  → CPU N
```

---

# 第四部分：CPU Interface（GICC/ICC）

## 12. ICC 系统寄存器总表

CPU Interface 通过 ICC_* 系统寄存器访问，实现在 `arm_gicv3_cpuif.c`（3,192 行）。

| 寄存器 | 读函数 | 写函数 | 位置 | 功能 |
|--------|--------|--------|------|------|
| `ICC_PMR_EL1` | `icc_pmr_read` | `icc_pmr_write` | `:1104-1156` | 优先级掩码 |
| `ICC_IAR0_EL1` | `icc_iar0_read` | — | `:1262-1283` | Group 0 中断确认 |
| `ICC_IAR1_EL1` | `icc_iar1_read` | — | `:1285-1310` | Group 1 中断确认 |
| `ICC_EOIR0_EL1` | — | `icc_eoir_write` | `:1645-1714` | Group 0 中断结束 |
| `ICC_EOIR1_EL1` | — | `icc_eoir_write` | `:1645-1714` | Group 1 中断结束 |
| `ICC_HPPIR0_EL1` | `icc_hppir0_read` | — | `:1716-1741` | 最高优先级挂起（G0） |
| `ICC_HPPIR1_EL1` | `icc_hppir1_read` | — | `:1716-1741` | 最高优先级挂起（G1） |
| `ICC_BPR0_EL1` | `icc_bpr_read` | `icc_bpr_write` | `:1744-1825` | 二进制点（G0） |
| `ICC_BPR1_EL1` | `icc_bpr_read` | `icc_bpr_write` | `:1744-1825` | 二进制点（G1） |
| `ICC_AP0R<n>_EL1` | `icc_ap_read` | `icc_ap_write` | `:1827-1914` | 活动优先级（G0） |
| `ICC_AP1R<n>_EL1` | `icc_ap_read` | `icc_ap_write` | `:1827-1914` | 活动优先级（G1） |
| `ICC_DIR_EL1` | — | `icc_dir_write` | `:1916-1997` | 去激活中断 |
| `ICC_RPR_EL1` | `icc_rpr_read` | — | `:1999-2038` | 运行优先级 |
| `ICC_SGI0R_EL1` | — | `icc_sgi0r_write` | `:2095-2103` | 生成 Group 0 SGI |
| `ICC_SGI1R_EL1` | — | `icc_sgi1r_write` | `:2105-2115` | 生成 Group 1 SGI |
| `ICC_CTLR_EL1` | `icc_ctlr_el1_read` | `icc_ctlr_el1_write` | `:2197-2241` | 控制（EOImode等） |
| `ICC_CTLR_EL3` | `icc_ctlr_el3_read` | `icc_ctlr_el3_write` | `:2244-2298` | EL3 控制 |
| `ICC_IGRPEN0_EL1` | `icc_igrpen_read` | `icc_igrpen_write` | `:2131-2170` | Group 0 使能 |
| `ICC_IGRPEN1_EL1` | `icc_igrpen_read` | `icc_igrpen_write` | `:2131-2170` | Group 1 使能 |
| `ICC_SRE_EL1/2/3` | — | — | `:2598-2649` | 系统寄存器使能（RAO/WI） |

## 13. 中断确认（IAR）与激活

### 13.1 IAR 读取流程

`icc_iar1_read()`（`arm_gicv3_cpuif.c:1285-1310`）：

```
CPU 读取 ICC_IAR1_EL1:
  1. 检查 Group 1 是否使能（IGRPEN1）
  2. 获取 HPPI（最高优先级挂起中断）
  3. 检查分组匹配（该中断必须属于 G1）
  4. 检查优先级能否抢占当前运行优先级
     → 不能抢占: 返回 0x3FF（伪中断 ID）
  5. 调用 icc_activate_irq() 激活中断
  6. 返回中断 ID
```

### 13.2 中断激活

`icc_activate_irq()`（`arm_gicv3_cpuif.c:1158-1187`）：

```c
static void icc_activate_irq(GICv3CPUState *cs, int irq) {
    // 1. 在 APR（Active Priority Register）中记录优先级
    //    → 用于追踪抢占嵌套
    icc_set_apr(cs, grp, prio_to_apr_bit(cs->hppi.prio));

    // 2. 标记中断为 active（清除 pending）
    if (irq < 32) {
        // SGI/PPI: Redistributor 位图
        cs->gicr_iactiver0 |= (1 << irq);
        cs->gicr_ipendr0 &= ~(1 << irq);
    } else {
        // SPI: Distributor 位图
        gicv3_gicd_active_set(s, irq);
        gicv3_gicd_pending_clear(s, irq);
    }

    // 3. 触发重计算（寻找下一个 HPPI）
    gicv3_update(cs->gic, irq, 1);
}
```

## 14. 中断结束（EOI）与去激活

### 14.1 EOI 模式

GICv3 支持两种 EOI 模式（由 `ICC_CTLR_EL1.EOImode` 控制）：

```
EOImode = 0（默认）:
  ICC_EOIR 写入同时完成:
  1. 优先级降低（Priority Drop）— 恢复运行优先级
  2. 去激活（Deactivate）— 清除 active 位

EOImode = 1（分离模式）:
  ICC_EOIR 写入仅完成:
  1. 优先级降低
  ICC_DIR 写入完成:
  2. 去激活
```

### 14.2 EOI 写入

`icc_eoir_write()`（`arm_gicv3_cpuif.c:1645-1714`）：

```c
static void icc_eoir_write(CPUARMState *env, ..., uint64_t value) {
    int irq = value & 0xFFFFFF;  // 提取中断 ID

    // 1. 安全/分组验证
    //    — 检查 IRQ 属于正确的组
    //    — 检查安全状态匹配

    // 2. 优先级降低
    icc_drop_prio(cs, grp);      // 清除 APR 中最高位

    // 3. 如果 EOImode == 0，同时去激活
    if (!icc_eoi_split(env, cs)) {
        icc_deactivate_irq(cs, irq);
    }

    // 4. 重计算 CPU 接口状态
    gicv3_cpuif_update(cs);
}
```

### 14.3 优先级降低

`icc_drop_prio()`（`arm_gicv3_cpuif.c:1340-1379`）：

```c
static int icc_drop_prio(GICv3CPUState *cs, int grp) {
    // 在 APR 寄存器中找到最高 active 优先级位，清除它
    // APR[0] 的最低位对应最高优先级
    for (i = 0; i < 4; i++) {
        if (cs->icc_apr[grp][i]) {
            int bit = ctz32(cs->icc_apr[grp][i]);
            cs->icc_apr[grp][i] &= ~(1 << bit);
            return (i * 32 + bit) << (8 - prebits);
        }
    }
    return 0xFF;  // 无活动中断
}
```

## 15. 优先级与抢占逻辑

### 15.1 优先级评估

`gicv3_cpuif_update()`（`arm_gicv3_cpuif.c:1046-1102`）是核心更新函数：

```c
void gicv3_cpuif_update(GICv3CPUState *cs) {
    // 1. 获取 HPPI（已缓存在 cs->hppi）
    int irq = cs->hppi.irq;
    int prio = cs->hppi.prio;
    int grp = cs->hppi.grp;

    // 2. 检查能否抢占
    if (!icc_hppi_can_preempt(cs)) {
        // 不能抢占 → 不改变 CPU 中断线
        qemu_irq_lower(cs->parent_irq);
        qemu_irq_lower(cs->parent_fiq);
        return;
    }

    // 3. 检查组使能
    if (!icc_grp_enabled(cs, grp)) {
        return;
    }

    // 4. 检查 PMR（优先级掩码）
    if (prio >= cs->icc_pmr_el1) {
        return;  // 被掩码屏蔽
    }

    // 5. 根据组决定信号线
    if (grp == GICV3_G0) {
        qemu_irq_raise(cs->parent_fiq);   // Group 0 → FIQ
    } else {
        qemu_irq_raise(cs->parent_irq);   // Group 1 → IRQ
    }
}
```

### 15.2 抢占检查

`icc_hppi_can_preempt()` 使用 BPR（Binary Point Register）进行优先级分组：

```
BPR 将 8 位优先级分为两部分:
  [7:BPR+1]  组优先级（用于抢占判断）
  [BPR:0]    子优先级（同组内排序，不触发抢占）

例: BPR = 3
  优先级 0x20 → 组优先级 = 0x20 (0010_0xxx)
  优先级 0x28 → 组优先级 = 0x20 (0010_0xxx) → 同组，不抢占
  优先级 0x10 → 组优先级 = 0x10 (0001_0xxx) → 更高，可抢占
```

抢占条件：`HPPI 组优先级 < 当前运行优先级的组优先级`

### 15.3 运行优先级

`icc_rpr_read()`（`arm_gicv3_cpuif.c:1999-2038`）：

```c
// 运行优先级 = APR 寄存器中最高 active 优先级
// 无 active 中断时运行优先级 = 0xFF（idle）
```

## 16. SGI 生成路径

`icc_generate_sgi()`（`arm_gicv3_cpuif.c:2042-2093`）：

```c
static void icc_generate_sgi(CPUARMState *env, uint64_t value, int grp, bool ns) {
    int irq = extract64(value, 24, 4);     // SGI ID [0-15]
    bool irm = extract64(value, 40, 1);    // 目标模式

    if (irm) {
        // IRM = 1: 发送到除自身外所有 CPU
        for (i = 0; i < s->num_cpu; i++) {
            if (i != current_cpu_idx) {
                gicv3_redist_send_sgi(s->cpu[i], grp, irq, ns);
            }
        }
    } else {
        // IRM = 0: 按亲和性 + 目标列表发送
        uint16_t tgtlist = extract64(value, 0, 16);
        uint64_t aff = extract64(value, 16, 24) | extract64(value, 48, 8) << 24;
        // 在匹配亲和性的 CPU 中，按 tgtlist 位图发送
        for (i = 0; i < s->num_cpu; i++) {
            if (match_affinity(s->cpu[i], aff) && (tgtlist & (1 << aff0))) {
                gicv3_redist_send_sgi(s->cpu[i], grp, irq, ns);
            }
        }
    }
}
```

## 17. Group 0 / Group 1 安全模型

### 17.1 中断分组

| 组 | 安全状态 | 信号线 | EL3 路由 |
|----|----------|--------|----------|
| Group 0 | Secure | **FIQ** | 总是路由到 EL3 |
| Secure Group 1 | Secure | IRQ（在安全世界） | 可选路由 EL3 |
| Non-secure Group 1 | Non-secure | IRQ（在非安全世界） | 可选路由 EL3 |

### 17.2 HPPIR 值计算

`icc_hppir0_value()`（`:1189-1224`）和 `icc_hppir1_value()`（`:1226-1260`）：

```
读取 ICC_HPPIR0_EL1（Group 0）:
  — 若 HPPI 属于 G0 且优先级满足 → 返回 IRQ ID
  — 若 HPPI 属于 G1 → 返回 1022（表示 G1 中断挂起）
  — 否则 → 返回 1023（无挂起中断）

读取 ICC_HPPIR1_EL1（Group 1）:
  — 若 HPPI 属于 G1 且优先级满足 → 返回 IRQ ID
  — 若 HPPI 属于 G0 → 返回 1023
  — 否则 → 返回 1023
```

---

# 第五部分：虚拟中断接口（ICH/ICV）

## 18. List Register 机制

Hypervisor 通过 ICH_LR<n>_EL2 向虚拟机注入虚拟中断。

### 18.1 List Register 格式

```
ICH_LR<n>_EL2（64 位）:
  [63:62] State    — 00=Invalid, 01=Pending, 10=Active, 11=Active+Pending
  [61]    HW       — 1=硬件中断（关联物理 ID）
  [60]    Group    — 0=Group 0, 1=Group 1
  [55:48] Priority — 虚拟优先级
  [44:32] pINTID   — 物理中断 ID（HW=1 时）
  [31:0]  vINTID   — 虚拟中断 ID
```

### 18.2 ICH 寄存器

| 寄存器 | 位置 | 功能 |
|--------|------|------|
| `ICH_HCR_EL2` | `:2746-2770` | Hypervisor 控制（使能虚拟接口、维护中断） |
| `ICH_VMCR_EL2` | `:2772-2801` | 虚拟机 PMR/BPR/CTLR/IGRPEN 镜像 |
| `ICH_LR0-15_EL2` | `:2803-2866` | 虚拟中断 List Registers |
| `ICH_MISR_EL2` | `:2887-2894` | 维护中断状态 |
| `ICH_AP0R/AP1R` | `:2925-2940` | 虚拟活动优先级 |

## 19. 虚拟中断交付流程

`gicv3_cpuif_virt_irq_fiq_update()`（`:471-524`）：

```
Hypervisor 写入 ICH_LR<n>_EL2（注入虚拟中断）
  → gicv3_cpuif_virt_update()                    [:526-558]
    → gicv3_cpuif_virt_irq_fiq_update()          [:471-524]
      1. 扫描所有 LR，找到最高优先级 Pending 虚拟中断
         → hppvi_index()                          [:179-256]
      2. 检查虚拟 PMR、虚拟 BPR、虚拟组使能
      3. 检查能否抢占虚拟运行优先级
         → icv_hppi_can_preempt()                 [:293-347]
      4. 路由到 vIRQ 或 vFIQ
         → qemu_irq_raise(cs->parent_virq/vfiq)
    → 检查维护中断条件
      → maintenance_interrupt_state()             [:434-468]
      → qemu_irq_raise/lower(gicv3_maintenance_interrupt)
```

### 19.1 虚拟 IAR/EOI

当 Guest（EL1）读写 ICC_IAR/EOIR 时，若 ICH_HCR_EL2 启用虚拟接口，这些访问被重定向到 ICV_* 虚拟寄存器处理函数，操作 List Register 而非物理状态。

## 20. 维护中断

`maintenance_interrupt_state()`（`:434-468`）检查以下条件生成维护中断：

| 条件 | ICH_HCR 位 | 含义 |
|------|-----------|------|
| 空 LR 列表 | `LRENPIE` | 所有 LR 无效（需要补充） |
| 无 Pending | `NPIE` | 无虚拟中断挂起 |
| Underflow | `UIE` | 少于2个有效 LR |
| EOI count | `EOIcount` | 有虚拟 EOI 需 Hypervisor 处理 |
| Group 使能变化 | `VGrp0EIE/VGrp1EIE` | Guest 修改虚拟组使能 |

---

# 第六部分：ITS（Interrupt Translation Service）

## 21. ITS 架构与 QOM 层次

ITS 实现在 `arm_gicv3_its.c`（2,037 行），负责将 MSI/MSI-X 的 DeviceID + EventID 翻译为目标 CPU 和 LPI ID。

```
SysBusDevice
  └── TYPE_ARM_GICV3_ITS_COMMON          [arm_gicv3_its_common.c:147-154]
        ├── TYPE_ARM_GICV3_ITS ("arm-gicv3-its")   [SW 实现]
        │     arm_gicv3_its.c:2024-2030
        └── TYPE_KVM_ARM_ITS ("arm-its-kvm")        [KVM 实现]
              arm_gicv3_its_kvm.c:258-264
```

## 22. ITS 表结构

ITS 使用三级表结构存储在客户机内存中（`arm_gicv3_its.c:1415-1521`）：

### 22.1 Device Table（设备表）

```
GITS_BASER[0] 配置:
  ┌──────────────────────────────────┐
  │ Device Table Entry (DTE)        │
  │  Valid 位                        │
  │  ITT 基地址 → 指向该设备的 ITT   │
  │  ITT 大小（EventID 位数）        │
  └──────────────────────────────────┘
  索引: DeviceID
```

### 22.2 Interrupt Translation Table（ITT）

```
每个设备一张 ITT:
  ┌──────────────────────────────────┐
  │ ITE (Interrupt Translation Entry)│
  │  Valid 位                        │
  │  pINTID — 物理 LPI 号            │
  │  ICID — Collection ID            │
  └──────────────────────────────────┘
  索引: EventID
```

### 22.3 Collection Table（集合表）

```
GITS_BASER[1] 配置:
  ┌──────────────────────────────────┐
  │ Collection Table Entry (CTE)    │
  │  Valid 位                        │
  │  RDbase — 目标 Redistributor     │
  └──────────────────────────────────┘
  索引: ICID (Collection ID)
```

### 22.4 翻译查找链

```
DeviceID → Device Table → DTE → ITT 基址
EventID  → ITT[EventID] → ITE → {pINTID, ICID}
ICID     → Collection Table → CTE → 目标 Redistributor
```

## 23. ITS 命令队列

### 23.1 命令队列机制

```
GITS_CBASER — 命令队列基地址和大小
GITS_CWRITER — 软件写指针（Guest 推进）
GITS_CREADR — 硬件读指针（ITS 推进）

Guest 写入命令到队列 → 更新 CWRITER
  → ITS 检测 CWRITER != CREADR
  → 处理命令 → 推进 CREADR
```

命令处理循环（`arm_gicv3_its.c:1246-1407`）。

### 23.2 ITS 命令总表

| 命令 | 位置 | 功能 |
|------|------|------|
| `MAPD` | `:822-848` | 映射 DeviceID → ITT（创建设备表条目） |
| `MAPC` | `:761-787` | 映射 ICID → Redistributor（创建集合条目） |
| `MAPTI` | `:579-647` | 映射 EventID → pINTID + ICID（创建 ITE） |
| `MAPI` | `:579-647` | 同 MAPTI（pINTID = EventID） |
| `INT` | `:556-577` | 触发中断（DeviceID + EventID） |
| `CLEAR` | `:508-553` | 清除中断挂起状态 |
| `DISCARD` | `:508-553` | 丢弃 ITE |
| `INV` | `:1186-1240` | 使指定 ITE 的缓存无效 |
| `INVALL` | `:1352-1365` | 使指定 Collection 的所有缓存无效 |
| `SYNC` | `:1314-1322` | 确保所有操作对指定 Redistributor 可见 |
| `MOVALL` | `:850-883` | 将所有中断从一个 Redistributor 移到另一个 |
| `MOVI` | `:885-930` | 移动单个中断到另一个 Collection |

## 24. LPI 翻译流程

MSI 写入 `GITS_TRANSLATER`（`arm_gicv3_its.c:1540-1576`）：

```
PCIe 设备写入 GITS_TRANSLATER:
  attrs.requester_id = DeviceID (PCI BDF)      [:1552-1566]
  data = EventID

  → ITS 翻译:
    1. DeviceID → DTE 查找                     [:350-401]
       → 获取 ITT 基址
    2. EventID → ITE 查找                      [:283-315]
       → 获取 {pINTID, ICID}
    3. ICID → CTE 查找                        [:405-430]
       → 获取目标 Redistributor
    4. 向目标 Redistributor 发送 LPI            [:465-477]
       → gicv3_redist_process_lpi()
       → 设置 Pending 表位
       → 触发 CPU Interface 更新
```

## 25. MSI/MSI-X 与 ITS 集成

### 25.1 PCI MSI 路由

PCI 设备的 MSI 通过以下路径到达 ITS：

```
PCI 设备触发 MSI
  → pci_msi_trigger()                          [pci.c:2379-2382]
  → 写入 MSI 地址（= GITS_TRANSLATER GPA）
     数据 = EventID
  → ITS MMIO 写处理
  → 翻译并投递 LPI
```

### 25.2 DeviceID 派生

PCI 的 DeviceID 从 BDF（Bus:Device:Function）派生（`pci.c:1249-1298`）：

```
DeviceID = (bus << 8) | (device << 3) | function
```

通过 `MemTxAttrs.requester_id` 字段传递到 ITS。

---

# 第七部分：KVM vGIC 集成

## 26. KVM GICv3 后端架构

`arm_gicv3_kvm.c`（979 行）实现 KVM 内核态 vGIC 后端。

### 26.1 类初始化

`kvm_arm_gicv3_class_init()`（`arm_gicv3_kvm.c:786-880`）：

```c
// 覆盖基类方法
agcc->pre_save = kvm_arm_gicv3_get;    // 迁移前：从内核获取状态
agcc->post_load = kvm_arm_gicv3_put;   // 迁移后：向内核写入状态
// 覆盖 realize
dc->realize = kvm_arm_gicv3_realize;
// 覆盖 set_irq
kgc->set_irq = kvm_arm_gicv3_set_irq;
```

### 26.2 创建内核 vGIC

`kvm_arm_gicv3_realize()`（`arm_gicv3_kvm.c:786-798`）：

```
1. 调用 KVM_CREATE_DEVICE 创建 KVM_DEV_TYPE_ARM_VGIC_V3
2. 设置 Distributor 基地址（KVM_DEV_ARM_VGIC_GRP_ADDR）
3. 设置 Redistributor 基地址
4. 调用 KVM_DEV_ARM_VGIC_GRP_CTRL 初始化
```

### 26.3 KVM 限制

KVM 后端不支持以下特性（`arm_gicv3_kvm.c:800-820`）：
- Security 扩展
- NMI 支持
- 非零 first-cpu-idx
- GICv4（直接虚拟 LPI 注入）

### 26.4 中断注入

`kvm_arm_gicv3_set_irq()`（`arm_gicv3_kvm.c:77-82`）：

```c
// 直接通过 KVM_IRQ_LINE ioctl 注入中断到内核 vGIC
// 无需经过 QEMU 侧的状态机
```

## 27. 状态同步机制

### 27.1 KVM 设备组

通过 `KVM_GET/SET_DEVICE_ATTR` ioctl 访问内核 vGIC 状态（`arm_gicv3_kvm.c:87-119`）：

| 设备组 | 用途 |
|--------|------|
| `KVM_DEV_ARM_VGIC_GRP_DIST_REGS` | Distributor 寄存器 |
| `KVM_DEV_ARM_VGIC_GRP_REDIST_REGS` | Redistributor 寄存器 |
| `KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS` | ICC_* 系统寄存器 |
| `KVM_DEV_ARM_VGIC_GRP_LEVEL_INFO` | 中断电平信息 |

### 27.2 同步函数

```
kvm_arm_gicv3_get()  — 从内核读取全部 vGIC 状态到 QEMU
  ├── 读取 GICD_* 寄存器
  ├── 读取每 CPU 的 GICR_* 寄存器
  ├── 读取每 CPU 的 ICC_* 系统寄存器
  └── 读取中断电平信息

kvm_arm_gicv3_put()  — 将 QEMU 状态写入内核 vGIC
  ├── 写入 GICD_* 寄存器
  ├── 写入每 CPU 的 GICR_* 寄存器
  ├── 写入每 CPU 的 ICC_* 系统寄存器
  └── 写入中断电平信息
```

## 28. KVM ITS

`arm_gicv3_its_kvm.c`（271 行）：

### 28.1 初始化

```c
// KVM ITS 创建                              [:92-108]
KVM_CREATE_DEVICE → KVM_DEV_TYPE_ARM_VGIC_ITS
设置 ITS 基地址
```

### 28.2 MSI 路由

```c
// KVM ITS 使用 KVM_SIGNAL_MSI            [:45-67]
// 直接通过内核发送 MSI 到 ITS
// QEMU 不参与翻译，全部在内核完成
```

### 28.3 表保存/恢复

`arm_gicv3_its_kvm.c:131-198`：

```
迁移时：
  pre_save:
    KVM_DEV_ARM_VGIC_GRP_ITS_REGS → 读取 ITS 寄存器
    KVM_DEV_ARM_VGIC_SAVE_ITS_TABLES → 内核将 ITS 表刷到客户机内存
  post_load:
    KVM_DEV_ARM_VGIC_GRP_ITS_REGS → 写入 ITS 寄存器
    KVM_DEV_ARM_VGIC_RESTORE_ITS_TABLES → 内核从客户机内存恢复 ITS 表
```

## 29. 迁移（VMState）

### 29.1 GICv3 VMState

`arm_gicv3_common.c` 定义 VMState 描述符：

```c
// vmstate_arm_gicv3 — 顶层
//   → vmstate_arm_gicv3_cpu — 每 CPU 状态
//     → Redistributor 寄存器
//     → ICC 寄存器
//     → 虚拟接口寄存器（ICH/ICV）
//   → Distributor 位图和寄存器
```

### 29.2 保存的状态内容

| 组件 | 保存内容 |
|------|----------|
| Distributor | CTLR, 所有中断位图, IROUTER, 优先级 |
| Redistributor | CTLR, WAKER, PROPBASER, PENDBASER, SGI/PPI 位图 |
| CPU Interface | PMR, BPR, APR, CTLR, IGRPEN, SRE |
| 虚拟接口 | ICH_HCR, ICH_VMCR, ICH_LR[0-15], ICH_APR |
| ITS | CTLR, CBASER, CWRITER, CREADR, BASER[0-7] |

### 29.3 ITS 迁移

ITS 迁移钩子（`arm_gicv3_its_common.c:29-66`）：
- `pre_save`：刷新 ITS 内部缓存到客户机内存表
- `post_load`：从客户机内存表重建 ITS 内部状态

---

# 第八部分：中断完整流程

## 30. SPI 端到端流程

```
外设触发中断（如 virtio-blk 完成请求）
  │
  ▼
qemu_set_irq(gic_spi[N], 1)
  │
  ▼
gicv3_set_irq()                             [arm_gicv3.c:372-399]
  ├── 判断: SPI（irq >= 32）
  └── gicv3_dist_set_irq()                  [arm_gicv3_dist.c:941-959]
      ├── 更新电平/边沿状态
      ├── 设置 pending 位
      └── gicv3_update()                    [arm_gicv3.c:325-333]
          │
          ▼
      gicv3_update_noirqset()              [arm_gicv3.c:257-323]
          ├── 遍历所有 CPU，检查 IROUTER 目标
          ├── 比较优先级，更新各 CPU 的 HPPI
          └── gicv3_cpuif_update(cs)        [cpuif.c:1046-1102]
              ├── 检查组使能 + PMR + 抢占条件
              └── qemu_irq_raise(cs->parent_fiq/irq)
                  │
                  ▼
              CPU 看到 IRQ/FIQ 信号
                  │
                  ▼
              异常进入 → 中断处理程序
                  │
                  ▼
              读 ICC_IAR1_EL1                [cpuif.c:1285-1310]
                  ├── icc_activate_irq()     → pending→active
                  └── 返回中断 ID
                  │
                  ▼
              处理中断，写 ICC_EOIR1_EL1     [cpuif.c:1645-1714]
                  ├── icc_drop_prio()        → 降低运行优先级
                  └── icc_deactivate_irq()   → 清 active 位
                  │
                  ▼
              gicv3_cpuif_update()           → 检查下一个挂起中断
```

## 31. LPI/MSI 端到端流程

```
PCIe 设备触发 MSI-X
  │
  ▼
写 GITS_TRANSLATER MMIO                     [its.c:1540-1576]
  │  DeviceID = PCI BDF (requester_id)
  │  EventID = MSI data
  │
  ▼
ITS 翻译:
  ├── DeviceID → DTE → ITT 基址              [its.c:350-401]
  ├── EventID → ITE → {pINTID, ICID}         [its.c:283-315]
  └── ICID → CTE → 目标 Redistributor        [its.c:405-430]
      │
      ▼
gicv3_redist_process_lpi()                   [redist.c:897-911]
  ├── 在 PENDBASER 表中设置 pending 位
  ├── 从 PROPBASER 表读取优先级和使能
  └── 更新 HPPI
      │
      ▼
gicv3_cpuif_update(cs)                       [cpuif.c:1046-1102]
  └── qemu_irq_raise(cs->parent_irq)
      │
      ▼
CPU 中断处理（IAR → EOI 流程同 SPI）
```

## 32. 状态重计算与级联更新

### 32.1 更新级联关系

```
状态变更触发源                    更新路径
─────────────────────────────────────────────────
GICD 寄存器写入                 → gicv3_update()
  (使能/挂起/优先级/路由等)        → gicv3_cpuif_update() per CPU

GICR 寄存器写入                 → gicv3_redist_update()
  (使能/挂起/优先级/配置等)        → gicv3_cpuif_update()

ICC 寄存器写入                  → gicv3_cpuif_update()
  (PMR/BPR/IGRPEN/CTLR等)        直接重计算

ICC_IAR 读取（激活中断）         → gicv3_update() 或 gicv3_redist_update()
                                 → gicv3_cpuif_update()

ICC_EOIR 写入（结束中断）        → gicv3_cpuif_update()
                                 → gicv3_update() 或 gicv3_redist_update()

外部中断电平变化                → gicv3_dist_set_irq() 或 gicv3_redist_set_irq()
                                 → gicv3_update() → gicv3_cpuif_update()

ITS 翻译触发 LPI                → gicv3_redist_process_lpi()
                                 → gicv3_cpuif_update()
```

### 32.2 全量重计算

`gicv3_full_update_noirqset()`（`arm_gicv3.c:335-357`）：

```c
// 在 GIC 复位、迁移恢复后调用
// 重新扫描所有中断状态，计算每个 CPU 的 HPPI
// 更新所有 CPU 的 IRQ/FIQ 输出线
```

---

# 附录

## A. 关键源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `hw/intc/arm_gicv3_cpuif.c` | 3,192 | CPU Interface（ICC/ICV/ICH 寄存器） |
| `hw/intc/arm_gicv3_its.c` | 2,037 | ITS 命令处理和翻译 |
| `hw/intc/arm_gicv3_redist.c` | 1,187 | Redistributor（GICR 寄存器） |
| `hw/intc/arm_gicv3_kvm.c` | 979 | KVM vGIC 后端 |
| `hw/intc/arm_gicv3_dist.c` | 960 | Distributor（GICD 寄存器） |
| `hw/intc/arm_gicv3_hvf.c` | 835 | macOS HVF 后端 |
| `hw/intc/arm_gicv3_common.c` | 672 | 公共基类（VMState/属性/复位） |
| `hw/intc/arm_gicv3.c` | 478 | GICv3 主设备（中断路由/更新） |
| `hw/intc/arm_gicv3_its_kvm.c` | 271 | KVM ITS 后端 |
| `hw/intc/arm_gicv3_its_common.c` | 171 | ITS 公共基类 |
| `hw/intc/gicv3_internal.h` | ~800 | 内部宏和偏移定义 |
| `include/hw/intc/arm_gicv3_common.h` | 345 | 状态结构体定义 |
| `include/hw/intc/arm_gicv3_its_common.h` | 135 | ITS 状态结构体 |
| `include/hw/intc/arm_gicv3.h` | 32 | GICv3 类型声明 |

## B. GICD/GICR/ICC 寄存器速查表

### GICD 寄存器

| 偏移 | 寄存器 | 访问 | 描述 |
|------|--------|------|------|
| 0x0000 | GICD_CTLR | RW | 全局控制 |
| 0x0004 | GICD_TYPER | RO | 类型信息 |
| 0x0008 | GICD_IIDR | RO | 实现 ID |
| 0x0080+ | GICD_IGROUPR | RW | 分组 |
| 0x0100+ | GICD_ISENABLER | WS | 使能置位 |
| 0x0180+ | GICD_ICENABLER | WC | 使能清除 |
| 0x0200+ | GICD_ISPENDR | WS | 挂起置位 |
| 0x0280+ | GICD_ICPENDR | WC | 挂起清除 |
| 0x0300+ | GICD_ISACTIVER | WS | 激活置位 |
| 0x0380+ | GICD_ICACTIVER | WC | 激活清除 |
| 0x0400+ | GICD_IPRIORITYR | RW | 优先级 |
| 0x0C00+ | GICD_ICFGR | RW | 触发配置 |
| 0x6000+ | GICD_IROUTER | RW | 亲和性路由 |

### ICC 系统寄存器

| 寄存器 | Op0 | Op1 | CRn | CRm | Op2 | 描述 |
|--------|-----|-----|-----|-----|-----|------|
| ICC_IAR0_EL1 | 3 | 0 | 12 | 8 | 0 | G0 确认 |
| ICC_IAR1_EL1 | 3 | 0 | 12 | 12 | 0 | G1 确认 |
| ICC_EOIR0_EL1 | 3 | 0 | 12 | 8 | 1 | G0 结束 |
| ICC_EOIR1_EL1 | 3 | 0 | 12 | 12 | 1 | G1 结束 |
| ICC_PMR_EL1 | 3 | 0 | 4 | 6 | 0 | 优先级掩码 |
| ICC_BPR0_EL1 | 3 | 0 | 12 | 8 | 3 | G0 二进制点 |
| ICC_BPR1_EL1 | 3 | 0 | 12 | 12 | 3 | G1 二进制点 |
| ICC_CTLR_EL1 | 3 | 0 | 12 | 12 | 4 | 控制 |
| ICC_SGI1R_EL1 | 3 | 0 | 12 | 11 | 5 | G1 SGI 生成 |
| ICC_IGRPEN0_EL1 | 3 | 0 | 12 | 12 | 6 | G0 使能 |
| ICC_IGRPEN1_EL1 | 3 | 0 | 12 | 12 | 7 | G1 使能 |

## C. ITS 命令速查表

| 命令 | DW0[7:0] | 参数 | 功能 |
|------|----------|------|------|
| MAPD | 0x08 | DeviceID, ITT_addr, Size | 映射设备 → ITT |
| MAPC | 0x09 | ICID, RDbase, Valid | 映射集合 → Redistributor |
| MAPTI | 0x0A | DeviceID, EventID, pINTID, ICID | 映射事件 → LPI + 集合 |
| MAPI | 0x0B | DeviceID, EventID, ICID | 同 MAPTI（pINTID=EventID） |
| INT | 0x03 | DeviceID, EventID | 触发中断 |
| CLEAR | 0x04 | DeviceID, EventID | 清除挂起 |
| DISCARD | 0x0F | DeviceID, EventID | 丢弃映射 |
| INV | 0x0C | DeviceID, EventID | 使缓存无效 |
| INVALL | 0x0D | ICID | 使集合缓存全无效 |
| SYNC | 0x05 | RDbase | 同步到 Redistributor |
| MOVALL | 0x0E | RDbase_src, RDbase_dst | 移动所有中断 |
| MOVI | 0x01 | DeviceID, EventID, ICID | 移动单个中断 |

---

> **相关文档**：
> - `darren/arm64/00-ARM64-CPU-GICv3-TCG深度分析.md` — ARM64 CPU 模型概览（GICv3 简介）
> - `darren/device-model/00-设备模型与virtio深度分析.md` — 设备模型框架（qemu_irq 连接）
> - `darren/memory/00-内存子系统深度分析.md` — MemoryRegion / MMIO 分发
> - `darren/accel/00-TCG深度分析.md` — TCG 执行循环（中断检查点）
> - ARM IHI 0069 — GICv3/v4 Architecture Specification
