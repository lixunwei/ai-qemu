# ARM64 EL 状态指令执行变化规范验证：EL1 / EL3、异常入口返回、PSTATE、指令可用性（DDI 0487 对照）

> 目的：对既有分析文档 `07` / `35` / `36` / `52` 与 Arm ARM DDI 0487 M.b（Armv9.6）Chapter D1 进行逐项交叉验证，并补充 **CPU 进入 EL1 与进入 EL3 时执行环境到底发生了什么变化**：可执行指令、PSTATE 改写、系统寄存器可见性、Trap 路由、MMU 体制、安全态。
>
> 主要规范依据：DDI 0487 §D1.1.2、§D1.4.1、§D1.4.2、§D1.4.4、§D1.4.5、§D1.4.8、§D1.5.1。
>
> 主要实现依据：
> - `target/arm/helper.c`
> - `target/arm/tcg/helper-a64.c`
> - `target/arm/tcg/op_helper.c`
> - `target/arm/tcg/translate-a64.c`
> - `target/arm/tcg/hflags.c`
> - `target/arm/cpu.h`
> - `target/arm/internals.h`

---

## 1. 概述

本文件解决两个问题：

1. **规范校验**：既有文档对“异常入口 / 异常返回 / PSTATE 保存恢复 / EL 切换”的描述，哪些与 DDI 0487 一致，哪些只是 QEMU 实现现象，哪些需要修正或补充。
2. **新分析**：当处理器进入 **EL1** 与进入 **EL3** 时，执行语义究竟有哪些关键差异：
   - 哪些指令可用；
   - 哪些系统寄存器可见；
   - Trap 会不会继续向上路由；
   - PSTATE 哪些位被强制改写；
   - 使用哪个 MMU regime；
   - 处于哪种 Security state。

### 1.1 结论先行

- **异常入口到 AArch64 ELx** 的规范核心是：`SPSR_ELx <- old PSTATE`、`ELR_ELx <- preferred return address`、`PSTATE.EL <- target EL`、`PSTATE.{D,A,I,F,SP} <- 1`、`PSTATE.{IL,nRW} <- 0`，以及按特性修改 `PAN/UAO/BTYPE/TCO/SSBS/ALLINT/EXLOCK/PACM/UINJ`。见 DDI 0487 §D1.4.2。
- **异常返回** 的规范核心是：`PSTATE <- SPSR_ELx`、`PC <- ELR_ELx`，但返回前先做“是否合法返回”检查。见 DDI 0487 §D1.4.4。
- **EL1** 是“受控的内核执行层”：可执行特权指令，但其大量系统寄存器访问和部分指令执行可被 EL2/EL3 Trap。
- **EL3** 是“最上层控制层”：没有更高 EL 再向上 Trap；它不仅自己执行环境最完整，还决定低 EL 的 Security state、SMC/HVC 路由、部分寄存器/指令 Trap 策略。无 FEAT_RME 时 EL3 在 **Secure state**；有 FEAT_RME 时 EL3 在 **Root state**。见 DDI 0487 §D1.1.2。
- **QEMU 总体实现与规范主线一致**，但有几处需要明确区分：
  - 某些既有文档把 **QEMU 的具体实现值** 写成了 **架构必然值**；
  - **非法异常返回** 的 QEMU 处理，比 DDI 0487 §D1.4.4 规定的字段恢复更保守，尤其是 `PAN`、`SS` 等位的处理不能直接当成规范结论。

### 1.2 本文与既有文档的关系

- `35`：最接近本文件的主题，偏异常进入/返回主线。
- `52`：更偏“异常处理全流程 + QEMU 调用链”。
- `07`：更偏“不同 EL 下的指令 Trap 机制”。
- `36`：更偏“执行流 / hflags / MMU / trap 体制”。

本文件不是重写它们，而是给出一个 **“规范—实现—既有文档”三方对照层**。

---

## 2. 规范核心：异常入口时发生什么（DDI 0487 §D1.4.2）

DDI 0487 §D1.4.2 对“异常被 taken 到使用 AArch64 state 的 ELx”规定得非常明确。

### 2.1 规范原文要点抽取

按 DDI 0487 §D1.4.2，异常 taken 到 AArch64 ELx 时：

1. `SPSR_ELx` 中对应字段根据 **异常前** 的 `PSTATE` 写入。
2. `ELR_ELx` 写入 **preferred exception return address**。
3. `PSTATE` 进入新的异常处理上下文。
4. 对同步异常和 SError，写入 `ESR_ELx`。
5. 从目标 EL 的异常向量开始执行。

随后，规范明确要求新 `PSTATE` 至少满足：

- `PSTATE.EL = target EL`
- `PSTATE.{D,A,I,F,SP} = 1`
- `PSTATE.{IL,nRW} = 0`
- `PSTATE.SS` 按 self-hosted debug 规则处理
- `PSTATE.PAN` 按 `SPAN`、`E2H/TGE` 等条件保留/置 1
- `PSTATE.UAO = 0`
- `PSTATE.BTYPE = 0b00`
- `PSTATE.TCO = 1`（若 FEAT_MTE）
- `PSTATE.SSBS = SCTLR_ELx.DSSBS`（若 FEAT_SSBS）
- `PSTATE.ALLINT = !SCTLR_ELx.SPINTMASK`（若 FEAT_NMI）
- `PSTATE.EXLOCK`：同 EL 异常时取 `GCSCR_ELx.EXLOCKEN`，升高 EL 时清 0（若 FEAT_GCS）
- `PSTATE.PACM = 0`（若 FEAT_PAuth_LR）
- `PSTATE.UINJ = 0`（若 FEAT_UINJ）
- 明确注明 **保持不变** 的字段只有：`DIT`、`SM`、`ZA`（按实现特性）

