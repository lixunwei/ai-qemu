# 设备模型子系统综合导航

> 目标：把 `device-model/` 目录下 8 篇专题文档串成一张“设备模型导航图”，帮助读者从 **QOM/设备对象**、**VirtIO**、**Block/Chardev/Network**、**VFIO/IOMMU**、**DMA**、**Timer** 六条主线理解 QEMU 设备模型子系统。  
> 阅读对象：正在做 QEMU 源码分析、希望快速定位入口文件、核心数据结构与端到端数据流的读者。  
> 目录说明：当前目录包含 `00, 02, 03, 04, 05, 06, 07, 08` 八篇文档，**没有 01**，本导航按实际文档组织。

---

## 1. 设备模型全景图

QEMU 设备模型可以粗略理解为：**QOM 提供对象体系，Device/Bus 提供设备组织方式，前端设备负责向 Guest 暴露寄存器/virtqueue/配置空间，后端子系统负责把请求送到宿主机资源**。

```text
+--------------------------------------------------------------------------------+
|                                QEMU Object Model                               |
|  Object  -->  DeviceState  -->  SysBusDevice / PCIDevice / VirtIODevice        |
+-------------------------------+------------------------------------------------+
                                |
                                v
+--------------------------------------------------------------------------------+
|                                   Bus Layer                                    |
|  SystemBus / PCI Bus / Virtio transport (virtio-mmio, virtio-pci)              |
+-------------------------------+------------------------------------------------+
                                |
                                v
+--------------------------------------------------------------------------------+
|                              Frontend Device Layer                             |
|  PL011 / VirtIOBlock / VirtIONet / VFIOPCIDevice / DMA ctl / ARM Timer         |
+-------------------------------+------------------------------------------------+
                                |
                +---------------+-------------------+----------------+
                |                                   |                |
                v                                   v                v
      +-------------------+              +----------------+   +----------------+
      |   Block Backend   |              | Char Backend   |   | Network Backend|
      | BB -> BDS -> drv  |              | stdio/socket   |   | TAP/vhost/...  |
      +-------------------+              +----------------+   +----------------+
                |                                   |                |
                +-------------------+---------------+----------------+
                                    |
                                    v
+--------------------------------------------------------------------------------+
|                               Host Kernel / Userspace                          |
|  file-posix / io_uring / eventfd / tap fd / vhost kernel / vhost-user daemon   |
+--------------------------------------------------------------------------------+
```

如果从源码工作流看，可以进一步记成四层：

1. **对象层**：QOM 类型注册、对象实例化、属性与 realize。
2. **总线层**：SysBus / PCI / virtio transport 负责挂载与发现。
3. **设备层**：具体设备实现寄存器访问、中断、DMA、virtqueue 处理。
4. **后端层**：Block / Chardev / Net / VFIO / Timer / DMA helper 与宿主资源交互。

---

## 2. 文档地图

| 编号 | 文档 | 主题定位 | 适合何时阅读 | 入口链接 |
|---|---|---|---|---|
| 00 | 设备模型与 virtio 深度分析 | QOM、Device、Bus、virtio 核心、ARM virt 拓扑 | 作为总入口时 | [00-设备模型与virtio深度分析.md](00-设备模型与virtio深度分析.md) |
| 02 | 块层 IO 路径深度分析 | BlockBackend、BDS 图、协程 AIO、io_uring | 关注块请求如何落盘时 | [02-块层IO路径深度分析.md](02-块层IO路径深度分析.md) |
| 03 | Chardev 子系统与 UART 交互深度分析 | chardev 前后端、PL011、串口收发链路 | 分析串口/控制台时 | [03-Chardev子系统与UART交互深度分析.md](03-Chardev子系统与UART交互深度分析.md) |
| 04 | Block 设备子系统深度分析 | Block 生命周期、qcow2、BlockJob、virtio-blk | 看块设备“控制面+数据面”时 | [04-Block设备子系统深度分析.md](04-Block设备子系统深度分析.md) |
| 05 | VFIO 设备直通与 IOMMU 集成深度分析 | VFIO 容器、中断、DMA、IOMMUFD、迁移 | 看直通设备与 IOMMU 交互时 | [05-VFIO设备直通与IOMMU集成深度分析.md](05-VFIO设备直通与IOMMU集成深度分析.md) |
| 06 | 网络后端深度分析 | TAP、vhost-net、vhost-user、slirp、virtio-net | 看网络后端与加速路径时 | [06-网络后端深度分析-TAP-vhost-net-vhost-user.md](06-网络后端深度分析-TAP-vhost-net-vhost-user.md) |
| 07 | DMA 设备模拟架构深度分析 | DMA API、SGList、PCI/SysBus DMA、IOMMU | 分析设备主存访问时 | [07-DMA设备模拟架构深度分析.md](07-DMA设备模拟架构深度分析.md) |
| 08 | ARM Generic Timer 深度分析 | 系统计数器、7 类定时器、EL 控制、KVM 集成 | 分析 CPU 内嵌定时器时 | [08-ARM-Generic-Timer深度分析-计数器-7类定时器-EL访问控制-VHE重定向与KVM集成.md](08-ARM-Generic-Timer深度分析-计数器-7类定时器-EL访问控制-VHE重定向与KVM集成.md) |

