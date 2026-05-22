# ARM64 Pointer Authentication (PAC) 完整实现分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/76 |
| 分析对象 | FEAT_PAuth (Pointer Authentication) 在 QEMU 中的完整实现 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §D5.1 (Pointer Authentication) |
| 核心文件 | `target/arm/tcg/pauth_helper.c` (632 行) |
| 核心结论 | **QEMU 实现了完整的 QARMA5/QARMA3/IMPDEF 三种 PAC 算法，支持 PauthFeat_1 到 PauthFeat_FPACCOMBINED 全部演进版本** |

---

## 1. PAC 架构概述

### 1.1 基本原理

Pointer Authentication (FEAT_PAuth, Armv8.3) 利用指针中未使用的高位嵌入认证码 (PAC)：

```
签名前 (原始指针):
┌──────────────────┬────────────────────────────────────────┐
│ 63:bot_bit (ext) │ bot_bit-1:0 (虚拟地址有效位)            │
└──────────────────┴────────────────────────────────────────┘

签名后 (带 PAC 的指针):
┌───┬──────────────┬────────────────────────────────────────┐
│55 │PAC (认证码)   │ bot_bit-1:0 (虚拟地址有效位)            │
└───┴──────────────┴────────────────────────────────────────┘
     ↑ bot_bit      ↑ top_bit (= 64 - 8*tbi)

PAC 位数 = top_bit - bot_bit (通常 7-16 位)
```

### 1.2 5 组密钥

```c
// target/arm/cpu.h:180
typedef struct ARMPACKey {
    uint64_t lo, hi;  // 128-bit 密钥
} ARMPACKey;

// cpu.h:716 - 存储在 CPU 状态中
struct CPUARMState {
    struct {
        ARMPACKey apia;  // 指令地址 key A
        ARMPACKey apib;  // 指令地址 key B
        ARMPACKey apda;  // 数据地址 key A
        ARMPACKey apdb;  // 数据地址 key B
        ARMPACKey apga;  // 通用认证 (PACGA)
    } keys;
};
```

### 1.3 PAC 特性演进

```c
// target/arm/cpu-features.h:923
typedef enum {
    PauthFeat_None         = 0,  // 无 PAC
    PauthFeat_1            = 1,  // Armv8.3: 基础 PAC
    PauthFeat_EPAC         = 2,  // Enhanced PAC
    PauthFeat_2            = 3,  // PAuth2: XOR-based insertion
    PauthFeat_FPAC         = 4,  // Faulting PAC: auth 失败触发异常
    PauthFeat_FPACCOMBINED = 5,  // Combined: AUT+use 失败也触发异常
} ARMPauthFeature;
```

---

## 2. PAC 计算算法

### 2.1 算法选择

