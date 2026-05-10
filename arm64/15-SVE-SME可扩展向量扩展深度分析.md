# ARM64 SVE/SME 可扩展向量扩展深度分析

## 1. 概述

ARM64 SVE（Scalable Vector Extension）和 SME（Scalable Matrix Extension）是 ARMv8/v9 架构中的可扩展向量和矩阵指令集扩展。SVE 引入了向量长度不可知（VLA）编程模型，允许硬件实现 128-2048 位的向量宽度，程序无需修改即可运行。SME 进一步引入矩阵运算和流式 SVE（Streaming SVE）模式。QEMU 完整实现了 SVE/SVE2 和 SME/SME2 的仿真。

**关键源文件：**
| 文件 | 职责 |
|------|------|
| `cpu.h:165-190,670-754` | SVE/SME 寄存器存储定义 |
| `cpu.h:315,1583-1585` | SVCR (PSTATE.SM/ZA) |
| `cpu64.c:63-261,487-520,878-914` | SVE/SME CPU 属性与初始化 |
| `helper.c:4598-4641` | `sve_exception_el()` SVE 陷入 |
| `helper.c:4647-4700` | `sme_exception_el()` SME 陷入 |
| `helper.c:4733-4777` | VL 管理与 ZCR 寄存器 |
| `helper.c:4823-4860` | `arm_reset_sve_state()` / `aarch64_set_svcr()` |
| `helper.c:10029-10128` | `aarch64_sve_narrow_vq()` / `aarch64_sve_change_el()` |
| `hflags.c:141-165,240-308` | SME/SVE TB flags |
| `translate-sve.c` | SVE 指令翻译（~5400 行） |
| `translate-sme.c` | SME 指令翻译 |
| `sve_helper.c` | SVE 运行时 helper 函数 |
| `sve.decode` / `sme.decode` | SVE/SME 指令解码定义 |
| `cpu-features.h:1097-1100,1442-1479` | SVE/SVE2/SME 特性检测 |

## 2. SVE 寄存器架构

### 2.1 向量寄存器（Z0-Z31）

```c
// cpu.h:168-172
#define ARM_MAX_VQ    16    // 最大 16 个 128-bit 四字 = 2048 位

typedef struct ARMVectorReg {
    uint64_t d[2 * ARM_MAX_VQ] QEMU_ALIGNED(16);  // 32 个 uint64 = 2048 位
} ARMVectorReg;

// cpu.h:675
ARMVectorReg zregs[32];  // 32 个可扩展向量寄存器
```

**Z 寄存器与 SIMD V 寄存器的关系：**
```
Z[n] = | 高位（SVE 扩展部分，依赖 VL）| V[n] (128-bit) |
                                         ↑
                                   NEON Q[n] / D[2n]:D[2n+1]
```

SVE Z 寄存器的低 128 位与 NEON/SIMD 的 V/Q 寄存器共享同一存储（`ARMVectorReg` 的 `d[0]` 和 `d[1]`）。

### 2.2 谓词寄存器（P0-P15 + FFR）

```c
// cpu.h:175-177
typedef struct ARMPredicateReg {
    uint64_t p[DIV_ROUND_UP(2 * ARM_MAX_VQ, 8)] QEMU_ALIGNED(16);
} ARMPredicateReg;

// cpu.h:677-679
#define FFR_PRED_NUM 16
ARMPredicateReg pregs[17];  // P0-P15 + FFR (pregs[16])
```

**谓词宽度**：每个向量元素对应 1 位谓词。对于 VL=2048 位（VQ=16）的字节向量，需要 256 位谓词（32 字节）。

**FFR（First Fault Register）**：用于 first-fault 和 no-fault 加载操作，作为 `pregs[16]` 存储。

### 2.3 向量长度参数

| 参数 | 含义 | 范围 |
|------|------|------|
| VL | 向量长度（位） | 128-2048，步长 128 |
| VQ | 向量四字数 = VL/128 | 1-16 |
| VQ-1 | QEMU 内部使用 | 0-15（存储在 ZCR.LEN） |

