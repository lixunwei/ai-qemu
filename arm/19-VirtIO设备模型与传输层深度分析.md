# VirtIO 设备模型与传输层深度分析

## 1. 概述

VirtIO 是 QEMU 中半虚拟化 I/O 的标准框架，提供统一的设备-驱动接口。本文分析 QEMU 11.0.50 中 VirtIO 的核心数据结构、Virtqueue 机制、MMIO/PCI 传输层、通知路径、feature 协商和 vhost 卸载。

**关键源文件：**
- `include/hw/virtio/virtio.h` — VirtIODevice 核心结构定义
- `hw/virtio/virtio.c` — VirtIO 核心逻辑（virtqueue、通知、生命周期）
- `hw/virtio/virtio-mmio.c` — MMIO 传输层
- `hw/virtio/virtio-pci.c` — PCI 传输层
- `hw/virtio/vhost.c` — vhost 后端卸载
- `hw/arm/virt.c` — ARM virt 机器的 VirtIO 集成

---

## 2. VirtIODevice 核心结构

### 2.1 结构定义

```c
// virtio.h:108-171
struct VirtIODevice {
    DeviceState parent_obj;
    const char *name;
    uint8_t status;          // 设备状态机（ACKNOWLEDGE/DRIVER/DRIVER_OK/FEATURES_OK）
    uint8_t isr;             // 中断状态寄存器
    uint16_t queue_sel;      // 当前选中的 virtqueue 索引

    VIRTIO_DECLARE_FEATURES(host_features);     // 设备提供的特性集
    VIRTIO_DECLARE_FEATURES(guest_features);    // 驱动选择的特性集
    VIRTIO_DECLARE_FEATURES(backend_features);  // 后端（vhost）支持的特性

    size_t config_len;
    void *config;            // 设备特定配置空间
    uint16_t config_vector;  // MSI-X 配置变更向量
    uint32_t generation;     // 配置空间代际号
    int nvectors;            // MSI-X 向量数
    VirtQueue *vq;           // virtqueue 数组（最大 VIRTIO_QUEUE_MAX）
    uint16_t device_id;      // 设备类型 ID（net=1, blk=2 等）
    bool vm_running;
    bool broken;             // 设备异常状态
    bool started;            // 设备已启动
    bool vhost_started;      // vhost 后端已激活
    AddressSpace *dma_as;    // DMA 地址空间
    EventNotifier config_notifier;
    bool device_iotlb_enabled;
};
```

### 2.2 设备类接口

`VirtioDeviceClass`（virtio.h:173+）定义了设备的虚方法：
- `realize` / `unrealize` — 设备实例化/销毁
- `get_config` / `set_config` — 配置空间读写
- `get_features` — 返回设备支持的特性位
- `set_status` — 状态变更回调
- `reset` — 设备重置

---

## 3. Virtqueue 与 Vring 机制

### 3.1 Split Vring 结构

```c
// virtio.c:65-98
typedef struct VRingDesc {     // 描述符
    uint64_t addr;             // 缓冲区物理地址
    uint32_t len;              // 缓冲区长度
    uint16_t flags;            // NEXT / WRITE / INDIRECT
    uint16_t next;             // 链式下一个描述符索引
} VRingDesc;

typedef struct VRingAvail {    // Available ring（驱动→设备）
    uint16_t flags;            // VRING_AVAIL_F_NO_INTERRUPT
    uint16_t idx;              // 下一个可用描述符索引
    uint16_t ring[];           // 描述符头索引数组
} VRingAvail;

typedef struct VRingUsed {     // Used ring（设备→驱动）
    uint16_t flags;            // VRING_USED_F_NO_NOTIFY
    uint16_t idx;              // 下一个已用描述符索引
    VRingUsedElem ring[];      // {id, len} 数组
} VRingUsed;
```

### 3.2 Packed Vring 结构

```c
// virtio.c:73-78
typedef struct VRingPackedDesc {
    uint64_t addr;
    uint32_t len;
    uint16_t id;               // 缓冲区 ID（替代 next 链式）
    uint16_t flags;            // AVAIL/USED 位，包裹计数器
} VRingPackedDesc;
```

