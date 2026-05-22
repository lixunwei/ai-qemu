# ARM64 WFE/WFI Trap 路径分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/68 |
| 分析对象 | WFE/WFI 指令的 Trap 路由机制 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §D1.12 (Traps on WFI/WFE) |
| 关键寄存器 | HCR_EL2.TWE/TWI, SCR_EL3.TWE/TWI, SCTLR_EL1.nTWE/nTWI |
| 核心发现 | **WFE 的 A-profile trap 完全未实现** |

---

## 1. ARM 规范定义

### 1.1 WFE/WFI Trap 控制位层级

ARM DDI 0487 §D1.12 定义了 WFI 和 WFE 的三级 trap 控制：

```
                    ┌─────────────┐
                    │  SCR_EL3    │ ← 第三级：trap 到 EL3
                    │  .TWE/.TWI  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  HCR_EL2    │ ← 第二级：trap 到 EL2
                    │  .TWE/.TWI  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  SCTLR_EL1  │ ← 第一级：trap 到 EL1
                    │  .nTWE/.nTWI│   (注意：n 前缀表示 NOT trap)
                    └─────────────┘
```

### 1.2 各控制位详细语义

| 控制位 | 位置 | 值=1 含义 | 影响范围 |
|--------|------|-----------|----------|
| `SCTLR_EL1.nTWE` | bit[18] | WFE **不** trap 到 EL1 | EL0 → EL1 |
| `SCTLR_EL1.nTWI` | bit[16] | WFI **不** trap 到 EL1 | EL0 → EL1 |
| `HCR_EL2.TWE` | bit[14] | WFE trap 到 EL2 | EL0/EL1 → EL2 |
| `HCR_EL2.TWI` | bit[13] | WFI trap 到 EL2 | EL0/EL1 → EL2 |
| `SCR_EL3.TWE` | bit[13] | WFE trap 到 EL3 | EL0/EL1/EL2 → EL3 |
| `SCR_EL3.TWI` | bit[12] | WFI trap 到 EL3 | EL0/EL1/EL2 → EL3 |

### 1.3 规范规定的优先级

当多个 trap 位同时设置时，优先级为（DDI 0487 §D1.12.4）：

1. **SCTLR_EL1.nTWx = 0**（EL0 执行时）→ trap 到 EL1（最高优先级）
2. **HCR_EL2.TWx = 1**（cur_el < 2 时）→ trap 到 EL2
3. **SCR_EL3.TWx = 1**（cur_el < 3 时）→ trap 到 EL3

### 1.4 Trap Syndrome 编码

WFx trap 使用 `EC = 0x01`（WFx trap），ISS 编码：

```
ISS[24:20] = CV:Cond  (条件码有效性)
ISS[1:0]   = TI       (00=WFI, 01=WFE, 10=WFIT, 11=WFET)
ISS[9:5]   = RN       (WFIT/WFET 的超时寄存器编号)
ISS[4]     = RV       (RN 是否有效)
```

### 1.5 Linux KVM 对 TWE 的使用

Linux KVM 在 `HCR_GUEST_FLAGS` 中设置了 `HCR_TWE`：

```c
// arch/arm64/include/asm/kvm_arm.h
#define HCR_GUEST_FLAGS (HCR_TSC | HCR_TSW | HCR_TWE | HCR_TWI | HCR_VM | \
                         HCR_BSU_IS | HCR_FB | HCR_TAC | \
                         HCR_AMO | HCR_SWIO | HCR_TIDCP | HCR_RW | HCR_TLOR | \
                         HCR_FMO | HCR_IMO | HCR_PTW)
```

目的：Guest 中的 WFE/WFI 被 trap 到 Hypervisor，让 KVM 有机会调度其他 vCPU。

---

## 2. QEMU 实现分析

### 2.1 代码位置

| 文件 | 函数 | 作用 |
|------|------|------|
| `target/arm/tcg/translate-a64.c:2030` | `trans_WFI()` | WFI 翻译 |
| `target/arm/tcg/translate-a64.c:2036` | `trans_WFE()` | WFE 翻译 |
| `target/arm/tcg/translate-a64.c:10919` | `DISAS_WFE` case | WFE TB 结束处理 |
| `target/arm/tcg/translate-a64.c:10927` | `DISAS_WFI` case | WFI TB 结束处理 |
| `target/arm/tcg/op_helper.c:322` | `check_wfx_trap()` | Trap 检查核心逻辑 |
| `target/arm/tcg/op_helper.c:370` | `HELPER(wfi)()` | WFI 运行时 helper |
| `target/arm/tcg/op_helper.c:486` | `HELPER(wfe)()` | WFE 运行时 helper |
| `target/arm/tcg/op_helper.c:526` | `HELPER(yield)()` | Yield helper |
| `target/arm/tcg/op_helper.c:47` | `raise_exception()` | 异常路由 |
| `target/arm/syndrome.h:680` | `syn_wfx()` | Syndrome 编码 |

