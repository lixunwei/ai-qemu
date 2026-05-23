# ARM64 浮点/NEON/SVE 指令翻译深度分析

> 文档编号：85  
> 分析目标：AArch64 浮点、NEON（AdvSIMD）和 SVE 指令的 TCG 翻译机制  
> 源码版本：QEMU 11.0.50  
> 核心文件：translate-a64.c (10961行)、translate-sve.c (8227行)、sve_helper.c (8634行)、vfp_helper.c (1375行)

---

## 一、概述

ARM64 的 SIMD/浮点指令集包含三大部分：

| 指令集 | 寄存器 | 向量宽度 | 翻译文件 |
|--------|--------|----------|----------|
| VFP/标量浮点 | D0-D31 (64位) / S0-S31 (32位) / H0-H31 (16位) | 标量 | translate-a64.c |
| NEON (AdvSIMD) | V0-V31 (128位) | 64/128位固定 | translate-a64.c |
| SVE/SVE2 | Z0-Z31 (可变长) + P0-P15 (谓词) | 128~2048位可变 | translate-sve.c |
| SME | ZA (矩阵) + ZT0 | VL×VL 矩阵 | translate-sme.c |

代码规模统计：
```
translate-a64.c     10961行  (含标量FP + NEON + 部分SME)
translate-sve.c      8227行  (SVE/SVE2 完整指令集)
sve_helper.c         8634行  (SVE 运行时 helper)
vfp_helper.c         1375行  (VFP/浮点 helper)
neon_helper.c        1452行  (NEON helper, 主要 AArch32)
a64.decode           1927行  (AArch64 解码描述)
sve.decode           1936行  (SVE 解码描述)
```

---

## 二、寄存器存储模型

### 2.1 向量寄存器

```c
// target/arm/cpu.h:168-172
#define ARM_MAX_VQ    16    // 最大向量长度 = 16 × 128bit = 2048bit

typedef struct ARMVectorReg {
    uint64_t d[2 * ARM_MAX_VQ] QEMU_ALIGNED(16);  // 32 × uint64_t = 2048bit
} ARMVectorReg;

// CPUARMState.vfp 中的寄存器:
struct {
    ARMVectorReg zregs[32];    // Z0-Z31（V0-V31 是低 128 位视图）
    ARMPredicateReg pregs[17]; // P0-P15 + FFR（P16）
    uint64_t fpsr;             // 浮点状态寄存器
    uint64_t fpcr;             // 浮点控制寄存器
    float_status fp_status[FPST_COUNT]; // softfloat 状态
    uint64_t zcr_el[4];        // ZCR_EL1/2/3 — 控制 VL
    uint64_t smcr_el[4];       // SMCR_EL1/2/3 — 控制 SVL
} vfp;
```

### 2.2 向量寄存器视图关系

```
Z0 (SVE 向量寄存器, VL bits):
┌──────────────────────────────────────────────────────┐
│ d[31] d[30] ... d[3] d[2] │ d[1] d[0]              │
│                            │ ← V0 (NEON, 128bit) → │
│                            │ ← D0 (64bit)│          │
│                            │ ← S0│                   │
└──────────────────────────────────────────────────────┘
```

### 2.3 翻译时的寄存器访问

```c
// translate-a64.h:110-127
static inline int vec_full_reg_offset(DisasContext *s, int regno) {
    return offsetof(CPUARMState, vfp.zregs[regno]);
}

static inline int vec_full_reg_size(DisasContext *s) {
    return s->vl;  // 当前向量长度（字节）
}

static inline int pred_full_reg_offset(DisasContext *s, int regno) {
    return offsetof(CPUARMState, vfp.pregs[regno]);
}
```

---

## 三、解码架构（decodetree）

### 3.1 解码文件层次

```
a64.decode (1927行)
├── Advanced SIMD scalar copy
├── Advanced SIMD copy (DUP/INS)
├── Advanced SIMD scalar three same
├── Advanced SIMD three same (ADD/SUB/MUL/...)
├── Advanced SIMD scalar x indexed element
├── Advanced SIMD vector x indexed element
├── Advanced SIMD Extract
├── Advanced SIMD Table Lookup (TBL/TBX)
├── Advanced SIMD Permute (UZP/ZIP/TRN)
├── Advanced SIMD Across Lanes (ADDV/SADDLV)
├── Advanced SIMD Modified Immediate / Shift by Immediate
├── Advanced SIMD scalar shift by immediate
├── Advanced SIMD scalar two-register miscellaneous
├── Advanced SIMD two-register miscellaneous
├── Floating-point conditional select
├── Floating-point data-processing (3 source)
├── Floating-point data-processing (1 source)
├── Floating-point Immediate
├── Floating-point Compare
├── Conversion between FP and fixed-point
└── Conversion between FP and integer

sve.decode (1936行)
├── SVE Integer Arithmetic
├── SVE Logical Operations  
├── SVE Predicate Operations
├── SVE Memory Operations (LD1/ST1/gather/scatter)
├── SVE Floating Point Arithmetic
└── SVE2 Extensions
```

