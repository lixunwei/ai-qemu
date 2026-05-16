# ARM CP15 系统寄存器与 MMU 页表管理深度分析

> 基于 QEMU 11.0.50 源码分析 · 分析日期: 2025-07

---

## 目录

1. [概述](#1-概述)
2. [ARMCPRegInfo 寄存器描述结构](#2-armcpreginfo-寄存器描述结构)
3. [CPAccessResult 访问权限模型](#3-cpaccessresult-访问权限模型)
4. [cp_access_ok() 权限位编码](#4-cp_access_ok-权限位编码)
5. [寄存器注册系统](#5-寄存器注册系统)
6. [MRC/MCR 指令解码路径](#6-mrcmcr-指令解码路径)
7. [运行时访问检查：access_check_cp_reg](#7-运行时访问检查access_check_cp_reg)
8. [env->cp15 存储结构](#8-envcp15-存储结构)
9. [Banked 寄存器宏](#9-banked-寄存器宏)
10. [SCTLR：系统控制寄存器](#10-sctlr系统控制寄存器)
11. [SCTLR 关键位定义](#11-sctlr-关键位定义)
12. [sctlr_write() 写入回调](#12-sctlr_write-写入回调)
13. [TTBR0/TTBR1：页表基地址寄存器](#13-ttbr0ttbr1页表基地址寄存器)
14. [TCR/TTBCR：翻译控制寄存器](#14-tcrttbcr翻译控制寄存器)
15. [MAIR：内存属性寄存器](#15-mair内存属性寄存器)
16. [MMU 翻译禁用判断](#16-mmu-翻译禁用判断)
17. [页表遍历总调度：get_phys_addr_nogpc()](#17-页表遍历总调度get_phys_addr_nogpc)
18. [ARMv5 短描述符页表遍历](#18-armv5-短描述符页表遍历)
19. [TTBR0/TTBR1 选择：get_level1_table_address()](#19-ttbr0ttbr1-选择get_level1_table_address)
20. [ARMv6 短描述符页表遍历](#20-armv6-短描述符页表遍历)
21. [LPAE/AArch64 长描述符页表遍历](#21-lpaeaarch64-长描述符页表遍历)
22. [Stage 2 页表遍历](#22-stage-2-页表遍历)
23. [权限检查：get_S1prot()](#23-权限检查get_s1prot)
24. [Domain 访问控制（短描述符）](#24-domain-访问控制短描述符)
25. [TLBI 操作实现](#25-tlbi-操作实现)
26. [翻译 Regime 概念](#26-翻译-regime-概念)
27. [HCR_EL2 对 MMU 的影响](#27-hcr_el2-对-mmu-的影响)
28. [QEMU TLB 缓存机制](#28-qemu-tlb-缓存机制)
29. [关键 CP15 寄存器注册表](#29-关键-cp15-寄存器注册表)
30. [页表遍历完整流程图](#30-页表遍历完整流程图)
31. [AArch32 vs AArch64 MMU 对比](#31-aarch32-vs-aarch64-mmu-对比)

---

## 1. 概述

ARM 系统寄存器（CP15 协处理器在 AArch32 中、系统寄存器空间在 AArch64 中）控制着 CPU 的核心行为：MMU 使能、页表基地址、内存属性、中断向量、缓存策略等。QEMU 使用统一的 **ARMCPRegInfo** 框架来描述、注册和访问所有这些寄存器，并在 `ptw.c` 中实现了完整的多格式页表遍历。

**核心源文件**：

| 文件 | 内容 |
|------|------|
| `target/arm/cpregs.h` | ARMCPRegInfo 结构、访问权限类型、字段偏移宏 |
| `target/arm/helper.c` | 寄存器定义数组、注册函数、CPSR/SCTLR 回调 |
| `target/arm/cpu.h` | env->cp15 存储结构、SCTLR 位定义 |
| `target/arm/ptw.c` | MMU 页表遍历（短/长描述符、AArch64、Stage 2） |
| `target/arm/tcg/translate.c` | MRC/MCR 解码、access_check 生成 |
| `target/arm/tcg/op_helper.c` | 运行时访问检查 |
| `target/arm/tcg/tlb-insns.c` | TLBI 指令实现 |
| `target/arm/internals.h` | regime 辅助函数 |

---

## 2. ARMCPRegInfo 寄存器描述结构

```c
// cpregs.h:920-1049
struct ARMCPRegInfo {
    const char *name;           // 寄存器名称（调试用）

    // 编址信息
    uint8_t cp;                 // 协处理器号（AArch32: 15; AArch64: 0x13 约定）
    uint8_t crn;                // CRn（主寄存器号）
    uint8_t crm;                // CRm（辅助寄存器号）
    uint8_t opc0;               // Op0（AArch64 编码空间）
    uint8_t opc1;               // Op1
    uint8_t opc2;               // Op2

    // 状态与类型
    CPState state;              // ARM_CP_STATE_AA32 / AA64 / BOTH
    int type;                   // ARM_CP_* 类型标志
    CPAccessRights access;      // PL*_[RW] 权限
    CPSecureState secure;       // 安全状态

    // Fine-Grained Trap / NV2 / VHE 重定向
    FGTBit fgt;
    uint32_t nv2_redirect_offset;
    uint32_t vhe_redir_to_el2, vhe_redir_to_el01;

    // 值与存储
    uint64_t resetvalue;        // 复位值
    ptrdiff_t fieldoffset;      // offsetof(CPUARMState, ...) — 直接存储偏移
    ptrdiff_t bank_fieldoffsets[2]; // [0]=Secure, [1]=NonSecure banked 偏移

    // 回调函数
    CPAccessFn *accessfn;       // 运行时访问检查
    CPReadFn *readfn;           // 自定义读
    CPWriteFn *writefn;         // 自定义写
    CPReadFn *raw_readfn;       // 原始读（迁移/调试用）
    CPWriteFn *raw_writefn;     // 原始写
    CPResetFn *resetfn;         // 自定义复位
};
```

**fieldoffset 机制**：当 `fieldoffset != 0` 时，读写操作直接通过 `env + fieldoffset` 访问，无需 readfn/writefn。对于 banked 寄存器，`bank_fieldoffsets[0]` 是 Secure 偏移，`bank_fieldoffsets[1]` 是 NonSecure 偏移。

---

## 3. CPAccessResult 访问权限模型

```c
// cpregs.h:335-375
typedef enum CPAccessResult {
    CP_ACCESS_OK = 0,                    // 访问允许

    CP_ACCESS_EL_MASK = 3,               // 低 2 位编码目标 EL

    // 分类异常陷阱（到指定 EL）
    CP_ACCESS_TRAP_BIT = (1 << 2),
    CP_ACCESS_TRAP_EL1 = CP_ACCESS_TRAP_BIT | 1,  // 陷入 EL1
    CP_ACCESS_TRAP_EL2 = CP_ACCESS_TRAP_BIT | 2,  // 陷入 EL2
    CP_ACCESS_TRAP_EL3 = CP_ACCESS_TRAP_BIT | 3,  // 陷入 EL3

    // UNDEFINED（无分类 syndrome）
    CP_ACCESS_UNDEFINED = (2 << 2),

    // GCS EXLOCK 异常
    CP_ACCESS_EXLOCK = (3 << 2),
} CPAccessResult;
```

当 `accessfn` 返回非 `CP_ACCESS_OK` 值时，QEMU 会根据返回值生成对应的异常。

---

## 4. cp_access_ok() 权限位编码

```c
// cpregs.h:1122-1126
static inline bool cp_access_ok(int current_el,
                                const ARMCPRegInfo *ri, int isread)
{
    return (ri->access >> ((current_el * 2) + isread)) & 1;
}
```

`access` 字段是一个位图，每 2 位对应一个 EL（读/写各 1 位）：

| 位 | 含义 |
|----|------|
| bit 0 | EL0 写 |
| bit 1 | EL0 读 |
| bit 2 | EL1 写 |
| bit 3 | EL1 读 |
| bit 4 | EL2 写 |
| bit 5 | EL2 读 |
| bit 6 | EL3 写 |
| bit 7 | EL3 读 |

常用预定义值如 `PL1_RW` = `0x3C`（EL1+ 读写）、`PL0_RW` = `0xFF`（所有 EL 读写）。

---

## 5. 寄存器注册系统

### 5.1 注册入口

```c
// helper.c:6212+ register_cp_regs_for_features()
```

此函数根据 CPU feature 标志，条件性地注册各组寄存器：

| 注册数组 | 位置 | 内容 |
|----------|------|------|
| `cp_reginfo[]` | helper.c:432 | FCSEIDR, CONTEXTIDR |
| `not_v8_cp_reginfo[]` | helper.c:474 | v7 及更早的寄存器 |
| `vmsa_pmsa_cp_reginfo[]` | helper.c:2817 | DFSR, IFSR, DFAR, FAR_EL1 |
| `vmsa_cp_reginfo[]` | helper.c:2841 | ESR_EL1, TTBR0/1, TCR, TTBCR |
| `lpae_cp_reginfo[]` | helper.c:3004 | LPAE 相关寄存器 |
| EL2/EL3 groups | helper.c:4067+ | Hypervisor/Secure 寄存器 |
| SCTLR/VBAR | helper.c:7317+ | 条件注册 |

### 5.2 哈希表存储

注册的寄存器通过编码 key（cp/crn/crm/opc 组合）插入 `cpu->cp_regs` 哈希表（helper.c:7523-7613）。查找时使用：

```c
// helper.c:8037-8040
const ARMCPRegInfo *get_arm_cp_reginfo(GHashTable *cpregs, uint32_t encoded_cp)
```

---

## 6. MRC/MCR 指令解码路径

### 6.1 翻译阶段

```c
// translate.c:2324+ trans_MCR / trans_MRC
```

MRC/MCR 指令被解码后调用 `do_coproc_insn()`，它：
1. 通过 key 查找 `ARMCPRegInfo`
2. 检查静态权限（`cp_access_ok()`）
3. 如果有 `accessfn` 或 FGT，生成运行时检查代码（translate.c:1837-1849）：

```c
// translate.c:1846-1848
gen_helper_access_check_cp_reg(tcg_ri, tcg_env,
                               tcg_constant_i32(key),
                               tcg_constant_i32(syndrome),
                               tcg_constant_i32(isread));
```

4. 根据 `fieldoffset` 或 `readfn`/`writefn` 生成读/写代码

### 6.2 特殊类型处理

```c
// translate.c:1860-1870
switch (ri->type & ARM_CP_SPECIAL_MASK) {
case ARM_CP_NOP:   return;          // 空操作
case ARM_CP_WFI:   DISAS_WFI;      // WFI 指令
// ...
}
```

---

## 7. 运行时访问检查：access_check_cp_reg

```c
// op_helper.c:802-890 HELPER(access_check_cp_reg)
```

运行时执行的检查包括：
1. **accessfn 回调**：调用 `ri->accessfn(env, ri, isread)`
2. **Fine-Grained Trap**：检查 `env->cp15.fgt_read/write/exec` 位图
3. **HSTR_EL2 陷阱**：EL0 访问 CP15 时的 Hypervisor 陷阱
4. 根据 `CPAccessResult` 生成对应异常

---

## 8. env->cp15 存储结构

```c
// cpu.h:320-597
struct {
    uint32_t c0_cpuid;

    // 使用 union 实现 EL 索引 + Secure/NonSecure 别名
    union {
        struct { uint64_t _unused; uint64_t sctlr_ns; uint64_t hsctlr; uint64_t sctlr_s; };
        uint64_t sctlr_el[4];    // sctlr_el[1]=NS, sctlr_el[2]=HYP, sctlr_el[3]=S
    };

    union {
        struct { uint64_t _unused; uint64_t ttbr0_ns; uint64_t _unused1; uint64_t ttbr0_s; };
        uint64_t ttbr0_el[4];    // ttbr0_el[1]=NS, ttbr0_el[3]=S
    };

    union {
        struct { uint64_t _unused; uint64_t ttbr1_ns; uint64_t _unused1; uint64_t ttbr1_s; };
        uint64_t ttbr1_el[4];
    };

    uint64_t vttbr_el2;          // Stage 2 基地址
    uint64_t tcr_el[4];          // 翻译控制
    uint64_t vtcr_el2;           // Stage 2 翻译控制
    uint64_t mair_el[4];         // 内存属性

    // Fault 寄存器
    uint64_t esr_el[4];          // 异常 Syndrome
    uint64_t far_el[4];          // 故障地址
    uint64_t dfsr_s, dfsr_ns;    // AArch32 Data Fault Status
    uint64_t ifsr_s, ifsr_ns;    // AArch32 Instruction Fault Status
    uint64_t dfar_s, dfar_ns;    // AArch32 Data Fault Address

    // 向量基地址
    uint64_t vbar_ns, vbar_s;    // AArch32 banked VBAR
    uint64_t mvbar;              // Monitor VBAR
    uint64_t hvbar;              // Hypervisor VBAR

    // 其他
    uint64_t contextidr_el[4];   // Context ID
    uint64_t dacr_s, dacr_ns;    // Domain Access Control
    // ... 更多字段
} cp15;
```

**设计原则**：使用 `union` 使得 `sctlr_el[1]` 和 `sctlr_ns` 指向同一存储位置，AArch64 代码通过 `el` 索引访问，AArch32 代码通过 `_ns`/`_s` 后缀访问。

---

## 9. Banked 寄存器宏

```c
// cpregs.h:1202-1228
#define A32_BANKED_REG_GET(_env, _regname, _secure) \
    ((_secure) ? (_env)->cp15._regname##_s : (_env)->cp15._regname##_ns)

#define A32_BANKED_CURRENT_REG_GET(_env, _regname) \
    A32_BANKED_REG_GET((_env), _regname, \
        (arm_is_secure(_env) && !arm_el_is_aa64((_env), 3)))
```

**选择逻辑**：当前处于 Secure 状态**且** EL3 不是 AArch64 时，使用 `_s` 版本；否则使用 `_ns` 版本。这是因为当 EL3 为 AArch64 时，Secure EL1 使用的是 `_el[1]`（= `_ns`）而非 `_s`。

用于 VBAR、SCTLR、DACR 等 banked CP15 寄存器的运行时访问。

---

## 10. SCTLR：系统控制寄存器

SCTLR 是 ARM 最重要的系统控制寄存器，控制 MMU 使能、缓存策略、对齐检查、大端模式等。

### 存储

```c
// cpu.h:332-339
union {
    struct { uint64_t _unused; uint64_t sctlr_ns; uint64_t hsctlr; uint64_t sctlr_s; };
    uint64_t sctlr_el[4];
};
```

- `sctlr_el[1]` = `sctlr_ns`（Non-secure EL1）
- `sctlr_el[2]` = `hsctlr`（EL2/HYP）
- `sctlr_el[3]` = `sctlr_s`（Secure EL3）

### 注册

```c
// helper.c:7336-7349（条件注册）
```

---

## 11. SCTLR 关键位定义

```c
// cpu.h:1398-1469
#define SCTLR_M       (1U << 0)   // MMU 使能
#define SCTLR_A       (1U << 1)   // 对齐检查使能
#define SCTLR_C       (1U << 2)   // 数据缓存使能
#define SCTLR_I       (1U << 12)  // 指令缓存使能
#define SCTLR_V       (1U << 13)  // 高向量（AArch32, 0xFFFF0000）
#define SCTLR_EE      (1U << 25)  // 异常大端
#define SCTLR_TE      (1U << 30)  // 异常进入 Thumb 状态（AArch32）
#define SCTLR_WXN     (1U << 19)  // Write implies XN
#define SCTLR_SPAN    (1U << 23)  // Set PAN on exception
#define SCTLR_E0E     (1U << 24)  // EL0 大端（AArch64）
#define SCTLR_XP      (1U << 23)  // v6: 扩展页表（v7+ RAO）
#define SCTLR_AFE     (1U << 29)  // AArch32: Access Flag Enable
#define SCTLR_TRE     (1U << 28)  // AArch32: TEX Remap Enable
```

**MMU 相关核心位**：
- **SCTLR.M**（bit 0）：MMU 使能/禁用总开关
- **SCTLR.C**（bit 2）：数据缓存使能
- **SCTLR.I**（bit 12）：指令缓存使能
- **SCTLR.WXN**（bit 19）：可写区域自动不可执行
- **SCTLR.XP**（bit 23）：v6 扩展页表格式（使能 AP[2:0] 新编码）
- **SCTLR.AFE**（bit 29）：访问标志使能（AP[0] 变为 Access Flag）
- **SCTLR.TRE**（bit 28）：TEX 重映射使能

---

## 12. sctlr_write() 写入回调

```c
// helper.c:3265-3298
static void sctlr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                        uint64_t value)
{
    ARMCPU *cpu = env_archcpu(env);

    if (arm_feature(env, ARM_FEATURE_PMSA) && !cpu->has_mpu) {
        value &= ~SCTLR_M;  // 无 MPU 时 M 位 RAZ/WI
    }

    // MTE 位过滤（无 MTE 特性时）
    if (ri->state == ARM_CP_STATE_AA64 && !cpu_isar_feature(aa64_mte, cpu)) {
        value &= ~(SCTLR_ITFSB | SCTLR_TCF | SCTLR_ATA | ...);
    }

    if (raw_read(env, ri) == value) {
        return;  // 值未变化时跳过 TLB 刷新（优化热路径）
    }

    raw_write(env, ri, value);
    tlb_flush(CPU(cpu));  // SCTLR 变化可能影响 MMU，刷新 TLB
}
```

**关键优化**：Linux 内核经常执行无意义的 SCTLR 写入（值不变），QEMU 通过比较跳过 TLB 刷新以避免性能损失。

---

## 13. TTBR0/TTBR1：页表基地址寄存器

### 注册定义

```c
// helper.c:2850-2869
{ .name = "TTBR0_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .opc1 = 0, .crn = 2, .crm = 0, .opc2 = 0,
  .access = PL1_RW, .accessfn = access_tvm_trvm,
  .writefn = vmsa_ttbr_write, .resetvalue = 0,
  .bank_fieldoffsets = { offsetof(CPUARMState, cp15.ttbr0_s),
                         offsetof(CPUARMState, cp15.ttbr0_ns) } },
{ .name = "TTBR1_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .opc1 = 0, .crn = 2, .crm = 0, .opc2 = 1,
  .access = PL1_RW, .accessfn = access_tvm_trvm,
  .writefn = vmsa_ttbr_write, .resetvalue = 0,
  .bank_fieldoffsets = { offsetof(CPUARMState, cp15.ttbr1_s),
                         offsetof(CPUARMState, cp15.ttbr1_ns) } },
```

**关键特性**：
- `ARM_CP_STATE_BOTH`：同时在 AArch32（CP15 c2,c0,0/1）和 AArch64（op0=3）空间可见
- `bank_fieldoffsets`：Secure/NonSecure 分别存储在 `ttbr0_s`/`ttbr0_ns`
- `vmsa_ttbr_write()`：写入时检查 ASID 是否变化，变化则刷新 TLB

### TTBR 中的 ASID

TTBR 的高位包含 ASID（Address Space Identifier），用于标识不同进程的地址空间。ASID 变化时需要 TLB 刷新（helper.c:2773-2782）。

---

## 14. TCR/TTBCR：翻译控制寄存器

### AArch64: TCR_EL1

```c
// helper.c:2870-2880
{ .name = "TCR_EL1", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .crn = 2, .crm = 0, .opc1 = 0, .opc2 = 2,
  .access = PL1_RW,
  .writefn = vmsa_tcr_el12_write,
  .fieldoffset = offsetof(CPUARMState, cp15.tcr_el[1]) },
```

### AArch32: TTBCR（TCR_EL1 别名）

```c
// helper.c:2881-2886
{ .name = "TTBCR", .cp = 15, .crn = 2, .crm = 0, .opc1 = 0, .opc2 = 2,
  .access = PL1_RW, .type = ARM_CP_ALIAS,
  .writefn = vmsa_ttbcr_write,
  .bank_fieldoffsets = { offsetoflow32(CPUARMState, cp15.tcr_el[3]),
                         offsetoflow32(CPUARMState, cp15.tcr_el[1])} },
```

**TCR 关键字段**：

| 字段 | 位域 | 功能 |
|------|------|------|
| T0SZ | [5:0] | TTBR0 管理的地址范围大小 (64-T0SZ 位) |
| T1SZ | [21:16] | TTBR1 管理的地址范围大小 |
| TG0 | [15:14] | TTBR0 粒度 (00=4KB, 01=64KB, 10=16KB) |
| TG1 | [31:30] | TTBR1 粒度 |
| IPS | [34:32] | 中间物理地址大小 |
| EPD0 | [7] | 禁用 TTBR0 遍历 |
| EPD1 | [23] | 禁用 TTBR1 遍历 |

### AArch32 TTBCR.N

在 AArch32 短描述符格式中，TTBCR.N（bit[2:0]）决定 TTBR0/TTBR1 的分界点：
- N=0：TTBR0 覆盖全部 4GB
- N=7：TTBR0 覆盖低 32MB，TTBR1 覆盖其余

---

## 15. MAIR：内存属性寄存器

```c
// helper.c:990-998（注册）
// 存储: env->cp15.mair_el[4]
```

### 页表遍历中的使用

```c
// ptw.c:2334-2346
attrindx = extract32(attrs, 2, 3);         // 从页表描述符提取 AttrIndx[2:0]
mair = env->cp15.mair_el[el];              // 获取对应 EL 的 MAIR
result->cacheattrs.attrs = extract64(mair, attrindx * 8, 8); // 8 位属性
```

MAIR 包含 8 个 8 位的属性条目（共 64 位），页表描述符中的 AttrIndx 字段选择使用哪个条目。每个条目定义内存类型：

| 属性值 | 内存类型 |
|--------|---------|
| 0x00 | Device-nGnRnE |
| 0x04 | Device-nGnRE |
| 0x44 | Normal Non-Cacheable |
| 0xFF | Normal Write-Back Cacheable |

---

## 16. MMU 翻译禁用判断

```c
// ptw.c:248-305
static bool regime_translation_disabled(CPUARMState *env, ARMMMUIdx mmu_idx,
                                        ARMSecuritySpace space)
```

判断逻辑：

| MMU 索引 | 禁用条件 |
|----------|---------|
| Stage2/Stage2_S | `(HCR_DC \| HCR_VM) == 0`（两者都未设时禁用 Stage2） |
| E10_0/E10_1（EL0/EL1） | `HCR_EL2.TGE = 1`（TGE 使 EL0/1 MMU 无效） |
| Stage1_E0/E1 | `HCR_EL2.DC = 1`（DC 使 Stage1 SCTLR.M 无效） |
| 其他 | `(regime_sctlr & SCTLR_M) == 0` |

---

## 17. 页表遍历总调度：get_phys_addr_nogpc()

```c
// ptw.c:3661-3797
static bool get_phys_addr_nogpc(CPUARMState *env, S1Translate *ptw,
                                vaddr address, MMUAccessType access_type,
                                MemOp memop, GetPhysAddrResult *result,
                                ARMMMUFaultInfo *fi)
```

这是 MMU 页表遍历的**核心调度函数**，处理流程：

### 步骤 1：物理地址直通（ptw.c:3679-3686）
```c
case ARMMMUIdx_Phys_S / Phys_NS / Phys_Root / Phys_Realm:
    return get_phys_addr_disabled(...);  // 无需翻译
```

### 步骤 2：两阶段翻译设置（ptw.c:3688-3728）
```c
case ARMMMUIdx_Stage1_E0 / E1 / E1_PAN:
    ptw->in_ptw_idx = ARMMMUIdx_Stage2;  // PTW 本身需要 Stage2 翻译
    break;
case ARMMMUIdx_E10_0 / E10_1 / E10_1_PAN:
    // 如果 EL2 存在且 Stage2 未禁用
    return get_phys_addr_twostage(...);  // 递归: Stage1 → Stage2
```

### 步骤 3：FCSE（旧 ARM，ptw.c:3743-3750）
```c
if (address < 0x02000000 && !ARM_FEATURE_V8) {
    address += env->cp15.fcseidr;  // Fast Context Switch Extension
}
```

### 步骤 4：PMSA/MPU（ptw.c:3752-3779）
PMSAv5 / PMSAv7 / PMSAv8 内存保护单元路径。

### 步骤 5：MMU 翻译禁用检查（ptw.c:3784-3787）
```c
if (regime_translation_disabled(env, mmu_idx, ptw->in_space)) {
    return get_phys_addr_disabled(...);
}
```

### 步骤 6：选择页表格式（ptw.c:3789-3797）
```c
if (regime_using_lpae_format(env, mmu_idx)) {
    return get_phys_addr_lpae(...);     // LPAE 或 AArch64
} else if (arm_feature(env, ARM_FEATURE_V7) || sctlr & SCTLR_XP) {
    return get_phys_addr_v6(...);       // ARMv6/v7 短描述符
} else {
    return get_phys_addr_v5(...);       // ARMv5 短描述符
}
```

---

## 18. ARMv5 短描述符页表遍历

```c
// ptw.c:1056-1177 get_phys_addr_v5()
```

### 两级页表结构

**L1 表**：16KB，4096 个条目，每条映射 1MB
- 基地址：TTBR（14 位对齐）
- 索引：VA[31:20]（12 位 → 4096 条目）

**L2 表**（如果 L1 条目指向）：1KB，256 个条目
- 基地址：L1 条目中的地址
- 索引：VA[19:12]（8 位 → 256 条目）

### L1 描述符类型

| bit[1:0] | 类型 | 映射 |
|----------|------|------|
| 00 | Fault | 翻译故障 |
| 01 | Coarse Page Table | 指向 L2 表（4KB 页） |
| 10 | Section | 1MB 直接映射 |
| 11 | Fine Page Table | 指向 L2 表（1KB 页，v5 only） |

### Domain 检查

```c
// ptw.c:1086-1103
domain = (desc >> 5) & 0x0f;                       // 从 L1 描述符取 domain[3:0]
dacr = env->cp15.dacr_ns;                           // DACR 寄存器
domain_prot = (dacr >> (domain * 2)) & 3;           // 2 位保护域权限
if (domain_prot == 0 || domain_prot == 2) {
    fi->type = ARMFault_Domain;                      // 域故障
}
```

---

## 19. TTBR0/TTBR1 选择：get_level1_table_address()

```c
// ptw.c:935-959
static bool get_level1_table_address(CPUARMState *env, ARMMMUIdx mmu_idx,
                                     uint32_t *table, uint32_t address)
{
    uint64_t tcr = regime_tcr(env, mmu_idx);
    int maskshift = extract32(tcr, 0, 3);             // TTBCR.N
    uint32_t mask = ~(((uint32_t)0xffffffffu) >> maskshift);

    if (address & mask) {
        // 地址高位匹配 → 使用 TTBR1
        if (tcr & TTBCR_PD1) return false;            // PD1=1 禁用 TTBR1 遍历
        *table = regime_ttbr(env, mmu_idx, 1) & 0xffffc000;
    } else {
        // 地址低位匹配 → 使用 TTBR0
        if (tcr & TTBCR_PD0) return false;            // PD0=1 禁用 TTBR0 遍历
        base_mask = ~((uint32_t)0x3fffu >> maskshift);
        *table = regime_ttbr(env, mmu_idx, 0) & base_mask;
    }
    *table |= (address >> 18) & 0x3ffc;              // L1 表偏移
    return true;
}
```

**TTBCR.N 的作用**：
- N=0：mask=0x00000000，所有地址使用 TTBR0
- N=1：mask=0x80000000，高半使用 TTBR1，低半使用 TTBR0
- N=7：mask=0xFE000000，只有低 32MB 使用 TTBR0

---

## 20. ARMv6 短描述符页表遍历

```c
// ptw.c:1179-1334 get_phys_addr_v6()
```

与 v5 的主要差异：
- 支持 SCTLR.XP 扩展页表格式（新的 AP[2:0] 编码）
- 支持 Execute-Never (XN) 位
- 支持 Privileged Execute-Never (PXN) 位
- L1 "Supersection" (16MB) 映射
- AP 权限编码扩展（AP[2] 引入只读权限）

---

## 21. LPAE/AArch64 长描述符页表遍历

```c
// ptw.c:1859-2449 get_phys_addr_lpae()
```

此函数同时处理 **AArch32 LPAE** 和 **AArch64** 页表遍历。

### 粒度与级别

| 粒度 | 页大小 | 表大小 | 级别数 | VA 范围 |
|------|--------|--------|--------|---------|
| 4KB | 4KB | 4KB (512 条目) | 4 级 (L0-L3) | 48 位 |
| 16KB | 16KB | 16KB (2048 条目) | 4 级 (L0-L3) | 47 位 |
| 64KB | 64KB | 64KB (8192 条目) | 3 级 (L1-L3) | 52 位 |

### 描述符格式（64 位）

| 位域 | 含义 |
|------|------|
| [0] | Valid |
| [1] | Table(1) 或 Block(0) |
| [11:2] | 低属性 (AttrIndx, NS, AP, SH, AF, nG) |
| [47:12] | 输出地址（4KB 对齐） |
| [63:48] | 高属性 (XN, PXN, Contiguous, DBM, etc.) |

### TTBR 选择（AArch64）

由 TCR_EL1.T0SZ/T1SZ 决定：
- 地址在 `[0, 2^(64-T0SZ))` 范围内 → TTBR0
- 地址在 `[2^(64-T1SZ)-1 的补码, 0xFFFF...]` 范围内 → TTBR1
- 中间的空洞 → Translation Fault

### IPA 大小（ptw.c:108-117）

`pamax_map[]` 将 TCR.IPS/PS 字段值映射为物理地址位宽：

| IPS 值 | PA 位宽 |
|--------|---------|
| 0 | 32 位 (4GB) |
| 1 | 36 位 (64GB) |
| 2 | 40 位 (1TB) |
| 3 | 42 位 (4TB) |
| 4 | 44 位 (16TB) |
| 5 | 48 位 (256TB) |
| 6 | 52 位 (4PB) |

---

## 22. Stage 2 页表遍历

### 两阶段翻译流程

```
Guest VA → [Stage 1] → IPA → [Stage 2] → PA
           TTBR0/TTBR1       VTTBR_EL2
           TCR_EL1            VTCR_EL2
```

### 调度逻辑

```c
// ptw.c:3709-3728
case ARMMMUIdx_E10_0:
    s1_mmu_idx = ARMMMUIdx_Stage1_E0;
    goto do_twostage;
// ...
do_twostage:
    ptw->in_mmu_idx = s1_mmu_idx;
    if (arm_feature(env, ARM_FEATURE_EL2) &&
        !regime_translation_disabled(env, ARMMMUIdx_Stage2, ...)) {
        return get_phys_addr_twostage(env, ptw, ...);
    }
```

### Stage 2 特有

- 使用 `VTTBR_EL2` 作为页表基地址
- 使用 `VTCR_EL2` 控制翻译参数
- 权限由 `get_S2prot()` 检查（ptw.c:1343-1382）
- S1 + S2 属性合并在 ptw.c:3362-3464

---

## 23. 权限检查：get_S1prot()

```c
// ptw.c:1434-1520
static int get_S1prot(CPUARMState *env, ARMMMUIdx mmu_idx, bool is_aa64,
                      int user_rw, int prot_rw, int xn, int pxn,
                      ARMSecuritySpace in_pa, ARMSecuritySpace out_pa)
```

### 权限判断流程

1. **用户态**（`is_user`）：使用 `user_rw` 权限
2. **特权态 + PAN**（ptw.c:1456-1462）：
   - 基本 PAN：如果 EL0 有数据权限，则 EL1 数据访问被禁止
   - PAN3 (EPAN)：如果 EL0 有数据或执行权限，EL1 数据访问被禁止

3. **安全状态过滤**（ptw.c:1465-1498）：
   - Root → 非 Root 输出：禁止指令获取
   - Secure + SCR.SIF：禁止从非安全内存获取指令

4. **WXN**（ptw.c:1507-1508）：
   ```c
   wxn = regime_sctlr(env, mmu_idx) & SCTLR_WXN;
   ```
   当 WXN=1 时，所有可写页面自动不可执行。

5. **XN/PXN 合并**（ptw.c:1511-1514）：
   ```c
   if (regime_has_2_ranges(mmu_idx) && !is_user) {
       xn = pxn || (user_rw & PAGE_WRITE);
   }
   ```

---

## 24. Domain 访问控制（短描述符）

AArch32 短描述符格式支持 16 个域（Domain 0-15），每个域在 DACR 中有 2 位控制：

| DACR 值 | 含义 |
|---------|------|
| 00 | No access — 任何访问都产生域故障 |
| 01 | Client — 检查 AP 权限位 |
| 10 | Reserved — 与 No access 相同 |
| 11 | Manager — 不检查 AP，允许所有访问 |

```c
// ptw.c:1086-1103（v5 实现）
domain = (desc >> 5) & 0x0f;
domain_prot = (dacr >> (domain * 2)) & 3;
```

**注意**：LPAE 和 AArch64 格式不使用 Domain，权限完全由 AP/XN 位控制。

---

## 25. TLBI 操作实现

### 实现位置

```
target/arm/tcg/tlb-insns.c
```

### TLBI 类型

| 操作 | 函数 | 作用 |
|------|------|------|
| TLBIALL | tlb-insns.c:119-129 | 刷新当前安全状态所有 TLB |
| TLBI VAE | tlb-insns.c:419-431 | 按虚拟地址刷新 |
| TLBI ASIDE | tlb-insns.c:326-335 | 按 ASID 刷新 |
| TLBI IPAS2E1 | tlb-insns.c:501-522 | 按 IPA 刷新 Stage 2 |
| TLBI VMALLE1 | tlb-insns.c:326-335 | 刷新当前 VMID 的所有 EL1 TLB |

### 底层 API

```c
// accel/tcg/cputlb.c
tlb_flush(CPUState *cpu);                          // 刷新所有 TLB
tlb_flush_by_mmuidx(cpu, mmuidx_mask);             // 按 MMU 索引刷新
tlb_flush_page_by_mmuidx(cpu, addr, mmuidx_mask);  // 按页地址 + 索引刷新
tlb_flush_*_all_cpus_synced(...);                   // 多 CPU 同步刷新
```

TLBI 指令通过 ARMCPRegInfo 的 `writefn` 注册，写入时触发 TLB 刷新。

---

## 26. 翻译 Regime 概念

翻译 Regime 是一组控制地址翻译的系统寄存器集合（SCTLR + TCR + TTBR + MAIR）。哪个 Regime 活跃取决于当前 EL 和 HCR_EL2 配置。

### Regime 访问函数

```c
// internals.h:1046-1050
regime_sctlr(env, mmu_idx) → env->cp15.sctlr_el[regime_el(mmu_idx)]

// internals.h:1062-1082
regime_tcr(env, mmu_idx)   → env->cp15.tcr_el[regime_el(mmu_idx)]
                              Stage2: vtcr_el2

// ptw.c:232-246
regime_ttbr(env, mmu_idx, n) → env->cp15.ttbr{n}_el[regime_el(mmu_idx)]
                                Stage2: vttbr_el2
```

### Regime 与 EL 的映射

| MMU Index | regime_el | 使用的寄存器集 |
|-----------|-----------|---------------|
| E10_0/E10_1 | 1 | SCTLR_EL1 + TCR_EL1 + TTBR0/1_EL1 + MAIR_EL1 |
| E2/E20_0/E20_2 | 2 | SCTLR_EL2 + TCR_EL2 + TTBR0_EL2 + MAIR_EL2 |
| E3 | 3 | SCTLR_EL3 + TCR_EL3 + TTBR0_EL3 + MAIR_EL3 |
| Stage2 | — | VTCR_EL2 + VTTBR_EL2 |

---

## 27. HCR_EL2 对 MMU 的影响

```c
// cpu.h:1695-1721
#define HCR_VM    (1ULL << 0)   // Stage 2 使能
#define HCR_DC    (1ULL << 12)  // Default Cacheable
#define HCR_TGE   (1ULL << 27)  // Trap General Exceptions
```

### 影响总结

| HCR 位 | 影响 |
|--------|------|
| **VM=1** | 使能 Stage 2 地址翻译 |
| **DC=1** | Stage 1 的 SCTLR.M 视为 0（禁用 Stage 1 MMU），同时 Stage 2 中 HCR_VM 视为 1 |
| **TGE=1** | EL0/1 的 SCTLR.M 视为 0，EL0 异常路由到 EL2 |

```c
// ptw.c:275-302（实现）
case Stage2:
    hcr_el2 = arm_hcr_el2_eff_secstate(env, space);
    return (hcr_el2 & (HCR_DC | HCR_VM)) == 0;  // DC 或 VM 任一为 1 则启用

case E10_0 / E10_1:
    if (hcr_el2 & HCR_TGE) return true;  // TGE 禁用 EL0/1 翻译

case Stage1_E0 / Stage1_E1:
    if (hcr_el2 & HCR_DC) return true;   // DC 禁用 Stage1 翻译
```

---

## 28. QEMU TLB 缓存机制

### TLB 填充

```c
// accel/tcg/cputlb.c:1040-1183 tlb_set_page_full()
```

当页表遍历成功后，结果被缓存到 QEMU 的 softmmu TLB 中：
- 快速路径 (`te`)：用于 TCG 生成代码的内联 TLB 查找
- 完整条目 (`fulltlb[index]`)：包含完整翻译信息

### TLB 刷新

```c
// accel/tcg/cputlb.c:369-421
tlb_flush_by_mmuidx(CPUState *cpu, uint16_t idxmap);
```

刷新触发条件：
- SCTLR 写入（MMU 使能/禁用变化）
- TTBR 写入（ASID 变化）
- TLBI 指令
- TCR 写入

---

## 29. 关键 CP15 寄存器注册表

| 寄存器 | CP15 编码 | AArch64 编码 | 定义位置 | 存储字段 |
|--------|-----------|-------------|---------|---------|
| SCTLR | c1,c0,0 | op0=3,crn=1,crm=0,opc2=0 | helper.c:7336 | sctlr_el[] |
| TTBR0 | c2,c0,0 | op0=3,crn=2,crm=0,opc2=0 | helper.c:2850 | ttbr0_el[] |
| TTBR1 | c2,c0,1 | op0=3,crn=2,crm=0,opc2=1 | helper.c:2860 | ttbr1_el[] |
| TCR/TTBCR | c2,c0,2 | op0=3,crn=2,crm=0,opc2=2 | helper.c:2870/2881 | tcr_el[] |
| DACR | c3,c0,0 | — (AArch32 only) | helper.c:480 | dacr_s/ns |
| DFSR | c5,c0,0 | — | helper.c:2818 | dfsr_s/ns |
| IFSR | c5,c0,1 | — | helper.c:2822 | ifsr_s/ns |
| ESR_EL1 | — | op0=3,crn=5,crm=2,opc2=0 | helper.c:2842 | esr_el[] |
| DFAR | c6,c0,0 | — | helper.c:2826 | dfar_s/ns |
| FAR_EL1 | — | op0=3,crn=6,crm=0,opc2=0 | helper.c:2830 | far_el[] |
| VBAR | c12,c0,0 | op0=3,crn=12,crm=0,opc2=0 | helper.c:7318 | vbar_s/ns |
| CONTEXTIDR | c13,c0,1 | op0=3,crn=13,crm=0,opc2=1 | helper.c:456 | contextidr_el[] |
| FCSEIDR | c13,c0,0 | — | helper.c:439 | fcseidr_s/ns |
| MAIR | — | op0=3,crn=10,crm=2,opc2=0 | helper.c:990 | mair_el[] |

---

## 30. 页表遍历完整流程图

```
Guest 虚拟地址访问
    │
    ├── QEMU softmmu TLB 查找
    │   ├── 命中 → 直接使用缓存翻译
    │   └── 未命中 → 触发页表遍历
    │
    ├── get_phys_addr() (ptw.c:3931)
    │   └── get_phys_addr_nogpc() (ptw.c:3661)
    │
    ├── 步骤 1: MMU Index 分类
    │   ├── Phys_* → get_phys_addr_disabled()（物理地址直通）
    │   ├── Stage1_* → 设置 Stage2 为 PTW 索引
    │   ├── E10_* → 检查是否需要两阶段翻译
    │   │   ├── EL2 存在 + Stage2 未禁用 → get_phys_addr_twostage()
    │   │   └── 否则 → 单阶段 Stage1
    │   └── Stage2_* → 设置物理为 PTW 索引
    │
    ├── 步骤 2: 翻译禁用检查
    │   └── regime_translation_disabled()
    │       ├── SCTLR.M = 0 → get_phys_addr_disabled()
    │       ├── HCR.TGE = 1 (EL0/1) → disabled
    │       └── HCR.DC = 1 (Stage1) → disabled
    │
    ├── 步骤 3: 选择页表格式
    │   ├── LPAE/AArch64 → get_phys_addr_lpae() (ptw.c:1859)
    │   │   ├── TCR T0SZ/T1SZ → 选择 TTBR0/TTBR1
    │   │   ├── 计算起始级别和粒度
    │   │   ├── 逐级遍历描述符（L0→L3）
    │   │   ├── Block 描述符 → 大页映射
    │   │   ├── Table 描述符 → 继续下一级
    │   │   └── Page 描述符 → 最终映射
    │   │
    │   ├── ARMv6/v7 → get_phys_addr_v6() (ptw.c:1179)
    │   │   ├── get_level1_table_address() → TTBR0/1 选择
    │   │   ├── L1 描述符：Section(1MB) / Page Table
    │   │   ├── Domain 检查 (DACR)
    │   │   └── L2 描述符：Large(64KB) / Small(4KB)
    │   │
    │   └── ARMv5 → get_phys_addr_v5() (ptw.c:1056)
    │       └── 类似 v6，但 AP 编码更简单
    │
    ├── 步骤 4: 权限检查
    │   ├── get_S1prot() — Stage 1 权限 (AP, XN, PXN, WXN, PAN)
    │   └── get_S2prot() — Stage 2 权限 (S2AP, XN)
    │
    ├── 步骤 5: 内存属性
    │   └── MAIR[AttrIndx] → 内存类型确定
    │
    └── 步骤 6: 结果缓存
        └── tlb_set_page_full() → 填入 QEMU softmmu TLB

两阶段翻译额外步骤:
    Stage 1 输出 IPA
        │
        └── Stage 2 遍历 (VTTBR_EL2 + VTCR_EL2)
            ├── IPA → PA 翻译
            ├── S2 权限检查
            └── S1 + S2 属性合并
```

---

## 31. AArch32 vs AArch64 MMU 对比

| 特性 | AArch32 短描述符 | AArch32 LPAE | AArch64 |
|------|-----------------|-------------|---------|
| **描述符宽度** | 32 位 | 64 位 | 64 位 |
| **页表级数** | 2 级 | 3 级 | 3-4 级 |
| **最小页大小** | 4KB | 4KB | 4KB/16KB/64KB |
| **最大映射** | 16MB (Supersection) | 1GB (Block) | 512GB (L1 Block) |
| **VA 范围** | 32 位 (4GB) | 40 位 (1TB) | 48/52 位 |
| **PA 范围** | 32 位 | 40 位 | 52 位 |
| **TTBR 选择** | TTBCR.N | TCR.T0SZ/T1SZ | TCR.T0SZ/T1SZ |
| **域 (Domain)** | 16 个域, DACR | 无 | 无 |
| **权限编码** | AP[2:0] | AP[2:1] | AP[2:1] |
| **内存属性** | TEX/C/B (+ TRE 重映射) | MAIR AttrIndx | MAIR AttrIndx |
| **执行权限** | XN (v6+) | XN + PXN | XN + PXN + UXN |
| **Stage 2** | 无 | 有 (VTTBR) | 有 (VTTBR_EL2) |
| **粒度选择** | 固定 4KB | 固定 4KB | TG0/TG1 可配置 |
| **安全状态** | NS 位 | NS 位 | NSTable + NS |
| **访问标志** | AFE (SCTLR.AFE) | 内建 AF | 内建 AF |
| **QEMU 实现** | get_phys_addr_v5/v6 | get_phys_addr_lpae | get_phys_addr_lpae |

---

> **文档信息**
> - 分析版本：QEMU 11.0.50
> - 源文件基线：target/arm/ 目录
> - 行号引用基于分析时的源码快照，代码更新后可能偏移
