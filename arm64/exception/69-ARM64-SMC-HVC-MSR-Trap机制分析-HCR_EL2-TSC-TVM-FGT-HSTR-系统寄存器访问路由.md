# ARM64 SMC/HVC/MSR Trap 到 EL2/EL3 机制分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/69 |
| 分析对象 | SMC/HVC/MSR(系统寄存器) 的 EL Trap 路由 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §D1.9 (Exception levels), §C5.1 (System registers) |
| 关键控制位 | HCR_EL2.TSC/HCD/TVM/TRVM, SCR_EL3.SMD/HCE, HSTR_EL2, FGT |
| 核心结论 | **SMC/HVC/MSR trap 实现完整且符合规范** |

---

## 1. 概述

ARM64 架构提供三类主要的 EL trap 机制：

| 机制 | 指令 | Trap 控制 | 目标 EL |
|------|------|-----------|---------|
| **SMC Trap** | SMC #imm | HCR_EL2.TSC | EL1 → EL2 |
| **HVC 控制** | HVC #imm | HCR_EL2.HCD, SCR_EL3.HCE | UNDEF/EL2/EL3 |
| **MSR/MRS Trap** | MSR/MRS Xx | HCR_EL2.TVM/TRVM, HSTR_EL2, FGT | EL1 → EL2 |

与 WFE 不同（doc 68 已证明 WFE trap 未实现），这三类 trap 在 QEMU 中**实现完整**。

---

## 2. SMC Trap 机制

### 2.1 ARM 规范定义

SMC (Secure Monitor Call) 指令从 NS-EL1 发起时：

```
优先级判断：
1. EL0 执行 → UNDEF（翻译时已拦截）
2. !ARM_FEATURE_EL3 && !HCR_EL2.NV → UNDEF
3. HCR_EL2.TSC == 1 && NS EL1 → Trap 到 EL2 (EC=0x17)
4. SCR_EL3.SMD == 1 → UNDEF
5. 正常路径 → Trap 到 EL3 (EXCP_SMC)
```

### 2.2 QEMU 翻译阶段

```c
// target/arm/tcg/translate-a64.c:3193
static bool trans_SMC(DisasContext *s, arg_i *a)
{
    if (s->current_el == 0) {
        unallocated_encoding(s);    // EL0 → UNDEF（编译时）
        return true;
    }
    gen_a64_update_pc(s, 0);        // PC 同步
    gen_helper_pre_smc(tcg_env, tcg_constant_i32(syn_aa64_smc(a->imm)));
    gen_ss_advance(s);              // 单步优先
    gen_exception_insn_el(s, 4, EXCP_SMC, syn_aa64_smc(a->imm), 3);
    return true;
}
```

关键设计：
- **EL0 拦截在翻译时**（`unallocated_encoding`），不需要 runtime 检查
- `gen_helper_pre_smc` 是运行时 trap 检查（可能 raise 到 EL2 或 UNDEF）
- 如果 `pre_smc` 没有异常，继续执行 `gen_exception_insn_el(..., 3)` trap 到 EL3

### 2.3 Runtime pre_smc 检查

