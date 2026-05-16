# TCG 后端深度分析 — IR 生成、寄存器分配与代码缓存

## 1. 概述

TCG（Tiny Code Generator）是 QEMU 的核心 JIT 编译引擎，负责将 Guest 指令翻译为 Host 机器码。整个翻译流程分为三阶段：**前端**（Guest → TCG IR）、**优化**（IR → 优化 IR）、**后端**（IR → Host 机器码）。本文从 IR 表示、寄存器分配、代码缓存管理和执行循环四个维度深入分析 TCG 架构。

**关键源文件：**
- `include/tcg/tcg.h` — TCG 核心数据结构（TCGTemp/TCGOp/TCGContext）
- `tcg/tcg.c` — 寄存器分配器与活性分析（~5500行）
- `tcg/optimize.c` — IR 优化遍
- `tcg/region.c` — 代码缓存区域管理
- `tcg/aarch64/tcg-target.c.inc` — AArch64 宿主后端
- `accel/tcg/translate-all.c` — TB 生成入口
- `accel/tcg/cpu-exec.c` — 执行循环

---

## 2. TCG 中间表示（IR）

### 2.1 TCGTemp — 临时变量

```c
// include/tcg/tcg.h:270-293
typedef struct TCGTemp {
    TCGReg reg:8;          // 分配的宿主寄存器
    TCGTempVal val_type:8; // 值状态: REG/MEM/CONST/DEAD
    TCGType base_type:8;   // 基础类型 (I32/I64/I128/V64/V128/V256)
    TCGType type:8;        // 当前类型
    TCGTempKind kind:3;    // GLOBAL/NORMAL/LOCAL/EBB/TB/CONST
    unsigned int mem_coherent:1;  // 内存中的值是否最新
    unsigned int mem_allocated:1; // 是否已分配栈帧槽位
    unsigned int temp_allocated:1;
    int64_t val;           // 常量值（kind=CONST 时）
    struct TCGTemp *mem_base;    // 栈帧基址临时变量
    intptr_t mem_offset;         // 栈帧偏移
} TCGTemp;

// TCGTempKind 分类:
//   GLOBAL: CPUState 字段（如 env->pc, env->regs[]），跨 TB 持久
//   NORMAL: 单次使用临时量（op 之后即死）
//   LOCAL:  函数内局部变量（跨 basic block）
//   EBB:    扩展基本块范围
//   TB:     整个翻译块范围
//   CONST:  编译期常量
```

### 2.2 TCGOp — IR 指令

```c
// include/tcg/tcg.h:310-329
struct TCGOp {
    TCGOpcode opc   : 8;   // 操作码 (INDEX_op_add, INDEX_op_mov 等)
    unsigned nargs  : 8;   // 参数个数
    unsigned param1 : 8;   // 操作类型/调用输入数
    unsigned param2 : 8;   // 标志/调用输出数/VECE
    TCGLifeData life;      // 操作数活性 (DEAD_ARG/SYNC_ARG)
    QTAILQ_ENTRY(TCGOp) link;  // 双向链表
    TCGRegSet output_pref[2];  // 输出寄存器偏好
    TCGArg args[];         // 操作数数组（TCGTemp 索引）
};

// IR 指令链表: TCGContext.ops（QTAILQ）
// 前端逐条追加，优化遍原地修改，后端逐条消费
```

### 2.3 TCGOpcode — 操作码

```
核心操作码分类:
  算术: add, sub, mul, div, rem, neg, and, or, xor, shl, shr, sar
  比较: setcond, movcond, brcond
  内存: ld (加载), st (存储), qemu_ld/st (Guest 内存)
  控制: br, goto_tb, goto_ptr, exit_tb
  类型: ext8s/16s/32s, trunc, extu, deposit, extract
  向量: add_vec, sub_vec, mul_vec, cmp_vec, ...
  调用: call (helper 函数调用)
  特殊: mb (内存屏障), set_label, discard
```

---

## 3. TCGContext — 编译上下文

