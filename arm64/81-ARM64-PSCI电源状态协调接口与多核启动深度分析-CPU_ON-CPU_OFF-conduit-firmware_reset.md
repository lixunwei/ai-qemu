# ARM64 PSCI 电源状态协调接口与多核启动深度分析

> 文档编号：81  
> 分析目标：PSCI (Power State Coordination Interface) 在 QEMU 中的完整实现  
> 源码版本：QEMU 11.0.50  
> 规范参考：ARM DEN 0022D.b (PSCI Spec)、ARM DEN 0028 (SMCCC)

---

## 一、PSCI 概述

PSCI 是 ARM 定义的固件接口标准，允许 OS 内核通过 SMC/HVC 指令管理 CPU 核心的电源状态。
在真实硬件上，PSCI 由 Trusted Firmware (TF-A/BL31) 实现；在 QEMU 中，PSCI 由模拟器
自身充当"虚假固件"直接处理。

### 1.1 PSCI 版本演进

| 版本 | 宏定义 | 说明 |
|------|--------|------|
| 0.1 | `QEMU_PSCI_VERSION_0_1` (0x00001) | 实现定义的函数号 |
| 0.2 | `QEMU_PSCI_VERSION_0_2` (0x00002) | 标准化函数号，增加 SYSTEM_OFF/RESET |
| 1.0 | `QEMU_PSCI_VERSION_1_0` (0x10000) | 增加 PSCI_FEATURES 查询 |
| 1.1 | `QEMU_PSCI_VERSION_1_1` (0x10001) | QEMU 当前报告的版本 |

QEMU 在 `arm_handle_psci_call()` 中对 `PSCI_VERSION` 查询返回 **1.1**。

---

## 二、调用通道（Conduit）配置

### 2.1 三种通道模式

```c
// target/arm/cpu.h
typedef enum {
    QEMU_PSCI_CONDUIT_DISABLED = 0,  // 不拦截，按架构定义处理
    QEMU_PSCI_CONDUIT_SMC = 1,       // 通过SMC指令调用
    QEMU_PSCI_CONDUIT_HVC = 2,       // 通过HVC指令调用
} QEMUPSCIConduit;
```

### 2.2 virt 机型的通道选择逻辑

```c
// hw/arm/virt.c:2676
if (vms->secure && firmware_loaded) {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_DISABLED;  // 真固件处理
} else if (vms->virt) {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_SMC;  // 有EL2时用SMC
} else {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_HVC;  // 默认用HVC
}
```

**决策逻辑**：
- 加载了真实固件（如 TF-A）→ 禁用 QEMU PSCI，由固件自己处理
- Guest 有 EL2（虚拟化扩展）→ 使用 SMC（陷入 EL3）
- 默认（仅 EL1）→ 使用 HVC（陷入 EL2，QEMU 模拟）
- KVM 模式 → 必须使用 HVC（KVM 在 EL2 运行）

### 2.3 DT 节点生成

```c
// hw/arm/boot.c:366 - fdt_add_psci_node()
qemu_fdt_add_subnode(fdt, "/psci");
// compatible = "arm,psci-1.0", "arm,psci-0.2", "arm,psci"
qemu_fdt_setprop_string(fdt, "/psci", "method", "hvc" 或 "smc");
qemu_fdt_setprop_cell(fdt, "/psci", "cpu_on", QEMU_PSCI_0_2_FN64_CPU_ON);
```

Guest Linux 通过 `/psci` DT 节点发现 PSCI 版本和调用方法。

---

## 三、多核启动流程

### 3.1 启动时序总览

