# ARM64 内存属性与缓存一致性深度分析：MAIR/TCR 属性编码、Device/Normal 类型、缓存维护指令

> 基于 QEMU 11.0.50 源码分析，涵盖 ARM64 内存属性与缓存一致性完整子系统：
> MAIR 属性编码（MAIR_EL1/EL2/EL3 存储 cp15.mair_el[4]、AttrIndx×8 位提取→8 位 MAIR 字段、
> 高 4 位 Outer+低 4 位 Inner 编码、0x00=Device-nGnRnE/0x04=Device-nGnRE/0x44=Normal-NC/
> 0xee=Normal-WT-RA/0xff=Normal-WB-RWA/0xf0=Tagged-WB-RWA）、
> TCR 属性（TCR_EL1/EL2/EL3 存储 cp15.tcr_el[4]、T0SZ/T1SZ VA 范围、TG0/TG1 粒度、
> SH0/SH1 共享性、ORGN0/ORGN1+IRGN0/IRGN1 PTW 缓存属性、EPD0/EPD1 翻译禁用、
> IPS/PS 物理地址大小、aa64_va_parameters 解码）、
> 内存类型分类（S1_attrs_are_device 高 4 位=0→Device 四子类型 nGnRnE/nGnRE/nGRE/GRE、
> Normal 六子类型 NC/WT-RA/WT-WA/WB-RA/WB-WA/WB-RWA+Inner×Outer 组合、
> ARMCacheAttrs attrs:8+shareability:2+is_s2_format:1）、
> SCTLR 内存位（SCTLR_M bit0 MMU 使能、SCTLR_A bit1 对齐检查、SCTLR_C bit2 数据缓存、
> SCTLR_I bit12 指令缓存、SCTLR_WXN bit19 写执行互斥、sctlr_write→tlb_flush）、
> HCR_EL2 内存位（HCR_DC Stage-1 禁用时默认 WB-RWA 缓存、HCR_CD Stage-2 缓存禁用→NC）、
> MemTxAttrs 事务属性（secure/space/user/memory/debug 五字段、PTW→result.f.attrs 传播）、
> MMU 禁用时属性（get_phys_addr_disabled：物理直通 Device-nGnRnE、
> HCR_DC→Normal-WB-RWA/DCT→Tagged、SCTLR_I→WT-RA 否则 NC、shareability=Outer）、
> S1+S2 缓存属性合并（combine_cacheattrs：共享性取最强、
> FWB 路径 combined_attrs_fwb S2=7 用 S1/S2=6 强制 WB/S2=5 Device 保留否则 NC/S2=0-3 强制 Device、
> 非 FWB combined_attrs_nofwb 取更弱类型、Device 或 NC 强制 Outer Shareable）、
> 缓存维护指令（IC IALLUIS/IALLU/IVAU 全部 NOP 除 user-mode IC_IVAU→tb_invalidate、
> DC IVAC/CVAC/CVAU/CIVAC 全部 NOP、DC ZVA 实际执行→gen_helper_dc_zva 零填充、
> DCZID_EL0 块大小+DZP 禁用位）、
> 内存屏障（DMB/DSB→tcg_gen_mb TCG_BAR_SC+MO_LD/ST/ALL、ISB→TB 结束+中断检查）。

---

## 目录

