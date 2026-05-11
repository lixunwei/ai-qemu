# ARM64 GCS/RME 及新扩展深度分析

> QEMU 11.0.50 源码分析 — 守护控制栈、领域管理扩展及 ARMv9 新特性
>
> 关联文档：[14-EL状态管理与指令执行变化深度分析](14-EL状态管理与指令执行变化深度分析.md) | [16-PAC-BTI-MTE安全特性深度分析](16-PAC-BTI-MTE安全特性深度分析.md)

---

## 目录

1. [概述](#1-概述)
2. [GCS 守护控制栈](#2-gcs-守护控制栈)
3. [GCS 状态存储与使能控制](#3-gcs-状态存储与使能控制)
4. [GCS BL/RET 推入与弹出](#4-gcs-blret-推入与弹出)
5. [GCS 指令集](#5-gcs-指令集)
6. [GCS 异常处理与 EXLOCK](#6-gcs-异常处理与-exlock)
7. [GCS MMU 索引与内存访问](#7-gcs-mmu-索引与内存访问)
8. [RME 领域管理扩展](#8-rme-领域管理扩展)
9. [RME 四安全状态模型](#9-rme-四安全状态模型)
10. [GPC 粒度保护检查](#10-gpc-粒度保护检查)
11. [RME MMU 索引与页表遍历](#11-rme-mmu-索引与页表遍历)
12. [RME 系统寄存器与异常路由](#12-rme-系统寄存器与异常路由)
13. [FEAT_NMI 不可屏蔽中断](#13-feat_nmi-不可屏蔽中断)
14. [FEAT_S1PIE/S1POE 权限间接与覆盖](#14-feat_s1pies1poe-权限间接与覆盖)
15. [FEAT_HAFDBS 硬件访问/脏位](#15-feat_hafdbs-硬件访问脏位)
16. [FEAT_AIE 属性间接扩展](#16-feat_aie-属性间接扩展)
17. [其他新特性概览](#17-其他新特性概览)
18. [TB 标志集成总览](#18-tb-标志集成总览)
19. [总结](#19-总结)

---

## 1. 概述

ARMv9 架构在安全性和资源管理方面引入了大量新扩展。本文档分析 QEMU 中以下特性的实现：

| 特性 | ARM 版本 | 核心功能 | QEMU 实现状态 |
|------|---------|---------|--------------|
| **GCS** (Guarded Control Stack) | v9.4 | 影子返回地址栈 | ✅ 完整实现 |
| **RME** (Realm Management Extension) | v9.2 | 四安全域隔离 | ✅ 完整实现 |
| **NMI** (Non-Maskable Interrupts) | v8.8 | 不可屏蔽中断 | ✅ 完整实现 |
| **S1PIE/S1POE** (Permission Indirection/Overlay) | v8.9 | 权限间接/覆盖 | ✅ 完整实现 |
| **HAFDBS** (HW Access Flag/Dirty Bit) | v8.1 | 硬件访问/脏位管理 | ✅ 完整实现 |
| **AIE** (Attribute Indirection) | v8.9 | 属性间接 | ✅ 寄存器实现 |
| **MPAM** (Memory Partitioning) | v8.4 | 内存分区与监控 | ⚠️ 仅存根 |

---

## 2. GCS 守护控制栈

### 2.1 基本原理

GCS（Guarded Control Stack）是 ARMv9.4 引入的硬件影子栈机制。每次函数调用（BL/BLR）时，CPU 自动将返回地址推入一个独立的、受写保护的影子栈；函数返回（RET）时，CPU 将影子栈中的地址与 LR 进行比较，不匹配则产生异常。

GCS 与 PAC 互补：
- **PAC**：签名嵌入指针内部，防止返回地址被篡改
- **GCS**：维护独立的返回地址栈，即使主栈被完全覆盖也能检测

### 2.2 关键概念

- **GCSPR_ELx**：GCS 栈指针，指向当前影子栈顶
- **GCSCR_ELx**：GCS 控制寄存器，控制使能和行为
- **PSTATE.EXLOCK**：异常锁定位，防止异常返回记录被利用
- **GCS 记录**：8 字节返回地址，存储在 GCS 内存区域

---

## 3. GCS 状态存储与使能控制

### 3.1 CPU 状态存储

```c
// cpu.h:586-587
uint64_t gcscr_el[4];   // GCSCRE0_EL1, GCSCR_EL[123]
uint64_t gcspr_el[4];   // GCSPR_EL[0123]
```

每个 EL 有独立的 GCS 控制寄存器和栈指针。`gcscr_el[0]` 对应 `GCSCRE0_EL1`（EL0 的 GCS 控制），`gcspr_el[0]` 对应 `GCSPR_EL0`。

系统寄存器定义在 `cpregs-gcs.c:83-123`，直接映射到 CPUARMState 字段。

### 3.2 PSTATE.EXLOCK

```c
// cpu.h:1570
#define PSTATE_EXLOCK (1ULL << 34)
```

EXLOCK 是 64 位 PSTATE 中的高位标志（bit 34），用于控制异常入口/返回时的 GCS 行为。

### 3.3 GCS 使能层级

```c
// hflags.c:452-471
if (cpu_isar_feature(aa64_gcs, env_archcpu(env))) {
    if (env->cp15.gcscr_el[el] & GCSCR_PCRSEL) {
        switch (el) {
        default:  // EL0/EL1
            if (!el_is_in_host(env, el)
                && !(arm_hcrx_el2_eff(env) & HCRX_GCSEN)) {
                break;  // EL2 未授权
            }
            /* fall through */
        case 2:
            if (arm_feature(env, ARM_FEATURE_EL3)
                && !(env->cp15.scr_el3 & SCR_GCSEN)) {
                break;  // EL3 未授权
            }
            /* fall through */
        case 3:
            DP_TBFLAG_A64(flags, GCS_EN, 1);
            break;
        }
    }
}
```

GCS 使能是多层级控制：

| 条件 | 控制源 | 说明 |
|------|-------|------|
| `GCSCR_ELx.PCRSEL` | 本级 | GCS 记录推入使能 |
| `HCRX_EL2.GCSEN` | EL2 | EL2 授权 EL0/EL1 使用 GCS |
| `SCR_EL3.GCSEN` | EL3 | EL3 授权低级 EL 使用 GCS |

EL3 自身的 GCS 不受外部控制（最高特权级），直接由 `GCSCR_EL3.PCRSEL` 决定。

### 3.4 返回值检查使能

```c
// hflags.c:474-477
if (env->cp15.gcscr_el[el] & GCSCR_RVCHKEN) {
    DP_TBFLAG_A64(flags, GCS_RVCEN, 1);
}
```

`GCSCR.RVCHKEN` 控制是否在 RET 时执行返回值检查（比较 GCS 记录与实际返回地址）。

### 3.5 GCSSTR 使能

```c
// hflags.c:479-487
if (!(env->cp15.gcscr_el[el] & GCSCR_STREN)) {
    DP_TBFLAG_A64(flags, GCSSTR_EL, el ? el : 1);
} else if (el == 1 && EX_TBFLAG_ANY(flags, FGT_ACTIVE)
           && !FIELD_EX64(env->cp15.fgt_exec[FGTREG_HFGITR],
                           HFGITR_EL2, NGCSSTR_EL1)) {
    DP_TBFLAG_A64(flags, GCSSTR_EL, 2);
}
```

`GCSSTR` 指令（GCS Store）受独立的使能/陷入控制，支持 FGT 细粒度陷入。

---

## 4. GCS BL/RET 推入与弹出

### 4.1 推入：gen_add_gcs_record()

```c
// translate-a64.c:439-449
static void gen_add_gcs_record(DisasContext *s, TCGv_i64 value)
{
    TCGv_i64 addr = tcg_temp_new_i64();
    TCGv_i64 gcspr = cpu_gcspr[s->current_el];
    int mmuidx = core_gcs_mem_index(s->mmu_idx);
    MemOp mop = finalize_memop(s, MO_64 | MO_ALIGN);

    tcg_gen_addi_i64(addr, gcspr, -8);           // GCSPR -= 8
    tcg_gen_qemu_st_i64(value, clean_data_tbi(s, addr),
                        mmuidx, mop);              // 存储返回地址
    tcg_gen_mov_i64(gcspr, addr);                  // 更新 GCSPR
}
```

推入流程（向下增长栈）：
1. `GCSPR -= 8`（预减）
2. 将返回地址写入新的 GCSPR 指向的位置
3. 使用 GCS 专用 MMU 索引进行内存访问

### 4.2 弹出：gen_load_check_gcs_record()

```c
// translate-a64.c:451-470
static void gen_load_check_gcs_record(DisasContext *s, TCGv_i64 target,
                                       GCSInstructionType it, int rt)
{
    TCGv_i64 gcspr = cpu_gcspr[s->current_el];
    int mmuidx = core_gcs_mem_index(s->mmu_idx);
    MemOp mop = finalize_memop(s, MO_64 | MO_ALIGN);
    TCGv_i64 rec_va = tcg_temp_new_i64();

    tcg_gen_qemu_ld_i64(rec_va, clean_data_tbi(s, gcspr),
                        mmuidx, mop);              // 从 GCSPR 加载记录

    if (s->gcs_rvcen) {
        TCGLabel *fail_label =
            delay_exception(s, EXCP_UDEF, syn_gcs_data_check(it, rt));
        tcg_gen_brcond_i64(TCG_COND_NE, rec_va, target,
                           fail_label);            // 比较记录与目标
    }

    gen_a64_set_pc(s, rec_va);                     // 跳转到记录地址
    tcg_gen_addi_i64(gcspr, gcspr, 8);             // GCSPR += 8
}
```

弹出流程：
1. 从 GCSPR 加载 GCS 记录
2. 如果 `RVCHKEN` 使能，比较记录与 LR（target）→ 不匹配则 GCS Data Check 异常
3. 将 PC 设置为记录值（而非 LR，确保即使 LR 被篡改也使用影子栈地址）
4. `GCSPR += 8`（后增）

### 4.3 BL/BLR 中的 GCS 集成

```c
// translate-a64.c:1704-1716 (trans_BL)
if (s->gcs_en) {
    gen_add_gcs_record(s, link);  // 推入返回地址
}

// translate-a64.c:1813-1815 (trans_BLR)
if (s->gcs_en) {
    gen_add_gcs_record(s, link);
}

// translate-a64.c:1828-1833 (trans_RET)
if (s->gcs_en) {
    gen_load_check_gcs_record(s, target, GCS_IT_RET_nPauth, a->rn);
}
```

所有带链接的分支（BL/BLR/BLRA/BLRAZ）在 `gcs_en` 时推入，所有返回（RET/RETA）在 `gcs_en` 时弹出并检查。

---

## 5. GCS 指令集

### 5.1 指令概览

| 指令 | 功能 | 源码位置 |
|------|------|---------|
| **GCSPUSHM** | 手动推入值到 GCS | translate-a64.c:2568-2585 |
| **GCSPOPM** | 手动从 GCS 弹出值 | translate-a64.c:2663+ |
| **GCSPUSHX** | 推入异常返回记录 | translate-a64.c:2586-2613 |
| **GCSPOPCX** | 弹出异常返回记录 | translate-a64.c:2615-2661 |
| **GCSPOPX** | 弹出异常返回记录（不验证）| translate-a64.c:2663+ |
| **GCSSS1/GCSSS2** | GCS 切换（Seal/Unseal）| translate-a64.c:2700-2742 |
| **GCSSTR** | GCS Store（写入 GCS 内存）| translate-a64.c:4293-4326 |
| **GCSLDR** | GCS Load（读取 GCS 内存）| cpregs-gcs.c:125-149 |

### 5.2 GCSSTR 陷入控制

```c
// translate-a64.c:4311-4325
// GCSSTR 受 GCSCR.STREN 和 FGT 双重控制
// 使用 core_gcs_mem_index(armidx) 进行 GCS 专用内存访问
```

`GCSSTR` 允许软件（如 OS）直接写入 GCS 内存，用于上下文切换时初始化新线程的 GCS。由于这绕过了 GCS 的写保护，受到独立的使能/陷入控制。

---

## 6. GCS 异常处理与 EXLOCK

### 6.1 GCSPUSHX — 异常入口推入

```c
// translate-a64.c:2586-2613
// gen_gcspushx() 实现:
// 1. 将异常返回地址推入 GCS
// 2. 清除 PSTATE.EXLOCK
```

异常入口时，CPU 将异常返回地址推入 GCS，然后清除 EXLOCK。这确保异常处理程序不能直接使用 ERET 返回而不经过 GCS 验证。

### 6.2 GCSPOPCX — 异常返回弹出

```c
// translate-a64.c:2615-2661
// GCSPOPCX 实现:
// 1. 从 GCS 弹出异常返回记录
// 2. 如果 GCSCR.EXLOCKEN 使能，设置 PSTATE.EXLOCK
```

异常返回前，软件执行 GCSPOPCX 弹出异常返回记录并设置 EXLOCK，然后才能执行 ERET。

### 6.3 EXLOCK 状态机

```
异常入口:
  GCSPUSHX → 推入记录 → EXLOCK = 0

异常返回前:
  GCSPOPCX → 弹出记录 → EXLOCK = 1 (if EXLOCKEN)

ERET:
  检查 EXLOCK → 执行返回 → EXLOCK = 0
```

EXLOCK 防止攻击者在不经过 GCSPOPCX 的情况下直接执行 ERET。

---

## 7. GCS MMU 索引与内存访问

### 7.1 GCS 专用 MMU 索引

```c
// translate-a64.c:158-162
// core_gcs_mem_index(s->mmu_idx) 将普通 MMU 索引转换为 GCS 变体
```

GCS 内存访问使用专用的 MMU 索引（`mmuidx.h:137-186`），这些索引对应 GCS 特殊的页表权限：

- GCS 页面标记为只读 + GCS 可写
- 普通 store 指令无法写入 GCS 页面
- 只有 GCS 推入操作（和 GCSSTR）通过 GCS MMU 索引访问时才能写入

MMU 索引映射表在 `mmuidx.c:26-53` 中定义，每个常规 MMU 索引有对应的 GCS 变体。

---

## 8. RME 领域管理扩展

### 8.1 基本原理

RME（Realm Management Extension）是 ARM CCA（Confidential Compute Architecture）的核心硬件机制。RME 将传统的两安全域（Secure/NonSecure）扩展为四安全域：

```
              +------------------+
              |      Root        |  EL3（固件/监控器）
              +--------+---------+
                       |
         +-------------+-------------+
         |             |             |
    +----+----+  +-----+-----+ +----+----+
    | Secure  |  | NonSecure | | Realm   |
    | (TEE)   |  | (REE)     | | (CCA)   |
    +---------+  +-----------+ +---------+
```

- **Root**：EL3，最高权限，管理所有安全域
- **Secure**：传统 TEE（TrustZone）
- **NonSecure**：正常世界（Linux/Hypervisor）
- **Realm**：机密计算领域，受 Realm Management Monitor 管理

---

## 9. RME 四安全状态模型

### 9.1 安全状态枚举

```c
// arm-security.h:18-23
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,   // NS=0, NSE=0
    ARMSS_NonSecure  = 1,   // NS=1, NSE=0
    ARMSS_Root       = 2,   // EL3 with RME
    ARMSS_Realm      = 3,   // NS=1, NSE=1
} ARMSecuritySpace;
```

枚举值的低 2 位对应 GPI（Granule Protection Information）编码，便于 GPC 检查时直接比较。

### 9.2 安全状态判定

```c
// helper.c:10131-10160
ARMSecuritySpace arm_security_space(CPUARMState *env)
{
    // EL3 + AArch64 + RME → Root
    if (is_a64(env)) {
        if (extract32(env->pstate, 2, 2) == 3) {
            if (cpu_isar_feature(aa64_rme, env_archcpu(env))) {
                return ARMSS_Root;     // EL3 with RME = Root
            } else {
                return ARMSS_Secure;   // EL3 without RME = Secure
            }
        }
    }
    return arm_security_space_below_el3(env);
}
```

### 9.3 SCR_EL3.NS/NSE 编码

```c
// helper.c:10163-10187
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env)
{
    if (!(env->cp15.scr_el3 & SCR_NS)) {
        return ARMSS_Secure;      // NS=0 → Secure
    } else if (env->cp15.scr_el3 & SCR_NSE) {
        return ARMSS_Realm;       // NS=1, NSE=1 → Realm
    } else {
        return ARMSS_NonSecure;   // NS=1, NSE=0 → NonSecure
    }
}
```

四安全状态编码矩阵：

| NSE | NS | 安全状态 |
|-----|----|---------|
| 0 | 0 | Secure |
| 0 | 1 | NonSecure |
| 1 | 0 | Reserved（QEMU 忽略 NSE）|
| 1 | 1 | Realm |

### 9.4 安全状态辅助函数

```c
// arm-security.h:26-28
static inline bool arm_space_is_secure(ARMSecuritySpace space)
{
    return space == ARMSS_Secure || space == ARMSS_Root;
}
```

Root 被视为 "secure" 以保持与传统 TrustZone 代码的向后兼容。

---

## 10. GPC 粒度保护检查

### 10.1 GPC 概述

GPC（Granule Protection Check）是 RME 的内存访问控制核心。每次内存访问都经过 GPC 检查，根据物理地址的 GPT（Granule Protection Table）条目判断请求的安全域是否有权访问该粒度。

### 10.2 GPT 遍历实现

```c
// ptw.c:333-571
bool arm_granule_protection_check(ARMGranuleProtectionConfig config,
                                  uint64_t paddress,
                                  ARMSecuritySpace pspace,
                                  ARMSecuritySpace ss,
                                  ARMMMUFaultInfo *fi)
```

GPC 检查流程：

**优先级 1 — 配置验证**：
```c
// ptw.c:364-367
pps = FIELD_EX64(gpccr, GPCCR, PPS);
if (pps > config.parange) goto fault_walk;  // PPS 超出物理地址范围
```

**优先级 2 — 地址空间禁用**：
```c
// ptw.c:408-419
static const uint8_t disable_masks[4] = {
    [ARMSS_Secure]    = R_GPCCR_SPAD_MASK,
    [ARMSS_NonSecure] = R_GPCCR_NSPAD_MASK,
    [ARMSS_Root]      = 0,                    // Root 不可被禁用
    [ARMSS_Realm]     = R_GPCCR_RLPAD_MASK,
};
if (gpccr & disable_masks[pspace]) goto fault_fail;
```

Root 域的访问不受 GPCCR 禁用位影响——这保证了 EL3 固件始终能访问所有内存。

**Level 0 查找**：
```c
// ptw.c:449-474
index = extract64(paddress, l0gptsz, pps - l0gptsz);
tableaddr += index * 8;
entry = address_space_ldq_le(config.gpt_as, tableaddr, attrs, &result);

switch (extract32(entry, 0, 4)) {
case 1: /* block descriptor */
    gpi = extract32(entry, 4, 4);
    goto found;
case 3: /* table descriptor */
    tableaddr = entry & ~0xf;  // 进入 Level 1
    break;
}
```

**Level 1 查找**：
```c
// ptw.c:476-505
level = 1;
index = extract64(paddress, pgs + 4, l0gptsz - pgs - 4);
// contiguous descriptor 或按粒度索引
gpi = extract64(entry, index * 4, 4);
```

**GPI 值解析**：
```c
// ptw.c:507-554
switch (gpi) {
case 0b0000: break;                  // 无访问
case 0b1111: return true;            // 全部可访问
case 0b1001: /* non-secure */
case 0b1010: /* root */
case 0b1011: /* realm */
case 0b1000: /* secure */
    if (pspace == (gpi & 3))          // 匹配安全域
        return true;
    break;
}
```

### 10.3 GPC 故障类型

| 故障类型 | 枚举值 | 触发条件 |
|---------|--------|---------|
| GPCF_Walk | Walk | GPT 条目非法（保留值、RES0 非零）|
| GPCF_Fail | Fail | 安全域不匹配或被禁用 |
| GPCF_EABT | EABT | GPT 内存访问外部中止 |
| GPCF_AddressSize | AddressSize | GPTBR 基地址超出 PPS |

---

## 11. RME MMU 索引与页表遍历

### 11.1 RME 新增 MMU 索引

```c
// mmuidx.h:157-175
ARMMMUIdx_E3          // EL3 翻译
ARMMMUIdx_Phys_Root   // Root 物理地址空间
ARMMMUIdx_Phys_Realm  // Realm 物理地址空间
```

每个物理地址空间有独立的 TLB（`mmuidx.h:83-85`）：Secure、NonSecure、Realm、Root。

### 11.2 Stage-2 PTW 路由

```c
// ptw.c:193-225
// ptw_idx_for_stage_2() 根据 below-EL3 安全状态路由 Stage-2 PTW 加载:
// - Realm → ARMMMUIdx_Phys_Realm
// - Secure → ARMMMUIdx_Phys_S
// - NonSecure → ARMMMUIdx_Phys_NS
```

### 11.3 Root 单阶段翻译

```c
// ptw.c:602-628
// Root 翻译始终是单阶段（无 Stage-2）
// EL3 使用 ARMMMUIdx_E3
```

Root 域（EL3）不存在 Stage-2 翻译——这是架构设计，确保 EL3 固件完全控制其地址空间。

---

## 12. RME 系统寄存器与异常路由

### 12.1 GPC 相关寄存器

```c
// helper.c:4973-4982
{ .name = "GPCCR_EL3", .opc0 = 3, .opc1 = 6, .crn = 2, .crm = 1, .opc2 = 6,
  .access = PL3_RW, .writefn = gpccr_write, .resetfn = gpccr_reset }
{ .name = "GPTBR_EL3", .opc0 = 3, .opc1 = 6, .crn = 2, .crm = 1, .opc2 = 4,
  .access = PL3_RW }
{ .name = "MFAR_EL3",  .opc0 = 3, .opc1 = 6, .crn = 6, .crm = 0, .opc2 = 5,
  .access = PL3_RW }
```

| 寄存器 | 功能 |
|--------|------|
| GPCCR_EL3 | GPC 配置（PPS/PGS/L0GPTSZ/地址空间禁用/GPC 使能）|
| GPTBR_EL3 | GPT 基地址（12 位左移）|
| MFAR_EL3 | GPC 故障地址记录 |

### 12.2 MECID 寄存器

```c
// helper.c:5062-5105
// MECID_P0/A0/P1/A1_EL2, MECID_RL_A_EL3, VMECID_P/A_EL2
// 用于 MEC (Memory Encryption Context) 标识
```

MECID 寄存器支持 Realm 间的内存加密上下文隔离（`helper.c:5025-5041` 中限制仅 Realm EL2 和 EL3 可访问）。

### 12.3 GPC 故障路由

```c
// tlb_helper.c:122-157
// GPC 故障始终路由到 EL3 (target_el = 3)
// 写入 MFAR_EL3 时附加 NS/NSE 位标识故障来源安全域
```

GPC 故障产生 `EXCP_GPC` 异常，总是路由到 EL3（Root），因为只有 EL3 有权修改 GPT。

### 12.4 特性检测

```c
// cpu-features.h:1107-1114
isar_feature_aa64_rme()       // ID_AA64PFR0.RME != 0
isar_feature_aa64_rme_gpc2()  // ID_AA64PFR0.RME >= 2 (GPC2 扩展)
```

---

## 13. FEAT_NMI 不可屏蔽中断

### 13.1 ALLINT 控制

```c
// cpu.h:1559
#define PSTATE_ALLINT (1U << 13)

// helper.c:4994-5003
static void aa64_allint_write(CPUARMState *env, const ARMCPRegInfo *ri,
                              uint64_t value)
{
    env->pstate = (env->pstate & ~PSTATE_ALLINT) | (value & PSTATE_ALLINT);
}
```

`PSTATE.ALLINT` 是 ARMv8.8 引入的"全中断屏蔽位"。当 ALLINT=1 时，即使是 NMI 也被屏蔽。

### 13.2 NMI 中断路由

```c
// cpu-irq.c:34-57
if (cpu_isar_feature(aa64_nmi, env_archcpu(env)) &&
    env->cp15.sctlr_el[target_el] & SCTLR_NMI && cur_el == target_el) {
    allIntMask = env->pstate & PSTATE_ALLINT ||
                 ((env->cp15.sctlr_el[target_el] & SCTLR_SPINTMASK) &&
                  (env->pstate & PSTATE_SP));
}

switch (excp_idx) {
case EXCP_NMI:
    pstate_unmasked = !allIntMask;
    break;
case EXCP_VINMI:  // 虚拟 NMI（需 HCR.IMO）
    return !allIntMask;
case EXCP_VFNMI:  // 虚拟 FIQ NMI（需 HCR.FMO）
    return !allIntMask;
}
```

NMI 屏蔽条件：
- `PSTATE.ALLINT = 1` → NMI 被屏蔽
- `SCTLR.SPINTMASK = 1` 且 `PSTATE.SP = 1`（使用 SP_ELx）→ NMI 被屏蔽

虚拟 NMI（VINMI/VFNMI）仅在虚拟化环境中有效（需 HCR_EL2.IMO/FMO）。

### 13.3 ALLINT 陷入控制

```c
// helper.c:5005-5010
static CPAccessResult aa64_allint_access(CPUARMState *env, ...)
{
    if (!isread && arm_current_el(env) == 1 &&
        (arm_hcrx_el2_eff(env) & HCRX_TALLINT)) {
        return CP_ACCESS_TRAP_EL2;  // EL1 写 ALLINT 陷入 EL2
    }
}
```

EL2 可通过 `HCRX_EL2.TALLINT` 控制 EL1 对 ALLINT 的写访问。

---

## 14. FEAT_S1PIE/S1POE 权限间接与覆盖

### 14.1 基本原理

传统页表权限直接编码在页表描述符的 AP/UXN/PXN 位中。S1PIE/S1POE 引入间接层：

- **S1PIE**（Permission Indirection Extension）：描述符中的 4-bit 索引查找 `PIR_EL1/PIRE0_EL1` 寄存器获取实际权限
- **S1POE**（Permission Overlay Extension）：在 S1PIE 之上叠加覆盖层（`POR_ELx`）

### 14.2 PIR 寄存器

```c
// helper.c:6120-6147
{ .name = "PIR_EL1", .fieldoffset = offsetof(CPUARMState, cp15.pir_el[1]) }
{ .name = "PIR_EL2", .fieldoffset = offsetof(CPUARMState, cp15.pir_el[2]) }
{ .name = "PIR_EL3", .fieldoffset = offsetof(CPUARMState, cp15.pir_el[3]) }
{ .name = "PIRE0_EL1", .fieldoffset = offsetof(CPUARMState, cp15.pir_el[0]) }
{ .name = "PIRE0_EL2", .fieldoffset = offsetof(CPUARMState, cp15.pire0_el2) }
```

每个 EL 有独立的 PIR（Permission Indirection Register）：
- `PIR_EL1`：EL1 权限索引表
- `PIRE0_EL1`：EL0 权限索引表
- 64-bit 寄存器，包含 16 个 4-bit 权限条目

### 14.3 S2PIR — Stage-2 权限间接

```c
// helper.c:6149-6154
{ .name = "S2PIR_EL2", .fieldoffset = offsetof(CPUARMState, cp15.s2pir_el2) }
```

Stage-2 也支持权限间接。

### 14.4 PTW 集成

```c
// ptw.c:2384-2395
// S1PIE/S2PIE 在页表遍历中处理脏位语义
// internals.h:770 使用 dirtybit 字段
```

权限间接与 HAFDBS 脏位管理集成，在页表遍历中统一处理。

---

## 15. FEAT_HAFDBS 硬件访问/脏位

### 15.1 基本原理

传统 ARM 页表的 AF（Access Flag）和 DBM（Dirty Bit Modifier）需要软件管理。HAFDBS 允许硬件自动设置这些位。

### 15.2 PTW 实现

**访问标志（AF）**：
```c
// ptw.c:2143-2163
if (!(descriptor & (1 << 10)) && !param.ha) {
    fi->type = ARMFault_AccessFlag;     // 无 HA: 产生 AF 故障
    goto do_fault;
}
if (!(descriptor & (1 << 10)) && param.ha) {
    new_descriptor |= 1 << 10;          // 有 HA: 硬件自动设置 AF
}
```

- `param.ha = 0`：AF=0 时产生 Access Flag 故障（软件处理）
- `param.ha = 1`：AF=0 时硬件自动设置 AF=1（无故障）

**脏位（DBM）**：
```c
// ptw.c:2171-2179
if (param.hd
    && extract64(descriptor, 51, 1)   // DBM 位
    && access_type == MMU_DATA_STORE) {
    if (regime_is_stage2(mmu_idx)) {
        new_descriptor |= 1ull << 7;    // S2AP[1] = 1
    } else {
        new_descriptor &= ~(1ull << 7); // AP[2] = 0 (可写)
    }
}
```

- `param.hd = 1` 且 `DBM = 1`：写访问时自动更新 AP/S2AP 位
- Stage-1：清除 AP[2]（使页面可写）
- Stage-2：设置 S2AP[1]（标记脏页）

### 15.3 PTE 回写

```c
// ptw.c:2400-2402
// 修改后的描述符通过原子操作回写到页表内存
```

### 15.4 特性检测

```c
// cpu-features.h:307
// ID_AA64MMFR1.HAFDBS 字段
// cpu64.c:1307 — CPU 模型中声明
```

---

## 16. FEAT_AIE 属性间接扩展

### 16.1 MAIR2/AMAIR2 寄存器

```c
// helper.c:6179-6209
{ .name = "MAIR2_EL1",  .fieldoffset = offsetof(CPUARMState, cp15.mair2_el[1]) }
{ .name = "MAIR2_EL2",  .fieldoffset = offsetof(CPUARMState, cp15.mair2_el[2]) }
{ .name = "MAIR2_EL3",  .fieldoffset = offsetof(CPUARMState, cp15.mair2_el[3]) }
{ .name = "AMAIR2_EL1", .fieldoffset = offsetof(CPUARMState, cp15.amair2_el[1]) }
{ .name = "AMAIR2_EL2", .fieldoffset = offsetof(CPUARMState, cp15.amair2_el[2]) }
{ .name = "AMAIR2_EL3", .fieldoffset = offsetof(CPUARMState, cp15.amair2_el[3]) }
```

AIE 将 MAIR（Memory Attribute Indirection Register）从 8 个条目扩展到 16 个条目，通过 MAIR2 提供额外 8 个属性槽位。AMAIR2 提供对应的辅助属性。

### 16.2 特性检测

```c
// cpu-features.h:344 — ID_AA64MMFR3.AIE 字段
// cpu64.c:1344 — CPU 模型声明
```

---

## 17. 其他新特性概览

### 17.1 FEAT_MPAM — 内存分区与监控

| 项目 | 状态 |
|------|------|
| 特性检测 | `ID_AA64PFR0.MPAM`（cpu-features.h:257）|
| 系统寄存器 | ⚠️ 未实现（仅 TODO 注释，cpu64.c:1015,1117）|
| 运行时行为 | QEMU 屏蔽 MPAM 字段（cpu.c:2207-2208），报告为不支持 |

MPAM 在 QEMU TCG 中仅为存根——因为 MPAM 主要影响缓存/内存控制器的资源分配，在纯软件模拟环境中无实际意义。

### 17.2 FEAT_HPMN0

```c
// cpu-features.h:375,413 — 特性字段定义
// cpu64.c:1367, cpu32.c:115 — CPU 模型声明
```

允许 PMU 中 HPMN 设为 0（所有计数器归 Hypervisor 管理）。

### 17.3 FEAT_FPMR

```c
// cpu-features.h:283 — 特性字段定义
// 无寄存器实现
```

Floating-point Mode Register，目前仅声明特性位，无运行时实现。

### 17.4 FEAT_THE — 翻译硬化扩展

```c
// cpu-features.h:275 — 特性字段定义
// syndrome.h:514 — 综合征解码
// 无完整实现
```

### 17.5 未实现的特性

以下特性在 QEMU 源码中未找到实现：
- **FEAT_ABLE**：无引用
- **FEAT_ANERR / FEAT_ADERR**：无引用

---

## 18. TB 标志集成总览

### 18.1 GCS TB 标志

| 标志 | 位置 | 含义 |
|------|------|------|
| `GCS_EN` | hflags.c:469 | GCS 推入/弹出使能 |
| `GCS_RVCEN` | hflags.c:476 | 返回值检查使能 |
| `GCSSTR_EL` | hflags.c:481 | GCSSTR 陷入目标 EL |

### 18.2 RME 相关标志

RME 主要通过安全状态和 MMU 索引影响翻译，不直接添加 TB 标志。GPC 检查在 PTW 中执行，透明于翻译块。

### 18.3 NMI 相关

NMI 处理在中断路由阶段（`cpu-irq.c`），不影响 TB 标志。`PSTATE.ALLINT` 通过标准 PSTATE 保存/恢复机制处理。

---

## 19. 总结

### 特性实现矩阵

```
+------------------+     +------------------+     +------------------+
|       GCS        |     |       RME        |     |    NMI/PIE/等    |
| 守护控制栈       |     | 领域管理扩展     |     | 其他新扩展       |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
   影子返回地址栈          四安全域隔离           中断/权限/属性增强
         |                        |                        |
   BL推入/RET弹出          GPT粒度保护表          ALLINT/PIR/HAFDBS
         |                        |                        |
  GCSPR_EL0-3栈指针       Root/Realm新状态        权限间接与脏位管理
         |                        |                        |
  EXLOCK异常锁定          SCR.NS/NSE编码          硬件自动AF/DBM
+--------+---------+     +--------+---------+     +--------+---------+
| cpregs-gcs.c     |     | ptw.c:333-571    |     | cpu-irq.c        |
| translate-a64.c  |     | helper.c         |     | helper.c         |
| hflags.c:452-487 |     | tlb_helper.c     |     | ptw.c:2143-2178  |
+------------------+     +------------------+     +------------------+
```

### 关键源码文件索引

| 文件 | 关键行号 | 内容 |
|------|---------|------|
| cpu.h | 586-587 | GCSCR/GCSPR 存储 |
| cpu.h | 1559,1570 | PSTATE_ALLINT, PSTATE_EXLOCK |
| translate-a64.c | 439-470 | gen_add/load_check_gcs_record |
| translate-a64.c | 1704-1948 | BL/BLR/RET GCS 集成 |
| translate-a64.c | 2568-2742 | GCS 指令（PUSHM/POPM/PUSHX/POPCX/SS1/SS2）|
| translate-a64.c | 4293-4326 | GCSSTR 翻译 |
| cpregs-gcs.c | 83-149 | GCS 系统寄存器定义 |
| hflags.c | 452-487 | GCS TB 标志 |
| mmuidx.h | 137-186 | GCS MMU 索引 |
| mmuidx.c | 26-53 | GCS MMU 索引映射 |
| arm-security.h | 18-23 | ARMSecuritySpace 四安全状态 |
| helper.c | 10131-10187 | arm_security_space() |
| ptw.c | 333-571 | arm_granule_protection_check() GPC |
| ptw.c | 193-225 | Stage-2 PTW 安全域路由 |
| helper.c | 4973-4992 | GPCCR/GPTBR/MFAR 寄存器 |
| helper.c | 5062-5105 | MECID 寄存器 |
| tlb_helper.c | 122-237 | GPC 故障路由 |
| cpu-features.h | 1107-1114 | FEAT_RME 检测 |
| cpu-features.h | 1170-1173 | FEAT_GCS 检测 |
| cpu-irq.c | 34-57 | NMI 中断路由 |
| helper.c | 4994-5010 | ALLINT 寄存器 |
| helper.c | 6120-6209 | PIR/PIRE0/S2PIR/MAIR2/AMAIR2 |
| ptw.c | 2143-2178 | HAFDBS AF/DBM 处理 |
| cpu-features.h | 257,272,307,335,344 | MPAM/NMI/HAFDBS/S1PIE/AIE 检测 |

---

> 文档生成时间：基于 QEMU 11.0.50 源码分析
> 分析工具：zoekt + ctags + cscope 索引
