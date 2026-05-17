# QEMU 架构子系统综合导航

> **版本**: QEMU 11.0.50  
> **定位**: `darren/architecture/` 目录总览、阅读入口与交叉索引  
> **覆盖范围**: 27 篇架构文档（00-26），聚焦 QOM、事件循环、Machine 建立、内存、Block、TCG、VirtIO、PCI、IOMMU 与调试相关子系统

---

## 目录

- [1. 全景架构图](#1-全景架构图)
- [2. 文档地图](#2-文档地图)
- [3. 全局架构](#3-全局架构)
- [4. QOM 对象模型](#4-qom-对象模型)
- [5. 执行循环与事件](#5-执行循环与事件)
- [6. Machine 建立](#6-machine-建立)
- [7. 内存子系统](#7-内存子系统)
- [8. Block / IO](#8-block--io)
- [9. TCG](#9-tcg)
- [10. VirtIO](#10-virtio)
- [11. 定时器](#11-定时器)
- [12. PMU](#12-pmu)
- [13. Debug](#13-debug)
- [14. IOMMU](#14-iommu)
- [15. PCI](#15-pci)
- [16. 阅读路线推荐](#16-阅读路线推荐)
- [17. 源文件索引与跨目录跳转](#17-源文件索引与跨目录跳转)

---

## 1. 全景架构图

```text
                          +-----------------------------+
                          |         system/main.c       |
                          |    main() / qemu_init()     |
                          +--------------+--------------+
                                         |
                                         v
                          +-----------------------------+
                          |     全局初始化 / 主循环      |
                          |  vl.c / runstate / main-loop |
                          +------+------------------+----+
                                 |                  |
                 +---------------+                  +----------------+
                 |                                                   |
                 v                                                   v
     +------------------------+                         +---------------------------+
     |      QOM 类型系统      |                         |     Machine/Board 建立    |
     | TypeInfo/ObjectClass   |                         | virt machine / CPU / Bus  |
     +-----------+------------+                         +-------------+-------------+
                 |                                                            |
      +----------+----------+                                 +---------------+----------------+
      |                     |                                 |               |                |
      v                     v                                 v               v                v
+------------+     +----------------+               +----------------+ +---------------+ +------------+
| Device/QDev|     | MemoryRegion树 |               | CPU / vCPU线程 | | PCI / VirtIO  | | 定时器/PMU |
| Bus / IRQ  |     | AddressSpace   |               | TCG / KVM循环  | | MMIO / DMA    | | Debug/IOMMU|
+-----+------+     +--------+-------+               +--------+-------+ +-------+-------+ +------+-----+
      |                     |                                |                 |                |
      |                     v                                v                 v                |
      |            +------------------+              +----------------+ +----------------+      |
      |            | FlatView / 分发  |<-------------| cpu_exec / TB  | | virtqueue/ring |      |
      |            | RAM / MMIO / DMA |              | 中断/异常/MMIO | | block/net/scsi |      |
      |            +--------+---------+              +--------+-------+ +--------+-------+      |
      |                     |                                 |                  |                |
      |                     +-------------------+-------------+------------------+----------------+
      |                                         |
      v                                         v
+--------------------+               +--------------------------+
| Block Layer / BDS  |<------------->| AioContext / Coroutine   |
| qcow2/raw/file-posix|               | BH / Timer / IOThread    |
+--------------------+               +--------------------------+
```

**把握主线的一个简式心智模型：**
- **QOM** 负责“对象怎么定义、注册、继承、实例化”。
- **Machine** 决定“这台虚拟机有哪些 CPU、内存、总线、设备”。
- **内存系统** 决定“Guest 地址如何映射到 RAM / MMIO / IOMMU”。
- **执行循环** 决定“CPU 指令、事件、IO 完成、定时器如何推进”。
- **Block / VirtIO / PCI** 决定“设备请求怎样穿过队列、总线和后端”。
- **TCG** 决定“无硬件加速时指令如何翻译、缓存、执行与失效”。

---

## 2. 文档地图

| # | 文档 | 大小 | 主题摘要 | 适合解答的问题 |
|---|---|---:|---|---|
| 00 | [00-全局架构概览](./00-全局架构概览.md) | 19KB | 项目结构、启动流程、核心框架总览 | QEMU 从哪启动？主框架有哪些？ |
| 01 | [01-QOM对象模型深度分析](./01-QOM对象模型深度分析.md) | 26KB | TypeImpl、注册、类/实例初始化、接口 | 新类型如何注册？class_init 做什么？ |
| 02 | [02-模拟执行循环与MMIO分发深度分析](./02-模拟执行循环与MMIO分发深度分析.md) | 47KB | main loop、cpu_exec、MMIO 分发、异步完成 | 一次 MMIO/virtio 请求怎样跑完整路径？ |
| 03 | [03-Machine建立流程深度分析](./03-Machine建立流程深度分析.md) | 45KB | virt 机型初始化、CPU/内存/设备创建 | ARM virt 机器如何搭起来？ |
| 04 | [04-线程模型与锁机制深度分析](./04-线程模型与锁机制深度分析.md) | 36KB | 线程角色、BQL、vCPU 与并发协作 | 哪些路径持 BQL？vCPU 与主线程如何配合？ |
| 05 | [05-事件循环与IO模型深度分析](./05-事件循环与IO模型深度分析.md) | 43KB | AioContext、poll、fd handler、事件模型 | QEMU 的 IO 事件是怎样被驱动的？ |
| 06 | [06-块层核心架构深度分析](./06-块层核心架构深度分析.md) | 13KB | Block 层核心对象与请求流 | 块层总入口是什么？ |
| 07 | [07-qcow2格式驱动与块IO后端深度分析](./07-qcow2格式驱动与块IO后端深度分析.md) | 12KB | qcow2/raw/file-posix 分层 | 镜像格式层与宿主文件后端如何衔接？ |
| 08 | [08-TCG后端深度分析-IR生成寄存器分配与代码缓存](./08-TCG后端深度分析-IR生成寄存器分配与代码缓存.md) | 17KB | TCG IR、寄存器分配、代码缓存 | TB 如何生成并落到代码缓存？ |
| 09 | [09-TCG深入分析-优化遍向量指令与Softmmu-TLB机制](./09-TCG深入分析-优化遍向量指令与Softmmu-TLB机制.md) | 21KB | 优化 pass、向量、SoftMMU TLB | TCG 优化点和 TLB 快路径在哪里？ |
| 10 | [10-VirtIO设备深度分析-Virtqueue实现vhost-user与IOMMU集成](./10-VirtIO设备深度分析-Virtqueue实现vhost-user与IOMMU集成.md) | 20KB | virtqueue、vhost-user、IOMMU 连接 | virtqueue 与 IOMMU / vhost-user 如何互动？ |
| 11 | [11-内存子系统深度分析-MemoryRegion树FlatView与RAMBlock](./11-内存子系统深度分析-MemoryRegion树FlatView与RAMBlock.md) | 20KB | MemoryRegion 树、FlatView、RAMBlock | 地址空间为何要“树 + 扁平化”？ |
| 12 | [12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障](./12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障.md) | 24KB | MTTCG、TB 失效、同步与屏障 | 多线程 TCG 如何保证一致性？ |
| 13 | [13-TCG前端翻译深度分析-指令解码IR生成与优化Pass](./13-TCG前端翻译深度分析-指令解码IR生成与优化Pass.md) | 28KB | 前端解码、IR 生成、优化 | Guest 指令怎样变成 TCG IR？ |
| 14 | [14-TCG后端代码生成深度分析-AArch64后端寄存器分配与TLB慢路径](./14-TCG后端代码生成深度分析-AArch64后端寄存器分配与TLB慢路径.md) | 27KB | AArch64 后端、发射、慢路径 | TLB miss / helper 慢路径怎样发射？ |
| 15 | [15-内存子系统深度分析-MemoryRegion-AddressSpace-FlatView与IOMMU](./15-内存子系统深度分析-MemoryRegion-AddressSpace-FlatView与IOMMU.md) | 37KB | AddressSpace、FlatView、IOMMU 关联 | IOMMU 如何嵌入内存框架？ |
| 16 | [16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架](./16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md) | 30KB | QEMUTimer、ptimer、Clock、ARM 定时器 | 定时器在哪个上下文触发？ |
| 17 | [17-PMU性能监控单元深度分析-事件计数器-溢出中断与EL过滤](./17-PMU性能监控单元深度分析-事件计数器-溢出中断与EL过滤.md) | 30KB | PMU 计数器、溢出中断、EL 过滤 | PMU 事件何时计数/溢出/注入？ |
| 18 | [18-Debug-Breakpoint-Watchpoint调试子系统深度分析-硬件断点-单步与GDB-Stub](./18-Debug-Breakpoint-Watchpoint调试子系统深度分析-硬件断点-单步与GDB-Stub.md) | 31KB | 断点、观察点、单步、GDB stub | 调试异常与 GDB stub 怎样接线？ |
| 19 | [19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译](./19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译.md) | 39KB | SMMUv3、Stream Table、PTW、命令队列 | DMA 地址翻译是如何完成的？ |
| 20 | [20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM](./20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md) | 40KB | PCI 设备模型、BAR、MSI/MSI-X、ECAM | PCI 配置空间与中断怎样工作？ |
| 21 | [21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速](./21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md) | 34KB | VRing、通知、PCI/MMIO、vhost | VirtIO 传输层差异在哪里？ |
| 22 | [22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型](./22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md) | 32KB | TypeInfo、ObjectClass、Property、宏与继承 | 设备类型声明宏如何落到 QOM？ |
| 23 | [23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread](./23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md) | 28KB | 协程、BH、定时器、IOThread | 协程与事件循环是怎样衔接的？ |
| 24 | [24-MemoryRegion-AddressSpace内存子系统深度分析-区域树-FlatView-地址分发与内存监听](./24-MemoryRegion-AddressSpace内存子系统深度分析-区域树-FlatView-地址分发与内存监听.md) | 29KB | 区域树、地址分发、内存监听 | 内存拓扑变化后谁接收通知？ |
| 25 | [25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流](./25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md) | 29KB | BDS、协程 IO、请求追踪、限流 | Block 请求如何穿过协程与限流器？ |
| 26 | [26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输](./26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md) | 27KB | virtio-blk/net、queue、PCI 传输 | 从 virtqueue 到具体设备回调怎么走？ |

> **使用方法**：先按“问题类型”找表，再跳到对应主题章节；如果问题横跨多个子系统，优先读 `00`、`02`、`03`、`11/15/24`、`21/26` 建立公共主线。

---

## 3. 全局架构

**关键概念**
- 主线是 `main() -> qemu_init() -> machine_run_board_init() -> qemu_main_loop()`。
- 全局框架由 QOM、Machine、主循环、内存、CPU/加速器、设备模型共同组成。
- 先建立“层次图”，后读细节，效率最高。

**读哪些文档，为什么**
- [00](./00-全局架构概览.md)：总入口，先拿到全局坐标系。

**交叉参考**
- 对象定义看 [4](#4-qom-对象模型)，运行时推进看 [5](#5-执行循环与事件)，平台装配看 [6](#6-machine-建立)。

---

## 4. QOM 对象模型

**关键概念**
- `TypeImpl/TypeInfo/ObjectClass/Object` 构成类型与实例骨架。
- `class_init` 定义类行为，`instance_init` 定义实例构造。
- Property 与 `DECLARE_*` 宏把设备声明、继承和访问标准化。

**读哪些文档，为什么**
- [01](./01-QOM对象模型深度分析.md)：看类型注册、接口、生命周期。
- [22](./22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md)：看 Property、声明宏、设备继承。

**交叉参考**
- Machine、MemoryRegion、PCI/VirtIO 设备都建立在 QOM 上，见 [6](#6-machine-建立)、[7](#7-内存子系统)、[10](#10-virtio)、[15](#15-pci)。

---

## 5. 执行循环与事件

**关键概念**
- 外层是 main loop，内层是 vCPU 执行循环。
- `AioContext`、BH、定时器、协程、IOThread 构成事件推进骨架。
- BQL 决定很多并发路径的真实边界。

**读哪些文档，为什么**
- [02](./02-模拟执行循环与MMIO分发深度分析.md)：抓总路径。
- [05](./05-事件循环与IO模型深度分析.md)：看 AIO/poll/fd handler。
- [04](./04-线程模型与锁机制深度分析.md)：看线程与 BQL。
- [23](./23-主事件循环与协程深度分析-AioContext-BH-定时器-Coroutine-defer_call与IOThread.md)：看协程、BH、IOThread。

**交叉参考**
- 后端 IO 见 [8](#8-block--io)，CPU 执行见 [9](#9-tcg)，定时事件见 [11](#11-定时器)。

---

## 6. Machine 建立

**关键概念**
- `machine_run_board_init()` 把抽象机型落成具体板级实例。
- ARM `virt` 会接好 CPU、内存、GIC、UART、PCIe、virtio-mmio/FDT。
- 建机阶段决定后续地址空间和设备拓扑。

**读哪些文档，为什么**
- [03](./03-Machine建立流程深度分析.md)：回答“设备/CPU/地址映射是在哪儿建出来的”。

**交叉参考**
- 对象来源见 [4](#4-qom-对象模型)，地址映射见 [7](#7-内存子系统)，设备接线见 [10](#10-virtio)、[15](#15-pci)。

---

## 7. 内存子系统

**关键概念**
- 核心模型是 **MemoryRegion 树 + FlatView + AddressSpace**。
- `RAMBlock` 管 RAM，`MemoryRegionOps` 管 MMIO。
- IOMMU、listener、别名与优先级决定高级路径行为。

**读哪些文档，为什么**
- [11](./11-内存子系统深度分析-MemoryRegion树FlatView与RAMBlock.md)：核心结构。
- [24](./24-MemoryRegion-AddressSpace内存子系统深度分析-区域树-FlatView-地址分发与内存监听.md)：分发与 listener。
- [15](./15-内存子系统深度分析-MemoryRegion-AddressSpace-FlatView与IOMMU.md)：AddressSpace 与 IOMMU 结合。

**交叉参考**
- MMIO 分发见 [5](#5-执行循环与事件)，softmmu/TLB 见 [9](#9-tcg)，DMA/IOMMU 见 [14](#14-iommu)。

---

## 8. Block / IO

**关键概念**
- `BlockDriverState` 是块层核心对象。
- qcow2/raw/file-posix 展示“格式层 + 协议/后端层”堆叠。
- Block 请求大量依赖协程与 AIO 上下文。

**读哪些文档，为什么**
- [06](./06-块层核心架构深度分析.md)：块层角色与入口。
- [07](./07-qcow2格式驱动与块IO后端深度分析.md)：镜像层与文件后端。
- [25](./25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md)：协程 IO、追踪、限流。

**交叉参考**
- 事件模型见 [5](#5-执行循环与事件)，前端设备样例见 [10](#10-virtio)。

---

## 9. TCG

**关键概念**
- TCG 分前端解码/IR 生成与后端优化/代码发射两层。
- TB 是翻译、缓存、执行的基本单位。
- SoftMMU TLB 与 MTTCG 分别对应访存性能和并行执行。

**读哪些文档，为什么**
- [13](./13-TCG前端翻译深度分析-指令解码IR生成与优化Pass.md)：前端入口。
- [08](./08-TCG后端深度分析-IR生成寄存器分配与代码缓存.md)：后端主线。
- [09](./09-TCG深入分析-优化遍向量指令与Softmmu-TLB机制.md)：优化与 TLB。
- [14](./14-TCG后端代码生成深度分析-AArch64后端寄存器分配与TLB慢路径.md)：AArch64 发射与慢路径。
- [12](./12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障.md)：并行与一致性。

**交叉参考**
- 运行时骨架见 [5](#5-执行循环与事件)，访存框架见 [7](#7-内存子系统)，断点单步见 [13](#13-debug)。

---

## 10. VirtIO

**关键概念**
- 稳定核心是 `virtqueue/vring/notify`。
- 差异主要在传输层：MMIO、PCI、vhost。
- virtio-blk/net 是观察前后端协作的最佳样例。

**读哪些文档，为什么**
- [21](./21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md)：统一看传输层与通知。
- [26](./26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md)：看具体设备路径。
- [10](./10-VirtIO设备深度分析-Virtqueue实现vhost-user与IOMMU集成.md)：看 vhost-user / IOMMU 集成。

**交叉参考**
- virtio-pci 见 [15](#15-pci)，virtio-blk 后端见 [8](#8-block--io)，DMA 见 [14](#14-iommu)。

---

## 11. 定时器

**关键概念**
- `QEMUTimer` 是基础设施，`ptimer` 是常见设备封装。
- `Clock` 框架连接通用时钟与架构定时器。
- 定时器触发语境要放回主循环/AIO 去看。

**读哪些文档，为什么**
- [16](./16-时钟与定时器子系统深度分析-QEMUTimer-ptimer-ARM-Generic-Timer与Clock框架.md)：从基础到 ARM Generic Timer 一次读全。

**交叉参考**
- 事件推进见 [5](#5-执行循环与事件)，与 PMU/Debug 的异步事件关系见 [12](#12-pmu)、[13](#13-debug)。

---

## 12. PMU

**关键概念**
- PMU 维护事件计数、溢出状态和中断注入。
- 计数是否生效受 EL、过滤条件和事件源影响。
- 本质上它挂在 CPU 执行流上。

**读哪些文档，为什么**
- [17](./17-PMU性能监控单元深度分析-事件计数器-溢出中断与EL过滤.md)：看计数、过滤、溢出。

**交叉参考**
- 计数挂接点可回看 [9](#9-tcg)，异步事件语境见 [11](#11-定时器)。

---

## 13. Debug

**关键概念**
- 断点、观察点、单步、GDB stub 共同构成调试链路。
- 调试能力会影响 TB 切分、访存拦截和异常注入。
- 外部调试器最终通过 GDB stub 连接 QEMU 内部状态。

**读哪些文档，为什么**
- [18](./18-Debug-Breakpoint-Watchpoint调试子系统深度分析-硬件断点-单步与GDB-Stub.md)：适合查单步、watchpoint、stub 接线。

**交叉参考**
- TB 行为看 [9](#9-tcg)，watchpoint 相关访存路径看 [7](#7-内存子系统)。

---

## 14. IOMMU

**关键概念**
- IOMMU 处理设备侧 DMA 地址翻译，不是 CPU 访存翻译。
- SMMUv3 关键点是 Stream Table、命令队列、PTW。
- 在 QEMU 里它既是设备模型，也是内存框架扩展点。

**读哪些文档，为什么**
- [19](./19-SMMUv3-IOMMU深度分析-Stream-Table页表遍历命令队列与DMA地址翻译.md)：看 SMMUv3 细节。
- [15](./15-内存子系统深度分析-MemoryRegion-AddressSpace-FlatView与IOMMU.md)：看它怎样嵌到通用内存系统。

**交叉参考**
- 地址空间基础见 [7](#7-内存子系统)，DMA 设备接入见 [10](#10-virtio)、[15](#15-pci)。

---

## 15. PCI

**关键概念**
- PCI/PCIe 组织设备发现、配置空间、BAR、中断与总线层级。
- ECAM 负责配置访问，BAR 负责暴露 MMIO/PIO 窗口。
- virtio-pci 是 VirtIO 接入总线的关键桥梁。

**读哪些文档，为什么**
- [20](./20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md)：回答配置空间、BAR、MSI/MSI-X、ECAM 的主问题。

**交叉参考**
- virtio-pci 见 [10](#10-virtio)，BAR 映射见 [7](#7-内存子系统)，host bridge 建立见 [6](#6-machine-建立)。

---

## 16. 阅读路线推荐

### 路线 A：入门
- [00](./00-全局架构概览.md) -> [03](./03-Machine建立流程深度分析.md) -> [02](./02-模拟执行循环与MMIO分发深度分析.md) -> [11](./11-内存子系统深度分析-MemoryRegion树FlatView与RAMBlock.md) -> [21](./21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md)
- **适合**：先搭全局地图。

### 路线 B：设备开发
- [01](./01-QOM对象模型深度分析.md) -> [22](./22-QOM对象模型深度分析-TypeInfo-ObjectClass-Property-接口继承与设备模型.md) -> [03](./03-Machine建立流程深度分析.md) -> [20](./20-PCI-PCIe子系统深度分析-设备模型-配置空间-BAR映射-MSI-MSI-X中断与ECAM.md) -> [26](./26-VirtIO设备模型深度分析-VirtQueue-通知机制-virtio-blk-net与PCI传输.md)
- **适合**：新增设备、属性、总线和通知逻辑。

### 路线 C：性能优化
- [02](./02-模拟执行循环与MMIO分发深度分析.md) -> [04](./04-线程模型与锁机制深度分析.md) -> [05](./05-事件循环与IO模型深度分析.md) -> [12](./12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障.md) -> [25](./25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md)
- **适合**：BQL、IOThread、协程、TB 失效路径。

### 路线 D：内存 / IO
- [11](./11-内存子系统深度分析-MemoryRegion树FlatView与RAMBlock.md) -> [24](./24-MemoryRegion-AddressSpace内存子系统深度分析-区域树-FlatView-地址分发与内存监听.md) -> [15](./15-内存子系统深度分析-MemoryRegion-AddressSpace-FlatView与IOMMU.md) -> [21](./21-VirtIO设备模型深度分析-VirtQueue-VRing-通知机制-PCI-MMIO传输与vhost加速.md) -> [25](./25-Block-Layer-IO子系统深度分析-BlockDriverState-协程IO-请求追踪与限流.md)
- **适合**：追踪访存、DMA、virtio-blk 数据通路。

### 路线 E：TCG 内核
- [13](./13-TCG前端翻译深度分析-指令解码IR生成与优化Pass.md) -> [08](./08-TCG后端深度分析-IR生成寄存器分配与代码缓存.md) -> [09](./09-TCG深入分析-优化遍向量指令与Softmmu-TLB机制.md) -> [14](./14-TCG后端代码生成深度分析-AArch64后端寄存器分配与TLB慢路径.md) -> [12](./12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障.md)
- **适合**：前端翻译、TB、TLB、后端发射、MTTCG。

---

## 17. 源文件索引与跨目录跳转

### 17.1 本目录主题对应的核心源码入口

| 主题 | 典型源码入口 |
|---|---|
| 启动 / 主循环 | `system/main.c`, `system/vl.c`, `system/runstate.c`, `util/main-loop.c` |
| QOM | `qom/object.c`, `include/qom/object.h` |
| qdev / 设备 | `hw/core/qdev.c`, `include/hw/qdev-core.h` |
| 内存 | `system/memory.c`, `system/physmem.c`, `include/system/memory.h` |
| 块层 | `block.c`, `block/`, `include/block/block_int-common.h` |
| TCG | `accel/tcg/`, `tcg/`, `include/exec/` |
| VirtIO | `hw/virtio/`, `include/hw/virtio/virtio.h` |
| PCI | `hw/pci/`, `include/hw/pci/pci.h` |
| 定时器 | `util/qemu-timer.c`, `include/qemu/timer.h`, `hw/timer/` |
| IOMMU | `hw/arm/`, `system/memory.c`, `include/system/memory.h` |
| 调试 | `gdbstub/`, `cpu-common.c`, `target/arm/` |

### 17.2 与其他中文分析目录的交叉引用

- [`../arm64/`](../arm64/)：聚焦 **ARM64 目标架构视角**，适合把本目录的通用机制映射到 EL、异常、TLB、GIC、TCG 翻译等 ARM64 具体实现。
- [`../accel/`](../accel/)：聚焦 **加速与执行后端**，尤其适合继续深挖 TCG、优化递次、SoftMMU、插件、多线程翻译。
- [`../arm/`](../arm/)：聚焦 **ARM 通用架构组件**，例如 Generic Timer、PMU、调试架构、SMMUv3、PCI/VirtIO 在 ARM 平台上的接入。
- [`../device-model/`](../device-model/)：聚焦 **设备模型与数据通路**，适合补 virtio、块设备、VFIO、网络后端、DMA 设备等具体设备实现。

### 17.3 一句话导航

- 想知道“**系统怎么起来**”：先看 `00 -> 03 -> 02`。
- 想知道“**对象怎么组织**”：先看 `01 -> 22`。
- 想知道“**地址怎么分发**”：先看 `11 -> 24 -> 15`。
- 想知道“**IO 怎么跑通**”：先看 `21/26 -> 06/25 -> 05/23`。
- 想知道“**TCG 怎么工作**”：先看 `13 -> 08 -> 09 -> 14 -> 12`。

---

这份导航文档的目标不是替代 27 篇原文，而是提供一个**低成本切入口**：先用它定位问题属于哪个层面，再跳到对应深度文档做细读。