## 3. SVE 向量长度管理

### 3.1 ZCR 寄存器层级

```c
// helper.c:4759-4777 — ZCR_EL1/EL2/EL3 定义
{ .name = "ZCR_EL1", .fieldoffset = offsetof(CPUARMState, vfp.zcr_el[1]) }
{ .name = "ZCR_EL2", .fieldoffset = offsetof(CPUARMState, vfp.zcr_el[2]) }
{ .name = "ZCR_EL3", .fieldoffset = offsetof(CPUARMState, vfp.zcr_el[3]) }
```

**有效 VL 的确定**（helper.c:4705-4731）：

```
effective_VQ = min(ZCR_EL1.LEN, ZCR_EL2.LEN, ZCR_EL3.LEN, cpu_max_vq) + 1

具体规则：
- EL1/EL0: min(ZCR_EL1.LEN, ZCR_EL2.LEN*, ZCR_EL3.LEN*)
- EL2:     min(ZCR_EL2.LEN, ZCR_EL3.LEN*)
- EL3:     min(ZCR_EL3.LEN, cpu_max)
  * 仅当对应 EL 存在时参与 min
```

高特权 EL 可以通过设置较小的 ZCR.LEN 来限制低 EL 的向量长度，但不能增加超过自身限制。

### 3.2 VL 变化时的状态管理

```c
// helper.c:4738-4756 — zcr_write()
static void zcr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    raw_write(env, ri, value & 0xf);          // ZCR.LEN = [3:0]
    new_len = sve_vqm1_for_el(env, cur_el);
    if (new_len < old_len) {
        aarch64_sve_narrow_vq(env, new_len + 1);  // 截断高位
    }
}

// helper.c:10029-10053 — aarch64_sve_narrow_vq()
void aarch64_sve_narrow_vq(CPUARMState *env, unsigned vq)
{
    // 清零 Z 寄存器超出新 VQ 的高位
    for (i = 0; i < 32; i++)
        memset(&env->vfp.zregs[i].d[2 * vq], 0, 16 * (ARM_MAX_VQ - vq));

    // 清零谓词寄存器超出新 VQ 的高位（包括 FFR）
    for (j = vq / 4; j < ARM_MAX_VQ / 4; j++)
        for (i = 0; i < 17; ++i)
            env->vfp.pregs[i].p[j] &= pmask;
}
```

**关键规则**：VL 只能缩小时清零不可访问的高位，VL 增大时高位内容未定义（架构要求软件不依赖高位内容）。

### 3.3 EL 切换时的 VL 处理

```c
// helper.c:10073-10128 — aarch64_sve_change_el()
void aarch64_sve_change_el(CPUARMState *env, int old_el, int new_el, bool el0_a64)
{
    // 1. AArch64↔AArch32 切换 + Streaming 模式 → 完全重置
    if (old_a64 != new_a64 && sm) {
        arm_reset_sve_state(env);  // 清零所有 Z/P/FFR
        return;
    }

    // 2. 计算旧/新 EL 的有效 VL
    old_len = old_a64 ? sve_vqm1_for_el_sm_ena(env, old_el, sm) : 0;
    new_len = new_a64 ? sve_vqm1_for_el_sm_ena(env, new_el, sm) : 0;

    // 3. 新 VL < 旧 VL → 截断高位
    if (new_len < old_len)
        aarch64_sve_narrow_vq(env, new_len + 1);
}
```

**调用时机**：
- 异常进入：`arm_cpu_do_interrupt_aarch64()` 中调用（helper.c:9214）
- 异常返回：`HELPER(exception_return)` 中调用（helper-a64.c:758）

## 4. SVE 陷入与使能控制

### 4.1 SVE 陷入判定

