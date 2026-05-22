# ARM64 Generic Timer 规范验证：DDI 0487 Chapter D12 与 QEMU 对照

> 目标：对照 ARM DDI 0487 M.b Chapter D12（Generic Timer），复核 QEMU 现有分析文档《12-Generic-Timer定时器深度分析.md》的主要结论。  
> 代码基线：QEMU 当前源码树。  
> 重点文件：`target/arm/helper.c`、`target/arm/cpu.c`、`target/arm/cpu.h`、`target/arm/gtimer.h`、`hw/arm/virt.c`、`include/hw/arm/bsa.h`。

---

## 1. 概述

ARM Generic Timer 在 DDI 0487 §D12 中由三层内容组成：

1. **系统计数器（System Counter）**：统一的系统时间源，规范位于 DDI 0487 §D12.1.2。  
2. **PE 视角的计数器与定时器**：物理计数器、虚拟计数器、各级定时器，规范位于 DDI 0487 §D12.2。  
3. **系统级中断/系统级 MMIO 组件**：规范要求系统级实现还要有 memory-mapped counter/timer 组件，见 DDI 0487 §D12.1.1。

QEMU 的实现是 **PE 内嵌式实现**，核心逻辑不在 `hw/timer/arm_generic_timer.c` 之类独立设备文件中，而是分散在：

- `target/arm/gtimer.h`：定时器枚举。
- `target/arm/cpu.h`：保存 `c14_cntfrq`、`cntvoff_el2`、`cntpoff_el2`、`c14_timer[]` 等状态。
- `target/arm/cpu.c`：分配 `QEMUTimer *gt_timer[NUM_GTIMERS]`。
- `target/arm/helper.c`：几乎全部体系寄存器语义、访问控制、偏移计算、到期重算逻辑。
- `hw/arm/virt.c`：把 CPU 的定时器 GPIO 输出连到 GIC PPI。

因此，QEMU 的 Generic Timer 更接近“**CPU sysreg 语义 + QEMUTimer 调度 + GIC PPI 路由**”的组合，而不是对 D12 系统级 MMIO 组件的完整建模。

### QEMU 当前支持的定时器集合

`target/arm/gtimer.h` 定义了 7 类 timer：

- `GTIMER_PHYS` → `CNTP_*`（EL1 physical）
- `GTIMER_VIRT` → `CNTV_*`（EL1 virtual）
- `GTIMER_HYP` → `CNTHP_*`（EL2 physical）
- `GTIMER_SEC` → `CNTPS_*`（EL3 physical / Secure timer）
- `GTIMER_HYPVIRT` → `CNTHV_*`（EL2 virtual，需 FEAT_VHE）
- `GTIMER_S_EL2_PHYS` → `CNTHPS_*`（Secure EL2 physical，需 FEAT_SEL2）
- `GTIMER_S_EL2_VIRT` → `CNTHVS_*`（Secure EL2 virtual，需 FEAT_SEL2）

这与 DDI 0487 §D12.2.4 的 timer 分类总体一致：EL1 物理/虚拟、EL2 物理、EL2 虚拟（VHE）、EL3 物理、Secure EL2 物理/虚拟（SEL2）。

---

## 2. 系统计数器验证（§D12.1.2）

## 2.1 规范要求

DDI 0487 §D12.1.2 对 system counter 的关键要求包括：

- **宽度**：Armv8.0–v8.5 至少 56 位；Armv8.6 起必须 64 位。
- **有效频率**：Armv8.6 起固定 1 GHz；更早版本通常为固定值，典型 1–50 MHz。
- **回绕时间**：不少于 40 年。
- **单调性**：系统中不同 agent 读取不应出现“时间倒退”。
- **CNTFRQ_EL0**：保存系统计数器的 effective frequency；复位后为 UNKNOWN，由最高实现异常级初始化。

## 2.2 QEMU 实现

QEMU 的计数器来自 `target/arm/helper.c:1339`：

```c
uint64_t gt_get_countervalue(CPUARMState *env)
{
    ARMCPU *cpu = env_archcpu(env);
    return qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) / gt_cntfrq_period_ns(cpu);
}
```

