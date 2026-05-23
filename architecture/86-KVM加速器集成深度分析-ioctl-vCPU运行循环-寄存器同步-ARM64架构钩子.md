# KVM 加速器集成深度分析

> 文档编号：86  
> 分析目标：KVM 加速器初始化、vCPU 运行循环、寄存器同步、ARM64 特有处理  
> 源码版本：QEMU 11.0.50  
> 核心文件：accel/kvm/kvm-all.c (4786行)、target/arm/kvm.c (2617行)

---

## 一、概述

KVM（Kernel-based Virtual Machine）是 QEMU 的硬件加速后端。当启用 KVM 时，guest 指令直接在物理 CPU 上执行，只有特权操作（MMIO、中断注入等）才会退出到 QEMU 用户态处理。

```
┌─────────────────────────────────┐
│          Guest (Linux)          │
│  直接在物理 CPU 执行            │
├─────────────────────────────────┤
│       KVM (内核模块)            │
│  拦截特权操作 → VM Exit        │
├─────────────────────────────────┤
│       QEMU 用户态               │
│  处理 MMIO / 中断 / 设备模拟   │
└─────────────────────────────────┘
```

---

## 二、初始化流程

### 2.1 kvm_init() — KVM 全局初始化

```c
// accel/kvm/kvm-all.c:2895
static int kvm_init(AccelState *as, MachineState *ms) {
    // 1. 打开 /dev/kvm
    s->fd = qemu_open_old("/dev/kvm", O_RDWR);
    
    // 2. 检查 API 版本
    ret = kvm_ioctl(s, KVM_GET_API_VERSION, 0);
    
    // 3. 创建 VM
    type = find_kvm_machine_type(ms);  // ARM: IPA 位数编码
    ret = do_kvm_create_vm(s, type);   // KVM_CREATE_VM
    s->vmfd = ret;
    
    // 4. 查询能力
    s->nr_slots_max = kvm_check_extension(s, KVM_CAP_NR_MEMSLOTS);
    kvm_immediate_exit = kvm_check_extension(s, KVM_CAP_IMMEDIATE_EXIT);
    
    // 5. 初始化内存槽
    kvm_memory_listener_register(s, ...);
    
    // 6. 信号处理（用于 vCPU 退出通知）
}
```

### 2.2 kvm_init_vcpu() — vCPU 创建

```c
// accel/kvm/kvm-all.c:712
int kvm_init_vcpu(CPUState *cpu) {
    // 1. KVM_CREATE_VCPU
    ret = kvm_vm_ioctl(s, KVM_CREATE_VCPU, kvm_arch_vcpu_id(cpu));
    cpu->kvm_fd = ret;
    
    // 2. mmap kvm_run 共享区域
    cpu->kvm_run = mmap(..., vcpu_mmap_size, PROT_READ|PROT_WRITE, MAP_SHARED, cpu->kvm_fd, 0);
    
    // 3. 架构特定初始化
    ret = kvm_arch_init_vcpu(cpu);  // → target/arm/kvm.c
}
```

### 2.3 kvm_arch_init_vcpu() — ARM64 vCPU 初始化

```c
// target/arm/kvm.c:1958
int kvm_arch_init_vcpu(CPUState *cs) {
    // 1. 设置 init features
    cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_POWER_OFF;  // powered-off
    cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_PSCI_0_2;   // PSCI v0.2+
    cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_PMU_V3;     // PMUv3
    cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_SVE;         // SVE
    cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_PTRAUTH_*;  // PAC
    cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_HAS_EL2;    // 嵌套虚拟化
    
    // 2. KVM_ARM_VCPU_INIT
    ret = kvm_arm_vcpu_init(cpu);
    
    // 3. SVE 向量长度配置
    if (aa64_sve) {
        kvm_arm_sve_set_vls(cpu);          // 设置允许的 VL 列表
        kvm_arm_vcpu_finalize(cpu, KVM_ARM_VCPU_SVE);
    }
    
    // 4. PSCI 版本协商
    kvm_set_one_reg(cs, KVM_REG_ARM_PSCI_VERSION, &psciver);
    kvm_get_one_reg(cs, KVM_REG_ARM_PSCI_VERSION, &psciver);
    
    // 5. 获取 MPIDR（KVM 自行分配）
    kvm_get_one_reg(cs, ARM64_SYS_REG(MPIDR), &mpidr);
    cpu->mp_affinity = mpidr & ARM64_AFFINITY_MASK;
    
    // 6. 构建 cpreg 列表（用于迁移）
    return kvm_arm_init_cpreg_list(cpu);
}
```

