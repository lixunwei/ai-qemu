# ARM64 TCG 内存访问子系统深度分析：softmmu TLB 结构、快慢路径、页表遍历、TLBI 指令与 MMIO 分发

> 基于 QEMU 11.0.50 源码分析，涵盖 TCG softmmu 完整内存访问子系统：
> CPUTLBEntry/CPUTLBEntryFull/CPUTLBDescFast/CPUTLBDesc/CPUTLB 五层数据结构、
> TLB 索引计算与 addend 技巧、TLB 标志位（TLB_MMIO/NOTDIRTY/FORCE_SLOW/WATCHPOINT）、
> 快速路径 probe_access_internal 与 TLB 命中判定、慢路径 TLB 填充与 MMIO 分发、
> ARM 页表遍历 get_phys_addr/get_phys_addr_lpae（LPAE 四级页表）、
> 两阶段翻译 S1→S2、MAIR/缓存属性组合、arm_cpu_tlb_fill_align 填充回调、
> ARMMMUIdx 22 种 MMU 索引、TLBI 指令实现（VMALLE1/VAE1/IPAS2E1/ALLE1-3/range）、
> AT 指令与 PAR_EL1、原子操作 atomic_mmu_lookup、脏页追踪 notdirty_write、
> TLB 动态调整与 victim cache、arm_deliver_fault 异常生成。

---

## 目录