也就是说，QEMU 并没有建模一个独立的“系统级 always-on counter module”，而是直接把：

- `QEMU_CLOCK_VIRTUAL` 作为底层时间基准；
- 再按 `gt_cntfrq_period_ns(cpu)` 折算成 architected counter tick。

### 2.2.1 CNTFRQ_EL0

QEMU 在 `helper.c:2036-2040` 把 `CNTFRQ_EL0` 注册为系统寄存器，并通过 `arm_gt_cntfrq_reset()` 在复位时把 `env->cp15.c14_cntfrq` 设为 `cpu->gt_cntfrq_hz`。这与规范的关系如下：

- **符合点**：`gt_cntfrq_access()` 保证写 `CNTFRQ_EL0` 仅允许在最高实现异常级进行，符合 DDI 0487 §D12.1.2 对写权限的要求。
- **偏离点**：规范说 `CNTFRQ_EL0` **reset 后 UNKNOWN**；QEMU 直接给出确定复位值，不模拟 UNKNOWN。
- **重要简化**：QEMU 注释明确说明 `CNTFRQ_EL0` 是 **reads-as-written only**，写它 **不会改变实际计数频率**；真实频率来自 `cpu->gt_cntfrq_hz`。

因此：

- 从“软件可见寄存器语义”看，QEMU 基本符合。  
- 从“系统频率初始化流程”看，QEMU 做了工程化简化，不严格模拟 UNKNOWN reset 状态。

### 2.2.2 计数器宽度

QEMU 内部统一使用 `uint64_t` 保存/计算计数值，没有实现“v8.0–v8.5 仅至少 56 位”的窄宽度变体；换言之，QEMU 实际上总是提供 **64 位计数器模型**。

这对绝大多数客体软件是“向上兼容”的，但它不是对早期架构最小实现的精确仿真。

### 2.2.3 频率与回绕

`target/arm/cpu.c` 中：

- 默认频率：`GTIMER_DEFAULT_HZ = 1000000000`（1 GHz）
- 兼容频率：`GTIMER_BACKCOMPAT_HZ = 62500000`（62.5 MHz）

这意味着：

- **1 GHz** 下，64 位计数器回绕时间约为 `2^64 ns`，约 584 年；
- **62.5 MHz** 下，回绕时间更长，约 9350 年。

都远大于 DDI 0487 §D12.1.2 的“**不少于 40 年**”要求。

### 2.2.4 单调性与 always-on

规范要求 system counter 在 always-on power domain 中提供统一时间视图。QEMU 没有硬件电源域模型，但 `QEMU_CLOCK_VIRTUAL` 在 VM 运行语义下提供单调递增的虚拟时间，因此对客体可见行为接近“统一单调时间源”。

**结论**：

- `CNTFRQ_EL0` 的读写权限语义：**基本符合**。  
- reset UNKNOWN：**未严格实现**。  
- counter width：**QEMU 一律 64 位，宽于早期最小规范**。  
- rollover ≥ 40 years：**满足且远超**。  
- 系统计数器作为独立系统组件：**未单独建模，采用 CPU 内部合成计数器近似**。

---

## 3. 物理定时器验证（§D12.2 / §D12.2.4）

## 3.1 规范要求

DDI 0487 §D12.2.4 给出的 CompareValue 视图定义是：

- `TimerConditionMet = (((PhysicalCountInt() - Offset) - CompareValue) >= 0)`

其中对 **EL1 physical timer**：

- `Offset` 在满足 `SCR_EL3.ECVEn=1`、`CNTHCTL_EL2.ECV=1`、EL2 已实现且 `HCR_EL2.{E2H,TGE} != {1,1}` 等条件时取 `CNTPOFF_EL2`；
- 否则 `Offset=0`。

同时 DDI 0487 §D12.2.4.2 定义 TimerValue 视图：

- 读：`TimerValue = (CompareValue - (PhysicalCountInt - Offset))[31:0]`
- 写：`CompareValue = (PhysicalCountInt - Offset) + SignExtend(TimerValue[31:0])`

## 3.2 QEMU 的 CNTP_* 实现

QEMU 用 `GTIMER_PHYS` 承载 EL1 physical timer：