**建议的总体顺序**：`00 → 03/04/06 → 02/07 → 05 → 08`。  
其中：`00` 是对象模型与 virtio 骨架；`03/04/06` 是三大典型设备面；`02/07/05` 是数据面和地址翻译深水区；`08` 则补齐 ARM 平台上的时间模型。

---

## 3. 00《设备模型与 virtio 深度分析》：总入口文档

- **核心价值**：建立设备模型的第一性框架，回答“QEMU 中设备是什么、挂在哪、何时创建、如何通过 virtio 与 Guest 通信”。
- **关键概念**：
  - QOM 类型层次：`Object -> DeviceState -> SysBusDevice/PCIDevice/VirtIODevice`
  - 总线系统：SystemBus、PCI/PCIe、virtio-mmio、virtio-pci
  - 生命周期：类型注册、instance_init、realize、unrealize、reset
  - virtio 核心：`VirtIODevice`、`VirtQueue`、特性协商、通知机制
  - ARM virt 拓扑：平台设备、GIC、PCIe 根桥、virtio 集成方式
- **阅读收益**：读完后能看懂后续文档为什么都从 `DeviceState`、总线挂载、回调表和 realize 路径切入。
- **建议重点**：
  1. 先看设备基类层级与 `DeviceClass`；
  2. 再看 virtio 核心结构与 `VirtQueue`；
  3. 最后看 ARM virt 机器拓扑，把抽象对象映射到机器板级布局。
- **后续跳转**：
  - 想看 virtio 块设备：跳 [04-Block设备子系统深度分析.md](04-Block设备子系统深度分析.md)
  - 想看 virtio 网络：跳 [06-网络后端深度分析-TAP-vhost-net-vhost-user.md](06-网络后端深度分析-TAP-vhost-net-vhost-user.md)
  - 想看设备主存访问：跳 [07-DMA设备模拟架构深度分析.md](07-DMA设备模拟架构深度分析.md)

---

## 4. 02《块层 IO 路径深度分析》：从请求到宿主文件系统

- **核心价值**：解释块请求如何穿过 `BlockBackend`、`BDS graph`、格式驱动与协议驱动，最终到达宿主机内核。
- **关键概念**：
  - `BlockBackend`：设备层与块层之间的接口面
  - `BlockDriverState`：块节点，构成有向图而不是简单栈
  - `BdrvChild`：父子节点边，连接过滤器、格式层、协议层
  - 协程化 I/O：`blk_aio_*`、`bdrv_co_*`、AioContext、线程池、io_uring
  - 多种路径：read / write / flush / discard / write-zeroes
- **阅读收益**：能把 `virtio-blk` 的请求处理和具体 `qcow2/file-posix` 落盘过程连起来。
- **建议重点**：
  1. 先掌握 Block 三大结构：BB、BDS、BlockDriver；
  2. 再看读/写路径图；
  3. 最后补协程与异步 I/O 基础设施。
- **和 04 的分工**：
  - `04` 更偏“设备和生命周期”；
  - `02` 更偏“数据路径和异步执行”。
- **推荐搭配**：与 [04-Block设备子系统深度分析.md](04-Block设备子系统深度分析.md) 连读，效果最佳。

---

## 5. 03《Chardev 子系统与 UART 交互深度分析》：字符设备前后端范式

- **核心价值**：这是理解“前端设备 + 后端抽象”模式最直观的一篇，适合用来建立对 QEMU I/O 解耦方式的感觉。
- **关键概念**：
  - Chardev 前后端分离：设备前端（PL011、16550、virtio-console）与后端（stdio/socket/pty/file）
  - 生命周期：创建、打开、关闭、销毁、回调安装
  - 前后端 API：写路径、读路径、事件通知、流控
  - PL011 UART：寄存器、FIFO、中断、与 chardev 的连接点
  - 端到端链路：Guest `printf` 到宿主终端显示；宿主输入到 Guest 读取
