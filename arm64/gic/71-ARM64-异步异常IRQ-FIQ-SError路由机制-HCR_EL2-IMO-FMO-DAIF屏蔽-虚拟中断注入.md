# ARM64 异步异常（IRQ/FIQ/SError）路由与处理机制分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/71 |
| 分析对象 | IRQ/FIQ/vIRQ/vFIQ/SError/NMI 的路由和响应 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §G1.13 (Physical interrupt routing) |
| 关键控制位 | HCR_EL2.IMO/FMO/AMO, SCR_EL3.IRQ/FIQ/EA, PSTATE.DAIF |
| 核心结论 | **异步异常路由完整实现，使用查找表精确匹配 ARM ARM Table G1-15** |

---

## 1. 概述

异步异常（中断）与同步异常（trap）的关键区别：

| 方面 | 同步异常 (SMC/WFI trap) | 异步异常 (IRQ/FIQ) |
|------|------------------------|-------------------|
| 触发时机 | 指令执行时 | 指令间隙检查 |
| PC 精确性 | 精确（当前指令） | 不精确（下一条指令开始前） |
| 可屏蔽性 | 不可屏蔽 | PSTATE.DAIF 可屏蔽 |
| 目标 EL | 由指令类型/trap 位决定 | 由 SCR/HCR 路由表决定 |
| 检查时机 | 翻译时 + runtime helper | cpu_exec 主循环 |

---

## 2. 中断检查入口：arm_cpu_exec_interrupt()

### 2.1 调用位置

QEMU 主循环在每个 TB 执行前检查中断：

```
cpu_exec()
  → cpu_handle_interrupt()
    → cc->tcg_ops->cpu_exec_interrupt(cs, interrupt_request)
      = arm_cpu_exec_interrupt(cs, interrupt_request)
```

### 2.2 优先级顺序

```c
// target/arm/cpu-irq.c:171
bool arm_cpu_exec_interrupt(CPUState *cs, int interrupt_request)
{
    // 优先级从高到低（IMPLEMENTATION DEFINED，QEMU 选择）:
    // 1. NMI (物理非屏蔽中断)
    // 2. VINMI (虚拟 NMI IRQ)
    // 3. VFNMI (虚拟 NMI FIQ)
    // 4. FIQ (物理快速中断)
    // 5. IRQ (物理中断)
    // 6. VIRQ (虚拟 IRQ)
    // 7. VFIQ (虚拟 FIQ)
    // 8. VSERR (虚拟 SError)
}
```

### 2.3 每个中断的判定逻辑

对每个中断执行两步判断：

```c
// 第一步：确定目标 EL
target_el = arm_phys_excp_target_el(cs, excp_idx, cur_el, secure);

// 第二步：检查是否可以响应（未被屏蔽）
if (arm_excp_unmasked(cs, excp_idx, target_el, cur_el, secure, hcr_el2)) {
    goto found;  // 可以响应
}
```

### 2.4 虚拟中断的特殊处理

虚拟中断（VIRQ/VFIQ/VSERR）的目标 EL 固定为 1（Guest EL1）：

```c
if (interrupt_request & CPU_INTERRUPT_VIRQ) {
    excp_idx = EXCP_VIRQ;
    target_el = 1;  // 虚拟中断总是到 EL1
    if (arm_excp_unmasked(...)) goto found;
}
```

VSERR 有额外动作：清除 HCR_EL2.VSE 位
```c
if (interrupt_request & CPU_INTERRUPT_VSERR) {
    // Taking a virtual abort clears HCR_EL2.VSE
    env->cp15.hcr_el2 &= ~HCR_VSE;
    cpu_reset_interrupt(cs, CPU_INTERRUPT_VSERR);
}
```

---

## 3. 物理中断目标 EL 路由：arm_phys_excp_target_el()

### 3.1 ARM ARM Table G1-15 查找表

QEMU 使用 6 维查找表精确实现 ARM DDI 0487 §G1.13.4：

```c
// target/arm/helper.c:8347
static const int8_t target_el_table[2][2][2][2][2][4] = {
    //                          [is64][scr][rw][hcr][secure][cur_el]
};
```

维度含义：

