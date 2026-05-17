# QEMU 网络子系统综合导航：TAP / SLIRP / VirtIO / vhost / 过滤器 / 多队列

> **定位**：为 `darren/network/` 目录建立一份中文导航文档，帮助读者从一篇深度分析文档快速定位 QEMU 网络子系统的核心框架、数据路径、后端实现与 virtio-net 高级特性。  
> **阅读前提**：建议具备基本的 QEMU 设备模型、事件循环与 VirtIO 概念。  
> **对应主文档**：[00-网络子系统深度分析.md](./00-网络子系统深度分析.md)

---

## 1. 标题和概述

QEMU 网络子系统承担 **Guest 网卡前端** 与 **Host 网络后端** 之间的数据转发、流控、过滤、队列调度与加速对接。它一端连接 `virtio-net/e1000/rtl8139` 等设备模型，另一端连接 `tap/slirp/socket/vhost-net` 等宿主侧实现；同时还通过过滤器、队列、多队列与 RSS 等机制，把“能跑”扩展到“可观测、可扩展、可优化”。

如果把 QEMU 看成一台分层虚拟机，网络子系统就是其中最典型的 **前后端解耦数据面**：
- 前端负责向 Guest 暴露网卡语义与 virtqueue；
- 核心层负责 `NetClientState`、`peer`、`NetQueue`、过滤器链与收发分发；
- 后端负责对接宿主机 TAP、用户态 NAT、Socket、内核 vhost 等能力；
- 高级特性层负责 MAC/VLAN 过滤、多队列、RSS、offload 等性能与功能增强。

---

## 2. 文档全景图

当前目录只有 1 篇主文档，但覆盖范围已经相当完整，可视为网络子系统的总入口。

| 文档 | 定位 | 覆盖重点 | 适合回答的问题 |
|---|---|---|---|
| [00-网络子系统深度分析.md](./00-网络子系统深度分析.md) | 网络子系统总纲 | 总体架构、核心结构、TX/RX、NetQueue、过滤器、TAP、SLIRP、Socket、vhost-net、virtio-net、MAC/VLAN、多队列、RSS、全链路追踪、源码索引 | QEMU 网络包怎样在前端、核心层、后端之间流转？TAP/SLIRP/vhost/virtio-net 的关系是什么？ |

**一句话使用建议：** 把 `00` 当成“网络主线地图”，先建立分层认知，再按问题类型跳到对应章节精读。

---

## 3. 知识地图

### 3.1 核心框架层

这一层回答“网络对象是什么、怎么创建、怎么连起来”。

| 主题 | 对应章节 | 关注点 |
|---|---|---|
| 总体架构 | 第 1 章 | 前端-核心-后端三层关系 |
| 核心数据结构总览 | 第 2 章 | `NetClientInfo` / `NetClientState` / `NICState` 关系 |
| 驱动操作表 | 第 3 章 | 各类后端如何通过回调表接入统一框架 |
| 网络客户端状态 | 第 4 章 | `peer`、`incoming_queue`、`filters`、`queue_index` |
| NIC 封装 | 第 5 章 | 设备侧如何持有一个或多个 `NetClientState` |
| 初始化流程 | 第 6 章 | `-netdev` 与 `-device` 如何分别创建后端和前端 |
| 后端驱动分发表 | 第 7 章 | `net_client_init_fun[]` 的类型分发 |
| Peer 对等连接机制 | 第 8 章 | 前端 `NetClientState` 与后端 `NetClientState` 如何互联 |

### 3.2 数据路径层

这一层回答“包怎么走、在哪里排队、在哪里被拦截”。

| 主题 | 对应章节 | 关注点 |
|---|---|---|
| TX 发送路径 | 第 9 章 | `qemu_send_packet_async()` 到 `qemu_deliver_packet_iov()` |
| RX 接收路径 | 第 10 章 | `qemu_receive_packet()` 如何驱动前端回调 |
| 队列层 | 第 11 章 | `NetQueue` 的缓冲、刷新、流控与延迟投递 |
| 过滤器框架 | 第 12 章 | `NetFilterState`、镜像/转储/缓冲类过滤器插入点 |

### 3.3 后端实现层

这一层回答“不同后端如何把数据面接到宿主机能力上”。

