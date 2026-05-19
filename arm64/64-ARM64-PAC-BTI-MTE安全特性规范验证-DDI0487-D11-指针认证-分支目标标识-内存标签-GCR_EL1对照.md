# ARM64 PAC/BTI/MTE 安全特性规范验证

> QEMU 11.0.50 vs ARM DDI 0487 M.b — 指针认证 / 分支目标标识 / 内存标签扩展对照
>
> 说明：ARM DDI 0487 M.b 中，PAC 的核心规范已不再集中放在旧版 D11 单章，而是分散到 `D8.10 Pointer authentication`、相关系统寄存器章节 `D24`、以及具体指令章节 `C6`；BTI 主要落在 `FEAT_BTI`、`D8.4`、`J1.2.3 BTypeCompatible_*` 与 `C6`；MTE 主要落在 `D10 The Memory Tagging Extension` 与 `D24`。下文按用户要求继续沿用“D11 对照”命名，但引用以 M.b 实际章节号为准。

## 1. 验证概述

本次对照的 QEMU 路径主要包括：

- PAC：`target/arm/tcg/pauth_helper.c`、`target/arm/helper.c`、`target/arm/tcg/translate-a64.c`
- BTI：`target/arm/tcg/translate-a64.c`、`target/arm/tcg/helper-a64.c`、`target/arm/ptw.c`
- MTE：`target/arm/tcg/mte_helper.c`、`target/arm/tcg/hflags.c`、`target/arm/helper.c`、`target/arm/internals.h`
- 特性枚举/ID 寄存器：`target/arm/cpu-features.h`、`target/arm/tcg/cpu64.c`、`target/arm/helper.c`

### 1.1 总体结论

- **PAC 基础行为总体正确**：密钥寄存器模型、PAC 字段定位、legacy/FPAC/FPACCOMBINE 分流、XPAC 剥离、RETAA/ERETAA 等组合认证的时序，QEMU 基本与规范一致。
- **BTI 基础行为总体正确**：BTYPE 取值、BR/BLR 对 BTYPE 的设置、BTI/BTI c/BTI j/BTI jc 兼容矩阵、GP 仅来自 stage 1、ESR.EC=0x0d，QEMU 都做到了。
- **MTE 功能实现最完整，但“硬件时序/随机性”简化最明显**：16B granule、逻辑标签提取、标签存储、LDG/STG/ST2G/DC GZVA、TCF 模式、TCO/ATA 门控都具备；但 `RRND=1` 的随机标签生成、异步故障写回 `TFSR_EL1/TFSRE0_EL1` 的同步时机、与异步 Data Abort 并发时的 UNKNOWN 语义，QEMU 明显是可运行的功能模型，而非严格的时序模型。

### 1.2 差异统计

| 类别 | ✅ 正确实现 | ⚠️ 简化/近似 | ❌ 缺失/未实现 | 备注 |
|---|---:|---:|---:|---|
| PAC | 6 | 1 | 1 | 缺失点集中在 FEAT_PAuth_LR / PACM 这一代新指令 |
| BTI | 5 | 0 | 1 | 缺失点与 FEAT_PAuth_LR 的隐式 BTI 着陆指令一起出现 |
| MTE | 7 | 3 | 0 | 主要是随机性与异步故障可见性/时序建模 |

### 1.3 最影响 Guest 行为的差异

1. **❌ FEAT_PAuth_LR / PACM 未实现**：`PACIASPPC/AUTIASPPC/RETAASPPC/...` 这类 Armv9.4 新增增强 LR 签名/返回指令，在 QEMU 11.0.50 中没有落地；依赖这些新编码的 guest 二进制在 QEMU 上会表现为未定义指令或根本无法暴露该特性。
2. **⚠️ MTE `GCR_EL1.RRND=1` 仍使用确定性 LFSR 路径**：规范要求 `RRND=1` 时由实现定义生成、且分布不应劣于 `RRND=0`；QEMU 明确选择继续使用确定性算法，仅在种子为 0 时补一次随机种子。
3. **⚠️ MTE 异步故障时序被“立即化”**：规范 `§D10.7.1` 要求 `TFSR_ELx/TFSRE0_EL1` 的可见性受 `DSB` / `SCTLR_ELx.ITFSB` 约束；QEMU 在 tag mismatch 发生时立即 `OR` 目标 bit，省略硬件延迟/同步窗口。
4. **⚠️ MTE 对“异步 Data Abort 与 TagCheckFault 并发”的 UNKNOWN 语义未建模**：规范要求某些情况下状态位结果 UNKNOWN；QEMU 直接置位，得到确定性结果。

---

## 2. PAC 指针认证规范验证

## 2.1 密钥架构

### 规范要求

- PAC 相关指令在 M.b 中仍以 `APIAKey_EL1/APIBKey_EL1/APDAKey_EL1/APDBKey_EL1/APGAKey_EL1` 为架构命名。
- 在指令表中，`AUTIA/AUTIASP/RETAA/ERETAA` 等均明确引用 `APIAKey_EL1` 或 `APIBKey_EL1`，并**没有**对应的 `APIAKey_EL2/APIAKey_EL3` 一组独立密钥。可从 `Table C3-15` 与 `C6.2.158 ERETAA,ERETAB` 看到这一点（知识库命中：`arm_architecture_reference_manual.md:17157-17215`、`99452-99462`）。
- EL2/EL3 对 PAC 的控制主要体现在陷入/访问控制（例如 `HCR_EL2.API`、`SCR_EL3.API`），不是“换一套 EL2/EL3 独立密钥”。

### QEMU 实现

- QEMU 只建模了一套 5 组 128-bit 密钥，挂在 `CPUARMState.keys`，通过 `APDAKEY* / APDBKEY* / APGAKEY* / APIAKEY* / APIBKEY*` 寄存器访问：`target/arm/helper.c:5224-5275`。
- 认证/签名 helper 全部直接引用 `env->keys.apia/apib/apda/apdb/apga`：`target/arm/tcg/pauth_helper.c:490-538`、`540-631`。
- EL2/EL3 仅通过 `pauth_check_trap()` 里的 `HCR_API`、`SCR_API` 控制是否 trap：`target/arm/tcg/pauth_helper.c:464-483`。

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 的“一套 EL1 命名密钥 + EL2/EL3 trap 控制”与 M.b 架构模型一致；“EL2/EL3 另有独立 APxAKey_EL2/EL3”的理解并不符合当前 ARM ARM。
- **对 Guest 影响**：无。EL0/EL1/EL2/EL3 在 PAC 密钥共享上的行为与真实硬件一致；VHE 语义仍通过 trap/host 模式选择体现，而不是换密钥组。