| 维度 | 取值 | 含义 |
|------|------|------|
| `is64` | 0/1 | EL3 是 AArch32/AArch64 |
| `scr` | 0/1 | SCR_EL3.{IRQ,FIQ,EA} 路由到 EL3 |
| `rw` | 0/1 | SCR_EL3.RW (低 EL 执行状态) |
| `hcr` | 0/1 | HCR_EL2.{IMO,FMO,AMO} 路由到 EL2 |
| `secure` | 0/1 | Non-secure / Secure 状态 |
| `cur_el` | 0-3 | 当前异常级别 |

### 3.2 路由控制位映射

```c
switch (excp_idx) {
case EXCP_IRQ:
case EXCP_NMI:
    scr = (env->cp15.scr_el3 & SCR_IRQ);   // SCR_EL3.IRQ
    hcr = hcr_el2 & HCR_IMO;               // HCR_EL2.IMO
    break;
case EXCP_FIQ:
    scr = (env->cp15.scr_el3 & SCR_FIQ);   // SCR_EL3.FIQ
    hcr = hcr_el2 & HCR_FMO;               // HCR_EL2.FMO
    break;
default:  // SError
    scr = (env->cp15.scr_el3 & SCR_EA);    // SCR_EL3.EA
    hcr = hcr_el2 & HCR_AMO;               // HCR_EL2.AMO
    break;
}

// HCR_EL2.TGE 也强制路由到 EL2
hcr |= (hcr_el2 & HCR_TGE) != 0;

target_el = target_el_table[is64][scr][rw][hcr][secure][cur_el];
```

### 3.3 典型路由场景

| 场景 | SCR.IRQ | HCR.IMO | cur_el | → target_el |
|------|:-------:|:-------:|:------:|:-----------:|
| 普通 EL0 App | 0 | 0 | 0 | 1 |
| KVM Guest (EL0) | 0 | 1 | 0 | 2 |
| KVM Guest (EL1) | 0 | 1 | 1 | 2 |
| Secure Monitor | 1 | x | 0 | 3 |
| EL2 Hypervisor | 0 | 1 | 2 | 2（HCR不影响自身） |

---

## 4. 中断屏蔽检查：arm_excp_unmasked()

### 4.1 核心逻辑

```c
// target/arm/cpu-irq.c:15
static inline bool arm_excp_unmasked(CPUState *cs, unsigned int excp_idx,
                                     unsigned int target_el,
                                     unsigned int cur_el, bool secure,
                                     uint64_t hcr_el2)
{
    // 规则1：不能向低 EL 发送异常
    if (cur_el > target_el) return false;

    // 规则2：FEAT_NMI ALLINT 屏蔽
    if (SCTLR.NMI && cur_el == target_el) {
        allIntMask = PSTATE.ALLINT || (SCTLR.SPINTMASK && PSTATE.SP);
    }

    // 规则3：根据异常类型检查 PSTATE mask bit
    switch (excp_idx) {
    case EXCP_NMI:   pstate_unmasked = !allIntMask; break;
    case EXCP_FIQ:   pstate_unmasked = !DAIF.F && !allIntMask; break;
    case EXCP_IRQ:   pstate_unmasked = !DAIF.I && !allIntMask; break;
    case EXCP_VFIQ:  return HCR.FMO && !TGE && !DAIF.F && !allIntMask;
    case EXCP_VIRQ:  return HCR.IMO && !TGE && !DAIF.I && !allIntMask;
    case EXCP_VSERR: return HCR.AMO && !TGE && !DAIF.A;
    }

    // 规则4：target_el > cur_el 时的特殊屏蔽覆盖
    if (target_el > cur_el && target_el != 1) {
        // AArch64:
        //   target=EL2: 除非 E2H+TGE，否则不可屏蔽
        //   target=EL3: 永远不可屏蔽
        // AArch32: 更复杂的 HCR/SCR 交互
    }

    return unmasked || pstate_unmasked;
}
```

### 4.2 虚拟中断的额外条件

虚拟中断只在 "hypervized" 状态下生效：

```c
case EXCP_VIRQ:
    if (!(hcr_el2 & HCR_IMO) || (hcr_el2 & HCR_TGE)) {
        return false;  // 没有虚拟化或 TGE 模式 → 忽略
    }
    return !(env->daif & PSTATE_I) && (!allIntMask);
```

