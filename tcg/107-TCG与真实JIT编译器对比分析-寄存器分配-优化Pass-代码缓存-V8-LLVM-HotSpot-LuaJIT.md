# Doc 107: QEMU TCG 与真实 JIT 编译器对比分析

## 文档信息
- **组件**: TCG JIT 编译器设计, 寄存器分配, 优化 Pass, 代码缓存
- **源码版本**: QEMU 11.0.50
- **对比目标**: V8 TurboFan, LLVM ORC JIT, Java HotSpot C2, LuaJIT
- **分析日期**: 2025-07
- **归档目录**: tcg/

---

## 目录
1. [引言与定位](#1-引言与定位)
2. [编译器分层架构对比](#2-编译器分层架构对比)
3. [中间表示 (IR) 设计对比](#3-中间表示-ir-设计对比)
4. [优化 Pass 对比](#4-优化-pass-对比)
5. [寄存器分配对比](#5-寄存器分配对比)
6. [代码缓存与生命周期管理](#6-代码缓存与生命周期管理)
7. [投机优化与去优化](#7-投机优化与去优化)
8. [分层编译](#8-分层编译)
9. [并发与线程安全](#9-并发与线程安全)
10. [编译延迟与吞吐量权衡](#10-编译延迟与吞吐量权衡)
11. [设计取舍总结](#11-设计取舍总结)
12. [源码实现索引](#12-源码实现索引)

---

## 1. 引言与定位

### 1.1 QEMU TCG 的设计目标

QEMU TCG (Tiny Code Generator) 是一个 **动态二进制翻译器 (DBT)**，其核心设计目标与通用 JIT 编译器截然不同:

| 维度 | QEMU TCG | 通用 JIT (V8/LLVM/HotSpot) |
|------|----------|---------------------------|
| **输入** | 任意 Guest ISA 机器码 | 单一高级语言字节码/AST |
| **语义** | 精确模拟 Guest 行为 | 优化语言规范允许的行为 |
| **正确性** | 必须 100% 精确 (含副作用) | 可利用 UB 做推测优化 |
| **延迟要求** | 极低 (首次翻译即执行) | 可接受预热延迟 |
| **确定性** | 需要 (replay 支持) | 不要求 |
| **代码寿命** | 短 (SMC/TLB flush 频繁) | 长 (函数长期驻留) |

### 1.2 各 JIT 编译器简介

| 编译器 | 宿主 | 特点 |
|--------|------|------|
| **V8 TurboFan** | Chrome/Node.js | Sea-of-nodes SSA, 投机优化, 内联 |
| **LLVM ORC JIT** | 通用 | 完整编译管线, 多 Pass, 强优化 |
| **HotSpot C2** | Java VM | 分层编译, 逃逸分析, 去虚拟化 |
| **LuaJIT** | Lua VM | Trace 编译, 线性 IR, 快照去优化 |
| **QEMU TCG** | 系统模拟器 | 单 Pass 翻译, TB 粒度, 极低延迟 |

---

## 2. 编译器分层架构对比

### 2.1 QEMU TCG 流水线

```
Guest Binary → 前端解码 → TCG IR (线性列表) → tcg_optimize() → 活性分析 → 寄存器分配+发射 → Host Code
                                                   ↑
                                            单个 TB 范围内
```

源码: `tcg/tcg.c:6556-6705` (`tcg_gen_code()`)

```c
// tcg_gen_code() 主流程:
1. tcg_optimize(s)           // 常量折叠/拷贝传播/死代码
2. reachable_code_pass(s)    // 删除不可达代码
3. liveness_pass_0(s)        // 调用参数活性
4. liveness_pass_1(s)        // 全局活性分析
5. liveness_pass_2(s)        // 输出活性传播
6. 逐 Op 寄存器分配 + 发射   // 单遍扫描
```

### 2.2 V8 TurboFan 流水线

```
JavaScript → Parser → AST → Bytecode (Ignition)
                                ↓ (热函数)
                         TurboFan Pipeline:
                         BytecodeGraphBuilder → Sea-of-Nodes IR
                         → Inlining → Type Narrowing → Load Elimination
                         → Escape Analysis → Simplified Lowering
                         → Effect Linearization → Register Allocation (Linear Scan)
                         → Code Generation → Relocation
```

### 2.3 LLVM ORC JIT 流水线

```
Source → LLVM IR (SSA) → Canonicalization → SROA → GVN → LICM → Loop Unroll
      → Instruction Selection (SelectionDAG/GlobalISel)
      → Register Allocation (Greedy/PBQP)
      → Machine Code → Link/Relocate
```

### 2.4 HotSpot C2 流水线

```
Bytecode → Interpreter (profiling) → C1 (快速编译)
                                       ↓ (计数器达阈值)
                                    C2 (Ideal Graph):
                                    Parse → IGVN → Escape Analysis
                                    → Loop Opts → Matching → Register Allocation
                                    → Code Generation
```

### 2.5 LuaJIT 流水线

```
Lua Bytecode → Interpreter (trace recording)
                    ↓ (热循环)
              SSA IR (线性) → FOLD Engine → Allocation Sinking
              → Register Allocation (Linear Scan) → Machine Code
              + Snapshot Points (deopt guards)
```

### 2.6 阶段数对比

| 编译器 | 优化阶段数 | 编译单元 | 编译触发 |
|--------|-----------|----------|----------|
| QEMU TCG | 3-5 (固定) | Translation Block (~数十条指令) | **每个 BB 首次执行** |
| V8 TurboFan | 15-20+ | 函数 | 热度计数器阈值 |
| LLVM ORC | 30-50+ | 函数/模块 | 按需 |
| HotSpot C2 | 20-30 | 方法 | 调用/回边计数器 |
| LuaJIT | 5-8 | Trace (循环路径) | 循环计数器 |

---

## 3. 中间表示 (IR) 设计对比

### 3.1 QEMU TCG IR

**特征**: 线性操作列表, 非 SSA, 可变临时变量

```c
// include/tcg/tcg.h:346-420
struct TCGOp {
    TCGOpcode opc;          // 操作码
    unsigned nargs;         // 参数数
    TCGArg args[MAX_ARGS];  // 参数 (temp 索引)
    TCGLifeData life;       // 活性信息 (后填)
    TCGRegSet output_pref[2]; // 输出寄存器偏好
    QTAILQ_ENTRY(TCGOp) link; // 双向链表
};
```

**关键设计选择**:
- **非 SSA**: 同一个 TCGTemp 可被多次赋值, 简化前端生成
- **线性链表**: Op 按生成顺序排列, 优化 Pass 可删除/替换节点
- **固定宽度参数**: 最多 `MAX_OPC_PARAM` 个参数, 避免动态分配
- **TB 作用域**: 所有 temp 的生命周期不超出一个 TB

源码: `tcg/tcg-op.c:40-105` (IR 生成 API, `tcg_emit_op`)

### 3.2 V8 TurboFan: Sea-of-Nodes

```
特征: SSA, 图结构, 值/效果/控制三维依赖
- 每个节点是一个操作 (Value/Effect/Control)
- 边表示数据依赖 (value edge) 和副作用顺序 (effect edge)
- 控制流通过 Control edge 表达
- 允许节点在图中自由移动 (只要依赖满足)
```

### 3.3 LLVM IR: SSA + CFG

```
特征: 显式 SSA (phi 节点), 模块/函数/BB/指令层次
- 强类型系统
- Phi 节点处理控制流合并
- Metadata 附加优化提示
- 多阶段: LLVM IR → SelectionDAG → MachineIR → MCInst
```

### 3.4 LuaJIT IR: 线性 SSA

```
特征: 编号 SSA 引用, 双向链 (类似 TCG 但有 SSA 属性)
- 操作引用通过整数编号
- FOLD Engine 做即时 CSE + 常量折叠
- Snapshot 记录去优化点的完整状态
```

### 3.5 IR 对比表

| 特性 | TCG IR | V8 Sea-of-Nodes | LLVM IR | LuaJIT IR |
|------|--------|-----------------|---------|-----------|
| SSA | ❌ | ✅ | ✅ | ✅ |
| 图结构 | 线性链表 | 有向图 | CFG+指令链 | 线性数组 |
| Phi 节点 | ❌ (无需) | 隐式 (merge) | ✅ | ❌ (trace) |
| 类型系统 | i32/i64/i128/vec | 抽象(Type) | 丰富(iN/fp/vec/struct) | Tagged |
| 跨 BB 分析 | 仅 EBB 内 | 全函数 | 全函数 | 全 trace |
| 内存模型 | 无 (顺序) | Effect chain | TBAA/Alias | Alias |
| 副作用表达 | 隐含顺序 | Effect edge | Memory SSA | 顺序 |

### 3.6 TCG 非 SSA 的设计原因

```
// optimize.c:850-862 注释
// "We only optimize across extended basic blocks"
// 即: 不跨控制流合并点优化 → 无需 phi → 无需 SSA
```

**原因分析**:
1. **翻译速度优先**: SSA 构建 (插入 phi + 重命名) 需要额外 Pass
2. **TB 粒度小**: 典型 TB 只有 5-50 条 Guest 指令, SSA 收益小
3. **Guest 寄存器映射**: ARM64 有 31 个 GPR, 直接映射为 global TCGTemp, 天然不需 phi
4. **确定性**: 简单的线性 IR 保证编译结果确定性 (replay 需要)

---

## 4. 优化 Pass 对比

### 4.1 QEMU TCG 优化 Pass

源码: `tcg/optimize.c:34-61`, `877-900`

| Pass | 实现 | 说明 |
|------|------|------|
| 常量折叠 | `fold_add/sub/and/or/...` | 编译时求值常量表达式 |
| 拷贝传播 | `fold_mov` + `ts_info` | 追踪 temp 的当前值来源 |
| 掩码追踪 | `z_mask/s_mask` | 追踪已知零位/符号位 |
| 死代码消除 | `liveness_pass` + `fold_*` | 删除结果无用的 Op |
| 交换律重排 | `swap_commutative` | 常量移到右操作数 |
| 代数简化 | `fold_*` 系列 | x+0=x, x&0=0, x|~0=~0 |
| 不可达代码删除 | `reachable_code_pass` | 条件常量时删除死路径 |
| 内存拷贝传播 | `fold_qemu_ld/st` | BB 内 load 消除 |

**范围限制** (`finish_bb()`/`finish_ebb()` 重置状态):
- 所有优化局限于 **Extended Basic Block (EBB)** 内
- 跨控制流合并点不传播任何信息
- 内存相关优化在 BB 边界重置

### 4.2 V8 TurboFan 优化 Pass

| Pass | 说明 |
|------|------|
| Inlining | 内联热函数 (基于 profile) |
| Type Narrowing | 利用类型反馈收窄类型 |
| Load Elimination | 消除冗余加载 |
| Escape Analysis | 标量替换堆分配 |
| Redundancy Elimination | GVN + CSE |
| Loop Peeling/Unrolling | 循环优化 |
| Simplified Lowering | 类型特化 (Int32 vs Float64) |
| Dead Code Elimination | 全图范围 |
| Branch Elimination | 条件常量化 |

### 4.3 LLVM 优化 Pass (O2 级别)

| 类别 | 代表 Pass |
|------|-----------|
| 标量 | SROA, GVN, InstCombine, SCCP, Reassociate |
| 循环 | LICM, Loop Unroll, Loop Vectorize, IndVarSimplify |
| 过程间 | Inlining, Argument Promotion, Dead Arg Elim |
| 分析 | Alias Analysis (BasicAA/TBAA), ScalarEvolution |
| 低级 | CodeGen Prepare, Tail Call Opt, Machine LICM |

### 4.4 HotSpot C2 优化

| Pass | 说明 |
|------|------|
| IGVN (Iterative GVN) | 迭代全局值编号 |
| Escape Analysis | 堆分配消除 |
| Loop Predication | 循环不变检查外提 |
| Range Check Elimination | 数组边界检查消除 |
| Superword | SLP 向量化 |
| Devirtualization | 利用 CHA 去虚拟化 |
| Null Check Elimination | 空指针检查消除 |

### 4.5 对比表

| 优化 | TCG | V8 TurboFan | LLVM | HotSpot C2 | LuaJIT |
|------|-----|-------------|------|------------|--------|
| 常量折叠 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 拷贝传播 | ✅ (EBB) | ✅ (全图) | ✅ (全函数) | ✅ (全方法) | ✅ |
| 死代码消除 | ✅ | ✅ | ✅ | ✅ | ✅ |
| CSE/GVN | ❌ | ✅ | ✅ | ✅ | ✅ (FOLD) |
| 循环优化 (LICM) | ❌ | ✅ | ✅ | ✅ | ❌ (trace) |
| 循环展开 | ❌ | ✅ | ✅ | ✅ | ❌ |
| 内联 | ❌ | ✅ | ✅ | ✅ | ✅ (trace) |
| 逃逸分析 | ❌ | ✅ | ❌(Pass) | ✅ | ✅ (Sink) |
| 向量化 | ❌ | ❌ | ✅ (SLP/Loop) | ✅ (Super) | ❌ |
| 别名分析 | 极简 | Effect chain | TBAA/AA | C2 Alias | 简单 |
| 投机优化 | ❌ | ✅ | ❌ | ✅ | ✅ |
| 类型特化 | ❌ | ✅ | ❌(已typed) | ✅ | ✅ |

### 4.6 TCG 不做这些优化的原因

1. **翻译单元太小**: TB 通常 5-50 条指令, 无循环/无函数调用, 循环优化无意义
2. **Guest 语义精确**: 不能推测 Guest 行为, 无法做逃逸分析或去虚拟化
3. **编译延迟**: 每条 Guest 路径首次执行即需翻译, 不能花 ms 级优化
4. **无 profile 数据**: TCG 不收集执行频率, 无法做 PGO
5. **跨 TB 优化代价高**: TB 可被独立失效, 跨 TB 依赖会导致连锁失效

---

## 5. 寄存器分配对比

### 5.1 QEMU TCG: 单遍本地分配器

源码: `tcg/tcg.c` — `tcg_reg_alloc()`, `tcg_reg_alloc_op()`

```c
// 算法伪代码:
for each Op in IR_list:
    1. 释放当前 Op 死亡的 input temps (活性分析标记)
    2. 为 input operands 分配寄存器:
       - 如果 temp 已在寄存器中且满足约束 → 直接使用
       - 否则从 free_regs & required_regs 中分配
       - 如果无空闲 → 溢出一个已占用寄存器到栈帧
    3. 为 output operands 分配寄存器 (类似)
    4. 发射宿主机器码
```

**关键特征**:
- **单遍**: 从前到后一次扫描, 边分配边发射
- **贪心**: 优先使用 `preferred_regs` (减少 MOV)
- **溢出策略**: 按固定顺序选择牺牲寄存器 (非启发式)
- **约束驱动**: 每个 Op 的约束 (`TCGArgConstraint`) 指定允许的寄存器集
- **无回溯**: 决策即最终, 不会重做

源码: `tcg/aarch64/tcg-target.c.inc:47-72` (AArch64 寄存器分配顺序)

### 5.2 V8 TurboFan: 线性扫描 (Linear Scan)

```
算法: Poletto & Sarkar 线性扫描变体
1. 计算活跃区间 (live range) for each virtual register
2. 按区间起点排序
3. 扫描: 分配物理寄存器, 冲突时溢出最远使用的
4. 分裂长活跃区间 (split at use positions)
5. 处理固定寄存器约束 (calling convention)
```

### 5.3 LLVM: Greedy + Bundle 分配器

```
算法: Greedy Register Allocator (默认)
1. 构建干涉图 (interference graph) 的紧凑表示
2. 按优先级 (spill weight) 排序虚拟寄存器
3. 贪心分配: 尝试分配, 冲突时:
   a. 尝试 evict 低权重寄存器
   b. 尝试 split 活跃区间
   c. 溢出到栈
4. 支持 Register Bundle (tied registers)
5. 可选: PBQP (Partitioned Boolean Quadratic Programming) 分配器
```

### 5.4 HotSpot C2: Chaitin-Briggs 图着色

```
算法: Chaitin-Briggs 图着色变体
1. 构建完整干涉图
2. Simplify: 度数 < K 的节点入栈
3. Spill: 选择高度节点溢出
4. Select: 出栈分配颜色
5. Coalesce: 合并拷贝相关寄存器
6. 迭代直到无溢出
```

### 5.5 LuaJIT: 线性扫描

```
算法: 修改版线性扫描
- 利用 SSA 形式简化活跃区间计算
- 寄存器 hint 传播 (减少 MOV)
- Rematerialization (常量不溢出, 重计算)
```

### 5.6 对比表

| 特性 | TCG | V8 | LLVM Greedy | HotSpot C2 | LuaJIT |
|------|-----|-----|-------------|------------|--------|
| **算法** | 单遍贪心 | 线性扫描 | 贪心+分裂 | 图着色 | 线性扫描 |
| **复杂度** | O(n) | O(n log n) | O(n² worst) | O(n²+) | O(n log n) |
| **活跃分析** | 后向 liveness | 区间计算 | 区间+分裂 | 干涉图 | SSA 隐含 |
| **溢出决策** | 固定顺序 | 最远下次使用 | 权重启发式 | 度数+权重 | 启发式 |
| **分裂支持** | ❌ | ✅ | ✅ | ❌(重迭代) | ❌ |
| **合并(Coalesce)** | preferred_regs | ✅ | ✅ | ✅ | hint |
| **重物化** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **编译时间** | ~μs | ~ms | ~10ms | ~10ms | ~μs |

### 5.7 TCG 简单分配器的合理性

1. **TB 很小**: 典型 TB 只有 10-50 个 TCG Op, 活跃同时数 < 寄存器数
2. **Global TCGTemp**: ARM64 x0-x30 映射为 global temp, 始终在固定寄存器中
3. **编译延迟**: 任何增加延迟的算法都不可接受 (影响首次执行)
4. **代码质量可接受**: 对 DBT 来说, 翻译开销 << 执行时间
5. **溢出代价低**: Global temp (Guest 寄存器) 的 "溢出" 只是写回 CPUState 结构

---

## 6. 代码缓存与生命周期管理

### 6.1 QEMU TCG 代码缓存

源码: `tcg/region.c:456-485`

| 参数 | 值 | 说明 |
|------|-----|------|
| 最小缓冲区 | 1 MiB | `MIN_CODE_GEN_BUFFER_SIZE` |
| 用户模式默认 | 128 MiB | 单 region |
| 系统模式默认 | ~1 GiB | 多 region (减少全局 flush) |
| 最大限制 | `MAX_CODE_GEN_BUFFER_SIZE` | 平台相关 |
| Guard page | 每 region 末尾 | 防止溢出 |
| Highwater | `end - TCG_HIGHWATER` | 接近满时触发 flush |

**Region 分区** (源码: `tcg/region.c:425-453`):
```
系统模式:
┌──────────────────────────────────────────────────────┐
│ Region 0 │ Guard │ Region 1 │ Guard │ ... │ Region N │
└──────────────────────────────────────────────────────┘
  ↑ vCPU 0            ↑ vCPU 1         ↑ vCPU N

用户模式:
┌──────────────────────────────────────────────────────┐
│                    Single Region                      │
└──────────────────────────────────────────────────────┘
```

**失效策略** (源码: `tcg/region.c:407-423`):
- **非 LRU**: 不追踪单个 TB 的最近使用
- **Region 整体重置**: `tcg_region_reset_all()` 清除全部代码
- **触发条件**: Region 满 (达到 highwater)、SMC (自修改代码)、TLB flush
- **TB 树销毁**: 重置时所有 TB 的查找树/链接结构一并清除

### 6.2 V8 代码缓存

| 特性 | 说明 |
|------|------|
| 分配 | 每函数独立 Code 对象, GC 管理 |
| 失效 | Deopt 时标记为无效, GC 回收 |
| 大小 | 动态增长, 无固定上限 |
| 生命周期 | 与 JSFunction 关联, 可被重新编译 |
| Tiering | Ignition → TurboFan, 旧代码可被替换 |

### 6.3 LLVM ORC JIT 代码缓存

| 特性 | 说明 |
|------|------|
| 分配 | JITDylib + MemoryManager, 按 section 分配 |
| 失效 | 手动 remove/replace module |
| 大小 | 按需分配, 系统内存为上限 |
| 重定位 | 支持完整 ELF/MachO 重定位 |

### 6.4 HotSpot 代码缓存

| 特性 | 说明 |
|------|------|
| Code Cache | 固定大小 (默认 ~256 MB) |
| 分区 | Non-profiled (C2), Profiled (C1), Non-method (stubs) |
| 失效 | Deopt/aging 触发, sweeper 线程清理 |
| 满处理 | 停止编译, emergency flush |

### 6.5 对比表

| 特性 | TCG | V8 | LLVM ORC | HotSpot |
|------|-----|-----|----------|---------|
| **总大小** | 128M-1G (固定) | 动态 | 动态 | ~256M (固定) |
| **粒度** | Region/全局 | 函数 | Module | 方法 |
| **失效策略** | 全 Region 重置 | GC + Deopt | 手动 | Sweeper + Deopt |
| **碎片处理** | 无 (整体重置) | GC 压缩 | 无 | Sweeper |
| **SMC 支持** | ✅ (TB 级失效) | N/A | N/A | N/A |
| **写保护** | RX (执行时只读) | RX | RWX→RX | RX |

---

## 7. 投机优化与去优化

### 7.1 QEMU TCG: 无投机优化

TCG **不做任何投机优化**:
- 不假设类型不变
- 不假设分支走向
- 不假设内存不被其他 CPU 修改
- 每条 Guest 指令精确翻译其完整语义

**原因**: Guest 行为完全不可预测, 任何假设都可能错误

### 7.2 V8 TurboFan: 深度投机

```
投机优化示例:
1. 类型投机: 假设 x 是 Smi (小整数) → 生成整数运算
2. 形状投机: 假设 obj 的 hidden class 不变 → 直接偏移访问
3. 内联缓存: 假设调用目标不变 → 内联展开
4. 分支投机: 假设某分支总是 true → 优化热路径

去优化 (Deoptimization):
- 检查失败时跳转到 Deopt 点
- 从 FrameState 重建解释器状态
- 回退到 Ignition 重新执行
- 收集新 profile 后可重新优化
```

### 7.3 LuaJIT: Trace 投机

```
投机方式:
- 记录热循环的执行路径 (trace)
- 在路径上的每个分支点插入 Guard
- Guard 失败 → Snapshot restore → 回到解释器
- Side trace: 从 Guard 失败点开启新 trace
```

### 7.4 对比

| 特性 | TCG | V8 | HotSpot | LuaJIT |
|------|-----|-----|---------|--------|
| 投机 | ❌ | ✅ (类型/形状) | ✅ (类型/CHA) | ✅ (路径) |
| Guard 检查 | N/A | Deopt check | Uncommon trap | Guard IR |
| 去优化 | N/A | FrameState | Interpreter state | Snapshot |
| Profile 收集 | ❌ | ✅ (Feedback) | ✅ (Counter/MDO) | ✅ (热度) |
| 重编译 | ❌ | ✅ | ✅ | ✅ (side trace) |

---

## 8. 分层编译

### 8.1 QEMU TCG: 无分层

```
TCG 只有一层:
Guest Code → 一次性翻译 → Host Code (固定质量)

无解释器层 (不逐条模拟)
无热度检测
无重新优化
```

### 8.2 V8: 2 层 (Ignition + TurboFan)

```
Tier 0: Ignition (字节码解释器, 收集 feedback)
Tier 1: TurboFan (优化编译, 利用 feedback)
         ↕ (OSR: On-Stack Replacement)
Deopt: TurboFan → Ignition (假设失败时)
```

### 8.3 HotSpot: 5 层

```
Tier 0: Interpreter (收集基础 profile)
Tier 1: C1 full optimization
Tier 2: C1 + invocation/backedge counters
Tier 3: C1 + full profiling (MDO)
Tier 4: C2 (基于 Tier 3 的 profile 做最强优化)
```

### 8.4 为什么 TCG 不分层

| 原因 | 说明 |
|------|------|
| **无稳定热点** | OS 内核代码路径变化频繁, 无稳定循环 |
| **SMC 频繁** | JIT 代码/模块加载导致 TB 频繁失效 |
| **全部代码都执行** | DBT 必须翻译所有路径, 不只是热路径 |
| **延迟不可接受** | 解释器层每条指令需 dispatch, 对 ISA 模拟太慢 |
| **复杂度/收益比** | 分层需要 profile 基础设施, 收益不确定 |

**理论上可能的 TCG 分层**:
- Tier 0: 当前 TCG (快速翻译)
- Tier 1: 对热 TB chain 做跨 TB 优化 (消除冗余寄存器保存/恢复)
- 但 QEMU 社区选择不实现, 因为复杂度太高

---

## 9. 并发与线程安全

### 9.1 QEMU MTTCG (Multi-Threaded TCG)

```
模型:
- 每个 vCPU 一个宿主线程
- 每个线程有自己的 TCGContext (避免锁)
- 共享: 代码缓存 Region, TB 查找表
- 同步: tb_lock (TB 创建/链接), TLB flush IPI

代码缓存并发:
- Region 分配: atomic pointer bump
- TB 查找: lock-free hash (RCU)
- TB 链接: cas-based chaining
```

### 9.2 V8: 单线程编译 + 后台编译

```
- 主线程: 执行 JS + Ignition
- 后台线程: TurboFan 编译 (concurrent compilation)
- Code 安装: 在 safepoint 或 OSR 点原子切换
- 共享堆: Write barrier + GC safepoints
```

### 9.3 HotSpot: 编译线程池

```
- Compiler threads (C1/C2) 并行编译不同方法
- 编译完成后在 safepoint 安装 nmethod
- Code cache 由 CodeCache_lock 保护
```

### 9.4 对比

| 特性 | TCG MTTCG | V8 | HotSpot |
|------|-----------|-----|---------|
| 执行并行 | ✅ (多 vCPU) | ❌ (单线程 JS) | ✅ (多 Java 线程) |
| 编译并行 | ✅ (每 vCPU 独立) | ✅ (后台) | ✅ (编译线程) |
| 共享代码 | ✅ (TB 缓存) | ✅ (Code对象) | ✅ (Code Cache) |
| 同步机制 | RCU + atomic | Safepoint | Safepoint + lock |

---

## 10. 编译延迟与吞吐量权衡

### 10.1 编译延迟测量

| 编译器 | 典型编译延迟 | 编译单元大小 | 启动延迟 |
|--------|-------------|-------------|----------|
| **TCG** | **1-10 μs** | ~5-50 指令 (TB) | **极低** (立即执行) |
| V8 TurboFan | 1-100 ms | ~100-10K 字节码 (函数) | 中等 (先解释) |
| LLVM O2 | 10-1000 ms | ~100-100K IR (函数) | 高 (完整编译) |
| HotSpot C2 | 1-50 ms | ~100-10K 字节码 (方法) | 低 (先解释/C1) |
| LuaJIT | 10-100 μs | ~10-1K 字节码 (trace) | 低 (先解释) |

### 10.2 代码质量 vs 编译时间

```
代码质量 (SPEC-like 性能):
LLVM O2 > HotSpot C2 > V8 TurboFan > LuaJIT >> TCG

编译速度:
TCG > LuaJIT >> HotSpot C1 > V8 TurboFan > HotSpot C2 >> LLVM O2

综合效率 (对各自用例):
每个都是其场景的最优解
```

### 10.3 TCG 的权衡选择

```
                  代码质量
                     ↑
                     │    LLVM
                     │      ★
                     │   HotSpot C2
                     │     ★
                     │  V8 TurboFan
                     │    ★
                     │  LuaJIT
                     │   ★
                     │
                     │          TCG ★ (最优 DBT 点)
                     │
                     └──────────────────→ 编译速度
```

TCG 选择了 **极致编译速度**, 牺牲代码质量:
- 对 DBT 来说, 翻译时间是 **直接可感知的延迟** (每个新 BB 都要翻译)
- 翻译后的代码通常只执行几次到几百次 (SMC/flush 导致)
- Guest 程序的真实性能瓶颈通常在 I/O 和内存访问, 非计算

---

## 11. 设计取舍总结

### 11.1 TCG 的核心设计哲学

| 原则 | TCG 选择 | 替代方案 (通用JIT) |
|------|----------|-------------------|
| **编译延迟** | μs 级, 首次即编译 | ms 级, 可延迟 |
| **优化深度** | 局部 (EBB) | 全局 (函数/trace) |
| **正确性** | 精确模拟, 无投机 | 可投机+去优化 |
| **IR 形式** | 非 SSA 线性链表 | SSA 图/数组 |
| **寄存器分配** | 单遍贪心 O(n) | 线性扫描/图着色 O(n²) |
| **代码缓存** | 固定, 全局 flush | 动态, 单个失效 |
| **分层** | 单层 | 2-5 层 |
| **Profile** | 无 | 有 (feedback driven) |
| **确定性** | 保证 | 不保证 |

### 11.2 TCG 的独特优势

1. **通用性**: 支持 20+ Guest 架构 × 多种 Host 架构
2. **可维护性**: 前后端分离, 新增架构只需实现一端
3. **确定性**: 支持 Record/Replay 精确重放
4. **低内存**: 元数据开销极小, 适合嵌入式
5. **快速启动**: 无预热延迟, 适合短生命周期程序
6. **精确性**: 100% 模拟 Guest 行为, 包括异常/中断时序

### 11.3 TCG 的已知劣势

1. **生成代码效率**: 比原生慢 5-20x (典型), LLVM JIT 仅慢 1.5-3x
2. **无热点优化**: 热循环不会越来越快
3. **冗余操作多**: 每个 TB 末尾保存/恢复 Guest 寄存器
4. **内存屏障保守**: 无别名分析, 不能消除冗余 barrier
5. **无向量化**: 不能将标量循环向量化 (但可翻译 Guest SIMD)

### 11.4 可能的改进方向 (社区讨论但未实现)

| 方向 | 说明 | 阻碍 |
|------|------|------|
| TB Chain 优化 | 消除 TB 边界的冗余保存/恢复 | TB 独立失效困难 |
| Profile-guided | 对热 TB 做更强优化 | 增加复杂度, 收益不确定 |
| LLVM 后端 | 用 LLVM 替换 TCG 后端 | 编译延迟不可接受 |
| Trace 编译 | 类似 LuaJIT 记录热路径 | 与精确模拟冲突 |
| 增量优化 | TB Chain 形成后局部优化 | 需要新的代码管理策略 |

---

## 12. 源码实现索引

| 文件 | 关键函数/结构 | 职责 |
|------|--------------|------|
| `tcg/tcg.c:6556-6705` | `tcg_gen_code()` | 完整编译流水线入口 |
| `tcg/tcg.c:tcg_reg_alloc` | `tcg_reg_alloc()` | 寄存器分配核心 |
| `tcg/tcg.c:tcg_reg_alloc_op` | `tcg_reg_alloc_op()` | 逐 Op 分配+发射 |
| `tcg/optimize.c:34-61` | `tcg_optimize()` | 优化 Pass 入口 |
| `tcg/optimize.c:850-862` | `finish_bb()/finish_ebb()` | EBB 边界重置 |
| `tcg/region.c:456-485` | buffer size 定义 | 代码缓冲区大小 |
| `tcg/region.c:407-423` | `tcg_region_reset_all()` | 全局 flush |
| `tcg/region.c:425-453` | region 分区逻辑 | 多 vCPU region |
| `tcg/tcg-op.c:40-105` | `tcg_emit_op()` | IR 生成 API |
| `include/tcg/tcg.h:246-420` | `TCGTemp/TCGOp/TCGContext` | 核心数据结构 |
| `tcg/aarch64/tcg-target.c.inc:47-72` | 寄存器定义 | AArch64 分配顺序 |

---

## 附录: 一句话总结

> **QEMU TCG 是一个为 "翻译一切, 立即执行" 而优化的 JIT,
> 而 V8/LLVM/HotSpot 是为 "只翻译热点, 深度优化" 而设计的 JIT。
> 两者解决的是根本不同的问题。**

---

*文档结束*
