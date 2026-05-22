# ARM64 调试架构规范验证：DDI 0487 M.b Chapter D2/H 与 QEMU 实现对照

> 验证对象：
> - 规范：DDI 0487 M.b Chapter D2（Self-hosted debug）与 Chapter H（External debug）
> - 既有文档：
>   - `46-ARM64-TCG插件与调试子系统深度分析-PluginAPI-GDBStub-断点单步-ARM调试寄存器与Tracing.md`
>   - `53-ARM64-调试架构深度分析-MDSCR-DBGBCR-DBGWCR-断点观察点-单步执行.md`
> - QEMU 源码：`target/arm/debug_helper.c`、`target/arm/tcg/debug.c`、`target/arm/helper.c`、`target/arm/cpu.h`、`target/arm/tcg/translate*.c`、`target/arm/tcg/helper-a64.c`
>
> 结论先行：QEMU 对 **Self-hosted debug（自托管调试）** 的核心路径实现较完整，尤其是 MDSCR_EL1、硬件断点/观察点、Software Step、异常综合征与路由；但对 **External debug（外部调试）** 只实现了极小子集或占位行为，**并未真正建模 H 章定义的 Debug state / Halting debug / EDSCR.HDE / ITR/DCC 执行环境**。此外，现有文档存在若干将 **gdbstub 单步** 与 **架构 Software Step** 混写、以及把 **OSLSR_EL1[0]** 误当作锁位的错误。

---

## 1. 概述

本文件的目标不是重复讲解 ARM 调试寄存器，而是做三件事：

1. 用 DDI 0487 M.b 的 D2/H 章节核对 QEMU ARM64 调试实现是否符合架构语义。
2. 明确 **QEMU 已实现、部分实现、未实现** 的调试能力边界。
3. 对既有文档中的表述做规范化勘误，尤其区分：
   - **Self-hosted debug exception**（断点/观察点/软件单步异常）
   - **External debug halting**（进入 Debug state）
   - **QEMU gdbstub 的模拟器级单步/断点**

从源码结构看，QEMU 当前调试实现可分为三层：

- **寄存器建模层**：`target/arm/debug_helper.c`
  - 定义 MDSCR_EL1 / OSLAR_EL1 / OSLSR_EL1 / DBGBVR/DBGBCR / DBGWVR/DBGWCR 等系统寄存器。
  - guest 写 DBGBCR/DBGWCR 等寄存器后，立即通过 write hook 同步内部断点/观察点对象。
- **自托管调试执行层**：`target/arm/tcg/debug.c`
  - 决定异常是否允许产生、路由到哪个 EL、断点/观察点如何匹配、异常综合征如何构造。
- **单步状态机与翻译层**：`target/arm/tcg/hflags.c`、`translate.h`、`translate.c`、`translate-a64.c`、`helper-a64.c`
  - 将 DDI 0487 §D2.11 的 Software Step 状态机落到 TB flag、PSTATE.SS 与 helper 调用上。

一个关键事实是：

> QEMU 现在把 **运行时断点/观察点匹配逻辑** 放在 `target/arm/tcg/debug.c`，而不是旧文档容易让人以为的 `target/arm/debug_helper.c`。后者主要负责 **CP 寄存器定义与写入钩子**。

---

## 2. 自托管调试验证 (§D2)

### 2.1 Debug exceptions 的类别

按照 DDI 0487 §D2.1.3，Self-hosted debug 关注的核心异常包括：

- Breakpoint instruction exception（如 BRK）
- Breakpoint exception（硬件断点）
- Watchpoint exception（硬件观察点）
- Software Step exception（软件单步）
- 以及 Vector Catch 等其他调试事件

QEMU 对应关系如下：

| 架构概念 | QEMU 入口 | 结果 |
|---|---|---|
| Breakpoint instruction exception | `HELPER(exception_bkpt_insn)` in `debug.c:514-539` | 产生 `EXCP_BKPT` |
| Hardware breakpoint exception | `arm_debug_check_breakpoint()` + `arm_debug_excp_handler()` | 产生 `EXCP_PREFETCH_ABORT` + `syn_breakpoint()` |
| Watchpoint exception | `check_watchpoints()` + `arm_debug_excp_handler()` | 产生 `EXCP_DATA_ABORT` + `syn_watchpoint()` |
| Software Step exception | `gen_step_complete_exception()` / `gen_swstep_exception()` | 产生 `EXCP_UDEF` + `syn_swstep()` |

