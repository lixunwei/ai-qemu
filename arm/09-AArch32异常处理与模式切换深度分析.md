# AArch32 异常处理与模式切换深度分析

> 基于 QEMU 11.0.50 源码分析 · 分析日期: 2025-07

---

## 目录

1. [概述](#1-概述)
2. [AArch32 处理器模式](#2-aarch32-处理器模式)
3. [CPSR 位布局与分散存储](#3-cpsr-位布局与分散存储)
4. [CPSR 读取：cpsr_read()](#4-cpsr-读取cpsr_read)
5. [CPSR 写入：cpsr_write()](#5-cpsr-写入cpsr_write)
6. [Banked 寄存器架构](#6-banked-寄存器架构)
7. [Bank 索引映射：bank_number()](#7-bank-索引映射bank_number)
8. [模式切换：switch_mode()](#8-模式切换switch_mode)
9. [异常进入总入口：arm_cpu_do_interrupt_aarch32()](#9-异常进入总入口arm_cpu_do_interrupt_aarch32)
10. [异常向量偏移表](#10-异常向量偏移表)
11. [向量基地址选择](#11-向量基地址选择)
12. [异常实际进入：take_aarch32_exception()](#12-异常实际进入take_aarch32_exception)
13. [HYP 模式异常进入](#13-hyp-模式异常进入)
14. [SVC 异常路径](#14-svc-异常路径)
15. [Abort 异常路径](#15-abort-异常路径)
16. [IRQ/FIQ 异常路径](#16-irqfiq-异常路径)
17. [SMC/HVC 异常路径](#17-smchvc-异常路径)
18. [Undefined 异常路径](#18-undefined-异常路径)
19. [AArch32 异常返回机制](#19-aarch32-异常返回机制)
20. [cpsr_write_eret() 异常返回实现](#20-cpsr_write_eret-异常返回实现)
21. [Thumb 状态与 IT 块处理](#21-thumb-状态与-it-块处理)
22. [AArch32 ↔ AArch64 寄存器同步](#22-aarch32--aarch64-寄存器同步)
23. [32→64 同步：aarch64_sync_32_to_64()](#23-3264-同步aarch64_sync_32_to_64)
24. [64→32 同步：aarch64_sync_64_to_32()](#24-6432-同步aarch64_sync_64_to_32)
25. [SPSR AArch64 Bank 索引映射](#25-spsr-aarch64-bank-索引映射)
26. [异常类型 Syndrome 编码](#26-异常类型-syndrome-编码)
27. [AArch32 与 AArch64 异常处理对比](#27-aarch32-与-aarch64-异常处理对比)
28. [完整异常进入时序图](#28-完整异常进入时序图)
29. [完整异常返回时序图](#29-完整异常返回时序图)
30. [各模式寄存器可见性总表](#30-各模式寄存器可见性总表)

---

## 1. 概述

ARM AArch32 执行状态使用**处理器模式**（Processor Mode）而非 AArch64 的 Exception Level 来管理特权等级。CPSR（Current Program Status Register）的 M[4:0] 位编码当前模式，每种模式拥有独立的 banked 寄存器（SP、LR、SPSR），异常进入时通过 **switch_mode()** 自动保存/恢复。

本文档分析 QEMU 中 AArch32 异常处理的完整路径：从指令解码触发异常、到 CPSR/banked 寄存器切换、到异常向量跳转，以及异常返回的 CPSR 恢复机制。同时分析 AArch32 ↔ AArch64 执行状态切换时的寄存器同步。

**核心源文件**：

| 文件 | 内容 |
|------|------|
| `target/arm/helper.c` | 异常进入、模式切换、CPSR 读写、寄存器同步 |
| `target/arm/cpu.h` | CPSR 常量、模式枚举、banked 寄存器存储 |
| `target/arm/internals.h` | bank_number()、BANK 常量 |
| `target/arm/tcg/translate.c` | SVC/HVC/SMC 解码、异常返回翻译 |
| `target/arm/tcg/op_helper.c` | cpsr_write_eret() 运行时 |
| `target/arm/syndrome.h` | syndrome 编码函数 |

---

## 2. AArch32 处理器模式

```c
// cpu.h:1910-1919
enum arm_cpu_mode {
  ARM_CPU_MODE_USR = 0x10,  // 用户模式
  ARM_CPU_MODE_FIQ = 0x11,  // 快速中断模式
  ARM_CPU_MODE_IRQ = 0x12,  // 中断模式
  ARM_CPU_MODE_SVC = 0x13,  // 管理模式
  ARM_CPU_MODE_MON = 0x16,  // 监控模式 (Security Extensions)
  ARM_CPU_MODE_ABT = 0x17,  // 中止模式
  ARM_CPU_MODE_HYP = 0x1a,  // 虚拟化模式 (Virtualization Extensions)
  ARM_CPU_MODE_UND = 0x1b,  // 未定义模式
  ARM_CPU_MODE_SYS = 0x1f   // 系统模式 (与 USR 共享寄存器)
};
```

**模式与 EL 的映射关系**：

| CPSR.M[4:0] | 模式名称 | EL | 说明 |
|-------------|----------|-----|------|
| 0x10 | USR | EL0 | 用户态，无特权 |
| 0x11 | FIQ | EL1 | 快速中断，额外 banked R8-R12 |
| 0x12 | IRQ | EL1 | 普通中断 |
| 0x13 | SVC | EL1 | 管理调用/复位 |
| 0x16 | MON | EL3 | 安全监控 |
| 0x17 | ABT | EL1 | 预取/数据中止 |
| 0x1a | HYP | EL2 | 虚拟化管理 |
| 0x1b | UND | EL1 | 未定义指令 |
| 0x1f | SYS | EL1 | 系统模式（与 USR 共享寄存器） |

---

## 3. CPSR 位布局与分散存储

AArch32 的 CPSR 是一个 32 位寄存器，但 QEMU 将其拆分存储以提高执行效率。

### 3.1 CPSR 位定义

```c
// cpu.h:1498-1528
#define CPSR_M      (0x1fU)       // bit[4:0]  模式
#define CPSR_T      (1U << 5)     // bit[5]    Thumb 状态
#define CPSR_F      (1U << 6)     // bit[6]    FIQ 屏蔽
#define CPSR_I      (1U << 7)     // bit[7]    IRQ 屏蔽
#define CPSR_A      (1U << 8)     // bit[8]    异步中止屏蔽
#define CPSR_E      (1U << 9)     // bit[9]    大端模式
#define CPSR_IT_2_7 (0xfc00U)     // bit[15:10] IT[7:2]
#define CPSR_GE     (0xfU << 16)  // bit[19:16] SIMD >= 标志
#define CPSR_IL     (1U << 20)    // bit[20]   非法执行状态
#define CPSR_DIT    (1U << 21)    // bit[21]   数据独立时序
#define CPSR_PAN    (1U << 22)    // bit[22]   特权访问禁止
#define CPSR_SSBS   (1U << 23)    // bit[23]   推测存储旁路安全
#define CPSR_J      (1U << 24)    // bit[24]   Jazelle 状态
#define CPSR_IT_0_1 (3U << 25)    // bit[26:25] IT[1:0]
#define CPSR_Q      (1U << 27)    // bit[27]   饱和标志
#define CPSR_V      (1U << 28)    // bit[28]   溢出
#define CPSR_C      (1U << 29)    // bit[29]   进位
#define CPSR_Z      (1U << 30)    // bit[30]   零
#define CPSR_N      (1U << 31)    // bit[31]   负

#define CPSR_NZCV   (CPSR_N | CPSR_Z | CPSR_C | CPSR_V)
#define CPSR_AIF    (CPSR_A | CPSR_I | CPSR_F)
#define CPSR_IT     (CPSR_IT_0_1 | CPSR_IT_2_7)
// 缓存在独立字段中的位
#define CACHED_CPSR_BITS (CPSR_T | CPSR_AIF | CPSR_GE | CPSR_IT | CPSR_Q | CPSR_NZCV)
```

### 3.2 分散存储布局

```c
// cpu.h:290-310
uint32_t uncached_cpsr;   // M, E, IL, DIT, PAN, SSBS, J 等非缓存位
uint32_t spsr;            // 当前模式的 SPSR

// CPSR 标志缓存（加速执行）
uint32_t CF;              // 进位标志 (0 或 1)
uint32_t VF;              // 溢出标志 (bit 31 有效)
uint32_t NF;              // 负标志 (bit 31 有效)
uint32_t ZF;              // 零标志 (==0 时 Z=1)
uint32_t QF;              // 饱和标志 (0 或 1)

// 其他缓存字段
uint32_t GE;              // GE[3:0]
uint32_t thumb;           // CPSR.T
uint32_t condexec_bits;   // IT[7:0] = IT_2_7[5:0] | IT_0_1[1:0]
uint32_t daif;            // A, I, F 位
```

**关键设计**：`uncached_cpsr` 不包含 NZCV、T、AIF、GE、IT、Q 这些经常被修改的位——它们分散在独立字段中，避免了每次条件码更新都需要读-改-写 CPSR 的开销。

---

## 4. CPSR 读取：cpsr_read()

```c
// helper.c:8106-8114
uint32_t cpsr_read(CPUARMState *env)
{
    int ZF;
    ZF = (env->ZF == 0);
    return env->uncached_cpsr
        | (env->NF & 0x80000000)          // N = NF[31]
        | (ZF << 30)                       // Z = (ZF == 0)
        | (env->CF << 29)                  // C = CF
        | ((env->VF & 0x80000000) >> 3)    // V = VF[31] → bit[28]
        | (env->QF << 27)                  // Q
        | (env->thumb << 5)                // T
        | ((env->condexec_bits & 3) << 25) // IT[1:0] → bit[26:25]
        | ((env->condexec_bits & 0xfc) << 8) // IT[7:2] → bit[15:10]
        | (env->GE << 16)                  // GE[3:0]
        | (env->daif & CPSR_AIF);          // A, I, F
}
```

这个函数将分散在各字段中的位**重新组装**回完整的 32 位 CPSR。主要用于：
- 异常进入时保存 SPSR（`env->spsr = cpsr_read(env)`）
- 调试器读取 CPSR
- AArch32 ↔ AArch64 状态同步

---

## 5. CPSR 写入：cpsr_write()

```c
// helper.c:8117-8122（函数签名和核心逻辑）
void cpsr_write(CPUARMState *env, uint32_t val, uint32_t mask,
                CPSRWriteType write_type)
{
    uint32_t changed_daif;
    bool rebuild_hflags = (write_type != CPSRWriteRaw) &&
        (mask & (CPSR_M | CPSR_E | CPSR_IL));
```

写入过程按位域分别处理：

1. **NZCV** → 分解到 `ZF`/`NF`/`CF`/`VF`（helper.c:8124-8128）
2. **Q** → `QF`（helper.c:8130-8131）
3. **T** → `thumb`（helper.c:8133-8134）
4. **IT[0:1]** → `condexec_bits` 低 2 位（helper.c:8136-8138）
5. **IT[2:7]** → `condexec_bits` 高 6 位（helper.c:8140-8142）
6. **GE** → `GE`（helper.c:8144-8145）
7. **AIF** → `daif`，受 SCR_EL3.AW/FW 限制（helper.c:8148-8176）
8. **M** → `uncached_cpsr`，触发 `switch_mode()` 切换 banked 寄存器（helper.c:8178+）

**CPSRWriteType** 控制写入行为：
- `CPSRWriteRaw`：直接写入，不做安全检查
- `CPSRWriteByInstr`：MSR 指令写入，受模式限制
- `CPSRWriteExceptionReturn`：异常返回，执行完整的模式切换和 hflags 重建

---

## 6. Banked 寄存器架构

```c
// cpu.h:296-303
uint64_t banked_spsr[8];   // 各模式的 SPSR（64 位以兼容 AArch64 SPSR 格式）
uint32_t banked_r13[8];    // 各模式的 SP (R13)
uint32_t banked_r14[8];    // 各模式的 LR (R14)

uint32_t usr_regs[5];      // USR 模式的 R8-R12（FIQ 切换时保存）
uint32_t fiq_regs[5];      // FIQ 模式的 R8-R12
```

**AArch32 寄存器 banking 规则**：
- **所有模式**（USR/SYS 除外）：有独立的 SP (R13) 和 SPSR
- **所有模式**（USR/SYS/HYP 除外）：有独立的 LR (R14)
- **HYP 特殊**：LR 与 USR/SYS 共享（因为 HYP 模式下 LR 是通用寄存器，异常返回地址存在 ELR_hyp 中）
- **FIQ 特殊**：额外 banking R8-R12（5 个寄存器），用于快速中断处理无需保存这些寄存器

---

## 7. Bank 索引映射：bank_number()

```c
// internals.h:40-48
#define BANK_USRSYS 0
#define BANK_SVC    1
#define BANK_ABT    2
#define BANK_UND    3
#define BANK_IRQ    4
#define BANK_FIQ    5
#define BANK_HYP    6
#define BANK_MON    7

// internals.h:342-364
static inline int bank_number(int mode)
{
    switch (mode) {
    case ARM_CPU_MODE_USR:
    case ARM_CPU_MODE_SYS:  return BANK_USRSYS;  // 0
    case ARM_CPU_MODE_SVC:  return BANK_SVC;      // 1
    case ARM_CPU_MODE_ABT:  return BANK_ABT;      // 2
    case ARM_CPU_MODE_UND:  return BANK_UND;      // 3
    case ARM_CPU_MODE_IRQ:  return BANK_IRQ;      // 4
    case ARM_CPU_MODE_FIQ:  return BANK_FIQ;      // 5
    case ARM_CPU_MODE_HYP:  return BANK_HYP;      // 6
    case ARM_CPU_MODE_MON:  return BANK_MON;      // 7
    }
    g_assert_not_reached();
}
```

### r14_bank_number() — HYP 特殊处理

```c
// internals.h:377-379
static inline int r14_bank_number(int mode)
{
    return (mode == ARM_CPU_MODE_HYP) ? BANK_USRSYS : bank_number(mode);
}
```

HYP 模式的 R14 与 USR/SYS 共享（BANK_USRSYS），因为 HYP 模式使用 ELR_hyp 而非 LR 作为异常返回地址。R13 (SP) 和 SPSR 仍然使用 BANK_HYP (6)。

---

## 8. 模式切换：switch_mode()

```c
// helper.c:8279-8307
static void switch_mode(CPUARMState *env, int mode)
{
    int old_mode;
    int i;

    old_mode = env->uncached_cpsr & CPSR_M;
    if (mode == old_mode) {
        return;  // 同模式不切换
    }

    // FIQ R8-R12 特殊处理
    if (old_mode == ARM_CPU_MODE_FIQ) {
        // 离开 FIQ：保存 FIQ R8-R12 → fiq_regs，恢复 USR R8-R12
        memcpy(env->fiq_regs, env->regs + 8, 5 * sizeof(uint32_t));
        memcpy(env->regs + 8, env->usr_regs, 5 * sizeof(uint32_t));
    } else if (mode == ARM_CPU_MODE_FIQ) {
        // 进入 FIQ：保存 USR R8-R12 → usr_regs，加载 FIQ R8-R12
        memcpy(env->usr_regs, env->regs + 8, 5 * sizeof(uint32_t));
        memcpy(env->regs + 8, env->fiq_regs, 5 * sizeof(uint32_t));
    }

    // 保存旧模式的 SP 和 SPSR
    i = bank_number(old_mode);
    env->banked_r13[i] = env->regs[13];
    env->banked_spsr[i] = env->spsr;

    // 加载新模式的 SP 和 SPSR
    i = bank_number(mode);
    env->regs[13] = env->banked_r13[i];
    env->spsr = env->banked_spsr[i];

    // 保存旧模式 LR，加载新模式 LR（注意 HYP 特殊处理）
    env->banked_r14[r14_bank_number(old_mode)] = env->regs[14];
    env->regs[14] = env->banked_r14[r14_bank_number(mode)];
}
```

**关键特性**：
- **FIQ 的 R8-R12 交换**：只在进出 FIQ 时触发，其他模式切换不涉及 R8-R12
- **SP/SPSR 使用 bank_number() 索引**：USR 和 SYS 映射到同一个 bank (0)
- **LR 使用 r14_bank_number() 索引**：HYP 映射到 BANK_USRSYS 而非 BANK_HYP

---

## 9. 异常进入总入口：arm_cpu_do_interrupt_aarch32()

```c
// helper.c:8875-9077
static void arm_cpu_do_interrupt_aarch32(CPUState *cs)
```

此函数是 AArch32 状态下所有异常的入口点。流程：

1. **调试异常处理**（8886-8908）：根据 syndrome EC 更新 DBGDSCR.MOE
2. **HYP 路由检查**（8910-8931）：`target_el == 2` 时转到 `arm_cpu_do_interrupt_aarch32_hyp()`
3. **异常分发 switch**（8933-9055）：根据 `exception_index` 确定 `new_mode`、`addr`（向量偏移）、`mask`（中断屏蔽位）、`offset`（返回地址偏移）
4. **向量基地址计算**（9057-9070）：MON→MVBAR，高向量→0xFFFF0000，其他→VBAR
5. **安全状态处理**（9072-9074）：从 MON 模式进入异常时清除 SCR.NS
6. **执行异常进入**（9076）：调用 `take_aarch32_exception()`

---

## 10. 异常向量偏移表

以下是从 `arm_cpu_do_interrupt_aarch32()` 中提取的完整异常分发表（helper.c:8933-9055）：

| 异常类型 | exception_index | 目标模式 | 向量偏移 | 屏蔽位 | 返回偏移 |
|----------|----------------|----------|---------|--------|---------|
| 未定义指令 | EXCP_UDEF | UND | 0x04 | I | Thumb:2, ARM:4 |
| SVC 调用 | EXCP_SWI | SVC | 0x08 | I | 0 |
| 断点 | EXCP_BKPT | ABT | 0x0C | A\|I | 4 |
| 预取中止 | EXCP_PREFETCH_ABORT | ABT | 0x0C | A\|I | 4 |
| 数据中止 | EXCP_DATA_ABORT | ABT | 0x10 | A\|I | 8 |
| IRQ | EXCP_IRQ | IRQ | 0x18 | A\|I | 4 |
| FIQ | EXCP_FIQ | FIQ | 0x1C | A\|I\|F | 4 |
| 虚拟 IRQ | EXCP_VIRQ | IRQ | 0x18 | A\|I | 4 |
| 虚拟 FIQ | EXCP_VFIQ | FIQ | 0x1C | A\|I\|F | 4 |
| 虚拟 SError | EXCP_VSERR | ABT | 0x10 | A\|I | 8 |
| SMC | EXCP_SMC | MON | 0x08 | A\|I\|F | 0 |
| MON 陷阱 | EXCP_MON_TRAP | MON | 0x04 | A\|I\|F | Thumb:2, ARM:4 |

**返回偏移 (offset)** 的含义：
- `LR = PC + offset`（当前指令之后第 N 字节）
- SVC/SMC offset=0：PC 已指向下一条指令
- IRQ/FIQ offset=4：返回到被中断指令的下一条
- Data Abort offset=8：返回到引发中止的指令（需要 -8 恢复到原始 PC）
- UDEF：Thumb 时 offset=2（16 位指令），ARM 时 offset=4（32 位指令）

### IRQ/FIQ 到 Monitor 模式路由

```c
// helper.c:8974-8995
case EXCP_IRQ:
    new_mode = ARM_CPU_MODE_IRQ;
    addr = 0x18;
    mask = CPSR_A | CPSR_I;
    offset = 4;
    if (env->cp15.scr_el3 & SCR_IRQ) {
        // SCR.IRQ=1：IRQ 路由到 Monitor 模式
        new_mode = ARM_CPU_MODE_MON;
        mask |= CPSR_F;  // 额外屏蔽 FIQ
    }
    break;
case EXCP_FIQ:
    new_mode = ARM_CPU_MODE_FIQ;
    addr = 0x1c;
    mask = CPSR_A | CPSR_I | CPSR_F;
    if (env->cp15.scr_el3 & SCR_FIQ) {
        // SCR.FIQ=1：FIQ 路由到 Monitor 模式
        new_mode = ARM_CPU_MODE_MON;
    }
    offset = 4;
    break;
```

当 SCR_EL3 的 IRQ/FIQ 位置 1 时，对应中断被路由到 Monitor 模式而非各自的 IRQ/FIQ 模式。这是 TrustZone 安全世界拦截非安全中断的机制。

---

## 11. 向量基地址选择

```c
// helper.c:9057-9070
if (new_mode == ARM_CPU_MODE_MON) {
    addr += env->cp15.mvbar;               // Monitor 使用 MVBAR
} else if (A32_BANKED_CURRENT_REG_GET(env, sctlr) & SCTLR_V) {
    addr += 0xffff0000;                     // SCTLR.V=1：高向量
} else {
    addr += A32_BANKED_CURRENT_REG_GET(env, vbar); // 否则使用 VBAR
}
```

**三种向量基地址**：

| 条件 | 基地址 | 适用场景 |
|------|--------|---------|
| Monitor 模式 | MVBAR | Secure Monitor 独立向量表 |
| SCTLR.V = 1 | 0xFFFF0000 | 高地址向量（传统内核） |
| SCTLR.V = 0 | VBAR | 可重映射向量（ARMv7+ 标准） |

**注意**：VBAR 和 SCTLR 使用 `A32_BANKED_CURRENT_REG_GET()` 宏获取，这意味着这些寄存器本身也是 banked 的（Secure 和 Non-secure 各有独立副本）。

---

## 12. 异常实际进入：take_aarch32_exception()

```c
// helper.c:8686-8763
static void take_aarch32_exception(CPUARMState *env, int new_mode,
                                   uint32_t mask, uint32_t offset,
                                   uint32_t newpc)
```

这是 AArch32 异常进入的**核心执行函数**，完整流程：

### 步骤 1：模式切换
```c
switch_mode(env, new_mode);  // 切换 banked 寄存器
```

### 步骤 2：保存 SPSR
```c
env->pstate &= ~PSTATE_SS;        // 清除单步标志
env->spsr = cpsr_read(env);        // SPSR = 当前完整 CPSR
```

### 步骤 3：清除 IT 位和 J 位
```c
env->condexec_bits = 0;            // 清除 IT 块状态
env->uncached_cpsr &= ~(CPSR_IL | CPSR_J);  // 清除非法执行/Jazelle
```

### 步骤 4：设置新模式
```c
env->uncached_cpsr = (env->uncached_cpsr & ~CPSR_M) | new_mode;
```

### 步骤 5：确定新的大端设置
```c
new_el = arm_current_el(env);     // 新模式对应的 EL
env->uncached_cpsr &= ~CPSR_E;
if (env->cp15.sctlr_el[new_el] & SCTLR_EE) {
    env->uncached_cpsr |= CPSR_E;  // 按目标 EL 的 SCTLR.EE 设置大端
}
```

### 步骤 6：设置中断屏蔽
```c
env->daif |= mask;  // 设置 A/I/F 屏蔽位
```

### 步骤 7：SSBS 处理
```c
if (cpu_isar_feature(aa32_ssbs, env_archcpu(env))) {
    if (env->cp15.sctlr_el[new_el] & SCTLR_DSSBS_32) {
        env->uncached_cpsr |= CPSR_SSBS;
    } else {
        env->uncached_cpsr &= ~CPSR_SSBS;
    }
}
```

### 步骤 8：HYP vs 非 HYP 分支

**HYP 模式**（helper.c:8726-8728）：
```c
if (new_mode == ARM_CPU_MODE_HYP) {
    env->thumb = (env->cp15.sctlr_el[2] & SCTLR_TE) != 0;
    env->elr_el[2] = env->regs[15];  // 返回地址存入 ELR_hyp
}
```

**非 HYP 模式**（helper.c:8729-8757）：
```c
else {
    // PAN 处理（helper.c:8731-8748）
    // 从非安全态进入 EL3：清除 PAN
    // 进入 EL1 且 SCTLR.SPAN=0：设置 PAN

    // Thumb 状态设置
    if (arm_feature(env, ARM_FEATURE_V4T)) {
        env->thumb = (A32_BANKED_CURRENT_REG_GET(env, sctlr) & SCTLR_TE) != 0;
    }
    env->regs[14] = env->regs[15] + offset;  // LR = PC + offset
}
```

### 步骤 9：跳转和 hflags 重建
```c
env->regs[15] = newpc;           // PC = 向量地址
if (tcg_enabled()) {
    arm_rebuild_hflags(env);     // 重建翻译标志
}
```

**关键差异：HYP vs 非 HYP**：
- HYP 使用 `ELR_hyp`（`elr_el[2]`）保存返回地址，offset 固定为 0
- 非 HYP 使用 `LR`（R14）保存返回地址，`LR = PC + offset`

---

## 13. HYP 模式异常进入

```c
// helper.c:8784-8873
static void arm_cpu_do_interrupt_aarch32_hyp(CPUState *cs)
```

当 `target_el == 2` 时，异常通过此独立路径进入 HYP 模式：

### 向量偏移

| 异常类型 | 向量偏移 |
|----------|---------|
| UDEF | 0x04 |
| SWI | 0x08 |
| BKPT/Prefetch Abort | 0x0C |
| Data Abort | 0x10 |
| HYP_TRAP | **0x14**（HYP 专用入口） |
| IRQ | 0x18 |
| FIQ | 0x1C |
| HVC | 0x08 |

### 关键特殊处理

**1. 非 HYP 源的重定向**（helper.c:8855-8856）：
```c
if (arm_current_el(env) != 2 && addr < 0x14) {
    addr = 0x14;  // 从低特权级陷入 HYP 时，统一使用 0x14 入口
}
```
当异常从 EL0/EL1 陷入 HYP 时（如 HVC、HYP_TRAP），不使用各类型自己的偏移，而是统一路由到偏移 0x14 的 HYP 入口。只有从 HYP 模式自身触发的异常才使用各自的偏移。

**2. SCR 控制的屏蔽位**（helper.c:8859-8868）：
```c
mask = 0;
if (!(env->cp15.scr_el3 & SCR_EA))  mask |= CPSR_A;
if (!(env->cp15.scr_el3 & SCR_IRQ)) mask |= CPSR_I;
if (!(env->cp15.scr_el3 & SCR_FIQ)) mask |= CPSR_F;
```
HYP 模式的 AIF 屏蔽由 SCR_EL3 的 EA/IRQ/FIQ 位控制：只有 SCR 中**未**路由到 Monitor 的中断才在 HYP 入口被屏蔽。

**3. 向量基地址**：
```c
addr += env->cp15.hvbar;  // 使用 HVBAR (Hypervisor Vector Base Address)
```

**4. Syndrome 写入**（helper.c:8838-8852）：
非 IRQ/FIQ 异常会将 syndrome 写入 `esr_el[2]`（HSR），v7 会清除 IL 位。

---

## 14. SVC 异常路径

### 14.1 指令解码

```c
// translate.c:5772-5791
static bool trans_SVC(DisasContext *s, arg_SVC *a)
{
    const uint32_t semihost_imm = s->thumb ? 0xab : 0x123456;

    if (!arm_dc_feature(s, ARM_FEATURE_M) &&
        semihosting_enabled(s->current_el == 0) &&
        (a->imm == semihost_imm)) {
        gen_exception_internal_insn(s, EXCP_SEMIHOST);  // semihosting 特殊路径
    } else {
        if (s->fgt_svc) {
            // Fine-Grained Trap：SVC 被陷入 EL2
            uint32_t syndrome = syn_aa32_svc(a->imm, s->thumb);
            gen_exception_insn_el(s, 0, EXCP_UDEF, syndrome, 2);
        } else {
            gen_update_pc(s, curr_insn_len(s));  // PC 指向下一条指令
            s->svc_imm = a->imm;                 // 保存立即数
            s->base.is_jmp = DISAS_SWI;          // 标记 TB 结束原因
        }
    }
    return true;
}
```

### 14.2 TB 结束时触发异常

```c
// translate.c:6856-6857
case DISAS_SWI:
    gen_exception(EXCP_SWI, syn_aa32_svc(dc->svc_imm, dc->thumb));
    break;
```

### 14.3 Syndrome 编码

```c
// syndrome.h:162-167
static inline uint32_t syn_aa32_svc(uint32_t imm16, bool is_16bit)
{
    uint32_t res = syn_set_ec(0, EC_AA32_SVC);    // EC = 0x11
    res = FIELD_DP32(res, SYNDROME, IL, !is_16bit); // Thumb 16 位: IL=0
    res = FIELD_DP32(res, ISS_IMM16, IMM16, imm16); // SVC 立即数
    return res;
}
```

### 14.4 异常处理

```c
// helper.c:8944-8950
case EXCP_SWI:
    new_mode = ARM_CPU_MODE_SVC;
    addr = 0x08;     // SVC 向量偏移
    mask = CPSR_I;   // 屏蔽 IRQ
    offset = 0;      // PC 已指向下一指令，无需调整
    break;
```

**SVC 完整路径**：`SVC #imm` → `trans_SVC()` → `DISAS_SWI` → `gen_exception(EXCP_SWI, syndrome)` → `arm_cpu_do_interrupt_aarch32()` → `switch_mode(SVC)` → `LR = PC + 0` → `PC = VBAR + 0x08`

---

## 15. Abort 异常路径

### 15.1 Prefetch Abort

```c
// helper.c:8951-8961
case EXCP_BKPT:
case EXCP_PREFETCH_ABORT:
    A32_BANKED_CURRENT_REG_SET(env, ifsr, env->exception.fsr);  // IFSR = 故障状态
    A32_BANKED_CURRENT_REG_SET(env, ifar, env->exception.vaddress); // IFAR = 故障地址
    new_mode = ARM_CPU_MODE_ABT;
    addr = 0x0c;
    mask = CPSR_A | CPSR_I;
    offset = 4;
    break;
```

### 15.2 Data Abort

```c
// helper.c:8963-8973
case EXCP_DATA_ABORT:
    A32_BANKED_CURRENT_REG_SET(env, dfsr, env->exception.fsr);   // DFSR = 故障状态
    A32_BANKED_CURRENT_REG_SET(env, dfar, env->exception.vaddress); // DFAR = 故障地址
    new_mode = ARM_CPU_MODE_ABT;
    addr = 0x10;
    mask = CPSR_A | CPSR_I;
    offset = 8;
    break;
```

### 15.3 AArch32 vs AArch64 故障寄存器对比

| 项目 | AArch32 | AArch64 |
|------|---------|---------|
| 指令故障状态 | IFSR (banked) | ESR_ELx |
| 指令故障地址 | IFAR (banked) | FAR_ELx |
| 数据故障状态 | DFSR (banked) | ESR_ELx |
| 数据故障地址 | DFAR (banked) | FAR_ELx |
| 故障编码格式 | Short/Long FSR | Syndrome EC+ISS |

### 15.4 Virtual SError (VSERR)

```c
// helper.c:9011-9034
case EXCP_VSERR:
    {
        ARMMMUFaultInfo fi = { .type = ARMFault_AsyncExternal, };

        if (extended_addresses_enabled(env)) {
            env->exception.fsr = arm_fi_to_lfsc(&fi);  // Long FSR 格式
        } else {
            env->exception.fsr = arm_fi_to_sfsc(&fi);  // Short FSR 格式
        }
        env->exception.fsr |= env->cp15.vsesr_el2 & 0xd000;
        A32_BANKED_CURRENT_REG_SET(env, dfsr, env->exception.fsr);

        new_mode = ARM_CPU_MODE_ABT;
        addr = 0x10;
        mask = CPSR_A | CPSR_I;
        offset = 8;
    }
    break;
```

VSERR 作为 Data Abort 报告，使用 DFSR，但 DFAR 的值是 UNKNOWN。FSR 格式取决于是否启用了扩展地址（LPAE）。

---

## 16. IRQ/FIQ 异常路径

### 16.1 标准 IRQ 进入

```c
// helper.c:8974-8985
case EXCP_IRQ:
    new_mode = ARM_CPU_MODE_IRQ;
    addr = 0x18;
    mask = CPSR_A | CPSR_I;    // 屏蔽 A 和 I
    offset = 4;                 // LR = PC + 4（返回到被中断的下一指令）
    if (env->cp15.scr_el3 & SCR_IRQ) {
        new_mode = ARM_CPU_MODE_MON;  // 路由到 Monitor
        mask |= CPSR_F;              // 额外屏蔽 FIQ
    }
    break;
```

### 16.2 FIQ 进入

```c
// helper.c:8986-8996
case EXCP_FIQ:
    new_mode = ARM_CPU_MODE_FIQ;
    addr = 0x1c;
    mask = CPSR_A | CPSR_I | CPSR_F;  // 全屏蔽 A/I/F
    if (env->cp15.scr_el3 & SCR_FIQ) {
        new_mode = ARM_CPU_MODE_MON;   // 路由到 Monitor
    }
    offset = 4;
    break;
```

### 16.3 FIQ 的 R8-R12 Banking

FIQ 模式进入时 `switch_mode()` 会：

```c
// helper.c:8292-8294
// 进入 FIQ 时
memcpy(env->usr_regs, env->regs + 8, 5 * sizeof(uint32_t)); // 保存 USR R8-R12
memcpy(env->regs + 8, env->fiq_regs, 5 * sizeof(uint32_t)); // 加载 FIQ R8-R12
```

这使得 FIQ 处理程序无需保存 R8-R12 即可直接使用，减少了中断延迟。

---

## 17. SMC/HVC 异常路径

### 17.1 SMC（Secure Monitor Call）

```c
// helper.c:9036-9041
case EXCP_SMC:
    new_mode = ARM_CPU_MODE_MON;
    addr = 0x08;               // 与 SVC 相同的偏移
    mask = CPSR_A | CPSR_I | CPSR_F;  // 全屏蔽
    offset = 0;                // PC 已指向下一指令
    break;
```

SMC 进入 Monitor 模式时使用 MVBAR 作为向量基地址：
```c
// helper.c:9057-9058
if (new_mode == ARM_CPU_MODE_MON) {
    addr += env->cp15.mvbar;
}
```

### 17.2 安全状态清除

```c
// helper.c:9072-9074
if ((env->uncached_cpsr & CPSR_M) == ARM_CPU_MODE_MON) {
    env->cp15.scr_el3 &= ~SCR_NS;  // 从 MON 模式异常时清除 NS 位
}
```

### 17.3 HVC（Hypervisor Call）

HVC 通过 `arm_cpu_do_interrupt_aarch32_hyp()` 处理：

```c
// helper.c:8828-8830
case EXCP_HVC:
    addr = 0x08;    // 与 SVC 相同偏移
    break;
```

AArch32 HVC 和 SVC 的 syndrome 编码：

```c
// translate.c:6859-6860
case DISAS_HVC:
    gen_exception_el(EXCP_HVC, syn_aa32_hvc(dc->svc_imm), 2);
    break;
case DISAS_SMC:
    gen_exception_el(EXCP_SMC, syn_aa32_smc(), 3);
    break;
```

---

## 18. Undefined 异常路径

```c
// helper.c:8933-8943
case EXCP_UDEF:
    new_mode = ARM_CPU_MODE_UND;
    addr = 0x04;
    mask = CPSR_I;
    if (env->thumb) {
        offset = 2;     // Thumb 指令为 2 字节
    } else {
        offset = 4;     // ARM 指令为 4 字节
    }
    break;
```

**Undefined 异常来源**：
1. **UDF 指令**（永久未定义）
2. **协处理器访问陷阱**（EXCP_NOCP）
3. **Fine-Grained Trap**（如 SVC 被陷入 EL2）
4. **非法指令编码**

### MON_TRAP — Monitor 陷阱

```c
// helper.c:9042-9051
case EXCP_MON_TRAP:
    new_mode = ARM_CPU_MODE_MON;
    addr = 0x04;                        // 与 UDEF 相同偏移
    mask = CPSR_A | CPSR_I | CPSR_F;    // 全屏蔽
    if (env->thumb) {
        offset = 2;
    } else {
        offset = 4;
    }
    break;
```

MON_TRAP 使用与 UDEF 相同的向量偏移 (0x04)，但目标模式是 MON 而非 UND。

---

## 19. AArch32 异常返回机制

AArch32 有多种异常返回方式：

### 19.1 MOVS PC, LR / SUBS PC, LR, #N

传统的异常返回指令，将 LR（或 LR-N）写入 PC 的同时恢复 SPSR 到 CPSR：

```c
// translate.c:1699-1702
static void gen_exception_return(DisasContext *s, TCGv_i32 pc)
{
    gen_rfe(s, pc, load_cpu_field(spsr));  // 使用当前 SPSR
}
```

### 19.2 RFE（Return From Exception）

```c
// translate.c:1686-1697
static void gen_rfe(DisasContext *s, TCGv_i32 pc, TCGv_i32 cpsr)
{
    store_pc_exc_ret(s, pc);       // 保存新 PC
    translator_io_start(&s->base);
    gen_helper_cpsr_write_eret(tcg_env, cpsr);  // 恢复 CPSR
    s->base.is_jmp = DISAS_EXIT;   // 必须退出循环检查中断
}
```

### 19.3 LDM {..., PC}^（带 ^ 的 LDM 加载 PC）

```c
// translate.c:5262-5264
gen_helper_cpsr_write_eret(tcg_env, tmp);
s->base.is_jmp = DISAS_EXIT;
```

所有这些路径最终都调用 `gen_helper_cpsr_write_eret()` → 运行时 `HELPER(cpsr_write_eret)()`。

---

## 20. cpsr_write_eret() 异常返回实现

```c
// op_helper.c:582-605
void HELPER(cpsr_write_eret)(CPUARMState *env, uint32_t val)
{
    uint32_t mask;

    bql_lock();
    arm_call_pre_el_change_hook(env_archcpu(env));  // EL 变化前回调
    bql_unlock();

    mask = aarch32_cpsr_valid_mask(env->features, &env_archcpu(env)->isar);
    cpsr_write(env, val, mask, CPSRWriteExceptionReturn);  // 恢复 CPSR

    // 根据新的 Thumb 状态对齐 PC
    env->regs[15] &= (env->thumb ? ~1 : ~3);

    arm_rebuild_hflags(env);  // 重建翻译标志

    bql_lock();
    arm_call_el_change_hook(env_archcpu(env));  // EL 变化后回调
    bql_unlock();
}
```

**关键步骤**：
1. `arm_call_pre_el_change_hook()`：EL 变化前的回调（如 debug 通知）
2. `cpsr_write(val, mask, CPSRWriteExceptionReturn)`：恢复 CPSR，包括模式切换
3. PC 对齐：Thumb 模式清除 bit[0]，ARM 模式清除 bit[1:0]
4. `arm_rebuild_hflags()`：重建 hflags 以反映新的模式/EL
5. `arm_call_el_change_hook()`：EL 变化后的回调

---

## 21. Thumb 状态与 IT 块处理

### 21.1 异常进入时的 Thumb/IT 处理

```c
// helper.c:8701-8702（take_aarch32_exception 内）
env->condexec_bits = 0;  // 清除 IT 状态
```

异常进入时 IT 块被无条件清除，确保异常处理程序不在 IT 块中执行。

### 21.2 Thumb 状态设置

异常进入时 CPSR.T 由目标 EL 的 SCTLR.TE 决定：

```c
// helper.c:8726-8727（HYP 路径）
env->thumb = (env->cp15.sctlr_el[2] & SCTLR_TE) != 0;

// helper.c:8753-8755（非 HYP 路径）
if (arm_feature(env, ARM_FEATURE_V4T)) {
    env->thumb = (A32_BANKED_CURRENT_REG_GET(env, sctlr) & SCTLR_TE) != 0;
}
```

### 21.3 SVC 的 Thumb 差异

```c
// translate.c:5774
const uint32_t semihost_imm = s->thumb ? 0xab : 0x123456;
```

- **ARM 模式**：`SVC #imm24`（24 位立即数），semihosting 使用 `0x123456`
- **Thumb 模式**：`SVC #imm8`（8 位立即数），semihosting 使用 `0xAB`

Syndrome 中 IL 位也反映指令长度：
```c
// syndrome.h:165
res = FIELD_DP32(res, SYNDROME, IL, !is_16bit);
// Thumb 16 位 SVC: IL=0, ARM 32 位 SVC: IL=1
```

---

## 22. AArch32 ↔ AArch64 寄存器同步

当异常导致执行状态在 AArch32 和 AArch64 之间切换时（例如 AArch32 EL1 的 SVC 被路由到 AArch64 EL3），需要同步寄存器映射。

### AArch64 X 寄存器 ↔ AArch32 R 寄存器映射

| X 寄存器 | 非 FIQ 模式的来源 | FIQ 模式的来源 |
|----------|-------------------|---------------|
| X0-X7 | R0-R7（直接映射） | R0-R7（直接映射） |
| X8-X12 | R8-R12（当前模式） | usr_regs[0-4]（USR 的 R8-R12） |
| X13 | banked_r13[USR] | banked_r13[USR] |
| X14 | banked_r14[USR] | banked_r14[USR] |
| X15 | banked_r13[HYP] / R13(HYP) | banked_r13[HYP] |
| X16 | banked_r14[IRQ] / R14(IRQ) | banked_r14[IRQ] |
| X17 | banked_r13[IRQ] / R13(IRQ) | banked_r13[IRQ] |
| X18 | banked_r14[SVC] / R14(SVC) | banked_r14[SVC] |
| X19 | banked_r13[SVC] / R13(SVC) | banked_r13[SVC] |
| X20 | banked_r14[ABT] / R14(ABT) | banked_r14[ABT] |
| X21 | banked_r13[ABT] / R13(ABT) | banked_r13[ABT] |
| X22 | banked_r14[UND] / R14(UND) | banked_r14[UND] |
| X23 | banked_r13[UND] / R13(UND) | banked_r13[UND] |
| X24-X28 | fiq_regs[0-4] | R8-R12（FIQ 的 R8-R12） |
| X29 | banked_r13[FIQ] | banked_r13[FIQ] |
| X30 | banked_r14[FIQ] | banked_r14[FIQ] |

---

## 23. 32→64 同步：aarch64_sync_32_to_64()

```c
// helper.c:8476-8573
void aarch64_sync_32_to_64(CPUARMState *env)
{
    int i;
    uint32_t mode = env->uncached_cpsr & CPSR_M;

    // R0-R7 → X0-X7（直接复制）
    for (i = 0; i < 8; i++) {
        env->xregs[i] = env->regs[i];
    }

    // R8-R12 / usr_regs → X8-X12（FIQ 特殊处理）
    if (mode == ARM_CPU_MODE_FIQ) {
        for (i = 8; i < 13; i++)
            env->xregs[i] = env->usr_regs[i - 8];  // FIQ 下取 USR 的
    } else {
        for (i = 8; i < 13; i++)
            env->xregs[i] = env->regs[i];           // 其他模式直接取
    }

    // X13-X14: USR SP/LR（helper.c:8505-8516）
    if (mode == ARM_CPU_MODE_USR || mode == ARM_CPU_MODE_SYS) {
        env->xregs[13] = env->regs[13];
        env->xregs[14] = env->regs[14];
    } else {
        env->xregs[13] = env->banked_r13[BANK_USRSYS];
        if (mode == ARM_CPU_MODE_HYP)
            env->xregs[14] = env->regs[14];  // HYP LR = USR LR
        else
            env->xregs[14] = env->banked_r14[BANK_USRSYS];
    }

    // X15: HYP SP（helper.c:8518-8522）
    // X16-X17: IRQ LR/SP（helper.c:8524-8530）
    // X18-X19: SVC LR/SP（helper.c:8532-8538）
    // X20-X21: ABT LR/SP（helper.c:8540-8546）
    // X22-X23: UND LR/SP（helper.c:8548-8554）

    // X24-X30: FIQ bank（helper.c:8561-8570）
    if (mode == ARM_CPU_MODE_FIQ) {
        for (i = 24; i < 31; i++)
            env->xregs[i] = env->regs[i - 16];    // R8-R14
    } else {
        for (i = 24; i < 29; i++)
            env->xregs[i] = env->fiq_regs[i - 24]; // fiq_regs[0-4]
        env->xregs[29] = env->banked_r13[BANK_FIQ];
        env->xregs[30] = env->banked_r14[BANK_FIQ];
    }

    env->pc = env->regs[15];
}
```

**设计原则**：当前活跃模式的寄存器直接从 `regs[]` 读取，非活跃模式的寄存器从 `banked_r13[]`/`banked_r14[]` 读取。

---

## 24. 64→32 同步：aarch64_sync_64_to_32()

```c
// helper.c:8581-8684
void aarch64_sync_64_to_32(CPUARMState *env)
```

逻辑与 32→64 完全对称，但方向相反：
- `env->regs[i] = env->xregs[i]`（X→R）
- `env->banked_r13[bank] = env->xregs[N]`
- 等等

最后将 AArch64 的 PC 写回 R15：
```c
// helper.c:8683
env->regs[15] = env->pc;
```

---

## 25. SPSR AArch64 Bank 索引映射

```c
// internals.h:330-339
static inline unsigned int aarch64_banked_spsr_index(unsigned int el)
{
    static const unsigned int map[4] = {
        [1] = BANK_SVC,  // EL1 → SPSR_EL1 → banked_spsr[1]
        [2] = BANK_HYP,  // EL2 → SPSR_EL2 → banked_spsr[6]
        [3] = BANK_MON,  // EL3 → SPSR_EL3 → banked_spsr[7]
    };
    assert(el >= 1 && el <= 3);
    return map[el];
}
```

这保证了 AArch64 的 SPSR_EL1/2/3 和 AArch32 的 SPSR_svc/SPSR_hyp/SPSR_mon 共享相同的物理存储位置。

---

## 26. 异常类型 Syndrome 编码

### 26.1 AArch32 SVC Syndrome

```c
// syndrome.h:162-167
static inline uint32_t syn_aa32_svc(uint32_t imm16, bool is_16bit)
{
    uint32_t res = syn_set_ec(0, EC_AA32_SVC);     // EC = 0x11
    res = FIELD_DP32(res, SYNDROME, IL, !is_16bit); // ARM: IL=1, Thumb16: IL=0
    res = FIELD_DP32(res, ISS_IMM16, IMM16, imm16); // 立即数
    return res;
}
```

### 26.2 AArch32 HVC Syndrome

```c
// syndrome.h:170-176
static inline uint32_t syn_aa32_hvc(uint32_t imm16)
{
    uint32_t res = syn_set_ec(0, EC_AA32_HVC);     // EC = 0x12
    res = FIELD_DP32(res, SYNDROME, IL, 1);         // 始终 IL=1（HVC 仅 ARM）
    res = FIELD_DP32(res, ISS_IMM16, IMM16, imm16);
    return res;
}
```

### 26.3 AArch32 SMC Syndrome

```c
// syndrome.h:178-182
static inline uint32_t syn_aa32_smc(void)
{
    uint32_t res = syn_set_ec(0, EC_AA32_SMC);     // EC = 0x13
    res = FIELD_DP32(res, SYNDROME, IL, 1);         // 始终 IL=1
    return res;  // SMC 无立即数
}
```

---

## 27. AArch32 与 AArch64 异常处理对比

| 特性 | AArch32 | AArch64 |
|------|---------|---------|
| **特权模型** | 9 种处理器模式 (M[4:0]) | 4 个 Exception Level (EL0-3) |
| **状态寄存器** | CPSR (32 位) | PSTATE (分散存储) |
| **保存状态** | SPSR + LR (banked) | SPSR_ELx + ELR_ELx (系统寄存器) |
| **向量表** | 每类型 4 字节（跳转指令） | 每类型 128 字节（完整代码块） |
| **向量表来源** | 1 维（7 种类型偏移） | 2 维（来源×类型 4×4） |
| **故障寄存器** | DFSR/IFSR + DFAR/IFAR | ESR_ELx + FAR_ELx |
| **返回机制** | MOVS PC,LR / LDM{^} / RFE | ERET |
| **Thumb 切换** | SCTLR.TE 控制 | 不适用（始终 A64） |
| **IT 块** | 异常进入清除 | 不适用 |
| **banked 寄存器** | SP/LR/SPSR × 8 bank | SP_ELx (4 个) |
| **FIQ R8-R12** | 独立 banking | 映射到 X24-X28 |
| **PAN 处理** | CPSR.PAN + SCTLR.SPAN | PSTATE.PAN + SCTLR.SPAN |
| **HYP 特殊** | ELR_hyp, 0x14 入口 | 标准 EL2 处理 |
| **MON 特殊** | MVBAR, SCR.NS 清除 | 标准 EL3 处理 |

---

## 28. 完整异常进入时序图

以 SVC 从 USR 模式进入 SVC 模式为例：

```
USR 模式执行中
    │
    ├── SVC #imm 指令被解码
    │   └── trans_SVC() → svc_imm = imm, DISAS_SWI
    │
    ├── TB 结束时生成异常
    │   └── gen_exception(EXCP_SWI, syn_aa32_svc(imm, thumb))
    │
    ├── QEMU 主循环捕获异常
    │   └── arm_cpu_do_interrupt()
    │       └── arm_cpu_do_interrupt_aarch32(cs)
    │
    ├── 异常分发 (helper.c:8944-8950)
    │   ├── new_mode = ARM_CPU_MODE_SVC
    │   ├── addr = 0x08
    │   ├── mask = CPSR_I
    │   └── offset = 0
    │
    ├── 向量基地址 (helper.c:9063-9069)
    │   └── addr += VBAR
    │
    └── take_aarch32_exception() (helper.c:8686)
        │
        ├── switch_mode(env, SVC) (helper.c:8693)
        │   ├── banked_r13[BANK_USRSYS] = R13  // 保存 USR SP
        │   ├── banked_spsr[BANK_USRSYS] = spsr
        │   ├── banked_r14[BANK_USRSYS] = R14  // 保存 USR LR
        │   ├── R13 = banked_r13[BANK_SVC]     // 加载 SVC SP
        │   ├── spsr = banked_spsr[BANK_SVC]
        │   └── R14 = banked_r14[BANK_SVC]     // 加载 SVC LR
        │
        ├── env->spsr = cpsr_read(env)         // SPSR_svc = CPSR
        │
        ├── env->condexec_bits = 0             // 清除 IT
        │
        ├── uncached_cpsr = (... & ~M) | SVC   // 设置模式位
        │
        ├── CPSR.E = SCTLR_EL1.EE             // 设置大端
        │
        ├── CPSR.J = 0, CPSR.IL = 0           // 清除
        │
        ├── daif |= CPSR_I                     // 屏蔽 IRQ
        │
        ├── thumb = SCTLR.TE                   // 设置 Thumb 状态
        │
        ├── R14 = PC + 0                       // LR = 返回地址
        │
        ├── R15 = VBAR + 0x08                  // PC = SVC 向量
        │
        └── arm_rebuild_hflags()               // 重建翻译标志
```

---

## 29. 完整异常返回时序图

以 `MOVS PC, LR`（从 SVC 返回 USR）为例：

```
SVC 模式执行中
    │
    ├── MOVS PC, LR 被解码
    │   └── gen_exception_return(s, pc)
    │       └── gen_rfe(s, pc=LR, cpsr=load_cpu_field(spsr))
    │           ├── store_pc_exc_ret(s, pc)     // 暂存新 PC
    │           ├── gen_helper_cpsr_write_eret(tcg_env, cpsr)
    │           └── is_jmp = DISAS_EXIT
    │
    └── HELPER(cpsr_write_eret)(env, val=SPSR_svc)
        │
        ├── arm_call_pre_el_change_hook()      // 通知即将切换
        │
        ├── mask = aarch32_cpsr_valid_mask()   // 获取有效位掩码
        │
        ├── cpsr_write(env, val, mask, CPSRWriteExceptionReturn)
        │   │
        │   ├── 恢复 NZCV → ZF/NF/CF/VF
        │   ├── 恢复 Q → QF
        │   ├── 恢复 T → thumb
        │   ├── 恢复 IT → condexec_bits
        │   ├── 恢复 GE → GE
        │   ├── 恢复 AIF → daif
        │   │
        │   └── 恢复 M = USR → switch_mode(env, USR)
        │       ├── banked_r13[BANK_SVC] = R13   // 保存 SVC SP
        │       ├── banked_spsr[BANK_SVC] = spsr
        │       ├── banked_r14[BANK_SVC] = R14   // 保存 SVC LR
        │       ├── R13 = banked_r13[BANK_USRSYS] // 恢复 USR SP
        │       ├── spsr = banked_spsr[BANK_USRSYS]
        │       └── R14 = banked_r14[BANK_USRSYS] // 恢复 USR LR
        │
        ├── PC &= (thumb ? ~1 : ~3)             // 对齐 PC
        │
        ├── arm_rebuild_hflags()                 // 重建翻译标志
        │
        └── arm_call_el_change_hook()            // 通知切换完成

USR 模式继续执行于 PC (= 原始 LR)
```

---

## 30. 各模式寄存器可见性总表

| 寄存器 | USR/SYS | SVC | ABT | UND | IRQ | FIQ | HYP | MON |
|--------|---------|-----|-----|-----|-----|-----|-----|-----|
| R0-R7 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| R8-R12 | USR | USR | USR | USR | USR | **FIQ** | USR | USR |
| R13 (SP) | USR | SVC | ABT | UND | IRQ | FIQ | HYP | MON |
| R14 (LR) | USR | SVC | ABT | UND | IRQ | FIQ | **USR** | MON |
| SPSR | 无 | SVC | ABT | UND | IRQ | FIQ | HYP | MON |
| PC (R15) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

**说明**：
- "USR" 表示与 USR 模式共享
- **FIQ** 的 R8-R12 是独立 banked 的
- **HYP** 的 R14 与 USR 共享（使用 ELR_hyp 作为返回地址）
- SYS 模式完全共享 USR 的所有寄存器（无 SPSR）

---

> **文档信息**
> - 分析版本：QEMU 11.0.50
> - 源文件基线：target/arm/ 目录
> - 行号引用基于分析时的源码快照，代码更新后可能偏移