```c
// helper.c:4598-4641 — sve_exception_el()
int sve_exception_el(CPUARMState *env, int el)
{
    // 第一层：CPACR_EL1.ZEN 控制 EL0/EL1
    if (el <= 1 && !el_is_in_host(env, el)) {
        switch (CPACR_EL1.ZEN) {
        case 0b00: return 1;  // 全部陷入 EL1
        case 0b01: if (el==0) return 1; break; // 仅 EL0 陷入
        case 0b10: return 1;  // 全部陷入 EL1
        case 0b11: break;     // 不陷入
        }
    }

    // 第二层：CPTR_EL2 控制 EL0/EL1/EL2
    if (el <= 2 && arm_is_el2_enabled(env)) {
        if (HCR_EL2.E2H) {
            // VHE 模式：CPTR_EL2.ZEN 字段
            switch (CPTR_EL2.ZEN) { /* 类似 CPACR.ZEN */ }
        } else {
            // 非 VHE：CPTR_EL2.TZ 位
            if (CPTR_EL2.TZ) return 2;
        }
    }

    // 第三层：CPTR_EL3.EZ 控制所有 EL（注意是反向逻辑）
    if (arm_feature(env, ARM_FEATURE_EL3) && !CPTR_EL3.EZ)
        return 3;

    return 0;  // 不陷入
}
```

**陷入控制矩阵：**
| 配置寄存器 | 位域 | 控制范围 | 陷入目标 |
|-----------|------|---------|---------|
| CPACR_EL1 | ZEN[1:0] | EL0, EL1 | → EL1 |
| CPTR_EL2 | TZ / ZEN | EL0, EL1, EL2 | → EL2 |
| CPTR_EL3 | EZ（反向） | EL0-EL2 | → EL3 |

### 4.2 SVE TB Flags

```c
// hflags.c:264-281 — SVE 相关 TB flags
int sve_el = sve_exception_el(env, el);
if (fp_el != 0) {
    if (sve_el > fp_el) sve_el = 0;  // FP 异常优先
} else if (sve_el == 0) {
    DP_TBFLAG_A64(flags, VL, sve_vqm1_for_el(env, el));  // 编码 VL
}
DP_TBFLAG_A64(flags, SVEEXC_EL, sve_el);  // SVE 陷入目标 EL
```

| TB Flag | 含义 |
|---------|------|
| `SVEEXC_EL` | SVE 陷入到哪个 EL（0=不陷入） |
| `VL` | 当前有效 VQ-1（0-15） |
| `PSTATE_SM` | 是否处于 Streaming SVE 模式 |
| `SME_TRAP_NONSTREAMING` | 非 Streaming 指令是否需要陷入 |
| `SMEEXC_EL` | SME 陷入目标 EL |
| `SVL` | Streaming 模式的 VQ-1 |

## 5. SVE 指令翻译

### 5.1 SVE 解码文件结构

`sve.decode` 定义了 SVE 指令的解码模式，主要分类：

| 指令组 | 代表指令 | 说明 |
|--------|---------|------|
| 整数算术/逻辑 | ADD, SUB, AND, ORR, EOR | 谓词化向量运算 |
| 索引生成 | INDEX | 生成序列向量 |
| 排列组合 | TBL, UZP, ZIP, TRN | 向量元素重排 |
| 谓词操作 | PTRUE, PFALSE, AND_pppp | 谓词生成与逻辑 |
| 谓词计数 | CNTP, INCP, DECP | 基于谓词的计数 |
| 比较 | CMP, CMPEQ | 向量比较生成谓词 |
| 浮点运算 | FADD, FMUL, FMLA | 浮点向量运算 |
| 连续加载 | LD1B, LD1W, LD1D | 连续内存加载 |
| 连续存储 | ST1B, ST1W, ST1D | 连续内存存储 |
| Gather 加载 | LD1_zpiz | 分散地址加载 |
| Scatter 存储 | ST1_scatter | 分散地址存储 |
| First-fault | LDFF1, LDNF1 | 容错加载 |
| SVE2 扩展 | MATCH, HISTCNT, CDOT | SVE2 新增指令 |

### 5.2 翻译模式