### 2.2 check_wfx_trap() — Trap 检查核心

```c
// target/arm/tcg/op_helper.c:322-367
static inline int check_wfx_trap(CPUARMState *env, bool is_wfe, uint32_t *excp)
{
    int cur_el = arm_current_el(env);
    uint64_t mask;

    *excp = EXCP_UDEF;

    // M-profile 永不 trap
    if (arm_feature(env, ARM_FEATURE_M)) {
        return 0;
    }

    // 第一级：EL0 → EL1（通过 SCTLR）
    if (cur_el < 1 && arm_feature(env, ARM_FEATURE_V8)) {
        mask = is_wfe ? SCTLR_nTWE : SCTLR_nTWI;
        if (!(arm_sctlr(env, cur_el) & mask)) {
            return exception_target_el(env);  // 通常返回 1
        }
    }

    // 第二级：EL0/EL1 → EL2（通过 HCR_EL2）
    if (cur_el < 2) {
        mask = is_wfe ? HCR_TWE : HCR_TWI;
        if (arm_hcr_el2_eff(env) & mask) {
            return 2;
        }
    }

    // 第三级：→ EL3（通过 SCR_EL3）
    if (arm_feature(env, ARM_FEATURE_V8) && !arm_is_el3_or_mon(env)) {
        mask = (is_wfe) ? SCR_TWE : SCR_TWI;
        if (env->cp15.scr_el3 & mask) {
            if (!arm_el_is_aa64(env, 3)) {
                *excp = EXCP_MON_TRAP;
            }
            return 3;
        }
    }

    return 0;  // 无 trap
}
```

该函数完整实现了 ARM 规范的三级优先级。**但问题是它只被 WFI 调用。**

### 2.3 WFI 完整 Trap 路径（✅ 正确实现）

#### 阶段一：翻译

```c
// translate-a64.c:2030
static bool trans_WFI(DisasContext *s, arg_WFI *a)
{
    s->base.is_jmp = DISAS_WFI;
    return true;
}
```

无条件标记 TB 以 `DISAS_WFI` 结束。

#### 阶段二：TB 结束代码生成

```c
// translate-a64.c:10927
case DISAS_WFI:
    gen_a64_update_pc(dc, 4);              // PC → WFI 下一条指令
    gen_helper_wfi(tcg_env, tcg_constant_i32(4));  // 调用 helper
    tcg_gen_exit_tb(NULL, 0);              // 强制退出 TB
    break;
```

关键点：`tcg_gen_exit_tb(NULL, 0)` 确保无论 helper 是否 trap，都会返回主循环。

#### 阶段三：Runtime Helper 执行

```c
// op_helper.c:370
void HELPER(wfi)(CPUARMState *env, uint32_t insn_len)
{
    CPUState *cs = env_cpu(env);
    uint32_t excp;
    int target_el = check_wfx_trap(env, false, &excp);  // ← 检查 trap

    if (cpu_has_work(cs)) {
        return;  // 有待处理中断，不 sleep 也不 trap
    }

    if (target_el) {
        // 需要 trap：PC 回退到 WFI 自身
        if (env->aarch64) {
            env->pc -= insn_len;
        } else {
            env->regs[15] -= insn_len;
        }
        // 产生 WFx trap 异常（EC=0x01）
        raise_exception(env, excp,
            syn_wfx(1, 0xe, 0, false, WFI, insn_len == 2),
            target_el);
    }

    // 不需要 trap：CPU 真正休眠
    cs->exception_index = EXCP_HLT;
    cs->halted = 1;
    cpu_loop_exit(cs);
}
```

#### 阶段四：异常路由到目标 EL

```c
// op_helper.c:47
void raise_exception(CPUARMState *env, uint32_t excp,
                     uint64_t syndrome, uint32_t target_el)
{
    CPUState *cs = env_cpu(env);

    // 特殊处理：如果 target_el==1 且 HCR_EL2.TGE==1，重定向到 EL2
    if (target_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE)) {
        target_el = 2;
        if (syn_get_ec(syndrome) == EC_ADVSIMDFPACCESSTRAP) {
            syndrome = syn_uncategorized();
        }
    }

    cs->exception_index = excp;
    env->exception.syndrome = syndrome;
    env->exception.target_el = target_el;
    cpu_loop_exit(cs);  // 退出执行循环，进入异常处理
}
```

