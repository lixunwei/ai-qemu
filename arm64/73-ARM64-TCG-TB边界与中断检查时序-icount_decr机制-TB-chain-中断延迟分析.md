# ARM64 TCG 中 TB 边界与中断检查的时序关系

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/73 |
| 分析对象 | Translation Block 边界、中断检查时机、TB chaining 与中断延迟 |
| QEMU 版本 | 11.0.50 |
| 关键源文件 | `accel/tcg/cpu-exec.c`, `accel/tcg/translator.c`, `target/arm/tcg/translate-a64.c` |
| 核心结论 | **中断只在 TB 边界检查，最大延迟 = 1 个 TB 的指令数（≤512条）；但 `icount_decr.u16.high` 机制可在 TB 入口立即触发退出** |

---

## 1. 概述：中断检查的根本约束

### 1.1 TCG 批量翻译模型

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│   TB 1   │────→│   TB 2   │────→│   TB 3   │
│ N1 条指令 │chain│ N2 条指令 │chain│ N3 条指令 │
└──────────┘     └──────────┘     └──────────┘
     ↑                                  ↑
     │                                  │
  中断检查点                         中断检查点
```

关键约束：
- **TB 内部不检查中断**（指令已编译为 host 代码，无检查点）
- **中断只在 TB 边界或 TB 入口处检查**
- TB 最大包含 **512 条 Guest 指令**（CF_COUNT_MASK = 0x1ff）

### 1.2 与真实硬件的对比

| 方面 | 真实 ARM CPU | QEMU TCG |
|------|-------------|----------|
| 检查粒度 | 每条指令间 | 每个 TB 边界 |
| 最大延迟 | 1 个时钟周期 | 1 个 TB (~100-512 条指令) |
| 被触发方式 | 物理 IRQ 线电平 | icount_decr.u16.high 设为 -1 |
| 链式执行中断 | N/A | 破坏 TB chain |

---

## 2. cpu_exec 主循环结构

### 2.1 循环层次

```c
// accel/tcg/cpu-exec.c:933
cpu_exec_loop(CPUState *cpu, SyncClocks *sc) {
    while (!cpu_handle_exception(cpu, &ret)) {     // 异常处理
        while (!cpu_handle_interrupt(cpu, &last_tb)) { // ← 中断检查
            tb = tb_lookup(cpu, s);
            if (!tb) tb = tb_gen_code(cpu, s);     // 翻译新 TB
            if (last_tb) tb_add_jump(last_tb, tb_exit, tb); // TB chain
            cpu_loop_exec_tb(cpu, tb, ...);         // 执行 TB
        }
    }
}
```

### 2.2 执行流程

```
┌─────────────┐
│  cpu_exec() │
└──────┬──────┘
       ▼
┌─────────────────────────────────┐
│ cpu_handle_interrupt()          │  ← TB 间中断检查
│  ├── check interrupt_request   │
│  ├── call cpu_exec_interrupt() │  ← arm_cpu_exec_interrupt
│  └── check exit_request        │
└──────────────┬──────────────────┘
               │ (no interrupt)
               ▼
┌─────────────────────────────────┐
│ tb_lookup() / tb_gen_code()     │  ← 查找/翻译 TB
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ cpu_loop_exec_tb()              │
│  ├── cpu_tb_exec(cpu, tb)       │  ← 执行已编译代码
│  │    └── (执行 N 条 host 指令) │
│  │        ├── TB 入口检查 ──────────→ TB_EXIT_REQUESTED
│  │        ├── goto_tb (chain)    │
│  │        └── exit_tb           │
│  └── 检查 tb_exit 类型          │
└──────────────┬──────────────────┘
               │ (TB 结束)
               ▼
       回到 cpu_handle_interrupt()
