# ARM64 TCG 前端/后端代码生成深度分析：IR 中间表示、翻译循环、优化 Pass、寄存器分配与 AArch64 主机代码发射

> 基于 QEMU 11.0.50 源码分析，涵盖 TCG（Tiny Code Generator）完整代码生成流水线：
> TCGOpcode 操作码体系（180+ 标量/向量指令）、TCGTemp 临时变量五种生命周期、TCGOp 链表与 TCGContext 全局上下文、
> translator_loop 通用翻译循环、ARM64 TranslatorOps 回调链、decodetree 解码树（a64.decode）、
> tcg_gen_* 前端 IR 生成 API、tcg_optimize 优化 Pass（常量折叠/拷贝传播/死代码消除）、
> liveness_pass_1 活性分析、寄存器分配器（单/双寄存器分配/溢出/同步）、
> AArch64 后端代码发射（tcg_out_* 指令编码/TLB 内联查找/TB 链接/Prologue-Epilogue）、
> guest 内存访问快慢路径、TB 缓冲区管理与代码刷新。

---

## 目录

1. [架构概述](#1-架构概述)
2. [TCG IR 中间表示系统](#2-tcg-ir-中间表示系统)
3. [TCGOpcode 操作码全景](#3-tcgopcode-操作码全景)
4. [TCGContext 全局上下文](#4-tcgcontext-全局上下文)
5. [translator_loop 通用翻译循环](#5-translator_loop-通用翻译循环)
6. [ARM64 前端翻译流程](#6-arm64-前端翻译流程)
7. [tcg_gen_* 前端 IR 生成 API](#7-tcg_gen-前端-ir-生成-api)
8. [tb_gen_code — TB 生成入口](#8-tb_gen_code--tb-生成入口)
9. [tcg_optimize — 优化 Pass](#9-tcg_optimize--优化-pass)
10. [liveness_pass_1 — 活性分析](#10-liveness_pass_1--活性分析)
11. [寄存器分配器](#11-寄存器分配器)
12. [AArch64 后端代码发射](#12-aarch64-后端代码发射)
13. [Guest 内存访问翻译](#13-guest-内存访问翻译)
14. [TB 链接与运行时补丁](#14-tb-链接与运行时补丁)
15. [Prologue 与 Epilogue](#15-prologue-与-epilogue)
16. [代码缓冲区管理](#16-代码缓冲区管理)
17. [完整流水线总结](#17-完整流水线总结)

---

## 1. 架构概述

TCG 是 QEMU 的动态二进制翻译引擎，将 guest 指令（如 ARM64）翻译为 host 指令（如 x86-64 或 AArch64）。翻译流水线分为三个阶段：

```
Guest 指令 ──→ [前端] ──→ TCG IR ops ──→ [优化] ──→ TCG IR ops ──→ [后端] ──→ Host 机器码
  ARM64          解码+生成      中间表示        常量折叠等        优化后 IR        寄存器分配+编码
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `include/tcg/tcg.h` | 59-64 | TCGOpcode 枚举（由 tcg-opc.h 生成） |
| `include/tcg/tcg.h` | 253-293 | TCGTempKind、TCGTemp 结构体 |
| `include/tcg/tcg.h` | 310-329 | TCGOp 结构体 |
| `include/tcg/tcg.h` | 346-441 | TCGContext 上下文 |
| `include/tcg/tcg-opc.h` | 29-183 | 所有 TCG 操作码定义 |
| `include/exec/translator.h` | 34-50 | DisasJumpType 枚举 |
| `include/exec/translator.h` | 95-125 | TranslatorOps 接口 |
| `accel/tcg/translator.c` | 122-248 | translator_loop 通用循环 |
| `accel/tcg/translate-all.c` | 261-420 | tb_gen_code TB 生成入口 |
| `tcg/tcg.c` | 1762-1821 | tcg_context_init / tcg_init |
| `tcg/tcg.c` | 3448-3519 | tcg_op_remove / tcg_emit_op |
| `tcg/tcg.c` | 3692-3718 | la_bb_end 活性分析 |
| `tcg/tcg.c` | 3882-4265 | liveness_pass_1 |
| `tcg/tcg.c` | 4645-4705 | 单寄存器分配 |
| `tcg/tcg.c` | 5133-5799 | 寄存器分配调度 / tcg_gen_code |
| `tcg/optimize.c` | 34-61 | OptContext / TempOptInfo |
| `tcg/optimize.c` | 3000-3060 | tcg_optimize 主函数 |
| `target/arm/tcg/translate-a64.c` | 10667-10750 | init_disas_context |
| `target/arm/tcg/translate-a64.c` | 10774-10810 | translate_insn |
| `target/arm/tcg/translate-a64.c` | 10876-10944 | tb_stop |
| `target/arm/tcg/translate-a64.c` | 10946-10960 | aarch64_translator_ops |
| `tcg/aarch64/tcg-target.h` | 19-46 | TCGReg 枚举 |
| `tcg/aarch64/tcg-target.c.inc` | 47-72 | 寄存器分配顺序 |
| `tcg/aarch64/tcg-target.c.inc` | 1104-1264 | tcg_out_movi / tcg_out_mov |
| `tcg/aarch64/tcg-target.c.inc` | 1650-1757 | prepare_host_addr TLB 查找 |
| `tcg/aarch64/tcg-target.c.inc` | 1991-2059 | tcg_out_exit_tb / goto_tb / 补丁 |
| `tcg/aarch64/tcg-target.c.inc` | 3467-3534 | Prologue / Epilogue |

---

## 2. TCG IR 中间表示系统

### 2.1 TCGTemp — 临时变量

```c
// include/tcg/tcg.h:253-293
typedef enum TCGTempKind {
    TEMP_EBB,      // 259: 扩展基本块内有效，块末死亡
    TEMP_TB,       // 261: 整个 TB 内有效，TB 结束后死亡
    TEMP_GLOBAL,   // 263: 跨 TB 有效（如 CPU 寄存器映射）
    TEMP_FIXED,    // 265: 绑定到固定物理寄存器
    TEMP_CONST,    // 267: 固定常量值
} TCGTempKind;

typedef struct TCGTemp {
    TCGReg reg:8;          // 当前分配的物理寄存器
    TCGTempVal val_type:8; // 值类型（寄存器/内存/常量/死亡）
    TCGType base_type:8;   // 基本类型
    TCGType type:8;        // 当前类型
    TCGTempKind kind:3;    // 生命周期种类
    int64_t val;           // 常量值
    struct TCGTemp *mem_base;  // 内存基址 temp
    intptr_t mem_offset;       // 内存偏移
    const char *name;          // 调试名称
    uintptr_t state;           // Pass 专用状态
    void *state_ptr;           // Pass 专用指针
} TCGTemp;
```

**五种生命周期**与 guest CPU 寄存器映射对应：
- `TEMP_GLOBAL`：`env->xregs[0..30]`、`env->pc` 等 CPU 寄存器映射到 TCG 全局变量
- `TEMP_FIXED`：`TCG_AREG0`（X19）绑定到 `env` 指针
- `TEMP_TB`/`TEMP_EBB`：翻译过程中的中间计算结果
- `TEMP_CONST`：立即数常量池

### 2.2 TCGOp — 操作节点

```c
// include/tcg/tcg.h:310-329
struct TCGOp {
    TCGOpcode opc   : 8;   // 操作码
    unsigned nargs  : 8;   // 参数数量
    unsigned param1 : 8;   // 操作码参数 1
    unsigned param2 : 8;   // 操作码参数 2
    TCGLifeData life;      // 活性数据（DEAD_ARG / SYNC_ARG 位图）
    QTAILQ_ENTRY(TCGOp) link;  // 链表连接
    TCGRegSet output_pref[2];  // 输出寄存器偏好
    TCGArg args[];             // 变长参数数组
};
```

所有 TCGOp 通过双向链表 `TCGContext.ops` 连接，形成 IR 指令流。

---

## 3. TCGOpcode 操作码全景

操作码在 `include/tcg/tcg-opc.h:29-183` 中通过 `DEF()` 宏定义，编译时展开为 `INDEX_op_*` 枚举。

### 3.1 标量操作码（29-124）

| 类别 | 操作码 | 行号 | 说明 |
|------|--------|------|------|
| **控制流** | discard, set_label, br, brcond | 30-37 | 标签/分支 |
| **内存屏障** | mb | 39 | 内存屏障 |
| **数据移动** | mov | 41 | 寄存器间传送 |
| **算术** | add, sub, mul, divs, divu, neg | 43-93 | 算术运算 |
| **逻辑** | and, or, xor, andc, orc, nand, nor, eqv, not | 44-94 | 位逻辑 |
| **移位** | shl, shr, sar, rotl, rotr | 82-88 | 移位/旋转 |
| **位操作** | deposit, extract, sextract, extract2 | 52-59 | 位域插入/提取 |
| **比较** | setcond, negsetcond, movcond | 67-85 | 条件设置/选择 |
| **字节序** | bswap16, bswap32, bswap64 | 46-48 | 字节翻转 |
| **位计数** | clz, ctz, ctpop | 49-51 | 前导零/尾零/popcount |
| **加载/存储** | ld8u/s, ld16u/s, ld32u/s, ld, st8, st16, st32, st | 60-92 | 内存访问（TCG 局部） |
| **进位链** | addco, addci, addcio, subbo, subbi, subbio | 96-104 | 多精度算术 |
| **类型转换** | ext_i32_i64, extu_i32_i64, extrl/h_i64_i32 | 107-110 | 32↔64 位 |
| **TB 控制** | exit_tb, goto_tb, goto_ptr | 114-116 | TB 退出/链接 |
| **Guest 内存** | qemu_ld, qemu_st, qemu_ld2, qemu_st2 | 121-124 | Guest 内存访问（含 TLB） |
| **指令标记** | insn_start | 112 | Guest 指令边界标记 |

### 3.2 向量操作码（126-179）

| 类别 | 操作码 | 说明 |
|------|--------|------|
| **数据移动** | mov_vec, dup_vec, dupm_vec | 128-134 |
| **加载/存储** | ld_vec, st_vec | 132-133 |
| **算术** | add_vec, sub_vec, mul_vec, neg_vec, abs_vec | 136-140 |
| **饱和** | ssadd, usadd, sssub, ussub | 141-144 |
| **比较/选择** | smin, umin, smax, umax, cmp_vec, cmpsel_vec, bitsel_vec | 145-179 |
| **逻辑** | and_vec, or_vec, xor_vec, andc_vec, orc_vec, nand_vec, nor_vec, eqv_vec, not_vec | 150-158 |
| **移位** | shli_vec, shri_vec, sari_vec, rotli_vec, shls_vec, shrs_vec, ... | 160-174 |

---

## 4. TCGContext 全局上下文

```c
// include/tcg/tcg.h:346-441
struct TCGContext {
    // 内存池
    uintptr_t pool_cur, pool_end;               // 347-348

    // 临时变量管理
    int nb_labels, nb_globals, nb_temps;         // 349-351
    int nb_indirects, nb_ops;                    // 352-353
    TCGType addr_type;                           // 354: guest 地址类型
    TCGBar guest_mo;                             // 355: guest 内存序

    // 寄存器分配状态
    TCGRegSet reserved_regs;                     // 357: 保留寄存器集
    intptr_t current_frame_offset;               // 358: 栈帧偏移
    TCGTemp *frame_temp;                         // 361: 栈帧基址 temp

    // 代码缓冲区
    TranslationBlock *gen_tb;                    // 363: 当前翻译的 TB
    tcg_insn_unit *code_buf;                     // 364: TB 代码起始
    tcg_insn_unit *code_ptr;                     // 365: 当前写入位置
    void *code_gen_buffer;                       // 376: 全局代码缓冲区
    size_t code_gen_buffer_size;                 // 377: 缓冲区大小
    void *code_gen_highwater;                    // 382: 刷新阈值

    // 操作链表
    QTAILQ_HEAD(, TCGOp) ops, free_ops;          // 423: 活跃/空闲 op 链表
    QSIMPLEQ_HEAD(, TCGLabel) labels;            // 424: 标签链表
    TCGOp *emit_before_op;                       // 430: 插入点

    // 临时变量数组
    GHashTable *const_table[TCG_TYPE_COUNT];     // 419: 常量哈希表
    TCGTempSet free_temps[TCG_TYPE_COUNT];        // 420: 空闲 temp 集
    TCGTemp temps[TCG_MAX_TEMPS];                // 421: 全局+局部 temp 数组

    // 寄存器映射
    TCGTemp *reg_to_temp[TCG_TARGET_NB_REGS];   // 434: 寄存器→temp 映射

    // 异常退出
    sigjmp_buf jmp_trans;                        // 440: 溢出跳转
};
```

**关键常量**：
- `TCG_MAX_TEMPS` = 512（tcg.h:121）
- `TCG_MAX_INSNS` = 512（tcg.h:122）— 每个 TB 最多翻译 512 条 guest 指令

---

## 5. translator_loop 通用翻译循环

```c
// accel/tcg/translator.c:122-248
void translator_loop(CPUState *cpu, TranslationBlock *tb, int *max_insns,
                     vaddr pc, void *host_pc, const TranslatorOps *ops,
                     DisasContextBase *db, TCGType addr_type)
{
    // 第1步：初始化 DisasContextBase (134-145)
    db->tb = tb; db->pc_first = pc; db->pc_next = pc;
    db->is_jmp = DISAS_NEXT; db->num_insns = 0;

    // 第2步：调用目标特定初始化 (148)
    ops->init_disas_context(db, cpu);

    // 第3步：生成 TB 入口代码 (152)
    icount_start_insn = gen_tb_start(db, cflags);
    ops->tb_start(db, cpu);                      // 153

    // 第4步：主翻译循环 (159-204)
    while (true) {
        *max_insns = ++db->num_insns;
        ops->insn_start(db, cpu);                // 161: 标记指令边界
        ops->translate_insn(db, cpu);            // 178: 翻译一条指令

        if (db->is_jmp != DISAS_NEXT) break;     // 194: 指令终止 TB
        if (tcg_op_buf_full() ||                 // 200: op 缓冲区满
            db->num_insns >= db->max_insns) {    // 200: 达到指令上限
            db->is_jmp = DISAS_TOO_MANY; break;
        }
    }

    // 第5步：生成 TB 退出代码 (207-208)
    ops->tb_stop(db, cpu);
    gen_tb_end(tb, cflags, icount_start_insn, db->num_insns);

    // 第6步：记录 TB 元数据 (226-227)
    tb->size = db->pc_next - db->pc_first;
    tb->icount = db->num_insns;
}
```

### DisasJumpType 终止条件

```c
// include/exec/translator.h:34-50
DISAS_NEXT         = 0,  // 继续下一条指令
DISAS_TOO_MANY     = 1,  // 达到指令/op 上限
DISAS_NORETURN     = 2,  // 后续代码不可达（异常/系统调用）
DISAS_TARGET_0..11       // 目标特定（ARM64 定义了 EXIT/JUMP/WFI/YIELD 等）
```

---

## 6. ARM64 前端翻译流程

### 6.1 TranslatorOps 回调表

```c
// target/arm/tcg/translate-a64.c:10946-10952
const TranslatorOps aarch64_translator_ops = {
    .init_disas_context = aarch64_tr_init_disas_context,  // 10667-10750
    .tb_start           = aarch64_tr_tb_start,
    .insn_start         = aarch64_tr_insn_start,
    .translate_insn     = aarch64_tr_translate_insn,       // 10774-10810
    .tb_stop            = aarch64_tr_tb_stop,              // 10876-10944
};
```

### 6.2 translate_insn 单指令翻译

```c
// translate-a64.c:10774-10810
static void aarch64_tr_translate_insn(DisasContextBase *dcbase, CPUState *cpu)
{
    // 1. 单步检查：ss_active && !pstate_ss → 生成单步异常 (10782-10797)
    // 2. PC 对齐检查：pc & 3 → 对齐错误 (10800-10810)
    // 3. 读取 4 字节指令 (translator_ldl_swap)
    // 4. 调用 decodetree 生成的解码函数
    // 5. 更新 pc_next += 4
}
```

### 6.3 decodetree 解码

ARM64 使用 `target/arm/tcg/a64.decode` 解码树文件，由 `scripts/decodetree.py` 编译为 C 解码函数。每条指令映射到 `trans_*` 函数：

```
a64.decode 规则 → 生成 decode_xxx() → 调用 trans_ADD_i() 等
→ trans_ADD_i() 调用 tcg_gen_add_i64() 生成 TCG IR
```

### 6.4 tb_stop — TB 退出代码生成

```c
// translate-a64.c:10876-10944
static void aarch64_tr_tb_stop(DisasContextBase *dcbase, CPUState *cpu)
{
    if (ss_active) {
        // 单步模式：生成单步完成异常 (10880-10896)
        gen_step_complete_exception(dc);
    } else {
        switch (dc->base.is_jmp) {
        case DISAS_NEXT / DISAS_TOO_MANY:
            gen_goto_tb(dc, 1, 4);           // 10901: 直接链接到下一 TB
        case DISAS_EXIT:
            tcg_gen_exit_tb(NULL, 0);        // 10908: 退出到主循环
        case DISAS_JUMP:
            tcg_gen_lookup_and_goto_ptr();   // 10914: 间接 TB 查找
        case DISAS_WFI:
            gen_helper_wfi(tcg_env, ...);    // 10933: 等待中断
            tcg_gen_exit_tb(NULL, 0);        // 10938: 必须返回主循环
        }
    }
}
```

---

## 7. tcg_gen_* 前端 IR 生成 API

### 7.1 API 声明与实现

- 声明：`include/tcg/tcg-op-common.h`（180+ 函数）
- 实现：`tcg/tcg-op.c`、`tcg/tcg-op-ldst.c`、`tcg/tcg-op-gvec.c`

### 7.2 Op 生成机制

```c
// tcg/tcg.c:3510-3519
TCGOp *tcg_emit_op(TCGOpcode opc, unsigned nargs)
{
    TCGOp *op = tcg_op_alloc(opc, nargs);  // 3512: 分配 op 节点

    if (tcg_ctx->emit_before_op) {
        QTAILQ_INSERT_BEFORE(tcg_ctx->emit_before_op, op, link);  // 3515
    } else {
        QTAILQ_INSERT_TAIL(&tcg_ctx->ops, op, link);  // 3517
    }
    return op;
}
```

每个 `tcg_gen_*` 函数最终调用 `tcg_emit_op()` 将新 op 追加到 `tcg_ctx->ops` 链表。

### 7.3 关键 gen 函数

| 函数 | 功能 | 对应操作码 |
|------|------|-----------|
| `tcg_gen_mov_i64` | 寄存器拷贝 | INDEX_op_mov |
| `tcg_gen_add_i64` | 64 位加法 | INDEX_op_add |
| `tcg_gen_sub_i64` | 64 位减法 | INDEX_op_sub |
| `tcg_gen_and_i64` | 位与 | INDEX_op_and |
| `tcg_gen_ld_i64` | 从 env 加载 | INDEX_op_ld |
| `tcg_gen_st_i64` | 存储到 env | INDEX_op_st |
| `tcg_gen_brcond_i64` | 条件分支 | INDEX_op_brcond |
| `tcg_gen_goto_tb` | TB 直接链接 | INDEX_op_goto_tb |
| `tcg_gen_exit_tb` | TB 退出 | INDEX_op_exit_tb |
| `tcg_gen_lookup_and_goto_ptr` | 间接 TB 查找 | INDEX_op_goto_ptr |

---

## 8. tb_gen_code — TB 生成入口

```c
// accel/tcg/translate-all.c:261-420
TranslationBlock *tb_gen_code(CPUState *cpu, TCGTBCPUState s)
{
    // 第1步：物理地址转换 (274)
    phys_pc = get_page_addr_code_hostp(env, s.pc, &host_pc);

    // 第2步：设置最大指令数 (281-285)
    max_insns = s.cflags & CF_COUNT_MASK;  // 默认 TCG_MAX_INSNS=512

    // 第3步：分配 TB (289)
    tb = tcg_tb_alloc(tcg_ctx);
    // 溢出时 → tb_flush → 重试 (290-302)

    // 第4步：填充 TB 元数据 (304-316)
    gen_code_buf = tcg_ctx->code_gen_ptr;
    tb->tc.ptr = tcg_splitwx_to_rx(gen_code_buf);
    tb->pc = s.pc; tb->flags = s.flags; tb->cflags = s.cflags;

    // 第5步：调用翻译（前端+优化+后端）(324)
    gen_code_size = setjmp_gen_code(env, tb, s.pc, host_pc, &max_insns, &ti);
    // 内部调用链：
    //   → target translate_code (aarch64_translate_code)
    //     → translator_loop（前端 IR 生成）
    //   → tcg_gen_code（优化 + 后端代码生成）

    // 第6步：处理溢出重试 (325-387)
    // -1: 代码缓冲区溢出 → tb_flush → 重试
    // -2: 单 TB 太大 → max_insns /= 2 → 重试
    // -3: 页锁序问题 → 重新翻译

    // 第7步：记录搜索信息 (391)
    search_size = encode_search(tb, ...);

    // 第8步：性能统计和日志 (403-420)
    tb->tc.size = gen_code_size;
}
```

---

## 9. tcg_optimize — 优化 Pass

### 9.1 优化上下文

```c
// tcg/optimize.c:50-61
typedef struct OptContext {
    TCGContext *tcg;
    TCGOp *prev_mb;              // 上一个内存屏障
    TCGTempSet temps_used;        // 已使用 temp 集
    IntervalTreeRoot mem_copy;    // 内存拷贝追踪（区间树）
    TCGType type;                 // 当前 op 类型
    int carry_state;              // 进位状态（-1/0/1）
} OptContext;

// tcg/optimize.c:41-48
typedef struct TempOptInfo {
    TCGTemp *prev_copy, *next_copy;  // 拷贝链（双向循环链表）
    uint64_t z_mask;   // 零位掩码：位=0 → 值=0
    uint64_t o_mask;   // 一位掩码：位=1 → 值=1
    uint64_t s_mask;   // 符号位掩码：位=1 → 值与 MSB 相同
} TempOptInfo;
```

### 9.2 主优化流程

```c
// tcg/optimize.c:3000-3060
void tcg_optimize(TCGContext *s)
{
    // 初始化每个 temp 的 TempOptInfo (3013-3016)

    QTAILQ_FOREACH_SAFE(op, &s->ops, link, op_next) {
        // 1. 处理 call 特殊情况 (3024-3027)
        if (opc == INDEX_op_call) { fold_call(&ctx, op); continue; }

        // 2. 初始化参数信息 (3030)
        init_arguments(&ctx, op, ...);

        // 3. 拷贝传播 (3031)
        copy_propagate(&ctx, op, ...);

        // 4. 操作码特定折叠 (3040-3180)
        switch (opc) {
            case INDEX_op_add: done = fold_add(&ctx, op); break;
            case INDEX_op_and: done = fold_and(&ctx, op); break;
            case INDEX_op_sub: done = fold_sub(&ctx, op); break;
            // ... 按字母序处理每种操作码
        }
    }
}
```

### 9.3 优化种类

| 优化 | 说明 | 机制 |
|------|------|------|
| **常量折叠** | 两个操作数都是常量 → 计算结果替换为常量 | fold_* 函数中检查 z_mask/o_mask |
| **拷贝传播** | 使用拷贝源替代当前 temp | copy_propagate + TempOptInfo 拷贝链 |
| **死代码消除** | 输出 temp 未被使用 → 删除 op | tcg_op_remove (tcg.c:3448-3464) |
| **条件折叠** | 常量条件分支 → 无条件分支或删除 | fold_brcond, fold_setcond |
| **掩码追踪** | z_mask/o_mask/s_mask 跨 op 传播 | 精确跟踪哪些位是已知的 |
| **内存拷贝追踪** | load 后未 store → 可用内存拷贝 | IntervalTree 追踪 ld/st 区间 |

---

## 10. liveness_pass_1 — 活性分析

```c
// tcg/tcg.c:3882-4265
static void liveness_pass_1(TCGContext *s)
{
    // 从末尾向前遍历 op 链表 (3899)
    QTAILQ_FOREACH_REVERSE_SAFE(op, &s->ops, link, op_prev) {
        switch (opc) {
        case INDEX_op_call:
            // 处理函数调用：caller-saved 寄存器全部死亡 (3909-3970)
            // 纯函数（NO_SIDE_EFFECTS）若输出未用 → 删除 (3918-3933)
            break;

        case INDEX_op_insn_start:
            // 指令边界：标记同步点 (3973-3983)
            break;

        case INDEX_op_discard:
            // 显式丢弃 temp (4031-4036)
            break;

        default:
            // 通用活性分析 (4156-4158)
            // 如果所有输出 temp 都死亡 → 删除整个 op
            if (all_outputs_dead) {
                tcg_op_remove(s, op);
                continue;
            }
            break;
        }

        // 设置 op->life 位图：标记每个参数的 DEAD/SYNC 位 (4200-4250)
    }
}
```

### 活性状态

```c
// tcg/tcg.c:3692-3718  la_bb_end()
// 基本块末尾：
//   TEMP_FIXED/GLOBAL/TB → TS_DEAD | TS_MEM（需回写内存）
//   TEMP_EBB/CONST       → TS_DEAD（直接死亡）
```

活性信息存储在 `op->life` 中（tcg.h:306-308）：
- `DEAD_ARG`（bit 4）：该参数在此 op 后死亡
- `SYNC_ARG`（bit 0）：该参数需要同步到内存

---

## 11. 寄存器分配器

### 11.1 AArch64 主机寄存器分配顺序

```c
// tcg/aarch64/tcg-target.c.inc:47-72
static const int tcg_target_reg_alloc_order[] = {
    // 优先使用 callee-saved（跨函数调用保持）
    X20, X21, X22, X23, X24, X25, X26, X27, X28,

    // 其次使用 caller-saved（函数调用后失效）
    X8, X9, X10, X11, X12, X13, X14, X15,
    X0, X1, X2, X3, X4, X5, X6, X7,

    // 保留寄存器（不参与分配）
    // X16/X17: TCG 临时寄存器
    // X18: 平台保留
    // X19: TCG_AREG0 (env 指针)
    // X29: FP, X30: LR

    // 向量寄存器
    V0-V7, V16-V31  // (V8-V15 是 callee-saved，跳过)
};
```

### 11.2 寄存器分配核心函数

```c
// tcg/tcg.c:4645-4705  单寄存器分配
static TCGReg tcg_reg_alloc(TCGContext *s, TCGRegSet required_regs,
                            TCGRegSet allocated_regs,
                            TCGRegSet preferred_regs, bool rev)
{
    // 1. 尝试从偏好寄存器中找空闲的 (4660-4670)
    // 2. 尝试从必需寄存器中找空闲的 (4672-4682)
    // 3. 所有必需寄存器都被占用 → 溢出一个到栈 (4686-4700)
    tcg_reg_free(s, reg, allocated_regs);  // 溢出
    return reg;
}

// tcg/tcg.c:4586-4624  溢出/同步
static void temp_sync(TCGContext *s, TCGTemp *ts, ...)
{
    // 如果 temp 不在内存中 → store 到栈帧
    if (!ts->mem_coherent) {
        tcg_out_st(s, ts->type, ts->reg, ts->mem_base->reg, ts->mem_offset);
        ts->mem_coherent = 1;
    }
    // 如果需要释放寄存器
    if (free_or_dead) {
        s->reg_to_temp[ts->reg] = NULL;
        ts->val_type = TS_MEM;
    }
}
```

### 11.3 Op 寄存器分配调度

```c
// tcg/tcg.c:5133-5799（概要）
// 遍历 ops 链表，对每个 op：
//   1. 读取 op->life 活性信息
//   2. 根据约束为输入/输出 temp 分配物理寄存器
//   3. 处理特殊 op（mov/call/goto_tb/exit_tb/qemu_ld/qemu_st）
//   4. 调用后端 tcg_out_* 发射 host 指令

// 特殊处理路径：
// tcg/tcg.c:4923-5018  tcg_reg_alloc_mov — mov 优化（可能消除）
// tcg/tcg.c:5874-6003  tcg_reg_alloc_call — 函数调用（保存/恢复）
```

---

## 12. AArch64 后端代码发射

### 12.1 后端引入方式

```c
// tcg/tcg.c:1098
#include "tcg-target.c.inc"  // 编译时包含目标后端
```

### 12.2 核心发射函数

| 函数 | 行号 | 功能 |
|------|------|------|
| `tcg_out_mov` | 1231-1264 | 寄存器间传送（可 32/64 位，可跨 GPR/VEC） |
| `tcg_out_movi` | 1104-1193 | 加载立即数（MOVZ/MOVK 序列，或 PC-relative） |
| `tcg_out_ld` | 1266-1290 | 从内存加载（LDR imm，可能分解为 ADD+LDR） |
| `tcg_out_st` | 1292-1322 | 存储到内存（STR imm） |
| `tcg_out_call` | 1410-1414 | 函数调用（BL 或间接 BLR） |
| `tcg_out_exit_tb` | 1991-2016 | TB 退出（返回值→X0，跳转 epilogue） |
| `tcg_out_goto_tb` | 2018-2033 | TB 直接链接（B 指令，可被补丁修改） |
| `tcg_out_goto_ptr` | 2035-2038 | 间接 TB 跳转（BR reg） |

### 12.3 指令编码层

```c
// tcg/aarch64/tcg-target.c.inc:661-905
// 底层编码辅助函数，将 TCG 操作映射为 AArch64 指令：
// tcg_out_insn(s, addsub_imm, ADDI, ...)   → ADD Xd, Xn, #imm
// tcg_out_insn(s, addsub_shift, ADD, ...)   → ADD Xd, Xn, Xm{, shift}
// tcg_out_insn(s, logic_imm, ANDI, ...)     → AND Xd, Xn, #imm
// tcg_out_insn(s, ldst_imm, LDRX, ...)      → LDR Xd, [Xn, #imm]
// tcg_out_insn(s, branch, B, ...)           → B offset
// tcg_out_insn(s, bcond_imm, B_C, ...)      → B.cond offset
// tcg_out_insn(s, bcond_reg, BR/BLR, ...)   → BR/BLR Xn
```

---

## 13. Guest 内存访问翻译

### 13.1 prepare_host_addr — TLB 内联查找

```c
// tcg/aarch64/tcg-target.c.inc:1650-1757
static TCGLabelQemuLdst *prepare_host_addr(TCGContext *s, HostAddress *h,
                                           TCGReg addr_reg, MemOpIdx oi,
                                           bool is_ld)
{
    if (tcg_use_softmmu) {
        // ===== SOFTMMU 路径（系统模式）=====

        // 1. 加载 TLB mask+table 指针（LDP 一次读两个 64 位）(1680-1681)
        //    CPUTLBDescFast.{mask, table} 在 AREG0 的负偏移处

        // 2. 从 guest 地址提取 TLB 索引（AND+LSR）(1684-1686)
        //    index = (addr >> PAGE_BITS) & mask

        // 3. 加上 table 基址得到 CPUTLBEntry 地址（ADD）(1689-1690)

        // 4. 加载 TLB 比较值和快速路径偏移（两次 LDR）(1694-1698)
        //    addr_read/addr_write + addend

        // 5. 页对齐比较（AND + CMP）(1716-1720)

        // 6. 不匹配 → 跳转慢路径（B.NE）(1723-1724)
        ldst->label_ptr[0] = s->code_ptr;  // 记录补丁点

        // 7. 匹配 → h->base = addend, h->index = addr_reg (1726-1728)

    } else {
        // ===== USER 模式 =====
        // 仅做对齐检查（TST + B.NE）(1730-1743)
        // 使用 guest_base 寄存器作为基址 (1746-1754)
    }
}
```

### 13.2 快速路径 — 直接加载/存储

```c
// tcg/aarch64/tcg-target.c.inc:1760-1819
static void tcg_out_qemu_ld_direct(TCGContext *s, MemOp memop, TCGType ext,
                                   TCGReg data_r, HostAddress h)
{
    // 根据 memop 大小选择 AArch64 load 指令：
    // MO_UB → LDRB    MO_SB → LDRSB
    // MO_UW → LDRH    MO_SW → LDRSH
    // MO_UL → LDRW    MO_SL → LDRSW
    // MO_UQ → LDR(X)
    // 使用 h.base + h.index 寻址
}
```

### 13.3 慢路径 — Helper 函数调用

```c
// tcg/aarch64/tcg-target.c.inc:1612-1638
// TLB miss 时跳转到此处：
//   1. 加载 helper 参数（env, addr, oi, retaddr）
//   2. 调用 helper_ld*_mmu / helper_st*_mmu
//   3. 跳回快速路径之后继续执行
```

---

## 14. TB 链接与运行时补丁

### 14.1 tcg_out_goto_tb — 初始生成

```c
// tcg/aarch64/tcg-target.c.inc:2018-2033
static void tcg_out_goto_tb(TCGContext *s, int which)
{
    // 1. 计算间接加载偏移并断言在范围内 (2025-2026)
    // 2. 记录跳转指令位置 (2028)
    set_jmp_insn_offset(s, which);
    // 3. 初始发射一条 B 指令（目标为自身，将被补丁）(2029)
    tcg_out32(s, Ibranch_B);
    // 4. 紧跟一条 BR TMP0（作为间接跳转备选）(2030)
    tcg_out_insn(s, bcond_reg, BR, TCG_REG_TMP0);
    // 5. 记录重置偏移 (2031)
    set_jmp_reset_offset(s, which);
    // 6. BTI 标记 (2032)
    tcg_out_bti(s, BTI_J);
}
```

### 14.2 tb_target_set_jmp_target — 运行时补丁

```c
// tcg/aarch64/tcg-target.c.inc:2040-2059
void tb_target_set_jmp_target(const TranslationBlock *tb, int n,
                              uintptr_t jmp_rx, uintptr_t jmp_rw)
{
    uintptr_t d_addr = tb->jmp_target_addr[n];
    ptrdiff_t d_offset = d_addr - jmp_rx;

    if (d_offset == sextract64(d_offset, 0, 28)) {
        // 直接分支：修改 B 指令的偏移 (2048-2049)
        insn = deposit32(Ibranch_B, 0, 26, d_offset >> 2);
    } else {
        // 间接分支：修改为 LDR TMP0, [PC+offset] (2051-2055)
        insn = deposit32(Ildlit_LDR | TCG_REG_TMP0, 5, 19, i_offset >> 2);
    }
    qatomic_set((uint32_t *)jmp_rw, insn);   // 2057: 原子写入
    flush_idcache_range(jmp_rx, jmp_rw, 4);   // 2058: 刷新 I/D cache
}
```

**补丁策略**：
- ±128MB 范围内 → 直接 B 跳转（最快，1 条指令）
- 超出范围 → LDR X16, [PC+off] 加载目标地址 → BR X16 间接跳转

---

## 15. Prologue 与 Epilogue

### 15.1 TCG 执行入口 Prologue

```c
// tcg/aarch64/tcg-target.c.inc:3467-3534
static void tcg_target_qemu_prologue(TCGContext *s)
{
    // 1. BTI 标记 (3471)
    tcg_out_bti(s, BTI_C);

    // 2. 保存 FP/LR，分配栈空间 (3474-3475)
    STP FP, LR, [SP, #-PUSH_SIZE]!

    // 3. 设置帧指针 (3478)
    MOV FP, SP

    // 4. 保存 callee-saved 寄存器 X19-X28 (3481-3484)
    for (X19..X27 步进 2): STP Xn, Xn+1, [SP, #ofs]

    // 5. 分配 TCG 局部变量空间 (3487-3488)
    SUB SP, SP, #(FRAME_SIZE - PUSH_SIZE)

    // 6. 设置 TCG 帧信息 (3491-3492)
    tcg_set_frame(s, SP, CALL_ARGS_SIZE, TEMP_BUF_SIZE)

    // 7. user 模式：加载 guest_base (3494-3503)
    if (!softmmu) MOV X28, guest_base

    // 8. 设置 env 指针并跳转到 TB 代码 (3505-3506)
    MOV X19(AREG0), X0(arg0)   // env = 第一个参数
    BR X1(arg1)                 // 跳转到 TB 代码

    // ===== Epilogue 部分 =====

    // 9. goto_ptr 返回点（返回值=0）(3512-3514)
    tcg_code_gen_epilogue:
    MOV X0, #0

    // 10. TB 正常返回点 (3517-3518)
    tb_ret_addr:
    BTI J

    // 11. 恢复栈和 callee-saved 寄存器 (3520-3528)
    ADD SP, SP, #(FRAME_SIZE - PUSH_SIZE)
    for (X19..X27): LDP Xn, Xn+1, [SP, #ofs]

    // 12. 恢复 FP/LR 并返回 (3530-3533)
    LDP FP, LR, [SP], #PUSH_SIZE
    RET
}
```

### 15.2 执行流程

```
cpu_exec_loop:
  → tcg_qemu_tb_exec(env, tb_ptr)
    → Prologue: 保存寄存器，设置 AREG0=env
    → BR tb_ptr  (跳转到 TB 代码)
    → TB 代码执行...
    → exit_tb: MOV X0, retval; B tb_ret_addr
    → Epilogue: 恢复寄存器，RET
  ← 返回值 = TB 退出信息
```

---

## 16. 代码缓冲区管理

### 16.1 初始化

```c
// tcg/tcg.c:1817-1821
void tcg_init(size_t tb_size, int splitwx, unsigned max_threads)
{
    tcg_context_init(max_threads);  // 1819: 初始化 TCGContext + 寄存器分配表
    tcg_region_init(tb_size, splitwx, max_threads);  // 1820: 分配代码缓冲区
}

// tcg/tcg.c:1762-1814  tcg_context_init()
// 1. 初始化 helper 调用布局 (1771-1776)
// 2. 调用 tcg_target_init (1778): 后端初始化
// 3. 构建 indirect_reg_alloc_order (1783-1794): callee-saved 逆序
// 4. 注册 TCG_AREG0 为 env 全局变量 (1813-1814)
```

### 16.2 TB 分配与水位检查

```c
// tcg/tcg.c:1827-1845
TranslationBlock *tcg_tb_alloc(TCGContext *s)
{
    // 检查是否超过高水位线 (1836-1840)
    if (unlikely(s->code_gen_ptr > s->code_gen_highwater)) {
        return NULL;  // 触发 tb_flush
    }
    // 对齐分配 TB 结构 + 代码空间
}
```

### 16.3 TB 刷新

```c
// accel/tcg/translate-all.c:543-639
// tb_flush: 清空所有翻译缓存
//   1. 序列化所有 vCPU
//   2. 重置代码缓冲区指针
//   3. 清空 TB 查找表
//   4. 释放所有 TB 元数据
```

---

## 17. 完整流水线总结

### 翻译一个 TB 的端到端流程

```
┌─────────────────────────────────────────────────────────┐
│  CPU 执行循环需要新 TB                                    │
│  cpu_exec → tb_gen_code(cpu, state)                     │
└───────────────┬─────────────────────────────────────────┘
                │
    ┌───────────▼───────────┐
    │  第1阶段：前端翻译       │
    │  translator_loop()      │
    │                        │
    │  ┌──────────────────┐  │
    │  │ init_disas_context│  │  从 hflags 提取 EL/MMU/功能标志
    │  └────────┬─────────┘  │
    │  ┌────────▼─────────┐  │
    │  │ while(DISAS_NEXT) │  │
    │  │  insn_start()     │  │  标记指令边界
    │  │  translate_insn() │  │  解码 ARM64 → 生成 TCG ops
    │  │  ├─ decodetree    │  │  a64.decode → trans_*()
    │  │  └─ tcg_gen_*()   │  │  tcg_gen_add_i64 等
    │  └────────┬─────────┘  │
    │  ┌────────▼─────────┐  │
    │  │ tb_stop()         │  │  生成 exit_tb/goto_tb/goto_ptr
    │  └──────────────────┘  │
    └───────────┬────────────┘
                │  TCG IR ops 链表
    ┌───────────▼───────────┐
    │  第2阶段：优化           │
    │  tcg_optimize()         │
    │                        │
    │  • 拷贝传播             │  copy_propagate
    │  • 常量折叠             │  fold_add/sub/and/...
    │  • 条件折叠             │  fold_brcond/setcond
    │  • 掩码追踪             │  z_mask/o_mask/s_mask
    │  • 死代码消除           │  tcg_op_remove
    └───────────┬────────────┘
                │  优化后 TCG IR
    ┌───────────▼───────────┐
    │  第3阶段：活性分析       │
    │  liveness_pass_1()      │
    │                        │
    │  反向遍历 ops：          │
    │  • 标记每个参数 DEAD/SYNC│
    │  • 删除无用 op           │
    │  • 计算寄存器偏好         │
    └───────────┬────────────┘
                │  带活性标注的 TCG IR
    ┌───────────▼───────────┐
    │  第4阶段：寄存器分配+发射 │
    │  tcg_gen_code()         │
    │                        │
    │  正向遍历 ops：          │
    │  ┌──────────────────┐  │
    │  │ tcg_reg_alloc_op  │  │  为每个 op 分配物理寄存器
    │  │  ├─ 加载输入 temp  │  │  temp → host reg (可能溢出)
    │  │  ├─ tcg_out_*(op)  │  │  发射 AArch64 指令
    │  │  └─ 释放死亡 temp  │  │  标记寄存器可用
    │  └──────────────────┘  │
    │                        │
    │  特殊 op：              │
    │  • goto_tb → B + BTI   │
    │  • exit_tb → MOV+B     │
    │  • qemu_ld → TLB 内联  │
    └───────────┬────────────┘
                │  AArch64 机器码
    ┌───────────▼───────────┐
    │  代码缓冲区              │
    │  code_gen_buffer        │
    │                        │
    │  [TB1 code][TB2 code]  │
    │  ← code_gen_ptr        │
    │  超过 highwater → flush │
    └─────────────────────────┘
```

### 关键数据流

```
Guest ARM64 指令
  ↓ (4 字节 fetch)
decodetree decode → trans_ADD_i()
  ↓
tcg_gen_add_i64(dst, src1, src2)  → TCGOp{opc=add, args=[dst,src1,src2]}
  ↓
tcg_optimize: fold_add → 常量折叠 / 拷贝传播
  ↓
liveness_pass_1: 标记 dst DEAD/SYNC
  ↓
tcg_reg_alloc_op: dst→X20, src1→X21, src2→X22
  ↓
tcg_out_insn(s, addsub_shift, ADD, 1, X20, X21, X22)
  ↓
AArch64 机器码: 0x8b160295  (ADD X20, X21, X22)
```

---

## 交叉参考

- [41-ARM64-EL切换TCG翻译变化深度分析](41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md) — hflags/TB 键/EL 切换
- [37-ARM64-MTTCG多线程执行深度分析](37-ARM64-MTTCG多线程执行深度分析-原子操作-内存屏障-TB缓存-中断处理与安全内存序.md) — MTTCG 并发 TB 执行
- [00-ARM64-CPU-GICv3-TCG深度分析](00-ARM64-CPU-GICv3-TCG深度分析.md) — TCG 基础架构

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