**Packed vs Split 选择：** 通过 `VIRTIO_F_RING_PACKED` feature 位协商（virtio.c:621-625），packed 模式合并 avail/used 环为单一描述符环，减少缓存行弹跳。

### 3.3 VRingMemoryRegionCaches

```c
// virtio.c:100-105
typedef struct VRingMemoryRegionCaches {
    struct rcu_head rcu;
    MemoryRegionCache desc;    // 描述符表的缓存映射
    MemoryRegionCache avail;   // Available ring 缓存
    MemoryRegionCache used;    // Used ring 缓存
} VRingMemoryRegionCaches;
```

利用 `MemoryRegionCache` 将 guest 物理地址预映射为 host 虚拟地址，避免每次访问的地址翻译开销。

---

## 4. 设备生命周期

### 4.1 virtio_init()

```c
// virtio.c:3568-3609
void virtio_init(VirtIODevice *vdev, uint16_t device_id, size_t config_size)
{
    // 1. 查询传输层 nvectors（用于 MSI-X）
    int nvectors = k->query_nvectors ? k->query_nvectors(qbus->parent) : 0;

    // 2. 初始化设备状态
    vdev->device_id = device_id;
    vdev->status = 0;
    qatomic_set(&vdev->isr, 0);
    vdev->queue_sel = 0;
    vdev->config_vector = VIRTIO_NO_VECTOR;

    // 3. 分配 VIRTIO_QUEUE_MAX 个 VirtQueue
    vdev->vq = g_new0(VirtQueue, VIRTIO_QUEUE_MAX);
    for (i = 0; i < VIRTIO_QUEUE_MAX; i++) {
        vdev->vq[i].vector = VIRTIO_NO_VECTOR;
        vdev->vq[i].vdev = vdev;
        vdev->vq[i].queue_index = i;
    }

    // 4. 分配配置空间
    vdev->config_len = config_size;
    vdev->config = config_size ? g_malloc0(config_size) : NULL;

    // 5. 注册 VM 状态变更回调
    vdev->vmstate = qdev_add_vm_change_state_handler(..., virtio_vmstate_change, vdev);
}
```

### 4.2 virtio_add_queue()

```c
// virtio.c:2558-2596
```
设备在 `realize` 中调用 `virtio_add_queue(vdev, queue_size, handle_output)` 注册每个 virtqueue 及其处理回调。

### 4.3 Feature 协商

```c
// virtio.c:3200-3239 — virtio_set_features()
```

协商流程：
1. **设备侧**：`get_features()` 返回 `host_features`（设备能力 + 后端能力）
2. **驱动侧**：写入 feature 寄存器，设置 `guest_features`
3. **生效**：`virtio_set_features()` 验证并应用，设置 `FEATURES_OK` 状态位
4. **版本选择**：`VIRTIO_F_VERSION_1` 表示 modern（非 legacy）模式

---

## 5. MMIO 传输层

### 5.1 寄存器读写

```c
// virtio-mmio.c:85-245 — virtio_mmio_read()
```

MMIO 传输采用平坦寄存器布局，关键寄存器偏移：

| 偏移 | 寄存器 | 功能 |
|------|--------|------|
| 0x000 | MAGIC_VALUE | 魔数 `0x74726976`（"virt"） |
| 0x004 | VERSION | 版本号（1=legacy, 2=modern） |
| 0x008 | DEVICE_ID | 设备类型 ID |
| 0x00c | VENDOR_ID | 厂商 ID |
| 0x010 | HOST_FEATURES | 设备特性位读取 |
| 0x020 | GUEST_FEATURES | 驱动特性位写入 |
| 0x030 | QUEUE_SEL | 选择 virtqueue |
| 0x044 | QUEUE_READY | 标记队列就绪 |
| 0x050 | QUEUE_NOTIFY | 通知设备（写 queue 索引） |
| 0x060 | INTERRUPT_STATUS | 中断状态 |
| 0x064 | INTERRUPT_ACK | 中断确认 |
| 0x070 | STATUS | 设备状态字 |
| 0x100+ | CONFIG | 设备特定配置空间 |

### 5.2 配置空间访问

