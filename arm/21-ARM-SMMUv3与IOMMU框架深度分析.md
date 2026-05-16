# ARM SMMUv3 与 IOMMU 框架深度分析

## 1. 概述

IOMMU（I/O Memory Management Unit）为设备 DMA 提供地址翻译和访问保护。ARM 平台使用 SMMUv3（System Memory Management Unit v3）。本文分析 QEMU 11.0.50 中通用 IOMMU 框架、SMMUv3 设备模型、流表查找、页表遍历、命令/事件队列和 IOTLB 缓存。

**关键源文件：**
- `include/system/memory.h` — IOMMUMemoryRegion 接口
- `system/physmem.c` — IOMMU 地址翻译集成
- `include/hw/arm/smmu-common.h` — SMMU 通用定义
- `hw/arm/smmu-common.c` — 通用 PTW 和 IOTLB
- `hw/arm/smmuv3.c` — SMMUv3 设备实现
- `hw/arm/smmuv3-internal.h` — SMMUv3 内部常量
- `hw/arm/virt.c` — ARM virt 的 SMMU 集成

---

## 2. QEMU IOMMU 框架

### 2.1 IOMMUMemoryRegion 接口

```c
// memory.h:372-543 — IOMMUMemoryRegionClass
```

QEMU 的通用 IOMMU 接口定义：
- `translate(mr, addr, flag, iommu_idx)` → `IOMMUTLBEntry` — 核心翻译方法
- `notify_flag_changed()` — 通知标志变更（用于 VFIO 监听）
- `attrs_to_index()` — 属性到 IOMMU 索引映射

### 2.2 IOMMUTLBEntry

```c
// memory.h:147-154
typedef struct IOMMUTLBEntry {
    AddressSpace *target_as;   // 翻译后的目标地址空间
    hwaddr iova;               // 输入虚拟地址（设备侧）
    hwaddr translated_addr;    // 翻译后的物理地址
    hwaddr addr_mask;          // 地址掩码（页大小 - 1）
    IOMMUAccessFlags perm;     // 权限（READ/WRITE/NONE）
} IOMMUTLBEntry;
```

### 2.3 地址翻译集成

```c
// physmem.c:422-467 — address_space_translate_iommu()
```

当设备发起 DMA 时：
1. 地址空间遍历遇到 IOMMUMemoryRegion
2. 调用 `translate()` 获取 `IOMMUTLBEntry`
3. 检查权限（读/写匹配）
4. 使用翻译后地址继续在目标地址空间中查找

### 2.4 IOMMU 区域初始化

```c
// memory.c:1694-1711 — memory_region_init_iommu()
```

SMMU 设备为每个下游设备创建 IOMMUMemoryRegion，挂载 `translate` 回调。

---

## 3. SMMUv3 设备模型

### 3.1 SMMUv3State

```c
// smmuv3.h:28-77
typedef struct SMMUv3State {
    SMMUState smmu_state;      // 通用 SMMU 状态（含 IOTLB）

    // 控制寄存器
    uint32_t features;
    uint32_t sid_split;        // 2 级流表拆分位

    // 命令/事件/PRI 队列
    SMMUQueue cmdq;            // 命令队列
    SMMUQueue evtq;            // 事件队列

    // MMIO 寄存器
    uint64_t strtab_base;      // 流表基地址
    uint32_t strtab_base_cfg;  // 流表配置

    // IRQ
    qemu_irq irq[4];          // GERROR, EVENT, CMD_SYNC, PRI

    // 互斥锁
    QemuMutex mutex;
};
```

### 3.2 设备实现

```c
// smmuv3.c:2016-2056 — smmuv3_realize()
```

realize 阶段：
1. 初始化 MMIO 区域和寄存器
2. 创建 IRQ 线
3. 初始化命令/事件队列
4. 注册 `smmuv3_translate` 到 IOMMUMemoryRegionClass

### 3.3 MMIO 寄存器

```c
// smmuv3.c:1585-1924 — smmu_writell/writel/readl
```

关键寄存器：

