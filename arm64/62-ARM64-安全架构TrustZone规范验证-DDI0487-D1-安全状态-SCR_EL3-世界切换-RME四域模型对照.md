# 62-ARM64 安全架构 TrustZone 规范验证：DDI 0487 D1 安全状态、SCR_EL3、世界切换与 RME 四域模型对照

> 基线：QEMU `target/arm` 与 `hw/intc` 当前源码  
> 对照规范：Arm Architecture Reference Manual for A-profile architecture, DDI 0487 M.b  
> 重点章节：`§D1.1.2`、`§D1.4.2`、`§D1.4.4`、`§D1.4.5.3`、`§D1.4.6`、`§D1.4.9`  
> 交叉核对文档：`08`、`23`、`37`、`39`、`48`

---

## 1. 概述

本文不是重新讲一遍 TrustZone，而是做一次“**规范 → QEMU 实现 → 既有分析文档**”三方交叉验证，目标集中在 4 个问题：

1. **安全状态到底如何定义**：Secure / Non-secure / Realm / Root 的成立条件是什么。
2. **SCR_EL3 到底控制什么**：哪些位决定低 EL 的安全状态、异常路由、执行状态、Secure EL2 可用性。
3. **世界切换到底发生在什么时刻**：SMC 到 EL3、EL3 中修改 SCR_EL3、ERET 返回后哪些状态真正改变。
4. **QEMU 是否真的实现了 RME 四域模型**：还是只保留了经典 Secure / Non-secure 二分。

本文只聚焦 A-profile AArch64 语义。AArch32 Monitor 与 M-profile 只在必要处作为补充，不展开。

### 1.1 结论先行

先给出结论，后文逐条展开：

- **规范层面**：`DDI 0487 §D1.1.2` 明确规定，PE 的 Security state 在无 RME 时是 Secure / Non-secure；在 `FEAT_RME` 下扩展为 **Secure / Non-secure / Realm / Root** 四态。`EL3` 在无 RME 时总在 **Secure**，有 RME 时总在 **Root**。`EL2` 及以下**不能处于 Root**（`DDI 0487 §D1.1.2`）。
- **SCR_EL3 层面**：低 EL 的安全状态由 `SCR_EL3.{NS,NSE}` 的 **Effective value** 选择；Secure EL2 是否存在还取决于 `FEAT_SEL2` 与 `SCR_EL3.EEL2`（`DDI 0487 §D1.1.2`）。
- **异常层面**：SMC 属于“exception-generating instruction”范畴，本质是**同步异常路径**；在 EL3 存在、且未被 `HCR_EL2.TSC` 抢先陷入 EL2、也未因 `SCR_EL3.SMD` 变成 UNDEF 时，SMC 才会被送到 EL3（`DDI 0487 §D1.4.9`）。
- **QEMU 层面**：QEMU 已经不是简单的二态模型。`ARMSecuritySpace` 明确建模了 `Secure / NonSecure / Root / Realm` 四态；但板级 `AddressSpace` 仍主要只注册 `NS/S` 两套地址空间，而 `Root/Realm` 更多通过 `MemTxAttrs.space`、物理 MMU index、GPT 检查来表达。
- **勘误层面**：现有 5 篇文档总体方向正确，但至少有两类需要统一修正：  
  1) **`SCR_EEL2` 位号应为 bit 18，不是 bit 21**；  
  2) **Root 不是由低 EL 的 `NSE:NS=1:0` 选出的状态**，而是 **EL3 在 FEAT_RME 下的固有状态**。`NSE=1, NS=0` 在 QEMU 里也明确按保留组合处理，不映射为 Root。

### 1.2 本文使用的关键代码入口

- `include/hw/arm/arm-security.h:18-35`：`ARMSecuritySpace` 与 `arm_space_is_secure()`。
- `target/arm/cpu.h:1756-1798`：`SCR_EL3` 位定义。
- `target/arm/helper.c:712-836`：`scr_write()`，含 `NS/NSE` 变化后的 TLB 刷新。
- `target/arm/helper.c:3818-3889`：`arm_hcr_el2_eff_secstate()`。
- `target/arm/helper.c:8309-8421`：`target_el_table` 与 `arm_phys_excp_target_el()`。
- `target/arm/helper.c:10131-10187`：`arm_security_space()` / `arm_security_space_below_el3()`。
- `target/arm/tcg/op_helper.c:1111-1200`：`pre_smc()`。
- `target/arm/tcg/translate-a64.c:3193-3204`：`trans_SMC()`。
- `target/arm/ptw.c:333-566`、`3811-3829`：GPT / GPC 检查。
- `hw/intc/arm_gicv3_cpuif.c:40-48`、`1060-1083`：GIC 安全 bank 选择与 Group→IRQ/FIQ 信号映射。

---

## 2. 安全状态验证（`DDI 0487 §D1.1.2`）

