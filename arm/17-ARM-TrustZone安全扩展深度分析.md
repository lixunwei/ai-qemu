# ARM TrustZone 与安全扩展深度分析

## 文档信息
- **QEMU 版本**: 11.0.50
- **分析目标**: ARM TrustZone、安全世界、RME (Realm Management Extension)、PSCI 在 QEMU 中的完整实现
- **核心源文件**:
  - `include/hw/arm/arm-security.h` — ARMSecuritySpace 定义
  - `target/arm/cpu.h` — 安全状态查询函数、SCR 位定义
  - `target/arm/helper.c` — SCR_EL3 处理、Secure Banking、异常路由、SMC/PSCI
  - `hw/arm/virt.c` — 安全内存、TrustZone 机器配置
  - `hw/misc/tz-mpc.c` — TrustZone 内存保护控制器
  - `hw/misc/tz-ppc.c` — TrustZone 外设保护控制器

---

## 第一部分：安全状态模型

### 1. ARMSecuritySpace 枚举

```c
// include/hw/arm/arm-security.h:12-29
typedef enum ARMSecuritySpace {
    ARMSS_Secure     = 0,   // 安全世界
    ARMSS_NonSecure  = 1,   // 非安全世界
    ARMSS_Root       = 2,   // RME: Root 安全状态
    ARMSS_Realm      = 3,   // RME: Realm 安全状态
} ARMSecuritySpace;

// 枚举值对应 GPI (Granule Protection Information) 低2位
// Root 和 Secure 都被视为"安全"（pre-v9 兼容）
static inline bool arm_space_is_secure(ARMSecuritySpace space)
{
    return space == ARMSS_Secure || space == ARMSS_Root;
}
```

### 2. 安全状态判断函数体系

```c
// cpu.h:2173-2265 — 安全状态查询API

// 核心：获取EL3以下的安全空间
ARMSecuritySpace arm_security_space_below_el3(CPUARMState *env);

// EL3以下是否安全
static inline bool arm_is_secure_below_el3(CPUARMState *env) {
    return arm_security_space_below_el3(env) == ARMSS_Secure;
}

// 当前CPU安全空间
ARMSecuritySpace arm_security_space(CPUARMState *env);

// 当前CPU是否安全
static inline bool arm_is_secure(CPUARMState *env) {
    return arm_space_is_secure(arm_security_space(env));
}

// 是否在EL3或Monitor模式
static inline bool arm_is_el3_or_mon(CPUARMState *env) {
    if (is_a64(env) && extract32(env->pstate, 2, 2) == 3) return true;
    if (!is_a64(env) && (cpsr & CPSR_M) == ARM_CPU_MODE_MON) return true;
    return false;
}
```

### 3. 无 EL3 时的安全状态

```c
// cpu.h:2241-2265 — CONFIG_USER_ONLY 或无EL3
static inline ARMSecuritySpace arm_security_space(CPUARMState *env) {
    return ARMSS_NonSecure;  // 始终非安全
}
static inline bool arm_is_secure(CPUARMState *env) { return false; }
```

---

## 第二部分：SCR_EL3 安全配置寄存器

### 4. SCR_EL3 关键位

| 位 | 名称 | 功能 |
|---|------|------|
| 0 | NS | Non-Secure：0=EL3以下为Secure, 1=Non-Secure |
| 1 | IRQ | IRQ 路由到 EL3 |
| 2 | FIQ | FIQ 路由到 EL3 |
| 3 | EA | 外部异常路由到 EL3 |
| 8 | HCE | HVC 指令使能 |
| 9 | SIF | Secure Instruction Fetch（禁止从NS内存取指） |
| 10 | RW | EL2 执行状态（0=AArch32, 1=AArch64）|
| 18 | EEL2 | 安全世界 EL2 使能 |
| 35 | NSE | RME: 与NS组合选择安全空间 |