| 寄存器 | 功能 |
|--------|------|
| CR0 | 全局控制（SMMU 使能、队列使能） |
| CR1 | 队列中断控制 |
| STRTAB_BASE | 流表基地址 |
| STRTAB_BASE_CFG | 流表格式（线性/2级）和大小 |
| CMDQ_BASE/PROD/CONS | 命令队列 base/producer/consumer |
| EVTQ_BASE/PROD/CONS | 事件队列 base/producer/consumer |
| GBPA | 全局旁路属性 |
| IDR0-IDR5 | 能力标识寄存器 |

---

## 4. 流表查找（STE/CD）

### 4.1 STE 查找

```c
// smmuv3.c:660-740 — smmu_find_ste()
```

流表条目（Stream Table Entry）查找支持两种格式：

**线性流表**（单级）：
```
addr = strtab_base + sid * sizeof(STE)
```

**2 级流表**：
```c
// L1 索引 = sid >> sid_split
// L2 索引 = sid & ((1 << sid_split) - 1)
l1ptr = strtab_base + l1_ste_offset * sizeof(L1STD);
// 读取 L1 描述符获取 L2 表基地址
l2ptr = l1std_l2ptr(&l1std);
addr = l2ptr + l2_ste_offset * sizeof(STE);
```

2 级流表节省内存：稀疏 SID 空间只需分配使用的 L2 页。

### 4.2 STE 解码

```c
// smmuv3.c:573-646 — decode_ste()
```

STE 包含：
- **Config**：Bypass / S1 / S2 / Nested 翻译模式
- **S1 配置**：S1ContextPtr（指向 CD 表）
- **S2 配置**：VMID、S2TTB（Stage-2 页表基地址）、输入/输出地址大小
- **属性**：MemAttr、MTCFG 等

### 4.3 CD 查找与解码

```c
// smmuv3.c:378-414 — smmu_get_cd()
// smmuv3.c:742-837 — decode_cd()
```

Context Descriptor 包含 Stage-1 翻译配置：
- **TTB0/TTB1**：页表基地址（用户空间/内核空间）
- **T0SZ/T1SZ**：地址大小
- **TG0/TG1**：页面粒度（4KB/16KB/64KB）
- **ASID**：地址空间标识符

### 4.4 配置缓存

```c
// smmuv3.c:840-927 — smmuv3_decode_config() / smmuv3_get_config()
```

配置查找结果（STE+CD 解码后的 `SMMUTransCfg`）被缓存，避免每次翻译重新遍历流表。

---

## 5. 页表遍历（PTW）

### 5.1 翻译入口

```c
// smmuv3.c:1064-1153 — smmuv3_translate()
static IOMMUTLBEntry smmuv3_translate(IOMMUMemoryRegion *mr, hwaddr addr,
                                      IOMMUAccessFlags flag, int iommu_idx)
{
    // 1. 检查 SMMU 是否使能
    if (!smmu_enabled(s)) {
        // GBPA.ABORT → 中止; 否则 → 旁路（直通）
    }

    // 2. 获取配置（STE + CD）
    cfg = smmuv3_get_config(sdev, &event);

    // 3. 查找 IOTLB 缓存
    cached_entry = smmu_iotlb_lookup(bs, cfg, &tt_combined, addr);
    if (cached_entry) {
        // 缓存命中 → 直接返回翻译结果
    }

    // 4. 缓存未命中 → 执行页表遍历
    // smmuv3_do_translate() → smmu_ptw()
}
```

### 5.2 Stage-1 遍历

```c
// smmu-common.c:458-571 — smmu_ptw_64_s1()
```

Stage-1 PTW（IOVA → IPA）：
1. 根据 IOVA 选择 TT0 或 TT1（通过高位判断用户/内核空间）
2. 计算起始级别：`level = 4 - (inputsize - 4) / stride`
3. 逐级遍历：
   - 读取 PTE（`get_pte()`）
   - 无效 PTE → 翻译错误
   - Table PTE → 检查 APTable 权限，进入下一级
   - Block/Page PTE → 提取输出地址，结束
4. 权限检查（AP 位 vs 访问类型）

