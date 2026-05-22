# ARM64 Hypervisor 虚拟化深度分析：Non-Secure EL2、Secure EL2、Stage-1/Stage-2 MMU 翻译

> 基于 QEMU 11.0.50 源码分析，聚焦 ARM64 虚拟化地址翻译主线：
> Non-Secure EL2、VHE（EL2&0）、Secure EL2（FEAT_SEL2）、Stage-1/Stage-2 两阶段翻译、
> S1 PTW through S2、权限/页大小/缓存属性合并、安全空间传播，以及 FEAT_NV/NV1/NV2 嵌套虚拟化框架。
>
> 关键源码入口：`mmuidx.h:9-198`、`helper.c:3818-3889`、`helper.c:9957-10008`、
> `ptw.c:22-90`、`ptw.c:193-246`、`ptw.c:645-730`、`ptw.c:1343-1541`、
> `ptw.c:1859-2447`、`ptw.c:3413-3658`、`ptw.c:3661-3943`、`internals.h:1062-1082`。

---

## 目录

1. [概述](#1-概述)
2. [翻译体制（Translation Regime）全景](#2-翻译体制translation-regime全景)
3. [Non-Secure EL2 — 标准 Hypervisor](#3-non-secure-el2--标准-hypervisor)
4. [Secure EL2 — FEAT_SEL2](#4-secure-el2--feat_sel2)
5. [Stage-1 翻译流程](#5-stage-1-翻译流程)
6. [Stage-2 翻译流程](#6-stage-2-翻译流程)
7. [S1+S2 结果合并](#7-s1s2-结果合并)
8. [S1Translate 上下文结构](#8-s1translate-上下文结构)
9. [安全空间与翻译体制的交互](#9-安全空间与翻译体制的交互)
10. [嵌套虚拟化（NV/NV1/NV2）概览](#10-嵌套虚拟化nvnv1nv2概览)
11. [完整调用链图](#11-完整调用链图)
12. [关键数据结构速查表](#12-关键数据结构速查表)
13. [关键函数索引](#13-关键函数索引)
14. [与其他文档的关联](#14-与其他文档的关联)
15. [附录 A：关键源文件索引](#附录-a关键源文件索引)
16. [附录 B：相关 Git Commit](#附录-b相关-git-commit)
17. [附录 C：推荐深入方向](#附录-c推荐深入方向)

---

## 1. 概述

### 1.1 ARM64 虚拟化框架总览

ARM64 虚拟化的本质，是把 **Guest 的虚拟地址访问** 分解成两层：

1. **Stage-1**：Guest VA → IPA（Intermediate Physical Address）
2. **Stage-2**：IPA → 最终 PA

在没有虚拟化时，EL1/EL0 只做 Stage-1；在有 Hypervisor 时，EL2 通过 `HCR_EL2.VM` 打开 Stage-2，QEMU 在 `ptw.c` 中把两段遍历串接起来。最核心的路径是 `get_phys_addr_nogpc()` 在 `ptw.c:3661-3735` 对不同 `ARMMMUIdx` 做分发，并在需要时转入 `get_phys_addr_twostage()`（`ptw.c:3554-3658`）。

```
           Guest EL0/EL1
                │
                │ VA
                ▼
      +-----------------------+
      | Stage-1 (TTBR/TCR)    |
      | Guest 自己的页表       |
      +-----------------------+
                │
                │ IPA
                ▼
      +-----------------------+
      | Stage-2 (VTTBR/VTCR)  |
      | Hypervisor 二级页表    |
      +-----------------------+
                │
                │ PA
                ▼
             物理内存
```

### 1.2 QEMU 中虚拟化相关源文件

| 文件 | 关键行 | 作用 |
|------|--------|------|
| `mmuidx.h` | `9-198` | 翻译体制与 `ARMMMUIdx` 总表 |
| `cpu.h` | `1695-1785` | `HCR_EL2` / `SCR_EL3` 位定义 |
| `cpu.h` | `2228-2239` | `arm_is_el2_enabled_secstate()` |
| `helper.c` | `712-836` | `scr_write()`，含 `SCR_EEL2` 掩码验证 |
| `helper.c` | `3818-3889` | `arm_hcr_el2_eff_secstate()` |
| `helper.c` | `9957-10008` | `arm_mmu_idx_el()`，EL→mmu_idx 映射 |
| `ptw.c` | `22-90` | `S1Translate` 遍历上下文 |
| `ptw.c` | `193-246` | `ptw_idx_for_stage_2()` / `regime_ttbr()` |
| `ptw.c` | `645-730` | `S1_ptw_translate()`，描述符加载 |
| `ptw.c` | `1343-1541` | `get_S2prot()` / `get_S1prot()` |
| `ptw.c` | `1859-2447` | `get_phys_addr_lpae()`，LPAE 遍历 |
| `ptw.c` | `3413-3464` | `combine_cacheattrs()` |
| `ptw.c` | `3554-3658` | `get_phys_addr_twostage()` |
| `ptw.c` | `3661-3943` | `get_phys_addr_nogpc()` / `get_phys_addr()` |
| `internals.h` | `1062-1082` | `regime_tcr()`，`Stage2_S` 合成 TCR |
| `arm-security.h` | `18-35` | `ARMSecuritySpace` 四状态 |
| `cpregs.h` | `963-970` | NV2 重定向 / VHE 重定向元数据说明 |

### 1.3 本文分析范围

本文只讨论 **A-profile AArch64 Hypervisor 路径**，重点回答四个问题：

- QEMU 如何把 ARM 架构的多种翻译体制压缩成一组 `ARMMMUIdx`？
- Non-Secure EL2 与 VHE 的差别在哪里？
- FEAT_SEL2 下 Secure EL2 为什么必须保留两套 Stage-2 索引？
- Stage-1 与 Stage-2 的页表遍历、描述符加载、权限合并、缓存属性合并如何落地？

---

## 2. 翻译体制（Translation Regime）全景

### 2.1 ARM 架构定义的翻译体制列表

`mmuidx.h:10-24` 把 ARM ARM 所说的 translation regime 用注释列得很清楚。若 EL3 为 64 位，则与 EL2 相关的 regime 包括：

- NonSecure EL1&0 stage 1
- NonSecure EL1&0 stage 2
- NonSecure EL2
- NonSecure EL2&0（VHE）
- Secure EL1&0 stage 1
- Secure EL1&0 stage 2（FEAT_SEL2）
- Secure EL2
- Secure EL2&0（FEAT_SEL2）
- Realm EL1&0 stage 1 / stage 2 / Realm EL2（FEAT_RME）
- EL3

QEMU 不是把“translation regime”一比一映射成 TLB 索引，而是做了二次编码。`mmuidx.h:32-61` 列出 9 条设计原则，其中对本文最关键的是：

1. `EL1&0` / `EL2&0` 要拆成 user 与 privileged 两个索引，因为权限不同；
2. QEMU 希望把 **S1+S2 的合成结果缓存到 TLB**；
3. `Stage1-only` 仅在慢路径 PTW/AT 指令里使用，因此不占真正 TLB；
4. **Secure Stage-2 与 NonSecure Stage-2 不能折叠**，因为 Secure EL2 会同时用到两者（`mmuidx.h:59-60`）。

### 2.2 QEMU `ARMMMUIdx` 到翻译体制的映射

从 `mmuidx.h:137-198` 看，QEMU 当前枚举实际上包含三组：

- **22 个 A-profile 核心索引**（真正对应 TLB 或架构 regime）
- **5 个 Stage1-only NOTLB 索引**（仅慢路径）
- **8 个 M-profile 索引**（本文不展开）

对虚拟化来说，最重要的映射是：

- `ARMMMUIdx_E10_0` / `E10_1` / `E10_1_PAN`：EL1&0 两阶段合成索引
- `ARMMMUIdx_E20_0` / `E20_2` / `E20_2_PAN`：VHE 下 EL2&0 单阶段索引
- `ARMMMUIdx_E2`：非 VHE 的 EL2 单阶段索引
- `ARMMMUIdx_Stage2`：Non-Secure / Realm 侧的 Stage-2
- `ARMMMUIdx_Stage2_S`：Secure Stage-2
- `ARMMMUIdx_Phys_S/NS/Root/Realm`：物理地址空间直通
- `ARMMMUIdx_Stage1_E0/E1/E1_PAN`：拆分后的 S1-only 索引

### 2.3 22 个 MMU 索引的完整分类表

下面给出 **A-profile 22 个核心索引**，它们正是 `mmuidx.h:63-85` 注释总结的那 22 类情况：

| 类别 | ARMMMUIdx | 含义 | 是否 TLB 常驻 |
|------|-----------|------|---------------|
| EL1&0 | `E10_0` | EL0，EL1&0 regime，S1+S2 合成 | 是 |
| EL1&0 | `E10_0_GCS` | EL0 + GCS | 是 |
| EL1&0 | `E10_1` | EL1，EL1&0 regime，S1+S2 合成 | 是 |
| EL1&0 | `E10_1_PAN` | EL1 + PAN | 是 |
| EL1&0 | `E10_1_GCS` | EL1 + GCS | 是 |
| EL2&0 | `E20_0` | EL0，VHE host regime | 是 |
| EL2&0 | `E20_0_GCS` | EL0 + GCS，VHE | 是 |
| EL2&0 | `E20_2` | EL2，VHE host regime | 是 |
| EL2&0 | `E20_2_PAN` | EL2 + PAN，VHE | 是 |
| EL2&0 | `E20_2_GCS` | EL2 + GCS，VHE | 是 |
| EL2 | `E2` | Non-VHE EL2 | 是 |
| EL2 | `E2_GCS` | Non-VHE EL2 + GCS | 是 |
| EL3 | `E3` | EL3 | 是 |
| EL3 | `E3_GCS` | EL3 + GCS | 是 |
| EL3&0(AA32) | `E30_0` | AArch32 Secure PL0/PL1&0 | 是 |
| EL3&0(AA32) | `E30_3_PAN` | AArch32 Secure PAN | 是 |
| Stage-2 | `Stage2_S` | Secure Stage-2 | 是 |
| Stage-2 | `Stage2` | Non-Secure / Realm Stage-2 | 是 |
| Phys | `Phys_S` | Secure 物理空间 | 是 |
| Phys | `Phys_NS` | Non-Secure 物理空间 | 是 |
| Phys | `Phys_Root` | Root 物理空间 | 是 |
| Phys | `Phys_Realm` | Realm 物理空间 | 是 |

此外，`mmuidx.h:177-185` 还定义了 5 个 `ARM_MMU_IDX_NOTLB` 索引：

- `Stage1_E0`
- `Stage1_E1`
- `Stage1_E1_PAN`
- `Stage1_E0_GCS`
- `Stage1_E1_GCS`

它们只用于：

- S1+S2 拆分遍历的 **第一阶段**
- `AT/ATS` 地址翻译指令

### 2.4 一个容易忽略但非常关键的设计点

`mmuidx.h:162-169` 对 `Stage2_S`/`Stage2` 的注释比枚举本身更重要：

> SecureEL2 中，两者会同时使用；S1 PTW 的 NS 位决定 S2 PTW 的安全性。

这意味着：

- 在 S-EL2 中，一个 Secure guest 可能需要 `Stage2_S`
- 同一个 S-EL2 又可能托管 Non-Secure guest，需要 `Stage2`
- 二者不能合并成一个“抽象 Stage-2”

这是 FEAT_SEL2 在 QEMU 实现中的一个核心分水岭。

---

## 3. Non-Secure EL2 — 标准 Hypervisor

### 3.1 Non-VHE 模式（`ARMMMUIdx_E2`）

Non-VHE 是最传统的 Hypervisor 形态：EL2 有自己独立的翻译体制，不与 EL0 共享 host regime。`arm_mmu_idx_el()` 在 `helper.c:9986-9996` 的逻辑非常直接：

```c
// helper.c:9986-9996（简化）
case 2:
    if (arm_hcr_el2_eff(env) & HCR_E2H) {
        idx = ARMMMUIdx_E20_2;   // VHE
    } else {
        idx = ARMMMUIdx_E2;      // 非 VHE 的独立 EL2 regime
    }
```

在这个模式下：

- EL2 使用 `ARMMMUIdx_E2`
- `regime_ttbr()` 对 `E2` 会落到通用路径，取 `TTBR0_EL2`（`ptw.c:241-245`）
- `regime_tcr()` 对 `E2` 取 `TCR_EL2`（`internals.h:1063-1081` 的默认分支）
- 这是 **单阶段翻译**，不会自动经过 `Stage2`

因此 `E2` 更像“宿主 Hypervisor 自己的地址空间”，而不是 Guest 两阶段翻译的一部分。

### 3.2 VHE 模式（`ARMMMUIdx_E20_0` / `ARMMMUIdx_E20_2`）

VHE（Virtualization Host Extensions）的判定点在 `helper.c:9966-9993`：

- EL0：只有当 `HCR_E2H | HCR_TGE` **同时为 1** 时，才选 `E20_0`（`helper.c:9969-9977`）
- EL2：只要 `HCR_E2H=1`，就选 `E20_2` / `E20_2_PAN`（`helper.c:9987-9993`）

这反映了 ARM 的 `ELIsInHost()` 语义：

- **EL0 进入 host 语义** 需要 `E2H+TGE`
- **EL2 自身进入 host 语义** 只看 `E2H`

`el_is_in_host()` 的注释直接写明它对应 ARM 伪代码 `ELIsInHost()`（`helper.c:3901-3912`）。

VHE 的意义不是“多一层 Stage-2”，而是让 host OS 更像运行在 EL2 的 EL1：

- EL0 与 EL2 共用 `EL2&0` regime
- 页表控制寄存器从 `TCR_EL2` / `TTBR*_EL2` 来
- QEMU 的系统寄存器框架支持寄存器重定向；`cpregs.h:968-970` 明说：在 `E2H` 下，EL2 访问 EL0/EL1 寄存器时可以重定向到 EL2 对应寄存器

可用下面的图理解：

```
非 VHE:
  EL0/EL1 --> E10_x --> TCR_EL1 / TTBR0_EL1 / TTBR1_EL1
  EL2     --> E2    --> TCR_EL2 / TTBR0_EL2

VHE:
  EL0     --> E20_0 --> TCR_EL2 / TTBR0_EL2 / TTBR1_EL2
  EL2     --> E20_2 --> TCR_EL2 / TTBR0_EL2 / TTBR1_EL2
```

### 3.3 EL2 对 Guest 的控制

`HCR_EL2` 的位定义位于 `cpu.h:1695-1754`。对本文主题最重要的位有：

- `HCR_VM`：打开 Stage-2
- `HCR_PTW`：限制页表遍历接触 Device memory
- `HCR_DC`：关闭 S1、强制开启 S2 语义
- `HCR_FWB`：改变 S2 描述符缓存属性解释
- `HCR_E2H`：VHE
- `HCR_TGE`：Trap General Exceptions，影响 EL0/EL1 像 host 一样行为
- `HCR_FMO/IMO/AMO`：FIQ/IRQ/SError 路由到 EL2
- `HCR_NV/NV1/NV2`：嵌套虚拟化

`do_hcr_write()` 把 MMU 语义变化的位作为 TLB flush 条件（`helper.c:3750-3763`）：

```c
// helper.c:3751-3760（简化）
// 这些位改变 MMU 建模语义：
// VM/PTW/DC/DCT/FWB/NV/NV1
if ((old ^ new) & (HCR_VM | HCR_PTW | HCR_DC | HCR_DCT |
                   HCR_FWB | HCR_NV | HCR_NV1)) {
    tlb_flush(...);
}
```

中断路由方面，`helper.c:8409-8416` 指出：对中断路由目标 EL 的选择来说，`TGE` 与 `AMO/IMO/FMO` 都会“把中断推向 EL2”。这说明 QEMU 并不是只在异常入口时考虑 TGE，而是把它纳入整体 EL2 路由判定。

---

## 4. Secure EL2 — FEAT_SEL2

### 4.1 使能条件

Secure EL2 的使能有三层条件：

#### （1）CPU 必须声明支持 FEAT_SEL2

在 `tcg/cpu64.c:1269-1278`，QEMU 给 AArch64 CPU ID 寄存器打上：

```c
// tcg/cpu64.c:1273-1275
t = FIELD_DP64(t, ID_AA64PFR0, SVE, 1);
t = FIELD_DP64(t, ID_AA64PFR0, SEL2, 1);   // FEAT_SEL2
t = FIELD_DP64(t, ID_AA64PFR0, DIT, 1);
```

即 `ID_AA64PFR0_EL1.SEL2 = 1`。

#### （2）EL3 必须允许 Secure EL2

`SCR_EL3.EEL2` 位定义在 `cpu.h:1774`：

```c
#define SCR_EEL2 (1ULL << 18)   // cpu.h:1774
```

#### （3）运行时谓词必须成立

QEMU 用 `arm_is_el2_enabled_secstate()` 统一表达“当前安全状态是否真的有 EL2”这一架构语义（`cpu.h:2228-2239`）：

```c
// cpu.h:2228-2234（简化）
return arm_feature(env, ARM_FEATURE_EL2)
       && (space != ARMSS_Secure || (env->cp15.scr_el3 & SCR_EEL2));
```

解释如下：

- 对 NonSecure/Realm：只要 CPU 有 EL2 即可
- 对 Secure：除了 CPU 支持 EL2，还必须 `SCR_EL3.EEL2=1`

### 4.2 `scr_write()` 如何验证 `SCR_EEL2`

`scr_write()` 在 `helper.c:712-836` 根据 CPU 特性动态构建 `valid_mask`。只有 `aa64_sel2` 存在时，才允许写 `SCR_EEL2`（`helper.c:741-743`）：

```c
// helper.c:741-743
if (cpu_isar_feature(aa64_sel2, cpu)) {
    valid_mask |= SCR_EEL2;
}
```

这意味着：

- 不支持 FEAT_SEL2 的 CPU，`SCR_EEL2` 会被掩掉
- 支持 FEAT_SEL2 的 CPU，`SCR_EEL2` 才进入有效写掩码

### 4.3 `HCR_EL2` 在 Secure 状态下何时生效

`arm_hcr_el2_eff_secstate()` 是理解 SEL2 的总开关（`helper.c:3818-3881`）。其一开始就做判定：

```c
// helper.c:3824-3840（简化）
if (!arm_is_el2_enabled_secstate(env, space)) {
    return 0;
}
```

所以如果当前是在 Secure 状态但 `SCR_EEL2=0`：

- 读写 `HCR_EL2` 的寄存器值仍存在
- 但 **行为效果全部视为 0**
- `E2H/VM/PTW/FWB/NV...` 都不会生效

这正是 Secure EL2 的“软件可见”和“架构生效”之间的分层。

### 4.4 Secure EL2 与 NS EL2 的共存

`mmuidx.h:59-60` 已经给出设计结论：

> 不能把 Secure Stage-2 和 NonSecure Stage-2 折叠，因为 Secure EL2 会同时使用它们。

QEMU 在 `arm_mmu_idx_to_security_space()` 中明确编码了这一点（`ptw.c:3882-3905`）：

- `ARMMMUIdx_Stage2`：在 S-EL2 环境下被强制解释为 **NonSecure**（`ptw.c:3882-3890`）
- `ARMMMUIdx_Stage2_S`：固定解释为 **Secure**（`ptw.c:3899-3905`）

也就是说：

```
Secure EL2 不是“只有 Secure Stage-2”
而是“同时具备 Secure Stage-2 与 NonSecure Stage-2 的管理权”
```

### 4.5 Secure EL2 独有寄存器

`helper.c:4256-4275` 注册了 FEAT_SEL2 独有寄存器：

- `VSTTBR_EL2`：Secure Stage-2 页表基址
- `VSTCR_EL2`：Secure Stage-2 翻译控制

并通过 `sel2_access()` 保证只有 EL3 或 Secure 状态能访问（`helper.c:4256-4263`）。

```c
// helper.c:4265-4275（简化）
{ .name = "VSTTBR_EL2", .accessfn = sel2_access,
  .fieldoffset = offsetof(..., cp15.vsttbr_el2) }
{ .name = "VSTCR_EL2",  .accessfn = sel2_access,
  .fieldoffset = offsetof(..., cp15.vstcr_el2) }
```

### 4.6 `VSTCR_EL2.SW` 与 `VTCR_EL2.NSW`

`ptw_idx_for_stage_2()` 在 Secure 状态下决定 **Stage-2 页表遍历本身从哪个物理空间读描述符**（`ptw.c:215-221`）：

- 若当前是 `Stage2_S`：看 `VSTCR_EL2.SW`
- 若当前是 `Stage2`：看 `VTCR_EL2.NSW`

```c
// ptw.c:216-221（简化）
if (stage2idx == ARMMMUIdx_Stage2_S) {
    s2walk_secure = !(env->cp15.vstcr_el2 & R_VSTCR_SW_MASK);
} else {
    s2walk_secure = !(env->cp15.vtcr_el2 & R_VTCR_NSW_MASK);
}
return s2walk_secure ? ARMMMUIdx_Phys_S : ARMMMUIdx_Phys_NS;
```

这两个位的意义可总结为：

- `VSTCR_EL2.SW=0`：Secure Stage-2 的页表遍历从 Secure 物理空间取描述符
- `VSTCR_EL2.SW=1`：Secure Stage-2 的页表遍历转去 NonSecure 物理空间
- `VTCR_EL2.NSW=0`：NonSecure Stage-2 的页表遍历在 S-EL2 下仍可落在 Secure 物理空间
- `VTCR_EL2.NSW=1`：强制走 NonSecure 物理空间

再进一步，`get_phys_addr_twostage()` 在最终合成输出 PA 安全属性时，还会结合 `VSTCR_EL2.SA/SW` 与 `VTCR_EL2.NSA/NSW`（`ptw.c:3646-3655`），说明 **表遍历安全空间** 与 **最终输出 PA 安全属性** 是两个不同层次的问题。

### 4.7 Secure Stage-2 的 TCR 不是完全独立的

`regime_tcr()` 在 `internals.h:1063-1080` 为 `Stage2_S` 做了特殊合成：

```c
// internals.h:1068-1079（简化）
if (mmu_idx == ARMMMUIdx_Stage2_S) {
    uint64_t v = env->cp15.vstcr_el2 & ~VTCR_SHARED_FIELD_MASK;
    v |= env->cp15.vtcr_el2 & VTCR_SHARED_FIELD_MASK;
    return v;
}
```

也就是说，Secure Stage-2 并不是简单地“只看 `VSTCR_EL2`”，而是：

- `VSTCR_EL2` 提供 Secure 独有字段
- `VTCR_EL2` 提供共享字段
- QEMU 在软件层合成为一个统一的 VTCR 格式值，供后续 `get_phys_addr_lpae()` 使用

---

## 5. Stage-1 翻译流程

### 5.1 翻译入口选择：`arm_mmu_idx_el()`

`arm_mmu_idx_el()` 是“当前 EL / HCR 状态 → 翻译体制索引”的第一跳（`helper.c:9957-10008`）。简化后如下：

```c
// helper.c:9967-10002（简化）
switch (el) {
case 0:
    if ((hcr & (HCR_E2H | HCR_TGE)) == (HCR_E2H | HCR_TGE))
        idx = ARMMMUIdx_E20_0;   // VHE：EL0 进入 EL2&0 regime
    else
        idx = ARMMMUIdx_E10_0;   // 普通：EL0 进入 EL1&0 regime
    break;
case 1:
    idx = arm_pan_enabled(env) ? ARMMMUIdx_E10_1_PAN : ARMMMUIdx_E10_1;
    break;
case 2:
    idx = (arm_hcr_el2_eff(env) & HCR_E2H) ? ARMMMUIdx_E20_2 : ARMMMUIdx_E2;
    break;
case 3:
    return ARMMMUIdx_E3;
}
```

注意：

- EL1 自己不会直接返回 `Stage1_E1`，因为 `Stage1_E1` 只是慢路径拆分索引
- 真正执行普通访存时，EL1 用的是 `E10_1`，它表示“**若开启虚拟化，则最终应缓存 S1+S2 合成结果**”
- 只有在 `get_phys_addr_nogpc()` 进入 `do_twostage` 后，才把 `E10_1` 临时改写成 `Stage1_E1`

### 5.2 LPAE 页表遍历：`get_phys_addr_lpae()`

`get_phys_addr_lpae()` 是整个 PTW 的核心，入口在 `ptw.c:1859`。它完成的事情包括：

1. 解析当前 regime 的 `TCR` / `TTBR`
2. 调 `aa64_va_parameters()` 解码 `T0SZ/T1SZ/TG/...`
3. 计算起始 level、stride、输入/输出地址大小
4. 逐级加载描述符
5. 解析 block/page/table 描述符
6. 生成输出 PA / 权限 / 属性 / 安全空间

关键初始化片段如下：

```c
// ptw.c:1875-1900（简化）
uint64_t tcr = regime_tcr(env, mmu_idx);
param = aa64_va_parameters(env, address, mmu_idx,
                           access_type != MMU_INST_FETCH,
                           !arm_el_is_aa64(env, 1));
ttbr = regime_ttbr(env, mmu_idx, param.select);
```

`ptw.c:1979-2036` 完成起始级别与 TTBR 基址处理；`ptw.c:2056` 开始进入每级遍历循环。这里最重要的不是“按几级页表走”，而是：QEMU 把 **Stage-1 与 Stage-2 走表的共性** 尽量复用到同一个 LPAE walker 中。

### 5.3 TCR 参数解析的实际意义

虽然 `get_phys_addr_lpae()` 注释里说 QEMU 忽略部分共享属性和 ASID（`ptw.c:1964-1970`），但它对以下字段仍高度敏感：

- `T0SZ/T1SZ`：决定输入地址宽度与起始 level
- `TG0/TG1`：决定 granule size 与 stride
- `PS`：决定输出地址上限
- `DS`：影响 LPA2 / 输出地址宽度
- `EPD`：关闭页表遍历时直接 Translation fault（`ptw.c:1979-1984`）

所以，对 Hypervisor 来说，`VTCR_EL2` / `VSTCR_EL2` 并不是“只换了 TTBR”，而是完整影响了二级页表 walker 的几何结构。

### 5.4 `NSTable` 降级机制

Stage-1 Secure regime 的一个关键行为，是上一级表的 `NSTable` 可以把后续表遍历及输出地址空间降级为 NonSecure。QEMU 在 `ptw.c:2061-2077` 直接实现：

```c
// ptw.c:2066-2076（简化）
if (ptw->cur_space == ARMSS_Secure &&
    !regime_is_stage2(mmu_idx) &&
    extract32(tableattrs, 4, 1)) {
    ptw->in_ptw_idx += 1;       // Stage2_S -> Stage2 或 Phys_S -> Phys_NS
    ptw->cur_space = ARMSS_NonSecure;
}
```

这段代码体现两条原则：

- **可以降级 Secure → NonSecure**
- **绝不能升级 NonSecure → Secure/Realm**

后者在 `get_phys_addr_nogpc()` 入口注释也写明了（`ptw.c:3670-3677`）。

### 5.5 描述符通过 Stage-2：S1 PTW through S2

这是虚拟化里最容易被忽略、但最重要的细节之一。Guest 页表本身通常位于 Guest IPA 空间，所以 **加载 S1 描述符时，也要先把“页表地址”做 Stage-2 翻译**。

QEMU 用 `S1_ptw_translate()` 处理这个问题（`ptw.c:645-730`）。如果 `ptw->in_ptw_idx` 是 `Stage2` / `Stage2_S`，则描述符地址会先走 S2：

```c
// ptw.c:659-670（debug path 简化）
S1Translate s2ptw = {
    .in_mmu_idx = s2_mmu_idx,
    .in_ptw_idx = ptw_idx_for_stage_2(env, s2_mmu_idx),
    .in_space = s2_space,
    .in_debug = true,
    .in_prot_check = PAGE_READ,
};
if (get_phys_addr_gpc(env, &s2ptw, addr, MMU_DATA_LOAD, 0, &s2, fi)) {
    goto fail;
}
```

TCG 正常路径里，则通过 `probe_access_full_mmu()` 用 `s2_mmu_idx` 去查询/填充 TLB（`ptw.c:680-699`）。

这就是“**S1 PTW through S2**”：

- S1 的“数据地址”是 guest 自己真正要访问的 VA
- S1 的“描述符地址”也处于 IPA 空间
- 所以 S1 的 descriptor load 必须再嵌套一次 S2 翻译

### 5.6 Fault 传播与 `fi->s1ptw`

当 S1 PTW 过程中因为 S2 失败而出错，QEMU 会设置：

- `fi->stage2 = true`
- `fi->s1ptw = true`
- `fi->s1ns = fault_s1ns(...)`

见 `ptw.c:710-715` 与 `ptw.c:722-730`。这意味着 Fault 不是“普通 Guest 数据访问失败”，而是“**在做 Stage-1 页表遍历时，页表本身访问失败**”。

另外，`HCR_PTW` 与 Device memory 的交互也在这里完成：若 S1 walk 触及被 S2 标成 Device 的内存，则生成 Permission fault（`ptw.c:702-716`）。

---

## 6. Stage-2 翻译流程

### 6.1 何时触发 Stage-2

触发点在 `get_phys_addr_nogpc()`（`ptw.c:3709-3727`）：

```c
// ptw.c:3709-3727（简化）
case ARMMMUIdx_E10_0:     s1_mmu_idx = ARMMMUIdx_Stage1_E0; goto do_twostage;
case ARMMMUIdx_E10_1:     s1_mmu_idx = ARMMMUIdx_Stage1_E1; goto do_twostage;
case ARMMMUIdx_E10_1_PAN: s1_mmu_idx = ARMMMUIdx_Stage1_E1_PAN;
do_twostage:
    ptw->in_mmu_idx = s1_mmu_idx;
    if (arm_feature(env, ARM_FEATURE_EL2) &&
        !regime_translation_disabled(env, ARMMMUIdx_Stage2, ptw->in_space)) {
        return get_phys_addr_twostage(...);
    }
```

只有满足以下条件，QEMU 才会做两阶段：

- 当前 regime 属于 `E10_0/E10_1/E10_1_PAN` 这类 guest EL1&0 访问
- CPU 具备 EL2
- `Stage-2` 没有被认为关闭

`regime_translation_disabled()` 对 `Stage2/Stage2_S` 的判断是：`(HCR_DC | HCR_VM) == 0` 才算关闭（`ptw.c:276-280`）。也就是说：

- `HCR_VM=1`：标准 Stage-2 打开
- `HCR_DC=1`：即使 `VM=0`，也按 Stage-2 有效处理

### 6.2 两阶段翻译完整流程：`get_phys_addr_twostage()`

`get_phys_addr_twostage()`（`ptw.c:3554-3658`）可以概括为四步。

#### 第一步：先做 Stage-1

```c
ret = get_phys_addr_nogpc(env, ptw, address, access_type,
                          memop, result, fi);   // ptw.c:3568-3569
```

此时 `ptw->in_mmu_idx` 已被上层改成 `Stage1_E0/E1/...`，因此拿到的是 IPA。

#### 第二步：由 S1 输出决定 S2 索引

```c
// ptw.c:3576-3583
ipa = result->f.phys_addr;
ipa_secure = result->f.attrs.secure;
ipa_space = result->f.attrs.space;
ptw->in_s1_is_el0 = ptw->in_mmu_idx == ARMMMUIdx_Stage1_E0;
ptw->in_mmu_idx = ipa_secure ? ARMMMUIdx_Stage2_S : ARMMMUIdx_Stage2;
ptw->in_space = ipa_space;
ptw->in_ptw_idx = ptw_idx_for_stage_2(env, ptw->in_mmu_idx);
```

关键点：**S2 索引不是由当前 CPU 所在 EL 决定，而是由 S1 产出的 IPA 安全属性决定。**

#### 第三步：再做 Stage-2

```c
ret = get_phys_addr_nogpc(env, ptw, ipa, access_type,
                          memop, result, fi);   // ptw.c:3595-3597
```

这一步把 IPA 变成最终 PA。

#### 第四步：合并 S1 与 S2 结果

- 权限：`result->f.prot = s1_prot & result->s2prot`（`ptw.c:3599-3600`）
- 页大小：若任一阶段小于 `TARGET_PAGE_SIZE` 则返回 0；否则取较大者用于 TLB 失效粒度（`ptw.c:3607-3623`）
- 缓存属性：`combine_cacheattrs()`（`ptw.c:3625-3641`）

### 6.3 Stage-2 TTBR 和 TCR

#### TTBR 选择：`regime_ttbr()`

`ptw.c:233-245`：

- `Stage2` → `VTTBR_EL2`
- `Stage2_S` → `VSTTBR_EL2`
- 其他 regime → `TTBR0/1_EL[regime_el]`

```c
// ptw.c:235-240
if (mmu_idx == ARMMMUIdx_Stage2)   return env->cp15.vttbr_el2;
if (mmu_idx == ARMMMUIdx_Stage2_S) return env->cp15.vsttbr_el2;
```

#### TCR 选择：`regime_tcr()`

`internals.h:1063-1081`：

- `Stage2` → `VTCR_EL2`
- `Stage2_S` → `VSTCR_EL2` 与 `VTCR_EL2` 共享字段合成值
- 其他 regime → `TCR_EL[el]`

这使得同一个 `get_phys_addr_lpae()` walker 能无感知地处理 Secure Stage-2。

### 6.4 Stage-2 PTW 的物理空间

`ptw_idx_for_stage_2()`（`ptw.c:193-225`）是 Secure EL2 最值得单独读的函数之一：

| 当前 EL3 以下安全空间 | Stage-2 PTW 读描述符使用的物理空间 |
|----------------------|------------------------------------|
| `ARMSS_NonSecure` | `Phys_NS` |
| `ARMSS_Realm` | `Phys_Realm` |
| `ARMSS_Secure` + `Stage2_S` | `VSTCR_EL2.SW ? Phys_NS : Phys_S` |
| `ARMSS_Secure` + `Stage2` | `VTCR_EL2.NSW ? Phys_NS : Phys_S` |

也就是说，在 S-EL2 中：

- `Stage2_S` 的 walker 本身未必从 Secure memory 读表
- `Stage2` 的 walker 也未必从 NS memory 读表

**“S2 针对哪个 IPA 空间”** 与 **“S2 页表描述符位于哪个物理空间”** 是分离建模的。

### 6.5 Stage-2 权限与属性

#### `S2AP` → `prot`

`get_S2prot()` 在 `ptw.c:1343-1382` 中把 S2AP/XN 转成 QEMU `PAGE_*` 权限。基本规则是：

- `S2AP[0]` → `PAGE_READ`
- `S2AP[1]` → `PAGE_WRITE`
- 执行权限由 `xn` 与 `s1_is_el0` 共同决定

对支持 `FEAT_TTS2UXN` 的 CPU，QEMU 用 4 种 `xn` 组合细分 EL0/EL1 执行许可（`ptw.c:1354-1373`）。这就是 `in_s1_is_el0` 必须传给第二阶段的原因。

#### 间接权限：`FEAT_S2PIE`

`get_S2prot_indirect()`（`ptw.c:1384-1420`）通过 `S2PIR_EL2` 查询 16 项权限表。函数同时返回：

- `result->f.prot`：用于 page table walk 的 `ttw` 权限
- 返回值：当前 `s1_is_el0` 所对应的访问权限

这表明 QEMU 已经把 Stage-2 间接权限模型接入主 walker，而不是事后 patch。

#### `HCR_FWB`

`combine_cacheattrs()` 在 `ptw.c:3439-3444` 里根据 `HCR_FWB` 选择：

- `combined_attrs_fwb()`：FWB 语义
- `combined_attrs_nofwb()`：传统“取弱者”语义

所以 `HCR_FWB` 实际改变的是 **Stage-2 描述符属性位的解释规则**，不是单纯的缓存开关。

---

## 7. S1+S2 结果合并

### 7.1 权限合并：取交集

`get_phys_addr_twostage()` 中最核心的一行是（`ptw.c:3599-3600`）：

```c
result->f.prot = s1_prot & result->s2prot;
```

含义非常直接：

- S1 允许，但 S2 禁止 ⇒ 最终禁止
- S2 允许，但 S1 禁止 ⇒ 最终禁止
- 只有两边都允许，最终才允许

这符合 ARM 架构对两阶段权限组合的本意：**Stage-2 不会放宽 Stage-1，Stage-1 也不会越过 Stage-2。**

### 7.2 页面大小合并：为 TLB 失效服务，而不是为实际映射放大

`ptw.c:3607-3623` 的注释非常值得细读。QEMU 的策略是：

- 若任一阶段页大小小于 `TARGET_PAGE_SIZE` ⇒ 返回 `lg_page_size = 0`，表示不要进 TLB
- 否则取 `max(S1_page_size, S2_page_size)`

这看起来像“取大值”，但注释说明其目的不是扩大单条 TLB 覆盖范围，而是 **让大页失效（invalidation）语义正确**。因为真正的 core TLB 仍不会创建超过 `TARGET_PAGE_SIZE` 的实体项。

### 7.3 缓存属性合并

#### （1）共享属性：取更强者

`combine_cacheattrs()` 在 `ptw.c:3427-3437`：

- 任一方 Outer Shareable ⇒ 结果 Outer Shareable
- 否则任一方 Inner Shareable ⇒ 结果 Inner Shareable
- 否则 Non-shareable

可概括为：**共享性取更强者**。

#### （2）`HCR.DC`：强制 S1 视为 WB-RWA

`ptw.c:3626-3639`：若 `HCR_DC=1`，QEMU 把第一阶段属性强制改写为：

- Normal
- Inner WB Read-Allocate Write-Allocate
- Outer WB Read-Allocate Write-Allocate
- Non-shareable

这完全对应架构定义的 “S1 disabled output under HCR.DC” 语义。

#### （3）`HCR.FWB`：S2 优先的合并规则

`combine_cacheattrs()` 在 `ptw.c:3439-3444` 根据 `HCR_FWB` 选择 FWB 或 non-FWB 路径。FWB 模式下，S2 对最终内存类型的主导权更强。

#### （4）非 FWB：取弱者

若 `HCR_FWB=0`，则走 `combined_attrs_nofwb()`，含义可概括为：

- Device 比 Normal 更“弱”，最终会落向 Device
- Non-cacheable 比 Write-back 更“弱”，最终会落向较弱缓存性

#### （5）Device / NC 特殊共享性修正

`ptw.c:3446-3456` 还补了一条：

- 若结果为 Device memory，或者 Normal NC/NC，则共享性强制视作 Outer Shareable

### 7.4 合并过程的时序图

```
Stage-1 完成
  ├─ 保存 S1: prot / lg_page_size / cacheattrs / guarded
  ├─ 以 IPA 进入 Stage-2
  ├─ 得到 S2: s2prot / lg_page_size / cacheattrs
  ├─ prot = S1 & S2
  ├─ lg_page_size = merge_for_tlb_invalidation()
  ├─ cacheattrs = combine_cacheattrs(HCR, S1, S2)
  └─ guarded = S1.guarded
```

其中 `guarded`（BTI GP 信息）只保留 Stage-1 结果，Stage-2 不提供新的 BTI 信息（`ptw.c:3643-3644`）。

---

## 8. S1Translate 上下文结构

### 8.1 字段详解

`S1Translate` 定义于 `ptw.c:22-90`。其中与虚拟化关系最大的字段如下：

| 字段 | 含义 | 关键点 |
|------|------|--------|
| `in_mmu_idx` | 当前 walk 使用哪个翻译 regime | 决定 TTBR/TCR/SCTLR/HCR 视角 |
| `in_ptw_idx` | 描述符加载用哪个 mmuidx | 可能是 `Stage2*`，也可能是 `Phys_*` |
| `in_space` | 本次翻译所属安全空间 | 与 `in_mmu_idx` 一起定义架构 regime |
| `cur_space` | 当前实际安全空间 | 可被 `NSTable` 降级 |
| `in_s1_is_el0` | 当前若在做 S2，S1 是否是 EL0 访问 | 影响 `get_S2prot()` 的 XN 解释 |
| `in_prot_check` | 需要检查的 `PAGE_*` 权限位 | AT/debug 可抑制 |
| `out_space` | walker 输出地址空间 | 从 S1/S2 描述符逐步传播 |
| `out_phys` | 当前 walker 最终物理地址 | 供 descriptor load 或最终访问使用 |

### 8.2 为什么 `in_mmu_idx` 与 `in_ptw_idx` 必须分开

这是 QEMU PTW 抽象最精妙的地方。

- `in_mmu_idx`：回答“**我要按谁的页表格式和寄存器解释地址？**”
- `in_ptw_idx`：回答“**我真正去读页表描述符时，访存应该走哪个地址空间？**”

例如一个 Secure guest 的 S1 遍历：

- `in_mmu_idx = Stage1_E1`
- `in_space = ARMSS_Secure`
- `in_ptw_idx = Stage2_S`

此时说明：

- S1 描述符的格式、权限、输出空间，都按 Secure EL1&0 stage-1 解释
- 但 **读取这些描述符的访存**，必须再走 Secure Stage-2

### 8.3 `NSTable` 如何影响上下文

`ptw.c:32-34` 和 `ptw.c:54-56` 已在结构注释中写明：

- `in_ptw_idx` 可被 `NSTable` 从 `Stage2_S` 降成 `Stage2`
- `cur_space` 可被 `NSTable` 从 Secure 降成 NonSecure

这意味着一个 walk 在进行过程中，其 **安全上下文是可变的**；但这个变化是单向的（只能降级）。

### 8.4 上下文在递归调用中的传递

在 `get_phys_addr_twostage()` 中，`S1Translate` 被原地复用：

1. 先用 `Stage1_E*` 做 S1
2. 再把同一个 `ptw` 对象改写成 `Stage2/Stage2_S`
3. 然后做 S2

这比“新建两个完全独立对象”更接近架构语义，因为：

- `in_s1_is_el0`
- `in_space`
- `cur_space`
- `fi->s2addr`

这些信息天然需要跨阶段传递。

---

## 9. 安全空间与翻译体制的交互

### 9.1 四种安全空间

`ARMSecuritySpace` 定义在 `arm-security.h:18-23`：

```c
typedef enum ARMSecuritySpace {
    ARMSS_Secure    = 0,
    ARMSS_NonSecure = 1,
    ARMSS_Root      = 2,
    ARMSS_Realm     = 3,
} ARMSecuritySpace;
```

`arm_space_is_secure()` 把 `Secure` 与 `Root` 都视为“pre-v9 意义上的 secure”（`arm-security.h:25-29`）。

运行时判定当前安全空间则由 `arm_security_space()` / `arm_security_space_below_el3()` 提供（`helper.c:10131-10187`）：

- 无 EL3 ⇒ 默认 `NonSecure`
- 当前在 AArch64 EL3 且有 RME ⇒ `Root`
- `SCR_EL3.NS=0` ⇒ `Secure`
- `SCR_EL3.NS=1 && SCR_EL3.NSE=1` ⇒ `Realm`
- 其余 ⇒ `NonSecure`

### 9.2 `mmu_idx` 到安全空间的初始映射

`arm_mmu_idx_to_security_space()`（`ptw.c:3858-3929`）为每个 `mmu_idx` 赋一个初始安全空间：

- `E10_* / E20_* / E2 / Stage1_*` ⇒ `arm_security_space_below_el3(env)`
- `Stage2` ⇒ 若当前 below EL3 是 Secure，则转成 `NonSecure`（`ptw.c:3882-3890`）
- `Stage2_S / Phys_S` ⇒ `Secure`
- `Phys_Root / Phys_Realm` ⇒ `Root / Realm`

这说明 **安全空间不是从页表走出来之后才决定的**，而是在进入 PTW 之前就有一层 regime 级别的初始值。

### 9.3 IPA 安全属性传播

`get_phys_addr_nogpc()` 每次 walk 开始时都先做两件事（`ptw.c:3675-3677`）：

```c
ptw->cur_space = ptw->in_space;
result->f.attrs.space = ptw->in_space;
result->f.attrs.secure = arm_space_is_secure(ptw->in_space);
```

含义是：

- 若后续没有任何 descriptor 改写，默认输出空间继承输入空间
- 之后才允许由 `NSTable` / `NS` / `NSE` / `SA/NSA/SW/NSW` 去修改输出空间

### 9.4 `NSTable/NS` 只能降级，不能升级

`get_phys_addr_nogpc()` 的注释（`ptw.c:3670-3674`）是整个安全属性传播的总原则：

> 页表项可以把 Secure 降为 NonSecure，但不能把 NonSecure 升级为 Secure 或 Realm。

这也是为什么 `ptw.c:2066-2076` 只处理 Secure→NonSecure 的 `NSTable` 情况。

### 9.5 Secure EL2 下的双 Stage-2 场景

在 FEAT_SEL2 下，最值得画图的是这两个场景：

#### 场景 A：Secure Guest 经过 Secure EL2

```
VA(S-EL1/S-EL0)
  └─ S1: Secure EL1&0 regime
       └─ 输出 IPA, secure=1
            └─ 选中 Stage2_S
                 └─ VSTTBR_EL2 / VSTCR_EL2
                      └─ 最终 PA 可能仍受 SA/SW 影响
```

#### 场景 B：Non-Secure Guest 由 Secure EL2 托管

```
VA(NS-EL1/NS-EL0)
  └─ S1: NonSecure EL1&0 regime
       └─ 输出 IPA, secure=0
            └─ 选中 Stage2
                 └─ VTTBR_EL2 / VTCR_EL2
                      └─ 页表遍历空间由 VTCR.NSW 决定
```

这正是 `Stage2_S` 与 `Stage2` 必须并存的直接原因。

---

## 10. 嵌套虚拟化（NV/NV1/NV2）概览

虽然本文焦点是 Stage-1/2，但 QEMU 已经把嵌套虚拟化相关位纳入 MMU 语义。

### 10.1 `HCR_NV/NV1/NV2`

位定义位于 `cpu.h:1736-1740`：

- `HCR_NV`
- `HCR_NV1`
- `HCR_NV2`

`arm_hcr_el2_nvx_eff()` 返回有效 NVx 位组合（`helper.c:3891-3898`）。

### 10.2 NV1 为什么会影响页表遍历

`do_hcr_write()` 把 `HCR_NV` / `HCR_NV1` 列为“改变 MMU setup”的位（`helper.c:3757-3760`），并在变化时 flush TLB。这不是偶然，因为 `get_phys_addr_lpae()` 在进入 walker 早期就缓存了 `in_nv1`（`ptw.c:1893-1900`）：

```c
if (el == 1) {
    uint64_t hcr = arm_hcr_el2_eff_secstate(env, ptw->in_space);
    ptw->in_nv1 = (hcr & (HCR_NV | HCR_NV1)) == (HCR_NV | HCR_NV1);
}
```

说明 NV1 会影响某些 descriptor 位的解释，因此必须在 PTW 级别生效。

### 10.3 Guest Hypervisor 的 EL2 访问如何被宿主捕获

QEMU 的系统寄存器框架在很多 EL2 寄存器上设置了 `nv2_redirect_offset`，例如：

- `VTCR_EL2`（`helper.c:4155-4160`）
- `VTTBR_EL2`（`helper.c:4167-4171`）
- `VNCR_EL2`（`helper.c:5693-5699`）

`cpregs.h:963-966` 注释说明了这个字段的意义：在 FEAT_NV2 下，访问可以被重定向到 `VNCR_EL2 + offset` 对应的内存页。`VNCR_EL2` 本身通过 `vncr_write()` 维护，低 12 位清零以保证对齐（`helper.c:5680-5691`）。

这套机制说明：

- QEMU 不只是“trap 一条 HVC”
- 它还为 guest hypervisor 的 EL2 系统寄存器视图提供了内存化后端

本文不展开 NV 的完整执行流，但要知道它已进入 MMU/PTW 的语义范围。

---

## 11. 完整调用链图

### 11.1 EL1&0 两阶段翻译调用链

`arm_cpu_tlb_fill_align()` 在 `tlb_helper.c:331-379` 中对 TLB miss 调 `get_phys_addr()`，后者再进入 `get_phys_addr_gpc()` / `get_phys_addr_nogpc()`。

```
arm_cpu_tlb_fill_align()                    [tlb_helper.c:331]
  └── get_phys_addr()                      [ptw.c:3931]
      └── get_phys_addr_gpc()              [ptw.c:3800]
          └── get_phys_addr_nogpc()        [ptw.c:3661]
              ├── case E10_0/E10_1/E10_1_PAN -> do_twostage
              │   └── get_phys_addr_twostage()           [ptw.c:3554]
              │       ├── get_phys_addr_nogpc(Stage1_E0/E1)   ← S1
              │       │   └── get_phys_addr_lpae()            [ptw.c:1859]
              │       │       └── S1_ptw_translate()          [ptw.c:645]
              │       │           └── get_phys_addr_gpc(Stage2/Stage2_S)
              │       │               └── get_phys_addr_nogpc()
              │       └── get_phys_addr_nogpc(Stage2/Stage2_S) ← S2
              │           └── get_phys_addr_lpae()
              └── case E2/E20_2 -> 单阶段
                  └── get_phys_addr_lpae()
```

### 11.2 Secure EL2 场景调用链

```
Secure Guest VA
  └─ get_phys_addr_nogpc(E10_1, in_space=Secure)
      └─ do_twostage
          ├─ Stage-1: in_mmu_idx = Stage1_E1
          │   └─ in_ptw_idx 初始为 Stage2_S            [ptw.c:3691-3696]
          │       └─ 若上级表置 NSTable
          │           ├─ Stage2_S -> Stage2            [ptw.c:2073-2075]
          │           └─ cur_space Secure -> NonSecure [ptw.c:2075-2076]
          ├─ S1 输出 IPA + attrs.space/secure
          └─ Stage-2:
              ├─ IPA secure=1 -> Stage2_S
              └─ IPA secure=0 -> Stage2
```

这张图说明，Secure EL2 场景里不仅“最终 S2 索引”是动态的，连 **S1 遍历描述符时所用的 `in_ptw_idx`** 也是动态可降级的。

### 11.3 `get_phys_addr_twostage()` 的逻辑骨架

```c
// ptw.c:3554-3658（高度简化）
S1(address) -> IPA
save(s1_prot, s1_lgpgsz, s1_cacheattrs)
select_s2_idx(ipa_secure ? Stage2_S : Stage2)
S2(IPA) -> PA
result.prot = s1_prot & s2prot
result.lg_page_size = merge_page_size(...)
result.cacheattrs = combine_cacheattrs(hcr, s1, s2)
if (in_space == ARMSS_Secure)
    result.attrs.secure/space = recompute_from_SA_SW_NSA_NSW(...)
```

---

## 12. 关键数据结构速查表

| 结构体/枚举 | 文件 | 作用 |
|------------|------|------|
| `ARMMMUIdx` | `mmuidx.h:137-198` | 所有翻译体制与物理空间索引 |
| `ARMMMUIdxBit` | `mmuidx.h:207-235` | TLB flush 的位图索引 |
| `ARMSecuritySpace` | `arm-security.h:18-23` | Secure / NS / Root / Realm |
| `S1Translate` | `ptw.c:22-90` | 一次 PTW 的输入/中间/输出上下文 |
| `GetPhysAddrResult` | `ptw.c` 相关使用点 | PTW 输出：PA、prot、attrs、cacheattrs |
| `ARMMMUFaultInfo` | `internals.h`（参见相关文档） | Translation/Permission/AddressSize 等故障信息 |
| `ARMCacheAttrs` | `ptw.c:3413-3464` 使用 | 缓存属性与共享性合并载体 |
| `ARMCPRegInfo` | `cpregs.h:956-970` | 系统寄存器元数据；支撑 VHE/NV2 重定向 |

---

## 13. 关键函数索引

| 函数 | 文件:行 | 作用 |
|------|---------|------|
| `arm_mmu_idx_el` | `helper.c:9957` | 当前 EL → `ARMMMUIdx` |
| `arm_hcr_el2_eff_secstate` | `helper.c:3818` | 计算指定安全空间下有效 `HCR_EL2` |
| `arm_is_el2_enabled_secstate` | `cpu.h:2228` | Secure/NS 状态下 EL2 是否可用 |
| `scr_write` | `helper.c:712` | `SCR_EL3` 写入与 `SCR_EEL2` 掩码控制 |
| `ptw_idx_for_stage_2` | `ptw.c:193` | Stage-2 页表遍历使用的物理空间 |
| `regime_ttbr` | `ptw.c:233` | 当前 regime 使用哪个 TTBR |
| `regime_tcr` | `internals.h:1063` | 当前 regime 使用哪个 TCR/VTCR/VSTCR |
| `S1_ptw_translate` | `ptw.c:645` | 加载页表描述符前先翻译 descriptor 地址 |
| `get_S2prot` | `ptw.c:1343` | Stage-2 S2AP/XN → QEMU 权限 |
| `get_S2prot_indirect` | `ptw.c:1384` | FEAT_S2PIE 间接权限 |
| `get_S1prot` | `ptw.c:1434` | Stage-1 AP/PXN/XN/PAN → QEMU 权限 |
| `get_phys_addr_lpae` | `ptw.c:1859` | 一阶段 LPAE 页表遍历 |
| `combine_cacheattrs` | `ptw.c:3413` | S1/S2 缓存属性合并 |
| `get_phys_addr_twostage` | `ptw.c:3554` | 串联执行 S1 + S2 |
| `get_phys_addr_nogpc` | `ptw.c:3661` | 主分发器 |
| `get_phys_addr_gpc` | `ptw.c:3800` | 在输出 PA 上叠加 GPC 检查 |
| `arm_mmu_idx_to_security_space` | `ptw.c:3858` | `mmu_idx` → 初始安全空间 |
| `arm_security_space_below_el3` | `helper.c:10163` | 根据 `SCR_EL3.NS/NSE` 推导 EL3 以下安全空间 |

---

## 14. 与其他文档的关联

- **arm64/09**：`09-虚拟化扩展深度分析-VHE-HCR_EL2-Stage2-MMU.md`  
  更适合深入看 `HCR_EL2` 位图、VHE、Stage-2 总体语义。
- **arm64/38**：`38-ARM64内存管理深度分析-页表遍历-TLB-Stage2翻译与属性合并.md`  
  更适合横向看 MMU/TLB/AT 指令及属性合并。
- **arm64/40**：`40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md`  
  更适合看 HVC、VHE 重定向、EL2 异常入口与 NV。
- **arm64/48**：`48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md`  
  更适合理解 `SCR_EL3`、`ARMSS_*` 四状态与 SEL2 使能背景。
- **arm64/49**：`49-ARM64-页表遍历PTW深度分析-Stage1-Stage2翻译-权限检查-Fault处理-安全属性传播.md`  
  更适合通读 PTW 子系统全貌。

本文可视为对 09/38/40/48/49 的一次“Hypervisor 视角重新编排”：把 EL2、SEL2、S1/S2、PTW、安全空间全部收束到同一条执行线上。

---

## 附录 A：关键源文件索引

| 文件 | 内容 | 关键行数 |
|------|------|----------|
| `mmuidx.h` | 翻译体制注释、22 个核心索引、NOTLB 索引 | `9-198` |
| `cpu.h` | `HCR_EL2`/`SCR_EL3` 位定义、EL2 使能谓词、物理空间映射 | `1695-1785`、`2228-2239`、`2376-2385` |
| `helper.c` | `scr_write()`、`arm_hcr_el2_eff_secstate()`、`arm_mmu_idx_el()`、安全空间判定 | `712-836`、`3818-3889`、`9957-10008`、`10131-10187` |
| `ptw.c` | PTW 上下文、Stage-2 PTW 空间、S1 descriptor load、权限、LPAE walk、twostage 分发 | `22-90`、`193-246`、`645-730`、`1343-1541`、`1859-2447`、`3413-3658`、`3661-3943` |
| `internals.h` | `regime_tcr()` 对 Secure Stage-2 的共享字段合成 | `1062-1082` |
| `arm-security.h` | 四种安全空间定义 | `18-35` |
| `cpregs.h` | NV2/VHE 重定向元数据说明 | `963-970` |
| `tlb_helper.c` | TLB miss 进入 PTW 的入口 | `331-379` |

---

## 附录 B：相关 Git Commit

以下提交与本文主题直接相关，可作为继续追溯设计意图的入口：

| Commit | 主题 | 启示 |
|--------|------|------|
| `150c24f34e` | `Update translation regime comment for new features` | `mmuidx.h` 顶部 regime 注释的演进背景 |
| `fcc0b0418f` | `Fix handling of SW and NSW bits for stage 2 walks` | `VSTCR.SW` / `VTCR.NSW` 是真实易错点 |
| `f04383e749` | `Honour VTCR_EL2 bits in Secure EL2` | Secure EL2 不是“只看 VSTCR_EL2” |
| `eeb9578c36` | `Account for FEAT_RME when applying {N}SW, SA bits` | 安全空间传播与 RME/SEL2 有耦合 |
| `4f51edd3cd` | `Set s1ns bit in fault info more consistently` | `fi->s1ns` 的故障语义非常关键 |
| `67f3eda009` | `Cache NV1 early in get_phys_addr_lpae` | NV1 会影响 walker 对 descriptor 的解释 |
| `f9f99d7ca5` | `Implement SEL2 physical and virtual timers` | FEAT_SEL2 不只是 MMU，也扩展 timer/异常模型 |
| `e886dccc82` | `Split out mmuidx.h from cpu.h` | `ARMMMUIdx` 模型从 `cpu.h` 中拆分后的组织结构 |

---

## 附录 C：推荐深入方向

1. **KVM 对 EL2/Stage-2 的利用**  
   进一步看 `KVM_ARM_VCPU_INIT`、KVM 下 VTTBR/VTCR 如何由宿主内核接管。

2. **SMMU/IOMMU 与 Stage-2 的交互**  
   CPU Stage-2 只是虚拟化的一半，I/O 地址翻译还要结合 SMMU 第二阶段。

3. **Nested Virtualization 完整实现**  
   继续深挖 `VNCR_EL2`、`nv2_redirect_offset`、`HCR_NV2` 对系统寄存器访问路径和 fault syndrome 的影响。

4. **RME Realm Management Extension 的 Stage-2**  
   继续分析 `ARMSS_Realm`、`Phys_Realm`、`GPT` 与 EL2/EL3/RME 的结合点。

5. **AT/ATS 指令对两阶段翻译的旁路观察**  
   结合 `get_phys_addr_for_at()`（`ptw.c:3835-3855`）与 `cpregs-at.c`，观察架构翻译指令如何复用 PTW。

---

## 总结

如果把本文压缩成一句话，那么 QEMU ARM64 Hypervisor MMU 的核心设计就是：

> **用 `ARMMMUIdx` 表达翻译体制，用 `S1Translate` 保存跨阶段上下文，用 `get_phys_addr_nogpc()` 做总分发，用 `get_phys_addr_twostage()` 把 S1 与 S2 串起来，并用安全空间与 `SW/NSW/SA/NSA/NSTable` 把 Secure EL2 的复杂语义压缩进同一套 walker。**

其中最值得记住的三个结论是：

1. `E2` / `E20_*` 是 **EL2 自己的单阶段 regime**；`E10_*` 才是 guest 两阶段翻译入口。  
2. FEAT_SEL2 下 `Stage2_S` 与 `Stage2` **不能合并**，因为 S-EL2 会同时托管 Secure 与 NonSecure guest。  
3. “S1 PTW through S2” 才是虚拟化 MMU 实现里真正的难点：不仅 guest 数据访问要过 S2，**guest 页表本身的 descriptor load 也要过 S2**。
