# ARM64 内存管理单元 (MMU/TLB) 深度分析

> QEMU 11.0.50 · 分析基于 commit `8557694`
> 交叉引用：[00-内存子系统](../memory/00-内存子系统深度分析.md) · [09-虚拟化扩展](09-虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU.md) · [06-异常级别状态管理](06-异常级别状态管理深度分析.md)

---

## 目录

1. [概述](#1-概述)
2. [ARMMMUIdx — 翻译体制索引](#2-armmmuidx--翻译体制索引)
3. [翻译体制选择：arm_mmu_idx_el()](#3-翻译体制选择arm_mmu_idx_el)
4. [体制辅助函数](#4-体制辅助函数)
5. [翻译禁用判定](#5-翻译禁用判定)
6. [页表遍历总入口：get_phys_addr()](#6-页表遍历总入口get_phys_addr)
7. [LPAE 页表遍历核心：get_phys_addr_lpae()](#7-lpae-页表遍历核心get_phys_addr_lpae)
8. [TCR 参数解析：aa64_va_parameters()](#8-tcr-参数解析aa64_va_parameters)
9. [TTBR 选择与基地址提取](#9-ttbr-选择与基地址提取)
10. [起始级别计算](#10-起始级别计算)
11. [描述符解析与遍历循环](#11-描述符解析与遍历循环)
12. [Access Flag 与 Dirty Bit 管理](#12-access-flag-与-dirty-bit-管理)
13. [Stage-1 权限检查：get_S1prot()](#13-stage-1-权限检查get_s1prot)
14. [Stage-2 权限检查：get_S2prot()](#14-stage-2-权限检查get_s2prot)
15. [间接权限 (FEAT_S1PIR/S2PIR)](#15-间接权限-feat_s1pirs2pir)
16. [内存属性与 MAIR](#16-内存属性与-mair)
17. [Device 内存检测](#17-device-内存检测)
18. [HCR_EL2 MMU 控制位](#18-hcr_el2-mmu-控制位)
19. [FEAT_LPA/LPA2 大物理地址支持](#19-feat_lpalpa2-大物理地址支持)
20. [Stage-2 页表遍历特殊处理](#20-stage-2-页表遍历特殊处理)
21. [AT 地址翻译指令](#21-at-地址翻译指令)
22. [故障类型与 ARMMMUFaultInfo](#22-故障类型与-armmmufaultinfo)
23. [故障分发：arm_deliver_fault()](#23-故障分发arm_deliver_fault)
24. [QEMU 软件 TLB 架构](#24-qemu-软件-tlb-架构)
25. [TLB 条目插入：tlb_set_page_full()](#25-tlb-条目插入tlb_set_page_full)
26. [TLB 填充路径：arm_cpu_tlb_fill()](#26-tlb-填充路径arm_cpu_tlb_fill)
27. [TLBI 指令实现](#27-tlbi-指令实现)
28. [ASID/VMID 与 TLB 管理](#28-asidvmid-与-tlb-管理)
29. [多核 TLB 一致性](#29-多核-tlb-一致性)
30. [地址空间分区：ARMASIdx](#30-地址空间分区armasidx)
31. [完整翻译流程图](#31-完整翻译流程图)
32. [关键源文件索引](#32-关键源文件索引)

---

## 1. 概述

ARM64 MMU 是 QEMU 中最复杂的子系统之一。QEMU 实现了完整的 AArch64 VMSA（Virtual Memory System Architecture），包括：

- **多级页表遍历**：支持 4KB/16KB/64KB 粒度，0-4 级页表
- **两阶段翻译**：Stage-1（VA→IPA）+ Stage-2（IPA→PA）
- **多翻译体制**：EL1&0、EL2、EL2&0 (VHE)、EL3 各有独立翻译
- **软件 TLB**：QEMU 实现直接映射 + victim cache 的软件 TLB
- **完整的 TLBI 指令集**：包括 FEAT_TLBIRANGE 范围失效

```
VA (虚拟地址)
  │
  ├── QEMU 软件 TLB 查找 (cputlb.c)
  │     ├── 命中 → 直接获取 PA
  │     └── 未命中 → tlb_fill()
  │
  ▼
arm_cpu_tlb_fill() → get_phys_addr()
  │
  ├── 翻译体制选择 (arm_mmu_idx_el)
  ├── 翻译禁用检查 (regime_translation_disabled)
  │
  ├── Stage-1 遍历 (get_phys_addr_lpae)
  │     ├── TCR 参数解析 (aa64_va_parameters)
  │     ├── TTBR 选择 (regime_ttbr)
  │     ├── 多级页表遍历
  │     ├── 权限检查 (get_S1prot)
  │     └── 属性提取 (MAIR 查找)
  │
  ├── Stage-2 遍历 (如果启用)
  │     ├── VTTBR_EL2 基地址
  │     ├── 多级页表遍历
  │     └── 权限检查 (get_S2prot)
  │
  └── tlb_set_page_full() → 插入 TLB
```

---

## 2. ARMMMUIdx — 翻译体制索引

`ARMMMUIdx` 枚举定义了 QEMU 中所有 ARM 翻译体制的索引，位于 `mmuidx.h:137-198`：

### 2.1 有 TLB 的索引（分配实际 TLB 条目）

| 索引 | 值 | 翻译体制 | 说明 |
|------|----|---------|------|
| `E10_0` | 0 | EL1&0 EL0 | Guest 用户态 |
| `E10_0_GCS` | 1 | EL1&0 EL0 GCS | GCS 保护的用户态 |
| `E10_1` | 2 | EL1&0 EL1 | Guest 内核态 |
| `E10_1_PAN` | 3 | EL1&0 EL1 PAN | PAN 启用的内核态 |
| `E10_1_GCS` | 4 | EL1&0 EL1 GCS | GCS 保护的内核态 |
| `E20_0` | 5 | EL2&0 EL0 | VHE host 用户态 |
| `E20_0_GCS` | 6 | EL2&0 EL0 GCS | VHE+GCS 用户态 |
| `E20_2` | 7 | EL2&0 EL2 | VHE host 内核态 |
| `E20_2_PAN` | 8 | EL2&0 EL2 PAN | VHE+PAN 内核态 |
| `E20_2_GCS` | 9 | EL2&0 EL2 GCS | VHE+GCS 内核态 |
| `E2` | 10 | EL2 (非VHE) | 传统 Hypervisor |
| `E2_GCS` | 11 | EL2 GCS | |
| `E3` | 12 | EL3 | Secure Monitor |
| `E3_GCS` | 13 | EL3 GCS | |
| `E30_0` | 14 | EL3&0 EL0 | AArch32 Secure EL0 |
| `E30_3_PAN` | 15 | EL3&0 EL3 PAN | |
| `Stage2_S` | 16 | Secure Stage-2 | |
| `Stage2` | 17 | NS Stage-2 | |
| `Phys_S` | 18 | 物理 Secure | 1:1 映射 |
| `Phys_NS` | 19 | 物理 NS | 1:1 映射 |
| `Phys_Root` | 20 | 物理 Root | 1:1 映射 |
| `Phys_Realm` | 21 | 物理 Realm | 1:1 映射 |

### 2.2 无 TLB 的索引（仅用于 AT 指令或 Stage-1-only 遍历）

| 索引 | 用途 |
|------|------|
| `Stage1_E0` | S12 遍历中的 Stage-1 EL0 |
| `Stage1_E1` | S12 遍历中的 Stage-1 EL1 |
| `Stage1_E1_PAN` | S12 遍历中的 Stage-1 EL1+PAN |
| `Stage1_E0_GCS` | Stage-1 EL0+GCS |
| `Stage1_E1_GCS` | Stage-1 EL1+GCS |

**设计说明**：PAN 和 GCS 需要独立的 MMU 索引，因为它们改变了相同 EL 下的权限行为，需要独立的 TLB 条目。

---

## 3. 翻译体制选择：arm_mmu_idx_el()

```c
// helper.c:9957-10008
ARMMMUIdx arm_mmu_idx_el(CPUARMState *env, int el)
```

根据 EL 和 HCR_EL2 配置选择翻译体制：

```
EL0 ──→ HCR.E2H+TGE ? ──→ E20_0  (VHE host 用户态)
     │   Secure+AArch32 EL3 ? → E30_0
     └── 否 → E10_0  (Guest 用户态)

EL1 ──→ PAN 启用 ? ──→ E10_1_PAN
     └── 否 → E10_1

EL2 ──→ HCR.E2H ? ──→ PAN 启用 ? → E20_2_PAN
     │                 └── 否 → E20_2
     └── 否 → E2  (传统 Hypervisor)

EL3 ──→ AArch32 + PAN ? → E30_3_PAN
     └── 否 → E3
```

**关键逻辑**（helper.c:9966-9996）：
- EL0 的体制取决于 VHE 状态：`E2H+TGE` 让 EL0 使用 host 翻译体制 `E20_0`
- EL1 始终使用 `E10_1`（PAN 变体仅改变权限检查）
- EL2 在 VHE 模式下使用 `E20_2`（与 EL0 共享同一地址空间）
- `arm_mmu_idx(env)` = `arm_mmu_idx_el(env, arm_current_el(env))`

---

## 4. 体制辅助函数

### 4.1 regime_el() — 控制 EL

```c
// mmuidx-internal.h:42-62
```

返回控制该翻译体制的 EL。例如 `E10_0` 和 `E10_1` 都由 EL1 控制。

### 4.2 regime_has_2_ranges() — 是否有双地址范围

返回该体制是否有 TTBR0/TTBR1 两个基地址（即高低地址空间分割）。EL1&0 和 EL2&0 (VHE) 有；EL2 (非VHE) 和 EL3 没有。

### 4.3 regime_ttbr() — TTBR 选择

```c
// ptw.c:232-245
static uint64_t regime_ttbr(CPUARMState *env, ARMMMUIdx mmu_idx, int ttbrn)
{
    if (mmu_idx == ARMMMUIdx_Stage2)    return env->cp15.vttbr_el2;
    if (mmu_idx == ARMMMUIdx_Stage2_S)  return env->cp15.vsttbr_el2;
    if (ttbrn == 0) return env->cp15.ttbr0_el[regime_el(mmu_idx)];
    else            return env->cp15.ttbr1_el[regime_el(mmu_idx)];
}
```

### 4.4 regime_tcr() — TCR 选择

```c
// internals.h:1063-1081
```

根据 `regime_el()` 返回对应的 TCR_ELx。Stage-2 使用 `VTCR_EL2`。

### 4.5 regime_sctlr() — SCTLR 选择

```c
// internals.h:1047-1050
```

返回控制该体制的 SCTLR_ELx。

---

## 5. 翻译禁用判定

```c
// ptw.c:248-331
static bool regime_translation_disabled(CPUARMState *env, ARMMMUIdx mmu_idx,
                                         ARMSecuritySpace space)
```

多层检查决定翻译是否被禁用：

```
┌────────────────────────────────────────────────────────┐
│ Stage-2 (Stage2/Stage2_S):                             │
│   禁用条件: (HCR.DC | HCR.VM) == 0                    │
│   即 HCR.DC 使 HCR.VM 表现为 1              [280]     │
├────────────────────────────────────────────────────────┤
│ E10_0/E10_1 (EL0/EL1 翻译):                           │
│   HCR.TGE=1 → 翻译禁用（EL0/1 表现为 SCTLR.M=0）     │
│   [288-291]                                            │
├────────────────────────────────────────────────────────┤
│ Stage1_E0/Stage1_E1 (S12 遍历中的 Stage-1):           │
│   HCR.DC=1 → Stage-1 翻译禁用（SCTLR_EL1.M 表现为 0）│
│   [299-303]                                            │
├────────────────────────────────────────────────────────┤
│ Phys_* (物理地址空间):                                 │
│   始终禁用（1:1 映射）                      [319-324]  │
├────────────────────────────────────────────────────────┤
│ 其余体制 (E20, E2, E3 等):                            │
│   检查 SCTLR.M 位                          [330]      │
└────────────────────────────────────────────────────────┘
```

---

## 6. 页表遍历总入口：get_phys_addr()

```c
// ptw.c:3931-3943
bool get_phys_addr(CPUARMState *env, vaddr address,
                   MMUAccessType access_type, MemOp memop, ARMMMUIdx mmu_idx,
                   GetPhysAddrResult *result, ARMMMUFaultInfo *fi)
{
    S1Translate ptw = {
        .in_mmu_idx = mmu_idx,
        .in_space = arm_mmu_idx_to_security_space(env, mmu_idx),
        .in_prot_check = 1 << access_type,
    };
    return get_phys_addr_gpc(env, &ptw, address, access_type,
                              memop, result, fi);
}
```

- 返回 `false` = 成功，`true` = 故障
- `S1Translate` 结构体携带遍历状态（安全空间、保护检查位等）
- `get_phys_addr_gpc()` 在翻译完成后执行 Granule Protection Check (FEAT_RME)

---

## 7. LPAE 页表遍历核心：get_phys_addr_lpae()

```c
// ptw.c:1859-2448
static bool get_phys_addr_lpae(CPUARMState *env, S1Translate *ptw,
                                uint64_t address, MMUAccessType access_type,
                                MemOp memop, GetPhysAddrResult *result,
                                ARMMMUFaultInfo *fi)
```

这是 QEMU 中最大的单个函数之一（~590 行），处理 AArch64 和 AArch32-LPAE 的页表遍历。

### 7.1 主要流程

```
1. 参数解析
   ├── AArch64: aa64_va_parameters()        [1888]
   ├── AArch32: aa32_va_parameters()        [1937]
   └── T0SZ/T1SZ 范围外检查               [1912-1913]

2. 粒度与步进计算
   └── stride = arm_granule_bits(gran) - 3  [1962]

3. TTBR 选择
   └── ttbr = regime_ttbr(env, mmu_idx, param.select)  [1972]

4. EPD 检查（翻译表遍历禁用）
   └── param.epd → Translation Fault        [1979-1984]

5. 起始级别计算                             [1987-2009]

6. 基地址提取（含 LPA/LPA2）               [2014-2030]

7. 遍历循环
   ├── 读取描述符 arm_ldq_ptw()             [2082]
   ├── 有效性检查                           [2089-2094]
   ├── Table 描述符 → 累积属性, 下一级      [2115-2125]
   └── Block/Page 描述符 → 最终地址         [2128-2141]

8. AF/Dirty 管理                            [2143-2179]

9. 属性提取与 table 属性合并                [2182-2197]

10. 权限检查
    ├── Stage-2: get_S2prot()               [2201-2227]
    └── Stage-1: get_S1prot()               [2234-2346]

11. 内存属性查找 (MAIR)                     [2334-2340]

12. 结果填充与 Stage-2 嵌套遍历             [2370-2448]
```

---

## 8. TCR 参数解析：aa64_va_parameters()

```c
// helper.c:9672-9757
ARMVAParameters aa64_va_parameters(CPUARMState *env, uint64_t va,
                                    ARMMMUIdx mmu_idx, bool data, bool el1_aarch32)
```

从 TCR_ELx 中提取所有翻译参数：

| 参数 | 来源 | 说明 |
|------|------|------|
| `tsz` | TCR.T0SZ 或 T1SZ | 虚拟地址大小 = 64 - tsz |
| `tbi` | TCR.TBI0/TBI1 | Top Byte Ignore |
| `epd` | TCR.EPD0/EPD1 | 翻译表遍历禁用 |
| `hpd` | TCR.HPD0/HPD1 | 层级权限禁用 |
| `gran` | TCR.TG0/TG1 | 粒度（4K/16K/64K） |
| `ha` | TCR.HA | 硬件 Access Flag 更新 |
| `hd` | TCR.HD | 硬件 Dirty Bit 管理 |
| `ds` | TCR.DS | FEAT_LPA2 描述符大小 |
| `select` | VA[55] | TTBR0 或 TTBR1 选择 |

**粒度映射**（helper.c:9687-9733）：

```
TG0 值:  0 → 4KB,  1 → 64KB,  2 → 16KB
TG1 值:  1 → 16KB, 2 → 4KB,   3 → 64KB
```

**关键**：TTBR0/TTBR1 选择由 VA 的 bit[55] 决定（对于双范围体制）：
- bit[55]=0 → 低地址空间，使用 TTBR0 + T0SZ
- bit[55]=1 → 高地址空间，使用 TTBR1 + T1SZ

---

## 9. TTBR 选择与基地址提取

### 9.1 TTBR 选择

```c
// ptw.c:232-245
static uint64_t regime_ttbr(CPUARMState *env, ARMMMUIdx mmu_idx, int ttbrn)
```

| 条件 | 返回 |
|------|------|
| `Stage2` | `vttbr_el2` |
| `Stage2_S` | `vsttbr_el2` |
| `ttbrn == 0` | `ttbr0_el[regime_el]` |
| `ttbrn == 1` | `ttbr1_el[regime_el]` |

### 9.2 基地址提取

```c
// ptw.c:2014-2030
descaddr = extract64(ttbr, 0, 48);

// FEAT_LPA (PS=6, 52位 PA): 位 [51:48] 存储在 TTBR[5:2]
if (outputsize > 48) {
    descaddr |= extract64(ttbr, 2, 4) << 48;
}
// 否则检查地址大小溢出
else if (descaddr >> outputsize) {
    fi->type = ARMFault_AddressSize;  // AddressSize fault
}
```

---

## 10. 起始级别计算

### 10.1 Stage-1

```c
// ptw.c:1987-2000
level = 4 - (inputsize - 4) / stride;
```

公式推导：
- `inputsize = 64 - T0SZ`（虚拟地址有效位数）
- `stride = granule_bits - 3`（每级索引的位数）
- 需要 `(inputsize - granule_bits) / stride` 级，向上取整
- 简化后：`level = 4 - (inputsize - 4) / stride`

**示例**（4KB 粒度, T0SZ=16, 48位 VA）：
- `inputsize = 48`, `stride = 12 - 3 = 9`
- `level = 4 - (48 - 4) / 9 = 4 - 4 = 0` → 从 Level 0 开始

### 10.2 Stage-2

```c
// ptw.c:2001-2009
int startlevel = check_s2_mmu_setup(cpu, aarch64, tcr, param.ds,
                                     inputsize, stride);
```

Stage-2 使用 `check_s2_mmu_setup()`（ptw.c:1727-1821），因为 VTCR_EL2.SL0 直接指定起始级别，并有额外的合法性验证。

---

## 11. 描述符解析与遍历循环

### 11.1 描述符格式

```
63                             50 49    48 47                  12 11    2 1 0
┌──────────────── 上层属性 ──────┐       ┌──── 输出地址 ────────┐        ┌─┐
│ XN PXN Contig DBM GP ...      │       │  OA[47:12]           │ ...    │V│
└────────────────────────────────┘       └──────────────────────┘        └─┘

bit[0] = Valid (1=有效)
bit[1] = 0→Block (Level 0-2), 1→Table (Level 0-2) / Page (Level 3)
```

### 11.2 有效性检查

```c
// ptw.c:2089-2094
if (!(descriptor & 1) ||
    (!(descriptor & 2) &&
     !lpae_block_desc_valid(cpu, param.ds, param.gran, level))) {
    goto do_translation_fault;  // 无效或非法级别的 Block 描述符
}
```

`lpae_block_desc_valid()`（ptw.c:1823-1840）检查 Block 描述符在当前级别是否合法。

### 11.3 Table 描述符处理

```c
// ptw.c:2115-2125
if ((descriptor & 2) && (level < 3)) {
    // Table 描述符：累积上层属性（bit[63:59]）
    tableattrs |= extract64(descriptor, 59, 5);
    level++;
    indexmask = indexmask_grainsize;
    goto next_level;  // 继续下一级遍历
}
```

Table 属性（NSTable, APTable, XNTable, PXNTable）通过 OR 累积，0 表示"无影响"。

### 11.4 Block/Page 描述符处理

```c
// ptw.c:2128-2141
page_size = (1ULL << ((stride * (4 - level)) + 3));
descaddr &= ~(hwaddr)(page_size - 1);      // 对齐到页面大小
descaddr |= (address & (page_size - 1));    // 合并页内偏移
```

**页面大小计算**（4KB 粒度为例）：
- Level 0: `2^(9*4+3) = 2^39 = 512GB`（通常不会是 Block）
- Level 1: `2^(9*3+3) = 2^30 = 1GB` Block
- Level 2: `2^(9*2+3) = 2^21 = 2MB` Block
- Level 3: `2^(9*1+3) = 2^12 = 4KB` Page

### 11.5 属性提取与 Table 属性合并

```c
// ptw.c:2182-2197
attrs = new_descriptor & (MAKE_64BIT_MASK(2, 10) | MAKE_64BIT_MASK(50, 14));
if (!param.hpd) {  // HPD 禁用时才合并 table 属性
    attrs |= extract64(tableattrs, 0, 2) << 53;     // XN, PXN
    attrs &= ~(extract64(tableattrs, 2, 1) << 6);   // !APT[0] => AP[1]
    attrs |= extract32(tableattrs, 3, 1) << 7;      // APT[1] => AP[2]
}
```

**HPD (Hierarchical Permission Disable)**：当 `TCR.HPD=1` 时，table 描述符中的权限属性被忽略（NSTable 除外），提高遍历效率。

---

## 12. Access Flag 与 Dirty Bit 管理

### 12.1 Access Flag (AF)

```c
// ptw.c:2143-2163
if (!(descriptor & (1 << 10))) {  // AF=0
    if (!param.ha) {
        fi->type = ARMFault_AccessFlag;  // 无 HA 支持 → 故障
        goto do_fault;
    }
    // HA 启用 → 自动设置 AF
    new_descriptor |= 1 << 10;
}
```

- `TCR.HA=0`：AF=0 产生 Access Flag Fault，由软件更新
- `TCR.HA=1`：QEMU 自动设置 AF 位（硬件辅助访问标志）

### 12.2 Dirty Bit (DBM)

```c
// ptw.c:2165-2179
if (param.hd                          // HD 启用
    && extract64(descriptor, 51, 1)   // DBM 位 = 1
    && access_type == MMU_DATA_STORE) {
    if (regime_is_stage2(mmu_idx)) {
        new_descriptor |= 1ull << 7;    // 设置 S2AP[1]（可写）
    } else {
        new_descriptor &= ~(1ull << 7); // 清除 AP[2]（允许写入）
    }
}
```

- `TCR.HD=1` + `DBM=1`：写访问时自动更新 dirty 状态
- Stage-1：清除 AP[2]（AP[2]=0 表示读写）
- Stage-2：设置 S2AP[1]（S2AP[1]=1 表示可写）

---

## 13. Stage-1 权限检查：get_S1prot()

```c
// ptw.c:1434-1541
static int get_S1prot(CPUARMState *env, ARMMMUIdx mmu_idx, bool is_aa64,
                       int user_rw, int prot_rw, int xn, int pxn,
                       ARMSecuritySpace in_pa, ARMSecuritySpace out_pa)
```

### 13.1 AP 位解释

```
AP[2:1]  EL0      EL1/EL2/EL3
00       无访问   读写
01       读写     读写
10       无访问   只读
11       只读     只读
```

### 13.2 PAN 处理

```c
// ptw.c:1456-1462
// PAN: 如果 EL0 有数据权限，禁止特权数据访问
if (user_rw && regime_is_pan(mmu_idx)) {
    prot_rw = 0;
}
// PAN3 (FEAT_PAN3 + SCTLR.EPAN): 如果 EL0 有执行权限也禁止
else if (cpu_isar_feature(aa64_pan3, cpu) && is_aa64 &&
         regime_is_pan(mmu_idx) &&
         (regime_sctlr(env, mmu_idx) & SCTLR_EPAN) && !xn) {
    prot_rw = 0;
}
```

### 13.3 安全状态检查

```c
// ptw.c:1465-1497
if (in_pa != out_pa) {
    // Root → 禁止执行非 Root 内存
    // Realm → 禁止 EL2/EL2&0 执行非 Realm 内存
    // Secure + SCR.SIF → 禁止执行非 Secure 内存
    return prot_rw;  // 返回不含 PAGE_EXEC
}
```

### 13.4 WXN 和 XN/PXN

```c
// ptw.c:1507-1540
// WXN (Write implies eXecute Never)
wxn = regime_sctlr(env, mmu_idx) & SCTLR_WXN;

// AArch64 特权模式: PXN 或 (user 可写) → XN
if (regime_has_2_ranges(mmu_idx) && !is_user) {
    xn = pxn || (user_rw & PAGE_WRITE);
}

// 最终判定：XN 或 (WXN 且可写) → 不可执行
if (xn || (wxn && (prot_rw & PAGE_WRITE))) {
    return prot_rw;       // 无 PAGE_EXEC
}
return prot_rw | PAGE_EXEC;
```

---

## 14. Stage-2 权限检查：get_S2prot()

```c
// ptw.c:1343-1382
static int get_S2prot(CPUARMState *env, int s2ap, int xn, bool s1_is_el0)
```

### 14.1 S2AP 读写权限

```
S2AP[1:0]   权限
00          无访问
01          只读
10          只写
11          读写
```

### 14.2 XN 执行权限（FEAT_XNX 扩展）

```c
// ptw.c:1354-1380
if (cpu_isar_feature(any_tts2uxn, env_archcpu(env))) {
    // FEAT_XNX: 4 值 XN 编码
    switch (xn) {
    case 0: prot |= PAGE_EXEC;                           break;  // 全部可执行
    case 1: if (s1_is_el0) prot |= PAGE_EXEC;            break;  // 仅 EL0 可执行
    case 2:                                               break;  // 不可执行
    case 3: if (!s1_is_el0) prot |= PAGE_EXEC;           break;  // 仅 EL1 可执行
    }
} else {
    // 传统: XN[1]=0 且 (AArch64 或 可读) → 可执行
    if (!extract32(xn, 1, 1)) {
        if (arm_el_is_aa64(env, 2) || prot & PAGE_READ) {
            prot |= PAGE_EXEC;
        }
    }
}
```

---

## 15. 间接权限 (FEAT_S1PIR/S2PIR)

### 15.1 Stage-1 间接权限

```c
// ptw.c:1549-1645
static int get_S1prot_indirect(CPUARMState *env, S1Translate *ptw,
                                ARMMMUIdx mmu_idx, int pi_index, int po_index, ...)
```

使用 PIR_EL1/PIRE0_EL1 中的 4-bit 索引查找 16 种权限组合。

### 15.2 Stage-2 间接权限

```c
// ptw.c:1384-1420
static int get_S2prot_indirect(CPUARMState *env, GetPhysAddrResult *result,
                                int pi_index, int po_index, bool s1_is_el0)
```

使用 S2PIR_EL2 查表，仅在 `SCR_EL3.PIEN=1` 时启用：

```c
// ptw.c:1415
uint64_t pir = (env->cp15.scr_el3 & SCR_PIEN ? env->cp15.s2pir_el2 : 0);
```

权限表（16 种编码，3 种访问类型：特权/用户/页表遍历）：

```
编码  特权          用户                页表遍历
0x0   无            无                  无
0x4   W             W                   无
0x8   R             R                   R
0x9   R             R+X                 R
0xA   R+X           R                   R
0xB   R+X           R+X                 R
0xC   R+W           R+W                 R+W
0xF   R+W+X         R+W+X              R+W
```

---

## 16. 内存属性与 MAIR

### 16.1 MAIR 查找

```c
// ptw.c:2334-2340
// AttrIndx 从描述符 bits[4:2] 提取
// MAIR_ELx 包含 8 个 8-bit 属性字段
// AttrIndx 选择其中一个
```

MAIR 属性字段编码：
```
高 4 位    低 4 位    含义
0000       00dd       Device memory (d=device 类型)
0000       0001       Device-nGnRnE (FEAT_XS)
oooo       iiii       Normal memory (o=outer, i=inner)
                      0b00=NC, 0b01=WB-RA, 0b10=WT, 0b11=WB
```

### 16.2 QEMU 的简化

QEMU 注释（ptw.c:1964-1970）明确指出：**QEMU 忽略 shareability 和 cacheability 属性**。这意味着 ORGN/IRGN/SH 字段在内部不影响模拟行为，但属性信息仍然被收集到 `result->cacheattrs` 中，供 Device 内存检测和 KVM 交互使用。

---

## 17. Device 内存检测

### 17.1 Stage-1 Device 检测

```c
// ptw.c:574-582
static bool S1_attrs_are_device(uint8_t attrs)
{
    return (attrs & 0xf0) == 0;  // MAIR 高 4 位全 0 = Device
}
```

### 17.2 Stage-2 Device 检测

```c
// ptw.c:584-600
static bool S2_attrs_are_device(uint64_t hcr, uint8_t attrs)
{
    if (hcr & HCR_FWB) {
        return (attrs & 0x4) == 0;  // FWB: bit[2]=0 为 Device
    } else {
        return (attrs & 0xc) == 0;  // 非 FWB: bits[3:2]=00 为 Device
    }
}
```

### 17.3 HCR.PTW 保护

```c
// ptw.c:705-716
if ((hcr & HCR_PTW) && S2_attrs_are_device(hcr, pte_attrs)) {
    // PTW=1 且 S1 页表遍历触碰了 S2 Device 内存
    // → Permission fault (保护页表遍历不访问 Device 内存)
    fi->type = ARMFault_Permission;
    fi->s1ptw = true;
    fi->stage2 = true;
}
```

---

## 18. HCR_EL2 MMU 控制位

| 控制位 | 作用 | QEMU 实现位置 |
|--------|------|--------------|
| `HCR.VM` | 启用 Stage-2 翻译 | ptw.c:280 |
| `HCR.DC` | 默认缓存性 — 使 VM 表现为 1，SCTLR_EL1.M 表现为 0 | ptw.c:275-303 |
| `HCR.PTW` | 保护页表遍历 — S1 遍历碰到 S2 Device 内存时产生故障 | ptw.c:705-716 |
| `HCR.FWB` | 强制写回 — 改变 S2 描述符属性解释 | ptw.c:584-598 |
| `HCR.TGE` | 陷入通用异常 — 禁用 EL0/1 翻译 | ptw.c:288-291 |

**HCR.DC 的完整效果**：
```
HCR.DC=1
  ├── Stage-2: HCR.VM 表现为 1 → Stage-2 启用      [275-280]
  ├── Stage-1: SCTLR_EL1.M 表现为 0 → Stage-1 禁用 [299-303]
  └── 效果: 所有 EL1/EL0 内存访问直接走 Stage-2
            Stage-2 返回 Normal WB Cacheable 属性
```

---

## 19. FEAT_LPA/LPA2 大物理地址支持

### 19.1 FEAT_LPA（52位 PA，传统描述符格式）

```c
// ptw.c:2017-2025
// TTBR 中: bits[5:2] 存储 OA[51:48]
if (outputsize > 48) {
    descaddr |= extract64(ttbr, 2, 4) << 48;
}

// 描述符中: bits[15:12] 存储 OA[51:48]
if (outputsize > 48) {
    descaddr |= extract64(descriptor, 12, 4) << 48;
}
```

### 19.2 FEAT_LPA2（52位 PA，新描述符格式）

```c
// ptw.c:2039-2045, 2104-2108
// DS=1 时，描述符地址掩码扩展到 50 位
if (param.ds) {
    descaddrmask = MAKE_64BIT_MASK(0, 50);
}

// 描述符中: bits[9:8] 存储 OA[51:50]
if (param.ds) {
    descaddr |= extract64(descriptor, 8, 2) << 50;
}
```

### 19.3 输出地址大小限制

```c
// ptw.c:1930-1935
// LPA2: 如果 TCR.DS=0，有效输出地址大小限制为 48 位
if (!param.ds && param.gran != Gran16K) {
    outputsize = MIN(outputsize, 48);
}
```

---

## 20. Stage-2 页表遍历特殊处理

### 20.1 Stage-2 起始级别

```c
// ptw.c:1727-1821 — check_s2_mmu_setup()
```

Stage-2 的起始级别由 `VTCR_EL2.SL0` 直接指定，而非从 inputsize 计算。函数还验证 SL0 与 T0SZ/granule 的组合是否合法。

### 20.2 Stage-2 页表遍历地址空间

```c
// ptw.c:193-225 — ptw_idx_for_stage_2()
```

Stage-2 页表遍历使用的物理地址空间取决于安全状态：
- NS → `Phys_NS`
- Realm → `Phys_Realm`
- Secure → 根据 VTCR.NSW/VSTCR.SW 决定 `Phys_S` 或 `Phys_NS`

### 20.3 S1+S2 级联

当 EL1&0 体制启用了 Stage-2 时，QEMU 进行两阶段翻译：
1. 使用 `stage_1_mmu_idx()` 将 `E10_0/E10_1` 映射到 `Stage1_E0/Stage1_E1`
2. Stage-1 遍历产生 IPA
3. Stage-2 遍历将 IPA 转换为 PA
4. Stage-1 页表遍历本身的内存访问也经过 Stage-2（如果 `s1ptw=true` 的故障）

---

## 21. AT 地址翻译指令

### 21.1 AT 指令实现

```c
// cpregs-at.c:310-356 — ats_write64()
```

处理 AArch64 AT 指令（AT S1E1R/W, AT S12E1R/W 等），调用 `do_ats_write()` 执行翻译。

### 21.2 PAR_EL1 结果格式

```c
// ptw.c:26-190 — do_ats_write()
```

64 位 PAR_EL1 格式：

**成功时**：
```
63       56 55  48 47            12 11  9  8  7   6    0
┌─ ATTR ──┐      ┌── PA[47:12] ──┐ SH  NS  F=0
```

**失败时**：
```
63                    10  9   6    1   0
                      FST  S  PTW  S1  F=1
```

- `F` (bit 0): 0=成功, 1=失败
- `PA[47:12]`: 翻译后物理地址
- `ATTR`: 内存属性
- `SH`: 共享性
- `FST`: 故障状态码
- `PTW`: Stage-1 页表遍历中的 Stage-2 故障
- `S`: Stage-2 故障

---

## 22. 故障类型与 ARMMMUFaultInfo

### 22.1 ARMFaultType 枚举

```c
// internals.h:705-730
typedef enum ARMFaultType {
    ARMFault_None,
    ARMFault_AccessFlag,        // AF=0 且无 HA
    ARMFault_Alignment,         // 对齐错误
    ARMFault_Background,        // MPU 背景故障
    ARMFault_Domain,            // AArch32 域故障
    ARMFault_Permission,        // 权限故障
    ARMFault_Translation,       // 翻译故障（无有效映射）
    ARMFault_AddressSize,       // 地址超出范围
    ARMFault_SyncExternal,      // 同步外部中止
    ARMFault_SyncExternalOnWalk,// 遍历中的外部中止
    ARMFault_Debug,             // 调试异常
    ARMFault_TLBConflict,       // TLB 冲突
    ARMFault_UnsuppAtomicUpdate,// 不支持的原子更新
    ARMFault_GPCFOnWalk,        // GPC 故障（遍历中）
    ARMFault_GPCFOnOutput,      // GPC 故障（输出）
    ...
} ARMFaultType;
```

### 22.2 ARMMMUFaultInfo 结构

```c
// internals.h:757-771
struct ARMMMUFaultInfo {
    ARMFaultType type;      // 故障类型
    ARMGPCF gpcf;           // GPC 子类型
    hwaddr s2addr;          // Stage-2 故障地址
    hwaddr paddr;           // GPC 物理地址
    ARMSecuritySpace paddr_space;  // GPC 物理地址空间
    int level;              // 故障级别 (0-3)
    int domain;             // AArch32 域
    bool stage2;            // Stage-2 故障
    bool s1ptw;             // S1 遍历中的 S2 故障
    bool s1ns;              // 非 Secure IPA
    bool ea;                // 外部中止类型位
    bool dirtybit;          // FEAT_S1PIE/S2PIE dirty 位
};
```

---

## 23. 故障分发：arm_deliver_fault()

```c
// tlb_helper.c:174-260
static void arm_deliver_fault(ARMCPU *cpu, vaddr addr,
                               MMUAccessType access_type,
                               ARMMMUFaultInfo *fi)
```

关键逻辑：

1. **目标 EL 确定**：
   - Stage-2 故障 → 固定到 EL2（tlb_helper.c:239-245）
   - 其他 → `exception_target_el()`

2. **Stage-2 故障处理**：
   ```c
   // tlb_helper.c:239-245
   if (fi->stage2) {
       target_el = 2;
       env->cp15.hpfar_el2 = extract64(fi->s2addr, 12, 36) << 4;
       if (fi->s1ns) {
           env->cp15.hpfar_el2 |= HPFAR_NS;
       }
   }
   ```

3. **综合征构建**：通过 `merge_syn_data_abort()` 或 `compute_fsr_fsc()` 生成 ESR/DFSR 值

---

## 24. QEMU 软件 TLB 架构

### 24.1 TLB 条目结构

```c
// tlb-common.h:24-54
typedef struct CPUTLBEntry {
    // 每种访问类型的地址标签
    uintptr_t addr_read;    // 读地址 (含标志)
    uintptr_t addr_write;   // 写地址 (含标志)
    uintptr_t addr_code;    // 取指地址 (含标志)
    // 主机内存偏移 (VA + addend = host pointer)
    uintptr_t addend;
} CPUTLBEntry;
```

### 24.2 TLB 标志

```c
// tlb-flags.h:40-83
TLB_INVALID_MASK    // 条目无效
TLB_NOTDIRTY        // 页面未标记为脏
TLB_MMIO            // MMIO 区域
TLB_FORCE_SLOW      // 强制走慢路径
TLB_DISCARD_WRITE   // 丢弃写入（ROM 等）
```

### 24.3 TLB 组织

QEMU 使用 **直接映射 + victim cache** 的两级 TLB 结构：

```
┌─────────────────────────────────────────┐
│  Direct-mapped TLB (主 TLB)              │
│  索引 = (VA >> TARGET_PAGE_BITS) & mask  │
│  每个 MMU index 独立                     │
├─────────────────────────────────────────┤
│  Victim Cache (受害者缓存)               │
│  被主 TLB 逐出的条目存放于此              │
│  全关联查找                              │
└─────────────────────────────────────────┘
```

查找流程（cputlb.c:95-140）：
1. 计算 `index = tlb_index(addr, size, mmu_idx)`
2. 比较 `tlb_entry[index].addr_read` 与 `addr & TARGET_PAGE_MASK`
3. 命中 → 使用 `addend` 直接访问主机内存
4. 未命中 → 检查 victim cache（cputlb.c:1304-1333），命中则交换
5. 两级都未命中 → 调用 `tlb_fill()`

---

## 25. TLB 条目插入：tlb_set_page_full()

```c
// cputlb.c:1024-1183
void tlb_set_page_full(CPUState *cpu, int mmu_idx, vaddr addr,
                        CPUTLBEntryFull *full)
```

主要步骤：
1. 更新大页跟踪（cputlb.c:1040-1045）
2. 计算读/写/执行标志（cputlb.c:1076-1103）
   - `PAGE_READ` → 设置 `addr_read`
   - `PAGE_WRITE` + dirty → 设置 `addr_write`
   - `PAGE_EXEC` → 设置 `addr_code`
   - MMIO → 设置 `TLB_MMIO` 标志
3. 逐出旧条目到 victim cache（cputlb.c:1118-1145）
4. 安装新条目（cputlb.c:1145-1183）

---

## 26. TLB 填充路径：arm_cpu_tlb_fill()

```c
// tlb_helper.c — arm_cpu_tlb_fill()
```

TLB 未命中时的完整调用链：

```
TCG 生成代码中的内存访问
  ↓ TLB 未命中
cputlb.c: tlb_fill()
  ↓
arm_cpu_tlb_fill()
  ↓
get_phys_addr()
  ├── 成功 → tlb_set_page_full() → 插入 TLB 条目
  └── 失败 → arm_deliver_fault() → 产生异常
```

---

## 27. TLBI 指令实现

### 27.1 TLBI 指令架构

TLBI 指令在 `tlb-insns.c` 中实现，主要分为：

**按范围**：
- `TLBI ALLE1` — 刷新所有 EL1 翻译（tlb-insns.c:224-486）
- `TLBI VAE1` — 按 VA 刷新 EL1（使用 `tlb_flush_page_bits_by_mmuidx`）
- `TLBI VMALLE1` — 刷新当前 VMID 的所有 EL1 翻译
- `TLBI IPAS2E1` — 按 IPA 刷新 Stage-2

**IS 变体**（Inner Shareable，广播到所有 CPU）：
```c
// tlb-insns.c:49-80
// IS 变体始终广播到所有 CPU
// 非 IS 变体根据 HCR_EL2.FB 决定是否广播
```

**FEAT_TLBIRANGE**（范围失效）：
```c
// tlb-insns.c:403-464
// VAE1/VAE2 路径使用 tlb_flush_page_bits_by_mmuidx*
// 支持指定范围的选择性 TLB 失效
```

### 27.2 TLB 掩码计算

不同 TLBI 指令刷新的 MMU 索引不同：

```c
// tlb-insns.c:224-486
// VMALLE1: E10_0 | E10_1 | E10_1_PAN | Stage1 变体
// ALLE1:   VMALLE1 + Stage2
// ALLE2:   E2 | E20_0 | E20_2 | E20_2_PAN
// ALLE3:   E3
```

### 27.3 底层 TLB 刷新 API

| 函数 | 位置 | 说明 |
|------|------|------|
| `tlb_flush()` | cputlb.c | 刷新单个 CPU 的全部 TLB |
| `tlb_flush_all_cpus_synced()` | cputlb.c:433-445 | 同步刷新所有 CPU |
| `tlb_flush_page_by_mmuidx()` | cputlb.c:600-656 | 按页+MMU索引刷新 |
| `tlb_flush_range_by_mmuidx()` | cputlb.c:766-853 | 范围刷新 |

---

## 28. ASID/VMID 与 TLB 管理

### 28.1 ASID

QEMU 的软件 TLB **不直接支持 ASID 标记**。注释明确说明（ptw.c:1968-1970）：

> QEMU's TLB doesn't currently implement any ASID-like capability so we can ignore it (instead we will always flush the TLB any time the ASID is changed).

这意味着：
- ASID 变更（如 `CONTEXTIDR` 写入）会触发完整 TLB 刷新（helper.c:392-400）
- 不需要 ASID 匹配逻辑，简化了 TLB 实现
- 代价是 ASID 切换时性能较低（全部刷新而非选择性保留）

### 28.2 VMID

```c
// tlb-insns.c:488-523
// IPAS2E1 选择 Stage2 vs Stage2_S 基于当前安全状态
// VTTBR_EL2 写入时，VMID 变更触发 alle1 TLB 刷新
```

VMID 变更（VTTBR_EL2 写入）同样触发相关 TLB 条目的完整刷新。

---

## 29. 多核 TLB 一致性

### 29.1 广播机制

QEMU 通过 `_synced` 后缀函数实现跨 CPU TLB 操作：

```c
// cputlb.c:617-655 — tlb_flush_page_by_mmuidx_all_cpus_synced()
// cputlb.c:803-844 — tlb_flush_range_by_mmuidx_all_cpus_synced()
```

### 29.2 IS 变体

```c
// tlb-insns.c:49-80
// Inner-Shareable TLBI 操作始终广播到所有 CPU
```

### 29.3 HCR.FB 升级

```c
// tlb-insns.c:97-143
// HCR_EL2.FB (Force Broadcast) 将本地 TLBI 操作升级为广播
// 即非 IS 变体在 FB=1 时也会刷新所有 CPU
```

---

## 30. 地址空间分区：ARMASIdx

```c
// cpu.h:2368-2374
typedef enum ARMASIdx {
    ARMASIdx_NS = 0,     // Non-Secure
    ARMASIdx_S = 1,      // Secure
    ARMASIdx_TagNS = 2,  // Tagged Non-Secure (MTE)
    ARMASIdx_TagS = 3,   // Tagged Secure (MTE)
} ARMASIdx;
```

```c
// cpu.h:2610-2613
// arm_addressspace() 根据安全状态和 MTE 选择地址空间
```

QEMU 为每个 CPU 创建 4 个地址空间（cpu.c:2294-2308），每个地址空间有独立的 MemoryRegion 树和 FlatView。

---

## 31. 完整翻译流程图

```
                        VA (虚拟地址)
                            │
                 ┌──────────▼──────────────┐
                 │ arm_mmu_idx_el(env, el)  │
                 │ 选择翻译体制              │
                 └──────────┬──────────────┘
                            │
                 ┌──────────▼──────────────┐
                 │ regime_translation_      │
                 │ disabled() ?             │
                 ├── 是 → PA = VA          │
                 └── 否 ───┬───────────────┘
                            │
              ┌─────────────▼──────────────────┐
              │ get_phys_addr_lpae()            │
              │                                 │
              │  ┌── aa64_va_parameters() ──┐   │
              │  │ TCR.T0SZ/T1SZ           │   │
              │  │ TCR.TG0/TG1 → 粒度      │   │
              │  │ VA[55] → TTBR0/TTBR1    │   │
              │  └─────────────────────────┘   │
              │                                 │
              │  ┌── 遍历循环 ────────────────┐ │
              │  │ Level 0 → 1 → 2 → 3       │ │
              │  │                             │ │
              │  │  读取描述符 (arm_ldq_ptw)   │ │
              │  │    │                        │ │
              │  │    ├── Invalid → TransFault │ │
              │  │    ├── Table → 累积属性     │ │
              │  │    │          下一级         │ │
              │  │    └── Block/Page → 完成    │ │
              │  └─────────────────────────────┘ │
              │                                 │
              │  ┌── 检查 ─────────────────────┐│
              │  │ AF 检查 (HA 自动设置)       ││
              │  │ Dirty Bit (HD 自动更新)     ││
              │  │ get_S1prot() 权限检查       ││
              │  │ MAIR 属性查找               ││
              │  └─────────────────────────────┘│
              └─────────┬───────────────────────┘
                        │ IPA
              ┌─────────▼───────────────────────┐
              │ Stage-2 启用 ?                   │
              │ (HCR.VM=1 或 HCR.DC=1)          │
              ├── 否 → PA = IPA                 │
              └── 是 ──┐                        │
                       │                        │
              ┌────────▼────────────────────────┐
              │ Stage-2 get_phys_addr_lpae()    │
              │ VTTBR_EL2 → 页表遍历            │
              │ get_S2prot() 权限检查            │
              │ HCR.PTW Device 检查              │
              │ HCR.FWB 属性覆盖                │
              └────────┬────────────────────────┘
                       │ PA
              ┌────────▼────────────────────────┐
              │ GPC 检查 (FEAT_RME)             │
              │ tlb_set_page_full()              │
              │ → 插入 QEMU 软件 TLB            │
              └─────────────────────────────────┘
```

---

## 32. 关键源文件索引

| 文件 | 行范围 | 内容 |
|------|--------|------|
| `ptw.c` | 160-179 | `stage_1_mmu_idx()` — Stage-1 MMU 索引映射 |
| `ptw.c` | 193-225 | `ptw_idx_for_stage_2()` — Stage-2 遍历地址空间选择 |
| `ptw.c` | 232-245 | `regime_ttbr()` — TTBR 选择 |
| `ptw.c` | 248-331 | `regime_translation_disabled()` — 翻译禁用判定 |
| `ptw.c` | 574-600 | `S1_attrs_are_device()` / `S2_attrs_are_device()` |
| `ptw.c` | 695-717 | HCR.PTW Device 保护 |
| `ptw.c` | 1343-1420 | `get_S2prot()` / `get_S2prot_indirect()` — Stage-2 权限 |
| `ptw.c` | 1434-1645 | `get_S1prot()` / `get_S1prot_indirect()` — Stage-1 权限 |
| `ptw.c` | 1727-1821 | `check_s2_mmu_setup()` — Stage-2 起始级别验证 |
| `ptw.c` | 1859-2448 | `get_phys_addr_lpae()` — LPAE 页表遍历核心 |
| `ptw.c` | 3931-3943 | `get_phys_addr()` — 翻译总入口 |
| `ptw.c` | 26-190 | `do_ats_write()` — AT 指令 PAR_EL1 结果生成 |
| `helper.c` | 9672-9757 | `aa64_va_parameters()` — TCR 参数解析 |
| `helper.c` | 9957-10013 | `arm_mmu_idx_el()` / `arm_mmu_idx()` |
| `mmuidx.h` | 137-198 | `ARMMMUIdx` 枚举定义 |
| `mmuidx-internal.h` | 42-62 | `regime_el()` / `regime_has_2_ranges()` |
| `internals.h` | 705-771 | `ARMFaultType` / `ARMMMUFaultInfo` |
| `internals.h` | 779-958 | 故障码映射函数 |
| `internals.h` | 1047-1100 | `regime_sctlr()` / `regime_tcr()` / `regime_using_lpae_format()` |
| `tlb_helper.c` | 25-260 | 故障综合征构建 + `arm_deliver_fault()` |
| `cpregs-at.c` | 310-421 | AT 指令实现 |
| `cputlb.c` | 95-140 | TLB 索引与查找 |
| `cputlb.c` | 1024-1207 | `tlb_set_page_full()` — TLB 插入 |
| `cputlb.c` | 1245-1333 | TLB 填充 + victim cache |
| `cputlb.c` | 433-853 | TLB 刷新 API（全量/按页/范围） |
| `tlb-insns.c` | 49-818 | TLBI 指令实现 |
| `tlb-common.h` | 24-54 | `CPUTLBEntry` 结构 |
| `tlb-flags.h` | 40-83 | TLB 标志定义 |
| `cpu.h` | 2368-2374 | `ARMASIdx` 地址空间索引 |

---

> **文档版本**：v1.0
> **生成时间**：2025 年
> **分析工具**：zoekt + ctags + cscope 索引，QEMU 源码直接阅读验证
