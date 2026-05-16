# 40 - ARM64 EL1/EL2 交互深度分析 — HVC 陷入、VHE 重定向、Stage-2 控制与嵌套虚拟化

> **基于 QEMU 11.0.50 源码**，深入分析 ARM64 EL1 与 EL2（Hypervisor）之间的交互机制：
> HVC 指令陷阱决策、异常进入 EL2 的向量偏移与状态保存、HCR_EL2 陷阱位全景、
> arm_hcr_el2_eff 有效值计算、VHE 寄存器重定向、Stage-2 使能对执行的影响、
> 细粒度陷阱（FGT）、WFE/WFI 陷阱链、嵌套虚拟化（NV/NV2）与 VNCR_EL2。

---

## 目录

1. [HVC 指令翻译与陷阱决策](#1-hvc-指令翻译与陷阱决策)
2. [异常进入 EL2](#2-异常进入-el2)
3. [HCR_EL2 位域全景](#3-hcr_el2-位域全景)
4. [arm_hcr_el2_eff：有效值计算](#4-arm_hcr_el2_eff有效值计算)
5. [VHE：寄存器重定向](#5-vhe寄存器重定向)
6. [VHE 对 MMU 索引的影响](#6-vhe-对-mmu-索引的影响)
7. [Stage-2 使能与 DC 位](#7-stage-2-使能与-dc-位)
8. [陷阱访问函数](#8-陷阱访问函数)
9. [细粒度陷阱（FGT）](#9-细粒度陷阱fgt)
10. [WFE/WFI 陷阱链](#10-wfewfi-陷阱链)
11. [ERET 从 EL2 返回](#11-eret-从-el2-返回)
12. [EL2 hflags 与 TB 标志](#12-el2-hflags-与-tb-标志)
13. [嵌套虚拟化（NV/NV2）](#13-嵌套虚拟化nvnv2)
14. [EL2 系统寄存器](#14-el2-系统寄存器)
15. [完整数据流](#15-完整数据流)

---

## 1. HVC 指令翻译与陷阱决策

### 1.1 TCG 翻译

```c
// translate-a64.c:3173-3190
static bool trans_HVC(DisasContext *s, arg_i *a)
{
    // EL3 执行 HVC → 异常到 EL3 自身
    // EL1/EL2 执行 HVC → 异常到 EL2
    int target_el = s->current_el == 3 ? 3 : 2;

    if (s->current_el == 0) {
        unallocated_encoding(s);    // EL0 不可用
        return true;
    }
    gen_helper_pre_hvc(tcg_env);    // 陷阱决策
    gen_exception_insn_el(s, 4, EXCP_HVC, syn_aa64_hvc(a->imm), target_el);
    return true;
}
```

### 1.2 pre_hvc Helper

```c
// op_helper.c:1071-1109
void HELPER(pre_hvc)(CPUARMState *env)
{
    bool undef;

    // ① PSCI 旁路：PSCI conduit=HVC 且有效调用 → 直接返回
    if (arm_is_psci_call(cpu, EXCP_HVC)) {
        return;
    }

    // ② 无 EL2 → UNDEF
    if (!arm_feature(env, ARM_FEATURE_EL2)) {
        undef = true;
    }
    // ③ 有 EL3 → SCR_EL3.HCE 优先（HCE=0 → UNDEF）
    else if (arm_feature(env, ARM_FEATURE_EL3)) {
        undef = !(env->cp15.scr_el3 & SCR_HCE);
    }
    // ④ 无 EL3 → HCR_EL2.HCD（HCD=1 → UNDEF）
    else {
        undef = env->cp15.hcr_el2 & HCR_HCD;
    }

    // ⑤ AArch32/安全态 EL1 → UNDEF
    if (secure && (!is_a64(env) || cur_el == 1)) {
        undef = true;
    }

    if (undef) {
        raise_exception(env, EXCP_UDEF, ...);
    }
}
```

**优先级**：PSCI > SCR_EL3.HCE > HCR_EL2.HCD > 安全态检查

---

## 2. 异常进入 EL2

```c
// helper.c:9198-9427（target_el = 2）
static void arm_cpu_do_interrupt_aarch64(CPUState *cs)
{
    vaddr addr = env->cp15.vbar_el[2];    // VBAR_EL2

    // 向量偏移计算
    if (cur_el < 2) {
        // 从 EL0/EL1 进入 EL2
        switch (new_el) {
        case 2:
            hcr = arm_hcr_el2_eff(env);
            if ((hcr & (HCR_E2H | HCR_TGE)) != (HCR_E2H | HCR_TGE)) {
                is_aa64 = (hcr & HCR_RW) != 0;  // HCR_RW 决定 EL1 宽度
            } else {
                is_aa64 = is_a64(env);            // VHE 模式
            }
        }
        addr += is_aa64 ? 0x400 : 0x600;
    } else {
        // EL2 → EL2
        addr += (pstate_read(env) & PSTATE_SP) ? 0x200 : 0;
    }

    // HVC syndrome 写入 ESR_EL2
    env->cp15.esr_el[2] = env->exception.syndrome;

    // 状态保存
    env->elr_el[2] = env->pc;                    // ELR_EL2
    env->banked_spsr[BANK_HYP] = old_mode;       // SPSR_EL2

    // PSTATE 设置（同 EL3 入口模式）
    pstate_write(env, PSTATE_DAIF | new_mode);    // DAIF 全屏蔽
    aarch64_restore_sp(env, 2);                    // SP_EL2
    arm_rebuild_hflags(env);                       // hflags → E2/E20_2
}
```

---

## 3. HCR_EL2 位域全景

```c
// cpu.h:1695-1754
#define HCR_VM        (1ULL << 0)   // 虚拟化 MMU 使能（Stage-2）
#define HCR_FMO       (1ULL << 3)   // FIQ 路由到 EL2
#define HCR_IMO       (1ULL << 4)   // IRQ 路由到 EL2
#define HCR_AMO       (1ULL << 5)   // SError 路由到 EL2
#define HCR_DC        (1ULL << 12)  // 默认可缓存（无 Stage-2 时）
#define HCR_TWI       (1ULL << 13)  // WFI 陷入 EL2
#define HCR_TWE       (1ULL << 14)  // WFE 陷入 EL2
#define HCR_TSC       (1ULL << 19)  // SMC 陷入 EL2
#define HCR_TVM       (1ULL << 26)  // 虚拟内存寄存器写陷入
#define HCR_TGE       (1ULL << 27)  // 通用异常陷入
#define HCR_HCD       (1ULL << 29)  // HVC 禁用
#define HCR_TRVM      (1ULL << 30)  // 虚拟内存寄存器读陷入
#define HCR_RW        (1ULL << 31)  // 低 EL 执行宽度
#define HCR_E2H       (1ULL << 34)  // EL2 Host（VHE）
#define HCR_NV        (1ULL << 42)  // 嵌套虚拟化
#define HCR_NV1       (1ULL << 43)  // NV 增强 1
#define HCR_NV2       (1ULL << 45)  // NV 增强 2
#define HCR_FWB       (1ULL << 46)  // Forced Write-Back
```

### 陷阱位分类

| 类别 | 位 | 作用 |
|------|-----|------|
| **中断路由** | FMO/IMO/AMO | IRQ/FIQ/SError → EL2 |
| **内存虚拟化** | VM/DC/PTW | Stage-2 使能/默认缓存/PTW 故障 |
| **指令陷阱** | TSC/HCD/TWI/TWE | SMC/HVC/WFI/WFE 陷入 |
| **寄存器陷阱** | TVM/TRVM/TACR/TIDCP | 写/读虚拟内存寄存器陷入 |
| **TLB 陷阱** | TTLB/TSW/TTLBIS/TTLBOS | TLB/缓存维护陷入 |
| **ID 陷阱** | TID0-TID5 | ID 寄存器读取陷入 |
| **VHE** | E2H/TGE | Host 扩展/通用异常陷入 |
| **NV** | NV/NV1/NV2/AT | 嵌套虚拟化控制 |
| **缓存** | FWB/TICAB/TOCU | 缓存属性控制 |

---

## 4. arm_hcr_el2_eff：有效值计算

```c
// helper.c:3862-3888
// arm_hcr_el2_eff_secstate() 计算 HCR_EL2 的有效值

// ① 安全态 + 无 Secure EL2 → 返回 0（HCR 完全无效）
// ② AArch32 过滤无效位

// ③ TGE=1 时的特殊处理：
if (ret & HCR_TGE) {
    if (ret & HCR_E2H) {
        // VHE 模式（E2H+TGE）：清除大量虚拟化位
        ret &= ~(HCR_VM | HCR_FMO | HCR_IMO | HCR_AMO |
                 HCR_DC | HCR_TWI | HCR_TWE |
                 HCR_TID0 | HCR_TID2 | HCR_TPCP | HCR_TPU |
                 HCR_TID4 | HCR_TICAB | HCR_TOCU | HCR_TID5 | ...);
    } else {
        // TGE 无 E2H：强制设置 FMO/IMO/AMO
        ret |= HCR_FMO | HCR_IMO | HCR_AMO;
    }
    // 两种情况都清除：
    ret &= ~(HCR_TVM | HCR_HCD | HCR_TRVM | HCR_TSC |
             HCR_TID1 | HCR_TID3 | HCR_TACR | HCR_TLOR | ...);
}
```

**关键设计**：
- **VHE（E2H+TGE）**：EL0 直接在 Host 下运行，无需虚拟化 → 清除 VM/DC/中断路由/陷阱
- **TGE 无 E2H**：所有异常路由到 EL2 → 强制 FMO/IMO/AMO
- 安全态无 SEL2 → HCR 完全无效（返回 0）

---

## 5. VHE：寄存器重定向

VHE（HCR_EL2.E2H=1）启用时，EL1 系统寄存器在 EL2 访问时重定向到 EL2 对应寄存器：

```c
// helper.c 中使用 .vhe_redir_to_el2 / .vhe_redir_to_el01 字段
// 共 28 对寄存器重定向

// 示例：
{ "SCTLR_EL1" → vhe_redir_to_el2 → SCTLR_EL2 }
{ "TTBR0_EL1" → vhe_redir_to_el2 → TTBR0_EL2 }
{ "TTBR1_EL1" → vhe_redir_to_el2 → TTBR1_EL2 }
{ "TCR_EL1"   → vhe_redir_to_el2 → TCR_EL2 }
{ "VBAR_EL1"  → vhe_redir_to_el2 → VBAR_EL2 }
{ "ELR_EL1"   → vhe_redir_to_el2 → ELR_EL2 }
{ "SPSR_EL1"  → vhe_redir_to_el2 → SPSR_EL2 }
{ "FAR_EL1"   → vhe_redir_to_el2 → FAR_EL2 }
{ "ESR_EL1"   → vhe_redir_to_el2 → ESR_EL2 }
{ "MAIR_EL1"  → vhe_redir_to_el2 → MAIR_EL2 }
{ "AMAIR_EL1" → vhe_redir_to_el2 → AMAIR_EL2 }
{ "CPTR_EL2"  → vhe_redir_to_el01 → CPACR_EL1 }
```

**重定向机制**：
- `vhe_redir_to_el2`：EL2 执行时，访问 `_EL1` 编码实际读写 `_EL2` 寄存器
- `vhe_redir_to_el01`：EL2 需要访问原始 EL1 寄存器时，使用 `_EL12` 编码

**el_is_in_host()**：

```c
// helper.c:3904-3927
bool el_is_in_host(CPUARMState *env, int el)
{
    // EL1/EL3 不在 Host → false
    if (el & 1) return false;
    // EL2 需要 E2H；EL0 需要 E2H + TGE
    mask = el ? HCR_E2H : HCR_E2H | HCR_TGE;
    if ((env->cp15.hcr_el2 & mask) != mask) return false;
    return arm_is_el2_enabled(env) && arm_el_is_aa64(env, 2);
}
```

---

## 6. VHE 对 MMU 索引的影响

```c
// helper.c:9957-10008
ARMMMUIdx arm_mmu_idx_el(CPUARMState *env, int el)
{
    switch (el) {
    case 0:
        if (hcr & (HCR_E2H | HCR_TGE) == (HCR_E2H | HCR_TGE)) {
            return ARMMMUIdx_E20_0;   // VHE Host EL0
        }
        return ARMMMUIdx_E10_0;       // 标准 Guest EL0
    case 2:
        if (env->cp15.hcr_el2 & HCR_E2H) {
            if (regime_is_pan(mmu_idx)) {
                return ARMMMUIdx_E20_2_PAN;
            }
            return ARMMMUIdx_E20_2;   // VHE Host EL2
        }
        return ARMMMUIdx_E2;          // 标准 EL2
    }
}
```

| 模式 | EL0 MMU 索引 | EL2 MMU 索引 | TTBR 来源 |
|------|-------------|-------------|-----------|
| 标准 | E10_0 | E2 | TTBR0_EL2（单范围） |
| VHE (E2H) | E20_0 | E20_2 | TTBR0/1_EL2（双范围） |

---

## 7. Stage-2 使能与 DC 位

### 7.1 HCR_EL2.VM — Stage-2 使能

```
VM=1 → Stage-2 翻译生效
  → EL0/EL1 地址翻译：VA → S1 → IPA → S2 → PA
  → TLB 使用两阶段翻译（get_phys_addr_twostage）

VM=0 → 仅 Stage-1
  → EL0/EL1 地址翻译：VA → S1 → PA
```

### 7.2 HCR_EL2.DC — 默认可缓存

```
DC=1（且 VM=0）→ Stage-1 禁用时，所有内存默认 Normal Write-Back Cacheable
  → 用于 Guest 启动早期尚未配置 MMU 的场景
  → 避免 Guest 以 Device 属性访问内存导致性能问题
```

### 7.3 VHE 模式下的 Stage-2

```c
// arm_hcr_el2_eff: TGE+E2H 时清除 VM 和 DC
if ((ret & HCR_TGE) && (ret & HCR_E2H)) {
    ret &= ~(HCR_VM | HCR_DC | ...);
}
// → VHE Host 模式下 Stage-2 自动禁用
// → Host OS 的 EL0 应用直接使用 Stage-1 翻译
```

---

## 8. 陷阱访问函数

### 8.1 TVM/TRVM — 虚拟内存寄存器陷阱

```c
// helper.c:332-343
CPAccessResult access_tvm_trvm(CPUARMState *env, const ARMCPRegInfo *ri,
                               bool isread)
{
    if (arm_current_el(env) == 1) {
        uint64_t trap = isread ? HCR_TRVM : HCR_TVM;
        if (arm_hcr_el2_eff(env) & trap) {
            return CP_ACCESS_TRAP_EL2;   // 陷入 EL2
        }
    }
    return CP_ACCESS_OK;
}
```

影响的寄存器：SCTLR_EL1、TCR_EL1、TTBR0/1_EL1、MAIR_EL1、AMAIR_EL1 等。

### 8.2 CP_ACCESS_TRAP 返回值

| 返回值 | 含义 |
|--------|------|
| CP_ACCESS_OK | 允许访问 |
| CP_ACCESS_TRAP_EL2 | 陷入 EL2（Hyp Trap） |
| CP_ACCESS_TRAP_EL3 | 陷入 EL3（Monitor Trap） |

---

## 9. 细粒度陷阱（FGT）

### 9.1 FGT 寄存器

```c
// helper.c:5652-5677
// HFGRTR_EL2 — 细粒度读取陷阱（Hypervisor Fine-Grained Read Trap）
// HFGWTR_EL2 — 细粒度写入陷阱
// HDFGRTR_EL2 — 细粒度调试读取陷阱
// HDFGWTR_EL2 — 细粒度调试写入陷阱
// HFGITR_EL2 — 细粒度指令陷阱（含 ERET 陷阱位）
```

### 9.2 FGT 门控

```c
// helper.c:5642-5650
static CPAccessResult access_fgt(CPUARMState *env, const ARMCPRegInfo *ri,
                                  bool isread)
{
    // EL2 访问 FGT 寄存器需要 SCR_EL3.FGTEN=1
    if (arm_current_el(env) == 2 &&
        arm_feature(env, ARM_FEATURE_EL3) &&
        !(env->cp15.scr_el3 & SCR_FGTEN)) {
        return CP_ACCESS_TRAP_EL3;
    }
    return CP_ACCESS_OK;
}
```

### 9.3 ERET 的 FGT 陷阱

```c
// hflags.c:374-382
if (arm_fgt_active(env, el)) {
    DP_TBFLAG_ANY(flags, FGT_ACTIVE, 1);
    if (FIELD_EX64(env->cp15.fgt_exec[FGTREG_HFGITR], HFGITR_EL2, ERET)) {
        DP_TBFLAG_A64(flags, TRAP_ERET, 1);
        // ERET 指令在 EL1 执行时陷入 EL2
    }
}
```

---

## 10. WFE/WFI 陷阱链

```c
// op_helper.c:322-365
static inline int check_wfx_trap(CPUARMState *env, bool is_wfe, uint32_t *excp)
{
    int cur_el = arm_current_el(env);

    // ① EL0 → SCTLR.nTWE/nTWI（陷入 EL1）
    if (cur_el < 1) {
        mask = is_wfe ? SCTLR_nTWE : SCTLR_nTWI;
        if (!(arm_sctlr(env, cur_el) & mask)) {
            return exception_target_el(env);
        }
    }

    // ② EL0/EL1 → HCR_EL2.TWE/TWI（陷入 EL2）
    if (cur_el < 2) {
        mask = is_wfe ? HCR_TWE : HCR_TWI;
        if (arm_hcr_el2_eff(env) & mask) {
            return 2;
        }
    }

    // ③ 非 EL3 → SCR_EL3.TWE/TWI（陷入 EL3）
    if (!arm_is_el3_or_mon(env)) {
        mask = is_wfe ? SCR_TWE : SCR_TWI;
        if (env->cp15.scr_el3 & mask) {
            return 3;
        }
    }

    return 0;  // 无陷阱
}
```

**陷阱优先级链**：SCTLR（→EL1）> HCR_EL2（→EL2）> SCR_EL3（→EL3）

---

## 11. ERET 从 EL2 返回

```c
// helper-a64.c:646-785
// ERET 从 EL2 返回的关键检查：

// ① 目标 EL 不能高于 EL2
if (new_el > cur_el) goto illegal_return;

// ② HCR_TGE 时不可返回到 EL1
if (new_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE)) {
    goto illegal_return;
}

// ③ 恢复 PSTATE + SP + hflags
pstate_write(env, spsr);
aarch64_restore_sp(env, new_el);
helper_rebuild_hflags_a64(env, new_el);
// → mmu_idx 从 E2/E20_2 切换到 E10_1/E10_0
```

---

## 12. EL2 hflags 与 TB 标志

### 12.1 rebuild_hflags_a64 的 EL2 特有行为

```c
// hflags.c:240-400（EL2 相关摘录）

// E2H 标志
if (hcr & HCR_E2H) {
    DP_TBFLAG_A64(flags, E2H, 1);
}

// UNPRIV（LDTR/STTR 在 VHE 下的 EL0 访问）
case ARMMMUIdx_E20_2:
case ARMMMUIdx_E20_2_PAN:
    if (env->cp15.hcr_el2 & HCR_TGE) {
        DP_TBFLAG_A64(flags, UNPRIV, 1);
    }

// NV 嵌套虚拟化标志
if (el == 1 && (hcr & HCR_NV)) {
    DP_TBFLAG_A64(flags, TRAP_ERET, 1);   // ERET 陷入 EL2
    DP_TBFLAG_A64(flags, NV, 1);
    if (hcr & HCR_NV1) DP_TBFLAG_A64(flags, NV1, 1);
    if (hcr & HCR_NV2) {
        DP_TBFLAG_A64(flags, NV2, 1);
        // NV2 内存后备寄存器的字节序
        if (env->cp15.sctlr_el[2] & SCTLR_EE) {
            DP_TBFLAG_A64(flags, NV2_MEM_BE, 1);
        }
    }
}
```

### 12.2 EL2 vs EL1 hflags 差异

| 标志 | EL1 (Guest) | EL2 (无 VHE) | EL2 (VHE) |
|------|-------------|--------------|-----------|
| MMUIDX | E10_1 | E2 | E20_2 |
| E2H | 0 | 0 | 1 |
| UNPRIV | 取决于 PAN | 0 | 取决于 TGE |
| NV/NV1/NV2 | 可能设置 | 0 | 0 |
| TRAP_ERET | NV 或 FGT | 0 | 0 |
| SVE/SME EL | CPTR_EL2 控制 | CPTR_EL2 | CPTR_EL2(重定向) |

---

## 13. 嵌套虚拟化（NV/NV2）

### 13.1 NV 基本原理

NV 允许 EL1 运行 Hypervisor 代码，EL2 作为"外部 Hypervisor"监管内部虚拟机。

```c
// cpu.h:1736-1739
#define HCR_NV        (1ULL << 42)  // 嵌套虚拟化使能
#define HCR_NV1       (1ULL << 43)  // NV 增强
#define HCR_NV2       (1ULL << 45)  // NV2 内存后备
```

### 13.2 NV 陷阱行为

```c
// hflags.c:384-399
if (el == 1 && (hcr & HCR_NV)) {
    // EL1 执行以下操作会陷入 EL2：
    // - ERET 指令（TRAP_ERET）
    // - EL2 系统寄存器访问（NV 自动陷阱）
    // - SMC 指令（通过 HCR_NV）
    // - AT/TLBI 指令（HCR_AT 位控制）
}
```

### 13.3 NV2 — 内存后备寄存器

```c
// helper.c:5680-5700
static void vncr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    // VNCR_EL2 存储内存后备区域基地址
    // 底部 12 位清零 → 确保 64 位对齐
    env->cp15.vncr_el2 = value & ~0xfffULL;
}
```

**NV2 机制**：
- `HCR_NV2=1` 时，EL1 访问 EL2 系统寄存器不再陷入 EL2
- 而是通过 `VNCR_EL2` 指向的内存区域进行读写
- 每个 EL2 寄存器在 VNCR 页面中有固定偏移（`nv2_redirect_offset`）
- 减少 VM Exit 次数，提升嵌套虚拟化性能

### 13.4 nv2_redirect_offset 示例

```c
// 每个 EL2 寄存器定义中包含 NV2 偏移
{ "VNCR_EL2",   nv2_redirect_offset = 0xb0 }
{ "HFGRTR_EL2", nv2_redirect_offset = 0x1b8 }
{ "HFGWTR_EL2", nv2_redirect_offset = 0x1c0 }
{ "HFGITR_EL2", nv2_redirect_offset = 0x1c8 }
```

---

## 14. EL2 系统寄存器

```c
// helper.c 中的 EL2 寄存器定义（opc1=4）
{ "HCR_EL2",     opc1=4, crn=1, crm=1, opc2=0,  PL2_RW }
{ "VTTBR_EL2",   opc1=4, crn=2, crm=1, opc2=0,  PL2_RW }  // [4155]
{ "VTCR_EL2",    opc1=4, crn=2, crm=1, opc2=2,  PL2_RW }  // [4163]
{ "ELR_EL2",     opc1=4, crn=4, crm=0, opc2=1,  PL2_RW }
{ "SPSR_EL2",    opc1=4, crn=4, crm=0, opc2=0,  PL2_RW }
{ "ESR_EL2",     opc1=4, crn=5, crm=2, opc2=0,  PL2_RW }
{ "FAR_EL2",     opc1=4, crn=6, crm=0, opc2=0,  PL2_RW }
{ "VBAR_EL2",    opc1=4, crn=12, crm=0, opc2=0, PL2_RW }
{ "VMPIDR_EL2",  opc1=4, crn=0, crm=0, opc2=5,  PL2_RW }  // [6807]
{ "VPIDR_EL2",   opc1=4, crn=0, crm=0, opc2=0,  PL2_RW }  // [6819]
{ "CNTHCTL_EL2", opc1=4, crn=14, crm=1, opc2=0, PL2_RW }  // [4190]
```

---

## 15. 完整数据流

### 15.1 HVC 从 EL1 进入 EL2

```
Guest EL1 执行 HVC #imm
  │
  ▼
trans_HVC()                              [translate-a64.c:3173]
  ├── target_el = 2（EL1 → EL2）
  ├── gen_helper_pre_hvc()               陷阱决策
  │     ├── PSCI-via-HVC → 旁路返回
  │     ├── 无 EL2 → UNDEF
  │     ├── SCR_EL3.HCE=0 → UNDEF（优先于 HCD）
  │     ├── HCR_EL2.HCD=1 → UNDEF
  │     └── 安全态 AArch32 EL1 → UNDEF
  └── gen_exception_insn_el(EXCP_HVC, 2)
  │
  ▼
arm_cpu_do_interrupt_aarch64()           [helper.c:9198]
  ├── VBAR_EL2 + 偏移
  │     ├── 低 EL AArch64 → +0x400
  │     └── 低 EL AArch32 → +0x600
  ├── ESR_EL2 = HVC syndrome
  ├── ELR_EL2 = 返回地址
  ├── SPSR_EL2 = 旧 PSTATE
  ├── PSTATE = DAIF=1111, EL=2, SP=1
  └── arm_rebuild_hflags → mmu_idx = E2 或 E20_2
```

### 15.2 VHE 模式下 EL0 执行

```
VHE Host（E2H=1, TGE=1）
  │
  ├── EL0 应用
  │   ├── mmu_idx = E20_0（Host EL0）
  │   ├── 使用 TTBR0_EL2 / TTBR1_EL2
  │   ├── 无 Stage-2（VM 被清除）
  │   └── 异常直接到 EL2（TGE）
  │
  ├── EL2 Host Kernel
  │   ├── mmu_idx = E20_2
  │   ├── 访问 SCTLR_EL1 → 重定向到 SCTLR_EL2
  │   ├── 访问 SCTLR_EL12 → 实际 EL1 寄存器
  │   └── CPTR_EL2 重定向为 CPACR_EL1 格式
  │
  └── arm_hcr_el2_eff 清除：
      VM/DC/FMO/IMO/AMO/TWI/TWE/TID*/TPCP/TPU/...
```

### 15.3 NV 嵌套虚拟化

```
外部 Hypervisor（EL2）
  │ 设置 HCR_EL2.NV=1, NV2=1
  │ 设置 VNCR_EL2 → 内存后备页面
  ▼
内部 Hypervisor（EL1）
  │
  ├── 访问 EL2 寄存器（如 HCR_EL2）
  │   ├── NV 无 NV2 → 陷入 EL2
  │   └── NV2 → 读写 VNCR_EL2 + offset 内存
  │
  ├── ERET 指令
  │   └── TRAP_ERET → 陷入 EL2
  │
  ├── AT/TLBI 指令
  │   └── HCR_AT 等位控制 → 可能陷入 EL2
  │
  └── SMC 指令
      └── HCR_NV → 可以陷入 EL2
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `target/arm/tcg/translate-a64.c` | ~3205 | trans_HVC (3173-3190)、trans_ERET (1951-1974) |
| `target/arm/tcg/op_helper.c` | ~1200 | HELPER(pre_hvc) (1071-1109)、check_wfx_trap (322-365) |
| `target/arm/helper.c` | ~10190 | arm_hcr_el2_eff (3862-3888)、el_is_in_host (3904-3927)、access_tvm_trvm (332-343)、access_fgt (5642-5650)、FGT 寄存器 (5652-5677)、VNCR_EL2 (5680-5700)、EL2 系统寄存器 (4155-4199, 6807-6824)、VHE 重定向（28 对 vhe_redir）、arm_cpu_do_interrupt_aarch64 (9198-9427)、arm_mmu_idx_el (9957-10008) |
| `target/arm/tcg/helper-a64.c` | ~785 | HELPER(exception_return) (646-785) |
| `target/arm/tcg/hflags.c` | ~575 | rebuild_hflags_a64 (240-400)：E2H/NV/NV2/TRAP_ERET/FGT 标志 |
| `target/arm/cpu.h` | ~1755 | HCR_EL2 位定义 (1695-1754)：60+ 位域 |
| `target/arm/tcg/psci.c` | ~120 | arm_is_psci_call (30-56)：HVC PSCI 旁路 |
