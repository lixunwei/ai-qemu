# ARM64 Fine-Grained Traps (FGT) 完整机制分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/74 |
| 分析对象 | FEAT_FGT (Fine-Grained Traps) 在 QEMU 中的完整实现 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §D1.14 (Fine-grained traps) |
| 关键寄存器 | HFGRTR_EL2, HFGWTR_EL2, HDFGRTR_EL2, HDFGWTR_EL2, HFGITR_EL2 |
| 核心结论 | **QEMU 使用优雅的 bit-encoding 设计完整实现 FGT，每个寄存器的 trap 位通过声明式 `.fgt` 字段自动生效** |

---

## 1. FGT 概述

### 1.1 什么是 Fine-Grained Traps

FGT (FEAT_FGT, Armv8.6) 允许 Hypervisor **逐个寄存器**控制 EL1/EL0 的访问是否 trap 到 EL2：

| 传统 Trap (粗粒度) | FGT (细粒度) |
|-------------------|-------------|
| HCR_EL2.TVM: trap **所有** EL1 虚存寄存器 | HFGRTR_EL2.TCR_EL1: 只 trap TCR_EL1 读 |
| HSTR_EL2: 按 coproc 编号 trap | HFGWTR_EL2.SCTLR_EL1: 只 trap SCTLR_EL1 写 |
| | HFGITR_EL2.TLBIVAE1: 只 trap 特定 TLBI |

### 1.2 FGT 寄存器分类

| 寄存器 | 功能 | 控制范围 |
|--------|------|---------|
| HFGRTR_EL2 | 系统寄存器**读** trap | EL1 功能寄存器 |
| HFGWTR_EL2 | 系统寄存器**写** trap | EL1 功能寄存器 |
| HDFGRTR_EL2 | 调试寄存器**读** trap | Debug/PMU 寄存器 |
| HDFGWTR_EL2 | 调试寄存器**写** trap | Debug/PMU 寄存器 |
| HFGITR_EL2 | 指令执行 trap | TLBI/cache/AT/SVC/ERET |

### 1.3 激活条件

```c
// target/arm/internals.h:1836
static inline bool arm_fgt_active(CPUARMState *env, int el)
{
    return cpu_isar_feature(aa64_fgt, env_archcpu(env))  // CPU 支持 FEAT_FGT
        && el < 2                                         // 当前在 EL0 或 EL1
        && arm_is_el2_enabled(env)                        // EL2 已使能
        && arm_el_is_aa64(env, 1)                         // EL1 是 AArch64
        && (arm_hcr_el2_eff(env) & (HCR_E2H | HCR_TGE))
           != (HCR_E2H | HCR_TGE)                        // 非 E2H+TGE 模式
        && (!arm_feature(env, ARM_FEATURE_EL3)
            || (env->cp15.scr_el3 & SCR_FGTEN));          // EL3 允许 FGT
}
```

---

## 2. QEMU 的 FGT 编码设计

### 2.1 FGTBit 枚举编码

QEMU 将每个 FGT trap bit 编码为一个 14-bit 整数：

```c
// target/arm/cpregs.h:686-691
FIELD(FGT, NXS,    13, 1)  // bit[13]: nXS 豁免标记
FIELD(FGT, TYPE,   10, 3)  // bit[12:10]: 类型 (R/W/EXEC)
FIELD(FGT, REV,     9, 1)  // bit[9]: 反转（0=trap, 1=not-trap）
FIELD(FGT, IDX,     6, 3)  // bit[8:6]: 数组索引
FIELD(FGT, BITPOS,  0, 6)  // bit[5:0]: 位位置 (0-63)
```

### 2.2 类型位含义

```c
FGT_R    = 1 << 10  // 读操作检查 fgt_read[]
FGT_W    = 2 << 10  // 写操作检查 fgt_write[]
FGT_EXEC = 4 << 10  // 执行操作检查 fgt_exec[]
FGT_RW   = FGT_R | FGT_W  // 读写都检查
```

### 2.3 寄存器数组映射

```c
// CPU 状态中的存储:
struct CPUARMState {
    struct {
        uint64_t fgt_read[2];   // [0]=HFGRTR_EL2, [1]=HDFGRTR_EL2
        uint64_t fgt_write[2];  // [0]=HFGWTR_EL2, [1]=HDFGWTR_EL2
        uint64_t fgt_exec[1];   // [0]=HFGITR_EL2
    } cp15;
};

// 寄存器组合编码:
FGT_HFGRTR  = FGT_RW   | (0 << 6)  // fgt_read[0] + fgt_write[0]
FGT_HFGWTR  = FGT_W    | (0 << 6)  // fgt_write[0] only
FGT_HDFGRTR = FGT_RW   | (1 << 6)  // fgt_read[1] + fgt_write[1]
FGT_HDFGWTR = FGT_W    | (1 << 6)  // fgt_write[1] only
FGT_HFGITR  = FGT_EXEC | (0 << 6)  // fgt_exec[0]
```

