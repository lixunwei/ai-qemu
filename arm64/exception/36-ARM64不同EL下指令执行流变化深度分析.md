# ARM64 不同异常级别下的指令执行流变化深度分析

> 基于 QEMU 11.0.50 源码分析，聚焦不同 EL（EL0-EL3）下 TB 翻译标志差异、
> DisasContext EL 感知翻译、系统寄存器访问权限与陷阱路由、MMU 配置体制切换、
> 内存访问权限（PAN/UAO）、FP/SVE/SME 陷阱控制、调试路由与 VHE 寄存器重定向。

---

## 目录

1. [hflags 与 TB 翻译标志](#1-hflags-与-tb-翻译标志)
2. [DisasContext 的 EL 感知翻译](#2-disascontext-的-el-感知翻译)
3. [系统寄存器访问权限控制](#3-系统寄存器访问权限控制)
4. [EL2/EL3 陷阱路由机制](#4-el2el3-陷阱路由机制)
5. [MMU 配置体制（Regime）](#5-mmu-配置体制regime)
6. [内存访问权限与 PAN/UAO](#6-内存访问权限与-panuao)
7. [FP/SIMD/SVE/SME 陷阱控制](#7-fpsimdsvesme-陷阱控制)
8. [中断屏蔽与 DAIF/ALLINT](#8-中断屏蔽与-daifallint)
9. [调试寄存器路由](#9-调试寄存器路由)
10. [VHE 寄存器重定向](#10-vhe-寄存器重定向)
11. [各 EL 差异对比表](#11-各-el-差异对比表)
12. [总结](#12-总结)

---

## 1. hflags 与 TB 翻译标志

QEMU 通过 **hflags** 将 EL 相关的运行时状态缓存到 TB 标志中，避免翻译时反复读取 CPU 状态。

### hflags 重建入口

```c
// target/arm/tcg/hflags.c:506-524
static CPUARMTBFlags rebuild_hflags_internal(CPUARMState *env) {
    int el = arm_current_el(env);               // 当前 EL
    int fp_el = fp_exception_el(env, el);       // FP 陷阱目标 EL（0=不陷阱）
    ARMMMUIdx mmu_idx = arm_mmu_idx_el(env, el); // EL→MMU 体制映射

    if (is_a64(env))
        return rebuild_hflags_a64(env, el, fp_el, mmu_idx);
    // ...
}

void arm_rebuild_hflags(CPUARMState *env) {
    env->hflags = rebuild_hflags_internal(env);
}
```

### rebuild_hflags_a64 — EL 敏感的标志计算

```c
// target/arm/tcg/hflags.c:240-503
static CPUARMTBFlags rebuild_hflags_a64(CPUARMState *env, int el, int fp_el,
                                        ARMMMUIdx mmu_idx) {
    // TBI (Top Byte Ignore) — 受 TCR_ELx 控制
    tbid = aa64_va_parameter_tbi(tcr, mmu_idx);

    // VHE (E2H) — 影响 EL2 行为
    if (hcr & HCR_E2H) DP_TBFLAG_A64(flags, E2H, 1);          // :260-262

    // SVE 陷阱级别 — 受 CPACR/CPTR 和当前 EL 控制
    sve_el = sve_exception_el(env, el);                          // :265
    DP_TBFLAG_A64(flags, SVEEXC_EL, sve_el);                    // :280
    if (sve_el == 0) DP_TBFLAG_A64(flags, VL, sve_vqm1_for_el(env, el));

    // SME 陷阱级别 + 流式模式
    sme_el = sme_exception_el(env, el);                          // :283
    DP_TBFLAG_A64(flags, SMEEXC_EL, sme_el);                   // :286

    // UNPRIV (LDTR/STTR 非特权访问) — 仅 EL1/EL2 有效
    if (!(env->pstate & PSTATE_UAO)) {                           // :346
        switch (mmu_idx) {
        case ARMMMUIdx_E10_1: case ARMMMUIdx_E10_1_PAN:
            DP_TBFLAG_A64(flags, UNPRIV, 1);                     // :352
            break;
        case ARMMMUIdx_E20_2: case ARMMMUIdx_E20_2_PAN:
            if (hcr_el2 & HCR_TGE) DP_TBFLAG_A64(flags, UNPRIV, 1);
            break;
        }
    }

    // FGT (Fine-Grained Traps) — EL2 控制 EL1 操作
    if (arm_fgt_active(env, el)) {                               // :374
        DP_TBFLAG_ANY(flags, FGT_ACTIVE, 1);
        // ERET 陷阱
        if (HFGITR_EL2.ERET) DP_TBFLAG_A64(flags, TRAP_ERET, 1);
    }

    // NV (嵌套虚拟化) — 仅 EL1 且 HCR.NV=1
    if (el == 1 && (hcr & HCR_NV)) {                            // :388
        DP_TBFLAG_A64(flags, NV, 1);
        if (hcr & HCR_NV2) DP_TBFLAG_A64(flags, NV2, 1);
    }

    // MTE (Memory Tagging) — EL 相关的 tag check 使能
    // ...根据 el、SCTLR.ATA/ATA0、TCR.TCMA 等计算                // :402-447
}
```

**关键设计**：TB 标志在 EL 切换后必须重建（`arm_rebuild_hflags`），否则翻译出的代码会使用旧的权限/MMU 设置。

---

## 2. DisasContext 的 EL 感知翻译

### DisasContext 结构

```c
// target/arm/tcg/translate.h:39-110
typedef struct DisasContext {
    int current_el;          // 当前异常级别
    int user;                // current_el == 0
    ARMMMUIdx mmu_idx;       // MMU 体制索引
    int fp_excp_el;          // FP 陷阱目标 EL（0=允许）
    int sve_excp_el;         // SVE 陷阱目标 EL
    int sme_excp_el;         // SME 陷阱目标 EL
    int vl;                  // 当前向量长度（字节）
    int svl;                 // 当前流式向量长度
    bool aarch64;            // AArch64 模式
    // ...
} DisasContext;
```

### EL 影响翻译的示例

**特权指令检查**：
```c
// ERET — 仅 EL1+ 可执行
if (s->current_el == 0) return false;  // translate-a64.c:1958

// HVC — EL0 不可执行
if (s->current_el == 0) { unallocated_encoding(s); return true; }

// SMC — EL0 不可执行
if (s->current_el == 0) { unallocated_encoding(s); return true; }
```

**FP/SVE 访问检查**：
```c
// 翻译时检查 fp_excp_el
if (s->fp_excp_el != 0) {
    // 生成陷阱到 fp_excp_el 的代码
    gen_exception_insn_el(s, EXCP_UDEF, syn_fp_access_trap(), s->fp_excp_el);
}
```

---

## 3. 系统寄存器访问权限控制

### ARMCPRegInfo 权限模型

```c
// target/arm/cpregs.h:29-152
enum {
    ARM_CP_CONST          = 1 << 4,   // 只读常量
    ARM_CP_SUPPRESS_TB_END = 1 << 6,  // 写入后不结束 TB
    ARM_CP_RAISES_EXC     = 1 << 11,  // 访问函数可能引发异常
    ARM_CP_FPU            = 1 << 16,  // 需要 FP 使能
    ARM_CP_SVE            = 1 << 17,  // 需要 SVE 使能
    ARM_CP_SME            = 1 << 22,  // 需要 SME 使能
    ARM_CP_NV2_REDIRECT   = 1 << 20,  // NV2 重定向到 VNCR 页
};

// 权限级别通过 .access 字段编码：
// PL0_R, PL0_RW — EL0 可访问
// PL1_R, PL1_RW — EL1+ 可访问
// PL2_R, PL2_RW — EL2+ 可访问
// PL3_R, PL3_RW — 仅 EL3 可访问
```

### accessfn — 运行时权限检查

每个系统寄存器可以定义 `.accessfn`，在运行时根据当前 EL 和配置动态决定访问权限：

```c
// target/arm/helper.c:332-363
// TVM/TRVM 陷阱 — EL1 访问虚拟内存寄存器被 trap 到 EL2
CPAccessResult access_tvm_trvm(CPUARMState *env, const ARMCPRegInfo *ri,
                               bool isread) {
    if (arm_current_el(env) == 1) {
        uint64_t trap = isread ? HCR_TRVM : HCR_TVM;
        if (arm_hcr_el2_eff(env) & trap)
            return CP_ACCESS_TRAP_EL2;
    }
    return CP_ACCESS_OK;
}

// TSW 陷阱 — EL1 的 cache 维护操作被 trap 到 EL2
static CPAccessResult access_tsw(...) {
    if (arm_current_el(env) == 1 && (hcr & HCR_TSW))
        return CP_ACCESS_TRAP_EL2;
}

// TACR 陷阱 — EL1 的 ACTLR 访问被 trap 到 EL2
static CPAccessResult access_tacr(...) {
    if (arm_current_el(env) == 1 && (hcr & HCR_TACR))
        return CP_ACCESS_TRAP_EL2;
}
```

### CPAccessResult 返回值

| 返回值 | 含义 |
|--------|------|
| `CP_ACCESS_OK` | 允许访问 |
| `CP_ACCESS_TRAP_EL2` | 陷阱到 EL2 |
| `CP_ACCESS_TRAP_EL3` | 陷阱到 EL3 |
| `CP_ACCESS_UNDEFINED` | UNDEF 异常 |

---

## 4. EL2/EL3 陷阱路由机制

### EL2 通过 HCR_EL2 控制 EL1 行为

| HCR_EL2 位 | 作用 | 影响的 EL1 操作 |
|------------|------|----------------|
| TVM | 陷阱虚拟内存写 | SCTLR/TTBR/TCR/MAIR/AMAIR 等写入 |
| TRVM | 陷阱虚拟内存读 | 同上读取 |
| TSW | 陷阱 cache set/way | DC ISW/CSW/CISW |
| TACR | 陷阱 ACTLR | ACTLR_EL1 访问 |
| TIDCP | 陷阱 impl-defined | 实现定义的协处理器寄存器 |
| TGE | 通用异常陷阱 | EL0 所有异常直接到 EL2 |
| IMO/FMO/AMO | 中断路由 | IRQ/FIQ/SError 路由到 EL2 |

### EL3 通过 SCR_EL3 控制低 EL 行为

```c
// target/arm/helper.c:315-329
// Secure EL1 访问被陷阱到 EL3
static CPAccessResult access_trap_aa32s_el1(...) {
    if (arm_current_el(env) == 3) return CP_ACCESS_OK;
    if (arm_is_secure_below_el3(env)) {
        if (scr_el3 & SCR_EEL2)
            return CP_ACCESS_TRAP_EL2;  // 有安全 EL2 时先 trap 到 EL2
        return CP_ACCESS_TRAP_EL3;
    }
    return CP_ACCESS_UNDEFINED;
}
```

---

## 5. MMU 配置体制（Regime）

### arm_mmu_idx_el — EL 到 MMU 体制映射

```c
// target/arm/helper.c:9957-10008
ARMMMUIdx arm_mmu_idx_el(CPUARMState *env, int el) {
    switch (el) {
    case 0:
        if ((hcr & (HCR_E2H | HCR_TGE)) == (HCR_E2H | HCR_TGE))
            return ARMMMUIdx_E20_0;   // VHE: EL0 使用 EL2 的 Stage1
        if (secure_below_el3 && !el3_aa64)
            return ARMMMUIdx_E30_0;   // Secure AArch32: EL0 使用 EL3 的 Stage1
        return ARMMMUIdx_E10_0;       // 正常: EL0 使用 EL1 的 Stage1

    case 1:
        if (arm_pan_enabled(env))
            return ARMMMUIdx_E10_1_PAN;  // PAN 模式
        return ARMMMUIdx_E10_1;

    case 2:
        if (hcr & HCR_E2H) {
            if (arm_pan_enabled(env))
                return ARMMMUIdx_E20_2_PAN;
            return ARMMMUIdx_E20_2;   // VHE: EL2 使用共享体制
        }
        return ARMMMUIdx_E2;          // 非 VHE: EL2 独立体制

    case 3:
        return ARMMMUIdx_E3;          // EL3 独立体制
    }
}
```

### MMU 体制全景

| 体制 | 使用者 | TTBR | TCR | 说明 |
|------|--------|------|-----|------|
| E10_0 | EL0 (正常) | TTBR0_EL1/TTBR1_EL1 | TCR_EL1 | 与 EL1 共享 Stage1 |
| E10_1 | EL1 | TTBR0_EL1/TTBR1_EL1 | TCR_EL1 | 内核态 |
| E10_1_PAN | EL1 + PAN | 同上 | 同上 | 用户页面不可直接访问 |
| E20_0 | EL0 (VHE) | TTBR0_EL2/TTBR1_EL2 | TCR_EL2 | VHE 下 EL0 |
| E20_2 | EL2 (VHE) | TTBR0_EL2/TTBR1_EL2 | TCR_EL2 | VHE 下 EL2 |
| E2 | EL2 (非 VHE) | TTBR0_EL2 | TCR_EL2 | 独立单地址空间 |
| E3 | EL3 | TTBR0_EL3 | TCR_EL3 | 安全监控独立 |
| Stage2 | EL2 控制 | VTTBR_EL2 | VTCR_EL2 | EL1&0 的第二阶段翻译 |

**关键点**：EL0 和 EL1 共享 Stage1 页表（TTBR0/TTBR1），但 EL1 可通过 PAN 限制直接访问用户页面。EL2 在 VHE 模式下也采用双地址空间（TTBR0/TTBR1）。

---

## 6. 内存访问权限与 PAN/UAO

### PAN（Privileged Access Never）

PAN 在 EL1 下阻止内核直接访问用户空间页面：

```c
// target/arm/tcg/hflags.c:348-364
// UNPRIV 标志 — 控制 LDTR/STTR 行为
if (!(env->pstate & PSTATE_UAO)) {
    switch (mmu_idx) {
    case ARMMMUIdx_E10_1:
    case ARMMMUIdx_E10_1_PAN:
        // EL1: LDTR/STTR 使用 EL0 权限（非特权访问）
        DP_TBFLAG_A64(flags, UNPRIV, 1);
        break;
    case ARMMMUIdx_E20_2:
    case ARMMMUIdx_E20_2_PAN:
        // VHE EL2: 仅 TGE=1 时 LDTR 使用 EL0 权限
        if (hcr_el2 & HCR_TGE) DP_TBFLAG_A64(flags, UNPRIV, 1);
        break;
    }
}
```

### UAO（User Access Override）

`PSTATE.UAO=1` 时，EL1/EL2 的 LDTR/STTR 使用当前特权级别的权限而非 EL0 权限。这使内核可以安全地使用 LDTR 访问内核空间。

### 各 EL 的内存访问模型

| EL | 普通 LDR/STR | LDTR/STTR | PAN 影响 |
|----|-------------|-----------|---------|
| EL0 | EL0 权限 | N/A | 无 |
| EL1 | EL1 权限 | EL0 权限（UAO=0） | PAN=1: EL0 页面不可直接 LDR/STR |
| EL1 | EL1 权限 | EL1 权限（UAO=1） | PAN 对 LDTR 也生效 |
| EL2(VHE) | EL2 权限 | EL0 权限（TGE=1） | 同 EL1 行为 |
| EL3 | EL3 权限 | N/A | 无 |

---

## 7. FP/SIMD/SVE/SME 陷阱控制

### 三级陷阱控制

```
CPACR_EL1.FPEN  →  控制 EL0/EL1 的 FP/SIMD 访问
CPTR_EL2.FPEN   →  控制 EL2 的 FP 陷阱（或 trap EL0/EL1 到 EL2）
CPTR_EL3.TFP    →  控制 EL3 的 FP 陷阱（trap 所有低 EL 到 EL3）
```

### hflags 中的 SVE/SME 陷阱级别

```c
// target/arm/tcg/hflags.c:264-308
// SVE 陷阱目标 EL
int sve_el = sve_exception_el(env, el);  // 返回 0(不陷阱)/1/2/3
DP_TBFLAG_A64(flags, SVEEXC_EL, sve_el);

// 若 SVE 不被陷阱，缓存向量长度
if (sve_el == 0)
    DP_TBFLAG_A64(flags, VL, sve_vqm1_for_el(env, el));

// SME 陷阱目标 EL
int sme_el = sme_exception_el(env, el);
DP_TBFLAG_A64(flags, SMEEXC_EL, sme_el);

// 流式模式标志
if (sm) {
    DP_TBFLAG_A64(flags, PSTATE_SM, 1);
    DP_TBFLAG_A64(flags, SME_TRAP_NONSTREAMING, !sme_fa64(env, el));
}
```

### 翻译时的访问检查

翻译器在遇到 FP/SVE/SME 指令时检查 DisasContext 中的陷阱级别：

```c
// fp_access_check_only(s) — 检查 s->fp_excp_el
// sve_access_check(s) — 检查 s->sve_excp_el 和 s->fp_excp_el
// sme_access_check(s) — 检查 s->sme_excp_el
```

若陷阱级别非 0，生成陷阱到目标 EL 的异常。

### SVE 向量长度随 EL 变化

SVE 向量长度（VL）可以按 EL 配置不同值：
- `ZCR_EL1.LEN` — EL0/EL1 的最大 VL
- `ZCR_EL2.LEN` — EL2 的最大 VL（也限制 EL0/EL1）
- `ZCR_EL3.LEN` — EL3 的最大 VL

EL 切换时，`aarch64_sve_change_el()` 处理 VL 变化（可能需要截断/零扩展寄存器）。

---

## 8. 中断屏蔽与 DAIF/ALLINT

### DAIF 屏蔽

```
PSTATE.D — Debug 异常屏蔽
PSTATE.A — SError 屏蔽
PSTATE.I — IRQ 屏蔽
PSTATE.F — FIQ 屏蔽
```

- 异常进入时 DAIF 全部置 1（所有异常屏蔽）
- 异常返回时从 SPSR 恢复
- EL0 不能写 DAIF（需要特权）

### ALLINT（FEAT_NMI）

```c
// PSTATE.ALLINT — 屏蔽所有物理中断（包括超级优先级中断）
// 仅在 SCTLR_ELx.SPINTMASK 允许时可设置
// HCRX_EL2.TALLINT — 从 EL1 设置 ALLINT 会 trap 到 EL2
```

### 中断路由与 EL 的关系

| 中断类型 | 路由控制 | EL3 截获 | EL2 截获 |
|---------|---------|---------|---------|
| IRQ | HCR_EL2.IMO | SCR_EL3.IRQ | HCR_EL2.IMO |
| FIQ | HCR_EL2.FMO | SCR_EL3.FIQ | HCR_EL2.FMO |
| SError | HCR_EL2.AMO | SCR_EL3.EA | HCR_EL2.AMO |
| NMI | ALLINT | SCR_EL3 | HCRX_EL2 |

---

## 9. 调试寄存器路由

### 三级调试陷阱

```c
// target/arm/debug_helper.c:21-77
// TDOSA — trap Debug OS 相关访问
static CPAccessResult access_tdosa(...) {
    if (el < 2 && (mdcr_el2 & MDCR_TDOSA || mdcr_el2 & MDCR_TDE || hcr & HCR_TGE))
        return CP_ACCESS_TRAP_EL2;
    if (el < 3 && (mdcr_el3 & MDCR_TDOSA))
        return CP_ACCESS_TRAP_EL3;
}

// TDRA — trap Debug ROM 访问
static CPAccessResult access_tdra(...) {
    if (el < 2 && (mdcr_el2 & MDCR_TDRA || MDCR_TDE || HCR_TGE))
        return CP_ACCESS_TRAP_EL2;
    if (el < 3 && (mdcr_el3 & MDCR_TDA))
        return CP_ACCESS_TRAP_EL3;
}

// TDA — trap 通用调试访问
static CPAccessResult access_tda(...) {
    if (el < 2 && (mdcr_el2 & MDCR_TDA || MDCR_TDE || HCR_TGE))
        return CP_ACCESS_TRAP_EL2;
    if (el < 3 && (mdcr_el3 & MDCR_TDA))
        return CP_ACCESS_TRAP_EL3;
}
```

### 调试控制寄存器层次

| 寄存器 | 控制范围 | 陷阱效果 |
|--------|---------|---------|
| MDSCR_EL1 | EL0/EL1 调试使能 | — |
| MDCR_EL2.TDA | EL0/EL1 调试访问 | → trap 到 EL2 |
| MDCR_EL2.TDE | EL0/EL1 全部调试 | → trap 到 EL2 |
| MDCR_EL2.TDOSA | Debug OS 访问 | → trap 到 EL2 |
| MDCR_EL3.TDA | EL0/EL1/EL2 调试访问 | → trap 到 EL3 |
| HCR_EL2.TGE | EL0 通用陷阱 | 增强 TDA/TDE 效果 |

---

## 10. VHE 寄存器重定向

VHE（HCR_EL2.E2H=1）使 EL2 像 EL1 一样运行宿主操作系统：

### 寄存器重定向规则

```c
// target/arm/helper.c:680-687
// CPACR_EL1 在 VHE 下重定向到 CPTR_EL2
{
    .name = "CPACR_EL1",
    .accessfn = cpacr_access,
    .vhe_redir_to_el2 = true,    // VHE: 访问重定向到 EL2 版本
    .vhe_redir_to_el01 = true,   // 反向重定向
}
```

### VHE 重定向表

| EL1 寄存器 | VHE 下实际访问 | 说明 |
|-----------|--------------|------|
| SCTLR_EL1 | SCTLR_EL2 | 系统控制 |
| TTBR0_EL1 | TTBR0_EL2 | 页表基地址 |
| TTBR1_EL1 | TTBR1_EL2 | 页表基地址 |
| TCR_EL1 | TCR_EL2 | 翻译控制 |
| MAIR_EL1 | MAIR_EL2 | 内存属性 |
| AMAIR_EL1 | AMAIR_EL2 | 辅助内存属性 |
| CPACR_EL1 | CPTR_EL2 | FP/SVE 控制 |
| VBAR_EL1 | VBAR_EL2 | 向量基地址 |
| CONTEXTIDR_EL1 | CONTEXTIDR_EL2 | 上下文 ID |

### VHE 对 MMU 的影响

```c
// target/arm/helper.c:9968-9977
// VHE 模式下：
// - EL0 使用 E20_0 体制（TTBR0_EL2/TTBR1_EL2）
// - EL2 使用 E20_2 体制（双地址空间）
// - Stage 2 翻译对 EL0/EL2 的访问不生效
if ((hcr & (HCR_E2H | HCR_TGE)) == (HCR_E2H | HCR_TGE))
    return ARMMMUIdx_E20_0;  // VHE EL0
```

---

## 11. 各 EL 差异对比表

### 指令可用性

| 指令/操作 | EL0 | EL1 | EL2 | EL3 |
|----------|-----|-----|-----|-----|
| SVC | ✓ | ✓ | ✓ | ✓ |
| HVC | ✗ | ✓ | ✓ | ✓ |
| SMC | ✗ | ✓ | ✓ | ✓ |
| ERET | ✗ | ✓ | ✓ | ✓ |
| MSR/MRS (PL1) | ✗ | ✓ | ✓ | ✓ |
| MSR/MRS (PL2) | ✗ | ✗ | ✓ | ✓ |
| MSR/MRS (PL3) | ✗ | ✗ | ✗ | ✓ |
| AT S1E1R/W | ✗ | ✓ | ✓(NV) | ✓ |
| TLBI | ✗ | ✓(部分) | ✓ | ✓ |
| DC ZVA | ✓(DCZID) | ✓ | ✓ | ✓ |

### MMU/内存体制

| 特性 | EL0 | EL1 | EL2 | EL3 |
|------|-----|-----|-----|-----|
| 地址空间 | TTBR0/1_EL1 | TTBR0/1_EL1 | TTBR0_EL2 (非VHE) | TTBR0_EL3 |
| Stage 2 | 受控 | 受控 | 不受 | 不受 |
| PAN | N/A | 可启用 | VHE 可启用 | AArch32 可 |
| UAO | N/A | 可启用 | 可启用 | N/A |
| MTE | 可配置 | 可配置 | 可配置 | 可配置 |

### 陷阱控制层次

```
EL0 操作 ──→ CPACR_EL1 检查 ──→ HCR_EL2/CPTR_EL2 检查 ──→ CPTR_EL3/SCR_EL3 检查
                  │                       │                         │
                  ▼                       ▼                         ▼
             trap→EL1              trap→EL2                   trap→EL3

EL1 操作 ──→ HCR_EL2/CPTR_EL2 检查 ──→ CPTR_EL3/SCR_EL3 检查
                     │                         │
                     ▼                         ▼
                trap→EL2                   trap→EL3

EL2 操作 ──→ CPTR_EL3/SCR_EL3 检查
                     │
                     ▼
                trap→EL3
```

---

## 12. 总结

### EL 切换对执行流的核心影响

1. **TB 标志重建**：每次 EL 变化后 `arm_rebuild_hflags()` 重新计算所有 EL 敏感的缓存标志，包括 MMU 体制、SVE/SME 陷阱级别、UNPRIV 标志、MTE 配置等。

2. **MMU 体制切换**：不同 EL 使用不同的 TTBR/TCR/MAIR 配置，EL0 与 EL1 共享 Stage1 但 EL2 有独立配置（VHE 例外）。

3. **陷阱级联**：EL0→EL1→EL2→EL3 形成层次化陷阱链，每一级都可以截获低级别的操作。

4. **指令集限制**：特权指令（HVC/SMC/ERET/系统寄存器访问）在翻译时检查 `current_el`，EL0 大部分特权操作直接 UNDEF。

5. **FP/SVE 向量长度**：VL 随 EL 变化（ZCR_ELx.LEN 限制），EL 切换时需要 `aarch64_sve_change_el()` 处理寄存器截断。

---

**关键源文件**：
- `target/arm/tcg/hflags.c` — `rebuild_hflags_a64()` EL 敏感的 TB 标志计算
- `target/arm/tcg/translate.h` — `DisasContext` 结构（current_el/mmu_idx/fp_excp_el 等）
- `target/arm/cpregs.h` — `ARMCPRegInfo` 权限标志定义
- `target/arm/helper.c` — 系统寄存器访问函数（access_tvm_trvm/tsw/tacr 等）、`arm_mmu_idx_el()`
- `target/arm/debug_helper.c` — 调试寄存器陷阱（access_tdosa/tdra/tda）
- `target/arm/tcg/translate-a64.c` — EL 检查的指令翻译（SVC/HVC/SMC/ERET）
