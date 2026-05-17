# ARM64 TCG 执行循环深度分析：cpu_exec 主循环、TB 查找与链接、中断异常处理、MTTCG 多线程与 icount 指令计数

> 基于 QEMU 11.0.50 源码分析，涵盖 TCG 执行引擎完整运行时：
> cpu_exec 主循环（sigsetjmp 异常恢复、cpu_exec_loop 双层循环、SyncClocks 时钟同步）、
> TB 查找（tb_lookup 两级缓存 jmp_cache→QHT、tb_htable_lookup 哈希查找）、
> TB 执行（cpu_tb_exec→tcg_qemu_tb_exec 进入翻译代码、TB_EXIT_MASK 退出分类）、
> TB 链接（tb_add_jump 原子补丁 + jmp_list 链表）、
> 中断处理（cpu_handle_interrupt 优先级：DEBUG>HALT>RESET>target>EXITTB>exit_request）、
> 异常处理（cpu_handle_exception→do_interrupt 回调→异步退出分类 EXCP_*）、
> MTTCG 多线程模型（mttcg_cpu_thread_fn 每 vCPU 独立线程、EXCP_ATOMIC→step_atomic）、
> RR 轮转模型（rr_cpu_thread_fn 单线程多 CPU、kick timer 时间片切换、BQL 锁定）、
> exclusive 执行区域（start_exclusive/end_exclusive/cpu_exec_start/cpu_exec_end）、
> icount 指令计数（icount_decr 递减计数器、CF_USE_ICOUNT、icount_prepare_for_run/process_data）、
> cpu_exec_step_atomic 原子单步执行。

---

## 目录

