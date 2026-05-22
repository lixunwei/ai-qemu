# VirtIO 设备深度分析 — Virtqueue 实现、vhost-user 与 IOMMU 集成

## 1. 概述

VirtIO 是 QEMU 虚拟化 I/O 的核心框架，定义了 Guest 与 Hypervisor 之间高效的设备通信协议。本文从三个维度深入分析：**Virtqueue 核心机制**（VRing 描述符、avail/used 环、请求处理流程）、**vhost-user 用户态后端**（协议消息、内存共享、virtqueue 卸载）、**IOMMU 集成**（virtio-iommu 设备、DMA 地址翻译）。

**关键源文件：**
- `include/hw/virtio/virtio.h` — VirtIODevice/VirtioDeviceClass
- `hw/virtio/virtio.c` — VirtQueue/VRing 核心实现（~3000行）
- `hw/virtio/virtio-pci.c` — PCI 传输层
- `hw/virtio/vhost.c` — vhost 通用框架
- `hw/virtio/vhost-user.c` — vhost-user 协议实现
- `hw/virtio/virtio-iommu.c` — VirtIO IOMMU 设备
- `hw/net/virtio-net.c` — VirtIO-Net 示例设备

---

## 2. VirtIODevice — 设备基类

```c
// include/hw/virtio/virtio.h:108-171
struct VirtIODevice {
    DeviceState parent_obj;       // QOM 父对象
    const char *name;             // 设备名称
    uint8_t status;               // 设备状态 (ACKNOWLEDGE/DRIVER/DRIVER_OK/FEATURES_OK)
    uint8_t isr;                  // 中断状态寄存器
    uint16_t queue_sel;           // 当前选中的队列
    
    // 特性协商（三层）:
    host_features;                // QEMU 设备可提供的全部特性
    guest_features;               // Guest 驱动选择的特性子集
    backend_features;             // 后端（如 vhost）支持的特性
    
    size_t config_len;            // 设备配置空间大小
    void *config;                 // 设备配置数据
    uint16_t config_vector;       // 配置变更 MSI-X 向量
    uint32_t generation;          // 配置代次（Guest 读一致性）
    int nvectors;                 // MSI-X 向量数
    VirtQueue *vq;                // Virtqueue 数组
    uint16_t device_id;           // VirtIO 设备 ID
    bool started;                 // 设备是否已启动
    bool vhost_started;           // vhost 后端是否已启动
    AddressSpace *dma_as;         // DMA 地址空间（IOMMU 感知）
    bool device_iotlb_enabled;    // 设备 IOTLB 是否启用
};
```

### 2.1 VirtioDeviceClass — 设备类回调

```c
// include/hw/virtio/virtio.h:173-241
struct VirtioDeviceClass {
    DeviceClass parent;
    void (*realize)(DeviceState *dev, Error **errp);  // 设备实例化
    void (*unrealize)(DeviceState *dev);              // 设备销毁
    uint64_t (*get_features)(VirtIODevice *vdev, ...); // 特性查询
    void (*set_features)(VirtIODevice *vdev, uint64_t); // 特性确认
    void (*get_config)(VirtIODevice *vdev, uint8_t *); // 读配置
    void (*set_config)(VirtIODevice *vdev, const uint8_t *); // 写配置
    void (*reset)(VirtIODevice *vdev);                // 设备重置
    void (*set_status)(VirtIODevice *vdev, uint8_t);  // 状态变更
    // guest_notifier_mask/pending: 中断通知控制
};
```

---

## 3. Virtqueue 核心

### 3.1 VRing — 虚环结构

