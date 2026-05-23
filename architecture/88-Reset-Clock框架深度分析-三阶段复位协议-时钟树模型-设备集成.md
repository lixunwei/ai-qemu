# Reset/Clock 框架深度分析

> 文档编号：88  
> 分析目标：三阶段 Reset 协议、Clock 树模型、设备集成 API  
> 源码版本：QEMU 11.0.50  
> 核心文件：hw/core/resettable.c (278行)、hw/core/reset.c (177行)、hw/core/clock.c (228行)、hw/core/qdev-clock.c (184行)

---

## 一、Reset 框架

### 1.1 设计动机

QEMU 需要精确模拟硬件复位行为。复位不是一个原子操作——真实硬件中，复位信号沿总线树逐级传播，设备间存在复位时序依赖。QEMU 使用**三阶段协议**解决这个问题。

### 1.2 三阶段复位协议

```
┌────────┐    ┌────────┐    ┌────────┐
│ Enter  │ →  │  Hold  │ →  │  Exit  │
│ Phase  │    │  Phase │    │  Phase │
└────────┘    └────────┘    └────────┘
  本地复位       可影响他人      离开复位
  不影响他人     (IRQ/DMA)      恢复运行
```

| 阶段 | 约束 | 用途 |
|------|------|------|
| **Enter** | 不得影响其他设备（不可拉 IRQ、不可写 DMA） | 重置本地寄存器 |
| **Hold** | 可以影响其他设备 | 拉低/释放 IRQ 线 |
| **Exit** | 可以影响其他设备 | 恢复设备运行 |

### 1.3 Resettable 接口

```c
// include/hw/core/resettable.h
struct ResettableClass {
    InterfaceClass parent_class;
    ResettablePhases phases;          // enter/hold/exit 回调
    ResettableGetState get_state;     // 获取复位状态
    ResettableChildForeach child_foreach; // 遍历子对象
};

struct ResettableState {
    unsigned count;                    // 复位引用计数
    bool hold_phase_pending;          // hold 阶段待执行
    bool exit_phase_in_progress;      // exit 阶段执行中
};

// ResetType 枚举
typedef enum ResetType {
    RESET_TYPE_COLD,              // 冷复位（上电）
    RESET_TYPE_SNAPSHOT_LOAD,     // 快照加载
    RESET_TYPE_WAKEUP,            // 唤醒
    RESET_TYPE_S390_CPU_INITIAL,  // S390 特有
    RESET_TYPE_S390_CPU_NORMAL,   // S390 特有
} ResetType;
```

### 1.4 核心执行逻辑

```c
// hw/core/resettable.c
void resettable_reset(Object *obj, ResetType type) {
    resettable_assert_reset(obj, type);   // Enter + Hold
    resettable_release_reset(obj, type);  // Exit
}

void resettable_assert_reset(Object *obj, ResetType type) {
    enter_phase_in_progress = true;
    resettable_phase_enter(obj, NULL, type);  // 递归所有子对象
    enter_phase_in_progress = false;
    
    resettable_phase_hold(obj, NULL, type);   // 递归所有子对象
}

void resettable_release_reset(Object *obj, ResetType type) {
    exit_phase_in_progress += 1;
    resettable_phase_exit(obj, NULL, type);   // 递归所有子对象
    exit_phase_in_progress -= 1;
}
```

### 1.5 引用计数机制

```c
static void resettable_phase_enter(Object *obj, void *opaque, ResetType type) {
    ResettableState *s = rc->get_state(obj);
    
    // 只有首次进入复位才执行 enter 回调
    if (s->count++ == 0) {
        action_needed = true;
    }
    assert(s->count <= 50);  // 防止无限循环
    
    // 递归处理所有子对象
    resettable_child_foreach(rc, obj, resettable_phase_enter, NULL, type);
    
    if (action_needed && rc->phases.enter) {
        rc->phases.enter(obj, type);
    }
}

static void resettable_phase_exit(Object *obj, void *opaque, ResetType type) {
    ResettableState *s = rc->get_state(obj);
    
    resettable_child_foreach(rc, obj, resettable_phase_exit, NULL, type);
    
    // 只有最后一个复位源释放时才执行 exit
    if (--s->count == 0) {
        if (rc->phases.exit) {
            rc->phases.exit(obj, type);
        }
    }
}
```

