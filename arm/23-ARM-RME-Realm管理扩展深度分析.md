# ARM Realm Management Extension（RME）深度分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中 ARM Realm Management Extension（RME, FEAT_RME）的完整实现。RME 将传统的 TrustZone 双世界（Secure/NonSecure）扩展为四世界安全模型（Root/Secure/Realm/NonSecure），并引入 Granule Protection Table（GPT）实现物理地址级别的安全颗粒管理。

**与 arm/17、arm/22 的关系：**
- arm/17 覆盖 TrustZone 架构概览
- arm/22 覆盖 MPC/PPC/MSC 安全组件模拟
- 本文聚焦 RME 四世界模型、GPT 查找、GPC 故障、Realm MMU 处理

**关键源文件：**
- `include/hw/arm/arm-security.h` — ARMSecuritySpace 四状态枚举
- `target/arm/cpu.h` — SCR_NSE 定义、安全空间辅助函数
- `target/arm/helper.c` — arm_security_space()、RME 系统寄存器
- `target/arm/ptw.c` — GPT 查找、GPC 检查、Realm MMU 索引
- `target/arm/internals.h` — GPC 故障类型
- `target/arm/cpu-features.h` — isar_feature_aa64_rme()
- `target/arm/tcg/cpu64.c` — x-rme QOM 属性

---

## 2. 四世界安全模型

### 2.1 ARMSecuritySpace 枚举

```c
// arm-security.h:18-23
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,   // Secure 世界
    ARMSS_NonSecure  = 1,   // Non-Secure 世界
    ARMSS_Root       = 2,   // Root（EL3 独占，RME 新增）
    ARMSS_Realm      = 3,   // Realm（机密计算，RME 新增）
} ARMSecuritySpace;
```

**编码与 GPI 的对应关系：** 枚举值的低 2 位与 GPT GPI 编码一致（`gpi & 3 == ARMSecuritySpace`），简化了 GPT 查找的比较逻辑。

### 2.2 SCR_EL3 的 NS/NSE 组合

```c
// cpu.h:1798
#define SCR_NSE  (1ULL << 62)  // NS Extension bit（RME 新增）
```

RME 通过 `SCR_EL3.NSE:NS` 两位选择四个安全状态：

| NSE | NS | 安全空间 | 说明 |
|-----|----|----------|------|
| 0 | 0 | Secure | 传统 Secure 世界 |
| 0 | 1 | NonSecure | 传统 Normal 世界 |
| 1 | 0 | — | **保留**（Reserved） |
| 1 | 1 | Realm | 机密计算世界 |

**注意：** `NSE=1, NS=0` 是保留组合，QEMU 忽略 NSE 当 NS=0 时。

### 2.3 安全空间判定函数

```c
// helper.c:10131-10161 — arm_security_space()
ARMSecuritySpace arm_security_space(CPUARMState *env)
{
    // M-profile：使用 env->v7m.secure
    if (arm_feature(env, ARM_FEATURE_M))
        return arm_secure_to_space(env->v7m.secure);

    // 无 EL3 → NonSecure
    if (!arm_feature(env, ARM_FEATURE_EL3))
        return ARMSS_NonSecure;

    // AArch64 EL3：RME → Root，否则 → Secure
    if (is_a64(env) && extract32(env->pstate, 2, 2) == 3) {
        if (cpu_isar_feature(aa64_rme, env_archcpu(env)))
            return ARMSS_Root;    // ← RME 核心变化
        else
            return ARMSS_Secure;
    }

    // AArch32 Monitor → Secure
    // 其余 → arm_security_space_below_el3()
    return arm_security_space_below_el3(env);
}
```

**关键设计：** 有 RME 的 CPU 在 EL3 时处于 **Root** 状态（而非 Secure），这是四世界模型的核心区别。

```c
// helper.c:10163-10187 — arm_security_space_below_el3()
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env)
{
    if (!(env->cp15.scr_el3 & SCR_NS))
        return ARMSS_Secure;           // NS=0 → Secure
    else if (env->cp15.scr_el3 & SCR_NSE)
        return ARMSS_Realm;            // NS=1, NSE=1 → Realm
    else
        return ARMSS_NonSecure;        // NS=1, NSE=0 → NonSecure
}
```