---

## 三、vCPU 运行循环

### 3.1 kvm_cpu_exec() — 核心运行循环

```c
// accel/kvm/kvm-all.c:3439
int kvm_cpu_exec(CPUState *cpu) {
    struct kvm_run *run = cpu->kvm_run;
    
    // 处理异步事件（如中断注入请求）
    if (kvm_arch_process_async_events(cpu)) return EXCP_HLT;
    
    bql_unlock();       // 释放大内核锁
    cpu_exec_start(cpu);
    
    do {
        // 1. 同步脏寄存器到 KVM
        if (cpu->vcpu_dirty) {
            kvm_cpu_synchronize_put(cpu, KVM_PUT_RUNTIME_STATE);
        }
        
        // 2. 架构特定预处理
        kvm_arch_pre_run(cpu, run);
        
        // 3. 检查退出请求
        if (qatomic_load_acquire(&cpu->exit_request)) {
            kvm_cpu_kick_self();  // 发信号确保快速退出
        }
        
        // 4. ★ 进入 KVM — 执行 guest 代码 ★
        run_ret = kvm_vcpu_ioctl(cpu, KVM_RUN, 0);
        
        // 5. 架构特定后处理
        attrs = kvm_arch_post_run(cpu, run);
        
        // 6. 处理退出原因
        switch (run->exit_reason) {
            case KVM_EXIT_IO:       → kvm_handle_io()
            case KVM_EXIT_MMIO:     → address_space_rw()
            case KVM_EXIT_SYSTEM_EVENT: → shutdown/reset/crash
            case KVM_EXIT_DEBUG:    → kvm_arch_handle_exit()
            case KVM_EXIT_DIRTY_RING_FULL: → 脏页环处理
            case KVM_EXIT_MEMORY_FAULT:    → 内存故障处理
            default:                → kvm_arch_handle_exit()
        }
    } while (ret == 0);   // ret=0 表示已处理，重入 guest
    
    cpu_exec_end(cpu);
    bql_lock();
    return ret;
}
```

### 3.2 退出原因处理

| 退出原因 | ARM64 场景 | QEMU 处理 |
|----------|------------|-----------|
| `KVM_EXIT_MMIO` | Guest 访问设备 MMIO 地址 | `address_space_rw()` 分发到设备模型 |
| `KVM_EXIT_IO` | ARM64 不使用（x86 PIO） | — |
| `KVM_EXIT_SYSTEM_EVENT` | PSCI SYSTEM_OFF/RESET | shutdown/reset 请求 |
| `KVM_EXIT_DEBUG` | 硬件断点/观察点触发 | 通知 GDB |
| `KVM_EXIT_ARM_NISV` | 外部数据中止（无有效 ISS） | 注入异常给 guest |
| `KVM_EXIT_DIRTY_RING_FULL` | 脏页环满（热迁移时） | 收割脏页、限流 |
| `KVM_EXIT_MEMORY_FAULT` | 私有内存故障（CoCo） | 内存转换 |

---

## 四、ARM64 架构钩子

### 4.1 kvm_arch_pre_run()

```c
// target/arm/kvm.c:1333
void kvm_arch_pre_run(CPUState *cs, struct kvm_run *run) {
    // 验证上一次注入的外部数据中止是否成功
    if (unlikely(env->ext_dabt_raised)) {
        if (!kvm_arm_verify_ext_dabt_pending(cpu)) {
            error_report("Failed to inject external data abort");
            abort();
        }
        env->ext_dabt_raised = 0;
    }
}
```

### 4.2 kvm_arch_post_run()

```c
// target/arm/kvm.c:1360
MemTxAttrs kvm_arch_post_run(CPUState *cs, struct kvm_run *run) {
    // 如果使用用户态中断控制器（非 kernel irqchip）
    // 同步 KVM 内部的定时器/PMU IRQ 状态到 QEMU
    
    if (run->s.regs.device_irq_level != cpu->device_irq_level) {
        // 虚拟定时器 IRQ 变化
        if (switched & KVM_ARM_DEV_EL1_VTIMER) {
            qemu_set_irq(cpu->gt_timer_outputs[GTIMER_VIRT], level);
        }
        // 物理定时器 IRQ 变化
        if (switched & KVM_ARM_DEV_EL1_PTIMER) {
            qemu_set_irq(cpu->gt_timer_outputs[GTIMER_PHYS], level);
        }
        // PMU IRQ 变化
        if (switched & KVM_ARM_DEV_PMU) {
            qemu_set_irq(cpu->pmu_interrupt, level);
        }
    }
}
```