```c
// target/arm/tcg/op_helper.c:1111
void HELPER(pre_smc)(CPUARMState *env, uint32_t syndrome)
{
    ARMCPU *cpu = env_archcpu(env);
    int cur_el = arm_current_el(env);
    bool secure = arm_is_secure(env);
    bool smd_flag = env->cp15.scr_el3 & SCR_SMD;
    bool smd = arm_feature(env, ARM_FEATURE_AARCH64) ? smd_flag
                                                     : smd_flag && !secure;

    // 情况1: 无EL3 且无NV → UNDEF
    if (!arm_feature(env, ARM_FEATURE_EL3) &&
        !(arm_hcr_el2_eff(env) & HCR_NV) &&
        cpu->psci_conduit != QEMU_PSCI_CONDUIT_SMC) {
        raise_exception(env, EXCP_UDEF, syn_uncategorized(),
                        exception_target_el(env));
    }

    // 情况2: HCR_EL2.TSC == 1 && NS EL1 → Trap 到 EL2
    if (cur_el == 1 && (arm_hcr_el2_eff(env) & HCR_TSC)) {
        raise_exception(env, EXCP_HYP_TRAP, syndrome, 2);
    }

    // 情况3: SMD || 无EL3 → UNDEF
    if (!arm_is_psci_call(cpu, EXCP_SMC) &&
        (smd || !arm_feature(env, ARM_FEATURE_EL3))) {
        raise_exception(env, EXCP_UDEF, syn_uncategorized(),
                        exception_target_el(env));
    }

    // 情况4: 正常 SMC → 继续执行到 gen_exception_insn_el(target_el=3)
}
```

### 2.4 SMC Trap 流程图

```
EL1 执行 SMC #imm
    │
    ▼
trans_SMC():
  gen_helper_pre_smc(syndrome)
  gen_exception_insn_el(EXCP_SMC, syn, target_el=3)
    │
    ▼
HELPER(pre_smc)(env, syndrome):
    │
    ├─ !EL3 && !NV ? → UNDEF [raise_exception]
    │
    ├─ HCR_EL2.TSC && cur_el==1 ?
    │   └─ YES → raise_exception(EXCP_HYP_TRAP, syn, target_el=2)
    │            → arm_cpu_do_interrupt()
    │            → ESR_EL2.EC = 0x17 (SMC trap)
    │            → ELR_EL2 = SMC 地址
    │            → VBAR_EL2 + 0x400
    │
    ├─ SMD || !EL3 ? → UNDEF [raise_exception]
    │
    └─ 正常返回 → 继续执行 gen_exception_insn_el
                   → raise_exception(EXCP_SMC, syn, target_el=3)
                   → arm_cpu_do_interrupt()
                   → 进入 EL3 (Secure Monitor)
```

### 2.5 PSCI 特殊处理

QEMU 实现了 PSCI (Power State Coordination Interface)，使用 SMC 作为调用通道：

```c
if (arm_is_psci_call(cpu, EXCP_SMC)) {
    // 跳过 SMD 检查，让 SMC 到达 EL3 handler
    // QEMU 在 arm_cpu_do_interrupt 中处理 PSCI 调用
}
```

---

## 3. HVC 控制机制

### 3.1 ARM 规范定义

HVC (Hypervisor Call) 的控制逻辑：

| 条件 | 结果 |
|------|------|
| EL0 执行 | UNDEF（编码层面非法） |
| !ARM_FEATURE_EL2 | UNDEF |
| SCR_EL3.HCE == 0 (有EL3时) | UNDEF |
| HCR_EL2.HCD == 1 (无EL3时) | UNDEF |
| Secure && AArch32 | UNDEF |
| 正常 | 同步异常到 EL2 (EC=0x16) |

### 3.2 QEMU 翻译阶段

```c
// target/arm/tcg/translate-a64.c:3173
static bool trans_HVC(DisasContext *s, arg_i *a)
{
    int target_el = s->current_el == 3 ? 3 : 2;

    if (s->current_el == 0) {
        unallocated_encoding(s);    // EL0 → 编译时 UNDEF
        return true;
    }

    gen_a64_update_pc(s, 0);
    gen_helper_pre_hvc(tcg_env);    // 运行时 UNDEF 检查
    gen_ss_advance(s);
    gen_exception_insn_el(s, 4, EXCP_HVC, syn_aa64_hvc(a->imm), target_el);
    return true;
}
```

注意：`target_el` 的选择：
- `current_el == 3`（从 EL3 执行 HVC）→ 异常到 EL3 自己
- 否则 → 异常到 EL2

### 3.3 Runtime pre_hvc 检查