之后 `arm_cpu_do_interrupt()` 会：
1. 保存当前 PSTATE 到 SPSR_ELx
2. 设置新 EL 的 PSTATE（DAIF mask 等）
3. 保存返回地址到 ELR_ELx
4. 写 ESR_ELx = syndrome
5. 跳转到 VBAR_ELx + offset

### 2.4 WFE 路径（❌ A-profile Trap 未实现）

#### 阶段一：翻译

```c
// translate-a64.c:2036
static bool trans_WFE(DisasContext *s, arg_WFI *a)
{
    // MTTCG 模式下 WFE 直接 NOP（不生成任何代码）
    if (!(tb_cflags(s->base.tb) & CF_PARALLEL)) {
        s->base.is_jmp = DISAS_WFE;
    }
    return true;
}
```

**差异 1**：MTTCG 模式下 WFE 是纯 NOP，连 yield 都不做。

#### 阶段二：TB 结束

```c
// translate-a64.c:10919
case DISAS_WFE:
    gen_a64_update_pc(dc, 4);
    gen_helper_wfe(tcg_env);      // 无 insn_len 参数
    break;                         // ← 注意：没有 tcg_gen_exit_tb!
```

**差异 2**：没有 `tcg_gen_exit_tb()`，意味着 helper 返回后可能继续执行后续 TB。

#### 阶段三：Runtime Helper

```c
// op_helper.c:486
void HELPER(wfe)(CPUARMState *env)
{
#ifdef CONFIG_USER_ONLY
    return;  // user-mode: NOP
#else
    if (arm_feature(env, ARM_FEATURE_M)) {
        // M-profile: 检查 event register，实现真正的 WFE 语义
        CPUState *cs = env_cpu(env);
        if (env->event_register) {
            env->event_register = false;
            return;
        }
        cs->exception_index = EXCP_HLT;
        cs->halted = 1;
        cpu_loop_exit(cs);
    } else {
        // A-profile: ❌ 不检查 trap，直接 yield
        HELPER(yield)(env);
    }
#endif
}
```

**差异 3（核心问题）**：A-profile 分支完全不调用 `check_wfx_trap()`。

#### 阶段四：Yield

```c
// op_helper.c:526
void HELPER(yield)(CPUARMState *env)
{
    CPUState *cs = env_cpu(env);
    // "non-trappable hint instruction"
    cs->exception_index = EXCP_YIELD;
    cpu_loop_exit(cs);
}
```

`EXCP_YIELD` 是 QEMU 内部异常，不会进入 `arm_cpu_do_interrupt()`，
只是让 QEMU 主循环有机会检查其他 vCPU。

---

## 3. WFI vs WFE 对比总结

### 3.1 功能对比表

| 方面 | WFI (✅) | WFE (❌) |
|------|----------|----------|
| 翻译输出 | `DISAS_WFI` (无条件) | `DISAS_WFE` (非MTTCG) |
| TB 结束 | `exit_tb` (强制退出) | 无 exit_tb |
| `check_wfx_trap()` | ✅ 调用 | ❌ 不调用 |
| SCTLR.nTWx 检查 | ✅ | ❌ |
| HCR_EL2.TWx 检查 | ✅ | ❌ |
| SCR_EL3.TWx 检查 | ✅ | ❌ |
| Trap syndrome | EC=0x01, TI=00 | — |
| PC 回退 | ✅ trap时回退 | — |
| 不trap行为 | halted=1 (真休眠) | EXCP_YIELD (让出时间片) |
| MTTCG 模式 | 仍检查 trap | 纯 NOP |
| M-profile | 不trap(return 0) | 有 event register 语义 |

### 3.2 执行流程对比图

