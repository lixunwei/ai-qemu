# 热迁移子系统深度分析

> 文档编号：89  
> 分析目标：热迁移架构、迭代传输模型、RAM 脏页跟踪、postcopy、multifd  
> 源码版本：QEMU 11.0.50  
> 核心文件：migration/migration.c (4021行)、migration/ram.c (4771行)、migration/savevm.c (3717行)、migration/multifd.c (1573行)

---

## 一、概述

QEMU 热迁移（Live Migration）在 VM 运行期间将其完整状态（RAM + 设备）传输到目标主机。核心挑战是：**如何在持续产生脏页的情况下收敛传输**。

```
源端 (Source)                          目标端 (Destination)
┌──────────────┐                      ┌──────────────┐
│ Guest 运行中  │  ──── 网络 ────→     │  等待接收    │
│              │                      │              │
│ 迭代传输 RAM │  setup → iterate*    │  加载 RAM    │
│ + 设备状态    │  → complete          │  + 设备状态  │
│              │                      │              │
│ 停机 → 完成  │  switchover          │  恢复运行 →  │
└──────────────┘                      └──────────────┘
```

---

## 二、迁移状态机

```c
// 状态转换
NONE → SETUP → ACTIVE → DEVICE → COMPLETED
                  │                    ↑
                  │ (postcopy)         │
                  ├→ POSTCOPY_DEVICE → POSTCOPY_ACTIVE → COMPLETED
                  │
                  ├→ CANCELLING → CANCELLED
                  └→ FAILING → FAILED
```

| 状态 | 含义 |
|------|------|
| SETUP | 初始化数据结构、建立连接 |
| ACTIVE | 迭代传输 RAM（precopy 阶段） |
| DEVICE | 停机后传输设备状态 |
| POSTCOPY_DEVICE | postcopy: 设备状态发送中 |
| POSTCOPY_ACTIVE | postcopy: 按需传输剩余页 |
| COMPLETED | 迁移成功完成 |
| COLO | 进入 COLO 持续复制模式 |

---

## 三、迁移线程主循环

```c
// migration/migration.c:3587
static void *migration_thread(void *opaque) {
    // 1. Setup 阶段
    multifd_send_setup();                    // 初始化多通道
    qemu_savevm_state_header(s->to_dst_file); // 发送流头部
    qemu_savevm_state_do_setup(f, ...);      // 各设备 save_setup()
    
    // 2. 迭代传输（VM 仍在运行）
    while (migration_is_active()) {
        if (!migration_rate_exceeded(f)) {
            MigIterateState iter = migration_iteration_run(s);
            if (iter == MIG_ITERATE_BREAK) break;  // 可以完成了
        }
        
        // 检测错误/恢复
        thr_error = migration_detect_error(s);
        
        // 限速
        urgent = migration_rate_limit();
    }
    
    // 3. 结束
    migration_iteration_finish(s);
}
```

### 3.1 每次迭代

```c
static MigIterateState migration_iteration_run(MigrationState *s) {
    // 查询待传输数据量
    qemu_savevm_query_pending(&pending, false);
    
    // precopy: 剩余数据 ≤ threshold_size → 可以完成
    if (can_switchover && pending.total_bytes <= s->threshold_size) {
        migration_completion(s);       // 停机 + 传输剩余
        return MIG_ITERATE_BREAK;
    }
    
    // 是否切换到 postcopy？
    if (postcopy_should_start(s, &pending)) {
        postcopy_start(s, ...);
        return MIG_ITERATE_SKIP;
    }
    
    // 继续迭代
    qemu_savevm_state_iterate(f, in_postcopy);
    return MIG_ITERATE_RESUME;
}
```

### 3.2 完成阶段（Switchover）

```c
static int migration_completion_precopy(MigrationState *s) {
    // 1. 停止 VM
    migration_stop_vm(s, RUN_STATE_FINISH_MIGRATE);
    
    // 2. 通知 switchover 开始
    migration_switchover_start(s, NULL);
    
    // 3. 传输最后一批数据
    qemu_savevm_state_complete_precopy(s);
    // → ram_save_complete(): 发送所有剩余脏页
    // → 各设备 save_complete(): 发送设备最终状态
}
```

---

## 四、SaveVM 框架

### 4.1 SaveStateEntry 注册

每个需要迁移的子系统通过 `register_savevm_live()` 注册处理器：