这与 DDI 0487 §D2.1.3、§D2.8、§D2.9、§D2.11 的分类总体一致。

### 2.2 Routing debug exceptions

DDI 0487 §D2.2 指出，Debug exception 的路由由以下控制共同决定：

- `HCR_EL2.TGE`
- `MDCR_EL2.TDE`
- 当前 Security state
- 当前 Exception level

QEMU 对应实现是 `arm_debug_target_el()`（`target/arm/tcg/debug.c:19-41`）：

```c
if (arm_is_el2_enabled(env)) {
    route_to_el2 = env->cp15.hcr_el2 & HCR_TGE ||
                   env->cp15.mdcr_el2 & MDCR_TDE;
}
if (route_to_el2) {
    return 2;
} else if (arm_feature(env, ARM_FEATURE_EL3) &&
           !arm_el_is_aa64(env, 3) && secure) {
    return 3;
} else {
    return 1;
}
```

验证结论：

- **EL2 路由**：与 DDI 0487 §D2.2 一致，`TGE` 或 `TDE` 任一为真即可把 debug target EL 设为 EL2。
- **默认路由**：落到 EL1。
- **Secure + AArch32 EL3 特判**：QEMU 仍保留 legacy monitor 风格行为，允许路由到 EL3；这更接近 Arm 架构对 AArch32 Monitor debug 模型的兼容语义，而不是纯 AArch64-only 视角下的简化描述。

### 2.3 Enable controls：MDSCR_EL1.KDE / MDE / OSLAR

DDI 0487 §D2.3 明确：

- **BRK/BKPT 类指令断点异常总是 enabled**。
- **硬件 breakpoint/watchpoint** 需要 `MDSCR_EL1.MDE` 加各自 `DBGBCR<n>.E / DBGWCR<n>.E`。
- **Software Step** 由 `MDSCR_EL1.SS` 驱动。
- OS Lock / Double Lock 会抑制相关调试能力。

QEMU 中：

- `MDSCR_EL1` 在 `debug_helper.c:193-199` 定义。
- 硬件断点/观察点全局门控在 `debug.c:363-365`、`387-389`：
  - `MDSCR_EL1.MDE == 0` 时直接忽略断点/观察点命中。
- 同 EL 调试使能由 `MDSCR_EL1.KDE` 与 `PSTATE.D` 控制（`debug.c:80-88`）。
- Software Step 激活条件见 `arm_singlestep_active()`（`debug.c:164-169`）。

### 2.4 OSLAR / OSLSR / Double Lock 的实现状态

DDI 0487 §D2.3 与相关寄存器定义要求：OS Lock / Double Lock 会影响 debug exception 生成。QEMU 的建模包括：

- `oslar_write()`：`debug_helper.c:123-139`
- `OSLAR_EL1` / `OSLSR_EL1`：`debug_helper.c:256-267`
- `OSDLR_EL1`：`debug_helper.c:268-274`
- `arm_generate_debug_exceptions()`：`debug.c:148-158`

但是这里存在一个**重要细节差异**：

- DDI 0487 的 `OSLSR_EL1` 定义中，**OSLK 是 bit[1]**。
- QEMU 的 `oslar_write()` 也确实把锁状态写到 `oslsr_el1` 的 **bit[1]**：
  `deposit32(env->cp15.oslsr_el1, 1, 1, oslock)`。
- 但 `arm_generate_debug_exceptions()` 检查的是：
  `if ((env->cp15.oslsr_el1 & 1) || (env->cp15.osdlr_el1 & 1)) return false;`
  这里测试了 **bit[0]**，而不是 bit[1]。

这意味着：

> **QEMU 当前代码对 OSLSR_EL1 的锁位判断与 DDI 0487 寄存器位定义不一致。**
>
> 因而，现有文档里“`OSLSR_EL1[0]=1` 则禁止调试异常”的说法并不符合规范；它只是复述了当前源码检查条件。

