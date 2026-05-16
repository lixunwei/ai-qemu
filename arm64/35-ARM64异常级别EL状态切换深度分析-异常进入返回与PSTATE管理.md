# ARM64 异常级别（EL）状态切换深度分析 — 异常进入/返回与 PSTATE 管理

> 基于 QEMU 11.0.50 源码分析，聚焦 ARM64 EL0→EL1→EL2→EL3 异常进入与返回、
> PSTATE 保存/恢复、异常向量表计算、ESR 综合征寄存器、EL2/EL3 路由机制、
> AArch32↔AArch64 互操作以及 TCG 翻译中的异常指令处理。

---

## 目录

1. [CPUARMState 中的 EL 状态表示](#1-cpuarmstate-中的-el-状态表示)
2. [PSTATE 位域定义与读写](#2-pstate-位域定义与读写)
3. [异常进入总流程](#3-异常进入总流程)
4. [异常向量表地址计算](#4-异常向量表地址计算)
5. [异常综合征（ESR_ELx）](#5-异常综合征esr_elx)
6. [PSTATE 在异常进入时的变化](#6-pstate-在异常进入时的变化)
7. [异常返回（ERET）](#7-异常返回eret)
8. [非法异常返回处理](#8-非法异常返回处理)
9. [EL2/EL3 路由机制](#9-el2el3-路由机制)
10. [AArch32↔AArch64 互操作](#10-aarch32aarch64-互操作)
11. [TCG 翻译中的异常指令](#11-tcg-翻译中的异常指令)
12. [总结与状态切换流程图](#12-总结与状态切换流程图)

---

## 1. CPUARMState 中的 EL 状态表示

### 核心状态字段

```c
// target/arm/cpu.h:270-318
struct CPUARMState {
    uint64_t pstate;        // PSTATE 非缓存部分（CurrentEL、SPSel、IL 等）
    bool aarch64;           // PSTATE.nRW 的反转：true=AArch64

    // 条件标志单独缓存以提高执行效率
    uint32_t CF;            // 进位标志（0 或 1）
    uint32_t VF;            // 溢出标志（bit 31）
    uint32_t NF;            // 负数标志（bit 31）
    uint32_t ZF;            // 零标志（0 表示 Z=1）
    uint64_t daif;          // DAIF 异常屏蔽位（独立缓存）
    uint32_t btype;         // BTI 分支类型（SPSR[11:10]）

    // EL 级别的专用寄存器
    uint64_t elr_el[4];     // 异常链接寄存器 ELR_EL1/2/3（索引 1-3）
    uint64_t sp_el[4];      // 栈指针 SP_EL0/1/2/3
    uint64_t banked_spsr[8]; // 存储的程序状态寄存器 SPSR
};
```

### arm_current_el() — 获取当前异常级别

```c
// target/arm/internals.h:489-515
static inline int arm_current_el(CPUARMState *env) {
    if (is_a64(env)) {
        return extract32(env->pstate, 2, 2);  // PSTATE.CurrentEL[3:2]
    }
    // AArch32 模式通过 CPSR 模式位推断
    switch (env->uncached_cpsr & 0x1f) {
    case ARM_CPU_MODE_USR: return 0;
    case ARM_CPU_MODE_HYP: return 2;
    case ARM_CPU_MODE_MON: return 3;
    default:
        if (arm_is_secure(env) && !arm_el_is_aa64(env, 3))
            return 3;  // Secure EL3 为 32 位时，所有安全特权模式在 EL3
        return 1;
    }
}
```

**关键设计**：`pstate` 字段并不存储所有 PSTATE 位。NZCV、DAIF、BTYPE 被单独缓存（`CACHED_PSTATE_BITS`），通过 `pstate_read()`/`pstate_write()` 统一读写。

---

## 2. PSTATE 位域定义与读写

### 位域常量

```c
// target/arm/cpu.h:1550-1581
#define PSTATE_SP       (1U)        // 栈指针选择（0=SP_EL0, 1=SP_ELx）
#define PSTATE_M        (0xFU)      // 模式字段 M[3:0]
#define PSTATE_nRW      (1U << 4)   // 执行状态（0=AArch64, 1=AArch32）
#define PSTATE_F        (1U << 6)   // FIQ 屏蔽
#define PSTATE_I        (1U << 7)   // IRQ 屏蔽
#define PSTATE_A        (1U << 8)   // SError 屏蔽
#define PSTATE_D        (1U << 9)   // Debug 异常屏蔽
#define PSTATE_IL       (1U << 20)  // 非法执行状态
#define PSTATE_SS       (1U << 21)  // 软件步进
#define PSTATE_PAN      (1U << 22)  // 特权访问禁止
#define PSTATE_UAO      (1U << 23)  // 用户访问覆盖
#define PSTATE_DIT      (1U << 24)  // 数据独立时序
#define PSTATE_TCO      (1U << 25)  // 标签检查覆盖
#define PSTATE_NZCV     (N|Z|C|V)   // 条件标志组合
#define PSTATE_DAIF     (D|A|I|F)   // 异常屏蔽组合
#define PSTATE_ALLINT   (1U << 13)  // 全中断屏蔽（FEAT_NMI）

// AArch64 模式编码
#define PSTATE_MODE_EL0t  0   // EL0, SP_EL0
#define PSTATE_MODE_EL1t  4   // EL1, SP_EL0
#define PSTATE_MODE_EL1h  5   // EL1, SP_EL1
#define PSTATE_MODE_EL2t  8   // EL2, SP_EL0
#define PSTATE_MODE_EL2h  9   // EL2, SP_EL2
#define PSTATE_MODE_EL3t  12  // EL3, SP_EL0
#define PSTATE_MODE_EL3h  13  // EL3, SP_EL3
```

### aarch64_pstate_mode() — 构造 PSTATE 模式

```c
// target/arm/cpu.h:1598-1601
static inline unsigned int aarch64_pstate_mode(unsigned int el, bool handler) {
    return (el << 2) | handler;  // handler=true 选择 SP_ELx
}
```

### pstate_read() — 汇聚分散的状态

```c
// target/arm/cpu.h:1607-1615
static inline uint64_t pstate_read(CPUARMState *env) {
    int ZF = (env->ZF == 0);
    return (env->NF & 0x80000000) | (ZF << 30)
        | (env->CF << 29) | ((env->VF & 0x80000000) >> 3)
        | env->pstate | env->daif | (env->btype << 10);
}
```

### pstate_write() — 分发到各缓存

```c
// target/arm/cpu.h:1617-1626
static inline void pstate_write(CPUARMState *env, uint64_t val) {
    env->ZF = (~val) & PSTATE_Z;       // Z=1 时 ZF=0
    env->NF = val;                      // N 在 bit 31
    env->CF = (val >> 29) & 1;          // C 在 bit 29
    env->VF = (val << 3) & 0x80000000;  // V 在 bit 28 → 移到 bit 31
    env->daif = val & PSTATE_DAIF;
    env->btype = (val >> 10) & 3;
    env->pstate = val & ~CACHED_PSTATE_BITS; // 其余位存入 pstate
}
```

---

## 3. 异常进入总流程

### arm_cpu_do_interrupt_aarch64()

```c
// target/arm/helper.c:9197-9428
static void arm_cpu_do_interrupt_aarch64(CPUState *cs) {
    unsigned int new_el = env->exception.target_el;   // 目标异常级别
    vaddr addr = env->cp15.vbar_el[new_el];           // VBAR_ELx 基址
    uint64_t new_mode = aarch64_pstate_mode(new_el, true); // ELxh 模式
    unsigned int cur_el = arm_current_el(env);

    // SVE 状态迁移（VL 可能随 EL 变化）
    aarch64_sve_change_el(env, cur_el, new_el, is_a64(env));

    // 1. 计算向量偏移（见第 4 节）
    // 2. 设置 ESR_ELx（见第 5 节）
    // 3. 保存旧状态
    if (is_a64(env)) {
        old_mode = pstate_read(env);              // 读取完整 PSTATE
        aarch64_save_sp(env, cur_el);             // 保存 SP 到 sp_el[cur_el]
        env->elr_el[new_el] = env->pc;            // PC → ELR_ELx
    } else {
        old_mode = cpsr_read_for_spsr_elx(env);   // AArch32: CPSR → SPSR 格式
        env->elr_el[new_el] = env->regs[15];      // R15 → ELR_ELx
        aarch64_sync_32_to_64(env);               // 同步 32 位寄存器到 64 位视图
    }
    env->banked_spsr[aarch64_banked_spsr_index(new_el)] = old_mode;

    // 4. 构建新 PSTATE（见第 6 节）
    // 5. 写入新 PSTATE 并跳转
    pstate_write(env, PSTATE_DAIF | new_mode);    // DAIF 全部屏蔽
    env->aarch64 = true;                          // 进入 AArch64 状态
    aarch64_restore_sp(env, new_el);              // 恢复 SP_ELx
    env->pc = addr;                               // 跳转到向量入口
}
```

**异常进入状态保存摘要**：

| 保存内容 | 源 | 目标 |
|---------|---|------|
| PC | `env->pc`（AArch64）/ `env->regs[15]`（AArch32） | `env->elr_el[new_el]` |
| PSTATE/CPSR | `pstate_read()` / `cpsr_read()` | `env->banked_spsr[new_el]` |
| SP | `env->sp_el[cur_el]` | 通过 `aarch64_save_sp()` 保存 |

---

## 4. 异常向量表地址计算

向量地址 = `VBAR_ELx` + 偏移，偏移取决于异常来源和类型：

```c
// target/arm/helper.c:9217-9255
if (cur_el < new_el) {
    // 从低 EL 进入高 EL
    switch (new_el) {
    case 3: is_aa64 = arm_scr_rw_eff(env); break;     // SCR_EL3.RW
    case 2: is_aa64 = (hcr & HCR_RW) != 0; break;     // HCR_EL2.RW
    case 1: is_aa64 = is_a64(env); break;              // 当前状态
    }
    if (is_aa64)
        addr += 0x400;   // 从 AArch64 低 EL 进入
    else
        addr += 0x600;   // 从 AArch32 低 EL 进入
} else {
    // 同 EL 异常
    if (pstate_read(env) & PSTATE_SP)
        addr += 0x200;   // 使用 SP_ELx（h 后缀）
    // 否则 +0x000（使用 SP_EL0, t 后缀）
}

// 异常类型偏移
// target/arm/helper.c:9257-9341
switch (cs->exception_index) {
case EXCP_PREFETCH_ABORT:
case EXCP_DATA_ABORT:
case EXCP_SWI:  case EXCP_HVC:  case EXCP_SMC:
    addr += 0x000;    // Synchronous
case EXCP_IRQ:  case EXCP_VIRQ:
    addr += 0x080;    // IRQ
case EXCP_FIQ:  case EXCP_VFIQ:
    addr += 0x100;    // FIQ
case EXCP_VSERR:
    addr += 0x180;    // SError
}
```

### 完整向量表布局

| 偏移 | 来源 EL / SP | 同步 | IRQ | FIQ | SError |
|------|-------------|------|-----|-----|--------|
| +0x000 | 同 EL, SP_EL0 (t) | +0x000 | +0x080 | +0x100 | +0x180 |
| +0x200 | 同 EL, SP_ELx (h) | +0x200 | +0x280 | +0x300 | +0x380 |
| +0x400 | 低 EL, AArch64 | +0x400 | +0x480 | +0x500 | +0x580 |
| +0x600 | 低 EL, AArch32 | +0x600 | +0x680 | +0x700 | +0x780 |

---

## 5. 异常综合征（ESR_ELx）

### syn_* 综合征构造函数

```c
// target/arm/syndrome.h:138-160
static inline uint32_t syn_aa64_svc(uint32_t imm16) {
    return syn_set_ec(0, EC_AA64_SVC) | FIELD_DP32(IL=1) | IMM16;
}
static inline uint32_t syn_aa64_hvc(uint32_t imm16) {
    return syn_set_ec(0, EC_AA64_HVC) | IL=1 | IMM16;
}
static inline uint32_t syn_aa64_smc(uint32_t imm16) {
    return syn_set_ec(0, EC_AA64_SMC) | IL=1 | IMM16;
}
```

ESR 在异常进入时写入：

```c
// target/arm/helper.c:9320
env->cp15.esr_el[new_el] = env->exception.syndrome;
```

对于 Data Abort，还写入 FAR（故障地址）：

```c
// target/arm/helper.c:9272
env->cp15.far_el[new_el] = env->exception.vaddress;
```

### ESR 格式

```
[31:26] EC  — 异常类（SVC=0x15, HVC=0x16, SMC=0x17, Data Abort=0x24/0x25 等）
[25]    IL  — 指令长度（1=32位, 0=16位 Thumb）
[24:0]  ISS — 指令特定综合征（imm16、DFSC、WnR 等）
```

---

## 6. PSTATE 在异常进入时的变化

```c
// target/arm/helper.c:9375-9415
// 初始 new_mode = aarch64_pstate_mode(new_el, true) = (el<<2)|1

// 1. PAN 处理（FEAT_PAN）
if (cpu_isar_feature(aa64_pan, cpu)) {
    new_mode |= old_mode & PSTATE_PAN;  // 默认保留旧 PAN
    if (new_el == 1 || (new_el == 2 && VHE)) {
        if (!(sctlr_el[new_el] & SCTLR_SPAN))
            new_mode |= PSTATE_PAN;     // SPAN=0 时强制 PAN=1
    }
}

// 2. TCO（FEAT_MTE）
if (cpu_isar_feature(aa64_mte, cpu))
    new_mode |= PSTATE_TCO;             // 异常进入时 TCO=1

// 3. SSBS（FEAT_SSBS）
if (sctlr_el[new_el] & SCTLR_DSSBS_64)
    new_mode |= PSTATE_SSBS;

// 4. ALLINT（FEAT_NMI）
if (!(sctlr_el[new_el] & SCTLR_SPINTMASK))
    new_mode |= PSTATE_ALLINT;

// 5. 最终写入：DAIF 全部屏蔽 + new_mode
pstate_write(env, PSTATE_DAIF | new_mode);
```

### 异常进入时 PSTATE 变化汇总

| 字段 | 异常进入后的值 | 说明 |
|------|--------------|------|
| CurrentEL | new_el | 提升到目标 EL |
| SPSel | 1 (h) | 使用 SP_ELx |
| DAIF | 全部置 1 | D、A、I、F 全部屏蔽 |
| NZCV | 清零 | 不保留（保存在 SPSR 中） |
| PAN | 条件保留/设置 | SCTLR.SPAN=0 时强制 PAN=1 |
| TCO | 1 | MTE 标签检查关闭 |
| IL | 0 | 清除非法状态 |
| SS | 0 | 清除单步 |
| nRW | 0 | 进入 AArch64 |

---

## 7. 异常返回（ERET）

### HELPER(exception_return) — 核心实现

```c
// target/arm/tcg/helper-a64.c:622-785
void HELPER(exception_return)(CPUARMState *env, uint64_t new_pc) {
    int cur_el = arm_current_el(env);
    uint64_t spsr = env->banked_spsr[aarch64_banked_spsr_index(cur_el)];
    bool return_to_aa64 = (spsr & PSTATE_nRW) == 0;

    aarch64_save_sp(env, cur_el);     // 保存当前 SP
    arm_clear_exclusive(env);          // 清除独占监视器

    // 从 SPSR 推断目标 EL
    new_el = el_from_spsr(spsr);

    // 合法性检查（见第 8 节）
    if (new_el > cur_el) goto illegal_return;  // 不能返回到更高 EL
    if (new_el == 2 && !arm_is_el2_enabled(env)) goto illegal_return;

    if (return_to_aa64) {
        // 返回 AArch64
        env->aarch64 = true;
        pstate_write(env, spsr);           // SPSR → PSTATE
        aarch64_restore_sp(env, new_el);   // 恢复 SP_ELx 或 SP_EL0
        // TBI 处理...
        env->pc = new_pc;                  // ELR_ELx → PC
    } else {
        // 返回 AArch32
        env->aarch64 = false;
        cpsr_write_from_spsr_elx(env, spsr); // SPSR → CPSR
        aarch64_sync_64_to_32(env);          // 同步 64→32 寄存器映射
        env->regs[15] = new_pc;              // ELR → R15
    }

    aarch64_sve_change_el(env, cur_el, new_el, return_to_aa64);
}
```

### 异常返回状态恢复摘要

| 恢复内容 | 源 | 目标 |
|---------|---|------|
| PSTATE | `banked_spsr[cur_el]` | `pstate_write()` 分发到各缓存 |
| PC | `elr_el[cur_el]` | `env->pc`（AArch64）/ `env->regs[15]`（AArch32） |
| SP | `sp_el[new_el]` | 通过 `aarch64_restore_sp()` 恢复 |
| 执行状态 | `SPSR.nRW` | `env->aarch64` |

---

## 8. 非法异常返回处理

ERET 返回时有多种非法情况：

```c
// target/arm/tcg/helper-a64.c:646-680
// 1. 无效模式字段
new_el = el_from_spsr(spsr);
if (new_el == -1) goto illegal_return;

// 2. 返回到更高或未实现的 EL
if (new_el > cur_el) goto illegal_return;
if (new_el == 2 && !arm_is_el2_enabled(env)) goto illegal_return;

// 3. RME 安全状态无效（从 EL3 返回低 EL，NSE=1 但 NS=0）
if (cur_el == 3 && new_el < 3 &&
    (scr_el3 & (SCR_NS | SCR_NSE)) == SCR_NSE)
    goto illegal_return;

// 4. 目标 EL 的执行宽度不匹配
if (new_el != 0 && arm_el_is_aa64(env, new_el) != return_to_aa64)
    goto illegal_return;

// 5. CPU 不支持 AArch32 但尝试返回 AArch32
if (!return_to_aa64 && !cpu_isar_feature(aa64_aa32, cpu))
    goto illegal_return;

// 6. HCR_EL2.TGE=1 时不能返回 EL1
if (new_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE))
    goto illegal_return;
```

### 非法返回的处理

```c
// target/arm/tcg/helper-a64.c:766-784
illegal_return:
    env->pstate |= PSTATE_IL;     // 设置非法执行状态位
    env->pc = new_pc;             // PC 仍然从 ELR 恢复
    // 仅恢复 NZCV 和 DAIF，其余保持不变
    spsr &= PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT;
    spsr |= pstate_read(env) & ~(PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT);
    pstate_write(env, spsr);
    // EL 不变，执行状态不变，SP 不变
```

---

## 9. EL2/EL3 路由机制

### SVC/HVC/SMC 的目标 EL

```c
// target/arm/tcg/translate-a64.c:3155-3205
// SVC → 目标 EL1（从 EL0）或当前 EL（从 EL1）
gen_exception_insn(s, 4, EXCP_SWI, syn_aa64_svc(imm));

// HVC → 目标 EL2（从 EL1）或 EL3（从 EL3）
int target_el = s->current_el == 3 ? 3 : 2;
gen_exception_insn_el(s, 4, EXCP_HVC, syn_aa64_hvc(imm), target_el);

// SMC → 始终目标 EL3
gen_exception_insn_el(s, 4, EXCP_SMC, syn_aa64_smc(imm), 3);
```

### SCR_EL3 — 安全监控配置

| 位 | 名称 | 作用 |
|----|------|------|
| RW | 执行状态 | 控制 EL2/EL1 是否为 AArch64 |
| NS | 非安全 | 低 EL 的安全状态 |
| NSE | 非安全扩展 | RME Realm 状态 |
| HCE | HVC 使能 | 是否允许 HVC 指令 |
| SMD | SMC 禁止 | SMD=1 禁止 SMC |
| EEL2 | 安全 EL2 使能 | FEAT_SEL2 |

### HCR_EL2 — 虚拟化配置

| 位 | 名称 | 作用 |
|----|------|------|
| RW | 执行状态 | 控制 EL1 是否为 AArch64 |
| TGE | 陷入通用异常 | EL0 异常直接到 EL2 |
| E2H | EL2 宿主使能 | VHE 模式 |
| AMO | SError 路由 | 路由到 EL2 |
| IMO | IRQ 路由 | 路由到 EL2 |
| FMO | FIQ 路由 | 路由到 EL2 |
| NV/NV2 | 嵌套虚拟化 | FEAT_NV/NV2 |

### 中断路由示例

当 EL1 执行时：
- `HCR_EL2.IMO=1` → IRQ 路由到 EL2（虚拟化场景）
- `HCR_EL2.FMO=1` → FIQ 路由到 EL2
- `SCR_EL3.IRQ=1` → IRQ 路由到 EL3（安全监控截获）

---

## 10. AArch32↔AArch64 互操作

### AArch32 → AArch64 异常进入

当从 AArch32 状态（如 EL1）进入 AArch64 状态（如 EL3）：

```c
// target/arm/helper.c:9361-9368
} else {
    // 当前是 AArch32
    old_mode = cpsr_read_for_spsr_elx(env);  // CPSR → SPSR 格式
    env->elr_el[new_el] = env->regs[15];     // R15 → ELR_ELx
    aarch64_sync_32_to_64(env);              // R0-R14 → X0-X14
    env->condexec_bits = 0;                  // 清除 IT 状态
}
```

### AArch64 → AArch32 异常返回

```c
// target/arm/tcg/helper-a64.c:697-717
if (!return_to_aa64) {
    env->aarch64 = false;
    cpsr_write_from_spsr_elx(env, spsr);  // SPSR → CPSR
    aarch64_sync_64_to_32(env);           // X0-X14 → R0-R14
    if (spsr & CPSR_T)
        env->regs[15] = new_pc & ~0x1;   // Thumb 对齐
    else
        env->regs[15] = new_pc & ~0x3;   // ARM 对齐
}
```

### 执行状态由什么决定？

| 场景 | 决定因素 |
|------|---------|
| 进入 EL3 | `SCR_EL3.RW`（固定，通常为 AArch64） |
| 进入 EL2 | `HCR_EL2.RW`（由 EL3 软件设置） |
| 进入 EL1 | `HCR_EL2.RW` 或 `SCR_EL3.RW`（取决于是否有 EL2） |
| 从 ERET 返回 | `SPSR.nRW` 位 |

---

## 11. TCG 翻译中的异常指令

### SVC 翻译

```c
// target/arm/tcg/translate-a64.c:3155-3171
static bool trans_SVC(DisasContext *s, arg_i *a) {
    uint32_t syndrome = syn_aa64_svc(a->imm);
    gen_ss_advance(s);                    // 单步状态机推进（架构要求先于异常）
    gen_exception_insn(s, 4, EXCP_SWI, syndrome);
    return true;
}
```

### HVC 翻译

```c
// target/arm/tcg/translate-a64.c:3173-3191
static bool trans_HVC(DisasContext *s, arg_i *a) {
    int target_el = s->current_el == 3 ? 3 : 2;
    gen_a64_update_pc(s, 0);
    gen_helper_pre_hvc(tcg_env);          // 运行时检查 HVC 是否被 trap
    gen_ss_advance(s);
    gen_exception_insn_el(s, 4, EXCP_HVC, syn_aa64_hvc(a->imm), target_el);
    return true;
}
```

### SMC 翻译

```c
// target/arm/tcg/translate-a64.c:3193-3205
static bool trans_SMC(DisasContext *s, arg_i *a) {
    gen_a64_update_pc(s, 0);
    gen_helper_pre_smc(tcg_env, ...);     // 运行时检查 SMC 是否被禁止
    gen_ss_advance(s);
    gen_exception_insn_el(s, 4, EXCP_SMC, syn_aa64_smc(a->imm), 3);
    return true;
}
```

### ERET 翻译

```c
// target/arm/tcg/translate-a64.c:1951-1975
static bool trans_ERET(DisasContext *s, arg_ERET *a) {
    if (s->current_el == 0) return false;  // EL0 不能执行 ERET

    if (s->trap_eret) {
        // FEAT_FGT: ERET 被 trap 到 EL2
        gen_exception_insn_el(s, 0, EXCP_UDEF, syn_erettrap(0), 2);
        return true;
    }

    // 加载 ELR_ELx
    tcg_gen_ld_i64(dst, tcg_env, offsetof(CPUARMState, elr_el[s->current_el]));
    translator_io_start(&s->base);        // 可能改变中断状态
    gen_helper_exception_return(tcg_env, dst);  // 调用运行时 helper
    s->base.is_jmp = DISAS_EXIT;          // 必须退出 TB
    return true;
}
```

**ERET 翻译的关键特点**：
- ERET 总是结束当前 TB（`DISAS_EXIT`），因为 EL/执行状态可能改变
- 调用 `translator_io_start()` 因为返回后可能有未屏蔽的中断需要处理
- ERETA（带 PAC 认证的 ERET）额外调用 `auth_branch_target()` 验证返回地址

---

## 12. 总结与状态切换流程图

### 异常进入流程（EL0 → EL1 SVC 为例）

```
用户态 EL0 执行 SVC #0
    │
    ▼
TCG 翻译: trans_SVC()
    │  syn_aa64_svc(0) → syndrome
    │  gen_exception_insn(EXCP_SWI, syndrome)
    ▼
运行时: arm_cpu_do_interrupt_aarch64()
    │
    ├── target_el = 1
    ├── addr = VBAR_EL1 + 0x400  (从低 EL AArch64 进入)
    │                    + 0x000  (Synchronous)
    │
    ├── 保存状态:
    │   ├── SPSR_EL1 = pstate_read()  [NZCV|DAIF|CurrentEL=0|SPSel|...]
    │   ├── ELR_EL1  = PC
    │   └── SP_EL0   = SP (aarch64_save_sp)
    │
    ├── 设置 ESR_EL1 = syndrome (EC=0x15, IMM16=0)
    │
    ├── 构建新 PSTATE:
    │   ├── CurrentEL = 1, SPSel = 1 (h)
    │   ├── DAIF = 全屏蔽 (D=A=I=F=1)
    │   ├── PAN = (SCTLR_EL1.SPAN=0 ? 1 : 保持)
    │   ├── TCO = 1, IL = 0, SS = 0
    │   └── nRW = 0 (AArch64)
    │
    └── PC = VBAR_EL1 + 0x400
        SP = SP_EL1 (aarch64_restore_sp)
```

### 异常返回流程（EL1 → EL0 ERET 为例）

```
内核 EL1 执行 ERET
    │
    ▼
TCG 翻译: trans_ERET()
    │  加载 ELR_EL1
    │  gen_helper_exception_return()
    ▼
运行时: HELPER(exception_return)
    │
    ├── 读取 SPSR_EL1 → spsr
    ├── el_from_spsr(spsr) → new_el = 0
    │
    ├── 合法性检查:
    │   ├── new_el(0) ≤ cur_el(1) ✓
    │   ├── return_to_aa64 = (spsr.nRW == 0) ✓
    │   └── HCR_EL2.TGE check ✓
    │
    ├── 恢复状态:
    │   ├── pstate_write(spsr)  [恢复 NZCV|DAIF|CurrentEL=0|SPSel=0]
    │   ├── aarch64_restore_sp(EL0)  [SP = SP_EL0]
    │   └── PC = ELR_EL1 (经 TBI 处理)
    │
    └── 返回 EL0 用户态执行
```

### EL 切换全景图

```
┌────────────────────────────────────────────────────┐
│                     EL3 (安全监控)                   │
│  SMC 进入 │ SCR_EL3 控制低 EL 安全/执行状态         │
│  ERET 返回 │ SPSR_EL3/ELR_EL3                      │
├────────────┼───────────────────────────────────────┤
│            │                                        │
│            ▼                                        │
│  ┌─────────────────────────────────────────────┐   │
│  │              EL2 (虚拟化/Hypervisor)          │   │
│  │  HVC 进入 │ HCR_EL2 控制 EL1 trap 与路由     │   │
│  │  ERET 返回 │ SPSR_EL2/ELR_EL2               │   │
│  ├────────────┼────────────────────────────────┤   │
│  │            │                                  │   │
│  │            ▼                                  │   │
│  │  ┌──────────────────────────────────────┐    │   │
│  │  │         EL1 (内核/OS)                 │    │   │
│  │  │  SVC 进入 │ SCTLR_EL1 控制行为       │    │   │
│  │  │  ERET 返回 │ SPSR_EL1/ELR_EL1       │    │   │
│  │  ├────────────┼───────────────────────┤    │   │
│  │  │            ▼                         │    │   │
│  │  │  ┌───────────────────────────┐      │    │   │
│  │  │  │   EL0 (用户态应用)         │      │    │   │
│  │  │  │   SVC → EL1               │      │    │   │
│  │  │  │   中断 → EL1/EL2/EL3      │      │    │   │
│  │  │  └───────────────────────────┘      │    │   │
│  │  └──────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘
```

---

**关键源文件**：
- `target/arm/helper.c` — `arm_cpu_do_interrupt_aarch64()` 异常进入主逻辑
- `target/arm/tcg/helper-a64.c` — `HELPER(exception_return)` 异常返回
- `target/arm/cpu.h` — PSTATE 位定义、`pstate_read()/write()`、ELR/SP/SPSR 存储
- `target/arm/internals.h` — `arm_current_el()`、辅助宏
- `target/arm/syndrome.h` — `syn_aa64_svc/hvc/smc()` 综合征构造
- `target/arm/tcg/translate-a64.c` — `trans_SVC/HVC/SMC/ERET()` TCG 翻译
