# QEMU 关键设备仿真深度分析：UART、磁盘、网卡

> QEMU 版本：11.0.50  
> 分析范围：PL011 UART 串口仿真、virtio-blk 块设备仿真、virtio-net 网络设备仿真  
> 聚焦：完整数据路径、寄存器级实现、中断/通知机制、后端集成、CLI 使用

---

## 目录

- [第一部分：PL011 UART 串口仿真](#第一部分pl011-uart-串口仿真)
  - [1. PL011 概述与 QOM 层级](#1-pl011-概述与-qom-层级)
  - [2. PL011State 结构体](#2-pl011state-结构体)
  - [3. MMIO 寄存器映射](#3-mmio-寄存器映射)
  - [4. 寄存器位域定义](#4-寄存器位域定义)
  - [5. TX 发送路径](#5-tx-发送路径)
  - [6. RX 接收路径](#6-rx-接收路径)
  - [7. 中断逻辑](#7-中断逻辑)
  - [8. Chardev 后端集成](#8-chardev-后端集成)
  - [9. 初始化与复位](#9-初始化与复位)
  - [10. virt 机器集成与使用](#10-virt-机器集成与使用)
- [第二部分：virtio-blk 块设备仿真](#第二部分virtio-blk-块设备仿真)
  - [11. virtio-blk 概述](#11-virtio-blk-概述)
  - [12. VirtIOBlock 结构体](#12-virtioblock-结构体)
  - [13. 设备 Realize 流程](#13-设备-realize-流程)
  - [14. 请求处理管线](#14-请求处理管线)
  - [15. 请求类型详解](#15-请求类型详解)
  - [16. I/O 完成路径](#16-io-完成路径)
  - [17. 多队列与 IOThread](#17-多队列与-iothread)
  - [18. BlockBackend 集成](#18-blockbackend-集成)
  - [19. 特性协商](#19-特性协商)
  - [20. CLI 使用与配置](#20-cli-使用与配置)
- [第三部分：virtio-net 网络设备仿真](#第三部分virtio-net-网络设备仿真)
  - [21. virtio-net 概述](#21-virtio-net-概述)
  - [22. VirtIONet 结构体](#22-virtionet-结构体)
  - [23. 设备 Realize 流程](#23-设备-realize-流程)
  - [24. RX 接收路径](#24-rx-接收路径)
  - [25. TX 发送路径](#25-tx-发送路径)
  - [26. 控制队列](#26-控制队列)
  - [27. 多队列与 RSS](#27-多队列与-rss)
  - [28. 特性协商](#28-特性协商)
  - [29. 网络后端与 vhost 集成](#29-网络后端与-vhost-集成)
  - [30. CLI 使用与配置](#30-cli-使用与配置)
- [附录](#附录)

---

## 第一部分：PL011 UART 串口仿真

### 1. PL011 概述与 QOM 层级

PL011 是 ARM PrimeCell UART，QEMU 对其进行了完整的寄存器级仿真。它是 ARM virt 机器的主要串口设备，用于控制台输出和调试。

**QOM 类型层级**：

```
Object
  └── DeviceState (TYPE_DEVICE)
        └── SysBusDevice (TYPE_SYS_BUS_DEVICE)
              └── PL011State (TYPE_PL011 = "pl011")
                    └── PL011LuminaryState (TYPE_PL011_LUMINARY = "pl011_luminary")
```

**注册**（pl011.c:702-729）：

```c
static const TypeInfo pl011_arm_info = {
    .name          = TYPE_PL011,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(PL011State),
    .instance_init = pl011_init,
    .class_init    = pl011_class_init,
};
type_init(pl011_register_types)
```

Luminary 变体继承自 PL011，共享相同结构体但覆盖 class_init 以调整行为。

### 2. PL011State 结构体

定义于 `pl011.h:31-61`：

```c
struct PL011State {
    SysBusDevice parent_obj;

    MemoryRegion iomem;            // MMIO 区域（0x1000 字节）

    /* 硬件寄存器 */
    uint32_t flags;                // FR: Flag Register（只读状态）
    uint32_t lcr;                  // LCR_H: Line Control Register
    uint32_t rsr;                  // RSR: Receive Status Register
    uint32_t cr;                   // CR: Control Register
    uint32_t dmacr;                // DMACR: DMA Control Register
    uint32_t int_enabled;          // IMSC: Interrupt Mask Set/Clear
    uint32_t int_level;            // RIS: Raw Interrupt Status
    uint32_t ilpr;                 // ILPR: IrDA Low-Power Counter
    uint32_t ibrd;                 // IBRD: Integer Baud Rate Divisor
    uint32_t fbrd;                 // FBRD: Fractional Baud Rate Divisor
    uint32_t ifl;                  // IFLS: Interrupt FIFO Level Select

    /* 接收 FIFO */
    uint32_t read_fifo[PL011_FIFO_DEPTH]; // 深度 16
    uint32_t read_pos;             // FIFO 读指针
    uint32_t read_count;           // FIFO 中的数据量
    uint32_t read_trigger;         // 触发中断的阈值

    /* 外部连接 */
    CharBackend chr;               // 字符后端（stdio/socket/pty）
    qemu_irq irq[6];              // 6 条中断线（合并 + 分开）
    Clock *clk;                    // 输入时钟

    /* 辅助状态 */
    bool migrate_clk;              // 迁移时钟状态
    bool logged_disabled_uart;     // 已记录禁用 UART 警告

    const unsigned char *id;       // 硬件 ID 寄存器值
};
```

### 3. MMIO 寄存器映射

PL011 的 MMIO 空间为 4KB（0x1000），寄存器按 32 位对齐，偏移除以 4 得到索引。

| 偏移 | 索引 | 名称 | 读/写 | 功能 | 代码位置 |
|------|------|------|-------|------|---------|
| 0x000 | 0 | UARTDR | R/W | 数据寄存器 | pl011.c:294,437 |
| 0x004 | 1 | UARTRSR/ECR | R/W | 接收状态/错误清除 | pl011.c:297,441 |
| 0x018 | 6 | UARTFR | R | 标志寄存器 | pl011.c:300 |
| 0x020 | 8 | UARTILPR | R/W | IrDA 低功耗计数 | pl011.c:303,447 |
| 0x024 | 9 | UARTIBRD | R/W | 整数波特率除数 | pl011.c:306,450 |
| 0x028 | 10 | UARTFBRD | R/W | 小数波特率除数 | pl011.c:309,454 |
| 0x02C | 11 | UARTLCR_H | R/W | 行控制寄存器 | pl011.c:312,458 |
| 0x030 | 12 | UARTCR | R/W | 控制寄存器 | pl011.c:315,473 |
| 0x034 | 13 | UARTIFLS | R/W | FIFO 中断级别选择 | pl011.c:318,482 |
| 0x038 | 14 | UARTIMSC | R/W | 中断掩码 | pl011.c:321,486 |
| 0x03C | 15 | UARTRIS | R | 原始中断状态 | pl011.c:324 |
| 0x040 | 16 | UARTMIS | R | 屏蔽后中断状态 | pl011.c:327 |
| 0x044 | 17 | UARTICR | W | 中断清除 | pl011.c:490 |
| 0x048 | 18 | UARTDMACR | R/W | DMA 控制 | pl011.c:330,494 |
| 0xFE0-0xFFC | - | ID | R | PrimeCell ID 寄存器 | pl011.c:333 |

### 4. 寄存器位域定义

定义于 `pl011.c:51-98`：

#### Flag Register (FR)

```c
#define PL011_FLAG_TXFE  0x80    // TX FIFO 空
#define PL011_FLAG_RXFF  0x40    // RX FIFO 满
#define PL011_FLAG_TXFF  0x20    // TX FIFO 满
#define PL011_FLAG_RXFE  0x10    // RX FIFO 空
```

#### 中断位

```c
#define INT_OE   (1 << 10)       // 溢出错误
#define INT_BE   (1 << 9)        // Break 错误
#define INT_PE   (1 << 8)        // 校验错误
#define INT_FE   (1 << 7)        // 帧错误
#define INT_RT   (1 << 6)        // 接收超时
#define INT_TX   (1 << 5)        // TX 中断
#define INT_RX   (1 << 4)        // RX 中断
#define INT_MS   (1 << 0)        // Modem 状态变更（实际多个位 3:0）
```

#### 控制寄存器 (CR)

```c
#define CR_UARTEN  (1 << 0)      // UART 使能
#define CR_LBE     (1 << 7)      // 环回使能
#define CR_TXE     (1 << 8)      // TX 使能
#define CR_RXE     (1 << 9)      // RX 使能
#define CR_DTR     (1 << 10)     // Data Terminal Ready
#define CR_RTS     (1 << 11)     // Request To Send
#define CR_OUT1    (1 << 12)     // 自定义输出 1
#define CR_OUT2    (1 << 13)     // 自定义输出 2
```

#### 行控制 (LCR_H)

```c
#define LCR_FEN    (1 << 4)      // FIFO 使能
#define LCR_BRK    (1 << 0)      // 发送 Break
```

### 5. TX 发送路径

Guest 向 UART 写入一个字节的完整路径：

```
Guest CPU 写 DR (偏移 0x000)
  │
  ▼
pl011_write() [pl011.c:428-504]
  └── case 0 (DR):
        └── pl011_write_txdata() [pl011.c:227-261]
              │
              ├── 1. 检查 UART 是否使能
              │     ├── !(s->cr & CR_UARTEN) → 警告（仅一次）并返回
              │     └── !(s->cr & CR_TXE)    → 警告并返回
              │
              ├── 2. 发送到字符后端
              │     └── qemu_chr_fe_write_all(&s->chr, &data, 1)
              │           └── 实际输出到 stdio / socket / pty 等
              │
              ├── 3. 环回处理
              │     └── pl011_loopback_tx() [pl011.c:199-225]
              │           └── 如果 CR_LBE 置位，将数据送回 RX FIFO
              │
              └── 4. 更新中断
                    ├── s->int_level |= INT_TX    // 设置 TX 中断
                    └── pl011_update()             // 重新评估中断线
```

**关键特性**：TX 路径是同步的，没有仿真的 TX FIFO 排空延迟。字节写入后立即传递给后端并立即触发 TX 中断。

**git 优化**：commit 907b8d5635 修改为只记录一次 "data written to disabled UART" 警告，避免日志风暴。

### 6. RX 接收路径

外部输入到达 Guest 的完整路径：

```
外部输入（终端键入 / 网络数据）
  │
  ▼
Chardev 后端
  │
  ├── 1. 询问是否可接收
  │     └── pl011_can_receive() [pl011.c:506-524]
  │           ├── FIFO 使能: 返回 PL011_FIFO_DEPTH(16) - read_count
  │           └── FIFO 禁用: 返回 1 - read_count（深度为 1）
  │
  ├── 2. 递送数据
  │     └── pl011_receive() [pl011.c:526-541]
  │           └── 逐字节调用 pl011_fifo_rx_put()
  │
  └── 3. Break 事件
        └── pl011_event(CHR_EVENT_BREAK) [pl011.c:543-548]
              └── 注入 DR_BE 标志到 FIFO
```

#### FIFO 写入细节

`pl011_fifo_rx_put()`（pl011.c:177-197）：

```
pl011_fifo_rx_put(s, value)
  │
  ├── 计算 FIFO 深度
  │     ├── LCR_FEN 置位: depth = 16
  │     └── LCR_FEN 未置位: depth = 1
  │
  ├── 如果 FIFO 已满 (read_count >= depth)
  │     └── 仅更新溢出位，丢弃数据
  │
  ├── 写入 FIFO
  │     └── read_fifo[(read_pos + read_count) & (depth-1)] = value
  │
  ├── 更新标志
  │     ├── read_count++
  │     ├── 清除 RXFE（FIFO 非空）
  │     └── 如果 read_count == depth → 设置 RXFF（FIFO 满）
  │
  └── 检查中断触发
        └── 如果 read_count == read_trigger
              └── s->int_level |= INT_RX → pl011_update()
```

#### Guest 读取 DR

`pl011_read_rxdata()`（pl011.c:263-285）：

```
Guest CPU 读 DR (偏移 0x000)
  │
  ▼
pl011_read_rxdata() [pl011.c:263-285]
  │
  ├── 取出 FIFO 头: c = read_fifo[read_pos]
  ├── read_count--; read_pos = (read_pos+1) & (depth-1)
  │
  ├── 更新标志
  │     ├── 如果 read_count == 0 → 设置 RXFE（空）
  │     └── 如果 read_count < depth → 清除 RXFF（非满）
  │
  ├── 检查中断
  │     └── 如果 read_count < read_trigger
  │           └── 清除 INT_RX → pl011_update()
  │
  ├── 更新 RSR（接收状态）
  │     └── s->rsr = c >> 8    // 高位含错误标志
  │
  └── 通知后端继续发送
        └── qemu_chr_fe_accept_input(&s->chr)
```

**git 修复**：commit 3e0f118f82 修复了 RX FIFO 深度计算错误，确保 FIFO 使能时真正使用 16 字节深度。

### 7. 中断逻辑

PL011 的中断系统通过三个寄存器协同工作：

```
                    ┌──────────┐
  各中断源 ────────►│ int_level │ = RIS (Raw Interrupt Status)
                    │ (原始中断)│
                    └────┬─────┘
                         │
                         │ AND
                         │
                    ┌────┴─────┐
                    │int_enabled│ = IMSC (Interrupt Mask)
                    │ (中断掩码)│
                    └────┬─────┘
                         │
                         ▼
                    MIS = RIS & IMSC
                         │
                         ▼
              ┌──────────────────────┐
              │ pl011_update()       │
              │ [pl011.c:132-142]    │
              │                      │
              │ flags = RIS & IMSC   │
              │ irq[5] = (flags != 0)│ ← 合并中断线（常用）
              │ irq[0] = modem IRQ   │
              │ irq[1] = rx IRQ      │ ← 分开的中断线
              │ irq[2] = tx IRQ      │
              │ irq[3] = rt IRQ      │
              │ irq[4] = err IRQ     │
              └──────────────────────┘
```

**中断源触发条件**：

| 中断 | 触发条件 | 清除方式 |
|------|---------|---------|
| INT_TX | 每次写 DR | 写 ICR bit 5 |
| INT_RX | FIFO 达到触发阈值 | 读 DR 使 count < trigger |
| INT_RT | 接收超时（QEMU 未完整仿真） | 写 ICR bit 6 |
| INT_OE | FIFO 溢出 | 写 ICR bit 10 |
| INT_BE | 收到 Break | 写 ICR bit 9 |
| INT_MS | Modem 状态变更 | 写 ICR bit 0 |

**IFLS（中断 FIFO 级别选择）** 控制 `read_trigger` 阈值，决定 RX FIFO 填充到多少字节时触发中断（默认 1/2 = 8 字节）。

### 8. Chardev 后端集成

PL011 通过 QEMU 的字符设备框架连接到外部 I/O：

```
PL011State                  CharBackend               实际后端
  │                           │                         │
  ├── chr (CharBackend) ──────┤                         │
  │                           ├── qemu_chr_fe_write_all() ──► stdio / socket / pty
  │                           │                         │
  │   pl011_can_receive() ◄───┤                         │
  │   pl011_receive()    ◄────┤◄──────── 输入数据 ◄──────┤
  │   pl011_event()      ◄────┤◄──────── Break 事件 ◄───┤
  │                           │                         │
  └── qemu_chr_fe_accept_input() ──► 解除流控 ──────────►│
```

**属性定义**（pl011.c:640-643）：

```c
static const Property pl011_properties[] = {
    DEFINE_PROP_CHR("chardev", PL011State, chr),
};
```

**后端安装**（pl011.c:663-669，在 realize 中）：

```c
qemu_chr_fe_set_handlers(&s->chr,
    pl011_can_receive,    // 询问可接收字节数
    pl011_receive,        // 递送数据
    pl011_event,          // 事件（Break 等）
    NULL,                 // 状态变更
    s, NULL, true);
```

**常见后端类型**：

| 后端 | CLI | 典型用途 |
|-----|-----|---------|
| stdio | `-serial stdio` | 控制台终端 |
| pty | `-serial pty` | 伪终端 |
| socket | `-serial tcp:host:port` | 网络串口 |
| file | `-serial file:path` | 日志输出 |
| null | `-serial null` | 丢弃 |

### 9. 初始化与复位

#### instance_init（pl011.c:645-661）

```c
pl011_init(obj)
  ├── memory_region_init_io(&s->iomem, obj, &pl011_ops, s, "pl011", 0x1000)
  ├── sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->iomem)
  ├── sysbus_init_irq(dev, &s->irq[i])    // × 6 条中断线
  ├── s->clk = qdev_init_clock_in(dev, "clk", pl011_clock_update, s, ClockUpdate)
  └── s->id = pl011_id_arm                 // ARM 标准 ID
```

#### reset（pl011.c:671-690）

```c
pl011_reset(dev)
  ├── s->lcr = 0;  s->rsr = 0;  s->dmacr = 0
  ├── s->int_enabled = 0;  s->int_level = 0
  ├── s->ilpr = 0;  s->ibrd = 0;  s->fbrd = 0
  ├── s->read_trigger = 1               // 默认：1 字节触发
  ├── s->ifl = 0x12                     // IFLS 默认值（1/2 满）
  ├── s->cr = CR_TXE | CR_RXE           // 默认 TX/RX 使能
  ├── s->flags = 0
  ├── s->flags |= PL011_FLAG_TXFE       // TX FIFO 空
  └── s->flags |= PL011_FLAG_RXFE       // RX FIFO 空
```

### 10. virt 机器集成与使用

#### 创建流程（virt.c:1295-1344）

```c
create_uart(VirtMachineState *vms, int uart, MemoryRegion *mem, Chardev *chr)
  │
  ├── qdev_new(TYPE_PL011)
  ├── qdev_prop_set_chr(dev, "chardev", chr)     // 连接字符后端
  ├── sysbus_realize_and_unref(SYS_BUS_DEVICE(dev))
  ├── sysbus_mmio_map(dev, 0, base)              // 0x09000000
  ├── sysbus_connect_irq(dev, 0, gic_spi)        // GIC SPI 1
  │
  └── 设备树节点:
        ├── compatible = "arm,pl011", "arm,primecell"
        ├── reg = <0x09000000, 0x1000>
        ├── interrupts = <GIC_SPI 1 IRQ_TYPE_LEVEL_HIGH>
        ├── clocks = <apb_pclk>
        └── /chosen { stdout-path = "/pl011@9000000" }  // UART0 作为控制台
```

#### 典型 CLI 使用

```bash
# 默认：UART0 连接到 stdio
qemu-system-aarch64 -M virt -serial stdio

# UART0 连接到 pty，UART1 连接到 socket
qemu-system-aarch64 -M virt \
  -serial pty \
  -serial tcp:localhost:4444,server=on,wait=off

# 仅 UART0，连接到文件
qemu-system-aarch64 -M virt -serial file:uart.log
```

#### 完整数据流示例（用户在终端输入字符 'A'）

```
1. 用户在 stdio 终端按下 'A'
2. QEMU 主循环 → chardev stdio 后端检测到输入
3. 调用 pl011_can_receive() → 返回 FIFO 空余空间
4. 调用 pl011_receive() → pl011_fifo_rx_put(s, 'A')
5. FIFO: read_fifo[0] = 0x41, read_count = 1
6. read_count(1) == read_trigger(1) → 设置 INT_RX
7. pl011_update() → irq[5] 拉高 → GIC SPI 1 触发
8. Guest 内核 UART 驱动收到中断
9. 中断处理：读取 MIS 确认 RX 中断
10. 读取 DR → 获得 0x41 ('A')
11. read_count 降为 0 → 清除 INT_RX → irq 拉低
12. qemu_chr_fe_accept_input() → 通知后端可继续发送
13. 字符 'A' 进入 tty 层 → 显示在 Guest 控制台
```

---

## 第二部分：virtio-blk 块设备仿真

### 11. virtio-blk 概述

virtio-blk 是 QEMU 中最常用的虚拟块设备，通过 virtio 框架提供高性能磁盘 I/O。它将 Guest 的块 I/O 请求转换为 QEMU 块层的异步操作。

**QOM 类型层级**：

```
Object → DeviceState → VirtIODevice → VirtIOBlock

传输层包装:
  VirtIOBlockPCI  (virtio-blk-pci)  — PCI 传输
  VirtIOMMIOProxy (virtio-blk-device) — MMIO 传输
```

### 12. VirtIOBlock 结构体

定义于 `virtio-blk.h:37-93`：

```c
struct VirtIOBlock {
    VirtIODevice parent_obj;       // virtio 设备基类

    /* 块后端 */
    BlockBackend *blk;             // 块后端（镜像文件/块设备）

    /* 请求管理 */
    void *rq;                      // 活跃请求链表
    QemuMutex rq_lock;             // 请求锁

    /* 配置 */
    VirtIOBlkConf conf;            // 设备配置
    uint64_t host_features;        // 主机特性
    size_t config_size;            // 配置空间大小

    /* 多队列 */
    AioContext **vq_aio_context;   // 每队列 AioContext
    uint64_t sector_mask;          // 扇区对齐掩码
};
```

#### VirtIOBlkConf 配置（virtio-blk.h:37-55）

```c
struct VirtIOBlkConf {
    BlockConf conf;                // 通用块配置（含 BlockBackend *blk）
    IOThread *iothread;            // 单 IOThread（旧方式）
    IOThreadVirtQueueMappingList *iothread_vq_mapping; // 每队列 IOThread 映射
    char *serial;                  // 设备序列号
    uint32_t request_merging;      // 请求合并使能
    uint16_t num_queues;           // 队列数
    uint16_t queue_size;           // 每队列深度
    uint32_t max_discard_sectors;  // 最大 discard 扇区数
    uint32_t max_write_zeroes_sectors;
    // ... 更多可调参数
};
```

#### VirtIOBlockReq 请求结构

```c
struct VirtIOBlockReq {
    VirtQueueElement elem;         // virtqueue 元素（SG 列表）
    VirtQueue *vq;                 // 所属队列
    struct virtio_blk_outhdr out;  // 请求头（type, ioprio, sector）
    QEMUIOVector qiov;             // I/O 向量
    struct VirtIOBlockReq *next;   // 完成链表
    struct VirtIOBlockReq *mr_next; // 合并链表
    VirtIOBlock *dev;              // 所属设备
};
```

### 13. 设备 Realize 流程

`virtio_blk_device_realize()`（virtio-blk.c:1723-1847）：

```
virtio_blk_device_realize(dev, errp)
  │
  ├── 1. 验证配置
  │     ├── 检查 drive= 属性已设置                    [1732-1739]
  │     ├── 验证 num_queues > 0                       [1740-1744]
  │     └── 验证 queue_size 合法                      [1746-1758]
  │
  ├── 2. 配置后端
  │     ├── blkconf_serial(&conf->conf, &conf->serial) [1760]
  │     ├── blkconf_apply_backend_options()            [1762-1764]
  │     ├── blkconf_geometry() → 读取磁盘几何          [1766]
  │     └── blkconf_blocksizes() → 读取块大小          [1768-1772]
  │
  ├── 3. 初始化 virtio
  │     └── virtio_init(vdev, VIRTIO_ID_BLOCK, config_size) [1801-1803]
  │
  ├── 4. 连接后端
  │     ├── s->blk = conf->conf.blk                   [1807]
  │     └── s->sector_mask = (s->conf.conf.logical_block_size / 512) - 1 [1809]
  │
  ├── 5. 创建 VirtQueue
  │     └── for i in 0..num_queues:                    [1811-1813]
  │           virtio_add_queue(vdev, queue_size, virtio_blk_handle_output)
  │
  ├── 6. 设置 IOThread 映射
  │     └── virtio_blk_vq_aio_context_init()           [1821-1829]
  │
  └── 7. 注册块操作回调
        └── blk_set_dev_ops(s->blk, &virtio_block_ops) [1838-1840]
```

### 14. 请求处理管线

这是 virtio-blk 最核心的数据路径：

```
Guest 驱动写入 QueueNotify
  │
  ▼
virtio_blk_handle_output() [virtio-blk.c:1044-1059]
  │
  ├── 延迟启动 ioeventfd（首次 kick 时）
  │
  └── virtio_blk_handle_vq() [virtio-blk.c:1011-1042]
        │
        ├── while (true):
        │     │
        │     ├── virtio_blk_get_request() [virtio-blk.c:170-177]
        │     │     └── virtqueue_pop(vq)
        │     │           └── 取出一个请求的描述符链
        │     │               构建 VirtQueueElement（in/out SG）
        │     │
        │     ├── virtio_blk_handle_request() [virtio-blk.c:821-1008]
        │     │     │
        │     │     ├── 解析请求头
        │     │     │     └── 从 out SG 拷贝 virtio_blk_outhdr
        │     │     │         ├── type:   IN/OUT/FLUSH/GET_ID/...
        │     │     │         ├── ioprio: I/O 优先级
        │     │     │         └── sector: 起始扇区号
        │     │     │
        │     │     └── 根据 type 分发（见下一节）
        │     │
        │     └── 如果队列空 → 退出循环
        │
        └── defer_call_end() → 批量完成通知
```

#### 读写请求详细路径（VIRTIO_BLK_T_IN / T_OUT）

```
virtio_blk_handle_request() — type == IN 或 OUT:
  │
  ├── 1. 计算 sector_num = virtio_ldq_p(&req->out.sector)     [866]
  │
  ├── 2. 构建 QEMUIOVector
  │     └── 将 in_sg（读）或 out_sg（写）映射到 QIOV           [870-880]
  │
  ├── 3. 范围检查
  │     └── sector_num + nb_sectors > 磁盘容量？→ 返回错误      [882-890]
  │
  ├── 4. 记账
  │     └── block_acct_start()                                 [892]
  │
  ├── 5. 尝试合并请求
  │     └── 如果启用合并 && 可与前一请求合并 → 链入 mr_next      [894-900]
  │
  └── 6. 提交 I/O
        └── submit_requests() [virtio-blk.c:215-266]
              │
              ├── 写请求: blk_aio_pwritev(s->blk, sector_num * 512,
              │               &req->qiov, 0, virtio_blk_rw_complete, req)
              │                                                [257-260]
              │
              └── 读请求: blk_aio_preadv(s->blk, sector_num * 512,
                              &req->qiov, 0, virtio_blk_rw_complete, req)
                                                               [261-264]
```

### 15. 请求类型详解

virtio-blk 支持以下请求类型（`virtio_blk.h:162-205`）：

| 类型 | 值 | 处理位置 | 说明 |
|-----|---|---------|------|
| `VIRTIO_BLK_T_IN` | 0 | 821-903 | 读数据 |
| `VIRTIO_BLK_T_OUT` | 1 | 821-903 | 写数据 |
| `VIRTIO_BLK_T_FLUSH` | 4 | 904-906, 337-351 | 刷新缓存 |
| `VIRTIO_BLK_T_GET_ID` | 8 | 928-941 | 获取设备序列号 |
| `VIRTIO_BLK_T_DISCARD` | 11 | 956-991 | 丢弃扇区（trim） |
| `VIRTIO_BLK_T_WRITE_ZEROES` | 13 | 956-991 | 写零 |
| `VIRTIO_BLK_T_ZONE_REPORT` | 16 | 907-909, 640-691 | Zoned 报告 |
| `VIRTIO_BLK_T_ZONE_OPEN` | 18 | 910-924, 709-749 | 打开 Zone |
| `VIRTIO_BLK_T_ZONE_CLOSE` | 20 | 910-924 | 关闭 Zone |
| `VIRTIO_BLK_T_ZONE_FINISH` | 22 | 910-924 | 完成 Zone |
| `VIRTIO_BLK_T_ZONE_RESET` | 24 | 910-924 | 重置 Zone |
| `VIRTIO_BLK_T_ZONE_APPEND` | 26 | 943-950, 783-819 | Zone 追加写 |
| `VIRTIO_BLK_T_SCSI_CMD` | 2 | 925-927 | SCSI 命令（已弃用） |

**安全修复**：commit 4913ae36f9 修复了 zone report 中的缓冲区越界（CVE-2026-5761），攻击者可通过精心构造的 zone report 请求触发 QEMU 进程的内存溢出。

### 16. I/O 完成路径

所有异步 I/O 请求通过回调完成：

```
块层 I/O 完成
  │
  ▼
virtio_blk_rw_complete() [virtio-blk.c:98-136]    ← 读/写
virtio_blk_flush_complete() [virtio-blk.c:138-150] ← flush
virtio_blk_discard_write_zeroes_complete() [152-168] ← discard/写零
  │
  ├── 记录 I/O 状态（成功/失败）
  ├── block_acct_done/failed()     — 记账
  │
  └── virtio_blk_req_complete() [virtio-blk.c:57-69]
        │
        ├── 设置 in_hdr.status
        │     ├── 成功: VIRTIO_BLK_S_OK (0)
        │     ├── I/O 错误: VIRTIO_BLK_S_IOERR (1)
        │     └── 不支持: VIRTIO_BLK_S_UNSUPP (2)
        │
        ├── virtqueue_push(vq, &req->elem, len)
        │     └── 填充 used ring
        │
        └── virtio_notify(vdev, vq)
              └── 触发中断/MSI-X 通知 Guest
```

### 17. 多队列与 IOThread

#### 多队列架构

```
Guest:
  vCPU 0 ──► VirtQueue 0 ──► IOThread A ──► blk_aio_preadv()
  vCPU 1 ──► VirtQueue 1 ──► IOThread B ──► blk_aio_preadv()
  vCPU 2 ──► VirtQueue 2 ──► IOThread A ──► blk_aio_preadv()  (共享)
  vCPU 3 ──► VirtQueue 3 ──► IOThread B ──► blk_aio_preadv()
```

**配置**：
- `num-queues=N`：创建 N 个 VirtQueue
- `iothread=iot0`：所有队列共享一个 IOThread
- `iothread-vq-mapping`：精细的队列→IOThread 映射

**初始化**（virtio-blk.c:1460-1513）：

```c
virtio_blk_vq_aio_context_init(s)
  ├── 为每个 VirtQueue 分配 AioContext
  ├── 根据 iothread-vq-mapping 或 iothread 设置
  └── 如果无 IOThread → 使用 QEMU 主循环 AioContext
```

**ioeventfd 数据面**（virtio-blk.c:1536-1632）：

当使用 KVM 加速时，ioeventfd 允许 Guest 的 QueueNotify 写操作直接由 KVM 内核处理，避免 VM Exit 到 QEMU 用户空间，显著降低 I/O 延迟。

**git 改进**：
- commit b50629c335：提取 `iothread-vq-mapping.h` 为通用 API
- commit 2fa67a7b1d：清理 iothread_vq_mapping 函数

### 18. BlockBackend 集成

virtio-blk 通过 QEMU 块层（Block Layer）访问实际存储：

```
VirtIOBlock
  │
  └── s->blk (BlockBackend)
        │
        └── BlockDriverState (BDS) 链
              │
              ├── 格式层: qcow2 / raw / vmdk
              │     └── bdrv_co_preadv() / bdrv_co_pwritev()
              │
              └── 协议层: file / nbd / iscsi / nvme
                    └── 实际文件 I/O / 网络 I/O
```

**关键 API 调用**：

| 操作 | 函数 | 位置 |
|------|------|------|
| 异步读 | `blk_aio_preadv()` | virtio-blk.c:261 |
| 异步写 | `blk_aio_pwritev()` | virtio-blk.c:257 |
| 异步 flush | `blk_aio_flush()` | virtio-blk.c:350 |
| 异步 discard | `blk_aio_pdiscard()` | virtio-blk.c:427 |
| 异步写零 | `blk_aio_pwrite_zeroes()` | virtio-blk.c:441 |

### 19. 特性协商

`virtio_blk_get_features()`（virtio-blk.c:1270-1300）：

| 特性 | 条件 | 说明 |
|------|------|------|
| `VIRTIO_BLK_F_SEG_MAX` | 总是 | 最大 SG 段数 |
| `VIRTIO_BLK_F_GEOMETRY` | 总是 | 磁盘几何信息 |
| `VIRTIO_BLK_F_TOPOLOGY` | 总是 | 块大小/对齐 |
| `VIRTIO_BLK_F_BLK_SIZE` | 总是 | 逻辑块大小 |
| `VIRTIO_BLK_F_WCE` | 后端启用写缓存 | 写缓存使能 |
| `VIRTIO_BLK_F_RO` | 后端只读 | 只读设备 |
| `VIRTIO_BLK_F_MQ` | num_queues > 1 | 多队列 |
| `VIRTIO_BLK_F_DISCARD` | 配置启用 | Discard/TRIM |
| `VIRTIO_BLK_F_WRITE_ZEROES` | 配置启用 | 写零 |
| `VIRTIO_BLK_F_ZONED` | Zoned 后端 | 分区存储 |

### 20. CLI 使用与配置

#### 基本用法

```bash
# qcow2 镜像 + virtio-blk-pci
qemu-system-aarch64 -M virt \
  -drive file=disk.qcow2,id=hd0,format=qcow2,if=none \
  -device virtio-blk-pci,drive=hd0

# raw 镜像 + virtio-blk-device（MMIO 传输）
qemu-system-aarch64 -M virt \
  -drive file=disk.raw,id=hd0,format=raw,if=none \
  -device virtio-blk-device,drive=hd0
```

#### 高级配置

```bash
# 多队列 + IOThread
qemu-system-aarch64 -M virt \
  -object iothread,id=iot0 \
  -drive file=disk.qcow2,id=hd0,format=qcow2,if=none,aio=io_uring \
  -device virtio-blk-pci,drive=hd0,num-queues=4,iothread=iot0

# 只读 CD-ROM
-drive file=install.iso,id=cd0,format=raw,if=none,readonly=on \
-device virtio-blk-pci,drive=cd0
```

#### 连接关系

```
-drive file=X,id=hd0  →  创建 BlockBackend "hd0"
-device virtio-blk-pci,drive=hd0  →  DEFINE_PROP_DRIVE("drive",...,blk)
                                      realize 时: s->blk = conf->conf.blk
```

---

## 第三部分：virtio-net 网络设备仿真

### 21. virtio-net 概述

virtio-net 是 QEMU 中性能最高的虚拟网卡，支持多队列、TSO/GSO 卸载、RSS 以及 vhost 加速。它是生产环境中最常用的虚拟网络设备。

**QOM 类型层级**：

```
Object → DeviceState → VirtIODevice → VirtIONet

传输层:
  VirtIONetPCI  (virtio-net-pci)    — PCI 传输
  VirtIOMMIOProxy (virtio-net-device) — MMIO 传输
```

### 22. VirtIONet 结构体

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

### 23. 设备 Realize 流程

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

### 24. RX 接收路径

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

### 25. TX 发送路径

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

### 26. 控制队列

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

### 27. 多队列与 RSS

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

### 28. 特性协商

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

### 29. 网络后端与 vhost 集成

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

### 30. CLI 使用与配置

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

## 附录

### A. 三种设备对比

| 维度 | PL011 UART | virtio-blk | virtio-net |
|------|-----------|-----------|-----------|
| **接口类型** | 寄存器级 MMIO | virtio VirtQueue | virtio VirtQueue |
| **传输层** | SysBus（固定 MMIO） | PCI / MMIO | PCI / MMIO |
| **数据流** | 单字节同步 | 块 I/O 异步 | 包级异步 |
| **中断** | Level（IMSC/RIS/MIS） | MSI-X / Level | MSI-X / Level |
| **FIFO/队列** | 16 字节 RX FIFO | 多 VirtQueue | 多队列对 |
| **后端** | CharBackend | BlockBackend | NetClientState |
| **加速** | 无 | ioeventfd + IOThread | vhost-net + eBPF RSS |
| **多路复用** | 无 | 多队列 | 多队列对 |
| **性能** | 低（调试用） | 高 | 高 |

### B. 关键源文件索引

| 文件 | 内容 | 行数级 |
|------|------|--------|
| hw/char/pl011.c | PL011 UART 完整实现 | ~730 行 |
| include/hw/char/pl011.h | PL011 结构定义 | ~65 行 |
| hw/block/virtio-blk.c | virtio-blk 完整实现 | ~1850 行 |
| include/hw/virtio/virtio-blk.h | virtio-blk 结构定义 | ~110 行 |
| hw/net/virtio-net.c | virtio-net 完整实现 | ~4050 行 |
| include/hw/virtio/virtio-net.h | virtio-net 结构定义 | ~235 行 |
| hw/arm/virt.c | virt 机器设备创建 | ~3000 行 |

### C. 关键 Git 提交

| Commit | 描述 | 影响设备 |
|--------|------|---------|
| 907b8d5635 | 只记录一次 disabled UART 警告 | PL011 |
| 3e0f118f82 | 修复 RX FIFO 深度计算 | PL011 |
| c715efe284 | 标记 PL011 为小端实现 | PL011 |
| 4913ae36f9 | 修复 zone report 缓冲区溢出 (CVE) | virtio-blk |
| b50629c335 | 提取 iothread-vq-mapping API | virtio-blk |
| 1e9181dc52 | 统一 virtio_notify_irqfd/notify | virtio-blk/net |
| 1c79ab6937 | 默认启用 UDP tunnel GSO | virtio-net |
| 3a7741c3bd | 实现扩展特性支持 | virtio-net |
| 9b960bbc13 | 移除 mtu_bypass_backend 字段 | virtio-net |

### D. 推荐深入方向

1. **e1000e 网卡仿真** — 对比 virtio-net 与传统设备仿真的差异
2. **NVMe 控制器仿真** — 现代存储接口仿真
3. **vhost-user 后端开发** — 自定义用户空间 virtio 后端
4. **QEMU 块层 I/O 路径** — 从 BlockBackend 到实际文件的完整链路
5. **TAP 网络后端** — 与主机内核网络栈的交互细节
