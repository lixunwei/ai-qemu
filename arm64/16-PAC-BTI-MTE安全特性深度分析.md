# ARM64 PAC/BTI/MTE 安全特性深度分析

> QEMU 11.0.50 源码分析 — ARM64 指针认证、分支目标识别与内存标签扩展
>
> 关联文档：[14-EL状态管理与指令执行变化深度分析](14-EL状态管理与指令执行变化深度分析.md) | [03-TCG翻译引擎深度分析](../accelerator/03-TCG翻译引擎深度分析.md)

---

## 目录

1. [概述](#1-概述)
2. [PAC 指针认证码](#2-pac-指针认证码)
3. [PAC 密钥存储与管理](#3-pac-密钥存储与管理)
4. [PAC 算法实现](#4-pac-算法实现)
5. [PAC 指令翻译](#5-pac-指令翻译)
6. [PAC 认证分支指令](#6-pac-认证分支指令)
7. [PAC 使能控制与陷入](#7-pac-使能控制与陷入)
8. [PAC 认证失败处理](#8-pac-认证失败处理)
9. [BTI 分支目标识别](#9-bti-分支目标识别)
10. [BTI BTYPE 状态机](#10-bti-btype-状态机)
11. [BTI 保护页检查](#11-bti-保护页检查)
12. [MTE 内存标签扩展](#12-mte-内存标签扩展)
13. [MTE 标签存储架构](#13-mte-标签存储架构)
14. [MTE 标签检查流程](#14-mte-标签检查流程)
15. [MTE 故障处理机制](#15-mte-故障处理机制)
16. [MTE 随机标签生成](#16-mte-随机标签生成)
17. [三大特性的 TB 标志集成](#17-三大特性的-tb-标志集成)
18. [特性检测与 CPU 属性](#18-特性检测与-cpu-属性)
19. [总结](#19-总结)

---

## 1. 概述

ARMv8.3/8.5 引入了三大硬件安全特性，分别针对不同攻击向量：

| 特性 | ARM 版本 | 防御目标 | 核心机制 |
|------|---------|---------|---------|
| **PAC** (Pointer Authentication Code) | v8.3 | ROP/JOP 攻击 | 在指针高位嵌入密码学签名 |
| **BTI** (Branch Target Identification) | v8.5 | JOP 攻击 | 限制间接分支只能跳转到标记的着陆点 |
| **MTE** (Memory Tagging Extension) | v8.5 | 内存安全漏洞 | 为每 16 字节内存分配 4-bit 标签并在访问时校验 |

QEMU 在 TCG 模式下完整模拟了这三大特性。本文档深入分析其源码实现。

---

## 2. PAC 指针认证码

### 2.1 基本原理

PAC 利用虚拟地址的未使用高位存放密码学签名（PAC 值）。64 位地址空间中，应用通常只用低 48/52 位，高位可用于存放 PAC：

```
63    56 55  52 51        0
+--------+----+-----------+
|  PAC   |ext | VA 低位    |   TBI 开启时
+--------+----+-----------+

63  61 60  52 51          0
+------+-----+-----------+
| PAC  | ext | VA 低位    |   TBI 关闭时
+------+-----+-----------+
```

PAC 字段宽度取决于 TBI（Top Byte Ignore）和虚拟地址大小（tsz）。

### 2.2 五个密钥家族

ARM 定义了 5 组密钥，每组 128 位（高 64 位 + 低 64 位）：

| 密钥 | 用途 | SCTLR 使能位 |
|------|------|-------------|
| **APIAKey** | 指令地址签名 A | SCTLR.EnIA |
| **APIBKey** | 指令地址签名 B | SCTLR.EnIB |
| **APDAKey** | 数据地址签名 A | SCTLR.EnDA |
| **APDBKey** | 数据地址签名 B | SCTLR.EnDB |
| **APGAKey** | 通用签名（PACGA） | 始终可用 |

---

## 3. PAC 密钥存储与管理

### 3.1 密钥数据结构

```c
// cpu.h:180-182
typedef struct ARMPACKey {
    uint64_t lo, hi;
} ARMPACKey;
```

每个密钥为 128 位，存储为 `lo`（低 64 位）和 `hi`（高 64 位）。

### 3.2 CPUARMState 中的密钥

```c
// cpu.h:715-721
struct {
    ARMPACKey apia;   // APIAKey_EL1
    ARMPACKey apib;   // APIBKey_EL1
    ARMPACKey apda;   // APDAKey_EL1
    ARMPACKey apdb;   // APDBKey_EL1
    ARMPACKey apga;   // APGAKey_EL1
} keys;
```

五组密钥统一存储在 `CPUARMState.keys` 结构中。注意：ARM 架构上密钥寄存器名称为 `*_EL1`，但所有 EL 共享同一组密钥——不同 EL 使用相同密钥值进行认证。

### 3.3 密钥系统寄存器

密钥通过 `APxAKEYHI_EL1` / `APxAKEYLO_EL1` 系统寄存器访问（`helper.c:5225-5270`），每个密钥对应一对 HI/LO 寄存器，共 10 个系统寄存器。

---

## 4. PAC 算法实现

### 4.1 算法选择

```c
// pauth_helper.c:315-324
static uint64_t pauth_computepac(CPUARMState *env, uint64_t data,
                                 uint64_t modifier, ARMPACKey key)
{
    if (cpu_isar_feature(aa64_pauth_qarma5, env_archcpu(env))) {
        return pauth_computepac_architected(data, modifier, key, false);
    } else if (cpu_isar_feature(aa64_pauth_qarma3, env_archcpu(env))) {
        return pauth_computepac_architected(data, modifier, key, true);
    } else {
        return pauth_computepac_impdef(data, modifier, key);
    }
}
```

QEMU 支持三种 PAC 算法：

| 算法 | 特性标志 | ID 寄存器字段 | 轮数 |
|------|---------|-------------|------|
| **QARMA5** | `aa64_pauth_qarma5` | ID_AA64ISAR1.APA | 5 轮 |
| **QARMA3** | `aa64_pauth_qarma3` | ID_AA64ISAR1.APA3 | 3 轮 |
| **IMPDEF** | 默认 | ID_AA64ISAR1.API | N/A |

### 4.2 QARMA 架构化算法

```c
// pauth_helper.c:227-306
static uint64_t pauth_computepac_architected(uint64_t data, uint64_t modifier,
                                             ARMPACKey key, bool isqarma3)
```

QARMA 是一种轻量级可调分组密码，核心流程：

1. **前向轮运算**：`iterations+1` 轮，每轮包含 `SubBytes (pac_sub)` → `ShuffleCell (pac_cell_shuffle)` → `MixColumn (pac_mult)` → 密钥/轮常数异或
2. **中间轮**：额外的 Shuffle + Mult + Sub 操作
3. **逆向轮运算**：对称的逆过程
4. **输出**：最终值异或 `modk0` 得到 PAC

QARMA3 与 QARMA5 的唯一区别是轮数（`iterations = isqarma3 ? 2 : 4`），以及使用 `pac_sub1()` 替代 `pac_sub()`/`pac_inv_sub()`。

### 4.3 IMPDEF 快速算法

```c
// pauth_helper.c:309-312
static uint64_t pauth_computepac_impdef(uint64_t data, uint64_t modifier,
                                        ARMPACKey key)
{
    return qemu_xxhash64_4(data, modifier, key.lo, key.hi);
}
```

QEMU 的 IMPDEF 实现使用 xxHash64 哈希算法，将 `data`、`modifier` 和 128 位密钥作为输入，性能远优于 QARMA 但安全性较低。

---

## 5. PAC 指令翻译

### 5.1 签名与认证指令

PAC 指令在 `translate-a64.c` 中翻译，每条指令映射到对应的 helper 函数：

| 指令 | Helper | 密钥 | 数据/指令 | 源码位置 |
|------|--------|------|----------|---------|
| PACIA | `helper_pacia` | APIA | 指令 | pauth_helper.c:490-497 |
| PACIB | `helper_pacib` | APIB | 指令 | pauth_helper.c:500-507 |
| PACDA | `helper_pacda` | APDA | 数据 | pauth_helper.c:510-517 |
| PACDB | `helper_pacdb` | APDB | 数据 | pauth_helper.c:520-527 |
| PACGA | `helper_pacga` | APGA | 通用 | pauth_helper.c:530+ |
| AUTIA | `helper_autia` | APIA | 指令 | 对称认证 |
| AUTIB | `helper_autib` | APIB | 指令 | 对称认证 |
| AUTDA | `helper_autda` | APDA | 数据 | 对称认证 |
| AUTDB | `helper_autdb` | APDB | 数据 | 对称认证 |
| XPACI | `pauth_strip` | — | 指令 | pauth_helper.c:450-455 |
| XPACD | `pauth_strip` | — | 数据 | pauth_helper.c:450-455 |

### 5.2 HINT 空间中的 PAC 指令

系统级 PAC 指令编码在 HINT 空间（`translate-a64.c:2091-2215`）：

- **PACIASP / PACIBSP**：使用 SP 作为 modifier 签名 LR（X30）
- **AUTIASP / AUTIBSP**：使用 SP 认证 LR
- **PACIAZ / PACIBZ**：使用零作为 modifier
- **PACIA1716 / AUTIA1716**：使用 X17 签名/认证 X16

这些指令仅在 `s->pauth_active` 为真时生成 helper 调用（`translate-a64.c:8798-8821`），否则为 NOP。

### 5.3 PAC 核心操作

**签名（addpac）**：

```c
// pauth_helper.c:327-378
static uint64_t pauth_addpac(CPUARMState *env, uint64_t ptr,
                             uint64_t modifier, ARMPACKey *key, bool data)
```

1. 通过 `aa64_va_parameters()` 获取地址参数（TBI、tsz）
2. 计算 PAC 字段范围：`bot_bit = 64 - tsz`，`top_bit = 64 - 8*tbi`
3. 调用 `pauth_computepac()` 计算 PAC 值
4. 将 PAC 嵌入指针高位

**剥离（strip）**：

```c
// pauth_helper.c:450-455
static uint64_t pauth_strip(CPUARMState *env, uint64_t ptr, bool data)
```

恢复指针原始值（符号扩展 bit55）。

---

## 6. PAC 认证分支指令

### 6.1 认证分支翻译

ARMv8.3 引入了认证分支指令族，在分支前自动执行 PAC 认证（`translate-a64.c:1837-1948`）：

```c
// translate-a64.c:1837-1856
static void auth_branch_target(DisasContext *s, TCGv_i64 dst,
                                TCGv_i64 modifier, bool use_key_a)
```

| 指令 | 密钥 | 行为 |
|------|------|------|
| **BRAA/BRAAZ** | A | 认证后 BR |
| **BRAB/BRABZ** | B | 认证后 BR |
| **BLRAA/BLRAAZ** | A | 认证后 BLR |
| **BLRAB/BLRABZ** | B | 认证后 BLR |
| **RETAA** | A | 认证后 RET |
| **RETAB** | B | 认证后 RET |
| **ERETAA** | A | 认证后 ERET |
| **ERETAB** | B | 认证后 ERET |

### 6.2 Combined 认证路径

认证分支使用 "combined" 模式（`is_combined=true`），这影响 FEAT_FPACCOMBINE 的行为：

```c
// pauth_helper.c:427-436
if (pauth_feature >= PauthFeat_2) {
    ARMPauthFeature fault_feature =
        is_combined ? PauthFeat_FPACCOMBINED : PauthFeat_FPAC;
    uint64_t result = ptr ^ (pac & cmp_mask);
    if (pauth_feature >= fault_feature
        && ((result ^ sextract64(result, 55, 1)) & cmp_mask)) {
        pauth_fail_exception(env, data, keynumber, ra);
    }
    return result;
}
```

- **FEAT_FPAC**：独立 AUT 指令认证失败时产生异常
- **FEAT_FPACCOMBINE**：认证分支中认证失败时产生异常

---

## 7. PAC 使能控制与陷入

### 7.1 SCTLR 使能位

每个密钥由对应的 SCTLR 位独立控制：

```c
// pauth_helper.c:485-497 (以 PACIA 为例)
uint64_t HELPER(pacia)(CPUARMState *env, uint64_t x, uint64_t y)
{
    int el = arm_current_el(env);
    if (!pauth_key_enabled(env, el, SCTLR_EnIA)) {
        return x;  // 未使能则直接返回原值（NOP）
    }
    pauth_check_trap(env, el, GETPC());
    return pauth_addpac(env, x, y, &env->keys.apia, false);
}
```

执行流程：
1. 检查 `SCTLR.EnXX` → 未使能则 NOP
2. 检查陷入条件 → 可能陷入 EL2/EL3
3. 执行 PAC 运算

### 7.2 EL2/EL3 陷入控制

```c
// pauth_helper.c:464-482
static void pauth_check_trap(CPUARMState *env, int el, uintptr_t ra)
{
    if (el < 2 && arm_is_el2_enabled(env)) {
        uint64_t hcr = arm_hcr_el2_eff(env);
        bool trap = !(hcr & HCR_API);
        // ...
        if (trap) {
            pauth_trap(env, 2, ra);  // 陷入 EL2
        }
    }
    if (el < 3 && arm_feature(env, ARM_FEATURE_EL3)) {
        if (!(env->cp15.scr_el3 & SCR_API)) {
            pauth_trap(env, 3, ra);  // 陷入 EL3
        }
    }
}
```

两级陷入控制：
- **HCR_EL2.API=0**：EL0/EL1 的 PAC 指令陷入 EL2
- **SCR_EL3.API=0**：EL0/EL1/EL2 的 PAC 指令陷入 EL3

### 7.3 TB 标志优化

```c
// hflags.c:320-329
if (cpu_isar_feature(aa64_pauth, env_archcpu(env))) {
    if (sctlr & (SCTLR_EnIA | SCTLR_EnIB | SCTLR_EnDA | SCTLR_EnDB)) {
        DP_TBFLAG_A64(flags, PAUTH_ACTIVE, 1);
    }
}
```

仅当至少一个密钥使能时，才设置 `PAUTH_ACTIVE`。翻译器据此决定是否生成 PAC helper 调用，禁用时所有 PAC 指令为 NOP，零开销。

---

## 8. PAC 认证失败处理

### 8.1 FEAT_PAuth2 / FEAT_FPAC 路径

```c
// pauth_helper.c:408-448
static uint64_t pauth_auth(CPUARMState *env, uint64_t ptr, ...)
{
    // ...
    if (pauth_feature >= PauthFeat_2) {
        uint64_t result = ptr ^ (pac & cmp_mask);  // 去除 PAC 并恢复指针
        if (pauth_feature >= fault_feature
            && ((result ^ sextract64(result, 55, 1)) & cmp_mask)) {
            // FEAT_FPAC: 认证失败直接产生异常
            pauth_fail_exception(env, data, keynumber, ra);
        }
        return result;
    }
    // 基础 PAuth: 认证失败不产生异常，而是破坏指针
    if ((pac ^ ptr) & cmp_mask) {
        int error_code = (keynumber << 1) | (keynumber ^ 1);
        if (param.tbi) {
            return deposit64(orig_ptr, 53, 2, error_code);  // 在 bit[54:53] 嵌入错误码
        } else {
            return deposit64(orig_ptr, 61, 2, error_code);  // 在 bit[62:61] 嵌入错误码
        }
    }
    return orig_ptr;
}
```

| 特性级别 | 失败行为 | 异常类 |
|---------|---------|--------|
| 基础 PAuth | 破坏指针高位（使用时触发地址错误） | 无直接异常 |
| PAuth2 | XOR 恢复但不检查 | 无直接异常 |
| FPAC | 立即产生 `EXCP_UDEF` | `syn_pacfail(data, keynumber)` |
| FPACCOMBINE | 认证分支中立即产生异常 | 同上 |

### 8.2 异常生成

```c
// pauth_helper.c:400-405
static G_NORETURN
void pauth_fail_exception(CPUARMState *env, bool data,
                          int keynumber, uintptr_t ra)
{
    raise_exception_ra(env, EXCP_UDEF, syn_pacfail(data, keynumber),
                       exception_target_el(env), ra);
}
```

PAC 认证失败异常路由到 `exception_target_el(env)` 确定的目标 EL。

---

## 9. BTI 分支目标识别

### 9.1 基本原理

BTI 通过限制间接分支的合法目标来防御 JOP 攻击：

1. 页表中的 **GP（Guarded Page）位**标记受保护的代码页
2. 间接分支设置 **PSTATE.BTYPE** 指示分支类型
3. 目标指令必须是合法的 **着陆点**（BTI 指令或 PACI*SP），否则产生异常

### 9.2 BTYPE 值定义

| BTYPE | 含义 | 设置场景 |
|-------|------|---------|
| 0 | 无活跃分支检查 | 顺序执行 / 直接分支 |
| 1 | BR 到 x16/x17 或非保护页 | `BR Xn`（n=16/17）|
| 2 | BLR（带链接间接分支）| `BLR Xn` |
| 3 | BR 到保护页（非 x16/x17）| `BR Xn`（n≠16/17，目标在保护页）|

### 9.3 BTYPE 存储

```c
// cpu.h:313 (注释)
// BTYPE is PSTATE bits [11:10]
// 实际存储在 env->btype 中
```

与其他 PSTATE 位类似，BTYPE 不存储在 `env->pstate` 中，而是独立字段 `env->btype`，便于 TCG 高效访问。

---

## 10. BTI BTYPE 状态机

### 10.1 分支指令设置 BTYPE

**BR 指令**（`translate-a64.c:1777-1789`）：

```c
static void set_btype_for_br(DisasContext *s, int rn)
{
    if (dc_isar_feature(aa64_bti, s)) {
        if (rn == 16 || rn == 17) {
            set_btype(s, 1);          // x16/x17: BTYPE=1
        } else {
            // 运行时检查目标页是否为保护页
            gen_helper_guarded_page_br(tcg_env, pc);
            s->btype = -1;            // 延迟确定
        }
    }
}
```

对于非 x16/x17 的 BR，需要运行时查询目标页的 GP 位：
- 保护页 → BTYPE=3
- 非保护页 → BTYPE=1

```c
// helper-a64.c:1768-1774
void HELPER(guarded_page_br)(CPUARMState *env, vaddr pc)
{
    env->btype = is_guarded_page(env, pc, GETPC()) ? 3 : 1;
}
```

**BLR 指令**（`translate-a64.c:1792-1797`）：

```c
static void set_btype_for_blr(DisasContext *s)
{
    if (dc_isar_feature(aa64_bti, s)) {
        set_btype(s, 2);  // BLR: 始终 BTYPE=2
    }
}
```

### 10.2 着陆点兼容性检查

TB 起始处检查第一条指令是否为合法着陆点（`translate-a64.c:10617-10651`）：

```c
static bool btype_destination_ok(uint32_t insn, bool bt, int btype)
{
    // HINT 空间中的着陆点
    switch (extract32(insn, 5, 7)) {
    case 0b011001: /* PACIASP */
    case 0b011011: /* PACIBSP */
        return !bt || btype != 3;    // BT=1 时不兼容 BTYPE=3
    case 0b100000: /* BTI */
        return false;                 // 裸 BTI 不兼容任何 BTYPE
    case 0b100010: /* BTI c */
        return btype != 3;           // 兼容 BTYPE=1,2
    case 0b100100: /* BTI j */
        return btype != 2;           // 兼容 BTYPE=1,3
    case 0b100110: /* BTI jc */
        return true;                  // 兼容所有 BTYPE
    }
    // BRK / HLT 优先处理断点
    ...
}
```

兼容性矩阵：

| 着陆点 | BTYPE=1 (BR) | BTYPE=2 (BLR) | BTYPE=3 (BR guarded) |
|--------|:-----------:|:------------:|:-------------------:|
| BTI | ✗ | ✗ | ✗ |
| BTI c | ✓ | ✓ | ✗ |
| BTI j | ✓ | ✗ | ✓ |
| BTI jc | ✓ | ✓ | ✓ |
| PACIASP (BT=0) | ✓ | ✓ | ✓ |
| PACIASP (BT=1) | ✓ | ✓ | ✗ |

### 10.3 TB 起始 BTI 检查

```c
// translate-a64.c:10846-10853
if (s->btype != 0
    && !btype_destination_ok(insn, s->bt, s->btype)) {
    gen_helper_guarded_page_check(tcg_env);
}
```

不兼容时调用 helper 进行最终的保护页确认：

```c
// helper-a64.c:1754-1765
void HELPER(guarded_page_check)(CPUARMState *env)
{
    if (is_guarded_page(env, env->pc, 0)) {
        raise_exception(env, EXCP_UDEF, syn_btitrap(env->btype),
                        exception_target_el(env));
    }
}
```

关键设计：BTI 检查分两阶段——翻译时静态检查指令兼容性，运行时动态检查页是否为保护页。非保护页上所有分支目标都合法。

---

## 11. BTI 保护页检查

### 11.1 GP 位来源

保护页标志来自页表属性（`ptw.c:2342-2345`）：

```c
result->f.extra.arm.guarded = extract64(attrs, 50, 1);
```

Stage 1 页表描述符的第 50 位即 GP（Guarded Page）位。

### 11.2 保护页查询

```c
// helper-a64.c:1738-1751
static bool is_guarded_page(CPUARMState *env, vaddr addr, uintptr_t ra)
{
#ifdef CONFIG_USER_ONLY
    return page_get_flags(addr) & PAGE_BTI;    // 用户模式：页标志
#else
    // 系统模式：通过 TLB 查询
    int flags = probe_access_full(env, addr, 0, MMU_INST_FETCH, mmu_idx,
                                  false, &host, &full, ra);
    return full->extra.arm.guarded;
#endif
}
```

### 11.3 BTI 异常

```c
// syndrome.h:43
EC_BTITRAP = 0x0d

// syndrome.h:439-446
static inline uint32_t syn_btitrap(int btype)
{
    return (EC_BTITRAP << ARM_EL_EC_SHIFT) | ARM_EL_IL | (btype & 3);
}
```

BTI 异常为 Undefined Instruction 异常（`EXCP_UDEF`），异常类 `EC_BTITRAP=0x0d`，ISS 字段包含触发的 BTYPE 值。

### 11.4 SCTLR.BT0/BT1

```c
// hflags.c:332-336
if (cpu_isar_feature(aa64_bti, env_archcpu(env))) {
    if (sctlr & (el == 0 ? SCTLR_BT0 : SCTLR_BT1)) {
        DP_TBFLAG_A64(flags, BT, 1);
    }
}
```

`SCTLR.BT0` 控制 EL0，`SCTLR.BT1` 控制 EL1（EL2/EL3 使用 BT1）。BT=1 时，PACIASP/PACIBSP 不再兼容 BTYPE=3，强制要求使用显式 BTI 指令。

---

## 12. MTE 内存标签扩展

### 12.1 基本原理

MTE 为每 16 字节（TAG_GRANULE）的物理内存分配一个 4-bit **分配标签（Allocation Tag）**，同时在虚拟地址的高位 [59:56] 携带一个 4-bit **逻辑标签（Logical Tag）**：

```
63  60 59 56 55               0
+-----+----+-----------------+
| TBI | LT |   虚拟地址低位    |
+-----+----+-----------------+
  ^     ^
  |     +-- 逻辑标签 (4-bit)
  +-------- Top Byte Ignore 区域
```

每次内存访问时，硬件将逻辑标签与分配标签进行比较。不匹配则根据配置产生同步/异步故障。

### 12.2 标签提取

```c
// internals.h:1620-1623
static inline int allocation_tag_from_addr(uint64_t ptr)
{
    return extract64(ptr, 56, 4);  // 提取 bits [59:56]
}

// internals.h:1625-1628
static inline uint64_t address_with_allocation_tag(uint64_t ptr, int rtag)
{
    return deposit64(ptr, 56, 4, rtag);  // 写入 bits [59:56]
}
```

### 12.3 TBI 与 TCMA 交互

```c
// internals.h:1630-1646
static inline bool tbi_check(uint32_t desc, int bit55)
{
    return (desc >> (R_MTEDESC_TBI_SHIFT + bit55)) & 1;
}

static inline bool tcma_check(uint32_t desc, int bit55, int ptr_tag)
{
    bool match = ((ptr_tag + bit55) & 0xf) == 0;  // 全0或全1标签
    bool tcma = (desc >> (R_MTEDESC_TCMA_SHIFT + bit55)) & 1;
    return tcma && match;
}
```

- **TBI 禁用**：标签位参与地址翻译，MTE 检查无效
- **TCMA 使能**：标签为全 0（`0x0`）或全 1（`0xF`）时跳过检查

---

## 13. MTE 标签存储架构

### 13.1 用户模式标签存储

```c
// mte_helper.c:66-92 (user-only 路径)
uint8_t *tags = page_get_target_data(clean_ptr, page_data_size);
index = extract32(ptr, LOG2_TAG_GRANULE + 1,
                  TARGET_PAGE_BITS - LOG2_TAG_GRANULE - 1);
return tags + index;
```

在用户模式下，每个页面有一块 `target_data`，每字节存储两个标签（每个标签 4 bit），一个页面（4KB）有 256 个 16 字节粒度，需要 128 字节标签存储。

要求页面同时具有 `PAGE_ANON` 和 `PAGE_MTE` 标志。

### 13.2 系统模式标签存储

```c
// cpu.c:1630-1644
// 标签内存通过独立地址空间 cpu->tag_memory / secure-tag-memory 访问
```

系统模式下，标签存储在独立的物理地址空间中。`allocation_tag_mem_probe()` 的流程：

1. 对原始 VA 进行 TLB 查询，获取物理地址和属性
2. 检查内存类型是否为 "Tagged Normal Memory"（`arm_tlb_mte_tagged()`）
3. 将物理地址转换为标签地址空间中的偏移
4. 通过 `address_space_translate()` 获取标签 RAM 指针

```c
// mte_helper.c:61-196
uint8_t *allocation_tag_mem_probe(CPUARMState *env, int ptr_mmu_idx,
                                  uint64_t ptr, ...)
{
    // 系统模式：TLB probe → 检查 Tagged Normal → 标签地址空间查找
    ...
    tag_asi = cyclecount ? ARMASIdx_TagDC : ARMASIdx_TagNS;
    tag_as = cpu_get_address_space(env_cpu(env), tag_asi);
    ...
}
```

---

## 14. MTE 标签检查流程

### 14.1 翻译时插入检查

```c
// translate-a64.c:300-321
static TCGv_i64 gen_mte_check1_mmuidx(DisasContext *s, TCGv_i64 addr,
                                       bool is_write, bool tag_checked,
                                       MemOp memop, bool is_unpriv, int core_idx)
{
    if (tag_checked && s->mte_active[is_unpriv]) {
        int desc = 0;
        desc = FIELD_DP32(desc, MTEDESC, MIDX, core_idx);
        desc = FIELD_DP32(desc, MTEDESC, TBI, s->tbid);
        desc = FIELD_DP32(desc, MTEDESC, TCMA, s->tcma);
        desc = FIELD_DP32(desc, MTEDESC, WRITE, is_write);
        desc = FIELD_DP32(desc, MTEDESC, ALIGN, memop_alignment_bits(memop));
        desc = FIELD_DP32(desc, MTEDESC, SIZEM1, memop_size(memop) - 1);

        ret = tcg_temp_new_i64();
        gen_helper_mte_check(ret, tcg_env, tcg_constant_i32(desc), addr);
        return ret;
    }
    return clean_data_tbi(s, addr);  // MTE 未激活：仅清理 TBI
}
```

`MTEDESC` 描述符打包了检查所需的所有参数：MMU 索引、TBI、TCMA、读写标志、对齐要求、访问大小。

### 14.2 运行时检查核心

```c
// mte_helper.c:869-902
uint64_t HELPER(mte_check)(CPUARMState *env, uint32_t desc, uint64_t ptr)
{
    // 1. 对齐检查（优先级高于翻译错误）
    unsigned align = FIELD_EX32(desc, MTEDESC, ALIGN);
    if (unlikely(align)) {
        align = (1u << align) - 1;
        if (unlikely(ptr & align)) {
            arm_cpu_do_unaligned_access(...);
        }
    }
    // 2. MTE 标签检查
    return mte_check(env, desc, ptr, GETPC());
}
```

### 14.3 标签比较逻辑

```c
// mte_helper.c:779-866
static int mte_probe_int(CPUARMState *env, uint32_t desc, uint64_t ptr, ...)
{
    // 1. TBI 检查
    if (unlikely(!tbi_check(desc, bit55))) return -1;

    // 2. 提取逻辑标签
    ptr_tag = allocation_tag_from_addr(ptr);  // bits[59:56]

    // 3. TCMA 检查
    if (tcma_check(desc, bit55, ptr_tag)) return 1;

    // 4. 计算标签粒度数量
    tag_count = ((tag_last - tag_first) / TAG_GRANULE) + 1;

    // 5. 获取分配标签内存
    mem1 = allocation_tag_mem(env, mmu_idx, ptr, type, ...);
    if (!mem1) return 1;  // 非 Tagged 内存，检查通过

    // 6. 逐粒度比较
    n = checkN(mem1, ptr & TAG_GRANULE, ptr_tag, tag_count);

    if (likely(n == tag_count)) return 1;  // 全部匹配
    // 失败：记录故障地址
    *fault = tag_first + n * TAG_GRANULE;
    return 0;
}
```

`checkN()` 函数逐个比较每个 TAG_GRANULE（16 字节）的分配标签与逻辑标签，支持跨页检查。

---

## 15. MTE 故障处理机制

### 15.1 故障类型选择

```c
// mte_helper.c:600-653
void mte_check_fail(CPUARMState *env, uint32_t desc,
                    uint64_t dirty_ptr, uintptr_t ra)
{
    // 根据 EL 选择 TCF 位
    switch (arm_mmu_idx) {
    case ARMMMUIdx_E10_0:
    case ARMMMUIdx_E20_0:
        el = 0;
        tcf = extract64(sctlr, 38, 2);  // SCTLR.TCF0 (EL0)
        break;
    default:
        el = reg_el;
        tcf = extract64(sctlr, 40, 2);  // SCTLR.TCF (EL1+)
    }

    switch (tcf) {
    case 0: g_assert_not_reached();     // TCF=0 不应到达此处
    case 1: mte_sync_check_fail(...);   // 同步故障
    case 2: mte_async_check_fail(...);  // 异步标志
    case 3:                              // 非对称：读同步/写异步
        if (FIELD_EX32(desc, MTEDESC, WRITE)) {
            mte_async_check_fail(...);
        } else {
            mte_sync_check_fail(...);
        }
    }
}
```

| TCF 值 | 行为 | 说明 |
|--------|------|------|
| 0 | 无效果 | QEMU 通过不设置 `MTE_ACTIVE` 避免运行时调用 |
| 1 | 同步数据中止 | 立即产生异常，DFSC=0x11 |
| 2 | 异步标志 | 设置 `TFSR_EL1` 标志，不中断执行 |
| 3 | 非对称 | 写操作异步，读操作同步（FEAT_MTE3）|

### 15.2 同步故障

```c
// mte_helper.c:563-574
static void mte_sync_check_fail(CPUARMState *env, uint32_t desc,
                                uint64_t dirty_ptr, uintptr_t ra)
{
    env->exception.vaddress = dirty_ptr;  // 记录故障地址（含标签）
    syn = syn_data_abort_no_iss(arm_current_el(env) != 0, 0, 0, 0, 0,
                                is_write, 0x11);  // DFSC=0x11
    raise_exception_ra(env, EXCP_DATA_ABORT, syn, exception_target_el(env), ra);
}
```

同步标签检查故障产生 Data Abort，DFSC（Data Fault Status Code）= `0x11`（Synchronous Tag Check Fault）。

### 15.3 异步故障

```c
// mte_helper.c:577-597
static void mte_async_check_fail(CPUARMState *env, uint64_t dirty_ptr,
                                 uintptr_t ra, ARMMMUIdx arm_mmu_idx, int el)
{
    int select = extract64(dirty_ptr, 55, 1);  // 地址空间选择
    env->cp15.tfsr_el[el] |= 1 << select;     // 设置 TFSR 标志
}
```

异步模式仅设置 `TFSR_ELx` 中的标志位，由软件在合适的时机处理（如中断返回时检查）。

---

## 16. MTE 随机标签生成

### 16.1 IRG 指令实现

```c
// mte_helper.c:209-254
uint64_t HELPER(irg)(CPUARMState *env, uint64_t rn, uint64_t rm)
{
    uint16_t exclude = extract32(rm | env->cp15.gcr_el1, 0, 16);
    int start = extract32(env->cp15.rgsr_el1, 0, 4);
    int seed = extract32(env->cp15.rgsr_el1, 8, 16);
    int offset, i, rtag;

    // 16-bit LFSR 生成随机偏移
    for (i = offset = 0; i < 4; ++i) {
        int top = (extract32(seed, 5, 1) ^ extract32(seed, 3, 1) ^
                   extract32(seed, 2, 1) ^ extract32(seed, 0, 1));
        seed = (top << 15) | (seed >> 1);
        offset |= top << i;
    }

    rtag = choose_nonexcluded_tag(start, offset, exclude);
    env->cp15.rgsr_el1 = rtag | (seed << 8);  // 更新种子

    return address_with_allocation_tag(rn, rtag);
}
```

IRG 随机标签生成流程：

1. **排除掩码**：`rm | GCR_EL1` 低 16 位，每位对应一个标签值（0-15），置 1 表示排除
2. **LFSR 随机**：16-bit 线性反馈移位寄存器，多项式为 `x^16 + x^6 + x^4 + x^3 + 1`
3. **标签选择**：`choose_nonexcluded_tag()` 从非排除标签中选择
4. **状态更新**：新标签和种子写回 `RGSR_EL1`

### 16.2 GCR_EL1 与排除机制

`GCR_EL1` 低 16 位（Exclude）控制哪些标签值被排除。例如 `GCR_EL1.Exclude = 0x0001` 表示排除标签 0，CPU 复位时初始化为 `0x1ffff`（排除所有标签，仅允许标签 0）。

### 16.3 ADDG / SUBG

```c
// mte_helper.c:257-264
uint64_t HELPER(addsubg)(CPUARMState *env, uint64_t ptr,
                         int32_t offset, uint32_t tag_offset)
{
    int start_tag = allocation_tag_from_addr(ptr);
    uint16_t exclude = extract32(env->cp15.gcr_el1, 0, 16);
    int rtag = choose_nonexcluded_tag(start_tag, tag_offset, exclude);
    return address_with_allocation_tag(ptr + offset, rtag);
}
```

ADDG/SUBG 基于当前标签加偏移计算新标签，同时应用排除掩码，并调整指针地址。

---

## 17. 三大特性的 TB 标志集成

### 17.1 PAC TB 标志

```c
// hflags.c:320-329
PAUTH_ACTIVE  // 任一 SCTLR.EnXX 使能时置位
```

单个标志控制所有 PAC 指令的翻译：
- 置位：生成 helper 调用
- 清除：所有 PAC 指令编译为 NOP

### 17.2 BTI TB 标志

```c
// hflags.c:332-336
BT     // SCTLR.BT0/BT1 置位时，影响 PACIASP 兼容性
BTYPE  // 从 env->btype 加载（每个 TB 入口检查）
```

### 17.3 MTE TB 标志

```c
// hflags.c:402-435
ATA          // Allocation Tag Access 使能
ATA0         // EL0 的 ATA（用于 LDTR 等非特权访问）
MTE_ACTIVE   // 特权访问的 MTE 检查激活
MTE0_ACTIVE  // EL0 访问的 MTE 检查激活
```

MTE 激活条件（四个条件必须同时满足）：

1. `TBI` 开启（地址高字节被忽略）
2. `PSTATE.TCO` = 0（未被覆盖）
3. `SCTLR.TCF/TCF0` ≠ 0（故障模式非忽略）
4. `allocation_tag_access_enabled()` 为真

### 17.4 MTEDESC 描述符

MTE 检查通过紧凑的 `MTEDESC` 描述符传递参数，避免多参数 helper 调用：

| 字段 | 含义 |
|------|------|
| MIDX | MMU 索引 |
| TBI | TBI0/TBI1 标志 |
| TCMA | TCMA0/TCMA1 标志 |
| WRITE | 写操作标志 |
| ALIGN | 对齐要求（2 的幂次） |
| SIZEM1 | 访问大小减 1 |

---

## 18. 特性检测与 CPU 属性

### 18.1 PAC 特性

```c
// cpu-features.h:922-929
typedef enum {
    PauthFeat_None,
    PauthFeat_1,
    PauthFeat_2,              // FEAT_PAuth2
    PauthFeat_FPAC,           // FEAT_FPAC
    PauthFeat_FPACCOMBINED,   // FEAT_FPACCOMBINE
} ARMPauthFeature;

// cpu-features.h:943-967
aa64_pauth           // ID_AA64ISAR1.APA/API/APA3 != 0
aa64_pauth_qarma5    // ID_AA64ISAR1.APA != 0
aa64_pauth_qarma3    // ID_AA64ISAR1.APA3 != 0
```

### 18.2 BTI 特性

```c
// cpu-features.h:1140-1142
aa64_bti             // ID_AA64PFR1.BT != 0
```

BTI 是单级特性，通过 `ID_AA64PFR1.BT` 字段检测。

### 18.3 MTE 特性

```c
// cpu-features.h:1145-1157
aa64_mte_insn_reg    // ID_AA64PFR1.MTE != 0  (FEAT_MTE: 仅指令/寄存器)
aa64_mte             // ID_AA64PFR1.MTE >= 2   (FEAT_MTE2: 完整标签检查)
aa64_mte3            // ID_AA64PFR1.MTE >= 3   (FEAT_MTE3: 非对称检查)
```

| 级别 | MTE 值 | 新增能力 |
|------|--------|---------|
| FEAT_MTE | 1 | IRG/ADDG/SUBG/GMI 指令，GCR/RGSR/TFSR 寄存器 |
| FEAT_MTE2 | 2 | 完整标签检查，STG/LDG 系列，同步/异步故障 |
| FEAT_MTE3 | 3 | 非对称故障模式（TCF=3: 读同步/写异步）|

---

## 19. 总结

### 三大特性协同关系

```
+------------------+     +------------------+     +------------------+
|       PAC        |     |       BTI        |     |       MTE        |
| 指针认证码       |     | 分支目标识别     |     | 内存标签扩展     |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
    指针完整性              控制流完整性              内存空间安全
         |                        |                        |
    防御 ROP/JOP            防御 JOP               防御 UAF/溢出
         |                        |                        |
   签名嵌入指针高位         分支类型+着陆点          地址标签+分配标签
         |                        |                        |
  QARMA/xxHash 算法        BTYPE 状态机          TAG_GRANULE=16B
         |                        |                        |
  5 组 128-bit 密钥         GP 页表属性            独立标签地址空间
+--------+---------+     +--------+---------+     +--------+---------+
| pauth_helper.c   |     | translate-a64.c  |     | mte_helper.c     |
| ~530 行           |     | helper-a64.c     |     | ~900 行           |
+------------------+     +------------------+     +------------------+
```

### 关键源码文件索引

| 文件 | 关键行号 | 内容 |
|------|---------|------|
| cpu.h | 180-182 | ARMPACKey 结构定义 |
| cpu.h | 715-721 | CPUARMState.keys 密钥存储 |
| pauth_helper.c | 227-306 | QARMA 算法实现 |
| pauth_helper.c | 309-324 | PAC 算法选择 |
| pauth_helper.c | 327-378 | pauth_addpac() 签名 |
| pauth_helper.c | 408-448 | pauth_auth() 认证/失败 |
| pauth_helper.c | 464-482 | EL2/EL3 陷入检查 |
| pauth_helper.c | 490-527 | PACIA/PACIB/PACDA/PACDB helper |
| translate-a64.c | 1777-1797 | set_btype_for_br/blr |
| translate-a64.c | 1837-1948 | 认证分支翻译 |
| translate-a64.c | 10617-10651 | btype_destination_ok() |
| translate-a64.c | 10846-10853 | TB 起始 BTI 检查 |
| translate-a64.c | 300-354 | gen_mte_check1/checkN |
| helper-a64.c | 1738-1774 | 保护页查询与 BTI 异常 |
| mte_helper.c | 61-196 | allocation_tag_mem_probe() |
| mte_helper.c | 209-254 | HELPER(irg) 随机标签 |
| mte_helper.c | 563-653 | MTE 故障处理 |
| mte_helper.c | 779-902 | mte_probe_int / mte_check |
| hflags.c | 320-435 | PAC/BTI/MTE TB 标志设置 |
| internals.h | 1620-1646 | 标签提取/TBI/TCMA 辅助函数 |
| cpu-features.h | 922-967 | PAC 特性检测 |
| cpu-features.h | 1140-1157 | BTI/MTE 特性检测 |
| syndrome.h | 43, 439-446 | BTI 异常综合征 |

---

> 文档生成时间：基于 QEMU 11.0.50 源码分析
> 分析工具：zoekt + ctags + cscope 索引
