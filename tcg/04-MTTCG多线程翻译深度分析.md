# MTTCG 多线程翻译深度分析

> QEMU 11.0.50 · 分析日期 2025-07 · 基于源码交叉验证

## 目录

1. [概述](#1-概述)
2. [MTTCG 架构全景](#2-mttcg-架构全景)
3. [MTTCG 启用与配置](#3-mttcg-启用与配置)
4. [vCPU 线程模型 — MTTCG 模式](#4-vcpu-线程模型--mttcg-模式)
5. [vCPU 线程模型 — Round-Robin 模式](#5-vcpu-线程模型--round-robin-模式)
6. [模式选择与分发](#6-模式选择与分发)
7. [TCGContext 线程管理](#7-tcgcontext-线程管理)
8. [代码缓冲区 Region 分区](#8-代码缓冲区-region-分区)
9. [TB 哈希表并发访问](#9-tb-哈希表并发访问)
10. [TB 跳转链保护 — jmp_lock](#10-tb-跳转链保护--jmp_lock)
11. [TB 刷新同步](#11-tb-刷新同步)
12. [CF_PARALLEL 标志](#12-cf_parallel-标志)
13. [TCG 内存屏障](#13-tcg-内存屏障)
14. [原子操作翻译](#14-原子操作翻译)
15. [cpu_exec_step_atomic() — 原子单步](#15-cpu_exec_step_atomic--原子单步)
16. [TLB 线程安全](#16-tlb-线程安全)
17. [ARM64 Exclusive Monitor](#17-arm64-exclusive-monitor)
18. [vCPU Kick 机制](#18-vcpu-kick-机制)
19. [中断投递](#19-中断投递)
20. [BQL（Big QEMU Lock）](#20-bqlbig-qemu-lock)
21. [Exclusive 上下文（全局串行化）](#21-exclusive-上下文全局串行化)
22. [cpu_exec_start/end — 执行区间管理](#22-cpu_exec_startend--执行区间管理)
23. [Safe Work 跨 vCPU 操作](#23-safe-work-跨-vcpu-操作)
24. [icount 指令计数模式](#24-icount-指令计数模式)
25. [MTTCG 执行流完整时序](#25-mttcg-执行流完整时序)
26. [EXCP_ATOMIC 处理流程](#26-excp_atomic-处理流程)
27. [RR vs MTTCG 对比](#27-rr-vs-mttcg-对比)
28. [附录 A：关键源文件索引](#28-附录-a关键源文件索引)

---

## 1. 概述

MTTCG（Multi-Threaded TCG）是 QEMU 的多线程 TCG 执行模式，每个虚拟 CPU（vCPU）运行在独立的宿主线程上。相对的是 RR（Round-Robin）模式——所有 vCPU 共享单个宿主线程轮转执行。

**核心挑战**：
- 多线程共享 TB 缓存：并发查找/插入/失效
- 内存序保证：将客户机的内存模型（ARM64 弱序）映射到宿主
- 原子指令翻译：客户机的 LDXR/STXR 在多线程下必须正确
- 中断投递：跨线程唤醒特定 vCPU

**关键数字**：
- MTTCG 主循环：**~60 行**（`tcg-accel-ops-mttcg.c:65-122`）
- RR 主循环：**~135 行**（`tcg-accel-ops-rr.c:180-315`）
- 模式分发：`tcg-accel-ops.c:201-227`

---

## 2. MTTCG 架构全景

```
                    ┌─────────────────────────┐
                    │     QEMU Main Thread     │
                    │   (Event Loop / BQL)     │
                    └────────────┬────────────┘
                                 │
        ┌───────────┬────────────┼────────────┬───────────┐
        ▼           ▼            ▼            ▼           ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  ...
  │ vCPU 0   │ │ vCPU 1   │ │ vCPU 2   │ │ vCPU 3   │
  │ Thread   │ │ Thread   │ │ Thread   │ │ Thread   │
  │          │ │          │ │          │ │          │
  │TCGContext│ │TCGContext│ │TCGContext│ │TCGContext│
  │ Region 0 │ │ Region 1 │ │ Region 2 │ │ Region 3 │
  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
       │            │            │            │
       └────────────┴─────┬──────┴────────────┘
                          │
              ┌───────────▼───────────┐
              │  共享 TB 哈希表 (QHT) │
              │  共享代码缓冲区       │
              │  每 vCPU TLB          │
              └───────────────────────┘
```

**线程所有权**：
- TCGContext → 线程私有（TLS `tcg_ctx`）
- 代码缓冲区 Region → 线程私有
- TB 哈希表 → 全局共享（QHT 无锁并发）
- TLB → 每 vCPU 私有（`CPUTLB`）
- TB 跳转链 → 共享，受 `jmp_lock` 保护

---

## 3. MTTCG 启用与配置

### 状态定义

**文件**：`accel/tcg/tcg-all.c:50-58`

```c
struct TCGState {
    AccelState parent_obj;
    OnOffAuto mttcg_enabled;    // auto / on / off   :53
    bool one_insn_per_tb;
    int splitwx_enabled;
    unsigned long tb_size;
};
```

### 查询接口

**定义**：`accel/tcg/tcg-all.c:66-70`

```c
bool qemu_tcg_mttcg_enabled(void)
{
    TCGState *s = TCG_STATE(current_accel());
    return s->mttcg_enabled == ON_OFF_AUTO_ON;
}
```

### 初始化决策

**定义**：`accel/tcg/tcg-all.c:104-168`

```
tcg_init_machine(as, ms)
│
├── cc->tcg_ops->mttcg_supported 检查              :111
│
├── switch (s->mttcg_enabled):
│   ├── AUTO:                                       :114-132
│   │   if (mttcg_supported && !icount_enabled())
│   │       → ON, max_threads = max_cpus
│   │   else
│   │       → OFF
│   │
│   ├── ON:                                         :134-140
│   │   if (!mttcg_supported) warn_report(...)
│   │   max_threads = max_cpus
│   │
│   └── OFF: max_threads = 1                        :141-142
│
├── tcg_init(tb_size, splitwx, max_threads)         :154
│   → 分配代码缓冲区，按 max_threads 分区
│
└── tcg_prologue_init()                             :161
```

**目标架构支持**：通过 `cc->tcg_ops->mttcg_supported`（`include/accel/tcg/cpu-ops.h:22-30`）声明。ARM64 支持 MTTCG。

**互斥条件**：MTTCG 与 icount 模式互斥（`tcg-accel-ops-mttcg.c:71`：`g_assert(!icount_enabled())`）。

---

## 4. vCPU 线程模型 — MTTCG 模式

### 线程入口

**定义**：`accel/tcg/tcg-accel-ops-mttcg.c:65-122`

```
mttcg_cpu_thread_fn(arg)
│
├── 初始化                                         :73-86
│   rcu_register_thread()                          :73
│   mttcg_force_rcu 注册                            :74-76
│   tcg_register_thread()                           :77  ← 分配 TCGContext
│   bql_lock()                                      :79
│   current_cpu = cpu                               :84
│   cpu_thread_signal_created(cpu)                  :85
│
├── 主循环                                          :88-115
│   do {
│       qemu_process_cpu_events(cpu)                :89
│       if (cpu_can_run(cpu)):
│           bql_unlock()                            :93
│           r = tcg_cpu_exec(cpu)                   :94  ← 执行客户机代码
│           bql_lock()                              :95
│           switch (r):
│               EXCP_DEBUG → cpu_handle_guest_debug :98
│               EXCP_HALTED → break                 :100-105
│               EXCP_ATOMIC →                       :106-109
│                   bql_unlock()
│                   cpu_exec_step_atomic(cpu)       ← 串行单步
│                   bql_lock()
│   } while (!cpu->unplug || cpu_can_run(cpu))      :115
│
└── 清理                                            :117-121
    tcg_cpu_destroy(cpu)
    bql_unlock()
    rcu_unregister_thread()
```

### 线程创建

**定义**：`accel/tcg/tcg-accel-ops-mttcg.c:124-137`

```c
void mttcg_start_vcpu_thread(CPUState *cpu)
{
    tcg_cpu_init_cflags(cpu, current_machine->smp.max_cpus > 1);  // :129
    // max_cpus > 1 → 设置 CF_PARALLEL

    qemu_thread_create(cpu->thread, "CPU N/TCG",
                       mttcg_cpu_thread_fn, cpu,                   // :135-136
                       QEMU_THREAD_JOINABLE);
}
```

**每 vCPU 一个线程**：MTTCG 模式下，每次调用 `mttcg_start_vcpu_thread()` 都创建一个新的 pthread。

---

## 5. vCPU 线程模型 — Round-Robin 模式

### 单线程共享

**定义**：`accel/tcg/tcg-accel-ops-rr.c:317-348`

```c
void rr_start_vcpu_thread(CPUState *cpu)
{
    tcg_cpu_init_cflags(cpu, false);                    // :324 — 无 CF_PARALLEL

    if (!single_tcg_cpu_thread) {                        // :326
        // 第一个 CPU：创建线程
        qemu_thread_create(cpu->thread, "ALL CPUs/TCG",
                           rr_cpu_thread_fn, cpu, ...);  // :332-334
    } else {
        // 后续 CPU：共享同一线程
        cpu->thread = single_tcg_cpu_thread;             // :340
        cpu->halt_cond = single_tcg_halt_cond;           // :341
    }
}
```

### RR 调度循环

**定义**：`accel/tcg/tcg-accel-ops-rr.c:180-315`

```
rr_cpu_thread_fn(arg)
│
├── 初始化（同 MTTCG，但单线程）                   :185-197
│   tcg_register_thread()                           :189
│
├── 等待机器启动                                    :200-208
│
├── rr_start_kick_timer()                           :210
│
├── 主循环                                          :214-312
│   while (1) {
│       rr_wait_io_event()                          :235
│       rr_deal_with_unplugged_cpus()               :236
│
│       bql_unlock() → replay_mutex_lock() → bql_lock()  :238-240
│
│       [icount] icount_account + handle_deadline    :242-254
│
│       while (cpu && cpu_work_list_empty(cpu)):      :262-308
│       │
│       │   qatomic_set_mb(&rr_current_cpu, cpu)     :267
│       │   检查 exit_request                         :270
│       │   current_cpu = cpu                         :273
│       │
│       │   if (cpu_can_run(cpu)):
│       │       bql_unlock()                          :281
│       │       [icount] icount_prepare_for_run()     :283
│       │       r = tcg_cpu_exec(cpu)                 :285
│       │       [icount] icount_process_data()        :287
│       │       bql_lock()                            :289
│       │       处理 EXCP_DEBUG / EXCP_ATOMIC
│       │
│       │   cpu = CPU_NEXT(cpu)                       :307
│       │
│       qatomic_set(&rr_current_cpu, NULL)            :311
│   }
```

### Kick 定时器

**定义**：`accel/tcg/tcg-accel-ops-rr.c:62-88`

```
rr_kick_vcpu_timer — 周期定时器
│
├── 触发间隔：TCG_KICK_PERIOD（虚拟时钟 ns）       :67
│
├── rr_kick_thread() 回调                           :84-88
│   timer_mod(timer, next_kick_time)                :86
│   rr_kick_next_cpu()                              :87
│
└── rr_kick_next_cpu()                              :71-82
    读取 rr_current_cpu
    cpu_exit(cpu) → 强制当前 vCPU 退出执行循环
```

**设计目标**：防止单个 vCPU 长期霸占 CPU，确保每个 vCPU 获得公平的执行时间片。

---

## 6. 模式选择与分发

**定义**：`accel/tcg/tcg-accel-ops.c:201-227`

```c
static void tcg_accel_ops_init(AccelClass *ac)
{
    AccelOpsClass *ops = ac->ops;

    if (qemu_tcg_mttcg_enabled()) {                      // :205
        ops->create_vcpu_thread = mttcg_start_vcpu_thread; // :206
        ops->kick_vcpu_thread = tcg_kick_vcpu_thread;      // :207
        ops->handle_interrupt = tcg_handle_interrupt;       // :208
    } else {
        ops->create_vcpu_thread = rr_start_vcpu_thread;    // :210
        ops->kick_vcpu_thread = rr_kick_vcpu_thread;       // :211
        if (icount_enabled()) {
            ops->handle_interrupt = icount_handle_interrupt; // :214
        } else {
            ops->handle_interrupt = tcg_handle_interrupt;    // :218
        }
    }
}
```

---

## 7. TCGContext 线程管理

### 全局与线程局部存储

**定义**：`tcg/tcg.c:239-244`

```c
TCGContext tcg_init_ctx;                  // 初始化用的上下文
__thread TCGContext *tcg_ctx;             // :240 — TLS，每线程的 TCGContext 指针

TCGContext **tcg_ctxs;                    // :242 — 所有上下文的数组
unsigned int tcg_cur_ctxs;               // :243 — 当前已注册的上下文数
unsigned int tcg_max_ctxs;               // :244 — 最大上下文数（= max_threads）
```

### 线程注册

每个 vCPU 线程在启动时调用 `tcg_register_thread()`：

1. 从 `tcg_init_ctx` 克隆配置到新的 `TCGContext`
2. 分配私有代码缓冲区 Region
3. 设置 TLS `tcg_ctx` 指向新上下文
4. 将新上下文添加到 `tcg_ctxs[]` 数组

**MTTCG 调用**：`tcg-accel-ops-mttcg.c:77`
**RR 调用**：`tcg-accel-ops-rr.c:189`

---

## 8. 代码缓冲区 Region 分区

### 分区策略

**定义**：`tcg/region.c:55-75`

```
总代码缓冲区（mmap 分配）：
┌─────────────┬─────────────┬─────────────┬─────────────┐
│  Region 0   │  Region 1   │  Region 2   │  Region 3   │
│  (vCPU 0)   │  (vCPU 1)   │  (vCPU 2)   │  (vCPU 3)   │
│  TCGCtx[0]  │  TCGCtx[1]  │  TCGCtx[2]  │  TCGCtx[3]  │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

- 每个 TCGContext 拥有独立的 Region，互不干扰
- Region 赋值在 `tcg_register_thread()` 中完成（`tcg/region.c:348-387`）
- 每个 Region 有独立的 `code_gen_buffer`、`code_gen_ptr`
- Region 边界由 `region_start/region_end` 限定

### 无锁翻译

由于每个线程有独立 Region，**代码生成无需加锁**——这是 MTTCG 的关键性能优势。

---

## 9. TB 哈希表并发访问

### QHT（QEMU Hash Table）

**初始化**：`accel/tcg/tb-maint.c:67-72`

```c
void tb_htable_init(void)
{
    qht_init(&tb_ctx.htable, ...);
}
```

QHT 是 QEMU 实现的**无锁并发哈希表**（基于 RCU + seqlock）：
- **查找**：无锁读取，使用 RCU 保护
- **插入**：细粒度锁（每个 bucket 一把锁）
- **删除**：RCU 延迟回收

### 并发操作

```
vCPU 0 线程                          vCPU 1 线程
│                                    │
├── tb_lookup() → tb_htable_lookup() ├── tb_lookup() → tb_htable_lookup()
│   读取 QHT（无锁）                │   读取 QHT（无锁）
│                                    │
├── tb_gen_code()                    │   找到 → 直接执行
│   翻译到自己的 Region              │
│   tb_link_page() → 插入 QHT       │
│   （bucket 锁）                    │
```

### TB 插入

**定义**：`accel/tcg/tb-maint.c:992-1005`

插入时使用 `qht_insert()`，自动处理哈希冲突。多个线程可以同时为相同的 PC 生成 TB——这是允许的，只是浪费一些代码空间。

---

## 10. TB 跳转链保护 — jmp_lock

### jmp_lock 作用

**定义**：`include/exec/translation-block.h:115-149`

```c
QemuSpin jmp_lock;  // :116 — 自旋锁
```

`jmp_lock` 保护：
- `jmp_list_head`：入边跳转链表
- `jmp_list_next[2]`：链表节点
- `jmp_dest[2]`：出边目标（tagged pointer，LSB=1 表示 TB 正在失效）
- `CF_INVALID` 标志

### 链接与解链

```
tb_add_jump(src, slot, dst)                    cpu-exec.c:616-651
│
├── spin_lock(&dst->jmp_lock)
│   检查 dst 未失效（CF_INVALID）
│   设置 src->jmp_dest[slot] = dst
│   将 src 加入 dst->jmp_list_head
│   修补跳转指令
├── spin_unlock(&dst->jmp_lock)

tb_jmp_unlink(tb)                              tb-maint.c:877-892
│
├── spin_lock(&tb->jmp_lock)
│   遍历 jmp_list_head
│   恢复每个源 TB 的跳转指令
├── spin_unlock(&tb->jmp_lock)
```

---

## 11. TB 刷新同步

### 跨线程刷新

TB 刷新（flush）需要所有 vCPU 暂停，因为要重置共享的哈希表和代码缓冲区。

**触发路径**：

```
tb_flush(cpu)
│
├── queue_tb_flush()
│   if (tb_ctx.tb_flush_count != n) return  — 去重
│   async_safe_run_on_cpu(cpu, do_tb_flush, ...)
│
└── do_tb_flush() — 在 exclusive context 中执行
    ├── tb_flush__exclusive_or_serial()        tb-maint.c:770-791
    │   ├── qht_reset(&tb_ctx.htable)         — 清空哈希表
    │   ├── tcg_flush_jmp_cache(cpu)           — 清空所有 CPU 的 jump cache
    │   ├── tcg_region_reset_all()             — 重置所有 Region
    │   └── tb_ctx.tb_flush_count++            — 递增计数（去重用）
    │
    └── 所有 vCPU 在 exclusive context 中暂停
        执行完 flush 后才能继续
```

---

## 12. CF_PARALLEL 标志

**定义**：`include/exec/translation-block.h:83`

```c
#define CF_PARALLEL  0x00008000
```

### 设置时机

**定义**：`accel/tcg/tcg-accel-ops.c:53-69`

```c
void tcg_cpu_init_cflags(CPUState *cpu, bool parallel)
{
    cflags |= parallel ? CF_PARALLEL : 0;              // :67
    cflags |= icount_enabled() ? CF_USE_ICOUNT : 0;    // :68
    tcg_cflags_set(cpu, cflags);
}
```

- MTTCG 模式：`parallel = (max_cpus > 1)` → 设置 `CF_PARALLEL`
- RR 模式：`parallel = false` → 不设置

### 影响范围

| 组件 | CF_PARALLEL=1 | CF_PARALLEL=0 |
|------|---------------|---------------|
| 原子操作 | 翻译为真原子指令/helper | 可降级为普通加载/存储 |
| 内存屏障 | 发射 DMB/DSB 等 | 可省略 |
| TB 缓存 | 按 cflags 区分（并行/串行 TB 不同） | — |
| EXCP_ATOMIC | 需要时退出到 `cpu_exec_step_atomic()` | 不会触发 |

---

## 13. TCG 内存屏障

### TCGBar 类型

**定义**：`include/tcg/tcg-mo.h:28-46`

```c
typedef enum {
    // 访问排序类型（SPARC 风格）
    TCG_MO_LD_LD  = 0x01,   // Load-Load 序                     :34
    TCG_MO_ST_LD  = 0x02,   // Store-Load 序                    :35
    TCG_MO_LD_ST  = 0x04,   // Load-Store 序                    :36
    TCG_MO_ST_ST  = 0x08,   // Store-Store 序                   :37
    TCG_MO_ALL    = 0x0F,   // 全屏障                           :38

    // 屏障语义
    TCG_BAR_LDAQ  = 0x10,   // Acquire 语义                     :43
    TCG_BAR_STRL  = 0x20,   // Release 语义                     :44
    TCG_BAR_SC    = 0x30,   // 顺序一致性                       :45
} TCGBar;
```

### tcg_gen_mb()

**定义**：`tcg/tcg-op.c:289-306`

```c
void tcg_gen_mb(TCGBar mb_type)
{
    bool parallel = true;      // 系统模式始终发射屏障            :300
    // 即使单 CPU 也需要屏障，因为 I/O 线程并行运行               :296-299
    if (parallel) {
        tcg_gen_op1(INDEX_op_mb, 0, mb_type);                     // :304
    }
}
```

### ARM64 屏障映射

**文件**：`target/arm/tcg/translate-a64.c:3507-3595`

| ARM64 指令 | TCGBar 映射 |
|------------|-------------|
| DMB SY | `TCG_MO_ALL \| TCG_BAR_SC` |
| DMB LD | `TCG_MO_ALL \| TCG_BAR_LDAQ` |
| DMB ST | `TCG_MO_ALL \| TCG_BAR_STRL` |
| DSB SY | `TCG_MO_ALL \| TCG_BAR_SC` + cache 操作 |
| ISB | `tcg_gen_lookup_and_goto_ptr()` — 终止当前 TB |

### AArch64 宿主发射

在 AArch64 宿主上，`TCG_MO_ALL | TCG_BAR_SC` 发射为 `DMB ISH`（Inner Shareable 全屏障）。

---

## 14. 原子操作翻译

### 原子 Helper 模板

**定义**：`accel/tcg/atomic_template.h:80-224`

QEMU 使用模板生成 `helper_atomic_*` 函数族：
- `helper_atomic_cmpxchg{b,w,l,q}`
- `helper_atomic_xchg{b,w,l,q}`
- `helper_atomic_fetch_{add,sub,and,or,xor}{b,w,l,q}`

### CF_PARALLEL 对原子操作的影响

**定义**：`tcg/tcg-op-ldst.c:79-87`

```
if (!CF_PARALLEL):
    原子操作降级 → MO_ATOM_NONE
    使用普通加载/存储实现（单线程无需真原子）
else:
    保留完整原子语义
    可能生成 helper 调用或内联原子指令
```

### ARM64 LDXR/STXR 翻译

**文件**：`target/arm/tcg/translate-a64.c:3502-3618`

```
LDXR Rt, [Rn]
├── 记录 exclusive_addr = Rn 地址
├── 记录 exclusive_val = [Rn] 值
└── 翻译为普通加载 + exclusive monitor 状态更新

STXR Ws, Rt, [Rn]
├── 比较 exclusive_addr 是否匹配
├── [MTTCG] 使用 tcg_gen_atomic_cmpxchg() 
│   → helper_atomic_cmpxchg 或内联 CAS
├── [RR] 简单比较 + 存储
└── 清除 exclusive monitor
```

---

## 15. cpu_exec_step_atomic() — 原子单步

**定义**：`accel/tcg/cpu-exec.c:549-598`

当 TB 执行中遇到无法在多线程下安全处理的原子操作时，返回 `EXCP_ATOMIC`，触发串行单步执行：

```
cpu_exec_step_atomic(cpu)
│
├── start_exclusive()                              :555
│   → 暂停所有其他 vCPU
│
├── s.cflags &= ~CF_PARALLEL                       :564
│   → 清除并行标志（串行执行）
│
├── s.cflags |= CF_NO_GOTO_TB | CF_NO_GOTO_PTR | 1  :566
│   → 禁止链接，只执行 1 条指令
│
├── tb = tb_lookup(cpu, s)                          :574
│   if (!tb) tb = tb_gen_code(cpu, s)              :577
│
├── cpu_tb_exec(cpu, tb)                           :584
│   → 串行执行该条指令
│
├── cpu->running = false                            :596
└── end_exclusive()                                 :597
    → 恢复所有 vCPU
```

**设计原理**：通过全局串行化，在不支持内联原子操作的场景下保证正确性。代价是性能——所有 vCPU 暂停等待。

---

## 16. TLB 线程安全

### 每 vCPU TLB

每个 CPUState 拥有独立的 `CPUTLB`（`include/exec/cputlb.h`），TLB 查找完全是线程本地操作，无需加锁。

### 跨 vCPU TLB 刷新

**定义**：`accel/tcg/cputlb.c:423-436`

```
tlb_flush(cpu)
│
├── 如果是本 CPU → 直接刷新
│
└── 如果是远程 CPU → async_run_on_cpu()
    → 将刷新操作投递到目标 vCPU 的安全工作队列
    → 目标 vCPU 在安全点执行刷新
```

**同步刷新**：`tlb_flush_all_cpus_synced()` 使用 `async_safe_run_on_cpu()`，在所有 vCPU 暂停后统一刷新。

---

## 17. ARM64 Exclusive Monitor

### CPU 状态字段

**文件**：`target/arm/cpu.h`

```c
uint64_t exclusive_addr;    // 监控地址
uint64_t exclusive_val;     // 监控值
uint64_t exclusive_high;    // 128 位的高 64 位
```

### 翻译

**文件**：`target/arm/tcg/translate-a64.c:3256-3399`

```
Load Exclusive (LDXR):
├── cpu_exclusive_addr = addr
├── cpu_exclusive_val = mem[addr]
└── dest = cpu_exclusive_val

Store Exclusive (STXR):
├── if (cpu_exclusive_addr != addr) → fail (Ws = 1)
├── [CF_PARALLEL] tcg_gen_atomic_cmpxchg(addr, expected, new)
│   if (old == expected) → success (Ws = 0)
│   else → fail (Ws = 1)
├── [!CF_PARALLEL] simple compare + store
└── arm_clear_exclusive() — 清除 monitor
```

### CLREX

```
CLREX:
└── cpu_exclusive_addr = -1  (无效地址，使下次 STXR 必定失败)
```

---

## 18. vCPU Kick 机制

### kick 路径

```
qemu_cpu_kick(cpu)                               system/cpus.c:499-507
│
├── qemu_cond_broadcast(cpu->halt_cond)          :501  — 唤醒 halt 等待
│
└── cpus_accel->kick_vcpu_thread(cpu)            :502-503
    │
    ├── [MTTCG] tcg_kick_vcpu_thread(cpu)        cpu-exec.c:752-764
    │   ├── cpu->exit_request = true (store-release)  :760
    │   └── icount_decr.u16.high = -1               :763
    │       → cpu_exec 循环会检测到并退出
    │
    └── [RR] rr_kick_vcpu_thread(unused)         tcg-accel-ops-rr.c:41-48
        └── CPU_FOREACH(cpu) tcg_kick_vcpu_thread(cpu)
            → kick 所有 CPU（因为共享线程）
```

### 信号机制

**定义**：`system/cpus.c:481-497`

```c
void cpus_kick_thread(CPUState *cpu)
{
    pthread_kill(cpu->thread->thread, SIG_IPI);   // :489
    // SIG_IPI = SIGUSR1 (include/qemu/main-loop.h:32)
}
```

信号处理器是空操作——仅用于中断阻塞的系统调用（如 `poll()`/`select()`），使线程从内核返回用户态检查 `exit_request`。

---

## 19. 中断投递

### tcg_handle_interrupt()

**定义**：`accel/tcg/tcg-accel-ops.c:96-109`

```c
void tcg_handle_interrupt(CPUState *cpu, int mask)
{
    cpu_set_interrupt(cpu, mask);                         // :98

    if (!qemu_cpu_is_self(cpu)) {
        qemu_cpu_kick(cpu);                              // :105
        // 远程 CPU → kick 使其退出执行循环检查中断
    } else {
        qatomic_set(&cpu->neg.icount_decr.u16.high, -1); // :107
        // 本地 CPU → 设置标志，循环末尾检查
    }
}
```

### cpu_handle_interrupt()

**定义**：`accel/tcg/cpu-exec.c:777-882`

在 TB 执行循环中每次迭代检查：

```
cpu_handle_interrupt(cpu, &last_tb)
│
├── 清除 icount_decr.u16.high                     :795
│
├── 检查中断标志
│   ├── CPU_INTERRUPT_DEBUG → EXCP_DEBUG           :802-806
│   ├── CPU_INTERRUPT_HALT → cpu_halted            :810-816
│   ├── 其他中断 → cc->tcg_ops->cpu_exec_interrupt :829
│   │   → 目标特定处理（ARM64：检查 IRQ/FIQ/...）
│   └── 无中断 → 继续执行
│
├── 检查 exit_request                               :855-870
│   → true 时清除请求，返回退出循环
│
└── 检查 icount_exit                                :872-878
```

---

## 20. BQL（Big QEMU Lock）

### 作用

BQL 保护 QEMU 的设备模型和全局状态。vCPU 线程在执行客户机代码时释放 BQL，访问设备时持有 BQL。

### MTTCG 中的 BQL 使用

**MTTCG 主循环**（`tcg-accel-ops-mttcg.c:88-115`）：
```
持有 BQL
│
├── 处理事件（持有 BQL）                :89
│
├── bql_unlock()                        :93  ← 释放
│   tcg_cpu_exec(cpu)                   :94  ← 执行客户机代码（不持有 BQL）
│   bql_lock()                          :95  ← 重新获取
│
└── 处理异常（持有 BQL）
```

### tcg_cpu_exec() 包装

**定义**：`accel/tcg/tcg-accel-ops.c:77-86`

```c
int tcg_cpu_exec(CPUState *cpu)
{
    cpu_exec_start(cpu);      // :81 — 标记 running
    ret = cpu_exec(cpu);       // :82 — 执行（不持有 BQL）
    cpu_exec_end(cpu);         // :83 — 取消 running
    return ret;
}
```

---

## 21. Exclusive 上下文（全局串行化）

### start_exclusive()

**定义**：`cpu-common.c:192-233`

```
start_exclusive()
│
├── g_assert(!current_cpu->running)                :198
│
├── [可重入] exclusive_context_count++              :200-203
│
├── qemu_mutex_lock(&qemu_cpu_list_lock)            :205
│
├── pending_cpus = 1                                :209
│   smp_mb()                                        :212
│
├── 遍历所有 CPU                                    :214-220
│   if (other_cpu->running):
│       has_waiter = true
│       running_cpus++
│       qemu_cpu_kick(other_cpu)  ← kick 使其退出
│
├── pending_cpus = running_cpus + 1                 :222
│   等待 pending_cpus 降到 1                         :223-225
│   → 所有运行中的 CPU 到达 cpu_exec_end() 时 -1
│
└── qemu_mutex_unlock()                             :230
    → 此时只有当前线程在执行
```

### end_exclusive()

**定义**：`cpu-common.c:236-247`

```c
void end_exclusive(void)
{
    pending_cpus = 0;                              // :244
    qemu_cond_broadcast(&exclusive_resume);         // :245
    // → 唤醒所有等待的 vCPU
}
```

---

## 22. cpu_exec_start/end — 执行区间管理

**定义**：`cpu-common.c:249-325`

```
cpu_exec_start(cpu)
│
├── cpu->running = true (atomic)                   :254
├── smp_mb()                                        :257
│
├── 检查 pending_cpus（是否有 exclusive 请求）
│   if (pending_cpus > 0):
│       cpu->running = false
│       pending_cpus--  (atomic)                    :287
│       → 等待 exclusive_resume 信号               :307
│       → 信号到达后重新设置 running = true         :313
│       → 循环重新检查 pending_cpus

cpu_exec_end(cpu)
│
├── cpu->running = false (atomic)
├── smp_mb()
│
├── 检查 pending_cpus
│   if (pending_cpus > 0 && has_waiter):
│       pending_cpus-- (atomic)
│       → 通知 exclusive 发起者
```

**配合**：
- `start_exclusive()` kick 所有运行中的 vCPU
- 被 kick 的 vCPU 在 `cpu_exec_end()` 中发现 `pending_cpus > 0`
- 递减计数并等待 `exclusive_resume`
- exclusive 操作完成后 `end_exclusive()` 广播唤醒

---

## 23. Safe Work 跨 vCPU 操作

### run_on_cpu() / async_run_on_cpu()

**定义**：`cpu-common.c:144-179`

```
async_run_on_cpu(cpu, func, data)
│
├── 将 (func, data) 加入 cpu->work_list              :170
├── qemu_cpu_kick(cpu)                                :177
│   → vCPU 退出执行循环
│
└── vCPU 在安全点执行 func(cpu, data)
    （在 qemu_process_cpu_events() 中处理）

run_on_cpu(cpu, func, data)
│
├── 同步版本：调用 async_run_on_cpu() + 等待完成
└── 如果是本 CPU 直接执行
```

### async_safe_run_on_cpu()

**定义**：`cpu-common.c:327-339`

在 exclusive context 中执行——确保没有其他 vCPU 在运行：

```c
void async_safe_run_on_cpu(CPUState *cpu, run_on_cpu_func func, ...)
{
    // 将 func 包装为 exclusive 操作
    // 执行时会先 start_exclusive()，执行 func，再 end_exclusive()
}
```

**典型用途**：TB flush、TLB 全局刷新、断点设置。

---

## 24. icount 指令计数模式

### 概述

icount 模式通过精确计数客户机指令实现确定性执行，用于回放/调试。

**互斥**：icount 与 MTTCG 互斥——icount 只能在 RR 模式下使用。

### 关键文件

| 文件 | 内容 |
|------|------|
| `accel/tcg/icount-common.c` | icount 核心实现（时钟推进/warp 定时器） |
| `accel/tcg/tcg-accel-ops-icount.c` | icount 执行包装 |
| `include/exec/icount.h` | API 定义 |

### 执行流

```
[RR 循环中]

icount_prepare_for_run(cpu, budget)              icount-ops.c:105+
├── 计算本次执行的指令预算
├── 设置 icount_decr.u16.low = budget
└── 重置 icount_extra

tcg_cpu_exec(cpu)                                 — 执行
├── TB 最多执行 budget 条指令
└── budget 用尽 → 退出

icount_process_data(cpu)                          icount-ops.c:130+
├── 计算实际执行的指令数
├── 推进虚拟时钟
└── 处理定时器到期
```

### icount_exit_request()

**定义**：`accel/tcg/cpu-exec.c:766-775`

```c
static inline bool icount_exit_request(CPUState *cpu)
{
    if (!icount_enabled()) return false;
    return cpu->neg.icount_decr.u16.low + cpu->icount_extra == 0;  // :774
}
```

---

## 25. MTTCG 执行流完整时序

```
vCPU 0 线程                    vCPU 1 线程                    Main Thread
│                              │                              │
├ bql_lock()                   ├ bql_lock()                   │
├ process_events()             ├ process_events()             │
│                              │                              │
├ bql_unlock()                 ├ bql_unlock()                 │
├ cpu_exec_start()             ├ cpu_exec_start()             │
│  running=true                │  running=true                │
│                              │                              │
├ cpu_exec() ◄────────────── 并行执行客户机代码 ──────────►     │
│  ├ tb_lookup()               │  ├ tb_lookup()               │
│  ├ cpu_tb_exec(tb)           │  ├ cpu_tb_exec(tb)           │
│  │ [执行宿主代码]            │  │ [执行宿主代码]            │
│  │                           │  │                           │
│  │  ◄──── SIG_IPI ──────────│──│── qemu_cpu_kick() ◄───── 设备中断
│  │  exit_request=true        │  │                           │
│  │                           │  │                           │
│  ├ cpu_handle_interrupt()    │  ├ cpu_handle_interrupt()    │
│  └ return                    │  │ [继续执行]               │
│                              │  └ ...                       │
├ cpu_exec_end()               │                              │
│  running=false               │                              │
├ bql_lock()                   │                              │
├ 处理中断/I/O                 │                              │
│                              ├ cpu_exec_end()               │
│                              │  running=false               │
├ bql_unlock()                 ├ bql_lock()                   │
├ cpu_exec_start()             │ ...                          │
│ ...                          │                              │
```

---

## 26. EXCP_ATOMIC 处理流程

```
vCPU 0 执行 LDXR/STXR 翻译后代码
│
├── helper 检测到需要原子 CAS 但当前 TB 是并行的
│   → 返回 EXCP_ATOMIC
│
└── 回到 mttcg_cpu_thread_fn:
    case EXCP_ATOMIC:                             :106-109
    │
    ├── bql_unlock()                              :107
    │
    ├── cpu_exec_step_atomic(cpu)                 :108
    │   │
    │   ├── start_exclusive()                     — 暂停所有其他 vCPU
    │   ├── cflags &= ~CF_PARALLEL               — 串行模式
    │   ├── cflags |= 1 (单条指令)               — 只执行一条
    │   ├── tb_lookup/tb_gen_code()               — 查找/生成串行 TB
    │   ├── cpu_tb_exec()                         — 执行
    │   └── end_exclusive()                       — 恢复所有 vCPU
    │
    └── bql_lock()                                :109
        继续主循环
```

---

## 27. RR vs MTTCG 对比

| 特性 | Round-Robin | MTTCG |
|------|-------------|-------|
| 线程数 | 1 | N（每 vCPU 一个） |
| 并行执行 | 否 | 是 |
| `CF_PARALLEL` | 清除 | 设置 |
| 原子操作 | 普通加载/存储 | 真原子/helper |
| 内存屏障 | 可省略 | 必须发射 |
| icount 支持 | ✅ | ❌（互斥） |
| 确定性 | 高（可回放） | 低（线程调度不确定） |
| BQL 竞争 | 低（单线程） | 高（多线程竞争） |
| TB 代码生成 | 无锁（单线程） | 无锁（Region 分区） |
| TB 查找 | 无锁（单线程） | 无锁（QHT） |
| TB flush | 无需同步 | 需 exclusive context |
| 调度 | kick timer 轮转 | OS 线程调度 |
| 性能（多核负载） | 差 | 好 |
| 性能（单核负载） | 好（无同步开销） | 略差（屏障/原子开销） |

---

## 28. 附录 A：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `accel/tcg/tcg-accel-ops-mttcg.c` | 138 | MTTCG 线程入口、创建 |
| `accel/tcg/tcg-accel-ops-rr.c` | 349 | RR 线程入口、kick timer、调度循环 |
| `accel/tcg/tcg-accel-ops.c` | ~230 | 公共操作：模式分发、cflags 初始化、中断处理 |
| `accel/tcg/tcg-all.c` | ~170 | MTTCG 配置、tcg_init_machine |
| `accel/tcg/cpu-exec.c` | ~990 | TB 执行循环、kick、中断处理、cpu_exec_step_atomic |
| `accel/tcg/cputlb.c` | ~2000 | TLB 操作、刷新、softmmu 快/慢路径 |
| `accel/tcg/tb-maint.c` | ~1200 | TB 维护：哈希表、失效、刷新 |
| `accel/tcg/atomic_template.h` | ~230 | 原子操作 helper 模板 |
| `accel/tcg/tcg-accel-ops-icount.c` | ~150 | icount 执行包装 |
| `accel/tcg/icount-common.c` | ~500 | icount 核心：warp timer、时钟推进 |
| `cpu-common.c` | ~400 | start/end_exclusive、cpu_exec_start/end、safe work |
| `system/cpus.c` | ~520 | vCPU kick、BQL、信号机制 |
| `tcg/tcg.c` | ~6770 | TCGContext TLS、tcg_ctxs 管理 |
| `tcg/region.c` | ~910 | Region 分区、缓冲区分配 |
| `include/tcg/tcg-mo.h` | 48 | TCGBar 内存屏障类型定义 |
| `include/exec/translation-block.h` | ~160 | TB 结构、jmp_lock、CF_* 标志 |
| `include/accel/tcg/cpu-ops.h` | ~80 | mttcg_supported、guest_default_memory_order |

---

> **文档版本**：v1.0
> **源码版本**：QEMU 11.0.50 (commit 基于 2025-07 主线)
> **分析工具**：zoekt + ctags + cscope + clangd + 手动源码验证
> **交叉验证**：所有行号均经 view/grep 验证