**关键设计**：多个复位源可以同时复位同一个设备（引用计数），只有所有源都释放后设备才真正退出复位。

### 1.6 全局复位容器

```c
// hw/core/reset.c
static ResettableContainer *get_root_reset_container(void) {
    static ResettableContainer *root_reset_container;
    if (!root_reset_container) {
        root_reset_container = RESETTABLE_CONTAINER(object_new(TYPE_RESETTABLE_CONTAINER));
    }
    return root_reset_container;
}

// 全局复位入口
void qemu_devices_reset(ResetType type) {
    resettable_reset(OBJECT(get_root_reset_container()), type);
}

// 注册到全局复位
void qemu_register_resettable(Object *obj) {
    resettable_container_add(get_root_reset_container(), obj);
}
```

### 1.7 Legacy 回调兼容

```c
// 旧式 API：qemu_register_reset(func, opaque)
// 内部包装为 LegacyReset 对象，在 hold 阶段调用 func
struct LegacyReset {
    Object parent;
    ResettableState reset_state;
    QEMUResetHandler *func;
    void *opaque;
    bool skip_on_snapshot_load;
};

static void legacy_reset_hold(Object *obj, ResetType type) {
    LegacyReset *lr = LEGACY_RESET(obj);
    if (type == RESET_TYPE_SNAPSHOT_LOAD && lr->skip_on_snapshot_load) {
        return;  // 快照加载时跳过
    }
    lr->func(lr->opaque);
}
```

### 1.8 设备热插拔时的复位传播

```c
void resettable_change_parent(Object *obj, Object *newp, Object *oldp) {
    unsigned newp_count = resettable_get_count(newp);
    unsigned oldp_count = resettable_get_count(oldp);
    
    // 新父节点复位深度更深 → 对设备追加复位
    for (i = oldp_count; i < newp_count; i++) {
        resettable_assert_reset(obj, RESET_TYPE_COLD);
    }
    // 旧父节点复位深度更深 → 释放多余的复位
    for (i = newp_count; i < oldp_count; i++) {
        resettable_release_reset(obj, RESET_TYPE_COLD);
    }
}
```

---

## 二、Clock 框架

### 2.1 设计模型

QEMU 使用**时钟树**模型来模拟硬件时钟分配网络。每个 Clock 对象存储其周期（period），通过 source→children 关系形成树状结构。

```
┌─────────────────┐
│  PLL (source)   │  period = CLOCK_PERIOD_FROM_HZ(1GHz)
│  multiplier=1   │
│  divider=1      │
└───────┬─────────┘
        │ children
   ┌────┴─────┐
   │          │
┌──▼──┐   ┌──▼──┐
│ CPU │   │ APB │  divider=4 → 250MHz
│Clock│   │Clock│
└─────┘   └──┬──┘
              │
          ┌───▼───┐
          │ UART  │
          │ Clock │
          └───────┘
```

### 2.2 Clock 数据结构

```c
// include/hw/core/clock.h
struct Clock {
    Object parent_obj;
    
    uint64_t period;           // 周期，单位：2^-32 纳秒
    char *canonical_path;      // QOM 路径缓存
    ClockCallback *callback;   // 频率变化回调
    void *callback_opaque;
    unsigned int callback_events;  // ClockUpdate | ClockPreUpdate
    
    uint32_t multiplier;       // 倍频系数
    uint32_t divider;          // 分频系数
    
    Clock *source;             // 父时钟
    QLIST_HEAD(, Clock) children;  // 子时钟列表
    QLIST_ENTRY(Clock) sibling;    // 兄弟链表节点
};
```

### 2.3 周期表示

```c
// 周期单位：2^-32 纳秒（定点数）
#define CLOCK_PERIOD_1SEC  (1000000000llu << 32)

// 频率 ↔ 周期转换
#define CLOCK_PERIOD_FROM_HZ(hz)  (CLOCK_PERIOD_1SEC / (hz))
#define CLOCK_PERIOD_TO_HZ(per)   (CLOCK_PERIOD_1SEC / (per))
#define CLOCK_PERIOD_FROM_NS(ns)  ((ns) * (CLOCK_PERIOD_1SEC / 1000000000llu))

// 精度：
// 100MHz → 分辨率 ~2mHz
// 1GHz   → 分辨率 ~0.2Hz
// 10GHz  → 分辨率 ~20Hz
```

