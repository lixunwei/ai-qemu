# TrustZone 安全扩展与 Secure World 深度分析

> QEMU 版本: v11.0.50  
> 源码基线: commit `4575da5ecb`  
> 关联文档: [06-异常级别状态管理深度分析](06-异常级别状态管理深度分析.md)、[07-不同EL下指令执行差异深度分析](07-不同EL下指令执行差异深度分析.md)  
> 分析日期: 2025-07

---

## 目录

1. [概述](#1-概述)
2. [安全状态模型](#2-安全状态模型)
3. [ARMSecuritySpace 枚举与判定函数](#3-armsecurityspace-枚举与判定函数)
4. [SCR_EL3 安全配置寄存器](#4-scr_el3-安全配置寄存器)
5. [CPU 复位与初始安全状态](#5-cpu-复位与初始安全状态)
6. [arm_emulate_firmware_reset() 固件模拟](#6-arm_emulate_firmware_reset-固件模拟)
7. [双地址空间架构](#7-双地址空间架构)
8. [ARM virt 机器的安全配置](#8-arm-virt-机器的安全配置)
9. [安全内存区域布局](#9-安全内存区域布局)
10. [安全 Flash 与固件加载](#10-安全-flash-与固件加载)
11. [安全 RAM 与安全外设](#11-安全-ram-与安全外设)
12. [MemTxAttrs 安全属性传递](#12-memtxattrs-安全属性传递)
13. [MMU 与安全状态的映射](#13-mmu-与安全状态的映射)
14. [页表遍历中的安全状态](#14-页表遍历中的安全状态)
15. [SMC 异常处理与路由](#15-smc-异常处理与路由)
16. [PSCI 电源状态协调接口](#16-psci-电源状态协调接口)
17. [世界切换机制](#17-世界切换机制)
18. [寄存器 Bank（安全/非安全副本）](#18-寄存器-bank安全非安全副本)
19. [GIC 安全配置](#19-gic-安全配置)
20. [中断路由与安全世界](#20-中断路由与安全世界)
21. [TrustZone 保护控制器（MPC/PPC）](#21-trustzone-保护控制器mpcppc)
22. [SMMU 与安全地址空间](#22-smmu-与安全地址空间)
23. [安全启动流程时序](#23-安全启动流程时序)
24. [总结](#24-总结)

---

## 1. 概述

ARM TrustZone 技术通过硬件级别的安全/非安全世界隔离，为系统提供可信执行环境（TEE）。在 QEMU 中，TrustZone 模拟涵盖以下核心组件：

```
TrustZone 模拟架构：

┌────────────────────────────────────────────────────┐
│                  EL3 (Secure Monitor)               │
│  ┌──────────────────┐  ┌────────────────────────┐  │
│  │  SCR_EL3.NS=0    │  │  SCR_EL3.NS=1          │  │
│  │  Secure World     │  │  Non-secure World      │  │
│  │  ┌────────────┐  │  │  ┌──────────────────┐  │  │
│  │  │ EL1(S)     │  │  │  │ EL2 (Hypervisor) │  │  │
│  │  │ OP-TEE/TFA │  │  │  │                  │  │  │
│  │  ├────────────┤  │  │  ├──────────────────┤  │  │
│  │  │ EL0(S)     │  │  │  │ EL1(NS) Linux    │  │  │
│  │  │ TA Apps    │  │  │  │ EL0(NS) Apps     │  │  │
│  │  └────────────┘  │  │  └──────────────────┘  │  │
│  └──────────────────┘  └────────────────────────┘  │
├────────────────────────────────────────────────────┤
│  Secure Address Space  │  Non-secure Address Space  │
│  ┌─────────────────┐   │  ┌──────────────────────┐ │
│  │ Secure Flash    │   │  │ Normal Flash         │ │
│  │ Secure RAM      │   │  │ Normal RAM           │ │
│  │ Secure UART     │   │  │ Normal Devices       │ │
│  └─────────────────┘   │  └──────────────────────┘ │
└────────────────────────────────────────────────────┘
```

---

## 2. 安全状态模型

ARM 架构定义了四种安全状态（ARMv9 扩展了 Realm 状态）：

| 安全状态 | SCR_EL3.NS | SCR_EL3.NSE | 说明 |
|---------|-----------|-------------|------|
| Secure | 0 | - | 传统安全世界 |
| Non-secure | 1 | 0 | 传统非安全世界 |
| Root | - | - | RME: EL3 自身的独立状态 |
| Realm | 1 | 1 | RME: CCA 机密计算 |

QEMU 通过 `SCR_EL3.NS` 和 `SCR_EL3.NSE` 两个位的组合来确定当前安全状态。EL3 本身总是被视为 Secure（或 Root，如果支持 RME）。

---

## 3. ARMSecuritySpace 枚举与判定函数

### 3.1 枚举定义

```c
// arm-security.h:18-23
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,   // 安全世界
    ARMSS_NonSecure  = 1,   // 非安全世界
    ARMSS_Root       = 2,   // RME: Root 世界
    ARMSS_Realm      = 3,   // RME: Realm 世界
} ARMSecuritySpace;

// arm-security.h:26-29
static inline bool arm_space_is_secure(ARMSecuritySpace space)
{
    return space == ARMSS_Secure || space == ARMSS_Root;
}
```

### 3.2 安全状态判定函数

```c
// helper.c:10131-10161
ARMSecuritySpace arm_security_space(CPUARMState *env)
{
    // M-profile：直接使用 v7m.secure
    if (arm_feature(env, ARM_FEATURE_M))
        return arm_secure_to_space(env->v7m.secure);

    // 无 EL3：默认非安全
    if (!arm_feature(env, ARM_FEATURE_EL3))
        return ARMSS_NonSecure;

    // AArch64 当前在 EL3
    if (is_a64(env) && extract32(env->pstate, 2, 2) == 3) {
        if (cpu_isar_feature(aa64_rme, ...))
            return ARMSS_Root;   // RME: EL3 = Root
        else
            return ARMSS_Secure; // 传统: EL3 = Secure
    }

    // AArch32 Monitor 模式
    if ((env->uncached_cpsr & CPSR_M) == ARM_CPU_MODE_MON)
        return ARMSS_Secure;

    // 其他 EL：由 SCR_EL3 决定
    return arm_security_space_below_el3(env);
}

// helper.c:10163-10187
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env)
{
    if (!arm_feature(env, ARM_FEATURE_EL3))
        return ARMSS_NonSecure;

    if (!(env->cp15.scr_el3 & SCR_NS))
        return ARMSS_Secure;       // NS=0 → Secure
    else if (env->cp15.scr_el3 & SCR_NSE)
        return ARMSS_Realm;        // NS=1, NSE=1 → Realm
    else
        return ARMSS_NonSecure;    // NS=1, NSE=0 → Non-secure
}
```

### 3.3 便利函数

```c
// arm_is_secure() = arm_space_is_secure(arm_security_space(env))
// arm_is_secure_below_el3() = (arm_security_space_below_el3(env) == ARMSS_Secure)
```

---

## 4. SCR_EL3 安全配置寄存器

SCR_EL3 是 TrustZone 的"总控开关"，其写入函数处理大量副作用。

### 4.1 关键位

| 位 | 名称 | 功能 |
|----|------|------|
| 0 | NS | Non-secure: 控制 EL3 以下的安全状态 |
| 1 | IRQ | IRQ 路由到 EL3 |
| 2 | FIQ | FIQ 路由到 EL3 |
| 3 | EA | External Abort 路由到 EL3 |
| 7 | SMD | SMC Disable |
| 8 | HCE | HVC Enable |
| 10 | RW | EL2 执行状态（1=AArch64） |
| 11 | ST | Secure Timer |
| 12 | TWI | WFI Trap 到 EL3 |
| 13 | TWE | WFE Trap 到 EL3 |
| 21 | EEL2 | Secure EL2 Enable |

### 4.2 写入函数

```c
// helper.c: scr_write()
// 关键副作用：
// 1. 根据 CPU 特性掩码有效位
// 2. AArch64 时强制设置 FW, AW 为 RES1
// 3. NS/NSE 变化时刷新 EL3 以下所有 TLB
// 4. RW 位变化影响 EL2 执行状态
// 5. EEL2 位控制 Secure EL2 使能
```

当 `SCR_EL3.NS` 发生变化时，所有 EL3 以下的 TLB 都需要刷新，因为安全状态改变意味着完全不同的地址翻译上下文。

---

## 5. CPU 复位与初始安全状态

### 5.1 复位时的 EL 选择

```c
// cpu.c:402-413
// AArch64: 复位到最高可用 EL
if (arm_feature(env, ARM_FEATURE_EL3)) {
    env->pstate = PSTATE_MODE_EL3h;    // 有 EL3 → 从 EL3 启动
} else if (arm_feature(env, ARM_FEATURE_EL2)) {
    env->pstate = PSTATE_MODE_EL2h;    // 有 EL2 → 从 EL2 启动
} else {
    env->pstate = PSTATE_MODE_EL1h;    // 否则 EL1 启动
}

// 复位向量来自 RVBAR
env->cp15.rvbar = cpu->rvbar_prop;
env->pc = env->cp15.rvbar;

// AArch32: 选择处理器模式
if (arm_feature(env, ARM_FEATURE_EL2) &&
    !arm_feature(env, ARM_FEATURE_EL3)) {
    env->uncached_cpsr = ARM_CPU_MODE_HYP;  // 无 EL3 有 EL2 → HYP
} else {
    env->uncached_cpsr = ARM_CPU_MODE_SVC;  // 其他 → SVC
}

// 所有中断屏蔽
env->daif = PSTATE_D | PSTATE_A | PSTATE_I | PSTATE_F;
```

**关键点**：有 EL3 的 CPU 复位后处于 **Secure EL3**，所有中断屏蔽，PC 指向 RVBAR。这是 Secure Monitor/EL3 固件的起始点。

---

## 6. arm_emulate_firmware_reset() 固件模拟

当 QEMU 不使用真实 EL3 固件时，通过此函数模拟固件完成后的状态。

```c
// cpu.c:729-777
void arm_emulate_firmware_reset(CPUState *cs, int target_el)
{
    // ... 根据 CPU 特性设置 SCR_EL3 的各功能位 ...
    // RW, TCR2EN, SCTLR2EN, PIEN, AIEN, MECEN 等

    if (target_el == 2) {
        env->cp15.scr_el3 |= SCR_HCE;  // EL2 需要 HVC 使能
    }

    // 关键：将 CPU 置于非安全状态
    env->cp15.scr_el3 |= SCR_NS;
    env->cp15.nsacr |= 3 << 10;  // 允许 NS 访问 FPU (CP10/CP11)

    // 如果有 EL2 且目标低于 EL2，配置 HCR_EL2
    if (have_el2 && target_el < 2) {
        if (env->aarch64) {
            env->cp15.hcr_el2 |= HCR_RW;  // EL1 使用 AArch64
        }
    }

    // 设置目标 EL 的 PSTATE/CPSR
    if (env->aarch64) {
        env->pstate = aarch64_pstate_mode(target_el, true);
    } else {
        cpsr_write(env, mode_for_el[target_el], CPSR_M, CPSRWriteRaw);
    }
}
```

**用途**：当没有加载 EL3 固件时，QEMU 模拟"固件已完成初始化"，将 CPU 从 EL3 降级到目标 EL（通常 EL2 或 EL1），并设置好所有必要的系统寄存器。

---

## 7. 双地址空间架构

QEMU 为支持 TrustZone 的 CPU 创建两个独立的物理地址空间。

### 7.1 地址空间索引

```c
// cpu.h 中定义
typedef enum ARMASIdx {
    ARMASIdx_NS = 0,      // Non-secure 地址空间
    ARMASIdx_S  = 1,      // Secure 地址空间
    ARMASIdx_TagNS = 2,   // Non-secure MTE Tag 空间
    ARMASIdx_TagS  = 3,   // Secure MTE Tag 空间
} ARMASIdx;
```

### 7.2 CPU 地址空间创建

```c
// cpu.c:2294-2302
// 每个 CPU 注册多个地址空间：
// "cpu-memory"         → ARMASIdx_NS
// "cpu-secure-memory"  → ARMASIdx_S（仅 secure=on 时）
// "cpu-tag-memory"     → ARMASIdx_TagNS（MTE 时）
// "cpu-secure-tag-memory" → ARMASIdx_TagS（MTE+secure 时）
```

### 7.3 地址空间选择

```c
// cpu.h:2601-2613
// arm_cpu_get_address_space() 根据 MemTxAttrs.secure 选择：
//   attrs.secure = true  → ARMASIdx_S
//   attrs.secure = false → ARMASIdx_NS
```

---

## 8. ARM virt 机器的安全配置

### 8.1 安全模式使能

```c
// virt.c:3876-3878
// 通过 machine 属性 "secure" 控制
// 命令行: -machine virt,secure=on

// virt.c:2649-2661
if (vms->secure) {
    // 创建安全地址空间：一个容器 MemoryRegion
    // 包含普通 sysmem 作为低优先级子区域
    // 安全专用设备以高优先级覆盖
    secure_sysmem = g_new(MemoryRegion, 1);
    vms->secure_sysmem = secure_sysmem;
    memory_region_init(secure_sysmem, OBJECT(machine),
                       "secure-memory", UINT64_MAX);
    // 将普通内存作为 overlay 的底层
    memory_region_add_subregion_overlap(secure_sysmem, 0, sysmem, -1);
}
```

### 8.2 安全地址空间的叠加模型

```
secure_sysmem (容器, 优先级控制):
┌──────────────────────────────────────────┐
│  高优先级：Secure-only 设备              │
│  ├─ Secure Flash (0x00000000-0x03FFFFFF) │
│  ├─ Secure UART1 (0x09040000)           │
│  ├─ Secure GPIO  (0x090b0000)           │
│  └─ Secure RAM   (0x0e000000-0x0eFFFFFF)│
├──────────────────────────────────────────┤
│  低优先级 (-1)：普通 sysmem 内容         │
│  （所有非安全设备和内存都可见）           │
└──────────────────────────────────────────┘
```

**设计关键**：安全世界**可以看到所有非安全内存和设备**（通过低优先级 sysmem），同时拥有额外的安全专用区域。非安全世界则**无法看到安全专用区域**。

### 8.3 CPU 链接安全地址空间

```c
// virt.c:2785-2788
if (vms->secure) {
    object_property_set_link(cpuobj, "secure-memory",
                             OBJECT(secure_sysmem), &error_abort);
}
```

---

## 9. 安全内存区域布局

ARM virt 机器的安全相关地址布局：

```c
// virt.c:173-208 (base_memmap)
```

| 区域 | 基地址 | 大小 | 安全属性 |
|------|--------|------|---------|
| VIRT_FLASH (前半) | 0x00000000 | 64MB | Secure only |
| VIRT_FLASH (后半) | 0x04000000 | 64MB | Non-secure |
| VIRT_UART0 | 0x09000000 | 4KB | Non-secure |
| VIRT_UART1 | 0x09040000 | 4KB | Secure only |
| VIRT_SECURE_GPIO | 0x090b0000 | 4KB | Secure only |
| VIRT_SECURE_MEM | 0x0e000000 | 16MB | Secure only |
| VIRT_MEM | 0x40000000+ | 可配置 | Non-secure |

---

## 10. 安全 Flash 与固件加载

### 10.1 Flash 分区

```c
// virt.c:1620-1638
static void virt_flash_map(VirtMachineState *vms,
                           MemoryRegion *sysmem,
                           MemoryRegion *secure_sysmem)
{
    hwaddr flashsize = vms->memmap[VIRT_FLASH].size / 2;  // 64MB each
    hwaddr flashbase = vms->memmap[VIRT_FLASH].base;      // 0x00000000

    // 前半 Flash → 仅映射到 secure_sysmem
    virt_flash_map1(vms->flash[0], flashbase, flashsize,
                    secure_sysmem);

    // 后半 Flash → 映射到普通 sysmem（两个世界都可见）
    virt_flash_map1(vms->flash[1], flashbase + flashsize, flashsize,
                    sysmem);
}
```

### 10.2 设备树标记

```c
// virt.c:1665-1673
// 安全 Flash 在 DT 中标记为：
//   status = "disabled"         → 非安全世界不可见
//   secure-status = "okay"      → 安全世界可用
```

### 10.3 固件加载方式

EL3 固件可通过以下方式加载：

1. **`-bios` 选项**：加载到 Flash 0（安全 Flash）
2. **`-device loader,file=fw.bin,addr=0x0`**：通用加载器，可指定地址
3. **`-pflash` 选项**：直接指定 Flash 映像

```c
// boot.c:50-67
// arm_boot_address_space() 选择启动地址空间：
//   如果有 EL3 且 secure_boot → 使用 ARMASIdx_S（安全空间）
//   否则 → 使用 ARMASIdx_NS
```

### 10.4 PSCI 与固件的关系

```c
// virt.c:2666-2670
// 如果加载了 EL3 固件（secure Flash 有内容），则禁用 QEMU 内置 PSCI
// 因为假设 EL3 固件（如 TF-A）会自己实现 PSCI
// 此时 secondary CPU 不再使用 PSCI powerdown 启动，而是全部运行
```

---

## 11. 安全 RAM 与安全外设

### 11.1 安全 RAM

```c
// virt.c:2091-2117
static void create_secure_ram(VirtMachineState *vms,
                              MemoryRegion *secure_sysmem, ...)
{
    hwaddr base = vms->memmap[VIRT_SECURE_MEM].base;  // 0x0e000000
    hwaddr size = vms->memmap[VIRT_SECURE_MEM].size;   // 0x01000000 (16MB)

    memory_region_init_ram(secram, NULL, "virt.secure-ram", size, ...);
    memory_region_add_subregion(secure_sysmem, base, secram);

    // DT 节点：/secram@e000000
    //   status = "disabled"
    //   secure-status = "okay"
}
```

### 11.2 安全 UART

```c
// virt.c:2893
// VIRT_UART1 创建时指定 secure=true
create_uart(vms, VIRT_UART1, secure_sysmem, serial_hd(1), true);

// virt.c:1295-1342
// secure=true 时：
//   status = "disabled"        → NS 世界不可见
//   secure-status = "okay"     → Secure 世界可用
```

### 11.3 安全 GPIO

安全 GPIO（VIRT_SECURE_GPIO = 0x090b0000）仅在 `vms->secure` 时创建，映射到 secure_sysmem。

---

## 12. MemTxAttrs 安全属性传递

每次内存事务都携带安全属性，贯穿整个内存层次。

### 12.1 MemTxAttrs 结构

```c
// memattrs.h:25-31
typedef struct MemTxAttrs {
    unsigned int secure:1;    // 1 = Secure 事务
    unsigned int space:2;     // ARMSecuritySpace
    unsigned int user:1;      // 1 = 非特权访问
    unsigned int memory:1;    // 1 = 内存事务（vs 设备）
    ...
} MemTxAttrs;
```

### 12.2 安全属性在内存层次中的传递

```
CPU 发起内存访问
    ↓ attrs.secure = arm_is_secure(env)
MMU / TLB 查找
    ↓ 选择 ARMASIdx_S 或 ARMASIdx_NS
AddressSpace 转换
    ↓ MemTxAttrs 传递给 MemoryRegion
MemoryRegion dispatch
    ↓ 设备可检查 attrs.secure 决定是否允许访问
设备 I/O 处理
```

---

## 13. MMU 与安全状态的映射

安全状态影响 MMU index 的选择，进而决定使用哪套页表。

```c
// helper.c:9957-10008
ARMMMUIdx arm_mmu_idx_el(CPUARMState *env, int el)
{
    // EL0 的 MMU index 取决于安全状态：
    if (arm_is_secure_below_el3(env) && !arm_el_is_aa64(env, 3)) {
        // AArch32 Secure 模式：使用 ARMMMUIdx_E30_0
        // （EL3 控制的 EL0 翻译）
    } else {
        // 普通模式：使用 ARMMMUIdx_E10_0
    }
    // EL3 使用专用 MMU index: ARMMMUIdx_E3
    // Secure EL1 使用与 NS EL1 不同的 TLB 条目
}
```

**关键设计**：安全和非安全世界使用**不同的 MMU index**，确保 TLB 条目不会跨安全边界混用。

---

## 14. 页表遍历中的安全状态

```c
// ptw.c:22-57, 193-221
// 页表遍历时跟踪 ARMSecuritySpace
// Stage-2 翻译根据安全状态选择不同的地址空间

// ptw.c:339-453
// GPT (Granule Protection Table) 检查使用 secure attrs
// 通过 address_space_ldq_le(..., attrs) 保持安全属性
```

安全状态影响的翻译路径：
- **Secure EL1/EL0**：使用 Secure Stage-1 页表（TTBR0_EL1(S)/TTBR1_EL1(S)）
- **Non-secure EL1/EL0**：使用 Non-secure Stage-1 页表 + 可选的 Stage-2
- **EL3**：使用 EL3 专用页表（TTBR0_EL3），总是 Secure 翻译
- **RME**：还需要通过 GPT (Granule Protection Table) 检查

---

## 15. SMC 异常处理与路由

SMC 是从非安全世界进入安全世界的主要通道。

### 15.1 翻译阶段

```c
// translate-a64.c:3193-3205
static bool trans_SMC(DisasContext *s, arg_i *a)
{
    if (s->current_el == 0) {
        unallocated_encoding(s);  // EL0 不可用
        return true;
    }
    gen_helper_pre_smc(tcg_env, ...);  // 运行时预检查
    gen_exception_insn_el(s, 4, EXCP_SMC, syn_aa64_smc(a->imm), 3);
    return true;
}
```

### 15.2 运行时预检查

```c
// op_helper.c:1155-1199
HELPER(pre_smc):
// 检查顺序：
// 1. 无 EL3 且无 NV 且非 PSCI-via-SMC → UNDEF
// 2. EL1 + HCR_EL2.TSC=1 → Trap 到 EL2（Hypervisor 拦截）
// 3. 非 PSCI 调用 + (SMD 或无 EL3) → UNDEF
// 4. 是 PSCI 调用 → 路由到 PSCI 处理

// SMD (SMC Disable) 位的 EL3 AArch64 vs AArch32 差异：
bool smd = arm_feature(env, ARM_FEATURE_AARCH64) ? smd_flag
                                                 : smd_flag && !secure;
// AArch64: SMD 对安全和非安全都生效
// AArch32: SMD 仅对非安全生效
```

### 15.3 SMC 路由图

```
Guest EL1 (Non-secure) 执行 SMC
    │
    ├─ HCR_EL2.TSC=1? ──YES──→ Trap 到 EL2 (Hypervisor)
    │
    ├─ PSCI 调用? ──YES──→ arm_handle_psci_call()
    │                        ├─ CPU_ON → arm_set_cpu_on()
    │                        ├─ SYSTEM_RESET → qemu_system_reset_request()
    │                        └─ CPU_OFF → cpu_off
    │
    ├─ SCR_EL3.SMD=1? ──YES──→ UNDEF
    │
    └─ 正常 SMC → 异常入口到 EL3
         └─ EL3 固件 (TF-A/OP-TEE) 处理
```

---

## 16. PSCI 电源状态协调接口

QEMU 内置了 PSCI v1.1 的基本实现。

### 16.1 支持的 PSCI 函数

```c
// psci.c:58-226
void arm_handle_psci_call(ARMCPU *cpu)
{
    switch (param[0]) {
    case PSCI_VERSION:          // 返回版本号 1.1
    case MIGRATE_INFO_TYPE:     // 返回不需要迁移
    case AFFINITY_INFO:         // 查询 CPU 电源状态
    case SYSTEM_RESET:          // 系统重启
    case SYSTEM_OFF:            // 系统关机
    case CPU_ON:                // 启动一个 CPU
    case CPU_OFF:               // 关闭当前 CPU
    case CPU_SUSPEND:           // 挂起当前 CPU
    case PSCI_FEATURES:         // 查询功能支持
    }
}
```

### 16.2 CPU_ON 实现

```c
// arm-powerctl.c:84-175
int arm_set_cpu_on(uint64_t cpuid, uint64_t entry, uint64_t context_id, ...)
{
    // 1. 找到目标 CPU
    // 2. 复位目标 CPU
    // 3. arm_emulate_firmware_reset(target_el) 设置启动 EL
    //    target_el = EL2 (如果可用) 或 EL1
    // 4. 设置 x0 = context_id, PC = entry
    // 5. 标记 CPU 为 ON
}
```

---

## 17. 世界切换机制

### 17.1 从非安全到安全（SMC 入口）

```
1. Guest (NS EL1) 执行 SMC 指令
2. CPU 进入异常入口：
   - SPSR_EL3 = 保存当前 PSTATE
   - ELR_EL3 = 返回地址
   - ESR_EL3 = syndrome (EC_AA64_SMC)
   - PSTATE = EL3h, DAIF 全屏蔽
3. PC = VBAR_EL3 + offset
4. 此时 CPU 在 EL3 Secure 状态
5. EL3 固件清除 SCR_EL3.NS → 切换到安全世界
6. 保存 NS 世界上下文，恢复 S 世界上下文
7. ERET 到 Secure EL1（OP-TEE）
```

### 17.2 从安全到非安全（ERET 返回）

```
1. Secure EL1 完成处理，SMC 回到 EL3
2. EL3 固件设置 SCR_EL3.NS=1 → 切换到非安全世界
3. 恢复 NS 世界上下文
4. ERET → 返回到 NS EL1
```

### 17.3 SCR_EL3.NS 切换的影响

当 `SCR_EL3.NS` 从 0 变为 1（或反之）时：
- 所有 EL3 以下的 TLB 条目失效
- EL1 系统寄存器切换到对应世界的 bank
- 地址空间从 Secure 切换到 Non-secure（或反之）

---

## 18. 寄存器 Bank（安全/非安全副本）

### 18.1 Bank 机制

ARM 架构中，某些系统寄存器有安全和非安全两个独立副本。

```c
// helper.c 中的 Bank 寄存器定义
// 使用 .bank_fieldoffsets 指定两个存储位置

// 示例（概念性）：
{
    .name = "SCTLR",
    .bank_fieldoffsets = {
        offsetof(CPUARMState, cp15.sctlr_ns),    // bank[0] = NS
        offsetof(CPUARMState, cp15.sctlr_s),      // bank[1] = S
    },
}
```

### 18.2 access_secure_reg() 判定

```c
// hflags.c:68-75
bool access_secure_reg(CPUARMState *env)
{
    // 仅在以下条件下返回 true：
    // - 有 EL3 特性
    // - EL3 是 AArch32（非 AArch64）
    // - SCR_EL3.NS = 0
    bool ret = (arm_feature(env, ARM_FEATURE_EL3) &&
                !arm_el_is_aa64(env, 3) &&
                !(env->cp15.scr_el3 & SCR_NS));
    return ret;
}
```

**说明**：在 AArch64 模式下，寄存器 banking 的处理方式不同 —— EL3 有自己独立的寄存器集（如 SCTLR_EL3、TTBR0_EL3），而 EL1 的寄存器由 SCR_EL3.NS 控制访问哪个安全上下文。

### 18.3 常见的 Bank 寄存器

| 寄存器 | 安全副本 | 非安全副本 |
|--------|---------|-----------|
| SCTLR_EL1 | sctlr_s | sctlr_ns |
| TTBR0_EL1 | ttbr0_s | ttbr0_ns |
| TTBR1_EL1 | ttbr1_s | ttbr1_ns |
| VBAR_EL1 | vbar_s | vbar_ns |
| DACR (AArch32) | dacr_s | dacr_ns |
| CSSELR | csselr_s | csselr_ns |

---

## 19. GIC 安全配置

### 19.1 中断分组

GICv3 将中断分为三组：

| 组 | 描述 | 典型用途 |
|----|------|---------|
| Group 0 | Secure | FIQ 传递，安全中断 |
| Secure Group 1 | Secure | 安全世界 IRQ |
| Non-secure Group 1 | Non-secure | 非安全世界 IRQ |

### 19.2 GICD_IGROUPR 访问控制

```c
// arm_gicv3_dist.c:17-29, 59-79, 92-193
// GICD_IGROUPR 控制每个中断的分组
// 非安全访问时：
//   Group 0 和 Secure Group 1 的中断 → RAZ/WI
//   仅 Non-secure Group 1 可见

// mask_group_and_nsacr() 函数过滤安全中断
// 确保非安全世界无法看到或修改安全中断配置
```

### 19.3 CPU Interface 安全 Bank

```c
// arm_gicv3_cpuif.c:40-48
// gicv3_use_ns_bank()：
// 当 CPU 不在 secure_below_el3 状态时
// GIC CPU interface 寄存器使用 Non-secure bank
```

---

## 20. 中断路由与安全世界

SCR_EL3 的 IRQ/FIQ/EA 位控制中断到 EL3 的路由。

### 20.1 路由位

| SCR_EL3 位 | 效果 |
|-----------|------|
| IRQ | IRQ 路由到 EL3（用于安全中断） |
| FIQ | FIQ 路由到 EL3（传统安全中断路由） |
| EA | External Abort 路由到 EL3 |

### 20.2 典型配置

**TF-A/OP-TEE 场景**：
- `SCR_EL3.FIQ=1`：Group 0 中断（安全）通过 FIQ 路由到 EL3
- `SCR_EL3.IRQ=0`：Group 1 NS 中断（非安全）直接到 NS EL1/EL2
- EL3 Monitor 收到 FIQ 后，切换到安全世界处理

```c
// cpu.c:781-833
// IRQ/FIQ/NMI GPIO 线映射到 CPU 中断输入
// 路由由 SCR_EL3.{IRQ,FIQ,EA} + HCR_EL2 位共同决定
```

---

## 21. TrustZone 保护控制器（MPC/PPC）

QEMU 实现了完整的 TrustZone 外设保护，主要用于 Cortex-M SSE (Subsystem for Embedded) 平台。

### 21.1 TZ-PPC（外设保护控制器）

```c
// hw/misc/tz-ppc.c:80-103
// PPC 根据 MemTxAttrs.secure 和 .user 决定是否阻止访问
// 非安全访问被标记为安全的外设 → 访问被拒绝
```

### 21.2 TZ-MPC（内存保护控制器）

```c
// hw/misc/tz-mpc.c:363-465
// MPC 将内存区域划分为安全和非安全块
// 使用 IOMMU 接口实现地址翻译
// 安全访问路由到下游内存，非安全访问到受保护区域时被阻止
```

### 21.3 使用 MPC/PPC 的平台

| 平台 | 文件 | TrustZone 组件 |
|------|------|---------------|
| MPS2-TZ | `hw/arm/mps2-tz.c` | PPC + MPC 阵列 |
| Musca (SSE-200) | `hw/arm/musca.c` | PPC + MPC，完整安全分区 |

**注意**：ARM virt 机器（用于 Linux/AArch64）不使用 MPC/PPC，而是通过地址空间隔离实现安全分区。MPC/PPC 主要用于 Cortex-M 嵌入式平台的精确安全边界控制。

---

## 22. SMMU 与安全地址空间

```c
// hw/arm/smmu-common.c:955-959
// SMMU 同时管理安全和非安全地址空间
// 创建 memory_as (Non-secure) 和 secure_memory_as (Secure)
// DMA 事务通过 MemTxAttrs.secure 选择正确的地址空间
```

这确保了非安全设备的 DMA 操作不会访问安全内存区域。

---

## 23. 安全启动流程时序

```
CPU 复位 (cpu.c:402-413)
│  PSTATE = EL3h
│  PC = RVBAR (指向安全 Flash)
│  DAIF = 全屏蔽
│  SCR_EL3 = 复位值（NS=0, Secure 状态）
│
├─ [有 EL3 固件] ─────────────────────────────────┐
│  │                                                │
│  │  EL3 固件初始化 (TF-A BL1/BL2)                │
│  │  ├─ 初始化安全 RAM/外设                        │
│  │  ├─ 加载 OP-TEE 到 Secure EL1                 │
│  │  ├─ 加载 UEFI/Linux 到 NS EL1/EL2            │
│  │  ├─ 设置 SCR_EL3（NS/IRQ/FIQ 路由）          │
│  │  └─ ERET 到 NS EL2/EL1 启动 OS               │
│  │                                                │
│  │  后续 SMC 调用：                               │
│  │  NS EL1 → SMC → EL3 Monitor → Secure EL1     │
│  │  Secure EL1 → SMC → EL3 Monitor → NS EL1     │
│                                                    │
├─ [无 EL3 固件] ─────────────────────────────────┐
│  │                                                │
│  │  do_cpu_reset() (boot.c:693-723)              │
│  │  ├─ arm_emulate_firmware_reset(target_el)     │
│  │  │   ├─ SCR_EL3.NS = 1（切换到非安全）       │
│  │  │   ├─ NSACR 允许 FPU                        │
│  │  │   ├─ HCR_EL2.RW = 1（如果目标 < EL2）    │
│  │  │   └─ PSTATE = target_el                    │
│  │  ├─ PC = kernel entry                         │
│  │  └─ PSCI 内置处理器就绪                       │
│  │                                                │
│  │  PSCI CPU_ON (psci.c:58-226):                 │
│  │  Primary CPU → SMC(CPU_ON, entry, ctx)        │
│  │  → arm_set_cpu_on() → secondary CPU 启动     │
│                                                    │
└──────────────────────────────────────────────────┘
```

---

## 24. 总结

### 24.1 QEMU TrustZone 实现的层次

| 层次 | 组件 | 实现方式 |
|------|------|---------|
| CPU 状态 | ARMSecuritySpace | SCR_EL3.NS/NSE 驱动的四状态模型 |
| 地址空间 | ARMASIdx_S/NS | 每 CPU 双地址空间，MemTxAttrs 传递 |
| 内存隔离 | secure_sysmem | overlay 模型，安全设备高优先级覆盖 |
| 外设保护 | MPC/PPC | Cortex-M SSE 平台的精确访问控制 |
| 异常路由 | SCR_EL3.IRQ/FIQ | 安全中断路由到 EL3 |
| 世界切换 | SMC/ERET | SCR_EL3.NS 翻转 + 上下文保存恢复 |
| 寄存器隔离 | Bank 机制 | AArch32 bank，AArch64 独立寄存器 |
| GIC 安全 | Group 0/1 | IGROUPR 分组，NS 访问过滤 |
| 固件支持 | PSCI | 内置 v1.1 实现，可禁用让 EL3 固件接管 |

### 24.2 关键设计思想

1. **安全世界是非安全世界的超集**：安全地址空间通过 overlay 包含所有非安全内存
2. **SCR_EL3.NS 是唯一的世界切换控制位**：所有安全状态判定最终归结于此
3. **TLB 按安全状态隔离**：不同 MMU index 确保不会跨安全边界缓存
4. **灵活的固件模拟**：可以加载真实 EL3 固件，也可以用 QEMU 内置 PSCI 模拟

---

## 附录 A：关键源文件索引

| 文件 | 内容 |
|------|------|
| `arm-security.h:18-35` | ARMSecuritySpace 枚举与判定 |
| `cpu.c:396-446` | CPU 复位 EL/PSTATE 设置 |
| `cpu.c:729-777` | `arm_emulate_firmware_reset()` |
| `cpu.c:2294-2302` | 双地址空间创建 |
| `helper.c:10131-10187` | `arm_security_space()` / `_below_el3()` |
| `helper.c:712-835` | SCR_EL3 寄存器定义与 `scr_write()` |
| `helper.c:9957-10008` | `arm_mmu_idx_el()` 安全状态映射 |
| `virt.c:173-208` | 安全内存地址布局 |
| `virt.c:1620-1683` | Flash 安全/非安全分区 |
| `virt.c:2091-2117` | 安全 RAM 创建 |
| `virt.c:2649-2661` | `secure_sysmem` 创建 |
| `boot.c:50-67, 693-723` | 安全启动地址空间与复位 |
| `op_helper.c:1155-1199` | SMC 预检查（SMD/TSC/PSCI） |
| `psci.c:58-226` | PSCI 内置实现 |
| `arm-powerctl.c:84-175` | CPU_ON 实现 |
| `ptw.c:22-57, 193-221` | 页表遍历安全状态 |
| `arm_gicv3_dist.c:17-79` | GIC 中断分组安全过滤 |
| `tz-ppc.c:80-103` | TZ-PPC 外设保护 |
| `tz-mpc.c:363-465` | TZ-MPC 内存保护 |

## 附录 B：ARM virt 安全地址空间图

```
地址空间视图对比：

Non-secure 视图 (sysmem):           Secure 视图 (secure_sysmem):
┌──────────────┐ 0x00000000         ┌──────────────┐ 0x00000000
│              │                    │ Secure Flash │ ← 仅安全可见
│ (不可见)     │                    │ (64MB)       │
│              │                    ├──────────────┤ 0x04000000
├──────────────┤ 0x04000000         │ Normal Flash │
│ Normal Flash │                    │ (64MB)       │
│ (64MB)       │                    ├──────────────┤ 0x08000000
├──────────────┤ 0x08000000         │ GIC          │
│ GIC          │                    ├──────────────┤ 0x09000000
├──────────────┤ 0x09000000         │ UART0        │
│ UART0        │                    ├──────────────┤ 0x09040000
├──────────────┤ 0x09040000         │ Secure UART1 │ ← 仅安全可见
│ (不可见)     │                    ├──────────────┤ 0x090b0000
├──────────────┤ 0x090b0000         │ Secure GPIO  │ ← 仅安全可见
│ (不可见)     │                    ├──────────────┤ 0x0e000000
├──────────────┤ 0x0e000000         │ Secure RAM   │ ← 仅安全可见
│ (不可见)     │                    │ (16MB)       │
├──────────────┤ 0x40000000         ├──────────────┤ 0x40000000
│ Normal RAM   │                    │ Normal RAM   │ ← 安全也可见
│              │                    │              │
└──────────────┘                    └──────────────┘
```

## 附录 C：交叉引用

| 文档 | 关联内容 |
|------|---------|
| [06-异常级别状态管理深度分析](06-异常级别状态管理深度分析.md) | EL 切换、PSTATE 保存恢复、ERET 流程 |
| [07-不同EL下指令执行差异深度分析](07-不同EL下指令执行差异深度分析.md) | SMC/HVC 路由、系统寄存器访问控制 |
| [04-中断注入与处理深度分析](04-中断注入与处理深度分析.md) | GICv3 中断分组与路由 |
| [03-GICv3中断控制器模拟架构深度分析](03-GICv3中断控制器模拟架构深度分析.md) | GIC 安全/非安全分组配置 |
| [../architecture/03-Machine建立流程深度分析](../architecture/03-Machine建立流程深度分析.md) | virt 机器初始化流程 |
| [05-FDT设备树深度分析](05-FDT设备树深度分析.md) | secure-status DT 属性 |
