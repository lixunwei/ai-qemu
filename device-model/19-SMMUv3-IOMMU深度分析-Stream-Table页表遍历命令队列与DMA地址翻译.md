# SMMUv3/IOMMU 深度分析：Stream Table、页表遍历、命令队列与 DMA 地址翻译

> **QEMU 版本**：11.0.50  
> **分析日期**：2025 年  
> **文档大小**：~28KB  
> **覆盖范围**：IOMMU 核心框架（IOMMUMemoryRegion）、SMMUv3 设备模型、Stream Table Entry/Context Descriptor、页表遍历（Stage-1/Stage-2/Nested）、IOTLB 缓存、命令队列与事件队列、中断机制、IOMMU Notifier、virt 机器集成

---

## 目录

1. [IOMMU 核心框架](#1-iommu-核心框架)
2. [SMMUv3 设备模型与状态结构](#2-smmuv3-设备模型与状态结构)
3. [SMMU 通用基础层](#3-smmu-通用基础层)
4. [Stream Table Entry（STE）](#4-stream-table-entryste)
5. [Context Descriptor（CD）](#5-context-descriptorcd)
6. [翻译主路径](#6-翻译主路径)
7. [页表遍历（PTW）](#7-页表遍历ptw)
8. [IOTLB 缓存管理](#8-iotlb-缓存管理)
9. [命令队列（CMDQ）](#9-命令队列cmdq)
10. [事件队列与错误报告](#10-事件队列与错误报告)
11. [中断机制](#11-中断机制)
12. [IOMMU Notifier 与 DMA 集成](#12-iommu-notifier-与-dma-集成)
13. [virt 机器集成](#13-virt-机器集成)

---

## 1. IOMMU 核心框架

### 1.1 IOMMUTLBEntry — 翻译结果条目

QEMU IOMMU 子系统的核心数据结构，表示一次 IOMMU 翻译的结果：

```c
// include/system/memory.h:147-154
struct IOMMUTLBEntry {
    AddressSpace    *target_as;
    hwaddr           iova;
    hwaddr           translated_addr;
    hwaddr           addr_mask;  /* 0xfff = 4k translation */
    IOMMUAccessFlags perm;
    uint32_t         pasid;
};
```

**字段含义**：
- `target_as`：翻译结果指向的地址空间（通常是 `address_space_memory`）
- `iova`：输入虚拟地址（Input Output Virtual Address）
- `translated_addr`：翻译后的物理地址
- `addr_mask`：页大小掩码（4KB = 0xfff，2MB = 0x1fffff，1GB = 0x3fffffff）
- `perm`：访问权限（IOMMU_RO/IOMMU_WO/IOMMU_RW/IOMMU_NONE）
- `pasid`：Process Address Space ID（用于 PASID/SubstreamID 支持）

### 1.2 IOMMUNotifier — 事件通知机制

IOMMU 通知器用于在翻译映射发生变化时通知关注方（如 VFIO）：

```c
// include/system/memory.h:186-194
typedef enum {
    IOMMU_NOTIFIER_NONE = 0,
    IOMMU_NOTIFIER_UNMAP = 0x1,          // 缓存失效（映射移除）
    IOMMU_NOTIFIER_MAP = 0x2,            // 新映射建立
    IOMMU_NOTIFIER_DEVIOTLB_UNMAP = 0x04, // 设备 IOTLB 失效
} IOMMUNotifierFlag;

// include/system/memory.h:205-214
struct IOMMUNotifier {
    IOMMUNotify notify;
    IOMMUNotifierFlag notifier_flags;
    hwaddr start;         // 监听的地址范围起始
    hwaddr end;           // 监听的地址范围结束
    int iommu_idx;
    void *opaque;
    QLIST_ENTRY(IOMMUNotifier) node;
};
```

**两种使用模式**（memory.h:156-184）：
1. **精确同步**（MAP|UNMAP）：设备维护影子页表，需要每次映射创建/失效的精确通知（如 VFIO 直通场景）
2. **缓存失效**（UNMAP only）：设备维护翻译缓存（IOTLB），只需失效通知即可通过 ATS 协议重新获取翻译

### 1.3 IOMMUMemoryRegionClass — IOMMU 虚函数表

```c
// include/system/memory.h:401-543
struct IOMMUMemoryRegionClass {
    MemoryRegionClass parent_class;
    
    // 核心翻译方法 — 给定 IOVA 返回 IOMMUTLBEntry
    IOMMUTLBEntry (*translate)(IOMMUMemoryRegion *iommu, hwaddr addr,
                               IOMMUAccessFlags flag, int iommu_idx);
    // 最小页大小（默认 TARGET_PAGE_SIZE）
    uint64_t (*get_min_page_size)(IOMMUMemoryRegion *iommu);
    // 通知标志变化回调
    int (*notify_flag_changed)(IOMMUMemoryRegion *iommu,
                               IOMMUNotifierFlag old, IOMMUNotifierFlag new,
                               Error **errp);
    // 重放所有当前映射给通知器
    void (*replay)(IOMMUMemoryRegion *iommu, IOMMUNotifier *notifier);
    // 获取 IOMMU 特定属性
    int (*get_attr)(IOMMUMemoryRegion *iommu, enum IOMMUMemoryRegionAttr attr,
                    void *data);
    int (*attrs_to_index)(IOMMUMemoryRegion *iommu, MemTxAttrs attrs);
    int (*num_indexes)(IOMMUMemoryRegion *iommu);
};
```

SMMUv3 对这些虚函数的实现注册在 `smmuv3_iommu_memory_region_class_init()`（smmuv3.c:2220-2227）：
- `translate` → `smmuv3_translate()`
- `notify_flag_changed` → `smmuv3_notify_flag_changed()`

---

## 2. SMMUv3 设备模型与状态结构

### 2.1 SMMUv3State — 主设备状态

```c
// include/hw/arm/smmuv3.h:36-77
struct SMMUv3State {
    SMMUState     smmu_state;       // 基类（包含 PCI 总线关联、IOTLB 缓存等）
    
    // 功能与配置
    uint32_t features;              // SMMU_FEATURE_2LVL_STE 等
    uint8_t sid_size;               // Stream ID 位宽
    uint8_t sid_split;              // 2 级 STE 拆分点
    
    // 寄存器
    uint32_t idr[6];                // IDR0-IDR5 标识寄存器
    uint32_t iidr;                  // 实现标识
    uint32_t aidr;                  // 架构标识
    uint32_t cr[3];                 // CR0/CR1/CR2 控制寄存器
    uint32_t cr0ack;                // CR0 确认
    uint32_t statusr;               // 状态寄存器
    uint32_t gbpa;                  // 全局旁路属性
    uint32_t irq_ctrl;              // 中断控制
    uint32_t gerror;                // 全局错误
    uint32_t gerrorn;               // 全局错误确认
    uint64_t gerror_irq_cfg0;       // GERROR IRQ 配置
    uint32_t gerror_irq_cfg1;
    uint32_t gerror_irq_cfg2;
    uint64_t strtab_base;           // Stream Table 基地址
    uint32_t strtab_base_cfg;       // Stream Table 配置
    uint64_t eventq_irq_cfg0;       // 事件队列 IRQ 配置
    
    SMMUQueue eventq, cmdq;         // 命令/事件队列
    
    qemu_irq     irq[4];           // 4 个中断线
    QemuMutex mutex;                // 翻译/命令互斥锁
    char *stage;                    // "1"/"2"/"nested"
    
    // 加速器支持（iommufd）
    bool accel;                     // 是否启用硬件加速
    struct SMMUv3AccelState *s_accel;
    uint64_t msi_gpa;              // MSI 门铃 GPA
    OnOffAuto ril;                  // Range Invalidation Length
    OnOffAuto ats;                  // ATS 支持
    OasMode oas;                    // Output Address Size
    SsidSizeMode ssidsize;          // SubstreamID 位宽
};
```

### 2.2 中断类型

```c
// include/hw/arm/smmuv3.h:79-84
typedef enum {
    SMMU_IRQ_EVTQ,       // 事件队列非空
    SMMU_IRQ_PRIQ,       // PRI 队列（未实现）
    SMMU_IRQ_CMD_SYNC,   // 命令同步完成
    SMMU_IRQ_GERROR,     // 全局错误
} SMMUIrq;
```

### 2.3 设备属性

```c
// smmuv3.c:2126-2144
static const Property smmuv3_properties[] = {
    DEFINE_PROP_STRING("stage", SMMUv3State, stage),          // "1"/"2"/"nested"
    DEFINE_PROP_BOOL("accel", SMMUv3State, accel, false),     // 硬件加速
    DEFINE_PROP_UINT64("msi-gpa", SMMUv3State, msi_gpa, 0),
    DEFINE_PROP_ON_OFF_AUTO("ril", SMMUv3State, ril, ON_OFF_AUTO_ON),
    DEFINE_PROP_ON_OFF_AUTO("ats", SMMUv3State, ats, ON_OFF_AUTO_OFF),
    DEFINE_PROP_OAS_MODE("oas", SMMUv3State, oas, OAS_MODE_44),
    DEFINE_PROP_SSIDSIZE_MODE("ssidsize", SMMUv3State, ssidsize, SSID_SIZE_MODE_0),
};
```

### 2.4 类型注册与层次

```c
// smmuv3.c:2229-2250
static const TypeInfo smmuv3_type_info = {
    .name          = TYPE_ARM_SMMUV3,    // "arm-smmuv3"
    .parent        = TYPE_ARM_SMMU,       // "arm-smmu"（基类）
    .instance_size = sizeof(SMMUv3State),
    .class_size    = sizeof(SMMUv3Class),
    .class_init    = smmuv3_class_init,
};

static const TypeInfo smmuv3_iommu_memory_region_info = {
    .parent = TYPE_IOMMU_MEMORY_REGION,
    .name = TYPE_SMMUV3_IOMMU_MEMORY_REGION,
    .class_init = smmuv3_iommu_memory_region_class_init,  // 注册 translate/notify
};
```

---

## 3. SMMU 通用基础层

### 3.1 SMMUState — 基类状态

```c
// include/hw/arm/smmu-common.h:150-170
struct SMMUState {
    SysBusDevice  dev;
    const char *mrtypename;
    MemoryRegion iomem;
    
    GHashTable *smmu_pcibus_by_busptr;    // PCIBus* → SMMUPciBus 映射
    GHashTable *configs;                   // SMMUDevice → SMMUTransCfg 配置缓存
    GHashTable *iotlb;                     // SMMUIOTLBKey → SMMUTLBEntry TLB 缓存
    SMMUPciBus *smmu_pcibus_by_bus_num[256]; // 按总线号索引的 PCI 总线
    PCIBus *pci_bus;
    QLIST_HEAD(, SMMUDevice) devices_with_notifiers; // 注册了通知的设备列表
    uint8_t bus_num;
    PCIBus *primary_bus;
    bool smmu_per_bus;
    MemoryRegion *memory;                  // 系统内存
    AddressSpace memory_as;
    MemoryRegion *secure_memory;           // 安全内存
    AddressSpace secure_memory_as;
    const PCIIOMMUOps *iommu_ops;
};
```

### 3.2 SMMUDevice — 每设备 IOMMU 上下文

```c
// include/hw/arm/smmu-common.h:121-130
typedef struct SMMUDevice {
    void               *smmu;        // 指向 SMMUState
    PCIBus             *bus;         // 所在 PCI 总线
    int                devfn;        // PCI device/function
    IOMMUMemoryRegion  iommu;        // IOMMU 内存区域（虚拟地址空间）
    AddressSpace       as;           // 设备的地址空间
    uint32_t           cfg_cache_hits;
    uint32_t           cfg_cache_misses;
    QLIST_ENTRY(SMMUDevice) next;
} SMMUDevice;
```

### 3.3 Stream ID 派生

Stream ID 直接从 PCI BDF（Bus:Device.Function）派生：

```c
// include/hw/arm/smmu-common.h:195-198
static inline uint16_t smmu_get_sid(SMMUDevice *sdev)
{
    return PCI_BUILD_BDF(pci_bus_num(sdev->bus), sdev->devfn);
}
```

这意味着 Stream ID = (Bus << 8) | (Device << 3) | Function，最多 16 位。

---

## 4. Stream Table Entry（STE）

### 4.1 STE 数据结构与字段布局

STE 是 SMMUv3 的核心配置结构，每个 Stream ID 对应一个 STE：

```c
// include/hw/arm/smmuv3-common.h:29-31
typedef struct STE {
    uint32_t word[16];    // 64 字节（512 位）
} STE;
```

**关键字段**（smmuv3-common.h:40-97）：

| 字段 | Word | 位域 | 含义 |
|------|------|------|------|
| `VALID` | word[0] | [0] | STE 有效位 |
| `CONFIG` | word[0] | [1:3] | 配置模式：S1/S2/Bypass/Abort |
| `S1FMT` | word[0] | [4:5] | Stage-1 Context Descriptor 格式 |
| `CTXPTR` | word[0:1] | [6:55] | Context Descriptor 基地址 |
| `S1CDMAX` | word[1] | [27:31] | 最大 CD 数量（2^S1CDMAX） |
| `S1STALLD` | word[2] | [27] | Stage-1 Stall 故障模型 |
| `EATS` | word[2] | [28:29] | ATS 使能 |
| `S2VMID` | word[4] | [0:15] | Stage-2 虚拟机 ID |
| `S2T0SZ` | word[5] | [0:5] | Stage-2 输入地址大小（64-S2T0SZ） |
| `S2SL0` | word[5] | [6:7] | Stage-2 起始遍历级别 |
| `S2TG` | word[5] | [14:15] | Stage-2 页大小（0=4KB/1=64KB/2=16KB） |
| `S2PS` | word[5] | [16:18] | Stage-2 物理地址大小 |
| `S2AA64` | word[5] | [19] | AArch64 翻译表（必须为 1） |
| `S2AFFD` | word[5] | [21] | AF 故障禁用 |
| `S2S` | word[5] | [25] | Stall 使能 |
| `S2R` | word[5] | [26] | 记录故障事件 |
| `S2TTB` | word[6:7] | | Stage-2 翻译表基地址 |

### 4.2 CONFIG 字段解读

```c
// smmuv3-common.h:98-102
#define STE_CFG_S1_ENABLED(config) (config & 0x1)  // bit0: Stage-1
#define STE_CFG_S2_ENABLED(config) (config & 0x2)  // bit1: Stage-2
#define STE_CFG_ABORT(config)      (!(config & 0x4)) // bit2=0: Abort
#define STE_CFG_BYPASS(config)     (config == 0x4)   // 0b100: Bypass
```

**CONFIG 组合**：
- `0b000`：Abort（禁止翻译，丢弃事务）
- `0b100`：Bypass（直通，不翻译）
- `0b101`：Stage-1 Only
- `0b110`：Stage-2 Only
- `0b111`：Nested（Stage-1 + Stage-2）

### 4.3 Stream Table 查找 — 线性与两级

```c
// smmuv3.c:660-740
int smmu_find_ste(SMMUv3State *s, uint32_t sid, STE *ste, SMMUEventInfo *event)
```

**线性 Stream Table**（单级）：
```
strtab_base + sid * sizeof(STE)
```

**两级 Stream Table**（SMMU_FEATURE_2LVL_STE）：
```
L1 索引 = sid >> sid_split
L2 索引 = sid & ((1 << sid_split) - 1)

L1 表项地址 = strtab_base + L1索引 * sizeof(STEDesc)
→ 读取 L1STD，获取 L2 指针和 span
L2 表项地址 = l2ptr + L2索引 * sizeof(STE)
```

**关键细节**：
- SID 范围检查：`sid >= (1 << MIN(log2size, SMMU_IDR1_SIDSIZE))` → `C_BAD_STREAMID`
- L1 span=0 表示 L2 无效 → `C_BAD_STREAMID`
- L2 偏移超出 span → `C_BAD_STE`

### 4.4 STE 解码

```c
// smmuv3.c:573-646
static int decode_ste(SMMUv3State *s, SMMUTransCfg *cfg, STE *ste, SMMUEventInfo *event)
```

解码流程：
1. **有效性检查**：`STE_VALID(ste)` 必须为 1
2. **CONFIG 解码**：`decode_ste_config()` 设置 `cfg->stage`/`cfg->aborted`/`cfg->bypassed`
3. **阶段支持检查**：S1 或 S2 在 SW 启用但硬件未公告 → `C_BAD_STE`
4. **VMID 提取**：即使 S2 未启用也提取 VMID（用于 TLB 键）
5. **S2 配置解码**：`decode_ste_s2_cfg()` — 颗粒度、起始级别、PA 范围、TTB 等
6. **SubstreamID 检查**：`S1CDMAX != 0` 需要 `ssidsize > 0`
7. **Stall 检查**：`S1STALLD` 不支持

---

## 5. Context Descriptor（CD）

### 5.1 CD 数据结构

```c
// smmuv3-common.h:34-36
typedef struct CD {
    uint32_t word[16];    // 64 字节
} CD;
```

**关键字段**（smmuv3-common.h:157-200）：

| 字段 | Word | 位域 | 含义 |
|------|------|------|------|
| `TSZ0/TSZ1` | word[0] | [0:5]/[16:21] | TTB0/TTB1 输入大小 |
| `TG0/TG1` | word[0] | [6:7]/[22:23] | TTB0/TTB1 页大小 |
| `EPD0/EPD1` | word[0] | [14]/[30] | TTB0/TTB1 禁用 |
| `VALID` | word[0] | [31] | CD 有效 |
| `IPS` | word[1] | [0:2] | 输出物理地址大小 |
| `AFFD` | word[1] | [3] | AF 故障禁用 |
| `TBI` | word[1] | [6:7] | Top Byte Ignore |
| `AARCH64` | word[1] | [9] | AArch64 翻译表 |
| `HD/HA` | word[1] | [10:11] | 硬件脏位/访问标志更新 |
| `ASID` | word[1] | [16:31] | 地址空间 ID |
| `HAD0/HAD1` | word[2]/[4] | [1] | 层次属性禁用 |
| `TTB0` | word[2:3] | | TTB0 基地址 |
| `TTB1` | word[4:5] | | TTB1 基地址 |

### 5.2 CD 获取与翻译

```c
// smmuv3.c:378-414
static int smmu_get_cd(SMMUv3State *s, STE *ste, SMMUTransCfg *cfg,
                       uint32_t ssid, CD *buf, SMMUEventInfo *event)
```

**Nested 模式下的 CD 地址翻译**：

在 Nested 翻译中，CD 的地址（CTXPTR）是 IPA（Stage-1 输出地址），需要先通过 Stage-2 翻译为 PA：

```c
if (cfg->stage == SMMU_NESTED) {
    status = smmuv3_do_translate(s, addr, cfg, event,
                                 IOMMU_RO, &entry, SMMU_CLASS_CD);
    if (status != SMMU_TRANS_SUCCESS) return -EINVAL;
    addr = CACHED_ENTRY_TO_ADDR(entry, addr);
}
```

这确保了虚拟机 Hypervisor 设置的 CD 地址被正确翻译。

### 5.3 CD 解码

```c
// smmuv3.c:742-838
static int decode_cd(SMMUv3State *s, SMMUTransCfg *cfg, CD *cd, SMMUEventInfo *event)
```

解码流程：
1. 有效性检查：`CD_VALID && CD_AARCH64`
2. 提取 OAS：`MIN(oas2bits(CD_IPS), oas2bits(IDR5.OAS))`
3. TBI、ASID、AFFD 提取
4. 对 TT0/TT1 分别解码：TSZ（16-39）、TG（颗粒度）、TTB 基地址
5. **TTB Nested 翻译**：TTB 地址也需要 Stage-2 翻译 → `smmuv3_do_translate(..., SMMU_CLASS_TT)`
6. HAD（Hierarchical Attribute Disable）

---

## 6. 翻译主路径

### 6.1 smmuv3_translate() — 翻译入口

```c
// smmuv3.c:1064-1153
static IOMMUTLBEntry smmuv3_translate(IOMMUMemoryRegion *mr, hwaddr addr,
                                      IOMMUAccessFlags flag, int iommu_idx)
```

这是 `IOMMUMemoryRegionClass.translate` 的实现，当 PCI 设备发起 DMA 时被调用。

**完整翻译流程**：

```
smmuv3_translate(mr, addr, flag)
  │
  ├── 获取 SMMUDevice、sid
  ├── mutex_lock
  │
  ├── SMMU 未启用？
  │   ├── GBPA.ABORT → SMMU_TRANS_ABORT
  │   └── 否则 → SMMU_TRANS_DISABLE（直通）
  │
  ├── smmuv3_get_config(sdev, &event) ← 配置缓存查找/解码
  │   │
  │   ├── [缓存命中] → 返回 cfg
  │   └── [缓存未命中] → smmuv3_decode_config(mr, cfg, event)
  │       ├── smmu_find_ste()      ← Stream Table 查找
  │       ├── decode_ste()          ← STE 解码
  │       ├── smmu_get_cd()         ← CD 获取（可能 S2 翻译）
  │       └── decode_cd()           ← CD 解码
  │
  ├── cfg->aborted → SMMU_TRANS_ABORT
  ├── cfg->bypassed → SMMU_TRANS_BYPASS
  │
  ├── smmuv3_do_translate(s, addr, cfg, &event, flag, &entry, SMMU_CLASS_IN)
  │   ├── smmu_translate()  ← IOTLB 查找 + PTW
  │   └── 错误处理 → SMMUEventInfo
  │
  └── mutex_unlock
      │
      ├── SMMU_TRANS_SUCCESS → entry.perm/translated_addr/addr_mask
      ├── SMMU_TRANS_DISABLE → perm=flag, 直通
      ├── SMMU_TRANS_BYPASS → perm=flag, 直通
      ├── SMMU_TRANS_ABORT → perm=IOMMU_NONE
      └── SMMU_TRANS_ERROR → smmuv3_record_event() 记录故障
```

### 6.2 smmuv3_get_config() — 配置缓存

```c
// smmuv3.c:898-927
static SMMUTransCfg *smmuv3_get_config(SMMUDevice *sdev, SMMUEventInfo *event)
```

使用 GHashTable 缓存每个 SMMUDevice 的 SMMUTransCfg：
- **命中**：直接返回，递增 `cfg_cache_hits`
- **未命中**：执行 `smmuv3_decode_config()` 完整解码，成功后插入缓存

缓存失效由 `smmuv3_flush_config()` 执行，在 CFGI_STE/CFGI_CD 命令处理时调用。

### 6.3 smmuv3_do_translate() — 翻译与错误映射

```c
// smmuv3.c:939-1041
static SMMUTranslationStatus smmuv3_do_translate(SMMUv3State *s, hwaddr addr,
                                                 SMMUTransCfg *cfg, ...)
```

**翻译分类（SMMUTranslationClass）**：

```c
// smmuv3-internal.h:36-40
typedef enum {
    SMMU_CLASS_CD,  // CD 获取时的 S2 翻译
    SMMU_CLASS_TT,  // TTB 地址的 S2 翻译
    SMMU_CLASS_IN,  // 正常输入翻译
} SMMUTranslationClass;
```

对于 CD/TT 类的描述符翻译（Nested 模式），函数临时切换 cfg 为 Stage-2：
```c
if (desc_s2_translation) {
    asid = cfg->asid;
    stage = cfg->stage;
    cfg->asid = -1;
    cfg->stage = SMMU_STAGE_2;
}
cached_entry = smmu_translate(bs, cfg, addr, flag, &ptw_info);
// 恢复
cfg->asid = asid;
cfg->stage = stage;
```

**错误映射**（PTW 错误 → SMMU 事件类型）：
- `SMMU_PTW_ERR_WALK_EABT` → `SMMU_EVT_F_WALK_EABT`
- `SMMU_PTW_ERR_TRANSLATION` → `SMMU_EVT_F_TRANSLATION`
- `SMMU_PTW_ERR_ADDR_SIZE` → `SMMU_EVT_F_ADDR_SIZE`
- `SMMU_PTW_ERR_ACCESS` → `SMMU_EVT_F_ACCESS`
- `SMMU_PTW_ERR_PERMISSION` → `SMMU_EVT_F_PERMISSION`

---

## 7. 页表遍历（PTW）

### 7.1 PTW 错误类型

```c
// include/hw/arm/smmu-common.h:46-53
typedef enum {
    SMMU_PTW_ERR_NONE,
    SMMU_PTW_ERR_WALK_EABT,   // 遍历外部中止（DMA 读取失败）
    SMMU_PTW_ERR_TRANSLATION, // 翻译故障（无效描述符）
    SMMU_PTW_ERR_ADDR_SIZE,   // 地址超出范围
    SMMU_PTW_ERR_ACCESS,      // 访问标志故障（AF=0）
    SMMU_PTW_ERR_PERMISSION,  // 权限故障
} SMMUPTWEventType;
```

### 7.2 翻译配置

```c
// include/hw/arm/smmu-common.h:101-119
typedef struct SMMUTransCfg {
    SMMUStage stage;         // SMMU_STAGE_1 / SMMU_STAGE_2 / SMMU_NESTED
    bool disabled;
    bool bypassed;
    bool aborted;
    bool affd;               // AF Fault Disable
    uint32_t iotlb_hits;
    uint32_t iotlb_misses;
    // Stage-1 专用
    bool aa64;
    bool record_faults;
    uint8_t oas;             // 输出地址宽度
    uint8_t tbi;
    int asid;
    SMMUTransTableInfo tt[2]; // TTB0/TTB1
    // Stage-2 专用
    struct SMMUS2Cfg s2cfg;
} SMMUTransCfg;
```

### 7.3 smmu_ptw() — PTW 调度器

```c
// smmu-common.c:730-770
int smmu_ptw(SMMUState *bs, SMMUTransCfg *cfg, dma_addr_t iova,
             IOMMUAccessFlags perm, SMMUTLBEntry *tlbe, SMMUPTWEventInfo *info)
```

```
smmu_ptw(cfg, iova)
  │
  ├── stage == STAGE_1 → smmu_ptw_64_s1()
  ├── stage == STAGE_2 → 地址范围检查 → smmu_ptw_64_s2()
  └── stage == NESTED
      ├── smmu_ptw_64_s1(iova) → S1 TLB Entry
      ├── ipa = CACHED_ENTRY_TO_ADDR(s1_entry, iova)
      ├── smmu_ptw_64_s2(ipa) → S2 TLB Entry
      └── combine_tlb(s1, s2, iova) → 合并结果
```

### 7.4 smmu_ptw_64_s1() — Stage-1 页表遍历

```c
// smmu-common.c:458-571
static int smmu_ptw_64_s1(SMMUState *bs, SMMUTransCfg *cfg,
                          dma_addr_t iova, IOMMUAccessFlags perm,
                          SMMUTLBEntry *tlbe, SMMUPTWEventInfo *info)
```

**遍历步骤**：

1. **选择翻译表**：`select_tt(cfg, iova)` — 根据 IOVA 高位选择 TTB0 或 TTB1
2. **计算起始级别**：`level = 4 - (inputsize - 4) / stride`
3. **逐级遍历**：
   ```
   while (level < 4):
     offset = iova_level_offset(iova, inputsize, level, granule_sz)
     pte = get_pte(baseaddr, offset)
     
     if invalid/reserved → TRANSLATION fault
     if table_pte:
       检查 APTable 权限（had=0 时）
       baseaddr = get_table_pte_address(pte)
       [Nested] → translate_table_addr_ipa() 将 IPA 翻译为 PA
       level++
     if page_pte:
       gpa = get_page_pte_address(pte)
     if block_pte:
       gpa = get_block_pte_address(pte)
     
     检查 AF（Access Flag）
     检查 AP（Access Permission）
     检查地址范围（gpa < 2^oas）
     
     填充 tlbe
   ```

4. **Nested 表地址翻译**：当 `cfg->stage == SMMU_NESTED` 时，每个中间表地址（table PTE 指向的下一级表基地址）都需要通过 Stage-2 翻译：
   ```c
   if (cfg->stage == SMMU_NESTED) {
       if (translate_table_addr_ipa(bs, &baseaddr, cfg, info)) {
           goto error;
       }
   }
   ```

### 7.5 smmu_ptw_64_s2() — Stage-2 页表遍历

```c
// smmu-common.c:587-695
static int smmu_ptw_64_s2(SMMUTransCfg *cfg, dma_addr_t ipa,
                          IOMMUAccessFlags perm, SMMUTLBEntry *tlbe, ...)
```

Stage-2 遍历与 Stage-1 类似，但有以下区别：

1. **起始级别**由 SL0 字段决定：`level = get_start_level(s2cfg.sl0, granule_sz)`
2. **支持拼接页表**：`idx = pgd_concat_idx(level, granule_sz, ipa)` — 最多 16 个拼接表
3. **输入范围检查**：`ipa >= (1ULL << inputsize)` → Translation fault
4. **S2 权限模型**：使用 `is_permission_fault_s2(s2ap, perm)` 检查
5. **输出地址检查**：`gpa >= (1ULL << eff_ps)` → Address Size fault
6. **无表地址 S2 翻译**：Stage-2 表地址是 PA，无需再翻译

### 7.6 combine_tlb() — 合并 S1+S2 翻译结果

```c
// smmu-common.c:701-716
static void combine_tlb(SMMUTLBEntry *tlbe, SMMUTLBEntry *tlbe_s2,
                        dma_addr_t iova, SMMUTransCfg *cfg)
```

Nested 翻译的最终结果是 S1 和 S2 TLB 条目的合并：
- **地址掩码**：取 S1 和 S2 中较小的（更细粒度）
- **翻译地址**：S2 翻译 S1 的输出地址
- **权限**：`parent_perm` 保存 S2 权限，`perm` 保存 S1 权限

---

## 8. IOTLB 缓存管理

### 8.1 IOTLB 键与哈希

```c
// include/hw/arm/smmu-common.h:137-143
typedef struct SMMUIOTLBKey {
    uint64_t iova;
    int asid;
    int vmid;
    uint8_t tg;      // 翻译颗粒度
    uint8_t level;    // 页表级别
} SMMUIOTLBKey;
```

哈希使用 Jenkins Hash（smmu-common.c:35-50），比较时需要所有 5 个字段完全匹配。

### 8.2 IOTLB 查找

```c
// smmu-common.c:109-139
SMMUTLBEntry *smmu_iotlb_lookup(SMMUState *bs, SMMUTransCfg *cfg,
                                SMMUTransTableInfo *tt, hwaddr iova)
```

查找策略是**逐级尝试**：从计算出的起始级别到 level 3，依次在每个级别查找匹配的 TLB 条目（smmu-common.c:70-95 `smmu_iotlb_lookup_all_levels`）。这确保了块映射（如 2MB/1GB）也能被命中。

**Nested 颗粒度回退**：当 Nested 模式下初始颗粒度未命中时，尝试 S2 颗粒度：
```c
if (!entry && (cfg->stage == SMMU_NESTED) &&
    (cfg->s2cfg.granule_sz != tt->granule_sz)) {
    tt->granule_sz = cfg->s2cfg.granule_sz;
    entry = smmu_iotlb_lookup_all_levels(bs, cfg, tt, iova);
}
```

### 8.3 IOTLB 插入

```c
// smmu-common.c:141-155
void smmu_iotlb_insert(SMMUState *bs, SMMUTransCfg *cfg, SMMUTLBEntry *new)
```

- 最大容量 256 条（`SMMU_IOTLB_MAX_SIZE`，smmu-common.h:224）
- 容量满时全部清空 → `smmu_iotlb_inv_all()`
- 键由 (ASID, VMID, IOVA, TG, Level) 构成

### 8.4 IOTLB 失效

提供多种粒度的失效操作：

| 函数 | 粒度 | 触发命令 |
|------|------|----------|
| `smmu_iotlb_inv_all()` | 全部清空 | TLBI_NSNH_ALL |
| `smmu_iotlb_inv_asid_vmid()` | 按 ASID+VMID | TLBI_NH_ASID |
| `smmu_iotlb_inv_vmid()` | 按 VMID | TLBI_S12_VMALL |
| `smmu_iotlb_inv_vmid_s1()` | 按 VMID（仅 S1） | TLBI_NH_ALL |
| `smmu_iotlb_inv_iova()` | 按 ASID+VMID+IOVA | TLBI_NH_VA |
| `smmu_iotlb_inv_ipa()` | 按 VMID+IPA（S2） | TLBI_S2_IPA |

### 8.5 smmu_translate() — TLB 查找 + PTW

```c
// smmu-common.c:772-822
SMMUTLBEntry *smmu_translate(SMMUState *bs, SMMUTransCfg *cfg, dma_addr_t addr,
                             IOMMUAccessFlags flag, SMMUPTWEventInfo *info)
```

完整翻译流程：
1. 确定翻译表参数（S2 用 s2cfg 颗粒度，S1 用 select_tt 选择）
2. `smmu_iotlb_lookup()` — TLB 命中则检查写权限后返回
3. TLB 未命中 → `smmu_ptw()` 执行页表遍历
4. 遍历成功 → `smmu_iotlb_insert()` 插入缓存
5. 返回翻译结果

---

## 9. 命令队列（CMDQ）

### 9.1 队列结构

```c
// include/hw/arm/smmuv3.h:28-34
typedef struct SMMUQueue {
    uint64_t base;        // 队列基地址寄存器
    uint32_t prod;        // 生产者索引
    uint32_t cons;        // 消费者索引
    uint8_t entry_size;   // 条目大小
    uint8_t log2size;     // 队列大小 = 2^log2size
} SMMUQueue;
```

### 9.2 命令类型

```c
// smmuv3-internal.h:135-161
// 支持的命令类型
SMMU_CMD_PREFETCH_CONFIG    // 预取 STE 配置
SMMU_CMD_PREFETCH_ADDR      // 预取地址翻译
SMMU_CMD_CFGI_STE           // 失效单个 STE 配置
SMMU_CMD_CFGI_STE_RANGE     // 失效 STE 范围
SMMU_CMD_CFGI_CD            // 失效单个 CD 配置
SMMU_CMD_CFGI_CD_ALL        // 失效所有 CD
SMMU_CMD_TLBI_NH_ALL        // 失效所有非安全 TLB（按 VMID）
SMMU_CMD_TLBI_NH_ASID       // 失效按 ASID 的 TLB
SMMU_CMD_TLBI_NH_VA         // 失效按 VA 的 TLB
SMMU_CMD_TLBI_NH_VAA        // 失效按 VA（忽略 ASID）
SMMU_CMD_TLBI_S12_VMALL     // 失效 S1+S2 按 VMID
SMMU_CMD_TLBI_S2_IPA        // 失效 S2 按 IPA
SMMU_CMD_TLBI_NSNH_ALL      // 失效所有非安全 TLB
SMMU_CMD_ATC_INV            // ATS 缓存失效
SMMU_CMD_SYNC               // 同步命令
```

### 9.3 命令字段提取宏

```c
// smmuv3-internal.h:187-207
#define CMD_TYPE(x)      extract32((x)->word[0], 0, 8)     // 命令类型
#define CMD_SID(x)       ((x)->word[1])                     // Stream ID
#define CMD_VMID(x)      extract32((x)->word[1], 0, 16)     // VMID
#define CMD_ASID(x)      extract32((x)->word[1], 16, 16)    // ASID
#define CMD_LEAF(x)      extract32((x)->word[2], 0, 1)      // 叶节点标志
#define CMD_TTL(x)       extract32((x)->word[2], 8, 2)      // TTL
#define CMD_TG(x)        extract32((x)->word[2], 10, 2)     // 翻译颗粒度
#define CMD_NUM(x)       extract32((x)->word[0], 12, 5)     // RIL: 页数
#define CMD_SCALE(x)     extract32((x)->word[0], 20, 5)     // RIL: 比例
#define CMD_ADDR(x)      // 48 位地址，跨 word[2:3]
```

### 9.4 smmuv3_cmdq_consume() — 命令处理循环

```c
// smmuv3.c:1308-1583
static int smmuv3_cmdq_consume(SMMUv3State *s, Error **errp)
```

**主循环**（smmuv3.c:1325-1571）：
```
while (!queue_empty(cmdq)):
  检查 GERROR.CMDQ_ERR 标志
  读取命令 → queue_read()
  mutex_lock
  
  switch (CMD_TYPE):
    SYNC:         → smmuv3_trigger_irq(CMD_SYNC)
    PREFETCH_*:   → 空操作（QEMU 不需要预取）
    CFGI_STE:     → smmuv3_flush_config(sdev) + accel_install_ste
    CFGI_STE_RANGE: → smmu_configs_inv_sid_range()
    CFGI_CD:      → smmuv3_flush_config() + accel_issue_inv_cmd
    
    TLBI_NH_ASID: → smmu_inv_notifiers_all() + smmu_iotlb_inv_asid_vmid()
    TLBI_NH_ALL:  → smmu_iotlb_inv_vmid_s1()（S2 支持时）
    TLBI_NSNH_ALL: → smmu_iotlb_inv_all()
    TLBI_NH_VA/VAA: → smmuv3_range_inval(STAGE_1)
    TLBI_S12_VMALL: → smmu_iotlb_inv_vmid()
    TLBI_S2_IPA:  → smmuv3_range_inval(STAGE_2)
    ATC_INV:      → smmuv3_accel_issue_inv_cmd()
  
  mutex_unlock
  queue_cons_incr()  // 命令完成后才递增消费者索引

if (cmd_error):
  smmu_write_cmdq_err(CERROR_ILL/ABT)
  smmuv3_trigger_irq(GERROR, CMDQ_ERR)
```

### 9.5 范围失效 — smmuv3_range_inval()

```c
// smmuv3.c:1249-1306
static void smmuv3_range_inval(SMMUState *s, Cmd *cmd, SMMUStage stage)
```

支持两种模式：
- **非 RIL 模式**（tg=0）：单页失效
- **RIL 模式**（tg≠0）：范围失效 — `num_pages = (NUM+1) * 2^SCALE`，颗粒度 = tg*2+10 位

范围失效将大范围拆分为 2 的幂次对齐的子范围，对每个子范围分别调用 notifier 和 IOTLB 失效。

---

## 10. 事件队列与错误报告

### 10.1 事件类型

```c
// smmuv3-internal.h:213-233
typedef enum SMMUEventType {
    SMMU_EVT_NONE               = 0x00,
    SMMU_EVT_F_UUT,                       // 未翻译事务
    SMMU_EVT_C_BAD_STREAMID,              // 无效 Stream ID
    SMMU_EVT_F_STE_FETCH,                 // STE 获取失败
    SMMU_EVT_C_BAD_STE,                   // 无效 STE
    SMMU_EVT_F_BAD_ATS_TREQ,             // ATS 翻译请求错误
    SMMU_EVT_F_STREAM_DISABLED,           // 流被禁用
    SMMU_EVT_F_TRANS_FORBIDDEN,           // 翻译被禁止
    SMMU_EVT_C_BAD_SUBSTREAMID,           // 无效 SubstreamID
    SMMU_EVT_F_CD_FETCH,                  // CD 获取失败
    SMMU_EVT_C_BAD_CD,                    // 无效 CD
    SMMU_EVT_F_WALK_EABT,                 // 页表遍历外部中止
    SMMU_EVT_F_TRANSLATION      = 0x10,   // 翻译故障
    SMMU_EVT_F_ADDR_SIZE,                 // 地址大小故障
    SMMU_EVT_F_ACCESS,                    // 访问标志故障
    SMMU_EVT_F_PERMISSION,                // 权限故障
    SMMU_EVT_F_TLB_CONFLICT     = 0x20,   // TLB 冲突
    SMMU_EVT_F_CFG_CONFLICT,              // 配置冲突
    SMMU_EVT_E_PAGE_REQ         = 0x24,   // 页请求事件
} SMMUEventType;
```

### 10.2 事件记录

```c
// smmuv3.c:185-269
void smmuv3_record_event(SMMUv3State *s, SMMUEventInfo *info)
```

根据事件类型填充 Evt 结构体（256 位 = 8 个 word），设置 SID、SSID、SSV、地址、读写标志、权限标志等字段，然后调用 `smmuv3_propagate_event()` 写入事件队列。

### 10.3 事件传播

```c
// smmuv3.c:172-183
void smmuv3_propagate_event(SMMUv3State *s, Evt *evt)
```

1. 加锁（mutex）
2. `smmuv3_write_eventq()` → 写入事件队列
3. 队列满 → 触发 `GERROR_EVENTQ_ABT_ERR`（全局错误中断）
4. 队列非空 → 触发 `SMMU_IRQ_EVTQ`（事件队列中断）

---

## 11. 中断机制

### 11.1 smmuv3_trigger_irq()

```c
// smmuv3.c:53-89
static void smmuv3_trigger_irq(SMMUv3State *s, SMMUIrq irq, uint32_t gerror_mask)
```

**四种中断**：

| IRQ | 触发条件 | 脉冲控制 |
|-----|----------|----------|
| `SMMU_IRQ_EVTQ` | 事件队列非空 | `IRQ_CTRL.EVENTQ_IRQEN` |
| `SMMU_IRQ_PRIQ` | PRI 队列（未实现） | — |
| `SMMU_IRQ_CMD_SYNC` | CMD_SYNC 命令完成 | 始终脉冲 |
| `SMMU_IRQ_GERROR` | 全局错误 | `IRQ_CTRL.GERROR_IRQEN` |

**GERROR 特殊处理**：
```c
uint32_t pending = s->gerror ^ s->gerrorn;    // 当前待处理的错误
uint32_t new_gerrors = ~pending & gerror_mask; // 仅切换未处理的新错误
s->gerror ^= new_gerrors;                       // 切换错误位
```

Guest 通过写 GERRORN 来确认已处理的错误（smmuv3.c:91-109 `smmuv3_write_gerrorn`）。

---

## 12. IOMMU Notifier 与 DMA 集成

### 12.1 smmuv3_notify_flag_changed()

```c
// smmuv3.c:2188-2218
static int smmuv3_notify_flag_changed(IOMMUMemoryRegion *iommu,
                                      IOMMUNotifierFlag old,
                                      IOMMUNotifierFlag new, Error **errp)
```

**当前限制**：
- 不支持 `IOMMU_NOTIFIER_DEVIOTLB_UNMAP`（设备 IOTLB 失效）
- 不支持 `IOMMU_NOTIFIER_MAP`（映射通知）
- 仅支持 `IOMMU_NOTIFIER_UNMAP`（失效通知）

当第一个通知器注册时，将 SMMUDevice 加入 `devices_with_notifiers` 链表。

### 12.2 smmuv3_notify_iova() — 通知器分发

```c
// smmuv3.c:1168-1227
static void smmuv3_notify_iova(IOMMUMemoryRegion *mr, IOMMUNotifier *n,
                               int asid, int vmid, dma_addr_t iova,
                               uint8_t tg, uint64_t num_pages, int stage)
```

- 获取设备配置（`smmuv3_get_config`）
- Nested 模式下仅通知 Stage-1 地址（Stage-2 失效不通知）
- 检查 ASID/VMID 匹配
- 构造 `IOMMUTLBEvent`（UNMAP 类型）
- 调用 `memory_region_notify_iommu_one()` 通知每个监听器

### 12.3 smmuv3_inv_notifiers_iova() — 批量通知

```c
// smmuv3.c:1230-1247
static void smmuv3_inv_notifiers_iova(SMMUState *s, int asid, int vmid,
                                      dma_addr_t iova, uint8_t tg,
                                      uint64_t num_pages, int stage)
```

遍历所有 `devices_with_notifiers` 中的设备，对每个设备的每个通知器调用 `smmuv3_notify_iova()`。

### 12.4 DMA 路径集成

当 PCI 设备（如 VirtIO）发起 DMA 时：
```
设备 DMA 写 → AddressSpace → FlatView → IOMMUMemoryRegion
  → smmuv3_translate(iova) → 物理地址
  → MemoryRegion（RAM/MMIO）
```

QEMU 内存子系统在 `address_space_translate_internal()` 中检测到 IOMMUMemoryRegion 后，调用其 `translate` 方法完成地址翻译。

---

## 13. virt 机器集成

### 13.1 SMMUv3 创建

```c
// hw/arm/virt.c:1850-1881
static void create_smmu(const VirtMachineState *vms, PCIBus *bus)
```

**创建流程**：
1. `qdev_new(TYPE_ARM_SMMUV3)` — 创建 SMMUv3 设备
2. 设置 `stage` = "nested"（默认，除非 `vmc->no_nested_smmu`）
3. 关联 `primary-bus` → PCIe 根总线
4. 关联 `memory` → 系统内存，`secure-memory` → 安全内存
5. `sysbus_realize_and_unref()` → 触发 `smmu_realize()`
6. MMIO 映射到 `VIRT_SMMU` 区域
7. 连接 4 个 IRQ 到 GIC

### 13.2 smmu_realize() — 设备实现

```c
// smmuv3.c:2016-2056
static void smmu_realize(DeviceState *d, Error **errp)
```

1. 属性验证
2. 加速器初始化（如 `accel=on`）+ 迁移阻止器
3. 父类 realize（PCI IOMMU 注册）
4. 初始化互斥锁
5. 创建 MMIO 区域（0x20000 大小）
6. 设置 `mrtypename = TYPE_SMMUV3_IOMMU_MEMORY_REGION`
7. 注册 sysbus MMIO 和 IRQ
8. 初始化 ID 寄存器

### 13.3 设备树绑定

```c
// hw/arm/virt.c:1822-1848
static void create_smmuv3_dev_dtb(VirtMachineState *vms, DeviceState *dev, ...)
```

生成以下设备树属性：
- `compatible = "arm,smmu-v3"`
- `#iommu-cells = <1>`
- PCI Host Bridge 的 `iommu-map = <0x0 smmu_phandle 0x0 0x10000>` — 映射所有 BDF 到 SMMU

### 13.4 命令行使用

典型命令行配置：
```bash
-device virtio-blk-pci,iommu_platform=on  # 启用 VirtIO 设备 IOMMU 支持
# virt 机器默认创建 SMMUv3（如果选择了 SMMU IOMMU 类型）
```

---

## 总结

### 翻译路径完整调用链

```
PCI 设备 DMA 请求
  │
  ├── AddressSpace → IOMMUMemoryRegion
  │
  └── smmuv3_translate(mr, iova, flag)         [smmuv3.c:1065]
      ├── smmuv3_get_config(sdev)               [smmuv3.c:898]
      │   └── smmuv3_decode_config(mr, cfg)     [smmuv3.c:851]
      │       ├── smmu_find_ste(sid)             [smmuv3.c:660]
      │       │   └── 线性/2级 Stream Table 查找
      │       ├── decode_ste(cfg, ste)           [smmuv3.c:573]
      │       │   └── decode_ste_s2_cfg()        [smmuv3.c:456]
      │       ├── smmu_get_cd(ste, cfg)          [smmuv3.c:378]
      │       │   └── [Nested] smmuv3_do_translate(CD addr, S2)
      │       └── decode_cd(cfg, cd)             [smmuv3.c:742]
      │           └── [Nested] smmuv3_do_translate(TTB addr, S2)
      │
      └── smmuv3_do_translate(iova, cfg, CLASS_IN) [smmuv3.c:939]
          └── smmu_translate(cfg, iova)          [smmu-common.c:772]
              ├── smmu_iotlb_lookup()            [smmu-common.c:109]
              │   └── smmu_iotlb_lookup_all_levels() [逐级查找]
              └── [未命中] smmu_ptw(cfg, iova)   [smmu-common.c:730]
                  ├── [S1] smmu_ptw_64_s1()      [smmu-common.c:458]
                  ├── [S2] smmu_ptw_64_s2()      [smmu-common.c:587]
                  └── [Nested] S1 → IPA → S2 → PA → combine_tlb()
```

### 架构层次

```
┌──────────────────────────────────────────────┐
│  Guest OS (Linux SMMU 驱动)                  │
│  - 配置 Stream Table / Context Descriptor    │
│  - 发送 CMDQ 命令（TLBI/CFGI/SYNC）         │
│  - 处理 EVTQ 事件                            │
├──────────────────────────────────────────────┤
│  SMMUv3 设备模型 (smmuv3.c)                  │
│  - 寄存器读写                                │
│  - 命令队列处理 (smmuv3_cmdq_consume)        │
│  - 翻译入口 (smmuv3_translate)               │
│  - 事件记录 (smmuv3_record_event)            │
│  - 中断触发 (smmuv3_trigger_irq)             │
├──────────────────────────────────────────────┤
│  SMMU 通用层 (smmu-common.c)                 │
│  - 页表遍历 (smmu_ptw_64_s1/s2)             │
│  - TLB 缓存 (smmu_iotlb_lookup/insert)      │
│  - 翻译调度 (smmu_translate)                 │
├──────────────────────────────────────────────┤
│  QEMU IOMMU 框架 (system/memory.c)           │
│  - IOMMUMemoryRegion 接口                    │
│  - Notifier 管理与分发                        │
│  - 内存子系统地址翻译集成                     │
├──────────────────────────────────────────────┤
│  PCI 设备 DMA (VirtIO/VFIO)                  │
│  - 通过 AddressSpace 发起 DMA                │
│  - IOMMU 翻译透明处理                        │
└──────────────────────────────────────────────┘
```
