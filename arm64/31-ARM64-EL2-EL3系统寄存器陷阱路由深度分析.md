# ARM64 EL2/EL3 系统寄存器陷阱路由深度分析

## 1. 概述

ARM64 虚拟化和安全扩展的核心机制之一是**系统寄存器陷阱路由**：当 Guest（EL0/EL1）访问某些系统寄存器时，Hypervisor（EL2）或 Secure Monitor（EL3）可以截获并处理这些访问。QEMU 实现了完整的陷阱路由框架，包括 HCR_EL2 粗粒度陷阱、SCR_EL3 安全陷阱、CPTR 协处理器陷阱，以及 FEAT_FGT 细粒度陷阱。

**关键源文件：**
- `target/arm/cpu.h` — HCR_EL2/SCR_EL3 位定义（1698-1800行）
- `target/arm/internals.h` — CPTR_EL2/EL3 位定义（144-161行）
- `target/arm/cpregs.h` — FGT 位定义（377-520行）
- `target/arm/helper.c` — accessfn 实现
- `target/arm/tcg/op_helper.c` — 陷阱→异常流程

---

## 2. HCR_EL2 陷阱位全景

```c
// cpu.h:1698-1754 — HCR_EL2 位定义
```

### 2.1 虚拟内存寄存器陷阱

| 位 | 名称 | Bit | 陷阱目标 |
|----|------|-----|---------|
| TVM | Trap Virtual Memory | 26 | EL1 **写** SCTLR/TCR/TTBR/MAIR/DACR/ESR/FAR 等 → EL2 |
| TRVM | Trap Read VM | 30 | EL1 **读** 上述寄存器 → EL2 |

```c
// helper.c:332-343 — access_tvm_trvm()
CPAccessResult access_tvm_trvm(CPUARMState *env, const ARMCPRegInfo *ri,
                               bool isread)
{
    if (arm_current_el(env) == 1) {
        uint64_t trap = isread ? HCR_TRVM : HCR_TVM;
        if (arm_hcr_el2_eff(env) & trap)
            return CP_ACCESS_TRAP_EL2;
    }
    return CP_ACCESS_OK;
}
// 使用此函数的寄存器: SCTLR_EL1, TCR_EL1, TTBR0/1_EL1, MAIR_EL1, ESR_EL1 等
```

### 2.2 中断路由

| 位 | 名称 | Bit | 效果 |
|----|------|-----|------|
| FMO | FIQ Mask Override | 3 | 物理 FIQ 路由到 EL2；使能 vFIQ |
| IMO | IRQ Mask Override | 4 | 物理 IRQ 路由到 EL2；使能 vIRQ |
| AMO | Async abort Mask Override | 5 | SError 路由到 EL2；使能 vSError |

```c
// cpu-irq.c:47-83 — 中断路由逻辑
// IMO=1 时: EL1 的 IRQ 被路由到 EL2
// 同时允许 HCR_EL2.VI 注入虚拟 IRQ
```

### 2.3 指令/操作陷阱

| 位 | 名称 | Bit | 陷阱对象 |
|----|------|-----|---------|
| TWI | Trap WFI | 13 | EL0/EL1 WFI → EL2 |
| TWE | Trap WFE | 14 | EL0/EL1 WFE → EL2 |
| TSC | Trap SMC | 19 | EL1 SMC → EL2 |
| TACR | Trap ACTLR | 21 | EL1 访问 ACTLR → EL2 |
| TSW | Trap set/way | 22 | EL1 cache set/way 操作 → EL2 |
| TPCP | Trap CP15 barrier | 23 | EL1 CP15 barrier 操作 → EL2 |
| TTLB | Trap TLB | 25 | EL1 TLBI 指令 → EL2 |
| TGE | Trap General Exceptions | 27 | EL0/EL1 异常全部路由到 EL2 |

```c
// 示例: WFI 陷阱处理
// tcg/op_helper.c:322-366
// 检查 HCR_EL2.TWI: EL0/EL1 执行 WFI → 异常到 EL2

// 示例: SMC 陷阱
// tcg/op_helper.c:1111-1199
// 检查 HCR_EL2.TSC: EL1 执行 SMC → 异常到 EL2
```

### 2.4 ID 寄存器陷阱

| 位 | 名称 | Bit | 陷阱对象 |
|----|------|-----|---------|
| TID0 | Trap ID group 0 | 15 | ISAR 寄存器 |
| TID1 | Trap ID group 1 | 16 | MIDR/CTR/TCMTR |
| TID2 | Trap ID group 2 | 17 | CCSIDR/CLIDR/CSSELR |
| TID3 | Trap ID group 3 | 18 | AArch64 ID 寄存器全组 |
| TID4 | Trap ID group 4 | 49 | 额外 ID 寄存器 |
| TID5 | Trap ID group 5 | 58 | GMID_EL1（MTE） |