### 2.4 向后兼容辅助函数

```c
// arm-security.h:25-29
static inline bool arm_space_is_secure(ARMSecuritySpace space)
{
    // Root 也被视为"secure"（兼容旧 TrustZone 谓词）
    return space == ARMSS_Secure || space == ARMSS_Root;
}
```

---

## 3. CPU 特性与 QOM 属性

### 3.1 特性检测

```c
// cpu-features.h:1107-1115
static inline bool isar_feature_aa64_rme(const ARMISARegisters *id)
{
    return FIELD_EX64_IDREG(id, ID_AA64PFR0, RME) != 0;
}
static inline bool isar_feature_aa64_rme_gpc2(const ARMISARegisters *id)
{
    return FIELD_EX64_IDREG(id, ID_AA64PFR0, RME) >= 2;
}
```

- `ID_AA64PFR0.RME`（bit 52-55）：0=无 RME，1=RME v1，2=RME v2（GPC2）
- `isar_feature_aa64_rme_gpc2()` 启用额外的 GPC 特性（如 NSO、NA6/NA7 GPI）

### 3.2 QOM 属性

```c
// tcg/cpu64.c:1406-1408
object_property_add_bool(obj, "x-rme", cpu_arm_get_rme, cpu_arm_set_rme);
object_property_add(obj, "x-l0gptsz", "uint32",
                    cpu_max_get_l0gptsz, cpu_max_set_l0gptsz, ...);
```

```c
// tcg/cpu64.c:163
// 设置 x-rme=true 时：
FIELD_DP64_IDREG(&cpu->isar, ID_AA64PFR0, RME, value ? 2 : 0);
```

- `x-rme`：布尔属性，启用 RME 支持（设置 `ID_AA64PFR0.RME = 2`）
- `x-l0gptsz`：Level-0 GPT 大小参数（影响 `GPCCR_EL3.L0GPTSZ` reset 值）
- 前缀 `x-` 表示实验性功能

**使用方式：**
```bash
qemu-system-aarch64 -cpu max,x-rme=true ...
```

---

## 4. RME 系统寄存器

### 4.1 寄存器定义

```c
// helper.c:4972-4986 — rme_reginfo[]
{ .name = "GPCCR_EL3",  // Granule Protection Check Control Register
  .opc0=3, .opc1=6, .crn=2, .crm=1, .opc2=6,
  .access = PL3_RW, .writefn = gpccr_write, .resetfn = gpccr_reset,
  .fieldoffset = offsetof(CPUARMState, cp15.gpccr_el3) },

{ .name = "GPTBR_EL3",  // Granule Protection Table Base Register
  .opc0=3, .opc1=6, .crn=2, .crm=1, .opc2=4,
  .access = PL3_RW,
  .fieldoffset = offsetof(CPUARMState, cp15.gptbr_el3) },

{ .name = "MFAR_EL3",   // Monitor Fault Address Register（GPC 故障地址）
  .opc0=3, .opc1=6, .crn=6, .crm=0, .opc2=5,
  .access = PL3_RW,
  .fieldoffset = offsetof(CPUARMState, cp15.mfar_el3) },

{ .name = "DC_CIPAPA",   // 缓存维护（NOP 实现）
  .access = PL3_W, .type = ARM_CP_NOP },
```

### 4.2 GPCCR_EL3 字段

| 字段 | 含义 |
|------|------|
| GPC | 全局 GPC 使能（0=禁用，1=启用） |
| PPS | Protected Physical address Size |
| PGS | Physical Granule Size（4KB/16KB/64KB） |
| L0GPTSZ | Level-0 GPT 大小（30+值） |
| SH/ORGN/IRGN | GPT 内存属性（共享/缓存） |
| SPAD | Secure Physical Address Disable |
| NSPAD | Non-Secure Physical Address Disable |
| RLPAD | Realm Physical Address Disable |
| SA | System Agent GPI 控制 |
| NSP | Non-Secure Protected GPI 控制 |
| NA6/NA7 | 保留 GPI 值控制（GPC2） |
| NSO | Non-Secure Only GPI 控制（GPC2） |
| APPSAA | 地址超 PPS 抑制故障 |

### 4.3 GPTBR_EL3 字段