```c
// hw/virtio/virtio.c:65-116

// 描述符: Guest 提交的 I/O 缓冲区
typedef struct VRingDesc {
    uint64_t addr;     // Guest 物理地址
    uint32_t len;      // 缓冲区长度
    uint16_t flags;    // VRING_DESC_F_NEXT | F_WRITE | F_INDIRECT
    uint16_t next;     // 链式描述符下一个索引
} VRingDesc;

// Available 环: Guest → Device 方向（Guest 填充）
typedef struct VRingAvail {
    uint16_t flags;    // VRING_AVAIL_F_NO_INTERRUPT
    uint16_t idx;      // 下一个可用描述符索引（单调递增）
    uint16_t ring[];   // 描述符索引数组
} VRingAvail;

// Used 环: Device → Guest 方向（Device 填充）
typedef struct VRingUsed {
    uint16_t flags;    // VRING_USED_F_NO_NOTIFY
    uint16_t idx;      // 下一个已用条目索引
    VRingUsedElem ring[];  // {id, len} 数组
} VRingUsed;

// 完整 VRing:
typedef struct VRing {
    unsigned int num;          // 描述符数量（2 的幂）
    unsigned int num_default;  // 默认大小
    unsigned int align;        // 对齐
    hwaddr desc;               // 描述符表 Guest 物理地址
    hwaddr avail;              // Available 环地址
    hwaddr used;               // Used 环地址
    VRingMemoryRegionCaches *caches; // DMA 缓存（加速访问）
} VRing;
```

### 3.2 Packed Virtqueue（VirtIO 1.1）

```c
// hw/virtio/virtio.c:73-78, 118-121
typedef struct VRingPackedDesc {
    uint64_t addr;     // 缓冲区地址
    uint32_t len;      // 缓冲区长度
    uint16_t id;       // 缓冲区 ID
    uint16_t flags;    // AVAIL/USED 位 + WRITE/NEXT/INDIRECT
} VRingPackedDesc;

// Packed 模式: 描述符同时充当 avail 和 used 环
// 使用 AVAIL/USED 标志位 + wrap counter 替代独立的 idx
// 优势: 更好的缓存局部性, 减少内存访问
```

### 3.3 VirtQueue 结构

```c
// hw/virtio/virtio.c:123-159
struct VirtQueue {
    VRing vring;                      // 底层 VRing
    VirtQueueElement *used_elems;     // 已使用元素缓存
    
    uint16_t last_avail_idx;          // 上次消费的 avail 索引
    bool last_avail_wrap_counter;     // Packed 模式 wrap 计数器
    uint16_t shadow_avail_idx;        // 缓存的 avail.idx（减少 DMA 读）
    uint16_t used_idx;                // 当前 used 索引
    uint16_t signalled_used;          // 上次通知的 used 索引
    bool notification;                // 通知是否启用
    
    uint16_t queue_index;             // 队列编号
    unsigned int inuse;               // 正在处理的描述符数
    uint16_t vector;                  // MSI-X 向量号
    VirtIOHandleOutput handle_output; // 队列通知回调
    VirtIODevice *vdev;               // 所属设备
    EventNotifier guest_notifier;     // Guest 通知（irqfd）
    EventNotifier host_notifier;      // Host 通知（ioeventfd）
};
```

---

## 4. Virtqueue 操作流程

### 4.1 请求处理流程

```
Guest 驱动:
  1. 在 desc 表中填充 I/O 缓冲区描述符
  2. 将描述符头索引写入 avail.ring[avail.idx % num]
  3. 递增 avail.idx
  4. 写 notify 寄存器 (MMIO/PIO) → 触发 VM Exit

QEMU 设备:
  5. virtio_queue_notify() → handle_output 回调
  6. virtqueue_pop() → 消费 avail 环中的描述符
  7. 处理 I/O 请求
  8. virtqueue_push() → 写入 used 环
  9. virtio_notify() → 向 Guest 注入中断
```

### 4.2 virtio_queue_notify() — Guest 踢设备

```c
// hw/virtio/virtio.c:2515-2533
void virtio_queue_notify(VirtIODevice *vdev, int n)
{
    VirtQueue *vq = &vdev->vq[n];
    
    if (vq->host_notifier_enabled) {
        // vhost/ioeventfd 模式: 通过 eventfd 通知
        event_notifier_set(&vq->host_notifier);
    } else if (vq->handle_output) {
        // 普通模式: 直接调用设备回调
        vq->handle_output(vdev, vq);
    }
}
```

### 4.3 virtqueue_pop() — 消费请求