```c
// helper.c:847-857 — access_tid4()
// helper.c:928-936 — access_tid1()
// helper.c:5365-5373 — access_tid5()
// helper.c:5765-5799 — access_tid3()
```

### 2.5 MMU 控制

| 位 | 名称 | Bit | 效果 |
|----|------|-----|------|
| VM | Virtualization MMU | 0 | 使能 Stage-2 翻译 |
| DC | Default Cacheable | 12 | 禁用 Stage-1，Stage-2 默认 cacheable |
| PTW | Protected TW | 1 | 禁止设备内存上的翻译表遍历 |
| E2H | Enable EL2 Host | 34 | VHE 模式，EL2 使用两段地址空间 |
| NV/NV1/NV2 | Nested Virt | 42-45 | 嵌套虚拟化支持 |

---

## 3. SCR_EL3 安全陷阱位

```c
// cpu.h:1756-1798 — SCR_EL3 位定义
```

### 3.1 核心安全位

| 位 | 名称 | Bit | 效果 |
|----|------|-----|------|
| NS | Non-Secure | 0 | EL1/EL0 运行在非安全世界 |
| IRQ | IRQ routing | 1 | 物理 IRQ 路由到 EL3 |
| FIQ | FIQ routing | 2 | 物理 FIQ 路由到 EL3 |
| EA | External Abort | 3 | SError 路由到 EL3 |
| SMD | Secure Monitor Disable | 7 | 禁用 SMC 指令 |
| HCE | HVC Enable | 8 | 使能 HVC 指令 |
| RW | Register Width | 10 | EL2 执行状态 (1=AArch64, 0=AArch32) |
| FGTEN | FGT Enable | 27 | 使能 Fine-Grained Traps |

### 3.2 中断路由优先级

```
物理 IRQ 路由决策:
  1. SCR_EL3.IRQ=1 → 路由到 EL3
  2. HCR_EL2.IMO=1 → 路由到 EL2
  3. 否则 → 路由到 EL1

// cpu-irq.c:132-160 — 完整路由逻辑
// 检查顺序: SCR → HCR → 默认目标
```

---

## 4. CPTR_EL2/EL3 — 协处理器陷阱

### 4.1 位定义

```c
// internals.h:144-161
// CPTR_EL2 (!E2H 模式):
FIELD(CPTR_EL2, TZ, 8, 1)     // 陷阱 SVE
FIELD(CPTR_EL2, TFP, 10, 1)   // 陷阱 FP/SIMD
FIELD(CPTR_EL2, TSM, 12, 1)   // 陷阱 SME
FIELD(CPTR_EL2, TTA, 28, 1)   // 陷阱 Trace 访问
FIELD(CPTR_EL2, TCPAC, 31, 1) // 陷阱 CPACR 访问

// CPTR_EL2 (E2H 模式): 使用 FPEN/ZEN/SMEN 字段（2位控制）
FIELD(CPTR_EL2, FPEN, 20, 2)  // 00/10=陷阱, 01=仅 EL0 陷阱, 11=不陷阱

// CPTR_EL3:
FIELD(CPTR_EL3, TFP, 10, 1)   // 陷阱 FP/SIMD 到 EL3
FIELD(CPTR_EL3, EZ, 8, 1)     // SVE 使能
FIELD(CPTR_EL3, ESM, 12, 1)   // SME 使能
FIELD(CPTR_EL3, TCPAC, 31, 1) // 陷阱 CPTR_EL2 访问
```

### 4.2 FP/SIMD 三级陷阱检查

```c
// helper.c:9873-9945 — fp_exception_el()
// 返回应该陷入的 EL (0=不陷阱, 1/2/3=目标EL)

// 检查顺序:
// 1. CPACR_EL1.FPEN (除非 E2H+TGE 覆盖)
//    00/10 → 陷入 EL1, 01 → 仅 EL0 陷入 EL1, 11 → 不陷阱
// 2. NSACR (AArch32 EL3 控制非安全 FP 访问)
// 3. CPTR_EL2.TFP/FPEN
//    非E2H: TFP=1 → 陷入 EL2
//    E2H:   FPEN 同 CPACR 编码
// 4. CPTR_EL3.TFP=1 → 陷入 EL3
```

### 4.3 CPACR 访问陷阱