### 2.2 保存了哪些旧状态

规范语义不是“只保存少数位”，而是：

```text
exception entry to ELx
    old PSTATE  ------------------> SPSR_ELx
    preferred return address -----> ELR_ELx
    syndrome (if applicable) -----> ESR_ELx
    control transfer -------------> VBAR_ELx + offset
```

对 AArch64 来说，`SPSR_ELx` 是“异常前执行上下文”的结构化快照，而不是只保存 `DAIF` 或 `CurrentEL`。

### 2.3 preferred return address 的含义

根据 DDI 0487 §D1.4.1 / §D1.4.2：

- 同步异常的 return address 与“触发异常的指令”有精确定义关系；
- 配置型 Trap（configurable instruction controls）导致的异常，**preferred return address 就是触发该异常的那条指令地址**，且该指令不执行；见 §D1.4.8；
- 对异步异常、Memory Copy/Set 等特殊情形，地址语义可能有额外约束。

这点对理解 “ERET 返回后会不会重执行 trap 指令” 很关键：**通常会回到触发点重新执行/重试，除非软件先改了控制条件**。

### 2.4 QEMU 对应实现

QEMU AArch64 异常入口主函数在 `helper.c:9198-9428`：

```c
old_mode = pstate_read(env);
aarch64_save_sp(env, arm_current_el(env));
env->elr_el[new_el] = env->pc;
env->banked_spsr[aarch64_banked_spsr_index(new_el)] = old_mode;
...
pstate_write(env, PSTATE_DAIF | new_mode);
env->aarch64 = true;
aarch64_restore_sp(env, new_el);
env->pc = addr;
```

对应关系：

- `pstate_read(env)` → 取异常前完整 PSTATE 视图
- `banked_spsr[...] = old_mode` → `SPSR_ELx <- old PSTATE`
- `elr_el[new_el] = env->pc` → `ELR_ELx <- return address`
- `pstate_write(env, PSTATE_DAIF | new_mode)` → 建立新 PSTATE
- `pc = addr` → 跳转到 `VBAR_ELx + vector offset`

### 2.5 对既有文档 35 / 52 的校验

#### 2.5.1 文档 35 校验

| 既有结论（文档 35） | 规范判定 | 说明 |
|---|---:|---|
| `SPSR_ELx <- old PSTATE` | ✓ | 与 DDI 0487 §D1.4.2 一致 |
| `ELR_ELx <- return address` | ✓ | 与 §D1.4.1 / §D1.4.2 一致 |
| `CurrentEL <- new_el` | ✓ | 与 §D1.4.2 一致 |
| `SP=1，进入 ELxh` | ✓ | 与 §D1.4.2 一致 |
| `DAIF 全部置 1` | ✓ | 与 §D1.4.2 一致 |
| `nRW=0, IL=0` | ✓ | 与 §D1.4.2 一致 |
| `PAN 条件保留/置 1` | ✓ | 与 §D1.4.2 一致 |
| `TCO=1 / SSBS / ALLINT` 条件设置 | ✓ | 与 §D1.4.2 一致 |
| `SS = 0` | ✗ | 规范写的是“按 self-hosted debug 规则处理”，不能简单概括为永远 0 |
| `NZCV 清零` | ✗ | 规范没有把 NZCV 列为通用必改字段；QEMU 实现写成 0 不能直接上升为架构结论 |
| 未提 `UAO=0` | ✗ | §D1.4.2 明确要求 `UAO=0` |
| 未提 `EXLOCK/PACM/UINJ` | ✗ | 属于规范遗漏，不影响主线，但若做完整规范对照需补上 |

#### 2.5.2 文档 52 校验

| 既有结论（文档 52） | 规范判定 | 说明 |
|---|---:|---|
| `PSTATE -> SPSR_ELx` 保存 | ✓ | 与 §D1.4.2 一致 |
| 新 `PSTATE` 的 `EL/DAIF/SP/nRW/IL/BTYPE/TCO` 调整 | ✓ | 主体正确 |
| `BTYPE=0` | ✓ | §D1.4.2 明确给出 |
| `SSBS/ALLINT` 取决于 `SCTLR_ELx` | ✓ | 与 §D1.4.2 一致 |
| `SS=0` | ✗ | 仍是过度简化 |
| `IL=0` | ✓ | 与规范一致 |
| `PAN` 受 `SPAN` 等条件影响 | ✓ | 与规范一致 |
| 未写 `UAO=0` | ✗ | 规范有，文档遗漏 |
| 把 `NZCV` 写成固定清零 | ✗ | 这是 QEMU 写法，不应直接视作架构强制 |

### 2.6 例外：QEMU 与规范粒度不同的地方

需要特别区分：

- **规范说的是“架构可观察语义”**；
- **QEMU 写的是“模拟器内部状态落地方式”**。

例如：

- `pstate_write(env, PSTATE_DAIF | new_mode)` 会把 `NZCV/BTYPE/UAO/...` 中未显式带入的位写成某个具体值；
- 但 DDI 0487 §D1.4.2 并没有把所有这些位都定义成“入口后必须为该值”；
- 所以文档若直接写“异常入口时 NZCV 清零”，就会把 **QEMU 实现细节** 误写成 **架构必然行为**。

### 2.7 异常入口状态图