### 2.1 规范原义

`DDI 0487 §D1.1.2` 明确给出以下规则：

- 架构定义的 Security state 为：
  - **Secure state**
  - **Non-secure state**
  - 若实现 `FEAT_RME`，再增加：
    - **Realm state**
    - **Root state**
- `EL3` 总是处于：
  - **无 FEAT_RME：Secure**
  - **有 FEAT_RME：Root**
- `SCR_EL3.{NSE,NS}` 的 Effective value 选择 **EL2 及以下**的 Security state。
- `EL2 and lower cannot be in Root state`。
- `EL2` 只有在下列条件同时满足时，才能位于 Secure state：
  - 低 EL 被 `SCR_EL3.{NSE,NS}` 选到 Secure
  - 实现 `FEAT_SEL2`
  - `SCR_EL3.EEL2 == 1`

这意味着：**Root 不是“低 EL 的一种普通取值”**，而是 **EL3 的专属状态**。这是本文核对现有文档时最重要、也最容易被误读的一点。

### 2.2 规范状态表

| 状态 | 成立条件 | 可出现的 EL | 备注 |
|---|---|---|---|
| Secure | 无 RME 时 EL3；或低 EL 被 `SCR_EL3.{NSE,NS}` 选到 Secure | EL3 / EL2 / EL1 / EL0 | 低 EL Secure-EL2 还要额外满足 `EEL2=1` |
| Non-secure | 低 EL 被 `SCR_EL3.{NSE,NS}` 选到 Non-secure | EL2 / EL1 / EL0 | 经典 NS 世界 |
| Realm | `FEAT_RME` + 低 EL 被 `SCR_EL3.{NSE,NS}` 选到 Realm | EL2 / EL1 / EL0 | EL3 不在 Realm |
| Root | `FEAT_RME` 下 EL3 固有状态 | **仅 EL3** | `EL2 and lower cannot be in Root state` |

### 2.3 QEMU 的状态判定是否符合规范

#### `ARMSecuritySpace` 枚举

`include/hw/arm/arm-security.h:18-23`：

```c
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,
    ARMSS_NonSecure  = 1,
    ARMSS_Root       = 2,
    ARMSS_Realm      = 3,
} ARMSecuritySpace;
```

这与规范四态模型一致。需要注意的是，注释写明：**枚举顺序只是在数值上对齐部分编码语义**，并不等于“任何状态都能用 `SCR_EL3.NSE:NS` 直接拼出来”。尤其 `Root` 是例外。

#### `arm_security_space()`：判定“当前”状态

`target/arm/helper.c:10131-10160` 的逻辑非常贴近 `§D1.1.2`：

- 无 `EL3`：QEMU 采取默认 `ARMSS_NonSecure`
- 当前在 AArch64 `EL3`：
  - 有 `RME` → `ARMSS_Root`
  - 无 `RME` → `ARMSS_Secure`
- AArch32 Monitor → `ARMSS_Secure`
- 其他情况 → 调用 `arm_security_space_below_el3()`

这恰好对应规范“EL3 总是 Secure/Root；低 EL 才由 SCR_EL3 选择”的层次。

#### `arm_security_space_below_el3()`：判定“EL3 以下”状态

`target/arm/helper.c:10163-10187`：

```c
if (!(env->cp15.scr_el3 & SCR_NS)) {
    return ARMSS_Secure;
} else if (env->cp15.scr_el3 & SCR_NSE) {
    return ARMSS_Realm;
} else {
    return ARMSS_NonSecure;
}
```

它还特别写了注释：

> `NSE cannot be set without RME, and NSE & !NS is Reserved. Ignoring NSE when !NS retains consistency...`

这说明 QEMU 明确知道：

- `NSE=1 & NS=0` 是**保留组合**，不是 Root；
- 当 `NS=0` 时，一律返回 `Secure`；
- `Root` 只能从“当前在 EL3 且 RME 存在”这条路径进入。

### 2.4 一个容易误读的点：`arm_is_secure()`

`target/arm/cpu.h:2219-2222`：

```c
static inline bool arm_is_secure(CPUARMState *env)
{
    return arm_space_is_secure(arm_security_space(env));
}
```

而 `arm_space_is_secure()` 定义为：

```c
return space == ARMSS_Secure || space == ARMSS_Root;
```

这不是说“Root = Secure”，而是说：**在 QEMU 的很多旧接口与布尔判断里，Root 被当作‘pre-v9 sense 的 secure side’处理**。这是一种**兼容旧二态代码的工程手段**，不是架构语义上的等价关系。对规范分析时必须把“Secure”与“Root”分开说。

---

## 3. `SCR_EL3` 控制位验证

### 3.1 与本文主题直接相关的位

`target/arm/cpu.h:1756-1798` 定义了本文关心的关键位：

