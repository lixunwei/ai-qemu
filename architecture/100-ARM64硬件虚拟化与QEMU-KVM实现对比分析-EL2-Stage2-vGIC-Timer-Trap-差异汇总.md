# Doc 100: ARM64 硬件虚拟化与 QEMU/KVM 实现对比分析

## 真实硬件行为 · KVM 内核映射 · QEMU 用户态职责 · 模拟差异与限制

> QEMU 11.0.50 · ARM Architecture Reference Manual (Armv9.6) · GICv3/v4 Architecture
> 分析日期: 2025-01
> 前置文档: Doc 98 (KVM内存管理)、Doc 99 (KVM ARM64特性)、知识库 arm64 域

---

## 目录

1. [总体架构对比](#1-总体架构对比)
2. [EL2 与 Hypervisor 配置](#2-el2-与-hypervisor-配置)
3. [Stage-2 地址翻译](#3-stage-2-地址翻译)
4. [中断虚拟化](#4-中断虚拟化)
5. [Timer 虚拟化](#5-timer-虚拟化)
6. [异常与 Trap 路由](#6-异常与-trap-路由)
7. [内存属性与缓存](#7-内存属性与缓存)
8. [PSCI 与电源管理](#8-psci-与电源管理)
9. [TLB 管理](#9-tlb-管理)
10. [调试虚拟化](#10-调试虚拟化)
11. [已知差异与限制汇总](#11-已知差异与限制汇总)
12. [TCG 模式 vs KVM 模式差异](#12-tcg-模式-vs-kvm-模式差异)

---

## 1. 总体架构对比

### 1.1 三种运行模式

```
┌─────────────────────────────────────────────────────────────────┐
│ 模式 A: 真实硬件 (裸金属 Hypervisor, 如 Xen)                     │
│                                                                  │
│ EL3: Secure Monitor (ATF/TF-A)                                  │
│ EL2: Hypervisor (Xen/KVM) — 直接操作 HCR_EL2/VTTBR_EL2         │
│ EL1: Guest OS                                                    │
│ EL0: Guest Application                                           │
├──────────────────────────────────────────────────────────────────┤
│ 模式 B: QEMU/KVM (硬件辅助虚拟化)                                │
│                                                                  │
│ EL2: KVM 内核模块 — 配置 HCR_EL2, 管理 Stage-2 页表              │
│ EL1: Guest OS — 直接在物理 CPU 执行                              │
│ EL0: Guest App — 直接在物理 CPU 执行                             │
│ User: QEMU — 处理设备模拟、管理面                                │
├──────────────────────────────────────────────────────────────────┤
│ 模式 C: QEMU/TCG (纯软件模拟)                                    │
│                                                                  │
│ Host EL0: QEMU 进程                                              │
│   内部模拟 EL3/EL2/EL1/EL0 — 全软件翻译与模拟                    │
│   无真实硬件虚拟化参与                                            │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 职责分工

| 硬件机制 | 真实硬件 | KVM 内核 | QEMU (KVM模式) | QEMU (TCG模式) |
|---------|---------|----------|---------------|---------------|
| HCR_EL2 配置 | 直接写寄存器 | 内核设置 | 通过 ONE_REG 接口 | 软件模拟 (`env->cp15.hcr_el2`) |
| Stage-2 页表 | 直接写 VTTBR_EL2 | 内核管理 | 提供 HVA (memslot) | 软件 TLB (`get_phys_addr_lpae`) |
| vGIC 中断 | 写 ICH_LR / GICR | KVM vGIC | 配置 + irqfd | 软件 GICv3 模型 |
| Timer | 设置 CNTVOFF_EL2 | 内核 vtimer | kvm_vtime 同步 | QEMUTimer + callback |
| WFI/WFE | HCR_EL2.TWI trap | vCPU halt | event loop 等待 | helper_wfi() halt |
| PSCI | SMC/HVC at EL2 | KVM 内核处理 | system_event 通知 | arm_handle_psci_call() |

---

## 2. EL2 与 Hypervisor 配置

### 2.1 HCR_EL2 关键位 — 硬件定义

> 来源: ARM Architecture Reference Manual, D13.2

```
HCR_EL2 (64-bit register):
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ VM  │SWIO │ PTW │ FMO │ IMO │ AMO │ VF  │ VI  │ Bits[7:0]
├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│ VSE │ FB  │ BSU │ DC  │ TWI │ TWE │ TID0│TID1 │ Bits[15:8]
├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│TID2 │TID3 │ TSC │TIDCP│TACR │ TSW │TPCP │ TPU │ Bits[23:16]
├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│ TDZ │ TGE │ TVM │TTLB │ RW  │ CD  │ ID  │ E2H │ Bits[31:24]
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
```

### 2.2 KVM 设置的 HCR_EL2 值

KVM 内核对 Guest 设置的典型 HCR_EL2：

| 位 | 值 | 含义 | 原因 |
|----|----|------|------|
| VM | 1 | Stage-2 使能 | Guest 地址隔离 |
| IMO | 1 | IRQ → EL2 | vGIC 内核处理中断 |
| FMO | 1 | FIQ → EL2 | vGIC 内核处理中断 |
| AMO | 1 | SError → EL2 | KVM 注入 SError |
| TWI | 1 | WFI trap | vCPU halt 处理 |
| TWE | 0 | WFE 不 trap | 自旋等待不应 exit |
| TSC | 1 | SMC trap | PSCI 拦截 |
| RW | 1 | EL1 AArch64 | ARM64 Guest |
| E2H | VHE时=1 | Host 使用 VHE | 减少寄存器切换 |
| TGE | Host时=1 | General Exception → EL2 | Host kernel 在 EL2 |
| DC | 0 | 不强制 cacheable | Guest 控制自己缓存 |
| TVM | 条件 | VM 寄存器写 trap | 需要拦截时设置 |

### 2.3 VHE (Virtualization Host Extensions)

> 来源: ARM Architecture Reference Manual, FEAT_VHE

```
硬件行为 (HCR_EL2.{E2H, TGE} = {1, 1}):
  - EL2 使用 EL1 的系统寄存器名(重定向)
  - SCTLR_EL1 → 实际访问 SCTLR_EL2
  - TTBR0_EL1 → 实际访问 TTBR0_EL2
  - 使 Host Linux 无需修改即可运行在 EL2

KVM 利用:
  - Host 代码运行在 EL2 + VHE
  - Guest 进入: 清 TGE → Guest 在 EL1 执行
  - Guest 退出: 置 TGE → Host 恢复 EL2 执行

QEMU TCG 模拟 (target/arm/helper.c):
  - arm_hcr_el2_eff(): 计算有效 HCR_EL2 值
  - regime_el(): 根据 E2H+TGE 决定翻译体制
  - 寄存器重定向在 cpregs 中通过 bank 实现
```

### 2.4 差异点

| 方面 | 真实硬件 | QEMU/KVM | 差异说明 |
|------|---------|----------|---------|
| HCR_EL2 写入 | 1 cycle | KVM 设置后直接硬件执行 | 无差异 |
| 位变化生效 | 立即（ISB后）| 需要 VCPU ioctl | 延迟可忽略 |
| HCR_EL2 读取 (Guest) | 返回真实值 | KVM 模拟返回值 | Guest看到的是 KVM 控制的值 |
| VHE context switch | ~200 cycles | ~200 cycles + ioctl overhead | KVM overhead ~1000 cycles |

---

## 3. Stage-2 地址翻译

### 3.1 硬件定义

> 来源: ARM Architecture Reference Manual, D8.4

```
Stage-2 翻译流程:
  Guest VA → [Stage-1: TTBR0/1_EL1] → IPA
  IPA      → [Stage-2: VTTBR_EL2]  → PA

关键寄存器:
  VTTBR_EL2:  Stage-2 页表基地址 + VMID
  VTCR_EL2:   Stage-2 翻译控制 (粒度/大小/共享属性)
  HPFAR_EL2:  Stage-2 fault 的 IPA (高位)
```

### 3.2 KVM 实现

```
QEMU:
  kvm_set_user_memory_region(GPA=0x4000_0000, HVA=mmap_ptr, size=1GB)

KVM 内核:
  记录 memslot: {GPA range → HVA}
  Guest 首次访问 GPA:
    → Stage-2 translation fault (no mapping)
    → KVM fault handler: kvm_handle_guest_abort()
      → 查找 memslot
      → 分配 Host 物理页
      → 填充 Stage-2 PTE: IPA → PA
      → 返回 Guest 继续执行

  Guest 后续访问同一 GPA:
    → Stage-2 TLB 命中 → 直接翻译 → 无 exit
```

### 3.3 对比

| 特性 | 真实硬件 | KVM | QEMU TCG |
|------|---------|-----|----------|
| 翻译粒度 | 4KB/16KB/64KB | 跟随 Host | 软件模拟,无真实粒度 |
| 大页 | 2MB/1GB blocks | THP 自动合并 | 不支持真实大页 |
| TLB | 硬件 ASID+VMID | 硬件管理 | 软件 TLB (QEMUTLBEntry) |
| Dirty tracking | FEAT_HAFDBS (DBM位) | 利用硬件/write-protect | N/A (TCG无KVM概念) |
| 内存属性合并 | Stage-1 ⊗ Stage-2 | 硬件执行 | `combine_s1s2_cacheattrs()` |
| Fault 报告 | ESR_EL2 + HPFAR_EL2 | 内核解析 → KVM_EXIT_MMIO | 软件 raise fault |
| MMIO 检测 | Stage-2 无映射 → fault | 内核无 memslot → exit | MemoryRegion dispatch |

### 3.4 QEMU 不模拟的 Stage-2 细节

- **FEAT_HAFDBS (Hardware Access Flag/Dirty Bit)**:
  - 硬件: Stage-2 PTE 的 AF/DBM 位可硬件自动设置
  - KVM: 利用此特性优化脏页追踪（无需 write-protect）
  - TCG: 不模拟硬件 AF/DBM 自动管理

- **Break-Before-Make**:
  - 硬件: 修改 PTE 必须先写无效→TLBI→再写新值
  - KVM: 内核保证正确序列
  - TCG: 无此约束（软件 TLB 无并发问题）

---

## 4. 中断虚拟化

### 4.1 GICv3 虚拟中断硬件架构

> 来源: GICv3/v4 Architecture Specification, Chapter 6

```
硬件虚拟中断路径:
  Hypervisor (EL2) 写 ICH_LR<n>_EL2:
    → GIC 硬件将虚拟中断呈现给 Guest
    → Guest 读 ICC_IAR1_EL1 (实际访问 ICV_IAR1_EL1)
    → GIC 返回虚拟 INTID
    → Guest 处理完毕写 ICC_EOIR1_EL1 (→ ICV_EOIR1_EL1)
    → GIC 硬件清除 LR

关键寄存器:
  ICH_LR0-15_EL2:  List Registers (最多16个虚拟中断)
  ICH_HCR_EL2:     Hypervisor Control (使能虚拟化)
  ICH_VMCR_EL2:    Virtual Machine Control (虚拟 Priority Mask)
  ICV_*:           Virtual CPU Interface (Guest EL1 视角)
```

### 4.2 中断路由规则

> 来源: GICv3 Spec §4.6.4

```
HCR_EL2.IMO = 1 时:
  - EL1 对 ICC_* 的访问被重定向到 ICV_* (虚拟接口)
  - 物理 IRQ 直接路由到 EL2
  - 虚拟 IRQ 由 ICH_LR 控制呈现

HCR_EL2.FMO = 1 时:
  - 同理 FIQ 路由到 EL2
  - 虚拟 FIQ 由 ICH_LR 或 HCR_EL2.VF 呈现
```

### 4.3 KVM vGIC 实现

```
KVM vGIC (内核态):
  QEMU: kvm_irqchip_add_irqfd(eventfd, gsi)
    → 设备产生中断 → eventfd signal
    → KVM 内核收到 eventfd → kvm_vgic_inject_irq()
    → 设置 vGIC pending state
    → 写 ICH_LR_EL2 → Guest 收到虚拟中断
    → Guest 执行 ISR → 写 ICV_EOIR1_EL1
    → GIC 硬件清除 LR → maintenance interrupt (如需)
    → 全程无 VM Exit！
```

### 4.4 对比

| 方面 | 真实硬件 | KVM vGIC | QEMU TCG GIC |
|------|---------|----------|--------------|
| LR 写入 | 直接 MSR | KVM 写入 | 软件模拟 `ich_lr_el2[]` |
| ICC → ICV 重定向 | 硬件自动 | 硬件执行 | `access_type` 判断 |
| 中断优先级 | 硬件比较器 | 硬件执行 | `gicv3_cpuif_virt_irq_fiq_update()` |
| EOI 处理 | 硬件清 LR | 硬件执行 | 软件清除 + maintenance |
| 注入延迟 | 0 (直接) | ~100ns (irqfd) | ~us (signal + dispatch) |
| 最大并发中断 | 16 (LR数) | 16 (硬件限制) | 无限制 (软件列表) |

### 4.5 GICv4 直通 (Direct Injection)

```
GICv4 硬件（LPI 直通）:
  设备 MSI → ITS → VLPI → 直接注入 Guest
  完全不经过 Hypervisor！

KVM GICv4:
  利用 ITS VLPI 机制实现 LPI 直通
  设备 MSI → 直接写入 Guest vCPU 的 ICH_LR
  无 VM Exit、无 KVM 干预

QEMU 角色:
  仅在初始化时配置 ITS VPE table
  运行时不参与 LPI 分发
```

---

## 5. Timer 虚拟化

### 5.1 硬件机制

```
ARM Generic Timer 虚拟化:
  CNTVOFF_EL2: 虚拟计数器偏移
  CNTV_CTL_EL0: 虚拟 timer 控制
  CNTV_CVAL_EL0: 虚拟 timer 比较值

  虚拟计数器: CNTVCT_EL0 = CNTPCT_EL0 - CNTVOFF_EL2
  虚拟 timer 到期: CNTVCT >= CNTV_CVAL → 产生 PPI 27 中断
```

### 5.2 KVM Timer 实现

```c
// KVM 内核:
// 1. VM 进入前: 设置 CNTVOFF_EL2 = 累计 VM 停止时间
// 2. VM 运行中: 硬件自动运行虚拟 timer
// 3. VM 退出后: 读取 timer 状态

// QEMU 同步 (target/arm/kvm.c):
kvm_arm_get_virtual_time(cpu)
  → kvm_get_one_reg(KVM_REG_ARM_TIMER_CNT)
  → 保存到 cpu->kvm_vtime

kvm_arm_put_virtual_time(cpu)
  → kvm_set_one_reg(KVM_REG_ARM_TIMER_CNT)
  → 恢复虚拟时间
```

### 5.3 对比

| 方面 | 真实硬件 | KVM | QEMU TCG |
|------|---------|-----|----------|
| CNTVOFF_EL2 | 直接写寄存器 | 内核管理 (no_adjvtime选项) | `env->cp15.cntvoff_el2` |
| timer 精度 | 纳秒级 | 纳秒级(硬件) | 微秒级(QEMUTimer) |
| timer 到期 | 硬件中断 | 硬件 → PPI inject | QEMUTimer callback → set IRQ |
| 时间流逝 | 单调递增 | 单调递增 | `icount` 或 host clock |
| 多核同步 | 硬件保证 | 硬件保证 | QEMU 不完全保证 |

---

## 6. 异常与 Trap 路由

### 6.1 硬件 Trap 机制

```
HCR_EL2 Trap 位效果:
  TSC=1: SMC → EL2 (trap)
  TWI=1: WFI → EL2 (trap, 可选delay)
  TVM=1: SCTLR/TCR/MAIR... 写 → EL2 (trap)
  TIDCP=1: 实现定义寄存器 → EL2 (trap)
  TACR=1: ACTLR_EL1 访问 → EL2 (trap)

陷入流程:
  Guest 执行 SMC #0
    → CPU 检查 HCR_EL2.TSC == 1
    → Trap to EL2 (不是 EL3!)
    → ESR_EL2.EC = 0x17 (SMC from AArch64)
    → Hypervisor 处理
```

### 6.2 KVM 处理映射

| Guest 操作 | ARM Trap 条件 | KVM 行为 | 是否 Exit 到 QEMU |
|-----------|-------------|---------|------------------|
| SMC (PSCI) | HCR_EL2.TSC | KVM 内核处理 PSCI | 仅 SYSTEM_OFF/RESET → QEMU |
| WFI | HCR_EL2.TWI | vCPU halt (内核) | 否 (除非有 pending event) |
| WFE | HCR_EL2.TWE=0 | 不 trap | 否 |
| MMIO 访问 | Stage-2 fault | KVM_EXIT_MMIO | ✅ 是 |
| MSR/MRS (系统寄存器) | HSTR/HCR.TVM | 通常内核模拟 | 极少 Exit |
| HVC | 固定 trap | KVM 处理或转发 | 条件性 |
| Debug (BRK) | MDCR_EL2 | KVM_EXIT_DEBUG | ✅ 是 |

### 6.3 差异

- **真实硬件**: 每个 trap 都有精确的 ESR_EL2 编码，可以细粒度判断
- **KVM**: 大部分 trap 在内核处理，仅少数需要 QEMU 参与
- **TCG**: 不存在真实 trap，所有异常都是软件 `raise_exception()` + handler dispatch

---

## 7. 内存属性与缓存

### 7.1 硬件 Stage-2 内存属性

```
Stage-2 PTE MemAttr[3:0]:
  0000: Device-nGnRnE (最强序)
  0001: Device-nGnRE
  0101: Normal Write-Back
  1111: Normal Write-Back (outer allocate)

属性合并规则:
  最终属性 = MOST_RESTRICTIVE(Stage-1 attr, Stage-2 attr)
  例: Stage-1 = WB, Stage-2 = Device → 最终 = Device
```

### 7.2 KVM 处理

- **RAM 区域**: KVM Stage-2 PTE 设为 Normal Write-Back
- **MMIO 区域**: 无 Stage-2 映射 → fault → exit 到 QEMU
- **设备直通**: Stage-2 映射为 Device memory 属性

### 7.3 QEMU TCG 模拟

```c
// target/arm/ptw.c:2195-2220 (概念)
static ARMCacheAttrs combine_s1s2_cacheattrs(ARMCacheAttrs s1, ARMCacheAttrs s2) {
    // 实现最严格合并规则
    if (s2.is_device) return s2;  // Device 最强
    if (s1.is_device) return s1;
    // Normal: 取更严格的 Inner/Outer 属性
    return most_restrictive(s1, s2);
}
```

### 7.4 差异

| 方面 | 真实硬件 | KVM | QEMU TCG |
|------|---------|-----|----------|
| Cache coherency | 硬件保证 | 硬件保证 | **不模拟真实缓存** |
| Shareability | Inner/Outer/Non | 跟随硬件 | 忽略（单一地址空间）|
| MAIR 效果 | 实际影响缓存行为 | 实际影响 | 仅记录，不影响 QEMU 内存访问 |
| DMB/DSB | 真实内存屏障 | 真实执行 | `tcg_gen_mb()` → 仅影响 TCG 内部排序 |
| Cache 维护指令 | 真实操作缓存 | 真实执行 | NOP (TCG 无缓存模型) |

**关键差异**: QEMU TCG **不模拟真实缓存行为**。对 Guest 来说缓存维护指令是 NOP。这在功能上通常无影响（因为 Guest 看到 coherent 内存），但对性能分析和某些驱动时序有偏差。

---

## 8. PSCI 与电源管理

### 8.1 硬件 PSCI 流程

```
Guest 调用 PSCI:
  SMC #0 (args: function_id, target_cpu, entry_point)
    → HCR_EL2.TSC = 1 → Trap to EL2
    → ESR_EL2.EC = 0x17 (SMC)
    → Hypervisor 解析 PSCI function_id
    → 执行电源管理操作
```

### 8.2 KVM 实现

```
Guest SMC → Trap to EL2 (KVM)
  → kvm_psci_call():
    PSCI_CPU_ON:
      → 创建/启动 vCPU 线程
      → 设置 entry point + context id
      → 返回 SUCCESS

    PSCI_CPU_OFF:
      → vCPU 进入 power-off 状态
      → 不再调度

    PSCI_SYSTEM_RESET:
      → KVM_EXIT_SYSTEM_EVENT (type=RESET)
      → QEMU: qemu_system_reset_request()

    PSCI_SYSTEM_OFF:
      → KVM_EXIT_SYSTEM_EVENT (type=SHUTDOWN)
      → QEMU: qemu_system_shutdown_request()
```

### 8.3 TCG 实现

```c
// target/arm/psci.c
void arm_handle_psci_call(ARMCPU *cpu) {
    switch (param[0]) {  // function_id
    case PSCI_FN_CPU_ON:
        target_cpu = arm_get_cpu_by_id(param[1]);
        target_cpu->env.regs[15] = param[2];  // entry point
        target_cpu->power_state = PSCI_ON;
        qemu_cpu_kick(target_cpu);
        break;
    case PSCI_FN_SYSTEM_RESET:
        qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
        break;
    }
}
```

### 8.4 差异

| 方面 | 真实 PSCI | KVM | TCG |
|------|----------|-----|-----|
| 调用路径 | SMC → EL3 TF-A | SMC → EL2 KVM | SMC/HVC → helper |
| CPU_ON 延迟 | 硬件唤醒 ~ms | 线程唤醒 ~100μs | qemu_cpu_kick ~10μs |
| 版本支持 | TF-A 决定 | KVM 决定 (0.2/1.0/1.1) | QEMU 配置 |
| Conduit | SMC (通过 EL3) | HVC (QEMU DT 写 "hvc") | SMC 或 HVC |

---

## 9. TLB 管理

### 9.1 硬件 TLBI 指令

```
ARM64 TLB 维护:
  TLBI VMALLE1IS:    清除所有 EL1 Stage-1 TLB (Inner Shareable)
  TLBI IPAS2E1IS:    清除指定 IPA 的 Stage-2 TLB
  TLBI ALLE2:        清除所有 EL2 TLB
  TLBI VALE1IS:      清除指定 VA 的 EL1 TLB (Last level)

VMID (8/16 bit):
  - 标记 Stage-2 TLB entry 属于哪个 VM
  - 切换 VM 时不必 flush 所有 TLB (VMID 隔离)
```

### 9.2 KVM 处理

- **Guest TLBI**: Guest 在 EL1 执行 TLBI → 硬件直接执行（不 trap）
- **Hypervisor TLBI**: KVM 管理 Stage-2 映射变化时自动发 TLBI
- **VMID 分配**: KVM 为每个 VM 分配唯一 VMID

### 9.3 TCG 处理

```c
// target/arm/tlb_helper.c
void helper_tlbi_vale1_is(CPUARMState *env, uint64_t value) {
    // 清除 QEMU 软件 TLB 中匹配的条目
    tlb_flush_page_by_mmuidx(cs, addr, ARMMMUIdxBit_E10_1 | ...);
}
```

### 9.4 差异

| 方面 | 真实硬件 | KVM | TCG |
|------|---------|-----|-----|
| TLB 大小 | 数千条 | 硬件 | QEMU_TLB_SIZE (256) |
| VMID | 8/16 bit 硬件标签 | 内核分配 | `arm_to_core_mmu_idx` |
| Broadcast (IS) | 跨核硬件信号 | 硬件执行 | `async_run_on_cpu` |
| 性能 | ~10 cycles/TLBI | ~10 cycles | ~100 cycles (软件查找) |

---

## 10. 调试虚拟化

### 10.1 硬件调试架构

```
ARM64 调试寄存器:
  DBGBCR<n>_EL1: 断点控制 (最多16组)
  DBGBVR<n>_EL1: 断点值
  DBGWCR<n>_EL1: watchpoint 控制 (最多16组)
  DBGWVR<n>_EL1: watchpoint 值
  MDSCR_EL1:     调试状态控制

虚拟化控制:
  MDCR_EL2.TDE:  Debug 异常路由到 EL2
  MDCR_EL2.TDA:  Debug 寄存器访问 trap
```

### 10.2 KVM 调试

```
QEMU GDB server → kvm_arch_update_guest_debug()
  → KVM_SET_GUEST_DEBUG:
    control = KVM_GUESTDBG_ENABLE | KVM_GUESTDBG_USE_SW_BP
    if hw_bp: control |= KVM_GUESTDBG_USE_HW
    dbg.arch.dbg_bcr[n] = ...  // 断点配置
    dbg.arch.dbg_bvr[n] = ...  // 断点地址

Guest 命中断点:
  → BRK/BKPT → Debug exception
  → MDCR_EL2.TDE → EL2 (KVM)
  → KVM_EXIT_DEBUG
  → QEMU: kvm_arm_handle_debug()
  → GDB server 报告断点
```

### 10.3 差异

| 方面 | 真实硬件 | KVM | TCG |
|------|---------|-----|-----|
| HW 断点数 | CPU 决定(通常 4-6) | 受限于物理 CPU | 无限制(软件) |
| 单步 | MDSCR_EL1.SS | KVM 设置 SS | `cpu_single_step()` |
| Watchpoint | 硬件地址比较器 | 利用真实硬件 WP | 软件 `check_watchpoint()` |
| 性能影响 | 无(硬件) | 无(硬件) | 有(每次内存访问检查) |

---

## 11. 已知差异与限制汇总

### 11.1 KVM 模式已知差异

| 编号 | 差异 | 影响 | 原因 |
|------|------|------|------|
| K1 | Timer 精度受 Host 调度影响 | Guest 时间可能跳变 | vCPU 被调度出时 timer 不停 |
| K2 | MMIO exit 延迟 ~3-8μs | 设备访问慢 | 用户态/内核态切换 |
| K3 | 最多 16 个并发虚拟中断 | 理论限制 | ICH_LR 硬件数量 |
| K4 | IPA 大小受限于 Host 硬件 | 通常 40/48 bit | ID_AA64MMFR0_EL1.PARange |
| K5 | 不支持真实 EL3 | 无 Secure Monitor | KVM 运行在 EL2 |
| K6 | Nested Virt 需要硬件 FEAT_NV | 部分硬件不支持 | 依赖 CPU 型号 |

### 11.2 TCG 模式已知差异

| 编号 | 差异 | 影响 | 原因 |
|------|------|------|------|
| T1 | 无真实缓存模型 | Cache 维护指令无效果 | 性能原因不模拟 |
| T2 | 内存序模型简化 | 多核 ordering 不精确 | TCG 使用粗粒度屏障 |
| T3 | Timer 精度依赖 icount/host | 时间行为不同 | 非真实硬件计时 |
| T4 | TLB 大小固定 256 entry | 不模拟 TLB 压力 | 性能折中 |
| T5 | FEAT_HAFDBS 不完全模拟 | AF/DBM 不自动置位 | 软件 TLB 无此逻辑 |
| T6 | PMU 不计真实事件 | 性能计数器不准确 | 纯模拟环境 |
| T7 | Spectre/Meltdown 不模拟 | 无微架构侧信道 | QEMU 无投机执行 |

### 11.3 对 Guest 功能无影响的差异

这些差异虽然存在，但不影响 Guest 功能正确性：
- 缓存维护指令 NOP → Guest 看到 coherent 内存，功能正确
- TLB 大小不同 → 仅影响性能，不影响翻译正确性
- Timer 精度 → 功能正确（时间只会晚到，不会缺失）

---

## 12. TCG 模式 vs KVM 模式差异

### 12.1 执行模型根本差异

```
TCG:
  Guest 指令 → TCG 前端翻译 → TCG IR → 后端代码生成 → Host 执行
  每条 Guest 指令都经过翻译（有 TB 缓存）
  所有系统行为在 QEMU 内部模拟

KVM:
  Guest 指令 → 直接在物理 CPU 执行（EL1）
  仅特权操作 trap 到 KVM/QEMU
  利用真实硬件虚拟化扩展
```

### 12.2 功能覆盖差异

| 功能 | TCG | KVM |
|------|-----|-----|
| 异构 CPU (A53+A72) | ✅ 可模拟不同CPU | ❌ 所有vCPU相同 |
| EL3 (Secure Monitor) | ✅ 完整模拟 | ❌ 不支持 |
| EL2 (Hypervisor in Guest) | ✅ 完整模拟 | ⚠️ 需要 FEAT_NV |
| 自定义 ID 寄存器 | ✅ 任意配置 | ⚠️ 受限于 Host |
| 指令追踪 | ✅ 每条指令可hook | ❌ 仅 exit 点 |
| 跨架构 (x86→ARM) | ✅ 主要用例 | ❌ 需要 ARM Host |
| 性能 | 10-100x 慢 | ~1x 原生 |

### 12.3 何时用 TCG vs KVM

| 场景 | 推荐 | 原因 |
|------|------|------|
| 生产虚拟化 | KVM | 性能接近原生 |
| 固件开发 (TF-A) | TCG | 需要 EL3 |
| 内核调试 | KVM + GDB | 真实行为 + 调试 |
| 驱动开发 | TCG | 完整设备模拟 |
| 安全研究 | TCG | 完整异常模型 |
| CI 测试 (x86 Host) | TCG | 跨架构运行 |

---

## 附录 A: 关键寄存器对照表

| 寄存器 | 硬件用途 | KVM 管理方式 | TCG 存储位置 |
|--------|---------|-------------|-------------|
| HCR_EL2 | Hypervisor 配置 | 内核直接设置 | `env->cp15.hcr_el2` |
| VTTBR_EL2 | Stage-2 基地址 | 内核页表管理 | `env->cp15.vttbr_el2` |
| VTCR_EL2 | Stage-2 翻译控制 | 内核设置 | `env->cp15.vtcr_el2` |
| ICH_LR<n>_EL2 | 虚拟中断 list | KVM vGIC | `gicv3_cpu.ich_lr_el2[]` |
| ICH_HCR_EL2 | vGIC 控制 | KVM vGIC | `gicv3_cpu.ich_hcr_el2` |
| CNTVOFF_EL2 | Timer 偏移 | 内核时间管理 | `env->cp15.cntvoff_el2` |
| MDCR_EL2 | 调试控制 | KVM_SET_GUEST_DEBUG | `env->cp15.mdcr_el2` |
| CPTR_EL2 | 协处理器 trap | 内核配置 | `env->cp15.cptr_el[2]` |

---

## 附录 B: 参考文档

| 来源 | 内容 | 位置 |
|------|------|------|
| ARM Architecture Reference Manual (Armv9.6) | EL2/HCR/Stage-2/FEAT_VHE | knowledge-base: arm64/arm_architecture_reference_manual.md |
| GICv3/v4 Architecture Specification | ICH_LR/IMO/FMO/虚拟中断 | knowledge-base: arm64/gic_architecture_v3v4.md |
| QEMU Doc 09 (知识库) | 虚拟化扩展深度分析 | qemu/arm64/09-虚拟化扩展深度分析 |
| QEMU Doc 04 (知识库) | 中断注入分析 (KVM章节) | qemu/arm64/04-中断注入与处理深度分析 |
| Doc 98 (本系列) | KVM 内存管理 | darren/architecture/98-... |
| Doc 99 (本系列) | KVM ARM64 VM Exit | darren/architecture/99-... |
| Doc 86 (本系列) | KVM 加速器概览 | darren/architecture/86-... |