含义：
- `HCR_EL2.IMO = 1`：物理 IRQ 路由到 EL2，Guest 看到虚拟 IRQ
- `HCR_EL2.TGE = 1`：Guest 不存在，虚拟中断无效

### 4.3 不可屏蔽原则

ARM 规范规定：**发往更高 EL 的中断不能被低 EL 屏蔽**

```c
if ((target_el > cur_el) && (target_el != 1)) {
    switch (target_el) {
    case 2:
        // EL0/EL1 → EL2: 不可屏蔽（除非 E2H+TGE）
        if ((hcr_el2 & (HCR_E2H | HCR_TGE)) != (HCR_E2H | HCR_TGE)) {
            unmasked = true;
        }
        break;
    case 3:
        // → EL3: 永远不可屏蔽
        unmasked = true;
        break;
    }
}
```

---

## 5. 虚拟中断注入机制

### 5.1 虚拟中断源

```
物理中断线                     HCR_EL2 位
GIC → cpu_interrupt(VIRQ)  OR  HCR_EL2.VI=1  →  VIRQ pending
GIC → cpu_interrupt(VFIQ)  OR  HCR_EL2.VF=1  →  VFIQ pending
                               HCR_EL2.VSE=1  →  VSERR pending
```

### 5.2 arm_cpu_update_virq()

```c
// cpu-irq.c:277
void arm_cpu_update_virq(ARMCPU *cpu)
{
    bool new_state = ((arm_hcr_el2_eff(env) & HCR_VI) &&
                      !(arm_hcrx_el2_eff(env) & HCRX_VINMI)) ||
                     (env->irq_line_state & CPU_INTERRUPT_VIRQ);

    if (new_state != cpu_test_interrupt(cs, CPU_INTERRUPT_VIRQ)) {
        if (new_state) cpu_interrupt(cs, CPU_INTERRUPT_VIRQ);
        else           cpu_reset_interrupt(cs, CPU_INTERRUPT_VIRQ);
    }
}
```

逻辑 OR：
- `HCR_EL2.VI` = 1 → 软件注入虚拟 IRQ（不经过 GIC）
- `irq_line_state & CPU_INTERRUPT_VIRQ` → GIC 的 virtual interface 产生

### 5.3 NMI 虚拟中断 (FEAT_NMI)

```c
// arm_cpu_update_vinmi()
bool new_state = ((arm_hcr_el2_eff(env) & HCR_VI) &&
                  (arm_hcrx_el2_eff(env) & HCRX_VINMI)) ||
                 (env->irq_line_state & CPU_INTERRUPT_VINMI);
```

当 `HCRX_EL2.VINMI=1` 时，`HCR_EL2.VI` 产生的是 VINMI（NMI 优先级）而非普通 VIRQ。

---

## 6. 异步异常进入 arm_cpu_do_interrupt_aarch64 的向量选择

### 6.1 向量偏移

```c
// helper.c:9322
case EXCP_IRQ:
case EXCP_VIRQ:
case EXCP_NMI:
case EXCP_VINMI:
    addr += 0x80;   // IRQ 向量偏移
    break;
case EXCP_FIQ:
case EXCP_VFIQ:
case EXCP_VFNMI:
    addr += 0x100;  // FIQ 向量偏移
    break;
case EXCP_VSERR:
    addr += 0x180;  // SError 向量偏移
    env->exception.syndrome = syn_serror(env->cp15.vsesr_el2 & 0x1ffffff);
    env->cp15.esr_el[new_el] = env->exception.syndrome;
    break;
```

### 6.2 最终向量地址计算

```
最终 PC = VBAR_ELx + base_offset + exception_offset

base_offset:
  从低 EL 来 (cur_el < new_el):
    AArch64 低 EL: +0x400
    AArch32 低 EL: +0x600
  同 EL 来:
    使用 SP_ELx: +0x200
    使用 SP_EL0: +0x000

exception_offset:
  Synchronous: +0x000
  IRQ:         +0x080
  FIQ:         +0x100
  SError:      +0x180
```

示例：EL1(AArch64) 收到 IRQ，路由到 EL2
```
PC = VBAR_EL2 + 0x400 + 0x080 = VBAR_EL2 + 0x480
```

### 6.3 ESR_ELx 处理