这是本次核查最值得保留的一个结论。

---

## 3. 断点验证

### 3.1 规范中的断点类型

DDI 0487 §D2.8.3 说明，breakpoint 可以是：

- 地址匹配 / 地址不匹配
- Context ID 匹配
- VMID 匹配
- Context ID + VMID 组合匹配
- 以及带链接（linked）的变体

这说明**规范上的 breakpoint 类型明显多于“普通地址断点”**。

### 3.2 QEMU 对 DBGBCR / DBGBVR 的建模

QEMU 在 `target/arm/cpu.h:529-538` 中保存：

- `dbgbvr[16]`
- `dbgbcr[16]`

在 `debug_helper.c:402-523` 的 `define_debug_regs()` 中按 CPU 能力动态注册 DBGBVR/DBGBCR；写入钩子位于：

- `dbgbvr_write()`：`debug_helper.c:371-381`
- `dbgbcr_write()`：`debug_helper.c:383-399`

写 DBGBCR 时，QEMU 还做了 BAS 镜像位归一化：

```c
value = deposit64(value, 6, 1, extract64(value, 5, 1));
value = deposit64(value, 8, 1, extract64(value, 7, 1));
```

这与 DDI 0487 对 AArch64 下 BAS 合法编程组合的限制是相容的，目的是把 guest 写入规整到可比较的形式。

### 3.3 QEMU 真正支持的 breakpoint 类型

核心函数：`hw_breakpoint_update()`（`target/arm/tcg/debug.c:652-736`）。

QEMU 对 BT 的处理可归纳为：

| BT | 规范语义 | QEMU 状态 |
|---|---|---|
| 0 | unlinked address match | **已实现** |
| 1 | linked address match | **已实现**（但链接条件还要在运行时由 `linked_bp_matches()` 通过） |
| 2 | unlinked context ID match | **未实现，LOG_UNIMP** |
| 3 | linked context ID match | **不作为独立触发源插入断点**，只可充当 link target |
| 4 / 5 | address mismatch | **未实现** |
| 8 | unlinked VMID match | **未实现，LOG_UNIMP** |
| 9 / 11 | linked VMID / context+VMID | **不支持** |
| 10 | unlinked context+VMID | **未实现，LOG_UNIMP** |
| 7 / 13 | linked `CONTEXTIDR_EL1/EL2` match | **仅在 `linked_bp_matches()` 中作为 link target 判定** |
| 15 | linked full context ID | **未支持** |

因此，QEMU 对 DDI 0487 §D2.8.3 的实现并不是“完整支持所有 breakpoint 类型”，而是：

- **完整支持地址匹配类**
- **部分支持 linked context-aware breakpoint 作为辅助条件**
- **不支持 unlinked context / VMID / full-context 触发**
- **不支持 address mismatch**

### 3.4 BAS 与地址匹配

DDI 0487 §D2.8.7 / §D2.8.9 要求 breakpoint 匹配不仅看地址，还看 BAS 与约束条件。QEMU 在 `hw_breakpoint_update()` 中做法是：

```c
int bas = extract64(bcr, 5, 4);
addr = bvr & ~3ULL;
if (bas == 0) {
    return;
}
if (bas == 0xc) {
    addr += 2;
}
```

可见 QEMU 将 A64 下常见 BAS 情况规约为四种：

- `0000`：禁用
- `0011`：命中 `addr`
- `1100`：命中 `addr + 2`
- `1111`：命中 `addr`

这与 DDI 0487 §D2.8.9 对 16-bit/32-bit 指令地址比较的解释一致，属于**架构允许的实现选择**。

### 3.5 linked breakpoint 的运行时判定

`linked_bp_matches()`（`debug.c:172-253`）是 QEMU 对 DDI 0487 §D2.8.3 “linking of breakpoints”的关键落点。

它验证：

- link 目标编号 `LBN` 是否有效
- 目标 breakpoint 是否启用
- link target 的 BT 是否属于 context-aware 类型
- `CONTEXTIDR_EL1` / `CONTEXTIDR_EL2` 是否与 `DBGBVR[lbn]` 匹配