SVE 指令翻译大量使用 QEMU 的 GVec（通用向量）框架：

```c
// translate-sve.c — 典型翻译模式

// 1. 使用 tcg_gen_gvec_* 进行向量运算
tcg_gen_gvec_2_ool(vec_full_reg_offset(s, rd),
                   vec_full_reg_offset(s, rn),
                   vsz, vsz, 0, gen_helper_sve_xxx);

// 2. 带谓词的运算使用 _ool (out-of-line) helper
tcg_gen_gvec_3_ool(d, n, m, vsz, vsz, 0, gen_helper_sve_add_zpzz_b);

// 3. 向量长度通过 TB flag VL 获取
int vsz = vec_full_reg_size(s);  // 从 hflags 中的 VL 计算
```

### 5.3 谓词操作翻译

```c
// translate-sve.c:1652-1753 — PTRUE
// PTRUE 生成全 1 谓词（受元素大小约束）
trans_PTRUE → do_predset()
    → gen_helper_sve_ptrue(tcg_env, preg, vsz, esz, pattern)

// translate-sve.c:3406-3580 — WHILE*
// WHILE 基于标量比较生成谓词
trans_WHILE_lt → do_WHILE()
    → gen_helper_sve_whilelt(tcg_env, ...)

// translate-sve.c:3137-3213 — BRKA/BRKB
// 谓词中断操作（find first true/false）
trans_BRKA → do_brk2()
    → gen_helper_sve_brka_m/z(...)
```

### 5.4 内存操作翻译

```c
// translate-sve.c:4585-5375 — 加载/存储

// 连续加载：LD1B, LD1H, LD1W, LD1D
trans_LD1_zprr → do_ld_zpa()
    → gen_helper_sve_ld1bb_r / ld1hh_le_r / ...

// First-fault 加载
trans_LDFF1_zprr → 使用 fault-tolerant helper
    → gen_helper_sve_ldff1bb_r / ...

// No-fault 加载
trans_LDNF1_zpri → 不产生页错误的加载
    → gen_helper_sve_ldnf1bb_r / ...

// Gather 加载（分散地址）
trans_LD1_zpiz → gen_helper_sve_ld1_zpiz(...)
```

### 5.5 SVE Helper 函数

```c
// sve_helper.c — 运行时 helper 模式

// 宏驱动的谓词化运算
#define DO_ZPZZ(NAME, TYPE, H, OP)                          \
void HELPER(NAME)(void *vd, void *vn, void *vm, void *vg,   \
                  uint32_t desc) {                           \
    intptr_t i = simd_oprsz(desc);                          \
    do { i -= sizeof(TYPE);                                 \
         if (*(uint8_t *)(vg + H1(i >> 3)) & (1 << ...))   \
             *(TYPE *)(vd + H(i)) = OP;                     \
    } while (i > 0);                                        \
}

// 谓词测试（设置 NZCV）
// sve_helper.c:58-117
HELPER(sve_predtest) / HELPER(sve_predtest1)
    → 扫描谓词设置 N(first)/Z(none)/C(last)/V(0) 标志

// 谓词迭代
HELPER(sve_pfirst) / HELPER(sve_pnext)
    → 查找谓词中的第一个/下一个 true 元素
```

## 6. SME 架构

### 6.1 SME 状态存储

```c
// cpu.h:315
uint64_t svcr;  // PSTATE.{SM,ZA} — SVCR 格式

// cpu.h:1583-1585
FIELD(SVCR, SM, 0, 1)  // bit 0: Streaming SVE 模式
FIELD(SVCR, ZA, 1, 1)  // bit 1: ZA 数组使能

// cpu.h:725-754 — ZA 存储
struct {
    uint64_t zt0[512 / 64] QEMU_ALIGNED(16);   // SME2 ZT0（512 位）
    ARMVectorReg za[ARM_MAX_VQ * 16];           // ZA: 256×256 字节矩阵
} za_state;

// cpu.h:700-701 — 向量长度控制
uint64_t zcr_el[4];   // ZCR_EL[1-3] — SVE VL 控制
uint64_t smcr_el[4];  // SMCR_EL[1-3] — Streaming SVE VL 控制
```

