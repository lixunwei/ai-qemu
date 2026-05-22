# TCG 深入分析 — 优化遍、向量指令翻译与 Softmmu TLB 机制

## 1. 概述

本文是 TCG 后端系列的第二篇，深入分析三个核心子系统：**IR 优化遍**（常量折叠、拷贝传播、死代码消除）、**向量指令翻译**（NEON/SVE → TCG 向量 IR → 宿主 SIMD）、**Softmmu TLB**（快速路径/慢速路径地址翻译）。这三个子系统共同决定了 TCG 模拟的性能上限。

**关键源文件：**
- `tcg/optimize.c` — IR 优化遍（~3100行）
- `tcg/tcg-op-vec.c` — 向量 IR 生成 API
- `tcg/tcg-op-gvec.c` — 通用向量操作（GVec）
- `include/tcg/tcg-opc.h` — 向量操作码定义
- `tcg/aarch64/tcg-target.c.inc` — AArch64 后端向量发射 & TLB 快速路径
- `accel/tcg/cputlb.c` — Softmmu TLB 核心实现
- `include/exec/tlb-common.h` — TLB 条目结构
- `include/exec/tlb-flags.h` — TLB 标志位
- `include/hw/core/cpu.h` — CPUTLBEntryFull/CPUTLBDesc/VTLB

---

## 2. 优化遍总体架构

### 2.1 OptContext — 优化上下文

```c
// tcg/optimize.c:50-61
typedef struct OptContext {
    TCGContext *tcg;       // 编译上下文
    TCGOp *prev_mb;       // 上一个内存屏障
    TCGTempSet temps_used; // 已使用的临时变量集
    IntervalTreeRoot mem_copy;  // 内存拷贝区间树
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_free;
    TCGType type;          // 当前操作类型
    int carry_state;       // 进位状态: -1=非常量, {0,1}=常量进位
} OptContext;
```

### 2.2 TempOptInfo — 每临时变量优化信息

```c
// tcg/optimize.c:41-48
typedef struct TempOptInfo {
    TCGTemp *prev_copy;    // 拷贝链: 前一个副本
    TCGTemp *next_copy;    // 拷贝链: 下一个副本（环形双向链表）
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_copy;
    uint64_t z_mask;       // 零掩码: bit=0 → 值必为 0
    uint64_t o_mask;       // 一掩码: bit=1 → 值必为 1
    uint64_t s_mask;       // 符号掩码: bit=1 → 值等于 MSB
} TempOptInfo;

// z_mask/o_mask/s_mask 三元组提供比特级精度的值范围追踪:
//   z_mask & o_mask → 已知位 (z=0,o=1 → 该位确定)
//   ~(z_mask ^ o_mask) → 未知位（两者相同 → 该位不确定）
//   s_mask → 符号扩展信息
```

---

## 3. 优化遍主循环

```c
// tcg/optimize.c:3000-3095
void tcg_optimize(TCGContext *s)
{
    OptContext ctx = { .tcg = s };
    
    // 初始化: 清空所有临时变量的优化状态
    for (i = 0; i < nb_temps; ++i)
        s->temps[i].state_ptr = NULL;
    
    // 遍历每条 IR 指令
    QTAILQ_FOREACH_SAFE(op, &s->ops, link, op_next) {
        // 1. call 指令特殊处理
        if (opc == INDEX_op_call) { fold_call(&ctx, op); continue; }
        
        // 2. 初始化操作数信息
        init_arguments(&ctx, op, def->nb_oargs + def->nb_iargs);
        
        // 3. 拷贝传播: 将操作数替换为其拷贝源
        copy_propagate(&ctx, op, def->nb_oargs, def->nb_iargs);
        
        // 4. 按操作码分发到 fold_* 函数（字母序排列）
        switch (opc) {
            case INDEX_op_add:     done = fold_add(&ctx, op);     break;
            case INDEX_op_and:     done = fold_and(&ctx, op);     break;
            case INDEX_op_brcond:  done = fold_brcond(&ctx, op);  break;
            case INDEX_op_sub:     done = fold_sub(&ctx, op);     break;
            // ... 80+ 操作码各有专用折叠函数
        }
    }
}
```