- `CNTP_CTL_EL0` → `cp15.c14_timer[GTIMER_PHYS].ctl`
- `CNTP_CVAL_EL0` → `cp15.c14_timer[GTIMER_PHYS].cval`
- `CNTP_TVAL_EL0` → 非 raw 存储，通过读写函数投影

核心比较逻辑在 `helper.c:1466-1525`：

```c
uint64_t offset = gt_indirect_access_timer_offset(&cpu->env, timeridx);
uint64_t count = gt_get_countervalue(&cpu->env);
int istatus = count - offset >= gt->cval;
```

这与 DDI 0487 §D12.2.4.1 的公式是一致的：QEMU 使用 **无符号 64 位比较** 来实现 `TimerConditionMet`。

## 3.3 CNTP_CTL_EL0

QEMU 仅允许软件写 `ctl` 的低两位：

- bit[0] = `ENABLE`
- bit[1] = `IMASK`
- bit[2] = `ISTATUS` 由 QEMU 根据当前比较结果计算

`gt_ctl_write()`：

- 若 `ENABLE` 改变，则重新计算定时器；
- 若只改 `IMASK`，则不重算 CompareValue，只根据 `ISTATUS` 更新 IRQ 线。

这符合体系结构的基本语义：`ISTATUS` 是派生状态，不是软件自由控制位。

## 3.4 CNTP_CVAL_EL0

`gt_cval_write()` 直接更新 `cval`，然后立即调用 `gt_recalc_timer()`。这与规范要求一致：CompareValue 一改，`TimerConditionMet` 的结果就可能变化。

## 3.5 CNTP_TVAL_EL0

QEMU 的 `do_tval_read()` / `do_tval_write()` 完全对应 DDI 0487 §D12.2.4.2：

```c
read : (uint32_t)(cval - (count - offset))
write: cval = count - offset + sextract64(value, 0, 32)
```

注意点：

- 写 TVAL 时使用 `sextract64(value, 0, 32)`，即 **32 位有符号扩展**，与规范一致。
- 读 TVAL 时返回 `uint32_t` 低 32 位，符合体系结构“32-bit downcounter 视图”。

## 3.6 物理定时器触发条件

QEMU 的触发链条是：

1. `ENABLE=1`
2. `count - offset >= cval` → `ISTATUS=1`
3. `IMASK=0` 时 `irqstate=(ctl & 6)==4`
4. `qemu_set_irq(cpu->gt_timer_outputs[GTIMER_PHYS], irqstate)`

因此，QEMU 的物理定时器条件可概括为：

- **体系结构条件**：比较式成立；
- **控制条件**：`ENABLE=1`；
- **中断输出条件**：`IMASK=0`；
- **QEMU 扩展条件**：在 RME Root/Realm 场景，还可能受 `CNTHCTL_EL2.CNTPMASK` 影响。

## 3.7 与规范的符合度

**符合**：

- CompareValue 视图公式。  
- TimerValue 视图公式。  
- `ENABLE/IMASK/ISTATUS` 的基本行为。  
- `CNTPOFF_EL2` 在 EL1 physical timer 中作为 offset 的用法。

**实现化简**：

- 不存在独立硬件比较器/系统级时钟分发，仅用 `QEMUTimer` 重编程来逼近“下一次状态变化时刻”。
- 读/写系统寄存器都是同步完成；规范里关于 speculative / self-synchronizing 的细粒度时序，QEMU 不做微架构级区分。

---

## 4. 虚拟定时器验证（§D12.2.2 / §D12.2.4）

## 4.1 规范要求

DDI 0487 §D12.2.2 明确指出：

- virtual counter = physical counter - `CNTVOFF_EL2`
- `CNTVCT_EL0` 保存当前 virtual counter 值
- EL1 virtual timer 的比较值使用 virtual count

DDI 0487 §D12.2.4.1 也明确指出：

- 对 **EL1 virtual timer**，`Offset` 就是 `CNTVOFF_EL2`
- 对 EL2/EL3/Secure EL2 等其它 timer，offset 为 0

## 4.2 QEMU 的虚拟计数器读取

QEMU 的 `gt_virt_cnt_read()`：

