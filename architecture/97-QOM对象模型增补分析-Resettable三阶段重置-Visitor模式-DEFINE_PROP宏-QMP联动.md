# Doc 97: QOM 对象模型增补分析

## Resettable 三阶段重置 · Visitor 模式 · DEFINE_PROP 宏 · QOM-QMP 联动

> QEMU 11.0.50 · qom/ + hw/core/ 子系统
> 分析日期: 2025-01
> 前置文档: Doc 01 (QOM基础)、Doc 22 (QOM完整分析)

---

## 目录

1. [Resettable 三阶段重置接口](#1-resettable-三阶段重置接口)
2. [Visitor 模式与属性访问](#2-visitor-模式与属性访问)
3. [DEFINE_PROP 宏系统](#3-define_prop-宏系统)
4. [QOM-QMP 联动](#4-qom-qmp-联动)
5. [综合流程：属性从命令行到设备](#5-综合流程属性从命令行到设备)

---

## 1. Resettable 三阶段重置接口

### 1.1 设计动机

旧的单阶段 `DeviceClass::reset` 存在问题：
- 设备 A 的 reset 回调可能拉高 IRQ 线
- 此时设备 B 还未 reset，接收到非预期的中断
- 结果取决于 reset 遍历顺序 → **不确定性**

三阶段设计将 reset 分为 enter/hold/exit，保证所有设备先进入 reset 状态（enter），再允许产生副作用（hold）。

### 1.2 核心数据结构

```c
// include/hw/core/resettable.h:111-127
typedef struct ResettablePhases {
    ResettableEnterPhase enter;   // 阶段1: 仅重置本地状态
    ResettableHoldPhase hold;     // 阶段2: 可产生外部副作用
    ResettableExitPhase exit;     // 阶段3: 离开 reset 状态
} ResettablePhases;

struct ResettableClass {
    InterfaceClass parent_class;
    ResettablePhases phases;          // 三个阶段回调
    ResettableGetState get_state;     // 获取 ResetState
    ResettableChildForeach child_foreach;  // 遍历子对象
};

// include/hw/core/resettable.h:141-145
struct ResettableState {
    unsigned count;               // reset 嵌套层级计数
    bool hold_phase_pending;      // 需要执行 hold
    bool exit_phase_in_progress;  // 正在执行 exit
};
```

### 1.3 三阶段协议

```
resettable_reset(obj, type)
  ├─ resettable_assert_reset()
  │    ├─ resettable_phase_enter(obj)    ← 阶段1
  │    │    ├─ count++ (首次进入时 action_needed=true)
  │    │    ├─ child_foreach → 递归 enter 所有子对象
  │    │    └─ rc->phases.enter(obj)     ← 重置本地状态
  │    │         (不允许: 拉 IRQ、读写 guest 内存)
  │    │
  │    └─ resettable_phase_hold(obj)     ← 阶段2
  │         ├─ child_foreach → 递归 hold 所有子对象 (先子后父)
  │         └─ rc->phases.hold(obj)      ← 可产生副作用
  │              (允许: 拉 IRQ、发出信号)
  │
  └─ resettable_release_reset()
       └─ resettable_phase_exit(obj)     ← 阶段3
            ├─ exit_phase_in_progress = true
            ├─ child_foreach → 递归 exit 所有子对象
            ├─ count--
            └─ if (count == 0): rc->phases.exit(obj) ← 离开 reset
```

### 1.4 实现细节

```c
// hw/core/resettable.c:96-141 — Enter 阶段
static void resettable_phase_enter(Object *obj, void *opaque, ResetType type) {
    ResettableState *s = rc->get_state(obj);

    if (s->count++ == 0) {
        action_needed = true;  // 首次进入才执行
    }
    assert(s->count <= 50);    // 防止无限循环

    // 先递归子对象（确保整棵树都进入 reset）
    resettable_child_foreach(rc, obj, resettable_phase_enter, NULL, type);

    if (action_needed) {
        if (rc->phases.enter) rc->phases.enter(obj, type);
        s->hold_phase_pending = true;
    }
}

// hw/core/resettable.c:143-166 — Hold 阶段
static void resettable_phase_hold(Object *obj, void *opaque, ResetType type) {
    // 先子后父！
    resettable_child_foreach(rc, obj, resettable_phase_hold, NULL, type);

    if (s->hold_phase_pending) {
        s->hold_phase_pending = false;
        if (rc->phases.hold) rc->phases.hold(obj, type);
    }
}

// hw/core/resettable.c:168-190 — Exit 阶段
static void resettable_phase_exit(Object *obj, void *opaque, ResetType type) {
    s->exit_phase_in_progress = true;
    resettable_child_foreach(rc, obj, resettable_phase_exit, NULL, type);

    if (--s->count == 0) {
        if (rc->phases.exit) rc->phases.exit(obj, type);
    }
    s->exit_phase_in_progress = false;
}
```

### 1.5 遍历顺序

| 阶段 | 遍历顺序 | 原因 |
|------|---------|------|
| Enter | 先父后子 | 父设备先进 reset，子设备随后 |
| Hold | **先子后父** | 子设备先产生副作用，父设备看到稳定状态 |
| Exit | 先父后子 | 父设备先离开 reset |

### 1.6 嵌套 Reset (count 机制)

```
外部 reset A: count 0→1, enter+hold 执行
内部 reset B: count 1→2, enter 不执行 (action_needed=false)
释放 B:     count 2→1, exit 不执行
释放 A:     count 1→0, exit 执行   ← 真正离开 reset
```

只有 count 从 0→1 时执行 enter，从 1→0 时执行 exit，实现嵌套 reset 的正确语义。

### 1.7 旧 API 兼容

```c
// hw/core/qdev.c:799-815
void device_class_set_legacy_reset(DeviceClass *dc, DeviceReset reset) {
    // 将旧的单阶段 reset 映射为 hold 阶段
    ResettableClass *rc = RESETTABLE_CLASS(dc);
    rc->phases.hold = legacy_reset_hold_wrapper;
    dc->legacy_reset = reset;
}
```

新设备应直接实现 `rc->phases.enter/hold/exit`，不再使用 `DeviceClass::reset`。

### 1.8 设备模型集成

```c
// hw/core/qdev.c:248-263
static void device_class_init(ObjectClass *class, const void *data) {
    ResettableClass *rc = RESETTABLE_CLASS(class);
    rc->get_state = device_get_reset_state;         // → &dev->reset
    rc->child_foreach = device_reset_child_foreach; // 遍历 dev->child_bus
}
```

设备的 reset 子对象 = 设备拥有的所有总线上的设备，形成树形遍历。

---

## 2. Visitor 模式与属性访问

### 2.1 Visitor 体系

```
                    Visitor (抽象基类)
                    include/qapi/visitor.h
                        │
            ┌───────────┼────────────┐
            │           │            │
     Input Visitor  Output Visitor  Clone Visitor
     (读取属性值)   (输出属性值)     (深拷贝)
            │           │
    ┌───────┼───┐   ┌───┼───────┐
    │           │   │           │
  String    QObject  String   QObject
  Input     Input    Output   Output
```

### 2.2 核心 Visitor 类型

| Visitor | 用途 | 数据源/目标 |
|---------|------|------------|
| `StringInputVisitor` | 解析字符串为值 | `-device foo,prop=value` |
| `QObjectInputVisitor` | 解析 QDict/QList | QMP JSON 命令 |
| `StringOutputVisitor` | 值转换为字符串 | `info qtree` 显示 |
| `QObjectOutputVisitor` | 值序列化为 QObject | QMP 响应 |
| `OptsVisitor` | 解析 QemuOpts | `-object`/`-netdev` 参数 |
| `CloneVisitor` | 深拷贝 QAPI 结构 | 内部使用 |

### 2.3 属性读写流程

```c
// qom/object.c:1355-1392
bool object_property_get(Object *obj, const char *name, Visitor *v, Error **errp) {
    ObjectProperty *prop = object_property_find_err(obj, name, errp);
    if (!prop->get) { error_setg(errp, "不可读"); return false; }
    prop->get(obj, v, name, prop->opaque, errp);  // 调用属性的 getter
    return true;
}

bool object_property_set(Object *obj, const char *name, Visitor *v, Error **errp) {
    ObjectProperty *prop = object_property_find_err(obj, name, errp);
    if (!prop->set) { error_setg(errp, "不可写"); return false; }
    prop->set(obj, v, name, prop->opaque, errp);  // 调用属性的 setter
    return true;
}
```

### 2.4 便捷包装函数

```c
// qom/object.c:1394-1519 (概念)
bool object_property_set_str(Object *obj, const char *name, const char *value, ...) {
    // 创建 QString → QObjectInputVisitor → object_property_set()
}

bool object_property_set_int(Object *obj, const char *name, int64_t value, ...) {
    // 创建 QNum → QObjectInputVisitor → object_property_set()
}

// 字符串解析:
bool object_property_parse(Object *obj, const char *name, const char *string, ...) {
    Visitor *v = string_input_visitor_new(string);
    object_property_set(obj, name, v, errp);
    visit_free(v);
}
```

### 2.5 QObject 桥接

```c
// qom/qom-qobject.c:20-45
bool object_property_set_qobject(Object *obj, const char *name, QObject *value, ...) {
    Visitor *v = qobject_input_visitor_new(value);   // JSON → Visitor
    bool ok = object_property_set(obj, name, v, errp);
    visit_free(v);
    return ok;
}

QObject *object_property_get_qobject(Object *obj, const char *name, ...) {
    QObject *ret = NULL;
    Visitor *v = qobject_output_visitor_new(&ret);   // Visitor → JSON
    object_property_get(obj, name, v, errp);
    visit_complete(v, &ret);
    visit_free(v);
    return ret;
}
```

### 2.6 Visitor 在 getter/setter 中的使用

以 uint32 属性为例：

```c
// hw/core/qdev-properties.c (概念)
static void prop_uint32_get(Object *obj, Visitor *v, const char *name, ...) {
    Property *prop = opaque;
    uint32_t *ptr = object_field_prop_ptr(obj, prop);  // obj + offset
    visit_type_uint32(v, name, ptr, errp);             // 通过 visitor 输出值
}

static void prop_uint32_set(Object *obj, Visitor *v, const char *name, ...) {
    Property *prop = opaque;
    uint32_t *ptr = object_field_prop_ptr(obj, prop);
    visit_type_uint32(v, name, ptr, errp);             // 通过 visitor 输入值
}
```

### 2.7 访问路径总结

```
命令行 "-device pl011,baudrate=115200"
  → string "115200"
  → string_input_visitor_new("115200")
  → object_property_set(obj, "baudrate", visitor)
  → prop->set = prop_uint32_set
  → visit_type_uint32(visitor, ..., &dev->baudrate)
  → dev->baudrate = 115200

QMP: {"execute":"qom-set","arguments":{"path":"/machine/peripheral/uart","property":"baudrate","value":115200}}
  → QObject (QNum 115200)
  → qobject_input_visitor_new(qnum)
  → object_property_set(obj, "baudrate", visitor)
  → 同上
```

---

## 3. DEFINE_PROP 宏系统

### 3.1 Property 结构体

```c
// include/hw/core/qdev-properties.h:16-31
struct Property {
    const char *name;            // 属性名（如 "baudrate"）
    const PropertyInfo *info;    // 类型信息（getter/setter）
    ptrdiff_t offset;            // 字段在设备结构体中的偏移
    const char *link_type;       // link 属性的目标类型名
    uint64_t bitmask;            // bit 属性的掩码
    union { int64_t i; uint64_t u; } defval;  // 默认值
    const PropertyInfo *arrayinfo;  // 数组元素类型
    int arrayoffset;             // 数组指针字段偏移
    int arrayfieldsize;          // 数组元素大小
    uint8_t bitnr;               // bit 编号
    bool set_default;            // 是否设置默认值
};
```

### 3.2 PropertyInfo 结构体

```c
// include/hw/core/qdev-properties.h:33-45
struct PropertyInfo {
    const char *type;              // 类型字符串（如 "uint32"）
    const char *description;       // 描述
    const QEnumLookup *enum_table; // 枚举查找表
    bool realized_set_allowed;     // realize 后是否可修改
    char *(*print)(Object *, const Property *);  // 打印函数
    void (*set_default_value)(ObjectProperty *, const Property *);
    ObjectProperty *(*create)(ObjectClass *, const char *, const Property *);
    ObjectPropertyAccessor *get;   // getter
    ObjectPropertyAccessor *set;   // setter
    ObjectPropertyRelease *release; // 释放函数
};
```

### 3.3 DEFINE_PROP 基础宏

```c
// include/hw/core/qdev-properties.h:79-85
#define DEFINE_PROP(_name, _state, _field, _prop, _type, ...) {  \
    .name      = (_name),                                    \
    .info      = &(_prop),                                   \
    .offset    = offsetof(_state, _field)                    \
        + type_check(_type, typeof_field(_state, _field)),   \
    __VA_ARGS__                                              \
}
```

**关键设计**：`type_check` 在编译时验证字段类型匹配，`offsetof` 计算字段偏移。

### 3.4 常用宏展开

```c
// DEFINE_PROP_UINT32("num-irqs", GICState, num_irq, 96)
// 展开为:
{
    .name = "num-irqs",
    .info = &qdev_prop_uint32,
    .offset = offsetof(GICState, num_irq) + type_check(uint32_t, ...),
    .set_default = true,
    .defval.u = 96
}

// DEFINE_PROP_BOOL("has-el3", ARMCPU, has_el3, true)
{
    .name = "has-el3",
    .info = &qdev_prop_bool,
    .offset = offsetof(ARMCPU, has_el3) + type_check(bool, ...),
    .set_default = true,
    .defval.u = true
}

// DEFINE_PROP_STRING("cpu-type", VirtMachineState, cpu_type)
{
    .name = "cpu-type",
    .info = &qdev_prop_string,
    .offset = offsetof(VirtMachineState, cpu_type) + type_check(char*, ...),
}

// DEFINE_PROP_LINK("gic", ..., TYPE_ARM_GICV3, GICv3State *)
{
    .name = "gic",
    .info = &qdev_prop_link,
    .offset = offsetof(...),
    .link_type = TYPE_ARM_GICV3,
}
```

### 3.5 数组属性

```c
// DEFINE_PROP_ARRAY("irqs", MyDevice, num_irqs, irq_array, qdev_prop_uint32, uint32_t)
{
    .name = "irqs",
    .info = &qdev_prop_array,
    .offset = offsetof(MyDevice, num_irqs),        // 长度字段
    .arrayoffset = offsetof(MyDevice, irq_array),  // 数组指针字段
    .arrayinfo = &qdev_prop_uint32,                // 元素类型
    .arrayfieldsize = sizeof(uint32_t),
    .set_default = true,
    .defval.u = 0,
}
```

### 3.6 属性注册流程

```c
// 设备 class_init 中:
static void my_device_class_init(ObjectClass *oc, const void *data) {
    DeviceClass *dc = DEVICE_CLASS(oc);
    device_class_set_props(dc, my_device_properties);
}

// hw/core/qdev-properties.c
void device_class_set_props(DeviceClass *dc, const Property *props) {
    dc->props_ = props;
    dc->props_count_ = count_props(props);
    // 属性在 realize 时通过 qdev_property_add_static() 注册到对象
}
```

### 3.7 默认值设置

```c
// qom/object.c:1521-1538 (概念)
void object_property_init_defval(Object *obj, ObjectProperty *prop) {
    if (prop->init) {
        // 用 QObjectInputVisitor 将 defval 写入字段
        Visitor *v = qobject_input_visitor_new(prop->defval);
        prop->set(obj, v, prop->name, prop->opaque, &error_abort);
        visit_free(v);
    }
}
```

### 3.8 偏移量访问

```c
// hw/core/qdev-properties.c:56-60
static void *object_field_prop_ptr(Object *obj, const Property *prop) {
    void *ptr = obj;
    ptr += prop->offset;  // 基址 + 编译时偏移量
    return ptr;
}
```

---

## 4. QOM-QMP 联动

### 4.1 QMP 命令列表

| QMP 命令 | 功能 | 实现位置 |
|---------|------|---------|
| `qom-list` | 列出对象的所有属性 | `qom-qmp-cmds.c:47-69` |
| `qom-get` | 获取属性值 | `qom-qmp-cmds.c:140-152` |
| `qom-set` | 设置属性值 | `qom-qmp-cmds.c:125-138` |
| `qom-list-types` | 列出所有注册的类型 | `qom-qmp-cmds.c:170-181` |
| `qom-list-get` | 批量获取多个对象属性 | `qom-qmp-cmds.c:105-123` |
| `device-list-properties` | 列出设备类的属性 | `qom-qmp-cmds.c:183+` |

### 4.2 路径解析

```c
// qom/qom-qmp-cmds.c:31-45
static Object *qom_resolve_path(const char *path, Error **errp) {
    bool ambiguous = false;
    Object *obj = object_resolve_path(path, &ambiguous);
    if (obj == NULL) {
        if (ambiguous) error_setg(errp, "Path '%s' is ambiguous", path);
        else error_set(errp, ERROR_CLASS_DEVICE_NOT_FOUND, ...);
    }
    return obj;
}
```

QOM 路径格式：`/machine/peripheral/uart0` — 从根对象开始，通过 child 属性逐级查找。

### 4.3 qom-list 实现

```c
// qom/qom-qmp-cmds.c:47-69
ObjectPropertyInfoList *qmp_qom_list(const char *path, Error **errp) {
    Object *obj = qom_resolve_path(path, errp);

    ObjectPropertyIterator iter;
    object_property_iter_init(&iter, obj);
    while ((prop = object_property_iter_next(&iter))) {
        // 收集 { name, type } 对
        value->name = g_strdup(prop->name);
        value->type = g_strdup(prop->type);
        QAPI_LIST_PREPEND(props, value);
    }
    return props;
}
```

### 4.4 qom-get / qom-set

```c
// qom/qom-qmp-cmds.c:125-152
void qmp_qom_set(const char *path, const char *property, QObject *value, ...) {
    Object *obj = object_resolve_path(path, NULL);
    object_property_set_qobject(obj, property, value, errp);
    //                          ↓
    // qom-qobject.c: qobject_input_visitor_new(value) → prop->set()
}

QObject *qmp_qom_get(const char *path, const char *property, ...) {
    Object *obj = object_resolve_path(path, NULL);
    return object_property_get_qobject(obj, property, errp);
    //     ↓
    // qom-qobject.c: qobject_output_visitor_new(&ret) → prop->get()
}
```

### 4.5 qom-list-types

```c
// qom/qom-qmp-cmds.c:170-181
ObjectTypeInfoList *qmp_qom_list_types(const char *implements, ...) {
    module_load_qom_all();  // 确保所有模块已加载
    object_class_foreach(qom_list_types_tramp, implements, abstract, &ret);
    // 遍历全局类型表，过滤实现了特定接口的类型
    return ret;
}
```

### 4.6 使用示例

```bash
# 列出 machine 对象的属性
{ "execute": "qom-list", "arguments": { "path": "/machine" } }

# 获取 CPU 类型
{ "execute": "qom-get", "arguments": { "path": "/machine", "property": "cpu-type" } }

# 设置 vCPU 数量
{ "execute": "qom-set", "arguments": { "path": "/machine/smp", "property": "cpus", "value": 4 } }

# 列出所有设备类型
{ "execute": "qom-list-types", "arguments": { "implements": "device" } }

# 查询设备类属性
{ "execute": "device-list-properties", "arguments": { "typename": "virtio-net-pci" } }
```

---

## 5. 综合流程：属性从命令行到设备

### 5.1 完整路径

```
命令行: -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:56

解析阶段:
  qemu_opts_parse() → QemuOpts { "netdev"="net0", "mac"="52:54:00:12:34:56" }

设备创建:
  qdev_device_add()
    → object_new("virtio-net-pci")
       → instance_init 链
       → 默认值通过 DEFINE_PROP 设置

属性设置:
  对每个 QemuOpts 选项:
    object_property_parse(dev, "netdev", "net0")
      → string_input_visitor_new("net0")
      → prop->set(dev, visitor, "netdev", ...)
        → (link 属性) 解析为对象引用

    object_property_parse(dev, "mac", "52:54:00:12:34:56")
      → string_input_visitor_new("52:54:00:12:34:56")
      → prop->set(dev, visitor, "mac", ...)
        → (MACAddress 属性) 解析为 6 字节 MAC

Realize:
  qdev_realize(dev, bus, errp)
    → device_set_realized(obj, true, errp)
      → dc->realize(dev, errp)
      → 属性不再可修改 (除非 realized_set_allowed)
```

### 5.2 QMP 动态修改

```
运行时: {"execute":"qom-set","arguments":{"path":"/machine/peripheral/net0","property":"rx-queue-size","value":1024}}

  → qmp_qom_set()
  → object_resolve_path("/machine/peripheral/net0")
  → object_property_set_qobject(dev, "rx-queue-size", QNum(1024))
  → qobject_input_visitor_new(QNum(1024))
  → prop->set(dev, visitor, "rx-queue-size", ...)
  → 检查 realized_set_allowed
  → 写入 dev->rx_queue_size = 1024
```

---

## 附录 A: Resettable 迁移指南

### 旧 API（已弃用）

```c
static void my_device_reset(DeviceState *dev) {
    MyDevice *s = MY_DEVICE(dev);
    s->reg = 0;
    qemu_irq_lower(s->irq);  // ⚠️ 可能影响未 reset 的设备
}

static void my_device_class_init(ObjectClass *oc, ...) {
    DeviceClass *dc = DEVICE_CLASS(oc);
    dc->reset = my_device_reset;  // 旧 API
}
```

### 新 API（推荐）

```c
static void my_device_reset_enter(Object *obj, ResetType type) {
    MyDevice *s = MY_DEVICE(obj);
    s->reg = 0;  // 仅本地状态
    // 不要 qemu_irq_lower！
}

static void my_device_reset_hold(Object *obj, ResetType type) {
    MyDevice *s = MY_DEVICE(obj);
    qemu_irq_lower(s->irq);  // 这里可以产生副作用
}

static void my_device_reset_exit(Object *obj, ResetType type) {
    // 可选：离开 reset 时的动作
}

static void my_device_class_init(ObjectClass *oc, ...) {
    ResettableClass *rc = RESETTABLE_CLASS(oc);
    rc->phases.enter = my_device_reset_enter;
    rc->phases.hold = my_device_reset_hold;
    rc->phases.exit = my_device_reset_exit;
}
```

---

## 附录 B: Visitor API 速查

| 函数 | 方向 | 用途 |
|------|------|------|
| `visit_type_uint32(v, name, &val, errp)` | 双向 | 读/写 uint32 |
| `visit_type_str(v, name, &str, errp)` | 双向 | 读/写字符串 |
| `visit_type_bool(v, name, &b, errp)` | 双向 | 读/写布尔 |
| `visit_start_struct(v, name, &obj, size, errp)` | 双向 | 开始结构体 |
| `visit_end_struct(v, &obj)` | 双向 | 结束结构体 |
| `visit_start_list(v, name, &list, size, errp)` | 双向 | 开始列表 |
| `visit_next_list(v, &prev, size)` | 双向 | 下一个元素 |
| `visit_end_list(v, &list)` | 双向 | 结束列表 |
| `visit_complete(v, &result)` | 输出 | 获取输出结果 |
| `visit_free(v)` | — | 释放 visitor |

---

## 附录 C: 源码文件索引

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| `include/hw/core/resettable.h` | ~236 | ResettableClass/State 定义、API 声明 |
| `hw/core/resettable.c` | ~242 | 三阶段实现、树遍历 |
| `include/qapi/visitor.h` | ~420 | Visitor 抽象接口 |
| `qapi/qobject-input-visitor.c` | ~260 | QDict/QList → 类型值 |
| `qom/qom-qobject.c` | 46 | set/get_qobject 桥接 |
| `include/hw/core/qdev-properties.h` | ~191 | Property/PropertyInfo + DEFINE_PROP_* |
| `hw/core/qdev-properties.c` | ~1130 | 各类型 getter/setter 实现 |
| `qom/qom-qmp-cmds.c` | ~240 | qom-list/get/set/list-types QMP 命令 |
| `hw/core/qdev.c` | ~815 | Resettable 集成、realize、legacy_reset |