```text
异常前（old context）
    PSTATE_old
    PC_old
    SP_old
        |
        | take exception to ELx
        v
+-----------------------------+
| SPSR_ELx <- PSTATE_old      |
| ELR_ELx  <- return address  |
| ESR_ELx  <- syndrome        |
| PSTATE.EL <- x              |
| PSTATE.DAIF <- 1111         |
| PSTATE.SP <- 1              |
| PSTATE.nRW <- 0             |
| PSTATE.IL <- 0              |
+-----------------------------+
        |
        v
VBAR_ELx + vector offset
```

---

## 3. 规范核心：异常返回时发生什么（DDI 0487 §D1.4.4）

### 3.1 哪些指令属于异常返回指令

DDI 0487 §D1.4.4 明确：AArch64 的异常返回指令是：

- `ERET`
- `ERETAA`
- `ERETAB`

并且：

- **EL0 执行这些指令是 UNDEFINED**。

QEMU 翻译器对应实现：`translate-a64.c:1951-2005`。

### 3.2 合法异常返回的规范语义

DDI 0487 §D1.4.4.1 规定，合法返回时：

1. `PSTATE` 中与 `SPSR_ELx` 对应的字段按 `SPSR_ELx` 描述恢复；
2. `PC <- ELR_ELx`；
3. 若返回 AArch32，要按 `T` 位规则屏蔽 `ELR_ELx` 低位；
4. `PSTATE <- SPSR_ELx`；
5. `PACM` 可能被强制清 0；
6. Event Register 被置位；
7. local exclusive monitor 被清；
8. 返回后 `ELR_ELx` 与 `SPSR_ELx` 变为 UNKNOWN；
9. 若 `SCTLR_ELx.EOS=1`，返回是 context synchronization event。

可以抽象成：

```text
legal ERET from ELx
    SPSR_ELx -----> PSTATE
    ELR_ELx  -----> PC
```

### 3.3 非法异常返回的规范语义

DDI 0487 §D1.4.4.2 明确列出非法返回条件，包括但不限于：

- 返回到更高 EL；
- 返回到未实现 EL；
- FEAT_RME 下 EL3 返回到不允许的 lower EL security state；
- `HCR_EL2.TGE=1` 却返回 EL1；
- 返回 EL2 但 `SCR_EL3.NS/EEL2` 组合不允许；
- `M[4]` / `M[3:0]` 非法；
- 试图按 AArch32 返回到一个只支持 AArch64 的 EL；
- GCS `EXLOCKEN` 约束不满足。

非法返回时，规范要求：

- `PC <- ELR_ELx`
- `PSTATE.IL <- 1`
- `PSTATE.{EL,nRW,SP}` **保持不变**
- `PSTATE.{N,Z,C,V,D,A,I,F}` 从 `SPSR_ELx` 恢复
- `PSTATE.SS` 按“合法返回”的规则处理
- 若 FEAT_PAN，则 `PSTATE.PAN` 从 `SPSR_ELx` 恢复
- `UAO/DIT/TCO/SSBS/BTYPE/PACM` 等为 UNKNOWN（按特性）
- `ALLINT/PM/UINJ` 等按规范各自处理

### 3.4 QEMU 对应实现

QEMU 的 AArch64 异常返回 helper 在 `target/arm/tcg/helper-a64.c:622-785`。

#### 3.4.1 合法返回路径

核心代码：

```c
spsr = env->banked_spsr[spsr_idx];
new_el = el_from_spsr(spsr);
...
pstate_write(env, spsr);
aarch64_restore_sp(env, new_el);
env->pc = new_pc;
```

这与 DDI 0487 §D1.4.4.1 主线一致：

- `SPSR_ELx -> PSTATE`
- `ELR_ELx -> PC`
- `SP` 按返回后的 `PSTATE.SP` 选择 `SP_ELx` 或 `SP_EL0`

#### 3.4.2 非法返回路径

QEMU 的非法返回处理：

```c
env->pstate |= PSTATE_IL;
env->pc = new_pc;
spsr &= PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT;
spsr |= pstate_read(env) & ~(PSTATE_NZCV | PSTATE_DAIF | PSTATE_ALLINT);
pstate_write(env, spsr);
```

语义是：

- `PC` 仍从 `ELR_ELx` 恢复；
- `IL=1`；
- `EL/nRW/SP` 不变；
- 只把 `NZCV/DAIF/ALLINT` 从 `SPSR` 恢复，其余大多保留当前值。

这与 DDI 0487 §D1.4.4.2 **不完全等价**：

- 规范要求 `PAN` 在非法返回时也按 `SPSR_ELx` 恢复；
- 规范还单独规定 `SS`、`UAO/DIT/TCO/SSBS/BTYPE/PACM` 的处理；
- QEMU 采用了更简化的近似模型。

### 3.5 对既有文档 35 / 52 的校验

#### 3.5.1 文档 35 校验

| 既有结论 | 规范判定 | 说明 |
|---|---:|---|
| `ERET` 在 EL0 UNDEF | ✓ | 与 §D1.4.4 一致 |
| `PSTATE <- SPSR` | ✓ | 合法返回主线正确 |
| `PC <- ELR` | ✓ | 合法返回主线正确 |
| 非法返回：`PC <- ELR` 且 `IL=1` | ✓ | 与 §D1.4.4.2 一致 |
| 非法返回：仅恢复 `NZCV/DAIF(+ALLINT)` | ✗ | 这是 QEMU 当前实现，不是规范完整语义 |
| 非法返回：`EL/nRW/SP` 不变 | ✓ | 与 §D1.4.4.2 一致 |