```c
// target/arm/tcg/op_helper.c:1071
void HELPER(pre_hvc)(CPUARMState *env)
{
    ARMCPU *cpu = env_archcpu(env);
    int cur_el = arm_current_el(env);
    bool secure = false;  // FIXME: Use actual secure state
    bool undef;

    // PSCI 通过 HVC 调用时跳过检查
    if (arm_is_psci_call(cpu, EXCP_HVC)) {
        return;
    }

    if (!arm_feature(env, ARM_FEATURE_EL2)) {
        undef = true;                           // 无 EL2 → UNDEF
    } else if (arm_feature(env, ARM_FEATURE_EL3)) {
        undef = !(env->cp15.scr_el3 & SCR_HCE); // EL3 禁止 HVC
    } else {
        undef = env->cp15.hcr_el2 & HCR_HCD;    // EL2 禁止 HVC
    }

    // Secure + AArch32 或 Secure EL1 → UNDEF
    if (secure && (!is_a64(env) || cur_el == 1)) {
        undef = true;
    }

    if (undef) {
        raise_exception(env, EXCP_UDEF, syn_uncategorized(),
                        exception_target_el(env));
    }
}
```

### 3.4 HVC 与 SMC 的关键区别

| 方面 | HVC | SMC |
|------|-----|-----|
| 目标 EL | EL2 (或 EL3 如从 EL3 执行) | EL3 |
| Trap 到中间 EL | 不适用（直接到 EL2） | HCR_EL2.TSC 可 trap 到 EL2 |
| 禁用控制 | SCR_EL3.HCE / HCR_EL2.HCD | SCR_EL3.SMD |
| EC syndrome | 0x16 | 0x17 |
| PSCI 支持 | ✅ | ✅ |

---

## 4. MSR/MRS 系统寄存器 Trap

### 4.1 ARM 规范定义

系统寄存器访问可以被多种机制 trap：

```
优先级（从高到低）：
1. UNDEFINED（权限不足）
2. accessfn 返回 TRAP_EL1（EL0 → EL1）
3. HSTR_EL2 trap（AArch32 CP15 寄存器）
4. Fine-Grained Traps (FGT)
5. accessfn 返回 TRAP_EL2/TRAP_EL3
6. HCR_EL2.TVM/TRVM trap
```

### 4.2 QEMU 实现架构

QEMU 使用 `ARMCPRegInfo` 结构体描述每个系统寄存器，其中 `accessfn` 字段指向访问检查函数：

```c
// target/arm/cpregs.h (概念)
typedef struct ARMCPRegInfo {
    const char *name;
    CPAccessRights access;      // 静态权限（PL0/PL1/PL2/PL3）
    CPAccessFn *accessfn;       // 动态访问检查（trap 逻辑）
    uint32_t fgt;               // Fine-Grained Trap 描述
    // ...
} ARMCPRegInfo;
```

### 4.3 access_check_cp_reg() — 核心检查函数