| 位 | 宏 | QEMU 含义 | 规范/实现影响 |
|---|---|---|---|
| 0 | `SCR_NS` | 低 EL 是否进入 NS 侧 | 与 `NSE` 共同决定低 EL state |
| 1 | `SCR_IRQ` | IRQ 路由到 EL3 | `arm_phys_excp_target_el()` 参与查表 |
| 2 | `SCR_FIQ` | FIQ 路由到 EL3 | 同上 |
| 3 | `SCR_EA` | External Abort / SError 路由到 EL3 | 同上 |
| 7 | `SCR_SMD` | 禁用 SMC（条件成立时变 UNDEF） | `pre_smc()` 检查 |
| 8 | `SCR_HCE` | 允许 HVC | `§D1.4.9` 对 HVC 可用性有约束 |
| 10 | `SCR_RW` | 低于 EL3 的执行状态宽度选择 | 异常入口向量偏移、EL1/EL2 执行态 |
| 18 | `SCR_EEL2` | Secure EL2 Enable | 影响 `EL2Enabled()` / `arm_hcr_el2_eff()` |
| 62 | `SCR_NSE` | Realm 相关选择位 | 与 `NS` 共同决定 Realm |

**重点校正**：QEMU 代码里 `SCR_EEL2` 明确是 **bit 18**，不是 bit 21。

### 3.2 `scr_write()` 的验证要点

`target/arm/helper.c:712-836` 有 4 个关键行为：

#### (1) 架构约束位的掩码化

- AArch64 EL3 下强制 `SCR_FW | SCR_AW` 为 `RES1`
- `SCR_NET` 按 `RES0` 清理
- 若不支持 AArch32 lower EL，则 `SCR_RW` 变成 `RAO/WI`
- 根据 `aa64_sel2`、`aa64_rme` 等特性动态扩展 `valid_mask`

这说明 QEMU 没把 `SCR_EL3` 当成“裸 64 位寄存器”，而是显式编码了规范要求的 `RES0/RES1/RAO-WI` 语义。

#### (2) `FEAT_SEL2` 与 `FEAT_RME` 的联动

```c
if (cpu_isar_feature(aa64_sel2, cpu)) {
    valid_mask |= SCR_EEL2;
} else if (cpu_isar_feature(aa64_rme, cpu)) {
    value |= SCR_NS;
}
...
if (cpu_isar_feature(aa64_rme, cpu)) {
    valid_mask |= SCR_NSE | SCR_GPF;
}
```

两点很关键：

- `SCR_EEL2` 只有在 `FEAT_SEL2` 存在时才有意义；
- `SCR_NSE` 与 `SCR_GPF` 只有在 `FEAT_RME` 存在时才开放；
- 若有 RME 但无 SEL2，QEMU 还会把 `NS` 强制为 1，这与现代规范对 RME/SEL2 组合的约束保持一致。

#### (3) 无 EL2 时 `HCE`/`SMD` 的有效性受限

QEMU 对 `HCE` 和部分 ARMv7 场景下的 `SMD` 做了 `valid_mask` 清理。这与规范“没有相应异常级别/扩展时控制位无效”的思想一致。

#### (4) `NS/NSE` 改变时刷新 EL3 以下 TLB

`target/arm/helper.c:818-835`：

```c
if (changed & (SCR_NS | SCR_NSE)) {
    tlb_flush_by_mmuidx(... EL3 以下全部 MMU index ...);
}
```

这与 `DDI 0487 §D1.1.2` 的抽象模型一致：**安全状态变化不是一个普通控制位变化，而是地址空间与权限域的切换**。QEMU 因此把 EL3 以下相关 TLB 全部失效。注意：

- 刷新的目标是 **EL3 以下**；
- 这再次证明 **EL3 自身不通过 `NS/NSE` 在 Secure/Non-secure/Realm 间切换**；
- 如果实现了 RME，则 `NSE` 变化同样触发刷新，因为 Secure / Non-secure / Realm 的翻译上下文不同。

### 3.3 `NS / NSE / EEL2 / RW / HCE / SMD` 的规范化理解

| 控制位 | 规范语义 | QEMU 验证结论 |
|---|---|---|
| `NS` | 低 EL 安全态选择的一部分 | 正确；但在 RME 中**不是唯一控制位** |
| `NSE` | RME 下与 `NS` 共同决定低 EL state | 正确；`NSE & !NS` 视为保留 |
| `EEL2` | 仅影响 Secure EL2 是否存在 | 正确；不决定 Root/Realm |
| `RW` | 控制低 EL 执行状态宽度 | 正确；异常入口也依赖它决定向量偏移 |
| `HCE` | HVC 是否有效 | 正确；需 EL2 存在且当前安全态启用 EL2 |
| `SMD` | SMC 是否 UNDEF / 限制 | 正确；且 `HCR_EL2.TSC` 优先级更高 |