`GPTBR_EL3` 存储 GPT Level-0 表的物理基地址（右移 12 位）：
```c
tableaddr = config.gptbr << 12;  // ptw.c:435
```

---

## 5. Granule Protection Table（GPT）查找

### 5.1 整体架构

GPT 是一个两级物理地址查找表，由 EL3（Root）固件配置，用于将每个物理页面（granule）映射到一个安全空间：

```
物理地址 → Level-0 GPT 表 → Level-1 GPT 表 → GPI（4-bit）→ 安全空间
```

### 5.2 查找入口

```c
// ptw.c:333-337 — arm_granule_protection_check()
bool arm_granule_protection_check(ARMGranuleProtectionConfig config,
                                  uint64_t paddress,      // 物理地址
                                  ARMSecuritySpace pspace, // 访问目标空间
                                  ARMSecuritySpace ss,     // 发起者空间
                                  ARMMMUFaultInfo *fi)
```

**参数含义：**
- `config`：包含 `gpccr`、`gptbr`、`parange`、`gpt_as`（Root 地址空间）
- `paddress`：MMU 翻译后的物理地址
- `pspace`：翻译结果声称的安全空间
- `ss`：发起访问的 CPU 安全空间

### 5.3 优先级检查流程

```c
// ptw.c:349-432 — 四级优先级检查
```

**Priority 1：GPCCR 配置有效性**
- PPS 超过实现的物理地址宽度 → `fault_walk`（L0）
- SH 字段组合非法 → `fault_walk`
- PGS 字段保留值 → `fault_walk`

**Priority 2：地址空间禁用位**
```c
// ptw.c:408-418
static const uint8_t disable_masks[4] = {
    [ARMSS_Secure]    = R_GPCCR_SPAD_MASK,
    [ARMSS_NonSecure] = R_GPCCR_NSPAD_MASK,
    [ARMSS_Root]      = 0,          // Root 永远不被禁用
    [ARMSS_Realm]     = R_GPCCR_RLPAD_MASK,
};
if (gpccr & disable_masks[pspace])
    goto fault_fail;
```

**Priority 3：地址超 PPS**
```c
// ptw.c:427-432
if (paddress & ~pps_mask) {
    if (pspace == ARMSS_NonSecure || FIELD_EX64(gpccr, GPCCR, APPSAA))
        return true;  // NonSecure 超 PPS 不报错
    goto fault_fail;
}
```

**Priority 4：GPTBR 基地址超 PPS**

### 5.4 Level-0 查找

```c
// ptw.c:449-474
index = extract64(paddress, l0gptsz, pps - l0gptsz);
tableaddr += index * 8;
entry = address_space_ldq_le(config.gpt_as, tableaddr, attrs, &result);

switch (extract32(entry, 0, 4)) {
case 1:  // Block descriptor — 直接获取 GPI
    gpi = extract32(entry, 4, 4);
    goto found;
case 3:  // Table descriptor — 继续 Level-1
    tableaddr = entry & ~0xf;
    break;
default: // Invalid
    goto fault_walk;
}
```

**Block descriptor：** 整个 L0 范围（2^l0gptsz 字节）共享同一 GPI。
**Table descriptor：** 指向 Level-1 表，继续细粒度查找。

### 5.5 Level-1 查找

```c
// ptw.c:476-505
index = extract64(paddress, pgs + 4, l0gptsz - pgs - 4);
tableaddr += index * 8;
entry = address_space_ldq_le(config.gpt_as, tableaddr, attrs, &result);

switch (extract32(entry, 0, 4)) {
case 1:  // Contiguous descriptor — GPI 适用于连续范围
    gpi = extract32(entry, 4, 4);
    break;
default: // Granule descriptor — 每 granule 独立 GPI
    index = extract64(paddress, pgs, 4);
    gpi = extract64(entry, index * 4, 4);  // 每 4-bit 一个 GPI
    break;
}
```

**Granule descriptor 编码：** 一个 64-bit entry 包含 16 个 4-bit GPI，每个对应一个 granule（由 PGS 决定大小）。

### 5.6 GPI 值解释

