# ARM64 TCG 内存模型与原子操作深度分析：屏障语义、MemOp 标志、Exclusive Monitor、LSE 原子指令与后端发射

> 基于 QEMU 11.0.50 源码分析，涵盖 TCG 内存排序模型完整子系统：
> TCGBar 屏障类型（TCG_MO_LD_LD/ST_LD/LD_ST/ST_ST/ALL + TCG_BAR_LDAQ/STRL/SC）、
> tcg_gen_mb 屏障生成（user-only CF_PARALLEL 门控 vs softmmu 始终生成）、
> MemOp 操作标志（大小/端序/对齐/原子性 MO_ATOM_IFALIGN/WITHIN16/SUBALIGN/NONE）、
> ARM64 屏障指令翻译（DMB→TCG_MO 映射、DSB 等效处理、ISB→TB 断裂）、
> Load-Acquire/Store-Release（LDAR→LDAQ 后屏障、STLR→STRL 前屏障）、
> Exclusive Monitor（exclusive_addr/val/high 三寄存器、LDXR 设置/STXR CAS 检查/CLREX 清除）、
> gen_load_exclusive/gen_store_exclusive 完整实现、
> LSE 原子指令（LDADD/LDCLR/LDEOR/LDSET/SWP→tcg_gen_atomic_fetch_* 映射）、
> CAS/CASP 比较交换、128 位原子（LDCLRP/LDSETP/SWPP→i128 原语）、
> TCG 原子 Helper 生成（atomic_template.h 模板、atomic_mmu_lookup TLB 交互）、
> AArch64 后端屏障发射（tcg_out_mb→DMB ISH/ISHLD/ISHST）、
> CF_PARALLEL 对内存操作的影响（MTTCG 全屏障 vs RR 非原子优化）。

---

## 目录