---

## 4. 世界切换验证

### 4.1 SMC 到 EL3：不是“自动换世界”，而是“先进入 EL3”

`DDI 0487 §D1.4.9` 先把 SMC 归类为 **exception-generating instruction**，这意味着它属于**指令触发的同步异常语义**。虽然 JSONL OCR 文本中有一处把 SMC 描述成 asynchronous exception，但从章节归属、规则上下文和 QEMU 实现都能确认：这里应按**同步异常**理解。

QEMU 的实现也完全符合这一点：

- `translate-a64.c:3193-3204` 中 `trans_SMC()`：
  - 先执行 `gen_helper_pre_smc()`
  - 然后生成 `EXCP_SMC`，目标 EL 固定写成 3
- `tcg/op_helper.c:1111-1200` 中 `pre_smc()`：
  - 先检查是否应 UNDEF
  - 再检查 `HCR_EL2.TSC` 是否应优先陷入 EL2
  - 最后才允许进入 EL3

因此，**SMC 的第一步不是从 NS 直接跳进 Secure EL1，而是同步异常进入 EL3 Monitor/Root Monitor**。

### 4.2 `HCR_EL2.TSC` 对 SMC 的优先级

`pre_smc()` 中明确写道：

```c
if (cur_el == 1 && (arm_hcr_el2_eff(env) & HCR_TSC)) {
    raise_exception(env, EXCP_HYP_TRAP, syndrome, 2);
}
```

这与 `DDI 0487 §D1.4.9` 一致：当 `SMC` 被 `HCR_EL2.TSC` 抢先 trap 时，它**不会进入 EL3**。

因此世界切换链路应理解为：

1. 先判断 SMC 是否 UNDEF；
2. 再判断是否 trap 到 EL2；
3. 只有剩余情况才真正把 SMC 送往 EL3；
4. EL3 软件再决定是否修改 `SCR_EL3` 并 `ERET` 到另一个世界。

### 4.3 异常进入 EL3 时，规范要求改什么

`DDI 0487 §D1.4.2` 规定：当异常取到某个 ELx 并使用 AArch64 状态时：

- `SPSR_ELx` 保存异常前 PSTATE
- `ELR_ELx` 保存返回地址
- PSTATE 更新为异常后状态
- `ESR_ELx` 在同步异常/SError 场景写入 syndrome
- 从对应异常向量开始执行

QEMU 在 `helper.c` 的 AArch64 异常入口路径中做了同样的事：

- 保存 `old_mode`
- 保存 `ELR_EL3`
- 写 `banked_spsr[aarch64_banked_spsr_index(new_el)]`
- `pstate_write(env, PSTATE_DAIF | new_mode)`
- 切到 `SP_EL3`
- `arm_rebuild_hflags(env)`
- `env->pc = VBAR_EL3 + vector_offset`

### 4.4 真正改变世界的是“EL3 修改 SCR_EL3 后的异常返回”

`DDI 0487 §D1.4.4` 规定，合法 `ERET` 返回时：

- PSTATE 从 `SPSR_ELx` 恢复
- PC 从 `ELR_ELx` 恢复
- 本地 exclusives monitor 被清空
- Event Register 被设置

但**规范并没有说“进入 EL3 就自动把低 EL 世界也一起改了”**。低 EL 的 Security state 仍由 `SCR_EL3.{NSE,NS}` 的 effective value 决定；因此真正的“世界切换”顺序应写成：

1. NS/S/Realm 侧通过 SMC 进入 EL3；
2. EL3 固件保存软件上下文；
3. EL3 写 `SCR_EL3`（必要时改 `NS` / `NSE` / `EEL2` / 路由位）；
4. QEMU 因 `scr_write()` 对 `NS/NSE` 变化刷新 EL3 以下 TLB；
5. `ERET` 返回到目标 EL；
6. 返回后的 `arm_security_space_below_el3()` 与 `arm_mmu_idx_el()` 才开始按新状态运行。

换言之：**进入 EL3 是“到切换控制点”；修改 SCR_EL3 + ERET 才是“完成切换”**。

### 4.5 世界切换后到底改变了什么

#### (1) AArch32 banked 通用寄存器 / SPSR

`helper.c:8298-8306` 等处可见：

- `banked_r13[]`
- `banked_spsr[]`

AArch32 模式下，异常模式寄存器 bank 切换是显式存在的。

#### (2) Secure / Non-secure banked 系统寄存器

QEMU 在 CP 寄存器定义里大量使用 `bank_fieldoffsets`，如：

- `helper.c:7344-7345`：`sctlr_s / sctlr_ns`
- `helper.c:2858-2859`：`ttbr0_s / ttbr0_ns`
- `helper.c:7327-7328`：`vbar_s / vbar_ns`
- `helper.c:1015-1016`：`mair0_s / mair0_ns`