```
Machine初始化
    │
    ├─ 创建所有 CPU 对象 (arm_cpu_initfn)
    │     └─ mp_affinity = arm_build_mp_affinity(idx, clustersz)
    │
    ├─ arm_load_kernel() → 配置 PSCI
    │     ├─ 主CPU (primary): start_powered_off = false
    │     └─ 从CPU (secondary): start_powered_off = true
    │
    ├─ cpu_reset() 对所有CPU调用
    │     └─ power_state = start_powered_off ? PSCI_OFF : PSCI_ON
    │
    └─ 开始执行
          ├─ 主CPU: 从 entry point 开始执行
          └─ 从CPU: halted=1, 等待 PSCI CPU_ON
```

### 3.2 MPIDR 亲和性编码

```c
// target/arm/cpu.c:1160
uint64_t arm_build_mp_affinity(int idx, uint8_t clustersz)
{
    uint32_t Aff1 = idx / clustersz;  // Cluster ID
    uint32_t Aff0 = idx % clustersz;  // Core ID within cluster
    return (Aff1 << ARM_AFF1_SHIFT) | Aff0;
}
```

**virt 机型示例**（clustersz=8）：
- CPU0: MPIDR = 0x00000000 (Cluster0, Core0)
- CPU1: MPIDR = 0x00000001 (Cluster0, Core1)
- CPU8: MPIDR = 0x00000100 (Cluster1, Core0)

### 3.3 从 CPU 的 powered-off 状态

```c
// hw/arm/boot.c:1257
if (ARM_CPU(cs) != info->primary_cpu) {
    object_property_set_bool(cpuobj, "start-powered-off", true, &error_abort);
}

// cpu_reset() 中的效果：
cpu->power_state = cs->start_powered_off ? PSCI_OFF : PSCI_ON;
// cs->halted = 1 (由 CPUState 框架设置)
```

处于 PSCI_OFF 状态的 CPU：
- `halted = 1`：不参与 TCG 执行调度
- `exception_index = EXCP_HLT`：标记为暂停原因

---

## 四、PSCI 调用分发机制

### 4.1 拦截点

PSCI 调用在异常处理路径中被拦截：

```
Guest执行 HVC/SMC
    │
    ├─ 翻译时: trans_HVC/trans_SMC → 生成 EXCP_HVC/EXCP_SMC
    │
    ├─ pre_hvc/pre_smc helper: 检查是否为 PSCI 调用
    │     └─ arm_is_psci_call() → 判断 conduit 匹配
    │
    └─ arm_cpu_do_interrupt() [target/arm/helper.c:9488]
          ├─ if (arm_is_psci_call(cpu, exception_index))
          │     └─ arm_handle_psci_call(cpu)  ← 处理 PSCI
          │           └─ return (不进入正常异常流程)
          └─ else → 正常 HVC/SMC 异常处理
```

### 4.2 conduit 匹配判断