```c
// virtio-mmio.c:117-141
if (offset >= VIRTIO_MMIO_CONFIG) {
    offset -= VIRTIO_MMIO_CONFIG;
    if (proxy->legacy) {
        return virtio_config_readb/w/l(vdev, offset);  // legacy: 原生字节序
    } else {
        return virtio_config_modern_readb/w/l(vdev, offset);  // modern: 小端
    }
}
```

### 5.3 ioeventfd 加速

```c
// virtio-mmio.c:46-70
```
MMIO 传输支持 ioeventfd：将 QUEUE_NOTIFY 寄存器的写操作映射为 eventfd，使 Guest 通知直接到达 KVM/vhost 后端，绕过 QEMU 用户态。

---

## 6. PCI 传输层

### 6.1 VirtIOPCIProxy

```c
// virtio-pci.h:115-172
```

PCI 传输通过 `VirtIOPCIProxy` 包装 VirtIODevice，支持：
- **Legacy 模式**：单一 I/O BAR，兼容 virtio 0.9
- **Modern 模式**：多 Capability 结构（common/notify/isr/device/pci_cfg），通过 PCIe Memory BAR 访问

### 6.2 MSI-X 中断

```c
// virtio-pci.c:73-85
```

PCI 传输优先使用 MSI-X：
- 每个 virtqueue 分配独立的 MSI-X 向量
- 配置变更使用专用 config_vector
- 通过 `msix_notify()` 投递中断，避免 INTx 共享

### 6.3 Notification 路径

```c
// virtio-pci.c:371-425 — PCI ioeventfd
```

Modern PCI 传输中，QUEUE_NOTIFY 寄存器映射为 ioeventfd，Guest 写入该地址触发 KVM 的 eventfd 信号，直接通知 QEMU 或 vhost 后端。

---

## 7. 通知机制

### 7.1 Guest → Device（kick）

```c
// virtio.c:2515-2533
void virtio_queue_notify(VirtIODevice *vdev, int n)
{
    VirtQueue *vq = &vdev->vq[n];
    if (unlikely(!vq->vring.desc || vdev->broken))
        return;

    if (vq->host_notifier_enabled) {
        event_notifier_set(&vq->host_notifier);  // ioeventfd 路径
    } else if (vq->handle_output) {
        vq->handle_output(vdev, vq);              // 直接回调
    }
}
```

**两条路径：**
1. **ioeventfd 路径**：Guest MMIO/PIO 写 → KVM eventfd → vhost 或 QEMU 事件循环
2. **直接回调**：QEMU 用户态处理 `handle_output`（非 vhost 场景）

### 7.2 Device → Guest（中断）

```c
// virtio.c:2730-2756 — virtio_notify()
```

设备完成请求后：
1. 更新 Used ring（`virtqueue_push` / `virtqueue_flush`）
2. 调用 `virtio_notify()` → 设置 `isr` → 触发传输层中断
3. **MMIO**：设置 SysBus IRQ
4. **PCI**：`msix_notify()` 或 `pci_set_irq()`（INTx 回退）

### 7.3 irqfd 加速

对于 vhost 场景，中断注入通过 irqfd 直接从内核注入 Guest，绕过 QEMU 用户态：
```
vhost-net (内核) → irqfd → KVM → Guest 中断
```

---

## 8. VirtIO 设备实例

### 8.1 virtio-net

```c
// hw/net/virtio-net.c:3867-3950 — virtio_net_device_realize()
```

- 创建 RX/TX virtqueue 对（可多队列）
- 收包：`virtio_net_receive()` → `virtqueue_push()` → `virtio_notify()`
- 发包：TX virtqueue 的 `handle_output` → `virtqueue_pop()` → 发送到网络后端

### 8.2 virtio-blk

```c
// hw/block/virtio-blk.c:1723+ — virtio_blk_device_realize()
```

- 单 virtqueue（或 multi-queue）
- 请求处理：`virtqueue_pop()` → 解析 `virtio_blk_outhdr` → 提交到块后端 → 完成回调 `virtio_blk_req_complete()`

---

## 9. vhost 后端卸载

### 9.1 核心原理

vhost 将 virtqueue 的数据面处理卸载到内核（vhost-net）或用户态进程（vhost-user）：