```c
// smmu-common.c:481-528
while (level < VMSA_LEVELS) {
    get_pte(baseaddr, offset, &pte, info);

    if (is_invalid_pte(pte) || is_reserved_pte(pte, level))
        break;  // 翻译错误

    if (is_table_pte(pte, level)) {
        ap = PTE_APTABLE(pte);
        if (is_permission_fault(ap, perm) && !tt->had)
            goto error;  // 权限错误
        baseaddr = get_table_pte_address(pte, granule_sz);
        level++;
        continue;
    } else if (is_page_pte(pte, level)) {
        gpa = get_page_pte_address(pte, granule_sz);
    } else {
        gpa = get_block_pte_address(pte, level, granule_sz, &block_size);
    }
    // ...
}
```

### 5.3 Stage-2 遍历

```c
// smmu-common.c:587-695 — smmu_ptw_64_s2()
```

Stage-2 PTW（IPA → PA）结构与 S1 类似，但：
- 输入地址是 IPA（中间物理地址）
- 使用 S2TTB 作为页表基地址
- 权限模型不同（S2AP 而非 AP）

### 5.4 嵌套翻译（Nested）

对于 Nested 模式（S1 + S2）：
- 先执行 S1 遍历获取 IPA
- S1 页表本身的地址也需要 S2 翻译（`translate_table_addr_ipa()`）
- 最终 IPA 再经过 S2 遍历得到 PA

---

## 6. 命令队列

### 6.1 队列结构

```c
// smmuv3.h:28-34 — SMMUQueue
// smmuv3.c:111-146 — queue_read/write
```

命令队列是环形缓冲区：
- **PROD**：生产者指针（Guest 写入）
- **CONS**：消费者指针（SMMU 更新）
- Guest 写 CMDQ_PROD doorbell 触发 SMMU 消费命令

### 6.2 命令处理

```c
// smmuv3.c:1308-1582 — smmuv3_cmdq_consume()
```

支持的命令类型（1355-1540）：

| 命令 | 功能 |
|------|------|
| CMDQ_OP_CFGI_STE | 使配置缓存中的 STE 无效 |
| CMDQ_OP_CFGI_CD | 使配置缓存中的 CD 无效 |
| CMDQ_OP_CFGI_ALL | 使所有配置缓存无效 |
| CMDQ_OP_TLBI_NH_ASID | 按 ASID 无效 IOTLB |
| CMDQ_OP_TLBI_NH_VA | 按虚拟地址无效 IOTLB |
| CMDQ_OP_TLBI_NSNH_ALL | 无效所有非安全 IOTLB |
| CMDQ_OP_TLBI_S12_VMALL | 按 VMID 无效 S1+S2 IOTLB |
| CMDQ_OP_CMD_SYNC | 同步屏障 |

---

## 7. 事件队列

### 7.1 事件记录

```c
// smmuv3.c:185-269 — smmuv3_record_event()
```

翻译错误、配置错误等异常记录到事件队列：
1. 构造事件记录（包含 SID、地址、错误类型）
2. 写入 EVTQ 环形缓冲区
3. 触发事件中断

### 7.2 事件类型

```c
// smmuv3-internal.h:213-255
```

| 事件 | 含义 |
|------|------|
| SMMU_EVT_F_TRANSLATION | 翻译错误（PTE 无效） |
| SMMU_EVT_F_PERMISSION | 权限错误 |
| SMMU_EVT_F_STE_FETCH | STE 获取失败 |
| SMMU_EVT_C_BAD_STREAMID | 无效的 Stream ID |
| SMMU_EVT_C_BAD_STE | STE 格式错误 |
| SMMU_EVT_C_BAD_CD | CD 格式错误 |

---

## 8. IOTLB 缓存

### 8.1 缓存操作

```c
// smmu-common.c:70-160
```

IOTLB 是 SMMUv3 的翻译缓存：
- **查找**（109-138）：以 (ASID, VMID, IOVA, level) 为键查找缓存条目
- **插入**（141-155）：将 PTW 结果存入缓存
- 基于 GHashTable 实现

### 8.2 缓存无效化

```c
// smmu-common.c:157-327
```

多种粒度的无效化：
- `inv_all` — 清除所有条目
- `inv_iova` — 按 IOVA 无效
- `inv_ipa` — 按 IPA 无效（影响 S2 缓存）
- `inv_asid_vmid` — 按 ASID+VMID 无效
- `inv_vmid` — 按 VMID 无效所有条目
- `inv_vmid_s1` — 按 VMID 无效 S1 条目