```
          WFI                                WFE
          ───                                ───
    ┌──────────────┐                   ┌──────────────┐
    │  trans_WFI   │                   │  trans_WFE   │
    │ DISAS_WFI    │                   │ MTTCG? NOP   │
    └──────┬───────┘                   │ else DISAS_WFE│
           │                           └──────┬───────┘
    ┌──────▼───────┐                   ┌──────▼───────┐
    │ gen_helper_  │                   │ gen_helper_  │
    │ wfi(env,4)   │                   │ wfe(env)     │
    │ exit_tb      │                   │ (no exit_tb) │
    └──────┬───────┘                   └──────┬───────┘
           │                                  │
    ┌──────▼───────┐                   ┌──────▼───────┐
    │check_wfx_trap│                   │ A-profile:   │
    │  ├ SCTLR     │                   │   yield()    │──→ EXCP_YIELD
    │  ├ HCR_EL2   │                   │              │    (内部异常)
    │  └ SCR_EL3   │                   │ M-profile:   │
    └──────┬───────┘                   │   event_reg? │
           │                           └──────────────┘
     ┌─────┴─────┐
     │           │
  target_el>0  target_el=0
     │           │
┌────▼────┐  ┌───▼───┐
│PC回退   │  │halted │
│raise_   │  │=1     │
│exception│  │EXCP_  │
│(EC=0x01)│  │HLT    │
└────┬────┘  └───────┘
     │
┌────▼─────────────┐
│arm_cpu_do_       │
│interrupt()       │
│ → SPSR/ELR保存  │
│ → VBAR跳转      │
│ → 进入目标EL    │
└──────────────────┘
```

---

## 4. Syndrome 编码详解

### 4.1 syn_wfx() 生成的 ESR_ELx 值

```c
// syndrome.h:680
static inline uint32_t syn_wfx(int cv, int cond, int rn, bool rv,
                                wfx_ti ti, bool is_16bit)
{
    uint32_t res = syn_set_ec(0, EC_WFX_TRAP);  // EC = 0x01
    res = FIELD_DP32(res, SYNDROME, IL, !is_16bit);  // IL=1 (32-bit)
    res = FIELD_DP32(res, WFX_ISS, CV, cv);     // CV=1 (cond valid)
    res = FIELD_DP32(res, WFX_ISS, COND, cond); // COND=0xe (AL)
    res = FIELD_DP32(res, WFX_ISS, TI, ti);     // TI=00(WFI)/01(WFE)
    res = FIELD_DP32(res, WFX_ISS, RN, rn);     // WFIT/WFET用
    res = FIELD_DP32(res, WFX_ISS, RV, rv);     // RN有效标志
    return res;
}
```

### 4.2 WFI Trap 的完整 ESR 值

对于 AArch64 WFI trap：
```
ESR_ELx = 0x04000000 | (1 << 25) | (0xe << 20) | (1 << 24) | 0x00
        = EC[31:26]=0x01, IL=1, CV=1, COND=0xe, TI=0b00
```

Hypervisor 解析 `ESR_EL2.EC == 0x01` 即知是 WFx trap，
再看 `TI` 字段区分 WFI/WFE/WFIT/WFET。

---

## 5. raise_exception() 路由机制

### 5.1 函数逻辑

```c
void raise_exception(CPUARMState *env, uint32_t excp,
                     uint64_t syndrome, uint32_t target_el)
{
    CPUState *cs = env_cpu(env);

    // HCR_EL2.TGE 重定向：EL1异常 → EL2
    if (target_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE)) {
        target_el = 2;
        // SIMD/FP trap 变为 uncategorized
        if (syn_get_ec(syndrome) == EC_ADVSIMDFPACCESSTRAP) {
            syndrome = syn_uncategorized();
        }
    }

    cs->exception_index = excp;
    env->exception.syndrome = syndrome;
    env->exception.target_el = target_el;
    cpu_loop_exit(cs);  // 退出到 QEMU 主循环
}
```

### 5.2 cpu_loop_exit 之后

`cpu_loop_exit()` 通过 `longjmp` 回到 `cpu_exec()` 主循环：

```
cpu_exec()
  → sigsetjmp()
  → ... 执行 TB ...
  → cpu_loop_exit() [longjmp 回到 sigsetjmp]
  → 检查 cs->exception_index
  → 调用 cc->tcg_ops->do_interrupt(cs)
       = arm_cpu_do_interrupt(cs)
```

### 5.3 arm_cpu_do_interrupt() 处理 WFx Trap

`arm_cpu_do_interrupt()` 对 target_el=2 的处理：

1. 保存 `PSTATE` → `SPSR_EL2`
2. 保存返回地址 → `ELR_EL2`（= WFI 指令地址，因为 PC 已回退）
3. 写 `ESR_EL2` = syndrome（EC=0x01）
4. 设置新 PSTATE：`DAIF = 0xF`（mask 所有中断）
5. 跳转到 `VBAR_EL2 + 0x400`（从低 EL 来的 AArch64 Synchronous）