```c
// target/arm/tcg/op_helper.c:802
const void *HELPER(access_check_cp_reg)(CPUARMState *env, uint32_t key,
                                        uint32_t syndrome, uint32_t isread)
{
    ARMCPU *cpu = env_archcpu(env);
    const ARMCPRegInfo *ri = get_arm_cp_reginfo(cpu->cp_regs, key);
    CPAccessResult res = CP_ACCESS_OK;

    // ① 调用寄存器的 accessfn
    if (ri->accessfn) {
        res = ri->accessfn(env, ri, isread);
    }

    // ② EL0→EL1 的 trap 优先级最高
    if (res != CP_ACCESS_OK && ((res & CP_ACCESS_EL_MASK) < 2) &&
        arm_current_el(env) == 0) {
        goto fail;
    }

    // ③ HSTR_EL2 检查（仅 AArch32 CP15，EL0 访问）
    if (!is_a64(env) && arm_current_el(env) == 0 && ri->cp == 15 &&
        arm_is_el2_enabled(env)) {
        uint32_t mask = 1 << ri->crn;
        if (env->cp15.hstr_el2 & mask) {
            res = CP_ACCESS_TRAP_EL2;
            goto fail;
        }
    }

    // ④ Fine-Grained Traps (FEAT_FGT)
    if (arm_fgt_active(env, arm_current_el(env))) {
        uint64_t trapword = 0;
        unsigned int idx = FIELD_EX32(ri->fgt, FGT, IDX);
        unsigned int bitpos = FIELD_EX32(ri->fgt, FGT, BITPOS);
        bool rev = FIELD_EX32(ri->fgt, FGT, REV);

        if (ri->fgt & FGT_EXEC) {
            trapword = env->cp15.fgt_exec[idx];
        } else if (isread && (ri->fgt & FGT_R)) {
            trapword = env->cp15.fgt_read[idx];
        } else if (!isread && (ri->fgt & FGT_W)) {
            trapword = env->cp15.fgt_write[idx];
        }

        bool trapbit = extract64(trapword, bitpos, 1);
        if (trapbit != rev) {
            res = CP_ACCESS_TRAP_EL2;
            goto fail;
        }
    }

    if (likely(res == CP_ACCESS_OK)) {
        return ri;  // 访问允许
    }

fail:
    // 根据 res 确定目标 EL 和异常类型
    target_el = res & CP_ACCESS_EL_MASK;
    raise_exception(env, excp, syndrome, target_el);
}
```

### 4.4 HCR_EL2.TVM/TRVM Trap 示例

`access_tvm_trvm` 是典型的寄存器级 trap 函数：

```c
// target/arm/helper.c:333
CPAccessResult access_tvm_trvm(CPUARMState *env, const ARMCPRegInfo *ri,
                               bool isread)
{
    if (arm_current_el(env) == 1) {
        uint64_t trap = isread ? HCR_TRVM : HCR_TVM;
        if (arm_hcr_el2_eff(env) & trap) {
            return CP_ACCESS_TRAP_EL2;
        }
    }
    return CP_ACCESS_OK;
}
```

影响的寄存器包括：SCTLR_EL1、TTBR0_EL1、TTBR1_EL1、TCR_EL1、ESR_EL1、FAR_EL1、
AFSR0_EL1、AFSR1_EL1、MAIR_EL1、AMAIR_EL1、CONTEXTIDR_EL1 等。

### 4.5 MSR Trap 完整流程图

```
EL1 执行 MSR SCTLR_EL1, X0
    │
    ▼
翻译阶段 (translate-a64.c:2883):
  ri->accessfn 存在? → gen_helper_access_check_cp_reg()
    │
    ▼
HELPER(access_check_cp_reg)(env, key, syndrome, isread=0):
    │
    ├─ ① ri->accessfn(env, ri, isread)
    │     = access_tvm_trvm(env, ri, false)
    │     └─ cur_el==1 && HCR_EL2.TVM==1 ?
    │         └─ YES → return CP_ACCESS_TRAP_EL2
    │
    ├─ ② HSTR_EL2 检查 (AArch32 only)
    │
    ├─ ③ FGT 检查 (fgt_write[idx] & bitpos)
    │
    └─ fail:
        target_el = 2
        raise_exception(env, EXCP_UDEF, syndrome, 2)
            │
            ▼
        arm_cpu_do_interrupt():
        → ESR_EL2.EC = 0x18 (MSR/MRS trap)
        → ESR_EL2.ISS = {Op0, Op1, CRn, CRm, Op2, Rt, direction}
        → ELR_EL2 = MSR 指令地址
        → VBAR_EL2 + 0x400
```

### 4.6 Fine-Grained Traps (FEAT_FGT)

FEAT_FGT 允许 EL2 对单个系统寄存器设置 trap，比 TVM 更细粒度：

