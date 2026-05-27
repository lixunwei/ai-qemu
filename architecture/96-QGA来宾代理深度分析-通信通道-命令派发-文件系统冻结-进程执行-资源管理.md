# Doc 96: QEMU Guest Agent (QGA) 深度分析

## 架构 · 通信通道 · 命令派发 · 文件系统冻结 · 进程执行 · 资源管理

> QEMU 11.0.50 · qga/ 子系统 · IBM 实现
> 分析日期: 2025-01

---

## 目录

1. [架构总览](#1-架构总览)
2. [通信模型](#2-通信模型)
3. [核心数据结构](#3-核心数据结构)
4. [初始化与生命周期](#4-初始化与生命周期)
5. [通道层实现](#5-通道层实现)
6. [消息解析与命令派发](#6-消息解析与命令派发)
7. [同步协议 (guest-sync)](#7-同步协议)
8. [文件操作命令](#8-文件操作命令)
9. [进程执行命令](#9-进程执行命令)
10. [文件系统冻结](#10-文件系统冻结)
11. [关机与电源管理](#11-关机与电源管理)
12. [网络信息查询](#12-网络信息查询)
13. [vCPU 管理](#13-vcpu-管理)
14. [内存块管理](#14-内存块管理)
15. [磁盘与文件系统信息](#15-磁盘与文件系统信息)
16. [时间同步](#16-时间同步)
17. [用户密码管理](#17-用户密码管理)
18. [Windows 特有功能](#18-windows-特有功能)
19. [Frozen 状态与安全](#19-frozen-状态与安全)
20. [与 Host 侧 QEMU 的关系](#20-与-host-侧-qemu-的关系)

---

## 1. 架构总览

### 1.1 什么是 QGA

QEMU Guest Agent 是运行在**虚拟机内部**的守护进程，提供 host 对 guest OS 的管理能力。它类似于 VMware Tools 或 Hyper-V Integration Services。

### 1.2 整体架构

```
┌─────────────── Host ───────────────┐     ┌──────────── Guest ─────────────┐
│                                     │     │                                │
│  管理工具 (libvirt/virsh/QMP)       │     │   qemu-ga (守护进程)            │
│         │                           │     │         │                      │
│         ▼                           │     │         ▼                      │
│  QEMU Monitor (QMP)                 │     │  命令分发 (qmp_dispatch)        │
│         │                           │     │         │                      │
│         ▼                           │     │         ▼                      │
│  chardev (socket/file)              │     │  命令实现 (commands-*.c)        │
│         │                           │     │         │                      │
│         ▼                           │     │         ▼                      │
│  virtio-serial-bus                  │ ←──→│  /dev/virtio-ports/...         │
│  (hw/char/virtio-serial-bus.c)      │     │  (channel-posix.c)             │
│                                     │     │                                │
└─────────────────────────────────────┘     └────────────────────────────────┘
```

### 1.3 关键特点

| 特性 | 说明 |
|------|------|
| 独立进程 | 在 guest 内作为独立守护进程运行 |
| JSON 协议 | 基于 QMP (QEMU Machine Protocol) 子集 |
| QAPI 定义 | 命令由 `qga/qapi-schema.json` 声明 |
| 多平台 | 支持 Linux/BSD/Windows |
| 多传输 | virtio-serial (默认)、ISA 串口、Unix socket、vsock |
| 安全控制 | 支持命令黑名单、frozen 状态 |

### 1.4 源码组织

```
qga/
├── main.c                  # 入口、初始化、事件循环、消息处理
├── guest-agent-core.h      # 核心声明 (GAState, GACommandState)
├── commands.c              # 通用命令 (sync/ping/info/exec/file-read)
├── commands-common.h       # 命令辅助声明
├── commands-posix.c        # POSIX 命令 (shutdown/file/fsfreeze/network/time)
├── commands-linux.c        # Linux 专有 (vcpu/memory/disk/fstrim/suspend)
├── commands-bsd.c          # BSD 专有
├── commands-win32.c        # Windows 专有
├── channel.h               # 通道抽象接口
├── channel-posix.c         # POSIX 通道实现
├── channel-win32.c         # Windows 通道实现
├── qapi-schema.json        # QAPI 命令定义
├── vss-win32.c/h           # Windows VSS 集成
├── cutils.c                # 辅助工具函数
└── service-win32.c/h       # Windows 服务注册
```

---

## 2. 通信模型

### 2.1 传输层选择

```c
// qga/main.c:751-777
static gboolean channel_init(GAState *s, const gchar *method, ...) {
    if (strcmp(method, "virtio-serial") == 0) {
        channel_method = GA_CHANNEL_VIRTIO_SERIAL;  // 默认
    } else if (strcmp(method, "isa-serial") == 0) {
        channel_method = GA_CHANNEL_ISA_SERIAL;
    } else if (strcmp(method, "unix-listen") == 0) {
        channel_method = GA_CHANNEL_UNIX_LISTEN;
    } else if (strcmp(method, "vsock-listen") == 0) {
        channel_method = GA_CHANNEL_VSOCK_LISTEN;
    }
}
```

### 2.2 virtio-serial 通道（默认）

**Guest 侧设备路径**：
- Linux: `/dev/virtio-ports/org.qemu.guest_agent.0`
- BSD: `/dev/vtcon/org.qemu.guest_agent.0`
- Windows: `\\.\Global\org.qemu.guest_agent.0`

**Host 侧配置**：
```bash
-device virtio-serial \
-chardev socket,path=/tmp/qga.sock,server=on,wait=off,id=qga0 \
-device virtserialport,chardev=qga0,name=org.qemu.guest_agent.0
```

### 2.3 协议格式

使用 JSON 文本协议（QMP 子集）：

**请求**：
```json
{"execute": "guest-file-open", "arguments": {"path": "/etc/hosts", "mode": "r"}}
```

**响应（成功）**：
```json
{"return": 1000}
```

**响应（错误）**：
```json
{"error": {"class": "GenericError", "desc": "No such file"}}
```

### 2.4 同步定界

`guest-sync-delimited` 使用 `0xFF` 字节作为响应前缀，帮助客户端定位响应边界：

```c
// qga/main.c:672-675
if (s->delimit_response) {
    s->delimit_response = false;
    g_string_prepend_c(response, QGA_SENTINEL_BYTE);  // 0xFF
}
```

---

## 3. 核心数据结构

### 3.1 GAState

```c
// qga/guest-agent-core.h:25-28
typedef struct GAState GAState;
typedef struct GACommandState GACommandState;
extern GAState *ga_state;       // 全局唯一实例
extern QmpCommandList ga_commands; // 命令注册表
```

GAState 包含（在 main.c 中定义）：
- 通道 (`GAChannel *channel`)
- JSON 解析器 (`JSONMessageParser parser`)
- 日志配置
- frozen 状态标志
- 持久化状态
- 命令状态管理器

### 3.2 GAChannel

```c
// qga/channel-posix.c:15-21
struct GAChannel {
    GIOChannel *listen_channel;   // 监听通道 (unix/vsock)
    GIOChannel *client_channel;   // 客户端连接
    GAChannelMethod method;       // 传输类型
    GAChannelCallback event_cb;   // 数据到达回调
    gpointer user_data;           // 回调参数 (GAState*)
};
```

### 3.3 命令注册

```c
// 通过 QAPI 代码生成:
extern QmpCommandList ga_commands;

// 命令注册在 qga-qapi-init-commands.c (自动生成)
// 每个命令对应一个 qmp_guest_xxx() 函数
```

---

## 4. 初始化与生命周期

### 4.1 启动流程

```
main()
  → 解析命令行参数
  → become_daemon()          (可选守护进程化)
  → initialize_agent()
      → 初始化日志
      → 加载持久化状态
      → ga_command_state_new()
      → ga_command_state_init()  (注册命令 init/cleanup 回调)
      → json_message_parser_init(&s->parser, process_event, s)
      → channel_init()          (打开通信通道)
      → replay_vmstate_init()   (如果适用)
  → run_agent()
      → g_main_loop_run()       (GLib 事件循环)
```

### 4.2 守护进程化

```c
// qga/main.c:610-654
static void become_daemon(const char *pidfile) {
    pid_t pid = fork();
    if (pid > 0) exit(EXIT_SUCCESS);  // 父进程退出
    setsid();                          // 新会话
    pid = fork();
    if (pid > 0) exit(EXIT_SUCCESS);  // 再次 fork
    reopen_fd_to_null(STDIN_FILENO);
    reopen_fd_to_null(STDOUT_FILENO);
    reopen_fd_to_null(STDERR_FILENO);
    // 写 PID 文件
}
```

### 4.3 重连机制

```c
// qga/main.c:1641-1655 (概念)
static int run_agent(GAState *s) {
    while (true) {
        int ret = run_agent_once(s);
        if (s->virtio && ret == 0) {
            // virtio-serial 断开后自动重连
            channel_init(s, ...);
            continue;
        }
        return ret;
    }
}
```

对于 virtio-serial，连接断开（host chardev 关闭）后会自动重连。

---

## 5. 通道层实现

### 5.1 virtio-serial 打开

```c
// qga/channel-posix.c:134-182
case GA_CHANNEL_VIRTIO_SERIAL: {
    fd = qga_open_cloexec(path, O_ASYNC | O_RDWR | O_NONBLOCK, 0);
    // FreeBSD: 禁用回显
    // Solaris: 设置 I_SETSIG
    ret = ga_channel_client_add(c, fd);  // 注册到 GLib 事件循环
    break;
}
```

### 5.2 GLib I/O Watch

```c
// qga/channel-posix.c:108-126
static int ga_channel_client_add(GAChannel *c, int fd) {
    GIOChannel *client_channel = g_io_channel_unix_new(fd);
    g_io_channel_set_encoding(client_channel, NULL, &err);  // 二进制模式
    g_io_add_watch(client_channel, G_IO_IN | G_IO_HUP,
                   ga_channel_client_event, c);
    c->client_channel = client_channel;
    return 0;
}
```

### 5.3 事件回调链

```
G_IO_IN 事件
  → ga_channel_client_event()
    → c->event_cb(condition, c->user_data)
      → channel_event_cb()  [main.c:714-749]
        → ga_channel_read()
        → json_message_parser_feed()
          → process_event()  [解析完成时]
```

### 5.4 ISA 串口通道

```c
// qga/channel-posix.c:184-215
case GA_CHANNEL_ISA_SERIAL: {
    fd = qga_open_cloexec(path, O_RDWR | O_NOCTTY | O_NONBLOCK, 0);
    // 设置终端为 raw 模式
    tio.c_iflag &= ~(所有输入处理标志);
    tio.c_oflag = 0;
    tio.c_lflag = 0;
    tio.c_cflag |= B38400;  // 默认波特率
    tcsetattr(fd, TCSAFLUSH, &tio);
    ret = ga_channel_client_add(c, fd);
}
```

---

## 6. 消息解析与命令派发

### 6.1 数据接收

```c
// qga/main.c:714-749
static gboolean channel_event_cb(GIOCondition condition, gpointer data) {
    GAState *s = data;
    gchar buf[QGA_READ_COUNT_DEFAULT + 1];  // 4096 + 1
    gsize count;
    GIOStatus status = ga_channel_read(s->channel, buf, QGA_READ_COUNT_DEFAULT, &count);

    switch (status) {
    case G_IO_STATUS_NORMAL:
        buf[count] = 0;
        json_message_parser_feed(&s->parser, (char *)buf, (int)count);
        break;
    case G_IO_STATUS_EOF:
        if (!s->virtio) return false;  // 非 virtio 连接断开
        // virtio: sleep 后重试
        g_usleep(G_USEC_PER_SEC / 10);
        return true;
    }
    return true;
}
```

### 6.2 命令派发

```c
// qga/main.c:687-711
static void process_event(void *opaque, QObject *obj, Error *err) {
    GAState *s = opaque;
    QDict *rsp;

    if (err) {
        rsp = qmp_error_response(err);  // JSON 解析错误
        goto end;
    }

    // 核心派发：根据 "execute" 字段查找并调用命令函数
    rsp = qmp_dispatch(&ga_commands, obj, false, NULL);

end:
    send_response(s, rsp);  // 序列化 JSON 响应并写回通道
    qobject_unref(rsp);
    qobject_unref(obj);
}
```

### 6.3 响应发送

```c
// qga/main.c:656-685
static int send_response(GAState *s, const QDict *rsp) {
    GString *response = qobject_to_json(QOBJECT(rsp));

    if (s->delimit_response) {
        g_string_prepend_c(response, QGA_SENTINEL_BYTE);  // 0xFF 前缀
    }
    g_string_append_c(response, '\n');  // 行终止符

    status = ga_channel_write_all(s->channel, response->str, response->len);
    return (status == G_IO_STATUS_NORMAL) ? 0 : -EIO;
}
```

---

## 7. 同步协议

### 7.1 guest-sync-delimited

```c
// qga/commands.c:46-55
GuestSyncDelimited *qmp_guest_sync_delimited(int64_t id, Error **errp) {
    ga_set_response_delimited(ga_state);  // 设置 0xFF 前缀标志
    return qmp_guest_sync(id, errp);
}
```

### 7.2 guest-sync

```c
int64_t qmp_guest_sync(int64_t id, Error **errp) {
    return id;  // 原样返回 ID，客户端以此确认通信同步
}
```

### 7.3 同步流程

```
Client                          QGA
  │                               │
  │─── {"execute":"guest-sync-delimited", ───→
  │     "arguments":{"id":12345}}  │
  │                               │
  │←── 0xFF{"return":12345}\n ────│
  │                               │
  │  (确认收到 ID 12345，通信已同步)  │
```

用途：
- 重新同步通信流（丢弃旧的未读响应）
- 0xFF 作为字节边界标记，帮助在字节流中定位 JSON 开始

---

## 8. 文件操作命令

### 8.1 guest-file-open

```c
// qga/commands-posix.c:512-545
int64_t qmp_guest_file_open(const char *path, const char *mode, Error **errp) {
    if (!mode) mode = "r";
    FILE *fh = safe_open_or_create(path, mode, &local_err);
    qemu_set_blocking(fileno(fh), false, errp);  // 非阻塞
    handle = guest_file_handle_add(fh, errp);     // 分配句柄 ID
    return handle;
}
```

### 8.2 guest-file-close

```c
// qga/commands-posix.c:547-565
void qmp_guest_file_close(int64_t handle, Error **errp) {
    GuestFileHandle *gfh = guest_file_handle_find(handle, errp);
    fclose(gfh->fh);
    QTAILQ_REMOVE(&guest_file_state.filehandles, gfh, next);
    g_free(gfh);
}
```

### 8.3 guest-file-read

从文件读取数据，返回 base64 编码：

```json
请求: {"execute":"guest-file-read","arguments":{"handle":1000,"count":1024}}
响应: {"return":{"count":512,"buf-b64":"SGVsbG8gV29ybGQ=","eof":false}}
```

### 8.4 guest-file-write

写入 base64 编码数据到文件：

```json
请求: {"execute":"guest-file-write","arguments":{"handle":1000,"buf-b64":"SGVsbG8="}}
响应: {"return":{"count":5,"eof":false}}
```

### 8.5 句柄管理

```c
// 内部文件句柄表（QTAILQ 链表）
static struct {
    QTAILQ_HEAD(, GuestFileHandle) filehandles;
} guest_file_state;

typedef struct GuestFileHandle {
    int64_t id;
    FILE *fh;
    QTAILQ_ENTRY(GuestFileHandle) next;
} GuestFileHandle;
```

---

## 9. 进程执行命令

### 9.1 guest-exec

```c
// qga/commands.c:409-530
GuestExec *qmp_guest_exec(const char *path, strList *arg, strList *env,
                           const char *input_data, bool capture_output, ...) {
    // 1. fork + exec
    // 2. 可选 stdin 输入 (base64 解码)
    // 3. 可选 stdout/stderr 捕获
    // 4. 返回 PID 作为句柄
}
```

### 9.2 guest-exec-status

```c
// qga/commands.c:147-225
GuestExecStatus *qmp_guest_exec_status(int64_t pid, Error **errp) {
    // 检查进程是否完成
    // 返回 exit code + stdout/stderr (base64)
}
```

### 9.3 异步执行模型

```
Host                        QGA                         Guest Process
 │                           │                              │
 │── guest-exec ───────────→ │── fork + exec ─────────────→ │
 │                           │                              │
 │←── {"return":{"pid":N}} ──│                              │ (运行中)
 │                           │                              │
 │── guest-exec-status ────→ │── waitpid(WNOHANG) ────────→ │
 │                           │                              │
 │←── {"return":           ──│  (完成后返回)                  │
 │     {"exited":true,       │                              │
 │      "exitcode":0,        │                              │
 │      "out-data":"..."}}   │                              │
```

---

## 10. 文件系统冻结

### 10.1 用途

`guest-fsfreeze-freeze` 用于在创建 VM 快照时冻结文件系统，保证快照的一致性。

### 10.2 冻结流程

```c
// qga/commands-posix.c:788-826
int64_t qmp_guest_fsfreeze_freeze_list(bool has_mountpoints, strList *mountpoints, ...) {
    // 1. 执行 fsfreeze hook 脚本
    execute_fsfreeze_hook(FSFREEZE_HOOK_FREEZE, &local_err);

    // 2. 构建文件系统挂载列表
    build_fs_mount_list(&mounts, &local_err);

    // 3. 设置 frozen 状态 (禁止新写入)
    ga_set_frozen(ga_state);

    // 4. 对每个文件系统执行 FIFREEZE ioctl
    ret = qmp_guest_fsfreeze_do_freeze_list(..., mounts, errp);

    return ret;  // 返回冻结的文件系统数量
}
```

### 10.3 Linux FIFREEZE/FITHAW

```c
// qga/commands-linux.c:220-275 (概念)
int64_t qmp_guest_fsfreeze_do_freeze_list(...) {
    QTAILQ_FOREACH(mount, &mounts, next) {
        fd = open(mount->dirname, O_RDONLY);
        ret = ioctl(fd, FIFREEZE);  // 内核文件系统冻结
        if (ret == 0) frozen_count++;
        close(fd);
    }
    return frozen_count;
}
```

### 10.4 解冻

```c
// qga/commands-posix.c:828-843
int64_t qmp_guest_fsfreeze_thaw(Error **errp) {
    ret = qmp_guest_fsfreeze_do_thaw(errp);  // ioctl(FITHAW)
    if (ret >= 0) {
        ga_unset_frozen(ga_state);     // 清除 frozen 标志
        execute_fsfreeze_hook(FSFREEZE_HOOK_THAW, errp);
    }
    return ret;
}
```

### 10.5 Frozen 状态影响

当 `ga_is_frozen()` 为 true 时：
- QGA 不会尝试写入磁盘（防止死锁）
- 日志输出被禁用
- 仅允许 thaw 相关命令执行
- 异常退出时自动 thaw（`guest_fsfreeze_cleanup()`）

---

## 11. 关机与电源管理

### 11.1 guest-shutdown

```c
// qga/commands-posix.c:223-291
void qmp_guest_shutdown(const char *mode, Error **errp) {
    // mode: "powerdown" | "halt" | "reboot"
    // 优先使用 /sbin/poweroff, /sbin/halt, /sbin/reboot
    // 否则使用 /sbin/shutdown -h/-P/-r +0
    ga_run_command(argv, NULL, "shutdown", &local_err);
}
```

### 11.2 guest-suspend-*

```c
// qga/commands-linux.c:1467-1480
void qmp_guest_suspend_disk(Error **errp) {
    // echo disk > /sys/power/state (休眠到磁盘)
}

void qmp_guest_suspend_ram(Error **errp) {
    // echo mem > /sys/power/state (挂起到内存)
}

void qmp_guest_suspend_hybrid(Error **errp) {
    // echo suspend > /sys/power/state (混合挂起)
}
```

---

## 12. 网络信息查询

### 12.1 guest-network-get-interfaces

```c
// qga/commands-posix.c (HAVE_GETIFADDRS 块)
GuestNetworkInterfaceList *qmp_guest_network_get_interfaces(Error **errp) {
    struct ifaddrs *ifap;
    getifaddrs(&ifap);
    // 遍历所有接口，收集:
    //   - 接口名
    //   - MAC 地址
    //   - IPv4/IPv6 地址列表
    //   - 前缀长度
    freeifaddrs(ifap);
}
```

### 12.2 返回格式

```json
{"return": [
  {"name": "eth0",
   "hardware-address": "52:54:00:12:34:56",
   "ip-addresses": [
     {"ip-address": "192.168.1.100", "ip-address-type": "ipv4", "prefix": 24}
   ]}
]}
```

### 12.3 guest-network-get-route

```c
// qga/commands-linux.c:2153
GuestNetworkRouteList *qmp_guest_network_get_route(Error **errp) {
    // 解析 /proc/net/route 和 /proc/net/ipv6_route
}
```

---

## 13. vCPU 管理

### 13.1 guest-get-vcpus

```c
// qga/commands-linux.c:1551-1591
GuestLogicalProcessorList *qmp_guest_get_vcpus(Error **errp) {
    // 扫描 /sys/devices/system/cpu/cpu*/
    // 读取 online 状态
    // 返回 CPU 列表 + 在线状态 + 是否可离线
}
```

### 13.2 guest-set-vcpus

```c
// qga/commands-linux.c:1593-1621
int64_t qmp_guest_set_vcpus(GuestLogicalProcessorList *vcpus, Error **errp) {
    // 写入 /sys/devices/system/cpu/cpuN/online (0 或 1)
    // 返回成功处理的 CPU 数量
}
```

### 13.3 配合 CPU 热插拔

```
Host 热插拔流程:
1. device_add → ACPI GED 通知 → Guest ACPI 扫描发现新 CPU
2. Guest kernel 注册新 CPU (初始 offline)
3. Host 通过 QGA: guest-set-vcpus → 将新 CPU online

Host 热拔出流程:
1. Host 通过 QGA: guest-set-vcpus → 将 CPU offline
2. device_del → ACPI 释放
```

---

## 14. 内存块管理

### 14.1 guest-get-memory-blocks

```c
// qga/commands-linux.c:1804-1865
GuestMemoryBlockList *qmp_guest_get_memory_blocks(Error **errp) {
    // 扫描 /sys/devices/system/memory/memory*/
    // 读取 state: online/offline
    // 返回内存块列表
}
```

### 14.2 guest-set-memory-blocks

```c
// qga/commands-linux.c:1867-1897
GuestMemoryBlockResponseList *qmp_guest_set_memory_blocks(...) {
    // 写入 /sys/devices/system/memory/memoryN/state
    // "online" 或 "offline"
}
```

### 14.3 guest-get-memory-block-info

```c
// qga/commands-linux.c:1897+
GuestMemoryBlockInfo *qmp_guest_get_memory_block_info(Error **errp) {
    // 返回 memory block 大小（从内核获取）
}
```

---

## 15. 磁盘与文件系统信息

### 15.1 guest-get-disks

```c
// qga/commands-linux.c:1003
GuestDiskInfoList *qmp_guest_get_disks(Error **errp) {
    // 扫描 /sys/block/
    // 收集: 设备名、大小、分区信息
    // 支持 NVMe 设备信息查询
}
```

### 15.2 guest-get-fsinfo

```c
// qga/commands-linux.c:1116
GuestFilesystemInfoList *qmp_guest_get_fsinfo(Error **errp) {
    // 解析 /proc/self/mountinfo
    // 返回: 挂载点、设备、文件系统类型、总空间、可用空间
}
```

### 15.3 guest-fstrim

```c
// qga/commands-linux.c:1151
GuestFilesystemTrimResponse *qmp_guest_fstrim(bool has_minimum, int64_t minimum, ...) {
    // 对每个文件系统执行 FITRIM ioctl
    // 返回每个 FS 释放的字节数
}
```

---

## 16. 时间同步

### 16.1 guest-set-time

```c
// qga/commands-posix.c:293-335
void qmp_guest_set_time(bool has_time, int64_t time_ns, Error **errp) {
    if (has_time) {
        tv.tv_sec = time_ns / 1000000000;
        tv.tv_usec = (time_ns % 1000000000) / 1000;
        settimeofday(&tv, NULL);       // 设置系统时间
    }
    // 同步硬件时钟: /sbin/hwclock -w (系统→RTC) 或 -s (RTC→系统)
    ga_run_command(argv, NULL, "set hardware clock", &local_err);
}
```

### 16.2 用途

VM 从快照恢复后，guest 系统时间可能过时。Host 可通过 `guest-set-time` 更新。

---

## 17. 用户密码管理

### 17.1 guest-set-user-password

```c
// qga/commands-posix.c:860+
void qmp_guest_set_user_password(const char *username,
                                  const char *password,
                                  bool crypted, Error **errp) {
    // password 是 base64 编码
    rawpasswddata = qbase64_decode(password, ...);
    // 通过 chpasswd 或 usermod 设置密码
}
```

---

## 18. Windows 特有功能

### 18.1 VSS (Volume Shadow Copy Service)

- `qga/vss-win32.c`: Windows VSS 提供者集成
- `guest-fsfreeze-freeze` 在 Windows 上使用 VSS API 创建卷影副本
- 比 Linux FIFREEZE 更可靠（应用感知）

### 18.2 Windows 服务

- `qga/service-win32.c`: 注册为 Windows 服务
- 自动启动、服务控制管理器集成

### 18.3 Windows 特有命令

- `guest-get-osinfo`: 读取注册表获取 Windows 版本
- VSS 冻结/解冻
- 磁盘信息通过 WMI 查询

---

## 19. Frozen 状态与安全

### 19.1 Frozen 状态

```c
// qga/guest-agent-core.h
bool ga_is_frozen(GAState *s);
void ga_set_frozen(GAState *s);
void ga_unset_frozen(GAState *s);
```

Frozen 状态下：
- 禁止文件系统写入（防止冻结时写入导致死锁）
- 禁止日志记录
- 仅处理 thaw 和基本查询命令

### 19.2 命令黑名单

```c
// 启动参数: --block-rpcs=guest-file-open,guest-exec
GList *ga_command_init_blockedrpcs(GList *blockedrpcs);
```

可以通过配置文件或命令行参数禁止特定命令，增强安全性。

### 19.3 安全注意事项

| 风险 | 缓解措施 |
|------|---------|
| 任意文件读写 | 黑名单 guest-file-* |
| 任意命令执行 | 黑名单 guest-exec |
| 密码修改 | 黑名单 guest-set-user-password |
| 关机/重启 | 黑名单 guest-shutdown |
| 信息泄露 | 限制访问通道权限 |

---

## 20. 与 Host 侧 QEMU 的关系

### 20.1 Host 侧 virtio-serial

```c
// hw/char/virtio-serial-bus.c:263-345
// 主机 QEMU 进程中的 virtio-serial 设备模型
// 将 chardev 后端 (socket/file) 连接到 virtio-serial 端口
// Guest 写入 → virtio I/O → chardev 后端 → socket 输出
// Socket 输入 → chardev → virtio I/O → Guest 可读
```

### 20.2 libvirt 集成

```xml
<!-- libvirt domain XML -->
<channel type='unix'>
  <source mode='bind' path='/var/lib/libvirt/qemu/domain-guest-agent.sock'/>
  <target type='virtio' name='org.qemu.guest_agent.0'/>
</channel>
```

libvirt 通过 `virDomainQemuAgentCommand()` API 与 QGA 交互。

### 20.3 典型管理操作

| 操作 | QGA 命令 | 用途 |
|------|---------|------|
| 一致性快照 | guest-fsfreeze-freeze → 快照 → guest-fsfreeze-thaw | 数据安全 |
| 获取 IP | guest-network-get-interfaces | 网络管理 |
| 文件传输 | guest-file-open/write/close | 配置注入 |
| 密码重置 | guest-set-user-password | 安全管理 |
| 优雅关机 | guest-shutdown | 安全关闭 |
| 时间同步 | guest-set-time | 快照恢复后 |
| CPU 在线 | guest-set-vcpus | 热插拔配合 |
| 执行脚本 | guest-exec | 自动化运维 |

### 20.4 与 QMP 的区别

| 方面 | QMP | QGA |
|------|-----|-----|
| 运行位置 | Host QEMU 进程内 | Guest 内独立进程 |
| 管理对象 | 虚拟机硬件/配置 | Guest OS |
| 传输 | Unix socket/TCP | virtio-serial/ISA/vsock |
| 能力 | 热插拔/迁移/快照 | 文件/进程/网络/FS |
| 可用性 | QEMU 运行即可用 | 需要安装 QGA |

---

## 附录 A: 完整命令列表

### 通用命令 (commands.c)
| 命令 | 功能 |
|------|------|
| guest-sync | 同步通信 |
| guest-sync-delimited | 带定界符同步 |
| guest-ping | 连通性检查 |
| guest-info | 代理信息和支持的命令列表 |
| guest-exec | 执行命令 |
| guest-exec-status | 查询执行状态 |

### POSIX 命令 (commands-posix.c)
| 命令 | 功能 |
|------|------|
| guest-shutdown | 关机/重启/停止 |
| guest-set-time | 设置系统时间 |
| guest-file-open/close/read/write/seek/flush | 文件操作 |
| guest-fsfreeze-status/freeze/thaw | 文件系统冻结 |
| guest-network-get-interfaces | 网络接口信息 |
| guest-set-user-password | 设置用户密码 |
| guest-get-osinfo | 操作系统信息 |

### Linux 命令 (commands-linux.c)
| 命令 | 功能 |
|------|------|
| guest-fstrim | TRIM/UNMAP |
| guest-suspend-disk/ram/hybrid | 挂起 |
| guest-get-vcpus / guest-set-vcpus | CPU 在线管理 |
| guest-get-memory-blocks / guest-set-memory-blocks | 内存在线管理 |
| guest-get-memory-block-info | 内存块大小 |
| guest-get-disks | 磁盘信息 |
| guest-get-fsinfo | 文件系统信息 |
| guest-get-diskstats | 磁盘统计 |
| guest-get-cpustats | CPU 统计 |
| guest-network-get-route | 路由表 |

---

## 附录 B: 源码文件索引

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| `qga/main.c` | ~1700 | 入口、初始化、事件循环、消息派发、守护进程化 |
| `qga/commands.c` | ~630 | 通用命令 (sync/ping/info/exec/file-read) |
| `qga/commands-posix.c` | ~1200 | POSIX 命令 (shutdown/file/fsfreeze/network) |
| `qga/commands-linux.c` | ~2250 | Linux 命令 (vcpu/memory/disk/suspend/fstrim) |
| `qga/channel-posix.c` | ~260 | POSIX 通道实现 (virtio-serial/ISA/unix/vsock) |
| `qga/guest-agent-core.h` | 62 | 核心数据结构声明 |
| `qga/qapi-schema.json` | ~1000+ | QAPI 命令定义 |
| `hw/char/virtio-serial-bus.c` | ~1000 | Host 侧 virtio-serial 设备模型 |