---

## 4. 核心优化策略

### 4.1 常量折叠

```c
// tcg/optimize.c:1132-1138
static bool fold_add(OptContext *ctx, TCGOp *op)
{
    // fold_const2_commutative: 若两个操作数都是常量 → 直接计算结果
    //   add(3, 5) → movi(8)
    // fold_xi_to_x: 若第二个操作数为 0 → 简化为 mov
    //   add(x, 0) → mov(x)
    if (fold_const2_commutative(ctx, op) ||
        fold_xi_to_x(ctx, op, 0)) {
        return true;
    }
    return finish_folding(ctx, op);
}

// 类似模式:
//   fold_sub: sub(x, 0) → mov(x), sub(x, x) → movi(0)
//   fold_and: and(x, 0) → movi(0), and(x, -1) → mov(x)
//   fold_or:  or(x, 0) → mov(x), or(x, -1) → movi(-1)
//   fold_mul: mul(x, 0) → movi(0), mul(x, 1) → mov(x)
```

### 4.2 拷贝传播

```c
// 拷贝链机制:
// 当 mov(a, b) 时，a 和 b 通过 prev_copy/next_copy 形成环形链表
// 后续使用 a 的地方，copy_propagate() 可替换为 b（或链中更优者）

// tcg/optimize.c:115-118
static inline bool ts_is_copy(TCGTemp *ts)
{
    return ts_info(ts)->next_copy != ts;
    // 环形链表: 只有自身 → 非拷贝; 有其他节点 → 是某个值的拷贝
}

// cmp_better_copy(): 优先使用 kind 更低的临时变量
//   CONST < GLOBAL < TB < EBB < LOCAL < NORMAL
//   → 常量和全局变量优先级最高
```

### 4.3 tcg_opt_gen_mov — 操作替换

```c
// tcg/optimize.c:357-417
static bool tcg_opt_gen_mov(OptContext *ctx, TCGOp *op, TCGArg dst, TCGArg src)
{
    // 1. 如果 dst 和 src 已是拷贝 → 直接删除 op
    if (ts_are_copies(dst_ts, src_ts)) {
        tcg_op_remove(ctx->tcg, op);  // 死代码消除
        return true;
    }
    
    // 2. 将操作改写为 mov 或 mov_vec
    op->opc = INDEX_op_mov;  // 或 INDEX_op_mov_vec
    op->args[0] = dst;
    op->args[1] = src;
    
    // 3. 传播掩码信息
    di->z_mask = si->z_mask;
    di->o_mask = si->o_mask;
    di->s_mask = si->s_mask;
    
    // 4. 将 dst 加入 src 的拷贝链
    di->next_copy = si->next_copy;
    di->prev_copy = src_ts;
    // ... 环形链表插入
}
```

### 4.4 分支条件折叠

```c
// tcg/optimize.c:1464-1479
static bool fold_brcond(OptContext *ctx, TCGOp *op)
{
    // do_constant_folding_cond1: 尝试常量求值分支条件
    int i = do_constant_folding_cond1(ctx, op, ...);
    
    if (i == 0) {
        // 条件永远为假 → 删除分支（死代码）
        tcg_op_remove(ctx->tcg, op);
        return true;
    }
    if (i > 0) {
        // 条件永远为真 → 改写为无条件跳转
        op->opc = INDEX_op_br;
        op->args[0] = op->args[3];  // 跳转目标
        finish_ebb(ctx);
    } else {
        // 无法确定 → 保留原始条件分支
        finish_bb(ctx);
    }
    return true;
}
```

---

## 5. 向量指令翻译

### 5.1 向量操作码