```c
// 寄存器定义中的 FGT 字段
{ .name = "TCR_EL1", ...,
  .accessfn = access_tvm_trvm,
  .fgt = FGT_TCR_EL1,        // ← Fine-Grained Trap 标识
}
```

FGT 寄存器组：
- `HFGRTR_EL2`：细粒度读 trap
- `HFGWTR_EL2`：细粒度写 trap
- `HDFGRTR_EL2`：调试/PMU 读 trap
- `HDFGWTR_EL2`：调试/PMU 写 trap
- `HFGITR_EL2`：指令执行 trap

QEMU 完整实现了 FGT，包括 `HCRX_EL2.FGTnXS` 对 TLBI nXS 变体的豁免。

---

## 5. 三类 Trap 的调用链对比

### 5.1 统一视角

```
                    ┌─────────────────────────────────────┐
                    │          翻译阶段                     │
                    │  (translate-a64.c)                    │
                    └─────────┬───────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
    │ trans_SMC  │      │ trans_HVC  │      │ handle_sys│
    │  ↓         │      │  ↓         │      │  ↓        │
    │pre_smc()   │      │pre_hvc()   │      │access_    │
    │gen_excep   │      │gen_excep   │      │check_cp   │
    │(EL3)       │      │(EL2)       │      │_reg()     │
    └─────┬──────┘      └─────┬──────┘      └─────┬─────┘
          │                   │                   │
    ┌─────▼──────────────────▼──────────────────▼─────────┐
    │              Runtime Helper                           │
    └─────┬──────────────────┬──────────────────┬─────────┘
          │                   │                   │
    ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
    │TSC→EL2    │      │HCD→UNDEF  │      │TVM→EL2    │
    │SMD→UNDEF  │      │!HCE→UNDEF │      │FGT→EL2    │
    │else→EL3   │      │else→EL2   │      │HSTR→EL2   │
    └─────┬─────┘      └─────┬─────┘      └─────┬─────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                    ┌─────────▼───────────────────────────┐
                    │      raise_exception()                │
                    │  → cs->exception_index = excp         │
                    │  → env->exception.syndrome            │
                    │  → env->exception.target_el           │
                    │  → cpu_loop_exit()                    │
                    └─────────┬───────────────────────────┘
                              │
                    ┌─────────▼───────────────────────────┐
                    │   arm_cpu_do_interrupt()              │
                    │   → SPSR_ELx = PSTATE                │
                    │   → ELR_ELx = fault PC               │
                    │   → ESR_ELx = syndrome               │
                    │   → PC = VBAR_ELx + offset           │
                    └─────────────────────────────────────┘
```

### 5.2 Syndrome EC 编码

| EC 值 | 异常类型 | 适用指令 |
|--------|----------|----------|
| 0x01 | WFx Trap | WFI/WFE/WFIT/WFET |
| 0x16 | HVC | HVC #imm (AArch64) |
| 0x17 | SMC | SMC #imm (AArch64) |
| 0x18 | MSR/MRS/System | 系统寄存器访问 |
| 0x00 | Uncategorized | UNDEF fallback |

### 5.3 PC 处理差异

| 指令 | ELR_ELx 值 | 原因 |
|------|-------------|------|
| WFI (trap) | WFI 地址（PC-4） | Guest 应重试 WFI |
| SMC (trap to EL2) | SMC 地址 | TSC trap 时 Guest 需知道原指令 |
| SMC (to EL3) | SMC+4 | EL3 不需要重执行 SMC |
| HVC | HVC+4 | EL2 handler 处理后返回下一条 |
| MSR (trap) | MSR 地址 | EL2 需要模拟这条 MSR |

---

## 6. CPAccessResult 类型系统

### 6.1 返回值编码

