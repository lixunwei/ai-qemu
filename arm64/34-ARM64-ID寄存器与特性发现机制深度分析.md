# ARM64 ID 寄存器与特性发现机制深度分析

## 1. 概述

ARM64 架构通过一组只读 ID 寄存器向软件暴露 CPU 支持的特性。操作系统和 Hypervisor 通过读取这些寄存器来发现 CPU 能力（如加密指令、SVE 支持、内存模型等），而无需探测指令是否会产生 UNDEFINED 异常。QEMU 通过 `ARMISARegisters` 结构体集中管理所有 ID 字段，通过 `isar_feature_aa64_*()` 内联函数检查特性，通过 `modify_arm_cp_regs()` 控制对用户态暴露的位域。

**关键源文件：**
- `target/arm/helper.c` — ID 寄存器定义与注册（6240-6760 行）
- `target/arm/cpu-features.h` — 特性位域定义与 isar_feature 检查（~1700行）
- `target/arm/cpu64.c` — CPU 模型 ISAR 初始化
- `target/arm/tcg/cpu64.c` — `-cpu max` TCG 特性填充

---

## 2. ID 寄存器空间编码

### 2.1 AArch64 ID 寄存器编码规则

```
所有 AArch64 ID 寄存器: op0=3, op1=0, CRn=0
  CRm=4: ID_AA64PFR{0-7}_EL1  — 处理器特性
  CRm=5: ID_AA64DFR{0-7}_EL1  — 调试特性
  CRm=6: ID_AA64ISAR{0-7}_EL1 — 指令集属性
  CRm=7: ID_AA64MMFR{0-7}_EL1 — 内存模型特性

每个寄存器以 4 位为单位划分字段，每个字段表示一个特性的实现级别:
  0 = 未实现    1 = 基础    2 = 扩展版本    ...
```

### 2.2 AArch32 ID 寄存器编码

```
op0=3, op1=0, CRn=0
  CRm=1: ID_PFR{0,1}, ID_DFR0, ID_AFR0, ID_MMFR{0-3}
  CRm=2: ID_ISAR{0-6}, ID_MMFR4
```

---

## 3. AArch64 ID 寄存器定义

### 3.1 ID_AA64PFR — 处理器特性

```c
// helper.c:6398-6420
// ID_AA64PFR0_EL1: 非 ARM_CP_CONST（GIC 字段需运行时填充）
//   readfn = id_aa64pfr0_read — 动态插入 GIC 版本字段
//   关键字段:
//     EL0/EL1/EL2/EL3 [3:0]/[7:4]/[11:8]/[15:12] — 各 EL 支持的执行状态
//     FP   [19:16] — 浮点支持 (0=FP, 0xF=不支持)
//     AdvSIMD [23:20] — SIMD 支持
//     GIC  [27:24] — GIC CPU 接口版本（运行时填充）
//     SVE  [35:32] — SVE 支持
//     SEL2 [39:36] — Secure EL2 支持
//     SME  [43:40] — SME 支持

// ID_AA64PFR1_EL1: ARM_CP_CONST
//   关键字段: BT, SSBS, MTE, SME（通过 exported_bits 控制用户可见性）
```

### 3.2 ID_AA64ISAR — 指令集属性

```c
// helper.c:6486-6525
// ID_AA64ISAR0_EL1: ARM_CP_CONST, resetvalue = GET_IDREG(isar, ID_AA64ISAR0)
//   字段: AES, SHA1, SHA2, CRC32, ATOMIC (LSE), RDM, SHA3, SM3, SM4, DP, FHM, TS, RNDR
// ID_AA64ISAR1_EL1: 字段: DPB, APA, API, JSCVT, FRINTTS, SB, SPECRES, BF16, I8MM, LRCPC
// ID_AA64ISAR2_EL1: 字段: WFxT, RPRES, MOPS, BC, RPRFM, CSSC
// ID_AA64ISAR3-7: 保留（resetvalue = 0）
```

### 3.3 ID_AA64MMFR — 内存模型特性