```c
// include/tcg/tcg-opc.h:126-179 — 向量操作码定义
// 所有向量操作带 TCG_OPF_VECTOR 标志

// 数据移动:
DEF(mov_vec, 1, 1, 0)     // 向量移动
DEF(dup_vec, 1, 1, 0)     // 标量→向量广播
DEF(dupm_vec, 1, 1, 1)    // 内存标量→向量广播
DEF(ld_vec, 1, 1, 1)      // 向量加载
DEF(st_vec, 0, 2, 1)      // 向量存储

// 算术:
DEF(add_vec, 1, 2, 0)     // 向量加
DEF(sub_vec, 1, 2, 0)     // 向量减
DEF(mul_vec, 1, 2, 0)     // 向量乘
DEF(neg_vec, 1, 1, 0)     // 向量取反
DEF(abs_vec, 1, 1, 0)     // 向量绝对值
DEF(ssadd_vec/usadd_vec)  // 饱和加
DEF(sssub_vec/ussub_vec)  // 饱和减
DEF(smin_vec/smax_vec)    // 有符号最小/最大
DEF(umin_vec/umax_vec)    // 无符号最小/最大

// 逻辑:
DEF(and_vec/or_vec/xor_vec/not_vec)  // 按位运算
DEF(andc_vec/orc_vec/nand_vec/nor_vec/eqv_vec)

// 移位（立即数/标量/向量）:
DEF(shli_vec/shri_vec/sari_vec/rotli_vec)   // 立即数
DEF(shls_vec/shrs_vec/sars_vec/rotls_vec)   // 标量
DEF(shlv_vec/shrv_vec/sarv_vec/rotlv_vec/rotrv_vec) // 向量

// 比较与选择:
DEF(cmp_vec, 1, 2, 1)      // 逐元素比较
DEF(bitsel_vec, 1, 3, 0)   // 位选择 (BSL)
DEF(cmpsel_vec, 1, 4, 1)   // 比较选择
```

### 5.2 向量类型

```c
// include/tcg/tcg.h:128-138
typedef enum TCGType {
    TCG_TYPE_I32,     // 32 位整数
    TCG_TYPE_I64,     // 64 位整数
    TCG_TYPE_I128,    // 128 位整数
    TCG_TYPE_V64  = 32, // 64 位向量 (NEON D 寄存器)
    TCG_TYPE_V128 = 33, // 128 位向量 (NEON Q 寄存器)
    TCG_TYPE_V256 = 34, // 256 位向量 (AVX)
} TCGType;

// VECE (Vector Element Count Exponent):
//   MO_8=0, MO_16=1, MO_32=2, MO_64=3
//   元素大小 = 1 << VECE 字节
```

### 5.3 向量 IR 生成 API

```c
// tcg/tcg-op-vec.c — 底层向量操作

// tcg_gen_mov_vec() (214): 向量移动
void tcg_gen_mov_vec(TCGv_vec r, TCGv_vec a) {
    if (r != a)
        vec_gen_op2(INDEX_op_mov_vec, 0, r, a);
}

// tcg_gen_dup_i64_vec() (227): 标量复制到向量所有元素
// tcg_gen_dup_mem_vec() (247): 从内存复制标量到向量

// tcg/tcg-op-gvec.c — 高级通用向量操作（GVec）
// tcg_gen_gvec_add() — 向量加法，自动选择最优实现:
//   1. 操作数 ≤ 宿主向量宽度 → 直接向量 IR
//   2. 操作数 > 宿主向量宽度 → 展开为多条向量 IR
//   3. 宿主无向量支持 → 展开为标量循环
//   4. 复杂操作 → 调用 helper 函数 (out-of-line, OOL)

// GVec OOL 接口:
// tcg_gen_gvec_2_ool() (156): 2 操作数, 调用 helper
// tcg_gen_gvec_3_ool() (209): 3 操作数, 调用 helper
// tcg_gen_gvec_3_ptr() (295): 3 操作数 + 指针 (用于 SVE)
```

