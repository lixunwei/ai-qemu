# TCG 后端代码生成深度分析 — AArch64 后端、寄存器分配与 TLB 慢路径

> 基于 QEMU 11.0.50 源码分析，聚焦 TCG 后端代码生成管线：
> tcg_gen_code() 主循环、寄存器分配（tcg_reg_alloc_op）、outop 表驱动架构、
> AArch64 指令编码与发射、TLB 快慢路径内联代码生成、goto_tb TB 链接与补丁、
> Prologue/Epilogue 生成、条件码映射与分支优化。

---

## 目录

1. [后端代码生成全景](#1-后端代码生成全景)
2. [tcg_gen_code 主循环](#2-tcg_gen_code-主循环)
3. [寄存器分配 — tcg_reg_alloc_op](#3-寄存器分配--tcg_reg_alloc_op)
4. [AArch64 寄存器布局与约定](#4-aarch64-寄存器布局与约定)
5. [outop 表驱动架构](#5-outop-表驱动架构)
6. [AArch64 指令编码与发射](#6-aarch64-指令编码与发射)
7. [立即数加载策略 — tcg_out_movi](#7-立即数加载策略--tcg_out_movi)
8. [TLB 快路径与慢路径](#8-tlb-快路径与慢路径)
9. [goto_tb TB 链接与补丁](#9-goto_tb-tb-链接与补丁)
10. [Prologue 与 Epilogue](#10-prologue-与-epilogue)
11. [条件码映射与分支优化](#11-条件码映射与分支优化)
12. [总结](#12-总结)

---

## 1. 后端代码生成全景

TCG 后端代码生成负责将前端优化后的 TCG IR（Op 链表）转换为 Host 机器码：

```
优化后 TCG Op 链表
     │
     ▼
┌────────────────────────────────────────────────┐
│  tcg_gen_code() — 后端驱动                      │
│  tcg/tcg.c:6556-6740                            │
│                                                  │
│  阶段1: 优化 Pass（前文已述）                     │
│  阶段2: tcg_reg_alloc_start() — 初始化寄存器状态 │
│  阶段3: 遍历 Op 链表 → 分发                     │
│    ├── mov/dup → tcg_reg_alloc_mov/dup           │
│    ├── call → tcg_reg_alloc_call                 │
│    ├── exit_tb → tcg_out_exit_tb                 │
│    ├── goto_tb → tcg_out_goto_tb                 │
│    ├── br → tcg_out_br                           │
│    ├── mb → tcg_out_mb                           │
│    └── default → tcg_reg_alloc_op                │
│         └── outop_xxx.out_rrr/out_rri            │
│              └── tcg_out_insn() → 机器码          │
│  阶段4: tcg_out_ldst_finalize() — 发射慢路径     │
│  阶段5: tcg_out_pool_finalize() — 常量池         │
└────────────────────────────────────────────────┘
```

---

## 2. tcg_gen_code 主循环

```c
// tcg/tcg.c:6634-6728
tcg_reg_alloc_start(s);                            // :6634 — 初始化寄存器状态

s->code_buf = tcg_splitwx_to_rw(tb->tc.ptr);       // :6641
s->code_ptr = s->code_buf;                          // :6642

QTAILQ_FOREACH(op, &s->ops, link) {                 // :6655
    switch (opc) {
    case INDEX_op_mov:
    case INDEX_op_mov_vec:
        tcg_reg_alloc_mov(s, op);                   // :6676
        break;
    case INDEX_op_dup_vec:
        tcg_reg_alloc_dup(s, op);                   // :6679
        break;
    case INDEX_op_insn_start:
        // 记录 Guest→Host PC 映射                  // :6681-6693
        break;
    case INDEX_op_discard:
        temp_dead(s, arg_temp(op->args[0]));         // :6696
        break;
    case INDEX_op_set_label:
        tcg_reg_alloc_bb_end(s, s->reserved_regs);   // :6699
        tcg_out_label(s, arg_label(op->args[0]));
        break;
    case INDEX_op_call:
        tcg_reg_alloc_call(s, op);                   // :6704
        break;
    case INDEX_op_exit_tb:
        tcg_out_exit_tb(s, op->args[0]);             // :6707
        break;
    case INDEX_op_goto_tb:
        tcg_out_goto_tb(s, op->args[0]);             // :6710
        break;
    case INDEX_op_br:
        tcg_out_br(s, arg_label(op->args[0]));       // :6713
        break;
    case INDEX_op_mb:
        tcg_out_mb(s, op->args[0]);                  // :6716
        break;
    default:
        tcg_reg_alloc_op(s, op);                     // :6726
        break;
    }
}
```

**set_label 处的 BB 边界**：`tcg_reg_alloc_bb_end()` 在基本块边界将所有 `TEMP_TB` 类型的临时变量同步回内存（`CPUARMState`），确保跨基本块的活性安全。

---

## 3. 寄存器分配 — tcg_reg_alloc_op

```c
// tcg/tcg.c:5133-5803
static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op) {
    // === 输入操作数分配（:5185-5390）===
    for (k = 0; k < nb_iargs; k++) {
        // 常量操作数：检查后端是否接受内联常量
        if (ts->val_type == TEMP_VAL_CONST) {
            if (tcg_target_const_match(ts->val, ...)) {
                const_args[i] = 1;                   // :5207-5210
                new_args[i] = ts->val;
                continue;
            }
        }
        // 非常量：确保在物理寄存器中
        // 可能需要 temp_load() 从内存加载
        // 根据约束选择合适的寄存器
    }

    // === 输出操作数分配（:5419-5479）===
    for (k = 0; k < nb_oargs; k++) {
        // 分配输出寄存器，考虑 output_pref
        // DEAD_ARG 的输入寄存器可复用
    }

    // === 发射指令（:5481-5779）===
    // 根据操作码类型调用对应的 outop 表函数
    // outop_add.out_rrr(s, type, a0, a1, a2);
}
```

### 关键辅助函数

| 函数 | 位置 | 作用 |
|------|------|------|
| `temp_load()` | tcg.c:4755-4803 | 将临时变量从内存/常量加载到物理寄存器 |
| `temp_save()` | tcg.c:4807-4812 | 将寄存器值保存回 `CPUARMState` 内存 |
| `tcg_reg_alloc_bb_end()` | tcg.c:4843-4865 | BB 边界同步所有 TB-scope temps |
| `tcg_reg_alloc_start()` | tcg.c（:6634 调用） | 初始化所有 temp 的寄存器分配状态 |

---

## 4. AArch64 寄存器布局与约定

### 寄存器角色

```c
// tcg/aarch64/tcg-target.c.inc:86-91
#define TCG_REG_TMP0     TCG_REG_X16   // 后端临时寄存器 0
#define TCG_REG_TMP1     TCG_REG_X17   // 后端临时寄存器 1
#define TCG_REG_TMP2     TCG_REG_X30   // 后端临时寄存器 2（LR 复用）
#define TCG_VEC_TMP0     TCG_REG_V31   // 向量临时寄存器
#define TCG_REG_GUEST_BASE TCG_REG_X28 // guest_base 指针
```

### 分配优先级

```c
// tcg/aarch64/tcg-target.c.inc:47-72
static const int tcg_target_reg_alloc_order[] = {
    // 被调用者保存（callee-saved）— 优先分配，无需跨 helper 调用保存
    X20, X21, X22, X23, X24, X25, X26, X27, X28,

    // 调用者保存（caller-saved）— 跨 helper 调用需保存/恢复
    X8, X9, X10, X11, X12, X13, X14, X15,

    // 参数寄存器 — 最后分配（helper 调用时会被覆盖）
    X0, X1, X2, X3, X4, X5, X6, X7,

    // 保留不可分配：
    // X16 — TCG_REG_TMP0
    // X17 — TCG_REG_TMP1
    // X18 — 平台保留
    // X19 — AREG0（CPUARMState *env 指针）
    // X29 — FP（帧指针）
    // X30 — TCG_REG_TMP2

    // 向量寄存器
    V0-V7, V16-V31,   // V8-V15 被调用者保存，跳过
};
```

### 函数调用约定

```c
// :74-84
static const int tcg_target_call_iarg_regs[8] = {
    X0, X1, X2, X3, X4, X5, X6, X7      // 参数寄存器
};
// 返回值：X0（或 X0+X1 对于 128 位）
```

---

## 5. outop 表驱动架构

QEMU 11.0 使用 **outop 表驱动**架构替代传统的 `tcg_out_op()` 大 switch：

```c
// tcg/aarch64/tcg-target.c.inc:2078-2082
// 每个 TCG 操作码对应一个 outop 结构体
static const TCGOutOpBinary outop_add = {
    .base.static_constraint = C_O1_I2(r, r, rA),  // 1输出 2输入，约束
    .out_rrr = tgen_add,     // 寄存器-寄存器-寄存器 发射
    .out_rri = tgen_addi,    // 寄存器-寄存器-立即数 发射
};

// 实际发射函数
static void tgen_add(TCGContext *s, TCGType type,
                     TCGReg a0, TCGReg a1, TCGReg a2) {
    tcg_out_insn(s, addsub_shift, ADD, type, a0, a1, a2);  // :2065
}

static void tgen_addi(TCGContext *s, TCGType type,
                      TCGReg a0, TCGReg a1, tcg_target_long a2) {
    if (a2 >= 0)
        tcg_out_insn(s, addsub_imm, ADDI, type, a0, a1, a2);
    else
        tcg_out_insn(s, addsub_imm, SUBI, type, a0, a1, -a2);  // :2071-2075
}
```

### 约束字符含义

| 字符 | 含义 |
|------|------|
| `r` | 通用寄存器 |
| `rA` | 寄存器 或 12 位加法立即数（AIMM） |
| `rL` | 寄存器 或 逻辑立即数（LIMM） |
| `rz` | 寄存器 或 零（XZR） |
| `w` | 向量寄存器 |

### 常见 outop 示例

```c
// AND — 逻辑立即数约束
static const TCGOutOpBinary outop_and = {
    .base.static_constraint = C_O1_I2(r, r, rL),     // :2173-2174
    .out_rrr = tgen_and,      // AND Xd, Xn, Xm      // :2175
    .out_rri = tgen_andi,     // AND Xd, Xn, #imm     // :2176
};

// qemu_ld — 内存加载
static const TCGOutOpQemuLdSt outop_qemu_ld = {
    .base.static_constraint = C_O1_I1(r, r),          // :1839
    .out = tgen_qemu_ld,                               // :1840
};

// qemu_st — 内存存储
static const TCGOutOpQemuLdSt outop_qemu_st = {
    .base.static_constraint = C_O0_I2(rz, r),         // :1860
    .out = tgen_qemu_st,                               // :1861
};
```

---

## 6. AArch64 指令编码与发射

### AArch64Insn 枚举

```c
// tcg/aarch64/tcg-target.c.inc:395-653
typedef enum {
    // 分支指令
    Ibranch_B     = 0x14000000,   // B <offset>
    Ibranch_BL    = 0x94000000,   // BL <offset>
    Ibcond_reg_BR = 0xd61f0000,   // BR Xn
    Ibcond_reg_BLR= 0xd63f0000,   // BLR Xn
    Ibcond_reg_RET= 0xd65f0000,   // RET

    // 加减法
    Iaddsub_shift_ADD  = 0x0b000000,
    Iaddsub_shift_ADDS = 0x2b000000,
    Iaddsub_shift_SUB  = 0x4b000000,
    Iaddsub_shift_SUBS = 0x6b000000,

    // 逻辑运算
    Ilogic_shift_AND = 0x0a000000,
    Ilogic_shift_ORR = 0x2a000000,
    Ilogic_shift_EOR = 0x4a000000,
    Ilogic_shift_BIC = 0x0a200000,

    // Load/Store
    Ildst_imm_LDRB  = 0x39400000,
    Ildst_imm_LDRX  = 0xf9400000,
    Ildst_imm_STRX  = 0xf9000000,

    // 系统指令
    NOP     = 0xd503201f,
    DMB_ISH = 0xd50338bf,
    BTI_C   = 0xd503245f,
    // ... 200+ 条目
} AArch64Insn;
```

### tcg_out_insn 宏 — 类型安全发射

```c
// :661-663
#define tcg_out_insn(S, FMT, OP, ...) \
    glue(tcg_out_insn_,FMT)(S, glue(glue(glue(I,FMT),_),OP), ## __VA_ARGS__)

// 用法：
tcg_out_insn(s, addsub_shift, ADD, type, rd, rn, rm);
// 展开为：
tcg_out_insn_addsub_shift(s, Iaddsub_shift_ADD, type, rd, rn, rm);
```

### tcg_out_mov — 寄存器移动

```c
// :1231-1264
static bool tcg_out_mov(TCGContext *s, TCGType type, TCGReg ret, TCGReg arg)
{
    if (ret == arg) return true;              // 无需移动

    if (ret < 32 && arg < 32) {
        tcg_out_movr(s, type, ret, arg);      // MOV Xd, Xm (整数)
    } else if (ret < 32) {
        tcg_out_insn(s, simd_copy, UMOV, ...); // SIMD→整数
    } else if (arg < 32) {
        tcg_out_insn(s, simd_copy, INS, ...);  // 整数→SIMD
    } else {
        tcg_out_insn(s, qrrr_e, ORR, ...);     // SIMD→SIMD (ORR v,v,v)
    }
}
```

### 32 位 vs 64 位

SF（Size Flag）位由 `TCGType` 直接映射：

```c
// :28-31
// TCG_TYPE_I32 == 0, TCG_TYPE_I64 == 1
// 直接用作 AArch64 指令的 sf 位（bit 31）
QEMU_BUILD_BUG_ON(TCG_TYPE_I32 != 0 || TCG_TYPE_I64 != 1);
```

---

## 7. 立即数加载策略 — tcg_out_movi

```c
// tcg/aarch64/tcg-target.c.inc:1104-1193
static void tcg_out_movi(TCGContext *s, TCGType type, TCGReg rd,
                         tcg_target_long value) {
    // 策略1: 16 位值 → MOVZ（1 条指令）
    if ((value & ~0xffffull) == 0) {
        tcg_out_insn(s, movw, MOVZ, type, rd, value, 0);        // :1136
        return;
    }
    // 策略2: 取反16位 → MOVN（1 条指令）
    if ((ivalue & ~0xffffull) == 0) {
        tcg_out_insn(s, movw, MOVN, type, rd, ivalue, 0);       // :1139
        return;
    }

    // 策略3: 位域立即数 → ORR XZR,#imm（1 条指令）
    if (is_limm(svalue)) {
        tcg_out_logicali(s, Ilogic_imm_ORRI, type, rd, XZR, svalue); // :1147
        return;
    }

    // 策略4: PC 相对 ±1MB → ADR（1 条指令）
    if (disp == sextract64(disp, 0, 21)) {
        tcg_out_insn(s, pcrel, ADR, rd, disp);                  // :1157
        return;
    }

    // 策略5: PC 相对 ±4GB → ADRP + ADD（2 条指令）
    if (page_disp == sextract64(page_disp, 0, 21)) {
        tcg_out_insn(s, pcrel, ADRP, rd, page_disp);
        if (value & 0xfff)
            tcg_out_insn(s, addsub_imm, ADDI, type, rd, rd, value & 0xfff);
        return;                                                   // :1162-1167
    }

    // 策略6: MOVZ + MOVK（2 条指令）
    if (t2 == 0) {
        tcg_out_insn_movw(s, opc, type, rd, t0 >> s0, s0);
        if (t1 != 0)
            tcg_out_insn(s, movw, MOVK, type, rd, value >> s1, s1);
        return;                                                   // :1182-1187
    }

    // 策略7: 常量池加载 → LDR（超过 2 条时回退）
    new_pool_label(s, value, R_AARCH64_CONDBR19, s->code_ptr, 0);
    tcg_out_insn(s, ldlit, LDR, 0, rd);                          // :1191-1192
}
```

**设计理念**：优先使用最少指令数加载立即数，从 1 条到常量池加载逐级回退。

---

## 8. TLB 快路径与慢路径

### 内存访问整体流程

```
tgen_qemu_ld(s, type, data_reg, addr_reg, oi)
    │
    ├── prepare_host_addr()        ← TLB 查找（快路径）
    │   ├── LDP mask,table         ← 加载 TLB 描述符
    │   ├── AND_LSR → TLB index    ← 计算索引
    │   ├── ADD → TLB entry addr   ← 定位条目
    │   ├── LDR comparator         ← 加载比较值
    │   ├── LDR addend             ← 加载 host 地址偏移
    │   ├── AND addr, page_mask    ← 提取页面地址
    │   ├── CMP comparator         ← 比较
    │   └── B.NE → slow_path      ← 不匹配跳转慢路径
    │
    ├── tcg_out_qemu_ld_direct()   ← Host 内存访问（快路径命中）
    │   └── LDR/LDRB/LDRH/...     ← 根据 MemOp 选择
    │
    └── [延迟] tcg_out_qemu_ld_slow_path()  ← 慢路径
        ├── 设置参数（env, addr, oi, retaddr）
        ├── CALL helper_ld{b,w,l,q}_mmu
        └── B → raddr（返回快路径后面）
```

### prepare_host_addr — TLB 快路径

```c
// tcg/aarch64/tcg-target.c.inc:1650-1757
static TCGLabelQemuLdst *prepare_host_addr(TCGContext *s, HostAddress *h,
                                           TCGReg addr_reg, MemOpIdx oi,
                                           bool is_ld) {
    if (tcg_use_softmmu) {
        // 1. 加载 TLB 描述符（mask + table 指针）
        tcg_out_insn(s, ldstpair, LDP, TMP0, TMP1, AREG0,
                     tlb_mask_table_ofs(s, mem_index), ...);  // :1680-1681

        // 2. 计算 TLB 索引
        // TMP0 = (TMP0_mask & (addr >> (PAGE_BITS - TLB_ENTRY_BITS)))
        tcg_out_insn(s, addsub_realshift, AND_LSR, I64,
                     TMP0, TMP0, addr_reg,
                     TARGET_PAGE_BITS - CPU_TLB_ENTRY_BITS);  // :1684-1686

        // 3. TLB 条目地址 = table + index
        tcg_out_insn(s, addsub_shift, ADD, 1,
                     TMP1, TMP1, TMP0);                       // :1689-1690

        // 4. 加载比较值和 host 地址偏移
        tcg_out_ld(s, addr_type, TMP0, TMP1,
                   is_ld ? offsetof(CPUTLBEntry, addr_read)
                         : offsetof(CPUTLBEntry, addr_write)); // :1694-1696
        tcg_out_ld(s, TCG_TYPE_PTR, TMP1, TMP1,
                   offsetof(CPUTLBEntry, addend));              // :1697-1698

        // 5. 提取页面地址并比较
        tcg_out_logicali(s, ANDI, addr_type, TMP2,
                         addr_adj, compare_mask);               // :1716-1717
        tcg_out_cmp(s, addr_type, TCG_COND_NE,
                    TMP0, TMP2, 0);                             // :1720

        // 6. 不匹配 → 跳转慢路径
        ldst->label_ptr[0] = s->code_ptr;
        tcg_out_insn(s, bcond_imm, B_C, TCG_COND_NE, 0);      // :1723-1724

        // 命中：host_addr = TMP1 + addr_reg
        h->base = TMP1;
        h->index = addr_reg;                                    // :1726-1728
    }
}
```

### tcg_out_qemu_ld_direct — Host 内存访问

```c
// :1760-1795
static void tcg_out_qemu_ld_direct(TCGContext *s, MemOp memop, TCGType ext,
                                   TCGReg data_r, HostAddress h) {
    switch (memop & MO_SSIZE) {
    case MO_UB:  LDRB  data, [base, index]   break;
    case MO_SB:  LDRSB data, [base, index]   break;
    case MO_UW:  LDRH  data, [base, index]   break;
    case MO_SW:  LDRSH data, [base, index]   break;
    case MO_UL:  LDRW  data, [base, index]   break;
    case MO_SL:  LDRSW data, [base, index]   break;
    case MO_UQ:  LDR   data, [base, index]   break;
    }
}
```

### 慢路径 — tcg_out_qemu_ld_slow_path

```c
// :1612-1625
static bool tcg_out_qemu_ld_slow_path(TCGContext *s, TCGLabelQemuLdst *lb) {
    // 修补条件分支目标到当前位置
    reloc_pc19(lb->label_ptr[0], tcg_splitwx_to_rx(s->code_ptr));

    // 设置 helper 参数
    tcg_out_ld_helper_args(s, lb, &ldst_helper_param);

    // 调用 helper_ld{b,w,l,q}_mmu
    tcg_out_call_int(s, qemu_ld_helpers[opc & MO_SIZE]);

    // 获取返回值
    tcg_out_ld_helper_ret(s, lb, false, &ldst_helper_param);

    // 跳回快路径后面
    tcg_out_goto(s, lb->raddr);
}
```

慢路径代码在 TB 翻译结束后由 `tcg_out_ldst_finalize()` 统一发射（在代码末尾），保持快路径紧凑。

---

## 9. goto_tb TB 链接与补丁

### goto_tb 发射

```c
// tcg/aarch64/tcg-target.c.inc:2018-2033
static void tcg_out_goto_tb(TCGContext *s, int which) {
    // 确保 LDR 常量池偏移在范围内
    intptr_t i_off = tcg_pcrel_diff(s, (void *)get_jmp_target_addr(s, which));
    tcg_debug_assert(i_off == sextract64(i_off, 0, 21));

    set_jmp_insn_offset(s, which);      // 记录补丁位置
    tcg_out32(s, Ibranch_B);            // 占位: B <offset>（初始偏移 0）
    tcg_out_insn(s, bcond_reg, BR, TMP0); // 备用: BR X16（间接跳转）
    set_jmp_reset_offset(s, which);     // 记录 reset 位置
    tcg_out_bti(s, BTI_J);             // BTI 着陆垫
}
```

### TB 链接补丁

```c
// :2040-2058
void tb_target_set_jmp_target(const TranslationBlock *tb, int n,
                              uintptr_t jmp_rx, uintptr_t jmp_rw) {
    uintptr_t d_addr = tb->jmp_target_addr[n];
    ptrdiff_t d_offset = d_addr - jmp_rx;

    if (d_offset == sextract64(d_offset, 0, 28)) {
        // 直接分支：补丁为 B <target>（±128MB 范围）
        insn = deposit32(Ibranch_B, 0, 26, d_offset >> 2);
    } else {
        // 间接跳转：补丁为 LDR X16, [PC+off]
        // 从 tb->jmp_target_addr[n] 加载目标地址
        insn = deposit32(Ildlit_LDR | TCG_REG_TMP0, 5, 19, i_offset >> 2);
    }

    // 原子写入补丁指令 + 刷新缓存
    qatomic_set((uint32_t *)jmp_rw, insn);
    flush_idcache_range(jmp_rx, jmp_rw, 4);
}
```

**双模式设计**：优先使用直接分支（B 指令，±128MB），超出范围时回退到从内存加载地址的间接跳转。

### exit_tb — 返回主循环

```c
// :1991-2016
static void tcg_out_exit_tb(TCGContext *s, uintptr_t a0) {
    if (a0 == 0) {
        // goto_ptr 失败 → 跳转到 epilogue（零返回值）
        target = tcg_code_gen_epilogue;
    } else {
        // 正常退出 → 将返回值加载到 X0，跳转到 epilogue
        tcg_out_movi(s, TCG_TYPE_I64, X0, a0);
        target = tb_ret_addr;
    }
    tcg_out_insn(s, branch, B, offset);  // B → epilogue
}
```

---

## 10. Prologue 与 Epilogue

```c
// tcg/aarch64/tcg-target.c.inc:3467-3534
static void tcg_target_qemu_prologue(TCGContext *s)
{
    // === Prologue ===
    tcg_out_bti(s, BTI_C);                             // :3471

    // 保存 FP, LR
    tcg_out_insn(s, ldstpair, STP, FP, LR,
                 SP, -PUSH_SIZE, 1, 1);                // :3474-3475

    // 设置帧指针
    tcg_out_movr_sp(s, I64, FP, SP);                   // :3478

    // 保存 callee-saved 寄存器 X19..X28
    for (r = X19; r <= X27; r += 2) {
        tcg_out_insn(s, ldstpair, STP, r, r+1,
                     SP, ofs, 1, 0);                    // :3481-3483
    }

    // 分配栈帧
    tcg_out_insn(s, addsub_imm, SUBI, I64,
                 SP, SP, FRAME_SIZE - PUSH_SIZE);       // :3487-3488

    // 设置 guest_base
    if (!tcg_use_softmmu) {
        tcg_out_movi(s, TCG_TYPE_PTR,
                     TCG_REG_GUEST_BASE, guest_base);   // :3501
    }

    // 加载 env 指针 + 跳转到 TB 代码
    tcg_out_mov(s, TCG_TYPE_PTR, AREG0, X0);           // :3505
    tcg_out_insn(s, bcond_reg, BR, X1);                // :3506

    // === Epilogue ===
    // goto_ptr 返回点（返回值=0）
    tcg_code_gen_epilogue = ...;
    tcg_out_movi(s, TCG_TYPE_REG, X0, 0);              // :3514

    // 通用返回点
    tb_ret_addr = ...;
    tcg_out_insn(s, addsub_imm, ADDI, I64,
                 SP, SP, FRAME_SIZE - PUSH_SIZE);       // :3521-3522

    // 恢复 callee-saved 寄存器
    for (r = X19; r <= X27; r += 2) {
        tcg_out_insn(s, ldstpair, LDP, r, r+1, ...);   // :3525-3527
    }

    // 恢复 FP, LR + 返回
    tcg_out_insn(s, ldstpair, LDP, FP, LR,
                 SP, PUSH_SIZE, 0, 1);                  // :3531-3532
    tcg_out_insn(s, bcond_reg, RET, LR);               // :3533
}
```

### Prologue 执行流

```
Host 调用入口
    │
    ├── STP FP, LR, [SP, -PUSH_SIZE]!   ← 保存帧
    ├── MOV FP, SP                        ← 设置帧指针
    ├── STP X19..X28                      ← 保存 callee-saved
    ├── SUB SP, SP, FRAME_SIZE            ← 分配局部变量
    ├── MOV X28, guest_base               ← 加载 guest 基地址
    ├── MOV X19, X0                       ← env 指针到 AREG0
    └── BR X1                             ← 跳转到 TB 代码
         │
         ▼ （TB 执行后）
    ├── tb_ret_addr:
    ├── ADD SP, SP, FRAME_SIZE            ← 释放栈帧
    ├── LDP X19..X28                      ← 恢复寄存器
    ├── LDP FP, LR, [SP], PUSH_SIZE      ← 恢复帧
    └── RET                               ← 返回 Host
```

---

## 11. 条件码映射与分支优化

### TCGCond 直接映射

QEMU 定义的 `TCGCond` 值直接对应 AArch64 NZCV 条件码编码，无需转换表：

### 分支优化 — tgen_brcondi

```c
// tcg/aarch64/tcg-target.c.inc:1434-1508
static void tgen_brcondi(TCGContext *s, TCGType ext, TCGCond c,
                         TCGReg a, tcg_target_long b, TCGLabel *l) {
    // 优化1: cmp X,0; b.eq → CBZ X
    if (c == TCG_COND_EQ && b == 0) {
        tcg_out_insn(s, cbz, CBZ, ext, a, 0);          // :1499
    }
    // 优化2: cmp X,0; b.ne → CBNZ X
    else if (c == TCG_COND_NE && b == 0) {
        tcg_out_insn(s, cbz, CBNZ, ext, a, 0);         // :1501
    }
    // 优化3: cmp X,0; b.lt → TBNZ X,63（测试符号位）
    else if (c == TCG_COND_LT && b == 0) {
        tcg_out_insn(s, tbz, TBNZ, a, ext ? 63 : 31, 0); // :1490
    }
    // 优化4: tst X,1<<B; b.ne → TBNZ X,B（测试单 bit）
    else if (is_power_of_2(b)) {
        tcg_out_insn(s, tbz, TBNZ, a, ctz64(b), 0);    // :1490
    }
    // 默认: CMP + B.cond
    else {
        tgen_cmpi(s, ext, c, a, b);
        tcg_out_insn(s, bcond_imm, B_C, c, 0);         // :1478-1479
    }
}
```

### 重定位类型

| 重定位类型 | 范围 | 用途 |
|-----------|------|------|
| R_AARCH64_JUMP26 | ±128MB | 无条件分支（B/BL） |
| R_AARCH64_CONDBR19 | ±1MB | 条件分支（B.cond/CBZ/CBNZ） |
| R_AARCH64_TSTBR14 | ±32KB | 测试位分支（TBZ/TBNZ） |

```c
// :131-146
static bool patch_reloc(tcg_insn_unit *code_ptr, int type,
                        intptr_t value, intptr_t addend) {
    switch (type) {
    case R_AARCH64_JUMP26:  return reloc_pc26(code_ptr, value);
    case R_AARCH64_CONDBR19: return reloc_pc19(code_ptr, value);
    case R_AARCH64_TSTBR14: return reloc_pc14(code_ptr, value);
    }
}
```

---

## 12. 总结

### 后端代码生成关键路径

```
tcg_gen_code()                           [tcg.c:6556]
  ├── tcg_optimize()                     — 优化 IR
  ├── liveness_pass_0/1/2()              — 活性分析
  ├── tcg_reg_alloc_start()              — 初始化寄存器状态
  ├── FOREACH(op):                       — 遍历 Op
  │   ├── mov → tcg_reg_alloc_mov()      — 寄存器移动
  │   ├── call → tcg_reg_alloc_call()    — 函数调用
  │   ├── exit_tb → tcg_out_exit_tb()    — 返回主循环
  │   ├── goto_tb → tcg_out_goto_tb()    — TB 链接
  │   └── default → tcg_reg_alloc_op()   — 通用寄存器分配
  │       ├── 输入分配 + temp_load()      — 确保操作数在寄存器
  │       ├── 输出分配                    — 分配结果寄存器
  │       └── outop_xxx.out_rrr()        — 调用后端发射
  │           └── tcg_out_insn()         — 编码并写入机器码
  ├── tcg_out_ldst_finalize()            — 发射 TLB 慢路径
  └── tcg_out_pool_finalize()            — 发射常量池
```

### 设计亮点

1. **outop 表驱动**：每个 TCG 操作码对应一个 outop 结构体，包含约束信息和发射函数。添加新指令只需填表，无需修改 switch 分支。

2. **7 级立即数加载**：`tcg_out_movi` 从 MOVZ 到常量池加载逐级尝试，确保最少指令数。

3. **延迟慢路径**：TLB 慢路径代码在 TB 末尾统一发射，保持快路径代码连续紧凑，提升 I-cache 命中率。

4. **双模式 TB 链接**：goto_tb 同时准备直接分支和间接跳转，通过原子补丁实现 TB 链接。

5. **分支优化**：条件分支智能选择 CBZ/CBNZ/TBZ/TBNZ 替代 CMP+B.cond，减少指令数。

6. **callee-saved 优先分配**：寄存器分配器优先使用 callee-saved 寄存器（X20-X28），减少跨 helper 调用的保存/恢复开销。

---

**关键源文件**：
- `tcg/tcg.c` — `tcg_gen_code()` 后端驱动、`tcg_reg_alloc_op()` 寄存器分配
- `tcg/aarch64/tcg-target.c.inc` — AArch64 后端全部实现
- `tcg/aarch64/tcg-target.h` — 寄存器枚举与常量
- `include/tcg/tcg.h` — TCGOp/TCGTemp/TCGContext 定义