---

## 6. exception_target_el() 详解

### 6.1 默认目标 EL 计算

```c
// op_helper.c:32
int exception_target_el(CPUARMState *env)
{
    int target_el = MAX(1, arm_current_el(env));

    // Secure EL1 + AArch32 EL3 → 路由到 EL3
    if (arm_is_secure(env) && !arm_el_is_aa64(env, 3) && target_el == 1) {
        target_el = 3;
    }

    return target_el;
}
```

这个函数用于 SCTLR.nTWx trap（第一级），当 EL0 的 WFx 需要 trap 到 "当前管理 EL" 时：
- EL0 → target = MAX(1, 0) = 1（trap 到 EL1）
- 特殊情况：Secure + AArch32 EL3 → target = 3

### 6.2 check_wfx_trap 中 SCTLR 分支为何用 exception_target_el

```c
if (cur_el < 1 && arm_feature(env, ARM_FEATURE_V8)) {
    mask = is_wfe ? SCTLR_nTWE : SCTLR_nTWI;
    if (!(arm_sctlr(env, cur_el) & mask)) {
        return exception_target_el(env);  // 通常返回 1
    }
}
```

为什么不直接 `return 1`？因为 Secure EL1 + AArch32 EL3 的特殊情况，
此时应该 trap 到 EL3 而不是 EL1。

---

## 7. FEAT_TWED（Trap WFE Delay）

### 7.1 规范定义

ARM DDI 0487 定义了 FEAT_TWED（从 Armv8.6 开始）：

- `SCTLR_EL1.TWEDEn`（bit[45]）：使能 WFE delay
- `SCTLR_EL1.TWEDEL`（bit[49:46]）：delay 值（2^(TWEDEL+8) 个周期）
- `HCR_EL2.TWEDEN`（bit[59]）/ `HCR_EL2.TWEDEL`（bit[63:60]）
- `SCR_EL3.TWEDEN`（bit[29]）/ `SCR_EL3.TWEDEL`（bit[33:30]）

含义：WFE 在 delay 时间内不 trap，超过后才 trap。用于减少不必要的 VM-exit。

### 7.2 QEMU 寄存器定义

```c
// cpu.h:1470-1471
#define SCTLR_TWEDEn  (1ULL << 45)
#define SCTLR_TWEDEL  MAKE_64_MASK(46, 4)

// cpu.h:1753-1754
#define HCR_TWEDEN    (1ULL << 59)
#define HCR_TWEDEL    MAKE_64BIT_MASK(60, 4)

// cpu.h:1782-1783
#define SCR_TWEDEN    (1ULL << 29)
#define SCR_TWEDEL    MAKE_64BIT_MASK(30, 4)

// cpu-features.h:308
FIELD(ID_AA64MMFR1, TWED, 32, 4)
```

### 7.3 QEMU 实现状态

QEMU 定义了 TWED 相关的位掩码，但由于 WFE trap 本身未实现，
FEAT_TWED 的 delay 逻辑自然也不存在。

---

## 8. 影响分析

### 8.1 对 TCG 模拟的影响

| 场景 | 影响程度 | 说明 |
|------|----------|------|
| KVM guest (HW加速) | ⚠️ 无影响 | KVM 在真实硬件执行，硬件处理 trap |
| TCG guest OS | ⚠️ 中等 | Guest hypervisor 无法通过 WFE trap 调度 vCPU |
| TCG 嵌套虚拟化 | ❌ 缺失 | Guest L1 hypervisor 看不到 L2 的 WFE |
| Bare-metal 固件 | ⚠️ 低 | 固件通常不依赖 WFE trap |
| Spin-lock 优化 | ⚠️ 低 | WFE 在 spin-lock 中主要用于省电 |

### 8.2 为什么 QEMU 选择不实现

代码注释透露了设计意图：

```c
// translate-a64.c:2038-2043
/*
 * When running in MTTCG we don't generate jumps to the yield and
 * WFE helpers as it won't affect the scheduling of other vCPUs.
 * If we wanted to more completely model WFE/SEV so we don't busy
 * spin unnecessarily we would need to do something more involved.
 */
```

```c
// op_helper.c:514-520
/*
 * For A-profile and others, we rely on the existing "yield" behavior.
 * Don't actually halt the CPU, just yield back to top
 * level loop. This is not going into a "low power state"
 * (ie halting until some event occurs), so we never take
 * a configurable trap to a different exception level
 */
```

