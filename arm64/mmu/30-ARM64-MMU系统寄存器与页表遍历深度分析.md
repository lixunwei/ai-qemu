# ARM64 MMU 系统寄存器与页表遍历深度分析

## 1. 概述

ARM64 MMU 的核心由三组系统寄存器控制：TCR（翻译控制）、TTBR（翻译表基址）、MAIR（内存属性）。QEMU 在 `target/arm/ptw.c` 中实现了完整的两级（Stage-1 + Stage-2）页表遍历，支持 4KB/16KB/64KB 页粒度和最多 52 位物理地址（FEAT_LPA）。本文深入分析这些寄存器的位定义、写副作用以及页表遍历的完整路径。

**关键源文件：**
- `target/arm/ptw.c` — 页表遍历核心（~2700行）
- `target/arm/helper.c` — TCR/TTBR/MAIR 寄存器定义与写处理
- `target/arm/internals.h` — VTCR 位域定义
- `target/arm/tcg/tlb-insns.c` — TLBI 指令实现

---

## 2. TCR_EL1 — 翻译控制寄存器

### 2.1 关键位域

```
TCR_EL1 (64位):
  ┌─────────┬──────┬──────────────────────────────────┐
  │ 位域     │ 位置  │ 功能                              │
  ├─────────┼──────┼──────────────────────────────────┤
  │ T0SZ    │ [5:0] │ TTBR0 地址空间大小 (VA=64-T0SZ)   │
  │ EPD0    │ [7]   │ 禁用 TTBR0 翻译（直接 fault）      │
  │ IRGN0   │ [9:8] │ TTBR0 内部缓存属性                 │
  │ ORGN0   │[11:10]│ TTBR0 外部缓存属性                 │
  │ SH0     │[13:12]│ TTBR0 共享属性                     │
  │ TG0     │[15:14]│ TTBR0 颗粒大小 (00=4K,01=64K,10=16K)│
  │ T1SZ    │[21:16]│ TTBR1 地址空间大小 (VA=64-T1SZ)   │
  │ A1      │ [22]  │ ASID 选择 (0=TTBR0, 1=TTBR1)     │
  │ EPD1    │ [23]  │ 禁用 TTBR1 翻译                   │
  │ IRGN1   │[25:24]│ TTBR1 内部缓存属性                 │
  │ ORGN1   │[27:26]│ TTBR1 外部缓存属性                 │
  │ SH1     │[29:28]│ TTBR1 共享属性                     │
  │ TG1     │[31:30]│ TTBR1 颗粒大小 (01=16K,10=4K,11=64K)│
  │ IPS/PS  │[34:32]│ 物理地址大小 (0=32,5=48,6=52)     │
  │ DS      │ [59]  │ 52位输出地址（FEAT_LPA2）           │
  └─────────┴──────┴──────────────────────────────────┘
```

### 2.2 写副作用

```c
// helper.c:2763-2770 — vmsa_tcr_el12_write()
// A1 位变化可能改变 ASID 来源 → 必须 TLB 全刷新
static void vmsa_tcr_el12_write(CPUARMState *env, const ARMCPRegInfo *ri,
                                uint64_t value)
{
    ARMCPU *cpu = env_archcpu(env);
    tlb_flush(CPU(cpu));    // 无条件刷新
    raw_write(env, ri, value);
}
```

### 2.3 PTW 如何使用 TCR

```c
// ptw.c:1875 — 读取 TCR
uint64_t tcr = regime_tcr(env, mmu_idx);

// ptw.c:1888-1890 — 解码 TCR 参数
param = aa64_va_parameters(env, address, mmu_idx, ...);
// aa64_va_parameters() 从 TCR 提取:
//   T0SZ/T1SZ → inputsize (有效虚拟地址位数)
//   TG0/TG1  → gran (颗粒大小 → stride)
//   EPD0/EPD1 → epd (翻译禁用标志)
//   A1 → select (ASID/TTBR 选择)

// ptw.c:1962 — 颗粒大小决定步长
stride = arm_granule_bits(param.gran) - 3;
// 4KB → stride=9, 16KB → stride=11, 64KB → stride=13
```

---