### 4.3 kvm_arch_handle_exit()

```c
// target/arm/kvm.c:1546
int kvm_arch_handle_exit(CPUState *cs, struct kvm_run *run) {
    switch (run->exit_reason) {
    case KVM_EXIT_DEBUG:
        // 硬件断点/观察点
        if (kvm_arm_handle_debug(cpu, &run->debug.arch))
            return EXCP_DEBUG;
        break;
    case KVM_EXIT_ARM_NISV:
        // 无有效 ISS 的数据中止（无法解码故障指令）
        return kvm_arm_handle_dabt_nisv(cpu, esr_iss, fault_ipa);
    }
}
```

---

## 五、寄存器同步

### 5.1 put_registers — QEMU → KVM

```c
// target/arm/kvm.c:2157
int kvm_arch_put_registers(CPUState *cs, KvmPutState level) {
    // 1. AArch32 → AArch64 转换（KVM 总用 64 位接口）
    if (!is_a64(env)) aarch64_sync_32_to_64(env);
    
    // 2. 通用寄存器 x0-x30
    for (i = 0; i < 31; i++)
        kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.regs[i]), &env->xregs[i]);
    
    // 3. SP_EL0, SP_EL1
    kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.sp), &env->sp_el[0]);
    kvm_set_one_reg(cs, AARCH64_CORE_REG(sp_el1), &env->sp_el[1]);
    
    // 4. PSTATE / CPSR
    kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.pstate), &val);
    
    // 5. PC
    kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.pc), &env->pc);
    
    // 6. ELR_EL1, SPSR[0-4]
    kvm_set_one_reg(cs, AARCH64_CORE_REG(elr_el1), &env->elr_el[1]);
    
    // 7. FP/SIMD 寄存器
    if (aa64_sve) kvm_arch_put_sve(cs, vq, true);  // Z寄存器 + P寄存器
    else kvm_arch_put_fpsimd(cs);                   // V寄存器 (128-bit)
    
    // 8. FPSR/FPCR
    kvm_set_one_reg(cs, AARCH64_SIMD_CTRL_REG(fp_regs.fpsr), &fpsr);
    kvm_set_one_reg(cs, AARCH64_SIMD_CTRL_REG(fp_regs.fpcr), &fpcr);
    
    // 9. 系统寄存器列表（cpreg list）
    write_cpustate_to_list(cpu, true);
    write_list_to_kvmstate(cpu, level);
    
    // 10. VCPU 事件 + MP State
    kvm_put_vcpu_events(cpu);
    kvm_arm_sync_mpstate_to_kvm(cpu);
}
```

### 5.2 get_registers — KVM → QEMU

```c
// target/arm/kvm.c:2342
int kvm_arch_get_registers(CPUState *cs) {
    // 镜像操作：从 KVM 读回所有寄存器到 env
    // 1. x0-x30, SP, PC, PSTATE
    // 2. AArch64 → AArch32 转换（如果 guest 在 AArch32 模式）
    // 3. ELR_EL1, SPSR[0-4]
    // 4. SVE Z/P 寄存器（或 FPSIMD V 寄存器）
    // 5. FPSR/FPCR
    // 6. VCPU events（异常/中断状态）
    // 7. 系统寄存器列表
    write_kvmstate_to_list(cpu);
    write_list_to_cpustate(cpu);
}
```

### 5.3 SVE 寄存器同步

```c
// target/arm/kvm.c:2306
static int kvm_arch_get_sve(CPUState *cs, uint32_t vq, bool have_ffr) {
    // Z 寄存器：KVM_REG_ARM64_SVE_ZREG(n, slice=0)
    for (n = 0; n < 32; n++) {
        kvm_get_one_reg(cs, KVM_REG_ARM64_SVE_ZREG(n, 0), &env->vfp.zregs[n]);
        sve_bswap64(r, r, vq * 2);  // 大端序字节交换
    }
    
    // P 寄存器：KVM_REG_ARM64_SVE_PREG(n, slice=0)
    for (n = 0; n < 16; n++) {
        kvm_get_one_reg(cs, KVM_REG_ARM64_SVE_PREG(n, 0), &env->vfp.pregs[n]);
    }
    
    // FFR：KVM_REG_ARM64_SVE_FFR(slice=0)
    if (have_ffr) {
        kvm_get_one_reg(cs, KVM_REG_ARM64_SVE_FFR(0), &env->vfp.pregs[FFR_PRED_NUM]);
    }
}
```