```c
// helper.c:623-639 — cpacr_access()
// EL1 访问 CPACR → 检查 CPTR_EL2.TCPAC → 可能陷入 EL2
// 任何 EL < 3 → 检查 CPTR_EL3.TCPAC → 可能陷入 EL3
```

---

## 5. Fine-Grained Traps (FEAT_FGT)

### 5.1 FGT 寄存器

```c
// cpregs.h:377-384 — FGT 寄存器索引
FGTREG_HFGRTR = 0   // 功能寄存器读陷阱 (fgt_read[0])
FGTREG_HDFGRTR = 1  // 调试寄存器读陷阱 (fgt_read[1])
FGTREG_HFGWTR = 0   // 功能寄存器写陷阱 (fgt_write[0])
FGTREG_HDFGWTR = 1  // 调试寄存器写陷阱 (fgt_write[1])
FGTREG_HFGITR = 0   // 指令陷阱 (fgt_exec[0])
```

### 5.2 逐寄存器陷阱位

```c
// cpregs.h:386-500 — HFGRTR_EL2/HFGWTR_EL2 位定义（部分）
FIELD(HFGRTR_EL2, MAIR_EL1, 24, 1)     // bit 24: MAIR_EL1 读陷阱
FIELD(HFGRTR_EL2, SCTLR_EL1, 29, 1)    // bit 29: SCTLR_EL1 读陷阱
FIELD(HFGRTR_EL2, TCR_EL1, 32, 1)      // bit 32: TCR_EL1 读陷阱
FIELD(HFGRTR_EL2, TTBR0_EL1, 36, 1)    // bit 36: TTBR0_EL1 读陷阱
FIELD(HFGRTR_EL2, TTBR1_EL1, 37, 1)    // bit 37: TTBR1_EL1 读陷阱
FIELD(HFGRTR_EL2, VBAR_EL1, 38, 1)     // bit 38: VBAR_EL1 读陷阱

// 对应 HFGWTR_EL2 也有同位号的写陷阱位
// 每个 ARMCPRegInfo 的 .fgt 字段编码 IDX + BITPOS
```

### 5.3 运行时检查

```c
// op_helper.c:858-890 — FGT 检查逻辑
if (arm_fgt_active(env, current_el)) {
    idx = FIELD_EX32(ri->fgt, FGT, IDX);      // 哪个 FGT 寄存器
    bitpos = FIELD_EX32(ri->fgt, FGT, BITPOS); // 哪一位
    rev = FIELD_EX32(ri->fgt, FGT, REV);       // 反向位（1=允许）
    
    if (isread) trapword = env->cp15.fgt_read[idx];
    else        trapword = env->cp15.fgt_write[idx];
    
    trapbit = extract64(trapword, bitpos, 1);
    if (rev) trapbit = !trapbit;               // 反向：位=1 允许
    if (trapbit) → CP_ACCESS_TRAP_EL2;
}

// FGT 使能条件: SCR_EL3.FGTEN=1 (cpu.h:1780)
```

### 5.4 FGT vs HCR 粗粒度陷阱对比

```
HCR_EL2.TVM:
  → 陷阱 ALL 虚拟内存寄存器的写操作（一刀切）
  → SCTLR + TCR + TTBR0 + TTBR1 + MAIR + ... 全部陷阱

HFGWTR_EL2:
  → 可以仅陷阱 SCTLR_EL1 的写，不影响 TCR/TTBR
  → 每个寄存器独立控制
  → 更精细，减少不必要的 VM Exit
```

---

## 6. 陷阱→异常执行流程

### 6.1 accessfn 返回陷阱

```
accessfn() 返回 CP_ACCESS_TRAP_EL2
  │
  └→ op_helper.c:896-956 — HELPER(access_check_cp_reg)
      │
      ├── CP_ACCESS_TRAP_EL3:
      │     EL3 是 AArch32? → EXCP_MON_TRAP
      │     EL3 是 AArch64? → EXCP_UDEF + syndrome
      │
      ├── CP_ACCESS_TRAP_EL2:
      │     target_el = 2
      │
      ├── CP_ACCESS_TRAP_EL1:
      │     target_el = 1
      │
      ├── CP_ACCESS_UNDEFINED:
      │     syndrome = syn_uncategorized()
      │     (FEAT_IDST: ID 寄存器保持 sysreg syndrome)
      │
      └→ raise_exception(env, excp, syndrome, target_el)
```

### 6.2 Syndrome 生成

