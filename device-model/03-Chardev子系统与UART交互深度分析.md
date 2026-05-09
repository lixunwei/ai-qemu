# Chardev 子系统与 UART 交互深度分析

> QEMU 版本：11.0.50  
> 分析日期：2025年  
> 目标平台：ARM virt machine（AArch64）  
> 重点设备：PL011 UART  
> 相关文档：[../architecture/01-QOM对象模型深度分析](../architecture/01-QOM对象模型深度分析.md) | [01-关键设备仿真分析-UART-磁盘-网卡](01-关键设备仿真分析-UART-磁盘-网卡.md)

---

## 目录

- [第一部分：Chardev 框架总览](#第一部分chardev-框架总览)
  - [§1 Chardev 架构概述](#1-chardev-架构概述)
  - [§2 Chardev QOM 类型层次](#2-chardev-qom-类型层次)
  - [§3 核心数据结构](#3-核心数据结构)
- [第二部分：Chardev 生命周期](#第二部分chardev-生命周期)
  - [§4 类型注册](#4-类型注册)
  - [§5 后端创建 (-serial / -chardev)](#5-后端创建--serial---chardev)
  - [§6 前端连接与回调安装](#6-前端连接与回调安装)
  - [§7 Open/Close 事件](#7-openclose-事件)
  - [§8 销毁与清理](#8-销毁与清理)
- [第三部分：前端 ↔ 后端 API](#第三部分前端--后端-api)
  - [§9 发送方向：Frontend → Backend (写)](#9-发送方向frontend--backend-写)
  - [§10 接收方向：Backend → Frontend (读)](#10-接收方向backend--frontend-读)
  - [§11 流控机制](#11-流控机制)
  - [§12 事件循环集成 (GSource/fd watch)](#12-事件循环集成-gsourcefd-watch)
- [第四部分：Chardev 后端实现](#第四部分chardev-后端实现)
  - [§13 stdio 后端](#13-stdio-后端)
  - [§14 socket 后端](#14-socket-后端)
  - [§15 pty 后端](#15-pty-后端)
  - [§16 file 后端](#16-file-后端)
  - [§17 null 后端](#17-null-后端)
  - [§18 pipe / serial / 其他后端](#18-pipe--serial--其他后端)
  - [§19 mux 后端 (多路复用)](#19-mux-后端-多路复用)
- [第五部分：PL011 UART 设备](#第五部分pl011-uart-设备)
  - [§20 PL011 设备状态结构](#20-pl011-设备状态结构)
  - [§21 寄存器映射表](#21-寄存器映射表)
  - [§22 FIFO 实现](#22-fifo-实现)
  - [§23 实例初始化与 Realize](#23-实例初始化与-realize)
  - [§24 中断机制](#24-中断机制)
- [第六部分：PL011 ↔ Chardev 数据路径](#第六部分pl011--chardev-数据路径)
  - [§25 TX 路径：Guest → Host](#25-tx-路径guest--host)
  - [§26 RX 路径：Host → Guest](#26-rx-路径host--guest)
  - [§27 事件处理 (Break / Open)](#27-事件处理-break--open)
  - [§28 流控与 FIFO 溢出](#28-流控与-fifo-溢出)
- [第七部分：端到端数据流](#第七部分端到端数据流)
  - [§29 完整 TX 时序：Guest printf → 终端显示](#29-完整-tx-时序guest-printf--终端显示)
  - [§30 完整 RX 时序：用户按键 → Guest 读取](#30-完整-rx-时序用户按键--guest-读取)
  - [§31 -serial stdio 完整连接链](#31--serial-stdio-完整连接链)
- [附录](#附录)
  - [附录A：Chardev 后端一览表](#附录achardev-后端一览表)
  - [附录B：关键源文件索引](#附录b关键源文件索引)
  - [附录C：关键 Git 提交](#附录c关键-git-提交)

---

## 第一部分：Chardev 框架总览

### §1 Chardev 架构概述

QEMU 的 Chardev（字符设备）子系统是**设备模拟与外部 I/O** 之间的抽象层。它将设备前端（如 UART、virtio-console）与 I/O 后端（如 stdio、socket、pty）解耦：

```
┌─────────────────────────────────────────────────┐
│                   Guest VM                       │
│  ┌─────────┐    ┌─────────┐    ┌──────────────┐ │
│  │ Console │    │ Serial  │    │ virtio-serial│ │
│  │ Driver  │    │ Driver  │    │ Driver       │ │
│  └────┬────┘    └────┬────┘    └──────┬───────┘ │
└───────┼──────────────┼────────────────┼─────────┘
     MMIO           MMIO            virtqueue
        │              │                │
┌───────┼──────────────┼────────────────┼─────────┐
│ QEMU  │              │                │         │
│  ┌────▼────┐    ┌────▼────┐    ┌──────▼───────┐ │
│  │ PL011   │    │ 16550   │    │ virtio-      │ │
│  │ Device  │    │ Device  │    │ console Dev  │ │
│  │ (前端)  │    │ (前端)  │    │ (前端)       │ │
│  └────┬────┘    └────┬────┘    └──────┬───────┘ │
│       │              │                │         │
│  ┌────▼────┐    ┌────▼────┐    ┌──────▼───────┐ │
│  │CharBack-│    │CharBack-│    │CharBackend   │ │
│  │end (FE) │    │end (FE) │    │(FE)          │ │
│  └────┬────┘    └────┬────┘    └──────┬───────┘ │
│       └──────────────┴────────────────┘         │
│                      │                          │
│              ┌───────▼────────┐                 │
│              │   Chardev Core │                 │
│              │   (char.c)     │                 │
│              └───────┬────────┘                 │
│                      │                          │
│       ┌──────┬───────┼───────┬──────┐          │
│  ┌────▼──┐ ┌─▼───┐ ┌─▼───┐ ┌▼────┐ ┌▼────┐   │
│  │ stdio │ │ pty │ │sock-│ │file │ │null │   │
│  │       │ │     │ │ et  │ │     │ │     │   │
│  └───┬───┘ └──┬──┘ └──┬──┘ └──┬──┘ └─────┘   │
└──────┼────────┼───────┼───────┼────────────────┘
       │        │       │       │
    终端/TTY  PTY 对  TCP/Unix  文件
```

### §2 Chardev QOM 类型层次

**源文件**：`char.h:231-247`，`char.c:356-365`

```
TYPE_OBJECT
  └─ TYPE_CHARDEV ("chardev")               [char.c:356, 抽象基类]
       ├─ TYPE_CHARDEV_NULL    ("chardev-null")     [char-null.c]
       ├─ TYPE_CHARDEV_MUX     ("chardev-mux")      [char-mux.c]
       ├─ TYPE_CHARDEV_HUB     ("chardev-hub")       [char-hub.c]
       ├─ TYPE_CHARDEV_RINGBUF ("chardev-ringbuf")   [char-ringbuf.c]
       ├─ TYPE_CHARDEV_FD      ("chardev-fd")        [char-fd.c, 抽象]
       │    ├─ TYPE_CHARDEV_STDIO  ("chardev-stdio")  [char-stdio.c]
       │    ├─ TYPE_CHARDEV_PIPE   ("chardev-pipe")   [char-pipe.c]
       │    ├─ TYPE_CHARDEV_SERIAL ("chardev-serial")  [char-serial.c]
       │    └─ TYPE_CHARDEV_FILE   ("chardev-file")    [char-file.c]
       ├─ TYPE_CHARDEV_PTY     ("chardev-pty")       [char-pty.c]
       ├─ TYPE_CHARDEV_SOCKET  ("chardev-socket")    [char-socket.c]
       ├─ TYPE_CHARDEV_UDP     ("chardev-udp")       [char-udp.c]
       ├─ TYPE_CHARDEV_MEMORY  ("chardev-memory")    [char-ringbuf.c]
       ├─ TYPE_CHARDEV_CONSOLE ("chardev-console")   [char-console.c]
       └─ TYPE_CHARDEV_PARALLEL("chardev-parallel")  [char-parallel.c]
```

### §3 核心数据结构

**Chardev（后端基类）**

**源文件**：`char.h:231-316`

```c
// char.h:265-316 (简化)
struct Chardev {
    Object parent_obj;

    QemuMutex chr_write_lock;     // 写操作互斥锁
    CharBackend *be;              // 指向前端的反向指针
    char *label;                  // 名称标识 (如 "serial0")
    char *filename;               // 描述字符串
    int logfd;                    // 日志文件描述符
    int be_open;                  // 后端是否已打开
    // ... 其他字段
};
```

**ChardevClass（后端虚函数表）**

**源文件**：`char.h:120-179`

```c
// char.h:120-179 (简化)
struct ChardevClass {
    ObjectClass parent_class;

    // 核心接口
    bool (*chr_open)(Chardev *chr, ...);     // 打开后端
    int  (*chr_write)(Chardev *s, const uint8_t *buf, int len);  // 写数据
    void (*chr_accept_input)(Chardev *chr);  // 前端准备好接收更多数据
    void (*chr_set_echo)(Chardev *chr, bool echo);  // 控制回显
    void (*chr_set_fe_open)(Chardev *chr, int fe_open);  // 前端打开/关闭
    void (*chr_be_event)(Chardev *s, QEMUChrEvent event); // 后端事件
    void (*chr_update_read_handler)(Chardev *s);  // 更新读处理器

    // 连接管理
    int  (*chr_add_client)(Chardev *chr, int fd);  // 添加客户端
    GSource *(*chr_add_watch)(Chardev *s, GIOCondition cond); // 添加 watch

    // 解析与创建
    void (*chr_parse)(QemuOpts *opts, ChardevBackend *backend, ...);
    // ...
};
```

**CharBackend / CharFrontend（前端接口）**

**源文件**：`char-fe.h:11-25`

```c
// char-fe.h:11-25
typedef struct CharBackend {  // 实际是"前端"，名称历史遗留
    Chardev *chr;                          // 指向后端
    IOEventHandler *chr_event;             // 事件回调
    IOCanReadHandler *chr_can_read;         // 查询可接收字节数
    IOReadHandler *chr_read;               // 数据接收回调
    BackendChangeHandler *chr_be_change;   // 后端变更通知
    void *opaque;                          // 回调上下文 (通常是设备 state)
    int tag;                               // GSource tag
    bool fe_is_open;                       // 前端是否已打开
} CharBackend;
```

> **命名说明**：结构名 `CharBackend` 实际代表的是"前端"（设备侧），这是历史遗留命名。在代码注释和新代码中也称为 `CharFrontend`。

---

## 第二部分：Chardev 生命周期

### §4 类型注册

**源文件**：`char.c:1331-1336`

```
// 基类注册
type_init(register_types)                    [char.c:1336]
  └─ type_register_static(&char_type_info)   [char.c:1331]
       // TYPE_CHARDEV, abstract, class_init = char_class_init

// 子类注册 (各后端文件)
type_init(register_types)                    [char-stdio.c, char-socket.c, ...]
  └─ type_register_static(&char_stdio_type_info)
       // parent = TYPE_CHARDEV_FD, class_init = ...
```

基类 `class_init` 设置默认虚函数：

```c
// char.c:333-340
static void char_class_init(ObjectClass *oc, void *data)
{
    ChardevClass *cc = CHARDEV_CLASS(oc);
    cc->chr_write = null_chr_write;     // 默认：丢弃写入数据
    cc->chr_be_event = chr_be_event;    // 默认：更新 be_open 标志
}
```

### §5 后端创建 (-serial / -chardev)

#### 5.1 -serial 命令行路径

**源文件**：`vl.c:188-189, 1459-1489`

```
命令行: qemu-system-aarch64 -serial stdio

解析流程:
  vl.c: foreach_device_config(DEV_SERIAL, serial_parse)
    └─ serial_parse(opts)                    [vl.c:1459]
         ├─ serial_hds[index] = qemu_chr_new_mux_mon(label, devname, ...)
         │    // label = "serial0", devname = "stdio"
         │    [vl.c:1470-1475]
         │
         │  实际创建:
         │  qemu_chr_new_mux_mon()
         │    └─ qemu_chr_new_noreplay()     [char.c:756]
         │         └─ qemu_chr_new_from_name() [char.c:790]
         │              ├─ qemu_chr_parse_compat("stdio", opts)
         │              │    // 将简短名 "stdio" 转为 ChardevBackend
         │              └─ qemu_chardev_new(label, backend)
         │                   └─ chardev_new(label, TYPE_CHARDEV_STDIO, ...)
         │                        [char.c:1044]
         │
         └─ num_serial_hds++

serial_hd(i) 获取:
  serial_hd(0) → serial_hds[0] → Chardev* (stdio 后端)
```

#### 5.2 -chardev 命令行路径

```
命令行: -chardev socket,id=chr0,host=...,port=...

解析流程:
  qemu_init() → chardev_init_func()
    └─ qemu_chardev_add(id, backend) → qmp_chardev_add()
         └─ chardev_new(id, TYPE_CHARDEV_SOCKET, ...)  [char.c:1098]
```

#### 5.3 chardev_new() 核心创建流程

**源文件**：`char.c:1044-1095`

```
chardev_new(id, type, backend, context, errp)
  │
  ├─ obj = object_new(type)              QOM 实例化
  │    → instance_init()
  │
  ├─ chr = CHARDEV(obj)
  ├─ chr->label = id
  │
  ├─ cc->chr_open(chr, backend, ...)     打开后端
  │    // stdio: stdio_chr_open()
  │    // socket: tcp_chr_open()
  │    // pty: pty_chr_open()
  │
  ├─ object_property_add_child(container, id, obj)
  │    // 注册到全局 chardevs 容器
  │
  └─ return chr
```

### §6 前端连接与回调安装

**源文件**：`char-fe.c:192-215, 242-300`

```
// 步骤1: 初始化前端链接
qemu_chr_fe_init(CharBackend *b, Chardev *s, ...)  [char-fe.c:192]
  ├─ b->chr = s           前端记录后端指针
  └─ s->be = b            后端记录前端指针 (反向链接)

// 步骤2: 安装回调 (设备 realize 时调用)
qemu_chr_fe_set_handlers(b, can_read, read, event, be_change,
                          opaque, context, set_open)
  └─ qemu_chr_fe_set_handlers_full()      [char-fe.c:242]
       │
       ├─ b->chr_can_read = can_read       安装"可读"查询回调
       ├─ b->chr_read = read               安装数据接收回调
       ├─ b->chr_event = event             安装事件回调
       ├─ b->chr_be_change = be_change     安装后端变更回调
       ├─ b->opaque = opaque               回调上下文
       │
       ├─ qemu_chr_be_update_read_handlers()  通知后端更新读处理器
       │    → cc->chr_update_read_handler()
       │    // stdio/fd: 重新安装 fd poll watch
       │    // socket: 重新安装 IO watch
       │
       ├─ if (set_open) qemu_chr_fe_set_open(b, true)
       │    → 通知后端前端已打开
       │
       └─ if (s->be_open) → 触发 CHR_EVENT_OPENED 回调
            // 如果后端已经打开，立即通知前端
```

PL011 的调用：

```c
// pl011.c:663-669
static void pl011_realize(DeviceState *dev, Error **errp)
{
    PL011State *s = PL011(dev);
    qemu_chr_fe_set_handlers(&s->chr,
                              pl011_can_receive,   // 查询 FIFO 可用空间
                              pl011_receive,       // 接收数据
                              pl011_event,         // 事件 (BREAK 等)
                              NULL,                // be_change
                              s,                   // opaque = PL011State
                              NULL,                // context (主线程)
                              true);               // set_open = true
}
```

### §7 Open/Close 事件

**源文件**：`char.c:54-83`

```
后端事件分发:
  qemu_chr_be_event(chr, event)
    └─ cc->chr_be_event(chr, event)          [char.c:54]
         └─ chr_be_event()                   [char.c:62-83]
              ├─ CHR_EVENT_OPENED:
              │    chr->be_open = 1
              │    → 前端 chr_event 回调 (如 pl011_event)
              │
              ├─ CHR_EVENT_CLOSED:
              │    chr->be_open = 0
              │    → 前端 chr_event 回调
              │
              └─ CHR_EVENT_BREAK:
                   → 前端 chr_event 回调
```

### §8 销毁与清理

**源文件**：`char.c:342-354, 1258-1277`

```
销毁路径:
  qmp_chardev_remove(id)                 [char.c:1258]
    └─ object_unparent(OBJECT(chr))
         └─ char_finalize()              [char.c:342]
              ├─ chr_be_event(CLOSED)
              ├─ 释放 label/filename
              ├─ 关闭 logfd
              └─ mutex 销毁

全局清理:
  qemu_chr_cleanup()                     [char.c:1326]
    └─ 遍历所有注册 chardev 并销毁
```

---

## 第三部分：前端 ↔ 后端 API

### §9 发送方向：Frontend → Backend (写)

**源文件**：`char-fe.c:33-53`，`char.c:144-193`

```
PL011 发送数据:
  qemu_chr_fe_write_all(&s->chr, &data, 1)  [char-fe.c:44]
    │
    └─ qemu_chr_write(chr, buf, len, true)   [char.c:195]
         └─ qemu_chr_write_buffer()          [char.c:144]
              │
              ├─ qemu_mutex_lock(&chr->chr_write_lock)  加锁
              │
              ├─ cc->chr_write(chr, buf, len)  调用后端写
              │    │
              │    ├─ stdio: fd_chr_write()   [char-fd.c:36]
              │    │    └─ io_channel_send(ioc_out, buf, len)
              │    │         └─ write(stdout_fd, buf, len)
              │    │
              │    ├─ socket: tcp_chr_write()  [char-socket.c:108]
              │    │    └─ io_channel_send_full()
              │    │
              │    └─ null: null_chr_write()   [char.c:329]
              │         └─ return len  (丢弃)
              │
              ├─ if (write_all && ret < len)
              │    → 重试写入 (处理 EAGAIN)
              │
              └─ qemu_mutex_unlock()
```

**关键点**：
- `qemu_chr_fe_write()` — 非阻塞，可能部分写入
- `qemu_chr_fe_write_all()` — 阻塞直到全部写完（PL011 使用此版本）
- 写操作在 `chr_write_lock` 互斥锁保护下执行

### §10 接收方向：Backend → Frontend (读)

**源文件**：`char.c:231-260`

```
后端收到数据后:
  qemu_chr_be_write(chr, buf, len)       [char.c:242]
    └─ qemu_chr_be_write_impl(chr, buf, len) [char.c:249]
         │
         ├─ 检查前端是否已连接 (chr->be)
         │
         └─ chr->be->chr_read(chr->be->opaque, buf, len)
              │
              └─ pl011_receive(opaque, buf, size) [pl011.c:526]
                   // 将数据放入 RX FIFO
                   // 触发中断

但数据从何处来？由各后端的 fd/IO 回调触发:

stdio/fd 后端:
  main_loop_wait() → GLib poll → fd 可读
    → fd_chr_read()                      [char-fd.c:64]
       ├─ s->max_size = qemu_chr_be_can_write(chr)
       │    → chr->be->chr_can_read(opaque)
       │    → pl011_can_receive(s) = fifo_depth - read_count
       │
       ├─ qio_channel_read(ioc_in, buf, max_size)
       │    → read(stdin_fd, buf, max_size)
       │
       └─ qemu_chr_be_write(chr, buf, bytes_read)
            → pl011_receive()
```

### §11 流控机制

```
流控协议 (后端 → 前端):

1. 后端准备接收数据前:
   qemu_chr_be_can_write(chr)                [char.c:231]
     → chr->be->chr_can_read(opaque)
     → pl011_can_receive(s)                  [pl011.c:506]
        return fifo_depth - read_count       // FIFO 剩余空间

2. 如果返回 0: 后端不再读取，暂停 fd watch

3. 前端消费数据后 (Guest 读 UARTDR):
   qemu_chr_fe_accept_input(&s->chr)         [char-fe.c:149]
     → cc->chr_accept_input(chr)
     // fd 后端: 重新检查 can_write，恢复 fd watch
     // socket 后端: 类似逻辑

4. 后端再次调用 qemu_chr_be_can_write()
   → FIFO 有空间 → 继续读取 → qemu_chr_be_write()

流控循环:
  ┌─────────────────────────────────────────┐
  │ 后端 poll fd                            │
  │   ↓                                    │
  │ can_write() → pl011_can_receive()      │
  │   ↓                                    │
  │ 有空间? ──否──→ 暂停 poll (不读取)      │
  │   ↓是                                  │
  │ read(fd) → be_write() → pl011_receive()│
  │   ↓                                    │
  │ FIFO 满? ──是──→ 暂停 poll             │
  │   ↓否                                  │
  │ 继续 poll                              │
  └─────────────────────────────────────────┘
        ↑
  Guest 读 UARTDR → FIFO 腾出空间
    → accept_input() → 恢复 poll
```

### §12 事件循环集成 (GSource/fd watch)

**源文件**：`char-io.c:44-75, 108-135`

```
io_add_watch_poll(chr, ioc, fd_can_read, fd_read, opaque)
  │                                        [char-io.c:108]
  │
  │  创建自定义 GSource:
  ├─ IOWatchPoll *iwp = g_source_new(&io_watch_poll_funcs, ...)
  │
  │  GSource 回调:
  ├─ prepare(): 调用 fd_can_read()
  │    → qemu_chr_be_can_write()
  │    → pl011_can_receive()
  │    如果返回 > 0:
  │      创建 QIOChannel watch (G_IO_IN)
  │      → 当 fd 可读时触发 dispatch
  │    如果返回 0:
  │      移除 watch → 不触发 dispatch
  │
  ├─ dispatch(): fd 可读时调用
  │    → fd_read()
  │    → fd_chr_read() (对于 fd 后端)
  │         → read + qemu_chr_be_write()
  │
  └─ 附加到 GMainContext (主线程或 IOThread)

关键函数调用链:
  main_loop_wait()
    → g_main_context_iteration()
       → GSource prepare/check/dispatch
          → io_watch_poll_prepare()  检查 can_read
          → io_watch_poll_dispatch()  读数据
             → fd_chr_read()         读 + 投递
```

---

## 第四部分：Chardev 后端实现

### §13 stdio 后端

**源文件**：`char-stdio.c:49-149`

```
stdio_chr_open()                         [char-stdio.c:88]
  ├─ 检查: 不允许 -daemonize, 不允许重复使用 stdio
  ├─ 保存终端原始状态: tcgetattr(0, &oldtty)
  ├─ 设置 stdin 非阻塞: fcntl(0, F_SETFL, O_NONBLOCK)
  ├─ qemu_chr_open_fd(chr, fd_in=0, fd_out=1)
  │    // 创建 QIOChannelFile 包装 stdin/stdout
  │    → fd_chr_class.chr_update_read_handler()
  │         → io_add_watch_poll() 安装读 watch
  ├─ atexit(term_exit)       退出时恢复终端
  ├─ signal(SIGCONT, term_stdio_handler)  恢复后重设终端
  └─ stdio_chr_set_echo(false)  关闭回显 (raw 模式)

终端设置 (raw 模式):
  stdio_chr_set_echo()                    [char-stdio.c:49]
    └─ tcsetattr(0, TCSANOW, &tty)
         cfmakeraw(&tty)        // 禁用所有终端处理
         tty.c_lflag &= ~ECHO   // 关闭回显
         tty.c_cc[VMIN] = 1     // 最少读 1 字节
         tty.c_cc[VTIME] = 0    // 无超时

数据流:
  写 (→ stdout): fd_chr_write() → io_channel_send(ioc_out)
  读 (← stdin):  io_add_watch_poll → fd_chr_read() → qemu_chr_be_write()
```

**继承关系**：`TYPE_CHARDEV_STDIO` → `TYPE_CHARDEV_FD` → `TYPE_CHARDEV`

stdio 后端继承自 `TYPE_CHARDEV_FD`，后者提供了基于文件描述符的通用读写实现（`char-fd.c`）。

### §14 socket 后端

**源文件**：`char-socket.c:108-1578`

```
命令行: -chardev socket,id=chr0,host=localhost,port=4444,server=on,wait=off

tcp_chr_open()                           [char-socket.c:1344]
  │
  ├─ 服务端模式 (server=on):
  │    qmp_chardev_open_socket_server()  [char-socket.c:1209]
  │    ├─ 创建 QIONetListener
  │    ├─ 绑定 + 监听
  │    └─ wait=on: 同步等待连接
  │       wait=off: 安装 accept 回调 (异步)
  │
  ├─ 客户端模式:
  │    qmp_chardev_open_socket_client()  [char-socket.c:1252]
  │    ├─ 同步连接 或 异步可重连连接
  │    └─ 重连: 使用 timer 定期重试
  │
  └─ 连接建立后:
       tcp_chr_connect()                  [char-socket.c:637]
       ├─ s->connected = true
       ├─ 安装 IO 读 handler
       └─ qemu_chr_be_event(OPENED)

断开处理:
  tcp_chr_disconnect()                    [char-socket.c:460]
    ├─ 移除 IO watch
    ├─ 关闭 QIOChannel
    ├─ qemu_chr_be_event(CLOSED)
    └─ 如果有 reconnect → 启动重连 timer

写: tcp_chr_write() → io_channel_send_full()  [char-socket.c:108]
读: tcp_chr_recv() → qio_channel_readv_full() [char-socket.c:277]
```

### §15 pty 后端

**源文件**：`char-pty.c:89-412`

```
命令行: -chardev pty,id=chr0

pty_chr_open()                           [char-pty.c:333]
  ├─ qemu_openpty_raw() → openpty()      创建 PTY 主从对
  ├─ 保存主端 fd → QIOChannelFile
  ├─ 关闭从端 fd (Guest 不直接使用)
  ├─ 打印: "char device redirected to /dev/pts/X"
  └─ 安装 timer 检测连接状态

连接检测:
  pty_chr_update_read_handler()          [char-pty.c:179]
    └─ pty_chr_state(chr, connected)
         ├─ poll(master_fd, POLLHUP) → 检测从端是否打开
         ├─ 有连接: 安装 io_add_watch_poll, 发送 OPENED
         └─ 无连接: 启动 timer 定期轮询

用户连接: screen /dev/pts/X
  → 从端打开 → HUP 消失 → timer 检测到 → OPENED → 开始 I/O
```

### §16 file 后端

**源文件**：`char-file.c:37-138`

```
命令行: -chardev file,id=chr0,path=/tmp/serial.log

char_file_open()                         [char-file.c:60]
  ├─ open(path, O_WRONLY|O_CREAT|O_TRUNC)
  ├─ qemu_chr_open_fd(chr, -1, out_fd)   只有输出 fd
  └─ qemu_chr_be_event(OPENED)

特点:
  - 仅输出，不可读取
  - 适用于日志/调试输出
  - 写操作使用 fd_chr_write() → write(fd, ...)
```

### §17 null 后端

**源文件**：`char-null.c:29-39`，`char.c:329-339`

```
最简后端:
  null_chr_open()                        [char-null.c:29]
    └─ return  // 什么也不做

  null_chr_write(chr, buf, len)          [char.c:329]
    └─ return len  // 丢弃所有写入数据

使用场景: 当不需要串口输出时 (-serial none)
```

### §18 pipe / serial / 其他后端

**pipe 后端**（`char-pipe.c`）：

```
POSIX:
  尝试打开 path.in (读) + path.out (写)
  如果只有一个文件 → 双向使用
  → qemu_chr_open_fd(chr, fd_in, fd_out)

Windows:
  CreateNamedPipe() → 命名管道
```

**serial 后端**（`char-serial.c`）：

```
命令行: -chardev serial,id=chr0,path=/dev/ttyUSB0

serial_chr_open()                         [char-serial.c:264]
  ├─ open(path, O_RDWR | O_NONBLOCK)
  ├─ tty_serial_init(fd, 115200, 'N', 8, 1)  默认配置
  │    [char-serial.c:61]
  │    └─ tcsetattr() 设置波特率/数据位/校验/停止位
  └─ qemu_chr_open_fd(chr, fd, fd)

ioctl 支持:
  serial_chr_ioctl()                      [char-serial.c:182]
  ├─ CHR_IOCTL_SERIAL_SET_PARAMS → tty_serial_init()
  ├─ CHR_IOCTL_SERIAL_SET_BREAK → tcsendbreak()
  └─ CHR_IOCTL_SERIAL_GET_TIOCM → ioctl(TIOCMGET)
```

### §19 mux 后端 (多路复用)

**源文件**：`char-mux.c:35-480`

```
命令行: -serial mon:stdio
  → 创建 mux chardev，同时承载 serial + monitor

mux 架构:
  ┌─────────────────────┐
  │    Mux Chardev       │
  │  ┌───────┬────────┐  │
  │  │ FE #0 │ FE #1  │  │ ← 多个前端 (serial + monitor)
  │  │(UART) │(QMP)   │  │
  │  └───┬───┴───┬────┘  │
  │      │ focus │       │ ← 当前焦点前端
  │  ┌───▼───────▼────┐  │
  │  │ 底层 Chardev    │  │ ← 实际后端 (stdio)
  │  └────────────────┘  │
  └─────────────────────┘

切换焦点: Ctrl-A C (在不同前端之间切换)
Ctrl-A X: 退出 QEMU

实现:
  mux_chr_write(): 只发送给底层后端     [char-mux.c:35]
  mux_chr_read(): 
    ├─ 解析 Ctrl-A 转义序列             [char-mux.c:68-130]
    │   Ctrl-A h: 帮助
    │   Ctrl-A x: 退出
    │   Ctrl-A c: 切换焦点
    └─ 将数据投递给当前焦点前端

  mux_chr_be_event():
    └─ 事件只发给当前焦点前端            [char-mux.c:138]
```

---

## 第五部分：PL011 UART 设备

### §20 PL011 设备状态结构

**源文件**：`pl011.h:28-55`

```c
// pl011.h:28-55 (简化)
#define PL011_FIFO_DEPTH 16

struct PL011State {
    SysBusDevice parent_obj;

    MemoryRegion iomem;           // 4 KB MMIO 区域
    CharBackend chr;              // Chardev 前端接口
    qemu_irq irq[6];             // 6 条中断线 (实际主要用第0条)

    // 寄存器状态
    uint32_t flags;               // UARTFR: 标志寄存器
    uint32_t lcr;                 // UARTLCR_H: 线控制
    uint32_t rsr;                 // UARTRSR: 接收状态
    uint32_t cr;                  // UARTCR: 控制寄存器
    uint32_t dmacr;               // UARTDMACR: DMA 控制
    uint32_t int_enabled;         // UARTIMSC: 中断屏蔽
    uint32_t int_level;           // UARTRIS: 原始中断状态
    uint32_t ilpr;                // UARTILPR: IrDA 低功耗
    uint32_t ibrd;                // UARTIBRD: 整数波特率
    uint32_t fbrd;                // UARTFBRD: 小数波特率

    // RX FIFO
    uint32_t read_fifo[PL011_FIFO_DEPTH];  // 16 项循环缓冲区
    uint32_t read_pos;            // 读指针
    uint32_t read_count;          // 当前数据量
    uint32_t read_trigger;        // 中断触发阈值

    // PrimeCell ID
    const unsigned char *id;      // 设备 ID 字节
};
```

### §21 寄存器映射表

**源文件**：`pl011.c:287-335, 428-499`

| 偏移 | 名称 | 读 | 写 | 说明 |
|------|------|---|---|------|
| 0x000 | UARTDR | 从 FIFO 读数据 | 发送数据 | 数据寄存器 |
| 0x004 | UARTRSR/ECR | 接收状态 | 清错误标志 | 错误状态 |
| 0x018 | UARTFR | 标志 (TXFE/RXFF/BUSY等) | - | 只读标志 |
| 0x020 | UARTILPR | IrDA 配置 | IrDA 配置 | 低功耗 IrDA |
| 0x024 | UARTIBRD | 整数波特率 | 设置波特率 | 波特率整数部分 |
| 0x028 | UARTFBRD | 小数波特率 | 设置波特率 | 波特率小数部分 |
| 0x02C | UARTLCR_H | 线控制 | 线控制 | 数据位/停止位/奇偶 |
| 0x030 | UARTCR | 控制 | 控制 | 使能/环回 |
| 0x034 | UARTIFLS | FIFO 水位 | FIFO 水位 | 中断触发阈值 |
| 0x038 | UARTIMSC | 中断屏蔽 | 中断屏蔽 | 使能/屏蔽各中断 |
| 0x03C | UARTRIS | 原始中断状态 | - | 只读 |
| 0x040 | UARTMIS | 屏蔽后中断状态 | - | = RIS & IMSC |
| 0x044 | UARTICR | - | 清中断 | 写 1 清除 |
| 0x048 | UARTDMACR | DMA 控制 | DMA 控制 | DMA 使能 |
| 0xFE0-0xFFF | ID | PrimeCell ID | - | 设备标识 |

**UARTFR 标志位**：

| 位 | 名称 | 含义 |
|---|------|------|
| 7 | TXFE | 发送 FIFO 空 |
| 6 | RXFF | 接收 FIFO 满 |
| 5 | TXFF | 发送 FIFO 满 |
| 4 | RXFE | 接收 FIFO 空 |
| 3 | BUSY | UART 忙 |

### §22 FIFO 实现

**源文件**：`pl011.c:154-197`

```
PL011 使用 16 项循环缓冲区实现 RX FIFO:

  read_fifo[16]
  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
  │  │  │  │D0│D1│D2│D3│  │  │  │  │  │  │  │  │  │
  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
              ↑              ↑
           read_pos     read_pos + read_count

入队 (pl011_fifo_rx_put):               [pl011.c:177]
  slot = (read_pos + read_count) % fifo_depth
  read_fifo[slot] = value
  read_count++
  if (read_count == fifo_depth) → flags |= RXFF  (FIFO 满)
  flags &= ~RXFE                                  (非空)
  if (read_count == read_trigger) → int_level |= INT_RX  (触发中断)
  pl011_update()                                  (更新 IRQ 线)

出队 (pl011_read_rxdata):               [pl011.c:263]
  value = read_fifo[read_pos]
  read_pos = (read_pos + 1) % fifo_depth
  read_count--
  if (read_count == 0) → flags |= RXFE   (FIFO 空)
  flags &= ~RXFF                          (非满)
  if (read_count < read_trigger) → int_level &= ~INT_RX  (清中断)
  pl011_update()
  qemu_chr_fe_accept_input(&s->chr)       通知后端可以继续发送
```

### §23 实例初始化与 Realize

**源文件**：`pl011.c:645-699`

```
pl011_init(obj)                          [pl011.c:645]
  ├─ memory_region_init_io(&s->iomem, obj, &pl011_ops, s,
  │                         "pl011", 0x1000)
  │    // 创建 4 KB MMIO 区域
  │    // ops: pl011_read(), pl011_write()
  │
  ├─ sysbus_init_mmio(sbd, &s->iomem)
  │    // 注册到 SysBus (待映射)
  │
  └─ for (i = 0; i < 6; i++)
       sysbus_init_irq(sbd, &s->irq[i])
       // 创建 6 条 IRQ 输出线

pl011_realize(dev)                       [pl011.c:663]
  └─ qemu_chr_fe_set_handlers(&s->chr,
       pl011_can_receive,                 // FIFO 剩余空间
       pl011_receive,                     // 数据入队
       pl011_event,                       // 事件 (BREAK)
       NULL, s, NULL, true)
```

### §24 中断机制

**源文件**：`pl011.c:64-78, 132-142`

```
中断位定义:                              [pl011.c:64-78]
  INT_OE   = 0x400    // 溢出错误
  INT_BE   = 0x200    // Break 错误
  INT_PE   = 0x100    // 奇偶校验错误
  INT_FE   = 0x080    // 帧错误
  INT_RT   = 0x040    // 接收超时
  INT_TX   = 0x020    // 发送中断
  INT_RX   = 0x010    // 接收中断
  INT_DSR  = 0x008    // DSR 变化
  INT_DCD  = 0x004    // DCD 变化
  INT_CTS  = 0x002    // CTS 变化
  INT_RI   = 0x001    // RI 变化

中断更新逻辑:
  pl011_update(s)                        [pl011.c:132]
    │
    ├─ flags = s->int_level & s->int_enabled
    │    // UARTMIS = UARTRIS & UARTIMSC
    │
    └─ for (i = 0; i < 6; i++)
         qemu_set_irq(s->irq[i], (flags & irq_mask[i]) != 0)
         // irq[0]: 所有中断的 OR (主要使用)
         // irq[1-5]: 独立中断线 (较少使用)

Guest 驱动典型操作:
  1. 写 UARTIMSC = INT_RX | INT_TX       使能收发中断
  2. 数据到达 → INT_RX 置位 → pl011_update() → IRQ 拉高
  3. Guest 收到中断:
     a. 读 UARTMIS 确认中断源
     b. 读 UARTDR 取出数据 (自动更新 FIFO/中断)
     c. 写 UARTICR 清除中断位
```

---

## 第六部分：PL011 ↔ Chardev 数据路径

### §25 TX 路径：Guest → Host

**源文件**：`pl011.c:253-261, 428-440`

```
Guest 执行: str x0, [UART_BASE]   (写 UARTDR)

  MMIO 写 → pl011_write(offset=0x000)   [pl011.c:428]
    │
    └─ pl011_write_txdata(s, value)       [pl011.c:253]
         │
         ├─ 检查: 如果 UART 未使能 (CR & 0x0001 == 0)
         │    → 仅打印警告，不发送            [pl011.c:255]
         │
         ├─ ch = value & 0xFF               取低 8 位
         │
         ├─ 环回模式? (CR & LCR_H 检查)
         │    是 → pl011_fifo_rx_put(ch)     数据直接入 RX FIFO
         │
         ├─ qemu_chr_fe_write_all(&s->chr, &ch, 1)  [pl011.c:259]
         │    │
         │    └─ → chardev 后端 chr_write()
         │         → stdio: write(stdout_fd, &ch, 1)
         │         → socket: send(sock_fd, &ch, 1)
         │         → file: write(file_fd, &ch, 1)
         │
         ├─ s->int_level |= INT_TX          设置发送完成中断
         │
         └─ pl011_update()                   触发 TX 中断 (如果使能)
```

### §26 RX 路径：Host → Guest

**源文件**：`pl011.c:506-541, 177-197, 263-285`

```
用户在终端按键 'A' (0x41):

  终端 stdin → GLib poll 检测到 fd 可读
    │
    ├─ io_watch_poll_prepare()
    │    → fd_can_read()
    │    → qemu_chr_be_can_write()
    │    → pl011_can_receive(s)              [pl011.c:506]
    │       return PL011_FIFO_DEPTH - s->read_count
    │       // 例如返回 15 (FIFO 还有空间)
    │
    ├─ io_watch_poll_dispatch()              fd 可读
    │    → fd_chr_read()                     [char-fd.c:64]
    │         read(stdin_fd, buf, 15)        读出 'A'
    │         qemu_chr_be_write(chr, "A", 1) 投递给前端
    │
    └─ qemu_chr_be_write_impl()              [char.c:249]
         └─ chr->be->chr_read(opaque, "A", 1)
              │
              └─ pl011_receive(s, "A", 1)    [pl011.c:526]
                   │
                   └─ for (i=0; i<size; i++)
                        pl011_fifo_rx_put(s, buf[i])  [pl011.c:177]
                          ├─ read_fifo[slot] = 0x41
                          ├─ read_count = 1
                          ├─ flags &= ~RXFE (非空)
                          ├─ read_count >= read_trigger?
                          │    是 → int_level |= INT_RX
                          └─ pl011_update()
                               └─ qemu_set_irq(irq[0], 1)
                                    → GIC 投递 SPI 1 (INTID 33)
                                    → vCPU 收到 IRQ

  Guest IRQ 处理:
    ├─ 读 UARTMIS → 确认 INT_RX
    ├─ 读 UARTDR:
    │    pl011_read(offset=0x000)             [pl011.c:287]
    │    └─ pl011_read_rxdata(s)              [pl011.c:263]
    │         ├─ c = read_fifo[read_pos]      取出 0x41
    │         ├─ read_pos = (read_pos+1) % 16
    │         ├─ read_count = 0
    │         ├─ flags |= RXFE (FIFO 空)
    │         ├─ int_level &= ~INT_RX         清 RX 中断
    │         ├─ pl011_update()               IRQ 拉低
    │         └─ qemu_chr_fe_accept_input()   通知后端继续
    │
    └─ 写 UARTICR = INT_RX → 清除中断 (冗余但安全)
```

### §27 事件处理 (Break / Open)

**源文件**：`pl011.c:543-548`

```
pl011_event(opaque, event)               [pl011.c:543]
  │
  ├─ CHR_EVENT_BREAK:
  │    // 模拟串口 BREAK 信号
  │    ├─ 如果非环回模式:
  │    │    pl011_fifo_rx_put(s, DR_BE)  // 带 Break Error 标志入 FIFO
  │    └─ 触发中断
  │
  ├─ CHR_EVENT_OPENED:
  │    // 后端连接 (如 socket 客户端连接)
  │    → 无特殊处理 (PL011 不区分连接状态)
  │
  └─ CHR_EVENT_CLOSED:
       // 后端断开
       → 无特殊处理
```

### §28 流控与 FIFO 溢出

```
流控机制:

1. pl011_can_receive() 返回 FIFO 剩余空间
   → 后端据此限制读取量
   → 永远不会超过 FIFO 容量

2. 当 FIFO 满 (read_count == 16):
   ├─ pl011_can_receive() 返回 0
   ├─ 后端 fd watch 暂停 (不从 stdin 读取)
   ├─ flags |= RXFF (FIFO 满标志)
   └─ 数据在 stdin 缓冲区中等待

3. Guest 读取 UARTDR 后:
   ├─ read_count 减少
   ├─ qemu_chr_fe_accept_input()
   │    → 后端 chr_accept_input()
   │    → 重新检查 can_write()
   │    → 恢复 fd watch
   └─ 后端继续从 stdin 读取

FIFO 溢出保护:
  pl011_fifo_rx_put() 检查 read_count >= fifo_depth
    → 如果满: 不入队，设置 OE (溢出) 错误位
    → int_level |= INT_OE
```

---

## 第七部分：端到端数据流

### §29 完整 TX 时序：Guest printf → 终端显示

```
Guest (Linux)         PL011 Device        Chardev Core       stdio Backend      终端
    │                     │                    │                   │              │
    │ printf("Hello")     │                    │                   │              │
    │ → tty driver        │                    │                   │              │
    │ → uart_write()      │                    │                   │              │
    ├─ str 'H' → UARTDR ►│                    │                   │              │
    │                     │                    │                   │              │
    │                     ├─ pl011_write()      │                   │              │
    │                     ├─ pl011_write_txdata │                   │              │
    │                     ├─ fe_write_all ─────►│                   │              │
    │                     │                    ├─ qemu_chr_write()  │              │
    │                     │                    ├─ chr_write_lock    │              │
    │                     │                    ├─ cc->chr_write ───►│              │
    │                     │                    │                   ├─ fd_chr_write │
    │                     │                    │                   ├─ write(1,'H')─►│
    │                     │                    │                   │              │ 显示 'H'
    │                     │                    │◄──────────────────┤              │
    │                     │                    │  return 1         │              │
    │                     │◄───────────────────┤                   │              │
    │                     │                    │                   │              │
    │                     ├─ int_level|=INT_TX  │                   │              │
    │                     ├─ pl011_update()     │                   │              │
    │                     │  (如果 TX 中断使能: │                   │              │
    │                     │   qemu_set_irq →   │                   │              │
    │                     │   GIC → CPU IRQ)    │                   │              │
    │                     │                    │                   │              │
    │ ◄── TX 中断 ────────┤                    │                   │              │
    │ 继续发送 'e','l',.. │                    │                   │              │
    │                     │                    │                   │              │
```

### §30 完整 RX 时序：用户按键 → Guest 读取

```
终端          stdio Backend    Chardev Core     PL011 Device      GIC        Guest
  │               │                │                │              │            │
  │ 按键 'A'      │                │                │              │            │
  ├──►stdin fd    │                │                │              │            │
  │               │                │                │              │            │
  │   main_loop_wait()             │                │              │            │
  │   GLib poll → stdin 可读       │                │              │            │
  │               │                │                │              │            │
  │   prepare:    │                │                │              │            │
  │   can_read?──►│                │                │              │            │
  │               ├─ be_can_write─►│                │              │            │
  │               │                ├─ can_receive──►│              │            │
  │               │                │                ├─ return 15   │            │
  │               │                │◄───────────────┤              │            │
  │               │◄───────────────┤ return 15      │              │            │
  │               │                │                │              │            │
  │   dispatch:   │                │                │              │            │
  │               ├─ read(0)='A'   │                │              │            │
  │               ├─ be_write ────►│                │              │            │
  │               │                ├─ chr_read ────►│              │            │
  │               │                │                ├─ receive()   │            │
  │               │                │                ├─ FIFO 入队   │            │
  │               │                │                ├─ INT_RX 置位 │            │
  │               │                │                ├─ update()    │            │
  │               │                │                ├─ set_irq ───►│            │
  │               │                │                │              ├─ SPI 1     │
  │               │                │                │              ├─ cpuif ───►│
  │               │                │                │              │            │ IRQ
  │               │                │                │              │            │
  │               │                │                │              │            ├─ 读 UARTMIS
  │               │                │                │◄─────────────┼────────────┤  → INT_RX
  │               │                │                │              │            │
  │               │                │                │              │            ├─ 读 UARTDR
  │               │                │                │◄─────────────┼────────────┤
  │               │                │                ├─ 出队 'A'    │            │
  │               │                │                ├─ INT_RX 清除 │            │
  │               │                │                ├─ update()    │            │
  │               │                │                ├─ IRQ 拉低 ──►│            │
  │               │                │                │              │            │
  │               │                │                ├─ accept_input│            │
  │               │                │◄───────────────┤              │            │
  │               │◄───────────────┤ chr_accept     │              │            │
  │               │  恢复 fd watch │                │              │            │
  │               │                │                │              │            │
  │               │                │                │              │        返回 'A'
```

### §31 -serial stdio 完整连接链

```
命令行: qemu-system-aarch64 -M virt -serial stdio

创建链 (从底到顶):

1. 后端创建 (vl.c):
   serial_parse("stdio")
   → qemu_chr_new_mux_mon("serial0", "stdio")
   → chardev_new("serial0", TYPE_CHARDEV_STDIO, ...)
   → stdio_chr_open()
      ├─ tcsetattr() → raw 模式
      ├─ QIOChannelFile(stdin) + QIOChannelFile(stdout)
      └─ io_add_watch_poll() → 注册 stdin 读 watch

2. 设备创建 (virt.c:1295-1312):
   create_uart(vms, VIRT_UART0, sysmem, serial_hd(0), false)
   → dev = qdev_new(TYPE_PL011)
   → qdev_prop_set_chr(dev, "chardev", serial_hd(0))
      // 将 Chardev* 绑定到 PL011 的 CharBackend.chr
   → sysbus_realize_and_unref(sbd)

3. Realize 连接 (pl011.c:663):
   pl011_realize()
   → qemu_chr_fe_set_handlers(&s->chr,
       pl011_can_receive, pl011_receive, pl011_event, ...)
      // 安装前端回调到 CharBackend
      // 通知后端更新读处理器

4. MMIO 映射 (virt.c:1311):
   memory_region_add_subregion(sysmem, 0x09000000, &sbd->mmio[0])

5. IRQ 连接 (virt.c:1312):
   sysbus_connect_irq(sbd, 0, gic_spi_1)

最终连接关系:
  stdin (fd 0) ← io_watch_poll ← GLib MainLoop
       ↓ 数据
  stdio Chardev (fd_chr_read)
       ↓ qemu_chr_be_write
  CharBackend (pl011 回调)
       ↓ pl011_receive
  PL011 RX FIFO
       ↓ INT_RX
  GIC SPI 1 (INTID 33)
       ↓
  vCPU IRQ → Guest 中断处理

  Guest 写 UARTDR
       ↓ MMIO trap
  PL011 (pl011_write_txdata)
       ↓ qemu_chr_fe_write_all
  stdio Chardev (fd_chr_write)
       ↓ write
  stdout (fd 1) → 终端显示
```

---

## 附录

### 附录A：Chardev 后端一览表

| 类型 | QOM 类型名 | 命令行 | 读 | 写 | 说明 |
|------|-----------|--------|---|---|------|
| null | chardev-null | `-serial none` | ✗ | 丢弃 | 空设备 |
| stdio | chardev-stdio | `-serial stdio` | ✓ | ✓ | 终端 stdin/stdout |
| pty | chardev-pty | `-serial pty` | ✓ | ✓ | 伪终端对 |
| file | chardev-file | `-serial file:path` | ✗ | ✓ | 输出到文件 |
| pipe | chardev-pipe | `-serial pipe:name` | ✓ | ✓ | 命名管道 |
| serial | chardev-serial | `-serial /dev/ttyS0` | ✓ | ✓ | 主机串口 |
| socket | chardev-socket | `-chardev socket,...` | ✓ | ✓ | TCP/Unix socket |
| udp | chardev-udp | `-chardev udp,...` | ✓ | ✓ | UDP socket |
| mux | chardev-mux | `-serial mon:stdio` | ✓ | ✓ | 多路复用 |
| ringbuf | chardev-ringbuf | `-chardev ringbuf,...` | ✓ | ✓ | 环形缓冲区 |
| memory | chardev-memory | `-chardev memory,...` | ✓ | ✓ | 内存缓冲区 |

### 附录B：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `char.h` | ~316 | Chardev/ChardevClass 定义、类型常量 |
| `char-fe.h` | ~200 | CharBackend 定义、前端 API 声明 |
| `char.c` | ~1,340 | Chardev 核心：创建/销毁/读写分发 |
| `char-fe.c` | ~370 | 前端 API 实现：init/set_handlers/write |
| `char-io.c` | ~152 | io_add_watch_poll GSource 集成 |
| `char-fd.c` | ~180 | fd 后端基类：fd_chr_read/write |
| `char-stdio.c` | ~150 | stdio 后端：终端 raw 模式 |
| `char-socket.c` | ~1,580 | socket 后端：TCP 服务端/客户端/重连 |
| `char-pty.c` | ~412 | pty 后端：伪终端 |
| `char-file.c` | ~138 | file 后端：文件输出 |
| `char-null.c` | ~39 | null 后端：空实现 |
| `char-mux.c` | ~480 | mux 后端：多路复用 + Ctrl-A 转义 |
| `char-serial.c` | ~316 | serial 后端：主机串口 |
| `char-pipe.c` | ~187 | pipe 后端：命名管道 |
| `pl011.c` | ~700 | PL011 UART 设备模拟 |
| `pl011.h` | ~55 | PL011 状态结构定义 |

### 附录C：关键 Git 提交

| 提交 | 说明 |
|------|------|
| `5c102ac9f1` | chardev: Consolidate yank registration |
| `3f0170505e` | chardev: Don't attempt to unregister yank function more than once |
| `1423c170b9` | chardev: Fix QIOChannel refcount |
| `907b8d5635` | hw/char/pl011: Only log "data written to disabled UART" once |
| `b108dcbebc` | chardev: add logtimestamp option |
| `518d1458bd` | chardev/char: qemu_char_open(): add return value |
| `bc339129b6` | chardev: rework filename handling |

---

> **交叉引用**
> - QOM 对象模型 → [../architecture/01-QOM对象模型深度分析](../architecture/01-QOM对象模型深度分析.md)
> - UART 设备仿真概述 → [01-关键设备仿真分析-UART-磁盘-网卡](01-关键设备仿真分析-UART-磁盘-网卡.md)
> - Machine 建立流程 (UART 创建) → [../architecture/03-Machine建立流程深度分析](../architecture/03-Machine建立流程深度分析.md)
> - 中断注入路径 (UART IRQ) → [../arm64/04-中断注入与处理深度分析](../arm64/04-中断注入与处理深度分析.md)
> - 执行循环与 MMIO 分发 → [../architecture/02-模拟执行循环与MMIO分发深度分析](../architecture/02-模拟执行循环与MMIO分发深度分析.md)