- **阅读收益**：能理解 QEMU 如何把“设备模拟”和“宿主 I/O”解耦，并为后续理解 Net/Block 后端建立模式识别。
- **建议重点**：
  1. 先看 Chardev 架构与 QOM 类型层次；
  2. 再看 `PL011 ↔ chardev` 数据路径；
  3. 最后看完整 TX/RX 时序图。
- **适合作为入门例子**：如果你对块层或网络层太重，可以先用串口路径建立“设备前端/后端”心智模型，再回看 Block/Net。

---

## 6. 04《Block 设备子系统深度分析》：块设备控制面总览

- **核心价值**：从设备对象、命令行、后端创建、格式驱动到 block job，把块设备体系完整串起来。
- **关键概念**：
  - `-drive` / `-blockdev` 命令行解析
  - `BlockBackend` 生命周期与权限模型
  - BDS 图与节点关系
  - `qcow2` 布局、L1/L2、refcount、snapshot
  - Block 作业：mirror / commit / stream / backup
  - `virtio-blk`：结构体、realize、请求处理、多队列、IOThread
- **阅读收益**：对“一个虚拟磁盘在 QEMU 内部是怎么搭出来的”建立完整认识。
- **建议重点**：
  1. 先看第二部分“创建与生命周期”；
  2. 再看第三部分 `qcow2`；
  3. 如果关心 live storage 操作，再看第四部分 block job；
  4. 最后回到 `virtio-blk` 连接设备入口与块后端。
- **与 02 的衔接点**：本篇的 `virtio-blk` 请求管线读完后，立即跳转到 [02-块层IO路径深度分析.md](02-块层IO路径深度分析.md) 看请求如何继续下沉。

---

## 7. 05《VFIO 设备直通与 IOMMU 集成深度分析》：最接近真实硬件的一条线

- **核心价值**：解释 QEMU 如何不再模拟完整设备数据面，而是把真实 PCI 设备通过 VFIO 暴露给 Guest，同时仍保留配置、DMA、迁移、热插拔等控制逻辑。
- **关键概念**：
  - VFIO 容器模型：legacy container 与 IOMMUFD
  - `VFIOPCIDevice` 的 QOM/PCI 层次
  - Config Space、BAR、ROM、INTx、MSI/MSI-X
  - irqfd、eventfd、内核协作
  - DMA 映射、MemoryListener、脏页跟踪
  - QEMU IOMMU 抽象、SMMUv3、virtio-iommu
  - 热插拔与迁移支持
- **阅读收益**：能分清 **纯模拟设备**、**半模拟半直通**、**完全内核/硬件数据面** 三种模式的边界。
- **建议重点**：
  1. 先看“容器模型”与“设备初始化流程”；
  2. 再看中断与 DMA 映射；
  3. 最后看 IOMMUFD、热插拔、迁移等高级主题。
- **阅读前置**：建议先有 [07-DMA设备模拟架构深度分析.md](07-DMA设备模拟架构深度分析.md) 和 `PCI/IOMMU` 背景，再读 VFIO 会更顺。

---

## 8. 06《网络后端深度分析》：数据面加速路径全集

- **核心价值**：这篇是网络方向的总索引，把 `NetClientState`、TAP、vhost-net、vhost-user、vDPA、slirp、`virtio-net` 一次讲全。
- **关键概念**：
  - QEMU 网络核心框架：`NetClientState`、peer 链接、NetQueue、过滤器
  - TAP：用户态 fd 驱动的经典路径
  - vhost-net：把 virtqueue 数据面搬到内核
  - vhost-user：把数据面外包给用户态 daemon
  - vhost-vDPA：把数据面进一步下沉到硬件/内核框架
  - `virtio-net`：前端设备模型、多队列、控制队列、RSS
- **阅读收益**：能快速回答“同样是 virtio-net，为何有 TAP/vhost-net/vhost-user 三套性能与隔离模型”。
- **建议重点**：
  1. 先看网络核心框架与 `NetClientState`；
  2. 再集中比较 TAP / vhost-net / vhost-user；
  3. 最后看 `virtio-net` 如何挂接这些后端。