### 5.4 ARM64 前端向量翻译

```c
// NEON (固定 128 位): target/arm/tcg/translate-neon.c
//   使用 tcg_gen_gvec_* 接口, 操作大小固定为 128 位
//   例: ADD V0.4S, V1.4S, V2.4S
//     → tcg_gen_gvec_add(MO_32, ofs_d, ofs_n, ofs_m, 16, 16)

// SVE (可变长度): target/arm/tcg/translate-sve.c
//   使用 tcg_gen_gvec_*_ool 和 gen_gvec_ool_zzz 等
//   操作大小由 VL (向量长度) 决定, 最大 2048 位
//   例: ADD Z0.S, Z1.S, Z2.S
//     → gen_gvec_fn_zzz(s, gen_helper_gvec_add32, rd, rn, rm)
//   复杂 SVE2 操作通过 OOL helper 实现

// GVec 自动展开策略:
//   VL=128, 宿主有 128 位 SIMD → 1 条向量指令
//   VL=256, 宿主有 128 位 SIMD → 2 条向量指令
//   VL=512, 宿主有 128 位 SIMD → 4 条向量指令
//   VL=2048 → 16 条或调用 OOL helper
```

### 5.5 AArch64 后端向量发射

```c
// tcg/aarch64/tcg-target.c.inc:3289+
void tcg_expand_vec_op(TCGOpcode opc, TCGType type, unsigned vece,
                        TCGArg a0, ...)
{
    // 将不能直接映射到宿主指令的向量操作展开:
    
    // rotli_vec → shri + SLI (右移 + 左移插入)
    // shrv_vec/sarv_vec → neg + shlv/sshl (AArch64 右移 = 负左移)
    // rotlv_vec → sub + shlv + shlv + or (两个方向移位再合并)
    // rotrv_vec → 类似 rotlv 但方向相反
}

// 直接支持的向量操作 → 一对一映射到 AArch64 SIMD 指令:
//   add_vec → ADD Vd.T, Vn.T, Vm.T
//   sub_vec → SUB
//   and_vec → AND
//   shlv_vec → USHL/SSHL
//   cmp_vec → CMEQ/CMGT/CMGE/CMHI/CMHS
//   bitsel_vec → BSL (Bit Select)
```

---

## 6. Softmmu TLB 架构

### 6.1 TLB 数据结构层次

```
CPUState
  └── neg.tlb (CPUTLB)
       ├── f[NB_MMU_MODES] (CPUTLBDescFast)  ← TCG 快速路径访问
       │    ├── mask     (条目数 - 1) << ENTRY_BITS
       │    └── *table   (CPUTLBEntry 数组)
       │         ├── addr_read    虚拟地址（读）
       │         ├── addr_write   虚拟地址（写）
       │         ├── addr_code    虚拟地址（取指）
       │         └── addend       虚拟→宿主地址偏移
       │
       ├── d[NB_MMU_MODES] (CPUTLBDesc)       ← 慢速路径扩展
       │    ├── large_page_addr/mask  大页追踪
       │    ├── vtable[8]    (CPUTLBEntry)     ← Victim TLB
       │    ├── vfulltlb[8]  (CPUTLBEntryFull) ← Victim 完整信息
       │    └── *fulltlb     (CPUTLBEntryFull) ← 主 TLB 完整信息
       │
       └── c (CPUTLBCommon)
            ├── lock         TLB 操作锁
            ├── generation   TLB 代次（刷新计数）
            └── dirty/clean  统计信息
```

### 6.2 CPUTLBEntry — 快速路径条目

