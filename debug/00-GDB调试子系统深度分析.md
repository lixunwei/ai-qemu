# QEMU GDB 调试子系统深度分析

> 基于 QEMU 11.0.50 源码，commit 6511d4eed7（gdbstub 最新提交）  
> 重点关注 ARM64 (AArch64) 架构下的 GDB 调试支持  
> 交叉参考：[ARM64-CPU-GICv3-TCG深度分析](../arm64/00-ARM64-CPU-GICv3-TCG深度分析.md)、[模拟执行循环与MMIO分发深度分析](../architecture/02-模拟执行循环与MMIO分发深度分析.md)

---

## 目录

1. [概述与架构总览](#1-概述与架构总览)
2. [源文件组织](#2-源文件组织)
3. [GDB 远程串行协议 (RSP) 基础](#3-gdb-远程串行协议-rsp-基础)
4. [核心数据结构](#4-核心数据结构)
5. [GDB Server 启动与连接建立](#5-gdb-server-启动与连接建立)
6. [连接生命周期](#6-连接生命周期)
7. [RSP 数据包接收状态机](#7-rsp-数据包接收状态机)
8. [RSP 命令分发与处理](#8-rsp-命令分发与处理)
9. [寄存器读写机制](#9-寄存器读写机制)
10. [ARM64 寄存器映射详解](#10-arm64-寄存器映射详解)
11. [XML 目标描述系统](#11-xml-目标描述系统)
12. [内存读写操作](#12-内存读写操作)
13. [断点与监视点](#13-断点与监视点)
14. [单步执行机制](#14-单步执行机制)
15. [多核调试（多线程映射）](#15-多核调试多线程映射)
16. [vCont 高级执行控制](#16-vcont-高级执行控制)
17. [异常/信号报告](#17-异常信号报告)
18. [Monitor 命令转发 (qRcmd)](#18-monitor-命令转发-qrcmd)
19. [物理内存模式](#19-物理内存模式)
20. [KVM 模式下的 GDB 调试](#20-kvm-模式下的-gdb-调试)
21. [System 模式 vs User 模式差异](#21-system-模式-vs-user-模式差异)
22. [qXfer 扩展查询](#22-qxfer-扩展查询)
23. [完整 GDB 调试会话流程](#23-完整-gdb-调试会话流程)
24. [TCG 与 KVM 调试能力对比](#24-tcg-与-kvm-调试能力对比)
25. [ARM64 SVE/SME/MTE GDB 扩展](#25-arm64-svesme-mte-gdb-扩展)
26. [性能考虑与限制](#26-性能考虑与限制)

**附录**
- [附录 A：RSP 命令完整映射表](#附录-a-rsp-命令完整映射表)
- [附录 B：ARM64 GDB XML 描述文件清单](#附录-b-arm64-gdb-xml-描述文件清单)
- [附录 C：关键 Git 提交历史](#附录-c-关键-git-提交历史)

---

## 1. 概述与架构总览

QEMU 内置完整的 GDB 远程调试桩（GDB stub），实现了 GDB 远程串行协议（Remote Serial Protocol, RSP），允许外部 GDB 客户端连接到 QEMU 虚拟机进行源码级调试。

### 1.1 整体架构

```
+------------------+         RSP (TCP/Socket)        +------------------+
|   GDB Client     | <============================>  |   QEMU GDB Stub  |
|  (gdb/gdb-multiarch)                               |                  |
+------------------+                                  +--------+---------+
                                                               |
                              +--------------------------------+--------------------------------+
                              |                                |                                |
                    +---------v---------+          +-----------v-----------+          +----------v----------+
                    |  gdbstub/gdbstub.c |          |  gdbstub/system.c     |          |  gdbstub/user.c     |
                    |  (共享协议前端)     |          |  (系统模式后端)        |          |  (用户模式后端)      |
                    |  RSP 解析/分发      |          |  chardev 传输层        |          |  socket 传输层      |
                    |  寄存器/内存/断点   |          |  VM 状态联动           |          |  信号/fork 处理     |
                    +--------+-----------+          +-----------+-----------+          +----------+----------+
                             |                                  |                                 |
                    +--------v-----------+          +-----------v-----------+          +----------v----------+
                    | target/arm/        |          |  AccelOpsClass        |          | user-target.c       |
                    | gdbstub64.c        |          |  (TCG/KVM 后端)       |          | (auxv/exec-file)    |
                    | gdbstub.c          |          |  断点/单步/调试寄存器  |          |                     |
                    +--------------------+          +-----------------------+          +---------------------+
```

### 1.2 两种工作模式

| 特性 | System 模式 (softmmu) | User 模式 (linux-user) |
|------|----------------------|----------------------|
| 调试对象 | 整个虚拟机（所有 CPU） | 单个用户态进程 |
| 传输层 | chardev（TCP/Unix/stdio） | 原生 socket |
| 内存访问 | 物理+虚拟地址空间 | 进程虚拟地址空间 |
| 多线程 | 每个 vCPU 一个 GDB 线程 | 进程内线程 |
| Monitor 集成 | 支持 qRcmd | 不支持 |
| 独有功能 | 物理内存模式、reverse exec | auxv/exec-file/qOffsets |

---

## 2. 源文件组织

### 2.1 核心文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `gdbstub/gdbstub.c` | 2,523 | RSP 协议前端：数据包解析、命令分发、寄存器/内存操作 |
| `gdbstub/internals.h` | ~200 | 内部数据结构定义（GDBState、GDBProcess、RSState） |
| `gdbstub/system.c` | 675 | 系统模式后端：chardev 传输、VM 状态联动、断点代理 |
| `gdbstub/user.c` | 988 | 用户模式后端：socket 传输、信号处理、fork 跟踪 |
| `gdbstub/user-target.c` | 429 | 用户模式目标特定：auxv、exec-file、errno 转换 |
| `gdbstub/syscalls.c` | ~200 | File-I/O 系统调用半主机支持 |

### 2.2 ARM64 GDB 支持

| 文件 | 行数 | 职责 |
|------|------|------|
| `target/arm/gdbstub64.c` | 933 | AArch64 寄存器读写、SVE/SME/MTE/TLS/pauth 注册 |
| `target/arm/gdbstub.c` | 575 | ARM 通用 GDB 功能：feature 注册、sysreg 动态生成 |
| `target/arm/kvm.c` | 2,617 | KVM 模式调试：hw bp/wp 同步、guest debug 设置 |

### 2.3 公共头文件

| 文件 | 职责 |
|------|------|
| `include/exec/gdbstub.h` | GDB stub 公共 API |
| `include/gdbstub/commands.h` | 命令解析框架（GdbCmdParseEntry、schema 定义） |
| `include/gdbstub/enums.h` | 断点类型枚举（GDB_BREAKPOINT_SW/HW/etc.） |
| `include/gdbstub/helpers.h` | 寄存器读写辅助函数（gdb_get_reg64 等） |

### 2.4 XML 描述文件

```
gdbstub/gdb-xml/
├── aarch64-core.xml      # AArch64 核心寄存器 (X0-X30, SP, PC, PSTATE)
├── aarch64-fpu.xml       # FPU/NEON (V0-V31, FPSR, FPCR)
├── aarch64-mte.xml       # 内存标签扩展
├── aarch64-pauth.xml     # 指针认证
├── aarch64-sme2.xml      # SME2 可伸缩矩阵扩展
├── arm-core.xml          # ARM32 核心
├── arm-neon.xml          # ARM32 NEON
├── arm-vfp.xml           # VFP 浮点
└── ... (共 30+ 个架构 XML)
```

---

## 3. GDB 远程串行协议 (RSP) 基础

### 3.1 数据包格式

```
$packet-data#checksum
```

- `$` — 包起始标记
- `packet-data` — 命令数据（ASCII 编码）
- `#` — 校验和分隔符
- `checksum` — 两位十六进制校验和（所有 packet-data 字节之和的低 8 位）

### 3.2 确认机制

```
GDB → QEMU:  $m0,4#xx       (读内存)
QEMU → GDB:  +               (ACK 确认)
QEMU → GDB:  $deadbeef#yy   (返回数据)
GDB → QEMU:  +               (ACK 确认)
```

- `+` — 正确接收（ACK）
- `-` — 校验失败，请求重传（NAK）

### 3.3 特殊编码

| 机制 | 说明 |
|------|------|
| 转义 `}` | `}` + (char XOR 0x20)，用于转义 `$`, `#`, `}`, `*` |
| RLE `*` | 行程长度编码：`*` + (重复次数 + 29)，最少重复 3 次 |
| 中断 `0x03` | Ctrl-C 中断正在运行的 VM（`gdbstub.c:2363`） |

### 3.4 最大包长度

```c
// internals.h:34
#define MAX_PACKET_LENGTH 131104  // 2 * 256 * 256 + 32
```

此值主要由 ARM SME ZA 存储寄存器大小决定（256×256 字节矩阵）。

---

## 4. 核心数据结构

### 4.1 GDBState — 全局连接状态

```c
// internals.h:69-93
typedef struct GDBState {
    bool init;                    // 是否已初始化
    CPUState *c_cpu;              // 当前 step/continue 操作的 CPU
    CPUState *g_cpu;              // 当前寄存器/内存操作的 CPU  
    CPUState *query_cpu;          // 线程枚举用 CPU
    enum RSState state;           // RSP 解析状态机
    char line_buf[MAX_PACKET_LENGTH]; // 数据包接收缓冲区
    int line_buf_index;           // 缓冲区写入位置
    int line_sum;                 // 运行校验和
    int line_csum;                // 预期校验和
    GByteArray *last_packet;      // 最后发送的包（用于重传）
    int signal;                   // 当前信号值
    bool multiprocess;            // 多进程模式
    GDBProcess *processes;        // 进程数组
    int process_num;              // 进程数量
    GString *str_buf;             // 临时字符串缓冲区
    GByteArray *mem_buf;          // 内存操作缓冲区
    int sstep_flags;              // 单步标志
    int supported_sstep_flags;    // 加速器支持的单步标志
    bool allow_stop_reply;        // 是否允许发送停止回复
} GDBState;
```

全局实例：`gdbserver_state`（`gdbstub.c:59`）。

### 4.2 GDBProcess — 进程表示

```c
// internals.h:53-57
typedef struct GDBProcess {
    uint32_t pid;           // 进程 ID
    bool attached;          // 是否已附加
    char *target_xml;       // 缓存的目标 XML 描述
} GDBProcess;
```

System 模式下，每个 CPU 集群（`CPUCluster`）映射为一个 GDB 进程。

### 4.3 GDBRegisterState — 寄存器集合

```c
// gdbstub.c:52-57
typedef struct GDBRegisterState {
    int base_reg;                // 基准寄存器编号
    gdb_get_reg_cb get_reg;      // 读回调
    gdb_set_reg_cb set_reg;      // 写回调
    const GDBFeature *feature;   // XML 特性描述
} GDBRegisterState;
```

每个 CPU 的 `cpu->gdb_regs`（`GArray`）存储多个 `GDBRegisterState`，按注册顺序排列。

### 4.4 GDBSystemState — 系统模式状态

```c
// system.c:36-39
typedef struct {
    CharFrontend chr;       // chardev 前端（到 GDB 的传输通道）
    Chardev *mon_chr;       // Monitor 字符设备（用于 qRcmd）
} GDBSystemState;
```

### 4.5 RSState — 接收状态机枚举

```c
// internals.h:59-67
enum RSState {
    RS_INACTIVE,       // GDB 未连接
    RS_IDLE,           // 等待新包
    RS_GETLINE,        // 接收包数据
    RS_GETLINE_ESC,    // 转义序列
    RS_GETLINE_RLE,    // RLE 编码
    RS_CHKSUM1,        // 校验和高位
    RS_CHKSUM2,        // 校验和低位
};
```

### 4.6 命令解析框架

```c
// commands.h:60-67
typedef struct GdbCmdParseEntry {
    GdbCmdHandler handler;       // 处理函数
    const char *cmd;             // 命令字符串
    bool cmd_startswith;         // 前缀匹配
    const char *schema;          // 参数解析 schema
    bool allow_stop_reply;       // 是否允许停止回复
    bool need_cpu_context;       // 是否需要 CPU 上下文
} GdbCmdParseEntry;
```

Schema 语法（`commands.h:39-51`）：
- `l` → unsigned long，`L` → unsigned long long
- `s` → 字符串，`o` → 单字符，`t` → 线程 ID
- 分隔符：`?`（任意分隔）、`0`（字符串结束）、`.`（跳过一字符）

---

## 5. GDB Server 启动与连接建立

### 5.1 命令行参数

| 选项 | 说明 | 示例 |
|------|------|------|
| `-gdb DEV` | 指定 GDB 连接设备 | `-gdb tcp::1234` |
| `-s` | `-gdb tcp::1234` 的简写 | `-s` |
| `-S` | 启动时暂停（等待 GDB 连接） | `-S` |

### 5.2 System 模式启动流程

```
qemu_init() [vl.c]
  └─ gdbserver_start(gdbdev, &error_fatal) [system.c:333-411]
       │
       ├─ 1. 检查 first_cpu 存在 (line 339-343)
       ├─ 2. 检查加速器支持调试 gdb_supports_guest_debug() (line 345-349)
       ├─ 3. 创建 chardev: qemu_chr_new_noreplay("gdb", ...) (line 376)
       │     ├─ TCP: "tcp::1234,wait=off,nodelay=on,server=on"
       │     ├─ Unix: unix socket path
       │     └─ stdio: 设置 SIGINT handler
       ├─ 4. 初始化 GDBState (首次):
       │     ├─ gdb_init_gdbserver_state() (line 384)
       │     ├─ 注册 VM 状态变化回调 gdb_vm_state_change (line 386)
       │     └─ 创建 Monitor 字符设备 (line 389-391)
       ├─ 5. 创建进程表 create_processes() (line 398)
       ├─ 6. 绑定 chardev 回调:
       │     ├─ gdb_chr_can_receive  — 可接收字节数
       │     ├─ gdb_chr_receive      — 接收数据
       │     └─ gdb_chr_event        — 连接事件 (line 402-405)
       └─ 7. 设置初始状态 RS_IDLE (line 407)
```

### 5.3 GDB 客户端连接事件

```c
// system.c:85-106
static void gdb_chr_event(void *opaque, QEMUChrEvent event)
{
    switch (event) {
    case CHR_EVENT_OPENED:
        // 附加第一个进程
        for (i = 0; i < s->process_num; i++)
            s->processes[i].attached = !i;
        // 选择第一个 CPU
        s->c_cpu = gdb_first_attached_cpu();
        s->g_cpu = s->c_cpu;
        // 暂停虚拟机！
        vm_stop(RUN_STATE_PAUSED);
        break;
    }
}
```

**关键行为**：GDB 连接时 QEMU 自动暂停所有 vCPU（`vm_stop(RUN_STATE_PAUSED)`）。

### 5.4 User 模式启动

User 模式通过 `-g port` 参数启动（`linux-user/main.c:347-350`），在 `gdbstub/user.c:473-535` 创建原生 TCP socket，然后 `accept()` 等待 GDB 连接。

---

## 6. 连接生命周期

```
GDB 连接                    GDB 操作                     GDB 断开
   │                           │                           │
   v                           v                           v
CHR_EVENT_OPENED ──> vm_stop() ──> RSP 包循环 ──> Detach/Kill
   │                               │     │
   │  ┌────────────────────────────┘     │
   │  │                                  │
   │  v                                  v
   │  gdb_read_byte()                    handle_detach()
   │    → RS_IDLE                          → gdb_continue() [恢复 VM]
   │    → RS_GETLINE                       → reset_gdbserver_state()
   │    → RS_CHKSUM1/2                   handle_kill()
   │    → gdb_handle_packet()              → gdb_exit()
   │                                       → qemu_system_shutdown_request()
   v
'?' 包 → 发送初始停止状态 T05
```

### 6.1 Detach (`D` 命令)

```c
// gdbstub.c:1021-1059 handle_detach()
```

分离操作恢复 VM 运行，不终止 QEMU。

### 6.2 Kill (`k` 命令)

```c
// gdbstub.c:2118-2123
case 'k':
    error_report("QEMU: Terminated via GDBstub");
    gdb_exit(0);
    gdb_qemu_exit(0);  // → qemu_system_shutdown_request()
```

Kill 命令终止整个 QEMU 进程。

---

## 7. RSP 数据包接收状态机

`gdb_read_byte()`（`gdbstub.c:2330-2491`）是核心数据接收入口，实现完整的 RSP 解析状态机：

```
                    ┌──────────────┐
                    │  RS_IDLE     │<─────────────────────────────┐
                    │  等待 '$'    │                               │
                    └──────┬───────┘                               │
                     '$'   │                                       │
                    ┌──────v───────┐                               │
                    │ RS_GETLINE   │ ──── '}' ──> RS_GETLINE_ESC  │
                    │ 接收数据     │ ──── '*' ──> RS_GETLINE_RLE  │
                    │              │                               │
                    └──────┬───────┘                               │
                     '#'   │                                       │
                    ┌──────v───────┐                               │
                    │ RS_CHKSUM1   │                               │
                    │ 校验和高位   │                               │
                    └──────┬───────┘                               │
                           │                                       │
                    ┌──────v───────┐     校验失败: 发送 '-'        │
                    │ RS_CHKSUM2   │────────────────────────────────┘
                    │ 校验和低位   │
                    └──────┬───────┘
                           │ 校验成功: 发送 '+'
                    ┌──────v───────┐
                    │gdb_handle_   │
                    │packet()      │
                    └──────────────┘
```

### 7.1 Ctrl-C 中断处理

```c
// gdbstub.c:2355-2368
if (runstate_is_running()) {
    // VM 正在运行时收到字符 → 只处理 0x03 (Ctrl-C)
    if (ch != 0x03) {
        trace_gdbstub_err_unexpected_runpkt(ch);
    } else {
        gdbserver_state.allow_stop_reply = true;
    }
    vm_stop(RUN_STATE_PAUSED);  // 暂停 VM
}
```

当 VM 运行时，GDB 发送 `0x03`（Ctrl-C）使 QEMU 暂停并发送停止回复。

---

## 8. RSP 命令分发与处理

### 8.1 主分发器

`gdb_handle_packet()`（`gdbstub.c:2062-2312`）根据包首字符分发到对应处理器：

| 命令 | Handler | 说明 | allow_stop_reply |
|------|---------|------|------------------|
| `!` | (直接回复OK) | 启用扩展模式 | 否 |
| `?` | `handle_target_halt` | 查询停止原因 | **是** |
| `c` | `handle_continue` | 继续执行 | **是** |
| `C` | `handle_cont_with_sig` | 带信号继续 | **是** |
| `s` | `handle_step` | 单步执行 | **是** |
| `b` | `handle_backward` | 反向执行(replay) | **是** |
| `k` | (内联) | 杀死目标 | 否 |
| `D` | `handle_detach` | 分离 | 否 |
| `g` | `handle_read_all_regs` | 读所有寄存器 | 否 |
| `G` | `handle_write_all_regs` | 写所有寄存器 | 否 |
| `m` | `handle_read_mem` | 读内存 | 否 |
| `M` | `handle_write_mem` | 写内存 | 否 |
| `p` | `handle_get_reg` | 读单个寄存器 | 否 |
| `P` | `handle_set_reg` | 写单个寄存器 | 否 |
| `Z` | `handle_insert_bp` | 插入断点/监视点 | 否 |
| `z` | `handle_remove_bp` | 删除断点/监视点 | 否 |
| `H` | `handle_set_thread` | 设置当前线程 | 否 |
| `T` | `handle_thread_alive` | 检查线程存活 | 否 |
| `q` | `handle_gen_query` | 通用查询 | 否 |
| `Q` | `handle_gen_set` | 通用设置 | 否 |
| `v` | `handle_v_commands` | v 命令族 | 视子命令 |
| `F` | `gdb_handle_file_io` | File-I/O 回复 | 否 |

### 8.2 命令解析机制

```c
// gdbstub.c:966-1003 process_string_cmd()
// 使用 GdbCmdParseEntry 表进行匹配和参数解析
```

`q`/`Q` 命令通过可扩展的查询/设置表处理，架构代码可通过 `gdb_extend_query_table()` / `gdb_extend_set_table()` 注入自定义命令。

---

## 9. 寄存器读写机制

### 9.1 寄存器编号分层

```
┌──────────────────────────────────────────────────────────────┐
│ GDB 寄存器编号空间                                           │
│                                                              │
│  0 ─── gdb_num_core_regs-1    核心寄存器 (cc->gdb_read_register)
│  │                                                           │
│  gdb_num_core_regs ───        协处理器/扩展寄存器             │
│  │                            (GDBRegisterState.get_reg)     │
│  gdb_num_regs-1               最后一个寄存器                  │
└──────────────────────────────────────────────────────────────┘
```

### 9.2 读寄存器流程

```c
// gdbstub.c:529-543
int gdb_read_register(CPUState *cpu, GByteArray *buf, int reg)
{
    if (reg < cpu->cc->gdb_num_core_regs) {
        // 核心寄存器：直接调用 CPU 类方法
        return cpu->cc->gdb_read_register(cpu, buf, reg);
    }
    // 扩展寄存器：遍历 gdb_regs 数组查找匹配的 GDBRegisterState
    for (i = 0; i < cpu->gdb_regs->len; i++) {
        r = &g_array_index(cpu->gdb_regs, GDBRegisterState, i);
        if (r->base_reg <= reg && reg < r->base_reg + r->feature->num_regs) {
            return r->get_reg(cpu, buf, reg - r->base_reg);
        }
    }
    return 0;
}
```

### 9.3 CPU 初始化时的寄存器注册

```c
// gdbstub.c:593-616 gdb_init_cpu()
void gdb_init_cpu(CPUState *cpu)
{
    const char *xmlfile = gdb_get_core_xml_file(cpu);
    cpu->gdb_regs = g_array_new(...);
    
    if (xmlfile) {
        // 从 XML 文件获取核心寄存器特性
        feature = gdb_find_static_feature(xmlfile);
        gdb_register_feature(cpu, 0, cc->gdb_read_register, 
                            cc->gdb_write_register, feature);
        cpu->gdb_num_regs = cpu->gdb_num_g_regs = feature->num_regs;
    }
}
```

### 9.4 协处理器注册

```c
// gdbstub.c:618-643
void gdb_register_coprocessor(CPUState *cpu, get_reg, set_reg, feature)
{
    // 检查重复注册
    for (i = 0; i < cpu->gdb_regs->len; i++) {
        if (s->feature == feature) return;  // 已注册
    }
    // 分配寄存器编号，追加到 gdb_regs
    base_reg = max(cpu->gdb_num_regs, feature->base_reg);
    gdb_register_feature(cpu, base_reg, get_reg, set_reg, feature);
    cpu->gdb_num_regs += feature->num_regs;
}
```

---

## 10. ARM64 寄存器映射详解

### 10.1 核心寄存器 (aarch64-core.xml)

```c
// gdbstub64.c:35-54 aarch64_cpu_gdb_read_register()
```

| GDB 编号 | 寄存器 | 大小 | 源码位置 |
|----------|--------|------|---------|
| 0-30 | X0-X30 | 64-bit | `env->xregs[n]` (line 40-43) |
| 31 | SP | 64-bit | `env->xregs[31]` (line 46) |
| 32 | PC | 64-bit | `env->pc` (line 48) |
| 33 | CPSR/PSTATE | 32-bit | `pstate_read(env)` (line 51) |

**注意**：PSTATE 读取为 32-bit（`gdb_get_reg32`），但源码注释提到"pstate is now a 64-bit value"，暗示未来可能扩展。

### 10.2 FPU/NEON 寄存器 (aarch64-fpu.xml)

```c
// gdbstub64.c:87-149 aarch64_gdb_get_fpu_reg() / aarch64_gdb_set_fpu_reg()
```

| 偏移编号 | 寄存器 | 大小 | 说明 |
|---------|--------|------|------|
| 0-31 | V0-V31 | 128-bit | `aa64_vfp_qreg(env, reg)`，LE 存储 |
| 32 | FPSR | 32-bit | `vfp_get_fpsr(env)` (line 99-100) |
| 33 | FPCR | 32-bit | `vfp_get_fpcr(env)` (line 102-103) |

### 10.3 SVE 寄存器

```c
// gdbstub64.c:151-252 aarch64_gdb_get_sve_reg() / aarch64_gdb_set_sve_reg()
```

| 偏移编号 | 寄存器 | 大小 | 说明 |
|---------|--------|------|------|
| 0-31 | Z0-Z31 | VL bits | 可伸缩向量寄存器 (line 156-167) |
| 32 | FPSR | 32-bit | 复用 FPU FPSR (line 168-169) |
| 33 | FPCR | 32-bit | 复用 FPU FPCR (line 170-171) |
| 34-49 | P0-P15 | VL/8 bits | 谓词寄存器 (line 172-183) |
| 50 | FFR | VL/8 bits | First Fault Register |
| 51 | VG | 64-bit | 向量粒度（只读）(line 182-190) |

### 10.4 ARM64 GDB 特性注册

```c
// gdbstub.c:523-575 arm_cpu_register_gdb_regs_for_features()
// → aarch64_cpu_register_gdb_regs_for_features() [gdbstub64.c:592-694]
```

注册顺序（按特性能力动态选择）：
1. **SVE** 或 **FPU** — 二选一（SVE 包含 FPU）
2. **pauth** — 指针认证寄存器
3. **MTE** — 内存标签扩展
4. **TLS** — 线程局部存储寄存器
5. **SME/SME2** — 可伸缩矩阵扩展
6. **sysreg** — 动态生成的系统寄存器 XML

---

## 11. XML 目标描述系统

### 11.1 目标描述组装

当 GDB 发送 `qXfer:features:read:target.xml:0,xxx` 请求时：

```c
// gdbstub.c:354-410 get_feature_xml()
// 动态组装 target.xml:
// 1. XML 头 + <target>
// 2. <architecture>aarch64</architecture>
// 3. 对每个 GDBRegisterState: <xi:include href="xxx.xml"/>
// 4. </target>
```

生成的 `target.xml` 示例：

```xml
<?xml version="1.0"?>
<!DOCTYPE target SYSTEM "gdb-target.dtd">
<target>
  <architecture>aarch64</architecture>
  <xi:include href="aarch64-core.xml"/>
  <xi:include href="aarch64-fpu.xml"/>
  <xi:include href="aarch64-pauth.xml"/>
  <xi:include href="aarch64-mte.xml"/>
  <xi:include href="org.gnu.gdb.aarch64.sysregs.xml"/>
</target>
```

### 11.2 XML 文件服务

GDB 通过 `qXfer:features:read:FILENAME:OFFSET,LENGTH` 逐步请求各 XML 文件。QEMU 在 `get_feature_xml()` 中查找匹配的 `GDBRegisterState.feature->xmlname`，返回对应的 XML 内容。

### 11.3 动态系统寄存器 XML

ARM64 的系统寄存器不使用静态 XML，而是在运行时动态生成：

```c
// gdbstub.c:558-559
gdb_register_coprocessor(cs, arm_gdb_get_sysreg, arm_gdb_set_sysreg,
                         arm_gen_dynamic_sysreg_feature(cs, cs->gdb_num_regs));
```

`arm_gen_dynamic_sysreg_feature()` 遍历 CPU 的 cpreg 列表，为每个可通过 GDB 访问的系统寄存器生成 XML 条目。

### 11.4 qSupported 特性协商

```c
// gdbstub.c:1678-1723 handle_query_supported()
```

QEMU 报告支持的特性：
- `xmlRegisters=i386`（在 x86 上）
- `qXfer:features:read+`
- `qXfer:auxv:read+`（user 模式）
- `qXfer:exec-file:read+`（user 模式）
- `multiprocess+`
- 架构特定特性（通过 `gdb_extend_qsupported_features()`）

---

## 12. 内存读写操作

### 12.1 GDB `m` 命令（读内存）

```
$m addr,length#xx
```

处理流程：

```
handle_read_mem() [gdbstub.c:1265-1297]
  └─ gdb_target_memory_rw_debug(cpu, addr, buf, len, false)
       │
       ├─ [物理内存模式] cpu_physical_memory_read(addr, buf, len)
       │                  → address_space_rw(&address_space_memory, ...)
       │
       └─ [虚拟内存模式]
            ├─ cpu->cc->memory_rw_debug()  (如果 CPU 类实现了)
            └─ cpu_memory_rw_debug()  [physmem.c:4030-4063]
                 ├─ cpu_get_phys_page_attrs_debug()  // 页表遍历
                 ├─ 虚拟→物理地址翻译
                 └─ address_space_rw(as, phys_addr, ...)
```

### 12.2 GDB `M` 命令（写内存）

```
$M addr,length:data#xx
```

与读操作对称，使用 `gdb_target_memory_rw_debug(cpu, addr, buf, len, true)`。

### 12.3 系统模式内存访问 (`gdb_target_memory_rw_debug`)

```c
// system.c:452-469
int gdb_target_memory_rw_debug(CPUState *cpu, hwaddr addr,
                               uint8_t *buf, int len, bool is_write)
{
    if (phy_memory_mode) {
        // 直接物理内存访问（绕过 MMU）
        cpu_physical_memory_write/read(addr, buf, len);
        return 0;
    }
    if (cpu->cc->memory_rw_debug) {
        // CPU 类自定义调试内存访问
        return cpu->cc->memory_rw_debug(cpu, addr, buf, len, is_write);
    }
    // 默认：通过 MMU 翻译
    return cpu_memory_rw_debug(cpu, addr, buf, len, is_write);
}
```

---

## 13. 断点与监视点

### 13.1 GDB 断点类型

```c
// include/gdbstub/enums.h:14-20
enum {
    GDB_BREAKPOINT_SW,       // Z0: 软件断点
    GDB_BREAKPOINT_HW,       // Z1: 硬件断点
    GDB_WATCHPOINT_WRITE,    // Z2: 写监视点
    GDB_WATCHPOINT_READ,     // Z3: 读监视点
    GDB_WATCHPOINT_ACCESS,   // Z4: 读写监视点
};
```

### 13.2 断点插入/删除

RSP 命令格式：`Z type,addr,kind` / `z type,addr,kind`

```c
// gdbstub.c:1166-1212 handle_insert_bp() / handle_remove_bp()
// → system.c:640-664
int gdb_breakpoint_insert(CPUState *cs, int type, vaddr addr, vaddr len)
{
    const AccelOpsClass *ops = cpus_get_accel();
    if (ops->insert_breakpoint)
        return ops->insert_breakpoint(cs, type, addr, len);
    return -ENOSYS;
}
```

断点操作通过 `AccelOpsClass` 抽象层，TCG 和 KVM 各有不同实现。

### 13.3 TCG 断点实现

```c
// accel/tcg/tcg-accel-ops.c:133-198
```

| 类型 | TCG 实现 |
|------|---------|
| SW 断点 (Z0) | `cpu_breakpoint_insert(cs, addr, BP_GDB, ...)` |
| HW 断点 (Z1) | 同 SW（TCG 不区分） |
| 写监视点 (Z2) | `cpu_watchpoint_insert(cs, addr, len, BP_GDB\|BP_MEM_WRITE, ...)` |
| 读监视点 (Z3) | `cpu_watchpoint_insert(cs, addr, len, BP_GDB\|BP_MEM_READ, ...)` |
| 访问监视点 (Z4) | `cpu_watchpoint_insert(cs, addr, len, BP_GDB\|BP_MEM_ACCESS, ...)` |

断点列表在 `cpu-common.c:391-458`，`BP_GDB` 标志的断点优先放在列表头部（`cpu-common.c:406-411`）。

监视点命中在 `accel/tcg/watchpoint.c:67-141` 检测，触发 `EXCP_DEBUG`。

### 13.4 KVM 断点实现

```c
// accel/kvm/kvm-all.c:3802-3925
```

| 类型 | KVM 实现 |
|------|---------|
| SW 断点 | 保存原始指令 → 写入断点指令 → `KVM_GUESTDBG_USE_SW_BP` |
| HW 断点/监视点 | 使用 ARM64 硬件调试寄存器（`DBGBCR/DBGBVR`、`DBGWCR/DBGWVR`）|

---

## 14. 单步执行机制

### 14.1 单步标志

```c
// gdbstub.c:70-77
gdbserver_state.supported_sstep_flags = accel_supported_gdbstub_sstep_flags();
gdbserver_state.sstep_flags = SSTEP_ENABLE | SSTEP_NOIRQ | SSTEP_NOTIMER;
gdbserver_state.sstep_flags &= gdbserver_state.supported_sstep_flags;
```

| 标志 | 含义 |
|------|------|
| `SSTEP_ENABLE` | 启用单步 |
| `SSTEP_NOIRQ` | 单步时不处理中断 |
| `SSTEP_NOTIMER` | 单步时不触发定时器 |

### 14.2 System 模式单步

```c
// system.c:558-603 gdb_continue_partial()
case 's':
    cpu_single_step(cpu, gdbserver_state.sstep_flags);
    cpu_resume(cpu);
    break;
case 'c':
    cpu_resume(cpu);
    break;
```

单步完成后，CPU 触发 `EXCP_DEBUG` 异常，VM 切换到 `RUN_STATE_DEBUG`，`gdb_vm_state_change()` 回调发送停止回复 `T05`。

### 14.3 TCG 单步

TCG 模式下 `cpu_single_step()` 设置 CPU 的单步标志，翻译引擎在生成每条指令后插入退出 TB 的逻辑，确保每条指令执行后返回主循环。

### 14.4 KVM 单步

```c
// accel/kvm/kvm-all.c:3821-3839
dbg->control |= KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_SINGLESTEP;
```

KVM 使用 `KVM_SET_GUEST_DEBUG` ioctl 的 `KVM_GUESTDBG_SINGLESTEP` 标志。

---

## 15. 多核调试（多线程映射）

### 15.1 CPU 到 GDB 线程映射

System 模式下，每个 vCPU 映射为一个 GDB 线程：

```c
// gdbstub.c:677-685 gdb_append_thread_id()
// 格式: p<pid>.<tid>  (多进程模式)
//       <tid>          (单进程模式)
```

### 15.2 进程创建

```c
// system.c:277-320 create_processes()
```

QEMU 遍历所有 `CPUCluster` QOM 对象，为每个集群创建一个 `GDBProcess`。没有归属集群的 CPU 归入默认进程。

### 15.3 线程枚举

```
GDB: $qfThreadInfo#xx          → 请求第一批线程列表
QEMU: $mp01.01,p01.02,...#xx   → 返回线程 ID 列表
GDB: $qsThreadInfo#xx          → 请求更多
QEMU: $l#xx                    → 列表结束
```

```c
// gdbstub.c:1594-1623 handle_query_thread_info()
```

### 15.4 线程选择

```
GDB: $Hg p01.02#xx    → 选择线程 p1.2 用于后续 g/G/m/M 操作
GDB: $Hc p01.01#xx    → 选择线程 p1.1 用于 c/s 操作
```

```c
// gdbstub.c:1114-1163 handle_set_thread()
// 'g' → 设置 gdbserver_state.g_cpu
// 'c' → 设置 gdbserver_state.c_cpu
```

---

## 16. vCont 高级执行控制

`vCont` 是现代 GDB 使用的高级执行控制命令，支持对不同线程执行不同操作。

### 16.1 格式

```
vCont[;action[:thread-id]]...
```

支持的 action：
- `c` — 继续
- `C sig` — 带信号继续
- `s` — 单步
- `S sig` — 带信号单步

### 16.2 实现

```c
// gdbstub.c:739-875 gdb_handle_vcont()
```

核心逻辑：
1. 为每个 CPU 分配 `newstates[]` 数组
2. 解析每个 `action:thread-id` 对，标记对应 CPU 的动作
3. 调用 `gdb_continue_partial(newstates)` 按照标记执行

```c
// system.c:558-603 gdb_continue_partial()
CPU_FOREACH(cpu) {
    switch (newstates[cpu->cpu_index]) {
    case 's':
        cpu_single_step(cpu, gdbserver_state.sstep_flags);
        cpu_resume(cpu);
        break;
    case 'c':
        cpu_resume(cpu);
        break;
    }
}
```

### 16.3 线程粒度控制

```
vCont;s:p1.1;c:p1.2    → CPU 0 单步，CPU 1 继续
vCont;s:p1.1;c          → CPU 0 单步，其余继续
vCont;c                 → 所有 CPU 继续
```

---

## 17. 异常/信号报告

### 17.1 VM 状态变化 → 停止回复

```c
// system.c:122-217 gdb_vm_state_change()
```

| RunState | GDB Signal | 说明 |
|----------|-----------|------|
| `RUN_STATE_DEBUG` | `SIGTRAP (5)` | 断点/监视点/单步命中 |
| `RUN_STATE_PAUSED` | `SIGINT (2)` | 用户暂停（Ctrl-C） |
| `RUN_STATE_SHUTDOWN` | `SIGQUIT (3)` | 关机 |
| `RUN_STATE_IO_ERROR` | `SIGSTOP (17)` | I/O 错误 |
| `RUN_STATE_WATCHDOG` | `SIGALRM (14)` | 看门狗 |
| `RUN_STATE_INTERNAL_ERROR` | `SIGABRT (6)` | 内部错误 |
| `RUN_STATE_FINISH_MIGRATE` | `SIGXCPU (24)` | 迁移完成 |

### 17.2 停止回复包格式

```
T AA thread:TTTT;                      — 普通停止
T 05 thread:p01.01;awatch:ADDR;       — 监视点命中（含地址）
W XX                                   — 进程退出
```

### 17.3 监视点命中报告

```c
// system.c:152-171
if (cpu->watchpoint_hit) {
    switch (cpu->watchpoint_hit->flags & BP_MEM_ACCESS) {
    case BP_MEM_READ:  type = "r"; break;    // rwatch
    case BP_MEM_ACCESS: type = "a"; break;   // awatch
    default: type = ""; break;               // watch (写)
    }
    g_string_printf(buf, "T%02xthread:%s;%swatch:%" VADDR_PRIx ";",
                    GDB_SIGNAL_TRAP, tid, type, cpu->watchpoint_hit->vaddr);
}
```

### 17.4 allow_stop_reply 机制

`allow_stop_reply`（`internals.h:88-93`）确保只在 GDB 期望停止回复时才发送：
- 只有标记了 `allow_stop_reply = true` 的命令（如 `c`、`s`、`?`）才能触发停止回复
- 发送停止回复后立即清零

---

## 18. Monitor 命令转发 (qRcmd)

System 模式独有功能，允许 GDB 执行 QEMU monitor 命令：

```
GDB: (gdb) monitor info registers
→ RSP: $qRcmd,<hex-encoded "info registers">#xx
```

```c
// system.c:512-535 gdb_handle_query_rcmd()
void gdb_handle_query_rcmd(GArray *params, void *ctx)
{
    // 1. 解码 hex 编码的命令字符串
    gdb_hextomem(gdbserver_state.mem_buf, data, len);
    // 2. 写入 Monitor 字符设备
    qemu_chr_be_write(gdbserver_system_state.mon_chr,
                      gdbserver_state.mem_buf->data, ...);
    gdb_put_packet("OK");
}
```

Monitor 字符设备在 `gdbserver_start()` 中创建（`system.c:389-391`），类型为 `TYPE_CHARDEV_GDB`，关联一个 HMP monitor 实例。

---

## 19. 物理内存模式

System 模式可切换为物理内存访问模式，绕过 MMU 页表翻译：

```
GDB: (gdb) monitor gdbstub phy_mem_mode 1
```

或通过 RSP 扩展命令：

```c
// system.c:490-509
// qemu.PhyMemMode query/set
static int phy_memory_mode;
```

启用后，GDB 的 `m/M` 命令直接使用 `cpu_physical_memory_read/write()`，适合调试内核页表、MMIO 等场景。

---

## 20. KVM 模式下的 GDB 调试

### 20.1 ARM64 Guest Debug 架构

```c
// target/arm/kvm.c:1615-1624
void kvm_arch_update_guest_debug(CPUState *cs, struct kvm_guest_debug *dbg)
{
    if (kvm_sw_breakpoints_active(cs)) {
        dbg->control |= KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_SW_BP;
    }
    if (kvm_arm_hw_debug_active(ARM_CPU(cs))) {
        dbg->control |= KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_HW;
        kvm_arm_copy_hw_debug_data(&dbg->arch);
    }
}
```

### 20.2 硬件调试寄存器同步

```c
// kvm.c:1598-1612 kvm_arm_copy_hw_debug_data()
// 将 QEMU 维护的 hw bp/wp 同步到 KVM:
//   dbg_bcr[i] / dbg_bvr[i] → 硬件断点控制/值寄存器
//   dbg_wcr[i] / dbg_wvr[i] → 硬件监视点控制/值寄存器
```

### 20.3 KVM Debug 控制标志

| 标志 | 说明 |
|------|------|
| `KVM_GUESTDBG_ENABLE` | 启用 guest 调试 |
| `KVM_GUESTDBG_USE_SW_BP` | 使用软件断点 |
| `KVM_GUESTDBG_USE_HW` | 使用硬件断点/监视点 |
| `KVM_GUESTDBG_SINGLESTEP` | 单步执行 |

### 20.4 KVM 软件断点

```c
// accel/kvm/kvm-all.c:3847-3922
// SW 断点工作原理：
// 1. 读取并保存断点地址原始指令
// 2. 写入断点指令（ARM64: BRK #0）
// 3. KVM 遇到断点 → VMEXIT → QEMU 报告给 GDB
// 4. 删除时恢复原始指令
```

---

## 21. System 模式 vs User 模式差异

### 21.1 架构对比

| 方面 | System 模式 | User 模式 |
|------|------------|----------|
| **传输层** | chardev (TCP/Unix/stdio) | 原生 socket |
| **状态类型** | `GDBSystemState` (system.c:36-39) | 内部结构 (user.c:31-107) |
| **VM 控制** | `vm_stop()` / `vm_start()` | 直接控制单进程 |
| **内存访问** | `gdb_target_memory_rw_debug()` | `/proc/self/mem` 回退 |
| **进程创建** | 按 CPUCluster 创建 | 单进程 |
| **信号处理** | RunState 映射 | 真实 POSIX 信号 |
| **qRcmd** | ✓ Monitor 命令 | ✗ |
| **qOffsets** | ✗ | ✓ 进程加载偏移 |
| **auxv** | ✗ | ✓ 辅助向量 |
| **exec-file** | ✗ | ✓ 可执行文件路径 |
| **物理内存** | ✓ `phy_mem_mode` | ✗ |

### 21.2 代码分割

共享前端 `gdbstub.c` 通过 `#ifdef CONFIG_USER_ONLY` 条件编译和函数指针实现模式分离。`system.c` 和 `user.c` 各自实现：
- `gdbserver_start()` — 不同的连接建立方式
- `gdb_continue()` / `gdb_continue_partial()` — 不同的执行恢复
- `gdb_breakpoint_insert/remove()` — 不同的断点后端
- 停止回复生成 — 不同的状态源

---

## 22. qXfer 扩展查询

### 22.1 支持的 qXfer 请求

| qXfer 类型 | 模式 | 说明 | 源码位置 |
|-----------|------|------|---------|
| `qXfer:features:read` | 两者 | XML 目标描述 | `gdbstub.c:1725-1772` |
| `qXfer:auxv:read` | User | 辅助向量 | `user-target.c:243-286` |
| `qXfer:exec-file:read` | User | 可执行文件路径 | `user-target.c:388-423` |
| `qXfer:siginfo:read` | User | 信号信息 | `gdbstub.c:1780-1810` |

### 22.2 其他查询命令

| 命令 | 模式 | 说明 | 源码位置 |
|------|------|------|---------|
| `qAttached` | System | 是否已附加 | `system.c:542-544` |
| `qC` | 两者 | 当前线程 ID | `gdbstub.c:1577-1592` |
| `qfThreadInfo` | 两者 | 线程枚举 | `gdbstub.c:1594-1623` |
| `qRcmd` | System | Monitor 命令 | `system.c:512-535` |
| `qOffsets` | User | 段偏移 | `user-target.c:217-230` |
| `qSupported` | 两者 | 特性协商 | `gdbstub.c:1678-1723` |
| `qemu.PhyMemMode` | System | 物理内存模式 | `system.c:490-509` |

---

## 23. 完整 GDB 调试会话流程

```
=== QEMU 启动 ===
$ qemu-system-aarch64 -M virt -cpu cortex-a72 -gdb tcp::1234 -S ...

=== GDB 连接 ===
$ gdb-multiarch vmlinux
(gdb) target remote :1234

=== RSP 交互时序 ===

GDB                              QEMU
 │                                 │
 │ TCP connect                     │ CHR_EVENT_OPENED
 │                                 │ vm_stop(RUN_STATE_PAUSED)
 │                                 │
 │ $qSupported:...#xx    ────>     │ 回复支持特性列表
 │ $Hg0#xx               ────>     │ 选择线程 0 (g_cpu)
 │ $?#xx                  ────>     │ T02thread:p01.01; (SIGINT)
 │                                 │
 │ $qXfer:features:read:          │
 │  target.xml:0,fff#xx  ────>     │ 返回 target.xml
 │ $qXfer:features:read:          │
 │  aarch64-core.xml#xx  ────>     │ 返回核心寄存器 XML
 │ $qXfer:features:read:          │
 │  aarch64-fpu.xml#xx   ────>     │ 返回 FPU 寄存器 XML
 │                                 │
 │ $g#xx                  ────>     │ 返回所有核心寄存器值
 │ $m ffff0000,4#xx       ────>     │ 返回内存内容
 │                                 │
 │ === 设置断点 ===                │
 │ $Z0,ffff800010001000,4 ────>    │ SW 断点 at 0xffff800010001000
 │                                 │ cpu_breakpoint_insert(BP_GDB)
 │                                 │ +OK
 │                                 │
 │ === 继续执行 ===                │
 │ $vCont;c:p1.1;c:p1.2  ────>    │ 所有 CPU 继续
 │                                 │ cpu_resume() × N
 │                                 │ ...运行中...
 │                                 │
 │                                 │ [命中断点]
 │                                 │ EXCP_DEBUG → RUN_STATE_DEBUG
 │                          <────  │ T05thread:p01.01; (SIGTRAP)
 │                                 │
 │ $p20#xx                ────>     │ 读 PC 寄存器
 │ $m addr,len#xx         ────>     │ 读内存（反汇编）
 │                                 │
 │ === 单步 ===                    │
 │ $vCont;s:p1.1;c        ────>    │ CPU 0 单步，其余继续
 │                                 │ cpu_single_step(sstep_flags)
 │                                 │ 执行一条指令
 │                          <────  │ T05thread:p01.01;
 │                                 │
 │ === 分离 ===                    │
 │ $D#xx                  ────>    │ handle_detach()
 │                                 │ gdb_continue() → vm_start()
 │                                 │ +OK
```

---

## 24. TCG 与 KVM 调试能力对比

| 能力 | TCG | KVM |
|------|-----|-----|
| 软件断点 | ✓ 无限制 | ✓ 通过指令替换 |
| 硬件断点 | ✓（映射为 SW） | ✓ DBGBCR/DBGBVR（有限数量） |
| 写监视点 | ✓ 无限制 | ✓ DBGWCR/DBGWVR（有限数量） |
| 读监视点 | ✓ 无限制 | ✓ 硬件支持 |
| 单步执行 | ✓ SSTEP_NOIRQ/NOTIMER | ✓ KVM_GUESTDBG_SINGLESTEP |
| 反向执行 | ✓ (replay 模式) | ✗ |
| 寄存器访问 | ✓ 直接内存 | ✓ 通过 KVM ioctl |
| 系统寄存器 | ✓ 完全访问 | 受限（部分只读） |
| 性能影响 | 断点/监视点影响翻译缓存 | 硬件辅助，影响较小 |

### 24.1 KVM 调试限制

1. **硬件断点/监视点数量有限**：ARM64 通常提供 4-16 个硬件断点和 4-16 个硬件监视点
2. **单步性能**：每次单步都需要 VMEXIT → 恢复，开销较大
3. **系统寄存器访问**：部分寄存器在 KVM 模式下只读或不可访问
4. **反向执行不支持**：KVM 不支持 replay/reverse 调试

---

## 25. ARM64 SVE/SME/MTE GDB 扩展

### 25.1 SVE (Scalable Vector Extension)

```c
// gdbstub64.c:592-694 aarch64_cpu_register_gdb_regs_for_features()
if (isar_feature_aa64_sve(&cpu->isar)) {
    gdb_register_coprocessor(cs, aarch64_gdb_get_sve_reg, 
                            aarch64_gdb_set_sve_reg, ...);
}
```

SVE 寄存器大小可变（128-2048 bit），通过 VG (Vector Granule) 伪寄存器报告当前宽度。

### 25.2 SME/SME2 (Scalable Matrix Extension)

```c
// gdbstub64.c 中注册 SME/SME2 寄存器
// XML: aarch64-sme2.xml
```

SME 引入 ZA 存储矩阵（最大 256×256 字节 = 64KB），这是 `MAX_PACKET_LENGTH` 设为 131104 的直接原因。

### 25.3 MTE (Memory Tagging Extension)

```c
// XML: aarch64-mte.xml
// 暴露 MTE 相关标签寄存器
```

### 25.4 Pointer Authentication (pauth)

```c
// XML: aarch64-pauth.xml
// 暴露指针认证密钥寄存器
```

### 25.5 TLS (Thread Local Storage)

```c
// gdbstub64.c 中动态注册 TLS 寄存器
// commit: 7fd82461dd "Implement org.gnu.gdb.aarch64.tls XML feature"
```

---

## 26. 性能考虑与限制

### 26.1 性能影响因素

| 因素 | 影响 | 缓解 |
|------|------|------|
| 断点数量 | TCG: 影响 TB 分割；KVM: 受硬件限制 | 尽量使用少量断点 |
| 单步频率 | 每步 VM exit + RSP 往返 | 使用 continue + 断点替代频繁单步 |
| 寄存器读取 | `g` 命令读所有寄存器开销大 | GDB 默认使用 `p` 按需读取 |
| 内存读取 | 每次需 MMU 翻译 | 使用物理内存模式减少翻译开销 |
| 包大小 | MAX_PACKET_LENGTH = 128KB+ | SME ZA 矩阵需要 |

### 26.2 已知限制

1. **All-stop 模式**：QEMU 仅支持 all-stop，不支持 non-stop 模式（所有 CPU 同时暂停/恢复）
2. **条件断点**：由 GDB 客户端实现（每次命中后 GDB 检查条件），非 QEMU 端
3. **进程创建/退出**：System 模式不支持 GDB 的 fork/exec 跟踪
4. **反向执行**：仅在 replay 模式下可用（`replay_mode == REPLAY_MODE_PLAY`）

---

## 附录 A：RSP 命令完整映射表

| 命令 | Schema | Handler | 位置 |
|------|--------|---------|------|
| `!` | — | (直接 OK) | `gdbstub.c:2069-2071` |
| `?` | — | `handle_target_halt` | `gdbstub.c:2072-2082` |
| `c[addr]` | `L0` | `handle_continue` | `gdbstub.c:2083-2094` |
| `C sig` | `l0` | `handle_cont_with_sig` | `gdbstub.c:2095-2106` |
| `D[;pid]` | `?.l0` | `handle_detach` | `gdbstub.c:2124-2134` |
| `F retcode,errno,Ctrl-C` | `L,L,o0` | `gdb_handle_file_io` | `gdbstub.c:2159-2169` |
| `g` | — | `handle_read_all_regs` | `gdbstub.c:2170-2178` |
| `G XX...` | `s0` | `handle_write_all_regs` | `gdbstub.c:2180-2190` |
| `Hc thread` | `o.t0` | `handle_set_thread` | `gdbstub.c:2257-2267` |
| `k` | — | (内联 kill) | `gdbstub.c:2118-2123` |
| `m addr,length` | `L,L0` | `handle_read_mem` | `gdbstub.c:2191-2201` |
| `M addr,length:XX` | `L,L:s0` | `handle_write_mem` | `gdbstub.c:2202-2212` |
| `p n` | `L0` | `handle_get_reg` | `gdbstub.c:2213-2222` |
| `P n=XX` | `L?s0` | `handle_set_reg` | `gdbstub.c:2224-2234` |
| `s[addr]` | `L0` | `handle_step` | `gdbstub.c:2135-2146` |
| `T thread` | `t0` | `handle_thread_alive` | `gdbstub.c:2268-2278` |
| `v...` | `s0` | `handle_v_commands` | `gdbstub.c:2107-2117` |
| `Z type,addr,kind` | `l?L?L0` | `handle_insert_bp` | `gdbstub.c:2235-2245` |
| `z type,addr,kind` | `l?L?L0` | `handle_remove_bp` | `gdbstub.c:2246-2256` |
| `q...` | `s0` | `handle_gen_query` | `gdbstub.c:2279-2289` |
| `Q...` | `s0` | `handle_gen_set` | `gdbstub.c:2290-2300` |
| `b` | `o0` | `handle_backward` | `gdbstub.c:2147-2158` |

### v 命令子表

| 子命令 | Handler | 说明 |
|--------|---------|------|
| `vCont` | `gdb_handle_vcont` | 高级执行控制 |
| `vCont?` | — | 查询 vCont 支持 |
| `vAttach;pid` | — | 附加到进程 |
| `vKill;pid` | — | 杀死进程 |
| `vFile:...` | — | File-I/O 操作 |

---

## 附录 B：ARM64 GDB XML 描述文件清单

| 文件 | 寄存器数 | 注册条件 |
|------|---------|---------|
| `aarch64-core.xml` | 34 (X0-X30, SP, PC, PSTATE) | 始终注册 |
| `aarch64-fpu.xml` | 34 (V0-V31, FPSR, FPCR) | 无 SVE 时 |
| SVE 动态生成 | 52+ (Z0-31, P0-15, FFR, VG, FPSR, FPCR) | `isar_feature_aa64_sve` |
| `aarch64-pauth.xml` | 5 | `isar_feature_aa64_pauth` |
| `aarch64-mte.xml` | 2 | `isar_feature_aa64_mte` + TCG |
| TLS 动态生成 | 1-3 | `isar_feature_aa64_tls` |
| `aarch64-sme2.xml` | 若干 | `isar_feature_aa64_sme2` |
| sysreg 动态生成 | 可变 | 始终注册（按 CPU cpreg 列表） |

---

## 附录 C：关键 Git 提交历史

### gdbstub/ 核心

| Commit | 说明 |
|--------|------|
| `6511d4eed7` | 生成单一 gdbstub-xml.c / gdb_static_features[] |
| `39fb349f74` | 将 gdb-xml/ 移入 gdbstub/ 目录 |
| `b3e88abb20` | GDBFeature::base_reg 在 gdb_register_coprocessor 中使用 |
| `aa6a508795` | 移除 gdb_register_coprocessor 的 @g_pos 参数 |
| `1e06ef47df` | 添加 XML 解析/生成的 trace events |
| `3b6cf87d42` | 简化 gdb_init_cpu() 逻辑 |

### ARM64 GDB 支持

| Commit | 说明 |
|--------|------|
| `05f32d2584` | SME 存在时报告正确的向量宽度 |
| `7e274980f2` | 提取 aarch64_cpu_register_gdb_regs_for_features |
| `7fd82461dd` | 实现 org.gnu.gdb.aarch64.tls XML feature |
| `841bb7d96f` | 实现 SME2 gdbstub 支持 |
| `de83d60a84` | 扩展 pstate 到 64 位 |
| `030f0ba117` | 添加 SME 寄存器暴露支持 |
| `97b3d732af` | 修复从 GDB 设置 SVE 寄存器 |
| `35cca0f95f` | 修复大端模式下 NEON GDB 远程调试 |

---

> **文档版本**：v1.0  
> **分析基准**：QEMU 11.0.50 (commit 6511d4eed7)  
> **分析工具**：zoekt + ctags + cscope 索引  
> **相关文档**：  
> - [ARM64-CPU-GICv3-TCG深度分析](../arm64/00-ARM64-CPU-GICv3-TCG深度分析.md)  
> - [TCG深度分析](../accel/00-TCG深度分析.md)  
> - [模拟执行循环与MMIO分发深度分析](../architecture/02-模拟执行循环与MMIO分发深度分析.md)