## 2.2 PAC 字段大小计算

### 规范要求

`§D8.10.1 PAC field` 明确给出 PAC 域定位规则：

- `bottom_PAC_bit = 64 - TCR_ELx.TnSZ`
- 若地址标签参与，则是否使用 `Xd[55]` 作为范围选择点与 `TBI/TBID` 相关
- 当 address tagging 生效时，PAC 域应位于 `Xd[54:bottom_PAC_bit]` 或 `Xd[55:bottom_PAC_bit]`
- 当 address tagging 不对 instruction address 生效时，instruction PAC 可扩展利用高字节（见 `TBID` 影响）

知识库摘录（`arm_architecture_reference_manual.md:336380-336404`）与 `D8.8 address tagging` 摘录（`336230+`）都支持这一点。

### QEMU 实现

- PAC 域掩码：`pauth_ptr_mask()` 用 `bot_pac_bit = 64 - param.tsz`、`top_pac_bit = 64 - 8 * param.tbi`：`target/arm/internals.h:1785-1797`
- `pauth_addpac()` / `pauth_auth()` 同样以 `bot_bit = 64 - param.tsz`、`top_bit = 64 - 8 * param.tbi` 计算域宽：`target/arm/tcg/pauth_helper.c:327-386`、`408-447`
- `aa64_va_parameters()` 会把 `TBI` 与 `TBID` 复合后再写入 `param.tbi`：
  - `aa64_va_parameter_tbid()`：`target/arm/helper.c:9566-9575`
  - `aa64_va_parameters()`：`target/arm/helper.c:9814-9819`

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 不是简单看 `TBI`，而是先把 `TBID` 对 instruction/data 的影响折算进 `param.tbi`，再统一参与 PAC 域计算；这与 `§D8.10.1` 和 `§D8.8` 的规则一致。
- **对 Guest 影响**：无。不同 `TCR_ELx.TnSZ/TBI/TBID` 组合下，PAC 域宽度和真实硬件一致。

## 2.3 认证失败行为：legacy / FPAC / FPACCOMBINE

### 规范要求

`§D8.10.5 Faulting on pointer authentication` 规定：

- 对 `AUT*`：
  - 无 `FEAT_FPAC`：把结果地址变成 non-canonical
  - 有 `FEAT_FPAC`：直接抛出 `PACFail`，`EC=0b011100`
- 对 combined authenticate-and-branch / authenticate-and-load：
  - 无 `FEAT_FPACCOMBINE`：使用 non-canonical 地址继续后续访存/分支
  - 有 `FEAT_FPACCOMBINE`：直接抛 `PACFail`

见知识库摘录：`arm_architecture_reference_manual.md:336670-336708`。

### QEMU 实现

- PAuth feature level 枚举：`PauthFeat_1 / EPAC / 2 / FPAC / FPACCOMBINED`：`target/arm/cpu-features.h:922-929`
- legacy 路径：`pauth_auth()` 中 PAC 不匹配时，向 PAC 位写入 error code，而不是直接异常：`target/arm/tcg/pauth_helper.c:439-447`
- `FPAC` / `FPACCOMBINE` 路径：
  - `fault_feature = is_combined ? PauthFeat_FPACCOMBINED : PauthFeat_FPAC`
  - 若 feature 足够且 mismatch，则 `pauth_fail_exception()` 抛 `EXCP_UDEF`，但 syndrome 用 `EC_PACFAIL`：`target/arm/tcg/pauth_helper.c:427-435`
- syndrome 编码：`target/arm/syndrome.h:413-421`

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 明确区分 legacy / `FEAT_FPAC` / `FEAT_FPACCOMBINE` 三条路径，且 `PACFAIL` syndrome 正确。
- **备注**：QEMU 内部用 `EXCP_UDEF` 作为实现细节承载 PACFail，但对 guest 可见的 `ESR_ELx.EC=0x1c` 是正确的；这不构成架构差异。

## 2.4 PACIA/PACIB/PACDA/PACDB、PACGA 与 modifier/context

### 规范要求

- `PACIA/PACIB/PACDA/PACDB` 使用两寄存器输入：一个被签名地址，一个 modifier。
- `PACIASP/PACIBSP` 用 `SP` 作为 modifier；`PACIAZ/PACIBZ` 用零作为 modifier。
- `PACGA` 使用独立的 `APGAKey_EL1`，且不受 `SCTLR.EnIA/EnIB/EnDA/EnDB` 影响。

### QEMU 实现

- helper 入口分别把第二个源操作数直接传给 `pauth_addpac()` / `pauth_auth()`：`target/arm/tcg/pauth_helper.c:490-527`、`540-621`
- HINT 空间中：
  - `PACIA1716/AUTIA1716`：`target/arm/tcg/translate-a64.c:2099-2128`
  - `PACIASP/PACIBSP/AUTIASP/AUTIBSP`：`2167-2219`
  - `XPACLRI`：`2091-2096`
- `PACGA` 单独翻译，且不依赖 `pauth_active`：`target/arm/tcg/translate-a64.c:8638-8645`；helper 中也直接使用 `env->keys.apga`：`target/arm/tcg/pauth_helper.c:530-538`

### 结论

- **判定：✅ 正确实现**
- **原因**：modifier 的传递方式与规范一致；`PACGA` 不受 `EnIA/EnIB/EnDA/EnDB` 门控这一点也实现正确。

## 2.5 Combined PAC 指令：RETAA / ERETAA 等

### 规范要求

- `C6.2.158 ERETAA,ERETAB`：先认证 `ELR_ELx`，modifier 为 `SP`，成功后恢复 PSTATE 并跳转；**认证后的地址不写回 ELR**。
- `RETAA/RETAB` 等 combined branch 指令：若 `FPACCOMBINE` 可用，认证失败应直接 PACFail；否则走 non-canonical 分支路径。