```c
// helper.c:6526-6545
// ID_AA64MMFR0_EL1: PARange, ASIDBits, BigEnd, SNSMem, TGran16/64/4, ECV
// ID_AA64MMFR1_EL1: HAFDBS, VMIDBits, VH (VHE), HPDS, LO, PAN, AFP
// ID_AA64MMFR2_EL1: CnP, UAO, LSM, IESB, VA, CCIDX, NV, AT, ST, TT
// ID_AA64MMFR3_EL1: S1PIE, S2PIE, SCTLR2, ANERR, ADERR
```

### 3.4 ID_AA64DFR — 调试特性

```c
// helper.c:6690-6692 — exported_bits 机制中
// ID_AA64DFR0_EL1: DebugVer, TraceVer, PMUVer, BRPs, WRPs, CTX_CMPs
//   fixed_bits: DebugVer = 0x6 (ARMv8 调试架构)
// ID_AA64DFR1_EL1: 保留
```

### 3.5 ID_AA64ZFR0 / ID_AA64SMFR0 — SVE/SME 特性

```c
// helper.c:6652-6675 — exported_bits 机制中
// ID_AA64ZFR0_EL1:
//   SVEver, AES, BitPerm, BFloat16, B16B16, SHA3, SM4, I8MM, F32MM, F64MM
// ID_AA64SMFR0_EL1:
//   F32F32, BI32I32, B16F32, F16F32, I8I32, F16F16, B16B16, I16I32,
//   F64F64, I16I64, SMEver, FA64
```

---

## 4. ARMISARegisters — 统一特性存储

### 4.1 结构定义

```c
// cpu.h:1085 — ARMISARegisters 结构体
// 包含所有 ID 寄存器的值:
//   id_aa64pfr0, id_aa64pfr1, id_aa64pfr2
//   id_aa64isar0, id_aa64isar1, id_aa64isar2
//   id_aa64mmfr0, id_aa64mmfr1, id_aa64mmfr2, id_aa64mmfr3, id_aa64mmfr4
//   id_aa64dfr0, id_aa64dfr1
//   id_aa64zfr0, id_aa64smfr0
//   id_pfr0, id_pfr1, id_dfr0, id_afr0
//   id_mmfr0-4, id_isar0-6
//   mvfr0, mvfr1, mvfr2
//   dbgdidr, dbgdevid, dbgdevid1
//   reset_pmcr_el0
```

### 4.2 GET_IDREG 宏

```c
// 注册时使用 GET_IDREG(isar, REG_NAME) 获取 ISAR 中的值
// 将 ARMISARegisters 中的字段作为 ARM_CP_CONST 的 resetvalue
// 所有 ID 寄存器均为只读（PL1_R）
```

---

## 5. 特性检查 — isar_feature 体系

### 5.1 位域定义

```c
// cpu-features.h:68-401 — FIELD() 宏定义 ID 寄存器位域
// 使用 QEMU 标准 FIELD() 宏生成 R_REG_FIELD_SHIFT/MASK/LENGTH
// 例:
//   FIELD(ID_AA64ISAR0, AES, 4, 4)     → R_ID_AA64ISAR0_AES_SHIFT = 4
//   FIELD(ID_AA64PFR0, SVE, 32, 4)     → R_ID_AA64PFR0_SVE_SHIFT = 32
//   FIELD(ID_AA64MMFR0, PARANGE, 0, 4) → R_ID_AA64MMFR0_PARANGE_SHIFT = 0
```

### 5.2 isar_feature 内联函数

```c
// cpu-features.h:807-1581 — 数百个 isar_feature_aa64_* 函数
// 模式统一: 从 ARMISARegisters 提取字段，检查阈值

static inline bool isar_feature_aa64_aes(const ARMISARegisters *id) {
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, AES) != 0;
}

static inline bool isar_feature_aa64_sha512(const ARMISARegisters *id) {
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, SHA2) > 1;  // 级别 2+
}

static inline bool isar_feature_aa64_lse(const ARMISARegisters *id) {
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, ATOMIC) >= 2;
}

// 比较模式:
//   != 0 — 只要实现就返回 true
//   > N  — 需要特定级别以上
//   >= N — 需要最低级别
```

### 5.3 cpu_isar_feature 宏

```c
// cpu-features.h:1696-1697
#define cpu_isar_feature(name, cpu) \
    ({ const ARMCPU *cpu_ = (cpu); isar_feature_##name(&cpu_->isar); })

// 用法: cpu_isar_feature(aa64_sve, cpu) → isar_feature_aa64_sve(&cpu->isar)
// 编译期内联展开，零运行时开销
```