```

---

## 3. TB 入口的中断退出机制

### 3.1 gen_tb_start — TB 头部生成

```c
// accel/tcg/translator.c:43
static TCGOp *gen_tb_start(DisasContextBase *db, uint32_t cflags)
{
    TCGv_i32 count = tcg_temp_new_i32();

    // 加载 cpu->neg.icount_decr.u32
    tcg_gen_ld_i32(count, tcg_env,
                   offsetof(CPUState, neg.icount_decr.u32) - sizeof(CPUState));

    if (!(cflags & CF_NOIRQ)) {
        // 生成条件分支：如果 count < 0，跳转到退出标签
        tcg_ctx->exitreq_label = gen_new_label();
        tcg_gen_brcondi_i32(TCG_COND_LT, count, 0, tcg_ctx->exitreq_label);
    }
}
```

### 3.2 gen_tb_end — 退出标签处理

```c
// translator.c:88
static void gen_tb_end(const TranslationBlock *tb, ...)
{
    if (tcg_ctx->exitreq_label) {
        gen_set_label(tcg_ctx->exitreq_label);
        tcg_gen_exit_tb(tb, TB_EXIT_REQUESTED);  // 返回主循环
    }
}
```

### 3.3 伪代码展示 TB 结构

```asm
TB_entry:
    ; === gen_tb_start ===
    load  r_count, [cpu + icount_decr.u32]
    cmp   r_count, 0
    blt   exit_requested           ; ← 负值 = 有中断请求

    ; === 翻译的 Guest 指令 ===
    insn_1:  ...
    insn_2:  ...
    ...
    insn_N:  ...

    ; === TB 结尾 ===
    goto_tb next_tb                ; 或 exit_tb

exit_requested:
    exit_tb(tb, TB_EXIT_REQUESTED) ; 返回主循环检查中断
```

---

## 4. 中断如何打断正在执行的 TB

### 4.1 icount_decr.u16.high 机制

```c
// icount_decr 结构:
union {
    uint32_t u32;     // gen_tb_start 检查这个
    struct {
        uint16_t low;  // icount 剩余指令计数
        uint16_t high; // 中断请求标志 (设为 -1 触发退出)
    } u16;
};
```

当 `u16.high = 0xFFFF`（-1 as int16）时，`u32` 变为负值，TB 入口检查会触发退出。

### 4.2 cpu_interrupt → 设置退出标志

```c
// accel/tcg/tcg-accel-ops.c:96
void tcg_handle_interrupt(CPUState *cpu, int mask)
{
    cpu_set_interrupt(cpu, mask);           // 设置 interrupt_request

    if (!qemu_cpu_is_self(cpu)) {
        qemu_cpu_kick(cpu);                 // 跨线程：唤醒目标 vCPU
    } else {
        // 同线程：直接设置退出标志
        qatomic_set(&cpu->neg.icount_decr.u16.high, -1);
    }
}
```

### 4.3 tcg_kick_vcpu_thread — 跨线程中断

```c
// cpu-exec.c:752
void tcg_kick_vcpu_thread(CPUState *cpu)
{
    // 设置 exit_request（主循环检查）
    qatomic_store_release(&cpu->exit_request, true);
    // 设置 icount_decr.high = -1（TB 入口检查）
    qatomic_store_release(&cpu->neg.icount_decr.u16.high, -1);
}
```

### 4.4 中断延迟分析

```
时间线:
─────────────────────────────────────────────→

TB_A 执行中        │ TB_B 入口         │ 中断处理
(无法中断)         │ 检查 decr < 0     │
                   │ → 退出到主循环     │
   ←──────────────→←───────────────────→
   最大延迟时间      立即响应

   如果 TB_A 有 100 条指令
   且中断在 TB_A 第 1 条时到达
   → 延迟 = 99 条指令的执行时间
```

**最坏情况**：中断在 TB 的第一条指令处到达 → 延迟 = (TB指令数 - 1) 条指令

---

## 5. TB Chain 与中断的交互

### 5.1 TB Chain (goto_tb) 机制

```c
// translate-a64.c:535
static void gen_goto_tb(DisasContext *s, unsigned tb_slot_idx, int64_t diff)
{
    if (use_goto_tb(s, s->pc_curr + diff)) {
        // 直接跳转到下一个 TB（不回主循环）
        tcg_gen_goto_tb(tb_slot_idx);
        gen_a64_update_pc(s, diff);
        tcg_gen_exit_tb(s->base.tb, tb_slot_idx);  // 返回值带 chain 信息
    } else {
        // 间接跳转（回主循环查表）
        gen_a64_update_pc(s, diff);
        tcg_gen_lookup_and_goto_ptr();
    }
}
```

### 5.2 Chain 执行中的中断检查

TB Chain 不经过主循环，但**每个被 chain 的 TB 入口仍有 gen_tb_start 检查**：

```
TB_A → (goto_tb) → TB_B → (goto_tb) → TB_C
                    ↑                    ↑
              入口检查 decr<0       入口检查 decr<0