### QEMU 实现

- `RETAA/RETAB`：`target/arm/tcg/translate-a64.c:1896-1912`
- `ERETAA/ERETAB`：`target/arm/tcg/translate-a64.c:1978-2004`
- common helper `auth_branch_target()` 会在分支前调用 `autia_combined/autib_combined`：`target/arm/tcg/translate-a64.c:1840-1856`
- `ERETAA/ERETAB` 路径里只把认证后的目标地址传给 `exception_return`，**没有写回 `elr_el[]`**，符合 `C6.2.158`：`translate-a64.c:1996-2004`

### 结论

- **判定：✅ 正确实现**
- **原因**：组合认证时机、失败路径、`ERETAA` 对 `ELR` 不回写，均符合规范。

## 2.6 EL2/EL3 控制与嵌套虚拟化细节

### 规范要求

PAC 在高 EL 的差异主要来自 trap 优先级和 `HCR_EL2.API` / `SCR_EL3.API` / NV 相关控制，而非独立密钥空间。

### QEMU 实现

- `pauth_check_trap()` 会先检查 `HCR_API` / host 模式，再检查 `SCR_API`：`target/arm/tcg/pauth_helper.c:464-483`
- 但源码中存在明确注释：
  - `/* FIXME: ARMv8.3-NV: HCR_NV trap takes precedence for ERETA[AB]. */`
  - 位置：`target/arm/tcg/pauth_helper.c:473`
- 与此同时，`trans_ERETA()` 也写明“FGT trap takes precedence over an auth trap”：`target/arm/tcg/translate-a64.c:1991-1994`

### 结论

- **判定：⚠️ 存在已知偏差**
- **偏差点**：**嵌套虚拟化场景下，ERETAA/ERETAB 的 HCR_NV trap 优先级尚未完全实现。**
- **严重度**：中
- **对 Guest 影响**：普通 EL0/EL1 Linux/应用几乎不受影响；但在启用 FEAT_NV/FEAT_NV2 的 hypervisor guest 中，ERETAA/ERETAB 的异常优先级可能与真实硬件不同。

## 2.7 XPAC / XPACD 剥离行为

### 规范要求

`C6.2.506 XPACD,XPACI,XPACLRI`：仅剥离 PAC，不做认证检查。

### QEMU 实现

- `XPACI/XPACD` 都只走 `pauth_strip()`，后者直接调用 `pauth_original_ptr()`，不比较签名：`target/arm/tcg/pauth_helper.c:450-455`、`624-631`
- HINT 形式 `XPACLRI` 也只是对 `LR` 调用 `gen_helper_xpaci`：`target/arm/tcg/translate-a64.c:2091-2096`

### 结论

- **判定：✅ 正确实现**
- **对 Guest 影响**：无。

## 2.8 FEAT_PAuth_LR / PACM：新规范功能缺失

### 规范要求

M.b 新增 `FEAT_PAuth_LR`：

- 特性介绍：`A2.2 FEAT_PAuth_LR`（知识库：`arm_architecture_reference_manual.md:7514-7525`）
- 新指令：`PACIASPPC/AUTIASPPC/RETAASPPC/...`（知识库：`17103-17112`、`110934-110955`、`35571-35573`）
- 新控制位：`SCTLR2_ELx.{EnPACM,EnPACM0}`（知识库：`420795-420827`）
- 还要求 `ID_AA64ISAR3_EL1.PACM` 暴露此能力

### QEMU 实现

- 只定义了 `SCTLR2_ENPACM` / `SCTLR2_ENPACM0` 常量：`target/arm/cpu.h:1491-1492`
- 但 `ID_AA64ISAR3_EL1` 在 QEMU 中仍是保留常量寄存器，值为 0：`target/arm/helper.c:6501-6505`
- `target/arm/tcg/translate-a64.c:2091-2219, 1840-2004, 8638-8645` 中只实现了旧一代 `PACIASP/AUTIASP/PACIA1716/BRAA/RETAA/ERETAA` 等路径
- 对 `PACIASPPC/AUTIASPPC/RETAASPPC/171615/PACM` 全仓库检索无实现匹配

### 结论

- **判定：❌ 缺失/未实现**
- **严重度**：高（仅对使用该新特性的 guest）
- **对 Guest 影响**：
  - 如果 guest/compiler/runtime 生成 `FEAT_PAuth_LR` 指令，QEMU 11.0.50 无法提供与 M.b 一致的执行语义。
  - 若 CPU model 不暴露该 feature，则表现为“功能缺失但自洽”；若未来模型误暴露 PACM 相关 ID 位，则会形成真实兼容性错误。

---

## 3. BTI 分支目标标识规范验证

## 3.1 BTYPE 状态机

### 规范要求

- `FEAT_BTI` 引入 `PSTATE.BTYPE`、page descriptor `GP` 位、`BTI` 指令（知识库：`arm_architecture_reference_manual.md:5133-5148`）
- `BTypeCompatible_BTI()` 伪代码给出兼容矩阵：
  - `BTI` (`hintcode 00`)：仅兼容 `BTYPE==00`
  - `BTI c` (`01`)：兼容除 `11` 以外
  - `BTI j` (`10`)：兼容除 `10` 以外
  - `BTI jc` (`11`)：全部兼容
  - 见知识库 `716643-716660`
- `BR` / `BLR` 对 `BTYPE` 的设置在架构上不同：`BR` 依赖寄存器/guarded page，`BLR` 固定 link-branch 类型。

### QEMU 实现

- `set_btype()` / `reset_btype()`：`target/arm/tcg/translate-a64.c:164-184`
- `BR`：
  - `x16/x17 -> BTYPE=1`
  - 否则通过 `guarded_page_br()` 在运行时决定 `1` 或 `3`
  - 见 `translate-a64.c:1777-1789`、`helper-a64.c:1768-1775`
- `BLR`：固定 `BTYPE=2`：`translate-a64.c:1792-1797`
- 目标指令兼容矩阵：`btype_destination_ok()`：`translate-a64.c:10617-10650`

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 对 `BR/BLR` 的 BTYPE 设置与 `BTypeCompatible_BTI` 的矩阵完全对上。

