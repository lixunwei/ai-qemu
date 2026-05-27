# Doc 99: KVM ARM64 特性与 VM Exit 深度分析

## VM Exit 处理 · ARM64 特性协商 · KVM Device API · cpreg 同步 · vCPU 线程模型

> QEMU 11.0.50 · target/arm/kvm.c (2617行) + accel/kvm/kvm-all.c
> 分析日期: 2025-01
> 前置文档: Doc 86 (KVM加速器概览)、Doc 98 (KVM内存管理)

---

## 目录

1. [VM Exit 完整处理流程](#1-vm-exit-完整处理流程)
2. [KVM_EXIT_MMIO 深入分析](#2-kvm_exit_mmio-深入分析)
3. [ARM64 特有退出处理](#3-arm64-特有退出处理)
4. [ARM64 特性/能力协商](#4-arm64-特性能力协商)
5. [SVE/MTE/PAuth/PMU 特性处理](#5-svemtepAuthpmu-特性处理)
6. [KVM Device API](#6-kvm-device-api)
7. [cpreg 系统寄存器管理](#7-cpreg-系统寄存器管理)
8. [Pre/Post Run 钩子](#8-prepost-run-钩子)
9. [vCPU 线程模型](#9-vcpu-线程模型)
10. [调试支持](#10-调试支持)
11. [与硬件对比：EL2 Hypervisor 行为](#11-与硬件对比el2-hypervisor-行为)

---

## 1. VM Exit 完整处理流程

### 1.1 kvm_cpu_exec() 主循环

```c
// accel/kvm/kvm-all.c:3439-3621
int kvm_cpu_exec(CPUState *cpu) {
    struct kvm_run *run = cpu->kvm_run;

    // 1. 预处理异步事件
    if (kvm_arch_process_async_events(cpu)) {
        cpu->exit_request = false;
        return EXCP_HLT;
    }

    // 2. 释放 BQL，允许其他线程运行
    bql_unlock();
    cpu_exec_start(cpu);

    do {
        // 3. 如果寄存器脏，同步到 KVM
        if (cpu->vcpu_dirty) {
            kvm_cpu_synchronize_put(cpu, KVM_PUT_RUNTIME_STATE, &error_abort);
        }

        // 4. 架构 pre-run
        kvm_arch_pre_run(cpu, run);

        // 5. 检查退出请求（SIGBUS 等）
        if (cpu->exit_request) {
            run->immediate_exit = 1;  // 设置后 KVM_RUN 立即返回
        }

        // 6. 进入 KVM
        run_ret = kvm_vcpu_ioctl(cpu, KVM_RUN, 0);

        // 7. 架构 post-run
        attrs = kvm_arch_post_run(cpu, run);

        // 8. 处理退出原因
        switch (run->exit_reason) {
            case KVM_EXIT_MMIO: ...
            case KVM_EXIT_IO: ...
            case KVM_EXIT_IRQ_WINDOW_OPEN: ...
            case KVM_EXIT_SHUTDOWN: ...
            case KVM_EXIT_INTERNAL_ERROR: ...
            case KVM_EXIT_SYSTEM_EVENT: ...
            case KVM_EXIT_DIRTY_RING_FULL: ...
            case KVM_EXIT_DEBUG: ...
            default: kvm_arch_handle_exit(cpu, run);
        }
    } while (ret == 0);

    // 9. 重新获取 BQL
    cpu_exec_end(cpu);
    bql_lock();
    return ret;  // EXCP_* 返回值
}
```

### 1.2 退出原因分类

| 退出原因 | 频率 | 处理方式 |
|---------|------|---------|
| `KVM_EXIT_MMIO` | 高 | QEMU MemoryRegion dispatch |
| `KVM_EXIT_DIRTY_RING_FULL` | 中 | reap + reset ring |
| `KVM_EXIT_DEBUG` | 低 | 断点/watchpoint 处理 |
| `KVM_EXIT_SYSTEM_EVENT` | 极低 | shutdown/reset/crash |
| `KVM_EXIT_INTERNAL_ERROR` | 极低 | emulation failure |
| `KVM_EXIT_ARM_NISV` | 极低 | 外部 DABT 注入 |

---

## 2. KVM_EXIT_MMIO 深入分析

### 2.1 处理路径

```c
// accel/kvm/kvm-all.c:3531-3539
case KVM_EXIT_MMIO:
    address_space_rw(&address_space_memory,
                     run->mmio.phys_addr,   // Guest 物理地址
                     attrs,                  // 内存事务属性
                     run->mmio.data,         // 数据缓冲区
                     run->mmio.len,          // 访问宽度 (1/2/4/8)
                     run->mmio.is_write);    // 读/写方向
    ret = 0;  // 继续运行
    break;
```

### 2.2 MMIO 分发链

```
KVM_EXIT_MMIO (GPA, data, len, is_write)
  → address_space_rw(&address_space_memory, ...)
    → flatview_access(FlatView, GPA)
      → flatview_lookup(GPA) → MemoryRegionSection
      → memory_region_dispatch_write/read(mr, offset, data, size)
        → mr->ops->write(opaque, offset, data, size)
          → 设备处理函数（如 pl011_write, gicv3_dist_write）
```

### 2.3 触发条件

MMIO exit 发生的条件：
1. Guest 访问的 GPA **不在任何 KVM memslot** 中
2. ARM64: Stage-2 Translation Fault（无映射）
3. KVM 检测到是 MMIO 区域 → 填充 `run->mmio` → exit

### 2.4 性能影响

每次 MMIO exit 代价：
- Guest → EL2 trap: ~100 cycles
- KVM exit to userspace: ~1000 cycles
- QEMU dispatch + device: ~500-5000 cycles
- KVM re-enter: ~1000 cycles
- **总计: ~3000-8000 cycles/次**

这就是 ioeventfd 存在的意义 — 对高频 MMIO（如 VirtIO notify），消除 exit。

---

## 3. ARM64 特有退出处理

### 3.1 kvm_arch_handle_exit()

```c
// target/arm/kvm.c:1546-1567
int kvm_arch_handle_exit(CPUState *cs, struct kvm_run *run) {
    switch (run->exit_reason) {
    case KVM_EXIT_DEBUG:
        return kvm_arm_handle_debug(cs, &run->debug.arch);

    case KVM_EXIT_ARM_NISV:
        // Not Instruction Syndrome Valid — 无法解码的 DABT
        return kvm_arm_handle_dabt_nisv(cs, ...);

    default:
        // 未知退出
        error_report("kvm: unexpected exit reason %d", run->exit_reason);
        return -1;
    }
}
```

### 3.2 KVM_EXIT_ARM_NISV

当 ARM64 Data Abort 的 ISS (Instruction Specific Syndrome) 无效时：
- 通常是 guest 执行了 LDM/STM 等复杂指令访问 MMIO
- KVM 无法解码 → 退出到 QEMU
- QEMU 可选择注入外部 DABT 到 guest，或尝试指令模拟

```c
// target/arm/kvm.c (概念)
static int kvm_arm_handle_dabt_nisv(CPUState *cs, uint64_t esr, uint64_t far) {
    if (cap_has_inject_ext_dabt) {
        // 注入 External Data Abort 到 guest
        kvm_arm_inject_ext_dabt(cs, far);
        return EXCP_INTERRUPT;
    }
    return -1;  // 无法处理
}
```

### 3.3 KVM_EXIT_SYSTEM_EVENT

```c
// accel/kvm/kvm-all.c:3577-3600
case KVM_EXIT_SYSTEM_EVENT:
    switch (run->system_event.type) {
    case KVM_SYSTEM_EVENT_SHUTDOWN:
        qemu_system_shutdown_request(SHUTDOWN_CAUSE_GUEST_SHUTDOWN);
        break;
    case KVM_SYSTEM_EVENT_RESET:
        qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
        break;
    case KVM_SYSTEM_EVENT_SEV_TERM:
    case KVM_SYSTEM_EVENT_CRASH:
        cpu_synchronize_state(cpu);
        qemu_system_guest_panicked(cpu_get_crash_info(cpu));
        break;
    default:
        ret = kvm_arch_handle_exit(cpu, run);
    }
```

ARM64 中 SYSTEM_EVENT 来源：
- Guest 执行 `PSCI SYSTEM_OFF` → KVM trap → `KVM_SYSTEM_EVENT_SHUTDOWN`
- Guest 执行 `PSCI SYSTEM_RESET` → KVM trap → `KVM_SYSTEM_EVENT_RESET`

---

## 4. ARM64 特性/能力协商

### 4.1 Host CPU 特性探测

```c
// target/arm/kvm.c:276-481
static void kvm_arm_get_host_cpu_features(ARMHostCPUFeatures *ahcf) {
    int fdarray[3];  // [0]=kvm_fd, [1]=vm_fd, [2]=vcpu_fd
    struct kvm_vcpu_init init = { .target = -1 };

    // 1. 设置要请求的特性
    if (kvm_check_extension(s, KVM_CAP_ARM_SVE))
        init.features[0] |= 1 << KVM_ARM_VCPU_SVE;
    if (kvm_arm_el2_supported())
        init.features[0] |= 1 << KVM_ARM_VCPU_HAS_EL2;
    if (kvm_arm_pauth_supported())
        init.features[0] |= 1 << KVM_ARM_VCPU_PTRAUTH_ADDRESS
                          |  1 << KVM_ARM_VCPU_PTRAUTH_GENERIC;
    if (kvm_check_extension(s, KVM_CAP_ARM_PMU_V3))
        init.features[0] |= 1 << KVM_ARM_VCPU_PMU_V3;

    // 2. 创建临时 VM + vCPU 探测 ID 寄存器
    kvm_arm_create_scratch_host_vcpu(fdarray, &init);

    // 3. 读取 ID 寄存器
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64pfr0, ARM64_SYS_REG(3,0,0,4,0));
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64pfr1, ARM64_SYS_REG(3,0,0,4,1));
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64mmfr0, ...);
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64mmfr1, ...);
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64mmfr2, ...);
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64isar0, ...);
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64isar1, ...);
    read_sys_reg64(vcpu_fd, &ahcf->isar.id_aa64dfr0, ...);
    // ... 更多 ID 寄存器

    // 4. 读取 SVE VLS (vector length set)
    if (sve_supported) {
        ahcf->sve_vls = kvm_arm_sve_get_vls(vcpu_fd);
    }

    // 5. 销毁临时 vCPU
    kvm_arm_destroy_scratch_host_vcpu(fdarray);

    ahcf->target = init.target;
    memcpy(ahcf->features, init.features, sizeof(init.features));
}
```

### 4.2 特性传播到 ARMCPU

```c
// target/arm/kvm.c:483-507
void kvm_arm_set_cpu_features_from_host(ARMCPU *cpu) {
    // 将探测到的 host 特性复制到 guest CPU 模型
    cpu->isar = arm_host_cpu_features.isar;  // 所有 ID 寄存器
    cpu->kvm_target = arm_host_cpu_features.target;
    memcpy(cpu->kvm_init_features, arm_host_cpu_features.features, ...);
}
```

### 4.3 关键能力检查

| 能力 | 含义 | 检查方式 |
|------|------|---------|
| `KVM_CAP_ARM_SVE` | SVE 向量扩展 | `kvm_check_extension()` |
| `KVM_CAP_ARM_MTE` | 内存标记扩展 | `kvm_arm_mte_supported()` |
| `KVM_CAP_ARM_PTRAUTH_ADDRESS` | Pointer Authentication | `kvm_arm_pauth_supported()` |
| `KVM_CAP_ARM_PMU_V3` | 性能监控单元 | `kvm_check_extension()` |
| `KVM_CAP_ARM_EL2` | Nested Virtualization | `kvm_arm_el2_supported()` |
| `KVM_CAP_ARM_INJECT_SERROR_ESR` | SError 注入 | `cap_has_inject_serror_esr` |
| `KVM_CAP_ARM_INJECT_EXT_DABT` | 外部 DABT 注入 | `cap_has_inject_ext_dabt` |

---

## 5. SVE/MTE/PAuth/PMU 特性处理

### 5.1 SVE

```c
// target/arm/kvm.c:247-274
static uint32_t kvm_arm_sve_get_vls(int fd) {
    // 读取 KVM_REG_ARM64_SVE_VLS 寄存器
    uint64_t val;
    ret = ioctl(fd, KVM_GET_ONE_REG, &(struct kvm_one_reg){
        .id = KVM_REG_ARM64_SVE_VLS, .addr = (uint64_t)&val
    });
    // val 是位图：bit N = 1 表示支持 VL = (N+1)*128 bits
    return (uint32_t)val;  // 返回支持的 VQ bitmap
}
```

SVE 寄存器同步：
- 每个 Z/P/FFR 寄存器通过 `KVM_REG_ARM64_SVE` 编码
- 大小取决于当前 VL

### 5.2 MTE

```c
// target/arm/kvm.c:127-139
// MTE 必须在 VCPU 创建前 enable
if (kvm_arm_mte_supported()) {
    ret = kvm_vm_enable_cap(kvm_state, KVM_CAP_ARM_MTE, 0);
    // 这确保 scratch VCPU 的 ID_AA64PFR1_EL1.MTE 字段正确
}
```

MTE 内存标签通过 `KVM_ARM64_MTE_COPY_TAGS` ioctl 在迁移时同步。

### 5.3 PAuth

```c
// target/arm/kvm.c:218-222
static bool kvm_arm_pauth_supported(void) {
    return kvm_check_extension(kvm_state, KVM_CAP_ARM_PTRAUTH_ADDRESS) &&
           kvm_check_extension(kvm_state, KVM_CAP_ARM_PTRAUTH_GENERIC);
}
```

PAuth 密钥寄存器 (APxAKey/APxBKey) 通过 cpreg 框架同步。

### 5.4 PMU

```c
// target/arm/kvm.c:329-333, 438-442
// 请求 PMU 支持:
init.features[0] |= 1 << KVM_ARM_VCPU_PMU_V3;

// 读取 PMCR_EL0 获取 counter 数量:
read_sys_reg64(vcpu_fd, &ahcf->isar.reset_pmcr_el0, ARM64_SYS_REG(3,3,9,12,0));
```

PMU 中断线在 `kvm_arch_post_run()` 中同步。

---

## 6. KVM Device API

### 6.1 设备地址注册

ARM64 的 GIC/Timer 地址需要在机器初始化完成后告知 KVM：

```c
// target/arm/kvm.c:760-784
void kvm_arm_register_device(MemoryRegion *mr, uint64_t devid,
                              uint64_t group, uint64_t attr, int dev_fd,
                              uint64_t addr_ormask) {
    if (!kvm_irqchip_in_kernel()) return;

    // 延迟初始化：注册 memory listener + machine-init-done notifier
    static bool first_time = true;
    if (first_time) {
        memory_listener_register(&devlistener, &address_space_memory);
        qemu_add_machine_init_done_notifier(&notify);
        first_time = false;
    }

    // 记录设备信息
    KVMDevice *kd = g_new0(KVMDevice, 1);
    kd->mr = mr;
    kd->kda.id = devid;
    kd->kda.group = group;
    kd->kda.attr = attr;
    kd->dev_fd = dev_fd;
    kd->kda_addr_ormask = addr_ormask;
    QSLIST_INSERT_HEAD(&kvm_devices_head, kd, entries);
}
```

### 6.2 Machine Init Done 回调

```c
// target/arm/kvm.c:741-754
static void kvm_arm_machine_init_done(Notifier *notifier, void *data) {
    KVMDevice *kd, *kd_next;
    QSLIST_FOREACH_SAFE(kd, &kvm_devices_head, entries, kd_next) {
        if (kd->kda.addr != 0) {
            kvm_arm_set_device_addr(kd);  // 设置 KVM 设备地址
        }
        g_free(kd);
    }
    memory_listener_unregister(&devlistener);
}
```

### 6.3 KVM_SET_DEVICE_ATTR

```c
// target/arm/kvm.c:724-739
static void kvm_arm_set_device_addr(KVMDevice *kd) {
    struct kvm_device_attr attr = {
        .group = kd->kda.group,
        .attr = kd->kda.attr,
        .addr = (uintptr_t)&kd->kda.addr,
    };
    kd->kda.addr |= kd->kda_addr_ormask;  // OR mask (如 legacy bit)
    ioctl(kd->dev_fd, KVM_SET_DEVICE_ATTR, &attr);
}
```

### 6.4 典型用途

```
GIC Distributor:
  kvm_arm_register_device(dist_mr, KVM_DEV_ARM_VGIC_GRP_ADDR,
                          KVM_VGIC_V3_ADDR_TYPE_DIST, dev_fd, 0)

GIC Redistributor:
  kvm_arm_register_device(redist_mr, KVM_DEV_ARM_VGIC_GRP_ADDR,
                          KVM_VGIC_V3_ADDR_TYPE_REDIST, dev_fd, 0)
```

---

## 7. cpreg 系统寄存器管理

### 7.1 初始化寄存器列表

```c
// target/arm/kvm.c:831-892
static int kvm_arm_init_cpreg_list(ARMCPU *cpu) {
    struct kvm_reg_list rl = { .n = 0 };

    // 1. 查询列表大小
    ioctl(cs->kvm_fd, KVM_GET_REG_LIST, &rl);  // 返回 n = 所需大小

    // 2. 分配并获取完整列表
    rl2 = g_malloc(sizeof(*rl2) + rl.n * sizeof(uint64_t));
    rl2->n = rl.n;
    ioctl(cs->kvm_fd, KVM_GET_REG_LIST, rl2);

    // 3. 排序
    qsort(rl2->reg, rl2->n, sizeof(uint64_t), compare_u64);

    // 4. 过滤：只保留 cpreg 可同步的寄存器
    for (i = 0; i < rl2->n; i++) {
        uint64_t regidx = rl2->reg[i];
        if (!kvm_arm_reg_syncs_via_cpreg_list(regidx)) continue;
        // 只接受 U32 和 U64
        size = (regidx & KVM_REG_SIZE_MASK);
        if (size != KVM_REG_SIZE_U32 && size != KVM_REG_SIZE_U64) continue;

        cpu->cpreg_indexes[arraylen] = regidx;
        arraylen++;
    }

    // 5. 初始读取所有寄存器值
    write_kvmstate_to_list(cpu);
}
```

### 7.2 寄存器 ID 编码

```
KVM ARM64 寄存器 ID (64 bit):
┌────────┬──────┬──────┬─────────────────────────┐
│ KVM_REG│ SIZE │ KIND │     Register Index       │
│ _ARM64 │ U32/ │CORE/ │                         │
│ (16bit)│ U64  │SYSREG│                         │
└────────┴──────┴──────┴─────────────────────────┘

ARM64 SYSREG 编码:
  KVM_REG_ARM64 | KVM_REG_SIZE_U64 | KVM_REG_ARM64_SYSREG |
  (Op0 << 14) | (Op1 << 11) | (CRn << 7) | (CRm << 3) | Op2

例: SCTLR_EL1 = (3,0,1,0,0) → 0x6030_0100_0010
```

### 7.3 同步方向

```c
// target/arm/kvm.c:921-950 — KVM → QEMU
bool write_kvmstate_to_list(ARMCPU *cpu) {
    for (i = 0; i < cpu->cpreg_array_len; i++) {
        uint64_t regidx = cpu->cpreg_indexes[i];
        ret = kvm_get_one_reg(cs, regidx, &val);
        cpu->cpreg_values[i] = val;
    }
}

// target/arm/kvm.c:1004-1073 — QEMU → KVM
bool write_list_to_kvmstate(ARMCPU *cpu, int level) {
    for (i = 0; i < cpu->cpreg_array_len; i++) {
        // 跳过高于当前 level 的寄存器
        if (kvm_arm_cpreg_level(regidx) > level) continue;
        kvm_set_one_reg(cs, regidx, &cpu->cpreg_values[i]);
    }
}
```

### 7.4 同步级别

```c
// target/arm/kvm.c:906-919
static int kvm_arm_cpreg_level(uint64_t regidx) {
    // Timer counter 寄存器只在完整同步时写入
    switch (regidx) {
    case KVM_REG_ARM_TIMER_CNT:
    case KVM_REG_ARM_PTIMER_CNT:
        return KVM_PUT_FULL_STATE;  // 仅 migration restore
    default:
        return KVM_PUT_RUNTIME_STATE;  // 每次 put 都同步
    }
}
```

| Level | 含义 | 时机 |
|-------|------|------|
| `KVM_PUT_RUNTIME_STATE` | 运行时同步 | 每次 vcpu_dirty 时 |
| `KVM_PUT_RESET_STATE` | 重置时同步 | CPU reset |
| `KVM_PUT_FULL_STATE` | 完整同步 | 迁移恢复 |

---

## 8. Pre/Post Run 钩子

### 8.1 kvm_arch_pre_run()

```c
// target/arm/kvm.c:1333-1358
MemTxAttrs kvm_arch_pre_run(CPUState *cs, struct kvm_run *run) {
    // ARM64 pre-run 很轻量
    // 处理 external DABT 注入标志
    if (env->ext_dabt_raised) {
        // 旧内核兼容：验证注入是否成功
        // 新内核通过 KVM_SET_VCPU_EVENTS 处理
        env->ext_dabt_raised = false;
    }
    return MEMTXATTRS_UNSPECIFIED;
}
```

### 8.2 kvm_arch_post_run()

```c
// target/arm/kvm.c:1360-1412
MemTxAttrs kvm_arch_post_run(CPUState *cs, struct kvm_run *run) {
    // 如果使用 in-kernel irqchip，直接返回
    if (kvm_irqchip_in_kernel()) {
        return MEMTXATTRS_UNSPECIFIED;
    }

    // 用户空间 irqchip 模式：同步中断线电平
    bql_lock();

    // 同步 virtual timer IRQ
    uint32_t device_irq_level = run->s.regs.device_irq_level;
    if ((device_irq_level ^ prev_level) & KVM_ARM_DEV_EL1_VTIMER) {
        qemu_set_irq(vtimer_irq, (device_irq_level >> VTIMER_BIT) & 1);
    }
    // 同步 physical timer IRQ
    if ((device_irq_level ^ prev_level) & KVM_ARM_DEV_EL1_PTIMER) {
        qemu_set_irq(ptimer_irq, (device_irq_level >> PTIMER_BIT) & 1);
    }
    // 同步 PMU IRQ
    if ((device_irq_level ^ prev_level) & KVM_ARM_DEV_PMU) {
        qemu_set_irq(pmu_irq, (device_irq_level >> PMU_BIT) & 1);
    }

    prev_level = device_irq_level;
    bql_unlock();
    return MEMTXATTRS_UNSPECIFIED;
}
```

### 8.3 对比

| 钩子 | ARM64 工作量 | x86 工作量 |
|------|------------|-----------|
| pre_run | 极轻（DABT 检查）| 重（APIC/TPR/中断窗口）|
| post_run | 轻（Timer/PMU IRQ sync）| 重（APIC 状态同步）|

ARM64 大部分工作由 in-kernel GIC 处理，用户态开销很小。

---

## 9. vCPU 线程模型

### 9.1 线程创建

```c
// accel/kvm/kvm-accel-ops.c:68-76
static void kvm_start_vcpu_thread(CPUState *cpu) {
    char thread_name[16];
    snprintf(thread_name, sizeof(thread_name), "CPU %d/KVM", cpu->cpu_index);
    qemu_thread_create(cpu->thread, thread_name, kvm_vcpu_thread_fn,
                       cpu, QEMU_THREAD_JOINABLE);
}
```

### 9.2 线程主函数

```c
// accel/kvm/kvm-accel-ops.c:31-65
static void *kvm_vcpu_thread_fn(void *arg) {
    CPUState *cpu = arg;

    rcu_register_thread();
    bql_lock();

    // 1. 初始化
    qemu_thread_get_self(cpu->thread);
    cpu->thread_id = qemu_get_thread_id();
    current_cpu = cpu;
    kvm_init_vcpu(cpu, &error_fatal);    // KVM_CREATE_VCPU
    kvm_init_cpu_signals(cpu);            // 设置信号处理
    cpu_thread_signal_created(cpu);       // 通知创建完成

    // 2. 主循环
    do {
        if (cpu_can_run(cpu)) {
            int r = kvm_cpu_exec(cpu);     // 进入 KVM
            if (r == EXCP_DEBUG) {
                cpu_handle_guest_debug(cpu);
            }
        }
        qemu_wait_io_event(cpu);           // 等待事件（halt/kick）
    } while (!cpu->unplug || cpu_can_run(cpu));

    // 3. 清理
    kvm_destroy_vcpu(cpu);
    cpu_thread_signal_destroyed(cpu);
    bql_unlock();
    rcu_unregister_thread();
    return NULL;
}
```

### 9.3 BQL 与 KVM_RUN 关系

```
vCPU 线程持有 BQL
  → kvm_cpu_exec()
    → bql_unlock()         ← 释放锁（允许其他线程/main loop）
    → KVM_RUN              ← 进入内核，Guest 执行
    → VM Exit              ← 回到 QEMU 用户态
    → cpu_exec_end()
    → bql_lock()           ← 重新获取锁
    → 处理 exit (MMIO etc) ← 在锁保护下
  → 返回主循环
```

### 9.4 vCPU Kick 机制

```c
// 当主线程需要中断 vCPU 时（如发送中断、热迁移同步）:
cpu_kick(cpu)
  → cpu->exit_request = true
  → pthread_kill(cpu->thread_id, SIG_IPI)
    → KVM_RUN 收到信号 → 返回 -EINTR
    → 或 run->immediate_exit 被设置
```

---

## 10. 调试支持

### 10.1 ARM64 调试退出处理

```c
// target/arm/kvm.c:1485-1543
static bool kvm_arm_handle_debug(CPUState *cs, struct kvm_debug_exit_arch *debug_exit) {
    ARMCPU *cpu = ARM_CPU(cs);
    kvm_cpu_synchronize_state(cs);  // 同步寄存器到 QEMU

    int hsr_ec = syn_get_ec(debug_exit->hsr);  // Exception Class

    switch (hsr_ec) {
    case EC_SOFTWARESTEP:
        // 单步调试
        if (find_breakpoint_at(env->pc)) {
            env->exception.vaddress = env->pc;
        }
        return true;

    case EC_AA64_BKPT:
        // AArch64 断点 (BRK指令)
        env->exception.vaddress = env->pc;
        return true;

    case EC_BREAKPOINT:
        // 硬件断点匹配
        env->exception.vaddress = env->pc;
        return true;

    case EC_WATCHPOINT:
        // 数据 watchpoint
        env->exception.vaddress = debug_exit->far;
        return true;
    }

    // 未处理的调试事件 → 转为 guest 异常
    arm_cpu_do_interrupt(cs);
    return false;
}
```

### 10.2 Guest Debug 配置

```c
// target/arm/kvm.c:1615-1624
int kvm_arch_update_guest_debug(CPUState *cs) {
    struct kvm_guest_debug dbg = {
        .control = KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_SW_BP,
    };
    if (kvm_arm_hw_debug_active(cs)) {
        dbg.control |= KVM_GUESTDBG_USE_HW;
        kvm_arm_copy_hw_debug_data(&dbg.arch);  // 复制 BP/WP 寄存器
    }
    return kvm_vcpu_ioctl(cs, KVM_SET_GUEST_DEBUG, &dbg);
}
```

---

## 11. 与硬件对比：EL2 Hypervisor 行为

### 11.1 VM Exit 对比

| ARM64 硬件行为 | KVM 实现 | QEMU 角色 |
|---------------|----------|-----------|
| HCR_EL2.IMO/FMO → IRQ/FIQ trap to EL2 | KVM 设置 HCR_EL2 | QEMU 配置 irqchip |
| Stage-2 Translation Fault | KVM 检查 memslot | QEMU MMIO dispatch |
| HVC #N → Trap to EL2 | KVM PSCI 处理 | QEMU 可选处理 |
| SMC → Trap to EL2 (HCR_EL2.TSC) | KVM trap → PSCI | 部分转发到 QEMU |
| WFI → Trap (HCR_EL2.TWI) | vCPU halt | QEMU event loop |
| Debug exception → Trap to EL2 | KVM_EXIT_DEBUG | QEMU GDB server |

### 11.2 HCR_EL2 关键位

| HCR_EL2 位 | 作用 | KVM 默认设置 |
|------------|------|-------------|
| VM | Stage-2 使能 | 1 |
| IMO | IRQ 路由到 EL2 | 1 (in-kernel GIC) |
| FMO | FIQ 路由到 EL2 | 1 |
| AMO | SError 路由到 EL2 | 1 |
| TWI | WFI trap | 1 |
| TWE | WFE trap | 0 (通常) |
| TSC | SMC trap | 1 |
| TVM | VM寄存器写 trap | 条件性 |

### 11.3 QEMU/KVM vs 裸金属 Hypervisor 差异

| 方面 | QEMU/KVM | 裸金属 EL2 (如 Xen) |
|------|----------|---------------------|
| Stage-2 管理 | KVM 内核，QEMU 提供 HVA | EL2 直接操作页表 |
| 中断处理 | vGIC (内核) + irqfd | 直接操作 GIC 硬件 |
| 设备模型 | QEMU 用户态模拟 | 直通 (passthrough) 为主 |
| 上下文切换 | EL2 entry/exit + 用户态/内核态 | 仅 EL2 entry/exit |
| Timer | KVM 内核 vtimer | 直接 CNTVOFF_EL2 |
| PSCI | KVM 内核处理 | EL2 直接处理 |
| 调试 | GDB → QEMU → KVM_SET_GUEST_DEBUG | 需要特殊调试接口 |

### 11.4 性能关键路径

```
快速路径 (0 exit):
  Guest 代码执行 → 内存访问(RAM) → Stage-2 命中 → 继续执行
  VirtIO I/O → ioeventfd → irqfd → 全内核处理

慢速路径 (2 exits):
  Guest MMIO → Stage-2 fault → EL2 → KVM_EXIT_MMIO → QEMU → KVM_RUN
  Guest 需要中断 → QEMU kick → KVM_RUN → inject → Guest

最慢路径 (多次 exit):
  复杂设备模拟（如 USB、网卡 non-dataplane）→ 频繁 MMIO exit
```

---

## 附录 A: kvm_vcpu_init.features 位

| Bit | 宏名 | 含义 |
|-----|------|------|
| 0 | `KVM_ARM_VCPU_POWER_OFF` | vCPU 初始断电 |
| 1 | `KVM_ARM_VCPU_EL1_32BIT` | AArch32 模式 |
| 2 | `KVM_ARM_VCPU_PSCI_0_2` | PSCI 0.2 接口 |
| 3 | `KVM_ARM_VCPU_PMU_V3` | PMU v3 |
| 4 | `KVM_ARM_VCPU_SVE` | SVE 向量扩展 |
| 5 | `KVM_ARM_VCPU_PTRAUTH_ADDRESS` | PAuth (地址) |
| 6 | `KVM_ARM_VCPU_PTRAUTH_GENERIC` | PAuth (通用) |
| 7 | `KVM_ARM_VCPU_HAS_EL2` | Nested Virt |

---

## 附录 B: 完整退出处理表

| exit_reason | 处理位置 | 动作 |
|-------------|---------|------|
| `KVM_EXIT_MMIO` | kvm-all.c:3531 | address_space_rw() |
| `KVM_EXIT_IO` | kvm-all.c:3525 | address_space_rw() (x86) |
| `KVM_EXIT_IRQ_WINDOW_OPEN` | kvm-all.c:3541 | ret=EXCP_INTERRUPT |
| `KVM_EXIT_SHUTDOWN` | kvm-all.c:3543 | qemu_system_shutdown |
| `KVM_EXIT_UNKNOWN` | kvm-all.c:3547 | qemu_system_reset |
| `KVM_EXIT_INTERNAL_ERROR` | kvm-all.c:3552 | kvm_handle_internal_error |
| `KVM_EXIT_DIRTY_RING_FULL` | kvm-all.c:3556 | reap + reset ring |
| `KVM_EXIT_SYSTEM_EVENT` | kvm-all.c:3577 | shutdown/reset/crash |
| `KVM_EXIT_DEBUG` | kvm-all.c:3614 | → kvm_arch_handle_exit |
| `KVM_EXIT_ARM_NISV` | arm/kvm.c:1557 | inject ext DABT |

---

## 附录 C: 源码文件索引

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| `accel/kvm/kvm-all.c` | 4786 | 通用 KVM：cpu_exec、exit 处理、memory |
| `target/arm/kvm.c` | 2617 | ARM64 KVM：特性探测、cpreg、debug、pre/post run |
| `accel/kvm/kvm-accel-ops.c` | ~110 | vCPU 线程创建与主循环 |
| `target/arm/kvm_arm.h` | 243 | ARM KVM 接口声明 |
| `target/arm/kvm-consts.h` | ~50 | KVM 常量定义 |
| `include/system/kvm.h` | 610 | KVM 公共 API |