1. [架构概述](#1-架构概述)
2. [TLB 数据结构五层体系](#2-tlb-数据结构五层体系)
3. [TLB 索引计算与 addend 技巧](#3-tlb-索引计算与-addend-技巧)
4. [TLB 标志位系统](#4-tlb-标志位系统)
5. [ARMMMUIdx — 22 种 MMU 索引](#5-armmmuidx--22-种-mmu-索引)
6. [快速路径 — TLB 命中](#6-快速路径--tlb-命中)
7. [慢路径 — TLB 未命中与填充](#7-慢路径--tlb-未命中与填充)
8. [ARM 页表遍历 — get_phys_addr](#8-arm-页表遍历--get_phys_addr)
9. [LPAE 长描述符格式页表遍历](#9-lpae-长描述符格式页表遍历)
10. [两阶段翻译 S1→S2](#10-两阶段翻译-s1s2)
11. [缓存属性与 MAIR](#11-缓存属性与-mair)
12. [tlb_set_page_full — TLB 安装](#12-tlb_set_page_full--tlb-安装)
13. [MMIO 分发路径](#13-mmio-分发路径)
14. [TLB 刷新与 TLBI 指令](#14-tlb-刷新与-tlbi-指令)
15. [AT 指令实现](#15-at-指令实现)
16. [原子操作与 TLB](#16-原子操作与-tlb)
17. [脏页追踪 — notdirty_write](#17-脏页追踪--notdirty_write)
18. [TLB 动态调整与 Victim Cache](#18-tlb-动态调整与-victim-cache)
19. [异常生成 — arm_deliver_fault](#19-异常生成--arm_deliver_fault)
20. [完整内存访问流程总结](#20-完整内存访问流程总结)

---

## 1. 架构概述

TCG softmmu 通过软件 TLB 实现 guest 虚拟地址到 host 物理地址的翻译。每次 guest 内存访问的核心路径：

```
Guest Load/Store
  │
  ├── TCG 后端内联代码：TLB 查找（快速路径）
  │   ├── 命中 → host_addr = guest_addr + addend → 直接访问
  │   └── 未命中 → 调用 helper
  │
  └── Helper 慢路径：
      ├── TLB 填充：arm_cpu_tlb_fill_align → get_phys_addr（PTW）
      │   → tlb_set_page_full（安装 TLB 条目）
      └── MMIO：io_prepare → memory_region_dispatch_read/write
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `include/exec/tlb-common.h` | 22-55 | CPUTLBEntry、CPUTLBDescFast |
| `include/hw/core/cpu.h` | 209-333 | CPUTLBEntryFull、CPUTLBDesc、CPUTLB |
| `include/exec/tlb-flags.h` | 34-83 | TLB 标志位定义 |
| `accel/tcg/cputlb.c` | 105-140 | TLB 索引/查找 |
| `accel/tcg/cputlb.c` | 418-615 | TLB flush 操作 |
| `accel/tcg/cputlb.c` | 1024-1207 | tlb_set_page_full 安装 |
| `accel/tcg/cputlb.c` | 1360-1412 | probe_access_internal |
| `accel/tcg/cputlb.c` | 1799-1903 | atomic_mmu_lookup |
| `target/arm/ptw.c` | 1843-2449 | LPAE 页表遍历 |
| `target/arm/ptw.c` | 3413-3463 | 缓存属性组合 |
| `target/arm/ptw.c` | 3554-3658 | 两阶段翻译 |
| `target/arm/ptw.c` | 3931-3943 | get_phys_addr 入口 |
| `target/arm/tcg/tlb_helper.c` | 173-273 | arm_deliver_fault |
| `target/arm/tcg/tlb_helper.c` | 331-379 | arm_cpu_tlb_fill_align |
| `target/arm/tcg/tlb-insns.c` | 87-523 | TLBI 指令实现 |
| `target/arm/tcg/cpregs-at.c` | 26-420 | AT 指令实现 |
| `target/arm/mmuidx.h` | 137-198 | ARMMMUIdx 枚举 |

---

## 2. TLB 数据结构五层体系

### 第 1 层：CPUTLBEntry — 快速路径条目

```c
// include/exec/tlb-common.h:25-41
typedef union CPUTLBEntry {
    struct {
        uintptr_t addr_read;   // 读比较地址（含标志位低位）
        uintptr_t addr_write;  // 写比较地址
        uintptr_t addr_code;   // 取指比较地址
        uintptr_t addend;      // host 地址 = guest_addr + addend
    };
    uintptr_t addr_idx[(1 << CPU_TLB_ENTRY_BITS) / sizeof(uintptr_t)];
} CPUTLBEntry;
// 大小 = 2^CPU_TLB_ENTRY_BITS（64 位系统 = 32 字节）
```

### 第 2 层：CPUTLBDescFast — 快速路径描述符

```c
// include/exec/tlb-common.h:49-54
typedef struct CPUTLBDescFast {
    uintptr_t mask;          // (n_entries - 1) << CPU_TLB_ENTRY_BITS
    CPUTLBEntry *table;      // TLB 条目数组指针
} CPUTLBDescFast QEMU_ALIGNED(2 * sizeof(void *));
```

**对齐到 16 字节**：AArch64 后端用一条 LDP 同时加载 mask+table。

### 第 3 层：CPUTLBEntryFull — 完整条目信息

```c
// include/hw/core/cpu.h:214-271
struct CPUTLBEntryFull {
    hwaddr xlat_offset;           // 222: 物理偏移（RAM 的 ram_addr 或 MMIO 偏移）
    MemoryRegionSection *section; // 225: 物理段（指向 MemoryRegion）
    hwaddr phys_addr;             // 231: 物理地址
    MemTxAttrs attrs;             // 234: 事务属性（安全/非安全等）
    uint8_t prot;                 // 237: 权限（R/W/X）
    uint8_t lg_page_size;         // 240: log2(页大小)
    uint8_t tlb_fill_flags;       // 243: 填充标志
    uint8_t slow_flags[3];        // 249: 慢路径标志（读/写/执行各一组）
    union {
        struct {                  // ARM 专用
            uint8_t pte_attrs;    // 266: 页表属性（MAIR 格式）
            uint8_t shareability; // 267: 共享性
            bool guarded;         // 268: GP 位（BTI 保护）
        } arm;
    } extra;
};
```

### 第 4 层：CPUTLBDesc — 每 MMU 模式描述符

```c
// include/hw/core/cpu.h:277-297
typedef struct CPUTLBDesc {
    vaddr large_page_addr;         // 284: 大页区域地址
    vaddr large_page_mask;         // 285: 大页区域掩码
    int64_t window_begin_ns;       // 287: 调整窗口起始时间
    size_t window_max_entries;     // 289: 窗口最大条目数
    size_t n_used_entries;         // 290: 当前使用条目数
    size_t vindex;                 // 292: victim cache 下一索引
    CPUTLBEntry vtable[8];         // 294: victim cache 条目
    CPUTLBEntryFull vfulltlb[8];   // 295: victim cache 完整信息
    CPUTLBEntryFull *fulltlb;      // 296: 主 TLB 完整信息数组
} CPUTLBDesc;
```

### 第 5 层：CPUTLB — 顶层 TLB 结构

```c
// include/hw/core/cpu.h:327-333
typedef struct CPUTLB {
    CPUTLBCommon c;                    // 329: 锁 + 统计 + dirty 位图
    CPUTLBDesc d[NB_MMU_MODES];        // 330: 22 个 MMU 模式的描述符
    CPUTLBDescFast f[NB_MMU_MODES];    // 331: 22 个快速路径描述符
} CPUTLB;
// NB_MMU_MODES = 22（ARM A-profile 最大值）
```

**内存布局**：CPUTLB 嵌入在 `CPUNegativeOffsetState` 中，使得 `f[mmu_idx]` 相对于 `env` 有固定负偏移，后端可用常量偏移直接访问。

---

## 3. TLB 索引计算与 addend 技巧

### 3.1 TLB 索引计算

```c
// accel/tcg/cputlb.c:127-133
static inline uintptr_t tlb_index(CPUState *cpu, uintptr_t mmu_idx, vaddr addr)
{
    uintptr_t size_mask = cpu_tlb_fast(cpu, mmu_idx)->mask >> CPU_TLB_ENTRY_BITS;
    return (addr >> TARGET_PAGE_BITS) & size_mask;
}
```

**公式**：`index = (guest_addr >> PAGE_BITS) & ((n_entries - 1))`

### 3.2 TLB 命中判定

```c
// accel/tcg/cputlb.c:105-118  tlb_read_idx()
static inline uint64_t tlb_read_idx(const CPUTLBEntry *entry,
                                    MMUAccessType access_type)
{
    const uintptr_t *ptr = &entry->addr_idx[access_type];
    return qatomic_read(ptr);  // 原子读取比较地址
}

// 比较逻辑（概念性）
tlb_addr = tlb_read_idx(entry, MMU_DATA_LOAD);
if ((addr & TARGET_PAGE_MASK) == (tlb_addr & (TARGET_PAGE_MASK | TLB_FLAGS_MASK))) {
    // TLB 命中！
    host_addr = addr + entry->addend;
}
```

### 3.3 addend 技巧

**核心思想**：`addend = host_ram_ptr - guest_page_addr`

```
host_addr = guest_addr + addend
          = guest_addr + (host_ram_ptr - guest_page_addr)
          = host_ram_ptr + (guest_addr - guest_page_addr)
          = host_ram_ptr + page_offset
```

这使得 TLB 命中只需要**一次加法**即可从 guest 地址得到 host 地址，无需任何额外查表。

---

## 4. TLB 标志位系统

### 4.1 快速路径标志（addr_idx 低位）

```c
// include/exec/tlb-flags.h:66-79
TLB_INVALID_MASK  = (1 << 6)   // 条目无效
TLB_NOTDIRTY      = (1 << 7)   // 干净 RAM 页（写时需通知）
TLB_FORCE_SLOW    = (1 << 8)   // 强制慢路径（更多标志在 Full 条目中）

TLB_FLAGS_MASK = TLB_INVALID_MASK | TLB_NOTDIRTY | TLB_FORCE_SLOW
```

### 4.2 慢路径标志（CPUTLBEntryFull.slow_flags）

```c
// include/exec/tlb-flags.h:44-58
TLB_BSWAP           = (1 << 0)  // 需要字节序交换
TLB_WATCHPOINT      = (1 << 1)  // 有 watchpoint
TLB_CHECK_ALIGNED   = (1 << 2)  // 需对齐检查（设备内存）
TLB_DISCARD_WRITE   = (1 << 3)  // 写入被丢弃（ROM）
TLB_MMIO            = (1 << 4)  // MMIO 访问（通过 MemoryRegion 分发）
```

**两级标志设计**：快速路径只检查 3 个标志位（在 addr_idx 低位），如果 `TLB_FORCE_SLOW` 被设置，才读取 `slow_flags` 做更细粒度检查。

---

## 5. ARMMMUIdx — 22 种 MMU 索引

```c
// target/arm/mmuidx.h:137-198
typedef enum ARMMMUIdx {
    // === A-profile 常规翻译 ===
    E10_0      = 0,   // EL0 in EL1&0 regime
    E10_0_GCS  = 1,   // EL0 + GCS
    E10_1      = 2,   // EL1 in EL1&0 regime
    E10_1_PAN  = 3,   // EL1 + PAN
    E10_1_GCS  = 4,   // EL1 + GCS

    E20_0      = 5,   // EL0 in EL2&0 regime (VHE)
    E20_0_GCS  = 6,   // EL0 + GCS (VHE)
    E20_2      = 7,   // EL2 in EL2&0 regime (VHE)
    E20_2_PAN  = 8,   // EL2 + PAN (VHE)
    E20_2_GCS  = 9,   // EL2 + GCS (VHE)

    E2         = 10,  // EL2 non-VHE
    E2_GCS     = 11,  // EL2 + GCS

    E3         = 12,  // EL3
    E3_GCS     = 13,  // EL3 + GCS
    E30_0      = 14,  // EL0 in EL3&0 regime
    E30_3_PAN  = 15,  // EL3 + PAN

    // === Stage-2 翻译 ===
    Stage2_S   = 16,  // Secure Stage-2
    Stage2     = 17,  // Non-secure Stage-2

    // === 物理地址空间（无翻译）===
    Phys_S     = 18,  // Secure physical
    Phys_NS    = 19,  // Non-secure physical
    Phys_Root  = 20,  // Root physical
    Phys_Realm = 21,  // Realm physical
} ARMMMUIdx;
```

每个 MMU 索引对应独立的 TLB 分区（`CPUTLB.f[idx]` + `CPUTLB.d[idx]`）。

---

## 6. 快速路径 — TLB 命中

### 6.1 TCG 后端内联 TLB 查找（AArch64 host）

参见 arm64/42 文档中 `prepare_host_addr`（tcg-target.c.inc:1666-1728）的详细分析。快速路径总结：

```asm
; AArch64 后端生成的内联代码（概念性）
LDP  TMP0, TMP1, [AREG0, #tlb_offset]   ; 加载 mask + table
AND  TMP0, TMP0, addr, LSR #PAGE_BITS   ; 计算 TLB 索引
ADD  TMP1, TMP1, TMP0                   ; TLB 条目地址
LDR  TMP0, [TMP1, #addr_read]           ; 加载比较地址
LDR  TMP1, [TMP1, #addend]              ; 加载 addend
AND  TMP2, addr, #PAGE_MASK             ; 页对齐
CMP  TMP0, TMP2                         ; 比较
B.NE slow_path                          ; 不匹配 → 慢路径
; 命中：host_addr = addr + TMP1(addend)
LDR  data, [TMP1, addr]                 ; 直接内存访问
```

### 6.2 probe_access_internal

```c
// accel/tcg/cputlb.c:1360-1412
static int probe_access_internal(CPUState *cpu, vaddr addr,
                                 int size, MMUAccessType access_type,
                                 int mmu_idx, bool nonfault,
                                 void **phost, CPUTLBEntryFull **pfull,
                                 uintptr_t retaddr, bool check_mem_cbs)
{
    // 1. 计算 TLB 索引并读取条目
    // 2. 检查标志位（TLB_INVALID_MASK/TLB_FLAGS_MASK）
    // 3. 命中 → *phost = addr + addend
    // 4. 未命中 → 返回错误码触发慢路径
}
```

---

## 7. 慢路径 — TLB 未命中与填充

### 7.1 TLB 填充回调

```c
// target/arm/tcg/tlb_helper.c:331-379
bool arm_cpu_tlb_fill_align(CPUState *cs, CPUTLBEntryFull *out,
                            vaddr address, MMUAccessType access_type,
                            int mmu_idx, MemOp memop, int size,
                            bool probe, uintptr_t ra)
{
    // 1. 对齐检查 (359-363)
    if (address & alignment_mask) → ARMFault_Alignment

    // 2. 页表遍历 (364-366)
    if (!get_phys_addr(&cpu->env, address, access_type, memop,
                       core_to_arm_mmu_idx(&cpu->env, mmu_idx),
                       &res, fi)) {
        // 3. 成功：填充 CPUTLBEntryFull (367-370)
        res.f.extra.arm.pte_attrs = res.cacheattrs.attrs;
        res.f.extra.arm.shareability = res.cacheattrs.shareability;
        *out = res.f;
        return true;
    }

    // 4. 失败：生成异常 (377-378)
    cpu_restore_state(cs, ra);
    arm_deliver_fault(cpu, address, access_type, mmu_idx, fi);
}
```

---

## 8. ARM 页表遍历 — get_phys_addr

### 8.1 入口函数

```c
// target/arm/ptw.c:3931-3943
bool get_phys_addr(CPUARMState *env, vaddr address,
                   MMUAccessType access_type, MemOp memop,
                   ARMMMUIdx mmu_idx, GetPhysAddrResult *result,
                   ARMMMUFaultInfo *fi)
{
    // 构建 S1Translate 上下文
    // 调用 get_phys_addr_gpc()（含 GPC 检查）
}
```

### 8.2 调用链

```
get_phys_addr()                           [3931]
  → get_phys_addr_gpc()                   [3800-3833]
    → get_phys_addr_nogpc()               [3661-3797]
      ├── MMU 禁用 → get_phys_addr_disabled() [3470]
      ├── PMSA → get_phys_addr_pmsav7/v8() [2619/3151]
      ├── VMSA short → get_phys_addr_v5/v6() [1056/1179]
      ├── VMSA LPAE → get_phys_addr_lpae()   [1859]
      └── 两阶段 → get_phys_addr_twostage() [3554]
```

---

## 9. LPAE 长描述符格式页表遍历

```c
// target/arm/ptw.c:1859-2449
static bool get_phys_addr_lpae(CPUARMState *env, S1Translate *ptw,
                               vaddr address, MMUAccessType access_type,
                               bool s1_is_el0, GetPhysAddrResult *result,
                               ARMMMUFaultInfo *fi)
{
    // 第1步：确定起始级别和粒度 (1884-2012)
    // 4KB → 级别 0-3，16KB → 级别 0-3，64KB → 级别 1-3
    // 根据 TCR.T0SZ/T1SZ 确定输入地址范围
    // 计算起始级别的 TTBR + 索引

    // 第2步：页表遍历循环 (2014-2126)
    for (level = start_level; level <= 3; level++) {
        // 读取描述符 (2050-2070)
        descriptor = arm_ldq_ptw(env, ptw, descaddr, fi);

        // 检查有效位 (2089-2094)
        if (!(descriptor & 1)) → Translation Fault

        // 表描述符 vs 块/页描述符 (2115-2142)
        if (descriptor & 2) {
            // 块/页描述符 → 遍历结束
            break;
        } else {
            // 表描述符 → 提取下一级表地址
            descaddr = descriptor & TABLE_ADDR_MASK;
            // 累积表描述符的 AP/XN 限制
        }
    }

    // 第3步：提取物理地址和属性 (2128-2417)
    // AF 检查和 DBM 处理 (2143-2178)
    // S1 vs S2 属性提取 (2181-2347)
    // 权限检查 (2349-2382)
    // 填充结果 (2419-2435)
}
```

### 页表级别与地址位分配（4KB 粒度）

```
63    48 47      39 38      30 29      21 20      12 11       0
┌───────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ 保留  │ L0 索引  │ L1 索引  │ L2 索引  │ L3 索引  │ 页内偏移 │
│       │ (9 bits) │ (9 bits) │ (9 bits) │ (9 bits) │ (12 bits)│
└───────┴──────────┴──────────┴──────────┴──────────┴──────────┘
         ↓          ↓          ↓          ↓
       512GB块    1GB块      2MB块      4KB页
```

---

## 10. 两阶段翻译 S1→S2

```c
// target/arm/ptw.c:3554-3658
static bool get_phys_addr_twostage(CPUARMState *env, S1Translate *ptw,
                                   vaddr address, ...)
{
    // 第1步：Stage-1 翻译 (3568)
    ret = get_phys_addr_nogpc(env, ptw, address, ...);
    // 得到 IPA（Intermediate Physical Address）

    // 第2步：Stage-2 翻译 (3595)
    ipa = result->f.phys_addr;
    ret = get_phys_addr_nogpc(env, ptw, ipa, ...);
    // 得到 PA（Physical Address）

    // 第3步：权限合并 (3600)
    // result.prot = s1_prot & s2_prot

    // 第4步：缓存属性组合 (3640-3641)
    cacheattrs = combine_cacheattrs(hcr, s1_cacheattrs, s2_cacheattrs);
}
```

---

## 11. 缓存属性与 MAIR

### 11.1 MAIR 属性提取

```c
// ptw.c:2334-2347  S1 MAIR 提取
// AttrIndx[2:0] 从页表描述符 bits[4:2] 提取
// 索引 MAIR_EL1 的对应 8 位字段
// 得到 Normal/Device 类型 + 内外缓存策略

// ptw.c:2226-2234  S2 属性提取
// S2 描述符 bits[5:2] 直接编码内存类型
```

### 11.2 缓存属性组合

```c
// ptw.c:3413-3463
static ARMCacheAttrs combine_cacheattrs(uint64_t hcr,
                                        ARMCacheAttrs s1, ARMCacheAttrs s2)
{
    if (hcr & HCR_FWB) {
        // FEAT_S2FWB：S2 可覆盖 S1 属性 (3366-3402)
        return combined_attrs_fwb(s1, s2);
    } else {
        // 传统方式：取 S1 和 S2 的较弱属性 (3302-3339)
        return combined_attrs_nofwb(hcr, s1, s2);
    }
}
```

---

## 12. tlb_set_page_full — TLB 安装

```c
// accel/tcg/cputlb.c:1024-1207
void tlb_set_page_full(CPUState *cpu, int mmu_idx,
                       vaddr addr, CPUTLBEntryFull *full)
{
    // 第1步：页大小处理 (1040-1045)
    // 大页 → tlb_add_large_page 记录大页区域

    // 第2步：物理地址到 MemoryRegionSection 映射 (1051-1052)
    section = address_space_translate_for_iotlb(cpu, asidx, paddr_page, ...);

    // 第3步：计算 TLB 标志 (1059-1103)
    if (is_ram) {
        if (需要脏追踪) read_flags |= TLB_NOTDIRTY;
        if (ROM) write_flags |= TLB_DISCARD_WRITE;
    } else {
        // MMIO
        write_flags |= TLB_MMIO;  // 1092-1103
    }
    if (slow_flags) read_flags |= TLB_FORCE_SLOW;

    // 第4步：计算 addend (1108-1115)
    addend = (uintptr_t)memory_region_get_ram_ptr(mr) + xlat - addr_page;

    // 第5步：填充 CPUTLBEntry (1117-1160)
    tn.addend = addend;
    tn.addr_read = addr_page | (prot & PAGE_READ ? 0 : TLB_INVALID_MASK);
    tn.addr_write = addr_page | (prot & PAGE_WRITE ? 0 : TLB_INVALID_MASK);
    tn.addr_code = addr_page | (prot & PAGE_EXEC ? 0 : TLB_INVALID_MASK);
    // 加上标志位

    // 第6步：安装到 TLB（可能驱逐到 victim cache）(1126-1138)
    // 旧条目移入 vtable[vindex]
    copy_tlb_helper_locked(te, &tn);
    desc->fulltlb[index] = *full;
}
```

---

## 13. MMIO 分发路径

```
Guest Store 到 MMIO 地址
  │
  ├── TLB 查找 → 命中但 TLB_MMIO 标志
  │   → TLB_FORCE_SLOW → 进入慢路径
  │
  └── 慢路径：
      1. io_prepare()  [cputlb.c:1272-1288]
         → 从 CPUTLBEntryFull 获取 MemoryRegionSection
         → 计算 MMIO 偏移 = xlat_offset + (addr & ~TARGET_PAGE_MASK)

      2. memory_region_dispatch_write()  [system/memory.c:1467-1526]
         → mr->ops->write(opaque, offset, val, size)
         → 调用设备的 MMIO 写回调函数
```

---

## 14. TLB 刷新与 TLBI 指令

### 14.1 核心 TLB 刷新函数

```c
// accel/tcg/cputlb.c:418-436
void tlb_flush(CPUState *cpu)               // 全量刷新
void tlb_flush_by_mmuidx(cpu, idxmap)       // 按 MMU 索引刷新
void tlb_flush_all_cpus_synced(cpu)          // 跨 CPU 同步刷新
void tlb_flush_page(cpu, addr)              // 单页刷新
void tlb_flush_page_by_mmuidx(cpu, addr, idxmap)  // 按索引+页刷新
```

### 14.2 ARM TLBI 指令实现

```c
// target/arm/tcg/tlb-insns.c

// AArch32 TLBI
tlbiall_write()    // 92-103:  TLBIALL — 刷新所有 EL1&0
tlbimva_write()    // 105-117: TLBIMVA — 按 VA+ASID 刷新
tlbiasid_write()   // 119-130: TLBIASID — 按 ASID 刷新
tlbimvaa_write()   // 132-144: TLBIMVAA — 按 VA（忽略 ASID）

// AArch64 TLBI（关键子集）
tlbi_aa64_vmalle1_write()  // 317-337:  VMALLE1 — EL1&0 全部刷新
tlbi_aa64_alle1_write()    // 350-357:  ALLE1 — 所有 EL1&0（含 ASID）
tlbi_aa64_alle2_write()    // 359-366:  ALLE2 — 所有 EL2
tlbi_aa64_alle3_write()    // 368-375:  ALLE3 — 所有 EL3
tlbi_aa64_vae1_write()     // 434-464:  VAE1 — 按 VA+ASID（EL1&0）
tlbi_aa64_vae2_write()     // 466-475:  VAE2 — 按 VA（EL2）
tlbi_aa64_ipas2e1_write()  // 488-523:  IPAS2E1 — 按 IPA（Stage-2）

// 范围 TLBI（FEAT_TLBIRANGE）
tlbi_aa64_get_range()      // 843-870:  解析范围参数
do_rvae_write()            // 872-900:  范围 VA 刷新
```

### 14.3 IS 变体（Inner-Shareable 广播）

IS 后缀变体使用 `tlb_flush_*_all_cpus_synced()` 将刷新广播到所有共享域内的 CPU。

---

## 15. AT 指令实现

```c
// target/arm/tcg/cpregs-at.c:26-420

// AArch32 AT
do_ats_write()     // 26-190:  通用 AT 实现
                   // 调用 get_phys_addr_for_at() → get_phys_addr()
                   // 结果写入 PAR（Physical Address Register）

// AArch64 AT
ats_write64()      // 310-420: AT S1E1R/W, S1E0R/W, S1E2R/W, S12E1R/W 等
                   // 选择正确的 MMU 索引
                   // 调用 do_ats_write → PTW → PAR_EL1
```

**AT 与 TLB 的关系**：AT 使用相同的 PTW 路径（`get_phys_addr`），但结果写入 PAR_EL1 而非安装到 TLB。AT 可用于软件调试和 OS 页表管理。

---

## 16. 原子操作与 TLB

```c
// accel/tcg/cputlb.c:1799-1903
static void *atomic_mmu_lookup(CPUState *cpu, vaddr addr,
                               MemOpIdx oi, int size, uintptr_t retaddr)
{
    // 1. 读取 TLB 写地址 (1820)
    tlb_addr = tlb_addr_write(entry);

    // 2. TLB 命中检查 (1830-1840)
    if (!tlb_hit(tlb_addr, addr)) {
        // 3. 检查 victim cache (1845-1860)
        // 4. 仍未命中 → tlb_fill_align (1865-1870)
    }

    // 5. 检查对齐要求 (1875-1880)
    // 原子操作通常要求自然对齐

    // 6. 返回 host 地址 (1895-1900)
    return (void *)(addr + entry->addend);
}
```

原子操作最终在 host 内存上执行原生原子指令（如 AArch64 的 LDXR/STXR 或 LSE 原子），因此必须确保返回的 host 地址指向实际 RAM。

---

## 17. 脏页追踪 — notdirty_write

```c
// accel/tcg/cputlb.c:1336-1358
static void notdirty_write(CPUState *cpu, vaddr mem_vaddr,
                           unsigned size, CPUTLBEntryFull *full,
                           uintptr_t retaddr)
{
    // 1. 计算 ram_addr
    ram_addr_t ram_addr = mem_vaddr + full->xlat_offset;

    // 2. 通知脏页追踪系统
    // 标记该 RAM 页为脏
    // 用于实时迁移、VGA 更新等

    // 3. 清除 TLB_NOTDIRTY 标志
    // 后续写入不再经过此路径（直到下次标记为干净）
}
```

**工作原理**：
- `tlb_set_page_full` 安装 TLB 时，如果页是干净的，写地址加上 `TLB_NOTDIRTY` 标志
- 首次写入触发 `notdirty_write`，通知迁移/VGA 子系统
- 清除标志后后续写入直接走快速路径

---

## 18. TLB 动态调整与 Victim Cache

### 18.1 动态调整

```c
// accel/tcg/cputlb.c:164-277
// TLB 大小在运行时根据访问模式动态调整：
// - 统计窗口（window_begin_ns, window_max_entries）
// - 如果命中率低 → 增大 TLB
// - 如果大部分条目未使用 → 缩小 TLB
// - 大小范围：256 到 4096 个条目（2^8 到 2^12）
```

### 18.2 Victim Cache

```c
// include/hw/core/cpu.h:294-295
CPUTLBEntry vtable[CPU_VTLB_SIZE];      // 8 个 victim 条目
CPUTLBEntryFull vfulltlb[CPU_VTLB_SIZE];
```

**策略**：被驱逐的 TLB 条目移入 8 条目全关联 victim cache。TLB 未命中时先查 victim cache，命中则与主 TLB 交换（swap），避免重新执行 PTW。

```c
// cputlb.c:1304-1333  victim cache 查找
for (vidx = 0; vidx < CPU_VTLB_SIZE; ++vidx) {
    if (victim_tlb_hit(cpu, mmu_idx, index, access_type, addr)) {
        // 将 victim 条目与主 TLB 条目交换
        CPUTLBEntry tmpentry = vtable[vidx];
        vtable[vidx] = *te;
        *te = tmpentry;
        // 同样交换 fulltlb
        return;  // 不需要 PTW
    }
}
```

---

## 19. 异常生成 — arm_deliver_fault

```c
// target/arm/tcg/tlb_helper.c:173-273
static void arm_deliver_fault(ARMCPU *cpu, vaddr addr,
                              MMUAccessType access_type,
                              int mmu_idx, ARMMMUFaultInfo *fi)
{
    int target_el = exception_target_el(env);  // 179: 确定目标 EL

    // GPC 异常处理（FEAT_RME）(200-229)
    if (report_as_gpc_exception(cpu, current_el, fi)) {
        target_el = 3;
        syn = syn_gpc(...);
        exc = EXCP_GPC;
    }

    // Stage-2 异常路由到 EL2 (239-245)
    if (fi->stage2) {
        target_el = 2;
        env->cp15.hpfar_el2 = extract64(fi->s2addr, 12, 47) << 4;
    }

    // 构建 syndrome (250-267)
    if (access_type == MMU_INST_FETCH) {
        syn = syn_insn_abort(same_el, fi->ea, fi->s1ptw, fsc);
        exc = EXCP_PREFETCH_ABORT;
    } else {
        syn = merge_syn_data_abort(...);
        exc = EXCP_DATA_ABORT;
    }

    // 触发异常 (270-272)
    env->exception.vaddress = addr;
    env->exception.fsr = fsr;
    raise_exception(env, exc, syn, target_el);
}
```

### 故障类型

| 故障 | ARMFault_* | 触发位置 |
|------|-----------|---------|
| Translation Fault | ARMFault_Translation | ptw.c:2089-2094（描述符无效） |
| Access Flag Fault | ARMFault_AccessFlag | ptw.c:2143-2148（AF=0） |
| Permission Fault | ARMFault_Permission | ptw.c:2379-2382（权限检查失败） |
| Alignment Fault | ARMFault_Alignment | tlb_helper.c:286-287 |

---

## 20. 完整内存访问流程总结

```
Guest LDR X0, [X1]
  │
  ├── TCG 翻译：qemu_ld op → 后端内联 TLB 查找代码
  │
  ├── 运行时快速路径：
  │   1. index = (X1 >> PAGE_BITS) & mask         TLB 索引
  │   2. tlb_addr = table[index].addr_read         读取比较地址
  │   3. if (X1 & PAGE_MASK) == (tlb_addr & MASK)  命中检查
  │   4. host_addr = X1 + table[index].addend       计算主机地址
  │   5. X0 = *host_addr                            直接内存读取
  │   └── 完成！（~5 条 host 指令）
  │
  ├── TLB 未命中（慢路径）：
  │   1. helper_ld64_mmu(env, addr, oi, retaddr)
  │   2. → victim cache 查找（8 条目）
  │   3. → 仍未命中 → tlb_fill_align()
  │   4.   → arm_cpu_tlb_fill_align()
  │   5.     → get_phys_addr()                    页表遍历
  │   6.       → get_phys_addr_lpae()             LPAE 4级遍历
  │   7.         → 返回 phys_addr + attrs
  │   8.     → tlb_set_page_full()                安装 TLB
  │   9.   → 重试快速路径 → 命中
  │
  └── MMIO 路径：
      1. TLB 命中但 TLB_FORCE_SLOW + TLB_MMIO
      2. → io_prepare() → 获取 MemoryRegionSection
      3. → memory_region_dispatch_read(mr, offset, ...)
      4. → mr->ops->read(opaque, offset, size)
      5. → 设备返回数据
```

---

## 交叉参考

- [42-ARM64-TCG前端后端代码生成深度分析](42-ARM64-TCG前端后端代码生成深度分析-IR中间表示-翻译循环-优化Pass-寄存器分配与AArch64代码发射.md) — prepare_host_addr TLB 内联代码
- [41-ARM64-EL切换TCG翻译变化深度分析](41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md) — MMUIDX 与 hflags
- [34-ARM64-MMU页表遍历深度分析](34-ARM64-MMU页表遍历深度分析-S1-S2翻译-LPAE描述符-TLB维护-AT指令与故障处理.md) — PTW 详细分析

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
