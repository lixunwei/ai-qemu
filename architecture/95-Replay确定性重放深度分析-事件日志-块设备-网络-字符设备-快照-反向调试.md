# Doc 95: Replay 确定性重放深度分析

## 事件日志 · 块设备 · 网络 · 字符设备 · 快照 · 反向调试

> QEMU 11.0.50 · replay/ 子系统 · ISP RAS 实现
> 分析日期: 2025-01

---

## 目录

1. [架构总览与设计哲学](#1-架构总览与设计哲学)
2. [与 icount 的关系](#2-与-icount-的关系)
3. [核心数据结构](#3-核心数据结构)
4. [日志文件格式](#4-日志文件格式)
5. [会话生命周期](#5-会话生命周期)
6. [互斥锁与线程同步](#6-互斥锁与线程同步)
7. [Checkpoint 同步点机制](#7-checkpoint-同步点机制)
8. [指令计数与事件边界](#8-指令计数与事件边界)
9. [异步事件队列](#9-异步事件队列)
10. [时钟录制与回放](#10-时钟录制与回放)
11. [输入设备重放](#11-输入设备重放)
12. [字符设备重放](#12-字符设备重放)
13. [网络包重放](#13-网络包重放)
14. [块设备重放](#14-块设备重放)
15. [快照与 VMState 集成](#15-快照与-vmstate-集成)
16. [反向调试原理](#16-反向调试原理)
17. [音频重放](#17-音频重放)
18. [随机数重放](#18-随机数重放)
19. [限制与 Blocker 机制](#19-限制与-blocker-机制)
20. [与硬件实际行为对比](#20-与硬件实际行为对比)
21. [使用示例与命令行](#21-使用示例与命令行)

---

## 1. 架构总览与设计哲学

### 1.1 核心思想

Record/Replay 的本质是：**将所有非确定性输入记录到日志，重放时从日志注入相同输入，配合确定性 icount 执行，实现完全可复现的执行**。

非确定性来源分类：

| 来源类别 | 具体事件 | 处理方式 |
|---------|---------|---------|
| 时间 | 主机时钟读取 | EVENT_CLOCK 记录返回值 |
| 外设输入 | 网络包到达 | ASYNC_EVENT_NET 记录包内容 |
| 用户输入 | 键盘/鼠标 | ASYNC_EVENT_INPUT 记录事件 |
| 串口输入 | 字符设备后端写入 | ASYNC_EVENT_CHAR_READ |
| 磁盘完成顺序 | 块 I/O 回调 | ASYNC_EVENT_BLOCK + request ID |
| Bottom Half 调度 | BH 回调时机 | ASYNC_EVENT_BH |
| 随机数 | 外部随机源 | EVENT_RANDOM |
| 关机事件 | 用户关闭窗口 | EVENT_SHUTDOWN |

### 1.2 源码组织

```
replay/
├── replay.c              # 主控: configure/start/finish/checkpoint
├── replay-internal.h     # 内部状态、事件枚举、序列化声明
├── replay-internal.c     # 日志 I/O、互斥锁、icount 推进
├── replay-events.c       # 异步事件队列管理
├── replay-time.c         # 时钟录制/回放
├── replay-input.c        # 输入设备录制/回放
├── replay-char.c         # 字符设备录制/回放
├── replay-net.c          # 网络包录制/回放
├── replay-snapshot.c     # VMState 快照集成
├── replay-random.c       # 随机数录制/回放
├── replay-debugging.c    # 反向调试支持

block/blkreplay.c         # 块设备 filter driver
net/filter-replay.c       # 网络 filter
include/system/replay.h   # 公共 API
include/exec/replay-core.h # ReplayMode 定义
```

### 1.3 关键约束

- **必须启用 icount**: `replay_start()` 强制要求 `icount_enabled()` 为 true
- **单 vCPU 串行执行**: replay_mutex 保证 vCPU 线程与主循环严格交替
- **确定性设备模型**: 所有影响 guest state 的设备必须只使用 virtual clock

---

## 2. 与 icount 的关系

### 2.1 依赖关系

```
icount 提供:
  - 确定性虚拟时间推进
  - 精确的指令计数 (qemu_icount)
  - TB 预算机制 (指令边界退出)

Replay 利用 icount:
  - 以指令数定位事件发生时刻
  - replay_get_instructions() 限制 TB 执行到下一事件
  - icount_to_ns() 将指令数转换为虚拟时间
```

### 2.2 icount → Replay 接口

```c
// replay 获取当前精确 icount
uint64_t replay_get_current_icount(void);

// replay 限制 TB 预算 (在 play 模式)
int replay_get_instructions(void) {
    // 返回 replay_state.instruction_count
    // TCG 将此作为最大执行指令数
}

// 执行完成后更新
void replay_account_executed_instructions(void);
```

### 2.3 与 Doc 93 的区别

Doc 93 侧重 icount 本身的预算/warp/timer 机制；本文档侧重 Replay 如何利用 icount 提供的精确指令位置来实现完整的 I/O、网络、输入等设备的确定性重放。

---

## 3. 核心数据结构

### 3.1 ReplayState（全局状态）

```c
// replay/replay-internal.h:90-101
typedef struct ReplayState {
    int64_t cached_clock[REPLAY_CLOCK_COUNT];  // 缓存的时钟值
    uint64_t current_icount;    // 已处理的指令总数
    int instruction_count;      // 到下一事件的剩余指令数
    unsigned int current_event; // 当前事件序号（调试用）
    unsigned int data_kind;     // 当前待处理事件类型
    bool has_unread_data;       // 事件是否已读取但未消费
    uint64_t file_offset;       // 快照时的日志文件偏移
    uint64_t block_request_id;  // 块设备请求 ID 计数器
    uint64_t read_event_id;     // 异步读事件 ID
    size_t n_audio_samples;     // 音频采样数
} ReplayState;

extern ReplayState replay_state;  // 唯一全局实例
```

### 3.2 事件类型枚举

```c
// replay/replay-internal.h:34-68
enum ReplayEvents {
    EVENT_INSTRUCTION,          // 指令计数事件
    EVENT_INTERRUPT,            // 中断同步
    EVENT_EXCEPTION,            // 异常同步
    EVENT_ASYNC,                // 异步事件组起始
    EVENT_ASYNC_LAST,           // = EVENT_ASYNC + REPLAY_ASYNC_COUNT - 1
    EVENT_SHUTDOWN,             // 关机事件组起始
    EVENT_SHUTDOWN_LAST,        // = EVENT_SHUTDOWN + SHUTDOWN_CAUSE__MAX
    EVENT_CHAR_WRITE,           // 字符设备写同步
    EVENT_CHAR_READ_ALL,        // 字符设备读同步
    EVENT_CHAR_READ_ALL_ERROR,  // 字符设备读错误
    EVENT_AUDIO_OUT,            // 音频输出
    EVENT_AUDIO_IN,             // 音频输入
    EVENT_RANDOM,               // 随机数
    EVENT_CLOCK,                // 时钟组起始
    EVENT_CLOCK_LAST,           // = EVENT_CLOCK + REPLAY_CLOCK_COUNT - 1
    EVENT_CHECKPOINT,           // 检查点组起始
    EVENT_CHECKPOINT_LAST,      // = EVENT_CHECKPOINT + CHECKPOINT_COUNT - 1
    EVENT_END,                  // 日志结束标记
    EVENT_COUNT
};
```

### 3.3 异步事件类型

```c
// replay/replay-internal.h:17-26
typedef enum ReplayAsyncEventKind {
    REPLAY_ASYNC_EVENT_BH,          // Bottom Half 回调
    REPLAY_ASYNC_EVENT_BH_ONESHOT,  // 一次性 BH
    REPLAY_ASYNC_EVENT_INPUT,       // 输入事件
    REPLAY_ASYNC_EVENT_INPUT_SYNC,  // 输入同步
    REPLAY_ASYNC_EVENT_CHAR_READ,   // 字符设备读
    REPLAY_ASYNC_EVENT_BLOCK,       // 块设备操作
    REPLAY_ASYNC_EVENT_NET,         // 网络包
    REPLAY_ASYNC_COUNT
} ReplayAsyncEventKind;
```

### 3.4 时钟与检查点种类

```c
// include/system/replay.h:22-43
enum ReplayClockKind {
    REPLAY_CLOCK_HOST,       // 主机时钟 (gettimeofday 等)
    REPLAY_CLOCK_VIRTUAL_RT, // 虚拟实时时钟 (warp 用)
    REPLAY_CLOCK_COUNT
};

enum ReplayCheckpoint {
    CHECKPOINT_CLOCK_WARP_START,    // warp 定时器启动
    CHECKPOINT_CLOCK_WARP_ACCOUNT,  // warp 结算
    CHECKPOINT_RESET_REQUESTED,     // 重置请求
    CHECKPOINT_SUSPEND_REQUESTED,   // 挂起请求
    CHECKPOINT_CLOCK_VIRTUAL,       // 虚拟时钟处理
    CHECKPOINT_CLOCK_HOST,          // 主机时钟处理
    CHECKPOINT_CLOCK_VIRTUAL_RT,    // 虚拟实时时钟处理
    CHECKPOINT_INIT,                // 初始化
    CHECKPOINT_RESET,               // 重置
    CHECKPOINT_COUNT
};
```

---

## 4. 日志文件格式

### 4.1 文件头

```
偏移  大小    内容
0     4B     REPLAY_VERSION (0xe0200d)
4     8B     保留 (fseek 跳过后写入)
```

- 版本号在 `replay_finish()` 时写入（RECORD 模式先 fseek 到文件头）
- 版本号变化表示日志格式不兼容

### 4.2 事件序列

```
[事件1][事件2]...[EVENT_END]

每个事件:
  1B: event_id (enum ReplayEvents)
  ?B: 事件参数（取决于类型）
```

### 4.3 各事件格式

| 事件 | ID | 参数 |
|------|-----|------|
| EVENT_INSTRUCTION | 0 | 4B 指令数 |
| EVENT_INTERRUPT | 1 | 无 |
| EVENT_EXCEPTION | 2 | 无 |
| EVENT_ASYNC + kind | 3~9 | kind-specific (见下) |
| EVENT_SHUTDOWN + cause | 10~21 | 无 |
| EVENT_CHAR_WRITE | 22 | 4B res + 4B offset |
| EVENT_CHAR_READ_ALL | 23 | 4B len + data[] |
| EVENT_CHAR_READ_ALL_ERROR | 24 | 4B error_code |
| EVENT_AUDIO_OUT | 25 | size_t played |
| EVENT_AUDIO_IN | 26 | size_t recorded + samples |
| EVENT_RANDOM | 27 | 4B len + data[] |
| EVENT_CLOCK + kind | 28~29 | 8B clock_value |
| EVENT_CHECKPOINT + id | 30~38 | 无 |
| EVENT_END | 39 | 无 |

### 4.4 异步事件子格式

| 异步类型 | 参数 |
|---------|------|
| BH | 8B operation_id |
| BH_ONESHOT | 8B operation_id |
| INPUT | 4B type + type-specific (9-16B) |
| INPUT_SYNC | 无 |
| CHAR_READ | 1B driver_id + 4B len + data[] |
| BLOCK | 8B request_id |
| NET | 1B adapter_id + 4B flags + 4B len + data[] |

### 4.5 序列化基础函数

```c
// replay/replay-internal.c:48-189
void replay_put_byte(uint8_t byte);    // fputc
void replay_put_word(uint16_t word);   // big-endian 2B
void replay_put_dword(uint32_t dword); // big-endian 4B
void replay_put_qword(int64_t qword); // big-endian 8B
void replay_put_array(const uint8_t *buf, size_t size);  // 4B len + data

uint8_t replay_get_byte(void);         // fgetc
uint16_t replay_get_word(void);
uint32_t replay_get_dword(void);
int64_t replay_get_qword(void);
void replay_get_array(uint8_t *buf, size_t *size);
void replay_get_array_alloc(uint8_t **buf, size_t *size);
```

所有多字节值使用**大端序**存储，保证跨平台可移植。

---

## 5. 会话生命周期

### 5.1 配置阶段

```c
// replay/replay.c:395-434
void replay_configure(QemuOpts *opts) {
    rr = qemu_opt_get(opts, "rr");        // "record" 或 "replay"
    fname = qemu_opt_get(opts, "rrfile"); // 日志文件路径
    replay_snapshot = qemu_opt_get(opts, "rrsnapshot"); // 可选快照名

    replay_vmstate_register();  // 注册 VMState (保存/恢复 replay 状态)
    replay_enable(fname, mode); // 打开文件，初始化状态
}
```

### 5.2 启用阶段

```c
// replay/replay.c:341-393
static void replay_enable(const char *fname, int mode) {
    replay_file = fopen(fname, mode == RECORD ? "wb" : "rb");
    replay_mode = mode;
    replay_mutex_init();

    // 初始化状态
    replay_state.data_kind = -1;
    replay_state.instruction_count = 0;
    replay_state.current_icount = 0;

    if (mode == RECORD) {
        fseek(replay_file, HEADER_SIZE, SEEK_SET);  // 跳过头，最后写
    } else {
        uint32_t version = replay_get_dword();       // 验证版本
        fseek(replay_file, HEADER_SIZE, SEEK_SET);
        replay_fetch_data_kind();                    // 读取第一个事件
    }

    replay_init_events();
}
```

### 5.3 启动阶段

```c
// replay/replay.c:436-454
void replay_start(void) {
    if (replay_blockers) { exit(1); }         // 检查 blocker
    if (!icount_enabled()) { exit(1); }       // 必须启用 icount
    replay_enable_events();                    // 启用事件队列
}
```

### 5.4 运行阶段

主循环中的交互：

```
┌──────────────────────────────────────────────────┐
│                  主循环线程                        │
│                                                  │
│  replay_mutex_lock()                             │
│  → replay_checkpoint(CLOCK_VIRTUAL)              │
│  → 处理虚拟时钟定时器                              │
│  → replay_async_events()  // 处理/记录异步事件     │
│  replay_mutex_unlock()                           │
│                                                  │
└──────────────────────────────────────────────────┘
         ↕ ping-pong (公平互斥)
┌──────────────────────────────────────────────────┐
│                  vCPU 线程                         │
│                                                  │
│  replay_mutex_lock()                             │
│  → 获取预算: replay_get_instructions()            │
│  → 执行 TB (最多 N 条指令)                        │
│  → replay_account_executed_instructions()        │
│  → replay_advance_current_icount()               │
│  replay_mutex_unlock()                           │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 5.5 结束阶段

```c
// replay/replay.c:456-491
void replay_finish(void) {
    replay_save_instructions();        // 刷新最后的指令计数
    if (replay_mode == RECORD) {
        replay_shutdown_request(SHUTDOWN_CAUSE_HOST_SIGNAL);
        replay_put_event(EVENT_END);   // 写入结束标记
        fseek(replay_file, 0, SEEK_SET);
        replay_put_dword(REPLAY_VERSION); // 回写版本号到文件头
    }
    fclose(replay_file);
    replay_mode = REPLAY_MODE_NONE;
}
```

---

## 6. 互斥锁与线程同步

### 6.1 公平排队锁

```c
// replay/replay-internal.c:236-277
static QemuMutex lock;
static QemuCond mutex_cond;
static unsigned long mutex_head, mutex_tail;
static __thread bool replay_locked;

void replay_mutex_lock(void) {
    g_assert(!bql_locked());           // 禁止 BQL 嵌套！
    g_assert(!replay_mutex_locked());  // 防止重入

    qemu_mutex_lock(&lock);
    unsigned long id = mutex_tail++;   // 取号
    while (id != mutex_head) {         // 等待轮到自己
        qemu_cond_wait(&mutex_cond, &lock);
    }
    replay_locked = true;
    qemu_mutex_unlock(&lock);
}

void replay_mutex_unlock(void) {
    qemu_mutex_lock(&lock);
    ++mutex_head;                      // 推进队列头
    replay_locked = false;
    qemu_cond_broadcast(&mutex_cond);  // 唤醒所有等待者
    qemu_mutex_unlock(&lock);
}
```

### 6.2 设计要点

- **取号排队**：`mutex_tail++` 分配序号，`mutex_head` 推进，保证 FIFO 顺序
- **禁止 BQL 嵌套**：replay_lock 是比 BQL 更粗粒度的锁，持有 BQL 时不能再获取 replay_lock（防止死锁）
- **线程局部状态**：`__thread bool replay_locked` 用于断言检查
- **确定性 ping-pong**：主循环线程和 vCPU 线程严格交替持有锁

### 6.3 锁序规则

```
正确: replay_mutex_lock() → bql_lock() → ... → bql_unlock() → replay_mutex_unlock()
错误: bql_lock() → replay_mutex_lock()  // 触发 assert 失败
```

---

## 7. Checkpoint 同步点机制

### 7.1 作用

Checkpoint 是主循环中的同步标记，确保定时器处理、异步事件、时钟 warp 等操作在 record/play 模式下以相同顺序发生。

### 7.2 实现

```c
// replay/replay.c:274-291
bool replay_checkpoint(ReplayCheckpoint checkpoint) {
    assert(EVENT_CHECKPOINT + checkpoint <= EVENT_CHECKPOINT_LAST);
    replay_save_instructions();  // 先刷新指令计数

    if (replay_mode == REPLAY_MODE_PLAY) {
        g_assert(replay_mutex_locked());
        if (replay_next_event_is(EVENT_CHECKPOINT + checkpoint)) {
            replay_finish_event();  // 消费此 checkpoint
        } else {
            return false;  // 不是期望的 checkpoint，跳过处理
        }
    } else if (replay_mode == REPLAY_MODE_RECORD) {
        g_assert(replay_mutex_locked());
        replay_put_event(EVENT_CHECKPOINT + checkpoint);
    }
    return true;
}
```

### 7.3 Checkpoint 类型与触发位置

| Checkpoint | 触发位置 | 作用 |
|-----------|---------|------|
| CLOCK_WARP_START | `icount_start_warp_timer()` | 开始虚拟时钟 warp |
| CLOCK_WARP_ACCOUNT | `icount_account_warp_timer()` | 结算 warp 时间 |
| RESET_REQUESTED | 主循环重置处理 | 同步重置请求 |
| SUSPEND_REQUESTED | 主循环挂起处理 | 同步挂起请求 |
| CLOCK_VIRTUAL | 虚拟时钟定时器处理 | 同步定时器回调 |
| CLOCK_HOST | 主机时钟定时器处理 | 同步主机时钟 |
| CLOCK_VIRTUAL_RT | 虚拟实时时钟处理 | 同步 VRT |
| INIT | VM 初始化完成 | 同步初始化 |
| RESET | VM 重置完成 | 同步重置 |

### 7.4 Play 模式语义

在 PLAY 模式下：
- 主循环尝试处理定时器时，先调用 `replay_checkpoint()`
- 如果日志中下一个事件不是对应 checkpoint，返回 false → **跳过这次定时器处理**
- 这保证了定时器触发顺序与录制时完全一致

---

## 8. 指令计数与事件边界

### 8.1 核心推进函数

```c
// replay/replay-internal.c:279-323
void replay_advance_current_icount(uint64_t current_icount) {
    int diff = (int)(current_icount - replay_state.current_icount);
    assert(diff >= 0);  // 时间只能前进

    if (replay_mode == REPLAY_MODE_RECORD) {
        if (diff > 0) {
            replay_put_event(EVENT_INSTRUCTION);
            replay_put_dword(diff);                    // 记录执行了多少条指令
            replay_state.current_icount += diff;
        }
    } else if (replay_mode == REPLAY_MODE_PLAY) {
        if (diff > 0) {
            replay_state.instruction_count -= diff;     // 消耗预算
            replay_state.current_icount += diff;
            if (replay_state.instruction_count == 0) {
                // 预算耗尽，读取下一个事件
                assert(replay_state.data_kind == EVENT_INSTRUCTION);
                replay_finish_event();
                qemu_notify_event();  // 唤醒 iothread 处理定时器
            }
        }
        // 到达断点
        if (replay_break_icount == replay_state.current_icount) {
            timer_mod_ns(replay_break_timer, qemu_clock_get_ns(QEMU_CLOCK_REALTIME));
        }
    }
}
```

### 8.2 事件读取状态机

```c
// replay/replay-internal.c:206-225
void replay_fetch_data_kind(void) {
    if (!replay_state.has_unread_data) {
        replay_state.data_kind = replay_getc();    // 读 1B 事件 ID
        replay_state.current_event++;
        if (replay_state.data_kind == EVENT_INSTRUCTION) {
            replay_state.instruction_count = replay_get_dword();  // 读 4B 计数
        }
        replay_state.has_unread_data = true;
    }
}
```

### 8.3 执行流程（PLAY 模式）

```
1. replay_fetch_data_kind() 读取 EVENT_INSTRUCTION(N)
2. replay_state.instruction_count = N
3. vCPU 执行最多 N 条指令
4. replay_advance_current_icount() 减少计数
5. 当 instruction_count == 0:
   - replay_finish_event() → replay_fetch_data_kind()
   - 读取下一事件 (可能是 CLOCK/CHECKPOINT/ASYNC/...)
   - 对应事件处理函数被调用
6. 循环
```

---

## 9. 异步事件队列

### 9.1 事件节点

```c
// replay/replay-events.c:20-29
typedef struct Event {
    ReplayAsyncEventKind event_kind;
    void *opaque;       // 事件数据 (BH指针/InputEvent/NetEvent/CharEvent)
    void *opaque2;      // 附加参数
    uint64_t id;        // 操作 ID
    QTAILQ_ENTRY(Event) events;
} Event;

static QTAILQ_HEAD(, Event) events_list;
static bool events_enabled;
```

### 9.2 事件入队

```c
// replay/replay-events.c:95-123
void replay_add_event(ReplayAsyncEventKind event_kind,
                      void *opaque, void *opaque2, uint64_t id) {
    if (!replay_file || !events_enabled) {
        // 无 replay 时直接执行
        Event e = { event_kind, opaque, opaque2, id };
        replay_run_event(&e);
        return;
    }

    Event *event = g_new0(Event, 1);
    event->event_kind = event_kind;
    event->opaque = opaque;
    event->opaque2 = opaque2;
    event->id = id;

    g_assert(replay_mutex_locked());
    QTAILQ_INSERT_TAIL(&events_list, event, events);
    cpu_exit(first_cpu);  // 踢 vCPU 退出执行循环
}
```

**关键设计**：`cpu_exit(first_cpu)` 强制 vCPU 在下一个 TB 边界退出，让主循环有机会处理异步事件。

### 9.3 事件保存与回放

```c
// RECORD 模式: replay_save_events()
void replay_save_events(void) {
    while (!QTAILQ_EMPTY(&events_list)) {
        Event *event = QTAILQ_FIRST(&events_list);
        // 写入异步事件
        replay_put_event(EVENT_ASYNC + event->event_kind);
        // 写入事件参数
        switch (event->event_kind) {
            case REPLAY_ASYNC_EVENT_BH: replay_put_qword(event->id); break;
            case REPLAY_ASYNC_EVENT_INPUT: replay_save_input_event(event->opaque); break;
            case REPLAY_ASYNC_EVENT_CHAR_READ: replay_event_char_read_save(event->opaque); break;
            case REPLAY_ASYNC_EVENT_BLOCK: replay_put_qword(event->id); break;
            case REPLAY_ASYNC_EVENT_NET: replay_event_net_save(event->opaque); break;
            ...
        }
        // 执行事件
        replay_run_event(event);
        QTAILQ_REMOVE(&events_list, event, events);
        g_free(event);
    }
}
```

### 9.4 事件派发

```c
// replay/replay-events.c:34-65
static void replay_run_event(Event *event) {
    switch (event->event_kind) {
    case REPLAY_ASYNC_EVENT_BH:
        aio_bh_call(event->opaque);              // 调用 BH 回调
        break;
    case REPLAY_ASYNC_EVENT_INPUT:
        qemu_input_event_send_impl(NULL, event->opaque);  // 注入输入
        qapi_free_InputEvent(event->opaque);
        break;
    case REPLAY_ASYNC_EVENT_CHAR_READ:
        replay_event_char_read_run(event->opaque); // 注入字符设备数据
        break;
    case REPLAY_ASYNC_EVENT_BLOCK:
        aio_bh_call(event->opaque);              // 唤醒块设备协程
        break;
    case REPLAY_ASYNC_EVENT_NET:
        replay_event_net_run(event->opaque);     // 注入网络包
        break;
    }
}
```

---

## 10. 时钟录制与回放

### 10.1 设计原理

虚拟时钟（Virtual Clock）由 icount 驱动，完全确定性，无需录制。需要录制的是：
- **Host Clock**: `clock_gettime(CLOCK_REALTIME)` 的返回值
- **Virtual RT Clock**: 用于 CPU idle 时的 warp 计算

### 10.2 保存时钟

```c
// replay/replay-time.c:17-32
int64_t replay_save_clock(ReplayClockKind kind, int64_t clock, int64_t raw_icount) {
    g_assert(replay_file && replay_mutex_locked());
    replay_advance_current_icount(raw_icount);  // 先记录到此刻的指令数
    replay_put_event(EVENT_CLOCK + kind);       // 写入时钟事件
    replay_put_qword(clock);                    // 写入时钟值
    return clock;
}
```

### 10.3 读取时钟

```c
// replay/replay-time.c:49-59
int64_t replay_read_clock(ReplayClockKind kind, int64_t raw_icount) {
    g_assert(replay_file && replay_mutex_locked());
    replay_advance_current_icount(raw_icount);
    if (replay_next_event_is(EVENT_CLOCK + kind)) {
        replay_read_next_clock(kind);  // 读取并缓存
    }
    return replay_state.cached_clock[kind];  // 返回缓存值
}
```

### 10.4 REPLAY_CLOCK 宏

```c
// include/system/replay.h:81-94
#define REPLAY_CLOCK(clock, value)                                      \
    !icount_enabled() ? (value) :                                       \
    (replay_mode == REPLAY_MODE_PLAY                                    \
        ? replay_read_clock((clock), icount_get_raw())                  \
        : replay_mode == REPLAY_MODE_RECORD                             \
            ? replay_save_clock((clock), (value), icount_get_raw())     \
            : (value))
```

设备模型中使用 `REPLAY_CLOCK(REPLAY_CLOCK_HOST, real_clock_value)` 替代直接读主机时钟。

---

## 11. 输入设备重放

### 11.1 输入事件分类

```c
// replay/replay-input.c 支持的输入类型:
INPUT_EVENT_KIND_KEY    // 键盘按键 (QCode/数字键码 + up/down)
INPUT_EVENT_KIND_BTN    // 鼠标按钮 (按钮ID + up/down)
INPUT_EVENT_KIND_REL    // 相对移动 (轴 + 值)
INPUT_EVENT_KIND_ABS    // 绝对位置 (轴 + 值)
INPUT_EVENT_KIND_MTT    // 多点触控 (类型+slot+tracking_id+轴+值)
```

### 11.2 录制流程

```c
// replay/replay-input.c:138-147
void replay_input_event(QemuConsole *src, InputEvent *evt) {
    if (replay_mode == REPLAY_MODE_PLAY) {
        // 忽略真实输入，等待日志注入
    } else if (replay_mode == REPLAY_MODE_RECORD) {
        replay_add_input_event(QAPI_CLONE(InputEvent, evt));
    } else {
        qemu_input_event_send_impl(src, evt);  // 直接传递
    }
}
```

### 11.3 序列化格式

以键盘事件为例：
```
4B: INPUT_EVENT_KIND_KEY
4B: KEY_VALUE_KIND_QCODE
4B: QKeyCode 值
1B: down (0/1)
```

以鼠标相对移动为例：
```
4B: INPUT_EVENT_KIND_REL
4B: InputAxis (X/Y)
8B: 移动值
```

### 11.4 Play 模式行为

- 来自真实设备的输入事件被**丢弃**（`REPLAY_MODE_PLAY` 时 `replay_input_event()` 什么都不做）
- 从日志读取的 INPUT 事件通过 `replay_run_event() → qemu_input_event_send_impl()` 注入

---

## 12. 字符设备重放

### 12.1 架构

字符设备（串口、控制台）具有双向通信：
- **后端→前端**（读/接收）: 外部数据到达 → 非确定性 → 需要录制
- **前端→后端**（写/发送）: guest 主动写 → 确定性，但完成状态需同步

### 12.2 驱动注册

```c
// replay/replay-char.c:41-49
void replay_register_char_driver(Chardev *chr) {
    if (replay_mode == REPLAY_MODE_NONE) return;
    char_drivers = g_realloc(char_drivers, sizeof(*char_drivers) * (drivers_count + 1));
    char_drivers[drivers_count++] = chr;
}
```

每个注册的字符设备获得一个数值 ID（数组索引），用于日志中标识设备。

### 12.3 后端写入录制

```c
// replay/replay-char.c:51-65
void replay_chr_be_write(Chardev *s, const uint8_t *buf, int len) {
    CharEvent *event = g_new0(CharEvent, 1);
    event->id = find_char_driver(s);
    event->buf = g_malloc(len);
    memcpy(event->buf, buf, len);
    event->len = len;
    replay_add_event(REPLAY_ASYNC_EVENT_CHAR_READ, event, NULL, 0);
}
```

### 12.4 前端写入同步

```c
// replay/replay-char.c:96-118
// RECORD: 保存 write 返回值
void replay_char_write_event_save(int res, int offset) {
    replay_save_instructions();
    replay_put_event(EVENT_CHAR_WRITE);
    replay_put_dword(res);      // 写入字节数/错误码
    replay_put_dword(offset);   // 缓冲区偏移
}

// PLAY: 读取 write 返回值
void replay_char_write_event_load(int *res, int *offset) {
    replay_account_executed_instructions();
    if (replay_next_event_is(EVENT_CHAR_WRITE)) {
        *res = replay_get_dword();
        *offset = replay_get_dword();
        replay_finish_event();
    } else {
        replay_sync_error("Missing character write event");
    }
}
```

---

## 13. 网络包重放

### 13.1 网络 Filter 架构

```
          Guest NIC
              │
    ┌─────────┼─────────┐
    │   NetFilterState   │  ← filter-replay
    │  (TYPE_FILTER_REPLAY)│
    └─────────┼─────────┘
              │
         Network Backend
        (tap/socket/user)
```

### 13.2 Filter 实现

```c
// net/filter-replay.c:32-53
static ssize_t filter_replay_receive_iov(NetFilterState *nf,
                                          NetClientState *sndr,
                                          unsigned flags,
                                          const struct iovec *iov,
                                          int iovcnt, ...) {
    switch (replay_mode) {
    case REPLAY_MODE_RECORD:
        if (nf->netdev == sndr) {
            // 只记录从后端发来的包（外部输入）
            replay_net_packet_event(nfrs->rns, flags, iov, iovcnt);
            return iov_size(iov, iovcnt);  // 消费包
        }
        return 0;  // guest 发出的包正常通过
    case REPLAY_MODE_PLAY:
        // 丢弃所有真实网络包，replay 模块会从日志注入
        return iov_size(iov, iovcnt);
    default:
        return 0;  // 无 replay，正常通过
    }
}
```

### 13.3 包录制

```c
// replay/replay-net.c:53-64
void replay_net_packet_event(ReplayNetState *rns, unsigned flags,
                             const struct iovec *iov, int iovcnt) {
    NetEvent *event = g_new(NetEvent, 1);
    event->flags = flags;
    event->data = g_malloc(iov_size(iov, iovcnt));
    event->size = iov_size(iov, iovcnt);
    event->id = rns->id;               // 网卡 filter ID
    iov_to_buf(iov, iovcnt, 0, event->data, event->size);
    replay_add_event(REPLAY_ASYNC_EVENT_NET, event, NULL, 0);
}
```

### 13.4 包回放

```c
// replay/replay-net.c:66-81
void replay_event_net_run(void *opaque) {
    NetEvent *event = opaque;
    struct iovec iov = { .iov_base = event->data, .iov_len = event->size };
    // 将包注入到对应 filter 的下游
    qemu_netfilter_pass_to_next(network_filters[event->id]->netdev,
        event->flags, &iov, 1, network_filters[event->id]);
    g_free(event->data);
    g_free(event);
}
```

### 13.5 序列化格式

```
1B: network_filter_id (0-255)
4B: flags
4B: packet_length
NB: packet_data
```

---

## 14. 块设备重放

### 14.1 blkreplay Filter Driver

```c
// block/blkreplay.c:143-160
static BlockDriver bdrv_blkreplay = {
    .format_name        = "blkreplay",
    .is_filter          = true,
    .bdrv_co_preadv     = blkreplay_co_preadv,
    .bdrv_co_pwritev    = blkreplay_co_pwritev,
    .bdrv_co_pwrite_zeroes = blkreplay_co_pwrite_zeroes,
    .bdrv_co_pdiscard   = blkreplay_co_pdiscard,
    .bdrv_co_flush      = blkreplay_co_flush,
    .bdrv_snapshot_goto = blkreplay_snapshot_goto,
};
```

### 14.2 确定性保证机制

块 I/O 的内容是确定性的（从相同磁盘镜像读写），但**完成顺序**是非确定性的。blkreplay 通过 request ID 保证完成顺序一致：

```c
// block/blkreplay.c:74-84
static int coroutine_fn blkreplay_co_preadv(...) {
    uint64_t reqid = blkreplay_next_id();         // 分配递增 ID
    int ret = bdrv_co_preadv(bs->file, ...);     // 实际 I/O
    block_request_create(reqid, bs, qemu_coroutine_self());  // 注册回调
    qemu_coroutine_yield();                       // 等待确定性唤醒
    return ret;
}
```

### 14.3 确定性唤醒

```c
// block/blkreplay.c:62-72
static void block_request_create(uint64_t reqid, BlockDriverState *bs, Coroutine *co) {
    Request *req = g_new(Request, 1);
    AioContext *ctx = qemu_coroutine_get_aio_context(co);
    req->co = co;
    req->bh = aio_bh_new(ctx, blkreplay_bh_cb, req);
    replay_block_event(req->bh, reqid);  // 将 BH 加入异步事件队列
}
```

- RECORD 模式：BH 加入队列，在 `replay_save_events()` 时按入队顺序执行并记录 reqid
- PLAY 模式：从日志读取 reqid，匹配到对应协程并唤醒

### 14.4 快照支持

```c
// block/blkreplay.c:131-141
static int blkreplay_snapshot_goto(BlockDriverState *bs, const char *snapshot_id) {
    return bdrv_snapshot_goto(bs->file->bs, snapshot_id, NULL);
}
```

直接转发到底层镜像的快照实现，支持反向调试时的状态回退。

---

## 15. 快照与 VMState 集成

### 15.1 VMState 定义

```c
// replay/replay-snapshot.c:48-66
static const VMStateDescription vmstate_replay = {
    .name = "replay",
    .version_id = 3,
    .pre_save = replay_pre_save,
    .post_load = replay_post_load,
    .fields = (const VMStateField[]) {
        VMSTATE_INT64_ARRAY(cached_clock, ReplayState, REPLAY_CLOCK_COUNT),
        VMSTATE_UINT64(current_icount, ReplayState),
        VMSTATE_INT32(instruction_count, ReplayState),
        VMSTATE_UINT32(current_event, ReplayState),
        VMSTATE_UINT32(data_kind, ReplayState),
        VMSTATE_BOOL(has_unread_data, ReplayState),
        VMSTATE_UINT64(file_offset, ReplayState),       // 关键！日志偏移
        VMSTATE_UINT64(block_request_id, ReplayState),
        VMSTATE_UINT64(read_event_id, ReplayState),
        VMSTATE_END_OF_LIST()
    },
};
```

### 15.2 保存前回调

```c
// replay/replay-snapshot.c:22-28
static int replay_pre_save(void *opaque) {
    ReplayState *state = opaque;
    state->file_offset = ftell(replay_file);  // 记录当前日志位置
    return 0;
}
```

### 15.3 加载后回调

```c
// replay/replay-snapshot.c:30-46
static int replay_post_load(void *opaque, int version_id) {
    ReplayState *state = opaque;
    if (replay_mode == REPLAY_MODE_PLAY) {
        fseek(replay_file, state->file_offset, SEEK_SET);  // 恢复日志位置
        replay_fetch_data_kind();  // 重新读取当前事件
    } else if (replay_mode == REPLAY_MODE_RECORD) {
        // 仅用于加载初始状态，重置计数器
        state->instruction_count = 0;
        state->block_request_id = 0;
    }
    return 0;
}
```

### 15.4 初始快照

```c
// replay/replay-snapshot.c:73-93
void replay_vmstate_init(void) {
    if (replay_snapshot) {
        if (replay_mode == REPLAY_MODE_RECORD) {
            save_snapshot(replay_snapshot, ...);  // 录制开始时保存初始快照
        } else if (replay_mode == REPLAY_MODE_PLAY) {
            load_snapshot(replay_snapshot, ...);  // 回放开始时加载初始快照
        }
    }
}
```

### 15.5 快照可行性检查

```c
// replay/replay-snapshot.c:95-99
bool replay_can_snapshot(void) {
    return replay_mode == REPLAY_MODE_NONE || !replay_has_events();
}
```

只在事件队列为空时允许快照，避免丢失未处理的事件。

---

## 16. 反向调试原理

### 16.1 实现机制

反向调试 = **快照 + Replay 日志定位**

```
时间轴:  [快照A]─────────────────[当前位置]
日志:     |... events ... events ...|

反向执行:
1. 加载快照 A (回到早期状态)
2. fseek(replay_file, snapshot_A.file_offset)
3. 从快照 A 开始正向重放
4. 在目标指令数处停止
```

### 16.2 Replay 断点

```c
// replay/replay.c:38-39
uint64_t replay_break_icount = -1ULL;  // 断点指令数
QEMUTimer *replay_break_timer;         // 断点触发回调

// 在 replay_advance_current_icount() 中检查:
if (replay_break_icount == replay_state.current_icount) {
    timer_mod_ns(replay_break_timer, qemu_clock_get_ns(QEMU_CLOCK_REALTIME));
}
```

### 16.3 GDB 反向调试流程

```
用户输入: (gdb) reverse-step

GDB Server 处理:
1. 确定目标 icount = current_icount - delta
2. 找到最近的快照 (icount < 目标)
3. load_snapshot() → 恢复 VM 状态 + 日志偏移
4. 设置 replay_break_icount = 目标 icount
5. 恢复 VM 运行
6. 到达断点 → 暂停并报告给 GDB
```

### 16.4 限制

- 每次反向操作需要从最近快照正向重放到目标位置
- 快照间隔越小，反向操作越快，但存储开销越大
- 不支持反向跨越快照未覆盖的区域

---

## 17. 音频重放

### 17.1 API

```c
// include/system/replay.h:166-176
void replay_audio_out(size_t *played);           // 输出采样数
void replay_audio_in_start(size_t *recorded);    // 输入开始
void replay_audio_in_sample_lr(uint64_t *left, uint64_t *right); // 采样值
void replay_audio_in_finish(void);               // 输入结束
```

### 17.2 原理

- 音频输出的 played 采样数是非确定性的（依赖主机音频缓冲区状态）
- 音频输入的采样数据完全是外部输入
- 通过 EVENT_AUDIO_OUT/IN 记录这些值

---

## 18. 随机数重放

### 18.1 作用

guest 通过 virtio-rng 或其他随机设备获取的随机数是非确定性的。

### 18.2 机制

- RECORD：从主机随机源读取后，通过 EVENT_RANDOM 记录到日志
- PLAY：忽略主机随机源，从日志读取相同的随机数据

---

## 19. 限制与 Blocker 机制

### 19.1 Blocker 注册

```c
// replay/replay.c:493-500
void replay_add_blocker(const char *feature) {
    Error *reason = NULL;
    error_setg(&reason, "Record/replay is not supported with %s", feature);
    replay_blockers = g_slist_prepend(replay_blockers, reason);
}
```

在 `replay_start()` 中检查：如果存在 blocker，报错退出。

### 19.2 已知不兼容特性

- KVM 加速器（仅 TCG 支持）
- 某些网络后端（如 vhost-net）
- 直通设备（PCI passthrough）
- 多 vCPU（replay 强制串行）
- 某些块设备后端配置

### 19.3 设备模型要求

| 要求 | 说明 |
|------|------|
| 使用虚拟时钟 | 影响 guest 状态的定时器必须用 QEMU_CLOCK_VIRTUAL |
| 确定性初始化 | 设备状态只依赖参数，不依赖主机环境 |
| 完整 VMState | 所有字段必须可保存/恢复 |
| 无跨设备依赖 | post_load 不调用其他设备的函数 |
| 使用 replay BH | 影响 guest 状态的 BH 通过 replay_bh_schedule_event |

---

## 20. 与硬件实际行为对比

### 20.1 Replay 是纯软件概念

真实硬件没有 "record/replay" 概念。这是 QEMU 特有的调试/分析功能。

### 20.2 确定性差异分析

| 方面 | 真实硬件 | QEMU Replay |
|------|---------|-------------|
| 中断时序 | 取决于物理时钟和总线仲裁 | 由 icount 精确控制，指令边界注入 |
| 网络包到达 | 物理层 + OS 协议栈延迟 | 以指令数为时间标记重现 |
| 磁盘完成 | 取决于磁盘物理特性 | reqid 序列化，与物理延迟无关 |
| 时钟读取 | 物理时钟单调递增 | 从日志回放缓存值 |
| 多核 | 真正并行 | 串行化（replay_mutex 强制） |

### 20.3 ARM64 特有考虑

- **Generic Timer (CNTPCT_EL0)**: 由 icount 驱动的虚拟时钟直接模拟，无需 replay 录制
- **GIC 中断传递**: EVENT_INTERRUPT 同步中断注入时刻
- **PSCI/SMC**: 确定性操作，不需要 replay 特殊处理
- **MMIO 读外部设备**: 如果设备使用 host clock，需通过 REPLAY_CLOCK 宏包裹

### 20.4 精度限制

- 指令级精度：事件定位到指令边界（非周期精确）
- TB 粒度退出：实际精度受 TB 大小限制（通常 < 几百条指令）
- 无微架构模拟：不记录流水线状态、缓存命中/缺失

---

## 21. 使用示例与命令行

### 21.1 录制会话

```bash
qemu-system-aarch64 \
    -M virt -cpu cortex-a57 -m 1G \
    -icount shift=auto,rr=record,rrfile=replay.bin,rrsnapshot=init \
    -drive file=disk.qcow2,if=none,id=disk0 \
    -drive driver=blkreplay,if=none,image.driver=qcow2,image.file.filename=disk.qcow2,id=disk0-rr \
    -device virtio-blk-pci,drive=disk0-rr \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0 \
    -object filter-replay,id=replay-net0,netdev=net0
```

### 21.2 回放会话

```bash
qemu-system-aarch64 \
    -M virt -cpu cortex-a57 -m 1G \
    -icount shift=auto,rr=replay,rrfile=replay.bin,rrsnapshot=init \
    -drive file=disk.qcow2,if=none,id=disk0 \
    -drive driver=blkreplay,if=none,image.driver=qcow2,image.file.filename=disk.qcow2,id=disk0-rr \
    -device virtio-blk-pci,drive=disk0-rr \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0 \
    -object filter-replay,id=replay-net0,netdev=net0 \
    -s -S  # GDB 调试端口
```

### 21.3 反向调试

```bash
# 启动 GDB
aarch64-linux-gnu-gdb vmlinux
(gdb) target remote :1234
(gdb) continue
# ... 运行到感兴趣的位置 ...
(gdb) reverse-step     # 反向单步
(gdb) reverse-continue # 反向继续到上一断点
```

### 21.4 关键参数

| 参数 | 说明 |
|------|------|
| `rr=record` | 录制模式 |
| `rr=replay` | 回放模式 |
| `rrfile=PATH` | 日志文件路径 |
| `rrsnapshot=NAME` | 初始快照名（推荐使用） |
| `shift=auto` | icount 自动调整 shift 值 |
| `driver=blkreplay` | 块设备必须用 blkreplay filter |
| `filter-replay` | 网络必须配置 filter-replay |

---

## 附录 A: 完整事件流示例

```
录制日志内容（概念表示）:

[HEADER: version=0xe0200d, reserved=0]
[EVENT_CHECKPOINT: INIT]
[EVENT_INSTRUCTION: count=1523]
[EVENT_CLOCK+HOST: value=1704067200000000000]
[EVENT_INSTRUCTION: count=847]
[EVENT_CHECKPOINT: CLOCK_VIRTUAL]
[EVENT_ASYNC+NET: id=0, flags=0, len=64, data=...]
[EVENT_INSTRUCTION: count=2091]
[EVENT_ASYNC+INPUT: KEY, QCODE=28(Return), down=1]
[EVENT_INSTRUCTION: count=156]
[EVENT_ASYNC+INPUT_SYNC]
[EVENT_INSTRUCTION: count=3200]
[EVENT_ASYNC+BLOCK: reqid=1]
[EVENT_ASYNC+BLOCK: reqid=2]
[EVENT_INSTRUCTION: count=500]
[EVENT_CHECKPOINT: CLOCK_WARP_START]
[EVENT_CLOCK+VIRTUAL_RT: value=...]
[EVENT_CHECKPOINT: CLOCK_WARP_ACCOUNT]
[EVENT_INSTRUCTION: count=10000]
[EVENT_SHUTDOWN: HOST_UI]
[EVENT_END]
```

---

## 附录 B: 源码文件索引

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| `replay/replay.c` | 500 | 主控制、checkpoint、async_events、configure/start/finish |
| `replay/replay-internal.h` | 217 | ReplayState、事件枚举、内部 API 声明 |
| `replay/replay-internal.c` | 323 | 日志 I/O、互斥锁、icount 推进、fetch_data_kind |
| `replay/replay-events.c` | ~260 | 异步事件队列、save/read/run_event |
| `replay/replay-time.c` | 59 | 时钟保存/读取 |
| `replay/replay-input.c` | 158 | 输入设备序列化/反序列化 |
| `replay/replay-char.c` | 157 | 字符设备录制/回放 |
| `replay/replay-net.c` | 101 | 网络包录制/回放 |
| `replay/replay-snapshot.c` | 99 | VMState 集成、初始快照 |
| `block/blkreplay.c` | 167 | 块设备 filter、协程确定性唤醒 |
| `net/filter-replay.c` | 89 | 网络 filter QOM 对象 |
| `include/system/replay.h` | 187 | 公共 API、宏、枚举 |
| `docs/devel/replay.rst` | 310 | 官方开发文档 |

---

## 附录 C: 与 Doc 93 的关系

| 主题 | Doc 93 覆盖 | Doc 95 扩展 |
|------|------------|------------|
| icount 基础 | ✅ 完整 | 引用 |
| TB 预算 | ✅ 完整 | 引用 |
| replay_get_instructions | ✅ 基本 | 详解实现细节 |
| 事件日志格式 | ❌ 未覆盖 | ✅ 完整 |
| 块设备 replay | ❌ 未覆盖 | ✅ 完整 |
| 网络 replay | ❌ 未覆盖 | ✅ 完整 |
| 字符设备 replay | ❌ 未覆盖 | ✅ 完整 |
| 快照集成 | ❌ 未覆盖 | ✅ 完整 |
| 反向调试 | ❌ 未覆盖 | ✅ 完整 |
| 互斥锁设计 | 简要提及 | ✅ 深入分析 |
| 音频/随机数 | ❌ 未覆盖 | ✅ 覆盖 |
