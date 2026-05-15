# CPU 特性与 ID 寄存器仿真深度分析

> QEMU 版本: 11.0.50  
> 源码路径: https://github.com/qemu/qemu  
> 文档更新: 2025-07  
> 相关文档: [00-ARM64-CPU-GICv3-TCG深度分析](00-ARM64-CPU-GICv3-TCG深度分析.md) | [06-异常级别状态管理深度分析](06-异常级别状态管理深度分析.md) | [07-不同EL下指令执行差异深度分析](07-不同EL下指令执行差异深度分析.md)

---

## 目录

1. [概述](#1-概述)
2. [ID 寄存器基础架构](#2-id-寄存器基础架构)
3. [ARMISARegisters 与 idregs 数组](#3-armisaregisters-与-idregs-数组)
4. [ID 寄存器索引定义：cpu-sysregs.h.inc](#4-id-寄存器索引定义cpu-sysregshinc)
5. [ID_AA64PFR0_EL1：处理器特性寄存器 0](#5-id_aa64pfr0_el1处理器特性寄存器-0)
6. [ID_AA64PFR1_EL1：处理器特性寄存器 1](#6-id_aa64pfr1_el1处理器特性寄存器-1)
7. [ID_AA64ISAR0_EL1：指令集属性寄存器 0](#7-id_aa64isar0_el1指令集属性寄存器-0)
8. [ID_AA64ISAR1_EL1：指令集属性寄存器 1](#8-id_aa64isar1_el1指令集属性寄存器-1)
9. [ID_AA64ISAR2_EL1：指令集属性寄存器 2](#9-id_aa64isar2_el1指令集属性寄存器-2)
10. [ID_AA64MMFR0/1/2_EL1：内存模型特性寄存器](#10-id_aa64mmfr012_el1内存模型特性寄存器)
11. [ID_AA64DFR0_EL1：调试特性寄存器](#11-id_aa64dfr0_el1调试特性寄存器)
12. [ID_AA64ZFR0/SMFR0_EL1：SVE/SME 特性寄存器](#12-id_aa64zfr0smfr0_el1svesme-特性寄存器)
13. [特性检测系统：isar_feature_*()](#13-特性检测系统isar_feature_)
14. [CPU 模型定义](#14-cpu-模型定义)
15. [Cortex-A57 模型详解](#15-cortex-a57-模型详解)
16. ["-cpu max" 类型：TCG 最大能力](#16--cpu-max-类型tcg-最大能力)
17. [CPU 特性 QOM 属性](#17-cpu-特性-qom-属性)
18. [特性终化流程：arm_cpu_finalize_features](#18-特性终化流程arm_cpu_finalize_features)
19. [arm_cpu_realizefn：CPU 实现函数](#19-arm_cpu_realizefncpu-实现函数)
20. [Guest 读取 ID 寄存器：系统寄存器访问](#20-guest-读取-id-寄存器系统寄存器访问)
21. [ID 寄存器 Trap：FEAT_IDST 与 TID3](#21-id-寄存器-trapfeat_idst-与-tid3)
22. [KVM 模式：宿主特性发现](#22-kvm-模式宿主特性发现)
23. [翻译时特性检查：dc_isar_feature()](#23-翻译时特性检查dc_isar_feature)
24. [TB Flags 中的特性编码](#24-tb-flags-中的特性编码)
25. [FEAT_LSE 完整实现追踪](#25-feat_lse-完整实现追踪)
26. [FEAT_PAuth 完整实现追踪](#26-feat_pauth-完整实现追踪)
27. [FEAT_MTE 完整实现追踪](#27-feat_mte-完整实现追踪)
28. [FEAT_BTI 完整实现追踪](#28-feat_bti-完整实现追踪)
29. [旧特性标志系统：ARM_FEATURE_*](#29-旧特性标志系统arm_feature_)
30. [迁移兼容性](#30-迁移兼容性)
31. [完整数据流总结](#31-完整数据流总结)
32. [总结](#32-总结)

---

## 1. 概述

ARM64 架构通过一组 **ID 寄存器**（Identification Registers）向软件暴露 CPU 的能力集。Guest OS 通过 `MRS` 指令读取这些寄存器来检测可用的指令集扩展（如 LSE 原子操作、SVE 向量扩展、PAuth 指针认证等），并据此选择代码路径。

QEMU 必须精确仿真这些 ID 寄存器，因为：
- Guest 内核（如 Linux）在启动时读取 ID 寄存器来决定启用哪些特性
- 不同 CPU 模型（如 Cortex-A57 vs Neoverse-N2）有不同的特性集
- `-cpu max` 模式需要暴露 QEMU 支持的所有特性
- KVM 模式需要将宿主 CPU 的真实特性传递给 Guest

本文深入分析 QEMU 如何定义、存储、访问和管理 ARM64 ID 寄存器及特性检测系统。

```
ID 寄存器仿真的完整数据流：

CPU 模型 initfn()
  ↓ SET_IDREG(isar, ID_AA64*, value)
ARMISARegisters.idregs[]
  ↓ arm_cpu_finalize_features()
  ↓ arm_cpu_realizefn()
最终 ID 寄存器值确定
  │
  ├── Guest 读取路径：MRS → ARMCPRegInfo → GET_IDREG() → 返回值
  ├── 翻译检查路径：isar_feature_*() → dc_isar_feature() → 条件编译
  └── TB Flags 路径：rebuild_hflags_a64() → TB 标志 → 翻译缓存
```

---

## 2. ID 寄存器基础架构

QEMU 的 ID 寄存器仿真围绕三个核心机制构建：

| 机制 | 文件 | 作用 |
|------|------|------|
| **统一存储** | `cpu.h:1085-1094` | `ARMISARegisters.idregs[]` 数组存储所有 ID 寄存器值 |
| **字段提取** | `cpu.h:885-900` | `FIELD_EX64_IDREG()` 宏从寄存器中提取指定字段 |
| **特性谓词** | `cpu-features.h:807-1140+` | `isar_feature_aa64_*()` 函数检查特定特性是否可用 |

核心设计原则：
- **单一数据源**：所有 ID 寄存器值存储在 `idregs[]` 数组中，通过索引访问
- **声明式字段定义**：使用 `FIELD()` 宏声明每个寄存器的字段布局
- **类型安全**：所有访问通过宏完成，确保编译时类型检查

---

## 3. ARMISARegisters 与 idregs 数组

```c
// cpu.h:1085-1094 — ARMCPU 内嵌的 ISA 寄存器集合
struct ARMISARegisters {
    uint32_t mvfr0;           // 浮点特性寄存器 0（AArch32）
    uint32_t mvfr1;           // 浮点特性寄存器 1
    uint32_t mvfr2;           // 浮点特性寄存器 2
    uint32_t dbgdidr;         // 调试 ID 寄存器
    uint32_t dbgdevid;        // 调试设备 ID
    uint32_t dbgdevid1;       // 调试设备 ID 1
    uint64_t reset_pmcr_el0;  // PMU 控制寄存器复位值
    uint64_t idregs[NUM_ID_IDX]; // 所有 AArch64/AArch32 ID 寄存器
} isar;
```

`idregs[]` 是一个固定大小的 `uint64_t` 数组，通过枚举索引访问。每个 ID 寄存器在数组中占一个 slot。

### 3.1 访问宏

```c
// cpu.h:903-914 — 读写 ID 寄存器的统一宏
#define SET_IDREG(ISAR, REG, VALUE)                                     \
    ({                                                                  \
        ARMISARegisters *i_ = (ISAR);                                   \
        i_->idregs[REG ## _EL1_IDX] = VALUE;                            \
    })

#define GET_IDREG(ISAR, REG)                                            \
    ({                                                                  \
        const ARMISARegisters *i_ = (ISAR);                             \
        i_->idregs[REG ## _EL1_IDX];                                    \
    })
```

### 3.2 字段提取宏

```c
// cpu.h:885-900 — 从 ID 寄存器中提取指定字段
#define FIELD_EX64_IDREG(ISAR, REG, FIELD)                              \
    ({                                                                  \
        const ARMISARegisters *i_ = (ISAR);                             \
        FIELD_EX64(i_->idregs[REG ## _EL1_IDX], REG, FIELD);            \
    })

// 32 位变体
#define FIELD_EX32_IDREG(ISAR, REG, FIELD)                              \
    ({                                                                  \
        const ARMISARegisters *i_ = (ISAR);                             \
        FIELD_EX32(i_->idregs[REG ## _EL1_IDX], REG, FIELD);            \
    })
```

### 3.3 字段修改宏

```c
// cpu.h:916+ — 修改 ID 寄存器的单个字段
#define FIELD_DP64_IDREG(ISAR, REG, FIELD, VALUE)                       \
    ({                                                                  \
        ARMISARegisters *i_ = (ISAR);                                   \
        uint64_t regval = i_->idregs[REG ## _EL1_IDX];                  \
        regval = FIELD_DP64(regval, REG, FIELD, VALUE);                  \
        i_->idregs[REG ## _EL1_IDX] = regval;                           \
    })
```

---

## 4. ID 寄存器索引定义：cpu-sysregs.h.inc

`target/arm/cpu-sysregs.h.inc` 使用 `DEF()` 宏为每个 ID 寄存器定义索引和系统寄存器编码：

```c
// cpu-sysregs.h.inc:1-43 — 所有 ID 寄存器的统一定义
// 格式: DEF(名称, op0, op1, crn, crm, op2)

// AArch64 处理器特性
DEF(ID_AA64PFR0_EL1, 3, 0, 0, 4, 0)
DEF(ID_AA64PFR1_EL1, 3, 0, 0, 4, 1)
DEF(ID_AA64PFR2_EL1, 3, 0, 0, 4, 2)
DEF(ID_AA64SMFR0_EL1, 3, 0, 0, 4, 5)

// AArch64 调试特性
DEF(ID_AA64DFR0_EL1, 3, 0, 0, 5, 0)
DEF(ID_AA64DFR1_EL1, 3, 0, 0, 5, 1)

// AArch64 辅助特性
DEF(ID_AA64AFR0_EL1, 3, 0, 0, 5, 4)
DEF(ID_AA64AFR1_EL1, 3, 0, 0, 5, 5)

// AArch64 指令集属性
DEF(ID_AA64ISAR0_EL1, 3, 0, 0, 6, 0)
DEF(ID_AA64ISAR1_EL1, 3, 0, 0, 6, 1)
DEF(ID_AA64ISAR2_EL1, 3, 0, 0, 6, 2)

// AArch64 内存模型特性
DEF(ID_AA64MMFR0_EL1, 3, 0, 0, 7, 0)
DEF(ID_AA64MMFR1_EL1, 3, 0, 0, 7, 1)
DEF(ID_AA64MMFR2_EL1, 3, 0, 0, 7, 2)
DEF(ID_AA64MMFR3_EL1, 3, 0, 0, 7, 3)
DEF(ID_AA64MMFR4_EL1, 3, 0, 0, 7, 4)

// SVE/SME 特性
DEF(ID_AA64ZFR0_EL1, 3, 0, 0, 4, 4)

// AArch32 兼容 ID 寄存器（共 20+ 个）
DEF(ID_PFR0_EL1, 3, 0, 0, 1, 0)
...
DEF(CLIDR_EL1, 3, 1, 0, 0, 1)    // Cache Level ID
DEF(CTR_EL0, 3, 3, 0, 0, 1)      // Cache Type Register
DEF(DCZID_EL0, 3, 3, 0, 0, 7)    // DC ZVA ID
```

这些 `DEF()` 宏会被展开两次：
1. 生成 `enum { ID_AA64PFR0_EL1_IDX, ... }` 枚举作为 `idregs[]` 的索引
2. 生成系统寄存器编码，用于 `ARMCPRegInfo` 注册

---

## 5. ID_AA64PFR0_EL1：处理器特性寄存器 0

```c
// cpu-features.h:247-262 — ID_AA64PFR0 字段定义
FIELD(ID_AA64PFR0, EL0,     0, 4)   // EL0 支持的执行状态（AArch64/AArch32）
FIELD(ID_AA64PFR0, EL1,     4, 4)   // EL1 支持的执行状态
FIELD(ID_AA64PFR0, EL2,     8, 4)   // EL2 支持的执行状态
FIELD(ID_AA64PFR0, EL3,    12, 4)   // EL3 支持的执行状态
FIELD(ID_AA64PFR0, FP,     16, 4)   // 浮点支持（0=FP, 1=FP16, 0xF=不支持）
FIELD(ID_AA64PFR0, ADVSIMD,20, 4)   // Advanced SIMD（NEON）支持
FIELD(ID_AA64PFR0, GIC,    24, 4)   // GIC 系统寄存器接口版本
FIELD(ID_AA64PFR0, RAS,    28, 4)   // RAS 扩展版本
FIELD(ID_AA64PFR0, SVE,    32, 4)   // SVE 支持（0=无, 1=SVEv1）
FIELD(ID_AA64PFR0, SEL2,   36, 4)   // Secure EL2 支持
FIELD(ID_AA64PFR0, MPAM,   40, 4)   // MPAM 版本
FIELD(ID_AA64PFR0, AMU,    44, 4)   // 活动监控单元版本
FIELD(ID_AA64PFR0, DIT,    48, 4)   // 数据独立时序
FIELD(ID_AA64PFR0, RME,    52, 4)   // Realm Management Extension
FIELD(ID_AA64PFR0, CSV2,   56, 4)   // Spectre-v2 缓解
FIELD(ID_AA64PFR0, CSV3,   60, 4)   // Spectre-v3 缓解
```

**典型值示例**：

| CPU 模型 | ID_AA64PFR0 值 | 含义 |
|---------|----------------|------|
| Cortex-A57 | `0x00002222` | EL0-3 仅 AArch64, 无 SVE/GIC |
| `-cpu max` (TCG) | 高位多字段设置 | 支持 SVE/RAS/DIT/CSV2/CSV3 等 |

---

## 6. ID_AA64PFR1_EL1：处理器特性寄存器 1

```c
// cpu-features.h:264-278 — ID_AA64PFR1 字段定义
FIELD(ID_AA64PFR1, BT,        0, 4)   // FEAT_BTI（分支目标标识）
FIELD(ID_AA64PFR1, SSBS,      4, 4)   // FEAT_SSBS（推测存储旁路安全）
FIELD(ID_AA64PFR1, MTE,       8, 4)   // FEAT_MTE（内存标记扩展，0/1/2/3）
FIELD(ID_AA64PFR1, RAS_FRAC, 12, 4)   // RAS 分数版本
FIELD(ID_AA64PFR1, MPAM_FRAC,16, 4)   // MPAM 分数版本
FIELD(ID_AA64PFR1, SME,      24, 4)   // FEAT_SME（可扩展矩阵扩展）
FIELD(ID_AA64PFR1, RNDR_TRAP,28, 4)   // RNDR/RNDRRS 陷阱控制
FIELD(ID_AA64PFR1, CSV2_FRAC,32, 4)   // CSV2 分数版本
FIELD(ID_AA64PFR1, NMI,      36, 4)   // FEAT_NMI（不可屏蔽中断）
FIELD(ID_AA64PFR1, MTE_FRAC, 40, 4)   // MTE 分数版本
FIELD(ID_AA64PFR1, GCS,      44, 4)   // FEAT_GCS（Guarded Control Stack）
FIELD(ID_AA64PFR1, THE,      48, 4)   // FEAT_THE
FIELD(ID_AA64PFR1, MTEX,     52, 4)   // MTE 扩展
FIELD(ID_AA64PFR1, DF2,      56, 4)   // 调试特性扩展
FIELD(ID_AA64PFR1, PFAR,     60, 4)   // 物理故障地址寄存器
```

---

## 7. ID_AA64ISAR0_EL1：指令集属性寄存器 0

```c
// cpu-features.h:198-212 — ID_AA64ISAR0 字段定义
FIELD(ID_AA64ISAR0, AES,    4, 4)   // AES 指令（0=无, 1=AES, 2=+PMULL）
FIELD(ID_AA64ISAR0, SHA1,   8, 4)   // SHA1 指令
FIELD(ID_AA64ISAR0, SHA2,  12, 4)   // SHA256/SHA512 指令
FIELD(ID_AA64ISAR0, CRC32, 16, 4)   // CRC32 指令
FIELD(ID_AA64ISAR0, ATOMIC, 20, 4)  // 原子操作（0=无, 2=LSE, 3=LSE128）
FIELD(ID_AA64ISAR0, TME,   24, 4)   // 事务内存扩展
FIELD(ID_AA64ISAR0, RDM,   28, 4)   // SQRDMLAH/SQRDMLSH 指令
FIELD(ID_AA64ISAR0, SHA3,  32, 4)   // SHA3 指令
FIELD(ID_AA64ISAR0, SM3,   36, 4)   // SM3 国密指令
FIELD(ID_AA64ISAR0, SM4,   40, 4)   // SM4 国密指令
FIELD(ID_AA64ISAR0, DP,    44, 4)   // 点积指令
FIELD(ID_AA64ISAR0, FHM,   48, 4)   // FMLAL/FMLSL 指令
FIELD(ID_AA64ISAR0, TS,    52, 4)   // 条件标志操作（FlagM/FlagM2）
FIELD(ID_AA64ISAR0, TLB,   56, 4)   // TLB 维护指令（TLBI Range）
FIELD(ID_AA64ISAR0, RNDR,  60, 4)   // 随机数生成指令
```

**ATOMIC 字段值含义**：

| 值 | 含义 | FEAT |
|----|------|------|
| 0 | 无原子操作扩展 | — |
| 2 | FEAT_LSE（CAS/CASP/LDADD/SWP 等） | LSE |
| 3 | FEAT_LSE128（128位原子操作） | LSE + LSE128 |

---

## 8. ID_AA64ISAR1_EL1：指令集属性寄存器 1

```c
// cpu-features.h:214-229 — ID_AA64ISAR1 字段定义
FIELD(ID_AA64ISAR1, DPB,     0, 4)   // DC CVAP/CVADP 数据缓存清理
FIELD(ID_AA64ISAR1, APA,     4, 4)   // PAuth QARMA5 算法（地址密钥）
FIELD(ID_AA64ISAR1, API,     8, 4)   // PAuth 实现定义算法
FIELD(ID_AA64ISAR1, JSCVT,  12, 4)   // JavaScript 浮点转换
FIELD(ID_AA64ISAR1, FCMA,   16, 4)   // 复数乘法累加
FIELD(ID_AA64ISAR1, LRCPC,  20, 4)   // LDAPR 系列指令（Release Consistency）
FIELD(ID_AA64ISAR1, GPA,    24, 4)   // PAuth QARMA5 通用密钥
FIELD(ID_AA64ISAR1, GPI,    28, 4)   // PAuth 实现定义通用密钥
FIELD(ID_AA64ISAR1, FRINTTS,32, 4)   // FRINT32/FRINT64 指令
FIELD(ID_AA64ISAR1, SB,     36, 4)   // 推测屏障指令
FIELD(ID_AA64ISAR1, SPECRES,40, 4)   // 推测限制（CFPRCTX 等）
FIELD(ID_AA64ISAR1, BF16,   44, 4)   // BFloat16 指令
FIELD(ID_AA64ISAR1, DGH,    48, 4)   // 数据收集提示
FIELD(ID_AA64ISAR1, I8MM,   52, 4)   // Int8 矩阵乘法
FIELD(ID_AA64ISAR1, XS,     56, 4)   // XS 属性（nXS TLBI）
FIELD(ID_AA64ISAR1, LS64,   60, 4)   // LD64B/ST64B 指令
```

---

## 9. ID_AA64ISAR2_EL1：指令集属性寄存器 2

```c
// cpu-features.h:231-245 — ID_AA64ISAR2 字段定义（部分）
FIELD(ID_AA64ISAR2, WFXT,    0, 4)   // WFET/WFIT 指令
FIELD(ID_AA64ISAR2, RPRES,   4, 4)   // 倒数/平方根精度
FIELD(ID_AA64ISAR2, GPA3,    8, 4)   // PAuth QARMA3 通用密钥
FIELD(ID_AA64ISAR2, APA3,   12, 4)   // PAuth QARMA3 地址密钥
FIELD(ID_AA64ISAR2, MOPS,   16, 4)   // 内存操作指令（CPYFP/SETP 等）
FIELD(ID_AA64ISAR2, BC,     20, 4)   // HBC（Hint-based Barriers）
FIELD(ID_AA64ISAR2, PAC_FRAC,24, 4)  // PAC 分数版本
FIELD(ID_AA64ISAR2, CLRBHB, 28, 4)   // CLRBHB 指令
FIELD(ID_AA64ISAR2, SYSREG128,32, 4) // 128 位系统寄存器
FIELD(ID_AA64ISAR2, SYSINSTR128,36,4) // 128 位系统指令
FIELD(ID_AA64ISAR2, PRFMSLC,40, 4)   // PRFM SLC 提示
FIELD(ID_AA64ISAR2, RPRFM,  48, 4)   // 范围预取
FIELD(ID_AA64ISAR2, CSSC,   52, 4)   // 通用标量/字符串操作
```

---

## 10. ID_AA64MMFR0/1/2_EL1：内存模型特性寄存器

### 10.1 MMFR0

```c
// cpu-features.h:285-298 — ID_AA64MMFR0 字段定义
FIELD(ID_AA64MMFR0, PARANGE,   0, 4)   // 物理地址范围（0=32位..6=52位）
FIELD(ID_AA64MMFR0, ASIDBITS,  4, 4)   // ASID 位数（0=8位, 2=16位）
FIELD(ID_AA64MMFR0, BIGEND,    8, 4)   // 混合端序支持
FIELD(ID_AA64MMFR0, SNSMEM,   12, 4)   // Secure/Non-secure 内存区分
FIELD(ID_AA64MMFR0, BIGENDEL0,16, 4)   // EL0 混合端序支持
FIELD(ID_AA64MMFR0, TGRAN16,  20, 4)   // 16KB 页粒度支持
FIELD(ID_AA64MMFR0, TGRAN64,  24, 4)   // 64KB 页粒度支持
FIELD(ID_AA64MMFR0, TGRAN4,   28, 4)   // 4KB 页粒度支持
FIELD(ID_AA64MMFR0, TGRAN16_2,32, 4)   // Stage-2 16KB 页支持
FIELD(ID_AA64MMFR0, TGRAN64_2,36, 4)   // Stage-2 64KB 页支持
FIELD(ID_AA64MMFR0, TGRAN4_2, 40, 4)   // Stage-2 4KB 页支持
FIELD(ID_AA64MMFR0, EXS,      44, 4)   // 错误异常同步
FIELD(ID_AA64MMFR0, FGT,      56, 4)   // FEAT_FGT（细粒度陷阱）
FIELD(ID_AA64MMFR0, ECV,      60, 4)   // FEAT_ECV（增强计数虚拟化）
```

### 10.2 关键 MMFR1/MMFR2 字段

| 寄存器 | 字段 | 含义 |
|--------|------|------|
| MMFR1 | HAFDBS | 硬件更新访问/脏标志 |
| MMFR1 | PAN | FEAT_PAN（特权访问不执行） |
| MMFR1 | LO | FEAT_LOR（限制排序区域） |
| MMFR1 | VH | FEAT_VHE（虚拟化宿主扩展） |
| MMFR2 | FWB | FEAT_S2FWB（Stage-2 强制回写） |
| MMFR2 | IDS | FEAT_IDST（ID 寄存器到 EL2 陷阱） |
| MMFR2 | NV | FEAT_NV（嵌套虚拟化） |
| MMFR2 | TTL | TLB 维护指令层级提示 |

---

## 11. ID_AA64DFR0_EL1：调试特性寄存器

```c
// cpu-features.h — ID_AA64DFR0 字段定义
FIELD(ID_AA64DFR0, DEBUGVER,  0, 4)   // 调试架构版本
FIELD(ID_AA64DFR0, TRACEVER,  4, 4)   // 跟踪版本
FIELD(ID_AA64DFR0, PMUVER,    8, 4)   // PMU 版本
FIELD(ID_AA64DFR0, BRPS,     12, 4)   // 断点数 - 1
FIELD(ID_AA64DFR0, WRPS,     20, 4)   // 观察点数 - 1
FIELD(ID_AA64DFR0, CTX_CMPS, 28, 4)   // 上下文比较器数 - 1
FIELD(ID_AA64DFR0, PMSVER,   32, 4)   // 统计分析版本（SPE）
FIELD(ID_AA64DFR0, DOUBLELOCK,36, 4)  // 双锁支持
FIELD(ID_AA64DFR0, TRACEFILT,40, 4)   // 跟踪过滤
FIELD(ID_AA64DFR0, TRACEBUFFER,44, 4) // 跟踪缓冲区
FIELD(ID_AA64DFR0, HPMN0,    60, 4)   // HPMN 为零行为
```

---

## 12. ID_AA64ZFR0/SMFR0_EL1：SVE/SME 特性寄存器

### 12.1 ID_AA64ZFR0（SVE 特性）

```c
// cpu-features.h:377-388 — SVE 子特性字段
FIELD(ID_AA64ZFR0, SVEVER,   0, 4)   // SVE 版本（0=SVE, 1=SVE2, 2=SVE2.1）
FIELD(ID_AA64ZFR0, AES,      4, 4)   // SVE AES 指令
FIELD(ID_AA64ZFR0, ELTPERM, 12, 4)   // 元素置换
FIELD(ID_AA64ZFR0, BITPERM, 16, 4)   // 位置换指令
FIELD(ID_AA64ZFR0, BFLOAT16,20, 4)   // SVE BFloat16
FIELD(ID_AA64ZFR0, B16B16,  24, 4)   // 非扩展 BF16
FIELD(ID_AA64ZFR0, SHA3,    32, 4)   // SVE SHA3
FIELD(ID_AA64ZFR0, SM4,     40, 4)   // SVE SM4
FIELD(ID_AA64ZFR0, I8MM,    44, 4)   // SVE Int8 矩阵乘法
FIELD(ID_AA64ZFR0, F16MM,   48, 4)   // SVE FP16 矩阵乘法
FIELD(ID_AA64ZFR0, F32MM,   52, 4)   // SVE FP32 矩阵乘法
FIELD(ID_AA64ZFR0, F64MM,   56, 4)   // SVE FP64 矩阵乘法
```

### 12.2 ID_AA64SMFR0（SME 特性）

| 字段 | 含义 |
|------|------|
| SMEVER | SME 版本（1=SME, 2=SME2） |
| FA64 | 流模式下支持全 A64 指令集 |
| F16F16 | FP16 外积 |
| F64F64 | FP64 外积 |
| I16I64 | Int16→Int64 外积 |
| F16F32 | FP16→FP32 外积 |

---

## 13. 特性检测系统：isar_feature_*()

QEMU 定义了 **168 个** `isar_feature_aa64_*()` 内联函数（`cpu-features.h:807-1140+`），每个函数检查一个特定的 ARM 特性是否可用。

### 13.1 命名约定

```
isar_feature_aa64_*   → AArch64 特性检查
isar_feature_aa32_*   → AArch32 特性检查
isar_feature_any_*    → 合并 AArch32/AArch64 检查
```

### 13.2 典型实现模式

```c
// cpu-features.h:807-810 — 简单非零检查
static inline bool isar_feature_aa64_aes(const ARMISARegisters *id)
{
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, AES) != 0;
}

// cpu-features.h:837-840 — 阈值检查（>= 2 表示 FEAT_LSE）
static inline bool isar_feature_aa64_lse(const ARMISARegisters *id)
{
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, ATOMIC) >= 2;
}

// cpu-features.h:842-845 — 更高阈值（>= 3 表示 FEAT_LSE128）
static inline bool isar_feature_aa64_lse128(const ARMISARegisters *id)
{
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, ATOMIC) >= 3;
}
```

### 13.3 复合检查模式（PAuth）

```c
// cpu-features.h:931-950 — 多字段联合检查
static inline ARMPauthFeature
isar_feature_pauth_feature(const ARMISARegisters *id)
{
    // 架构上，APA/API/APA3 只能有一个非零
    return (FIELD_EX64_IDREG(id, ID_AA64ISAR1, APA) |
            FIELD_EX64_IDREG(id, ID_AA64ISAR1, API) |
            FIELD_EX64_IDREG(id, ID_AA64ISAR2, APA3));
}

static inline bool isar_feature_aa64_pauth(const ARMISARegisters *id)
{
    return isar_feature_pauth_feature(id) != PauthFeat_None;
}
```

### 13.4 CPU 级别调用宏

```c
// cpu-features.h:1696-1698 — 从 ARMCPU 结构体调用
#define cpu_isar_feature(name, cpu) \
    ({ const ARMCPU *cpu_ = (cpu); isar_feature_##name(&cpu_->isar); })
```

### 13.5 关键特性映射表

| 特性函数 | ID 寄存器 | 字段 | 阈值 | 对应 FEAT |
|---------|----------|------|------|----------|
| `aa64_aes` | ID_AA64ISAR0 | AES | ≠ 0 | FEAT_AES |
| `aa64_pmull` | ID_AA64ISAR0 | AES | > 1 | FEAT_PMULL |
| `aa64_sha256` | ID_AA64ISAR0 | SHA2 | ≠ 0 | FEAT_SHA256 |
| `aa64_sha512` | ID_AA64ISAR0 | SHA2 | > 1 | FEAT_SHA512 |
| `aa64_lse` | ID_AA64ISAR0 | ATOMIC | ≥ 2 | FEAT_LSE |
| `aa64_lse128` | ID_AA64ISAR0 | ATOMIC | ≥ 3 | FEAT_LSE128 |
| `aa64_rdm` | ID_AA64ISAR0 | RDM | ≠ 0 | FEAT_RDM |
| `aa64_dp` | ID_AA64ISAR0 | DP | ≠ 0 | FEAT_DotProd |
| `aa64_pauth` | ID_AA64ISAR1 | APA\|API\|APA3 | ≠ 0 | FEAT_PAuth |
| `aa64_sb` | ID_AA64ISAR1 | SB | ≠ 0 | FEAT_SB |
| `aa64_bti` | ID_AA64PFR1 | BT | ≠ 0 | FEAT_BTI |
| `aa64_mte` | ID_AA64PFR1 | MTE | ≥ 2 | FEAT_MTE2 |
| `aa64_sme` | ID_AA64PFR1 | SME | ≠ 0 | FEAT_SME |
| `aa64_sve` | ID_AA64PFR0 | SVE | ≠ 0 | FEAT_SVE |
| `aa64_fgt` | ID_AA64MMFR0 | FGT | ≠ 0 | FEAT_FGT |
| `aa64_vh` | ID_AA64MMFR1 | VH | ≠ 0 | FEAT_VHE |
| `aa64_pan` | ID_AA64MMFR1 | PAN | ≠ 0 | FEAT_PAN |
| `aa64_mops` | ID_AA64ISAR2 | MOPS | ≠ 0 | FEAT_MOPS |

---

## 14. CPU 模型定义

QEMU 为 ARM64 定义了多个 CPU 模型，分布在两个文件中：

### 14.1 主模型表（cpu64.c:897-901）

```c
// cpu64.c:897-901 — 核心 CPU 模型（始终可用）
{ .name = "cortex-a57",  .initfn = aarch64_a57_initfn },
{ .name = "cortex-a53",  .initfn = aarch64_a53_initfn },
{ .name = "max",         .initfn = aarch64_max_initfn },
#if defined(CONFIG_KVM) || defined(CONFIG_HVF) || defined(CONFIG_WHPX)
{ .name = "host",        .initfn = aarch64_host_initfn },
#endif
```

### 14.2 TCG 专用模型（tcg/cpu64.c:1413-1428）

```c
// tcg/cpu64.c:1413-1428 — 仅 TCG 模式可用的模型
{ .name = "cortex-a35",    .initfn = aarch64_a35_initfn },
{ .name = "cortex-a55",    .initfn = aarch64_a55_initfn },
{ .name = "cortex-a72",    .initfn = aarch64_a72_initfn },
{ .name = "cortex-a76",    .initfn = aarch64_a76_initfn },
{ .name = "cortex-a78ae",  .initfn = aarch64_a78ae_initfn },
{ .name = "cortex-a710",   .initfn = aarch64_a710_initfn },
{ .name = "a64fx",         .initfn = aarch64_a64fx_initfn },
{ .name = "neoverse-n1",   .initfn = aarch64_neoverse_n1_initfn },
{ .name = "neoverse-v1",   .initfn = aarch64_neoverse_v1_initfn },
{ .name = "neoverse-n2",   .initfn = aarch64_neoverse_n2_initfn },
```

### 14.3 模型与真实 CPU 的对应关系

| QEMU 模型 | 真实 CPU | 架构版本 | 关键特性 |
|-----------|---------|---------|---------|
| cortex-a35 | Cortex-A35 | ARMv8.0 | 基础 AArch64 |
| cortex-a53 | Cortex-A53 | ARMv8.0 | 低功耗核心 |
| cortex-a55 | Cortex-A55 | ARMv8.2 | DotProd |
| cortex-a57 | Cortex-A57 | ARMv8.0 | 标准性能核 |
| cortex-a72 | Cortex-A72 | ARMv8.0 | 高性能核 |
| cortex-a76 | Cortex-A76 | ARMv8.2 | DotProd, FP16 |
| cortex-a78ae | Cortex-A78AE | ARMv8.2 | 安全增强 |
| cortex-a710 | Cortex-A710 | ARMv9.0 | SVE2, MTE |
| neoverse-n1 | Neoverse N1 | ARMv8.2 | 服务器级 |
| neoverse-v1 | Neoverse V1 | ARMv8.4 | SVE |
| neoverse-n2 | Neoverse N2 | ARMv9.0 | SVE2, MTE3 |
| a64fx | Fujitsu A64FX | ARMv8.2 | SVE (512位) |
| max | — | 最新 | 所有 TCG 支持的特性 |
| host | — | 宿主 | KVM/HVF 透传 |

---

## 15. Cortex-A57 模型详解

`aarch64_a57_initfn()`（`cpu64.c:689-749`）是最常用的基础 CPU 模型，也是 `-cpu max` 在 TCG 模式下的起点：

```c
// cpu64.c:689-749 — Cortex-A57 初始化（关键摘录）
static void aarch64_a57_initfn(Object *obj)
{
    ARMCPU *cpu = ARM_CPU(obj);
    ARMISARegisters *isar = &cpu->isar;

    // 旧特性标志系统
    set_feature(&cpu->env, ARM_FEATURE_V8);
    set_feature(&cpu->env, ARM_FEATURE_NEON);
    set_feature(&cpu->env, ARM_FEATURE_AARCH64);
    set_feature(&cpu->env, ARM_FEATURE_EL2);
    set_feature(&cpu->env, ARM_FEATURE_EL3);
    set_feature(&cpu->env, ARM_FEATURE_PMU);

    cpu->midr = 0x411fd070;  // 实现者=ARM(0x41), 变体=1, 部件号=D07

    // AArch64 ID 寄存器设置
    SET_IDREG(isar, ID_AA64PFR0,  0x00002222);  // EL0-3 均为 AArch64
    SET_IDREG(isar, ID_AA64DFR0,  0x10305106);  // Debug v8, PMU v3
    SET_IDREG(isar, ID_AA64ISAR0, 0x00011120);  // AES+PMULL, SHA1, SHA256, CRC32
    SET_IDREG(isar, ID_AA64MMFR0, 0x00001124);  // 44位 PA, 16位 ASID, 4K/64K 页

    // 缓存配置
    cpu->ccsidr[0] = make_ccsidr(..., 32 * KiB, ...);  // 32KB L1 数据缓存
    cpu->ccsidr[1] = make_ccsidr(..., 48 * KiB, ...);  // 48KB L1 指令缓存
    cpu->ccsidr[2] = make_ccsidr(..., 2 * MiB, ...);   // 2MB L2 统一缓存
}
```

解读 `ID_AA64ISAR0 = 0x00011120`：

```
[63:60] RNDR  = 0  → 无随机数指令
[59:56] TLB   = 0  → 无 TLBI Range
[55:52] TS    = 0  → 无 FlagM
[51:48] FHM   = 0  → 无 FMLAL
[47:44] DP    = 0  → 无 DotProd
[23:20] ATOMIC= 0  → 无 LSE 原子操作
[19:16] CRC32 = 1  → CRC32 指令
[15:12] SHA2  = 1  → SHA256 指令
[11:8]  SHA1  = 1  → SHA1 指令
[7:4]   AES   = 2  → AES + PMULL 指令
```

---

## 16. "-cpu max" 类型：TCG 最大能力

`-cpu max` 是 QEMU 特殊的 CPU 类型，暴露 QEMU TCG 引擎支持的**所有**特性。

### 16.1 初始化流程

```c
// cpu64.c:878-893 — max 类型的双路径初始化
static void aarch64_max_initfn(Object *obj)
{
    if (kvm_enabled() || hvf_enabled() || whpx_enabled()) {
        // 硬件加速模式：max == host（透传宿主能力）
        aarch64_host_initfn(obj);
    } else {
        // TCG 模式：以 A57 为基础，叠加所有扩展
        aarch64_a57_initfn(obj);
        aarch64_max_tcg_initfn(obj);
    }
}
```

### 16.2 TCG max 叠加的特性

`aarch64_max_tcg_initfn()`（`tcg/cpu64.c:1163-1409`）在 A57 基础上设置大量 ID 寄存器字段：

```c
// tcg/cpu64.c:1230-1260 — ID_AA64ISAR0 最大能力设置
t = GET_IDREG(isar, ID_AA64ISAR0);
t = FIELD_DP64(t, ID_AA64ISAR0, AES, 2);      // FEAT_PMULL
t = FIELD_DP64(t, ID_AA64ISAR0, SHA1, 1);     // FEAT_SHA1
t = FIELD_DP64(t, ID_AA64ISAR0, SHA2, 2);     // FEAT_SHA512
t = FIELD_DP64(t, ID_AA64ISAR0, CRC32, 1);    // FEAT_CRC32
t = FIELD_DP64(t, ID_AA64ISAR0, ATOMIC, 3);   // FEAT_LSE + LSE128
t = FIELD_DP64(t, ID_AA64ISAR0, RDM, 1);      // FEAT_RDM
t = FIELD_DP64(t, ID_AA64ISAR0, SHA3, 1);     // FEAT_SHA3
t = FIELD_DP64(t, ID_AA64ISAR0, SM3, 1);      // FEAT_SM3（国密）
t = FIELD_DP64(t, ID_AA64ISAR0, SM4, 1);      // FEAT_SM4（国密）
t = FIELD_DP64(t, ID_AA64ISAR0, DP, 1);       // FEAT_DotProd
t = FIELD_DP64(t, ID_AA64ISAR0, FHM, 1);      // FEAT_FHM
t = FIELD_DP64(t, ID_AA64ISAR0, TS, 2);       // FEAT_FlagM2
t = FIELD_DP64(t, ID_AA64ISAR0, TLB, 2);      // FEAT_TLBIRANGE
t = FIELD_DP64(t, ID_AA64ISAR0, RNDR, 1);     // FEAT_RNG
SET_IDREG(isar, ID_AA64ISAR0, t);
```

### 16.3 max 类型的特殊处理

```c
// tcg/cpu64.c:1193-1210 — 重设 MIDR，避免 Guest 误认为真实 CPU
t = FIELD_DP64(0, MIDR_EL1, IMPLEMENTER, 0);    // 0 = 软件保留
t = FIELD_DP64(t, MIDR_EL1, ARCHITECTURE, 0xf); // 0xf = 查 ID 寄存器
t = FIELD_DP64(t, MIDR_EL1, PARTNUM, 'Q');      // 'Q' = QEMU
cpu->midr = t;
```

---

## 17. CPU 特性 QOM 属性

QEMU 通过 QOM 属性系统允许用户在命令行控制 CPU 特性：

```bash
# 典型用法
-cpu max,sve=off,pauth=on,sve-max-vq=4
```

### 17.1 属性注册

```c
// cpu.c:487-524 — SVE 属性注册
static void aarch64_add_sve_properties(Object *obj)
{
    // sve=on/off（总开关）
    object_property_add_bool(obj, "sve", cpu_arm_get_sve, cpu_arm_set_sve);
    // sve<N>=on/off（各个向量长度开关，N=128,256,...,2048）
    for (vq = 1; vq <= ARM_MAX_VQ; vq++) {
        name = g_strdup_printf("sve%d", vq * 128);
        object_property_add_bool(obj, name, ...);
    }
    // sve-max-vq=N（最大向量长度）
    // sve-default-vector-length=N（默认向量长度）
}

// cpu.c:526-549 — SME 属性注册
static void aarch64_add_sme_properties(Object *obj) { ... }

// cpu.c:646-667 — PAuth 属性注册
static void aarch64_add_pauth_properties(Object *obj) { ... }
```

### 17.2 属性如何修改 ID 寄存器

```c
// cpu.c:333-337 — sve=off 处理
static void cpu_arm_set_sve(Object *obj, bool value, Error **errp)
{
    ARMCPU *cpu = ARM_CPU(obj);
    if (!value) {
        // 清除 ID_AA64PFR0.SVE 字段
        FIELD_DP64_IDREG(&cpu->isar, ID_AA64PFR0, SVE, 0);
    }
}
```

### 17.3 可用属性一览

| 属性名 | 类型 | 影响的 ID 寄存器字段 |
|--------|------|---------------------|
| `sve` | bool | ID_AA64PFR0.SVE |
| `sve<N>` | bool | SVE 向量长度位图 |
| `sve-max-vq` | int | SVE 最大向量长度 |
| `sme` | bool | ID_AA64PFR1.SME |
| `sme<N>` | bool | SME 流向量长度位图 |
| `pauth` | bool | ID_AA64ISAR1.APA/API, ID_AA64ISAR2.APA3 |
| `pauth-impdef` | bool | 选择实现定义算法 |
| `pauth-qarma3` | bool | 选择 QARMA3 算法 |
| `x-rme` | bool | ID_AA64PFR0.RME |
| `lpa2` | bool | 52 位地址支持 |

---

## 18. 特性终化流程：arm_cpu_finalize_features

在 CPU 实现（realize）前，需要对各特性进行终化处理，确保依赖关系正确：

```c
// cpu.c:1680-1717 — 特性终化入口
void arm_cpu_finalize_features(ARMCPU *cpu, Error **errp)
{
    if (arm_feature(&cpu->env, ARM_FEATURE_AARCH64)) {
        arm_cpu_sve_finalize(cpu, &local_err);    // SVE 向量长度验证
        arm_cpu_sme_finalize(cpu, &local_err);    // SME 向量长度验证
        arm_cpu_pauth_finalize(cpu, &local_err);  // PAuth 算法选择
        arm_cpu_lpa2_finalize(cpu, &local_err);   // LPA2 52 位地址验证
    }
    if (kvm_enabled()) {
        kvm_arm_steal_time_finalize(cpu, &local_err);  // KVM 时间窃取
    }
}
```

### 18.1 SVE 终化

`arm_cpu_sve_finalize()`（`cpu64.c:63-260`）处理：
- 验证用户请求的向量长度是否在 CPU 模型支持范围内
- KVM 模式下验证宿主是否支持请求的向量长度
- 生成最终的 `sve_vq` 位图
- 如果关闭 SVE，清除 `ID_AA64PFR0.SVE` 和 `ID_AA64ZFR0`

### 18.2 PAuth 终化

`arm_cpu_pauth_finalize()`（`cpu.c:551-635`）处理：
- 用户指定 `pauth=off` → 清除所有 PAuth 字段
- 用户指定 `pauth-impdef` → 设置 `ID_AA64ISAR1.API/GPI`
- 用户指定 `pauth-qarma3` → 设置 `ID_AA64ISAR2.APA3/GPA3`
- 默认 → 设置 `ID_AA64ISAR1.APA/GPA`（QARMA5 算法）

---

## 19. arm_cpu_realizefn：CPU 实现函数

`arm_cpu_realizefn()`（`cpu.c:1740-1901`）是 CPU 对象最终实例化的入口：

```c
// cpu.c:1740+ — CPU 实现流程（简化）
static void arm_cpu_realizefn(DeviceState *dev, Error **errp)
{
    ARMCPU *cpu = ARM_CPU(dev);

    // 1. 传播特性隐含关系
    arm_cpu_propagate_feature_implications(cpu);
    // 例: V8 → V7VE, V7 → VAPA + THUMB2, EL3 → TrustZone

    // 2. 特性终化（SVE/SME/PAuth/LPA2）
    arm_cpu_finalize_features(cpu, &local_err);

    // 3. 对纯 AArch32 CPU 清除 AArch64 ID 寄存器
    if (!arm_feature(&cpu->env, ARM_FEATURE_AARCH64)) {
        arm_clear_aarch64_idregs(cpu);
    }

    // 4. 验证 VFP/NEON 一致性
    // 5. 注册系统寄存器
    // 6. 初始化 hflags
}
```

### 19.1 特性隐含传播

```c
// cpu.c:1388-1478 — 特性依赖关系传播
static void arm_cpu_propagate_feature_implications(ARMCPU *cpu)
{
    // V8 自动包含 V7VE, V7, V6K, V6, V5, V4T
    if (arm_feature(env, ARM_FEATURE_V8)) {
        set_feature(env, ARM_FEATURE_V7VE);
    }
    if (arm_feature(env, ARM_FEATURE_V7)) {
        set_feature(env, ARM_FEATURE_VAPA);
        set_feature(env, ARM_FEATURE_THUMB2);
        set_feature(env, ARM_FEATURE_MPIDR);
    }
    // ... 更多隐含关系
}
```

---

## 20. Guest 读取 ID 寄存器：系统寄存器访问

Guest 通过 `MRS Xd, <ID_REG>` 指令读取 ID 寄存器。QEMU 通过 `ARMCPRegInfo` 结构注册这些寄存器的访问处理：

```c
// helper.c:6392-6460 — v8 ID 寄存器注册（摘录）
ARMCPRegInfo v8_idregs[] = {
    { .name = "ID_AA64PFR0_EL1",
      .state = ARM_CP_STATE_AA64,
      .opc0 = 3, .opc1 = 0, .crn = 0, .crm = 4, .opc2 = 0,
      .access = PL1_R,            // EL1 可读
#ifdef CONFIG_USER_ONLY
      .type = ARM_CP_CONST,       // 用户模式：常量
      .resetvalue = GET_IDREG(isar, ID_AA64PFR0)
#else
      .type = ARM_CP_NO_RAW,      // 系统模式：动态读取
      .accessfn = access_tid3,    // TID3 陷阱检查
      .readfn = id_aa64pfr0_read, // 特殊读函数（动态调整 GIC 字段）
      .writefn = arm_cp_write_ignore
#endif
    },

    { .name = "ID_AA64PFR1_EL1",
      .opc0 = 3, .opc1 = 0, .crn = 0, .crm = 4, .opc2 = 1,
      .access = PL1_R, .type = ARM_CP_CONST,
      .accessfn = access_tid3,
      .resetvalue = GET_IDREG(isar, ID_AA64PFR1) },

    { .name = "ID_AA64ISAR0_EL1",
      .opc0 = 3, .opc1 = 0, .crn = 0, .crm = 6, .opc2 = 0,
      .access = PL1_R, .type = ARM_CP_CONST,
      .accessfn = access_tid3,
      .resetvalue = GET_IDREG(isar, ID_AA64ISAR0) },

    // ... 更多 ID 寄存器
    // 未实现的保留寄存器 → resetvalue = 0（RAZ）
    { .name = "ID_AA64PFR3_EL1_RESERVED",
      .opc0 = 3, .opc1 = 0, .crn = 0, .crm = 4, .opc2 = 3,
      .access = PL1_R, .type = ARM_CP_CONST,
      .accessfn = access_tid3,
      .resetvalue = 0 },  // Read-As-Zero
};
```

### 20.1 关键设计点

- **PL1_R**：仅 EL1+ 可读（EL0 读取会产生未定义异常）
- **ARM_CP_CONST**：大多数 ID 寄存器为常量，值来自 `GET_IDREG()`
- **access_tid3**：所有 ID 寄存器统一经过 TID3 陷阱检查
- **RAZ**：未实现的寄存器返回零（`resetvalue = 0`）
- **id_aa64pfr0_read**：PFR0 需要特殊处理（GIC 字段在运行时确定）

---

## 21. ID 寄存器 Trap：FEAT_IDST 与 TID3

### 21.1 access_tid3 检查

```c
// helper.c:5784-5800 — TID3 陷阱检查
static CPAccessResult access_tid3(CPUARMState *env, const ARMCPRegInfo *ri,
                                  bool isread)
{
    // 对 v8+ CPU，TID3 空间的寄存器必须可被 trap
    // QEMU 选择始终启用 trap 检查（符合 FEAT_FGT 要求）
    if (arm_feature(env, ARM_FEATURE_V8)) {
        return access_v7a_tid3(env, ri, isread);
    }
    return CP_ACCESS_OK;
}
```

### 21.2 Trap 到 EL2

当 Hypervisor 设置 `HCR_EL2.TID3 = 1` 时，Guest（EL1）读取 ID 寄存器会被陷入 EL2，让 Hypervisor 控制 Guest 看到的 CPU 特性。

### 21.3 FEAT_IDST

`ID_AA64MMFR2.IDS` 字段控制 FEAT_IDST（ID Space Trap），允许 EL2 在不使用 HCR_EL2.TID3 的情况下，通过细粒度陷阱控制单个 ID 寄存器的访问。

`-cpu max` 设置 `IDS = 1`，启用此特性。

---

## 22. KVM 模式：宿主特性发现

KVM 模式下，QEMU 需要从宿主 CPU 读取真实 ID 寄存器值：

```c
// kvm.c:276-420 — 宿主 CPU 特性发现
static void kvm_arm_get_host_cpu_features(ARMHostCPUFeatures *ahcf)
{
    // 1. 创建临时 VM 和 vCPU
    int fdarray[3];
    // ... 创建 scratch KVM VM

    // 2. 探测 KVM 能力
    bool sve_supported = kvm_check_extension(KVM_ARM_VCPU_SVE);
    bool el2_supported = kvm_check_extension(KVM_ARM_VCPU_HAS_EL2);

    // 3. 通过 KVM_GET_ONE_REG 读取宿主 ID 寄存器
    // 读取所有 AArch64 ID 寄存器：
    //   ID_AA64PFR0/1/2
    //   ID_AA64DFR0/1
    //   ID_AA64ISAR0/1/2
    //   ID_AA64MMFR0/1/2/3
    //   ID_AA64SMFR0
    //   ID_AA64ZFR0（仅在 SVE 可用时）
    //
    // 也读取 AArch32 ID 寄存器：
    //   ID_PFR0/1/2, ID_DFR0/1, ID_MMFR0-5, ID_ISAR0-6

    // 4. 从 ID 寄存器合成 DBGDIDR（不能直接读取）
    // kvm.c:408-436

    // 5. 销毁临时 VM
}
```

### 22.1 KVM vs TCG 对比

| 方面 | TCG | KVM |
|------|-----|-----|
| ID 寄存器来源 | CPU 模型 initfn 合成 | 宿主 CPU 真实值 |
| 特性范围 | 可超过宿主能力 | 严格限于宿主能力 |
| SVE 向量长度 | 软件定义（最大 2048位） | 受宿主硬件限制 |
| `-cpu max` 含义 | QEMU 支持的所有特性 | 等同于 `-cpu host` |
| 性能 | 软件模拟 | 硬件执行 |

---

## 23. 翻译时特性检查：dc_isar_feature()

TCG 翻译器在将 Guest 指令翻译为 TCG IR 时，需要检查当前 CPU 是否支持某条指令：

```c
// translate.h:651-652 — 翻译上下文中的特性检查
#define dc_isar_feature(name, ctx) \
    ({ const DisasContext *ctx_ = (ctx); isar_feature_##name(ctx_->isar); })
```

### 23.1 DisasContext 中的 ISAR 指针

```c
// translate.h:39-43 — DisasContext 保存 ISAR 引用
typedef struct DisasContext {
    ...
    const ARMISARegisters *isar;  // 指向 ARMCPU.isar
    ...
} DisasContext;
```

### 23.2 使用示例

```c
// translate-a64.c — CAS 指令翻译（需要 FEAT_LSE）
static bool trans_CAS(DisasContext *s, arg_CAS *a)
{
    if (!dc_isar_feature(aa64_lse, s)) {
        return false;  // 特性不可用 → 视为未分配编码 → UNDEF
    }
    // ... 生成 CAS 的 TCG IR
}
```

### 23.3 TRANS_FEAT 宏

QEMU 定义了便捷宏来简化特性门控的指令翻译：

```c
// translate.h:864-866 — 特性门控翻译宏
#define TRANS_FEAT(NAME, FEAT, FUNC, ...) \
    static bool trans_##NAME(DisasContext *s, arg_##NAME *a) \
    { return dc_isar_feature(FEAT, s) && FUNC(s, __VA_ARGS__); }
```

使用示例：
```c
// translate-a64.c — MTE 指令门控
TRANS_FEAT(ADDG_i, aa64_mte_insn_reg, gen_gvec_fn2i, ...)
TRANS_FEAT(IRG, aa64_mte_insn_reg, gen_helper_irg, ...)
```

---

## 24. TB Flags 中的特性编码

某些特性状态会被编码到翻译块（TB）的标志中，以便在翻译时快速访问：

```c
// hflags.c:240-310 — rebuild_hflags_a64() 构建 TB 标志（摘录）
static CPUARMTBFlags rebuild_hflags_a64(CPUARMState *env, int el, int fp_el,
                                        ARMMMUIdx mmu_idx)
{
    CPUARMTBFlags flags = {};

    DP_TBFLAG_ANY(flags, AARCH64_STATE, 1);  // 标记 AArch64 状态

    // 地址标记位
    DP_TBFLAG_A64(flags, TBII, tbii);  // TBI 用于指令地址
    DP_TBFLAG_A64(flags, TBID, tbid);  // TBI 用于数据地址

    // VHE/NV2 控制
    if (hcr & HCR_E2H) {
        DP_TBFLAG_A64(flags, E2H, 1);
    }

    // SVE 状态
    if (cpu_isar_feature(aa64_sve, env_archcpu(env))) {
        int sve_el = sve_exception_el(env, el);
        DP_TBFLAG_A64(flags, SVEEXC_EL, sve_el);
        DP_TBFLAG_A64(flags, VL, sve_vqm1_for_el(env, el));
    }

    // SME 状态
    if (cpu_isar_feature(aa64_sme, env_archcpu(env))) {
        DP_TBFLAG_A64(flags, SMEEXC_EL, sme_el);
        DP_TBFLAG_A64(flags, PSTATE_SM, sm);
        DP_TBFLAG_A64(flags, PSTATE_ZA, za);
    }

    // PAuth 活动状态
    DP_TBFLAG_A64(flags, PAUTH_ACTIVE, pauth_active);

    // BTI 状态
    DP_TBFLAG_A64(flags, BT, bt);

    // MTE 状态
    DP_TBFLAG_A64(flags, ATA, ata);
    DP_TBFLAG_A64(flags, MTE_ACTIVE, mte_active);
    DP_TBFLAG_A64(flags, MTE0_ACTIVE, mte0_active);
}
```

### 24.1 TB Flags vs dc_isar_feature 区别

| 检查方式 | 时机 | 变化频率 | 示例 |
|---------|------|---------|------|
| **dc_isar_feature()** | 翻译时 | 不变（CPU 生命周期内固定） | LSE 指令是否存在 |
| **TB Flags** | 翻译时 | 随运行状态变化 | MTE 是否当前激活 |

关键区别：
- **ID 寄存器**决定指令是否**存在**（架构能力）
- **TB Flags**决定指令在当前上下文是否**可用**（运行时状态）

例如：MTE 指令的 `dc_isar_feature(aa64_mte, s)` 检查 CPU 是否**支持** MTE，而 `MTE_ACTIVE` TB flag 检查 MTE 是否在当前内存区域**启用**。

---

## 25. FEAT_LSE 完整实现追踪

以 FEAT_LSE（Large System Extensions，原子操作指令集）为例，追踪从 ID 寄存器到指令执行的完整路径：

### 25.1 ID 寄存器定义

```
ID_AA64ISAR0_EL1.ATOMIC[23:20]:
  0 → 不支持
  2 → FEAT_LSE（CAS/CASP/LDADD/LDCLR/LDEOR/LDSET/SWP 等）
  3 → FEAT_LSE128（128 位原子操作）
```

### 25.2 CPU 模型设置

```c
// cpu64.c (A57): ID_AA64ISAR0 = 0x00011120  → ATOMIC = 0（不支持）
// tcg/cpu64.c (max): FIELD_DP64(t, ID_AA64ISAR0, ATOMIC, 3)  → 支持 LSE+LSE128
```

### 25.3 特性检查

```c
// cpu-features.h:837-840
static inline bool isar_feature_aa64_lse(const ARMISARegisters *id)
{
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, ATOMIC) >= 2;
}
```

### 25.4 翻译时门控

```c
// translate-a64.c — CAS 指令翻译
static bool trans_CAS(DisasContext *s, arg_CAS *a)
{
    if (!dc_isar_feature(aa64_lse, s)) {
        return false;  // → UNDEF
    }
    // 生成 TCG IR：比较并交换
    TCGv_i64 addr = ...;
    gen_helper_cas_le(tcg_env, addr, ...);
    return true;
}
```

### 25.5 Guest 感知

```c
// Guest Linux 启动时：
// MRS X0, ID_AA64ISAR0_EL1  → 读取 ISAR0
// UBFX X0, X0, #20, #4      → 提取 ATOMIC 字段
// CMP X0, #2                 → 检查是否 >= 2
// B.LT no_lse               → 不支持则跳过
// ... 使用 CAS/LDADD 等原子指令
```

---

## 26. FEAT_PAuth 完整实现追踪

### 26.1 ID 寄存器

涉及三个字段（互斥，只有一个非零）：
- `ID_AA64ISAR1.APA` — QARMA5 算法
- `ID_AA64ISAR1.API` — 实现定义算法
- `ID_AA64ISAR2.APA3` — QARMA3 算法

### 26.2 特性检查

```c
// cpu-features.h:931+ — 复合检查
static inline bool isar_feature_aa64_pauth(const ARMISARegisters *id)
{
    return isar_feature_pauth_feature(id) != PauthFeat_None;
}
```

### 26.3 QOM 属性

```bash
-cpu max,pauth=on           # 启用 PAuth（默认 QARMA5）
-cpu max,pauth-impdef=on    # 使用实现定义算法
-cpu max,pauth-qarma3=on    # 使用 QARMA3 算法
```

### 26.4 TB Flags

```c
// hflags.c — PAUTH_ACTIVE 标志
DP_TBFLAG_A64(flags, PAUTH_ACTIVE, pauth_active);
```

### 26.5 翻译时处理

```c
// translate-a64.c — PACIA 指令翻译
static bool trans_PACIAZ(DisasContext *s, arg_PACIAZ *a)
{
    if (s->pauth_active) {
        gen_helper_pacia(cpu_X[30], tcg_env, cpu_X[30], tcg_constant_i64(0));
    }
    return true;
}
```

---

## 27. FEAT_MTE 完整实现追踪

### 27.1 ID 寄存器

```
ID_AA64PFR1_EL1.MTE[11:8]:
  0 → 不支持
  1 → FEAT_MTE（仅指令，无检查）
  2 → FEAT_MTE2（完整内存标记）
  3 → FEAT_MTE3（非对称标记检查）
```

### 27.2 特性检查层次

```c
// cpu-features.h:1150+
isar_feature_aa64_mte()       // MTE >= 2（完整功能）
isar_feature_aa64_mte3()      // MTE >= 3（非对称检查）
isar_feature_aa64_mte_insn_reg() // MTE >= 1（仅指令可用）
```

### 27.3 TB Flags

```c
// hflags.c — MTE 运行时状态
DP_TBFLAG_A64(flags, ATA, ata);             // 分配标记访问使能
DP_TBFLAG_A64(flags, MTE_ACTIVE, mte);      // EL1+ MTE 检查激活
DP_TBFLAG_A64(flags, MTE0_ACTIVE, mte0);    // EL0 MTE 检查激活
DP_TBFLAG_A64(flags, TCMA, tcma);           // 标记检查不匹配时的行为
```

### 27.4 翻译时门控

```c
// translate-a64.c — ADDG 指令（MTE 标记操作）
TRANS_FEAT(ADDG_i, aa64_mte_insn_reg, gen_addg_i, a)
// 仅在 aa64_mte_insn_reg (MTE >= 1) 时可用

// 内存访问中的标记检查
// 在 MTE_ACTIVE 标志为真时，load/store 辅助函数会执行标记比较
```

---

## 28. FEAT_BTI 完整实现追踪

### 28.1 ID 寄存器

```
ID_AA64PFR1_EL1.BT[3:0]:
  0 → 不支持
  1 → FEAT_BTI（分支目标标识）
```

### 28.2 特性检查

```c
// cpu-features.h:1140-1143
static inline bool isar_feature_aa64_bti(const ARMISARegisters *id)
{
    return FIELD_EX64_IDREG(id, ID_AA64PFR1, BT) != 0;
}
```

### 28.3 TB Flags

```c
// hflags.c — BTI 状态
DP_TBFLAG_A64(flags, BT, bt);  // PSTATE.BTYPE 当前值
```

### 28.4 翻译时处理

```c
// translate-a64.c — 分支指令更新 BTYPE
// BR Xn 执行后设置 BTYPE
set_btype_for_br(s);   // 检查 dc_isar_feature(aa64_bti, s)

// BTI 指令本身验证 BTYPE
// 如果 BTYPE 不匹配着陆模式 → Branch Target Exception
```

---

## 29. 旧特性标志系统：ARM_FEATURE_*

QEMU 同时维护一个旧的特性标志系统 `ARM_FEATURE_*`（位图），与 ID 寄存器系统并存：

```c
// cpu.h — ARM_FEATURE_* 枚举（部分）
enum arm_features {
    ARM_FEATURE_V4T,
    ARM_FEATURE_V5,
    ARM_FEATURE_V6,
    ARM_FEATURE_V7,
    ARM_FEATURE_V8,
    ARM_FEATURE_THUMB2,
    ARM_FEATURE_NEON,
    ARM_FEATURE_AARCH64,    // 支持 AArch64
    ARM_FEATURE_EL2,        // 支持 Hypervisor EL2
    ARM_FEATURE_EL3,        // 支持 Secure Monitor EL3
    ARM_FEATURE_PMU,        // 性能监控单元
    ARM_FEATURE_GENERIC_TIMER,
    ARM_FEATURE_CBAR_RO,
    ...
};
```

### 29.1 操作函数

```c
static void set_feature(CPUARMState *env, int feature);
static void unset_feature(CPUARMState *env, int feature);
static bool arm_feature(CPUARMState *env, int feature);
```

### 29.2 新旧系统关系

| 旧系统 (ARM_FEATURE_*) | 新系统 (isar_feature_*) |
|------------------------|------------------------|
| 粗粒度（整个特性组） | 细粒度（单个 FEAT_xxx） |
| 位图存储（env->features） | ID 寄存器字段提取 |
| 控制大的行为开关 | 控制单条指令可用性 |
| 仍用于 EL2/EL3/PMU 等 | 用于所有 ARMv8+ 扩展 |

迁移趋势：AArch64 特性越来越多地从 `ARM_FEATURE_*` 迁移到 `isar_feature_*`，但 `ARM_FEATURE_AARCH64`、`ARM_FEATURE_EL2`、`ARM_FEATURE_EL3` 等基础标志仍在广泛使用。

---

## 30. 迁移兼容性

### 30.1 ID 寄存器的 VMState

CPU 迁移时，ID 寄存器作为 CPU 状态的一部分传输。主要机制：

- `target/arm/machine.c` — VMState 定义
- `cpreg_vmstate_*` — 协处理器寄存器迁移数组
- `vmstate_vfp` — 浮点状态迁移

### 30.2 兼容性问题

```c
// machine.c:25-46 — FPSCR/FPCR/FPSR 布局兼容处理
// 旧版 QEMU 使用 FPSCR（AArch32 风格）
// 新版 QEMU 使用 FPCR + FPSR（AArch64 风格）
// 迁移时需要正确转换
```

### 30.3 KVM 迁移容差

```c
// cpu64.c:813-848 — KVM 寄存器迁移容差注册
arm_register_cpreg_mig_tolerance(...)
// 某些寄存器在不同 CPU 版本间有差异，迁移时需要容忍
```

---

## 31. 完整数据流总结

```
┌───────────────────────────────────────────────────────────────────────┐
│                      CPU 模型初始化阶段                              │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. CPU model initfn()              例: aarch64_a57_initfn()         │
│     ├── set_feature(ARM_FEATURE_*)  旧特性标志                       │
│     └── SET_IDREG(isar, ID_AA64*, value)  写入 idregs[]              │
│                                                                       │
│  2. QOM 属性应用                    例: -cpu max,sve=off             │
│     └── FIELD_DP64_IDREG()          修改 idregs[] 中的字段           │
│                                                                       │
│  3. arm_cpu_finalize_features()     SVE/SME/PAuth/LPA2 终化          │
│     └── 验证约束，确定最终 ID 值                                     │
│                                                                       │
│  4. arm_cpu_realizefn()             CPU 实例化完成                    │
│     ├── 传播特性隐含关系                                             │
│     └── 注册系统寄存器 (ARMCPRegInfo)                                │
│                                                                       │
├───────────────────────────────────────────────────────────────────────┤
│                      运行时阶段                                      │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Guest MRS 路径:                                                      │
│    MRS Xd, ID_AA64ISAR0_EL1                                          │
│      → access_tid3() 检查 → GET_IDREG(isar, ID_AA64ISAR0) → 返回值  │
│                                                                       │
│  翻译路径:                                                            │
│    Guest CAS 指令                                                     │
│      → dc_isar_feature(aa64_lse, s)                                   │
│      → isar_feature_aa64_lse(&cpu->isar)                              │
│      → FIELD_EX64_IDREG(id, ID_AA64ISAR0, ATOMIC) >= 2               │
│      → true: 生成 TCG IR  /  false: 返回 UNDEF                       │
│                                                                       │
│  TB Flags 路径:                                                       │
│    rebuild_hflags_a64()                                               │
│      → cpu_isar_feature(aa64_sve, cpu) → 设置 SVEEXC_EL/VL 标志      │
│      → cpu_isar_feature(aa64_sme, cpu) → 设置 SMEEXC_EL/SM/ZA 标志   │
│      → 编码到 TB flags → 影响 TB 缓存键                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 32. 总结

### 32.1 核心设计模式

| 模式 | 说明 |
|------|------|
| **统一存储** | 所有 ID 寄存器统一存储在 `idregs[]` 数组中 |
| **声明式字段** | `FIELD()` 宏声明字段布局，`FIELD_EX64_IDREG()` 提取 |
| **谓词函数** | 168 个 `isar_feature_*()` 函数封装特性检查逻辑 |
| **模型初始化** | 每个 CPU 模型的 `initfn()` 设置对应的 ID 寄存器值 |
| **属性覆盖** | QOM 属性允许命令行微调特性开关 |
| **终化验证** | `finalize_features()` 确保特性组合有效 |
| **双路径读取** | Guest 通过 MRS 读取，翻译器通过 `dc_isar_feature()` 检查 |

### 32.2 关键源码文件

| 文件 | 作用 |
|------|------|
| `target/arm/cpu.h:885-914` | FIELD_EX64_IDREG / SET_IDREG / GET_IDREG 宏 |
| `target/arm/cpu.h:1085-1094` | ARMISARegisters 结构体 |
| `target/arm/cpu-sysregs.h.inc` | 43 个 ID 寄存器索引定义 |
| `target/arm/cpu-features.h:198-402` | ID 寄存器字段声明（FIELD 宏） |
| `target/arm/cpu-features.h:807-1140+` | 168 个 `isar_feature_*()` 检查函数 |
| `target/arm/cpu-features.h:1696` | `cpu_isar_feature()` 宏 |
| `target/arm/cpu64.c:689-749` | Cortex-A57 模型初始化 |
| `target/arm/cpu64.c:878-893` | `-cpu max` 初始化逻辑 |
| `target/arm/tcg/cpu64.c:1163-1409` | TCG max 完整特性设置 |
| `target/arm/cpu.c:1388-1478` | 特性隐含关系传播 |
| `target/arm/cpu.c:1680-1717` | 特性终化流程 |
| `target/arm/cpu.c:1740-1901` | CPU 实现函数 |
| `target/arm/helper.c:6392-6759` | ID 寄存器系统寄存器注册 |
| `target/arm/helper.c:5784-5800` | access_tid3 陷阱检查 |
| `target/arm/kvm.c:276-420` | KVM 宿主特性发现 |
| `target/arm/tcg/translate.h:651` | `dc_isar_feature()` 宏 |
| `target/arm/tcg/hflags.c:240-310` | TB Flags 中的特性编码 |
| `target/arm/machine.c` | 迁移兼容性处理 |

---

## 附录 A：AArch64 可用 CPU 模型完整列表

| 模型名 | initfn 位置 | 说明 |
|--------|------------|------|
| cortex-a35 | tcg/cpu64.c:32 | ARMv8.0 低功耗 |
| cortex-a53 | cpu64.c:751 | ARMv8.0 能效核 |
| cortex-a55 | tcg/cpu64.c:203 | ARMv8.2 能效核 |
| cortex-a57 | cpu64.c:689 | ARMv8.0 性能核 |
| cortex-a72 | tcg/cpu64.c:276 | ARMv8.0 高性能 |
| cortex-a76 | tcg/cpu64.c:336 | ARMv8.2 高性能 |
| cortex-a78ae | tcg/cpu64.c:410 | ARMv8.2 安全增强 |
| cortex-a710 | tcg/cpu64.c:960 | ARMv9.0 SVE2 |
| a64fx | tcg/cpu64.c:483 | ARMv8.2 SVE 512 位 |
| neoverse-n1 | tcg/cpu64.c:657 | ARMv8.2 服务器 |
| neoverse-v1 | tcg/cpu64.c:733 | ARMv8.4 SVE |
| neoverse-n2 | tcg/cpu64.c:1062 | ARMv9.0 SVE2 |
| max | cpu64.c:878 | QEMU 最大能力 |
| host | cpu64.c:851 | KVM/HVF 透传 |

## 附录 B：ID_AA64ISAR0 完整字段编码表

| 位域 | 字段 | 值: 含义 |
|------|------|----------|
| [7:4] | AES | 0: 无, 1: AES, 2: +PMULL |
| [11:8] | SHA1 | 0: 无, 1: SHA1 |
| [15:12] | SHA2 | 0: 无, 1: SHA256, 2: +SHA512 |
| [19:16] | CRC32 | 0: 无, 1: CRC32 |
| [23:20] | ATOMIC | 0: 无, 2: LSE, 3: LSE128 |
| [27:24] | TME | 0: 无, 1: TME |
| [31:28] | RDM | 0: 无, 1: SQRDML |
| [35:32] | SHA3 | 0: 无, 1: SHA3 |
| [39:36] | SM3 | 0: 无, 1: SM3 |
| [43:40] | SM4 | 0: 无, 1: SM4 |
| [47:44] | DP | 0: 无, 1: DotProd |
| [51:48] | FHM | 0: 无, 1: FMLAL |
| [55:52] | TS | 0: 无, 1: FlagM, 2: FlagM2 |
| [59:56] | TLB | 0: 无, 1: TLBIOS, 2: +Range |
| [63:60] | RNDR | 0: 无, 1: RNG |

## 附录 C：特性检查调用关系图

```
-cpu max,sve=off
    │
    ├─→ aarch64_a57_initfn()           SET_IDREG(ID_AA64PFR0, 0x00002222)
    │     └─→ SET_IDREG(ID_AA64ISAR0, 0x00011120)
    │
    ├─→ aarch64_max_tcg_initfn()       FIELD_DP64(ATOMIC, 3)  → LSE128
    │     └─→ FIELD_DP64(SVE, 1)      → SVE 启用
    │
    ├─→ cpu_arm_set_sve(false)          FIELD_DP64(SVE, 0)  → SVE 关闭
    │
    ├─→ arm_cpu_sve_finalize()          验证 SVE 已关闭
    │
    ├─→ arm_cpu_realizefn()             注册最终 ID 寄存器
    │     └─→ helper.c: v8_idregs[]    GET_IDREG(ID_AA64PFR0) = 无 SVE
    │
    └─→ Guest: MRS X0, ID_AA64PFR0_EL1
          └─→ SVE 字段 = 0 → Linux 不启用 SVE
```
