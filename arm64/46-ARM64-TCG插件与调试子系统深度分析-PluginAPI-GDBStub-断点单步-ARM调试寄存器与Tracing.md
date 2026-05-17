# ARM64 TCG 插件与调试子系统深度分析：Plugin API、GDB Stub、断点/单步、ARM 调试寄存器与 Tracing

> 基于 QEMU 11.0.50 源码分析，涵盖 TCG 插件与调试完整子系统：
> TCG Plugin 公共 API（qemu_plugin_install 入口、register_vcpu_tb_trans_cb/insn_exec_cb/mem_cb 三级回调注册、
> syscall/flush/atexit 回调、scoreboard 计数框架、qemu_plugin_id_t 插件标识）、
> 插件内部结构（qemu_plugin_state 全局状态、qemu_plugin_ctx 插件上下文、
> qemu_plugin_insn/tb 翻译时数据结构、qemu_plugin_dyn_cb 动态回调）、
> 插件加载生命周期（plugin_load→g_module_open→qemu_plugin_install→版本校验→ID 生成、
> plugin_reset_destroy 卸载/重置）、
> 翻译时注入（plugin_gen_tb_start/insn_start/insn_end/tb_end→plugin_gen_inject→
> INDEX_op_plugin_cb/plugin_mem_cb 占位→inject_cb/inject_mem_cb 替换为 helper 调用）、
> GDB Stub 协议处理（gdb_handle_packet 主分发 c/s/g/G/m/M/Z/z 命令、
> gdb_read_byte 字节状态机、gdbserver_start 系统模式 chardev 绑定）、
> 断点子系统（CPUBreakpoint/CPUWatchpoint 结构、cpu_breakpoint_insert/remove、
> check_for_breakpoints_slow CF_BP_PAGE 页匹配、EXCP_DEBUG 触发）、
> 单步执行（CF_SINGLE_STEP→cpu_single_step→curr_cflags→EXCP_DEBUG）、
> ARM 寄存器映射（arm_cpu_gdb_read/write_register AArch32→AArch64 分派）、
> ARM 调试寄存器（DBGBVR/DBGBCR/DBGWVR/DBGWCR→hw_breakpoint_update/hw_watchpoint_update、
> arm_debug_check_breakpoint/watchpoint 架构匹配、arm_debug_excp_handler 异常路由）、
> Tracing 子系统（TraceEvent 结构、事件注册/迭代/启禁、simple 后端环形缓冲+写出线程）。

---

## 目录