```c
// include/migration/register.h
typedef struct SaveVMHandlers {
    // BQL 持有下执行
    int (*save_setup)(QEMUFile *f, void *opaque, Error **errp);
    int (*save_complete)(QEMUFile *f, void *opaque);
    void (*save_cleanup)(void *opaque);
    void (*save_state)(QEMUFile *f, void *opaque);
    
    // 迁移线程中执行（可能不持有 BQL）
    int (*save_live_iterate)(QEMUFile *f, void *opaque);
    
    // pending 查询
    void (*state_pending_estimate)(void *opaque, MigPendingData *p);
    void (*state_pending_exact)(void *opaque, MigPendingData *p);
    
    // 加载端
    int (*load_state)(QEMUFile *f, void *opaque, int version_id);
    int (*load_setup)(QEMUFile *f, void *opaque, Error **errp);
    void (*load_cleanup)(void *opaque);
    
    // postcopy 支持
    bool (*has_postcopy)(void *opaque);
    int (*load_state_buffer)(void *opaque, char *data, size_t len);
} SaveVMHandlers;
```

### 4.2 迁移流格式

```
┌─────────────────────────────────────────────┐
│ Header: Magic + Version                      │
├─────────────────────────────────────────────┤
│ Configuration Section (optional)             │
├─────────────────────────────────────────────┤
│ Section: QEMU_VM_SECTION_START (id, name)   │
│   → save_setup() 输出                       │
├─────────────────────────────────────────────┤
│ Section: QEMU_VM_SECTION_PART (id)          │  ← 重复多次
│   → save_live_iterate() 输出                │
├─────────────────────────────────────────────┤
│ Section: QEMU_VM_SECTION_END (id)           │
│   → save_complete() 输出                    │
├─────────────────────────────────────────────┤
│ Section: QEMU_VM_SECTION_FULL               │
│   → 非迭代设备的 vmstate                    │
├─────────────────────────────────────────────┤
│ QEMU_VM_EOF                                 │
└─────────────────────────────────────────────┘
```

### 4.3 迭代流程

```c
// migration/savevm.c:1504
int qemu_savevm_state_iterate(QEMUFile *f, bool postcopy) {
    bool all_finished = true;
    
    QTAILQ_FOREACH(se, &savevm_state.handlers, entry) {
        if (!se->ops->save_live_iterate) continue;
        
        save_section_header(f, se, QEMU_VM_SECTION_PART);
        ret = se->ops->save_live_iterate(f, se->opaque);
        save_section_footer(f, se);
        
        if (!ret) all_finished = false;  // 0 = 还有更多数据
    }
    return all_finished;
}
```

---

## 五、RAM 迁移

### 5.1 架构

RAM 是最大的迁移组件（通常 GB 级），使用专门的脏页跟踪和迭代传输机制：

```
┌──────────────────────────────────────────────┐
│              RAM 迁移状态                      │
│                                              │
│  dirty_bitmap[block][page]  → 标记脏页       │
│  ram_find_and_save_block()  → 扫描脏页发送   │
│  migration_bitmap_sync()    → 从 KVM 同步    │
│                                              │
│  压缩/去重:                                   │
│    XBZRLE  → 增量压缩                        │
│    zero-page → 零页检测                       │
│    multifd → 多通道并行                       │
└──────────────────────────────────────────────┘
```

### 5.2 核心数据结构

```c
// migration/ram.c
struct RAMState {
    QemuMutex bitmap_mutex;
    
    // 脏页统计
    uint64_t migration_dirty_pages;  // 剩余脏页数
    
    // 扫描状态
    RAMBlock *last_seen_block;
    ram_addr_t last_page;
    
    // 性能计数
    uint64_t target_page_count;
    uint64_t zero_pages;
    uint64_t norm_pages;
};
```

### 5.3 迭代传输

```c
// ram_save_iterate() — 每次迭代调用
static int ram_save_iterate(QEMUFile *f, void *opaque) {
    while (!migration_rate_exceeded(f) || postcopy_has_request(rs)) {
        // 找到下一个脏页并发送
        pages = ram_find_and_save_block(rs);
        if (pages == 0) { done = 1; break; }  // 没有更多脏页
        
        // 限制单次迭代时间 (MAX_WAIT ~50ms)
        if (elapsed > MAX_WAIT) break;
    }
    return done;  // 1 = 所有页已发送
}
```