### 2.4 具体 bit 定义示例

```c
// DO_BIT 宏展开:
// FGT_TCR_EL1 = FGT_HFGRTR | R_HFGRTR_EL2_TCR_EL1_SHIFT
//             = (FGT_RW | 0<<6) | 34
//             = 0x0C00 | 34 = "type=RW, idx=0, bitpos=34"

// DO_REV_BIT 宏展开 (反转位):
// FGT_NTPIDR2_EL0 = FGT_HFGRTR | FGT_REV | 38
//                 = "type=RW, idx=0, bitpos=38, reversed"
// 含义: bit=0 时 trap, bit=1 时不 trap (默认 trap)
```

---

## 3. ARMCPRegInfo 中的 FGT 声明

### 3.1 声明方式

每个寄存器定义中用 `.fgt` 字段标记其 FGT trap bit：

```c
// 示例: TCR_EL1 的 FGT trap
{ .name = "TCR_EL1", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 0, .crn = 2, .crm = 0, .opc2 = 2,
  .access = PL1_RW,
  .accessfn = access_tvm_trvm,
  .fgt = FGT_TCR_EL1,           // ← FGT trap 声明
  ...
},
```

### 3.2 TLBI 指令的 FGT

```c
// TLBI 使用 FGT_EXEC 类型:
{ .name = "TLBI_VALE1IS", ...
  .fgt = FGT_TLBIVALE1IS,       // fgt_exec[0] bit X
},

// nXS 变体有特殊豁免:
{ .name = "TLBI_VALE1ISNXS", ...
  .fgt = FGT_TLBIVALE1ISNXS,   // 带 NXS 标记，受 HCRX.FGTnXS 控制
},
```

### 3.3 FGT 与其他 trap 的优先级

在 `access_check_cp_reg()` 中，FGT 检查的优先级：

```
1. UNDEF to EL1 (不存在的寄存器)     ← 最高
2. HSTR_EL2 (按 coproc 编号 trap)
3. Fine-Grained Traps (FGT)          ← 本文焦点
4. accessfn (TVM/TRVM 等粗粒度 trap)
5. Trap to EL3 (SCR_EL3 控制)         ← 最低
```

---

## 4. Runtime FGT 检查实现

### 4.1 access_check_cp_reg 中的 FGT 处理

```c
// target/arm/tcg/op_helper.c:858
if (arm_fgt_active(env, arm_current_el(env))) {
    uint64_t trapword = 0;
    unsigned int idx = FIELD_EX32(ri->fgt, FGT, IDX);
    unsigned int bitpos = FIELD_EX32(ri->fgt, FGT, BITPOS);
    bool rev = FIELD_EX32(ri->fgt, FGT, REV);
    bool nxs = FIELD_EX32(ri->fgt, FGT, NXS);
    bool trapbit;

    // 根据操作类型选择检查的寄存器
    if (ri->fgt & FGT_EXEC) {
        trapword = env->cp15.fgt_exec[idx];      // HFGITR_EL2
    } else if (isread && (ri->fgt & FGT_R)) {
        trapword = env->cp15.fgt_read[idx];      // HFGRTR/HDFGRTR
    } else if (!isread && (ri->fgt & FGT_W)) {
        trapword = env->cp15.fgt_write[idx];     // HFGWTR/HDFGWTR
    }

    // HCRX_EL2.FGTnXS 豁免检查
    if (nxs && (arm_hcrx_el2_eff(env) & HCRX_FGTNXS)) {
        trapbit = 0;  // nXS 变体在 FGTnXS=1 时不 trap
    } else {
        trapbit = extract64(trapword, bitpos, 1);
    }

    // 反转位: rev=1 → bit=0 trap, bit=1 not-trap
    if (trapbit != rev) {
        res = CP_ACCESS_TRAP_EL2;  // Trap to EL2
        goto fail;
    }
}
```

### 4.2 判断逻辑真值表

| `trapbit` | `rev` | `trapbit != rev` | 结果 |
|:---------:|:-----:|:----------------:|:----:|
| 0 | 0 | false | 不 trap |
| 1 | 0 | true | **trap** |
| 0 | 1 | true | **trap** |
| 1 | 1 | false | 不 trap |

- 普通位 (rev=0): bit=1 → trap
- 反转位 (rev=1): bit=0 → trap (默认 trap，需要显式清除)