```c
// target/arm/tcg/psci.c:30
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

**关键设计**：只要 conduit 匹配就认为是 PSCI 调用，不再检查函数号。
未知的函数号在 `arm_handle_psci_call()` 中返回 `NOT_SUPPORTED`。

---

## 五、核心 PSCI 操作实现

### 5.1 CPU_ON — 启动从核

```c
// target/arm/tcg/psci.c:133
case QEMU_PSCI_0_2_FN_CPU_ON:
case QEMU_PSCI_0_2_FN64_CPU_ON:
{
    int target_el = arm_feature(env, ARM_FEATURE_EL2) ? 2 : 1;
    bool target_aarch64 = arm_el_is_aa64(env, target_el);
    
    mpidr = param[1];      // 目标 CPU 的 MPIDR
    entry = param[2];      // 启动入口地址
    context_id = param[3]; // 传给目标 CPU 的 x0 值
    
    ret = arm_set_cpu_on(mpidr, entry, context_id, target_el, target_aarch64);
}
```

**目标 EL 选择规则**：
- 有 EL2 → 从 EL2 启动（QEMU 模拟 EL3 固件行为）
- 仅 EL1 → 从 EL1 启动

### 5.2 arm_set_cpu_on 实现

```c
// target/arm/arm-powerctl.c:84
int arm_set_cpu_on(uint64_t cpuid, uint64_t entry, uint64_t context_id,
                   uint32_t target_el, bool target_aa64)
{
    // 1. 通过 MPIDR 查找目标 CPU
    target_cpu_state = arm_get_cpu_by_id(cpuid);
    
    // 2. 检查电源状态
    if (target_cpu->power_state == PSCI_ON) return ALREADY_ON;
    if (target_cpu->power_state == PSCI_ON_PENDING) return ON_PENDING;
    
    // 3. 验证参数
    if (target_aa64 && (entry & 3)) return INVALID_PARAM;  // 4字节对齐
    
    // 4. 异步执行上电（在目标 CPU 上下文中）
    info = g_new(struct CpuOnInfo, 1);
    info->entry = entry;
    info->context_id = context_id;
    info->target_el = target_el;
    info->target_aa64 = target_aa64;
    
    async_run_on_cpu(target_cpu_state, arm_set_cpu_on_async_work,
                     RUN_ON_CPU_HOST_PTR(info));
}
```

### 5.3 异步上电工作

```c
// target/arm/arm-powerctl.c:49
static void arm_set_cpu_on_async_work(CPUState *target_cpu_state, ...)
{
    // 1. 完整 CPU 复位
    cpu_reset(target_cpu_state);
    
    // 2. 模拟固件初始化（设置 SCR_EL3/HCR_EL2 等）
    arm_emulate_firmware_reset(target_cpu_state, info->target_el);
    
    // 3. 取消暂停
    target_cpu_state->halted = 0;
    
    // 4. 设置 context_id → x0/r0
    target_cpu->env.xregs[0] = info->context_id;
    
    // 5. 重建 hflags（TCG 模式）
    arm_rebuild_hflags(&target_cpu->env);
    
    // 6. 设置 PC 为入口地址
    cpu_set_pc(target_cpu_state, info->entry);
    
    // 7. 更新电源状态
    target_cpu->power_state = PSCI_ON;
}
```

### 5.4 arm_emulate_firmware_reset — 固件模拟

此函数模拟真实固件（如 TF-A）在将控制权交给 OS 前所做的寄存器配置：

```c
// target/arm/cpu.c:661
void arm_emulate_firmware_reset(CPUState *cpustate, int target_el)
{
    // 如果目标是最高 EL，cpu_reset 已完成初始化
    if (target_el == 3 && have_el3) return;
    if (target_el == 2 && !have_el3) return;
    if (target_el == 1 && !have_el3 && !have_el2) return;

    if (have_el3) {
        // 设置 SCR_EL3：允许低 EL 使用 AArch64
        env->cp15.scr_el3 |= SCR_RW;
        
        // 启用各种特性的 EL3 trap 旁路
        if (has_pauth) env->cp15.scr_el3 |= SCR_API | SCR_APK;
        if (has_mte)   env->cp15.scr_el3 |= SCR_ATA;
        if (has_sve)   env->cp15.cptr_el[3] |= R_CPTR_EL3_EZ_MASK;
        if (has_sme)   env->cp15.cptr_el[3] |= R_CPTR_EL3_ESM_MASK;
        if (has_fgt)   env->cp15.scr_el3 |= SCR_FGTEN;
        // ... 更多特性

        // 置为非安全状态
        env->cp15.scr_el3 |= SCR_NS;
        // 允许 NS 访问 FPU
        env->cp15.nsacr |= 3 << 10;
    }

    if (have_el2 && target_el < 2) {
        // 允许 EL1 使用 AArch64
        env->cp15.hcr_el2 |= HCR_RW;
    }

    // 设置 PSTATE 到目标 EL
    env->pstate = aarch64_pstate_mode(target_el, true);
}
```

**这是 QEMU 最关键的"假固件"逻辑**：确保 Guest 看到的初始状态与从真实固件启动一致。

---

## 六、CPU_OFF / CPU_SUSPEND / SYSTEM_RESET

### 6.1 CPU_OFF

```c
// target/arm/arm-powerctl.c:236
static void arm_set_cpu_off_async_work(CPUState *target_cpu_state, ...)
{
    target_cpu->power_state = PSCI_OFF;
    target_cpu_state->halted = 1;
    target_cpu_state->exception_index = EXCP_HLT;
}
```

关闭后 CPU 完全退出调度，直到收到 CPU_ON 才能恢复。

### 6.2 CPU_SUSPEND

```c
// target/arm/tcg/psci.c:163
case QEMU_PSCI_0_2_FN_CPU_SUSPEND:
    // QEMU 不支持真正的 powerdown，总是退化为 WFI
    if (param[1] & 0xfffe0000) {
        ret = QEMU_PSCI_RET_INVALID_PARAMS;  // affinity level 不支持
        break;
    }
    env->xregs[0] = 0;  // 返回值预设
    helper_wfi(env, 4);  // 执行 WFI（等待中断）