### 2.4 核心操作

```c
// 设置时钟周期
bool clock_set(Clock *clk, uint64_t period) {
    if (clk->period == period) return false;  // 未变化
    clk->period = period;
    return true;
}

// 传播到子时钟
void clock_propagate(Clock *clk) {
    clock_propagate_period(clk, true);
}

static void clock_propagate_period(Clock *clk, bool call_callbacks) {
    uint64_t child_period = muldiv64(clk->period, clk->multiplier, clk->divider);
    
    QLIST_FOREACH(child, &clk->children, sibling) {
        if (child->period != child_period) {
            clock_call_callback(child, ClockPreUpdate);  // 预通知
            child->period = child_period;
            clock_call_callback(child, ClockUpdate);     // 更新通知
            clock_propagate_period(child, call_callbacks); // 递归
        }
    }
}

// 连接时钟源
void clock_set_source(Clock *clk, Clock *src) {
    assert(!clk->source);  // 不支持动态更换源
    clk->period = clock_get_child_period(src);
    QLIST_INSERT_HEAD(&src->children, clk, sibling);
    clk->source = src;
}

// 设置倍频/分频
bool clock_set_mul_div(Clock *clk, uint32_t multiplier, uint32_t divider) {
    clk->multiplier = multiplier;
    clk->divider = divider;
    return true;  // 调用者需手动 clock_propagate()
}
```

### 2.5 设备时钟集成 API

```c
// hw/core/qdev-clock.c

// 声明输出时钟
Clock *qdev_init_clock_out(DeviceState *dev, const char *name);

// 声明输入时钟（带回调）
Clock *qdev_init_clock_in(DeviceState *dev, const char *name,
                          ClockCallback *callback, void *opaque,
                          unsigned int events);

// 批量声明（用于 class_init）
void qdev_init_clocks(DeviceState *dev, const ClockPortInitArray clocks);

// 连接时钟
void qdev_connect_clock_in(DeviceState *dev, const char *name, Clock *source);

// 别名（暴露内部时钟到外部接口）
Clock *qdev_alias_clock(DeviceState *dev, const char *name,
                        DeviceState *alias_dev, const char *alias_name);
```

**使用模式**：
```c
// 设备 realize 前声明时钟
static void my_device_init(Object *obj) {
    MyDevice *s = MY_DEVICE(obj);
    s->clk_in = qdev_init_clock_in(DEVICE(obj), "clk",
                                    my_clk_callback, s, ClockUpdate);
    s->clk_out = qdev_init_clock_out(DEVICE(obj), "clk-out");
}

// 板级连接
qdev_connect_clock_in(uart, "clk", qdev_get_clock_out(pll, "apb-clk"));
```

### 2.6 VMState 迁移支持

```c
// hw/core/clock-vmstate.c
extern const VMStateDescription vmstate_clock;

// 在设备 vmsd 中包含：
#define VMSTATE_CLOCK(field, state) \
    VMSTATE_CLOCK_V(field, state, 0)
```

时钟的 period 值会随 VMState 迁移保存/恢复。

---

## 三、Reset 与 Clock 的协作

### 3.1 复位时的时钟行为

设备在 Enter 阶段通常会：
1. 重置内部寄存器
2. 停止内部定时器

设备在 Hold 阶段可以：
1. 将输出时钟设为 0（停止）
2. 拉低中断线

设备在 Exit 阶段：
1. 恢复时钟输出
2. 重启定时器

### 3.2 典型设备实现

```c
static void my_timer_reset_enter(Object *obj, ResetType type) {
    MyTimer *s = MY_TIMER(obj);
    s->counter = 0;
    s->control = 0;
}

static void my_timer_reset_hold(Object *obj, ResetType type) {
    MyTimer *s = MY_TIMER(obj);
    // 安全地影响其他设备
    qemu_set_irq(s->irq, 0);  // 释放中断
    clock_set(s->clk_out, 0); // 停止输出时钟
    clock_propagate(s->clk_out);
}

static void my_timer_reset_exit(Object *obj, ResetType type) {
    MyTimer *s = MY_TIMER(obj);
    // 恢复正常时钟
    clock_set(s->clk_out, CLOCK_PERIOD_FROM_HZ(s->default_freq));
    clock_propagate(s->clk_out);
}
```

---