## 3.2 BTI 指令变体与检查时机

### 规范要求

- BTI 检查只在**guarded page**里触发；非 guarded page 上 `BTI` 本身等价于 NOP。
- 当 `PSTATE.BTYPE != 0` 且取指目标位于 guarded page，若目标指令不是兼容 BTI landing pad / 隐式 BTI 着陆点，则抛 `Branch Target exception`，`ESR_ELx.EC=0x0D`。
- 见知识库摘录：`arm_architecture_reference_manual.md:334470-334500`

### QEMU 实现

- 翻译首条指令时先静态检查目标指令编码：`translate-a64.c:10832-10849`
- 真正是否抛异常由运行时 helper 决定是否在 guarded page：`helper-a64.c:1754-1765`
- 非 guarded page 时，helper 什么也不做，因此不抛异常：同上
- syndrome：`target/arm/syndrome.h:439-446`

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 做的是“两阶段检查”：
  1. 先判断目标指令编码是否兼容；
  2. 再在运行时查询目标页是否 `GP/guarded`。
  这正好匹配 BTI 的架构语义。

## 3.3 SCTLR.BT0/BT1 与 PACIASP/PACIBSP 的隐式 BTI

### 规范要求

M.b 明确规定：

- `PACIASP/PACIBSP` 具有隐式 BTI 属性
- 当 `SCTLR_ELx.BT*` 关闭时，其兼容性等价于 `BTI jc`
- 当 `SCTLR_ELx.BT*` 打开时，其兼容性收紧为 `BTI c`
- 且**这一隐式 BTI 属性与 `SCTLR_ELx.{EnIA,EnIB}` 是否开启无关**

见知识库摘录：`arm_architecture_reference_manual.md:334500-334515`。

### QEMU 实现

- `btype_destination_ok()` 中：
  - `PACIASP/PACIBSP` 若 `bt==0`，则允许所有非零 BTYPE；若 `bt==1`，则拒绝 `btype==3`
  - 位置：`target/arm/tcg/translate-a64.c:10619-10628`
- `BT` 标志来自 `SCTLR_BT0/BT1`：`target/arm/tcg/hflags.c:332-336`
- `PACIASP/PACIBSP` 的**指令执行**是否真的做 PAC 计算，则另由 `pauth_active` / `EnIA/EnIB` 决定：`translate-a64.c:2167-2187`；这和 BTI landing pad 属性分离

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 把“隐式 BTI 属性”和“PAC 是否真正执行”拆开处理，符合规范要求。

## 3.4 Guard Page 属性与 Stage 2

### 规范要求

- BTI 的 `GP` 位来自 **stage 1** 页表描述符
- stage 2 不再提供第二份 GP 语义

### QEMU 实现

- stage 1 页表遍历时从属性 bit[50] 提取 `GP` 到 TLB：`target/arm/ptw.c:2342-2345`
- 合并 stage 2 时显式保留 stage 1 的 guarded 值，并写有注释：`/* No BTI GP information in stage 2, we just use the S1 value */`：`target/arm/ptw.c:3643-3644`

### 结论

- **判定：✅ 正确实现**
- **对 Guest 影响**：无。

## 3.5 异常入口/返回时 BTYPE

### 规范要求

- `SPSR_ELx.BTYPE` 在异常入口时保存当前 `PSTATE.BTYPE`，在异常返回时恢复
- 异常目标 EL 的当前 `PSTATE.BTYPE` 不继续沿用旧值

知识库中 `SPSR_EL1/EL2` 条目显示：`BTYPE` “Set to the value of PSTATE.BTYPE on taking an exception, and copied to PSTATE.BTYPE on exception return”（命中：`arm_architecture_reference_manual.md:46436-46439`, `47055-47058`）。

### QEMU 实现

- `pstate_write()` 会显式同步 `env->btype = (val >> 10) & 3`：`target/arm/cpu.h:1618-1625`
- AArch64 异常入口中：
  - 旧 PSTATE 先保存到 `banked_spsr[]`
  - 新 PSTATE 由 `PSTATE_DAIF | new_mode` 构造，不包含旧 BTYPE，因此目标 EL 的 `BTYPE` 被清为 0
  - 位置：`target/arm/helper.c:9343-9415`
- 异常返回 `exception_return()` 中：`pstate_write(env, spsr)` 会把 `SPSR_ELx.BTYPE` 恢复回 `PSTATE.BTYPE`：`target/arm/tcg/helper-a64.c:718-747`

### 结论

- **判定：✅ 正确实现**
- **对 Guest 影响**：无。跨 EL 异常入口不会错误继承旧的 BTYPE。

## 3.6 BTI 违例异常类型与 ESR 编码

### 规范要求

- `Branch Target exception` 的 `ESR_ELx.EC` 为 `0x0D`
- ISS 中带有 `BTYPE`

### QEMU 实现

- syndrome 构造：`target/arm/syndrome.h:439-446`
- 运行时抛异常：`target/arm/tcg/helper-a64.c:1762-1764`

### 结论

- **判定：✅ 正确实现**
- **备注**：QEMU 内部仍复用 `EXCP_UDEF` 这一路异常基础设施，但 guest 看到的 `ESR_ELx` 编码与 ARM ARM 相符。

## 3.7 与最新规范的差异：缺少 FEAT_PAuth_LR 对应的 BTI 着陆点

### 规范要求

M.b 把 `PACIASPPC/PACIBSPPC` 也纳入隐式 BTI 着陆点集合（见 `334484-334515`、`110941-110944`）。

### QEMU 实现

- `btype_destination_ok()` 只识别 `PACIASP/PACIBSP`，不识别 `PACIASPPC/PACIBSPPC`：`target/arm/tcg/translate-a64.c:10621-10628`
- 根因不是 BTI 核心逻辑错误，而是 **QEMU 根本没有实现 `FEAT_PAuth_LR` 新指令**（见前文 `2.8`）

### 结论