```c
// pauth_helper.c:315
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

### 2.2 QARMA5/QARMA3 算法 (规范定义)

QARMA 是 ARM 定义的轻量级可调块密码 (tweakable block cipher):

```c
// pauth_helper.c:227
static uint64_t pauth_computepac_architected(uint64_t data, uint64_t modifier,
                                             ARMPACKey key, bool isqarma3)
{
    int iterations = isqarma3 ? 2 : 4;  // QARMA3=2轮, QARMA5=4轮
    uint64_t key0 = key.hi, key1 = key.lo;

    // 前向变换 (Forward)
    modk0 = (key0 << 63) | ((key0 >> 1) ^ (key0 >> 63));
    workingval = data ^ key0;

    for (i = 0; i <= iterations; ++i) {
        workingval ^= key1 ^ runningmod ^ RC[i];
        if (i > 0) {
            workingval = pac_cell_shuffle(workingval);  // 64-bit 置换
            workingval = pac_mult(workingval);           // 4x4 MDS 矩阵
        }
        workingval = pac_sub(workingval);               // 4-bit S-box
        runningmod = tweak_shuffle(runningmod);          // modifier 演化
    }

    // 中间层 (Middle)
    workingval ^= modk0 ^ runningmod;
    workingval = pac_cell_shuffle → pac_mult → pac_sub
              → pac_cell_shuffle → pac_mult;
    workingval ^= key1;

    // 反向变换 (Backward) - 使用逆运算
    workingval = pac_cell_inv_shuffle → pac_inv_sub → pac_mult
              → pac_cell_inv_shuffle;
    for (i = 0; i <= iterations; ++i) {
        workingval = pac_inv_sub → pac_mult → pac_cell_inv_shuffle;
        workingval ^= key1 ^ runningmod ^ RC ^ alpha;
    }
    workingval ^= modk0;

    return workingval;
}
```

### 2.3 IMPDEF 算法 (QEMU 快速路径)

```c
// pauth_helper.c:309
static uint64_t pauth_computepac_impdef(uint64_t data, uint64_t modifier,
                                        ARMPACKey key)
{
    return qemu_xxhash64_4(data, modifier, key.lo, key.hi);
}
```

- 使用 xxhash64 作为 IMPDEF 实现
- 比 QARMA 快得多，适用于不要求密码强度的场景
- 通过 ID_AA64ISAR1.API 字段标识

### 2.4 QARMA 子组件

| 函数 | 功能 | 描述 |
|------|------|------|
| `pac_cell_shuffle` | 64-bit 置换 | 16 个 4-bit cell 重排 |
| `pac_cell_inv_shuffle` | 逆置换 | shuffle 的逆运算 |
| `pac_sub` / `pac_sub1` | S-box 替换 | 16-entry 4-bit lookup |
| `pac_inv_sub` | 逆 S-box | sub 的逆运算 |
| `pac_mult` | MDS 矩阵乘法 | 4x4 GF(2^4) 矩阵 |
| `tweak_shuffle` | modifier 演化 | 含 cell rotation |
| `rot_cell` | 4-bit 循环左移 | 用于 mult 和 tweak |

---

## 3. PAC 签名 (pauth_addpac)

### 3.1 签名流程

```c
// pauth_helper.c:327
static uint64_t pauth_addpac(CPUARMState *env, uint64_t ptr,
                             uint64_t modifier, ARMPACKey *key, bool data)
{
    // 1. 获取地址空间参数 (TBI, TSZ)
    ARMVAParameters param = aa64_va_parameters(env, ptr, mmu_idx, data, false);

    // 2. 确定 extension bit (用于区分高/低地址空间)
    ext = param.tbi ? sextract64(ptr, 55, 1) : sextract64(ptr, 63, 1);

    // 3. PAC 位范围
    top_bit = 64 - 8 * param.tbi;  // TBI=1 → top=56; TBI=0 → top=64
    bot_bit = 64 - param.tsz;       // 虚拟地址有效位顶部

    // 4. 构建 "干净" 指针 (扩展 PAC 区域)
    ext_ptr = deposit64(ptr, bot_bit, top_bit - bot_bit, ext);

    // 5. 计算 PAC
    pac = pauth_computepac(env, ext_ptr, modifier, *key);

    // 6. 检查指针是否已有坏 bits (双重签名检测)
    test = sextract64(ptr, bot_bit, top_bit - bot_bit);
    if (test != 0 && test != -1) {
        // 根据 PauthFeat 版本处理冲突
        if (pauth_feature >= PauthFeat_2) { /* 无操作 */ }
        else if (pauth_feature == PauthFeat_EPAC) { pac = 0; }
        else { pac ^= MAKE_64BIT_MASK(top_bit - 2, 1); }
    }

    // 7. PAuth2: PAC 与原始指针 XOR (而非直接插入)
    if (pauth_feature >= PauthFeat_2) {
        pac ^= ptr;
    }

    // 8. 组合结果: ptr 有效位 | PAC | bit55 (address select)
    return pac | ext | ptr;
}
```

### 3.2 PAC 位范围计算

```
典型配置 (48-bit VA, TBI=1):
  top_bit = 56 (bit[63:56] 被 TBI 忽略)
  bot_bit = 64 - 48 = 16... 不对

实际: tsz 来自 TCR_ELx.TnSZ
  例如 TCR.T0SZ = 16 → tsz = 48 → bot_bit = 64-48 = 16
  PAC bits = [55:16] = 40 位 (非常大的 PAC)

  但 TBI=1 时 top_bit = 56:
  PAC bits = [55:16] 中排除 bit[55] → 39 位 PAC

典型实际: TSZ=25 (39-bit VA):
  bot_bit = 64-39 = 25
  PAC bits = [55:25] = 31 位 (还是很大)
  通常 PAC 有效位 ≈ 7-24 位
