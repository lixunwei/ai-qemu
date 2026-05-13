# QEMU 网络后端深度分析：TAP / vhost-net / vhost-user

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 分析范围：网络后端核心框架、TAP 设备、vhost-net 内核加速、vhost-user 用户态数据面、vDPA 硬件卸载、slirp 用户模式网络  
> 关联文档：[设备模型与virtio深度分析](00-设备模型与virtio深度分析.md) · [关键设备仿真分析](01-关键设备仿真分析-UART-磁盘-网卡.md)  
> 关键提交：`29e182003b` (cocci cleanup) · `c7c15bc178` (vhost-user 编译单元重构) · `25730acda4` (vDPA SVQ GSO whitelist)

---

## 目录

1. [源码规模与文件布局](#1-源码规模与文件布局)
2. [网络后端核心框架](#2-网络后端核心框架)
3. [NetClientState 核心数据结构](#3-netclientstate-核心数据结构)
4. [数据包收发核心路径](#4-数据包收发核心路径)
5. [NetQueue 流控机制](#5-netqueue-流控机制)
6. [NIC 前端注册与 Peer 链接](#6-nic-前端注册与-peer-链接)
7. [命令行解析与网络初始化](#7-命令行解析与网络初始化)
8. [Hub 虚拟交换机制](#8-hub-虚拟交换机制)
9. [NetFilter 数据包过滤框架](#9-netfilter-数据包过滤框架)
10. [TAP 后端核心](#10-tap-后端核心)
11. [TAP 设备创建与初始化](#11-tap-设备创建与初始化)
12. [TAP 数据包收发路径](#12-tap-数据包收发路径)
13. [TAP VNET Header 与 Offload](#13-tap-vnet-header-与-offload)
14. [TAP 多队列支持](#14-tap-多队列支持)
15. [TAP 高级特性](#15-tap-高级特性)
16. [vhost 核心框架](#16-vhost-核心框架)
17. [vhost-net 内核加速](#17-vhost-net-内核加速)
18. [vhost 内存与 IOMMU 跟踪](#18-vhost-内存与-iommu-跟踪)
19. [Shadow Virtqueue (SVQ)](#19-shadow-virtqueue-svq)
20. [vhost-user 用户态数据面](#20-vhost-user-用户态数据面)
21. [vhost-user 协议详解](#21-vhost-user-协议详解)
22. [vhost-user 重连与多队列](#22-vhost-user-重连与多队列)
23. [vhost-vDPA 硬件卸载](#23-vhost-vdpa-硬件卸载)
24. [slirp 用户模式网络](#24-slirp-用户模式网络)
25. [Socket 后端](#25-socket-后端)
26. [全链路数据路径对比](#26-全链路数据路径对比)
27. [端到端 TX/RX 流程图](#27-端到端-txrx-流程图)

**第七部分：virtio-net 设备仿真**
28. [virtio-net 概述](#28-virtio-net-概述)
29. [VirtIONet 结构体](#29-virtionet-结构体)
30. [设备 Realize 流程](#30-设备-realize-流程)
31. [RX 接收路径](#31-rx-接收路径)
32. [TX 发送路径](#32-tx-发送路径)
33. [控制队列](#33-控制队列)
34. [多队列与 RSS](#34-多队列与-rss)
35. [特性协商](#35-特性协商)
36. [网络后端与 vhost 集成](#36-网络后端与-vhost-集成)
37. [CLI 使用与配置](#37-cli-使用与配置)

**附录**
- [A. 关键数据结构速查表](#附录-a-关键数据结构速查表)
- [B. 源码文件索引](#附录-b-源码文件索引)
- [C. 关联文档索引](#附录-c-关联文档索引)

---

## 1. 源码规模与文件布局

网络后端涉及 16 个核心源文件，总计约 16,900 行：

| 文件 | 行数 | 职责 |
|------|------|------|
| `hw/virtio/vhost-user.c` | 3,066 | vhost-user 协议实现 |
| `hw/virtio/vhost.c` | 2,554 | vhost 核心框架 |
| `net/net.c` | 2,175 | 网络后端核心 |
| `net/vhost-vdpa.c` | 1,898 | vDPA 后端 |
| `net/slirp.c` | 1,310 | 用户模式网络 |
| `net/tap.c` | 1,066 | TAP 后端 |
| `net/socket.c` | 786 | Socket 后端 |
| `hw/virtio/vhost-shadow-virtqueue.c` | 782 | Shadow Virtqueue |
| `hw/net/vhost_net.c` | 697 | vhost-net 网络加速 |
| `net/filter-mirror.c` | 526 | 过滤器：mirror/redirector |
| `net/vhost-user.c` | 480 | vhost-user 网络后端 |
| `net/filter.c` | 377 | 过滤器框架 |
| `net/tap-linux.c` | 360 | TAP Linux 特定操作 |
| `net/hub.c` | 331 | Hub 虚拟交换 |
| `net/queue.c` | 293 | 网络队列 |
| `net/filter-buffer.c` | 207 | 缓冲过滤器 |

---

## 2. 网络后端核心框架

QEMU 网络子系统采用**前端-后端**分离架构：

```
+---------------------+          +---------------------+
|    NIC 前端          |          |    网络后端          |
|  (virtio-net/e1000) |  peer    |  (TAP/slirp/...)    |
|                     |<-------->|                     |
|  NetClientState     |          |  NetClientState     |
+---------------------+          +---------------------+
         |                                |
         |  NetClientInfo 回调            |  NetClientInfo 回调
         |  .receive / .receive_iov       |  .receive / .receive_iov
         v                                v
   qemu_send_packet()             tap_receive() / ...
```

核心设计原则：
- **NetClientState** 是所有网络端点的统一抽象（`net.h:86-134`）
- NIC 和 backend 各自拥有一个 NetClientState，通过 `peer` 指针互连
- 数据包通过回调函数在两端之间传递，方向由调用方决定
- **NetClientDriver** 枚举定义了所有支持的后端类型（`qapi/net.json:854-862`）

### 2.1 支持的后端类型

```c
// qapi/net.json:854-862 — NetClientDriver 枚举
{ 'none', 'nic', 'user',              // 基础类型
  'tap', 'l2tpv3', 'socket',          // 内核接口类型
  'stream', 'dgram',                   // 新式 socket 替代
  'vde', 'bridge', 'hubport',          // 桥接/hub 类型
  'netmap', 'vhost-user', 'vhost-vdpa', // 高性能类型
  'passt', 'af-xdp',                   // 新式旁路类型
  'vmnet-host', 'vmnet-shared', 'vmnet-bridged' } // macOS vmnet
```

---

## 3. NetClientState 核心数据结构

### 3.1 NetClientInfo — 回调接口

```c
// net.h:86-134
struct NetClientInfo {
    NetClientDriver type;
    size_t size;                        // 含私有数据的总大小

    /* 核心收发回调 */
    NetReceive *receive;                // 接收普通 buffer
    NetReceiveIOV *receive_iov;         // 接收 iov 散列 buffer
    NetCanReceive *can_receive;         // 流控：是否可接收

    /* 生命周期回调 */
    NetCleanup *cleanup;               // 销毁清理
    LinkStatusChanged *link_status_changed; // 链路状态变化

    /* 迁移回调 */
    NetSaveFD *save;
    NetLoadFD *load;

    /* 通知/轮询回调 */
    NetPoll *poll;
    HasUfo *has_ufo;                    // UFO offload 支持
    HasUso *has_uso;                    // USO offload 支持
    HasVnetHdr *has_vnet_hdr;           // VNET header 支持
    SetOffload *set_offload;            // 设置 offload 参数
    SetVnetHdrLen *set_vnet_hdr_len;    // VNET header 长度
    SetVnetBE *set_vnet_be;             // 大端模式
    SetSteeringEBPF *set_steering_ebpf; // eBPF RSS 导流
    NetAnnounce *announce;              // GARP 公告
};
```

### 3.2 NetClientState — 网络端点实例

```c
// net.h:86-134
struct NetClientState {
    NetClientInfo *info;                // 回调表指针
    int link_down;                      // 链路状态
    QTAILQ_ENTRY(NetClientState) next;  // 全局链表
    NetClientState *peer;               // 对端指针（NIC↔backend）
    NetQueue *incoming_queue;           // 入站队列（流控）
    char *model;                        // 设备型号
    char *name;                         // 实例名
    char info_str[256];                 // 描述字符串
    unsigned int queue_index;           // 多队列索引
    unsigned int receive_disabled : 1;  // 接收禁用标志
    QTAILQ_HEAD(, NetFilterState) filters; // 过滤器链
};
```

### 3.3 初始化流程

```c
// net.c:264-291
qemu_net_client_setup(NetClientState *nc, NetClientInfo *info,
                      NetClientState *peer, const char *model,
                      const char *name, NetClientDestructor *destructor)
{
    nc->info = info;
    nc->model = g_strdup(model);
    nc->name = name ? g_strdup(name) : assign_name(nc, model);
    nc->incoming_queue = qemu_new_net_queue(qemu_deliver_packet_iov, nc);
    nc->destructor = destructor;
    QTAILQ_INIT(&nc->filters);
    QTAILQ_INSERT_TAIL(&net_clients, nc, next);  // 加入全局链表

    // 建立双向 peer 连接
    if (peer) {
        nc->peer = peer;
        peer->peer = nc;
    }
}
```

---

## 4. 数据包收发核心路径

### 4.1 TX 路径：设备 → 后端

```
virtio-net TX              QEMU 网络核心               TAP 后端
    |                          |                          |
    +-- qemu_sendv_packet_async()                         |
    |   (net.c:884-924)        |                          |
    |                          |                          |
    +-- filter_receive_iov()   |                          |
    |   (net.c:647-693)        |                          |
    |   遍历 TX filters        |                          |
    |                          |                          |
    +-- qemu_net_queue_send_iov()                         |
    |   (queue.c:196-244)      |                          |
    |                          |                          |
    +-- qemu_deliver_packet_iov()                         |
        (net.c:829-882)        |                          |
        |                      |                          |
        +-- filter_receive_iov() (RX filters)             |
        |   (net.c:647-693)                               |
        |                                                 |
        +-- nc->info->receive_iov()                       |
            即 tap_receive_iov()  ----------------------->+
            (tap.c:152-170)                               |
                                                    write(tap_fd)
```

关键函数调用链：

```c
// net.c:884-924 — TX 入口
qemu_sendv_packet_async(NetClientState *sender, ...) {
    // 1. 检查对端是否存在和可接收
    if (!sender->peer || sender->peer->receive_disabled)
        return 0;

    // 2. 通过入站队列发送（支持异步）
    return qemu_net_queue_send_iov(
        sender->peer->incoming_queue,  // 对端的入站队列
        sender, flags, iov, iovcnt, sent_cb);
}
```

```c
// net.c:829-882 — 实际投递
qemu_deliver_packet_iov(NetClientState *sender,
                        unsigned flags, ...) {
    NetClientState *nc = sender->peer;

    // 1. 经过 RX 方向过滤器
    ret = filter_receive_iov(nc, NET_FILTER_DIRECTION_RX, ...);
    if (ret) return ret;

    // 2. 调用接收端的回调
    if (nc->info->receive_iov)
        ret = nc->info->receive_iov(nc, iov, iovcnt);
    else
        // 兼容：从 iov 拼成连续 buffer 调用 receive()
        ret = nc->info->receive(nc, buf, size);
    return ret;
}
```

### 4.2 RX 路径：后端 → 设备

```
TAP fd 可读                QEMU 网络核心               virtio-net
    |                          |                          |
    +-- tap_send()             |                          |
    |   (tap.c:195-247)        |                          |
    |   read(tap_fd, buf)      |                          |
    |                          |                          |
    +-- qemu_send_packet_async()                          |
        (net.c:729-775)        |                          |
        |                      |                          |
        +-- filter_receive_iov() (TX filters)             |
        |                                                 |
        +-- qemu_net_queue_send()                         |
        |   (queue.c:196-244)                             |
        |                                                 |
        +-- qemu_deliver_packet_iov()                     |
            |                                             |
            +-- filter_receive_iov() (RX filters)         |
            |                                             |
            +-- nc->info->receive()  -------------------->+
                即 virtio_net_receive()                   |
                (virtio-net.c)                      放入 VirtQueue
```

---

## 5. NetQueue 流控机制

### 5.1 NetQueue 结构

```c
// queue.c:52-61
struct NetQueue {
    void *opaque;                       // 回调上下文
    uint32_t nq_maxlen;                 // 最大队列长度
    uint32_t nq_count;                  // 当前排队包数
    bool delivering;                    // 是否正在投递（防重入）
    QTAILQ_HEAD(, NetPacket) packets;   // 排队的数据包链表
    NetQueueDeliverFunc *deliver;       // 投递回调
};
```

### 5.2 流控逻辑

```c
// queue.c:196-244
qemu_net_queue_send_iov(...) {
    // 1. 若队列为空且不在投递中，直接尝试投递
    if (queue->delivering || !QTAILQ_EMPTY(&queue->packets)) {
        // 队列中有积压 → 入队等待
        qemu_net_queue_append_iov(queue, ...);
        return 0;
    }

    // 2. 尝试直接投递
    ret = qemu_net_queue_deliver_iov(queue, ...);
    if (ret == 0) {
        // 对端暂不可接收 → 入队
        qemu_net_queue_append_iov(queue, ...);
        return 0;
    }
    return ret;
}
```

### 5.3 队列刷新

```c
// net.c:704-727
qemu_flush_queued_packets(NetClientState *nc) {
    // 刷新对端的入站队列
    nc->incoming_queue 中积压的包逐个重新投递
}

// net.c:1690-1707
// VM 状态变化时自动处理队列：
// - running → flush all queued packets
// - stopped → purge all queued packets
```

---

## 6. NIC 前端注册与 Peer 链接

### 6.1 NIC 创建

```c
// net.c:325-352
NICState *qemu_new_nic(NetClientInfo *info, NICConf *conf,
                       const char *model, const char *name, ...) {
    NICState *nic;
    int i, queues = MAX(1, conf->peers.queues);

    // 为每个队列创建一个 NetClientState
    nic = g_malloc0(info->size + sizeof(NetClientState) * queues);

    for (i = 0; i < queues; i++) {
        NetClientState *nc = &nic->ncs[i];
        // peer = conf->peers.ncs[i]（来自命令行 -netdev 匹配）
        qemu_net_client_setup(nc, info, conf->peers.ncs[i],
                              model, name, NULL);
        nc->queue_index = i;
    }
    return nic;
}
```

### 6.2 virtio-net 注册示例

```c
// virtio-net.c:3629-3637 — NIC 回调表
static NetClientInfo net_virtio_info = {
    .type = NET_CLIENT_DRIVER_NIC,
    .size = sizeof(NICState),
    .can_receive = virtio_net_can_receive,
    .receive = virtio_net_receive,
    .link_status_changed = virtio_net_set_link_status,
    .announce = virtio_net_announce,
};

// virtio-net.c:3867-4025
virtio_net_device_realize() {
    ...
    n->nic = qemu_new_nic(&net_virtio_info, &n->nic_conf, ...);
    ...
}
```

### 6.3 Peer 匹配

NIC 的 `conf->peers` 由命令行参数决定：
- `-netdev tap,id=net0` 创建名为 `net0` 的后端 NetClientState
- `-device virtio-net-pci,netdev=net0` 通过 `qemu_find_netdev("net0")`（`net.c:926-939`）找到后端
- 在 `qemu_new_nic()` 中建立双向 `peer` 连接

---

## 7. 命令行解析与网络初始化

### 7.1 初始化总入口

```c
// net.c:1916-1933
net_init_clients() {
    // 1. 处理所有 -netdev 参数
    qemu_opts_foreach(qemu_find_opts("netdev"), net_init_netdev, ...);

    // 2. 处理所有 -nic 参数
    qemu_opts_foreach(qemu_find_opts("nic"), net_param_nic, ...);

    // 3. 处理 legacy -net 参数
    qemu_opts_foreach(qemu_find_opts("net"), net_init_client, ...);
}
```

### 7.2 后端分发

```c
// net.c:1320-1377
net_client_init1(Netdev *netdev, ...) {
    NetClientDriver type = netdev->type;

    // 静态分发表：每种后端对应一个 net_init_xxx() 函数
    // 例如：
    //   NET_CLIENT_DRIVER_TAP → net_init_tap()
    //   NET_CLIENT_DRIVER_USER → net_init_slirp()
    //   NET_CLIENT_DRIVER_VHOST_USER → net_init_vhost_user()
    //   NET_CLIENT_DRIVER_VHOST_VDPA → net_init_vhost_vdpa()
    net_client_init_fun[type](netdev, name, peer, errp);
}
```

### 7.3 Legacy 模式默认 Hub

```c
// net.c:1348-1352
// legacy -net 模式下，没有显式 peer 的后端自动连接到 hub 0
if (!peer) {
    peer = net_hub_add_port(0, NULL, NULL);
}
```

---

## 8. Hub 虚拟交换机制

Hub 实现类似二层交换机的广播功能：

```c
// hub.c:32-45
struct NetHub {
    int id;                             // hub 编号
    QTAILQ_HEAD(, NetHubPort) ports;    // 端口链表
    QTAILQ_ENTRY(NetHub) next;          // 全局 hub 链表
};

struct NetHubPort {
    NetClientState nc;                  // 继承 NetClientState
    int id;
    NetHub *hub;                        // 所属 hub
    QTAILQ_ENTRY(NetHubPort) next;
};
```

### 8.1 广播逻辑

```c
// hub.c:49-77
net_hub_receive(NetHub *hub, NetHubPort *source_port,
                const uint8_t *buf, size_t len) {
    NetHubPort *port;
    // 遍历 hub 上所有端口，跳过源端口，向其他端口广播
    QTAILQ_FOREACH(port, &hub->ports, next) {
        if (port == source_port) continue;
        qemu_send_packet(&port->nc, buf, len);
    }
}
```

### 8.2 使用场景

```
Legacy 模式 (-net):
  -net nic -net tap        → 自动通过 hub 0 连接
  -net nic -net nic -net tap  → 两个 NIC + TAP 全在 hub 0，互相可达

Modern 模式 (-netdev + -device):
  直接 peer 连接，不经过 hub
```

---

## 9. NetFilter 数据包过滤框架

### 9.1 过滤器类接口

```c
// filter.h:38-48
struct NetFilterClass {
    ObjectClass parent_class;
    void (*setup)(NetFilterState *, Error **);
    void (*cleanup)(NetFilterState *);
    void (*status_changed)(NetFilterState *, Error **);
    void (*handle_event)(NetFilterState *, int, Error **);

    // 核心：拦截数据包
    ssize_t (*receive_iov)(NetFilterState *, NetClientState *sender,
                           unsigned flags, const struct iovec *iov,
                           int iovcnt, NetPacketSent *sent_cb);
};
```

### 9.2 过滤器在数据路径中的位置

```c
// net.c:647-693
filter_receive_iov(NetClientState *nc,
                   NetFilterDirection direction, ...) {
    NetFilterState *nf;
    // 遍历 nc 上挂载的所有 filter
    QTAILQ_FOREACH(nf, &nc->filters, next) {
        // 方向匹配（TX/RX/ALL）
        if (nf->direction == direction || nf->direction == NET_FILTER_DIRECTION_ALL) {
            ret = NETFILTER_GET_CLASS(nf)->receive_iov(nf, sender, ...);
            if (ret) return ret;  // 非0 = 过滤器已处理（拦截）
        }
    }
    return 0;  // 0 = 放行
}
```

### 9.3 内置过滤器类型

| 过滤器 | 文件 | 功能 |
|--------|------|------|
| filter-buffer | `filter-buffer.c` | 延迟释放缓冲，用于 COLO |
| filter-mirror | `filter-mirror.c:198-239` | 镜像数据包到另一个 chardev |
| filter-redirector | `filter-mirror.c` | 从 chardev 接收注入到网络 |
| filter-rewriter | `filter-rewriter.c:70-220` | TCP 连接重写（COLO） |
| filter-dump | `net/dump.c` | 抓包到文件（pcap 格式） |

---

## 10. TAP 后端核心

### 10.1 TAPState 数据结构

```c
// tap.c:70-86
typedef struct TAPState {
    NetClientState nc;                  // 继承网络端点
    int fd;                             // TAP 设备文件描述符
    char down_script[1024];             // 关闭脚本
    char down_script_arg[128];          // 脚本参数
    uint8_t buf[NET_BUFSIZE];           // 读缓冲区
    bool read_poll;                     // 是否监听可读
    bool write_poll;                    // 是否监听可写
    bool using_vnet_hdr;                // 是否使用 VNET header
    bool has_ufo;                       // UFO offload 支持
    bool has_uso;                       // USO offload 支持
    bool enabled;                       // 队列是否启用
    VHostNetState *vhost_net;           // vhost-net 加速器
    unsigned host_vnet_hdr_len;         // VNET header 长度
    Notifier exit_notifier;             // 退出通知
} TAPState;
```

### 10.2 TAP 回调表

```c
// tap.c 中注册
static NetClientInfo net_tap_info = {
    .type = NET_CLIENT_DRIVER_TAP,
    .size = sizeof(TAPState),
    .receive = tap_receive,             // 写入 TAP fd
    .receive_iov = tap_receive_iov,     // iov 写入 TAP fd
    .poll = tap_poll,
    .cleanup = tap_cleanup,
    .has_ufo = tap_has_ufo,
    .has_uso = tap_has_uso,
    .has_vnet_hdr = tap_has_vnet_hdr,
    .set_offload = tap_set_offload,
    .set_vnet_hdr_len = tap_set_vnet_hdr_len,
    .set_steering_ebpf = tap_set_steering_ebpf,
};
```

---

## 11. TAP 设备创建与初始化

### 11.1 net_init_tap() 多路径分发

```c
// tap.c:818-1030
net_init_tap(const Netdev *netdev, ...) {
    NetdevTapOptions *tap = &netdev->u.tap;

    if (tap->has_fd) {
        // 路径1: fd= 直接传入已打开的 TAP fd
        // tap.c:846-879
        fd = monitor_fd_param(tap->fd);
        net_init_tap_one(tap, peer, "tap", name, NULL, fd, ...);

    } else if (tap->has_fds) {
        // 路径2: fds= 多队列，逗号分隔多个 fd
        // tap.c:880-953
        for (i = 0; i < nfds; i++) {
            net_init_tap_one(tap, peer, "tap", name, NULL, fds[i], ...);
        }

    } else if (tap->has_helper) {
        // 路径3: helper= 使用 qemu-bridge-helper
        // tap.c:954-980
        fd = net_bridge_run_helper(tap->helper, tap->br, ...);
        net_init_tap_one(tap, peer, "bridge", name, NULL, fd, ...);

    } else {
        // 路径4: 默认路径 — 创建 TAP 设备
        // tap.c:986-1027
        int queues = tap->has_queues ? tap->queues : 1;
        for (i = 0; i < queues; i++) {
            fd = tap_open(ifname, sizeof(ifname), ...);
            if (i == 0) launch_script(tap->script, ifname);
            net_init_tap_one(tap, peer, "tap", name, ifname, fd, ...);
        }
    }
}
```

### 11.2 tap_open() — 打开 /dev/net/tun

```c
// tap-linux.c:40-134
int tap_open(char *ifname, int ifname_size, int *vnet_hdr,
             int vnet_hdr_required, int mq_required, Error **errp) {
    int fd = open("/dev/net/tun", O_RDWR);

    struct ifreq ifr;
    ifr.ifr_flags = IFF_TAP | IFF_NO_PI;  // TAP 模式，无协议信息头

    // VNET header 探测
    if (ioctl(fd, TUNGETFEATURES, &features) == 0) {
        if (features & IFF_VNET_HDR) {
            ifr.ifr_flags |= IFF_VNET_HDR;
            *vnet_hdr = 1;
        }
    }

    // 多队列支持
    if (mq_required) {
        ifr.ifr_flags |= IFF_MULTI_QUEUE;  // 每个队列独立 fd
    } else {
        ifr.ifr_flags |= IFF_ONE_QUEUE;    // 单队列优化
    }

    // 创建/附加 TAP 设备
    ioctl(fd, TUNSETIFF, &ifr);
    return fd;
}
```

### 11.3 TAP 设备标志说明

| 标志 | 含义 |
|------|------|
| `IFF_TAP` | TAP 模式（二层帧），而非 TUN（三层包） |
| `IFF_NO_PI` | 不附加协议信息头（4 字节 flags+proto） |
| `IFF_VNET_HDR` | 帧前附加 `virtio_net_hdr`，支持 offload 协商 |
| `IFF_MULTI_QUEUE` | 多队列模式，每个 fd 独立队列 |
| `IFF_ONE_QUEUE` | 单队列优化 |

---

## 12. TAP 数据包收发路径

### 12.1 RX：从 TAP fd 读取 → 送给 NIC

```c
// tap.c:195-247
static void tap_send(void *opaque) {   // GLib fd 可读回调
    TAPState *s = opaque;

    while (true) {
        // 1. 从 TAP fd 读取一帧
        size = tap_read_packet(s->fd, s->buf, sizeof(s->buf));
        if (size <= 0) break;

        // 2. 跳过 VNET header（不传给上层）
        // 若 using_vnet_hdr，前 host_vnet_hdr_len 字节是 virtio_net_hdr

        // 3. 送给 peer（NIC 前端）
        size = qemu_send_packet_async(&s->nc, s->buf, size, tap_send_completed);
        if (size == 0) {
            // NIC 不能接收 → 暂停读取 TAP fd
            tap_read_poll(s, false);
            break;
        }
    }
}
```

### 12.2 TX：从 NIC 接收 → 写入 TAP fd

```c
// tap.c:152-170
static ssize_t tap_receive_iov(NetClientState *nc,
                               const struct iovec *iov, int iovcnt) {
    TAPState *s = DO_UPCAST(TAPState, nc, nc);
    // 使用 writev() 零拷贝写入 TAP fd
    return tap_write_packet(s, iov, iovcnt);
}

// tap.c:138-150
static ssize_t tap_write_packet(TAPState *s,
                                const struct iovec *iov, int iovcnt) {
    ssize_t len = writev(s->fd, iov, iovcnt);
    if (len == -1 && errno == EINTR) {
        // 中断重试
        len = writev(s->fd, iov, iovcnt);
    }
    return len;
}

// tap.c:172-180
static ssize_t tap_receive(NetClientState *nc,
                           const uint8_t *buf, size_t size) {
    TAPState *s = DO_UPCAST(TAPState, nc, nc);
    struct iovec iov = { .iov_base = (char *)buf, .iov_len = size };
    return tap_write_packet(s, &iov, 1);
}
```

### 12.3 fd 事件管理

```c
// tap.c:109-115
static void tap_update_fd_handler(TAPState *s) {
    // 注册 GLib/QEMU 事件源
    qemu_set_fd_handler(s->fd,
                        s->read_poll ? tap_send : NULL,      // 可读回调
                        s->write_poll ? tap_writable : NULL,  // 可写回调
                        s);
}

// tap.c:117-127
tap_read_poll(TAPState *s, bool enable) {
    s->read_poll = enable;
    tap_update_fd_handler(s);
}

tap_write_poll(TAPState *s, bool enable) {
    s->write_poll = enable;
    tap_update_fd_handler(s);
}
```

---

## 13. TAP VNET Header 与 Offload

### 13.1 VNET Header 结构

```
+-------------------+---------------------------+
| virtio_net_hdr    |     以太网帧数据          |
| (10 或 12 字节)   |                           |
+-------------------+---------------------------+
```

`virtio_net_hdr` 包含：
- `flags`：需要校验和 offload 标志
- `gso_type`：GSO 类型（TCPv4/TCPv6/UDP/ECN）
- `hdr_len`：头部长度
- `gso_size`：GSO 分段大小
- `csum_start/csum_offset`：校验和位置

### 13.2 Offload 探测

```c
// tap-linux.c:156-169
int tap_probe_vnet_hdr(int fd) {
    // 检查内核是否支持 IFF_VNET_HDR
    if (ioctl(fd, TUNGETFEATURES, &features) != 0)
        return 0;
    return !!(features & IFF_VNET_HDR);
}

// tap-linux.c:171-204
int tap_probe_has_ufo(int fd) {
    // UFO = UDP Fragmentation Offload
    unsigned offload = TUN_F_UFO;
    return ioctl(fd, TUNSETOFFLOAD, offload) == 0;
}

int tap_probe_has_uso(int fd) {
    // USO = UDP Segmentation Offload (newer)
    unsigned offload = TUN_F_USO4 | TUN_F_USO6;
    return ioctl(fd, TUNSETOFFLOAD, offload) == 0;
}
```

### 13.3 Offload 配置

```c
// tap-linux.c:250-297
void tap_fd_set_offload(int fd, int csum, int tso4, int tso6,
                        int ecn, int ufo, int uso4, int uso6) {
    unsigned int offload = 0;
    if (csum) {
        offload |= TUN_F_CSUM;
        if (tso4) offload |= TUN_F_TSO4;
        if (tso6) offload |= TUN_F_TSO6;
        if (ecn)  offload |= TUN_F_TSO_ECN;
        if (ufo)  offload |= TUN_F_UFO;
        if (uso4) offload |= TUN_F_USO4;
        if (uso6) offload |= TUN_F_USO6;
    }
    ioctl(fd, TUNSETOFFLOAD, offload);
}
```

---

## 14. TAP 多队列支持

### 14.1 多队列架构

```
+------------------+       +------------------+
| virtio-net       |       |  TAP 设备        |
| Queue 0 -------> | peer  | TAP fd[0]        |
| Queue 1 -------> | peer  | TAP fd[1]        |
| Queue 2 -------> | peer  | TAP fd[2]        |
| ...              |       | ...              |
+------------------+       +------------------+
```

### 14.2 队列动态启用/禁用

```c
// tap.c:1033-1066
int tap_enable(NetClientState *nc) {
    TAPState *s = DO_UPCAST(TAPState, nc, nc);
    int ret = tap_fd_enable(s->fd);     // TUNSETQUEUE + IFF_ATTACH_QUEUE
    if (ret == 0) {
        s->enabled = true;
        tap_update_fd_handler(s);       // 重新注册 fd handler
    }
    return ret;
}

int tap_disable(NetClientState *nc) {
    TAPState *s = DO_UPCAST(TAPState, nc, nc);
    tap_read_poll(s, false);
    tap_write_poll(s, false);
    int ret = tap_fd_disable(s->fd);    // TUNSETQUEUE + IFF_DETACH_QUEUE
    if (ret == 0) s->enabled = false;
    return ret;
}

// tap-linux.c:299-333
int tap_fd_enable(int fd) {
    struct ifreq ifr = { .ifr_flags = IFF_ATTACH_QUEUE };
    return ioctl(fd, TUNSETQUEUE, &ifr);
}
int tap_fd_disable(int fd) {
    struct ifreq ifr = { .ifr_flags = IFF_DETACH_QUEUE };
    return ioctl(fd, TUNSETQUEUE, &ifr);
}
```

virtio-net 通过控制队列命令触发 TAP 队列的启用/禁用。

---

## 15. TAP 高级特性

### 15.1 Bridge Helper

```c
// tap.c:537-636
static int net_bridge_run_helper(const char *helper,
                                 const char *bridge, ...) {
    // fork + exec qemu-bridge-helper
    // helper 进程拥有 CAP_NET_ADMIN 特权
    // 通过 socketpair 传回已配置好的 TAP fd
    // 用户无需 root 权限即可创建桥接 TAP
}

// tap.c:638-668
net_init_bridge(const Netdev *netdev, ...) {
    fd = net_bridge_run_helper(helper, br, ...);
    net_init_tap_one(tap, peer, "bridge", name, NULL, fd, ...);
}
```

### 15.2 eBPF RSS 导流

```c
// tap-linux.c:349-360
int tap_fd_set_steering_ebpf(int fd, int prog_fd) {
    // 将 eBPF RSS 程序附加到 TAP 设备
    // 实现基于包头的多队列导流
    return ioctl(fd, TUNSETSTEERINGEBPF, &prog_fd);
}

// tap.c:366-372
int tap_set_steering_ebpf(NetClientState *nc, int prog_fd) {
    TAPState *s = DO_UPCAST(TAPState, nc, nc);
    return tap_fd_set_steering_ebpf(s->fd, prog_fd);
}
```

### 15.3 sndbuf 优化

```c
// tap.c:714-719
// net_init_tap_one() 中：
// 默认设置 sndbuf = INT_MAX
// 这允许 TAP fd 缓冲区无限大，避免因缓冲区满导致的丢包
tap_set_sndbuf(s->fd, tap->sndbuf ? tap->sndbuf : INT_MAX);
```

---

## 16. vhost 核心框架

### 16.1 struct vhost_dev

```c
// vhost.h:75-137
struct vhost_dev {
    VirtIODevice *vdev;                 // 关联的 VirtIO 设备
    MemoryListener memory_listener;     // 内存变化监听器
    MemoryListener iommu_listener;      // IOMMU 变化监听器
    struct vhost_virtqueue *vqs;        // vhost virtqueue 数组
    unsigned int nvqs;                  // virtqueue 数量
    uint64_t features;                  // 协商的特性
    uint64_t acked_features;            // 已确认的特性
    uint64_t backend_features;          // 后端特性
    uint64_t protocol_features;         // 协议特性
    uint64_t max_queues;                // 最大队列数
    uint64_t backend_cap;               // 后端能力
    bool started;                       // 是否已启动
    bool log_enabled;                   // 脏页日志
    int log_size;                       // 日志大小
    const VhostOps *vhost_ops;          // 后端操作表
    void *opaque;                       // 后端私有数据
    struct vhost_log *log;              // 脏页日志
    QLIST_HEAD(, vhost_iommu) iommu_list; // IOMMU 映射列表
    IOMMUNotifier n;                    // IOMMU 通知器
};
```

### 16.2 初始化序列

```c
// vhost.c:1552-1663
int vhost_dev_init(struct vhost_dev *hdev, void *opaque,
                   VhostBackendType backend_type, ...) {
    // 1. 选择后端操作表
    hdev->vhost_ops = vhost_set_backend_type(hdev, backend_type);

    // 2. 初始化后端（打开设备/socket）
    hdev->vhost_ops->vhost_backend_init(hdev, opaque);

    // 3. 获取后端特性
    hdev->vhost_ops->vhost_get_features(hdev, &hdev->features);

    // 4. 分配 virtqueue 数组
    hdev->vqs = g_new0(struct vhost_virtqueue, hdev->nvqs);

    // 5. 设置 owner
    hdev->vhost_ops->vhost_set_owner(hdev);

    // 6. 注册内存监听器
    memory_listener_register(&hdev->memory_listener, &address_space_memory);

    return 0;
}
```

### 16.3 Virtqueue 启动

```c
// vhost.c:1257-1368
static int vhost_virtqueue_start(struct vhost_dev *dev,
                                 struct VirtIODevice *vdev,
                                 struct vhost_virtqueue *vq,
                                 unsigned idx) {
    // 1. 获取 vring 地址
    virtio_queue_get_ring_addr(vdev, idx, &desc, &avail, &used);

    // 2. 设置 vring 参数
    vhost_set_vring_num(dev, &state);       // 队列大小
    vhost_set_vring_base(dev, &state);      // 起始 index
    vhost_set_vring_addr(dev, &addr);       // desc/avail/used 地址

    // 3. 设置 kick/call eventfd
    vhost_set_vring_kick(dev, &file);       // guest → host 通知
    vhost_set_vring_call(dev, &file);       // host → guest 通知

    return 0;
}
```

---

## 17. vhost-net 内核加速

### 17.1 架构对比

```
=== 普通 TAP 模式 ===                    === vhost-net 模式 ===

Guest virtio-net                         Guest virtio-net
    |                                        |
    v                                        v
[KVM EXIT]                               [KVM EXIT]
    |                                        |
    v                                        v
QEMU virtio-net                          QEMU virtio-net
    |                                     (仅控制面)
    v                                        |
QEMU TAP backend                         vhost-net 内核模块
    |                                        |
    v                                        v
  TAP fd                                   TAP fd
    |                                        |
    v                                        v
Linux 网络栈                              Linux 网络栈
```

### 17.2 vhost-net 初始化

```c
// vhost_net.c:232-310
struct vhost_net *vhost_net_init(VhostNetOptions *options) {
    struct vhost_net *net = g_new0(struct vhost_net, 1);

    // 1. 选择后端类型
    if (options->backend_type == VHOST_BACKEND_TYPE_KERNEL) {
        // 打开 /dev/vhost-net
        r = vhost_net_get_fd(options);
    }

    // 2. 初始化 vhost_dev
    vhost_dev_init(&net->dev, options->opaque,
                   options->backend_type, ...);

    // 3. 获取 TAP fd 用于后端设置
    if (backend_type == VHOST_BACKEND_TYPE_KERNEL) {
        net->dev.vhost_ops->vhost_set_vring_call(...);
    }

    return net;
}
```

### 17.3 启动数据面

```c
// vhost_net.c:325-406
static int vhost_net_start_one(struct vhost_net *net,
                               VirtIODevice *dev, int idx) {
    // 1. 启动 vhost 设备（设置 vring、内存等）
    vhost_dev_start(&net->dev, dev, false);

    // 2. 将 TAP fd 传给 vhost-net
    //    内核 vhost-net 直接从 TAP fd 收发数据
    file.fd = net->backend;
    vhost_net_set_backend(&net->dev, &file);

    // 3. 此后 QEMU 的 tap_send()/tap_receive() 不再被调用
    //    数据面完全在内核处理
}

// vhost_net.c:412+
int vhost_net_start(VirtIODevice *dev, NetClientState *ncs,
                    int data_queue_pairs, int cvq) {
    for (i = 0; i < data_queue_pairs; i++) {
        // 对每个数据队列对启动 vhost-net
        vhost_net_start_one(get_vhost_net(ncs[i].peer), dev, i * 2);
        vhost_net_start_one(get_vhost_net(ncs[i].peer), dev, i * 2 + 1);
    }
}
```

### 17.4 vhost-net 性能优势

| 特性 | 普通 TAP | vhost-net |
|------|---------|-----------|
| 数据拷贝 | Guest→QEMU→TAP（2次用户/内核切换） | Guest→内核 vhost→TAP（0次用户态切换） |
| 中断处理 | QEMU eventfd→用户态处理→注入 | irqfd 直接内核注入 |
| 上下文切换 | 每个包至少2次 | 接近零（batched） |
| CPU 开销 | 高（QEMU 用户态处理） | 低（内核态直通） |

---

## 18. vhost 内存与 IOMMU 跟踪

### 18.1 内存区域跟踪

```c
// vhost.c:616-852
// vhost 通过 MemoryListener 跟踪 guest 内存布局变化

static void vhost_begin(MemoryListener *listener) {
    // 开始一次内存映射更新事务
}

static void vhost_region_addnop(MemoryListener *listener,
                                MemoryRegionSection *section) {
    // 记录新增/变化的内存区域
    // 用于构建 VHOST_SET_MEM_TABLE 参数
}

static void vhost_commit(MemoryListener *listener) {
    // 提交内存映射变化
    // 调用 VHOST_SET_MEM_TABLE ioctl 更新内核 vhost
    vhost_dev_set_mem_table(dev);
}
```

### 18.2 IOMMU 集成

```c
// vhost.c:866-926
static void vhost_iommu_region_add(MemoryListener *listener,
                                   MemoryRegionSection *section) {
    // 对 IOMMU 保护的内存区域
    // 注册 IOMMU 通知器，跟踪地址翻译变化
    memory_region_register_iommu_notifier(section->mr, &iommu->n, ...);
}

static void vhost_iommu_region_del(MemoryListener *listener,
                                   MemoryRegionSection *section) {
    memory_region_unregister_iommu_notifier(section->mr, &iommu->n);
}

// vhost.c:928-947
// IOMMU 失效通知：当 IOMMU 映射变化时
// 通知 vhost 后端刷新其 IOTLB
static void vhost_iommu_region_notifier(...) {
    vhost_device_iotlb_miss(dev, iova, ...);
}
```

---

## 19. Shadow Virtqueue (SVQ)

SVQ 在 QEMU 用户态维护一个影子 virtqueue，拦截 guest 与 backend 之间的通信：

```c
// vhost-shadow-virtqueue.c:27-63
// SVQ 特性验证：检查后端和 guest 的特性兼容性

// vhost-shadow-virtqueue.c:271-520
// SVQ 核心操作：
// - vhost_svq_add()：将 guest 描述符翻译并添加到影子队列
// - vhost_svq_flush()：将影子队列提交给后端
// - vhost_svq_poll()：轮询后端完成
```

### 19.1 SVQ 使用场景

```
Guest                SVQ (QEMU)              Backend (vDPA HW)
  |                    |                        |
  +-- desc/avail -->   |                        |
  |                    +-- 翻译 GPA→HPA         |
  |                    +-- desc/avail -------->  |
  |                    |                        +-- 处理
  |                    |    <-- used ---------- +
  |                    +-- 翻译回 GPA           |
  |  <-- used -------- +                        |
```

**核心用途**：
- **vDPA 迁移**：SVQ 拦截 CVQ（控制队列），在迁移时记录和重放设备状态
- **地址翻译**：当后端不能直接访问 guest 内存时，SVQ 做 GPA→HPA 翻译

---

## 20. vhost-user 用户态数据面

### 20.1 架构

```
QEMU                     Unix Socket              DPDK/OVS vhost-user
+-------------------+    (fd passing)     +----------------------------+
| VirtIO 设备       |                     | vhost-user backend         |
| (控制面)          |    控制消息          | (数据面)                   |
|                   | <================> |                            |
| vhost-user.c      |                     | 直接访问 guest 共享内存    |
+-------------------+                     +----------------------------+
        |                                           |
        +-- 共享内存 (mmap) -------------------------+
            guest RAM 直接映射到 backend 进程
```

### 20.2 VhostUserState

```c
// net/vhost-user.c:66-74
typedef struct VhostUserState {
    NetClientState nc;                  // 网络端点
    CharBackend chr;                    // chardev 后端（Unix socket）
    VHostNetState *vhost_net;           // vhost-net 实例
    guint watch;                        // GLib 事件源 ID
    uint64_t acked_features;            // 已确认特性
    bool started;                       // 是否已启动
} VhostUserState;
```

### 20.3 初始化流程

```c
// net/vhost-user.c:371-480
net_init_vhost_user(const Netdev *netdev, ...) {
    // 1. 获取 chardev（Unix socket）
    chr = net_vhost_claim_chardev(netdev->u.vhost_user, ...);

    // 2. 验证 chardev 是否支持 reconnect（推荐）
    if (!chr->reconnecting) warn("no reconnect");

    // 3. 验证 chardev 是否支持 fd passing
    if (!qemu_chr_has_feature(chr, QEMU_CHAR_FEATURE_FD_PASS))
        error("chardev must support fd passing");

    // 4. 创建 NetClientState
    nc = qemu_new_net_client(&net_vhost_user_info, ...);

    // 5. 连接并启动 vhost-user
    vhost_user_start(peer_id, chr, ...);
}
```

---

## 21. vhost-user 协议详解

### 21.1 消息格式

```c
// vhost-user.c:57-103
typedef struct VhostUserMsg {
    int request;                        // 请求类型（枚举）
    uint32_t flags;                     // 标志（版本、回复请求等）
    uint32_t size;                      // payload 大小
    union {
        uint64_t u64;
        struct vhost_vring_state state;
        struct vhost_vring_addr addr;
        VhostUserMemory memory;         // 含 fd 数组用于内存共享
        VhostUserLog log;
        VhostUserConfig config;
        ...
    } payload;
} VhostUserMsg;
```

### 21.2 关键消息类型

| 消息 | 方向 | 用途 |
|------|------|------|
| `GET_FEATURES` | QEMU→Backend | 查询后端支持的特性 |
| `SET_FEATURES` | QEMU→Backend | 设置协商后的特性 |
| `SET_MEM_TABLE` | QEMU→Backend | 传递 guest 内存区域 + mmap fd |
| `SET_VRING_NUM` | QEMU→Backend | 设置 vring 大小 |
| `SET_VRING_ADDR` | QEMU→Backend | 设置 vring desc/avail/used 地址 |
| `SET_VRING_KICK` | QEMU→Backend | 传递 kick eventfd |
| `SET_VRING_CALL` | QEMU→Backend | 传递 call eventfd |
| `SET_LOG_BASE` | QEMU→Backend | 迁移脏页日志基址 |
| `GET_QUEUE_NUM` | QEMU→Backend | 查询多队列数 |

### 21.3 内存共享机制

```c
// vhost-user.c:476-553, 997-1050
// VHOST_USER_SET_MEM_TABLE 消息：
// 1. QEMU 枚举 guest RAM 区域
// 2. 每个区域的 mmap fd 通过 Unix socket 的 SCM_RIGHTS 辅助消息传递
// 3. Backend 收到 fd 后 mmap 到自己的地址空间
// 4. 此后 Backend 可直接读写 guest RAM（零拷贝）

vhost_user_set_mem_table(dev, ...) {
    // 构造 VhostUserMemory
    for (i = 0; i < dev->mem->nregions; i++) {
        msg.payload.memory.regions[i] = {
            .guest_phys_addr = region->guest_phys_addr,
            .memory_size = region->memory_size,
            .userspace_addr = region->userspace_addr,
            .mmap_offset = region->mmap_offset,
        };
        fds[i] = region->fd;  // mmap fd
    }
    // 通过 sendmsg + SCM_RIGHTS 发送
    vhost_user_write(dev, &msg, fds, fd_num);
}
```

### 21.4 特性协商

```c
// vhost-user.c:1111-1170, 2164-2288
vhost_user_backend_init(struct vhost_dev *dev, ...) {
    // 1. GET_FEATURES — 获取后端支持的 VirtIO 特性
    vhost_user_get_features(dev, &features);

    // 2. 协议特性协商
    //    VHOST_USER_PROTOCOL_F_MQ — 多队列
    //    VHOST_USER_PROTOCOL_F_LOG_SHMFD — 脏页日志
    //    VHOST_USER_PROTOCOL_F_REPLY_ACK — 回复确认
    //    VHOST_USER_PROTOCOL_F_BACKEND_REQ — 后端主动请求
    //    VHOST_USER_PROTOCOL_F_CROSS_ENDIAN — 跨端序
    vhost_user_get_u64(dev, VHOST_USER_GET_PROTOCOL_FEATURES, &protocol_features);

    // 3. SET_FEATURES — 设置最终协商结果
    vhost_user_set_features(dev, features);
}
```

### 21.5 协议特性标志

```c
// include/hw/virtio/vhost-user.h:14-36
#define VHOST_USER_PROTOCOL_F_MQ                0  // 多队列
#define VHOST_USER_PROTOCOL_F_LOG_SHMFD         1  // 脏页日志共享内存
#define VHOST_USER_PROTOCOL_F_RARP              2  // RARP 注入
#define VHOST_USER_PROTOCOL_F_REPLY_ACK         3  // 回复确认
#define VHOST_USER_PROTOCOL_F_MTU               4  // MTU 协商
#define VHOST_USER_PROTOCOL_F_BACKEND_REQ       5  // 后端请求通道
#define VHOST_USER_PROTOCOL_F_CROSS_ENDIAN      6  // 跨端序
#define VHOST_USER_PROTOCOL_F_CRYPTO_SESSION    7  // 加密会话
#define VHOST_USER_PROTOCOL_F_PAGEFAULT         8  // 页错误处理
#define VHOST_USER_PROTOCOL_F_CONFIG            9  // 配置空间
// ... 更多特性
```

---

## 22. vhost-user 重连与多队列

### 22.1 断开重连

```c
// net/vhost-user.c:281-317, 319-362
// 当 Unix socket 断开时：
vhost_user_event(void *opaque, QEMUChrEvent event) {
    if (event == CHR_EVENT_CLOSED) {
        // 异步关闭当前 vhost-user 连接
        vhost_user_async_close(nc, ...);
        // chardev 的 reconnect 特性会自动重连
        // 重连后触发 CHR_EVENT_OPENED → 重新初始化
    }
}

// hw/virtio/vhost-user.c: vhost_user_async_close
// 在 BH 中安全地停止 vhost-user 数据面
// 等待所有 inflight I/O 完成后清理资源
```

### 22.2 多队列支持

```c
// vhost-user.c:2238-2248
// 多队列通过 VHOST_USER_PROTOCOL_F_MQ 特性协商
// QEMU 查询 GET_QUEUE_NUM 获取后端支持的最大队列数
vhost_user_get_queue_num(dev, &queue_num);

// 每个队列使用独立的 vring 配置
// virtio-net 根据 guest 请求动态启用/禁用队列
```

---

## 23. vhost-vDPA 硬件卸载

### 23.1 架构定位

```
=== vhost-net ===              === vhost-vDPA ===
内核 vhost-net 模块            vDPA 硬件设备
    |                              |
  TAP fd                      /dev/vhost-vdpa-X
    |                              |
Linux 网络栈                   硬件 NIC（直接处理）
    |                              |
物理 NIC                      物理 NIC
```

### 23.2 VhostVDPAState

```c
// net/vhost-vdpa.c:34-50
typedef struct VhostVDPAState {
    NetClientState nc;                  // 网络端点
    struct vhost_vdpa vdpa;             // vDPA 设备状态
    VHostNetState *vhost_net;           // vhost-net 实例
    bool started;
    bool always_svq;                    // 始终使用 SVQ
    bool cvq_isolated;                  // 控制队列隔离
    VhostVDPAMigrationState migration_state; // 迁移状态
};
```

### 23.3 初始化

```c
// net/vhost-vdpa.c:998+
net_init_vhost_vdpa(const Netdev *netdev, ...) {
    // 1. 打开 /dev/vhost-vdpa-X
    fd = qemu_open(opts->vhostdev, O_RDWR, ...);

    // 2. 设置后端类型为 VHOST_BACKEND_TYPE_VDPA
    //    vhost-vdpa 使用与 vhost-net 相同的 ioctl 接口
    //    但操作的是硬件 vDPA 设备而非内核模块

    // 3. SVQ 配置（用于迁移支持）
    if (always_svq) {
        // 控制队列通过 SVQ 拦截
        // 数据队列直接走硬件
    }
}
```

### 23.4 vDPA 与 vhost-net 区别

| 特性 | vhost-net | vhost-vDPA |
|------|-----------|------------|
| 数据面位置 | 内核 vhost-net 模块 | 硬件 NIC |
| 设备文件 | /dev/vhost-net | /dev/vhost-vdpa-X |
| 内核拷贝 | TAP→网络栈（有拷贝） | 硬件直接处理（零拷贝） |
| 迁移支持 | 标准 virtio 迁移 | SVQ 拦截控制面 + 状态保存 |
| 灵活性 | 任意 TAP 配置 | 依赖硬件能力 |
| 性能 | 高 | 最高（接近裸机） |

---

## 24. slirp 用户模式网络

### 24.1 架构

```
Guest                   QEMU slirp                    Host
+----------+     +------------------------+     +-----------+
| eth0     |     | 用户态 TCP/IP 栈       |     | 普通      |
| 10.0.2.x | --> | NAT + DHCP + DNS       | --> | socket    |
|          |     | + TFTP + 端口转发       |     | connect() |
+----------+     +------------------------+     +-----------+
                 无需 root 权限
                 无法接收外部连接（除端口转发）
```

### 24.2 SlirpState

```c
// slirp.c:89-99
typedef struct SlirpState {
    NetClientState nc;                  // 网络端点
    QTAILQ_ENTRY(SlirpState) entry;    // 全局链表
    Slirp *slirp;                       // libslirp 实例
    Notifier poll_notifier;             // 轮询通知
    Notifier exit_notifier;             // 退出通知
    // 内置服务配置
    char *bootfile;                     // TFTP 启动文件
    char *tftp_prefix;                  // TFTP 根目录
    // ...
} SlirpState;
```

### 24.3 初始化与内置服务

```c
// slirp.c:431-709
net_init_slirp(const Netdev *netdev, ...) {
    NetdevUserOptions *user = &netdev->u.user;

    // 1. 网络配置
    //    默认网段: 10.0.2.0/24
    //    默认网关: 10.0.2.2（也是 DNS）
    //    默认 DHCP 起始: 10.0.2.15
    net = 0x0a000200;       // 10.0.2.0
    mask = 0xffffff00;      // /24

    // 2. 创建 libslirp 实例
    slirp = slirp_new(&cfg, &slirp_cb, s);

    // 3. 配置端口转发
    //    hostfwd: host:port → guest:port
    //    guestfwd: guest 连接 → host 程序的 stdin/stdout
    for (each hostfwd in user->hostfwd) {
        slirp_hostfwd(s, hostfwd, ...);   // slirp.c:802-955
    }
    for (each guestfwd in user->guestfwd) {
        slirp_guestfwd(s, guestfwd, ...); // slirp.c:683-690
    }
}
```

### 24.4 slirp 局限性

| 局限 | 说明 |
|------|------|
| 无原始套接字 | 不能发送/接收 ICMP（除非有 CAP_NET_RAW） |
| 性能较低 | 完全用户态 TCP/IP 栈，每个包都要协议处理 |
| 无外部接入 | 默认不监听端口，需显式配置 hostfwd |
| 无二层功能 | 不支持 bridge、VLAN 等二层特性 |

---

## 25. Socket 后端

### 25.1 用途

Socket 后端用于**连接多个 QEMU 实例**，构建虚拟网络：

```
QEMU-1                    QEMU-2
+---------+  TCP/UDP/mcast  +---------+
| socket  | <============> | socket  |
| backend |                | backend |
+---------+                +---------+
```

### 25.2 连接模式

```c
// socket.c:423-786

// 模式1: TCP listen/connect
// -netdev socket,id=s0,listen=:1234        (QEMU-1, server)
// -netdev socket,id=s0,connect=host:1234   (QEMU-2, client)

// 模式2: UDP multicast
// -netdev socket,id=s0,mcast=230.0.0.1:1234   (多个 QEMU 加入同一组播)

// 模式3: UDP unicast
// -netdev socket,id=s0,udp=host:port,localaddr=local:port
```

### 25.3 数据结构

```c
// socket.c:36-46
typedef struct NetSocketState {
    NetClientState nc;                  // 网络端点
    int listen_fd;                      // 监听 fd（server 模式）
    int fd;                             // 数据 fd
    SocketReadState rs;                 // 读状态（帧解析）
    unsigned int send_index;            // 发送偏移
    // ...
} NetSocketState;
```

---

## 26. 全链路数据路径对比

### 26.1 各后端数据拷贝次数

```
后端类型        Guest→NIC   QEMU处理   后端→Host    总拷贝数    特点

slirp           copy        用户态     socket       3+         最慢，最易用
                            TCP/IP栈                           无需权限

TAP             copy        QEMU      copy→fd      2          标准方案
                            用户态     TAP→内核栈              需要 root/CAP

vhost-net       共享内存    内核       内核→TAP     1          高性能
                vring       vhost-net  直接操作              需要 /dev/vhost-net

vhost-user      共享内存    用户态     mmap直接     0-1        DPDK 方案
                vring       DPDK       访问                   需要大页内存

vDPA            共享内存    硬件       硬件直接     0          最高性能
                vring       vDPA NIC   处理                   需要硬件支持
```

### 26.2 性能层次图

```
性能 (高→低)
  ^
  |  vDPA (硬件卸载，接近裸机)
  |  ├── 零拷贝，硬件直接处理 vring
  |
  |  vhost-user + DPDK (用户态 PMD)
  |  ├── 共享内存 mmap，DPDK 轮询模式
  |
  |  vhost-net (内核数据面)
  |  ├── 内核直接操作 vring + TAP
  |  ├── irqfd 减少上下文切换
  |
  |  TAP (标准模式)
  |  ├── QEMU 用户态拷贝
  |  ├── VNET header 支持 offload
  |
  |  slirp (用户模式)
  |  ├── 完整用户态 TCP/IP 栈
  |  ├── NAT + 协议处理开销
  v
```

---

## 27. 端到端 TX/RX 流程图

### 27.1 TAP 模式完整 TX 流程

```
Guest App                                                Host
  |                                                       |
  +-- write(sock_fd, data)                                |
  |                                                       |
  +-- Guest 内核网络栈                                     |
  |   构造以太网帧                                         |
  |                                                       |
  +-- virtio-net TX vring                                  |
  |   写入 desc → kick (MMIO/ioport)                       |
  |                                                       |
  +-- [KVM_EXIT / TCG 返回]                                |
  |                                                       |
  +-- virtio_net_handle_tx()                               |
  |   (virtio-net.c)                                      |
  |   读取 vring desc → 构造 iov                           |
  |                                                       |
  +-- qemu_sendv_packet_async()                            |
  |   (net.c:884)                                         |
  |                                                       |
  +-- filter_receive_iov() [TX filters]                    |
  |   (net.c:647)                                         |
  |                                                       |
  +-- qemu_net_queue_send_iov()                            |
  |   (queue.c:196)                                       |
  |                                                       |
  +-- qemu_deliver_packet_iov()                            |
  |   (net.c:829)                                         |
  |                                                       |
  +-- filter_receive_iov() [RX filters]                    |
  |   (net.c:647)                                         |
  |                                                       |
  +-- tap_receive_iov()                                    |
  |   (tap.c:152)                                         |
  |                                                       |
  +-- writev(tap_fd, iov)          -----+                  |
      (tap.c:138)                       |                  |
                                        v                  |
                                   TAP 设备 (/dev/net/tun) |
                                        |                  |
                                        v                  |
                                   Linux 网络栈 ---------->+
                                                      物理 NIC
```

### 27.2 vhost-net 模式完整 TX 流程

```
Guest App                                                Host
  |                                                       |
  +-- write(sock_fd, data)                                |
  |                                                       |
  +-- Guest 内核网络栈                                     |
  |   构造以太网帧                                         |
  |                                                       |
  +-- virtio-net TX vring                                  |
  |   写入 desc → kick (eventfd)                           |
  |                                                       |
  +-- [内核 vhost-net 线程被唤醒]                           |
  |   （不经过 QEMU 用户态！）                              |
  |                                                       |
  +-- vhost_net 内核模块                                    |
  |   读取 vring desc                                      |
  |   直接将数据写入 TAP fd                                 |
  |                                                       |
  +-- TAP 设备                                             |
  |                                                       |
  +-- Linux 网络栈 ---------------------------------------->+
                                                      物理 NIC
```

### 27.3 vhost-user + DPDK 模式

```
Guest App                                                Host
  |                                                       |
  +-- virtio-net TX vring                                  |
  |   写入 desc → kick (eventfd)                           |
  |                                                       |
  +-- [DPDK vhost-user backend 轮询]                       |
  |   （DPDK PMD 持续轮询 vring，无中断）                   |
  |                                                       |
  +-- DPDK 直接从共享内存读取帧                             |
  |   （mmap guest RAM，零拷贝）                            |
  |                                                       |
  +-- DPDK PMD 发送到物理 NIC --------------------------->+
      （绕过整个内核网络栈）                           物理 NIC
```

---


## 第七部分：virtio-net 设备仿真

## 28. virtio-net 概述

virtio-net 是 QEMU 中性能最高的虚拟网卡，支持多队列、TSO/GSO 卸载、RSS 以及 vhost 加速。它是生产环境中最常用的虚拟网络设备。

**QOM 类型层级**：

```
Object → DeviceState → VirtIODevice → VirtIONet

传输层:
  VirtIONetPCI  (virtio-net-pci)    — PCI 传输
  VirtIOMMIOProxy (virtio-net-device) — MMIO 传输
```

## 29. VirtIONet 结构体

定义于 `virtio-net.h:158-233`：

```c
struct VirtIONet {
    VirtIODevice parent_obj;

    /* 网络接口 */
    NICState *nic;                 // NIC 状态
    NICConf nic_conf;              // NIC 配置（MAC, model, peers）

    /* MAC 与链路 */
    uint8_t mac[ETH_ALEN];         // MAC 地址
    uint16_t status;               // 链路状态
    uint16_t mtu;                  // 最大传输单元

    /* 队列 */
    VirtIONetQueue *vqs;           // 队列对数组
    VirtQueue *ctrl_vq;            // 控制队列
    uint16_t max_queue_pairs;      // 最大队列对数
    uint16_t curr_queue_pairs;     // 当前活跃队列对数

    /* 接收过滤 */
    uint8_t promisc;               // 混杂模式
    uint8_t allmulti;               // 接收所有多播
    uint8_t alluni;                // 接收所有单播
    uint8_t nomulti;               // 不接收多播
    uint8_t nouni;                 // 不接收单播
    uint8_t nobcast;               // 不接收广播
    struct { ... } mac_table;      // MAC 过滤表
    uint32_t *vlans;               // VLAN 过滤位图

    /* 卸载特性 */
    uint32_t has_vnet_hdr;         // vnet header 支持
    uint32_t curr_guest_offloads;  // 当前 Guest 卸载特性
    int multiqueue;                // 多队列标志

    /* RSS */
    VirtioNetRssData rss_data;     // RSS 配置
    bool rss_data_loaded;
    struct NetRxPkt *rx_pkt;       // RSS 辅助结构

    /* vhost */
    VHostNetState *vhost_net;      // vhost 后端状态
};
```

#### VirtIONetQueue（virtio-net.h:147-157）

```c
struct VirtIONetQueue {
    VirtQueue *rx_vq;              // 接收 VirtQueue
    VirtQueue *tx_vq;              // 发送 VirtQueue
    QEMUTimer *tx_timer;           // TX 定时器（定时器模式）
    QEMUBH *tx_bh;                 // TX Bottom Half（BH 模式）
    uint32_t tx_waiting;           // TX 等待计数
    struct {
        VirtQueueElement *elem;    // 异步发送中的元素
    } async_tx;
    VirtIONet *n;                  // 反向指针
};
```

## 30. 设备 Realize 流程

`virtio_net_device_realize()`（virtio-net.c:3867-4049）：

```
virtio_net_device_realize(dev, errp)
  │
  ├── 1. 验证参数
  │     ├── MTU 范围检查
  │     ├── speed/duplex 验证
  │     └── failover 配置
  │
  ├── 2. 初始化 virtio
  │     └── virtio_init(vdev, VIRTIO_ID_NET, sizeof(virtio_net_config))
  │
  ├── 3. 分配队列
  │     ├── n->vqs = g_new0(VirtIONetQueue, max_queue_pairs)
  │     ├── virtio_net_add_queue(n, 0)   — 第一个队列对
  │     └── virtio_add_queue(ctrl_vq)    — 控制队列
  │
  ├── 4. 设置 MAC 地址
  │     ├── qemu_macaddr_default_if_unset(&n->nic_conf.macaddr)
  │     └── memcpy(n->mac, ...)
  │
  ├── 5. 创建 NIC
  │     └── n->nic = qemu_new_nic(&net_virtio_info, &n->nic_conf, ...)
  │           │                                    [3991-3998]
  │           └── net_virtio_info 回调:
  │                 ├── .can_receive = virtio_net_can_receive
  │                 ├── .receive     = virtio_net_receive
  │                 └── .link_status_changed = virtio_net_set_link_status
  │
  └── 6. 后端连接
        └── NIC 自动与 -netdev 指定的 peer 关联
```

## 31. RX 接收路径

外部网络数据包到达 Guest 的完整路径：

```
网络后端（TAP/用户/socket）
  │
  ▼
QEMU 网络层
  │
  ├── 1. 检查可接收性
  │     └── virtio_net_can_receive() [virtio-net.c:1628-1648]
  │           ├── VM 是否运行？
  │           ├── 驱动是否就绪（VIRTIO_CONFIG_S_DRIVER_OK）？
  │           └── 队列是否有可用缓冲区？
  │
  ├── 2. 可选 RSS 路由
  │     └── virtio_net_process_rss() [virtio-net.c:1848-1901]
  │           └── 根据 RSS 哈希将包路由到正确的队列对
  │
  ├── 3. 接收过滤
  │     └── receive_filter() [virtio-net.c:1737-1786]
  │           ├── 混杂模式？→ 直接通过
  │           ├── 广播？→ 检查 nobcast
  │           ├── 多播？→ 检查 allmulti / MAC 表
  │           └── 单播？→ 检查 alluni / MAC 表
  │
  └── 4. 复制到 Guest
        └── virtio_net_receive_rcu() [virtio-net.c:1904-2054]
              │
              ├── 构建 virtio-net header
              │     └── receive_header() [virtio-net.c:1715-1735]
              │           └── 填充 checksum/TSO 卸载提示
              │
              ├── 从 RX 队列弹出缓冲区
              │     └── virtqueue_pop(rx_vq) [1956]
              │
              ├── 复制数据到 Guest 内存
              │     ├── 非 mergeable：一个描述符放完整包
              │     └── mergeable：可跨多个描述符，更新 num_buffers
              │
              └── 完成通知
                    ├── virtqueue_fill()      — 填充 used ring
                    ├── virtqueue_flush()     — 更新 used idx
                    └── virtio_notify()       — 触发中断
```

#### RSC（Receive Segment Coalescing）

在标准 RX 路径之前，可选的 RSC 层合并属于同一 TCP 流的多个小包：

```
virtio_net_receive()
  └── virtio_net_rsc_receive4/6() [virtio-net.c:2502-2605]
        ├── 匹配 TCP 流（5-tuple）
        ├── 合并数据段到缓存包
        ├── 定时器触发刷新 → virtio_net_rsc_purge()
        └── 合并包一次性递送给 Guest
```

## 32. TX 发送路径

Guest 发送网络数据包的完整路径：

```
Guest 驱动写入 TX QueueNotify
  │
  ▼
virtio_net_handle_tx_timer() [virtio-net.c:2822-2849]  ← 定时器模式
virtio_net_handle_tx_bh()    [virtio-net.c:2851-2875]  ← BH 模式
  │
  └── 调度定时器或 BH，避免频繁处理
        │
        ▼
  virtio_net_tx_timer() [2877-2925]  或  virtio_net_tx_bh() [2927-2974]
        │
        └── virtio_net_flush_tx() [virtio-net.c:2718-2818]
              │
              ├── while (包数 < TX_BURST):
              │     │
              │     ├── virtqueue_pop(tx_vq) [2740]
              │     │     └── 取出一个包的描述符链
              │     │
              │     ├── 验证头长度
              │     │     └── out_sg[0] >= sizeof(virtio_net_hdr)
              │     │
              │     ├── 处理 vnet header
              │     │     └── virtio_net_hdr_swap() — 端序转换
              │     │
              │     ├── 发送到网络后端
              │     │     └── qemu_sendv_packet_async(nc, out_sg, out_num,
              │     │               virtio_net_tx_complete) [2795-2801]
              │     │
              │     ├── 如果异步发送未完成:
              │     │     ├── 保存 async_tx.elem
              │     │     └── 禁用通知，退出循环
              │     │
              │     └── 如果同步完成:
              │           ├── virtqueue_push(tx_vq, elem, 0)
              │           └── virtio_notify(vdev, tx_vq)
              │
              └── 如果达到 TX_BURST 限制:
                    └── 重新调度定时器/BH 继续发送
```

#### TX 定时器 vs BH 模式

| 模式 | 触发方式 | 延迟 | 吞吐量 | 代码位置 |
|------|---------|------|--------|---------|
| Timer | 延迟合并 | 较高 | 较高 | 2877-2925 |
| BH | 立即调度 | 较低 | 较低 | 2927-2974 |

默认使用 BH 模式，可通过属性 `tx=timer` 切换。

#### 异步发送完成

```
网络后端发送完成
  │
  ▼
virtio_net_tx_complete() [virtio-net.c:2685-2715]
  ├── virtqueue_push(tx_vq, async_tx.elem, 0)
  ├── virtio_notify(vdev, tx_vq)
  ├── async_tx.elem = NULL
  └── 重新进入 flush_tx() 处理更多包
```

## 33. 控制队列

virtio-net 的控制队列（ctrl_vq）用于配置网络行为：

```
virtio_net_handle_ctrl() [virtio-net.c:1593-1616]
  └── virtio_net_handle_ctrl_iov() [1550-1591]
        │
        ├── 解析控制命令头: class + command
        │
        └── 分发到子处理器:
              │
              ├── VIRTIO_NET_CTRL_RX
              │     └── virtio_net_handle_rx_mode() [1005-1036]
              │           ├── PROMISC: 设置混杂模式
              │           ├── ALLMULTI: 接收所有多播
              │           ├── ALLUNI: 接收所有单播
              │           ├── NOMULTI: 不接收多播
              │           ├── NOUNI: 不接收单播
              │           └── NOBCAST: 不接收广播
              │
              ├── VIRTIO_NET_CTRL_MAC
              │     └── virtio_net_handle_mac() [1083-1177]
              │           ├── ADDR_SET: 设置 MAC 地址
              │           └── TABLE_SET: 批量设置 MAC 过滤表
              │
              ├── VIRTIO_NET_CTRL_VLAN
              │     └── virtio_net_handle_vlan_table() [1179-1206]
              │           ├── ADD: 添加 VLAN ID 到位图
              │           └── DEL: 从位图删除 VLAN ID
              │
              ├── VIRTIO_NET_CTRL_ANNOUNCE
              │     └── virtio_net_handle_announce() [1208-1222]
              │           └── 触发 GARP 通告（迁移后）
              │
              ├── VIRTIO_NET_CTRL_MQ
              │     └── virtio_net_handle_mq() [1497-1548]
              │           └── 设置活跃队列对数
              │
              ├── VIRTIO_NET_CTRL_GUEST_OFFLOADS
              │     └── virtio_net_handle_offloads() [1038-1081]
              │           └── 动态调整 Guest 卸载特性
              │
              └── VIRTIO_NET_CTRL_RSS
                    └── virtio_net_handle_rss() [1377-1495]
                          └── 配置 RSS 哈希/indirection 表
```

## 34. 多队列与 RSS

#### 多队列架构

```
Guest:
  vCPU 0 ──► TX Queue 0 ──►┐
  vCPU 0 ◄── RX Queue 0 ◄──┤── Queue Pair 0
                            │
  vCPU 1 ──► TX Queue 1 ──►┐
  vCPU 1 ◄── RX Queue 1 ◄──┤── Queue Pair 1
                            │
  ...                       │
                            │
  所有 vCPU ──► Ctrl Queue  ── 控制命令
```

**队列编号规则**（virtio-net.c:141-144）：

```c
#define vq2q(queue_index) ((queue_index) / 2)
// RX 队列: 偶数索引 (0, 2, 4, ...)
// TX 队列: 奇数索引 (1, 3, 5, ...)
// Ctrl 队列: 最后一个
```

**队列对管理**：
- 启动时创建 `max_queue_pairs` 个队列对
- Guest 通过 ctrl_vq 发送 `VIRTIO_NET_CTRL_MQ` 设置 `curr_queue_pairs`
- `virtio_net_set_multiqueue()`（3056-3064）动态增减队列

#### RSS（接收端缩放）

RSS 允许根据数据包的流哈希将 RX 包分发到不同的队列：

```
收到数据包
  │
  ▼
virtio_net_process_rss() [virtio-net.c:1848-1901]
  │
  ├── 计算流哈希（基于 IP + 端口等）
  ├── 查 indirection table → 目标队列索引
  └── 将包路由到对应 RX 队列
```

**eBPF RSS 加速**：
- `virtio_net_attach_ebpf_rss()`（1245-1266）将 RSS 逻辑卸载到 eBPF 程序
- eBPF 在 TAP fd 上直接执行哈希和分发，避免进入 QEMU 用户空间

## 35. 特性协商

`virtio_net_get_features()`（virtio-net.c:3073-3180）：

| 特性 | 条件 | 说明 |
|------|------|------|
| `VIRTIO_NET_F_MAC` | 总是 | MAC 地址报告 |
| `VIRTIO_NET_F_STATUS` | 总是 | 链路状态通知 |
| `VIRTIO_NET_F_MRG_RXBUF` | 总是 | 可合并 RX 缓冲区 |
| `VIRTIO_NET_F_HOST_TSO4/6` | peer 支持 vnet_hdr | 主机端 TSO 卸载 |
| `VIRTIO_NET_F_GUEST_CSUM` | peer 支持 vnet_hdr | Guest 校验和卸载 |
| `VIRTIO_NET_F_GUEST_TSO4/6` | peer 支持 vnet_hdr | Guest TSO 卸载 |
| `VIRTIO_NET_F_MQ` | max_queue_pairs > 1 | 多队列 |
| `VIRTIO_NET_F_RSS` | peer 支持 | 接收端缩放 |
| `VIRTIO_NET_F_CTRL_VQ` | 总是 | 控制队列 |
| `VIRTIO_NET_F_CTRL_RX` | 总是 | RX 模式控制 |
| `VIRTIO_NET_F_CTRL_VLAN` | 总是 | VLAN 过滤 |
| `VIRTIO_NET_F_CTRL_MAC_ADDR` | 总是 | MAC 地址设置 |

**vnet_hdr 的作用**：vnet_hdr 是在包数据前附加的 virtio-net 头，包含 TSO/GSO/checksum 等卸载提示。只有当网络后端（如 TAP）支持 vnet_hdr 时，才能启用这些卸载特性。

**git 趋势**：
- commit 1c79ab6937：默认启用 UDP tunnel GSO 支持
- commit 3a7741c3bd：实现扩展特性支持（>64 位特性）

## 36. 网络后端与 vhost 集成

#### 网络后端类型

```
virtio-net (VirtIODevice)
  │
  └── NICState → NetClientState
        │
        └── peer (NetClientState) ← 网络后端
              │
              ├── TAP 后端
              │     └── TAP fd → 主机内核网桥/路由
              │
              ├── User 后端（SLIRP）
              │     └── 用户空间 TCP/IP 栈（NAT 模式）
              │
              ├── Socket 后端
              │     └── UDP/TCP socket → 另一个 QEMU 实例
              │
              └── vhost-net 后端
                    └── 内核 vhost → 直接 TAP I/O
```

#### vhost-net 加速

```
标准路径:
  Guest VirtQueue ←→ QEMU 用户空间 ←→ TAP fd
  （每个包两次用户/内核切换）

vhost-net 路径:
  Guest VirtQueue ←→ vhost-net 内核模块 ←→ TAP fd
  （零用户空间拷贝，内核直接处理）
```

**vhost 启动/停止**（virtio-net.c:282-340）：

```c
virtio_net_vhost_status(n, status)
  ├── 条件满足（DRIVER_OK + vhost 可用）
  │     └── vhost_net_start(vdev, n->nic->ncs, queues)
  │           ├── 编程 vring 地址到内核
  │           ├── 传递 kick/call eventfd
  │           └── 数据面转移到内核
  │
  └── 条件不满足
        └── vhost_net_stop() → 数据面回到 QEMU
```

## 37. CLI 使用与配置

#### 基本用法

```bash
# TAP 后端（最常用，高性能）
qemu-system-aarch64 -M virt \
  -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
  -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:56

# 用户网络（无需特权，NAT 模式）
qemu-system-aarch64 -M virt \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0

# MMIO 传输
qemu-system-aarch64 -M virt \
  -netdev user,id=net0 \
  -device virtio-net-device,netdev=net0
```

#### 高级配置

```bash
# 多队列 + vhost
qemu-system-aarch64 -M virt \
  -netdev tap,id=net0,queues=4,vhost=on \
  -device virtio-net-pci,netdev=net0,mq=on,vectors=10

# 自定义 MTU
-device virtio-net-pci,netdev=net0,host_mtu=9000

# 关闭 TX 定时器（使用 BH 模式）
-device virtio-net-pci,netdev=net0,tx=bh
```

#### 连接关系

```
-netdev tap,id=net0,...    → 创建 TAP NetClientState
-device virtio-net-pci,netdev=net0  → 创建 NIC，peer = net0
                                       realize 时: qemu_new_nic() 连接
```


---

## 附录 A. 关键数据结构速查表

| 数据结构 | 定义位置 | 用途 |
|----------|----------|------|
| `NetClientState` | `net.h:86-134` | 所有网络端点的基类 |
| `NetClientInfo` | `net.h:86-134` | 回调函数表 |
| `NICState` | `net.h:138-144` | NIC 前端（含多个 NetClientState） |
| `NICConf` | `net.h:32-36` | NIC 配置（MAC、peers） |
| `NetQueue` | `queue.c:52-61` | 数据包排队与流控 |
| `NetFilterState` | `filter.h:51-63` | 数据包过滤器实例 |
| `NetFilterClass` | `filter.h:38-48` | 过滤器类接口 |
| `NetHub` / `NetHubPort` | `hub.c:32-45` | 虚拟交换 |
| `TAPState` | `tap.c:70-86` | TAP 后端状态 |
| `VHostNetState` | `vhost.h:143-152` | vhost-net 加速器状态 |
| `struct vhost_dev` | `vhost.h:75-137` | vhost 核心设备 |
| `VhostUserState` | `vhost-user.c:66-74` (net/) | vhost-user 网络状态 |
| `VhostUserMsg` | `vhost-user.c:57-103` (hw/) | vhost-user 协议消息 |
| `VhostVDPAState` | `vhost-vdpa.c:34-50` | vDPA 后端状态 |
| `SlirpState` | `slirp.c:89-99` | slirp 用户模式状态 |
| `NetSocketState` | `socket.c:36-46` | Socket 后端状态 |

---

## 附录 B. 源码文件索引

| 分类 | 文件 | 行数 | 说明 |
|------|------|------|------|
| **核心框架** | `net/net.c` | 2,175 | 网络后端核心 |
| | `net/queue.c` | 293 | 数据包队列 |
| | `net/hub.c` | 331 | Hub 虚拟交换 |
| | `net/filter.c` | 377 | 过滤器框架 |
| | `include/net/net.h` | — | 核心头文件 |
| | `include/net/filter.h` | — | 过滤器头文件 |
| **TAP** | `net/tap.c` | 1,066 | TAP 后端 |
| | `net/tap-linux.c` | 360 | TAP Linux ioctl |
| | `net/tap_int.h` | — | TAP 内部头文件 |
| **vhost** | `hw/virtio/vhost.c` | 2,554 | vhost 核心 |
| | `hw/net/vhost_net.c` | 697 | vhost-net 加速 |
| | `include/hw/virtio/vhost.h` | — | vhost 头文件 |
| | `hw/virtio/vhost-shadow-virtqueue.c` | 782 | SVQ |
| **vhost-user** | `hw/virtio/vhost-user.c` | 3,066 | 协议实现 |
| | `net/vhost-user.c` | 480 | 网络后端 |
| | `include/hw/virtio/vhost-user.h` | — | 协议特性定义 |
| **vDPA** | `net/vhost-vdpa.c` | 1,898 | vDPA 后端 |
| **其他** | `net/slirp.c` | 1,310 | 用户模式网络 |
| | `net/socket.c` | 786 | Socket 后端 |
| | `net/filter-mirror.c` | 526 | mirror/redirector |
| | `net/filter-buffer.c` | 207 | 缓冲过滤器 |
| | `net/filter-rewriter.c` | — | COLO TCP 重写 |
| | `hw/net/virtio-net.c` | ~4,100 | virtio-net NIC |

---

## 附录 C. 关联文档索引

| 文档 | 路径 | 关联内容 |
|------|------|----------|
| 设备模型与 virtio 深度分析 | `darren/device-model/00-设备模型与virtio深度分析.md` | VirtIO 框架、VirtQueue 机制 |
| 关键设备仿真分析 | `darren/device-model/01-关键设备仿真分析-UART-磁盘-网卡.md` | virtio-net 前端详细分析 |
| VFIO 设备直通 | `darren/device-model/05-VFIO设备直通与IOMMU集成深度分析.md` | IOMMU/DMA 映射 |
| 模拟执行循环 | `darren/architecture/02-模拟执行循环与MMIO分发深度分析.md` | 事件循环与 fd handler |
| Machine 建立流程 | `darren/architecture/03-Machine建立流程深度分析.md` | 网络设备创建时序 |
| 内存子系统 | `darren/memory/00-内存子系统深度分析.md` | MemoryListener 机制 |

---

> 文档生成时间：基于 QEMU 11.0.50 源码分析  
> 索引工具：zoekt + ctags + cscope（`.ai-search/`）