1. [架构概述](#1-架构概述)
2. [cpu_exec 主入口](#2-cpu_exec-主入口)
3. [cpu_exec_loop — 双层执行循环](#3-cpu_exec_loop--双层执行循环)
4. [TB 查找 — tb_lookup 两级缓存](#4-tb-查找--tb_lookup-两级缓存)
5. [TB 执行 — cpu_tb_exec](#5-tb-执行--cpu_tb_exec)
6. [TB 链接 — tb_add_jump](#6-tb-链接--tb_add_jump)
7. [中断处理 — cpu_handle_interrupt](#7-中断处理--cpu_handle_interrupt)
8. [异常处理 — cpu_handle_exception](#8-异常处理--cpu_handle_exception)
9. [EXCP_* 异步退出码](#9-excp_-异步退出码)
10. [CF_* 编译标志](#10-cf_-编译标志)
11. [MTTCG 多线程模型](#11-mttcg-多线程模型)
12. [RR 轮转单线程模型](#12-rr-轮转单线程模型)
13. [Exclusive 执行区域](#13-exclusive-执行区域)
14. [icount 指令计数模式](#14-icount-指令计数模式)
15. [SyncClocks 时钟同步](#15-syncclocks-时钟同步)
16. [cpu_exec_step_atomic — 原子单步](#16-cpu_exec_step_atomic--原子单步)
17. [完整执行流程总结](#17-完整执行流程总结)

---

## 1. 架构概述

TCG 执行引擎是 QEMU 翻译执行的核心运行时，负责：

```
QEMU 主循环
  │
  ├── MTTCG 模式：每 vCPU 独立 OS 线程
  │   └── mttcg_cpu_thread_fn → tcg_cpu_exec → cpu_exec
  │
  └── RR 模式：单线程轮转多 vCPU
      └── rr_cpu_thread_fn → tcg_cpu_exec → cpu_exec

cpu_exec() 内部双层循环：
  while (!exception) {          ← 外层：异常处理
    while (!interrupt) {        ← 内层：TB 执行
      tb = tb_lookup(pc, flags)
      if (!tb) tb = tb_gen_code()  ← JIT 编译
      tb_add_jump(last_tb, tb)     ← TB 链接
      cpu_loop_exec_tb(tb)         ← 执行翻译代码
      align_clocks()               ← 时钟同步
    }
  }
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `accel/tcg/cpu-exec.c` | 52-56 | SyncClocks 结构 |
| `accel/tcg/cpu-exec.c` | 71-141 | align_clocks / init_delay_params |
| `accel/tcg/cpu-exec.c` | 195-263 | tb_htable_lookup / tb_lookup |
| `accel/tcg/cpu-exec.c` | 427-491 | cpu_tb_exec |
| `accel/tcg/cpu-exec.c` | 549-598 | cpu_exec_step_atomic |
| `accel/tcg/cpu-exec.c` | 616-651 | tb_add_jump |
| `accel/tcg/cpu-exec.c` | 687-750 | cpu_handle_exception |
| `accel/tcg/cpu-exec.c` | 777-882 | cpu_handle_interrupt |
| `accel/tcg/cpu-exec.c` | 884-928 | cpu_loop_exec_tb |
| `accel/tcg/cpu-exec.c` | 932-1007 | cpu_exec_loop |
| `accel/tcg/cpu-exec.c` | 1009-1046 | cpu_exec_setjmp / cpu_exec |
| `accel/tcg/tcg-accel-ops-mttcg.c` | 65-137 | mttcg_cpu_thread_fn |
| `accel/tcg/tcg-accel-ops-rr.c` | 41-348 | RR 线程与调度 |
| `cpu-common.c` | 183-325 | exclusive 区域 |
| `include/exec/cpu-common.h` | 17-22 | EXCP_* 定义 |
| `include/exec/translation-block.h` | 46-88 | TranslationBlock 与 CF_* |

---

## 2. cpu_exec 主入口

```c
// accel/tcg/cpu-exec.c:1019-1046
int cpu_exec(CPUState *cpu)
{
    SyncClocks sc = { 0 };
    current_cpu = cpu;                    // 1025: 设置当前 CPU

    if (cpu_handle_halt(cpu)) {           // 1027: 检查 halted 状态
        return EXCP_HALTED;
    }

    RCU_READ_LOCK_GUARD();                // 1031: RCU 读锁
    cpu_exec_enter(cpu);                  // 1032: 进入钩子（target 可重写）
    init_delay_params(&sc, cpu);          // 1040: 初始化时钟同步参数

    ret = cpu_exec_setjmp(cpu, &sc);      // 1042: 设置 longjmp 恢复点

    cpu_exec_exit(cpu);                   // 1044: 退出钩子
    return ret;
}
```

### sigsetjmp 异常恢复

```c
// accel/tcg/cpu-exec.c:1009-1017
static int cpu_exec_setjmp(CPUState *cpu, SyncClocks *sc)
{
    if (unlikely(sigsetjmp(cpu->jmp_env, 0) != 0)) {
        cpu_exec_longjmp_cleanup(cpu);    // longjmp 回来后清理
    }
    return cpu_exec_loop(cpu, sc);
}
```

当翻译代码执行中发生异常（如 TLB miss 后的 fault），`cpu_loop_exit()` 调用 `siglongjmp(cpu->jmp_env)` 跳回此处，然后重新进入 `cpu_exec_loop` 处理异常。

---

## 3. cpu_exec_loop — 双层执行循环

```c
// accel/tcg/cpu-exec.c:932-1007
static int cpu_exec_loop(CPUState *cpu, SyncClocks *sc)
{
    int ret;

    // 外层循环：异常处理
    while (!cpu_handle_exception(cpu, &ret)) {        // 938
        TranslationBlock *last_tb = NULL;
        int tb_exit = 0;

        // 内层循环：TB 执行
        while (!cpu_handle_interrupt(cpu, &last_tb)) { // 942
            // 1. 获取 CPU 翻译状态
            TCGTBCPUState s = cpu->cc->tcg_ops->get_tb_cpu_state(cpu); // 944
            s.cflags = cpu->cflags_next_tb;

            // 2. 处理 cflags（icount / 精确 smc / watchpoint）
            if (s.cflags == -1) {                      // 954
                s.cflags = curr_cflags(cpu);
            } else {
                cpu->cflags_next_tb = -1;
            }

            // 3. 检查断点
            if (check_for_breakpoints(cpu, s.pc, &s.cflags)) break;

            // 4. TB 查找
            tb = tb_lookup(cpu, s);                    // 964
            if (tb == NULL) {
                mmap_lock();
                tb = tb_gen_code(cpu, s);              // 970: JIT 编译
                mmap_unlock();
                // 插入 jmp_cache
                h = tb_jmp_cache_hash_func(s.pc);
                jc->array[h].pc = s.pc;
                qatomic_set(&jc->array[h].tb, tb);
            }

            // 5. TB 链接（跨两页的 TB 不链接）
            if (last_tb) {
                tb_add_jump(last_tb, tb_exit, tb);     // 996
            }

            // 6. 执行 TB
            cpu_loop_exec_tb(cpu, tb, s.pc, &last_tb, &tb_exit); // 999

            // 7. 时钟同步
            align_clocks(sc, cpu);                     // 1003
        }
    }
    return ret;
}
```

---

## 4. TB 查找 — tb_lookup 两级缓存

### 第一级：per-CPU jmp_cache（直接映射）

```c
// accel/tcg/cpu-exec.c:227-263
static inline TranslationBlock *tb_lookup(CPUState *cpu, TCGTBCPUState s)
{
    uint32_t hash = tb_jmp_cache_hash_func(s.pc);  // 236: PC 哈希
    CPUJumpCache *jc = cpu->tb_jmp_cache;

    tb = qatomic_read(&jc->array[hash].tb);        // 239: 原子读
    if (likely(tb &&
               jc->array[hash].pc == s.pc &&        // PC 匹配
               tb->cs_base == s.cs_base &&           // cs_base 匹配
               tb->flags == s.flags &&               // flags 匹配
               tb_cflags(tb) == s.cflags)) {         // cflags 匹配
        goto hit;                                    // 快速命中！
    }
    // → 第二级查找
    tb = tb_htable_lookup(cpu, s);                   // 248
    if (tb == NULL) return NULL;                     // 需要 JIT 编译

    // 回填 jmp_cache
    jc->array[hash].pc = s.pc;
    qatomic_set(&jc->array[hash].tb, tb);
    return tb;
}
```

### 第二级：全局 QHT 哈希表

```c
// accel/tcg/cpu-exec.c:195-211
static TranslationBlock *tb_htable_lookup(CPUState *cpu, TCGTBCPUState s)
{
    phys_pc = get_page_addr_code(env, s.pc);        // 203: 虚拟→物理
    h = tb_hash_func(phys_pc, pc, flags, cs_base, cflags); // 208
    return qht_lookup_custom(&tb_ctx.htable, &desc, h, tb_lookup_cmp);
}
```

**两级缓存设计**：
- jmp_cache：O(1) 直接映射，per-CPU 无锁，但可能冲突
- QHT：全局共享，开放寻址哈希表，处理冲突，用物理地址索引

---

## 5. TB 执行 — cpu_tb_exec

```c
// accel/tcg/cpu-exec.c:427-491
static inline TranslationBlock *
cpu_tb_exec(CPUState *cpu, TranslationBlock *itb, int *tb_exit)
{
    const void *tb_ptr = itb->tc.ptr;               // 432: 翻译代码地址

    qemu_thread_jit_execute();                       // 438: JIT 执行模式
    ret = tcg_qemu_tb_exec(cpu_env(cpu), tb_ptr);   // 439: 进入翻译代码！

    cpu->neg.can_do_io = true;                       // 440: 恢复 IO 能力

    // 解析返回值
    last_tb = tcg_splitwx_to_rw((void *)(ret & ~TB_EXIT_MASK));  // 450
    *tb_exit = ret & TB_EXIT_MASK;                   // 451

    // 如果 TB 未被执行（如 icount 耗尽）
    if (*tb_exit > TB_EXIT_IDX1) {                   // 455
        // 恢复 guest PC 到 TB 起始地址
        tcg_ops->synchronize_from_tb(cpu, last_tb);  // 464
    }

    // 单步调试
    if (unlikely(cpu->singlestep_enabled) && cpu->exception_index == -1) {
        cpu->exception_index = EXCP_DEBUG;           // 486
        cpu_loop_exit(cpu);
    }

    return last_tb;
}
```

### TB 退出码

```
ret = tcg_qemu_tb_exec(env, tb_ptr);
  │
  ├── ret & ~TB_EXIT_MASK → last_tb 指针
  └── ret & TB_EXIT_MASK  → 退出原因
      ├── TB_EXIT_IDX0 (0) — 从 goto_tb slot 0 退出（正常链跳转）
      ├── TB_EXIT_IDX1 (1) — 从 goto_tb slot 1 退出（条件分支另一路）
      └── TB_EXIT_REQUESTED (2+) — 被中断请求退出
```

### cpu_loop_exec_tb — 退出后处理

```c
// accel/tcg/cpu-exec.c:884-928
static inline void cpu_loop_exec_tb(CPUState *cpu, TranslationBlock *tb,
                                    vaddr pc, TranslationBlock **last_tb,
                                    int *tb_exit)
{
    tb = cpu_tb_exec(cpu, tb, tb_exit);              // 889: 执行
    if (*tb_exit != TB_EXIT_REQUESTED) {
        *last_tb = tb;                               // 891: 记录用于链接
        return;
    }

    *last_tb = NULL;                                 // 895: 不链接
    if (cpu_loop_exit_requested(cpu)) return;        // 896: 外部请求退出

    // icount 到期：补充并继续
    icount_update(cpu);                              // 911
    cpu->neg.icount_decr.u16.low = insns_left;      // 914: 补充计数
    cpu->icount_extra = cpu->icount_budget - insns_left; // 915

    // 如果剩余指令数不足下一个 TB → 生成精确大小 TB
    if (insns_left > 0 && insns_left < tb->icount) {
        cpu->cflags_next_tb = (tb->cflags & ~CF_COUNT_MASK) | insns_left;
    }
}
```

---

## 6. TB 链接 — tb_add_jump

```c
// accel/tcg/cpu-exec.c:616-651
static inline void tb_add_jump(TranslationBlock *tb, int n,
                               TranslationBlock *tb_next)
{
    qemu_thread_jit_write();                         // 621: JIT 写模式
    qemu_spin_lock(&tb_next->jmp_lock);              // 623

    if (tb_next->cflags & CF_INVALID) goto out;      // 626: 目标 TB 已失效

    // 原子 CAS 抢占跳转槽
    old = qatomic_cmpxchg(&tb->jmp_dest[n], NULL, tb_next); // 630
    if (old) goto out;                               // 已被占用

    // 补丁本机跳转地址
    tb_set_jmp_target(tb, n, tb_next->tc.ptr);       // 637

    // 加入反向链表（用于 TB 失效时撤销链接）
    tb->jmp_list_next[n] = tb_next->jmp_list_head;   // 640
    tb_next->jmp_list_head = (uintptr_t)tb | n;      // 641

    qemu_spin_unlock(&tb_next->jmp_lock);
}
```

**链接机制**：
1. `tb->jmp_dest[0/1]`：两个跳转槽（对应 goto_tb 0/1）
2. `tb_set_jmp_target`：修改 TB 代码中的跳转指令，直接跳到下一个 TB
3. `jmp_list_head/next`：反向链表，当 `tb_next` 被失效时，遍历链表撤销所有指向它的跳转

---

## 7. 中断处理 — cpu_handle_interrupt

```c
// accel/tcg/cpu-exec.c:777-882
static inline bool cpu_handle_interrupt(CPUState *cpu,
                                        TranslationBlock **last_tb)
{
    // CF_NOIRQ TB 跳过中断检查
    if (cpu->cflags_next_tb != -1 && cpu->cflags_next_tb & CF_NOIRQ) {
        return false;                                // 786
    }

    // 清除中断标志（与 tcg_kick_vcpu_thread 配对）
    qatomic_set_mb(&cpu->neg.icount_decr.u16.high, 0); // 795

    if (unlikely(cpu_test_interrupt(cpu, ~0))) {
        bql_lock();

        // 优先级 1：调试中断
        if (CPU_INTERRUPT_DEBUG) {                    // 802
            cpu->exception_index = EXCP_DEBUG;
            return true;
        }

        // 优先级 2：挂起
        if (CPU_INTERRUPT_HALT) {                     // 810
            cpu->halted = 1;
            cpu->exception_index = EXCP_HLT;
            return true;
        }

        // 优先级 3：复位
        if (CPU_INTERRUPT_RESET) {                    // 821
            tcg_ops->cpu_exec_reset(cpu);
            return true;
        }

        // 优先级 4：目标特定中断（IRQ/FIQ 等）
        if (tcg_ops->cpu_exec_interrupt(cpu, interrupt_request)) { // 839
            // 中断已接受，重新开始 TB 执行
            *last_tb = NULL;                          // 855
        }

        // 优先级 5：退出 TB（TLB flush 等触发）
        if (CPU_INTERRUPT_EXITTB) {                   // 858
            *last_tb = NULL;  // 断开 TB 链
        }

        bql_unlock();
    }

    // 最后检查：退出请求 / icount 到期
    if (cpu->exit_request || icount_exit_request(cpu)) { // 874
        cpu->exception_index = EXCP_INTERRUPT;
        return true;
    }
    return false;
}
```

---

## 8. 异常处理 — cpu_handle_exception

```c
// accel/tcg/cpu-exec.c:687-750
static inline bool cpu_handle_exception(CPUState *cpu, int *ret)
{
    // 无异常
    if (cpu->exception_index < 0) {                   // 689
        // replay 模式可能强制执行单条指令触发日志异常
        return false;
    }

    // 异步退出（EXCP_INTERRUPT/EXCP_DEBUG/EXCP_HLT 等）
    if (cpu->exception_index >= EXCP_INTERRUPT) {     // 701
        *ret = cpu->exception_index;
        if (*ret == EXCP_DEBUG) cpu_handle_debug_exception(cpu);
        cpu->exception_index = -1;
        return true;  // 退出 cpu_exec_loop，返回到调用者
    }

    // 同步异常（guest 异常：data abort、syscall 等）
    replay_exception();
    bql_lock();
    tcg_ops->do_interrupt(cpu);                       // 728: target 异常处理
    bql_unlock();
    cpu->exception_index = -1;
    // 继续执行（外层循环再次进入内层循环）
    return false;
}
```

---

## 9. EXCP_* 异步退出码

```c
// include/exec/cpu-common.h:17-22
#define EXCP_INTERRUPT  0x10000  // 异步中断（exit_request）
#define EXCP_HLT        0x10001  // HLT / WFI 指令
#define EXCP_DEBUG      0x10002  // 断点 / 单步
#define EXCP_HALTED     0x10003  // CPU 挂起（等待外部事件）
#define EXCP_YIELD      0x10004  // 让出时间片
#define EXCP_ATOMIC     0x10005  // 需要 stop-the-world 原子执行
```

**分类**：
- `< EXCP_INTERRUPT`：同步 guest 异常（由 `do_interrupt` 处理）
- `>= EXCP_INTERRUPT`：异步退出（直接返回给线程循环）

---

## 10. CF_* 编译标志

```c
// include/exec/translation-block.h:76-88
CF_COUNT_MASK  = 0x000001ff  // 最大指令数（512）
CF_NO_GOTO_TB  = 0x00000200  // 禁止 goto_tb 链接
CF_NO_GOTO_PTR = 0x00000400  // 禁止 goto_ptr 链接
CF_SINGLE_STEP = 0x00000800  // GDB 单步生效
CF_MEMI_ONLY   = 0x00001000  // 仅插桩内存操作
CF_USE_ICOUNT  = 0x00002000  // icount 模式
CF_INVALID     = 0x00004000  // TB 已失效
CF_PARALLEL    = 0x00008000  // 并行上下文（MTTCG）
CF_NOIRQ       = 0x00010000  // 不可中断 TB
CF_PCREL       = 0x00020000  // PC 相对寻址
CF_BP_PAGE     = 0x00040000  // 代码页有断点
CF_CLUSTER_MASK = 0xff000000  // 集群 ID（高 8 位）
```

---

## 11. MTTCG 多线程模型

### 线程主循环

```c
// accel/tcg/tcg-accel-ops-mttcg.c:65-122
static void *mttcg_cpu_thread_fn(void *arg)
{
    CPUState *cpu = arg;
    g_assert(!icount_enabled());                      // 71: MTTCG 不支持 icount

    rcu_register_thread();                            // 73
    tcg_register_thread();                            // 77
    bql_lock();                                       // 79

    do {
        qemu_process_cpu_events(cpu);                 // 89: 处理待处理事件

        if (cpu_can_run(cpu)) {
            bql_unlock();
            r = tcg_cpu_exec(cpu);                    // 94: → cpu_exec()
            bql_lock();

            switch (r) {
            case EXCP_DEBUG:                          // 97
                cpu_handle_guest_debug(cpu);
                break;
            case EXCP_HALTED:                         // 100
                break;
            case EXCP_ATOMIC:                         // 106
                bql_unlock();
                cpu_exec_step_atomic(cpu);            // 108: exclusive 单步
                bql_lock();
                break;
            }
        }
    } while (!cpu->unplug || cpu_can_run(cpu));       // 115

    tcg_cpu_destroy(cpu);
    return NULL;
}
```

### 线程创建

```c
// accel/tcg/tcg-accel-ops-mttcg.c:124-137
void mttcg_start_vcpu_thread(CPUState *cpu)
{
    tcg_cpu_init_cflags(cpu, max_cpus > 1);           // 129: CF_PARALLEL 如果多 CPU
    // 每 vCPU 创建一个线程 "CPU N/TCG"
    qemu_thread_create(cpu->thread, thread_name,
                       mttcg_cpu_thread_fn, cpu, QEMU_THREAD_JOINABLE);
}
```

---

## 12. RR 轮转单线程模型

### Kick Timer 调度

```c
// accel/tcg/tcg-accel-ops-rr.c:62-99
static QEMUTimer *rr_kick_vcpu_timer;
static CPUState *rr_current_cpu;

// 周期性触发，强制当前 CPU 退出
static void rr_kick_next_cpu(void)                    // 71-82
{
    cpu = qatomic_read(&rr_current_cpu);
    if (cpu) cpu_exit(cpu);                           // 77: 设置 exit_request
}

static void rr_start_kick_timer(void)                 // 90-99
{
    // 定时器周期 = TCG_KICK_PERIOD（纳秒）
    timer_mod(rr_kick_vcpu_timer, rr_next_kick_time());
}
```

### 单线程主循环

```c
// accel/tcg/tcg-accel-ops-rr.c:180-314
static void *rr_cpu_thread_fn(void *arg)
{
    bql_lock();                                       // 191
    // 等待机器启动
    while (cpu_is_stopped(first_cpu)) {               // 200
        qemu_cond_wait_bql(first_cpu->halt_cond);
    }
    rr_start_kick_timer();                            // 210

    cpu = first_cpu;
    while (1) {                                       // 214
        rr_wait_io_event();                           // 235: 等待 IO 事件
        rr_deal_with_unplugged_cpus();                // 236

        if (icount_enabled()) {
            icount_account_warp_timer();              // 246
            icount_handle_deadline();                 // 251
            cpu_budget = icount_percpu_budget(cpu_count); // 253
        }

        // 内层循环：逐 CPU 执行
        while (cpu && cpu_work_list_empty(cpu)) {     // 262
            qatomic_set_mb(&rr_current_cpu, cpu);     // 267

            if (cpu_can_run(cpu)) {
                bql_unlock();
                if (icount_enabled()) {
                    icount_prepare_for_run(cpu, cpu_budget); // 283
                }
                r = tcg_cpu_exec(cpu);                // 285: → cpu_exec()
                if (icount_enabled()) {
                    icount_process_data(cpu);         // 287
                }
                bql_lock();
                // 处理 EXCP_DEBUG / EXCP_ATOMIC
            }
            cpu = CPU_NEXT(cpu);                      // 307: 下一个 CPU
        }
        qatomic_set(&rr_current_cpu, NULL);           // 311
    }
}
```

### 线程创建（共享线程）

```c
// accel/tcg/tcg-accel-ops-rr.c:317-348
void rr_start_vcpu_thread(CPUState *cpu)
{
    tcg_cpu_init_cflags(cpu, false);                  // 324: 无 CF_PARALLEL

    if (!single_tcg_cpu_thread) {
        // 第一个 CPU：创建线程 "ALL CPUs/TCG"
        qemu_thread_create(cpu->thread, "ALL CPUs/TCG",
                           rr_cpu_thread_fn, cpu, ...);
    } else {
        // 后续 CPU：复用同一线程
        cpu->thread = single_tcg_cpu_thread;          // 340
        cpu->halt_cond = single_tcg_halt_cond;        // 341
    }
}
```

---

## 13. Exclusive 执行区域

### start_exclusive — 独占开始

```c
// cpu-common.c:192-233
void start_exclusive(void)
{
    g_assert(!current_cpu->running);                  // 198

    // 可重入计数
    if (current_cpu->exclusive_context_count) {       // 200
        current_cpu->exclusive_context_count++;
        return;
    }

    qemu_mutex_lock(&qemu_cpu_list_lock);
    exclusive_idle();                                 // 206: 等待前一个 exclusive 完成

    qatomic_set(&pending_cpus, 1);                   // 209: 标记有 exclusive 请求
    smp_mb();

    // 踢出所有正在运行的 CPU
    CPU_FOREACH(other_cpu) {
        if (qatomic_read(&other_cpu->running)) {
            other_cpu->has_waiter = true;
            running_cpus++;
            qemu_cpu_kick(other_cpu);                // 218: 发送退出信号
        }
    }

    qatomic_set(&pending_cpus, running_cpus + 1);
    while (pending_cpus > 1) {                       // 223: 等待所有 CPU 停止
        qemu_cond_wait(&exclusive_cond, &qemu_cpu_list_lock);
    }
    qemu_mutex_unlock(&qemu_cpu_list_lock);
}
```

### end_exclusive — 独占结束

```c
// cpu-common.c:236-247
void end_exclusive(void)
{
    if (--current_cpu->exclusive_context_count) return; // 可重入
    qatomic_set(&pending_cpus, 0);                   // 244
    qemu_cond_broadcast(&exclusive_resume);          // 245: 唤醒所有等待 CPU
}
```

### cpu_exec_start / cpu_exec_end — 执行屏障

```c
// cpu-common.c:250-289 / 292-325
void cpu_exec_start(CPUState *cpu)
{
    qatomic_set(&cpu->running, true);                // 254
    smp_mb();

    // 如果有 exclusive 请求
    if (pending_cpus) {
        if (!cpu->has_waiter) {
            cpu->running = false;                    // 279: 先停下
            exclusive_idle();                        // 280: 等待 exclusive 完成
            cpu->running = true;                     // 282: 恢复
        }
    }
}

void cpu_exec_end(CPUState *cpu)
{
    qatomic_set(&cpu->running, false);               // 294
    if (pending_cpus && cpu->has_waiter) {
        pending_cpus--;                              // 318
        if (pending_cpus == 1) {
            qemu_cond_signal(&exclusive_cond);       // 320: 通知 start_exclusive
        }
    }
}
```

**使用场景**：
- `EXCP_ATOMIC` → `cpu_exec_step_atomic()` 使用 `start_exclusive/end_exclusive`
- TLB 跨 CPU 同步刷新
- 自修改代码处理

---

## 14. icount 指令计数模式

### 计数器布局

```c
// include/hw/core/cpu.h:342-376
// cpu->neg.icount_decr 是一个联合体：
//   u32  — 完整 32 位
//   u16.low  — 低 16 位：剩余指令数（每执行一条递减）
//   u16.high — 高 16 位：-1 表示强制退出（kick）

// 配合 cpu->icount_extra：存储溢出的大预算
// 配合 cpu->icount_budget：本轮总预算
```

### 工作流程

```
1. icount_prepare_for_run(cpu, budget)
   → cpu->icount_budget = budget
   → cpu->neg.icount_decr.u16.low = MIN(budget, 0xFFFF)
   → cpu->icount_extra = budget - low

2. TB 执行中：每条指令递减 icount_decr.u16.low

3. low == 0 时：
   → 翻译代码检查 → TB_EXIT_REQUESTED
   → cpu_loop_exec_tb：
     a) 如果 icount_extra > 0：补充 low，继续执行
     b) 如果 budget 耗尽：返回上层循环

4. icount_process_data(cpu)
   → 更新全局 icount 计数
```

### CF_USE_ICOUNT 对翻译的影响

```c
// accel/tcg/translator.c:43-104
// 当 CF_USE_ICOUNT 设置时：
// - 每条指令开头插入 icount 递减检查
// - TB 最大指令数可能受限于 icount_decr.u16.low
// - gen_tb_start 生成退出检查代码
```

---

## 15. SyncClocks 时钟同步

```c
// accel/tcg/cpu-exec.c:52-56
typedef struct SyncClocks {
    int64_t diff_clk;            // guest 时钟与 host 时钟差值
    int64_t last_cpu_icount;     // 上次记录的 icount
    int64_t realtime_clock;      // 上次记录的真实时钟
} SyncClocks;
```

### align_clocks — 节流

```c
// accel/tcg/cpu-exec.c:71-98
static void align_clocks(SyncClocks *sc, CPUState *cpu)
{
    if (!icount_align_option) return;                // 75

    cpu_icount = cpu->icount_extra + cpu->neg.icount_decr.u16.low;
    sc->diff_clk += icount_to_ns(sc->last_cpu_icount - cpu_icount);
    sc->last_cpu_icount = cpu_icount;

    if (sc->diff_clk > VM_CLOCK_ADVANCE) {           // 83: guest 跑太快
        nanosleep(&sleep_delay, &rem_delay);          // 88: 让 host 休眠
    }
}
```

**作用**：当 guest 时钟领先 host 时钟超过 `VM_CLOCK_ADVANCE` 时，通过 `nanosleep` 让出 CPU 时间，避免 guest 时间过快。

---

## 16. cpu_exec_step_atomic — 原子单步

```c
// accel/tcg/cpu-exec.c:549-598
void cpu_exec_step_atomic(CPUState *cpu)
{
    if (sigsetjmp(cpu->jmp_env, 0) == 0) {
        start_exclusive();                           // 555: 停止所有其他 CPU

        TCGTBCPUState s = get_tb_cpu_state(cpu);
        s.cflags = curr_cflags(cpu);
        s.cflags &= ~CF_PARALLEL;                    // 564: 非并行模式
        s.cflags |= CF_NO_GOTO_TB | CF_NO_GOTO_PTR | 1; // 566: 单条指令

        tb = tb_lookup(cpu, s);
        if (tb == NULL) tb = tb_gen_code(cpu, s);

        cpu_exec_enter(cpu);
        cpu_tb_exec(cpu, tb, &tb_exit);              // 584: 执行一条指令
        cpu_exec_exit(cpu);
    }

    cpu->running = false;
    end_exclusive();                                 // 597: 恢复其他 CPU
}
```

**触发条件**：MTTCG 模式下遇到需要原子性保证的指令（如 LDXR/STXR 序列在非原生支持的情况下），返回 `EXCP_ATOMIC`，然后在 exclusive 上下文中以非并行模式重新执行单条指令。

---

## 17. 完整执行流程总结

```
QEMU 启动
  │
  ├── MTTCG: qemu_thread_create("CPU N/TCG", mttcg_cpu_thread_fn)
  │   每 vCPU 一个 OS 线程
  │
  └── RR: qemu_thread_create("ALL CPUs/TCG", rr_cpu_thread_fn)
      所有 vCPU 共享一个线程

线程主循环 (MTTCG/RR):
  │
  ├── 事件处理 (qemu_process_cpu_events)
  ├── cpu_can_run(cpu)？
  │   ├── No → 等待
  │   └── Yes → bql_unlock → tcg_cpu_exec → cpu_exec()
  │
  └── cpu_exec():
      1. cpu_handle_halt → 如果 halted 直接返回 EXCP_HALTED
      2. cpu_exec_enter
      3. init_delay_params (SyncClocks)
      4. sigsetjmp(cpu->jmp_env)  ←─── longjmp 回到这里
      5. cpu_exec_loop:
      │
      │  外层 while (!cpu_handle_exception):
      │    │  ← guest 异常：do_interrupt → 继续
      │    │  ← 异步退出（EXCP_INTERRUPT 等）→ 返回
      │    │
      │    └── 内层 while (!cpu_handle_interrupt):
      │         ├── get_tb_cpu_state → (pc, flags, cflags)
      │         ├── check_for_breakpoints
      │         ├── tb_lookup(pc, flags)
      │         │   ├── jmp_cache 命中 → O(1)
      │         │   ├── QHT 命中 → 回填 jmp_cache
      │         │   └── 未命中 → tb_gen_code (JIT)
      │         ├── tb_add_jump(last_tb → tb)  ← 运行时补丁
      │         ├── cpu_loop_exec_tb:
      │         │   ├── cpu_tb_exec → tcg_qemu_tb_exec
      │         │   │   │
      │         │   │   ├── TB 执行（翻译后的 host 代码）
      │         │   │   │   ├── 正常跳转 → 链到下一个 TB → 继续
      │         │   │   │   ├── 中断/退出 → TB_EXIT_REQUESTED
      │         │   │   │   └── 异常 → cpu_loop_exit → siglongjmp
      │         │   │   │
      │         │   │   └── 返回 (last_tb, tb_exit)
      │         │   │
      │         │   └── icount 到期 → 补充或生成精确大小 TB
      │         │
      │         └── align_clocks (guest 太快则 nanosleep)
      │
      6. cpu_exec_exit
      7. 返回退出码 → 线程循环处理

EXCP_ATOMIC 特殊路径（仅 MTTCG）:
  → cpu_exec_step_atomic
  → start_exclusive (停止所有其他 vCPU)
  → 编译并执行 1 条指令（CF_NO_GOTO_TB | ~CF_PARALLEL）
  → end_exclusive (恢复所有 vCPU)
```

---

## 交叉参考

- [42-ARM64-TCG前端后端代码生成深度分析](42-ARM64-TCG前端后端代码生成深度分析-IR中间表示-翻译循环-优化Pass-寄存器分配与AArch64代码发射.md) — tb_gen_code、translator_loop、后端代码发射
- [43-ARM64-TCG-softmmu-TLB深度分析](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — TLB 查找/填充/MMIO 路径
- [41-ARM64-EL切换TCG翻译变化深度分析](41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md) — hflags/TB 键/TB 链断裂

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