```c
// hw/virtio/virtio.c:2030-2041
void *virtqueue_pop(VirtQueue *vq, size_t sz)
{
    if (virtio_vdev_has_feature(vq->vdev, VIRTIO_F_RING_PACKED)) {
        return virtqueue_packed_pop(vq, sz);  // Packed 模式
    } else {
        return virtqueue_split_pop(vq, sz);   // Split 模式
    }
    // 返回 VirtQueueElement:
    //   index: 描述符链头索引
    //   in_num/out_num: 输入/输出缓冲区数
    //   in_addr[]/out_addr[]: Guest 地址数组
    //   in_sg[]/out_sg: scatter-gather 列表（已映射到宿主）
}
```

### 4.4 virtqueue_push() + virtio_notify() — 完成通知

```c
// hw/virtio/virtio.c:1222-1228
void virtqueue_push(VirtQueue *vq, const VirtQueueElement *elem, unsigned int len)
{
    virtqueue_fill(vq, elem, len, 0);  // 写入 used.ring
    virtqueue_flush(vq, 1);            // 更新 used.idx
}

// hw/virtio/virtio.c:2730-2740
void virtio_notify(VirtIODevice *vdev, VirtQueue *vq)
{
    if (!virtio_should_notify(vdev, vq))  // 检查 Guest 是否禁止通知
        return;
    virtio_irq(vq);  // 注入中断 (MSI-X 或 legacy)
}
```

---

## 5. VirtIO PCI 传输层

### 5.1 VirtIOPCIProxy

```c
// include/hw/virtio/virtio-pci.h:115-154
struct VirtIOPCIProxy {
    PCIDevice pci_dev;        // PCI 设备基类
    MemoryRegion bar;         // Legacy BAR
    union {
        struct {
            VirtIOPCIRegion common;      // Common 配置
            VirtIOPCIRegion isr;         // ISR 寄存器
            VirtIOPCIRegion device;      // 设备特定配置
            VirtIOPCIRegion notify;      // 通知区域 (MMIO)
            VirtIOPCIRegion notify_pio;  // 通知区域 (PIO)
        };
        VirtIOPCIRegion regs[5];
    };
    MemoryRegion modern_bar;  // Modern BAR (VirtIO 1.0+)
    uint32_t nvectors;        // MSI-X 向量数
    VirtIOPCIQueue vqs[VIRTIO_QUEUE_MAX];
    VirtioBusState bus;       // VirtIO 总线
};
```

### 5.2 PCI 中断路径

```c
// hw/virtio/virtio-pci.c:73-85
static void virtio_pci_notify(DeviceState *d, uint16_t vector)
{
    VirtIOPCIProxy *proxy = to_virtio_pci_proxy_fast(d);
    
    if (msix_enabled(&proxy->pci_dev)) {
        // Modern: MSI-X 中断 → 直接投递到目标 vCPU
        msix_notify(&proxy->pci_dev, vector);
    } else {
        // Legacy: 电平触发 INTx → 设置 ISR 位
        pci_set_irq(&proxy->pci_dev, qatomic_read(&vdev->isr) & 1);
    }
}

// PCI Capability 结构映射:
//   BAR 内偏移 → Common/ISR/Device/Notify 配置空间
//   Guest 写 Notify BAR + queue_offset → virtio_queue_notify()
//   ioeventfd 优化: 写 Notify → 直接 eventfd → 跳过 VM Exit
```

---

## 6. VirtIO-Net 示例

### 6.1 接收路径（RX）

```c
// hw/net/virtio-net.c
// virtio_net_receive() (2672-2681): 入口
//   → virtio_net_handle_rx() (1620-1902): 核心处理
//     1. virtqueue_pop() — 获取 Guest 提供的 RX 缓冲区
//     2. 填充 virtio_net_hdr (校验和、GSO 信息)
//     3. iov_from_buf() — 拷贝数据包到 Guest 缓冲区
//     4. virtqueue_push() — 写入 used 环
//     5. virtio_notify() — 通知 Guest 有新数据
//
// MRG_RXBUF 特性: 大数据包可跨多个 RX 缓冲区
//   num_buffers 字段指示使用了几个缓冲区
```