---

## 5. Translate-time FGT 优化

### 5.1 FGT_ACTIVE TB Flag

FGT 状态被编码到 TB flags 中，避免每次翻译时重新计算：

```c
// target/arm/cpu.h:2448
FIELD(TBFLAG_ANY, FGT_ACTIVE, 12, 1)
FIELD(TBFLAG_ANY, FGT_SVC, 13, 1)
```

### 5.2 SVC FGT Trap

SVC 的 FGT trap 特殊：在翻译时预计算：

```c
// target/arm/tcg/hflags.c:26
// HFGITR_EL2.SVC_EL0 或 SVC_EL1
return el == 0 ?
    FIELD_EX64(env->cp15.fgt_exec[FGTREG_HFGITR], HFGITR_EL2, SVC_EL0) :
    FIELD_EX64(env->cp15.fgt_exec[FGTREG_HFGITR], HFGITR_EL2, SVC_EL1);
```

这通过 TB flag 传递给翻译器，使得 SVC 的 trap 判断在翻译时完成。

### 5.3 ERET FGT Trap

```c
// hflags.c:376
if (FIELD_EX64(env->cp15.fgt_exec[FGTREG_HFGITR], HFGITR_EL2, ERET)) {
    // 设置 s->trap_eret 标志
    // 翻译时生成 trap 代码
}
```

ERET 的 FGT trap 也在翻译时确定（通过 TB flag），这是性能优化。

---

## 6. FGT 覆盖的寄存器清单

### 6.1 HFGRTR_EL2 / HFGWTR_EL2 (系统功能寄存器)

| 位 | 寄存器 | 说明 |
|---|--------|------|
| 0 | AFSR0_EL1 | Auxiliary Fault Status |
| 1 | AFSR1_EL1 | Auxiliary Fault Status |
| 3 | AMAIR_EL1 | Auxiliary Memory Attribute |
| 4-8 | APDAKEY/APDBKEY/... | PAC 密钥寄存器 |
| 9 | CCSIDR_EL1 | Cache Size ID |
| 11 | CONTEXTIDR_EL1 | Context ID |
| 12 | CPACR_EL1 | Coprocessor Access Control |
| 14 | CTR_EL0 | Cache Type |
| 16 | ESR_EL1 | Exception Syndrome |
| 17 | FAR_EL1 | Fault Address |
| 26 | MAIR_EL1 | Memory Attribute |
| 27 | MIDR_EL1 | Main ID |
| 31 | SCTLR_EL1 | System Control |
| 34 | TCR_EL1 | Translation Control |
| 37-38 | TTBR0/TTBR1_EL1 | Translation Table Base |
| 39 | VBAR_EL1 | Vector Base Address |

### 6.2 HDFGRTR_EL2 / HDFGWTR_EL2 (调试/PMU 寄存器)

| 位 | 寄存器 | 说明 |
|---|--------|------|
| 0-3 | DBGBCRn/DBGBVRn/... | 断点/观察点寄存器 |
| 4 | MDSCR_EL1 | Debug System Control |
| 10-19 | PMEVCNTRn/PMCCntr/... | PMU 计数器 |

### 6.3 HFGITR_EL2 (指令执行)

| 位 | 指令/操作 | 说明 |
|---|----------|------|
| 0-2 | ICIALLUIS/ICIALLU/ICIVAU | Cache 维护 |
| 3-10 | DCIVAC/DCISW/... | Data Cache 操作 |
| 11 | DCZVA | DC ZVA (零化) |
| 12-17 | ATS1E1R/W/... | 地址翻译 |
| 18-49 | TLBI* | TLB 维护 (含 nXS) |
| 51 | ERET | 异常返回 |
| 52 | SVC_EL0 | 系统调用 (EL0) |
| 53 | SVC_EL1 | 系统调用 (EL1) |

---

## 7. 反转位 (REV) 的设计意义

### 7.1 正常位 vs 反转位

```
正常位 (DO_BIT):    bit=0 不trap, bit=1 trap
反转位 (DO_REV_BIT): bit=0 trap,   bit=1 不trap
```

### 7.2 使用场景

反转位用于**新增特性的寄存器**，默认应该被 trap：

```c
DO_REV_BIT(HFGRTR, NTPIDR2_EL0),   // FEAT_SME: TPIDR2_EL0
DO_REV_BIT(HFGRTR, NPIRE0_EL1),    // FEAT_S1PIE: PIRE0_EL1
DO_REV_BIT(HFGRTR, NPIR_EL1),      // FEAT_S1PIE: PIR_EL1
DO_REV_BIT(HFGRTR, NMAIR2_EL1),    // FEAT_AIE: MAIR2_EL1
DO_REV_BIT(HFGITR, NGCSPUSHM_EL1), // FEAT_GCS: GCS push
DO_REV_BIT(HFGITR, NGCSEPP),       // FEAT_GCS: GCS exception
```