```c
// syndrome.h:119+ — syn_aa64_sysregtrap()
// 生成 EC=0x18 (系统寄存器访问) 的 syndrome:
//   [24:20] = EC = 0x18
//   [19:17] = op0
//   [16:14] = op1  
//   [13:10] = CRn
//   [9:5]   = CRm
//   [4:2]   = op2
//   [1]     = Direction (0=写, 1=读)
//   [0]     = Rt (目标寄存器)
// Hypervisor 可据此完全识别被陷阱的寄存器和操作
```

### 6.3 raise_exception 路由

```c
// tcg/op_helper.c:47-69 — raise_exception()
// 额外路由: HCR_EL2.TGE=1 时
// 目标为 EL1 的异常重新路由到 EL2
if (target_el == 1 && (hcr_el2 & HCR_TGE)) {
    target_el = 2;  // TGE 覆盖
}
```

---

## 7. 访问检查优先级

```
系统寄存器访问检查顺序（从高到低优先级）:

1. EL 权限检查 (cp_access_ok)
   → 基于 PLx_R/W 和当前 EL
   → 失败 → UNDEF (无 syndrome)

2. HSTR_EL2 位图 (AArch32)
   → cp15 协处理器级别陷阱
   → 失败 → CP_ACCESS_TRAP_EL2

3. Fine-Grained Trap (FGT)
   → 逐寄存器位控制
   → 需要 SCR_EL3.FGTEN=1
   → 失败 → CP_ACCESS_TRAP_EL2

4. 寄存器 accessfn() 回调
   → 检查 HCR/SCR/CPTR 各陷阱位
   → 可返回 TRAP_EL1/2/3

5. 翻译时权限 (cp_access_ok)
   → 编译阶段优化
```

---

## 8. 陷阱位分类汇总

| 寄存器 | 陷阱类别 | 主要陷阱位 | 代码位置 |
|--------|---------|-----------|---------|
| **HCR_EL2** | VM 寄存器 | TVM(26), TRVM(30) | helper.c:332-343 |
| **HCR_EL2** | 中断路由 | FMO(3), IMO(4), AMO(5) | cpu-irq.c:47-83 |
| **HCR_EL2** | 指令操作 | TWI(13), TWE(14), TSC(19) | op_helper.c:322-407 |
| **HCR_EL2** | TLB 操作 | TTLB(25), TTLBIS(54), TTLBOS(55) | tlb-insns.c:17-47 |
| **HCR_EL2** | ID 寄存器 | TID0-5(15-18,49,58) | helper.c:847-936 |
| **HCR_EL2** | Cache 操作 | TSW(22), TPCP(23), TPU(24) | helper.c:345-363 |
| **HCR_EL2** | MMU 控制 | VM(0), DC(12), E2H(34) | do_hcr_write() |
| **SCR_EL3** | 中断路由 | IRQ(1), FIQ(2), EA(3) | cpu-irq.c:132-160 |
| **SCR_EL3** | 安全控制 | NS(0), RW(10), SMD(7), HCE(8) | op_helper.c:1086-1199 |
| **SCR_EL3** | FGT 使能 | FGTEN(27) | op_helper.c:858 |
| **CPTR_EL2** | FP/SIMD | TFP(10), FPEN(20:21) | helper.c:9873-9945 |
| **CPTR_EL2** | SVE/SME | TZ(8), TSM(12), ZEN(16:17) | helper.c:9873+ |
| **CPTR_EL2** | CPACR | TCPAC(31) | helper.c:623-639 |
| **CPTR_EL3** | FP/SVE | TFP(10), EZ(8), ESM(12) | helper.c:9941-9945 |
| **HFGRTR/HFGWTR** | 逐寄存器 | 64位，每位一个寄存器 | op_helper.c:858-890 |

---

## 9. 小结

| 组件 | 实现 |
|------|------|
| **HCR_EL2** | 56+ 位，覆盖 VM 寄存器/中断路由/指令/TLB/ID/Cache/MMU 陷阱 |
| **SCR_EL3** | 安全世界控制，IRQ/FIQ/EA 路由到 EL3，SMC/HVC 控制 |
| **CPTR_EL2/EL3** | FP/SIMD/SVE/SME 三级陷阱链，TCPAC 控制 CPACR 访问 |
| **FGT** | HFGRTR/HFGWTR 逐寄存器位控制，比 HCR.TVM 更精细 |
| **accessfn** | 每个寄存器可注册自定义检查，检查 HCR/SCR/CPTR 相关位 |
| **异常流程** | accessfn → CP_ACCESS_TRAP_ELx → syndrome 生成 → raise_exception |
| **TGE 覆盖** | HCR_EL2.TGE=1 将 EL1 目标异常全部重路由到 EL2 |
| **优先级** | EL 权限 > HSTR > FGT > accessfn |