验证结论：

- QEMU **支持“地址断点 / 观察点 + linked context breakpoint”** 这种组合。
- 但 QEMU **没有实现 VMID 链接、full context 链接、address mismatch 链接**。
- 因此它覆盖了 DDI 0487 §D2.8.3 的一个实用子集，而非全集。

---

## 4. 观察点验证

### 4.1 规范中的 watchpoint 模型

DDI 0487 §D2.9.1 定义，watchpoint 基于数据地址产生事件，不对 instruction fetch 生效。调试器通过：

- `DBGWVR<n>_EL1`：地址/值
- `DBGWCR<n>_EL1`：控制

来描述匹配范围、访问类型、字节选择等。

### 4.2 QEMU 对 DBGWCR / DBGWVR 的建模

QEMU 在 `cpu.h:531-532` 保存：

- `dbgwvr[16]`
- `dbgwcr[16]`

寄存器定义与 write hook：

- `dbgwvr_write()`：`debug_helper.c:334-357`
- `dbgwcr_write()`：`debug_helper.c:359-369`

`internals.h:111-121` 给出了 `DBGWCR` 的位域宏，包括：

- `E`
- `PAC`
- `LSC`
- `BAS`
- `HMC`
- `SSC`
- `LBN`
- `WT`
- `MASK`
- `SSCE`

这与 DDI 0487 §D2.9 对观察点控制字段的主体结构是一致的。

### 4.3 地址范围匹配：MASK 与 BAS

QEMU 在 `hw_watchpoint_update()`（`debug.c:546-633`）里严格区分两种模式：

1. **MASK 模式**：`MASK != 0`
   - 匹配 `2^MASK` 大小的对齐区域。
2. **BAS 模式**：`MASK == 0`
   - 由 BAS 选择字节范围。

对应实现：

```c
mask = FIELD_EX64(wcr, DBGWCR, MASK);
if (mask == 1 || mask == 2) {
    return;
} else if (mask) {
    len = 1ULL << mask;
    wvr &= ~(len - 1);
} else {
    int bas = FIELD_EX64(wcr, DBGWCR, BAS);
    ...
    basstart = ctz32(bas);
    len = cto32(bas >> basstart);
    wvr += basstart;
}
```

与 DDI 0487 §D2.9.1、§D2.8.7 对编程约束的对应关系：

- `MASK == 1 || 2`：规范保留值，QEMU选择“视为禁用”。
- BAS 非连续：规范允许 CONSTRAINED UNPREDICTABLE；QEMU 选择“只取第一段连续 1”。
- `MASK` 与 `BAS` 同时使用：规范允许实现自选；QEMU 选择“忽略 BAS，只按 MASK 区域触发”。

这类处理方式虽然不是唯一，但属于 **DDI 0487 明确允许的实现自由度**。

### 4.4 访问类型与异常信息

`DBGWCR.LSC` 在 QEMU 中映射到：

- `01` → `BP_MEM_READ`
- `10` → `BP_MEM_WRITE`
- `11` → `BP_MEM_ACCESS`

命中后 `arm_debug_excp_handler()`（`debug.c:464-508`）设置：

- `exception.fsr = arm_debug_exception_fsr(env)`
- `exception.vaddress = wp_hit->hitaddr`
- `syn_watchpoint(0, 0, wnr)`

这与 DDI 0487 §D2.9.8 “watchpoint 需要记录 syndrome 与 fault address”的大方向一致。

### 4.5 观察点与 EL / Security / link 条件

观察点真正是否构成架构 watchpoint 命中，不只由地址决定，还要通过 `bp_wp_matches()`（`debug.c:255-352`）检查：

- `SSC`：安全状态过滤
- `PAC/HMC`：EL 过滤
- `WT/LBN`：linked breakpoint 条件

这里与 DDI 0487 §D2.9、§D2.8.3 的对应关系是正确的：

> QEMU 先用通用 CPU watchpoint 机制筛出“地址/访问类型命中”，再用 ARM-specific 逻辑做架构级二次确认。

这是一个很合理的分层实现。

---

## 5. 单步执行验证