#### 3.5.2 文档 52 校验

| 既有结论 | 规范判定 | 说明 |
|---|---:|---|
| `ERET/ERETA` 翻译期 EL0 不可用 | ✓ | 与 §D1.4.4 一致 |
| `HELPER(exception_return)` 主路径恢复 `PSTATE/PC/SP` | ✓ | 与 §D1.4.4.1 一致 |
| `el_from_spsr()` 解析目标 EL | ✓ | 与规范非法情况集合相符 |
| `HCR_EL2.TGE=1` 时返回 EL1 非法 | ✓ | 与 §D1.4.4.2 一致 |
| 非法返回只恢复 `NZCV+DAIF+ALLINT` | ✗ | 这是 QEMU 行为，不是完整规范 |
| 未说明 `PAN` 在非法返回时规范上应从 `SPSR_ELx` 恢复 | ✗ | 规范遗漏 |

### 3.6 异常返回状态图

```text
ERET / ERETAA / ERETAB
        |
        v
   读取 SPSR_ELx / ELR_ELx
        |
   +----+-------------------+
   |                        |
   | legal return           | illegal return
   |                        |
   v                        v
PSTATE <- SPSR_ELx      PSTATE.IL <- 1
PC <- ELR_ELx           EL/nRW/SP unchanged
SP select restored      NZCV/DAIF/... per spec
```

---

## 4. EL1 指令执行环境

这里回答一个更实战的问题：**CPU 进入 EL1 之后，“能执行什么、不能执行什么、会不会被 trap”到底变了什么？**

### 4.1 EL1 的本质

EL1 不是“完全自由的特权态”，而是：

- 比 EL0 多了特权指令与特权寄存器访问能力；
- 但它仍处在 **EL2/EL3 的监管之下**；
- 它的很多行为仍可能被 `HCR_EL2` / `CPTR_EL2` / `HSTR_EL2` / `HFGITR_EL2` / `CPTR_EL3` / `SCR_EL3` / `MDCR_EL3` 等寄存器向上重路由。

### 4.2 EL1 相对 EL0 的直接提升

依据 DDI 0487 §D1.4.8、§D1.5.1 以及 QEMU 的 `cp_access_ok()`：

- EL1 可以执行大多数 **PL1** 指令/系统寄存器访问；
- `MSR/MRS` 对 `.access = PL1_RW/PL1_R` 的系统寄存器变为可见；
- `ERET`、`HVC`、`SMC` 这类 EL0 不可执行的指令，在 EL1 变为可编码、可执行（但可能被动态禁用或 Trap）；
- `DAIF` / `PAN` / `UAO` / `DIT` / `SSBS` / `TCO` / `SPSel` 等 PSTATE 访问指令，EL1 可以直接使用（DDI 0487 §D1.5.1）。

QEMU 的静态权限模型很直观：

```c
// cpregs.h:1122-1126
static inline bool cp_access_ok(int current_el,
                                const ARMCPRegInfo *ri, int isread)
{
    return (ri->access >> ((current_el * 2) + isread)) & 1;
}
```

因此：

- `.access = PL1_RW`：EL1/EL2/EL3 可读写；EL0 不行。
- `.access = PL2_RW`：EL1 不行，只能 EL2/EL3。
- `.access = PL3_RW`：只有 EL3。

### 4.3 EL1 能执行但常被 EL2 Trap 的典型操作

#### 4.3.1 虚拟内存控制寄存器

`helper.c:332-343`：

- `HCR_EL2.TVM`：Trap EL1 对 `SCTLR_EL1/TCR_EL1/TTBRx_EL1/MAIR_EL1/...` 的写
- `HCR_EL2.TRVM`：Trap 对应读

这意味着：**Guest 内核虽然在 EL1，看起来可以改页表寄存器，但 Hypervisor 可以全部拦截。**

#### 4.3.2 Cache / TLB / 地址翻译维护

既有文档 `07` / `36` 的主结论是对的，QEMU 也确实这么做：

- `HCR_EL2.TSW`：Trap EL1 的 set/way cache 维护
- `HCR_EL2.TTLB / TTLBIS / TTLBOS`：Trap EL1 的 TLBI
- `HCR_EL2.AT`：Trap EL1 的 `AT S1E1*`
- `HCR_EL2.TDZ`：Trap `DC ZVA`
- `HCR_EL2.TPCP/TICAB/TOCU/TPU`：Trap 各类 cache 维护

#### 4.3.3 WFI/WFE / SMC / 系统寄存器

`op_helper.c:338-357` 体现了典型三级链路：

- EL0/EL1 的 `WFI/WFE` 先可能被 EL1 的 `SCTLR_EL1.nTWI/nTWE` 控制；
- 再可能被 EL2 的 `HCR_EL2.TWI/TWE` 控制；
- 再可能被 EL3 的 `SCR_EL3.TWI/TWE` 控制。

`pre_smc()` 还表明：

- EL1 执行 `SMC` 时，若 `HCR_EL2.TSC=1`，可先 trap 到 EL2；
- 否则再按 `SCR_EL3.SMD`、EL3 是否存在等规则决定是 trap 到 EL3 还是 UNDEF。

#### 4.3.4 HSTR_EL2 / FGT / ID traps