```c
// ptw.c:507-554
switch (gpi) {
case 0b0000:  // No access → 故障
    break;
case 0b1111:  // All access → 任何空间都可访问
    return true;
case 0b1000:  // Secure（需 SEL2 支持）
case 0b1001:  // Non-Secure
case 0b1010:  // Root
case 0b1011:  // Realm
    if (pspace == (gpi & 3))  // 低 2 位 == ARMSecuritySpace 枚举值
        return true;
    break;
case 0b0100:  // System Agent Only
    if (FIELD_EX64(gpccr, GPCCR, SA) == 0) goto fault_walk;
    break;
case 0b0101:  // Non-Secure Protected
    if (FIELD_EX64(gpccr, GPCCR, NSP) == 0) goto fault_walk;
    break;
case 0b1101:  // Non-Secure Only（GPC2）
    if (FIELD_EX64(gpccr, GPCCR, NSO))
        return (pspace == ARMSS_NonSecure &&
                (ss == ARMSS_NonSecure || ss == ARMSS_Root));
    goto fault_walk;
default:      // Reserved → 故障
    goto fault_walk;
}
```

**GPI 与 ARMSecuritySpace 的映射：**

| GPI（4-bit） | 含义 | 允许的访问空间 |
|--------------|------|----------------|
| 0000 | No Access | 无 |
| 0100 | System Agent | 由 GPCCR.SA 控制 |
| 0101 | NS Protected | 由 GPCCR.NSP 控制 |
| 1000 | Secure | Secure only |
| 1001 | Non-Secure | NonSecure only |
| 1010 | Root | Root only |
| 1011 | Realm | Realm only |
| 1101 | NS Only | NS（且发起者为 NS/Root） |
| 1111 | All Access | 任何空间 |

---

## 6. GPC 故障处理

### 6.1 故障类型

```c
// internals.h:728-738
typedef enum ARMFaultType {
    ...
    ARMFault_GPCFOnWalk,    // GPT 遍历本身失败
    ARMFault_GPCFOnOutput,  // MMU 翻译成功但 GPC 检查失败
} ARMFaultType;

typedef enum ARMGPCF {
    GPCF_None,
    GPCF_AddressSize,  // 地址超 PPS 或 GPTBR 超 PPS
    GPCF_Walk,         // GPT 条目无效/保留
    GPCF_EABT,         // GPT 内存读取外部中止
    GPCF_Fail,         // GPI 不匹配（访问被拒绝）
} ARMGPCF;
```

### 6.2 GPC 检查时机

```c
// ptw.c:3800-3833 — get_phys_addr_gpc()
static bool get_phys_addr_gpc(CPUARMState *env, S1Translate *ptw,
                              vaddr address, ...)
{
    // 1. 先执行普通 MMU 翻译
    if (get_phys_addr_nogpc(env, ptw, address, ...))
        return true;  // MMU 翻译失败，直接返回

    // 2. 翻译成功后，检查 GPC
    if (FIELD_EX64(env->cp15.gpccr_el3, GPCCR, GPC)) {
        ARMGranuleProtectionConfig config = {
            .gpccr = env->cp15.gpccr_el3,
            .gptbr = env->cp15.gptbr_el3,
            .parange = ...,
            .support_sel2 = ...,
            .gpt_as = arm_addressspace(env_cpu(env), attrs)  // Root AS
        };
        if (!arm_granule_protection_check(config,
                result->f.phys_addr,    // 翻译后的物理地址
                result->f.attrs.space,  // 翻译结果的安全空间
                ptw->in_space,          // 发起者安全空间
                fi)) {
            fi->type = ARMFault_GPCFOnOutput;  // GPC 故障
            return true;
        }
    }
    return false;  // 翻译 + GPC 均成功
}
```

**关键设计：** GPC 在 MMU 翻译**之后**执行，检查翻译产生的物理地址是否被 GPT 允许。

### 6.3 Walk 期间的 GPC 故障

```c
// ptw.c:724-726
```

当 Stage-1 PTW 读取页表条目时，如果页表所在的物理地址被 GPC 拒绝，故障类型从 `GPCFOnOutput` 转换为 `GPCFOnWalk`。

### 6.4 故障综合码（FSC）

