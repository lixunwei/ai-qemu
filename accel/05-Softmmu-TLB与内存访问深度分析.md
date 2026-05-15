# Softmmu TLB 与内存访问深度分析

> QEMU 11.0.50 · 分析日期 2025-07 · 基于源码交叉验证

## 目录

1. [概述](#1-概述)
2. [TLB 数据结构总览](#2-tlb-数据结构总览)
3. [CPUTLBEntry — 快路径条目](#3-cputlbentry--快路径条目)
4. [CPUTLBDescFast — 快路径描述符](#4-cputlbdescfast--快路径描述符)
5. [CPUTLBEntryFull — 完整条目](#5-cputlbentryfull--完整条目)
6. [CPUTLB 与 CPUNegativeOffsetState](#6-cputlb-与-cpunegativeoffsetstate)
7. [TLB 标志位体系](#7-tlb-标志位体系)
8. [TLB 索引与寻址](#8-tlb-索引与寻址)
9. [动态 TLB 大小调整](#9-动态-tlb-大小调整)
10. [Victim TLB](#10-victim-tlb)
11. [TLB 条目填充 — tlb_set_page_full()](#11-tlb-条目填充--tlb_set_page_full)
12. [内联快路径 — AArch64 后端](#12-内联快路径--aarch64-后端)
13. [慢路径入口 — helper_ld/st_mmu](#13-慢路径入口--helper_ldst_mmu)
14. [TLB 重填 — tlb_fill_align()](#14-tlb-重填--tlb_fill_align)
15. [mmu_lookup1() — 核心查找](#15-mmu_lookup1--核心查找)
16. [mmu_watch_or_dirty() — 特殊页处理](#16-mmu_watch_or_dirty--特殊页处理)
17. [MMIO 分发路径](#17-mmio-分发路径)
18. [跨页访问处理](#18-跨页访问处理)
19. [字节序处理](#19-字节序处理)
20. [脏页追踪集成](#20-脏页追踪集成)
21. [Watchpoint 处理](#21-watchpoint-处理)
22. [TLB 刷新操作](#22-tlb-刷新操作)
23. [ARM64 页表遍历 — arm_cpu_tlb_fill_align()](#23-arm64-页表遍历--arm_cpu_tlb_fill_align)
24. [地址翻译完整链路](#24-地址翻译完整链路)
25. [io_prepare() 与 MemoryRegion 分发](#25-io_prepare-与-memoryregion-分发)
26. [完整 qemu_ld 执行示例](#26-完整-qemu_ld-执行示例)
27. [附录 A：关键源文件索引](#27-附录-a关键源文件索引)

---

## 1. 概述

Softmmu 是 QEMU 系统模式下的内存访问核心——在没有硬件虚拟化（KVM）的情况下，所有客户机内存访问都经过软件 TLB 翻译。

**核心机制**：
- **快路径**：编译到 TB 代码中的内联 TLB 检查（~10 条宿主指令）
- **慢路径**：TLB 未命中 → 页表遍历 → TLB 填充 → MMIO/watchpoint 分发
- **地址翻译链**：Guest VA → TLB → Physical Addr → MemoryRegion → 设备回调

**关键数字**：
- `cputlb.c`：**~2800 行**，TLB 管理与内存访问实现
- `CPUTLBEntry`：**32 字节**（64 位系统），快路径结构
- `CPUTLBEntryFull`：**~64 字节**，完整条目（慢路径使用）
- `NB_MMU_MODES = 22`：ARM64 有 22 种 MMU 模式
- `CPU_VTLB_SIZE = 8`：Victim TLB 8 条目

---

## 2. TLB 数据结构总览

```
CPUState
└── neg (CPUNegativeOffsetState)                cpu.h:363-376
    └── tlb (CPUTLB)                            cpu.h:327-333
        ├── c (CPUTLBCommon)                    cpu.h:302-319
        │   ├── lock (QemuSpin)                 — 串行化更新
        │   ├── dirty (MMUIdxMap)               — 脏位图
        │   └── 统计计数器                       — flush_count 等
        │
        ├── d[NB_MMU_MODES] (CPUTLBDesc)        cpu.h:274-297
        │   ├── n_used_entries                  — 当前使用条目数
        │   ├── large_page_addr/mask            — 大页追踪
        │   ├── window_{max_entries,begin_ns}   — 动态调整窗口
        │   ├── vindex                          — victim 轮转索引
        │   ├── vtable[8] (CPUTLBEntry)         — victim 快路径
        │   ├── vfulltlb[8] (CPUTLBEntryFull)   — victim 完整
        │   └── fulltlb* (CPUTLBEntryFull[])    — 主完整 TLB
        │
        └── f[NB_MMU_MODES] (CPUTLBDescFast)    cpu.h:331
            ├── mask                            — (n_entries-1) << BITS
            └── table* (CPUTLBEntry[])          — 主快路径 TLB
```

---

## 3. CPUTLBEntry — 快路径条目

**定义**：`include/exec/tlb-common.h:25-41`

```c
typedef union CPUTLBEntry {
    struct {
        uintptr_t addr_read;      // :27 — 读地址（含标志位）
        uintptr_t addr_write;     // :28 — 写地址（含标志位）
        uintptr_t addr_code;      // :29 — 取指地址（含标志位）
        uintptr_t addend;         // :34 — 虚拟地址→宿主地址偏移
    };
    uintptr_t addr_idx[...];      // :40 — 索引访问
} CPUTLBEntry;
// sizeof = 32 字节 (64位)，1 << CPU_TLB_ENTRY_BITS
```

**地址比较机制**：
- `addr_read/write/code` 存储的是 `(客户机页地址 | 标志位)`
- 比较时：`(addr_in_entry ^ guest_va) & (TARGET_PAGE_MASK | alignment)` 为 0 则命中
- 标志位占用低位（页对齐以下的位）：`TLB_INVALID_MASK`、`TLB_NOTDIRTY`、`TLB_FORCE_SLOW`

**addend 的魔法**：
```
host_addr = guest_va + addend
```
TLB 命中时，一条加法即可将客户机虚拟地址转为宿主地址，实现零开销地址翻译。

---

## 4. CPUTLBDescFast — 快路径描述符

**定义**：`include/exec/tlb-common.h:49-54`

```c
typedef struct CPUTLBDescFast {
    uintptr_t mask;              // :51 — (n_entries - 1) << CPU_TLB_ENTRY_BITS
    CPUTLBEntry *table;          // :53 — TLB 条目数组指针
} CPUTLBDescFast QEMU_ALIGNED(2 * sizeof(void *));
```

**16 字节对齐**：`mask` 和 `table` 紧邻且对齐，AArch64 后端可用单条 `LDP` 同时加载两者。

---

## 5. CPUTLBEntryFull — 完整条目

**定义**：`include/hw/core/cpu.h:214-260`

```c
struct CPUTLBEntryFull {
    hwaddr xlat_offset;                 // :222 — 页内翻译偏移
    MemoryRegionSection *section;       // :225 — 物理内存区域
    hwaddr phys_addr;                   // :231 — 物理地址
    MemTxAttrs attrs;                   // :234 — 事务属性
    uint8_t prot;                       // :237 — 页保护位 (R/W/X)
    uint8_t lg_page_size;               // :240 — log2(页大小)
    uint8_t tlb_fill_flags;             // :243 — 填充标志
    uint8_t slow_flags[MMU_ACCESS_COUNT]; // :249 — 慢路径标志
    union { ... } extra;                // :256 — 目标特定扩展
};
```

**与 CPUTLBEntry 的关系**：
- `CPUTLBEntry`（快路径）：只存地址比较值 + addend，32 字节
- `CPUTLBEntryFull`（慢路径）：存完整翻译信息，由 `fulltlb[index]` 索引
- 两者通过相同的 TLB 索引关联

---

## 6. CPUTLB 与 CPUNegativeOffsetState

### CPUNegativeOffsetState

**定义**：`include/hw/core/cpu.h:355-376`

```c
typedef struct CPUNegativeOffsetState {
    CPUTLB tlb;                         // :364
    IcountDecr icount_decr;             // :374
    bool can_do_io;                     // :375
} CPUNegativeOffsetState;
```

### 负偏移优化

`CPUNegativeOffsetState` 位于 `CPUState` 末尾（通过 `MUST BE LAST` 强制）：

```
内存布局：
                    ┌──────────────────────┐
低地址              │ CPUNegativeOffsetState│ ← 负偏移
                    │  ├── CPUTLB          │
                    │  │  ├── c            │
                    │  │  ├── d[22]        │
                    │  │  └── f[22]        │ ← 最接近 env
                    │  ├── icount_decr     │
                    │  └── can_do_io       │
                    ├──────────────────────┤
env 指针 ──────────►│ CPUArchState         │ ← 零偏移
                    │  (target/arm/cpu.h)  │
                    └──────────────────────┘
```

**设计目标**：TB 代码通过 `env`（X19/AREG0）的小负偏移访问 TLB，在 AArch64 后端中可用紧凑的 LDR 指令编码。

---

## 7. TLB 标志位体系

### 快路径标志（addr_idx 低位）

**定义**：`include/exec/tlb-flags.h:60-79`

| 标志 | 位 | 含义 |
|------|------|------|
| `TLB_INVALID_MASK` | bit 6 | 条目无效 |
| `TLB_NOTDIRTY` | bit 7 | 干净 RAM 页（写触发脏页记录） |
| `TLB_FORCE_SLOW` | bit 8 | 强制走慢路径（有更多标志在 slow_flags 中） |

```c
#define TLB_FLAGS_MASK  (TLB_INVALID_MASK | TLB_NOTDIRTY | TLB_FORCE_SLOW)  // :78
```

### 慢路径标志（slow_flags）

**定义**：`include/exec/tlb-flags.h:45-58`

| 标志 | 位 | 含义 |
|------|------|------|
| `TLB_BSWAP` | bit 0 | 需要字节序交换 |
| `TLB_WATCHPOINT` | bit 1 | 包含监视点 |
| `TLB_CHECK_ALIGNED` | bit 2 | 需要对齐检查 |
| `TLB_DISCARD_WRITE` | bit 3 | 写入被丢弃 |
| `TLB_MMIO` | bit 4 | IO 回调 |

**两级标志设计**：快路径只检查 3 个标志位（全零 = 快速通过），慢路径才检查 5 个额外标志。

---

## 8. TLB 索引与寻址

### 索引计算

**定义**：`accel/tcg/cputlb.c:126-140`

```
index = (guest_va >> TARGET_PAGE_BITS) & (mask >> CPU_TLB_ENTRY_BITS)
```

即：
1. 客户机虚拟地址右移页位数，得到页号
2. 页号与掩码做 AND，得到直接映射索引
3. 索引用于访问 `f[mmu_idx].table[index]`

### MMU 索引反转

**定义**：`include/hw/core/cpu.h` 中 `mmuidx_to_fast_index`

TLB 快路径数组 `f[]` 的索引顺序被反转，使得最常用的 MMU 模式在数组末尾（最接近 `env`），编码为最小的负偏移。

---

## 9. 动态 TLB 大小调整

**定义**：`accel/tcg/cputlb.c:165-277`

TLB 大小可在运行时动态增长/缩小：

```
窗口统计（100ms 周期）：
│
├── 追踪 n_used_entries / window_max_entries
│
├── 如果使用率高（接近满）
│   → 扩大 TLB（大小翻倍），最大 256K 条目
│
├── 如果使用率低
│   → 缩小 TLB（大小减半），最小 256 条目
│
└── 调整触发 TLB flush + 重新分配 table/fulltlb
```

**调整逻辑**：
- `window_begin_ns`：统计窗口起始时间
- `window_max_entries`：窗口内最大使用数
- 每次 `n_used_entries` 更新时检查是否需要扩容
- 每次 flush 时检查是否可以缩容

---

## 10. Victim TLB

**定义**：`include/hw/core/cpu.h:206-207`

```c
#define CPU_VTLB_SIZE 8  // 8 条目全相联 victim 缓存
```

### 结构

```c
CPUTLBDesc {
    ...
    uint16_t vindex;                        // victim 轮转索引
    CPUTLBEntry vtable[CPU_VTLB_SIZE];      // victim 快路径条目
    CPUTLBEntryFull vfulltlb[CPU_VTLB_SIZE]; // victim 完整条目
};
```

### 工作原理

1. 主 TLB 直接映射——冲突时旧条目被驱逐到 victim TLB（`cputlb.c:1123-1138`）
2. 主 TLB 未命中时，搜索 victim TLB（全相联查找）
3. victim 命中 → 交换到主 TLB（swap victim ↔ main）
4. victim 满时 → 轮转替换（`vindex++`）

---

## 11. TLB 条目填充 — tlb_set_page_full()

**定义**：`accel/tcg/cputlb.c:1024-1183`

```
tlb_set_page_full(cpu, mmu_idx, addr, full)
│
├── 计算页大小和对齐                               :1040-1046
│
├── address_space_translate_for_iotlb()            :1051-1052
│   → 物理地址 → MemoryRegionSection + xlat
│
├── 确定是否 RAM / ROMD                            :1059-1072
│
├── 设置读标志                                     :1059-1103
│   ├── is_ram && clean → TLB_NOTDIRTY
│   ├── !is_ram → TLB_MMIO（或 TLB_FORCE_SLOW）
│   └── watchpoint → TLB_WATCHPOINT
│
├── 设置写标志                                     :1104-1120
│   ├── 只读页 → TLB_INVALID_MASK（写触发 fault）
│   ├── MMIO → TLB_MMIO
│   └── 代码页 → write flag + TLB_NOTDIRTY
│
├── 处理旧条目                                     :1123-1138
│   旧条目 ≠ 当前页 → 移入 victim TLB
│
├── 填充 CPUTLBEntry                               :1140-1180
│   tn.addend = addr - vaddr + host_ram_addr
│   tn.addr_read = addr_page | read_flags
│   tn.addr_write = addr_page | write_flags
│   tn.addr_code = addr_page | read_flags
│   atomic_set 写入 table[index]
│
└── 填充 CPUTLBEntryFull                           
    full->section = section
    full->xlat_offset = xlat - addr_page
```

---

## 12. 内联快路径 — AArch64 后端

### prepare_host_addr()

**定义**：`tcg/aarch64/tcg-target.c.inc:1645-1755`

这是编译到每个 `qemu_ld`/`qemu_st` 操作的内联 TLB 检查序列：

```asm
; ====== TLB 快路径 (AArch64 宿主) ======

; 1. 加载 {mask, table}（LDP 配对加载）               :1672-1680
LDP  X_mask, X_table, [AREG0, #tlb_f_offset]
; tlb_f_offset = offsetof(CPUNeg, tlb.f[mmu_idx])
; 利用 CPUTLBDescFast 的 16 字节对齐，一条指令加载两个字段

; 2. 计算 TLB 索引                                    :1685-1690
AND  X_idx, X_addr, X_mask, LSR #(TARGET_PAGE_BITS - CPU_TLB_ENTRY_BITS)
; (guest_va >> page_bits) & (mask >> entry_bits)

; 3. 获取 TLB 条目地址                                :1693
ADD  X_entry, X_table, X_idx
; entry = &table[index]

; 4. 加载比较器和 addend                               :1697-1704
LDP  X_comparator, X_addend, [X_entry, #addr_read_offset]
; 加载 addr_read (或 addr_write) 和 addend

; 5. 比较客户机地址                                    :1710-1720
; 构造比较值：(addr & (PAGE_MASK | alignment)) ^ comparator
AND  X_tmp, X_addr, #(PAGE_MASK | align_mask)
CMP  X_tmp, X_comparator
B.NE slow_path_label
; 不匹配 → 跳转慢路径

; 6. TLB 命中！计算宿主地址                            :1724-1730
ADD  X_host, X_addr, X_addend
; host_addr = guest_va + addend

; 7. 执行内存访问                                      
LDR  X_dest, [X_host]           ; 加载
; 或
STR  X_src, [X_host]            ; 存储
```

### 快路径开销

| 步骤 | 指令数 | 说明 |
|------|--------|------|
| 加载 mask+table | 1 | LDP（配对加载） |
| 计算索引 | 1 | AND+LSR |
| 获取条目 | 1 | ADD |
| 加载比较器+addend | 1 | LDP |
| 比较+分支 | 3 | AND + CMP + B.NE |
| 计算宿主地址 | 1 | ADD |
| **总计** | **~8 条** | TLB 命中时的额外开销 |

---

## 13. 慢路径入口 — helper_ld/st_mmu

### Helper 声明

**定义**：`include/tcg/tcg-ldst.h:29-61`

```c
// 加载 helper
uint8_t  helper_ldb_mmu(CPUArchState *env, ...);
uint16_t helper_ldw_mmu(CPUArchState *env, ...);
uint32_t helper_ldl_mmu(CPUArchState *env, ...);
uint64_t helper_ldq_mmu(CPUArchState *env, ...);
Int128   helper_ld16_mmu(CPUArchState *env, ...);

// 存储 helper
void helper_stb_mmu(CPUArchState *env, ...);
void helper_stw_mmu(CPUArchState *env, ...);
void helper_stl_mmu(CPUArchState *env, ...);
void helper_stq_mmu(CPUArchState *env, ...);
void helper_st16_mmu(CPUArchState *env, ...);
```

### AArch64 慢路径发射

**定义**：`tcg/aarch64/tcg-target.c.inc:1612-1638`

```
tcg_out_qemu_ld_slow_path(s, lb)
│
├── 从 ldst_label 中恢复参数
├── MOV X0, env (AREG0)
├── MOV X1, addr
├── MOV X2, oi (MemOpIdx)
├── MOV X3, retaddr
├── BL  qemu_ld_helpers[size]      — helper_ld{b,w,l,q}_mmu
├── MOV dest, X0                   — 返回值
└── B   continue_label             — 返回快路径继续点
```

---

## 14. TLB 重填 — tlb_fill_align()

**定义**：`accel/tcg/cputlb.c:1238-1262`

```
tlb_fill_align(cpu, addr, type, mmu_idx, memop, size, probe, ra)
│
├── ops = cpu->cc->tcg_ops
│
├── [新接口] ops->tlb_fill_align()                 :1245-1250
│   → 目标实现页表遍历
│   → 成功 → tlb_set_page_full()
│   → 失败 → 返回 false（probe 模式）或触发异常
│
└── [旧接口] ops->tlb_fill()                       :1252-1258
    → 先检查对齐
    → 调用旧版 tlb_fill
```

**注意**：`tlb_fill_align()` 可能触发 TLB 大小调整，调用者必须丢弃之前缓存的 TLB 条目指针。

---

## 15. mmu_lookup1() — 核心查找

**定义**：`accel/tcg/cputlb.c:1640-1682`

```
mmu_lookup1(cpu, data, memop, mmu_idx, access_type, ra)
│
├── 计算 TLB 索引和条目                             :1644-1646
│   index = tlb_index(cpu, mmu_idx, addr)
│   entry = tlb_entry(cpu, mmu_idx, addr)
│   tlb_addr = entry->addr_{read|write|code}
│
├── 检查 TLB 命中                                   :1652
│   if (!tlb_hit(tlb_addr, addr)):
│   │
│   ├── 搜索 victim TLB                             :1653-1654
│   │   victim_tlb_hit() → 找到则 swap 回主 TLB
│   │
│   └── victim 也未命中 → tlb_fill_align()          :1655-1657
│       → 页表遍历 + 填充 TLB
│       → 重新计算 index/entry（可能 resize）
│
├── 获取完整条目                                     :1664
│   full = &cpu->neg.tlb.d[mmu_idx].fulltlb[index]
│
├── 合并标志                                         :1665-1666
│   flags = (tlb_addr & TLB_FLAGS_MASK) | full->slow_flags[type]
│
├── [条件] 对齐检查                                  :1668-1673
│
└── 返回                                             :1676-1681
    data->full = full
    data->flags = flags
    data->haddr = addr + entry->addend
```

---

## 16. mmu_watch_or_dirty() — 特殊页处理

**定义**：`accel/tcg/cputlb.c:1694-1715`

```
mmu_watch_or_dirty(cpu, data, access_type, ra)
│
├── if (flags & TLB_WATCHPOINT):                    :1703-1706
│   wp = (store ? BP_MEM_WRITE : BP_MEM_READ)
│   cpu_check_watchpoint(cpu, addr, size, ...)
│   → 命中则 longjmp 退出
│   flags &= ~TLB_WATCHPOINT
│
└── if (flags & TLB_NOTDIRTY):                      :1710-1712
    notdirty_write(cpu, addr, size, full, ra)
    → 记录脏页 + 失效相关 TB
    flags &= ~TLB_NOTDIRTY
```

---

## 17. MMIO 分发路径

### 检测

TLB 条目的 `slow_flags` 包含 `TLB_MMIO` 时，访问被路由到 MMIO 路径。

### 读路径

```
do_ld_mmio_beN() / int_ld_mmio_beN()              cputlb.c:1936-1967
│
├── io_prepare(&mr_offset, cpu, full, addr, ra)    :1272-1288
│   section = full->section
│   mr_offset = full->xlat_offset + addr
│
├── memory_region_dispatch_read()                   system/memory.c:1444-1488
│   │
│   ├── 调整访问大小/对齐
│   └── mr->ops->read_with_attrs(mr, offset, &val, size, attrs)
│       → 最终调用设备的 MMIO 读回调
│
└── 返回读取值
```

### 写路径

```
do_st_mmio_leN() / int_st_mmio_leN()              cputlb.c:2450-2482
│
├── io_prepare()
│
├── memory_region_dispatch_write()                  system/memory.c:1517-1560
│   └── mr->ops->write_with_attrs(mr, offset, val, size, attrs)
│       → 最终调用设备的 MMIO 写回调
│
└── 检查返回状态
```

### io_prepare()

**定义**：`accel/tcg/cputlb.c:1272-1288`

```c
static MemoryRegionSection *
io_prepare(hwaddr *out_offset, CPUState *cpu, CPUTLBEntryFull *full,
           vaddr addr, uintptr_t retaddr)
{
    section = full->section;                        // :1279
    mr_offset = full->xlat_offset + addr;           // :1280
    cpu->mem_io_pc = retaddr;                       // :1281
    if (!cpu->neg.can_do_io) {
        cpu_io_recompile(cpu, retaddr);             // :1283
    }
    *out_offset = mr_offset;
    return section;
}
```

**can_do_io 检查**：TB 执行中 `can_do_io = false`（因为翻译的代码可能被优化而破坏 IO 语义），遇到 MMIO 时需要重新编译一个允许 IO 的 TB。

---

## 18. 跨页访问处理

当一次内存访问跨越两个页面时，拆分为两部分：

### 加载（以 8 字节为例）

**定义**：`accel/tcg/cputlb.c:2356-2367`

```
do_ld8_mmu(cpu, addr, memop, mmu_idx, ra)
│
├── mmu_lookup() — 查找两个页面                     :1718-1780
│   ├── mmu_lookup1(page1) — 第一页
│   └── mmu_lookup1(page2) — 第二页（如果跨页）
│
├── if (跨页):
│   ├── 加载 page1 部分（从 addr 到页边界）
│   ├── 加载 page2 部分（从页边界到 addr+8）
│   └── 拼接两部分数据
│
└── if (单页):
    └── 直接加载 8 字节
```

### 存储同理

`do_st8_mmu()`（`:2754-2766`）类似地将写入拆分为两次页面写入。

### 128 位访问

`do_ld16_mmu()`（`:2379-2427`）和 `do_st16_mmu()`（`:2777-2815`）处理 128 位跨页，拆分可能产生 4 次页面访问。

---

## 19. 字节序处理

### 加载端序

```
加载路径：
1. 从宿主内存以宿主端序读取                  — 直接 LDR
2. 检查 TLB_BSWAP 标志                       — cputlb.c:2237-2285
3. 如果需要交换 → bswap{16,32,64}()
4. 返回正确端序的值给客户机
```

### 存储端序

```
存储路径：
1. 接收客户机端序的值
2. 检查 TLB_BSWAP 标志                       — cputlb.c:2637-2687
3. 如果需要交换 → bswap{16,32,64}()
4. 以宿主端序写入宿主内存                  — 直接 STR
```

### MMIO 端序

MMIO 路径使用大端 (`beN`) / 小端 (`leN`) 后缀的函数，根据 MemOp 中的端序位选择。

---

## 20. 脏页追踪集成

### TLB_NOTDIRTY 机制

1. **标记**：`tlb_set_page_full()` 中，如果 RAM 页是干净的：
   ```c
   if (physical_memory_is_clean(addr)) {
       read_flags |= TLB_NOTDIRTY;  // cputlb.c:1084-1089
   }
   ```

2. **触发**：写入标记为 `TLB_NOTDIRTY` 的页面时：
   - 快路径比较失败（`addr_write` 含 `TLB_NOTDIRTY` 位）
   - 进入慢路径
   - `mmu_watch_or_dirty()` 调用 `notdirty_write()`

### notdirty_write()

**定义**：`accel/tcg/cputlb.c:1336-1357`

```
notdirty_write(cpu, addr, size, full, ra)
│
├── 失效该地址的 TB（自修改代码检查）
│
├── 在所有脏页位图中标记该页为脏
│   memory_region_set_dirty()
│
├── 如果页面不再需要追踪
│   → 清除 TLB_NOTDIRTY（后续写入走快路径）
│
└── 设置 tlb dirty 位图
    tlb_set_dirty()                               :943-972
```

**用途**：迁移脏页追踪、VGA 显存变更检测、KVM 脏页同步。

---

## 21. Watchpoint 处理

### TLB 集成

Watchpoint 通过 TLB 标志实现零开销监控：

1. 设置 watchpoint 时：
   - 对相关页面的 TLB 条目设置 `TLB_WATCHPOINT`（慢路径标志）
   - 同时设置 `TLB_FORCE_SLOW`（快路径标志）
   - 强制访问走慢路径

2. 慢路径中：
   - `mmu_watch_or_dirty()` 检查 `TLB_WATCHPOINT`
   - 调用 `cpu_check_watchpoint()`（`cputlb.c:1703-1706`）
   - 命中 → longjmp 到异常处理
   - 未命中 → 清除标志，继续正常访问

### probe_access() 中的检查

**定义**：`accel/tcg/cputlb.c:1489-1511`

`probe_access()` 用于预检查访问是否会触发 watchpoint/fault，不实际执行内存操作。

---

## 22. TLB 刷新操作

### 全局刷新

**定义**：`accel/tcg/cputlb.c:418-420`

```c
void tlb_flush(CPUState *cpu)
{
    tlb_flush_by_mmuidx(cpu, ALL_MMUIDX_BITS);
}
```

### 按 MMU 索引刷新

**定义**：`accel/tcg/cputlb.c:369-416`

```
tlb_flush_by_mmuidx(cpu, idxmap)
│
├── 如果是本 CPU → tlb_flush_by_mmuidx_async_work()
│   ├── 清空指定 mmu_idx 的 TLB 条目
│   ├── 重置 n_used_entries
│   ├── 更新统计计数器
│   └── 清空 jump cache
│
└── 如果是远程 CPU → async_run_on_cpu()
    → 投递到目标 CPU 的安全工作队列
```

### 单页刷新

**定义**：`accel/tcg/cputlb.c:502-615`

```
tlb_flush_page_by_mmuidx(cpu, addr, idxmap)
│
├── 检查是否与大页重叠                              :510-520
│   重叠 → 退化为全刷新
│
├── 对每个 mmu_idx:                                 :530-600
│   ├── 清除主 TLB 中该页的条目
│   └── 清除 victim TLB 中该页的条目
│
└── 清除 jump cache 中相关条目
```

### 跨 CPU 同步刷新

```
tlb_flush_all_cpus_synced(cpu)                      :423-436
│
└── async_safe_run_on_cpu()
    → 在 exclusive context 中执行
    → 所有 vCPU 暂停后统一刷新
```

---

## 23. ARM64 页表遍历 — arm_cpu_tlb_fill_align()

**定义**：`target/arm/tcg/tlb_helper.c:331-378`

```
arm_cpu_tlb_fill_align(cpu, full, addr, type, mmu_idx, memop, size, probe, ra)
│
├── get_phys_addr(env, addr, type, mmu_idx, &result)  :364-366
│   → ARM64 多级页表遍历
│   → 返回物理地址 + 属性 + 保护位
│
├── [成功] 填充 CPUTLBEntryFull                       :369-370
│   full->phys_addr = result.phys_addr
│   full->attrs = result.attrs
│   full->prot = result.prot
│   full->lg_page_size = result.page_size
│   return true
│
└── [失败] 触发 fault                                  :376-378
    cpu_restore_state(cpu, ra)
    arm_deliver_fault(cpu, addr, type, mmu_idx, &result.fi)
    → 注入 Data Abort / Instruction Abort 异常
```

---

## 24. 地址翻译完整链路

```
Guest VA
    │
    ▼
┌─────────────────────┐
│ 内联 TLB 快路径     │ ← TB 生成代码中
│ (8 条宿主指令)      │
│                     │
│ LDP mask, table     │
│ AND index           │
│ ADD entry           │
│ LDP comp, addend    │
│ CMP → 命中?         │
└───┬──────────┬──────┘
    │命中       │未命中
    ▼           ▼
host_addr    慢路径 helper
= va+addend  │
    │        ├── mmu_lookup1()
    │        │   ├── victim TLB 搜索
    │        │   └── tlb_fill_align()
    │        │       └── arm_cpu_tlb_fill_align()
    │        │           └── get_phys_addr()
    │        │               └── 多级页表遍历
    │        │                   └── tlb_set_page_full()
    │        │                       └── address_space_translate_for_iotlb()
    │        │                           └── FlatView → MemoryRegionSection
    │        │
    │        ├── mmu_watch_or_dirty()
    │        │   ├── watchpoint → cpu_check_watchpoint()
    │        │   └── notdirty → notdirty_write()
    │        │
    │        └── 访问类型分发
    │            ├── RAM → 直接加载/存储
    │            └── MMIO → io_prepare()
    │                     → memory_region_dispatch_{read,write}()
    │                     → mr->ops->read/write()
    │                     → 设备回调
    │
    ▼
 内存操作完成
```

---

## 25. io_prepare() 与 MemoryRegion 分发

### address_space_translate_for_iotlb()

**定义**：`system/physmem.c:685-750`

在 `tlb_set_page_full()` 中调用，将物理地址映射到 `MemoryRegionSection`：

```
address_space_translate_for_iotlb(cpu, asidx, paddr, &xlat, &sz, attrs, &prot)
│
├── 获取 CPU 的 AddressSpace                        :695
│   as = cpu->cpu_ases[asidx].as
│
├── FlatView 查找                                    :700-710
│   section = address_space_translate_internal(
│       address_space_to_dispatch(as), paddr, &xlat, &sz, ...)
│
├── [IOMMU] 递归翻译                                :715-740
│   如果是 IOMMU 区域 → 再次翻译
│
└── 返回 section + xlat
```

### memory_region_dispatch_read/write()

**定义**：`system/memory.c:1444-1560`

```
memory_region_dispatch_read(mr, offset, &val, size, attrs)
│
├── 调整对齐和大小                                   :1452-1464
│
├── 调用设备回调
│   mr->ops->read_with_attrs(mr, offset, &val, size, attrs)
│   或 mr->ops->read(mr, offset, size)
│
└── 返回 MemTxResult
```

---

## 26. 完整 qemu_ld 执行示例

### 场景：ARM64 客户机 `LDR X0, [X1]`

```
1. 前端翻译 → tcg_gen_qemu_ld_i64(tmp, addr, mmu_idx, MO_64)

2. 后端发射（AArch64 宿主）：
   
   快路径内联代码：
   LDP  X16, X17, [X19, #tlb_f_offset]    ; 加载 mask + table
   AND  X16, X1, X16, LSR #7               ; index = (addr>>12) & mask>>5
   ADD  X16, X17, X16                       ; entry = table + index
   LDP  X17, X30, [X16, #0]                ; 加载 addr_read + addend
   AND  X16, X1, #PAGE_MASK                 ; 对齐的客户机地址
   CMP  X16, X17                            ; 比较
   B.NE .Lslow                              ; 不匹配 → 慢路径

   ; 快路径命中
   LDR  X0, [X1, X30]                       ; 直接加载：host = addr + addend

   ; 慢路径（out-of-line）
   .Lslow:
   MOV  X0, X19                             ; env
   MOV  X1, addr                            ; 客户机地址
   MOV  X2, #oi                             ; MemOpIdx
   ADR  X3, .Lreturn                        ; 返回地址
   BL   helper_ldq_mmu                      ; 调用慢路径 helper
   MOV  X0, X0                              ; 结果
   B    .Lcontinue                           ; 返回

3. 慢路径内部：
   helper_ldq_mmu()
   → mmu_lookup1() → TLB 未命中 → victim 搜索
   → tlb_fill_align() → arm_cpu_tlb_fill_align()
   → get_phys_addr() → 四级页表遍历
   → tlb_set_page_full() → 填充 TLB 条目
   → 重新查找 TLB → 命中
   → RAM: 返回 *(host_addr)
   → MMIO: io_prepare() → memory_region_dispatch_read()
```

---

## 27. 附录 A：关键源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `accel/tcg/cputlb.c` | ~2800 | TLB 管理、内存访问快/慢路径、MMIO 分发 |
| `accel/tcg/ldst_common.c.inc` | ~120 | helper_ld/st_mmu 包装 |
| `include/exec/tlb-common.h` | 56 | CPUTLBEntry、CPUTLBDescFast |
| `include/exec/tlb-flags.h` | 85 | TLB 标志位定义 |
| `include/exec/cputlb.h` | ~100 | TLB API（tlb_set_page_full 等） |
| `include/hw/core/cpu.h` | ~500 | CPUTLBEntryFull、CPUTLBDesc、CPUTLB、CPUNegativeOffsetState |
| `include/tcg/tcg-ldst.h` | ~65 | helper_ld/st_mmu 声明 |
| `tcg/aarch64/tcg-target.c.inc` | 3592 | prepare_host_addr 快路径、慢路径发射 |
| `target/arm/tcg/tlb_helper.c` | ~380 | ARM64 TLB 填充、get_phys_addr 调用 |
| `system/physmem.c` | ~800 | address_space_translate_for_iotlb |
| `system/memory.c` | ~1600 | memory_region_dispatch_read/write |

---

> **文档版本**：v1.0
> **源码版本**：QEMU 11.0.50 (commit 基于 2025-07 主线)
> **分析工具**：zoekt + ctags + cscope + clangd + 手动源码验证
> **交叉验证**：所有行号均经 view/grep 验证