### 6.2 ZA 矩阵存储布局

```
ZA 是一个 SVL×SVL 字节的二维矩阵（SVL = Streaming Vector Length）

存储方式: za[N] 是 ARMVectorReg（最大 256 字节），代表 ZA 的第 N 行
最大: 256 行 × 256 字节 = 64KB

Tile 映射（以元素大小 esz 字节为单位）:
  Tile T 的第 N 行 = ZA[T + N * esz]

例如 32 位元素 (esz=4):
  ZA0S 行 0 = ZA[0]     ZA1S 行 0 = ZA[1]     ZA2S 行 0 = ZA[2]     ZA3S 行 0 = ZA[3]
  ZA0S 行 1 = ZA[4]     ZA1S 行 1 = ZA[5]     ZA2S 行 1 = ZA[6]     ZA3S 行 1 = ZA[7]
  ...
```

### 6.3 Streaming SVE 模式（PSTATE.SM）

```c
// helper.c:4832-4860 — aarch64_set_svcr()
void aarch64_set_svcr(CPUARMState *env, uint64_t new, uint64_t mask)
{
    uint64_t change = (env->svcr ^ new) & mask;
    if (change == 0) return;
    env->svcr ^= change;

    // SM 变化 → 完全重置 SVE 状态
    if (change & R_SVCR_SM_MASK) {
        arm_reset_sve_state(env);  // 清零所有 Z/P/FFR + 重置 FPSR
    }

    // ZA 新启用 → 清零 ZA 存储
    if (change & new & R_SVCR_ZA_MASK) {
        memset(&env->za_state, 0, sizeof(env->za_state));
    }

    arm_rebuild_hflags(env);  // 重建 TB flags
}
```

**SM 切换的影响：**
| 操作 | SM 0→1 (进入 Streaming) | SM 1→0 (退出 Streaming) |
|------|----------------------|----------------------|
| Z 寄存器 | 全部清零 | 全部清零 |
| P 寄存器 | 全部清零 | 全部清零 |
| FFR | 清零 | 清零 |
| FPSR | 重置为 0x0800009f | 重置为 0x0800009f |
| ZA 状态 | 不受影响 | 不受影响 |
| VL | 切换到 SVL（SMCR） | 切换回 SVE VL（ZCR） |

```c
// helper.c:4823-4830 — arm_reset_sve_state()
static void arm_reset_sve_state(CPUARMState *env)
{
    memset(env->vfp.zregs, 0, sizeof(env->vfp.zregs));
    memset(env->vfp.pregs, 0, sizeof(env->vfp.pregs));  // 含 FFR
    vfp_set_fpsr(env, 0x0800009f);
}
```

### 6.4 SVL（Streaming Vector Length）

```c
// helper.c:4733-4736
uint32_t sve_vqm1_for_el(CPUARMState *env, int el)
{
    return sve_vqm1_for_el_sm(env, el, FIELD_EX64(env->svcr, SVCR, SM));
}
```

- **非 Streaming 模式**：使用 `zcr_el[]` + `cpu->sve_vq.map`
- **Streaming 模式**：使用 `smcr_el[]` + `cpu->sme_vq.map`

SVL 和 SVE VL 可以不同。例如 CPU 可以配置 SVE VL=512 位但 SVL=256 位。

### 6.5 PSTATE.ZA 管理

ZA 的使能/禁用独立于 SM：
- `SMSTART ZA` → 设置 PSTATE.ZA=1，清零 ZA 存储
- `SMSTOP ZA` → 设置 PSTATE.ZA=0（ZA 内容变不可访问）
- `SMSTART SM` → 设置 PSTATE.SM=1，不影响 ZA
- `SMSTART` → 同时设置 SM 和 ZA

## 7. SME 陷入控制

### 7.1 SME 陷入判定

