# 21 - VirtIO 设备模型深度分析 — VirtQueue、VRing、通知机制、PCI/MMIO 传输与 vhost 加速

> **基于 QEMU 11.0.50 源码**，深入分析 VirtIO 子系统的完整实现：
> 从核心数据结构（VirtIODevice/VirtQueue/VRing）到描述符链处理、
> 中断抑制、Feature 协商、PCI/MMIO 传输层，以及 vhost 内核/用户态加速。

---

## 目录

1. [VirtIODevice 与 VirtioDeviceClass：设备核心](#1-virtiodevice-与-virtiodeviceclass设备核心)
2. [VRing 数据结构：Split 与 Packed 格式](#2-vring-数据结构split-与-packed-格式)
3. [VirtQueue：队列状态与索引管理](#3-virtqueue队列状态与索引管理)
4. [描述符链处理：Pop、Fill、Flush](#4-描述符链处理popfillflush)
5. [通知与中断：Kick、Notify 与事件抑制](#5-通知与中断kicknotify-与事件抑制)
6. [Feature 协商与状态机](#6-feature-协商与状态机)
7. [VirtIO 总线层：VirtioBus](#7-virtio-总线层virtiobus)
8. [VirtIO-PCI 传输层：Modern 与 Legacy](#8-virtio-pci-传输层modern-与-legacy)
9. [VirtIO-MMIO 传输层](#9-virtio-mmio-传输层)
10. [vhost 加速框架：核心架构](#10-vhost-加速框架核心架构)
11. [vhost 后端：Kernel、User、vDPA](#11-vhost-后端kerneluservdpa)
12. [vhost-net：网络数据面加速](#12-vhost-net网络数据面加速)
13. [典型设备：virtio-net 与 virtio-blk](#13-典型设备virtio-net-与-virtio-blk)

---

## 1. VirtIODevice 与 VirtioDeviceClass：设备核心

### 1.1 VirtIODevice — 设备实例

```c
// virtio.h:108-171
struct VirtIODevice {
    DeviceState parent_obj;
    const char *name;
    uint8_t status;                // VIRTIO_CONFIG_S_* 状态位
    uint8_t isr;                   // 中断状态寄存器
    uint16_t queue_sel;            // 当前选中队列

    /* Feature 三层模型 */
    VIRTIO_DECLARE_FEATURES(host_features);     // 设备可提供的全部特性
    VIRTIO_DECLARE_FEATURES(guest_features);    // 驱动协商选择的特性
    VIRTIO_DECLARE_FEATURES(backend_features);  // vhost 后端支持的特性

    size_t config_len;             // 设备配置空间大小
    void *config;                  // 设备配置空间数据
    uint16_t config_vector;        // 配置变更 MSI-X 向量
    uint32_t generation;           // 配置 generation 计数
    int nvectors;                  // MSI-X 向量数
    VirtQueue *vq;                 // 队列数组（VIRTIO_QUEUE_MAX = 1024）
    QLIST_HEAD(, VirtQueue) *vector_queues; // 按 MSI-X 向量分组的队列链表
    uint16_t device_id;            // VirtIO 设备 ID
    bool vm_running;               // VM 运行状态
    bool broken;                   // 设备异常，需复位
    bool started;                  // DRIVER_OK 已设置
    bool start_on_kick;            // virtio 1.0 之前首次 kick 时启动
    bool vhost_started;            // vhost 后端已启动
    uint8_t device_endian;         // 设备字节序
    bool use_guest_notifier_mask;  // 是否使用通知掩码
    AddressSpace *dma_as;          // DMA 地址空间
    EventNotifier config_notifier; // 配置变更通知器
    bool device_iotlb_enabled;
};
```

**设计要点**：
- **Feature 三层模型**：`host_features`（设备能力）→ 协商 → `guest_features`（最终共识），`backend_features` 用于 vhost
- **1024 个队列槽位**：`vq` 数组在 `virtio_init()` 时一次性分配
- **generation 计数**：每次配置变更递增，驱动可检测配置是否变化

### 1.2 VirtioDeviceClass — 设备类虚函数表

```c
// virtio.h:173-241
struct VirtioDeviceClass {
    DeviceClass parent;

    /* 核心生命周期 */
    DeviceRealize realize;
    DeviceUnrealize unrealize;

    /* Feature 协商 */
    void (*get_features_ex)(VirtIODevice *vdev, uint64_t *requested, Error **errp);
    void (*set_features_ex)(VirtIODevice *vdev, const uint64_t *val);
    uint64_t (*get_features)(VirtIODevice *vdev, uint64_t requested, Error **errp);
    void (*set_features)(VirtIODevice *vdev, uint64_t val);
    int (*validate_features)(VirtIODevice *vdev);

    /* 配置空间 */
    void (*get_config)(VirtIODevice *vdev, uint8_t *config);
    void (*set_config)(VirtIODevice *vdev, const uint8_t *config);

    /* 状态管理 */
    void (*reset)(VirtIODevice *vdev);
    int (*set_status)(VirtIODevice *vdev, uint8_t val);
    void (*queue_reset)(VirtIODevice *vdev, uint32_t queue_index);
    void (*queue_enable)(VirtIODevice *vdev, uint32_t queue_index);

    /* 通知 */
    bool (*guest_notifier_pending)(VirtIODevice *vdev, int n);
    void (*guest_notifier_mask)(VirtIODevice *vdev, int n, bool mask);
    int (*start_ioeventfd)(VirtIODevice *vdev);
    void (*stop_ioeventfd)(VirtIODevice *vdev);

    /* vhost */
    struct vhost_dev *(*get_vhost)(VirtIODevice *vdev);

    /* 迁移 */
    void (*save)(VirtIODevice *vdev, QEMUFile *f);
    int (*load)(VirtIODevice *vdev, QEMUFile *f, int version_id);
    int (*post_load)(VirtIODevice *vdev);
    const VMStateDescription *vmsd;
};
```

### 1.3 virtio_init — 设备初始化

```c
// virtio.c:3568-3609
void virtio_init(VirtIODevice *vdev, uint16_t device_id, size_t config_size)
{
    // 1. 分配 MSI-X 向量→队列映射表
    vdev->vector_queues = g_malloc0(sizeof(...) * nvectors);

    // 2. 分配 VIRTIO_QUEUE_MAX 个队列
    vdev->vq = g_new0(VirtQueue, VIRTIO_QUEUE_MAX);
    for (i = 0; i < VIRTIO_QUEUE_MAX; i++) {
        vdev->vq[i].vector = VIRTIO_NO_VECTOR;
        vdev->vq[i].vdev = vdev;
        vdev->vq[i].queue_index = i;
    }

    // 3. 初始化状态
    vdev->device_id = device_id;
    vdev->status = 0;
    vdev->config_vector = VIRTIO_NO_VECTOR;
    vdev->config_len = config_size;
    vdev->config = config_size ? g_malloc0(config_size) : NULL;

    // 4. 注册 VM 状态变更回调
    vdev->vmstate = qdev_add_vm_change_state_handler(DEVICE(vdev),
                        NULL, virtio_vmstate_change, vdev);
}
```

---

## 2. VRing 数据结构：Split 与 Packed 格式

### 2.1 Split VRing 结构

```c
// virtio.c:65-116
typedef struct VRingDesc {           // 描述符表条目（16 字节）
    uint64_t addr;                   // Guest 物理地址
    uint32_t len;                    // 数据长度
    uint16_t flags;                  // NEXT/WRITE/INDIRECT 标志
    uint16_t next;                   // 链表下一个描述符索引
} VRingDesc;

typedef struct VRingAvail {          // Available Ring（驱动→设备）
    uint16_t flags;                  // VRING_AVAIL_F_NO_INTERRUPT
    uint16_t idx;                    // 下一个可用描述符索引
    uint16_t ring[];                 // 描述符头索引数组
} VRingAvail;

typedef struct VRingUsedElem {       // Used Ring 条目
    uint32_t id;                     // 描述符链头索引
    uint32_t len;                    // 设备写入的总字节数
} VRingUsedElem;

typedef struct VRingUsed {           // Used Ring（设备→驱动）
    uint16_t flags;                  // VRING_USED_F_NO_NOTIFY
    uint16_t idx;                    // 下一个 used 索引
    VRingUsedElem ring[];            // used 条目数组
} VRingUsed;

typedef struct VRing {
    unsigned int num;                // 队列大小（描述符数）
    unsigned int num_default;        // 默认大小
    unsigned int align;              // 对齐（4096）
    hwaddr desc;                     // 描述符表 GPA
    hwaddr avail;                    // Available Ring GPA
    hwaddr used;                     // Used Ring GPA
    VRingMemoryRegionCaches *caches; // 内存区域缓存
} VRing;
```

### 2.2 Packed VRing 格式

```c
// virtio.c:73-78, 118-121
typedef struct VRingPackedDesc {     // Packed 描述符（16 字节）
    uint64_t addr;
    uint32_t len;
    uint16_t id;                     // 描述符 ID（替代 next 链）
    uint16_t flags;                  // 含 AVAIL/USED 位 + wrap 计数
} VRingPackedDesc;

typedef struct VRingPackedDescEvent { // 事件抑制结构
    uint16_t off_wrap;               // 偏移 + wrap 计数
    uint16_t flags;                  // ENABLE/DISABLE/DESC
} VRingPackedDescEvent;
```

**Split vs Packed 对比**：
| 特性 | Split | Packed |
|------|-------|--------|
| 环形结构 | 3 个独立区域（Desc/Avail/Used） | 单一描述符数组 |
| 可用性标记 | Avail Ring 索引 | 描述符内 AVAIL/USED 位 |
| 链接方式 | `next` 字段链表 | 顺序排列或 `id` 标识 |
| Cache 友好性 | 差（3 次内存访问） | 好（单次访问） |
| Wrap 计数 | 无 | `wrap_counter` 翻转 |

---

## 3. VirtQueue：队列状态与索引管理

```c
// virtio.c:123-159
struct VirtQueue {
    VRing vring;                     // VRing 配置
    VirtQueueElement *used_elems;    // used 元素缓冲

    /* 消费端索引 */
    uint16_t last_avail_idx;         // 下一个要 pop 的 avail 索引
    bool last_avail_wrap_counter;    // Packed 模式 wrap 计数

    uint16_t shadow_avail_idx;       // 缓存的 avail idx（减少内存读取）
    bool shadow_avail_wrap_counter;

    /* 生产端索引 */
    uint16_t used_idx;               // 下一个 used 写入位置
    bool used_wrap_counter;          // Packed 模式 used wrap

    /* 中断抑制 */
    uint16_t signalled_used;         // 上次通知时的 used 索引
    bool signalled_used_valid;
    bool notification;               // 是否允许通知

    uint16_t queue_index;            // 队列编号
    unsigned int inuse;              // 当前使用中的描述符数
    uint16_t vector;                 // MSI-X 向量

    VirtIOHandleOutput handle_output; // 队列处理回调
    VirtIODevice *vdev;

    EventNotifier guest_notifier;    // Guest → Host 通知（irqfd）
    EventNotifier host_notifier;     // Host → Guest 通知（ioeventfd）
    bool host_notifier_enabled;
};
```

### 3.1 添加队列

```c
// virtio.c:2558-2578
VirtQueue *virtio_add_queue(VirtIODevice *vdev, int queue_size,
                            VirtIOHandleOutput handle_output)
{
    // 找到第一个 num == 0 的空闲槽位
    for (i = 0; i < VIRTIO_QUEUE_MAX; i++)
        if (vdev->vq[i].vring.num == 0) break;

    vdev->vq[i].vring.num = queue_size;
    vdev->vq[i].vring.num_default = queue_size;
    vdev->vq[i].vring.align = VIRTIO_PCI_VRING_ALIGN;  // 4096
    vdev->vq[i].handle_output = handle_output;
    vdev->vq[i].used_elems = g_new0(VirtQueueElement, queue_size);
    return &vdev->vq[i];
}
```

---

## 4. 描述符链处理：Pop、Fill、Flush

### 4.1 VirtQueueElement — 描述符链抽象

```c
// virtio.h:66-79
typedef struct VirtQueueElement {
    unsigned int index;      // 描述符链头索引
    unsigned int len;        // 已处理长度
    unsigned int ndescs;     // 链中描述符数量
    unsigned int out_num;    // 设备只读 scatter 段数
    unsigned int in_num;     // 设备可写 scatter 段数
    bool in_order_filled;
    hwaddr *in_addr;         // 可写段 GPA 数组
    hwaddr *out_addr;        // 只读段 GPA 数组
    struct iovec *in_sg;     // 可写段 iovec（HVA）
    struct iovec *out_sg;    // 只读段 iovec（HVA）
} VirtQueueElement;
```

### 4.2 virtqueue_split_pop — Split 模式消费描述符

```c
// virtio.c:1733-1878（关键流程）
static void *virtqueue_split_pop(VirtQueue *vq, size_t sz)
{
    // 1. 检查队列是否为空
    if (virtio_queue_empty_rcu(vq)) return NULL;
    smp_rmb();  // 内存屏障：确保读到最新 avail idx

    // 2. 从 Available Ring 获取描述符头索引
    virtqueue_get_head(vq, vq->last_avail_idx++, &head);

    // 3. 更新 EVENT_IDX（告诉驱动下次通知的位置）
    if (VIRTIO_RING_F_EVENT_IDX)
        vring_set_avail_event(vq, vq->last_avail_idx);

    // 4. 读取第一个描述符
    vring_split_desc_read(vdev, &desc, desc_cache, head);

    // 5. 间接描述符处理
    if (desc.flags & VRING_DESC_F_INDIRECT) {
        // 映射间接描述符表
        address_space_cache_init(&indirect_desc_cache, vdev->dma_as,
                                 desc.addr, desc.len, false);
        max = desc.len / sizeof(VRingDesc);
        i = 0;
    }

    // 6. 遍历描述符链
    do {
        if (desc.flags & VRING_DESC_F_WRITE)
            // 设备可写 → in_sg
        else
            // 设备只读 → out_sg

        // 映射 Guest 内存
        virtqueue_map_desc(vdev, &in_num, addr, iov, ...);
    } while (desc.flags & VRING_DESC_F_NEXT);

    // 7. 构造 VirtQueueElement
    elem = virtqueue_alloc_element(sz, out_num, in_num);
    elem->index = head;
    vq->inuse++;
}
```

### 4.3 Fill / Flush — 写回 Used Ring

**Fill**（写单个条目到 Used Ring）：
```c
// virtio.c:954-968 (split)
static void virtqueue_split_fill(VirtQueue *vq, const VirtQueueElement *elem,
                                 unsigned int len, unsigned int idx)
{
    idx = (idx + vq->used_idx) % vq->vring.num;
    uelem.id = elem->index;  // 描述符链头索引
    uelem.len = len;          // 设备写入的总字节数
    vring_used_write(vq, &uelem, idx);
}
```

**Flush**（更新 Used Ring idx，使驱动可见）：
```
virtqueue_push(vq, elem, len)
  → virtqueue_fill(vq, elem, len, idx)    // 写 used 条目
  → virtqueue_flush(vq, 1)                 // 递增 used_idx
    → smp_wmb()                            // 写屏障
    → vring_used_idx_set(vq, vq->used_idx) // 更新 Guest 可见的 used.idx
```

---

## 5. 通知与中断：Kick、Notify 与事件抑制

### 5.1 Guest → Device（Kick）

```c
// virtio.c:2515-2533
void virtio_queue_notify(VirtIODevice *vdev, int n)
{
    VirtQueue *vq = &vdev->vq[n];
    if (!vq->vring.desc || vdev->broken) return;

    if (vq->host_notifier_enabled)
        event_notifier_set(&vq->host_notifier);  // ioeventfd 路径
    else if (vq->handle_output)
        vq->handle_output(vdev, vq);              // 直接调用处理回调
}
```

**ioeventfd 加速**：Guest 写入 Notify 寄存器 → KVM ioeventfd → EventNotifier → QEMU 事件循环 → `handle_output()`。避免每次 kick 都 VM Exit 到 QEMU 主循环。

### 5.2 Device → Guest（Notify / 中断注入）

```c
// virtio.c:2730-2740
void virtio_notify(VirtIODevice *vdev, VirtQueue *vq)
{
    // 中断抑制检查
    if (!virtio_should_notify(vdev, vq)) return;

    virtio_irq(vq);  // 注入中断
}

// virtio.c:2698-2728
static void virtio_irq(VirtQueue *vq)
{
    virtio_set_isr(vq->vdev, 0x1);  // 设置 ISR bit 0

    if (qemu_in_iothread())
        defer_call(virtio_notify_irqfd_deferred_fn, &vq->guest_notifier);
    else
        virtio_notify_vector(vq->vdev, vq->vector);  // → 传输层通知
}
```

### 5.3 中断抑制（Event Index）

```c
// virtio.c:2679-2686
static bool virtio_should_notify(VirtIODevice *vdev, VirtQueue *vq)
{
    if (VIRTIO_F_RING_PACKED)
        return virtio_packed_should_notify(vdev, vq);
    else
        return virtio_split_should_notify(vdev, vq);
}
```

**Split 模式抑制**（virtio.c:2612-2633）：
- `VIRTIO_RING_F_EVENT_IDX`：驱动在 avail ring 末尾写入"下次通知阈值"，设备仅在 used_idx 越过该阈值时才通知
- `VRING_USED_F_NO_NOTIFY`：驱动完全禁止通知

**Packed 模式抑制**（virtio.c:2649-2676）：
- `VRING_PACKED_EVENT_FLAG_ENABLE/DISABLE/DESC`：三种模式
- `DESC` 模式：基于 off_wrap 精确控制

### 5.4 配置变更通知

```c
// virtio.c:2742-2755
void virtio_notify_config(VirtIODevice *vdev)
{
    if (!(vdev->status & VIRTIO_CONFIG_S_DRIVER_OK)) return;
    virtio_set_isr(vdev, 0x3);    // ISR bit 0 + bit 1（配置变更）
    vdev->generation++;             // 递增 generation
    virtio_notify_vector(vdev, vdev->config_vector);
}
```

---

## 6. Feature 协商与状态机

### 6.1 设备状态位

```
VIRTIO_CONFIG_S_ACKNOWLEDGE = 1    Guest OS 已识别设备
VIRTIO_CONFIG_S_DRIVER      = 2    Guest 驱动已加载
VIRTIO_CONFIG_S_FEATURES_OK = 8    Feature 协商完成
VIRTIO_CONFIG_S_DRIVER_OK   = 4    驱动准备就绪，设备可工作
VIRTIO_CONFIG_S_NEEDS_RESET = 64   设备需要复位
VIRTIO_CONFIG_S_FAILED      = 128  驱动初始化失败
```

### 6.2 状态转换处理

```c
// virtio.c:2282-2313
int virtio_set_status(VirtIODevice *vdev, uint8_t val)
{
    // 1. FEATURES_OK 转换时验证 Feature
    if (VIRTIO_F_VERSION_1 && !(old & FEATURES_OK) && (val & FEATURES_OK))
        ret = virtio_validate_features(vdev);

    // 2. DRIVER_OK 转换时设置 started 状态
    if ((old & DRIVER_OK) != (val & DRIVER_OK))
        virtio_set_started(vdev, val & DRIVER_OK);

    // 3. 调用设备特定 set_status
    if (k->set_status)
        ret = k->set_status(vdev, val);

    vdev->status = val;
}
```

### 6.3 设备复位

```c
// virtio.c:3244-3286
void virtio_reset(VirtIODevice *vdev)
{
    virtio_set_status(vdev, 0);           // 清除所有状态
    vdev->device_endian = ...;            // 重置字节序

    // 复位 vhost 后端
    if (k->get_vhost) vhost_reset_device(hdev);

    // 调用设备特定 reset
    if (k->reset) k->reset(vdev);

    // 清除所有运行时状态
    vdev->started = false;
    vdev->broken = false;
    virtio_set_features_nocheck(vdev, zeros);
    vdev->status = 0;
    vdev->isr = 0;
    vdev->config_vector = VIRTIO_NO_VECTOR;

    // 复位所有队列
    for (i = 0; i < VIRTIO_QUEUE_MAX; i++)
        __virtio_queue_reset(vdev, i);
}
```

### 6.4 Feature 协商流程

```
1. virtio_bus_device_plugged()                  [virtio-bus.c:43-104]
     → vdc->get_features_ex() / get_features()  // 收集设备 host_features
     → 传输层可能添加 IOMMU_PLATFORM 等

2. Guest 驱动读取 host_features

3. Guest 驱动写入选择的 features
     → virtio_set_features_ex()                  [virtio.c:3208-3242]
       → 与 host_features 取交集
       → vdc->set_features_ex()                  // 通知设备
       → 更新 EVENT_IDX 等缓存

4. Guest 写 FEATURES_OK 到 status
     → virtio_validate_features()                // 最终验证
```

---

## 7. VirtIO 总线层：VirtioBus

### 7.1 VirtioBusClass — 传输层接口

```c
// virtio-bus.h:39-97
struct VirtioBusClass {
    BusClass parent;
    void (*notify)(DeviceState *d, uint16_t vector);  // 中断注入
    void (*pre_plugged)(DeviceState *d, Error **errp);
    void (*device_plugged)(DeviceState *d, Error **errp);
    void (*device_unplugged)(DeviceState *d);
    int (*ioeventfd_assign)(DeviceState *d, EventNotifier *n, int idx, bool assign);
    bool (*ioeventfd_enabled)(DeviceState *d);
    AddressSpace *(*get_dma_as)(DeviceState *d);
    bool (*iommu_enabled)(DeviceState *d);
    bool (*queue_enabled)(DeviceState *d, int n);
    int (*query_nvectors)(DeviceState *d);
};
```

### 7.2 设备插入流程

```c
// virtio-bus.c:43-104
void virtio_bus_device_plugged(VirtIODevice *vdev)
{
    // 1. 调用传输层 pre_plugged
    k->pre_plugged(qbus->parent, errp);

    // 2. 获取设备 features
    vdc->get_features_ex(vdev, &features, errp);

    // 3. 调用传输层 device_plugged
    k->device_plugged(qbus->parent, errp);

    // 4. 设置 DMA 地址空间
    vdev->dma_as = k->get_dma_as ? k->get_dma_as(qbus->parent)
                                  : &address_space_memory;
}
```

### 7.3 ioeventfd 管理

```c
// virtio-bus.c:224-267
int virtio_bus_start_ioeventfd(VirtioBusState *bus)
{
    // 需要传输层支持 ioeventfd_assign + ioeventfd_enabled
    // 调用 vdc->start_ioeventfd(vdev)
    bus->ioeventfd_started = true;
}

// virtio-bus.c:281-315
int virtio_bus_set_host_notifier(VirtioBusState *bus, int n, bool assign)
{
    // 初始化/销毁 EventNotifier
    event_notifier_init(&vq->host_notifier, true);
    // 调用传输层 ioeventfd_assign
    k->ioeventfd_assign(proxy, &vq->host_notifier, n, assign);
}
```

---

## 8. VirtIO-PCI 传输层：Modern 与 Legacy

### 8.1 VirtIOPCIProxy 结构

```c
// virtio-pci.h:115-154
struct VirtIOPCIProxy {
    PCIDevice pci_dev;              // PCI 设备基类
    MemoryRegion bar;               // Legacy IO BAR
    union {
        struct {
            VirtIOPCIRegion common;  // Common Configuration
            VirtIOPCIRegion isr;     // ISR Status
            VirtIOPCIRegion device;  // Device-specific Config
            VirtIOPCIRegion notify;  // Queue Notify（MMIO）
            VirtIOPCIRegion notify_pio; // Queue Notify（PIO）
        };
        VirtIOPCIRegion regs[5];
    };
    MemoryRegion modern_bar;        // Modern BAR
    MemoryRegion io_bar;            // IO BAR

    uint32_t nvectors;              // MSI-X 向量数
    uint32_t dfselect;              // Device Feature 选择器
    uint32_t gfselect;              // Guest Feature 选择器
    uint32_t guest_features[...];   // Guest 选择的 feature

    VirtIOPCIQueue vqs[VIRTIO_QUEUE_MAX]; // 每队列 PCI 状态
    VirtIOIRQFD *vector_irqfd;      // KVM irqfd 数组
    VirtioBusState bus;              // VirtIO 总线实例
};
```

### 8.2 Modern BAR 布局

```
偏移 0x0000: Common Configuration    — 设备状态/Feature/队列配置
偏移 0x1000: ISR Status              — 中断状态读取（自动清除）
偏移 0x2000: Device-specific Config  — 设备特定配置空间
偏移 0x3000: Queue Notify            — 队列 Kick（MMIO 写入）
```

每个区域通过 PCI Capability 结构描述位置和大小。

### 8.3 PCI 中断投递

```c
// virtio-pci.c:73-85
static void virtio_pci_notify(DeviceState *d, uint16_t vector)
{
    VirtIOPCIProxy *proxy = to_virtio_pci_proxy_fast(d);

    if (msix_enabled(&proxy->pci_dev)) {
        // MSI-X 路径
        if (vector != VIRTIO_NO_VECTOR)
            msix_notify(&proxy->pci_dev, vector);
    } else {
        // Legacy INTx 路径
        VirtIODevice *vdev = virtio_bus_get_device(&proxy->bus);
        pci_set_irq(&proxy->pci_dev, qatomic_read(&vdev->isr) & 1);
    }
}
```

### 8.4 Modern Common Configuration 读写

```c
// virtio-pci.c:1538-1630 (read), 1633-1767 (write)
```

**读**操作处理：
- `DFSELECT/DF`：返回 host_features 的高/低 32 位
- `GFSELECT/GF`：返回已协商的 guest_features
- `MSIX_CONFIG`：配置变更 MSI-X 向量
- `NUM_QUEUES`：队列总数
- `STATUS`：设备状态
- `CONFIG_GENERATION`：配置 generation
- `Q_SIZE/Q_MSIX/Q_ENABLE/Q_NOFF/Q_DESC/Q_AVAIL/Q_USED`：队列参数

**写**操作处理：
- Feature 协商通过 `GFSELECT` + `GF` 分批写入
- `STATUS` 写入触发 `virtio_set_status()`，状态 0 触发复位
- 队列地址通过 `Q_DESC_LO/HI`、`Q_AVAIL_LO/HI`、`Q_USED_LO/HI` 设置
- `Q_ENABLE` 激活队列

### 8.5 Notify 写入（Kick）

```c
// virtio-pci.c:1782-1807
static void virtio_pci_notify_write(void *opaque, hwaddr addr,
                                    uint64_t val, unsigned size)
{
    // addr / notify_offset_multiplier → queue index
    virtio_queue_notify(vdev, val);  // 或通过 NOTIFICATION_DATA 解码
}
```

---

## 9. VirtIO-MMIO 传输层

### 9.1 寄存器布局

| 偏移 | 寄存器 | 方向 | 说明 |
|------|--------|------|------|
| 0x000 | MAGIC_VALUE | R | "virt" (0x74726976) |
| 0x004 | VERSION | R | 1(Legacy) 或 2(Modern) |
| 0x008 | DEVICE_ID | R | VirtIO 设备 ID |
| 0x00C | VENDOR_ID | R | 子系统 Vendor ID |
| 0x010 | DEVICE_FEATURES | R | 设备 Feature（由 sel 选择） |
| 0x020 | DRIVER_FEATURES | W | 驱动 Feature 写入 |
| 0x030 | QUEUE_SEL | W | 队列选择器 |
| 0x044 | QUEUE_READY | RW | 队列就绪状态 |
| 0x050 | QUEUE_NOTIFY | W | 队列 Kick |
| 0x060 | INTERRUPT_STATUS | R | 中断状态 |
| 0x064 | INTERRUPT_ACK | W | 中断确认 |
| 0x070 | STATUS | RW | 设备状态 |
| 0x100+ | CONFIG | RW | 设备特定配置 |

### 9.2 MMIO vs PCI 对比

| 特性 | VirtIO-MMIO | VirtIO-PCI |
|------|-------------|------------|
| 发现机制 | 设备树/ACPI | PCI 枚举 |
| 中断 | 平台 IRQ（SPI） | MSI-X / INTx |
| 配置访问 | 寄存器偏移 | PCI Capability + BAR |
| 多设备 | 每设备独立 MMIO 区域 | 共享 PCI 总线 |
| Feature 宽度 | 32 位分批选择 | 32 位分批选择 |
| 热插拔 | 不支持 | 支持 |

---

## 10. vhost 加速框架：核心架构

### 10.1 vhost_dev — 后端设备状态

```c
// vhost.h:79-137
struct vhost_dev {
    VirtIODevice *vdev;                    // 关联的 VirtIO 设备
    MemoryListener memory_listener;        // 内存变更监听
    MemoryListener iommu_listener;         // IOMMU 变更监听
    struct vhost_memory *mem;              // 内存表
    struct vhost_virtqueue *vqs;           // vhost 队列数组
    unsigned int nvqs;                     // 队列数

    VIRTIO_DECLARE_FEATURES(features);         // 后端支持的特性
    VIRTIO_DECLARE_FEATURES(acked_features);   // 已协商特性
    VIRTIO_DECLARE_FEATURES(backend_features);

    uint64_t protocol_features;            // vhost-user 协议特性
    uint64_t max_queues;
    bool started;                          // 后端已启动
    bool log_enabled;                      // 脏页日志
    uint64_t log_size;
    Error *migration_blocker;
    const VhostOps *vhost_ops;             // 后端操作函数表
    void *opaque;
    struct vhost_log *log;
};
```

### 10.2 vhost_dev_init — 初始化

```c
// vhost.c:1552-1655
int vhost_dev_init(struct vhost_dev *hdev, void *opaque,
                   VhostBackendType backend_type, ...)
{
    // 1. 选择后端类型（kernel/user/vdpa）
    vhost_set_backend_type(hdev, backend_type);

    // 2. 后端初始化 + SET_OWNER
    hdev->vhost_ops->vhost_backend_init(hdev, opaque);

    // 3. 读取后端特性
    hdev->vhost_ops->vhost_get_features(hdev, &features);

    // 4. 初始化队列
    hdev->vqs = g_new0(struct vhost_virtqueue, hdev->nvqs);

    // 5. 注册内存监听器
    memory_listener_register(&hdev->memory_listener, ...);
}
```

### 10.3 vhost_dev_start — 启动数据面

```c
// vhost.c:2109-2231
int vhost_dev_start(struct vhost_dev *hdev, VirtIODevice *vdev, bool vrings)
{
    hdev->started = true;

    // 1. 设置特性
    vhost_set_features(hdev);

    // 2. 注册 IOMMU 监听（如需要）
    vhost_dev_set_iommu(hdev, vdev);

    // 3. 推送内存表
    vhost_set_mem_table(hdev);

    // 4. 启动所有队列
    for (i = 0; i < hdev->nvqs; i++)
        vhost_virtqueue_start(hdev, vdev, hdev->vqs + i, ...);

    // 5. 配置中断
    // 6. 可选：后端特定启动
}
```

### 10.4 vhost_virtqueue_start — 队列卸载

```c
// vhost.c:1257-1383
static int vhost_virtqueue_start(struct vhost_dev *dev, VirtIODevice *vdev,
                                  struct vhost_virtqueue *vq, ...)
{
    // 1. 获取队列地址和大小
    vhost_set_vring_num(dev, &state);       // SET_VRING_NUM
    vhost_set_vring_base(dev, &state);      // SET_VRING_BASE (last_avail_idx)

    // 2. 映射 desc/avail/used 内存
    vq->desc = vhost_memory_map(dev, ...);
    vq->avail = vhost_memory_map(dev, ...);
    vq->used = vhost_memory_map(dev, ...);

    // 3. 设置队列地址
    vhost_set_vring_addr(dev, &addr);       // SET_VRING_ADDR

    // 4. 设置 Kick fd（ioeventfd → vhost）
    file.fd = event_notifier_get_fd(virtio_queue_get_host_notifier(vvq));
    vhost_set_vring_kick(dev, &file);       // SET_VRING_KICK

    // 5. 设置 Call fd（vhost → irqfd）
    vhost_set_vring_call(dev, &file);       // SET_VRING_CALL
}
```

**数据面卸载后的路径**：
```
Guest kick → ioeventfd → vhost 内核线程处理
vhost 完成 → irqfd → KVM 直接注入中断 → Guest
（完全绕过 QEMU 用户态）
```

---

## 11. vhost 后端：Kernel、User、vDPA

### 11.1 vhost-kernel

```c
// vhost-backend.c:28-38
static int vhost_kernel_call(struct vhost_dev *dev, unsigned long int request,
                             void *arg)
{
    return ioctl(fd, request, arg);  // 直接 ioctl 到 /dev/vhost-net
}
```

操作函数表（vhost-backend.c:359-390）：`kernel_ops` 包含 `SET_MEM_TABLE`、`SET_VRING_KICK/CALL` 等 ioctl 封装。

### 11.2 vhost-user

通过 Unix Socket 与用户态后端通信（如 DPDK、SPDK）。

```c
// vhost-user.c:57-227
enum VhostUserRequest {
    VHOST_USER_GET_FEATURES, SET_FEATURES,
    SET_MEM_TABLE, SET_VRING_NUM, SET_VRING_ADDR,
    SET_VRING_BASE, GET_VRING_BASE,
    SET_VRING_KICK, SET_VRING_CALL,
    ...
};
```

消息格式：`VhostUserHeader`（request + flags + size）+ `payload`（包含 fd 传递）。

**关键操作**：
- `vhost_user_set_mem_table()`（vhost-user.c:997-1050）：通过 fd 传递共享内存
- `vhost_user_set_vring_kick/call()`（vhost-user.c:1365-1375）：传递 eventfd

### 11.3 vhost-vdpa

```c
// vhost-vdpa.c — vdpa_ops
```

通过 `/dev/vhost-vdpa-N` 设备操作硬件加速的 vDPA 设备，支持硬件直接处理 VirtQueue。

---

## 12. vhost-net：网络数据面加速

### 12.1 初始化

```c
// vhost_net.c:232-316
struct vhost_net *vhost_net_init(VhostNetOptions *options)
{
    // 1. 获取 TAP backend fd
    net->dev.nvqs = 2;  // TX + RX
    net->dev.vq_index = 0;

    // 2. 初始化 vhost 核心
    vhost_dev_init(&net->dev, options->opaque, options->backend_type, ...);
}
```

### 12.2 启动流程

```c
// vhost_net.c:412-505
int vhost_net_start(VirtIODevice *dev, NetClientState *ncs, ...)
{
    // 1. 设置 Guest Notifier（irqfd）
    // 2. 设置 Host Notifier（ioeventfd）
    // 3. 逐队列启动 vhost
    for (i = 0; i < nvhosts; i++)
        vhost_net_start_one(get_vhost_net(ncs[i].peer), dev);
}

// vhost_net.c:325-390
static int vhost_net_start_one(struct vhost_net *net, VirtIODevice *dev)
{
    vhost_dev_start(&net->dev, dev, true);
    // TAP 后端：绑定 fd 到各队列
    vhost_net_set_backend(&net->dev, &file);  // VHOST_NET_SET_BACKEND
}
```

**数据流（vhost-net-kernel）**：
```
Guest TX → VirtQueue kick → ioeventfd → vhost-net 内核线程
  → 从 VRing 读描述符 → 发送到 TAP fd → 物理网络
  → 写 Used Ring → irqfd → KVM 注入中断 → Guest

网络 RX → TAP fd → vhost-net 内核线程
  → 从 VRing 获取可用描述符 → 写入数据
  → 写 Used Ring → irqfd → Guest
```

---

## 13. 典型设备：virtio-net 与 virtio-blk

### 13.1 virtio-net

```c
// virtio-net.c:3867-3985 — virtio_net_device_realize()
// 创建 RX/TX 队列对 + 控制队列
virtio_net_add_queue(n, i);  // 每对创建 RX + TX 队列
virtio_add_queue(vdev, ..., virtio_net_handle_ctrl);  // 控制队列
```

队列结构：`n->vqs[i].rx_vq` + `n->vqs[i].tx_vq`，每对由 `virtio_net_add_queue()` 创建（virtio-net.c:2976-3000）。

### 13.2 virtio-blk

```c
// virtio-blk.c:1044-1058 — virtio_blk_handle_output()
// 处理块设备 I/O 请求
// virtio-blk.c:1536-1632 — 启动 dataplane / ioeventfd
```

使用 ioeventfd 实现 dataplane 模式，I/O 请求在独立 IOThread 中处理。

---

## 总结

### 数据流全景图

```
┌─────────┐                          ┌──────────────┐
│  Guest   │                          │  vhost 后端  │
│  Driver  │                          │ (kernel/user)│
└────┬─────┘                          └──────┬───────┘
     │ ① kick (写 Notify 寄存器)             │
     ▼                                       │
┌────────────────┐  ioeventfd    ┌───────────┴────┐
│ VirtIO 传输层  │ ────────────→ │ vhost 数据面   │
│ (PCI/MMIO)     │               │ 直接处理 VRing │
└────────────────┘               └───────────────┘
     │ (无 vhost 时)                     │
     ▼                                   │ ② 完成
┌────────────────┐                       │
│ QEMU 设备模型  │                       │
│ handle_output  │                       │
│ virtqueue_pop  │                       │
│ ...处理...     │                       │
│ virtqueue_push │                       │
│ virtio_notify  │                       │
└────────┬───────┘                       │
         │ ③ 中断                        │ irqfd
         ▼                               ▼
┌────────────────────────────────────────────┐
│            Guest 中断处理                   │
│  ISR 读取 → MSI-X/INTx → GIC → CPU        │
└────────────────────────────────────────────┘
```

### 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/hw/virtio/virtio.h` | ~250 | VirtIODevice (108-171)、VirtioDeviceClass (173-241)、VirtQueueElement (66-79) |
| `hw/virtio/virtio.c` | ~3700 | VRing 结构 (65-116)、VirtQueue (123-159)、virtio_init (3568)、virtio_add_queue (2558)、pop/fill/flush (954-2041)、notify (2730)、should_notify (2679)、set_status (2282)、reset (3244) |
| `include/hw/virtio/virtio-bus.h` | ~140 | VirtioBusClass (39-97)、VirtioBusState (99-116) |
| `hw/virtio/virtio-bus.c` | ~315 | device_plugged (43-104)、ioeventfd (224-315) |
| `include/hw/virtio/virtio-pci.h` | ~160 | VirtIOPCIProxy (115-154)、VirtIOPCIQueue (102-113) |
| `hw/virtio/virtio-pci.c` | ~2400 | PCI notify (73-85)、common read/write (1538-1767)、notify_write (1782)、isr_read (1810)、modern_regions_init (1882) |
| `hw/virtio/virtio-mmio.c` | ~665 | MMIO read (85-245)、write (247-524)、reset (644) |
| `include/hw/virtio/vhost.h` | ~140 | vhost_dev (79-137) |
| `hw/virtio/vhost.c` | ~2300 | vhost_dev_init (1552)、vhost_dev_start (2109)、virtqueue_start (1257)、mem_table (663) |
| `hw/virtio/vhost-backend.c` | ~390 | kernel_ops、ioctl 封装 |
| `hw/virtio/vhost-user.c` | ~3050 | vhost-user 协议、user_ops |
| `hw/net/vhost_net.c` | ~510 | vhost_net_init (232)、start (412)、start_one (325) |
| `hw/net/virtio-net.c` | ~4000 | virtio_net_device_realize (3867)、add_queue (2976) |
| `hw/block/virtio-blk.c` | ~1650 | handle_output (1044)、dataplane (1536) |