```c
typedef enum CPAccessResult {
    CP_ACCESS_OK = 0,           // 允许访问

    // Trap 到指定 EL（带正确 syndrome）
    CP_ACCESS_TRAP_EL1 = 5,     // (1<<2) | 1
    CP_ACCESS_TRAP_EL2 = 6,     // (1<<2) | 2
    CP_ACCESS_TRAP_EL3 = 7,     // (1<<2) | 3

    // UNDEF（EC=0x00，到默认目标 EL）
    CP_ACCESS_UNDEFINED = 8,    // (2<<2)

    // GCS EXLOCK 异常
    CP_ACCESS_EXLOCK = 12,      // (3<<2)
} CPAccessResult;
```

### 6.2 低 2 位提取目标 EL

```c
target_el = res & CP_ACCESS_EL_MASK;  // 0x3
```

这使得 `CP_ACCESS_TRAP_EL2` 的低 2 位 = 2，直接得到 target EL。

### 6.3 常见 accessfn 返回值

| accessfn | 条件 | 返回值 |
|----------|------|--------|
| `access_tvm_trvm` | EL1 && HCR.TVM/TRVM | `CP_ACCESS_TRAP_EL2` |
| `access_tsw` | EL1 && HCR.TSW | `CP_ACCESS_TRAP_EL2` |
| `access_tacr` | EL1 && HCR.TACR | `CP_ACCESS_TRAP_EL2` |
| `access_el3_aa32ns` | NS && AArch32 EL3 | `CP_ACCESS_TRAP_UNCATEGORIZED` |
| FGT 匹配 | bit set | `CP_ACCESS_TRAP_EL2` |

---

## 7. HSTR_EL2 寄存器 Trap

### 7.1 功能说明

`HSTR_EL2` (Hypervisor System Trap Register) 允许按 CRn 编号批量 trap AArch32 CP15 寄存器：

```
HSTR_EL2[n] = 1 → CP15 CRn=n 的所有寄存器访问 trap 到 EL2
```

### 7.2 QEMU 实现位置

**EL1 访问的 HSTR 检查在翻译时**（translate.c:1771）：
```c
// 翻译时对 EL1 检查 HSTR（性能优化）
if (s->current_el == 1 && arm_is_el2_enabled(env)) {
    // 检查 HSTR_EL2 对应位
}
```

**EL0 访问的 HSTR 检查在运行时**（op_helper.c:835）：
```c
// 运行时对 EL0 检查 HSTR
if (!is_a64(env) && arm_current_el(env) == 0 && ri->cp == 15) {
    uint32_t mask = 1 << ri->crn;
    if (env->cp15.hstr_el2 & mask) {
        res = CP_ACCESS_TRAP_EL2;
    }
}
```

### 7.3 两阶段检查的原因

- EL1 的 HSTR 检查放在翻译时：HSTR 值在 TB 有效期内不变（TB invalidation 机制保证）
- EL0 的 HSTR 检查放在运行时：因为 EL0 的其他 trap 检查（如 accessfn）也在运行时

---

## 8. 与 WFE/WFI Trap 的实现质量对比

### 8.1 实现完整度评分

| 机制 | 翻译时检查 | 运行时检查 | Trap 路由 | Syndrome | 总评 |
|------|:---:|:---:|:---:|:---:|:---:|
| **WFI trap** | ✅ | ✅ check_wfx_trap | ✅ | ✅ EC=0x01 | ⭐⭐⭐⭐⭐ |
| **WFE trap** | ⚠️ MTTCG=NOP | ❌ 不检查 | ❌ | — | ⭐☆☆☆☆ |
| **SMC trap** | ✅ EL0 UNDEF | ✅ pre_smc | ✅ | ✅ EC=0x17 | ⭐⭐⭐⭐⭐ |
| **HVC** | ✅ EL0 UNDEF | ✅ pre_hvc | ✅ | ✅ EC=0x16 | ⭐⭐⭐⭐⭐ |
| **MSR/MRS trap** | ✅ 部分 HSTR | ✅ access_check | ✅ | ✅ EC=0x18 | ⭐⭐⭐⭐⭐ |
| **FGT** | — | ✅ fgt_read/write/exec | ✅ | ✅ | ⭐⭐⭐⭐⭐ |