```c
// helper.c:4647-4700 — sme_exception_el()
int sme_exception_el(CPUARMState *env, int el)
{
    // 第一层：CPACR_EL1.SMEN 控制 EL0/EL1
    if (el <= 1 && !el_is_in_host(env, el)) {
        switch (CPACR_EL1.SMEN) {
        case 0b00: return 1;  // 陷入 EL1
        case 0b01: if (el==0) return 1;
        case 0b10: return 1;
        case 0b11: break;
        }
    }

    // 第二层：CPTR_EL2.SMEN/TSM
    if (el <= 2 && arm_is_el2_enabled(env)) {
        // E2H 模式: CPTR_EL2.SMEN
        // 非 E2H: CPTR_EL2.TSM
    }

    // 第三层：CPTR_EL3.ESM（反向逻辑）
    if (arm_feature(env, ARM_FEATURE_EL3) && !CPTR_EL3.ESM)
        return 3;

    return 0;
}
```

### 7.2 ZT0 陷入

```c
// SME2 ZT0 有独立的陷入控制
int zt0_exception_el(CPUARMState *env, int el)
{
    // 检查 SMCR_EL1/EL2/EL3.EZT0 位
    // EZT0=0 → 陷入到相应 EL
}
```

### 7.3 FA64（Full A64 in Streaming Mode）

```c
// hflags.c:141-165
static bool sme_fa64(CPUARMState *env, int el)
{
    // 需要 CPU 支持 FEAT_SME_FA64 且所有适用 EL 的 SMCR.FA64=1
    if (!cpu_isar_feature(aa64_sme_fa64, cpu)) return false;
    if (el <= 1 && !el_is_in_host(env, el))
        if (!FIELD_EX64(smcr_el[1], SMCR, FA64)) return false;
    if (el <= 2 && arm_is_el2_enabled(env))
        if (!FIELD_EX64(smcr_el[2], SMCR, FA64)) return false;
    if (arm_feature(env, ARM_FEATURE_EL3))
        if (!FIELD_EX64(smcr_el[3], SMCR, FA64)) return false;
    return true;
}
```

**FA64 的作用**：
- **FA64=0**（默认）：Streaming 模式下只能执行 Streaming 兼容指令子集
- **FA64=1**：Streaming 模式下可以执行所有 A64 指令（包括非 Streaming 指令）

在 TB flags 中：
```c
// hflags.c:296-298
if (sm) {
    DP_TBFLAG_A64(flags, SME_TRAP_NONSTREAMING, !sme_fa64(env, el));
}
```

## 8. SME 指令翻译

### 8.1 SME 指令分类

| 指令 | 功能 | 翻译位置 |
|------|------|---------|
| `SMSTART` | 进入 Streaming/启用 ZA | 写 SVCR 寄存器 |
| `SMSTOP` | 退出 Streaming/禁用 ZA | 写 SVCR 寄存器 |
| `ZERO` | 清零 ZA tiles | translate-sme.c:155-178 |
| `ZERO {ZT0}` | 清零 ZT0（SME2） | translate-sme.c:167-177 |
| `MOVA` | ZA 行 ↔ Z 寄存器移动 | translate-sme.c:180-248 |
| `LD1/ST1` | ZA 行加载/存储 | translate-sme.c / sme.decode |
| `LDR/STR` | ZA/ZT0 批量保存恢复 | translate-sme.c:300+ |
| `FMOPA/BFMOPA` | 外积累加到 ZA tile | translate-sme.c |
| `ADDHA/ADDVA` | 水平/垂直累加 | translate-sme.c |

### 8.2 SME 检查函数

```c
// translate-sme.c:34-46
static bool sme2_zt0_enabled_check(DisasContext *s)
{
    // 检查 SME2 + ZT0 + ZA 使能
    // 未使能 → 产生异常
}

// sme_za_enabled_check(s) — 检查 PSTATE.ZA=1
// sme_sm_enabled_check(s) — 检查 PSTATE.SM=1
```