```

**与硬件差异**：真实硬件可进入深度低功耗状态（power-down），QEMU 仅模拟为 WFI。

### 6.3 SYSTEM_RESET / SYSTEM_OFF

```c
case QEMU_PSCI_0_2_FN_SYSTEM_RESET:
    qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
    goto cpu_off;  // 立即关闭当前 CPU，不返回

case QEMU_PSCI_0_2_FN_SYSTEM_OFF:
    qemu_system_shutdown_request(SHUTDOWN_CAUSE_GUEST_SHUTDOWN);
    goto cpu_off;
```

---

## 七、电源状态机

```
                  CPU_ON(成功)
    ┌─────────────────────────────────┐
    │                                 ▼
 ┌──┴───┐     async_run_on_cpu   ┌────────────┐
 │ OFF  │ ──────────────────────▶│ ON_PENDING │
 └──────┘                        └─────┬──────┘
    ▲                                  │ async work 完成
    │  CPU_OFF                         ▼
    │                             ┌────────┐
    └─────────────────────────────┤   ON   │
                                  └────────┘
                                       │
                                       │ CPU_SUSPEND
                                       ▼
                                  ┌────────┐
                                  │  WFI   │ (QEMU不区分suspend和idle)
                                  └────────┘
```

### 7.1 状态定义

```c
typedef enum ARMPSCIState {
    PSCI_ON = 0,           // CPU 正在运行
    PSCI_OFF = 1,          // CPU 已关闭
    PSCI_ON_PENDING = 2    // CPU 正在启动中（防竞态）
} ARMPSCIState;
```

### 7.2 竞态保护

`arm_set_cpu_on()` 要求 BQL（Big QEMU Lock）已持有，且检查 ON_PENDING 状态防止重复启动：
```c
assert(bql_locked());
if (target_cpu->power_state == PSCI_ON_PENDING) {
    return QEMU_ARM_POWERCTL_ON_PENDING;  // 对应 PSCI_RET_ON_PENDING (-5)
}
```

---

## 八、KVM 模式下的 PSCI

### 8.1 KVM PSCI 配置

```c
// target/arm/kvm.c:1978
if (cpu->psci_version != QEMU_PSCI_VERSION_0_1 &&
    kvm_check_extension(cs->kvm_state, KVM_CAP_ARM_PSCI_0_2)) {
    cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_PSCI_0_2;
}

// 设置 PSCI 版本
kvm_set_one_reg(cs, KVM_REG_ARM_PSCI_VERSION, &psciver);
```

### 8.2 KVM vs TCG 的 PSCI 差异

| 方面 | TCG 模式 | KVM 模式 |
|------|----------|----------|
| PSCI 处理者 | QEMU 用户态代码 | KVM 内核 (host EL2) |
| 调用通道 | HVC 或 SMC | 通常 HVC |
| CPU_ON 机制 | `async_run_on_cpu` + reset | KVM `KVM_ARM_VCPU_POWER_OFF` → `PSCI_CPU_ON` |
| 电源状态同步 | 直接修改 `power_state` | 通过 `KVM_SET_MP_STATE` 同步 |
| CPU_SUSPEND | 退化为 WFI | KVM 可能真正 halt vCPU 线程 |

```c
// KVM 电源状态同步：target/arm/kvm.c:1136
.mp_state = (cpu->power_state == PSCI_OFF) ?
    KVM_MP_STATE_STOPPED : KVM_MP_STATE_RUNNABLE;