- **与 00 的衔接点**：`00` 提供 virtio transport 和 VirtQueue 框架，`06` 负责把这些队列请求连接到不同网络后端。

---

## 9. 07《DMA 设备模拟架构深度分析》：设备访问主存的中枢文档

- **核心价值**：DMA 是设备模型的“暗线”。很多数据路径看似不同，最终都会回到 `AddressSpace`、IOMMU、SGList、memory API。该文档正是这条暗线的汇总。
- **关键概念**：
  - DMA 核心 API：`dma_memory_read/write`、`dma_memory_map/unmap`
  - `QEMUSGList`：散列聚合列表
  - `dma_blk_io`：块设备 DMA 帮助函数
  - `AddressSpace` 与 `MemoryRegion` 的联系
  - Bounce Buffer、IOMMU 翻译、Bus Master Enable
  - PCI 设备 DMA 与 SysBus 设备 DMA 差异
  - 设备实例：e1000、AHCI、virtio-blk、PL330、PL080
- **阅读收益**：能把“设备读写 Guest RAM”这件事从概念层面落到 API、地址翻译与案例代码上。
- **建议重点**：
  1. 先看 DMA 总体架构与核心 API；
  2. 再看 IOMMU 与 AddressSpace；
  3. 最后选一个设备案例顺着跟代码。
- **与 05 的衔接点**：`07` 讲模拟设备 DMA，`05` 讲直通设备 DMA；两者共享 `AddressSpace/IOMMU/MemoryListener` 这套底层抽象。

---

## 10. 08《ARM Generic Timer 深度分析》：CPU 内嵌设备模型的典型案例

- **核心价值**：虽然 Generic Timer 不像网卡/磁盘那样表现为独立外设，但它是 ARM 平台中极关键的“内嵌设备模型”：有寄存器视图、有中断、有宿主时间源绑定、有 KVM 同步。
- **关键概念**：
  - System Counter、物理/虚拟计数器
  - 7 类定时器变体
  - `CTL/CVAL/TVAL` 三组寄存器语义
  - EL0~EL3 访问控制、VHE 重定向
  - `QEMUTimer` 基础设施与 PPI 中断接线
  - `virt` 机器中的 DTB 生成与 KVM 状态同步
- **阅读收益**：能理解“CPU 对象内部也可以承载设备功能”，从而扩展对设备模型的认知边界。
- **建议重点**：
  1. 先看架构概述和定时器变体；
  2. 再看寄存器访问控制和重算/中断逻辑；
  3. 最后看 `virt` 机器接线与 KVM 集成。
- **适合与谁联读**：
  - 与 `../arm/13-ARM-Generic-Timer深度分析.md` 对照架构语义；
  - 与 `../architecture/16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md` 对照通用定时器框架。

---

## 11. 设备 I/O 数据流：端到端统一视角

不同设备子系统形式不同，但都可以被归纳到一个统一模型中：**Guest driver 发起 I/O，QEMU 前端设备接收，转换为内部请求，再交给后端或宿主资源完成，最后通过中断/状态返回 Guest**。

```text
Guest Driver
   |
   | MMIO / PIO / PCI config / virtqueue kick
   v
Frontend Device in QEMU
   |
   | decode register / pop VirtQueue / build request
   v
Subsystem Backend
   |
   | Block / Char / Net / VFIO / DMA / Timer infra
   v
Host Resource
   |
   | file fd / socket fd / tap fd / eventfd / kernel ioctl / host clock
   v
Completion Path
   |
   | callback / bh / coroutine wakeup / irq injection
   v
Guest interrupt + driver consume completion
```

### 11.1 几类典型映射

| 设备类型 | Guest 入口 | QEMU 前端 | 后端/基础设施 | 宿主资源 |
|---|---|---|---|---|
| virtio-blk | virtqueue | `VirtIOBlock` | BlockBackend / BDS | 文件、块设备、io_uring |
| PL011 UART | MMIO 寄存器 | `PL011State` | Chardev | stdio、pty、socket |
| virtio-net | virtqueue | `VirtIONet` | NetClientState / vhost | TAP、内核 vhost、用户态 daemon |
| VFIO PCI | PCI config/BAR | `VFIOPCIDevice` | VFIO / IOMMUFD | 真实 PCI 设备、内核 VFIO |
| DMA 控制器 | MMIO + 描述符 | DMA engine model | DMA API / AddressSpace | Guest RAM、IOMMU |
| ARM Timer | sysreg | `ARMCPU` 内嵌逻辑 | QEMUTimer / host clock | 宿主时间源 |