1. [架构概述](#1-架构概述)
2. [MAIR 属性编码](#2-mair-属性编码)
3. [TCR 翻译控制寄存器](#3-tcr-翻译控制寄存器)
4. [内存类型分类](#4-内存类型分类)
5. [ARMCacheAttrs 结构](#5-armcacheattrs-结构)
6. [SCTLR 内存控制位](#6-sctlr-内存控制位)
7. [HCR_EL2 内存控制位](#7-hcr_el2-内存控制位)
8. [MemTxAttrs 事务属性](#8-memtxattrs-事务属性)
9. [MMU 禁用时的属性](#9-mmu-禁用时的属性)
10. [S1+S2 缓存属性合并](#10-s1s2-缓存属性合并)
11. [缓存维护指令](#11-缓存维护指令)
12. [DC ZVA 零填充](#12-dc-zva-零填充)
13. [内存屏障指令](#13-内存屏障指令)
14. [QEMU TCG 内存模型](#14-qemu-tcg-内存模型)
15. [完整属性流总结](#15-完整属性流总结)

---

## 1. 架构概述

ARM64 内存属性系统决定每次内存访问的类型（Device/Normal）、缓存策略（WB/WT/NC）和共享性（Non/Inner/Outer Shareable）。QEMU 通过软件模拟属性计算，但不模拟实际缓存行为。

```
页表描述符 AttrIndx[2:0]
    │
    ▼
MAIR_ELx 8位属性 ──→ ARMCacheAttrs.attrs
    │                      │
    ▼                      ▼
S1_attrs_are_device?   combine_cacheattrs(S1, S2)
    │                      │
    ▼                      ▼
Device → 对齐检查      最终属性 → MemTxAttrs
Normal → 缓存策略                  → TLB 条目
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `cpu.h` | 469-470 | cp15.mair_el[4] |
| `cpu.h` | 368-369 | cp15.tcr_el[4] |
| `cpu.h` | 1398-1435 | SCTLR_M/A/C/I/WXN 定义 |
| `internals.h` | 1465-1474 | ARMCacheAttrs |
| `memattrs.h` | 25-71 | MemTxAttrs |
| `ptw.c` | 574-582 | S1_attrs_are_device |
| `ptw.c` | 2334-2346 | MAIR AttrIndx 提取 |
| `ptw.c` | 3366-3463 | combine_cacheattrs / combined_attrs_fwb |
| `ptw.c` | 3478-3551 | get_phys_addr_disabled |
| `helper.c` | 3484-3575 | IC/DC 缓存维护寄存器定义 |
| `translate-a64.c` | 2243-2281 | DMB/DSB/ISB 翻译 |
| `translate-a64.c` | 2996-3012 | DC ZVA 翻译 |

---

## 2. MAIR 属性编码

### 寄存器定义

```c
// helper.c:990-1002 — MAIR_EL1/EL3 定义
{ .name = "MAIR_EL1", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 0, .crn = 10, .crm = 2, .opc2 = 0,
  .access = PL1_RW, .fieldoffset = offsetof(CPUARMState, cp15.mair_el[1]),
  .resetvalue = 0 },
// MAIR_EL3: .fieldoffset = cp15.mair_el[3]

// cpu.h:469-470
uint64_t mair_el[4];  // 索引: [1]=EL1, [2]=EL2, [3]=EL3
```

### AttrIndx → 8 位属性

```c
// ptw.c:2334-2340 — LPAE 遍历中 MAIR 提取
attrindx = extract32(attrs, 2, 3);           // 描述符 bit[4:2]
mair = (param.aie && extract64(attrs, 59, 1)  // FEAT_AIE: MAIR2 选择
        ? env->cp15.mair2_el[el]
        : env->cp15.mair_el[el]);
result->cacheattrs.attrs = extract64(mair, attrindx * 8, 8);
```

### 8 位 MAIR 字段编码

```
MAIR 字段格式: [Outer:4][Inner:4]

Device 类型（高 4 位 = 0000）:
  0b0000_00_00 = Device-nGnRnE (0x00)  最严格: 不聚合、不重排、不提前
  0b0000_01_00 = Device-nGnRE  (0x04)  允许提前
  0b0000_10_00 = Device-nGRE   (0x08)  允许重排+提前
  0b0000_11_00 = Device-GRE    (0x0C)  允许聚合+重排+提前

Normal 类型（高 4 位 ≠ 0000）:
  每个 nibble (Inner/Outer):
    0b0000 = 不可用（保留，除非整个字节为 Device）
    0b0100 = Non-cacheable               (NC)
    0b10RW = Write-Through, RA=R, WA=W   (WT)
    0b11RW = Write-Back, RA=R, WA=W      (WB)

常见组合:
  0x00 = Device-nGnRnE
  0x04 = Device-nGnRE
  0x44 = Normal, Non-cacheable
  0xaa = Normal, Write-Through, Read-Allocate
  0xee = Normal, Write-Through, Read-Allocate, No-Transient
  0xff = Normal, Write-Back, Read+Write Allocate
  0xf0 = Tagged Normal, Write-Back, RWA (FEAT_MTE)
```

---

## 3. TCR 翻译控制寄存器

```c
// helper.c:2870-2880 — TCR_EL1 定义
{ .name = "TCR_EL1", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 0, .crn = 2, .crm = 0, .opc2 = 2,
  .access = PL1_RW, .writefn = vmsa_tcr_el12_write, ... }

// cpu.h:368-369
uint64_t tcr_el[4];  // [1]=EL1, [2]=EL2, [3]=EL3
```

### TCR 关键字段

| 字段 | 位域 | 含义 |
|------|------|------|
| T0SZ | [5:0] | TTBR0 VA 范围 (64 - T0SZ 位) |
| T1SZ | [21:16] | TTBR1 VA 范围 |
| TG0 | [15:14] | TTBR0 粒度 (00=4KB, 01=64KB, 10=16KB) |
| TG1 | [31:30] | TTBR1 粒度 |
| SH0 | [13:12] | TTBR0 PTW 共享性 |
| SH1 | [29:28] | TTBR1 PTW 共享性 |
| ORGN0 | [11:10] | TTBR0 PTW Outer 缓存属性 |
| IRGN0 | [9:8] | TTBR0 PTW Inner 缓存属性 |
| ORGN1 | [27:26] | TTBR1 PTW Outer 缓存属性 |
| IRGN1 | [25:24] | TTBR1 PTW Inner 缓存属性 |
| EPD0 | [7] | 禁用 TTBR0 翻译 |
| EPD1 | [23] | 禁用 TTBR1 翻译 |
| IPS/PS | [34:32] | 物理地址大小 |
| A1 | [22] | ASID 选择 (TTBR0 或 TTBR1) |
| AS | [36] | ASID 大小 (8 或 16 位) |

**注意**：QEMU 在 PTW 中忽略 ORGN/IRGN（不模拟缓存），仅用于属性计算。

---

## 4. 内存类型分类

### Device 检测

```c
// ptw.c:574-582
static bool S1_attrs_are_device(uint8_t attrs)
{
    // MAIR 字段高 4 位 = 0 → Device 类型
    // 0b0000dd01 (FEAT_XS) 也是 Device
    return (attrs & 0xf0) == 0;
}
```

### Device 四子类型

| 编码 | 类型 | 聚合 | 重排 | 提前 |
|------|------|------|------|------|
| 0x00 | nGnRnE | ✗ | ✗ | ✗ |
| 0x04 | nGnRE | ✗ | ✗ | ✓ |
| 0x08 | nGRE | ✗ | ✓ | ✓ |
| 0x0C | GRE | ✓ | ✓ | ✓ |

**QEMU 影响**：Device 内存强制对齐检查、不可缓存、Outer Shareable。

### Normal 缓存策略

| Inner/Outer nibble | 策略 |
|-------|--------|
| 0100 (4) | Non-cacheable |
| 1010 (A) | Write-Through, Read-Allocate |
| 1110 (E) | Write-Through, RA, Non-Transient |
| 1011 (B) | Write-Back, Read-Allocate |
| 1111 (F) | Write-Back, Read+Write Allocate |

---

## 5. ARMCacheAttrs 结构

```c
// internals.h:1465-1474
typedef struct ARMCacheAttrs {
    unsigned int attrs:8;           // MAIR 8 位格式（或 S2 描述符 [5:2]）
    unsigned int shareability:2;    // 0=Non, 2=Outer, 3=Inner
    bool is_s2_format:1;            // true = S2 4 位格式, false = MAIR 8 位
} ARMCacheAttrs;
```

---

## 6. SCTLR 内存控制位

```c
// cpu.h:1398-1435
#define SCTLR_M       (1U << 0)   // MMU 使能
#define SCTLR_A       (1U << 1)   // 对齐检查使能
#define SCTLR_C       (1U << 2)   // 数据缓存使能
#define SCTLR_I       (1U << 12)  // 指令缓存使能
#define SCTLR_WXN     (1U << 19)  // 可写→不可执行
#define SCTLR_DZE     (1U << 14)  // EL0 DC ZVA 使能
```

### SCTLR 写入处理

```c
// helper.c:3265-3297 — sctlr_write
// SCTLR_M 变化 → tlb_flush()（全 TLB 刷新）
// PMSA/无 MPU: SCTLR_M 强制为 0
// 总是 rebuild_hflags → 重建翻译标志
```

### SCTLR 对 PTW 的影响

| 位 | 影响 |
|------|------|
| SCTLR_M=0 | MMU 禁用 → get_phys_addr_disabled |
| SCTLR_C=0 | 数据缓存禁用（QEMU 不模拟） |
| SCTLR_I=1 | 指令缓存启用 → MMU 禁用时 WT-RA 属性 |
| SCTLR_I=0 | 指令缓存禁用 → MMU 禁用时 NC 属性 |
| SCTLR_WXN=1 | 可写页面自动不可执行 |

---

## 7. HCR_EL2 内存控制位

```c
// cpu.h:1706,1726
#define HCR_DC    (1ULL << 12)  // 默认缓存性
#define HCR_CD    (1ULL << 30)  // Stage-2 缓存禁用
```

### HCR_DC 效果

```c
// ptw.c:294-303 — Stage-1 禁用判定
if (r_el == 1 && (hcr & HCR_DC)) {
    // HCR_DC=1 使 SCTLR_EL1.M 表现为 0
    // 但不是真正禁用 → 强制 Normal WB-RWA 属性
}

// ptw.c:3522-3530 — 禁用时默认缓存属性
if (r_el == 1 && (hcr & HCR_DC)) {
    if (hcr & HCR_DCT)
        memattr = 0xf0;   // Tagged, Normal, WB, RWA
    else
        memattr = 0xff;   // Normal, WB, RWA
}
```

### HCR_CD 效果

```c
// ptw.c:3255-3268 — Stage-2 缓存禁用
// HCR_CD=1 → Stage-2 Normal 内存强制 Non-cacheable
// Device 内存不受影响
```

---

## 8. MemTxAttrs 事务属性

```c
// include/exec/memattrs.h:25-71
typedef struct MemTxAttrs {
    unsigned int secure:1;          // 30: TrustZone 安全访问
    unsigned int space:2;           // 36: ArmSecuritySpace (S/NS/Root/Realm)
    unsigned int user:1;            // 38: 用户态（非特权）
    unsigned int memory:1;          // 46: 仅允许访问"普通"内存
    unsigned int debug:1;           // 48: 调试访问（可写 ROM）
    unsigned int requester_id:16;   // 50: 请求者 ID（MSI）
    unsigned int pid:8;             // 55: PCI PASID
    unsigned int address_type:1;    // 58: PCI 地址类型
    bool unspecified;               // 67: 未指定（区分全零 vs 默认）
} MemTxAttrs;
```

### PTW 到 MemTxAttrs 的传播

```c
// ptw.c:3676-3677 — get_phys_addr_nogpc
result->f.attrs.space = ptw->in_space;
result->f.attrs.secure = arm_space_is_secure(ptw->in_space);

// → TLB 条目 → 每次内存访问携带这些属性
// → address_space_ldq/stq 使用 attrs 选择目标地址空间
```

---

## 9. MMU 禁用时的属性

```c
// ptw.c:3478-3551 — get_phys_addr_disabled
uint8_t memattr = 0x00;       // 默认: Device-nGnRnE
uint8_t shareability = 0;     // 默认: Non-shareable

// 物理索引 (Stage2/Phys_*): 保持 Device-nGnRnE
// 其他索引:
if (HCR_DC) {
    memattr = 0xff;            // Normal, WB, RWA
    // HCR_DCT → 0xf0 Tagged
}
if (memattr == 0 && 指令取指) {
    if (SCTLR_I)
        memattr = 0xee;        // Normal, WT, RA, NT
    else
        memattr = 0x44;        // Normal, NC
    shareability = 2;          // Outer Shareable
}
// 数据访问且无 HCR_DC → Device-nGnRnE
```

---

## 10. S1+S2 缓存属性合并

### 共享性合并

```c
// ptw.c:3427-3456 — combine_cacheattrs
// 规则: 取"最强"的共享性
if (s1.shareability == 2 || s2.shareability == 2)
    ret.shareability = 2;  // Outer Shareable 胜出
else if (s1.shareability == 3 || s2.shareability == 3)
    ret.shareability = 3;  // Inner Shareable 胜出
else
    ret.shareability = 0;  // 都是 Non-shareable

// 特殊: Device 或 NC 强制 Outer Shareable
if ((ret.attrs & 0xf0) == 0 || ret.attrs == 0x44)
    ret.shareability = 2;
```

### FWB (Forced Write-Back) 路径

```c
// ptw.c:3366-3403 — combined_attrs_fwb
switch (s2.attrs) {        // S2 4 位格式
case 7:  return s1.attrs;  // 使用 S1 属性
case 6:  强制 Write-Back;   // S1 Device→0xff, S1 Cacheable→保留 hints
case 5:  // S1 Device→保留, S1 Normal→NC (0x44)
case 0..3: return s2.attrs << 2;  // 强制 Device 子类型
}
```

### 非 FWB 路径

```c
// combined_attrs_nofwb: 取 S1 和 S2 的"更弱"类型
// Device > NC > WT > WB（Device 最弱/最严格）
// 每个 nibble 独立合并
```

---

## 11. 缓存维护指令

### IC 指令（指令缓存）

```c
// helper.c:3484-3507
IC_IALLUIS: ARM_CP_NOP    // 全部内部共享域指令缓存无效
IC_IALLU:   ARM_CP_NOP    // 全部指令缓存无效
IC_IVAU:    ARM_CP_NOP    // 按 VA 指令缓存无效
            // user-mode 例外: ic_ivau_write → tb_invalidate_phys_range
```

### DC 指令（数据缓存）

```c
// helper.c:3509-3575 — "all NOPs since we don't emulate caches"
DC_IVAC:  ARM_CP_NOP      // 按 VA 无效
DC_ISW:   ARM_CP_NOP      // 按 Set/Way 无效
DC_CVAC:  ARM_CP_NOP      // 按 VA 清理
DC_CVAU:  ARM_CP_NOP      // 按 VA 清理到 PoU
DC_CSW:   ARM_CP_NOP      // 按 Set/Way 清理
DC_CIVAC: ARM_CP_NOP      // 按 VA 清理并无效
DC_CISW:  ARM_CP_NOP      // 按 Set/Way 清理并无效
DC_CVAP:  ARM_CP_NOP      // 按 VA 清理到 PoP
DC_CVADP: ARM_CP_NOP      // 按 VA 清理到 PoDP
DC_GVA:   特殊             // MTE tag 零填充
DC_GZVA:  特殊             // MTE tag + data 零填充
DC_ZVA:   特殊             // 数据零填充（实际执行！）
```

**QEMU 原则**：不模拟缓存 → 绝大多数缓存维护指令为 NOP。唯一的实际操作是 DC ZVA/GVA/GZVA（零填充）。

---

## 12. DC ZVA 零填充

```c
// translate-a64.c:2996-3012 — DC ZVA 翻译
case ARM_CP_DC_ZVA:
    // MTE 检查（如启用）
    if (s->mte_active[0]) {
        gen_helper_mte_check_zva(...);      // MTE tag 检查
    } else {
        tcg_rt = clean_data_tbi(s, ...);    // TBI 地址清理
    }
    gen_helper_dc_zva(tcg_env, tcg_rt);     // 3011: 零填充 helper

// DCZID_EL0: 零填充块大小
// helper.c:3227-3239 — aa64_dczid_read
//   返回 log2(block_size/4) | DZP位
//   DZP=1 → DC ZVA 禁用
//   块大小通常 64 字节（缓存行大小）

// 访问控制:
// helper.c:3199-3225 — aa64_zva_access
//   EL0: 需要 SCTLR_DZE (bit14)
//   EL1: 总是允许
//   HCR_TDZ: 陷入 EL2
```

---

## 13. 内存屏障指令

### DMB/DSB

```c
// translate-a64.c:2243-2261 — trans_DSB_DMB
switch (a->types) {
case 1: /* MBReqTypes_Reads */
    bar = TCG_BAR_SC | TCG_MO_LD_LD | TCG_MO_LD_ST;   // 读屏障
    break;
case 2: /* MBReqTypes_Writes */
    bar = TCG_BAR_SC | TCG_MO_ST_ST;                   // 写屏障
    break;
default: /* MBReqTypes_All */
    bar = TCG_BAR_SC | TCG_MO_ALL;                     // 全屏障
    break;
}
tcg_gen_mb(bar);  // 生成 TCG 内存屏障
```

### DSB nXS

```c
// translate-a64.c:2263-2269
// FEAT_XS: nXS 变体 DSB → 全屏障
tcg_gen_mb(TCG_BAR_SC | TCG_MO_ALL);
```

### ISB

```c
// translate-a64.c:2272-2281
// ISB 不是内存屏障，而是指令同步屏障:
//   1. 结束当前 Translation Block
//   2. 确保后续指令看到所有系统寄存器变更
//   3. 立即处理挂起中断
reset_btype(s);
gen_goto_tb(s, 0, 4);  // → 结束 TB, 跳到下一指令
```

### ARM 屏障域

| 指令 | 域 | 含义 |
|------|------|------|
| DMB ISH LD | Inner Shareable, Loads | 读-读/读-写屏障 |
| DMB ISH ST | Inner Shareable, Stores | 写-写屏障 |
| DMB ISH | Inner Shareable, All | 全屏障 |
| DSB ISH | Inner Shareable, All | 全屏障+完成所有操作 |
| DMB OSH | Outer Shareable | 跨集群屏障 |
| DMB SY | Full System | 系统级屏障 |

---

## 14. QEMU TCG 内存模型

### 弱序模型近似

```
QEMU TCG 内存模型特点:
1. ARM 弱序不完全建模 — 正常加载/存储使用 host 内存序
2. 屏障 → host 内存屏障 + TB 断点
3. 单 CPU: 天然顺序一致
4. 多 CPU: tcg_gen_mb() 生成 host barrier
5. Device 内存: 无特殊排序保证（因为不模拟缓存）
```

### TCG 屏障实现

```c
// accel/tcg/backend-ldst.h:14-39
// tcg_req_mo() / cpu_req_mo(): host 特定屏障
// x86 host: 大部分屏障是 NOP（已是强序）
// ARM host: 需要实际 DMB 指令
```

---

## 15. 完整属性流总结

```
=== Stage-1 翻译 ===
页表描述符:
  AttrIndx[2:0] ─→ MAIR_EL1[AttrIndx*8 +: 8] ─→ 8 位 attrs
  SH[1:0]       ─→ shareability (0/2/3)
  AP[2:1]       ─→ prot (RW permissions)

=== S1 属性判定 ===
S1_attrs_are_device(attrs)?
  YES → Device (0x00/04/08/0C)
      → 强制 Outer Shareable
      → 强制对齐检查
  NO  → Normal (0x44/ee/ff/...)
      → 使用 SH 字段

=== Stage-2 翻译（如有） ===
S2 描述符:
  MemAttr[3:0] → is_s2_format=true
  SH[1:0]      → shareability

=== S1+S2 合并 ===
combine_cacheattrs(s1, s2):
  共享性: 取最强 (OSH > ISH > NSH)
  属性:
    FWB=1: combined_attrs_fwb()
      S2=7→S1 / S2=6→WB / S2=5→Device保留或NC / S2=0-3→Device
    FWB=0: combined_attrs_nofwb()
      取更弱 (Device < NC < WT < WB)
  Device/NC → 强制 Outer Shareable

=== 结果 ===
GetPhysAddrResult:
  .f.phys_addr  → 物理地址
  .f.prot       → R/W/X 权限
  .f.attrs      → MemTxAttrs (secure/space)
  .cacheattrs   → ARMCacheAttrs (attrs/shareability)
  → tlb_set_page_full() → TLB 缓存
  → 每次内存访问: attrs 传递给 address_space

=== MMU 禁用特殊路径 ===
get_phys_addr_disabled:
  默认: Device-nGnRnE, Non-shareable
  HCR_DC: Normal-WB-RWA (或 Tagged)
  SCTLR_I + 指令: Normal-WT-RA
  !SCTLR_I + 指令: Normal-NC
  数据（无 HCR_DC）: Device-nGnRnE
```

---

## 交叉参考

- [49-ARM64-页表遍历PTW深度分析](49-ARM64-页表遍历PTW深度分析-Stage1-Stage2翻译-权限检查-Fault处理-安全属性传播.md) — get_phys_addr_lpae MAIR 提取与 ARMCacheAttrs 设置
- [43-ARM64-TCG-softmmu-TLB深度分析](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — TLB 条目存储与 MMIO 分发
- [47-ARM64-系统寄存器与CP访问深度分析](47-ARM64-系统寄存器与CP访问深度分析-ARMCPRegInfo框架-MRS-MSR翻译-cpregs哈希表-EL银行与访问控制.md) — MAIR/TCR/SCTLR 寄存器定义框架

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
