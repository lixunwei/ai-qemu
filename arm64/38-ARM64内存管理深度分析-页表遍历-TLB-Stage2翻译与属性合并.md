# 38 - ARM64 内存管理深度分析 — 页表遍历、TLB、Stage-2 翻译与属性合并

> **基于 QEMU 11.0.50 源码**，深入分析 ARM64 内存管理单元（MMU）在 QEMU 中的完整实现：
> ARMMMUIdx 翻译体制索引、get_phys_addr/get_phys_addr_lpae 页表遍历、
> TCR/TTBR 解析与粒度选择、描述符格式与权限检查、Stage-1+Stage-2 两阶段翻译、
> QEMU SoftMMU TLB 结构与 TLBI 指令、MAIR 属性合并、AT 指令与 PAR_EL1。

---

## 目录

1. [ARMMMUIdx：翻译体制索引](#1-armmmuidx翻译体制索引)
2. [翻译体制选择](#2-翻译体制选择)
3. [get_phys_addr：页表遍历入口](#3-get_phys_addr页表遍历入口)
4. [TCR 解析与 VA 参数](#4-tcr-解析与-va-参数)
5. [get_phys_addr_lpae：LPAE 页表遍历](#5-get_phys_addr_lpaelpae-页表遍历)
6. [描述符格式](#6-描述符格式)
7. [Stage-1 权限检查](#7-stage-1-权限检查)
8. [Stage-2 翻译](#8-stage-2-翻译)
9. [S1+S2 两阶段翻译](#9-s1s2-两阶段翻译)
10. [属性合并](#10-属性合并)
11. [QEMU SoftMMU TLB](#11-qemu-softmmu-tlb)
12. [arm_cpu_tlb_fill：TLB 填充](#12-arm_cpu_tlb_filltlb-填充)
13. [TLBI 指令模拟](#13-tlbi-指令模拟)
14. [AT 指令与 PAR_EL1](#14-at-指令与-par_el1)
15. [地址标签（TBI）](#15-地址标签tbi)
16. [数据流全景图](#16-数据流全景图)

---

## 1. ARMMMUIdx：翻译体制索引

```c
// mmuidx.h:137-198
typedef enum ARMMMUIdx {
    /* A-profile：对应不同 EL 和翻译体制 */
    ARMMMUIdx_E10_0      = 0,    // EL1&0 体制，EL0 访问
    ARMMMUIdx_E10_0_GCS  = 1,    // EL1&0 体制，EL0 GCS
    ARMMMUIdx_E10_1      = 2,    // EL1&0 体制，EL1 访问
    ARMMMUIdx_E10_1_PAN  = 3,    // EL1&0 体制，EL1 PAN
    ARMMMUIdx_E10_1_GCS  = 4,    // EL1&0 体制，EL1 GCS

    ARMMMUIdx_E20_0      = 5,    // EL2&0 体制（VHE），EL0 访问
    ARMMMUIdx_E20_0_GCS  = 6,    // EL2&0 体制，EL0 GCS
    ARMMMUIdx_E20_2      = 7,    // EL2&0 体制，EL2 访问
    ARMMMUIdx_E20_2_PAN  = 8,    // EL2&0 体制，EL2 PAN
    ARMMMUIdx_E20_2_GCS  = 9,    // EL2&0 体制，EL2 GCS

    ARMMMUIdx_E2         = 10,   // EL2 体制（无 VHE）
    ARMMMUIdx_E2_GCS     = 11,   // EL2 GCS
    ARMMMUIdx_E3         = 12,   // EL3 体制
    ARMMMUIdx_E3_GCS     = 13,   // EL3 GCS
    ARMMMUIdx_E30_0      = 14,   // EL3&0 体制
    ARMMMUIdx_E30_3_PAN  = 15,   // EL3&0 PAN

    /* Stage-2：第二阶段翻译 */
    ARMMMUIdx_Stage2_S   = 16,   // Secure Stage-2
    ARMMMUIdx_Stage2     = 17,   // Non-Secure Stage-2

    /* 物理地址空间：直通映射 */
    ARMMMUIdx_Phys_S     = 18,   // Secure 物理
    ARMMMUIdx_Phys_NS    = 19,   // Non-Secure 物理
    ARMMMUIdx_Phys_Root  = 20,   // Root 物理
    ARMMMUIdx_Phys_Realm = 21,   // Realm 物理

    /* Stage-1 only：不分配 TLB，用于 AT 指令和 S12 遍历 */
    ARMMMUIdx_Stage1_E0     = ..., // S1 only EL0
    ARMMMUIdx_Stage1_E1     = ..., // S1 only EL1
    ARMMMUIdx_Stage1_E1_PAN = ..., // S1 only EL1 PAN
} ARMMMUIdx;
```

---

## 2. 翻译体制选择

```c
// helper.c:9957-10008
ARMMMUIdx arm_mmu_idx_el(CPUARMState *env, int el)
{
    switch (el) {
    case 0:
        // VHE (HCR_EL2.E2H+TGE) → E20_0
        // Secure EL3&0 → E30_0
        // 默认 → E10_0
    case 1:
        // PAN 启用 → E10_1_PAN
        // 默认 → E10_1
    case 2:
        // VHE → E20_2 / E20_2_PAN
        // 无 VHE → E2
    case 3:
        // E30_3_PAN 或 E3
    }
}
```

**翻译体制映射表**：

| 当前 EL | 条件 | ARMMMUIdx | TCR | TTBR |
|---------|------|-----------|-----|------|
| EL0 | 标准 | E10_0 | TCR_EL1 | TTBR0/1_EL1 |
| EL0 | VHE | E20_0 | TCR_EL2 | TTBR0/1_EL2 |
| EL1 | 标准 | E10_1 | TCR_EL1 | TTBR0/1_EL1 |
| EL2 | 无 VHE | E2 | TCR_EL2 | TTBR0_EL2 |
| EL2 | VHE | E20_2 | TCR_EL2 | TTBR0/1_EL2 |
| EL3 | — | E3 | TCR_EL3 | TTBR0_EL3 |

---

## 3. get_phys_addr：页表遍历入口

```c
// ptw.c:3931-3943
bool get_phys_addr(CPUARMState *env, vaddr address,
                   MMUAccessType access_type, MemOp memop,
                   ARMMMUIdx mmu_idx,
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

### ARMMMUFaultInfo

```c
// internals.h:741-771
struct ARMMMUFaultInfo {
    ARMFaultType type;           // 故障类型（Translation/Access/Permission/...）
    ARMGPCF gpcf;                // GPC 故障子类型
    hwaddr s2addr;               // Stage-2 故障的 IPA
    hwaddr paddr;                // GPC 故障物理地址
    ARMSecuritySpace paddr_space;
    int level;                   // 页表遍历级别
    int domain;                  // 域（AArch32 only）
    bool stage2;                 // 是否 Stage-2 故障
    bool s1ptw;                  // S1 遍历时的 S2 故障
    bool s1ns;                   // Non-Secure IPA 故障
    bool ea;                     // 外部中止
    bool dirtybit;               // FEAT_S1PIE/S2PIE 脏位故障
};
```

---

## 4. TCR 解析与 VA 参数

```c
// helper.c:9672-9836
ARMVAParameters aa64_va_parameters(CPUARMState *env, uint64_t va,
                                   ARMMMUIdx mmu_idx, bool data, bool el1_is_aa32)
{
    // 关键字段解析：
    // T0SZ / T1SZ     — 地址空间大小（64 - TnSZ = 输入地址位数）
    // TG0 / TG1       — 粒度大小（0=4KB, 1=64KB, 2=16KB）
    // SH0 / SH1       — 共享属性
    // ORGN0 / IRGN0   — 缓存属性
    // EPD0 / EPD1     — 禁用 TTBR0/TTBR1 翻译
    // HA / HD         — 硬件 Access Flag / Dirty Bit 管理
    // DS              — 52 位输出地址

    // TTBR 选择：VA 的高位决定使用 TTBR0 或 TTBR1
    // select = VA[55]（对于有两个范围的体制）
}
```

**粒度大小与页表级别**：

| 粒度 | 页大小 | L0 | L1 | L2 | L3 |
|------|--------|----|----|----|----|
| 4KB  | 4KB    | 512GB | 1GB | 2MB | 4KB |
| 16KB | 16KB   | — | 128GB | 32MB | 16KB |
| 64KB | 64KB   | — | 4TB | 512MB | 64KB |

---

## 5. get_phys_addr_lpae：LPAE 页表遍历

```c
// ptw.c:1859-2449（关键流程摘录）
static bool get_phys_addr_lpae(CPUARMState *env, S1Translate *ptw,
                                uint64_t address, ...)
{
    // ① 获取 TCR 和 VA 参数
    tcr = regime_tcr(env, mmu_idx);
    param = aa64_va_parameters(env, address, mmu_idx, ...);

    // ② 选择 TTBR（根据 VA 高位 select）
    ttbr = regime_ttbr(env, mmu_idx, param.select);

    // ③ 计算起始级别和步长
    //    stride = granule_bits - 3（4KB→9, 16KB→11, 64KB→13）
    //    level = 起始级别（由 inputsize 和 stride 决定）

    // ④ 提取基地址
    descaddr = extract64(ttbr, 0, 48) & ~indexmask;

    // ⑤ 逐级遍历循环
    for (;;) {
        // 读取描述符
        descriptor = arm_ldq_ptw(env, ptw, descaddr, fi);

        // 检查描述符有效性
        if (!(descriptor & 1)) {
            fi->type = ARMFault_Translation;
            fi->level = level;
            return true;  // Translation fault
        }

        // Table 描述符（非最后一级 + bit[1]=1）
        if ((descriptor & 2) && level < 3) {
            // 提取下一级表地址
            descaddr = descriptor & descaddrmask;
            // 累计表属性（NSTable, APTable, PXNTable, UXNTable）
            tableattrs |= extract64(descriptor, 59, 5);
            level++;
            continue;
        }

        // Block/Page 描述符 — 遍历结束
        break;
    }

    // ⑥ 提取输出地址
    page_size = 1ULL << ((stride * (4 - level)) + 3);
    // OA = descriptor[47:page_bits] | VA[page_bits-1:0]

    // ⑦ Access Flag 检查（HA 自动设置）
    if (!(descriptor & (1 << 10))) {  // AF bit
        if (param.ha) {
            // 硬件 AF 管理：原子设置 AF
        } else {
            fi->type = ARMFault_AccessFlag;
            return true;
        }
    }

    // ⑧ Dirty Bit 管理（HD）
    // 如果写访问且 DBM 启用 → 原子清除 AP[2]

    // ⑨ 权限检查
    ap = extract32(attrs, 6, 2);  // AP[2:1]
    xn = extract64(attrs, 54, 1);
    pxn = extract64(attrs, 53, 1);
    prot = get_S1prot(...) 或 get_S2prot(...);
}
```

---

## 6. 描述符格式

### 6.1 AArch64 LPAE 描述符

```
63    59 58    52 51  48 47              12 11  2 1 0
┌──────┬────────┬──────┬──────────────────┬──────┬───┐
│ 上层 │ 上层   │ OA   │  OA[47:12]       │ 下层 │V T│
│ 属性 │ 属性   │ 高位 │  (输出地址)       │ 属性 │   │
└──────┴────────┴──────┴──────────────────┴──────┴───┘

Bit[0] = Valid
Bit[1] = Table(1) / Block(0)（L0-L2）；Page(1)（L3）

下层属性 [11:2]:
  AttrIndx[4:2]   — MAIR 索引
  NS[5]           — Non-Secure（仅 Secure 状态）
  AP[7:6]         — 访问权限
  SH[9:8]         — 共享属性
  AF[10]          — Access Flag

上层属性 [58:50, 63:59]:
  Contiguous[52]  — 连续提示
  PXN[53]         — 特权执行禁止
  UXN/XN[54]      — 用户/执行禁止
  DBM[51]         — Dirty Bit Management
```

### 6.2 AP 位编码（Stage-1）

| AP[2:1] | EL1 | EL0 |
|---------|-----|-----|
| 00 | RW | — |
| 01 | RW | RW |
| 10 | RO | — |
| 11 | RO | RO |

### 6.3 S2AP 位编码（Stage-2）

| S2AP[1:0] | 权限 |
|-----------|------|
| 00 | 无访问 |
| 01 | 只读 |
| 10 | 只写 |
| 11 | 读写 |

---

## 7. Stage-1 权限检查

```c
// ptw.c:1434-1541
static int get_S1prot(CPUARMState *env, ARMMMUIdx mmu_idx, bool is_aa64,
                       int user_rw, int prot_rw, int xn, int pxn,
                       ARMSecuritySpace in_pa, ARMSecuritySpace out_pa)
{
    bool is_user = regime_is_user(mmu_idx);

    // ① PAN 检查
    if (is_user) {
        prot_rw = user_rw;
    } else if (user_rw && regime_is_pan(mmu_idx)) {
        prot_rw = 0;    // PAN 阻止特权访问用户可访问页面
    }
    // PAN3 + EPAN：EL0 可执行也触发 PAN

    // ② 安全域检查
    if (in_pa != out_pa) {
        // Root → 不可从非 Root 取指
        // Realm → 不可从非 Realm 取指
        // Secure + SIF → 不可从非 Secure 取指
    }

    // ③ WXN（Write implies XN）
    wxn = regime_sctlr(env, mmu_idx) & SCTLR_WXN;

    // ④ 计算 XN
    // AArch64 EL1&0：PXN || (user 可写)
    // AArch32：不同规则

    // ⑤ 最终权限
    if (xn || (wxn && (prot_rw & PAGE_WRITE))) {
        return prot_rw;              // 不可执行
    }
    return prot_rw | PAGE_EXEC;      // 可执行
}
```

---

## 8. Stage-2 翻译

### 8.1 VTCR_EL2 / VTTBR_EL2

Stage-2 翻译由 Hypervisor 配置：
- `VTTBR_EL2`：Stage-2 页表基地址 + VMID
- `VTCR_EL2`：T0SZ（IPA 大小）、TG0（粒度）、SL0（起始级别）、SH0/ORGN0/IRGN0

### 8.2 Stage-2 权限

```c
// ptw.c:1343-1381
static int get_S2prot(CPUARMState *env, int s2ap, int xn, bool s1_is_el0)
{
    int prot = 0;

    if (s2ap & 1) prot |= PAGE_READ;   // S2AP[0] → 读
    if (s2ap & 2) prot |= PAGE_WRITE;  // S2AP[1] → 写

    if (!xn) {
        // XN=0 且 S2AP 允许读 → 可执行
        if (prot & PAGE_READ) {
            prot |= PAGE_EXEC;
        }
    }
    return prot;
}
```

### 8.3 S2 描述符特殊性

- 使用 S2AP[1:0] 替代 AP[2:1]
- MemAttr[3:0] 直接编码内存类型（不经过 MAIR）
- 支持 FWB（Forced Write-Back）特性

---

## 9. S1+S2 两阶段翻译

```c
// ptw.c:3554-3659
static bool get_phys_addr_twostage(CPUARMState *env, S1Translate *ptw,
                                    vaddr address, ...)
{
    // ① Stage-1 遍历：VA → IPA
    ret = get_phys_addr_nogpc(env, ptw, address, ...);
    if (ret) return ret;   // S1 故障直接返回

    ipa = result->f.phys_addr;
    s1_prot = result->f.prot;
    cacheattrs1 = result->cacheattrs;

    // ② 切换到 Stage-2
    ptw->in_mmu_idx = ipa_secure ? ARMMMUIdx_Stage2_S : ARMMMUIdx_Stage2;

    // ③ Stage-2 遍历：IPA → PA
    ret = get_phys_addr_nogpc(env, ptw, ipa, ...);
    fi->s2addr = ipa;    // 记录 IPA 用于故障报告

    // ④ 合并 S1 和 S2 权限
    result->f.prot = s1_prot & result->s2prot;

    // ⑤ 合并属性（MAIR/cacheability/shareability）
    combine_cacheattrs(result, cacheattrs1, ...);

    // ⑥ 选择最小页面大小
    result->f.lg_page_size = MIN(s1_lgpgsz, result->f.lg_page_size);
}
```

### S1 遍历中的 S2 翻译

S1 页表遍历过程中，每次读取页表描述符（S1 ptw）的地址本身也需要经过 S2 翻译：

```c
// ptw.c:644-731
static bool S1_ptw_translate(CPUARMState *env, S1Translate *ptw,
                              hwaddr addr, ARMMMUFaultInfo *fi)
{
    // S1 页表描述符地址 → S2 翻译 → 物理地址
    // 故障时设置 fi->s1ptw = true
}
```

---

## 10. 属性合并

### 10.1 无 FWB（标准合并）

```c
// ptw.c:3297-3339
// S1 属性通过 MAIR 索引获取内存类型
// S2 属性使用描述符中的 MemAttr[3:0]
// 合并规则：取更强约束
//   Device > Normal Non-Cacheable > Normal Cacheable
//   Inner 和 Outer 分别合并
```

### 10.2 FWB（Forced Write-Back）

```c
// ptw.c:3366-3403
// FEAT_S2FWB：S2 描述符可以强制覆盖 S1 的缓存属性
// MemAttr[3:2] = 01 → 强制 Non-Cacheable
// MemAttr[3:2] = 10 → 强制 Write-Through
// MemAttr[3:2] = 11 → 强制 Write-Back
// MemAttr[3:2] = 00 → 使用 S1 属性（标准合并）
```

### 10.3 共享属性合并

```c
// ptw.c:3413-3463
// 最终共享属性 = max(S1_shareability, S2_shareability)
// Device 内存总是 Outer Shareable
```

---

## 11. QEMU SoftMMU TLB

### 11.1 TLB 条目

```c
// tlb-common.h:25-54
typedef struct CPUTLBEntry {
    /* 三个 tag 字段，按访问类型索引 */
    uintptr_t addr_read;    // 读 tag（虚拟地址 + 标志）
    uintptr_t addr_write;   // 写 tag
    uintptr_t addr_code;    // 取指 tag

    /* 偏移量：vaddr + addend = host 地址 */
    uintptr_t addend;
} CPUTLBEntry;
```

### 11.2 TLB 结构

```c
// cpu.h:206-333
#define CPU_VTLB_SIZE 8              // Victim TLB 大小

typedef struct CPUTLBDesc {
    CPUTLBEntry vtable[CPU_VTLB_SIZE];     // Victim TLB 条目
    CPUTLBEntryFull vfulltlb[CPU_VTLB_SIZE]; // Victim 完整信息
    CPUTLBEntryFull *fulltlb;              // 主 TLB 完整信息
    // ...
} CPUTLBDesc;

typedef struct CPUTLB {
    CPUTLBDescFast f[NB_MMU_MODES];  // 快速路径（主 TLB 表）
    CPUTLBDesc d[NB_MMU_MODES];      // 描述（Victim TLB + 完整信息）
    // ...
} CPUTLB;
```

### 11.3 TLB 填充

```c
// cputlb.c:1024-1183
void tlb_set_page_full(CPUState *cpu, int mmu_idx, vaddr addr,
                       CPUTLBEntryFull *full)
{
    // 1. 计算 TLB 索引
    // 2. 旧条目写入 Victim TLB
    // 3. 设置 addr_read/write/code tag
    // 4. 计算 addend（vaddr → HVA 偏移）
    // 5. 对 MMIO/不可缓存设置特殊标志
}
```

---

## 12. arm_cpu_tlb_fill：TLB 填充

```c
// tlb_helper.c:331-379
bool arm_cpu_tlb_fill_align(CPUState *cs, CPUTLBEntryFull *out,
                             vaddr address, MMUAccessType access_type,
                             int mmu_idx, MemOp memop, int size,
                             bool probe, uintptr_t ra)
{
    // ① 对齐检查
    if (PC 非 4 字节对齐 || 内存访问非对齐) {
        fi->type = ARMFault_Alignment;
    }
    // ② 调用 get_phys_addr() 执行页表遍历
    else if (!get_phys_addr(&cpu->env, address, access_type, memop,
                            core_to_arm_mmu_idx(&cpu->env, mmu_idx),
                            &res, fi)) {
        // 成功：填充 TLB
        res.f.extra.arm.pte_attrs = res.cacheattrs.attrs;
        res.f.extra.arm.shareability = res.cacheattrs.shareability;
        *out = res.f;
        return true;
    }

    // ③ probe 模式：返回 false 不产生异常
    if (probe) return false;

    // ④ 产生真实 CPU 异常
    cpu_restore_state(cs, ra);
    arm_deliver_fault(cpu, address, access_type, mmu_idx, fi);
}
```

---

## 13. TLBI 指令模拟

```c
// tlb-insns.c:317-737（关键 TLBI 指令处理）

// TLBI VMALLE1 — 清除当前 VMID 的所有 EL1&0 TLB
// → tlb_flush_by_mmuidx(cs, alle1_tlbmask(env))

// TLBI VAE1 — 按 VA 清除 EL1&0 TLB
// → tlb_flush_page_by_mmuidx(cs, pageaddr, alle1_tlbmask(env))

// TLBI ALLE1 — 清除所有 EL1&0 TLB（跨 VMID）
// → tlb_flush_by_mmuidx_all_cpus_synced(cs, alle1_tlbmask(env))

// TLBI IPAS2E1 — 按 IPA 清除 Stage-2 TLB
// → 特殊处理：清除匹配 IPA 的 S2 条目
```

### mmuidx 掩码

```c
// helper.c:2796-2812
static inline uint16_t alle1_tlbmask(CPUARMState *env)
{
    // 返回 E10_0 | E10_1 | E10_1_PAN | ... 的位掩码
    // 用于按 mmuidx 批量清除 TLB
}
```

---

## 14. AT 指令与 PAR_EL1

```c
// cpregs-at.c:26-190
static uint64_t do_ats_write(CPUARMState *env, uint64_t value,
                              MMUAccessType access_type, ARMMMUIdx mmu_idx,
                              ARMSecuritySpace ss)
{
    // 1. 调用 get_phys_addr_for_at() 执行页表遍历

    // 2. 构建 PAR_EL1
    if (成功) {
        par64 = phys_addr & 0xFFFFFFFFF000;
        par64 |= (attrs << 56);           // ATTR
        par64 |= (shareability << 7);     // SH
        // PAR_EL1[0] = 0 (成功)
    } else {
        par64 = 1;                         // PAR_EL1[0] = 1 (失败)
        par64 |= (fsr & 0x3F) << 1;       // FST
        par64 |= (fi.stage2) << 9;        // S
        par64 |= (fi.s1ptw) << 8;         // PTW
    }
    return par64;
}

// cpregs-at.c:323-356 — AT 指令分发
// AT S1E1R  → do_ats_write(E10_1, READ)
// AT S1E1W  → do_ats_write(E10_1, WRITE)
// AT S12E1R → do_ats_write(E10_1, READ)  + Stage-2
// AT S1E2R  → do_ats_write(E2/E20_2, READ)
```

---

## 15. 地址标签（TBI）

```c
// helper.c:9554-9575
// aa64_va_parameter_tbi() — 获取 TBI 设置
// Stage-2 returns 0 (TBI 不适用于 IPA)

// ptw.c:3495-3519
// TBI 在翻译前检查：如果 TBI 启用，忽略 VA[63:56]
// 如果地址超出 TnSZ 定义的范围 → Address Size Fault

// hflags.c:406-448
// TBI/TBID 标志缓存到 hflags 中，翻译时使用
```

---

## 16. 数据流全景图

### 16.1 两阶段翻译全路径

```
Guest 内存访问（VA）
  │
  ▼
arm_cpu_tlb_fill_align()              [tlb_helper.c:331]
  │ TLB 未命中
  ▼
get_phys_addr()                        [ptw.c:3931]
  │ 构建 S1Translate
  ▼
get_phys_addr_gpc()                    [ptw.c:3784]
  │ GPC 检查（RME）
  ▼
get_phys_addr_twostage()              [ptw.c:3554]
  │
  ├── ① Stage-1：VA → IPA
  │     get_phys_addr_nogpc()
  │       └── get_phys_addr_lpae()    [ptw.c:1859]
  │             ├── aa64_va_parameters() — TCR/TG/T0SZ 解析
  │             ├── regime_ttbr() — TTBR0/TTBR1 选择
  │             ├── 逐级遍历描述符
  │             │    └── S1_ptw_translate() — 描述符地址的 S2 翻译
  │             ├── AF/DBM 硬件管理
  │             └── get_S1prot() — S1 权限检查
  │
  ├── ② Stage-2：IPA → PA
  │     get_phys_addr_nogpc()
  │       └── get_phys_addr_lpae()
  │             ├── VTTBR_EL2 / VTCR_EL2 解析
  │             ├── 逐级遍历 S2 描述符
  │             └── get_S2prot() — S2 权限检查
  │
  ├── ③ 权限合并：S1_prot & S2_prot
  │
  └── ④ 属性合并：combine_cacheattrs()
        ├── FWB → S2 覆盖 S1
        └── 非 FWB → 取更强约束
  │
  ▼
tlb_set_page_full()                    [cputlb.c:1024]
  │ 填充 SoftMMU TLB
  ▼
后续访问 TLB 命中 → 直接翻译
```

---

## 源文件索引

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `target/arm/mmuidx.h` | ~200 | ARMMMUIdx 枚举 (137-198)、mmuidx 掩码辅助函数 (202+) |
| `target/arm/ptw.c` | ~3950 | S1_ptw_translate (644-731)、get_S2prot (1343-1381)、get_S1prot (1434-1541)、check_s2_mmu_setup (1716-1800)、get_phys_addr_lpae (1859-2449)、属性合并 (3297-3463)、get_phys_addr_twostage (3554-3659)、get_phys_addr_gpc (3784-3833)、get_phys_addr (3931-3943) |
| `target/arm/internals.h` | ~1500 | ARMMMUFaultInfo (741-771) |
| `target/arm/helper.c` | ~10100 | VTCR_EL2/VTTBR_EL2 定义 (4155-4171)、TBI (9554-9575)、aa64_va_parameters (9672-9836)、arm_mmu_idx_el (9957-10008) |
| `target/arm/tcg/tlb_helper.c` | ~380 | arm_cpu_tlb_fill_align (331-379) |
| `target/arm/tcg/tlb-insns.c` | ~740 | TLBI 指令模拟 (317-737) |
| `target/arm/tcg/cpregs-at.c` | ~360 | do_ats_write (26-190)、AT 指令分发 (323-356) |
| `include/exec/tlb-common.h` | ~55 | CPUTLBEntry (25-54) |
| `include/hw/core/cpu.h` | ~340 | CPU_VTLB_SIZE (206)、CPUTLBDesc (277-297)、CPUTLB (327-333) |
| `accel/tcg/cputlb.c` | ~2600 | TLB flush (369-430)、tlb_set_page_full (1024-1183)、Victim TLB (1126-1138, 1304-1317) |