### 11.2 统一分析模板

阅读任何设备模型时，建议都按下面五个问题追踪：

1. **对象是谁**：对应哪个 `DeviceState`/CPU 内嵌结构？
2. **挂在哪里**：SysBus、PCI、virtio-mmio 还是 virtio-pci？
3. **请求从哪里来**：寄存器访问、描述符队列还是事件回调？
4. **后端是什么**：Block、Char、Net、VFIO、DMA API 还是定时器框架？
5. **完成如何返回**：设置状态位、唤醒协程、发 IRQ、注入 MSI/PPI？

---

## 12. Block I/O 完整路径：virtio-blk → BlockBackend → BDS → qcow2 → file-posix

块路径是本目录最典型、也最值得反复看的主线之一。

```text
Guest block driver
   |
   | submit request to virtqueue
   v
virtio-blk frontend
   |
   | parse vring desc / build VirtIOBlockReq
   v
BlockBackend (blk_aio_preadv / pwritev)
   |
   v
BDS graph
   |
   +--> filter node (throttle / snapshot / quorum ...)
   |
   +--> format node (qcow2)
   |
   `--> protocol node (file-posix / host_device)
   v
host kernel I/O
   |
   v
completion callback / coroutine wakeup
   |
   v
virtio-blk status byte + interrupt
   |
   v
Guest driver completes bio/request
```

### 12.1 建议结合文档的切分方式

- **设备入口**：看 [04-Block设备子系统深度分析.md](04-Block设备子系统深度分析.md) 的 `virtio-blk` 章节。
- **块层下沉**：看 [02-块层IO路径深度分析.md](02-块层IO路径深度分析.md) 的读/写路径与协程章节。
- **DMA 视角补充**：如果你关心数据缓冲区如何从 Guest RAM 映射出来，再看 [07-DMA设备模拟架构深度分析.md](07-DMA设备模拟架构深度分析.md)。

### 12.2 关键观察点

1. `virtio-blk` 负责“协议解包”和完成通知，不直接管理复杂存储格式。
2. `BlockBackend` 是设备与块层之间最稳定的接口。
3. 真正的存储栈是 **BDS 图**，不是单一对象。
4. `qcow2` 负责格式语义，`file-posix` 负责落到宿主 fd。
5. 协程/AioContext 决定请求如何异步推进，而不是阻塞在设备线程里。

---

## 13. 网络数据路径对比：TAP vs vhost-net vs vhost-user

网络后端最容易混淆的地方不在 `virtio-net` 前端，而在于**数据面究竟在哪里跑**。

| 路径 | 数据面位置 | 典型优点 | 典型代价 | 适合场景 |
|---|---|---|---|---|
| TAP | QEMU 用户态 + tap fd | 简单直观，调试友好 | 上下文切换更多，吞吐较低 | 开发、教学、基础功能验证 |
| vhost-net | Linux 内核 | 性能好，减少 QEMU 参与数据面 | 依赖内核实现，灵活性受限 | 通用高性能 virtio-net |
| vhost-user | 外部用户态进程 | 易与 OVS/DPDK/SPDK 等集成，隔离清晰 | 部署更复杂，需额外 daemon | 用户态高性能数据面、NFV |

### 13.1 三条路径的本质区别

```text
TAP:
Guest -> virtio-net -> QEMU NetClientState -> tap fd -> host net stack

vhost-net:
Guest -> virtqueue -> kernel vhost-net thread -> tap/socket -> host net stack
              ^ QEMU mainly sets up vring, eventfd, memory table

vhost-user:
Guest -> virtqueue -> external userspace backend (via Unix socket protocol)
              ^ QEMU mainly acts as control-plane coordinator
