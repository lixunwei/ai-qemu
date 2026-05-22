# TCG 前端 IR 生成与优化 Pass 深度分析

> 基于 QEMU 11.0.50 源码分析，聚焦 TCG (Tiny Code Generator) 核心翻译流水线：
> IR 操作码体系设计、TCGTemp 五种生命周期与临时变量管理、tcg_gen_* 前端 IR 生成 API 层次、
> tcg_optimize 优化 Pass 详解（常量折叠/拷贝传播/掩码追踪/内存拷贝消除/死代码消除/条件折叠）、
> 活性分析（liveness_pass_0/1/2）、tcg_gen_code 后端流水线（寄存器分配/代码发射）。
> 分析 ARM64 前端如何将 guest 指令映射为 TCG IR，以及优化器如何缩减 IR 体积。

---

## 目录

1. [TCG 翻译流水线总体架构](#1-tcg-翻译流水线总体架构)
2. [IR 操作码体系 (tcg-opc.h)](#2-ir-操作码体系)
3. [TCGTemp 临时变量系统](#3-tcgtemp-临时变量系统)
4. [TCGOp 操作节点与链表](#4-tcgop-操作节点与链表)
5. [tcg_gen_* 前端 IR 生成 API](#5-前端-ir-生成-api)
6. [ARM64 前端翻译实例](#6-arm64-前端翻译实例)
7. [tcg_optimize 优化 Pass](#7-tcg_optimize-优化-pass)
8. [活性分析 (Liveness Analysis)](#8-活性分析)
9. [tcg_gen_code 后端流水线](#9-tcg_gen_code-后端流水线)
10. [Guest 内存访问 IR 生成](#10-guest-内存访问-ir-生成)
11. [与硬件实际执行的对比](#11-与硬件实际执行的对比)

---

## 1. TCG 翻译流水线总体架构

```
Guest ARM64 指令
    │
    ▼
┌─────────────────────────────────────────┐
│  前端 (translate-a64.c)                  │
│  decodetree 解码 → trans_* 函数          │
│  调用 tcg_gen_* API 生成 IR ops          │
└──────────────────┬──────────────────────┘
                   │ TCGOp 链表 (未优化 IR)
                   ▼
┌─────────────────────────────────────────┐
│  tcg_optimize (optimize.c)              │
│  - 常量折叠 (Constant Folding)           │
│  - 拷贝传播 (Copy Propagation)           │
│  - 掩码追踪 (z_mask/o_mask/s_mask)       │
│  - 内存拷贝消除 (Memory Copy Tracking)    │
│  - 死代码消除 (Dead Code Elimination)     │
│  - 条件简化 (Condition Simplification)    │
└──────────────────┬──────────────────────┘
                   │ 优化后 TCGOp 链表
                   ▼
┌─────────────────────────────────────────┐
│  reachable_code_pass (tcg.c:3565)       │
│  删除不可达代码                           │
├─────────────────────────────────────────┤
│  liveness_pass_0/1/2 (tcg.c:3804-4270)  │
│  - pass_0: TEMP_TB→TEMP_EBB 降级        │
│  - pass_1: 反向扫描标记 DEAD/SYNC        │
│  - pass_2: indirect→direct 替换         │
├─────────────────────────────────────────┤
│  tcg_gen_code (tcg.c:6556)              │
│  - tcg_reg_alloc_start: 寄存器初始化     │
│  - 遍历 op 链表: 寄存器分配 + 代码发射    │
│  - tcg_out_ldst_finalize: 慢路径修正     │
│  - tcg_resolve_relocs: 重定位            │
│  - flush_idcache_range: I-cache 刷新    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
              Host 机器码 (TB)
```

**关键调用链**：
```
tb_gen_code()
  → aarch64_translate_code()
    → translator_loop()
      → aarch64_tr_translate_insn()  [循环翻译每条 guest 指令]
  → tcg_gen_code()
    → tcg_optimize()
    → reachable_code_pass()
    → liveness_pass_0() / liveness_pass_1() / liveness_pass_2()
    → [遍历 op 链表] tcg_reg_alloc_op() → tcg_out_* [代码发射]
```

---

## 2. IR 操作码体系

### 2.1 操作码定义 (include/tcg/tcg-opc.h)

所有操作码通过 `DEF(name, oargs, iargs, cargs, flags)` 宏定义：

```c
// include/tcg/tcg-opc.h:26-27
// DEF(name, oargs, iargs, cargs, flags)
//   oargs: 输出参数数
//   iargs: 输入参数数
//   cargs: 常量参数数
//   flags: 操作码标志
```

### 2.2 标量操作码分类 (行 30-124)

| 类别 | 操作码 | 输出/输入/常量 | 说明 |
|------|--------|---------------|------|
| **控制流** | `discard` | 1/0/0 | 丢弃 temp |
| | `set_label` | 0/0/1 | 设置标签 |
| | `br` | 0/0/1 | 无条件分支 |
| | `brcond` | 0/2/2 | 条件分支 (BB_END) |
| **内存屏障** | `mb` | 0/0/1 | 内存屏障 |
| **数据移动** | `mov` | 1/1/0 | 寄存器传送 |
| **算术** | `add`, `sub`, `mul`, `neg` | 1/2/0 或 1/1/0 | 基本算术 |
| | `divs`, `divu`, `rems`, `remu` | 1/2/0 | 除法/取余 |
| | `muls2`, `mulu2` | 2/2/0 | 宽乘法 (128-bit 结果) |
| | `mulsh`, `muluh` | 1/2/0 | 乘法高位 |
| **逻辑** | `and`, `or`, `xor` | 1/2/0 | 位逻辑 |
| | `andc`, `orc`, `eqv`, `nand`, `nor`, `not` | 1/2/0 或 1/1/0 | 复合逻辑 |
| **移位** | `shl`, `shr`, `sar`, `rotl`, `rotr` | 1/2/0 | 移位/旋转 |
| **位操作** | `deposit` | 1/2/2 | 位域插入 |
| | `extract`, `sextract` | 1/1/2 | 位域提取 |
| | `extract2` | 1/2/1 | 双寄存器位域 |
| **比较** | `setcond`, `negsetcond` | 1/2/1 | 条件设置 |
| | `movcond` | 1/4/1 | 条件选择 |
| **字节序** | `bswap16/32/64` | 1/1/1 | 字节翻转 |
| **位计数** | `clz`, `ctz`, `ctpop` | 1/2/0 或 1/1/0 | 前导零等 |
| **进位链** | `addco`, `addc1o`, `addci`, `addcio` | 1/2/0 | 多精度加法 |
| | `subbo`, `subb1o`, `subbi`, `subbio` | 1/2/0 | 多精度减法 |
| **类型转换** | `ext_i32_i64`, `extu_i32_i64` | 1/1/0 | 32→64 扩展 |
| | `extrl_i64_i32`, `extrh_i64_i32` | 1/1/0 | 64→32 截断 |
| **加载/存储** | `ld8u/s`, `ld16u/s`, `ld32u/s`, `ld` | 1/1/1 | 从 env 加载 |
| | `st8`, `st16`, `st32`, `st` | 0/2/1 | 存储到 env |
| **TB 控制** | `exit_tb` | 0/0/1 | 退出 TB |
| | `goto_tb` | 0/0/1 | TB 直接链接 |
| | `goto_ptr` | 0/1/0 | 间接 TB 跳转 |
| **Guest 内存** | `qemu_ld` | 1/1/1 | Guest 加载 (含 TLB) |
| | `qemu_st` | 0/2/1 | Guest 存储 (含 TLB) |
| | `qemu_ld2` | 2/1/1 | 128-bit 加载 |
| | `qemu_st2` | 0/3/1 | 128-bit 存储 |

### 2.3 进位链操作码 (新特性)

```c
// tcg-opc.h:96-104 — 进位链操作码
DEF(addco, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_OUT)   // add，产生进位
DEF(addc1o, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_OUT)  // add+1，产生进位
DEF(addci, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_IN)    // add+carry_in
DEF(addcio, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_IN | TCG_OPF_CARRY_OUT)

DEF(subbo, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_OUT)   // sub，产生借位
DEF(subb1o, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_OUT)  // sub-1，产生借位
DEF(subbi, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_IN)    // sub-borrow_in
DEF(subbio, 1, 2, 0, TCG_OPF_INT | TCG_OPF_CARRY_IN | TCG_OPF_CARRY_OUT)
```

进位链是 QEMU 11.0 新引入的机制，用于高效翻译多精度算术（如 ARM64 的 ADCS/SBCS 指令链）。
隐式进位通过 `TCGContext.carry_live` 标志追踪，避免显式 temp 传递开销。

### 2.4 向量操作码 (行 126-183)

覆盖 SIMD 运算：`add_vec`, `sub_vec`, `mul_vec`, `cmp_vec`, `bitsel_vec` 等。
支持 V64/V128/V256 三种向量宽度，通过 `TCGOP_VECE` 指定元素宽度。

### 2.5 操作码标志

```c
TCG_OPF_BB_END         // 基本块终止
TCG_OPF_BB_EXIT        // TB 退出
TCG_OPF_CALL_CLOBBER   // 破坏 caller-saved 寄存器
TCG_OPF_SIDE_EFFECTS   // 有副作用，不可消除
TCG_OPF_NOT_PRESENT    // 不直接出现在后端（前端伪操作）
TCG_OPF_INT            // 整数操作
TCG_OPF_VECTOR         // 向量操作
TCG_OPF_COND_BRANCH    // 条件分支
TCG_OPF_CARRY_IN       // 消费隐式进位
TCG_OPF_CARRY_OUT      // 产生隐式进位
```

---

## 3. TCGTemp 临时变量系统

### 3.1 五种生命周期

```c
// include/tcg/tcg.h:253-268
typedef enum TCGTempKind {
    TEMP_EBB,      // 扩展基本块内有效，跨分支即死亡
    TEMP_TB,       // 整个 TB 内有效
    TEMP_GLOBAL,   // 跨 TB 有效 (CPU 寄存器映射)
    TEMP_FIXED,    // 固定物理寄存器 (如 env 指针)
    TEMP_CONST,    // 常量 (只读，不参与分配)
} TCGTempKind;
```

### 3.2 TCGTemp 结构体

```c
// include/tcg/tcg.h:270-293
typedef struct TCGTemp {
    TCGReg reg:8;           // 当前分配的物理寄存器
    TCGTempVal val_type:8;  // DEAD/REG/MEM/CONST
    TCGType base_type:8;    // 基础类型
    TCGType type:8;         // 当前类型 (I32/I64/V64/V128/V256)
    TCGTempKind kind:3;     // 生命周期种类
    unsigned int indirect_reg:1;
    unsigned int indirect_base:1;
    unsigned int mem_coherent:1;  // 内存映像是否与寄存器一致
    unsigned int mem_allocated:1; // 是否已分配栈空间
    int64_t val;                  // 常量值 (TEMP_CONST)
    struct TCGTemp *mem_base;     // 内存基址 temp
    intptr_t mem_offset;          // 内存偏移
    const char *name;             // 调试名称
    uintptr_t state;              // Pass 专用
    void *state_ptr;              // Pass 专用指针
} TCGTemp;
```

### 3.3 ARM64 CPU 寄存器映射为 TEMP_GLOBAL

```c
// target/arm/tcg/translate.c 中初始化：
cpu_X[0..30]  → TEMP_GLOBAL (偏移到 env->xregs[i])
cpu_pc        → TEMP_GLOBAL (偏移到 env->pc)
cpu_NF/ZF/CF/VF → TEMP_GLOBAL (条件标志)
```

TEMP_GLOBAL 的特殊性：
- 在 TB 边界必须同步到内存（env 结构体）
- helper 调用可能读/写全局变量 → 活性分析需考虑

### 3.4 TEMP_EBB vs TEMP_TB

`liveness_pass_0` (tcg.c:3804-3866) 负责将仅在单个 EBB 中使用的 TEMP_TB 降级为 TEMP_EBB：
- TEMP_EBB 在基本块末尾可直接标记为死亡
- TEMP_TB 需在 TB 内所有 EBB 间保持活性

```c
// tcg.c:3860-3865
for (int i = s->nb_globals; i < nb_temps; ++i) {
    TCGTemp *ts = &s->temps[i];
    if (ts->kind == TEMP_TB && ts->state_ptr != multiple_ebb) {
        ts->kind = TEMP_EBB;  // 降级：仅在单 EBB 使用
    }
}
```

---

## 4. TCGOp 操作节点与链表

### 4.1 TCGOp 结构体

```c
// include/tcg/tcg.h:310-329
struct TCGOp {
    TCGOpcode opc   : 8;       // 操作码
    unsigned nargs  : 8;       // 参数数量
    unsigned param1 : 8;       // 多用途: TYPE (普通op) / CALLI (call)
    unsigned param2 : 8;       // 多用途: FLAGS/VECE (普通op) / CALLO (call)
    TCGLifeData life;          // 活性位图: DEAD_ARG | SYNC_ARG
    QTAILQ_ENTRY(TCGOp) link;  // 双向链表
    TCGRegSet output_pref[2];  // 输出寄存器偏好
    TCGArg args[];             // 变长参数数组
};
```

### 4.2 Op 发射流程

```c
// tcg/tcg-op.c:40-46
TCGOp *tcg_gen_op1(TCGOpcode opc, TCGType type, TCGArg a1)
{
    TCGOp *op = tcg_emit_op(opc, 1);  // 分配并追加到链表
    TCGOP_TYPE(op) = type;
    op->args[0] = a1;
    return op;
}

// tcg/tcg.c:3510-3519 (tcg_emit_op)
// 将新 op 追加到 tcg_ctx->ops 尾部（或 emit_before_op 前）
```

### 4.3 链表操作

- `tcg_op_remove(s, op)`: 从链表删除（死代码消除使用）
- `tcg_op_insert_before/after`: 插入新 op（优化器使用）
- 遍历: `QTAILQ_FOREACH(op, &s->ops, link)` (正向)
- 反向遍历: `QTAILQ_FOREACH_REVERSE_SAFE` (活性分析使用)

---

## 5. 前端 IR 生成 API

### 5.1 API 层次结构

```
高层 API (tcg-op-common.h 声明)
  │
  ├── tcg_gen_add_i64(ret, a, b)     → tcg_gen_op3_i64(INDEX_op_add, ...)
  ├── tcg_gen_addi_i64(ret, a, imm)  → 优化: imm==0 → mov, else → add + constant
  ├── tcg_gen_andi_i64(ret, a, mask) → 优化: mask==0→movi, -1→mov, 2^n-1→extract
  │
  ├── tcg_gen_qemu_ld_i64(v, addr, idx, memop) → gen_ldst1(qemu_ld, ...)
  ├── tcg_gen_qemu_st_i64(v, addr, idx, memop) → gen_ldst1(qemu_st, ...)
  │
  ├── tcg_gen_brcond_i64(cond, a, b, label)  → INDEX_op_brcond
  ├── tcg_gen_goto_tb(idx)                    → INDEX_op_goto_tb
  └── tcg_gen_exit_tb(tb, idx)                → INDEX_op_exit_tb

底层 API (tcg-op.c:40-105)
  │
  └── tcg_gen_op1/2/3/4/5/6(opc, type, args...) → tcg_emit_op(opc, nargs)
```

### 5.2 前端优化 (tcg-op.c 中的简单优化)

tcg_gen_* 函数在生成 IR 之前会进行简单优化：

```c
// tcg_gen_addi_i32: imm==0 → mov
void tcg_gen_addi_i32(TCGv_i32 ret, TCGv_i32 arg1, int32_t arg2) {
    if (arg2 == 0) {
        tcg_gen_mov_i32(ret, arg1);   // 直接消除
    } else {
        tcg_gen_add_i32(ret, arg1, tcg_constant_i32(arg2));
    }
}

// tcg_gen_andi_i32: 特殊 mask 处理
void tcg_gen_andi_i32(TCGv_i32 ret, TCGv_i32 arg1, int32_t arg2) {
    switch (arg2) {
    case 0:  tcg_gen_movi_i32(ret, 0); return;      // 全零
    case -1: tcg_gen_mov_i32(ret, arg1); return;     // 全一
    default:
        if (!(arg2 & (arg2 + 1))) {                  // 连续低位掩码
            tcg_gen_extract_i32(ret, arg1, 0, len);  // 转为 extract
            return;
        }
    }
    tcg_gen_and_i32(ret, arg1, tcg_constant_i32(arg2));
}

// tcg_gen_muli_i32: 2^n 转为移位
void tcg_gen_muli_i32(TCGv_i32 ret, TCGv_i32 arg1, int32_t arg2) {
    if (arg2 == 0) tcg_gen_movi_i32(ret, 0);
    else if (is_power_of_2(arg2)) tcg_gen_shli_i32(ret, arg1, ctz32(arg2));
    else tcg_gen_mul_i32(ret, arg1, tcg_constant_i32(arg2));
}
```

### 5.3 后端能力探测

```c
// 检查后端是否支持某操作码，不支持则分解为更基础的 op
if (tcg_op_supported(INDEX_op_andc, TCG_TYPE_I32, 0)) {
    tcg_gen_op3_i32(INDEX_op_andc, ret, arg1, arg2);
} else {
    tcg_gen_not_i32(t0, arg2);
    tcg_gen_and_i32(ret, arg1, t0);
}
```

---

## 6. ARM64 前端翻译实例

### 6.1 ADD_i (立即数加法)

```c
// target/arm/tcg/translate-a64.c:4994
TRANS(ADD_i, gen_rri, a, 1, 1, tcg_gen_add_i64)

// gen_rri 展开 (L4957-4968):
static bool gen_rri(DisasContext *s, arg_rri_sf *a,
                    bool rd_sp, bool rn_sp, ArithTwoOp *fn) {
    TCGv_i64 tcg_rn = rn_sp ? cpu_reg_sp(s, a->rn) : cpu_reg(s, a->rn);
    TCGv_i64 tcg_rd = rd_sp ? cpu_reg_sp(s, a->rd) : cpu_reg(s, a->rd);
    TCGv_i64 tcg_imm = tcg_constant_i64(a->imm);
    fn(tcg_rd, tcg_rn, tcg_imm);    // → tcg_gen_add_i64(rd, rn, imm)
    if (!a->sf) tcg_gen_ext32u_i64(tcg_rd, tcg_rd);  // W 寄存器零扩展
    return true;
}
```

**生成的 IR** (以 `ADD X0, X1, #0x10` 为例)：
```
add_i64  X0, X1, $0x10
```

### 6.2 ADDS_i (带标志加法)

```c
// translate-a64.c:4996
TRANS(ADDS_i, gen_rri, a, 0, 1, a->sf ? gen_add64_CC : gen_add32_CC)

// gen_add64_CC (L1013-1033):
static void gen_add64_CC(TCGv_i64 dest, TCGv_i64 t0, TCGv_i64 t1) {
    result = tcg_temp_new_i64();
    flag = tcg_temp_new_i64();
    tmp = tcg_temp_new_i64();

    tcg_gen_movi_i64(tmp, 0);
    tcg_gen_add2_i64(result, flag, t0, tmp, t1, tmp);  // 128-bit 加法获取进位
    tcg_gen_extrl_i64_i32(cpu_CF, flag);                // CF = carry out

    gen_set_NZ64(result);                               // NF/ZF 设置

    tcg_gen_xor_i64(flag, result, t0);                  // V = (res^a) & ~(a^b)
    tcg_gen_xor_i64(tmp, t0, t1);
    tcg_gen_andc_i64(flag, flag, tmp);
    tcg_gen_extrh_i64_i32(cpu_VF, flag);                // 取高 32 位作为 VF

    tcg_gen_mov_i64(dest, result);
}
```

**生成的 IR** (以 `ADDS X0, X1, #1` 为例)：
```
movi_i64   tmp3, $0x0
add2_i64   tmp1, tmp2, X1, tmp3, $0x1, tmp3   // (result, carry) = X1 + 1
extrl_i64_i32  CF, tmp2                         // CF = carry
[NZ 设置 IR...]
xor_i64    tmp2, tmp1, X1                       // 溢出计算
xor_i64    tmp3, X1, $0x1
andc_i64   tmp2, tmp2, tmp3
extrh_i64_i32  VF, tmp2                         // VF = overflow
mov_i64    X0, tmp1                             // X0 = result
```

### 6.3 LDR (Guest 内存加载)

```c
// translate-a64.c:3988-4005
static bool trans_LDR(DisasContext *s, arg_ldst *a) {
    TCGv_i64 addr = tcg_temp_new_i64();
    // 计算地址: base + offset
    tcg_gen_add_i64(addr, cpu_reg_sp(s, a->rn), cpu_reg(s, a->rm));
    // 生成 qemu_ld
    tcg_gen_qemu_ld_i64(cpu_reg(s, a->rt), addr, get_mem_index(s), mop);
}
```

**生成的 IR**:
```
add_i64    tmp1, SP, X2           // addr = SP + X2
qemu_ld_i64  X0, tmp1, $oi       // X0 = MEM[addr] (含 TLB 查找)
```

---

## 7. tcg_optimize 优化 Pass

### 7.1 OptContext 与 TempOptInfo

```c
// optimize.c:50-61
typedef struct OptContext {
    TCGContext *tcg;
    TCGOp *prev_mb;           // 上一个内存屏障
    TCGTempSet temps_used;    // 本 pass 已处理的 temp
    IntervalTreeRoot mem_copy; // 内存拷贝追踪 (区间树)
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_free;
    TCGType type;             // 当前 op 类型
    int carry_state;          // 进位常量状态 (-1/0/1)
} OptContext;

// optimize.c:41-48
typedef struct TempOptInfo {
    TCGTemp *prev_copy, *next_copy;  // 拷贝环形链表
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_copy;
    uint64_t z_mask;   // 位=0 → 值一定为 0
    uint64_t o_mask;   // 位=1 → 值一定为 1
    uint64_t s_mask;   // 位=1 → 值一定与 MSB 相同 (符号扩展追踪)
} TempOptInfo;
```

**掩码语义**：
- `z_mask`: 零掩码。若 `ti->z_mask == ti->o_mask`，则 temp 为常量
- `o_mask`: 一掩码。`ti_const_val(ti) = ti->z_mask` (当为常量时)
- `s_mask`: 符号掩码。追踪符号扩展模式

### 7.2 主循环结构 (optimize.c:3000-3244)

```c
void tcg_optimize(TCGContext *s) {
    OptContext ctx = { .tcg = s };
    QTAILQ_FOREACH_SAFE(op, &s->ops, link, op_next) {
        // 1. call 特殊处理
        if (opc == INDEX_op_call) { fold_call(&ctx, op); continue; }
        // 2. 初始化参数 TempOptInfo
        init_arguments(&ctx, op, ...);
        // 3. 拷贝传播: 将输入参数替换为拷贝链中的更佳选择
        copy_propagate(&ctx, op, ...);
        // 4. 按操作码分派到 fold_* 函数
        switch (opc) {
            case INDEX_op_add: done = fold_add(&ctx, op); break;
            case INDEX_op_and: done = fold_and(&ctx, op); break;
            // ... 60+ 种操作码
        }
    }
}
```

### 7.3 常量折叠 (Constant Folding)

```c
// optimize.c:427-599
static uint64_t do_constant_folding_2(TCGOpcode op, TCGType type,
                                      uint64_t x, uint64_t y) {
    switch (op) {
    case INDEX_op_add: return x + y;
    case INDEX_op_sub: return x - y;
    case INDEX_op_mul: return x * y;
    case INDEX_op_and: return x & y;
    case INDEX_op_shl:
        return (type == TCG_TYPE_I32) ? (uint32_t)x << (y & 31)
                                      : (uint64_t)x << (y & 63);
    // ... 30+ 种操作码
    }
}
```

当两个输入操作数都是常量（`z_mask == o_mask`）时，直接计算结果并替换为 `mov const`。

### 7.4 拷贝传播 (Copy Propagation)

```c
// copy_propagate: 遍历输入参数
// 对每个输入 temp，在拷贝链中查找 "更好的" 版本
// "更好" 的定义: TEMP_CONST > TEMP_FIXED > TEMP_GLOBAL > TEMP_TB > TEMP_EBB
// (cmp_better_copy, L120-123)
```

拷贝链通过 `TempOptInfo.prev_copy/next_copy` 维护为双向循环链表。
当 `tcg_opt_gen_mov` 将 dst 设为 src 的拷贝时，将 dst 加入 src 的拷贝链。

### 7.5 掩码追踪 (Mask Tracking)

每个 fold_* 函数在执行后更新输出 temp 的掩码：

```c
// fold_and 示例:
static bool fold_and(OptContext *ctx, TCGOp *op) {
    // ... 常量/恒等/零值简化 ...
    t1 = arg_info(op->args[1]);
    t2 = arg_info(op->args[2]);
    z_mask = t1->z_mask & t2->z_mask;        // AND: 零位取并集
    o_mask = t1->o_mask & t2->o_mask;        // AND: 一位取交集
    s_mask = t1->s_mask & t2->s_mask;        // 符号位取交集
    return fold_masks_zos(ctx, op, z_mask, o_mask, s_mask);
}

// fold_xor 示例 (optimize.c:2977-2997):
z_mask = (t1->z_mask | t2->z_mask) & ~(t1->o_mask & t2->o_mask);
o_mask = (t1->o_mask & ~t2->z_mask) | (t2->o_mask & ~t1->z_mask);
```

### 7.6 内存拷贝追踪 (Memory Copy Tracking)

追踪 `ld/st` 对 env 结构体的访问，消除冗余加载：

```c
// fold_tcg_ld_memcopy (optimize.c:2893-2912):
// 如果目标 offset 有已知的 temp 拷贝 → 直接用 mov 替代 ld
src = find_mem_copy_for(ctx, type, ofs);
if (src && src->base_type == type) {
    return tcg_opt_gen_mov(ctx, op, temp_arg(dst), temp_arg(src));
}

// fold_tcg_st_memcopy (optimize.c:2945-2975):
// 如果存储与已有内存拷贝的值相同 → 删除冗余 st
if (ts_is_const(src)) {
    TCGTemp *prev = find_mem_copy_for(ctx, type, ofs);
    if (src == prev) {
        tcg_op_remove(ctx->tcg, op);
        return true;
    }
}
```

使用 `IntervalTree` 按偏移范围追踪内存拷贝状态。

### 7.7 条件折叠 (Condition Folding)

```c
// optimize.c:698-729
static int do_constant_folding_cond(TCGType type, TCGArg x, TCGArg y, TCGCond c)
{
    // 情况1: 两操作数都是常量 → 直接求值
    if (arg_is_const(x) && arg_is_const(y)) { ... }
    // 情况2: 两操作数相同 (拷贝) → 等价判断
    else if (args_are_copies(x, y)) { return do_constant_folding_cond_eq(c); }
    // 情况3: y==0 的特殊优化
    else if (arg_is_const_val(y, 0)) {
        switch (c) {
        case TCG_COND_LTU: return 0;   // x < 0 永假
        case TCG_COND_GEU: return 1;   // x >= 0 永真
        }
    }
    return -1;  // 不可折叠
}
```

### 7.8 EBB 边界处理

```c
// optimize.c:3230-3237
case INDEX_op_set_label:
case INDEX_op_br:
case INDEX_op_exit_tb:
case INDEX_op_goto_tb:
case INDEX_op_goto_ptr:
    finish_ebb(&ctx);  // 清除所有非常量优化信息
    done = true;
    break;
```

EBB 边界处 `finish_ebb` 重置所有 temp 的掩码和拷贝链，因为跳转目标处不能假设任何先前分析结果仍有效。

---

## 8. 活性分析

### 8.1 reachable_code_pass (tcg.c:3565-3653)

正向扫描，删除不可达代码：
- 无条件分支/exit_tb/goto_ptr 后的代码标记为 dead
- noreturn helper 调用后的代码标记为 dead
- 未被引用的 label 删除
- 分支到紧接 label 的 br 删除

### 8.2 liveness_pass_0 (tcg.c:3804-3866)

将仅在单个 EBB 中使用的 TEMP_TB 降级为 TEMP_EBB，减少后续分析开销。

### 8.3 liveness_pass_1 (tcg.c:3883-4265)

**反向扫描** op 链表，为每个 op 的参数标记 DEAD_ARG 和 SYNC_ARG：

```
活性状态: TS_DEAD (bit 0) | TS_MEM (bit 1)
- TEMP 初始: GLOBAL/TB → TS_DEAD|TS_MEM, EBB/CONST → TS_DEAD
- 输出参数: 如果 temp 已 dead → 标记 DEAD_ARG; 然后标记为 dead
- 输入参数: 如果 temp 是 dead → 标记 DEAD_ARG; 然后标记为 live
- 纯函数（NO_SIDE_EFFECTS）: 如果所有输出 dead → 删除整个 op
```

### 8.4 liveness_pass_2 (tcg.c:4270+)

将 indirect temp（通过指针访问的 temp）替换为 direct temp（直接在栈帧分配），
减少间接寻址开销。如果有修改，重新运行 liveness_pass_1。

---

## 9. tcg_gen_code 后端流水线

### 9.1 完整流程 (tcg.c:6556-6767)

```c
int tcg_gen_code(TCGContext *s, TranslationBlock *tb, uint64_t pc_start) {
    // 1. [调试] 打印未优化 IR (CPU_LOG_TB_OP)
    // 2. 优化 + 活性分析
    tcg_optimize(s);                    // L6592
    reachable_code_pass(s);             // L6594
    liveness_pass_0(s);                 // L6595
    liveness_pass_1(s);                 // L6596
    [liveness_pass_2 + 重做 pass_1]    // L6598-6614

    // 3. [调试] 打印优化后 IR (CPU_LOG_TB_OP_OPT)

    // 4. 初始化寄存器分配
    tcg_reg_alloc_start(s);             // L6634

    // 5. 遍历 op 链表，逐 op 分配寄存器 + 发射代码
    QTAILQ_FOREACH(op, &s->ops, link) {
        switch (opc) {
        case INDEX_op_mov:      tcg_reg_alloc_mov(s, op);  break;
        case INDEX_op_call:     tcg_reg_alloc_call(s, op); break;
        case INDEX_op_insn_start: [记录 guest→host 偏移映射]; break;
        case INDEX_op_exit_tb:  tcg_out_exit_tb(s, args[0]); break;
        case INDEX_op_goto_tb:  tcg_out_goto_tb(s, args[0]); break;
        default:                tcg_reg_alloc_op(s, op);   break;
        }
        // 溢出检查: code_ptr > code_gen_highwater → return -1
    }

    // 6. 慢路径代码发射 + 常量池 + 重定位
    tcg_out_ldst_finalize(s);           // L6747
    tcg_out_pool_finalize(s);           // L6751
    tcg_resolve_relocs(s);              // L6755

    // 7. 刷新 I-cache
    flush_idcache_range(...);           // L6761
    return tcg_current_code_size(s);
}
```

### 9.2 寄存器分配核心

```c
// tcg_reg_alloc (tcg.c:4645-4705):
// 1. 在偏好寄存器集中找空闲寄存器
// 2. 在必需寄存器集中找空闲寄存器
// 3. 都没有 → 溢出一个到栈帧 (temp_sync → tcg_out_st)

// tcg_reg_alloc_op (tcg.c:5133-5468):
// 对每个 op:
// 1. 为输入参数分配寄存器，满足约束 (reg/imm/mem)
// 2. 如果输入 temp 在此 op 后死亡 → 复用其寄存器给输出
// 3. 为输出参数分配寄存器
// 4. 调用 tcg_out_op / tcg_out_vec_op 发射 host 指令
```

### 9.3 溢出与同步

```c
// tcg.c:4586-4624 (temp_sync):
// 如果 temp 不在内存中 → 生成 store 到栈帧
if (!ts->mem_coherent) {
    tcg_out_st(s, ts->type, ts->reg, ts->mem_base->reg, ts->mem_offset);
    ts->mem_coherent = 1;
}
```

---

## 10. Guest 内存访问 IR 生成

### 10.1 qemu_ld/st 操作码

```c
// tcg-op-ldst.c:241-284 (tcg_gen_qemu_ld_i32_int):
void tcg_gen_qemu_ld_i32_int(TCGv_i32 val, TCGTemp *addr, TCGArg idx, MemOp memop) {
    // 1. 内存序要求
    tcg_gen_req_mo(TCG_MO_LD_LD | TCG_MO_ST_LD);
    // 2. 规范化 memop (对齐、字节序、符号)
    memop = tcg_canonicalize_memop(memop, 0, 0);
    // 3. 字节序处理: 如果后端不支持 bswap → 拆分为 ld + bswap op
    if ((memop & MO_BSWAP) && !tcg_target_has_memory_bswap(memop)) { ... }
    // 4. 发射 qemu_ld op
    gen_ldst1(INDEX_op_qemu_ld, TCG_TYPE_I32, val_temp, addr, oi);
    // 5. 如果需要软件 bswap
    if (bswap_needed) { tcg_gen_bswap16/32_i32(val, val, ...); }
    // 6. Plugin 回调
    plugin_gen_mem_callbacks_i32(val, ...);
}
```

### 10.2 MemOpIdx 编码

```c
MemOpIdx = make_memop_idx(memop, mmu_idx)
// memop: 访问属性 (size | sign | endian | align | atom)
// mmu_idx: TLB 索引 (区分 EL0/EL1/S/NS 等地址空间)
```

### 10.3 串行模式原子性降级

```c
// tcg-op-ldst.c:83-88 (tcg_canonicalize_memop):
// 在非并行模式 (单核) 下，移除原子性要求以简化生成代码
if (!(tcg_ctx->gen_tb->cflags & CF_PARALLEL)) {
    op &= ~MO_ATOM_MASK;
    op |= MO_ATOM_NONE;
}
```

### 10.4 后端展开 (快慢路径)

后端收到 `qemu_ld/qemu_st` 后展开为：
1. **快路径** (内联 TLB 查找): 比较 guest 页号 → 命中 → 计算 host 地址 → load/store
2. **慢路径** (out-of-line): TLB miss → 调用 `helper_ldXX_mmu` → refill → 重试

---

## 11. 与硬件实际执行的对比

### 11.1 指令语义忠实度

| 方面 | QEMU TCG 实现 | 硬件行为 | 差异 |
|------|--------------|---------|------|
| **算术运算** | 使用 host 原生运算 | 硬件 ALU 执行 | 语义无差异 |
| **标志计算** | 每次 ADDS/SUBS 显式计算 NF/ZF/CF/VF | 硬件隐式并行设置 | TCG 需多条 IR (约 8-10 条) |
| **内存序** | `tcg_gen_mb` → host fence | ARM DMB/DSB | 正确但可能过保守 |
| **条件执行** | brcond + label | 条件码直接驱动管线 | TCG 转为控制流 |
| **TB 粒度** | 最多 512 条 guest 指令/TB | 硬件连续执行 | TB 边界引入退出开销 |
| **进位链** | addco/addci 隐式进位 | ADCS 硬件进位标志 | 语义等价 |

### 11.2 优化器与硬件微架构对比

| TCG 优化 | 硬件等价机制 | 说明 |
|----------|-------------|------|
| 常量折叠 | JIT 独有 | 利用翻译时可知的常量 |
| 拷贝传播 | 寄存器重命名 (RAT) | TCG 静态；硬件动态 |
| 死代码消除 | 投机执行取消 | TCG 确定性；硬件投机 |
| 掩码追踪 | 无直接等价 | TCG 独特优势：精确位追踪 |
| 内存拷贝追踪 | Store-to-Load Forwarding | TCG 跨 op；硬件在 LSQ 中 |
| EBB 边界重置 | 分支预测器 | TCG 保守；硬件投机 |

### 11.3 性能模型

```
首次执行一条 guest 指令的开销:
  翻译 (解码+IR生成+优化+后端) ≈ 数千 host 周期

后续命中 TB cache 的开销:
  直接链接 (goto_tb) ≈ 1 条 host 跳转指令
  间接链接 (goto_ptr) ≈ hash 查找 ≈ 10-20 host 指令
  
单条 guest 指令的翻译膨胀率:
  简单 ALU (ADD/AND/...) → 1-2 条 host 指令 (≈1:1)
  带标志 (ADDS/SUBS)    → 8-12 条 host 指令 (标志计算开销)
  内存访问 (LDR/STR)    → 5-10 条 host 指令 (TLB 快路径)
  系统指令 (MSR/MRS)    → helper 调用 ≈ 20+ 条 host 指令
```

---

## 关键源文件索引

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `include/tcg/tcg-opc.h` | 29-185 | 全部操作码定义 |
| `include/tcg/tcg.h` | 59-64,128-329,346+ | TCGOpcode/Type/Temp/Op/Context |
| `tcg/tcg-op.c` | 40-2570 | 前端 IR 生成 API 实现 |
| `tcg/tcg-op-ldst.c` | 37-1348 | Guest 内存访问 IR 生成 |
| `tcg/optimize.c` | 34-3244 | tcg_optimize 优化 pass |
| `tcg/tcg.c:3565-3653` | — | reachable_code_pass |
| `tcg/tcg.c:3804-3866` | — | liveness_pass_0 |
| `tcg/tcg.c:3883-4265` | — | liveness_pass_1 |
| `tcg/tcg.c:4270+` | — | liveness_pass_2 |
| `tcg/tcg.c:4645-4705` | — | tcg_reg_alloc (单寄存器) |
| `tcg/tcg.c:5133-5468` | — | tcg_reg_alloc_op (通用) |
| `tcg/tcg.c:6556-6767` | — | tcg_gen_code 主流程 |
| `target/arm/tcg/translate-a64.c` | 4957-4997 | ARM64 算术指令翻译 |
| `target/arm/tcg/translate-a64.c` | 1013-1080 | 标志计算 IR 生成 |