---

## 9. 中断机制

### 9.1 IRQ 类型

```c
// smmuv3.h:79-84
```

SMMUv3 支持 4 类中断：
1. **GERROR**：全局错误（配置错误等严重错误）
2. **EVENTQ**：事件队列非空
3. **CMD_SYNC**：命令同步完成
4. **PRI**：Page Request Interface

### 9.2 IRQ 触发

```c
// smmuv3.c:53-89 — smmuv3_trigger_irq()
```

中断触发采用脉冲（pulse）模式，通过 `qemu_irq_pulse()` 发送中断到 GIC。

---

## 10. ARM virt SMMU 集成

### 10.1 创建流程

```c
// virt.c:1850-1880 — create_smmu()
```

1. 创建 `TYPE_ARM_SMMUV3` 设备实例
2. 链接 `primary-bus`（GPEX PCIe 总线）
3. 链接 `memory` 和 `secure-memory` 地址空间
4. 映射 MMIO 寄存器到 VIRT_SMMU 地址
5. 连接 4 个 IRQ 到 GIC SPI
6. 生成设备树绑定

### 10.2 PCIe IOMMU 映射

```c
// virt.c:1845-1847
```

通过 `iommu-map` 设备树属性将 PCIe Requester ID（BDF）映射到 SMMU Stream ID，使所有 PCIe 设备的 DMA 经过 SMMU 翻译。

---

## 11. VFIO 与 SMMU 交互

### 11.1 IOMMU 通知器

```c
// hw/vfio/listener.c:77-201
```

VFIO 监听 IOMMUMemoryRegion 的 map/unmap 通知：
- **MAP**：将 Guest IOVA→HPA 映射传递给物理 IOMMU
- **UNMAP**：移除物理 IOMMU 映射

### 11.2 iommufd 后端

```c
// hw/vfio/iommufd.c:130-180
```

现代 VFIO 使用 iommufd API（替代传统 container）：
- `bind` — 将设备绑定到 iommufd
- `connect` — 建立 IOAS（I/O Address Space）
- 支持嵌套翻译（Guest SMMU S1 + Host SMMU S2）

---

## 12. 完整翻译路径

### 12.1 设备 DMA 翻译

```
1. PCI 设备发起 DMA（IOVA 地址）
2. address_space_translate_iommu()
3. → smmuv3_translate()
4.   → 检查 SMMU 使能状态
5.   → smmuv3_get_config()  — STE + CD 查找/缓存
6.   → IOTLB 查找
7.   → [缓存未命中] smmu_ptw()
8.     → smmu_ptw_64_s1() — Stage-1（IOVA→IPA）
9.     → smmu_ptw_64_s2() — Stage-2（IPA→PA）
10.  → 返回 IOMMUTLBEntry{PA, mask, perm}
11. → 继续物理内存访问
```

### 12.2 错误处理

```
翻译错误 → smmuv3_record_event()
  → 写入 EVTQ → 触发 EVENTQ IRQ → Guest 处理错误
严重错误 → GERROR IRQ → Guest 错误恢复
```

---

## 13. 小结

| 方面 | 实现要点 |
|------|----------|
| **IOMMU 框架** | IOMMUMemoryRegion + translate() 接口，physmem.c 集成 |
| **SMMUv3** | MMIO 寄存器组 + 命令/事件队列 + 流表 + 页表遍历 |
| **流表** | 线性/2 级查找 STE → CD，配置缓存避免重复解码 |
| **PTW** | S1（IOVA→IPA）+ S2（IPA→PA），支持 4KB/16KB/64KB 页 |
| **命令队列** | CFGI（配置无效）+ TLBI（TLB 无效）+ SYNC（同步） |
| **事件队列** | 翻译/权限/配置错误记录，EVENTQ IRQ 通知 |
| **IOTLB** | GHashTable 实现，多粒度无效化（ASID/VMID/IOVA/IPA） |
| **ARM 集成** | virt 机器链接 GPEX 总线，iommu-map 属性映射 BDF→SID |
| **VFIO** | 监听 map/unmap 通知，iommufd 支持嵌套翻译 |