```

---

## 九、AFFINITY_INFO 查询

```c
case QEMU_PSCI_0_2_FN_AFFINITY_INFO:
    mpidr = param[1];
    switch (param[2]) {  // affinity_level
    case 0:  // CPU 级别
        target_cpu_state = arm_get_cpu_by_id(mpidr);
        ret = target_cpu->power_state;  // 直接返回状态值
        break;
    default:
        ret = 0;  // 更高级别总是 ON
    }
```

Guest 通过此调用轮询从核是否已上线。

---

## 十、PSCI_FEATURES 特性发现

```c
case QEMU_PSCI_1_0_FN_PSCI_FEATURES:
    switch (param[1]) {
    case QEMU_PSCI_0_2_FN_PSCI_VERSION:
    case QEMU_PSCI_0_2_FN_CPU_ON:
    case QEMU_PSCI_0_2_FN_CPU_OFF:
    case QEMU_PSCI_0_2_FN_CPU_SUSPEND:
    case QEMU_PSCI_0_2_FN_SYSTEM_RESET:
    case QEMU_PSCI_0_2_FN_SYSTEM_OFF:
    // ... 其他已实现功能
        ret = 0;  // 支持
        break;
    default:
        ret = QEMU_PSCI_RET_NOT_SUPPORTED;
    }
```

QEMU 报告支持核心 PSCI 功能但不支持 MIGRATE（无 Trusted OS）。

---

## 十一、与 ARM 规范对比

### 11.1 QEMU 实现符合规范之处

| 规范要求 | QEMU 实现 |
|----------|-----------|
| CPU_ON 入口须 4 字节对齐 | ✅ `if (target_aa64 && (entry & 3))` 检查 |
| CPU_ON 返回错误码 (-4/-5) | ✅ ALREADY_ON / ON_PENDING |
| SYSTEM_RESET 不返回 | ✅ 调用后立即 cpu_off |
| CPU_OFF 不可自行恢复 | ✅ 必须由其他 CPU 调用 CPU_ON |
| PSCI_VERSION 返回正确格式 | ✅ major<<16 \| minor |
| AFFINITY_INFO 返回标准值 | ✅ 0=ON, 1=OFF, 2=ON_PENDING |

### 11.2 QEMU 简化/差异

| 规范要求 | QEMU 行为 | 差异说明 |
|----------|-----------|----------|
| CPU_SUSPEND 可 powerdown | 始终退化为 WFI | 无功耗模拟需求 |
| MIGRATE 迁移 Trusted OS | 返回 NOT_SUPPORTED | 无 Trusted OS |
| OS-initiated suspend mode | 不支持 | 仅平台协调模式 |
| SYSTEM_RESET2 (warm reset) | 不支持 | QEMU 未实现 |
| SYSTEM_OFF2 (hibernate) | 不支持 | PSCI 1.3 新增 |
| CPU_FREEZE | 不支持 | 低功耗状态无意义 |
| STAT_RESIDENCY/COUNT | 不支持 | 无电源域统计 |
| affinity level > 0 查询 | 总是返回 ON | 无 cluster power domain |
| AArch64 CPU 以 AArch32 启动 | 不支持 (TODO) | 代码中有注释 |

### 11.3 函数号编码

```
SMC32/HVC32: 0x84000000 + fn   (32-bit 调用)
SMC64/HVC64: 0xC4000000 + fn   (64-bit 调用)

