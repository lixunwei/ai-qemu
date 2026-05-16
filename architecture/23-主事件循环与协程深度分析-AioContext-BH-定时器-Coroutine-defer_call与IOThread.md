# 23 - 主事件循环与协程深度分析 — AioContext、BH、定时器、Coroutine、defer_call 与 IOThread

> **基于 QEMU 11.0.50 源码**，深入分析 QEMU 事件驱动架构的完整实现：
> AioContext 异步 I/O 上下文、主事件循环、Bottom Half、定时器系统、
> 协程（Coroutine）生命周期与同步原语、defer_call 批处理、
> EventNotifier 事件通知以及 IOThread 多线程模型。

---

## 目录

1. [AioContext：异步 I/O 上下文](#1-aiocontext异步-io-上下文)
2. [主事件循环：main_loop_wait](#2-主事件循环main_loop_wait)
3. [aio_poll：核心轮询机制](#3-aio_poll核心轮询机制)
4. [Bottom Half（BH）：延迟回调](#4-bottom-halfbh延迟回调)
5. [定时器系统：QEMUTimer](#5-定时器系统qemutimer)
6. [Coroutine：协程核心](#6-coroutine协程核心)
7. [协程同步原语：CoMutex 与 CoQueue](#7-协程同步原语comutex-与-coqueue)
8. [协程与 AioContext 集成](#8-协程与-aiocontext-集成)
9. [defer_call：批量延迟调用](#9-defer_call批量延迟调用)
10. [EventNotifier：事件通知](#10-eventnotifier事件通知)
11. [IOThread：独立事件循环线程](#11-iothread独立事件循环线程)
12. [数据流全景与应用场景](#12-数据流全景与应用场景)

---

## 1. AioContext：异步 I/O 上下文

### 1.1 AioContext 结构

```c
// aio.h:201-339
struct AioContext {
    GSource source;                      // GLib 事件源（可集成 GMainLoop）

    QemuRecMutex lock;                   // 多线程保护锁
    struct BdrvGraphRWlock *bdrv_graph;   // 块层图读写锁

    /* fd 处理器 */
    AioHandlerList aio_handlers;          // 已注册的 AIO 处理器
    AioHandlerList deleted_aio_handlers;  // 待删除的处理器

    /* 唤醒机制 */
    uint32_t notify_me;                   // 唤醒请求计数
    bool notified;                        // 是否已通知
    EventNotifier notifier;               // eventfd 通知器

    /* 锁计数 */
    QemuLockCnt list_lock;                // 处理器列表锁

    /* Bottom Half */
    BHList bh_list;                       // 待处理 BH 链表
    QSIMPLEQ_HEAD(, BHListSlice) bh_slice_list;

    /* 协程调度 */
    QSLIST_HEAD(, Coroutine) scheduled_coroutines;
    QEMUBH *co_schedule_bh;              // 协程调度 BH

    /* 线程池 */
    int thread_pool_min;
    int thread_pool_max;
    struct ThreadPoolAio *thread_pool;

    /* 定时器 */
    QEMUTimerListGroup tlg;               // 每种时钟类型一个 TimerList

    /* 自适应轮询参数 */
    int poll_disable_cnt;                 // 禁用轮询的处理器数
    int64_t poll_ns;                      // 当前轮询时间（纳秒）
    int64_t poll_max_ns;                  // 最大轮询时间
    int64_t poll_grow;                    // 增长因子
    int64_t poll_shrink;                  // 收缩因子
    int64_t aio_max_batch;                // 最大批处理请求数

    AioHandlerList poll_aio_handlers;     // 用户态轮询处理器
    bool poll_started;                    // 是否处于轮询模式

    /* epoll */
    int epollfd;                          // epoll 文件描述符
    void *epollfd_tag;

    const FDMonOps *fdmon_ops;            // fd 监控后端（epoll/io_uring/poll）
    bool initialized;
};
```

**设计要点**：
- 每个 AioContext 是一个独立的事件循环单元
- 集成了 fd 监控、BH、定时器、协程调度四大组件
- `notify_me` + `notifier` 实现跨线程唤醒
- 支持三种 fd 监控后端：epoll、io_uring、poll

### 1.2 AioHandler — fd 处理器

```c
// aio-posix.h:23-44
struct AioHandler {
    QLIST_ENTRY(AioHandler) node;
    QLIST_ENTRY(AioHandler) node_ready;
    QLIST_ENTRY(AioHandler) node_deleted;
    QLIST_ENTRY(AioHandler) node_poll;

    int pfd_idx;                    // pollfd 数组索引
    IOHandler *io_read;             // 可读回调
    IOHandler *io_write;            // 可写回调
    AioPollFn *io_poll;             // 用户态轮询回调
    IOHandler *io_poll_ready;       // 轮询就绪回调
    void *opaque;
    bool is_external;
    bool poll_idle;
};
```

### 1.3 创建与引用计数

```c
// async.c:552-627
AioContext *aio_context_new(Error **errp)
{
    // 1. 初始化 GSource
    // 2. 创建 EventNotifier（eventfd）
    // 3. 注册 notifier 到 aio_set_event_notifier()
    // 4. 初始化 BH 列表、定时器组
    // 5. 创建 co_schedule_bh 用于协程调度
    // 6. 初始化 epoll（如果可用）
}

// async.c:714-722
void aio_context_ref(AioContext *ctx)   { g_source_ref(&ctx->source); }
void aio_context_unref(AioContext *ctx) { g_source_unref(&ctx->source); }
```

---

## 2. 主事件循环：main_loop_wait

### 2.1 顶层循环

```c
// runstate.c:943-951
int qemu_main_loop(void)
{
    int status = EXIT_SUCCESS;
    while (!main_loop_should_exit(&status)) {
        main_loop_wait(false);  // 核心：等待并处理事件
    }
    return status;
}
```

### 2.2 main_loop_wait — 单次迭代

```c
// main-loop.c:563-604
void main_loop_wait(int nonblocking)
{
    // 1. 重置 pollfd 数组
    g_array_set_size(gpollfds, 0);

    // 2. 通知所有 poll 观察者（填充 pollfd）
    notifier_list_notify(&main_loop_poll_notifiers, &mlpoll);

    // 3. 计算超时：取定时器最近到期时间
    timeout_ns = qemu_soonest_timeout(timeout_ns,
                    timerlistgroup_deadline_ns(&main_loop_tlg));

    // 4. 阻塞等待（os_host_main_loop_wait → ppoll/g_poll）
    ret = os_host_main_loop_wait(timeout_ns);

    // 5. 通知 poll 观察者结果
    notifier_list_notify(&main_loop_poll_notifiers, &mlpoll);

    // 6. 运行所有到期定时器
    qemu_clock_run_all_timers();
}
```

**主循环 vs AioContext**：
- `main_loop_wait()` 管理全局 GLib 事件源（GTK/SDL 等 UI 事件）
- `qemu_aio_context` 是主线程的 AioContext，处理块层 I/O
- IOThread 各自拥有独立的 AioContext

---

## 3. aio_poll：核心轮询机制

```c
// aio-posix.c:652-758
bool aio_poll(AioContext *ctx, bool blocking)
{
    // 1. 计算超时
    timeout = blocking ? aio_compute_timeout(ctx) : 0;

    // 2. 尝试用户态轮询（busy-polling）
    if (ctx->poll_max_ns != 0)
        progress = try_poll_mode(ctx, &ready_list, &timeout);

    // 3. 设置 notify_me 标志（允许跨线程唤醒）
    if (timeout != 0)
        qatomic_set(&ctx->notify_me, ... + 2);

    // 4. 检查是否已被通知（避免不必要阻塞）
    if (qatomic_read(&ctx->notified))
        timeout = 0;

    // 5. 内核态等待（epoll_wait/ppoll/io_uring）
    if (timeout || need_wait)
        ctx->fdmon_ops->wait(ctx, &ready_list, timeout);

    // 6. 清除 notify_me
    qatomic_store_release(&ctx->notify_me, ... - 2);
    aio_notify_accept(ctx);

    // 7. 分发所有就绪事件
    progress |= aio_bh_poll(ctx);                         // BH
    progress |= aio_dispatch_ready_handlers(ctx, ...);    // fd 回调
    aio_free_deleted_handlers(ctx);

    // 8. 运行定时器
    progress |= timerlistgroup_run_timers(&ctx->tlg);

    return progress;
}
```

### 3.1 自适应轮询

aio_poll 支持两种模式：
1. **用户态轮询**（busy-polling）：在 `poll_ns` 时间内不断调用 `io_poll()` 回调，避免系统调用
2. **内核态等待**：使用 epoll_wait/ppoll 阻塞等待

当用户态轮询在 `poll_ns` 内有结果，`poll_ns` 按 `poll_grow` 增长；
无结果时按 `poll_shrink` 收缩。这实现了自适应的延迟/吞吐量平衡。

### 3.2 fd 处理器注册

```c
// aio-posix.c:98-176
void aio_set_fd_handler(AioContext *ctx, int fd, ...)
{
    // 查找或创建 AioHandler
    // 设置 io_read / io_write / io_poll 回调
    // 注册到 ctx->aio_handlers 链表
    // 通过 fdmon_ops->update() 更新 epoll
}
```

### 3.3 分发流程

```c
// aio-posix.c:293-359 — 单个处理器分发
static bool aio_dispatch_handler(AioContext *ctx, AioHandler *node)
{
    revents = node->pfd.revents;
    if ((revents & G_IO_IN) && node->io_read)
        node->io_read(node->opaque);
    if ((revents & G_IO_OUT) && node->io_write)
        node->io_write(node->opaque);
    return progress;
}

// aio-posix.c:361-382 — 遍历所有就绪处理器
static bool aio_dispatch_ready_handlers(AioContext *ctx, AioHandlerList *ready)
{
    while ((node = QLIST_FIRST(ready))) {
        progress |= aio_dispatch_handler(ctx, node);
    }
}
```

---

## 4. Bottom Half（BH）：延迟回调

### 4.1 QEMUBH 结构

```c
// async.c:63-71
struct QEMUBH {
    AioContext *ctx;            // 所属 AioContext
    const char *name;          // 调试名称
    QEMUBHFunc *cb;            // 回调函数
    void *opaque;              // 回调参数
    QSLIST_ENTRY(QEMUBH) next;
    unsigned flags;            // BH_PENDING | BH_SCHEDULED | BH_IDLE | ...
    MemReentrancyGuard *reentrancy_guard;
};
```

### 4.2 BH 生命周期

```c
// async.c:144-157 — 创建
QEMUBH *aio_bh_new_full(AioContext *ctx, QEMUBHFunc *cb, void *opaque, ...)
{
    bh = g_new(QEMUBH, 1);
    *bh = (QEMUBH){ .ctx = ctx, .cb = cb, .opaque = opaque, ... };
    return bh;
}

// async.c:235-238 — 调度
void qemu_bh_schedule(QEMUBH *bh)
{
    aio_bh_enqueue(bh, BH_SCHEDULED);
}
```

### 4.3 BH 入队与唤醒

```c
// async.c:74-108
static void aio_bh_enqueue(QEMUBH *bh, unsigned new_flags)
{
    // 1. 原子设置 BH_PENDING 标志
    old_flags = qatomic_fetch_or(&bh->flags, BH_PENDING | new_flags);

    // 2. 首次挂起：插入 ctx->bh_list（无锁链表）
    if (!(old_flags & BH_PENDING))
        QSLIST_INSERT_HEAD_ATOMIC(&ctx->bh_list, bh, next);

    // 3. 唤醒 AioContext（通过 eventfd）
    aio_notify(ctx);
}
```

### 4.4 BH 执行（aio_bh_poll）

```c
// async.c:181-228
int aio_bh_poll(AioContext *ctx)
{
    // 1. 原子移动整个 bh_list 到本地 slice
    QSLIST_MOVE_ATOMIC(&slice.bh_list, &ctx->bh_list);

    // 2. 遍历并执行
    while ((bh = aio_bh_dequeue(&s->bh_list, &flags))) {
        if ((flags & BH_SCHEDULED) && !(flags & BH_DELETED))
            aio_bh_call(bh);  // → bh->cb(bh->opaque)
        if (flags & (BH_DELETED | BH_ONESHOT))
            g_free(bh);
    }
}
```

**BH vs 直接调用**：BH 将回调推迟到 aio_poll 的分发阶段，避免在中断/信号上下文中执行复杂操作。类似 Linux 内核的 Bottom Half 概念。

---

## 5. 定时器系统：QEMUTimer

### 5.1 时钟类型

```c
// timer.h:48-54
typedef enum {
    QEMU_CLOCK_REALTIME = 0,    // 真实时间（VM 暂停也运行）
    QEMU_CLOCK_VIRTUAL = 1,     // 虚拟时间（仅 VM 运行时推进）
    QEMU_CLOCK_HOST = 2,        // 宿主时间（NTP 感知）
    QEMU_CLOCK_VIRTUAL_RT = 3,  // 虚拟实时（icount 用）
    QEMU_CLOCK_MAX
} QEMUClockType;
```

### 5.2 QEMUTimer 结构

```c
// timer.h:85-93
struct QEMUTimer {
    int64_t expire_time;         // 到期时间（纳秒）
    QEMUTimerList *timer_list;   // 所属定时器链表
    QEMUTimerCB *cb;             // 回调函数
    void *opaque;                // 回调参数
    QEMUTimer *next;             // 链表下一个
    int attributes;              // QEMU_TIMER_ATTR_EXTERNAL 等
    int scale;                   // 时间缩放因子
};

// timer.h:78-80
struct QEMUTimerListGroup {
    QEMUTimerList *tl[QEMU_CLOCK_MAX];  // 每种时钟一个列表
};
```

### 5.3 定时器操作

```c
// timer.h:540-544
static inline QEMUTimer *timer_new(QEMUTimerList *timer_list,
                                   int scale, QEMUTimerCB *cb, void *opaque)

// qemu-timer.c:498-501
void timer_mod(QEMUTimer *ts, int64_t expire_time)
// 修改定时器到期时间（如已在链表中则先移除再插入）

// qemu-timer.c:447-456
void timer_del(QEMUTimer *ts)
// 从链表移除定时器
```

### 5.4 定时器执行

```c
// qemu-timer.c:518-603
bool timerlist_run_timers(QEMUTimerList *timer_list)
{
    current_time = qemu_clock_get_ns(timer_list->clock->type);

    while ((ts = timer_list->active_timers)) {
        if (ts->expire_time > current_time) break;

        // 移除并执行
        timer_list->active_timers = ts->next;
        ts->cb(ts->opaque);
    }
}
```

### 5.5 与 AioContext 集成

```c
// aio.h:300-303 — AioContext 内嵌定时器组
QEMUTimerListGroup tlg;

// aio-posix.c:756 — aio_poll 末尾运行定时器
progress |= timerlistgroup_run_timers(&ctx->tlg);
```

定时器同时影响 `aio_poll` 的阻塞超时：
- `aio_compute_timeout()` 返回最近定时器到期时间
- 确保 `aio_poll` 不会阻塞超过下一个定时器到期时间

---

## 6. Coroutine：协程核心

### 6.1 Coroutine 结构

```c
// coroutine_int.h:38-42
typedef enum {
    COROUTINE_YIELD = 1,       // 协程让出
    COROUTINE_TERMINATE = 2,   // 协程结束
    COROUTINE_ENTER = 3,       // 协程进入
} CoroutineAction;

// coroutine_int.h:44-70
struct Coroutine {
    CoroutineEntry *entry;           // 入口函数
    void *entry_arg;                 // 入口参数
    Coroutine *caller;               // 调用者协程

    QSLIST_ENTRY(Coroutine) pool_next; // 对象池链表

    size_t locks_held;               // 持有的锁数量

    AioContext *ctx;                  // 让出时记录的 AioContext

    const char *scheduled;           // 调度该协程的函数名（调试）

    QSIMPLEQ_ENTRY(Coroutine) co_queue_next;

    /* 当前协程结束/让出时要唤醒的协程队列 */
    QSIMPLEQ_HEAD(, Coroutine) co_queue_wakeup;

    QSLIST_ENTRY(Coroutine) co_scheduled_next;
};
```

### 6.2 协程生命周期

```c
// qemu-coroutine.c:217-244 — 创建
Coroutine *qemu_coroutine_create(CoroutineEntry *entry, void *opaque)
{
    // 优先从对象池获取，否则新建
    co = coroutine_pool_get();
    if (!co) co = qemu_coroutine_new();

    co->entry = entry;
    co->entry_arg = opaque;
    return co;
}

// qemu-coroutine.c:314-340 — 进入
void qemu_coroutine_enter(Coroutine *co)
// 切换到协程执行，直到 yield 或 terminate

// coroutine-core.h:95-97 — 让出
void qemu_coroutine_yield(void)
// 保存上下文，返回到 caller

// qemu-coroutine.c:235-244 — 终止回收
static void coroutine_delete(Coroutine *co)
// 放回对象池
```

### 6.3 协程后端

QEMU 支持多种协程实现：

| 后端 | 文件 | 机制 |
|------|------|------|
| ucontext | `coroutine-ucontext.c` | POSIX `makecontext`/`swapcontext` |
| sigaltstack | `coroutine-sigaltstack.c` | 信号栈 + `longjmp` |
| Windows Fiber | `coroutine-windows.c` | Win32 Fiber API |
| WASM | `coroutine-wasm.c` | Emscripten Fiber |

```c
// coroutine-ucontext.c:273-359 — ucontext 后端
static void qemu_coroutine_switch(Coroutine *from, Coroutine *to,
                                  CoroutineAction action)
// swapcontext(&from->env, &to->env)
```

### 6.4 对象池

```c
// qemu-coroutine.c:29-47, 176-215
// 每线程维护一个协程对象池，避免频繁 malloc/free
// coroutine_pool_get(): 从池中获取
// coroutine_pool_put(): 放回池中（有上限）
```

---

## 7. 协程同步原语：CoMutex 与 CoQueue

### 7.1 CoMutex — 协程互斥锁

```c
// coroutine.h:49-70
struct CoMutex {
    unsigned locked;              // 是否已锁定
    Coroutine *holder;            // 持有者协程
    CoQueue queue;                // 等待者队列
    // ... 调试字段
};
```

```c
// qemu-coroutine-lock.c:236-274 — 加锁
void qemu_co_mutex_lock(CoMutex *mutex)
{
    // 快速路径：CAS 尝试获取
    if (qatomic_cmpxchg(&mutex->locked, 0, 1) == 0) {
        mutex->holder = self;
        return;
    }
    // 慢速路径：加入等待队列 + yield
    qemu_co_queue_wait(&mutex->queue, ...);
    mutex->holder = self;
}

// qemu-coroutine-lock.c:276-330 — 解锁
void qemu_co_mutex_unlock(CoMutex *mutex)
{
    mutex->holder = NULL;
    // 唤醒下一个等待者
    Coroutine *next = qemu_co_queue_next(&mutex->queue);
    if (next)
        aio_co_wake(next);   // 在 AioContext 中恢复协程
    else
        qatomic_set(&mutex->locked, 0);
}
```

### 7.2 CoQueue — 协程等待队列

```c
// coroutine.h:95-97
struct CoQueue {
    QSIMPLEQ_HEAD(, Coroutine) entries;
};

// qemu-coroutine-lock.c:41-72 — 等待
void qemu_co_queue_wait_impl(CoQueue *queue, ...)
{
    // 将当前协程加入队列
    QSIMPLEQ_INSERT_TAIL(&queue->entries, self, co_queue_next);
    // 让出执行权
    qemu_coroutine_yield();
}

// qemu-coroutine-lock.c:74-111 — 唤醒
bool qemu_co_queue_next(CoQueue *queue)
{
    next = QSIMPLEQ_FIRST(&queue->entries);
    if (next) {
        QSIMPLEQ_REMOVE_HEAD(&queue->entries, co_queue_next);
        aio_co_wake(next);
    }
}

void qemu_co_queue_restart_all(CoQueue *queue)
// 唤醒队列中所有等待者
```

---

## 8. 协程与 AioContext 集成

### 8.1 aio_co_schedule — 跨线程调度协程

```c
// async.c:629-653
void aio_co_schedule(AioContext *ctx, Coroutine *co)
{
    // 1. 记录目标 AioContext
    co->ctx = ctx;

    // 2. 加入 scheduled_coroutines 链表（无锁）
    QSLIST_INSERT_HEAD_ATOMIC(&ctx->scheduled_coroutines, co, co_scheduled_next);

    // 3. 调度 BH（确保在目标 AioContext 线程中执行）
    qemu_bh_schedule(ctx->co_schedule_bh);
}
```

### 8.2 co_schedule_bh_cb — BH 回调进入协程

```c
// async.c:529-550
static void co_schedule_bh_cb(void *opaque)
{
    AioContext *ctx = opaque;
    // 原子取出所有待调度协程
    QSLIST_MOVE_ATOMIC(&straight, &ctx->scheduled_coroutines);

    // 逐个进入协程
    while (co) {
        qemu_aio_coroutine_enter(ctx, co);
    }
}
```

### 8.3 aio_co_wake — 唤醒协程

```c
// async.c:685-710
void aio_co_wake(Coroutine *co)
{
    if (同一线程且同一 AioContext)
        qemu_aio_coroutine_enter(ctx, co);  // 直接进入
    else
        aio_co_schedule(co->ctx, co);        // 跨线程调度
}
```

### 8.4 块层协程模式

```c
// 典型的块层 I/O 协程模式（block/io.c）
int bdrv_co_preadv(BdrvChild *child, ...)
{
    // 在协程中执行
    ret = bdrv_co_do_preadv(child, ...);
    // ↓ 如果需要等待 I/O，协程 yield
    //   I/O 完成后通过 BH 恢复协程
    return ret;
}
```

---

## 9. defer_call：批量延迟调用

### 9.1 设计目的

在 IOThread 中，多个 VirtQueue 的中断通知可能在一次 aio_poll 中产生。`defer_call` 将这些通知批量合并，避免重复的 `event_notifier_set` 系统调用。

### 9.2 数据结构

```c
// defer-call.c:28-37
typedef struct {
    void (*fn)(void *);
    void *opaque;
} DeferredCall;

typedef struct {
    unsigned nesting_level;              // 嵌套层级
    GArray *deferred_call_array;         // 延迟调用数组
} DeferCallThreadState;
```

### 9.3 核心实现

```c
// defer-call.c:67-103
void defer_call(void (*fn)(void *), void *opaque)
{
    // 不在 defer 区间内：立即执行
    if (nesting_level == 0) {
        fn(opaque);
        return;
    }

    // 去重检查（线性扫描，数量少）
    for (i = 0; i < array->len; i++)
        if (fns[i].fn == fn && fns[i].opaque == opaque)
            return;  // 已存在，跳过

    // 追加到延迟数组
    g_array_append_val(array, new_fn);
}

// defer-call.c:115-122
void defer_call_begin(void)
{
    thread_state->nesting_level++;    // 支持嵌套
}

// defer-call.c:130-156
void defer_call_end(void)
{
    if (--nesting_level > 0) return;  // 仅最外层执行

    // 执行所有延迟调用
    for (i = 0; i < array->len; i++)
        fns[i].fn(fns[i].opaque);

    g_array_set_size(array, 0);       // 清空（不释放内存）
}
```

### 9.4 VirtIO 中断批处理

```c
// virtio.c:2698-2727 — virtio_irq()
static void virtio_irq(VirtQueue *vq)
{
    virtio_set_isr(vq->vdev, 0x1);

    if (qemu_in_iothread())
        // IOThread 中：延迟到 defer_call_end() 批量执行
        defer_call(virtio_notify_irqfd_deferred_fn, &vq->guest_notifier);
    else
        // 主线程：直接通知
        virtio_notify_vector(vq->vdev, vq->vector);
}
```

---

## 10. EventNotifier：事件通知

### 10.1 结构

```c
// event_notifier.h:21-29
struct EventNotifier {
    int rfd;          // 读端 fd（eventfd 时 rfd == wfd）
    int wfd;          // 写端 fd
    bool initialized;
};
```

### 10.2 操作

```c
// event_notifier-posix.c:36-79 — 初始化
int event_notifier_init(EventNotifier *e, int active)
{
    // 优先使用 eventfd（单 fd，高效）
    ret = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (ret >= 0) {
        e->rfd = e->wfd = ret;
    } else {
        // 回退到 pipe
        pipe2(fds, O_NONBLOCK | O_CLOEXEC);
        e->rfd = fds[0]; e->wfd = fds[1];
    }
}

// event_notifier-posix.c:107-125 — 触发
int event_notifier_set(EventNotifier *e)
{
    uint64_t value = 1;
    write(e->wfd, &value, sizeof(value));  // 写入 eventfd
}

// event_notifier-posix.c:127-145 — 消费
int event_notifier_test_and_clear(EventNotifier *e)
{
    uint64_t value;
    read(e->rfd, &value, sizeof(value));   // 读取 eventfd（清除）
    return value != 0;
}
```

### 10.3 与 AioContext 集成

```c
// aio-posix.c:192-201
void aio_set_event_notifier(AioContext *ctx, EventNotifier *notifier,
                            EventNotifierHandler *io_read, ...)
{
    // 封装为 fd handler
    aio_set_fd_handler(ctx, event_notifier_get_fd(notifier),
                       io_read, NULL, NULL, NULL, notifier);
}
```

**用途**：
- AioContext 跨线程唤醒（`ctx->notifier`）
- ioeventfd（VirtIO kick → QEMU 唤醒）
- irqfd（vhost 中断注入）

---

## 11. IOThread：独立事件循环线程

### 11.1 IOThread 结构

```c
// iothread.h:41-59
struct IOThread {
    Object parent_obj;
    QemuThread thread;
    AioContext *ctx;              // 独立的 AioContext
    GMainContext *worker_context; // GLib 主上下文
    GMainLoop *main_loop;
    QemuSemaphore init_done_sem;
    bool running;
    bool run_gcontext;           // 是否运行 GLib 事件循环
    int64_t thread_id;
    ...
};
```

### 11.2 IOThread 主循环

```c
// iothread.c:28-65
static void *iothread_run(void *opaque)
{
    IOThread *iothread = opaque;

    rcu_register_thread();
    g_main_context_push_thread_default(iothread->worker_context);
    qemu_set_current_aio_context(iothread->ctx);

    while (iothread->running) {
        // 核心：在 IOThread 自己的 AioContext 上执行 aio_poll
        aio_poll(iothread->ctx, true);

        // 可选：运行 GLib 事件循环（用于 glib 组件）
        if (iothread->running && qatomic_read(&iothread->run_gcontext))
            g_main_loop_run(iothread->main_loop);
    }

    rcu_unregister_thread();
}
```

**IOThread 隔离**：每个 IOThread 拥有独立的 AioContext，块设备 I/O 可分配到不同 IOThread，实现 I/O 并行化，避免与主线程 BQL 竞争。

---

## 12. 数据流全景与应用场景

### 12.1 事件循环全景图

```
                        ┌──────────────────────────┐
                        │      qemu_main_loop()    │
                        │  while(!should_exit)     │
                        │    main_loop_wait()      │
                        └──────────┬───────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
    ┌─────▼──────┐          ┌──────▼──────┐          ┌──────▼──────┐
    │ GLib 事件源 │          │qemu_aio_ctx │          │  IOThread   │
    │ (UI/GTK)   │          │ aio_poll()  │          │ aio_poll()  │
    └────────────┘          └──────┬──────┘          └──────┬──────┘
                                   │                        │
          ┌──────────┬─────────────┼──────────┬─────────────┤
          │          │             │          │             │
     ┌────▼───┐ ┌───▼────┐ ┌─────▼────┐ ┌──▼───┐  ┌─────▼──────┐
     │fd 回调  │ │  BH    │ │  定时器  │ │协程   │  │ defer_call │
     │io_read │ │aio_bh  │ │timerlist │ │co_   │  │ 批量中断   │
     │io_write│ │_poll() │ │_run_     │ │schedule│ │ 通知       │
     └────────┘ └────────┘ │timers()  │ │_bh   │  └────────────┘
                           └──────────┘ └──────┘
```

### 12.2 VirtIO I/O 路径中的事件循环

```
Guest kick (Notify 寄存器写入)
  → KVM ioeventfd
  → EventNotifier (eventfd)
  → AioContext aio_poll() 检测到 fd 事件
  → aio_dispatch_handler() → io_read 回调
  → virtio_queue_host_notifier_read()
  → 进入 defer_call_begin() 区间
  → handle_output() 处理 VirtQueue
  → virtio_notify() → defer_call(irqfd_fn)  [批量合并]
  → defer_call_end() → event_notifier_set() [单次系统调用]
  → KVM irqfd → 注入中断 → Guest
```

### 12.3 块层 I/O 路径中的协程

```
Guest 写 VirtIO-blk 请求
  → handle_output()
  → qemu_coroutine_create(virtio_blk_handle_vq_co)
  → qemu_coroutine_enter()
    → bdrv_co_pwritev()
      → 提交 I/O 请求到底层驱动
      → qemu_coroutine_yield()  [等待 I/O 完成]
  → aio_poll() 继续处理其他事件

I/O 完成（内核回调）
  → fd handler → aio_co_wake(co)
  → qemu_coroutine_enter(co)
    → bdrv_co_pwritev() 返回
    → virtqueue_push() + virtio_notify()
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/qemu/aio.h` | ~340 | AioContext (201-339) |
| `util/aio-posix.h` | ~50 | AioHandler (23-44) |
| `util/aio-posix.c` | ~800 | aio_set_fd_handler (98)、aio_dispatch_handler (293)、aio_dispatch_ready_handlers (361)、aio_dispatch (384)、aio_poll (652-758) |
| `util/async.c` | ~730 | QEMUBH (63-71)、aio_bh_enqueue (74)、aio_bh_new_full (144)、aio_bh_call (159)、aio_bh_poll (181)、qemu_bh_schedule (235)、co_schedule_bh_cb (529)、aio_context_new (552)、aio_co_schedule (629)、aio_co_wake (685)、aio_context_ref/unref (714) |
| `util/main-loop.c` | ~610 | qemu_aio_context (135)、main_loop_wait (563-604) |
| `system/runstate.c` | ~955 | qemu_main_loop (943-951) |
| `include/qemu/timer.h` | ~550 | QEMUClockType (48-54)、QEMUTimerListGroup (78-80)、QEMUTimer (85-93)、timer_new (540) |
| `util/qemu-timer.c` | ~610 | QEMUTimerList (68-78)、timer_del (447)、timer_mod (498)、timerlist_run_timers (518-603) |
| `include/qemu/coroutine_int.h` | ~72 | CoroutineAction (38-42)、Coroutine (44-70) |
| `util/qemu-coroutine.c` | ~400 | 对象池 (29-215)、qemu_coroutine_create (217)、coroutine_delete (235)、qemu_aio_coroutine_enter (246)、qemu_coroutine_enter (314) |
| `util/coroutine-ucontext.c` | ~360 | coroutine_trampoline (148)、qemu_coroutine_switch (273) |
| `include/qemu/coroutine.h` | ~155 | CoMutex (49-70)、CoQueue (95-97) |
| `util/qemu-coroutine-lock.c` | ~470 | co_queue_wait (41)、co_queue_next (74)、co_mutex_lock (236)、co_mutex_unlock (276) |
| `util/defer-call.c` | ~157 | defer_call (67-103)、defer_call_begin (115)、defer_call_end (130-156) |
| `include/qemu/event_notifier.h` | ~45 | EventNotifier (21-29) |
| `util/event_notifier-posix.c` | ~150 | init (36)、set (107)、test_and_clear (127) |
| `iothread.c` | ~200 | iothread_run (28-65) |
| `include/system/iothread.h` | ~60 | IOThread (41-59) |