QEMU 的立场：
1. WFE 是 **hint 指令**（NOP 是合法实现）
2. 真正的 WFE 语义需要 event register + SEV 配合，实现复杂
3. Yield 已经解决了单线程 TCG 的 busy-loop 问题
4. MTTCG 模式下 WFE 应该真正影响 host 调度，但目前没做

### 8.3 与规范的差异等级

| 差异项 | 严重度 | 理由 |
|--------|--------|------|
| WFE trap 到 EL2 未实现 | P2 | hint 指令，NOP 合法，但 hypervisor 功能受限 |
| FEAT_TWED 未实现 | P3 | 依赖于 WFE trap，且是优化特性 |
| WFE event register (A-profile) | P2 | SEV/WFE 配对不工作 |
| MTTCG 下 WFE 是纯 NOP | P3 | 不影响正确性，影响性能模型 |

---

## 9. 修复建议

### 9.1 最小修复：添加 Trap 检查

```c
void HELPER(wfe)(CPUARMState *env)
{
#ifdef CONFIG_USER_ONLY
    return;
#else
    if (arm_feature(env, ARM_FEATURE_M)) {
        // M-profile: 保持现有 event register 逻辑
        CPUState *cs = env_cpu(env);
        if (env->event_register) {
            env->event_register = false;
            return;
        }
        cs->exception_index = EXCP_HLT;
        cs->halted = 1;
        cpu_loop_exit(cs);
    } else {
        // A-profile: 添加 trap 检查
        uint32_t excp;
        int target_el = check_wfx_trap(env, true, &excp);  // is_wfe=true

        if (target_el) {
            if (env->aarch64) {
                env->pc -= 4;
            } else {
                env->regs[15] -= (env->thumb ? 2 : 4);
            }
            raise_exception(env, excp,
                syn_wfx(1, 0xe, 0, false, WFE, !env->aarch64 && env->thumb),
                target_el);
        }

        // 不 trap 时保持 yield 行为
        HELPER(yield)(env);
    }
#endif
}
```

### 9.2 完整修复：还需处理 WFET

`trans_WFET()` 同样没有 trap 检查，需要类似修复。

### 9.3 注意事项

- PC 回退必须在 trap 之前完成（规范要求 ELR_ELx 指向 WFE 本身）
- 需要考虑 AArch32 Thumb 模式（指令长度 2 vs 4）
- `syn_wfx` 的 TI 字段需要正确设置为 WFE(01) 而非 WFI(00)

---

## 10. 总结

### 10.1 核心结论

QEMU 对 WFI 和 WFE 的 trap 实现存在严重不对称：

- **WFI**：完整实现了三级 trap 优先级（SCTLR → HCR_EL2 → SCR_EL3），
  包括正确的 PC 回退、syndrome 编码、和异常路由。
  
- **WFE**：A-profile 完全跳过 trap 检查，直接作为 yield/NOP 处理。
  虽然 ARM 规范允许 WFE 作为 NOP 实现（hint 指令），
  但 trap 是**强制行为**（不依赖 WFE 本身是否 NOP）。

### 10.2 规范引用

> DDI 0487 §D1.12.3: "If the trap is enabled, the PE must take the trap
> regardless of whether WFE would otherwise complete immediately."

即使 WFE 不会实际等待（event register 已 set），只要 trap 使能，
就**必须**产生 trap 异常。QEMU 当前实现违反了这一强制要求。

### 10.3 数据流总览

```
EL0 执行 WFE
    │
    ▼
翻译：DISAS_WFE (非MTTCG) / NOP (MTTCG)
    │
    ▼
helper_wfe()
    │
    ├── M-profile → event_register 语义 (✅)
    │
    └── A-profile → yield() 直接返回 (❌ 应先检查 trap)
                        │
                        │ [应该的路径]
                        ▼
                    check_wfx_trap(is_wfe=true)
                        │
                    ┌───┼───┐
                    │   │   │
                 SCTLR HCR SCR
                 nTWE  TWE TWE
                 =0?   =1? =1?
                    │   │   │
                    ▼   ▼   ▼
                   EL1 EL2 EL3
                    │
                    ▼
              raise_exception(EC=0x01, TI=WFE)
                    │
                    ▼
              arm_cpu_do_interrupt()
              → SPSR_EL2 = PSTATE
              → ELR_EL2 = WFE 地址
              → ESR_EL2 = 0x04000001 (EC=1,TI=01)
              → PC = VBAR_EL2 + 0x400
```