```c
uint64_t offset = gt_direct_access_timer_offset(env, GTIMER_VIRT);
return gt_get_countervalue(env) - offset;
```

而 `gt_direct_access_timer_offset()` 对 `GTIMER_VIRT` 的逻辑是：

- 普通 EL1/EL0 视角：返回 `cntvoff_el2`
- EL2 且 `HCR_EL2.E2H=1`：返回 0
- EL0 且 `HCR_EL2.{E2H,TGE}==11`（host userspace 视角）：返回 0

这对应 D12 对 host/guest 视角差异的实现化表达：**来宾看见的是 virtual time，host 视角下的某些 EL2/EL0 访问看见的是未减 offset 的物理时间**。

## 4.3 QEMU 的虚拟定时器比较语义

`gt_indirect_access_timer_offset()` 对 `GTIMER_VIRT` 永远返回 `env->cp15.cntvoff_el2`，所以 `gt_recalc_timer()` 中：

```c
istatus = count - cntvoff_el2 >= cval;
```

这正是 DDI 0487 §D12.2.4.1 对 EL1 virtual timer 的定义。

## 4.4 CNTV_TVAL_EL0 / CNTVOFF_EL2

QEMU 对虚拟定时器 TVAL 仍然使用通用公式：

- 读：`cval - (count - offset)`
- 写：`cval = count - offset + SignExtend(TVAL)`

而 `gt_cntvoff_write()` 在写入 `CNTVOFF_EL2` 后会立即：

```c
gt_recalc_timer(cpu, GTIMER_VIRT);
```

这很关键，因为一旦 offset 变化：

- `CNTVCT_EL0` 的读值变化；
- `CNTV_TVAL_EL0` 的投影变化；
- `count - offset >= cval` 的真假也可能变化。

所以 QEMU 在这一点上与 D12 语义是严格一致的。

## 4.5 QEMU 对 EL02 视图的处理

`helper.c` 对 `CNTV_TVAL_EL02` 做了专门注释：

- 与底层 `CNTV_TVAL_EL0` 不同，`CNTV_TVAL_EL02` **总是**应用 `CNTVOFF_EL2`。

这说明 QEMU 并不是“简单复用同一读写函数”，而是刻意对 EL02 视图做了与架构伪代码一致的特殊化处理。

## 4.6 与规范的符合度

**符合**：

- virtual counter = physical counter - `CNTVOFF_EL2`。  
- EL1 virtual timer 以 `CNTVOFF_EL2` 作为比较 offset。  
- `CNTVOFF_EL2` 变化后重算 timer。  
- TVAL 的有符号 32 位语义。

**化简**：

- `CNTVCTSS_EL0` / `CNTVCT_EL0` 在 QEMU 中使用同一读函数处理；QEMU 注释明确说明“所有 sysreg 都可视为 self-synchronizing”，因此不模拟 spec 的 speculative 差异。

---

## 5. EL2 定时器验证（CNTHP / CNTHV）

## 5.1 规范要求

根据 DDI 0487 §D12.2.4：

- EL2 physical timer：`CNTHP_{CVAL,TVAL,CTL}_EL2`
- EL2 virtual timer：`CNTHV_{CVAL,TVAL,CTL}_EL2`，仅在 **FEAT_VHE** 实现时存在
- 这两个 timer 的 offset 都是 **0**

## 5.2 QEMU 的 CNTHP_*

QEMU 在 `helper.c:4210-4230` 注册：

- `CNTHP_CVAL_EL2`
- `CNTHP_TVAL_EL2`
- `CNTHP_CTL_EL2`

并统一映射到 `GTIMER_HYP`。其读写函数最终都走通用逻辑：

- `gt_hyp_cval_write()` → `gt_cval_write(..., GTIMER_HYP, ...)`
- `gt_hyp_tval_read()/write()` → `gt_tval_read()/write(..., GTIMER_HYP, ...)`
- `gt_hyp_ctl_write()` → `gt_ctl_write(..., GTIMER_HYP, ...)`

由于 `gt_indirect_access_timer_offset()` / `gt_direct_access_timer_offset()` 对 `GTIMER_HYP` 都返回 0，因此 QEMU 对 `CNTHP_*` 的语义与 DDI 0487 §D12.2.4.1 一致。