**NS:NSE 组合**：
| NSE | NS | 安全空间 |
|-----|----| --------|
| 0 | 0 | Secure |
| 0 | 1 | Non-Secure |
| 1 | 0 | Root |
| 1 | 1 | Realm |

### 5. SCR_EL3 与 EL2 使能

```c
// cpu.h:2228-2239
static inline bool arm_is_el2_enabled_secstate(CPUARMState *env,
                                                ARMSecuritySpace space)
{
    assert(space != ARMSS_Root);
    return arm_feature(env, ARM_FEATURE_EL2)
           && (space != ARMSS_Secure || (env->cp15.scr_el3 & SCR_EEL2));
}
```

安全世界 EL2 需要 SCR_EL3.EEL2 = 1（Secure EL2 扩展，FEAT_SEL2）。

---

## 第三部分：安全/非安全寄存器分组 (Banking)

### 6. 分组寄存器机制

ARM TrustZone 的核心是寄存器分组（Banking）：同一个系统寄存器在 Secure 和 Non-Secure 世界有不同的物理拷贝。

```c
// helper.c:7538-7548 — add_cpreg_to_hashtable() 处理分组
{
    bool isbanked = r->bank_fieldoffsets[0] && r->bank_fieldoffsets[1];
    if (isbanked) {
        // 根据 ns 标志选择对应的 fieldoffset
        r->fieldoffset = r->bank_fieldoffsets[ns];
    }
    // ...
}
```

### 7. AArch32 分组寄存器注册

```c
// helper.c:7615-7649 — add_cpreg_to_hashtable_aa32()
static void add_cpreg_to_hashtable_aa32(ARMCPU *cpu, ARMCPRegInfo *r)
{
    switch (r->secure) {
    case ARM_CP_SECSTATE_NS:
        key |= CP_REG_AA32_NS_MASK;
        add_cpreg_to_hashtable(cpu, r, AA32, NS, key);
        break;
    case ARM_CP_SECSTATE_S:
        add_cpreg_to_hashtable(cpu, r, AA32, S, key);
        break;
    case ARM_CP_SECSTATE_BOTH:
        // 创建 "_S" 后缀的安全副本
        r_s = alloc_cpreg(r, "_S");
        add_cpreg_to_hashtable(cpu, r_s, AA32, S, key);
        key |= CP_REG_AA32_NS_MASK;
        add_cpreg_to_hashtable(cpu, r, AA32, NS, key);
        break;
    }
}
```

### 8. 分组寄存器示例

典型的分组寄存器定义：
- `VBAR_EL1`：非安全和安全世界各有独立的向量基址
- `SCTLR_EL1`：非安全和安全世界各有独立的系统控制
- `TTBR0_EL1/TTBR1_EL1`：独立的页表基址
- `MAIR_EL1`：独立的内存属性

使用 `.bank_fieldoffsets = { offsetof(..._ns), offsetof(..._s) }` 语法。

---

## 第四部分：安全中断路由

### 9. arm_phys_excp_target_el()

```c
// helper.c:8369-8421 — 物理异常目标EL计算
uint32_t arm_phys_excp_target_el(CPUState *cs, uint32_t excp_idx,
                                  uint32_t cur_el, bool secure)
{
    switch (excp_idx) {
    case EXCP_IRQ:
    case EXCP_NMI:
        scr = (SCR_EL3 & SCR_IRQ);   // SCR.IRQ=1 → 路由到EL3
        hcr = HCR_EL2 & HCR_IMO;     // HCR.IMO=1 → 路由到EL2
        break;
    case EXCP_FIQ:
        scr = (SCR_EL3 & SCR_FIQ);   // SCR.FIQ=1 → 路由到EL3
        hcr = HCR_EL2 & HCR_FMO;     // HCR.FMO=1 → 路由到EL2
        break;
    default: // SError
        scr = (SCR_EL3 & SCR_EA);    // SCR.EA=1 → 路由到EL3
        hcr = HCR_EL2 & HCR_AMO;    // HCR.AMO=1 → 路由到EL2
    }

    hcr |= (HCR_EL2 & HCR_TGE) != 0;  // TGE也强制到EL2

    // 查表确定目标EL
    target_el = target_el_table[is64][scr][rw][hcr][secure][cur_el];
}
```

