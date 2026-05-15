# QEMU 设备模型与 virtio 框架深度分析

> QEMU 版本：11.0.50  
> 分析范围：设备基类体系、总线系统、设备生命周期、virtio 核心框架、传输层、vhost 集成、ARM virt 机器设备拓扑  
> 源码基线：commit 37863fff59 及周边

---

## 目录

- [第一部分：设备模型框架](#第一部分设备模型框架)
  - [1. 设备基类层级](#1-设备基类层级)
  - [2. DeviceState 与 DeviceClass](#2-devicestate-与-deviceclass)
  - [3. 总线系统](#3-总线系统)
  - [4. SysBusDevice — 平台设备基类](#4-sysbusdevice--平台设备基类)
  - [5. PCIDevice — PCI 设备基类](#5-pcidevice--pci-设备基类)
  - [6. 设备生命周期](#6-设备生命周期)
  - [7. 设备属性系统](#7-设备属性系统)
  - [8. 设备注册模式](#8-设备注册模式)
- [第二部分：virtio 框架](#第二部分virtio-框架)
  - [9. VirtIODevice 核心结构](#9-virtiodevice-核心结构)
  - [10. VirtioDeviceClass 回调接口](#10-virtiodeviceclass-回调接口)
  - [11. VirtQueue 与 vring 机制](#11-virtqueue-与-vring-机制)
  - [12. 传输层：virtio-mmio](#12-传输层virtio-mmio)
  - [13. 传输层：virtio-pci](#13-传输层virtio-pci)
  - [14. 特性协商流程](#14-特性协商流程)
  - [15. virtio-blk 块设备](#15-virtio-blk-块设备)
  - [16. virtio-net 网络设备](#16-virtio-net-网络设备)
  - [17. vhost 集成](#17-vhost-集成)
- [第三部分：ARM virt 机器设备拓扑](#第三部分arm-virt-机器设备拓扑)
  - [18. 内存与中断映射表](#18-内存与中断映射表)
  - [19. 设备创建顺序](#19-设备创建顺序)
  - [20. 平台设备集成](#20-平台设备集成)
  - [21. PCIe 主桥与拓扑](#21-pcie-主桥与拓扑)
  - [22. virtio 设备集成](#22-virtio-设备集成)
  - [23. 设备树生成](#23-设备树生成)
  - [24. 热插拔机制](#24-热插拔机制)
- [附录](#附录)

---

## 第一部分：设备模型框架

### 1. 设备基类层级

QEMU 设备模型构建在 QOM（QEMU Object Model）之上，形成清晰的类型层级：

```
Object (qom/object.h)
  └── DeviceState (TYPE_DEVICE)
        ├── SysBusDevice (TYPE_SYS_BUS_DEVICE)    — 平台/MMIO 设备
        │     ├── PL011State (TYPE_PL011)          — UART
        │     ├── PL031State                       — RTC
        │     ├── GICv3State                       — 中断控制器
        │     └── SMMUv3State                      — IOMMU
        ├── PCIDevice (TYPE_PCI_DEVICE)            — PCI/PCIe 设备
        │     ├── VirtIOPCIProxy                   — virtio-pci 传输
        │     ├── E1000EState                      — 网卡仿真
        │     └── NVMeDevice                       — NVMe 控制器
        └── VirtIODevice (TYPE_VIRTIO_DEVICE)      — virtio 后端设备
              ├── VirtIOBlock                      — 块设备
              ├── VirtIONet                        — 网络设备
              └── VirtIOGPU                        — GPU 设备
```

类型定义位置：
- `TYPE_DEVICE`：`qdev.h:77`
- `TYPE_SYS_BUS_DEVICE`：`sysbus.h:18`（`TYPE_SYSTEM_BUS` 在 `sysbus.h:13`）
- `TYPE_PCI_DEVICE`：`pci_device.h:9`
- `TYPE_VIRTIO_DEVICE`：`virtio.h:88`

### 2. DeviceState 与 DeviceClass

#### DeviceClass（qdev.h:115-189）

DeviceClass 定义设备类级别的行为和元数据：

```c
struct DeviceClass {
    ObjectClass parent_class;

    /* 元数据 */
    DECLARE_BITMAP(categories, DEVICE_CATEGORY_MAX);  // 设备分类（网络/存储/显示等）
    const char *fw_name;           // 固件接口名（如 DT compatible）
    const char *desc;              // 人类可读描述
    const Property *props_;        // 设备属性数组
    uint16_t props_count_;         // 属性计数

    /* 控制标志 */
    bool user_creatable;           // 用户是否可通过 -device 创建
    bool hotpluggable;             // 是否支持热插拔

    /* 回调函数 */
    DeviceReset legacy_reset;      // 旧式 reset（正在迁移到 Resettable 接口）
    DeviceRealize realize;         // 设备实现化（关键回调）
    DeviceUnrealize unrealize;     // 设备反实现化
    DeviceSyncConfig sync_config;  // 配置同步
    const VMStateDescription *vmsd; // 迁移状态描述
    const char *bus_type;          // 所属总线类型
};
```

**关键设计**：`realize` 是设备生命周期的核心回调，在属性全部设置完成后被调用，负责完成设备的硬件仿真初始化。

#### DeviceState（qdev.h:222-260）

DeviceState 是每个设备实例的运行时状态：

```c
struct DeviceState {
    Object parent_obj;

    char *id;                      // 设备 ID（-device 的 id= 参数）
    char *canonical_path;          // QOM 路径
    bool realized;                 // 是否已 realize
    bool pending_deleted_event;    // 是否等待删除事件
    int64_t pending_deleted_expires_ms;
    QDict *opts;                   // 设备选项

    /* 运行时状态 */
    BusState *parent_bus;          // 所在总线
    QLIST_HEAD(, NamedGPIOList) gpios;  // GPIO 列表（IRQ 线）
    QLIST_HEAD(, NamedClockList) clocks; // 时钟列表
    QLIST_HEAD(, BusState) child_bus;    // 子总线列表

    /* Reset 状态 */
    int num_child_bus;
    ResettableState reset;
    GSList *unplug_blockers;       // 阻止拔出的标志
};
```

### 3. 总线系统

#### BusClass 与 BusState（qdev.h:322-405）

```c
struct BusClass {
    ObjectClass parent_class;
    void (*print_dev)(Monitor *mon, DeviceState *dev, int indent);
    char *(*get_dev_path)(DeviceState *dev);
    char *(*get_fw_dev_path)(DeviceState *dev);
    bool (*check_address)(BusState *bus, DeviceState *dev, Error **errp);
    BusRealize realize;
    BusUnrealize unrealize;
    int max_dev;                   // 最大设备数（0 = 无限制）
    bool automatic_ids;            // 自动分配 child ID
};

struct BusState {
    Object obj;
    DeviceState *parent;           // 总线所属设备
    char *name;
    HotplugHandler *hotplug_handler;
    int max_index;
    bool realized;
    bool full;
    int num_children;
    QTAILQ_HEAD(, BusChild) children;  // 挂载的子设备链表
    QLIST_ENTRY(BusState) sibling;     // 同父设备下的兄弟总线
    ResettableState reset;
};
```

#### 总线-设备关系

```
DeviceState (父设备)
  ├── child_bus → BusState (子总线)
  │                 └── children → DeviceState (子设备)
  │                                  └── parent_bus → 指回 BusState
  └── parent_bus → BusState (所在总线)
```

关键函数：
- `bus_add_child()`（qdev.c:82-110）：将设备添加到总线，创建 `child[N]` QOM 属性
- `bus_remove_child()`（qdev.c:59-80）：从总线移除设备
- `qdev_set_parent_bus()`（qdev.c:112-145）：设置/迁移设备所在总线

#### 主要总线类型

| 总线类型 | TYPE 常量 | 注册文件 | 用途 |
|---------|----------|---------|------|
| System Bus | `TYPE_SYSTEM_BUS` | sysbus.h:13 | 平台/MMIO 设备 |
| PCI Bus | `TYPE_PCI_BUS` | pci.c:301 | PCI/PCIe 设备 |
| I2C Bus | `TYPE_I2C_BUS` | core.c:26 | I2C 外设 |
| USB Bus | `TYPE_USB_BUS` | bus.c:41 | USB 设备 |
| virtio Bus | `TYPE_VIRTIO_BUS` | virtio-bus.c | virtio 设备后端 |

### 4. SysBusDevice — 平台设备基类

定义于 `sysbus.h:18-89`（`TYPE_SYS_BUS_DEVICE` 在第 18 行）：

```c
struct SysBusDevice {
    DeviceState parent_obj;

    int num_mmio;
    struct {
        hwaddr addr;
        MemoryRegion *memory;      // 关联的 MemoryRegion
    } mmio[QDEV_MAX_MMIO];

    int num_pio;
    uint32_t pio[QDEV_MAX_PIO];    // Port I/O 基地址
};
```

**核心 API**：

| 函数 | 作用 |
|-----|------|
| `sysbus_init_mmio(dev, mr)` | 注册 MMIO 区域 |
| `sysbus_init_irq(dev, irq)` | 注册中断输出线 |
| `sysbus_mmio_map(dev, n, addr)` | 将第 n 个 MMIO 映射到物理地址 |
| `sysbus_connect_irq(dev, n, irq)` | 连接第 n 条 IRQ 到目标（如 GIC SPI） |
| `sysbus_realize_and_unref(dev, errp)` | realize 并释放引用 |

**使用模式**（以 PL011 UART 为例）：

```c
// hw/arm/virt.c:1304-1313
DeviceState *dev = qdev_new(TYPE_PL011);
sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, base);        // 映射 MMIO
sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0, irq);      // 连接 IRQ
```

### 5. PCIDevice — PCI 设备基类

定义于 `pci_device.h:9-190`：

#### PCIDeviceClass

```c
struct PCIDeviceClass {
    DeviceClass parent_class;

    /* PCI 设备标识 */
    uint16_t vendor_id;
    uint16_t device_id;
    uint8_t revision;
    uint16_t class_id;
    uint16_t subsystem_vendor_id;
    uint16_t subsystem_id;

    /* 回调 */
    void (*realize)(PCIDevice *dev, Error **errp);
    void (*config_write)(PCIDevice *dev, uint32_t addr, uint32_t val, int len);
    uint32_t (*config_read)(PCIDevice *dev, uint32_t addr, int len);
    bool is_express;               // 是否为 PCIe 设备
};
```

#### PCIDevice 关键字段

```c
struct PCIDevice {
    DeviceState qdev;

    /* PCI 配置空间（256 字节标准 / 4096 字节 PCIe） */
    uint8_t *config;               // 配置寄存器值
    uint8_t *cmask;                // 可更改位掩码
    uint8_t *wmask;                // 写掩码
    uint8_t *w1cmask;              // Write-1-to-Clear 掩码
    uint8_t *used;                 // 已使用区域标记

    /* BAR（基地址寄存器） */
    PCIIORegion io_regions[PCI_NUM_REGIONS];

    /* 中断 */
    MSIXInfo *msix;                // MSI-X 信息
    MSIInfo *msi;                  // MSI 信息

    /* PCIe 能力 */
    PCIExpressDevice *exp;         // PCIe 扩展能力
    SHPCDevice *shpc;              // 标准热插拔控制器

    /* ROM */
    MemoryRegion *rom;
    uint32_t rom_bar;
};
```

**BAR 注册**：通过 `pci_register_bar()` 将 `MemoryRegion` 关联到 BAR，PCI 核心负责地址解码和 MMIO 路由。

### 6. 设备生命周期

#### 6.1 创建流程（-device CLI 路径）

```
命令行: -device virtio-blk-pci,drive=hd0,id=blk0
          │
          ▼
qdev_device_add() [qdev-monitor.c:652-764]
  ├── 解析 driver=, bus=, id= 参数
  ├── qdev_device_add_from_qdict()
  │     ├── object_new(driver)         → QOM 实例化（instance_init 链）
  │     ├── 设置属性（drive=, id= 等）
  │     └── qdev_realize()
  │           └── object_property_set_bool("realized", true)
  │                 └── device_set_realized()  [qdev.c:474-638]
  └── 返回 DeviceState*
```

#### 6.2 device_set_realized() 详细流程

`device_set_realized()`（qdev.c:474-638）是设备实现化的核心：

```
device_set_realized(obj, true, errp)
  │
  ├── 1. 热插拔检查
  │     └── hotplug_handler_pre_plug()   — 通知总线准备
  │
  ├── 2. 调用设备 realize 回调
  │     └── dc->realize(dev, errp)
  │           └── 对于 PCI 设备：pci_qdev_realize()
  │                 └── pc->realize(pci_dev, errp)
  │                       └── 具体设备 realize
  │
  ├── 3. 注册 VMState（用于迁移）
  │     └── vmstate_register_with_alias_id()
  │
  ├── 4. realize 子总线
  │     └── 遍历 dev->child_bus，逐一 realize
  │
  ├── 5. 热插拔完成通知
  │     └── hotplug_handler_plug()
  │
  └── 失败路径：
        ├── vmstate_unregister()
        ├── dc->unrealize(dev)
        └── 通知监听器
```

#### 6.3 unrealize 流程

```
device_set_realized(obj, false, errp)
  ├── 通知删除监听器
  ├── unrealize 子总线
  ├── vmstate_unregister()
  ├── dc->unrealize(dev)
  └── hotplug_handler_unplug()
```

### 7. 设备属性系统

#### Property 定义（qdev-properties.h:16-257）

```c
struct Property {
    const char *name;              // 属性名
    const PropertyInfo *info;      // 类型信息（get/set/parse 回调）
    ptrdiff_t offset;              // 在 DeviceState 中的偏移
    uint8_t bitnr;                 // bit 属性的位号
    bool set_default;
    union {
        int64_t i;
        uint64_t u;
    } defval;
};
```

#### 标准属性类型

| 类型 | PropertyInfo | C 类型 | 示例 |
|-----|-------------|--------|-----|
| `DEFINE_PROP_BOOL` | prop_info_bool | bool | `hotpluggable` |
| `DEFINE_PROP_UINT32` | prop_info_uint32 | uint32_t | `vectors` |
| `DEFINE_PROP_STRING` | prop_info_string | char* | `serial` |
| `DEFINE_PROP_LINK` | prop_info_link | Object* | `drive` |
| `DEFINE_PROP_BIT` | prop_info_bit | uint32_t bit | `ioeventfd` |
| `DEFINE_PROP_ON_OFF_AUTO` | prop_info_on_off_auto | OnOffAuto | `msi` |

#### 属性设置流程

```
CLI: -device virtio-blk-pci,drive=hd0
  │
  ├── qdev_device_add_from_qdict()
  │     └── object_property_set() → 设置每个属性
  │           └── PropertyInfo->set() → 类型特定的解析和赋值
  │
  └── 全局/兼容属性
        └── object_apply_compat_props() [compat-properties.c]
              └── 机器版本兼容属性覆盖
```

#### 兼容属性机制

`qdev_prop_register_global()` / `qdev_prop_set_globals()`（qdev-properties.c）提供全局属性覆盖，用于机器版本兼容：

```c
// 示例：旧机器版本禁用某个功能
GlobalProperty compat_props[] = {
    { "virtio-blk-pci", "ioeventfd", "off" },
};
```

### 8. 设备注册模式

所有 QEMU 设备遵循统一的注册模式：

```c
// 1. 定义 TypeInfo
static const TypeInfo my_device_info = {
    .name          = TYPE_MY_DEVICE,
    .parent        = TYPE_SYS_BUS_DEVICE,  // 或 TYPE_PCI_DEVICE
    .instance_size = sizeof(MyDeviceState),
    .instance_init = my_device_init,       // 早期初始化
    .class_init    = my_device_class_init, // 类初始化
};

// 2. class_init 中设置回调
static void my_device_class_init(ObjectClass *klass, const void *data) {
    DeviceClass *dc = DEVICE_CLASS(klass);
    dc->realize = my_device_realize;       // 必须：realize 回调
    dc->vmsd = &vmstate_my_device;         // 迁移状态
    device_class_set_props(dc, my_device_props);  // 属性
    set_bit(DEVICE_CATEGORY_MISC, dc->categories);
}

// 3. 构造函数注册
static void my_device_register_types(void) {
    type_register_static(&my_device_info);
}
type_init(my_device_register_types)
```

**git 趋势**：commit 12d1a768bd 将 `class_init()` 的 data 参数改为 `const`，提升 API 安全性。

---

## 第二部分：virtio 框架

### 9. VirtIODevice 核心结构

定义于 `virtio.h:108-171`：

```c
struct VirtIODevice {
    DeviceState parent_obj;        // QOM 父类

    const char *name;              // 设备名
    uint8_t status;                // VirtIO 设备状态字段
    uint8_t isr;                   // 中断状态寄存器
    uint16_t queue_sel;            // 当前选中的队列号

    /* 特性协商三级结构 */
    uint64_t host_features;        // 主机端（QEMU）支持的完整特性集
    uint64_t guest_features;       // 客户机驱动选择的特性集
    uint64_t backend_features;     // 后端（如 vhost）支持的特性子集

    /* 配置空间 */
    size_t config_len;             // 设备配置空间长度
    void *config;                  // 配置空间数据
    uint16_t config_vector;        // 配置变更的 MSI-X 向量
    uint32_t generation;           // 配置空间代数（用于原子读取）

    /* 队列 */
    int nvectors;                  // MSI-X 向量数
    VirtQueue *vq;                 // VirtQueue 数组

    /* 通知 */
    MemoryListener listener;       // 地址空间变更监听
    uint16_t device_id;            // VirtIO 设备 ID（blk=2, net=1, gpu=16）

    /* 运行时状态 */
    bool vm_running;               // VM 运行状态
    bool broken;                   // 设备异常标志
    bool disabled;                 // 临时禁用
    bool use_started;              // 使用 started 标志（modern 4.1+）
    bool started;                  // 设备已启动
    bool start_on_kick;            // 未协商 1.0 时的启动方式
    bool vhost_started;            // vhost 是否已启动

    /* DMA */
    AddressSpace *dma_as;          // DMA 地址空间

    /* 设备 IOTLB */
    bool device_iotlb_enabled;     // 设备 IOTLB 是否启用
};
```

**三级特性架构**是 virtio 设计的核心：
1. `host_features`：QEMU 设备端全部可提供的特性
2. `backend_features`：vhost 后端的能力约束
3. `guest_features`：客户机驱动最终选择的交集

### 10. VirtioDeviceClass 回调接口

定义于 `virtio.h:173-241`：

```c
struct VirtioDeviceClass {
    DeviceClass parent;

    /* 生命周期 */
    DeviceRealize realize;         // 设备实现化
    DeviceUnrealize unrealize;     // 反实现化

    /* 特性协商 */
    void (*get_features_ex)(...);  // 扩展特性获取（支持 >64 位）
    void (*set_features_ex)(...);  // 扩展特性设置
    uint64_t (*get_features)(...); // 传统特性获取
    uint64_t (*bad_features)(...); // 坏特性列表
    void (*set_features)(...);     // 传统特性设置
    int (*validate_features)(...); // 特性验证

    /* 配置空间 */
    void (*get_config)(...);       // 读取设备配置
    void (*set_config)(...);       // 写入设备配置

    /* 状态管理 */
    void (*reset)(...);            // 设备复位
    int (*set_status)(...);        // 状态变更回调
    void (*queue_reset)(...);      // 单队列复位
    void (*queue_enable)(...);     // 单队列启用

    /* 通知 */
    bool (*guest_notifier_pending)(...);  // 检查待处理通知
    void (*guest_notifier_mask)(...);     // 掩码/取消掩码通知

    /* ioeventfd */
    int (*start_ioeventfd)(...);   // 启动 ioeventfd
    void (*stop_ioeventfd)(...);   // 停止 ioeventfd

    /* 迁移 */
    int (*pre_load_queues)(...);   // 加载前队列准备
    void (*save)(...);             // 保存状态
    int (*load)(...);              // 加载状态
    int (*post_load)(...);         // 加载后回调
    const VMStateDescription *vmsd;

    /* vhost */
    struct vhost_dev *(*get_vhost)(...);  // 获取 vhost 设备
    uint64_t legacy_features;             // 仅 legacy 接口暴露的特性
};
```

### 11. VirtQueue 与 vring 机制

#### VirtQueue 内部结构（virtio.c:123-159）

VirtQueue 是 QEMU 内部结构，不直接暴露给外部：

```c
struct VirtQueue {
    VRing vring;                   // vring 描述符

    /* 设备端状态 */
    uint16_t last_avail_idx;       // 设备已处理的 avail 索引
    uint16_t shadow_avail_idx;     // avail 索引缓存
    uint16_t used_idx;             // 已填充的 used 索引
    uint16_t signalled_used;       // 上次通知时的 used 索引
    bool signalled_used_valid;

    /* 通知控制 */
    bool notification;             // 是否启用通知
    uint16_t queue_index;          // 队列编号
    unsigned int inuse;            // 正在使用的描述符数

    /* 回调 */
    VirtIOHandleOutput handle_output;  // 设备处理函数
    VirtIOHandleAIOOutput handle_aio_output;

    /* ioeventfd */
    EventNotifier guest_notifier;  // 客户→主机通知
    EventNotifier host_notifier;   // 主机→客户通知
    bool host_notifier_enabled;

    VirtIODevice *vdev;            // 所属设备
    QLIST_ENTRY(VirtQueue) node;
};
```

#### vring 内存布局

virtio 规范定义的三段式环形缓冲区：

```
Guest 物理内存:
┌─────────────────────────────────────┐
│ Descriptor Table (16 bytes × N)     │ ← 描述符数组
│   addr[64], len[32], flags[16],     │
│   next[16]                          │
├─────────────────────────────────────┤
│ Available Ring                      │ ← 驱动→设备
│   flags[16], idx[16],               │
│   ring[N], used_event[16]           │
├─────────────────────────────────────┤
│ Used Ring                           │ ← 设备→驱动
│   flags[16], idx[16],               │
│   ring[N]{id,len}, avail_event[16]  │
└─────────────────────────────────────┘
```

#### 关键数据流 API

**设备端处理请求**：

```
1. virtqueue_pop() [virtio.c:2030-2040]
   ├── 从 avail ring 取出描述符链
   ├── 构建 VirtQueueElement（scatter-gather 列表）
   └── 返回给设备处理

2. 设备处理请求（异步 I/O）

3. virtqueue_push() [virtio.c:954-1228]
   ├── virtqueue_fill()  — 填充 used ring 条目
   └── virtqueue_flush() — 更新 used idx

4. virtio_notify() [virtio.c:2730]
   ├── virtio_should_notify() — 检查是否需要通知
   └── virtio_notify_vector() — 触发中断/MSI-X
```

**VirtQueueElement（virtio.c:66-79）**：

```c
typedef struct VirtQueueElement {
    unsigned int index;            // 描述符链头索引
    unsigned int len;              // 总长度
    unsigned int ndescs;           // 描述符数量
    unsigned int out_num;          // 输出 SG 段数（驱动→设备）
    unsigned int in_num;           // 输入 SG 段数（设备→驱动）
    hwaddr *in_addr;               // 输入段物理地址数组
    hwaddr *out_addr;              // 输出段物理地址数组
    struct iovec *in_sg;           // 输入 scatter-gather
    struct iovec *out_sg;          // 输出 scatter-gather
} VirtQueueElement;
```

#### Split vs Packed Ring

QEMU 同时支持两种 vring 格式，描述符遍历代码位于 `virtio.c:1285-1593`：

- **Split Ring**（传统）：三段独立内存，描述符链表形式
- **Packed Ring**（virtio 1.1）：统一环，通过 wrap counter 和 flags 实现无锁操作

### 12. 传输层：virtio-mmio

#### VirtIOMMIOProxy（virtio-mmio.h:44-72）

```c
struct VirtIOMMIOProxy {
    SysBusDevice parent_obj;       // 继承平台设备

    MemoryRegion iomem;            // MMIO 寄存器区域（0x200 字节）
    qemu_irq irq;                  // 中断输出线

    bool legacy;                   // 是否 legacy 模式
    uint32_t host_features_sel;    // 主机特性选择器
    uint32_t guest_features_sel;   // 客户特性选择器
    uint32_t guest_page_shift;     // 页大小

    VirtIODevice *vdev;            // 关联的 virtio 设备
};
```

#### MMIO 寄存器空间

virtio-mmio 提供固定的寄存器接口，映射到 0x200 字节的 MMIO 区域：

| 偏移 | 名称 | 方向 | 作用 |
|------|------|------|------|
| 0x000 | MagicValue | R | 魔数 0x74726976 ("virt") |
| 0x004 | Version | R | 版本号 |
| 0x008 | DeviceID | R | 设备类型 |
| 0x010 | HostFeatures | R | 主机特性 |
| 0x014 | HostFeaturesSel | W | 特性页选择 |
| 0x020 | GuestFeatures | W | 客户特性 |
| 0x030 | QueueSel | W | 队列选择 |
| 0x044 | QueueReady | RW | 队列就绪 |
| 0x050 | QueueNotify | W | 队列通知（kick） |
| 0x060 | InterruptStatus | R | 中断状态 |
| 0x064 | InterruptACK | W | 中断确认 |
| 0x070 | Status | RW | 设备状态 |

**realize 流程**（virtio-mmio.c:772-805）：

```
virtio_mmio_realizefn()
  ├── sysbus_init_mmio(dev, &proxy->iomem)   — 注册 MMIO
  └── sysbus_init_irq(dev, &proxy->irq)      — 注册 IRQ
```

### 13. 传输层：virtio-pci

#### VirtIOPCIProxy（virtio-pci.h:86-154）

```c
struct VirtIOPCIProxy {
    PCIDevice pci_dev;             // PCI 设备基类

    /* PCI 资源 */
    MemoryRegion bar;              // PCI BAR 区域
    union {
        struct {
            MemoryRegion common;   // Common config
            MemoryRegion isr;      // ISR config
            MemoryRegion device;   // Device config
            MemoryRegion notify;   // Notify config
            MemoryRegion notify_pio; // PIO notify
        };
    };

    /* 配置 */
    uint32_t flags;
    bool disable_modern;           // 禁用 modern 模式
    bool disable_legacy;           // 禁用 legacy 模式

    /* ioeventfd */
    VirtIOIRQFD *vector_irqfd;
    int nvqs_with_notifiers;

    VirtIODevice *vdev;
};
```

#### Modern vs Legacy 模式

| 特性 | Legacy (virtio 0.9) | Modern (virtio 1.0+) |
|-----|---------------------|---------------------|
| 配置访问 | PCI I/O 端口 | PCI Capability 指向的 BAR |
| 特性位 | 32 位 | 64+ 位 |
| 端序 | Guest native | Little-endian |
| 实现 | virtio-pci.c:427-520 | virtio-pci.c:652-860 |

**Modern 模式配置空间**通过 PCI Capability 结构指向 BAR 中的不同区域：

```
PCI Config Space
  └── Capabilities
        ├── VIRTIO_PCI_CAP_COMMON_CFG  → BAR 区域: common config
        ├── VIRTIO_PCI_CAP_ISR_CFG     → BAR 区域: ISR
        ├── VIRTIO_PCI_CAP_DEVICE_CFG  → BAR 区域: device config
        └── VIRTIO_PCI_CAP_NOTIFY_CFG  → BAR 区域: notify
```

#### ioeventfd 优化

`virtio-pci.c:371-414` 中的 ioeventfd 绑定将 notify 写操作直接路由到 KVM，避免 VM Exit 到 QEMU 用户空间：

```
Guest 写 notify 寄存器
  │
  ├── 无 ioeventfd: VM Exit → QEMU → virtqueue_handle_output()
  │
  └── 有 ioeventfd: KVM 直接触发 eventfd → QEMU IO 线程处理
```

### 14. 特性协商流程

virtio 特性协商是驱动和设备之间的握手过程：

```
                     设备端 (QEMU)                    驱动端 (Guest)
                         │                                 │
  1. virtio_init()       │                                 │
     设置 host_features  │                                 │
                         │                                 │
  2.                     │◄──── 读取 HostFeatures ─────────│
                         │      （guest 发现设备支持什么）    │
                         │                                 │
  3.                     │──── 写入 GuestFeatures ────────►│
                         │     （guest 选择需要的特性）       │
                         │                                 │
  4. virtio_set_features()│                                │
     验证 + 保存          │                                 │
                         │                                 │
  5.                     │◄──── 写入 Status = FEATURES_OK──│
                         │                                 │
  6. virtio_set_status()  │                                │
     最终确认             │                                 │
```

**关键实现**（virtio.c:2282-2313）：

```c
virtio_set_status(vdev, val)
  ├── 如果 val 包含 VIRTIO_CONFIG_S_FEATURES_OK
  │     └── 调用 vdc->validate_features()
  │           └── 验证 guest 选择的特性组合是否合法
  ├── vdc->set_status(vdev, val)
  └── 如果 DRIVER_OK 被设置 → 设备完全就绪
```

### 15. virtio-blk 块设备

#### VirtIOBlock 结构（virtio-blk.h:37-109）

```c
struct VirtIOBlock {
    VirtIODevice parent_obj;

    BlockBackend *blk;             // 块后端（镜像文件/块设备）
    void *rq;                      // 请求队列
    VirtIOBlkConf conf;            // 配置（serial, num-queues 等）

    /* 多队列 */
    uint16_t num_queues;           // 队列数
    struct VirtIOBlockDataPlane *dataplane;  // 数据面加速

    /* 统计 */
    unsigned int in_flight;
    uint64_t max_transfer;
};
```

#### 请求处理流程

```
Guest 驱动写入 QueueNotify
  │
  ▼
virtio_blk_handle_output() [virtio-blk.c]
  │
  ├── virtqueue_pop()              — 取出请求描述符
  │     └── 解析 virtio_blk_outhdr（type, sector, ioprio）
  │
  ├── 根据 type 分发:
  │     ├── VIRTIO_BLK_T_IN/OUT    → blk_aio_preadv/pwritev()
  │     ├── VIRTIO_BLK_T_FLUSH     → blk_aio_flush()
  │     ├── VIRTIO_BLK_T_GET_ID    → 返回设备序列号
  │     └── VIRTIO_BLK_T_ZONE_*    → 区域管理 [virtio-blk.c:640-749]
  │
  └── 完成回调 [virtio-blk.c:57-69]:
        ├── virtqueue_push(vq, elem, len)
        └── virtio_notify(vdev, vq)
```

**多队列**：通过 `num-queues` 属性配置，每个队列可绑定到不同的 IO 线程（iothread-vq-mapping）。

**git 安全修复**：commit 4913ae36f9 修复了 zone report 的缓冲区溢出漏洞（CVE-2026-5761）。

### 16. virtio-net 网络设备

#### VirtIONet 结构（virtio-net.h:158-233）

```c
struct VirtIONet {
    VirtIODevice parent_obj;

    /* 队列对（TX + RX 每个 vCPU 一对） */
    uint16_t max_queue_pairs;
    uint16_t curr_queue_pairs;

    /* 网络后端 */
    NICState *nic;
    NICConf nic_conf;

    /* vhost 集成 */
    VHostNetState *vhost_net;
    bool vhost_started;

    /* 配置 */
    uint8_t mac[ETH_ALEN];         // MAC 地址
    uint16_t status;               // 链路状态
    uint16_t mtu;                  // MTU

    /* 功能 */
    uint32_t has_vnet_hdr;         // vnet header 支持
    int multiqueue;                // 多队列模式
    uint32_t rss_data_loaded;      // RSS 数据
};
```

#### 数据路径

```
                RX 路径                              TX 路径
                  │                                    │
 网络后端 →  qemu_send_packet()              virtio_net_handle_tx()
                  │                                    │
         virtio_net_receive()                 virtqueue_pop()
                  │                                    │
         virtqueue_push(rx_vq)               qemu_sendv_packet_async()
                  │                                    │
         virtio_notify()                     virtqueue_push(tx_vq)
                  │                                    │
         Guest RX 中断                        Guest TX 完成
```

**多队列**（virtio-net.c:768-786）：每个队列对对应一个 TX/RX VirtQueue，支持 vCPU 亲和性，通过 `virtio_net_set_multiqueue()` 动态调整。

**特性协商**（virtio-net.c:929-1003）：包括 VIRTIO_NET_F_MQ（多队列）、VIRTIO_NET_F_HOST_TSO4（TSO 卸载）、VIRTIO_NET_F_RSS（接收端缩放）等。

**git 趋势**：commit 1c79ab6937 默认启用 UDP tunnel GSO 支持；commit a5289563ad 实现 UDP tunnel 特性卸载。

### 17. vhost 集成

#### 架构概览

vhost 是一种将 virtqueue 处理从 QEMU 用户空间卸载到内核或独立进程的加速机制：

```
                  标准路径                          vhost 路径
                    │                                  │
  Guest ──── VirtQueue ──── QEMU          Guest ──── VirtQueue ──── vhost
              (用户空间处理)                             (内核/进程直接处理)
              每次 I/O 都经过 QEMU                       绕过 QEMU 数据面
```

#### vhost_dev 结构（vhost.h:79-137）

```c
struct vhost_dev {
    VirtIODevice *vdev;            // 关联的 virtio 设备
    MemoryListener memory_listener; // 内存变更监听
    struct vhost_virtqueue *vqs;   // vhost virtqueue 数组
    unsigned int nvqs;             // 队列数

    /* 后端接口 */
    const VhostOps *vhost_ops;     // 操作回调（kernel/user/vDPA）
    void *opaque;                  // 后端私有数据

    /* 特性 */
    uint64_t features;             // 协商的特性
    uint64_t acked_features;       // 已确认的特性
    uint64_t backend_features;     // 后端能力

    /* IOTLB */
    bool backend_cap;
    bool started;
    bool log_enabled;
    uint64_t log_size;
    Error *migration_blocker;
};
```

#### vhost_virtqueue（vhost.h:24-41）

```c
struct vhost_virtqueue {
    int kick;                      // kick eventfd（guest → backend）
    int call;                      // call eventfd（backend → guest）
    void *desc;                    // 描述符表映射
    void *avail;                   // avail ring 映射
    void *used;                    // used ring 映射
    int num;                       // 队列深度
    unsigned long long used_phys;  // used ring 物理地址
    unsigned used_size;            // used ring 大小
    void *desc_phys;               // 描述符物理地址
    unsigned desc_size;
    bool notif_sent;
};
```

#### vhost-kernel vs vhost-user

| 对比项 | vhost-kernel | vhost-user |
|-------|-------------|-----------|
| 后端位置 | Linux 内核模块 | 独立用户空间进程 |
| 通信机制 | ioctl | Unix socket + 共享内存 |
| 代码 | vhost.c | vhost-user.c |
| 性能 | 极高（零拷贝） | 高（仍需跨进程通知） |
| 灵活性 | 有限（内核代码） | 高（可自定义后端） |
| 典型设备 | vhost-net | DPDK vhost-user |

#### vhost-user 协议（vhost-user.c:57-103）

vhost-user 通过 Unix socket 传递控制消息，通过 mmap 共享 vring 内存：

```
QEMU (master)                     vhost-user backend (slave)
     │                                    │
     │── VHOST_USER_SET_OWNER ──────────►│
     │── VHOST_USER_GET_FEATURES ──────►│
     │◄── features ──────────────────────│
     │── VHOST_USER_SET_MEM_TABLE ─────►│  (附带 fd 数组)
     │── VHOST_USER_SET_VRING_NUM ─────►│
     │── VHOST_USER_SET_VRING_ADDR ────►│
     │── VHOST_USER_SET_VRING_KICK ────►│  (传递 eventfd)
     │── VHOST_USER_SET_VRING_CALL ────►│  (传递 eventfd)
     │                                    │
     │  数据面：Guest 直接通过共享内存和       │
     │  eventfd 与 backend 交互             │
```

**git 趋势**：commit e0822e6085 引入跳过 drain 的协议特性；commit 1ba9a52203 使 `SET_VRING_FILE` 同步化。

---

## 第三部分：ARM virt 机器设备拓扑

### 18. 内存与中断映射表

ARM virt 机器定义了固定的地址映射（virt.c:173-254）：

#### 物理地址映射

| 设备 | 基地址 | 大小 | 说明 |
|-----|--------|------|------|
| Flash | 0x0000_0000 | 128 MiB | pflash（启动固件） |
| GIC Dist | 0x0800_0000 | 64 KiB | GIC 分发器 |
| GIC CPU | 0x0801_0000 | 64 KiB | GIC CPU 接口 |
| GIC ITS | 0x0808_0000 | 128 KiB | GIC ITS (MSI) |
| GIC Redist | 0x080A_0000 | ~15 MiB | GIC 再分发器 |
| UART0 | 0x0900_0000 | 4 KiB | PL011 串口 |
| RTC | 0x0901_0000 | 4 KiB | PL031 RTC |
| fw_cfg | 0x0902_0000 | 24 字节 | QEMU fw_cfg |
| GPIO | 0x0903_0000 | 4 KiB | PL061 GPIO |
| UART1 | 0x0904_0000 | 4 KiB | 第二串口 |
| SMMU | 0x0905_0000 | 变长 | SMMUv3 IOMMU |
| ACPI GED | 0x0908_0000 | 变长 | 通用事件设备 |
| virtio-mmio | 0x0A00_0000 | 512B × N | virtio MMIO 传输 |
| Platform Bus | 0x0C00_0000 | 32 MiB | 动态设备 |
| PCIe MMIO | 0x1000_0000 | ~767 MiB | PCIe MMIO 窗口 |
| PCIe PIO | 0x3EFF_0000 | 64 KiB | PCIe PIO 窗口 |
| PCIe ECAM | 0x3F00_0000 | 16 MiB | PCIe 配置空间 |
| RAM | 0x4000_0000 (1GiB) | 变长 | 系统内存 |

**高地址区域**（extended_memmap，virt.c:233-241）：

```
256 GiB+ (浮动，对齐到区域大小):
  ├── GIC Redist2     64 MiB    （>123 CPU 时使用）
  ├── CXL Host        1 MiB     （CXL 设备）
  ├── High ECAM       256 MiB   （扩展 PCIe）
  └── High PCIe MMIO  512 GiB   （64 位 MMIO 窗口）
```

#### IRQ 分配（a15irqmap，virt.c:243-253）

| 设备 | GIC SPI 号 | 说明 |
|-----|-----------|------|
| UART0 | 1 | 串口中断 |
| RTC | 2 | RTC 中断 |
| PCIe | 3-6 | INTx (A/B/C/D) |
| GPIO | 7 | GPIO 中断 |
| UART1 | 8 | 第二串口 |
| ACPI GED | 9 | 热插拔/电源事件 |
| virtio-mmio | 16-47 | 32 个 virtio 传输 |
| GICv2M | 48-79 | GICv2M MSI |
| SMMU | 74-77 | SMMUv3 中断 |
| Platform Bus | 112+ | 平台设备中断 |

### 19. 设备创建顺序

`machvirt_init()`（virt.c:2737+）按以下顺序创建设备：

```
machvirt_init()
  │
  ├── 1. create_fdt()                    [virt.c:364-445]
  │     └── 创建空设备树，设置基本属性
  │
  ├── 2. CPU 创建
  │     ├── fdt_add_cpu_nodes()          [virt.c:598-874]
  │     └── 创建 ARMCPU 实例
  │
  ├── 3. 固件初始化
  │     ├── virt_flash_fdt()             [virt.c:1641-1683]
  │     └── virt_firmware_init()         [virt.c:2663+]
  │
  ├── 4. create_gic()                    [virt.c:2857]
  │     └── GICv3/v2 中断控制器
  │
  ├── 5. UART                            [virt.c:2883-2894]
  │     └── create_uart() × 2           [virt.c:1295-1345]
  │
  ├── 6. create_rtc()                    [virt.c:2907, 1347-1369]
  │
  ├── 7. create_pcie()                   [virt.c:2909, 1912-2043]
  │     └── GPEX PCIe 主桥
  │
  ├── 8. GPIO / ACPI GED                 [virt.c:2912-2923]
  │     ├── create_gpio_devices()        [virt.c:1453-1502]
  │     └── create_acpi_ged()            [virt.c:1021-1062]
  │
  ├── 9. create_virtio_devices()         [virt.c:2933, 1504-1568]
  │     └── 创建 32 个 virtio-mmio 传输
  │
  ├── 10. fw_cfg + platform bus          [virt.c:2935-2938]
  │
  └── 11. 完成
        ├── DTB 加载                      [virt.c:2952-2962]
        └── virt_machine_done()          [virt.c:2162-2200]
              └── virt_acpi_setup()       — ACPI 表生成
```

### 20. 平台设备集成

#### PL011 UART 创建（virt.c:1295-1345）

```c
create_uart(vms, VIRT_UART0, sysmem, serial_hd(0))
  ├── qdev_new(TYPE_PL011)
  ├── qdev_prop_set_chr(dev, "chardev", chr)   — 连接字符后端
  ├── sysbus_realize_and_unref(SYS_BUS_DEVICE(dev))
  ├── sysbus_mmio_map(dev, 0, base)            — 0x09000000
  ├── sysbus_connect_irq(dev, 0, qdev_get_gpio_in(vms->gic, irq))
  │                                             — 连接到 GIC SPI 1
  └── DT 节点: compatible = "arm,pl011"
```

#### PL031 RTC 创建（virt.c:1347-1369）

```c
create_rtc(vms)
  └── sysbus_create_simple("pl031", base, qdev_get_gpio_in(vms->gic, irq))
        // 一行搞定：创建 + realize + MMIO 映射 + IRQ 连接
```

#### PL061 GPIO + 电源/复位控制（virt.c:1453-1502）

```c
create_gpio_devices(vms)
  ├── qdev_new(TYPE_PL061)                     — GPIO 控制器
  ├── sysbus_mmio_map(dev, 0, 0x09030000)
  ├── sysbus_connect_irq(dev, 0, GIC SPI 7)
  │
  ├── gpio-key (电源按钮)                       [virt.c:1398-1415]
  │     ├── qdev_new(TYPE_GPIO_KEY)
  │     └── GPIO 线连接到 PL061 pin 3
  │
  └── gpio-pwr (关机/复位)                      [virt.c:1420-1451]
        ├── qdev_new("gpio-pwr")
        └── GPIO 线连接到 PL061 pin 0, 1
```

### 21. PCIe 主桥与拓扑

#### GPEX 主桥创建（virt.c:1912-2043）

```c
create_pcie(vms)
  │
  ├── 创建 GPEX 主桥
  │     ├── qdev_new(TYPE_GPEX_HOST)
  │     └── sysbus_realize_and_unref()
  │
  ├── ECAM 映射
  │     └── memory_region_add_subregion(sysmem, ecam_base,
  │           sysbus_mmio_get_region(dev, 0))   — 0x3F000000
  │
  ├── MMIO 窗口映射
  │     ├── mmio_reg → sysmem @ 0x10000000      — 32 位 MMIO
  │     └── high_mmio_reg → sysmem @ 256GiB+    — 64 位 MMIO
  │
  ├── PIO 窗口映射
  │     └── pio_reg → sysmem @ 0x3EFF0000
  │
  ├── INTx 中断路由
  │     └── 4 条 INTx 线 → GIC SPI 3-6          [virt.c:1974-1978]
  │
  └── DT 节点
        ├── compatible = "pci-host-ecam-generic"
        ├── ranges (MMIO/PIO 窗口)
        ├── interrupt-map (INTx → GIC)           [virt.c:1757-1792]
        ├── msi-map (MSI → ITS)
        └── iommu-map (如果启用 SMMU)
```

#### PCI 中断路由详解

`create_pcie_irq_map()`（virt.c:1757-1792）构建 `interrupt-map`：

```
PCI 设备 (dev, fn)
  │
  ├── INTx pin = (slot + pin) % 4
  │
  └── 映射到 GIC SPI:
        INTA → SPI 3
        INTB → SPI 4
        INTC → SPI 5
        INTD → SPI 6
```

### 22. virtio 设备集成

#### virtio-mmio 传输创建（virt.c:1504-1568）

```c
create_virtio_devices(vms)
  │
  ├── for i in 0..NUM_VIRTIO_TRANSPORTS-1:     — 通常 32 个
  │     ├── base = VIRT_MMIO.base + i * 0x200   — 0x0A000000 + i*512
  │     ├── irq = VIRT_MMIO.irq + i             — SPI 16 + i
  │     │
  │     ├── sysbus_create_simple("virtio-mmio", base, irq)
  │     │     └── 创建空的 virtio-mmio 传输（无设备附加）
  │     │
  │     └── DT 节点（反序创建以确保 /dev/vd* 排序一致）
  │           ├── compatible = "virtio,mmio"
  │           ├── reg = <base, 0x200>
  │           └── interrupts = <GIC_SPI, irq, IRQ_TYPE_EDGE_RISING>
  │
  └── 设备附加:
        // 用户通过 -device virtio-blk-device,... 附加
        // virtio-mmio 传输自动绑定
```

**属性控制**：commit ee4d1ff3af 添加了 `virtio-mmio-transports` 属性，允许用户配置 virtio-mmio 传输数量。

#### virtio-pci 设备附加流程

```
CLI: -device virtio-blk-pci,drive=hd0

qdev_device_add()
  │
  ├── 创建 VirtIOBlockPCI 实例（包含 VirtIOPCIProxy + VirtIOBlock）
  │
  ├── 设置属性（drive=hd0 等）
  │
  └── qdev_realize()
        └── virtio_blk_pci_realize()       [virtio-blk-pci.c:49-64]
              ├── 设置队列/向量默认值
              ├── qdev_realize(vdev, BUS(&vpci_dev->bus))
              │     └── VirtIOBlock.realize()
              └── virtio_pci_realize()     [virtio-pci.c]
                    ├── 注册 PCI BAR
                    ├── 设置 MSI-X
                    └── 建立 common/isr/device/notify 区域
```

### 23. 设备树生成

#### FDT 创建流程

```
create_fdt() [virt.c:364-445]
  ├── /chosen 节点
  ├── /memory 节点
  └── 基本属性（#address-cells, #size-cells, model）

各 create_*() 函数依次添加节点:
  ├── fdt_add_cpu_nodes()          [virt.c:598-874]
  │     ├── /cpus 节点
  │     └── /cpus/cpu@N 节点（每个 CPU）
  │
  ├── fdt_add_timer_nodes()        [virt.c:447-512]
  │     └── /timer 节点
  │
  ├── fdt_add_gic_node()           [virt.c:915-991]
  │     └── /intc 节点
  │
  ├── create_uart() → DT 节点
  ├── create_rtc() → DT 节点
  ├── create_pcie() → DT 节点
  └── create_virtio_devices() → DT 节点

最终:
  virt_machine_done() [virt.c:2162-2200]
    ├── platform_bus_add_all_fdt_nodes()  — 动态设备节点
    └── machvirt_dtb() [virt.c:2119-2128] — 输出 DTB
```

#### DT 辅助函数

```c
qemu_fdt_add_subnode(fdt, nodepath)       // 添加子节点
qemu_fdt_setprop_string(fdt, node, prop, val) // 字符串属性
qemu_fdt_setprop_cells(fdt, node, prop, ...)  // 整数数组属性
qemu_fdt_setprop_sized_cells(fdt, node, prop, ...) // 变长整数
```

### 24. 热插拔机制

#### PCIe 热插拔

ARM virt 机器通过 ACPI GED（通用事件设备）支持 PCIe 热插拔：

```
create_acpi_ged() [virt.c:1021-1062]
  ├── 创建 TYPE_ACPI_GED 设备
  ├── 映射到 0x09080000
  ├── 连接 IRQ → GIC SPI 9
  └── 设置事件位:
        ├── ACPI_GED_MEM_HOTPLUG_EVT    — 内存热插拔
        ├── ACPI_GED_PCI_HOTPLUG_EVT    — PCIe 热插拔
        ├── ACPI_GED_NVDIMM_HOTPLUG_EVT — NVDIMM 热插拔
        └── ACPI_GED_CPU_HOTPLUG_EVT    — CPU 热插拔

热插拔流程:
  device_add virtio-blk-pci,drive=hd1 (QMP)
    │
    ├── qdev_device_add() → realize PCI 设备
    ├── hotplug_handler_plug()
    │     └── ACPI GED 生成 SCI 中断
    └── Guest ACPI handler 响应 → 枚举新设备
```

#### virtio-mmio 不支持热插拔

virtio-mmio 传输在 `machvirt_init()` 中预创建固定数量的 MMIO 插槽，没有动态热插拔机制。设备必须在启动时通过 `-device virtio-*-device` 附加。

#### ACPI vs DTB

| 方面 | DTB | ACPI |
|------|-----|------|
| 用途 | 静态设备描述 | 动态事件 + 热插拔 |
| 创建时机 | machvirt_init() | virt_machine_done() |
| 热插拔 | 不支持 | 通过 GED SCI 支持 |
| 适用场景 | Linux 直接内核启动 | UEFI 固件启动 |

---

## 附录

### A. 关键源文件索引

| 文件 | 内容 | 代码量级 |
|------|------|---------|
| hw/core/qdev.c | 设备核心生命周期 | ~900 行 |
| include/hw/core/qdev.h | DeviceClass/DeviceState/BusState | ~860 行 |
| include/hw/core/sysbus.h | SysBusDevice 定义 | ~90 行 |
| include/hw/pci/pci_device.h | PCIDevice 定义 | ~190 行 |
| hw/virtio/virtio.c | virtio 核心实现 | ~3500 行 |
| include/hw/virtio/virtio.h | VirtIODevice 定义 | ~400 行 |
| hw/virtio/virtio-mmio.c | virtio-mmio 传输 | ~800 行 |
| hw/virtio/virtio-pci.c | virtio-pci 传输 | ~2500 行 |
| hw/block/virtio-blk.c | virtio 块设备 | ~1500 行 |
| hw/net/virtio-net.c | virtio 网络设备 | ~4000 行 |
| hw/virtio/vhost.c | vhost 内核后端 | ~2000 行 |
| hw/virtio/vhost-user.c | vhost-user 后端 | ~2500 行 |
| hw/arm/virt.c | ARM virt 机器 | ~3000 行 |

### B. 关键 Git 提交

| Commit | 描述 | 影响 |
|--------|------|------|
| 12d1a768bd | class_init() data 参数改 const | 全设备 API 安全性 |
| 4913ae36f9 | virtio-blk zone report 缓冲区溢出修复 | CVE-2026-5761 安全修复 |
| ee4d1ff3af | virtio-mmio-transports 属性 | virt 机器 virtio 配置灵活性 |
| 1e9181dc52 | 统一 virtio_notify_irqfd/virtio_notify | 中断通知路径简化 |
| 1c79ab6937 | 默认启用 UDP tunnel GSO | virtio-net 性能 |
| 37863fff59 | HVF vGIC 默认启用 | Apple Silicon 虚拟化 |
| e0822e6085 | vhost-user 跳过 drain 协议 | 迁移性能优化 |
| 843a97fa2c | virtio_reset 传递 VirtIODevice* | API 改进 |
| 64495d7cfe | 添加 virtio_vdev_is_legacy() | Legacy 模式判断封装 |

### C. 推荐深入方向

1. **virtio-gpu 与显示子系统** — GPU 命令解析、2D/3D 渲染、virgl 加速
2. **vhost-vDPA** — 硬件加速 virtio 的新范式
3. **ACPI 表生成** — ARM virt 的 DSDT/SSDT 构建
4. **PCI 设备直通**（VFIO） — 设备直通与 IOMMU 集成
5. **virtio-fs** — 文件系统共享，virtiofsd 后端
