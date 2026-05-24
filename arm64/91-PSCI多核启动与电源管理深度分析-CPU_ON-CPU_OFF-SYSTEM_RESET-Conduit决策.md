# QEMU ARM64 PSCI 多核启动与电源管理深度分析

> QEMU 版本：11.0.50  
> 源码路径：`/home/nio/sda/source/qemu`  
> 分析范围：target/arm/tcg/psci.c (226行)、target/arm/arm-powerctl.c (311行)、target/arm/cpu.c (arm_emulate_firmware_reset)  
> 规范参考：ARM DEN 0022D.b (PSCI Specification)、ARM DEN 0028 (SMC Calling Convention)  
> 关联文档：[virt Machine初始化](../architecture/90-virt-Machine初始化深度分析-内存布局-GIC创建-PCIe-ACPI-FDT-固件加载.md) · [KVM加速器](../architecture/86-KVM加速器集成深度分析-ioctl-vCPU运行循环-寄存器同步-ARM64架构钩子.md)

---

## 目录

1. [PSCI 概述与架构定位](#1-psci-概述与架构定位)
2. [QEMU PSCI 实现架构](#2-qemu-psci-实现架构)
3. [PSCI Conduit 选择机制](#3-psci-conduit-选择机制)
4. [TCG 模式：SMC/HVC 拦截流程](#4-tcg-模式smchvc-拦截流程)
5. [PSCI 函数分发：arm_handle_psci_call()](#5-psci-函数分发arm_handle_psci_call)
6. [CPU_ON：多核启动核心流程](#6-cpu_on多核启动核心流程)
7. [CPU_OFF：CPU 关闭](#7-cpu_offcpu-关闭)
8. [CPU_SUSPEND：挂起语义](#8-cpu_suspend挂起语义)
9. [SYSTEM_RESET / SYSTEM_OFF](#9-system_reset--system_off)
10. [AFFINITY_INFO：电源状态查询](#10-affinity_info电源状态查询)
11. [PSCI_FEATURES：能力发现](#11-psci_features能力发现)
12. [arm_emulate_firmware_reset()：固件模拟](#12-arm_emulate_firmware_reset固件模拟)
13. [Secondary CPU 初始 Powered-Off 状态](#13-secondary-cpu-初始-powered-off-状态)
14. [KVM 模式 PSCI 处理](#14-kvm-模式-psci-处理)
15. [Power State 状态机](#15-power-state-状态机)
16. [与 ARM PSCI 规范对比](#16-与-arm-psci-规范对比)
17. [与真实固件 (ATF/TF-A) 对比](#17-与真实固件-atftf-a-对比)

---

## 1. PSCI 概述与架构定位

**PSCI (Power State Coordination Interface)** 是 ARM 定义的标准固件接口，用于：
- **多核启动**：Primary CPU 唤醒 Secondary CPU
- **CPU 电源管理**：on/off/suspend/resume
- **系统级操作**：reset/shutdown

在真实硬件上，PSCI 由 **ATF (ARM Trusted Firmware)** 在 EL3 实现。QEMU 直接在 hypervisor 层面截获 SMC/HVC 指令并模拟 PSCI 行为，充当"假 EL3 固件"。

### 源码组织

| 文件 | 行数 | 职责 |
|------|------|------|
| `target/arm/tcg/psci.c` | 226 | PSCI 调用识别与分发 |
| `target/arm/arm-powerctl.c` | 311 | CPU 电源操作实现 |
| `target/arm/arm-powerctl.h` | 93 | 接口定义与返回码 |
| `target/arm/kvm-consts.h` | ~100 | PSCI 函数号常量 |
| `target/arm/cpu.c:661` | ~120 | arm_emulate_firmware_reset() |

---

## 2. QEMU PSCI 实现架构

```
Guest OS (Linux)
    │
    │ SMC #0 / HVC #0  (param[0] = PSCI function ID)
    ▼
┌─────────────────────────────────────────┐
│ TCG: EXCP_SMC / EXCP_HVC               │
│                                          │
│ op_helper.c:                            │
│   arm_is_psci_call(cpu, EXCP_xxx)?      │
│     ├── YES → arm_handle_psci_call()    │
│     └── NO  → 正常异常处理              │
│               (EL2 trap / EL3 / UNDEF)  │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ psci.c: arm_handle_psci_call()          │
│                                          │
│ switch (param[0]) {                     │
│   CPU_ON   → arm_set_cpu_on()           │
│   CPU_OFF  → arm_set_cpu_off()          │
│   SUSPEND  → helper_wfi()              │
│   SYSTEM_RESET → qemu_system_reset()   │
│   SYSTEM_OFF   → qemu_system_shutdown()│
│   VERSION  → return 1.1                │
│   FEATURES → check function support    │
│ }                                        │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ arm-powerctl.c: CPU 电源操作            │
│                                          │
│ arm_set_cpu_on():                       │
│   1. arm_get_cpu_by_id(mpidr)           │
│   2. 检查 power_state != ON/ON_PENDING  │
│   3. async_run_on_cpu(target, work)     │
│      → cpu_reset() + set entry/ctx      │
│      → target->halted = 0              │
│      → power_state = PSCI_ON           │
└─────────────────────────────────────────┘
```

---

## 3. PSCI Conduit 选择机制

### 3.1 决策逻辑（virt Machine）

```c
// hw/arm/virt.c:2676-2682
if (secure && firmware_loaded) {
    // 外部固件（如 ATF）自己处理 PSCI
    psci_conduit = QEMU_PSCI_CONDUIT_DISABLED;
} else if (vms->virt) {
    // 有 EL2: 用 SMC (像 EL3 固件一样)
    psci_conduit = QEMU_PSCI_CONDUIT_SMC;
} else {
    // 无 EL2: 用 HVC (兼容旧行为 + KVM 要求)
    psci_conduit = QEMU_PSCI_CONDUIT_HVC;
}
```

### 3.2 Conduit 传递给 Guest

| 接口 | 位置 | 值 |
|------|------|---|
| FDT | `/psci { method = "hvc" \| "smc" }` | 字符串 |
| ACPI FADT | `arm_boot_arch` | PSCI_COMPLIANT \| PSCI_USE_HVC |

### 3.3 Conduit 传递到 CPU

```c
// hw/arm/boot.c 中通过 arm_boot_info.psci_conduit
// 设置到每个 CPU 的 cpu->psci_conduit 字段
```

---

## 4. TCG 模式：SMC/HVC 拦截流程

### 4.1 HVC 拦截

```c
// target/arm/tcg/op_helper.c:1079-1084
// 在 HELPER(exception_with_syndrome)(EXCP_HVC) 处理中:
if (arm_is_psci_call(cpu, EXCP_HVC)) {
    // PSCI 调用覆盖正常 HVC 行为
    return;  // arm_handle_psci_call() 已在调用链中执行
}
// 否则: 正常 HVC 行为 (trap to EL2 / UNDEF)
```

### 4.2 SMC 拦截

```c
// target/arm/tcg/op_helper.c:1195-1199
if (!arm_is_psci_call(cpu, EXCP_SMC) &&
    (smd || !arm_feature(env, ARM_FEATURE_EL3))) {
    // 不是 PSCI 且无 EL3/SMD → UNDEF
    raise_exception(env, EXCP_UDEF, ...);
}
// 是 PSCI → arm_handle_psci_call() 处理
```

### 4.3 arm_is_psci_call() 判定

```c
// target/arm/tcg/psci.c:30-56
bool arm_is_psci_call(ARMCPU *cpu, int excp_type) {
    switch (excp_type) {
    case EXCP_HVC:
        return cpu->psci_conduit == QEMU_PSCI_CONDUIT_HVC;
    case EXCP_SMC:
        return cpu->psci_conduit == QEMU_PSCI_CONDUIT_SMC;
    default:
        return false;
    }
}
```

关键点：**只要 conduit 匹配，所有 HVC/SMC 都被当作 PSCI 处理**（不检查函数号），不支持的函数号返回 NOT_SUPPORTED。

---

## 5. PSCI 函数分发：arm_handle_psci_call()

```c
// target/arm/tcg/psci.c:58-226
void arm_handle_psci_call(ARMCPU *cpu) {
    // 参数读取：AArch64 用 xregs[0-3]，AArch32 用 regs[0-3]
    for (i = 0; i < 4; i++)
        param[i] = is_a64(env) ? env->xregs[i] : env->regs[i];
    
    // 64 位函数在 32 位模式下不支持
    if ((param[0] & QEMU_PSCI_0_2_64BIT) && !is_a64(env))
        → NOT_SUPPORTED
    
    switch (param[0]) { ... }  // 分发
    
    // 返回值写入 x0/r0
    env->xregs[0] = ret;  // 或 env->regs[0]
}
```

### 支持的函数号

| 函数 | 32-bit ID | 64-bit ID | 实现 |
|------|-----------|-----------|------|
| PSCI_VERSION | 0x84000000 | — | 返回 1.1 |
| CPU_SUSPEND | 0x84000001 | 0xC4000001 | WFI |
| CPU_OFF | 0x84000002 | — | halted + PSCI_OFF |
| CPU_ON | 0x84000003 | 0xC4000003 | reset + jump |
| AFFINITY_INFO | 0x84000004 | 0xC4000004 | 查询 power_state |
| MIGRATE_INFO_TYPE | 0x84000006 | — | MIGRATION_NOT_REQUIRED |
| SYSTEM_OFF | 0x84000008 | — | shutdown |
| SYSTEM_RESET | 0x84000009 | — | reset |
| PSCI_FEATURES | 0x8400000A | — | 能力查询 |

---

## 6. CPU_ON：多核启动核心流程

### 6.1 PSCI 层参数准备

```c
// psci.c:133-157
case QEMU_PSCI_0_2_FN_CPU_ON:
case QEMU_PSCI_0_2_FN64_CPU_ON:
{
    // target_el: 调用方当前最高可用 NS EL
    int target_el = arm_feature(env, ARM_FEATURE_EL2) ? 2 : 1;
    // target_aarch64: 调用方该 EL 的执行模式
    bool target_aarch64 = arm_el_is_aa64(env, target_el);
    
    mpidr = param[1];          // 目标 CPU 的 MPIDR
    entry = param[2];          // 入口地址
    context_id = param[3];     // 传给目标 CPU 的 x0/r0
    
    ret = arm_set_cpu_on(mpidr, entry, context_id, target_el, target_aarch64);
}
```

### 6.2 arm_set_cpu_on() 实现

```c
// arm-powerctl.c:84-176
int arm_set_cpu_on(uint64_t cpuid, uint64_t entry, uint64_t context_id,
                   uint32_t target_el, bool target_aa64) {
    // 1. 验证
    assert(target_el >= 1 && target_el <= 3);
    if (target_aa64 && (entry & 3)) return INVALID_PARAM;  // 4字节对齐
    
    // 2. 查找目标 CPU (通过 MPIDR)
    target_cpu_state = arm_get_cpu_by_id(cpuid);
    if (!target_cpu_state) return INVALID_PARAM;
    
    // 3. 状态检查
    if (power_state == PSCI_ON) return ALREADY_ON;
    if (power_state == PSCI_ON_PENDING) return ON_PENDING;
    
    // 4. 异步执行 (在目标 CPU 线程上下文中)
    info = { entry, context_id, target_el, target_aa64 };
    async_run_on_cpu(target_cpu_state, arm_set_cpu_on_async_work, info);
    
    return SUCCESS;
}
```

### 6.3 目标 CPU 上执行的工作

```c
// arm-powerctl.c:49-82
static void arm_set_cpu_on_async_work(CPUState *target_cpu_state, ...) {
    // 1. 完整 CPU 复位
    cpu_reset(target_cpu_state);
    
    // 2. 模拟固件设置 (配置 EL3/EL2 寄存器使目标 EL 可用)
    arm_emulate_firmware_reset(target_cpu_state, info->target_el);
    
    // 3. 取消挂起
    target_cpu_state->halted = 0;
    
    // 4. 验证目标 EL
    assert(info->target_el == arm_current_el(&target_cpu->env));
    
    // 5. 设置 context_id (x0/r0)
    target_cpu->env.xregs[0] = info->context_id;  // AArch64
    // 或 target_cpu->env.regs[0] = ...            // AArch32
    
    // 6. 重建 hflags (TCG 模式)
    arm_rebuild_hflags(&target_cpu->env);
    
    // 7. 设置 PC 为入口地址
    cpu_set_pc(target_cpu_state, info->entry);
    
    // 8. 更新电源状态
    target_cpu->power_state = PSCI_ON;
}
```

### 6.4 arm_get_cpu_by_id() — MPIDR 查找

```c
// arm-powerctl.c:22-39
CPUState *arm_get_cpu_by_id(uint64_t id) {
    CPUState *cpu;
    CPU_FOREACH(cpu) {
        ARMCPU *armcpu = ARM_CPU(cpu);
        if (arm_cpu_mp_affinity(armcpu) == id)
            return cpu;
    }
    return NULL;  // 未找到
}
```

---

## 7. CPU_OFF：CPU 关闭

```c
// psci.c:158-160
case QEMU_PSCI_0_2_FN_CPU_OFF:
    goto cpu_off;

// psci.c:221-225
cpu_off:
    ret = arm_set_cpu_off(arm_cpu_mp_affinity(cpu));
    assert(ret == SUCCESS);  // 自己关闭自己不应失败
```

### arm_set_cpu_off() 实现

```c
// arm-powerctl.c:236-274
static void arm_set_cpu_off_async_work(CPUState *target_cpu_state, ...) {
    target_cpu->power_state = PSCI_OFF;
    target_cpu_state->halted = 1;
    target_cpu_state->exception_index = EXCP_HLT;
}

int arm_set_cpu_off(uint64_t cpuid) {
    target_cpu_state = arm_get_cpu_by_id(cpuid);
    if (target_cpu->power_state == PSCI_OFF) return IS_OFF;
    
    async_run_on_cpu(target_cpu_state, arm_set_cpu_off_async_work, NULL);
    return SUCCESS;
}
```

关键点：
- `halted = 1`：CPU 进入挂起状态，不再被调度执行
- `exception_index = EXCP_HLT`：标记为 halt 异常
- `power_state = PSCI_OFF`：阻止 `arm_cpu_is_work()` 返回 true

---

## 8. CPU_SUSPEND：挂起语义

```c
// psci.c:161-176
case QEMU_PSCI_0_2_FN_CPU_SUSPEND:
case QEMU_PSCI_0_2_FN64_CPU_SUSPEND:
    // Affinity levels 不支持 (QEMU 简化)
    if (param[1] & 0xfffe0000)
        return INVALID_PARAMS;
    
    // 不支持 powerdown，总是退化为 WFI
    env->xregs[0] = 0;  // 返回值 = SUCCESS (中断唤醒后)
    helper_wfi(env, 4);  // 等待中断
    break;
```

**QEMU 简化**：PSCI 规范定义了 standby/retention/powerdown 等多种挂起级别，QEMU 将所有级别统一实现为 **WFI (Wait For Interrupt)**。

---

## 9. SYSTEM_RESET / SYSTEM_OFF

```c
// psci.c:122-132
case QEMU_PSCI_0_2_FN_SYSTEM_RESET:
    qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
    goto cpu_off;  // PSCI 规范要求不返回

case QEMU_PSCI_0_2_FN_SYSTEM_OFF:
    qemu_system_shutdown_request(SHUTDOWN_CAUSE_GUEST_SHUTDOWN);
    goto cpu_off;  // 不返回
```

**语义**：
- `SYSTEM_RESET`：请求整机复位（异步执行）
- `SYSTEM_OFF`：请求整机关机
- 两者都在发起后将当前 CPU 关闭（因为规范要求不返回）

---

## 10. AFFINITY_INFO：电源状态查询

```c
// psci.c:101-121
case QEMU_PSCI_0_2_FN_AFFINITY_INFO:
    mpidr = param[1];
    switch (param[2]) {  // Affinity level
    case 0:  // 单个 CPU
        target_cpu_state = arm_get_cpu_by_id(mpidr);
        ret = target_cpu->power_state;  // PSCI_ON(0) 或 PSCI_OFF(1)
        break;
    default:
        // Affinity level > 0: 总是 ON (QEMU 不建模 cluster 电源)
        ret = 0;
    }
```

---

## 11. PSCI_FEATURES：能力发现

```c
// psci.c:177-205
case QEMU_PSCI_1_0_FN_PSCI_FEATURES:
    switch (param[1]) {
    // 列举所有支持的函数:
    case VERSION / MIGRATE_INFO_TYPE / AFFINITY_INFO / SYSTEM_RESET /
         SYSTEM_OFF / CPU_ON / CPU_OFF / CPU_SUSPEND / PSCI_FEATURES:
        if (!(param[1] & 64BIT) || is_a64(env))
            ret = 0;  // 支持 (features bitmap = 0)
        break;
    default:
        ret = NOT_SUPPORTED;
    }
```

QEMU 报告 PSCI 版本 **1.1** (`QEMU_PSCI_VERSION_1_1 = 0x10001`)。

---

## 12. arm_emulate_firmware_reset()：固件模拟

当 CPU_ON 唤醒目标 CPU 时，需要模拟 ATF/EL3 固件的初始化行为：

```c
// target/arm/cpu.c:661-777
void arm_emulate_firmware_reset(CPUState *cpustate, int target_el) {
    // 如果 target_el 就是最高 EL，cpu_reset 已完成，直接返回
    if (target_el == 3 && have_el3) return;
    if (target_el == 2 && !have_el3) return;
    if (target_el == 1 && !have_el3 && !have_el2) return;
    
    // === 配置 EL3 (模拟 ATF 设置) ===
    if (have_el3) {
        // 基本 NS 使能
        scr_el3 |= SCR_RW;          // EL2 为 AArch64
        scr_el3 |= SCR_NS;          // Non-Secure 状态
        nsacr |= 3 << 10;           // NS 可访问 FPU
        
        // 特性透传 (让低 EL 可使用)
        if (pauth)   scr_el3 |= SCR_API | SCR_APK;
        if (mte)     scr_el3 |= SCR_ATA;
        if (sve)     cptr_el3.EZ = 1; zcr_el3 = 0xf;
        if (sme)     cptr_el3.ESM = 1; scr_el3 |= SCR_ENTP2;
        if (hcx)     scr_el3 |= SCR_HXEN;
        if (fgt)     scr_el3 |= SCR_FGTEN;
        if (gcs)     scr_el3 |= SCR_GCSEN;
        if (tcr2)    scr_el3 |= SCR_TCR2EN;
        if (sctlr2)  scr_el3 |= SCR_SCTLR2EN;
        if (s1pie/s2pie) scr_el3 |= SCR_PIEN;
        if (aie)     scr_el3 |= SCR_AIEN;
        if (mec)     scr_el3 |= SCR_MECEN;
        
        // 如果目标是 EL2，允许 HVC
        if (target_el == 2) scr_el3 |= SCR_HCE;
    }
    
    // === 配置 EL2 (让 EL1 可用) ===
    if (have_el2 && target_el < 2) {
        if (aarch64) hcr_el2 |= HCR_RW;  // EL1 为 AArch64
    }
    
    // === 设置目标 EL 的执行状态 ===
    if (aarch64) {
        pstate = aarch64_pstate_mode(target_el, true);  // ELx_h (SP_ELx)
    } else {
        // AArch32: EL1→SVC, EL2→HYP, EL3→SVC(Mon)
        cpsr = mode_for_el[target_el];
    }
}
```

### 关键理解

这个函数做的是**真实固件 (ATF) 在引导阶段对 EL3 寄存器的配置**：
- `SCR_EL3.RW=1`：使下层 EL 可运行 AArch64
- `SCR_EL3.NS=1`：Non-Secure 世界
- 各种 `SCR_EL3` 位：透传 CPU 新特性给 Guest

---

## 13. Secondary CPU 初始 Powered-Off 状态

### 13.1 机制

```c
// hw/core/cpu-common.c:113
cpu->halted = cpu->start_powered_off;

// target/arm/cpu.c:329 (在 cpu_reset 中)
cpu->power_state = cs->start_powered_off ? PSCI_OFF : PSCI_ON;
```

### 13.2 virt Machine 如何设置

在 `hw/arm/boot.c` 的 `do_cpu_reset()` 回调中，secondary CPU（非 primary）在 PSCI 启用时被设为 powered-off：

```c
// boot.c:743 — do_arm_linux_init() 为 secondary CPU 设置
// PSCI 模式下, virt Machine 使 secondary CPU start_powered_off=true
// 通过 CPU property "start-powered-off" = true
```

### 13.3 CPU Work 调度保护

```c
// target/arm/cpu.c:148-159
// arm_cpu_has_work():
return (cpu->power_state != PSCI_OFF)    // OFF 状态的 CPU 不响应任何中断
    && cpu_test_interrupt(cs, ALL_INTERRUPTS);
```

当 `power_state == PSCI_OFF` 时：
- CPU 处于 `halted = 1` 状态
- 不会被中断唤醒
- 只能被其他 CPU 的 `CPU_ON` 唤醒

---

## 14. KVM 模式 PSCI 处理

### 14.1 KVM 处理方式

在 KVM 模式下，PSCI 由 **KVM 内核模块** 直接处理，不经过 QEMU：

```c
// target/arm/kvm.c:1975-1984
// vCPU 初始化时配置:
if (cs->start_powered_off) {
    kvm_init_features[0] |= 1 << KVM_ARM_VCPU_POWER_OFF;
}
if (psci_version >= 0.2 && KVM_CAP_ARM_PSCI_0_2) {
    kvm_init_features[0] |= 1 << KVM_ARM_VCPU_PSCI_0_2;
}

// 设置 PSCI 版本寄存器:
kvm_set_one_reg(cs, KVM_REG_ARM_PSCI_VERSION, &psciver);
```

### 14.2 KVM vs TCG PSCI 对比

| 方面 | KVM | TCG |
|------|-----|-----|
| 处理层 | 内核 KVM 模块 | QEMU 用户态 |
| 拦截方式 | KVM 直接处理 SMC/HVC | EXCP_SMC/HVC → helper |
| CPU_ON | KVM_ARM_VCPU_POWER_OFF 标志 | async_run_on_cpu() |
| 版本 | KVM_REG_ARM_PSCI_VERSION | 硬编码 1.1 |
| 特性 | 依赖内核版本 | 完整实现 |
| 性能 | 无 VM exit 开销 | TCG helper 调用 |

### 14.3 MP State 同步

```c
// target/arm/kvm.c:1136
// KVM_SET_MP_STATE:
.mp_state = (cpu->power_state == PSCI_OFF) ?
    KVM_MP_STATE_STOPPED : KVM_MP_STATE_RUNNABLE;

// KVM_GET_MP_STATE:
cpu->power_state = (mp_state == KVM_MP_STATE_STOPPED) ?
    PSCI_OFF : PSCI_ON;
```

---

## 15. Power State 状态机

```
                    ┌──────────────────────────┐
                    │                          │
                    ▼                          │
              ┌──────────┐   CPU_ON      ┌────┴─────┐
    reset ──> │ PSCI_OFF │ ──────────── >│ PSCI_ON  │
              │ halted=1 │               │ halted=0 │
              └──────────┘               └──────────┘
                    ▲                          │
                    │    CPU_OFF               │
                    └──────────────────────────┘
                    
              ┌──────────────┐
              │ ON_PENDING   │  (CPU_ON 已发起但尚未完成)
              │ 瞬态         │
              └──────────────┘
```

状态值定义（来自 kvm-consts.h）：
- `PSCI_ON = 0`
- `PSCI_OFF = 1`  
- `PSCI_ON_PENDING = 2`

### 竞态保护

```c
// arm-powerctl.c:154-159
// 如果目标正在 ON_PENDING，新的 CPU_ON 请求失败:
if (target_cpu->power_state == PSCI_ON_PENDING)
    return QEMU_ARM_POWERCTL_ON_PENDING;
```

这符合 PSCI 规范 §6.6 (CPU_ON/CPU_OFF races)。

---

## 16. 与 ARM PSCI 规范对比

| 规范功能 | QEMU 实现 | 差异 |
|---------|-----------|------|
| PSCI_VERSION | ✅ 返回 1.1 | 规范最新 1.3，QEMU 报告 1.1 |
| CPU_ON (v0.1/v0.2/64bit) | ✅ 完整 | 一致 |
| CPU_OFF | ✅ | 一致 |
| CPU_SUSPEND | ⚠️ 简化为 WFI | 不支持 powerdown、无 affinity level |
| AFFINITY_INFO | ✅ level 0 | level>0 总返回 ON（不建模 cluster 电源） |
| SYSTEM_OFF | ✅ | 一致 |
| SYSTEM_RESET | ✅ | 一致 |
| SYSTEM_RESET2 | ❌ 不支持 | PSCI 1.1 新增，未实现 |
| PSCI_FEATURES | ✅ | 一致 |
| MIGRATE / MIGRATE_INFO | ❌ NOT_SUPPORTED | 无 Trusted OS 迁移需求 |
| CPU_FREEZE | ❌ 不支持 | PSCI 1.0+ |
| MEM_PROTECT | ❌ 不支持 | PSCI 1.1+ |
| SYSTEM_SUSPEND | ❌ 不支持 | PSCI 1.0+ |
| CPU_DEFAULT_SUSPEND | ❌ 不支持 | PSCI 1.0+ |
| NODE_HW_STATE | ❌ 不支持 | PSCI 1.0+ |
| STAT_RESIDENCY/COUNT | ❌ 不支持 | PSCI 1.0+ |

### 关键差异分析

1. **CPU_SUSPEND 简化**：真实 ATF 实现多级挂起（standby/retention/powerdown），QEMU 统一为 WFI。对 Guest OS 透明（Linux 通过 cpuidle 探测实际支持的级别）。

2. **无 Cluster 电源管理**：AFFINITY_INFO 在 level>0 时总返回 ON，不建模 CPU cluster 的整体 off/on。

3. **版本差距**：报告 1.1 但缺少 SYSTEM_RESET2、SYSTEM_SUSPEND 等 1.0+ 函数。实际上 Linux Guest 不依赖这些高级功能。

---

## 17. 与真实固件 (ATF/TF-A) 对比

| 方面 | QEMU PSCI 实现 | ARM Trusted Firmware (ATF) |
|------|---------------|---------------------------|
| 运行层级 | Hypervisor/QEMU 内部 | EL3 (Secure Monitor) |
| CPU_ON | `cpu_reset()` + 设置寄存器 | 真实硬件上电序列 (PORESET) |
| CPU_OFF | `halted=1` + 不调度 | 写 PSYSR/PPONR 电源寄存器 |
| SUSPEND | WFI | 配置 retention/powerdown + WFI/WFE |
| 时钟/电压 | 无建模 | 真实 DVFS/PLL 操作 |
| cache flush | 无需（模拟不建模 cache） | 必须 flush L1/L2 再断电 |
| 安全性 | 无隔离（直接操作） | EL3 隔离 + SMC 入口验证 |
| Warm boot | 直接设 PC | 配置 RVBAR/warm boot 入口 |
| Platform specifics | 通用 | 每个 SoC 有 platform port |

### QEMU 的简化是合理的

QEMU 的 PSCI 实现足以支持 Linux、FreeBSD 等标准 Guest 的多核启动。高级电源管理功能（如 cpuidle deep states）在虚拟化环境下本身就没有意义——虚拟 CPU 的"省电"由宿主调度器决定，而非 Guest。

---

## 附录 A：PSCI 函数号速查表

```c
// target/arm/kvm-consts.h
#define QEMU_PSCI_0_2_FN_BASE        0x84000000
#define QEMU_PSCI_0_2_FN64_BASE      0xC4000000

// SMC32 (32-bit) 函数号
PSCI_VERSION        = 0x84000000
CPU_SUSPEND         = 0x84000001
CPU_OFF             = 0x84000002
CPU_ON              = 0x84000003
AFFINITY_INFO       = 0x84000004
MIGRATE             = 0x84000005
MIGRATE_INFO_TYPE   = 0x84000006
SYSTEM_OFF          = 0x84000008
SYSTEM_RESET        = 0x84000009
PSCI_FEATURES       = 0x8400000A

// SMC64 (64-bit) 函数号
CPU_SUSPEND_64      = 0xC4000001
CPU_ON_64           = 0xC4000003
AFFINITY_INFO_64    = 0xC4000004
```

## 附录 B：完整调用链

```
Guest: SMC #0 (x0=0x84000003, x1=mpidr, x2=entry, x3=ctx_id)
  │
  ├── [TCG] translate → EXCP_SMC
  │     → op_helper.c: HELPER(pre_smc)()
  │       → arm_is_psci_call(cpu, EXCP_SMC) == true
  │       → arm_handle_psci_call(cpu)
  │         → switch(0x84000003) → CPU_ON
  │         → arm_set_cpu_on(mpidr, entry, ctx_id, target_el, aa64)
  │           → arm_get_cpu_by_id(mpidr)
  │           → async_run_on_cpu(target, arm_set_cpu_on_async_work)
  │               [在目标 CPU 线程中]:
  │               → cpu_reset(target)
  │               → arm_emulate_firmware_reset(target, el)
  │               → target->halted = 0
  │               → cpu_set_pc(target, entry)
  │               → target->xregs[0] = ctx_id
  │               → power_state = PSCI_ON
  │         → return SUCCESS (写入调用方 x0)
  │
  └── [KVM] KVM 内核直接处理 SMC
        → 唤醒目标 vCPU (KVM_ARM_VCPU_POWER_OFF 清除)
        → 设置目标 vCPU 寄存器
```
