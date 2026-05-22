# ARM64 页表遍历（PTW）深度分析：Stage-1/2 翻译、权限检查、Fault 处理、安全属性传播

> 基于 QEMU 11.0.50 源码分析，涵盖 ARM64 页表遍历（Page Table Walk）完整子系统：
> get_phys_addr 入口（S1Translate 结构 in_mmu_idx/in_ptw_idx/in_space/cur_space/in_s1_is_el0/in_prot_check/out_space 七字段、
> arm_mmu_idx_to_security_space→in_space 初始化、get_phys_addr_gpc→get_phys_addr_nogpc 分发）、
> ARMMMUIdx 索引系统（E10_0/E10_1/E10_1_PAN/E20_0/E20_2/E2/E3 Stage1 索引 22 个+Stage2/Stage2_S+
> Phys_S/NS/Root/Realm 物理+Stage1_E0/E1 无 TLB 仅遍历、ARM_MMU_IDX_A=0x20 编码）、
> Stage-1 LPAE 遍历（get_phys_addr_lpae：aa64_va_parameters→T0SZ/T1SZ/TG 解码、
> regime_ttbr→TTBR0/1 选择、L0-L3 四级遍历、描述符解析 bit[1:0] valid+table/block/page、
> tableattrs 五位向下传播、AF 标志检查+硬件管理、descaddr 物理地址拼接+页大小计算、
> 输出 out_space/attrs/prot）、
> Stage-2 翻译（get_phys_addr_twostage：S1→IPA→S2 串联、ptw_idx_for_stage_2 物理索引选择、
> S1 prot/cacheattrs 保存→S2 遍历→result.prot = s1_prot & s2prot 权限合并、页大小取大值）、
> 权限检查（get_S1prot：PAN/EPAN 数据访问禁止、SCR_SIF 安全指令取指禁止、
> WXN/UWXN/PXN/XN 执行禁止、Root/Realm 跨安全域指令取指禁止、
> get_S2prot：S2AP bit[1:0]→R/W、XN 四值+TTS2UXN EL0/EL1 分离执行控制、
> get_S2prot_indirect：S2PIR_EL2 间接权限 16 条目×3 列表）、
> 故障处理（ARMFaultType 24 种故障类型 Translation/AccessFlag/Permission/AddressSize/Debug 等、
> ARMMMUFaultInfo 结构 type/level/stage2/s1ptw/s1ns/ea/domain 11 字段、
> arm_fi_to_lfsc 长格式 FSC 编码、arm_fi_to_sfsc 短格式 FSC 编码）、
> 安全属性传播（in_space→cur_space NSTable 降级、result.attrs.secure/space 最终标记、
> S1 Secure 可降级为 NS 但不能升级）、
> TLB 交互（tlb_set_page_full 结果写入 TLB、TLBI 按 mmuidx/ASID/VMID 失效、
> vae1_tlbmask/alle1_tlbmask 掩码构建）、
> AT 地址翻译指令（do_ats_write→get_phys_addr_for_at→PAR_EL1 结果格式化）。

---

## 目录