```
Guest → ioeventfd → vhost 后端 → 直接处理 → irqfd → Guest
                    (绕过 QEMU 用户态)
```

### 9.2 vhost 初始化

```c
// hw/virtio/vhost.c:1257-1450 — vhost_dev_init()
// hw/virtio/vhost.c:1552-1804 — vhost_dev_start()
```

启动流程：
1. `vhost_dev_init()` — 打开 vhost 设备，设置内存映射
2. `vhost_dev_start()` — 传递 virtqueue 地址、启用 kick/irqfd
3. 数据面完全在 vhost 后端运行

### 9.3 vhost-user

```c
// hw/virtio/vhost-user.c:237-380
```

vhost-user 通过 Unix socket 将 virtqueue 数据面卸载到独立进程（如 DPDK、SPDK），支持：
- 共享内存映射（mmap guest memory）
- 跨进程 eventfd 传递
- 热迁移协调

---

## 10. ARM virt 机器的 VirtIO 集成

### 10.1 MMIO 传输创建

```c
// virt.c:1504-1543 — create_virtio_devices()
for (i = 0; i < vms->virtio_transports; i++) {
    int irq = vms->irqmap[VIRT_MMIO] + i;
    hwaddr base = vms->memmap[VIRT_MMIO].base + i * size;
    sysbus_create_simple("virtio-mmio", base,
                         qdev_get_gpio_in(vms->gic, irq));
}
```

**关键设计：**
- 默认创建多个 MMIO 传输实例（通常 32 个）
- 每个实例分配独立的 MMIO 基地址和 GIC SPI 中断号
- 地址递增排列，设备树按递减顺序添加节点

### 10.2 MMIO 地址映射

每个 virtio-mmio 实例占用固定大小的 MMIO 窗口，由 `vms->memmap[VIRT_MMIO]` 定义基地址和大小。IRQ 从 `vms->irqmap[VIRT_MMIO]` 连续分配。

### 10.3 PCI 传输

ARM virt 同时支持 PCI 传输：VirtIO PCI 设备挂在 GPEX Host Bridge 下，利用 MSI-X 和 PCIe 能力结构。

---

## 11. VirtIO 数据面完整路径

### 11.1 Guest → Device（以 virtio-blk 为例）

```
1. Guest 驱动填充描述符 → 更新 Available ring
2. Guest 写 QUEUE_NOTIFY 寄存器
3. [MMIO] → KVM exit / ioeventfd → virtio_queue_notify()
4. handle_output → virtqueue_pop() 取出描述符链
5. 解析请求头 → 提交到块层
6. 块层完成 → virtqueue_push() 写 Used ring
7. virtio_notify() → 设置 ISR → 触发 GIC SPI 中断
8. Guest 收到中断 → 处理 Used ring
```

### 11.2 vhost 加速路径

```
1. Guest 驱动填充描述符 → 更新 Available ring
2. Guest 写 QUEUE_NOTIFY → ioeventfd → vhost-net（内核）
3. vhost-net 直接读取 Available ring → 处理数据
4. vhost-net 更新 Used ring → irqfd → KVM → Guest 中断
   （全程不经过 QEMU 用户态）
```

---

## 12. 小结

| 方面 | 实现要点 |
|------|----------|
| **核心结构** | VirtIODevice（status/features/vq[]）+ VirtQueue（vring desc/avail/used） |
| **Vring 模式** | Split（链式描述符）vs Packed（单环 + wrap counter），VIRTIO_F_RING_PACKED 协商 |
| **MMIO 传输** | 平坦寄存器布局（0x000-0x0ff + config），ioeventfd 加速通知 |
| **PCI 传输** | Modern capability 结构 + MSI-X，Legacy I/O BAR 兼容 |
| **通知路径** | Guest→Device：ioeventfd/直接回调；Device→Guest：ISR+IRQ/MSI-X/irqfd |
| **vhost** | 数据面卸载到内核（vhost-net）或用户进程（vhost-user），ioeventfd+irqfd 旁路 |
| **ARM 集成** | virt 机器创建 32 个 virtio-mmio 实例，连接 GIC SPI；支持 PCI 传输 |
