# Doc 111: QEMU 性能分析与 Profiling 实践指南

## 文档信息
- **组件**: TCG Profiling, perf 集成, ftrace, 监控统计, PMU 仿真
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [概述与工具全景](#1-概述与工具全景)
2. [TCG 性能分析 (perf 集成)](#2-tcg-性能分析-perf-集成)
3. [QEMU 内置性能监控](#3-qemu-内置性能监控)
4. [Tracing 框架用于性能分析](#4-tracing-框架用于性能分析)
5. [Guest 内 perf/ftrace 使用](#5-guest-内-perftrace-使用)
6. [宿主机 perf 分析 QEMU 进程](#6-宿主机-perf-分析-qemu-进程)
7. [TCG 代码缓存调优](#7-tcg-代码缓存调优)
8. [PMU 仿真与 Guest 计数器](#8-pmu-仿真与-guest-计数器)
9. [VirtIO Balloon 统计](#9-virtio-balloon-统计)
10. [综合性能调优实战](#10-综合性能调优实战)
11. [常见性能问题诊断](#11-常见性能问题诊断)
12. [参考资源](#12-参考资源)

---

## 1. 概述与工具全景

### 1.1 QEMU 性能分析层次

```
┌─────────────────────────────────────────────┐
│  Guest 内工具                                │
│  perf, ftrace, /proc/stat, sar              │
├─────────────────────────────────────────────┤
│  QEMU 内部                                   │
│  trace events, sync-profile, TCG stats       │
├─────────────────────────────────────────────┤
│  宿主机工具                                   │
│  perf record, perf top, flamegraph           │
│  + QEMU jitdump/perfmap (JIT 符号解析)       │
└─────────────────────────────────────────────┘
```

### 1.2 工具适用场景速查

| 场景 | 推荐工具 | 说明 |
|------|----------|------|
| TCG 翻译热点 | `perf + -perfmap` | 定位哪些 Guest 代码块被频繁执行 |
| TCG 编译耗时 | `-d perf` 日志 + sync-profile | 翻译 vs 执行时间比 |
| Guest 内核函数热点 | Guest perf + PMU | 标准 Linux perf 工作流 |
| QEMU 自身 CPU 消耗 | 宿主 perf record | QEMU helper/device model 热点 |
| I/O 延迟 | trace events (virtio) | block/net 路径分析 |
| 锁竞争 | sync-profile | QEMU 全局锁/BQL 分析 |
| 内存带宽 | perf stat + LLC 事件 | 缓存缺失定位 |

---

## 2. TCG 性能分析 (perf 集成)

### 2.1 原理

TCG 动态生成 Host 机器码，默认对 perf 不可见（显示为 `[unknown]`）。QEMU 提供两种机制让 perf 解析 JIT 代码：

| 机制 | 文件 | 功能 |
|------|------|------|
| perfmap | `/tmp/perf-<pid>.map` | 简单地址→符号映射 |
| jitdump | `jit-<pid>.dump` | 完整 JIT 信息 (含源码行号) |

**源码位置**: `tcg/perf.c` (约 260 行)

### 2.2 使用 -perfmap

```bash
# 启动 QEMU (TCG 模式):
qemu-system-aarch64 \
    -M virt -cpu max -m 2G \
    -kernel Image -initrd rootfs.cpio.gz \
    -append "console=ttyAMA0 nokaslr" \
    -nographic \
    -perfmap

# 生成 /tmp/perf-<QEMU_PID>.map
# 格式: <起始地址> <大小> <TB名称>
# 例如: 7f1234000000 200 tb_guest_0x0000000040000000

# 宿主机录制:
perf record -p $(pidof qemu-system-aarch64) -g -- sleep 10
perf report
# 现在能看到 Guest TB 的符号名!
```

### 2.3 使用 -jitdump (更丰富)

```bash
qemu-system-aarch64 \
    -M virt -cpu max -m 2G \
    -kernel Image -nographic \
    -jitdump

# 生成 jit-<PID>.dump (ELF-like 格式)
# 包含: 代码加载事件, 调试信息, 行号映射

# 注入 perf 数据:
perf inject -j -i perf.data -o perf.data.jitted
perf report -i perf.data.jitted
# 可以看到 Guest 函数名 + 源码位置
```

### 2.4 源码分析: tcg/perf.c 关键函数

```c
// 初始化 perfmap 文件:
void perf_enable_perfmap(void)
{
    // 打开 /tmp/perf-<pid>.map
    // 每次 TB 翻译完成后写入条目
}

// 初始化 jitdump:
void perf_enable_jitdump(void)
{
    // 创建 jit-<pid>.dump
    // 写入 ELF header + jr_code_load 记录
}

// TB 翻译完成后报告:
void perf_report_code(uint64_t guest_pc, TranslationBlock *tb,
                      const void *start)
{
    // perfmap: write_perfmap_entry(guest_pc, size, name)
    // jitdump: write_jr_code_load(...)
}

// Prologue 代码报告:
void perf_report_prologue(const void *start, size_t size)
{
    // 记录 TCG prologue (exec loop 入口)
}
```

### 2.5 实战: 生成火焰图

```bash
# 录制 + perfmap:
qemu-system-aarch64 -M virt -cpu max -m 2G \
    -kernel Image -nographic -perfmap &
QEMU_PID=$!

perf record -F 99 -p $QEMU_PID -g -- sleep 30

# 生成火焰图:
perf script | stackcollapse-perf.pl | flamegraph.pl > qemu-flame.svg

# 火焰图中可见:
# - QEMU helper functions (helper_*)
# - TCG translated blocks (tb_guest_0x...)
# - Device model (virtio_*, gic_*)
```

---

## 3. QEMU 内置性能监控

### 3.1 sync-profile (锁/同步分析)

QEMU 提供内置的同步原语性能分析，用于定位 BQL (Big QEMU Lock) 竞争：

```bash
# 启动时启用:
qemu-system-aarch64 ... -enable-sync-profile

# HMP 命令:
(qemu) sync-profile on      # 开始采集
(qemu) sync-profile off     # 停止
(qemu) sync-profile reset   # 重置计数器
(qemu) info sync-profile    # 查看结果
```

**源码**: `monitor/hmp-cmds.c` → `hmp_sync_profile()` / `hmp_info_sync_profile()`

### 3.2 sync-profile 输出解读

```
Type        Object              Count    Avg Wait (ns)    Max Wait (ns)
mutex       bql                 154302   1250             89000
mutex       iothread            23100    340              12000
condvar     cpu_cond            8900     2100             150000
```

**关键指标**:
- **Count**: 锁获取次数
- **Avg Wait**: 平均等待时间 (越高越有问题)
- **Max Wait**: 最大等待 (尾延迟)

### 3.3 TCG 日志统计

```bash
# 查看翻译统计:
qemu-system-aarch64 ... -d out_asm,op_count

# 运行时获取:
(qemu) info jit    # JIT 代码缓存使用情况
(qemu) info opcount  # 操作码统计 (如果编译时启用)
```

### 3.4 代码缓存信息

```
(qemu) info jit
Translation buffer state:
gen code size       12345678/134217728 (9.2%)
TB count            98765
TB avg host size    125 bytes
TB avg guest size   32 bytes
cross page TB       1234
direct jump count   45678 (46%)
```

---

## 4. Tracing 框架用于性能分析

### 4.1 启用 ftrace 后端

```bash
# 编译时选择 ftrace 后端:
../configure --enable-trace-backends=ftrace

# 或运行时选择:
qemu-system-aarch64 ... \
    --trace "virtio_*" \
    --trace "memory_region_ops_*"
```

### 4.2 关键性能 Trace Events

```bash
# Block I/O 性能:
--trace "blk_*"
--trace "bdrv_*"
--trace "virtio_blk_*"

# 网络:
--trace "virtio_net_*"
--trace "net_*"

# 中断:
--trace "gic*"
--trace "kvm_irq*"

# 内存:
--trace "memory_region_ops_*"
--trace "flatview_*"

# TCG:
--trace "exec_tb*"
```

### 4.3 simple 后端分析

```bash
# 使用 simple 后端 (输出到文件):
qemu-system-aarch64 ... \
    --trace events=trace-events.txt,file=trace.log

# 分析:
scripts/simpletrace.py trace-events trace.log | \
    sort -k3 -n -r | head -20
# 按耗时排序前 20 事件
```

### 4.4 动态启用/禁用

```
(qemu) trace-event virtio_blk_* on    # 开启 VirtIO Block 追踪
(qemu) trace-event virtio_blk_* off   # 关闭
(qemu) info trace-events              # 查看所有事件状态
```

---

## 5. Guest 内 perf/ftrace 使用

### 5.1 前提条件

Guest 内核需要编译以下选项:

```
CONFIG_PERF_EVENTS=y
CONFIG_HW_PERF_EVENTS=y       # ARM PMU 支持
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_DYNAMIC_FTRACE=y
CONFIG_FTRACE_SYSCALLS=y
```

### 5.2 Guest perf 基本用法

```bash
# Guest 中:
perf stat ls                          # 基础计数
perf record -g -- ./workload          # 采样
perf report                           # 分析

# 注意: TCG 模式下 PMU 是模拟的，计数不精确
# KVM 模式下使用真实硬件 PMU，数据准确
```

### 5.3 Guest ftrace 用法

```bash
# Guest 中:
cd /sys/kernel/debug/tracing

# 函数追踪:
echo function > current_tracer
echo schedule > set_ftrace_filter
echo 1 > tracing_on
cat trace_pipe

# 函数图:
echo function_graph > current_tracer
echo do_page_fault > set_graph_function
echo 1 > tracing_on
cat trace

# 事件追踪:
echo 1 > events/sched/sched_switch/enable
echo 1 > tracing_on
cat trace_pipe
```

### 5.4 TCG 模式下的局限性

| 特性 | KVM 模式 | TCG 模式 |
|------|----------|----------|
| PMU 计数器 | 精确 (硬件) | 近似 (模拟) |
| 采样中断 | 硬件 PMI | 定时器模拟 |
| cycles 事件 | 真实 | icount 或 host 定时器 |
| cache 事件 | 硬件 LLC/L1 | 不支持 |
| branch 事件 | 硬件 BPU | 不支持 |

---

## 6. 宿主机 perf 分析 QEMU 进程

### 6.1 基本采样

```bash
# 对 QEMU 进程采样:
perf record -p $(pidof qemu-system-aarch64) -g --call-graph dwarf -- sleep 30

# 分析:
perf report --hierarchy
```

### 6.2 典型热点分析

```
宿主 perf report 常见热点:
─────────────────────────────
  35%  cpu_exec_loop            ← TCG 执行循环
  15%  helper_lookup_tb_ptr     ← TB 查找
   8%  tcg_gen_code             ← TCG 编译
   5%  address_space_ldq        ← MMIO 访问
   4%  gic_set_irq              ← 中断模拟
   3%  virtio_queue_notify      ← VirtIO 通知
```

### 6.3 perf stat 全局概览

```bash
perf stat -p $(pidof qemu-system-aarch64) -- sleep 10

# 输出:
#  Performance counter stats:
#  12,345,678,900  cycles
#   3,456,789,000  instructions      # 0.28 IPC
#      45,678,000  cache-misses      # 1.2% of cache refs
#     123,456,000  branch-misses     # 3.4% of branches
```

### 6.4 针对性分析

```bash
# 分析缓存行为:
perf stat -e cache-references,cache-misses,LLC-load-misses \
    -p $(pidof qemu-system-aarch64) -- sleep 10

# 分析分支预测:
perf stat -e branches,branch-misses \
    -p $(pidof qemu-system-aarch64) -- sleep 10

# TLB 压力:
perf stat -e dTLB-load-misses,iTLB-load-misses \
    -p $(pidof qemu-system-aarch64) -- sleep 10
```

---

## 7. TCG 代码缓存调优

### 7.1 代码缓存大小

```bash
# 默认大小:
# - linux-user: 128 MB
# - system: ~1 GB (取决于 RAM)

# 自定义:
qemu-system-aarch64 ... -accel tcg,tb-size=256  # 256 MB

# 查看使用率:
(qemu) info jit
```

### 7.2 缓存满时行为

**源码**: `tcg/region.c`

```
代码缓存满 → region_reset() → 清空整个 region
→ 所有 TB 失效 → 重新翻译

监测指标:
- "gen code size" 接近总量 → 缓存压力大
- "TB count" 突然归零 → 发生了 flush
```

### 7.3 TB 链接优化

```bash
# 查看直接跳转比例:
(qemu) info jit
# "direct jump count" — 越高越好

# 影响因素:
# - 跨页 TB 无法链接 (cross page TB)
# - 自修改代码导致失效
# - 中断打断链
```

### 7.4 多线程 TCG (MTTCG)

```bash
# 启用多线程 TCG:
qemu-system-aarch64 ... -accel tcg,thread=multi

# 每个 vCPU 一个宿主线程
# 性能提升但增加同步开销
# 适合多核 Guest 工作负载
```

---

## 8. PMU 仿真与 Guest 计数器

### 8.1 ARM PMU 仿真

**源码**: `target/arm/cpregs-pmu.c`

```c
// 支持的事件 (pm_events[]):
// - SWINC (0x00): 软件自增
// - INST_RETIRED (0x08): 指令退休
// - CPU_CYCLES (0x11): CPU 周期
// - STALL_FRONTEND (0x23): 前端停顿
// - STALL_BACKEND (0x24): 后端停顿

void pmu_init(ARMCPU *cpu)
{
    // 初始化 PMU 计数器
    // 连接到 icount 或 host 时间
}
```

### 8.2 启用 PMU

```bash
# 启用 PMU (TCG):
qemu-system-aarch64 -M virt -cpu max,pmu=on ...

# KVM 模式 (使用真实 PMU):
qemu-system-aarch64 -M virt -cpu host,pmu=on -enable-kvm ...
```

### 8.3 TCG PMU 局限

- **CPU_CYCLES**: 基于 icount 或 host 纳秒时钟, 不反映微架构
- **INST_RETIRED**: 每条翻译指令计数, 相对准确
- **CACHE 事件**: 不支持 (TCG 没有缓存模型)
- **BRANCH 事件**: 不支持

---

## 9. VirtIO Balloon 统计

### 9.1 内存统计收集

**源码**: `hw/virtio/virtio-balloon.c`

```bash
# 启用:
qemu-system-aarch64 ... \
    -device virtio-balloon-pci,id=balloon0

# QMP 查询:
{"execute": "qom-get",
 "arguments": {"path": "/machine/peripheral/balloon0",
               "property": "guest-stats"}}
```

### 9.2 可用统计项

| 统计项 | 含义 |
|--------|------|
| stat-swap-in | 换入页数 |
| stat-swap-out | 换出页数 |
| stat-major-faults | 主要缺页 |
| stat-minor-faults | 次要缺页 |
| stat-free-memory | 空闲内存 |
| stat-total-memory | 总内存 |
| stat-available-memory | 可用内存 |

### 9.3 轮询配置

```bash
# 设置统计轮询间隔 (秒):
{"execute": "qom-set",
 "arguments": {"path": "/machine/peripheral/balloon0",
               "property": "guest-stats-polling-interval",
               "value": 2}}
```

---

## 10. 综合性能调优实战

### 10.1 场景: Guest 启动缓慢

```bash
# 步骤 1: 宿主 perf 定位瓶颈
perf record -p $QEMU_PID -g -- sleep 30
perf report

# 可能发现:
# 1) tcg_gen_code 占比高 → 代码缓存太小或大量新代码
# 2) address_space_ldq 高 → MMIO 频繁
# 3) helper_* 高 → 复杂指令模拟

# 步骤 2: TCG 编译分析
(qemu) info jit
# 检查 TB count 和 code size

# 步骤 3: 增大代码缓存
qemu-system-aarch64 ... -accel tcg,tb-size=512
```

### 10.2 场景: I/O 性能差

```bash
# 步骤 1: 启用 block trace:
qemu-system-aarch64 ... --trace "virtio_blk_*,blk_*"

# 步骤 2: 检查是否有不必要的同步写:
# trace 中看到大量 blk_aio_flush → 减少 Guest fsync

# 步骤 3: 优化后端:
# - 使用 io_uring: -drive file=x.qcow2,aio=io_uring
# - 使用 cache=none: 绕过宿主 page cache
# - 启用 VirtIO multiqueue
```

### 10.3 场景: 多核扩展性差

```bash
# 步骤 1: sync-profile 检查锁竞争:
(qemu) sync-profile on
# ... 运行一段时间 ...
(qemu) info sync-profile

# 如果 BQL 竞争严重:
# - 检查是否有设备在 BQL 下做大量工作
# - 考虑 iothread: -object iothread,id=iothread0

# 步骤 2: MTTCG 检查:
# 确保使用 thread=multi
qemu-system-aarch64 ... -accel tcg,thread=multi -smp 4
```

### 10.4 场景: 中断风暴

```bash
# Guest 内:
cat /proc/interrupts
# 如果某个 IRQ 计数异常高

# QEMU 侧:
(qemu) info irq
# 或:
qemu-system-aarch64 ... --trace "gic_set_irq*"

# 定位后:
# - 检查是否 timer 频率过高
# - 网络: 启用 interrupt coalescing
# - 存储: 使用 MSI-X + multiqueue
```

---

## 11. 常见性能问题诊断

### 11.1 诊断决策树

```
性能问题
├── CPU 利用率高 (宿主机)
│   ├── perf top → tcg_gen_code 高 → 代码缓存/自修改代码
│   ├── perf top → helper_* 高 → 复杂指令, 考虑 KVM
│   └── perf top → device model 高 → 减少 MMIO, 用 VirtIO
├── CPU 利用率低但 Guest 慢
│   ├── sync-profile → 锁等待高 → iothread / 减少 BQL 持有
│   └── info jit → flush 频繁 → 增大 tb-size
├── I/O 延迟高
│   ├── trace virtio_blk → 排队延迟 → multiqueue
│   └── 宿主 iostat → 宿主盘慢 → io_uring / SSD
└── 内存问题
    ├── perf stat → TLB miss 高 → hugepage
    └── balloon stats → Guest 内存不足 → 增加 -m
```

### 11.2 关键调优参数汇总

| 参数 | 默认值 | 调优建议 |
|------|--------|----------|
| `-accel tcg,tb-size=N` | 自动 | 重 workload 增至 512MB |
| `-accel tcg,thread=multi` | single | 多核必开 |
| `-m N` | - | 留 Guest 足够内存 |
| `-smp N` | 1 | 匹配宿主核心数 |
| `aio=io_uring` | threads | Linux 5.1+ 推荐 |
| `cache=none` | writeback | 数据库/高 IOPS 场景 |
| `mem-prealloc=on` | off | 减少运行时缺页 |
| `-object iothread` | 无 | 解耦 I/O 与 vCPU |

---

## 12. 参考资源

### 12.1 源码文件索引

| 文件 | 功能 |
|------|------|
| `tcg/perf.c` | perfmap + jitdump 实现 |
| `tcg/region.c` | 代码缓存 region 管理 |
| `include/tcg/perf.h` | perf 集成 API |
| `monitor/hmp-cmds.c` | sync-profile HMP 命令 |
| `target/arm/cpregs-pmu.c` | ARM PMU 仿真 |
| `hw/virtio/virtio-balloon.c` | Balloon 统计 |
| `trace/ftrace.c` | ftrace 后端 |
| `trace/simple.c` | simple 文件后端 |

### 12.2 相关文档

| 编号 | 主题 | 关联 |
|------|------|------|
| 102 | 监测与追踪系统 | Trace events 详解 |
| 103 | 调试系统 | TCG 日志分类 |
| 44 | TCG 执行循环 | TB 生命周期 |
| 107 | TCG vs JIT 对比 | 代码缓存/优化设计 |
| 110 | 内核调试工作流 | GDB 环境搭建 |

---

*文档结束*
