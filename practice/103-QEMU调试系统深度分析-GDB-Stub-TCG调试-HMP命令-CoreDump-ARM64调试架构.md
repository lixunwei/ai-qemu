# Doc 103: QEMU 调试系统深度分析

## 文档信息
- **组件**: GDB Stub, 日志系统, Core Dump, ARM64 调试, HMP 调试命令
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [概述](#1-概述)
2. [GDB Stub 远程调试](#2-gdb-stub-远程调试)
3. [TCG 翻译调试](#3-tcg-翻译调试)
4. [HMP 调试命令](#4-hmp-调试命令)
5. [Guest Core Dump](#5-guest-core-dump)
6. [ARM64 调试架构支持](#6-arm64-调试架构支持)
7. [错误报告机制](#7-错误报告机制)
8. [调试实践手册](#8-调试实践手册)

---

## 1. 概述

QEMU 提供多层调试能力：

```
┌─────────────────────────────────────────────────────────────────┐
│                    调试工具层                                     │
│  GDB (远程调试) │ HMP (交互式) │ 日志 (-d) │ Core Dump         │
└───────┬─────────────────┬──────────────┬──────────────┬────────┘
        │                 │              │              │
        ▼                 ▼              ▼              ▼
┌───────────────┐ ┌──────────────┐ ┌──────────┐ ┌──────────────┐
│  GDB Stub     │ │  Monitor     │ │ Logging  │ │  Dump Engine │
│  (RSP 协议)   │ │  (QMP/HMP)   │ │ (util/)  │ │  (dump/)     │
└───────┬───────┘ └──────────────┘ └──────────┘ └──────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│              CPU/内存/设备 模拟层                                 │
│  断点 │ 观察点 │ 单步 │ 寄存器访问 │ 内存读写                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. GDB Stub 远程调试

### 2.1 架构

```
┌─────────────┐         TCP/Unix Socket         ┌──────────────┐
│   GDB       │ ◄────── RSP Protocol ──────────► │  QEMU        │
│   Client    │                                  │  GDB Stub    │
└─────────────┘                                  └──────┬───────┘
                                                        │
                                              ┌─────────┴─────────┐
                                              │  gdbstub/         │
                                              │  ├── gdbstub.c    │ ← 协议引擎
                                              │  ├── system.c     │ ← 系统模式
                                              │  └── user.c       │ ← 用户模式
                                              └───────────────────┘
```

### 2.2 核心实现

```c
// gdbstub/gdbstub.c 核心:

// RSP 报文处理:
// - gdb_put_packet() / gdb_put_packet_binary()  [L137-177]
//   构造 $<data>#<checksum> 回复
// - gdb_read_byte() / 报文解析器 [L900+]
//   解析客户端请求

// 命令分发:
// - process_string_cmd() [L901-1003]
//   将 RSP 命令映射到处理函数
// - 支持的命令: Hg/Hc, ?, c, s, vCont, Z0-Z4, g, G, m, M, ...
//   [L1021-1250]

// 多线程:
// - gdb_get_cpu_index() = cpu_index + 1  [system.c:57-60]
// - multiprocess 格式: pPID.TID  [gdbstub.c:677-685]
// - vCont: 按线程/进程分别 continue/step  [L739-875]
```

### 2.3 断点/观察点

```c
// gdbstub/system.c:640-664
int gdb_breakpoint_insert(int type, vaddr addr, vaddr len)
{
    CPUState *cpu;
    switch (type) {
    case GDB_BREAKPOINT_SW:      // 软件断点 (Z0)
    case GDB_BREAKPOINT_HW:      // 硬件断点 (Z1)
        CPU_FOREACH(cpu) {
            cpu_breakpoint_insert(cpu, addr, BP_GDB, NULL);
        }
        break;
    case GDB_WATCHPOINT_WRITE:   // 写观察点 (Z2)
    case GDB_WATCHPOINT_READ:    // 读观察点 (Z3)
    case GDB_WATCHPOINT_ACCESS:  // 访问观察点 (Z4)
        CPU_FOREACH(cpu) {
            cpu_watchpoint_insert(cpu, addr, len, type, NULL);
        }
        break;
    }
}
```

### 2.4 单步执行

```c
// gdbstub/system.c:558-603
void gdb_continue_partial(GDBState *s, ...)
{
    CPU_FOREACH(cpu) {
        switch (action) {
        case 's':  // step
            cpu_single_step(cpu, SSTEP_ENABLE | SSTEP_NOIRQ | SSTEP_NOTIMER);
            cpu_resume(cpu);
            break;
        case 'c':  // continue
            cpu_single_step(cpu, 0);  // 清除单步
            cpu_resume(cpu);
            break;
        }
    }
}

// 停止时发送 T 包:
// gdbstub/system.c:122-217
void gdb_vm_state_change(...)
{
    // 构造停止回复: T05thread:01;watch:addr;...
}
```

### 2.5 多 CPU 调试

```c
// 系统模式:
// - 连接时默认 attach 第一个 CPU
// - Hg<tid> 切换当前 CPU
// - vCont;s:1;c 对 CPU 1 单步, 其余 continue

// 用户模式:
// - 支持 fork follow
// - clone() 创建的线程自动注册为 GDB thread
// - gdbstub/user.c:31-107 管理线程映射
```

### 2.6 使用方法

```bash
# 启动 QEMU 等待 GDB:
qemu-system-aarch64 -s -S ...
# -s = -gdb tcp::1234
# -S = 启动时暂停

# 或指定端口:
qemu-system-aarch64 -gdb tcp::9000 -S ...

# GDB 端:
$ aarch64-linux-gnu-gdb vmlinux
(gdb) target remote :1234
(gdb) break start_kernel
(gdb) continue
(gdb) info threads          # 查看所有 vCPU
(gdb) thread 2              # 切换到 CPU 1
(gdb) info registers        # 查看寄存器
(gdb) x/10i $pc             # 反汇编
(gdb) watch *0xffff0000     # 内存观察点
```

---

## 3. TCG 翻译调试

### 3.1 日志选项全解

| 选项 | 输出内容 | 用途 |
|------|----------|------|
| `in_asm` | Guest 汇编 (每个 TB) | 查看翻译了什么 |
| `out_asm` | Host 汇编 (每个 TB) | 查看生成了什么 |
| `op` | TCG IR 操作码 | 查看中间表示 |
| `op_opt` | 优化后 TCG IR | 查看优化效果 |
| `op_ind` | indirect lowering 前 | 查看间接调用 |
| `exec` | TB 执行序列 (PC) | 追踪执行路径 |
| `cpu` | 执行前 CPU 完整状态 | 寄存器/标志位 |
| `fpu` | 浮点状态 | FP 调试 |
| `int` | 中断/异常事件 | 异常追踪 |
| `mmu` | MMU/TLB 操作 | 地址翻译调试 |
| `page` | 页表更新 | 页映射变化 |
| `nochain` | 禁止 TB 链 | 确保每条 trace 完整 |
| `unimp` | 未实现警告 | 发现缺失模拟 |
| `guest_errors` | Guest 违规 | 检测 Guest bug |
| `strace` | 系统调用追踪 | linux-user 调试 |

### 3.2 典型调试场景

```bash
# 场景 1: 为什么 Guest 崩溃?
qemu-system-aarch64 -d int,cpu -D /tmp/crash.log ...
# 查看最后的异常和 CPU 状态

# 场景 2: 翻译是否正确?
qemu-system-aarch64 -d in_asm,op_opt,out_asm -D /tmp/tcg.log \
    -dfilter 0xffff800080000000..0xffff800080001000 ...
# -dfilter 限制地址范围, 避免日志爆炸

# 场景 3: 执行路径追踪
qemu-system-aarch64 -d exec,nochain -D /tmp/exec.log ...
# nochain 阻止 TB 链, 确保每次执行都被记录

# 场景 4: 内存翻译问题
qemu-system-aarch64 -d mmu,page -D /tmp/mmu.log ...
# 查看 TLB miss 和页表遍历

# 场景 5: linux-user 系统调用
qemu-aarch64 -d strace ./myprogram
# 类似 strace 的系统调用追踪
```

### 3.3 地址过滤

```bash
# -dfilter 限制日志到特定地址范围:
-dfilter 0x1000..0x2000           # 单个范围
-dfilter 0x1000..0x2000,0x5000..0x6000  # 多个范围

# 只记录特定函数的翻译:
qemu-system-aarch64 -d in_asm,out_asm \
    -dfilter 0xffff800080012340..0xffff8000800123f0 ...
```

---

## 4. HMP 调试命令

### 4.1 内存检查

```
# 物理内存转储 (xp = examine physical):
(qemu) xp /10gx 0x40000000       # 10个8字节, 十六进制, 物理地址
(qemu) xp /20wx 0x40000000       # 20个4字节
(qemu) xp /4i 0xffff800080000000 # 4条指令反汇编

# 物理→虚拟地址转换:
(qemu) gpa2hva 0x40000000        # GPA → 进程内 HVA

# 内存保存到文件:
(qemu) memsave 0x1000 0x1000 /tmp/vmem.bin   # 虚拟地址
(qemu) pmemsave 0x40000000 0x1000 /tmp/pmem.bin  # 物理地址
```

### 4.2 CPU 状态

```
# 查看所有寄存器:
(qemu) info registers            # 当前 CPU 全部寄存器
(qemu) info registers -a         # 所有 CPU 的寄存器

# 切换 CPU 上下文:
(qemu) cpu 0                     # 切换到 CPU 0
(qemu) cpu 3                     # 切换到 CPU 3
(qemu) info cpus                 # 列出所有 CPU 及状态
```

### 4.3 设备与内存布局

```
# 内存区域树 (核心调试命令):
(qemu) info mtree                # 完整 MemoryRegion 树
(qemu) info mtree -f             # 含 FlatView 展平视图

# 设备拓扑:
(qemu) info qtree               # QOM 设备树 (含属性)
(qemu) info qom-tree            # QOM 对象层次
(qemu) info pci                 # PCI 设备/BAR/中断

# TLB:
(qemu) info tlb                 # 当前 TLB 内容
```

### 4.4 运行控制

```
# 暂停/恢复:
(qemu) stop                     # 暂停所有 vCPU
(qemu) cont                     # 恢复执行

# 单步 (需 GDB 或特殊支持):
(qemu) singlestep on           # 启用单步模式
(qemu) singlestep off

# 系统操作:
(qemu) system_reset            # 复位 VM
(qemu) system_powerdown        # 发送关机请求
(qemu) quit                    # 退出 QEMU

# GDB 服务:
(qemu) gdbserver tcp::1234     # 运行时启动 GDB 服务
(qemu) gdbserver none          # 关闭 GDB 服务
```

### 4.5 热路径分析

```
# TCG 翻译统计:
(qemu) info jit
# 输出: 翻译块数量, 代码缓存使用, TB 平均大小等

# 锁竞争:
(qemu) info sync-profile
# 输出: 各锁等待时间, 竞争次数
```

---

## 5. Guest Core Dump

### 5.1 架构

```c
// dump/dump.c 核心流程:
//
// qmp_dump_guest_memory() → DumpState 初始化
//     → dump_begin()      → 写 ELF 头 + PT_NOTE (CPU寄存器)
//     → dump_iterate()    → 遍历 RAM, 写 PT_LOAD 段
//     → dump_end()        → 写 section 数据
//
// 支持格式: ELF core, kdump (makedumpfile 兼容), Windows dump
```

### 5.2 使用方法

```bash
# HMP:
(qemu) dump-guest-memory /tmp/guest.core

# QMP:
{"execute": "dump-guest-memory",
 "arguments": {"paging": false, "protocol": "file:/tmp/guest.core"}}

# 选项:
(qemu) dump-guest-memory -z /tmp/guest.core.gz    # zlib 压缩
(qemu) dump-guest-memory -l /tmp/guest.core.lzo   # LZO 压缩
(qemu) dump-guest-memory -s /tmp/guest.core.snappy # Snappy 压缩

# 然后用 crash/gdb 分析:
$ crash vmlinux /tmp/guest.core
$ aarch64-linux-gnu-gdb vmlinux -c /tmp/guest.core
```

### 5.3 Dump 格式

```
ELF Core 文件结构:
┌────────────────────┐
│ ELF Header         │ (ET_CORE)
├────────────────────┤
│ Program Headers    │
│ ├── PT_NOTE        │ ← CPU 寄存器状态 (每个 vCPU)
│ ├── PT_LOAD        │ ← RAM 段 1
│ ├── PT_LOAD        │ ← RAM 段 2
│ └── ...            │
├────────────────────┤
│ Note Data          │ (NT_PRSTATUS per CPU)
├────────────────────┤
│ RAM Pages          │
└────────────────────┘
```

---

## 6. ARM64 调试架构支持

### 6.1 实现位置

```c
// target/arm/debug_helper.c:17-524

// QEMU 实现了 ARMv8 调试架构:
// - MDSCR_EL1 (Monitor Debug System Control)
// - DBGBVRn_EL1 (Breakpoint Value Registers)
// - DBGBCRn_EL1 (Breakpoint Control Registers)
// - DBGWVRn_EL1 (Watchpoint Value Registers)
// - DBGWCRn_EL1 (Watchpoint Control Registers)
// - OSLAR_EL1, OSLSR_EL1 (OS Lock)
// - DBGCLAIMSET, DBGCLAIMCLR
```

### 6.2 断点寄存器

```c
// target/arm/debug_helper.c:371-400
static void dbgbvr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    int i = ri->crm;  // 断点编号
    env->cp15.dbgbvr[i] = value;
    hw_breakpoint_update(cpu, i);  // 同步到 TCG 断点
}

static void dbgbcr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    int i = ri->crm;
    env->cp15.dbgbcr[i] = value;
    hw_breakpoint_update(cpu, i);  // 更新控制: 启用/类型/特权级
}
```

### 6.3 观察点寄存器

```c
// target/arm/debug_helper.c:334-369
static void dbgwvr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    env->cp15.dbgwvr[i] = value;
    hw_watchpoint_update(cpu, i);  // 同步到 TCG 观察点
}

static void dbgwcr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    env->cp15.dbgwcr[i] = value;
    hw_watchpoint_update(cpu, i);  // 更新: 大小/类型/特权级
}
```

### 6.4 调试访问陷阱

```c
// target/arm/debug_helper.c:21-121
// 控制从 EL0/EL1 到 EL2/EL3 的调试寄存器访问陷阱:

static CPAccessResult access_tdosa(CPUARMState *env, ...)
{
    // MDCR_EL2.TDOSA: OS Lock 相关寄存器陷阱到 EL2
}

static CPAccessResult access_tdra(CPUARMState *env, ...)
{
    // MDCR_EL2.TDRA: 调试 ROM 相关寄存器陷阱到 EL2
}

static CPAccessResult access_tda(CPUARMState *env, ...)
{
    // MDCR_EL2.TDA: 调试寄存器陷阱到 EL2
    // MDCR_EL3.TDA: 陷阱到 EL3
}

static CPAccessResult access_tdcc(CPUARMState *env, ...)
{
    // MDCR_EL2.TDCC / MDCR_EL3.TDCC: 调试通信通道陷阱
}
```

### 6.5 Self-hosted vs External Debug

| 模式 | 说明 | QEMU 支持 |
|------|------|-----------|
| Self-hosted | Guest OS 使用自己的调试寄存器 | ✅ 完整支持 |
| External | 外部调试器通过 JTAG/DAP | ⚠️ 部分桩实现 |
| Halting | 外部停止 CPU 检查状态 | ❌ 不支持 |

- Self-hosted: Guest 写 DBGBVRn → QEMU 同步到 TCG 断点 → 触发时产生 Debug 异常
- External 相关寄存器（DCC/vector catch 等）以 RAZ/WI 方式桩实现

---

## 7. 错误报告机制

### 7.1 错误输出 API

```c
// util/error-report.c:17-459

// 严重错误 (会影响运行):
void error_report(const char *fmt, ...);
// → 输出到 HMP (如果可用) 或 stderr

// 警告:
void warn_report(const char *fmt, ...);

// 信息:
void info_report(const char *fmt, ...);

// 带位置信息:
void error_report_at(const char *file, int line, const char *fmt, ...);

// GLib 日志桥:
void error_init(const char *argv0);
// 拦截 GLib g_warning/g_error → 统一格式输出
```

### 7.2 致命错误处理

```c
// 不可恢复错误:
void error_report_abort(void);   // 打印 + abort()
// 用于内部一致性检查失败

// QEMU 专用 abort:
#define g_assert_not_reached()
// 用于不应到达的代码路径
```

---

## 8. 调试实践手册

### 8.1 Guest 无法启动

```bash
# 步骤 1: 启用基本日志
qemu-system-aarch64 -d unimp,guest_errors,int -D /tmp/boot.log \
    -M virt -cpu cortex-a57 -kernel Image ...

# 步骤 2: 检查异常
grep -i "exception\|fault\|error" /tmp/boot.log

# 步骤 3: 如果卡住, 用 GDB 附加
qemu-system-aarch64 -s -S ...
(gdb) target remote :1234
(gdb) break *0xffff800080000000
(gdb) continue
(gdb) bt
```

### 8.2 Guest 运行中崩溃

```bash
# 方法 1: 自动 dump
qemu-system-aarch64 -action panic=pause ...
# Guest panic 后 QEMU 暂停, 然后:
(qemu) dump-guest-memory /tmp/crash.core
(qemu) info registers

# 方法 2: pvpanic 设备
qemu-system-aarch64 -device pvpanic-pci ...
# Guest panic 通过 QMP 事件通知
```

### 8.3 TCG 翻译问题

```bash
# 怀疑翻译错误 (结果不符合预期):
qemu-system-aarch64 \
    -d in_asm,op_opt,out_asm \
    -dfilter 0xffff800080001000..0xffff800080001100 \
    -D /tmp/translate.log ...

# 查看日志:
# IN: (guest assembly)
#   0xffff800080001000: mrs x0, CurrentEL
# OP after optimization:
#   ... TCG ops ...
# OUT: (host assembly)
#   0x7f1234000000: mov w0, #0x4
```

### 8.4 内存/MMIO 问题

```bash
# 检查内存布局:
(qemu) info mtree -f
# 确认设备 MMIO 地址正确

# 追踪 MMIO 访问:
qemu-system-aarch64 --trace "memory_region_ops_*" -D /tmp/mmio.log ...

# 物理内存检查:
(qemu) xp /4gx 0x09000000    # 检查 GIC 地址
(qemu) xp /4wx 0x09010000    # 检查 UART 地址
```

### 8.5 中断问题

```bash
# 查看中断统计:
(qemu) info irq

# 追踪中断注入:
qemu-system-aarch64 -d int -D /tmp/int.log ...
qemu-system-aarch64 --trace "gicv3_*" -D /tmp/gic.log ...

# 检查 GIC 状态:
(qemu) info qtree | grep -A20 gic
```

### 8.6 性能问题

```bash
# TCG 热点分析:
qemu-system-aarch64 -accel tcg,perfmap=true ...
perf record -p $(pidof qemu-system-aarch64) -g sleep 30
perf report --sort comm,dso,symbol

# JIT 统计:
(qemu) info jit
# Translation buffer state:
#   gen code size: 12345678/268435456
#   TB count: 12345
#   average TB size: 1000

# 锁分析:
(qemu) info sync-profile
```

### 8.7 KVM 调试

```bash
# KVM trace:
qemu-system-aarch64 --trace "kvm_*" -D /tmp/kvm.log ...

# KVM 内核事件 (需 root):
echo 1 > /sys/kernel/debug/tracing/events/kvm/enable
cat /sys/kernel/debug/tracing/trace_pipe

# 查看 KVM 状态:
(qemu) info kvm
```

### 8.8 常用调试组合

```bash
# 全面调试 (慢, 适合小测试):
-d in_asm,out_asm,op_opt,exec,cpu,int,mmu -D /tmp/full.log

# 异常诊断:
-d int,cpu -D /tmp/exception.log

# 内存问题:
-d mmu,page,guest_errors -D /tmp/memory.log

# 启动调试 (加 GDB):
-s -S -d unimp,guest_errors

# 性能分析:
-accel tcg,perfmap=true
```

---

## 附录 A: GDB RSP 命令速查

| RSP 命令 | GDB 操作 | QEMU 处理 |
|----------|----------|-----------|
| `g` | info registers | 读取所有寄存器 |
| `G<hex>` | set $reg | 写入所有寄存器 |
| `m<addr>,<len>` | x/... | 读内存 |
| `M<addr>,<len>:<hex>` | set *addr | 写内存 |
| `c` | continue | 恢复执行 |
| `s` | step/stepi | 单步 |
| `Z0,<addr>,<len>` | break *addr | 设软断点 |
| `Z1,<addr>,<len>` | hbreak *addr | 设硬断点 |
| `Z2,<addr>,<len>` | watch *addr | 写观察点 |
| `Z3,<addr>,<len>` | rwatch *addr | 读观察点 |
| `Z4,<addr>,<len>` | awatch *addr | 访问观察点 |
| `z<type>,<addr>` | delete | 删除断点/观察点 |
| `Hg<tid>` | thread <n> | 切换线程 |
| `vCont;c:1;s:2` | - | 线程级 continue/step |
| `?` | - | 查询停止原因 |
| `qSupported` | - | 能力协商 |

## 附录 B: 源文件索引

| 文件 | 职责 |
|------|------|
| `gdbstub/gdbstub.c` | GDB RSP 协议引擎 |
| `gdbstub/system.c` | 系统模式 GDB (断点/观察点/多 CPU) |
| `gdbstub/user.c` | 用户模式 GDB (线程/fork) |
| `include/exec/gdbstub.h` | GDB stub API |
| `util/log.c` | 日志系统 (30+ 分类) |
| `include/qemu/log.h` | 日志 API/类别定义 |
| `dump/dump.c` | Guest core dump (ELF/kdump) |
| `target/arm/debug_helper.c` | ARM64 调试寄存器模拟 |
| `util/error-report.c` | 错误/警告报告 |
| `monitor/hmp-cmds.c` | HMP 调试命令实现 |
| `hmp-commands.hx` | HMP 命令定义 |
| `hmp-commands-info.hx` | HMP info 命令定义 (71个) |

---

*文档结束*