## 3. TTBR0/TTBR1_EL1 — 翻译表基址寄存器

### 3.1 布局

```
TTBR0/1_EL1 (64位):
  ┌─────────┬──────┬──────────────────────────────────┐
  │ ASID    │[63:48]│ 地址空间标识符（16位）              │
  │ BADDR   │[47:1] │ 翻译表基地址（物理地址）            │
  │ CnP     │ [0]   │ 通用非私有位（多核共享页表）         │
  └─────────┴──────┴──────────────────────────────────┘
  FEAT_LPA: bits[5:2] 存储 BADDR[51:48]
```

### 3.2 写副作用

```c
// helper.c:2773-2783 — vmsa_ttbr_write()
static void vmsa_ttbr_write(CPUARMState *env, const ARMCPRegInfo *ri,
                            uint64_t value)
{
    // 仅当 ASID 字段变化时才刷新 TLB
    if (cpreg_field_type(ri) == MO_64 &&
        extract64(raw_read(env, ri) ^ value, 48, 16) != 0) {
        ARMCPU *cpu = env_archcpu(env);
        tlb_flush(CPU(cpu));     // ASID 变 → 全刷
    }
    raw_write(env, ri, value);   // BADDR 变但 ASID 不变 → 不刷
}
// 优化: 页表基址变更不刷 TLB（QEMU 不跟踪 ASID）
```

### 3.3 PTW 提取基地址

```c
// ptw.c:1972 — 选择 TTBR
ttbr = regime_ttbr(env, mmu_idx, param.select);
// param.select: 0 → TTBR0, 1 → TTBR1

// ptw.c:2014-2026 — 提取基地址
descaddr = extract64(ttbr, 0, 48);       // 标准 48 位
if (outputsize > 48) {
    descaddr |= extract64(ttbr, 2, 4) << 48;  // FEAT_LPA 52位
}
```

---

## 4. MAIR_EL1 — 内存属性间接寄存器

### 4.1 结构

```
MAIR_EL1 (64位) = 8 × 8位属性条目:
  Attr[n] = MAIR_EL1[(n*8+7):(n*8)]    n = 0..7

每个 Attr 编码:
  0x00         → Device-nGnRnE（最严格设备内存）
  0x04         → Device-nGnRE
  0x08         → Device-nGRE
  0x0C         → Device-GRE
  0x44         → Normal Non-Cacheable
  0xBB         → Normal Write-Back Read/Write-Allocate
  0xFF         → Normal Write-Back R/W-Allocate（典型内存）
```

### 4.2 PTW 属性解析

```c
// 页表项 (PTE) 低位包含 AttrIndx[2:0]
// PTW 流程:
//   1. 从 PTE 提取 AttrIndx
attrindx = extract32(attrs, 2, 3);

//   2. 用 AttrIndx 索引 MAIR 获取属性字节
mair = env->cp15.mair_el[regime_el(mmu_idx)];
uint8_t attr = extract64(mair, attrindx * 8, 8);

//   3. 根据 attr 高 4 位判断 Device/Normal
//      高4位=0 → Device，否则 Normal
```

---

## 5. TCR_EL2 与 VTCR_EL2 — 虚拟化翻译控制

### 5.1 VTCR_EL2 位域

```c
// internals.h:267+ — VTCR 位域定义
FIELD(VTCR, T0SZ, 0, 6)    // Stage-2 地址大小
FIELD(VTCR, SL0, 6, 2)     // 起始查找级别
FIELD(VTCR, IRGN0, 8, 2)   // 内部缓存属性
FIELD(VTCR, ORGN0, 10, 2)  // 外部缓存属性
FIELD(VTCR, SH0, 12, 2)    // 共享属性
FIELD(VTCR, TG0, 14, 2)    // 颗粒大小
FIELD(VTCR, PS, 16, 3)     // 物理地址大小
FIELD(VTCR, VS, 19, 1)     // VMID 大小 (0=8位, 1=16位)
FIELD(VTCR, HA, 21, 1)     // 硬件脏位
FIELD(VTCR, HD, 22, 1)     // 硬件脏位
FIELD(VTCR, DS, 32, 1)     // 52 位输出地址
FIELD(VTCR, SL2, 33, 1)    // SL0 扩展（4KB+DS）
```

