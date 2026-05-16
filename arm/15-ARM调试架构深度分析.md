# ARM 调试架构深度分析

> 基于 QEMU 11.0.50 源码，分析 ARM 自托管调试架构的完整实现  
> 包括硬件断点/监视点、软件单步、调试异常、BRK/BKPT、GDB/KVM 集成

---

## 目录

1. [概述](#1-概述)
2. [CPUARMState 调试寄存器](#2-cpuarmstate-调试寄存器)
3. [调试寄存器定义文件](#3-调试寄存器定义文件)
4. [MDSCR_EL1 调试系统控制](#4-mdscr_el1-调试系统控制)
5. [OSLAR/OSLSR/OSDLR OS 锁](#5-oslaroslsrosdlr-os-锁)
6. [调试寄存器访问控制](#6-调试寄存器访问控制)
7. [access_tdosa/tdra/tda 陷阱](#7-access_tdosatdratda-陷阱)
8. [DBGBVR/DBGBCR 断点寄存器](#8-dbgbvrdbgbcr-断点寄存器)
9. [dbgbvr_write/dbgbcr_write 实现](#9-dbgbvr_writedbgbcr_write-实现)
10. [DBGWVR/DBGWCR 监视点寄存器](#10-dbgwvrdbgwcr-监视点寄存器)
11. [dbgwvr_write/dbgwcr_write 实现](#11-dbgwvr_writedbgwcr_write-实现)
12. [define_debug_regs() 寄存器注册](#12-define_debug_regs-寄存器注册)
13. [调试目标异常级别](#13-调试目标异常级别)
14. [调试异常生成条件](#14-调试异常生成条件)
15. [AArch64 调试异常生成](#15-aarch64-调试异常生成)
16. [AArch32 调试异常生成](#16-aarch32-调试异常生成)
17. [arm_generate_debug_exceptions()](#17-arm_generate_debug_exceptions)
18. [硬件断点匹配逻辑](#18-硬件断点匹配逻辑)
19. [bp_wp_matches() 核心匹配](#19-bp_wp_matches-核心匹配)
20. [linked_bp_matches() 链式断点](#20-linked_bp_matches-链式断点)
21. [hw_breakpoint_update() TCG 绑定](#21-hw_breakpoint_update-tcg-绑定)
22. [硬件监视点匹配逻辑](#22-硬件监视点匹配逻辑)
23. [hw_watchpoint_update() TCG 绑定](#23-hw_watchpoint_update-tcg-绑定)
24. [arm_debug_check_breakpoint()](#24-arm_debug_check_breakpoint)
25. [arm_debug_check_watchpoint()](#25-arm_debug_check_watchpoint)
26. [arm_debug_excp_handler() 异常处理](#26-arm_debug_excp_handler-异常处理)
27. [BRK 指令 (AArch64)](#27-brk-指令-aarch64)
28. [BKPT 指令 (AArch32)](#28-bkpt-指令-aarch32)
29. [HELPER(exception_bkpt_insn)](#29-helperexception_bkpt_insn)
30. [软件单步实现](#30-软件单步实现)
31. [arm_singlestep_active()](#31-arm_singlestep_active)
32. [TCG 单步翻译](#32-tcg-单步翻译)
33. [调试异常综合征](#33-调试异常综合征)
34. [arm_debug_exception_fsr()](#34-arm_debug_exception_fsr)
35. [DCC 通信通道](#35-dcc-通信通道)
36. [DBGAUTHSTATUS 调试认证](#36-dbgauthstatus-调试认证)
37. [MDCR_EL2 Hypervisor 调试控制](#37-mdcr_el2-hypervisor-调试控制)
38. [MDCR_EL3 安全调试控制](#38-mdcr_el3-安全调试控制)
39. [GDB 远程调试集成](#39-gdb-远程调试集成)
40. [KVM 调试支持](#40-kvm-调试支持)
41. [完整断点触发流程](#41-完整断点触发流程)
42. [完整监视点触发流程](#42-完整监视点触发流程)
43. [调试异常优先级](#43-调试异常优先级)
44. [关键数据结构汇总](#44-关键数据结构汇总)
45. [总结](#45-总结)

---

## 1. 概述

ARM 调试架构提供**自托管调试 (self-hosted debug)** 能力，允许运行在 CPU 上的软件（如 OS 调试器或 Hypervisor）使用硬件断点和监视点。QEMU 实现了完整的自托管调试框架：

- **16 个硬件断点** (DBGBVR/DBGBCR) — 地址匹配断点
- **16 个硬件监视点** (DBGWVR/DBGWCR) — 数据访问监视点
- **软件单步** — MDSCR.SS + PSTATE.SS
- **BRK/BKPT 指令** — 软件断点
- **OS 锁** — OSLAR/OSLSR/OSDLR
- **调试异常路由** — 目标 EL 由 MDCR_EL2/EL3 控制

**关键源文件**：
- `target/arm/debug_helper.c` — 调试系统寄存器定义和访问控制
- `target/arm/tcg/debug.c` — 断点/监视点匹配、调试异常处理

---

## 2. CPUARMState 调试寄存器

```c
// cpu.h:529-538
uint64_t dbgbvr[16];   /* breakpoint value registers */
uint64_t dbgbcr[16];   /* breakpoint control registers */
uint64_t dbgwvr[16];   /* watchpoint value registers */
uint64_t dbgwcr[16];   /* watchpoint control registers */
uint64_t dbgclaim;     /* DBGCLAIM bits */
uint64_t mdscr_el1;    /* Monitor Debug System Control */
uint64_t oslsr_el1;    /* OS Lock Status */
uint64_t osdlr_el1;    /* OS DoubleLock status */
uint64_t mdcr_el2;     /* Hyp Debug Control */
uint64_t mdcr_el3;     /* Secure Debug Control */
```

16 对断点 + 16 对监视点是 ARM 架构允许的最大值。实际数量由 CPU 模型决定（`arm_num_brps(cpu)` / `arm_num_wrps(cpu)`）。

---

## 3. 调试寄存器定义文件

调试寄存器分布在两个源文件中：

| 文件 | 内容 |
|------|------|
| `debug_helper.c` | 系统寄存器定义（MDSCR、OSLAR、DBGBVR/BCR/WVR/WCR 等）、访问控制函数 |
| `tcg/debug.c` | 断点/监视点匹配逻辑、调试异常处理、TCG 断点/监视点绑定 |

---

## 4. MDSCR_EL1 调试系统控制

```c
// debug_helper.c:192-199
{ .name = "MDSCR_EL1", .state = ARM_CP_STATE_BOTH,
  .cp = 14, .opc0 = 2, .opc1 = 0, .crn = 0, .crm = 2, .opc2 = 2,
  .access = PL1_RW, .accessfn = access_tda,
  .fieldoffset = offsetof(CPUARMState, cp15.mdscr_el1),
  .resetvalue = 0 },
```

**关键位**：

| 位 | 名称 | 功能 |
|----|------|------|
| [0] | SS | 软件单步使能 |
| [13] | KDE | 内核调试使能（同 EL 调试异常）|
| [15] | MDE | 监控调试使能（断点/监视点全局使能）|

- **MDE=1**：硬件断点和监视点才能触发调试异常
- **KDE=1**：允许同 EL 的调试异常（否则只有高 EL 可接收）
- **SS=1**：使能软件单步

AArch32 别名：`DBGDSCRint`（debug_helper.c:251-255）

---

## 5. OSLAR/OSLSR/OSDLR OS 锁

```c
// debug_helper.c:256-274
{ .name = "OSLAR_EL1", ... .writefn = oslar_write },
{ .name = "OSLSR_EL1", ... .resetvalue = 10,
  .fieldoffset = offsetof(CPUARMState, cp15.oslsr_el1) },
{ .name = "OSDLR_EL1", ... .writefn = osdlr_write,
  .fieldoffset = offsetof(CPUARMState, cp15.osdlr_el1) },
```

```c
// debug_helper.c:123-139
static void oslar_write(CPUARMState *env, const ARMCPRegInfo *ri,
                        uint64_t value)
{
    if (value & 1) {
        env->cp15.oslsr_el1 = 1 | (1 << 3);  // 上锁
    } else {
        env->cp15.oslsr_el1 = 1 << 3;         // 解锁
    }
}
```

**OS 锁机制**：
- 写 OSLAR_EL1 = 1 上锁，= 0 解锁
- 锁定状态下，调试异常被禁止（`arm_generate_debug_exceptions` 返回 false）
- OSLSR_EL1 反映当前锁定状态
- OSDLR_EL1 是双重锁定（类似 OS 锁，额外一层）
- 用于 OS 电源管理期间防止调试干扰

---

## 6. 调试寄存器访问控制

调试寄存器的访问被三组控制位保护：

| 控制位 | 保护范围 | 检查函数 |
|--------|---------|---------|
| MDCR_EL2.TDOSA / TDE / HCR.TGE | 电源调试寄存器 (OSLAR/OSLSR/OSDLR) | `access_tdosa()` |
| MDCR_EL2.TDRA / TDE / HCR.TGE | 调试 ROM 寄存器 | `access_tdra()` |
| MDCR_EL2.TDA / TDE / HCR.TGE | 通用调试寄存器 (MDSCR/DBGB*/DBGW*) | `access_tda()` |
| MDCR_EL2.TDCC / TDE | 调试通信通道 | `access_tdcc()` |

---

## 7. access_tdosa/tdra/tda 陷阱

```c
// debug_helper.c:21-36
static CPAccessResult access_tdosa(CPUARMState *env, ...)
{
    int el = arm_current_el(env);
    bool mdcr_el2_tdosa = (mdcr_el2 & MDCR_TDOSA) ||
                           (mdcr_el2 & MDCR_TDE) ||
                           (arm_hcr_el2_eff(env) & HCR_TGE);
    if (el < 2 && mdcr_el2_tdosa)
        return CP_ACCESS_TRAP_EL2;
    if (el < 3 && (mdcr_el3 & MDCR_TDOSA))
        return CP_ACCESS_TRAP_EL3;
    return CP_ACCESS_OK;
}
```

**陷阱逻辑**：TDE (Trap Debug Exceptions) 是总开关——当 TDE=1 时，所有调试寄存器访问陷入 EL2。HCR_EL2.TGE=1 也有同等效果。各个细粒度位 (TDOSA/TDRA/TDA/TDCC) 可独立控制。

---

## 8. DBGBVR/DBGBCR 断点寄存器

**DBGBVR_EL1[n]** — 断点值寄存器（64 位地址）：
- 存储要匹配的虚拟地址

**DBGBCR_EL1[n]** — 断点控制寄存器：

| 位域 | 名称 | 功能 |
|------|------|------|
| [0] | E | 断点使能 |
| [8:5] | BAS | 字节地址选择（16 位指令支持）|
| [15:14] | SSC | 安全状态控制 |
| [13] | HMC | 高模式控制（EL2/EL3 匹配）|
| [2:1] | PAC/PMC | 特权访问控制（EL0/EL1 匹配）|
| [23:20] | BT | 断点类型（地址匹配/上下文匹配/链式）|
| [19:16] | LBN | 链式断点编号 |
| [4:3] | LSC/WT | 链式/类型控制 |

---

## 9. dbgbvr_write/dbgbcr_write 实现

```c
// debug_helper.c:371-400
static void dbgbvr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                         uint64_t value)
{
    ARMCPU *cpu = env_archcpu(env);
    int i = ri->crm;
    raw_write(env, ri, value);
    if (tcg_enabled()) {
        hw_breakpoint_update(cpu, i);  // 同步到 QEMU 断点
    }
}

static void dbgbcr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                         uint64_t value)
{
    // BAS[3] = BAS[2] 的只读副本, BAS[1] = BAS[0] 的只读副本
    value = deposit64(value, 6, 1, extract64(value, 5, 1));
    value = deposit64(value, 8, 1, extract64(value, 7, 1));

    raw_write(env, ri, value);
    if (tcg_enabled()) {
        hw_breakpoint_update(cpu, i);
    }
}
```

每次写入 DBGBVR/DBGBCR 都会立即调用 `hw_breakpoint_update()` 同步到 QEMU 的 cpu_breakpoint 系统。

---

## 10. DBGWVR/DBGWCR 监视点寄存器

**DBGWVR_EL1[n]** — 监视点值寄存器（64 位地址，低 2 位 RES0）

**DBGWCR_EL1[n]** — 监视点控制寄存器：

| 位域 | 名称 | 功能 |
|------|------|------|
| [0] | E | 监视点使能 |
| [2:1] | PAC | 特权访问控制 |
| [4:3] | LSC | 加载/存储控制（01=读，10=写，11=读写）|
| [12:5] | BAS | 字节地址选择 |
| [15:14] | SSC | 安全状态控制 |
| [13] | HMC | 高模式控制 |
| [28:24] | MASK | 地址掩码（0=使用 BAS，3-31=掩码 2^MASK 字节）|
| [23:20] | BT/WT | 链式控制 |
| [19:16] | LBN | 链式断点编号 |

---

## 11. dbgwvr_write/dbgwcr_write 实现

```c
// debug_helper.c:334-369
static void dbgwvr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                         uint64_t value)
{
    value &= ~3ULL;  // 低 2 位 RES0
    raw_write(env, ri, value);
    if (tcg_enabled()) {
        hw_watchpoint_update(cpu, i);
    }
}

static void dbgwcr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                         uint64_t value)
{
    raw_write(env, ri, value);
    if (tcg_enabled()) {
        hw_watchpoint_update(cpu, i);
    }
}
```

---

## 12. define_debug_regs() 寄存器注册

```c
// debug_helper.c:402-524
void define_debug_regs(ARMCPU *cpu)
{
    int wrps = arm_num_wrps(cpu);  // 实际监视点数量
    int brps = arm_num_brps(cpu);  // 实际断点数量
    int ctx_cmps;                   // 上下文比较断点数量

    // 动态定义 DBGBVRn/DBGBCRn (n=0..brps-1)
    for (i = 0; i < brps; i++) {
        define_one_arm_cp_reg(cpu, &dbgbvr/dbgbcr);
    }
    // 动态定义 DBGWVRn/DBGWCRn (n=0..wrps-1)
    for (i = 0; i < wrps; i++) {
        define_one_arm_cp_reg(cpu, &dbgwvr/dbgwcr);
    }
}
```

断点和监视点寄存器是动态注册的，数量由 CPU 模型的 `brps/wrps` 属性决定。

---

## 13. 调试目标异常级别

```c
// tcg/debug.c:19-41
static int arm_debug_target_el(CPUARMState *env)
{
    bool route_to_el2 = false;

    if (arm_is_el2_enabled(env)) {
        route_to_el2 = env->cp15.hcr_el2 & HCR_TGE ||
                       env->cp15.mdcr_el2 & MDCR_TDE;
    }

    if (route_to_el2) {
        return 2;
    } else if (arm_feature(env, ARM_FEATURE_EL3) &&
               !arm_el_is_aa64(env, 3) && secure) {
        return 3;
    } else {
        return 1;
    }
}
```

**调试目标 EL 决策**：
1. HCR_EL2.TGE=1 或 MDCR_EL2.TDE=1 → 路由到 EL2
2. 安全态 + EL3 是 AArch32 → 路由到 EL3
3. 否则 → 路由到 EL1

---

## 14. 调试异常生成条件

调试异常只在特定条件下才能被生成。核心检查函数：

```c
// tcg/debug.c:148-158
bool arm_generate_debug_exceptions(CPUARMState *env)
{
    if ((env->cp15.oslsr_el1 & 1) || (env->cp15.osdlr_el1 & 1)) {
        return false;  // OS 锁定时禁止调试
    }
    if (is_a64(env)) {
        return aa64_generate_debug_exceptions(env);
    } else {
        return aa32_generate_debug_exceptions(env);
    }
}
```

**阻止条件**：
- OS 锁 (OSLSR.OSLK=1) 或双重锁 (OSDLR.DLK=1)
- 当前 EL ≥ 调试目标 EL 且 KDE=0（同 EL 禁止）
- PSTATE.D=1（调试掩码位）且是同 EL 调试
- Secure 态的安全调试禁止 (MDCR_EL3.SDD)

---

## 15. AArch64 调试异常生成

```c
// tcg/debug.c:64-92
static bool aa64_generate_debug_exceptions(CPUARMState *env)
{
    int cur_el = arm_current_el(env);

    if (cur_el == 3)
        return false;  // EL3 永不产生调试异常

    // MDCR_EL3.SDD 在安全态禁用调试
    if (arm_is_secure_below_el3(env) &&
        extract32(env->cp15.mdcr_el3, 16, 1))
        return false;

    int debug_el = arm_debug_target_el(env);

    if (cur_el == debug_el) {
        // 同 EL: 需要 MDSCR.KDE=1 且 PSTATE.D=0
        return extract32(env->cp15.mdscr_el1, 13, 1)
            && !(env->daif & PSTATE_D);
    }

    // 调试目标必须是更高 EL
    return debug_el > cur_el;
}
```

---

## 16. AArch32 调试异常生成

```c
// tcg/debug.c:94-134
static bool aa32_generate_debug_exceptions(CPUARMState *env)
{
    // 安全态: 检查 SDER.SUIDEN (EL0) 和 SDCR.SPD
    if (arm_is_secure(env)) {
        if (el == 0 && (env->cp15.sder & 1))
            return true;  // SUIDEN 使能安全 EL0 调试

        int spd = extract32(env->cp15.mdcr_el3, 14, 2);
        switch (spd) {
        case 0: case 1: return true;  // QEMU 总是允许
        case 2: return false;
        case 3: return true;
        }
    }
    return el != 2;  // EL2 不产生 AArch32 调试异常
}
```

QEMU 的调试认证始终允许（DBGEN/SPIDEN/NIDEN/SPNIDEN 均视为高电平）。

---

## 17. arm_generate_debug_exceptions()

这是调试异常的总开关，被所有调试事件检查路径调用。它综合了 OS 锁、安全策略、EL 级别和掩码位的检查。

---

## 18. 硬件断点匹配逻辑

断点匹配需要通过多层检查：

1. **全局使能**：MDSCR_EL1.MDE=1（bit[15]）
2. **调试异常可生成**：`arm_generate_debug_exceptions()` 返回 true
3. **单步优先**：如果单步 active-pending，跳过断点
4. **PC 对齐检查**：PC 未对齐优先于断点
5. **逐个检查**：遍历所有断点寄存器

---

## 19. bp_wp_matches() 核心匹配

```c
// tcg/debug.c:255-352
static bool bp_wp_matches(ARMCPU *cpu, int n, bool is_wp)
{
    // 断点: 检查 PC == DBGBVR[n]
    // 监视点: 检查 WP_HIT 标志

    // SSC (安全状态控制) 检查
    switch (ssc) {
    case 1: case 3: if (is_secure) return false; break;
    case 2: if (!is_secure) return false; break;
    }

    // PAC/HMC (特权/模式控制) 检查
    switch (access_el) {
    case 3: case 2: if (!hmc) return false; break;
    case 1: if (!(pac & 1)) return false; break;
    case 0: if (!(pac & 2)) return false; break;
    }

    // 链式断点检查
    if (wt && !linked_bp_matches(cpu, lbn))
        return false;

    return true;
}
```

**匹配条件**：
1. 地址匹配（断点由 QEMU cpu_breakpoint 预过滤，监视点由 BP_WATCHPOINT_HIT 标志）
2. 安全状态匹配（SSC 字段）
3. 特权级别匹配（PAC/HMC 字段）
4. 链式条件满足（如果 WT=1，链接的断点也需匹配）

---

## 20. linked_bp_matches() 链式断点

```c
// tcg/debug.c:171-253
static bool linked_bp_matches(ARMCPU *cpu, int lbn)
```

链式断点允许组合条件：一个地址匹配断点可以链接到一个上下文 ID 匹配断点，只有两者都满足时才触发。QEMU 实现了链式逻辑但标记上下文匹配类型为 `LOG_UNIMP`。

---

## 21. hw_breakpoint_update() TCG 绑定

```c
// tcg/debug.c:652-736
void hw_breakpoint_update(ARMCPU *cpu, int n)
{
    uint64_t bvr = env->cp15.dbgbvr[n];
    uint64_t bcr = env->cp15.dbgbcr[n];

    // 删除旧断点
    if (env->cpu_breakpoint[n])
        cpu_breakpoint_remove_by_ref(CPU(cpu), env->cpu_breakpoint[n]);

    if (!extract64(bcr, 0, 1))  // E=0, 禁用
        return;

    int bt = extract64(bcr, 20, 4);  // 断点类型
    switch (bt) {
    case 0: case 1:  // 地址匹配（实现）
        int bas = extract64(bcr, 5, 4);
        addr = bvr & ~3ULL;
        if (bas == 0xc) addr += 2;  // Thumb 半字
        cpu_breakpoint_insert(CPU(cpu), addr, BP_CPU, &env->cpu_breakpoint[n]);
        break;

    case 4: case 5:  // 地址不匹配（LOG_UNIMP）
    case 2: case 8: case 10:  // 上下文/VMID 匹配（LOG_UNIMP）
    default:  // 链式/保留
        return;
    }
}
```

**实现状态**：
- ✅ 地址匹配断点 (BT=0/1)：完整实现
- ⚠️ 上下文 ID 匹配 (BT=2)：LOG_UNIMP
- ⚠️ 地址不匹配 (BT=4/5)：LOG_UNIMP
- ⚠️ VMID 匹配 (BT=8/10)：LOG_UNIMP

---

## 22. 硬件监视点匹配逻辑

监视点使用 QEMU 的 `cpu_watchpoint` 机制，在 TLB 级别拦截内存访问。

---

## 23. hw_watchpoint_update() TCG 绑定

```c
// tcg/debug.c:546-633
void hw_watchpoint_update(ARMCPU *cpu, int n)
{
    vaddr wvr = env->cp15.dbgwvr[n];
    uint64_t wcr = env->cp15.dbgwcr[n];
    int flags = BP_CPU | BP_STOP_BEFORE_ACCESS;

    // LSC: 加载/存储控制
    switch (FIELD_EX64(wcr, DBGWCR, LSC)) {
    case 0: return;               // 保留 → 禁用
    case 1: flags |= BP_MEM_READ; break;
    case 2: flags |= BP_MEM_WRITE; break;
    case 3: flags |= BP_MEM_ACCESS; break;
    }

    // MASK 字段: 对齐的大区域监视
    int mask = FIELD_EX64(wcr, DBGWCR, MASK);
    if (mask >= 3) {
        len = 1ULL << mask;   // 2^mask 字节对齐区域
        wvr &= ~(len - 1);
    } else if (mask == 0) {
        // BAS 字段: 字节精确选择
        int bas = FIELD_EX64(wcr, DBGWCR, BAS);
        int basstart = ctz32(bas);
        len = cto32(bas >> basstart);
        wvr += basstart;
    }

    cpu_watchpoint_insert(CPU(cpu), wvr, len, flags, &env->cpu_watchpoint[n]);
}
```

**监视点地址范围**：
- **MASK 模式**：掩码 3-31 → 监视 2^MASK 字节对齐区域（最大 2GB）
- **BAS 模式**：字节地址选择，精确到字节级别（8 字节内）
- MASK 和 BAS 同时设置是 CONSTRAINED UNPREDICTABLE，QEMU 忽略 BAS

---

## 24. arm_debug_check_breakpoint()

```c
// tcg/debug.c:376-420
bool arm_debug_check_breakpoint(CPUState *cs)
{
    // 检查 MDSCR.MDE (bit[15])
    if (extract32(env->cp15.mdscr_el1, 15, 1) == 0
        || !arm_generate_debug_exceptions(env))
        return false;

    // 单步 active-pending 优先于断点
    if (arm_singlestep_active(env) && !(env->pstate & PSTATE_SS))
        return false;

    // PC 对齐错误优先于断点
    pc = is_a64(env) ? env->pc : env->regs[15];
    if ((is_a64(env) || !env->thumb) && (pc & 3) != 0)
        return false;

    // 遍历所有断点
    for (n = 0; n < ARRAY_SIZE(env->cpu_breakpoint); n++) {
        if (bp_wp_matches(cpu, n, false))
            return true;
    }
    return false;
}
```

此函数由 QEMU 核心代码在 CPU 断点命中时调用，判断是否是架构断点匹配。

---

## 25. arm_debug_check_watchpoint()

```c
// tcg/debug.c:422-431
bool arm_debug_check_watchpoint(CPUState *cs, CPUWatchpoint *wp)
{
    ARMCPU *cpu = ARM_CPU(cs);
    return check_watchpoints(cpu);
}
```

`check_watchpoints()` 遍历所有监视点，与 `arm_debug_check_breakpoint()` 类似但检查 WP_HIT。

---

## 26. arm_debug_excp_handler() 异常处理

```c
// tcg/debug.c:464-508
void arm_debug_excp_handler(CPUState *cs)
{
    CPUWatchpoint *wp_hit = cs->watchpoint_hit;

    if (wp_hit) {
        if (wp_hit->flags & BP_CPU) {
            // 架构监视点
            bool wnr = (wp_hit->flags & BP_WATCHPOINT_HIT_WRITE) != 0;
            env->exception.fsr = arm_debug_exception_fsr(env);
            env->exception.vaddress = wp_hit->hitaddr;
            raise_exception_debug(env, EXCP_DATA_ABORT, syn_watchpoint(0, 0, wnr));
        }
    } else {
        uint64_t pc = is_a64(env) ? env->pc : env->regs[15];

        // GDB 断点优先
        if (cpu_breakpoint_test(cs, pc, BP_GDB)
            || !cpu_breakpoint_test(cs, pc, BP_CPU))
            return;

        env->exception.fsr = arm_debug_exception_fsr(env);
        env->exception.vaddress = 0;  // FAR is UNKNOWN
        raise_exception_debug(env, EXCP_PREFETCH_ABORT, syn_breakpoint(0));
    }
}
```

**异常处理流程**：
1. 检查是监视点还是断点
2. **监视点**：EXCP_DATA_ABORT + syn_watchpoint 综合征
3. **断点**：先排除 GDB 断点，然后 EXCP_PREFETCH_ABORT + syn_breakpoint
4. FAR（故障地址寄存器）：监视点设为命中地址，断点设为 UNKNOWN (0)

---

## 27. BRK 指令 (AArch64)

BRK 是 AArch64 的软件断点指令，编码 16 位立即数（#imm16）。

```
BRK #imm16
```

翻译时 (`translate-a64.c`)：`trans_BRK()` → `gen_exception_bkpt_insn(syn_aa64_bkpt(imm16))`

综合征：EC=0x3C (BRK from AArch64)，ISS=imm16

---

## 28. BKPT 指令 (AArch32)

BKPT 是 AArch32/Thumb 的软件断点指令。

```
BKPT #imm
```

翻译时 (`translate.c`)：`trans_BKPT()` → `gen_exception_bkpt_insn(syn_aa32_bkpt(imm, false))`

综合征：EC=0x38 (BKPT from AArch32)

---

## 29. HELPER(exception_bkpt_insn)

```c
// tcg/debug.c:514-539
void HELPER(exception_bkpt_insn)(CPUARMState *env, uint32_t syndrome)
{
    int debug_el = arm_debug_target_el(env);
    int cur_el = arm_current_el(env);

    env->exception.fsr = arm_debug_exception_fsr(env);
    env->exception.vaddress = 0;

    // BRK/BKPT 特殊：如果调试目标 EL < 当前 EL，
    // 则在当前 EL 处理（不像其他调试异常会被抑制）
    if (debug_el < cur_el) {
        debug_el = cur_el;
    }
    raise_exception(env, EXCP_BKPT, syndrome, debug_el);
}
```

**BRK/BKPT 的特殊性**：它们总是产生异常，不受调试使能条件限制。如果配置的调试目标 EL 低于当前 EL，异常在当前 EL 处理。

---

## 30. 软件单步实现

ARM 软件单步通过 MDSCR_EL1.SS + PSTATE.SS 实现：

| MDSCR.SS | PSTATE.SS | 状态 | 行为 |
|----------|-----------|------|------|
| 0 | - | 禁用 | 无单步 |
| 1 | 1 | Active-not-pending | 执行一条指令后设 SS=0 |
| 1 | 0 | Active-pending | 下条指令前产生单步异常 |

---

## 31. arm_singlestep_active()

```c
// tcg/debug.c:164-169
bool arm_singlestep_active(CPUARMState *env)
{
    return extract32(env->cp15.mdscr_el1, 0, 1)    // MDSCR.SS=1
        && arm_el_is_aa64(env, arm_debug_target_el(env))  // 目标EL是AArch64
        && arm_generate_debug_exceptions(env);       // 调试异常可生成
}
```

注意：软件单步仅在目标 EL 为 AArch64 时有效（ARMv8 架构限制）。

---

## 32. TCG 单步翻译

在 TCG 翻译时：

```c
// translate-a64.c:511-563
// dc->ss_active = arm_singlestep_active(env)
// dc->pstate_ss = env->pstate & PSTATE_SS

// 如果 ss_active:
//   - 每个翻译块最多翻译一条指令
//   - 禁止 TB 链接 (gen_goto_tb 不跳转)
//   - 指令末尾检查 pstate_ss → gen_step_complete_exception()
```

```c
// HELPER(exception_swstep) — tcg/debug.c:541-544
void HELPER(exception_swstep)(CPUARMState *env, uint32_t syndrome)
{
    raise_exception_debug(env, EXCP_UDEF, syndrome);
}
```

---

## 33. 调试异常综合征

不同调试事件产生不同的 ESR_ELx.EC 值：

| EC 值 | 事件 | 综合征函数 |
|-------|------|-----------|
| 0x30 | 低 EL 断点 (AArch32) | `syn_breakpoint()` |
| 0x31 | 同 EL 断点 (AArch64) | `syn_breakpoint()` |
| 0x32 | 低 EL 软件单步 (AArch32) | `syn_swstep()` |
| 0x33 | 同 EL 软件单步 (AArch64) | `syn_swstep()` |
| 0x34 | 低 EL 监视点 (AArch32) | `syn_watchpoint()` |
| 0x35 | 同 EL 监视点 (AArch64) | `syn_watchpoint()` |
| 0x38 | BKPT (AArch32) | `syn_aa32_bkpt()` |
| 0x3C | BRK (AArch64) | `syn_aa64_bkpt()` |

---

## 34. arm_debug_exception_fsr()

```c
// tcg/debug.c:437-462
static uint32_t arm_debug_exception_fsr(CPUARMState *env)
{
    ARMMMUFaultInfo fi = { .type = ARMFault_Debug };
    int target_el = arm_debug_target_el(env);

    // AArch64 或 EL2 使用长描述符格式
    // AArch32 使用短描述符格式
    return arm_fi_to_sfsc/lfsc(&fi);
}
```

为 AArch32 兼容生成 FSR (Fault Status Register) 值，AArch64 使用 ESR 综合征。

---

## 35. DCC 通信通道

```c
// debug_helper.c:204-235
// MDCCSR_EL0, OSDTRRX_EL1, OSDTRTX_EL1, DBGDTRTX_EL0, DBGDTR_EL0
// 全部实现为 RAZ/WI (Read-As-Zero, Write-Ignore)
// 带 access_tdcc 陷阱检查
```

Debug Communication Channel 未实现，但寄存器以 RAZ/WI 方式存在，带正确的陷阱控制，防止 Guest 访问时产生 SIGILL。

---

## 36. DBGAUTHSTATUS 调试认证

QEMU 不模拟外部调试认证信号。DBGEN、SPIDEN、NIDEN、SPNIDEN 都隐含为高电平（始终允许调试），注释在 `aa32_generate_debug_exceptions` 中说明。

---

## 37. MDCR_EL2 Hypervisor 调试控制

| 位 | 名称 | 功能 |
|----|------|------|
| TDE | Trap Debug Exceptions | 调试异常路由到 EL2 |
| TDA | Trap Debug Access | 调试寄存器访问陷入 EL2 |
| TDOSA | Trap Debug OS-related Access | OS 锁寄存器陷入 EL2 |
| TDRA | Trap Debug ROM Access | ROM 寄存器陷入 EL2 |
| TDCC | Trap DCC access | DCC 寄存器陷入 EL2 |
| TPM | Trap PMU | PMU 寄存器陷入 EL2 |
| HPMN | Hyp PMU Number | 限制 Guest PMU 计数器数量 |
| HLP | Hyp Long counter | Hypervisor PMU 64 位使能 |

---

## 38. MDCR_EL3 安全调试控制

| 位 | 名称 | 功能 |
|----|------|------|
| SDD | Secure Debug Disable | 禁用安全态调试 |
| SPD | Secure Privilege Debug | 安全特权调试控制 |
| TDOSA | Trap Debug OS Access | OS 锁寄存器陷入 EL3 |
| TDA | Trap Debug Access | 调试寄存器陷入 EL3 |
| TPM | Trap PMU | PMU 寄存器陷入 EL3 |

---

## 39. GDB 远程调试集成

QEMU ARM 的 GDB 调试：

- `gdbstub.c:42-115`：GDB 寄存器读写入口
- GDB 断点通过 `cpu_breakpoint_insert(BP_GDB)` 设置
- 架构断点使用 `BP_CPU` 标志
- 在 `arm_debug_excp_handler` 中：GDB 断点优先于架构断点（先检查 BP_GDB）
- GDB 硬件断点/监视点映射到 QEMU 的 `cpu_breakpoint/watchpoint` API

---

## 40. KVM 调试支持

```c
// kvm.c
// kvm_arm_handle_debug() — 处理 KVM_EXIT_DEBUG
// kvm_arch_update_guest_debug() — 设置 KVM_GUESTDBG_USE_SW_BP / USE_HW
// kvm_arch_insert_hw_breakpoint() — 插入硬件断点
// kvm_arch_remove_hw_breakpoint() — 删除硬件断点
```

KVM 模式下：
- 硬件断点/监视点数量由 KVM 能力查询获得
- 使用 `KVM_SET_GUEST_DEBUG` ioctl 配置
- `KVM_EXIT_DEBUG` 退出时映射到 EXCP_DEBUG

---

## 41. 完整断点触发流程

```
1. Guest 配置断点:
   DBGBVR0_EL1 = target_addr
   DBGBCR0_EL1 = {E=1, BAS=0xF, BT=0 (addr match)}
   MDSCR_EL1.MDE = 1

2. dbgbcr_write() → hw_breakpoint_update(cpu, 0)
   → cpu_breakpoint_insert(addr, BP_CPU)

3. CPU 执行到 target_addr:
   → QEMU 检测到 cpu_breakpoint 命中
   → arm_debug_check_breakpoint() 检查:
     - MDSCR.MDE=1 ✓
     - arm_generate_debug_exceptions() ✓
     - 非 active-pending 单步 ✓
     - bp_wp_matches() 检查 SSC/PAC/HMC/链式 ✓

4. arm_debug_excp_handler():
   → 确认是 BP_CPU（非 BP_GDB）
   → raise_exception_debug(EXCP_PREFETCH_ABORT, syn_breakpoint(0))

5. 异常进入:
   → 目标 EL = arm_debug_target_el()
   → SPSR 保存当前 PSTATE
   → ELR 保存 PC
   → ESR.EC = 0x31 (同EL断点) 或 0x30 (低EL断点)
```

---

## 42. 完整监视点触发流程

```
1. Guest 配置监视点:
   DBGWVR0_EL1 = watch_addr
   DBGWCR0_EL1 = {E=1, LSC=3 (读写), BAS=0xFF}

2. dbgwcr_write() → hw_watchpoint_update(cpu, 0)
   → cpu_watchpoint_insert(addr, len, BP_MEM_ACCESS | BP_CPU)

3. CPU 访问 watch_addr:
   → TLB 检查 → watchpoint 命中
   → arm_debug_check_watchpoint() 
   → check_watchpoints() → bp_wp_matches()

4. arm_debug_excp_handler():
   → wp_hit->flags & BP_CPU
   → wnr = (写访问?)
   → raise_exception_debug(EXCP_DATA_ABORT, syn_watchpoint(wnr))

5. 异常进入:
   → ESR.EC = 0x35 (同EL监视点) 或 0x34 (低EL)
   → FAR = 命中地址
```

---

## 43. 调试异常优先级

在同一条指令上，调试事件的优先级：

1. **软件单步 (active-pending)** — 最高优先级
2. **PC 对齐错误** — 高于断点
3. **指令中止** — 高于断点
4. **硬件断点** — 标准优先级
5. **BRK/BKPT** — 指令执行时触发

---

## 44. 关键数据结构汇总

| 结构/字段 | 位置 | 用途 |
|----------|------|------|
| `dbgbvr[16]` | cpu.h:529 | 断点值寄存器 |
| `dbgbcr[16]` | cpu.h:530 | 断点控制寄存器 |
| `dbgwvr[16]` | cpu.h:531 | 监视点值寄存器 |
| `dbgwcr[16]` | cpu.h:532 | 监视点控制寄存器 |
| `mdscr_el1` | cpu.h:534 | 调试系统控制 (SS/KDE/MDE) |
| `oslsr_el1` | cpu.h:535 | OS 锁状态 |
| `osdlr_el1` | cpu.h:536 | OS 双重锁状态 |
| `mdcr_el2` | cpu.h:537 | Hypervisor 调试控制 |
| `mdcr_el3` | cpu.h:538 | 安全调试控制 |
| `cpu_breakpoint[16]` | CPUState | QEMU 断点对象指针 |
| `cpu_watchpoint[16]` | CPUState | QEMU 监视点对象指针 |

---

## 45. 总结

QEMU ARM 调试架构实现的关键特点：

1. **完整的自托管调试**：硬件断点/监视点 + 软件单步 + BRK/BKPT
2. **双层映射**：ARM 架构寄存器 → QEMU cpu_breakpoint/watchpoint API
3. **三层访问控制**：TDA/TDOSA/TDRA + MDCR_EL2/EL3 陷阱
4. **OS 锁**：OSLAR/OSLSR/OSDLR 电源管理保护
5. **调试路由**：TDE/TGE 控制调试异常目标 EL
6. **安全调试**：SDD/SPD 控制安全态调试行为
7. **GDB/KVM 兼容**：BP_GDB vs BP_CPU 区分，KVM 直通调试
8. **部分实现**：上下文匹配、VMID 匹配、地址不匹配断点标记为 UNIMP
9. **DCC 存根**：通信通道 RAZ/WI 但带正确陷阱
10. **认证开放**：DBGEN/SPIDEN 始终允许，简化调试体验