- `HSTR_EL2`：AArch32 系统寄存器访问分组 Trap；QEMU 在 `hflags.c:182-185`、`op_helper.c:832-849` 中体现。
- `FGT`：Fine-Grained Trap，允许 EL2 精确 Trap 某条指令或某个系统寄存器；QEMU 在 `hflags.c:374-380` 和 `translate-a64.c:2883-2892` 中实现。
- `ERET` 本身也可被 EL2 Trap：
  - `HFGITR_EL2.ERET`
  - 或 FEAT_NV 下 `HCR_EL2.NV`
  QEMU 在 `hflags.c:374-390` 与 `translate-a64.c:1961-1963` 中实现为 `TRAP_ERET`。

### 4.4 EL1 的 PSTATE.EL=01 有什么实际含义

`arm_current_el()` 在 `internals.h:489-515` 中通过 `PSTATE.CurrentEL[3:2]` 识别当前 EL。进入 EL1 后：

- `CurrentEL = 0b01`
- 若异常入口，`SP=1`，即默认使用 `SP_EL1`
- EL1 下的 MMU regime 是 `E10_1` 或 `E10_1_PAN`（`helper.c:9979-9985`）
- EL1 使用 **EL1 stage-1 页表体系**（`TTBR0_EL1/TTBR1_EL1/TCR_EL1`）
- EL1 与 EL0 共享 stage-1 地址空间框架，只是访问权限、PAN/UAO、特权检查不同

### 4.5 EL1 不是“最高特权”的三个根本原因

1. **还会被 EL2 Trap**：虚拟化控制是 EL1 最核心的约束来源。
2. **还会被 EL3 控制安全世界策略**：尤其 Secure/Non-secure/Realm 切分、SMC、部分调试与系统控制。
3. **异常路由可能被改写**：例如 `HCR_EL2.TGE`、`IMO/FMO/AMO` 会改变异常最终落点。

### 4.6 EL1 执行环境总结

可以把 EL1 记成：

```text
EL1 = OS kernel privilege
    + 可执行 PL1 指令/寄存器
    + 可运行异常处理、页表管理、TLBI、cache 维护、SVC/ERET/HVC/SMC
    - 但受 EL2 虚拟化策略监管
    - 也受 EL3 安全策略约束
```

---

## 5. EL3 指令执行环境

### 5.1 EL3 的本质

EL3 是架构定义的最高异常级别。DDI 0487 §D1.1.2 规定：

- **无 FEAT_RME**：EL3 总是在 **Secure state**；
- **有 FEAT_RME**：EL3 总是在 **Root state**；
- `EL2 and lower cannot be in Root state`。

QEMU 在 `helper.c:10131-10160` 中也严格映射：

- AArch64 且 `CurrentEL==3`：
  - 若 `aa64_rme` → `ARMSS_Root`
  - 否则 → `ARMSS_Secure`

### 5.2 EL3 为什么说“没有更高层 Trap”

DDI 0487 §D1.4.8 把 configurable instruction controls 列到 EL3 就结束了，没有 EL4。

因此对 EL3 而言：

- 不存在再向上重路由到更高 EL 的 trap；
- `PL3_RW` 系统寄存器访问在权限上到顶；
- 不会再被“更高特权级”截获。

这并不意味着 “EL3 什么都不会异常”，而是：

- 仍可能因指令未实现、编码非法、特性缺失、地址翻译失败、对齐错误等产生异常；
- 但这些异常仍然 **留在 EL3 本级** 或按 EL3 的异常入口处理，而不是被“更高 EL”接管。

### 5.3 EL3 可见的系统寄存器与控制权

EL3 至少具备以下层面的控制能力：

1. **PL3 专属寄存器可见**
   - 例如 `SCR_EL3`、`VBAR_EL3`、`ELR_EL3`、`SPSR_EL3` 等。
2. **可读写更低层共享寄存器的高权限版本**
   - 例如大量 `PL1/PL2` 资源在 EL3 也可访问。
3. **决定低 EL 的世界划分和执行宽度**
   - `SCR_EL3.{NS,NSE,EEL2,RW,...}`
4. **决定低 EL 的某些 Trap / 禁用行为**
   - `SCR_EL3.SMD/HCE/TWI/TWE/EA/IRQ/FIQ/...`
   - `CPTR_EL3` / `MDCR_EL3`

从 QEMU 视角看：

- `SCR_EL3` 定义在 `helper.c:4352+`
- `ELR_EL3` / `VBAR_EL3` / `SPSR_EL3` 也在 `helper.c` 中以 `PL3_RW` 方式建模
- `arm_security_space()`、`arm_security_space_below_el3()`、`arm_is_el2_enabled()` 等函数都以 `SCR_EL3` 为决策源头

### 5.4 EL3 对低 EL 的关键影响

#### 5.4.1 Security state

DDI 0487 §D1.1.2：

- `SCR_EL3.{NSE,NS}` 选择 EL2 及以下所处的安全空间；
- 有 FEAT_SEL2 且 `SCR_EL3.EEL2=1` 时，EL2 可处于 Secure state；
- EL2 以下永远不能是 Root state。

QEMU：

- `helper.c:10163-10187`：`SCR_EL3.NS/NSE` 决定 `ARMSS_Secure / ARMSS_Realm / ARMSS_NonSecure`
- `cpu.h:2228-2238`：`arm_is_el2_enabled()` 还要看当前 security space 与 `SCR_EL3.EEL2`

#### 5.4.2 对 HVC / SMC / 中断路由的影响

- `SCR_EL3.HCE`：影响 `HVC` 是否允许
- `SCR_EL3.SMD`：影响 `SMC` 是否禁用/UNDEF
- `SCR_EL3.IRQ/FIQ/EA`：决定物理中断/SError 是否路由 EL3
- `SCR_EL3.TWI/TWE`：可截获低 EL 的 `WFI/WFE`