- **判定：❌ 缺失/未实现（随 FEAT_PAuth_LR 一并缺失）**
- **严重度**：高（仅对 Armv9.4+ 使用该特性的 guest）
- **对 Guest 影响**：新一代 PAC prologue/epilogue 若依赖 `PACIASPPC` 同时承担隐式 BTI landing pad，QEMU 11.0.50 不能与真实硬件一致运行。

---

## 4. MTE 内存标签规范验证

## 4.1 标签粒度与标签存储架构

### 规范要求

- allocation tag 粒度固定为 **16 bytes / Tag Granule**
- 逻辑标签来自地址 bits `[59:56]`
- `LDG/STG/ST2G/STZG/STZ2G` 等都围绕这个 16B 粒度工作

### QEMU 实现

- 地址逻辑标签提取：`allocation_tag_from_addr()` 直接取 `[59:56]`：`target/arm/internals.h:1620-1628`
- tag granule 相关索引/比较全部基于 `TAG_GRANULE`：
  - `allocation_tag_mem_probe()`：`target/arm/tcg/mte_helper.c:61-197`
  - `mte_probe_int()` 中按 `QEMU_ALIGN_DOWN(ptr, TAG_GRANULE)` 划分粒度：`780-866`
- user-mode：tag 存在 page 附加数据区；system-mode：tag 走独立 tag address space，对应 RAMBlock 标签内存：`mte_helper.c:66-92`、`164-196`

### 结论

- **判定：✅ 正确实现**
- **对 Guest 影响**：无。QEMU 对 16B granule 的遵守是严格的，不是近似实现。

## 4.2 GCR_EL1.Exclude 对 ADDG / IRG 的影响

### 规范要求

- `§D24.2.52 GCR_EL1`：`Exclude[15:0]` 表示禁止被选中的 allocation tag；若全 1，则返回 tag 0
- `ADDG/SUBG` 使用 `ChooseNonExcludedTag`
- `IRG` 既要考虑寄存器给出的 exclude，也要 OR 上 `GCR_EL1.Exclude`
- 对应指令描述见：
  - `ADDG`：知识库 `82326-82361`
  - `IRG`：知识库 `100294-100327`

### QEMU 实现

- `choose_nonexcluded_tag()`：若 `exclude == 0xffff` 返回 0，否则按 offset 在允许标签集合中前进：`target/arm/tcg/mte_helper.c:42-59`
- `IRG`：`exclude = rm[15:0] | GCR_EL1.Exclude`：`mte_helper.c:209-255`
- `ADDG/SUBG`：仅看 `GCR_EL1.Exclude`：`mte_helper.c:257-264`

### 结论

- **判定：✅ 正确实现**
- **原因**：指令级 exclude 合并规则、全禁用时回落到 tag 0 的行为，与规范一致。

## 4.3 RGSR_EL1 与随机标签生成

### 规范要求

- `§D24.2.156 RGSR_EL1`：保存 `TAG` 与 `SEED`
- `§D24.2.52 GCR_EL1.RRND`：
  - `RRND=0`：按 `RandomTag() + ChooseNonExcludedTag()` 路径
  - `RRND=1`：由实现定义生成 tag，分布不得劣于 `RRND=0`，Arm 建议尽量接近均匀分布

### QEMU 实现

- `IRG` helper：
  - 从 `RGSR_EL1.TAG/SEED` 取初值：`target/arm/tcg/mte_helper.c:211-215`
  - 若 `RRND=1` 且种子为 0，则临时从 `qemu_guest_getrandom()` 取一个非零 16-bit seed：`224-241`
  - 之后仍然执行 4 次 LFSR 演进，得到 offset，再走 `choose_nonexcluded_tag()`：`243-253`
- 源码里有明确注释：
  - `Our IMPDEF choice for GCR_EL1.RRND==1 is to continue to use the deterministic algorithm.`
  - 位置：`target/arm/tcg/mte_helper.c:217-223`

### 结论

- **判定：⚠️ 简化/近似**
- **偏差点**：`RRND=1` 时，QEMU 没有模拟“真正实现定义的随机标签生成器”，而是继续沿用确定性 LFSR，只在 seed 为 0 时补一次随机种子。
- **严重度**：中
- **对 Guest 影响**：
  - 对一般功能正确性影响很小；`IRG/ADDG` 仍可避开 exclude 标签。
  - 对依赖标签分布质量、重现真实硬件随机性、或做 MTE 硬件统计/鲁棒性验证的软件，会出现与真实硬件不一致的概率分布。

## 4.4 LDG / STG / STZG / ST2G / STZ2G / DC GZVA

### 规范要求

- `LDG`：从 allocation tag storage 读取 tag，并写回目标寄存器的 logical tag
- `STG/ST2G`：写 allocation tag，不清数据
- `STZG/STZ2G`：写 allocation tag 并清零数据
- `DC GZVA`：按 ZVA block 清数据并清标签

### QEMU 实现

- `LDG`：`target/arm/tcg/mte_helper.c:273-289`
- `STG/ST2G`：`325-409`
- `STZGM tags` / `mte_check_zva()` / `translate-a64.c` 中 `DC_GZVA` 路径：
  - helper：`mte_helper.c:536-560`, `922-1023`
  - 翻译：`target/arm/tcg/translate-a64.c:3006-3046`
- 当 `ATA` 关闭时，QEMU 仍保留对普通数据访存/对齐异常的探测，但标签读返回 0、标签写退化为 stub：
  - `LDG`：`translate-a64.c:4723-4753`
  - `STG/ST2G`：`4766-4836`
  - `ADDG/SUBG`：`5058-5068`

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 对 tag-only 与 tag+data 清零两类指令都做了区分；ATA 关闭时也正确保留普通地址访问 fault 语义。

## 4.5 Tag Check：何时需要检查

### 规范要求

Tag check 只发生在 **Checked access**：

- 需要有可用逻辑标签（TBI 打开）
- 不能被 `PSTATE.TCO` 全局关闭
- 不能被 `TCMA` 豁免
- 需要 `SCTLR_ELx.{ATA/ATA0}` 允许 allocation tag access
- 需要 `SCTLR_ELx.{TCF/TCF0}` 不是 No effect

### QEMU 实现