```

所以即使 TB 被 chain，中断仍会在下一个 TB 入口被检测到。

### 5.3 Chain 被打断的时机

```c
// cpu-exec.c:884
static inline void cpu_loop_exec_tb(CPUState *cpu, TranslationBlock *tb,
                                    ...)
{
    tb = cpu_tb_exec(cpu, tb, tb_exit);
    if (*tb_exit != TB_EXIT_REQUESTED) {
        *last_tb = tb;       // chain 继续
        return;
    }
    // TB_EXIT_REQUESTED → chain 中断
    *last_tb = NULL;         // 不再 chain
    // 回到 cpu_handle_interrupt()
}
```

### 5.4 CPU_INTERRUPT_EXITTB

```c
// cpu_handle_interrupt():
if (cpu_test_interrupt(cpu, CPU_INTERRUPT_EXITTB)) {
    cpu_reset_interrupt(cpu, CPU_INTERRUPT_EXITTB);
    *last_tb = NULL;  // 强制不 chain
}
```

用途：TLB flush、memory map 变更等需要重新翻译时，打断当前 chain。

---

## 6. ARM64 特有的 TB 终止条件

### 6.1 DISAS_* 导致的 TB 终止

| DISAS 类型 | 含义 | TB 出口行为 | 可 chain? |
|-----------|------|------------|:---------:|
| DISAS_NEXT / TOO_MANY | 正常结束 | gen_goto_tb | ✅ |
| DISAS_JUMP | 间接跳转 | lookup_and_goto_ptr | ✅* |
| DISAS_EXIT | 异常/特殊 | exit_tb(NULL, 0) | ❌ |
| DISAS_UPDATE_EXIT | 更新 PC + 退出 | exit_tb(NULL, 0) | ❌ |
| DISAS_UPDATE_NOCHAIN | 更新 PC + 查表 | lookup_and_goto_ptr | ✅* |
| DISAS_WFI | 等待中断 | exit_tb(NULL, 0) | ❌ |
| DISAS_WFE | 等待事件 | helper_wfe → yield | ❌ |
| DISAS_NORETURN | 已处理 | 无 | — |

### 6.2 导致 TB 提前终止的 ARM64 指令

| 指令 | 原因 | DISAS 类型 |
|------|------|-----------|
| ERET | EL 切换 | EXIT |
| HVC/SMC | 异常 | EXIT |
| MSR (特殊) | 状态变更 | UPDATE_EXIT |
| ISB | 指令同步屏障 | UPDATE_EXIT |
| WFI | 等待中断 | WFI |
| WFE | 等待事件 | WFE |
| B/BL (远跳) | 跨页跳转 | JUMP |
| SVC | 系统调用 | NORETURN |
| 条件分支 | 翻译无法继续 | NEXT (结束当前TB) |

### 6.3 CF_NOIRQ 的使用

```c
// cpu_handle_interrupt():
if (cpu->cflags_next_tb != -1 && cpu->cflags_next_tb & CF_NOIRQ) {
    return false;  // 跳过中断检查
}
```

用途：
- 精确异常恢复时需要不被中断打断
- 单步调试的某些情况

---

## 7. MTTCG (多线程 TCG) 下的中断时序

### 7.1 MTTCG 架构

```
┌─────────────┐     ┌─────────────┐
│  vCPU 0     │     │  vCPU 1     │
│  (Thread 0) │     │  (Thread 1) │
│             │     │             │
│  cpu_exec() │     │  cpu_exec() │
└──────┬──────┘     └──────┬──────┘
       │                    │
       │     ┌──────────┐   │
       └─────│   BQL    │───┘
             │ (互斥锁) │
             └──────────┘
```

### 7.2 跨线程中断投递

```
vCPU 0 (iothread)          vCPU 1 (正在执行 TB)
    │                           │
    │ cpu_interrupt(cpu1, IRQ)  │
    │ → cpu_set_interrupt()     │
    │ → qemu_cpu_kick(cpu1)     │  (因为 !qemu_cpu_is_self)
    │     │                     │
    │     └──→ 发送 signal ─────→ 中断 cpu1 的 host 执行
    │                           │ → signal handler
    │                           │ → 设置 exit_request=1
    │                           │ → 设置 icount_decr.high=-1
    │                           │
    │                         [TB 入口] → 检测到 decr<0
    │                           │ → TB_EXIT_REQUESTED
    │                           │ → cpu_handle_interrupt
    │                           │ → arm_cpu_exec_interrupt
    │                           │ → do_interrupt (进入中断)
```

### 7.3 Round-Robin TCG (单线程模式)

在 RR 模式下，所有 vCPU 共享一个线程：

```c
// tcg-accel-ops-rr.c:46
// 调度时检查所有 vCPU 的中断
tcg_kick_vcpu_thread(cpu);
```

RR 模式中断延迟更大（可能需要等待当前 vCPU 的时间片用完）。

---

## 8. icount 模式下的精确中断

### 8.1 icount 原理

```c
// CF_USE_ICOUNT: 每条指令递减 icount_decr.u16.low
// 当 low = 0 时，TB 退出检查是否需要处理 IO/timer

