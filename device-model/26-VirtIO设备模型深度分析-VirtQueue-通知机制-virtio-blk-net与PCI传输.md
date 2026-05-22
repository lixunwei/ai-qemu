# 26 - VirtIO 设备模型深度分析 — VirtQueue、通知机制、virtio-blk/net 与 PCI 传输

> **基于 QEMU 11.0.50 源码**，深入分析 QEMU VirtIO 子系统完整实现：
> VirtIODevice/VirtQueue 核心结构、VRing 描述符环、Split/Packed 两种队列模式、
> Guest↔Host 双向通知机制（ioeventfd/irqfd/defer_call）、VirtIO-PCI 传输层、
> virtio-blk 块设备与 virtio-net 网络设备实现、IOThread dataplane 集成。

---

## 目录

1. [VirtIODevice：设备核心结构](#1-virtiodevice设备核心结构)
2. [VirtIODeviceClass：设备类回调](#2-virtiodeviceclass设备类回调)
3. [VirtQueue：虚拟队列](#3-virtqueue虚拟队列)
4. [VRing：描述符环结构](#4-vring描述符环结构)
5. [VirtQueueElement：请求元素](#5-virtqueueelement请求元素)
6. [Virtqueue 操作循环](#6-virtqueue-操作循环)
7. [通知机制](#7-通知机制)
8. [Packed Virtqueue](#8-packed-virtqueue)
9. [VirtIO-PCI 传输层](#9-virtio-pci-传输层)
10. [VirtioBus：总线抽象](#10-virtiobus总线抽象)
11. [virtio-blk：块设备实现](#11-virtio-blk块设备实现)
12. [virtio-net：网络设备实现](#12-virtio-net网络设备实现)
13. [IOThread Dataplane 集成](#13-iothread-dataplane-集成)
14. [Feature 协商](#14-feature-协商)
15. [数据流全景图](#15-数据流全景图)

---

## 1. VirtIODevice：设备核心结构

```c
// virtio.h:108-171
struct VirtIODevice {
    DeviceState parent_obj;              // QOM 设备基类

    const char *name;                    // 设备名称
    uint8_t status;                      // 设备状态位
    uint8_t isr;                         // 中断状态寄存器
    uint16_t queue_sel;                  // 当前选中的队列

    /* Feature 集 */
    VIRTIO_DECLARE_FEATURES(host_features);     // 设备可提供的完整特性
    VIRTIO_DECLARE_FEATURES(guest_features);    // 驱动选中的特性
    VIRTIO_DECLARE_FEATURES(backend_features);  // 后端（vhost）支持的特性

    /* 配置空间 */
    size_t config_len;                   // 配置空间长度
    void *config;                        // 配置空间数据
    uint16_t config_vector;              // 配置变更的 MSI-X 向量
    uint32_t generation;                 // 配置代（原子读一致性）

    /* 队列 */
    int nvectors;                        // MSI-X 向量数
    VirtQueue *vq;                       // VirtQueue 数组
    QLIST_HEAD(, VirtQueue) *vector_queues;  // 按向量分组的队列

    /* 状态 */
    uint16_t device_id;                  // 设备类型 ID
    bool vm_running;                     // VM 运行状态
    bool broken;                         // 设备异常需重置
    bool disabled;                       // 临时禁用
    bool use_started;                    // 使用 started 标志检查状态
    bool started;                        // 设备已启动
    bool start_on_kick;                  // virtio 1.0 未协商时 kick 启动
    bool vhost_started;                  // vhost 后端已启动
    bool use_guest_notifier_mask;        // 使用 guest_notifier_mask 回调
    bool device_iotlb_enabled;           // 设备 IOTLB 启用

    MemoryListener listener;             // IOMMU 内存监听器
    AddressSpace *dma_as;                // DMA 地址空间
    EventNotifier config_notifier;       // 配置变更通知器
};
```

---

## 2. VirtIODeviceClass：设备类回调

```c
// virtio.h:173-241（关键回调摘录）
struct VirtIODeviceClass {
    /* 生命周期 */
    DeviceRealize realize;                     // 设备实现（初始化）
    DeviceUnrealize unrealize;                 // 设备销毁

    /* Feature 协商 */
    uint64_t (*get_features)(VirtIODevice *, uint64_t, Error **);
    uint64_t (*bad_features)(VirtIODevice *);  // 需要过滤的特性
    void (*set_features)(VirtIODevice *, uint64_t);
    int (*validate_features)(VirtIODevice *, Error **);

    /* 配置空间 */
    void (*get_config)(VirtIODevice *, uint8_t *);
    void (*set_config)(VirtIODevice *, const uint8_t *);

    /* 状态与复位 */
    void (*set_status)(VirtIODevice *, uint8_t);
    void (*reset)(VirtIODevice *);
    void (*queue_reset)(VirtIODevice *, uint32_t);
    void (*queue_enable)(VirtIODevice *, uint32_t);

    /* 通知 */
    bool (*guest_notifier_pending)(VirtIODevice *, int);
    void (*guest_notifier_mask)(VirtIODevice *, int, bool);

    /* ioeventfd */
    void (*start_ioeventfd)(VirtIODevice *);
    void (*stop_ioeventfd)(VirtIODevice *);

    /* 迁移 */
    void (*save)(VirtIODevice *, QEMUFile *);
    int (*load)(VirtIODevice *, QEMUFile *, int);

    /* vhost */
    struct vhost_dev *(*get_vhost)(VirtIODevice *);
    void (*toggle_device_iotlb)(VirtIODevice *);
};
```

---

## 3. VirtQueue：虚拟队列

```c
// virtio.c:123-159
struct VirtQueue {
    VRing vring;                         // Vring 描述符环
    VirtQueueElement *used_elems;        // Used 元素缓存

    /* 消费者索引 */
    uint16_t last_avail_idx;             // 下一个要取的 avail 索引
    bool last_avail_wrap_counter;        // Packed 环绕计数器

    uint16_t shadow_avail_idx;           // 缓存的 avail idx（减少 MMIO）
    bool shadow_avail_wrap_counter;

    uint16_t used_idx;                   // 已填充的 used 索引
    bool used_wrap_counter;              // Packed used 环绕计数器

    /* 通知抑制 */
    uint16_t signalled_used;             // 上次发信号时的 used 值
    bool signalled_used_valid;           // signalled_used 是否有效
    bool notification;                   // 通知是否启用

    uint16_t queue_index;                // 队列编号
    unsigned int inuse;                  // 正在使用的描述符数

    uint16_t vector;                     // MSI-X 中断向量
    VirtIOHandleOutput handle_output;    // 处理 Guest kick 的回调
    VirtIODevice *vdev;                  // 所属设备

    EventNotifier guest_notifier;        // Host→Guest 通知（irqfd）
    EventNotifier host_notifier;         // Guest→Host 通知（ioeventfd）
    bool host_notifier_enabled;
};
```

---

## 4. VRing：描述符环结构

### 4.1 Split Ring（传统格式）

```c
// virtio.c:65-116
typedef struct VRingDesc {           // 描述符
    uint64_t addr;                   // Guest 物理地址
    uint32_t len;                    // 长度
    uint16_t flags;                  // NEXT/WRITE/INDIRECT
    uint16_t next;                   // 链中下一个描述符索引
} VRingDesc;

typedef struct VRingAvail {          // Available 环（Guest→Host）
    uint16_t flags;                  // 通知抑制标志
    uint16_t idx;                    // 下一个可用条目索引
    uint16_t ring[];                 // 描述符头索引数组
} VRingAvail;

typedef struct VRingUsedElem {       // Used 元素
    uint32_t id;                     // 描述符链头索引
    uint32_t len;                    // 已写入长度
} VRingUsedElem;

typedef struct VRingUsed {           // Used 环（Host→Guest）
    uint16_t flags;                  // 通知抑制标志
    uint16_t idx;                    // 下一个使用条目索引
    VRingUsedElem ring[];            // 已完成元素数组
} VRingUsed;

typedef struct VRing {               // 完整 Vring
    unsigned int num;                // 队列大小（描述符数）
    unsigned int num_default;        // 默认大小
    unsigned int align;              // 对齐
    hwaddr desc;                     // 描述符表 GPA
    hwaddr avail;                    // Avail 环 GPA
    hwaddr used;                     // Used 环 GPA
    VRingMemoryRegionCaches *caches; // 地址缓存（加速访问）
} VRing;
```

### 4.2 内存布局

```
GPA:
  desc_addr  ┌──────────────────┐
             │  VRingDesc[num]  │  描述符表
             └──────────────────┘
  avail_addr ┌──────────────────┐
             │  flags + idx     │  Available 环
             │  ring[num]       │
             │  used_event      │  EVENT_IDX 扩展
             └──────────────────┘
  used_addr  ┌──────────────────┐
             │  flags + idx     │  Used 环
             │  ring[num]       │
             │  avail_event     │  EVENT_IDX 扩展
             └──────────────────┘
```

---

## 5. VirtQueueElement：请求元素

```c
// virtio.h:66-79
typedef struct VirtQueueElement {
    unsigned int index;              // 描述符链头索引
    unsigned int len;                // 总长度
    unsigned int ndescs;             // 描述符数
    unsigned int out_num;            // 输出 scatter（Guest→Host）
    unsigned int in_num;             // 输入 gather（Host→Guest）
    bool in_order_filled;            // IN_ORDER 特性：已处理标记

    hwaddr *in_addr;                 // 输入缓冲区 GPA 数组
    hwaddr *out_addr;                // 输出缓冲区 GPA 数组
    struct iovec *in_sg;             // 输入 scatter-gather（HVA）
    struct iovec *out_sg;            // 输出 scatter-gather（HVA）
} VirtQueueElement;
```

---

## 6. Virtqueue 操作循环

### 6.1 核心 API

```c
// virtio.c — 操作循环

// ① 从 Avail 环取出请求
// virtio.c:2030-2040
void *virtqueue_pop(VirtQueue *vq, size_t sz)
// 读取 avail 环 → 遍历描述符链 → 映射到 HVA → 返回 VirtQueueElement

// ② 填充 Used 环
// virtio.c:1067-1098
void virtqueue_fill(VirtQueue *vq, const VirtQueueElement *elem,
                    unsigned int len, unsigned int idx)
// 写入 used 元素到 vq->used_elems[idx]

// ③ 刷新 Used 环（使 Guest 可见）
// virtio.c:1206-1218
void virtqueue_flush(VirtQueue *vq, unsigned int count)
// 更新 used->idx，内存屏障

// ④ 一步完成填充+刷新+通知
// virtio.c:1222-1227
void virtqueue_push(VirtQueue *vq, const VirtQueueElement *elem,
                    unsigned int len)
{
    virtqueue_fill(vq, elem, len, 0);
    virtqueue_flush(vq, 1);
}
```

### 6.2 完整处理周期

```
Guest 写入 Avail 环 → kick（写 notify 寄存器 / ioeventfd）
  │
  ▼
Host: handle_output() 回调被调用
  │
  ├── virtqueue_pop() — 取出 VirtQueueElement
  │     ├── 读取 avail->ring[last_avail_idx]
  │     ├── 遍历描述符链（desc[i].flags & NEXT）
  │     └── address_space_map() 映射 GPA→HVA
  │
  ├── 设备处理（读写磁盘 / 收发网络包）
  │
  ├── virtqueue_push() — 完成请求
  │     ├── virtqueue_fill() 写入 used_elems
  │     └── virtqueue_flush() 更新 used->idx
  │
  └── virtio_notify() — 通知 Guest
        ├── virtio_should_notify() — 检查通知抑制
        └── virtio_irq() — 发送中断
```

---

## 7. 通知机制

### 7.1 Guest→Host（Kick）

```
Guest 驱动写入 notify 寄存器
  │
  ├─ [无 ioeventfd] → MMIO 退出到 QEMU → virtio_queue_notify()
  │     → vq->handle_output(vdev, vq) — 直接调用处理回调
  │
  └─ [有 ioeventfd] → KVM eventfd 触发 → host_notifier 事件
        → AioContext 分发 → handle_output() — 在 IOThread 中处理
```

```c
// virtio.c:2515-2528
void virtio_queue_notify(VirtIODevice *vdev, int n)
{
    VirtQueue *vq = &vdev->vq[n];
    if (vq->host_notifier_enabled) {
        event_notifier_set(&vq->host_notifier);  // ioeventfd 路径
    } else if (vq->handle_output) {
        vq->handle_output(vdev, vq);              // 直接调用
    }
}
```

### 7.2 Host→Guest（Notify）

```c
// virtio.c:2730-2740
void virtio_notify(VirtIODevice *vdev, VirtQueue *vq)
{
    if (!virtio_should_notify(vdev, vq)) {
        return;                                    // 通知抑制
    }
    virtio_irq(vq);
}

// virtio.c:2698-2728
static void virtio_irq(VirtQueue *vq)
{
    virtio_set_isr(vq->vdev, 0x1);               // 设置 ISR 位

    if (qemu_in_iothread()) {
        // IOThread 中：defer_call 批量延迟 → irqfd
        defer_call(virtio_notify_irqfd_deferred_fn, &vq->guest_notifier);
    } else {
        // 主线程：直接触发 MSI-X / INTx
        virtio_notify_vector(vq->vdev, vq->vector);
    }
}
```

### 7.3 通知抑制（EVENT_IDX）

当 `VIRTIO_RING_F_EVENT_IDX` 协商后：
- **Guest→Host**：Host 在 avail_event 写入期望的 avail_idx，Guest 只在超过时才 kick
- **Host→Guest**：Guest 在 used_event 写入期望的 used_idx，Host 只在超过时才通知

这大幅减少 VM Exit 和中断次数。

---

## 8. Packed Virtqueue

Packed 格式（`VIRTIO_F_RING_PACKED`）用单一描述符环替代 Split 的三个环：

```c
// virtio.c:73-78
typedef struct VRingPackedDesc {
    uint64_t addr;                // Guest 物理地址
    uint32_t len;                 // 长度
    uint16_t id;                  // 缓冲区 ID
    uint16_t flags;               // AVAIL/USED/WRITE/NEXT/INDIRECT
} VRingPackedDesc;

// virtio.c:118-121
typedef struct VRingPackedDescEvent {
    uint16_t off_wrap;            // 偏移 + 环绕计数器
    uint16_t flags;               // ENABLE/DISABLE/DESC
} VRingPackedDescEvent;
```

**关键差异**：
- AVAIL/USED 标志嵌入描述符中，通过 wrap counter 区分
- 无需独立的 avail/used 环，cache 更友好
- 通知抑制通过 `VRingPackedDescEvent` 实现

**实现**：
- `virtqueue_packed_pop()`：`virtio.c:1880-2037`
- `virtqueue_packed_fill()`：`virtio.c:970-1081`
- `virtqueue_packed_flush()`：`virtio.c:1107-1216`

---

## 9. VirtIO-PCI 传输层

### 9.1 VirtIOPCIProxy

```c
// virtio-pci.h:115-154
struct VirtIOPCIProxy {
    PCIDevice pci_dev;                    // PCI 设备基类

    MemoryRegion bar;                     // BAR 内存区域

    /* Modern (virtio 1.0+) capability regions */
    MemoryRegion common;                  // Common 配置
    MemoryRegion isr;                     // ISR
    MemoryRegion device;                  // 设备特定配置
    MemoryRegion notify;                  // 通知区域（MMIO kick）
    MemoryRegion notify_pio;              // PIO kick

    uint32_t modern_bar;                  // Modern BAR 编号
    uint32_t io_bar;                      // Legacy I/O BAR

    uint64_t guest_features;              // Guest 协商特性
    VirtIOPCIQueue *vqs;                  // 每队列 PCI 状态
    VirtIOIRQFD *vector_irqfd;           // 每向量 irqfd
    int nvqs_with_notifiers;              // 有 notifier 的队列数
    VirtioBusState bus;                   // VirtIO 总线
};
```

### 9.2 关键流程

```c
// virtio-pci.c:2219-2347 — PCI 设备初始化
virtio_pci_realize()
{
    // 1. 注册 modern capabilities (common/isr/notify/device)
    // 2. 设置 BAR 内存区域
    // 3. 初始化 MSI-X
    // 4. 调用 virtio_pci_bus_ops.device_plugged()
}

// virtio-pci.c:566-640 — Common 配置 MMIO 读写
virtio_pci_common_read()   // Guest 读配置
virtio_pci_common_write()  // Guest 写配置/状态/队列选择

// virtio-pci.c:371-425 — ioeventfd 设置
virtio_pci_ioeventfd_assign()
// 为每个队列设置 ioeventfd → KVM eventfd → Guest kick 直达 Host
```

---

## 10. VirtioBus：总线抽象

```c
// virtio-bus.h:39-97
struct VirtioBusClass {
    void (*notify)(DeviceState *, uint16_t vector);     // 中断通知
    void (*save_config)(DeviceState *, QEMUFile *);     // 保存配置
    void (*load_config)(DeviceState *, QEMUFile *);     // 加载配置

    void (*set_guest_notifiers)(DeviceState *, int, bool); // 设置 irqfd
    void (*pre_plugged)(DeviceState *, Error **);       // 插入前回调
    void (*device_plugged)(DeviceState *, Error **);    // 插入后回调
    void (*device_unplugged)(DeviceState *);            // 移除回调

    bool (*ioeventfd_enabled)(DeviceState *);           // ioeventfd 支持
    int (*ioeventfd_assign)(DeviceState *, EventNotifier *, int, bool);
    bool (*queue_enabled)(DeviceState *, int);          // 队列是否启用
    AddressSpace *(*get_dma_as)(DeviceState *);         // DMA 地址空间
    bool (*iommu_enabled)(DeviceState *);               // IOMMU 启用
};

// virtio-bus.h:99-116
struct VirtioBusState {
    BusState parent_obj;
    bool ioeventfd_started;              // ioeventfd 已启动
    bool ioeventfd_grabbed;              // ioeventfd 被外部抢占（dataplane）
};
```

---

## 11. virtio-blk：块设备实现

### 11.1 VirtIOBlock 结构

```c
// virtio-blk.h:54-77
struct VirtIOBlock {
    VirtIODevice parent_obj;
    BlockBackend *blk;                   // 块设备后端
    VirtIOBlockConf conf;                // 配置参数
    unsigned short sector_mask;          // 扇区掩码
    struct VirtIOBlockDataPlane *dataplane;
    uint64_t host_features;
    // ...
};
```

### 11.2 请求结构

```c
// virtio-blk.h:79-93
typedef struct VirtIOBlockReq {
    int64_t sector_num;                  // 起始扇区
    VirtIOBlock *dev;
    VirtQueue *vq;
    VirtIOBlockConf *conf;
    struct virtio_blk_inhdr *in;         // 状态返回
    struct virtio_blk_outhdr outhdr;     // 请求头
    QEMUIOVector qiov;                   // I/O 向量
    size_t in_len;
    VirtQueueElement elem;               // 队列元素（内嵌）
} VirtIOBlockReq;
```

### 11.3 关键流程

```c
// virtio-blk.c:1723-1845 — 设备初始化
virtio_blk_device_realize()
{
    // 1. virtio_init() 初始化 VirtIODevice
    // 2. s->blk = conf->conf.blk — 关联 BlockBackend
    // 3. virtio_add_queue() 添加请求队列
    //    handle_output = virtio_blk_handle_output
}

// virtio-blk.c:1011-1042 — 处理队列请求
virtio_blk_handle_vq(VirtIOBlock *s, VirtQueue *vq)
{
    while ((req = virtio_blk_get_request(s, vq))) {
        // 解析请求类型（READ/WRITE/FLUSH/DISCARD/...）
        virtio_blk_handle_request(req, &mrb);
    }
    // 批量提交
    if (mrb.num_reqs) {
        virtio_blk_submit_multireq(s, &mrb);
    }
}

// virtio-blk.c:286-335 — 批量 I/O 提交
virtio_blk_submit_multireq(VirtIOBlock *s, MultiReqBuffer *mrb)
{
    // 合并相邻请求
    // blk_aio_preadv() / blk_aio_pwritev() → BlockBackend
}
```

---

## 12. virtio-net：网络设备实现

### 12.1 VirtIONet 结构

```c
// virtio-net.h:158-233（关键字段摘录）
struct VirtIONet {
    VirtIODevice parent_obj;
    NICState *nic;                       // NIC 状态
    NICConf nic_conf;                    // NIC 配置
    VirtIONetQueue *vqs;                 // 队列数组
    int max_queue_pairs;                 // 最大队列对数
    int curr_queue_pairs;                // 当前队列对数
    // ...
};
```

### 12.2 多队列结构

每个队列对包含一个 RX 队列和一个 TX 队列：

```c
// virtio-net.c:3939-3963
// realize 时分配：
n->vqs = g_new0(VirtIONetQueue, n->max_queue_pairs);
// 每对创建 rx + tx VirtQueue
```

### 12.3 关键处理回调

```c
// 接收：virtio-net.c:1620-1626
virtio_net_handle_rx(VirtIODevice *vdev, VirtQueue *vq)
// Guest 准备好接收缓冲区时触发

// 发送（timer 模式）：virtio-net.c:2822-2849
virtio_net_handle_tx_timer(VirtIODevice *vdev, VirtQueue *vq)
// 设置定时器延迟批量发送

// 发送（BH 模式）：virtio-net.c:2851-2875
virtio_net_handle_tx_bh(VirtIODevice *vdev, VirtQueue *vq)
// 使用 Bottom Half 立即调度发送
```

---

## 13. IOThread Dataplane 集成

### 13.1 工作原理

IOThread dataplane 让 virtio 设备在独立线程的 AioContext 中处理 I/O，避免 BQL 竞争：

```
                  主线程（BQL）              IOThread（无 BQL）
                  ┌─────────────┐          ┌──────────────────┐
Guest kick ──→ KVM eventfd ──→ ioeventfd ──→ AioContext
                                            │
                                            ├─ handle_output()
                                            │    └─ virtqueue_pop()
                                            │    └─ blk_aio_preadv()
                                            │
                                            ├─ I/O 完成回调
                                            │    └─ virtqueue_push()
                                            │    └─ virtio_notify()
                                            │         └─ defer_call → irqfd
                                            └──────────────────┘
```

### 13.2 关键代码

```c
// virtio-blk.c:1459-1515 — AIO 上下文初始化
virtio_blk_vq_aio_context_init(VirtIOBlock *s)
// 为每个 VirtQueue 设置 AioContext（主线程或 IOThread）

// virtio-blk.c:1593-1612 — BlockBackend AIO 上下文切换
blk_set_aio_context(s->blk, ctx, ...)
// 将 BlockBackend 绑定到 IOThread 的 AioContext

// virtio.c:4135-4192 — ioeventfd 启动
virtio_device_start_ioeventfd_impl()
// 为每个队列注册 ioeventfd → Guest kick 直达 IOThread
```

### 13.3 IOThread 中的通知优化

```c
// virtio.c:2698-2728 — virtio_irq()
if (qemu_in_iothread()) {
    // 在 IOThread 中使用 defer_call 批量延迟
    // 多个队列的通知合并为单次 irqfd write
    defer_call(virtio_notify_irqfd_deferred_fn, &vq->guest_notifier);
}
```

---

## 14. Feature 协商

### 14.1 通用 Feature 位

```c
// virtio_config.h:60-104
VIRTIO_F_ANY_LAYOUT         = 27    // 灵活布局
VIRTIO_F_VERSION_1          = 32    // VirtIO 1.0
VIRTIO_F_RING_PACKED        = 34    // Packed virtqueue
VIRTIO_F_IN_ORDER           = 35    // 按序完成
VIRTIO_F_NOTIFICATION_DATA  = 38    // 通知数据

// virtio_ring.h:79-85
VIRTIO_RING_F_INDIRECT_DESC = 28    // 间接描述符
VIRTIO_RING_F_EVENT_IDX     = 29    // EVENT_IDX 通知抑制
```

### 14.2 协商流程

```
1. 设备初始化
   host_features = 设备类默认 | 传输层添加 | 后端支持

2. Guest 驱动读取 host_features

3. Guest 驱动写入 guest_features（选择子集）
   → VirtIODeviceClass::set_features() 回调

4. Guest 写入 FEATURES_OK 到 status
   → VirtIODeviceClass::validate_features() 检查

5. 协商完成，双方按 guest_features 工作
```

---

## 15. 数据流全景图

### 15.1 virtio-blk 读取全路径

```
Guest 驱动
  │ 填充 VRingDesc（out: blk_header, in: data_buffer + status）
  │ 更新 avail->idx
  │ 写 notify 寄存器 → KVM ioeventfd
  ▼
IOThread AioContext
  │ host_notifier 触发
  │ virtio_blk_handle_output(vdev, vq)
  ▼
virtio_blk_handle_vq()
  │ virtqueue_pop() → VirtIOBlockReq
  │ 解析 outhdr → VIRTIO_BLK_T_IN（读取）
  │ virtio_blk_submit_multireq()
  ▼
BlockBackend
  │ blk_aio_preadv() → 协程 I/O
  │ → bdrv_co_preadv_part() → 驱动 → 实际磁盘读取
  ▼
I/O 完成回调
  │ virtio_blk_rw_complete()
  │ virtqueue_push(vq, &req->elem, req->in_len)
  │ virtio_notify(vdev, vq)
  │   └── defer_call → event_notifier_set(&vq->guest_notifier)
  │       └── KVM irqfd → MSI-X 中断
  ▼
Guest 驱动
  │ 读取 used->idx
  │ 获取完成的数据
```

### 15.2 架构层次

```
┌─────────────────────────────────────────────┐
│               Guest 驱动                      │
│  (virtio-blk-pci / virtio-net-pci)          │
├─────────────────────────────────────────────┤
│         VirtIO 设备层                         │
│  VirtIOBlock / VirtIONet                     │
│  ┌─────────────┐  ┌─────────────┐           │
│  │ VirtQueue 0  │  │ VirtQueue 1  │  ...     │
│  └──────┬───────┘  └──────┬───────┘          │
├─────────┼──────────────────┼────────────────┤
│         │  VirtioBus 抽象   │                 │
├─────────┼──────────────────┼────────────────┤
│         ▼                  ▼                  │
│    VirtIO-PCI 传输层                          │
│    MSI-X / ioeventfd / irqfd                 │
├─────────────────────────────────────────────┤
│    PCI 总线 / MemoryRegion                    │
├─────────────────────────────────────────────┤
│    KVM / TCG                                  │
└─────────────────────────────────────────────┘
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `include/hw/virtio/virtio.h` | ~450 | VirtQueueElement (66-79)、VirtIODevice (108-171)、VirtIODeviceClass (173-241)、Feature 宏 (390-404) |
| `hw/virtio/virtio.c` | ~4300 | VRingDesc/Avail/Used (65-98)、VRing (107-116)、VRingPackedDesc (73-78)、VirtQueue (123-159)、virtqueue_fill/flush/push (1067-1227)、virtqueue_pop (2030-2040)、virtio_add_queue (2558-2572)、virtio_queue_notify (2515-2528)、virtio_irq (2698-2728)、virtio_notify (2730-2740)、ioeventfd (4135-4235) |
| `include/hw/virtio/virtio-pci.h` | ~160 | VirtIOPCIProxy (115-154) |
| `hw/virtio/virtio-pci.c` | ~2400 | virtio_pci_common_read/write (566-640)、ioeventfd_assign (371-415)、virtio_pci_realize (2219-2347) |
| `include/hw/virtio/virtio-bus.h` | ~120 | VirtioBusClass (39-97)、VirtioBusState (99-116) |
| `include/hw/virtio/virtio-blk.h` | ~95 | VirtIOBlock (54-77)、VirtIOBlockReq (79-93) |
| `hw/block/virtio-blk.c` | ~1900 | submit_multireq (286-335)、handle_vq (1011-1042)、handle_output (1044-1059)、aio_context_init (1459-1515)、realize (1723-1845) |
| `include/hw/virtio/virtio-net.h` | ~240 | VirtIONet (158-233) |
| `hw/net/virtio-net.c` | ~4000 | handle_rx (1620-1626)、handle_tx_timer (2822-2849)、handle_tx_bh (2851-2875)、realize (3867-3975) |
| `include/standard-headers/linux/virtio_config.h` | ~110 | VIRTIO_F_* 特性位 (60-104) |