- **IRQ/FIQ**: 不写 ESR_ELx（异步异常无 syndrome）
- **SError/VSERR**: 写 ESR_ELx（syndrome 来自 VSESR_EL2 或硬件）
- **同步异常**: 写 ESR_ELx（syndrome 来自 env->exception.syndrome）

---

## 7. NMI 支持 (FEAT_NMI)

### 7.1 NMI 特性概述

FEAT_NMI (Non-Maskable Interrupts) 从 Armv8.8 引入：

- `PSTATE.ALLINT`：mask 所有具有 superpriority 的中断
- `SCTLR_ELx.NMI`：使能 NMI 行为
- `SCTLR_ELx.SPINTMASK`：SP_ELx 使用时自动 mask

### 7.2 QEMU 中 NMI 的优先级处理

```c
if (cpu_isar_feature(aa64_nmi, env_archcpu(env)) &&
    (arm_sctlr(env, cur_el) & SCTLR_NMI)) {
    // NMI 使能：检查 NMI 类中断
    if (interrupt_request & CPU_INTERRUPT_NMI) { ... }
    if (interrupt_request & CPU_INTERRUPT_VINMI) { ... }
    if (interrupt_request & CPU_INTERRUPT_VFNMI) { ... }
} else {
    // NMI 未使能：降级为普通中断
    if (interrupt_request & CPU_INTERRUPT_NMI)
        interrupt_request |= CPU_INTERRUPT_HARD;   // NMI → IRQ
    if (interrupt_request & CPU_INTERRUPT_VINMI)
        interrupt_request |= CPU_INTERRUPT_VIRQ;   // VINMI → VIRQ
    if (interrupt_request & CPU_INTERRUPT_VFNMI)
        interrupt_request |= CPU_INTERRUPT_VFIQ;   // VFNMI → VFIQ
}
```

### 7.3 ALLINT 屏蔽逻辑

```c
if (cpu_isar_feature(aa64_nmi, env_archcpu(env)) &&
    env->cp15.sctlr_el[target_el] & SCTLR_NMI && cur_el == target_el) {
    allIntMask = env->pstate & PSTATE_ALLINT ||
                 ((env->cp15.sctlr_el[target_el] & SCTLR_SPINTMASK) &&
                  (env->pstate & PSTATE_SP));
}
```

ALLINT 设置条件：
- `PSTATE.ALLINT = 1`（MSR 指令显式设置）
- `SCTLR.SPINTMASK = 1` 且 `PSTATE.SP = 1`（使用 SP_ELx 时自动 mask）

---

## 8. 中断从 GIC 到 CPU 的完整路径

### 8.1 物理 IRQ 路径

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Device   │─IRQ─→│   GIC    │─IRQ─→│   CPU    │
│ (Timer)  │      │ (vGIC)   │      │ (vCPU)   │
└──────────┘      └──────────┘      └──────────┘
                       │
                       ▼
              gic_update_internal()
              → 选择最高优先级 pending IRQ
              → qemu_set_irq(cpu_irq[n])
                       │
                       ▼
              arm_cpu_set_irq()
              → cpu_interrupt(cs, CPU_INTERRUPT_HARD)
              → cs->interrupt_request |= CPU_INTERRUPT_HARD
                       │
                       ▼
              cpu_exec() 主循环检测到 interrupt_request
              → arm_cpu_exec_interrupt()
                       │
                       ▼
              arm_phys_excp_target_el() → target_el
              arm_excp_unmasked() → true
                       │
                       ▼
              cs->exception_index = EXCP_IRQ
              env->exception.target_el = target_el
              arm_cpu_do_interrupt(cs)
                       │
                       ▼
              arm_cpu_do_interrupt_aarch64(cs)
              → 保存 SPSR/ELR
              → PC = VBAR_ELx + offset + 0x80
              → 进入中断处理