## 四、系统复位触发路径

```
用户请求 (system_reset / PSCI SYSTEM_RESET)
    │
    ▼
qemu_system_reset_request(SHUTDOWN_CAUSE_*)
    │
    ▼
main_loop → qemu_system_reset()
    │
    ▼
MachineClass->reset()
    │
    ▼
qemu_devices_reset(RESET_TYPE_COLD)
    │
    ▼
resettable_reset(root_container)
    │
    ├─ Phase 1: resettable_phase_enter() — 递归所有注册对象
    │   ├─ Machine → Bus → Device → Bus → Device ...
    │   └─ 每个设备: DeviceClass->phases.enter()
    │
    ├─ Phase 2: resettable_phase_hold() — 递归所有注册对象
    │   ├─ 每个设备: DeviceClass->phases.hold()
    │   └─ Legacy: QEMUResetHandler->func()
    │
    └─ Phase 3: resettable_phase_exit() — 递归所有注册对象
        └─ 每个设备: DeviceClass->phases.exit()
```

---

## 五、与硬件对比

| 方面 | 真实硬件 | QEMU 模拟 |
|------|----------|-----------|
| 复位传播 | 电气信号沿复位树传播 | 三阶段递归遍历 QOM 树 |
| 时序 | 有真实延迟（µs/ms） | 瞬时（同一时间点完成） |
| 多源复位 | 多个复位引脚可同时有效 | 引用计数 (count) |
| 时钟分配 | PLL/分频器物理电路 | Clock 对象树 + 数学计算 |
| 时钟 glitch | 切换时可能有毛刺 | 无（原子更新 period） |
| 频率精度 | 连续值 | 2^-32ns 定点精度 |

**差异分析**：
- QEMU 的三阶段协议是对硬件复位时序的**语义等价抽象**——保证 enter 全部完成后才允许 hold，hold 全部完成后才允许 exit
- 真实硬件中时钟切换可能产生 glitch，QEMU 不模拟这一点
- QEMU 不模拟复位传播的物理延迟

---

## 六、源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `hw/core/resettable.c` | 278 | 三阶段复位核心实现 |
| `hw/core/reset.c` | 177 | 全局复位容器 + Legacy 兼容 |
| `hw/core/resetcontainer.c` | 78 | ResettableContainer 类型 |
| `hw/core/clock.c` | 228 | Clock QOM 对象实现 |
| `hw/core/qdev-clock.c` | 184 | 设备时钟声明/连接 API |
| `hw/core/clock-vmstate.c` | ~40 | Clock VMState 迁移 |
| `include/hw/core/resettable.h` | 238 | Resettable 接口定义 |
| `include/hw/core/clock.h` | 373 | Clock 结构 + 内联 API |
| `include/hw/core/qdev-clock.h` | ~80 | 设备时钟 API 声明 |
| `include/system/reset.h` | 127 | 全局复位 API |

| 关键函数 | 位置 | 职责 |
|----------|------|------|
| `resettable_reset()` | resettable.c:42 | 完整复位（assert + release） |
| `resettable_assert_reset()` | resettable.c:49 | Enter + Hold 阶段 |
| `resettable_release_reset()` | resettable.c:63 | Exit 阶段 |
| `resettable_phase_enter()` | resettable.c:96 | 递归 Enter（引用计数++） |
| `resettable_phase_hold()` | resettable.c:143 | 递归 Hold |
| `resettable_phase_exit()` | resettable.c:168 | 递归 Exit（引用计数--） |
| `resettable_change_parent()` | resettable.c:205 | 热插拔复位传播 |
| `qemu_devices_reset()` | reset.c:173 | 全局复位入口 |
| `qemu_register_resettable()` | reset.c:163 | 注册到全局容器 |
| `clock_set()` | clock.c:53 | 设置时钟周期 |
| `clock_propagate()` | clock.c:107 | 传播到子时钟 |
| `clock_set_source()` | clock.c:113 | 连接时钟源 |
| `clock_set_mul_div()` | clock.c:143 | 设置倍频/分频 |
| `qdev_init_clock_in()` | qdev-clock.c:77 | 设备输入时钟 |
| `qdev_init_clock_out()` | qdev-clock.c:68 | 设备输出时钟 |
| `qdev_connect_clock_in()` | qdev-clock.c:180 | 连接设备时钟 |