| 主题 | 对应章节 | 关注点 |
|---|---|---|
| TAP 后端 | 第 13 章 | `TAPState`、初始化、读写、fd 事件、宿主网桥接入 |
| SLIRP 用户态网络 | 第 14 章 | NAT、用户态协议栈、端口转发、无需 root 的网络接入 |
| Socket 后端 | 第 15 章 | UDP/TCP/Multicast 三种模式与点对点互联 |
| vhost-net 内核加速 | 第 16 章 | virtqueue 数据面下沉到内核，减少 QEMU 参与 |
| 其他网络后端 | 第 17 章 | VDE、netmap、L2TPv3、passt 等扩展后端 |

### 3.4 高级特性层

这一层回答“virtio-net 如何把网络功能做深、把性能做高”。

| 主题 | 对应章节 | 关注点 |
|---|---|---|
| virtio-net 设备模型 | 第 18 章 | 网络设备前端在 QEMU 中的总体形态 |
| VirtIONet 核心结构 | 第 19 章 | `VirtIONet`、`VirtIONetQueue`、队列对与设备状态 |
| 设备初始化 | 第 20 章 | realize、NIC 创建、队列初始化、后端绑定 |
| 特性协商 | 第 21 章 | 功能位、控制能力与前后端对齐 |
| virtio-net TX | 第 22 章 | virtqueue → QEMU 网络核心 → 后端 |
| virtio-net RX | 第 23 章 | 后端 → QEMU 网络核心 → virtqueue |
| Offload 卸载 | 第 24 章 | checksum/GSO/GRO 相关头与路径 |
| MAC 过滤 / VLAN | 第 25 章 | 接收模式、地址表、VLAN 过滤 |
| Multiqueue 多队列 | 第 26 章 | 队列扩展、并行收发、控制路径 |
| RSS 接收端扩展 | 第 27 章 | 哈希分流、队列选择、接收扩展 |

### 3.5 端到端追踪

这一层回答“从 Guest 到 Host、再从 Host 回 Guest，完整链路到底经过哪些函数”。

| 主题 | 对应章节 | 关注点 |
|---|---|---|
| 完整数据包流转 | 第 28 章 | TX/RX 全路径串联，适合建立全局时序图 |
| 关键源文件索引 | 第 29 章 | `net/`、`include/net/`、`hw/net/virtio-net*` 等入口文件 |

---

## 4. 核心架构速览

```text
+--------------------------------------------------------------------------------+
|                                Guest OS / Driver                               |
|                     virtio-net / e1000 / rtl8139 / ...                         |
+-----------------------------------------+--------------------------------------+
                                          |
                                          v
+--------------------------------------------------------------------------------+
|                           设备前端层（hw/net/）                                 |
|         virtio-net.c / e1000.c / rtl8139.c / 网卡设备 realize / virtqueue      |
+-----------------------------------------+--------------------------------------+
                                          |
                                          v
+--------------------------------------------------------------------------------+
|                           网络核心层（include/net/ + net/）                     |
|  NetClientInfo   NetClientState <----peer----> NetClientState   NICState       |
|  qemu_send_packet* / qemu_receive_packet / NetQueue / filter chain             |
+---------------------------+--------------------------+--------------------------+
                            |                          |
                            | 过滤/排队/流控           | 分发到后端 receive/can_receive
                            v                          v
+--------------------------------------------------------------------------------+
|                           后端实现层（net/*.c / hw/net/）                       |
|      TAP        SLIRP        Socket        vhost-net        其他后端            |
+-----------------------------------------+--------------------------------------+
                                          |
                                          v
+--------------------------------------------------------------------------------+
|                           Host kernel / userspace / NIC                         |
|           tap fd / socket fd / slirp stack / vhost kernel threads              |
+--------------------------------------------------------------------------------+
```

**最重要的心智模型：**
1. 一个网络连接通常表现为一对 `NetClientState`；
2. 两端通过 `peer` 互连；
3. 包的发送本质上是“本端调用发送接口，核心层把包投递给对端回调”；
4. `NetQueue` 和 `NetFilterState` 分别提供“排队/流控”和“观测/改写/旁路”能力；
5. `virtio-net` 是最重要的前端，`tap/slirp/socket/vhost-net` 是最常见的后端与加速路径。

---

## 5. 关键数据结构一览

