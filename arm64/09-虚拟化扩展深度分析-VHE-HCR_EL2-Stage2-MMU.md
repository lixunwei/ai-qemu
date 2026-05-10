# ARM64 虚拟化扩展深度分析 — VHE / HCR_EL2 / Stage-2 MMU

> QEMU 11.0.50 · 基于 commit 2e4de6e 及后续  
> 交叉引用：[06-异常级别状态管理深度分析.md](06-异常级别状态管理深度分析.md) · [07-不同EL下指令执行差异深度分析.md](07-不同EL下指令执行差异深度分析.md) · [08-TrustZone安全扩展与Secure-World深度分析.md](08-TrustZone安全扩展与Secure-World深度分析.md)

---

## 目录

1. [概述](#1-概述)
2. [EL2 使能与初始化](#2-el2-使能与初始化)
3. [HCR_EL2 寄存器定义](#3-hcr_el2-寄存器定义)
4. [HCR_EL2 写入与副作用](#4-hcr_el2-写入与副作用)
5. [有效 HCR 计算 — arm_hcr_el2_eff()](#5-有效-hcr-计算--arm_hcr_el2_eff)
6. [VHE — E2H 与 TGE 位](#6-vhe--e2h-与-tge-位)
7. [VHE 寄存器重定向机制](#7-vhe-寄存器重定向机制)
8. [ELIsInHost 判定](#8-elisinhost-判定)
9. [HCR_EL2 Trap 位分类与检查](#9-hcr_el2-trap-位分类与检查)
10. [中断虚拟化 — FMO/IMO/AMO 与 VI/VF/VSE](#10-中断虚拟化--fmoimoamo-与-vivfvse)
11. [TB Flags 中的 EL2/NV 标志](#11-tb-flags-中的-el2nv-标志)
12. [MMU Index 与 EL2 翻译体制](#12-mmu-index-与-el2-翻译体制)
13. [Stage-2 翻译概述](#13-stage-2-翻译概述)
14. [VTTBR_EL2 与 VTCR_EL2](#14-vttbr_el2-与-vtcr_el2)
15. [Stage-2 页表遍历实现](#15-stage-2-页表遍历实现)
16. [Stage-2 起始级别校验 — check_s2_mmu_setup()](#16-stage-2-起始级别校验--check_s2_mmu_setup)
17. [Stage-2 访问权限 — get_S2prot()](#17-stage-2-访问权限--get_s2prot)
18. [Stage-2 间接权限 — S2PIR_EL2](#18-stage-2-间接权限--s2pir_el2)
19. [Stage-2 缓存属性与 FWB](#19-stage-2-缓存属性与-fwb)
20. [Stage-2 TLB 管理](#20-stage-2-tlb-管理)
21. [Stage-2 故障处理](#21-stage-2-故障处理)
22. [HCR_EL2 对 Stage-2 的影响](#22-hcr_el2-对-stage-2-的影响)
23. [异常路由到 EL2](#23-异常路由到-el2)
24. [EL2 异常入口 — arm_cpu_do_interrupt_aarch64()](#24-el2-异常入口--arm_cpu_do_interrupt_aarch64)
25. [嵌套虚拟化 — NV/NV2](#25-嵌套虚拟化--nvnv2)
26. [虚拟定时器](#26-虚拟定时器)
27. [EL2 系统寄存器表](#27-el2-系统寄存器表)
28. [virt 机器 EL2 配置](#28-virt-机器-el2-配置)
29. [KVM 与 EL2](#29-kvm-与-el2)
30. [完整数据流时序图](#30-完整数据流时序图)

**附录**  
[A. 关键源文件索引](#附录-a-关键源文件索引)  
[B. HCR_EL2 位域完整映射表](#附录-b-hcr_el2-位域完整映射表)  
[C. 关键 Git 提交历史](#附录-c-关键-git-提交历史)

---

## 1. 概述

ARM64 虚拟化扩展是 Hypervisor（EL2）运行的硬件基础。QEMU 在 TCG 模式下完整模拟了这些扩展，在 KVM 模式下则利用真实硬件 EL2。本文档深度分析三大核心机制：

1. **HCR_EL2**（Hypervisor Configuration Register）— 60+ 位控制位，控制 trap、中断路由、内存虚拟化
2. **VHE**（Virtualization Host Extensions）— E2H+TGE 使 EL2 直接运行 Host OS
3. **Stage-2 MMU** — 二级地址翻译，将 IPA（中间物理地址）翻译为 PA（物理地址）

```
                    ┌──────────────────┐
                    │   Guest OS (EL1) │
                    └────────┬─────────┘
                             │ VA → IPA (Stage-1)
                    ┌────────▼─────────┐
                    │  Hypervisor EL2  │
                    │  HCR_EL2 控制    │
                    └────────┬─────────┘
                             │ IPA → PA (Stage-2)
                    ┌────────▼─────────┐
                    │   物理内存 (PA)   │
                    └──────────────────┘
```

---

## 2. EL2 使能与初始化

### 2.1 virt 机器的 virtualization 属性

```
hw/arm/virt.c:3882-3883
  object_class_property_add_bool(oc, "virtualization",
                                 virt_get_virt, virt_set_virt);
```

- 默认 `vms->virt = false`（`virt.c:4018`）
- 当 `virtualization=off` 时，CPU 的 `has_el2` 被强制关闭：

```c
// virt.c:2764-2765
if (!vms->virt && object_property_find(cpuobj, "has_el2")) {
    object_property_set_bool(cpuobj, "has_el2", false, NULL);
}
```

### 2.2 arm_is_el2_enabled()

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

**关键逻辑**：
- 必须有 `ARM_FEATURE_EL2`
- 在 Secure 状态下还需要 `SCR_EL3.EEL2`（ARMv8.4-SecEL2）
- 若 EL2 未使能，`arm_hcr_el2_eff()` 返回 0（所有 HCR 位无效）

---

## 3. HCR_EL2 寄存器定义

所有位域定义在 `cpu.h:1695-1754`，共计 60 个控制位：

```c
// cpu.h:1695-1754（精选）
#define HCR_VM        (1ULL << 0)   // Stage-2 使能
#define HCR_SWIO      (1ULL << 1)   // Set/Way 无效化
#define HCR_PTW       (1ULL << 2)   // Protected Table Walk
#define HCR_FMO       (1ULL << 3)   // FIQ 路由到 EL2
#define HCR_IMO       (1ULL << 4)   // IRQ 路由到 EL2
#define HCR_AMO       (1ULL << 5)   // SError 路由到 EL2
#define HCR_VF        (1ULL << 6)   // 虚拟 FIQ
#define HCR_VI        (1ULL << 7)   // 虚拟 IRQ
#define HCR_VSE       (1ULL << 8)   // 虚拟 SError
#define HCR_DC        (1ULL << 12)  // 默认可缓存
#define HCR_TWI       (1ULL << 13)  // Trap WFI
#define HCR_TWE       (1ULL << 14)  // Trap WFE
#define HCR_TSC       (1ULL << 19)  // Trap SMC
#define HCR_TVM       (1ULL << 26)  // Trap 虚拟内存控制
#define HCR_TGE       (1ULL << 27)  // Trap 通用异常
#define HCR_RW        (1ULL << 31)  // EL1 执行模式 (AArch64)
#define HCR_E2H       (1ULL << 34)  // EL2 Host Extensions
#define HCR_NV        (1ULL << 42)  // 嵌套虚拟化
#define HCR_NV1       (1ULL << 43)  // 嵌套虚拟化 1
#define HCR_NV2       (1ULL << 45)  // 嵌套虚拟化 2
#define HCR_FWB       (1ULL << 46)  // Forced Write-Back
```

按功能分类见[附录 B](#附录-b-hcr_el2-位域完整映射表)。

---

## 4. HCR_EL2 写入与副作用

### 4.1 写入处理函数

```c
// helper.c:3786-3788
static void hcr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                      uint64_t value) {
    do_hcr_write(env, value, 0);
}
```

### 4.2 do_hcr_write() 核心逻辑

**步骤 1 — 有效位掩码过滤**（`helper.c:3640-3737`）：
- 基础 valid_mask 包含 AArch32/AArch64 公共位
- 根据 CPU 特性动态添加：VHE(E2H)、RAS(TERR/TEA)、Pauth(API/APK)、MTE(ATA/DCT/TID5)、NV/NV2 等
- 清除 RES0 位：`value &= valid_mask`

**步骤 2 — 强制位设置**（`helper.c:3739-3748`）：
```c
// 若 EL1 只支持 AArch64，RW 位为 RAO/WI
if (!cpu_isar_feature(aa64_aa32_el1, cpu)) {
    value |= HCR_RW;
}
// 若不支持 FEAT_E2H0，E2H 为 RES1
if (!cpu_isar_feature(aa64_e2h0, cpu)) {
    value |= HCR_E2H;
}
```

**步骤 3 — MMU 相关位变化时刷新 TLB**（`helper.c:3759-3762`）：
```c
if ((env->cp15.hcr_el2 ^ value) &
    (HCR_VM | HCR_PTW | HCR_DC | HCR_DCT | HCR_FWB | HCR_NV | HCR_NV1)) {
    tlb_flush(CPU(cpu));
}
```

**步骤 4 — 更新虚拟中断状态**（`helper.c:3776-3783`）：
```c
g_assert(bql_locked());
arm_cpu_update_virq(cpu);
arm_cpu_update_vfiq(cpu);
arm_cpu_update_vserr(cpu);
if (cpu_isar_feature(aa64_nmi, cpu)) {
    arm_cpu_update_vinmi(cpu);
    arm_cpu_update_vfnmi(cpu);
}
```

> VI/VF/VSE 位变化后立即更新虚拟中断线状态，但由于只有 EL0/EL1 才能接收虚拟中断，在 EL2 写入 HCR 后不会立即触发。

---

## 5. 有效 HCR 计算 — arm_hcr_el2_eff()

```c
// helper.c:3818-3889
uint64_t arm_hcr_el2_eff_secstate(CPUARMState *env, ARMSecuritySpace space)
{
    uint64_t ret = env->cp15.hcr_el2;

    // EL2 未使能 → 所有位无效
    if (!arm_is_el2_enabled_secstate(env, space)) {
        return 0;
    }

    // AArch32 EL2：过滤掉 AArch64-only 位
    if (!arm_el_is_aa64(env, 2)) {
        uint64_t aa32_valid = MAKE_64BIT_MASK(0, 32) & ~(HCR_RW | HCR_TDZ);
        aa32_valid |= (HCR_CD | HCR_ID | HCR_TERR | ...);
        ret &= aa32_valid;
    }

    // TGE 位的副作用（关键！）
    if (ret & HCR_TGE) {
        if (ret & HCR_E2H) {
            // VHE Host 模式：清除大量 trap/虚拟化位
            ret &= ~(HCR_VM | HCR_FMO | HCR_IMO | HCR_AMO |
                     HCR_BSU_MASK | HCR_DC | HCR_TWI | HCR_TWE |
                     HCR_TID0 | HCR_TID2 | ...);
        } else {
            // 非 VHE 的 TGE=1：强制 FMO/IMO/AMO = 1
            ret |= HCR_FMO | HCR_IMO | HCR_AMO;
        }
        // TGE 始终清除这些位
        ret &= ~(HCR_SWIO | HCR_PTW | HCR_VF | HCR_VI | HCR_VSE |
                 HCR_FB | HCR_TID1 | HCR_TID3 | HCR_TSC | HCR_TACR |
                 HCR_TSW | HCR_TTLB | HCR_TVM | HCR_HCD | HCR_TRVM |
                 HCR_TLOR);
    }
    return ret;
}
```

这是 QEMU 虚拟化控制的**核心函数**，被大量调用点引用。

---

## 6. VHE — E2H 与 TGE 位

### 6.1 VHE 核心思想

传统模式下，Host OS 运行在 EL1，Hypervisor 在 EL2 负责 trap 和转发。VHE（FEAT_VHE）允许 Host OS 直接运行在 EL2，消除了 EL1↔EL2 切换开销。

```
传统模式：                    VHE 模式：
┌──────────┐                 ┌──────────┐
│ Guest EL1│                 │ Guest EL1│
│ Guest EL0│                 │ Guest EL0│
├──────────┤                 ├──────────┤
│ Hyp  EL2 │ ← trap/转发     │ Host EL2 │ ← 直接运行 OS
├──────────┤                 │          │ (E2H=1, TGE=1)
│ Host EL1 │ ← OS 运行       └──────────┘
│ Host EL0 │
└──────────┘
```

### 6.2 E2H 与 TGE 组合效果

| E2H | TGE | 模式 | 行为 |
|-----|-----|------|------|
| 0 | 0 | 传统 Guest | EL0/EL1 受 Stage-2 和 trap 控制 |
| 0 | 1 | TGE 模式 | EL0 异常路由到 EL2，FMO/IMO/AMO 强制 |
| 1 | 0 | VHE Guest | EL2 使用 E1x 别名寄存器管理 Guest |
| 1 | 1 | VHE Host | EL2 运行 Host OS，Stage-2/trap 位被清除 |

---

## 7. VHE 寄存器重定向机制

### 7.1 重定向原理

VHE 模式下，EL1 的系统寄存器（如 `SCTLR_EL1`）实际访问 EL2 的对应寄存器（`SCTLR_EL2`），同时提供 `_EL12` 后缀的别名供 Hypervisor 管理 Guest 使用。

```
VHE 下 EL2 访问 SCTLR_EL1 → 实际操作 SCTLR_EL2  (重定向)
VHE 下 EL2 访问 SCTLR_EL12 → 实际操作 SCTLR_EL1 (访问 Guest 寄存器)
```

### 7.2 QEMU 实现

在 `add_cpreg_to_hashtable_aa64()`（`helper.c:7651-7738`）中：

```c
// helper.c:7683-7737
if (!r->vhe_redir_to_el01) {
    assert(!r->vhe_redir_to_el2);
} else if (!arm_feature(&cpu->env, ARM_FEATURE_EL2) ||
           !cpu_isar_feature(aa64_vh, cpu)) {
    // 不支持 VHE → 清除重定向
    r->vhe_redir_to_el2 = 0;
    r->vhe_redir_to_el01 = 0;
} else {
    // 创建 FOO_EL12 别名
    ARMCPRegInfo *r2 = alloc_cpreg(r, "2");
    uint32_t key2 = r->vhe_redir_to_el01;

    r->vhe_redir_to_el01 = 0;
    r2->vhe_redir_to_el2 = 0;
    r2->vhe_redir_to_el01 = key;  // 从 _EL12 指回 _EL1

    r2->type |= ARM_CP_ALIAS | ARM_CP_NO_RAW;
    r2->access &= PL2_RW | PL3_RW;
    // ... 设置 encoding 字段
}
```

### 7.3 重定向示例

以 `FAR_EL1` 为例（`helper.c:2830-2835`）：
```c
{ .name = "FAR_EL1", ...
  .vhe_redir_to_el2 = ENCODE_AA64_CP_REG(3, 4, 6, 0, 0),
  // 指向 FAR_EL2 的 encoding
```

安装后产生三个寄存器：
- `FAR_EL1`：正常 EL1 访问；VHE 下 EL2 访问时重定向到 `FAR_EL2`
- `FAR_EL12`：VHE 下 EL2 访问 Guest 的 `FAR_EL1`
- `FAR_EL2`：正常 EL2 寄存器

---

## 8. ELIsInHost 判定

```c
// helper.c:3904-3927
bool el_is_in_host(CPUARMState *env, int el)
{
    uint64_t mask;

    // EL1/EL3 永远不在 Host 中
    if (el & 1) return false;

    // EL2 只需 E2H；EL0 需要 E2H+TGE
    mask = el ? HCR_E2H : HCR_E2H | HCR_TGE;
    if ((env->cp15.hcr_el2 & mask) != mask) return false;

    // 还需 EL2 已使能且 AArch64
    return arm_is_el2_enabled(env) && arm_el_is_aa64(env, 2);
}
```

**返回 true 的条件**：
- EL2：`E2H=1` 且 EL2 使能
- EL0：`E2H=1 && TGE=1` 且 EL2 使能

---

## 9. HCR_EL2 Trap 位分类与检查

### 9.1 Trap 位分类

| 类别 | 位 | 作用 | 检查函数 |
|------|-----|------|---------|
| **指令 Trap** | TWI | Trap WFI | `check_wfx_trap()` (`op_helper.c:322`) |
| | TWE | Trap WFE | `check_wfx_trap()` |
| | TSC | Trap SMC | `pre_smc()` (`op_helper.c:1155`) |
| | HCD | Trap HVC disable | `trans_HVC()` (`translate-a64.c`) |
| **内存控制 Trap** | TVM | Trap 虚存控制写 | `access_tvm_trvm()` (`helper.c:333`) |
| | TRVM | Trap 虚存控制读 | `access_tvm_trvm()` |
| | TSW | Trap Set/Way 操作 | `access_tsw()` (`helper.c:345`) |
| | TTLB | Trap TLBI | `access_ttlb()` (`tlb-insns.c:17`) |
| | TPU/TPCP | Trap PoU/PoC 操作 | `access_tocu()` / `access_tpcp()` |
| **ID 寄存器 Trap** | TID0 | Trap ID_PFR0 等 | `access_tid0()` |
| | TID1 | Trap AIDR 等 | `access_tid1()` (`helper.c:928`) |
| | TID2 | Trap CTR/CCSIDR | `access_tid2()` |
| | TID3 | Trap ID_AA64* | `access_tid3()` (`helper.c:5784`) |
| | TID4 | Trap MRS 编码空间 | `access_tid4()` (`helper.c:847`) |
| | TID5 | Trap GMID_EL1 | `access_tid5()` (`helper.c:5365`) |
| **其他控制 Trap** | TACR | Trap ACTLR | `access_tacr()` (`helper.c:357`) |
| | TERR | Trap 错误寄存器 | `access_terr()` (`helper.c:4504`) |
| | TLOR | Trap LOR 寄存器 | `access_lor_ns()` |

### 9.2 Trap 检查模式

所有 trap 函数返回 `CPAccessResult`，典型模式：
```c
// helper.c:333-343
static CPAccessResult access_tvm_trvm(CPUARMState *env,
                                       const ARMCPRegInfo *ri, bool isread) {
    if (arm_current_el(env) == 1) {
        uint64_t trap = isread ? HCR_TRVM : HCR_TVM;
        if (arm_hcr_el2_eff(env) & trap) {
            return CP_ACCESS_TRAP_EL2;
        }
    }
    return CP_ACCESS_OK;
}
```

---

## 10. 中断虚拟化 — FMO/IMO/AMO 与 VI/VF/VSE

### 10.1 中断路由

`arm_excp_unmasked()`（`cpu-irq.c:15-80`）控制中断是否被屏蔽：

```c
// cpu-irq.c:46-77
case EXCP_VINMI:
    // 虚拟 NMI IRQ，仅在 IMO=1 且 TGE=0 时有效
    if (!(hcr_el2 & HCR_IMO) || (hcr_el2 & HCR_TGE))
        return false;
    return !allIntMask;

case EXCP_VFIQ:
    if (!(hcr_el2 & HCR_FMO) || (hcr_el2 & HCR_TGE))
        return false;
    return !(env->daif & PSTATE_F) && (!allIntMask);

case EXCP_VIRQ:
    if (!(hcr_el2 & HCR_IMO) || (hcr_el2 & HCR_TGE))
        return false;
    return !(env->daif & PSTATE_I) && (!allIntMask);

case EXCP_VSERR:
    if (!(hcr_el2 & HCR_AMO) || (hcr_el2 & HCR_TGE))
        return false;
    ...
```

### 10.2 虚拟中断注入

HCR_EL2 写入后更新虚拟中断线：
```c
// helper.c:3777-3783
arm_cpu_update_virq(cpu);   // VI 位 → EXCP_VIRQ
arm_cpu_update_vfiq(cpu);   // VF 位 → EXCP_VFIQ
arm_cpu_update_vserr(cpu);  // VSE 位 → EXCP_VSERR
arm_cpu_update_vinmi(cpu);  // VINMI (NMI)
arm_cpu_update_vfnmi(cpu);  // VFNMI (NMI)
```

### 10.3 路由总结

```
物理 IRQ ──┬── IMO=0 → EL1 处理 (直接)
           └── IMO=1 → EL2 处理 (trap)
                         │
                         ▼
           Hypervisor 注入 HCR.VI=1
                         │
                         ▼
           Guest EL1 收到 EXCP_VIRQ
```

---

## 11. TB Flags 中的 EL2/NV 标志

`rebuild_hflags_a64()`（`hflags.c:240-400`）预计算 EL2/NV 相关 TB 标志：

| 标志 | 设置条件 | 用途 |
|------|---------|------|
| `E2H` | `hcr & HCR_E2H` | VHE 寄存器重定向 |
| `NV` | `el==1 && (hcr & HCR_NV)` | 嵌套虚拟化 trap |
| `NV1` | `hcr & HCR_NV1`（NV 有效时） | NV1 行为修改 |
| `NV2` | `hcr & HCR_NV2`（NV 有效时） | NV2 内存重定向 |
| `NV2_MEM_BE` | `sctlr_el[2] & SCTLR_EE` | NV2 内存大端序 |
| `TRAP_ERET` | `FGT ERET` 或 `NV` | Trap ERET 指令 |
| `UNPRIV` | E20_2 + TGE 等条件 | LDTR 行为 |
| `FGT_ACTIVE` | `arm_fgt_active()` | 细粒度 trap |

```c
// hflags.c:388-400
if (el == 1 && (hcr & HCR_NV)) {
    DP_TBFLAG_A64(flags, TRAP_ERET, 1);
    DP_TBFLAG_A64(flags, NV, 1);
    if (hcr & HCR_NV1) {
        DP_TBFLAG_A64(flags, NV1, 1);
    }
    if (hcr & HCR_NV2) {
        DP_TBFLAG_A64(flags, NV2, 1);
        if (env->cp15.sctlr_el[2] & SCTLR_EE) {
            DP_TBFLAG_A64(flags, NV2_MEM_BE, 1);
        }
    }
}
```

---

## 12. MMU Index 与 EL2 翻译体制

### 12.1 ARMMMUIdx 枚举

```c
// mmuidx.h:137-176
typedef enum ARMMMUIdx {
    // EL1&0 翻译体制（受 Stage-2 管控）
    ARMMMUIdx_E10_0      = 0,  // EL0, regime EL1&0
    ARMMMUIdx_E10_1      = 2,  // EL1, regime EL1&0
    ARMMMUIdx_E10_1_PAN  = 3,  // EL1 + PAN

    // EL2&0 翻译体制（VHE 模式）
    ARMMMUIdx_E20_0      = 5,  // EL0 in VHE Host
    ARMMMUIdx_E20_2      = 7,  // EL2 in VHE Host
    ARMMMUIdx_E20_2_PAN  = 8,  // EL2 + PAN in VHE

    // EL2 独立翻译体制（非 VHE）
    ARMMMUIdx_E2         = 10, // EL2 non-VHE

    // Stage-2 专用
    ARMMMUIdx_Stage2_S   = 16, // Secure Stage-2
    ARMMMUIdx_Stage2     = 17, // Non-secure Stage-2

    // 物理地址空间（直通）
    ARMMMUIdx_Phys_S     = 18,
    ARMMMUIdx_Phys_NS    = 19,
    ...
} ARMMMUIdx;
```

### 12.2 翻译体制对应关系

| MMU Index | 翻译体制 | Stage-1 | Stage-2 | 使用场景 |
|-----------|---------|---------|---------|---------|
| E10_0/1 | EL1&0 | TTBR0/1_EL1 | VTTBR_EL2 | Guest EL0/EL1 |
| E20_0/2 | EL2&0 | TTBR0/1_EL2 | 无 | VHE Host |
| E2 | EL2 | TTBR0_EL2 | 无 | 非 VHE Hypervisor |
| Stage2 | Stage-2 only | — | VTTBR_EL2 | 页表遍历二级 |

---

## 13. Stage-2 翻译概述

### 13.1 地址翻译层次

```
Guest 虚拟地址 (VA)
    │  ← Stage-1: SCTLR_EL1, TTBR0/1_EL1, TCR_EL1
    ▼
中间物理地址 (IPA)
    │  ← Stage-2: HCR_EL2.VM, VTTBR_EL2, VTCR_EL2
    ▼
物理地址 (PA)
```

### 13.2 Stage-2 启用条件

```c
// ptw.c:275-280
case ARMMMUIdx_Stage2:
case ARMMMUIdx_Stage2_S:
    hcr_el2 = arm_hcr_el2_eff_secstate(env, space);
    return (hcr_el2 & (HCR_DC | HCR_VM)) == 0;
    // DC=1 或 VM=1 时 Stage-2 启用
```

- `HCR_VM=1`：显式启用 Stage-2
- `HCR_DC=1`：Default Cacheability，隐式启用 Stage-2（同时禁用 Stage-1）

---

## 14. VTTBR_EL2 与 VTCR_EL2

### 14.1 VTTBR_EL2

```c
// helper.c:4167-4171
{ .name = "VTTBR_EL2", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 4, .crn = 2, .crm = 1, .opc2 = 0,
  .access = PL2_RW, .writefn = vttbr_write, .raw_writefn = raw_write,
  .nv2_redirect_offset = 0x20,
  .fieldoffset = offsetof(CPUARMState, cp15.vttbr_el2) },
```

写入副作用（`helper.c:2801-2815`）：
```c
static void vttbr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                        uint64_t value) {
    // VMID 变化（bit[63:48]）时刷新 Stage-2 + EL1&0 TLB
    if (extract64(raw_read(env, ri) ^ value, 48, 16) != 0) {
        tlb_flush_by_mmuidx(cs, alle1_tlbmask(env));
    }
    raw_write(env, ri, value);
}
```

**VTTBR_EL2 格式**：
```
[63:48] VMID  — 虚拟机标识符
[47:1]  BADDR — Stage-2 页表基地址
[0]     CnP   — Common not Private
```

### 14.2 VTCR_EL2

```c
// helper.c:4155-4160
{ .name = "VTCR_EL2", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 4, .crn = 2, .crm = 1, .opc2 = 2,
  .access = PL2_RW,
  .nv2_redirect_offset = 0x40,
  .fieldoffset = offsetof(CPUARMState, cp15.vtcr_el2) },
```

**VTCR_EL2 关键字段**：
- `T0SZ [5:0]`：IPA 地址宽度 = 64 - T0SZ
- `SL0 [7:6]`：起始翻译级别
- `SL2 [33]`（DS=1 + 4KB 时）：扩展起始级别
- `IRGN0/ORGN0`：Stage-2 页表遍历缓存属性
- `SH0`：共享属性
- `TG0 [15:14]`：页面大小（4KB/16KB/64KB）
- `PS [18:16]`：物理地址大小

---

## 15. Stage-2 页表遍历实现

### 15.1 核心函数 get_phys_addr_lpae()

主要的 LPAE 页表遍历函数位于 `ptw.c:1859-2448`，同时处理 Stage-1 和 Stage-2。对 Stage-2 的区分通过 `regime_is_stage2(mmu_idx)` 判定。

### 15.2 Stage-2 遍历流程

```
get_phys_addr_lpae()
    │
    ├── regime_tcr() → 取 VTCR_EL2
    ├── regime_ttbr() → 取 VTTBR_EL2
    │
    ├── 计算 IPA 宽度（64 - T0SZ）
    ├── check_s2_mmu_setup() → 校验起始级别
    │
    ├── 多级页表遍历循环：
    │   ├── 读取描述符
    │   ├── S2 属性提取（S2AP, XN, MemAttr）
    │   └── 检查 Block/Table 描述符类型
    │
    ├── get_S2prot() → 计算访问权限
    ├── 属性合并（Stage-1 + Stage-2）
    │
    └── 最终物理地址 = 描述符输出地址 + 页内偏移
```

### 15.3 Stage-1 通过 Stage-2 的页表遍历

当 Stage-1 页表遍历时，每次读取页表描述符本身也需要经过 Stage-2 翻译（`ptw.c:695-717`）：

```c
// ptw.c:702-717
if (regime_is_stage2(s2_mmu_idx)) {
    uint64_t hcr = arm_hcr_el2_eff_secstate(env, ptw->cur_space);
    if ((hcr & HCR_PTW) && S2_attrs_are_device(hcr, pte_attrs)) {
        // PTW 位：禁止 Device 内存中的页表
        fi->type = ARMFault_Permission;
        fi->s2addr = addr;
        fi->stage2 = true;
        fi->s1ptw = true;  // 标记为 Stage-1 PTW 触发的 Stage-2 故障
        fi->s1ns = fault_s1ns(ptw->cur_space, s2_mmu_idx);
        return false;
    }
}
```

---

## 16. Stage-2 起始级别校验 — check_s2_mmu_setup()

```c
// ptw.c:1727-1821
static int check_s2_mmu_setup(ARMCPU *cpu, bool is_aa64,
                              uint64_t tcr, bool ds,
                              int iasize, int stride)
{
    int sl0, sl2, startlevel, granulebits, levels;

    sl0 = extract32(tcr, 6, 2);
    if (is_aa64) {
        switch (stride) {
        case 9:  /* 4KB granule */
            sl2 = extract64(tcr, 33, 1);
            if (ds && sl2) {
                startlevel = -1;  // 52-bit IPA
            } else {
                startlevel = 2 - sl0;
            }
            break;
        case 11: /* 16KB granule */
            startlevel = 3 - sl0;
            break;
        case 13: /* 64KB granule */
            startlevel = 3 - sl0;
            break;
        }
    }
    // 校验 IPA 大小与级别的一致性
    levels = 3 - startlevel;
    granulebits = stride + 3;
    s1_min_iasize = levels * stride + granulebits + 1;
    s1_max_iasize = s1_min_iasize + (stride - 1) + 4;

    if (iasize >= s1_min_iasize && iasize <= s1_max_iasize) {
        return startlevel;
    }
    return INT_MIN;  // 配置无效
}
```

---

## 17. Stage-2 访问权限 — get_S2prot()

```c
// ptw.c:1343-1382
static int get_S2prot(CPUARMState *env, int s2ap, int xn, bool s1_is_el0)
{
    int prot = 0;

    // S2AP 权限位
    if (s2ap & 1) prot |= PAGE_READ;
    if (s2ap & 2) prot |= PAGE_WRITE;

    // XN 执行权限（支持 FEAT_XNX）
    if (cpu_isar_feature(any_tts2uxn, env_archcpu(env))) {
        switch (xn) {
        case 0: prot |= PAGE_EXEC; break;
        case 1: if (s1_is_el0)  prot |= PAGE_EXEC; break;
        case 2: break;  // 完全不可执行
        case 3: if (!s1_is_el0) prot |= PAGE_EXEC; break;
        }
    } else {
        if (!extract32(xn, 1, 1)) {
            if (arm_el_is_aa64(env, 2) || prot & PAGE_READ)
                prot |= PAGE_EXEC;
        }
    }
    return prot;
}
```

**S2AP 编码**：
| S2AP | 权限 |
|------|------|
| 00 | 无访问 |
| 01 | 只读 |
| 10 | 只写 |
| 11 | 读写 |

---

## 18. Stage-2 间接权限 — S2PIR_EL2

FEAT_S2PIE 引入了间接权限模型（`ptw.c:1384-1420`）：

```c
static int get_S2prot_indirect(CPUARMState *env, GetPhysAddrResult *result,
                               int pi_index, int po_index, bool s1_is_el0)
{
    static const uint8_t perm_table[16][3] = {
        /* [index] = { priv, unpriv, ttw } */
        /* 0 */ { 0, 0, 0 },                        // 无访问
        /* 8 */ { PAGE_READ, PAGE_READ, PAGE_READ }, // 只读
        /* C */ { PAGE_READ|PAGE_WRITE, ... },       // 读写
        /* F */ { PAGE_READ|PAGE_WRITE|PAGE_EXEC, ...}, // 全权限
        // ... 16 个 overlay 组合
    };

    uint64_t pir = (env->cp15.scr_el3 & SCR_PIEN
                    ? env->cp15.s2pir_el2 : 0);
    int s2pi = extract64(pir, pi_index * 4, 4);

    result->f.prot = perm_table[s2pi][2];  // TTW 权限
    return perm_table[s2pi][s1_is_el0];    // 数据权限
}
```

---

## 19. Stage-2 缓存属性与 FWB

### 19.1 S2_attrs_are_device()

```c
// ptw.c:584-600
static bool S2_attrs_are_device(uint64_t hcr, uint8_t attrs)
{
    // Stage-2 描述符中的内存类型判定
    if (hcr & HCR_FWB) {
        // FWB=1：bit[4]（attrs bit[2]）= 0 → Device
        return (attrs & 0x4) == 0;
    } else {
        // FWB=0：bits[5:4]（attrs bits[3:2]）= 00 → Device
        return (attrs & 0xc) == 0;
    }
}
```

### 19.2 FWB（Forced Write-Back）影响

当 `HCR_EL2.FWB=1` 时，Stage-2 描述符的内存属性字段被重新解释：

| 描述符 bits[5:4] | FWB=0 含义 | FWB=1 含义 |
|-----------------|-----------|-----------|
| 00 | Device-nGnRnE | Device (由 bit[3:2] 决定) |
| 01 | 由 MAIR 查找 | Normal Non-cacheable |
| 10 | 由 MAIR 查找 | Normal Write-Through |
| 11 | 由 MAIR 查找 | Normal Write-Back |

FWB 简化了 Stage-2 属性管理，不再需要查找 MAIR 寄存器。

---

## 20. Stage-2 TLB 管理

### 20.1 Stage-2 TLB 无效化操作

```c
// tlb-insns.c:167-183（IPAS2E1 类操作）
// 按 IPA 无效化 Stage-2 TLB
tlb_flush_page_by_mmuidx(..., ARMMMUIdxBit_Stage2);

// tlb-insns.c:202-217（ALLE2 类操作）
// 无效化所有 EL2 TLB
tlb_flush_by_mmuidx(..., ARMMMUIdxBit_E2 | ARMMMUIdxBit_E2_GCS);
```

### 20.2 VTTBR_EL2 写入的 TLB 刷新

VMID 变化时刷新所有受影响的 TLB：
```c
// helper.c:2811-2813
if (extract64(raw_read(env, ri) ^ value, 48, 16) != 0) {
    tlb_flush_by_mmuidx(cs, alle1_tlbmask(env));
}
```

`alle1_tlbmask()` 返回 EL1&0 + Stage-2 的 MMU index 位掩码。

### 20.3 HCR_EL2 变化的 TLB 刷新

```c
// helper.c:3759-3762
// VM/PTW/DC/DCT/FWB/NV/NV1 任一变化 → 全量 TLB 刷新
if ((env->cp15.hcr_el2 ^ value) &
    (HCR_VM | HCR_PTW | HCR_DC | HCR_DCT | HCR_FWB | HCR_NV | HCR_NV1)) {
    tlb_flush(CPU(cpu));
}
```

---

## 21. Stage-2 故障处理

### 21.1 故障信息结构

Stage-2 故障通过 `ARMMMUFaultInfo` 传递：
- `fi->stage2 = true`：标记为 Stage-2 故障
- `fi->s1ptw`：Stage-1 页表遍历期间的 Stage-2 故障
- `fi->s1ns`：Stage-1 在 NS 空间
- `fi->s2addr`：触发故障的 IPA 地址

### 21.2 故障递送 — arm_deliver_fault()

```c
// tlb_helper.c:239-272
if (fi->stage2) {
    target_el = 2;  // Stage-2 故障始终路由到 EL2
    env->cp15.hpfar_el2 = extract64(fi->s2addr, 12, 47) << 4;
    if (arm_is_secure_below_el3(env) && fi->s1ns) {
        env->cp15.hpfar_el2 |= HPFAR_NS;
    }
}

// 构建 ESR 异常综合征
fsr = compute_fsr_fsc(env, fi, target_el, mmu_idx, &fsc);
if (access_type == MMU_INST_FETCH) {
    syn = syn_insn_abort(same_el, fi->ea, fi->s1ptw, fsc);
    exc = EXCP_PREFETCH_ABORT;
} else {
    syn = merge_syn_data_abort(...);
    exc = EXCP_DATA_ABORT;
}

raise_exception(env, exc, syn, target_el);
```

### 21.3 HPFAR_EL2 格式

```
HPFAR_EL2:
  [47:4]  FIPA — 故障 IPA[47:12]（页对齐）
  [0]     NS   — 非安全标志（ARMv8.4-SecEL2）
```

---

## 22. HCR_EL2 对 Stage-2 的影响

| 位 | 作用 | Stage-2 影响 |
|----|------|-------------|
| VM (bit 0) | Stage-2 使能 | 启用 IPA→PA 翻译 |
| PTW (bit 2) | Protected Table Walk | 禁止 Device 内存中的页表 |
| DC (bit 12) | Default Cacheability | 隐式启用 Stage-2，禁用 Stage-1 |
| FWB (bit 46) | Forced Write-Back | 改变 Stage-2 描述符属性解释 |
| DCT (bit 57) | Default Cacheability Tagging | DC 模式下的标签控制 |

`regime_translation_disabled()` 中的判定：
```c
// ptw.c:275-280
case ARMMMUIdx_Stage2:
case ARMMMUIdx_Stage2_S:
    hcr_el2 = arm_hcr_el2_eff_secstate(env, space);
    return (hcr_el2 & (HCR_DC | HCR_VM)) == 0;
```

> 当 `DC=1` 时，Stage-1 被禁用（所有 EL0/EL1 地址不经 Stage-1 翻译），但 Stage-2 仍然有效，Guest 的所有访问以 Normal Write-Back 缓存属性通过 Stage-2。

---

## 23. 异常路由到 EL2

### 23.1 路由判定条件

异常是否路由到 EL2 取决于 HCR_EL2 的有效值：

| 异常类型 | 路由条件 | HCR 位 |
|---------|---------|--------|
| 物理 IRQ | `arm_hcr_el2_eff() & HCR_IMO` | IMO |
| 物理 FIQ | `arm_hcr_el2_eff() & HCR_FMO` | FMO |
| SError | `arm_hcr_el2_eff() & HCR_AMO` | AMO |
| SMC | `arm_hcr_el2_eff() & HCR_TSC` | TSC |
| WFI | `arm_hcr_el2_eff() & HCR_TWI` | TWI |
| WFE | `arm_hcr_el2_eff() & HCR_TWE` | TWE |
| Stage-2 fault | 始终路由到 EL2 | — |

### 23.2 TGE 对路由的影响

当 `TGE=1` 时：
- 非 VHE（E2H=0）：`FMO/IMO/AMO` 被强制为 1，所有中断路由到 EL2
- VHE（E2H=1）：`FMO/IMO/AMO` 被清除，EL0 在 Host 上下文中，中断按正常路径处理

---

## 24. EL2 异常入口 — arm_cpu_do_interrupt_aarch64()

```c
// helper.c:9198-9370（精选）
static void arm_cpu_do_interrupt_aarch64(CPUState *cs)
{
    unsigned int new_el = env->exception.target_el;
    vaddr addr = env->cp15.vbar_el[new_el];  // VBAR_EL2

    // 异常向量偏移
    if (cur_el < new_el) {
        // 来自低 EL
        switch (new_el) {
        case 2:
            hcr = arm_hcr_el2_eff(env);
            if ((hcr & (HCR_E2H | HCR_TGE)) != (HCR_E2H | HCR_TGE)) {
                is_aa64 = (hcr & HCR_RW) != 0;
            }
            break;
        }
        addr += is_aa64 ? 0x400 : 0x600;
    } else {
        // 同 EL 异常
        if (pstate_read(env) & PSTATE_SP)
            addr += 0x200;
    }

    // ESR_EL2 写入
    env->cp15.esr_el[new_el] = env->exception.syndrome;

    // VSerror 特殊处理
    case EXCP_VSERR:
        addr += 0x180;
        env->exception.syndrome = syn_serror(env->cp15.vsesr_el2 & 0x1ffffff);
        break;

    // SPSR 保存与 NV 特殊处理
    if (cur_el == 1 && new_el == 1) {
        uint64_t hcr = arm_hcr_el2_eff(env);
        if ((hcr & (HCR_NV | HCR_NV1 | HCR_NV2)) == HCR_NV ||
            (hcr & (HCR_NV | HCR_NV2)) == (HCR_NV | HCR_NV2)) {
            // NV: SPSR.M[3:2] 报告为 EL2
            old_mode = deposit64(old_mode, 2, 2, 2);
        }
    }
    env->banked_spsr[aarch64_banked_spsr_index(new_el)] = old_mode;
}
```

### 24.1 异常向量表偏移

```
VBAR_EL2 + 偏移：
  +0x000  同步异常，使用 SP_EL0
  +0x080  IRQ/vIRQ
  +0x100  FIQ/vFIQ
  +0x180  SError/vSError
  +0x200  同步异常，使用 SP_ELx
  +0x280  IRQ/vIRQ
  +0x300  FIQ/vFIQ
  +0x380  SError/vSError
  +0x400  来自低 EL (AArch64)
  +0x480  IRQ
  +0x500  FIQ
  +0x580  SError
  +0x600  来自低 EL (AArch32)
  ...
```

---

## 25. 嵌套虚拟化 — NV/NV2

### 25.1 FEAT_NV 概述

嵌套虚拟化允许 Guest Hypervisor（L1 运行在 EL1）执行 EL2 操作，由 Host Hypervisor（L0 运行在 EL2）通过 trap 或内存重定向来模拟。

### 25.2 HCR_EL2 NV 位

```c
// cpu.h:1736-1739
#define HCR_NV        (1ULL << 42)  // 嵌套虚拟化基本支持
#define HCR_NV1       (1ULL << 43)  // NV 行为修改
#define HCR_NV2       (1ULL << 45)  // 内存重定向优化
```

### 25.3 NV 的 Trap 行为

当 `HCR.NV=1` 且 CPU 在 EL1 时：
- ERET 指令被 trap（`hflags.c:388-389`）
- EL2 系统寄存器访问被 trap（或 NV2 重定向到内存）
- SPSR 中 M[3:2] 报告为 EL2（`helper.c:9348-9359`）

### 25.4 NV2 内存重定向

FEAT_NV2 通过 `VNCR_EL2` 寄存器指定的内存区域替代 trap，Guest Hypervisor 写 EL2 寄存器直接写入内存：

```c
// helper.c:5680-5700（VNCR_EL2 定义）
// 各寄存器的 nv2_redirect_offset 字段：
// HCR_EL2     → 0x78
// VTTBR_EL2   → 0x20
// VTCR_EL2    → 0x40
// CNTVOFF_EL2 → 0x60
// TPIDR_EL2   → 0x90
```

### 25.5 QEMU NV 状态

QEMU 在 TCG 模式下部分支持 FEAT_NV/NV2：
- ID 寄存器暴露：`ID_AA64MMFR1.VH=1`、`ID_AA64MMFR2.NV=2`（`cpu64.c:1309,1326`）
- 寄存器 trap 和 NV2 内存重定向已实现
- 完整的嵌套虚拟化（Guest 中运行 Hypervisor）功能持续完善中

---

## 26. 虚拟定时器

### 26.1 定时器寄存器

```c
// helper.c:4190-4230
{ .name = "CNTHCTL_EL2", ...
  .resetvalue = 3,  // EL1PCTEN=1, EL1PCEN=1
  .writefn = gt_cnthctl_write },

{ .name = "CNTVOFF_EL2", ...
  .writefn = gt_cntvoff_write,
  .nv2_redirect_offset = 0x60 },

{ .name = "CNTHP_CVAL_EL2", ...  // Hypervisor 物理定时器
  .writefn = gt_hyp_cval_write },

{ .name = "CNTHP_CTL_EL2", ...   // Hypervisor 定时器控制
  .writefn = gt_hyp_ctl_write },
```

### 26.2 CNTHCTL_EL2 访问控制

`CNTHCTL_EL2` 控制 EL0/EL1 对定时器的访问：

```c
// helper.c:1157-1188
// gt_counter_access() — 控制 CNTPCT 访问
// gt_timer_access()   — 控制 CNTP_*/CNTV_* 访问
```

- `EL1PCTEN=0`：Trap EL0/EL1 对物理计数器的访问
- `EL1PCEN=0`：Trap EL0/EL1 对物理定时器的访问

### 26.3 CNTVOFF_EL2 — 虚拟计数器偏移

Guest 看到的虚拟计数器 = 物理计数器 - CNTVOFF_EL2。Hypervisor 通过设置此寄存器来为每个 Guest 提供独立的时间基准。

---

## 27. EL2 系统寄存器表

`el2_cp_reginfo[]`（`helper.c:4067-4230`）定义了完整的 EL2 寄存器集：

| 寄存器 | 编码 (Op0,Op1,CRn,CRm,Op2) | NV2 偏移 | 功能 |
|--------|---------------------------|----------|------|
| HCR_EL2 | 3,4,1,1,0 | 0x78 | Hypervisor 配置 |
| HACR_EL2 | 3,4,1,1,7 | — | 辅助控制（CONST 0） |
| ELR_EL2 | 3,4,4,0,1 | NV2_REDIRECT | 异常链接寄存器 |
| ESR_EL2 | 3,4,5,2,0 | NV2_REDIRECT | 异常综合征 |
| FAR_EL2 | 3,4,6,0,0 | NV2_REDIRECT | 故障地址 |
| SPSR_EL2 | 3,4,4,0,0 | NV2_REDIRECT | 保存的程序状态 |
| VBAR_EL2 | 3,4,12,0,0 | — | 向量基地址 |
| SP_EL2 | 3,6,4,1,0 | — | 栈指针（PL3 访问） |
| CPTR_EL2 | 3,4,1,1,2 | — | 协处理器 trap |
| MAIR_EL2 | 3,4,10,2,0 | — | 内存属性 |
| TCR_EL2 | 3,4,2,0,2 | — | 翻译控制 |
| VTCR_EL2 | 3,4,2,1,2 | 0x40 | Stage-2 翻译控制 |
| VTTBR_EL2 | 3,4,2,1,0 | 0x20 | Stage-2 基地址 |
| SCTLR_EL2 | 3,4,1,0,0 | — | 系统控制 |
| TTBR0_EL2 | 3,4,2,0,0 | — | 翻译表基地址 |
| TPIDR_EL2 | 3,4,13,0,2 | 0x90 | 线程 ID |
| CNTHCTL_EL2 | 3,4,14,1,0 | — | 定时器控制 |
| CNTVOFF_EL2 | 3,4,14,0,3 | 0x60 | 虚拟计数器偏移 |

---

## 28. virt 机器 EL2 配置

### 28.1 配置流程

```
qemu-system-aarch64 -machine virt,virtualization=on
                                 │
                    ┌─────────────▼──────────────┐
                    │ virt_set_virt()             │
                    │ vms->virt = true            │
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │ machvirt_init()             │
                    │ if (!vms->virt)             │
                    │   has_el2 = false           │
                    │ else                        │
                    │   has_el2 保持 CPU 默认      │
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │ GIC 配置                    │
                    │ GICv2: 设置                 │
                    │   has-virtualization-ext    │
                    │ GICv3: 默认支持虚拟中断      │
                    └────────────────────────────┘
```

### 28.2 GIC 虚拟化支持

```c
// virt.c:1202-1204
qdev_prop_set_bit(vms->gic, "has-virtualization-extensions",
                  vms->virt);
```

GICv3 在 EL2 模式下额外提供 `ICH_*` 系列寄存器（Virtual Interface Control）用于虚拟中断管理。

---

## 29. KVM 与 EL2

### 29.1 KVM 模式下的分工

| 层面 | QEMU 负责 | KVM 负责 |
|------|----------|---------|
| HCR_EL2 | 通过 ONE_REG 接口设置 | 硬件写入/执行 |
| Stage-2 MMU | 配置 IPA 大小 | 硬件页表管理 |
| TLB | — | 硬件 TLB 管理 |
| 虚拟中断 | 配置 GIC 中断路由 | vGIC 内核实现 |
| 异常 trap | — | 硬件 trap + KVM exit |

### 29.2 IPA 大小配置

```c
// kvm.c:106-123
// 查询 KVM 支持的最大 IPA 大小
KVM_CAP_ARM_VM_IPA_SIZE
// 创建 VM 时指定 IPA 大小
kvm_ioctl(s, KVM_CREATE_VM, ipa_size);
```

### 29.3 内存映射

QEMU 通过 `KVM_SET_USER_MEMORY_REGION` 将 Guest RAM 注册为 KVM memslot，KVM 内核自动管理 Stage-2 页表：

```
Guest RAM (QEMU mmap)
    │
    ▼
KVM_SET_USER_MEMORY_REGION
    │
    ▼
KVM Stage-2 页表（硬件管理）
    │
    ▼
物理内存
```

### 29.4 KVM Exit

- `KVM_EXIT_MEMORY_FAULT`：Stage-2 页表缺页，需要 QEMU 处理内存映射
- `KVM_EXIT_ARM_NISV`：Not Implemented System Register Value，需要 QEMU 模拟

---

## 30. 完整数据流时序图

### 30.1 Guest 内存访问完整路径

```
Guest EL1 执行 LDR X0, [X1]
    │
    ├── 取 VA = X1
    │
    ├── Stage-1 翻译 (SCTLR_EL1.M=1)
    │   ├── TTBR0/1_EL1 + TCR_EL1 → 多级页表遍历
    │   │   ├── 每级描述符读取 → 经 Stage-2 翻译 (HCR.PTW 检查)
    │   │   └── 提取权限 (AP, XN) → get_S1prot()
    │   └── 输出 IPA (中间物理地址)
    │
    ├── Stage-2 翻译 (HCR.VM=1 或 DC=1)
    │   ├── VTTBR_EL2 + VTCR_EL2 → 多级页表遍历
    │   │   ├── check_s2_mmu_setup() 校验
    │   │   └── 提取权限 (S2AP, XN) → get_S2prot()
    │   ├── 属性合并 (Stage-1 + Stage-2)
    │   └── 输出 PA (物理地址)
    │
    ├── 故障？
    │   ├── Stage-1 故障 → 路由到 EL1 (或 EL2 if HCR trap)
    │   └── Stage-2 故障 → 路由到 EL2
    │       ├── HPFAR_EL2 = IPA[47:12] << 4
    │       ├── ESR_EL2 = 异常综合征
    │       └── arm_cpu_do_interrupt_aarch64(target_el=2)
    │
    └── 成功 → PA 访问物理内存
```

### 30.2 Hypervisor 处理 Stage-2 缺页

```
EL2 异常入口 (VBAR_EL2 + 0x400)
    │
    ├── 读取 ESR_EL2 → 解析故障类型
    ├── 读取 HPFAR_EL2 → 获取 IPA
    ├── 读取 FAR_EL2 → 获取 VA (可选)
    │
    ├── Hypervisor 决策：
    │   ├── 建立映射 → 写入 Stage-2 页表
    │   ├── 设备 MMIO → 模拟设备访问
    │   └── 内存保护 → 注入故障给 Guest
    │
    └── ERET → 返回 Guest EL1
```

### 30.3 VHE 模式下的 Host 切换

```
Guest EL0/EL1 执行
    │
    ├── 异常发生 (IRQ / 同步 / ...)
    │   └── HCR.IMO/FMO = 1 → trap 到 EL2
    │
    ├── EL2 Hypervisor 入口
    │   ├── 保存 Guest SPSR/ELR/SP
    │   ├── HCR_EL2.TGE = 1  (切换到 Host 上下文)
    │   │   └── arm_hcr_el2_eff() 清除 trap/Stage-2 位
    │   └── 跳转到 Host 内核
    │
    ├── Host 内核处理 (EL2, VHE)
    │   ├── 使用 SCTLR_EL2/TTBR0_EL2/TCR_EL2 (通过 E2H 重定向)
    │   ├── 仅 Stage-1 翻译（无 Stage-2）
    │   └── 处理完成
    │
    ├── 返回 Guest
    │   ├── HCR_EL2.TGE = 0
    │   ├── 恢复 Guest SPSR/ELR/SP
    │   └── ERET → Guest EL1
    │
    └── Guest 继续执行
```

---

## 附录 A. 关键源文件索引

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `cpu.h` | 1695-1754 | HCR_EL2 位定义 |
| `cpu.h` | 2228-2239 | `arm_is_el2_enabled()` |
| `helper.c` | 333-357 | `access_tvm_trvm()` 等 trap 函数 |
| `helper.c` | 2801-2815 | `vttbr_write()` |
| `helper.c` | 3640-3784 | `do_hcr_write()` |
| `helper.c` | 3818-3889 | `arm_hcr_el2_eff_secstate()` |
| `helper.c` | 3904-3927 | `el_is_in_host()` |
| `helper.c` | 4067-4230 | `el2_cp_reginfo[]` 寄存器表 |
| `helper.c` | 7651-7738 | `add_cpreg_to_hashtable_aa64()` VHE 重定向 |
| `helper.c` | 9198-9370 | `arm_cpu_do_interrupt_aarch64()` |
| `ptw.c` | 270-290 | `regime_translation_disabled()` Stage-2 |
| `ptw.c` | 584-600 | `S2_attrs_are_device()` |
| `ptw.c` | 695-717 | Stage-1 PTW 经 Stage-2 检查 |
| `ptw.c` | 1343-1420 | `get_S2prot()` / `get_S2prot_indirect()` |
| `ptw.c` | 1727-1821 | `check_s2_mmu_setup()` |
| `ptw.c` | 1859-2448 | `get_phys_addr_lpae()` |
| `cpu-irq.c` | 15-80 | `arm_excp_unmasked()` 中断路由 |
| `hflags.c` | 240-400 | `rebuild_hflags_a64()` TB flags |
| `mmuidx.h` | 137-198 | `ARMMMUIdx` 枚举 |
| `tlb_helper.c` | 239-272 | Stage-2 故障递送 |
| `tlb-insns.c` | 167-217 | Stage-2 TLBI 操作 |
| `virt.c` | 2764-2765 | CPU has_el2 设置 |
| `virt.c` | 3882-3883 | virtualization 属性定义 |

---

## 附录 B. HCR_EL2 位域完整映射表

| 位 | 名称 | 功能分类 | 说明 |
|-----|------|---------|------|
| 0 | VM | 内存虚拟化 | Stage-2 使能 |
| 1 | SWIO | 内存虚拟化 | Set/Way 无效化拦截 |
| 2 | PTW | 内存虚拟化 | Protected Table Walk |
| 3 | FMO | 中断路由 | FIQ 路由到 EL2 |
| 4 | IMO | 中断路由 | IRQ 路由到 EL2 |
| 5 | AMO | 中断路由 | SError 路由到 EL2 |
| 6 | VF | 虚拟中断 | 虚拟 FIQ 挂起 |
| 7 | VI | 虚拟中断 | 虚拟 IRQ 挂起 |
| 8 | VSE | 虚拟中断 | 虚拟 SError 挂起 |
| 9 | FB | 广播控制 | 强制广播 |
| 10-11 | BSU | 广播控制 | 屏障共享性升级 |
| 12 | DC | 内存虚拟化 | 默认可缓存性 |
| 13 | TWI | 指令 Trap | Trap WFI |
| 14 | TWE | 指令 Trap | Trap WFE |
| 15-18 | TID0-3 | ID Trap | Trap ID 寄存器 |
| 19 | TSC | 指令 Trap | Trap SMC |
| 20 | TIDCP | 指令 Trap | Trap 实现定义功能 |
| 21 | TACR | 寄存器 Trap | Trap ACTLR |
| 22 | TSW | 缓存 Trap | Trap Set/Way 缓存操作 |
| 23 | TPCP | 缓存 Trap | Trap PoC/PoP 缓存操作 |
| 24 | TPU | 缓存 Trap | Trap PoU 缓存操作 |
| 25 | TTLB | TLB Trap | Trap TLBI |
| 26 | TVM | 寄存器 Trap | Trap 虚存控制写 |
| 27 | TGE | VHE 控制 | 通用异常 trap |
| 28 | TDZ | 指令 Trap | Trap DC ZVA |
| 29 | HCD | 指令 Trap | HVC disable |
| 30 | TRVM | 寄存器 Trap | Trap 虚存控制读 |
| 31 | RW | 执行模式 | EL1 AArch64/AArch32 |
| 32 | CD | 内存虚拟化 | Stage-2 D-cache 禁止 |
| 33 | ID | 内存虚拟化 | Stage-2 I-cache 禁止 |
| 34 | E2H | VHE 控制 | EL2 Host Extensions |
| 35 | TLOR | 寄存器 Trap | Trap LOR 寄存器 |
| 36 | TERR | 寄存器 Trap | Trap 错误记录寄存器 |
| 37 | TEA | 异常路由 | Trap 外部中止到 EL2 |
| 38 | MIOCNCE | 内存序 | 非对齐内存序 |
| 39 | TME | 内存事务 | Trap TME |
| 40 | APK | PAuth | PAuth key 访问 |
| 41 | API | PAuth | PAuth 指令控制 |
| 42 | NV | 嵌套虚拟化 | 基本嵌套虚拟化 |
| 43 | NV1 | 嵌套虚拟化 | NV 行为修改 |
| 44 | AT | 嵌套虚拟化 | Trap AT 指令 |
| 45 | NV2 | 嵌套虚拟化 | 内存重定向 |
| 46 | FWB | 内存虚拟化 | Forced Write-Back |
| 47 | FIEN | 异常路由 | 故障注入使能 |
| 48 | GPF | RME | GPF 路由到 EL2 |
| 49 | TID4 | ID Trap | Trap ID 空间 4 |
| 50 | TICAB | 缓存 Trap | Trap IC IALLUIS |
| 51 | AMVOFFEN | 虚拟偏移 | AMU 虚拟偏移使能 |
| 52 | TOCU | 缓存 Trap | Trap PoU 到 PoC 缓存操作 |
| 53 | ENSCXT | 上下文 | SCXTNUM 使能 |
| 54 | TTLBIS | TLB Trap | Trap TLBI IS |
| 55 | TTLBOS | TLB Trap | Trap TLBI OS |
| 56 | ATA | MTE | Allocation Tag 访问 |
| 57 | DCT | 内存虚拟化 | DC 模式标签 |
| 58 | TID5 | ID Trap | Trap GMID_EL1 |
| 59 | TWEDEN | 指令 Trap | WFE delay 使能 |
| 60-63 | TWEDEL | 指令 Trap | WFE delay 值 |

---

## 附录 C. 关键 Git 提交历史

Stage-2 和 VHE 相关的 QEMU 关键提交可通过以下命令查看：

```bash
# HCR_EL2 写入逻辑演进
git log --oneline -20 -- target/arm/helper.c | grep -i 'hcr\|vhe\|el2'

# Stage-2 页表遍历演进
git log --oneline -20 -- target/arm/ptw.c | grep -i 'stage.2\|s2\|vtcr'

# 嵌套虚拟化
git log --oneline -20 -- target/arm/helper.c | grep -i 'nv2\|nested\|vncr'

# VHE 重定向
git log --oneline -20 -- target/arm/helper.c | grep -i 'vhe\|redir\|e2h'
```

---

> **文档版本**: v1.0  
> **最后更新**: 2025-07  
> **相关文档**:  
> - [06-异常级别状态管理深度分析.md](06-异常级别状态管理深度分析.md) — EL 切换基础  
> - [07-不同EL下指令执行差异深度分析.md](07-不同EL下指令执行差异深度分析.md) — 指令 trap 机制  
> - [08-TrustZone安全扩展与Secure-World深度分析.md](08-TrustZone安全扩展与Secure-World深度分析.md) — 安全世界  
> - [../memory/00-内存子系统深度分析.md](../memory/00-内存子系统深度分析.md) — 内存模型基础  
> - [04-中断注入与处理深度分析.md](04-中断注入与处理深度分析.md) — 中断路径