这说明切世界后，**低 EL 看到的系统寄存器银行确实不同**。

#### (3) 地址空间与 TLB 上下文

- `cpu.c:2294-2301`：CPU 注册了 `cpu-memory` 与 `cpu-secure-memory`
- `cpu.h:2601-2613`：`arm_asidx_from_attrs()` / `arm_addressspace()` 根据 `attrs.secure` 选 `NS/S AddressSpace`
- `memattrs.h:25-36`：`MemTxAttrs` 同时带 `secure` 与 `space`
- `scr_write()`：`NS/NSE` 改变时刷新 EL3 以下全部相关 TLB

所以世界切换至少同时牵涉：

- banked 系统寄存器视图
- TLB / MMU index
- 总线事务属性（`secure` / `space`）
- 某些外设访问可见性

#### (4) 不会自动改变的东西

- AArch64 通用寄存器 `X0-X30` 并不存在 Secure/NS 双银行；
- 世界切换时是否保存/恢复它们，取决于 EL3 固件软件约定，不是硬件自动 bank；
- QEMU 也没有把 AArch64 GPR 设计成 Secure / Non-secure 双份拷贝。

这点对理解 OP-TEE / TF-A 风格切换尤其关键：**硬件负责异常级别与部分架构状态，软件负责大部分世界上下文切换。**

---

## 5. RME 四域模型验证

### 5.1 规范上，RME 不只是“多了一个 Realm”

`DDI 0487 §D1.1.2` 讲的是两层扩展：

1. **执行状态层**：从 Secure / Non-secure 扩展成 Secure / Non-secure / Realm / Root；
2. **PA 空间层**：从 Secure / Non-secure PA space 扩展成 Secure / Non-secure / Realm / Root PA space。

因此 RME 不是“TrustZone + 一个 Realm 侧标记位”，而是把 **执行态、地址空间、翻译检查、异常模型** 一起扩展了。

### 5.2 QEMU 对四域的建模是存在的

从 `ARMSecuritySpace` 与 `arm_space_to_phys()` / `arm_phys_to_space()` 可见：

- `Secure`
- `NonSecure`
- `Root`
- `Realm`

都被映射到独立的物理 MMU index：

```c
QEMU_BUILD_BUG_ON(ARMMMUIdx_Phys_Root != ARMMMUIdx_Phys_S + ARMSS_Root);
QEMU_BUILD_BUG_ON(ARMMMUIdx_Phys_Realm != ARMMMUIdx_Phys_S + ARMSS_Realm);
```

这说明 QEMU 内核层并不是“假装支持 RME”，而是真把 `Root/Realm` 放进了地址翻译与 fault 判断路径。

### 5.3 但板级 `AddressSpace` 仍主要只有 NS / S 两套

`cpu.c:2294-2301` 只注册：

- `ARMASIdx_NS`
- `ARMASIdx_S`

也就是说：**板级 memory topology 仍偏经典 TrustZone 两路**。RME 的额外区分更多体现在：

- `MemTxAttrs.space`
- `ARMMMUIdx_Phys_Root / Phys_Realm`
- GPT 检查
- 输出物理空间选择

因此若从“QEMU machine board 是否为 Root/Realm 分配独立 AddressSpace”这个角度看，答案是**没有完全板级化**；但若从 CPU / MMU / PTW / fault 模型看，答案是**已经实现四域语义**。

### 5.4 GPT：QEMU 的关键验证点

`DDI 0487 §D1.4.5.3` 规定：若实现 `FEAT_RME` 且 `GPCCR_EL3.GPC=1`，物理地址访问要做 Granule Protection Checks，可能产生：

- GPF at EL3
- GPF at EL2/EL1/EL0
- GPT walk fault
- GPT address size fault
- Synchronous External abort on GPT fetch

QEMU 的实现集中在 `ptw.c`：

#### GPT 检查使用 Root 空间访问 GPT 本身

`ptw.c:339-342`：

```c
MemTxAttrs attrs = {
    .secure = true,
    .space = ARMSS_Root,
};
```

#### 最终物理地址翻译成功后仍可追加 GPC 检查

`ptw.c:3811-3829`：

- 若 `GPCCR_EL3.GPC == 1`
- 构造 `ARMGranuleProtectionConfig`
- `gpt_as = arm_addressspace(env_cpu(env), attrs)`
- 执行 `arm_granule_protection_check()`
- 失败则记为 `ARMFault_GPCFOnOutput`

这与规范精神一致：**RME 不是只在页表阶段做权限，而是物理输出地址还要过 GPT 保护检查。**

### 5.5 Realm 的几个实现细节

#### 低 EL Realm 由 `NS=1, NSE=1` 选择

`arm_security_space_below_el3()` 已给出：

- `NS=0` → Secure
- `NS=1, NSE=0` → Non-secure
- `NS=1, NSE=1` → Realm

