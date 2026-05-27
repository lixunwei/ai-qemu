# Doc 106: QEMU GDB 远程调试完全指南 — Guest 调试命令与用法

## 文档信息
- **组件**: GDB Stub, RSP 协议, ARM64 寄存器, 调试命令
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [启动与连接](#1-启动与连接)
2. [支持的 RSP 命令完整列表](#2-支持的-rsp-命令完整列表)
3. [ARM64 寄存器访问](#3-arm64-寄存器访问)
4. [断点与观察点](#4-断点与观察点)
5. [内存访问](#5-内存访问)
6. [线程与多核调试](#6-线程与多核调试)
7. [monitor 命令透传](#7-monitor-命令透传)
8. [反向调试](#8-反向调试)
9. [高级特性](#9-高级特性)
10. [GDB 实用命令速查](#10-gdb-实用命令速查)
11. [调试场景实践](#11-调试场景实践)
12. [源码实现索引](#12-源码实现索引)

---

## 1. 启动与连接

### 1.1 启动 QEMU GDB 服务

```bash
# 方式 1: 启动时开启 (-s = tcp::1234, -S = 暂停等待连接)
qemu-system-aarch64 -M virt -cpu max -m 2G \
    -kernel Image -append "console=ttyAMA0 nokaslr" \
    -nographic -s -S

# 方式 2: 自定义端口
qemu-system-aarch64 ... -gdb tcp::9000 -S

# 方式 3: Unix socket (更安全)
qemu-system-aarch64 ... -gdb unix:/tmp/qemu-gdb.sock -S

# 方式 4: 运行时启动 (HMP)
(qemu) gdbserver tcp::1234

# 方式 5: 运行时启动 (QMP)
{"execute": "human-monitor-command",
 "arguments": {"command-line": "gdbserver tcp::1234"}}
```

### 1.2 GDB 连接

```bash
# TCP 连接:
$ aarch64-linux-gnu-gdb vmlinux
(gdb) target remote :1234

# Unix socket:
(gdb) target remote /tmp/qemu-gdb.sock

# 扩展模式 (支持多进程):
(gdb) target extended-remote :1234
```

### 1.3 用户模式调试

```bash
# 启动带 GDB 的用户模式:
qemu-aarch64 -g 1234 ./my_program

# GDB 连接:
$ gdb-multiarch ./my_program
(gdb) target remote :1234
```

---

## 2. 支持的 RSP 命令完整列表

### 2.1 基础命令

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `!` | (连接时) | 启用扩展模式 | `gdbstub.c:2068` |
| `?` | (连接时) | 查询停止原因 | `gdbstub.c:2072` |
| `c` | `continue` | 恢复执行 | `gdbstub.c:2083` |
| `C sig` | `signal SIG` | 带信号恢复 | `gdbstub.c:2095` |
| `s` | `stepi` | 单步执行 | `gdbstub.c:2135` |
| `k` | `kill` | 终止目标 | `gdbstub.c:2118` |
| `D` | `detach` | 断开连接 | `gdbstub.c:2124` |

### 2.2 寄存器访问

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `g` | `info registers` | 读取所有寄存器 | `gdbstub.c:2170` |
| `G <hex>` | `set $reg=val` | 写入所有寄存器 | `gdbstub.c:2180` |
| `p <n>` | `print $reg` | 读取单个寄存器 | `gdbstub.c:2213` |
| `P <n>=<hex>` | `set $reg=val` | 写入单个寄存器 | `gdbstub.c:2223` |

### 2.3 内存访问

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `m addr,len` | `x/...` | 读内存 | `gdbstub.c:2191` |
| `M addr,len:data` | `set *addr=val` | 写内存 | `gdbstub.c:2201` |

### 2.4 断点/观察点

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `Z0,addr,len` | `break *addr` | 软件断点 | `gdbstub.c:2235` |
| `Z1,addr,len` | `hbreak *addr` | 硬件断点 | 同上 |
| `Z2,addr,len` | `watch *addr` | 写观察点 | 同上 |
| `Z3,addr,len` | `rwatch *addr` | 读观察点 | 同上 |
| `Z4,addr,len` | `awatch *addr` | 访问观察点 | 同上 |
| `z0,addr,len` | `delete` | 删除软件断点 | `gdbstub.c:2245` |
| `z1-z4` | `delete` | 删除硬/观察点 | 同上 |

### 2.5 线程控制

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `Hg tid` | `thread <n>` | 设置当前 g 线程 | `gdbstub.c:2257` |
| `Hc tid` | - | 设置当前 c 线程 | 同上 |
| `T tid` | - | 检查线程是否存活 | `gdbstub.c:2268` |
| `qC` | - | 查询当前线程 ID | `gdbstub.c:1855` |
| `qfThreadInfo` | `info threads` | 获取线程列表(首) | `gdbstub.c:1869` |
| `qsThreadInfo` | - | 获取线程列表(续) | `gdbstub.c:1861` |
| `qThreadExtraInfo` | - | 线程描述信息 | `gdbstub.c:1873` |

### 2.6 vCont 高级执行控制

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `vCont?` | - | 查询支持的动作 | `gdbstub.c:1404` |
| `vCont;c` | `continue` | 全部继续 | `gdbstub.c:739` |
| `vCont;s:tid` | `stepi` (当前线程) | 指定线程单步 | 同上 |
| `vCont;c:tid;s:tid2` | - | 混合动作 | 同上 |
| `vAttach;pid` | `attach` | 附加到进程 | `gdbstub.c:1425` |
| `vKill;pid` | `kill` | 终止进程 | `gdbstub.c:1452` |

### 2.7 查询命令

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `qSupported` | (自动) | 能力协商 | `gdbstub.c:1678` |
| `qAttached` | - | 是否已附加 | `gdbstub.c:1931` |
| `qXfer:features:read` | - | XML 目标描述 | `gdbstub.c:1725` |
| `qXfer:auxv:read` | - | 辅助向量(user) | `gdbstub.c:1908` |
| `qXfer:exec-file:read` | - | 可执行文件路径 | `gdbstub.c:1929` |
| `qXfer:siginfo:read` | - | 信号信息(user) | `gdbstub.c:1918` |
| `qRcmd,<hex>` | `monitor <cmd>` | HMP 命令透传 | `system.c:512` |
| `qGDBServerVersion` | - | 服务器版本 | `gdbstub.c:1865` |
| `qemu.PhyMemMode` | - | 物理内存模式查询 | `gdbstub.c:1944` |

### 2.8 设置命令

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `Qqemu.sstep` | - | 设置单步掩码 | `gdbstub.c:1783` |
| `Qqemu.PhyMemMode` | - | 设置物理内存模式 | `gdbstub.c:1959` |
| `QCatchSyscalls` | `catch syscall` | 系统调用捕获(user) | `gdbstub.c:1975` |

### 2.9 反向调试

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `bc` | `reverse-continue` | 反向继续 | `gdbstub.c:1376` |
| `bs` | `reverse-stepi` | 反向单步 | `gdbstub.c:1389` |

### 2.10 文件 I/O (用户模式)

| RSP 包 | GDB 命令 | 功能 | 源码位置 |
|--------|----------|------|----------|
| `vFile:open` | - | 打开文件 | `gdbstub.c:1463` |
| `vFile:close` | - | 关闭文件 | 同上 |
| `vFile:pread` | - | 读文件 | 同上 |
| `vFile:readlink` | - | 读链接 | 同上 |
| `F result` | - | File I/O 回复 | `gdbstub.c:2159` |

---

## 3. ARM64 寄存器访问

### 3.1 核心寄存器 (aarch64-core.xml)

```
(gdb) info registers

寄存器列表 (34 个核心寄存器):
x0-x30    : 通用寄存器 (64-bit)
sp        : 栈指针 (x31 alias)
pc        : 程序计数器
cpsr/pstate: 处理器状态 (NZCV + EL + SPSel + DAIF)
```

| 编号 | 名称 | 位宽 | GDB 访问 |
|------|------|------|----------|
| 0-30 | x0-x30 | 64 | `$x0` - `$x30` |
| 31 | sp | 64 | `$sp` |
| 32 | pc | 64 | `$pc` |
| 33 | cpsr | 32 | `$cpsr` |

### 3.2 FP/SIMD 寄存器 (aarch64-fpu.xml)

```
(gdb) info float
(gdb) print $v0.d.u[0]

寄存器列表 (34 个):
v0-v31    : 128-bit SIMD/FP 向量
fpsr      : 浮点状态寄存器
fpcr      : 浮点控制寄存器
```

| 编号 | 名称 | 位宽 | GDB 访问 |
|------|------|------|----------|
| 34-65 | v0-v31 | 128 | `$v0` - `$v31` |
| 66 | fpsr | 32 | `$fpsr` |
| 67 | fpcr | 32 | `$fpcr` |

### 3.3 SVE 寄存器 (动态生成 sve-registers.xml)

```
(gdb) info registers sve
(gdb) print $z0.d.u[0]

寄存器 (根据 VL 动态):
z0-z31    : 可扩展向量 (VL bits)
p0-p15    : 谓词寄存器 (VL/8 bits)
ffr       : 首次故障寄存器
vg        : 向量粒度 (VL/64)
fpsr, fpcr
```

### 3.4 SME 寄存器 (动态生成 sme-registers.xml)

```
寄存器:
svg       : 流向量粒度
svcr      : 流向量控制
za        : ZA 矩阵存储 (SVL × SVL bytes)
```

### 3.5 SME2 寄存器

```
zt0       : ZT0 查找表寄存器 (512 bits)
```

### 3.6 TLS 寄存器 (tls-registers.xml)

```
tpidr_el0   : 用户态线程 ID
tpidr2_el0  : EL0 线程 ID 2 (SME 用)
```

### 3.7 Pointer Authentication 伪寄存器 (aarch64-pauth.xml)

```
pauth_dmask      : 数据指针掩码
pauth_cmask      : 代码指针掩码
pauth_dmask_high : 高位数据掩码 (FEAT_PAuth2)
pauth_cmask_high : 高位代码掩码 (FEAT_PAuth2)
```

用途: GDB 使用这些掩码自动去除 PAC 签名以正确回溯调用栈。

### 3.8 MTE 标签访问 (aarch64-mte.xml)

ARM64 MTE 通过自定义 GDB 命令扩展支持内存标签的读写:
- 注册在 `target/arm/gdbstub64.c:843-880`
- 允许 GDB 查询/设置内存区域的 Allocation Tag

### 3.9 系统寄存器 (动态生成 system-registers.xml)

```
(gdb) info registers system

QEMU 导出所有 ARMCPRegInfo 中标记了 GDB 可访问的系统寄存器:
- SCTLR_EL1, TCR_EL1, TTBR0_EL1, TTBR1_EL1
- ESR_EL1, FAR_EL1, VBAR_EL1
- MAIR_EL1, CONTEXTIDR_EL1
- (根据 CPU 特性和 EL 动态生成)
```

源码: `target/arm/gdbstub.c:324, 558-563`

---

## 4. 断点与观察点

### 4.1 软件断点 (Z0)

```gdb
(gdb) break *0xffff800080000000    # 绝对地址
(gdb) break start_kernel           # 符号名
(gdb) break kernel/sched/core.c:3000  # 源码行

# 条件断点:
(gdb) break *0xffff800080001000 if $x0 == 0x1234

# 临时断点:
(gdb) tbreak *0xffff800080000000
```

实现: QEMU 在目标地址插入特殊指令 (ARM: BRK), 命中时产生调试异常。

### 4.2 硬件断点 (Z1)

```gdb
(gdb) hbreak *0xffff800080000000

# ARM64 硬件断点通过 DBGBVR/DBGBCR 实现
# 数量有限 (通常 6-16 个)
```

### 4.3 观察点 (Z2/Z3/Z4)

```gdb
# 写观察点 (变量被写入时触发):
(gdb) watch *0xffff800080100000
(gdb) watch my_global_var

# 读观察点:
(gdb) rwatch *0xffff800080100000

# 读写观察点:
(gdb) awatch *0xffff800080100000

# 带条件:
(gdb) watch *0xffff800080100000 if *0xffff800080100000 > 100
```

实现: QEMU 通过 DBGWVR/DBGWCR (TCG) 或 KVM 硬件支持。

### 4.4 断点管理

```gdb
(gdb) info breakpoints           # 列出所有断点
(gdb) delete 3                   # 删除断点 3
(gdb) disable 2                  # 禁用断点 2
(gdb) enable 2                   # 启用断点 2
(gdb) clear *0xffff800080000000  # 清除地址上的断点
```

---

## 5. 内存访问

### 5.1 读取内存

```gdb
# 格式: x/[数量][格式][大小] 地址
# 格式: x=十六进制, d=十进制, u=无符号, t=二进制, i=指令, s=字符串, a=地址
# 大小: b=1字节, h=2字节, w=4字节, g=8字节

(gdb) x/10gx 0xffff800080000000    # 10个8字节, 十六进制
(gdb) x/20i $pc                    # 从 PC 开始反汇编 20 条
(gdb) x/s 0xffff800080200000       # 字符串
(gdb) x/4wx $sp                    # 栈顶 4 个 word
```

### 5.2 写入内存

```gdb
(gdb) set *((uint64_t*)0xffff800080100000) = 0x12345678
(gdb) set *((uint32_t*)0xffff800080100000) = 0xABCD
(gdb) set {int}0xffff800080100000 = 42
```

### 5.3 物理内存模式

QEMU 扩展: 可切换 GDB 内存访问为物理地址模式:

```gdb
# 切换到物理内存模式:
(gdb) monitor qemu.PhyMemMode 1

# 现在 x 命令使用物理地址:
(gdb) x/10gx 0x40000000           # 直接访问物理地址

# 恢复虚拟地址模式:
(gdb) monitor qemu.PhyMemMode 0
```

源码: `gdbstub.c:1944-1974` (`Qqemu.PhyMemMode`)

---

## 6. 线程与多核调试

### 6.1 查看所有 vCPU

```gdb
(gdb) info threads
  Id   Target Id         Frame
* 1    Thread 1.1 (CPU#0 [running]) 0xffff800080000000 in start_kernel ()
  2    Thread 1.2 (CPU#1 [halted])  0xffff800080012000 in secondary_entry ()
  3    Thread 1.3 (CPU#2 [halted])  0xffff800080012000 in secondary_entry ()
  4    Thread 1.4 (CPU#3 [halted])  0xffff800080012000 in secondary_entry ()
```

### 6.2 切换 CPU 上下文

```gdb
(gdb) thread 2                     # 切换到 CPU#1
(gdb) info registers               # 查看 CPU#1 的寄存器
(gdb) bt                           # CPU#1 的调用栈
(gdb) thread 1                     # 切回 CPU#0
```

### 6.3 对指定 CPU 操作

```gdb
# 只在 CPU#0 设断点:
(gdb) break *0xffff800080000000 thread 1

# 对所有 CPU 执行:
(gdb) thread apply all bt          # 所有 CPU 的调用栈
(gdb) thread apply all info registers  # 所有 CPU 的寄存器
```

### 6.4 vCont 精细控制

```gdb
# 继续 CPU#0, 暂停其他:
# (通过 RSP 层面实现, GDB 自动使用 vCont)

# 单步指定 CPU:
(gdb) thread 2
(gdb) stepi                        # 只单步 CPU#1
```

---

## 7. monitor 命令透传

### 7.1 基本用法

GDB 的 `monitor` 命令可以直接执行 **任何 HMP 命令**:

```gdb
(gdb) monitor help                 # 查看所有可用命令
(gdb) monitor info status          # VM 状态
(gdb) monitor info cpus            # CPU 列表
(gdb) monitor info registers       # 寄存器 (HMP格式)
(gdb) monitor info mtree           # 内存区域树
(gdb) monitor info tlb             # TLB 内容
(gdb) monitor info qtree           # 设备树
```

### 7.2 内存检查

```gdb
(gdb) monitor xp /10gx 0x40000000      # 物理内存
(gdb) monitor gpa2hva 0x40000000       # 地址转换
(gdb) monitor pmemsave 0x40000000 0x1000 /tmp/mem.bin  # 保存物理内存
```

### 7.3 运行控制

```gdb
(gdb) monitor system_reset         # 复位 VM
(gdb) monitor stop                 # 暂停 (与 Ctrl-C 类似)
(gdb) monitor cont                 # 恢复
(gdb) monitor quit                 # 退出 QEMU
```

### 7.4 设备操作

```gdb
(gdb) monitor info block           # 块设备
(gdb) monitor info network         # 网络设备
(gdb) monitor info pci             # PCI 设备
(gdb) monitor info irq             # 中断统计
(gdb) monitor info jit             # TCG 统计
```

### 7.5 快照

```gdb
(gdb) monitor savevm debug_snap    # 保存快照
(gdb) monitor loadvm debug_snap    # 恢复快照
(gdb) monitor info snapshots       # 列出快照
```

### 7.6 追踪

```gdb
(gdb) monitor trace-event gicv3_* on       # 启用 GIC 追踪
(gdb) monitor trace-event memory_region_* on
(gdb) monitor info trace-events            # 查看状态
```

### 7.7 日志控制

```gdb
(gdb) monitor log in_asm           # 启用翻译日志
(gdb) monitor log int              # 启用中断日志
(gdb) monitor log none             # 关闭日志
```

---

## 8. 反向调试

### 8.1 前提条件

反向调试需要 QEMU 的确定性重放 (Record/Replay):

```bash
# 录制:
qemu-system-aarch64 -M virt -cpu max -m 2G \
    -icount shift=auto,rr=record,rrfile=replay.bin \
    -kernel Image -nographic -net none \
    -drive file=disk.qcow2,if=virtio,snapshot=on

# 重放 + GDB:
qemu-system-aarch64 -M virt -cpu max -m 2G \
    -icount shift=auto,rr=replay,rrfile=replay.bin \
    -kernel Image -nographic -net none \
    -drive file=disk.qcow2,if=virtio,snapshot=on \
    -s -S
```

### 8.2 反向执行命令

```gdb
(gdb) target remote :1234

# 正向执行到感兴趣的位置:
(gdb) break panic
(gdb) continue
# ... 命中 panic ...

# 反向调试:
(gdb) reverse-continue            # 反向执行到上一个断点
(gdb) reverse-stepi               # 反向单步 (一条指令)
(gdb) reverse-step                # 反向单步 (一行源码)
(gdb) reverse-next                # 反向 next (不进入函数)
(gdb) reverse-finish              # 反向到当前函数入口
```

### 8.3 使用场景

```gdb
# 找到导致崩溃的写入:
(gdb) watch *corrupted_address    # 设置写观察点
(gdb) reverse-continue            # 反向到最后修改该地址的时刻
# 此时 $pc 指向导致问题的指令
(gdb) bt                          # 查看调用栈
```

### 8.4 qSupported 协商

QEMU 在 `qSupported` 回复中声明:
- `ReverseStep+` — 支持 `bs` (反向单步)
- `ReverseContinue+` — 支持 `bc` (反向继续)

仅当启用了 replay 模式时这些特性才可用。

---

## 9. 高级特性

### 9.1 XML 目标描述

QEMU 通过 `qXfer:features:read` 向 GDB 动态报告目标寄存器布局:

**静态 XML 文件** (`gdbstub/gdb-xml/`):
| 文件 | 内容 |
|------|------|
| `aarch64-core.xml` | x0-x30, sp, pc, cpsr |
| `aarch64-fpu.xml` | v0-v31, fpsr, fpcr |
| `aarch64-pauth.xml` | PAC 掩码伪寄存器 |
| `aarch64-mte.xml` | MTE 标签扩展 |
| `aarch64-sme2.xml` | SME2 zt0 |

**动态生成 XML**:
| 内容 | 生成位置 |
|------|----------|
| SVE (z/p/ffr/vg) | `gdbstub64.c:540-590` |
| SME (svg/svcr/za) | `gdbstub64.c:600-602` |
| TLS (tpidr*) | `gdbstub64.c:633-635` |
| 系统寄存器 | `gdbstub.c:324, 558-563` |

### 9.2 多进程模式

```
qSupported: multiprocess+

# 线程 ID 格式: pPID.TID
# 例: p1.1 = 进程1的线程1

# vAttach;pid — 附加到进程
# D;pid — 从进程分离
```

### 9.3 物理内存模式 (QEMU 扩展)

```gdb
# 查询当前模式:
(gdb) maintenance packet qqemu.PhyMemMode
# 回复: 0 (虚拟) 或 1 (物理)

# 设置物理模式:
(gdb) maintenance packet Qqemu.PhyMemMode:1
# 或:
(gdb) monitor qemu.PhyMemMode 1
```

### 9.4 单步模式配置 (QEMU 扩展)

```gdb
# 查询单步掩码:
(gdb) maintenance packet qqemu.sstep

# 设置单步掩码:
# 位含义: SSTEP_ENABLE(1) | SSTEP_NOIRQ(2) | SSTEP_NOTIMER(4)
(gdb) maintenance packet Qqemu.sstep=7
# 7 = 启用 + 屏蔽中断 + 屏蔽定时器 (最稳定调试体验)
```

### 9.5 用户模式特有功能

```gdb
# 系统调用捕获:
(gdb) catch syscall                # 所有系统调用
(gdb) catch syscall write read     # 特定系统调用

# 辅助向量:
(gdb) info auxv

# 信号信息:
(gdb) info signals
```

---

## 10. GDB 实用命令速查

### 10.1 执行控制

| 命令 | 缩写 | 功能 |
|------|------|------|
| `continue` | `c` | 继续执行 |
| `stepi` | `si` | 单步一条指令 |
| `step` | `s` | 单步一行源码 |
| `nexti` | `ni` | 下一条指令(不进入) |
| `next` | `n` | 下一行(不进入函数) |
| `finish` | `fin` | 执行完当前函数 |
| `until <loc>` | `u` | 执行到指定位置 |
| `advance <loc>` | - | 继续到指定位置 |

### 10.2 信息查看

| 命令 | 功能 |
|------|------|
| `info registers` | 所有寄存器 |
| `info registers system` | 系统寄存器 |
| `info float` | 浮点寄存器 |
| `info threads` | 所有 CPU/线程 |
| `bt` / `backtrace` | 调用栈 |
| `bt full` | 调用栈 + 局部变量 |
| `frame <n>` | 切换栈帧 |
| `disassemble` | 反汇编当前函数 |
| `disassemble /r` | 带原始字节 |
| `list` | 显示源码 |

### 10.3 内存与数据

| 命令 | 功能 |
|------|------|
| `x/Nfmt addr` | 检查内存 |
| `print expr` | 求值表达式 |
| `print/x $x0` | 十六进制打印寄存器 |
| `set $x0 = 0x1234` | 设置寄存器 |
| `set *addr = val` | 设置内存 |
| `find addr, +len, val` | 搜索内存 |
| `dump memory file addr1 addr2` | 保存内存到文件 |

### 10.4 ARM64 常用

```gdb
# 查看异常级别:
(gdb) print/x $cpsr & 0xc          # bits[3:2] = EL
# 0x0=EL0, 0x4=EL1, 0x8=EL2, 0xc=EL3

# 查看 PSTATE 标志:
(gdb) print/x ($cpsr >> 28) & 0xf  # NZCV

# 查看当前 SP:
(gdb) print/x $sp

# 链接寄存器 (返回地址):
(gdb) print/x $x30

# 系统寄存器 (通过 monitor):
(gdb) monitor info registers       # 包含 SCTLR_EL1 等

# 页表基地址:
(gdb) monitor info tlb
```

---

## 11. 调试场景实践

### 11.1 内核启动调试

```gdb
$ aarch64-linux-gnu-gdb vmlinux
(gdb) target remote :1234

# 设置内核入口断点:
(gdb) break start_kernel
(gdb) c
# ... 等待命中 ...

# 检查早期状态:
(gdb) print/x $pc
(gdb) print/x $cpsr
(gdb) bt
(gdb) info threads         # 确认只有 CPU#0 活跃
```

### 11.2 中断处理调试

```gdb
# 在中断向量设断点:
(gdb) break vectors + 0x280        # IRQ from EL1 (kernel IRQ)
(gdb) c

# 命中后检查:
(gdb) print/x $x0                  # 第一个参数
(gdb) monitor info irq            # 中断统计
(gdb) bt                          # 中断上下文调用栈
```

### 11.3 页错误调试

```gdb
# 在 page fault handler 设断点:
(gdb) break do_page_fault
(gdb) c

# 检查故障信息:
(gdb) print/x $far_el1             # 故障地址 (需系统寄存器)
(gdb) monitor info registers       # 看 ESR_EL1 获取故障类型
```

### 11.4 死锁/挂起诊断

```gdb
# VM 挂起时, Ctrl-C 中断:
^C
(gdb) thread apply all bt          # 所有 CPU 的调用栈
(gdb) thread apply all print/x $pc # 所有 CPU 的 PC

# 检查是否在 WFI:
(gdb) x/i $pc                     # 如果是 wfi, CPU 在等中断
```

### 11.5 内存损坏追踪

```gdb
# 监视特定地址:
(gdb) watch *0xffff800080300000
(gdb) c
# 命中时:
(gdb) bt                          # 谁写了这个地址
(gdb) print/x $x0                 # 写入的值是什么

# 结合反向调试:
(gdb) reverse-continue            # 回到上一次修改
```

### 11.6 用户模式程序调试

```bash
# 启动:
qemu-aarch64 -g 1234 -L /usr/aarch64-linux-gnu ./my_program

# GDB:
$ gdb-multiarch ./my_program
(gdb) target remote :1234
(gdb) break main
(gdb) c
(gdb) catch syscall openat         # 捕获文件打开
(gdb) c
```

---

## 12. 源码实现索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `gdbstub/gdbstub.c` | ~2300 | RSP 协议引擎, 命令分发 |
| `gdbstub/system.c` | ~680 | 系统模式: 断点/监控/多CPU |
| `gdbstub/user.c` | ~520 | 用户模式: 线程/fork/syscall |
| `target/arm/gdbstub.c` | ~600 | ARM 通用: 系统寄存器 XML 生成 |
| `target/arm/gdbstub64.c` | ~930 | ARM64: 核心/FP/SVE/SME/TLS/PAC/MTE |
| `gdbstub/gdb-xml/aarch64-core.xml` | - | 核心寄存器描述 |
| `gdbstub/gdb-xml/aarch64-fpu.xml` | - | FP/SIMD 描述 |
| `gdbstub/gdb-xml/aarch64-pauth.xml` | - | PAC 伪寄存器 |
| `gdbstub/gdb-xml/aarch64-mte.xml` | - | MTE 扩展 |

---

## 附录: qSupported 完整特性列表

连接时 QEMU 通过 `qSupported` 声明以下能力:

| 特性 | 含义 | 条件 |
|------|------|------|
| `PacketSize=<hex>` | 最大包大小 | 始终 |
| `qXfer:features:read+` | XML 目标描述 | 始终 |
| `vContSupported+` | 支持 vCont | 始终 |
| `multiprocess+` | 多进程调试 | 始终 |
| `ReverseStep+` | 反向单步 | Replay 模式 |
| `ReverseContinue+` | 反向继续 | Replay 模式 |
| `qXfer:auxv:read+` | 辅助向量 | User 模式 |
| `QCatchSyscalls+` | 系统调用捕获 | User 模式 (Linux) |
| `qXfer:siginfo:read+` | 信号信息 | User 模式 |
| `qXfer:exec-file:read+` | 可执行路径 | User 模式 |

---

*文档结束*