#### 5.4.3 EL3 的 MMU 体制独立

QEMU `arm_mmu_idx_el()`：

- `EL3 -> ARMMMUIdx_E3`
- 这是与 EL1 的 `E10_1/E10_1_PAN` 完全不同的独立 regime。

也就是说：**EL3 不共享 EL1 的页表体制**。

### 5.5 EL3 进入异常时 PSTATE 的特点

异常进入到 EL3 仍遵循 DDI 0487 §D1.4.2 的通用规则，但与进入 EL1 相比有两个明显差异：

1. `CurrentEL` 直接变为 `0b11`；
2. `PAN` 的规则不同：规范里强制置 `PAN=1` 的场景只明确列了 EL1 和特定 EL2 情况，没有“进入 EL3 强制 PAN=1”这一条。

QEMU 也对应体现：

- `helper.c:9375-9393` 对 `PAN` 的特殊处理只覆盖 `new_el == 1` 与特定 `new_el == 2` 场景；
- 没有给 EL3 单独加“强制 PAN=1”逻辑。

### 5.6 EL3 执行环境总结

```text
EL3 = monitor / root controller
    + 无更高 EL 再向上 trap
    + 可见 PL3 寄存器
    + 可控制低 EL 的安全态/异常路由/调用约束
    + 使用独立 E3 MMU regime
    + 无 FEAT_RME 时位于 Secure，启用 FEAT_RME 时位于 Root
```

---

## 6. EL1 vs EL3 指令执行关键差异表

### 6.1 总览表

```text
+----------------------+------------------------------+-----------------------------------+
| 维度                 | EL1                          | EL3                               |
+----------------------+------------------------------+-----------------------------------+
| 当前 PSTATE.EL       | 01                           | 11                                |
| 是否最高 EL          | 否                           | 是                                |
| 安全态               | 由 SCR_EL3.{NS,NSE} 决定     | 无 RME: Secure / 有 RME: Root     |
| MMU regime           | E10_1 / E10_1_PAN            | E3                                |
| Stage-1 页表         | TTBRx_EL1 / TCR_EL1          | TTBR0_EL3 / TCR_EL3               |
| 是否可能再向上 trap  | 是，可到 EL2/EL3             | 否，无更高 EL                     |
| 可访问系统寄存器层级 | PL1 + 部分更高共享寄存器     | PL3 + 低层可见资源                |
| ERET 可用性          | 可用，但可被 EL2 trap        | 可用，不会再向上 trap             |
| HVC 可用性           | 可用，通常到 EL2             | 可用，留在 EL3                    |
| SMC 可用性           | 可用，可到 EL3/EL2/UNDEF     | 可用，留在 EL3                    |
| WFI/WFE              | 可执行，但可被 EL2/EL3 trap  | 可执行，不会再向上 trap           |
| FP/SVE/SME           | 可被 CPTR_EL2/EL3 trap       | 仅受本级 CPTR_EL3/特性控制        |
| 调试访问             | 可被 MDCR_EL2/EL3 trap       | 无更高级 trap                     |
| 异常入口 SP          | 进入 EL1h, SP=1              | 进入 EL3h, SP=1                   |
| PAN 特殊规则         | SPAN=0 时常被强制置 1        | 无对应“进入 EL3 必置 1”规则       |
| 低 EL 控制权         | 无                           | 强，可设置低 EL 世界与路由        |
+----------------------+------------------------------+-----------------------------------+
```

### 6.2 指令视角的直观差异

```text
EL0 --SVC--> EL1                : 常规 syscall / guest kernel entry
EL1 --HVC--> EL2                : hypercall
EL1 --SMC--> EL3                : secure monitor call
EL1 --ERET--> lower EL          : 可能被 EL2 trap
EL3 --ERET--> lower EL          : 无更高 EL trap，但要过合法性检查
```

### 6.3 Trap 拓扑图

```text
在 EL1 执行：
    指令/寄存器访问
        |
        +--> 本级允许执行
        |
        +--> HCR_EL2 / CPTR_EL2 / HFGITR_EL2 / HSTR_EL2 / MDCR_EL2 trap 到 EL2
        |
        +--> CPTR_EL3 / SCR_EL3 / MDCR_EL3 trap 到 EL3

在 EL3 执行：
    指令/寄存器访问
        |
        +--> 本级执行 / 本级异常
        |
        +--> 不存在更高 EL 再向上 trap
```

---

## 7. QEMU 实现验证

本节把规范语义压到 QEMU 具体代码上，确认“它到底怎么模拟”。

### 7.1 异常入口：`arm_cpu_do_interrupt_aarch64()`

入口在 `helper.c:9198-9428`。

#### 7.1.1 向量偏移计算

QEMU 先根据：

- 是否从 lower EL 来；
- lower EL 是 AArch64 还是 AArch32；
- same EL 时是否走 `SP_EL0` 还是 `SP_ELx`；
- 异常类型是 Sync / IRQ / FIQ / SError；

计算 `VBAR_ELx + offset`。这与架构向量布局吻合。

#### 7.1.2 状态保存

```c
old_mode = pstate_read(env);
aarch64_save_sp(env, arm_current_el(env));
env->elr_el[new_el] = env->pc;
env->banked_spsr[...] = old_mode;
```

这对应：

- 保存旧 PSTATE
- 保存旧 SP
- 保存返回地址

#### 7.1.3 新 PSTATE 建立

QEMU 显式实现了：

- `PAN`
- `TCO`
- `SSBS`
- `ALLINT`
- `DAIF`
- `SP=1`
- `CurrentEL`
- `nRW=0`