### 5.1 必须区分两类“单步”

现有文档最容易混淆的点就是这里。

1. **架构 Software Step**（DDI 0487 §D2.11）
   - 由 `MDSCR_EL1.SS`、`PSTATE.SS`、异常返回路径、debug enable 条件共同驱动。
   - 这是 guest 可见的 ARM 调试架构行为。
2. **QEMU gdbstub single-step**
   - 由 `cpu->singlestep_enabled`、`CF_SINGLE_STEP`、`EXCP_DEBUG` 驱动。
   - 这是模拟器为了让 GDB 执行 `'s'` 命令而提供的执行控制机制。
   - 它**不是** DDI 0487 §D2.11 的 Software Step 架构状态机。

### 5.2 架构 Software Step 的激活条件

DDI 0487 §D2.11.1、§D2.11.4 说明：

- 调试器设置 `MDSCR_EL1.SS = 1`
- 通过 exception return 把 `SPSR_ELx.SS` 复制进 `PSTATE.SS`
- 且只有在“返回前所在 EL debug disabled、返回目标 EL debug enabled”等条件成立时，才进入 **Active-not-pending**

QEMU 的三个关键落点：

- `arm_singlestep_active()`：`debug.c:164-169`
- `exception_return()`：`helper-a64.c:635-644`, `703-706`, `723-726`
- TB flag 构建：`hflags.c:77-88`, `677-688`

`arm_singlestep_active()` 的语义是：

```c
return extract32(env->cp15.mdscr_el1, 0, 1)
    && arm_el_is_aa64(env, arm_debug_target_el(env))
    && arm_generate_debug_exceptions(env);
```

这与 DDI 0487 §D2.11 的“Software Step 只在 AArch64 debug target 且 debug exceptions 可生成时才活跃”相符。

### 5.3 QEMU 对三态状态机的映射

DDI 0487 §D2.11.4 / §D2.11.5 / §D2.11.6 定义三态：

| 状态 | QEMU 表示 |
|---|---|
| Inactive | `SS_ACTIVE=0` |
| Active-not-pending | `SS_ACTIVE=1` 且 `PSTATE.SS=1` |
| Active-pending | `SS_ACTIVE=1` 且 `PSTATE.SS=0` |

QEMU 在 `hflags.c:678-688` 直接写出了这张状态表，说明实现者是按架构模型来落地的，而不是“拍脑袋的单步 TB 限制”。

### 5.4 状态推进与异常生成

- `gen_ss_advance()`（`translate.h:414-421`）
  - 执行完一步后，把 `PSTATE.SS` 清零：Active-not-pending → Active-pending。
- `gen_step_complete_exception()`（`translate-a64.c:511-525`）
  - 先推进状态，再生成 `syn_swstep()`。
- `gen_swstep_exception()`（`translate.h:423-429`）
  - 最终调用 `HELPER(exception_swstep)`。

这与 DDI 0487 §D2.11.5 / §D2.11.6 的主语义一致：

> 先完成“被单步的那条指令”，然后在下一条指令之前触发 Software Step exception。

### 5.5 TB 级实现策略

在 `translate.c` / `translate-a64.c` 中，只要 `ss_active` 为真，QEMU 就把 `max_insns` 限制为 1。这不是规范要求的外部表现，而是 QEMU 为实现 §D2.11 状态机所做的内部工程化手段。

这是一种正确实现方式，因为它保证：

- 单步期间最多只执行 1 条 guest 指令
- 在正确时机清理 `PSTATE.SS`
- 在 TB 尾部精确插入 Software Step exception

### 5.6 一个实现选择：异步异常优先级

`gen_step_complete_exception()` 的注释明确说：QEMU 在某些边界条件下选择让 swstep exception 优先于一部分异步异常，这是一个 **IMPDEF choice**。这与 DDI 0487 §D2.11.5 允许的实现自由度是兼容的，但需要在分析文档里点明，避免误以为“规范只有一种唯一时序”。

### 5.7 gdbstub 单步不是 §D2.11 Software Step

QEMU gdbstub 的单步路径在：

- `accel/tcg/cpu-exec-common.c:42-50`
- `accel/tcg/cpu-exec.c:300-309`
- `include/hw/core/cpu.h:1129-1135`

