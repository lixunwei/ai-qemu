# QEMU 模拟执行循环与 MMIO 分发深度分析

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 关键 commit：`9f733abb1a`（MemoryRegionOps 对齐断言）、`a4e85f89e8`（FlatView 优化）  
> 参考文档：`docs/devel/multiple-iothreads.txt`、`docs/devel/main-loop.txt`  

---

## 目录

- [第一部分：全局执行架构](#第一部分全局执行架构)
  - [1. 从 main() 到主循环](#1-从-main-到主循环)
  - [2. 主循环总览图](#2-主循环总览图)
- [第二部分：主事件循环（Main Loop）](#第二部分主事件循环main-loop)
  - [3. qemu_main_loop() — 大循环](#3-qemu_main_loop--大循环)
  - [4. main_loop_wait() — 单次迭代](#4-main_loop_wait--单次迭代)
  - [5. AioContext 与事件分发](#5-aiocontext-与事件分发)
  - [6. Bottom Half（BH）机制](#6-bottom-halfbh机制)
  - [7. 定时器基础设施](#7-定时器基础设施)
  - [8. IOThread 独立事件循环](#8-iothread-独立事件循环)
- [第三部分：CPU 执行循环](#第三部分cpu-执行循环)
  - [9. vCPU 线程模型](#9-vcpu-线程模型)
  - [10. cpu_exec() — 内层执行循环](#10-cpu_exec--内层执行循环)
  - [11. TB 查找与执行](#11-tb-查找与执行)
  - [12. 中断与异常处理](#12-中断与异常处理)
  - [13. TCG 与主循环交互](#13-tcg-与主循环交互)
  - [14. KVM 执行循环（对比）](#14-kvm-执行循环对比)
- [第四部分：MMIO 分发流程](#第四部分mmio-分发流程)
  - [15. TCG MMIO 路径](#15-tcg-mmio-路径)
  - [16. KVM MMIO 路径](#16-kvm-mmio-路径)
  - [17. 内存分发核心](#17-内存分发核心)
  - [18. FlatView 与地址空间查找](#18-flatview-与地址空间查找)
- [第五部分：设备 I/O 与完成回调](#第五部分设备-io-与完成回调)
  - [19. 设备 Handler 执行（virtio-blk 示例）](#19-设备-handler-执行virtio-blk-示例)
  - [20. 异步 I/O 完成路径](#20-异步-io-完成路径)
  - [21. 中断注入与 CPU 唤醒](#21-中断注入与-cpu-唤醒)
  - [22. BQL 与线程安全](#22-bql-与线程安全)
- [第六部分：端到端完整流程](#第六部分端到端完整流程)
  - [23. 完整 I/O 请求生命周期](#23-完整-io-请求生命周期)
  - [24. 时序分析：一次 virtio-blk 读请求](#24-时序分析一次-virtio-blk-读请求)
- [附录](#附录)
  - [A. 关键源文件索引](#a-关键源文件索引)
  - [B. CPU_INTERRUPT 标志表](#b-cpu_interrupt-标志表)
  - [C. EXCP 异常码表](#c-excp-异常码表)

---

# 第一部分：全局执行架构

## 1. 从 main() 到主循环

### 1.1 启动序列

```
main()                                      [system/main.c:69-96]
  │
  ├── qemu_init(argc, argv)                 [system/vl.c]
  │   ├── 解析命令行参数
  │   ├── 初始化加速器（TCG/KVM）
  │   ├── 创建机器（machvirt_init）
  │   ├── 创建 CPU、内存、设备
  │   └── realize 所有设备
  │
  └── qemu_default_main()                   [system/vl.c:44-55]
      ├── bql_lock()                        // 获取 Big QEMU Lock
      ├── status = qemu_main_loop()         // 进入大循环
      ├── qemu_cleanup()                    // 清理
      └── exit(status)
```

### 1.2 macOS 特殊处理

`system/vl.c:87-95`：macOS 上 `qemu_main` 在独立线程运行（Cocoa UI 需要主线程）。

## 2. 主循环总览图

```
┌──────────────────────────────────────────────────────────────────────┐
│                    QEMU 进程                                         │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              主线程（Main Thread）                            │    │
│  │                                                              │    │
│  │   qemu_main_loop()        [runstate.c:943]                   │    │
│  │     └── while (running) {                                    │    │
│  │           main_loop_wait(false)    [main-loop.c:563]         │    │
│  │           ├── 1. pre-poll 通知                               │    │
│  │           ├── 2. 计算超时（定时器截止时间）                    │    │
│  │           ├── 3. os_host_main_loop_wait()                    │    │
│  │           │     └── GLib: prepare → query → poll → check     │    │
│  │           │              → dispatch                          │    │
│  │           ├── 4. post-poll 通知                              │    │
│  │           └── 5. qemu_clock_run_all_timers()                 │    │
│  │         }                                                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     ┌──────────────┐    │
│  │ vCPU 0   │  │ vCPU 1   │  │ vCPU N   │     │ IOThread     │    │
│  │          │  │          │  │          │     │              │    │
│  │cpu_exec()│  │cpu_exec()│  │cpu_exec()│     │ aio_poll()   │    │
│  │ TB 执行  │  │ TB 执行  │  │ TB 执行  │     │ 独立事件循环  │    │
│  │ MMIO 分发│  │ MMIO 分发│  │ MMIO 分发│     │              │    │
│  └──────────┘  └──────────┘  └──────────┘     └──────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  qemu_aio_context（共享 AIO 上下文）                          │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐               │   │
│  │  │BH 列表 │ │FD 处理 │ │定时器  │ │协程队列│               │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

# 第二部分：主事件循环（Main Loop）

## 3. qemu_main_loop() — 大循环

`system/runstate.c:943-948`：

```c
static int qemu_main_loop(void) {
    while (!main_loop_should_exit(&status)) {
        main_loop_wait(false);              // 单次迭代
    }
    return status;
}
```

`main_loop_should_exit()` 检查以下退出条件：
- `qemu_debug_requested()`
- `qemu_suspend_requested()`
- `qemu_shutdown_requested()`
- `qemu_reset_requested()`
- `qemu_wakeup_requested()`
- `qemu_powerdown_requested()`

## 4. main_loop_wait() — 单次迭代

`util/main-loop.c:563-604`：

```c
void main_loop_wait(bool nonblocking) {
    // 1. 通知 pre-poll 监听者
    notifier_list_notify(&main_loop_poll_notifiers, &mlpoll);

    // 2. 计算超时
    //    — 取定时器最近截止时间
    //    — nonblocking 模式超时为 0
    timeout_ns = timerlistgroup_deadline_ns(&main_loop_tlg);
    if (nonblocking) timeout_ns = 0;

    // 3. 执行宿主事件循环
    os_host_main_loop_wait(timeout_ns);
    //   内部使用 GLib:
    //   g_main_context_prepare()
    //   g_main_context_query()     → 获取要监控的 fd 列表
    //   qemu_poll_ns(fds, timeout) → poll/epoll 等待
    //   g_main_context_check()
    //   g_main_context_dispatch()  → 分发就绪事件

    // 4. 通知 post-poll 监听者
    notifier_list_notify(&main_loop_poll_notifiers, &mlpoll);

    // 5. 执行到期的定时器
    qemu_clock_run_all_timers();
}
```

### 4.1 GLib 集成

`util/main-loop.c:257-317`：

QEMU 将自己的 AIO 事件源注册到 GLib 主上下文中。GLib 的 `prepare/query/check/dispatch` 模型与 QEMU 的 `aio_poll` 集成：

```
GLib MainContext
  ├── QEMU AIO GSource（来自 qemu_aio_context）
  │     — fd 处理器
  │     — BH
  │     — 定时器
  ├── 网络 Socket GSource
  ├── 监控 Socket GSource
  └── 其他 GLib 事件源
```

## 5. AioContext 与事件分发

### 5.1 AioContext 结构

`include/qemu/aio.h:201-339`：

```c
struct AioContext {
    // GLib 事件源（用于集成）
    GSource source;

    // BH 列表
    QSLIST_HEAD(, QEMUBH) bh_list;

    // 定时器列表组
    QEMUTimerListGroup tlg;

    // fd 监控
    AioHandler *aio_handlers;       // fd 处理器链表

    // 通知机制
    EventNotifier notifier;         // 用于唤醒 poll

    // 线程池（用于文件 I/O）
    struct ThreadPool *thread_pool;

    // Linux AIO
    struct LinuxAioState *linux_aio;
    struct LuringState *linux_io_uring;
};
```

### 5.2 aio_poll() 核心流程

`util/aio-posix.c:652-758`：

```c
bool aio_poll(AioContext *ctx, bool blocking) {
    // 1. 执行 BH（优先级最高）
    progress = aio_bh_poll(ctx);

    // 2. 计算超时
    timeout = blocking ? aio_compute_timeout(ctx) : 0;

    // 3. 可选：busy-poll 模式（减少延迟）
    if (poll_mode) {
        // 短时间自旋检查 fd 状态
    }

    // 4. 等待事件（poll/epoll）
    fdmon_wait(ctx, timeout);

    // 5. 接受通知
    aio_notify_accept(ctx);

    // 6. 分发就绪的 fd 处理器
    dispatch_ready_handlers();

    // 7. 再次执行 BH（可能在 fd 处理中新调度了 BH）
    progress |= aio_bh_poll(ctx);

    // 8. 执行到期定时器
    progress |= timerlistgroup_run_timers(&ctx->tlg);

    return progress;
}
```

### 5.3 两个 AioContext

`util/main-loop.c:135-187, 616-649`：

```
qemu_aio_context  — 主 AIO 上下文
  — 附加到 GLib 默认上下文
  — 处理块设备 I/O、网络、协程

iohandler_ctx     — IO 处理器上下文
  — 独立于 qemu_aio_context
  — 处理"不能被 aio_poll(qemu_aio_context) 轮询"的 fd
  — 例如：信号 fd、一些监控 fd
```

## 6. Bottom Half（BH）机制

### 6.1 调度

`util/async.c:74-108, 235-238`：

```c
void qemu_bh_schedule(QEMUBH *bh) {
    aio_bh_enqueue(bh, BH_SCHEDULED);
    // 1. 设置 BH_PENDING | BH_SCHEDULED 标志
    // 2. 插入 AioContext 的 BH 列表
    // 3. aio_notify(ctx) — 唤醒正在 poll 的线程
}
```

### 6.2 执行

`util/async.c:181-227`：

```c
int aio_bh_poll(AioContext *ctx) {
    // 遍历 BH 列表
    // 对每个 BH_SCHEDULED 的 BH:
    //   清除 SCHEDULED 标志
    //   调用 bh->cb(bh->opaque)   — 执行回调
    // 返回处理数量
}
```

### 6.3 BH 执行时机

```
BH 在以下时机执行:
  1. aio_poll() 中 — 在 fd 分发前和后各执行一次
  2. main_loop_wait() 中 — 通过 GLib dispatch
  3. 协程调度中 — qemu_aio_coroutine_enter()

优先级顺序（aio_poll 内）:
  BH → fd 就绪处理 → BH（再次） → 定时器
```

### 6.4 BH vs 定时器 vs fd 处理器

| 机制 | 延迟 | 适用场景 |
|------|------|----------|
| BH | 最低（下次 poll 立即执行） | I/O 完成回调、状态通知 |
| fd 处理器 | 取决于 poll 超时 | 网络/文件事件驱动 |
| 定时器 | 精确到 ns（取决于时钟） | 延迟操作、超时、模拟时钟 |

## 7. 定时器基础设施

### 7.1 时钟类型

| 时钟 | 语义 | 用途 |
|------|------|------|
| `QEMU_CLOCK_REALTIME` | 宿主真实时间 | 网络超时、迁移截止 |
| `QEMU_CLOCK_VIRTUAL` | 虚拟机时间（可暂停） | Guest 定时器、icount |
| `QEMU_CLOCK_HOST` | 宿主单调时钟 | 性能计量 |
| `QEMU_CLOCK_VIRTUAL_RT` | 虚拟+真实混合 | 输入事件时间戳 |

### 7.2 定时器执行

`util/qemu-timer.c:518-647`：

```c
// 在 main_loop_wait() 结束时调用
qemu_clock_run_all_timers()
  → timerlistgroup_run_timers(&main_loop_tlg)
    → timerlist_run_timers(tl)
      → 遍历定时器列表
      → 到期: cb(opaque)
      → 未到期: 计算下次截止时间

// 超时计算用于 poll 等待
timerlistgroup_deadline_ns(&main_loop_tlg)
  → 返回最近到期定时器的时间差
```

## 8. IOThread 独立事件循环

### 8.1 IOThread 架构

`iothread.c:28-65, 174-214`：

```c
// 每个 IOThread 拥有独立的事件循环
static void *iothread_run(void *opaque) {
    IOThread *iothread = opaque;

    // 设置当前线程的 AIO 上下文
    qemu_set_current_aio_context(iothread->ctx);

    while (!iothread->stopping) {
        aio_poll(iothread->ctx, true);     // 独立 poll
        // 或者
        g_main_loop_run(iothread->main_loop);
    }
}

// 初始化
iothread_init() {
    iothread->ctx = aio_context_new(...);  // 独立 AIO 上下文
    // 创建 GLib 主循环
    iothread->main_loop = g_main_loop_new(g_main_ctx, true);
    // 启动线程
    qemu_thread_create(&iothread->thread, "iothread", iothread_run, ...);
}
```

### 8.2 IOThread 与主线程的关系

```
主线程                          IOThread
  │                               │
  │ qemu_aio_context              │ iothread->ctx
  │ main_loop_wait()              │ aio_poll()
  │ GLib 默认上下文               │ 独立 GLib 上下文
  │                               │
  │ 处理: 监控、管理、UI          │ 处理: 块设备 I/O
  │ BQL 持有                      │ 无需 BQL
  │                               │
  └───── 通过 BH/通知跨线程通信 ──┘
```

**IOThread 的优势**：块设备 I/O 可以不持有 BQL 执行，减少主线程竞争。

---

# 第三部分：CPU 执行循环

## 9. vCPU 线程模型

### 9.1 MTTCG（多线程 TCG）

`accel/tcg/tcg-accel-ops-mttcg.c:65-136`：

```c
// 每个 vCPU 一个宿主线程
static void *mttcg_cpu_thread_fn(void *arg) {
    CPUState *cpu = arg;

    // 1. 线程初始化
    tcg_cpu_init_cflags(cpu, current_machine->smp.max_cpus > 1);
    rcu_register_thread();

    // 2. 获取 BQL
    bql_lock();

    // 3. 主循环
    do {
        if (cpu_can_run(cpu)) {
            // 释放 BQL → 执行 → 重新获取 BQL
            bql_unlock();
            error = tcg_cpus_exec(cpu);     // → cpu_exec()
            bql_lock();
        }
        // 处理待处理事件（暂停、复位等）
        qemu_process_cpu_events(cpu);
    } while (!cpu->unplug || cpu_can_run(cpu));

    return NULL;
}
```

### 9.2 Round-Robin（单线程 TCG）

`accel/tcg/tcg-accel-ops-rr.c:180-315`：

```c
// 所有 vCPU 共享一个宿主线程，轮转执行
static void *rr_cpu_thread_fn(void *arg) {
    bql_lock();

    while (1) {
        // 遍历所有 CPU
        CPU_FOREACH(cpu) {
            if (cpu_can_run(cpu)) {
                bql_unlock();
                error = tcg_cpus_exec(cpu);
                bql_lock();
            }
            qemu_process_cpu_events(cpu);
        }

        // 所有 CPU 空闲时等待
        if (all_cpu_threads_idle()) {
            qemu_cond_wait(first_cpu->halt_cond, &bql);
        }
    }
}
```

### 9.3 MTTCG vs RR 对比

| 特性 | MTTCG | Round-Robin |
|------|-------|-------------|
| 线程模型 | N 个宿主线程（N=vCPU数） | 1 个宿主线程 |
| 并行度 | 真正并行执行 | 串行轮转 |
| 内存序 | 需要原子操作 | 天然串行安全 |
| 适用场景 | 多核 Guest（默认） | 调试、确定性重放 |

## 10. cpu_exec() — 内层执行循环

### 10.1 入口

`accel/tcg/cpu-exec.c:1019-1045`：

```c
int cpu_exec(CPUState *cpu) {
    // 1. 快速路径：CPU 处于 halt 状态
    if (cpu_handle_halt(cpu)) {           // [:653-669]
        return EXCP_HALTED;
    }

    // 2. 设置 longjmp 点（用于异常退出）
    ret = cpu_exec_setjmp(cpu, &sc);      // [:1009-1017]
    // setjmp 返回后进入主循环

    // 3. 清理并返回
    return ret;
}
```

### 10.2 cpu_exec_loop() — 核心循环

`accel/tcg/cpu-exec.c:932-1007`：

```c
static int cpu_exec_loop(CPUState *cpu, SyncClocks *sc) {
    while (true) {
        // ① 处理异常
        while (!cpu_handle_exception(cpu, &ret)) {
            // 有异常需要报告 → 退出
            return ret;
        }

        // ② 处理中断
        while (!cpu_handle_interrupt(cpu, &ret)) {
            // 有中断/退出请求需要处理 → 退出
            return ret;
        }

        // ③ TB 查找
        tb = tb_lookup(cpu, pc, cs_base, flags, cflags);
        if (tb == NULL) {
            // 缓存未命中 → 翻译新 TB
            tb = tb_gen_code(cpu, pc, cs_base, flags, cflags);
            // 插入 QHT 和 jmp_cache
        }

        // ④ 执行 TB
        cpu_loop_exec_tb(cpu, tb, pc, &last_tb, &tb_exit);
        // 内部调用 cpu_tb_exec() → tcg_qemu_tb_exec()

        // ⑤ TB 链接（用于加速连续 TB 执行）
        if (last_tb && !qemu_loglevel_mask(CPU_LOG_TB_NOCHAIN)) {
            tb_add_jump(last_tb, tb_exit, tb);
        }

        // 循环回到 ①
    }
}
```

### 10.3 循环流程图

```
                ┌──────────────────┐
                │  cpu_exec_loop() │
                └────────┬─────────┘
                         │
                ┌────────▼─────────┐
        ┌──────│ 有异常需处理？     │
        │ 是   │cpu_handle_exception│
        │      └────────┬──────────┘
        │               │ 否（继续）
        │      ┌────────▼──────────┐
        │ ┌────│ 有中断/退出请求？  │
        │ │ 是 │cpu_handle_interrupt│
        │ │    └────────┬──────────┘
        │ │             │ 否（继续）
        │ │    ┌────────▼──────────┐
        │ │    │  TB 查找           │
        │ │    │  tb_lookup()       │
        │ │    │  ↓ 未命中          │
        │ │    │  tb_gen_code()     │
        │ │    └────────┬──────────┘
        │ │             │
        │ │    ┌────────▼──────────┐
        │ │    │  执行 TB           │
        │ │    │  cpu_tb_exec()     │
        │ │    │  → 生成代码执行    │
        │ │    └────────┬──────────┘
        │ │             │
        │ │    ┌────────▼──────────┐
        │ │    │  TB 链接           │
        │ │    │  tb_add_jump()     │
        │ │    └────────┬──────────┘
        │ │             │
        │ │             └──→ 回到循环顶部
        │ │
        │ └──→ 退出 cpu_exec_loop()
        └────→ 退出 cpu_exec_loop()
```

## 11. TB 查找与执行

### 11.1 两级查找

`tb_lookup()`（`cpu-exec.c:227-263`）：

```c
static TranslationBlock *tb_lookup(CPUState *cpu, vaddr pc, ...) {
    // 第一级：per-CPU 跳转缓存（O(1)）
    hash = tb_jmp_cache_hash_func(pc);
    tb = cpu->tb_jmp_cache[hash];
    if (tb && tb->pc == pc && tb->flags == flags) {
        return tb;  // 命中！
    }

    // 第二级：全局哈希表 QHT（O(1)~O(n)）
    tb = tb_htable_lookup(cpu, pc, cs_base, flags, cflags);
    if (tb) {
        cpu->tb_jmp_cache[hash] = tb;  // 填充跳转缓存
        return tb;
    }

    return NULL;  // 未命中，需要翻译
}
```

### 11.2 TB 执行

`cpu_tb_exec()`（`cpu-exec.c:427-491`）：

```c
static inline TranslationBlock *cpu_tb_exec(CPUState *cpu, TranslationBlock *itb, ...) {
    // 调用生成的宿主代码
    ret = tcg_qemu_tb_exec(env, tb->tc.ptr);     // [:439]
    // tcg_qemu_tb_exec 是一个函数指针，直接调用生成代码

    // 解析返回值
    last_tb = (TranslationBlock *)(ret & ~TB_EXIT_MASK);
    tb_exit = ret & TB_EXIT_MASK;
    // TB_EXIT_IDX0/1: 正常退出（跳转到下一个 TB）
    // TB_EXIT_REQUESTED: 退出请求（中断、异常等）

    return last_tb;
}
```

### 11.3 TB 链接

`tb_add_jump()`（`cpu-exec.c:617-651`）：

```
TB A 执行完毕 → 退出码 = TB_EXIT_IDX0
  → tb_add_jump(A, 0, B)
  → 修补 A 的直接跳转指令指向 B 的生成代码
  → 下次 A 执行到末尾直接跳转到 B（不返回 cpu_exec_loop）
```

这是 TB 链接优化的核心：避免每个 TB 执行完都回到解释器循环。

## 12. 中断与异常处理

### 12.1 cpu_handle_exception()

`cpu-exec.c:687-750`：

```c
static bool cpu_handle_exception(CPUState *cpu, int *ret) {
    if (cpu->exception_index < 0) {
        return true;  // 无异常，继续执行
    }

    // 特殊异常类型
    if (cpu->exception_index == EXCP_INTERRUPT) {
        *ret = 0;  // 外部中断唤醒
        return false;
    }
    if (cpu->exception_index == EXCP_DEBUG) {
        *ret = EXCP_DEBUG;
        return false;
    }
    if (cpu->exception_index == EXCP_HALTED) {
        *ret = EXCP_HALTED;
        return false;
    }
    if (cpu->exception_index == EXCP_ATOMIC) {
        // 需要序列化执行的原子操作
        cpu_exec_step_atomic(cpu);
        return true;  // 继续执行
    }

    // Guest 异常（同步异常、SVC、页故障等）
    // → 调用目标架构的 do_interrupt()
    cc->tcg_ops->do_interrupt(cpu);       // [:724-741]
    // 修改 PC 到异常向量，继续执行
    return true;
}
```

### 12.2 cpu_handle_interrupt()

`cpu-exec.c:777-881`：

```c
static bool cpu_handle_interrupt(CPUState *cpu, int *ret) {
    // 检查退出请求
    if (cpu->exit_request) {
        *ret = EXCP_INTERRUPT;
        return false;
    }

    // 检查中断标志
    if (cpu->interrupt_request & CPU_INTERRUPT_HALT) {
        cpu->halted = 1;
        *ret = EXCP_HALTED;
        return false;
    }
    if (cpu->interrupt_request & CPU_INTERRUPT_RESET) {
        cpu_reset(cpu);
        return true;
    }

    // 目标架构特定中断处理
    // → 检查 IRQ/FIQ 是否可以被接受
    if (cc->tcg_ops->cpu_exec_interrupt(cpu, interrupt_request)) {
        // 中断被接受 → do_interrupt 已修改 PC
        return true;  // 继续执行（在中断处理程序中）
    }

    // icount 检查
    if (icount_exit_request()) {
        *ret = EXCP_INTERRUPT;
        return false;
    }

    return true;  // 无需处理，继续
}
```

### 12.3 异常退出机制（longjmp）

```c
// 设备/helper 中触发异常退出
cpu_loop_exit(cpu)                        // [cpu-exec-common.c:67-82]
  → cpu->exception_index = EXCP_...
  → siglongjmp(cpu->jmp_env, 1)         // 跳回 cpu_exec_setjmp
  → cpu_exec_loop() 重新进入异常处理

cpu_loop_exit_restore(cpu, retaddr)
  → 恢复 guest 状态到 TB 入口点
  → cpu_loop_exit()
```

**使用场景**：MMIO 访问时的 `cpu_loop_exit()`、TLB 填充失败、不支持的指令等。

## 13. TCG 与主循环交互

### 13.1 BQL 协作

```
MTTCG 模式:
  vCPU 线程持有 BQL
    → 释放 BQL
    → cpu_exec()（执行 guest 代码）
    → MMIO 时重新获取 BQL
    → MMIO 完成后释放 BQL
    → 继续 cpu_exec()
    → 退出 cpu_exec()
    → 重新获取 BQL
    → 处理事件
```

### 13.2 CPU 唤醒

`system/cpus.c:499-507`：

```c
void qemu_cpu_kick(CPUState *cpu) {
    qemu_cond_broadcast(cpu->halt_cond);   // 唤醒 halt 等待
    if (cpus_accel->kick_vcpu_thread) {
        cpus_accel->kick_vcpu_thread(cpu);  // 加速器特定唤醒
    }
}
```

TCG 踢（`cpu-exec.c:752-764`）：
```c
// 设置 exit_request，使 cpu_exec_loop 在下次检查时退出
qatomic_set(&cpu->exit_request, 1);
// 如果 CPU 在执行生成代码，发送信号中断
```

### 13.3 空闲检测

`system/cpus.c:87-102, 331-340`：

```c
bool cpu_thread_is_idle(CPUState *cpu) {
    // halted 且无 exit_request 且无待处理工作 → 空闲
    return cpu->halted && !cpu->exit_request && !...;
}

bool cpu_can_run(CPUState *cpu) {
    // 非停止、非未实现 → 可运行
    return !cpu->stop && !cpu->stopped;
}
```

## 14. KVM 执行循环（对比）

`accel/kvm/kvm-all.c:3439-3565`：

```c
int kvm_cpu_exec(CPUState *cpu) {
    do {
        // 1. 处理异步事件（中断注入等）
        kvm_arch_pre_run(cpu, run);         // [:3446-3448]

        // 2. 进入内核执行
        //    → 释放 BQL
        //    → KVM_RUN ioctl → 在内核态执行 Guest
        //    → 退出内核态
        //    → 重新获取 BQL
        ret = kvm_vcpu_ioctl(cpu, KVM_RUN, 0);   // [:3476]

        // 3. 处理退出原因
        switch (run->exit_reason) {
        case KVM_EXIT_IO:                   // [:3522-3529]
            kvm_handle_io(run->io...);
            break;

        case KVM_EXIT_MMIO:                 // [:3531-3538]
            // ★ MMIO 退出 → QEMU 模拟设备
            address_space_rw(&address_space_memory,
                run->mmio.phys_addr, attrs,
                run->mmio.data, run->mmio.len,
                run->mmio.is_write);
            break;

        case KVM_EXIT_IRQ_WINDOW_OPEN:      // [:3540]
            // 中断窗口打开
            break;

        case KVM_EXIT_SHUTDOWN:             // [:3554]
            qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
            break;
        }
    } while (ret == 0);

    return ret;
}
```

### 14.1 TCG vs KVM MMIO 处理对比

| 方面 | TCG | KVM |
|------|-----|-----|
| MMIO 检测 | 软件 TLB 标记 `TLB_MMIO` | 硬件 EPT/Stage2 故障 |
| 退出方式 | `cpu_loop_exit()` (longjmp) | `KVM_EXIT_MMIO` (ioctl 返回) |
| BQL | 在慢速路径中获取 | KVM_RUN 前已持有 |
| 调用 | `memory_region_dispatch_*()` | `address_space_rw()` |
| 回路 | `cpu_exec_loop()` 继续 | `KVM_RUN` ioctl 重入 |

---

# 第四部分：MMIO 分发流程

## 15. TCG MMIO 路径

### 15.1 TLB 与 MMIO 标记

Guest 内存访问在 TCG 中经过软件 TLB：

```
生成代码执行内存访问（LD/ST 指令）
  │
  ├── 内联 TLB 快速路径
  │   ├── TLB 命中 + 普通内存 → 直接访问宿主内存（最快）
  │   └── TLB 命中 + MMIO 标记 → 进入慢速路径
  │
  └── TLB 未命中 → tlb_fill → 页表遍历
      ├── 结果是 RAM → 填充 TLB（普通条目）
      └── 结果是 MMIO MemoryRegion → 填充 TLB（设置 TLB_MMIO 标志）
```

### 15.2 慢速路径处理

`accel/tcg/cputlb.c`：

**加载路径**：
```
do_ld1_mmu()                              [:1729-1793]
  → mmu_lookup()                          // 查找 TLB 条目
  → 检测到 TLB_MMIO 标记                  [:1873-1877]
  → do_ld_mmio_beN()                      [:1970]
    → int_ld_mmio_beN()                   [:1936]
      → BQL_LOCK_GUARD()                  [:1983-1985] ★ 获取 BQL
      → memory_region_dispatch_read()     [:1952]
        → MemoryRegionOps.read()          // 调用设备读处理
      → BQL 释放                          // GUARD 析构
```

**存储路径**：
```
do_st1_mmu()                              [:1820-1877]
  → mmu_lookup()
  → 检测到 TLB_MMIO 标记
  → do_st_mmio_leN()                      [:2484]
    → int_st_mmio_leN()                   [:2450]
      → BQL_LOCK_GUARD()                  [:2500-2502] ★ 获取 BQL
      → memory_region_dispatch_write()
        → MemoryRegionOps.write()         // 调用设备写处理
      → BQL 释放
```

### 15.3 关键点

- **BQL 保护**：MMIO 分发期间持有 BQL，保证设备访问的线程安全
- **同步执行**：MMIO 读写是同步的 — Guest 代码暂停直到设备 handler 返回
- **返回值**：读操作的返回值直接作为 Guest 寄存器值

## 16. KVM MMIO 路径

```
Guest 执行 MMIO 访问
  → EPT/Stage2 页表缺页
  → VM Exit → KVM_EXIT_MMIO
  → kvm_cpu_exec() 处理              [kvm-all.c:3531-3538]
  → address_space_rw(phys_addr, data, len, is_write)
    → flatview_translate()            // 查找目标 MemoryRegion
    → memory_region_dispatch_*()      // 调用设备 handler
  → 数据写入 run->mmio.data
  → KVM_RUN 重入（带着 MMIO 结果继续执行）
```

## 17. 内存分发核心

### 17.1 memory_region_dispatch_read/write()

`system/memory.c:1467-1545`：

```c
MemTxResult memory_region_dispatch_read(MemoryRegion *mr,
                                         hwaddr addr,
                                         uint64_t *pval,
                                         MemOp op,
                                         MemTxAttrs attrs) {
    // 1. 解析别名链
    //    — MemoryRegion 可能是 alias → 追溯到实际 MR

    // 2. 大小检查与对齐处理
    //    — 如果访问大小与 MR 定义不匹配 → 拆分/合并

    // 3. 调用实际 handler
    return memory_region_dispatch_read1(mr, addr, pval, op, attrs);
    //   → mr->ops->read(mr->opaque, addr, size)
    //   或 mr->ops->read_with_attrs(mr->opaque, addr, &val, size, attrs)
}
```

### 17.2 MemoryRegionOps 接口

```c
typedef struct MemoryRegionOps {
    // 基本读写（无属性）
    uint64_t (*read)(void *opaque, hwaddr addr, unsigned size);
    void (*write)(void *opaque, hwaddr addr, uint64_t data, unsigned size);

    // 带属性读写（安全状态、requester_id 等）
    MemTxResult (*read_with_attrs)(void *opaque, hwaddr addr, uint64_t *data,
                                   unsigned size, MemTxAttrs attrs);
    MemTxResult (*write_with_attrs)(void *opaque, hwaddr addr, uint64_t data,
                                    unsigned size, MemTxAttrs attrs);

    // 有效范围
    struct { unsigned min, max; } valid;    // 有效访问大小
    struct { unsigned min, max; bool unaligned; } impl;  // 实现大小
    Endianness endianness;                  // 端序
} MemoryRegionOps;
```

### 17.3 address_space_rw() 入口

`system/physmem.c:3434-3450`：

```c
// 顶层物理内存访问 API
MemTxResult address_space_rw(AddressSpace *as, hwaddr addr,
                              MemTxAttrs attrs, void *buf,
                              hwaddr len, bool is_write) {
    if (is_write) {
        return address_space_write(as, addr, attrs, buf, len);
    } else {
        return address_space_read(as, addr, attrs, buf, len);
    }
}
```

## 18. FlatView 与地址空间查找

### 18.1 查找流程

```
address_space_rw(as, phys_addr, ...)
  │
  ▼
flatview_translate(fv, addr, &xlat, &l, is_write, attrs)
  │  [physmem.c:321-360]
  │
  ├── AddressSpaceDispatch 树查找
  │   — 6 级 PhysPageEntry 基数树
  │   — 根据物理地址逐级查找
  │
  ├── 找到 MemoryRegionSection
  │   — section->mr = 目标 MemoryRegion
  │   — xlat = MR 内部偏移
  │
  └── 返回 MemoryRegion + 偏移
      │
      ▼
  memory_region_dispatch_read/write(mr, xlat, ...)
    │
    └── mr->ops->read/write(mr->opaque, xlat, size)
        → 设备具体 handler
```

### 18.2 FlatView 更新

`system/memory.c:823-835`：

```
MemoryRegion 树变更（设备添加/移除/映射修改）
  → memory_region_transaction_commit()
  → address_space_update_topology()
  → generate_memory_topology()
  → 生成新 FlatView
  → RCU 发布（无锁替换旧 FlatView）
```

**RCU 机制**保证 MMIO 查找在多线程环境下无锁：
- 读取方（vCPU）通过 `rcu_read_lock()` 访问当前 FlatView
- 写入方（设备热插拔）生成新 FlatView，通过 RCU 原子替换

---

# 第五部分：设备 I/O 与完成回调

## 19. 设备 Handler 执行（virtio-blk 示例）

### 19.1 同步阶段（MMIO handler 内）

```
Guest 写 virtio MMIO 寄存器（通知队列）
  │
  ▼
virtio_mmio_write()                          [virtio-mmio.c:247]
  │  offset = VIRTIO_MMIO_QUEUE_NOTIFY
  │
  ├── virtio_queue_notify()                  [virtio.c:2515]
  │   │
  │   ├── virtio_blk_handle_output()         [virtio-blk.c:1044]
  │   │   │
  │   │   ├── 从 VirtQueue 读取请求描述符
  │   │   ├── 解析请求（读/写/flush）
  │   │   │
  │   │   └── submit_requests()              [virtio-blk.c:215-265]
  │   │       │
  │   │       ├── blk_aio_preadv()           // 异步读
  │   │       └── blk_aio_pwritev()          // 异步写
  │   │           → 提交到块层（异步执行）
  │   │           → 立即返回
  │   │
  │   └── 返回                               // ★ MMIO write 完成
  │
  └── Guest 代码继续执行                      // ★ MMIO 同步部分结束
```

### 19.2 关键点

- MMIO handler（`virtio_mmio_write`）是**同步返回**的
- 但实际 I/O 是**异步**的 — `blk_aio_preadv` 只是提交请求
- Guest 不会在 MMIO 写入时阻塞等待 I/O 完成
- I/O 完成通过中断通知 Guest

## 20. 异步 I/O 完成路径

### 20.1 块层完成回调

```
宿主磁盘 I/O 完成
  │  （通过 io_uring / libaio / 线程池）
  │
  ▼
AioContext 事件分发（aio_poll 或 main_loop_wait）
  │
  ├── fd 就绪 → 读取完成事件
  └── 回调链:
      │
      blk_aio_complete()                     // 块设备层
        │
        virtio_blk_rw_complete()             [virtio-blk.c:98]
          │
          virtio_blk_req_complete()           [virtio-blk.c:57-68]
            │
            ├── 写入完成状态到 VirtQueue      // 填充 used ring
            └── virtio_notify()              [virtio.c:2730]
                │
                virtio_irq()                 [virtio.c:2698]
                  │
                  └── qemu_set_irq(proxy->irq, 1)  [virtio-mmio.c:550]
```

### 20.2 完成路径时序

```
时间线:
  ├── T0: Guest 写 MMIO（通知队列）→ 提交异步 I/O → MMIO 返回
  ├── T1: Guest 继续执行其他代码
  ├── T2: 宿主 I/O 完成
  ├── T3: aio_poll/main_loop_wait 检测到完成
  ├── T4: 回调链 → virtio_notify() → qemu_set_irq()
  ├── T5: CPU 在下次 cpu_handle_interrupt() 中看到 IRQ
  └── T6: Guest 进入中断处理 → 读取 VirtQueue → 处理完成
```

## 21. 中断注入与 CPU 唤醒

### 21.1 qemu_set_irq() 到 CPU 中断

```
qemu_set_irq(gic_spi[N], 1)
  │
  ▼
GIC 中断处理（见 GICv3 文档）
  │ gicv3_set_irq() → gicv3_update() → gicv3_cpuif_update()
  │
  ▼
qemu_irq_raise(cs->parent_irq)
  │
  ▼
arm_cpu_set_irq()                           [target/arm/cpu.c]
  │
  ├── cpu->interrupt_request |= CPU_INTERRUPT_HARD
  │   // 设置中断挂起标志
  │
  └── cpu_interrupt(cpu)
      │
      ├── qatomic_set(&cpu->icount_decr.u16.high, -1)
      │   // 使 icount 立即过期 → 强制 TB 退出
      │
      └── qemu_cpu_kick(cpu)
          │
          ├── qemu_cond_broadcast(cpu->halt_cond)
          │   // 如果 CPU halted → 唤醒
          │
          └── tcg_kick_vcpu_thread(cpu)
              └── qatomic_set(&cpu->exit_request, 1)
                  // 强制 cpu_exec_loop 在下次检查时退出
```

### 21.2 CPU 看到中断

```
cpu_exec_loop() 下次迭代:
  │
  ├── cpu_handle_interrupt(cpu)              [cpu-exec.c:777-881]
  │   ├── 检查 CPU_INTERRUPT_HARD
  │   ├── 调用 arm_cpu_exec_interrupt()
  │   │   ├── 检查 PSTATE.I/F（中断掩码）
  │   │   ├── 如果可以接受 → arm_cpu_do_interrupt()
  │   │   │   ├── 保存当前状态到 SPSR
  │   │   │   ├── 读 ICC_IAR_EL1（确认中断）
  │   │   │   ├── PC = VBAR_EL1 + 0x280（IRQ 向量偏移）
  │   │   │   └── 继续执行（在中断处理程序中）
  │   │   └── 如果掩码禁止 → 保持挂起
  │   └── return true（继续 TB 执行）
  │
  └── 查找/执行中断处理程序的 TB
```

## 22. BQL 与线程安全

### 22.1 BQL（Big QEMU Lock）规则

```
谁持有 BQL:
  ┌─────────────────────────────────┐
  │ 主线程       — 始终持有          │
  │ vCPU 线程    — 大部分时间持有     │
  │              — cpu_exec() 期间释放│
  │              — MMIO 时重新获取    │
  │ IOThread     — 不持有            │
  └─────────────────────────────────┘

BQL 保护什么:
  — 设备状态（MemoryRegionOps handler）
  — 全局状态（运行状态、迁移状态）
  — 大多数设备寄存器访问
```

### 22.2 MMIO 路径的 BQL 序列

```
TCG MMIO:
  vCPU 线程（无 BQL，正在执行 guest 代码）
    → 遇到 MMIO 地址
    → BQL_LOCK_GUARD()               // 获取 BQL
    → memory_region_dispatch_*()      // 设备 handler（BQL 下执行）
    → BQL 释放（GUARD 析构）
    → 继续 guest 执行（无 BQL）

KVM MMIO:
  vCPU 线程（持有 BQL）
    → KVM_RUN（释放 BQL）
    → KVM_EXIT_MMIO
    → 重新获取 BQL
    → address_space_rw()              // 设备 handler（BQL 下执行）
    → KVM_RUN（释放 BQL）
```

### 22.3 无 BQL 的设备访问

某些设备使用细粒度锁（而非依赖 BQL）：

```c
// virtio-blk 使用内部锁保护请求列表
WITH_QEMU_LOCK_GUARD(&s->rq_lock) {
    // 访问请求队列
}
```

`virtio_irq()` 设计为可从任何线程调用（ISR 更新是原子的，`virtio.c:2698+`）。

---

# 第六部分：端到端完整流程

## 23. 完整 I/O 请求生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                    完整 I/O 请求生命周期                              │
│                                                                      │
│  Guest 代码                                                          │
│  ├── 1. 准备 VirtQueue 描述符（DMA 地址、长度、标志）                 │
│  ├── 2. 写 MMIO: VIRTIO_MMIO_QUEUE_NOTIFY                          │
│  │      ↓ [MMIO 分发]                                               │
│  │      virtio_mmio_write() → virtio_queue_notify()                  │
│  │      → virtio_blk_handle_output() → blk_aio_preadv()            │
│  │      → 提交异步 I/O → 返回                                       │
│  ├── 3. Guest 继续执行（不等待 I/O）                                 │
│  │                                                                   │
│  │      ═══ 异步间隔 ═══                                            │
│  │                                                                   │
│  │      宿主 I/O 完成                                                │
│  │      ↓ [AIO 事件分发]                                             │
│  │      aio_poll() → 回调链                                          │
│  │      → virtio_blk_rw_complete()                                   │
│  │      → virtio_notify() → qemu_set_irq()                         │
│  │      → GIC → cpu_interrupt()                                      │
│  │      → exit_request = 1                                           │
│  │                                                                   │
│  ├── 4. cpu_handle_interrupt() 检测到 IRQ                            │
│  ├── 5. 进入中断处理程序                                             │
│  ├── 6. 读 ICC_IAR_EL1（确认中断）                                   │
│  ├── 7. 读 VirtQueue used ring（获取完成信息）                       │
│  ├── 8. 处理数据（DMA 区域已包含读取的数据）                         │
│  ├── 9. 写 ICC_EOIR_EL1（结束中断）                                  │
│  └── 10. 返回正常执行                                                │
└─────────────────────────────────────────────────────────────────────┘
```

## 24. 时序分析：一次 virtio-blk 读请求

```
时间轴（非精确比例）:

  vCPU 线程                    主线程/IOThread           宿主内核
  ──────────                   ────────────────          ────────
  │                            │                         │
  │ Guest 准备描述符           │                         │
  │ (DMA 写入 RAM)             │                         │
  │                            │                         │
  │ ★ MMIO write ★             │                         │
  │ → virtio_mmio_write()      │                         │
  │ → blk_aio_preadv()         │                         │
  │   提交 I/O request ─────────┼── ──→ io_submit() ───→ │ 磁盘 I/O
  │ ← MMIO 返回                │                         │ 执行中...
  │                            │                         │
  │ 继续执行 Guest 代码        │                         │
  │ (计算/内存/其他 I/O)       │                         │
  │                            │                         │
  │                            │                         │ ← I/O 完成
  │                            │ aio_poll() 检测完成     │
  │                            │ → 完成回调              │
  │                            │ → virtio_notify()       │
  │                            │ → qemu_set_irq() ──────┼──→ vCPU 中断
  │                            │                         │
  │ ← exit_request = 1         │                         │
  │ cpu_handle_interrupt()     │                         │
  │ → IRQ 接受                 │                         │
  │ → 跳转到中断向量           │                         │
  │                            │                         │
  │ 中断处理程序:              │                         │
  │ → 读 ICC_IAR_EL1           │                         │
  │ → 读 used ring             │                         │
  │ → 处理数据                 │                         │
  │ → 写 ICC_EOIR_EL1          │                         │
  │ → 中断返回                 │                         │
  │                            │                         │
  │ 恢复正常执行               │                         │
```

---

# 附录

## A. 关键源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `system/main.c` | ~100 | 入口 main() |
| `system/vl.c` | ~4,000 | qemu_init()、qemu_default_main() |
| `system/runstate.c` | ~1,000 | qemu_main_loop()、运行状态管理 |
| `util/main-loop.c` | ~650 | main_loop_wait()、GLib 集成 |
| `util/aio-posix.c` | ~760 | aio_poll()、fd 监控 |
| `util/async.c` | ~240 | BH 调度与执行 |
| `util/qemu-timer.c` | ~650 | 定时器基础设施 |
| `iothread.c` | ~220 | IOThread 独立事件循环 |
| `accel/tcg/cpu-exec.c` | 1,087 | cpu_exec()、cpu_exec_loop() |
| `accel/tcg/cputlb.c` | 2,901 | 软件 TLB、MMIO 慢速路径 |
| `accel/tcg/tcg-accel-ops-mttcg.c` | ~140 | MTTCG vCPU 线程 |
| `accel/tcg/tcg-accel-ops-rr.c` | ~350 | RR 单线程 vCPU |
| `accel/kvm/kvm-all.c` | ~5,000 | KVM 加速器（kvm_cpu_exec） |
| `system/memory.c` | ~3,500 | MemoryRegion 分发 |
| `system/physmem.c` | ~3,500 | 物理内存访问、FlatView |
| `include/qemu/aio.h` | ~340 | AioContext 定义 |

## B. CPU_INTERRUPT 标志表

| 标志 | 值 | 含义 |
|------|-----|------|
| `CPU_INTERRUPT_HARD` | 0x0002 | 硬件中断（IRQ） |
| `CPU_INTERRUPT_EXITTB` | 0x0004 | 退出当前 TB |
| `CPU_INTERRUPT_HALT` | 0x0020 | CPU 停机 |
| `CPU_INTERRUPT_RESET` | 0x0040 | CPU 复位 |
| `CPU_INTERRUPT_FIQ` | 0x0100 | ARM FIQ（快速中断） |
| `CPU_INTERRUPT_VIRQ` | 0x0400 | 虚拟 IRQ |
| `CPU_INTERRUPT_VFIQ` | 0x0800 | 虚拟 FIQ |
| `CPU_INTERRUPT_NMI` | 0x1000 | 不可屏蔽中断 |
| `CPU_INTERRUPT_VINMI` | 0x2000 | 虚拟 NMI |

## C. EXCP 异常码表

| 异常码 | 含义 | 处理方式 |
|--------|------|----------|
| `EXCP_INTERRUPT` | 外部中断唤醒 | cpu_exec 返回 0 |
| `EXCP_DEBUG` | 调试断点 | cpu_exec 返回 EXCP_DEBUG |
| `EXCP_HALTED` | CPU halt | cpu_exec 返回 EXCP_HALTED |
| `EXCP_ATOMIC` | 原子指令序列化 | cpu_exec_step_atomic() |
| `EXCP_YIELD` | 让出执行 | cpu_exec 返回 |
| 其他 | Guest 同步异常 | do_interrupt() 处理 |

---

> **相关文档**：
> - `darren/accel/00-TCG深度分析.md` — TCG 翻译/优化/代码生成
> - `darren/memory/00-内存子系统深度分析.md` — MemoryRegion 树与 FlatView
> - `darren/device-model/00-设备模型与virtio深度分析.md` — virtio 框架与设备拓扑
> - `darren/device-model/02-块层IO路径深度分析.md` — 块层异步 I/O 路径
> - `darren/arm64/03-GICv3中断控制器模拟架构深度分析.md` — 中断分发与注入
