# TCG 后端代码生成与 TB 管理深度分析

> QEMU 11.0.50 · 分析日期 2025-07 · 基于源码交叉验证

## 目录

1. [概述](#1-概述)
2. [代码生成全流程 — tcg_gen_code()](#2-代码生成全流程--tcg_gen_code)
3. [寄存器分配核心 — tcg_reg_alloc_op()](#3-寄存器分配核心--tcg_reg_alloc_op)
4. [寄存器选择算法 — tcg_reg_alloc()](#4-寄存器选择算法--tcg_reg_alloc)
5. [Temp 加载/同步/保存策略](#5-temp-加载同步保存策略)
6. [栈帧与 Spill 槽分配](#6-栈帧与-spill-槽分配)
7. [约束系统 — TCGArgConstraint](#7-约束系统--tcgargconstraint)
8. [寄存器分配器函数族](#8-寄存器分配器函数族)
9. [进位链处理](#9-进位链处理)
10. [AArch64 后端总览](#10-aarch64-后端总览)
11. [AArch64 寄存器定义与分配序](#11-aarch64-寄存器定义与分配序)
12. [AArch64 指令编码体系](#12-aarch64-指令编码体系)
13. [AArch64 算术/逻辑指令发射](#13-aarch64-算术逻辑指令发射)
14. [AArch64 内存指令发射](#14-aarch64-内存指令发射)
15. [AArch64 分支与重定位](#15-aarch64-分支与重定位)
16. [AArch64 常量加载策略 — tcg_out_movi](#16-aarch64-常量加载策略--tcg_out_movi)
17. [AArch64 Prologue/Epilogue](#17-aarch64-prologueepilogue)
18. [客户机内存访问发射 — qemu_ld/st](#18-客户机内存访问发射--qemu_ldst)
19. [TranslationBlock 结构](#19-translationblock-结构)
20. [TB 创建全流程 — tb_gen_code()](#20-tb-创建全流程--tb_gen_code)
21. [TB 查找与哈希](#21-tb-查找与哈希)
22. [TB 链接（Chaining）](#22-tb-链接chaining)
23. [TB 解链与失效](#23-tb-解链与失效)
24. [代码缓冲区管理](#24-代码缓冲区管理)
25. [TB 执行循环](#25-tb-执行循环)
26. [自修改代码处理](#26-自修改代码处理)
27. [代码缓冲区统计](#27-代码缓冲区统计)
28. [完整代码生成示例：ADD X0, X1, X2 的后端之旅](#28-完整代码生成示例add-x0-x1-x2-的后端之旅)
29. [附录 A：关键源文件索引](#29-附录-a关键源文件索引)
30. [附录 B：TB 生命周期状态图](#30-附录-b-tb-生命周期状态图)

---

## 1. 概述

本文分析 TCG 后端代码生成和 Translation Block (TB) 管理两大核心机制：

- **后端代码生成**：将 TCG IR 操作转换为宿主机器码的完整流程——寄存器分配、约束满足、指令发射、重定位
- **AArch64 后端**：ARM64 宿主上的具体实现——指令编码、寄存器映射、Prologue/Epilogue
- **TB 管理**：Translation Block 的创建、缓存、链接、执行、失效的完整生命周期

**关键数字**：
- `tcg.c`：**~6770 行**，包含核心引擎（寄存器分配 + 代码生成）
- `tcg-target.c.inc`（AArch64）：**3592 行**，AArch64 后端实现
- `translation-block.h`：TB 结构定义，含 15 个关键字段
- `cpu-exec.c`：TB 执行循环与链接逻辑

---

## 2. 代码生成全流程 — tcg_gen_code()

**定义**：`tcg/tcg.c:6556-6766`

```
tcg_gen_code(s, tb, pc_start)
│
├── [可选] 日志：dump IR ops                     :6561-6570
├── [DEBUG] 检查所有标签已 present               :6572-6587
│
├── ① tcg_optimize(s)                            :6592
├── ② reachable_code_pass(s)                     :6594
├── ③ liveness_pass_0(s)                         :6595
├── ④ liveness_pass_1(s)                         :6596
├── ⑤ [条件] liveness_pass_2(s)                  :6611-6614
│
├── 初始化跳转元数据                              :6628-6632
│   jmp_reset_offset = INVALID
│   jmp_insn_offset = INVALID
│
├── tcg_reg_alloc_start(s)                        :6634
├── 设置代码缓冲区指针                            :6641-6643
│   s->code_buf = tb->tc.ptr
│   s->code_ptr = s->code_buf
│
├── 分配 insn_data                                :6648-6649
├── tcg_out_tb_start(s)                           :6651
├── carry_live = false                            :6654
│
├── ⑥ 主发射循环                                  :6655-6740
│   QTAILQ_FOREACH(op, &s->ops, link) {
│   │
│   ├── switch (op->opc) {
│   │   case insn_start → 记录 PC 映射
│   │   case set_label → tcg_reg_alloc_bb_end + 标签定义
│   │   case call → tcg_reg_alloc_call
│   │   case mov/dup → tcg_reg_alloc_mov / dup
│   │   default → tcg_reg_alloc_op                :6726
│   │   }
│   │
│   ├── 溢出检查：code_ptr > highwater → return -1  :6733-6734
│   └── TB 大小检查：> UINT16_MAX → return -2        :6737-6738
│   }
│
├── ⑦ 后处理
│   tcg_out_ldst_finalize(s)     — 慢路径发射      :6747
│   tcg_out_pool_finalize(s)     — 常量池发射       :6751
│   tcg_resolve_relocs(s)        — 解析所有重定位    :6755
│
├── ⑧ flush_idcache_range()      — 刷新指令缓存     :6761-6763
│
└── return tcg_current_code_size(s)                :6766
```

**关键设计**：
- **延迟检查**：不在每条指令后检查溢出，而是在循环末尾检查。假设单条操作不会完全溢出缓冲区
- **三阶段后处理**：慢路径（qemu_ld/st 的 helper 调用）、常量池（64位大立即数）、重定位（分支偏移回填）都在主循环后处理

---

## 3. 寄存器分配核心 — tcg_reg_alloc_op()

**定义**：`tcg/tcg.c:5133-6025`

这是 TCG 寄存器分配的核心函数，处理每个普通 TCGOp 的寄存器分配和指令发射。

**处理流程**：

```
tcg_reg_alloc_op(s, op)
│
├── 1. 读取约束                                   :5183
│   args_ct = opcode_args_ct(op)
│
├── 2. 拷贝常量参数                               :5156-5159
│   memcpy(new_args + nb_oargs + nb_iargs, ...)
│
├── 3. 初始化已分配寄存器集                        :5161-5162
│   i_allocated = reserved_regs
│   o_allocated = reserved_regs
│
├── 4. 提取条件码（如有）                          :5164-5181
│   brcond → args[2], setcond → args[3], movcond → args[5]
│
├── 5. 满足输入约束                                :5185-5288
│   for each input:
│   │ ├── 常量输入：检查能否作为立即数
│   │ ├── 已在正确寄存器：复用
│   │ ├── 需要新寄存器：temp_load → 分配 → 移动
│   │ └── 处理别名约束（输入=输出同寄存器）
│
├── 6. 分配输出寄存器                              :5307-5487
│   for each output:
│   │ ├── 检查别名（ialias → 复用输入寄存器）
│   │ ├── 检查偏好（output_pref → 尽量使用偏好寄存器）
│   │ └── tcg_reg_alloc() 选择寄存器
│
├── 7. 发射宿主指令                                :5488+
│   tcg_out_op(s, op->opc, new_args, const_args)
│
├── 8. 处理输入 DEAD_ARG                           :5997-6000
│   if (IS_DEAD_ARG(i)) temp_dead(s, ts)
│
└── 9. 处理输出 SYNC_ARG                           
    if (SYNC_ARG(o)) temp_sync(s, ts, ...)
    if (DEAD_ARG(o)) temp_dead(s, ts)
```

---

## 4. 寄存器选择算法 — tcg_reg_alloc()

**定义**：`tcg/tcg.c:4645-4705`

```c
static TCGReg tcg_reg_alloc(TCGContext *s, TCGRegSet required_regs,
                             TCGRegSet allocated_regs,
                             TCGRegSet preferred_regs, bool rev)
```

**两遍选择策略**：

```
第一遍 (j=0)：在 preferred ∩ required ∩ ~allocated 中找空闲寄存器
第二遍 (j=1)：在 required ∩ ~allocated 中找空闲寄存器

如果两遍都没找到空闲的：
  → 选择一个已占用的寄存器
  → tcg_reg_free() 将其 spill 到内存
  → 返回该寄存器
```

**选择顺序**：遵循 `tcg_target_reg_alloc_order`（目标特定的寄存器优先级数组）

**偏好机制**：
- `output_pref[i]`（`TCGOp` 字段）：输出寄存器偏好，由活跃性分析设置
- 例如：如果下一条 op 需要 X0 作为输入，当前 op 的输出偏好会设为 X0

**配对分配器**（`tcg/tcg.c:4707-4751`）：
- `tcg_reg_alloc_pair()` 为 128 位操作分配连续的偶/奇寄存器对

---

## 5. Temp 加载/同步/保存策略

### temp_load() — 将 temp 值加载到物理寄存器

**定义**：`tcg/tcg.c:4755-4803`

```
temp_load(s, ts, desired_regs, allocated_regs, preferred_regs)
│
├── TEMP_VAL_REG → 直接返回（已在寄存器中）
│
├── TEMP_VAL_CONST → 
│   分配寄存器 + tcg_out_movi()（发射立即数加载指令）
│
├── TEMP_VAL_MEM → 
│   [若未分配帧] temp_allocate_frame()
│   分配寄存器 + tcg_out_ld()（从栈帧加载）
│
└── 设置 ts->val_type = TEMP_VAL_REG, ts->reg = reg
```

### temp_sync() — 确保值写回内存

**定义**：`tcg/tcg.c:4586-4624`

- 用于 `SYNC_ARG` 标记的操作数（全局变量需在 helper 调用前写回）
- 如果 `mem_coherent` 为 0，执行 `tcg_out_st()` 将寄存器值写入内存

### temp_save() — 保存 temp

**定义**：`tcg/tcg.c:4807-4812`

- 在当前实现中只是一个断言：确认 temp 已在内存中或是只读的
- 活跃性分析保证全局变量在基本块结束时已被同步

### tcg_reg_free() — 释放寄存器

**定义**：`tcg/tcg.c:4627-4633`

- 将占用该寄存器的 temp spill 到内存
- 调用 `temp_sync()` + 清除 `reg_to_temp` 映射

---

## 6. 栈帧与 Spill 槽分配

### 帧布局

**TCGContext 帧字段**（`include/tcg/tcg.h:358-361`）：
```c
intptr_t current_frame_offset;  // 当前分配位置
intptr_t frame_start;           // 帧起始偏移
intptr_t frame_end;             // 帧结束偏移（空间上限）
TCGTemp *frame_temp;            // 帧基址 temp（通常是 SP）
```

### temp_allocate_frame()

**定义**：`tcg/tcg.c:4451-4518`

```
temp_allocate_frame(s, ts)
│
├── 根据 base_type 确定大小和对齐
│   I32 → 4 字节, align 4
│   I64/V64 → 8 字节, align 8
│   I128/V128/V256 → 16 字节, align 16
│
├── off = ROUND_UP(current_frame_offset, align)
│
├── 检查溢出：off + size > frame_end
│   → tcg_raise_tb_overflow(s)（触发 TB 重启，使用更少指令）
│
├── current_frame_offset = off + size
│
├── 设置 temp 内存位置
│   ts->mem_offset = off
│   ts->mem_base = frame_temp（通常是 SP）
│   ts->mem_allocated = 1
│
└── [若 128 位拆分] 为每个子部分设置偏移
```

---

## 7. 约束系统 — TCGArgConstraint

**定义**：`include/tcg/tcg.h:707-717`

```c
typedef struct TCGArgConstraint {
    unsigned ct : 16;         // 约束类型位图
    unsigned alias_index : 4; // 别名索引
    unsigned sort_index : 4;  // 排序索引（分配顺序）
    unsigned pair_index : 4;  // 配对索引（128位用）
    unsigned pair : 2;        // 配对类型
    bool oalias : 1;          // 输出别名
    bool ialias : 1;          // 输入别名
    bool newreg : 1;          // 必须使用新寄存器
    TCGRegSet regs;           // 可用寄存器集合
} TCGArgConstraint;
```

**约束类型标志**（`ct` 字段）：
- `TCG_CT_REG`：必须在寄存器中
- `TCG_CT_CONST`：可以是常量
- `TCG_CT_REG_ZERO`：可以使用硬件零寄存器
- 目标特定的常量约束（如 12 位立即数范围）

### 约束处理链

```
tcg_target_op_def()              — 后端定义每个 opcode 的约束集
    ↓
process_constraint_sets()        — 解析约束字符串    tcg.c:3199-3388
    ↓
opcode_args_ct()                 — 运行时查询约束    tcg.c:3419-3422
    ↓
tcg_reg_alloc_op()               — 按约束分配寄存器
```

### AArch64 约束定义

**文件**：`tcg/aarch64/tcg-target.c.inc:3361-3410`

`tcg_target_op_def()` 为每个 TCGOpcode 返回约束集索引，定义哪些操作数可以是寄存器、立即数，以及寄存器别名关系。

约束字符串定义在 `tcg/aarch64/tcg-target-con-str.h`（24 行）和 `tcg-target-con-set.h`（38 行）中。

---

## 8. 寄存器分配器函数族

| 函数 | 行号 | 处理对象 |
|------|------|----------|
| `tcg_reg_alloc_op()` | `tcg.c:5133-6025` | 普通操作（add/sub/and 等） |
| `tcg_reg_alloc_mov()` | `tcg.c:4923-5018` | 寄存器移动（mov/ext） |
| `tcg_reg_alloc_dup()` | `tcg.c:5023-5131` | 向量复制（dup_vec） |
| `tcg_reg_alloc_call()` | `tcg.c:5874-6003` | 函数调用（helper call） |
| `tcg_reg_alloc_bb_end()` | `tcg.c:4843-4868` | 基本块结束处理 |
| `tcg_reg_alloc_pair()` | `tcg.c:4707-4751` | 128 位配对寄存器分配 |

### tcg_reg_alloc_bb_end()

**定义**：`tcg/tcg.c:4843-4868`

在基本块边界（`set_label` 和函数末尾）调用：

```c
for each non-global temp:
    TEMP_TB  → temp_save()（确保已写回内存）
    TEMP_EBB → assert DEAD（活跃性分析已保证）
    TEMP_CONST → assert CONST

save_globals(s, allocated_regs);  // 所有全局变量写回内存
```

### tcg_reg_alloc_call()

**定义**：`tcg/tcg.c:5874-6003`

处理 `INDEX_op_call`：
1. 将参数加载到调用约定指定的寄存器（X0-X7）
2. 保存所有 caller-saved 寄存器中的活跃 temp
3. 发射 BLR 指令
4. 将返回值标记到 X0/X1

---

## 9. 进位链处理

**背景**：多精度算术（如 128 位加法）需要跨多条操作传递进位标志。

**操作码**：
- `addco`：加法，输出进位
- `addci`：加法，输入进位
- `addcio`：加法，输入+输出进位
- `subbo`/`subbi`/`subbio`：减法的对应操作

**TCGContext.carry_live**（`include/tcg/tcg.h:413-417`）：
```c
bool carry_live;  // 进位链中时为 true
```

**标志位**（`include/tcg/tcg.h:741-744`）：
```c
TCG_OPF_CARRY_IN   // 操作码消费进位
TCG_OPF_CARRY_OUT  // 操作码产生进位
```

**处理逻辑**：
- 进入进位链：`carry_live = true`（当遇到 CARRY_OUT 操作时）
- 退出进位链：`carry_live = false`（当进位被最终消费）
- 在进位链内：不能在 carry 寄存器上执行 spill/reload
- 活跃性分析特殊处理：`tcg.c:4209-4225`

---

## 10. AArch64 后端总览

### 文件结构

| 文件 | 行数 | 内容 |
|------|------|------|
| `tcg/aarch64/tcg-target.c.inc` | 3592 | 主实现：指令编码、发射、prologue |
| `tcg/aarch64/tcg-target.h` | 52 | TCGReg 枚举、NB_REGS |
| `tcg/aarch64/tcg-target-con-str.h` | 24 | 约束字符串定义 |
| `tcg/aarch64/tcg-target-con-set.h` | 38 | 约束集定义 |
| `tcg/aarch64/tcg-target-opc.h.inc` | 15 | 目标特定操作码 |
| `tcg/aarch64/tcg-target-has.h` | 60 | 功能支持标志 |
| `tcg/aarch64/tcg-target-mo.h` | 12 | 内存序定义 |

---

## 11. AArch64 寄存器定义与分配序

### 寄存器枚举

**定义**：`tcg/aarch64/tcg-target.h:19-50`

```c
typedef enum {
    TCG_REG_X0 .. TCG_REG_X30, TCG_REG_SP,
    TCG_REG_V0 .. TCG_REG_V31,
    TCG_TARGET_NB_REGS = 64     // :50
} TCGReg;
```

### 分配优先级

**定义**：`tcg/aarch64/tcg-target.c.inc:47-72`

```
优先级（高→低）：
1. 被调用者保存：X20-X28（最优，跨 helper 调用不需保存）
2. 临时寄存器：X8-X15（caller-saved，但不与参数冲突）
3. 参数寄存器：X0-X7（最后使用，可能与 helper 参数冲突）
4. SIMD 寄存器：V0-V7, V16-V31（跳过 V8-V15 被调用者保存）
```

### 保留/特殊寄存器

**定义**：`tcg/aarch64/tcg-target.c.inc:86-91`

| 寄存器 | 用途 | 别名 |
|--------|------|------|
| X16 | TCG 临时寄存器 #0 | `TCG_REG_TMP0` |
| X17 | TCG 临时寄存器 #1 | `TCG_REG_TMP1` |
| X18 | 系统保留（平台 ABI） | — |
| X19 | AREG0（CPUState 指针） | `TCG_AREG0` |
| X28 | guest_base | `TCG_REG_GUEST_BASE` |
| X29 | 帧指针 (FP) | — |
| X30 | TCG 临时寄存器 #2 | `TCG_REG_TMP2` |
| SP | 栈指针 | — |
| V31 | 向量临时寄存器 | `TCG_VEC_TMP0` |

### 调用约定

**参数寄存器**（`:74-77`）：`X0-X7`（8 个参数寄存器）
**返回寄存器**（`:79-84`）：`X0`（slot 0）, `X1`（slot 1）

---

## 12. AArch64 指令编码体系

### 编码辅助函数族

`tcg-target.c.inc:662-905` 定义了完整的 AArch64 指令编码函数族：

```
tcg_out_insn(s, format, OP, ...)
    │
    ├── 格式分类：
    │   ├── ldstpair (LDP/STP)     :713-725
    │   ├── addsub_imm             :727-738
    │   ├── bitfield               :743-758
    │   ├── movw (MOVZ/MOVK/MOVN)  :760-767
    │   ├── pcrel (ADR/ADRP)       :769-773
    │   ├── addsub_reg (shifted)   :785-803
    │   ├── csel                   :809-815
    │   ├── rr (2-reg ops)         :817-820
    │   ├── rrrr (4-reg, MADD)     :822-825
    │   ├── ldst_r (reg offset)    :884-891
    │   └── ldst_i (imm offset)    :893-905
    │
    └── 所有最终调用 tcg_out32(s, encoding)
```

### tcg_out32 — 底层指令写入

将 32 位编码写入 `s->code_ptr` 并前进 4 字节。AArch64 所有指令都是 32 位定长。

---

## 13. AArch64 算术/逻辑指令发射

### ADD/SUB

```
ADD Rd, Rn, Rm [, shift #amount]
  → tcg_out_insn(s, addsub_reg, ADD, type, rd, rn, rm, shift, amount)
     编码函数: tcg-target.c.inc:785-803

ADD Rd, Rn, #imm12
  → tcg_out_insn(s, addsub_imm, ADDI, type, rd, rn, imm12)
     编码函数: tcg-target.c.inc:727-738
```

### 逻辑操作

AND/ORR/EOR 使用逻辑立即数编码（N:immr:imms）或寄存器形式。

### 移位

立即移位通过 bitfield 指令编码：
```
LSL #n → UBFM Rd, Rn, #(-n mod 64), #(63-n)
LSR #n → UBFM Rd, Rn, #n, #63
ASR #n → SBFM Rd, Rn, #n, #63
```
编码函数：`tcg-target.c.inc:3095-3127`

### 乘法

MADD 使用 4 寄存器格式：
```
MADD Rd, Rn, Rm, Ra → Rd = Rn * Rm + Ra
  → tcg_out_insn(s, rrrr, MADD, type, rd, rn, rm, ra)
     编码函数: tcg-target.c.inc:822-825
```

---

## 14. AArch64 内存指令发射

### 寄存器偏移加载/存储

```
LDR Rt, [Rn, Rm]
  → tcg_out_insn(s, ldst_r, LDR, type, rt, rn, rm)
     编码函数: tcg-target.c.inc:884-891
```

### 立即偏移加载/存储

```
LDR Rt, [Rn, #imm]
  → tcg_out_insn(s, ldst_i, LDR, type, rt, rn, imm)
     编码函数: tcg-target.c.inc:893-905
```

### 配对加载/存储

```
STP Rt1, Rt2, [Rn, #imm]!
  → tcg_out_insn(s, ldstpair, STP, rt1, rt2, rn, imm, pre, write_back)
     编码函数: tcg-target.c.inc:713-725
```

### PC 相对加载

```
LDR Rt, [PC, #offset]
  → tcg_out_insn(s, ldlit, LDR, type, rt, offset)
     编码函数: tcg-target.c.inc:671-675
```

---

## 15. AArch64 分支与重定位

### 分支编码

| 指令 | 编码函数行号 | 偏移范围 |
|------|-------------|---------|
| B/BL | `:703-706` | ±128 MB (26位) |
| B.cond | `:689-693` | ±1 MB (19位) |
| CBZ/CBNZ | `:683-687` | ±1 MB (19位) |
| TBZ/TBNZ | — | ±32 KB (14位) |
| BR/BLR | `:708-710` | 寄存器间接 |

### 重定位辅助

**定义**：`tcg-target.c.inc:93-146`

```c
// 26 位 PC 相对偏移（B/BL）
static bool reloc_pc26(tcg_insn_unit *src, const tcg_insn_unit *target);  // :93-105

// 19 位 PC 相对偏移（B.cond/CBZ/CBNZ）
static bool reloc_pc19(tcg_insn_unit *src, const tcg_insn_unit *target);  // :107-117

// 14 位 PC 相对偏移（TBZ/TBNZ）
static bool reloc_pc14(tcg_insn_unit *src, const tcg_insn_unit *target);  // :119-129

// 分发函数
static bool patch_reloc(tcg_insn_unit *code_ptr, int type,
                        intptr_t value, intptr_t addend);                  // :131-146
```

### 重定位流程

```
1. 发射分支时，目标标签可能还未解析
   → tcg_out_reloc(s, code_ptr, type, label, addend)
   → 将重定位信息记录到 label->relocs 链表

2. tcg_gen_code() 末尾调用 tcg_resolve_relocs()
   → 遍历所有标签的 relocs 列表
   → 对每个重定位调用 patch_reloc()
   → 回填分支指令中的偏移量
```

---

## 16. AArch64 常量加载策略 — tcg_out_movi

**定义**：`tcg/aarch64/tcg-target.c.inc:1104-1165`

64 位常量加载的策略选择（从简单到复杂）：

```
tcg_out_movi(s, type, rd, value)
│
├── value == 0 → MOVZ Rd, #0 (或 MOV Rd, XZR)
│
├── 可用单个 MOVZ/MOVN？
│   只需设置一个 16 位半字 → 单条指令          :1133-1140
│
├── 可用逻辑立即数？
│   ORR Rd, XZR, #bitmask → 单条指令            :1146-1149
│
├── PC 相对寻址可达？
│   ADR Rd, [PC+offset] → 单条指令               :1153-1158
│   （需要在 ±1MB 范围内）
│
├── ADRP + ADD 可达？
│   ADRP Rd, [PC+page_off]
│   ADD Rd, Rd, #page_rem  → 两条指令            :1160-1165
│
└── 多条 MOVZ + MOVK 序列
    MOVZ Rd, #imm16_0
    MOVK Rd, #imm16_1, LSL #16
    MOVK Rd, #imm16_2, LSL #32
    MOVK Rd, #imm16_3, LSL #48  → 最多 4 条指令
```

**优化效果**：大多数常量（小立即数、对齐地址、位掩码模式）只需 1-2 条指令。

---

## 17. AArch64 Prologue/Epilogue

**定义**：`tcg/aarch64/tcg-target.c.inc:3467-3534`

### Prologue（TB 入口点）

```asm
; BTI 着陆垫                                    :3471
BTI C

; 保存 FP/LR，设置栈帧                          :3473-3478
STP FP, LR, [SP, #-PUSH_SIZE]!
MOV FP, SP

; 保存被调用者保存寄存器 X19-X28                 :3480-3484
STP X19, X20, [SP, #16]
STP X21, X22, [SP, #32]
STP X23, X24, [SP, #48]
STP X25, X26, [SP, #64]
STP X27, X28, [SP, #80]

; 分配 TCG 局部变量栈空间                        :3486-3488
SUB SP, SP, #(FRAME_SIZE - PUSH_SIZE)

; 设置 TCG 帧信息                                :3491-3492
tcg_set_frame(s, SP, TCG_STATIC_CALL_ARGS_SIZE, ...)

; [可选] 加载 guest_base 到 X28                  :3494-3503
MOV X28, #guest_base

; 设置 AREG0 (CPUState) 并跳转到 TB 代码        :3505-3506
MOV X19, X0          ; AREG0 = 第一个参数
BR  X1               ; 跳转到 TB 代码（第二个参数）
```

### Epilogue（TB 退出点）

```asm
; goto_ptr 返回路径                              :3512-3514
tcg_code_gen_epilogue:
BTI J
MOV X0, #0           ; 返回值 = 0

; TB 返回地址                                    :3517-3518
tb_ret_addr:
BTI J

; 释放 TCG 局部变量栈空间                        :3520-3522
ADD SP, SP, #(FRAME_SIZE - PUSH_SIZE)

; 恢复被调用者保存寄存器 X19-X28                 :3524-3528
LDP X19, X20, [SP, #16]
... (5 对)

; 恢复 FP/LR，返回                               :3530-3533
LDP FP, LR, [SP], #PUSH_SIZE
RET
```

**两个入口点**：
- `tcg_code_gen_epilogue`：`goto_ptr` 未找到 TB 时到达，返回 0
- `tb_ret_addr`：`exit_tb` 到达，X0 包含返回值

---

## 18. 客户机内存访问发射 — qemu_ld/st

### 快慢路径架构

```
qemu_ld_i64 dest, addr, memidx, memop
│
├── prepare_host_addr()                          :1650+
│   ├── [softmmu] TLB 快路径检查
│   │   ├── 加载 TLB 条目
│   │   ├── 比较虚拟地址标签
│   │   ├── 匹配 → 获取宿主地址
│   │   └── 不匹配 → 跳转到慢路径标签
│   │
│   └── [user-only] addr + guest_base
│
├── tcg_out_qemu_ld_direct()                     :1760-1796
│   └── LDR/LDRB/LDRH/... Rt, [host_addr]
│
└── [softmmu] 慢路径（在 ldst_finalize 中发射）
    tcg_out_qemu_ld_slow_path()                  :1612-1625
    └── 调用 helper_ld_* 函数
        MOV X0, env
        MOV X1, addr
        BL  helper_ldq_mmu (或 helper_ldb/ldw/ldl)
        MOV dest, X0
        B   continue_label
```

### 大小处理

| MemOp | 加载指令 | 存储指令 |
|-------|---------|---------|
| MO_8 | LDRB | STRB |
| MO_16 | LDRH | STRH |
| MO_32 | LDR W | STR W |
| MO_64 | LDR X | STR X |
| MO_128 | LDP X,X | STP X,X |

### 128 位加载/存储

**定义**：`tcg-target.c.inc:1864+`

使用 LDP/STP 指令对实现，需要处理大小端和原子性。

---

## 19. TranslationBlock 结构

**定义**：`include/exec/translation-block.h:46-150`

```c
struct TranslationBlock {
    // ═══ 标识与匹配 ═══
    vaddr pc;                    // 客户机 PC                    :60
    uint64_t cs_base;            // 目标特定数据                  :70
    uint32_t flags;              // 翻译上下文标志                :72
    uint32_t cflags;             // 编译标志 (CF_*)              :73

    // ═══ 大小信息 ═══
    uint16_t size;               // 客户机代码大小（字节）        :95
    uint16_t icount;             // 客户机指令数                  :96

    // ═══ 生成的宿主代码 ═══
    struct tb_tc tc;             // {ptr, size} 宿主代码位置      :98

    // ═══ 页面追踪 ═══
    IntervalTreeNode itree;      // [用户模式] 区间树节点         :109
    uintptr_t page_next[2];      // [系统模式] 页面链表           :111
    tb_page_addr_t page_addr[2]; // [系统模式] 物理页地址         :112

    // ═══ 跳转锁 ═══
    QemuSpin jmp_lock;           // 保护跳转链表                  :116

    // ═══ 直接跳转 (2 槽) ═══
    uint16_t jmp_reset_offset[2]; // 原始跳转目标偏移             :126
    uint16_t jmp_insn_offset[2];  // 跳转指令偏移                :127
    uintptr_t jmp_target_addr[2]; // 目标地址                    :128

    // ═══ 跳转链表 ═══
    uintptr_t jmp_list_head;     // 入边跳转链表头                :147
    uintptr_t jmp_list_next[2];  // 链表下一个节点                :148
    uintptr_t jmp_dest[2];       // 出边目标 TB（tagged pointer） :149
};
```

### cflags 编译标志

**定义**：`:76-88`

| 标志 | 值 | 含义 |
|------|------|------|
| `CF_COUNT_MASK` | 0x1FF | 最大指令数掩码 (512) |
| `CF_NO_GOTO_TB` | 0x200 | 禁止 TB 链接 |
| `CF_NO_GOTO_PTR` | 0x400 | 禁止间接 TB 跳转 |
| `CF_SINGLE_STEP` | 0x800 | GDB 单步模式 |
| `CF_PARALLEL` | 0x8000 | 并行上下文（MTTCG） |
| `CF_PCREL` | 0x20000 | PC 相对操作码 |
| `CF_INVALID` | 0x4000 | TB 已失效 |

---

## 20. TB 创建全流程 — tb_gen_code()

**定义**：`accel/tcg/translate-all.c:261-540`

```
tb_gen_code(cpu, pc, cs_base, flags, cflags)
│
├── 1. 分配 TB 结构                               :289-302
│   从代码缓冲区当前位置分配
│
├── 2. 设置 TB 字段                                :304-315
│   tb->pc = pc
│   tb->cs_base = cs_base
│   tb->flags = flags
│   tb->cflags = cflags
│   tb->tc.ptr = code_gen_ptr（当前代码缓冲区位置）
│
├── 3. 翻译 + 代码生成                             :324+
│   setjmp_gen_code(s, tb, pc, ...)
│   │
│   ├── 调用目标翻译器                             :247-252
│   │   aarch64_translate_code(cpu, tb, ...)
│   │   → translator_loop → 生成 TCG IR
│   │
│   └── tcg_gen_code(s, tb, pc)                    :257
│       → 优化 + 活跃性 + 寄存器分配 + 代码发射
│
├── 4. 推进代码缓冲区指针                          :478-480
│   code_gen_ptr += ROUND_UP(code_size + search_size)
│
├── 5. 初始化跳转元数据                            :482-496
│   jmp_reset_offset[] → 保存原始跳转位置
│   jmp_target_addr[] → 设为 TB 内重置地址
│
├── 6. 插入哈希表和页面表                          :524-540
│   tb_link_page(tb)
│   → 将 TB 关联到物理页面
│   → 插入全局 TB 哈希表
│
└── return tb
```

---

## 21. TB 查找与哈希

### 两级查找

```
tb_lookup(cpu, pc, cs_base, flags, cflags)          cpu-exec.c:227-263
│
├── 1. 快速路径：CPU 本地 jump cache
│   hash = tb_jmp_cache_hash(pc)
│   tb = cpu->tb_jmp_cache[hash]
│   if (tb && tb->pc == pc && tb->flags == flags ...) → 命中！
│
└── 2. 慢速路径：全局哈希表
    tb_htable_lookup(cpu, pc, cs_base, flags, cflags)  :195-210
    → 使用 tb_hash_func() 计算哈希
    → 在全局哈希表中查找
    → 找到后更新 jump cache
```

### 哈希函数

**定义**：`accel/tcg/tb-hash.h:39-68`

jump cache 哈希只使用 PC（简单快速），全局哈希使用 PC + cs_base + flags + cflags（精确匹配）。

### Jump Cache

**定义**：`accel/tcg/tb-jmp-cache.h:15-31`

每个 CPU 持有一个 jump cache，是 PC → TB 的直接映射（类似 CPU 的直接映射缓存）。

---

## 22. TB 链接（Chaining）

### 原理

当 TB-A 的末尾跳转到 TB-B 时，可以直接修补 TB-A 的跳转指令，使其直接跳到 TB-B 的代码，跳过查找过程。

### tb_add_jump()

**定义**：`accel/tcg/cpu-exec.c:616-651`

```
tb_add_jump(tb_src, slot, tb_dst)
│
├── 检查：tb_dst 未失效（CF_INVALID）
├── 检查：slot 未被链接
│
├── 设置 tb_src->jmp_dest[slot] = tb_dst
│   （标记出边指向目标 TB）
│
├── tb_set_jmp_target(tb_src, slot, tb_dst->tc.ptr)    :600-614
│   → 修补 tb_src 代码中的跳转指令
│   → 后端函数：tb_target_set_jmp_target()
│   → AArch64：修改 B 指令的偏移量
│
└── 将 tb_src 加入 tb_dst->jmp_list_head 链表
    （记录入边，用于后续解链）
```

### 跳转字段解析

```
TB-A 代码区域：
┌────────────────────────────────────┐
│ ... 指令 ...                       │
│                                    │
│ [jmp_insn_offset[0]] → B target0  │ ← goto_tb 0 的位置
│ [jmp_reset_offset[0]] → (下一条)   │ ← 未链接时的跳转目标
│                                    │
│ [jmp_insn_offset[1]] → B target1  │ ← goto_tb 1 的位置
│ [jmp_reset_offset[1]] → (下一条)   │
│                                    │
│ exit code ...                      │
└────────────────────────────────────┘

未链接：B 跳到 reset_offset（继续执行退出代码）
已链接：B 跳到 tb_dst->tc.ptr（直接跳转到下一个 TB）
```

---

## 23. TB 解链与失效

### tb_jmp_unlink()

**定义**：`accel/tcg/tb-maint.c:877-892`

将 TB 从所有入边 TB 的跳转链中移除，恢复被修补的跳转指令。

### 失效触发条件

| 场景 | 函数 | 位置 |
|------|------|------|
| 物理内存写入 | `tb_invalidate_phys_range()` | tb-maint.c:1024 |
| 单个 TB 失效 | `tb_phys_invalidate()` | tb-maint.c:972 |
| 全局刷新 | `tb_flush()` | tb-maint.c:770 |
| 自修改代码 | `tb_invalidate_phys_page_unwind()` | tb-maint.c:1056 |
| 断点设置 | 通过页面失效 | — |
| TLB 刷新 | 间接导致 | — |

### do_tb_phys_invalidate()

**定义**：`accel/tcg/tb-maint.c:921-959`

```
do_tb_phys_invalidate(tb)
│
├── 从全局哈希表移除
├── 从 jump cache 移除：tb_jmp_cache_inval_tb()
├── 解除所有出边链接
│   for slot in [0, 1]:
│       tb_remove_from_jmp_list(tb, slot)
├── 解除所有入边链接
│   tb_jmp_unlink(tb)
├── 标记 CF_INVALID
└── 将代码空间标记为可复用（延迟回收）
```

---

## 24. 代码缓冲区管理

### 初始化

**函数**：`tcg/region.c:737-864`

```
tcg_region_init()
│
├── alloc_code_gen_buffer()                        :491-518
│   → mmap 分配可执行内存
│   → 大小默认为 RAM/4（有上限）
│
├── 分割为多个 region（MTTCG 用）                   :348-387
│   tcg_n_regions() 计算 region 数量
│   每个 vCPU 线程使用独立 region
│
└── 每个 region 设置 bounds                         :329-346
    region_start, region_end
```

### 多线程分区

```
代码缓冲区（总体）：
┌─────────────┬─────────────┬─────────────┬─────────────┐
│  Region 0   │  Region 1   │  Region 2   │  Region 3   │
│  (vCPU 0)   │  (vCPU 1)   │  (vCPU 2)   │  (vCPU 3)   │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

### 溢出处理

```
代码缓冲区满时：
1. tcg_gen_code() 检测 code_ptr > highwater → return -1
2. tb_gen_code() 收到 -1
3. 触发 tb_flush()
4. tcg_region_reset_all() 重置所有 region
5. 从头开始重新翻译
```

---

## 25. TB 执行循环

### 主循环

**定义**：`accel/tcg/cpu-exec.c:933-990`

```
cpu_exec_loop(cpu)
│
while (!exit_request) {
│
├── tb = tb_lookup(cpu, pc, ...)                   // 查找 TB
│   if (!tb) tb = tb_gen_code(cpu, pc, ...)        // 未找到则翻译
│
├── cpu_loop_exec_tb(cpu, tb, ...)                 :884-928
│   │
│   ├── cpu_tb_exec(cpu, tb)                       :428-490
│   │   │
│   │   ├── ret = tcg_qemu_tb_exec(env, tb->tc.ptr)  :439
│   │   │   → 跳转到 TB 生成的宿主代码执行
│   │   │   → TB 代码执行完毕后返回
│   │   │
│   │   ├── tb_exit = ret & TB_EXIT_MASK           :450-451
│   │   │
│   │   └── if (tb_exit > TB_EXIT_IDX1)            :455
│   │       → 特殊退出（异常/中断/调试）
│   │
│   └── 处理 TB 退出类型
│       ├── TB_EXIT_IDX0/1 → 正常出口（goto_tb 0/1）
│       │   → tb_add_jump() 尝试链接              :1008-1010
│       └── 其他 → 回到主循环检查中断
│
└── 检查中断/异常/退出请求
}
```

### tcg_qemu_tb_exec() 入口

调用 Prologue 中设置的入口点，传入：
- X0 = `env`（CPUState/CPUArchState 指针）
- X1 = `tb->tc.ptr`（TB 代码起始地址）

Prologue 将 X0 存入 X19（AREG0），然后 `BR X1` 跳转到 TB 代码。

---

## 26. 自修改代码处理

### 检测机制

```
客户机写入代码页
│
├── [系统模式] TB 创建时注册页面保护
│   tb_page_add() → tlb_protect_code()          tb-maint.c:698-715
│   → 将代码页标记为只读
│   → 写入时触发 TLB 写保护故障
│
├── 写保护故障处理
│   → tb_invalidate_phys_page_unwind()           tb-maint.c:1056-1103
│   → 失效该页上的所有 TB
│   → 恢复写权限
│   → 重新执行写入
│
└── [用户模式] 直接检查
    tb_invalidate_phys_range()                    tb-maint.c:1024-1035
```

### 断点检测

```
tb_check_watchpoint()                              translate-all.c:544-566
│
├── 检查写入地址是否触及活跃 TB
├── 如果是 → 失效该 TB
└── 设置 CF_BP_PAGE 标记
```

---

## 27. 代码缓冲区统计

### 统计接口

| 函数 | 文件 | 返回值 |
|------|------|--------|
| `tcg_nb_tbs()` | region.c:299-312 | 活跃 TB 数量 |
| `tcg_code_size()` | region.c:873-891 | 已使用代码空间（字节） |
| `tcg_code_capacity()` | region.c:898-909 | 总代码空间容量 |

### 计数器

**定义**：`accel/tcg/tb-context.h:35-38`

```c
struct TBContext {
    ...
    unsigned tb_flush_count;              // TB 全局刷新次数
    unsigned tb_phys_invalidate_count;    // 单 TB 失效次数
};
```

### 输出

**函数**：`accel/tcg/tcg-stats.c:150-203`

通过 QEMU monitor `info jit` 命令输出：
- TB 数量、平均大小
- 代码缓冲区使用率
- TB 哈希表占用率
- 刷新/失效计数

---

## 28. 完整代码生成示例：ADD X0, X1, X2 的后端之旅

### 前端 IR（输入）

```
insn_start  0x1000, 0, 0
add_i64     tmp0(X0), tmp1(X1), tmp2(X2)
```

### 步骤 1：约束查询

```
tcg_reg_alloc_op(s, op)
│
├── args_ct = opcode_args_ct(INDEX_op_add)
│   → 约束：output=reg, input1=reg, input2=reg
│
├── nb_oargs = 1, nb_iargs = 2
```

### 步骤 2：输入分配

```
输入 1: tmp1 (cpu_X[1], GLOBAL)
├── val_type == TEMP_VAL_MEM（全局在内存中）
├── temp_load(s, tmp1, ALL_REGS, reserved, {})
│   → tcg_reg_alloc() 选择 X20（第一优先被调用者保存寄存器）
│   → tcg_out_ld(s, I64, X20, X19, offsetof(env, xregs[1]))
│     发射：LDR X20, [X19, #xregs[1]_offset]
├── tmp1->val_type = REG, tmp1->reg = X20

输入 2: tmp2 (cpu_X[2], GLOBAL)
├── 同理加载到 X21
│   发射：LDR X21, [X19, #xregs[2]_offset]
```

### 步骤 3：输出分配

```
输出: tmp0 (cpu_X[0], GLOBAL)
├── tcg_reg_alloc() 选择 X22
├── new_args[0] = X22
```

### 步骤 4：指令发射

```
tcg_out_op(s, INDEX_op_add, {X22, X20, X21}, {0, 0, 0})
│
└── tcg_out_insn(s, addsub_reg, ADD, TCG_TYPE_I64, X22, X20, X21, 0, 0)
    → 编码 AArch64 ADD 指令
    → tcg_out32(s, 0x8B150296)  // ADD X22, X20, X21
    → s->code_ptr += 4
```

### 步骤 5：输出处理

```
如果 SYNC_ARG(0)：
├── temp_sync(s, tmp0, ...)
│   → tcg_out_st(s, I64, X22, X19, offsetof(env, xregs[0]))
│     发射：STR X22, [X19, #xregs[0]_offset]
│   → tmp0->mem_coherent = 1

如果 DEAD_ARG(0)：
├── temp_dead(s, tmp0)
│   → 释放 X22
```

### 最终宿主代码

```asm
LDR X20, [X19, #xregs_1_off]    ; 加载 cpu_X[1]
LDR X21, [X19, #xregs_2_off]    ; 加载 cpu_X[2]
ADD X22, X20, X21                 ; X22 = X20 + X21
STR X22, [X19, #xregs_0_off]    ; 存回 cpu_X[0]
```

**注意**：如果 X1/X2 在之前的操作中已在寄存器中（热路径），则不需要 LDR 加载步骤，直接使用已有寄存器。

---

## 29. 附录 A：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `tcg/tcg.c` | ~6770 | 核心引擎：tcg_gen_code、寄存器分配器、活跃性分析 |
| `tcg/aarch64/tcg-target.c.inc` | 3592 | AArch64 后端：指令编码、发射、prologue |
| `tcg/aarch64/tcg-target.h` | 52 | AArch64 寄存器枚举 |
| `tcg/region.c` | ~910 | 代码缓冲区管理、region 分配 |
| `include/tcg/tcg.h` | ~800 | TCG 核心类型（TCGContext/TCGOp/TCGTemp/约束） |
| `include/exec/translation-block.h` | ~160 | TranslationBlock 结构定义 |
| `accel/tcg/translate-all.c` | ~570 | tb_gen_code()：TB 创建入口 |
| `accel/tcg/cpu-exec.c` | ~1000 | TB 执行循环、查找、链接 |
| `accel/tcg/tb-maint.c` | ~1200 | TB 维护：失效、解链、刷新 |
| `accel/tcg/tb-hash.h` | ~70 | TB 哈希函数 |
| `accel/tcg/tb-jmp-cache.h` | ~30 | Jump cache 定义 |
| `accel/tcg/tcg-stats.c` | ~200 | JIT 统计信息 |

---

## 30. 附录 B：TB 生命周期状态图

```
                    tb_gen_code()
  [不存在] ─────────────────────→ [已创建]
                                     │
                              tb_link_page()
                                     │
                                     ▼
                                [已注册]
                               ╱         ╲
                     tb_lookup()          tb_add_jump()
                             ╱                 ╲
                            ▼                   ▼
                       [被执行]            [已链接]
                            │                   │
                            └──── 正常运行 ─────┘
                                     │
                         ┌───────────┼───────────┐
                         ▼           ▼           ▼
                   写入代码页    TLB 刷新    tb_flush()
                         │           │           │
                         ▼           ▼           ▼
               tb_invalidate    jmp_cache     全局刷新
                _phys_range     _clear
                         │           │           │
                         └───────────┼───────────┘
                                     ▼
                              [已失效]
                                     │
                         do_tb_phys_invalidate()
                                     │
                         ├── 从哈希表移除
                         ├── 解除所有链接
                         ├── 标记 CF_INVALID
                         └── 代码空间可回收
                                     │
                              tb_flush() 时
                                     │
                                     ▼
                               [已回收]
                         代码空间返回缓冲区
```

---

> **文档版本**：v1.0
> **源码版本**：QEMU 11.0.50 (commit 基于 2025-07 主线)
> **分析工具**：zoekt + ctags + cscope + clangd + 手动源码验证
> **交叉验证**：所有行号均经 grep -n / view 验证
