# ARM Cache 管理与维护操作深度分析

> 基于 QEMU 11.0.50 源码，深度分析 ARM Cache 管理在 QEMU 中的实现：
> Cache ID 寄存器、CCSIDR/CLIDR/CTR 配置、Cache 维护操作（DC/IC 指令）、
> DC ZVA 内存清零、自修改代码检测、PoC/PoU 概念、CPU 模型缓存参数、
> AArch32 CP15 缓存操作。

---

## 目录

1. [QEMU 缓存模型概述](#1-qemu-缓存模型概述)
2. [Cache ID 寄存器体系](#2-cache-id-寄存器体系)
3. [CCSIDR 寄存器与 make_ccsidr()](#3-ccsidr-寄存器与-make_ccsidr)
4. [ccsidr_read() 实现](#4-ccsidr_read-实现)
5. [CSSELR 缓存级别选择](#5-csselr-缓存级别选择)
6. [CLIDR 缓存级别标识](#6-clidr-缓存级别标识)
7. [CTR_EL0 缓存类型寄存器](#7-ctr_el0-缓存类型寄存器)
8. [DCZID_EL0 与 DC ZVA 块大小](#8-dczid_el0-与-dc-zva-块大小)
9. [ARMCPU 缓存参数存储](#9-armcpu-缓存参数存储)
10. [CPU 模型缓存配置示例](#10-cpu-模型缓存配置示例)
11. [arm_cpu_realizefn() 缓存初始化](#11-arm_cpu_realizefn-缓存初始化)
12. [AArch64 Cache 维护操作注册](#12-aarch64-cache-维护操作注册)
13. [指令缓存操作 (IC)](#13-指令缓存操作-ic)
14. [数据缓存操作 (DC)](#14-数据缓存操作-dc)
15. [DC ZVA 实现：真正的内存清零](#15-dc-zva-实现真正的内存清零)
16. [HELPER(dc_zva) 详细分析](#16-helperdc_zva-详细分析)
17. [PoC/PoU 访问控制](#17-pocpou-访问控制)
18. [aa64_cacheop_poc_access() PoC 陷阱](#18-aa64_cacheop_poc_access-poc-陷阱)
19. [do_cacheop_pou_access() PoU 陷阱](#19-do_cacheop_pou_access-pou-陷阱)
20. [自修改代码与 IC IVAU](#20-自修改代码与-ic-ivau)
21. [ic_ivau_write() TB 失效](#21-ic_ivau_write-tb-失效)
22. [AArch32 CP15 缓存操作](#22-aarch32-cp15-缓存操作)
23. [分支预测操作 (BP)](#23-分支预测操作-bp)
24. [HCR_EL2 缓存陷阱控制](#24-hcr_el2-缓存陷阱控制)
25. [SCTLR 缓存控制位](#25-sctlr-缓存控制位)
26. [QEMU TLB 与缓存操作的交互](#26-qemu-tlb-与缓存操作的交互)
27. [内存类型属性处理](#27-内存类型属性处理)
28. [关键函数总览表](#28-关键函数总览表)
29. [Cache 操作指令完整映射表](#29-cache-操作指令完整映射表)
30. [QEMU 缓存模拟策略总结](#30-qemu-缓存模拟策略总结)

---

## 1. QEMU 缓存模型概述

QEMU **不模拟真实缓存硬件**。这是一个关键的设计决策：

- **所有 DC（数据缓存）操作均为 NOP**：QEMU 内存模型是一致的，无需 clean/invalidate
- **所有 IC（指令缓存）操作均为 NOP**：除了 user-mode 下的 `IC IVAU`
- **DC ZVA 是唯一有实际效果的缓存操作**：它真正清零内存块
- **缓存 ID 寄存器**（CCSIDR/CLIDR/CTR）按 CPU 模型配置，供 Guest OS 读取

这意味着：
1. Guest 看到的缓存拓扑是正确的（可以正确查询缓存级别、行大小等）
2. Guest 的缓存维护操作不会失败（访问权限检查正常执行）
3. 但操作本身不做任何事（因为没有真实缓存需要维护）

---

## 2. Cache ID 寄存器体系

ARM 定义了一组寄存器让软件查询缓存拓扑：

| 寄存器 | 功能 | QEMU 来源 |
|--------|------|-----------|
| CLIDR_EL1 | 缓存级别标识（几级缓存、每级类型） | `cpu->clidr` |
| CCSIDR_EL1 | 缓存大小标识（行大小、关联度、组数） | `cpu->ccsidr[index]` |
| CSSELR_EL1 | 缓存选择（选择 CCSIDR 返回哪级缓存） | `cp15.csselr_s/ns` |
| CTR_EL0 | 缓存类型（I/D 最小行大小、CWG、ERG） | `cpu->ctr` |
| DCZID_EL0 | DC ZVA 块大小 | 从 `cpu->dcz_blocksize` 计算 |

---

## 3. CCSIDR 寄存器与 make_ccsidr()

```c
// target/arm/cpu-features.h:1644-1690
typedef enum {
    CCSIDR_FORMAT_LEGACY,  // 32-bit 格式
    CCSIDR_FORMAT_CCIDX,   // 64-bit 格式 (FEAT_CCIDX)
} CCSIDRFormat;

static inline uint64_t make_ccsidr(CCSIDRFormat format, unsigned assoc,
                                    unsigned linesize, unsigned cachesize,
                                    uint8_t flags)
{
    unsigned lg_linesize = ctz32(linesize);
    unsigned sets = cachesize / (assoc * linesize);

    if (format == CCSIDR_FORMAT_LEGACY) {
        // 32-bit CCSIDR:
        // [27:13] = sets - 1
        // [12:3]  = associativity - 1
        // [2:0]   = log2(linesize) - 4
        ccsidr = deposit32(ccsidr, 13, 15, sets - 1);
        ccsidr = deposit32(ccsidr,  3, 10, assoc - 1);
        ccsidr = deposit32(ccsidr,  0,  3, lg_linesize - 4);
    } else {
        // 64-bit CCSIDR_EL1 (CCIDX):
        // [55:32] = sets - 1
        // [23:3]  = associativity - 1
        // [2:0]   = log2(linesize) - 4
        ccsidr = deposit64(ccsidr, 32, 24, sets - 1);
        ccsidr = deposit64(ccsidr,  3, 21, assoc - 1);
        ccsidr = deposit64(ccsidr,  0,  3, lg_linesize - 4);
    }
    return ccsidr;
}
```

**用法**：CPU 模型初始化时调用 `make_ccsidr()` 填充 `cpu->ccsidr[]` 数组。

---

## 4. ccsidr_read() 实现

```c
// target/arm/helper.c:859-871
static uint64_t ccsidr_read(CPUARMState *env, const ARMCPRegInfo *ri)
{
    ARMCPU *cpu = env_archcpu(env);

    // 获取当前 CSSELR 索引（区分 Secure/Non-Secure bank）
    uint32_t index = A32_BANKED_REG_GET(env, csselr,
                                         ri->secure & ARM_CP_SECSTATE_S);
    return cpu->ccsidr[index];
}
```

**CSSELR 编码**：`[3:1] = 缓存级别 (0=L1, 1=L2, ...)`, `[0] = InD (0=Data/Unified, 1=Instruction)`

所以 `index = 0` = L1D，`index = 1` = L1I，`index = 2` = L2D，以此类推。

---

## 5. CSSELR 缓存级别选择

```c
// target/arm/helper.c:873-877, 948-955
static void csselr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                          uint64_t value)
{
    raw_write(env, ri, value & 0xf);  // 只有低 4 位有效
}

// 寄存器定义（v7_cp_reginfo 数组中）
{ .name = "CSSELR", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .crn = 0, .crm = 0, .opc1 = 2, .opc2 = 0,
  .access = PL1_RW,
  .accessfn = access_tid4,      // HCR_EL2.TID4 陷阱
  .fgt = FGT_CSSELR_EL1,
  .writefn = csselr_write,
  .bank_fieldoffsets = { offsetof(CPUARMState, cp15.csselr_s),
                          offsetof(CPUARMState, cp15.csselr_ns) } },
```

**Banked 存储**：Secure 和 Non-Secure 各有独立的 CSSELR，可以同时选择不同缓存级别。

---

## 6. CLIDR 缓存级别标识

```c
// target/arm/helper.c:6344-6356 (register_cp_regs_for_features 中)
// CLIDR 寄存器直接读取 cpu->clidr
```

**CLIDR 编码**：每 3 位描述一个缓存级别的类型：
- `000` = 无缓存
- `001` = 仅指令缓存
- `010` = 仅数据缓存
- `011` = 独立的 I/D 缓存
- `100` = 统一缓存

典型值（如 Cortex-A57）：`0x0A200023`
- Level 1: `011` = 独立 I/D
- Level 2: `100` = 统一
- Level 3+: 无

---

## 7. CTR_EL0 缓存类型寄存器

```c
// target/arm/helper.c:7072-7080
// CTR/CTR_EL0 寄存器定义
{ .name = "CTR",
  .access = PL1_R, ...
  .fieldoffset = offsetof(CPUARMState, cp15.c0_cacheattr),  // AArch32
  // 或 cpu->ctr (AArch64)
}
```

**CTR_EL0 字段**：

| 字段 | 位域 | 说明 |
|------|------|------|
| IminLine | [3:0] | I-cache 最小行大小 (log2 字数) |
| L1Ip | [15:14] | L1 I-cache 策略 (PIPT/VIPT/AIVIVT) |
| DminLine | [19:16] | D-cache 最小行大小 (log2 字数) |
| ERG | [23:20] | Exclusives Reservation Granule |
| CWG | [27:24] | Cache Writeback Granule |
| DIC | [29] | IC 指令不需要（数据→指令一致性） |
| IDC | [28] | DC 不需要（数据→指令可见性） |

**典型值**：`0x8444c004`（A57/A72）→ DminLine=4(64B)，IminLine=4(64B)，CWG=4，ERG=4

**User-mode 特殊处理**（`cpu.c:1883-1890`）：
```c
// 强制 DIC=0，使得 IC IVAU 被使用（而非被跳过）
// 这是为了让 JIT 代码的自修改能被检测到
```

---

## 8. DCZID_EL0 与 DC ZVA 块大小

```c
// target/arm/helper.c:3466-3470 (v8_cp_reginfo 数组中)
{ .name = "DCZID_EL0", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 3, .opc2 = 7, .crn = 0, .crm = 0,
  .access = PL0_R, .type = ARM_CP_NO_RAW,
  .fgt = FGT_DCZID_EL0,
  .readfn = aa64_dczid_read },
```

```c
// target/arm/cpu.h:1188-1198
static inline uint32_t get_dczid_bs(ARMCPU *cpu)
{
    return cpu->dcz_blocksize;
}
// blocksize 字段：编码为 log2(字数) - 2
// 例：blocksize=4 表示 4<<4 = 64 字节
```

---

## 9. ARMCPU 缓存参数存储

```c
// target/arm/cpu.h:1095-1106
typedef struct ARMCPU {
    ...
    uint64_t ctr;                // CTR_EL0 值
    /* 缓存大小标识数组，按 L1D, L1I, L2D, L2I, ... 顺序 */
    uint64_t ccsidr[16];
    ...
};
```

每个 CPU 模型在初始化时设置这些值，Guest 通过系统寄存器读取。

---

## 10. CPU 模型缓存配置示例

```c
// target/arm/cpu64.c / target/arm/tcg/cpu64.c
// Cortex-A57 示例：
static void aarch64_a57_initfn(Object *obj)
{
    ...
    cpu->ctr = 0x8444c004;
    cpu->clidr = 0x0a200023;
    cpu->ccsidr[0] = 0x701fe00a; // L1D: 32KB, 2-way, 64B line
    cpu->ccsidr[1] = 0x201fe012; // L1I: 48KB, 3-way, 64B line
    cpu->ccsidr[2] = 0x70ffe07a; // L2: 2MB, 16-way, 64B line
    cpu->dcz_blocksize = 4;      // DC ZVA 64 字节块
    ...
}
```

**不同 CPU 模型的缓存差异**：
- A35/A53：较小缓存，低功耗配置
- A57/A72：大缓存，高性能配置
- Neoverse：服务器级大缓存
- `max`：QEMU 定义的最大能力 CPU

---

## 11. arm_cpu_realizefn() 缓存初始化

```c
// target/arm/cpu.c:1740-1905 (arm_cpu_realizefn 片段)
// DC ZVA 块大小验证
if (cpu->dcz_blocksize > ...) {
    // 确保块大小不超过页大小
    assert(4 << cpu->dcz_blocksize <= TARGET_PAGE_SIZE);
}
```

`realizefn` 在 CPU 创建完成时被调用，负责最终的一致性检查和特性注册。

---

## 12. AArch64 Cache 维护操作注册

所有 AArch64 缓存操作在 `v8_cp_reginfo[]` 数组中注册（`helper.c:3444-3575`）：

```c
// helper.c:3487-3541 — AArch64 IC 操作
{ .name = "IC_IALLUIS", .state = ARM_CP_STATE_AA64,
  .opc0 = 1, .opc1 = 0, .crn = 7, .crm = 1, .opc2 = 0,
  .access = PL1_W, .type = ARM_CP_NOP,          // ← NOP!
  .fgt = FGT_ICIALLUIS,
  .accessfn = access_ticab },

// helper.c:3509-3541 — AArch64 DC 操作
{ .name = "DC_IVAC", .state = ARM_CP_STATE_AA64,
  .opc0 = 1, .opc1 = 0, .crn = 7, .crm = 6, .opc2 = 1,
  .access = PL1_W, .accessfn = aa64_cacheop_poc_access,
  .fgt = FGT_DCIVAC,
  .type = ARM_CP_NOP },                          // ← NOP!

{ .name = "DC_CVAC", ...
  .type = ARM_CP_NOP },                          // ← NOP!

{ .name = "DC_CIVAC", ...
  .type = ARM_CP_NOP },                          // ← NOP!
```

**关键点**：几乎所有操作的 `type` 都包含 `ARM_CP_NOP`，意味着写入被忽略。

---

## 13. 指令缓存操作 (IC)

| 指令 | 编码 | QEMU 行为 | 源码位置 |
|------|------|-----------|----------|
| IC IALLUIS | op0=1,op1=0,crn=7,crm=1,op2=0 | **NOP** | helper.c:3487-3491 |
| IC IALLU | op0=1,op1=0,crn=7,crm=5,op2=0 | **NOP** | helper.c:3492-3496 |
| IC IVAU | op0=1,op1=3,crn=7,crm=5,op2=1 | **NOP** (system) / **TB 失效** (user) | helper.c:3497-3508 |

**IC IVAU 的双重身份**：
```c
// helper.c:3497-3508
{ .name = "IC_IVAU", .state = ARM_CP_STATE_AA64,
  ...
#ifdef CONFIG_USER_ONLY
  .type = ARM_CP_NO_RAW,
  .writefn = ic_ivau_write        // User-mode: 实际执行 TB 失效
#else
  .type = ARM_CP_NOP              // System-mode: NOP
#endif
},
```

---

## 14. 数据缓存操作 (DC)

| 指令 | 操作 | QEMU 行为 | 访问控制 |
|------|------|-----------|----------|
| DC IVAC | Invalidate by VA to PoC | NOP | PoC (aa64_cacheop_poc_access) |
| DC ISW | Invalidate by Set/Way | NOP | access_tsw (HCR.TSW 陷阱) |
| DC CVAC | Clean by VA to PoC | NOP | PoC |
| DC CSW | Clean by Set/Way | NOP | access_tsw |
| DC CVAU | Clean by VA to PoU | NOP | PoU (access_tocu) |
| DC CIVAC | Clean+Invalidate by VA to PoC | NOP | PoC |
| DC CISW | Clean+Invalidate by Set/Way | NOP | access_tsw |
| **DC ZVA** | **Zero by VA** | **实际清零内存** | aa64_zva_access |

**唯一例外**：DC ZVA 不是 NOP，它真正将内存块清零为全 0。

---

## 15. DC ZVA 实现：真正的内存清零

```c
// helper.c:3471-3479 (寄存器定义)
{ .name = "DC_ZVA", .state = ARM_CP_STATE_AA64,
  .opc0 = 1, .opc1 = 3, .crn = 7, .crm = 4, .opc2 = 1,
  .access = PL0_W, .type = ARM_CP_DC_ZVA,   // 特殊类型！
  .accessfn = aa64_zva_access,               // EL0 陷阱控制
  .fgt = FGT_DCZVA,
},
```

**ARM_CP_DC_ZVA**：这是一个特殊的 CP 类型标记，translate 层会生成对 `HELPER(dc_zva)` 的调用，而不是简单的 NOP。

---

## 16. HELPER(dc_zva) 详细分析

```c
// target/arm/tcg/helper-a64.c:788-830+
void HELPER(dc_zva)(CPUARMState *env, uint64_t vaddr_in)
{
    uintptr_t ra = GETPC();

    // 计算块大小和对齐地址
    int blocklen = 4 << get_dczid_bs(env_archcpu(env));  // 通常 64 字节
    uint64_t vaddr = vaddr_in & ~(blocklen - 1);          // 对齐到块边界
    int mmu_idx = arm_env_mmu_index(env);

    // 快速路径：尝试无陷阱查找
    void *mem = tlb_vaddr_to_host(env, vaddr, MMU_DATA_STORE, mmu_idx);

    if (unlikely(!mem)) {
        // 慢速路径：可能是无效页、I/O、watchpoint
        // 1. 先用原始地址 probe（检测无效页 → 产生异常）
        (void) probe_write(env, vaddr_in, 1, mmu_idx, ra);
        // 2. 再用对齐地址 probe（检测 watchpoint）
        mem = probe_write(env, vaddr, blocklen, mmu_idx, ra);

        if (unlikely(!mem)) {
            // I/O 内存：逐字节写零（架构要求）
            for (int i = 0; i < blocklen; i++) {
                cpu_stb_mmuidx_ra(env, vaddr + i, 0, mmu_idx, ra);
            }
            return;
        }
    }

    // 快速路径：直接 memset
    memset(mem, 0, blocklen);
}
```

**关键设计**：
1. **对齐**：地址按块大小对齐，多余的低位被清零
2. **快速路径**：`tlb_vaddr_to_host()` 直接获取宿主内存指针，用 `memset` 清零
3. **慢速路径**：处理 I/O 映射、watchpoint、页错误
4. **不检查内存属性**：注释明确说不实现 Device 内存的对齐异常（QEMU 通用行为）

---

## 17. PoC/PoU 访问控制

ARM 缓存操作按"点"分类：
- **PoC (Point of Coherency)**：所有观察者看到一致数据的点
- **PoU (Point of Unification)**：I-cache 和 D-cache 看到一致数据的点

QEMU 不模拟这些概念本身，但实现了相关的**陷阱控制**：

---

## 18. aa64_cacheop_poc_access() PoC 陷阱

```c
// target/arm/helper.c:3145-3165
static CPAccessResult aa64_cacheop_poc_access(CPUARMState *env,
                                               const ARMCPRegInfo *ri,
                                               bool isread)
{
    switch (arm_current_el(env)) {
    case 0:
        // EL0 必须检查 SCTLR_EL1.UCI
        if (!(arm_sctlr(env, 0) & SCTLR_UCI))
            return CP_ACCESS_TRAP_EL1;
        /* fall through */
    case 1:
        // EL1 检查 HCR_EL2.TPCP（Trap PoC cache maintenance）
        if (arm_hcr_el2_eff(env) & HCR_TPCP)
            return CP_ACCESS_TRAP_EL2;
        break;
    }
    return CP_ACCESS_OK;
}
```

---

## 19. do_cacheop_pou_access() PoU 陷阱

```c
// target/arm/helper.c:3167-3185
static CPAccessResult do_cacheop_pou_access(CPUARMState *env,
                                             uint64_t hcrflags)
{
    switch (arm_current_el(env)) {
    case 0:
        if (!(arm_sctlr(env, 0) & SCTLR_UCI))
            return CP_ACCESS_TRAP_EL1;
        /* fall through */
    case 1:
        if (arm_hcr_el2_eff(env) & hcrflags)
            return CP_ACCESS_TRAP_EL2;
        break;
    }
    return CP_ACCESS_OK;
}

// 具体使用
static CPAccessResult access_ticab(...)  // IC IALLUIS
{
    return do_cacheop_pou_access(env, HCR_TICAB | HCR_TPU);
}

static CPAccessResult access_tocu(...)  // IC IALLU, IC IVAU, DC CVAU
{
    return do_cacheop_pou_access(env, HCR_TOCU | HCR_TPU);
}
```

**Hypervisor 陷阱**：即使操作是 NOP，EL2 仍然可以通过 HCR_EL2 位陷阱这些操作。这对虚拟化正确性很重要。

---

## 20. 自修改代码与 IC IVAU

QEMU TCG 将 Guest 代码翻译成 Translation Blocks (TB)。当 Guest 代码修改自身时，需要使 TB 失效。

**问题**：QEMU 不模拟缓存，所以数据写入直接对所有读者可见。但 TB 缓存了旧的翻译结果，需要被失效。

**解决方案**：在 user-mode 下，`IC IVAU` 触发 TB 失效。

---

## 21. ic_ivau_write() TB 失效

```c
// target/arm/helper.c:3414-3441
#ifdef CONFIG_USER_ONLY
/*
 * IC IVAU 用于改善与 JIT 的兼容性：
 * 双映射代码绕过 W^X 限制 — 一个区域可写，另一个可执行。
 * 可执行区域永远不会被写入，所以我们无法通过写来检测代码变更，
 * 依赖模拟的 JIT 通过执行此指令来通知我们。
 */
static void ic_ivau_write(CPUARMState *env, const ARMCPRegInfo *ri,
                           uint64_t value)
{
    uint64_t icache_line_mask, start_address, end_address;
    const ARMCPU *cpu = env_archcpu(env);

    // 从 CTR_EL0 获取 I-cache 行大小
    icache_line_mask = (4 << extract32(cpu->ctr, 0, 4)) - 1;
    start_address = value & ~icache_line_mask;
    end_address = value | icache_line_mask;

    mmap_lock();
    // 失效指定地址范围的翻译块
    tb_invalidate_phys_range(env_cpu(env), start_address, end_address);
    mmap_unlock();
}
#endif
```

**关键细节**：
- 使用 `CTR_EL0.IminLine` 确定失效范围（一个 I-cache 行）
- `tb_invalidate_phys_range()` 遍历并移除覆盖该地址范围的所有 TB
- 仅在 `CONFIG_USER_ONLY` 下生效（system mode 使用其他机制检测代码修改）

---

## 22. AArch32 CP15 缓存操作

```c
// target/arm/helper.c:3549-3575 (v8_cp_reginfo 数组中的 32-bit 操作)

// 指令缓存操作
{ .name = "ICIALLUIS", .cp = 15, .opc1 = 0, .crn = 7, .crm = 1, .opc2 = 0,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = access_ticab },
{ .name = "ICIALLU", .cp = 15, .opc1 = 0, .crn = 7, .crm = 5, .opc2 = 0,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = access_tocu },
{ .name = "ICIMVAU", .cp = 15, .opc1 = 0, .crn = 7, .crm = 5, .opc2 = 1,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = access_tocu },

// 数据缓存操作
{ .name = "DCIMVAC", .cp = 15, .opc1 = 0, .crn = 7, .crm = 6, .opc2 = 1,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = aa64_cacheop_poc_access },
{ .name = "DCISW", .cp = 15, .opc1 = 0, .crn = 7, .crm = 6, .opc2 = 2,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = access_tsw },
{ .name = "DCCMVAC", .cp = 15, .opc1 = 0, .crn = 7, .crm = 10, .opc2 = 1,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = aa64_cacheop_poc_access },
{ .name = "DCCMVAU", .cp = 15, .opc1 = 0, .crn = 7, .crm = 11, .opc2 = 1,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = access_tocu },
{ .name = "DCCIMVAC", .cp = 15, .opc1 = 0, .crn = 7, .crm = 14, .opc2 = 1,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = aa64_cacheop_poc_access },
{ .name = "DCCISW", .cp = 15, .opc1 = 0, .crn = 7, .crm = 14, .opc2 = 2,
  .type = ARM_CP_NOP, .access = PL1_W, .accessfn = access_tsw },
```

**AArch32 vs AArch64 对应**：

| AArch32 (MCR p15) | AArch64 (SYS) | 操作 |
|-------------------|---------------|------|
| ICIALLUIS | IC IALLUIS | 全 IS 指令缓存失效 |
| ICIALLU | IC IALLU | 本地指令缓存失效 |
| ICIMVAU | IC IVAU | 按 VA 指令缓存失效 |
| DCIMVAC | DC IVAC | 按 VA 数据缓存失效 |
| DCCMVAC | DC CVAC | 按 VA 数据缓存清理 |
| DCCIMVAC | DC CIVAC | 按 VA 数据缓存清理+失效 |

---

## 23. 分支预测操作 (BP)

```c
// helper.c:3552-3561
{ .name = "BPIALLUIS", .cp = 15, .opc1 = 0, .crn = 7, .crm = 1, .opc2 = 6,
  .type = ARM_CP_NOP, .access = PL1_W },
{ .name = "BPIALL", .cp = 15, .opc1 = 0, .crn = 7, .crm = 5, .opc2 = 6,
  .type = ARM_CP_NOP, .access = PL1_W },
{ .name = "BPIMVA", .cp = 15, .opc1 = 0, .crn = 7, .crm = 5, .opc2 = 7,
  .type = ARM_CP_NOP, .access = PL1_W },
```

QEMU 不模拟分支预测器，所有 BP 操作均为 NOP。

---

## 24. HCR_EL2 缓存陷阱控制

Hypervisor 可以通过 HCR_EL2 位陷阱 Guest 的缓存操作：

| HCR_EL2 位 | 陷阱的操作 | 检查函数 |
|------------|-----------|----------|
| TPU | PoU 操作 (IC IALLU, IC IVAU, DC CVAU) | do_cacheop_pou_access |
| TPCP | PoC 操作 (DC CVAC, DC IVAC, DC CIVAC) | aa64_cacheop_poc_access |
| TSW | Set/Way 操作 (DC ISW, DC CSW, DC CISW) | access_tsw |
| TICAB | IC IALLUIS (全 IS 范围) | access_ticab |
| TOCU | PoU 操作 (额外陷阱) | access_tocu |

这些陷阱允许 Hypervisor：
- 跟踪 Guest 的缓存维护操作
- 实现影子页表的一致性维护
- 在 Set/Way 操作时执行全 TLB 刷新

---

## 25. SCTLR 缓存控制位

| SCTLR 位 | 说明 | QEMU 影响 |
|-----------|------|-----------|
| M (bit 0) | MMU 使能 | 控制地址翻译 |
| A (bit 1) | 对齐检查 | QEMU 不完全实现 |
| C (bit 2) | D-cache 使能 | QEMU 忽略（无真实缓存） |
| I (bit 12) | I-cache 使能 | QEMU 忽略 |
| UCI (bit 26) | EL0 缓存操作陷阱 | **实现**（控制 DC/IC 访问权限） |

**SCTLR.C 和 SCTLR.I**：在真实硬件上控制缓存开关。QEMU 记录这些位但不改变内存行为，因为没有真实缓存。

---

## 26. QEMU TLB 与缓存操作的交互

QEMU 的 softmmu TLB 缓存了虚拟→物理地址映射。大多数缓存操作不触发 TLB 操作，但：

1. **sctlr_write()**（`helper.c:3265-3298`）：
   ```c
   static void sctlr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                             uint64_t value)
   {
       if (raw_read(env, ri) == value) return;  // 值未变则跳过
       // ...
       tlb_flush(CPU(cpu));  // SCTLR 改变 → 全 TLB 刷新
   }
   ```

2. **DC ZVA**：通过 `tlb_vaddr_to_host()` 使用 TLB 查找宿主地址

3. **IC IVAU (user-mode)**：通过 `tb_invalidate_phys_range()` 失效翻译块，不触发 TLB 刷新

---

## 27. 内存类型属性处理

QEMU 对内存类型的处理是简化的：

- **Normal Cacheable**：正常读写
- **Normal Non-Cacheable**：QEMU 不区分，行为相同
- **Device Memory**：部分实现（不实现对齐异常、不实现 DC ZVA 的 Device 异常）
- **Write-Through vs Write-Back**：QEMU 不区分

```c
// helper-a64.c:793-797 (DC ZVA 注释)
/*
 * Note that we do not implement the (architecturally mandated)
 * alignment fault for attempts to use this on Device memory
 * (which matches the usual QEMU behaviour of not implementing either
 * alignment faults or any memory attribute handling).
 */
```

---

## 28. 关键函数总览表

| 函数 | 文件 | 行号 | 说明 |
|------|------|------|------|
| `ccsidr_read()` | helper.c | 859-871 | 读取 CCSIDR（按 CSSELR 索引） |
| `csselr_write()` | helper.c | 873-877 | 写入 CSSELR（缓存选择） |
| `make_ccsidr()` | cpu-features.h | 1649-1690 | 构造 CCSIDR 值 |
| `aa64_dczid_read()` | helper.c | 3466-3470 | 读取 DCZID_EL0 |
| `aa64_cacheop_poc_access()` | helper.c | 3145-3165 | PoC 操作访问检查 |
| `do_cacheop_pou_access()` | helper.c | 3167-3185 | PoU 操作访问检查 |
| `access_ticab()` | helper.c | 3187-3191 | IC IALLUIS 陷阱 |
| `access_tocu()` | helper.c | 3193-3197 | PoU 操作陷阱 |
| `aa64_zva_access()` | helper.c | 3199+ | DC ZVA 访问检查 |
| `ic_ivau_write()` | helper.c | 3424-3441 | IC IVAU TB 失效（user-mode） |
| `HELPER(dc_zva)` | helper-a64.c | 788-830+ | DC ZVA 内存清零 |
| `sctlr_write()` | helper.c | 3265-3298 | SCTLR 写入（触发 TLB 刷新） |
| `get_dczid_bs()` | cpu.h | 1188-1198 | 获取 DC ZVA 块大小 |

---

## 29. Cache 操作指令完整映射表

### AArch64 指令

| 指令 | 系统编码 | 访问级别 | 访问控制 | QEMU 行为 | 源码行 |
|------|----------|----------|----------|-----------|--------|
| IC IALLUIS | 1,0,7,1,0 | PL1_W | access_ticab | NOP | 3487-3491 |
| IC IALLU | 1,0,7,5,0 | PL1_W | access_tocu | NOP | 3492-3496 |
| IC IVAU | 1,3,7,5,1 | PL0_W | access_tocu | NOP/TB失效 | 3497-3508 |
| DC IVAC | 1,0,7,6,1 | PL1_W | poc_access | NOP | 3510-3514 |
| DC ISW | 1,0,7,6,2 | PL1_W | access_tsw | NOP | 3515-3518 |
| DC CVAC | 1,3,7,10,1 | PL0_W | poc_access | NOP | 3519-3523 |
| DC CSW | 1,0,7,10,2 | PL1_W | access_tsw | NOP | 3524-3527 |
| DC CVAU | 1,3,7,11,1 | PL0_W | access_tocu | NOP | 3528-3532 |
| DC CIVAC | 1,3,7,14,1 | PL0_W | poc_access | NOP | 3533-3537 |
| DC CISW | 1,0,7,14,2 | PL1_W | access_tsw | NOP | 3538-3541 |
| **DC ZVA** | 1,3,7,4,1 | PL0_W | zva_access | **清零内存** | 3471-3479 |

### AArch32 指令 (MCR p15, 0, Rd, c7, ...)

| 指令 | CRm.opc2 | QEMU 行为 | 源码行 |
|------|----------|-----------|--------|
| ICIALLUIS | c1.0 | NOP | 3550-3551 |
| ICIALLU | c5.0 | NOP | 3554-3555 |
| ICIMVAU | c5.1 | NOP | 3556-3557 |
| DCIMVAC | c6.1 | NOP | 3562-3563 |
| DCISW | c6.2 | NOP | 3564-3565 |
| DCCMVAC | c10.1 | NOP | 3566-3567 |
| DCCMVAU | c11.1 | NOP | 3570-3571 |
| DCCIMVAC | c14.1 | NOP | 3572-3573 |
| DCCISW | c14.2 | NOP | 3574-3575 |
| BPIALLUIS | c1.6 | NOP | 3552-3553 |
| BPIALL | c5.6 | NOP | 3558-3559 |
| BPIMVA | c5.7 | NOP | 3560-3561 |

---

## 30. QEMU 缓存模拟策略总结

```
┌──────────────────────────────────────────────────────────────────────┐
│                    QEMU ARM 缓存模拟策略                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 缓存 ID 寄存器 ✅ 完整模拟                                       │
│     CCSIDR/CLIDR/CTR/DCZID → 按 CPU 模型配置                        │
│     Guest 可以正确查询缓存拓扑和行大小                                  │
│                                                                      │
│  2. 缓存维护操作 ⚡ 访问权限检查 + NOP                                │
│     DC IVAC/CVAC/CIVAC/ISW/CSW/CISW → NOP                           │
│     IC IALLUIS/IALLU → NOP                                           │
│     ✓ 访问权限检查完整实现（SCTLR.UCI, HCR.TPCP/TPU/TSW）           │
│     ✓ EL 陷阱机制完整实现                                             │
│     ✗ 不实际操作缓存（因为没有缓存）                                    │
│                                                                      │
│  3. DC ZVA ✅ 完整实现                                                │
│     真正将内存块清零                                                   │
│     支持快速路径 (memset) 和慢速路径 (逐字节)                           │
│     块大小按 CPU 模型配置                                              │
│                                                                      │
│  4. IC IVAU (user-mode) ✅ 功能实现                                  │
│     TB 失效：支持自修改代码和 JIT 双映射                                │
│     使用 CTR.IminLine 确定失效范围                                     │
│     tb_invalidate_phys_range() 移除旧翻译块                           │
│                                                                      │
│  5. 分支预测操作 → 全部 NOP                                           │
│     BPIALLUIS/BPIALL/BPIMVA → NOP                                    │
│                                                                      │
│  6. 内存属性 ⚠ 简化处理                                              │
│     不区分 Cacheable/Non-Cacheable                                    │
│     不区分 Write-Through/Write-Back                                   │
│     不实现 Device 内存对齐异常                                         │
│                                                                      │
│  设计原则：                                                           │
│  • QEMU 内存模型是强一致的（所有写立即对所有读可见）                     │
│  • 无需缓存一致性维护                                                  │
│  • 保留访问权限语义（陷阱/EL 检查）以支持虚拟化正确性                    │
│  • DC ZVA 因为有可观测副作用（清零）所以必须实现                        │
│  • IC IVAU 因为影响 TCG TB 缓存所以在 user-mode 实现                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

**适合读者**：需要理解 QEMU 如何处理 ARM 缓存操作、DC ZVA 实现机制、缓存 ID 寄存器配置的开发者。

**关键源文件**：
- `target/arm/helper.c`（~10200行）— 缓存寄存器定义、访问控制、IC IVAU
- `target/arm/tcg/helper-a64.c`（~1000行）— DC ZVA 实现
- `target/arm/cpu.h`（~3500行）— ccsidr[]、ctr、dcz_blocksize
- `target/arm/cpu-features.h`（~1700行）— make_ccsidr()
- `target/arm/cpu.c`（~2500行）— arm_cpu_realizefn 缓存初始化
- `target/arm/cpu64.c`（~1000行）— CPU 模型缓存参数