```c
// include/exec/tlb-common.h:25-41
typedef union CPUTLBEntry {
    struct {
        uintptr_t addr_read;   // 虚拟地址（读匹配用）
        uintptr_t addr_write;  // 虚拟地址（写匹配用）
        uintptr_t addr_code;   // 虚拟地址（取指匹配用）
        uintptr_t addend;      // 地址转换偏移: host_addr = guest_va + addend
    };
} CPUTLBEntry;
// 大小: 32 字节 (64位系统), 对齐到 2^CPU_TLB_ENTRY_BITS

// CPUTLBDescFast: 快速路径最小结构 (16 字节, 对齐 16)
//   mask: (n_entries-1) << CPU_TLB_ENTRY_BITS
//   table: CPUTLBEntry* 数组指针
```

### 6.3 CPUTLBEntryFull — 完整条目

```c
// include/hw/core/cpu.h:214-271
struct CPUTLBEntryFull {
    hwaddr xlat_offset;     // 物理地址偏移
    MemoryRegionSection *section; // 内存区域
    hwaddr phys_addr;       // 物理地址
    MemTxAttrs attrs;       // 内存事务属性
    uint8_t prot;           // 保护位 (PAGE_READ|WRITE|EXEC)
    uint8_t lg_page_size;   // log2(页面大小)
    uint8_t tlb_fill_flags; // 填充标志
    uint8_t slow_flags[MMU_ACCESS_COUNT]; // 慢速路径标志
    union {
        struct { uint8_t pte_attrs; uint8_t shareability; bool guarded; } arm;
    } extra;                // 目标特定字段 (ARM: PTE 属性/共享性/Guarded)
};
```

---

## 7. TLB 标志位

```c
// include/exec/tlb-flags.h

// === CPUTLBEntryFull.slow_flags 中的标志 (低位) ===
TLB_BSWAP          (1 << 0)  // 需要字节交换（跨端序访问）
TLB_WATCHPOINT     (1 << 1)  // 地址有 Watchpoint
TLB_CHECK_ALIGNED  (1 << 2)  // 需要对齐检查
TLB_DISCARD_WRITE  (1 << 3)  // 写入被丢弃（只读 RAM 段）
TLB_MMIO           (1 << 4)  // I/O 内存 → 必须走 MMIO 路径

// === CPUTLBEntry.addr_* 中嵌入的标志 (高位, [9:6]) ===
TLB_INVALID_MASK   (1 << 6)  // TLB 条目无效
TLB_NOTDIRTY       (1 << 7)  // 干净页面 → 写入需标记脏
TLB_FORCE_SLOW     (1 << 8)  // 强制慢速路径

// 快速路径检查掩码:
TLB_FLAGS_MASK = TLB_INVALID_MASK | TLB_NOTDIRTY | TLB_FORCE_SLOW
// 快速路径只检查这 3 位 → 非零则走慢速路径
```

---

## 8. TLB 快速路径（TCG 生成代码内联）

### 8.1 AArch64 后端 TLB 查找

```c
// tcg/aarch64/tcg-target.c.inc:1677-1720
// Guest 内存访问（qemu_ld/st）的 TLB 查找序列:

// 步骤 1: 加载 CPUTLBDescFast.{mask, table}（一条 LDP 指令）
LDP TMP0, TMP1, [AREG0, #tlb_offset]
// TMP0 = mask, TMP1 = table 指针

// 步骤 2: 从 Guest 地址提取 TLB 索引
AND_LSR TMP0, TMP0, addr_reg, #(PAGE_BITS - ENTRY_BITS)
// TMP0 = (addr >> PAGE_BITS) & mask（已左移 ENTRY_BITS）

// 步骤 3: 计算 CPUTLBEntry 地址
ADD TMP1, TMP1, TMP0  // TMP1 = &table[index]

// 步骤 4: 加载比较器和 addend
LDR TMP0, [TMP1, #offsetof(addr_read)]  // 或 addr_write
LDR TMP1, [TMP1, #offsetof(addend)]

// 步骤 5: 页面对齐比较
AND TMP2, addr_reg, #(PAGE_MASK | alignment_mask)
CMP TMP0, TMP2
B.NE slow_path  // 不匹配 → 慢速路径

// 步骤 6: 快速路径命中 → 直接访问
ADD host_addr, addr_reg, TMP1  // host = guest + addend
LDR result, [host_addr]        // 直接内存访问
```