```

### 13.2 阅读建议

- 如果你关心**最基础的网络收发**：先看 TAP。
- 如果你关心**为什么性能提升**：重点看 vhost-net 的内核接管点。
- 如果你关心**QEMU 与外部数据面解耦**：重点看 vhost-user 协议、共享内存与重连机制。
- 如果你关心**virtio-net 本身**：最后回到 `virtio-net` 章节，看队列与后端如何拼起来。

对应主文档： [06-网络后端深度分析-TAP-vhost-net-vhost-user.md](06-网络后端深度分析-TAP-vhost-net-vhost-user.md)

---

## 14. 阅读路线推荐

### 路线 A：设备模型总览路线（第一次读）

`00 → 03 → 04 → 06`

- 目标：先建立对象模型、总线、前后端解耦与 virtio 的整体感觉。
- 特点：概念最平衡，适合从零到一建立全局图。

### 路线 B：块设备/存储路线

`00 → 04 → 02 → 07 → 05`

- 目标：从 `virtio-blk` 到块层，再到 DMA 与 IOMMU，把存储 I/O 路径吃透。
- 特点：适合研究磁盘性能、qcow2、异步 I/O、VFIO 存储直通等问题。

### 路线 C：网络/高性能 I/O 路线

`00 → 06 → 07 → 05`

- 目标：先看 virtio-net 和网络后端，再补 DMA/IOMMU/VFIO 的性能与地址翻译背景。
- 特点：适合做 virtio-net、vhost、vDPA、SR-IOV、直通网卡相关分析。

### 路线 D：ARM virt 平台路线

`00 → 03 → 08 → 07 → 05`

- 目标：以 ARM virt 机器为主线，串起 UART、Timer、DMA、SMMUv3/VFIO。
- 特点：适合聚焦 AArch64 平台设备、CPU 定时器与 ARM IOMMU 集成。

### 一条经验法则

- **先总览，再案例，再深水区。**
- `00` 是总览；`03/04/06` 是案例；`02/05/07/08` 是深水区。
- 如果中途迷路，优先回到 `00` 和本导航，而不是在某个专题里硬钻。

---

## 15. 与 `architecture/`、`arm/` 目录的交叉引用

本目录聚焦“设备模型子系统”，但很多概念在其他目录有更底层或更架构化的解释，建议交叉阅读。

### 15.1 与 `architecture/` 的推荐联读

- QOM / 类型系统基础：
  - [../architecture/01-QOM对象模型深度分析.md](../architecture/01-QOM对象模型深度分析.md)
  - [../architecture/22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md](../architecture/22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md)
- Machine 与板级创建流程：
  - [../architecture/03-Machine建立流程深度分析.md](../architecture/03-Machine建立流程深度分析.md)
- 事件循环、协程、IOThread：
  - [../architecture/05-事件循环与IO模型深度分析.md](../architecture/05-事件循环与IO模型深度分析.md)
  - [../architecture/23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md](../architecture/23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md)
- 块层/virtio/定时器/IOMMU：
  - [../architecture/25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md](../architecture/25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md)
  - [../architecture/21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md](../architecture/21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md)
  - [../architecture/16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md](../architecture/16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md)
  - [../architecture/19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译.md](../architecture/19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译.md)
  - [../architecture/20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md](../architecture/20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md)

### 15.2 与 `arm/` 的推荐联读

- ARM Generic Timer 架构背景：
  - [../arm/13-ARM-Generic-Timer深度分析.md](../arm/13-ARM-Generic-Timer深度分析.md)
- GIC 与中断控制：
  - [../arm/11-GICv3中断控制器深度分析.md](../arm/11-GICv3中断控制器深度分析.md)
  - [../arm/24-ARM中断控制器架构GICv2与GICv3对比分析.md](../arm/24-ARM中断控制器架构GICv2与GICv3对比分析.md)
- ARM VirtIO / PCI / SMMUv3 背景：
  - [../arm/19-VirtIO设备模型与传输层深度分析.md](../arm/19-VirtIO设备模型与传输层深度分析.md)
  - [../arm/20-PCI-PCIe子系统深度分析.md](../arm/20-PCI-PCIe子系统深度分析.md)
  - [../arm/21-ARM-SMMUv3与IOMMU框架深度分析.md](../arm/21-ARM-SMMUv3与IOMMU框架深度分析.md)

### 15.3 如何把三个目录串起来

可以用下面的方式理解：

- `architecture/`：解释抽象机制与共性框架。
- `device-model/`：解释这些机制如何落到具体设备/后端子系统。
- `arm/`：解释 ARM 架构语义如何约束这些设备模型与平台实现。

换句话说：

```text
architecture = 原理层
device-model = 设备实现层
arm = 架构语义/平台约束层
```

---

## 结语

如果只保留一句导航建议，那就是：**先用 `00` 建骨架，再用 `03/04/06` 看典型设备，再回到 `02/05/07/08` 追数据面、地址翻译与时间模型。**

这样读，整个 `device-model/` 目录就不再是 8 篇孤立文档，而会变成一张从 **QOM → Bus → Device → Backend → Host Resource** 连续展开的完整地图。