- `allocation_tag_access_enabled()`：检查 `SCR_ATA` / `HCR_ATA` / `SCTLR_ATA{,0}`：`target/arm/internals.h:1430-1446`
- TB flag 生成时，只有在以下条件全满足才置 `MTE_ACTIVE`：
  1. `ATA` 可用
  2. `TBI` 打开
  3. `PSTATE.TCO==0`
  4. `TCF != 0`
  - 见 `target/arm/tcg/hflags.c:402-450`
- 运行时检查：
  - `tbi_check()`：`internals.h:1630-1634`
  - `tcma_check()`：`1636-1646`
  - `mte_probe_int()`：`mte_helper.c:792-801`

### 结论

- **判定：✅ 正确实现**
- **对 Guest 影响**：无。QEMU 没有错误地把所有访存都做 tag check，而是严格按照 Checked/Unchecked access 规则门控。

## 4.6 同步 / 异步 / 不对称故障模式

### 规范要求

- `SCTLR_EL1.TCF0` / `SCTLR_EL1.TCF` / `SCTLR_EL2.TCF` 控制 TagCheckFault 行为
- QEMU 自身在文档与测试中也按以下语义组织：
  - 0：No effect
  - 1：Sync
  - 2：Async
  - 3：Asymmetric（读同步/写异步，`FEAT_MTE3/FEAT_MTE_ASYM_FAULT`）
- 对应测试也能侧面印证：`tests/tcg/aarch64/mte-2.c`、`mte-3.c`、`mte-6.c` 等分别覆盖 sync/async 模式

### QEMU 实现

- `mte_check_fail()`：
  - `TCF=1 -> mte_sync_check_fail()`：`target/arm/tcg/mte_helper.c:623-627`
  - `TCF=2 -> mte_async_check_fail()`：`637-640`
  - `TCF=3 -> 写异步，读同步`：`642-651`
- QEMU `max` CPU 直接把 `ID_AA64PFR1_EL1.MTE` 设为 3（`FEAT_MTE3`）：`target/arm/tcg/cpu64.c:1280-1289`
- 文档也声明支持 `FEAT_MTE3` / `FEAT_MTE_ASYM_FAULT`：`docs/system/arm/emulation.rst:109-113`

### 结论

- **判定：✅ 正确实现**
- **原因**：就“功能语义”而言，QEMU 已把 sync/async/asymmetric 三种 guest 可见模式区分开。

## 4.7 TFSR_EL1 / TFSRE0_EL1 的累积行为与时序

### 规范要求

`§D10.7.1 Asynchronous TagCheckFaults` 要求：

- 异步 TagCheckFault 会累计到 `TFSR_ELx` 或 `TFSRE0_EL1`
- 该寄存器的隐式写入何时对软件可见，要受以下同步事件约束：
  - `DSB LD` 或 `DSB`（既非 LD 也非 ST）
  - 或异常入口且 `SCTLR_ELx.ITFSB == 1`
- 如果在状态还未被这些机制同步前，就修改了影响 tag-fault 处理的系统寄存器，其效果是 **CONSTRAINED UNPREDICTABLE**

见知识库摘录：`arm_architecture_reference_manual.md:341151-341199`。

### QEMU 实现

- 一旦异步 mismatch 发生，QEMU 立即：
  - 选 bit55 对应的状态位
  - `env->cp15.tfsr_el[el] |= 1 << select`
  - user-mode 下还直接 `cpu_exit()` 提前回主循环
  - 位置：`target/arm/tcg/mte_helper.c:577-597`
- 全仓库检索没有看到针对 `ITFSB` 或 `DSB` 去延迟/提交这些 tag-fault 状态位的专门逻辑；`SCTLR_ITFSB` 仅作为 bit 定义/写掩码出现：`target/arm/cpu.h:1464`、`target/arm/helper.c:3277-3282`

### 结论

- **判定：⚠️ 简化/近似**
- **偏差点**：QEMU 把异步 TagCheckFault 的 `TFSR` 更新做成了“故障点立即可见”，没有建模 ARM 规定的 `DSB/ITFSB` 同步窗口。
- **严重度**：中到高（对依赖精确时序的软件）
- **对 Guest 影响**：
  - 常规 Linux MTE 使用通常不受影响，因为内核/运行时只关心“最终有无 fault”。
  - 若 guest 用 `TFSR_EL1/TFSRE0_EL1` 做精确时序观察、屏障验证、或验证 `ITFSB` 行为，QEMU 会比真实硬件“更早看到”状态位。

## 4.8 与异步 Data Abort 并发时的 UNKNOWN 语义

### 规范要求

`§D10.7.1` 还要求：若同一访存同时产生**异步 Data Abort** 和 **异步 TagCheckFault**，且目标状态位原先为 0，则该位结果可能是 **UNKNOWN**。

### QEMU 实现

- `mte_async_check_fail()` 无条件执行 `env->cp15.tfsr_el[el] |= 1 << select`：`target/arm/tcg/mte_helper.c:577-588`
- 代码中没有为“与异步 Data Abort 并发”的 UNKNOWN 结果建模，也没有延迟合并逻辑

### 结论

- **判定：⚠️ 简化/近似**
- **偏差点**：规范要求此场景可能为 UNKNOWN；QEMU 给出确定性的“置 1”。
- **严重度**：低到中
- **对 Guest 影响**：一般 OS/应用几乎无感；但做架构一致性/异常竞争测试时会与硬件不同。

## 4.9 Device 内存、未标记页、Stage 2 与 tag check

### 规范要求

- Device memory 不是 tagged normal memory，不应走 allocation tag compare
- 若页不支持 tags，则访问应作为 unchecked access 处理
- Stage 2 参与普通数据访问翻译；BTI 的 GP 不来自 stage 2；MTE tag 存储本质上从最终物理位置导出

### QEMU 实现

- system-mode 中，若 `pte_attrs != 0xf0`，直接返回 `NULL`，表示 unchecked：`target/arm/tcg/mte_helper.c:120-123`
- 若页被标成 tagged normal memory 但实际对应 MMIO，则记 guest error 并不做 tag storage：`126-134`
- 用数据访问解析得到的最终物理地址 `ptr_paddr` 推导 `tag_paddr`：`140-170`
- stage 2 合并 cacheattrs 时保留 tagged attribute，不单独再做第二套 tag 权限判定：`target/arm/ptw.c:3625-3641`