| 结构体 | 所在层 | 角色 | 阅读重点 |
|---|---|---|---|
| `NetClientInfo` | 核心框架 | 网络驱动操作表/vtable | `receive`、`receive_iov`、`can_receive`、`cleanup`、offload 相关回调 |
| `NetClientState` | 核心框架 | 网络端点统一抽象 | `peer`、`incoming_queue`、`filters`、`queue_index`、`link_down` |
| `NICState` | 前端桥接层 | NIC 设备侧封装 | `ncs` 数组、多队列、`opaque` 指向具体设备 |
| `NetQueue` | 数据路径层 | 入站缓冲与流控队列 | 队列刷新、延迟交付、接收受阻后的重试 |
| `NetFilterState` | 数据路径层 | 过滤器统一抽象 | 过滤器链挂接位置、包拦截方向、mirror/dump/buffer |
| `TAPState` | TAP 后端 | TAP 文件描述符与宿主接口状态 | fd、读写回调、vnet header、事件处理 |
| `SlirpState` | SLIRP 后端 | 用户态 NAT / 协议栈状态 | NAT、端口转发、用户态收发接线 |
| `NetSocketState` | Socket 后端 | UDP/TCP/组播连接状态 | 连接模式、fd 事件、对端组织方式 |
| `VHostNetState` | vhost-net | 内核加速状态 | vhost 启停、通知器转移、内核数据面 |
| `VirtIONetQueue` | virtio-net | 队列对抽象 | TX/RX 队列、通知、与多队列的关系 |
| `VirtIONet` | virtio-net | virtio-net 主设备结构 | 特性位、MAC/VLAN、MQ/RSS、控制队列、后端绑定 |
| `virtio_net_hdr_v1` | offload | Guest/Host 间卸载元数据头 | checksum、GSO、分段与合包协同 |

---

## 6. 数据包生命周期

### 6.1 TX：Guest 发包到 Host

```text
Guest 驱动
  -> virtqueue 填充描述符
  -> virtio_net_flush_tx()
  -> qemu_send_packet_async()
  -> qemu_net_queue_send() / qemu_deliver_packet_iov()
  -> 过滤器链（可选）
  -> peer->info->receive / receive_iov
  -> tap_receive() / slirp_receive() / net_socket_receive() / vhost 路径
  -> Host 内核或用户态网络栈
```

**关键观察点：**
- `virtio_net_flush_tx()` 是 virtio-net 侧 TX 主入口；
- `qemu_send_packet_async()` 是网络核心统一发送入口；
- `qemu_deliver_packet_iov()` 完成最终投递；
- 真正的“落地”由后端自己的 `receive/receive_iov` 回调决定。

### 6.2 RX：Host 收包到 Guest

```text
Host tap/socket/slirp/vhost 数据到达
  -> 后端 fd 回调 / 后端收包函数
  -> qemu_receive_packet()
  -> qemu_net_queue_deliver() / 对端 receive 路径
  -> virtio_net_receive_rcu()
  -> 写入 RX virtqueue / 填充 virtio_net_hdr_v1（按需）
  -> 通知 Guest
  -> Guest 驱动取包
```

**关键观察点：**
- 后端通常先在事件循环中收到 fd 事件；
- `qemu_receive_packet()` 是宿主进包进入网络核心的统一入口；
- `virtio_net_receive_rcu()` 是 virtio-net 前端接收路径核心；
- 多队列/RSS 会影响“进来的包最终投递到哪个 RX 队列”。

### 6.3 生命周期中的三个控制点

| 控制点 | 作用 | 典型位置 |
|---|---|---|
| 流控 | 对端暂时不可接收时延迟处理 | `can_receive` / `receive_disabled` / `NetQueue` |
| 过滤 | 观测、镜像、限流、缓存或改写 | `NetFilterState` 链 |
| 加速 | 减少 QEMU 数据面参与 | `vhost-net`、offload、多队列、RSS |

---

## 7. 推荐阅读路径

### 7.1 面向初学者

**目标**：先搞清“对象是谁、包怎么走”。

建议顺序：
1. 第 1 章：总体架构  
2. 第 2-5 章：核心结构体  
3. 第 8 章：Peer 机制  
4. 第 9-10 章：TX/RX 主路径  
5. 第 28 章：完整流转追踪  
6. 第 29 章：按源码文件回跳

### 7.2 面向网络开发者

**目标**：快速定位后端实现、过滤器扩展点与设备接入方式。