```c
// include/tcg/tcg.h:346-400+
struct TCGContext {
    // 常量池
    uintptr_t pool_cur, pool_end;
    
    // 临时变量管理
    int nb_globals;           // 全局变量数（CPUState 映射）
    int nb_temps;             // 总临时变量数
    int nb_ops;               // IR 操作数
    
    // 寄存器分配
    TCGRegSet reserved_regs;  // 保留寄存器（不可分配）
    intptr_t current_frame_offset; // 当前栈帧偏移
    TCGTemp *frame_temp;      // 栈帧基址
    
    // 代码生成
    TranslationBlock *gen_tb; // 当前翻译的 TB
    tcg_insn_unit *code_buf;  // TB 代码起始
    tcg_insn_unit *code_ptr;  // 当前写入位置
    
    // 代码缓存
    void *code_gen_buffer;       // 代码缓存基址
    size_t code_gen_buffer_size; // 缓存总大小
    void *code_gen_highwater;    // 刷新阈值
    
    // 每线程（vCPU）
    CPUState *cpu;            // 当前 vCPU
};
// 每个 vCPU 线程有独立的 TCGContext
```

---

## 4. 前端 — Guest 到 IR

### 4.1 tcg_gen_* API

```c
// tcg/tcg-op.c — IR 生成 API
// 前端调用这些函数生成 IR 指令

tcg_gen_mov_i64(dst, src)     // 移动: dst = src
tcg_gen_movi_i64(dst, imm)    // 立即数: dst = imm
tcg_gen_add_i64(dst, a, b)    // 加法: dst = a + b
tcg_gen_addi_i64(dst, a, imm) // 立即数加: dst = a + imm
tcg_gen_sub_i64(dst, a, b)    // 减法
tcg_gen_and_i64(dst, a, b)    // 按位与
tcg_gen_ld_i64(dst, base, off) // 加载: dst = *(base + off)
tcg_gen_st_i64(src, base, off) // 存储: *(base + off) = src
tcg_gen_qemu_ld_i64(dst, addr, memop) // Guest 内存加载
tcg_gen_qemu_st_i64(src, addr, memop) // Guest 内存存储

// 智能优化: tcg_gen_add_i64 检测 b=0 时优化为 mov
//           tcg_gen_addi_i64 检测 imm=0 时优化为 mov
```

### 4.2 ARM64 前端示例

```c
// target/arm/tcg/translate-a64.c
// ARM64 前端将每条 Guest 指令翻译为 TCG IR 序列:

// ADD Xd, Xn, Xm:
tcg_gen_add_i64(cpu_reg(s, rd), cpu_reg(s, rn), cpu_reg(s, rm));

// LDR Xt, [Xn, #imm]:
tcg_gen_addi_i64(addr, cpu_reg(s, rn), imm);
tcg_gen_qemu_ld_i64(cpu_reg(s, rt), addr, memop);

// cpu_reg(s, n): 返回映射到 Guest X0-X30 的 TCGTemp 全局变量
```

---

## 5. 优化遍

### 5.1 活性分析（Liveness Analysis）

```c
// tcg/tcg.c — 三遍活性分析

// 遍 0: liveness_pass_0() (tcg.c:3804)
// 将 TEMP_TB 降级为 TEMP_EBB（如果仅在单个 EBB 内使用）
// 减少寄存器压力

// 遍 1: liveness_pass_1() (tcg.c:3883)
// 反向遍历 IR，标记每个 op 的操作数为 DEAD 或 SYNC
// DEAD_ARG: 此操作后该临时变量不再使用 → 可释放寄存器
// SYNC_ARG: 需要写回内存（跨 basic block 活性）

// 遍 2: liveness_pass_2() (tcg.c:4270)
// 前向遍历，移除死代码（输出全 DEAD 的操作→discard）
```

### 5.2 常量折叠与窥孔优化

```c
// tcg/optimize.c:3000 — tcg_optimize()
// 优化内容:
//   1. 常量传播: 操作数全为常量时计算结果
//   2. 常量折叠: add(x, 0) → mov(x)
//   3. 死代码消除: 输出未使用的操作
//   4. 拷贝传播: mov(a, b) 后续使用 a 替换为 b
//   5. 条件简化: 已知条件的分支优化
```

---

## 6. 寄存器分配器

### 6.1 tcg_reg_alloc_op() — 通用操作分配