例：CPU_ON (64-bit) = 0xC4000003
    CPU_OFF         = 0x84000002
    SYSTEM_RESET    = 0x84000009
```

---

## 十二、多核启动完整流程时序图

```
┌─────────┐        ┌─────────┐         ┌──────────────┐
│ Primary │        │Secondary│         │  QEMU Core   │
│  CPU    │        │  CPU    │         │              │
└────┬────┘        └────┬────┘         └──────┬───────┘
     │                  │                      │
     │ ← cpu_reset →   │ ← cpu_reset →        │
     │ power_state=ON   │ power_state=OFF      │
     │ halted=0         │ halted=1             │
     │                  │                      │
     │ 开始执行内核     │ (不参与调度)          │
     │ ...              │                      │
     │                  │                      │
     │ HVC #0 (CPU_ON, │                      │
     │   mpidr=1,      │                      │
     │   entry=0x...,  │                      │
     │   ctx=0)        │                      │
     │─────────────────────────────────────────▶│
     │                  │         arm_is_psci_call()=true
     │                  │         arm_handle_psci_call()
     │                  │              │
     │                  │              │ arm_set_cpu_on()
     │                  │              │   查找CPU by MPIDR
     │                  │              │   检查power_state
     │                  │              │   async_run_on_cpu()
     │                  │              │
     │ x0=0 (SUCCESS)  │              │
     │◀─────────────────────────────────────────│
     │                  │              │
     │                  │◀─────────────│ (目标CPU上下文)
     │                  │ cpu_reset()  │
     │                  │ arm_emulate_firmware_reset()
     │                  │ halted=0     │
     │                  │ x0=context_id│
     │                  │ PC=entry     │
     │                  │ power_state=ON
     │                  │              │
     │                  │ 开始执行     │
     │                  │ (从 entry)   │
     ▼                  ▼              ▼
```

---

## 十三、源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `target/arm/tcg/psci.c` | 226 | PSCI 调用分发与函数号处理 |
| `target/arm/arm-powerctl.c` | 311 | 电源控制底层实现（ON/OFF/RESET） |
| `target/arm/arm-powerctl.h` | 93 | API 声明与返回值定义 |
| `target/arm/kvm-consts.h` | ~110 | PSCI 常量定义（与 KVM 对齐） |
| `target/arm/cpu.c:661` | ~120 | `arm_emulate_firmware_reset()` 固件模拟 |
| `target/arm/cpu.c:329` | 1 | `power_state` 初始化 |
| `target/arm/helper.c:9488` | 5 | PSCI 拦截入口点 |
| `hw/arm/boot.c:366` | ~60 | FDT PSCI 节点生成 |
| `hw/arm/boot.c:1229` | ~70 | 多核 PSCI 配置与启动 |
| `hw/arm/virt.c:2676` | ~10 | virt 机型 conduit 选择 |
| `target/arm/kvm.c:1978` | ~50 | KVM PSCI 初始化 |
| `linux-headers/linux/psci.h` | 142 | Linux 内核 PSCI 标准定义 |
| `target/arm/multiprocessing.h` | 15 | MPIDR/亲和性接口 |

---

## 十四、总结

QEMU 的 PSCI 实现是一个**精简但功能完备的固件仿真层**：

1. **充当虚假固件**：QEMU 在 TCG 模式下扮演 EL3/EL2 固件角色，直接拦截 SMC/HVC
2. **核心功能齐全**：CPU_ON/OFF/SUSPEND、SYSTEM_RESET/OFF、AFFINITY_INFO、FEATURES
3. **异步安全**：通过 `async_run_on_cpu()` 确保 CPU 上电在目标上下文中执行
4. **KVM 兼容**：通过 `KVM_SET_MP_STATE` 与内核同步电源状态
5. **合理简化**：省略低功耗状态、电源域层级等对模拟无意义的功能
6. **Linux 兼容**：生成标准 DT 节点，Linux PSCI 驱动可直接使用