### 6.2 发送路径（TX）

```c
// hw/net/virtio-net.c
// virtio_net_handle_tx_bh() (2851-2870): BH 模式入口
//   → virtio_net_flush_tx() (2718-2849): 核心处理
//     1. virtqueue_pop() — 获取 Guest 填充的 TX 描述符
//     2. 解析 virtio_net_hdr
//     3. qemu_sendv_packet_async() — 发送到网络后端
//     4. virtqueue_push() — 标记完成
//     5. virtio_notify() — 通知 Guest TX 完成
//
// TX 模式:
//   BH: 使用 bottom-half, 延迟批量发送
//   Timer: 使用定时器合并通知
```

---

## 7. vhost-user 用户态后端

### 7.1 vhost_dev 结构

```c
// include/hw/virtio/vhost.h:79-137
struct vhost_dev {
    VirtIODevice *vdev;              // 关联的 VirtIO 设备
    MemoryListener memory_listener;  // 内存变更监听
    MemoryListener iommu_listener;   // IOMMU 变更监听
    struct vhost_memory *mem;        // 共享内存表
    struct vhost_virtqueue *vqs;     // Virtqueue 描述
    unsigned int nvqs;               // 队列数
    int vq_index;                    // 起始队列索引
    
    // 特性协商:
    features;                        // 后端可用特性
    acked_features;                  // 协商后特性
    uint64_t protocol_features;      // vhost-user 协议特性
    
    uint64_t max_queues;             // 最大队列数
    bool started;                    // 是否已启动
    const VhostOps *vhost_ops;       // 操作函数表 (kernel_ops/user_ops)
};
```

### 7.2 vhost-user 协议消息

```c
// hw/virtio/vhost-user.c:57-103
typedef enum VhostUserRequest {
    VHOST_USER_GET_FEATURES = 1,       // 查询后端特性
    VHOST_USER_SET_FEATURES = 2,       // 设置特性
    VHOST_USER_SET_OWNER = 3,          // 设置所有者
    VHOST_USER_SET_MEM_TABLE = 5,      // 共享内存表 (带 fd 传递)
    VHOST_USER_SET_VRING_NUM = 8,      // 设置队列大小
    VHOST_USER_SET_VRING_ADDR = 9,     // 设置 VRing 地址
    VHOST_USER_SET_VRING_BASE = 10,    // 设置 avail 基址
    VHOST_USER_SET_VRING_KICK = 12,    // 设置 kick eventfd
    VHOST_USER_SET_VRING_CALL = 13,    // 设置 call eventfd
    VHOST_USER_GET_PROTOCOL_FEATURES = 15,
    VHOST_USER_SET_PROTOCOL_FEATURES = 16,
    VHOST_USER_GET_CONFIG = 24,        // 获取设备配置
    VHOST_USER_SET_CONFIG = 25,        // 设置设备配置
    VHOST_USER_ADD_MEM_REG = 37,       // 动态添加内存区域
    VHOST_USER_REM_MEM_REG = 38,       // 动态移除内存区域
    VHOST_USER_SET_STATUS = 39,        // 设置设备状态
    VHOST_USER_GET_STATUS = 40,        // 获取设备状态
    // ... 总计 43 种消息类型
} VhostUserRequest;

// 消息通过 Unix Domain Socket 传输
// 文件描述符通过 SCM_RIGHTS 附带传递 (内存 fd, eventfd)
```

### 7.3 vhost 生命周期

```c
// vhost_dev_start() — hw/virtio/vhost.c:2109-2279
// 启动 vhost 后端:
//   1. vhost_dev_set_features() — 协商特性
//   2. vhost_set_mem_table() — 共享 Guest 内存映射
//   3. 对每个 VQ:
//      vhost_virtqueue_start() — hw/virtio/vhost.c:1257-1398
//        a. SET_VRING_NUM — 设置队列大小
//        b. SET_VRING_ADDR — 设置 desc/avail/used 地址
//        c. SET_VRING_BASE — 设置 avail 起始索引
//        d. SET_VRING_KICK — 传递 kick eventfd
//        e. SET_VRING_CALL — 传递 call eventfd
//   4. 此后 Virtqueue 操作由后端进程直接处理
//      Guest kick → eventfd → 后端进程 (绕过 QEMU)
//      后端完成 → eventfd → irqfd → Guest 中断 (绕过 QEMU)

// vhost_dev_stop() — hw/virtio/vhost.c:2283-2310
// 停止 vhost: 恢复 QEMU 对 Virtqueue 的控制
```