### 10. 中断路由决策表

```
中断到来
  │
  ├── SCR_EL3.IRQ/FIQ/EA = 1?
  │     └── YES → 路由到 EL3
  │
  ├── HCR_EL2.IMO/FMO/AMO = 1 或 HCR_EL2.TGE = 1?
  │     └── YES → 路由到 EL2
  │
  └── 默认 → 路由到 EL1
```

GICv3 典型配置：
- **Group 0 (Secure)**：FIQ，SCR.FIQ=1 路由到 EL3
- **Group 1 NS**：IRQ，HCR.IMO=1 时虚拟化到 EL2
- **Group 1 S**：在安全世界作为 IRQ 处理

---

## 第五部分：SMC 指令与 EL3 入口

### 11. AArch32 SMC → Monitor 模式

```c
// helper.c:9036-9076
case EXCP_SMC:
    new_mode = ARM_CPU_MODE_MON;  // 进入Monitor模式
    addr = 0x08;                   // 向量偏移 0x08（SMC）
    mask = CPSR_A | CPSR_I | CPSR_F;  // 屏蔽所有异常
    offset = 0;
    break;

// ...

// Monitor模式入口：强制进入安全世界
if ((env->uncached_cpsr & CPSR_M) == ARM_CPU_MODE_MON) {
    env->cp15.scr_el3 &= ~SCR_NS;  // 清除NS位 → 安全世界
}
```

### 12. PSCI 处理

```c
// helper.c:9488-9492
// 检测是否为PSCI调用，如是则分发
arm_handle_psci_call(cpu);
```

PSCI (Power State Coordination Interface) 通过 SMC/HVC 实现：
- `CPU_ON`：唤醒指定 CPU
- `CPU_OFF`：关闭当前 CPU
- `SYSTEM_RESET`：系统复位
- `SYSTEM_OFF`：系统关机

### 13. virt 机器 PSCI 配置

```c
// hw/arm/virt.c:2676-2681
if (vms->secure && firmware_loaded) {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_DISABLED;  // 有固件时禁用QEMU PSCI
} else {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_SMC;  // EL3时用SMC
    // 或 QEMU_PSCI_CONDUIT_HVC                  // 无EL3时用HVC
}
```

---

## 第六部分：TrustZone 机器模型

### 14. virt 机器安全内存

```c
// hw/arm/virt.c:2649-2661
if (vms->secure) {
    // 创建安全内存容器
    secure_sysmem = g_new(MemoryRegion, 1);
    memory_region_init(secure_sysmem, OBJECT(machine), "secure-memory",
                       UINT64_MAX);
    vms->secure_sysmem = secure_sysmem;
}
```

### 15. 安全 RAM 创建

```c
// hw/arm/virt.c:2091-2103
static void create_secure_ram(VirtMachineState *vms, ...)
{
    memory_region_init_ram(secram, NULL, "virt.secure-ram", size, &error_fatal);
    memory_region_add_subregion(secure_sysmem, base, secram);
}
```

### 16. CPU 安全内存链接

```c
// hw/arm/virt.c:2785-2786
if (vms->secure) {
    object_property_set_link(cpuobj, "secure-memory",
                             OBJECT(vms->secure_sysmem), ...);
}
```

每个 CPU 有 `secure-memory` 属性指向安全内存地址空间，安全世界访问使用该地址空间。

### 17. virt 机器 TrustZone 完整配置

```
virt 机器 (secure=on):
  │
  ├── 普通内存空间 (sysmem)
  │   └── RAM、设备、GIC 等
  │
  ├── 安全内存空间 (secure-sysmem)
  │   ├── 安全 RAM (virt.secure-ram)
  │   ├── 安全 Flash
  │   └── 安全外设
  │
  ├── GIC: has-security-extensions = true
  │   ├── Group 0 (Secure) → FIQ
  │   └── Group 1 (Non-Secure) → IRQ
  │
  └── CPU: secure-memory → secure-sysmem
```