```c
// internals.h:938-948
case ARMFault_GPCFOnWalk:
    // level = -1: 0b100011; level 0-3: 0b100100 | level
    if (fi->level < 0) fsc = 0b100011;
    else fsc = 0b100100 | fi->level;
    break;
case ARMFault_GPCFOnOutput:
    fsc = 0b101000;  // 输出地址 GPC 故障
    break;
```

---

## 7. Realm MMU 处理

### 7.1 物理空间 MMUIDX

```c
// mmuidx.h 中定义
ARMMMUIdx_Phys_S,      // Secure 物理空间
ARMMMUIdx_Phys_NS,     // NonSecure 物理空间
ARMMMUIdx_Phys_Root,   // Root 物理空间（RME 新增）
ARMMMUIdx_Phys_Realm,  // Realm 物理空间（RME 新增）
```

### 7.2 Stage-2 PTW 索引选择

```c
// ptw.c:210-225 — ptw_idx_for_stage_2()
switch (arm_security_space_below_el3(env)) {
case ARMSS_NonSecure:
    return ARMMMUIdx_Phys_NS;
case ARMSS_Realm:
    return ARMMMUIdx_Phys_Realm;    // Realm 使用独立的物理索引
case ARMSS_Secure:
    // Secure 下根据 VTCR/VSTCR 的 NSW/SW 位选择
    if (stage2idx == ARMMMUIdx_Stage2_S)
        s2walk_secure = !(env->cp15.vstcr_el2 & R_VSTCR_SW_MASK);
    else
        s2walk_secure = !(env->cp15.vtcr_el2 & R_VTCR_NSW_MASK);
    return s2walk_secure ? ARMMMUIdx_Phys_S : ARMMMUIdx_Phys_NS;
}
```

### 7.3 Realm 地址空间隔离

在 Realm 安全空间中：
1. **Stage-1 PTW**：使用 Realm EL1 的 TTBR0/TTBR1，在 Realm 物理空间中查找页表
2. **Stage-2 PTW**：使用 Realm EL2 的 VTTBR_EL2，在 `ARMMMUIdx_Phys_Realm` 空间中遍历
3. **GPC 检查**：翻译后的物理地址必须被 GPT 标记为 Realm（GPI=0b1011）

---

## 8. 四世界架构总览

### 8.1 异常级别与安全空间

```
┌─────────────────────────────────────────────────────┐
│                      EL3 (Root)                      │
│  Monitor / RMM Dispatcher / TF-A                     │
│  arm_security_space() → ARMSS_Root                   │
│  所有 GPT/GPCCR/GPTBR 寄存器仅 EL3 可访问            │
├────────────┬────────────┬────────────┬───────────────┤
│ Secure     │ NonSecure  │ Realm      │               │
│ SCR NS=0   │ SCR NS=1   │ SCR NS=1   │               │
│ NSE=0      │ NSE=0      │ NSE=1      │               │
├────────────┼────────────┼────────────┤               │
│ S-EL2      │ NS-EL2     │ R-EL2      │               │
│ S-EL1/EL0  │ NS-EL1/EL0 │ R-EL1/EL0  │               │
│ Secure OS  │ Rich OS    │ Realm VM   │               │
└────────────┴────────────┴────────────┴───────────────┘
```

### 8.2 GPT 在架构中的位置

```
CPU 发起内存访问
  │
  ▼
MMU Stage-1（虚拟地址 → 中间物理地址）
  │
  ▼
MMU Stage-2（中间物理地址 → 物理地址）[如果有 EL2]
  │
  ▼
GPC 检查（物理地址 → GPT 查找 → GPI 匹配）[如果 GPCCR.GPC=1]
  │
  ├── 匹配 → 访问物理内存
  └── 不匹配 → GPC 故障（GPCFOnOutput）
```

### 8.3 安全空间转换

| 操作 | 源 | 目标 | 机制 |
|------|-----|------|------|
| SMC | NS/Secure/Realm | Root (EL3) | 异常入口 |
| ERET | Root (EL3) | NS/Secure/Realm | SCR_EL3.NSE:NS |
| RMM 调用 | Realm EL2 | Root (EL3) | SMC |
| Realm VM 创建 | Root | Realm | 设置 SCR.NSE+NS + GPT 标记 |

---

## 9. Realm 与 TrustZone 对比