### 7.4 内存共享

```c
// hw/virtio/vhost-user.c:997-1052
// vhost_user_set_mem_table():
//   将 QEMU 的内存映射通过 fd 传递给后端进程
//   后端 mmap() 这些 fd → 直接访问 Guest 内存
//   
//   消息格式: VhostUserMemory + fd 数组
//     每个区域: guest_phys_addr, memory_size, userspace_addr, mmap_offset
//   
//   热迁移时: 使用 ADD_MEM_REG/REM_MEM_REG 动态更新
```

---

## 8. vhost 数据路径优化

```
传统路径（无 vhost）:
  Guest kick → VM Exit → QEMU → 处理 I/O → 注入中断 → VM Entry

vhost-kernel 路径:
  Guest kick → VM Exit → KVM eventfd → 内核 vhost → 直接 I/O
  完成 → irqfd → KVM → Guest 中断（无 QEMU 参与）

vhost-user 路径:
  Guest kick → VM Exit → KVM eventfd → 用户态后端进程
  后端进程 → DMA 读写 Guest 内存 → eventfd → irqfd → Guest 中断

关键优化:
  ioeventfd: Guest 写 Notify → eventfd 信号（跳过 QEMU 主循环）
  irqfd: 后端 eventfd → KVM 直接注入中断（跳过 QEMU）
  零拷贝: 后端直接 mmap Guest 内存（无 QEMU 中介拷贝）
```

---

## 9. IOMMU 集成

### 9.1 VirtIO IOMMU 设备

```c
// include/hw/virtio/virtio-iommu.h:53-73
struct VirtIOIOMMU {
    VirtIODevice parent_obj;
    VirtQueue *req_vq;           // 请求队列（map/unmap 命令）
    VirtQueue *event_vq;         // 事件队列（故障通知）
    struct virtio_iommu_config config;  // 设备配置
    GTree *domains;              // 域树（domain_id → mapping table）
    GTree *endpoints;            // 端点树（BDF → endpoint）
    QemuRecMutex mutex;          // 保护并发访问
    GHashTable *as_by_busptr;    // 总线 → 地址空间映射
};
```

### 9.2 virtio_iommu_translate() — 地址翻译

```c
// hw/virtio/virtio-iommu.c:1137-1257
static IOMMUTLBEntry virtio_iommu_translate(IOMMUMemoryRegion *mr,
                                             hwaddr addr, ...)
{
    // 1. 获取设备 BDF (Bus/Device/Function)
    sid = virtio_iommu_get_bdf(sdev);
    
    // 2. 查找端点 → 域
    ep = g_tree_lookup(s->endpoints, GUINT_TO_POINTER(sid));
    
    // 3. 检查 bypass 模式
    if (!ep) {
        if (bypass_allowed) → 直通翻译 (IOVA = PA)
        else → 报告故障
    }
    
    // 4. 在域的映射表中查找 IOVA → PA 映射
    mapping_value = g_tree_lookup(ep->domain->mappings, &interval);
    
    // 5. 检查读写权限
    if (flag & IOMMU_WO && !(flags & VIRTIO_IOMMU_MAP_F_WRITE))
        → write fault
    
    // 6. 返回 IOMMUTLBEntry (target_as, translated_addr, perm)
}
```

### 9.3 DMA 地址空间

