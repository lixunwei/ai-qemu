# QOM（QEMU Object Model）深度分析

> **版本**: QEMU 11.0.50  
> **分析日期**: 2026-05-08  
> **源码关键文件**: `qom/object.c`、`include/qom/object.h`、`include/qemu/module.h`  
> **关联文档**: `docs/devel/qom.rst`  

---

## 目录

1. [概述](#1-概述)
2. [TypeImpl — 类型内部表示](#2-typeimpl--类型内部表示)
3. [类型注册机制](#3-类型注册机制)
4. [类初始化（class_init）](#4-类初始化class_init)
5. [实例创建（instance_init）](#5-实例创建instance_init)
6. [实例销毁与引用计数](#6-实例销毁与引用计数)
7. [接口系统（Interface）](#7-接口系统interface)
8. [属性系统（Property）](#8-属性系统property)
9. [对象组合树](#9-对象组合树)
10. [类型转换（Cast）](#10-类型转换cast)
11. [ARM64 具体继承示例](#11-arm64-具体继承示例)
12. [关键设计总结](#12-关键设计总结)

---

## 1. 概述

QOM 是 QEMU 的面向对象基础设施，为纯 C 代码提供了完整的 OOP 能力：

- **单继承** — 每个类型有且仅有一个父类型
- **多接口** — 类型可以实现多个接口（类似 Java Interface）
- **惰性初始化** — 类对象在首次使用时才初始化
- **属性系统** — 支持运行时可发现、可配置的属性（类型安全，Visitor 模式）
- **对象组合树** — 通过 child/link 属性建立对象层级关系
- **引用计数** — 自动内存管理

QOM 中万物皆对象：CPU、设备、总线、机器、加速器、后端……全部基于 QOM 构建。

---

## 2. TypeImpl — 类型内部表示

`TypeImpl` 是 QOM 类型系统的核心内部结构，**不对外暴露**，仅在 `qom/object.c:48-75` 中定义：

```c
struct TypeImpl {
    const char *name;               // 类型名（全局唯一，如 "pl011"）
    size_t class_size;              // 类对象大小（ObjectClass 子类）
    size_t instance_size;           // 实例对象大小（Object 子类）
    size_t instance_align;          // 实例内存对齐

    // 回调函数
    void (*class_init)(...);        // 类初始化回调
    void (*class_base_init)(...);   // 基类初始化回调（每层继承都调用）
    const void *class_data;         // 传给 class_init 的数据

    void (*instance_init)(Object *obj);       // 实例构造
    void (*instance_post_init)(Object *obj);  // 实例后构造
    void (*instance_finalize)(Object *obj);   // 实例析构

    bool abstract;                  // 是否抽象类型
    const char *parent;             // 父类型名
    TypeImpl *parent_type;          // 父类型指针（解析后）

    ObjectClass *class;             // 类对象（惰性创建）

    int num_interfaces;             // 接口数量
    InterfaceImpl interfaces[MAX_INTERFACES];  // 接口列表
};
```

### 全局类型表

| 组件 | 位置 | 说明 |
|------|------|------|
| `type_table_get()` | object.c:79-88 | 获取/创建全局 GHashTable（类型名 → TypeImpl*） |
| `type_table_lookup()` | object.c:98-101 | 按名查找 TypeImpl |
| `type_register_internal()` | object.c:163-176 | 核心注册：验证名称、分配 TypeImpl、插入哈希表 |

---

## 3. 类型注册机制

### 3.1 注册 API

对外暴露的 API 只有一个：

```c
// include/qom/object.h:877-882
TypeImpl *type_register_static(const TypeInfo *info);
```

调用链：`type_register_static()` → `type_register_internal()`

> 当前代码树中 **没有** 独立的 `type_register()` 公开符号，所有注册统一走 `type_register_static()`。

### 3.2 自动注册机制

注册通过 GCC 的 `__attribute__((constructor))` 实现自动化：

```
type_init(fn)                              [include/qemu/module.h:56]
  └─ module_init(fn, MODULE_INIT_QOM)      [module.h:35]
       └─ __attribute__((constructor))      # 生成构造函数
            └─ register_module_init(fn)     [util/module.c:70-113]
                 └─ 追加到 per-type 尾队列
```

**执行时序**：

1. 编译器为每个 `type_init()` 生成一个 constructor 函数
2. 程序启动时（`main()` 之前），所有 constructor 自动执行
3. 每个 constructor 调用 `register_module_init()`，将回调注册到 `MODULE_INIT_QOM` 队列
4. `main()` 中调用 `module_call_init(MODULE_INIT_QOM)` 逐个执行回调
5. 每个回调调用 `type_register_static()`，将 TypeInfo 注册到全局类型表

> **DSO 模块**: 对于动态加载的模块，使用 `register_dso_module_init()`，
> 在 `module_load_dso()` 时执行 DSO 的 init，然后合并到全局列表。
> （`util/module.c:157-199`）

### 3.3 TypeInfo — 用户填写的注册信息

```c
// include/qom/object.h:435-495
struct TypeInfo {
    const char *name;                    // 类型名
    const char *parent;                  // 父类型名

    size_t instance_size;                // 实例大小
    size_t instance_align;               // 实例对齐
    void (*instance_init)(Object *obj);  // 实例构造回调
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    size_t class_size;                   // 类大小
    void (*class_init)(ObjectClass *, const void *);
    void (*class_base_init)(ObjectClass *, const void *);
    const void *class_data;

    bool abstract;                       // 是否抽象
    const InterfaceInfo *interfaces;     // 实现的接口列表
};
```

---

## 4. 类初始化（class_init）

### 4.1 惰性初始化

类对象（`ObjectClass`）在 **首次使用时** 才创建，入口为 `type_initialize()`
（`qom/object.c:336-419`）：

```
type_initialize(TypeImpl *ti)
  │
  ├─ if ti->class 已存在 → 直接返回（惰性守卫）     [340-342]
  │
  ├─ 计算继承尺寸（class_size/instance_size）        [344-346]
  │
  ├─ 自动标记抽象（instance_size==0 的类型）          [347-352]
  │
  ├─ 接口类型强制 abstract + 无实例钩子               [353-360]
  │
  ├─ 分配 ObjectClass 内存                           [361]
  │
  ├─ 递归初始化父类型                                [363-365]
  │   └─ type_initialize(ti->parent_type)  ← 递归！
  │
  ├─ 拷贝父类的 class 内存（memcpy）                 [371]
  │   # 这就是"继承"的实现：子类从父类拷贝所有虚函数指针
  │
  ├─ 继承父类的接口列表                              [374-379]
  │
  ├─ 添加本类型直接声明的接口                        [381-401]
  │   └─ type_initialize_interface()  （见第7节）
  │
  ├─ 创建类属性哈希表                                [404-405]
  │
  ├─ 从上到下调用 class_base_init                    [409-414]
  │   # 对从根到当前的每一层，调用其 class_base_init
  │   # 用于每层重新初始化被 memcpy 覆盖的字段
  │
  └─ 调用本类型的 class_init                         [416-418]
      # 子类在这里覆写虚函数（如 DeviceClass.realize）
```

### 4.2 class_init vs class_base_init

| 回调 | 调用时机 | 调用次数 | 用途 |
|------|---------|---------|------|
| `class_init` | 仅在本类型初始化时 | 1 次 | 设置虚函数、注册属性、设置默认值 |
| `class_base_init` | 每次有子类初始化时 | 每层 1 次 | 修正 memcpy 带来的问题（如深拷贝指针字段） |

### 4.3 class_init 链示例（ARM64 virt 机器）

```
1. Object.class_init          → 添加 "type" 属性              [object.c:2836-2840]
2. DeviceState.class_init     → 设置 hotpluggable、resettable  [qdev.c:754-790]
                                 vmstate hooks、realized 属性
3. MachineState.class_init    → 机器通用设置                   [hw/core/machine.c:1043]
4. virt_machine_class_init    → 设置 mc->init=machvirt_init    [virt.c:3820-4056]
                                 max_cpus、热插拔处理器
                                 secure/virt/highmem/GIC 属性
```

---

## 5. 实例创建（instance_init）

### 5.1 创建流程

```
object_new("pl011")                        [object.c:725-730]
  └─ type_get_or_load_by_name("pl011")     # 查找/加载类型
  └─ object_new_with_type(type)            [object.c:690-718]
       ├─ type_initialize(type)            # 确保类已初始化
       ├─ g_malloc(instance_size)          # 分配实例内存
       └─ object_initialize_with_type()    [object.c:496-512]
            ├─ 检查大小、非抽象
            ├─ memset(obj, 0, size)        # 清零
            ├─ obj->class = type->class    # 关联类对象
            ├─ object_ref(obj)             # 初始引用计数 = 1
            ├─ object_class_property_init_all()  # 运行类属性初始化器
            ├─ 创建实例属性哈希表
            ├─ object_init_with_type()     # 调用 instance_init 链
            └─ object_post_init_with_type()  # 调用 instance_post_init 链
```

### 5.2 instance_init 调用顺序（父优先）

`object_init_with_type()` (`object.c:421-430`)：

```c
static void object_init_with_type(Object *obj, TypeImpl *ti)
{
    if (type_has_parent(ti)) {
        object_init_with_type(obj, type_get_parent(ti));  // 先递归父类
    }
    if (ti->instance_init) {
        ti->instance_init(obj);  // 再调用当前类
    }
}
```

**调用顺序**：`Object.init → DeviceState.init → SysBusDevice.init → PL011.init`

### 5.3 instance_post_init 调用顺序

`object_post_init_with_type()` (`object.c:432-440`) 同样递归，但在所有 init 完成之后调用。

> **佐证 commit**: `220c739903` "qom: reverse order of instance_post_init calls" —
> 说明 post_init 的调用顺序曾被调整过，当前为父优先顺序。

### 5.4 realize — 设备实现

`instance_init` 完成后，对象处于"已构造但未实现"状态。需要显式调用 `qdev_realize()` 
（`hw/core/qdev.c:265-292`）来触发 `DeviceClass.realize` 回调：

```
object_new("pl011")        → 分配 + instance_init 链
qdev_realize(dev, bus)      → 调用 DeviceClass.realize（= pl011_realize）
                              连接到总线、映射 MMIO/IRQ
```

**设计意图**：分离构造和实现，允许在 realize 之前配置属性。

---

## 6. 实例销毁与引用计数

### 6.1 引用计数

| API | 位置 | 说明 |
|-----|------|------|
| `object_ref(obj)` | object.c:1148-1160 | 递增引用计数 |
| `object_unref(obj)` | object.c:1162-1174 | 递减引用计数，归零时触发销毁 |

### 6.2 销毁流程

```
object_unref(obj)           # ref 降为 0
  └─ object_finalize(obj)   [object.c:663-676]
       ├─ object_property_del_all(obj)   # 删除所有实例属性
       ├─ object_deinit(obj, ti)         # 调用 instance_finalize 链
       │    ├─ ti->instance_finalize(obj)  # 当前类的析构
       │    └─ object_deinit(obj, parent)  # 递归到父类
       ├─ assert(ref == 0 && parent == NULL)
       └─ obj->free(obj)                # 释放内存
```

**析构顺序**：与构造相反 — **子类优先**（先调用子类的 `instance_finalize`，再递归父类）。

---

## 7. 接口系统（Interface）

### 7.1 核心定义

```c
// include/qom/object.h:562-587
typedef struct InterfaceInfo {
    const char *type;              // 接口类型名
} InterfaceInfo;

typedef struct InterfaceClass {
    ObjectClass parent_class;      // 继承 ObjectClass
    Type interface_type;           // 关联的接口 TypeImpl
} InterfaceClass;

#define TYPE_INTERFACE  "interface"  // 所有接口的根类型
```

### 7.2 接口声明

在 `TypeInfo` 中通过 `interfaces` 数组声明：

```c
static const TypeInfo virt_machine_info = {
    .name = TYPE_VIRT_MACHINE,
    .parent = TYPE_MACHINE,
    .interfaces = (const InterfaceInfo[]) {
        { TYPE_HOTPLUG_HANDLER },     // 实现热插拔接口
        { }                           // 哨兵终止
    },
    ...
};
```

### 7.3 接口类型合成

`type_initialize_interface()` (`object.c:300-320`) 在类初始化时合成接口类型：

```
type_initialize_interface(ti, iface_impl)
  ├─ 创建合成类型名: "<type>::<interface>"
  │   例如: "virt-machine::hotplug-handler"
  ├─ 设置 parent = 父类的同名接口（如果有）
  ├─ 调用 type_initialize() 初始化合成类型
  ├─ 设置 InterfaceClass.interface_type
  └─ 将 InterfaceClass 追加到 ti->class->interfaces 链表
```

### 7.4 接口继承

在 `type_initialize()` 中 (`object.c:374-401`)：

1. **继承父类接口** — 将父类的 `interfaces` 列表拷贝到子类
2. **添加直接接口** — 将本类 TypeInfo 中声明的接口追加

因此子类自动继承父类的所有接口实现。

### 7.5 接口动态转换

`object_class_dynamic_cast()` (`object.c:883-928`)：

```
object_class_dynamic_cast(class, target_typename)
  ├─ 快速路径：类型名完全匹配 → 直接返回        [896-898]
  ├─ 如果目标是接口类型：
  │   └─ 遍历 class->interfaces 链表             [906-923]
  │       └─ 检查接口的 type 是否是 target 的祖先
  │       └─ 拒绝歧义（found > 1）
  └─ 否则：普通祖先检查                          [924-926]
```

### 7.6 常见接口示例

| 接口 | TYPE 名 | 头文件 | 用途 |
|------|---------|--------|------|
| `Resettable` | `TYPE_RESETTABLE_INTERFACE` | `include/hw/core/resettable.h:18-23` | 三阶段复位协议 |
| `HotplugHandler` | `TYPE_HOTPLUG_HANDLER` | `include/hw/core/hotplug.h:17-24` | 设备热插拔处理 |
| `FWPathProvider` | `TYPE_FW_PATH_PROVIDER` | `include/hw/core/fw-path-provider.h:23-37` | 固件路径提供 |
| `VMStateIf` | `TYPE_VMSTATE_IF` | — | 迁移状态接口 |

---

## 8. 属性系统（Property）

### 8.1 ObjectProperty 结构

```c
// include/qom/object.h:89-101
typedef struct ObjectProperty {
    char *name;           // 属性名
    char *type;           // 属性类型字符串（如 "bool"、"uint32"、"link<pl011>"）
    char *description;    // 描述
    ObjectPropertyAccessor *get;     // getter
    ObjectPropertyAccessor *set;     // setter
    ObjectPropertyResolve *resolve;  // 路径解析（child/link 用）
    ObjectPropertyRelease *release;  // 释放回调
    ObjectPropertyInit *init;        // 初始化回调
    void *opaque;         // 用户数据
    QObject *defval;      // 默认值
} ObjectProperty;
```

### 8.2 类属性 vs 实例属性

| 属性类型 | 存储位置 | API | 特点 |
|---------|---------|-----|------|
| **类属性** | `ObjectClass.properties` (GHashTable) | `object_class_property_add()` | 同类所有实例共享定义 |
| **实例属性** | `Object.properties` (GHashTable) | `object_property_add()` | 每个实例独立 |

**查找优先级** (`object_property_find()`, object.c:1266-1277)：
1. 先查类属性（含父类链）
2. 再查实例属性

**类属性查找** (`object_class_property_find()`, object.c:1317-1331)：
递归向上遍历父类链。

### 8.3 属性的添加

```c
// object.c:1176-1236
ObjectProperty *object_property_try_add(Object *obj, const char *name, ...)
{
    // 1. 重名检查
    if (object_property_find(obj, name)) → 报错

    // 2. 创建 ObjectProperty 并存入 obj->properties
    prop = g_new0(ObjectProperty, 1);
    prop->name = g_strdup(name);
    prop->type = g_strdup(type);
    prop->get = get;
    prop->set = set;
    ...
    g_hash_table_insert(obj->properties, prop->name, prop);
    return prop;
}
```

### 8.4 属性的读写（Visitor 模式）

```
object_property_get(obj, name, visitor)     [object.c:1355-1373]
  └─ prop->get(obj, visitor, name, prop->opaque, &err)

object_property_set(obj, name, visitor)     [object.c:1375-1392]
  └─ prop->set(obj, visitor, name, prop->opaque, &err)
```

**字符串解析**（命令行 `-device` 参数）：

```
object_property_parse(obj, name, string)    [object.c:1634-1642]
  ├─ visitor = string_input_visitor_new(string)
  └─ object_property_set(obj, name, visitor)
```

**QMP/QDict 设置**：

```
object_set_properties_from_qdict()          [qom/object_interfaces.c:47-63]
  ├─ visit_start_struct
  ├─ 逐 key 调用 object_property_set
  └─ visit_end_struct
```

### 8.5 Child 属性（对象组合）

```c
// object.c:1762-1783
object_property_add_child(Object *parent, const char *name, Object *child)
  ├─ 属性类型: "child<typename>"
  ├─ resolve: 返回子对象指针
  ├─ child->parent = parent
  └─ object_ref(child)           // 强引用
```

**释放时** (`object.c:1750-1760`)：
- 调用 `object_unparent()` → `object_property_del_child()`
- 清除 `child->parent`
- `object_unref(child)`

### 8.6 Link 属性（对象关联）

```c
// object.c:1971-1979
object_property_add_link(Object *obj, const char *name,
                         const char *type, Object **targetp, ...)
  ├─ 属性类型: "link<typename>"
  ├─ set: 将路径字符串解析为目标对象指针
  └─ 强引用模式下会 ref/unref 目标
```

与 child 的区别：link 是**引用关系**（不拥有），child 是**包含关系**（拥有）。

---

## 9. 对象组合树

### 9.1 根容器

`object_root_initialize()` (`object.c:1672-1693`) 创建全局根对象和默认容器：

```
/                           ← object_get_root()
├── /audiodevs              ← 音频设备
├── /chardevs               ← 字符设备
├── /objects                ← object_get_objects_root()，用户创建的对象
├── /backend                ← 后端对象
└── /machine                ← MachineState 实例
     ├── /peripheral        ← 外设
     ├── /unattached        ← 未挂载设备
     └── ...
```

### 9.2 路径解析

| API | 位置 | 说明 |
|-----|------|------|
| `object_resolve_path(path)` | object.c:2174-2177 | 绝对路径解析 |
| `object_resolve_path_component(parent, part)` | object.c:2073-2085 | 解析一个路径分量 |
| `object_resolve_partial_path()` | object.c:2109-2144 | 从根开始的模糊匹配 |

路径解析依赖属性的 `resolve` 回调 — child 属性返回子对象，link 属性返回链接目标。

---

## 10. 类型转换（Cast）

### 10.1 声明宏

```c
// include/qom/object.h:234-259

OBJECT_DECLARE_TYPE(InstanceType, ClassType, MODULE_OBJ_NAME)
  → typedef InstanceType ...
  → typedef ClassType ...
  → G_DEFINE_AUTOPTR_CLEANUP_FUNC(...)
  → DECLARE_OBJ_CHECKERS(...)

OBJECT_DECLARE_SIMPLE_TYPE(InstanceType, MODULE_OBJ_NAME)
  → 同上，但不需要单独的 ClassType 结构
```

### 10.2 转换宏

| 宏 | 展开 | 用途 |
|----|------|------|
| `OBJECT_CHECK(type, obj, name)` | `object_dynamic_cast_assert(...)` | 实例向下转换 |
| `OBJECT_CLASS_CHECK(type, cls, name)` | `object_class_dynamic_cast_assert(...)` | 类向下转换 |
| `OBJECT_GET_CLASS(type, obj, name)` | 从实例获取类指针并转换 | 获取类对象 |
| `INTERFACE_CHECK(type, obj, name)` | 接口转换断言 | 转换到接口类型 |

位置：`include/qom/object.h:530-609`

### 10.3 运行时转换

```c
// object.c:835-842
Object *object_dynamic_cast(Object *obj, const char *typename)
{
    if (object_class_dynamic_cast(obj->class, typename)) {
        return obj;   // 转换成功
    }
    return NULL;      // 转换失败
}
```

`object_class_dynamic_cast()` (`object.c:883-928`)：
1. **快速路径**: 类型名匹配 → 直接返回
2. **接口检查**: 遍历 `class->interfaces`，检查是否有匹配的接口类型
3. **祖先检查**: 沿继承链向上查找

assert 版本（`object_dynamic_cast_assert`）在转换失败时直接 abort，
并支持 **cast cache** 加速重复转换 (`object.c:844-880`)。

---

## 11. ARM64 具体继承示例

### 11.1 类型继承关系图

```
                            Object
                              │
                        ┌─────┴──────┐
                   DeviceState    AccelState
                   │    │    │        │    │
              ┌────┘    │    └──┐   KVM   TCG
         SysBusDevice  CPUState  MachineState
              │         │              │
         ┌────┼────┐  ARMCPU    VirtMachineState
         │    │    │
       PL011 PL031 GICv3Common
                      │
                    GICv3
```

### 11.2 VirtMachineState 完整继承链

```
Object                              [qom/object.c:2836-2855]
  └─ DeviceState                    [hw/core/qdev.c:874-890]
       接口: Resettable, VMStateIf
       └─ MachineState              [include/hw/core/boards.h:23-26]
            └─ VirtMachineState     [hw/arm/virt.c:4067-4079]
                 接口: HotplugHandler (直接声明)
                 继承接口: Resettable, VMStateIf (来自 DeviceState)
```

**TypeInfo 定义** (`hw/arm/virt.c:4067-4079`):

```c
static const TypeInfo virt_machine_info = {
    .name          = TYPE_VIRT_MACHINE,
    .parent        = TYPE_MACHINE,
    .instance_size = sizeof(VirtMachineState),
    .class_size    = sizeof(VirtMachineClass),
    .class_init    = virt_machine_class_init,
    .interfaces    = (const InterfaceInfo[]) {
        { TYPE_HOTPLUG_HANDLER },
        { }
    },
};
```

### 11.3 ARMCPU 继承链

```
Object → DeviceState → CPUState → ARMCPU
```

**TypeInfo** (`target/arm/cpu.c:2573-2583`):
- `instance_init`: ARM CPU 初始化
- `class_init`: 设置 CPU 类钩子

### 11.4 PL011 — 完整生命周期示例

**注册**:
```
type_init(pl011_register_types)              [pl011.c:723-729]
  └─ pl011_register_types()
       └─ type_register_static(&pl011_arm_info)
            TypeInfo {
                .name = TYPE_PL011,
                .parent = TYPE_SYS_BUS_DEVICE,
                .instance_init = pl011_init,
                .class_init = pl011_class_init,
            }                                [pl011.c:702-708]
```

**class_init 链**:
```
1. Object.class_init          → "type" 属性
2. DeviceState.class_init     → hotpluggable、resettable、realized
3. SysBusDevice.class_init    → 系统总线默认行为
4. pl011_class_init           → dc->realize = pl011_realize
                                 vmsd、properties        [pl011.c:692-700]
```

**instance_init 链**（创建 `object_new("pl011")` 时）:
```
1. Object.init         → 基础初始化
2. DeviceState.init    → 设备基础初始化
3. SysBusDevice.init   → 系统总线设备初始化
4. pl011_init          → 初始化 MMIO 区域
                          sysbus_init_mmio()
                          sysbus_init_irq()
                          时钟连接              [pl011.c:645-661]
```

**realize**（调用 `qdev_realize()` 时）:
```
pl011_realize()                              [pl011.c:663-669]
  └─ 安装 chardev 处理函数
```

**销毁**（instance_finalize 链）:
```
4. PL011.finalize      → PL011 特定清理（如果有）
3. SysBusDevice.finalize
2. DeviceState.finalize
1. Object.finalize
```

### 11.5 GICv3 继承链

```
Object → DeviceState → SysBusDevice → GICv3Common → GICv3
```

- `GICv3Common` (`hw/intc/arm_gicv3_common.c:638-645`): 通用 GICv3 基类，包含状态/迁移
- `GICv3` (`hw/intc/arm_gicv3.c:465-475`): 具体实现，覆写 realize

---

## 12. 关键设计总结

### 12.1 设计模式

| 模式 | QOM 实现 | 说明 |
|------|---------|------|
| **模板方法** | class_init + 虚函数覆写 | 父类定义框架，子类覆写行为 |
| **工厂方法** | `object_new(typename)` | 按类型名创建对象 |
| **观察者** | 属性 notify（间接） | 属性变更通知 |
| **组合模式** | child 属性 + 对象树 | 树状对象层级 |
| **访问者** | Visitor 模式读写属性 | 支持 string/QObject/JSON 多种访问 |
| **策略模式** | 接口 | 同一对象可通过不同接口访问不同能力 |

### 12.2 关键约束

1. **单继承** — 每个类型只有一个 parent
2. **类型名全局唯一** — 类型名是哈希表的 key
3. **realize 之后不可改属性** — `qdev-properties.c` 中有检查
4. **接口无实例** — 接口类型强制 abstract，不能创建实例
5. **惰性类初始化** — 类对象在首次 `type_initialize()` 时才创建
6. **构造与实现分离** — `instance_init` ≠ `realize`，中间可以配置属性

### 12.3 与传统 OOP 的对照

| 概念 | C++ / Java | QOM |
|------|-----------|-----|
| 类定义 | `class Foo` | `TypeInfo + class_init` |
| 虚函数 | `virtual` | `ObjectClass` 中的函数指针 |
| 构造函数 | `Foo()` | `instance_init` |
| 析构函数 | `~Foo()` | `instance_finalize` |
| 接口 | `interface` / 纯虚类 | `InterfaceInfo[]` |
| `dynamic_cast` | `dynamic_cast<T>` | `object_dynamic_cast()` |
| 属性 | getter/setter | `ObjectProperty + Visitor` |
| 对象树 | 成员变量 | child/link 属性 |
| 引用计数 | `shared_ptr` | `object_ref/unref` |
| RTTI | `typeid` | `object_get_typename()` |

---

## 附录：相关 Git Commit

| Commit | 说明 |
|--------|------|
| `150072398f` | 将兼容属性 API 限制为仅系统模拟可用 |
| `da36151de1` | 将 GlobalProperty 声明移到 `qom/compat-properties.h` |
| `261c09999b` | 使用 `g_ascii_strcasecmp` 替代 `strcasecmp` |
| `d1000ecae2` | 将 `hw/qdev-core.h` 移动并重命名为 `hw/core/qdev.h` |
| `220c739903` | 调整 `instance_post_init` 调用顺序 |
| `2cd09e47aa` | 将 `InterfaceInfo[]` 标记为 const |