```c
// tcg/tcg.c:5133+
static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op)
{
    // 1. 获取操作定义（输入/输出/常量参数数）
    // 2. 加载输入操作数到寄存器:
    //    - TEMP_VAL_CONST: 可作为立即数或加载到寄存器
    //    - TEMP_VAL_MEM: 从栈帧加载 (temp_load)
    //    - TEMP_VAL_REG: 已在寄存器中
    // 3. 满足约束: 检查寄存器类约束（通用/特定/配对）
    // 4. 分配输出寄存器: 优先使用 output_pref
    // 5. 调用 all_outop[] 表发射宿主指令
    // 6. 释放死操作数寄存器
    // 7. SYNC_ARG: 将结果写回栈帧
}
```

### 6.2 tcg_reg_alloc_mov() — MOV 优化

```c
// tcg/tcg.c:4923+
static void tcg_reg_alloc_mov(TCGContext *s, const TCGOp *op)
{
    // 常量传播: src 为常量 → 直接 movi 到 dst
    // 寄存器复用: src 死亡 → 直接将 src 寄存器赋给 dst（零开销）
    // 避免 mov: 如果 src 和 dst 已在同一寄存器 → 无操作
    // 类型截断: otype != itype 时可能需要截断
}
```

### 6.3 寄存器分配策略

```
分配策略:
  1. 偏好匹配: 优先使用 output_pref 中的寄存器
  2. 死值复用: 输入操作数在本 op 后死亡 → 复用其寄存器作为输出
  3. 溢出/恢复: 无空闲寄存器时溢出到栈帧（temp_save/temp_load）
  4. 保留寄存器: reserved_regs 不参与分配（如 TCG 栈指针、ENV 指针）
  5. 约束满足: 特定操作要求特定寄存器（如 x86 div 需要 rax/rdx）
```

---

## 7. 后端 — IR 到宿主机器码

### 7.1 后端分发表

```c
// tcg/tcg.c:1158-1234 — all_outop[]
// 每个 TCGOpcode 对应一个后端发射函数:
//   INDEX_op_add → tgen_add
//   INDEX_op_sub → tgen_sub
//   INDEX_op_qemu_ld → tcg_out_qemu_ld
// 后端通过 tcg_out_* 系列函数发射宿主指令
```

### 7.2 AArch64 后端示例

```c
// tcg/aarch64/tcg-target.c.inc

// tcg_out_ldst() (1207+): 发射 LDR/STR 指令
// tcg_out_qemu_ld_direct() (1760+): Guest 加载 → 宿主 LDR
// tcg_out_qemu_st_direct() (1797+): Guest 存储 → 宿主 STR

// 地址翻译:
//   softmmu 模式: 先查 TLB → 命中则 LDR, 未命中则 call helper
//   user 模式: 直接 LDR (Guest 地址 + guest_base)
```

### 7.3 Prologue/Epilogue

```c
// tcg/aarch64/tcg-target.c.inc:3467+
// tcg_target_qemu_prologue()
// 生成共享入口代码:
//   1. 保存 callee-saved 寄存器 (X19-X28, LR, FP)
//   2. 设置 TCG 栈帧
//   3. 加载 env 指针到固定寄存器
//   4. 跳转到 TB 代码
// Epilogue: 恢复寄存器并返回到 C 代码
```

---

## 8. 翻译块（Translation Block）

### 8.1 TranslationBlock 结构

```c
// include/exec/translation-block.h:46-150
struct TranslationBlock {
    vaddr pc;          // Guest PC（虚拟地址）
    uint64_t cs_base;  // 目标特定数据（ARM: 扩展 flags）
    uint32_t flags;    // 翻译上下文标志
    uint32_t cflags;   // 编译标志 (CF_*)
    uint16_t size;     // Guest 代码大小（字节）
    uint16_t icount;   // Guest 指令计数
    struct tb_tc tc;   // 宿主代码 {ptr, size}
    
    // TB 链接（直接跳转）
    uint16_t jmp_reset_offset[2];  // 原始跳转目标偏移
    uint16_t jmp_insn_offset[2];   // 直接跳转指令偏移
    uintptr_t jmp_target_addr[2];  // 跳转目标地址
    
    // TB 链表（用于失效）
    uintptr_t jmp_list_head;       // 入跳转链表头
    uintptr_t jmp_list_next[2];    // 链表节点
    uintptr_t jmp_dest[2];         // 出跳转目标 TB
    QemuSpin jmp_lock;             // 保护链接操作
};
```

