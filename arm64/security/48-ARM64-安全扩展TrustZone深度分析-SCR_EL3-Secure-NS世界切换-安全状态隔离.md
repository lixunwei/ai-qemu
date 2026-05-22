# ARM64 安全扩展（TrustZone）深度分析：SCR_EL3、Secure/NS 世界切换、安全状态隔离

> 基于 QEMU 11.0.50 源码分析，涵盖 ARM TrustZone 安全扩展完整子系统：
> SCR_EL3 寄存器（el3_cp_reginfo 定义 op0=3/opc1=6/crn=1/crm=1/opc2=0 + AArch32 SCR 别名、
> scr_write 42+特性位掩码验证+RES1 强制+SCR_NS|SCR_NSE 变化→EL3 以下全 TLB 刷新 12 个 MMU 索引）、
> SCR_EL3 位定义（SCR_NS bit0 安全状态选择、SCR_IRQ/FIQ/EA bit1-3 中断路由到 EL3、
> SCR_HCE bit8 HVC 使能、SCR_RW bit10 EL2/EL1 AArch64 使能、SCR_EEL2 bit18 Secure EL2 使能、
> SCR_NSE bit62 Realm 状态 RME）、
> 安全状态判定（ARMSecuritySpace 四状态 Secure/NonSecure/Root/Realm、
> arm_security_space 实现：M-profile→v7m.secure、无 EL3→NonSecure、AArch64 EL3→Root(RME)/Secure、
> AArch32 Monitor→Secure、其余→arm_security_space_below_el3 SCR_NS/NSE 解码、
> arm_is_secure/arm_is_secure_below_el3/arm_is_el2_enabled 层次化安全谓词）、
> SMC 世界切换（trans_SMC AArch64 翻译→gen_helper_pre_smc 检查+EXCP_SMC 到 EL3、
> AArch32 EXCP_SMC→ARM_CPU_MODE_MON addr=0x08 Monitor 模式入口、
> PSCI conduit QEMU_PSCI_CONDUIT_SMC/HVC 固件接口路由）、
> 银行状态隔离（bank_fieldoffsets[2] Secure/NS 双银行：SCTLR sctlr_s/ns、
> TTBR0/1 ttbr0_s/ns+ttbr1_s/ns、VBAR vbar_s/ns、MAIR mair0_s/ns、
> CP15 union 结构 sctlr_el[4] 索引访问统一银行）、
> 地址空间隔离（ARMASIdx_NS/S 双地址空间、arm_addressspace attrs.secure→AS 选择、
> cpu_address_space_init 启动时注册 cpu-memory/cpu-secure-memory、
> MemTxAttrs.secure/space 事务级安全属性传播、页表遍历 result.attrs.secure 标记）、
> 中断安全路由（SCR_EL3.IRQ/FIQ/EA 路由到 Monitor/EL3、
> GICv3 安全组 Group0/Group1S/Group1NS 银行、gicv3_use_ns_bank 安全状态选择、
> 非安全不可见 Group0 中断隔离）、
> CPU 复位与 EL3（arm_cpu_reset_hold EL3→PSTATE_MODE_EL3h、RVBAR 采样→PC 初始化）、
> Secure EL2（FEAT_SEL2 SCR_EEL2 使能、arm_is_el2_enabled_secstate 安全 EL2 条件检查）。

---

## 目录