以及：

- same EL 异常时 `EXLOCKEN`
- AArch64 入口统一恢复 `SP_ELx`

与 DDI 0487 §D1.4.2 的主线高度一致。

### 7.2 异常返回：`HELPER(exception_return)`

实现位于 `target/arm/tcg/helper-a64.c:622-785`。

#### 7.2.1 合法性检查

QEMU 检查：

- `el_from_spsr()` 是否给出合法目标 EL
- 返回目标不能高于当前 EL
- 返回 EL2 时 EL2 必须启用
- 返回目标 EL 的执行宽度必须与 `SPSR.nRW` 一致
- AArch64-only CPU 不允许返回 AArch32
- `HCR_EL2.TGE=1` 时不允许返回 EL1
- FEAT_RME / GCS 的约束

这与 DDI 0487 §D1.4.4.2 的非法返回条件集合基本一致。

#### 7.2.2 合法返回动作

QEMU 在合法返回路径上做了这些事：

- `aarch64_save_sp(env, cur_el)`
- `arm_clear_exclusive(env)`
- `pstate_write(env, spsr)` 或 `cpsr_write_from_spsr_elx(env, spsr)`
- `aarch64_restore_sp(env, new_el)`
- `helper_rebuild_hflags_a64/a32()`
- `env->pc = new_pc`
- `aarch64_sve_change_el(env, cur_el, new_el, return_to_aa64)`

这与规范主线一致，尤其符合：

- `PSTATE <- SPSR`
- `PC <- ELR`
- 清 exclusives
- 返回后上下文切换完成

### 7.3 `hflags` 重建：EL 切换后执行环境真的变了

QEMU 不是只改 `PSTATE`，还会立即重建 **TB flags / hflags**。这一步很关键。

- 异常入口后：`helper.c:9419-9421` 调 `arm_rebuild_hflags(env)`
- AArch32 异常入口后：`helper.c:8761-8762`
- ERET 返回后：`helper-a64.c:714/728/782` 等路径调 `helper_rebuild_hflags_*`

这意味着 EL 切换会改变：

- 当前 MMU regime
- FP/SVE/SME trap 目标 EL
- VHE / NV / FGT 标志
- UAO / PAN / TBI / BTI 等翻译时行为

所以 **进入 EL1 和进入 EL3 不只是“CurrentEL 数字变了”，而是整套翻译上下文都重算了。**

### 7.4 TB 处理：不是全局失效，而是退出当前 TB

用户关注“TB invalidation on EL switch”，QEMU 这里的真实行为是：

- `arm_cpu_do_interrupt()` 在异常 taken 后对非 KVM 路径调用 `cpu_set_interrupt(cs, CPU_INTERRUPT_EXITTB)`（`helper.c:9469+` 所在路径，既有文档 52 已引用）
- `translate-a64.c:1971-1973` 在 `ERET` 翻译时把 `s->base.is_jmp = DISAS_EXIT`

因此：

- **不是粗暴 flush 全局 TB cache**；
- 而是 **强制退出当前 TB**，随后用新的 `hflags` / EL / MMU 状态重新翻译或重新进入执行流。

这是“EL 切换后翻译环境立刻生效”的关键机制。

### 7.5 MMU regime 验证

`helper.c:9957-10008`：

- EL1 → `E10_1` / `E10_1_PAN`
- EL3 → `E3`

所以进入 EL1 与进入 EL3 的地址翻译体制确实不同，不是只换个异常级别标签。

### 7.6 Security state 验证

`helper.c:10131-10160` / `10163-10187`：

- 当前在 EL3 时：
  - RME → `Root`
  - 非 RME → `Secure`
- EL3 以下：根据 `SCR_EL3.NS/NSE` 决定 `Secure / Realm / NonSecure`

这与 DDI 0487 §D1.1.2 对齐。

### 7.7 是否存在明显规范不一致

#### 一致项

- 异常入口主流程：✓
- 合法异常返回主流程：✓
- `TGE` / `RME` / `EL2 enabled` / 执行宽度检查：✓
- EL1 / EL3 MMU regime 差异：✓
- EL3 安全空间判定：✓

#### 需要标注为“实现近似”的项

- **非法异常返回字段恢复**：QEMU 简化为“恢复 `NZCV/DAIF/ALLINT`，其余多数保持当前值”，而 DDI 0487 §D1.4.4.2 规定得更细，尤其包括 `PAN`。
- **异常入口后 NZCV/SS 等位的具体取值**：QEMU 内部写法不应直接当成架构必然值。

---

## 8. 规范引用与勘误

本节专门给既有文档补“规范锚点”和“必要修正”。

### 8.1 对文档 35 的勘误建议

#### 建议修正 1：`SS=0` 不应写成架构定值

- 现有表达：异常入口时 `SS=0`
- 建议改为：
  - **QEMU 当前实现路径中通常会把 `SS` 清掉**；
  - 但 **DDI 0487 §D1.4.2** 的规范表述是：`PSTATE.SS` 按 *AArch64 Self-hosted Debug* 规则处理，不能简化成一条固定值。

#### 建议修正 2：`NZCV 清零` 不应写成规范结论

- 现有表达：异常入口 `NZCV` 清零
- 建议改为：
  - **QEMU 的 `pstate_write(PSTATE_DAIF | new_mode)` 会把未显式带入的 `NZCV` 变为 0**；
  - 但 **DDI 0487 §D1.4.2** 并未把 `NZCV` 列为通用必改字段，因此应写成“QEMU 实现现象”，而不是“架构强制”。