// gen_tb_start 中:
if (cflags & CF_USE_ICOUNT) {
    tcg_gen_sub_i32(count, count, tcg_constant_i32(num_insns));
    // count -= TB指令数 → 如果 count < 0 则退出
}
```

### 8.2 icount 的中断精度

icount 模式下，中断可以被**精确到某一条指令**：

```c
// icount_exit_request():
return cpu->neg.icount_decr.u16.low + cpu->icount_extra == 0;
```

当 timer 到期时：
1. 设置 icount budget = 0
2. 正在执行的 TB 在入口检查时发现 budget 用完
3. 退出到主循环
4. 处理 timer 中断

**icount 模式 = 确定性中断时序**（适合调试和 replay）

---

## 9. 中断延迟的实际影响

### 9.1 典型 TB 大小

ARM64 TB 通常在以下条件之一时终止：
- 遇到分支指令（最常见）
- 跨越页边界
- 达到最大指令数 (512)
- 遇到 ISB/DSB/WFI 等同步指令
- TCG buffer 满

实际中典型 TB 大小：**5-50 条指令**（基本块通常很短）

### 9.2 影响场景

| 场景 | 影响 | 严重度 |
|------|------|:------:|
| Timer 中断精度 | 可能延迟 10-50 条指令 | P3 (通常无感) |
| IPI (核间中断) | 可能延迟 1 个 TB | P3 |
| 外设中断响应 | 可能延迟 1 个 TB | P3 |
| 自旋锁 fairness | 可能影响（MTTCG） | P2 |
| 实时操作系统 | 精度不够 | P2 |

### 9.3 缓解措施

1. **icount 模式**：精确到指令级，但性能更低
2. **TB 大小限制**：分支密集代码自然产生小 TB
3. **ISB/DSB 指令**：Guest OS 在关键路径使用屏障强制 TB 边界
4. **WFI 后立即响应**：WFI 退出后中断检查是第一步

---

## 10. TB 边界与异常级别切换

### 10.1 EL 切换必须终止 TB

以下操作强制结束当前 TB（DISAS_EXIT）：
- ERET（异常返回）
- HVC/SMC（超级调用）
- 写 SCTLR_EL1（修改 MMU/endian）
- 写 HCR_EL2（修改 trap 控制）
- ISB 后的所有系统寄存器修改

原因：TB 的翻译依赖当前 EL/MMU/endian 状态，状态变更后必须重新翻译。

### 10.2 TB cflags 编码当前状态

```c
// get_tb_cpu_state 返回 TB 的查找 key:
// - PC
// - 当前 EL (影响权限检查)
// - MMU idx (影响地址翻译)
// - Endian (影响内存访问)
// - PSTATE.{SS, IL, BTYPE, ...}
```

不同 EL 下的 TB 不能被 chain。

---

## 11. 与规范的差异评估

### 11.1 中断响应延迟

| 规范要求 | QEMU 行为 | 差异 |
|---------|----------|------|
| 中断在指令边界响应 | 在 TB 边界响应 | ⚠️ 延迟增大 |
| WFI 被中断唤醒精确 | WFI 后立即检查中断 | ✅ 精确 |
| 异步异常不在 ISB 前响应 | ISB 终止 TB，下次入口检查 | ✅ 正确 |
| SError 可延迟（imprecise） | TB 边界检查 | ✅ (本身就 imprecise) |

### 11.2 功能正确性

中断延迟**不影响功能正确性**：
- Guest 不能观测到中断的精确 cycle 时机
- ARM 规范允许 IMPLEMENTATION DEFINED 的中断延迟
- 只有 icount 模式需要精确时序（QEMU 已支持）

### 11.3 总结

| 方面 | 评估 |
|------|------|
| 功能正确性 | ✅ 完全正确 |
| 中断响应延迟 | ⚠️ 比真实硬件大（TB 粒度） |
| TB chain 中断检查 | ✅ 每个 TB 入口都检查 |
| 跨线程中断通知 | ✅ signal + atomic 机制 |
| icount 精确模式 | ✅ 指令级精确 |
| EL 切换 TB 终止 | ✅ 强制退出 |

TCG 的 TB 粒度中断检查是**架构级的设计权衡**（性能 vs 精度），不是 bug。对绝大多数 Guest OS（Linux/FreeBSD 等非实时系统）完全透明。