## 9. SVE/SME CPU 属性配置

### 9.1 SVE 属性

```c
// cpu64.c:63-261 — arm_cpu_sve_finalize()
// cpu64.c:487-520 — aarch64_add_sve_properties()

// 用户可配置的属性:
// -cpu max,sve=on              # 启用 SVE
// -cpu max,sve128=on           # 启用 128 位向量
// -cpu max,sve256=on           # 启用 256 位向量
// -cpu max,sve512=on           # 启用 512 位向量
// ...
// -cpu max,sve2048=on          # 启用 2048 位向量
// -cpu max,sve-max-vq=4        # 最大 VQ=4 (512 位)
// -cpu max,sve-default-vector-length=256  # 默认 VL（linux-user 模式）
```

**初始化规则**（`arm_cpu_sve_finalize()`）：
1. 如果显式启用了某个 `sve<N>`，所有其他长度隐式禁用
2. 所有小于最大启用长度的 2 的幂次长度自动启用
3. 所有大于最大禁用的 2 的幂次长度自动禁用
4. SVE 禁用时，SVE 专用的 ID 寄存器字段被清除

### 9.2 SME 属性

```c
// cpu64.c:878-914 — aarch64_add_sme_properties()

// 用户可配置的属性:
// -cpu max,sme=on              # 启用 SME
// -cpu max,sme_fa64=on         # 启用 FA64
// -cpu max,sme128=on           # 启用 128 位 SVL
// -cpu max,sme256=on           # 启用 256 位 SVL
// ...
// -cpu max,sme2048=on          # 启用 2048 位 SVL
```

**SME 与 SVE 的独立性**：
- SME 可以在没有 SVE 的情况下启用（此时 Streaming 模式提供 SVE 指令子集）
- SVE 可以在没有 SME 的情况下启用
- 两者都启用时，SVE VL 和 SVL 可以配置为不同值

## 10. SVE2 扩展

### 10.1 SVE2 特性检测

```c
// cpu-features.h:1442-1479
isar_feature_aa64_sve2()        // ID_AA64ZFR0.SVEVER != 0
isar_feature_aa64_sve2p1()      // SVEVER >= 2
isar_feature_aa64_sve2_aes()    // AES != 0
isar_feature_aa64_sve2_pmull128() // AES >= 2
isar_feature_aa64_sve2_bitperm() // BITPERM != 0
isar_feature_aa64_sve2_sha3()   // SHA3 != 0
isar_feature_aa64_sve2_sm4()    // SM4 != 0
```

### 10.2 SVE2 新增指令类别

| 扩展 | 指令示例 | 功能 |
|------|---------|------|
| 基础 SVE2 | MATCH, NMATCH | 字符搜索 |
| | HISTCNT, HISTSEG | 直方图计算 |
| | CDOT | 复数点积 |
| | SQRDMLAH/SH | 饱和舍入乘加 |
| | TBX | 扩展表查找 |
| SVE2-AES | AESE, AESD, AESMC | AES 加密 |
| SVE2-PMULL | PMULLB, PMULLT | 128 位多项式乘法 |
| SVE2-BitPerm | BEXT, BDEP, BGRP | 位排列 |
| SVE2-SHA3 | RAX1 | SHA3 旋转异或 |
| SVE2-SM4 | SM4E, SM4EKEY | SM4 加密 |

## 11. SME2 扩展

### 11.1 SME2 特性

```c
// cpu-features.h
isar_feature_aa64_sme2()   // ID_AA64SMFR0.SMEVER != 0
isar_feature_aa64_sme2p1() // SMEVER >= 2
```

### 11.2 ZT0 寄存器

```c
// cpu.h:726-727
uint64_t zt0[512 / 64] QEMU_ALIGNED(16);  // 512 位查找表寄存器
```

ZT0 是 SME2 引入的 512 位寄存器，用于 TABLE 查找操作。它有独立的陷入控制（SMCR.EZT0）和专用的清零指令（`ZERO {ZT0}`）。