设计原理：
- 旧 Hypervisor 不知道新寄存器 → FGT 寄存器该位 = 0 (reset 值)
- 反转位: 0 = trap → 旧 Hypervisor 默认 trap 新寄存器 ✅
- 确保向后兼容：不认识的新特性默认被 trap

---

## 8. nXS (FEAT_XS) 豁免机制

### 8.1 问题背景

FEAT_XS 引入了 TLBI 指令的 nXS 变体（不等待 XS 屏障）。对于同一个 FGT bit，普通 TLBI 和 nXS TLBI 可能需要不同的 trap 行为。

### 8.2 HCRX_EL2.FGTnXS 控制

```c
// op_helper.c:877
if (nxs && (arm_hcrx_el2_eff(env) & HCRX_FGTNXS)) {
    trapbit = 0;  // FGTnXS=1: nXS 变体不受 FGT 控制
}
```

| HCRX.FGTnXS | TLBI 普通 | TLBI nXS |
|:-----------:|:---------:|:--------:|
| 0 | 受 FGT 控制 | 受 FGT 控制 |
| 1 | 受 FGT 控制 | **不受 FGT 控制** |

### 8.3 QEMU 实现

```c
// DO_TLBINXS_BIT 宏:
#define DO_TLBINXS_BIT(REG, BITNAME)                             \
    FGT_##BITNAME = FGT_##REG | R_##REG##_EL2_##BITNAME##_SHIFT, \
    FGT_##BITNAME##NXS = FGT_##BITNAME | R_FGT_NXS_MASK

// 普通 TLBI 使用 FGT_TLBIVAE1
// nXS TLBI 使用 FGT_TLBIVAE1NXS（带 NXS 标记）
```

---

## 9. SCR_EL3.FGTEN 控制

### 9.1 EL3 的 FGT 控制

```c
// arm_fgt_active() 中:
(!arm_feature(env, ARM_FEATURE_EL3) || (env->cp15.scr_el3 & SCR_FGTEN))
```

- `SCR_EL3.FGTEN = 0`: FGT 被禁用（EL3 安全固件可以完全关闭 FGT）
- `SCR_EL3.FGTEN = 1`: FGT 生效

### 9.2 安全用途

安全固件（EL3）可能需要禁用 FGT 来防止 Hypervisor 过度 trap 导致的安全问题。

---

## 10. 与规范的一致性评估

### 10.1 实现完整度

| 规范要求 | QEMU 状态 | 说明 |
|---------|:--------:|------|
| HFGRTR_EL2 / HFGWTR_EL2 | ✅ | 64 位，覆盖所有功能寄存器 |
| HDFGRTR_EL2 / HDFGWTR_EL2 | ✅ | Debug/PMU 寄存器 |
| HFGITR_EL2 | ✅ | 指令 trap (TLBI/cache/AT/SVC/ERET) |
| 反转位 (nXXX) | ✅ | DO_REV_BIT 宏 |
| nXS 豁免 | ✅ | HCRX.FGTnXS + NXS 标记 |
| SCR_EL3.FGTEN | ✅ | arm_fgt_active() 检查 |
| E2H+TGE 禁用 | ✅ | arm_fgt_active() 检查 |
| 优先级 (HSTR > FGT > accessfn) | ✅ | access_check_cp_reg 顺序 |
| SVC trap | ✅ | TB flag 翻译时优化 |
| ERET trap | ✅ | TB flag 翻译时优化 |

### 10.2 设计优势

QEMU 的 FGT 实现有以下突出的设计优势：

1. **声明式**：每个寄存器只需添加 `.fgt = FGT_XXX`，自动生效
2. **零运行时开销**（FGT 未使能时）：`arm_fgt_active()` 快速返回 false
3. **紧凑编码**：14-bit 整数编码所有信息（type + idx + bitpos + rev + nxs）
4. **可扩展**：添加新寄存器只需在 enum 中加一行

### 10.3 总结

FGT 是 QEMU ARM64 中**架构设计最优雅的子系统**：
- 用 C 宏生成所有 FGT bit 的枚举常量
- 用统一的 `access_check_cp_reg` 路径处理所有 FGT 检查
- 支持完整的正向/反转/nXS 豁免逻辑
- 翻译时 TB flag 优化 (SVC/ERET)
- **与规范完全一致，无已知差异**