---

## 六、系统寄存器（cpreg）同步

### 6.1 cpreg 列表

KVM 使用 `KVM_GET_REG_LIST` 获取 vCPU 所有可访问寄存器的 ID 列表。QEMU 在 `kvm_arch_init_vcpu()` 中调用 `kvm_arm_init_cpreg_list()` 构建这个列表。

### 6.2 寄存器 ID 编码

```c
// target/arm/kvm-consts.h:150-190
#define CP_REG_ARM64              0x6000000000000000ULL
#define CP_REG_ARM64_SYSREG      (0x0013 << CP_REG_ARM_COPROC_SHIFT)
#define CP_REG_ARM64_SYSREG_OP0_MASK  0x000000000000c000ULL

// ARM64_SYS_REG(op0, op1, crn, crm, op2) 宏构造 KVM 寄存器 ID
// 示例: SCTLR_EL1 = ARM64_SYS_REG(3, 0, 1, 0, 0)
```

### 6.3 同步时机

| 操作 | 方向 | 时机 |
|------|------|------|
| `kvm_arch_put_registers(FULL)` | QEMU→KVM | vCPU 首次运行、迁移恢复 |
| `kvm_arch_put_registers(RUNTIME)` | QEMU→KVM | 每次 KVM_RUN 前（如脏） |
| `kvm_arch_get_registers()` | KVM→QEMU | GDB 查询、迁移保存、调试 |

---

## 七、VGIC/PMU/Timer 集成

### 7.1 VGIC（虚拟 GIC）

```c
// target/arm/kvm.c:1643
int kvm_arm_vgic_probe(void) {
    // 探测 KVM 支持的 GIC 版本（GICv2/v3）
    // 返回: KVM_DEV_TYPE_ARM_VGIC_V3 或 KVM_DEV_TYPE_ARM_VGIC_V2
}

// kvm_arch_irqchip_create():
// 使用 KVM_CREATE_DEVICE 创建内核态 VGIC 设备
```

### 7.2 PMU

```c
// target/arm/kvm.c:1845
int kvm_arm_pmu_init(CPUState *cs) {
    // 使用 KVM_SET_DEVICE_ATTR 在 vCPU 上启用 PMU
    // 设置 PMU overflow IRQ
}
```

### 7.3 Timer

```c
// kvm_arch_post_run() 中：
// 同步虚拟/物理定时器的 IRQ 电平
// KVM_ARM_DEV_EL1_VTIMER / KVM_ARM_DEV_EL1_PTIMER

// kvm_arm_vm_state_change():
// VM 暂停/恢复时同步虚拟时间偏移
if (running) kvm_arm_put_virtual_time(cpu);
else kvm_arm_get_virtual_time(cpu);
```

---

## 八、调试支持

```c
// target/arm/kvm.c:1615
void kvm_arch_update_guest_debug(CPUState *cs, struct kvm_guest_debug *dbg) {
    // 软件断点
    if (kvm_sw_breakpoints_active(cs))
        dbg->control |= KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_SW_BP;
    
    // 硬件断点/观察点（DBGBCR/DBGBVR/DBGWCR/DBGWVR）
    if (kvm_arm_hw_debug_active(cpu))
        dbg->control |= KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_HW;
        kvm_arm_copy_hw_debug_data(&dbg->arch);
}
```

---

## 九、ioctl 层次

```
QEMU 用户态
    │
    ├─ kvm_ioctl(KVM_GET_API_VERSION, KVM_CREATE_VM, ...)
    │   └─ ioctl(s->fd, ...)         ← /dev/kvm 文件描述符
    │
    ├─ kvm_vm_ioctl(KVM_CREATE_VCPU, KVM_SET_USER_MEMORY_REGION, ...)
    │   └─ ioctl(s->vmfd, ...)       ← VM 文件描述符
    │
    └─ kvm_vcpu_ioctl(KVM_RUN, KVM_GET_ONE_REG, KVM_SET_ONE_REG, ...)
        └─ ioctl(cpu->kvm_fd, ...)   ← vCPU 文件描述符
```