### 5.4 脏页跟踪与同步

```c
// migration_bitmap_sync(): 从 KVM 获取脏页信息
static void migration_bitmap_sync(RAMState *rs, bool last_stage) {
    // 调用 KVM_GET_DIRTY_LOG 或 dirty ring
    memory_global_dirty_log_sync(last_stage);
    
    // 遍历所有 RAMBlock
    RAMBLOCK_FOREACH_NOT_IGNORED(block) {
        // 将 KVM 脏位图合并到迁移位图
        ramblock_sync_dirty_bitmap(rs, block);
    }
}
```

### 5.5 页面发送

```c
static int ram_save_target_page(RAMState *rs, PageSearchStatus *pss) {
    // 1. 零页检测
    if (buffer_is_zero(page_data, TARGET_PAGE_SIZE)) {
        // 发送零页标志（仅几字节）
        return ram_save_zero_page(rs, pss);
    }
    
    // 2. Multifd 通道发送
    if (multifd_enabled()) {
        return ram_save_multifd_page(block, offset);
    }
    
    // 3. 普通发送（主通道）
    return ram_save_page(rs, pss);
}
```

---

## 六、Multifd（多通道并行传输）

### 6.1 架构

```
源端                                    目标端
┌─────────┐                            ┌─────────┐
│ Main CH │ ← 设备状态/控制 →          │ Main CH │
├─────────┤                            ├─────────┤
│ CH #0   │ ──── RAM pages ────→       │ CH #0   │
│ CH #1   │ ──── RAM pages ────→       │ CH #1   │
│ CH #2   │ ──── RAM pages ────→       │ CH #2   │
│  ...    │                            │  ...    │
│ CH #N   │ ──── RAM pages ────→       │ CH #N   │
└─────────┘                            └─────────┘
```

### 6.2 压缩方法

| 方法 | 文件 | 说明 |
|------|------|------|
| none | multifd-nocomp.c | 无压缩直传 |
| zlib | multifd-zlib.c | zlib 压缩 |
| qpl | multifd-qpl.c | Intel QAT 加速 |
| qatzip | multifd-qatzip.c | Intel QATzip |
| uadk | multifd-uadk.c | HiSilicon UADK |

### 6.3 发送流程

```c
// ram_save_multifd_page(): 将页面推入 multifd 队列
// multifd 发送线程从队列取出并通过独立 socket 发送
```

---

## 七、Postcopy 迁移

### 7.1 原理

Postcopy 在 switchover 后仍有未传输的 RAM 页面。目标端 VM 立即恢复运行，当访问到未到达的页面时，通过 userfaultfd 通知源端发送。

```
时间线：
Source:  [--- precopy iterate ---][stop][switch] → idle
Dest:    [--- receive pages ---]  [resume]  → 按需请求页面
                                              ← userfaultfd page fault
```

### 7.2 关键文件

```c
// migration/postcopy-ram.c (2239行)
// - postcopy_ram_setup(): 初始化 userfaultfd
// - postcopy_place_page(): 将接收的页面放置到 guest 地址空间
// - postcopy_request_page(): 请求特定页面
```

---

## 八、收敛策略

### 8.1 Auto-Converge

当脏页速率持续超过传输速率时，QEMU 逐步降低 guest CPU 执行时间：

```c
// cpu_throttle_set(): 设置 CPU 降速比例 (0-99%)
// 效果：减少脏页产生速率，使迁移收敛
```

### 8.2 下线时间预算

```
threshold_size = bandwidth * max_downtime
```

当剩余数据量 ≤ `threshold_size` 时，可以安全 switchover（停机时间不超过 max_downtime）。

### 8.3 XBZRLE 增量压缩

对重复传输的页面，只发送与上次传输内容的差异（XOR + RLE 编码）。

---

## 九、设备状态迁移（VMState）

### 9.1 VMStateDescription

```c
// 声明式设备状态描述
const VMStateDescription vmsd_my_device = {
    .name = "my_device",
    .version_id = 2,
    .minimum_version_id = 1,
    .fields = (VMStateField[]) {
        VMSTATE_UINT32(reg_a, MyDevice),
        VMSTATE_UINT64(reg_b, MyDevice),
        VMSTATE_TIMER_PTR(timer, MyDevice),
        VMSTATE_END_OF_LIST()
    },
};
```

### 9.2 VMState vs SaveVMHandlers