## 5.3 QEMU 的 CNTHV_*

QEMU 在 `helper.c:5855-5871` 的 `vhe_reginfo[]` 中注册：

- `CNTHV_CVAL_EL2`
- `CNTHV_TVAL_EL2`
- `CNTHV_CTL_EL2`

并映射到 `GTIMER_HYPVIRT`。这说明：

- **CNTHV 不是无条件存在**；
- QEMU 与 DDI 0487 §D12.2.4 的要求一致，只在 VHE 语义路径中暴露它。

同样，`GTIMER_HYPVIRT` 的 offset 也是 0，符合规范。

## 5.4 EL1 对物理/虚拟 timer 的 trap 控制

QEMU 在 `gt_timer_access()` / `gt_counter_access()` 中落实 `CNTHCTL_EL2` 控制：

- `EL1PCTEN`：控制 EL1 对 physical counter 的访问
- `EL1PCEN`（no E2H）/ `EL1PTEN`（E2H）：控制 EL1 对 physical timer 的访问
- `EL1TVCT`：控制 EL1 对 virtual counter 访问是否 trap
- `EL1TVT`：控制 EL1 对 virtual timer 访问是否 trap

这与 D12 对 EL2 trap/host 托管场景的描述是匹配的。QEMU 还进一步处理了：

- EL0 host（`E2H=1,TGE=1`）时检查 `CNTHCTL_EL2.EL0[PV]CTEN/EL0[PV]TEN`
- VHE/EL02/NV2 视图下的重定向寄存器路径

因此 QEMU 在 EL2 控制面上不只是“有寄存器”，而是实现了比较完整的访问控制矩阵。

## 5.5 与规范的符合度

**符合**：

- `CNTHP_*` 一直存在。  
- `CNTHV_*` 仅在 FEAT_VHE 下存在。  
- EL2 timers 的 offset 为 0。  
- `CNTHCTL_EL2` 对 EL1/EL0 访问控制基本齐全。

**补充说明**：

- QEMU 对 VHE/NV2/EL02 视图的处理比 D12 主干正文更“实现细节化”，属于架构特性扩展层面的额外落地。

---

## 6. Secure 定时器验证（CNTPS / Secure EL1）

## 6.1 规范要求

DDI 0487 §D12.2.4 对 secure physical timer 的描述是：

- EL3 physical timer 由 `CNTPS_{CVAL,TVAL,CTL}_EL1` 表示；
- 它从 EL3 总可访问；
- 从 Secure EL1 是否可访问，由 `SCR_EL3.{ST,EEL2}` 决定。

这里必须注意：**规范文字不是只提 `SCR_EL3.ST`，而是明确写了 `{ST,EEL2}`。**

## 6.2 QEMU 的 CNTPS_*

QEMU 在 `helper.c:2200-2222` 注册：

- `CNTPS_TVAL_EL1`
- `CNTPS_CTL_EL1`
- `CNTPS_CVAL_EL1`

对应 `GTIMER_SEC`。

访问控制在 `gt_stimer_access()`：

- `EL3`：允许
- `EL2` / `EL0`：UNDEFINED
- `EL1`：
  - 非 Secure → UNDEFINED
  - 如果 `arm_is_el2_enabled(env)` → UNDEFINED
  - 若 `SCR_EL3.ST == 0` → trap 到 EL3
  - 否则允许

## 6.3 规范对照

QEMU 至少实现了以下正确点：

- `CNTPS_*` 只面向 Secure 世界；
- EL3 始终可访问；
- Secure EL1 的访问受 `SCR_EL3.ST` 影响；
- `CNTPS_*` 用独立 timer 实例，不复用 `CNTP_*`。

但 QEMU 这里也有一个值得明确写出的点：

- 在 `gt_stimer_access()` 中，看见的是 `arm_is_el2_enabled(env)` 这一**粗粒度判定**；
- 并没有像规范描述那样，在这个函数里直接按 `SCR_EL3.{ST,EEL2}` 两位做完整展开。

因此更准确的说法是：