### 5.2 Stage-2 起始级别

```c
// ptw.c:1727-1820 — check_s2_mmu_setup()
// SL0 + 颗粒大小 → 起始查找级别:
//
// 4KB 颗粒 (stride=9):
//   SL0=0 → level 2, SL0=1 → level 1, SL0=2 → level 0
//   DS=1 + SL2=1 → level -1（concatenated）
//
// 16KB 颗粒 (stride=11):
//   SL0=0 → level 3, SL0=1 → level 2, SL0=2 → level 1
//
// 64KB 颗粒 (stride=13):
//   SL0=0 → level 3, SL0=1 → level 2, SL0=2 → level 1
```

### 5.3 TCR/TTBR EL2 写（VHE 模式）

```c
// helper.c:2785-2799 — vmsa_tcr_ttbr_el2_write()
// E2H 使能时 EL2 有两段地址空间（像 EL1）
// ASID 变化时仅刷 EL2 相关 TLB:
if (extract64(old ^ value, 48, 16) && (arm_hcr_el2_eff(env) & HCR_E2H)) {
    tlb_flush_by_mmuidx(env_cpu(env), alle2_tlbmask());
}
```

---

## 6. 页表遍历完整流程

### 6.1 get_phys_addr_lpae()

```c
// ptw.c:1859-2140+ — 核心遍历函数
get_phys_addr_lpae(env, ptw, address, access_type, memop, result, fi)
{
    // 1. 读取 TCR，确定翻译体制
    tcr = regime_tcr(env, mmu_idx);
    
    // 2. 解码翻译参数
    param = aa64_va_parameters(env, address, mmu_idx, ...);
    // → inputsize, gran, select, epd, tbi, ...
    
    // 3. 计算步长
    stride = arm_granule_bits(param.gran) - 3;
    
    // 4. 选择 TTBR 并提取基地址
    ttbr = regime_ttbr(env, mmu_idx, param.select);
    descaddr = extract64(ttbr, 0, 48);
    
    // 5. EPD 检查（翻译禁用 → Translation Fault）
    if (param.epd) goto do_translation_fault;
    
    // 6. 计算起始级别
    //    Stage-1: level = 4 - (inputsize - 4) / stride
    //    Stage-2: check_s2_mmu_setup()
    
    // 7. 逐级遍历
    for (;;) {
        // 读取描述符（可能触发 Stage-2 翻译）
        descriptor = arm_ldq_ptw(env, ptw, descaddr, fi);
        
        // 描述符类型判断:
        if (!(descriptor & 1)) → 无效 → Translation Fault
        if (level < 3 && (descriptor & 2)) {
            // Table 描述符 → 提取下一级表地址
            descaddr = descriptor & table_addr_mask;
            // 应用 APTable/PXNTable/UXNTable 限制
            level++;
            continue;
        }
        // Block/Page 描述符 → 输出地址
        break;
    }
    
    // 8. 权限检查 (AP, PXN, UXN)
    // 9. 属性解析 (AttrIndx → MAIR → Device/Normal)
    // 10. 返回物理地址和属性
}
```

### 6.2 两级翻译（Stage-1 + Stage-2）

```
Guest VA → [Stage-1 Walk] → IPA → [Stage-2 Walk] → PA

get_phys_addr_twostage():
  1. Stage-1: get_phys_addr_lpae(TCR_EL1, TTBR_EL1)
     → 使用 EL1 翻译体制
     → 输出 IPA (中间物理地址)
  
  2. Stage-2: get_phys_addr_lpae(VTCR_EL2, VTTBR_EL2)
     → 使用 EL2 虚拟化翻译体制
     → 输出 PA (物理地址)
  
  3. 权限合并: S1 ∩ S2 权限
  4. 属性合并: S1 属性受 S2 限制
```

### 6.3 翻译体制选择