```c
// VirtIODevice.dma_as — include/hw/virtio/virtio.h:163
// 每个 VirtIO 设备可选配 IOMMU 感知的 DMA 地址空间

// 无 IOMMU: dma_as = &address_space_memory (直接物理访问)
// 有 IOMMU: dma_as = IOMMUDevice.as (通过 IOMMU 翻译)

// hw/virtio/virtio-iommu.c:403-448
// virtio_iommu_find_add_as(): 为设备创建 IOMMU 地址空间
//   1. memory_region_init_iommu() — 创建 IOMMUMemoryRegion
//   2. address_space_init() — 初始化地址空间
//   3. translate 回调 = virtio_iommu_translate

// VRing DMA 访问使用 dma_as:
// hw/virtio/virtio.c:281-308 — vring_desc_read/avail_read/used_write
//   address_space_read(vdev->dma_as, desc_addr, ...)
//   → 经过 IOMMU 翻译（如果启用）→ 实际物理地址
```

---

## 10. VirtIO 设备创建流程

```
1. QEMU 命令行: -device virtio-net-pci,...
   │
   ▼
2. QOM 实例化: virtio_net_pci_instance_init()
   → 创建 VirtIOPCIProxy + VirtIONet
   │
   ▼
3. realize: virtio_net_device_realize()
   → virtio_init(vdev, VIRTIO_ID_NET, config_size)
   → virtio_add_queue(vdev, 256, handle_rx)   // RX 队列
   → virtio_add_queue(vdev, 256, handle_tx)   // TX 队列
   → virtio_add_queue(vdev, 64, handle_ctrl)  // 控制队列
   │
   ▼
4. PCI realize: virtio_pci_realize()
   → 注册 BAR (common/isr/device/notify)
   → MSI-X 初始化
   → ioeventfd 设置
   │
   ▼
5. Guest 驱动: 读特性 → 写特性 → 配置队列 → DRIVER_OK
   │
   ▼
6. 开始 I/O: Guest 写 Notify → virtio_queue_notify()
```

---

## 11. 性能关键路径

```
热路径优化:
  
  VRingMemoryRegionCaches (virtio.c:100-105):
    为 desc/avail/used 区域创建 MemoryRegionCache
    避免每次访问都做 AddressSpace 查找
    → 直接 memcpy 到/从 Guest 内存
  
  ioeventfd (PCI notify → eventfd):
    Guest 写 Notify 寄存器
    → KVM 拦截 → 直接写 eventfd → 不触发完整 VM Exit
    → QEMU IOThread 或 vhost 后端直接收到通知
  
  Notification 抑制:
    VRING_AVAIL_F_NO_INTERRUPT: Guest 告诉设备暂不需要中断
    VRING_USED_F_NO_NOTIFY: 设备告诉 Guest 暂不需要 kick
    Event Index: 更精细的通知控制（仅达到特定索引时通知）
    → 批量处理时减少中断/kick 次数
  
  Packed Virtqueue (VirtIO 1.1):
    单一描述符数组替代 desc+avail+used 三个区域
    更好的缓存局部性（3 个区域 → 1 个区域）
    in-order 模式: 描述符按序完成（额外优化）
```

---

## 12. 小结

| 组件 | 实现 |
|------|------|
| **VirtIODevice** | QOM 设备基类，三层特性协商（host/guest/backend） |
| **VRing** | Split 模式: desc+avail+used 三区域，Packed 模式: 单一描述符数组 |
| **VirtQueue** | 核心 I/O 路径: pop(消费)→处理→push(完成)→notify(中断) |
| **PCI 传输** | Modern BAR (common/isr/device/notify)，MSI-X/INTx 中断 |
| **ioeventfd** | Guest kick → KVM eventfd → 跳过完整 VM Exit |
| **vhost-user** | Unix Socket + fd 传递，43 种消息类型，内存 mmap 共享 |
| **vhost 数据路径** | Guest → eventfd → 后端进程 → irqfd → Guest（绕过 QEMU） |
| **VirtIO IOMMU** | GTree 域/端点管理，IOMMUMemoryRegion 翻译回调 |
| **DMA 地址空间** | dma_as 可选 IOMMU 感知，VRing 访问经过 IOMMU 翻译 |
| **通知抑制** | NO_INTERRUPT/NO_NOTIFY 标志 + Event Index 精细控制 |
