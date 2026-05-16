# ARM64 特殊系统寄存器与 Cache/AT 指令深度分析

## 1. 概述

除 MMU 和陷阱相关的核心寄存器外，ARM64 还有大量功能性系统寄存器和系统指令。本文分析 CPACR（协处理器访问控制）、CONTEXTIDR（上下文 ID）、PAR/AT（地址翻译）、TPIDR（线程 ID）、FPCR/FPSR（浮点控制/状态）、Timer 寄存器以及 DC/IC Cache 维护指令在 QEMU 中的完整实现。

**关键源文件：**
- `target/arm/helper.c` — 寄存器定义与写处理
- `target/arm/tcg/cpregs-at.c` — AT 地址翻译指令（~510行）
- `target/arm/tcg/vfp_helper.c` — FPCR/FPSR 底层实现

---

## 2. CPACR_EL1 — 协处理器访问控制

### 2.1 寄存器定义

```c
// helper.c:680-687
{ .name = "CPACR_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .opc1 = 0, .crn = 1, .crm = 0, .opc2 = 2,
  .access = PL1_RW,
  .accessfn = cpacr_access,         // CPTR_EL2/EL3.TCPAC 陷阱
  .fgt = FGT_CPACR_EL1,
  .writefn = cpacr_write, .readfn = cpacr_read,
  .resetfn = cpacr_reset },
```

### 2.2 关键位域

```
CPACR_EL1:
  FPEN [21:20] — FP/SIMD 访问控制
    00/10: EL0+EL1 陷阱  01: 仅 EL0 陷阱  11: 不陷阱
  ZEN  [17:16] — SVE 访问控制（同编码）
  SMEN [25:24] — SME 访问控制（同编码）
```

### 2.3 三级 FP/SIMD 陷阱链

```c
// helper.c:9873-9945 — fp_exception_el()
// 返回应陷入的 EL (0=不陷阱)

// 级别 1: CPACR_EL1.FPEN
//   除非 E2H+TGE 覆盖（VHE 宿主模式跳过）
//   FPEN=00/10 → 陷入 EL1

// 级别 2: CPTR_EL2.TFP/FPEN
//   非 E2H: TFP=1 → 陷入 EL2
//   E2H:    FPEN 同 CPACR 编码

// 级别 3: CPTR_EL3.TFP
//   TFP=1 → 陷入 EL3（最终拦截）
```

### 2.4 写副作用

```c
// helper.c:549-595 — cpacr_write()
// 1. ARMv7: 仅保留已实现的协处理器位（CP10/CP11/ASEDIS/D32DIS）
// 2. ARMv8: 大部分位 RES0
// 3. NSACR.CP10=0 时，非安全 CPACR.CP10/CP11 写被忽略
// 4. 直接写入 env->cp15.cpacr_el1（无 TLB 刷新）
```

---

## 3. CONTEXTIDR_EL1 — 上下文标识符

```c
// helper.c:456-465
{ .name = "CONTEXTIDR_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .opc1 = 0, .crn = 13, .crm = 0, .opc2 = 1,
  .access = PL1_RW, .accessfn = access_tvm_trvm,
  .fgt = FGT_CONTEXTIDR_EL1,
  .fieldoffset = offsetof(CPUARMState, cp15.contextidr_el[1]),
  .writefn = contextidr_write },

// helper.c:387-401 — contextidr_write()
// 副作用:
//   VMSA 短描述符格式（非 LPAE）时 ASID 嵌入 CONTEXTIDR
//   → 值变化时 tlb_flush()
//   LPAE/PMSA → 不刷 TLB（ASID 在 TTBR 中）
```

---

## 4. PAR_EL1 与 AT 地址翻译指令

### 4.1 PAR_EL1 布局

```
PAR_EL1 (翻译成功时):
  F    [0]     = 0 (成功)
  SH   [8:7]   = 共享属性
  ATTR [63:56] = 内存属性（来自 MAIR）
  PA   [51:12] = 物理地址

PAR_EL1 (翻译失败时):
  F    [0]     = 1 (失败)
  FST  [6:1]   = 故障状态码
  S    [9]     = Stage-2 故障标志
```

### 4.2 AT 指令类型