| 特性 | VMStateDescription | SaveVMHandlers |
|------|-------------------|----------------|
| 适用 | 简单设备状态 | 复杂子系统（RAM、block） |
| 方式 | 声明式（自动序列化） | 过程式（手动实现） |
| 迭代 | 不支持 | 支持 (save_live_iterate) |
| 前向兼容 | 通过版本号 | 手动处理 |

---

## 十、关键流程时序

```
Source                                    Destination
  │                                         │
  ├─ qmp_migrate("tcp:host:port")           │
  │  → migrate_fd_connect()                 │
  │  → 启动 migration_thread               │
  │                                         │
  ├─ [SETUP] savevm_state_header()          │
  ├─ [SETUP] savevm_state_setup()           │
  │  → ram_save_setup(): 初始化位图          ├─ load_setup()
  │                                         │
  ├─ [ACTIVE] iterate #1                    │
  │  → ram_save_iterate(): 发送脏页         ├─ ram_load(): 接收页面
  ├─ [ACTIVE] iterate #2                    │
  │  ...                                    │
  ├─ [ACTIVE] iterate #N                    │
  │  (pending ≤ threshold)                  │
  │                                         │
  ├─ [DEVICE] migration_stop_vm()           │
  ├─ [DEVICE] savevm_complete_precopy()     │
  │  → ram_save_complete(): 最后脏页        ├─ ram_load_complete()
  │  → 各设备 vmstate 序列化               ├─ 设备 vmstate 恢复
  │  → QEMU_VM_EOF                          ├─ vm_start()
  │                                         │
  ├─ [COMPLETED]                            ├─ Guest 恢复运行
  │                                         │
```

---

## 十一、源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `migration/migration.c` | 4021 | 迁移状态机、线程主循环、QMP 接口 |
| `migration/ram.c` | 4771 | RAM 脏页跟踪、迭代发送、零页/压缩 |
| `migration/savevm.c` | 3717 | SaveVM 框架、流格式、设备状态编排 |
| `migration/multifd.c` | 1573 | 多通道并行传输框架 |
| `migration/postcopy-ram.c` | 2239 | Postcopy userfaultfd 机制 |
| `migration/vmstate.c` | 923 | VMState 序列化/反序列化 |
| `migration/vmstate-types.c` | 926 | VMState 类型实现 |
| `migration/qemu-file.c` | 912 | 迁移文件 I/O 抽象 |
| `migration/options.c` | 1587 | 迁移参数/能力管理 |
| `migration/channel.c` | 425 | 传输通道抽象 |
| `migration/rdma.c` | 4006 | RDMA 传输后端 |
| `migration/xbzrle.c` | 323 | XBZRLE 增量压缩 |
| `migration/block-dirty-bitmap.c` | 1268 | 块设备脏位图迁移 |
| `migration/colo.c` | 950 | COLO 持续复制 |
| `migration/dirtyrate.c` | 934 | 脏页速率计算 |
| `include/migration/register.h` | 340 | SaveVMHandlers 接口定义 |
| `migration/migration.h` | 607 | MigrationState 结构 |

| 关键函数 | 位置 | 职责 |
|----------|------|------|
| `migration_thread()` | migration.c:3587 | ★ 迁移主线程 |
| `migration_iteration_run()` | migration.c:3293 | 每次迭代决策 |
| `migration_completion()` | migration.c:2833 | switchover + 完成 |
| `migration_completion_precopy()` | migration.c:2784 | 停 VM + 传最后数据 |
| `qemu_savevm_state_header()` | savevm.c:1360 | 发送流头部 |
| `qemu_savevm_state_iterate()` | savevm.c:1504 | 调用各设备迭代 |
| `qemu_savevm_state_complete_precopy()` | savevm.c:1775 | 发送最终状态 |
| `ram_save_setup()` | ram.c:3112 | RAM 初始化脏位图 |
| `ram_save_iterate()` | ram.c:3255 | RAM 迭代发送脏页 |
| `ram_save_complete()` | ram.c:3368 | RAM 发送剩余脏页 |
| `ram_find_and_save_block()` | ram.c | 扫描位图找脏页 |
| `migration_bitmap_sync()` | ram.c:1134 | 从 KVM 同步脏位图 |
| `multifd_send_setup()` | multifd.c | 初始化多通道 |
| `postcopy_start()` | migration.c | 切换到 postcopy |