### 8.2 cflags 编译标志

```
CF_COUNT_MASK  — 最大指令数 (0=512)
CF_NO_GOTO_TB  — 禁止 TB 链接
CF_SINGLE_STEP — GDB 单步
CF_PCREL       — PC 无关代码（可在不同虚拟地址复用）
CF_PARALLEL    — 并行上下文（MTTCG 多线程安全）
CF_INVALID     — TB 已失效（设置后不可再链接）
CF_BP_PAGE     — 页面有断点
```

---

## 9. 执行循环

### 9.1 cpu_exec_loop() — 主循环

```c
// accel/tcg/cpu-exec.c:933-1000
cpu_exec_loop(CPUState *cpu, SyncClocks *sc)
{
    while (!cpu_handle_exception(cpu, &ret)) {     // 异常处理
        while (!cpu_handle_interrupt(cpu, &last_tb)) { // 中断检查
            // 1. 获取 Guest CPU 状态
            TCGTBCPUState s = cpu->cc->tcg_ops->get_tb_cpu_state(cpu);
            
            // 2. 断点检查
            check_for_breakpoints(cpu, s.pc, &s.cflags);
            
            // 3. TB 查找（哈希表 + JIT 缓存）
            tb = tb_lookup(cpu, s);
            
            // 4. 未找到 → 翻译生成
            if (tb == NULL) {
                tb = tb_gen_code(cpu, s);
                // 插入 JIT 缓存
                jc->array[h].tb = tb;
            }
            
            // 5. TB 链接（直接跳转优化）
            if (last_tb) {
                tb_add_jump(last_tb, tb_exit, tb);
            }
            
            // 6. 执行 TB
            cpu_loop_exec_tb(cpu, tb, s.pc, &last_tb, &tb_exit);
        }
    }
}
```

### 9.2 TB 查找层次

```
层级 1: tb_jmp_cache（per-vCPU 哈希表，O(1)）
  → PC 直接哈希，最快路径
  
层级 2: tcg_tb_lookup（全局 QHT 哈希表）
  → PC + flags + cflags 匹配
  
层级 3: tb_gen_code（翻译生成）
  → 调用前端翻译 + 优化 + 后端代码生成
```

---

## 10. TB 生成流程

```c
// accel/tcg/translate-all.c:261-520
TranslationBlock *tb_gen_code(CPUState *cpu, TCGTBCPUState s)
{
    // 1. 物理地址解析
    phys_pc = get_page_addr_code_hostp(env, s.pc, &host_pc);
    
    // 2. 分配 TB 结构
    tb = tcg_tb_alloc(tcg_ctx);
    // 缓存满 → tb_flush 清空所有 TB
    
    // 3. 初始化 TB 元数据
    tb->pc = s.pc;  tb->flags = s.flags;  tb->cflags = s.cflags;
    
    // 4. 前端翻译（Guest → IR）
    //    setjmp_gen_code → gen_intermediate_code()
    //    逐条解码 Guest 指令 → 生成 TCG IR
    
    // 5. 优化遍
    //    tcg_optimize() — 常量折叠/传播/死代码消除
    
    // 6. 活性分析
    //    liveness_pass_0/1/2 — 标记 DEAD/SYNC
    
    // 7. 后端代码生成（IR → Host）
    //    tcg_reg_alloc_op/mov — 寄存器分配
    //    tcg_out_* — 发射宿主指令
    
    // 8. 注册 TB
    //    tcg_tb_insert() — 插入全局哈希表
    //    tb_page_add() — 关联物理页面（用于失效）
    
    return tb;
}
```

---

## 11. 代码缓存管理

### 11.1 区域结构

```c
// tcg/region.c:61-75
struct tcg_region_state {
    void *start_aligned;   // 缓存起始（对齐）
    void *after_prologue;  // prologue 之后的可用起始
    size_t n;              // 区域数量
    size_t size;           // 单个区域大小
    size_t stride;         // size + guard 页大小
    size_t total_size;     // 总缓存大小
    size_t current;        // 当前分配区域索引
    size_t agg_size_full;  // 已满区域累计大小
};
```