---

## 第七部分：TrustZone 外设保护

### 18. tz-mpc：内存保护控制器

```c
// hw/misc/tz-mpc.c:1-31
// ARM AHB5 TrustZone Memory Protection Controller 模拟

// 两个IOMMU索引
enum {
    IOMMU_IDX_S,    // 安全事务
    IOMMU_IDX_NS,   // 非安全事务
    IOMMU_NUM_INDEXES,
};
```

TZ-MPC 功能：
- 将内存区域划分为安全/非安全块
- 每个块有独立的安全属性
- 非安全访问安全区域被阻止
- 通过 IOMMU 接口实现地址重映射

### 19. tz-ppc：外设保护控制器

```c
// hw/misc/tz-ppc.c:24-30
static void tz_ppc_update_irq(TZPPC *s)
{
    bool level = s->irq_status && s->irq_enable;
    qemu_set_irq(s->irq, level);
}
```

TZ-PPC 功能：
- 控制外设端口的安全/非安全访问权限
- 每个端口可独立配置（安全/非安全/两者）
- 非法访问触发中断或返回 RAZ/WI
- 支持动态配置（运行时可改变端口安全属性）

---

## 第八部分：RME (Realm Management Extension)

### 20. RME 安全状态扩展

```c
// include/hw/arm/arm-security.h:12-23
// ARMv9 引入 Root 和 Realm 两个新安全状态

// SCR_EL3.NSE:NS 组合:
// 00 = Secure, 01 = Non-Secure, 10 = Root, 11 = Realm
```

### 21. RME 寄存器

```c
// cpu.h:578-581 — RME 特有寄存器
uint64_t gpccr_el3;   // Granule Protection Check Control Register
uint64_t gptbr_el3;   // Granule Protection Table Base Register
uint64_t mfar_el3;    // Memory Fault Address Register
```

### 22. RME 寄存器定义

```c
// helper.c:4972-4986 — rme_reginfo
// 注册 GPCCR_EL3、GPTBR_EL3、MFAR_EL3 系统寄存器
// 仅在 aa64_rme 特性存在时注册
```

### 23. Root 与 Realm 检查

```c
// helper.c:5030-5059
// 检查 ARMSS_Realm / ARMSS_Root 安全空间
// 用于访问控制决策
```

---

## 第九部分：安全启动流程

### 24. TF-A (Trusted Firmware-A) 引导

```
EL3 入口点 (Reset)
  │
  ├── QEMU 加载固件到安全Flash/RAM
  │   hw/arm/virt.c:1703-1732 — load_bios()
  │
  ├── CPU 从 EL3 开始执行
  │   ├── BL1 (Trusted ROM) → 初始化安全世界
  │   ├── BL2 (Trusted Boot) → 加载BL31/BL32/BL33
  │   ├── BL31 (EL3 Runtime) → 安全监控，处理SMC
  │   ├── BL32 (Secure OS, 可选) → OP-TEE 等
  │   └── BL33 (Normal World) → U-Boot/UEFI → Linux
  │
  └── PSCI conduit = DISABLED（固件自行处理）
```

### 25. 无固件直接引导

```
无安全固件时:
  │
  ├── QEMU 内置 PSCI 实现
  │   psci_conduit = SMC 或 HVC
  │
  ├── 内核直接从 EL2 或 EL1 启动
  │
  └── CPU_ON/OFF 由 QEMU 处理
```

---

## 第十部分：安全世界异常处理

### 26. AArch32 Monitor 模式异常向量

| 偏移 | 异常类型 |
|------|---------|
| 0x00 | Reset |
| 0x04 | Undefined (MON_TRAP) |
| 0x08 | SMC |
| 0x0C | Prefetch Abort |
| 0x10 | Data Abort |
| 0x18 | IRQ |
| 0x1C | FIQ |

