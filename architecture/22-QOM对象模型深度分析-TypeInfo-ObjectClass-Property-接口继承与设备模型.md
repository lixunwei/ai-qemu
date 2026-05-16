# 22 - QOM 对象模型深度分析 — TypeInfo、ObjectClass、Property、接口继承与设备模型

> **基于 QEMU 11.0.50 源码**，深入分析 QEMU Object Model (QOM) 的完整实现：
> 类型注册与延迟初始化、对象生命周期、类继承链、动态类型转换、
> 属性系统、接口机制、以及 DeviceState/SysBusDevice 设备模型集成。

---

## 目录

1. [TypeInfo 与 TypeImpl：类型定义](#1-typeinfo-与-typeimpl类型定义)
2. [类型注册与全局类型表](#2-类型注册与全局类型表)
3. [type_initialize：类初始化链](#3-type_initialize类初始化链)
4. [Object 生命周期：创建与销毁](#4-object-生命周期创建与销毁)
5. [ObjectClass：类结构与类型转换](#5-objectclass类结构与类型转换)
6. [DECLARE 宏家族：类型安全接口](#6-declare-宏家族类型安全接口)
7. [属性系统：Property](#7-属性系统property)
8. [接口机制：InterfaceClass](#8-接口机制interfaceclass)
9. [动态类型转换：cast 实现](#9-动态类型转换cast-实现)
10. [DeviceState 与 DeviceClass：设备模型](#10-devicestate-与-deviceclass设备模型)
11. [SysBusDevice：系统总线设备](#11-sysbusdevice系统总线设备)
12. [QDev 属性助手：DEFINE_PROP 宏](#12-qdev-属性助手define_prop-宏)
13. [实例分析：ARMCPU 类型链](#13-实例分析armcpu-类型链)
14. [对象路径：QOM 树](#14-对象路径qom-树)

---

## 1. TypeInfo 与 TypeImpl：类型定义

### 1.1 TypeInfo — 用户可见的类型描述

```c
// object.h:476-495
struct TypeInfo {
    const char *name;                    // 类型名，如 "arm-cpu"
    const char *parent;                  // 父类型名，如 "cpu"

    size_t instance_size;                // 对象实例大小
    size_t instance_align;               // 对齐要求
    void (*instance_init)(Object *obj);  // 实例初始化
    void (*instance_post_init)(Object *obj); // 后初始化（所有 init 完成后）
    void (*instance_finalize)(Object *obj);  // 实例析构

    bool abstract;                       // 是否抽象类型
    size_t class_size;                   // 类对象大小

    void (*class_init)(ObjectClass *klass, const void *data);      // 类初始化
    void (*class_base_init)(ObjectClass *klass, const void *data); // 基类初始化
    const void *class_data;              // 传递给 class_init 的数据

    const InterfaceInfo *interfaces;     // 接口列表（NULL 终止）
};
```

### 1.2 TypeImpl — 内部类型实现

```c
// object.c:48-75
struct TypeImpl {
    const char *name;
    size_t class_size;
    size_t instance_size;
    size_t instance_align;

    void (*class_init)(ObjectClass *klass, const void *data);
    void (*class_base_init)(ObjectClass *klass, const void *data);
    const void *class_data;

    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;

    const char *parent;
    TypeImpl *parent_type;               // 解析后的父类型指针

    ObjectClass *class;                  // 已初始化的类对象（懒加载）

    int num_interfaces;
    InterfaceImpl interfaces[MAX_INTERFACES];
};
```

**设计要点**：
- `TypeInfo` 是注册 API，`TypeImpl` 是内部存储
- `class` 字段为 NULL 表示尚未初始化（懒加载）
- `parent_type` 在首次使用时解析（`type_get_parent()`）

---

## 2. 类型注册与全局类型表

### 2.1 type_init 宏

```c
// module.h:56
#define type_init(function) module_init(function, MODULE_INIT_QOM)

// module.h:35-39
#define module_init(function, type)                                      \
static void __attribute__((constructor)) do_qemu_init_##function(void)  \
{                                                                       \
    register_module_init(function, type);                               \
}
```

**`__attribute__((constructor))`** 使函数在 `main()` 之前执行，将注册函数加入模块链表。

### 2.2 全局类型哈希表

```c
// object.c:79-96
static GHashTable *type_table_get(void)
{
    static GHashTable *type_table;  // 单例
    if (type_table == NULL)
        type_table = g_hash_table_new(g_str_hash, g_str_equal);
    return type_table;
}

static void type_table_add(TypeImpl *ti)
{
    g_hash_table_insert(type_table_get(), (void *)ti->name, ti);
}
```

### 2.3 注册流程

```c
// object.c:163-182
static TypeImpl *type_register_internal(const TypeInfo *info)
{
    TypeImpl *ti = type_new(info);    // 复制 TypeInfo → TypeImpl
    type_table_add(ti);               // 插入全局哈希表
    return ti;
}

TypeImpl *type_register_static(const TypeInfo *info)
{
    assert(info->parent);
    return type_register_internal(info);
}
```

### 2.4 模块化类型加载

```c
// object.c:202-222
static TypeImpl *type_get_or_load_by_name(const char *name, Error **errp)
{
    TypeImpl *type = type_get_by_name_noload(name);
#ifdef CONFIG_MODULES
    if (!type) {
        module_load_qom(name, errp);      // 按需加载 .so 模块
        type = type_get_by_name_noload(name);
    }
#endif
    return type;
}
```

**完整注册时序**：
```
程序加载 → __attribute__((constructor)) 执行
  → register_module_init(fn, MODULE_INIT_QOM)
     → 加入全局模块链表

main() → module_call_init(MODULE_INIT_QOM)
  → 遍历链表，调用所有注册函数
    → type_register_static(&type_info)
      → type_new() + type_table_add()
      → 此时 TypeImpl.class == NULL（尚未初始化）
```

---

## 3. type_initialize：类初始化链

```c
// object.c:336-419
static void type_initialize(TypeImpl *ti)
{
    if (ti->class) return;  // 已初始化，跳过（幂等）

    // 1. 计算 class/instance 大小（继承自父类）
    ti->class_size = type_class_get_size(ti);
    ti->instance_size = type_object_get_size(ti);

    // 2. 零 instance_size 隐含 abstract
    if (ti->instance_size == 0) ti->abstract = true;

    // 3. 分配类对象
    ti->class = g_malloc0(ti->class_size);

    // 4. 递归初始化父类
    parent = type_get_parent(ti);
    if (parent) {
        type_initialize(parent);

        // 5. 拷贝父类 class 内容（继承虚函数表）
        memcpy(ti->class, parent->class, parent->class_size);
        ti->class->interfaces = NULL;

        // 6. 继承父类接口
        for (e = parent->class->interfaces; e; e = e->next) {
            type_initialize_interface(ti, iface->interface_type, klass->type);
        }

        // 7. 注册本类新增接口
        for (i = 0; i < ti->num_interfaces; i++) {
            type_initialize_interface(ti, t, t);
        }
    }

    // 8. 创建类属性哈希表
    ti->class->properties = g_hash_table_new_full(...);
    ti->class->type = ti;

    // 9. 沿继承链调用 class_base_init（从根到叶）
    while (parent) {
        if (parent->class_base_init)
            parent->class_base_init(ti->class, ti->class_data);
        parent = type_get_parent(parent);
    }

    // 10. 调用本类 class_init（可覆盖父类虚函数）
    if (ti->class_init)
        ti->class_init(ti->class, ti->class_data);
}
```

**类初始化顺序图**：
```
Object.class_init
  ↓ (memcpy 继承)
Device.class_base_init → Device.class_init
  ↓ (memcpy 继承)
CPU.class_base_init → CPU.class_init
  ↓ (memcpy 继承)
ARMCPU.class_init（可覆盖 realize/reset 等虚函数）
```

---

## 4. Object 生命周期：创建与销毁

### 4.1 object_new — 堆分配创建

```c
// object.c:725-730
Object *object_new(const char *typename)
{
    TypeImpl *ti = type_get_or_load_by_name(typename, &error_fatal);
    return object_new_with_type(ti);
}

// object.c:690-718
static Object *object_new_with_type(Type type)
{
    type_initialize(type);  // 确保类已初始化

    size = type->instance_size;
    align = type->instance_align;

    // 分配内存（对齐或普通）
    if (align <= __alignof__(qemu_max_align_t))
        obj = g_malloc(size);
    else
        obj = qemu_memalign(align, size);

    object_initialize_with_type(obj, size, type);
    obj->free = obj_free;  // 记录释放函数
    return obj;
}
```

### 4.2 object_initialize_with_type — 核心初始化

```c
// object.c:496-512
static void object_initialize_with_type(Object *obj, size_t size, TypeImpl *type)
{
    type_initialize(type);      // 确保类已初始化

    memset(obj, 0, type->instance_size);
    obj->class = type->class;   // 绑定类
    object_ref(obj);             // 引用计数 = 1

    // 初始化类属性中的 init 回调
    object_class_property_init_all(obj);

    // 创建实例属性哈希表
    obj->properties = g_hash_table_new_full(...);

    // 递归调用 instance_init（先父后子）
    object_init_with_type(obj, type);

    // 递归调用 instance_post_init（先父后子）
    object_post_init_with_type(obj, type);
}
```

### 4.3 实例初始化递归链

```c
// object.c:421-430
static void object_init_with_type(Object *obj, TypeImpl *ti)
{
    if (type_has_parent(ti))
        object_init_with_type(obj, type_get_parent(ti));  // 先初始化父类
    if (ti->instance_init)
        ti->instance_init(obj);  // 再初始化自身
}

// object.c:432-440
static void object_post_init_with_type(Object *obj, TypeImpl *ti)
{
    if (type_has_parent(ti))
        object_post_init_with_type(obj, type_get_parent(ti));
    if (ti->instance_post_init)
        ti->instance_post_init(obj);
}
```

### 4.4 引用计数与销毁

```c
// object.c:1148-1174
Object *object_ref(void *objptr)
{
    Object *obj = OBJECT(objptr);
    qatomic_fetch_inc(&obj->ref);  // 原子递增
    return obj;
}

void object_unref(void *objptr)
{
    Object *obj = OBJECT(objptr);
    if (qatomic_fetch_dec(&obj->ref) == 1)  // 降为 0
        object_finalize(obj);                 // 触发析构
}

// object.c:663-676
static void object_finalize(void *data)
{
    Object *obj = data;
    TypeImpl *ti = obj->class->type;

    object_property_del_all(obj);   // 删除所有实例属性
    object_deinit(obj, ti);         // 递归 instance_finalize（先子后父）
    g_assert(obj->ref == 0);
    if (obj->free) obj->free(obj);  // 释放内存
}
```

---

## 5. ObjectClass：类结构与类型转换

### 5.1 ObjectClass 结构

```c
// object.h:128-140
struct ObjectClass {
    Type type;                      // 指回 TypeImpl
    GSList *interfaces;             // 接口类链表

    const char *object_cast_cache[4];  // 实例转换缓存
    const char *class_cast_cache[4];   // 类转换缓存

    ObjectUnparent *unparent;       // 从组合树移除回调
    GHashTable *properties;         // 类属性哈希表
};
```

### 5.2 Object 结构

```c
// object.h:142-168（概要）
struct Object {
    ObjectClass *class;      // 指向类对象
    ObjectFree *free;        // 释放回调
    GHashTable *properties;  // 实例属性
    uint32_t ref;            // 引用计数
    Object *parent;          // QOM 树父对象
};
```

---

## 6. DECLARE 宏家族：类型安全接口

### 6.1 OBJECT_DECLARE_TYPE — 完整声明

```c
// object.h:234-241
#define OBJECT_DECLARE_TYPE(InstanceType, ClassType, MODULE_OBJ_NAME) \
    typedef struct InstanceType InstanceType;                          \
    typedef struct ClassType ClassType;                                \
    G_DEFINE_AUTOPTR_CLEANUP_FUNC(InstanceType, object_unref)         \
    DECLARE_OBJ_CHECKERS(InstanceType, ClassType,                     \
                         MODULE_OBJ_NAME, TYPE_##MODULE_OBJ_NAME)
```

### 6.2 DECLARE_OBJ_CHECKERS — 生成转换函数

```c
// object.h:215-218
#define DECLARE_OBJ_CHECKERS(InstanceType, ClassType, OBJ_NAME, TYPENAME) \
    DECLARE_INSTANCE_CHECKER(InstanceType, OBJ_NAME, TYPENAME)            \
    DECLARE_CLASS_CHECKERS(ClassType, OBJ_NAME, TYPENAME)
```

### 6.3 DECLARE_INSTANCE_CHECKER — 实例转换

```c
// object.h:176-179
#define DECLARE_INSTANCE_CHECKER(InstanceType, OBJ_NAME, TYPENAME) \
    static inline InstanceType *                                    \
    OBJ_NAME(const void *obj)                                      \
    { return OBJECT_CHECK(InstanceType, obj, TYPENAME); }
```

### 6.4 DECLARE_CLASS_CHECKERS — 类转换

```c
// object.h:193-200
#define DECLARE_CLASS_CHECKERS(ClassType, OBJ_NAME, TYPENAME)       \
    static inline ClassType *                                        \
    OBJ_NAME##_GET_CLASS(const void *obj)                           \
    { return OBJECT_GET_CLASS(ClassType, obj, TYPENAME); }          \
    static inline ClassType *                                        \
    OBJ_NAME##_CLASS(const void *klass)                             \
    { return OBJECT_CLASS_CHECK(ClassType, klass, TYPENAME); }
```

### 6.5 OBJECT_DECLARE_SIMPLE_TYPE — 无自定义 Class

```c
// object.h:254-259
#define OBJECT_DECLARE_SIMPLE_TYPE(InstanceType, MODULE_OBJ_NAME) \
    typedef struct InstanceType InstanceType;                      \
    G_DEFINE_AUTOPTR_CLEANUP_FUNC(InstanceType, object_unref)     \
    DECLARE_INSTANCE_CHECKER(InstanceType, MODULE_OBJ_NAME, TYPE_##MODULE_OBJ_NAME)
```

**使用示例**：
```c
// 声明带自定义 Class 的类型（如 DeviceState + DeviceClass）
OBJECT_DECLARE_TYPE(DeviceState, DeviceClass, DEVICE)
// 展开生成：DEVICE(obj), DEVICE_GET_CLASS(obj), DEVICE_CLASS(klass)

// 声明简单类型（无自定义 Class）
OBJECT_DECLARE_SIMPLE_TYPE(SysBusDevice, SYS_BUS_DEVICE)
// 展开生成：SYS_BUS_DEVICE(obj)
```

---

## 7. 属性系统：Property

### 7.1 ObjectProperty 结构

```c
// object.h:89-101
struct ObjectProperty {
    char *name;
    char *type;
    char *description;
    ObjectPropertyAccessor *get;    // getter 回调
    ObjectPropertyAccessor *set;    // setter 回调
    ObjectPropertyResolve *resolve; // 路径解析回调
    ObjectPropertyRelease *release; // 释放回调
    ObjectPropertyInit *init;       // 初始化回调
    void *opaque;                   // 回调上下文
    QObject *defval;                // 默认值
};
```

### 7.2 实例属性 vs 类属性

| 维度 | 实例属性 | 类属性 |
|------|----------|--------|
| 存储位置 | `obj->properties` | `klass->properties` |
| 注册 API | `object_property_add()` | `object_class_property_add()` |
| 生命周期 | 随对象创建/销毁 | 随类初始化，全局共享 |
| 继承 | 不继承 | 通过 `object_class_property_find()` 沿继承链查找 |
| 典型用途 | `child`、`link` | `DEFINE_PROP_*` 设备属性 |

```c
// object.c:1176-1236 — 实例属性添加
ObjectProperty *object_property_try_add(Object *obj, const char *name, ...)
{
    prop = g_malloc0(sizeof(*prop));
    prop->name = g_strdup(name);
    prop->type = g_strdup(type);
    prop->get = get; prop->set = set;
    g_hash_table_insert(obj->properties, prop->name, prop);
    return prop;
}

// object.c:1238-1264 — 类属性添加
ObjectProperty *object_class_property_add(ObjectClass *klass, ...)
```

### 7.3 属性查找（实例优先，类兜底）

```c
// object.c:1266-1277
ObjectProperty *object_property_find(Object *obj, const char *name)
{
    // 先查实例属性
    prop = g_hash_table_lookup(obj->properties, name);
    if (prop) return prop;

    // 再查类属性（含继承链）
    return object_class_property_find(obj->class, name);
}
```

### 7.4 常用属性类型

| API | 类型字符串 | 用途 |
|-----|-----------|------|
| `object_property_add_str` | `"string"` | 字符串属性 |
| `object_property_add_bool` | `"bool"` | 布尔属性 |
| `object_property_add_enum` | 枚举名 | 枚举属性 |
| `object_property_add_uint*` | `"uint8"` 等 | 整数属性 |
| `object_property_add_link` | `"link<TYPE>"` | 对象间引用 |
| `object_property_add_child` | `"child<TYPE>"` | 父子层级关系 |

### 7.5 Link 属性 — 对象间引用

```c
// object.c:1786-1790
ObjectProperty *object_property_add_child(Object *obj, const char *name,
                                          Object *child)
// 建立父子关系，child->parent = obj
// 在 QOM 树中形成层级：/machine/unattached/device[0]

// object.c:1890-1924
ObjectProperty *object_property_add_link(Object *obj, ...)
// 对象间引用（不建立所有权），用于 bus→device 等关系
```

---

## 8. 接口机制：InterfaceClass

### 8.1 接口定义

```c
// object.h:567-569
struct InterfaceInfo {
    const char *type;    // 接口类型名
};

// object.h:582-587
struct InterfaceClass {
    ObjectClass parent_class;     // 继承 ObjectClass
    Type interface_type;          // 接口类型引用
};
```

### 8.2 接口注册

在 `TypeInfo.interfaces` 数组中声明：
```c
static const TypeInfo my_device_info = {
    .name = TYPE_MY_DEVICE,
    .parent = TYPE_DEVICE,
    .interfaces = (InterfaceInfo[]) {
        { TYPE_USER_CREATABLE },
        { TYPE_RESETTABLE_INTERFACE },
        { }  // NULL 终止
    },
};
```

### 8.3 接口初始化

```c
// object.c:300-320
static void type_initialize_interface(TypeImpl *ti, TypeImpl *interface_type,
                                      TypeImpl *parent_type)
{
    // 1. 创建接口类型（名称格式 "device_type::interface_type"）
    info.parent = parent_type->name;
    info.name = g_strdup_printf("%s::%s", ti->name, interface_type->name);
    info.abstract = true;

    iface_impl = type_new(&info);
    type_initialize(iface_impl);

    // 2. 设置接口类型引用
    new_iface = (InterfaceClass *)iface_impl->class;
    new_iface->interface_type = interface_type;

    // 3. 追加到类的接口列表
    ti->class->interfaces = g_slist_append(ti->class->interfaces, new_iface);
}
```

### 8.4 接口转换

```c
// object.h:607-609
#define INTERFACE_CHECK(interface, obj, name) \
    ((interface *)object_dynamic_cast_assert(OBJECT(obj), (name), \
                                             __FILE__, __LINE__, __func__))
```

---

## 9. 动态类型转换：cast 实现

### 9.1 object_dynamic_cast — 实例转换

```c
// object.c:835-842
Object *object_dynamic_cast(Object *obj, const char *typename)
{
    if (obj && object_class_dynamic_cast(object_get_class(obj), typename))
        return obj;
    return NULL;
}
```

### 9.2 object_class_dynamic_cast — 类转换（核心）

```c
// object.c:883-928
ObjectClass *object_class_dynamic_cast(ObjectClass *class, const char *typename)
{
    // 快速路径：指针比较（叶类常命中）
    if (type->name == typename) return class;

    target_type = type_get_by_name_noload(typename);

    // 接口查找路径
    if (target 是接口类型) {
        for (i = class->interfaces; i; i = i->next) {
            if (type_is_ancestor(target_class->type, target_type))
                ret = target_class;  // 找到匹配接口
        }
        // 多重匹配则返回 NULL（歧义）
    }
    // 普通继承路径
    else if (type_is_ancestor(type, target_type)) {
        ret = class;
    }
    return ret;
}
```

### 9.3 assert 版本与缓存

```c
// object.c:844-881
Object *object_dynamic_cast_assert(Object *obj, const char *typename, ...)
{
    // 1. 检查 4 槽缓存（object_cast_cache）
    for (i = 0; i < 4; i++)
        if (obj->class->object_cast_cache[i] == typename) goto out;

    // 2. 缓存未命中，执行完整转换
    inst = object_dynamic_cast(obj, typename);
    if (!inst && obj) abort();  // 转换失败→致命错误

    // 3. LRU 更新缓存
    for (i = 1; i < 4; i++)
        cache[i-1] = cache[i];
    cache[3] = typename;
}
```

**OBJECT_CHECK 宏展开链**：
```
DEVICE(obj)
  → OBJECT_CHECK(DeviceState, obj, TYPE_DEVICE)
    → object_dynamic_cast_assert(OBJECT(obj), "device", ...)
      → 缓存命中 → 直接返回
      → 缓存未命中 → object_class_dynamic_cast → type_is_ancestor 遍历
```

---

## 10. DeviceState 与 DeviceClass：设备模型

### 10.1 DeviceClass

```c
// qdev.h:115-189
struct DeviceClass {
    ObjectClass parent_class;

    DECLARE_BITMAP(categories, DEVICE_CATEGORY_MAX);
    const char *fw_name;        // firmware 名称
    const char *desc;           // 人类可读描述

    const Property *props_;     // 设备属性数组
    uint16_t props_count_;

    bool user_creatable;        // 用户可通过 -device 创建
    bool hotpluggable;          // 支持热插拔

    /* 回调 */
    DeviceReset legacy_reset;   // 旧式 reset
    DeviceRealize realize;      // 实例化回调
    DeviceUnrealize unrealize;  // 反实例化回调
    DeviceSyncConfig sync_config;

    const VMStateDescription *vmsd;  // 迁移状态描述
    const char *bus_type;            // 挂载的总线类型
};
```

### 10.2 DeviceState

```c
// qdev.h:222-298
struct DeviceState {
    Object parent_obj;

    char *id;                    // 全局设备 ID
    char *canonical_path;        // QOM 树路径
    bool realized;               // 是否已实例化
    int hotplugged;              // 是否热插拔添加
    BusState *parent_bus;        // 所属总线
    NamedGPIOListHead gpios;     // GPIO 列表
    NamedClockListHead clocks;   // 时钟列表
    BusStateHead child_bus;      // 子总线链表
    int num_child_bus;
    ResettableState reset;       // Resettable 接口状态
    GSList *unplug_blockers;     // 拔出阻止列表
    MemReentrancyGuard mem_reentrancy_guard;
};
```

### 10.3 realize 链 — device_set_realized

```c
// qdev.c:513-577（realize 路径核心）
static void device_set_realized(Object *obj, bool value, Error **errp)
{
    if (value && !dev->realized) {
        // 1. 调用 DeviceClass.realize
        dc->realize(dev, &local_err);

        // 2. 通知 realize 监听器
        DEVICE_LISTENER_CALL(realize, Forward, dev);

        // 3. 记录规范路径
        dev->canonical_path = object_get_canonical_path(OBJECT(dev));

        // 4. 注册 VMState（迁移）
        vmstate_register_with_alias_id(...);

        // 5. realize 所有子总线
        QLIST_FOREACH(bus, &dev->child_bus, sibling)
            qbus_realize(bus, errp);

        // 6. 热插拔设备执行 reset
        if (dev->hotplugged)
            resettable_assert_reset(OBJECT(dev), RESET_TYPE_COLD);

        // 7. 调用 hotplug handler
        hotplug_handler_plug(hotplug_ctrl, dev, ...);

        // 8. 设置 realized = true
        qatomic_store_release(&dev->realized, true);
    }
}
```

---

## 11. SysBusDevice：系统总线设备

### 11.1 结构

```c
// sysbus.h:33-53
struct SysBusDeviceClass {
    DeviceClass parent_class;
    char *(*explicit_ofw_unit_address)(const SysBusDevice *dev);
};

// sysbus.h:55-67
struct SysBusDevice {
    DeviceState parent_obj;
    int num_mmio;
    struct {
        hwaddr addr;
        MemoryRegion *memory;
    } mmio[QDEV_MAX_MMIO];    // MMIO 区域数组
    int num_pio;
    uint32_t pio[QDEV_MAX_PIO]; // PIO 端口数组
};
```

### 11.2 MMIO/IRQ 注册

```c
// sysbus.c:177-185
void sysbus_init_mmio(SysBusDevice *dev, MemoryRegion *memory)
{
    dev->mmio[dev->num_mmio].memory = memory;
    dev->num_mmio++;
}

// sysbus.c:165-169
void sysbus_init_irq(SysBusDevice *dev, qemu_irq *p)
{
    qdev_init_gpio_out_named(DEVICE(dev), p, SYSBUS_DEVICE_GPIO_IRQ, 1);
}

// sysbus.c:143-146
void sysbus_mmio_map(SysBusDevice *dev, int n, hwaddr addr)
// 将 MMIO 区域映射到系统地址空间

// sysbus.c:105-108
void sysbus_connect_irq(SysBusDevice *dev, int n, qemu_irq irq)
// 连接设备 IRQ 到系统中断控制器
```

---

## 12. QDev 属性助手：DEFINE_PROP 宏

### 12.1 Property 结构

```c
// qdev-properties.h:16-31
struct Property {
    const char *name;
    const PropertyInfo *info;    // 类型信息（get/set/default）
    ptrdiff_t offset;            // 在 DeviceState 内的偏移
    uint8_t bitnr;
    bool set_default;
    union {
        int64_t i;
        uint64_t u;
    } defval;
};
```

### 12.2 常用 DEFINE_PROP 宏

```c
// qdev-properties.h:79-191（部分）
#define DEFINE_PROP_BOOL(_name, _state, _field, _defval) ...
#define DEFINE_PROP_UINT32(_name, _state, _field, _defval) ...
#define DEFINE_PROP_STRING(_name, _state, _field) ...
#define DEFINE_PROP_LINK(_name, _state, _field, _type, _ptr_type) ...
```

### 12.3 属性注册到 QOM

```c
// qdev-properties.c:1114-1128
void device_class_set_props_n(DeviceClass *dc, const Property *props, int count)
{
    dc->props_ = props;
    dc->props_count_ = count;
    // 属性在 class_init 时注册到 QOM 类属性系统
}

// 每个 Property 通过 qdev_class_add_property() 桥接到 QOM：
// qdev-properties.c:1092-1112
void qdev_class_add_property(DeviceClass *dc, const char *name,
                              ObjectPropertyAccessor *get,
                              ObjectPropertyAccessor *set, ...)
{
    object_class_property_add(OBJECT_CLASS(dc), name, type, get, set, ...);
}
```

---

## 13. 实例分析：ARMCPU 类型链

### 13.1 类型注册

```c
// target/arm/cpu.c:2573-2590
static const TypeInfo arm_cpu_type_info = {
    .name = TYPE_ARM_CPU,                // "arm-cpu"
    .parent = TYPE_CPU,                  // "cpu"
    .instance_size = sizeof(ARMCPU),
    .instance_align = __alignof__(ARMCPU),
    .instance_init = arm_cpu_initfn,     // 实例初始化
    .instance_finalize = arm_cpu_finalizefn,
    .abstract = true,                    // 抽象类，不可直接实例化
    .class_size = sizeof(ARMCPUClass),
    .class_init = arm_cpu_class_init,    // 类初始化
};

type_init(arm_cpu_register_types)
```

### 13.2 继承链

```
TYPE_OBJECT             instance_size = sizeof(Object)
  └─ TYPE_DEVICE        instance_size = sizeof(DeviceState)
      └─ TYPE_CPU       instance_size = sizeof(CPUState)
          └─ TYPE_ARM_CPU  instance_size = sizeof(ARMCPU), abstract
              └─ TYPE_AARCH64_CPU  (具体型号如 "cortex-a57-arm-cpu")
```

### 13.3 初始化顺序

```
object_new("cortex-a57-arm-cpu")
  1. type_initialize() 链：
     Object.class_init → Device.class_init → CPU.class_init → ARMCPU.class_init
     （仅首次，后续复用缓存的 class）

  2. object_init_with_type() 递归：
     Object.instance_init (无) → Device.instance_init → CPU.instance_init
       → arm_cpu_initfn()
         → 初始化 CPUARMState、寄存器、feature 位

  3. object_post_init_with_type() 递归：
     Object.instance_post_init (无) → ... → arm_cpu_post_init()
       → 注册 QOM 属性（"midr", "has_el2", ...）

  4. device_set_realized(true)：
     → arm_cpu_realizefn()
       → 初始化指令集、register CP regs、初始化 GDB 描述
```

---

## 14. 对象路径：QOM 树

### 14.1 根对象

```c
// object.c:1705
Object *object_get_root(void)
// 返回全局根对象单例
```

### 14.2 路径解析

```c
// object.c:2174
Object *object_resolve_path(const char *path, bool *ambiguous)
// 绝对路径（"/"开头）：沿 QOM 树查找
// 相对路径：从根开始部分匹配

// object.c:2073
Object *object_resolve_path_component(Object *parent, const char *part)
// 查找 parent 的 "child<*>" 属性中名为 part 的子对象
```

### 14.3 典型 QOM 树结构

```
/ (root)
├── machine/                     ← MachineState (virt-9.2-machine)
│   ├── peripheral/              ← 用户创建的设备
│   ├── peripheral-anon/         ← 匿名设备
│   ├── unattached/              ← 未挂载总线的设备
│   │   ├── device[0]            ← 如 GIC
│   │   ├── device[1]            ← 如 UART
│   │   └── ...
│   ├── virt.flash0              ← Flash 设备
│   ├── cpu[0]                   ← ARMCPU 实例
│   ├── cpu[1]
│   └── ...
├── chardev/                     ← 字符设备后端
├── objects/                     ← 用户对象（如 memory-backend）
└── type/                        ← 类型注册表
```

**路径访问示例**：
- `/machine/cpu[0]` → 第一个 CPU
- `/machine/peripheral/virtio-net0` → 用户命名的 virtio 网卡
- `qdev_get_machine()` → `object_resolve_path_component(root, "machine")`

---

## 总结

### QOM 核心设计模式

```
┌────────────────────────────────────────────────────────┐
│                    TypeInfo (用户定义)                    │
│  name / parent / class_init / instance_init / ...      │
└──────────────────────┬─────────────────────────────────┘
                       │ type_register_static()
                       ▼
┌────────────────────────────────────────────────────────┐
│                TypeImpl (内部存储)                       │
│  全局哈希表 type_table: name → TypeImpl                 │
│  class 字段初始为 NULL（懒加载）                         │
└──────────────────────┬─────────────────────────────────┘
                       │ type_initialize() — 首次使用时
                       ▼
┌────────────────────────────────────────────────────────┐
│              ObjectClass (类对象，全局唯一)               │
│  通过 memcpy 继承父类 → class_base_init → class_init    │
│  虚函数表 + 类属性 + 接口列表                            │
└──────────────────────┬─────────────────────────────────┘
                       │ object_new() — 每次创建实例
                       ▼
┌────────────────────────────────────────────────────────┐
│              Object (实例，每对象独立)                    │
│  obj->class 指向共享的 ObjectClass                      │
│  obj->properties 实例属性                               │
│  obj->ref 引用计数                                      │
│  instance_init 递归（先父后子）                          │
│  instance_post_init 递归（先父后子）                     │
└────────────────────────────────────────────────────────┘
```

### 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/qom/object.h` | ~700 | ObjectProperty (89-101)、ObjectClass (128-140)、DECLARE 宏 (176-259)、TypeInfo (476-495)、InterfaceInfo/Class (567-587)、INTERFACE_CHECK (607-609)、OBJECT_CHECK/CLASS_CHECK (530-559) |
| `qom/object.c` | ~2800 | TypeImpl (48-75)、type_table (79-96)、type_register (163-182)、type_initialize (336-419)、object_init_with_type (421-430)、object_initialize_with_type (496-512)、object_new (690-730)、object_finalize (663-676)、object_dynamic_cast (835-842)、object_class_dynamic_cast (883-928)、object_ref/unref (1148-1174)、property_add (1176-1264)、resolve_path (2073-2184) |
| `include/qemu/module.h` | ~80 | module_init/type_init 宏 (28-56)、MODULE_INIT_QOM |
| `include/hw/core/qdev.h` | ~300 | DeviceClass (115-189)、DeviceState (222-298) |
| `hw/core/qdev.c` | ~800 | device_set_realized (513-577)、device_class_init (754-790) |
| `include/hw/core/sysbus.h` | ~90 | SysBusDeviceClass (33-53)、SysBusDevice (55-67) |
| `hw/core/sysbus.c` | ~340 | sysbus_init_mmio (177)、sysbus_init_irq (165)、sysbus_mmio_map (143)、sysbus_connect_irq (105) |
| `include/hw/core/qdev-properties.h` | ~200 | Property (16-31)、DEFINE_PROP_* 宏 (79-191) |
| `hw/core/qdev-properties.c` | ~1130 | qdev_class_add_property (1092)、device_class_set_props_n (1114) |
| `target/arm/cpu.c` | ~2590 | arm_cpu_initfn (1172)、arm_cpu_post_init (1480)、arm_cpu_class_init (2505)、arm_cpu_type_info (2573)、type_init (2590) |