### 结论

- **判定：✅ 正确实现**
- **原因**：QEMU 在“只对 tagged normal memory 做 tag compare”这一点上是严格的；Device / 非 tagged 页都会退化为 unchecked access。

## 4.10 FEAT_MTE3 的 TCO 行为

### 规范要求

- `TCO` 寄存器/`PSTATE.TCO` 允许全局关闭 load/store 的 tag check（`TCO=1 -> unchecked`）
- `TCO` 在异常入口时保存/恢复；`MSR TCO` 仅在 full MTE 时真正可写，否则可 RAZ/WI
- 见 `TCO` 专门寄存器说明（知识库 `48754+`）以及 `SPSR_ELx.TCO` 条目（`46371-46374`, `46990-46993`, `47536-47539`）

### QEMU 实现

- `TCO` 系统寄存器：`target/arm/helper.c:5433-5441, 5472-5475`
- `MSR TCO`：
  - full MTE 时真的修改 `PSTATE_TCO`
  - 只有 instruction/register subset 时，按 WI 处理
  - 见 `target/arm/tcg/translate-a64.c:2431-2448`
- 异常入口默认置 `PSTATE_TCO`：`target/arm/helper.c:9395-9397`
- 异常返回通过 `pstate_write(env, spsr)` 恢复：`target/arm/tcg/helper-a64.c:721-747`

### 结论

- **判定：✅ 正确实现**
- **对 Guest 影响**：无。

---

## 5. 综合差异汇总表

| ID | 类别 | 结论 | 规范依据 | QEMU 位置 | 差异描述 | Guest 影响 |
|---|---|---|---|---|---|---|
| PAC-1 | 密钥架构 | ✅ | `C3-15`, `C6.2.158` | `helper.c:5224-5275`, `pauth_helper.c:490-538` | 仅建模一套 EL1 命名 PAC keys，符合架构，不存在应实现而未实现的 EL2/EL3 独立 key | 无 |
| PAC-2 | PAC 域宽度 | ✅ | `§D8.10.1`, `§D8.8` | `internals.h:1791-1797`, `helper.c:9566-9575, 9814-9819`, `pauth_helper.c:327-447` | `TCR.TSZ + TBI/TBID` 折算正确 | 无 |
| PAC-3 | FPAC/FPACCOMBINE | ✅ | `§D8.10.5` | `cpu-features.h:922-929`, `pauth_helper.c:427-447`, `syndrome.h:413-421` | legacy / FPAC / FPACCOMBINE 分流正确 | 无 |
| PAC-4 | ERETAA/RETAA | ✅ | `C6.2.158` | `translate-a64.c:1840-2004` | 认证后分支、`ERETAA` 不回写 ELR，符合规范 | 无 |
| PAC-5 | NV 优先级 | ⚠️ | NV + PAC trap precedence | `pauth_helper.c:473` | `ERETA[AB]` 的 `HCR_NV` trap precedence 仍是 FIXME | 嵌套虚拟化场景下异常优先级可能不同 |
| PAC-6 | FEAT_PAuth_LR/PACM | ❌ | `A2.2 FEAT_PAuth_LR`, `C6.2.307` | `cpu.h:1491-1492`, `helper.c:6501-6505`, `translate-a64.c:2091-2219,1840-2004` | `PACIASPPC/AUTIASPPC/RETAASPPC/...` 与 `PACM` 未实现 | 使用 Armv9.4 新 PAC 指令的 guest 不兼容 |
| BTI-1 | BTYPE 状态机 | ✅ | `BTypeCompatible_BTI`, `FEAT_BTI` | `translate-a64.c:1777-1797, 10617-10650`, `helper-a64.c:1768-1775` | `BR/BLR` 与 BTI 兼容矩阵正确 | 无 |
| BTI-2 | GP 仅 stage1 | ✅ | FEAT_BTI / stage1 GP | `ptw.c:2342-2345, 3643-3644` | 只使用 S1 的 GP，符合规范 | 无 |
| BTI-3 | 异常入口/返回 | ✅ | `SPSR_ELx.BTYPE` | `helper.c:9343-9415`, `helper-a64.c:721-747`, `cpu.h:1618-1625` | 异常入口清零当前 BTYPE，异常返回恢复 | 无 |
| BTI-4 | ESR 编码 | ✅ | BTI exception, `EC=0x0D` | `syndrome.h:439-446`, `helper-a64.c:1762-1764` | guest 可见 syndrome 正确 | 无 |
| BTI-5 | PAuth_LR landing pad | ❌ | `334484-334515`, `110941-110944` | `translate-a64.c:10621-10628` | 缺少 `PACIASPPC/PACIBSPPC` 隐式 BTI 着陆点支持（根因是 `FEAT_PAuth_LR` 未实现） | 新版编译器生成的 landing pad 不兼容 |
| MTE-1 | 16B granule / tag store | ✅ | `D10` | `internals.h:1620-1628`, `mte_helper.c:61-197, 780-866` | granule、逻辑标签、tag storage 模型正确 | 无 |
| MTE-2 | GCR_EL1.Exclude | ✅ | `§D24.2.52`, `ADDG`, `IRG` | `mte_helper.c:42-59, 209-264` | Exclude 语义正确 | 无 |
| MTE-3 | RRND=1 随机性 | ⚠️ | `§D24.2.52`, `§D24.2.156` | `mte_helper.c:217-253` | `RRND=1` 仍走确定性 LFSR，仅补随机 seed | 分布与真实硬件可能不同 |
| MTE-4 | sync/async/asym 模式 | ✅ | `D10` / `SCTLR.TCF*` | `mte_helper.c:600-654`, `cpu64.c:1280-1289` | 0/1/2/3 四种模式区分正确 | 无 |
| MTE-5 | TFSR/TFSRE0 时序 | ⚠️ | `§D10.7.1` | `mte_helper.c:577-597`, `helper.c:3277-3282` | QEMU 立即置位状态位，未建模 `DSB/ITFSB` 同步窗口 | 精确时序测试与硬件不同 |
| MTE-6 | UNKNOWN 竞争语义 | ⚠️ | `§D10.7.1` | `mte_helper.c:577-588` | 异步 Data Abort + TagCheckFault 并发时，QEMU 不建模 UNKNOWN，直接置 1 | 异常竞争测试结果更“确定” |
| MTE-7 | Device/untagged 页面 | ✅ | `D10` | `mte_helper.c:120-134`, `ptw.c:3625-3641` | 仅 tagged normal memory 做检查，Device/非 tagged 页退化 unchecked | 无 |
| MTE-8 | TCO/ATA 门控 | ✅ | `TCO`, `SPSR_ELx.TCO` | `helper.c:5433-5475, 9395-9397`, `translate-a64.c:2431-2448`, `hflags.c:402-450` | `TCO/ATA` 行为正确 | 无 |