它的本质是：

- `cpu_single_step(cpu, enabled)` 设置 `singlestep_enabled`
- 让 TCG TB 只执行 1 条指令
- 执行结束后抛出 `EXCP_DEBUG`
- 返回给 GDB

这条路径：

- 不依赖 `MDSCR_EL1.SS`
- 不依赖 `PSTATE.SS`
- 不构造 guest 可见的 `EC_SOFTWARESTEP`
- 不属于 DDI 0487 §D2.11 定义的 self-hosted Software Step

因此，现有文档如果把二者并列但不明确区分，容易误导读者。

---

## 6. Debug 异常路由

### 6.1 路由控制位

DDI 0487 §D2.2、§D2.5 相关的关键控制位，在 QEMU 中分别体现为：

- `MDCR_EL2.TDE`：路由到 EL2（`internals.h:179` 定义，`debug.c:28-31` 使用）
- `HCR_EL2.TGE`：与 TDE 一起决定 EL2 路由（`debug.c:28-31`）
- `MDCR_EL3.SDD`：禁止 Secure state 下的 debug exceptions（`debug.c:73-77`）

### 6.2 raise_exception_debug 与 same-EL syndrome

`raise_exception_debug()`（`debug.c:47-61`）会根据 `debug_el == cur_el` 设置 syndrome 的 same-EL 变体位：

```c
syndrome |= (debug_el == cur_el) << R_SYNDROME_EC_SHIFT;
```

这与 DDI 0487 §D2.1.3 / §D2.5 对 EC 编码区分 same-EL 与 lower-to-higher EL 的要求一致。QEMU 后续使用：

- `syn_breakpoint()` → `EC_BREAKPOINT / EC_BREAKPOINT_SAME_EL`
- `syn_watchpoint()` → `EC_WATCHPOINT / EC_WATCHPOINT_SAME_EL`
- `syn_swstep()` → `EC_SOFTWARESTEP / EC_SOFTWARESTEP_SAME_EL`

### 6.3 BRK 的特殊路由

DDI 0487 §D2.3、§D2.5 指出 breakpoint instruction exceptions（BRK/BKPT）与普通硬件断点不同，它们始终 enabled。QEMU 的 `HELPER(exception_bkpt_insn)`（`debug.c:514-539`）也体现了这一点：

- 即使 `debug_el < cur_el`，也会 fallback 到当前 EL，而不是简单忽略。
- 不受 `MDSCR_EL1.MDE` 控制。

所以，**BRK ≠ hardware breakpoint**，两者异常模型不同，这一点在规范和 QEMU 中都很清楚。

### 6.4 调试寄存器访问 trap 与调试异常路由不是一回事

`debug_helper.c:21-120` 中有一组 `access_tda / access_tdosa / access_tdra / access_tdcc`，它们根据：

- `MDCR_EL2.TDA / TDOSA / TDRA / TDCC`
- `MDCR_EL3.TDA / TDOSA / TDCC`
- `HCR_EL2.TGE`
- `MDSCR_EL1.TDCC`

决定**访问调试寄存器时是 trap 到 EL1/EL2/EL3，还是直接访问成功**。

这部分语义属于“debug register access control/trap”，不是“breakpoint/watchpoint/software-step exception routing”。既有文档如果把这两者混为一个“调试异常路由”，概念上会发散。

---

## 7. 外部调试 (§H) 概要

### 7.1 规范要求：Debug state 与 Halting debug

DDI 0487 §H2.1、§H2.2、§H2.3、§H2.4 定义的 external debug 核心对象是：

- **Debug state**：PE 停止正常取指，不再按 PC 执行，而是由外部调试接口控制。
- **ITR / EDITR**：在 Debug state 中注入执行的指令。
- **DCC**：调试通信通道。
- **EDSCR.HDE**：Halting debug enable。
- **进入 Debug state / 退出 Debug state**：保存 `DLR` / `DSPSR`，再由外部调试器恢复执行。

DDI 0487 §H2.2.3 还明确指出：若 `EDSCR.HDE == 1` 且 halting allowed，则 breakpoint/watchpoint 可能导致 **进入 Debug state**，而不是生成 debug exception。