```

---

## 4. PAC 认证 (pauth_auth)

### 4.1 认证流程

```c
// pauth_helper.c:408
static uint64_t pauth_auth(CPUARMState *env, uint64_t ptr, uint64_t modifier,
                           ARMPACKey *key, bool data, int keynumber, ...)
{
    // 1. 恢复原始指针 (去掉 PAC)
    orig_ptr = pauth_original_ptr(ptr, param);

    // 2. 用恢复的指针重新计算 PAC
    pac = pauth_computepac(env, orig_ptr, modifier, *key);

    // 3. 构建比较掩码 (排除 bit55)
    cmp_mask = MAKE_64BIT_MASK(bot_bit, top_bit - bot_bit);
    cmp_mask &= ~MAKE_64BIT_MASK(55, 1);

    // 4. PAuth2+: XOR 验证
    if (pauth_feature >= PauthFeat_2) {
        uint64_t result = ptr ^ (pac & cmp_mask);

        // FPAC/FPACCOMBINED: 验证失败直接异常
        if (pauth_feature >= fault_feature
            && ((result ^ sextract64(result, 55, 1)) & cmp_mask)) {
            pauth_fail_exception(env, data, keynumber, ra);
        }
        return result;
    }

    // 5. PAuth1: 直接比较
    if ((pac ^ ptr) & cmp_mask) {
        // 认证失败: 插入错误码使指针无效
        int error_code = (keynumber << 1) | (keynumber ^ 1);
        if (param.tbi) {
            return deposit64(orig_ptr, 53, 2, error_code);
        } else {
            return deposit64(orig_ptr, 61, 2, error_code);
        }
    }
    return orig_ptr;
}
```

### 4.2 认证失败的处理方式演进

| 版本 | 失败行为 |
|------|---------|
| PauthFeat_1 | 插入错误码到 bit[53:52] 或 [61:60], 后续使用触发 translation fault |
| PauthFeat_EPAC | PAC 清零, 同上 |
| PauthFeat_2 | XOR 恢复 (错误 PAC → 坏指针) |
| PauthFeat_FPAC | AUT 指令直接触发异常 (syn_pacfail) |
| PauthFeat_FPACCOMBINED | AUT+combined 使用 (如 RETAA) 也直接异常 |

---

## 5. Trap 和使能控制

### 5.1 SCTLR_ELx 使能位

```c
// pauth_helper.c:485
static bool pauth_key_enabled(CPUARMState *env, int el, uint32_t bit)
{
    return (arm_sctlr(env, el) & bit) != 0;
}

// 4 个独立使能:
SCTLR_EnIA  // 指令 key A (PACIA/AUTIA)
SCTLR_EnIB  // 指令 key B (PACIB/AUTIB)
SCTLR_EnDA  // 数据 key A (PACDA/AUTDA)
SCTLR_EnDB  // 数据 key B (PACDB/AUTDB)
```

### 5.2 EL2/EL3 Trap 控制

```c
// pauth_helper.c:464
static void pauth_check_trap(CPUARMState *env, int el, uintptr_t ra)
{
    if (el < 2 && arm_is_el2_enabled(env)) {
        // HCR_EL2.API = 0 → trap 到 EL2
        bool trap = !(hcr & HCR_API);
        if (el == 0) {
            trap &= (hcr & (HCR_E2H | HCR_TGE)) != (HCR_E2H | HCR_TGE);
        }
        if (trap) pauth_trap(env, 2, ra);
    }
    if (el < 3 && arm_feature(env, ARM_FEATURE_EL3)) {
        // SCR_EL3.API = 0 → trap 到 EL3
        if (!(env->cp15.scr_el3 & SCR_API)) {
            pauth_trap(env, 3, ra);
        }
    }
}
```

### 5.3 TB Flag 优化 (PAUTH_ACTIVE)

```c
// target/arm/tcg/hflags.c:320
if (cpu_isar_feature(aa64_pauth, env_archcpu(env))) {
    // 任一 key 使能 → PAUTH_ACTIVE=1
    if (sctlr & (SCTLR_EnIA | SCTLR_EnIB | SCTLR_EnDA | SCTLR_EnDB)) {
        DP_TBFLAG_A64(flags, PAUTH_ACTIVE, 1);
    }
}
```

翻译时：
- `PAUTH_ACTIVE=0`: PAC 指令翻译为 NOP (零开销)
- `PAUTH_ACTIVE=1`: 生成 helper 调用执行签名/验证

---

## 6. PAC 指令映射

### 6.1 签名指令

| 指令 | Helper | Key | Modifier |
|------|--------|-----|----------|
| PACIASP | pacia | APIA | SP (X31) |
| PACIAZ | pacia | APIA | 0 |
| PACIBSP | pacib | APIB | SP (X31) |
| PACIBZ | pacib | APIB | 0 |
| PACIA1716 | pacia | APIA | X17→X16 |
| PACDA | pacda | APDA | Xm |
| PACDB | pacdb | APDB | Xm |
| PACGA | pacga | APGA | (Xn,Xm)→Xd[63:32] |

### 6.2 认证指令

| 指令 | Helper | Key | 失败行为 |
|------|--------|-----|---------|
| AUTIASP | autia | APIA | 按版本处理 |
| AUTIAZ | autia | APIA | 按版本处理 |
| AUTIBSP | autib | APIB | 按版本处理 |
| RETAA | autia_combined | APIA | FPACCOMBINED → 异常 |
| RETAB | autib_combined | APIB | FPACCOMBINED → 异常 |
| BRAA | autia_combined | APIA | FPACCOMBINED → 异常 |

### 6.3 辅助指令

| 指令 | 功能 |
|------|------|
| XPACI | 去除 PAC, 恢复指令指针 |
| XPACD | 去除 PAC, 恢复数据指针 |

---

## 7. 指针恢复逻辑

### 7.1 pauth_original_ptr

```c
// pauth_helper.c:388
static uint64_t pauth_original_ptr(uint64_t ptr, ARMVAParameters param)
{
    uint64_t mask = pauth_ptr_mask(param);

    // bit55 决定高/低地址空间
    if (extract64(ptr, 55, 1)) {
        return ptr | mask;   // 高地址: PAC 区域填 1
    } else {
        return ptr & ~mask;  // 低地址: PAC 区域填 0
    }
}