- **QEMU 大体体现了 Secure EL1 对 `CNTPS_*` 的受控访问语义；**
- **但代码层面对 `{ST,EEL2}` 的建模不是完全按规范原文逐位展开。**

## 6.4 与规范的符合度

- `CNTPS_CTL_EL1`/`CNTPS_CVAL_EL1`/`CNTPS_TVAL_EL1` 的存在与基本行为：**符合**。  
- Secure EL1 访问依赖安全态和 `SCR_EL3.ST`：**符合**。  
- 对 `SCR_EL3.EEL2` 的显式处理：**QEMU 在此处不是规范原文级别的精确表达，属于实现化简/间接建模**。

---

## 7. Timer 中断路由（PPI 映射与 CNTHCTL_EL2 控制）

## 7.1 规范要求

DDI 0487 §D12.2.4 只要求：

- 每个已实现 timer 对系统输出一个信号；
- 若连接 GIC，则向 GIC 送一个 PPI；
- 在多 PE 系统中，同类 timer 在各 PE 必须使用相同中断号。

**规范并没有在 D12 中硬编码 PPI 号。**

## 7.2 QEMU 的 virt 板映射

`hw/arm/virt.c` 为 `virt` 机型建立如下映射：

- `GTIMER_PHYS` → `ARCH_TIMER_NS_EL1_IRQ` = PPI 30
- `GTIMER_VIRT` → `ARCH_TIMER_VIRT_IRQ` = PPI 27
- `GTIMER_HYP` → `ARCH_TIMER_NS_EL2_IRQ` = PPI 26
- `GTIMER_SEC` → `ARCH_TIMER_S_EL1_IRQ` = PPI 29
- `GTIMER_HYPVIRT` → `ARCH_TIMER_NS_EL2_VIRT_IRQ` = PPI 28
- `GTIMER_S_EL2_PHYS` → `ARCH_TIMER_S_EL2_IRQ` = PPI 20
- `GTIMER_S_EL2_VIRT` → `ARCH_TIMER_S_EL2_VIRT_IRQ` = PPI 19

这些值来自 `include/hw/arm/bsa.h`，是 **QEMU virt / SBSA 风格机器** 采用的 architectural INTID 约定。

## 7.3 这是否是 D12 级别“架构固定”？

不是。

更准确的表述应是：

- **在 QEMU `virt` / `sbsa-ref` 等板级实现里，定时器 PPI 号采用 30/27/26/29/28/20/19。**
- **D12 本身只要求“每个 PE 同类 timer 的中断号一致”，不在此节规定具体数字。**

因此，如果把“PPI 30/27/26/29 是架构规范直接硬编码”作为结论，那是不精确的；若限定为“QEMU virt 板的连线方式”，则是正确的。

## 7.4 CNTHCTL_EL2 对中断/访问的控制

QEMU 在 `gt_cnthctl_write()` 中处理：

- `EL0PCTEN/EL0VCTEN`
- `EL0PTEN/EL0VTEN`
- `EL1PCTEN/EL1PTEN`
- `EL1TVT/EL1TVCT`（FEAT_ECV traps）
- `ECV`
- `CNTVMASK/CNTPMASK`（RME）

其中 `CNTVMASK/CNTPMASK` 的实现点非常具体：

- `gt_update_irq()` 在 Root/Realm 安全空间下，会让这两个位覆盖 `IMASK` 的效果，直接压低 IRQ 输出。

这不是 D12.2.4 的基本 timer control 位，而是 EL2/RME 扩展控制面的实现；QEMU 对这部分做了合理扩展。

---

## 8. QEMU 实现验证

## 8.1 定时器对象模型

QEMU 在 `cpu.c` 中为每个 CPU 创建 `gt_timer[NUM_GTIMERS]`：

- 全部基于 `QEMU_CLOCK_VIRTUAL`
- `scale = gt_cntfrq_period_ns(cpu)`
- 每个 architected timer 各有一个 callback

即：**一个 architected timer = 一个 `ARMGenericTimer` 状态 + 一个 `QEMUTimer` 调度器 + 一根 GPIO IRQ 输出线**。

这与真实硬件“比较器 + 中断线”非常接近，但本质上仍是软件调度模拟。