---

## 6. CPU 模型初始化

### 6.1 具名 CPU 模型

```c
// cpu64.c — 每个 CPU 型号定义自己的 ISAR 值
// 示例: cortex-a57, cortex-a72, neoverse-n1 等
// cpu64.c:52-74, 220-247, 299-317, 353-380 等
// 直接赋值:
//   cpu->isar.id_aa64pfr0 = 0x0000000000000011;  // EL0+EL1 AArch64
//   cpu->isar.id_aa64isar0 = 0x00000000...;       // 该 CPU 支持的指令
```

### 6.2 `-cpu max` 模型

```c
// tcg/cpu64.c:1386-1407 — TCG 最大特性填充
// 启用 TCG 可模拟的所有特性:
//   SVE, PAC, MTE, BTI, LSE, SHA, AES, SM3/SM4, FP16, BF16, I8MM, RNG 等
// 通过 FIELD_DP64 逐字段设置 ISAR 值
// aarch64_add_pauth_properties() 添加 PAuth 相关属性
// cpu_max_get_sve_max_vq() / setter 控制 SVE 向量长度
```

---

## 7. ID 寄存器陷阱控制

### 7.1 HCR_EL2.TID 位

```
HCR_EL2 陷阱位（cpu.h:1709-1752）:
  TID0 — Jazelle ID (ID_JOSCR 等)
  TID1 — REVIDR_EL1, AIDR_EL1 等
  TID2 — CCSIDR, CCSIDR2, CLIDR, CSSELR
  TID3 — 所有 ID_AA64* 和 ID_* 寄存器（核心陷阱位）
  TID4 — 虚拟缓存控制
  TID5 — GMID_EL1 (MTE Granule)
```

### 7.2 access_tid3 — 核心 ID 陷阱

```c
// helper.c:5784-5799 — access_tid3()
// v8+: 无条件检查 HCR_EL2.TID3
// v7:  不检查（使用 access_v7a_tid3 仅对 v7A 固定集合）

// helper.c:5765-5782 — access_v7a_tid3()
// v7A 固定陷阱集: ID_PFR{0,1}, ID_DFR0, ID_AFR0,
//   ID_MMFR{0-3}, ID_ISAR{0-5}
// EL < 2 且 HCR_EL2.TID3=1 → CP_ACCESS_TRAP_EL2
```

### 7.3 其他 TID 陷阱

```c
// helper.c:928-936 — access_tid1()
// HCR_EL2.TID1 → 陷阱 REVIDR_EL1, AIDR_EL1

// helper.c:847-857 — access_tid4()
// HCR_EL2.TID2|TID4 → 陷阱 CCSIDR 等虚拟缓存 ID

// helper.c:5365 — access_tid5()
// HCR_EL2.TID5 → 陷阱 GMID_EL1
```

---

## 8. 用户态特性暴露 — modify_arm_cp_regs

### 8.1 exported_bits / fixed_bits 机制

```c
// helper.c:6637-6737 — v8_user_idregs[]
// 在 CONFIG_USER_ONLY 构建中，修改 ID 寄存器的可见位:
//   exported_bits: 仅这些位对用户态可见（其余清零）
//   fixed_bits:    强制设置这些位（不可修改）

// 示例:
{ .name = "ID_AA64PFR0_EL1",
  .exported_bits = R_ID_AA64PFR0_FP_MASK |
                   R_ID_AA64PFR0_ADVSIMD_MASK |
                   R_ID_AA64PFR0_SVE_MASK |
                   R_ID_AA64PFR0_DIT_MASK },
  // EL0/EL1/EL2/EL3 字段不暴露给用户态进程

{ .name = "ID_AA64MMFR0_EL1",
  .exported_bits = R_ID_AA64MMFR0_ECV_MASK,
  .fixed_bits = (0xfu << R_ID_AA64MMFR0_TGRAN64_SHIFT) |
                (0xfu << R_ID_AA64MMFR0_TGRAN4_SHIFT) },
  // Granule 大小字段固定为 0xF（支持），仅 ECV 可变
```

### 8.2 modify_arm_cp_regs() 调用