### 11.2 区域管理

```c
// tcg/region.c
// tcg_region_init() (737-760): 初始化代码缓存
//   默认大小: 系统模式 256MB，用户模式 128MB
//   分为 N 个区域（N = vCPU 数），每个 vCPU 独占一个区域
//   区域间有 guard 页防止越界

// tcg_region_alloc() (374+): 为 vCPU 分配新区域
//   当前区域满 → 分配下一个区域
//   所有区域满 → 触发 tb_flush

// tcg_region_prologue_set() (851-886): 设置 prologue 后的起始
//   prologue 代码写入第一个区域开头
//   所有 TB 共享同一个 prologue
```

### 11.3 TB 失效

```c
// accel/tcg/tb-maint.c
// tb_invalidate_phys_page_range__locked() (1112+):
//   物理页面内容变化时失效该页的所有 TB
//   断开 TB 链接 → 标记 CF_INVALID → 从页面列表移除

// queue_tb_flush() (801):
//   代码缓存满时请求全量刷新
//   安全点执行: 所有 vCPU 停止 → 清空缓存 → 重置区域
```

---

## 12. TB 链接（Block Chaining）

```
TB 链接消除查找开销:
  TB_A 的结尾是跳转到 TB_B
  → 直接 patch TB_A 的跳转指令指向 TB_B 的宿主代码

goto_tb 实现:
  1. 翻译时: 在 TB 结尾生成跳转指令（占位）
  2. 链接时: tb_add_jump() patch 跳转目标
  3. 失效时: tb_reset_jump() 恢复原始跳转（回到 dispatcher）

最多 2 个出口:
  jmp_insn_offset[0]: 条件跳转 taken
  jmp_insn_offset[1]: 条件跳转 not-taken / 顺序执行

链接保护:
  jmp_lock 自旋锁保护链接操作
  CF_INVALID 位阻止链接到已失效 TB
```

---

## 13. 完整翻译流水线

```
Guest 指令 (ARM64)
  │
  ▼ 前端 (translate-a64.c)
TCG IR (中间表示)
  │ tcg_gen_add_i64, tcg_gen_qemu_ld_i64, ...
  ▼ 优化 (optimize.c)
优化后 IR
  │ 常量折叠、拷贝传播、死代码消除
  ▼ 活性分析 (tcg.c:3804-4270)
标注 DEAD/SYNC 的 IR
  │ liveness_pass_0/1/2
  ▼ 寄存器分配 (tcg.c:4923-5300)
分配宿主寄存器的 IR
  │ tcg_reg_alloc_op/mov，溢出/恢复
  ▼ 后端 (aarch64/tcg-target.c.inc)
宿主机器码 (AArch64)
  │ tcg_out_*, prologue/epilogue
  ▼ 写入代码缓存
TranslationBlock (可执行)
  │
  ▼ 执行 (cpu-exec.c)
cpu_exec_loop → tb_lookup → 执行 → tb_add_jump → 循环
```

---

## 14. 小结

| 组件 | 实现 |
|------|------|
| **TCG IR** | TCGOp 链表 + TCGTemp 临时变量，统一 32/64 位操作，支持向量 |
| **前端** | tcg_gen_* API 将 Guest 指令转为 IR，每个目标架构独立实现 |
| **优化** | tcg_optimize() 常量折叠/传播/死代码消除 + 三遍活性分析 |
| **寄存器分配** | 线性扫描分配器，支持偏好/约束/溢出，tcg_reg_alloc_op/mov |
| **后端** | 每个宿主架构独立实现 tcg_out_*，通过 all_outop[] 表分发 |
| **TranslationBlock** | Guest PC → Host 代码映射，支持最多 2 出口链接 |
| **代码缓存** | 多区域设计（per-vCPU），满时全量刷新，guard 页保护 |
| **执行循环** | 三级查找（JIT cache → QHT → gen）+ TB 链接消除查找 |
| **Prologue** | 共享入口代码：保存寄存器→设置 env→跳转 TB |