建议顺序：
1. 第 6-8 章：初始化与连接机制  
2. 第 11-12 章：NetQueue 与过滤器框架  
3. 第 13-17 章：TAP / SLIRP / Socket / vhost-net / 其他后端  
4. 第 18-23 章：virtio-net 前端与 TX/RX 对接  
5. 第 29 章：结合源文件索引做源码穿透

### 7.3 面向性能优化者

**目标**：找到吞吐、延迟、并行度与加速相关的关键位置。

建议顺序：
1. 第 11 章：NetQueue 流控与排队  
2. 第 16 章：vhost-net 内核加速  
3. 第 22-24 章：virtio-net TX/RX 与 offload  
4. 第 26 章：Multiqueue 多队列  
5. 第 27 章：RSS 接收端扩展  
6. 第 28 章：全链路追踪，定位瓶颈插点

---

## 8. 与其他子系统的关联

网络子系统并不是孤立存在的，它与对象模型、事件循环、VirtIO、设备模型都高度耦合。建议结合以下文档交叉阅读：

| 关联方向 | 推荐文档 | 为什么要一起读 |
|---|---|---|
| 架构总览 | [../architecture/27-QEMU架构子系统综合导航-QOM-内存-TCG-VirtIO-事件循环-Block-PCI.md](../architecture/27-QEMU架构子系统综合导航-QOM-内存-TCG-VirtIO-事件循环-Block-PCI.md) | 先把网络放回 QEMU 全局框架中，明确它与 QOM、事件、VirtIO 的位置关系 |
| 事件循环 / fd 事件 | [../architecture/23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md](../architecture/23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md) | TAP、Socket、SLIRP 等后端都依赖 fd 事件与 AioContext 驱动 |
| VirtIO 传输与 virtqueue | [../architecture/26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md](../architecture/26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md) | 理解 virtio-net 时，必须补齐 VirtQueue、通知与传输层基础 |
| 设备模型总览 | [../device-model/09-设备模型子系统综合导航-VirtIO-Block-Chardev-VFIO-网络-DMA-Timer.md](../device-model/09-设备模型子系统综合导航-VirtIO-Block-Chardev-VFIO-网络-DMA-Timer.md) | 建立“前端设备 + 后端子系统”这一通用模式的整体认知 |
| 网络后端专题 | [../device-model/06-网络后端深度分析-TAP-vhost-net-vhost-user.md](../device-model/06-网络后端深度分析-TAP-vhost-net-vhost-user.md) | 如果关注 TAP / vhost / vhost-user / slirp，更适合作为后端专项补充材料 |

**推荐交叉顺序：**
- 想看“网络在全局架构中的位置”：先读 `architecture/27`；
- 想看“fd 事件为什么能驱动网络包收发”：补 `architecture/23`；
- 想看“virtio-net 队列怎么和设备模型接上”：补 `architecture/26`；
- 想看“网络后端的更宽视角”：补 `device-model/06` 与 `device-model/09`。

---

## 9. 待深入方向

当前主文档已经覆盖网络主线，但仍有一些值得继续扩展的主题：

| 方向 | 说明 |
|---|---|
| vhost-user / vDPA | 当前目录主文档重点是 `vhost-net`，若关注用户态数据面或硬件卸载，可单独展开 vhost-user、SVQ、vDPA |
| 非 virtio 传统网卡 | e1000、e1000e、rtl8139、vmxnet3 等设备模型行为与 virtio-net 差异值得单列分析 |
| 网络迁移与状态保存 | 迁移时网卡状态、队列、过滤器、vhost 协调如何保存/恢复 |
| eBPF / steering | `NetClientInfo` 中已出现 eBPF 导流相关回调，可继续追踪现代导流能力 |
| XDP / AF_XDP / io_uring 网络化 | 面向高性能用户态网络的宿主侧集成仍可扩展 |
| 安全与隔离 | 包过滤、反欺骗、MAC/VLAN 策略、宿主网络命名空间与权限模型 |
| 可观测性 | trace-events、filter-dump、pcap 导出、统计计数器与性能剖析方法 |
| 多线程数据面 | IOThread、datapath、锁粒度、BQL 影响与多队列并行度分析 |

---

## 结语

如果只记住一句话：**QEMU 网络子系统的核心就是“前端网卡 + NetClientState/peer + NetQueue/filter + 后端实现/加速路径”这条主线。** 先抓住这条主线，再分别进入 TAP、SLIRP、virtio-net、vhost、多队列和 RSS，就不容易迷失在大量源码细节里。