## 12. Streaming 模式与非 Streaming 模式对比

| 特性 | 非 Streaming (SM=0) | Streaming (SM=1) |
|------|-------------------|-----------------|
| VL 来源 | ZCR_EL1/2/3 | SMCR_EL1/2/3 |
| VL 范围 | cpu->sve_vq.map | cpu->sme_vq.map |
| Z 寄存器宽度 | SVE VL | SVL |
| 可用指令 | 全部 A64 | Streaming 子集 (除非 FA64) |
| NEON 指令 | ✓ 可用 | ✗ 陷入（除非 FA64） |
| SVE 指令 | ✓ 全部 | ○ Streaming 兼容子集 |
| ZA 访问 | 需要 PSTATE.ZA=1 | 需要 PSTATE.ZA=1 |
| FP/SIMD | 正常 | 受限（除非 FA64） |

**Streaming 兼容指令子集**：大部分 SVE 算术/逻辑/谓词指令兼容，但以下不兼容：
- Gather/Scatter 加载存储
- First-fault / No-fault 加载
- 某些排列组合指令
- 非 Streaming SVE2 指令

## 13. EL 切换与 SVE/SME 状态的完整交互

```
异常进入 (EL_low → EL_high):
  1. aarch64_sve_change_el(env, cur_el, new_el)
     - 如果 AArch64↔AArch32 + SM=1 → arm_reset_sve_state()
     - 如果 new_VL < old_VL → aarch64_sve_narrow_vq()
  2. 新 EL 的 ZCR/SMCR 决定新 VL
  3. TB flags 中 VL/SVL 更新

异常返回 (EL_high → EL_low):
  1. aarch64_sve_change_el(env, cur_el, new_el)
     - 同上逻辑
  2. 恢复到旧 EL 的 ZCR/SMCR VL

SMSTART/SMSTOP:
  1. aarch64_set_svcr() 修改 PSTATE.SM/ZA
  2. SM 变化 → arm_reset_sve_state()（清零 Z/P/FFR）
  3. ZA 启用 → memset(&za_state, 0, sizeof(za_state))
  4. arm_rebuild_hflags()（VL 可能变化）
```

## 14. 与其他文档的关联

| 相关文档 | 关联点 |
|---------|--------|
| [01-CPU模型](01-CPU模型与初始化深度分析.md) | SVE/SME CPU 属性注册 |
| [04-执行循环](../architecture/02-模拟执行循环与MMIO分发深度分析.md) | TB flags 与翻译块缓存 |
| [09-特殊指令](09-ARM64特殊指令处理深度分析.md) | 指令翻译框架 |
| [14-EL状态管理](14-EL状态管理与指令执行变化深度分析.md) | EL 切换时 SVE 状态处理 |
| [13-PMU](13-PMU性能监控单元深度分析.md) | EL 变化钩子链 |

## 15. 总结

QEMU 的 ARM64 SVE/SME 实现核心机制：

1. **向量长度不可知**：Z 寄存器最大 2048 位，通过 ZCR/SMCR 动态控制有效 VL，翻译块按 VL 隔离缓存
2. **谓词驱动**：所有 SVE 运算通过 P 寄存器控制活跃通道，helper 函数使用宏驱动的逐元素迭代
3. **双模式架构**：SVE 正常模式（ZCR 控制 VL）和 Streaming 模式（SMCR 控制 SVL）可以配置不同的向量长度
4. **状态隔离**：SM 切换清零所有 Z/P/FFR；ZA 启用清零矩阵；EL 切换时 VL 缩小导致高位截断
5. **三层陷入**：CPACR.ZEN/SMEN → CPTR_EL2.TZ/TSM → CPTR_EL3.EZ/ESM 逐层控制
6. **ZA 矩阵存储**：最大 256×256 字节，按行存储，Tile 行交错映射
7. **GVec 翻译框架**：SVE 指令通过 `tcg_gen_gvec_*` 和 out-of-line helper 实现，支持任意 VL