---

## 6. 对 Guest 软件的影响

## 6.1 对 Linux 内核的影响

### PAC

- Linux/ELF ABI 当前广泛使用的 `PACIASP/AUTIASP/RETAA/ERETAA` 路径，QEMU 基本能正确运行。
- 若 guest hypervisor 依赖 `FEAT_NV + ERETAA/ERETAB` 的精确 trap precedence，QEMU 可能与硬件不同。
- 若未来 Linux/编译器开始启用 `FEAT_PAuth_LR`（如 `PACIASPPC/RETAASPPC`），QEMU 11.0.50 不能作为严格对等硬件。

### BTI

- 现有使用 `BTI c/j/jc` 与 `PACIASP/PACIBSP` 作为 prologue landing pad 的用户态/内核态代码，QEMU 行为与硬件大体一致。
- 但若 toolchain 切换到 `PACIASPPC/PACIBSPPC` 这类新的隐式 BTI landing pad，QEMU 11.0.50 不可用。

### MTE

- Linux 常见的 sync/async/asymmetric 用户态接口在 QEMU 上能跑通，说明“功能模型”是完整的。
- 但内核若去验证：
  - `RRND=1` 下标签分布质量
  - `DSB/ITFSB` 后 `TFSR_EL1/TFSRE0_EL1` 的可见时机
  - 异步 Data Abort 与 async tag fault 并发时的状态位竞争
  则 QEMU 会比真实硬件更简单、更确定。

## 6.2 对编译器/运行时的影响

- **Clang/GCC 的传统 PAC/BTI 序言尾声**：基本可依赖 QEMU。
- **Armv9.4+ 新 PACM / PAuth_LR 代码生成**：当前不应把 QEMU 11.0.50 当成可运行目标。
- **MTE fuzz / sanitizer / allocator 评估**：
  - 功能性验证没问题；
  - 随机性、时序性、异常竞争性验证不宜把 QEMU 结果当成硬件等价结论。

## 6.3 建议的使用边界

### 可把 QEMU 视为“接近硬件”的场景

- PAC 基础指令功能验证
- BTI 着陆点与 BTYPE 流程验证
- MTE 基础 tag check、tag storage、fault mode（sync/async/asym）功能验证

### 不应把 QEMU 视为“硬件等价”的场景

- `FEAT_PAuth_LR / PACM` 新指令与控制位验证
- `FEAT_NV + ERETAA/ERETAB` trap precedence 精确验证
- `RRND=1` 的真实随机性/分布测试
- `TFSR_EL1/TFSRE0_EL1` 的屏障同步时序验证
- 异步 fault 竞争、UNKNOWN 语义验证

---

## 7. 最终结论

QEMU 11.0.50 对 ARM64 **PAC/BTI/MTE** 的实现，整体上属于：

- **PAC/BTI：架构功能高度对齐，只有最新扩展没跟上**
- **MTE：功能完整，但故障时序与随机性是“工程化简化模型”**

如果目标是：

- **运行真实 guest OS / 用户态程序**：QEMU 的 PAC/BTI/MTE 支持已经足够强；
- **做 ARM ARM M.b 级别的严格一致性验证**：需要特别标记本文列出的 5 个偏差点，尤其是 **`FEAT_PAuth_LR` 缺失** 与 **MTE 异步故障时序/随机性简化**。

## 8. 附：本次核对用到的关键源码位置

- `target/arm/tcg/pauth_helper.c:327-447` — PAC 插入、认证、legacy/FPAC 分流
- `target/arm/tcg/pauth_helper.c:464-483` — PAC trap 控制（含 NV FIXME）
- `target/arm/helper.c:5224-5275` — AP*KEY* 系统寄存器
- `target/arm/helper.c:9566-9575, 9814-9819` — `TBID` 与 `TBI` 折算
- `target/arm/tcg/translate-a64.c:1777-1797` — `BR/BLR` 的 `BTYPE` 设置
- `target/arm/tcg/translate-a64.c:10617-10650` — BTI 目标兼容矩阵
- `target/arm/tcg/helper-a64.c:1754-1775` — guarded page runtime 检查
- `target/arm/ptw.c:2342-2345, 3643-3644` — GP 仅来自 stage 1
- `target/arm/tcg/mte_helper.c:42-59` — `Exclude` 标签选择
- `target/arm/tcg/mte_helper.c:209-264` — `IRG/ADDG/SUBG`
- `target/arm/tcg/mte_helper.c:273-409` — `LDG/STG/ST2G`
- `target/arm/tcg/mte_helper.c:563-654` — sync/async/asym fault handling
- `target/arm/tcg/mte_helper.c:577-597` — `TFSR` 立即置位简化
- `target/arm/tcg/hflags.c:402-450` — `MTE_ACTIVE/ATA/TCO/TCMA` 门控
- `target/arm/tcg/cpu64.c:1280-1289` — `max` CPU 的 `MTE=3` 暴露
- `target/arm/helper.c:6501-6505` — `ID_AA64ISAR3_EL1` 仍为保留寄存器（PACM 未暴露）

---

> 备注：本次任务仅产出分析文档，未修改 QEMU 代码路径。自动化测试层面仅核对了现有相关测试源分布（`tests/tcg/aarch64/{pauth-*,bti-*,mte-*}` 与 `tests/qtest/arm-cpu-features.c`），未执行完整回归。