```c
// helper.c:6737
modify_arm_cp_regs(v8_idregs, v8_user_idregs);
// 在注册前修改 reginfo 数组
// 匹配名称后应用 exported_bits/fixed_bits 掩码
// 使用户态读取的 ID 值仅包含有意义的信息
```

---

## 9. 未实现 ID 寄存器的 RAZ 处理

```c
// helper.c:7031-7045 — 空白 ID 寄存器块
// 对于未实现的 ID 寄存器空间（CRn=0 的保留编码）
// 统一注册为 ARM_CP_CONST, resetvalue=0
// 保证 Guest 读取任何未实现的 ID 寄存器返回 0
// 而不是产生 UNDEFINED 异常（ARM 架构要求）
```

---

## 10. ID 寄存器分类汇总

| 寄存器 | CRm | 功能域 | 关键字段 |
|--------|------|--------|---------|
| ID_AA64PFR0 | 4 | 处理器特性 | EL0-EL3 状态, FP, SIMD, GIC, SVE, SEL2, SME |
| ID_AA64PFR1 | 4 | 处理器特性2 | BT, SSBS, MTE, SME |
| ID_AA64DFR0 | 5 | 调试特性 | DebugVer, PMUVer, BRPs, WRPs |
| ID_AA64ISAR0 | 6 | 指令集 | AES, SHA, CRC, LSE, RDM, DP, RNDR |
| ID_AA64ISAR1 | 6 | 指令集2 | DPB, JSCVT, SB, LRCPC, BF16, I8MM |
| ID_AA64ISAR2 | 6 | 指令集3 | WFxT, MOPS, BC, CSSC |
| ID_AA64MMFR0 | 7 | 内存模型 | PARange, TGran, ASIDBits, ECV |
| ID_AA64MMFR1 | 7 | 内存模型2 | VHE, PAN, HPDS, AFP |
| ID_AA64MMFR2 | 7 | 内存模型3 | NV, UAO, AT, CnP |
| ID_AA64ZFR0 | 4 | SVE 特性 | SVEver, AES, SHA3, BFloat16, I8MM, F32MM |
| ID_AA64SMFR0 | 4 | SME 特性 | SMEver, FA64, F32F32, I8I32 |

---

## 11. 特性发现流程

```
Guest 启动 → 读取 ID 寄存器
  │
  ├── MRS X0, ID_AA64ISAR0_EL1
  │   → handle_sys() 查找 reginfo
  │   → access_tid3() 检查 HCR_EL2.TID3
  │     ├── TID3=1 且 EL<2 → CP_ACCESS_TRAP_EL2（Hypervisor 可伪造值）
  │     └── TID3=0 → 返回 GET_IDREG(isar, ID_AA64ISAR0) 常量值
  │
  ├── QEMU 内部特性检查:
  │   cpu_isar_feature(aa64_sve, cpu)
  │   → isar_feature_aa64_sve(&cpu->isar)
  │   → FIELD_EX64_IDREG(&isar, ID_AA64PFR0, SVE) != 0
  │   → 编译期内联，零开销
  │
  └── 用户态 (linux-user):
      MRS X0, ID_AA64ISAR0_EL1
      → 返回 (原始值 & exported_bits) | fixed_bits
      → 仅暴露用户态有意义的指令集特性
```

---

## 12. 小结

| 组件 | 实现 |
|------|------|
| **ID 寄存器空间** | 4 组 × 8 个 = 32 个 AArch64 ID 寄存器 + AArch32 ID 寄存器，全部只读 PL1_R |
| **ARMISARegisters** | 统一存储所有 ID 值，CPU 模型初始化时赋值，运行时不变 |
| **isar_feature** | 数百个内联函数检查特性位域，cpu_isar_feature 宏连接 ARMCPU → isar |
| **陷阱控制** | HCR_EL2.TID3 控制所有 ID_AA64* 陷阱，TID0-TID5 分组控制 |
| **-cpu max** | TCG 模式启用所有可模拟特性，通过 FIELD_DP64 逐字段填充 |
| **用户态过滤** | modify_arm_cp_regs + exported_bits/fixed_bits 控制用户可见位 |
| **未实现 RAZ** | 保留编码空间统一返回 0（不产生 UNDEF），符合 ARM 架构要求 |
| **GIC 动态字段** | ID_AA64PFR0.GIC 字段运行时由 readfn 填充（依赖 GIC 设备创建时机） |