### 8.2 快速路径条件

```
快速路径命中条件（全部满足）:
  1. TLB 索引处的 addr_read/write 页号匹配 Guest 地址页号
  2. 没有任何 TLB_FLAGS_MASK 标志位
  3. 访问不跨页边界
  
快速路径开销: ~6 条宿主指令（2 load + 2 算术 + 1 比较 + 1 跳转）
→ 命中时几乎零开销的地址翻译
```

---

## 9. TLB 慢速路径

### 9.1 tlb_set_page_full() — TLB 填充

```c
// accel/tcg/cputlb.c:1024-1110
void tlb_set_page_full(CPUState *cpu, int mmu_idx,
                        vaddr addr, CPUTLBEntryFull *full)
{
    // 1. 页面大小处理
    if (full->lg_page_size > TARGET_PAGE_BITS)
        tlb_add_large_page(cpu, mmu_idx, addr, sz);
    
    // 2. 物理地址翻译 → MemoryRegionSection
    section = address_space_translate_for_iotlb(cpu, asidx, paddr_page, ...);
    
    // 3. RAM 与 I/O 路径区分
    if (is_ram || is_romd) {
        addend = memory_region_get_ram_ptr(section->mr) + xlat;
        // RAM: host_addr = guest_va + addend
        
        if (physical_memory_is_clean(iotlb))
            write_flags |= TLB_NOTDIRTY;  // 干净页面标记
    } else {
        addend = 0;  // I/O 没有宿主内存映射
        write_flags |= TLB_MMIO;  // 标记为 MMIO
    }
    
    // 4. Watchpoint 检查
    wp_flags = cpu_watchpoint_address_matches(cpu, addr_page, TARGET_PAGE_SIZE);
    
    // 5. 写入 TLB 条目（替换旧条目）
    index = tlb_index(cpu, mmu_idx, addr_page);
    // 旧条目 → 移入 victim TLB
    // 新条目 → 写入主 TLB
}
```

### 9.2 Victim TLB 机制

```c
// include/hw/core/cpu.h:207
#define CPU_VTLB_SIZE 8  // 8 条 victim TLB 条目

// accel/tcg/cputlb.c:1306-1333
static bool victim_tlb_hit(CPUState *cpu, size_t mmu_idx, size_t index,
                            MMUAccessType access_type, vaddr page)
{
    // 线性搜索 8 条 victim 条目
    for (vidx = 0; vidx < CPU_VTLB_SIZE; ++vidx) {
        CPUTLBEntry *vtlb = &cpu->neg.tlb.d[mmu_idx].vtable[vidx];
        uint64_t cmp = tlb_read_idx(vtlb, access_type);
        
        if (cmp == page) {
            // 命中! 交换 victim 条目和主 TLB 条目
            // 原子操作: 在锁保护下交换
            qemu_spin_lock(&cpu->neg.tlb.c.lock);
            copy_tlb_helper_locked(&tmptlb, tlb);    // 主→临时
            copy_tlb_helper_locked(tlb, vtlb);        // victim→主
            copy_tlb_helper_locked(vtlb, &tmptlb);   // 临时→victim
            qemu_spin_unlock(&cpu->neg.tlb.c.lock);
            
            // 同步交换 CPUTLBEntryFull
            tmpf = *f1; *f1 = *f2; *f2 = tmpf;
            return true;
        }
    }
    return false;
}

// Victim TLB 设计:
//   主 TLB 直接映射 (1路) → 冲突驱逐频繁
//   Victim TLB 8 路全关联 → 捕获被驱逐的热条目
//   命中后交换回主 TLB → 下次快速路径直接命中
```

