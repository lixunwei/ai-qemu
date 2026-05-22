# TCG IR 设计与前端翻译深度分析

> QEMU 11.0.50 · 分析日期 2025-07 · 基于源码交叉验证

## 目录

1. [概述](#1-概述)
2. [TCG IR 类型系统总览](#2-tcg-ir-类型系统总览)
3. [TCGTemp — 临时变量核心结构](#3-tcgtemp--临时变量核心结构)
4. [TCGTempKind — 临时变量生命周期分类](#4-tcgtempkind--临时变量生命周期分类)
5. [TCGTempVal — 运行时值状态](#5-tcgtempval--运行时值状态)
6. [TCGType — 数据类型系统](#6-tcgtype--数据类型系统)
7. [TCGv 句柄类型与类型安全](#7-tcgv-句柄类型与类型安全)
8. [TCGOp — IR 操作节点](#8-tcgop--ir-操作节点)
9. [TCGOpcode — 操作码定义体系](#9-tcgopcode--操作码定义体系)
10. [操作码分类完整索引](#10-操作码分类完整索引)
11. [TCGLabel — 分支标签](#11-tcglabel--分支标签)
12. [TCGContext — 翻译上下文](#12-tcgcontext--翻译上下文)
13. [临时变量生命周期管理](#13-临时变量生命周期管理)
14. [TCG Op 生成 API](#14-tcg-op-生成-api)
15. [ARM64 全局寄存器映射](#15-arm64-全局寄存器映射)
16. [NZCV 条件标志缓存体系](#16-nzcv-条件标志缓存体系)
17. [前端翻译架构总览](#17-前端翻译架构总览)
18. [DisasContext — 反汇编上下文](#18-disascontext--反汇编上下文)
19. [translator_loop — 翻译主循环](#19-translator_loop--翻译主循环)
20. [指令解码树 (decodetree)](#20-指令解码树-decodetree)
21. [数据处理指令翻译模式](#21-数据处理指令翻译模式)
22. [条件码生成详解](#22-条件码生成详解)
23. [条件码评估 — arm_test_cc](#23-条件码评估--arm_test_cc)
24. [条件比较与条件选择 (CCMP/CSEL)](#24-条件比较与条件选择-ccmpcsel)
25. [内存访问翻译](#25-内存访问翻译)
26. [分支与控制流翻译](#26-分支与控制流翻译)
27. [异常与系统指令翻译](#27-异常与系统指令翻译)
28. [TB 终止与退出策略](#28-tb-终止与退出策略)
29. [Helper 函数机制](#29-helper-函数机制)
30. [完整翻译示例：ADD X0, X1, X2 全流程](#30-完整翻译示例add-x0-x1-x2-全流程)
31. [附录 A：关键源文件索引](#31-附录-a关键源文件索引)
32. [附录 B：TCGv 与 TCGTemp 映射关系图](#32-附录-b-tcgv-与-tcgtemp-映射关系图)

---

## 1. 概述

TCG (Tiny Code Generator) 的 IR (Intermediate Representation) 是 QEMU 二进制翻译的核心。每条客户机指令被前端翻译器分解为一系列 TCG IR 操作（TCGOp），然后由优化器和后端将这些 IR 操作转换为宿主机器码。

本文深入分析：
- **IR 类型系统**：TCGTemp、TCGOp、TCGLabel、TCGContext 等核心数据结构
- **前端翻译**：ARM64 → TCG IR 的完整翻译流程
- **条件码体系**：NZCV 缓存编码、生成模式、评估机制
- **指令翻译模式**：数据处理、内存访问、分支控制、异常处理的具体翻译策略

**关键数字**：
- `translate-a64.c`：**10961 行**，**156 个** `trans_` 翻译函数
- `tcg-opc.h`：**126 个** DEF 操作码（+ 目标特定扩展）
- `tcg.h`：核心类型定义，TCGContext 约 100 个字段

---

## 2. TCG IR 类型系统总览

TCG IR 类型系统由以下核心类型构成：

```
┌─────────────────────────────────────────────────────┐
│                    TCGContext                         │
│  ┌─────────┐  ┌──────────┐  ┌─────────────────┐    │
│  │TCGTemp[]│  │QTAILQ ops│  │QSIMPLEQ labels  │    │
│  │ (temps) │  │ (TCGOp)  │  │   (TCGLabel)     │    │
│  └────┬────┘  └────┬─────┘  └────────┬────────┘    │
│       │            │                  │              │
│  ┌────▼────┐  ┌────▼─────┐  ┌────────▼────────┐    │
│  │kind     │  │opc       │  │id, has_value     │    │
│  │type     │  │args[]    │  │branches (uses)   │    │
│  │val_type │  │life      │  │relocs            │    │
│  │reg      │  │link      │  └──────────────────┘    │
│  │val/mem  │  │output_pref│                          │
│  └─────────┘  └──────────┘                           │
└─────────────────────────────────────────────────────┘
```

**类型层次**（`include/tcg/tcg.h`）：

| 类型 | 行号 | 用途 |
|------|------|------|
| `TCGType` | 128-149 | 数据宽度（I32/I64/I128/V64/V128/V256） |
| `TCGTempVal` | 246-251 | 运行时值位置（DEAD/REG/MEM/CONST） |
| `TCGTempKind` | 253-268 | 生命周期分类（EBB/TB/GLOBAL/FIXED/CONST） |
| `TCGTemp` | 270-293 | 临时变量主结构 |
| `TCGOp` | 310-329 | IR 操作节点 |
| `TCGLabel` | 99-111 | 分支标签 |
| `TCGContext` | 346-441 | 翻译上下文 |
| `TCGv_i32/i64/vec/ptr` | 201-205 | 类型安全句柄 |

---

## 3. TCGTemp — 临时变量核心结构

**定义**：`include/tcg/tcg.h:270-293`

```c
typedef struct TCGTemp {
    TCGReg reg:8;              // 当前分配的物理寄存器
    TCGTempVal val_type:8;     // 值状态(DEAD/REG/MEM/CONST)
    TCGType base_type:8;       // 基础类型(创建时)
    TCGType type:8;            // 实际类型(可能因拆分而不同)
    TCGTempKind kind:3;        // 生命周期类型
    unsigned int indirect_reg:1;   // 是否间接寄存器引用
    unsigned int indirect_base:1;  // 是否间接基址
    unsigned int mem_coherent:1;   // 内存中值是否与reg一致
    unsigned int mem_allocated:1;  // 是否已分配栈帧空间
    unsigned int temp_allocated:1; // 是否已分配
    unsigned int temp_subindex:2;  // 子索引(128位拆分用)

    int64_t val;               // TEMP_VAL_CONST时的常量值
    struct TCGTemp *mem_base;  // 内存存储的基址temp
    intptr_t mem_offset;       // 相对基址的偏移量
    const char *name;          // 调试名称

    uintptr_t state;           // 优化遍使用的状态
    void *state_ptr;           // 优化遍使用的指针
} TCGTemp;
```

**关键语义**：

- **`reg` + `val_type`**：寄存器分配器通过这两个字段追踪 temp 的当前位置。`val_type == TEMP_VAL_REG` 表示值在 `reg` 指定的物理寄存器中
- **`mem_base` + `mem_offset`**：当 temp 被 spill 到栈上时，其位置为 `mem_base + mem_offset`，其中 `mem_base` 通常是帧指针
- **`state` + `state_ptr`**：优化遍临时使用的字段。`optimize.c` 将 `state_ptr` 指向 `TempOptInfo`，用于 z_mask/o_mask/s_mask 追踪
- **`temp_subindex`**：128 位值拆分为 2 个 64 位 temp 时，用于标识低 (0) / 高 (1) 半部

---

## 4. TCGTempKind — 临时变量生命周期分类

**定义**：`include/tcg/tcg.h:253-268`

```c
typedef enum TCGTempKind {
    TEMP_EBB,      // EBB 内有效，EBB 结束时 dead
    TEMP_TB,       // 整个 TB 有效，TB 结束时 dead
    TEMP_GLOBAL,   // 跨 TB 有效（CPU 状态）
    TEMP_FIXED,    // 固定寄存器（不可分配）
    TEMP_CONST,    // 常量（只读）
} TCGTempKind;
```

**生命周期规则**：

```
         ┌─ TEMP_EBB ─── 最短：条件分支后可复用
         │
         ├─ TEMP_TB ──── 中等：整个 TB 有效
生命周期  │
(短→长)  ├─ TEMP_GLOBAL ─ 长：跨 TB 存活，对应 CPUState 字段
         │
         ├─ TEMP_FIXED ── 永久：固定映射到物理寄存器（如帧指针）
         │
         └─ TEMP_CONST ── 永久：编译时常量，只读
```

**关键判断**（`include/tcg/tcg.h:443-446`）：
```c
static inline bool temp_readonly(TCGTemp *ts)
{
    return ts->kind >= TEMP_FIXED;  // FIXED 和 CONST 均只读
}
```

**EBB vs TB**：
- **EBB (Extended Basic Block)**：从 TB 入口到第一个条件分支的直线区域。`TEMP_EBB` 在条件分支后不保证有效
- **TB (Translation Block)**：完整的翻译块。`TEMP_TB` 在整个 TB 内有效，但活跃性分析 `liveness_pass_0` 可能将其降级为 `TEMP_EBB`

---

## 5. TCGTempVal — 运行时值状态

**定义**：`include/tcg/tcg.h:246-251`

```c
typedef enum TCGTempVal {
    TEMP_VAL_DEAD,    // 未初始化 / 已释放
    TEMP_VAL_REG,     // 值在物理寄存器中
    TEMP_VAL_MEM,     // 值已 spill 到内存
    TEMP_VAL_CONST,   // 值是编译时常量
} TCGTempVal;
```

**状态转换图**：

```
              tcg_temp_new()
  [不存在] ──────────────────→ DEAD
                                 │
                          首次写入(alloc reg)
                                 │
                                 ▼
                               REG ◄─── temp_load()
                              ╱   ╲         ↑
                   spill()  ╱     ╲   tcg_reg_alloc_op()
                           ╱       ╲
                          ▼         ╲
                        MEM ─────────→ 需要时重新加载
                                 
  CONST: 只有 TEMP_CONST kind 的 temp 使用此状态
```

---

## 6. TCGType — 数据类型系统

**定义**：`include/tcg/tcg.h:128-149`

```c
typedef enum TCGType {
    TCG_TYPE_I32,     // 32 位整数
    TCG_TYPE_I64,     // 64 位整数
    TCG_TYPE_I128,    // 128 位整数（需拆分为两个 64 位）

    TCG_TYPE_V64,     // 64 位向量
    TCG_TYPE_V128,    // 128 位向量
    TCG_TYPE_V256,    // 256 位向量

    TCG_TYPE_REG = TCG_TYPE_I64,  // 宿主寄存器大小
    TCG_TYPE_PTR = TCG_TYPE_I64,  // 宿主指针大小
} TCGType;
```

**ARM64 客户机映射**：
- 通用寄存器 X0-X30：`TCG_TYPE_I64`
- 32 位操作 W0-W30：虽然操作在 I64 temp 上，但结果被零扩展到 64 位
- SIMD 寄存器 V0-V31：`TCG_TYPE_V128`
- NZCV 标志 (cpu_NF/ZF/CF/VF)：`TCG_TYPE_I32`

---

## 7. TCGv 句柄类型与类型安全

**定义**：`include/tcg/tcg.h:201-205`

```c
typedef struct TCGv_i32_d *TCGv_i32;
typedef struct TCGv_i64_d *TCGv_i64;
typedef struct TCGv_i128_d *TCGv_i128;
typedef struct TCGv_ptr_d *TCGv_ptr;
typedef struct TCGv_vec_d *TCGv_vec;
typedef TCGv_ptr TCGv_env;
```

**设计精妙之处**：`TCGv_i32_d` 和 `TCGv_i64_d` 是**从未定义的结构体**。TCGv 实际上是一个编码了 TCGTemp 索引的指针值，但通过声明为不同的不完整类型指针，编译器会在类型不匹配时报错。

**TCGv ↔ TCGTemp 转换**（`include/tcg/tcg.h:494-574`）：

```c
// TCGv → TCGTemp（句柄解码为 temp 指针）
static inline TCGTemp *tcgv_i64_temp(TCGv_i64 v) {
    uintptr_t o = (uintptr_t)v;
    return (void *)tcg_ctx + o;   // v 是相对于 tcg_ctx 的偏移
}

// TCGTemp → TCGv（temp 指针编码为句柄）
static inline TCGv_i64 temp_tcgv_i64(TCGTemp *t) {
    uintptr_t o = (void *)t - (void *)tcg_ctx;
    return (TCGv_i64)o;
}
```

**通用别名**（`include/tcg/tcg-op.h:35-43`）：
```c
// 在 64 位宿主上：
typedef TCGv_i64 TCGv;     // TCGv 是 TCGv_i64 的别名
```

---

## 8. TCGOp — IR 操作节点

**定义**：`include/tcg/tcg.h:310-329`

```c
struct TCGOp {
    TCGOpcode opc   : 8;       // 操作码（INDEX_op_add 等）
    unsigned nargs  : 8;       // 参数数量

    unsigned param1 : 8;       // 操作码参数1（如 MemOp）
    unsigned param2 : 8;       // 操作码参数2（如条件码）

    TCGLifeData life;          // 活跃性数据（DEAD_ARG/SYNC_ARG）

    QTAILQ_ENTRY(TCGOp) link;  // 双向链表节点

    TCGRegSet output_pref[2];  // 输出寄存器偏好（最多2个输出）

    TCGArg args[];             // 柔性数组：操作数列表
};
```

**关键字段详解**：

- **`opc`**：8 位操作码，值为 `TCGOpcode` 枚举成员（如 `INDEX_op_add = 2`）
- **`life`**：活跃性分析结果。每 5 位对应一个参数，使用 `DEAD_ARG`（bit 4）和 `SYNC_ARG`（bit 0）标记：
  ```c
  #define DEAD_ARG  (1 << 4)   // include/tcg/tcg.h:306
  #define SYNC_ARG  (1 << 0)   // include/tcg/tcg.h:307
  ```
- **`args[]`**：柔性数组成员，依次存放输出 temp 索引、输入 temp 索引、常量参数
- **`link`**：TCGOp 通过 QTAILQ 双向链表连接，存储在 `TCGContext.ops` 中

**Op 分配与插入**（`tcg/tcg.c:2500-2587`）：
```c
// 从空闲列表或新分配
static TCGOp *tcg_op_alloc(TCGOpcode opc, unsigned nargs);  // :2500

// 默认插入到 ops 尾部，或插入到 emit_before_op 之前
QTAILQ_INSERT_BEFORE / QTAILQ_INSERT_TAIL                    // :2583-2587
```

---

## 9. TCGOpcode — 操作码定义体系

**定义方式**：`include/tcg/tcg.h:59-64`

```c
typedef enum TCGOpcode {
#define DEF(name, oargs, iargs, cargs, flags) INDEX_op_ ## name,
#include "tcg/tcg-opc.h"
#undef DEF
    NB_OPS,
} TCGOpcode;
```

操作码在 `include/tcg/tcg-opc.h:25-185` 中通过 `DEF` 宏定义：

```c
DEF(name, oargs, iargs, cargs, flags)
//   |       |      |      |       |
//   名称  输出数  输入数  常量数  标志位
```

**标志位**：
- `TCG_OPF_BB_END`：基本块结束（br、brcond、set_label）
- `TCG_OPF_CALL_CLOBBER`：调用指令，破坏寄存器
- `TCG_OPF_NOT_PRESENT`：不直接出现在 op 流中（mov、call 等特殊处理）
- `TCG_OPF_INT`：整数操作（受 TCGType 影响）
- `TCG_OPF_COND_BRANCH`：条件分支（影响 EBB 分析）

---

## 10. 操作码分类完整索引

共 **126 个通用操作码** + 目标特定扩展（`include/tcg/tcg-opc.h:25-185`）。

### 预定义/控制操作（行 30-39）

| 操作码 | 输出/输入/常量 | 说明 |
|--------|---------------|------|
| `discard` | 1/0/0 | 丢弃 temp（释放寄存器） |
| `set_label` | 0/0/1 | 设置标签（分支目标） |
| `call` | 0/0/3 | 函数调用（特殊处理） |
| `br` | 0/0/1 | 无条件跳转 |
| `brcond` | 0/2/2 | 条件分支（2 输入 + cond + label） |
| `mb` | 0/0/1 | 内存屏障 |

### 整数算术/逻辑操作（行 41-104）

| 类别 | 操作码 | 说明 |
|------|--------|------|
| 数据移动 | `mov` | 寄存器间移动（NOT_PRESENT，通过优化处理） |
| 算术 | `add`, `sub`, `mul`, `neg` | 基本算术 |
| 算术(宽) | `muls2`, `mulu2` | 带高位结果的乘法 |
| 除法 | `divs`, `divu`, `divs2`, `divu2` | 有/无符号除法 |
| 逻辑 | `and`, `or`, `xor`, `andc`, `orc`, `eqv`, `nand`, `nor`, `not` | 完整逻辑操作集 |
| 移位 | `shl`, `shr`, `sar`, `rotl`, `rotr` | 移位/旋转 |
| 位操作 | `clz`, `ctz`, `ctpop` | 前导零/尾随零/popcount |
| 位域 | `deposit`, `extract`, `extract2`, `sextract` | 位域插入/提取 |
| 字节序 | `bswap16`, `bswap32`, `bswap64` | 字节序翻转 |
| 条件 | `setcond`, `negsetcond`, `movcond` | 条件设置/移动 |
| 进位链 | `addco`, `addci`, `addcio`, `subbo`, `subbi`, `subbio` | 带进位/借位的加减 |

### 类型转换操作（行 106-110）

| 操作码 | 说明 |
|--------|------|
| `ext_i32_i64` | 32→64 符号扩展 |
| `extu_i32_i64` | 32→64 零扩展 |
| `extrl_i64_i32` | 64→32 低半截取 |
| `extrh_i64_i32` | 64→32 高半截取 |

### TB/分支操作（行 112-116）

| 操作码 | 说明 |
|--------|------|
| `insn_start` | 标记客户机指令开始 |
| `exit_tb` | 退出 TB（返回执行循环） |
| `goto_tb` | 跳转到另一个 TB（TB 链接） |
| `goto_ptr` | 间接跳转到 TB（通过 lookup） |

### 客户机内存操作（行 121-124）

| 操作码 | 说明 |
|--------|------|
| `qemu_ld` | 客户机内存读取（经 softmmu/TLB） |
| `qemu_st` | 客户机内存存储 |
| `qemu_ld2` | 128 位客户机读取 |
| `qemu_st2` | 128 位客户机存储 |

### 向量操作（行 128-179）

| 类别 | 操作码示例 |
|------|-----------|
| 向量移动 | `mov_vec`, `dup_vec`, `dupm_vec` |
| 向量算术 | `add_vec`, `sub_vec`, `mul_vec`, `neg_vec` |
| 向量逻辑 | `and_vec`, `or_vec`, `xor_vec`, `andc_vec`, `orc_vec`, `not_vec` |
| 向量移位 | `shlv_vec`, `shrv_vec`, `sarv_vec`, `rotlv_vec`, `rotrv_vec` |
| 向量比较 | `cmp_vec`, `cmpsel_vec` |
| 向量饱和 | `ssadd_vec`, `usadd_vec`, `sssub_vec`, `ussub_vec` |

### AArch64 目标特定操作码

`tcg/aarch64/tcg-target-opc.h.inc:14-15`：
- `aa64_sshl_vec`：有符号移位左
- `aa64_sli_vec`：移位并插入

---

## 11. TCGLabel — 分支标签

**定义**：`include/tcg/tcg.h:99-111`

```c
struct TCGLabel {
    bool present;              // 是否已在 op 流中出现（set_label）
    bool has_value;            // 是否已解析为地址
    uint16_t id;               // 唯一标识符
    union {
        uintptr_t value;           // 解析后的地址值
        const tcg_insn_unit *value_ptr;  // 指针形式
    } u;
    QSIMPLEQ_HEAD(, TCGLabelUse) branches;  // 引用此标签的分支列表
    QSIMPLEQ_HEAD(, TCGRelocation) relocs;  // 需要回填的重定位
    QSIMPLEQ_ENTRY(TCGLabel) next;          // 标签链表
};
```

**使用流程**：
1. `gen_new_label()` → 创建标签（`present=false`）
2. `tcg_gen_brcond_i64(..., label)` → 添加到 `branches` 列表
3. `gen_set_label(label)` → 标记标签位置（`present=true`）
4. 后端代码生成时解析 → `has_value=true`, `u.value` 指向实际代码位置

---

## 12. TCGContext — 翻译上下文

**定义**：`include/tcg/tcg.h:346-441`

TCGContext 是 TCG 的核心状态容器，每个 vCPU 线程持有一个实例。

**关键字段分组**：

### 内存池
```c
uintptr_t pool_cur, pool_end;           // 当前分配位置
TCGPool *pool_first, *pool_current;     // 池链表
```

### 临时变量管理
```c
int nb_labels;              // 标签数量
int nb_globals;             // 全局 temp 数量（CPU 状态映射）
int nb_temps;               // 总 temp 数量
TCGTemp temps[TCG_MAX_TEMPS]; // temp 数组：globals 在前，locals 在后
TCGTempSet free_temps[TCG_TYPE_COUNT]; // 按类型的空闲 temp 位图
```

### 代码生成
```c
TranslationBlock *gen_tb;   // 当前正在翻译的 TB
tcg_insn_unit *code_buf;    // TB 代码缓冲区起始
tcg_insn_unit *code_ptr;    // 当前代码写入位置
void *code_gen_buffer;      // 全局代码缓冲区
size_t code_gen_buffer_size;
void *code_gen_highwater;   // 刷新阈值
```

### 操作列表
```c
QTAILQ_HEAD(, TCGOp) ops, free_ops;  // 操作链表 + 空闲链表
QSIMPLEQ_HEAD(, TCGLabel) labels;     // 标签链表
TCGOp *emit_before_op;               // 非 NULL 时新 op 插入此 op 之前
```

### 寄存器分配
```c
TCGRegSet reserved_regs;                   // 保留寄存器集
TCGTemp *reg_to_temp[TCG_TARGET_NB_REGS];  // 物理寄存器→temp 反向映射
intptr_t current_frame_offset;             // 栈帧当前偏移
bool carry_live;                           // 进位链优化状态
```

**线程本地访问**（`include/tcg/tcg.h:448`）：
```c
extern __thread TCGContext *tcg_ctx;
```

---

## 13. 临时变量生命周期管理

### 分配

**核心函数**：`tcg/tcg.c:2051-2150`

```c
// 内部分配（指定 kind 和 type）
static TCGTemp *tcg_temp_new_internal(TCGType type, TCGTempKind kind);
// :2051-2110

// 公开 API
TCGv_i32 tcg_temp_new_i32(void);  // → TEMP_EBB, TCG_TYPE_I32   :2113
TCGv_i64 tcg_temp_new_i64(void);  // → TEMP_EBB, TCG_TYPE_I64   :2119
TCGv_vec tcg_temp_new_vec(TCGType type);  //                      :2131
TCGv_ptr tcg_temp_new_ptr(void);  // → TEMP_EBB, TCG_TYPE_PTR   :2137
TCGv_i128 tcg_temp_new_i128(void); // → TEMP_EBB, TCG_TYPE_I128 :2143

// EBB 范围临时变量（显式指定）
TCGv_i64 tcg_temp_ebb_new_i64(void); //                          :2125
```

**分配流程**：
1. 检查 `free_temps[type]` 位图是否有空闲 temp
2. 有 → 复用已有 temp（清除其状态，设置 `temp_allocated = 1`）
3. 无 → 从 `temps[]` 数组末尾分配新 temp（`nb_temps++`）

### 释放

**函数**：`tcg/tcg.c:2188-2230`

```c
TCGv_i64 tcg_temp_free_i64(TCGv_i64 arg);  // :2213-2216
```

**释放规则**：
- 只有 `TEMP_EBB` 类型可以被释放（加回空闲列表）
- `TEMP_TB` 类型的 free 操作被忽略（`tcg/tcg.c:2192-2205`）
- `TEMP_GLOBAL`/`TEMP_FIXED`/`TEMP_CONST` 不可释放

### 全局变量创建

**函数**：`tcg_global_mem_new_i64`（将 CPUState 字段映射为全局 temp）

用法示例（ARM64，`translate-a64.c:91-98`）：
```c
cpu_pc = tcg_global_mem_new_i64(tcg_env,
                                offsetof(CPUARMState, pc), "pc");
for (i = 0; i < 32; i++) {
    cpu_X[i] = tcg_global_mem_new_i64(tcg_env,
                                      offsetof(CPUARMState, xregs[i]),
                                      regnames[i]);
}
```

### 常量 temp

```c
TCGv_i64 tcg_constant_i64(int64_t val);
```
返回 `TEMP_CONST` 类型的 temp，值存储在 `TCGTemp.val` 中。常量 temp 通过 `GHashTable *const_table[TCG_TYPE_COUNT]` 去重。

---

## 14. TCG Op 生成 API

**文件**：`tcg/tcg-op.c`，`include/tcg/tcg-op.h`，`include/tcg/tcg-op-common.h`

### 基本模式

大多数 op 生成函数是薄包装，调用底层的 `tcg_gen_op[N]_i{32,64}` 辅助函数。

**算术/逻辑操作**（`tcg/tcg-op.c:1392-1410`）：
```c
void tcg_gen_add_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b)  // :1392
void tcg_gen_sub_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b)  // :1397
void tcg_gen_and_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b)  // :1402
void tcg_gen_or_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b)
void tcg_gen_xor_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b)
```

**数据移动**（`tcg/tcg-op.c:1325-1335`）：
```c
void tcg_gen_mov_i64(TCGv_i64 ret, TCGv_i64 arg)     // :1325
void tcg_gen_movi_i64(TCGv_i64 ret, int64_t arg)     // :1332
// movi 实际实现为 mov(ret, tcg_constant_i64(arg))
```

**内存访问（宿主内存）**（`tcg/tcg-op.c:1367-1387`）：
```c
void tcg_gen_ld_i64(TCGv_i64 ret, TCGv_ptr base, intptr_t off)  // :1367
void tcg_gen_st_i64(TCGv_i64 arg, TCGv_ptr base, intptr_t off)  // :1387
```

**分支**（`tcg/tcg-op.c:1552-1560`）：
```c
void tcg_gen_brcond_i64(TCGCond c, TCGv_i64 a, TCGv_i64 b, TCGLabel *l)
```

### 客户机内存访问

**定义**：`include/tcg/tcg-op-mem.h:31-65`

```c
void tcg_gen_qemu_ld_i64(TCGv_i64 val, TCGv_i64 addr,
                          int memidx, MemOp memop);
void tcg_gen_qemu_st_i64(TCGv_i64 val, TCGv_i64 addr,
                          int memidx, MemOp memop);
```

**与普通 ld/st 的区别**：
- `qemu_ld/st` 访问**客户机地址空间**，经过 softmmu TLB 查找
- 普通 `ld/st` 访问**宿主内存**（如 CPUState 字段），直接基址+偏移

**MemOp 编码**：
- 低 2 位：大小（`MO_8`/`MO_16`/`MO_32`/`MO_64`）
- 位 2：符号扩展（`MO_SIGN`）
- 位 3：字节序（`MO_BE`/`MO_LE`）
- 高位：对齐要求（`MO_ALIGN`）等

### 条件操作

```c
void tcg_gen_setcond_i64(TCGCond c, TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_movcond_i64(TCGCond c, TCGv_i64 ret,
                          TCGv_i64 c1, TCGv_i64 c2,
                          TCGv_i64 v1, TCGv_i64 v2);
```

---

## 15. ARM64 全局寄存器映射

### AArch64 寄存器（`translate-a64.c:32-37, 82-109`）

```c
static TCGv_i64 cpu_X[32];          // X0-X30 + SP(X31 复用)
static TCGv_i64 cpu_gcspr[4];       // GCS 指针(EL0-EL3)
static TCGv_i64 cpu_pc;             // 程序计数器
static TCGv_i64 cpu_exclusive_high; // 独占监视器高半
```

初始化映射（`a64_translate_init()`, `translate-a64.c:83-109`）：
```c
cpu_pc → offsetof(CPUARMState, pc)                     // :91-93
cpu_X[i] → offsetof(CPUARMState, xregs[i])             // :94-98
cpu_exclusive_high → offsetof(CPUARMState, exclusive_high) // :100-101
cpu_gcspr[i] → offsetof(CPUARMState, cp15.gcspr_el[i]) // :103-108
```

### AArch32 寄存器与标志（`translate.c:49-80`）

```c
static TCGv_i32 cpu_R[16];          // R0-R15
TCGv_i32 cpu_CF, cpu_NF, cpu_VF, cpu_ZF;  // NZCV 标志
TCGv_i64 cpu_exclusive_addr;        // 独占监视器地址
TCGv_i64 cpu_exclusive_val;         // 独占监视器值
```

初始化（`arm_translate_init()`, `translate.c:61-80`）：
```c
cpu_R[i] → offsetof(CPUARMState, regs[i])    // :65-69
cpu_CF → offsetof(CPUARMState, CF)            // :70
cpu_NF → offsetof(CPUARMState, NF)            // :71
cpu_VF → offsetof(CPUARMState, VF)            // :72
cpu_ZF → offsetof(CPUARMState, ZF)            // :73
```

**注意**：NZCV 标志是 **TCGv_i32** 类型（32位），即使在 AArch64 模式也是如此。这是因为标志位的编码只需要 32 位。

---

## 16. NZCV 条件标志缓存体系

### 缓存编码约定

QEMU 不直接存储 PSTATE.NZCV 的 4 个单比特，而是使用**缓存值编码**（`target/arm/cpu.h:270-315`）：

| 标志 | CPUARMState 字段 | TCG 全局 | 编码约定 |
|------|-----------------|----------|----------|
| **N** (Negative) | `NF` | `cpu_NF` | **bit 31** 是 N 标志。值的第 31 位 == 1 表示 N=1 |
| **Z** (Zero) | `ZF` | `cpu_ZF` | **反转缓存**：`ZF == 0` 表示 Z=1（结果为零），`ZF != 0` 表示 Z=0 |
| **C** (Carry) | `CF` | `cpu_CF` | 直接值：`CF == 1` 表示 C=1 |
| **V** (Overflow) | `VF` | `cpu_VF` | **bit 31** 是 V 标志。与 N 相同编码 |

**为什么这样编码？** 因为大多数运算的结果可以直接用作 N 和 Z 的缓存值：
- 结果的 bit 31 自然就是 N 标志
- 结果本身非零则 Z=0，结果为 0 则 Z=1（反转存储避免额外比较）

### PSTATE 重构（`target/arm/cpu.h:1607-1625`）

`pstate_read()` 函数将缓存标志重构为架构 PSTATE 值：
```c
// N: 取 NF 的 bit 31
nzcv |= (env->NF & 0x80000000U);
// Z: ZF == 0 时 Z=1
nzcv |= (env->ZF == 0) << 30;
// C: 直接取
nzcv |= env->CF << 29;
// V: 取 VF 的 bit 31，移到 bit 28
nzcv |= (env->VF >> 3) & (1 << 28);
```

### MRS/MSR NZCV 翻译

**gen_get_nzcv()**（`translate-a64.c:2506-2523`）— 读取 NZCV 到通用寄存器：
```c
static void gen_get_nzcv(TCGv_i64 tcg_rt)
{
    TCGv_i32 nzcv = tcg_temp_new_i32();
    // N: bit 31 直接取
    tcg_gen_andi_i32(nzcv, cpu_NF, (1U << 31));       // :2512
    // Z: 比较 ZF == 0，存入 bit 30
    tcg_gen_setcondi_i32(TCG_COND_EQ, tmp, cpu_ZF, 0); // :2514
    tcg_gen_deposit_i32(nzcv, nzcv, tmp, 30, 1);       // :2515
    // C: 存入 bit 29
    tcg_gen_deposit_i32(nzcv, nzcv, cpu_CF, 29, 1);    // :2517
    // V: bit 31 → bit 28
    tcg_gen_shri_i32(tmp, cpu_VF, 31);                  // :2519
    tcg_gen_deposit_i32(nzcv, nzcv, tmp, 28, 1);        // :2520
    tcg_gen_extu_i32_i64(tcg_rt, nzcv);                 // :2522
}
```

**gen_set_nzcv()**（`translate-a64.c:2525-2543`）— 从通用寄存器写入 NZCV：
```c
static void gen_set_nzcv(TCGv_i64 tcg_rt)
{
    // N: 直接取 bit 31
    tcg_gen_andi_i32(cpu_NF, nzcv, (1U << 31));        // :2533
    // Z: 取 bit 30，反转（Z=1 时 ZF=0）
    tcg_gen_andi_i32(cpu_ZF, nzcv, (1 << 30));         // :2535
    tcg_gen_setcondi_i32(TCG_COND_EQ, cpu_ZF, cpu_ZF, 0); // :2536
    // C: 取 bit 29
    tcg_gen_shri_i32(cpu_CF, ..., 29);                  // :2538-2539
    // V: 取 bit 28 → 左移到 bit 31
    tcg_gen_shli_i32(cpu_VF, ..., 3);                   // :2541-2542
}
```

---

## 17. 前端翻译架构总览

```
 ┌─────────────────────────────────────────────────────┐
 │                 translator_loop()                     │
 │              accel/tcg/translator.c:122                │
 │                                                       │
 │  ┌─────────────┐   ┌──────────────┐   ┌───────────┐ │
 │  │init_disas_  │   │translate_insn│   │ tb_stop   │ │
 │  │  context    │   │  (per insn)  │   │           │ │
 │  └──────┬──────┘   └──────┬───────┘   └─────┬─────┘ │
 │         │                 │                  │       │
 └─────────┼─────────────────┼──────────────────┼───────┘
           │                 │                  │
           ▼                 ▼                  ▼
 ┌────────────────┐ ┌───────────────┐ ┌─────────────────┐
 │ aarch64_tr_    │ │ aarch64_tr_   │ │ aarch64_tr_     │
 │ init_disas_    │ │ translate_insn│ │ tb_stop         │
 │ context        │ │               │ │                 │
 │ :10653         │ │ :10774        │ │ :10876          │
 └────────────────┘ └───────┬───────┘ └─────────────────┘
                            │
                     ┌──────▼──────┐
                     │  insn fetch │
                     │ & decode    │
                     └──────┬──────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        disas_a64()   disas_sme()   disas_sve()
        (decode-a64   (decode-sme   (decode-sve
         .c.inc)       .c.inc)      .c.inc)
              │
              ▼
        trans_XXX() ── 156 个翻译函数
              │
              ▼
        tcg_gen_* API → TCGOp 链表
```

---

## 18. DisasContext — 反汇编上下文

**定义**：`target/arm/tcg/translate.h:39-204`

DisasContext 携带翻译单条指令所需的全部状态：

### 核心翻译状态
```c
DisasContextBase base;      // 基类（pc_first, pc_next, is_jmp, tb 等）
vaddr pc_curr;              // 当前指令地址               :45
vaddr pc_save;              // 上次更新 cpu_pc 时的 PC    :59
uint32_t insn;              // 当前 32 位指令字            :61
```

### CPU 模式信息
```c
int current_el;             // 当前异常等级 (0-3)          :106
bool aarch64;               // AArch64 模式               :109
bool thumb;                 // Thumb 模式                  :110
ARMMMUIdx mmu_idx;          // MMU 索引                    :81
```

### ISA 特性
```c
const ARMISARegisters *isar; // ISA 寄存器（特性检测用）   :41
uint64_t features;           // CPU 特性位                  :108
bool lse2;                   // LSE2 原子操作支持           :111
bool pauth_active;           // 指针认证激活               :138
```

### 条件执行（AArch32）
```c
int condexec_mask;          // IT 块条件执行掩码           :67
int condexec_cond;          // IT 块条件码                 :68
int condjmp;                // 非零=条件跳转挂起           :63
DisasLabel condlabel;       // 条件跳转目标标签            :65
```

### FP/SIMD 状态
```c
int fp_excp_el;             // FP 异常级别 (0=启用)         :86
int sve_excp_el;            // SVE 异常级别                 :87
int sme_excp_el;            // SME 异常级别                 :88
int vl;                     // 当前向量长度（字节）         :90
int svl;                    // 当前流式向量长度             :91
```

### 调试/异常
```c
bool ss_active;             // 单步调试激活                :128
bool pstate_ss;             // PSTATE.SS                   :129
bool is_ldex;               // 当前是独占加载指令           :134
```

---

## 19. translator_loop — 翻译主循环

**定义**：`accel/tcg/translator.c:122-208`

```
translator_loop(cpu, tb, max_insns, pc, host_pc, ops, db, addr_type)
│
├── 初始化 DisasContextBase                     :133-146
│   db->pc_first = pc
│   db->is_jmp = DISAS_NEXT
│
├── ops->init_disas_context(db, cpu)            :148
│   → aarch64_tr_init_disas_context()
│
├── gen_tb_start() — 生成 TB 入口代码           :152
├── ops->tb_start(db, cpu)                      :153
│
├── while (true) {                              :159
│   ├── ops->insn_start(db, cpu)                :161
│   │   → 生成 insn_start op
│   │
│   ├── ops->translate_insn(db, cpu)            :178
│   │   → aarch64_tr_translate_insn()
│   │     ├── 取指：translator_ldl_end()
│   │     ├── 解码：disas_a64(s, insn)
│   │     └── 设置 is_jmp
│   │
│   ├── if (db->is_jmp != DISAS_NEXT) break     :194
│   └── if (buf_full || num >= max) break       :200
│   }
│
├── ops->tb_stop(db, cpu)                       :207
│   → aarch64_tr_tb_stop()
│
└── gen_tb_end()                                :208
```

**翻译终止条件**：
1. `is_jmp != DISAS_NEXT` → 指令设置了跳转/异常
2. `tcg_op_buf_full()` → 操作缓冲区已满
3. `num_insns >= max_insns` → 达到最大指令数

---

## 20. 指令解码树 (decodetree)

### 构建配置

**文件**：`target/arm/tcg/meson.build:1-6`

```python
decodetree.process('a64.decode', extra_args: ['--static-decode=disas_a64'])
```

生成的解码器包含在 `translate-a64.c:79-80`：
```c
#include "decode-sme-fa64.c.inc"
#include "decode-a64.c.inc"
```

### 解码入口

**`aarch64_tr_translate_insn()`**（`translate-a64.c:10774-10874`）中的解码调用：

```c
// 取指
insn = translator_ldl_end(env, &s->base, pc, MO_LE);  // :10816

// 三级解码：a64 → sme → sve
if (!disas_a64(s, insn) &&     // :10861
    !disas_sme(s, insn) &&     // :10862
    !disas_sve(s, insn)) {     // :10863
    unallocated_encoding(s);   // :10864 — 未识别的编码
}
```

### 解码树结构

**文件**：`target/arm/tcg/a64.decode`

解码树使用 QEMU decodetree DSL 定义，按指令分组：

```
# ADD/SUB immediate (行 107-121)
ADD_i   . 0 01 00010 0 ............ ..... .....  @addsub_imm sf=1
ADDS_i  . 0 11 00010 0 ............ ..... .....  @addsub_imm sf=1
SUB_i   . 1 01 00010 0 ............ ..... .....  @addsub_imm sf=1
SUBS_i  . 1 11 00010 0 ............ ..... .....  @addsub_imm sf=1

# Logical immediate (行 131-144)
AND_i   . 00 100100 0 ...... ...... ..... .....  @logic_imm sf=1
ORR_i   . 01 100100 0 ...... ...... ..... .....  @logic_imm sf=1
EOR_i   . 10 100100 0 ...... ...... ..... .....  @logic_imm sf=1
ANDS_i  . 11 100100 0 ...... ...... ..... .....  @logic_imm sf=1

# Branches (行 189-214)
B       0 00101 ..........................        @branch
BL      1 00101 ..........................        @branch
```

decodetree 工具自动生成 `disas_a64()` 函数，该函数根据指令位模式分发到对应的 `trans_XXX()` 函数。

---

## 21. 数据处理指令翻译模式

### ADD/SUB (立即数)

解码树条目（`a64.decode:107-121`）匹配后调用 `trans_ADD_i()` / `trans_SUB_i()`：

**翻译模式**：
```
ARM64: ADD X0, X1, #imm        → tcg_gen_addi_i64(cpu_X[0], cpu_X[1], imm)
ARM64: ADD W0, W1, #imm        → tcg_gen_addi_i64(tmp, cpu_X[1], imm)
                                   tcg_gen_ext32u_i64(cpu_X[0], tmp)
ARM64: ADDS X0, X1, #imm       → gen_add_CC(sf=1, cpu_X[0], cpu_X[1], imm_temp)
```

**32 位操作的关键**：虽然 W 寄存器是 32 位，但 TCG 中统一使用 I64 类型。32 位结果通过 `tcg_gen_ext32u_i64()` 零扩展到 64 位。

### 逻辑操作 (立即数)

**位掩码立即数解码**（`translate-a64.c:5099`）：
```c
static bool logic_imm_decode_wmask(uint64_t *result, ...)
```

ARM64 逻辑立即数使用复杂的 N:immr:imms 编码，`logic_imm_decode_wmask()` 将其解码为 64 位掩码值。

**翻译模式**：
```
AND X0, X1, #mask → tcg_gen_andi_i64(cpu_X[0], cpu_X[1], mask)
ANDS X0, X1, #mask → tcg_gen_andi_i64(cpu_X[0], cpu_X[1], mask)
                      gen_logic_CC(sf, cpu_X[0])
```

### 移位操作

**移位类型枚举**（`translate-a64.c:46-51`）：
```c
enum a64_shift_type {
    A64_SHIFT_TYPE_LSL = 0,    // 逻辑左移
    A64_SHIFT_TYPE_LSR = 1,    // 逻辑右移
    A64_SHIFT_TYPE_ASR = 2,    // 算术右移
    A64_SHIFT_TYPE_ROR = 3     // 循环右移
};
```

**TCG 映射**：
```
LSL → tcg_gen_shl_i64()
LSR → tcg_gen_shr_i64()
ASR → tcg_gen_sar_i64()
ROR → tcg_gen_rotr_i64()
```

### 乘法

```
MADD X0, X1, X2, X3  →  tmp = X1 * X2; X0 = tmp + X3
    tcg_gen_mul_i64(tmp, cpu_X[1], cpu_X[2])
    tcg_gen_add_i64(cpu_X[0], tmp, cpu_X[3])

MSUB X0, X1, X2, X3  →  tmp = X1 * X2; X0 = X3 - tmp
    tcg_gen_mul_i64(tmp, cpu_X[1], cpu_X[2])
    tcg_gen_sub_i64(cpu_X[0], cpu_X[3], tmp)
```

---

## 22. 条件码生成详解

### gen_logic_CC — 逻辑操作标志设置

**文件**：`translate-a64.c:1000-1010`

```c
static inline void gen_logic_CC(int sf, TCGv_i64 result)
{
    if (sf) {
        gen_set_NZ64(result);        // 64 位：拆分为高低 32 位设置 NZ
    } else {
        tcg_gen_extrl_i64_i32(cpu_ZF, result);  // 32 位：取低 32 位
        tcg_gen_mov_i32(cpu_NF, cpu_ZF);         // NF = ZF（共享结果值）
    }
    tcg_gen_movi_i32(cpu_CF, 0);     // C 清零
    tcg_gen_movi_i32(cpu_VF, 0);     // V 清零
}
```

### gen_add_CC / gen_add64_CC — 加法标志设置

**文件**：`translate-a64.c:1013-1059`

64 位加法标志（`gen_add64_CC`, `:1013-1033`）：
```c
// C: 通过 add2 (128位加法) 的高位获取进位
tcg_gen_movi_i64(tmp, 0);
tcg_gen_add2_i64(result, flag, t0, tmp, t1, tmp);  // :1021
tcg_gen_extrl_i64_i32(cpu_CF, flag);                 // :1023

// N, Z: 从 result 设置
gen_set_NZ64(result);                                 // :1025

// V: (result ^ t0) & ~(t0 ^ t1) 的 bit 31
tcg_gen_xor_i64(flag, result, t0);     // :1027
tcg_gen_xor_i64(tmp, t0, t1);         // :1028
tcg_gen_andc_i64(flag, flag, tmp);     // :1029 — andc = A & ~B
tcg_gen_extrh_i64_i32(cpu_VF, flag);   // :1030
```

**溢出检测公式解析**：
- 有符号溢出发生在：两个同号操作数相加，结果符号不同
- `(result ^ t0)` → 结果与 t0 符号不同时 bit 31 = 1
- `(t0 ^ t1)` → t0 与 t1 符号不同时 bit 31 = 1
- `~(t0 ^ t1)` → t0 与 t1 同号时 bit 31 = 1
- 最终：同号相加且结果变号 → V=1

### gen_sub_CC / gen_sub64_CC — 减法标志设置

**文件**：`translate-a64.c:1062-1110`

64 位减法标志（`gen_sub64_CC`, `:1062-1082`）：
```c
// 计算 result = t0 - t1
tcg_gen_sub_i64(result, t0, t1);                     // :1069

// N, Z
gen_set_NZ64(result);                                 // :1071

// C: 无符号不借位 = t0 >= t1
tcg_gen_setcond_i64(TCG_COND_GEU, flag, t0, t1);    // :1073
tcg_gen_extrl_i64_i32(cpu_CF, flag);                  // :1074

// V: (result ^ t0) & (t0 ^ t1) 的 bit 31
tcg_gen_xor_i64(flag, result, t0);    // :1076
tcg_gen_xor_i64(tmp, t0, t1);        // :1078
tcg_gen_and_i64(flag, flag, tmp);     // :1079 — 注意是 AND 不是 ANDC
tcg_gen_extrh_i64_i32(cpu_VF, flag);  // :1080
```

**减法溢出公式**：与加法对称：
- `(result ^ t0)` → 结果与被减数符号不同
- `(t0 ^ t1)` → 被减数与减数符号不同（即不同号相减）
- 两者都为 1 → 不同号相减且结果符号异常 → V=1

### gen_adc_CC — 带进位加法标志设置

**文件**：`translate-a64.c:1126-1160`

使用 `addco`/`addcio` 进位链操作码（TCG 原生进位支持）：
```c
tcg_gen_addco_i64(result, t0, t1);    // 加法+输出进位
tcg_gen_addcio_i64(result, result, cf); // 加进位+输入/输出进位
```

---

## 23. 条件码评估 — arm_test_cc

**定义**：`translate.c:641-720`

`arm_test_cc()` 将 ARM 条件码（0-15）转换为 TCG 比较操作：

```c
void arm_test_cc(DisasCompare *cmp, int cc)
{
    switch (cc) {
    case 0: /* EQ: Z=1 */
    case 1: /* NE: Z=0 */
        cond = TCG_COND_EQ;
        value = cpu_ZF;           // ZF==0 表示 Z=1（反转缓存）
        break;

    case 2: /* CS/HS: C=1 */
    case 3: /* CC/LO: C=0 */
        cond = TCG_COND_NE;
        value = cpu_CF;           // CF!=0 表示 C=1
        break;

    case 4: /* MI: N=1 */
    case 5: /* PL: N=0 */
        cond = TCG_COND_LT;
        value = cpu_NF;           // NF<0 表示 bit31=1 即 N=1
        break;

    case 6: /* VS: V=1 */
    case 7: /* VC: V=0 */
        cond = TCG_COND_LT;
        value = cpu_VF;           // VF<0 表示 bit31=1 即 V=1
        break;

    case 8: /* HI: C=1 && Z=0 */
    case 9: /* LS: C=0 || Z=1 */
        value = -CF & ZF;        // 构造组合值
        break;

    case 10: /* GE: N==V */
    case 11: /* LT: N!=V */
        value = NF ^ VF;         // 异或后检查符号位
        cond = TCG_COND_GE;      // >=0 表示 N==V
        break;

    case 12: /* GT: Z=0 && N==V */
    case 13: /* LE: Z=1 || N!=V */
        // ~(NF^VF) 的符号位传播后 AND ZF
        value = ZF & ~((NF^VF) >> 31);
        break;

    case 14: /* AL: always */
    case 15: /* AL: always */
        cond = TCG_COND_ALWAYS;
        break;
    }

    if (cc & 1) cond = tcg_invert_cond(cond);  // 奇数条件码取反
}
```

**精妙设计**：
- 利用缓存编码的特性，每个条件只需 1-3 个 TCG 操作
- 奇数条件码是偶数的取反，统一通过 `tcg_invert_cond` 处理
- `TCG_COND_ALWAYS` 在优化阶段可以直接折叠掉分支

---

## 24. 条件比较与条件选择 (CCMP/CSEL)

### CCMP (Conditional Compare)

**定义**：`translate-a64.c:9115-9192`

CCMP 是 ARM64 最复杂的条件指令之一：如果条件为真，执行比较并设置标志；如果条件为假，将标志设置为立即数 nzcv。

**翻译逻辑**：
```
CCMP Xn, #imm5, #nzcv, cond
│
├── 评估条件 cond → T0 = !cond (0 或 1)
│   arm_test_cc(&c, cond)                              // :9127
│   tcg_gen_setcondi_i32(invert(c.cond), T0, c.value, 0) // :9128
│
├── 执行比较（无论条件如何都执行）
│   if (op) gen_sub_CC(sf, tmp, Xn, Y)  // CCMP        // :9140
│   else    gen_add_CC(sf, tmp, Xn, Y)  // CCMN        // :9142
│
├── 构造掩码
│   T1 = -T0  (cond false → 0xFFFFFFFF, true → 0)     // :9151
│   T2 = T0 - 1 (cond false → 0, true → 0xFFFFFFFF)   // :9152
│
└── 合并标志：对每个标志位
    if (nzcv bit set):
        flag |= T1    (false 时强制设为 1)              // :9157
    else:
        flag &= T2    (false 时强制清零)                // :9162
        或 flag &= ~T1 (用 andc)                        // :9160
```

### CSEL (Conditional Select) 家族

**定义**：`translate-a64.c:9195-9229`

```
CSEL  Xd, Xn, Xm, cond → Xd = cond ? Xn : Xm
CSINC Xd, Xn, Xm, cond → Xd = cond ? Xn : Xm + 1
CSINV Xd, Xn, Xm, cond → Xd = cond ? Xn : ~Xm
CSNEG Xd, Xn, Xm, cond → Xd = cond ? Xn : -Xm
```

**翻译模式**：
```c
// 标准 CSEL
tcg_gen_movcond_i64(c.cond, tcg_rd, c.value, zero, t_true, t_false);

// 特殊情况 CSET (CSINC XZR, XZR, cond) — 简化为 setcond
tcg_gen_setcond_i64(tcg_invert_cond(c.cond), tcg_rd, c.value, zero);

// 特殊情况 CSETM (CSINV XZR, XZR, cond) — 简化为 negsetcond
tcg_gen_negsetcond_i64(tcg_invert_cond(c.cond), tcg_rd, c.value, zero);
```

---

## 25. 内存访问翻译

### GPR 加载/存储核心函数

**存储**（`translate-a64.c:1169-1189`）：
```c
static void do_gpr_st_memidx(DisasContext *s, TCGv_i64 source,
                              TCGv_i64 tcg_addr, MemOp memop, int memidx, ...)
{
    tcg_gen_qemu_st_i64(source, tcg_addr, memidx, memop);  // :1175
    // 可选：生成 ISS (Instruction Specific Syndrome) 用于调试
}
```

**加载**（`translate-a64.c:1204-1228`）：
```c
static void do_gpr_ld_memidx(DisasContext *s, TCGv_i64 dest,
                              TCGv_i64 tcg_addr, MemOp memop, ...)
{
    tcg_gen_qemu_ld_i64(dest, tcg_addr, memidx, memop);    // :1209
    if (extend && (memop & MO_SIGN)) {
        tcg_gen_ext32u_i64(dest, dest);  // 32 位有符号加载后零扩展到 64 位
    }
}
```

### 寻址模式翻译

ARM64 支持多种寻址模式，翻译为地址计算 + qemu_ld/st：

```
[Xn, #imm]  → addr = Xn + imm; qemu_ld(dest, addr)
[Xn, #imm]! → addr = Xn + imm; qemu_ld(dest, addr); Xn = addr  (pre-index)
[Xn], #imm  → qemu_ld(dest, Xn); Xn = Xn + imm               (post-index)
[Xn, Xm]    → addr = Xn + Xm; qemu_ld(dest, addr)
[Xn, Xm, LSL #n] → addr = Xn + (Xm << n); qemu_ld(dest, addr)
```

### 独占加载/存储 (LDXR/STXR)

**gen_load_exclusive()**（`translate-a64.c:3241-3284`）：
```c
// 1. 加载值到 cpu_exclusive_val
tcg_gen_qemu_ld_i64(cpu_exclusive_val, clean_addr, idx, memop);  // :3280
// 2. 将值复制到目标寄存器
tcg_gen_mov_i64(cpu_reg(s, rt), cpu_exclusive_val);               // :3281
// 3. 记录独占地址
tcg_gen_mov_i64(cpu_exclusive_addr, clean_addr);                  // :3283
```

**gen_store_exclusive()**（`translate-a64.c:3286-3340+`）：
```c
// 伪代码：
if (cpu_exclusive_addr == addr) {
    if (cpu_exclusive_val == *addr) {  // 值未变
        *addr = Rt;
        Rd = 0;  // 成功
    }
} else {
    Rd = 1;  // 失败
}
cpu_exclusive_addr = -1;  // 清除监视器
```

---

## 26. 分支与控制流翻译

### 无条件分支 (B / BL)

解码树（`a64.decode:194-195`）：
```
B   0 00101 .......................... @branch
BL  1 00101 .......................... @branch
```

**翻译**：
```
B target   → pc = pc_curr + offset
             is_jmp = DISAS_NEXT (如果在 TB 内跳转，使用 goto_tb)

BL target  → X30 = pc + 4          (保存返回地址)
             pc = pc_curr + offset
             is_jmp = DISAS_NORETURN
```

### 寄存器分支 (BR / BLR / RET)

**trans_BR()**（`translate-a64.c:1800-1806`）：
```c
static bool trans_BR(DisasContext *s, arg_r *a)
{
    set_btype_for_br(s, a->rn);
    gen_a64_set_pc(s, cpu_reg(s, a->rn));   // pc = Xn
    s->base.is_jmp = DISAS_JUMP;            // 间接跳转
    return true;
}
```

**trans_BLR()**（`translate-a64.c:1808-1822`）：
```c
static bool trans_BLR(DisasContext *s, arg_r *a)
{
    gen_pc_plus_diff(s, link, 4);           // link = pc + 4
    gen_a64_set_pc(s, cpu_reg(s, a->rn));   // pc = Xn
    tcg_gen_mov_i64(cpu_reg(s, 30), link);  // X30 = link
    s->base.is_jmp = DISAS_JUMP;
    return true;
}
```

**trans_RET()**（`translate-a64.c:1824-1830+`）：
```c
static bool trans_RET(DisasContext *s, arg_r *a)
{
    gen_a64_set_pc(s, cpu_reg(s, a->rn));   // 默认 X30
    s->base.is_jmp = DISAS_JUMP;
    return true;
}
```

### 条件分支 (B.cond / CBZ / TBZ)

**trans_B_cond()**（`translate-a64.c:1756-1775`）：
```c
static bool trans_B_cond(DisasContext *s, arg_B_cond *a)
{
    // 1. 评估条件
    arm_test_cc(&c, a->cond);                    // :1766
    // 2. 如果 ALWAYS 条件，直接无条件跳转
    // 3. 否则生成条件分支
    gen_goto_tb(s, 0, a->imm);   // 条件为真时的目标
    gen_goto_tb(s, 1, 4);        // 条件为假时继续
}
```

**trans_CBZ()**（`translate-a64.c:1720-1735`）：
```c
// 比较并分支：if (Xn == 0) goto target
tcg_gen_brcondi_i64(TCG_COND_EQ, cpu_reg(s, rn), 0, label);
```

**trans_TBZ()**（`translate-a64.c:1737-1754`）：
```c
// 测试位并分支：if (Xn & (1<<bit) == 0) goto target
tcg_gen_andi_i64(tmp, cpu_reg(s, rn), 1ULL << bit);
tcg_gen_brcondi_i64(TCG_COND_EQ, tmp, 0, label);
```

---

## 27. 异常与系统指令翻译

### SVC (Supervisor Call)

**定义**：`translate-a64.c:3155-3171`

```c
static bool trans_SVC(DisasContext *s, arg_i *a)
{
    uint32_t syndrome = syn_aa64_svc(a->imm);
    if (s->fgt_svc) {                               // :3164
        gen_exception_insn_el(s, 0, EXCP_UDEF, syndrome, 2);
        return true;
    }
    gen_ss_advance(s);                               // :3168 — 单步推进
    gen_exception_insn(s, 4, EXCP_SWI, syndrome);    // :3169 — 产生 SWI 异常
    return true;
}
```

**关键点**：
- `gen_ss_advance` 在异常前推进单步状态机（架构要求）
- `EXCP_SWI` 是 SVC 对应的异常类型
- `syndrome` 编码了 SVC 的立即数和异常类型信息

### HVC / SMC

**trans_HVC()**（`translate-a64.c:3173-3191`）：
```c
// EL0 不可调用 HVC
if (s->current_el == 0) { unallocated_encoding(s); return true; }
gen_helper_pre_hvc(tcg_env);     // 运行时检查是否被 trap
gen_ss_advance(s);
gen_exception_insn_el(s, 4, EXCP_HVC, syndrome, target_el);
```

**trans_SMC()**（`translate-a64.c:3193-3205`）：
```c
// EL0 不可调用 SMC
gen_helper_pre_smc(tcg_env, ...);  // 运行时检查
gen_ss_advance(s);
gen_exception_insn_el(s, 4, EXCP_SMC, syndrome, 3);  // 始终到 EL3
```

### BRK (断点)

**trans_BRK()**（`translate-a64.c:3207-3211`）：
```c
static bool trans_BRK(DisasContext *s, arg_i *a)
{
    gen_exception_bkpt_insn(s, syn_aa64_bkpt(a->imm));
    return true;
}
```

### 未定义指令

**unallocated_encoding()**（`translate.c:1155-1159`）：
```c
static void unallocated_encoding(DisasContext *s)
{
    gen_exception_insn(s, 0, EXCP_UDEF, syn_uncategorized());
}
```

### 内存屏障

**trans_DSB_DMB()**（`translate-a64.c:2243-2261`）：
```c
// DSB/DMB → TCG 内存屏障
tcg_gen_mb(barrier_type);
// barrier_type 根据 CRm 字段编码（LD/ST/FULL）
```

**trans_ISB()**（`translate-a64.c:2272-2281`）：
```c
// ISB → 结束 TB + 刷新管线
s->base.is_jmp = DISAS_UPDATE_EXIT;
```

---

## 28. TB 终止与退出策略

### DisasJumpType 枚举

**通用定义**（`include/exec/translator.h:34-50`）：
```c
typedef enum DisasJumpType {
    DISAS_NEXT,         // 继续下一条指令
    DISAS_TOO_MANY,     // 达到最大指令数
    DISAS_NORETURN,     // 后续代码不可达
    DISAS_TARGET_0..11, // 目标特定扩展
} DisasJumpType;
```

**ARM 特定别名**（`translate.h:331-358`）：

| 别名 | 基础值 | 含义 |
|------|--------|------|
| `DISAS_JUMP` | TARGET_0 | PC 已动态修改（间接跳转） |
| `DISAS_UPDATE_EXIT` | TARGET_1 | 需更新 PC 后退出 |
| `DISAS_WFI` | TARGET_2 | WFI 指令 |
| `DISAS_SWI` | TARGET_3 | SVC 指令 |
| `DISAS_WFE` | TARGET_4 | WFE 指令 |
| `DISAS_HVC` | TARGET_5 | HVC 指令 |
| `DISAS_SMC` | TARGET_6 | SMC 指令 |
| `DISAS_YIELD` | TARGET_7 | YIELD 指令 |
| `DISAS_BX_EXCRET` | TARGET_8 | BX 异常返回 (AArch32) |
| `DISAS_EXIT` | TARGET_9 | 直接退出（不更新 PC） |
| `DISAS_UPDATE_NOCHAIN` | TARGET_10 | 更新 PC，不链接 TB |

### aarch64_tr_tb_stop 退出代码生成

**定义**：`translate-a64.c:10876-10944`

```
tb_stop(is_jmp)
│
├── ss_active (单步调试模式)
│   ├── default → 更新 PC + 步进完成异常
│   ├── EXIT/JUMP → 步进完成异常
│   └── NORETURN → 无操作
│
└── 正常模式
    ├── NEXT / TOO_MANY
    │   → gen_goto_tb(1, 4)     // TB 链接到下一条指令
    │
    ├── UPDATE_EXIT
    │   → gen_a64_update_pc(4)
    │   → tcg_gen_exit_tb(NULL, 0)  // 退出到执行循环
    │
    ├── EXIT
    │   → tcg_gen_exit_tb(NULL, 0)  // 直接退出
    │
    ├── UPDATE_NOCHAIN
    │   → gen_a64_update_pc(4)
    │   → tcg_gen_lookup_and_goto_ptr()  // 间接查找+跳转
    │
    ├── JUMP
    │   → tcg_gen_lookup_and_goto_ptr()  // 间接查找+跳转
    │
    ├── NORETURN / SWI
    │   → 无操作（已在指令翻译中处理）
    │
    ├── WFI
    │   → gen_helper_wfi() + exit_tb
    │
    ├── WFE → gen_helper_wfe()
    └── YIELD → gen_helper_yield()
```

**三种 TB 退出方式**：

1. **`tcg_gen_goto_tb(n)`** — TB 链接（最快）
   - 直接跳转到已链接的下一个 TB
   - 适用于已知目标地址的直接跳转

2. **`tcg_gen_lookup_and_goto_ptr()`** — 间接查找
   - 从 TB 哈希表查找目标 TB 并跳转
   - 适用于间接分支（BR/BLR/RET）

3. **`tcg_gen_exit_tb(NULL, 0)`** — 退出到执行循环
   - 返回 QEMU 主循环
   - 适用于需要检查中断、异常的场景

---

## 29. Helper 函数机制

### 定义宏

**文件**：`target/arm/tcg/helper-a64-defs.h`

```c
DEF_HELPER_FLAGS_2(udiv64, TCG_CALL_NO_RWG_SE, i64, i64, i64)
DEF_HELPER_FLAGS_2(sdiv64, TCG_CALL_NO_RWG_SE, s64, s64, s64)
DEF_HELPER_FLAGS_1(rbit64, TCG_CALL_NO_RWG_SE, i64, i64)
DEF_HELPER_2(msr_i_spsel, void, env, i32)
DEF_HELPER_2(msr_i_daifset, void, env, i32)
DEF_HELPER_2(msr_i_daifclear, void, env, i32)
```

**宏格式**：
```
DEF_HELPER_FLAGS_N(name, flags, ret_type, arg1_type, ..., argN_type)
//                  |      |       |          |
//               函数名  调用属性  返回类型    参数类型列表
```

**常用标志**：
- `TCG_CALL_NO_RWG_SE`：无副作用（不读写全局状态，安全可消除）
- `TCG_CALL_NO_WG`：不写全局状态（但可读）
- 无标志：可能有任意副作用

### 实现

**文件**：`target/arm/tcg/helper-a64.c`

```c
uint64_t HELPER(udiv64)(uint64_t num, uint64_t den)
{
    if (den == 0) {
        return 0;   // ARM64: 除零返回 0，不产生异常
    }
    return num / den;
}
```

### 生成的包装

`DEF_HELPER` 宏自动生成 `gen_helper_xxx()` 包装函数，翻译器调用这些包装函数来生成 helper 调用的 TCG 操作：

```c
// 翻译器中的调用
gen_helper_udiv64(cpu_X[rd], cpu_X[rn], cpu_X[rm]);
// 等价于生成：
// INDEX_op_call [helper_udiv64, cpu_X[rd], cpu_X[rn], cpu_X[rm]]
```

### Helper 调用底层

**`tcg_gen_callN()`**（`tcg/tcg.c:2502-2593`）：
- 分配 `INDEX_op_call` 操作
- 编码参数（输入/输出 temp + 常量参数）
- 标记调用破坏的寄存器（`TCG_OPF_CALL_CLOBBER`）

---

## 30. 完整翻译示例：ADD X0, X1, X2 全流程

### 步骤 1：取指与解码

```
pc = 0x1000, insn = 0x8B020020 (ADD X0, X1, X2)

aarch64_tr_translate_insn()                          // translate-a64.c:10774
├── insn = translator_ldl_end(..., pc, MO_LE)        // :10816
├── disas_a64(s, 0x8B020020)                         // :10861
│   └── 解码树匹配 → trans_ADD_r(s, arg)
```

### 步骤 2：trans_ADD_r 翻译

```
trans_ADD_r(s, {sf=1, rd=0, rn=1, rm=2, shift_type=0, imm6=0})
│
├── 读取源操作数
│   tcg_rn = cpu_X[1]          // TCGv_i64，全局 temp
│   tcg_rm = cpu_X[2]          // TCGv_i64，全局 temp
│
├── 移位处理（shift=0, imm=0 → 无移位）
│   tcg_rm_shifted = tcg_rm    // 直接使用
│
├── 生成加法操作
│   tcg_gen_add_i64(cpu_X[0], cpu_X[1], cpu_X[2])
│   │
│   └── 底层：
│       TCGOp *op = tcg_op_alloc(INDEX_op_add, 3)
│       op->args[0] = temp_idx(cpu_X[0])  // 输出
│       op->args[1] = temp_idx(cpu_X[1])  // 输入1
│       op->args[2] = temp_idx(cpu_X[2])  // 输入2
│       QTAILQ_INSERT_TAIL(&tcg_ctx->ops, op)
│
└── 返回 true（解码成功）
```

### 步骤 3：IR 形式

```
insn_start  0x1000, 0, 0
add_i64     X0, X1, X2
```

### 步骤 4：优化

```
tcg_optimize(s)
└── 遍历 ops：
    op = INDEX_op_add
    ├── fold_add()
    │   ├── 检查 X1, X2 是否常量 → 否（全局 temp）
    │   ├── 检查 X2 == 0 → 未知
    │   └── fold_masks_zosa_int() → 更新 z_mask/o_mask
    └── 无法折叠，保持原样
```

### 步骤 5：活跃性分析

```
liveness_pass_1(s)（反向遍历）
└── op = add_i64 X0, X1, X2
    ├── X0 是输出：标记 X0 为 live
    ├── X1, X2 是输入：标记为 live
    ├── 如果后续无人使用 X0 → 标记 DEAD_ARG
    └── 如果 X0 是 GLOBAL → 可能需要 SYNC_ARG
```

### 步骤 6：寄存器分配与代码生成

```
tcg_reg_alloc_op()
├── 为 X1 分配/加载宿主寄存器 (如 x1)
├── 为 X2 分配/加载宿主寄存器 (如 x2)
├── 为 X0 分配输出寄存器 (如 x0)
└── 发射宿主指令：
    ADD x0, x1, x2    (ARM64 宿主的原生 ADD 指令)
```

### 完整 Pipeline 图

```
 ARM64 客户机指令        TCG IR              优化后            宿主机器码
━━━━━━━━━━━━━━━    ━━━━━━━━━━━━━━━     ━━━━━━━━━━━━━    ━━━━━━━━━━━━━━
ADD X0, X1, X2  →  add_i64 X0,X1,X2  →  add_i64 X0,X1,X2  →  ADD x0,x1,x2
                   (3个全局temp)         (不可优化)           (原生指令)
```

---

## 31. 附录 A：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `include/tcg/tcg.h` | ~600 | TCG 核心类型定义（TCGTemp/TCGOp/TCGContext） |
| `include/tcg/tcg-opc.h` | 185 | 126 个操作码定义（DEF 宏） |
| `include/tcg/tcg-op.h` | ~50 | TCGv 别名，包含 tcg-op-common.h |
| `include/tcg/tcg-op-common.h` | ~400 | temp 创建/标签/分支生成 API |
| `include/tcg/tcg-op-mem.h` | ~65 | qemu_ld/st 客户机内存 API |
| `tcg/tcg-op.c` | ~2000 | Op 生成函数实现（mov/add/sub/brcond 等） |
| `tcg/tcg.c` | ~6600 | 核心引擎（分配/活跃性/寄存器分配/代码生成） |
| `accel/tcg/translator.c` | ~250 | 翻译主循环 translator_loop() |
| `include/exec/translator.h` | ~150 | DisasJumpType/DisasContextBase/TranslatorOps |
| `target/arm/tcg/translate.h` | ~360 | DisasContext 定义 + ARM DisasJumpType 别名 |
| `target/arm/tcg/translate.c` | ~6000 | ARM32 翻译器 + NZCV 全局 + arm_test_cc |
| `target/arm/tcg/translate-a64.c` | 10961 | ARM64 翻译器（156 个 trans_ 函数） |
| `target/arm/tcg/a64.decode` | ~1500 | ARM64 解码树定义 |
| `target/arm/tcg/helper-a64-defs.h` | ~400 | ARM64 helper 定义 |
| `target/arm/tcg/helper-a64.c` | ~800 | ARM64 helper 实现 |
| `target/arm/cpu.h` | ~2000 | CPUARMState + PSTATE 重构函数 |

---

## 32. 附录 B: TCGv 与 TCGTemp 映射关系图

```
用户视角（类型安全）          内部表示（统一索引）
━━━━━━━━━━━━━━━━━━━━     ━━━━━━━━━━━━━━━━━━━━━━
                          TCGContext.temps[]:
TCGv_i64 cpu_X[0]   ──→  [0] {kind=GLOBAL, type=I64,
                               mem_base=env, offset=xregs[0]}

TCGv_i64 cpu_X[1]   ──→  [1] {kind=GLOBAL, type=I64,
                               mem_base=env, offset=xregs[1]}
  ...                       ...

TCGv_i64 cpu_pc      ──→  [34] {kind=GLOBAL, type=I64,
                                mem_base=env, offset=pc}

TCGv_i32 cpu_NF      ──→  [35] {kind=GLOBAL, type=I32,
                                mem_base=env, offset=NF}
  ...                       ...
                          ─── nb_globals 分界线 ───

TCGv_i64 tmp         ──→  [N] {kind=EBB, type=I64,
  (tcg_temp_new_i64)          temp_allocated=1}

TCGv_i64 const_42    ──→  [M] {kind=CONST, type=I64,
  (tcg_constant_i64)          val=42}

转换方式：
  TCGv_i64 v = temp_tcgv_i64(&temps[idx])
  TCGTemp *t = tcgv_i64_temp(v)
  // v 实际上是 (TCGv_i64)((void*)t - (void*)tcg_ctx)
```

---

> **文档版本**：v1.0
> **源码版本**：QEMU 11.0.50 (commit 基于 2025-07 主线)
> **分析工具**：zoekt + ctags + cscope + clangd + 手动源码验证
> **交叉验证**：所有行号均经 grep -n 验证