#### Root 不是下层输出态

`ptw.c:2239-2250` 对 EL3 regime 的输出地址空间处理写得很清楚：

```c
nse = extract32(attrs, 11, 1);
out_space = (nse << 1) | ns;
if (out_space == ARMSS_Secure && !cpu_isar_feature(aa64_sel2, cpu)) {
    out_space = ARMSS_NonSecure;
}
```

这里 `out_space = (nse << 1) | ns` 只是**输出 PA space 编码**的一种中间表达；它并不意味着“EL2/EL1 可以处于 Root 执行态”。架构层的规则仍然是：**EL2 and lower cannot be in Root state**。

#### Realm Stage-2 的 bit55 含义发生变化

`ptw.c:2215-2223` 注释：

- 在 Realm security state 的 stage-2 中，`bit55` 是 `NS`
- 从 non-Realm 取指会导致 stage-2 permission fault

这对应 RME 对 stage-2 输出空间与可执行性的特殊扩展，不再是传统 NS/S 两态能覆盖的行为。

### 5.6 对“四域模型”的一句准确定义

若要用一句话概括：

> **QEMU 现状不是“板级四套总线地址空间”，而是“CPU/MMU/PTW/fault 层实现四域语义，板级内存拓扑仍以 NS/S 为主、RME 通过 GPT 与 security-space 元数据补足”。**

---

## 6. 中断路由与安全状态

### 6.1 先分清两层：GIC 组别信号映射 vs CPU 目标 EL 路由

这是现有分析文档最容易混在一起的地方。实际上有两层决策：

1. **GIC 层**：某个 pending interrupt 最终表现为 IRQ 线还是 FIQ 线；
2. **CPU 异常路由层**：收到 IRQ/FIQ/SError 后，根据 `SCR_EL3` 与 `HCR_EL2` 决定 target EL。

### 6.2 CPU 异常路由：`SCR_EL3.{IRQ,FIQ,EA}` 真正决定“是否上送 EL3”

`target/arm/helper.c:8369-8421` 的 `arm_phys_excp_target_el()`：

- `EXCP_IRQ / EXCP_NMI` → 看 `SCR_IRQ` + `HCR_IMO`
- `EXCP_FIQ` → 看 `SCR_FIQ` + `HCR_FMO`
- 其他异步物理异常 → 看 `SCR_EA` + `HCR_AMO`
- 再把 `HCR_TGE` 并入 EL2 强制路由条件
- 最终查 `target_el_table`

这与 `DDI 0487 §D1.4.x` 的路由模型一致：

- `SCR_EL3` 路由位优先决定是否进 EL3
- `HCR_EL2.{IMO,FMO,AMO}` 决定是否在低于 EL3 时改送 EL2
- `TGE` 可把部分原本去 EL1 的情况改送 EL2

因此，**“安全中断”并不自动等于“进 EL3”**；真正使其到 EL3 的，是 `SCR_EL3` 的路由控制位。

### 6.3 GIC Group 到 IRQ/FIQ 的 QEMU 映射

`hw/intc/arm_gicv3_cpuif.c:1067-1083`：

| 组 | QEMU 信号规则 | 含义 |
|---|---|---|
| `GICV3_G0` | 总是 FIQ | 典型 EL3/最高优先级路径 |
| `GICV3_G1` | 当前不安全或当前在 AArch64 EL3 时 → FIQ，否则 IRQ | Secure Group1 跨世界时常转为 FIQ |
| `GICV3_G1NS` | 当前安全时 → FIQ，否则 IRQ | Non-secure Group1 对 Secure side 作为 FIQ 进入 |

这说明 QEMU 对 GICv3 安全组的处理不是“Secure 一律 FIQ、NS 一律 IRQ”那么简单，而是依赖**当前 CPU 所在 security side**。

### 6.4 Distributor 侧的安全可见性

现有文档 `23` 的主结论基本正确：当 GIC security enabled 时，Non-secure 软件不能任意看到/改动 Secure 组配置。QEMU 在 Distributor 里通过 `IGROUPR` / `IGRPMODR` 与访问过滤实现这件事。

这与规范的高层语义一致：**安全世界与非安全世界不只体现在 CPU 上，也体现在中断控制器可见性上。**

### 6.5 `gicv3_use_ns_bank()` 的含义

`hw/intc/arm_gicv3_cpuif.c:40-48`：

```c
return !arm_is_secure_below_el3(env);
```

注意它用的是 **below_el3**，不是 `arm_is_secure()`。这很合理，因为 GIC bank 选择要回答的问题是：

> “当前 EL3 以下视角应使用 Secure bank 还是 Non-secure bank？”

而不是“当前 CPU 当前态是否把 Root 也算 secure-like”。这也再次说明：**Root 与 Secure 不能在所有层面混为一谈。**