1. [架构概述](#1-架构概述)
2. [SCR_EL3 寄存器定义与位字段](#2-scr_el3-寄存器定义与位字段)
3. [scr_write 写入处理](#3-scr_write-写入处理)
4. [安全状态判定函数](#4-安全状态判定函数)
5. [ARMSecuritySpace 四状态模型](#5-armsecurityspace-四状态模型)
6. [SMC 世界切换](#6-smc-世界切换)
7. [AArch32 中断路由到 Monitor](#7-aarch32-中断路由到-monitor)
8. [AArch64 异常入口与安全状态](#8-aarch64-异常入口与安全状态)
9. [银行寄存器隔离](#9-银行寄存器隔离)
10. [地址空间隔离](#10-地址空间隔离)
11. [内存事务安全属性](#11-内存事务安全属性)
12. [GICv3 安全组与中断隔离](#12-gicv3-安全组与中断隔离)
13. [CPU 复位与 EL3 入口](#13-cpu-复位与-el3-入口)
14. [Secure EL2（FEAT_SEL2）](#14-secure-el2feat_sel2)
15. [完整世界切换流程总结](#15-完整世界切换流程总结)

---

## 1. 架构概述

ARM TrustZone 将系统划分为 **Secure** 和 **Non-Secure** 两个安全世界，通过 EL3（Monitor）作为切换守门人。QEMU 的实现贯穿多个子系统：

```
┌──────────────────────────────────────────────────────┐
│                    EL3 (Monitor)                      │
│  SCR_EL3.NS/NSE 控制下层安全状态                      │
│  SCR_EL3.IRQ/FIQ/EA 控制中断路由                      │
│  SCR_EL3.RW 控制 EL2/EL1 架构宽度                     │
│  SCR_EL3.EEL2 控制 Secure EL2 使能                    │
├──────────────┬───────────────────────────────────────┤
│ Secure World │           Non-Secure World             │
│  S-EL2 (opt) │           NS-EL2                       │
│  S-EL1       │           NS-EL1                       │
│  S-EL0       │           NS-EL0                       │
│              │                                        │
│ secure AS    │        non-secure AS                   │
│ secure TLB   │        non-secure TLB                  │
│ Group0/G1S   │        Group1NS 中断                   │
│ 银行寄存器_s │        银行寄存器_ns                    │
└──────────────┴───────────────────────────────────────┘
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `target/arm/cpu.h` | 1756-1798 | SCR_EL3 位定义 |
| `target/arm/cpu.h` | 2165-2239 | arm_is_secure 系列函数 |
| `target/arm/cpu.h` | 2367-2374 | ARMASIdx 地址空间索引 |
| `target/arm/cpu.h` | 2593-2613 | PSCI conduit / arm_addressspace |
| `target/arm/helper.c` | 712-836 | scr_write 写入处理 |
| `target/arm/helper.c` | 4351-4360 | SCR_EL3 / SCR 寄存器定义 |
| `target/arm/helper.c` | 10131-10187 | arm_security_space 实现 |
| `target/arm/tcg/translate-a64.c` | 3193-3204 | trans_SMC 翻译 |
| `target/arm/helper.c` | 8974-9041 | AArch32 中断/SMC 路由 |
| `target/arm/helper.c` | 9197-9290 | AArch64 异常入口 |
| `target/arm/cpu.c` | 396-413 | arm_cpu_reset_hold EL3 启动 |
| `target/arm/cpu.c` | 2290-2302 | 地址空间初始化 |
| `include/hw/arm/arm-security.h` | 18-35 | ARMSecuritySpace 枚举 |
| `include/exec/memattrs.h` | 25-36 | MemTxAttrs.secure/space |
| `hw/intc/arm_gicv3_cpuif.c` | 40-48 | gicv3_use_ns_bank |

---

## 2. SCR_EL3 寄存器定义与位字段

### 寄存器注册

```c
// target/arm/helper.c:4351-4360
static const ARMCPRegInfo el3_cp_reginfo[] = {
    { .name = "SCR_EL3", .state = ARM_CP_STATE_AA64,      // 4352
      .opc0 = 3, .opc1 = 6, .crn = 1, .crm = 1, .opc2 = 0,
      .access = PL3_RW,
      .fieldoffset = offsetof(CPUARMState, cp15.scr_el3),
      .resetfn = scr_reset,
      .writefn = scr_write, .raw_writefn = raw_write },
    { .name = "SCR", .type = ARM_CP_ALIAS | ARM_CP_NEWEL, // 4356
      .cp = 15, .opc1 = 0, .crn = 1, .crm = 1, .opc2 = 0,
      .access = PL1_RW, .accessfn = access_trap_aa32s_el1,
      .fieldoffset = offsetoflow32(CPUARMState, cp15.scr_el3),
      .writefn = scr_write },
};
```

注意：SCR（AArch32）是 SCR_EL3 的 **别名**（ARM_CP_ALIAS），使用 `offsetoflow32` 访问低 32 位。

### 位字段定义

```c
// target/arm/cpu.h:1756-1798
#define SCR_NS       (1ULL << 0)   // Secure 状态选择（0=Secure, 1=Non-Secure）
#define SCR_IRQ      (1ULL << 1)   // IRQ 路由到 EL3
#define SCR_FIQ      (1ULL << 2)   // FIQ 路由到 EL3
#define SCR_EA       (1ULL << 3)   // 外部中止路由到 EL3
#define SCR_FW       (1ULL << 4)   // CPSR.F 非安全写使能
#define SCR_AW       (1ULL << 5)   // CPSR.A 非安全写使能
#define SCR_NET      (1ULL << 6)   // 未实现（RES0）
#define SCR_SMD      (1ULL << 7)   // SMC 禁用
#define SCR_HCE      (1ULL << 8)   // HVC 使能
#define SCR_SIF      (1ULL << 9)   // 安全指令取指禁止
#define SCR_RW       (1ULL << 10)  // EL2/EL1 使用 AArch64
#define SCR_ST       (1ULL << 11)  // 安全定时器使能
#define SCR_EEL2     (1ULL << 18)  // Secure EL2 使能（FEAT_SEL2）
#define SCR_EASE     (1ULL << 19)  // 外部同步中止→SError 向量（FEAT_DoubleFault）
#define SCR_FGTEN    (1ULL << 27)  // FGT 使能
#define SCR_NSE      (1ULL << 62)  // Realm 状态（FEAT_RME, NS:NSE=1:1→Realm）
```

---

## 3. scr_write 写入处理

```c
// target/arm/helper.c:712-836
static void scr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    uint64_t valid_mask = 0x3fff;   // 715: 基础 v8.0 掩码
    uint64_t changed;

    // === AArch64 EL3 模式 ===
    if (arm_el_is_aa64(env, 3)) {                              // 724
        value |= SCR_FW | SCR_AW;       // RES1 强制           // 725
        valid_mask &= ~SCR_NET;          // RES0 清除           // 726

        // 不支持 AArch32 时 RW 强制为 1
        if (!aa64_aa32_el1 && !aa64_aa32_el2)                  // 728-731
            value |= SCR_RW;

        // 逐特性添加有效位
        if (aa64_sel2)    valid_mask |= SCR_EEL2;              // 741
        if (aa64_rme)     valid_mask |= SCR_NSE | SCR_GPF;    // 765
        if (aa64_fgt)     valid_mask |= SCR_FGTEN;            // 762
        if (aa64_gcs)     valid_mask |= SCR_GCSEN;            // 771
        // ... 共 20+ 特性检查                                  // 732-789
    }
    // === AArch32 模式 ===
    else {
        valid_mask &= ~(SCR_RW | SCR_ST);                      // 791
    }

    // 无 EL2 时禁用 HCE/SMD
    if (!ARM_FEATURE_EL2) valid_mask &= ~SCR_HCE;             // 797-810

    value &= valid_mask;                                        // 814
    changed = env->cp15.scr_el3 ^ value;                       // 815
    env->cp15.scr_el3 = value;                                 // 816

    // === 关键：安全状态切换 → TLB 刷新 ===
    if (changed & (SCR_NS | SCR_NSE)) {                        // 822
        tlb_flush_by_mmuidx(env_cpu(env),                      // 823-834
            ARMMMUIdxBit_E10_0 | ARMMMUIdxBit_E10_0_GCS |
            ARMMMUIdxBit_E20_0 | ARMMMUIdxBit_E20_0_GCS |
            ARMMMUIdxBit_E10_1 | ARMMMUIdxBit_E10_1_PAN |
            ARMMMUIdxBit_E10_1_GCS |
            ARMMMUIdxBit_E20_2 | ARMMMUIdxBit_E20_2_PAN |
            ARMMMUIdxBit_E20_2_GCS |
            ARMMMUIdxBit_E2 | ARMMMUIdxBit_E2_GCS);
        // 刷新 EL3 以下所有 12 个 MMU 索引
        // 因为 Secure 和 NS 使用不同的物理地址空间
    }
}
```

---

## 4. 安全状态判定函数

### 层次化谓词

```c
// target/arm/cpu.h:2165-2239
// 声明层次:
arm_security_space(env)           → ARMSecuritySpace（当前精确状态）
arm_security_space_below_el3(env) → ARMSecuritySpace（EL3 以下）
arm_is_secure(env)                → bool（Secure 或 Root = true）
arm_is_secure_below_el3(env)      → bool（仅 Secure = true）
arm_is_el2_enabled(env)           → bool（当前安全状态是否有 EL2）

// 关键实现:
static inline bool arm_is_secure(CPUARMState *env) {       // 2219-2222
    return arm_space_is_secure(arm_security_space(env));
    // arm_space_is_secure: Secure || Root → true
}

static inline bool arm_is_secure_below_el3(CPUARMState *env) { // 2182-2186
    return arm_security_space_below_el3(env) == ARMSS_Secure;
}

static inline bool arm_is_el2_enabled_secstate(env, space) {   // 2228-2233
    return ARM_FEATURE_EL2
           && (space != ARMSS_Secure || (scr_el3 & SCR_EEL2));
    // Secure 世界需要 SCR_EEL2 才有 EL2
}
```

---

## 5. ARMSecuritySpace 四状态模型

```c
// include/hw/arm/arm-security.h:18-29
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,   // Secure 世界
    ARMSS_NonSecure  = 1,   // Non-Secure 世界
    ARMSS_Root       = 2,   // Root（EL3, FEAT_RME）
    ARMSS_Realm      = 3,   // Realm（FEAT_RME）
} ARMSecuritySpace;

static inline bool arm_space_is_secure(ARMSecuritySpace space) {
    return space == ARMSS_Secure || space == ARMSS_Root;   // 28
}
```

### arm_security_space 实现

```c
// target/arm/helper.c:10131-10161
ARMSecuritySpace arm_security_space(CPUARMState *env)
{
    if (M-profile) return v7m.secure;                      // 10133-10134
    if (!EL3)      return ARMSS_NonSecure;                 // 10141-10143

    // AArch64 + 当前在 EL3
    if (is_a64 && pstate.EL == 3) {
        if (RME) return ARMSS_Root;                        // 10148-10149
        else     return ARMSS_Secure;                      // 10151
    }
    // AArch32 Monitor 模式
    if (!is_a64 && mode == ARM_CPU_MODE_MON)
        return ARMSS_Secure;                               // 10155-10157

    return arm_security_space_below_el3(env);              // 10160
}

// target/arm/helper.c:10163-10187
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env)
{
    if (!EL3) return ARMSS_NonSecure;                      // 10171
    if (!(scr_el3 & SCR_NS)) return ARMSS_Secure;         // 10180
    if (scr_el3 & SCR_NSE)   return ARMSS_Realm;          // 10182
    return ARMSS_NonSecure;                                // 10185
}
```

### SCR_NS / SCR_NSE 解码表

| SCR_NS | SCR_NSE | 安全状态 |
|--------|---------|---------|
| 0 | 0 | Secure |
| 1 | 0 | Non-Secure |
| 0 | 1 | Reserved |
| 1 | 1 | Realm (FEAT_RME) |

---

## 6. SMC 世界切换

### AArch64 翻译

```c
// target/arm/tcg/translate-a64.c:3193-3204
static bool trans_SMC(DisasContext *s, arg_i *a)
{
    if (s->current_el == 0) {                              // 3195
        unallocated_encoding(s);                           // EL0 不能执行 SMC
        return true;
    }
    gen_a64_update_pc(s, 0);                               // 3199
    gen_helper_pre_smc(env, syn_aa64_smc(imm));            // 3200: 检查 SCR_SMD
    gen_ss_advance(s);                                     // 3202: 单步推进
    gen_exception_insn_el(s, 4, EXCP_SMC, syndrome, 3);   // 3203: 异常到 EL3
    return true;
}
```

### AArch32 SMC 异常处理

```c
// target/arm/helper.c:9036-9041
case EXCP_SMC:
    new_mode = ARM_CPU_MODE_MON;   // 进入 Monitor 模式
    addr = 0x08;                    // Monitor 向量偏移
    mask = CPSR_A | CPSR_I | CPSR_F;  // 屏蔽所有中断
    offset = 0;
    break;
```

### PSCI 固件接口

```c
// target/arm/cpu.h:2593-2597
enum {
    QEMU_PSCI_CONDUIT_DISABLED = 0,  // PSCI 禁用
    QEMU_PSCI_CONDUIT_SMC = 1,       // 通过 SMC 调用 PSCI
    QEMU_PSCI_CONDUIT_HVC = 2,       // 通过 HVC 调用 PSCI
};
// target/arm/tcg/psci.c:30-56 — PSCI 调度检查 conduit 类型
```

---

## 7. AArch32 中断路由到 Monitor

```c
// target/arm/helper.c:8974-8995
case EXCP_IRQ:
    new_mode = ARM_CPU_MODE_IRQ;
    addr = 0x18;
    mask = CPSR_A | CPSR_I;
    if (env->cp15.scr_el3 & SCR_IRQ) {          // 8980
        new_mode = ARM_CPU_MODE_MON;             // IRQ 路由到 Monitor
        mask |= CPSR_F;
    }
    break;

case EXCP_FIQ:
    new_mode = ARM_CPU_MODE_FIQ;
    addr = 0x1c;
    mask = CPSR_A | CPSR_I | CPSR_F;
    if (env->cp15.scr_el3 & SCR_FIQ) {          // 8991
        new_mode = ARM_CPU_MODE_MON;             // FIQ 路由到 Monitor
    }
    break;
```

**路由规则**：
- `SCR_EL3.IRQ=1` → 所有 IRQ 先到 Monitor（EL3 处理后决定转发）
- `SCR_EL3.FIQ=1` → 所有 FIQ 先到 Monitor
- `SCR_EL3.EA=1` → 外部中止路由到 EL3

---

## 8. AArch64 异常入口与安全状态

```c
// target/arm/helper.c:9197-9255
static void arm_cpu_do_interrupt_aarch64(CPUState *cs)
{
    unsigned int new_el = env->exception.target_el;       // 9202
    vaddr addr = env->cp15.vbar_el[new_el];               // 9203
    // new_el=3 → 使用 VBAR_EL3 向量基址

    aarch64_sve_change_el(env, cur_el, new_el, ...);      // 9214

    // 向量偏移选择
    if (cur_el < new_el) {                                // 9217
        switch (new_el) {
        case 3:
            is_aa64 = arm_scr_rw_eff(env);               // 9227: SCR_EL3.RW
            break;
        // ...
        }
        if (is_aa64) addr += 0x400;                       // 9243: 低 EL 用 AArch64
        else         addr += 0x600;                       // 9246: 低 EL 用 AArch32
    }

    // SMC 异常处理
    case EXCP_SMC:                                        // 9281
        // syndrome 编码在 env->exception.syndrome
        break;

    // FEAT_DoubleFault: 同步外部中止→SError 向量
    if (new_el == 3 && (scr_el3 & SCR_EASE) &&           // 9268
        syndrome_is_sync_extabt(syndrome))
        addr += 0x180;                                    // 9270
}
```

### 向量表偏移

| 来源 | 目标 EL | 偏移 |
|------|---------|------|
| 当前 EL, SP_EL0 | 当前 | +0x000 |
| 当前 EL, SP_ELx | 当前 | +0x200 |
| 低 EL, AArch64 | 高 | +0x400 |
| 低 EL, AArch32 | 高 | +0x600 |

---

## 9. 银行寄存器隔离

### bank_fieldoffsets 机制

Secure 和 Non-Secure 世界各有独立的寄存器副本，通过 `bank_fieldoffsets[2]` 实现：

```c
// 示例：SCTLR_EL1
{ .name = "SCTLR_EL1",
  .bank_fieldoffsets = {
      offsetof(CPUARMState, cp15.sctlr_s),    // [0] = Secure
      offsetof(CPUARMState, cp15.sctlr_ns)    // [1] = Non-Secure
  }
}
// add_cpreg_to_hashtable 根据 secstate 选择:
// r->fieldoffset = r->bank_fieldoffsets[ns];   // ns=0→Secure, ns=1→NS
```

### 银行寄存器清单

| 寄存器 | Secure 字段 | NS 字段 | 定义位置 |
|--------|------------|---------|---------|
| SCTLR_EL1 | sctlr_s | sctlr_ns | helper.c:7336-7347 |
| TTBR0_EL1 | ttbr0_s | ttbr0_ns | helper.c:2850-2859 |
| TTBR1_EL1 | ttbr1_s | ttbr1_ns | helper.c:2860-2869 |
| VBAR_EL1 | vbar_s | vbar_ns | helper.c:7317-7329 |
| MAIR_EL1 | mair0_s/mair1_s | mair0_ns/mair1_ns | helper.c:990-998 |
| CSSELR | csselr_s | csselr_ns | cpu.h:324-331 |
| CONTEXTIDR | contextidr_s | contextidr_ns | cpu.h:487-495 |
| DFSR/IFSR/DFAR | banked _s | banked _ns | helper.c:2818-2829 |

### CP15 union 结构

```c
// target/arm/cpu.h:332-340
union {
    struct { uint64_t _unused; sctlr_ns; hsctlr; sctlr_s; };
    uint64_t sctlr_el[4];  // [1]=NS, [2]=Hyp, [3]=Secure
};
// 索引访问: sctlr_el[1] == sctlr_ns
// 银行访问: sctlr_s 或 sctlr_ns 直接访问
```

---

## 10. 地址空间隔离

### 双地址空间注册

```c
// target/arm/cpu.h:2367-2374
typedef enum ARMASIdx {
    ARMASIdx_NS = 0,     // Non-Secure 地址空间
    ARMASIdx_S = 1,       // Secure 地址空间
    ARMASIdx_TagNS = 2,   // MTE 标签 NS
    ARMASIdx_TagS = 3,    // MTE 标签 S
} ARMASIdx;

// target/arm/cpu.c:2294-2302
cpu_address_space_init(cs, ARMASIdx_NS, "cpu-memory", cs->memory);
if (has_secure) {
    if (!cpu->secure_memory)
        cpu->secure_memory = cs->memory;    // 默认共享
    cpu_address_space_init(cs, ARMASIdx_S, "cpu-secure-memory",
                           cpu->secure_memory);
}
```

### 地址空间选择

```c
// target/arm/cpu.h:2600-2613
static inline int arm_asidx_from_attrs(CPUState *cs, MemTxAttrs attrs)
{
    return attrs.secure ? ARMASIdx_S : ARMASIdx_NS;       // 2603
}

static inline AddressSpace *arm_addressspace(CPUState *cs, MemTxAttrs attrs)
{
    return cpu_get_address_space(cs, arm_asidx_from_attrs(cs, attrs));
}
```

---

## 11. 内存事务安全属性

### MemTxAttrs

```c
// include/exec/memattrs.h:25-36
typedef struct MemTxAttrs {
    unsigned int secure:1;   // 30: TrustZone 安全访问标记
    unsigned int space:2;    // 36: ARMSecuritySpace（ARMv9 RME）
    unsigned int user:1;     // 38: 用户态（非特权）
    unsigned int memory:1;   // 46: 仅访问内存（非设备）
    unsigned int debug:1;    // 48: 调试访问（可写 ROM）
    // ...
};
```

### 页表遍历中的安全传播

```c
// target/arm/ptw.c — 关键路径
// 页表遍历结果标记安全属性:
result->f.attrs.secure = arm_space_is_secure(out_space);   // 2419-2421
// TLB 条目携带安全标记，确保 Secure/NS 访问不交叉
```

---

## 12. GICv3 安全组与中断隔离

### 安全银行选择

```c
// hw/intc/arm_gicv3_cpuif.c:40-48
static bool gicv3_use_ns_bank(CPUARMState *env)
{
    // GICv3 寄存器按安全状态银行
    // 即使 AArch64 也银行（不同于其他 CP15 寄存器）
    return !arm_is_secure_below_el3(env);                   // 47
}
```

### 中断组模型

```
GICv3 中断组:
  Group 0     → FIQ (安全固件/EL3)
  Group 1 Secure → Secure 世界中断
  Group 1 Non-Secure → Non-Secure 世界中断

路由规则:
  SCR_EL3.IRQ=1 → 所有 IRQ 路由到 EL3
  SCR_EL3.FIQ=1 → 所有 FIQ 路由到 EL3
  Group 0 始终为 FIQ → SCR_EL3.FIQ=1 时路由到 EL3
  Non-Secure 不能看到 Group 0 中断（安全隔离）
```

### 非安全不可见

```c
// hw/intc/arm_gicv3_cpuif.c:2008-2019
// 非安全访问时，Group 0 中断对 NS 世界不可见
// 确保安全中断不会泄露给非安全软件
```

---

## 13. CPU 复位与 EL3 入口

```c
// target/arm/cpu.c:396-413
// arm_cpu_reset_hold:

// 复位到最高可用 EL
if (arm_feature(env, ARM_FEATURE_EL3)) {                   // 403
    env->pstate = PSTATE_MODE_EL3h;                        // 404: EL3 + SP_EL3
} else if (arm_feature(env, ARM_FEATURE_EL2)) {
    env->pstate = PSTATE_MODE_EL2h;                        // 406
} else {
    env->pstate = PSTATE_MODE_EL1h;                        // 408
}

// 采样 RVBAR → PC
env->cp15.rvbar = cpu->rvbar_prop;                         // 412
env->pc = env->cp15.rvbar;                                 // 413
// 固件（如 ARM Trusted Firmware）从 RVBAR 地址开始执行

// AArch32 v8 复位:
if (arm_feature(env, ARM_FEATURE_V8)) {                    // 423
    env->cp15.rvbar = cpu->rvbar_prop;                     // 424
    env->regs[15] = cpu->rvbar_prop;                       // 425
}
```

**复位序列**：
1. CPU 进入 EL3h（最高特权）
2. PC 设为 RVBAR（复位向量基址）
3. 安全固件（ATF/TF-A）初始化安全世界
4. 配置 SCR_EL3（设置 NS=1 切换到 NS）
5. ERET 进入 Non-Secure 世界启动操作系统

---

## 14. Secure EL2（FEAT_SEL2）

```c
// target/arm/cpu.h:1774
#define SCR_EEL2  (1ULL << 18)   // Secure EL2 使能

// target/arm/cpu.h:2228-2233
static inline bool arm_is_el2_enabled_secstate(env, space)
{
    assert(space != ARMSS_Root);
    return arm_feature(env, ARM_FEATURE_EL2)
           && (space != ARMSS_Secure || (scr_el3 & SCR_EEL2));
    // Non-Secure: EL2 总是使能（如果存在）
    // Secure: 需要 SCR_EEL2=1 才使能 Secure EL2
}
```

**Secure EL2 用途**：
- 允许在 Secure 世界运行 Hypervisor
- 隔离多个 Secure 分区（如多个 Trusted OS）
- 安全 Stage-2 翻译：Secure EL2 可以控制 Secure EL1 的物理地址映射

---

## 15. 完整世界切换流程总结

```
=== Non-Secure EL1 → Secure EL3 (SMC) ===

1. Guest 执行 SMC #0
   trans_SMC (translate-a64.c:3193)
     → gen_helper_pre_smc: 检查 SCR_SMD 未禁用
     → gen_exception_insn_el(EXCP_SMC, target_el=3)

2. 异常入口 (helper.c:9197)
   arm_cpu_do_interrupt_aarch64:
     new_el = 3
     addr = VBAR_EL3 + 0x400  (来自低 EL AArch64)
     保存: SPSR_EL3 = PSTATE, ELR_EL3 = PC
     设置: PSTATE.EL=3, PSTATE.SP=1, 屏蔽中断
     PC = VBAR_EL3 + 0x400 + 异常类型偏移

3. EL3 固件处理
   读取 SMC 参数 (X0-X7)
   执行安全服务...

=== Secure EL3 → Non-Secure EL1 (ERET) ===

4. EL3 准备返回
   SCR_EL3.NS = 1  (切换到 NS)
     → scr_write (helper.c:712)
     → SCR_NS 变化 → tlb_flush_by_mmuidx (12 个索引)
   设置 SPSR_EL3 = NS EL1 的 PSTATE
   设置 ELR_EL3 = NS EL1 返回地址

5. ERET 执行
   HELPER(exception_return) (helper-a64.c:622)
     恢复 PSTATE (从 SPSR_EL3)
     PC = ELR_EL3
     arm_rebuild_hflags: 重建翻译标志
     现在 arm_security_space_below_el3 = ARMSS_NonSecure
     后续内存访问使用 NS 地址空间

=== 状态隔离 ===
   银行寄存器: SCTLR_ns, TTBR0_ns, VBAR_ns 自动激活
   地址空间: arm_addressspace → ARMASIdx_NS
   TLB: 已刷新，重建 NS TLB 条目
   GIC: gicv3_use_ns_bank → NS 银行，Group0 不可见
```

---

## 交叉参考

- [47-ARM64-系统寄存器与CP访问深度分析](47-ARM64-系统寄存器与CP访问深度分析-ARMCPRegInfo框架-MRS-MSR翻译-cpregs哈希表-EL银行与访问控制.md) — ARMCPRegInfo bank_fieldoffsets 银行机制详解
- [44-ARM64-TCG执行循环深度分析](44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md) — 异常/中断处理主循环
- [43-ARM64-TCG-softmmu-TLB深度分析](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — TLB 刷新与安全属性

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