---

## 10. TLB 刷新

```c
// accel/tcg/cputlb.c:418-420
void tlb_flush(CPUState *cpu)
{
    tlb_flush_by_mmuidx(cpu, ALL_MMUIDX_BITS);
    // 刷新所有 MMU 模式的所有 TLB 条目
}

// tlb_flush_page() (612-614): 仅刷新特定页面
// tlb_flush_page_locked() (502-518): 锁定版本
//   1. 检查 large_page 区域 → 可能触发全 TLB 刷新
//   2. 定位 TLB 索引
//   3. 清除主 TLB 条目 (addr_* = -1)
//   4. 清除 victim TLB 中匹配条目

// 刷新触发时机:
//   TLBI 指令 (ARM)      → tlb_flush / tlb_flush_page
//   页表修改              → tlb_flush_page
//   ASID 切换             → tlb_flush_by_mmuidx
//   MMU 开启/关闭         → tlb_flush
//   写保护页面被写入      → 单页刷新
```

---

## 11. 完整内存访问流程

```
Guest 执行 LDR X0, [X1]
  │
  ▼ TCG 生成代码
tcg_gen_qemu_ld_i64(dst, addr, memop)
  │
  ▼ 后端发射 (AArch64)
[1] LDP mask, table, [env, #tlb_offset]     ← 加载 TLB 描述符
[2] AND_LSR idx, mask, addr, #shift          ← 计算索引
[3] ADD entry, table, idx                     ← 条目地址
[4] LDR cmp, [entry, #addr_read]            ← 加载比较器
[5] LDR addend, [entry, #addend]            ← 加载偏移
[6] AND page, addr, #PAGE_MASK              ← 提取页号
[7] CMP cmp, page                           ← 比较
[8] B.NE slow_path                           ← 不匹配→慢速
[9] ADD host, addr, addend                   ← 快速路径
[10] LDR result, [host]                      ← 宿主内存读取
  │
  ├── 命中 → [9,10] 直接访问（~2 额外指令）
  │
  └── 未命中 → slow_path:
       │
       ├── victim_tlb_hit() → 交换到主 TLB → 重试
       │
       └── tlb_fill() → 页表遍历 → tlb_set_page_full()
            │
            ├── RAM → addend = host_ptr + xlat
            │         → 填充 TLB → 重试快速路径
            │
            └── MMIO → TLB_MMIO 标志
                      → io_readx() → MemoryRegion.ops->read
```

---

## 12. 小结

| 组件 | 实现 |
|------|------|
| **优化遍** | tcg_optimize() 遍历 IR，80+ fold_* 函数按操作码分发 |
| **常量折叠** | fold_const2 计算常量结果，fold_xi_to_x 简化恒等操作 |
| **拷贝传播** | TempOptInfo.prev/next_copy 环形链表追踪拷贝关系 |
| **掩码追踪** | z_mask/o_mask/s_mask 三元组提供比特级值范围分析 |
| **死代码消除** | tcg_opt_gen_mov 检测已有拷贝→tcg_op_remove 删除冗余 |
| **向量操作码** | 55+ 向量 DEF（算术/逻辑/移位/比较），V64/V128/V256 类型 |
| **GVec API** | tcg_gen_gvec_* 自动选择向量 IR / 标量展开 / OOL helper |
| **向量展开** | tcg_expand_vec_op 将复杂向量操作分解为基础操作序列 |
| **TLB 快速路径** | ~6 条内联指令：LDP→索引→查找→比较→addend→访问 |
| **TLB 慢速路径** | victim TLB(8路)→页表遍历→tlb_set_page_full 填充 |
| **TLB 标志** | INVALID/NOTDIRTY/FORCE_SLOW(快速)/MMIO/WATCHPOINT(慢速) |
| **TLB 刷新** | 全量/按页/按 ASID 三种粒度，ARM TLBI 指令触发 |