1. [架构概述](#1-架构概述)
2. [TCGBar — 屏障类型系统](#2-tcgbar--屏障类型系统)
3. [tcg_gen_mb — 屏障 IR 生成](#3-tcg_gen_mb--屏障-ir-生成)
4. [MemOp — 内存操作标志](#4-memop--内存操作标志)
5. [ARM64 DMB/DSB/ISB 翻译](#5-arm64-dmbdsbisb-翻译)
6. [Load-Acquire / Store-Release](#6-load-acquire--store-release)
7. [Exclusive Monitor 状态](#7-exclusive-monitor-状态)
8. [gen_load_exclusive — 独占加载](#8-gen_load_exclusive--独占加载)
9. [gen_store_exclusive — 独占存储](#9-gen_store_exclusive--独占存储)
10. [LDXR/STXR/LDXP/STXP 翻译](#10-ldxrstxrldxpstxp-翻译)
11. [LSE 原子指令翻译](#11-lse-原子指令翻译)
12. [CAS/CASP 比较交换](#12-cascasp-比较交换)
13. [128 位原子操作](#13-128-位原子操作)
14. [TCG 原子 Helper 生成](#14-tcg-原子-helper-生成)
15. [AArch64 后端屏障发射](#15-aarch64-后端屏障发射)
16. [CF_PARALLEL 与 MTTCG 内存序](#16-cf_parallel-与-mttcg-内存序)
17. [完整内存操作流程总结](#17-完整内存操作流程总结)

---

## 1. 架构概述

TCG 内存模型解决的核心问题：**如何在 host 上正确模拟 guest 的内存排序语义**。

```
Guest ARM64 指令          TCG IR                 Host AArch64 指令
─────────────────    ─────────────────       ─────────────────────
DMB ISH             → INDEX_op_mb(MO_ALL)   → DMB ISH
LDAR X0, [X1]       → qemu_ld + mb(LDAQ)    → LDR + DMB ISHLD
STLR X0, [X1]       → mb(STRL) + qemu_st    → DMB ISH + STR
LDXR X0, [X1]       → qemu_ld + 设置 monitor → LDR + 记录 exclusive
STXR W0, X1, [X2]   → cmpxchg + 清除 monitor → LDXR+STXR 或 CAS
LDADD X0, X1, [X2]  → atomic_fetch_add      → LDADD（LSE host）
CAS X0, X1, [X2]    → atomic_cmpxchg        → CAS（LSE host）
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `include/tcg/tcg-mo.h` | 28-46 | TCGBar 枚举（MO_*/BAR_*） |
| `include/exec/memop.h` | 17-130 | MemOp 完整定义 |
| `tcg/tcg-op.c` | 289-306 | tcg_gen_mb 屏障生成 |
| `tcg/tcg-op-ldst.c` | 48-89 | tcg_canonicalize_memop |
| `tcg/tcg-op-ldst.c` | 880-1320 | 原子操作前端 API |
| `target/arm/tcg/translate-a64.c` | 2237-2282 | CLREX/DMB/DSB/ISB |
| `target/arm/tcg/translate-a64.c` | 3241-3400 | gen_load/store_exclusive |
| `target/arm/tcg/translate-a64.c` | 3502-3597 | LDXR/STXR/LDXP/STXP |
| `target/arm/tcg/translate-a64.c` | 3526-3572 | STLR/LDAR |
| `target/arm/tcg/translate-a64.c` | 4062-4162 | LSE 原子指令 |
| `target/arm/cpu.h` | 704-713 | exclusive_addr/val/high |
| `tcg/aarch64/tcg-target.c.inc` | 1584-1594 | tcg_out_mb 后端发射 |
| `accel/tcg/atomic_template.h` | 80-391 | 原子 helper 模板 |
| `accel/tcg/cputlb.c` | 1799-1903 | atomic_mmu_lookup |

---

## 2. TCGBar — 屏障类型系统

```c
// include/tcg/tcg-mo.h:28-46
typedef enum {
    // 排序类型（A_B 表示 A 在 B 之前保序）
    TCG_MO_LD_LD  = 0x01,  // Load→Load 保序
    TCG_MO_ST_LD  = 0x02,  // Store→Load 保序
    TCG_MO_LD_ST  = 0x04,  // Load→Store 保序
    TCG_MO_ST_ST  = 0x08,  // Store→Store 保序
    TCG_MO_ALL    = 0x0F,  // 全保序

    // 屏障种类
    TCG_BAR_LDAQ  = 0x10,  // Load-Acquire：后续操作不可前移
    TCG_BAR_STRL  = 0x20,  // Store-Release：先前操作不可后移
    TCG_BAR_SC    = 0x30,  // 顺序一致性（LDAQ | STRL）
} TCGBar;
```

### 组合含义

| 组合 | 含义 | ARM 对应 |
|------|------|---------|
| `TCG_BAR_SC \| TCG_MO_ALL` | 全屏障 | DMB SY / DMB ISH |
| `TCG_BAR_SC \| TCG_MO_LD_LD \| TCG_MO_LD_ST` | 读屏障 | DMB ISHLD |
| `TCG_BAR_SC \| TCG_MO_ST_ST` | 写屏障 | DMB ISHST |
| `TCG_BAR_LDAQ \| TCG_MO_ALL` | Load-Acquire | LDAR 后屏障 |
| `TCG_BAR_STRL \| TCG_MO_ALL` | Store-Release | STLR 前屏障 |

---

## 3. tcg_gen_mb — 屏障 IR 生成

```c
// tcg/tcg-op.c:289-306
void tcg_gen_mb(TCGBar mb_type)
{
#ifdef CONFIG_USER_ONLY
    bool parallel = tcg_ctx->gen_tb->cflags & CF_PARALLEL;
#else
    // softmmu 始终生成屏障（即使单 CPU，因为 IO 线程并行运行）
    bool parallel = true;
#endif
    if (parallel) {
        tcg_gen_op1(INDEX_op_mb, 0, mb_type);
    }
}
```

**关键设计**：
- **user-only 模式**：仅在 `CF_PARALLEL`（MTTCG）时生成屏障
- **softmmu 模式**：**始终生成**屏障，因为 virtio 等 IO 线程与 vCPU 并行，需要保序

---

## 4. MemOp — 内存操作标志

```c
// include/exec/memop.h:17-130

// === 大小 ===
MO_8    = 0,  MO_16   = 1,  MO_32   = 2,  MO_64   = 3,
MO_128  = 4,  MO_SIZE = 0x07

// === 符号扩展 ===
MO_SIGN = 0x08

// === 端序 ===
MO_BSWAP = 0x10     // host 反端序
MO_LE / MO_BE       // 基于 host 端序的便捷宏

// === 对齐 ===
MO_UNALN    = 0     // 不检查对齐
MO_ALIGN_2  = 1<<5  // 2 字节对齐
MO_ALIGN_4  = 2<<5  // 4 字节对齐
MO_ALIGN_8  = 3<<5  // 8 字节对齐
MO_ALIGN_16 = 4<<5  // 16 字节对齐
MO_ALIGN    = 7<<5  // 自然对齐

// === ARM 特殊 ===
MO_ALIGN_TLB_ONLY = 1<<8  // 仅 TLB 慢路径检查对齐
                           // Normal 页不检查，Device 页强制对齐

// === 原子性保证 ===
MO_ATOM_IFALIGN       = 0<<9   // 对齐时单拷贝原子（默认）
MO_ATOM_IFALIGN_PAIR  = 1<<9   // 半操作对齐时各自原子（ARM pre-LSE2 LDP）
MO_ATOM_WITHIN16      = 2<<9   // 不跨 16 字节边界则原子（ARM FEAT_LSE2 LDR）
MO_ATOM_WITHIN16_PAIR = 3<<9   // 16 字节内则原子，否则退化为半操作对（LSE2 LDP）
MO_ATOM_SUBALIGN      = 4<<9   // 按对齐粒度部分原子（IBM Power）
MO_ATOM_NONE          = 5<<9   // 无原子性要求
```

### CF_PARALLEL 对原子性的影响

```c
// tcg/tcg-op-ldst.c:48-89  tcg_canonicalize_memop()
// 非 CF_PARALLEL 模式下：
// MO_ATOM_* 降级为 MO_ATOM_NONE → 不生成原子操作
// 这是 RR 模式的关键优化：单线程无需原子性
```

---

## 5. ARM64 DMB/DSB/ISB 翻译

### DMB / DSB

```c
// target/arm/tcg/translate-a64.c:2243-2261
static bool trans_DSB_DMB(DisasContext *s, arg_DSB_DMB *a)
{
    TCGBar bar;
    switch (a->types) {
    case 1: /* MBReqTypes_Reads → 读屏障 */
        bar = TCG_BAR_SC | TCG_MO_LD_LD | TCG_MO_LD_ST;     // 2250
        break;
    case 2: /* MBReqTypes_Writes → 写屏障 */
        bar = TCG_BAR_SC | TCG_MO_ST_ST;                     // 2253
        break;
    default: /* MBReqTypes_All → 全屏障 */
        bar = TCG_BAR_SC | TCG_MO_ALL;                       // 2256
        break;
    }
    tcg_gen_mb(bar);
    return true;
}
```

**注意**：QEMU 将 DMB 和 DSB **等效处理**（2245 注释），因为 TCG 不模拟缓存一致性协议层面的差异。

### ISB

```c
// target/arm/tcg/translate-a64.c:2272-2281
static bool trans_ISB(DisasContext *s, arg_ISB *a)
{
    reset_btype(s);                  // 2279: 清除 BTI 状态
    gen_goto_tb(s, 0, 4);           // 2280: 终止当前 TB，开新 TB
    return true;
}
```

ISB **不生成屏障 IR**，而是通过**断裂 TB** 实现：后续指令在新 TB 中翻译，确保所有前序副作用（如系统寄存器写入、TLB 操作）生效。

---

## 6. Load-Acquire / Store-Release

### STLR（Store-Release）

```c
// target/arm/tcg/translate-a64.c:3526-3549
static bool trans_STLR(DisasContext *s, arg_stlr *a)
{
    tcg_gen_mb(TCG_MO_ALL | TCG_BAR_STRL);          // 3543: 屏障在 store 之前
    memop = check_ordered_align(s, a->rn, 0, true, a->sz);
    clean_addr = gen_mte_check1(s, ...);
    do_gpr_st(s, cpu_reg(s, a->rt), clean_addr, memop, ...);
    return true;
}
```

### LDAR（Load-Acquire）

```c
// target/arm/tcg/translate-a64.c:3552-3572
static bool trans_LDAR(DisasContext *s, arg_stlr *a)
{
    memop = check_ordered_align(s, a->rn, 0, false, a->sz);
    clean_addr = gen_mte_check1(s, ...);
    do_gpr_ld(s, cpu_reg(s, a->rt), clean_addr, memop, ...);
    tcg_gen_mb(TCG_MO_ALL | TCG_BAR_LDAQ);          // 3571: 屏障在 load 之后
    return true;
}
```

**语义**：
- **STLR**：`mb(STRL)` → `store` — 确保之前所有操作在 store 之前完成
- **LDAR**：`load` → `mb(LDAQ)` — 确保 load 在之后所有操作之前完成

---

## 7. Exclusive Monitor 状态

```c
// target/arm/cpu.h:704-713
uint64_t exclusive_addr;   // 704: 监控的虚拟地址（-1 表示无监控）
uint64_t exclusive_val;    // 705: 监控的值（LDXR 读取的值）
uint64_t exclusive_high;   // 713: LDXP 第二个 64 位值（128 位配对）
```

### 清除监控

```c
// target/arm/internals.h:693-697
static inline void arm_clear_exclusive(CPUARMState *env)
{
    env->exclusive_addr = -1;
}
// 在异常入口、上下文切换、CLREX 指令时调用
```

### CLREX 指令

```c
// target/arm/tcg/translate-a64.c:2237-2240
static bool trans_CLREX(DisasContext *s, arg_CLREX *a)
{
    tcg_gen_movi_i64(cpu_exclusive_addr, -1);
    return true;
}
```

---

## 8. gen_load_exclusive — 独占加载

```c
// target/arm/tcg/translate-a64.c:3241-3284
static void gen_load_exclusive(DisasContext *s, int rt, int rt2, int rn,
                               int size, bool is_pair)
{
    s->is_ldex = true;                               // 3248: 标记为独占加载
    clean_addr = gen_mte_check1(s, ...);             // MTE 标签检查

    if (is_pair) {
        if (size == 2) {
            // 32+32 = 64 位配对
            tcg_gen_qemu_ld_i64(cpu_exclusive_val, clean_addr, idx, memop);
            // 拆分到 rt 和 rt2
            tcg_gen_extract_i64(cpu_reg(s, rt), cpu_exclusive_val, 0, 32);
            tcg_gen_extract_i64(cpu_reg(s, rt2), cpu_exclusive_val, 32, 32);
        } else {
            // 64+64 = 128 位配对
            TCGv_i128 t16 = tcg_temp_new_i128();
            tcg_gen_qemu_ld_i128(t16, clean_addr, idx, memop);
            tcg_gen_extr_i128_i64(cpu_exclusive_val, cpu_exclusive_high, t16);
            tcg_gen_mov_i64(cpu_reg(s, rt), cpu_exclusive_val);
            tcg_gen_mov_i64(cpu_reg(s, rt2), cpu_exclusive_high);
        }
    } else {
        // 单寄存器：8/16/32/64 位
        tcg_gen_qemu_ld_i64(cpu_exclusive_val, clean_addr, idx, memop);
        tcg_gen_mov_i64(cpu_reg(s, rt), cpu_exclusive_val);
    }

    // 记录监控地址
    tcg_gen_mov_i64(cpu_exclusive_addr, clean_addr);  // 3283
}
```

---

## 9. gen_store_exclusive — 独占存储

```c
// target/arm/tcg/translate-a64.c:3286-3400
static void gen_store_exclusive(DisasContext *s, int rd, int rt, int rt2,
                                int rn, int size, int is_pair)
{
    // 伪代码：
    // if (env->exclusive_addr == addr &&
    //     env->exclusive_val == [addr]) {
    //     [addr] = Rt;
    //     Rd = 0;  // 成功
    // } else {
    //     Rd = 1;  // 失败
    // }
    // env->exclusive_addr = -1;

    // 1. 地址比较
    clean_addr = clean_data_tbi(s, cpu_reg_sp(s, rn));
    tcg_gen_brcond_i64(TCG_COND_NE, clean_addr,
                       cpu_exclusive_addr, fail_label);   // 3316

    // 2. 值比较 + 条件存储（使用 atomic cmpxchg）
    if (is_pair && size == MO_64) {
        // 128 位：tcg_gen_atomic_cmpxchg_i128
        tcg_gen_atomic_cmpxchg_i128(t16, clean_addr, c16, s16, idx, memop);
    } else {
        // 64 位及以下：tcg_gen_atomic_cmpxchg_i64
        tcg_gen_atomic_cmpxchg_i64(tmp, clean_addr,
                                   cpu_exclusive_val, val, idx, memop);
    }

    // 3. 检查 CAS 结果
    tcg_gen_brcond_i64(TCG_COND_NE, tmp, cpu_exclusive_val, fail_label);

    // 4. 成功路径
    tcg_gen_movi_i32(cpu_reg(s, rd), 0);
    tcg_gen_br(done_label);

    // 5. 失败路径
    gen_set_label(fail_label);
    tcg_gen_movi_i32(cpu_reg(s, rd), 1);

    // 6. 清除监控
    gen_set_label(done_label);
    tcg_gen_movi_i64(cpu_exclusive_addr, -1);            // 3399
}
```

**关键设计**：STXR 实现使用 **CAS（Compare-And-Swap）** 原语而非真正的 exclusive monitor 硬件。这在 MTTCG 多线程环境下保证原子性，但可能产生虚假成功（ABA 问题），对 QEMU 来说这是可接受的。

---

## 10. LDXR/STXR/LDXP/STXP 翻译

```c
// target/arm/tcg/translate-a64.c:3502-3597

// STXR / STLXR
static bool trans_STXR(DisasContext *s, arg_stxr *a) {     // 3502
    if (a->lasr) tcg_gen_mb(TCG_MO_ALL | TCG_BAR_STRL);   // 3508: Release
    gen_store_exclusive(s, a->rs, a->rt, a->rt2, a->rn, a->sz, false);
}

// LDXR / LDAXR
static bool trans_LDXR(DisasContext *s, arg_stxr *a) {     // 3514
    gen_load_exclusive(s, a->rt, a->rt2, a->rn, a->sz, false);
    if (a->lasr) tcg_gen_mb(TCG_MO_ALL | TCG_BAR_LDAQ);   // 3521: Acquire
}

// STXP / STLXP
static bool trans_STXP(DisasContext *s, arg_stxr *a) {     // 3575
    if (a->lasr) tcg_gen_mb(TCG_MO_ALL | TCG_BAR_STRL);
    gen_store_exclusive(s, a->rs, a->rt, a->rt2, a->rn, a->sz, true);
}

// LDXP / LDAXP
static bool trans_LDXP(DisasContext *s, arg_stxr *a) {     // 3587
    gen_load_exclusive(s, a->rt, a->rt2, a->rn, a->sz, true);
    if (a->lasr) tcg_gen_mb(TCG_MO_ALL | TCG_BAR_LDAQ);
}
```

**lasr 标志**：当 `lasr=1` 时为 LDAXR/STLXR（带 Acquire/Release 语义），通过附加 `tcg_gen_mb` 实现。

---

## 11. LSE 原子指令翻译

### do_atomic_ld — 通用原子加载-修改框架

```c
// target/arm/tcg/translate-a64.c:4062-4103
static bool do_atomic_ld(DisasContext *s, arg_atomic *a,
                         AtomicThreeOpFn *fn, int sign, bool invert)
{
    mop = check_atomic_align(s, a->rn, mop);
    clean_addr = gen_mte_check1(s, ...);
    tcg_rs = read_cpu_reg(s, a->rs, true);
    if (invert) tcg_gen_not_i64(tcg_rs, tcg_rs);  // LDCLR 取反

    // TCG 原子原语都是全屏障，因此忽略 Acquire/Release 位
    fn(tcg_rt, clean_addr, tcg_rs, get_mem_index(s), mop);  // 4083
}
```

### LSE 指令到 TCG 原语映射

```c
// target/arm/tcg/translate-a64.c:4105-4113
LDADD  → tcg_gen_atomic_fetch_add_i64    // 原子加
LDCLR  → tcg_gen_atomic_fetch_and_i64    // 原子清位（取反后 AND）
LDEOR  → tcg_gen_atomic_fetch_xor_i64    // 原子异或
LDSET  → tcg_gen_atomic_fetch_or_i64     // 原子置位
LDSMAX → tcg_gen_atomic_fetch_smax_i64   // 原子有符号最大值
LDSMIN → tcg_gen_atomic_fetch_smin_i64   // 原子有符号最小值
LDUMAX → tcg_gen_atomic_fetch_umax_i64   // 原子无符号最大值
LDUMIN → tcg_gen_atomic_fetch_umin_i64   // 原子无符号最小值
SWP    → tcg_gen_atomic_xchg_i64         // 原子交换
```

---

## 12. CAS/CASP 比较交换

### CAS（64 位）

```c
// target/arm/tcg/translate-a64.c:3402-3416
static void gen_compare_and_swap(DisasContext *s, int rs, int rt,
                                 int rn, int size)
{
    // 使用 tcg_gen_atomic_cmpxchg_i64
    // 比较 [addr] 与 Rs，相等则写入 Rt
    tcg_gen_atomic_cmpxchg_i64(tcg_rs, clean_addr, tcg_rs, tcg_rt,
                               get_mem_index(s), memop);
}
```

### CASP（128 位配对）

```c
// target/arm/tcg/translate-a64.c:3420-3477
static void gen_compare_and_swap_pair(DisasContext *s, int rs, int rt,
                                      int rn, int size)
{
    if (size == 2) {
        // 32+32 → 64 位 cmpxchg
        tcg_gen_atomic_cmpxchg_i64(cmp, clean_addr, cmp, val, idx, memop);
    } else {
        // 64+64 → 128 位 cmpxchg
        tcg_gen_atomic_cmpxchg_i128(c128, clean_addr, c128, v128, idx, memop);
    }
}
```

---

## 13. 128 位原子操作

### FEAT_LSE128 指令

```c
// target/arm/tcg/translate-a64.c:4117-4162
static bool do_atomic128_ld(DisasContext *s, arg_atomic128 *a,
                            Atomic128ThreeOpFn *fn, bool invert)
{
    mop = check_atomic_align(s, a->rn, MO_128);
    // TCG 原子原语都是全屏障
    t16 = tcg_temp_new_i128();
    tcg_gen_concat_i64_i128(t16, tlo, thi);
    fn(t16, clean_addr, t16, get_mem_index(s), mop);      // 4151
    tcg_gen_extr_i128_i64(cpu_reg(s, rlo), cpu_reg(s, rhi), t16);
}

// 4157-4162
LDCLRP → tcg_gen_atomic_fetch_and_i128   // 128 位原子清位
LDSETP → tcg_gen_atomic_fetch_or_i128    // 128 位原子置位
SWPP   → tcg_gen_atomic_xchg_i128        // 128 位原子交换
```

---

## 14. TCG 原子 Helper 生成

### 前端 API 分发

```c
// tcg/tcg-op-ldst.c:1240-1320
// CF_PARALLEL 模式：
//   → 调用 helper_atomic_fetch_add_i64 等原子 helper
// 非 CF_PARALLEL 模式：
//   → 退化为普通 load + 计算 + store（非原子）
```

### atomic_template.h 模板

```c
// accel/tcg/atomic_template.h:80-391
// 通过宏生成所有大小（1/2/4/8 字节）× 所有端序的原子 helper：
//   helper_atomic_cmpxchg{l,q}_{le,be}
//   helper_atomic_fetch_add{b,w,l,q}_{le,be}
//   helper_atomic_fetch_and{b,w,l,q}_{le,be}
//   ...

// 每个 helper 的流程：
// 1. atomic_mmu_lookup(cpu, addr, ...) → 获取 host 地址
// 2. 在 host 地址上执行原生原子操作（GCC __atomic_* 或 cmpxchg 循环）
// 3. 对于 acquire/release 变体，添加 smp_mb()
```

### atomic_mmu_lookup — TLB 交互

```c
// accel/tcg/cputlb.c:1799-1903
static void *atomic_mmu_lookup(CPUState *cpu, vaddr addr,
                               MemOpIdx oi, int size, uintptr_t retaddr)
{
    // 1. TLB 查找（使用写地址，因为原子操作需要写权限）
    // 2. 对齐检查（原子操作通常要求自然对齐）
    // 3. MMIO 检查 → 不支持原子 MMIO，触发 EXCP_ATOMIC
    // 4. 返回 host 地址 = addr + entry->addend
}
```

---

## 15. AArch64 后端屏障发射

```c
// tcg/aarch64/tcg-target.c.inc:1584-1594
static void tcg_out_mb(TCGContext *s, unsigned a0)
{
    static const uint32_t sync[] = {
        [0 ... TCG_MO_ALL]            = DMB_ISH | DMB_LD | DMB_ST,  // 全屏障
        [TCG_MO_ST_ST]                = DMB_ISH | DMB_ST,           // 写屏障
        [TCG_MO_LD_LD]                = DMB_ISH | DMB_LD,           // 读屏障
        [TCG_MO_LD_ST]                = DMB_ISH | DMB_LD,           // 读→写屏障
        [TCG_MO_LD_ST | TCG_MO_LD_LD] = DMB_ISH | DMB_LD,          // 读+读→写
    };
    tcg_out32(s, sync[a0 & TCG_MO_ALL]);
}
```

### 映射关系

| TCG 屏障 | AArch64 host 指令 | 含义 |
|----------|------------------|------|
| `TCG_MO_ALL` | `DMB ISH` | 全屏障（内部共享域） |
| `TCG_MO_ST_ST` | `DMB ISHST` | 仅 Store→Store |
| `TCG_MO_LD_LD` | `DMB ISHLD` | 仅 Load→Load |
| `TCG_MO_LD_ST` | `DMB ISHLD` | Load→Store（用 ISHLD 覆盖） |

**注意**：所有屏障使用 **ISH**（Inner Shareable）域，因为 QEMU vCPU 线程共享同一进程地址空间。

---

## 16. CF_PARALLEL 与 MTTCG 内存序

### CF_PARALLEL 的全局影响

```
MTTCG 模式（多线程）:
  CF_PARALLEL = 1
  → tcg_gen_mb 始终生成 INDEX_op_mb
  → tcg_canonicalize_memop 保留 MO_ATOM_* 原子性标志
  → 原子操作使用 helper_atomic_* 原子 helper
  → gen_store_exclusive 使用 atomic cmpxchg

RR 模式（单线程）:
  CF_PARALLEL = 0
  → user-only: tcg_gen_mb 不生成屏障（被优化掉）
  → softmmu: tcg_gen_mb 仍然生成（IO 线程需要）
  → tcg_canonicalize_memop 将 MO_ATOM_* 降为 MO_ATOM_NONE
  → 原子操作退化为普通 load+compute+store
  → gen_store_exclusive 仍使用 cmpxchg（单线程无竞争）
```

### guest_default_memory_order

```c
// include/accel/tcg/cpu-ops.h:22-44
// target 可定义 guest_default_memory_order 用于 tcg_gen_req_mo()
// ARM64 是弱序架构，默认 memory order 较低
// 只有显式屏障指令才生成强保序
```

---

## 17. 完整内存操作流程总结

### 普通 Load/Store

```
Guest LDR X0, [X1]
  → tcg_gen_qemu_ld_i64(val, addr, idx, memop)
  → 后端：TLB 内联查找 → host LDR（无屏障）
```

### Load-Acquire

```
Guest LDAR X0, [X1]
  → tcg_gen_qemu_ld_i64(val, addr, idx, memop)
  → tcg_gen_mb(TCG_MO_ALL | TCG_BAR_LDAQ)
  → 后端：host LDR → DMB ISHLD
```

### Store-Release

```
Guest STLR X0, [X1]
  → tcg_gen_mb(TCG_MO_ALL | TCG_BAR_STRL)
  → tcg_gen_qemu_st_i64(val, addr, idx, memop)
  → 后端：DMB ISH → host STR
```

### Exclusive (LDXR/STXR)

```
Guest LDXR X0, [X1]
  → gen_load_exclusive:
    qemu_ld → cpu_exclusive_val
    cpu_exclusive_addr = clean_addr

Guest STXR W0, X2, [X1]
  → gen_store_exclusive:
    比较 clean_addr == cpu_exclusive_addr
    → 匹配: atomic_cmpxchg(addr, exclusive_val, X2)
      → 成功: W0 = 0
      → 失败: W0 = 1
    → 不匹配: W0 = 1
    cpu_exclusive_addr = -1
```

### LSE 原子操作

```
Guest LDADD X0, X1, [X2]
  → do_atomic_ld:
    tcg_gen_atomic_fetch_add_i64(rt, addr, rs, idx, mop)
    → CF_PARALLEL: helper_atomic_fetch_add_i64
      → atomic_mmu_lookup → host 地址
      → __atomic_fetch_add (GCC 原生原子)
    → 非 CF_PARALLEL: load + add + store（非原子）
```

### 屏障

```
Guest DMB ISH
  → tcg_gen_mb(TCG_BAR_SC | TCG_MO_ALL)
  → INDEX_op_mb IR
  → 后端: DMB ISH (AArch64 host)

Guest ISB
  → gen_goto_tb(s, 0, 4)  // 终止 TB
  → 无屏障 IR，通过 TB 断裂保证
```

---

## 交叉参考

- [44-ARM64-TCG执行循环深度分析](44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md) — MTTCG/RR 线程模型、EXCP_ATOMIC、cpu_exec_step_atomic
- [43-ARM64-TCG-softmmu-TLB深度分析](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — atomic_mmu_lookup、TLB 快慢路径
- [42-ARM64-TCG前端后端代码生成深度分析](42-ARM64-TCG前端后端代码生成深度分析-IR中间表示-翻译循环-优化Pass-寄存器分配与AArch64代码发射.md) — TCG IR 系统、后端代码发射

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