### 8.2 为什么 WFE 例外

推测原因：
1. **历史原因**：早期 QEMU 不模拟 EL2，WFE 只需简单 yield
2. **实用性**：纯 TCG 下很少测试嵌套虚拟化的 WFE trap 行为
3. **复杂度**：SMC/HVC/MSR 的 trap 是**功能必需**（否则虚拟化完全无法工作），而 WFE trap 是**优化层面**

---

## 9. Nested Virtualization (FEAT_NV/NV2)

### 9.1 对 SMC 的影响

FEAT_NV 引入 `HCR_EL2.NV` 位，影响 SMC 行为：

```c
// op_helper.c:1165
if (!arm_feature(env, ARM_FEATURE_EL3) &&
    !(arm_hcr_el2_eff(env) & HCR_NV) &&
    cpu->psci_conduit != QEMU_PSCI_CONDUIT_SMC) {
    // 无 EL3 时 SMC UNDEF，但 NV=1 时允许 trap 到 EL2
    raise_exception(env, EXCP_UDEF, ...);
}
```

### 9.2 对 MSR/MRS 的影响

FEAT_NV2 允许 EL1 访问 EL2 寄存器时重定向而非 trap：

```c
// translate-a64.c:2916
if (nv_redirect_reg) {
    // FEAT_NV2: 重定向 EL2 寄存器到 EL1 编码
    key = ENCODE_AA64_CP_REG(op0, 0, crn, crm, op2);
    ri = redirect_cpreg(s, key, isread);
}
```

---

## 10. 总结

### 10.1 QEMU EL Trap 实现状态总览

```
┌──────────────────────────────────────────────────────────┐
│                    EL Trap 机制完整度                      │
├─────────────┬─────────┬─────────┬────────────────────────┤
│  机制        │ 状态     │ EC      │ 说明                   │
├─────────────┼─────────┼─────────┼────────────────────────┤
│ SMC→EL2     │ ✅ 完整  │ 0x17    │ HCR_EL2.TSC + NV      │
│ SMC→EL3     │ ✅ 完整  │ —       │ 正常 SMC 路径          │
│ HVC→EL2     │ ✅ 完整  │ 0x16    │ HCD/HCE 控制          │
│ MSR→EL2     │ ✅ 完整  │ 0x18    │ TVM/TRVM/FGT/HSTR    │
│ WFI→EL2     │ ✅ 完整  │ 0x01    │ HCR_EL2.TWI           │
│ WFE→EL2     │ ❌ 缺失  │ —       │ HCR_EL2.TWE 未检查    │
│ WFET→EL2    │ ❌ 缺失  │ —       │ 同 WFE                │
│ SVE/SME→EL2 │ ✅ 完整  │ 0x19    │ CPTR_EL2 控制         │
│ FP→EL2      │ ✅ 完整  │ 0x07    │ CPTR_EL2/HCR.TGE     │
└─────────────┴─────────┴─────────┴────────────────────────┘
```

### 10.2 设计模式总结

QEMU 的 trap 实现遵循一致的两阶段模式：

1. **翻译时**：处理静态可确定的非法（EL0 执行 SMC/HVC）和可缓存的 trap 条件（HSTR_EL2 for EL1）
2. **运行时 Helper**：处理需要读取动态寄存器的 trap 条件（HCR_EL2、SCR_EL3、FGT）

这种设计在 **正确性** 和 **性能** 之间取得良好平衡：
- 翻译时检查避免了不必要的 helper 调用
- TB invalidation 机制确保翻译时假设失效时会重新翻译

### 10.3 与 doc 68 (WFE) 的关联

WFE 是目前唯一已知的 **应该 trap 但未实现** 的 A-profile 指令。
所有其他需要 trap 到 EL2/EL3 的指令（SMC、HVC、MSR/MRS、SVE、FP）都正确实现。