---

## 7. QEMU 实现验证

### 7.1 `ARMSecuritySpace`：四态建模成立

验证结论：**成立**。

- 枚举显式包含 `Secure / NonSecure / Root / Realm`
- `arm_security_space()` 区分“当前 EL3”与“EL3 以下”
- `arm_security_space_below_el3()` 不会把 `NSE=1,NS=0` 解释成 Root

### 7.2 `arm_is_secure()`：是“兼容旧接口”的布尔投影

验证结论：**成立，但语义需谨慎表述**。

- Root 会被视作 secure-like
- 适合旧代码里用布尔分支判断“是否属于安全侧”
- 不适合直接拿来替代架构层四态分析

### 7.3 `arm_hcr_el2_eff_secstate()`：正确体现 `EEL2`

验证结论：**成立**。

`helper.c:3824-3840` 明确写道：若当前 Security state 下 EL2 不启用，则 `HCR_EL2` **整体无效并返回 0**。而 `arm_is_el2_enabled_secstate()` 又要求：

```c
arm_feature(env, ARM_FEATURE_EL2)
&& (space != ARMSS_Secure || (env->cp15.scr_el3 & SCR_EEL2));
```

因此：

- NS / Realm：有 EL2 就能启用
- Secure：必须 `EEL2=1`
- Root：不走这条路径

这与 `§D1.1.2` 对 Secure EL2 的约束吻合。

### 7.4 TCG 如何跟踪安全状态

验证结论：**通过 MMU index + hflags 重建，而不是单独维护一个“大而全”的 security-state 变量**。

关键路径：

- `helper.c:9957-10007`：`arm_mmu_idx_el()` 根据 EL、`arm_is_secure_below_el3()`、`HCR_E2H/TGE` 等选择 MMU index
- `tcg/hflags.c:506-523`：`rebuild_hflags_internal()` / `arm_rebuild_hflags()` 重新生成 `env->hflags`
- `cpu.h:2470`：AArch32 还专门缓存 `TBFLAG_A32.NS`

也就是说：

- **AArch64** 更依赖 `MMUIDX` / hflags 的综合结果来反映当前翻译上下文；
- **AArch32** 还会显式缓存“访问 banked cpreg 时用 NS 还是 S bank”；
- `scr_write()`、异常进入、异常返回等只要改变了相关架构状态，都会触发 `arm_rebuild_hflags()` 或后续重新计算路径。

### 7.5 地址空间实现的边界

验证结论：**经典 TrustZone 的 NS/S AddressSpace 明确存在；RME 的 Root/Realm 更多在 PTW / attrs / MMU index 层实现。**

这意味着：

- 若问题是“QEMU 是否模拟了 Secure memory 与 Non-secure memory 的分离？” → **是**；
- 若问题是“QEMU board 层是否显式建了 Root memory / Realm memory 两套独立 AddressSpace？” → **通常不是**；
- 若问题是“QEMU CPU/MMU 层是否理解 Root/Realm 并做 GPT/GPC 检查？” → **是**。

---

## 8. 既有文档勘误（5 篇）

### 8.1 总览表

| 文档 | 结论 | 主要结论 |
|---|---|---|
| `08-TrustZone安全扩展与Secure-World深度分析.md` | ✗ | 总体框架对，但 `SCR_EEL2` 位号写错；“`SCR_EL3.NS` 是唯一世界切换控制位”在 RME 下不成立 |
| `23-ARM64安全与非安全中断路由流转深度分析.md` | ✓ | 中断组、CPU 信号映射、SCR/HCR 路由主结论基本成立；建议补充“GIC 线级映射 ≠ target EL 决策” |
| `37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md` | ✗ | 把 Root 写成 `NSE:NS=1:0` 是实质性错误；Root 仅属于 EL3，`NSE=1,NS=0` 是保留组合 |
| `39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md` | ✗ | 大体流程对，但 `SCR_EEL2` 位号写成 21；应进一步强调“进 EL3 不等于低 EL 世界已切换” |
| `48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md` | ✓ | 对 `SCR_EEL2`、`SCR_NSE`、四态与 TLB 刷新描述较准确，但应补一句 Root 不是 `NS/NSE` 给低 EL 选出的状态 |

### 8.2 分文档说明

#### 文档 08

优点：

- 已意识到 ARMv9/RME 扩展为四态；
- `arm_security_space()` / `arm_security_space_below_el3()` 的主体逻辑引用正确；
- 对 `SCR_NS` 改变触发 TLB flush 的理解正确。

问题：