// pauth_ptr_mask: PAC 区域的掩码
static inline uint64_t pauth_ptr_mask(ARMVAParameters param)
{
    int bot_pac_bit = 64 - param.tsz;
    int top_pac_bit = 64 - 8 * param.tbi;
    return MAKE_64BIT_MASK(bot_pac_bit, top_pac_bit - bot_pac_bit);
}
```

---

## 8. PACGA (通用认证)

```c
// pauth_helper.c:530
uint64_t HELPER(pacga)(CPUARMState *env, uint64_t x, uint64_t y)
{
    pauth_check_trap(env, arm_current_el(env), GETPC());
    pac = pauth_computepac(env, x, y, env->keys.apga);
    return pac & 0xffffffff00000000ull;  // 只返回高 32 位
}
```

- PACGA 不修改指针，只计算 MAC
- 用于通用数据完整性校验 (如 vtable 保护)
- 结果放在目标寄存器高 32 位

---

## 9. 与 ARM 规范的一致性评估

### 9.1 实现完整度

| 规范要求 | QEMU 状态 | 说明 |
|---------|:--------:|------|
| QARMA5 算法 (ID_AA64ISAR1.APA) | ✅ | 完整的 forward/middle/backward |
| QARMA3 算法 (ID_AA64ISAR2.APA3) | ✅ | iterations=2 |
| IMPDEF 算法 (ID_AA64ISAR1.API) | ✅ | xxhash64 |
| 5 组 128-bit 密钥 | ✅ | APIA/APIB/APDA/APDB/APGA |
| PauthFeat_1 错误码插入 | ✅ | deposit64(ptr, 53/61, 2, code) |
| PauthFeat_EPAC PAC 清零 | ✅ | pac = 0 |
| PauthFeat_2 XOR-based PAC | ✅ | pac ^= ptr |
| PauthFeat_FPAC 直接异常 | ✅ | pauth_fail_exception |
| PauthFeat_FPACCOMBINED | ✅ | is_combined 参数 |
| SCTLR.En{IA,IB,DA,DB} 使能 | ✅ | pauth_key_enabled |
| HCR_EL2.API trap | ✅ | pauth_check_trap |
| SCR_EL3.API trap | ✅ | pauth_check_trap |
| TBI 感知 PAC 位范围 | ✅ | aa64_va_parameters |
| bit55 地址空间选择 | ✅ | pauth_original_ptr |
| PACGA 通用认证 | ✅ | 高 32 位返回 |
| XPACI/XPACD strip | ✅ | pauth_strip |

### 9.2 算法验证

QEMU 的 QARMA 实现直接对应 ARM 规范伪代码：
- `pac_cell_shuffle` = CellShuffle (64-bit permutation)
- `pac_sub` / `pac_sub1` = SubCells (S-box, QARMA5 vs QARMA3 用不同 table)
- `pac_mult` = MixColumns (4x4 MDS matrix over GF(2^4))
- `tweak_shuffle` = TweakShuffle (运行中 modifier 演化)
- Round constants RC[0..4] = 规范定义的常量
- alpha = 0xC0AC29B7C97C50DD (规范定义)

### 9.3 与真实硬件的差异

| 方面 | 真实硬件 | QEMU |
|------|---------|------|
| 算法执行 | 专用硬件加速, ~1 cycle | 软件模拟 ~数十 cycles |
| 密钥保护 | 密钥寄存器受特权保护 | 存储在 CPUARMState 中 |
| 时序侧信道 | 恒定时间实现 | 软件无恒定时间保证 |
| PAC 位数 | 取决于 VA size 配置 | ✅ 相同计算逻辑 |
| 双重签名 | 硬件检测 | ✅ test != 0 && test != -1 |

### 9.4 总结

QEMU 的 PAC 实现**与规范完全一致**：
- 三种算法 (QARMA5/QARMA3/IMPDEF) 全部实现
- 五代特性演进 (PauthFeat_1 到 FPACCOMBINED) 完整支持
- Trap 层次 (EL2/EL3) 正确
- TB flag 优化使得未使能时零开销
- 唯一差异在性能（软件 vs 硬件加速）和侧信道安全性（功能模拟器不需要）