```c
// cpregs-at.c:389-453 — v8_ats_reginfo[]
// AArch64 AT 操作:
AT S1E1R/W   — EL1 Stage-1 翻译（读/写）      [opc1=0, opc2=0/1]
AT S1E0R/W   — EL0 Stage-1 翻译                [opc1=0, opc2=2/3]
AT S12E1R/W  — EL1 Stage-1+2 翻译              [opc1=4, opc2=4/5]
AT S12E0R/W  — EL0 Stage-1+2 翻译              [opc1=4, opc2=6/7]
AT S1E2R/W   — EL2 Stage-1 翻译                [opc1=4, opc2=0/1]
AT S1E3R/W   — EL3 Stage-1 翻译                [opc1=6, opc2=0/1]
```

### 4.3 do_ats_write() 核心实现

```c
// cpregs-at.c:26-112 — do_ats_write()
uint64_t do_ats_write(CPUARMState *env, uint64_t value,
                      unsigned prot_check, ARMMMUIdx mmu_idx, ARMSecuritySpace ss)
{
    // 1. 调用 get_phys_addr_for_at() 执行翻译
    bool ret = get_phys_addr_for_at(env, value, prot_check, mmu_idx, ss, &res, &fi);
    
    // 2. 翻译失败时特殊处理:
    //    - Stage-2 fault + AT S1E*（EL1）→ 异常到 EL2（而非写 PAR）
    //    - SyncExternalOnWalk → Data Abort 异常
    
    // 3. 翻译成功: 构建 PAR_EL1 值
    //    PA[51:12] = phys_addr
    //    SH = par_el1_shareability()  // Device/Normal-NC 强制为 0b10
    //    ATTR = cacheattrs.attrs
    //    F = 0
    
    // 4. 翻译失败(非异常): 构建失败 PAR
    //    F = 1, FST = fault_status_code
    
    return par64;
}
```

### 4.4 MMU 索引选择

```c
// cpregs-at.c:330-356 — ats_write64()
// 根据操作类型选择翻译体制:
case 0: // AT S1E1R/W
    mmu_idx = ARMMMUIdx_Stage1_E1;    // 仅 Stage-1
case 2: // AT S1E0R/W
    mmu_idx = ARMMMUIdx_Stage1_E0;    // 仅 Stage-1 EL0
case 4: // AT S12E1R/W
    mmu_idx = ARMMMUIdx_E10_1;        // Stage-1 + Stage-2
case 6: // AT S12E0R/W
    mmu_idx = ARMMMUIdx_E10_0;        // Stage-1 + Stage-2 EL0

// VHE (E2H) 模式调整:
// AT S1E2R → ARMMMUIdx_E20_2（两段地址空间）
// 非 E2H   → ARMMMUIdx_E2（单段）
```

---

## 5. TPIDR — 线程 ID 寄存器

```c
// helper.c:1071-1103

// TPIDR_EL0: 用户可读写线程 ID
{ .name = "TPIDR_EL0", .opc0 = 3, .opc1 = 3, .crn = 13, .crm = 0, .opc2 = 2,
  .access = PL0_RW,
  .fieldoffset = offsetof(CPUARMState, cp15.tpidr_el[0]) },

// TPIDRRO_EL0: 用户只读线程 ID（内核设置）
{ .name = "TPIDRRO_EL0", .opc0 = 3, .opc1 = 3, .crn = 13, .crm = 0, .opc2 = 3,
  .access = PL0_R | PL1_W,          // EL0 只读，EL1 可写
  .fieldoffset = offsetof(CPUARMState, cp15.tpidrro_el[0]) },

// TPIDR_EL1: 内核线程 ID
{ .name = "TPIDR_EL1", .opc0 = 3, .opc1 = 0, .crn = 13, .crm = 0, .opc2 = 4,
  .access = PL1_RW,
  .fieldoffset = offsetof(CPUARMState, cp15.tpidr_el[1]) },

// Linux 用法:
//   TPIDR_EL0  → 用户空间 TLS 基地址
//   TPIDRRO_EL0 → 只读 TLS（vDSO 等）
//   TPIDR_EL1  → 内核 current_thread_info 指针
```

---

## 6. CurrentEL — 当前异常级别