### 7.2 QEMU 的实际情况：几乎没有 external debug 建模

QEMU 对 H 章只实现了非常有限的占位或兼容行为：

1. **DCC 寄存器大多 RAZ/WI / 占位**
   - `MDCCSR_EL0`：`debug_helper.c:201-207`
   - `OSDTRRX_EL1` / `OSDTRTX_EL1` / `DBGDTR*_EL0`：`debug_helper.c:214-235`
   - 注释明确说 Debug Communications Channel **not implemented**。
2. **OSECCR / memory mapped external debug 组件未实现**
   - `OSECCR_EL1` 在 `debug_helper.c:237-245` 明确只是 dummy。
3. **HLT 不作为 external halting debug 指令实现**
   - `translate-a64.c:3213-3226`
   - `translate.c:1170-1177`
   - 注释直接说明：**Since QEMU doesn't implement external debug, we treat this as halting debug disabled: it will UNDEF.**

这意味着：

> QEMU 没有真正实现 DDI 0487 Chapter H 所定义的 “外部调试状态机”。

### 7.3 与 gdbstub 的关系

QEMU gdbstub 能“暂停 CPU、检查寄存器、单步、插断点”，但它做的事情是：

- 在**模拟器外层**控制执行循环
- 使用 `BP_GDB`、`cpu_single_step()`、`EXCP_DEBUG`
- 通过 GDB RSP 与宿主 GDB 通信

它**不是** ARM external debug 架构里的：

- `EDSCR`
- `Debug state`
- `DLR / DSPSR`
- `ITR / EDITR`
- `Halting Step`

因此，最准确的说法应是：

> **QEMU gdbstub 是 emulator-side debug frontend，不是 ARM Chapter H external debug 的架构实现。**

---

## 8. QEMU 实现验证

### 8.1 与规范一致的部分

1. **Breakpoint / Watchpoint 的全局门控**
   - `MDSCR_EL1.MDE` + 各自 `E` 位
   - 对应 DDI 0487 §D2.3
2. **Same-EL / higher-EL debug exception 路由与 EC 编码**
   - 对应 DDI 0487 §D2.2 / §D2.5
3. **Software Step 三态状态机**
   - `SS_ACTIVE` + `PSTATE.SS`
   - 对应 DDI 0487 §D2.11.4 / §D2.11.5 / §D2.11.6
4. **BAS / MASK / LSC / SSC / HMC / PAC 的匹配逻辑**
   - 对应 DDI 0487 §D2.8 / §D2.9
5. **BRK 与硬件断点分流**
   - 对应 DDI 0487 §D2.3 / §D2.5

### 8.2 部分实现的部分

1. **linked breakpoint**：只实现 context-aware 子集。
2. **context/VMID/full-context breakpoint**：仅部分 link target 语义，未实现完整异常生成。
3. **AArch32 / Monitor / EL3 相关 legacy 路由**：QEMU 做了兼容，但分析时必须把它和纯 AArch64 模型分开。

### 8.3 未实现或仅占位的部分

1. **Chapter H 外部调试状态机**
2. **EDSCR.HDE 驱动的 halting debug**
3. **ITR/DCC 型 Debug state 执行环境**
4. **Address mismatch breakpoint**
5. **完整 VMID / full-context breakpoint 支持**
6. **Vector catch 的真实行为**（`DBGVCR` 只是 dummy）

### 8.4 发现的规范对照问题

最明确的一项是：

- **OSLSR_EL1 锁位判断**：规范是 `OSLK=bit[1]`，但 `arm_generate_debug_exceptions()` 测的是 `bit[0]`。

如果后续要继续完善 QEMU ARM64 debug 架构文档，这个问题应被单列。

---

## 9. 既有文档勘误

### 9.1 关于 `target/arm/debug_helper.c` 的职责

**原文倾向**：把 `debug_helper.c` 视为“断点/观察点实现主文件”。

**更正**：

- `debug_helper.c` 主要负责 **系统寄存器定义、访问控制、write hook**。
- 自托管调试运行时核心逻辑已经在 `target/arm/tcg/debug.c`。

