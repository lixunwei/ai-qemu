# ARM64 异常处理完整流程深度分析

## 同步/异步异常入口、VBAR 向量表、PSTATE 保存恢复、异常返回

> 基于 QEMU 11.0.50 源码分析，涵盖异常入口完整流程（同步/IRQ/FIQ/SError）、VBAR 向量偏移计算、PSTATE→SPSR 保存与恢复、目标 EL 路由、SVC/HVC/SMC 生成、ERET 异常返回、中断 unmasking 与注入机制。

---

## 目录

1. [异常处理总体架构](#1-异常处理总体架构)
2. [异常类型与 EC 编码](#2-异常类型与-ec-编码)
3. [目标 EL 路由机制](#3-目标-el-路由机制)
4. [异常入口主函数 arm_cpu_do_interrupt](#4-异常入口主函数-arm_cpu_do_interrupt)
5. [AArch64 异常入口 arm_cpu_do_interrupt_aarch64](#5-aarch64-异常入口-arm_cpu_do_interrupt_aarch64)
6. [VBAR 向量表与偏移计算](#6-vbar-向量表与偏移计算)
7. [ESR_ELx 综合征写入](#7-esr_elx-综合征写入)
8. [PSTATE 保存与新 PSTATE 设置](#8-pstate-保存与新-pstate-设置)
9. [SP 切换机制](#9-sp-切换机制)
10. [中断注入与 unmasking](#10-中断注入与-unmasking)
11. [SVC/HVC/SMC 异常生成](#11-svchvcsmc-异常生成)
12. [ERET 异常返回](#12-eret-异常返回)
13. [非法异常返回处理](#13-非法异常返回处理)
14. [PSTATE 位定义与缓存](#14-pstate-位定义与缓存)
15. [总结与关键设计](#15-总结与关键设计)

---

## 1. 异常处理总体架构

QEMU ARM64 异常处理遵循 ARM 架构规范，核心流程：

```
指令执行 → 异常触发 → 确定目标 EL → VBAR+偏移计算 → 保存 PSTATE/PC → 设置新 PSTATE → 跳转异常向量
                                                                                    ↓
异常处理程序执行 ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ←
                                                                                    ↓
ERET ← 恢复 SPSR→PSTATE、ELR→PC → 返回原执行流
```

**异常分类**：
- **同步异常**：SVC/HVC/SMC、数据/指令中止（Data/Instruction Abort）、对齐错误、非法状态、调试异常
- **异步异常**：IRQ、FIQ、SError（System Error）
- **虚拟异常**：VIRQ、VFIQ、VSERR（由 HCR_EL2 注入）
- **NMI**：不可屏蔽中断（FEAT_NMI），VINMI、VFNMI

**入口调用链**：
```
TCG 执行循环 → cpu_handle_interrupt/cpu_handle_exception
  → arm_cpu_exec_interrupt()           [cpu-irq.c:171]     — 异步中断检测
  → arm_cpu_do_interrupt()             [helper.c:9469]     — 异常入口主分发
    → arm_cpu_do_interrupt_aarch64()   [helper.c:9198]     — AArch64 路径
    → arm_cpu_do_interrupt_aarch32()   [helper.c:8875]     — AArch32 路径
```

---

## 2. 异常类型与 EC 编码

### 2.1 Exception Class (EC) 枚举

**文件**：`target/arm/syndrome.h:31-77`

```c
enum arm_exception_class {
    EC_UNCATEGORIZED          = 0x00,   // 未分类异常
    EC_WFX_TRAP               = 0x01,   // WFI/WFE 陷入
    EC_CP15RTTRAP             = 0x03,   // CP15 MCR/MRC 陷入
    EC_ADVSIMDFPACCESSTRAP    = 0x07,   // FP/SIMD 访问陷入
    EC_ILLEGALSTATE           = 0x0e,   // 非法执行状态
    EC_AA64_SVC               = 0x15,   // AArch64 SVC
    EC_AA64_HVC               = 0x16,   // AArch64 HVC
    EC_AA64_SMC               = 0x17,   // AArch64 SMC
    EC_SYSTEMREGISTERTRAP     = 0x18,   // 系统寄存器陷入
    EC_SVEACCESSTRAP          = 0x19,   // SVE 访问陷入
    EC_ERETTRAP               = 0x1a,   // ERET 陷入
    EC_GPC                    = 0x1e,   // Granule Protection Check
    EC_INSNABORT              = 0x20,   // 指令中止（低 EL）
    EC_INSNABORT_SAME_EL      = 0x21,   // 指令中止（同 EL）
    EC_DATAABORT              = 0x24,   // 数据中止（低 EL）
    EC_DATAABORT_SAME_EL      = 0x25,   // 数据中止（同 EL）
    EC_SPALIGNMENT            = 0x26,   // SP 对齐异常
    EC_AA64_FPTRAP            = 0x2c,   // AArch64 FP 陷入
    EC_SERROR                 = 0x2f,   // SError
    EC_BREAKPOINT             = 0x30,   // 断点（低 EL）
    EC_SOFTWARESTEP           = 0x32,   // 软件单步（低 EL）
    EC_WATCHPOINT             = 0x34,   // 观察点（低 EL）
    EC_AA64_BKPT              = 0x3c,   // AArch64 BRK
};
```

### 2.2 综合征构造函数

**文件**：`target/arm/syndrome.h:138-160`

```c
// SVC: EC=0x15, IL=1, imm16=立即数
static inline uint32_t syn_aa64_svc(uint32_t imm16) {
    return syn_set_ec(0, EC_AA64_SVC) | IL | imm16;
}

// HVC: EC=0x16, IL=1, imm16=立即数
static inline uint32_t syn_aa64_hvc(uint32_t imm16) {
    return syn_set_ec(0, EC_AA64_HVC) | IL | imm16;
}

// SMC: EC=0x17, IL=1, imm16=立即数
static inline uint32_t syn_aa64_smc(uint32_t imm16) {
    return syn_set_ec(0, EC_AA64_SMC) | IL | imm16;
}
```

**综合征格式** (ESR_ELx)：
```
[31:26] EC    — Exception Class，标识异常类型
[25]    IL    — Instruction Length（0=16位，1=32位）
[24:0]  ISS   — Instruction Specific Syndrome，异常特定信息
```

---

## 3. 目标 EL 路由机制

### 3.1 物理异常目标 EL 查表

**文件**：`target/arm/helper.c:8309-8364`

ARM 架构用 6 维查找表确定物理异常（IRQ/FIQ/SError）的目标 EL：

```c
static const int8_t target_el_table[2][2][2][2][2][4] = {
    // 维度: [64bit_EL3][SCR_EA/IRQ/FIQ][RW][HCR_AMO/IMO/FMO][Secure][cur_EL]
    // 值: 0-3=EL0-EL3, -1=不可能
    ...
};
```

**维度含义**：
| 维度 | 含义 | 索引来源 |
|------|------|----------|
| [0] | EL3 是否 64 位 | `arm_feature(ARM_FEATURE_AARCH64)` |
| [1] | SCR 路由位 | `SCR_IRQ`/`SCR_FIQ`/`SCR_EA` |
| [2] | RW（低 EL 执行状态） | `arm_scr_rw_eff()` |
| [3] | HCR 路由位 | `HCR_IMO`/`HCR_FMO`/`HCR_AMO` + `HCR_TGE` |
| [4] | 安全状态 | `arm_is_secure()` |
| [5] | 当前 EL | `arm_current_el()` |

### 3.2 arm_phys_excp_target_el

**文件**：`target/arm/helper.c:8369-8421`

```c
uint32_t arm_phys_excp_target_el(CPUState *cs, uint32_t excp_idx,
                                 uint32_t cur_el, bool secure)
{
    // 根据异常类型选择 SCR/HCR 路由位
    switch (excp_idx) {
    case EXCP_IRQ:
    case EXCP_NMI:
        scr = (env->cp15.scr_el3 & SCR_IRQ);   // SCR_EL3.IRQ
        hcr = hcr_el2 & HCR_IMO;               // HCR_EL2.IMO
        break;
    case EXCP_FIQ:
        scr = (env->cp15.scr_el3 & SCR_FIQ);   // SCR_EL3.FIQ
        hcr = hcr_el2 & HCR_FMO;               // HCR_EL2.FMO
        break;
    default: // SError
        scr = (env->cp15.scr_el3 & SCR_EA);    // SCR_EL3.EA
        hcr = hcr_el2 & HCR_AMO;               // HCR_EL2.AMO
    }
    hcr |= (hcr_el2 & HCR_TGE) != 0;          // TGE 也强制路由 EL2

    target_el = target_el_table[is64][scr][rw][hcr][secure][cur_el];
    return target_el;
}
```

### 3.3 同步异常目标 EL

**文件**：`target/arm/tcg/op_helper.c:32-45`

```c
int exception_target_el(CPUARMState *env) {
    int target_el = MAX(1, arm_current_el(env));  // 至少 EL1
    // Secure EL1 + 32位 EL3 → 路由到 EL3
    if (arm_is_secure(env) && !arm_el_is_aa64(env, 3) && target_el == 1)
        target_el = 3;
    return target_el;
}
```

**路由规则总结**：
| 异常类型 | 默认目标 EL | 路由条件 |
|----------|-------------|----------|
| SVC | EL1 | 始终（EL0→EL1） |
| HVC | EL2 | EL1→EL2，EL3→EL3 |
| SMC | EL3 | 始终 |
| IRQ | EL1 | SCR.IRQ→EL3，HCR.IMO→EL2 |
| FIQ | EL1 | SCR.FIQ→EL3，HCR.FMO→EL2 |
| SError | EL1 | SCR.EA→EL3，HCR.AMO→EL2 |
| Data/Insn Abort | ≥cur_el | 取决于 Stage-2 fault 或 HCR 配置 |

---

## 4. 异常入口主函数 arm_cpu_do_interrupt

**文件**：`target/arm/helper.c:9469-9530`

```c
void arm_cpu_do_interrupt(CPUState *cs) {
    unsigned int new_el = env->exception.target_el;  // 预先确定的目标 EL

    // 1. PSCI 调用拦截（SMC 路径用于电源管理）
    if (tcg_enabled() && arm_is_psci_call(cpu, cs->exception_index)) {
        arm_handle_psci_call(cpu);
        return;
    }

    // 2. 半主机调用拦截
    if (cs->exception_index == EXCP_SEMIHOST) {
        tcg_handle_semihosting(cs);
        return;
    }

    // 3. EL 切换前钩子
    arm_call_pre_el_change_hook(cpu);

    // 4. 根据目标 EL 的执行状态分发
    if (arm_el_is_aa64(env, new_el)) {
        arm_cpu_do_interrupt_aarch64(cs);   // AArch64 路径
    } else {
        arm_cpu_do_interrupt_aarch32(cs);   // AArch32 路径
    }

    // 5. EL 切换后钩子
    arm_call_el_change_hook(cpu);

    // 6. 强制退出当前 TB（确保中断响应）
    if (!kvm_enabled())
        cpu_set_interrupt(cs, CPU_INTERRUPT_EXITTB);
}
```

**关键设计**：
- `arm_el_is_aa64(env, new_el)` 检查目标 EL 的执行状态，不是当前状态
- 即使从 AArch32 状态触发异常，如果目标 EL 是 AArch64，也走 `_aarch64` 路径
- `pre_el_change_hook` / `el_change_hook` 用于通知 GICv3 等组件 EL 变化

---

## 5. AArch64 异常入口 arm_cpu_do_interrupt_aarch64

**文件**：`target/arm/helper.c:9198-9428`

这是 AArch64 异常处理的核心函数，完整流程：

### 5.1 初始化

```c
static void arm_cpu_do_interrupt_aarch64(CPUState *cs) {
    unsigned int new_el = env->exception.target_el;
    vaddr addr = env->cp15.vbar_el[new_el];           // VBAR 基地址
    uint64_t new_mode = aarch64_pstate_mode(new_el, true); // (new_el << 2) | 1 = ELxh
    unsigned int cur_el = arm_current_el(env);

    // SVE 向量长度切换
    aarch64_sve_change_el(env, cur_el, new_el, is_a64(env));
```

### 5.2 VBAR 偏移计算

```c
    if (cur_el < new_el) {
        // 从低 EL 进入高 EL
        // 判断低 EL 的执行状态
        bool is_aa64;
        switch (new_el) {
        case 3: is_aa64 = arm_scr_rw_eff(env); break;
        case 2: is_aa64 = (hcr & HCR_RW); break;   // 简化
        case 1: is_aa64 = is_a64(env); break;
        }
        if (is_aa64)
            addr += 0x400;    // 低 EL 是 AArch64: 偏移 0x400
        else
            addr += 0x600;    // 低 EL 是 AArch32: 偏移 0x600
    } else {
        // 同 EL 异常
        if (pstate_read(env) & PSTATE_SP)
            addr += 0x200;    // 使用 SP_ELx: 偏移 0x200
        // 否则使用 SP_EL0: 偏移 0x000
    }
```

### 5.3 异常类型偏移

```c
    switch (cs->exception_index) {
    case EXCP_SWI:  // SVC
    case EXCP_HVC:
    case EXCP_SMC:
    case EXCP_HYP_TRAP:
        // 同步异常: 偏移 +0x000（已包含在基础偏移中）
        env->cp15.esr_el[new_el] = env->exception.syndrome;
        break;
    case EXCP_IRQ: case EXCP_VIRQ: case EXCP_NMI: case EXCP_VINMI:
        addr += 0x80;     // IRQ 类: +0x80
        break;
    case EXCP_FIQ: case EXCP_VFIQ: case EXCP_VFNMI:
        addr += 0x100;    // FIQ 类: +0x100
        break;
    case EXCP_VSERR:
        addr += 0x180;    // SError: +0x180
        env->cp15.esr_el[new_el] = syn_serror(env->cp15.vsesr_el2 & 0x1ffffff);
        break;
    }
```

### 5.4 PSTATE/ELR 保存

```c
    if (is_a64(env)) {
        old_mode = pstate_read(env);        // 读取完整 PSTATE
        aarch64_save_sp(env, cur_el);       // 保存当前 SP
        env->elr_el[new_el] = env->pc;     // PC → ELR_ELx
        // FEAT_NV: 可能修改 SPSR 中的 EL 字段
    } else {
        old_mode = cpsr_read_for_spsr_elx(env);  // AArch32 CPSR
        env->elr_el[new_el] = env->regs[15];     // R15 → ELR_ELx
        aarch64_sync_32_to_64(env);               // 32→64 位寄存器同步
    }
    env->banked_spsr[aarch64_banked_spsr_index(new_el)] = old_mode;  // → SPSR_ELx
```

### 5.5 新 PSTATE 设置

```c
    // PAN 保留（FEAT_PAN）
    new_mode |= old_mode & PSTATE_PAN;
    if (new_el == 1 && !(sctlr_el[1] & SCTLR_SPAN))
        new_mode |= PSTATE_PAN;    // SPAN=0 → 异常入口强制 PAN=1

    // MTE: 设置 TCO=1
    if (cpu_isar_feature(aa64_mte, cpu))
        new_mode |= PSTATE_TCO;

    // SSBS: 根据 SCTLR.DSSBS 设置
    if (sctlr_el[new_el] & SCTLR_DSSBS_64)
        new_mode |= PSTATE_SSBS;

    // NMI: ALLINT 根据 SCTLR.SPINTMASK
    if (!(sctlr_el[new_el] & SCTLR_SPINTMASK))
        new_mode |= PSTATE_ALLINT;

    // 写入新 PSTATE: DAIF 全部置位（屏蔽所有异常） + 新模式
    pstate_write(env, PSTATE_DAIF | new_mode);
    env->aarch64 = true;
    aarch64_restore_sp(env, new_el);   // 切换到 SP_ELx

    arm_rebuild_hflags(env);           // 重建翻译标志
    env->pc = addr;                    // 跳转到异常向量
```

---

## 6. VBAR 向量表与偏移计算

### 6.1 VBAR 寄存器定义

**文件**：`target/arm/helper.c`

| 寄存器 | cpreg 定义行 | 存储位置 |
|--------|-------------|----------|
| VBAR_EL1 | helper.c:7317-7332 | `env->cp15.vbar_el[1]` |
| VBAR_EL2 | helper.c:4106-4110 | `env->cp15.vbar_el[2]` |
| VBAR_EL3 | helper.c:4399-4403 | `env->cp15.vbar_el[3]` |

**写入约束**（`vbar_write`, helper.c:699-710）：
```c
static void vbar_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value) {
    raw_write(env, ri, value & ~0x1FULL);  // 低 5 位 RAZ/WI（32 字节对齐）
}
```

### 6.2 完整向量偏移表

VBAR_ELn 基地址 + 两级偏移 = 最终向量地址：

| 来源条件 | 基础偏移 | Sync +0x000 | IRQ +0x080 | FIQ +0x100 | SError +0x180 |
|----------|----------|-------------|------------|------------|---------------|
| 同 EL，SP_EL0 (SPSel=0) | +0x000 | 0x000 | 0x080 | 0x100 | 0x180 |
| 同 EL，SP_ELx (SPSel=1) | +0x200 | 0x200 | 0x280 | 0x300 | 0x380 |
| 低 EL，AArch64 | +0x400 | 0x400 | 0x480 | 0x500 | 0x580 |
| 低 EL，AArch32 | +0x600 | 0x600 | 0x680 | 0x700 | 0x780 |

**每个向量入口 128 字节（0x80）**，共 16 个入口，总表大小 2048 字节（0x800）。

### 6.3 偏移计算代码对应

**文件**：`target/arm/helper.c`

```
基础偏移选择:
  cur_el < new_el:
    AArch64 低 EL → +0x400    [helper.c:9244]
    AArch32 低 EL → +0x600    [helper.c:9246]
  cur_el >= new_el:
    PSTATE.SP=1   → +0x200    [helper.c:9250]
    PSTATE.SP=0   → +0x000    (无额外偏移)

异常类型偏移:
  Sync (SVC/HVC/SMC/Abort/Trap) → +0x000   [helper.c:9257-9321]
  IRQ/VIRQ/NMI/VINMI           → +0x080   [helper.c:9322-9327]
  FIQ/VFIQ/VFNMI               → +0x100   [helper.c:9328-9332]
  SError/VSERR                  → +0x180   [helper.c:9333-9338]
```

---

## 7. ESR_ELx 综合征写入

### 7.1 同步异常

**文件**：`target/arm/helper.c:9257-9321`

同步异常在入口时写入 ESR_ELx：

```c
case EXCP_SWI:     // SVC
case EXCP_HVC:     // HVC
case EXCP_SMC:     // SMC
case EXCP_HYP_TRAP: // 各种陷入
    // AArch32→AArch64 的寄存器编号转换（4位→5位）
    switch (syn_get_ec(env->exception.syndrome)) {
    case EC_CP14RTTRAP:
    case EC_CP15RTTRAP:
        rt = extract32(syndrome, 5, 4);
        rt = aarch64_regnum(env, rt);      // 映射到 64 位寄存器号
        syndrome = deposit32(syndrome, 5, 5, rt);
        break;
    }
    env->cp15.esr_el[new_el] = env->exception.syndrome;  // → ESR_ELx
    break;
```

### 7.2 异步异常

IRQ/FIQ 不写入 ESR（无综合征）。SError 写入：

```c
case EXCP_VSERR:
    env->exception.syndrome = syn_serror(env->cp15.vsesr_el2 & 0x1ffffff);
    env->cp15.esr_el[new_el] = env->exception.syndrome;
    break;
```

---

## 8. PSTATE 保存与新 PSTATE 设置

### 8.1 PSTATE → SPSR_ELx 保存

**保存路径**（helper.c:9343-9369）：

```
AArch64 来源:
  old_mode = pstate_read(env)     →  [NZCV|DAIF|BTYPE|SS|IL|PAN|UAO|DIT|TCO|SSBS|ALLINT|EXLOCK|M[4:0]]
  env->banked_spsr[index] = old_mode

AArch32 来源:
  old_mode = cpsr_read_for_spsr_elx(env)  →  [NZCV|Q|IT|GE|E|A|I|F|T|M[4:0]]
  aarch64_sync_32_to_64(env)              →  32 位寄存器映射到 64 位
  env->banked_spsr[index] = old_mode
```

### 8.2 新 PSTATE 构造

异常入口后新 PSTATE（helper.c:9375-9415）：

| 位 | 值 | 来源/条件 |
|----|----|-----------|
| M[3:0] | new_el<<2 \| 1 | ELxh 模式（SP=SP_ELx） |
| nRW | 0 | AArch64 |
| DAIF | 1111 | 全部屏蔽 |
| PAN | 保留或强制1 | SCTLR.SPAN=0 时强制 PAN=1 |
| TCO | 1 | FEAT_MTE 启用时 |
| SSBS | SCTLR.DSSBS | 由 SCTLR 控制 |
| ALLINT | 0 或 1 | SCTLR.SPINTMASK=0 时设置 |
| SS | 0 | 清除单步 |
| IL | 0 | 清除非法状态 |
| BTYPE | 0 | 清除分支类型 |

### 8.3 pstate_read / pstate_write

**文件**：`target/arm/cpu.h:1607-1626`

```c
static inline uint64_t pstate_read(CPUARMState *env) {
    int ZF = (env->ZF == 0);
    return (env->NF & 0x80000000) | (ZF << 30)
        | (env->CF << 29) | ((env->VF & 0x80000000) >> 3)
        | env->pstate | env->daif | (env->btype << 10);
}

static inline void pstate_write(CPUARMState *env, uint64_t val) {
    env->ZF = (~val) & PSTATE_Z;
    env->NF = val;
    env->CF = (val >> 29) & 1;
    env->VF = (val << 3) & 0x80000000;
    env->daif = val & PSTATE_DAIF;
    env->btype = (val >> 10) & 3;
    env->pstate = val & ~CACHED_PSTATE_BITS;
}
```

**缓存位分离设计**：NZCV/DAIF/BTYPE 是 "热" 位（频繁访问），单独存储在 `env->NF/ZF/CF/VF/daif/btype` 中，避免每次位操作。`CACHED_PSTATE_BITS = NZCV | DAIF | BTYPE`。其余 PSTATE 位存储在 `env->pstate` 中。

---

## 9. SP 切换机制

### 9.1 aarch64_save_sp / aarch64_restore_sp

**文件**：`target/arm/internals.h:565-581`

```c
static inline void aarch64_save_sp(CPUARMState *env, int el) {
    if (env->pstate & PSTATE_SP)
        env->sp_el[el] = env->xregs[31];   // SP_ELx ← X31
    else
        env->sp_el[0] = env->xregs[31];    // SP_EL0 ← X31
}

static inline void aarch64_restore_sp(CPUARMState *env, int el) {
    if (env->pstate & PSTATE_SP)
        env->xregs[31] = env->sp_el[el];   // X31 ← SP_ELx
    else
        env->xregs[31] = env->sp_el[0];    // X31 ← SP_EL0
}
```

**异常入口的 SP 切换流程**：
1. `aarch64_save_sp(env, cur_el)` — 保存当前 X31 到对应 SP_ELx 或 SP_EL0
2. `pstate_write(env, ... | SP=1)` — 新 PSTATE 设置 SP=1（使用 SP_ELx）
3. `aarch64_restore_sp(env, new_el)` — 从 SP_ELx 加载到 X31

### 9.2 PSTATE.SP (SPSel) 含义

```
PSTATE.SP = 0 (PSTATE_MODE_ELxt): 使用 SP_EL0 作为堆栈指针
PSTATE.SP = 1 (PSTATE_MODE_ELxh): 使用 SP_ELx 作为堆栈指针
```

异常入口始终设置 `SP=1`（`aarch64_pstate_mode(new_el, true) = (new_el << 2) | 1`），即使用目标 EL 自己的栈指针。

---

## 10. 中断注入与 unmasking

### 10.1 arm_cpu_exec_interrupt

**文件**：`target/arm/cpu-irq.c:171-270`

TCG 执行循环在每个 TB 边界调用此函数检查待处理中断：

```c
bool arm_cpu_exec_interrupt(CPUState *cs, int interrupt_request) {
    // 优先级顺序（IMPLEMENTATION DEFINED）:
    // 1. NMI        (CPU_INTERRUPT_NMI)    → EXCP_NMI
    // 2. VINMI      (CPU_INTERRUPT_VINMI)  → EXCP_VINMI
    // 3. VFNMI      (CPU_INTERRUPT_VFNMI)  → EXCP_VFNMI
    // 4. FIQ        (CPU_INTERRUPT_FIQ)    → EXCP_FIQ
    // 5. IRQ        (CPU_INTERRUPT_HARD)   → EXCP_IRQ
    // 6. VIRQ       (CPU_INTERRUPT_VIRQ)   → EXCP_VIRQ
    // 7. VFIQ       (CPU_INTERRUPT_VFIQ)   → EXCP_VFIQ
    // 8. VSERR      (CPU_INTERRUPT_VSERR)  → EXCP_VSERR

    // 对每种中断:
    //   1. 确定 target_el (物理异常查表/虚拟异常固定 EL1)
    //   2. 检查 arm_excp_unmasked() 是否未被屏蔽
    //   3. 如果通过 → 设置 exception_index + target_el → 返回 true

    // NMI 未启用时，NMI→IRQ，VINMI→VIRQ，VFNMI→VFIQ（降级）
}
```

### 10.2 arm_excp_unmasked

**文件**：`target/arm/cpu-irq.c:15-169`

```c
static inline bool arm_excp_unmasked(CPUState *cs, unsigned int excp_idx,
                                     unsigned int target_el, ...) {
    // 规则 1: target_el < cur_el → 不响应（异常不能向下路由）
    if (cur_el > target_el) return false;

    // 规则 2: NMI/ALLINT 检查
    // FEAT_NMI: ALLINT 位控制超优先级中断屏蔽
    // SCTLR.SPINTMASK + PSTATE.SP 也参与

    // 规则 3: DAIF 屏蔽
    // IRQ: !(PSTATE.I) && !allIntMask
    // FIQ: !(PSTATE.F) && !allIntMask
    // NMI: !allIntMask (不受 DAIF 影响)

    // 规则 4: 虚拟异常前提条件
    // VIRQ: 需要 HCR.IMO && !HCR.TGE
    // VFIQ: 需要 HCR.FMO && !HCR.TGE
    // VSERR: 需要 HCR.AMO && !HCR.TGE

    // 规则 5: 高 EL 异常不可屏蔽
    // target_el > cur_el && target_el != 1 → 不可屏蔽
    // 例外: EL2+E2H+TGE 时仍可屏蔽
    // target_el == 3 → 绝对不可屏蔽
}
```

### 10.3 虚拟 SError 特殊处理

```c
if (interrupt_request & CPU_INTERRUPT_VSERR) {
    ...
    // 响应虚拟 SError 时:
    env->cp15.hcr_el2 &= ~HCR_VSE;        // 清除 HCR_EL2.VSE
    cpu_reset_interrupt(cs, CPU_INTERRUPT_VSERR);  // 清除中断请求
    goto found;
}
```

---

## 11. SVC/HVC/SMC 异常生成

### 11.1 翻译阶段

**文件**：`target/arm/tcg/translate-a64.c:3155-3204`

```c
static bool trans_SVC(DisasContext *s, arg_i *a) {
    uint32_t syndrome = syn_aa64_svc(a->imm);  // EC=0x15 + imm16
    gen_ss_advance(s);                           // 先推进单步状态机
    gen_exception_insn(s, 4, EXCP_SWI, syndrome);
    return true;
}

static bool trans_HVC(DisasContext *s, arg_i *a) {
    int target_el = s->current_el == 3 ? 3 : 2;  // 目标 EL
    gen_helper_pre_hvc(tcg_env);                   // 预检查（HCR_EL2.HCD 陷入）
    gen_ss_advance(s);
    gen_exception_insn_el(s, 4, EXCP_HVC, syn_aa64_hvc(a->imm), target_el);
    return true;
}

static bool trans_SMC(DisasContext *s, arg_i *a) {
    gen_helper_pre_smc(tcg_env, ...);              // 预检查（SCR_EL3.SMD 禁用/EL2 陷入）
    gen_ss_advance(s);
    gen_exception_insn_el(s, 4, EXCP_SMC, syn_aa64_smc(a->imm), 3);
    return true;
}
```

**关键设计**：
- `gen_ss_advance(s)` 在异常前推进单步状态机（ARM 架构要求）
- HVC/SMC 有 `pre_` helper 进行预检查（可能陷入 EL2 或被禁用）
- SVC 可能被 FGT（Fine-Grained Trap）拦截到 EL2

### 11.2 异常号映射

| 指令 | EXCP_ 常量 | 综合征 EC | 目标 EL |
|------|-----------|----------|---------|
| SVC | EXCP_SWI | 0x15 (EC_AA64_SVC) | EL1 |
| HVC | EXCP_HVC | 0x16 (EC_AA64_HVC) | EL2（EL3 时→EL3） |
| SMC | EXCP_SMC | 0x17 (EC_AA64_SMC) | EL3 |

---

## 12. ERET 异常返回

### 12.1 翻译阶段

**文件**：`target/arm/tcg/translate-a64.c:1951-1976`

```c
static bool trans_ERET(DisasContext *s, arg_ERET *a) {
    if (s->current_el == 0) return false;          // EL0 不能 ERET
    if (s->trap_eret) {                             // FEAT_NV: ERET 陷入
        gen_exception_insn_el(s, 0, EXCP_UDEF, syn_erettrap(0), 2);
        return true;
    }
    // 加载 ELR_ELx
    tcg_gen_ld_i64(dst, tcg_env, offsetof(CPUARMState, elr_el[s->current_el]));
    gen_helper_exception_return(tcg_env, dst);
    s->base.is_jmp = DISAS_EXIT;                   // 必须退出循环
    return true;
}
```

### 12.2 HELPER(exception_return) 完整流程

**文件**：`target/arm/tcg/helper-a64.c:622-785`

```c
void HELPER(exception_return)(CPUARMState *env, uint64_t new_pc) {
    int cur_el = arm_current_el(env);
    uint64_t spsr = env->banked_spsr[aarch64_banked_spsr_index(cur_el)];
    bool return_to_aa64 = (spsr & PSTATE_nRW) == 0;

    // 1. 保存当前 SP
    aarch64_save_sp(env, cur_el);
    arm_clear_exclusive(env);              // 清除独占监视器

    // 2. 单步 SS 位处理
    if (arm_generate_debug_exceptions(env))
        spsr &= ~PSTATE_SS;               // 调试使能时清除 SS

    // 3. 从 SPSR 解析目标 EL
    new_el = el_from_spsr(spsr);           // [helper-a64.c:584-620]
    if (new_el == -1) goto illegal_return;

    // 4. 合法性检查
    if (new_el > cur_el) goto illegal_return;           // 不能返回更高 EL
    if (new_el == 2 && !arm_is_el2_enabled(env)) goto illegal_return;
    if (new_el != 0 && arm_el_is_aa64(env, new_el) != return_to_aa64)
        goto illegal_return;                             // 执行状态不匹配
    if (new_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE))
        goto illegal_return;                             // TGE 时不能返回 EL1

    // 5a. 返回 AArch32
    if (!return_to_aa64) {
        env->aarch64 = false;
        cpsr_write_from_spsr_elx(env, spsr);  // SPSR → CPSR
        aarch64_sync_64_to_32(env);            // 64→32 位寄存器映射
        env->regs[15] = new_pc & ~(T ? 0x1 : 0x3);  // 对齐 PC
        helper_rebuild_hflags_a32(env, new_el);
    }

    // 5b. 返回 AArch64
    else {
        env->aarch64 = true;
        spsr &= aarch64_pstate_valid_mask(&cpu->isar);  // 过滤无效位
        pstate_write(env, spsr);                          // SPSR → PSTATE
        if (!arm_singlestep_active(env))
            env->pstate &= ~PSTATE_SS;
        aarch64_restore_sp(env, new_el);                  // 切换 SP
        helper_rebuild_hflags_a64(env, new_el);

        // TBI 地址处理
        tbii = EX_TBFLAG_A64(env->hflags, TBII);
        if ((tbii >> extract64(new_pc, 55, 1)) & 1)
            new_pc = sextract64(new_pc, 0, 56);          // 符号扩展
        env->pc = new_pc;
    }

    // 6. SVE 向量长度切换
    aarch64_sve_change_el(env, cur_el, new_el, return_to_aa64);
    arm_call_el_change_hook(cpu);
}
```

### 12.3 el_from_spsr — SPSR → 目标 EL 解析

**文件**：`target/arm/tcg/helper-a64.c:584-620`

```c
static int el_from_spsr(uint32_t spsr) {
    if (spsr & PSTATE_nRW) {
        // AArch32 模式映射
        switch (spsr & CPSR_M) {
        case ARM_CPU_MODE_USR: return 0;
        case ARM_CPU_MODE_HYP: return 2;
        case ARM_CPU_MODE_FIQ/IRQ/SVC/ABT/UND/SYS: return 1;
        case ARM_CPU_MODE_MON: return -1;  // 非法
        }
    } else {
        // AArch64 模式
        if (extract32(spsr, 1, 1)) return -1;  // M[1]=1 保留
        if (extract32(spsr, 0, 4) == 1) return -1;  // EL0h 非法
        return extract32(spsr, 2, 2);  // M[3:2] = EL
    }
}
```

---

## 13. 非法异常返回处理

**文件**：`target/arm/tcg/helper-a64.c:766-785`

```c
illegal_return:
    // 架构定义的非法返回行为:
    env->pstate |= PSTATE_IL;                    // 设置 Illegal State 位
    env->pc = new_pc;                             // 仍然跳转到 ELR
    // 仅恢复 NZCV + DAIF + ALLINT
    spsr &= PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT;
    spsr |= pstate_read(env) & ~(PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT);
    pstate_write(env, spsr);
    // 不改变 EL、执行状态、SP 选择
```

**非法返回条件**：
- SPSR 中 EL 字段无效（M[1]=1、EL0h、AArch32 MON 模式）
- 返回 EL > 当前 EL
- 返回 EL2 但 EL2 未启用
- 目标 EL 执行状态与 SPSR.nRW 不匹配
- HCR_EL2.TGE=1 时返回 EL1
- FEAT_RME 安全状态无效
- FEAT_GCS EXLOCK 检查失败

---

## 14. PSTATE 位定义与缓存

### 14.1 完整 PSTATE 位定义

**文件**：`target/arm/cpu.h:1546-1573`

| 位 | 名称 | 值 | 含义 |
|----|------|----|------|
| [0] | SP | 0x01 | 栈指针选择（0=SP_EL0，1=SP_ELx） |
| [3:0] | M | 0x0F | 模式字段（EL<<2 \| SP） |
| [4] | nRW | 0x10 | 执行状态（0=AArch64，1=AArch32） |
| [6] | F | 0x40 | FIQ 屏蔽 |
| [7] | I | 0x80 | IRQ 屏蔽 |
| [8] | A | 0x100 | SError 屏蔽 |
| [9] | D | 0x200 | 调试异常屏蔽 |
| [11:10] | BTYPE | 0xC00 | 分支类型（BTI） |
| [12] | SSBS | 0x1000 | 推测存储旁路安全 |
| [13] | ALLINT | 0x2000 | 所有中断屏蔽（FEAT_NMI） |
| [20] | IL | 0x100000 | 非法执行状态 |
| [21] | SS | 0x200000 | 软件单步 |
| [22] | PAN | 0x400000 | 特权访问不可 |
| [23] | UAO | 0x800000 | 用户访问覆盖 |
| [24] | DIT | 0x1000000 | 数据独立时序 |
| [25] | TCO | 0x2000000 | Tag 检查覆盖（MTE） |
| [31:28] | NZCV | 0xF0000000 | 条件标志 |
| [34] | EXLOCK | bit34 | 异常锁定（FEAT_GCS） |

### 14.2 AArch64 PSTATE 模式值

```c
#define PSTATE_MODE_EL0t  0   // EL0, SP_EL0
#define PSTATE_MODE_EL1t  4   // EL1, SP_EL0
#define PSTATE_MODE_EL1h  5   // EL1, SP_EL1
#define PSTATE_MODE_EL2t  8   // EL2, SP_EL0
#define PSTATE_MODE_EL2h  9   // EL2, SP_EL2
#define PSTATE_MODE_EL3t  12  // EL3, SP_EL0
#define PSTATE_MODE_EL3h  13  // EL3, SP_EL3
```

`aarch64_pstate_mode(el, handler)` = `(el << 2) | handler`

---

## 15. 总结与关键设计

### 15.1 异常入口完整时序

```
1. 确定目标 EL        arm_phys_excp_target_el / exception_target_el
2. SVE 长度切换       aarch64_sve_change_el(cur_el → new_el)
3. VBAR 偏移计算      vbar_el[new_el] + 来源偏移 + 类型偏移
4. ESR 写入          esr_el[new_el] = syndrome（同步异常）
5. 保存 SP           aarch64_save_sp(env, cur_el)
6. 保存 ELR          elr_el[new_el] = PC
7. 保存 SPSR         banked_spsr[index] = pstate_read()
8. 设置新 PSTATE      DAIF=1111 + PAN/TCO/SSBS/ALLINT + M=ELxh
9. 恢复 SP           aarch64_restore_sp(env, new_el)
10. 重建 hflags       arm_rebuild_hflags(env)
11. 跳转              env->pc = addr
```

### 15.2 异常返回完整时序

```
1. 保存当前 SP        aarch64_save_sp(env, cur_el)
2. 清除独占监视器      arm_clear_exclusive(env)
3. 读取 SPSR          banked_spsr[index]
4. 解析目标 EL        el_from_spsr(spsr) → new_el
5. 合法性检查          EL/nRW/TGE/RME/GCS 多项验证
6. 恢复 PSTATE        pstate_write(spsr) 或 cpsr_write_from_spsr_elx
7. SS 处理            !singlestep_active → 清除 SS
8. 恢复 SP            aarch64_restore_sp(env, new_el)
9. 重建 hflags        helper_rebuild_hflags_a64/a32
10. TBI 地址处理       56 位截断或符号扩展
11. 恢复 PC           env->pc = new_pc（来自 ELR_ELx）
12. SVE 长度切换      aarch64_sve_change_el(cur_el → new_el)
```

### 15.3 QEMU 特有设计

| 设计 | 说明 |
|------|------|
| PSTATE 缓存分离 | NZCV/DAIF/BTYPE 单独字段存储，避免位操作开销 |
| banked_spsr[] | 用数组索引替代 ARM 架构的命名寄存器 |
| hflags 重建 | 每次 EL 切换后重建翻译标志，驱动 TB 查找 |
| pre_el_change_hook | 通知外设（如 GICv3）EL 即将变化 |
| EXITTB | 异常入口后强制退出当前 TB |
| PSCI 拦截 | SMC/HVC 路径上拦截电源管理调用 |
| 6 维查表 | target_el_table[2][2][2][2][2][4] 实现架构规范表 G1-15 |

### 15.4 核心源文件索引

| 文件 | 关键内容 | 行范围 |
|------|----------|--------|
| helper.c | arm_cpu_do_interrupt 主入口 | 9469-9530 |
| helper.c | arm_cpu_do_interrupt_aarch64 | 9198-9428 |
| helper.c | target_el_table 6 维查表 | 8309-8364 |
| helper.c | arm_phys_excp_target_el | 8369-8421 |
| helper.c | vbar_write | 699-710 |
| helper-a64.c | HELPER(exception_return) | 622-785 |
| helper-a64.c | el_from_spsr | 584-620 |
| cpu-irq.c | arm_cpu_exec_interrupt | 171-270 |
| cpu-irq.c | arm_excp_unmasked | 15-169 |
| translate-a64.c | trans_SVC/HVC/SMC | 3155-3204 |
| translate-a64.c | trans_ERET | 1951-1976 |
| op_helper.c | exception_target_el | 32-45 |
| syndrome.h | arm_exception_class 枚举 | 31-77 |
| syndrome.h | syn_aa64_svc/hvc/smc | 138-160 |
| cpu.h | PSTATE 位定义 | 1546-1573 |
| cpu.h | pstate_read/pstate_write | 1607-1626 |
| internals.h | aarch64_save_sp/restore_sp | 565-581 |