### 3.2 解码模式示例

```
# a64.decode — FADD (scalar)
FADD_s          0001 1110 .. 1 ..... 001 010 ..... .....    @rrr_hsd

# sve.decode — FADD (predicated vector)
FADD_zpzz       01100101 .. 00 0000 100 ... ..... .....     @rdn_pg_rm

# a64.decode — DUP (element to vector)
DUP_element_v   0 . 00 1110 000 imm:5 0 0000 1 rn:5 rd:5   q=%q_DUP
```

---

## 四、浮点指令翻译模式

### 4.1 fp_access_check — 浮点访问检查

每条 FP/SIMD 指令翻译前必须调用：

```c
static bool fp_access_check(DisasContext *s) {
    return fp_access_check_only(s) && nonstreaming_check(s);
}

static bool fp_access_check_only(DisasContext *s) {
    if (s->fp_excp_el) {
        // FP/SIMD 被 trap 到更高 EL（CPACR_EL1.FPEN/CPTR_EL2 等）
        gen_exception_insn_el(s, 0, EXCP_UDEF,
                              syn_fp_access_trap(1, 0xe, false, 0),
                              s->fp_excp_el);
        return false;  // 指令不翻译
    }
    s->fp_access_checked = 1;
    return true;
}
```

`fp_excp_el` 在翻译块开头由 `aarch64_tr_init_disas_context()` 根据 CPACR_EL1/CPTR_EL2/CPTR_EL3 计算。

### 4.2 标量浮点翻译模式

```c
// 典型 3 操作数标量浮点指令（如 FADD）
static bool do_fp3_scalar(DisasContext *s, arg_rrr_e *a, const FPScalar *f,
                          int mergereg) {
    switch (a->esz) {
    case MO_64:  // double
        if (fp_access_check(s)) {
            TCGv_i64 t0 = read_fp_dreg(s, a->rn);  // 读源操作数
            TCGv_i64 t1 = read_fp_dreg(s, a->rm);
            f->gen_d(t0, t0, t1, fpstatus_ptr(FPST_A64));  // helper 调用
            write_fp_dreg_merging(s, a->rd, mergereg, t0);  // 写结果
        }
        break;
    case MO_32:  // float
        if (fp_access_check(s)) {
            TCGv_i32 t0 = read_fp_sreg(s, a->rn);
            TCGv_i32 t1 = read_fp_sreg(s, a->rm);
            f->gen_s(t0, t0, t1, fpstatus_ptr(FPST_A64));
            write_fp_sreg_merging(s, a->rd, mergereg, t0);
        }
        break;
    case MO_16:  // half
        if (!dc_isar_feature(aa64_fp16, s)) return false;
        // 类似处理...
    }
    return true;
}
```

**关键设计**：
- `FPScalar` 结构体包含 `gen_h/gen_s/gen_d` 三个 helper 指针
- `fpstatus_ptr()` 返回指向 softfloat `float_status` 的指针
- `write_fp_dreg_merging()` 处理 FPCR.NEP（保留还是清零高位）

### 4.3 FPCR.AH 处理

FEAT_AFP（Alternative floating-point，Armv9.2）引入 FPCR.AH 位，改变 NaN 处理行为：

```c
// FPCR.AH == 1 时 NEG 不翻转 NaN 的符号位
static void gen_vfp_ah_negs(TCGv_i32 d, TCGv_i32 s) {
    TCGv_i32 abs_s = tcg_temp_new_i32();
    gen_vfp_negs(chs_s, s);
    gen_vfp_abss(abs_s, s);
    // if (|s| > 0x7f800000) d = s; else d = -s;
    tcg_gen_movcond_i32(TCG_COND_GTU, d,
                        abs_s, tcg_constant_i32(0x7f800000),
                        s, chs_s);
}
```

---

## 五、NEON (AdvSIMD) 向量翻译

### 5.1 GVec 框架

QEMU 使用 `tcg_gen_gvec_*` 系列函数进行向量操作，自动根据主机 SIMD 能力选择最优实现：

