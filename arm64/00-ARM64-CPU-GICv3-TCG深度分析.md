# ARM64 CPU 模型、GICv3 与 TCG 翻译深度分析

> **版本**: QEMU 11.0.50  
> **分析日期**: 2026-05-08  
> **重点架构**: AArch64 (ARM64)  
> **核心源码**: `target/arm/cpu.c`、`target/arm/cpu.h`、`hw/intc/arm_gicv3*.c`、`target/arm/tcg/translate-a64.c`  

---

## 目录

**第一部分：ARM64 CPU 模型**
1. [CPUState 基类](#1-cpustate-基类)
2. [ARMCPU 结构体](#2-armcpu-结构体)
3. [CPUARMState — 架构状态](#3-cpuarmstate--架构状态)
4. [CPU 类型注册与层级](#4-cpu-类型注册与层级)
5. [CPU 初始化流程](#5-cpu-初始化流程)
6. [系统寄存器（cpregs）](#6-系统寄存器cpregs)
7. [异常处理](#7-异常处理)

**第二部分：GICv3 中断控制器**
8. [GICv3 文件结构](#8-gicv3-文件结构)
9. [GICv3 状态结构体](#9-gicv3-状态结构体)
10. [GICv3 QOM 类型层级](#10-gicv3-qom-类型层级)
11. [中断分发逻辑](#11-中断分发逻辑)
12. [GICv3 MMIO 寄存器](#12-gicv3-mmio-寄存器)
13. [ITS — 中断翻译服务](#13-its--中断翻译服务)
14. [virt 机器 GIC 集成](#14-virt-机器-gic-集成)

**第三部分：TCG 翻译引擎**
15. [TCG 概述](#15-tcg-概述)
16. [翻译流程](#16-翻译流程)
17. [TranslationBlock](#17-translationblock)
18. [TCG IR 与寄存器映射](#18-tcg-ir-与寄存器映射)
19. [MMU/TLB 与页表遍历](#19-mmutlb-与页表遍历)
20. [中断与异常在 TCG 中的处理](#20-中断与异常在-tcg-中的处理)

---

# 第一部分：ARM64 CPU 模型

## 1. CPUState 基类

`include/hw/core/cpu.h:101-197` 定义了所有架构共享的 CPU 基类。

### 1.1 CPUClass — 类虚方法

| 方法 | 说明 |
|------|------|
| `reset_hold()` | CPU 复位 |
| `has_work()` | 是否有待处理的工作（中断等） |
| `dump_state()` | 导出 CPU 状态 |
| `set_pc()` / `get_pc()` | 设置/获取程序计数器 |
| `memory_rw_debug()` | 调试器内存读写 |
| `get_phys_page_debug()` | 调试器地址翻译 |
| `gdb_read_register()` / `gdb_write_register()` | GDB 寄存器访问 |
| `tcg_ops` | TCG 操作集（翻译、中断等） |
| `sysemu_ops` | 系统模拟操作集 |

### 1.2 CPUState — 关键实例字段

| 字段 | 说明 |
|------|------|
| `nr_threads` / `thread_id` | 线程信息 |
| `interrupt_request` | 待处理中断位掩码 |
| `halted` | 是否处于 halt 状态 |
| `stop` / `stopped` | 停止请求/已停止 |
| `exception_index` | 当前异常编号 |
| `crash_occurred` | 是否已崩溃 |
| `cpu_ases` / `as` | 关联的 AddressSpace |
| `mem_io_pc` | 最近 I/O 操作的 PC |
| `env_ptr` | 指向架构特定状态 |

---

## 2. ARMCPU 结构体

`target/arm/cpu.h:921-1186` — ARM CPU 实例（继承 CPUState）：

```
ARMCPU (ArchCPU)
  ├── parent_obj (CPUState)                    [921]
  ├── env (CPUARMState)                        [925]  ← 核心架构状态
  │
  ├── cpreg_* (系统寄存器表/迁移)              [926-947]
  ├── GDB 动态特性                             [948-954]
  ├── gt_timer[] (通用定时器)                  [955-970]
  ├── MMIO 区域                                [972-977]
  ├── power_state                              [991-992]
  │
  ├── 特性标志:                                [994-1012]
  │    has_el2, has_el3, has_pmu,
  │    has_vfp, has_vfp_d32, has_neon,
  │    has_dsp, has_mpu, kvm_mte
  │
  ├── KVM 特有字段                             [1030-1047]
  ├── SVE/SME:                                 [1150-1151]
  │    sve_max_vq, sme_max_vq
  └── gt_cntfrq_hz (定时器频率)                [1163]
```

---

## 3. CPUARMState — 架构状态

`target/arm/cpu.h:259-820` — 这是 ARM CPU 最核心的运行时状态：

### 3.1 寄存器与 PC

| 字段组 | 位置 | 说明 |
|--------|------|------|
| 核心寄存器 / AArch64 寄存器 | 260-316 | X0-X30、SP、PC、PSTATE |

### 3.2 CP15 / 系统寄存器

| 字段组 | 位置 | 说明 |
|--------|------|------|
| cp15 系统寄存器 | 320-597 | SCTLR、TCR、TTBR、MAIR、VBAR 等 |
| v7M 状态 | 599-642 | Cortex-M 专用 |

### 3.3 异常与中断

| 字段组 | 位置 | 说明 |
|--------|------|------|
| 异常信息 | 644-655 | `exception.syndrome`、`exception.target_el` 等 |
| SError 状态 | 657-662 | 异步错误 |
| IRQ 线状态 | 666-754 | VFP、PAC 密钥、SME ZA |

---

## 4. CPU 类型注册与层级

### 4.1 QOM 继承链

```
Object → DeviceState → CPUState → ARMCPU
```

### 4.2 基础类型注册

```c
// target/arm/cpu.c:2573-2583
static const TypeInfo arm_cpu_type_info = {
    .name = TYPE_ARM_CPU,
    .parent = TYPE_CPU,
    .instance_size = sizeof(ARMCPU),
    .instance_init = arm_cpu_initfn,
    .class_size = sizeof(ARMCPUClass),
    .class_init = arm_cpu_class_init,
    .abstract = true,              // 抽象基类
};

// cpu.c:2585-2590
type_init(arm_cpu_register_types)
```

### 4.3 arm_cpu_class_init

`target/arm/cpu.c:2505-2538` 设置类虚方法：

```
arm_cpu_class_init(klass)
  ├── cc->dump_state = arm_cpu_dump_state
  ├── cc->set_pc = arm_cpu_set_pc
  ├── cc->get_pc = arm_cpu_get_pc
  ├── cc->gdb_* = GDB 寄存器读写
  ├── cc->sysemu_ops = arm 系统模拟操作
  └── cc->tcg_ops = arm TCG 操作（翻译、中断等）
```

### 4.4 具体 CPU 模型

AArch64 CPU 模型定义在多个文件中：

| 文件 | 模型 | 初始化函数 |
|------|------|-----------|
| `target/arm/cpu64.c` | cortex-a57 | `aarch64_a57_initfn()` :689 |
| `target/arm/cpu64.c` | cortex-a53 | `aarch64_a53_initfn()` :751 |
| `target/arm/tcg/cpu64.c` | cortex-a72 | `aarch64_a72_initfn()` |
| `target/arm/tcg/cpu64.c` | cortex-a35 | `aarch64_a35_initfn()` :32 |
| `target/arm/cpu64.c` | 模型表 | `aarch64_cpus[]` :896-903 |

每个 CPU 模型的 `initfn` 设置：
- ID 寄存器值（ID_AA64PFR0、ID_AA64MMFR0 等）
- 特性标志（has_el2、has_vfp 等）
- 缓存参数
- 系统寄存器定义

**示例: cortex-a57** (`cpu64.c:689-749`):
```
aarch64_a57_initfn(obj)
  ├── 设置 ID_AA64PFR0/1、ID_AA64DFR0/1
  ├── 设置 ID_AA64ISAR0/1
  ├── 设置 ID_AA64MMFR0/1
  ├── 设置缓存类型寄存器 (CTR_EL0, CCSIDR 等)
  ├── cpu->midr = 0x411fd070  (Cortex-A57 MIDR)
  ├── define_cortex_a72_a57_a53_cp_reginfo(cpu)  [cortex-regs.c:73-76]
  └── 设置 resetvalue 等
```

> **佐证 commit**: `970ea8478c` "Allow 'aarch64=off' to be set for TCG CPUs" —
> 允许 TCG CPU 模型禁用 AArch64 模式。

---

## 5. CPU 初始化流程

### 5.1 完整生命周期

```
object_new(TYPE_ARM_CPU_xxx)
  └─ arm_cpu_initfn()              [cpu.c:1172]
       ├── 初始化定时器
       ├── 初始化 cpreg 哈希表
       └── 注册 GPIO 中断线

qdev_realize()
  └─ arm_cpu_realizefn()           [cpu.c:1740]
       ├── arm_cpu_finalize_features()  [cpu.c:1680]
       │    ├── arm_cpu_sve_finalize()     # SVE 向量长度 [63-260]
       │    ├── arm_cpu_sme_finalize()     # SME 矩阵长度 [339-396]
       │    ├── arm_cpu_pauth_finalize()   # 指针认证
       │    └── arm_cpu_lpa2_finalize()    # LPA2 大页 [cpu64.c:669-687]
       ├── 注册系统寄存器 (define_arm_cp_regs)
       ├── 初始化 AddressSpace
       ├── 注册 GDB 寄存器
       └── cpu_reset(cpu)
            └── arm_cpu_reset_hold()  [cpu.c:306]
```

### 5.2 arm_cpu_reset_hold

`target/arm/cpu.c:306-457` — 设置 AArch64 复位状态：

```
arm_cpu_reset_hold(obj)
  ├── 清零 CPUARMState
  ├── 设置初始 PSTATE（EL1h 或 EL3h）
  ├── 设置 SCTLR 初始值
  ├── 初始化 SVE/SME/MTE/PAuth/MOPS/GCS/TBI 等特性
  ├── 根据 has_el2/has_el3 设置 HCR_EL2/SCR_EL3
  ├── 设置 CPTR_EL2/EL3
  └── 重置定时器/PMU 状态
```

---

## 6. 系统寄存器（cpregs）

### 6.1 ARMCPRegInfo 结构

`target/arm/cpregs.h:921-1049`:

```c
typedef struct ARMCPRegInfo {
    const char *name;         // 寄存器名（如 "SCTLR_EL1"）

    // 编码（用于匹配 MRS/MSR 指令）
    uint8_t cp;               // 协处理器号（AArch64 固定为 0）
    uint8_t crn;              // CRn 字段
    uint8_t crm;              // CRm 字段
    uint8_t opc0;             // op0
    uint8_t opc1;             // op1
    uint8_t opc2;             // op2

    int state;                // ARM_CP_STATE_AA64 / AA32 / BOTH
    int type;                 // ARM_CP_* 类型标志
    int access;               // 访问权限（PL0/PL1/PL2/PL3 读/写）
    int secure;               // 安全属性

    ARMCPFGTInfo fgt;         // Fine-Grained Trap 信息
    uint64_t resetvalue;      // 复位值
    ptrdiff_t fieldoffset;    // 在 CPUARMState 中的偏移

    // 回调
    CPReadFn *readfn;         // 自定义读回调
    CPWriteFn *writefn;       // 自定义写回调
    CPAccessFn *accessfn;     // 访问检查回调
    CPResetFn *resetfn;       // 自定义复位回调
} ARMCPRegInfo;
```

### 6.2 注册 API

| 函数 | 位置 | 说明 |
|------|------|------|
| `define_one_arm_cp_reg()` | cpregs.h:1051 | 注册单个系统寄存器 |
| `define_arm_cp_regs()` | cpregs.h:1054-1058 | 批量注册（宏，遍历数组） |
| `get_arm_cp_reginfo()` | cpregs.h:1060 | 按编码查找已注册的寄存器 |

### 6.3 Fine-Grained Trap (FGT)

`cpregs.h:415-479` 定义了 HFG{R,W}TR_EL2 的 trap 位，
用于 EL2 对 EL1 系统寄存器访问的细粒度陷入控制。

**示例寄存器陷入**:
- `SCTLR_EL1`: `HFGRTR_EL2` 和 `HFGWTR_EL2` 中有对应的 trap 位
- `TCR_EL1`、`TTBR0_EL1`: 同上

---

## 7. 异常处理

### 7.1 异常触发

```
arm_cpu_do_interrupt()                  [helper.c]
  ├── arm_cpu_do_interrupt_aarch64()    # AArch64 异常入口
  ├── arm_cpu_do_interrupt_aarch32()    # AArch32 异常入口
  └── arm_cpu_do_interrupt_aarch32_hyp() # Hyp 模式
```

### 7.2 异常路由

`arm_phys_excp_target_el()` 根据异常类型和当前 EL 决定目标 EL：

| 异常源 | 可能的目标 EL |
|--------|-------------|
| Guest 应用 (EL0) | EL1（内核）/ EL2（Hypervisor） |
| Guest 内核 (EL1) | EL1 / EL2 / EL3 |
| Hypervisor (EL2) | EL2 / EL3 |

### 7.3 异常信息

```c
// cpu.h:644-655
CPUARMState.exception = {
    uint32_t syndrome;      // ESR_ELx 内容
    uint32_t fsr;           // Fault Status Register
    uint64_t vaddress;      // 故障虚拟地址
    uint32_t target_el;     // 目标异常级别
    ...
};
```

---

# 第二部分：GICv3 中断控制器

## 8. GICv3 文件结构

```
hw/intc/
  ├── arm_gicv3.c              # GICv3 核心逻辑（中断分发）
  ├── arm_gicv3_common.c       # GICv3 通用框架（QOM、迁移）
  ├── arm_gicv3_dist.c         # Distributor MMIO 寄存器实现
  ├── arm_gicv3_redist.c       # Redistributor MMIO 寄存器实现
  ├── arm_gicv3_cpuif.c        # CPU Interface 逻辑
  ├── arm_gicv3_cpuif_common.c # CPU Interface 通用部分
  ├── arm_gicv3_its.c          # ITS（中断翻译服务）
  ├── arm_gicv3_its_common.c   # ITS 通用框架
  ├── arm_gicv3_kvm.c          # KVM vGIC 后端
  ├── arm_gicv3_its_kvm.c      # KVM ITS 后端
  ├── arm_gicv3_hvf.c          # HVF vGIC 后端
  ├── arm_gicv3_whpx.c         # WHPX vGIC 后端
  └── gicv3_internal.h         # 内部头文件

include/hw/intc/
  ├── arm_gicv3_common.h       # GICv3State、GICv3CPUState 定义
  ├── arm_gicv3.h              # GICv3 公共接口
  └── arm_gicv3_its_common.h   # ITS 通用定义
```

---

## 9. GICv3 状态结构体

### 9.1 GICv3State（Distributor 全局状态）

`include/hw/intc/arm_gicv3_common.h:225-281`:

```
GICv3State
  ├── parent_obj (SysBusDevice)
  ├── num_cpu                        [236]  CPU 数量
  ├── num_irq                        [237]  中断数量
  ├── revision                       [238]  GIC 版本
  │
  ├── Distributor 状态:
  │    gicd_ctlr                     [258]  GICD 控制寄存器
  │    gicd_statusr[2]               [259]  状态寄存器(NS/S)
  │    gicd_group[]                  [260]  中断分组位图
  │    gicd_grpmod[]                 [261]  分组修改位图
  │    gicd_enabled[]                [262]  使能位图
  │    gicd_pending[]                [263]  挂起位图
  │    gicd_active[]                 [264]  活跃位图
  │    gicd_level[]                  [265]  电平状态
  │    gicd_edge_trigger[]           [266]  边沿/电平触发
  │    gicd_nmi[]                    [267]  NMI 位图
  │    gicd_ipriority[]              [268]  优先级
  │    gicd_irouter[]                [269]  路由目标
  │    gicd_irouter_target[]         [273]  路由目标 CPU 缓存
  │    gicd_nsacr[]                  [274]  非安全访问控制
  │
  └── cpu[] (GICv3CPUState*)         # 每 CPU 状态数组
```

### 9.2 GICv3CPUState（Per-CPU Redistributor + CPU Interface）

`include/hw/intc/arm_gicv3_common.h:127-213`:

```
GICv3CPUState
  ├── Redistributor 寄存器:
  │    gicr_ctlr                     [140]
  │    gicr_typer                    [141]
  │    gicr_statusr[2]              [142]
  │    gicr_waker                    [143]
  │    gicr_propbaser                [144]  LPI 配置表基址
  │    gicr_pendbaser                [145]  LPI 挂起表基址
  │    gicr_igroupr0                 [147]  PPI/SGI 分组
  │    gicr_ienabler0                [148]  PPI/SGI 使能
  │    gicr_ipendr0                  [149]  PPI/SGI 挂起
  │    gicr_iactiver0                [150]  PPI/SGI 活跃
  │    gicr_inmir0                   [151]  PPI/SGI NMI
  │    gicr_ipriorityr[]             [155]  PPI/SGI 优先级
  │
  ├── CPU Interface 寄存器:
  │    icc_ctlr_el1[2]              [161]  ICC 控制(NS/S)
  │    icc_ctlr_el3                  [162]
  │    icc_pmr_el1                   [163]  优先级掩码
  │    icc_bpr[3]                    [164]  二进制点
  │    icc_apr[3][4]                 [165]  活跃优先级
  │    icc_igrpen[3]                 [166]  中断组使能
  │    icc_sre_el1                   [169]  系统寄存器使能
  │
  ├── 虚拟化 (vGIC):
  │    ich_apr[3][4]                 [173]
  │    ich_hcr_el2                   [174]
  │    ich_lr_el2[16]                [175]  List Registers
  │    ich_vmcr_el2                  [176]
  │
  └── HPPI 缓存:
       hppi { irq, grp, prio, nmi } [193]  最高优先级挂起中断
       hpplpi                        [199]  最高优先级 LPI
       hppvlpi                       [202]  最高优先级虚拟 LPI
```

---

## 10. GICv3 QOM 类型层级

```
Object → DeviceState → SysBusDevice → GICv3Common → GICv3 (软件模拟)
                                          │
                                          ├── KVMGICv3 (KVM vGIC)
                                          └── HVFGICv3 (HVF vGIC)
```

| 类型 | 文件 | TypeInfo 位置 | 说明 |
|------|------|-------------|------|
| `GICv3Common` | arm_gicv3_common.c | :637-648 | 抽象基类，定义通用状态和迁移 |
| `GICv3` | arm_gicv3.c | :465-471 | 软件模拟实现 |
| `KVMGICv3` | arm_gicv3_kvm.c | :966-972 | KVM 内核 vGIC 后端 |
| `HVFGICv3` | arm_gicv3_hvf.c | — | HVF (macOS) vGIC 后端 |

**KVM 变体特点**：
- 使用相同的 `GICv3State` 结构
- realize/reset 通过 `kvm_device_access()` 同步状态到内核 VGIC
- 中断分发由内核 KVM 模块处理，QEMU 只管理配置

---

## 11. 中断分发逻辑

### 11.1 中断类型

| 类型 | ID 范围 | 说明 |
|------|---------|------|
| **SGI** | 0-15 | 软件生成中断（CPU 间通信） |
| **PPI** | 16-31 | 私有外设中断（每 CPU 独立） |
| **SPI** | 32-1019 | 共享外设中断（设备中断） |
| **LPI** | 8192+ | 消息信号中断（MSI，通过 ITS） |

### 11.2 外部 IRQ 输入

`gicv3_set_irq()` (`arm_gicv3.c:373-400`):

```
gicv3_set_irq(opaque, irq, level)
  ├── SPI (irq < num_irq - GIC_INTERNAL):
  │    └─ 转发到 Distributor (irq + GIC_INTERNAL)
  │         设置/清除 gicd_level[]、gicd_pending[]
  └── PPI (剩余的 irq):
       └─ 计算目标 CPU = irq / GIC_INTERNAL
            设置/清除 gicr_ipendr0 等
```

> **注意**: SGI 不通过 `gicv3_set_irq()` — 它们通过 ICC_SGI 系统寄存器写入触发。

### 11.3 优先级比较

`irqbetter()` (`arm_gicv3.c:24-53`):

```c
// 判断 new 是否比 old 更优先
irqbetter(GICv3CPUState *cs, int new_irq, uint8_t new_prio, bool new_nmi,
                              int old_irq, uint8_t old_prio, bool old_nmi)
{
    // 1. NMI 优先于非 NMI（同优先级时）
    if (new_nmi != old_nmi) return new_nmi;       // [41-43]

    // 2. 数值更小 = 更高优先级
    if (new_prio < old_prio) return true;          // [33-35]
    if (new_prio > old_prio) return false;

    // 3. 同优先级：更小的 INTID 胜出
    return new_irq <= old_irq;                     // [45-52]
}
```

> **佐证 commit**: `d89daa893f` "Implement NMI interrupt priority" — 实现了 NMI 的优先级支持。

### 11.4 中断分发流水线

```
设备触发中断
  │  qemu_set_irq() → gicv3_set_irq()
  ▼
Distributor 检查                        [arm_gicv3.c]
  │  gicd_int_pending()                 [55-99]
  │   ├── 检查 group、enabled、pending、active
  │   └── 确定候选中断
  ▼
Redistributor 更新                      [arm_gicv3.c]
  │  gicr_int_pending()                 [101-139]
  │   ├── 检查 PPI/SGI pending
  │   └── 优先级比较 (irqbetter)
  │  gicv3_redist_update()              [247-251]
  │   └── 更新 HPPI 缓存
  ▼
CPU Interface 更新                      [arm_gicv3_cpuif.c]
  │  gicv3_cpuif_update()
  │   ├── 比较 HPPI 与 PMR（优先级掩码）
  │   ├── 比较 HPPI 与 Running Priority
  │   └── 决定是否信号 CPU
  ▼
ARMCPU GPIO 线
  qemu_set_irq(cpu->irq[ARM_CPU_IRQ])   # 或 FIQ/VIRQ/VFIQ/NMI
  └── CPUState.interrupt_request |= CPU_INTERRUPT_HARD
```

---

## 12. GICv3 MMIO 寄存器

### 12.1 MMIO 区域创建

`gicv3_init_irqs_and_mmio()` (`arm_gicv3_common.c:314-369`):

```
创建 Distributor MMIO:
  memory_region_init_io(&s->iomem_dist, ...)     [350-352]
    → MemoryRegionOps: gicv3_dist_read/write

创建 Redistributor MMIO:
  memory_region_init_io(&region->iomem, ...)      [354-367]
    → MemoryRegionOps: gicv3_redist_read/write
```

> **佐证 commit**: `a051f78714` "declare GICv3 regions as little endian" — 
> 将 GICv3 MMIO 区域声明为小端。
> `31164ebf08` "Specify valid and impl in MemoryRegionOps" — 
> 为 GICv3 的 MemoryRegionOps 添加了 valid/impl 约束。

### 12.2 关键 Distributor 寄存器

`arm_gicv3_dist.c` 实现：

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `GICD_CTLR` | 0x0000 | 控制：EnableGrp0/1S/1NS、ARE、DS |
| `GICD_ISENABLER` | 0x0100+ | 中断使能设置（按位） |
| `GICD_ICENABLER` | 0x0180+ | 中断使能清除 |
| `GICD_ISPENDR` | 0x0200+ | 中断挂起设置 |
| `GICD_ICPENDR` | 0x0280+ | 中断挂起清除 |
| `GICD_IPRIORITYR` | 0x0400+ | 中断优先级 |
| `GICD_IROUTER` | 0x6100+ | 中断路由目标 |

**位图操作辅助函数**:
- `gicd_write_set_bitmap_reg()` (`dist.c:115-137`)
- `gicd_write_clear_bitmap_reg()` (`dist.c:139-161`)
- 读取 pending = `PENDING | (level & ~edge)` (`dist.c:183-191`)

### 12.3 关键 Redistributor 寄存器

`arm_gicv3_redist.c` 实现：

| 寄存器 | 说明 |
|--------|------|
| `GICR_CTLR` | Redistributor 控制 |
| `GICR_WAKER` | 唤醒控制（ChildrenAsleep 位） |
| `GICR_ISENABLER0` | PPI/SGI 使能设置 |
| `GICR_ICPENDR0` | PPI/SGI 挂起清除 |
| `GICR_PROPBASER` | LPI 配置表基址 |
| `GICR_PENDBASER` | LPI 挂起表基址 |

---

## 13. ITS — 中断翻译服务

### 13.1 概述

ITS（Interrupt Translation Service）是 GICv3 的 MSI/LPI 翻译组件，
将设备发出的 MSI 写入翻译为 LPI 中断。

### 13.2 核心数据结构

`arm_gicv3_its.c`:

```c
// 命令类型 [37-42]
typedef enum ItsCmdType { ... } ItsCmdType;

// 翻译表条目 [44-69]
typedef struct DTEntry { ... } DTEntry;  // Device Table Entry
typedef struct CTEntry { ... } CTEntry;  // Collection Table Entry
typedef struct ITEntry { ... } ITEntry;  // Interrupt Translation Entry
typedef struct VTEntry { ... } VTEntry;  // Virtual Translation Entry
```

### 13.3 MSI 翻译流程

```
设备写入 GITS_TRANSLATER 寄存器（MSI 写入）
  │
  ▼
ITS 查找翻译表（Guest 内存中）
  ├── table_entry_addr()                [129-172]  定位表条目
  ├── get_ite()                         [249-260]  获取 IT Entry
  │    └── address_space_ldq_le()       读取 Guest 内存
  ├── get_cte()                         [180-206]  获取 Collection Entry
  │    └── 确定目标 CPU
  └── 触发 LPI
       └── gicv3_redist_process_lpi()   向目标 Redistributor 发送 LPI
```

### 13.4 ITS 命令队列

`process_cmdq()` 处理 Guest 写入的命令队列：

| 命令 | 处理函数 | 说明 |
|------|---------|------|
| `MAPD` | `process_mapd()` | 映射 Device Table |
| `MAPC` | `process_mapc()` | 映射 Collection Table |
| `MAPTI` | `process_mapti()` :579-647 | 映射中断翻译 |
| `MOVI` | `process_movi()` | 移动中断到其他 Collection |
| `INV` | `process_inv()` | 使翻译缓存无效 |
| `INT` | — | 软件触发中断 |
| `VMAPTI` | `process_vmapti()` :649-725 | 虚拟中断映射（vGIC） |

---

## 14. virt 机器 GIC 集成

### 14.1 create_gic()

`hw/arm/virt.c` 中 `create_gic()` 的工作：

```
create_gic(vms)
  ├── 选择 GIC 类型:
  │    gicv3_class_name()              # v3/v4: "arm-gicv3" 或 "kvm-arm-gicv3"
  │
  ├── 设置属性:
  │    revision, num-cpu, num-irq (= NUM_IRQS + 32)
  │    redist-region-count (v3)
  │    has-lpi, has-security-extensions
  │    has-virtualization-extensions, has-nmi
  │
  ├── Realize + MMIO 映射:
  │    sysbus_realize_and_unref()
  │    map GICD → VIRT_GIC_DIST
  │    map GICR → VIRT_GIC_REDIST / VIRT_HIGH_GIC_REDIST2
  │
  └── IRQ 连线:
       ├── GIC → CPU (每 CPU):
       │    IRQ, FIQ, VIRQ, VFIQ, NMI, VINMI
       └── 设备 → GIC:
            qdev_get_gpio_in(vms->gic, irq)
```

### 14.2 设备 IRQ 连线示例

| 设备 | 源码位置 | 连线方式 |
|------|----------|---------|
| PL011 UART | virt.c:1295-1344 | `sysbus_connect_irq()` → GIC SPI |
| PL031 RTC | virt.c:1347-1368 | `sysbus_connect_irq()` → GIC SPI |
| PCIe 根复合体 | virt.c:1974-1977 | IRQ map → GIC SPI |
| Platform Bus | virt.c:2071-2074 | 批量 → GIC SPI |
| SMMUv3 | virt.c:1876-1879 | `sysbus_connect_irq()` → GIC SPI |

### 14.3 完整中断流路径

```
设备 (如 UART 收到数据)
  │  qemu_set_irq(uart_irq, 1)
  ▼
GIC SPI 线
  │  gicv3_set_irq()
  ▼
GICD 检查 → GICR 更新 → GICC 更新
  │
  ▼
ARMCPU GPIO: ARM_CPU_IRQ
  │  CPUState.interrupt_request |= CPU_INTERRUPT_HARD
  ▼
[KVM] KVM 注入中断到 vCPU
[TCG] arm_cpu_exec_interrupt() 检查 → 触发异常
  │
  ▼
Guest 内核 IRQ handler 执行
```

---

# 第三部分：TCG 翻译引擎

## 15. TCG 概述

TCG（Tiny Code Generator）是 QEMU 的动态二进制翻译引擎：

```
Guest ARM64 指令 → TCG IR（中间表示）→ Host 原生代码
```

### 文件组织

| 目录 | 说明 |
|------|------|
| `tcg/` | TCG 核心：IR 定义、后端代码生成 |
| `accel/tcg/` | TCG 加速器框架：TB 管理、执行循环 |
| `target/arm/tcg/` | ARM64 前端：指令解码、翻译到 TCG IR |

### ARM64 TCG 关键文件

| 文件 | 说明 |
|------|------|
| `translate-a64.c` | AArch64 指令解码与翻译 |
| `translate.c` | ARM 前端入口 |
| `cpu64.c` | AArch64 TCG CPU 模型 |
| `helper-a64.c` | AArch64 运行时辅助函数 |
| `tlb_helper.c` | TLB 异常处理 |
| `op_helper.c` | 通用操作辅助 |
| `sve_helper.c` | SVE 指令辅助 |
| `sme_helper.c` | SME 指令辅助 |
| `pauth_helper.c` | 指针认证辅助 |
| `mte_helper.c` | 内存标签扩展辅助 |

---

## 16. 翻译流程

### 16.1 指令解码

ARM64 使用 **decodetree** 生成的解码器（非手写）：

```c
// translate-a64.c:76-80
#include "decode-sme-fa64.c.inc"    // SME FA64 解码表
#include "decode-a64.c.inc"          // 主 AArch64 解码表
```

### 16.2 翻译入口

```
arm_translate_code()                    [translate.c:6899]
  └── aarch64_translate_code()          [translate-a64.c:10954]
       ├── arm_tr_init_disas_context()  [translate.c:6310]
       │    └── 初始化反汇编上下文
       ├── arm_tr_tb_start()            [translate.c:6417]
       │    └── TB 开始准备
       └── 循环解码每条指令:
            ├── 从 Guest 内存读取指令
            ├── decode-a64.c.inc 匹配解码模式
            └── 生成对应的 TCG IR ops

> **佐证 commit**: `d41238625e` "extract aarch64_translate_code()" — 
> 从通用入口中提取了 AArch64 专用翻译函数。
```

---

## 17. TranslationBlock

### 17.1 结构

`include/exec/translation-block.h:46-150`:

```
TranslationBlock (TB)
  ├── 标志: CF_NO_GOTO_TB, CF_INVALID, CF_PCREL    [77-86]
  ├── 两个出口跳转目标                              [118-149]
  ├── 入口跳转链表（其他 TB 跳入此 TB）
  └── 翻译后的 Host 代码
```

### 17.2 TB 链接（Chaining）

- TB 执行完毕后通过 `goto_tb` / 直接代码修补跳转到下一个 TB
- 避免了回到 QEMU 主循环的开销
- `tb_add_jump()` 在运行时建立 TB 间的链接

### 17.3 TB 失效（Invalidation）

- 当 Guest 代码页被修改时触发
- `tb_invalidate_phys_range()` (`accel/tcg/tb-maint.c:1024`)
- `CF_INVALID` 标记无效 TB
- `tb_link_page()` (`tb-maint.c:992`) 将 TB 链入页面结构用于追踪

---

## 18. TCG IR 与寄存器映射

### 18.1 AArch64 寄存器映射

`translate-a64.c:83-109`:

```c
// AArch64 通用寄存器 → TCG 全局变量
cpu_X[32]  →  tcg_global_mem_new_i64(
                  cpu_env, offsetof(CPUARMState, xregs[i]), ...)
cpu_pc     →  TCG 全局（程序计数器）
cpu_exclusive_high  →  独占访问高位
cpu_gcspr[]         →  GCS 栈指针
```

### 18.2 关键 TCG 操作

| TCG Op | 用途 |
|--------|------|
| `tcg_gen_qemu_ld_i64` | 生成内存加载 |
| `tcg_gen_qemu_st_i64` | 生成内存存储 |
| `tcg_gen_goto_tb` | 生成 TB 间直接跳转 |
| `tcg_gen_lookup_and_goto_ptr` | 查找 TB 并跳转 |
| `tcg_gen_addi_i64` 等 | 算术操作 |

---

## 19. MMU/TLB 与页表遍历

### 19.1 TLB 结构

TCG 维护软件 TLB 用于加速 Guest 虚拟地址翻译：

```
Guest VA → 查找 TLB
  ├── 命中 → 直接访问 Host 内存
  └── 未命中 → 慢速路径
       └── ARM MMU 页表遍历
            ├── 读取 TTBR0/TTBR1
            ├── 多级页表查表
            └── 填充 TLB 条目
```

### 19.2 地址空间辅助

`translate-a64.c:122-156`:
- `full_a64_user_mem_index()` — 完整 mem index
- `core_a64_user_mem_index()` — 核心 mem index

`translate-a64.c:215-244`:
- `gen_top_byte_ignore()` — TBI（Top Byte Ignore）处理

### 19.3 TLB 异常处理

`target/arm/tcg/tlb_helper.c`:
- 故障/syndrome 构造 (:25+)
- `arm_deliver_fault()` (:173) — 将故障传递给异常处理

---

## 20. 中断与异常在 TCG 中的处理

### 20.1 TB 间中断检查

```
TB 执行完毕
  └── HELPER(lookup_tb_ptr)            [cpu-exec.c:374-405]
       ├── 检查 CPUState.interrupt_request
       └── 如果有待处理中断:
            └── arm_cpu_exec_interrupt()  [cpu-irq.c:171]
                 ├── 评估 IRQ/FIQ/NMI/VIRQ 等
                 ├── 决定是否在当前 EL 响应
                 └── 如果响应: 设置 exception_index
                      └── arm_cpu_do_interrupt()
                           └── 进入 Guest 异常处理向量
```

### 20.2 同步异常

翻译过程中遇到特权指令或内存故障时：
- 生成辅助函数调用（`gen_helper_*`）
- 运行时辅助函数设置 `exception.syndrome` 和 `exception.target_el`
- 触发 `arm_cpu_do_interrupt()`

---

## 附录：相关 Git Commit

| Commit | 说明 |
|--------|------|
| `bb36be6fd7` | 引入 cpreg 迁移容忍度基础设施 |
| `970ea8478c` | 允许 TCG CPU 设置 `aarch64=off` |
| `95146de5d2` | 禁用 AArch64 时清除 ID 寄存器 |
| `a051f78714` | 将 GICv3 区域声明为小端 |
| `d89daa893f` | GICv3 实现 NMI 中断优先级 |
| `31164ebf08` | GICv3 MemoryRegionOps 添加 valid/impl 约束 |
| `703090770c` | GICD_CTLR.EnableGrp1NS 对 LPI 的影响 |
| `d41238625e` | 提取 `aarch64_translate_code()` |
| `6981e88add` | 使用 `translator_ldl_end` 替代 `arm_ldl_code` |