```c
// ptw.c — mmu_idx 决定翻译体制
// ARMMMUIdx_E10_0  → EL1&0 Stage-1 (TCR_EL1, TTBR0/1_EL1)
// ARMMMUIdx_E10_1  → EL1&0 Stage-1 (同上)
// ARMMMUIdx_Stage2 → EL1&0 Stage-2 (VTCR_EL2, VTTBR_EL2)
// ARMMMUIdx_E2     → EL2 (TCR_EL2, TTBR0_EL2)
// ARMMMUIdx_E20_0  → VHE EL2&0 (TCR_EL2, TTBR0/1_EL2)
// ARMMMUIdx_SE3    → EL3 (TCR_EL3, TTBR0_EL3)

// regime_has_2_ranges(): ptw.c:1511-1527
// EL1&0 和 VHE EL2&0 有两段地址空间 (TTBR0 + TTBR1)
// EL2(非VHE) 和 EL3 只有一段 (TTBR0)
```

---

## 7. TLBI 指令

### 7.1 实现位置

```c
// tcg/tlb-insns.c — TLBI 指令实现
// 注册为 ARMCPRegInfo，writefn 触发 TLB 操作

// 全局操作:
tlbiall_is_write()   → tlb_flush_all_cpus_synced()     // TLBI VMALLE1IS
tlbiasid_is_write()  → tlb_flush_all_cpus_synced()     // TLBI ASIDE1IS
tlbimva_is_write()   → tlb_flush_page_all_cpus_synced() // TLBI VAE1IS

// 本地操作:
tlbiall_write()      → tlb_flush(cs)                    // TLBI VMALLE1
tlbiasid_write()     → tlb_flush(cs)                    // TLBI ASIDE1
```

### 7.2 VHE/EL2 感知的掩码

```c
// tlb-insns.c:224-260
// vae1_tlbmask(): 根据 HCR_EL2.E2H 决定刷新范围
// 非 E2H: 仅刷 EL1&0 TLB 条目
// E2H:    可能也刷 EL2 TLB 条目
```

### 7.3 HCR_EL2.TTLB 陷阱

```c
// tlb-insns.c:17-47
// EL1 执行 TLBI 时，HCR_EL2.TTLB 位可陷入 EL2
// access_ttlb():   检查 HCR_TTLB
// access_ttlbis(): 检查 HCR_TTLB | HCR_TTLBIS
// access_ttlbos(): 检查 HCR_TTLB | HCR_TTLBOS
```

---

## 8. QEMU TLB 简化

```c
// ptw.c:1964-1970 — QEMU 不实现 ASID
// "QEMU ignores shareability and cacheability attributes,
//  so we don't need to do anything with SH, ORGN, IRGN.
//  Similarly, TCR.A1 selects ASID source, but QEMU's TLB
//  doesn't implement ASID-like capability so we always flush
//  the TLB any time the ASID is changed."

// 简化影响:
// 1. ASID 变化 → 全局 TLB 刷新（真实硬件仅失效该 ASID）
// 2. SH/IRGN/ORGN 位不影响行为（QEMU 不模拟缓存一致性）
// 3. TLBI ASIDE1IS 等效于 TLBI VMALLE1IS
```

---

## 9. 小结

| 组件 | 实现 |
|------|------|
| **TCR_EL1** | T0SZ/T1SZ 控制 VA 位宽，TG0/TG1 控制颗粒，写时无条件刷 TLB |
| **TTBR0/1_EL1** | BADDR 提供页表基地址，ASID[63:48] 变化时才刷 TLB |
| **MAIR_EL1** | 8 个 8 位属性槽，PTE.AttrIndx 索引获取 Device/Normal 属性 |
| **VTCR_EL2** | SL0 控制 Stage-2 起始级别，T0SZ 控制 IPA 空间大小 |
| **PTW** | get_phys_addr_lpae() 逐级遍历，支持 4K/16K/64K 颗粒和 52 位地址 |
| **两级翻译** | Stage-1(EL1) → IPA → Stage-2(EL2) → PA，权限/属性合并 |
| **TLBI** | tlb-insns.c 实现，IS 变体跨 CPU 同步，受 HCR_EL2.TTLB 陷阱 |
| **简化** | QEMU 不跟踪 ASID，不模拟缓存一致性，SH/IRGN/ORGN 忽略 |