1. [架构概述](#1-架构概述)
2. [TCG Plugin 公共 API](#2-tcg-plugin-公共-api)
3. [插件内部数据结构](#3-插件内部数据结构)
4. [插件加载与生命周期](#4-插件加载与生命周期)
5. [翻译时插件注入](#5-翻译时插件注入)
6. [插件回调执行路径](#6-插件回调执行路径)
7. [GDB Stub 协议处理](#7-gdb-stub-协议处理)
8. [GDB 系统模式与用户模式](#8-gdb-系统模式与用户模式)
9. [断点子系统](#9-断点子系统)
10. [单步执行机制](#10-单步执行机制)
11. [ARM 寄存器 GDB 映射](#11-arm-寄存器-gdb-映射)
12. [ARM 调试寄存器与硬件断点/观察点](#12-arm-调试寄存器与硬件断点观察点)
13. [ARM 调试异常处理](#13-arm-调试异常处理)
14. [Tracing 子系统](#14-tracing-子系统)
15. [完整调试流程总结](#15-完整调试流程总结)

---

## 1. 架构概述

QEMU 的调试与插桩子系统分为三个层次：

```
┌─────────────────────────────────────────────────┐
│              外部调试器 (GDB)                     │
│  GDB RSP 协议 ←→ gdbstub (c/s/g/G/m/M/Z/z)     │
├─────────────────────────────────────────────────┤
│              TCG Plugin 框架                      │
│  TB 翻译回调 → 指令/内存回调 → scoreboard 计数    │
├─────────────────────────────────────────────────┤
│              QEMU Tracing                        │
│  trace-events 静态定义 → 后端 (simple/ftrace/log) │
├─────────────────────────────────────────────────┤
│              ARM 架构调试                         │
│  DBGBVR/BCR 硬件断点 · DBGWVR/WCR 硬件观察点     │
│  MDSCR_EL1 调试控制 · arm_debug_excp_handler     │
└─────────────────────────────────────────────────┘
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `include/plugins/qemu-plugin.h` | 44-993 | Plugin 公共 API 完整定义 |
| `plugins/plugin.h` | 22-68 | 内部 state/ctx 结构 |
| `include/qemu/plugin.h` | 57-142 | 回调签名联合体、dyn_cb、insn/tb 结构 |
| `plugins/loader.c` | 179-268 | plugin_load 加载流程 |
| `plugins/core.c` | 26-776 | 运行时回调注册与执行 |
| `accel/tcg/plugin-gen.c` | 251-510 | 翻译时注入 IR |
| `gdbstub/gdbstub.c` | 1085-2485 | GDB 协议分发与状态机 |
| `gdbstub/system.c` | 122-411 | 系统模式 GDB 服务 |
| `include/exec/breakpoint.h` | 15-28 | CPUBreakpoint/Watchpoint 结构 |
| `cpu-common.c` | 392-458 | 断点 insert/remove |
| `accel/tcg/cpu-exec.c` | 293-364 | 断点运行时检查 |
| `accel/tcg/cpu-exec-common.c` | 42-57 | CF_SINGLE_STEP 设置 |
| `target/arm/gdbstub.c` | 42-116 | ARM 寄存器读写 |
| `target/arm/gdbstub64.c` | 35-100 | AArch64 寄存器映射 |
| `target/arm/debug_helper.c` | 174-524 | 调试寄存器定义与写入 |
| `target/arm/tcg/debug.c` | 376-751 | 调试匹配与硬件断/观察点 |
| `trace/event-internal.h` | 33-38 | TraceEvent 结构 |
| `trace/control.c` | 65-302 | 事件管理与后端初始化 |
| `trace/simple.c` | 36-425 | simple 后端实现 |

---

## 2. TCG Plugin 公共 API

### 插件入口

```c
// include/plugins/qemu-plugin.h:44
typedef uint64_t qemu_plugin_id_t;

// 135-137：每个插件必须导出此函数
QEMU_PLUGIN_EXPORT int qemu_plugin_install(qemu_plugin_id_t id,
                                           const qemu_info_t *info,
                                           int argc, char **argv);
```

### 核心回调注册 API

| 行号 | 函数 | 用途 |
|------|------|------|
| 227-228 | `qemu_plugin_uninstall()` | 异步卸载插件 |
| 241-242 | `qemu_plugin_reset()` | 异步重置回调 |
| 437-439 | `register_vcpu_tb_trans_cb()` | TB 翻译时回调（最核心） |
| 518-522 | `register_vcpu_insn_exec_cb()` | 每指令执行回调 |
| 775-780 | `register_vcpu_mem_cb()` | 内存访问回调 |
| 896-898 | `register_vcpu_syscall_cb()` | 系统调用前回调 |
| 912-915 | `register_vcpu_syscall_filter_cb()` | 系统调用过滤 |
| 928-930 | `register_vcpu_syscall_ret_cb()` | 系统调用返回回调 |
| 974-976 | `register_flush_cb()` | 代码缓存刷新回调 |
| 991-993 | `register_atexit_cb()` | 退出时回调 |

### 三级回调模型

```
1. TB 翻译回调 (tb_trans_cb)
   ↓ 插件在此注册后续回调
2. 指令/TB 执行回调 (insn_exec_cb, tb_exec_cb)
   + 条件回调 (insn_exec_cond_cb)
   + 内联操作 (inline_per_vcpu)
3. 内存访问回调 (mem_cb)
   + 内联内存操作 (mem_inline_per_vcpu)
```

---

## 3. 插件内部数据结构

### 全局状态

```c
// plugins/plugin.h:22-52
struct qemu_plugin_state {
    QTAILQ_HEAD(, qemu_plugin_ctx) ctxs;           // 23: 所有加载的插件
    QLIST_HEAD(, qemu_plugin_cb) cb_lists[EV_MAX];  // 24: 按事件类型的回调链
    GHashTable *id_ht;                              // 29: id→ctx 哈希表
    GHashTable *cpu_ht;                             // 34: cpu→index 映射
    QLIST_HEAD(, qemu_plugin_scoreboard) scoreboards; // 35: 计分板链表
    size_t scoreboard_alloc_size;                    // 36
    DECLARE_BITMAP(mask, EV_MAX);                   // 37: 活跃事件位图
    QemuRecMutex lock;                              // 44: 递归互斥锁
    struct qht dyn_cb_arr_ht;                       // 49: 动态回调哈希表
    int num_vcpus;                                  // 51
};
```

### 插件上下文

```c
// plugins/plugin.h:55-68
struct qemu_plugin_ctx {
    GModule *handle;                     // 56: 动态库句柄
    qemu_plugin_id_t id;                 // 57: 随机 64 位 ID
    struct qemu_plugin_cb *callbacks[EV_MAX]; // 58: 每事件类型回调
    QTAILQ_ENTRY(qemu_plugin_ctx) entry; // 59: 链表节点
    struct qemu_plugin_desc *desc;       // 64: 描述符（保持 argv 有效）
    bool installing, uninstalling, resetting; // 65-67: 生命周期状态
};
```

### 动态回调（运行时注入代码缓存）

```c
// include/qemu/plugin.h:106-113
struct qemu_plugin_dyn_cb {
    enum plugin_dyn_cb_type type;  // REGULAR/COND/MEM_REGULAR/INLINE_ADD/STORE
    union {
        struct qemu_plugin_regular_cb regular;     // 109: 普通回调
        struct qemu_plugin_conditional_cb cond;    // 110: 条件回调
        struct qemu_plugin_inline_cb inline_insn;  // 111: 内联操作
    };
};
```

### 翻译时结构

```c
// include/qemu/plugin.h:116-125
struct qemu_plugin_insn {
    uint64_t vaddr;          // 117: 指令虚拟地址
    GArray *insn_cbs;        // 118: 指令级回调数组
    GArray *mem_cbs;         // 119: 内存级回调数组
    uint8_t len;             // 120: 指令长度
    bool calls_helpers;      // 121: 是否调用 helper
    bool mem_helper;         // 124: helper 是否访问 guest 内存
};

// include/qemu/plugin.h:134-142
struct qemu_plugin_tb {
    GPtrArray *insns;        // 135: qemu_plugin_insn* 数组
    size_t n;                // 136: 指令数
    bool mem_helper;         // 139: TB 级内存 helper 标记
    GArray *cbs;             // 141: TB 级回调数组
};
```

---

## 4. 插件加载与生命周期

### plugin_load

```c
// plugins/loader.c:179-268
static int plugin_load(struct qemu_plugin_desc *desc, ...)
{
    // 1. g_module_open 加载 .so                          190
    // 2. g_module_symbol 查找 "qemu_plugin_install"      196
    // 3. g_module_symbol 查找 "qemu_plugin_version"      208
    //    版本校验 [MIN_VERSION, CURRENT_VERSION]          214-224
    // 4. xorshift64star 生成随机 ID                       234
    // 5. 调用 install(ctx->id, info, argc, argv)         246
    // 6. 失败则 plugin_reset_uninstall 清理              256
}
```

### 生命周期

```
qemu_plugin_load_list()        // loader.c:283 命令行 -plugin 解析
  → plugin_load() × N         // 每个插件
    → qemu_plugin_install()   // 用户实现的入口
      → register_*_cb()       // 注册各种回调

运行中:
  TB 翻译 → tb_trans_cb → 注册 insn/mem 回调
  TB 执行 → insn_exec_cb / mem_cb 触发

卸载:
  qemu_plugin_uninstall()      // 异步
  → plugin_reset_destroy()     // loader.c:321 锁定+清理回调
  → cb 通知完成
```

---

## 5. 翻译时插件注入

### TB 翻译钩子

```c
// accel/tcg/plugin-gen.c:414-442
bool plugin_gen_tb_start(CPUState *cpu, const DisasContextBase *db)
{
    // 检查 QEMU_PLUGIN_EV_VCPU_TB_TRANS 事件掩码    418-420
    // 重置/创建 plugin_tb                           427-438
    // 发射 PLUGIN_GEN_FROM_TB 占位 opcode            440
    return true;
}

// 444-475: plugin_gen_insn_start — 每指令开始
//   记录 vaddr、重置回调数组、发射 PLUGIN_GEN_FROM_INSN

// 477-485: plugin_gen_insn_end — 每指令结束
//   计算 insn->len、发射 PLUGIN_GEN_AFTER_INSN

// 493-510: plugin_gen_tb_end — TB 翻译结束
//   调用 qemu_plugin_tb_trans_cb → 收集插件注册的回调
//   调用 plugin_gen_inject → 替换占位为实际 helper 调用
```

### 注入流程

```c
// accel/tcg/plugin-gen.c:293-412
static void plugin_gen_inject(struct qemu_plugin_tb *plugin_tb)
{
    // 遍历 TCG ops 链表                               316
    QTAILQ_FOREACH_SAFE(op, &tcg_ctx->ops, link, next) {
        switch (op->opc) {
        case INDEX_op_plugin_cb:                    // 322
            // 根据 from 类型注入不同回调:
            // PLUGIN_GEN_FROM_TB → 注入 TB 级回调    349-357
            // PLUGIN_GEN_FROM_INSN → 启用 mem + 注入指令级回调  359-368
            // PLUGIN_GEN_AFTER_INSN/TB → 禁用 mem helper    336-348
            tcg_op_remove(tcg_ctx, op);             // 376: 删除占位

        case INDEX_op_plugin_mem_cb:                // 380
            // 注入内存访问回调                        396-400
            tcg_op_remove(tcg_ctx, op);             // 403
        }
    }
}
```

### inject_cb 回调注入

```c
// accel/tcg/plugin-gen.c:251-270
static void inject_cb(struct qemu_plugin_dyn_cb *cb) {
    switch (cb->type) {
    case PLUGIN_CB_REGULAR:    → gen_udata_cb()      // helper 调用
    case PLUGIN_CB_COND:       → gen_udata_cond_cb() // 条件 helper
    case PLUGIN_CB_INLINE_ADD: → gen_inline_add_u64() // 无 helper 内联加
    case PLUGIN_CB_INLINE_STORE: → gen_inline_store() // 无 helper 内联存
    }
}
```

---

## 6. 插件回调执行路径

### TB 翻译回调

```c
// plugins/core.c:503-517 (api.c 中调用)
void qemu_plugin_tb_trans_cb(CPUState *cpu, struct qemu_plugin_tb *tb)
{
    // 遍历所有注册了 VCPU_TB_TRANS 事件的插件
    // 调用每个插件的 vcpu_tb_trans 回调
    // 插件在此回调中调用 register_vcpu_insn_exec_cb / register_vcpu_mem_cb
}
```

### 内存访问回调

```c
// plugins/core.c:735-776 (通过 api.c)
void qemu_plugin_vcpu_mem_cb(unsigned int vcpu_index,
                              qemu_plugin_meminfo_t info, vaddr vaddr,
                              void *userdata)
{
    // 在 qemu_ld/st IR 执行时触发
    // 传递 meminfo（大小/符号/端序/读写）和虚拟地址
}
```

### 插件输出

```c
// plugins/api.c:390-393
// qemu_plugin_outs() → qemu_log_mask(CPU_LOG_PLUGIN, ...)
// 通过 QEMU 日志系统输出，非 trace 后端
```

---

## 7. GDB Stub 协议处理

### 主分发器

```c
// gdbstub/gdbstub.c:2062-2311
static int gdb_handle_packet(const char *line_buf)
{
    switch (line_buf[0]) {
    case '?': handle_target_halt       // 状态查询
    case 'c': handle_continue          // 1085-1093: 继续执行
    case 'C': handle_cont_with_sig     // 带信号继续
    case 's': handle_step              // 1366-1374: 单步
    case 'g': handle_read_all_regs     // 1346-1363: 读所有寄存器
    case 'G': handle_write_all_regs    // 1321-1344: 写所有寄存器
    case 'p': handle_get_reg           // 1241-1263: 读单个寄存器
    case 'P': handle_set_reg           // 1225-1239: 写单个寄存器
    case 'm': handle_read_mem          // 1292-1319: 读内存
    case 'M': handle_write_mem         // 1265-1290: 写内存
    case 'Z': handle_insert_bp         // 1166-1188: 插入断点/观察点
    case 'z': handle_remove_bp         // 1190-1212: 移除断点/观察点
    case 'q': handle_query             // 查询命令
    case 'v': handle_v_commands        // vCont 等扩展
    // ...更多命令
    }
}
```

### 字节状态机

```c
// gdbstub/gdbstub.c:2330-2485
void gdb_read_byte(uint8_t ch)
{
    // 状态机: RS_IDLE → '$' → RS_GETLINE → '#' → RS_CHKSUM1 → RS_CHKSUM2
    // 处理 ACK('+') / NACK('-') / 中断(0x03)
    // 完整包: $<data>#<checksum>
    // 校验通过 → gdb_handle_packet()
}
```

### handle_step — 单步核心

```c
// gdbstub/gdbstub.c:1366-1374
static void handle_step(GArray *params, void *user_ctx)
{
    if (params->len)
        gdb_set_cpu_pc(gdb_get_cmd_param(params, 0)->val_ull);
    cpu_single_step(gdbserver_state.c_cpu, gdbserver_state.sstep_flags);
    gdb_continue();  // 恢复执行，但只执行一条指令
}
```

---

## 8. GDB 系统模式与用户模式

### 系统模式

```c
// gdbstub/system.c:333-411
bool gdbserver_start(const char *device, Error **errp)
{
    // 1. 检查 CPU 存在性和调试支持              339-349
    // 2. 创建 chardev (TCP/stdio/socket)       358-381
    // 3. 初始化 gdbserver state                383-396
    // 4. 注册 vm_change_state_handler           386
    //    → gdb_vm_state_change 处理 VM 状态变化
    // 5. 绑定 chardev 接收处理                  400-405
    //    → gdb_chr_receive → gdb_read_byte
}

// gdbstub/system.c:122-217
static void gdb_vm_state_change(void *opaque, bool running, RunState state)
{
    // RUN_STATE_DEBUG → 检查 watchpoint_hit → 发送 T05 包
    // RUN_STATE_PAUSED → 发送 T02 (SIGINT)
    // 其他状态 → 对应信号号
    cpu_single_step(cpu, 0);  // 216: 停止时禁用单步
}
```

### 用户模式

```c
// gdbstub/user.c:202-270
int gdb_handlesig(CPUState *cpu, int sig)
{
    // 用户模式下 guest 收到信号时调用
    // 构造 T<signal> 包发送给 GDB
    // 循环处理 GDB 命令直到 continue/step
}

// gdbstub/user.c:473-520
bool gdbserver_start(const char *device, Error **errp)
{
    // 创建 TCP socket → accept → 开始通信
}
```

---

## 9. 断点子系统

### 数据结构

```c
// include/exec/breakpoint.h:15-28
typedef struct CPUBreakpoint {
    vaddr pc;              // 断点地址
    int flags;             // BP_GDB | BP_CPU | BP_... 标志
    QTAILQ_ENTRY(CPUBreakpoint) entry;
} CPUBreakpoint;

typedef struct CPUWatchpoint {
    vaddr vaddr;           // 观察点地址
    vaddr len;             // 观察范围长度
    vaddr hitaddr;         // 命中地址
    MemTxAttrs hitattrs;   // 命中属性
    int flags;             // BP_MEM_READ | BP_MEM_WRITE | BP_CPU
    QTAILQ_ENTRY(CPUWatchpoint) entry;
} CPUWatchpoint;
```

### 断点插入/删除

```c
// cpu-common.c:392-458
int cpu_breakpoint_insert(CPUState *cpu, vaddr pc, int flags, CPUBreakpoint **bp)
{
    // GDB 断点插在链表头（优先级更高）     407-408
    // CPU 断点插在链表尾                   410
}

int cpu_breakpoint_remove(CPUState *cpu, vaddr pc, int flags)
{
    // 遍历链表查找匹配 pc+flags            430-436
}

void cpu_breakpoint_remove_all(CPUState *cpu, int mask)
{
    // 删除所有匹配 mask 的断点             453-457
}
```

### 执行时断点检查

```c
// accel/tcg/cpu-exec.c:293-357
static bool check_for_breakpoints_slow(CPUState *cpu, vaddr pc, uint32_t *cflags)
{
    // 1. 单步优先于断点                     308-310
    // 2. 遍历断点链表:
    //    精确匹配 pc → 检查 BP_GDB 或 BP_CPU
    //      BP_GDB: 直接命中                 320-321
    //      BP_CPU: 调用 debug_check_breakpoint() 架构检查  326-329
    //    命中 → exception_index = EXCP_DEBUG              333
    // 3. 同页非精确匹配:
    //    设置 CF_BP_PAGE|CF_NO_GOTO_TB|1    354
    //    → 每指令单步，重新检查断点
}
```

---

## 10. 单步执行机制

### CF_SINGLE_STEP 设置

```c
// accel/tcg/cpu-exec-common.c:42-57
// curr_cflags() 在构建 TB 编译标志时:
if (unlikely(cpu->singlestep_enabled)) {
    cflags |= CF_NO_GOTO_TB    // 禁止 TB 链跳转
            | CF_NO_GOTO_PTR    // 禁止 goto_ptr
            | CF_SINGLE_STEP    // 单步标记
            | 1;                // TB 只包含 1 条指令
}
```

### 单步执行流程

```
GDB 's' 命令
  → handle_step()                        gdbstub.c:1366
  → cpu_single_step(cpu, sstep_flags)    设置 singlestep_enabled
  → gdb_continue()                       恢复执行

cpu_exec_loop():
  → curr_cflags() → CF_SINGLE_STEP|1
  → 编译/执行仅含 1 条指令的 TB
  → cpu_tb_exec() 返回后:
    if (cflags & CF_SINGLE_STEP)         cpu-exec.c:481-488
      → cpu->exception_index = EXCP_DEBUG
  → cpu_handle_exception()
    → EXCP_DEBUG → 返回到 cpu_exec 调用者
  → gdb_vm_state_change()               system.c
    → 发送 T05 包给 GDB
    → cpu_single_step(cpu, 0)           禁用单步
```

---

## 11. ARM 寄存器 GDB 映射

### AArch32 模式

```c
// target/arm/gdbstub.c:42-65
int arm_cpu_gdb_read_register(CPUState *cs, GByteArray *mem_buf, int n)
{
    if (arm_gdbstub_is_aarch64(cpu))
        return aarch64_cpu_gdb_read_register(cs, mem_buf, n);  // 48: 转发

    if (n < 16) return gdb_get_reg32(mem_buf, env->regs[n]);   // 51-53: R0-R15
    if (n == 25) {
        // M-profile: XPSR, 否则: CPSR
        return gdb_get_reg32(mem_buf, cpsr_read(env));          // 55-61
    }
}
```

### AArch64 模式

```c
// target/arm/gdbstub64.c:35-100
int aarch64_cpu_gdb_read_register(CPUState *cs, GByteArray *mem_buf, int n)
{
    if (n < 31) return gdb_get_reg64(mem_buf, env->xregs[n]);  // X0-X30
    if (n == 31) return gdb_get_reg64(mem_buf, env->xregs[31]); // SP
    if (n == 32) return gdb_get_reg64(mem_buf, env->pc);        // PC
    if (n == 33) return gdb_get_reg32(mem_buf, pstate_read(env)); // CPSR/PSTATE
}
```

### 寄存器扩展

```c
// target/arm/gdbstub.c:496-575
void arm_cpu_register_gdb_commands(...)
{
    // 注册 VFP/NEON 寄存器、SVE 寄存器、系统寄存器等扩展 XML
    // 支持 qXfer:features:read 协议扩展
}
```

---

## 12. ARM 调试寄存器与硬件断点/观察点

### 调试寄存器定义

```c
// target/arm/debug_helper.c:174-309
static const ARMCPRegInfo debug_cp_reginfo[] = {
    // MDSCR_EL1 — 调试系统控制寄存器                    193-199
    //   bit[15] MDE: 监控调试事件使能
    //   bit[13] KDE: 内核调试事件使能
    // DBGBVR<n> — 断点值寄存器 (通过 define_debug_regs 动态生成)
    // DBGBCR<n> — 断点控制寄存器
    // DBGWVR<n> — 观察点值寄存器
    // DBGWCR<n> — 观察点控制寄存器
    // OSLAR_EL1, OSLSR_EL1 — OS 锁                     256-267
    // DBGCLAIMSET/CLR_EL1 — Claim 标签                  297-308
};

// 402-524: define_debug_regs() — 根据 CPU 特性动态生成 BVR/BCR/WVR/WCR
```

### 硬件观察点更新

```c
// target/arm/tcg/debug.c:546-633
void hw_watchpoint_update(ARMCPU *cpu, int n)
{
    // 1. 删除旧的 QEMU 观察点                          555-558
    // 2. 检查 WCR.E 使能位                             560-563
    // 3. 解析 WCR.LSC → 读/写/读写标志                  565-578
    // 4. 计算地址和长度:
    //    MASK 模式 → 对齐区域最大 2GB                    593-601
    //    BAS 模式 → 字节选择位                          603-628
    // 5. cpu_watchpoint_insert() 安装 QEMU 观察点        631
}
```

### 硬件断点更新

```c
// target/arm/tcg/debug.c:652-738
void hw_breakpoint_update(ARMCPU *cpu, int n)
{
    // 解析 BCR 控制位:
    //   BT 类型（地址匹配 / 上下文匹配 / VMID 匹配）
    //   E 使能位
    //   PMC 特权级匹配
    // 调用 cpu_breakpoint_insert(cpu, addr, BP_CPU, ...)
}
```

### 写入钩子

```c
// target/arm/debug_helper.c:334-400
// dbgwvr_write → raw_write + hw_watchpoint_update        334-357
// dbgwcr_write → raw_write + hw_watchpoint_update        359-369
// dbgbvr_write → raw_write + hw_breakpoint_update        371-381
// dbgbcr_write → raw_write + hw_breakpoint_update        383-400
// 每次 guest 写调试寄存器，立即更新 QEMU 断/观察点
```

---

## 13. ARM 调试异常处理

### 架构断点检查

```c
// target/arm/tcg/debug.c:376-420
bool arm_debug_check_breakpoint(CPUState *cs)
{
    // 1. MDSCR_EL1.MDE (bit 15) 必须为 1               387-389
    // 2. arm_generate_debug_exceptions 检查当前 EL       388
    // 3. 单步状态优先于断点                              396-398
    // 4. PC 对齐检查                                    403-406
    // 5. 遍历所有断点寄存器 → bp_wp_matches() 匹配      414-418
}

// 422-431: arm_debug_check_watchpoint — 类似，调用 check_watchpoints()
```

### 调试异常路由

```c
// target/arm/tcg/debug.c:464-508
void arm_debug_excp_handler(CPUState *cs)
{
    if (wp_hit) {
        // 观察点命中:
        // 设置 exception.fsr = Debug fault                480
        // 设置 exception.vaddress = hitaddr               481
        // raise_exception(EXCP_DATA_ABORT, syn_watchpoint) 482-483
    } else {
        // 断点命中:
        // GDB 断点优先 → 返回给 GDB 处理                  494-496
        // CPU 断点 → raise_exception(EXCP_PREFETCH_ABORT, syn_breakpoint) 506
    }
}
```

### 异常处理链

```
TCG 执行 → check_for_breakpoints_slow()
  → BP_CPU 匹配 → debug_check_breakpoint() 回调
    → arm_debug_check_breakpoint()
      → bp_wp_matches() 逐寄存器匹配
  → EXCP_DEBUG
→ cpu_handle_exception()
  → do_interrupt() → arm_debug_excp_handler()
    → 架构异常 (PREFETCH_ABORT/DATA_ABORT)
    或
    → 返回 GDB (BP_GDB)
```

---

## 14. Tracing 子系统

### TraceEvent 结构

```c
// trace/event-internal.h:33-38
typedef struct TraceEvent {
    uint32_t id;           // 唯一 ID（运行时分配）
    const char *name;      // 事件名（如 "gdbstub_io_command"）
    const bool sstate;     // 静态编译状态
    uint16_t *dstate;      // 动态状态指针
} TraceEvent;
```

### 事件管理

```c
// trace/control.c:65-78
void trace_event_register_group(TraceEvent **events)
{
    // 为每个事件分配 ID，添加到 event_groups 数组
}

// 81-139: 事件查找与迭代
TraceEvent *trace_event_name(const char *name)  // 按名查找
void trace_event_iter_init_all/pattern/group()  // 迭代器
TraceEvent *trace_event_iter_next()             // 带 glob 过滤

// 141-199: 启用/禁用事件
// 264-302: 后端初始化与命令行解析
```

### simple 后端

```c
// trace/simple.c:36-425
// 环形缓冲区 trace_buf[4096*64]              52
// 写出线程 trace_available_cond              41-44
// 记录格式: event_id + timestamp + args      
// HEADER_MAGIC = 0xf2b177cb0aa429b4          25
// HEADER_VERSION = 4                          28
// FLUSH 阈值: TRACE_BUF_LEN / 4              49
```

### trace-events 定义

```
# 定义在各子系统的 trace-events 文件中:
# gdbstub/trace-events:
gdbstub_io_command(const char *command) "%s"
gdbstub_hit_break(void) ""
gdbstub_hit_watchpoint(const char *type, int cpu_index, uint64_t vaddr) ...
breakpoint_insert(int cpu_index, uint64_t pc, int flags) ...
breakpoint_remove(int cpu_index, uint64_t pc, int flags) ...
```

### 后端类型

| 后端 | 文件 | 特点 |
|------|------|------|
| simple | `trace/simple.c` | 环形缓冲+写出线程，二进制格式 |
| log | 内置 | 通过 qemu_log 输出到 stderr/file |
| ftrace | `trace/ftrace.c` | 写入 Linux ftrace trace_marker |
| dtrace | 编译时生成 | SystemTap/DTrace 探针 |

---

## 15. 完整调试流程总结

### GDB 连接与断点设置

```
GDB: target remote :1234
  → gdbserver_start("tcp::1234")                    system.c:333
  → chardev 绑定 gdb_chr_receive → gdb_read_byte    402-405

GDB: break *0x40080000
  → 'Z0,40080000,4' 包
  → handle_insert_bp()                              gdbstub.c:1166
  → gdb_breakpoint_insert()
  → cpu_breakpoint_insert(pc, BP_GDB)               cpu-common.c:392
```

### 断点命中

```
cpu_exec_loop():
  → check_for_breakpoints(pc)                        cpu-exec.c:362
  → 精确匹配 BP_GDB → EXCP_DEBUG                     333
  → cpu_handle_exception() → 返回 cpu_exec
  → VM 暂停 → gdb_vm_state_change(DEBUG)             system.c:122
  → 发送 T05 给 GDB
```

### 架构调试 (ARM 硬件断点)

```
Guest 写 DBGBVR0_EL1 = 0x40080000
  → dbgbvr_write()                                   debug_helper.c:371
  → hw_breakpoint_update()                            debug.c:652
  → cpu_breakpoint_insert(0x40080000, BP_CPU)

执行到 0x40080000:
  → check_for_breakpoints → BP_CPU
  → debug_check_breakpoint() → arm_debug_check_breakpoint()
  → MDSCR_EL1.MDE 检查 + bp_wp_matches()
  → EXCP_DEBUG → arm_debug_excp_handler()
  → raise_exception(EXCP_PREFETCH_ABORT, syn_breakpoint)
  → Guest 异常向量处理
```

### Plugin 插桩

```
-plugin libhotblocks.so:
  → qemu_plugin_install()
  → register_vcpu_tb_trans_cb(my_tb_trans)

TB 翻译:
  → plugin_gen_tb_start()                            plugin-gen.c:414
  → 翻译每条指令 → plugin_gen_insn_start/end()       444,477
  → plugin_gen_tb_end()                              493
    → qemu_plugin_tb_trans_cb() → my_tb_trans()
      → register_vcpu_insn_exec_cb(insn, my_insn_cb)
    → plugin_gen_inject() → 替换占位 IR 为 helper 调用

TB 执行:
  → helper 调用 → my_insn_cb(vcpu_index, userdata)
  → 插件更新计数器/分析数据
```

---

## 交叉参考

- [44-ARM64-TCG执行循环深度分析](44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md) — cpu_exec_loop、cpu_handle_exception、EXCP_DEBUG、CF_SINGLE_STEP
- [43-ARM64-TCG-softmmu-TLB深度分析](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — TLB 观察点检查、内存访问分发
- [42-ARM64-TCG前端后端代码生成深度分析](42-ARM64-TCG前端后端代码生成深度分析-IR中间表示-翻译循环-优化Pass-寄存器分配与AArch64代码发射.md) — TCG IR 系统、INDEX_op_plugin_cb/plugin_mem_cb

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