```c
// 2操作数向量操作
static void gen_gvec_fn2(DisasContext *s, bool is_q, int rd, int rn,
                         GVecGen2Fn *gvec_fn, int vece) {
    gvec_fn(vece,
            vec_full_reg_offset(s, rd),   // 目标偏移
            vec_full_reg_offset(s, rn),   // 源偏移
            is_q ? 16 : 8,               // 操作长度（Q=128bit, !Q=64bit）
            vec_full_reg_size(s));        // 完整寄存器大小（清高位用）
}

// 3操作数 + fpstatus（浮点向量运算）
static void gen_gvec_op3_fpst(DisasContext *s, bool is_q, int rd, int rn,
                              int rm, ARMFPStatusFlavour fpsttype, int data,
                              gen_helper_gvec_3_ptr *fn) {
    TCGv_ptr fpst = fpstatus_ptr(fpsttype);
    tcg_gen_gvec_3_ptr(vec_full_reg_offset(s, rd),
                       vec_full_reg_offset(s, rn),
                       vec_full_reg_offset(s, rm), fpst,
                       is_q ? 16 : 8, vec_full_reg_size(s), data, fn);
}
```

### 5.2 Q 位与高位清零

NEON 指令中 Q=0 操作 64 位（V寄存器低半），Q=1 操作 128 位。**无论哪种情况，SVE 寄存器 128 位以上都被清零**：

```c
// 操作长度 = is_q ? 16 : 8
// 最大长度 = vec_full_reg_size(s)  // = VL bytes
// GVec 框架自动用 0 填充 oprsz 到 maxsz 之间的字节
```

### 5.3 整数向量指令（无需 softfloat）

```c
// ADD (vector) — 直接内联到 TCG IR
TRANS(ADD_v, do_gvec_fn3, a, tcg_gen_gvec_add)

// AND/ORR/EOR — 逻辑操作内联
TRANS(AND_v, do_gvec_fn3, a, tcg_gen_gvec_and)

// 这些被 GVec 框架展开为主机 SIMD（AVX/NEON）或标量循环
```

### 5.4 浮点向量指令（需要 helper）

```c
// FADD (vector) — 必须调用 helper（softfloat 有副作用）
static gen_helper_gvec_3_ptr * const f_scalar_fadd[3] = {
    gen_helper_gvec_fadd_h,  // FP16
    gen_helper_gvec_fadd_s,  // FP32
    gen_helper_gvec_fadd_d,  // FP64
};
// helper 内部循环调用 float32_add() / float64_add()
```

---

## 六、SVE 翻译

### 6.1 SVE 访问检查

```c
bool sve_access_check(DisasContext *s) {
    if (dc_isar_feature(aa64_sme, s)) {
        // SME Streaming 模式：检查 SVCR.SM 与指令兼容性
        // 某些 SVE 指令在 Streaming 模式下不允许
    }
    if (s->sve_excp_el) {
        gen_exception_insn_el(s, 0, EXCP_UDEF,
                              syn_sve_access_trap(), s->sve_excp_el);
        return false;
    }
    s->sve_access_checked = true;
    return fp_access_check_only(s);
}
```

### 6.2 谓词化操作

SVE 的核心特征——每个元素操作受谓词寄存器控制：

```c
// SVE FADDA — 带谓词的跨通道浮点累加
static bool trans_FADDA(DisasContext *s, arg_rprr_esz *a) {
    unsigned vsz = vec_full_reg_size(s);  // VL 字节
    
    // 读取源、谓词寄存器指针
    TCGv_i64 t_val = load_esz(tcg_env, vec_reg_offset(s, a->rn, 0, a->esz), a->esz);
    tcg_gen_addi_ptr(t_rm, tcg_env, vec_full_reg_offset(s, a->rm));
    tcg_gen_addi_ptr(t_pg, tcg_env, pred_full_reg_offset(s, a->pg));
    t_fpst = fpstatus_ptr(a->esz == MO_16 ? FPST_A64_F16 : FPST_A64);
    t_desc = tcg_constant_i32(simd_desc(vsz, vsz, 0));
    
    // 调用 helper — 内部遍历 VL 中所有元素，跳过谓词为 0 的
    gen_helper_sve_fadda_s(t_val, t_val, t_rm, t_pg, t_fpst, t_desc);
    
    write_fp_dreg(s, a->rd, t_val);
    return true;
}
```

### 6.3 SVE 浮点操作宏