1. `SCR_EEL2` 写成 **bit 21**，与 `target/arm/cpu.h:1774` 不符，应为 **bit 18**。  
2. “`SCR_EL3.NS` 是唯一的世界切换控制位”只适用于传统 Secure/NS 二态；在 RME 下至少还要考虑 `NSE`，而 Secure EL2 是否可落点还受 `EEL2` 约束。  
3. 若按规范措辞，应该把“当前 EL3 是 Secure/Root”与“EL3 以下由 `NS/NSE` 选择”分开写，避免读者误解成 EL3 也受 `NS/NSE` 驱动。

#### 文档 23

优点：

- 对 GICv3 Group0 / Group1S / Group1NS 的映射关系把握较好；
- 对 `SCR_EL3.{IRQ,FIQ,EA}` 与 `HCR_EL2.{IMO,FMO,AMO}` 的联动理解基本正确；
- 对“跨安全世界的中断往往以 FIQ 呈现”的 QEMU 行为描述贴近源码。

建议补充而非推翻：

- 需要更明确区分“**GIC 把中断作为 IRQ/FIQ 送给 CPU**”与“**CPU 再根据 SCR/HCR 决定 target EL**”这两层；
- 若把这两层混写，读者容易以为“FIQ 一定去 EL3”或“Secure 中断一定去 EL3”，这在规范上并不严谨。

#### 文档 37

这是 5 篇里**最需要修正**的一篇。

问题 1：把 Root 写成 `NSE:NS=1:0`。  
这与 `DDI 0487 §D1.1.2` 和 QEMU `helper.c:10175-10183` 都不一致。正确说法应为：

- `NS=0` → Secure
- `NS=1, NSE=0` → Non-secure
- `NS=1, NSE=1` → Realm
- `Root` 仅在“当前执行于 EL3 且实现 FEAT_RME”时成立

问题 2：文中“进入 EL3 时自动切换到 Secure/Root 世界”的说法需要收束语义。  
更准确的表述是：

- **当前执行上下文**在进入 EL3 后当然处于 Secure/Root Monitor；
- 但**EL3 以下的目标世界**并未自动切换，仍取决于 EL3 软件是否修改 `SCR_EL3` 并最终 `ERET` 返回。

#### 文档 39

优点：

- `trans_SMC()` / `pre_smc()` / `EXCP_SMC` / 异常入口 / `ERET` 的主流程整理得较完整；
- 已经认识到 `HCR_TSC` 可以优先把 SMC 抢到 EL2。

问题：

1. `SCR_EEL2` 位号写成 **21**，应更正为 **18**。  
2. 建议把“EL3 执行中修改 `SCR_EL3.NS` 后执行 `ERET`”写成**世界切换完成点**，而不是把“异常进入 EL3”本身写成完整切换。

#### 文档 48

这是 5 篇里最接近本文结论的一篇。

优点：

- `SCR_EEL2 bit18` 正确；
- `SCR_NSE bit62` 正确；
- `arm_security_space()`、`arm_is_secure()`、`arm_is_el2_enabled_secstate()` 的引用较准确；
- 已明确指出 `NS/NSE` 变化触发 EL3 以下 TLB 刷新。

建议补充：

- 可再补一句：**Root 不是 `NS/NSE` 为低 EL 选出的状态，而是 RME 下 EL3 的专属状态**；
- 同时可把 GPT / GPC 与 `SCR_GPF` 的关系补进来，使 RME 视角更完整。

---

## 9. 总结

把本次核对压缩成 9 条结论：

1. **`DDI 0487 §D1.1.2` 明确是四态模型，不是二态模型打补丁。**  
2. **EL3 在无 RME 时恒为 Secure，在有 RME 时恒为 Root。**  
3. **EL2 及以下不能处于 Root。**  
4. **低 EL 的安全状态由 `SCR_EL3.{NS,NSE}` 选择；其中 `NS=1,NSE=1` 才是 Realm。**  
5. **`NSE=1,NS=0` 不是 Root，而是保留组合；QEMU 也没有把它解释成 Root。**  
6. **Secure EL2 不是单靠 Secure state 就成立，还必须有 `FEAT_SEL2` 且 `SCR_EL3.EEL2=1`。**  
7. **SMC 是世界切换入口，但不是世界切换完成点；完成点是 EL3 修改 `SCR_EL3` 后的 `ERET`。**  
8. **QEMU 已实现四域语义：`ARMSecuritySpace`、MMU index、GPT/GPC 检查都理解 Root/Realm；但 board address space 仍主要是 NS/S 两套。**  
9. **现有文档最关键的统一勘误有两条：`SCR_EEL2=bit18`；Root 仅属于 EL3，不是 `NSE:NS=1:0`。**

如果后续继续扩展这条专题，最自然的下一篇应当聚焦：

- `GPCCR_EL3 / GPTBR_EL3 / SCR_GPF` 的完整 RME/GPT 异常路径；
- Realm stage-2 输出地址空间与 `bit55 NS` 语义；
- `arm_mmu_idx_el()` / `ptw.c` 中 Secure / Realm / Root 的跨级翻译组合。