| 特性 | TrustZone（无 RME） | RME |
|------|---------------------|-----|
| 安全世界数 | 2（Secure/NS） | 4（Root/Secure/Realm/NS） |
| EL3 角色 | Secure Monitor | Root（独立于 Secure） |
| 物理隔离 | MPC/PPC（外设级） | GPT（页面级，硬件强制） |
| 隔离粒度 | 块/端口 | 物理页面（4KB/16KB/64KB） |
| 配置者 | EL3 固件 | EL3 固件（GPT 表） |
| 机密计算 | 不支持 | Realm VM（内存对 Hypervisor 不可见） |
| Hypervisor 信任 | 必须信任 | Realm 不信任 Hypervisor |
| 内存保护 | 软件配置 MPC | 硬件 GPC 自动检查 |

### 9.1 Realm 的核心价值

Realm 的关键安全属性：
1. **Hypervisor 不可见**：Realm VM 的内存被 GPT 标记为 GPI=Realm，NS EL2 无法访问
2. **Root 管理**：只有 EL3（Root）可以修改 GPT，将物理页面分配给 Realm
3. **硬件强制**：GPC 在每次物理地址访问时自动检查，无需软件参与
4. **RMM（Realm Management Monitor）**：运行在 Realm EL2，管理 Realm VM 的 Stage-2 页表

---

## 10. QEMU 实现状态

### 10.1 已实现

- ✅ `ARMSecuritySpace` 四状态枚举
- ✅ `SCR_NSE` 位处理
- ✅ `arm_security_space()` / `arm_security_space_below_el3()` 四世界判定
- ✅ GPT 两级遍历（Level-0 Block/Table，Level-1 Contiguous/Granule）
- ✅ GPC 检查（`get_phys_addr_gpc()`）
- ✅ GPC 故障类型（`GPCFOnWalk`、`GPCFOnOutput`）
- ✅ `GPCCR_EL3`、`GPTBR_EL3`、`MFAR_EL3` 系统寄存器
- ✅ `ARMMMUIdx_Phys_Realm` / `ARMMMUIdx_Phys_Root` 物理索引
- ✅ `x-rme` / `x-l0gptsz` QOM 属性
- ✅ GPC2 扩展（NA6/NA7/NSO GPI）

### 10.2 virt 机器支持

目前 `hw/arm/virt.c` 中**未发现**显式的 RME/Realm 内存区域支持。RME 主要通过 CPU 特性（`x-rme=true`）启用，GPT 表由 Guest 固件（如 TF-A）在运行时配置，不需要机器级别的额外支持。

---

## 11. 小结

| 组件 | 实现位置 | 核心功能 |
|------|----------|----------|
| **四状态枚举** | arm-security.h:18-23 | ARMSS_Secure/NonSecure/Root/Realm |
| **SCR_NSE** | cpu.h:1798 | NSE:NS 组合选择四世界 |
| **安全空间判定** | helper.c:10131-10187 | EL3+RME→Root，NSE+NS→Realm |
| **GPT 遍历** | ptw.c:333-572 | 两级查找，16 种 GPI 值 |
| **GPC 检查** | ptw.c:3800-3833 | MMU 翻译后检查物理地址 |
| **GPC 故障** | internals.h:728-738 | GPCFOnWalk/OnOutput + 5 种子类型 |
| **RME 寄存器** | helper.c:4972-4986 | GPCCR/GPTBR/MFAR_EL3 |
| **Realm MMU** | ptw.c:210-225 | ARMMMUIdx_Phys_Realm |
| **CPU 特性** | cpu-features.h:1107-1115 | isar_feature_aa64_rme/gpc2 |
| **QOM 属性** | tcg/cpu64.c:1406-1408 | x-rme, x-l0gptsz |

**关键设计原则：**
1. RME 将 EL3 从 Secure 提升为 Root，独立于三个 Lower EL 安全空间
2. GPT 实现物理页面级安全隔离，由 EL3 固件配置，硬件自动执行
3. GPI 低 2 位 == ARMSecuritySpace 枚举值，简化查找比较
4. GPC 在 MMU 翻译后执行，是最后一道安全关卡
5. Realm 隔离不信任 Hypervisor，适用于机密计算场景