## 8.2 `gt_recalc_timer()` 的建模质量

这是 QEMU Generic Timer 最关键的正确性函数：

- 根据当前 `count`、`offset`、`cval` 计算 `ISTATUS`
- 预测下一次状态翻转点 `nexttick`
- 把 `QEMUTimer` 编程到该时间点
- 最后同步 IRQ 输出

优点：

- 严格使用无符号 64 位算术，正确处理 wraparound 语义。
- `uadd64_overflow()` 用于处理 `cval + offset` 溢出。 
- 当 `nexttick` 超出 `QEMUTimer` 的 `int64` 表达范围时，退化为 `INT64_MAX`，等下次 callback 再继续推进，避免“超远未来时间点”丢失。

这说明 QEMU 不只是“到点拉高中断”，而是较认真地实现了 D12 的 wraparound 行为。

## 8.3 `CNT*SS_EL0` 的简化

QEMU 在 `helper.c:2225` 附近明确写道：

- `CNTPCTSS_EL0` / `CNTVCTSS_EL0` 与普通 `CNTPCT_EL0` / `CNTVCT_EL0` 共用同一读逻辑；
- 理由是“对 QEMU 而言，所有 sysreg 都可看作 self-synchronizing”。

因此：

- **从功能值上看**，QEMU 支持这些寄存器。  
- **从 speculative / ordering 微架构语义上看**，QEMU 没有区分 normal view 与 self-synchronized view。

这属于典型的软件模拟器合理化简。

## 8.4 系统级 MMIO 组件缺失

DDI 0487 §D12.1.1 要求 Generic Timer 的 system-level implementation 还包含：

- memory-mapped counter module
- memory-mapped timer control module
- 可选 memory-mapped timers

本次核对的 QEMU 实现重点是 **CPU system register 接口**。从所核查文件看，并没有在这套 ARM CPU Generic Timer 路径里实现一个与 D12.1.1 等价的独立系统级 counter module。

所以应明确区分：

- **QEMU 已较完整实现 PE sysreg 视角的 Generic Timer；**
- **并未在该实现层面完整覆盖 D12.1.1 的系统级 MMIO 组件架构。**

## 8.5 板级差异

`virt.c`、`sbsa-ref.c`、`allwinner-*`、`bcm283*` 等板子都能把 `gt_timer_outputs[]` 连到不同的 IRQ 号。说明：

- timer 核心语义在 CPU；
- IRQ 号和板级 wiring 在 machine/SoC；
- 文档分析时不应把某个板子的 IRQ 号误写成 D12 的通用规范。

---

## 9. 既有文档勘误（对《12-Generic-Timer定时器深度分析.md》的复核）

下表中的“✓/✗”表示该说法是否可以直接保留；若需要加限定条件，也记为 ✗ 并给出修正建议。

| 既有文档中的说法 | 结论 | 复核结论 |
|---|---:|---|
| “QEMU 中 Generic Timer 不是独立 QOM 设备，而是嵌入 ARM CPU 对象中实现” | ✓ | 与源码一致，核心逻辑在 `target/arm/helper.c` / `cpu.c`。 |
| “7 种定时器类型，每种对应一个独立的 QEMUTimer 和 GPIO 输出” | ✓ | 对 system emulation 下的当前 QEMU ARM CPU 实现成立。 |
| “`CNTFRQ_EL0` 对软件是 reads-as-written；写入不会改变 QEMU 真实计时频率” | ✓ | 与 `generic_timer_cp_reginfo[]` 注释和实现一致。 |
| “`CNTPCT = PhysCount - CNTPOFF`（ECV 条件满足时）” | ✗ | 需加限定：该公式对 **EL0/EL1 直接读 CNTPCT** 成立；**EL2/EL3 直接读** 在 QEMU 中 offset 为 0。 |
| “`CNTVCT = PhysCount - CNTVOFF`” | ✗ | 也需加限定：对 guest EL1/EL0 成立；在 `E2H=1` 的 EL2 或 `E2H,TGE=11` 的 host EL0 视角，QEMU 直接返回未减 offset 的值。 |
| “EL3 安全定时器由 `SCR_EL3.ST` 控制 Secure EL1 访问” | ✗ | 规范原文是 `SCR_EL3.{ST,EEL2}`；QEMU 代码里也不是单纯只看 `ST`。旧文表述过于简化。 |
| “Secure EL2 定时器：EL3 始终允许” | ✗ | QEMU `gt_sel2timer_access()` 明确要求 `SCR_EEL2=1` 时 EL3 才可访问，否则 UNDEFINED。 |
| “PPI 30/27/26/29/28/20/19 是 Generic Timer 的固定架构中断号” | ✗ | 这些是 QEMU `virt`/BSA 风格板的映射，不是 D12 在该节直接规定的固定数字。 |
| “写 `CNTVOFF_EL2` 后必须重算虚拟定时器” | ✓ | `gt_cntvoff_write()` 明确调用 `gt_recalc_timer(cpu, GTIMER_VIRT)`。 |
| “QEMU 实现了两套 offset：间接访问 offset 与直接访问 offset” | ✓ | 与 `gt_indirect_access_timer_offset()` / `gt_direct_access_timer_offset()` 一致。 |
| “CNTHV_* 仅在 FEAT_VHE 场景下存在” | ✓ | 与 `vhe_reginfo[]` 中的实现一致。 |