向量基址来自 MVBAR (Monitor Vector Base Address Register)。

### 27. Monitor 模式入口处理

进入 Monitor 模式时：
1. 保存当前 CPSR 到 SPSR_mon
2. 设置 CPSR.M = ARM_CPU_MODE_MON
3. 屏蔽异常（A|I|F）
4. 清除 SCR.NS → 进入安全世界
5. 跳转到 MVBAR + offset

---

## 第十一部分：TrustZone 与虚拟化交互

### 28. Secure EL2 (FEAT_SEL2)

ARMv8.4 引入安全世界 EL2：
- `SCR_EL3.EEL2 = 1`：使能安全世界的 Hypervisor
- 安全世界的 Guest OS 可以在安全 EL1 运行
- 安全 Hypervisor 在安全 EL2 管理安全世界虚拟化

### 29. HCR_EL2 与安全状态

- `HCR_EL2.TGE`：影响非安全和安全世界的 EL0 行为
- 安全世界 EL2 使用独立的 `VSTCR_EL2`、`VSTTBR_EL2`
- Stage 2 翻译在安全世界有独立控制

---

## 第十二部分：完整安全状态切换流程

### 30. Non-Secure → Secure 切换

```
Non-Secure EL1 执行 SMC
  │
  ├── 异常进入 EL3
  │   ├── AArch64: 使用 VBAR_EL3 + offset
  │   └── AArch32: MVBAR + 0x08, 进入Monitor模式
  │
  ├── EL3 固件处理
  │   ├── 保存 Non-Secure 上下文
  │   ├── 设置 SCR_EL3.NS = 0 (切换到Secure)
  │   └── ERET 到 Secure EL1
  │
  └── Secure EL1 执行
      └── 使用Secure Banking寄存器
```

### 31. Secure → Non-Secure 切换

```
Secure EL1 完成处理
  │
  ├── SMC 返回 EL3
  │
  ├── EL3 固件处理
  │   ├── 保存 Secure 上下文
  │   ├── 设置 SCR_EL3.NS = 1 (切换到Non-Secure)
  │   └── ERET 到 Non-Secure EL1
  │
  └── Non-Secure EL1 继续执行
      └── 使用Non-Secure Banking寄存器
```

### 32. 安全状态与地址空间

| 安全状态 | SCR.NSE:NS | 物理地址空间 | 寄存器 Bank |
|----------|-----------|-------------|------------|
| Secure | 0:0 | Secure PA | S bank |
| Non-Secure | 0:1 | Non-Secure PA | NS bank |
| Root | 1:0 | Root PA | — (EL3 only) |
| Realm | 1:1 | Realm PA | — (RME only) |

---

## 附录

### A. TrustZone 关键源文件

| 文件 | 行数 | 内容 |
|------|------|------|
| `include/hw/arm/arm-security.h` | ~35 | ARMSecuritySpace 定义 |
| `target/arm/cpu.h` | ~3500 | 安全状态查询、SCR 位、EL2 使能 |
| `target/arm/helper.c` | ~10200 | SCR_EL3、Banking、异常路由、SMC、PSCI |
| `hw/arm/virt.c` | ~3000 | 安全内存、PSCI conduit、TF-A 引导 |
| `hw/misc/tz-mpc.c` | ~400 | TrustZone 内存保护控制器 |
| `hw/misc/tz-ppc.c` | ~300 | TrustZone 外设保护控制器 |
| `target/arm/cpu-irq.c` | ~240 | 中断安全路由 |

### B. QEMU TrustZone 启动选项

| 选项 | 说明 |
|------|------|
| `-machine virt,secure=on` | 启用 TrustZone |
| `-bios firmware.bin` | 加载安全固件到 Flash |
| `-device loader,file=bl1.bin,addr=0x0` | 加载 BL1 到安全 ROM |
| `-cpu max` | 启用所有安全特性 |
| 内核 `psci` 节点 | 配置 PSCI 通道（SMC/HVC）|
