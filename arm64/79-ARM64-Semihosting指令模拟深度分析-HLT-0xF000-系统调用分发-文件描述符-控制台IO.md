# ARM64 Semihosting 指令模拟深度分析

> 基于 QEMU 11.0.50 源码，分析 ARM Semihosting 完整实现：
> AArch64 陷入指令检测 (HLT #0xF000)、AArch32 陷入 (SVC #0x123456 / BKPT #0xAB)、
> 系统调用分发 (25 种操作)、Guest 文件描述符管理、控制台 I/O、
> GDB/Host 双后端、配置机制。
> 参考规范：ARM Semihosting for AArch32 and AArch64 Release 2.0

---

## 目录

1. [Semihosting 概述与规范](#1-semihosting-概述与规范)
2. [陷入指令检测](#2-陷入指令检测)
3. [调用分发主流程](#3-调用分发主流程)
4. [25 种 Semihosting 操作详解](#4-semihosting-操作详解)
5. [Guest 文件描述符管理](#5-guest-文件描述符管理)
6. [控制台 I/O](#6-控制台-io)
7. [GDB/Host 双后端](#7-gdbhost-双后端)
8. [配置与启用机制](#8-配置与启用机制)
9. [Target 适配层](#9-target-适配层)
10. [与硬件调试器的对比](#10-与硬件调试器的对比)

---

## 1. Semihosting 概述与规范

### 1.1 什么是 Semihosting

Semihosting 是 ARM 定义的一种调试辅助机制，允许运行在 target（裸机/RTOS）上的代码
通过特殊的陷入指令请求 host（调试器/模拟器）执行 I/O 操作，如文件读写、控制台输出、
获取命令行参数等。

典型使用场景：
- 裸机程序通过 `printf` → newlib `_write` → semihosting SYS_WRITE 输出到主机终端
- bootloader 通过 SYS_OPEN/SYS_READ 从主机加载文件
- 测试框架通过 SYS_EXIT 报告测试结果

### 1.2 规范文档

```
ARM Semihosting for AArch32 and AArch64 Release 2.0
https://github.com/ARM-software/abi-aa/blob/main/semihosting/semihosting.rst
```

### 1.3 陷入指令定义

| 状态 | 指令 | 编码 |
|------|------|------|
| AArch64 | `HLT #0xF000` | 本为外部 halting debug 指令 |
| AArch32 (ARM) | `SVC #0x123456` | supervisor call |
| AArch32 (Thumb) | `BKPT #0xAB` | breakpoint |
| M-Profile (Thumb) | `BKPT #0xAB` | breakpoint |

### 1.4 调用约定

| 状态 | 操作码 | 参数块地址 | 返回值 |
|------|--------|-----------|--------|
| AArch64 | X0 (W0) | X1 | X0 |
| AArch32 | R0 | R1 | R0 |

参数块位于 guest 内存中，每个操作的参数通过指针传递，
参数大小：AArch64 为 8 字节/参数，AArch32 为 4 字节/参数。

---

## 2. 陷入指令检测

### 2.1 AArch64: HLT #0xF000

```c
// target/arm/tcg/translate-a64.c:3213-3227
static bool trans_HLT(DisasContext *s, arg_i *a)
{
    /*
     * HLT. This has two purposes.
     * Architecturally, it is an external halting debug instruction.
     * Since QEMU doesn't implement external debug, we treat this as
     * it is required for halting debug disabled: it will UNDEF.
     * Secondly, "HLT 0xf000" is the A64 semihosting syscall instruction.
     */
    if (semihosting_enabled(s->current_el == 0) && a->imm == 0xf000) {
        gen_exception_internal_insn(s, EXCP_SEMIHOST);
    } else {
        unallocated_encoding(s);
    }
    return true;
}
```

**关键逻辑**：
- `semihosting_enabled(s->current_el == 0)`：检查 semihosting 是否启用
  - 参数 `is_user` 为 true 当 EL0（用户态需单独配置 `userspace` 选项）
- `a->imm == 0xf000`：16 位立即数匹配
- 匹配时生成 `EXCP_SEMIHOST` 内部异常，TB 在此终止
- 不匹配时作为 UNDEF 处理（QEMU 不实现外部 halting debug）

### 2.2 AArch32 ARM 模式: SVC #0x123456

```c
// target/arm/tcg/translate.c:5772-5790
static bool trans_SVC(DisasContext *s, arg_SVC *a)
{
    const uint32_t semihost_imm = s->thumb ? 0xab : 0x123456;

    if (!arm_dc_feature(s, ARM_FEATURE_M) &&
        semihosting_enabled(s->current_el == 0) &&
        a->imm == semihost_imm) {
        gen_exception_internal_insn(s, EXCP_SEMIHOST);
    } else {
        ...  // 正常 SVC 处理
    }
    return true;
}
```

### 2.3 AArch32 Thumb/M-Profile: BKPT #0xAB

```c
// target/arm/tcg/translate.c:3571-3585
static bool trans_BKPT(DisasContext *s, arg_BKPT *a)
{
    /* M-profile semihosting uses BKPT */
    if (arm_dc_feature(s, ARM_FEATURE_M) &&
        semihosting_enabled(s->current_el == 0) &&
        a->imm == 0xab) {
        gen_exception_internal_insn(s, EXCP_SEMIHOST);
    } else {
        ...  // 正常 BKPT
    }
    return true;
}
```

### 2.4 异常处理路径

```
TCG 翻译检测到 semihosting 指令
    → gen_exception_internal_insn(s, EXCP_SEMIHOST)
    → TB 终止，返回到 cpu_exec 循环
    → 异常分发: EXCP_SEMIHOST 路径
    → do_common_semihosting(cs)
    → 返回后继续翻译执行下一条指令
```

---

## 3. 调用分发主流程

### 3.1 入口函数

```c
// semihosting/arm-compat-semi.c:385-826
void do_common_semihosting(CPUState *cs)
{
    CPUArchState *env = cpu_env(cs);
    uint64_t args;
    int nr;

    // 从 X0/R0 读取操作码
    nr = common_semi_arg(cs, 0) & 0xffffffffU;
    // 从 X1/R1 读取参数块地址
    args = common_semi_arg(cs, 1);

    switch (nr) {
    case TARGET_SYS_OPEN:   ... break;
    case TARGET_SYS_CLOSE:  ... break;
    case TARGET_SYS_WRITEC: ... break;
    // ... 25 种操作
    default:
        fprintf(stderr, "qemu: Unsupported SemiHosting SWI 0x%02x\n", nr);
        abort();
    }
}
```

### 3.2 参数访问宏

```c
// GET_ARG(n): 从 guest 参数块读取第 n 个参数
// SET_ARG(n, val): 向 guest 参数块写回第 n 个参数
// 根据 is_64bit_semihosting() 决定每参数 8 或 4 字节
```

### 3.3 回调机制

大部分操作通过异步回调完成（支持 GDB 远程系统调用）：

```c
// common_semi_cb: 通用回调，设置返回值和 errno
static void common_semi_cb(CPUState *cs, uint64_t ret, int err) {
    if (err) { set_swi_errno(cs, err); }
    common_semi_set_ret(cs, ret);
}
```

---

## 4. Semihosting 操作详解

### 4.1 操作码一览

```c
// semihosting/arm-compat-semi.c:54-78
#define TARGET_SYS_OPEN        0x01   // 打开文件
#define TARGET_SYS_CLOSE       0x02   // 关闭文件
#define TARGET_SYS_WRITEC      0x03   // 写一个字符到控制台
#define TARGET_SYS_WRITE0      0x04   // 写零结尾字符串到控制台
#define TARGET_SYS_WRITE       0x05   // 写数据到文件
#define TARGET_SYS_READ        0x06   // 从文件读数据
#define TARGET_SYS_READC       0x07   // 从控制台读一个字符
#define TARGET_SYS_ISERROR     0x08   // 判断返回值是否错误
#define TARGET_SYS_ISTTY       0x09   // 判断是否为终端
#define TARGET_SYS_SEEK        0x0a   // 文件定位
#define TARGET_SYS_FLEN        0x0c   // 获取文件长度
#define TARGET_SYS_TMPNAM      0x0d   // 获取临时文件名
#define TARGET_SYS_REMOVE      0x0e   // 删除文件
#define TARGET_SYS_RENAME      0x0f   // 重命名文件
#define TARGET_SYS_CLOCK       0x10   // 获取处理器时钟
#define TARGET_SYS_TIME        0x11   // 获取当前时间
#define TARGET_SYS_SYSTEM      0x12   // 执行系统命令
#define TARGET_SYS_ERRNO       0x13   // 获取上次错误码
#define TARGET_SYS_GET_CMDLINE 0x15   // 获取命令行
#define TARGET_SYS_HEAPINFO    0x16   // 获取堆栈信息
#define TARGET_SYS_EXIT        0x18   // 退出程序
#define TARGET_SYS_SYNCCACHE   0x19   // 同步缓存 (仅A64)
#define TARGET_SYS_EXIT_EXTENDED 0x20 // 扩展退出 (带状态码)
#define TARGET_SYS_ELAPSED     0x30   // 获取已用时间
#define TARGET_SYS_TICKFREQ    0x31   // 获取时钟频率
```

### 4.2 SYS_OPEN (0x01) — 文件打开

```c
// arm-compat-semi.c:399-451
case TARGET_SYS_OPEN:
    // 参数: arg0=文件名地址, arg1=模式(0-11), arg2=文件名长度
    s = lock_user_string(arg0);

    if (strcmp(s, ":tt") == 0) {
        // 特殊文件 ":tt" → 映射到 stdin/stdout/stderr
        // 模式 0-3 → stdin, 4-7 → stdout, 8-11 → stderr
        hostfd = (arg1 < 4) ? STDIN : (arg1 < 8) ? STDOUT : STDERR;
        ret = alloc_guestfd();
        associate_guestfd(ret, hostfd);
    } else if (strcmp(s, ":semihosting-features") == 0) {
        // 特殊文件: 返回 feature byte 数组 (SH_EXT_EXIT_EXTENDED | SH_EXT_STDOUT_STDERR)
        ret = alloc_guestfd();
        staticfile_guestfd(ret, featurefile_data, sizeof(featurefile_data));
    } else {
        // 普通文件: 调用 host open()
        semihost_sys_open(cs, common_semi_cb, arg0, arg2+1,
                          gdb_open_modeflags[arg1], 0644);
    }
```

**模式映射表** (arm-compat-semi.c:88-100)：

| 模式 | 含义 | Host Flags |
|------|------|-----------|
| 0 | "r" | O_RDONLY |
| 1 | "rb" | O_RDONLY |
| 2 | "r+" | O_RDWR |
| 3 | "r+b" | O_RDWR |
| 4 | "w" | O_WRONLY\|O_CREAT\|O_TRUNC |
| 5 | "wb" | O_WRONLY\|O_CREAT\|O_TRUNC |
| 6 | "w+" | O_RDWR\|O_CREAT\|O_TRUNC |
| 7 | "w+b" | O_RDWR\|O_CREAT\|O_TRUNC |
| 8 | "a" | O_WRONLY\|O_CREAT\|O_APPEND |
| 9 | "ab" | O_WRONLY\|O_CREAT\|O_APPEND |
| 10 | "a+" | O_RDWR\|O_CREAT\|O_APPEND |
| 11 | "a+b" | O_RDWR\|O_CREAT\|O_APPEND |

### 4.3 SYS_WRITE/SYS_READ (0x05/0x06) — 文件 I/O

```c
case TARGET_SYS_WRITE:
    // 参数: arg0=guestfd, arg1=缓冲区地址, arg2=字节数
    semihost_sys_write(cs, common_semi_rw_cb, arg0, arg1, arg2);

case TARGET_SYS_READ:
    // 参数: arg0=guestfd, arg1=缓冲区地址, arg2=字节数
    semihost_sys_read(cs, common_semi_rw_cb, arg0, arg1, arg2);
```

**返回值语义**：返回未传输的字节数（0 = 全部成功）。

### 4.4 SYS_WRITEC/SYS_WRITE0 (0x03/0x04) — 控制台输出

```c
case TARGET_SYS_WRITEC:
    // args 直接指向要写的字符 (1 字节)
    semihost_sys_write_gf(cs, common_semi_dead_cb,
                          &console_out_gf, args, 1);

case TARGET_SYS_WRITE0:
    // args 指向零结尾字符串
    len = target_strlen(args);
    semihost_sys_write_gf(cs, common_semi_dead_cb,
                          &console_out_gf, args, len);
```

### 4.5 SYS_GET_CMDLINE (0x15) — 获取命令行

```c
// arm-compat-semi.c:592-686
case TARGET_SYS_GET_CMDLINE:
    // arg0=缓冲区地址, arg1=缓冲区大小
    cmdline = semihosting_get_cmdline();  // 从 -semihosting-config arg=... 获取
    // 写入 guest 缓冲区
    // 更新 arg1 为实际长度
```

### 4.6 SYS_HEAPINFO (0x16) — 堆栈信息

```c
// arm-compat-semi.c:688-749
case TARGET_SYS_HEAPINFO:
    // 返回 4 个值: heap_base, heap_limit, stack_base, stack_limit
    // system 模式: 从 machine layout 计算
    // user 模式: 使用 brk() 分配 128MB 堆
    retvals[0] = heapbase;
    retvals[1] = heaplimit;
    retvals[2] = stack_base;  // SP 初始值
    retvals[3] = stack_limit;
```

### 4.7 SYS_EXIT/SYS_EXIT_EXTENDED (0x18/0x20) — 退出

```c
// arm-compat-semi.c:751-784
case TARGET_SYS_EXIT:
case TARGET_SYS_EXIT_EXTENDED:
    // A64 或 SYS_EXIT_EXTENDED: 从参数块读取 (reason, subcode)
    // A32 SYS_EXIT: args 直接是 reason
    if (arg0 == ADP_Stopped_ApplicationExit) {
        ret = arg1;  // 正常退出，使用 subcode 作为 exit code
    } else {
        ret = 1;     // 异常退出
    }
    gdb_exit(ret);
    exit(ret);       // 直接终止 QEMU 进程
```

### 4.8 SYS_SYNCCACHE (0x19) — 缓存同步

```c
// arm-compat-semi.c:806-815
case TARGET_SYS_SYNCCACHE:
    // 清除 D-cache 并失效 I-cache (仅 A64 可用)
    // QEMU 不模拟 cache → no-op
    if (common_semi_has_synccache(env)) {
        common_semi_set_ret(cs, 0);
    }
```

### 4.9 SYS_ELAPSED/SYS_TICKFREQ (0x30/0x31) — 时间

```c
case TARGET_SYS_ELAPSED:
    elapsed = get_clock() - clock_start;  // 纳秒精度
    // A64: 一个 64-bit 值写回; A32: 两个 32-bit 值

case TARGET_SYS_TICKFREQ:
    common_semi_set_ret(cs, 1000000000);  // 1GHz (纳秒)
```

---

## 5. Guest 文件描述符管理

### 5.1 GuestFD 数据结构

```c
// semihosting/guestfd.c
typedef enum {
    GuestFDUnused = 0,
    GuestFDHost,       // 映射到 host fd
    GuestFDGDB,        // 通过 GDB 远程 I/O
    GuestFDStatic,     // 内存静态文件 (如 :semihosting-features)
    GuestFDConsole,    // 控制台 (chardev)
} GuestFDType;

typedef struct {
    GuestFDType type;
    union {
        int hostfd;           // Host fd
        struct { uint8_t *data; size_t len; size_t off; }; // Static file
    };
} GuestFD;
```

### 5.2 FD 分配与映射

```c
// guestfd.c:51-67
int alloc_guestfd(void) {
    // 从 slot 1 开始搜索第一个 Unused slot
    // 动态扩容 guestfd_array
    return gf;  // 返回 guest fd 号
}

// guestfd.c:111-118
void associate_guestfd(int guestfd, int hostfd) {
    // 将 guest fd 关联到 host fd
    guestfd_array[guestfd].type = use_gdb_syscalls() ? GuestFDGDB : GuestFDHost;
    guestfd_array[guestfd].hostfd = hostfd;
}
```

### 5.3 初始化（stdio 预映射）

```c
// arm-compat-semi.c:363-375
void semihosting_arm_compatible_init(void) {
    if (use_gdb_syscalls()) {
        console_in_gf.type = GuestFDGDB;
        console_in_gf.hostfd = 0;         // stdin
        console_out_gf.type = GuestFDGDB;
        console_out_gf.hostfd = 2;        // stderr (GDB 约定)
    } else {
        console_in_gf.type = GuestFDConsole;
        console_out_gf.type = GuestFDConsole;
    }
}
```

### 5.4 静态文件 (semihosting-features)

```c
// arm-compat-semi.c:350-356
static const uint8_t featurefile_data[] = {
    SHFB_MAGIC_0, SHFB_MAGIC_1, SHFB_MAGIC_2, SHFB_MAGIC_3,  // "SHFB"
    SH_EXT_EXIT_EXTENDED | SH_EXT_STDOUT_STDERR,               // Feature byte 0
};
```

Guest 可通过打开 `:semihosting-features` 文件探测 host 支持的扩展功能。

---

## 6. 控制台 I/O

### 6.1 控制台架构

```c
// semihosting/console.c:31-38
typedef struct SemihostingConsole {
    Chardev *chr;           // 关联的字符设备
    GSList *sleeping_cpus;  // 等待输入的 CPU 列表
    bool got;               // 是否有输入就绪
    Fifo8 fifo;            // 输入 FIFO 缓冲
    QemuMutex mutex;       // 保护 FIFO 的互斥锁
} SemihostingConsole;
```

### 6.2 输出路径

```c
// console.c:110-118
ssize_t qemu_semihosting_console_write(const uint8_t *buf, int len) {
    if (console.chr) {
        return qemu_chr_write_all(console.chr, buf, len);
    } else {
        return fwrite(buf, 1, len, stderr);  // 无 chardev → 输出到 stderr
    }
}
```

### 6.3 输入路径（阻塞）

```c
// console.c:78-92 (阻塞等待输入)
void qemu_semihosting_console_block_until_ready(CPUState *cs) {
    if (fifo8_is_empty(&console.fifo)) {
        console.sleeping_cpus = g_slist_prepend(console.sleeping_cpus, cs);
        cs->halted = 1;        // halt CPU
        cs->exception_index = EXCP_HALTED;
        cpu_loop_exit(cs);     // 退出 cpu_exec 循环
    }
}

// console.c:94-107 (读取一个字符)
int qemu_semihosting_console_read(CPUState *cs, void *buf, int len) {
    qemu_semihosting_console_block_until_ready(cs);
    // FIFO 有数据后，弹出并返回
    while (len-- && !fifo8_is_empty(&console.fifo)) {
        *(p++) = fifo8_pop(&console.fifo);
    }
}
```

### 6.4 输入回调（chardev 数据到达）

```c
// console.c:59-68
static void console_read(void *opaque, const uint8_t *buf, int size) {
    // 将 chardev 数据推入 FIFO
    while (size-- && !fifo8_is_full(&console.fifo)) {
        fifo8_push(&console.fifo, *buf++);
    }
    // 唤醒所有等待的 CPU
    GSList *list = console.sleeping_cpus;
    console.sleeping_cpus = NULL;
    g_slist_foreach(list, (GFunc)cpu_resume, NULL);
}
```

---

## 7. GDB/Host 双后端

### 7.1 后端选择

```c
// 由 use_gdb_syscalls() 决定:
// - target=gdb: 所有 I/O 通过 GDB File I/O 协议
// - target=native: 直接使用 host 系统调用
// - target=auto (默认): GDB 连接时用 GDB，否则用 host
```

### 7.2 Host 后端 (semihosting/syscalls.c)

直接调用 host 系统调用：
```c
semihost_sys_open()  → open()
semihost_sys_close() → close()
semihost_sys_read()  → read()
semihost_sys_write() → write()
semihost_sys_lseek() → lseek()
semihost_sys_isatty() → isatty()
semihost_sys_remove() → unlink()
semihost_sys_rename() → rename()
semihost_sys_system() → system()
```

### 7.3 GDB 后端

通过 GDB File I/O 扩展协议，将系统调用请求发送到远程调试器：
```c
gdb_do_syscall(callback, "open,%s,%x,%x", path, flags, mode);
gdb_do_syscall(callback, "read,%x,%x,%x", fd, buf, len);
gdb_do_syscall(callback, "write,%x,%x,%x", fd, buf, len);
```

GDB 执行实际 I/O 后，通过 `F` reply 返回结果，QEMU 回调完成 guest 寄存器更新。

---

## 8. 配置与启用机制

### 8.1 命令行选项

```
-semihosting                   # 简单启用
-semihosting-config enable=on,target=native,userspace=on,chardev=chr0,arg=hello
```

### 8.2 配置数据结构

```c
// semihosting/config.c
typedef struct SemihostingConfig {
    bool enabled;              // 是否启用
    bool userspace_enabled;    // 是否允许 EL0 使用
    SemihostingTarget target;  // native/gdb/auto
    char **argv;               // 命令行参数
    int argc;
    const char *cmdline;       // 拼接后的命令行字符串
    Chardev *chardev;          // 控制台字符设备
} SemihostingConfig;
```

### 8.3 启用检查

```c
// config.c:66-69
bool semihosting_enabled(bool is_user) {
    return semihosting.enabled && (!is_user || semihosting.userspace_enabled);
}
```

- `is_user=true`（EL0）时，需要同时启用 `userspace=on`
- `is_user=false`（EL1+）时，只需 `enabled=true`

---

## 9. Target 适配层

### 9.1 ARM 寄存器访问

```c
// target/arm/common-semi-target.c:15-24
uint64_t common_semi_arg(CPUState *cs, int argno) {
    if (is_a64(env)) return env->xregs[argno];   // A64: X0, X1
    else             return env->regs[argno];     // A32: R0, R1
}

// :26-35
void common_semi_set_ret(CPUState *cs, uint64_t ret) {
    if (is_a64(env)) env->xregs[0] = ret;
    else             env->regs[0] = ret;
}
```

### 9.2 栈底获取

```c
// :47-52
uint64_t common_semi_stack_bottom(CPUState *cs) {
    return is_a64(env) ? env->xregs[31] : env->regs[13];  // SP
}
```

### 9.3 A64 特有功能

```c
// :37-39
bool common_semi_sys_exit_is_extended(CPUState *cs) {
    return is_a64(cpu_env(cs));  // A64 的 SYS_EXIT 始终使用扩展格式
}

// :54-58
bool common_semi_has_synccache(CPUArchState *env) {
    return is_a64(env);  // SYS_SYNCCACHE 仅 A64 可用
}
```

---

## 10. 与硬件调试器的对比

### 10.1 行为差异

| 方面 | QEMU 实现 | 硬件调试器 (如 J-Link/DS-5) |
|------|----------|---------------------------|
| **陷入方式** | 翻译时识别，直接调用 handler | CPU 进入 Debug 状态，通知调试器 |
| **执行暂停** | 不暂停，同步完成 (host 后端) | CPU halt → 调试器处理 → 恢复 |
| **性能** | 近零开销 (host 系统调用) | 需要 JTAG 通信延迟 |
| **控制台阻塞** | CPU halt + FIFO + chardev 回调 | 调试器串口窗口 |
| **GDB 集成** | GDB File I/O 协议 | 相同协议 |
| **EL 限制** | 可配置 EL0 启用 | 通常仅特权级 |
| **HLT 指令** | 无 halting debug → UNDEF 或 semihost | 真正的 halting debug 入口 |

### 10.2 QEMU 特有行为

1. **SYS_SYNCCACHE 为 no-op**：QEMU 不模拟 cache，清除/失效无实际效果
2. **SYS_CLOCK 精度**：使用 host `clock()` 而非模拟硬件时钟
3. **SYS_TICKFREQ = 1GHz**：固定为纳秒精度，不对应任何实际硬件频率
4. **SYS_HEAPINFO**：system 模式从 machine layout 计算；user 模式用 brk 分配 128MB

### 10.3 规范兼容性

QEMU 实现完全兼容 ARM Semihosting v2.0 规范：
- ✅ 支持 SH_EXT_EXIT_EXTENDED 扩展
- ✅ 支持 SH_EXT_STDOUT_STDERR 扩展（`:tt` 模式区分 stdin/stdout/stderr）
- ✅ 支持 `:semihosting-features` 特征文件探测
- ✅ AArch64 SYS_EXIT 使用参数块格式（reason + subcode）
- ✅ SYS_SYNCCACHE 仅 A64 可用

---

## 关键源文件索引

| 文件 | 行号 | 内容 |
|------|------|------|
| `target/arm/tcg/translate-a64.c` | 3213-3227 | AArch64 HLT #0xF000 检测 |
| `target/arm/tcg/translate.c` | 5772-5790 | AArch32 SVC #0x123456 检测 |
| `target/arm/tcg/translate.c` | 3571-3585 | M-Profile BKPT #0xAB 检测 |
| `target/arm/common-semi-target.c` | 15-58 | ARM 寄存器访问适配层 |
| `semihosting/arm-compat-semi.c` | 54-78 | 操作码定义 |
| `semihosting/arm-compat-semi.c` | 88-100 | 文件模式映射表 |
| `semihosting/arm-compat-semi.c` | 350-356 | Feature file 数据 |
| `semihosting/arm-compat-semi.c` | 385-826 | do_common_semihosting 主分发 |
| `semihosting/guestfd.c` | 16-141 | Guest FD 管理 |
| `semihosting/console.c` | 31-134 | 控制台 I/O (FIFO/阻塞/唤醒) |
| `semihosting/syscalls.c` | 684-760 | Host/GDB 后端系统调用实现 |
| `semihosting/config.c` | 28-191 | 配置解析与启用逻辑 |
| `include/semihosting/common-semi.h` | 37-43 | 公共 API 声明 |
| `include/semihosting/semihost.h` | 23-50 | 配置 API 声明 |