```c
// helper.c:3480-3482
{ .name = "CURRENTEL", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 0, .opc2 = 2, .crn = 4, .crm = 2,
  .access = PL1_R, .type = ARM_CP_CURRENTEL },

// ARM_CP_CURRENTEL: 特殊类型
// TCG 翻译时直接返回 (current_el << 2)
// 不走 helper 路径，翻译时常量折叠
```

---

## 7. FPCR / FPSR — 浮点控制与状态

### 7.1 寄存器定义

```c
// helper.c:3458-3465
{ .name = "FPCR", .opc0 = 3, .opc1 = 3, .crn = 4, .crm = 4, .opc2 = 0,
  .access = PL0_RW, .type = ARM_CP_FPU,
  .readfn = aa64_fpcr_read, .writefn = aa64_fpcr_write },

{ .name = "FPSR", .opc0 = 3, .opc1 = 3, .crn = 4, .crm = 4, .opc2 = 1,
  .access = PL0_RW, .type = ARM_CP_FPU | ARM_CP_SUPPRESS_TB_END,
  .readfn = aa64_fpsr_read, .writefn = aa64_fpsr_write },
```

### 7.2 FPCR 关键位

```
FPCR:
  RMode [23:22] — 舍入模式 (00=RN, 01=RP, 10=RM, 11=RZ)
  FZ    [24]    — Flush-to-Zero（非正规数清零）
  DN    [25]    — Default NaN
  AHP   [26]    — Alternative Half-Precision
  FZ16  [19]    — FP16 Flush-to-Zero
  Len   [18:16] — 向量长度（仅 AArch32 遗留）
  Stride [21:20] — 向量步长（仅 AArch32 遗留）
```

### 7.3 读写实现

```c
// helper.c:3036-3055
// aa64_fpcr_write → vfp_set_fpcr(env, value)
//   → 设置 env->vfp.fpcr
//   → 更新 softfloat 舍入模式: set_float_rounding_mode()
//   → 更新 flush-to-zero 等标志

// aa64_fpsr_write → vfp_set_fpsr(env, value)
//   → 设置 env->vfp.fpsr
//   → 更新异常标志位 (IOC/DZC/OFC/UFC/IXC/IDC)

// ARM_CP_SUPPRESS_TB_END: FPSR 写不结束当前 TB
// （FPSR 变化不影响翻译行为）
```

---

## 8. Timer 寄存器

### 8.1 CNTFRQ_EL0 — 计数器频率

```c
// helper.c:2036-2041
// 只读（EL0 可读），返回系统计数器频率
// 典型值: QEMU 默认 62500000 (62.5MHz)
```

### 8.2 CNTVCT_EL0 — 虚拟计数器

```c
// helper.c:2145-2149
// 读取: gt_virt_cnt_read()
// 返回 physical_count - CNTVOFF_EL2
// EL2 通过 CNTVOFF 控制虚拟计数器偏移
```

### 8.3 CNTHCTL_EL2 陷阱

```c
// helper.c:1166-1227
// CNTHCTL_EL2 控制 EL0/EL1 对 Timer 寄存器的访问:
//   EL0PCTEN: EL0 物理计数器访问
//   EL0VCTEN: EL0 虚拟计数器访问
//   EL1PCTEN: EL1 物理计数器访问
//   EL1TVT:   EL1 虚拟 Timer 寄存器陷阱
//   EL1TVCT:  EL1 虚拟计数器陷阱
```

---

## 9. DC/IC Cache 维护指令

### 9.1 DC ZVA — 数据缓存零化

```c
// helper.c:3471-3479
{ .name = "DC_ZVA", .opc0 = 1, .opc1 = 3, .crn = 7, .crm = 4, .opc2 = 1,
  .access = PL0_W, .type = ARM_CP_DC_ZVA,
  .accessfn = aa64_zva_access },   // 特殊访问检查

// helper.c:3199-3225 — aa64_zva_access()
// 陷阱检查:
//   EL0 + 非 VHE: SCTLR_EL1.DZE=0 → 陷入 EL1
//   EL0 + VHE:    SCTLR_EL2.DZE=0 → 陷入 EL2
//   EL0/EL1: HCR_EL2.TDZ=1 → 陷入 EL2

// ARM_CP_DC_ZVA: 特殊类型
// TCG 翻译时生成 memset(addr, 0, block_size) 操作
```

### 9.2 IC IVAU — 指令缓存失效