### 建议如何修正文档 12

如果后续要修《12-Generic-Timer定时器深度分析.md》，建议至少改三处：

1. 把 `CNTPOFF` / `CNTVOFF` 的公式都改成“**带当前 EL / host-guest 视角限定**”的写法。  
2. 把 Secure timer 一节改成“**按 DDI 0487 §D12.2.4 的 `{ST,EEL2}` 口径描述，再补充 QEMU 的具体实现逻辑**”。  
3. 把 PPI 号改成“**virt 板/QEMU BSA 风格连线**”，不要写成 D12 本身规定的架构常量。

---

## 10. 总结

本次对照可以得出以下结论：

### 10.1 QEMU 与 DDI 0487 Chapter D12 的主要一致点

- `CNTP_*` / `CNTV_*` / `CNTHP_*` / `CNTHV_*` / `CNTPS_*` / `CNTHPS_*` / `CNTHVS_*` 的寄存器集合基本齐全。  
- `CompareValue` 与 `TimerValue` 的核心公式实现正确。  
- `CNTVOFF_EL2`、`CNTPOFF_EL2` 的 offset 语义总体符合 D12。  
- `CNTHCTL_EL2` 对 EL0/EL1 访问陷入的控制实现较完整。  
- `CNTHV_*` 与 Secure EL2 timer 的特性门控（VHE / SEL2）方向正确。  
- `gt_recalc_timer()` 对 wraparound 与下一次翻转点的处理质量较高。

### 10.2 QEMU 相对规范的主要简化点

- 未严格模拟 `CNTFRQ_EL0` reset UNKNOWN。  
- 统一提供 64 位计数器，不区分早期架构“至少 56 位”的最小实现。  
- `CNTPCTSS_EL0` / `CNTVCTSS_EL0` 不区分 speculative/self-synchronized 微架构语义。  
- 没有在这一实现层面单独建模 D12.1.1 规定的 system-level MMIO counter/timer module。  
- Secure timer 某些访问条件在代码中做了工程化简化，不是对规范描述逐位逐分支照抄。

### 10.3 对既有文档的总体评价

《12-Generic-Timer定时器深度分析.md》作为 **QEMU 实现导向** 的分析文档，主体结论大多成立，尤其在：

- `gt_recalc_timer()`、
- `gt_update_irq()`、
- `CNTVOFF/CNTPOFF`、
- `QEMUTimer + GPIO + GIC` 这几块分析比较扎实。

但若把它提升为“**规范对照文档**”，则至少还需要补上三类限定：

1. **当前 EL / host-guest 视角限定**；  
2. **Secure/SEL2/VHE 特性限定**；  
3. **板级实现 vs 架构规范** 的边界说明。

一句话概括：**QEMU 的 Generic Timer PE 视角实现，与 DDI 0487 §D12.2 总体高度一致；差异主要集中在 system-level 组件未单独建模、若干 reset/ordering/secure 细节做了工程化简化。**