1. [架构概述](#1-架构概述)
2. [ARMMMUIdx 索引系统](#2-armmmuidx-索引系统)
3. [S1Translate 遍历上下文](#3-s1translate-遍历上下文)
4. [get_phys_addr 入口与分发](#4-get_phys_addr-入口与分发)
5. [Stage-1 LPAE 遍历](#5-stage-1-lpae-遍历)
6. [描述符解析](#6-描述符解析)
7. [Stage-2 翻译](#7-stage-2-翻译)
8. [Stage-1 权限检查 (get_S1prot)](#8-stage-1-权限检查-get_s1prot)
9. [Stage-2 权限检查 (get_S2prot)](#9-stage-2-权限检查-get_s2prot)
10. [故障类型与 ARMMMUFaultInfo](#10-故障类型与-armmmufaultinfo)
11. [安全属性传播](#11-安全属性传播)
12. [TLB 交互](#12-tlb-交互)
13. [AT 地址翻译指令](#13-at-地址翻译指令)
14. [ARMCacheAttrs 与缓存属性](#14-armcacheattrs-与缓存属性)
15. [完整 Stage-1+2 遍历流程总结](#15-完整-stage-12-遍历流程总结)

---

## 1. 架构概述

QEMU ARM64 页表遍历实现于 `target/arm/ptw.c`（约 4000 行），支持 AArch64 LPAE（4KB/16KB/64KB 粒度）和 AArch32 多种格式。核心流程：

```
get_phys_addr(env, VA, access_type, mmu_idx)
  → S1Translate 初始化（安全空间、MMU 索引）
  → get_phys_addr_gpc → get_phys_addr_nogpc
    ├── 物理索引 → get_phys_addr_disabled（直通）
    ├── Stage1+2 → get_phys_addr_twostage
    │     ├── S1: get_phys_addr_nogpc → get_phys_addr_lpae → IPA
    │     └── S2: get_phys_addr_nogpc → get_phys_addr_lpae → PA
    │     → prot = s1_prot & s2prot
    └── Stage1-only → get_phys_addr_lpae → PA
  → GetPhysAddrResult（phys_addr, prot, attrs, cacheattrs）
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `target/arm/ptw.c` | 22-90 | S1Translate 结构定义 |
| `target/arm/ptw.c` | 1343-1420 | get_S2prot / get_S2prot_indirect |
| `target/arm/ptw.c` | 1434-1541 | get_S1prot |
| `target/arm/ptw.c` | 1843-2449 | get_phys_addr_lpae（LPAE 核心遍历） |
| `target/arm/ptw.c` | 3554-3658 | get_phys_addr_twostage（S1+S2 串联） |
| `target/arm/ptw.c` | 3661-3798 | get_phys_addr_nogpc（主分发） |
| `target/arm/ptw.c` | 3931-3943 | get_phys_addr 公共入口 |
| `target/arm/mmuidx.h` | 137-198 | ARMMMUIdx 枚举 |
| `target/arm/internals.h` | 705-771 | ARMFaultType / ARMMMUFaultInfo |
| `target/arm/internals.h` | 779-957 | arm_fi_to_sfsc / arm_fi_to_lfsc |
| `target/arm/internals.h` | 1465-1474 | ARMCacheAttrs |
| `target/arm/tcg/cpregs-at.c` | 26-259 | do_ats_write / ats_write |

---

## 2. ARMMMUIdx 索引系统

```c
// target/arm/mmuidx.h:137-198
typedef enum ARMMMUIdx {
    // === A-profile Stage-1 有 TLB ===
    ARMMMUIdx_E10_0      = 0 | 0x20,   // EL1&0 regime, EL0
    ARMMMUIdx_E10_0_GCS  = 1 | 0x20,   // EL0 + GCS
    ARMMMUIdx_E10_1      = 2 | 0x20,   // EL1
    ARMMMUIdx_E10_1_PAN  = 3 | 0x20,   // EL1 + PAN
    ARMMMUIdx_E10_1_GCS  = 4 | 0x20,   // EL1 + GCS

    ARMMMUIdx_E20_0      = 5 | 0x20,   // EL2&0 regime (VHE), EL0
    ARMMMUIdx_E20_2      = 7 | 0x20,   // EL2&0 regime, EL2
    ARMMMUIdx_E20_2_PAN  = 8 | 0x20,   // EL2 + PAN

    ARMMMUIdx_E2         = 10 | 0x20,  // EL2 (non-VHE)
    ARMMMUIdx_E3         = 12 | 0x20,  // EL3

    // === Stage-2 ===
    ARMMMUIdx_Stage2_S   = 16 | 0x20,  // Secure Stage-2
    ARMMMUIdx_Stage2     = 17 | 0x20,  // Non-Secure Stage-2

    // === 物理直通 ===
    ARMMMUIdx_Phys_S     = 18 | 0x20,  // Secure 物理
    ARMMMUIdx_Phys_NS    = 19 | 0x20,  // NS 物理
    ARMMMUIdx_Phys_Root  = 20 | 0x20,  // Root 物理（RME）
    ARMMMUIdx_Phys_Realm = 21 | 0x20,  // Realm 物理

    // === 无 TLB（仅用于 S1 遍历/AT 指令） ===
    ARMMMUIdx_Stage1_E0     = 0 | 0x40, // S1+S2 拆分的 S1 EL0
    ARMMMUIdx_Stage1_E1     = 1 | 0x40, // S1 EL1
    ARMMMUIdx_Stage1_E1_PAN = 2 | 0x40, // S1 EL1+PAN
} ARMMMUIdx;
```

**编码规则**：低 5 位为 core TLB index，高位标识 A-profile(`0x20`) / M-profile(`0x80`) / NOTLB(`0x40`)。

---

## 3. S1Translate 遍历上下文

```c
// target/arm/ptw.c:22-90
typedef struct S1Translate {
    ARMMMUIdx in_mmu_idx;      // 27: 哪个 TTBR/TCR/翻译 regime
    ARMMMUIdx in_ptw_idx;      // 35: 描述符加载用的 MMU 索引（Stage2 或 Phys）
    ARMSecuritySpace in_space; // 52: 架构翻译 regime 的安全空间
    ARMSecuritySpace cur_space;// 57: 当前安全空间（可被 NSTable 降级）
    bool in_debug;             // 63: QEMU 调试访问（不更新 AF）
    bool in_at;                // 69: AT 指令或调试
    bool in_s1_is_el0;         // 75: S2 遍历时 S1 是否 EL0
    uint8_t in_prot_check;     // 81: PAGE_READ/WRITE/EXEC 权限掩码
    bool in_nv1;               // 83: NV1 缓存
    bool out_rw;               // 84: 输出读写属性
    bool out_be;               // 85: 输出大端
    ARMSecuritySpace out_space;// 86: 输出安全空间
    hwaddr out_virt;           // 87: 输出虚拟地址
    hwaddr out_phys;           // 88: 输出物理地址
    void *out_host;            // 89: 输出 host 地址
} S1Translate;
```

---

## 4. get_phys_addr 入口与分发

### 公共入口

```c
// target/arm/ptw.c:3931-3943
bool get_phys_addr(CPUARMState *env, vaddr address,
                   MMUAccessType access_type, MemOp memop,
                   ARMMMUIdx mmu_idx,
                   GetPhysAddrResult *result, ARMMMUFaultInfo *fi)
{
    S1Translate ptw = {
        .in_mmu_idx = mmu_idx,
        .in_space = arm_mmu_idx_to_security_space(env, mmu_idx),  // 3937
        .in_prot_check = 1 << access_type,                        // 3938
    };
    return get_phys_addr_gpc(env, &ptw, address, access_type,
                             memop, result, fi);
}
```

### get_phys_addr_nogpc 分发

```c
// target/arm/ptw.c:3661-3798
static bool get_phys_addr_nogpc(env, ptw, address, ...)
{
    ptw->cur_space = ptw->in_space;                         // 3675
    result->f.attrs.space = ptw->in_space;                  // 3676
    result->f.attrs.secure = arm_space_is_secure(in_space); // 3677

    switch (mmu_idx) {
    case Phys_S/NS/Root/Realm:                               // 3680-3686
        return get_phys_addr_disabled();  // 物理直通

    case Stage1_E0/E1/E1_PAN:                               // 3688-3697
        ptw->in_ptw_idx = Stage2/Stage2_S;  // S1 描述符通过 S2 加载
        break;

    case Stage2/Stage2_S:                                    // 3699-3707
        ptw->in_ptw_idx = ptw_idx_for_stage_2();  // S2 用物理索引
        break;

    case E10_0:                                              // 3709+
        s1_mmu_idx = Stage1_E0;
        → get_phys_addr_twostage()  // S1+S2 串联
    case E10_1/E10_1_PAN:
        → get_phys_addr_twostage()
    case E20_*/E2/E3:
        → get_phys_addr_lpae() 或其他  // 仅 S1
    }
}
```

---

## 5. Stage-1 LPAE 遍历

```c
// target/arm/ptw.c:1843-2449
static bool get_phys_addr_lpae(env, ptw, address, access_type, ...)
{
    // 1. 参数解析
    ARMVAParameters param = aa64_va_parameters(env, address, mmu_idx, ...);
    // → T0SZ/T1SZ, TG0/TG1, select(VA 高位选 TTBR0/1)            1888-1890

    // 2. TTBR 选择
    ttbr = regime_ttbr(env, mmu_idx, param.select);                // 1964-1973
    // select=0 → TTBR0, select=1 → TTBR1

    // 3. 起始级别与步长计算
    // 4KB: stride=9, L0-L3; 16KB: stride=11; 64KB: stride=13
    stride = arm_granule_stride(param.gran);                       // 1987-2009
    inputsize = 64 - param.tsz;  // VA 有效位数
    level = 起始级别（基于 inputsize 和 stride 计算）

    // 4. 遍历循环
    while (level < 3) {
        descaddr = ttbr | (VA[level 对应位] << 3);
        descriptor = arm_ldq_ptw(env, ptw, fi);                    // 2082
        if (!(descriptor & 1)) → Translation fault                 // 2089
        if (descriptor & 2 && level < 3) {
            // Table 描述符 → 下一级
            tableattrs |= descriptor[63:59];                        // 2122
            level++;  goto next_level;                              // 2123-2125
        }
        break;  // Block/Page 描述符
    }

    // 5. 最终物理地址
    page_size = 1 << (stride * (4 - level) + 3);                   // 2139
    descaddr = (descriptor & mask) | (address & (page_size - 1));  // 2140-2141

    // 6. Access Flag 检查
    if (!(descriptor & AF) && !param.ha) → AccessFlag fault        // 2145

    // 7. 权限检查
    ap = extract64(descriptor, 6, 2);                               // AP 位
    prot = get_S1prot(env, mmu_idx, ...);                          // S1 权限

    // 8. 结果输出
    result->f.phys_addr = descaddr;
    result->f.prot = prot;
    result->f.lg_page_size = ctz64(page_size);
}
```

### 粒度与级别

| 粒度 | stride | 起始级别 | 页大小 | 块大小 |
|------|--------|---------|--------|--------|
| 4KB | 9 | L0 | 4KB (L3) | 2MB (L2), 1GB (L1) |
| 16KB | 11 | L0/L1 | 16KB (L3) | 32MB (L2) |
| 64KB | 13 | L1/L2 | 64KB (L3) | 512MB (L2) |

---

## 6. 描述符解析

```
AArch64 LPAE 描述符格式 (64-bit):

Table (level 0-2):
  [63:59] 属性位（NSTable, APTable, UXNTable, PXNTable）
  [47:12] 下一级表物理地址
  [1]     = 1 (table)
  [0]     = 1 (valid)

Block (level 1-2) / Page (level 3):
  [54]    UXN (Unprivileged Execute-Never)
  [53]    PXN (Privileged Execute-Never)
  [52]    Contiguous hint
  [51]    DBM (Dirty Bit Modifier)
  [47:12] 输出物理地址
  [11]    nG (non-Global)
  [10]    AF (Access Flag)
  [9:8]   SH (Shareability)
  [7:6]   AP (Access Permission)
  [5]     NS (Non-Secure, Stage-1 only)
  [4:2]   AttrIndx (MAIR 索引)
  [1]     Block=0 / Page=1
  [0]     = 1 (valid)
```

```c
// ptw.c:2089-2126 — 描述符有效性检查
if (!(descriptor & 1))           → 无效 → Translation fault
if (!(descriptor & 2) && level<3) → Block 描述符（须在有效级别）
if ((descriptor & 2) && level<3)  → Table → tableattrs |= [63:59], level++
if ((descriptor & 2) && level==3) → Page 描述符（L3 最终）

// ptw.c:2115-2126 — Table 属性向下传播
tableattrs |= extract64(descriptor, 59, 5);
// NSTable: 降级安全状态
// APTable: 限制下层权限
// UXNTable/PXNTable: 强制执行禁止
```

---

## 7. Stage-2 翻译

```c
// target/arm/ptw.c:3554-3658
static bool get_phys_addr_twostage(env, ptw, address, ...)
{
    // === Stage-1: VA → IPA ===
    ret = get_phys_addr_nogpc(env, ptw, address, ...);              // 3568
    if (ret) return ret;  // S1 失败                                // 3572

    ipa = result->f.phys_addr;                                      // 3576
    s1_prot = result->f.prot;                                       // 3589
    cacheattrs1 = result->cacheattrs;                               // 3592

    // === 切换到 Stage-2: IPA → PA ===
    ptw->in_s1_is_el0 = (原 mmu_idx == Stage1_E0);                 // 3580
    ptw->in_mmu_idx = ipa_secure ? Stage2_S : Stage2;              // 3581
    ptw->in_space = ipa_space;                                      // 3582
    ptw->in_ptw_idx = ptw_idx_for_stage_2(env, mmu_idx);           // 3583

    // === Stage-2 遍历 ===
    ret = get_phys_addr_nogpc(env, ptw, ipa, ...);                  // 3595
    fi->s2addr = ipa;                                               // 3597

    // === 合并权限 ===
    result->f.prot = s1_prot & result->s2prot;                     // 3600
    // S1 和 S2 权限取交集

    // === 合并页大小 ===
    result->f.lg_page_size = MAX(s1_lgpgsz, result->f.lg_page_size); // 3617

    // === 合并缓存属性 ===
    combine_cacheattrs(result, cacheattrs1);                        // 3642
}
```

---

## 8. Stage-1 权限检查 (get_S1prot)

```c
// target/arm/ptw.c:1434-1541
static int get_S1prot(env, mmu_idx, is_aa64, user_rw, prot_rw,
                      xn, pxn, in_pa, out_pa)
{
    bool is_user = regime_is_user(mmu_idx);                        // 1439

    // === 1. 用户/特权选择 ===
    if (is_user)
        prot_rw = user_rw;                                         // 1446

    // === 2. PAN 处理 ===
    if (user_rw && regime_is_pan(mmu_idx))
        prot_rw = 0;  // PAN: 禁止 EL1 访问 EL0 可访问数据         // 1456-1457
    // EPAN (PAN3): 即使 EL0 仅有执行权限也禁止                     // 1458-1462

    // === 3. 安全域指令取指限制 ===
    if (in_pa != out_pa) {
        Root → 非 Root: 禁止执行                                   // 1467-1472
        Realm → 非 Realm: 禁止执行 (EL2 regime)                    // 1473-1488
        Secure + SCR_SIF: 禁止从 NS 执行                           // 1490-1493
    }

    // === 4. WXN/PXN/XN ===
    wxn = SCTLR.WXN;                                              // 1508
    if (AArch64 && 双范围 && !user)
        xn = pxn || (user_rw & PAGE_WRITE);                       // 1513
    if (xn || (wxn && prot_rw & PAGE_WRITE))
        return prot_rw;  // 不可执行                               // 1537-1538
    return prot_rw | PAGE_EXEC;                                    // 1540
}
```

### AP 位解码（AArch64 Stage-1）

| AP[2:1] | EL1 | EL0 | 含义 |
|---------|-----|-----|------|
| 00 | RW | None | 特权读写 |
| 01 | RW | RW | 全访问 |
| 10 | RO | None | 特权只读 |
| 11 | RO | RO | 全只读 |

---

## 9. Stage-2 权限检查 (get_S2prot)

```c
// target/arm/ptw.c:1343-1382
static int get_S2prot(env, s2ap, xn, s1_is_el0)
{
    // S2AP 位直接映射
    if (s2ap & 1) prot |= PAGE_READ;                              // 1347
    if (s2ap & 2) prot |= PAGE_WRITE;                             // 1350

    // XN 处理（支持 TTS2UXN 时四值）
    if (any_tts2uxn) {
        xn=0: EXEC both                                            // 1356-1357
        xn=1: EXEC EL0 only                                       // 1360-1362
        xn=2: no EXEC                                              // 1364-1365
        xn=3: EXEC EL1 only                                       // 1367-1369
    } else {
        !xn[1]: EXEC if readable                                   // 1375-1379
    }
    return prot;
}
```

### 间接权限（FEAT_S2PIE）

```c
// ptw.c:1384-1420
static int get_S2prot_indirect(env, result, pi_index, po_index, s1_is_el0)
{
    // S2PIR_EL2 寄存器的 4 位 per-index 编码
    // 16 种权限组合 × 3 列（priv, unpriv, ttw）
    // perm_table[16][3] 查表                                      1388-1413
    result->f.prot = perm_table[s2pi][2];  // TTW 权限
    return perm_table[s2pi][s1_is_el0];     // 数据权限
}
```

---

## 10. 故障类型与 ARMMMUFaultInfo

### ARMFaultType

```c
// target/arm/internals.h:705-730
typedef enum ARMFaultType {
    ARMFault_None,                  // 无故障
    ARMFault_AccessFlag,            // AF 未设置
    ARMFault_Alignment,             // 对齐故障
    ARMFault_Background,            // MPU 背景故障
    ARMFault_Domain,                // 域故障（AArch32 短格式）
    ARMFault_Permission,            // 权限故障
    ARMFault_Translation,           // 翻译故障（无映射）
    ARMFault_AddressSize,           // 地址大小故障
    ARMFault_SyncExternal,          // 同步外部中止
    ARMFault_SyncExternalOnWalk,    // 遍历中外部中止
    ARMFault_SyncParity,            // 奇偶校验
    ARMFault_SyncParityOnWalk,      // 遍历中奇偶校验
    ARMFault_AsyncParity,           // 异步奇偶校验
    ARMFault_AsyncExternal,         // 异步外部
    ARMFault_Debug,                 // 调试异常
    ARMFault_TLBConflict,           // TLB 冲突
    ARMFault_UnsuppAtomicUpdate,    // 不支持的原子更新
    ARMFault_Lockdown,              // 锁定
    ARMFault_Exclusive,             // 独占故障
    ARMFault_ICacheMaint,           // 指令缓存维护
    ARMFault_GPCFOnWalk,            // GPC 遍历故障
    ARMFault_GPCFOnOutput,          // GPC 输出故障
} ARMFaultType;
```

### ARMMMUFaultInfo

```c
// target/arm/internals.h:758-771
struct ARMMMUFaultInfo {
    ARMFaultType type;              // 759: 故障类型
    ARMGPCF gpcf;                   // 760: GPC 子类型
    hwaddr s2addr;                  // 761: S2 故障的 IPA
    hwaddr paddr;                   // 762: GPC 故障物理地址
    ARMSecuritySpace paddr_space;   // 763: 故障物理地址空间
    int level;                      // 764: 页表遍历级别 (0-3)
    int domain;                     // 765: 域号（AArch32 短格式）
    bool stage2;                    // 766: 是否 S2 故障
    bool s1ptw;                     // 767: S2 故障发生在 S1 遍历期间
    bool s1ns;                      // 768: 故障 IPA 是否 NS
    bool ea;                        // 769: 外部中止类型位
    bool dirtybit;                  // 770: FEAT_S1PIE/S2PIE dirty bit
};
```

### FSC 编码

```c
// internals.h:779-957
// arm_fi_to_sfsc(): 短格式故障状态码（AArch32 非 LPAE）
// arm_fi_to_lfsc(): 长格式故障状态码（AArch64 / AArch32 LPAE）
//
// 长格式示例:
//   Translation fault level N → 0x04 | level
//   AccessFlag fault level N → 0x08 | level
//   Permission fault level N → 0x0C | level
//   AddressSize fault level N → 0x00 | level
```

---

## 11. 安全属性传播

```c
// ptw.c:3661-3677 — 初始化
ptw->cur_space = ptw->in_space;
result->f.attrs.space = ptw->in_space;
result->f.attrs.secure = arm_space_is_secure(ptw->in_space);

// 遍历期间:
// NSTable 位（Table 描述符 bit[63]）可将 Secure 降级为 NonSecure
// 但不能将 NonSecure 升级为 Secure

// ptw.c:3857-3918 — arm_mmu_idx_to_security_space
// EL1/EL2 → arm_security_space_below_el3(env)
// EL3 → Secure (或 Root if RME)
// Stage2 → NS（Secure EL2 特殊降级）
// Stage2_S → Secure
// Phys_S/NS/Root/Realm → 对应空间
```

---

## 12. TLB 交互

### PTW 结果写入 TLB

```c
// accel/tcg/cputlb.c:1024-1183
// tlb_set_page_full() 将 PTW 结果存入 TLB:
//   → fulltlb[index] = GetPhysAddrResult
//   → 设置快速 TLB 条目（addend 计算）
//   → victim TLB 溢出处理
```

### TLBI 失效

```c
// target/arm/tcg/tlb-insns.c:224-260
// vae1_tlbmask(): EL1&0 regime 的 TLB 掩码
// → E10_0 | E10_0_GCS | E10_1 | E10_1_PAN | E10_1_GCS

// tlb-insns.c:167-183
// tlbiipas2_hyp_write(): Stage2 TLB 失效
// → ARMMMUIdxBit_Stage2

// helper.c:404-420
// alle1_tlbmask(): "ALL EL1" 包含 S1 和 S2
// → E10_* | Stage2 | Stage2_S
```

---

## 13. AT 地址翻译指令

```c
// target/arm/tcg/cpregs-at.c:26-259
// do_ats_write(): AT 指令核心
//   1. 构建 S1Translate（in_at=true, in_debug=false）
//   2. get_phys_addr_for_at() → 地址翻译
//   3. 成功: 格式化 PAR_EL1
//      → PA[47:12], NS, ATTR(MAIR), SH
//   4. 失败: PAR_EL1.F=1 + FSC 编码

// ats_write(): AT S1E1R/W, AT S12E1R/W 分发
//   AT S1E1R → Stage1 only
//   AT S12E1R → Stage1 + Stage2

// PAR_EL1 格式（成功）:
//   [63:56] ATTR (MAIR attributes)
//   [9]     NS (Non-Secure)
//   [8:7]   SH (Shareability)
//   [47:12] PA (Physical Address)
//   [0]     F = 0 (success)
```

---

## 14. ARMCacheAttrs 与缓存属性

```c
// target/arm/internals.h:1465-1474
typedef struct ARMCacheAttrs {
    unsigned int attrs:8;          // MAIR 8 位格式（或 S2 描述符 [5:2]）
    unsigned int shareability:2;   // SH 字段
    bool is_s2_format:1;           // S2 格式标志
} ARMCacheAttrs;
```

S1+S2 合并时，S1 和 S2 的缓存属性取"更弱"的一个（如 S1 Cacheable + S2 Non-cacheable = Non-cacheable）。

---

## 15. 完整 Stage-1+2 遍历流程总结

```
Guest: LDR X0, [X1]    (VA = 0x0000_1000_0000)

=== get_phys_addr 入口 ===
  mmu_idx = E10_0 (EL0, EL1&0 regime)
  in_space = ARMSS_NonSecure
  → get_phys_addr_twostage()

=== Stage-1: VA → IPA ===
  get_phys_addr_lpae:
    param = aa64_va_parameters(): T0SZ=16, TG=4KB, select=0(TTBR0)
    ttbr = TTBR0_EL1
    ① L0: ttbr + VA[47:39]*8 → 读 L0 描述符 → Table → L1 基址
    ② L1: L1基址 + VA[38:30]*8 → 读 L1 描述符 → Block(1GB) 或 Table → L2
    ③ L2: L2基址 + VA[29:21]*8 → 读 L2 描述符 → Block(2MB) 或 Table → L3
    ④ L3: L3基址 + VA[20:12]*8 → 读 L3 描述符 → Page(4KB)
    AF 检查 → get_S1prot(AP, XN, PXN, PAN, WXN)
    result: IPA = 0x4000_1000, prot = RW|EXEC

=== Stage-2: IPA → PA ===
  ptw.in_mmu_idx = Stage2
  ptw.in_ptw_idx = Phys_NS
  get_phys_addr_lpae:
    tcr = VTCR_EL2, ttbr = VTTBR_EL2
    遍历 IPA → PA
    get_S2prot(S2AP, XN, s1_is_el0)
    result: PA = 0x8000_1000, s2prot = RW|EXEC

=== 合并 ===
  prot = s1_prot & s2prot = RW|EXEC
  page_size = MAX(s1_page, s2_page) = 4KB
  attrs.secure = false (NS)
  → tlb_set_page_full() → TLB 缓存

=== 后续访问 ===
  TLB 命中 → 直接 host 地址访问
```

---

## 交叉参考

- [48-ARM64-安全扩展TrustZone深度分析](48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md) — SCR_NS 安全状态切换与 TLB 刷新
- [43-ARM64-TCG-softmmu-TLB深度分析](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — TLB 数据结构与快慢路径
- [47-ARM64-系统寄存器与CP访问深度分析](47-ARM64-系统寄存器与CP访问深度分析-ARMCPRegInfo框架-MRS-MSR翻译-cpregs哈希表-EL银行与访问控制.md) — SCTLR/TCR/TTBR 寄存器定义

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