```c
#define DO_FP3(NAME, name) \
    static gen_helper_gvec_3_ptr * const name##_fns[4] = { \
        gen_helper_gvec_##name##_b16, gen_helper_gvec_##name##_h, \
        gen_helper_gvec_##name##_s, gen_helper_gvec_##name##_d  \
    }; \
    TRANS_FEAT(NAME, aa64_sme_or_sve, gen_gvec_fpst_arg_zzz, \
               name##_fns[a->esz], a, 0)

DO_FP3(FADD_zzz, fadd)   // SVE FADD (unpredicated)
DO_FP3(FSUB_zzz, fsub)
DO_FP3(FMUL_zzz, fmul)
```

### 6.4 SVE 内存访问（gather/scatter）

```c
// LD1 (scalar+vector) — gather load
static bool trans_LD1_zprz(DisasContext *s, arg_LD1_zprz *a) {
    // 每个元素独立地址 = base + Z[i]*scale
    gen_helper_gvec_mem_scatter(tcg_env, t_zt, t_pg, t_zn, t_base, t_desc);
}
```

---

## 七、Softfloat 集成

### 7.1 浮点状态

```c
// FPST_COUNT 个独立的 float_status 实例:
enum ARMFPStatusFlavour {
    FPST_A32,       // AArch32 VFP
    FPST_A64,       // AArch64 FP32/FP64
    FPST_A64_F16,   // AArch64 FP16
    FPST_AH,        // FPCR.AH=1 时的替代状态
    FPST_AH_F16,    // FPCR.AH=1 + FP16
    FPST_COUNT
};
```

### 7.2 FPCR 到 float_status 映射

```
FPCR.RMode[23:22] → float_status.rounding_mode
FPCR.FZ[24]       → float_status.flush_to_zero
FPCR.FZ16[19]     → float_status_f16.flush_to_zero
FPCR.DN[25]       → float_status.default_nan_mode
FPCR.AH[1]       → 选择 FPST_AH 替代状态
```

### 7.3 典型 helper 实现

```c
// fpu/softfloat.c 提供的原子操作：
float64 float64_add(float64 a, float64 b, float_status *s);
float64 float64_mul(float64 a, float64 b, float_status *s);
float64 float64_div(float64 a, float64 b, float_status *s);
float64 float64_sqrt(float64 a, float_status *s);

// sve_helper.c 中的向量 helper：
void HELPER(gvec_fadd_s)(void *vd, void *vn, void *vm, void *vfpst, uint32_t desc) {
    intptr_t i, opr_sz = simd_oprsz(desc);
    float_status *fpst = vfpst;
    
    for (i = 0; i < opr_sz; i += sizeof(float32)) {
        float32 n = *(float32 *)(vn + i);
        float32 m = *(float32 *)(vm + i);
        *(float32 *)(vd + i) = float32_add(n, m, fpst);
    }
    clear_tail(vd, opr_sz, simd_maxsz(desc));  // 清高位
}
```

---

## 八、寄存器写入语义

### 8.1 NEON 写入

```
write_fp_dreg(s, reg, val):
  V[reg][63:0] = val
  V[reg][127:64] = 0      // 清高 64 位
  Z[reg][VL-1:128] = 0    // 清 SVE 高位

write_vec(s, reg, Q=0):
  V[reg][63:0] = result
  V[reg][VL-1:64] = 0

write_vec(s, reg, Q=1):
  V[reg][127:0] = result
  Z[reg][VL-1:128] = 0
```

### 8.2 SVE 写入

```
SVE 操作写满 VL 长度，不清零（因为已经是 VL 宽）
clear_tail(vd, oprsz, maxsz):
  仅当 oprsz < maxsz 时清零（AArch64 实际不会出现）
```

### 8.3 FPCR.NEP 处理

```c
// FEAT_AFP: Non-widening Element Preservation
write_fp_dreg_merging(s, rd, mergereg, val):
  if (!s->fpcr_nep) {
      // 传统行为：清高位
      write_fp_dreg(s, rd, val);
  } else {
      // NEP=1：保留高位元素（从 mergereg 复制）
      V[rd] = V[mergereg];
      V[rd][63:0] = val;
  }
```

---

## 九、VL（向量长度）管理

### 9.1 编译时

```c
// DisasContext 中的 vl/svl
s->vl = sve_vq(env) * 16;   // 字节单位的当前 VL
s->svl = sme_svq(env) * 16; // SME Streaming VL
```

### 9.2 运行时

```c
// ZCR_EL1.LEN[3:0] 控制 VL（编码为 VQ-1）
// 有效 VL = min(ZCR_EL1.LEN, ZCR_EL2.LEN, ZCR_EL3.LEN, 实现最大VQ) × 128 bits

uint32_t sve_vqm1_for_el(CPUARMState *env, int el) {
    // 计算生效的 VQ-1
}
```

