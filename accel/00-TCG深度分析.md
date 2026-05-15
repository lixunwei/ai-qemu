# TCG（Tiny Code Generator）深度分析

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 关键 commit：`5e3906dcfa`（tcg-target 指令编码重命名）、`744eb39667`（optimize deposit 展开）、`e4cebfc664`（extract2 降级优化）  
> 参考文档：`docs/devel/tcg.rst`、`docs/devel/decodetree.rst`  

---

## 目录

- [第一部分：TCG 中间表示（IR）](#第一部分tcg-中间表示ir)
  - [1. TCG 概述与架构定位](#1-tcg-概述与架构定位)
  - [2. 核心数据结构](#2-核心数据结构)
  - [3. TCG 操作码完整分类](#3-tcg-操作码完整分类)
  - [4. TCG 类型系统](#4-tcg-类型系统)
  - [5. 条件码系统](#5-条件码系统)
  - [6. IR 生成 API（tcg_gen_*）](#6-ir-生成-apitcg_gen_)
  - [7. Helper 函数机制](#7-helper-函数机制)
- [第二部分：ARM64 指令翻译](#第二部分arm64-指令翻译)
  - [8. 翻译入口与 TranslatorOps](#8-翻译入口与-translatorops)
  - [9. DisasContext 上下文](#9-disascontext-上下文)
  - [10. Decodetree 解码器](#10-decodetree-解码器)
  - [11. 数据处理指令翻译](#11-数据处理指令翻译)
  - [12. 内存访问指令翻译](#12-内存访问指令翻译)
  - [13. 分支指令翻译](#13-分支指令翻译)
  - [14. 系统指令翻译](#14-系统指令翻译)
  - [15. 条件标志与异常处理](#15-条件标志与异常处理)
  - [16. 翻译块边界](#16-翻译块边界)
- [第三部分：TCG 优化](#第三部分tcg-优化)
  - [17. 优化器架构](#17-优化器架构)
  - [18. 常量折叠与传播](#18-常量折叠与传播)
  - [19. 拷贝传播](#19-拷贝传播)
  - [20. 死码消除与分支优化](#20-死码消除与分支优化)
  - [21. 强度削减与扩展优化](#21-强度削减与扩展优化)
- [第四部分：寄存器分配与代码生成](#第四部分寄存器分配与代码生成)
  - [22. 活跃性分析](#22-活跃性分析)
  - [23. 寄存器分配算法](#23-寄存器分配算法)
  - [24. AArch64 后端代码生成](#24-aarch64-后端代码生成)
  - [25. 客户机内存访问代码生成](#25-客户机内存访问代码生成)
- [第五部分：执行引擎](#第五部分执行引擎)
  - [26. TranslationBlock 管理](#26-translationblock-管理)
  - [27. CPU 执行循环](#27-cpu-执行循环)
  - [28. 软件 TLB](#28-软件-tlb)
  - [29. TB 链接与直接跳转](#29-tb-链接与直接跳转)
  - [30. MTTCG 多线程 TCG](#30-mttcg-多线程-tcg)
- [附录](#附录)
  - [A. TCG 完整操作码速查表](#a-tcg-完整操作码速查表)
  - [B. 关键源文件索引](#b-关键源文件索引)
  - [C. 一条 ARM64 指令的完整生命周期](#c-一条-arm64-指令的完整生命周期)

---

# 第一部分：TCG 中间表示（IR）

## 1. TCG 概述与架构定位

TCG（Tiny Code Generator）是 QEMU 的软件动态二进制翻译引擎，最初由 Fabrice Bellard 于 2008 年设计，用于替代 QEMU 早期的 dyngen 方案。TCG 在 QEMU 加速器架构中的定位如下：

```
┌─────────────────────────────────────────────────────┐
│                    Guest 应用                        │
├─────────────────────────────────────────────────────┤
│              Guest OS / 裸机固件                     │
├─────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │   KVM 加速器      │  │     TCG 加速器            │ │
│  │  (硬件虚拟化)      │  │  (软件二进制翻译)          │ │
│  └──────────────────┘  └──────────────────────────┘ │
│                    QEMU 核心                         │
├─────────────────────────────────────────────────────┤
│                  Host OS / 硬件                      │
└─────────────────────────────────────────────────────┘
```

**TCG 的核心工作流是一个三阶段管道**：

```
Guest 指令 ──→ [前端翻译] ──→ TCG IR ──→ [优化] ──→ TCG IR' ──→ [后端代码生成] ──→ Host 机器码
  (ARM64)      (translate-a64)           (optimize.c)            (tcg-target.c.inc)    (AArch64)
```

关键设计原则：
- **RISC 风格 IR**：所有操作都是寄存器到寄存器（load/store 架构），简化后端适配
- **类型化临时变量**：每个 TCGTemp 带有明确的类型（i32/i64/i128/v64/v128/v256）
- **一次前向遍历优化**：优化器在单次遍历中完成常量折叠、拷贝传播、死码消除
- **贪心寄存器分配**：无需昂贵的图着色算法，通过活跃性分析驱动 spill/reload

核心源文件规模：

| 文件 | 行数 | 职责 |
|------|------|------|
| `tcg.c` | 7,046 | IR 生成、寄存器分配、代码发射框架 |
| `optimize.c` | 3,244 | IR 优化器 |
| `tcg-target.c.inc`（aarch64） | 3,592 | AArch64 后端代码生成 |
| `cpu-exec.c` | 1,087 | CPU 执行循环 |
| `cputlb.c` | 2,901 | 软件 TLB |
| `translate-a64.c` | 10,961 | ARM64 前端翻译器 |
| `tcg-opc.h` | 185 | 操作码定义 |

## 2. 核心数据结构

### 2.1 TCGTemp — 临时变量/寄存器

每个 TCG IR 中的操作数（变量、常量、全局状态字段）都表示为 `TCGTemp`：

```c
// tcg.h:270-293
typedef struct TCGTemp {
    TCGReg reg:8;              // 当前分配的宿主寄存器（若已分配）
    TCGTempVal val_type:8;     // 值状态：TEMP_VAL_DEAD/REG/MEM/CONST
    TCGType base_type:8;       // 基础类型（可能比 type 更宽）
    TCGType type:8;            // 实际类型：I32/I64/I128/V64/V128/V256
    TCGTempKind kind:3;        // 种类：GLOBAL/NORMAL/LOCAL/CONST/FIXED
    unsigned int indirect_reg:1;    // 是否通过寄存器间接访问
    unsigned int indirect_base:1;   // 是否是间接基址
    unsigned int mem_coherent:1;    // 内存中的值是否与寄存器一致
    unsigned int mem_allocated:1;   // 是否已分配栈帧槽位
    unsigned int temp_allocated:1;  // 是否已分配
    unsigned int temp_subindex:2;   // 128 位值的子索引

    int64_t val;               // 常量值（当 val_type == TEMP_VAL_CONST）
    struct TCGTemp *mem_base;  // 内存基址临时变量（通常是 env 指针）
    intptr_t mem_offset;       // 相对于 mem_base 的偏移
    const char *name;          // 调试名称
    uintptr_t state;           // 优化遍使用的状态
    void *state_ptr;           // 优化遍使用的状态指针
} TCGTemp;
```

**TCGTempKind 种类说明**：

| 种类 | 含义 | 示例 |
|------|------|------|
| `TEMP_GLOBAL` | CPU 状态全局变量 | `cpu_env`（CPUArchState 指针） |
| `TEMP_TB` | 翻译块内有效，块结束时失效 | 算术运算中间结果 |
| `TEMP_CONST` | 编译时常量 | 立即数 |
| `TEMP_FIXED` | 固定绑定到宿主寄存器 | 帧指针 |

### 2.2 TCGOp — IR 操作指令

```c
// tcg.h:310-329
struct TCGOp {
    TCGOpcode opc   : 8;       // 操作码（INDEX_op_add 等）
    unsigned nargs  : 8;       // 参数数量

    unsigned param1 : 8;       // 操作码特定参数（如条件码、内存属性）
    unsigned param2 : 8;       // 操作码特定参数

    TCGLifeData life;          // 活跃性数据（DEAD_ARG/SYNC_ARG 位图）

    QTAILQ_ENTRY(TCGOp) link;  // 双向链表指针

    TCGRegSet output_pref[2];  // 输出寄存器偏好集

    TCGArg args[];             // 柔性数组：操作数参数
};
```

所有 `TCGOp` 通过 `QTAILQ` 链表组织在 `TCGContext.ops` 中，形成线性 IR 序列。

### 2.3 TCGLabel — 分支目标

```c
// tcg.h:99-111
typedef struct TCGLabel {
    bool present;              // 标签是否已定义
    bool has_value;            // 是否已绑定到代码地址
    uint16_t id;               // 标签 ID
    union {
        uintptr_t value;       // 绑定的代码地址
        const tcg_insn_unit *value_ptr;  // 代码指针
    } u;
    QSIMPLEQ_HEAD(, TCGLabelUse) branches;  // 引用此标签的分支列表
    QSIMPLEQ_HEAD(, TCGRelocation) relocs;  // 需要回填的重定位列表
    QSIMPLEQ_ENTRY(TCGLabel) next;          // 标签链表
} TCGLabel;
```

### 2.4 TCGContext — 编译上下文

`TCGContext` 是 TCG 编译的核心状态容器，每个 vCPU 线程持有一个：

```c
// tcg.h:346-441
struct TCGContext {
    // === 内存池 ===
    uintptr_t pool_cur, pool_end;
    TCGPool *pool_first, *pool_current;

    // === 计数器 ===
    int nb_labels;             // 标签数量
    int nb_globals;            // 全局变量数量
    int nb_temps;              // 临时变量总数
    int nb_ops;                // 操作数量

    // === 地址类型 ===
    TCGType addr_type;         // 客户机地址类型：I32 或 I64
    TCGBar guest_mo;           // 客户机内存序模型

    // === 寄存器状态 ===
    TCGRegSet reserved_regs;   // 保留寄存器集
    TCGTemp *reg_to_temp[TCG_TARGET_NB_REGS]; // 寄存器→临时变量映射

    // === 代码缓冲区 ===
    TranslationBlock *gen_tb;  // 当前翻译的 TB
    tcg_insn_unit *code_buf;   // TB 代码起始指针
    tcg_insn_unit *code_ptr;   // 当前代码写入位置
    void *code_gen_buffer;     // 全局代码生成缓冲区
    size_t code_gen_buffer_size;
    void *code_gen_highwater;  // 缓冲区刷新阈值

    // === IR 操作队列 ===
    QTAILQ_HEAD(, TCGOp) ops, free_ops;  // 操作链表 + 空闲链表
    QSIMPLEQ_HEAD(, TCGLabel) labels;    // 标签链表
    TCGOp *emit_before_op;     // 插入点（用于在特定位置插入 op）

    // === 临时变量池 ===
    TCGTemp temps[TCG_MAX_TEMPS]; // 全局在前，临时在后

    // === 指令映射 ===
    uint16_t gen_insn_end_off[TCG_MAX_INSNS]; // 每条客户指令对应的宿主码偏移
    uint64_t *gen_insn_data;   // 客户指令元数据

    sigjmp_buf jmp_trans;      // 翻译溢出跳转点
};
```

## 3. TCG 操作码完整分类

所有 TCG 操作码定义在 `tcg-opc.h:29-183`，使用宏 `DEF(name, oargs, iargs, cargs, flags)` 定义：

| 参数 | 含义 |
|------|------|
| `oargs` | 输出操作数数量 |
| `iargs` | 输入操作数数量 |
| `cargs` | 常量参数数量 |
| `flags` | 操作码标志（`TCG_OPF_*`） |

### 3.1 控制流操作（tcg-opc.h:29-37, 112-116）

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `discard` | (1,0,0) | 丢弃临时变量 |
| `set_label` | (0,0,1) | 设置标签位置 |
| `call` | (0,0,3) | 调用 helper 函数 |
| `br` | (0,0,1) | 无条件分支到标签 |
| `brcond` | (0,2,2) | 条件分支（比较两个操作数） |
| `mb` | (0,0,1) | 内存屏障 |
| `exit_tb` | (0,0,1) | 退出翻译块（返回到执行循环） |
| `goto_tb` | (0,0,1) | 直接跳转到另一个 TB（可链接） |
| `goto_ptr` | (0,1,0) | 通过指针间接跳转到 TB |

### 3.2 数据移动（tcg-opc.h:41, 67）

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `mov` | (1,1,0) | 寄存器间移动 |
| `movcond` | (1,4,1) | 条件移动：`dest = (a cond b) ? c : d` |

### 3.3 算术运算（tcg-opc.h:43, 53-56, 68-75, 93, 96-104）

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `add` | (1,2,0) | 加法 |
| `sub` | (1,2,0) | 减法 |
| `neg` | (1,1,0) | 取反 |
| `mul` | (1,2,0) | 乘法（低半部分） |
| `muls2` | (2,2,0) | 有符号乘法（完整结果） |
| `mulu2` | (2,2,0) | 无符号乘法（完整结果） |
| `mulsh` | (1,2,0) | 有符号乘法高半部分 |
| `muluh` | (1,2,0) | 无符号乘法高半部分 |
| `divs` | (1,2,0) | 有符号除法 |
| `divu` | (1,2,0) | 无符号除法 |
| `rems` | (1,2,0) | 有符号取余 |
| `remu` | (1,2,0) | 无符号取余 |
| `addco/addci/addcio` | — | 带进位加法系列 |
| `subbo/subbi/subbio` | — | 带借位减法系列 |

### 3.4 逻辑运算（tcg-opc.h:44-45, 57, 73-79, 94）

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `and` | (1,2,0) | 按位与 |
| `andc` | (1,2,0) | 按位与非（a & ~b） |
| `or` | (1,2,0) | 按位或 |
| `orc` | (1,2,0) | 按位或非（a \| ~b） |
| `xor` | (1,2,0) | 按位异或 |
| `not` | (1,1,0) | 按位取反 |
| `nand` | (1,2,0) | 与非 |
| `nor` | (1,2,0) | 或非 |
| `eqv` | (1,2,0) | 异或非（同或） |

### 3.5 移位与旋转（tcg-opc.h:82-88）

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `shl` | (1,2,0) | 逻辑左移 |
| `shr` | (1,2,0) | 逻辑右移 |
| `sar` | (1,2,0) | 算术右移 |
| `rotl` | (1,2,0) | 循环左移 |
| `rotr` | (1,2,0) | 循环右移 |

### 3.6 位域与扩展（tcg-opc.h:46-52, 58-59, 86, 106-110）

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `bswap16/32/64` | (1,1,1) | 字节序交换 |
| `clz` | (1,2,0) | 前导零计数 |
| `ctz` | (1,2,0) | 尾部零计数 |
| `ctpop` | (1,1,0) | 人口计数（popcount） |
| `deposit` | (1,2,2) | 位域插入 |
| `extract` | (1,1,2) | 无符号位域提取 |
| `sextract` | (1,1,2) | 有符号位域提取 |
| `extract2` | (1,2,1) | 双源位域提取 |
| `ext_i32_i64` | (1,1,0) | i32 有符号扩展到 i64 |
| `extu_i32_i64` | (1,1,0) | i32 无符号扩展到 i64 |
| `extrl_i64_i32` | (1,1,0) | 提取 i64 低 32 位 |
| `extrh_i64_i32` | (1,1,0) | 提取 i64 高 32 位 |

### 3.7 CPU 状态加载/存储（tcg-opc.h:60-66, 89-92）

这些操作用于访问 `CPUArchState` 结构体中的字段（通过 `env` 指针偏移）：

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `ld8u/ld8s` | (1,1,1) | 从 CPU 状态加载 8 位（无符号/有符号扩展） |
| `ld16u/ld16s` | (1,1,1) | 从 CPU 状态加载 16 位 |
| `ld32u/ld32s` | (1,1,1) | 从 CPU 状态加载 32 位 |
| `ld` | (1,1,1) | 从 CPU 状态加载自然宽度 |
| `st8/st16/st32` | (0,2,1) | 向 CPU 状态存储（截断） |
| `st` | (0,2,1) | 向 CPU 状态存储自然宽度 |

### 3.8 客户机内存访问（tcg-opc.h:121-124）

这些操作执行带地址翻译和 TLB 查找的客户机内存访问：

| 操作码 | 签名 | 说明 |
|--------|------|------|
| `qemu_ld` | (1,1,1) | 客户机内存加载（带 TLB 和端序处理） |
| `qemu_st` | (0,2,1) | 客户机内存存储 |
| `qemu_ld2` | (2,1,1) | 客户机内存加载 128 位 |
| `qemu_st2` | (0,3,1) | 客户机内存存储 128 位 |

**`qemu_ld/st` 与 `ld/st` 的关键区别**：
- `ld/st`：直接通过偏移访问 CPU 状态结构体（已知宿主地址）
- `qemu_ld/st`：访问客户机虚拟内存，需经过 TLB 查找、地址翻译、端序转换

### 3.9 向量/SIMD 操作（tcg-opc.h:126-179）

TCG 提供完整的向量操作集，用于翻译 ARM NEON/SVE 等 SIMD 指令：

| 类别 | 操作码 |
|------|--------|
| 移动/复制 | `mov_vec`, `dup_vec`, `dupm_vec` |
| 加载/存储 | `ld_vec`, `st_vec` |
| 算术 | `add_vec`, `sub_vec`, `mul_vec`, `neg_vec`, `abs_vec` |
| 饱和算术 | `ssadd_vec`, `usadd_vec`, `sssub_vec`, `ussub_vec` |
| 最值 | `smin_vec`, `umin_vec`, `smax_vec`, `umax_vec` |
| 逻辑 | `and_vec`, `or_vec`, `xor_vec`, `andc_vec`, `orc_vec`, `nand_vec`, `nor_vec`, `eqv_vec`, `not_vec` |
| 立即移位 | `shli_vec`, `shri_vec`, `sari_vec`, `rotli_vec` |
| 标量移位 | `shls_vec`, `shrs_vec`, `sars_vec`, `rotls_vec` |
| 向量移位 | `shlv_vec`, `shrv_vec`, `sarv_vec`, `rotlv_vec`, `rotrv_vec` |
| 比较 | `cmp_vec` |
| 选择 | `bitsel_vec`, `cmpsel_vec` |

## 4. TCG 类型系统

```c
// tcg.h:128-149
typedef enum TCGType {
    TCG_TYPE_I32,     // 32 位整数
    TCG_TYPE_I64,     // 64 位整数
    TCG_TYPE_I128,    // 128 位整数

    TCG_TYPE_V64,     // 64 位向量（ARM NEON D 寄存器）
    TCG_TYPE_V128,    // 128 位向量（ARM NEON Q 寄存器）
    TCG_TYPE_V256,    // 256 位向量（AVX）

    TCG_TYPE_REG = TCG_TYPE_I64,   // 宿主寄存器宽度别名
    TCG_TYPE_PTR = TCG_TYPE_I64,   // 宿主指针宽度别名
} TCGType;
```

**内存操作属性**（`memop.h:17-167`）：

`MemOp` 枚举编码了内存访问的尺寸、符号性、端序和原子性：

| 位域 | 含义 |
|------|------|
| `MO_SIZE` (bit 0-2) | 访问尺寸：8/16/32/64/128/256 |
| `MO_SIGN` (bit 3) | 有符号扩展 |
| `MO_BSWAP` (bit 4) | 字节序交换 |
| `MO_ALIGN` (bit 5-7) | 对齐要求 |
| `MO_ATOM_*` (bit 8+) | 原子性级别 |

## 5. 条件码系统

TCG 条件码采用位编码设计，便于通过位操作进行条件反转和交换（`tcg-cond.h:36-60`）：

```c
typedef enum {
    TCG_COND_NEVER  = 0b0000,   // 永不成立
    TCG_COND_ALWAYS = 0b0001,   // 永远成立

    // 等式比较
    TCG_COND_EQ     = 0b1000,   // 等于
    TCG_COND_NE     = 0b1001,   // 不等于

    // 测试比较（AND 后与 0 比较）
    TCG_COND_TSTEQ  = 0b1100,   // (a & b) == 0
    TCG_COND_TSTNE  = 0b1101,   // (a & b) != 0

    // 有符号比较
    TCG_COND_LT     = 0b0010,   // 小于
    TCG_COND_GE     = 0b0011,   // 大于等于
    TCG_COND_GT     = 0b0110,   // 大于
    TCG_COND_LE     = 0b0111,   // 小于等于

    // 无符号比较
    TCG_COND_LTU    = 0b1010,   // 无符号小于
    TCG_COND_GEU    = 0b1011,   // 无符号大于等于
    TCG_COND_GTU    = 0b1110,   // 无符号大于
    TCG_COND_LEU    = 0b1111,   // 无符号小于等于
} TCGCond;
```

**位编码设计的优雅之处**：
- 条件反转：`tcg_invert_cond(c) = c ^ 1`（翻转最低位）
- 操作数交换：`tcg_swap_cond()` 通过位重排实现 LT↔GT 等互换
- 有符号/无符号转换：`tcg_unsigned_cond()` / `tcg_signed_cond()` 通过 bit3 切换

**ARM NZCV → TCG 条件码映射**：

| ARM 条件 | NZCV 含义 | TCG 条件 |
|----------|-----------|----------|
| EQ | Z=1 | `TCG_COND_EQ` |
| NE | Z=0 | `TCG_COND_NE` |
| CS/HS | C=1 | `TCG_COND_GEU` |
| CC/LO | C=0 | `TCG_COND_LTU` |
| GE | N=V | `TCG_COND_GE` |
| LT | N≠V | `TCG_COND_LT` |
| GT | Z=0 && N=V | `TCG_COND_GT` |
| LE | Z=1 \|\| N≠V | `TCG_COND_LE` |

## 6. IR 生成 API（tcg_gen_*）

前端翻译器通过 `tcg_gen_*` API 发射 TCG IR。这些 API 定义在 `tcg-op-common.h:32-260` 和 `tcg-op.h:22-300` 中。

### 6.1 API 分类

**算术运算**（`tcg-op-common.h:91-192`）：
```c
void tcg_gen_add_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_sub_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_mul_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_neg_i64(TCGv_i64 ret, TCGv_i64 a);
void tcg_gen_div_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);   // 有符号
void tcg_gen_divu_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);  // 无符号
```

**逻辑运算**：
```c
void tcg_gen_and_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_or_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_xor_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_not_i64(TCGv_i64 ret, TCGv_i64 a);
void tcg_gen_andc_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);  // a & ~b
```

**移位**：
```c
void tcg_gen_shl_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);   // 左移
void tcg_gen_shr_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);   // 逻辑右移
void tcg_gen_sar_i64(TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);   // 算术右移
```

**内存访问**：
```c
// CPU 状态访问（通过 env 指针偏移，编译时已知地址）
void tcg_gen_ld_i64(TCGv_i64 ret, TCGv_ptr base, intptr_t offset);
void tcg_gen_st_i64(TCGv_i64 val, TCGv_ptr base, intptr_t offset);

// 客户机虚拟内存访问（运行时 TLB 查找）
void tcg_gen_qemu_ld_i64(TCGv_i64 ret, TCGv_i64 addr, TCGArg idx, MemOp memop);
void tcg_gen_qemu_st_i64(TCGv_i64 val, TCGv_i64 addr, TCGArg idx, MemOp memop);
```

**分支与条件**：
```c
void tcg_gen_br(TCGLabel *label);                               // 无条件跳转
void tcg_gen_brcond_i64(TCGCond cond, TCGv_i64 a, TCGv_i64 b, TCGLabel *l);
void tcg_gen_setcond_i64(TCGCond cond, TCGv_i64 ret, TCGv_i64 a, TCGv_i64 b);
void tcg_gen_movcond_i64(TCGCond cond, TCGv_i64 ret, TCGv_i64 c1, TCGv_i64 c2,
                         TCGv_i64 v1, TCGv_i64 v2);
```

**扩展**（`tcg-op-common.h:152-159`）：
```c
void tcg_gen_ext8s_i64(TCGv_i64 ret, TCGv_i64 arg);    // 8 位有符号扩展
void tcg_gen_ext8u_i64(TCGv_i64 ret, TCGv_i64 arg);    // 8 位无符号扩展
void tcg_gen_ext16s_i64(TCGv_i64 ret, TCGv_i64 arg);
void tcg_gen_ext32s_i64(TCGv_i64 ret, TCGv_i64 arg);
void tcg_gen_ext_i32_i64(TCGv_i64 ret, TCGv_i32 arg);  // i32→i64 有符号
void tcg_gen_extu_i32_i64(TCGv_i64 ret, TCGv_i32 arg); // i32→i64 无符号
```

### 6.2 宽度别名

`tcg-op.h` 提供了根据目标架构自动选择 i32/i64 的别名宏：

```c
// AArch64 目标上：TCGv == TCGv_i64
#define tcg_gen_add_tl    tcg_gen_add_i64
#define tcg_gen_sub_tl    tcg_gen_sub_i64
// ...
```

## 7. Helper 函数机制

复杂的客户机指令（浮点运算、系统寄存器访问、异常处理等）无法直接用 TCG IR 表示，需要通过 **Helper 函数** 回调到 C 代码。

### 7.1 DEF_HELPER 宏

Helper 函数通过 `DEF_HELPER_FLAGS_{0..7}` 宏声明（`helper-info.c.inc:18-86`）：

```c
// 声明格式：DEF_HELPER_FLAGS_N(name, flags, ret_type, arg1_type, ...)
// 示例（target/arm/tcg/helper.h 中）：
DEF_HELPER_FLAGS_1(wfi, TCG_CALL_NO_WG, void, env)
DEF_HELPER_FLAGS_3(vfp_adds, TCG_CALL_NO_RWG, f32, f32, f32, fpst)
```

**flags 含义**：

| Flag | 含义 |
|------|------|
| `TCG_CALL_NO_READ_GLOBALS` | 不读取全局状态（允许更激进优化） |
| `TCG_CALL_NO_WRITE_GLOBALS` | 不写入全局状态 |
| `TCG_CALL_NO_SIDE_EFFECTS` | 无副作用（可被死码消除） |
| `TCG_CALL_NO_RWG` | 不读写全局 = NO_READ + NO_WRITE |

### 7.2 Helper 元数据

每个 Helper 被编码为 `TCGHelperInfo`（`helper-info.h:58-77`）：

```c
struct TCGHelperInfo {
    void *func;                // C 函数指针
    const char *name;          // 调试名称
    unsigned typemask;         // 参数类型位图（每个参数 4 位编码类型）
    unsigned flags;            // TCG_CALL_* 标志
    unsigned nr_in, nr_out;    // 输入/输出参数数量
    TCGCallArgumentKind out_kind; // 返回值类型
    // ... 参数布局信息
};
```

### 7.3 调用发射

`tcg_gen_callN()`（`tcg.c:2502-2593`）负责将 Helper 调用编码为 TCG IR：

1. 将参数打包到 `TCGOp.args[]` 中
2. 处理 i32→i64 参数扩展（在 64 位宿主上）
3. 处理返回值分配
4. 发射 `INDEX_op_call` 操作码

---

# 第二部分：ARM64 指令翻译

## 8. 翻译入口与 TranslatorOps

ARM64 前端翻译器的入口位于 `translate-a64.c:10954-10961`：

```c
void aarch64_translate_code(CPUState *cpu, TranslationBlock *tb,
                            int *max_insns, vaddr pc, void *host_pc)
{
    DisasContext dc;
    translator_loop(cpu, tb, max_insns, pc, host_pc,
                    &aarch64_translator_ops, &dc.base, TCG_TYPE_VA);
}
```

**TranslatorOps 回调表**（`translate-a64.c:10946-10952`）：

```c
static const TranslatorOps aarch64_translator_ops = {
    .init_disas_context = aarch64_tr_init_disas_context,  // 初始化上下文
    .tb_start           = aarch64_tr_tb_start,             // TB 开始
    .insn_start         = aarch64_tr_insn_start,           // 每条指令开始
    .translate_insn     = aarch64_tr_translate_insn,        // 翻译单条指令
    .tb_stop            = aarch64_tr_tb_stop,               // TB 结束处理
};
```

`translator_loop()` 是通用翻译循环框架，驱动以下流程：

```
init_disas_context()
tb_start()
while (指令数 < max_insns && !结束条件) {
    insn_start()                    // 记录 PC → 宿主码偏移映射
    insn = 取指(pc)                 // translator_ldl_end()
    translate_insn(insn)            // 解码 + 发射 TCG IR
    pc += 4                         // AArch64 固定 4 字节指令
}
tb_stop()                           // 处理 TB 退出
```

## 9. DisasContext 上下文

`DisasContext`（`translate.h:39-204`）是翻译过程中的核心状态，每个 TB 翻译周期一个：

```c
typedef struct DisasContext {
    DisasContextBase base;      // 通用基类（PC、TB、单步等）

    // === ARM64 特有字段 ===
    target_ulong pc_curr;       // 当前指令 PC
    target_ulong pc_save;       // 保存的 PC（用于异常回退）
    int mmu_idx;                // MMU 索引（用于 TLB 查找）
    int current_el;             // 当前异常级别（EL0-EL3）
    bool aarch64;               // 是否 AArch64 模式（vs AArch32）
    bool thumb;                 // 是否 Thumb 模式（AArch32）

    // === 特性标志 ===
    uint64_t features;          // CPU 特性位图（FEAT_*）
    bool fp_excp_el;            // 浮点异常目标 EL
    bool sve_excp_el;           // SVE 异常目标 EL
    bool sme_excp_el;           // SME 异常目标 EL

    // === 地址标记 ===
    bool tbii;                  // 顶部字节忽略（TBI）— 指令地址
    bool tbid;                  // 顶部字节忽略（TBI）— 数据地址
    bool tcma;                  // MTE 标签检查配置

    // === 单步与调试 ===
    bool ss_active;             // 软件单步激活
    bool pstate_ss;             // PSTATE.SS 位

    // === 安全与权限 ===
    bool pauth_active;          // 指针认证激活
    bool mte_active[2];         // MTE 标签检查激活（读/写）
    bool unpriv;                // 非特权访问模式
    int btype;                  // BTI（分支目标标识）状态

    // === 系统寄存器 ===
    GHashTable *cp_regs;        // 系统寄存器查找表

    uint32_t insn;              // 当前待翻译的指令编码
} DisasContext;
```

**初始化**（`translate-a64.c:10653-10756`）：
- 从 `tb->flags` 和 `tb->cs_base` 解包所有翻译上下文标志
- 设置 `dc->aarch64 = true`
- 根据 `EL` 和 `SCTLR` 配置端序、对齐、特性
- 初始化 `cp_regs` 用于系统寄存器查找

## 10. Decodetree 解码器

QEMU 使用 **decodetree** 工具（`scripts/decodetree.py`）从 `.decode` 模式描述文件自动生成 C 解码器。

### 10.1 .decode 文件列表

`target/arm/tcg/` 下共有 15 个解码文件：

| 文件 | 覆盖范围 |
|------|----------|
| `a64.decode` | **AArch64 核心指令集** |
| `sve.decode` | **SVE/SVE2 可伸缩向量扩展** |
| `sme.decode` | **SME 可伸缩矩阵扩展** |
| `sme-fa64.decode` | SME FA64 模式 |
| `a32.decode` | AArch32 ARM 模式 |
| `a32-uncond.decode` | AArch32 无条件指令 |
| `t16.decode` | Thumb-16 |
| `t32.decode` | Thumb-32 |
| `vfp.decode` / `vfp-uncond.decode` | VFP 浮点 |
| `neon-dp.decode` / `neon-ls.decode` / `neon-shared.decode` | NEON SIMD |
| `mve.decode` | M-profile 向量扩展 |
| `m-nocp.decode` | M-profile 无协处理器 |

### 10.2 解码文件语法

以 `a64.decode:22-95` 为例：

```python
# 字段提取器定义
%rd             0:5                    # 提取 bit[4:0] 作为 rd
%esz_sd         22:1 !function=plus_2  # 提取 bit[22]，通过函数 plus_2 变换

# 参数集定义
&r              rn                     # 只含 rn 的参数集
&rrr            rd rn rm               # 含 rd, rn, rm 的参数集
&ri             rd rn imm              # 含 rd, rn, imm 的参数集

# 格式定义（匹配模板）
@addsub_imm     .. ...... . imm:12 rn:5 rd:5   &ri
@logic_shift    .. ...... shift:2 rm:5 imm:6 rn:5 rd:5  &rrr_shift

# 指令模式定义
ADD_i           . 00 100010 . ............ ..... .....  @addsub_imm
SUB_i           . 10 100010 . ............ ..... .....  @addsub_imm
AND_i           . 00 100100 . ...... ...... ..... .....  @logic_imm
B               0 00101 imm:s26                          # 无条件分支
BL              1 00101 imm:s26                          # 带链接分支
B_cond          0101010 0 imm:s19 0 cond:4               # 条件分支
BR              1101011 0000 11111 000000 rn:5 00000      # 寄存器分支
BLR             1101011 0001 11111 000000 rn:5 00000
RET             1101011 0010 11111 000000 rn:5 00000
```

**语法元素**：
- `%field` — 位域提取函数，从指令编码中提取特定位
- `&args` — 参数集声明，定义翻译函数接收的参数结构
- `@format` — 匹配格式模板，将固定位和变量位映射到参数
- 指令行 — 固定位模式 + 格式引用，匹配后调用 `trans_<NAME>()` 函数

### 10.3 解码分发

生成的解码器被包含在翻译器中（`translate-a64.c:79-80`）：

```c
#include "decode-sme-fa64.c.inc"
#include "decode-a64.c.inc"
```

翻译单条指令时（`aarch64_tr_translate_insn()`，`translate-a64.c:10774-10874`）：

```c
static void aarch64_tr_translate_insn(DisasContextBase *dcbase, CPUState *cpu)
{
    // 1. 取指
    s->insn = translator_ldl_end(env, &s->base, pc, MO_LE);  // :10814-10818

    // 2. 尝试多个解码器
    if (!disas_a64(s, insn))           // 核心 A64 解码器
        if (!disas_sme(s, insn))       // SME 解码器
            if (!disas_sve(s, insn))   // SVE 解码器
                unallocated_encoding(s); // 未分配编码
}
```

## 11. 数据处理指令翻译

### 11.1 ADD/SUB 立即数

`a64.decode:113-121` 定义模式，生成的解码器调用 `trans_ADD_i()` / `trans_SUB_i()`（`translate-a64.c:4991-4997`）：

```
Guest:  ADD X0, X1, #42
        ↓ [trans_ADD_i]
TCG IR: tmp = tcg_gen_add_i64(cpu_reg(rd), cpu_reg(rn), tcg_constant_i64(42))
```

翻译流程：
1. `cpu_reg(s, rn)` 获取源寄存器对应的 TCGv_i64（指向 `CPUARMState.xregs[rn]`）
2. 生成立即数常量 `tcg_constant_i64(imm)`
3. 调用 `tcg_gen_add_i64()` 发射 `INDEX_op_add` 操作
4. 结果写入 `cpu_reg(s, rd)`

### 11.2 ADD/SUB 寄存器（带移位）

通过 `do_addsub_reg()` 处理：
1. 读取 `rm` 并根据 `shift` 字段应用移位（`tcg_gen_shli/shri/sari_i64`）
2. 发射 `tcg_gen_add_i64` 或 `tcg_gen_sub_i64`
3. 若指令是 ADDS/SUBS（设置标志），额外调用 `gen_add64_CC()` / `gen_sub64_CC()` 更新 NZCV

### 11.3 逻辑指令

AND/ORR/EOR/BIC 通过 `do_logic_reg()` 处理，翻译为对应的 `tcg_gen_and/or/xor/andc_i64`。

### 11.4 ADR/ADRP

`trans_ADR()` / `trans_ADRP()`（`translate-a64.c:4975-4988`）：
- ADR：`result = PC + imm`
- ADRP：`result = (PC & ~0xFFF) + (imm << 12)`
- 直接计算为常量写入目标寄存器

## 12. 内存访问指令翻译

### 12.1 LDR/STR 立即数偏移

`trans_STR_i()` / `trans_LDR_i()`（`translate-a64.c:3904-3937`）：

```
Guest:  LDR X0, [X1, #8]
        ↓
TCG IR: addr = tcg_gen_add_i64(cpu_reg(rn), tcg_constant_i64(8))
        tcg_gen_qemu_ld_i64(cpu_reg(rd), addr, mmu_idx, MO_64 | MO_LE)
```

关键步骤：
1. **地址计算**：`op_addr_ldst_imm_pre()`（`translate-a64.c:3873-3902`）处理 pre-index / post-index / unsigned offset 三种寻址模式
2. **内存访问**：`do_gpr_ld_memidx()`（`translate-a64.c:1204-1227`）发射 `tcg_gen_qemu_ld_i64`
3. **MMU 索引**：通过 `get_mem_index(s)`（`translate-a64.c:153-156`）获取当前 EL 对应的 TLB 索引

### 12.2 LDR/STR 寄存器偏移

`trans_LDR()` / `trans_STR()`（`translate-a64.c:3998-4020`）：
- 支持 LSL/SXTW/UXTW 等索引扩展
- 通过 `op_addr_ldst_pre()` 计算最终地址

### 12.3 存储路径

`do_gpr_st_memidx()`（`translate-a64.c:1169-1188`）发射 `tcg_gen_qemu_st_i64`：

```
Guest:  STR X0, [X1]
        ↓
TCG IR: tcg_gen_qemu_st_i64(cpu_reg(rd), cpu_reg(rn), mmu_idx, MO_64 | MO_LE)
```

### 12.4 qemu_ld/st 与 ld/st 的本质区别

| 特性 | `tcg_gen_ld_i64` | `tcg_gen_qemu_ld_i64` |
|------|-------------------|------------------------|
| 目标 | CPU 状态结构体 | 客户机虚拟内存 |
| 地址 | 编译时常量偏移 | 运行时虚拟地址 |
| TLB | 不需要 | 需要 TLB 查找 |
| 端序 | 宿主端序 | 按 MemOp 指定端序 |
| 代码生成 | 简单 load 指令 | 内联 TLB 快速路径 + 慢速路径 |

## 13. 分支指令翻译

### 13.1 B / BL（无条件分支/带链接分支）

`translate-a64.c:1697-1716`：

```
Guest:  BL label
        ↓
TCG IR: cpu_reg(30) = PC + 4           // 保存返回地址到 LR（X30）
        gen_goto_tb(s, 0, dest)        // 跳转到目标 TB
```

`gen_goto_tb()`（`translate-a64.c:527-564`）：
- 调用 `translator_use_goto_tb()` 检查目标是否在同一页面（可直接链接）
- 若可链接：发射 `tcg_gen_goto_tb()` + `tcg_gen_exit_tb()`
- 否则：发射 `tcg_gen_lookup_and_goto_ptr()` 进行动态查找

### 13.2 B.cond（条件分支）

`translate-a64.c:1756-1774`：

```
Guest:  B.EQ label
        ↓
TCG IR: arm_test_cc(&c, cond)          // 生成 NZCV 比较
        tcg_gen_brcond_i32(c.cond, c.value, tcg_constant_i32(0), label_match)
        gen_goto_tb(s, 0, pc+4)        // 不跳转路径
        set_label(label_match)
        gen_goto_tb(s, 1, dest)        // 跳转路径
```

### 13.3 BR / BLR / RET（寄存器分支）

`translate-a64.c:1800-1834`：

```
Guest:  RET (等价于 BR X30)
        ↓
TCG IR: gen_a64_set_pc(s, cpu_reg(rn))     // PC = Xn
        tcg_gen_lookup_and_goto_ptr()       // 动态 TB 查找（无法直接链接）
```

**PC 更新机制**：
- `gen_a64_set_pc()`（`translate-a64.c:196-200`）将值写入 `CPUARMState.pc`
- `gen_pc_plus_diff()`（`translate-a64.c:246-254`）计算 PC 相对偏移

## 14. 系统指令翻译

### 14.1 MSR / MRS / SYS / SYSL

核心处理在 `handle_sys()`（`translate-a64.c:2764-3146`），这是 ARM64 翻译器中最复杂的函数之一：

```
Guest:  MRS X0, SCTLR_EL1
        ↓
1. redirect_cpreg() 查找系统寄存器
   → get_arm_cp_reginfo(s->cp_regs, key)    // :2748-2755
2. 访问权限检查
3. 生成 helper 调用或直接 TCG load
   → 若寄存器有 .readfn: tcg_gen_callN(helper)
   → 若寄存器直接映射: tcg_gen_ld_i64(env_offset)
4. 结果存入目标通用寄存器
```

系统寄存器通过 `ARMCPRegInfo` 结构体描述（参见 `darren/arm64/00-ARM64-CPU-GICv3-TCG深度分析.md`），包含 `readfn`/`writefn` 回调。

### 14.2 特殊系统指令

`trans_SYS()`（`translate-a64.c:3149-3153`）委托给 `handle_sys()` 处理：
- **TLBI**（TLB 无效化）
- **DC**（数据缓存操作）
- **IC**（指令缓存操作）
- **AT**（地址翻译）

## 15. 条件标志与异常处理

### 15.1 NZCV 惰性求值

ARM64 使用 4 个独立的 TCG 全局变量跟踪条件标志：

```c
// 全局 TCGv 变量（在翻译器初始化时创建）
cpu_NF    // 负标志：存储结果最高位
cpu_ZF    // 零标志：存储完整结果（==0 则 Z=1）
cpu_CF    // 进位标志
cpu_VF    // 溢出标志
```

**惰性求值策略**：
- 不每条指令都计算 NZCV，而是存储产生标志的操作结果
- 仅在条件分支或 MRS NZCV 时才求值标志
- `gen_get_nzcv()`（`translate-a64.c:2506-2543`）将惰性状态转换为标准 NZCV 位图
- `gen_set_nzcv()`（`translate-a64.c:同上`）将 NZCV 位图分拆到惰性变量

### 15.2 异常指令

| 指令 | 翻译函数 | 位置 | 行为 |
|------|----------|------|------|
| SVC | `trans_SVC` | `translate-a64.c:3155-3170` | 发射 helper 触发同步异常 |
| HVC | `trans_HVC` | `translate-a64.c:3173-3190` | EL2 超级调用 |
| SMC | `trans_SMC` | `translate-a64.c:3193-3204` | EL3 安全监视器调用 |
| BRK | `trans_BRK` | `translate-a64.c:3207-3210` | 软件断点 |

异常通过 `gen_exception_insn()` 发射，最终触发 `cpu_loop_exit()` 退出翻译码执行。

## 16. 翻译块边界

### 16.1 TB 结束原因

翻译循环在以下情况结束：
1. **遇到分支指令**（B/BL/BR/RET/B.cond）
2. **异常指令**（SVC/HVC/SMC/BRK）
3. **达到最大指令数**（`max_insns`，默认 512）
4. **跨越页面边界**（翻译块不跨页）
5. **单步模式**（每条指令一个 TB）
6. **遇到需要运行时检查的指令**

### 16.2 TB 退出处理

`aarch64_tr_tb_stop()`（`translate-a64.c:10876-10944`）根据 `DisasJumpType` 处理退出：

```c
switch (dc->base.is_jmp) {
case DISAS_NEXT:           // 正常顺序执行，TB 因指令数限制结束
case DISAS_TOO_MANY:       // → gen_goto_tb(s, 0, pc)   可链接跳转
    break;

case DISAS_UPDATE_EXIT:    // PC 已更新，退出执行循环
case DISAS_EXIT:           // → tcg_gen_exit_tb(NULL, 0) 返回执行循环
    break;

case DISAS_UPDATE_NOCHAIN: // PC 已更新，不可链接
case DISAS_JUMP:           // → tcg_gen_lookup_and_goto_ptr() 动态查找
    break;

case DISAS_WFI:            // 等待中断
    gen_a64_update_pc(s);
    gen_helper_wfi(tcg_env, ...);
    break;

case DISAS_YIELD:          // 让步
    gen_a64_update_pc(s);
    gen_helper_yield(tcg_env);
    break;
}
```

---

# 第三部分：TCG 优化

## 17. 优化器架构

TCG 优化器位于 `optimize.c`（3,244 行），入口为 `tcg_optimize()`（`optimize.c:2999-3244`）。

**设计特点**：
- **单次前向遍历**：使用 `QTAILQ_FOREACH_SAFE` 遍历所有 `TCGOp`
- **每操作码分发**：通过 `fold_*()` 函数族处理各类操作码
- **就地修改**：直接修改 TCGOp 链表，无需构建新 IR
- **基本块感知**：在 BB 边界（`TCG_OPF_BB_END`）调用 `finish_ebb()` 重置状态

```
tcg_optimize() 主循环:
  ┌─→ copy_propagate(op)           // 拷贝传播：替换操作数
  │   fold_<opcode>(ctx, op)       // 操作码特定优化
  │   finish_folding(ctx, op)      // 更新定义/使用信息
  │   if (BB_END) finish_ebb()     // 扩展基本块结束处理
  └─  continue
```

## 18. 常量折叠与传播

### 18.1 常量折叠

在编译时计算常量操作数的运算结果（`optimize.c:391-729`）：

```
优化前: add tmp, const(3), const(5)
优化后: mov tmp, const(8)
```

关键函数：
- `do_constant_folding_2()`（`optimize.c:391-500`）：二元运算常量折叠
- `fold_const1()` / `fold_const2()`（`optimize.c:888-909`）：检查操作数是否为常量并触发折叠

### 18.2 常量传播

跟踪每个临时变量的已知值和已知位掩码（`optimize.c:125-154`）：

```c
// 每个 TCGTemp 的优化状态
typedef struct TempOptInfo {
    TCGTemp *prev_copy;    // 拷贝链
    TCGTemp *next_copy;
    uint64_t z_mask;       // 零位掩码（已知为 0 的位）
    uint64_t s_mask;       // 符号位掩码（已知与符号位相同的位）
    int64_t val;           // 已知常量值
    bool is_const;         // 是否为常量
} TempOptInfo;
```

`tcg_opt_gen_movi()`（`optimize.c:357-400`）将操作替换为常量加载。

## 19. 拷贝传播

当检测到 `mov dst, src` 时，后续使用 `dst` 的地方可以直接使用 `src`（`optimize.c:839-848`）：

```c
static void copy_propagate(OptContext *ctx, TCGOp *op, ...) {
    // 对每个输入操作数，查找是否有更好的拷贝源
    for (i = 0; i < nb_iargs; i++) {
        TCGTemp *ts = find_better_copy(arg_temp(op->args[nb_oargs + i]));
        // 替换操作数
    }
}
```

`find_better_copy()`（`optimize.c:195-209`）在拷贝链中寻找最优源（优先选择常量或全局变量）。

## 20. 死码消除与分支优化

### 20.1 死码消除

通过两个机制实现：

1. **`finish_folding()`**（`optimize.c:864-875`）：重置已被覆盖的定义，使旧定义成为死码
2. **`reachable_code_pass()`**（`optimize.c:3563-3653`）：移除不可达代码
   - 在 `br` / `exit_tb` / `goto_ptr` / 无返回 `call` 之后的代码标记为不可达
   - 移除跳转到紧邻下一标签的分支（`branch-to-next` 优化）

### 20.2 分支优化

`fold_brcond()`（`optimize.c:748-829`）：
- 若两个比较操作数均为常量，在编译时求值分支条件
- 常为真 → 替换为无条件分支 `br`
- 常为假 → 删除分支操作

```
优化前: brcond const(5), const(3), GT, label    // 5 > 3 恒真
优化后: br label                                 // 无条件跳转
```

## 21. 强度削减与扩展优化

### 21.1 强度削减

将昂贵操作替换为等价的廉价操作：

**`fold_sub()`**（`optimize.c:2703-2719`）：
```
sub x, 0      →  neg x          // 减 0 = 取反... 实际上保持不变
sub x, const  →  add x, -const  // 减常量 → 加负常量
```

**`fold_mul()`**（`optimize.c:1281-1361`）：
```
mul x, 1      →  mov x          // 乘 1 = 恒等
mul x, 0      →  movi 0         // 乘 0 = 零
mul x, 2^n    →  shl x, n       // 乘 2 的幂 → 左移
```

**`fold_shift()`**（`optimize.c:2609-2657`）：
```
shl x, 0      →  mov x          // 移 0 位 = 恒等
```

### 21.2 扩展优化

`fold_exts()` / `fold_extu()`（`optimize.c:3099-3105`）和 `fold_masks_zosa_int()`（`optimize.c:923-983`）：
- 通过零位掩码（z_mask）和符号位掩码（s_mask）跟踪值的有效位范围
- 若值已在目标范围内，消除多余的扩展操作

```
优化前: ext8u x     // x 已知高 56 位为 0（z_mask 表明）
优化后: (删除)       // 扩展是多余的
```

### 21.3 Deposit/Extract 优化

commit `744eb39667`（tcg/optimize: possibly expand deposit into zero with shifts）：
- `fold_deposit()` 可将 deposit 操作展开为移位操作组合
- commit `e4cebfc664`（tcg/optimize: Lower unsupported extract2 during optimize）：将不支持的 `extract2` 降级为基本操作

---

# 第四部分：寄存器分配与代码生成

## 22. 活跃性分析

寄存器分配之前需要分析每个临时变量的活跃区间（liveness），确定何时需要 spill/reload。

### 22.1 三遍活跃性分析

QEMU TCG 使用多遍活跃性分析（`tcg.c:3803-4449`）：

| 遍次 | 函数 | 位置 | 职责 |
|------|------|------|------|
| Pass 0 | `liveness_pass_0()` | `tcg.c:3803-3866` | 标记 helper 调用的副作用（哪些全局变量需要同步） |
| Pass 1 | `liveness_pass_1()` | `tcg.c:3882-4266` | 反向遍历，计算每个 op 的活跃变量集，标记 DEAD/SYNC |
| Pass 2 | `liveness_pass_2()` | `tcg.c:4268-4449` | 前向遍历，插入必要的 save/restore 操作 |

**迭代收敛**：Pass 1 和 Pass 2 可能需要迭代执行直到稳定：
```
liveness_pass_0()
do {
    liveness_pass_1()
} while (liveness_pass_2())   // Pass 2 返回是否有修改
```

### 22.2 活跃性数据编码

每个 `TCGOp` 的 `life` 字段是一个位图（`TCGLifeData`），每个操作数占 5 位：

```c
#define DEAD_ARG  (1 << 4)    // 该操作数在此 op 后不再使用
#define SYNC_ARG  (1 << 0)    // 该操作数需要同步到内存
```

## 23. 寄存器分配算法

TCG 使用**贪心寄存器分配**（而非图着色），特点是快速但可能产生更多 spill。

### 23.1 分配流程

`tcg_reg_alloc_op()`（`tcg.c` 中）对每个 `TCGOp` 执行：

```
1. 读取输入操作数约束（哪些寄存器可用）
2. 对每个输入：
   a. 若已在合适寄存器中 → 直接使用
   b. 若在内存中 → temp_load() 加载到空闲寄存器
   c. 若无空闲寄存器 → tcg_reg_free() 溢出最不活跃的
3. 对每个输出：
   a. 根据约束和偏好选择目标寄存器
   b. 若冲突 → 溢出占用者
4. 发射宿主指令
5. 对标记 DEAD 的操作数 → 释放寄存器
6. 对标记 SYNC 的操作数 → temp_sync() 写回内存
```

### 23.2 约束系统

每个 TCG 操作码在后端定义寄存器约束（`tcg.c:883-912`）：

```c
// 从 tcg-target-con-set.h 解析约束集
static const TCGTargetOpDef constraint_sets[] = {
    // 每个条目定义：哪些寄存器可用于输入/输出
};
```

`tcg_target_op_def()`（`tcg.c:3389-3417`）为每个操作码返回其约束。

### 23.3 Spill/Reload

- **Spill**：`temp_sync()`（`tcg.c`）— 将寄存器值写回到栈帧槽位
- **Reload**：`temp_load()`（`tcg.c`）— 从栈帧槽位加载值到寄存器
- **栈帧管理**：`TCGContext` 的 `frame_start/frame_end/current_frame_offset` 跟踪帧分配

## 24. AArch64 后端代码生成

AArch64 后端位于 `tcg/aarch64/tcg-target.c.inc`（3,592 行），负责将 TCG IR 翻译为 AArch64 宿主机器码。

### 24.1 操作码映射表

`all_outop[]`（`tcg-target.c.inc:1158-1234`）将每个 TCG 操作码映射到代码生成函数：

```c
static const OutOp all_outop[] = {
    [INDEX_op_add]       = { .out = tgen_add },
    [INDEX_op_sub]       = { .out = tgen_sub },
    [INDEX_op_and]       = { .out = tgen_and },
    [INDEX_op_or]        = { .out = tgen_or },
    [INDEX_op_xor]       = { .out = tgen_xor },
    [INDEX_op_shl]       = { .out = tgen_shl },
    [INDEX_op_shr]       = { .out = tgen_shr },
    [INDEX_op_sar]       = { .out = tgen_sar },
    [INDEX_op_brcond]    = { .out = outop_brcond },
    [INDEX_op_qemu_ld]   = { .out = outop_qemu_ld },
    [INDEX_op_qemu_st]   = { .out = outop_qemu_st },
    // ...
};
```

### 24.2 代码生成示例

**`tgen_add()`**（`tcg-target.c.inc:2062-2088`）：

```
TCG IR:  add tmp1, tmp2, tmp3   (i64)
         ↓ [tgen_add]
AArch64: ADD X<reg1>, X<reg2>, X<reg3>
```

后端直接编码 AArch64 指令到代码缓冲区，使用 `tcg_out32()` 写入 32 位指令编码。

### 24.3 宿主寄存器分配

AArch64 后端的寄存器配置（`tcg-target.c.inc:3413-3450` 和 `tcg-target.h:50`）：

```
TCG_TARGET_NB_REGS = 64   (32 通用 + 32 SIMD/FP)

参数传递寄存器: X0-X7          (tcg-target.c.inc:74-77)
返回值寄存器:   X0-X1          (tcg-target.c.inc:79-84)
Guest Base:     X28             (tcg-target.c.inc:91)  — 客户机内存基址
保留寄存器:     SP, FP, X18    (tcg-target.c.inc:3413-3450)
```

### 24.4 后端初始化

`tcg_target_init()`（`tcg-target.c.inc:3413-3450`）：
- 设置可用寄存器集
- 标记调用破坏寄存器
- 设置保留寄存器（SP/FP/X18/临时寄存器/guest base）

## 25. 客户机内存访问代码生成

这是 TCG 后端中最关键也最复杂的部分——将 `qemu_ld/qemu_st` 翻译为带内联 TLB 查找的宿主代码。

### 25.1 内联 TLB 快速路径

`prepare_host_addr()`（`tcg-target.c.inc:1650-1757`）生成内联 TLB 查找：

```asm
; === AArch64 生成的 TLB 快速路径 ===

; 1. 计算 TLB 索引
AND     Xtmp, Xaddr, TLB_MASK         ; 地址与 TLB 掩码

; 2. 加载 TLB 条目
ADD     Xtmp, Xtmp, [XCPU, #tlb_table_offset]  ; TLB 表基址 + 索引
LDP     Xtag, Xhost, [Xtmp]           ; 加载 {comparator, host_addr}

; 3. 比较页标记
AND     Xcmp, Xaddr, PAGE_MASK         ; 提取页号
CMP     Xcmp, Xtag                     ; 比较 TLB 标记

; 4. 快速路径 vs 慢速路径
B.NE    slow_path                      ; 不匹配 → 慢速路径

; 5. 快速路径：直接通过宿主地址访问
ADD     Xhost, Xhost, Xaddr_offset    ; 计算宿主地址
LDR     Xresult, [Xhost]              ; 直接加载！
```

### 25.2 加载/存储发射

匹配 TLB 后的直接内存访问：

- `tcg_out_qemu_ld_direct()`（`tcg-target.c.inc:1760-1795`）：根据 MemOp 大小生成 LDRB/LDRH/LDR/LDP
- `tcg_out_qemu_st_direct()`（`tcg-target.c.inc:1797-1820`）：生成 STRB/STRH/STR/STP

### 25.3 TLB 慢速路径

TLB 未命中时调用 C helper 函数：

```c
// 慢速路径 helper 表（tcg-target.c.inc:211-228）
static const void * const qemu_ld_helpers[] = {
    [MO_8]  = helper_ldb_mmu,
    [MO_16] = helper_ldw_mmu,
    [MO_32] = helper_ldl_mmu,
    [MO_64] = helper_ldq_mmu,
};
```

`tcg_out_qemu_ld_slow_path()`（`tcg-target.c.inc:1612-1625`）：
1. 保存当前状态
2. 调用 `helper_ld*_mmu()` 进行完整地址翻译
3. 将结果传回快速路径的目标寄存器

### 25.4 性能关键路径

TLB 命中率通常 > 99%，因此快速路径的质量直接决定 TCG 性能：

```
TLB 命中（热路径）: ~5-7 条宿主指令（AND+ADD+LDP+CMP+B.NE+ADD+LDR）
TLB 未命中（冷路径）: helper 函数调用 → 页表遍历 → TLB 填充 → 重试
```

---

# 第五部分：执行引擎

## 26. TranslationBlock 管理

### 26.1 TranslationBlock 结构

`translation-block.h:46-150` 定义了翻译块的完整元数据：

```c
struct TranslationBlock {
    // === 客户机信息 ===
    vaddr pc;                  // 起始客户机 PC
    uint32_t cs_base;          // 条件执行状态
    uint32_t flags;            // TB 标志（EL、端序、特性等）
    uint32_t cflags;           // 编译标志（CF_*）

    // === 大小信息 ===
    uint16_t size;             // 客户机代码大小（字节）
    uint16_t icount;           // 客户机指令数

    // === 宿主代码 ===
    void *tc_ptr;              // 翻译后的宿主代码指针
    size_t tc_size;            // 宿主代码大小

    // === 跳转/链接 ===
    uintptr_t jmp_reset_offset[2];     // 跳转重置偏移
    uintptr_t jmp_insn_offset[2];      // 跳转指令偏移（用于链接）
    uintptr_t jmp_target_addr[2];      // 跳转目标地址

    // === 链接列表 ===
    uintptr_t jmp_list_head;           // 跳转到此 TB 的链表头
    uintptr_t jmp_list_next[2];        // 链表下一个

    // === 失效管理 ===
    PageDesc *page_next[2];            // 物理页关联
    // ...
};
```

### 26.2 TB 查找

`tb_lookup()`（`cpu-exec.c:195-262`）实现两级查找：

```
1. 一级缓存：cpu->tb_jmp_cache[hash(pc)]
   → 直接比较 pc/cs_base/flags/cflags
   → 命中率 ~95%+

2. 二级哈希表：tb_htable_lookup()
   → 全局 QHT 哈希表查找
   → 命中 → 填充一级缓存

3. 未找到 → 调用翻译器创建新 TB
```

`HELPER(lookup_tb_ptr)()`（`cpu-exec.c:374-405`）是从生成代码内部调用的 TB 查找入口（`goto_ptr` 使用）。

### 26.3 TB 失效

TB 需要在以下情况失效（`tb-maint.c`）：

| 场景 | 函数 | 说明 |
|------|------|------|
| 自修改代码 | `tb_invalidate_phys_page()` | 检测到客户机修改了已翻译的代码页 |
| 范围失效 | `tb_invalidate_phys_range()` | 指定地址范围内的所有 TB |
| 全量刷新 | `tb_flush()` | 刷新所有 TB（API：`include/exec/tb-flush.h:12-36`） |
| TLB 变更 | 间接触发 | ASID 切换、页表修改 |

## 27. CPU 执行循环

### 27.1 主循环

`cpu_exec_loop()`（`cpu-exec.c:930-980`）是 TCG 执行的核心：

```c
static int cpu_exec_loop(CPUState *cpu, SyncClocks *sc) {
    while (true) {
        // 1. 处理异常
        cpu_handle_exception(cpu, &ret);

        // 2. 处理中断
        cpu_handle_interrupt(cpu, &last_tb);

        // 3. 查找/翻译 TB
        tb = tb_lookup(cpu, pc, cs_base, flags, cflags);
        if (!tb) {
            tb = tb_gen_code(cpu, pc, cs_base, flags, cflags);  // 翻译！
        }

        // 4. 执行生成代码
        cpu_tb_exec(cpu, tb, &last_tb, &tb_exit);

        // 5. TB 链接（直接跳转优化）
        tb_add_jump(last_tb, tb_exit, tb);
    }
}
```

### 27.2 进入生成代码

`cpu_tb_exec()`（`cpu-exec.c:427-450`）：

```c
static inline void cpu_tb_exec(CPUState *cpu, TranslationBlock *tb, ...) {
    uintptr_t ret = tcg_qemu_tb_exec(env, tb->tc_ptr);  // 跳入生成代码！
    // ret 编码了退出原因和下一个 TB 指针
}
```

`tcg_qemu_tb_exec()` 是一个宏/函数，直接调用到生成的宿主机器码。生成代码执行完毕后通过 `exit_tb` 返回。

### 27.3 异常与中断处理

**异常处理**（`cpu-exec.c:687-750`）：
```c
static bool cpu_handle_exception(CPUState *cpu, int *ret) {
    if (cpu->exception_index == EXCP_HLT)       → 挂起
    if (cpu->exception_index == EXCP_DEBUG)      → 调试断点
    if (cpu->exception_index == EXCP_INTERRUPT)  → 重新检查中断
    // 调用目标架构的异常处理回调
    cc->tcg_ops->do_interrupt(cpu);
}
```

**中断处理**（`cpu-exec.c:777-881`）：
```c
static bool cpu_handle_interrupt(CPUState *cpu, ...) {
    if (cpu->interrupt_request & CPU_INTERRUPT_*) {
        cc->tcg_ops->cpu_exec_interrupt(cpu, int_req);
        // 中断可能改变 PC → 清除 last_tb 防止错误链接
    }
}
```

### 27.4 setjmp/longjmp 异常机制

`cpu_exec_setjmp()`（`cpu-exec.c:512-547`）建立异常跳转点：

```c
static int cpu_exec_setjmp(CPUState *cpu, SyncClocks *sc) {
    if (sigsetjmp(cpu->jmp_env, 0) == 0) {
        return cpu_exec_loop(cpu, sc);     // 正常执行
    } else {
        cpu_exec_longjmp_cleanup(cpu);     // 异常恢复
        return cpu_exec_loop(cpu, sc);     // 重新执行
    }
}
```

生成代码中的异常通过 `cpu_loop_exit()` 触发 `longjmp`，跳回到执行循环。

## 28. 软件 TLB

### 28.1 CPUTLBEntry 结构

`tlb-common.h:24-41` 定义 TLB 条目：

```c
typedef union CPUTLBEntry {
    /* 快速比较用的地址标记（包含页号+属性） */
    union {
        struct {
            vaddr addr_read;       // 读地址标记
            vaddr addr_write;      // 写地址标记
            vaddr addr_code;       // 取指地址标记
        };
        vaddr addr_idx[3];         // 按索引访问
    };
    /* 客户机地址到宿主地址的偏移量（加法即可得到宿主地址） */
    uintptr_t addend;
} CPUTLBEntry;
```

**工作原理**：
- `addr_read` 等字段存储页对齐的客户机地址（高位）+属性位（低位）
- `addend` 是一个偏移量：`host_addr = guest_addr + addend`
- TLB 查找只需：比较 `(guest_addr & PAGE_MASK)` 与 `addr_read`，匹配则 `addend + guest_addr` 得到宿主地址

### 28.2 TLB 操作

**查找**（`cputlb.c:126-140`）：
```c
static inline size_t tlb_index(uint64_t addr) {
    return (addr >> TARGET_PAGE_BITS) & (TLB_SIZE - 1);
}

static inline CPUTLBEntry *tlb_entry(CPUState *cpu, int mmu_idx, vaddr addr) {
    return &cpu->neg.tlb.f[mmu_idx].table[tlb_index(addr)];
}
```

**填充**（`cputlb.h:39-85`）：
- `tlb_set_page_full()` / `tlb_set_page_with_attrs()` 由目标架构的页表遍历调用
- 将翻译结果写入 TLB 条目

**刷新**（`cputlb.c:369-520`）：
- `tlb_flush()` — 刷新所有条目（上下文切换时）
- `tlb_flush_by_mmuidx_async_work()` — 按 MMU 索引异步刷新
- `tlb_flush_page()` — 刷新单页

### 28.3 慢速路径 Helper

TLB 未命中时由后端生成的慢速路径调用（`cputlb.c`）：

```
load_helper():
  1. 重新查找 TLB（可能被其他 CPU 填充）
  2. 若仍未命中 → 调用 tlb_fill() 触发页表遍历
  3. 检查 MMIO vs RAM
  4. MMIO → memory_region_dispatch_read()
  5. RAM → 直接通过 addend 访问
```

## 29. TB 链接与直接跳转

### 29.1 直接块链接

当 TB_A 的出口跳转到 TB_B 时，可以直接修补 TB_A 的跳转指令使其直接跳到 TB_B 的宿主代码，避免返回执行循环。

**链接建立**（`cpu-exec.c:616-651`）：

```c
static inline void tb_add_jump(TranslationBlock *tb, int n, TranslationBlock *tb_next) {
    // 1. 检查是否已链接
    if (tb->jmp_list_head == 0) {
        // 2. 原子地将 tb 加入 tb_next 的跳转链表
        // 3. 修补跳转目标
        tb_set_jmp_target(tb, n, tb_next->tc_ptr);
    }
}
```

### 29.2 AArch64 跳转修补

`tb_target_set_jmp_target()`（`tcg-target.c.inc:2040-2059`）：

```c
// 直接修改生成代码中的跳转指令
void tb_target_set_jmp_target(const TranslationBlock *tb, int n, uintptr_t jmp_rx, uintptr_t addr) {
    // 计算偏移
    ptrdiff_t offset = addr - jmp_rx;
    // 编码为 AArch64 B 指令（26 位偏移）
    tcg_out32(code, encode_b(offset));
    // 刷新指令缓存
    flush_idcache_range(jmp_rx, 4);
}
```

**链接/解除链接时机**：
- 链接：执行循环发现 TB_A 出口对应的 TB_B 已存在
- 解除：TB_B 被失效时，遍历其 `jmp_list_head` 链表，将所有指向它的跳转重置

## 30. MTTCG 多线程 TCG

MTTCG（Multi-Threaded TCG）允许每个 vCPU 在独立的宿主线程中执行 TCG 翻译码。

### 30.1 线程模型

`tcg-accel-ops-mttcg.c:59-137`：

```c
static void *mttcg_cpu_thread_fn(void *arg) {
    CPUState *cpu = arg;

    rcu_register_thread();          // RCU 注册
    tcg_register_thread();          // TCG 线程注册

    while (true) {
        if (cpu_can_run(cpu)) {
            r = tcg_cpu_exec(cpu);  // 执行循环
        }

        // 处理 EXCP_ATOMIC
        if (cpu->exception_index == EXCP_ATOMIC) {
            cpu_exec_step_atomic(cpu);  // 原子步骤执行
        }

        qemu_wait_io_event(cpu);    // 等待事件
    }
}
```

### 30.2 原子指令处理

MTTCG 中的原子指令需要特殊处理（`cpu-exec.c:549-598`）：

当一个 vCPU 遇到原子指令（如 LDXR/STXR）：
1. 触发 `EXCP_ATOMIC` 异常
2. `cpu_exec_step_atomic()` 在序列化模式下执行该指令
3. 使用 `CF_NO_GOTO_TB | CF_NO_GOTO_PTR` 标志确保单指令 TB
4. 获取 BQL（Big QEMU Lock）保证原子性
5. 执行完成后释放锁，恢复并发

### 30.3 同步机制

| 机制 | 用途 |
|------|------|
| RCU | TB 查找/失效的无锁读（读端零开销） |
| BQL | 设备访问、TB 生成的互斥 |
| per-CPU TCGContext | 每个线程独立的 TCG 编译上下文 |
| 原子操作 | TB 链接/解除链接的原子 CAS |

---

# 附录

## A. TCG 完整操作码速查表

| 类别 | 操作码 | 签名(O,I,C) | 标志 |
|------|--------|-------------|------|
| **控制流** | `discard` | 1,0,0 | NOT_PRESENT |
| | `set_label` | 0,0,1 | BB_END, NOT_PRESENT |
| | `call` | 0,0,3 | CALL_CLOBBER, NOT_PRESENT |
| | `br` | 0,0,1 | BB_END, NOT_PRESENT |
| | `brcond` | 0,2,2 | BB_END, COND_BRANCH, INT |
| | `mb` | 0,0,1 | NOT_PRESENT |
| | `exit_tb` | 0,0,1 | BB_END, NOT_PRESENT |
| | `goto_tb` | 0,0,1 | BB_END, NOT_PRESENT |
| | `goto_ptr` | 0,1,0 | BB_END, NOT_PRESENT |
| **数据移动** | `mov` | 1,1,0 | INT, NOT_PRESENT |
| | `movcond` | 1,4,1 | INT |
| **算术** | `add` | 1,2,0 | INT |
| | `sub` | 1,2,0 | INT |
| | `neg` | 1,1,0 | INT |
| | `mul` | 1,2,0 | INT |
| | `divs/divu` | 1,2,0 | INT |
| | `rems/remu` | 1,2,0 | INT |
| **逻辑** | `and/or/xor` | 1,2,0 | INT |
| | `andc/orc/nand/nor/eqv` | 1,2,0 | INT |
| | `not` | 1,1,0 | INT |
| **移位** | `shl/shr/sar/rotl/rotr` | 1,2,0 | INT |
| **位域** | `deposit/extract/sextract` | 1,1-2,1-2 | INT |
| | `bswap16/32/64` | 1,1,1 | INT |
| | `clz/ctz/ctpop` | 1,1-2,0 | INT |
| **CPU 状态** | `ld/ld8u/../ld32s` | 1,1,1 | — |
| | `st/st8/st16/st32` | 0,2,1 | — |
| **客户机内存** | `qemu_ld/qemu_st` | 1/0,1-2,1 | — |
| | `qemu_ld2/qemu_st2` | 2/0,1-3,1 | — |
| **向量** | `add_vec/../cmpsel_vec` | 1,2-4,0-1 | VECTOR |

## B. 关键源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `include/tcg/tcg-opc.h` | 185 | TCG 操作码定义 |
| `include/tcg/tcg.h` | ~450 | TCGTemp/TCGOp/TCGContext 定义 |
| `include/tcg/tcg-cond.h` | ~130 | 条件码定义与操作 |
| `include/tcg/tcg-op-common.h` | ~260 | IR 生成 API 声明 |
| `include/tcg/tcg-op.h` | ~300 | 宽度别名宏 |
| `include/exec/memop.h` | ~167 | 内存操作属性 |
| `include/tcg/helper-info.h` | ~77 | Helper 元数据 |
| `include/exec/translation-block.h` | ~150 | TB 结构体 |
| `include/exec/tlb-common.h` | ~41 | TLB 条目 |
| `tcg/tcg.c` | 7,046 | IR 生成、寄存器分配、代码发射 |
| `tcg/optimize.c` | 3,244 | IR 优化器 |
| `tcg/aarch64/tcg-target.c.inc` | 3,592 | AArch64 后端 |
| `tcg/aarch64/tcg-target.h` | ~50+ | AArch64 后端常量 |
| `accel/tcg/cpu-exec.c` | 1,087 | CPU 执行循环 |
| `accel/tcg/cputlb.c` | 2,901 | 软件 TLB |
| `accel/tcg/tcg-accel-ops-mttcg.c` | ~137 | MTTCG 线程 |
| `accel/tcg/tb-maint.c` | — | TB 失效维护 |
| `target/arm/tcg/translate-a64.c` | 10,961 | ARM64 前端翻译器 |
| `target/arm/tcg/translate.h` | ~204 | DisasContext 定义 |
| `target/arm/tcg/a64.decode` | — | A64 解码模式 |
| `target/arm/tcg/sve.decode` | — | SVE 解码模式 |

## C. 一条 ARM64 指令的完整生命周期

以 `LDR X0, [X1, #8]` 为例，追踪从客户机指令到宿主执行的完整路径：

```
═══════════════════════════════════════════════════════════════
  阶段 1：翻译（一次性，结果缓存在 TB 中）
═══════════════════════════════════════════════════════════════

1. cpu_exec_loop() 查找 TB 未命中
   → tb_gen_code() 创建新 TranslationBlock

2. aarch64_translate_code()
   → translator_loop() 驱动翻译循环

3. aarch64_tr_translate_insn()
   → insn = 0xF9400420  (LDR X0, [X1, #8] 的编码)
   → disas_a64(s, insn) 调用 decodetree 解码器

4. decode-a64.c.inc 匹配 LDR_i 模式
   → trans_LDR_i(s, &args)

5. trans_LDR_i():
   a. op_addr_ldst_imm_pre(s, rn=1, imm=8)
      → addr_tmp = tcg_gen_add_i64(cpu_reg(1), const(8))
      → 发射 TCG IR: add addr, X1, #8

   b. do_gpr_ld_memidx(s, rd=0, addr_tmp, MO_64|MO_LE, mmu_idx)
      → tcg_gen_qemu_ld_i64(cpu_reg(0), addr_tmp, ...)
      → 发射 TCG IR: qemu_ld X0, addr, (i64, LE)

═══════════════════════════════════════════════════════════════
  阶段 2：优化（tcg_optimize）
═══════════════════════════════════════════════════════════════

6. 常量传播：若 X1 之前被加载为常量，add 可折叠
7. 拷贝传播：若 addr 只使用一次，可能内联
8. 死码消除：若 addr_tmp 在 qemu_ld 后不再使用，标记为 DEAD

═══════════════════════════════════════════════════════════════
  阶段 3：寄存器分配 + 代码生成（AArch64 后端）
═══════════════════════════════════════════════════════════════

9. liveness_pass_1(): 反向标记每个 TCGTemp 的活跃区间

10. tcg_reg_alloc_op() 对 add 操作:
    → 分配宿主寄存器 X9 ← addr, X10 ← cpu_reg(1)
    → tgen_add(): 编码 ADD X9, X10, #8

11. tcg_reg_alloc_op() 对 qemu_ld 操作:
    → outop_qemu_ld():

    a. prepare_host_addr():  // 内联 TLB 快速路径
       AND X11, X9, #TLB_MASK
       ADD X11, X11, [X28, #tlb_offset]
       LDP X12, X13, [X11]          // 加载 TLB {tag, addend}
       AND X14, X9, #PAGE_MASK
       CMP X14, X12
       B.NE slow_path

    b. tcg_out_qemu_ld_direct():   // 快速路径加载
       ADD X13, X13, X9             // host_addr = addend + guest_addr
       LDR X0_host, [X13]           // 实际内存加载！

    c. tcg_out_qemu_ld_slow_path(): // 慢速路径
       slow_path:
       BL helper_ldq_mmu            // 调用 C helper

═══════════════════════════════════════════════════════════════
  阶段 4：执行（每次遇到此 TB 都执行）
═══════════════════════════════════════════════════════════════

12. cpu_exec_loop() → tb_lookup() 查找 TB → 命中！

13. cpu_tb_exec() → tcg_qemu_tb_exec(env, tb->tc_ptr)
    → 跳入阶段 3 生成的宿主机器码

14. 执行 ADD X9, X10, #8

15. 执行 TLB 查找序列:
    → 命中: 直接 LDR X0, [X13] — 约 5ns
    → 未命中: BL helper_ldq_mmu — 约 100-500ns

16. 继续执行 TB 中的下一条指令，或通过 exit_tb 返回循环

═══════════════════════════════════════════════════════════════
  阶段 5：TB 链接（可选优化）
═══════════════════════════════════════════════════════════════

17. 若此 TB 的出口跳转到另一个已翻译的 TB_B:
    → tb_add_jump() 修补跳转指令
    → 下次执行直接跳到 TB_B，不返回执行循环
```

---

> **相关文档**：
> - `darren/architecture/00-全局架构概览.md` — QEMU 全局架构
> - `darren/arm64/00-ARM64-CPU-GICv3-TCG深度分析.md` — ARM64 CPU 模型与 TCG 翻译概览
> - `darren/memory/00-内存子系统深度分析.md` — 内存子系统（与 TLB 相关）
> - `docs/devel/tcg.rst` — 官方 TCG 开发文档
> - `docs/devel/decodetree.rst` — Decodetree 语法参考