---

## 十、与 TCG 模式对比

| 方面 | KVM 模式 | TCG 模式 |
|------|----------|----------|
| 指令执行 | 物理 CPU 直接执行 | 逐块翻译+执行 |
| 性能 | 接近原生 (~95%) | ~5-20x 慢 |
| 特权操作 | VM Exit 到 QEMU | helper 函数 |
| 中断 | KVM/VGIC 内核态注入 | QEMU 模拟 |
| 内存 | 硬件 Stage-2 MMU | softmmu TLB |
| MMIO | Exit + address_space_rw | MemoryRegion 回调 |
| 寄存器 | KVM 管理，需要时同步 | env 直接访问 |
| 调试 | 硬件断点 + KVM_GUESTDBG | TCG 插桩 |
| 跨架构 | 不支持 | 支持（任意 host→guest） |
| 嵌套虚拟化 | 需要硬件支持 | 完全模拟 |

---

## 十一、关键数据结构

```c
// KVMState — 全局 KVM 状态
struct KVMState {
    int fd;           // /dev/kvm 文件描述符
    int vmfd;         // VM 文件描述符
    int nr_slots_max; // 最大内存槽数
    struct KVMAs *as; // 地址空间
    ...
};

// CPUState — 每 vCPU KVM 状态
struct CPUState {
    int kvm_fd;           // vCPU 文件描述符
    struct kvm_run *kvm_run;  // mmap 共享运行区域
    bool vcpu_dirty;      // 寄存器需要同步到 KVM
    ...
};

// kvm_run — KVM↔QEMU 共享数据
struct kvm_run {
    __u32 exit_reason;    // 退出原因
    union {
        struct { ... } io;     // IO exit 信息
        struct { __u64 phys_addr; __u8 data[8]; __u32 len; __u8 is_write; } mmio;
        struct { ... } debug;
        struct { __u32 type; } system_event;
    };
    struct { __u64 device_irq_level; } s.regs;  // 设备 IRQ 电平
};
```

---

## 十二、源文件索引

| 文件 | 行数 | 关键内容 |
|------|------|----------|
| `accel/kvm/kvm-all.c` | 4786 | 通用 KVM 后端：init、run loop、内存、IRQ 路由 |
| `accel/kvm/kvm-accel-ops.c` | 129 | 加速器 class 注册 |
| `target/arm/kvm.c` | 2617 | ARM64 KVM 集成：vcpu init、寄存器同步、exit 处理 |
| `target/arm/kvm-consts.h` | 193 | KVM 寄存器编码、PSCI 常量 |
| `include/system/kvm.h` | 610 | KVM 公共 API 声明 |
| `target/arm/kvm_arm.h` | ~300 | ARM KVM 内部 API |

| 函数 | 位置 | 职责 |
|------|------|------|
| `kvm_init()` | kvm-all.c:2895 | 全局初始化：打开 /dev/kvm、创建 VM |
| `kvm_init_vcpu()` | kvm-all.c:712 | 创建 vCPU、mmap kvm_run |
| `kvm_cpu_exec()` | kvm-all.c:3439 | ★ 核心运行循环：KVM_RUN + exit 处理 |
| `kvm_arch_init_vcpu()` | kvm.c:1958 | ARM vCPU 初始化：features/SVE/PSCI |
| `kvm_arch_put_registers()` | kvm.c:2157 | QEMU→KVM 寄存器同步 |
| `kvm_arch_get_registers()` | kvm.c:2342 | KVM→QEMU 寄存器同步 |
| `kvm_arch_pre_run()` | kvm.c:1333 | KVM_RUN 前预处理 |
| `kvm_arch_post_run()` | kvm.c:1360 | KVM_RUN 后：定时器/PMU IRQ 同步 |
| `kvm_arch_handle_exit()` | kvm.c:1546 | ARM 特定退出处理（DEBUG/NISV） |
| `kvm_arm_vgic_probe()` | kvm.c:1643 | 探测内核 VGIC 版本 |
| `kvm_arm_pmu_init()` | kvm.c:1845 | PMU 初始化 |
| `kvm_arm_sve_set_vls()` | kvm.c:1942 | SVE 向量长度配置 |
| `kvm_arch_update_guest_debug()` | kvm.c:1615 | 调试断点设置 |