#### 建议修正 3：补上 `UAO=0`

- DDI 0487 §D1.4.2 明确：若实现 FEAT_UAO，则异常入口后 `PSTATE.UAO=0`。
- 现文档未写，应补。

### 8.2 对文档 52 的勘误建议

#### 建议修正 1：非法异常返回不要只写“恢复 NZCV+DAIF+ALLINT”

- 现文档是对 **QEMU helper-a64.c** 的忠实摘要；
- 但若标题/语气写成“架构定义”，会误导。

建议改写为：

> QEMU 当前简化实现中，非法异常返回路径只显式从 `SPSR_ELx` 恢复 `NZCV/DAIF/ALLINT`，并保持 `EL/nRW/SP` 不变；但 DDI 0487 §D1.4.4.2 对非法返回规定更细，尤其要求若实现 FEAT_PAN，则 `PSTATE.PAN` 也应从 `SPSR_ELx` 取值，并对 `SS/UAO/DIT/TCO/SSBS/BTYPE/PACM` 等字段分别给出规范语义。

#### 建议修正 2：补充 `ELR_ELx/SPSR_ELx becomes UNKNOWN`

DDI 0487 §D1.4.4.1 / §D1.4.4.2 都说明：

- 返回完成后，`ELR_ELx` 与 `SPSR_ELx` 变为 UNKNOWN。

QEMU 不会显式把它们清掉，但这不构成错误；应说明为：

- “规范要求 become UNKNOWN；QEMU 未随机化，保留旧值属于 UNKNOWN 的一个实现选择。”

### 8.3 对文档 07 / 36 的补强建议

这两篇整体方向基本正确，建议增强的主要是“规范锚点”：

| 主题 | 建议补充规范引用 |
|---|---|
| 可配置指令 Trap | DDI 0487 §D1.4.8 |
| PSTATE 可访问字段 | DDI 0487 §D1.5.1 |
| EL0 异常被 `HCR_EL2.TGE` 重定向 | DDI 0487 §D1.4.5.1 |
| EL3 安全态 / Root state | DDI 0487 §D1.1.2 |
| 异常入口与返回主语义 | DDI 0487 §D1.4.2 / §D1.4.4 |

### 8.4 最重要的“规范 vs QEMU”边界线

建议在后续所有 EL/异常类文档里坚持以下表达规范：

```text
如果某字段变化是 Arm ARM 明文规定：写“规范要求”。
如果某字段变化只是 QEMU 具体落地：写“QEMU 实现中……”。
如果规范允许 UNKNOWN / IMPLEMENTATION DEFINED：不要写成固定值结论。
```

---

## 9. 总结

### 9.1 关键发现

1. **异常入口到 AArch64 ELx 的真正规范核心**，是 `SPSR_ELx` / `ELR_ELx` / `PSTATE` / `ESR_ELx` / `VBAR_ELx` 五件套一起变，不只是“切 EL”。
2. **EL1 与 EL3 的最大差异** 不是“权限高低抽象概念”，而是：
   - EL1 仍会被 EL2/EL3 广泛监管；
   - EL3 没有更高 EL，可直接定义低 EL 的安全态与路由规则。
3. **QEMU 对异常入口、合法 ERET、MMU regime、安全态** 的模拟与 DDI 0487 主线一致。
4. **需要修正的主要不是 QEMU 代码，而是文档表述粒度**：
   - `SS=0`、`NZCV 清零` 这类写法过度把实现细节当成规范；
   - 非法异常返回的字段恢复，既有文档更像“QEMU 行为摘要”，不是“Arm ARM 完整语义”。

### 9.2 对既有文档的关系判断

- `35`：主框架正确，适合作为“异常进入/返回主文”；但需补规范粒度与几个字段修正。
- `52`：QEMU 调用链和实现描述很有价值；但遇到“非法返回字段语义”时必须显式标注“这是 QEMU 近似实现”。
- `07`：EL 下 trap 机制分析与本次规范验证是互补关系，建议在关键表格后补 DDI 0487 §D1.4.8 引用。
- `36`：对 hflags / MMU / trap 体制的理解与本次验证高度一致，是“进入 EL1 vs EL3 执行环境变化”的最佳实现侧配套文档。

### 9.3 一句话归纳

```text
进入 EL1：是进入“仍受监管的特权内核环境”。
进入 EL3：是进入“无更高 EL、且负责定义低 EL 规则的顶层控制环境”。
```

---

## 附：快速引用索引

- DDI 0487 §D1.1.2：Security states
- DDI 0487 §D1.4.1：Exception entry terminology
- DDI 0487 §D1.4.2：Exception entry
- DDI 0487 §D1.4.4：Exception return
- DDI 0487 §D1.4.5：Synchronous exception types
- DDI 0487 §D1.4.8：Configurable instruction controls / traps
- DDI 0487 §D1.5.1：PSTATE fields meaningful in AArch64 state

### 相关源码定位

- `helper.c:9198-9428`：AArch64 异常入口
- `helper-a64.c:622-785`：AArch64 异常返回
- `translate-a64.c:1951-2005`：`ERET/ERETA`
- `op_helper.c:1071-1200`：`pre_hvc()` / `pre_smc()`
- `helper.c:332-360`：`TVM/TRVM/TSW/TACR` Trap
- `helper.c:9957-10008`：`arm_mmu_idx_el()`
- `helper.c:10131-10187`：security space 判定
- `cpregs.h:1122-1126`：`cp_access_ok()`
- `internals.h:489-515`：`arm_current_el()`
