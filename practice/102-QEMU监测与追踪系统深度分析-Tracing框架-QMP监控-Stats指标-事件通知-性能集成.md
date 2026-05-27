# Doc 102: QEMU 监测与追踪系统深度分析

## 文档信息
- **组件**: Tracing Framework, QMP Monitoring, Stats/Metrics, 事件系统
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [概述](#1-概述)
2. [Tracing 追踪框架](#2-tracing-追踪框架)
3. [QMP 监控接口](#3-qmp-监控接口)
4. [Stats 统计框架](#4-stats-统计框架)
5. [日志系统](#5-日志系统)
6. [事件通知系统](#6-事件通知系统)
7. [HMP 信息命令](#7-hmp-信息命令)
8. [性能集成](#8-性能集成)
9. [健康与状态监测](#9-健康与状态监测)
10. [实践指南](#10-实践指南)

---

## 1. 概述

QEMU 提供了多层监测能力：

| 层级 | 机制 | 用途 |
|------|------|------|
| 底层 | Tracing Framework | 细粒度事件追踪, 性能分析 |
| 中层 | Logging System | 调试日志输出 |
| 应用层 | QMP/HMP | 运行时查询/控制 |
| 统计层 | Stats Framework | 结构化性能指标 |
| 事件层 | QMP Events | 异步状态变更通知 |

```
┌─────────────────────────────────────────────────────────┐
│                    管理工具/客户端                        │
│  (libvirt, virsh, virt-manager, 自定义脚本)             │
└────────────┬───────────────────────┬────────────────────┘
             │ QMP (JSON)            │ HMP (文本)
             ▼                       ▼
┌─────────────────────────────────────────────────────────┐
│              Monitor 子系统                               │
│  QMP 命令 ← QAPI Schema → HMP 命令                     │
│  事件发射 (SHUTDOWN/RESET/GUEST_PANICKED/...)           │
└────────────┬───────────────────────┬────────────────────┘
             │                       │
     ┌───────┴───────┐      ┌───────┴────────┐
     │ Stats 框架    │      │  Trace 框架     │
     │ (结构化指标)  │      │  (6种后端)      │
     └───────────────┘      └────────────────┘
```

---

## 2. Tracing 追踪框架

### 2.1 架构设计

QEMU 的 Tracing 框架是一个编译时 + 运行时混合的高性能事件追踪系统：

```
┌──────────────────────┐
│  trace-events 文件   │  ← 各子系统声明追踪点
│  (每目录一个)        │
└──────────┬───────────┘
           │ tracetool (Python 代码生成)
           ▼
┌──────────────────────┐
│  trace-*.h / trace-*.c│  ← 自动生成的桩函数
│  trace_<name>(args)  │
└──────────┬───────────┘
           │ 运行时调用
           ▼
┌──────────────────────────────────────────┐
│            Trace Backend                  │
│  ┌─────┐ ┌──────┐ ┌─────┐ ┌──────────┐ │
│  │simple│ │ftrace│ │ log │ │  dtrace  │ │
│  └─────┘ └──────┘ └─────┘ └──────────┘ │
│  ┌──────┐ ┌─────┐                       │
│  │syslog│ │ ust │                       │
│  └──────┘ └─────┘                       │
└──────────────────────────────────────────┘
```

### 2.2 trace-events 格式

```c
// trace-events (根目录和各子目录)
// 格式: [disable] <name>(<type> <arg>, ...) "<format-string>"

// 示例:
memory_region_ops_read(void *mr, uint64_t addr, uint64_t value, unsigned size, const char *name) "mr %p addr 0x%"PRIx64" value 0x%"PRIx64" size %u name '%s'"
memory_region_ops_write(void *mr, uint64_t addr, uint64_t value, unsigned size, const char *name) "mr %p addr 0x%"PRIx64" value 0x%"PRIx64" size %u name '%s'"

// disable 前缀表示默认禁用:
disable kvm_run_exit(int cpu_index, uint32_t reason) "cpu_index %d, reason %d"
```

### 2.3 六种追踪后端

| 后端 | 特点 | 开销 | 适用场景 |
|------|------|------|----------|
| **simple** | 二进制文件写入, 缓冲 IO | 低 | 离线分析, 自动化测试 |
| **ftrace** | 写入 Linux ftrace ring buffer | 极低 | 与内核 trace 统一分析 |
| **log** | 直接写入 stderr/logfile | 中 | 快速调试 |
| **dtrace** | DTrace/SystemTap 探针 | 极低(未激活时) | 生产环境动态追踪 |
| **syslog** | 写入 syslog | 中 | 系统级日志集成 |
| **ust** | LTTng UST 用户态追踪 | 低 | 高性能用户态追踪 |

### 2.4 控制 API

```c
// trace/control.h:15-212

// 迭代所有追踪事件
TraceEventIter iter;
TraceEvent *ev;
trace_event_iter_init_all(&iter);
while ((ev = trace_event_iter_next(&iter)) != NULL) {
    // ev->name, ev->state
}

// 启用/禁用事件 (支持 glob 模式)
trace_event_set_state_dynamic(ev, true);   // 动态启用
trace_enable_events("memory_region_*");    // 按模式启用

// 初始化
trace_init_backends();                     // 初始化后端
trace_init_file("/path/to/trace.log");     // 设置输出文件
```

### 2.5 simple 后端实现

```c
// trace/simple.c:1-220
// 二进制格式:
// - 文件头: magic + version
// - 事件记录: event_id + timestamp + args...
// - dropped 记录: 当缓冲满时记录丢弃计数

// 使用独立的写入线程 + 环形缓冲区
// 最小化对主线程的影响
```

### 2.6 代码生成 (tracetool)

```
scripts/tracetool/
├── __init__.py          # 主入口
├── backend/
│   ├── simple.py        # simple 后端代码生成
│   ├── ftrace.py        # ftrace 后端
│   ├── log.py           # log 后端
│   ├── dtrace.py        # DTrace 探针
│   ├── syslog.py        # syslog 后端
│   └── ust.py           # LTTng UST 后端
└── format/
    ├── h.py             # 生成 .h 头文件
    ├── c.py             # 生成 .c 实现
    ├── rs.py            # 生成 Rust 绑定
    ├── d.py             # DTrace 脚本
    └── stap.py          # SystemTap 脚本
```

构建时由 `trace/meson.build:16-99` 驱动代码生成。

---

## 3. QMP 监控接口

### 3.1 QMP 协议概述

QMP (QEMU Machine Protocol) 是基于 JSON 的管理接口：

```json
// 连接时的 greeting:
{"QMP": {"version": {"qemu": {"micro": 50, "minor": 0, "major": 11}}, "capabilities": []}}

// 协商:
→ {"execute": "qmp_capabilities"}
← {"return": {}}

// 查询:
→ {"execute": "query-status"}
← {"return": {"status": "running", "running": true}}
```

### 3.2 可查询的监控数据

| QMP 命令 | 返回信息 | 用途 |
|----------|----------|------|
| `query-status` | 运行状态 (running/paused/...) | 生命周期监控 |
| `query-cpus-fast` | vCPU 列表/线程 ID/架构信息 | CPU 监控 |
| `query-memory-size-summary` | 内存总量/Plugged | 内存监控 |
| `query-memory-devices` | 热插内存设备列表 | DIMM 管理 |
| `query-balloon` | 气球设备当前大小 | 内存调整 |
| `query-block` | 块设备列表 | 存储监控 |
| `query-blockstats` | 块设备 IO 统计 | IO 性能 |
| `query-iothreads` | IOThread 列表 | 线程监控 |
| `query-migrate` | 迁移进度/状态 | 迁移监控 |
| `query-name` | VM 名称 | 标识 |
| `query-uuid` | VM UUID | 标识 |
| `query-machines` | 支持的机器类型 | 能力发现 |
| `query-stats` | 新统计框架指标 | 性能分析 |
| `query-dump` | Dump 进度 | Core dump 监控 |

### 3.3 QAPI Schema 定义

```json
// qapi/run-state.json:105-133
{ 'struct': 'StatusInfo',
  'data': {'running': 'bool',
           'status': 'RunState'} }

{ 'command': 'query-status',
  'returns': 'StatusInfo' }

// qapi/machine.json:107-142
{ 'command': 'query-cpus-fast',
  'returns': ['CpuInfoFast'] }
```

---

## 4. Stats 统计框架

### 4.1 架构

新的统计框架（QEMU 7.1+）提供结构化的性能指标收集：

```c
// include/system/stats.h:13-43

// 回调类型:
typedef void (*StatRetrieveFunc)(StatsResultList **results, StatsFilter *filter, ...);
typedef void (*SchemaRetrieveFunc)(StatsSchemaList **schemas, ...);

// 注册接口:
void add_stats_callbacks(StatsProvider provider,
                         StatRetrieveFunc stats_fn,
                         SchemaRetrieveFunc schemas_fn);

// 查询结果构建:
void add_stats_entry(StatsResultList **results, ...);
void add_stats_schema(StatsSchemaList **schemas, ...);
```

### 4.2 提供者注册

```c
// 各子系统注册为 stats provider:
// 例如 KVM, TCG, vhost 等都可以注册统计回调

add_stats_callbacks(STATS_PROVIDER_KVM,
                    kvm_query_stats,
                    kvm_query_stats_schemas);
```

### 4.3 查询接口

```json
// QMP:
→ {"execute": "query-stats", "arguments": {"target": "vm"}}
← {"return": [{"provider": "kvm", "stats": [...]}]}

→ {"execute": "query-stats-schemas", "arguments": {"provider": "kvm"}}
← {"return": [...]}
```

```
// HMP:
(qemu) info stats
```

### 4.4 实现

```c
// stats/stats-qmp-cmds.c:14-165
// - QTAILQ 存储注册的 callback
// - qmp_query_stats() 遍历所有 provider, 调用各自 stats_fn
// - qmp_query_stats_schemas() 获取 schema 定义

// stats/stats-hmp-cmds.c:17-252
// - HMP "info stats" 命令
// - 构造 StatsFilter → 调用 QMP 层 → 格式化输出
```

---

## 5. 日志系统

### 5.1 日志分类

```c
// include/qemu/log.h:17-40  +  util/log.c:484-533

// TCG 翻译相关:
CPU_LOG_TB_OUT_ASM    "out_asm"     // 宿主机生成代码
CPU_LOG_TB_IN_ASM     "in_asm"      // 目标机原始汇编
CPU_LOG_TB_OP         "op"          // TCG 中间操作
CPU_LOG_TB_OP_OPT     "op_opt"      // 优化后的 TCG 操作
CPU_LOG_TB_OP_IND     "op_ind"      // indirect lowering 前

// 执行相关:
CPU_LOG_EXEC          "exec"        // TB 执行追踪
CPU_LOG_INT           "int"         // 中断/异常
CPU_LOG_PCALL         "pcall"       // 特权调用

// CPU 状态:
CPU_LOG_CPU           "cpu"         // 执行前 CPU 状态
CPU_LOG_FPU           "fpu"         // FPU 状态
CPU_LOG_VPU           "vpu"         // 向量处理单元
CPU_LOG_RESET         "cpu_reset"   // CPU 复位

// 内存:
CPU_LOG_MMU           "mmu"         // MMU 操作
CPU_LOG_PAGE          "page"        // 页表操作
CPU_LOG_INVALID_MEM   "invalid_mem" // 非法内存访问

// 其他:
LOG_UNIMP             "unimp"       // 未实现特性
LOG_GUEST_ERROR       "guest_errors"// Guest 错误
LOG_STRACE            "strace"      // linux-user 系统调用
LOG_NOCHAIN           "nochain"     // 禁止 TB 链
LOG_PLUGIN            "plugin"      // TCG 插件
LOG_TID               "tid"         // 线程 ID 前缀
```

### 5.2 日志 API

```c
// include/qemu/log.h

// 检查是否启用
bool qemu_log_enabled(void);
bool qemu_loglevel_mask(int mask);

// 输出日志 (需加锁)
FILE *f = qemu_log_trylock();
if (f) {
    fprintf(f, "...");
    qemu_log_unlock(f);
}

// 便捷宏
qemu_log("simple message\n");
qemu_log_mask(CPU_LOG_MMU, "MMU: addr=0x%lx\n", addr);

// 配置
qemu_set_log(int log_flags);
qemu_set_log_filename(const char *filename);
```

### 5.3 命令行使用

```bash
# 基本日志:
qemu-system-aarch64 -d in_asm,out_asm       # 翻译日志
qemu-system-aarch64 -d cpu,exec             # 执行追踪
qemu-system-aarch64 -d int                  # 中断日志
qemu-system-aarch64 -d mmu,page             # 内存管理日志
qemu-system-aarch64 -d unimp,guest_errors   # 问题诊断

# 输出到文件:
qemu-system-aarch64 -d in_asm -D /tmp/qemu.log

# 组合:
qemu-system-aarch64 -d in_asm,op,op_opt,out_asm -D /tmp/tcg.log

# 运行时修改 (HMP):
(qemu) log in_asm
(qemu) log none
```

---

## 6. 事件通知系统

### 6.1 QMP 事件

QEMU 通过 QMP 异步事件通知管理工具状态变更：

| 事件 | 触发条件 | 含义 |
|------|----------|------|
| `SHUTDOWN` | VM 关机 | Guest 请求关机 |
| `POWERDOWN` | 电源按钮 | 外部关机请求 |
| `RESET` | 系统复位 | VM 重启 |
| `STOP` | VM 暂停 | 执行停止 |
| `RESUME` | VM 恢复 | 执行恢复 |
| `SUSPEND` | 挂起 | Guest 进入 S3 |
| `SUSPEND_DISK` | 休眠 | Guest 进入 S4 |
| `WAKEUP` | 唤醒 | 从挂起/休眠恢复 |
| `WATCHDOG` | 看门狗超时 | Guest 无响应 |
| `GUEST_PANICKED` | 内核 panic | Guest 崩溃 |
| `GUEST_CRASHLOADED` | Crash kernel 加载 | kdump 就绪 |
| `MEMORY_FAILURE` | 内存错误 | 硬件错误模拟 |
| `DEVICE_DELETED` | 设备移除 | 热拔完成 |
| `BALLOON_CHANGE` | 气球大小变化 | 内存调整完成 |

### 6.2 事件格式

```json
// 异步推送给 QMP 客户端:
{"event": "GUEST_PANICKED",
 "data": {"action": "pause",
           "info": {"type": "hyper-v", "data": {...}}},
 "timestamp": {"seconds": 1625000000, "microseconds": 123456}}
```

### 6.3 RunState 状态机

```
                    ┌──────────┐
                    │ prelaunch │
                    └─────┬────┘
                          │ 启动
                          ▼
┌───────────┐      ┌──────────┐      ┌───────────┐
│  paused   │◄────►│ running  │─────►│  shutdown │
└───────────┘      └──────┬───┘      └───────────┘
      ▲                   │
      │            ┌──────┴───────────────┐
      │            ▼                       ▼
      │     ┌──────────┐          ┌──────────────┐
      ├─────│ suspended│          │guest-panicked│
      │     └──────────┘          └──────────────┘
      │            │
      │            ▼
      │     ┌───────────┐
      └─────│  inmigrate│
            └───────────┘
```

RunState 枚举值（`qapi/run-state.json:12-62`）:
`debug`, `finish-migrate`, `inmigrate`, `internal-error`, `io-error`,
`paused`, `postmigrate`, `prelaunch`, `restore-vm`, `running`,
`save-vm`, `shutdown`, `suspended`, `watchdog`, `guest-panicked`,
`colo`

---

## 7. HMP 信息命令

### 7.1 完整 info 命令列表 (71 个)

**系统状态:**
| 命令 | 说明 |
|------|------|
| `info status` | VM 运行状态 |
| `info version` | QEMU 版本 |
| `info name` | VM 名称 |
| `info uuid` | VM UUID |
| `info cpus` | vCPU 列表 |
| `info kvm` | KVM 状态 |
| `info accel` | 加速器状态 |

**内存:**
| 命令 | 说明 |
|------|------|
| `info mtree` | MemoryRegion 树 |
| `info mem` | 虚拟地址空间映射 |
| `info ramblock` | RAM 块列表 |
| `info memory-devices` | 热插内存设备 |
| `info memory_size_summary` | 内存大小汇总 |
| `info numa` | NUMA 拓扑 |
| `info tlb` | TLB 内容 |

**设备:**
| 命令 | 说明 |
|------|------|
| `info pci` | PCI 设备树 |
| `info qtree` | 完整设备树 |
| `info qom-tree` | QOM 对象树 |
| `info qdm` | 设备模型列表 |
| `info usb` | USB 设备 |
| `info chardev` | 字符设备 |
| `info mice` | 输入设备 |

**块设备/存储:**
| 命令 | 说明 |
|------|------|
| `info block` | 块设备列表 |
| `info blockstats` | 块设备 IO 统计 |
| `info block-jobs` | 后台块操作 |
| `info snapshots` | 快照列表 |

**网络:**
| 命令 | 说明 |
|------|------|
| `info network` | 网络后端 |
| `info usernet` | SLIRP 信息 |

**显示:**
| 命令 | 说明 |
|------|------|
| `info vnc` | VNC 连接状态 |
| `info spice` | SPICE 连接状态 |

**性能/调试:**
| 命令 | 说明 |
|------|------|
| `info jit` | TCG JIT 统计 |
| `info sync-profile` | 锁竞争分析 |
| `info trace-events` | 追踪事件状态 |
| `info registers` | CPU 寄存器 |
| `info irq` | 中断统计 |
| `info pic` | 中断控制器状态 |
| `info stats` | 统计框架指标 |

**迁移:**
| 命令 | 说明 |
|------|------|
| `info migrate` | 迁移状态 |
| `info migrate_capabilities` | 迁移能力 |
| `info migrate_parameters` | 迁移参数 |
| `info dirty_rate` | 脏页速率 |

**其他:**
| 命令 | 说明 |
|------|------|
| `info hotpluggable-cpus` | 可热插 CPU |
| `info iothreads` | IOThread |
| `info replay` | 重放状态 |
| `info balloon` | 气球大小 |
| `info virtio` | VirtIO 设备 |
| `info tpm` | TPM 设备 |
| `info roms` | ROM 列表 |

---

## 8. 性能集成

### 8.1 Linux perf 集成

```c
// tcg/perf.c:1-220
// 生成 perf map 文件, 使 perf 能识别 TCG 生成的代码

void perf_enable_perfmap(void)
{
    // 创建 /tmp/perf-<pid>.map
    // 每个 TB 翻译完成后写入:
    // <host_addr> <size> <guest_pc>_<tb_name>
}

void perf_enable_jitdump(void)
{
    // 生成 jitdump 格式
    // 支持 perf inject --jit 反解
    // 包含完整的代码段信息
}
```

### 8.2 使用方法

```bash
# 启用 perf map:
qemu-system-aarch64 -accel tcg,perfmap=true ...

# 使用 jitdump (更详细):
qemu-system-aarch64 -accel tcg,jitdump=true ...

# 然后用 perf 分析:
perf record -p $(pidof qemu-system-aarch64) sleep 10
perf inject --jit -i perf.data -o perf.jit.data
perf report -i perf.jit.data
```

### 8.3 ftrace 集成

```bash
# 编译时选择 ftrace 后端:
../configure --enable-trace-backends=ftrace

# 运行时启用特定事件:
echo 1 > /sys/kernel/debug/tracing/tracing_on
qemu-system-aarch64 --trace "memory_region_*" ...

# 查看结果:
cat /sys/kernel/debug/tracing/trace
```

---

## 9. 健康与状态监测

### 9.1 运行时健康检查

```bash
# QMP 查询 VM 状态:
echo '{"execute": "query-status"}' | nc -U /tmp/qemu.sock
# → {"return": {"status": "running", "running": true}}

# 检查 Guest Agent 响应:
echo '{"execute": "guest-ping"}' | nc -U /tmp/qga.sock
```

### 9.2 看门狗集成

```bash
# 配置看门狗:
qemu-system-aarch64 -watchdog-action pause ...

# Guest 无响应时 QEMU 发出:
# {"event": "WATCHDOG", "data": {"action": "pause"}}
```

### 9.3 Guest Panic 检测

```bash
# KVM 模式下, Guest panic 通过 pvpanic 设备或 KVM 机制检测:
-device pvpanic-pci

# Guest panic 时:
# {"event": "GUEST_PANICKED", "data": {"action": "pause"}}
```

---

## 10. 实践指南

### 10.1 基本监测配置

```bash
# 最小 QMP 监控 (Unix socket):
qemu-system-aarch64 \
    -qmp unix:/tmp/qemu-monitor.sock,server,nowait \
    ...

# 多个 monitor:
qemu-system-aarch64 \
    -qmp unix:/tmp/qmp.sock,server,nowait \
    -monitor telnet:127.0.0.1:4444,server,nowait \
    ...
```

### 10.2 追踪配置

```bash
# 方法 1: 命令行
qemu-system-aarch64 --trace "virtio_*" --trace "memory_region_ops_*" ...

# 方法 2: 事件文件
echo "virtio_queue_notify" > /tmp/events
echo "memory_region_ops_read" >> /tmp/events
qemu-system-aarch64 --trace events=/tmp/events ...

# 方法 3: 运行时 (HMP)
(qemu) trace-event virtio_queue_notify on
(qemu) trace-event memory_region_* on
(qemu) info trace-events
```

### 10.3 性能分析工作流

```bash
# 1. TCG 性能:
qemu-system-aarch64 -accel tcg,perfmap=true ...
perf record -g -p $(pidof qemu-system-aarch64) sleep 30
perf report

# 2. 内部统计:
(qemu) info jit              # TCG 翻译统计
(qemu) info sync-profile     # 锁竞争
(qemu) info stats            # 结构化指标

# 3. 块 IO:
(qemu) info blockstats       # 读写统计

# 4. 追踪热路径:
qemu-system-aarch64 --trace "kvm_run_exit" --trace "kvm_*" -D /tmp/trace.log
```

### 10.4 故障诊断

```bash
# Guest 启动失败:
qemu-system-aarch64 -d unimp,guest_errors -D /tmp/err.log ...

# 中断问题:
qemu-system-aarch64 -d int ...
(qemu) info irq

# 内存问题:
(qemu) info mtree            # 内存布局
(qemu) info tlb              # TLB 状态
(qemu) xp /10gx 0x40000000  # 物理内存检查

# 设备问题:
(qemu) info qtree            # 设备拓扑
(qemu) info pci              # PCI 设备
```

---

## 附录 A: 源文件索引

| 文件 | 职责 |
|------|------|
| `trace/control.h` | Trace 控制 API |
| `trace/control.c` | Trace 初始化/启用/禁用 |
| `trace/simple.c` | Simple 二进制后端 |
| `trace/ftrace.c` | Ftrace 后端 |
| `trace/meson.build` | 构建配置 |
| `scripts/tracetool/` | 代码生成工具 |
| `include/system/stats.h` | Stats 框架 API |
| `stats/stats-qmp-cmds.c` | Stats QMP 实现 |
| `stats/stats-hmp-cmds.c` | Stats HMP 实现 |
| `util/log.c` | 日志系统实现 |
| `include/qemu/log.h` | 日志 API/分类 |
| `tcg/perf.c` | Perf map/jitdump |
| `qapi/run-state.json` | 状态机/事件定义 |
| `monitor/qmp-cmds.c` | QMP 命令实现 |
| `monitor/hmp-cmds.c` | HMP 命令实现 |
| `hmp-commands-info.hx` | 71 个 info 命令定义 |

---

*文档结束*
