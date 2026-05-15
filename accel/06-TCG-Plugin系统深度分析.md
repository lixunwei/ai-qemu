# TCG Plugin 系统深度分析

> QEMU 11.0.50 · 分析日期 2025-07 · 基于源码交叉验证

## 目录

1. [概述](#1-概述)
2. [Plugin API 版本与类型体系](#2-plugin-api-版本与类型体系)
3. [Plugin 加载流程 — plugin_load()](#3-plugin-加载流程--plugin_load)
4. [命令行接口 — -plugin 选项](#4-命令行接口---plugin-选项)
5. [Plugin 核心状态 — qemu_plugin_state](#5-plugin-核心状态--qemu_plugin_state)
6. [回调注册机制](#6-回调注册机制)
7. [TB 翻译集成](#7-tb-翻译集成)
8. [plugin_gen_tb_start/end — TB 级插桩](#8-plugin_gen_tb_startend--tb-级插桩)
9. [plugin_gen_insn_start/end — 指令级插桩](#9-plugin_gen_insn_startend--指令级插桩)
10. [plugin_gen_inject() — 代码注入](#10-plugin_gen_inject--代码注入)
11. [内存回调插桩](#11-内存回调插桩)
12. [Scoreboard 机制](#12-scoreboard-机制)
13. [内联操作 — Inline Ops](#13-内联操作--inline-ops)
14. [条件回调 — Conditional Callbacks](#14-条件回调--conditional-callbacks)
15. [内存回调分发 — qemu_plugin_vcpu_mem_cb()](#15-内存回调分发--qemu_plugin_vcpu_mem_cb)
16. [TB 翻译回调分发 — qemu_plugin_tb_trans_cb()](#16-tb-翻译回调分发--qemu_plugin_tb_trans_cb)
17. [vCPU 生命周期回调](#17-vcpu-生命周期回调)
18. [Syscall 回调](#18-syscall-回调)
19. [Plugin 与 cputlb 集成](#19-plugin-与-cputlb-集成)
20. [Plugin 卸载与清理](#20-plugin-卸载与清理)
21. [线程安全模型](#21-线程安全模型)
22. [构建系统](#22-构建系统)
23. [示例 Plugin 分析](#23-示例-plugin-分析)
24. [完整 Plugin 执行流程](#24-完整-plugin-执行流程)
25. [附录 A：公共 API 函数索引](#25-附录-a公共-api-函数索引)
26. [附录 B：关键源文件索引](#26-附录-b关键源文件索引)

---

## 1. 概述

TCG Plugin 系统为 QEMU 提供动态二进制分析能力——无需修改 QEMU 源码，通过共享库插件即可在翻译和执行阶段插入自定义回调，实现指令计数、缓存模拟、代码覆盖率、性能分析等功能。

**核心架构**：
```
┌─────────────────────────────────────────────────────┐
│                     Plugin (.so)                     │
│  qemu_plugin_install() → 注册回调                    │
└──────────┬──────────────────────────┬────────────────┘
           │ 翻译时回调               │ 执行时回调
           ▼                          ▼
┌─────────────────────┐    ┌─────────────────────────┐
│ TB Trans Callback   │    │ TB/Insn/Mem Exec CB     │
│ (plugin_gen_tb_end) │    │ (注入到生成代码中)       │
│                     │    │ - helper 回调            │
│ 遍历指令，注册      │    │ - inline 操作            │
│ 执行时回调          │    │ - 条件回调               │
└─────────────────────┘    └─────────────────────────┘
```

**关键数字**：
- **API 版本**：6（`QEMU_PLUGIN_VERSION`）
- **Plugin 源码**：`plugins/` 目录 ~4 个核心文件
- **代码生成集成**：`accel/tcg/plugin-gen.c` ~510 行
- **示例插件**：`contrib/plugins/` 14 个 + `tests/tcg/plugins/` 11 个

---

## 2. Plugin API 版本与类型体系

### API 版本

**定义**：`include/plugins/qemu-plugin.h:47-87`

```c
#define QEMU_PLUGIN_VERSION 6                          // :87

// 版本演进：
// v2: 移除 n_vcpus，用 per_vcpu 替代旧 inline API
// v3: 修改 qemu_plugin_insn_data 接口
// v4: 添加 qemu_plugin_read_memory_vaddr
// v5: 添加 read/write_memory_hwaddr, write_register, translate_vaddr
// v6: read/write_register 返回 bool, 添加 set_pc, discontinuity, syscall filter
```

### 核心类型

**定义**：`include/plugins/qemu-plugin.h:41-170`

```c
typedef uint64_t qemu_plugin_id_t;                     // :44

// 回调函数类型层次
typedef void (*qemu_plugin_simple_cb_t)(id);                    // :145
typedef void (*qemu_plugin_udata_cb_t)(id, userdata);           // :153
typedef void (*qemu_plugin_vcpu_simple_cb_t)(id, vcpu_index);   // :160
typedef void (*qemu_plugin_vcpu_udata_cb_t)(vcpu_index, userdata); // :169

// TB 翻译回调
typedef void (*qemu_plugin_vcpu_tb_trans_cb_t)(id, tb);         // :422

// 内存访问回调
typedef void (*qemu_plugin_vcpu_mem_cb_t)(vcpu_index, info, vaddr, userdata); // :750
```

### Plugin 安装入口

**定义**：`include/plugins/qemu-plugin.h:119-137`

```c
QEMU_PLUGIN_EXPORT int qemu_plugin_install(qemu_plugin_id_t id,
                                           const qemu_info_t *info,
                                           int argc, char **argv);  // :135-137
```

每个插件**必须**导出此符号。QEMU 加载插件时调用它，插件在此注册各种回调。

---

## 3. Plugin 加载流程 — plugin_load()

**定义**：`plugins/loader.c:179-260`

```
plugin_load(desc, info, errp)
│
├── 缓存行对齐分配 ctx                              :186-188
│   qemu_memalign(qemu_dcache_linesize, sizeof(*ctx))
│
├── g_module_open(desc->path, G_MODULE_BIND_LOCAL)   :190-193
│   → 打开共享库（Linux 上底层调用 dlopen）
│
├── 查找 qemu_plugin_install 符号                    :196-206
│   g_module_symbol(ctx->handle, "qemu_plugin_install", &sym)
│
├── 查找 qemu_plugin_version 符号                    :208-225
│   → 版本检查：min_version ≤ plugin_version ≤ QEMU_PLUGIN_VERSION
│
├── 分配唯一 plugin ID                               :227-243
│   qemu_rec_mutex_lock(&plugin.lock)
│   ctx->id = xorshift64star() 生成唯一随机 ID
│   g_hash_table_insert(plugin.id_ht, &ctx->id)
│   QTAILQ_INSERT_TAIL(&plugin.ctxs, ctx, entry)
│
├── 调用插件安装函数                                 :246
│   rc = install(ctx->id, info, desc->argc, desc->argv)
│
└── 错误处理                                         :248-258
    安装失败 → plugin_reset_uninstall() 清理
```

---

## 4. 命令行接口 — -plugin 选项

### 解析入口

- **系统模式**：`system/vl.c` — `case QEMU_OPTION_plugin: qemu_plugin_opt_parse(optarg, &plugin_list)`
- **用户模式**：`linux-user/main.c` — `handle_arg_plugin()` → `qemu_plugin_opt_parse(arg, &plugins)`

### 选项解析

**定义**：`plugins/loader.c:91-158`

```
plugin_add(desc_head, opt)                             :91-143
│
├── 解析 "file=path,arg=val1,arg=val2" 格式
├── desc->path = file 路径
├── desc->argv[] = 参数数组
└── QTAILQ_INSERT_TAIL(desc_head, desc)

qemu_plugin_opt_parse(optarg, head)                    :145-158
│
├── 支持简写："path" 等价于 "file=path"
└── 调用 plugin_add() 构建描述符
```

### 多插件加载

**定义**：`plugins/loader.c:292-312`

```c
int qemu_plugin_load_list(QemuPluginList *head, Error **errp)
{
    QTAILQ_FOREACH(desc, head, entry) {
        rc = plugin_load(desc, &info, errp);     // 逐个加载
    }
}
```

命令行示例：
```bash
qemu-system-aarch64 \
  -plugin contrib/plugins/hotblocks.so \
  -plugin contrib/plugins/execlog.so,arg=noexec
```

---

## 5. Plugin 核心状态 — qemu_plugin_state

### 内部数据结构

**定义**：`plugins/plugin.h:22-52`

```c
struct qemu_plugin_state {
    QTAILQ_HEAD(, qemu_plugin_ctx) ctxs;        // :23 — 已加载插件链
    QLIST_HEAD(, qemu_plugin_cb) cb_lists[...];  // :24 — 各事件回调链
    QLIST_HEAD(, qemu_plugin_scoreboard) scoreboards; // :35 — 计分板链
    GHashTable *id_ht;                           // ID → ctx 哈希表
    QemuRecMutex lock;                           // 全局插件锁
    unsigned long *mask;                         // 事件掩码位图
    unsigned int scoreboard_alloc_size;          // 计分板容量
};
```

### Plugin 上下文

**定义**：`plugins/plugin.h:55-68`

```c
struct qemu_plugin_ctx {
    GModule *handle;                     // dlopen 句柄
    qemu_plugin_id_t id;                 // 唯一 ID
    struct qemu_plugin_desc *desc;       // 描述符（路径、参数）
    bool installing;                     // 安装中标志
    bool uninstalling;                   // 卸载中标志
    QTAILQ_ENTRY(qemu_plugin_ctx) entry; // 链表节点
};
```

### 回调结构

**定义**：`plugins/core.c:26-31`

```c
struct qemu_plugin_cb {
    struct qemu_plugin_ctx *ctx;         // :27 — 所属插件
    union qemu_plugin_cb_sig f;          // :28 — 回调函数指针
    void *udata;                         // :29 — 用户数据
    QLIST_ENTRY(qemu_plugin_cb) entry;   // :30 — 链表节点
};
```

---

## 6. 回调注册机制

### 通用注册函数

**定义**：`plugins/core.c:180-214`

```
do_plugin_register_cb(id, ev, f, udata)
│
├── QEMU_LOCK_GUARD(&plugin.lock)                    :186
│
├── plugin_id_to_ctx_locked(id) → ctx                :188
│
├── 创建 qemu_plugin_cb                              :190-195
│   cb->ctx = ctx
│   cb->f = f
│   cb->udata = udata
│
├── QLIST_INSERT_HEAD_RCU(&plugin.cb_lists[ev], cb)  :197
│
└── 设置全局事件掩码位                                :200-214
    set_bit(ev, plugin.mask)
    → 异步通知所有 CPU 更新 event_mask
```

### 注册 API 层次

**定义**：`plugins/api.c:71-217`

| API 函数 | 位置 | 事件类型 |
|----------|------|---------|
| `register_vcpu_init_cb()` | `api.c:71-75` | VCPU_INIT |
| `register_vcpu_exit_cb()` | `api.c:77-81` | VCPU_EXIT |
| `register_vcpu_tb_exec_cb()` | `api.c:88-97` | TB 执行 |
| `register_vcpu_tb_exec_cond_cb()` | `api.c:99-115` | TB 条件执行 |
| `register_vcpu_tb_exec_inline_per_vcpu()` | `api.c:117-126` | TB 内联操作 |
| `register_vcpu_insn_exec_cb()` | `api.c:128-137` | 指令执行 |
| `register_vcpu_insn_exec_cond_cb()` | `api.c:139-155` | 指令条件执行 |
| `register_vcpu_insn_exec_inline_per_vcpu()` | `api.c:157-167` | 指令内联操作 |
| `register_vcpu_mem_cb()` | `api.c:174-182` | 内存访问 |
| `register_vcpu_mem_inline_per_vcpu()` | `api.c:183-191` | 内存内联操作 |
| `register_vcpu_tb_trans_cb()` | `api.c:193-198` | TB 翻译 |
| `register_vcpu_syscall_cb()` | `api.c:199-210` | 系统调用 |

---

## 7. TB 翻译集成

### 翻译循环中的 Hook 点

**定义**：`accel/tcg/translator.c:152-231`

```
translator_loop()
│
├── plugin_gen_tb_start(cpu, db)                     :152-157
│   → 初始化 plugin_tb 结构
│   → 发射 PLUGIN_GEN_FROM_TB 标记
│
├── for each instruction:
│   ├── plugin_gen_insn_start(cpu, db)               :168-175
│   │   → 初始化 plugin_insn 结构
│   │   → 发射 PLUGIN_GEN_FROM_INSN 标记
│   │
│   ├── [翻译指令 → 生成 TCG IR]
│   │
│   └── plugin_gen_insn_end()                        :183-191
│       → 记录指令长度
│       → 发射 PLUGIN_GEN_AFTER_INSN 标记
│
└── plugin_gen_tb_end(cpu, num_insns)                :229-231
    → 调用 qemu_plugin_tb_trans_cb() 通知插件
    → 调用 plugin_gen_inject() 注入插桩代码
```

---

## 8. plugin_gen_tb_start/end — TB 级插桩

### plugin_gen_tb_start()

**定义**：`accel/tcg/plugin-gen.c:414-442`

```c
bool plugin_gen_tb_start(CPUState *cpu, const DisasContextBase *db)
{
    // 检查是否有 TB 翻译回调注册                      :418-421
    if (!test_bit(QEMU_PLUGIN_EV_VCPU_TB_TRANS,
                  cpu->plugin_state->event_mask)) {
        return false;   // 无插件关注 → 跳过
    }

    ptb = tcg_ctx->plugin_tb;
    if (ptb) {
        // 复用已有结构，重置回调数组                   :429-433
        g_array_set_size(ptb->cbs, 0);
        ptb->n = 0;
    } else {
        ptb = g_new0(struct qemu_plugin_tb, 1);        // :435
        tcg_ctx->plugin_tb = ptb;
    }

    tcg_gen_plugin_cb(PLUGIN_GEN_FROM_TB);             // :440
    return true;
}
```

### plugin_gen_tb_end()

**定义**：`accel/tcg/plugin-gen.c:493-510`

```c
void plugin_gen_tb_end(CPUState *cpu, size_t num_insns)
{
    ptb->n = num_insns;                                // :499

    // 通知所有已注册的翻译回调                        :502
    qemu_plugin_tb_trans_cb(cpu, ptb);

    // 将回调注入到生成的 TCG IR 中                    :505
    plugin_gen_inject(ptb);

    tcg_ctx->plugin_db = NULL;                         // :508
    tcg_ctx->plugin_insn = NULL;                       // :509
}
```

---

## 9. plugin_gen_insn_start/end — 指令级插桩

### plugin_gen_insn_start()

**定义**：`accel/tcg/plugin-gen.c:444-475`

```
plugin_gen_insn_start(cpu, db)
│
├── 获取或创建 insn 结构                              :453-459
│   ptb->n = n (当前指令序号)
│   insn = g_ptr_array_index(ptb->insns, n-1)
│   或 insn = g_new0(struct qemu_plugin_insn, 1)
│
├── 初始化指令状态                                    :461-469
│   insn->calls_helpers = false
│   insn->mem_helper = false
│   g_array_set_size(insn->insn_cbs, 0)
│   g_array_set_size(insn->mem_cbs, 0)
│
├── 记录指令地址                                      :471-472
│   insn->vaddr = db->pc_next
│
└── tcg_gen_plugin_cb(PLUGIN_GEN_FROM_INSN)           :474
```

### plugin_gen_insn_end()

**定义**：`accel/tcg/plugin-gen.c:477-485`

```c
void plugin_gen_insn_end(void)
{
    pinsn->len = db->fake_insn ? db->record_len       // :482
                               : db->pc_next - pinsn->vaddr;
    tcg_gen_plugin_cb(PLUGIN_GEN_AFTER_INSN);          // :484
}
```

---

## 10. plugin_gen_inject() — 代码注入

**定义**：`accel/tcg/plugin-gen.c:293-412`

```
plugin_gen_inject(ptb)
│
├── 遍历 TCG IR 中的 PLUGIN_GEN 标记                  :300-410
│
├── PLUGIN_GEN_FROM_TB:                               :310-330
│   对 ptb->cbs 中的每个回调：
│   ├── PLUGIN_CB_REGULAR → inject_cb()
│   ├── PLUGIN_CB_INLINE_ADD/STORE → inject_inline()
│   └── PLUGIN_CB_COND → inject_cond_cb()
│
├── PLUGIN_GEN_FROM_INSN:                             :335-370
│   对 insn->insn_cbs 中的每个回调：
│   ├── 同上三种类型
│   └── gen_enable_mem_helper() — 如需内存回调
│
├── PLUGIN_GEN_AFTER_INSN:                            :375-395
│   gen_disable_mem_helper() — 清除内存回调指针
│
└── PLUGIN_GEN_AFTER_TB:                              :400-410
    最终清理
```

### inject_cb() — Helper 回调注入

**定义**：`accel/tcg/plugin-gen.c:251-270`

```
inject_cb(cb)
│
├── gen_udata_cb()                                    :117-134
│   → 生成 TCG 调用指令：
│   tcg_gen_call2(helper_plugin_vcpu_udata_cb,
│                 cpu_index, cb_func, userdata)
│
└── 或 gen_udata_cond_cb()                            :174-204
    → 加载 scoreboard 值
    → tcg_gen_brcondi_i64(cond, val, imm, skip_label)
    → 条件不满足 → 跳过回调
    → 条件满足 → 调用回调
```

---

## 11. 内存回调插桩

### gen_enable_mem_helper()

**定义**：`accel/tcg/plugin-gen.c:47-92`

```
gen_enable_mem_helper(ptb, insn)
│
├── 检查指令是否调用 helper                           :66-68
│   if (!insn->calls_helpers) return;
│
├── 检查是否有内存回调                                :70-73
│   if (!insn->mem_cbs || !insn->mem_cbs->len) return;
│
├── 复制回调描述符数组                                :84-88
│   arr = g_array_sized_new(...)
│   g_array_append_vals(arr, insn->mem_cbs->data, len)
│
└── 将数组指针存入 CPUState.neg.plugin_mem_cbs        :90-91
    tcg_gen_st_ptr(tcg_constant_ptr(arr), tcg_env,
        offsetof(CPUState, neg.plugin_mem_cbs) - sizeof(CPUState))
```

### gen_disable_mem_helper()

**定义**：`accel/tcg/plugin-gen.c:94-98`

```c
static void gen_disable_mem_helper(void)
{
    tcg_gen_st_ptr(tcg_constant_ptr(0), tcg_env,       // :96-97
        offsetof(CPUState, neg.plugin_mem_cbs) - sizeof(CPUState));
}
```

### 内存值捕获

**定义**：`accel/tcg/plugin-gen.c:193-238`

`plugin_gen_mem_callbacks_i32/i64/i128()` 在内存访问前将值存储到：
- `CPUState.neg.plugin_mem_value_low`（低 64 位）
- `CPUState.neg.plugin_mem_value_high`（高 64 位，128 位访问时）

---

## 12. Scoreboard 机制

### 概念

Scoreboard 是 Plugin 系统的**per-vCPU 数据存储**，为每个 vCPU 提供独立的计数器/状态槽位。

### API

**定义**：`include/plugins/qemu-plugin.h:1283-1352`

```c
// 创建/销毁
qemu_plugin_scoreboard_new(elem_size)                  // :1283
qemu_plugin_scoreboard_free(scoreboard)                // :1295

// 查找 per-vCPU 数据
qemu_plugin_scoreboard_find(scoreboard, vcpu_index)    // :1302

// 便捷宏 — u64 类型
qemu_plugin_scoreboard_u64(scoreboard)                 // :1313
qemu_plugin_scoreboard_u64_in_struct(scoreboard, type, field) // :1317
```

### 内部实现

**定义**：`plugins/api.c:636-684`

```c
// 创建                                                :636
struct qemu_plugin_scoreboard *qemu_plugin_scoreboard_new(size_t elem_size)
// → 分配 GArray，大小 = scoreboard_alloc_size * elem_size

// 查找                                                :646-656
void *qemu_plugin_scoreboard_find(scoreboard, vcpu_index)
// → 返回 base + vcpu_index * elem_size

// u64 操作                                            :662-684
qemu_plugin_u64_add(entry, vcpu_index, added)
qemu_plugin_u64_get(entry, vcpu_index)
qemu_plugin_u64_set(entry, vcpu_index, val)
qemu_plugin_u64_sum(entry)  // 所有 vCPU 求和
```

### 内存布局

```
Scoreboard (GArray):
┌──────────────┬──────────────┬──────────────┬───┐
│  vCPU 0 slot │  vCPU 1 slot │  vCPU 2 slot │...│
│  (elem_size) │  (elem_size) │  (elem_size) │   │
└──────────────┴──────────────┴──────────────┴───┘
       ↑                ↑
  find(sb, 0)      find(sb, 1)
```

**注意**：Scoreboard 不是 TLS，而是通过 `vcpu_index` 索引的共享数组。vCPU 只访问自己的槽位，无需加锁。

---

## 13. 内联操作 — Inline Ops

### 操作类型

**定义**：`include/plugins/qemu-plugin.h:488-491`

```c
enum qemu_plugin_op {
    QEMU_PLUGIN_INLINE_ADD_U64,                        // :489
    QEMU_PLUGIN_INLINE_STORE_U64,                      // :490
};
```

### 工作原理

内联操作**直接在生成代码中**修改 Scoreboard 值，**不调用 helper 函数**：

**代码生成**：`accel/tcg/plugin-gen.c:206-227`

```
gen_inline_add_u64_cb(cb)                              :206-217
│
├── 计算 scoreboard 槽位地址
│   ptr = scoreboard_base + vcpu_index * elem_size + offset
│
├── tcg_gen_ld_i64(tmp, ptr)
├── tcg_gen_addi_i64(tmp, tmp, imm)
└── tcg_gen_st_i64(tmp, ptr)
    → 3 条 TCG 操作 → ~3 条宿主指令

gen_inline_store_u64_cb(cb)                            :219-227
│
├── tcg_gen_st_i64(tcg_constant_i64(imm), ptr)
└── → 1 条 TCG 操作 → ~1 条宿主指令
```

### 内联 vs Helper 对比

| 特性 | Helper 回调 | 内联操作 |
|------|------------|---------|
| 开销 | 函数调用 (~20 周期) | 3 条指令 (~3 周期) |
| 灵活性 | 完全自定义逻辑 | 仅 ADD/STORE |
| 数据 | 任意 userdata | Scoreboard u64 |
| 适用场景 | 复杂分析 | 简单计数 |

---

## 14. 条件回调 — Conditional Callbacks

**定义**：`include/plugins/qemu-plugin.h:456-480`

条件回调在执行时**先检查 Scoreboard 值**，满足条件才调用 helper：

### 代码生成

**定义**：`accel/tcg/plugin-gen.c:174-204`

```
gen_udata_cond_cb(cb)
│
├── 加载 scoreboard 值                                :180-185
│   addr = scoreboard_base + vcpu_index * elem_size + offset
│   tcg_gen_ld_i64(val, addr)
│
├── 条件分支                                          :190-195
│   tcg_gen_brcondi_i64(cond, val, imm, skip_label)
│   → 条件不满足 → 跳过回调
│
├── 调用 helper 回调                                  :197-200
│   gen_udata_cb(cb)
│
└── skip_label:                                       :202
    gen_set_label(skip_label)
```

### 条件类型

```c
enum qemu_plugin_cond {
    QEMU_PLUGIN_COND_NEVER,     // 永不执行
    QEMU_PLUGIN_COND_ALWAYS,    // 总是执行（等价于普通回调）
    QEMU_PLUGIN_COND_EQ,        // ==
    QEMU_PLUGIN_COND_NE,        // !=
    QEMU_PLUGIN_COND_LT,        // <
    QEMU_PLUGIN_COND_LE,        // <=
    QEMU_PLUGIN_COND_GT,        // >
    QEMU_PLUGIN_COND_GE,        // >=
};
```

**典型用途**：每 N 次执行触发一次回调（先用 inline ADD 计数，条件回调检查计数器）。

---

## 15. 内存回调分发 — qemu_plugin_vcpu_mem_cb()

**定义**：`plugins/core.c:735-776`

```c
void qemu_plugin_vcpu_mem_cb(CPUState *cpu, uint64_t vaddr,
                             uint64_t value_low, uint64_t value_high,
                             MemOpIdx oi, enum qemu_plugin_mem_rw rw)
{
    GArray *arr = cpu->neg.plugin_mem_cbs;             // :740
    if (arr == NULL) return;                           // :743-744

    // 存储访问值供插件查询                            :747-748
    cpu->neg.plugin_mem_value_low = value_low;
    cpu->neg.plugin_mem_value_high = value_high;

    for (i = 0; i < arr->len; i++) {                   // :750
        cb = &g_array_index(arr, struct qemu_plugin_dyn_cb, i);

        switch (cb->type) {
        case PLUGIN_CB_MEM_REGULAR:                    // :755
            if (rw & cb->regular.rw) {                 // :756
                cb->regular.f.vcpu_mem(cpu->cpu_index, // :760
                    make_plugin_meminfo(oi, rw),
                    vaddr, cb->regular.userp);
            }
            break;

        case PLUGIN_CB_INLINE_ADD_U64:                 // :766
        case PLUGIN_CB_INLINE_STORE_U64:               // :767
            if (rw & cb->inline_insn.rw) {             // :768
                exec_inline_op(cb->type,               // :769
                    &cb->inline_insn, cpu->cpu_index);
            }
            break;
        }
    }
}
```

---

## 16. TB 翻译回调分发 — qemu_plugin_tb_trans_cb()

**定义**：`plugins/core.c:503-517`

```c
void qemu_plugin_tb_trans_cb(CPUState *cpu, struct qemu_plugin_tb *tb)
{
    QLIST_FOREACH_SAFE_RCU(cb, &plugin.cb_lists[ev], entry, next) {
        func = cb->f.vcpu_tb_trans;                    // :511

        qemu_plugin_set_cb_flags(cpu, QEMU_PLUGIN_CB_RW_REGS); // :513
        func(cb->ctx->id, tb);                         // :514
        qemu_plugin_set_cb_flags(cpu, QEMU_PLUGIN_CB_NO_REGS); // :515
    }
}
```

**翻译回调中**，插件通常：
1. 遍历 TB 中的指令（`qemu_plugin_tb_get_insn(tb, i)`）
2. 对感兴趣的指令注册执行回调或内联操作

---

## 17. vCPU 生命周期回调

### vCPU 初始化

**定义**：`plugins/core.c:282-305`

```
qemu_plugin_vcpu_init__async(cpu, data)                :282-299
│
├── 扩展所有 scoreboard（如需要）                      :265-279
│   start_exclusive() → 全局暂停
│   qemu_plugin_scoreboard_expand(scoreboard, max)
│   end_exclusive()
│
├── 遍历所有 VCPU_INIT 回调                           :290-295
│   QLIST_FOREACH_SAFE_RCU(cb, &plugin.cb_lists[VCPU_INIT])
│   cb->f.vcpu_simple(cb->ctx->id, cpu->cpu_index)
│
└── qemu_plugin_vcpu_init_hook(cpu)                    :301-305
    → async_run_on_cpu() 投递到目标 CPU
```

### vCPU 退出

**定义**：`plugins/core.c:307-320`

```
qemu_plugin_vcpu_exit_hook(cpu)
│
└── 遍历 VCPU_EXIT 回调
    cb->f.vcpu_simple(cb->ctx->id, cpu->cpu_index)
```

### vCPU 空闲/恢复

- `system/cpus.c` 中调用 `qemu_plugin_vcpu_idle_cb(cpu)` 和 `qemu_plugin_vcpu_resume_cb(cpu)`

---

## 18. Syscall 回调

### 分发实现

**定义**：`plugins/core.c:519-625`

```
qemu_plugin_vcpu_syscall()                             :543-563
│ → 通知插件系统调用发生（调用号 + 参数）

qemu_plugin_vcpu_syscall_ret()                         :571-587
│ → 通知插件系统调用返回（返回值）

qemu_plugin_vcpu_syscall_filter()                      :595-625
│ → v6 新增：允许插件拦截/跳过系统调用
│   返回 true → 跳过原始系统调用
```

### 触发点

- **Linux 用户模式**：`linux-user/syscall.c` 在 syscall 前后调用
- **系统模式**：不直接触发（Guest OS 内的 syscall 对 QEMU 不可见）

---

## 19. Plugin 与 cputlb 集成

### 强制 MMIO 路径

**定义**：`accel/tcg/cputlb.c:1360-1411`

当指令有内存回调时，cputlb 强制走 MMIO 路径以触发回调：

```c
force_mmio = check_mem_cbs && cpu_plugin_mem_cbs_enabled(cpu);
```

这意味着：
1. 指令注册了内存回调
2. 即使是普通 RAM 访问，也会走慢路径
3. 慢路径中调用 `qemu_plugin_vcpu_mem_cb()`

### 回调调用点

- **加载/存储 helper**：`accel/tcg/ldst_common.c.inc:126-205`
- **原子操作 helper**：`accel/tcg/atomic_common.c.inc:16-30`

---

## 20. Plugin 卸载与清理

### 卸载流程

**定义**：`plugins/loader.c:322-419`

```
plugin_reset_uninstall(id, cb, reset)
│
├── plugin_reset_destroy__locked()                     :322-368
│   ├── 移除所有回调                                   :330-340
│   ├── 刷新 TB 缓存（TB 中可能嵌入了回调）          :345-350
│   │   tb_flush(cpu)
│   ├── 调用用户注册的 reset/uninstall 回调           :355-360
│   └── [卸载] g_module_close(ctx->handle)            :365
│       → dlclose 释放共享库
│
└── plugin_reset_uninstall()                           :385-419
    qemu_rec_mutex_lock(&plugin.lock)
    plugin_reset_destroy__locked()
    qemu_rec_mutex_unlock(&plugin.lock)
```

### atexit 清理

**定义**：`plugins/core.c:778-832`

```c
void qemu_plugin_atexit_cb(void)                       // :778-781
{
    plugin_cb__udata(QEMU_PLUGIN_EV_ATEXIT);
}

// 注册：
static void __attribute__((__constructor__)) plugin_init(void)  // :858-874
{
    atexit(qemu_plugin_atexit_cb);
}
```

---

## 21. 线程安全模型

### 锁层次

| 锁 | 保护范围 | 位置 |
|-----|---------|------|
| `plugin.lock` | 插件注册/卸载、回调链修改 | `plugins/plugin.h:39-44` |
| RCU | 回调链遍历（读路径无锁） | `core.c` 各 `QLIST_FOREACH_SAFE_RCU` |
| `start_exclusive()` | Scoreboard 扩容、全局 TB flush | `core.c:265-279` |

### 关键安全规则

1. **回调注册**（写路径）：持有 `plugin.lock`
2. **回调分发**（读路径）：RCU 保护，无锁遍历
3. **Scoreboard 扩容**：需要 `start_exclusive()` 全局暂停
4. **TB Flush**：卸载插件时刷新所有 TB（因为生成代码中嵌入了回调）
5. **per-vCPU 事件掩码**：通过 `async_run_on_cpu()` 异步更新（`core.c:49-62`）

---

## 22. 构建系统

**定义**：`plugins/meson.build`

```meson
# 导出符号列表                                        :5-9
plugin_ldflags = []

# Darwin: 使用 -exported_symbols_list                  :13-25
# Windows: 生成 .def + delay-load lib                  :28-68
# Linux: 使用 --dynamic-list                           

# Plugin 依赖对象                                      :70-79
# 编译核心文件                                         :87-88
common_ss.add(files('loader.c'))
```

Plugin 共享库格式：
- Linux: `.so`（`g_module_open` → `dlopen`）
- macOS: `.dylib`
- Windows: `.dll`

---

## 23. 示例 Plugin 分析

### contrib/plugins/ — 14 个官方示例

| 插件 | 功能 |
|------|------|
| `hotblocks.c` | 热点 TB 统计（支持 inline 计数） |
| `hotpages.c` | 热点内存页统计 |
| `execlog.c` | 执行日志记录 |
| `cache.c` | 缓存模拟（L1/L2） |
| `howvec.c` | 指令分类统计 |
| `hwprofile.c` | MMIO 设备访问分析 |
| `cflow.c` | 控制流分析 |
| `drcov.c` | 代码覆盖率（DynamoRIO 格式） |
| `ips.c` | 指令/秒性能计数 |
| `bbv.c` | 基本块向量（SimPoint 格式） |
| `lockstep.c` | 双实例锁步比较 |
| `stoptrigger.c` | 条件停止触发器 |
| `traps.c` | 异常/中断追踪 |
| `uftrace.c` | 函数追踪（uftrace 格式） |

### tests/tcg/plugins/ — 11 个测试插件

`bb`, `discons`, `empty`, `inline`, `insn`, `mem`, `patch`, `registers`, `reset`, `setpc`, `syscall`

### hotblocks 示例分析

**定义**：`contrib/plugins/hotblocks.c:145-159`

```c
// 翻译回调中注册 TB 执行计数
static void vcpu_tb_trans(qemu_plugin_id_t id, struct qemu_plugin_tb *tb)
{
    // 方式 1: inline 操作（高性能）
    qemu_plugin_register_vcpu_tb_exec_inline_per_vcpu(
        tb, QEMU_PLUGIN_INLINE_ADD_U64, count, 1);

    // 方式 2: helper 回调（灵活）
    qemu_plugin_register_vcpu_tb_exec_cb(
        tb, vcpu_tb_exec, QEMU_PLUGIN_CB_NO_REGS, NULL);
}
```

---

## 24. 完整 Plugin 执行流程

```
1. 启动阶段
   ─────────
   qemu-system-aarch64 -plugin hotblocks.so
   │
   ├── qemu_plugin_opt_parse("hotblocks.so")        → 构建描述符
   ├── qemu_plugin_load_list()                       → 遍历插件列表
   └── plugin_load()
       ├── g_module_open("hotblocks.so")             → dlopen
       ├── 查找 qemu_plugin_install 符号
       ├── 版本检查 (≤ QEMU_PLUGIN_VERSION 6)
       └── qemu_plugin_install(id, info, argc, argv)
           └── register_vcpu_tb_trans_cb(id, vcpu_tb_trans)

2. 翻译阶段（每个新 TB）
   ───────────────────────
   translator_loop()
   │
   ├── plugin_gen_tb_start(cpu, db)                  → 初始化 plugin_tb
   │
   ├── for each insn:
   │   ├── plugin_gen_insn_start()                   → 初始化 plugin_insn
   │   ├── [翻译ARM64指令 → TCG IR]
   │   └── plugin_gen_insn_end()                     → 记录长度
   │
   └── plugin_gen_tb_end(cpu, num_insns)
       ├── qemu_plugin_tb_trans_cb(cpu, ptb)
       │   └── vcpu_tb_trans(id, tb)                 ← 插件的翻译回调
       │       ├── 遍历 TB 中的指令
       │       └── register_vcpu_tb_exec_inline_per_vcpu(
       │               tb, ADD_U64, count, 1)
       │
       └── plugin_gen_inject(ptb)
           └── 将 inline ADD 注入到生成代码中
               → tcg_gen_ld_i64 + addi + st (3条)

3. 执行阶段（每次 TB 执行）
   ───────────────────────
   生成的宿主代码执行：
   │
   ├── [正常 TB 代码]
   │
   ├── [内联操作] ← 3 条宿主指令
   │   LDR X0, [scoreboard + vcpu_offset]
   │   ADD X0, X0, #1
   │   STR X0, [scoreboard + vcpu_offset]
   │
   └── [继续执行或退出 TB]

4. 退出阶段
   ─────────
   atexit → qemu_plugin_atexit_cb()
   └── 插件的 atexit 回调
       └── 输出统计结果
```

---

## 25. 附录 A：公共 API 函数索引

| 函数 | 位置 | 用途 |
|------|------|------|
| `qemu_plugin_install()` | `qemu-plugin.h:135` | 插件入口（必须导出） |
| `qemu_plugin_uninstall()` | `qemu-plugin.h:217` | 卸载插件 |
| `qemu_plugin_reset()` | `qemu-plugin.h:242` | 重置插件状态 |
| `register_vcpu_init_cb()` | `qemu-plugin.h:271` | vCPU 初始化回调 |
| `register_vcpu_exit_cb()` | `qemu-plugin.h:285` | vCPU 退出回调 |
| `register_vcpu_tb_trans_cb()` | `qemu-plugin.h:437` | TB 翻译回调 |
| `register_vcpu_tb_exec_cb()` | `qemu-plugin.h:450` | TB 执行回调 |
| `register_vcpu_tb_exec_cond_cb()` | `qemu-plugin.h:473` | TB 条件执行回调 |
| `register_vcpu_tb_exec_inline_per_vcpu()` | `qemu-plugin.h:502` | TB 内联操作 |
| `register_vcpu_insn_exec_cb()` | `qemu-plugin.h:510` | 指令执行回调 |
| `register_vcpu_mem_cb()` | `qemu-plugin.h:775` | 内存访问回调 |
| `register_vcpu_mem_inline_per_vcpu()` | `qemu-plugin.h:793` | 内存内联操作 |
| `register_vcpu_syscall_cb()` | `qemu-plugin.h:888` | 系统调用回调 |
| `qemu_plugin_scoreboard_new()` | `qemu-plugin.h:1283` | 创建 Scoreboard |
| `qemu_plugin_scoreboard_find()` | `qemu-plugin.h:1302` | 查找 per-vCPU 数据 |
| `qemu_plugin_register_atexit_cb()` | `qemu-plugin.h:979` | 退出回调 |

---

## 26. 附录 B：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `include/plugins/qemu-plugin.h` | ~1400 | 公共 API 头文件（所有 Plugin 引用此文件） |
| `plugins/plugin.h` | ~70 | 内部头文件（qemu_plugin_state、ctx） |
| `plugins/loader.c` | ~420 | 加载/卸载/命令行解析 |
| `plugins/core.c` | ~880 | 回调分发、vCPU 生命周期、内存回调、atexit |
| `plugins/api.c` | ~740 | API 实现（注册、Scoreboard、u64 操作） |
| `accel/tcg/plugin-gen.c` | ~510 | 代码生成集成（注入回调到 TCG IR） |
| `accel/tcg/translator.c` | ~240 | 翻译循环 Plugin hook 点 |
| `contrib/plugins/` | 14 文件 | 官方示例插件 |
| `tests/tcg/plugins/` | 11 文件 | 测试插件 |

---

> **文档版本**：v1.0
> **源码版本**：QEMU 11.0.50 (commit 基于 2025-07 主线)
> **分析工具**：zoekt + ctags + cscope + 手动源码验证
> **交叉验证**：所有行号均经 view 验证
