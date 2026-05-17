# ARM64 调试架构深度分析

## MDSCR/DBGBCR/DBGWCR、断点/观察点、单步执行

> 基于 QEMU 11.0.50 源码分析，涵盖调试寄存器完整定义（MDSCR_EL1/DBGBCR/DBGBVR/DBGWCR/DBGWVR）、硬件断点与软件断点实现、数据观察点与 BAS 字段处理、单步执行状态机、调试异常路由与屏蔽、翻译时调试检查、GDB 桩集成机制。

---

## 目录

1. [调试架构总体概述](#1-调试架构总体概述)
2. [调试寄存器存储与定义](#2-调试寄存器存储与定义)
3. [MDSCR_EL1 — 调试系统控制](#3-mdscr_el1-调试系统控制)
4. [DBGBCR/DBGBVR — 断点控制与值寄存器](#4-dbgbcrdbgbvr-断点控制与值寄存器)
5. [DBGWCR/DBGWVR — 观察点控制与值寄存器](#5-dbgwcrdbgwvr-观察点控制与值寄存器)
6. [调试异常路由](#6-调试异常路由)
7. [调试异常使能条件](#7-调试异常使能条件)
8. [硬件断点实现](#8-硬件断点实现)
9. [硬件观察点实现](#9-硬件观察点实现)
10. [断点/观察点匹配核心逻辑](#10-断点观察点匹配核心逻辑)
11. [调试异常处理入口](#11-调试异常处理入口)
12. [软件断点 BRK 指令](#12-软件断点-brk-指令)
13. [单步执行机制](#13-单步执行机制)
14. [翻译时调试检查与 TB 标志](#14-翻译时调试检查与-tb-标志)
15. [总结与关键设计](#15-总结与关键设计)

---

## 1. 调试架构总体概述

QEMU ARM64 调试子系统实现了 ARMv8 Self-hosted Debug 架构：

```
调试寄存器层 (MDSCR/DBGBCR/DBGWCR)
         ↓ 写入时触发
QEMU 断点/观察点注册 (cpu_breakpoint_insert / cpu_watchpoint_insert)
         ↓ TB 执行时检查
匹配逻辑 (bp_wp_matches → arm_debug_check_breakpoint/watchpoint)
         ↓ 匹配成功
异常生成 (raise_exception_debug → arm_cpu_do_interrupt)
         ↓
调试向量处理 (VBAR + Sync offset, EC=BREAKPOINT/WATCHPOINT/SOFTWARESTEP)
```

**三类调试事件**：
| 类型 | 触发方式 | EC 编码 | QEMU 实现 |
|------|----------|---------|-----------|
| 硬件断点 | DBGBCR/DBGBVR 地址匹配 | EC_BREAKPOINT (0x30/0x31) | cpu_breakpoint_insert |
| 硬件观察点 | DBGWCR/DBGWVR 数据访问匹配 | EC_WATCHPOINT (0x34/0x35) | cpu_watchpoint_insert |
| 软件断点 | BRK 指令 | EC_AA64_BKPT (0x3c) | 翻译时直接生成异常 |
| 软件单步 | MDSCR.SS + PSTATE.SS | EC_SOFTWARESTEP (0x32/0x33) | TB 标志 + 状态机 |

---

## 2. 调试寄存器存储与定义

### 2.1 CPUARMState 中的调试寄存器

**文件**：`target/arm/cpu.h:529-538`

```c
struct CPUARMState {
    struct {
        ...
        uint64_t dbgbvr[16];    // 断点值寄存器 (DBGBVR0-15_EL1)
        uint64_t dbgbcr[16];    // 断点控制寄存器 (DBGBCR0-15_EL1)
        uint64_t dbgwvr[16];    // 观察点值寄存器 (DBGWVR0-15_EL1)
        uint64_t dbgwcr[16];    // 观察点控制寄存器 (DBGWCR0-15_EL1)
        uint64_t dbgclaim;      // DBGCLAIM 声明标记
        uint64_t mdscr_el1;     // 调试系统控制寄存器
        uint64_t oslsr_el1;     // OS Lock 状态
        uint64_t osdlr_el1;     // OS DoubleLock 状态
        uint64_t mdcr_el2;      // EL2 调试控制
        uint64_t mdcr_el3;      // EL3 调试控制
    } cp15;
};
```

### 2.2 断点/观察点数量

由 `arm_num_brps()` / `arm_num_wrps()` / `arm_num_ctx_cmps()` 从 CPU ID 寄存器提取。每种最多 16 个。

### 2.3 cpreg 动态注册

**文件**：`target/arm/debug_helper.c:477-523`

断点和观察点寄存器以循环方式动态注册：

```c
// 断点: DBGBVR{i}_EL1 + DBGBCR{i}_EL1
for (i = 0; i < brps; i++) {
    // opc2=4: DBGBVR, opc2=5: DBGBCR
    // fieldoffset = offsetof(cp15.dbgbvr[i]) / offsetof(cp15.dbgbcr[i])
    // writefn = dbgbvr_write / dbgbcr_write
}

// 观察点: DBGWVR{i}_EL1 + DBGWCR{i}_EL1
for (i = 0; i < wrps; i++) {
    // opc2=6: DBGWVR, opc2=7: DBGWCR
    // writefn = dbgwvr_write / dbgwcr_write
}
```

写入时触发 `hw_breakpoint_update(cpu, n)` 或 `hw_watchpoint_update(cpu, n)` 同步到 QEMU 内部断点/观察点系统。

---

## 3. MDSCR_EL1 — 调试系统控制

### 3.1 寄存器定义

**文件**：`target/arm/debug_helper.c:192-199`

```c
{ .name = "MDSCR_EL1", .state = ARM_CP_STATE_BOTH,
  .cp = 14, .opc0 = 2, .opc1 = 0, .crn = 0, .crm = 2, .opc2 = 2,
  .access = PL1_RW, .accessfn = access_tda,
  .fieldoffset = offsetof(CPUARMState, cp15.mdscr_el1),
  .resetvalue = 0 },
```

### 3.2 关键位域

| 位 | 名称 | 含义 | 使用位置 |
|----|------|------|----------|
| [0] | SS | 软件单步使能 | debug.c:166 `arm_singlestep_active()` |
| [13] | KDE | 内核调试使能（同 EL 调试） | debug.c:86 同 EL 调试条件 |
| [15] | MDE | 监视器调试使能 | debug.c:363,387 断点/观察点全局开关 |

```c
// MDE 控制: 断点/观察点全局使能
if (extract32(env->cp15.mdscr_el1, 15, 1) == 0)  // MDE=0
    return false;  // 断点/观察点被禁用

// KDE 控制: 同 EL 调试
if (cur_el == debug_el) {
    return extract32(env->cp15.mdscr_el1, 13, 1)  // KDE=1
        && !(env->daif & PSTATE_D);                 // DAIF.D=0
}
```

---

## 4. DBGBCR/DBGBVR — 断点控制与值寄存器

### 4.1 DBGBCR 位域

DBGBCR 与 DBGWCR 共享部分布局（`bp_wp_matches` 中复用字段提取）：

| 位 | 名称 | 含义 |
|----|------|------|
| [0] | E | 使能位 |
| [2:1] | PMC | 特权模式控制（EL 匹配） |
| [8:5] | BAS | 字节地址选择 |
| [13] | HMC | 高模式控制（EL2/EL3 匹配） |
| [15:14] | SSC | 安全状态控制 |
| [19:16] | LBN | 链接断点编号 |
| [23:20] | BT | 断点类型 |

### 4.2 断点类型 (BT)

**文件**：`target/arm/tcg/debug.c:671-733`

| BT | 类型 | QEMU 实现 |
|----|------|-----------|
| 0 | 未链接地址匹配 | cpu_breakpoint_insert |
| 1 | 链接地址匹配 | cpu_breakpoint_insert + linked_bp_matches |
| 2 | 未链接上下文 ID 匹配 | LOG_UNIMP（未实现） |
| 3 | 链接上下文 ID 匹配 | 无事件生成 |
| 4 | 未链接地址不匹配 | LOG_UNIMP（AArch64 保留） |
| 5 | 链接地址不匹配 | LOG_UNIMP（AArch64 保留） |
| 8 | 未链接 VMID 匹配 | LOG_UNIMP |
| 10 | 未链接上下文 ID + VMID | LOG_UNIMP |

### 4.3 BAS 字段处理

```c
// debug.c:706-714
int bas = extract64(bcr, 5, 4);
addr = bvr & ~3ULL;         // 低 2 位清零
if (bas == 0) return;        // BAS=0 → 断点禁用
if (bas == 0xc)
    addr += 2;               // BAS=1100 → 16位指令偏移 +2
// BAS=0011 或 0xf → 使用 addr
```

---

## 5. DBGWCR/DBGWVR — 观察点控制与值寄存器

### 5.1 DBGWCR 位域

**文件**：`target/arm/internals.h:111-121`

```c
FIELD(DBGWCR, E, 0, 1)       // 使能
FIELD(DBGWCR, PAC, 1, 2)     // 特权访问控制
FIELD(DBGWCR, LSC, 3, 2)     // Load/Store 控制
FIELD(DBGWCR, BAS, 5, 8)     // 字节地址选择（8位）
FIELD(DBGWCR, HMC, 13, 1)    // 高模式控制
FIELD(DBGWCR, SSC, 14, 2)    // 安全状态控制
FIELD(DBGWCR, LBN, 16, 4)    // 链接断点编号
FIELD(DBGWCR, WT, 20, 1)     // 观察点类型（0=未链接，1=链接）
FIELD(DBGWCR, MASK, 24, 5)   // 地址掩码
FIELD(DBGWCR, SSCE, 29, 1)   // 安全状态控制扩展
```

### 5.2 LSC — Load/Store 控制

| LSC | 含义 | QEMU 标志 |
|-----|------|-----------|
| 00 | 保留（禁用） | return |
| 01 | Load（读） | BP_MEM_READ |
| 10 | Store（写） | BP_MEM_WRITE |
| 11 | Load 和 Store | BP_MEM_ACCESS |

### 5.3 MASK vs BAS

**文件**：`target/arm/tcg/debug.c:580-629`

```c
mask = FIELD_EX64(wcr, DBGWCR, MASK);
if (mask == 1 || mask == 2) return;   // 保留值 → 禁用
if (mask) {
    // MASK 模式: 观察 2^mask 字节对齐区域（最大 2GB）
    len = 1ULL << mask;
    wvr &= ~(len - 1);               // 对齐
} else {
    // BAS 模式: 逐字节选择
    int bas = FIELD_EX64(wcr, DBGWCR, BAS);
    basstart = ctz32(bas);             // 第一个 1 位
    len = cto32(bas >> basstart);      // 连续 1 的个数
    wvr += basstart;
}
cpu_watchpoint_insert(CPU(cpu), wvr, len, flags, &env->cpu_watchpoint[n]);
```

---

## 6. 调试异常路由

### 6.1 arm_debug_target_el

**文件**：`target/arm/tcg/debug.c:18-41`

```c
static int arm_debug_target_el(CPUARMState *env) {
    bool route_to_el2 = false;

    if (arm_is_el2_enabled(env)) {
        route_to_el2 = env->cp15.hcr_el2 & HCR_TGE      // TGE=1
                    || env->cp15.mdcr_el2 & MDCR_TDE;     // MDCR_EL2.TDE=1
    }

    if (route_to_el2) return 2;
    else if (arm_feature(ARM_FEATURE_EL3) && !arm_el_is_aa64(env, 3) && secure)
        return 3;              // Secure + 32位 EL3 → EL3
    else return 1;             // 默认 EL1
}
```

### 6.2 raise_exception_debug

**文件**：`target/arm/tcg/debug.c:47-61`

```c
static void raise_exception_debug(CPUARMState *env, uint32_t excp, uint32_t syndrome) {
    int debug_el = arm_debug_target_el(env);
    int cur_el = arm_current_el(env);
    assert(debug_el >= cur_el);

    // 同 EL 异常: 设置 EC 最高位（Same-EL 变体）
    syndrome |= (debug_el == cur_el) << R_SYNDROME_EC_SHIFT;
    // 例: EC_BREAKPOINT(0x30) → EC_BREAKPOINT_SAME_EL(0x31)
    raise_exception(env, excp, syndrome, debug_el);
}
```

### 6.3 路由控制寄存器

| 寄存器 | 位 | 效果 |
|--------|-----|------|
| MDCR_EL2.TDE | [8] | 调试异常路由到 EL2 |
| HCR_EL2.TGE | [27] | 通用陷入 EL2（包括调试） |
| MDCR_EL3.SDD | [16] | 安全调试禁用（Secure 状态不产生调试异常） |
| MDSCR_EL1.KDE | [13] | 同 EL 调试使能（需要 + DAIF.D=0） |
| MDSCR_EL1.MDE | [15] | 断点/观察点全局使能 |

---

## 7. 调试异常使能条件

### 7.1 arm_generate_debug_exceptions

**文件**：`target/arm/tcg/debug.c:148-158`

```c
bool arm_generate_debug_exceptions(CPUARMState *env) {
    // OS Lock / DoubleLock 激活 → 禁止所有调试异常
    if ((env->cp15.oslsr_el1 & 1) || (env->cp15.osdlr_el1 & 1))
        return false;

    if (is_a64(env))
        return aa64_generate_debug_exceptions(env);
    else
        return aa32_generate_debug_exceptions(env);
}
```

### 7.2 aa64_generate_debug_exceptions

**文件**：`target/arm/tcg/debug.c:64-92`

```c
static bool aa64_generate_debug_exceptions(CPUARMState *env) {
    int cur_el = arm_current_el(env);

    // EL3 不产生调试异常
    if (cur_el == 3) return false;

    // MDCR_EL3.SDD=1: Secure 状态禁止调试
    if (arm_is_secure_below_el3(env) && extract32(env->cp15.mdcr_el3, 16, 1))
        return false;

    int debug_el = arm_debug_target_el(env);

    if (cur_el == debug_el) {
        // 同 EL: 需要 KDE=1 且 DAIF.D=0
        return extract32(env->cp15.mdscr_el1, 13, 1)  // KDE
            && !(env->daif & PSTATE_D);                 // !D
    }

    // 高 EL: debug_el > cur_el → 允许
    return debug_el > cur_el;
}
```

**使能条件总结**：

| 场景 | 条件 |
|------|------|
| EL0 → EL1 调试 | 始终允许（debug_el=1 > cur_el=0） |
| EL1 → EL1 调试 | MDSCR.KDE=1 && DAIF.D=0 |
| EL0/EL1 → EL2 调试 | MDCR_EL2.TDE=1 或 HCR.TGE=1 |
| Secure → 禁止 | MDCR_EL3.SDD=1 |
| EL3 | 始终禁止 |
| OS Lock | OSLSR_EL1[0]=1 → 全部禁止 |

---

## 8. 硬件断点实现

### 8.1 hw_breakpoint_update

**文件**：`target/arm/tcg/debug.c:652-736`

当 DBGBCR/DBGBVR 被写入时调用，将架构断点同步到 QEMU 内部系统：

```c
void hw_breakpoint_update(ARMCPU *cpu, int n) {
    uint64_t bvr = env->cp15.dbgbvr[n];
    uint64_t bcr = env->cp15.dbgbcr[n];

    // 移除旧断点
    if (env->cpu_breakpoint[n])
        cpu_breakpoint_remove_by_ref(CPU(cpu), env->cpu_breakpoint[n]);

    // E=0 → 禁用
    if (!extract64(bcr, 0, 1)) return;

    // 检查断点类型 BT[23:20]
    int bt = extract64(bcr, 20, 4);
    switch (bt) {
    case 0: case 1:  // 地址匹配（链接/未链接）
        addr = bvr & ~3ULL;
        int bas = extract64(bcr, 5, 4);
        if (bas == 0) return;
        if (bas == 0xc) addr += 2;   // Thumb 半字偏移
        break;
    case 2: case 8: case 10:         // 上下文/VMID 匹配 → 未实现
        return;
    default:                          // 链接上下文/保留 → 忽略
        return;
    }

    cpu_breakpoint_insert(CPU(cpu), addr, BP_CPU, &env->cpu_breakpoint[n]);
}
```

### 8.2 arm_debug_check_breakpoint

**文件**：`target/arm/tcg/debug.c:376-420`

TB 执行前由 QEMU 核心调用：

```c
bool arm_debug_check_breakpoint(CPUState *cs) {
    // 全局检查: MDE=0 → 禁用
    if (extract32(mdscr_el1, 15, 1) == 0) return false;
    // 调试异常不可用 → 禁用
    if (!arm_generate_debug_exceptions(env)) return false;

    // 单步 Active-pending 优先级高于断点
    if (arm_singlestep_active(env) && !(env->pstate & PSTATE_SS))
        return false;

    // PC 对齐错误优先级高于断点
    if ((is_a64(env) || !env->thumb) && (pc & 3) != 0) return false;

    // 遍历所有断点
    for (n = 0; n < ARRAY_SIZE(env->cpu_breakpoint); n++) {
        if (bp_wp_matches(cpu, n, false)) return true;
    }
    return false;
}
```

---

## 9. 硬件观察点实现

### 9.1 hw_watchpoint_update

**文件**：`target/arm/tcg/debug.c:546-633`

```c
void hw_watchpoint_update(ARMCPU *cpu, int n) {
    vaddr wvr = env->cp15.dbgwvr[n];
    uint64_t wcr = env->cp15.dbgwcr[n];
    int flags = BP_CPU | BP_STOP_BEFORE_ACCESS;

    // E=0 → 禁用
    if (!FIELD_EX64(wcr, DBGWCR, E)) return;

    // LSC 选择 Load/Store/Both
    switch (FIELD_EX64(wcr, DBGWCR, LSC)) {
    case 0: return;                    // 保留 → 禁用
    case 1: flags |= BP_MEM_READ;     // Load
    case 2: flags |= BP_MEM_WRITE;    // Store
    case 3: flags |= BP_MEM_ACCESS;   // Both
    }

    // MASK vs BAS (详见第 5.3 节)
    // ... 计算 wvr + len ...

    cpu_watchpoint_insert(CPU(cpu), wvr, len, flags, &env->cpu_watchpoint[n]);
}
```

### 9.2 check_watchpoints / arm_debug_check_watchpoint

**文件**：`target/arm/tcg/debug.c:354-431`

```c
static bool check_watchpoints(ARMCPU *cpu) {
    // MDE=0 → 禁用
    if (extract32(mdscr_el1, 15, 1) == 0) return false;
    if (!arm_generate_debug_exceptions(env)) return false;

    for (n = 0; n < ARRAY_SIZE(env->cpu_watchpoint); n++) {
        if (bp_wp_matches(cpu, n, true)) return true;
    }
    return false;
}

// QEMU 核心 watchpoint 命中时的回调
bool arm_debug_check_watchpoint(CPUState *cs, CPUWatchpoint *wp) {
    return check_watchpoints(ARM_CPU(cs));
}
```

---

## 10. 断点/观察点匹配核心逻辑

### 10.1 bp_wp_matches

**文件**：`target/arm/tcg/debug.c:255-352`

这是断点和观察点共用的匹配核心函数：

```c
static bool bp_wp_matches(ARMCPU *cpu, int n, bool is_wp) {
    // 1. 基础匹配检查
    if (is_wp) {
        // 观察点: 检查 cpu_watchpoint 已命中 (BP_WATCHPOINT_HIT)
        cr = env->cp15.dbgwcr[n];
    } else {
        // 断点: 检查 PC 匹配
        if (env->cpu_breakpoint[n]->pc != pc) return false;
        cr = env->cp15.dbgbcr[n];
    }

    // 2. SSC — 安全状态检查
    pac = FIELD_EX64(cr, DBGWCR, PAC);
    hmc = FIELD_EX64(cr, DBGWCR, HMC);
    ssc = FIELD_EX64(cr, DBGWCR, SSC);

    switch (ssc) {
    case 0: break;                        // 任意安全状态
    case 1: case 3: if (secure) return false; break;  // 仅非安全
    case 2: if (!secure) return false; break;          // 仅安全
    }

    // 3. EL 匹配 (HMC + PAC/PMC)
    switch (access_el) {
    case 3: case 2: if (!hmc) return false; break;  // EL2/3 需要 HMC=1
    case 1: if (!(pac & 1)) return false; break;     // EL1 需要 PAC[0]=1
    case 0: if (!(pac & 2)) return false; break;     // EL0 需要 PAC[1]=1
    }

    // 4. 链接断点检查
    wt = FIELD_EX64(cr, DBGWCR, WT);
    lbn = FIELD_EX64(cr, DBGWCR, LBN);
    if (wt && !linked_bp_matches(cpu, lbn)) return false;

    return true;
}
```

### 10.2 匹配流程图

```
地址命中 (QEMU 内部 bp/wp 系统)
  ↓
bp_wp_matches()
  ├── SSC 安全状态过滤
  ├── HMC + PAC/PMC EL 权限过滤
  ├── WT + LBN 链接断点验证
  └── 全部通过 → true
  ↓
arm_debug_check_breakpoint / check_watchpoints
  ├── MDE 全局开关
  ├── arm_generate_debug_exceptions 使能检查
  └── 单步优先级 / PC 对齐优先级
  ↓
arm_debug_excp_handler → raise_exception_debug
```

---

## 11. 调试异常处理入口

### 11.1 arm_debug_excp_handler

**文件**：`target/arm/tcg/debug.c:464-508`

QEMU 核心在断点/观察点命中时调用：

```c
void arm_debug_excp_handler(CPUState *cs) {
    CPUWatchpoint *wp_hit = cs->watchpoint_hit;

    if (wp_hit) {
        // 观察点命中
        if (wp_hit->flags & BP_CPU) {
            bool wnr = (wp_hit->flags & BP_WATCHPOINT_HIT_WRITE) != 0;
            env->exception.fsr = arm_debug_exception_fsr(env);
            env->exception.vaddress = wp_hit->hitaddr;
            raise_exception_debug(env, EXCP_DATA_ABORT,
                                  syn_watchpoint(0, 0, wnr));
        }
    } else {
        // 断点命中
        // GDB 断点优先处理
        if (cpu_breakpoint_test(cs, pc, BP_GDB)
            || !cpu_breakpoint_test(cs, pc, BP_CPU))
            return;  // GDB 处理或非 CPU 断点

        env->exception.fsr = arm_debug_exception_fsr(env);
        env->exception.vaddress = 0;  // FAR 为 UNKNOWN
        raise_exception_debug(env, EXCP_PREFETCH_ABORT,
                              syn_breakpoint(0));
    }
}
```

### 11.2 异常类型映射

| 调试事件 | EXCP_ 常量 | 综合征 EC | Same-EL EC |
|----------|-----------|----------|------------|
| 硬件断点 | EXCP_PREFETCH_ABORT | 0x30 | 0x31 |
| 硬件观察点 | EXCP_DATA_ABORT | 0x34 | 0x35 |
| 软件断点 BRK | EXCP_BKPT | 0x3c | 0x3c（无 Same-EL 变体） |
| 软件单步 | EXCP_UDEF | 0x32 | 0x33 |

---

## 12. 软件断点 BRK 指令

### 12.1 翻译阶段

**文件**：`target/arm/tcg/translate-a64.c:504-509`

```c
static void gen_exception_bkpt_insn(DisasContext *s, uint32_t syndrome) {
    gen_a64_update_pc(s, 0);
    gen_helper_exception_bkpt_insn(tcg_env, tcg_constant_i32(syndrome));
    s->base.is_jmp = DISAS_NORETURN;
}
```

### 12.2 Helper 实现

**文件**：`target/arm/tcg/debug.c:514-539`

```c
void HELPER(exception_bkpt_insn)(CPUARMState *env, uint32_t syndrome) {
    int debug_el = arm_debug_target_el(env);
    int cur_el = arm_current_el(env);

    env->exception.fsr = arm_debug_exception_fsr(env);
    env->exception.vaddress = 0;

    // BRK 特殊: 如果 debug_el < cur_el，仍然在 cur_el 处理
    // （其他调试异常在此情况下被忽略）
    if (debug_el < cur_el)
        debug_el = cur_el;

    raise_exception(env, EXCP_BKPT, syndrome, debug_el);
}
```

**BRK vs 硬件断点的关键区别**：
- BRK 始终产生异常（至少在当前 EL），不依赖 MDE/KDE
- 硬件断点需要 MDE=1，同 EL 还需 KDE=1 + DAIF.D=0
- BRK 的 EC 是 0x3c (EC_AA64_BKPT)，没有 Same-EL 变体

---

## 13. 单步执行机制

### 13.1 arm_singlestep_active

**文件**：`target/arm/tcg/debug.c:164-169`

```c
bool arm_singlestep_active(CPUARMState *env) {
    return extract32(env->cp15.mdscr_el1, 0, 1)       // MDSCR.SS=1
        && arm_el_is_aa64(env, arm_debug_target_el(env)) // 调试目标 EL 是 AArch64
        && arm_generate_debug_exceptions(env);            // 调试异常使能
}
```

### 13.2 单步状态机

**文件**：`target/arm/tcg/hflags.c:678-688`

ARM 架构定义的三态状态机：

| SS_ACTIVE | PSTATE.SS | 状态 | 含义 |
|-----------|-----------|------|------|
| 0 | x | Inactive | 单步未激活 |
| 1 | 1 | Active-not-pending | 等待执行一条指令 |
| 1 | 0 | Active-pending | 已执行一条指令，待产生异常 |

```
            ┌──────────────┐
            │   Inactive   │ (SS_ACTIVE=0)
            └──────┬───────┘
                   │ MDSCR.SS=1 + 调试使能 + ERET 或异常入口
                   ↓
       ┌───────────────────────┐
       │  Active-not-pending   │ (SS_ACTIVE=1, PSTATE.SS=1)
       └───────────┬───────────┘
                   │ 执行一条指令 (gen_ss_advance: PSTATE.SS ← 0)
                   ↓
       ┌───────────────────────┐
       │   Active-pending      │ (SS_ACTIVE=1, PSTATE.SS=0)
       └───────────┬───────────┘
                   │ gen_swstep_exception → 软件单步异常
                   ↓
            ┌──────────────┐
            │   Inactive   │
            └──────────────┘
```

### 13.3 gen_ss_advance — 状态推进

**文件**：`target/arm/tcg/translate.h:414-421`

```c
static inline void gen_ss_advance(DisasContext *s) {
    if (s->ss_active) {
        s->pstate_ss = 0;
        clear_pstate_bits(PSTATE_SS);  // Active-not-pending → Active-pending
    }
}
```

### 13.4 gen_step_complete_exception — 完成单步

**文件**：`target/arm/tcg/translate-a64.c:511-525`

```c
static void gen_step_complete_exception(DisasContext *s) {
    // 指令执行完成后:
    gen_ss_advance(s);                          // 推进状态机
    gen_swstep_exception(s, 1, s->is_ldex);    // ISV=1, EX=is_ldex
    s->base.is_jmp = DISAS_NORETURN;
}
```

### 13.5 翻译时单步处理

**文件**：`target/arm/tcg/translate-a64.c:10744-10797`

```c
// 初始化
dc->ss_active = EX_TBFLAG_ANY(tb_flags, SS_ACTIVE);
dc->pstate_ss = EX_TBFLAG_ANY(tb_flags, PSTATE__SS);

// 单步激活时限制 TB 为单条指令
if (dc->ss_active) {
    bound = 1;    // 每个 TB 最多 1 条指令
}

// 翻译每条指令时检查
if (s->ss_active && !s->pstate_ss) {
    // Active-pending 状态: 在执行前产生异常
    // (场景: 异常处理程序的第一条指令或刚解除 DAIF.D 屏蔽)
    gen_swstep_exception(s, 0, 0);  // ISV=0, EX=0 (未执行指令)
    s->base.is_jmp = DISAS_NORETURN;
    return;
}
```

### 13.6 单步禁止 TB 链接

**文件**：`target/arm/tcg/translate-a64.c:527-530`

```c
static inline bool use_goto_tb(DisasContext *s, uint64_t dest) {
    if (s->ss_active) return false;  // 单步时禁止 TB 链接
    ...
}
```

---

## 14. 翻译时调试检查与 TB 标志

### 14.1 hflags 中的调试标志

**文件**：`target/arm/tcg/hflags.c:77-88, 675-688`

```c
// rebuild_hflags_common: 每次 hflags 重建时计算
if (arm_singlestep_active(env)) {
    DP_TBFLAG_ANY(flags, SS_ACTIVE, 1);  // 编码到 TB 标志
}

// aarch64_cpu_get_tb_cpu_state: 每个 TB 开始时计算
if (EX_TBFLAG_ANY(flags, SS_ACTIVE) && (env->pstate & PSTATE_SS)) {
    DP_TBFLAG_ANY(flags, PSTATE__SS, 1);  // PSTATE.SS 当前值
}
```

### 14.2 TB 标志对翻译的影响

| 标志 | 效果 |
|------|------|
| SS_ACTIVE=1 | TB 限制为 1 条指令，禁止 TB 链接 |
| PSTATE__SS=1 | Active-not-pending，执行后推进 |
| PSTATE__SS=0 + SS_ACTIVE=1 | Active-pending，执行前立即产生异常 |
| SS_ACTIVE=0 | 正常翻译，不受单步影响 |

### 14.3 HELPER(exception_swstep)

**文件**：`target/arm/tcg/debug.c:541-544`

```c
void HELPER(exception_swstep)(CPUARMState *env, uint32_t syndrome) {
    raise_exception_debug(env, EXCP_UDEF, syndrome);
    // → 路由到 arm_debug_target_el，EC 自动加 Same-EL 位
}
```

单步异常使用 `EXCP_UDEF` 但综合征编码为 `EC_SOFTWARESTEP (0x32)` 或 `EC_SOFTWARESTEP_SAME_EL (0x33)`。

---

## 15. 总结与关键设计

### 15.1 QEMU ARM64 调试子系统架构

```
┌─────────────────────────────────────────────────────────┐
│                    Guest OS / Debugger                    │
│  写入 MDSCR_EL1 / DBGBCR / DBGBVR / DBGWCR / DBGWVR   │
└─────────────────────┬───────────────────────────────────┘
                      │ cpreg writefn
                      ↓
┌─────────────────────────────────────────────────────────┐
│  hw_breakpoint_update / hw_watchpoint_update             │
│  → cpu_breakpoint_insert / cpu_watchpoint_insert         │
│  (QEMU 内部 bp/wp 系统)                                  │
└─────────────────────┬───────────────────────────────────┘
                      │ TB 执行时触发
                      ↓
┌─────────────────────────────────────────────────────────┐
│  arm_debug_check_breakpoint / arm_debug_check_watchpoint │
│  → bp_wp_matches (SSC/HMC/PAC + linked_bp + address)    │
│  → arm_debug_excp_handler                                │
│  → raise_exception_debug (syn_breakpoint/syn_watchpoint) │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│  arm_cpu_do_interrupt → VBAR + Sync 偏移                 │
│  ESR_ELx = EC_BREAKPOINT / EC_WATCHPOINT / EC_SWSTEP    │
└─────────────────────────────────────────────────────────┘
```

### 15.2 关键设计特点

| 设计 | 说明 |
|------|------|
| cpreg writefn 同步 | DBGBCR/DBGWCR 写入立即同步到 QEMU bp/wp 系统 |
| 共享匹配逻辑 | bp_wp_matches 断点和观察点复用 BCR/WCR 相同布局 |
| 优先级规则 | 单步 > 断点 > 观察点 > 普通指令 |
| Same-EL EC 位 | 运行时通过 `(debug_el == cur_el) << EC_SHIFT` 设置 |
| TB 标志驱动 | SS_ACTIVE + PSTATE__SS 编码到 TB 标志，影响翻译 |
| 单指令 TB | 单步激活时强制每 TB 1 条指令 + 禁止链接 |
| BRK 特殊性 | 始终产生异常，不依赖 MDE/KDE，目标 EL 不低于 cur_el |
| GDB 共存 | BP_GDB vs BP_CPU 区分 GDB 断点和架构断点 |

### 15.3 核心源文件索引

| 文件 | 关键内容 | 行范围 |
|------|----------|--------|
| tcg/debug.c | arm_debug_target_el | 18-41 |
| tcg/debug.c | raise_exception_debug | 47-61 |
| tcg/debug.c | aa64_generate_debug_exceptions | 64-92 |
| tcg/debug.c | arm_generate_debug_exceptions | 148-158 |
| tcg/debug.c | arm_singlestep_active | 164-169 |
| tcg/debug.c | bp_wp_matches | 255-352 |
| tcg/debug.c | check_watchpoints | 354-374 |
| tcg/debug.c | arm_debug_check_breakpoint | 376-420 |
| tcg/debug.c | arm_debug_excp_handler | 464-508 |
| tcg/debug.c | HELPER(exception_bkpt_insn) | 514-539 |
| tcg/debug.c | HELPER(exception_swstep) | 541-544 |
| tcg/debug.c | hw_watchpoint_update | 546-633 |
| tcg/debug.c | hw_breakpoint_update | 652-736 |
| debug_helper.c | MDSCR_EL1 cpreg 定义 | 192-199 |
| debug_helper.c | DBGBCR/DBGBVR 注册循环 | 477-499 |
| debug_helper.c | DBGWCR/DBGWVR 注册循环 | 501-523 |
| internals.h | DBGWCR 位域定义 | 111-121 |
| cpu.h | dbgbvr/dbgbcr/dbgwvr/dbgwcr 数组 | 529-538 |
| translate.h | gen_ss_advance / gen_swstep_exception | 414-429 |
| translate-a64.c | gen_step_complete_exception | 511-525 |
| translate-a64.c | ss_active / pstate_ss 初始化 | 10744-10754 |
| translate-a64.c | Active-pending 异常生成 | 10781-10797 |
| hflags.c | SS_ACTIVE 编码 | 84-86 |
| hflags.c | PSTATE__SS 编码 | 678-688 |