```

### 8.2 虚拟 IRQ 路径（KVM 场景）

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│ vDevice  │─vIRQ→│  vGIC    │─list─→│ Hypervisor│
│ (vTimer) │      │ (EL2)    │ reg  │  (EL2)    │
└──────────┘      └──────────┘      └──────────┘
                       │
                       ▼
              vGIC 写 List Register
              或 设置 HCR_EL2.VI = 1
                       │
                       ▼
              arm_cpu_update_virq()
              → cpu_interrupt(cs, CPU_INTERRUPT_VIRQ)
                       │
                       ▼
              arm_cpu_exec_interrupt():
              → excp_idx = EXCP_VIRQ
              → target_el = 1 (Guest EL1)
              → arm_excp_unmasked(): HCR.IMO=1 && !TGE && !DAIF.I
                       │
                       ▼
              arm_cpu_do_interrupt_aarch64():
              → new_el = 1 (Guest 内部中断)
              → PC = VBAR_EL1 + 0x480 (Lower EL AArch64 IRQ)
              → Guest OS 中断处理程序
```

---

## 9. SError (异步数据中止) 特殊处理

### 9.1 FEAT_DoubleFault

```c
// helper.c:9268
if (new_el == 3 && (env->cp15.scr_el3 & SCR_EASE) &&
    syndrome_is_sync_extabt(env->exception.syndrome)) {
    addr += 0x180;  // 同步外部中止 → SError 向量
}
```

FEAT_DoubleFault 允许将同步外部中止路由到 SError 向量入口。

### 9.2 VSERR Syndrome 来源

```c
case EXCP_VSERR:
    addr += 0x180;
    env->exception.syndrome = syn_serror(env->cp15.vsesr_el2 & 0x1ffffff);
    env->cp15.esr_el[new_el] = env->exception.syndrome;
    break;
```

虚拟 SError 的 syndrome 来自 `VSESR_EL2` 寄存器（Hypervisor 设置）。

---

## 10. 异步异常与 PSTATE.DAIF 的交互

### 10.1 异常入口时 DAIF 设置

```c
// helper.c:9415
pstate_write(env, PSTATE_DAIF | new_mode);
// 进入中断 handler 时所有异步异常都被 mask
```

### 10.2 中断 handler 需要手动 unmask

```asm
// Linux 内核 entry.S 示例
el1h_64_irq:
    // ... 保存上下文 ...
    msr  daifclr, #2    // 清除 DAIF.I，允许 IRQ 嵌套
    bl   handle_irq
    msr  daifset, #2    // 恢复 DAIF.I mask
    // ... ERET ...
```

### 10.3 ERET 时 DAIF 恢复

ERET 从 SPSR 恢复 DAIF → 回到被中断时的 mask 状态。

---

## 11. 与规范的一致性评估

### 11.1 实现完整度

| 方面 | 状态 | 说明 |
|------|:----:|------|
| IRQ/FIQ 物理路由 | ✅ | Table G1-15 查找表 |
| SCR_EL3.{IRQ,FIQ,EA} | ✅ | 路由到 EL3 |
| HCR_EL2.{IMO,FMO,AMO} | ✅ | 路由到 EL2 |
| HCR_EL2.TGE 强制路由 | ✅ | 合并到 hcr 位 |
| VIRQ/VFIQ/VSERR | ✅ | target_el=1, HCR 控制 |
| PSTATE.DAIF 屏蔽 | ✅ | 完整检查 |
| 高 EL 不可屏蔽 | ✅ | target > cur 时覆盖 |
| FEAT_NMI | ✅ | ALLINT, SPINTMASK, 降级 |
| FEAT_DoubleFault | ✅ | SCR.EASE 检查 |
| VSESR_EL2 | ✅ | 虚拟 SError syndrome |
| NMI → IRQ 降级 | ✅ | SCTLR.NMI=0 时 |

### 11.2 已知简化

| 方面 | 说明 | 严重度 |
|------|------|:------:|
| 中断延迟 | TB 粒度检查，非每指令 | P3 |
| 优先级 | IMPLEMENTATION DEFINED，QEMU 选择固定顺序 | — |
| SError 时序 | 同步检测，非真正异步 | P2 |
| 中断嵌套窗口 | 与 DAIF unmask 时机一致 | ✅ |

### 11.3 总结

异步异常是 QEMU ARM64 中**实现最完善的子系统之一**：
- 6 维查找表精确匹配 ARM ARM Table G1-15
- 虚拟中断（VIRQ/VFIQ/VSERR）完整支持 KVM guest 场景
- FEAT_NMI 全面实现（superpriority、ALLINT、SPINTMASK、降级）
- 唯一的"不精确"是 TB 粒度的检查时机（P3，几乎不影响功能）
