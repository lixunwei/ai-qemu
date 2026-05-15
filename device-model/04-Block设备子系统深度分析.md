# Block 设备子系统深度分析

> QEMU 版本：11.0.50  
> 分析日期：2025年  
> 范围：Block 设备框架、qcow2 格式驱动、Block 作业与高级特性  
> 相关文档：[02-块层IO路径深度分析](02-块层IO路径深度分析.md) | [01-关键设备仿真分析-UART-磁盘-网卡](01-关键设备仿真分析-UART-磁盘-网卡.md) | [../architecture/03-Machine建立流程深度分析](../architecture/03-Machine建立流程深度分析.md)

---

## 目录

- [第一部分：Block 层架构总览](#第一部分block-层架构总览)
  - [§1 Block 层分层架构](#1-block-层分层架构)
  - [§2 核心数据结构关系](#2-核心数据结构关系)
- [第二部分：Block 设备创建与生命周期](#第二部分block-设备创建与生命周期)
  - [§3 -drive 命令行解析](#3--drive-命令行解析)
  - [§4 -blockdev 命令行解析](#4--blockdev-命令行解析)
  - [§5 -drive vs -blockdev 对比](#5--drive-vs--blockdev-对比)
  - [§6 BlockBackend 生命周期](#6-blockbackend-生命周期)
  - [§7 BlockDriverState 打开流程](#7-blockdriverstate-打开流程)
  - [§8 BDS 图结构与节点关系](#8-bds-图结构与节点关系)
  - [§9 权限模型](#9-权限模型)
  - [§10 与 virtio-blk 的连接](#10-与-virtio-blk-的连接)
- [第三部分：qcow2 格式驱动](#第三部分qcow2-格式驱动)
  - [§11 qcow2 磁盘布局](#11-qcow2-磁盘布局)
  - [§12 L1/L2 表地址映射](#12-l1l2-表地址映射)
  - [§13 引用计数 (Refcount)](#13-引用计数-refcount)
  - [§14 qcow2 驱动注册与打开](#14-qcow2-驱动注册与打开)
  - [§15 qcow2 读路径](#15-qcow2-读路径)
  - [§16 qcow2 写路径](#16-qcow2-写路径)
  - [§17 快照机制](#17-快照机制)
  - [§18 L2/Refcount 缓存](#18-l2refcount-缓存)
  - [§19 性能特性](#19-性能特性)
- [第四部分：Block 作业框架](#第四部分block-作业框架)
  - [§20 Job 基类与生命周期](#20-job-基类与生命周期)
  - [§21 BlockJob 子类](#21-blockjob-子类)
  - [§22 Mirror 作业](#22-mirror-作业)
  - [§23 Commit 作业](#23-commit-作业)
  - [§24 Stream 作业](#24-stream-作业)
  - [§25 Backup 作业](#25-backup-作业)
- [第五部分：高级 Block 特性](#第五部分高级-block-特性)
  - [§26 脏位图 (Dirty Bitmap)](#26-脏位图-dirty-bitmap)
  - [§27 Block 限流 (Throttle)](#27-block-限流-throttle)
  - [§28 Block 过滤器](#28-block-过滤器)
- [第六部分：virtio-blk 设备仿真](#第六部分virtio-blk-设备仿真)
  - [§29 virtio-blk 概述](#29-virtio-blk-概述)
  - [§30 VirtIOBlock 结构体](#30-virtioblock-结构体)
  - [§31 设备 Realize 流程](#31-设备-realize-流程)
  - [§32 请求处理管线](#32-请求处理管线)
  - [§33 请求类型详解](#33-请求类型详解)
  - [§34 I/O 完成路径](#34-io-完成路径)
  - [§35 多队列与 IOThread](#35-多队列与-iothread)
  - [§36 BlockBackend 集成](#36-blockbackend-集成)
  - [§37 特性协商](#37-特性协商)
  - [§38 CLI 使用与配置](#38-cli-使用与配置)
- [附录](#附录)
  - [附录A：Block 驱动一览](#附录ablock-驱动一览)
  - [附录B：关键源文件索引](#附录b关键源文件索引)
  - [附录C：关键 Git 提交](#附录c关键-git-提交)

---

## 第一部分：Block 层架构总览

### §1 Block 层分层架构

```
┌──────────────────────────────────────────────────────────┐
│                    Guest VM                               │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│   │ 文件系统  │    │ 块设备驱动│    │ 分区管理  │           │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘           │
└────────┼───────────────┼───────────────┼─────────────────┘
         │ MMIO/virtqueue│               │
┌────────┼───────────────┼───────────────┼─────────────────┐
│ QEMU   │               │               │                 │
│   ┌────▼───────────────▼───────────────▼─────┐           │
│   │         设备模拟层 (Device)                │           │
│   │   virtio-blk / IDE / NVMe / SCSI         │           │
│   └────────────────┬─────────────────────────┘           │
│                    │ BlockBackend API                     │
│   ┌────────────────▼─────────────────────────┐           │
│   │         BlockBackend (BB)                 │           │
│   │    "一块磁盘" 的抽象表示                   │           │
│   │    权限管理、AioContext 绑定               │           │
│   └────────────────┬─────────────────────────┘           │
│                    │ BdrvChild (root)                     │
│   ┌────────────────▼─────────────────────────┐           │
│   │    格式驱动层 (Format Driver)             │           │
│   │    qcow2 / qed / vmdk / vdi / raw        │           │
│   │    L1/L2 映射、COW、快照、压缩            │           │
│   └────────────────┬─────────────────────────┘           │
│                    │ BdrvChild (file/backing)             │
│   ┌────────────────▼─────────────────────────┐           │
│   │    过滤层 (Filter Driver, 可选)           │           │
│   │    throttle / copy-before-write /         │           │
│   │    blkdebug / blkverify                  │           │
│   └────────────────┬─────────────────────────┘           │
│                    │ BdrvChild (file)                     │
│   ┌────────────────▼─────────────────────────┐           │
│   │    协议驱动层 (Protocol Driver)           │           │
│   │    file-posix / nbd / gluster /           │           │
│   │    iscsi / rbd / ssh / curl              │           │
│   └────────────────┬─────────────────────────┘           │
│                    │                                     │
└────────────────────┼─────────────────────────────────────┘
                     │ 系统调用 / 网络协议
              ┌──────▼──────┐
              │  Host 存储   │
              │  磁盘/NFS/iSCSI/Ceph
              └─────────────┘
```

### §2 核心数据结构关系

```
                       ┌──────────────┐
                       │  virtio-blk  │
                       │   Device     │
                       └──────┬───────┘
                              │ conf.blk
                       ┌──────▼───────┐
                       │ BlockBackend │ "一块磁盘"
                       │  (BB)        │
                       │ ┌──────────┐ │
                       │ │perm/share│ │ 权限
                       │ │ctx (AIO) │ │ AioContext
                       │ │root ─────┼─┤
                       │ └──────────┘ │
                       └──────┬───────┘
                              │ BdrvChild (root)
                       ┌──────▼───────┐
                       │  BDS: qcow2  │ 格式驱动
                       │  (overlay)   │
                       │ ┌──────────┐ │
                       │ │L1/L2 表  │ │
                       │ │Refcount  │ │
                       │ │Cache     │ │
                       │ └──────────┘ │
                       ├──────┬───────┤
                  file │      │       │ backing
            ┌──────────▼──┐ ┌─▼──────────┐
            │ BDS: file-  │ │ BDS: qcow2 │ 基础镜像
            │ posix       │ │ (base)     │
            │ (overlay.   │ └──────┬─────┘
            │  qcow2)     │        │ file
            └─────────────┘ ┌──────▼─────┐
                            │ BDS: file- │
                            │ posix      │
                            │ (base.     │
                            │  qcow2)    │
                            └────────────┘
```

---

## 第二部分：Block 设备创建与生命周期

### §3 -drive 命令行解析

**源文件**：`vl.c:2922-2970`，`blockdev.c:777-1038`

```
命令行: -drive file=disk.qcow2,format=qcow2,if=none,id=hd0

解析流程:
  vl.c: QEMU_OPTION_drive                    [vl.c:2922]
    → qemu_opts_parse_noisily(qemu_find_opts("drive"), optarg)
    → 存入 drive opts 队列

  后续初始化:
    drive_init_func(opts)
      └─ drive_new(opts, ...)                [blockdev.c:777]
           │
           ├─ 解析 if= 参数                   [blockdev.c:893]
           │    if_name[] = {"none","ide","scsi","floppy",
           │                 "pflash","mtd","sd","virtio","xen"}
           │                                  [blockdev.c:77-86]
           │
           ├─ 如果 if=virtio:
           │    自动创建 virtio-blk-device      [blockdev.c:965-982]
           │
           ├─ blockdev_init(opts)              创建 BlockBackend
           │    ├─ blk_new_open(filename, format, ...)
           │    │    [block-backend.c:423]
           │    │    ├─ bdrv_open() → 打开 BDS
           │    │    └─ blk_new() + blk_insert_bs()
           │    │
           │    └─ 返回 BlockBackend*
           │
           ├─ dinfo = g_new0(DriveInfo, 1)
           │    dinfo->type = if_type
           │    dinfo->bus = bus
           │    dinfo->unit = unit
           │
           └─ blk_set_legacy_dinfo(blk, dinfo)
                                              [block-backend.c:812]
```

### §4 -blockdev 命令行解析

**源文件**：`vl.c:2949-2963`，`blockdev.c:1041-1059`

```
命令行: -blockdev driver=qcow2,node-name=hd0,\
              file.driver=file,file.filename=disk.qcow2

解析流程:
  vl.c: QEMU_OPTION_blockdev                [vl.c:2949]
    → 解析为 QAPI BlockdevOptions
    → 加入 blockdev 队列

  后续:
    qmp_blockdev_add(options)                [blockdev.c:1041+]
      └─ bds_tree_init(qdict)               [blockdev.c:663]
           └─ bdrv_open(NULL, NULL, options, ...)
                → 创建 BDS 图 (无 BlockBackend)
                → 后续由 -device 引用 node-name
```

### §5 -drive vs -blockdev 对比

| 特性 | -drive | -blockdev |
|------|--------|----------|
| 创建 BlockBackend | ✓ 自动创建 | ✗ 仅创建 BDS 节点 |
| 创建 DriveInfo | ✓ 有 legacy info | ✗ 无 |
| 自动创建设备 | if=virtio 自动创建 | 不自动，需 -device |
| 格式探测 | 支持自动探测 | 必须显式指定 driver= |
| 节点命名 | id= (backend 名) | node-name= (图节点名) |
| 推荐使用 | 简单场景 | 复杂 BDS 图、热插拔 |
| 内部实现 | blockdev_init() | bds_tree_init() |

### §6 BlockBackend 生命周期

**源文件**：`block-backend.c:43-95, 355-520`

```c
// block-backend.c:43-95 (简化)
struct BlockBackend {
    char *name;                    // 名称 (如 "hd0")
    int refcnt;                    // 引用计数
    BdrvChild *root;               // 指向 BDS 图的根 (BdrvChild)
    AioContext *ctx;               // 绑定的 AIO 上下文
    DriveInfo *legacy_dinfo;       // 旧版 DriveInfo (仅 -drive)

    // 权限
    uint64_t perm;                 // 本端持有的权限
    uint64_t shared_perm;          // 共享给其他用户的权限

    // 状态
    bool allow_aio_context_change;
    bool allow_write_beyond_eof;
    // ...
};
```

生命周期：

```
创建:
  blk_new(ctx, perm, shared_perm)           [block-backend.c:355]
    ├─ 分配 BlockBackend
    ├─ 绑定 AioContext
    └─ 设置权限

  blk_new_open(filename, ref, options, ...)  [block-backend.c:423]
    ├─ bdrv_open() → 打开 BDS
    ├─ blk_new() → 创建 BB
    └─ blk_insert_bs(blk, bs) → 关联

挂载 BDS:
  blk_insert_bs(blk, bs)                    [block-backend.c:900]
    ├─ blk->root = bdrv_root_attach_child(bs, ...)
    │    // 创建 BdrvChild，建立父子关系
    └─ 申请权限

卸载 BDS:
  blk_remove_bs(blk)                        [block-backend.c:859]
    ├─ bdrv_root_unref_child(blk->root)
    └─ blk->root = NULL

销毁:
  blk_unref(blk)                            [block-backend.c:513]
    └─ refcnt-- == 0?
         → blk_delete()                     [block-backend.c:478]
              ├─ blk_remove_bs()
              ├─ 释放 DriveInfo
              └─ g_free(blk)
```

### §7 BlockDriverState 打开流程

**源文件**：`block.c`

```
bdrv_open(filename, reference, options, flags, errp)
  └─ bdrv_open_inherit(filename, ref, options, flags, parent, child_class, ...)
       │
       ├─ 1. 查找驱动
       │    bdrv_find_format(driver_name)
       │    // 或自动探测: bdrv_find_whitelisted_format()
       │
       ├─ 2. 创建 BDS
       │    bdrv_new() → 分配 BlockDriverState
       │
       ├─ 3. 打开协议层
       │    bdrv_open_common(bs, options, flags)     [block.c:1877]
       │    └─ bdrv_open_driver(bs, drv, ...)        [block.c:1659]
       │         └─ drv->bdrv_open(bs, options, flags)
       │              // file-posix: raw_open() → open(filename)
       │              // nbd: nbd_open() → TCP 连接
       │
       ├─ 4. 打开格式层 (如果是格式驱动)
       │    bdrv_open_driver(bs, drv, ...)
       │    └─ drv->bdrv_open(bs, ...)
       │         // qcow2: qcow2_open() → 读头部/L1 表/Refcount
       │
       ├─ 5. 打开 backing 文件 (如果有)
       │    bdrv_open_backing_file(bs)
       │    └─ bdrv_open(backing_filename, ...)
       │         // 递归打开整个 backing 链
       │
       └─ 6. 设置权限
            bdrv_node_refresh_perm(bs)
```

### §8 BDS 图结构与节点关系

```
BDS 节点通过 BdrvChild 连接:

  BdrvChild 结构:
    ├─ bs          → 子 BDS (被引用的节点)
    ├─ opaque      → 父对象 (BB 或 BDS)
    ├─ klass       → 子节点角色 (child_of_bds / child_root)
    ├─ name        → 角色名 ("file", "backing", "filter")
    ├─ perm        → 持有的权限
    └─ shared_perm → 共享的权限

典型 BDS 图:

  BlockBackend "hd0"
       │
       │ root (child_root)
       ▼
  ┌─────────────┐ file    ┌──────────────┐
  │ qcow2       ├────────►│ file-posix   │ overlay.qcow2
  │ (overlay)   │         └──────────────┘
  │             │
  │             │ backing  ┌─────────────┐ file    ┌──────────────┐
  │             ├─────────►│ qcow2       ├────────►│ file-posix   │ base.qcow2
  │             │          │ (base)      │         └──────────────┘
  └─────────────┘          └─────────────┘

带过滤器:

  BlockBackend
       │ root
       ▼
  ┌──────────┐ file    ┌──────────┐ file    ┌──────────────┐
  │ throttle ├────────►│ qcow2    ├────────►│ file-posix   │
  │ (filter) │         │ (format) │         │ (protocol)   │
  └──────────┘         └──────────┘         └──────────────┘
```

### §9 权限模型

**源文件**：`block.c:2283-2921`

```
权限位:                                  含义
  BLK_PERM_READ             = 0x01    // 读取数据
  BLK_PERM_WRITE            = 0x02    // 写入数据
  BLK_PERM_CONSISTENT_READ  = 0x04    // 读取一致性 (不允许并发写)
  BLK_PERM_WRITE_UNCHANGED  = 0x08    // 写入不改变内容 (如 flush)
  BLK_PERM_RESIZE           = 0x10    // 修改大小
  BLK_PERM_GRAPH_MOD        = 0x20    // 修改 BDS 图结构

权限传播:
  BB 设置顶级权限 → 通过 BdrvChild 传播到子节点

  blk_insert_bs(blk, bs)
    → bdrv_root_attach_child(bs, ..., perm, shared)
       → bdrv_node_refresh_perm(bs)
          → 遍历所有 parent 收集 cumulative perm
          → 验证不冲突
          → 递归传播到子节点

冲突检查:
  如果两个 parent 的权限冲突:
  例: parent1 要 WRITE, parent2 要 CONSISTENT_READ
  → 冲突! 因为 WRITE 与 CONSISTENT_READ 不兼容
  → bdrv_check_perm() 返回错误
```

### §10 与 virtio-blk 的连接

**源文件**：`virtio-blk.c`，`include/hw/block/block.h`

```
属性定义:
  DEFINE_BLOCK_PROPERTIES(VirtIOBlock, conf.conf)
    → DEFINE_PROP_DRIVE("drive", VirtIOBlock, conf.conf.blk)
    // "drive" 属性类型为 BlockBackend*

命令行: -device virtio-blk-device,drive=hd0

连接过程:
  1. QOM 属性解析: "drive=hd0"
     → 在全局 BlockBackend 列表中查找 name="hd0"
     → 设置 conf.conf.blk = blk (找到的 BlockBackend)

  2. Realize 时:
     virtio_blk_device_realize()          [virtio-blk.c:1723]
       ├─ s->blk = conf->conf.blk         绑定 BlockBackend
       ├─ 检查 blk 非空                    [1732-1738]
       ├─ blk_set_perm(blk, READ|WRITE, ...)  设置权限
       ├─ virtio_init(vdev, VIRTIO_ID_BLOCK, ...)
       └─ 创建 virtqueue

  3. I/O 路径:
     Guest 写请求 → virtqueue → virtio_blk_handle_vq()
       → blk_aio_pwritev(s->blk, ...)     通过 BB 下发
       → qcow2 / file-posix 层处理
```

---

## 第三部分：qcow2 格式驱动

### §11 qcow2 磁盘布局

**源文件**：`qcow2.h:155-183`

```
qcow2 文件物理布局:

  偏移 0x0000_0000  ┌───────────────────────────┐
                    │ QCowHeader (104+ bytes)    │
                    │  magic = 0x514649FB        │
                    │  version = 3               │
                    │  cluster_bits = 16 (64KB)  │
                    │  size (虚拟磁盘大小)        │
                    │  l1_size, l1_table_offset  │
                    │  refcount_table_offset     │
                    │  snapshots_offset          │
                    │  feature bits              │
                    └───────────────────────────┘

  L1 Table          ┌───────────────────────────┐
                    │ L1 Entry[0] → L2 偏移     │
                    │ L1 Entry[1] → L2 偏移     │
                    │ ...                        │
                    └───────────────────────────┘

  L2 Tables         ┌───────────────────────────┐
                    │ L2 Entry[0] → 数据簇偏移   │
                    │ L2 Entry[1] → 数据簇偏移   │
                    │ ...                        │
                    └───────────────────────────┘

  Refcount Table    ┌───────────────────────────┐
                    │ RT Entry[0] → RB 偏移     │
                    │ RT Entry[1] → RB 偏移     │
                    └───────────────────────────┘

  Refcount Blocks   ┌───────────────────────────┐
                    │ 每个簇的引用计数           │
                    └───────────────────────────┘

  Data Clusters     ┌───────────────────────────┐
                    │ 实际数据 (64KB per cluster)│
                    │ ...                        │
                    └───────────────────────────┘

  Snapshot Table    ┌───────────────────────────┐
                    │ Snapshot Headers           │
                    │ (各快照的 L1 表副本)        │
                    └───────────────────────────┘
```

### §12 L1/L2 表地址映射

**源文件**：`qcow2.h:82-107`，`qcow2-cluster.c:586-743`

```
虚拟地址 → 物理偏移 的两级映射:

  虚拟偏移: 0x1234_5678 (cluster_bits=16, cluster_size=64KB)
  ┌──────────────────────────────────────────┐
  │  L1 Index  │  L2 Index   │ Cluster Offset│
  │ bits[31:30]│ bits[29:16] │ bits[15:0]    │
  │    = 0     │    = 0x1234 │   = 0x5678    │
  └──────────────────────────────────────────┘

  查找过程:
  1. L1 Entry = l1_table[l1_index]
     ├─ 为 0 → 未分配 (读返回全零/读 backing)
     └─ 非零 → L2 表的物理偏移

  2. 加载 L2 表 (可能从缓存命中)
     L2 Entry = l2_table[l2_index]
     ├─ 标准 Entry (8 bytes):
     │    bits[63:62] = 类型 (00=标准, 01=压缩, 10=零+分配, 11=零)
     │    bits[55:0]  = 簇的物理偏移 (host offset)
     │
     └─ 扩展 Entry (16 bytes, extended_l2 feature):
          额外 8 bytes 存储子簇 (subcluster) 位图
          每个簇分为 32 个子簇，每个 2KB

  3. 最终物理偏移 = host_offset + cluster_offset

qcow2_get_host_offset():                [qcow2-cluster.c:586]
  ├─ l1_index = offset >> (cluster_bits + l2_bits)
  ├─ l2_offset = l1_table[l1_index]
  ├─ 加载 L2 表 (缓存查找)
  ├─ l2_entry = l2_table[l2_index]
  ├─ 解码 entry 类型:
  │    QCOW2_SUBCLUSTER_UNALLOCATED  → 未分配
  │    QCOW2_SUBCLUSTER_ZERO_PLAIN   → 全零 (无物理簇)
  │    QCOW2_SUBCLUSTER_ZERO_ALLOC   → 全零 (有物理簇)
  │    QCOW2_SUBCLUSTER_NORMAL       → 正常数据
  │    QCOW2_SUBCLUSTER_COMPRESSED   → 压缩数据
  └─ 返回 host_offset, 类型, 连续簇数
```

### §13 引用计数 (Refcount)

**源文件**：`qcow2-refcount.c:101-930`

```
引用计数管理每个簇的使用状态:

  Refcount Table → Refcount Block → 每簇计数

  ┌──────────────┐
  │ Refcount     │  RT Entry = Refcount Block 偏移
  │ Table        │  (每项 8 bytes)
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │ Refcount     │  每项 refcount_bits (默认 16 bits)
  │ Block        │  值 = 该簇的引用次数
  └──────────────┘

引用计数值含义:
  0  → 空闲簇 (可分配)
  1  → 正常使用
  2+ → 被快照共享 (COW 时需要复制)

操作:
  分配簇:   refcount 0 → 1     (qcow2_alloc_clusters)
  释放簇:   refcount 1 → 0     (qcow2_free_clusters)
  快照创建: refcount +1         (共享簇)
  快照删除: refcount -1         (如果减到0则释放)

  update_refcount()              [qcow2-refcount.c:811]
    ├─ 计算簇所在的 Refcount Block
    ├─ 加载 Refcount Block (可能从缓存)
    ├─ 更新计数值
    ├─ 标记 Refcount Block 为脏
    └─ 如果 refcount=0 → 加入 discard 队列
```

**关键引用计数函数**（qcow2-refcount.c）：

| 操作 | 函数 | 位置 |
|------|------|------|
| 初始化 | `qcow2_refcount_init()` | qcow2-refcount.c:101 |
| 更新引用计数 | `update_refcount()` | qcow2-refcount.c:811 |
| 分配簇 | 搜索 refcount=0 的簇 | qcow2-cluster.c |
| 释放簇 | 设置 refcount=0 | `update_refcount()` |
| 快照增减 | `qcow2_update_snapshot_refcount()` | qcow2-snapshot.c |

### §14 qcow2 驱动注册与打开

**源文件**：`qcow2.c:1403-1710, 6294-6366`

```
驱动注册:
  bdrv_qcow2 = {
      .format_name         = "qcow2",
      .bdrv_open           = qcow2_open,
      .bdrv_close          = qcow2_close,
      .bdrv_co_preadv_part = qcow2_co_preadv_part,
      .bdrv_co_pwritev_part= qcow2_co_pwritev_part,
      .bdrv_co_create_opts = qcow2_co_create_opts,
      .bdrv_snapshot_create= qcow2_snapshot_create,
      // ...
  };                                      [qcow2.c:6294]
  bdrv_register(&bdrv_qcow2)              构造函数中注册

打开流程:
  qcow2_open()                            [qcow2.c:2029]
    └─ qcow2_do_open()                    [qcow2.c:1403]
         │
         ├─ 1. 读取并验证文件头
         │    bdrv_pread(bs->file, 0, &header, sizeof(header))
         │    验证 magic, version, cluster_bits
         │
         ├─ 2. 加载 L1 表
         │    s->l1_table = g_malloc(l1_size * L1E_SIZE)
         │    bdrv_pread(bs->file, l1_table_offset, s->l1_table, ...)
         │
         ├─ 3. 初始化引用计数
         │    qcow2_refcount_init(bs)      [qcow2-refcount.c:101]
         │    加载 Refcount Table
         │
         ├─ 4. 创建 L2/Refcount 缓存
         │    s->l2_table_cache = qcow2_cache_create(...)
         │    s->refcount_block_cache = qcow2_cache_create(...)
         │
         ├─ 5. 处理特性标志
         │    lazy_refcounts, extended_l2, compression_type
         │    不兼容标志检查 (如果有未知标志 → 拒绝打开)
         │
         ├─ 6. 打开外部数据文件 (如果有)
         │    bdrv_co_open_child(data_file_name, ...)
         │
         └─ 7. 加载快照表
              qcow2_read_snapshots(bs)
```

### §15 qcow2 读路径

**源文件**：`qcow2.c:2473-2533`，`qcow2-cluster.c:586-743`

```
qcow2_co_preadv_part(bs, offset, bytes, qiov, ...)
  │                                        [qcow2.c:2473]
  │
  └─ while (bytes > 0) {
       │
       ├─ 1. 查找簇映射
       │    qcow2_get_host_offset(bs, offset, &bytes, &host_offset, &type)
       │    // 返回: host 偏移、子簇类型、连续字节数
       │
       ├─ 2. 根据类型处理
       │    │
       │    ├─ UNALLOCATED (未分配):
       │    │    有 backing? → 从 backing 文件读取
       │    │    无 backing  → 返回全零
       │    │
       │    ├─ ZERO_PLAIN/ZERO_ALLOC (全零):
       │    │    → 填充零到 qiov
       │    │
       │    ├─ NORMAL (正常数据):
       │    │    → bdrv_co_preadv_part(s->data_file,
       │    │         host_offset, cur_bytes, qiov, ...)
       │    │    // 从数据文件的物理偏移读取
       │    │
       │    └─ COMPRESSED (压缩):
       │         → 解压缩读取 (整个压缩簇)
       │
       ├─ offset += cur_bytes
       └─ bytes -= cur_bytes
     }
```

**并行读优化**：大 I/O 请求使用 `AioTaskPool` + `QCOW2_MAX_WORKERS` 并行处理多个子簇请求，提高吞吐量。加密簇还需经过 `qcow2_co_preadv_encrypted()` 解密。

### §16 qcow2 写路径

**源文件**：`qcow2.c:2763-2843`，`qcow2-cluster.c:755-815`

```
qcow2_co_pwritev_part(bs, offset, bytes, qiov, ...)
  │                                        [qcow2.c:2763]
  │
  └─ while (bytes > 0) {
       │
       ├─ 1. 分配/查找簇
       │    qcow2_alloc_host_offset(bs, offset, &bytes,
       │                            &host_offset, &l2meta)
       │    │
       │    ├─ 已分配且非共享 (refcount=1):
       │    │    直接使用现有 host_offset
       │    │
       │    ├─ 未分配:
       │    │    qcow2_alloc_clusters() → 找空闲簇
       │    │    → 更新 L2 表项
       │    │    → 更新引用计数
       │    │
       │    └─ 共享 (refcount>1, 快照):
       │         COW: 分配新簇 → 复制旧数据 → 更新 L2
       │
       ├─ 2. COW 处理 (部分簇写入)
       │    merge_cow() → 读旧数据 + 合并新数据
       │                                    [qcow2.c:2537]
       │
       ├─ 3. 写入数据
       │    bdrv_co_pwritev_part(s->data_file,
       │         host_offset, cur_bytes, qiov, ...)
       │
       ├─ 4. 更新 L2 元数据
       │    qcow2_handle_l2meta(bs, &l2meta, true)
       │    → 写入 L2 表项 (host_offset + 标志)
       │
       ├─ offset += cur_bytes
       └─ bytes -= cur_bytes
     }
```


**QCOW_OFLAG_COPIED 语义**：L2 表项中的 `QCOW_OFLAG_COPIED` 标志表示该簇引用计数为 1（独占），可直接覆写。无此标志表示被快照共享（refcount > 1），写入时必须先 COW。`l2_allocate()` 在 L2 表本身也需要分配时，会分配新 L2 簇、复制旧内容、更新 L1 指针并设置 `QCOW_OFLAG_COPIED`。

### §17 快照机制

**源文件**：`qcow2-snapshot.c:637-984`

```
快照创建:
  qcow2_snapshot_create(bs, sn_info)       [qcow2-snapshot.c:637]
    │
    ├─ 1. 刷新所有缓存
    │    qcow2_cache_flush_all(bs)
    │
    ├─ 2. 复制当前 L1 表
    │    new_l1_table = g_malloc(l1_size * L1E_SIZE)
    │    memcpy(new_l1_table, s->l1_table, ...)
    │    // 快照 L1 指向同一组 L2/数据簇
    │                                      [qcow2-snapshot.c:672]
    │
    ├─ 3. 写入快照 L1 表到文件
    │    l1_table_offset = qcow2_alloc_clusters(bs, ...)
    │    bdrv_pwrite(bs->file, l1_table_offset, new_l1_table, ...)
    │
    ├─ 4. 增加所有簇的引用计数
    │    qcow2_update_snapshot_refcount(bs, l1_table_offset, +1)
    │    // 被快照和活动镜像共享的簇 refcount=2
    │                                      [qcow2-snapshot.c:707]
    │
    ├─ 5. 记录快照元数据
    │    sn->l1_table_offset = l1_table_offset
    │    sn->l1_size = l1_size
    │    sn->vm_state_size = ...
    │
    └─ 6. 追加到快照表
         qcow2_write_snapshots(bs)

后续写入触发 COW:
  写入共享簇 (refcount=2) 时:
    → 分配新簇 (refcount=1)
    → 复制旧数据到新簇
    → 更新活动 L2 表指向新簇
    → 旧簇仍被快照 L1 引用 (refcount=1)

快照删除:
  qcow2_snapshot_delete()                  [qcow2-snapshot.c:909]
    ├─ 减少快照 L1 引用的所有簇的 refcount
    │    qcow2_update_snapshot_refcount(bs, l1_offset, -1)
    ├─ 释放快照 L1 表
    └─ 从快照表移除条目
```

### §18 L2/Refcount 缓存

**源文件**：`qcow2-cache.c:31-462`

```
Qcow2Cache 结构:
  ┌─────────────────────────────────────────┐
  │ Qcow2Cache                              │
  │  ├─ entries[N] (缓存条目数组)            │
  │  │    ├─ offset     (缓存的表偏移)       │
  │  │    ├─ table      (缓存的数据)         │
  │  │    ├─ dirty      (是否被修改)         │
  │  │    ├─ ref        (引用计数)           │
  │  │    └─ lru_counter (LRU 计数器)        │
  │  └─ size (每个条目大小 = cluster_size)   │
  └─────────────────────────────────────────┘

两个独立缓存:
  s->l2_table_cache      L2 表缓存 (默认 32 个条目)
  s->refcount_block_cache  引用计数块缓存 (默认 4 个条目)

操作:
  qcow2_cache_get(cache, offset, &table)   [qcow2-cache.c:406]
    ├─ 查找: 遍历 entries 匹配 offset
    ├─ 命中: ref++, 返回 table 指针
    └─ 未命中:
         ├─ 选择 LRU 条目 (ref=0 的最旧条目)
         ├─ 如果 dirty → 先写回磁盘
         ├─ 从磁盘读取新表数据
         └─ 返回 table 指针

  qcow2_cache_put(cache, &table)           [qcow2-cache.c:418]
    └─ ref--  (释放引用)

  qcow2_cache_entry_mark_dirty(cache, table) [qcow2-cache.c:432]
    └─ entry->dirty = true

  qcow2_cache_flush(cache)                 [qcow2-cache.c:259]
    └─ 遍历所有 dirty 条目 → 写回磁盘
```

### §19 性能特性

| 特性 | 说明 | 源文件参考 |
|------|------|-----------|
| **Lazy Refcounts** | 延迟刷新引用计数，crash 后需 repair | `qcow2.c:1155-1164` |
| **Extended L2** | 子簇 (subcluster) 支持，减少 COW 粒度 | `qcow2.h:82-101` |
| **外部数据文件** | 元数据与数据分离存储 | `qcow2.c:1722-1789` |
| **压缩簇** | zlib/zstd 压缩，减少磁盘占用 | `qcow2-cluster.c:668-678` |
| **Preallocation** | off/metadata/falloc/full 预分配 | 创建时选项 |
| **Discard/TRIM** | 主动释放 Guest 不用的空间 | `qcow2-refcount.c:736-805` |
| **L2 缓存** | 内存缓存 L2 表，减少磁盘读取 | `qcow2-cache.c` |

---

## 第四部分：Block 作业框架

### §20 Job 基类与生命周期

**源文件**：`job.c:58-84, 391-456`

```
Job 状态机:

  CREATED ──start()──► RUNNING ──ready()──► READY
     │                    │                    │
     │                    │                    │ complete()
     │                    ▼                    ▼
     │                 WAITING ◄─────────── WAITING
     │                    │
     │                    ▼
     │                 PENDING ──finalize──► CONCLUDED ──dismiss──► NULL
     │                    │
     │                    ▼
     └──cancel()───► ABORTING ──────────► CONCLUDED

Job 创建:
  job_create(id, driver, txn, ctx, ...)    [job.c:391]
    ├─ 分配 Job 结构
    ├─ 创建协程: job->co = qemu_coroutine_create(job->driver->run)
    ├─ 设置 AioContext
    └─ 状态 = JOB_STATUS_CREATED

Job 启动:
  job_start(job)                           [job.c:1121]
    ├─ 状态 → RUNNING
    └─ aio_co_enter(ctx, job->co)          进入协程

Job 控制:
  job_pause(job)   → 设置 pause_count++, 协程在 yield 点暂停
  job_resume(job)  → pause_count--, 恢复协程
  job_cancel(job)  → 设置 cancelled, 等待协程退出
  job_complete(job)→ 设置 should_complete, 协程完成剩余工作
```

### §21 BlockJob 子类

**源文件**：`blockjob.c:77-249`，`include/block/blockjob.h:42-95`

```c
// blockjob.h:42-95 (简化)
struct BlockJob {
    Job job;                    // 基类

    BlockBackend *blk;          // 关联的 BlockBackend
    BlockDeviceIoStatus iostatus;
    int64_t speed;              // 速率限制 (bytes/sec)
    RateLimit limit;            // 限速器
    // ...
};

block_job_create(job_id, driver, txn, bs, ...)  [blockjob.c:192]
  ├─ job_create(id, &driver->job_driver, ...)
  ├─ 创建临时 BlockBackend 关联 bs
  ├─ 设置 block notifiers (block_job_change_cb)
  └─ 安装节点 blocker (防止图修改)
```

### §22 Mirror 作业

**源文件**：`mirror.c:887-1304`

```
mirror_start(job_id, bs, target, ...)
  └─ mirror_start_job(...)                 [mirror.c:887]
       ├─ 创建脏位图 (dirty bitmap)
       │    bdrv_create_dirty_bitmap(bs, granularity, ...)
       ├─ block_job_create("mirror", ...)
       └─ job_start()

mirror_run() 协程:                         [mirror.c:1018]
  │
  ├─ 阶段1: 全量复制 (Bulk Copy)
  │    while (offset < s->bdev_length) {
  │      if (bdrv_dirty_bitmap_get(s->dirty_bitmap, offset))
  │        mirror_do_copy(offset, bytes) → 读源+写目标
  │      offset += granularity
  │    }
  │
  ├─ 阶段2: 增量同步 (Convergence)
  │    while (!s->should_complete) {
  │      // 处理新产生的脏块
  │      offset = bdrv_dirty_iter_next(s->dbi)
  │      if (offset >= 0) mirror_do_copy(offset, ...)
  │      else job_sleep() → 等待新的写入
  │    }
  │    // 当脏块为 0 → job_transition_to_ready()
  │                                        [mirror.c:1173]
  │
  └─ 阶段3: 完成 (Pivot)
       mirror_complete()                    [mirror.c:1269]
       ├─ 暂停源 I/O
       ├─ 复制最后剩余脏块
       └─ bdrv_replace_node(bs, target)     切换到目标
            // 所有指向源的 parent 改为指向目标

Active Mirroring (写拦截):
  Guest 写入源 → mirror_write_active()
    ├─ 同时写入源和目标
    └─ 标记为已同步 (清除脏位)
```

### §23 Commit 作业

**源文件**：`commit.c:131-520`

```
commit_start(job_id, bs, base, top, ...)    [commit.c:355]
  // 将 overlay 的数据合并到 backing (base)
  
  场景: base.qcow2 ← overlay.qcow2
        将 overlay 中分配的簇写入 base
        完成后删除 overlay

commit_run() 协程:                          [commit.c:209]
  while (offset < len) {
    // 读 overlay 中已分配的数据
    ret = blk_co_is_allocated_above(top_blk, base_blk, ...)
    if (allocated) {
      blk_co_pread(top_blk, offset, ...)   读 overlay
      blk_co_pwrite(base_blk, offset, ...) 写 base
    }
    offset += n
  }
  // 完成后: bdrv_drop_intermediate(top, base)
  // → 从图中移除中间节点
```

### §24 Stream 作业

**源文件**：`stream.c:54-237`

```
stream_start(job_id, bs, base, ...)
  // 将 backing chain 中的数据"拉平"到当前镜像
  // 完成后可断开 backing 关系

  场景: base.qcow2 ← mid.qcow2 ← active.qcow2
        将 base 中的数据复制到 active
        完成后 active 不再依赖 base

stream_run() 协程:                          [stream.c:152]
  while (offset < len) {
    ret = blk_co_is_allocated(above_blk, ...)
    if (!allocated_in_active) {
      // 数据在 backing 中 → 读出并写入 active
      blk_co_pread(backing_blk, offset, ...)
      blk_co_pwrite(active_blk, offset, ...)
    }
    offset += n
  }
  // 完成后: bdrv_set_backing_hd(active, NULL)
  // → 移除 backing 关系
```

### §25 Backup 作业

**源文件**：`backup.c:355-506`

```
backup_job_create(job_id, bs, target, sync_mode, bitmap, ...)
  │                                        [backup.c:355]
  │
  ├─ sync_mode 类型:
  │    FULL        全量备份 (复制所有数据)
  │    INCREMENTAL 增量备份 (仅复制脏位图标记的块)
  │    BITMAP      位图模式 (使用指定位图)
  │    TOP         仅当前层分配的数据
  │    NONE        不主动复制 (仅 CBW 保护)
  │
  ├─ 插入 CBW 过滤器
  │    bdrv_cbw_append(bs, target, ...)     [backup.c:461]
  │    // 在源和格式驱动之间插入 copy-before-write 过滤
  │    // Guest 写入前，先将旧数据复制到 target (备份保护)
  │
  └─ 使用脏位图追踪哪些块需要备份

CBW (Copy-Before-Write) 机制:
  Guest 写入请求到达:
    copy-before-write filter 拦截
    ├─ 检查该区域是否已备份
    ├─ 未备份 → 先读旧数据 → 写入 target (备份)
    ├─ 标记为已备份
    └─ 放行原始写入

  保证: 备份过程中的一致性 (point-in-time snapshot)
```

---

## 第五部分：高级 Block 特性

### §26 脏位图 (Dirty Bitmap)

**源文件**：`dirty-bitmap.c:33-516`

```
BdrvDirtyBitmap 结构:                     [dirty-bitmap.c:33]
  ├─ name          位图名称
  ├─ granularity   粒度 (字节，通常 = cluster_size)
  ├─ bitmap        HBitmap (位图数据)
  ├─ persistent    是否持久化到 qcow2
  ├─ disabled      是否禁用追踪
  └─ busy          是否正在被作业使用

创建:
  bdrv_create_dirty_bitmap(bs, granularity, name)
                                           [dirty-bitmap.c:99]
  → 分配 HBitmap → 挂到 bs->dirty_bitmaps 链表

使用场景:
  ┌─────────────────────────────────────────┐
  │ Mirror 作业:                            │
  │   创建位图 → Guest 写入自动标记脏 →      │
  │   mirror 协程遍历脏位找需要同步的块      │
  │                                         │
  │ 增量备份:                               │
  │   位图记录上次备份后的变化 →              │
  │   下次备份只复制脏块 →                   │
  │   完成后清除位图                         │
  │                                         │
  │ 持久化位图 (qcow2):                     │
  │   位图数据保存在 qcow2 文件中 →          │
  │   重启后恢复 → 支持跨重启增量备份        │
  └─────────────────────────────────────────┘

持久化到 qcow2:                    [qcow2-bitmap.c]
  qcow2_store_persistent_dirty_bitmaps()
    → 将位图数据序列化到 qcow2 额外 header 区域
  qcow2_load_dirty_bitmaps()
    → 打开时从 qcow2 恢复位图
```

### §27 Block 限流 (Throttle)

**源文件**：`throttle-groups.c:42-688`

```
命令行: -drive ...,throttling.iops-total=1000,throttling.bps-total=50m

限流架构:
  ┌──────────────────────┐
  │   ThrottleGroup      │  共享限流策略
  │   "group1"           │
  │  ┌────────────────┐  │
  │  │ ThrottleState  │  │  IOPS/BPS 参数
  │  │ cfg:           │  │
  │  │  iops[R/W/T]   │  │
  │  │  bps[R/W/T]    │  │
  │  │  burst 参数    │  │
  │  └────────────────┘  │
  │                      │
  │  成员 (round-robin): │
  │  ├─ tgm1 (drive1)   │
  │  ├─ tgm2 (drive2)   │
  │  └─ ...              │
  └──────────────────────┘

限流实现:
  throttle_group_co_io_limits_intercept()  [throttle-groups.c:349]
    │
    ├─ 检查是否超过限额
    │    throttle_compute_wait(&ts->cfg, ...)
    │    → 根据令牌桶算法计算等待时间
    │
    ├─ 如果需要等待:
    │    设置 timer (wait_ns 纳秒后唤醒)
    │    qemu_coroutine_yield()  ← 协程挂起
    │    // timer 到期 → 唤醒协程
    │    // → 继续执行 I/O
    │
    └─ 不需要等待: 直接放行

  令牌桶:
    每秒补充 iops/bps 配额
    每次 I/O 消耗配额
    超额 → 计算等待时间 → yield
    支持突发 (burst_length)
```

### §28 Block 过滤器

```
过滤器是插入 BDS 图中间的透明处理层:

  BlockBackend
       │
  ┌────▼─────┐          ┌──────────┐
  │ throttle │ file ───►│ qcow2    │ ──► file-posix
  │ (filter) │          │ (format) │
  └──────────┘          └──────────┘

常见过滤器:

  1. throttle (限流)
     在 I/O 路径插入速率控制

  2. copy-before-write (CBW)            [copy-before-write.c]
     备份时插入，写前先保护旧数据
     bdrv_cbw_append(bs, target)         [copy-before-write.c:548]
       → bdrv_insert_node(bs, node)  在图中插入节点

  3. blkdebug (调试/错误注入)           [blkdebug.c]
     -drive driver=blkdebug,image.filename=...,
            inject-error.event=read_aio,inject-error.errno=5
     → 指定事件时注入 I/O 错误 (用于测试)

  4. blkverify (数据验证)               [blkverify.c]
     同时读两个设备，比较结果
     → 用于验证新驱动正确性

过滤器通用操作:
  bdrv_insert_node(parent_bs, filter_node)   [block.c:843]
    → 在 parent 和 child 之间插入 filter

  bdrv_drop_filter(filter_node, &errp)
    → 从图中移除 filter，恢复直连
```

---


## 第六部分：virtio-blk 设备仿真

### §29 virtio-blk 概述

virtio-blk 是 QEMU 中最常用的虚拟块设备，通过 virtio 框架提供高性能磁盘 I/O。它将 Guest 的块 I/O 请求转换为 QEMU 块层的异步操作。

**QOM 类型层级**：

```
Object → DeviceState → VirtIODevice → VirtIOBlock

传输层包装:
  VirtIOBlockPCI  (virtio-blk-pci)  — PCI 传输
  VirtIOMMIOProxy (virtio-blk-device) — MMIO 传输
```

### §30 VirtIOBlock 结构体

定义于 `virtio-blk.h:54-93`：

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

### §31 设备 Realize 流程

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

### §32 请求处理管线

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

### §33 请求类型详解

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

### §34 I/O 完成路径

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

### §35 多队列与 IOThread

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

### §36 BlockBackend 集成

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

### §37 特性协商

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

### §38 CLI 使用与配置

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

---

## 附录

### 附录A：Block 驱动一览

**格式驱动 (Format)**：

| 驱动 | 文件 | 说明 |
|------|------|------|
| qcow2 | `block/qcow2.c` | QEMU 原生格式，快照/COW/压缩 |
| raw | `block/raw-format.c` | 直通，无格式化 |
| qed | `block/qed.c` | QEMU Enhanced Disk |
| vmdk | `block/vmdk.c` | VMware 磁盘格式 |
| vdi | `block/vdi.c` | VirtualBox 磁盘格式 |
| vhdx | `block/vhdx.c` | Hyper-V 磁盘格式 |
| vpc | `block/vpc.c` | VHD 格式 (旧版 Hyper-V) |
| dmg | `block/dmg.c` | macOS 磁盘镜像 |
| luks | `block/crypto.c` | 加密磁盘 |
| parallels | `block/parallels.c` | Parallels 磁盘格式 |

**协议驱动 (Protocol)**：

| 驱动 | 文件 | 说明 |
|------|------|------|
| file | `block/file-posix.c` | 本地文件 |
| host_device | `block/file-posix.c` | 块设备直接访问 |
| nbd | `block/nbd.c` | NBD 网络块设备 |
| gluster | `block/gluster.c` | GlusterFS |
| rbd | `block/rbd.c` | Ceph RADOS 块设备 |
| iscsi | `block/iscsi.c` | iSCSI 协议 |
| ssh | `block/ssh.c` | SSH/SFTP 远程文件 |
| curl | `block/curl.c` | HTTP/FTP 远程文件 |
| nvme | `block/nvme.c` | NVMe 用户态直接访问 |

**过滤驱动 (Filter)**：

| 驱动 | 文件 | 说明 |
|------|------|------|
| throttle | `block/throttle.c` | I/O 限流 |
| copy-before-write | `block/copy-before-write.c` | 备份 CBW |
| blkdebug | `block/blkdebug.c` | 错误注入/调试 |
| blkverify | `block/blkverify.c` | 数据校验 |
| preallocate | `block/preallocate.c` | 预分配 |
| snapshot-access | `block/snapshot-access.c` | 快照只读访问 |

### 附录B：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `block-backend.c` | ~2,800 | BlockBackend 实现 |
| `block.c` | ~10,000+ | BDS 核心：open/close/权限/图操作 |
| `blockdev.c` | ~1,800 | -drive/-blockdev 解析、QMP 接口 |
| `qcow2.c` | ~6,400 | qcow2 格式驱动主文件 |
| `qcow2-cluster.c` | ~2,000 | qcow2 簇映射/分配 |
| `qcow2-refcount.c` | ~3,600 | qcow2 引用计数管理 |
| `qcow2-snapshot.c` | ~1,200 | qcow2 快照操作 |
| `qcow2-cache.c` | ~480 | qcow2 L2/Refcount 缓存 |
| `qcow2-bitmap.c` | ~1,800 | qcow2 持久化脏位图 |
| `qcow2.h` | ~1,000 | qcow2 头部/常量/结构定义 |
| `mirror.c` | ~1,400 | Mirror 作业 |
| `commit.c` | ~520 | Commit 作业 |
| `stream.c` | ~300 | Stream 作业 |
| `backup.c` | ~510 | Backup 作业 |
| `copy-before-write.c` | ~600 | CBW 过滤器 |
| `dirty-bitmap.c` | ~520 | 脏位图框架 |
| `throttle-groups.c` | ~700 | 限流组实现 |
| `job.c` | ~1,250 | Job 基类状态机 |
| `blockjob.c` | ~600 | BlockJob 子类 |
| `file-posix.c` | ~4,000 | POSIX 文件协议驱动 |

### 附录C：关键 Git 提交

| 提交 | 说明 |
|------|------|
| `9ac85f4cc7` | block/mirror: fix assertion failure upon duplicate complete |
| `695d481a12` | qcow2: Simplify size round-up in co_create_opts |
| `910451bc5b` | qcow2: Add keep_data_file command-line option |
| `0f51f9c342` | mirror: Fix missed dirty bitmap writes during startup |
| `4a7b1bd18d` | block/mirror: check range when setting zero bitmap |
| `307bc43095` | block: Fix BDS use after free during shutdown |

---

> **交叉引用**
> - 块层 I/O 路径 (协程/AIO) → [02-块层IO路径深度分析](02-块层IO路径深度分析.md)
> - virtio-blk 设备仿真 → [01-关键设备仿真分析-UART-磁盘-网卡](01-关键设备仿真分析-UART-磁盘-网卡.md)
> - Machine 建立流程 (-drive 解析) → [../architecture/03-Machine建立流程深度分析](../architecture/03-Machine建立流程深度分析.md)
> - 设备模型与 virtio 框架 → [00-设备模型与virtio深度分析](00-设备模型与virtio深度分析.md)
> - Chardev 子系统 → [03-Chardev子系统与UART交互深度分析](03-Chardev子系统与UART交互深度分析.md)