### 9.2 关于 `OSLSR_EL1[0]` 的表述

**原文问题**：把 `OSLSR_EL1[0]` 当作 OS Lock 状态位，并据此写出“`OSLSR_EL1[0]=1` 会禁止调试异常”。

**更正**：

- 按 DDI 0487，`OSLK` 是 **bit[1]**，不是 bit[0]。
- 现有文档是跟随当前 QEMU 代码逻辑写的，但从规范角度看不正确。

### 9.3 关于“单步执行”的混写

**原文问题**：`46` 文档中的“单步执行机制”主要描述的是 **gdbstub/TCG 单步**；`53` 文档中的“单步执行机制”描述的是 **ARM Software Step**。两者容易被读者当成同一件事。

**更正**：

- gdbstub 单步：模拟器控制执行循环。
- Software Step：DDI 0487 §D2.11 的 guest-visible debug exception 机制。
- 两者可以共存，但不可混为一层。

### 9.4 关于 Active-not-pending 的进入条件

`53` 文档中状态机图把 “`MDSCR.SS=1 + 调试使能 + ERET 或异常入口`” 归入进入 **Active-not-pending** 的条件，这不够精确。

**更正**：

- 依据 DDI 0487 §D2.11.4，进入 Active-not-pending 的关键事件是：
  **异常返回把 `SPSR_ELx.SS=1` 复制到 `PSTATE.SS`**。
- 异常入口本身更相关的是进入 Active-pending 或改变后续可否继续维持 step 状态，不应笼统归并成“异常入口即可进入 Active-not-pending”。

### 9.5 关于 breakpoint 类型支持范围

**原文问题**：既有文档虽然提到“地址匹配 / 上下文匹配 / VMID 匹配”，但容易给人“QEMU 都支持”的印象。

**更正**：

- QEMU 只完整支持地址匹配类。
- context/VMID/full-context 只实现了很小的 linked 子集。
- address mismatch 未实现。

### 9.6 关于外部调试与 gdbstub 的关系

**原文缺口**：对 external debug 的 H 章与 QEMU gdbstub 的关系没有明确切开。

**更正**：

- gdbstub ≠ Debug state。
- gdbstub ≠ EDSCR/HDE/ITR/DCC。
- `HLT` 在 QEMU 中不会像 Chapter H 那样真正把 PE 带入 Debug state；除 semihosting 例外外，通常按 UNDEF 处理。

---

## 10. 总结

基于 DDI 0487 M.b Chapter D2 / H 与 QEMU 当前源码，可以给出如下总评：

1. **QEMU 对自托管调试（Chapter D2）实现较扎实。**
   - `MDSCR_EL1.MDE/KDE/SS`
   - `DBGBCR/DBGBVR`
   - `DBGWCR/DBGWVR`
   - `Software Step`
   - `MDCR_EL2.TDE` / `MDCR_EL3.SDD` 路由
   都有明确代码落点。

2. **QEMU 对 Chapter H 外部调试没有做完整架构建模。**
   - 没有真正的 Debug state
   - 没有真正的 EDSCR.HDE 控制
   - 没有真正的 ITR/DCC 执行环境
   - gdbstub 只是模拟器侧调试前端，不是 Arm external debug 架构本身

3. **既有文档的主要问题不是“大方向错”，而是“层次混写与边界不清”。**
   - gdbstub 单步 vs Software Step
   - debug register access trap vs debug exception routing
   - breakpoint 类型“规范全集” vs “QEMU已实现子集”

4. **本次核查发现一个值得进一步跟踪的具体问题：**
   - `OSLSR_EL1` 的锁状态位，规范定义为 bit[1]，而当前 `arm_generate_debug_exceptions()` 检查的是 bit[0]。

如果后续继续扩展这组 ARM64 调试文档，建议把文档体系拆成三条独立主线：

- **Self-hosted debug 架构线（DDI D2）**
- **External debug / Debug state 架构线（DDI H）**
- **QEMU gdbstub / TCG / host-side debug 控制线**

这样最不容易把“guest 架构语义”和“模拟器实现技巧”混在一起。