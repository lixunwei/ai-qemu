# 多线程 TCG（MTTCG）深度分析 — 并行执行、TB 失效与内存屏障

> 基于 QEMU 11.0.50 源码分析，聚焦 MTTCG 多 vCPU 并行执行架构、
> TranslationBlock 失效与自修改代码检测、内存排序屏障映射、原子操作回退机制、
> 独占执行上下文、TB 链接与 icount 确定性模式。

---

## 目录

1. [MTTCG 架构与 vCPU 线程模型](#1-mttcg-架构与-vcpu-线程模型)
2. [TCGState 与 MTTCG 初始化](#2-tcgstate-与-mttcg-初始化)
3. [per-vCPU 执行循环](#3-per-vcpu-执行循环)
4. [TB 失效机制](#4-tb-失效机制)
5. [自修改代码（SMC）检测](#5-自修改代码smc检测)
6. [PageDesc 与页面级 TB 追踪](#6-pagedesc-与页面级-tb-追踪)
7. [内存排序屏障（Memory Barrier）](#7-内存排序屏障memory-barrier)
8. [原子操作与回退机制](#8-原子操作与回退机制)
9. [独占执行上下文（Exclusive Context）](#9-独占执行上下文exclusive-context)
10. [TB 链接（Chaining）的多线程安全](#10-tb-链接chaining的多线程安全)
11. [icount 确定性执行模式](#11-icount-确定性执行模式)
12. [总结与架构图](#12-总结与架构图)

---

## 1. MTTCG 架构与 vCPU 线程模型

QEMU TCG 支持两种执行模式：

| 模式 | 线程数 | 适用场景 |
|------|--------|---------|
| **RR（Round-Robin）** | 单线程轮转所有 vCPU | icount 确定性回放、不支持 MTTCG 的目标 |
| **MTTCG** | 每 vCPU 独立线程 | 多核客户机、需要真正并行执行 |

模式选择在 `tcg_accel_ops_init()` 中根据 `qemu_tcg_mttcg_enabled()` 分流：

```c
// accel/tcg/tcg-accel-ops.c:201-220
static void tcg_accel_ops_init(AccelClass *ac) {
    if (qemu_tcg_mttcg_enabled()) {
        ops->create_vcpu_thread = mttcg_start_vcpu_thread;  // 每 vCPU 一个线程
        ops->kick_vcpu_thread = tcg_kick_vcpu_thread;
    } else {
        ops->create_vcpu_thread = rr_start_vcpu_thread;     // 单线程轮转
        ops->kick_vcpu_thread = rr_kick_vcpu_thread;
    }
}
```

---

## 2. TCGState 与 MTTCG 初始化

### TCGState 结构

```c
// accel/tcg/tcg-all.c:50-57
struct TCGState {
    AccelState parent_obj;
    OnOffAuto mttcg_enabled;   // ON_OFF_AUTO_ON/OFF/AUTO
    bool one_insn_per_tb;
    int splitwx_enabled;
    unsigned long tb_size;
};
```

### 初始化决策逻辑

```c
// accel/tcg/tcg-all.c:104-154
static int tcg_init_machine(AccelState *as, MachineState *ms) {
    unsigned max_threads = 1;
    bool mttcg_supported = cc->tcg_ops->mttcg_supported;  // 目标架构是否支持

    switch (s->mttcg_enabled) {
    case ON_OFF_AUTO_AUTO:
        // 自动模式：目标支持 && 未启用 icount → 开启 MTTCG
        if (mttcg_supported && !icount_enabled()) {
            s->mttcg_enabled = ON_OFF_AUTO_ON;
            max_threads = ms->smp.max_cpus;    // 线程数 = vCPU 数
        }
        break;
    case ON_OFF_AUTO_ON:
        if (!mttcg_supported)
            warn_report("Guest not yet converted to MTTCG");
        max_threads = ms->smp.max_cpus;
        break;
    }
    tcg_init(s->tb_size * MiB, s->splitwx_enabled, max_threads);
}
```

### 命令行参数解析

```c
// accel/tcg/tcg-all.c:178-193
static void tcg_set_thread(Object *obj, const char *value, Error **errp) {
    if (strcmp(value, "multi") == 0) {
        if (icount_enabled())
            error_setg(errp, "No MTTCG when icount is enabled");
        else
            s->mttcg_enabled = ON_OFF_AUTO_ON;
    } else if (strcmp(value, "single") == 0) {
        s->mttcg_enabled = ON_OFF_AUTO_OFF;
    }
}
```

MTTCG 启用的前提条件：
1. 目标架构设置 `mttcg_supported = true`（ARM64 已支持）
2. 未启用 icount 模式
3. 用户未显式指定 `thread=single`

---

## 3. per-vCPU 执行循环

### MTTCG 线程主函数

```c
// accel/tcg/tcg-accel-ops-mttcg.c:65-122
static void *mttcg_cpu_thread_fn(void *arg) {
    CPUState *cpu = arg;
    g_assert(!icount_enabled());      // MTTCG 与 icount 互斥

    rcu_register_thread();             // 注册 RCU 读端
    tcg_register_thread();             // 注册 TCG 上下文
    bql_lock();                        // 获取 BQL

    do {
        qemu_process_cpu_events(cpu);  // 处理挂起事件（reset/stop 等）

        if (cpu_can_run(cpu)) {
            bql_unlock();
            r = tcg_cpu_exec(cpu);     // 真正执行：cpu_exec_start + cpu_exec + cpu_exec_end
            bql_lock();

            switch (r) {
            case EXCP_DEBUG:
                cpu_handle_guest_debug(cpu);
                break;
            case EXCP_ATOMIC:          // 原子指令回退
                bql_unlock();
                cpu_exec_step_atomic(cpu);  // 进入独占模式执行
                bql_lock();
            }
        }
    } while (!cpu->unplug || cpu_can_run(cpu));
}
```

### tcg_cpu_exec 封装

```c
// accel/tcg/tcg-accel-ops.c:77-86
int tcg_cpu_exec(CPUState *cpu) {
    cpu_exec_start(cpu);     // 标记 running=true, 与独占段协调
    ret = cpu_exec(cpu);     // 进入 TB 查找→执行循环
    cpu_exec_end(cpu);       // 标记 running=false, 通知独占等待者
    return ret;
}
```

**关键要点**：
- 每个 vCPU 线程持有 BQL 处理事件，释放 BQL 后进入 TCG 执行
- `EXCP_ATOMIC` 表示遇到无法在并行模式下安全执行的原子指令
- 线程循环持续到 `cpu->unplug`（vCPU 热拔出）

---

## 4. TB 失效机制

### 全局 TB 刷新

```c
// accel/tcg/tb-maint.c:770-791
void tb_flush__exclusive_or_serial(void) {
    // 必须在独占上下文或停机状态
    assert(!runstate_is_running() ||
           cpu_in_serial_context(current_cpu));

    CPU_FOREACH(cpu) {
        tcg_flush_jmp_cache(cpu);    // 清空所有 vCPU 的跳转缓存
    }
    qht_reset_size(&tb_ctx.htable, CODE_GEN_HTABLE_SIZE);  // 重置哈希表
    tb_remove_all();                 // 移除所有 TB
    tcg_region_reset_all();          // 重置代码缓存区域
    qatomic_inc(&tb_ctx.tb_flush_count);
}
```

异步触发路径：

```c
// accel/tcg/tb-maint.c:801-808
void queue_tb_flush(CPUState *cs) {
    unsigned tb_flush_count = qatomic_read(&tb_ctx.tb_flush_count);
    async_safe_run_on_cpu(cs, do_tb_flush,
                          RUN_ON_CPU_HOST_INT(tb_flush_count));
}
```

`async_safe_run_on_cpu()` 会进入独占上下文执行，保证所有 vCPU 停止后再刷新。

### 单 TB 失效

```c
// accel/tcg/tb-maint.c:921-959
static void do_tb_phys_invalidate(TranslationBlock *tb, bool rm_from_page_list) {
    // 1. 设置 CF_INVALID 阻止新链接
    qemu_spin_lock(&tb->jmp_lock);
    qatomic_set(&tb->cflags, tb->cflags | CF_INVALID);
    qemu_spin_unlock(&tb->jmp_lock);

    // 2. 从哈希表移除
    qht_remove(&tb_ctx.htable, tb, h);

    // 3. 从页面列表移除
    if (rm_from_page_list) tb_remove(tb);

    // 4. 清除跳转缓存
    tb_jmp_cache_inval_tb(tb);

    // 5. 从两个跳出链表移除
    tb_remove_from_jmp_list(tb, 0);
    tb_remove_from_jmp_list(tb, 1);

    // 6. 取消所有跳入链接
    tb_jmp_unlink(tb);
}
```

跳转缓存失效策略：

```c
// accel/tcg/tb-maint.c:894-913
static void tb_jmp_cache_inval_tb(TranslationBlock *tb) {
    if (tb_cflags(tb) & CF_PCREL) {
        // PC 相对模式：TB 可能在任意虚拟地址，必须全部清除
        CPU_FOREACH(cpu) { tcg_flush_jmp_cache(cpu); }
    } else {
        // 固定 PC：只清除对应哈希槽位
        uint32_t h = tb_jmp_cache_hash_func(tb->pc);
        CPU_FOREACH(cpu) {
            if (qatomic_read(&jc->array[h].tb) == tb)
                qatomic_set(&jc->array[h].tb, NULL);
        }
    }
}
```

### 范围失效

```c
// accel/tcg/tb-maint.c:1024-1035
void tb_invalidate_phys_range(CPUState *cpu, tb_page_addr_t start,
                              tb_page_addr_t last) {
    PAGE_FOR_EACH_TB(start, last, unused, tb, n) {
        tb_phys_invalidate__locked(tb);  // 逐个失效范围内的 TB
    }
}
```

---

## 5. 自修改代码（SMC）检测

当客户机写入代码页时，QEMU 需要检测并处理 SMC（Self-Modifying Code）：

```c
// accel/tcg/tb-maint.c:1056-1103
bool tb_invalidate_phys_page_unwind(CPUState *cpu, tb_page_addr_t addr,
                                    uintptr_t pc) {
    if (!pc || !cpu || !cpu->cc->tcg_ops->precise_smc) {
        tb_invalidate_phys_page(addr);  // 非精确模式：简单失效
        return false;
    }

    // 精确 SMC 处理
    current_tb = tcg_tb_lookup(pc);     // 查找当前正在执行的 TB

    PAGE_FOR_EACH_TB(addr, last, unused, tb, n) {
        if (current_tb == tb && (tb_cflags(current_tb) & CF_COUNT_MASK) != 1) {
            // 当前 TB 正在被修改！
            current_tb_modified = true;
            cpu_restore_state_from_tb(cpu, current_tb, pc);  // 恢复 CPU 状态
        }
        tb_phys_invalidate__locked(tb);
    }

    if (current_tb_modified) {
        // 强制下一次只执行一条指令
        cpu->cflags_next_tb = 1 | CF_NOIRQ | curr_cflags(cpu);
        return true;  // 调用者需要中止当前 TB
    }
    return false;
}
```

**SMC 处理流程**：
1. 客户机写入代码页 → softmmu 写入路径触发
2. 查找页面上的所有 TB → 逐个失效
3. 如果修改的正是当前执行的 TB：
   - 恢复 CPU 状态到修改点
   - 设置 `cflags_next_tb` 只执行一条指令
   - 返回 true 让调用者退出当前 TB

---

## 6. PageDesc 与页面级 TB 追踪

```c
// accel/tcg/tb-maint.c:189-193
struct PageDesc {
    QemuSpin lock;        // 自旋锁保护页面上的 TB 链表
    uintptr_t first_tb;   // 该页面上第一个 TB 的指针（LSB 编码）
};
```

页面描述符通过多级页表组织（类似 MMU 页表）：

```c
// accel/tcg/tb-maint.c:185-187
#define V_L1_MAX_SIZE (1 << V_L1_MAX_BITS)
static void *l1_map[V_L1_MAX_SIZE];  // 一级页表
```

页面锁管理：

```c
// accel/tcg/tb-maint.c:386-476
// tb_lock_pages() / tb_unlock_pages() 管理 PageDesc::lock
// 保证 TB 创建/失效时的页面级互斥
```

**设计要点**：
- 每个物理页面有独立的自旋锁，避免全局锁竞争
- TB 跨页时需要同时锁定多个 PageDesc
- `first_tb` 的 LSB 用于编码 TB 在该页的位置索引

---

## 7. 内存排序屏障（Memory Barrier）

### TCGBar 枚举

```c
// include/tcg/tcg-mo.h:28-46
typedef enum {
    // 访问类型排序
    TCG_MO_LD_LD  = 0x01,   // Load-Load 排序
    TCG_MO_ST_LD  = 0x02,   // Store-Load 排序
    TCG_MO_LD_ST  = 0x04,   // Load-Store 排序
    TCG_MO_ST_ST  = 0x08,   // Store-Store 排序
    TCG_MO_ALL    = 0x0F,   // 全屏障

    // 获取/释放语义
    TCG_BAR_LDAQ  = 0x10,   // Acquire：后续操作不前移
    TCG_BAR_STRL  = 0x20,   // Release：前序操作不后移
    TCG_BAR_SC    = 0x30,   // 顺序一致性：全部不跨越
} TCGBar;
```

### tcg_gen_mb 生成逻辑

```c
// tcg/tcg-op.c:289-306
void tcg_gen_mb(TCGBar mb_type) {
#ifdef CONFIG_USER_ONLY
    bool parallel = tcg_ctx->gen_tb->cflags & CF_PARALLEL;
#else
    // 系统模式始终生成屏障！
    // 即使单 vCPU 也有 I/O 线程并行运行，
    // 缺少内存排序可能导致 virtio 队列读取错误
    bool parallel = true;
#endif
    if (parallel)
        tcg_gen_op1(INDEX_op_mb, 0, mb_type);
}
```

### AArch64 后端映射

```c
// tcg/aarch64/tcg-target.c.inc:1584-1594
static void tcg_out_mb(TCGContext *s, unsigned a0) {
    static const uint32_t sync[] = {
        [0 ... TCG_MO_ALL]            = DMB_ISH | DMB_LD | DMB_ST,  // 全屏障
        [TCG_MO_ST_ST]                = DMB_ISH | DMB_ST,           // Store-Store
        [TCG_MO_LD_LD]                = DMB_ISH | DMB_LD,           // Load-Load
        [TCG_MO_LD_ST]                = DMB_ISH | DMB_LD,           // Load-Store
        [TCG_MO_LD_ST | TCG_MO_LD_LD] = DMB_ISH | DMB_LD,
    };
    tcg_out32(s, sync[a0 & TCG_MO_ALL]);
}
```

**屏障映射表**：

| 客户机屏障 | TCGBar | AArch64 宿主指令 | x86 宿主指令 |
|-----------|--------|-----------------|-------------|
| DMB LD | TCG_MO_LD_LD | `DMB ISHLD` | 无操作（x86 自带 LD-LD 序） |
| DMB ST | TCG_MO_ST_ST | `DMB ISHST` | 无操作（x86 自带 ST-ST 序） |
| DMB SY | TCG_MO_ALL | `DMB ISH` | `MFENCE` |
| DSB | TCG_MO_ALL + TCG_BAR_SC | `DMB ISH` | `MFENCE` |

---

## 8. 原子操作与回退机制

### 原子模板系统

```c
// accel/tcg/atomic_template.h:23-50
// 通过宏模板为 1/2/4/8/16 字节生成原子操作
// DATA_SIZE=16 → SUFFIX=o (Int128), DATA_SIZE=8 → SUFFIX=q (uint64_t)
// 每种大小生成: cmpxchg, xchg, fetch_add, fetch_or, fetch_xor 等
```

### 原子性需求判定

```c
// accel/tcg/ldst_atomicity.c.inc:22-93
static int required_atomicity(CPUState *cpu, uintptr_t p, MemOp memop) {
    MemOp atom = memop & MO_ATOM_MASK;

    switch (atom) {
    case MO_ATOM_NONE:      return MO_8;                   // 无需原子性
    case MO_ATOM_IFALIGN:   return aligned ? size : MO_8;  // 对齐时原子
    case MO_ATOM_WITHIN16:  return within_16 ? size : MO_8; // 16 字节内原子
    case MO_ATOM_SUBALIGN:  return MIN(size, alignment);   // 子对象对齐
    }

    // 关键优化：串行上下文无需宿主原子性
    if (cpu_in_serial_context(cpu))
        return MO_8;

    return atmax;
}
```

### EXCP_ATOMIC 回退机制

当 MTTCG 模式下遇到宿主无法原子执行的操作：

```c
// accel/tcg/cpu-exec.c:549-598
void cpu_exec_step_atomic(CPUState *cpu) {
    start_exclusive();           // 停止所有其他 vCPU
    cpu->running = true;

    // 关键：清除 CF_PARALLEL，在串行上下文翻译
    s.cflags &= ~CF_PARALLEL;
    // 只执行一条指令后返回
    s.cflags |= CF_NO_GOTO_TB | CF_NO_GOTO_PTR | 1;

    tb = tb_lookup(cpu, s);      // 查找/生成非并行版本的 TB
    if (tb == NULL) {
        tb = tb_gen_code(cpu, s); // 生成串行版本代码
    }
    cpu_tb_exec(cpu, tb, &tb_exit);

    cpu->running = false;
    end_exclusive();             // 恢复其他 vCPU 执行
}
```

**回退流程**：
1. vCPU 在并行模式遇到无法原子化的指令 → 返回 `EXCP_ATOMIC`
2. 线程函数调用 `cpu_exec_step_atomic()`
3. 进入 `start_exclusive()` → 等待所有其他 vCPU 暂停
4. 清除 `CF_PARALLEL` → 以串行模式翻译该指令（不需要宿主原子性）
5. 执行一条指令后 → `end_exclusive()` 恢复并行

---

## 9. 独占执行上下文（Exclusive Context）

### start_exclusive

```c
// cpu-common.c:192-233
void start_exclusive(void) {
    g_assert(!current_cpu->running);  // 调用者必须已停止执行

    // 支持嵌套
    if (current_cpu->exclusive_context_count) {
        current_cpu->exclusive_context_count++;
        return;
    }

    qemu_mutex_lock(&qemu_cpu_list_lock);
    exclusive_idle();                  // 等待前一个独占完成

    // 设置 pending_cpus 通知所有运行中的 vCPU 暂停
    qatomic_set(&pending_cpus, 1);
    smp_mb();                          // 写 pending_cpus 先于读 running

    running_cpus = 0;
    CPU_FOREACH(other_cpu) {
        if (qatomic_read(&other_cpu->running)) {
            other_cpu->has_waiter = true;
            running_cpus++;
            qemu_cpu_kick(other_cpu);  // 中断正在执行的 vCPU
        }
    }

    qatomic_set(&pending_cpus, running_cpus + 1);
    while (pending_cpus > 1) {
        qemu_cond_wait(&exclusive_cond, &qemu_cpu_list_lock);
    }
    // 此时所有其他 vCPU 已暂停
}
```

### cpu_exec_start / cpu_exec_end 协作

```c
// cpu-common.c:250-289
void cpu_exec_start(CPUState *cpu) {
    qatomic_set(&cpu->running, true);
    smp_mb();  // 写 running 先于读 pending_cpus

    if (unlikely(qatomic_read(&pending_cpus))) {
        // 有独占请求，需要等待
        qatomic_set(&cpu->running, false);
        exclusive_idle();              // 等待独占完成
        qatomic_set(&cpu->running, true);
    }
}

// cpu-common.c:292-325
void cpu_exec_end(CPUState *cpu) {
    qatomic_set(&cpu->running, false);
    smp_mb();

    if (unlikely(qatomic_read(&pending_cpus))) {
        if (cpu->has_waiter) {
            cpu->has_waiter = false;
            qatomic_set(&pending_cpus, pending_cpus - 1);
            if (pending_cpus == 1)
                qemu_cond_signal(&exclusive_cond);  // 最后一个暂停的通知等待者
        }
    }
}
```

**独占上下文使用场景**：
- `tb_flush` — 全局 TB 刷新
- `cpu_exec_step_atomic` — 原子指令回退
- `async_safe_run_on_cpu` — 安全的跨 CPU 操作

---

## 10. TB 链接（Chaining）的多线程安全

### tb_add_jump — 原子链接

```c
// accel/tcg/cpu-exec.c:616-651
static inline void tb_add_jump(TranslationBlock *tb, int n,
                               TranslationBlock *tb_next) {
    qemu_spin_lock(&tb_next->jmp_lock);

    // 检查目标 TB 是否有效
    if (tb_next->cflags & CF_INVALID)
        goto out_unlock;

    // 原子 CAS 抢占跳转槽位（只有 NULL→tb_next 才成功）
    old = qatomic_cmpxchg(&tb->jmp_dest[n], (uintptr_t)NULL,
                          (uintptr_t)tb_next);
    if (old) goto out_unlock;  // 已被其他线程链接

    // 修补本地跳转指令
    tb_set_jmp_target(tb, n, (uintptr_t)tb_next->tc.ptr);

    // 加入 tb_next 的跳入链表
    tb->jmp_list_next[n] = tb_next->jmp_list_head;
    tb_next->jmp_list_head = (uintptr_t)tb | n;

    qemu_spin_unlock(&tb_next->jmp_lock);
}
```

### tb_set_jmp_target — 修补跳转目标

```c
// accel/tcg/cpu-exec.c:600-614
void tb_set_jmp_target(TranslationBlock *tb, int n, uintptr_t addr) {
    const TranslationBlock *c_tb = tcg_splitwx_to_rx(tb);
    uintptr_t offset = tb->jmp_insn_offset[n];
    uintptr_t jmp_rx = (uintptr_t)tb->tc.ptr + offset;
    uintptr_t jmp_rw = jmp_rx - tcg_splitwx_diff;

    tb->jmp_target_addr[n] = addr;
    tb_target_set_jmp_target(c_tb, n, jmp_rx, jmp_rw);  // 架构相关的跳转修补
}
```

### TranslationBlock 链接元数据

```c
// include/exec/translation-block.h:115-149
struct TranslationBlock {
    QemuSpin jmp_lock;                // 保护链接操作
    uint16_t jmp_reset_offset[2];     // 原始跳转目标偏移
    uint16_t jmp_insn_offset[2];      // 直接跳转指令偏移
    uintptr_t jmp_target_addr[2];     // 跳转目标地址

    // 跳入链表（NULL 终止，LSB 编码槽位号）
    uintptr_t jmp_list_head;          // 所有跳入此 TB 的链表头
    uintptr_t jmp_list_next[2];       // 链表 next 指针
    uintptr_t jmp_dest[2];            // 跳出目标（LSB=1 表示失效中）
};
```

**多线程安全设计**：
- `jmp_lock` 保护目标 TB 的链表操作
- `jmp_dest[n]` 使用 `qatomic_cmpxchg` 确保只链接一次
- `CF_INVALID` 防止向已失效的 TB 建立新链接
- `jmp_dest[]` 的 LSB 标记用于失效时阻止新链接

---

## 11. icount 确定性执行模式

### icount 模式

```c
// accel/tcg/icount-common.c:47-65
static bool icount_sleep = true;
ICountMode use_icount = ICOUNT_DISABLED;  // DISABLED / PRECISE / ADAPTATIVE

static void icount_enable_precise(void) {
    use_icount = ICOUNT_PRECISE;   // 固定 insn→ns 转换（shift 参数）
}

static void icount_enable_adaptive(void) {
    use_icount = ICOUNT_ADAPTATIVE; // 运行时自适应调整 shift
}
```

### 与 MTTCG 的互斥

```c
// accel/tcg/tcg-all.c:127-132
if (mttcg_supported && !icount_enabled()) {
    s->mttcg_enabled = ON_OFF_AUTO_ON;  // 仅在未启用 icount 时开启 MTTCG
}

// accel/tcg/tcg-all.c:182-184
if (icount_enabled())
    error_setg(errp, "No MTTCG when icount is enabled");
```

**互斥原因**：
- icount 需要精确控制每条指令的执行顺序
- MTTCG 的并行执行使得全局指令计数不可确定
- icount 使用 RR 模式（单线程轮转），本质上是协作式调度
- RR 路径中集成了 icount 预算管理（每次执行固定条数指令后切换 vCPU）

---

## 12. 总结与架构图

### MTTCG 执行架构

```
┌─────────────────────────────────────────────────────┐
│                    QEMU 进程                         │
│                                                     │
│  ┌──────────┐  ┌──────────┐       ┌──────────┐     │
│  │ vCPU 0   │  │ vCPU 1   │  ...  │ vCPU N   │     │
│  │ Thread   │  │ Thread   │       │ Thread   │     │
│  ├──────────┤  ├──────────┤       ├──────────┤     │
│  │bql_lock  │  │bql_lock  │       │bql_lock  │     │
│  │events    │  │events    │       │events    │     │
│  │bql_unlock│  │bql_unlock│       │bql_unlock│     │
│  ├──────────┤  ├──────────┤       ├──────────┤     │
│  │exec_start│  │exec_start│       │exec_start│     │
│  │cpu_exec()│  │cpu_exec()│       │cpu_exec()│     │
│  │exec_end  │  │exec_end  │       │exec_end  │     │
│  └────┬─────┘  └────┬─────┘       └────┬─────┘     │
│       │             │                   │           │
│       ▼             ▼                   ▼           │
│  ┌─────────────────────────────────────────────┐    │
│  │           共享 TB 代码缓存                    │    │
│  │  qht_htable (无锁哈希) + PageDesc (页面锁)   │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │           独占执行上下文                       │    │
│  │  start_exclusive() ←→ end_exclusive()        │    │
│  │  用于: tb_flush / atomic 回退 / safe_work    │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### 关键同步机制汇总

| 机制 | 保护对象 | 实现 |
|------|---------|------|
| `PageDesc::lock` | 页面上的 TB 链表 | QemuSpin 自旋锁 |
| `TB::jmp_lock` | TB 跳转链表 | QemuSpin 自旋锁 |
| `qht_htable` | 全局 TB 哈希表 | 无锁并发哈希表（RCU） |
| `exclusive_context` | 全局独占操作 | pending_cpus + condvar |
| `cpu->running` | vCPU 执行状态 | qatomic + smp_mb |
| `CF_INVALID` | TB 有效性 | 原子位设置 |
| `jmp_dest[] CAS` | TB 链接槽位 | qatomic_cmpxchg |

### MTTCG 与 RR 对比

| 特性 | MTTCG | RR（Round-Robin） |
|------|-------|-------------------|
| 线程数 | N（每 vCPU 一个） | 1（共享） |
| 并行度 | 真正并行 | 协作式轮转 |
| 原子指令 | 需要宿主原子支持或回退 | 天然串行安全 |
| 内存屏障 | 必须生成宿主屏障 | 可优化省略 |
| icount | 不兼容 | 兼容 |
| TB CF_PARALLEL | 设置 | 不设置 |
| 性能 | 多核客户机优势大 | 单核或确定性场景 |

---

**关键源文件**：
- `accel/tcg/tcg-all.c` — TCGState、MTTCG 初始化与参数解析
- `accel/tcg/tcg-accel-ops-mttcg.c` — per-vCPU 线程主循环
- `accel/tcg/tcg-accel-ops.c` — MTTCG/RR 分流与 tcg_cpu_exec
- `accel/tcg/tb-maint.c` — TB 失效、页面追踪、SMC 检测
- `accel/tcg/cpu-exec.c` — TB 链接、原子回退（cpu_exec_step_atomic）
- `cpu-common.c` — start_exclusive/end_exclusive、cpu_exec_start/end
- `include/tcg/tcg-mo.h` — TCGBar 内存屏障枚举
- `tcg/tcg-op.c` — tcg_gen_mb 屏障生成
- `tcg/aarch64/tcg-target.c.inc` — AArch64 后端屏障映射
- `accel/tcg/ldst_atomicity.c.inc` — 原子性需求判定
- `accel/tcg/atomic_template.h` — 原子操作模板
- `accel/tcg/icount-common.c` — icount 确定性模式
