# TCG 优化递次深度分析

> 基于 QEMU 11.0.50 源码，深入分析 TCG（Tiny Code Generator）的优化管线：
> 常量折叠、拷贝传播、掩码追踪、死代码消除、活跃性分析与寄存器分配。

---

## 目录

1. [优化管线总览](#1-优化管线总览)
2. [核心数据结构](#2-核心数据结构)
3. [TempOptInfo — 临时变量优化信息](#3-tempoptinfo--临时变量优化信息)
4. [OptContext — 优化上下文](#4-optcontext--优化上下文)
5. [tcg_optimize 主循环](#5-tcg_optimize-主循环)
6. [常量折叠框架](#6-常量折叠框架)
7. [拷贝传播](#7-拷贝传播)
8. [z_mask / o_mask / s_mask 掩码追踪](#8-zmask--omask--smask-掩码追踪)
9. [fold_masks_zosa_int — 掩码传播核心](#9-fold_masks_zosa_int--掩码传播核心)
10. [算术运算优化](#10-算术运算优化)
11. [逻辑运算优化](#11-逻辑运算优化)
12. [分支条件优化](#12-分支条件优化)
13. [扩展与位域优化](#13-扩展与位域优化)
14. [内存操作优化](#14-内存操作优化)
15. [内存屏障优化](#15-内存屏障优化)
16. [进位/借位链优化](#16-进位借位链优化)
17. [全部 fold 函数索引](#17-全部-fold-函数索引)
18. [可达代码分析](#18-可达代码分析)
19. [活跃性分析 Pass 0](#19-活跃性分析-pass-0)
20. [活跃性分析 Pass 1](#20-活跃性分析-pass-1)
21. [活跃性分析 Pass 2](#21-活跃性分析-pass-2)
22. [活跃性数据编码](#22-活跃性数据编码)
23. [寄存器分配框架](#23-寄存器分配框架)
24. [约束系统](#24-约束系统)
25. [Spill/Reload 机制](#25-spillreload-机制)
26. [ARM64 延迟标志优化](#26-arm64-延迟标志优化)
27. [优化效果观察](#27-优化效果观察)
28. [完整优化管线追踪示例](#28-完整优化管线追踪示例)
29. [关键源文件索引](#29-关键源文件索引)

---

## 1. 优化管线总览

TCG 的优化管线在 `tcg_gen_code()` 中按严格顺序执行：

```c
// tcg/tcg.c:6556 — tcg_gen_code() 优化管线
int tcg_gen_code(TCGContext *s, TranslationBlock *tb, uint64_t pc_start)
{
    // [LOG] 输出优化前的 IR（CPU_LOG_TB_OP）

    tcg_optimize(s);              // ① 常量折叠 + 拷贝传播 + 掩码分析
                                  //    tcg/optimize.c:2999

    reachable_code_pass(s);       // ② 不可达代码消除
                                  //    tcg/tcg.c:3565

    liveness_pass_0(s);           // ③ 生命周期降级（TB→EBB）
                                  //    tcg/tcg.c:3804

    liveness_pass_1(s);           // ④ 反向活跃性分析 + 死代码删除
                                  //    tcg/tcg.c:3883

    if (liveness_pass_2(s)) {     // ⑤ 间接临时变量→直接临时变量
        liveness_pass_1(s);       //    如有替换，重跑 Pass 1
    }                             //    tcg/tcg.c:4270

    // [LOG] 输出优化后的 IR（CPU_LOG_TB_OP_OPT）

    // ⑥ 寄存器分配 + 代码生成（在同一遍中完成）
    // tcg_reg_alloc_* 系列函数
}
```

```
IR 生成 → ① tcg_optimize → ② reachable_code → ③ pass_0 → ④ pass_1
                                                                ↓
                                                          ⑤ pass_2
                                                           (替换?)
                                                              ↓ 是
                                                          ④ pass_1 (再次)
                                                              ↓
                                                     ⑥ 寄存器分配 + 代码生成
```

---

## 2. 核心数据结构

优化器使用两个核心数据结构追踪临时变量的状态：

```
OptContext (全局优化上下文)
    │
    ├── tcg: TCGContext*        — TCG 编译上下文
    ├── prev_mb: TCGOp*         — 上一个内存屏障（用于合并）
    ├── carry_state: int        — 常量进位状态
    ├── mem_copy: IntervalTree  — 内存拷贝追踪
    └── temps_used: TCGTempSet  — 已初始化的临时变量集合

TempOptInfo (每个临时变量)
    │
    ├── z_mask: uint64_t  — 已知为零的位（0=确定为0）
    ├── o_mask: uint64_t  — 已知为一的位（1=确定为1）
    ├── s_mask: int64_t   — 符号重复掩码
    ├── prev_copy / next_copy  — 拷贝环形链表
    └── mem_copy: MemCopyInfo   — 内存拷贝列表
```

---

## 3. TempOptInfo — 临时变量优化信息

```c
// tcg/optimize.c:41-48
typedef struct TempOptInfo {
    TCGTemp *prev_copy;   // 拷贝链：前驱
    TCGTemp *next_copy;   // 拷贝链：后继
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_copy;  // 内存拷贝列表
    uint64_t z_mask;      // 已知零位掩码：bit=0 当且仅当 value bit=0
    uint64_t o_mask;      // 已知一位掩码：bit=1 当且仅当 value bit=1
    uint64_t s_mask;      // 符号重复掩码：bit=1 表示该位与 MSB 相同
} TempOptInfo;
```

### 掩码含义

| 掩码 | 含义 | 示例 |
|------|------|------|
| `z_mask` | 值可能为 1 的位集合 | `z_mask = 0xFF` → 值在 0x00-0xFF |
| `o_mask` | 值确定为 1 的位集合 | `o_mask = 0x80` → bit7 确定为 1 |
| `s_mask` | 与 MSB 相同的位集合 | `s_mask = 0xFFFF_FFFF_0000_0000` → 高32位=符号扩展 |

**关键不变量**: `o_mask & ~z_mask == 0`（已知为 1 的位必须在可能为 1 的集合内）

**常量判定**: 当 `z_mask == o_mask` 时，所有位的值都已确定 → 该临时变量是常量。

```c
// tcg/optimize.c:73-88
static inline bool ti_is_const(TempOptInfo *ti)
{
    return ti->z_mask == ti->o_mask;
}

static inline uint64_t ti_const_val(TempOptInfo *ti)
{
    return ti->o_mask;
}
```

---

## 4. OptContext — 优化上下文

```c
// tcg/optimize.c:50-61
typedef struct OptContext {
    TCGContext *tcg;           // TCG 编译上下文
    TCGOp *prev_mb;           // 上一个内存屏障 op（用于合并/消除）
    TCGTempSet temps_used;    // 已初始化的临时变量位图

    IntervalTreeRoot mem_copy;         // 内存拷贝区间树
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_free;  // 空闲内存拷贝节点

    TCGType type;             // 当前 op 的类型（I32/I64/V64/...）
    int carry_state;          // 进位状态: -1=非常量, 0/1=常量进位
} OptContext;
```

关键设计：
- **`carry_state`** — 用于多精度加减法优化（addci/addco/subbi/subbo），追踪链式进位是否为常量
- **`mem_copy`** — 用于追踪 TCG 加载/存储的值传播，实现"加载已知值"优化
- **`prev_mb`** — 追踪前一个内存屏障，用于合并连续屏障

---

## 5. tcg_optimize 主循环

```c
// tcg/optimize.c:2999-3244
void tcg_optimize(TCGContext *s)
{
    int nb_temps, i;
    TCGOp *op, *op_next;
    OptContext ctx = { .tcg = s };

    // 初始化所有已存在的临时变量
    nb_temps = s->nb_temps;
    for (i = 0; i < nb_temps; i++) {
        // 全局变量: z_mask/o_mask = 全满, s_mask = 0
        // 常量: z_mask = o_mask = 常量值
        init_ts_info(&ctx, &s->temps[i]);
    }

    // 遍历所有 IR 操作
    QTAILQ_FOREACH_SAFE(op, &s->ops, link, op_next) {
        TCGOpcode opc = op->opc;

        // 1. 初始化新创建的临时变量
        init_arguments(&ctx, op, ...);

        // 2. 拷贝传播：将输入参数替换为更优的拷贝
        copy_propagate(&ctx, op, ...);

        // 3. 设置当前操作类型
        ctx.type = ...;

        // 4. 分发到对应的 fold 函数
        switch (opc) {
        case INDEX_op_add:     fold_add(&ctx, op);     break;
        case INDEX_op_sub:     fold_sub(&ctx, op);     break;
        case INDEX_op_and:     fold_and(&ctx, op);     break;
        case INDEX_op_or:      fold_or(&ctx, op);      break;
        case INDEX_op_xor:     fold_xor(&ctx, op);     break;
        case INDEX_op_brcond:  fold_brcond(&ctx, op);  break;
        case INDEX_op_mov:     fold_mov(&ctx, op);     break;
        // ... 76 个 fold 函数
        }
    }
}
```

每个 fold 函数的标准模式：
1. 尝试常量折叠 → 如果成功，替换为 `movi` 或删除
2. 尝试代数简化 → 如 `x + 0 → x`, `x & 0 → 0`
3. 更新输出的 z_mask/o_mask/s_mask
4. 调用 `finish_folding()` 清理

---

## 6. 常量折叠框架

### 6.1 常量折叠入口

```c
// tcg/optimize.c:427-589 — do_constant_folding_2()
// 对两个常量操作数执行编译时计算
static uint64_t do_constant_folding_2(TCGOpcode op, uint64_t x, uint64_t y)
{
    switch (op) {
    case INDEX_op_add:  return x + y;
    case INDEX_op_sub:  return x - y;
    case INDEX_op_mul:  return x * y;
    case INDEX_op_and:  return x & y;
    case INDEX_op_or:   return x | y;
    case INDEX_op_xor:  return x ^ y;
    case INDEX_op_shl:  return x << (y & 63);
    case INDEX_op_shr:  return x >> (y & 63);
    case INDEX_op_sar:  return (int64_t)x >> (y & 63);
    case INDEX_op_rotr: return ror64(x, y & 63);
    case INDEX_op_rotl: return rol64(x, y & 63);
    // ... 更多操作码
    }
}

// tcg/optimize.c:591-599 — do_constant_folding()
// 包装函数：处理 32 位类型截断
static uint64_t do_constant_folding(TCGOpcode op, TCGType type,
                                    uint64_t x, uint64_t y)
{
    uint64_t res = do_constant_folding_2(op, x, y);
    if (type == TCG_TYPE_I32) res = (int32_t)res;  // 32 位符号扩展
    return res;
}
```

### 6.2 fold_const1 / fold_const2 — 通用折叠辅助

```c
// tcg/optimize.c:888-908
static bool fold_const1(OptContext *ctx, TCGOp *op)
{
    // 如果唯一输入是常量 → 计算结果并替换为 movi
    if (arg_is_const(op->args[1])) {
        uint64_t t = arg_info(op->args[1])->o_mask;
        t = do_constant_folding(op->opc, ctx->type, t, 0);
        return tcg_opt_gen_movi(ctx, op, op->args[0], t);
    }
    return false;
}

static bool fold_const2(OptContext *ctx, TCGOp *op)
{
    // 如果两个输入都是常量 → 计算结果并替换为 movi
    if (arg_is_const(op->args[1]) && arg_is_const(op->args[2])) {
        uint64_t t1 = arg_info(op->args[1])->o_mask;
        uint64_t t2 = arg_info(op->args[2])->o_mask;
        t1 = do_constant_folding(op->opc, ctx->type, t1, t2);
        return tcg_opt_gen_movi(ctx, op, op->args[0], t1);
    }
    return false;
}
```

### 6.3 fold_const2_commutative — 交换律 + 折叠

```c
// tcg/optimize.c:911-917
static bool fold_commutative(OptContext *ctx, TCGOp *op)
{
    // 规范化：将常量操作数放到 args[2]
    swap_commutative(op->args[0], &op->args[1], &op->args[2]);
    return false;
}

static bool fold_const2_commutative(OptContext *ctx, TCGOp *op)
{
    return fold_const2(ctx, op) || fold_commutative(ctx, op);
}
```

---

## 7. 拷贝传播

### 7.1 拷贝链追踪

拷贝关系通过 `TempOptInfo` 中的 `prev_copy/next_copy` 环形链表追踪：

```c
// tcg/optimize.c:125-154 — init_ts_info()
// 初始化时，每个 temp 指向自身（无拷贝）

// tcg/optimize.c:357-417 — tcg_opt_gen_mov()
// 当生成 mov 操作时，将 src 和 dst 加入同一拷贝链
static bool tcg_opt_gen_mov(OptContext *ctx, TCGOp *op,
                            TCGArg dst, TCGArg src)
{
    // 如果 src == dst，直接删除 op
    if (ts_are_copies(dst_ts, src_ts)) {
        tcg_op_remove(ctx->tcg, op);
        return true;
    }
    // 将 dst_ts 加入 src_ts 的拷贝环
    // 拷贝 z_mask/o_mask/s_mask
}
```

### 7.2 拷贝替换

```c
// tcg/optimize.c:195-208 — find_better_copy()
// 在拷贝链中选择"最优"的临时变量
static TCGTemp *find_better_copy(TCGTemp *ts)
{
    // 优先级：全局变量 > 局部临时变量
    // 同类中选择索引最小的
}

// tcg/optimize.c:839-848 — copy_propagate()
// 在主循环中，将每个输入参数替换为拷贝链中最优的表示
static void copy_propagate(OptContext *ctx, TCGOp *op, ...)
{
    for (i = 0; i < nb_iargs; i++) {
        TCGTemp *ts = arg_temp(op->args[nb_oargs + i]);
        if (ts_is_copy(ts)) {
            op->args[nb_oargs + i] = temp_arg(find_better_copy(ts));
        }
    }
}
```

### 7.3 拷贝传播示例

```
优化前:                      优化后:
  mov  t1, t0               (删除 — t1 加入 t0 的拷贝链)
  add  t2, t1, #5           add  t2, t0, #5  (t1 替换为 t0)
  str  t1, [addr]           str  t0, [addr]   (t1 替换为 t0)
```

---

## 8. z_mask / o_mask / s_mask 掩码追踪

掩码追踪是 TCG 优化器最精妙的机制，通过跟踪每个临时变量的**位级别信息**实现高级优化。

### 8.1 三种掩码

| 掩码 | 全称 | 语义 | 用途 |
|------|------|------|------|
| `z_mask` | zero mask | bit=0 ↔ 值bit确定为0; bit=1 ↔ 值bit可能为0或1 | 上界：值 ≤ z_mask |
| `o_mask` | one mask | bit=1 ↔ 值bit确定为1; bit=0 ↔ 值bit可能为0或1 | 下界：值 ≥ o_mask |
| `s_mask` | sign mask | bit=1 ↔ 该位与 MSB 值相同 | 符号扩展追踪 |

### 8.2 掩码传播规则

| 操作 | z_mask 规则 | o_mask 规则 | s_mask 规则 |
|------|------------|------------|------------|
| AND | `z1 & z2` | `o1 & o2` | `s1 & s2` |
| OR | `z1 \| z2` | `o1 \| o2` | `s1 & s2` |
| XOR | `z1 \| z2` | `~z1 & ~z2` (交叉) | `s1 & s2` |
| SHL n | `z << n` | `o << n` | `s << n` |
| SHR n | `z >> n` | `o >> n` | 0 (逻辑右移) |
| SAR n | `z >> n` | `o >> n` | `s >> n` (算术保留) |
| EXTU 8 | `z & 0xFF` | `o & 0xFF` | `INT64_MIN >> 55` |
| EXTS 8 | 符号扩展 z | 符号扩展 o | `INT64_MIN >> 55` |

### 8.3 常量自动检测

当经过一系列操作后 `z_mask == o_mask` 时，说明所有位都已确定 → 自动折叠为常量：

```
例: 假设 t0 的 z_mask = 0xFF00, o_mask = 0x0000
    执行 AND t1, t0, #0xFF00
    → t1.z_mask = 0xFF00 & 0xFF00 = 0xFF00
    → t1.o_mask = 0x0000 & 0xFF00 = 0x0000
    （不是常量，值在 0x0000-0xFF00 范围内）

    执行 AND t2, t0, #0x8000
    → t2.z_mask = 0xFF00 & 0x8000 = 0x8000
    → t2.o_mask = 0x0000 & 0x8000 = 0x0000
    （不是常量，值为 0x0000 或 0x8000）

    如果 t0.o_mask = 0x8000:
    → t2.o_mask = 0x8000 & 0x8000 = 0x8000
    → z_mask == o_mask == 0x8000 → 常量 0x8000!
```

---

## 9. fold_masks_zosa_int — 掩码传播核心

这是优化器中最重要的函数，几乎所有 fold 函数最终都通过它更新输出掩码：

```c
// tcg/optimize.c:929-983
static bool fold_masks_zosa_int(OptContext *ctx, TCGOp *op,
                                uint64_t z_mask, uint64_t o_mask,
                                int64_t s_mask, uint64_t a_mask)
{
    // 32 位类型处理：符号扩展
    if (ctx->type == TCG_TYPE_I32) {
        z_mask = (int32_t)z_mask;
        o_mask = (int32_t)o_mask;
        s_mask |= INT32_MIN;
        a_mask = (uint32_t)a_mask;
    }

    // 不变量检查：已知为 1 的位必须在可能为 1 的集合内
    tcg_debug_assert((o_mask & ~z_mask) == 0);

    // ★ 所有位确定 → 折叠为常量
    if (z_mask == o_mask) {
        return tcg_opt_gen_movi(ctx, op, op->args[0], o_mask);
    }

    // ★ 无位受影响 → 折叠为拷贝
    if (a_mask == 0) {
        return tcg_opt_gen_mov(ctx, op, op->args[0], op->args[1]);
    }

    // 更新输出临时变量的掩码
    ts = arg_temp(op->args[0]);
    reset_ts(ctx, ts);
    ti = ts_info(ts);
    ti->z_mask = z_mask;
    ti->o_mask = o_mask;

    // 规范化 s_mask：合并 z/o 信息
    rep = clz64(~s_mask);
    rep = MAX(rep, clz64(z_mask));     // 高位全零 → 符号位
    rep = MAX(rep, clz64(~o_mask));    // 高位全一 → 符号位
    rep = MAX(rep - 1, 0);
    ti->s_mask = INT64_MIN >> rep;

    return false;  // 未完全折叠，保留操作
}
```

### 便捷包装函数

```c
// tcg/optimize.c:985-1018
fold_masks_zosa(ctx, op, z, o, s)  // a_mask = z ^ o（自动计算）
fold_masks_zos(ctx, op, z, o, s)   // 无 a_mask
fold_masks_zo(ctx, op, z, o)       // 无 s_mask
fold_masks_zs(ctx, op, z, s)       // 无 o_mask
fold_masks_z(ctx, op, z)           // 仅 z_mask
fold_masks_s(ctx, op, s)           // 仅 s_mask
```

---

## 10. 算术运算优化

### 10.1 fold_add — 加法

```c
// tcg/optimize.c:1132-1139
static bool fold_add(OptContext *ctx, TCGOp *op)
{
    if (fold_const2_commutative(ctx, op) ||  // 两个常量 → 计算
        fold_xi_to_x(ctx, op, 0)) {          // x + 0 → x
        return true;
    }
    return finish_folding(ctx, op);
}
```

### 10.2 fold_sub — 减法

```c
// tcg/optimize.c:2703-2719
static bool fold_sub(OptContext *ctx, TCGOp *op)
{
    if (fold_const2(ctx, op) ||              // 两个常量 → 计算
        fold_xx_to_i(ctx, op, 0) ||          // x - x → 0
        fold_xi_to_x(ctx, op, 0) ||          // x - 0 → x
        fold_sub_to_neg(ctx, op)) {           // 0 - x → neg(x)
        return true;
    }
    // 常量右操作数转换为加法: x - C → x + (-C)
    // 有利于后续 add 的交换律优化
    return finish_folding(ctx, op);
}
```

### 10.3 fold_mul — 乘法

```c
// tcg/optimize.c:2153-2162
static bool fold_mul(OptContext *ctx, TCGOp *op)
{
    if (fold_const2_commutative(ctx, op) ||  // 常量折叠
        fold_xi_to_i(ctx, op, 0) ||          // x * 0 → 0
        fold_xi_to_x(ctx, op, 1)) {          // x * 1 → x
        return true;
    }
    return finish_folding(ctx, op);
}
```

### 10.4 代数简化模式

| 辅助函数 | 模式 | 含义 |
|---------|------|------|
| `fold_xi_to_x(op, C)` | `x ⊕ C → x` | 恒等元素 |
| `fold_xi_to_i(op, C)` | `x ⊕ C → C` | 吸收元素 |
| `fold_xx_to_i(op, C)` | `x ⊕ x → C` | 自操作为常量 |
| `fold_xx_to_x(op)` | `x ⊕ x → x` | 自操作为自身 |
| `fold_ix_to_i(op, C)` | `C ⊕ x → C` | 左吸收 |
| `fold_xi_to_not(op, C)` | `x ⊕ C → NOT(x)` | 全1异或取反 |
| `fold_to_not(op)` | 操作 → NOT | 模式匹配取反 |

---

## 11. 逻辑运算优化

### 11.1 fold_and — 与运算

```c
// tcg/optimize.c:1315-1360
static bool fold_and(OptContext *ctx, TCGOp *op)
{
    if (fold_const2_commutative(ctx, op) ||  // 常量折叠
        fold_xi_to_i(ctx, op, 0) ||          // x & 0 → 0
        fold_xi_to_x(ctx, op, -1) ||         // x & -1 → x
        fold_xx_to_x(ctx, op)) {             // x & x → x
        return true;
    }

    // 掩码传播：AND 使已知零位增多
    TempOptInfo *t1 = arg_info(op->args[1]);
    TempOptInfo *t2 = arg_info(op->args[2]);
    return fold_masks_zosa(ctx, op,
                           t1->z_mask & t2->z_mask,    // z: 两者的交集
                           t1->o_mask & t2->o_mask,    // o: 两者的交集
                           t1->s_mask & t2->s_mask);   // s: 两者的交集
}
```

### 11.2 fold_or — 或运算

```c
// tcg/optimize.c:2285-2307
static bool fold_or(OptContext *ctx, TCGOp *op)
{
    if (fold_const2_commutative(ctx, op) ||
        fold_xi_to_x(ctx, op, 0) ||          // x | 0 → x
        fold_xi_to_i(ctx, op, -1) ||          // x | -1 → -1
        fold_xx_to_x(ctx, op)) {              // x | x → x
        return true;
    }
    // z_mask = z1 | z2 (并集)
    // o_mask = o1 | o2 (并集)
    return fold_masks_zosa(ctx, op, z1 | z2, o1 | o2, s1 & s2);
}
```

### 11.3 fold_xor — 异或运算

```c
// tcg/optimize.c:2977-2996
static bool fold_xor(OptContext *ctx, TCGOp *op)
{
    if (fold_const2_commutative(ctx, op) ||
        fold_xx_to_i(ctx, op, 0) ||          // x ^ x → 0
        fold_xi_to_x(ctx, op, 0) ||          // x ^ 0 → x
        fold_xi_to_not(ctx, op, -1)) {        // x ^ -1 → NOT(x)
        return true;
    }
    // z_mask = z1 | z2（可能为1的位是两者的并集）
    // o_mask = ~z1 & ~z2（两者都确定为0的位在结果中也为0...但XOR反转）
    return fold_masks_zosa(ctx, op, ...);
}
```

---

## 12. 分支条件优化

### 12.1 fold_brcond — 条件分支

```c
// tcg/optimize.c:1464-1480
static bool fold_brcond(OptContext *ctx, TCGOp *op)
{
    // 尝试在编译时求值条件
    int i = do_constant_folding_cond1(ctx, op, ...);

    if (i == 0) {
        // ★ 条件恒假 → 删除整个分支指令
        tcg_op_remove(ctx->tcg, op);
        return true;
    }
    if (i > 0) {
        // ★ 条件恒真 → 转换为无条件跳转
        op->opc = INDEX_op_br;
        op->args[0] = op->args[3];  // 跳转目标
        finish_ebb(ctx);             // 结束扩展基本块
    } else {
        // 条件未知 → 保留，标记基本块结束
        finish_bb(ctx);
    }
    return true;
}
```

### 12.2 fold_setcond — 条件设置

```c
// tcg/optimize.c:2548-2565
static bool fold_setcond(OptContext *ctx, TCGOp *op)
{
    // 常量条件 → 替换为 movi 0 或 movi 1
    int i = do_constant_folding_cond1(ctx, op, ...);
    if (i >= 0) {
        return tcg_opt_gen_movi(ctx, op, op->args[0], i);
    }

    // z_mask 优化：结果是布尔值
    return fold_masks_z(ctx, op, 1);  // 结果只有 bit0
}
```

### 12.3 fold_movcond — 条件移动

```c
// tcg/optimize.c:2099-2151
static bool fold_movcond(OptContext *ctx, TCGOp *op)
{
    // 常量条件 → 直接选择 true/false 操作数
    int i = do_constant_folding_cond1(ctx, op, ...);
    if (i >= 0) {
        // i=1: 选择 true 值, i=0: 选择 false 值
        return tcg_opt_gen_mov(ctx, op, op->args[0],
                               op->args[i ? 3 : 4]);
    }
    // true == false → 无论条件如何结果相同
    if (args_are_copies(op->args[3], op->args[4])) {
        return tcg_opt_gen_mov(ctx, op, op->args[0], op->args[3]);
    }
    return finish_folding(ctx, op);
}
```

---

## 13. 扩展与位域优化

### 13.1 fold_exts — 符号扩展

```c
// tcg/optimize.c:2015-2039
static bool fold_exts(OptContext *ctx, TCGOp *op)
{
    if (fold_const1(ctx, op)) return true;  // 常量折叠

    // 利用 s_mask：如果输入的符号已经扩展到目标宽度
    // 例如 ext32s(x) 但 x 的 s_mask 已涵盖高 32 位
    // → 折叠为 copy
    uint64_t s_mask = ...;
    return fold_masks_zs(ctx, op, z_mask, s_mask);
}
```

### 13.2 fold_extu — 零扩展

```c
// tcg/optimize.c:2041-2068
static bool fold_extu(OptContext *ctx, TCGOp *op)
{
    if (fold_const1(ctx, op)) return true;

    // 如果 z_mask 显示高位已经为零 → 折叠为 copy
    // 例如 ext32u(x) 但 x 的 z_mask & 0xFFFF_FFFF_0000_0000 == 0
    return fold_masks_z(ctx, op, z_mask & mask);
}
```

### 13.3 fold_deposit — 位域插入

```c
// tcg/optimize.c:1653-1858
static bool fold_deposit(OptContext *ctx, TCGOp *op)
{
    // deposit(base, insert, pos, len)
    // 将 insert 的低 len 位插入到 base 的 [pos:pos+len-1]

    // 1. 常量折叠：base 和 insert 都是常量
    // 2. 全零/全一优化：
    //    - insert 全零 → AND mask（清除字段）
    //    - base 全零 → shift + AND（提取并放置）
    // 3. 掩码传播：精确追踪哪些位来自 base，哪些来自 insert
}
```

这是最复杂的 fold 函数之一（约 200 行），因为 deposit 操作涉及位域拼接。

---

## 14. 内存操作优化

### 14.1 TCG 加载/存储的值追踪

优化器通过 `mem_copy` 机制追踪 TCG `ld`/`st` 操作，实现**加载已知值**的优化：

```c
// tcg/optimize.c:260-283 — record_mem_copy()
// 记录"temp T 的值存储在 CPUState 偏移 offset 处"

// tcg/optimize.c:311-320 — find_mem_copy_for()
// 查找"CPUState 偏移 offset 处是否有已知值"
```

### 14.2 fold_tcg_ld_memcopy — 加载优化

```c
// tcg/optimize.c:2891-2912
static bool fold_tcg_ld_memcopy(OptContext *ctx, TCGOp *op)
{
    // 如果目标偏移处的值已知（之前存储过）
    // → 将 ld 替换为 mov（从已知 temp 拷贝）
    MemCopyInfo *mc = find_mem_copy_for(ctx, ...);
    if (mc) {
        return tcg_opt_gen_mov(ctx, op, op->args[0], temp_arg(mc->ts));
    }
    return false;
}
```

### 14.3 fold_tcg_st_memcopy — 存储追踪

```c
// tcg/optimize.c:2945-2975
static bool fold_tcg_st_memcopy(OptContext *ctx, TCGOp *op)
{
    // 记录此存储：offset → temp 映射
    // 失效：覆盖同一 offset 的旧记录
    record_mem_copy(ctx, ...);
    return false;
}
```

### 14.4 fold_qemu_ld / fold_qemu_st — Guest 内存

```c
// tcg/optimize.c:2353-2387
static bool fold_qemu_ld_1reg(OptContext *ctx, TCGOp *op)
{
    // Guest 内存加载无法常量折叠（值不可知）
    // 但可以：
    // 1. 根据加载宽度设置 z_mask（如 ld8u → z_mask = 0xFF）
    // 2. 清除 mem_copy 缓存（Guest 内存访问可能使 CPU state 失效）
    return finish_folding(ctx, op);
}

static bool fold_qemu_st(OptContext *ctx, TCGOp *op)
{
    // Guest 内存存储：清除 prev_mb（屏障不能跨过 guest 存储合并）
    return false;
}
```

---

## 15. 内存屏障优化

```c
// tcg/optimize.c:2070-2092
static bool fold_mb(OptContext *ctx, TCGOp *op)
{
    // 连续两个内存屏障可以合并
    if (ctx->prev_mb) {
        // 新屏障的类型 = 两者的并集（取更强的）
        // 删除旧屏障
        tcg_op_remove(ctx->tcg, ctx->prev_mb);
    }
    ctx->prev_mb = op;
    return false;
}
```

示例：

```
优化前:                    优化后:
  mb acquire               mb seq_cst   (合并为最强)
  mb release
```

---

## 16. 进位/借位链优化

多精度算术（128 位加法等）通过进位链优化：

```c
// tcg/optimize.c:1184-1280 — fold_addci / fold_addcio / fold_addco
// tcg/optimize.c:2756-2855 — fold_subbi / fold_subbio / fold_subbo
```

`OptContext.carry_state` 追踪进位/借位状态：
- `-1` — 非常量（运行时值）
- `0` — 常量零进位
- `1` — 常量一进位

当进位为常量时，`addc` 可以简化为普通 `add`（加上常量偏移）。

---

## 17. 全部 fold 函数索引

`tcg/optimize.c` 共 **3244 行**，包含 **76 个 fold 函数**：

### 基础设施函数

| 函数 | 行号 | 功能 |
|------|------|------|
| `fold_const1` | 888 | 单输入常量折叠 |
| `fold_const2` | 899 | 双输入常量折叠 |
| `fold_commutative` | 911 | 交换律规范化 |
| `fold_const2_commutative` | 917 | 交换律 + 常量折叠 |
| `fold_masks_zosa_int` | 929 | 掩码传播核心 |
| `fold_masks_zosa` | 985 | 包装：z+o+s+a |
| `fold_masks_zos` | 992 | 包装：z+o+s |
| `fold_masks_zo` | 998 | 包装：z+o |
| `fold_masks_zs` | 1004 | 包装：z+s |
| `fold_masks_z` | 1010 | 包装：仅 z |
| `fold_masks_s` | 1015 | 包装：仅 s |

### 代数简化函数

| 函数 | 行号 | 模式 |
|------|------|------|
| `fold_to_not` | 1026 | op → NOT |
| `fold_ix_to_i` | 1055 | C ⊕ x → C |
| `fold_ix_to_not` | 1064 | C ⊕ x → NOT(x) |
| `fold_xi_to_i` | 1073 | x ⊕ C → C |
| `fold_xi_to_x` | 1082 | x ⊕ C → x |
| `fold_xi_to_not` | 1091 | x ⊕ C → NOT(x) |
| `fold_xx_to_i` | 1100 | x ⊕ x → C |
| `fold_xx_to_x` | 1109 | x ⊕ x → x |
| `fold_sub_to_neg` | 2659 | 0 - x → neg(x) |

### 算术/逻辑 fold 函数

| 函数 | 行号 | 操作 |
|------|------|------|
| `fold_add` | 1132 | 加法 |
| `fold_add_vec` | 1142 | 向量加法 |
| `fold_addci` | 1184 | 带进位加法(进位输入) |
| `fold_addcio` | 1221 | 带进位加法(进位输入输出) |
| `fold_addco` | 1281 | 带进位加法(进位输出) |
| `fold_sub` | 2703 | 减法 |
| `fold_sub_vec` | 2693 | 向量减法 |
| `fold_subbi` | 2756 | 带借位减法(借位输入) |
| `fold_subbio` | 2795 | 带借位减法(借位输入输出) |
| `fold_subbo` | 2842 | 带借位减法(借位输出) |
| `fold_mul` | 2153 | 乘法 |
| `fold_mul_highpart` | 2163 | 高位乘法 |
| `fold_multiply2` | 2172 | 双结果乘法 |
| `fold_divide` | 1861 | 除法 |
| `fold_remainder` | 2389 | 取余 |
| `fold_neg` | 2248 | 取负 |
| `fold_neg_no_const` | 2239 | 取负(无常量折叠) |
| `fold_and` | 1315 | 与 |
| `fold_andc` | 1363 | 与非 |
| `fold_or` | 2285 | 或 |
| `fold_orc` | 2309 | 或非 |
| `fold_xor` | 2977 | 异或 |
| `fold_eqv` | 1880 | 同或(XNOR) |
| `fold_nand` | 2219 | 与非(NAND) |
| `fold_nor` | 2253 | 或非(NOR) |
| `fold_not` | 2273 | 取反 |

### 移位/位域/扩展

| 函数 | 行号 | 操作 |
|------|------|------|
| `fold_shift` | 2609 | 通用移位 |
| `fold_exts` | 2015 | 符号扩展 |
| `fold_extu` | 2041 | 零扩展 |
| `fold_extract` | 1918 | 位域提取 |
| `fold_extract2` | 1937 | 双寄存器提取 |
| `fold_sextract` | 2587 | 符号位域提取 |
| `fold_deposit` | 1653 | 位域插入(~200行) |
| `fold_bswap` | 1482 | 字节交换 |
| `fold_count_zeros` | 1599 | 前导零计数 |
| `fold_ctpop` | 1632 | 位计数 |
| `fold_dup` | 1870 | 复制 |

### 控制流/条件

| 函数 | 行号 | 操作 |
|------|------|------|
| `fold_brcond` | 1464 | 条件分支 |
| `fold_setcond` | 2548 | 条件设置 |
| `fold_negsetcond` | 2567 | 取负条件设置 |
| `fold_movcond` | 2099 | 条件移动 |

### 内存/调用

| 函数 | 行号 | 操作 |
|------|------|------|
| `fold_mov` | 2094 | 移动 |
| `fold_mb` | 2070 | 内存屏障 |
| `fold_call` | 1532 | 函数调用 |
| `fold_qemu_ld_1reg` | 2353 | Guest 加载(单寄存器) |
| `fold_qemu_ld_2reg` | 2375 | Guest 加载(双寄存器) |
| `fold_qemu_st` | 2382 | Guest 存储 |
| `fold_tcg_ld` | 2861 | TCG 加载 |
| `fold_tcg_ld_memcopy` | 2891 | TCG 加载(内存拷贝优化) |
| `fold_tcg_st` | 2914 | TCG 存储 |
| `fold_tcg_st_memcopy` | 2945 | TCG 存储(内存拷贝追踪) |

### 向量

| 函数 | 行号 | 操作 |
|------|------|------|
| `fold_cmp_vec` | 1569 | 向量比较 |
| `fold_cmpsel_vec` | 1578 | 向量条件选择 |
| `fold_bitsel_vec` | 1408 | 向量位选择 |

---

## 18. 可达代码分析

```c
// tcg/tcg.c:3565 — reachable_code_pass()
```

在 `tcg_optimize()` 之后执行，删除无条件跳转后的不可达代码：

```
优化前:                    优化后:
  br label1                br label1
  add t1, t2, t3           (删除 — 不可达)
  st t1, [addr]            (删除 — 不可达)
label1:                    label1:
  ...                      ...
```

---

## 19. 活跃性分析 Pass 0

```c
// tcg/tcg.c:3804 — liveness_pass_0()
```

Pass 0 执行**生命周期降级**：
- 检查标记为 `TEMP_TB` 的临时变量（仅在 TB 内有效）
- 如果实际生命周期不跨越基本块边界，降级为 `TEMP_EBB`
- 这使得后续 Pass 1 可以更积极地回收寄存器

---

## 20. 活跃性分析 Pass 1

```c
// tcg/tcg.c:3883 — liveness_pass_1()
```

Pass 1 是核心的**反向数据流分析**：

```
从最后一条 IR 反向遍历到第一条：
    对于每条操作 op：
        1. 检查所有输出（oargs）：
           - 如果输出标记为 DEAD → 此操作的此输出无用
           - 如果所有输出都 DEAD 且无副作用 → 删除整条操作
        2. 标记所有输入（iargs）为 LIVE
        3. 在基本块边界重置全局变量状态
```

### 死代码删除

```c
// Pass 1 中的关键逻辑
if (all_outputs_dead && !(def->flags & TCG_OPF_SIDE_EFFECTS)) {
    tcg_op_remove(s, op);  // ★ 删除死操作
    continue;
}
```

### 活跃性编码

```c
// tcg/tcg.c:3888-3890
#define DEAD_ARG  4
#define SYNC_ARG  1
// op->life 编码每个参数的状态：
// bit 0 (SYNC_ARG) — 值需要同步到内存
// bit 2 (DEAD_ARG) — 值在此操作后死亡
```

---

## 21. 活跃性分析 Pass 2

```c
// tcg/tcg.c:4270 — liveness_pass_2()
```

Pass 2 执行**间接→直接临时变量替换**：
- 间接临时变量（`TEMP_EBB` 需要经过内存中转）
- 如果 Pass 2 发现可以用直接临时变量替换 → 减少 load/store
- 如果有替换发生 → 返回 true → 重跑 Pass 1

```c
// tcg/tcg.c:6611-6613
if (liveness_pass_2(s)) {     // 如果有替换
    liveness_pass_1(s);       // 重新运行 Pass 1
}
```

---

## 22. 活跃性数据编码

每个 `TCGTemp` 的活跃状态：

| 状态 | 含义 |
|------|------|
| `TS_DEAD` | 临时变量当前无有效值 |
| `TS_MEM` | 值存储在内存中（已 spill） |

每个 `TCGOp` 的 `op->life` 字段（`TCGLifeData`）：
- 为每个参数编码 `DEAD_ARG`（此操作后不再使用）和 `SYNC_ARG`（需要写回内存）
- 寄存器分配器根据此信息决定寄存器回收和 spill

### 活跃性辅助函数

```c
// tcg/tcg.c:3662-3788
la_temp_pref(ts)       // 获取 temp 的寄存器偏好集合
la_reset_pref(ts)      // 重置偏好
la_func_end(s)         // 函数结束：同步所有全局变量
la_bb_end(s, nb_temps) // 基本块结束：同步全局变量
la_global_sync(s, ts)  // 标记全局变量需要同步
la_global_kill(s)      // 标记所有全局变量为 DEAD
la_cross_call(s)       // 跨函数调用：杀死调用者保存寄存器
```

---

## 23. 寄存器分配框架

寄存器分配与代码生成在**同一遍**中完成，前向遍历 IR：

```c
// tcg/tcg.c — 寄存器分配核心函数
tcg_reg_alloc_start()    // 2649 — 初始化分配状态
tcg_reg_alloc_mov()      // 4923 — MOV 指令分配
tcg_reg_alloc_dup()      // 5023 — DUP 指令分配
tcg_reg_alloc_op()       // 5133 — 通用操作分配
tcg_reg_alloc_call()     // 5874 — 函数调用分配
tcg_reg_alloc_pair()     // 4707 — 寄存器对分配
tcg_reg_alloc_bb_end()   // 4843 — 基本块结束处理
tcg_reg_alloc_cbranch()  // 4875 — 条件分支处理
tcg_reg_alloc_do_movi()  // 4902 — MOVI 分配
```

### 通用分配流程（tcg_reg_alloc_op）

```
1. 为每个输入参数：
   a. 检查是否已在寄存器中
   b. 如果不在，调用 temp_load() 加载
   c. 确保满足约束条件

2. 为每个输出参数：
   a. 分配空闲寄存器（优先满足偏好）
   b. 如果无空闲，spill 一个现有 temp

3. 调用后端 tcg_out_op() 生成宿主机代码

4. 根据 DEAD_ARG 释放死亡的 temp
5. 根据 SYNC_ARG 将需要同步的 temp 写回内存
```

---

## 24. 约束系统

### 24.1 TCGArgConstraint

```c
// tcg/tcg.c:910+ — constraint_sets[]
// tcg/tcg.c:3199-3299 — process_constraint_sets()
```

约束定义了每个操作码的每个参数可以使用哪些寄存器：

| 约束 | 含义 |
|------|------|
| `r` | 任意通用寄存器 |
| `R` | 与特定参数相同的寄存器（alias） |
| `p` | 成对寄存器（pair） |
| `0` | 与 arg[0] 相同寄存器（two-address） |
| `i` | 立即数 |

### 24.2 约束解析

```c
// tcg/tcg.c:3390-3421 — opcode_args_ct()
// 根据操作码和类型选择约束集
```

---

## 25. Spill/Reload 机制

### 25.1 temp_load — 重新加载

```c
// tcg/tcg.c:4666+ — temp_load()
// 将临时变量从常量/内存加载到寄存器
//   常量 → tcg_out_movi()
//   内存 → tcg_out_ld()
//   向量常量 → tcg_out_dupi_vec()
```

### 25.2 temp_save — 保存到内存

```c
// tcg/tcg.c:4718+ — temp_save()
// 将寄存器中的值写回到 CPUState 偏移
//   → tcg_out_st()
```

### 25.3 Spill 选择策略

```c
// tcg/tcg.c:4696-4756 — tcg_reg_alloc()
// 1. 优先选择空闲寄存器（在偏好集合中）
// 2. 如果无空闲，选择目标架构分配顺序中的第一个可用寄存器
// 3. 驱逐该寄存器中的 temp（调用 temp_sync + 标记 TS_MEM）
```

---

## 26. ARM64 延迟标志优化

### 26.1 条件标志全局变量

ARM64 的 NZCV 条件标志在 TCG 中以 4 个独立的全局变量表示：

```c
// target/arm/tcg/translate.c:49-53
// cpu_NF, cpu_ZF, cpu_CF, cpu_VF — TCG 全局变量
// 每个标志独立追踪，允许部分更新
```

### 26.2 延迟计算模式

ARM64 翻译器**仅在指令需要设置标志时才生成标志计算代码**：

```c
// target/arm/tcg/translate-a64.c:1000-1021 — gen_logic_CC()
static void gen_logic_CC(int sf, TCGv_i64 result)
{
    // 仅设置 N 和 Z 标志（逻辑运算不影响 C/V）
    tcg_gen_mov_i64(cpu_NF, result);  // N = result 的最高位
    tcg_gen_mov_i64(cpu_ZF, result);  // Z = (result == 0)
}
```

### 26.3 标志物化

当需要从 PSTATE.NZCV 寄存器读取完整标志时：

```c
// target/arm/tcg/translate-a64.c:2525-2534 — gen_set_nzcv()
static void gen_set_nzcv(TCGv_i64 tcg_rt)
{
    // 从完整 NZCV 值提取各个标志
    // N = bit[31], Z = bit[30], C = bit[29], V = bit[28]
    tcg_gen_shri_i64(cpu_NF, tcg_rt, 31);  // ...
}
```

### 26.4 FP 标志辅助

```c
// target/arm/tcg/helper-a64.c:120-143 — float_rel_to_flags()
// 将浮点比较结果转换为 NZCV 标志
```

### 26.5 A32 条件执行

对于 ARM32（A32/T32），条件执行更复杂：

```c
// target/arm/tcg/translate.c:6336-6340 — condexec_mask / condexec_cond
// IT block 状态追踪
// 每条指令都可能有条件前缀
```

---

## 27. 优化效果观察

### 27.1 使用 -d 选项查看

```bash
# 查看优化前的 IR
qemu-system-aarch64 -d op ...

# 查看优化后的 IR
qemu-system-aarch64 -d op_opt ...

# 查看最终生成的宿主机代码
qemu-system-aarch64 -d out_asm ...
```

### 27.2 典型优化效果

```
优化前 IR:
  movi_i64 t0, 0x1000       // 加载常量
  movi_i64 t1, 0x2000       // 加载常量
  add_i64  t2, t0, t1       // t0 + t1
  mov_i64  t3, t2           // 拷贝
  st_i64   t3, env, 0x100   // 存储

优化后 IR:
  movi_i64 t2, 0x3000       // ★ 常量折叠: 0x1000+0x2000=0x3000
  st_i64   t2, env, 0x100   // ★ 拷贝传播: t3→t2; 死代码删除: t0,t1,mov
```

---

## 28. 完整优化管线追踪示例

以 ARM64 `ADD X0, X1, #0` 为例：

### Stage 1: IR 生成

```
ld_i64   t0, env, offsetof(regs[1])    // 加载 X1
movi_i64 t1, 0                          // 常量 0
add_i64  t2, t0, t1                     // X1 + 0
st_i64   t2, env, offsetof(regs[0])    // 存储到 X0
```

### Stage 2: tcg_optimize()

1. **常量传播**: t1 标记为 `z_mask = o_mask = 0`（常量 0）
2. **fold_add**: 检测 `fold_xi_to_x(ctx, op, 0)` → `x + 0 → x`
3. 替换为 `mov t2, t0`
4. **fold_mov**: `tcg_opt_gen_mov()` → t2 加入 t0 的拷贝链
5. **拷贝传播**: st 的输入 t2 替换为 t0

```
ld_i64   t0, env, offsetof(regs[1])
st_i64   t0, env, offsetof(regs[0])    // ★ 3条→2条
```

### Stage 3: liveness_pass_1()

- t1: DEAD（从未使用）→ movi 已删除
- t0: 在 st 后 DEAD → 寄存器可回收

### Stage 4: 寄存器分配 + 代码生成

```asm
; AArch64 宿主机代码
ldr x20, [x19, #offsetof(regs[1])]     ; 加载 X1
str x20, [x19, #offsetof(regs[0])]     ; 存储到 X0
; 仅 2 条宿主机指令！
```

---

## 29. 关键源文件索引

| 文件 | 内容 | 行数 |
|------|------|------|
| `tcg/optimize.c` | 常量折叠/拷贝传播/掩码追踪/76 个 fold 函数 | 3244 |
| `tcg/tcg.c` | 活跃性分析/寄存器分配/代码生成框架 | ~6600+ |
| `tcg/region.c` | 代码缓冲区管理 | ~900 |
| `include/tcg/tcg-opc.h` | TCG 操作码定义 + 标志 | ~180 |
| `target/arm/tcg/translate-a64.c` | ARM64 前端翻译（含标志优化） | ~10700 |
| `target/arm/tcg/translate.c` | ARM32 前端翻译（含条件执行） | ~9000 |

### 关键函数速查

| 函数 | 文件:行 | 作用 |
|------|---------|------|
| `tcg_gen_code` | tcg.c:6556 | 完整代码生成入口 |
| `tcg_optimize` | optimize.c:2999 | 优化器主入口 |
| `fold_masks_zosa_int` | optimize.c:929 | 掩码传播核心 |
| `do_constant_folding_2` | optimize.c:427 | 常量计算引擎 |
| `copy_propagate` | optimize.c:839 | 拷贝替换 |
| `reachable_code_pass` | tcg.c:3565 | 不可达代码消除 |
| `liveness_pass_0` | tcg.c:3804 | 生命周期降级 |
| `liveness_pass_1` | tcg.c:3883 | 反向活跃性分析 |
| `liveness_pass_2` | tcg.c:4270 | 间接→直接替换 |
| `tcg_reg_alloc_op` | tcg.c:5133 | 通用寄存器分配 |

---

> **文档版本**: v1.0
> **分析基础**: QEMU 11.0.50 (commit HEAD)
> **验证方法**: 所有行号引用均通过 sed/grep 交叉验证