```c
// helper.c:3414-3441
// QEMU 不模拟指令缓存，但 IC IVAU 触发 TB 失效
// → 确保自修改代码正确执行
// linux-user 模式: 影响 JIT 代码缓存一致性
```

### 9.3 Cache 维护陷阱

```c
// HCR_EL2 陷阱位:
HCR_TSW  (22) — Cache set/way 操作 → EL2    // helper.c:345-353
HCR_TPCP (23) — CP15 barrier 操作 → EL2     // helper.c:3158-3164
HCR_TPU  (24) — Cache PoU 操作 → EL2        // helper.c:3167-3197
HCR_TDZ  (28) — DC ZVA → EL2                // helper.c:3199-3225

// 虚拟化场景: Hypervisor 截获 Guest Cache 维护
// 避免 Guest 直接影响物理缓存
```

---

## 10. QEMU 实现特点

### 10.1 不模拟缓存

```
QEMU 不实现真实的指令/数据缓存:
  - DC CVAC/CIVAC/CVAU → NOP（无缓存可刷新）
  - DC ZVA → 直接 memset 零化（不经缓存）
  - IC IVALLU → NOP（但 IC IVAU 触发 TB 失效）
  - Cache 维护指令仅保留陷阱语义（虚拟化正确性）
```

### 10.2 AT 指令的异常行为

```
AT 指令在翻译失败时可能直接抛异常（而非写 PAR）:
  - Stage-2 fault on AT S1E* from EL1 → 异常到 EL2
  - SyncExternalOnWalk → Data Abort 异常
这些特殊路径在 cpregs-at.c:44-111 中精确实现
```

---

## 11. 寄存器汇总

| 寄存器 | 编码 | EL | 核心功能 | 写副作用 |
|--------|------|-----|---------|---------|
| CPACR_EL1 | 3,0,1,0,2 | PL1 | FP/SVE/SME 访问控制 | 位过滤，无 TLB 刷新 |
| CONTEXTIDR_EL1 | 3,0,13,0,1 | PL1 | 进程上下文 ID | 短描述符时刷 TLB |
| PAR_EL1 | 3,0,7,4,0 | PL1 | AT 指令结果 | AT 指令写入 |
| TPIDR_EL0 | 3,3,13,0,2 | PL0 | 用户线程 ID | 无 |
| TPIDRRO_EL0 | 3,3,13,0,3 | PL0_R | 用户只读线程 ID | EL1 写 |
| TPIDR_EL1 | 3,0,13,0,4 | PL1 | 内核线程 ID | 无 |
| CurrentEL | 3,0,4,2,2 | PL1_R | 当前 EL | 只读 |
| FPCR | 3,3,4,4,0 | PL0 | FP 舍入/异常控制 | softfloat 模式更新 |
| FPSR | 3,3,4,4,1 | PL0 | FP 状态标志 | 异常位更新 |
| CNTFRQ_EL0 | 3,3,14,0,0 | PL0_R | Timer 频率 | 只读 |
| CNTVCT_EL0 | 3,3,14,0,2 | PL0_R | 虚拟计数器 | 只读 |
| DC_ZVA | 1,3,7,4,1 | PL0_W | 缓存行零化 | memset + TDZ 陷阱 |

---

## 12. 小结

| 组件 | 实现 |
|------|------|
| **CPACR_EL1** | 三级 FP/SIMD 陷阱链（CPACR → CPTR_EL2 → CPTR_EL3），FPEN/ZEN/SMEN 2 位控制 |
| **CONTEXTIDR** | 短描述符格式时含 ASID → TLB 刷新；LPAE/PMSA 无副作用 |
| **AT 指令** | do_ats_write() 执行完整翻译，结果写入 PAR_EL1；Stage-2 fault 可变异常 |
| **TPIDR** | 三个线程 ID 寄存器（EL0 RW / EL0 RO / EL1 RW），用于 TLS 和 current 指针 |
| **FPCR/FPSR** | FPCR 控制舍入模式和 FTZ，FPSR 记录异常标志，通过 softfloat 库实现 |
| **Timer** | CNTVCT = 物理计数 - CNTVOFF，CNTHCTL_EL2 控制 EL0/EL1 访问陷阱 |
| **DC/IC** | QEMU 不模拟缓存，DC ZVA 直接 memset，IC IVAU 触发 TB 失效，保留陷阱语义 |