### 9.3 VL 对翻译的影响

VL 是翻译块的关键参数——VL 改变时必须终止当前 TB：
- `s->vl` 嵌入到 `simd_desc()` 中传递给 helper
- helper 内部用 `simd_oprsz(desc)` 获取操作长度
- GVec 框架根据 oprsz/maxsz 自动处理填充

---

## 十、与真实硬件对比

| 方面 | QEMU TCG | 真实 ARM 硬件 |
|------|----------|---------------|
| 执行模型 | 逐元素循环（helper）或 GVec 展开 | 硬件 SIMD ALU 并行 |
| 浮点精度 | softfloat 严格 IEEE 754 | 硬件 FPU（可能有微差异） |
| 性能 | FP 向量操作 ~10-50x 慢于原生 | 单周期或流水线 |
| VL 支持 | ARM_MAX_VQ=16 (2048bit) | 实现定义（通常 128-512bit） |
| 异常标志 | 每次操作更新 float_status | 硬件 FPSR 累积 |
| FPCR.AH | 完整实现（选择不同 helper） | 硬件透明处理 |
| NaN 处理 | softfloat 默认 NaN / 传播 | 硬件可能有微小差异 |
| 非规格数 | softfloat 精确处理 | FZ=1 时硬件 flush to zero |
| gather/scatter | 逐元素串行 | 微架构可部分并行 |

### 10.1 已知差异

1. **性能**：浮点 helper 调用 softfloat，每个元素一次函数调用——这是 TCG FP 模拟最大的性能瓶颈
2. **FPSR 更新粒度**：QEMU 在 helper 返回后批量更新 FPSR，真实硬件每条指令结束时更新
3. **SVE 向量操作**：QEMU 无法利用主机 SIMD 来加速 guest SIMD（因为 softfloat 需要完整状态）
4. **整数向量操作**：通过 GVec 框架可以利用主机 SIMD（如 x86 AVX）加速

---

## 十一、源文件索引

| 文件 | 行数 | 关键内容 |
|------|------|----------|
| `target/arm/tcg/translate-a64.c` | 10961 | AArch64 主翻译器：标量FP + NEON + 系统指令 |
| `target/arm/tcg/translate-sve.c` | 8227 | SVE/SVE2 翻译器：谓词化操作、gather/scatter |
| `target/arm/tcg/translate-vfp.c` | 3420 | AArch32 VFP 翻译（CP10/CP11） |
| `target/arm/tcg/translate-neon.c` | 3390 | AArch32 NEON 翻译 |
| `target/arm/tcg/sve_helper.c` | 8634 | SVE 运行时 helper（向量循环） |
| `target/arm/tcg/vfp_helper.c` | 1375 | VFP 浮点 helper |
| `target/arm/tcg/neon_helper.c` | 1452 | NEON helper（AArch32 主要） |
| `target/arm/tcg/a64.decode` | 1927 | AArch64 指令解码描述 |
| `target/arm/tcg/sve.decode` | 1936 | SVE 指令解码描述 |
| `target/arm/tcg/translate-a64.h` | ~190 | 共享声明：vec 偏移、pred 偏移、VL |
| `target/arm/cpu.h:168-702` | — | 寄存器存储：ARMVectorReg、zregs、pregs、fpcr |
| `fpu/softfloat.c` | ~8000 | IEEE 754 软件浮点库 |

| 关键函数/宏 | 位置 | 职责 |
|------------|------|------|
| `fp_access_check()` | translate-a64.c:1455 | FP/SIMD 访问权限检查 |
| `sve_access_check()` | translate-a64.c:1508 | SVE 访问权限检查 |
| `do_fp3_scalar()` | translate-a64.c:5729 | 标量 3 操作数 FP 翻译 |
| `gen_gvec_op3_fpst()` | translate-a64.c:830 | 向量 FP 操作翻译 |
| `gen_gvec_fn2/fn3()` | translate-a64.c:774 | 整数向量操作翻译 |
| `write_fp_dreg_merging()` | translate-a64.c:718 | 标量结果写入（NEP） |
| `vec_full_reg_offset()` | translate-a64.h:110 | 向量寄存器偏移计算 |
| `DO_FP3()` | translate-sve.c:4166 | SVE 浮点操作宏 |
| `trans_FADDA()` | translate-sve.c:4126 | SVE 跨通道浮点累加 |
| `fpstatus_ptr()` | translate.h | 获取 softfloat 状态指针 |
