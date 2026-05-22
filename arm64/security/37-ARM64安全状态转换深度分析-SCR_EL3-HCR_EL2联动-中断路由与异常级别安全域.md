# 37 - ARM64 安全状态转换深度分析 — SCR_EL3/HCR_EL2 联动、中断路由与异常级别安全域

> **基于 QEMU 11.0.50 源码**，深入分析 ARM64 安全状态（Secure/Non-Secure/Root/Realm）
> 在 EL 切换时的转换机制：SCR_EL3 位域控制、HCR_EL2 有效值遮罩、
> 中断路由联动、SMC/HVC 异常路由、ERET 安全性校验、VHE 寄存器重定向
> 以及 Secure EL2 支持。

---

## 目录

1. [ARMSecuritySpace：四域安全模型](#1-armsecurityspace四域安全模型)
2. [SCR_EL3：安全配置寄存器](#2-scr_el3安全配置寄存器)
3. [安全状态判定函数](#3-安全状态判定函数)
4. [HCR_EL2：Hypervisor 配置寄存器](#4-hcr_el2hypervisor-配置寄存器)
5. [arm_hcr_el2_eff：有效值遮罩与安全状态联动](#5-arm_hcr_el2_eff有效值遮罩与安全状态联动)
6. [SCR_EL3 写入与安全状态切换](#6-scr_el3-写入与安全状态切换)
7. [中断路由：SCR_EL3 与 HCR_EL2 联动](#7-中断路由scr_el3-与-hcr_el2-联动)
8. [中断屏蔽与安全状态](#8-中断屏蔽与安全状态)
9. [SMC 异常路由](#9-smc-异常路由)
10. [HVC 异常路由](#10-hvc-异常路由)
11. [异常进入时的安全状态处理](#11-异常进入时的安全状态处理)
12. [ERET 异常返回时的安全性校验](#12-eret-异常返回时的安全性校验)
13. [arm_el_is_aa64：寄存器宽度与安全状态](#13-arm_el_is_aa64寄存器宽度与安全状态)
14. [Secure EL2（SEL2）支持](#14-secure-el2sel2支持)
15. [VHE 寄存器重定向](#15-vhe-寄存器重定向)
16. [Stage 2 MMU 与安全状态](#16-stage-2-mmu-与安全状态)
17. [安全状态转换全景图](#17-安全状态转换全景图)

---

## 1. ARMSecuritySpace：四域安全模型

ARMv9 引入了四种安全域（Security Space），扩展了传统的 Secure/Non-Secure 二分模型：

```c
// arm-security.h:18-23
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,   // 安全世界
    ARMSS_NonSecure  = 1,   // 非安全世界
    ARMSS_Root       = 2,   // 根世界（RME EL3）
    ARMSS_Realm      = 3,   // 领域世界（RME）
} ARMSecuritySpace;

// arm-security.h:26-29
static inline bool arm_space_is_secure(ARMSecuritySpace space)
{
    return space == ARMSS_Secure || space == ARMSS_Root;
}
```

**编码规则**：枚举值对应 `NSE:NS` 的低 2 位拼接：

| NSE | NS | 安全域 |
|-----|----|--------|
| 0 | 0 | Secure |
| 0 | 1 | Non-Secure |
| 1 | 0 | Root（RME EL3）|
| 1 | 1 | Realm |

---

## 2. SCR_EL3：安全配置寄存器

SCR_EL3 是 EL3 控制低异常级别安全行为的核心寄存器。

### 2.1 关键位域定义

```c
// cpu.h:1756-1798
#define SCR_NS      (1ULL << 0)   // Non-Secure：控制 EL2/EL1/EL0 安全状态
#define SCR_IRQ     (1ULL << 1)   // IRQ 路由到 EL3
#define SCR_FIQ     (1ULL << 2)   // FIQ 路由到 EL3
#define SCR_EA      (1ULL << 3)   // 外部异常路由到 EL3
#define SCR_FW      (1ULL << 4)   // 非安全态 CPSR.F 可写
#define SCR_AW      (1ULL << 5)   // 非安全态 CPSR.A 可写
#define SCR_NET     (1ULL << 6)   // Not Early Termination（RES0 in AArch64）
#define SCR_SMD     (1ULL << 7)   // 禁用 SMC 指令
#define SCR_HCE     (1ULL << 8)   // 使能 HVC 指令
#define SCR_SIF     (1ULL << 9)   // Secure Instruction Fetch
#define SCR_RW      (1ULL << 10)  // 低于 EL3 的寄存器宽度（1=AArch64）
#define SCR_ST      (1ULL << 11)  // Secure Timer 访问
#define SCR_TWI     (1ULL << 12)  // WFI 陷阱到 EL3
#define SCR_TWE     (1ULL << 13)  // WFE 陷阱到 EL3
#define SCR_EEL2    (1ULL << 18)  // 使能 Secure EL2
#define SCR_NSE     (1ULL << 62)  // NSE 位（RME：与 NS 组合确定安全域）
```

### 2.2 安全状态分类

| SCR_EL3.NS | SCR_EL3.NSE | 低 EL 安全域 |
|------------|-------------|-------------|
| 0 | 0 | Secure |
| 1 | 0 | Non-Secure |
| 0 | 1 | **保留（非法）** |
| 1 | 1 | Realm（需 RME） |

---

## 3. 安全状态判定函数

### 3.1 arm_security_space — 当前安全域

```c
// helper.c:10131-10161
ARMSecuritySpace arm_security_space(CPUARMState *env)
{
    if (!arm_feature(env, ARM_FEATURE_EL3))
        return ARMSS_NonSecure;    // 无 EL3 默认非安全

    // AArch64 EL3
    if (is_a64(env)) {
        if (extract32(env->pstate, 2, 2) == 3) {   // 当前在 EL3
            if (cpu_isar_feature(aa64_rme, ...))
                return ARMSS_Root;   // RME：EL3 = Root 世界
            else
                return ARMSS_Secure; // 无 RME：EL3 = Secure
        }
    } else {
        if ((env->uncached_cpsr & CPSR_M) == ARM_CPU_MODE_MON)
            return ARMSS_Secure;     // AArch32 Monitor = Secure
    }

    return arm_security_space_below_el3(env);  // 非 EL3 级别
}
```

### 3.2 arm_security_space_below_el3 — EL3 以下的安全域

```c
// helper.c:10163-10187
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env)
{
    if (!arm_feature(env, ARM_FEATURE_EL3))
        return ARMSS_NonSecure;

    if (!(env->cp15.scr_el3 & SCR_NS))
        return ARMSS_Secure;          // NS=0 → Secure
    else if (env->cp15.scr_el3 & SCR_NSE)
        return ARMSS_Realm;           // NS=1, NSE=1 → Realm
    else
        return ARMSS_NonSecure;       // NS=1, NSE=0 → Non-Secure
}
```

**关键洞察**：低 EL 的安全状态完全由 SCR_EL3.{NS, NSE} 决定，与 PSTATE 无关。EL3 的安全状态由是否支持 RME 决定（Root vs Secure）。

---

## 4. HCR_EL2：Hypervisor 配置寄存器

### 4.1 关键位域定义

```c
// cpu.h:1695-1755（部分）
#define HCR_VM      (1ULL << 0)   // Stage 2 地址翻译使能
#define HCR_FMO     (1ULL << 3)   // FIQ 路由到 EL2（物理）
#define HCR_IMO     (1ULL << 4)   // IRQ 路由到 EL2（物理）
#define HCR_AMO     (1ULL << 5)   // SError 路由到 EL2
#define HCR_VF      (1ULL << 6)   // 虚拟 FIQ 挂起
#define HCR_VI      (1ULL << 7)   // 虚拟 IRQ 挂起
#define HCR_VSE     (1ULL << 8)   // 虚拟 SError 挂起
#define HCR_DC      (1ULL << 12)  // 默认 Cacheable（禁用 Stage 1 启用 Stage 2）
#define HCR_TSC     (1ULL << 19)  // SMC 陷阱到 EL2
#define HCR_HCD     (1ULL << 29)  // HVC 禁用
#define HCR_TGE     (1ULL << 27)  // 陷阱通用异常（EL0 陷阱到 EL2）
#define HCR_RW      (1ULL << 31)  // EL1 寄存器宽度（1=AArch64）
#define HCR_E2H     (1ULL << 34)  // VHE 使能（EL2 Host 扩展）
#define HCR_NV      (1ULL << 42)  // 嵌套虚拟化
#define HCR_FWB     (1ULL << 46)  // Stage 2 Forced Write-Back
```

---

## 5. arm_hcr_el2_eff：有效值遮罩与安全状态联动

这是 QEMU 中最关键的安全状态联动函数。HCR_EL2 的"有效值"取决于当前安全状态。

```c
// helper.c:3818-3889
uint64_t arm_hcr_el2_eff_secstate(CPUARMState *env, ARMSecuritySpace space)
{
    uint64_t ret = env->cp15.hcr_el2;

    // ① 安全状态检查：EL2 未使能时，HCR 整体无效
    if (!arm_is_el2_enabled_secstate(env, space)) {
        // 安全态下 EL2 仅在 SCR_EEL2=1 时使能
        // 非安全态下 EL2 总是使能（如果存在）
        return 0;   // HCR 所有位无效！
    }

    // ② AArch32 EL2 过滤：去除 AArch32 无效位
    if (!arm_el_is_aa64(env, 2)) {
        uint64_t aa32_valid = MAKE_64BIT_MASK(0, 32) & ~(HCR_RW | HCR_TDZ);
        aa32_valid |= (HCR_CD | HCR_ID | HCR_TERR | ...);
        ret &= aa32_valid;
    }

    // ③ TGE 模式调整
    if (ret & HCR_TGE) {
        if (ret & HCR_E2H) {
            // VHE + TGE：清除虚拟化相关位
            ret &= ~(HCR_VM | HCR_FMO | HCR_IMO | HCR_AMO |
                     HCR_BSU_MASK | HCR_DC | HCR_TWI | HCR_TWE | ...);
        } else {
            // 非 VHE + TGE：强制路由中断到 EL2
            ret |= HCR_FMO | HCR_IMO | HCR_AMO;
        }
        // TGE 始终清除这些位
        ret &= ~(HCR_SWIO | HCR_PTW | HCR_VF | HCR_VI | HCR_VSE |
                 HCR_FB | HCR_TID1 | HCR_TID3 | HCR_TSC | ...);
    }

    return ret;
}

// helper.c:3883-3889 — 包装函数
uint64_t arm_hcr_el2_eff(CPUARMState *env)
{
    if (arm_feature(env, ARM_FEATURE_M)) return 0;
    return arm_hcr_el2_eff_secstate(env, arm_security_space_below_el3(env));
}
```

**核心规则**：

| 安全状态 | SCR_EL3.EEL2 | HCR_EL2 有效值 |
|----------|-------------|---------------|
| Non-Secure | — | 正常读取 |
| Secure | 0 | **全部为 0**（HCR 无效）|
| Secure | 1 | 正常读取（Secure EL2）|
| Realm | — | 正常读取 |

---

## 6. SCR_EL3 写入与安全状态切换

```c
// helper.c:712-836
static void scr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    uint64_t valid_mask = 0x3fff;  // 基础 v8.0

    // ① AArch64 EL3 特殊处理
    if (arm_el_is_aa64(env, 3)) {
        value |= SCR_FW | SCR_AW;      // RES1
        valid_mask &= ~SCR_NET;         // RES0

        // 按特性扩展有效位
        if (cpu_isar_feature(aa64_sel2, cpu))
            valid_mask |= SCR_EEL2;     // Secure EL2
        if (cpu_isar_feature(aa64_rme, cpu))
            valid_mask |= SCR_NSE | SCR_GPF;  // RME
        // ... 其他特性 ...
    }

    // ② EL2 不存在时，清除 HCE 和 SMD（ARMv7）
    if (!arm_feature(env, ARM_FEATURE_EL2))
        valid_mask &= ~SCR_HCE;

    // ③ 写入并计算变化
    value &= valid_mask;
    changed = env->cp15.scr_el3 ^ value;
    env->cp15.scr_el3 = value;

    // ④ 安全状态切换时刷新 TLB
    if (changed & (SCR_NS | SCR_NSE)) {
        tlb_flush_by_mmuidx(env_cpu(env),
            ARMMMUIdxBit_E10_0 | ARMMMUIdxBit_E10_0_GCS |
            ARMMMUIdxBit_E20_0 | ARMMMUIdxBit_E20_0_GCS |
            ARMMMUIdxBit_E10_1 | ARMMMUIdxBit_E10_1_PAN |
            ARMMMUIdxBit_E10_1_GCS |
            ARMMMUIdxBit_E20_2 | ARMMMUIdxBit_E20_2_PAN |
            ARMMMUIdxBit_E20_2_GCS |
            ARMMMUIdxBit_E2   | ARMMMUIdxBit_E2_GCS);
    }
}
```

**关键洞察**：当 `SCR_NS` 或 `SCR_NSE` 发生变化（即安全域切换），QEMU 必须刷新 EL3 以下所有 MMU 索引的 TLB，因为不同安全域使用不同的地址翻译表。

---

## 7. 中断路由：SCR_EL3 与 HCR_EL2 联动

### 7.1 物理中断目标 EL 查找表

```c
// helper.c:8310-8364
// 6 维查找表：target_el_table[is64][scr][rw][hcr][secure][cur_el]
//                                |     |    |    |     |       |
//                              64位EL3 SCR位 RW  HCR位 安全态  当前EL

// helper.c:8369-8420
uint32_t arm_phys_excp_target_el(CPUState *cs, uint32_t excp_idx,
                                 uint32_t cur_el, bool secure)
{
    // ① 确定 SCR 路由位
    switch (excp_idx) {
    case EXCP_IRQ:
    case EXCP_NMI:
        scr = (env->cp15.scr_el3 & SCR_IRQ);   // SCR_EL3.IRQ
        hcr = hcr_el2 & HCR_IMO;                // HCR_EL2.IMO
        break;
    case EXCP_FIQ:
        scr = (env->cp15.scr_el3 & SCR_FIQ);   // SCR_EL3.FIQ
        hcr = hcr_el2 & HCR_FMO;                // HCR_EL2.FMO
        break;
    default:  // SError/External Abort
        scr = (env->cp15.scr_el3 & SCR_EA);    // SCR_EL3.EA
        hcr = hcr_el2 & HCR_AMO;                // HCR_EL2.AMO
        break;
    }

    // ② TGE 等同于 HCR 路由
    hcr |= (hcr_el2 & HCR_TGE) != 0;

    // ③ 查表
    target_el = target_el_table[is64][scr][rw][hcr][secure][cur_el];
}
```

### 7.2 路由优先级规则

**SCR_EL3 优先级高于 HCR_EL2**：

| SCR_EL3.IRQ | HCR_EL2.IMO | 安全态 | IRQ 目标 |
|-------------|-------------|--------|---------|
| 1 | × | × | **EL3**（SCR 路由覆盖一切）|
| 0 | 1 | NS | **EL2**（HCR 路由） |
| 0 | 0 | NS | **EL1**（默认） |
| 0 | × | Secure | **EL1**（HCR 在安全态无效，除非 SEL2）|

FIQ 和 SError 遵循相同规则（用 SCR_FIQ/FMO 和 SCR_EA/AMO）。

---

## 8. 中断屏蔽与安全状态

```c
// cpu-irq.c:41-165
```

### 8.1 虚拟中断仅在虚拟化激活时有效

```c
// cpu-irq.c:46-83
case EXCP_VINMI:
    if (!(hcr_el2 & HCR_IMO) || (hcr_el2 & HCR_TGE))
        return false;   // 未虚拟化时忽略虚拟中断
    return !allIntMask;

case EXCP_VFIQ:
    if (!(hcr_el2 & HCR_FMO) || (hcr_el2 & HCR_TGE))
        return false;   // VFIQ 仅在 FMO=1 且 TGE=0 时有效

case EXCP_VSERR:
    if (!(hcr_el2 & HCR_AMO) || (hcr_el2 & HCR_TGE))
        return false;
```

### 8.2 目标 EL 高于当前 EL 时的屏蔽规则

```c
// cpu-irq.c:93-161
if ((target_el > cur_el) && (target_el != 1)) {
    if (arm_feature(env, ARM_FEATURE_AARCH64)) {
        switch (target_el) {
        case 2:
            // VHE 模式（E2H+TGE）下可屏蔽，否则不可屏蔽
            if ((hcr_el2 & (HCR_E2H | HCR_TGE)) != (HCR_E2H | HCR_TGE))
                unmasked = true;
            break;
        case 3:
            unmasked = true;   // 目标 EL3 的中断不可屏蔽
            break;
        }
    } else {
        // AArch32：SCR/HCR 影响屏蔽行为
        // FIQ：SCR_FIQ && !(SCR_FW && !HCR_FMO) → 可覆盖 CPSR.F
        if ((scr || hcr) && !secure)
            unmasked = true;
    }
}
```

**关键规则**：
- 路由到 **EL3** 的中断 **永远不可屏蔽**
- 路由到 **EL2** 的中断在非 VHE 模式下不可被 EL1/EL0 屏蔽
- VHE 模式（E2H+TGE）下 EL2 中断可被 PSTATE 屏蔽

---

## 9. SMC 异常路由

```c
// op_helper.c:1111-1200
void HELPER(pre_smc)(CPUARMState *env, uint32_t syndrome)
{
    bool secure = arm_is_secure(env);
    bool smd_flag = env->cp15.scr_el3 & SCR_SMD;

    // AArch64：SMD 对安全态和非安全态都生效
    // AArch32：SMD 仅对非安全态生效
    bool smd = arm_feature(env, ARM_FEATURE_AARCH64) ? smd_flag
                                                     : smd_flag && !secure;

    // ① 无 EL3 且无嵌套虚拟化：SMC 总是 UNDEF
    if (!arm_feature(env, ARM_FEATURE_EL3) && !(arm_hcr_el2_eff(env) & HCR_NV)
        && cpu->psci_conduit != QEMU_PSCI_CONDUIT_SMC)
        raise_exception(... EXCP_UDEF ...);

    // ② NS EL1 + HCR_TSC=1：SMC 陷阱到 EL2（优先于 SMD）
    if (cur_el == 1 && (arm_hcr_el2_eff(env) & HCR_TSC))
        raise_exception(... EXCP_HYP_TRAP ... 2);

    // ③ 非 PSCI 调用 + (SMD=1 或无 EL3)：UNDEF
    if (!arm_is_psci_call(cpu, EXCP_SMC) &&
        (smd || !arm_feature(env, ARM_FEATURE_EL3)))
        raise_exception(... EXCP_UDEF ...);

    // ④ 否则：正常进入 EL3
}
```

### SMC 路由决策树

```
SMC 指令执行
  ├── 无 EL3 且无 HCR_NV → UNDEF
  ├── NS EL1 + HCR_TSC=1 → 陷阱到 EL2
  ├── SCR_SMD=1（非 PSCI）→ UNDEF
  ├── PSCI 调用 → QEMU PSCI 处理
  └── 其他 → 进入 EL3
```

---

## 10. HVC 异常路由

```c
// op_helper.c:1071-1108
void HELPER(pre_hvc)(CPUARMState *env)
{
    // ① 无 EL2：HVC 总是 UNDEF
    if (!arm_feature(env, ARM_FEATURE_EL2))
        undef = true;

    // ② 有 EL3：SCR_HCE 控制 HVC（优先于 HCR_HCD）
    else if (arm_feature(env, ARM_FEATURE_EL3))
        undef = !(env->cp15.scr_el3 & SCR_HCE);

    // ③ 无 EL3：HCR_HCD 控制 HVC
    else
        undef = env->cp15.hcr_el2 & HCR_HCD;

    // ④ 安全态 AArch32 或安全态 EL1：HVC UNDEF
    if (secure && (!is_a64(env) || cur_el == 1))
        undef = true;

    if (undef)
        raise_exception(... EXCP_UDEF ...);
}
```

### HVC 路由决策树

```
HVC 指令执行
  ├── 无 EL2 → UNDEF
  ├── 有 EL3 + SCR_HCE=0 → UNDEF
  ├── 无 EL3 + HCR_HCD=1 → UNDEF
  ├── 安全态 AArch32 → UNDEF
  ├── 安全态 EL1（AArch64）→ UNDEF
  └── 其他 → 进入 EL2
```

**SCR_HCE 优先于 HCR_HCD**：当 EL3 存在时，EL3 通过 SCR_HCE 控制 HVC 是否可用；当 EL3 不存在时，EL2 自行通过 HCR_HCD 控制。

---

## 11. 异常进入时的安全状态处理

```c
// helper.c:9197-9255
static void arm_cpu_do_interrupt_aarch64(CPUState *cs)
{
    unsigned int new_el = env->exception.target_el;
    vaddr addr = env->cp15.vbar_el[new_el];

    // 向量偏移取决于低一级 EL 的执行状态
    if (cur_el < new_el) {
        switch (new_el) {
        case 3:
            // EL3 入口：SCR_EL3.RW 决定低级 EL 是否 AArch64
            is_aa64 = arm_scr_rw_eff(env);
            break;
        case 2:
            // EL2 入口：HCR_EL2.RW 决定 EL1 是否 AArch64
            hcr = arm_hcr_el2_eff(env);
            is_aa64 = (hcr & HCR_RW) != 0;
            break;
        case 1:
            is_aa64 = is_a64(env);  // 当前执行状态
            break;
        }
        addr += is_aa64 ? 0x400 : 0x600;
        // 低级 AArch64 → +0x400，低级 AArch32 → +0x600
    } else {
        if (pstate_read(env) & PSTATE_SP)
            addr += 0x200;  // 使用 SP_ELx → +0x200
    }
}
```

**安全状态在异常进入时的影响**：
1. 进入 EL3 时，CPU 自动切换到 Secure/Root 世界
2. VBAR_EL3 始终在安全世界中
3. 向量偏移由 SCR_EL3.RW 控制（决定从 AArch64 还是 AArch32 进入）

---

## 12. ERET 异常返回时的安全性校验

```c
// helper-a64.c:622-728
void HELPER(exception_return)(CPUARMState *env, uint64_t new_pc)
{
    int cur_el = arm_current_el(env);
    uint64_t spsr = env->banked_spsr[spsr_idx];
    int new_el = el_from_spsr(spsr);

    // ① 不能返回到更高 EL
    if (new_el > cur_el || (new_el == 2 && !arm_is_el2_enabled(env)))
        goto illegal_return;

    // ② RME 安全域校验：EL3 返回低 EL 时
    //    SCR_NS=0, SCR_NSE=1 是保留组合（非法）
    if (cur_el == 3 && new_el < 3 &&
        (env->cp15.scr_el3 & (SCR_NS | SCR_NSE)) == SCR_NSE)
        goto illegal_return;

    // ③ 寄存器宽度匹配：目标 EL 的配置宽度必须匹配 SPSR
    if (new_el != 0 && arm_el_is_aa64(env, new_el) != return_to_aa64)
        goto illegal_return;

    // ④ TGE 检查：HCR_TGE=1 时不能返回到 EL1
    if (new_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE))
        goto illegal_return;

    // ⑤ 执行返回
    if (!return_to_aa64) {
        cpsr_write_from_spsr_elx(env, spsr);
        helper_rebuild_hflags_a32(env, new_el);
    } else {
        pstate_write(env, spsr);
        helper_rebuild_hflags_a64(env, new_el);
    }
}
```

**关键安全检查**：
- EL3 ERET 时，SCR_EL3.{NS, NSE} 决定返回目标的安全域
- 如果 `NSE=1, NS=0`（保留组合），ERET 非法
- 返回后的安全状态由 SCR_EL3 当前值决定，不是由 SPSR 中的值决定

---

## 13. arm_el_is_aa64：寄存器宽度与安全状态

```c
// internals.h:430-483
static inline bool arm_scr_rw_eff(CPUARMState *env)
{
    // SCR_EL3.RW 有效值：考虑 EL2 硬件能力
    if (env->cp15.scr_el3 & SCR_RW) return true;

    if (env->cp15.scr_el3 & SCR_NS) {
        // NS：如果 EL2 存在但不支持 AArch32，RW 有效值为 1
        return arm_feature(env, ARM_FEATURE_EL2) &&
            !cpu_isar_feature(aa64_aa32_el2, cpu);
    } else {
        // Secure：如果 SEL2 使能，RW 有效值为 1
        return env->cp15.scr_el3 & SCR_EEL2;
    }
}

static inline bool arm_el_is_aa64(CPUARMState *env, int el)
{
    bool aa64 = arm_feature(env, ARM_FEATURE_AARCH64);

    if (el == 3) return aa64;  // EL3 宽度固定

    // SCR_EL3.RW 控制 EL2 宽度
    if (arm_feature(env, ARM_FEATURE_EL3))
        aa64 = aa64 && arm_scr_rw_eff(env);

    if (el == 2) return aa64;

    // HCR_EL2.RW 控制 EL1 宽度
    if (arm_is_el2_enabled(env))
        aa64 = aa64 && (env->cp15.hcr_el2 & HCR_RW);

    return aa64;
}
```

**宽度控制链**：
```
EL3 宽度 → 硬件固定（ARM_FEATURE_AARCH64）
EL2 宽度 → SCR_EL3.RW（EL3 控制 EL2）
EL1 宽度 → HCR_EL2.RW（EL2 控制 EL1）
```

---

## 14. Secure EL2（SEL2）支持

```c
// cpu.h:2228-2234
static inline bool arm_is_el2_enabled_secstate(CPUARMState *env,
                                               ARMSecuritySpace space)
{
    assert(space != ARMSS_Root);
    return arm_feature(env, ARM_FEATURE_EL2)
           && (space != ARMSS_Secure || (env->cp15.scr_el3 & SCR_EEL2));
}
```

**SEL2 使能条件**：
- EL2 硬件存在（`ARM_FEATURE_EL2`）
- 非安全态：EL2 总是使能
- 安全态：仅当 `SCR_EEL2=1` 时使能

**SEL2 的影响**：
1. 安全态下 HCR_EL2 变为有效（否则全部为 0）
2. 安全态下 Stage 2 地址翻译可用
3. 安全态下虚拟中断可路由到 EL2

---

## 15. VHE 寄存器重定向

VHE（Virtualization Host Extension）通过 `HCR_EL2.E2H=1` 使能，在 EL2 以 Host 模式运行时将 EL1 系统寄存器重定向到 EL2。

### 15.1 重定向机制

```c
// helper.c 中的 cpreg 定义使用 vhe_redir_to_el2/vhe_redir_to_el01 字段
// 例如 SCTLR_EL1 的 reginfo：
{
    .name = "SCTLR_EL1",
    // ...
    .vhe_redir_to_el2  = ENCODE_AA64_CP_REG(3, 4, ...),  // E2H=1 时 → SCTLR_EL2
    .vhe_redir_to_el01 = ENCODE_AA64_CP_REG(3, 5, ...),  // 访问原始 EL1 → _EL12
}
```

### 15.2 重定向寄存器列表（部分）

| EL1 寄存器 | VHE 模式重定向到 | 访问原始 EL1 通过 |
|-----------|----------------|----------------|
| SCTLR_EL1 | SCTLR_EL2 | SCTLR_EL12 |
| TTBR0_EL1 | TTBR0_EL2 | TTBR0_EL12 |
| TCR_EL1 | TCR_EL2 | TCR_EL12 |
| VBAR_EL1 | VBAR_EL2 | VBAR_EL12 |
| CONTEXTIDR_EL1 | CONTEXTIDR_EL2 | CONTEXTIDR_EL12 |

### 15.3 VHE + TGE 对 HCR 有效值的影响

```c
// helper.c:3862-3878
if (ret & HCR_TGE) {
    if (ret & HCR_E2H) {
        // VHE Host 模式：清除虚拟化陷阱位
        // EL2 直接运行 Host OS，不需要虚拟化陷阱
        ret &= ~(HCR_VM | HCR_FMO | HCR_IMO | HCR_AMO | ...);
    } else {
        // 非 VHE + TGE：强制 FMO/IMO/AMO（所有中断路由到 EL2）
        ret |= HCR_FMO | HCR_IMO | HCR_AMO;
    }
}
```

---

## 16. Stage 2 MMU 与安全状态

```c
// helper.c:3750-3762
// HCR_EL2 写入时，MMU 相关位变化触发 TLB 刷新
if ((env->cp15.hcr_el2 ^ value) &
    (HCR_VM | HCR_PTW | HCR_DC | HCR_DCT | HCR_FWB | HCR_NV | HCR_NV1))
    tlb_flush(CPU(cpu));
```

**Stage 2 与安全状态的关系**：
- `HCR_VM=1`：启用 Stage 2 地址翻译
- `HCR_DC=1`：禁用 Stage 1，启用默认 Cacheable Stage 2
- 安全态下如果 `HCR_EL2 有效值 = 0`（无 SEL2），Stage 2 自动禁用
- 非安全态下 Stage 2 正常由 HCR_VM 控制

---

## 17. 安全状态转换全景图

### 17.1 EL 切换时的安全状态变化

```
EL0/EL1/EL2 (Non-Secure)
    │
    │ SMC 指令（SCR_SMD=0, 非 HCR_TSC 陷阱）
    ▼
EL3 (Root/Secure)
    │
    │ SCR_EL3.NS = 0, ERET
    ▼
EL1/EL2 (Secure)         ←→ HCR_EL2 有效仅当 SCR_EEL2=1
    │
    │ SMC → EL3 → SCR_EL3.NS = 1, ERET
    ▼
EL0/EL1/EL2 (Non-Secure)
```

### 17.2 SCR_EL3 与 HCR_EL2 联动矩阵

```
┌────────────────────────────────────────────────────────────┐
│                     SCR_EL3（EL3 控制）                     │
│                                                            │
│  NS=0, NSE=0 → Secure  ─┐                                │
│  NS=1, NSE=0 → Non-Sec ─┤ arm_security_space_below_el3() │
│  NS=1, NSE=1 → Realm   ─┘                                │
│                                                            │
│  SCR_IRQ/FIQ/EA → EL3 中断路由（最高优先级）                │
│  SCR_SMD → 禁用 SMC                                       │
│  SCR_HCE → 使能 HVC（优先于 HCR_HCD）                      │
│  SCR_RW → EL2 寄存器宽度                                   │
│  SCR_EEL2 → Secure EL2 使能                                │
├────────────────────────────────────────────────────────────┤
│                     HCR_EL2（EL2 控制）                     │
│                                                            │
│  有效值 = arm_hcr_el2_eff()                                │
│    ├── Secure + 无 SEL2 → 返回 0（完全无效）               │
│    ├── TGE + E2H → 清除虚拟化位（VHE Host）                │
│    └── TGE + 非 E2H → 强制 FMO/IMO/AMO                    │
│                                                            │
│  HCR_FMO/IMO/AMO → EL2 中断路由（受 SCR 制约）             │
│  HCR_TSC → SMC 陷阱到 EL2                                  │
│  HCR_HCD → 禁用 HVC（仅无 EL3 时生效）                     │
│  HCR_RW → EL1 寄存器宽度                                   │
│  HCR_VM/DC → Stage 2 MMU 控制                              │
│  HCR_E2H → VHE 寄存器重定向                                │
└────────────────────────────────────────────────────────────┘
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/hw/arm/arm-security.h` | ~37 | ARMSecuritySpace 枚举 (18-23)、arm_space_is_secure (26) |
| `target/arm/cpu.h` | ~2250 | SCR_EL3 位域 (1756-1798)、HCR_EL2 位域 (1695-1755)、arm_is_el2_enabled_secstate (2228-2234) |
| `target/arm/internals.h` | ~490 | arm_scr_rw_eff (431-449)、arm_el_is_aa64 (452-483) |
| `target/arm/helper.c` | ~10190 | scr_write (712-836)、HCR_EL2 写入 (3750-3763)、arm_hcr_el2_eff_secstate (3818-3881)、arm_hcr_el2_eff (3883-3889)、target_el_table (8347-8364)、arm_phys_excp_target_el (8369-8420)、arm_cpu_do_interrupt_aarch64 (9197-9255)、arm_security_space (10131-10161)、arm_security_space_below_el3 (10163-10187) |
| `target/arm/tcg/helper-a64.c` | ~730 | helper_exception_return (622-728) |
| `target/arm/tcg/op_helper.c` | ~1200 | pre_hvc (1071-1108)、pre_smc (1111-1200) |
| `target/arm/cpu-irq.c` | ~165 | 虚拟中断屏蔽 (41-83)、中断屏蔽与安全状态 (93-161) |
