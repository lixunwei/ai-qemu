# Monitor/QMP/QAPI 深度分析 — 命令分发、协议协商、Schema 代码生成、事件系统与 Visitor 模式

> **基于 QEMU 11.0.50 源码**，深入分析 Monitor 子系统全栈：
> HMP/QMP 双协议架构、QMP 会话生命周期、命令注册与分发、
> QAPI Schema 定义与代码生成管线、事件广播与限流、Visitor 序列化模式。

---

## 目录

1. [Monitor 子系统全景](#1-monitor-子系统全景)
2. [核心数据结构](#2-核心数据结构)
3. [Monitor 初始化与生命周期](#3-monitor-初始化与生命周期)
4. [QMP 协议会话](#4-qmp-协议会话)
5. [QMP 命令分发路径](#5-qmp-命令分发路径)
6. [QMP 命令注册表](#6-qmp-命令注册表)
7. [HMP 命令系统](#7-hmp-命令系统)
8. [QAPI Schema 体系](#8-qapi-schema-体系)
9. [QAPI 代码生成管线](#9-qapi-代码生成管线)
10. [生成产物详解](#10-生成产物详解)
11. [Marshal 函数工作原理](#11-marshal-函数工作原理)
12. [事件系统](#12-事件系统)
13. [Visitor 模式](#13-visitor-模式)
14. [OOB 带外命令](#14-oob-带外命令)
15. [数据流全景图](#15-数据流全景图)

---

## 1. Monitor 子系统全景

QEMU Monitor 是管理和控制虚拟机的核心接口，支持两种协议：

| 协议 | 全称 | 用途 | 格式 |
|------|------|------|------|
| **HMP** | Human Monitor Protocol | 人工交互（命令行） | 文本行，readline |
| **QMP** | QEMU Machine Protocol | 机器/管理工具交互 | JSON-RPC 变体 |

**核心设计原则**：HMP 内部调用 QMP 实现 — 所有功能通过 QMP 暴露，HMP 仅是人类友好的包装层。

**关键源文件：**

| 文件 | 行数 | 职责 |
|------|------|------|
| `monitor/monitor-internal.h` | ~210 | Monitor/MonitorHMP/MonitorQMP 结构定义 |
| `monitor/monitor.c` | ~770 | Monitor 全局管理、事件限流、初始化 |
| `monitor/qmp.c` | ~490 | QMP 协议处理、协程调度器、JSON 解析 |
| `monitor/qmp-cmds-control.c` | ~110 | qmp_capabilities/query_version 实现 |
| `monitor/qmp-cmds.c` | ~200 | 核心 QMP 命令实现 |
| `monitor/hmp.c` | ~320 | HMP 命令解析与分发 |
| `monitor/hmp-cmds.c` | ~210 | HMP 命令包装函数 |
| `qapi/qmp-dispatch.c` | ~310 | QMP 命令分发核心 |
| `qapi/qmp-registry.c` | ~95 | 命令注册表 |
| `include/qapi/qmp-registry.h` | ~67 | QmpCommand/QmpCommandList 定义 |
| `scripts/qapi/main.py` | ~110 | QAPI 代码生成入口 |
| `qapi/qapi-schema.json` | ~80 | QAPI 主 Schema（include 汇总） |

---

## 2. 核心数据结构

### 2.1 Monitor 基类

```c
// monitor/monitor-internal.h:104-128
struct Monitor {
    CharFrontend chr;           // 字符设备前端（连接 chardev）
    int suspend_cnt;            // 挂起计数（原子访问）
    bool is_qmp;                // true=QMP, false=HMP
    bool skip_flush;
    bool use_io_thread;         // 是否使用 IOThread

    char *mon_cpu_path;         // 当前选中的 CPU 路径
    QTAILQ_ENTRY(Monitor) entry; // 全局 monitor 链表

    QemuMutex mon_lock;         // 每 monitor 锁

    // mon_lock 保护的成员：
    QLIST_HEAD(, mon_fd_t) fds; // 传递的文件描述符列表
    GString *outbuf;            // 输出缓冲区
    guint out_watch;            // GSource watch
    int mux_out;
    int reset_seen;
};
```

### 2.2 MonitorHMP — 人工交互

```c
// monitor/monitor-internal.h:130-141
struct MonitorHMP {
    Monitor common;             // 基类嵌入
    bool use_readline;
    ReadLineState *rs;          // readline 状态机
};
```

### 2.3 MonitorQMP — 机器协议

```c
// monitor/monitor-internal.h:143-163
typedef struct {
    Monitor common;                    // 基类嵌入
    JSONMessageParser parser;          // JSON 增量解析器
    bool pretty;                       // 美化 JSON 输出

    // 命令集指针 — 状态机核心：
    // 协商阶段 → &qmp_cap_negotiation_commands
    // 正常阶段 → &qmp_commands
    const QmpCommandList *commands;

    bool capab_offered[QMP_CAPABILITY__MAX];  // 服务端提供的能力
    bool capab[QMP_CAPABILITY__MAX];          // 协商后启用的能力

    QemuMutex qmp_queue_lock;          // 请求队列锁
    GQueue *qmp_requests;              // 已解析的 QMP 请求队列
} MonitorQMP;
```

### 2.4 HMPCommand — HMP 命令表项

```c
// monitor/monitor-internal.h:71-102
typedef struct HMPCommand {
    const char *name;           // 命令名
    const char *args_type;      // 参数类型描述串（如 "F:filename,i:count"）
    const char *params;         // 帮助中显示的参数
    const char *help;           // 帮助文本
    const char *flags;          // "p"=preconfig 阶段可用
    void (*cmd)(Monitor *mon, const QDict *qdict);  // 命令处理函数

    // 无参数简单输出命令的快捷方式：
    HumanReadableText *(*cmd_info_hrt)(Error **errp);

    struct HMPCommand *sub_table;  // 二级子命令表
    void (*command_completion)(ReadLineState *rs, int nb_args, const char *str);

    uint32_t arch_bitmask;      // QEMU_ARCH_* 架构过滤
    bool coroutine;             // 是否在协程中执行
} HMPCommand;
```

### 2.5 QmpCommand — QMP 命令注册项

```c
// include/qapi/qmp-registry.h:20-42
typedef void (QmpCommandFunc)(QDict *, QObject **, Error **);

typedef enum QmpCommandOptions {
    QCO_NO_SUCCESS_RESP  = (1U << 0),  // 成功时不返回 {"return": {}}
    QCO_ALLOW_OOB        = (1U << 1),  // 允许带外执行
    QCO_ALLOW_PRECONFIG  = (1U << 2),  // preconfig 阶段可用
    QCO_COROUTINE        = (1U << 3),  // 在协程中执行
} QmpCommandOptions;

typedef struct QmpCommand {
    const char *name;           // 命令名（如 "query-version"）
    QmpCommandFunc *fn;         // 生成的 marshal 函数
    QmpCommandOptions options;  // 选项位掩码
    uint64_t features;          // 特性位（兼容策略）
    QTAILQ_ENTRY(QmpCommand) node;
    bool enabled;               // 是否启用
    const char *disable_reason; // 禁用原因
} QmpCommand;

typedef QTAILQ_HEAD(QmpCommandList, QmpCommand) QmpCommandList;
```

---

## 3. Monitor 初始化与生命周期

### 3.1 静态初始化

```c
// monitor/monitor.c:698-705
static void __attribute__((__constructor__(QEMU_CONSTRUCTOR_EARLY)))
monitor_init_static(void)
{
    qemu_mutex_init(&monitor_lock);
    coroutine_mon = g_hash_table_new(NULL, NULL);           // 协程→monitor 映射
    monitor_qapi_event_state = g_hash_table_new(            // 事件限流状态
        qapi_event_throttle_hash, qapi_event_throttle_equal);
}
```

### 3.2 全局初始化

```c
// monitor/monitor.c:707-716
void monitor_init_globals(void)
{
    // 创建 QMP 调度器协程，调度到 iohandler AioContext
    qmp_dispatcher_co = qemu_coroutine_create(
        monitor_qmp_dispatcher_co, NULL);
    aio_co_schedule(iohandler_get_aio_context(), qmp_dispatcher_co);
}
```

### 3.3 Monitor 实例创建

```c
// monitor/monitor.c:718-753
int monitor_init(MonitorOptions *opts, bool allow_hmp, Error **errp)
{
    chr = qemu_chr_find(opts->chardev);
    // 自动选择模式：allow_hmp ? READLINE : CONTROL
    switch (opts->mode) {
    case MONITOR_MODE_CONTROL:
        monitor_init_qmp(chr, opts->pretty, errp);   // QMP
        break;
    case MONITOR_MODE_READLINE:
        monitor_init_hmp(chr, true, errp);            // HMP
        break;
    }
}
```

**关键设计**：
- 每个 Monitor 绑定一个 `Chardev`（串口/stdio/socket/websocket）
- QMP monitor 创建时立即进入 **能力协商模式**
- 全局 `mon_list` 链表管理所有活跃 monitor

---

## 4. QMP 协议会话

### 4.1 会话状态机

```
客户端连接
    │
    ▼
┌───────────────────┐
│ 能力协商模式      │  commands = &qmp_cap_negotiation_commands
│ 仅接受:           │  服务端发送 greeting（版本+能力列表）
│ qmp_capabilities  │
└────────┬──────────┘
         │ qmp_capabilities 成功
         ▼
┌───────────────────┐
│ 正常命令模式      │  commands = &qmp_commands
│ 接受所有 QMP 命令 │  双向 JSON 通信
└───────────────────┘
```

### 4.2 连接建立 — Greeting

```c
// monitor/qmp.c:458-472
static void monitor_qmp_event(void *opaque, QEMUChrEvent event)
{
    switch (event) {
    case CHR_EVENT_OPENED:
        // 进入协商模式
        mon->commands = &qmp_cap_negotiation_commands;
        monitor_qmp_caps_reset(mon);
        // 发送 greeting
        data = qmp_greeting(mon);   // {"QMP": {"version": ..., "capabilities": [...]}}
        qmp_send_response(mon, data);
        break;
    ...
}
```

### 4.3 Greeting 构造

```c
// monitor/qmp.c:436-456
static QDict *qmp_greeting(MonitorQMP *mon)
{
    // 查询版本信息
    qmp_marshal_query_version(args, &ver, NULL);
    // 构建能力列表
    for (cap = 0; cap < QMP_CAPABILITY__MAX; cap++) {
        if (mon->capab_offered[cap]) {
            qlist_append_str(cap_list, QMPCapability_str(cap));
        }
    }
    return qdict_from_jsonf_nofail(
        "{'QMP': {'version': %p, 'capabilities': %p}}", ver, cap_list);
}
```

### 4.4 能力协商

```c
// monitor/qmp-cmds-control.c:72-93
void qmp_qmp_capabilities(bool has_enable, QMPCapabilityList *enable, Error **errp)
{
    if (mon->commands == &qmp_commands) {
        error_set(errp, ...);  // 已经协商完成，拒绝重复
        return;
    }
    if (!qmp_caps_accept(mon, enable, errp)) {
        return;                // 能力校验失败
    }
    mon->commands = &qmp_commands;  // 切换到正常命令集
}
```

**当前支持的能力**：仅 `oob`（Out-of-Band，带外命令）。

---

## 5. QMP 命令分发路径

### 5.1 完整数据流

```
                        chardev 读回调
                             │
                             ▼
              monitor_qmp_read()          [qmp.c:429-434]
              JSON 增量解析器 feed
                             │
                             ▼
              handle_qmp_command()        [qmp.c:365-427]
              ┌──────────────┼──────────────┐
              │              │              │
          OOB 命令      普通命令      解析错误
          立即分发      入队等待      错误响应
              │              │
              ▼              ▼
     monitor_qmp_dispatch()   qmp_dispatcher_co_wake()
              │                        │
              │                        ▼
              │         monitor_qmp_dispatcher_co()  [qmp.c:274-352]
              │         从队列取请求，协程上下文执行
              │              │
              └──────┬───────┘
                     ▼
            qmp_dispatch()              [qmp-dispatch.c:147-306]
            命令查找/校验/参数提取
                     │
                     ▼
            cmd->fn(args, &ret, &err)   // 生成的 qmp_marshal_xxx()
                     │
                     ▼
            构造响应 {"return": ...} 或 {"error": ...}
```

### 5.2 JSON 解析

```c
// monitor/qmp.c:429-434
static void monitor_qmp_read(void *opaque, const uint8_t *buf, int size)
{
    MonitorQMP *mon = opaque;
    json_message_parser_feed(&mon->parser, (const char *) buf, size);
    // 解析完成后回调 handle_qmp_command()
}
```

QEMU 使用增量 JSON 解析器 `JSONMessageParser`，支持不完整 JSON 的流式解析。

### 5.3 命令入队与调度

```c
// monitor/qmp.c:365-427 — handle_qmp_command()

// OOB 命令立即执行（不入队）：
if (qdict && qmp_is_oob(qdict)) {
    monitor_qmp_dispatch(mon, req);     // [qmp.c:379-391]
    return;
}

// 普通命令入队：
req_obj = g_new0(QMPRequest, 1);
req_obj->mon = mon;
req_obj->req = req;
g_queue_push_tail(mon->qmp_requests, req_obj);  // [qmp.c:422]

// 唤醒调度器协程：
qmp_dispatcher_co_wake();                        // [qmp.c:426]
```

### 5.4 协程调度器

```c
// monitor/qmp.c:274-352 — monitor_qmp_dispatcher_co()
void coroutine_fn monitor_qmp_dispatcher_co(void *data)
{
    while ((req_obj = monitor_qmp_dispatcher_pop_any()) != NULL) {
        mon = req_obj->mon;
        oob_enabled = qmp_oob_enabled(mon);

        // OOB 启用时提前恢复 monitor（允许并发 OOB 请求）
        if (oob_enabled && 队列接近满) {
            monitor_resume(&mon->common);
        }

        // 执行命令
        monitor_qmp_dispatch(mon, req_obj->req);     // [qmp.c:335]

        // OOB 禁用时才在此恢复（串行化）
        if (!oob_enabled) {
            monitor_resume(&mon->common);
        }
    }
}
```

### 5.5 核心分发函数

```c
// qapi/qmp-dispatch.c:147-306 — qmp_dispatch()
QDict *qmp_dispatch(const QmpCommandList *cmds, QObject *request,
                    bool allow_oob, Monitor *cur_mon)
{
    // 1. 校验 JSON 对象格式
    dict = qobject_to(QDict, request);
    qmp_dispatch_check_obj(dict, allow_oob, &err);    // [44-90]

    // 2. 提取命令名（"execute" 或 "exec-oob"）
    command = qdict_get_try_str(dict, "execute");      // [173]
    if (!command) {
        command = qdict_get_str(dict, "exec-oob");     // [177]
        oob = true;
    }

    // 3. 查找命令
    cmd = qmp_find_command(cmds, command);              // [180]

    // 4. 兼容策略检查
    compat_policy_input_ok(cmd->features, ...);        // [186-189]

    // 5. 启用/禁用检查
    if (!cmd->enabled) { error... }                    // [191-197]

    // 6. OOB 权限检查
    if (oob && !(cmd->options & QCO_ALLOW_OOB)) { ... } // [199-203]

    // 7. 提取参数
    args = qdict_get_qdict(dict, "arguments");         // [209-214]

    // 8. 调用 marshal 函数
    cmd->fn(args, &ret, &err);                         // [234/128]

    // 9. 构造响应
    // 成功: {"return": ret, "id": id}
    // 失败: {"error": {"class": "...", "desc": "..."}, "id": id}
}
```

---

## 6. QMP 命令注册表

### 6.1 注册函数

```c
// qapi/qmp-registry.c:18-33
void qmp_register_command(QmpCommandList *cmds, const char *name,
                          QmpCommandFunc *fn, QmpCommandOptions options,
                          uint64_t features)
{
    QmpCommand *cmd = g_malloc0(sizeof(*cmd));
    // QCO_COROUTINE 和 QCO_ALLOW_OOB 不兼容
    assert(!((options & QCO_COROUTINE) && (options & QCO_ALLOW_OOB)));
    cmd->name = name;
    cmd->fn = fn;
    cmd->enabled = true;
    cmd->options = options;
    cmd->features = features;
    QTAILQ_INSERT_TAIL(cmds, cmd, node);
}
```

### 6.2 查找与遍历

```c
// qapi/qmp-registry.c:35-45
const QmpCommand *qmp_find_command(const QmpCommandList *cmds, const char *name)
{
    QTAILQ_FOREACH(cmd, cmds, node) {
        if (strcmp(cmd->name, name) == 0) return cmd;
    }
    return NULL;
}
```

注册表是简单的链表结构，命令查找是 O(n) 线性扫描。考虑到 QMP 命令总数约 200-300 个且查找不在热路径上，这是可接受的。

### 6.3 动态启用/禁用

```c
// qapi/qmp-registry.c:47-69
void qmp_disable_command(cmds, name, disable_reason);  // 禁用命令（如迁移期间禁用某些操作）
void qmp_enable_command(cmds, name);                   // 重新启用
```

---

## 7. HMP 命令系统

### 7.1 命令表定义

HMP 命令使用 `.hx` 格式定义，同时生成 C 代码和文档：

```
// hmp-commands.hx:13-19 示例
{
    .name       = "quit",
    .args_type  = "",
    .params     = "",
    .help       = "quit the emulator",
    .cmd        = hmp_quit,
},
```

两个 `.hx` 文件：
- `hmp-commands.hx` — 主命令表（quit/info/stop/cont/migrate 等）
- `hmp-commands-info.hx` — `info` 子命令表（info version/status/cpus 等）

### 7.2 HMP 包装 QMP

HMP 命令内部调用生成的 QMP 函数：

```c
// monitor/hmp-cmds.c 典型模式
void hmp_quit(Monitor *mon, const QDict *qdict)
{
    qmp_quit(NULL);    // 直接调用 QMP 实现
}

void hmp_cont(Monitor *mon, const QDict *qdict)
{
    Error *err = NULL;
    qmp_cont(&err);
    hmp_handle_error(mon, err);  // 错误转为文本输出到 monitor
}
```

### 7.3 HMP 反向调用

QMP 也可以执行 HMP 命令：

```c
// monitor/qmp-cmds.c:165-191
HumanReadableText *qmp_human_monitor_command(const char *command_line,
                                              bool has_cpu_index, ...)
{
    // 创建一个临时 HMP monitor
    // 在其上执行 HMP 命令
    // 捕获文本输出并返回
}
```

---

## 8. QAPI Schema 体系

### 8.1 Schema 组织

主入口 `qapi/qapi-schema.json` 通过 `include` 引入所有子模块：

```json
// qapi/qapi-schema.json — 子模块包含
{ 'include': 'pragma.json' }
{ 'include': 'error.json' }
{ 'include': 'common.json' }
{ 'include': 'control.json' }
{ 'include': 'machine.json' }
{ 'include': 'block-core.json' }
{ 'include': 'migration.json' }
{ 'include': 'net.json' }
{ 'include': 'ui.json' }
// ... 约 40 个 .json 子模块
```

总计 ~26,600 行 JSON Schema，定义 QEMU 的完整管理 API。

### 8.2 Schema 语法

**命令定义：**
```json
// qapi/control.json:38-40
{ 'command': 'qmp_capabilities',
  'data': { '*enable': [ 'QMPCapability' ] },
  'allow-preconfig': true }
```

**枚举定义：**
```json
// qapi/control.json:53-54
{ 'enum': 'QMPCapability',
  'data': [ 'oob' ] }
```

**结构体定义：**
```json
// qapi/control.json 中 VersionTriple
{ 'struct': 'VersionTriple',
  'data': { 'major': 'int', 'minor': 'int', 'micro': 'int' } }
```

**事件定义：**
```json
{ 'event': 'SHUTDOWN',
  'data': { 'guest': 'bool', 'reason': 'ShutdownCause' } }
```

**关键语法元素：**
| 元素 | 用途 |
|------|------|
| `'command'` | 定义 QMP 命令 |
| `'event'` | 定义 QMP 事件 |
| `'struct'` | 定义数据结构 |
| `'enum'` | 定义枚举 |
| `'union'` | 定义标签联合体 |
| `'alternate'` | 定义可选择类型 |
| `'include'` | 包含其他 schema 文件 |
| `'*field'` | 可选字段（前缀 `*`） |

### 8.3 主要 Schema 模块

| 模块 | 行数 | 覆盖领域 |
|------|------|----------|
| `block-core.json` | ~6000 | 块设备、快照、镜像、IO 限流 |
| `migration.json` | ~2500 | 迁移状态、参数、能力 |
| `machine.json` | ~2000 | 机器信息、CPU 拓扑、内存 |
| `ui.json` | ~1700 | VNC/Spice/Display |
| `net.json` | ~1200 | 网络设备、过滤器 |
| `control.json` | ~220 | QMP 协议控制 |
| `virtio.json` | ~1000 | VirtIO 设备内省 |

---

## 9. QAPI 代码生成管线

### 9.1 生成器入口

```python
# scripts/qapi/main.py:60-109
def main():
    parser = argparse.ArgumentParser(description='Generate code from a QAPI schema')
    parser.add_argument('-o', '--output-dir', ...)
    parser.add_argument('-p', '--prefix', ...)
    parser.add_argument('-B', '--backend', ...)
    args = parser.parse_args()

    schema = QAPISchema(args.schema)          # 解析 schema
    backend = create_backend(args.backend)    # 加载后端
    backend.generate(schema, ...)             # 生成代码
```

### 9.2 生成管线

```
qapi/*.json                     QAPI Schema 定义
    │
    ▼
scripts/qapi/schema.py          解析 JSON → 构建 IR（中间表示）
scripts/qapi/parser.py          JSON 解析
scripts/qapi/expr.py            表达式校验
    │
    ▼
scripts/qapi/commands.py        命令 marshal/unmarshal 生成
scripts/qapi/events.py          事件发送函数生成
scripts/qapi/types.py           C 类型/枚举/清理函数生成
scripts/qapi/visit.py           Visitor 函数生成
scripts/qapi/introspect.py      内省数据生成
    │
    ▼
build/qapi/                     生成的 C/H 文件
```

### 9.3 生成器脚本职责

| 脚本 | 行数 | 生成内容 |
|------|------|----------|
| `commands.py` | ~394 | `qmp_marshal_*()` 函数、`qmp_init_marshal()` 注册 |
| `events.py` | ~251 | `qapi_event_send_*()` 函数、事件枚举 |
| `types.py` | ~388 | C struct/enum 定义、`qapi_free_*()` 清理函数 |
| `visit.py` | ~428 | `visit_type_*()` 序列化/反序列化函数 |
| `introspect.py` | ~393 | QLit 内省数据（`query-qmp-schema` 的数据源） |
| `schema.py` | ~1500 | Schema IR 构建、校验、visitor 分发 |
| `gen.py` | ~260 | 代码生成框架（缩进/头文件/模块管理） |

---

## 10. 生成产物详解

### 10.1 文件组织

生成产物按 schema 模块组织：

```
build/qapi/
├── qapi-types.h/c              # 全局类型（non-module）
├── qapi-types-control.h/c      # control.json 的类型
├── qapi-types-machine.h/c      # machine.json 的类型
├── qapi-types-migration.h/c    # ...
├── qapi-visit-*.h/c            # 对应的 visitor
├── qapi-commands-*.h/c         # 对应的命令 marshal
├── qapi-events-*.h/c           # 对应的事件发送
├── qapi-introspect.h/c         # 内省数据
├── qapi-init-commands.h/c      # 命令注册初始化
├── qapi-emit-events.h/c        # 事件发射入口
├── qapi-builtin-types.h/c      # 内建类型（int/str/bool/...）
└── qapi-builtin-visit.h/c      # 内建类型 visitor
```

### 10.2 注册初始化

生成的 `qmp_init_marshal()` 在启动时被调用，将所有命令注册到全局命令列表：

```c
// 生成代码示例（概念性）
void qmp_init_marshal_control(QmpCommandList *cmds)
{
    qmp_register_command(cmds, "qmp_capabilities",
                         qmp_marshal_qmp_capabilities,
                         QCO_ALLOW_PRECONFIG, 0);
    qmp_register_command(cmds, "query-version",
                         qmp_marshal_query_version,
                         QCO_ALLOW_PRECONFIG, 0);
    // ...
}
```

---

## 11. Marshal 函数工作原理

Marshal 函数是 QAPI 代码生成的核心产物，负责 JSON ↔ C 类型转换：

### 11.1 典型 Marshal 函数结构

```c
// 生成的 qmp_marshal_query_version() 概念示例
static void qmp_marshal_query_version(QDict *args, QObject **ret, Error **errp)
{
    Visitor *v;
    VersionInfo *retval;

    // 1. 输入 visitor — 解析参数（query-version 无参数）
    v = qobject_input_visitor_new_qmp(QOBJECT(args));
    if (!visit_start_struct(v, NULL, NULL, 0, errp)) goto out;
    visit_check_struct(v, errp);
    visit_end_struct(v, NULL);
    visit_free(v);

    // 2. 调用实际实现
    retval = qmp_query_version(errp);
    if (*errp) goto out;

    // 3. 输出 visitor — 序列化返回值
    v = qobject_output_visitor_new_qmp(ret);
    visit_type_VersionInfo(v, "unused", &retval, NULL);
    visit_complete(v, ret);
    visit_free(v);

out:
    qapi_free_VersionInfo(retval);
}
```

### 11.2 数据流

```
QMP JSON 请求                       C 函数调用
{"execute": "query-version"}
         │                              │
         ▼                              ▼
  qmp_dispatch()                  qmp_query_version()
  查找命令 → marshal              实际业务逻辑
         │                              │
         ▼                              ▼
  Input Visitor                   VersionInfo *retval
  JSON → C args                        │
                                       ▼
                                 Output Visitor
                                 C struct → QObject
                                       │
                                       ▼
                              {"return": {"qemu": {"major":11,...}}}
```

---

## 12. 事件系统

### 12.1 事件定义与生成

Schema 定义：
```json
{ 'event': 'SHUTDOWN', 'data': { 'guest': 'bool', 'reason': 'ShutdownCause' } }
```

生成的发送函数：
```c
void qapi_event_send_shutdown(bool guest, ShutdownCause reason);
```

### 12.2 事件发射入口

```c
// monitor/monitor.c:401-440
void qapi_event_emit(QAPIEvent event, QDict *qdict)
{
    // 防重入：使用线程局部事件队列
    static __thread QSIMPLEQ_HEAD(, MonitorQapiEvent) event_queue;
    static __thread bool reentered;

    // 入队
    ev = g_new(MonitorQapiEvent, 1);
    ev->qdict = qobject_ref(qdict);
    ev->event = event;
    QSIMPLEQ_INSERT_TAIL(&event_queue, ev, entry);

    if (reentered) return;  // 递归调用时仅入队
    reentered = true;

    // 逐个处理队列
    while ((ev = QSIMPLEQ_FIRST(&event_queue)) != NULL) {
        QSIMPLEQ_REMOVE_HEAD(&event_queue, entry);
        monitor_qapi_event_queue_no_reenter(ev->event, ev->qdict);
        qobject_unref(ev->qdict);
        g_free(ev);
    }
    reentered = false;
}
```

### 12.3 事件限流

```c
// monitor/monitor.c:271-282
static MonitorQAPIEventConf monitor_qapi_event_conf[QAPI_EVENT__MAX] = {
    // Guest 可触发的事件限制为每秒 1 次
    [QAPI_EVENT_RTC_CHANGE]        = { 1000 * SCALE_MS },
    [QAPI_EVENT_BLOCK_IO_ERROR]    = { 1000 * SCALE_MS },
    [QAPI_EVENT_WATCHDOG]          = { 1000 * SCALE_MS },
    [QAPI_EVENT_BALLOON_CHANGE]    = { 1000 * SCALE_MS },
    [QAPI_EVENT_QUORUM_REPORT_BAD] = { 1000 * SCALE_MS },
    // ... 共 9 种限流事件
};
```

**限流状态机**：

```c
// monitor/monitor.c:42-47
typedef struct MonitorQAPIEventState {
    QAPIEvent event;      // 事件类型
    QDict *data;          // 限流键（用于区分同类型不同数据的事件）
    QEMUTimer *timer;     // 延迟发送定时器
    QDict *qdict;         // 缓存的延迟事件
} MonitorQAPIEventState;
```

限流逻辑（`monitor/monitor.c:328-398`）：
1. **无限流配置** → 直接广播
2. **有限流 + 首次** → 立即广播，启动定时器
3. **有限流 + 定时器内** → 缓存事件，等定时器触发
4. **定时器触发** → 发送缓存事件（如有），重置状态

### 12.4 事件广播

```c
// monitor/monitor.c:300-320
static void monitor_qapi_event_emit(QAPIEvent event, QDict *qdict)
{
    QTAILQ_FOREACH(mon, &mon_list, entry) {
        if (!monitor_is_qmp(mon)) continue;     // 仅广播给 QMP monitor
        qmp_mon = container_of(mon, MonitorQMP, common);
        // 跳过仍在协商阶段的 monitor
        if (qmp_mon->commands == &qmp_cap_negotiation_commands) continue;
        qmp_send_response(qmp_mon, qdict);      // 发送给每个活跃 QMP 客户端
    }
}
```

---

## 13. Visitor 模式

### 13.1 Visitor 架构

Visitor 是 QAPI 的核心序列化抽象，将 C 结构体与 QObject/JSON/字符串之间转换：

```
                    visit_type_Xxx()
                         │
            ┌────────────┼────────────┐
            │            │            │
   Input Visitor   Output Visitor  Dealloc Visitor
   JSON → C struct  C struct → JSON  释放 C struct
```

### 13.2 Input Visitor

```c
// qapi/qmp-dispatch.c:28-34
Visitor *qobject_input_visitor_new_qmp(QObject *obj)
{
    Visitor *v = qobject_input_visitor_new(obj);
    visit_set_policy(v, &compat_policy);  // 应用兼容策略
    return v;
}
```

`QObjectInputVisitor`（`qapi/qobject-input-visitor.c:33-57`）使用栈式遍历：
- `visit_start_struct()` → push QDict 到栈
- `visit_type_str()` → 从当前 QDict 取字符串
- `visit_check_struct()` → 检查未消费的成员（报错）
- `visit_end_struct()` → pop 栈

### 13.3 Output Visitor

```c
// qapi/qmp-dispatch.c:36-42
Visitor *qobject_output_visitor_new_qmp(QObject **result)
{
    Visitor *v = qobject_output_visitor_new(result);
    visit_set_policy(v, &compat_policy);
    return v;
}
```

`QObjectOutputVisitor`（`qapi/qobject-output-visitor.c:27-39`）构建 QObject 树：
- `visit_start_struct()` → 创建 QDict
- `visit_type_int()` → 向当前 QDict 插入 QNum
- `visit_complete()` → 返回根 QObject

### 13.4 其他 Visitor 类型

| Visitor | 文件 | 用途 |
|---------|------|------|
| **Dealloc** | `qapi-dealloc-visitor.c` | 释放 QAPI 生成的 C 结构体 |
| **Clone** | `qapi-clone-visitor.c` | 深拷贝 C 结构体 |
| **String Input** | `string-input-visitor.c` | 从字符串解析（用于 `-object` 参数） |
| **String Output** | `string-output-visitor.c` | 序列化为字符串 |
| **Opts** | `opts-visitor.c` | 从 QemuOpts 解析（命令行参数） |
| **Forward** | `qapi-forward-visitor.c` | 转发到另一个 visitor |

---

## 14. OOB 带外命令

### 14.1 设计动机

普通 QMP 命令是串行的 — 一个命令执行完才处理下一个。某些紧急命令（如 `stop`）需要在正常命令阻塞时也能执行。

### 14.2 OOB 协议

1. 客户端在 `qmp_capabilities` 中启用 OOB：
```json
{"execute": "qmp_capabilities", "arguments": {"enable": ["oob"]}}
```

2. 使用 `exec-oob` 代替 `execute` 发送带外命令：
```json
{"exec-oob": "stop", "id": "urgent-1"}
```

### 14.3 实现细节

```c
// qapi/qmp-dispatch.c:107-111 — OOB 判定
bool qmp_is_oob(const QDict *dict)
{
    return qdict_haskey(dict, "exec-oob") && !qdict_haskey(dict, "execute");
}

// qapi/qmp-registry.c:24-25 — OOB 与协程互斥
assert(!((options & QCO_COROUTINE) && (options & QCO_ALLOW_OOB)));
```

OOB 命令直接在 chardev I/O 线程中执行（`monitor/qmp.c:379-391`），不经过协程调度队列。

---

## 15. 数据流全景图

```
                        ┌─────────────────────────┐
                        │    QAPI Schema (.json)   │
                        │  commands/events/types   │
                        └───────────┬─────────────┘
                                    │ scripts/qapi/*.py
                                    ▼
                     ┌──────────────────────────────┐
                     │   生成代码 (build/qapi/)      │
                     │  marshal/visit/types/events  │
                     └──────────────┬───────────────┘
                                    │ qmp_init_marshal()
                                    ▼
┌─────────┐     JSON        ┌────────────────┐    qmp_find    ┌──────────────┐
│  Client  │ ──────────────→│  QMP Monitor   │ ──────────────→│  Command     │
│ (libvirt │     chardev    │  (qmp.c)       │    命令查找    │  Registry    │
│  virsh)  │                │  JSON 解析     │               │  (链表)      │
└─────────┘                 │  协程调度      │               └──────┬───────┘
     ▲                      └────────┬───────┘                      │
     │                               │                              ▼
     │                               │              ┌───────────────────────────┐
     │         JSON 响应              │              │   qmp_marshal_xxx()       │
     │    {"return":...}             │              │   Input Visitor → C args  │
     │    {"error":...}              │              │   qmp_xxx() 业务逻辑      │
     │    {"event":...}              │              │   Output Visitor → JSON   │
     └───────────────────────────────┘              └───────────────────────────┘
                                    ▲
                                    │
                              qapi_event_emit()
                              事件广播 + 限流
```

**核心架构特点：**

1. **Schema 驱动** — 所有 QMP API 由 JSON Schema 定义，代码自动生成
2. **类型安全** — Visitor 模式确保 JSON ↔ C 类型转换的正确性
3. **协程调度** — 普通命令在协程中执行，避免阻塞事件循环
4. **OOB 旁路** — 紧急命令绕过队列直接执行
5. **事件限流** — Guest 可触发的事件有速率限制，防止客户端被淹没
6. **HMP 兼容** — HMP 命令内部调用 QMP 实现，保证功能一致性
