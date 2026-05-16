# ARM TrustZone 安全组件模拟深度分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中 ARM TrustZone 安全模拟的完整实现，重点覆盖 TZASC 相关安全组件（MPC/PPC/MSC）、MMU 安全态处理、中断安全路由和世界切换机制。与 arm/17 的架构概览不同，本文聚焦于安全组件的硬件模拟细节。

**关键源文件：**
- `hw/misc/tz-mpc.c` — TrustZone 内存保护控制器（IOMMU 实现）
- `hw/misc/tz-ppc.c` — TrustZone 外设保护控制器
- `hw/misc/tz-msc.c` — TrustZone 主设备安全控制器
- `target/arm/ptw.c` — 页表遍历中的安全态处理
- `target/arm/helper.c` — 安全状态查询、中断路由、世界切换
- `hw/arm/virt.c` — virt 机器安全内存架构
- `include/hw/arm/arm-security.h` — 安全空间枚举

---

## 2. TrustZone 内存保护控制器（TZ-MPC）

### 2.1 IOMMU 安全索引

TZ-MPC 是 ARM AHB5 TrustZone Memory Protection Controller 的模拟，实现为 QEMU IOMMU 设备。

```c
// tz-mpc.c:24-31
enum {
    IOMMU_IDX_S,      // Secure 事务索引
    IOMMU_IDX_NS,     // Non-Secure 事务索引
    IOMMU_NUM_INDEXES,
};
```

每个内存事务携带安全属性（`MemTxAttrs.secure`），MPC 据此选择 IOMMU 索引来决定是否允许访问。

### 2.2 寄存器映射

```c
// tz-mpc.c:33-54
REG32(CTRL, 0x00)        // 控制：SEC_RESP(4), AUTOINC(8), LOCKDOWN(31)
REG32(BLK_MAX, 0x10)     // 最大块数
REG32(BLK_CFG, 0x14)     // 块大小配置
REG32(BLK_IDX, 0x18)     // LUT 索引
REG32(BLK_LUT, 0x1c)     // 查找表（每位对应一个块的 NS 属性）
REG32(INT_STAT, 0x20)     // 中断状态
REG32(INT_CLEAR, 0x24)    // 中断清除
REG32(INT_EN, 0x28)       // 中断使能
REG32(INT_INFO1, 0x2c)    // 违规地址
REG32(INT_INFO2, 0x30)    // 违规信息（HMASTER, HNONSEC, CFG_NS）
```

### 2.3 块级安全配置查找

```c
// tz-mpc.c:349-361 — tz_mpc_cfg_ns()
static inline bool tz_mpc_cfg_ns(TZMPC *s, hwaddr addr)
{
    hwaddr blknum = addr / s->blocksize;    // 计算块号
    hwaddr blkword = blknum / 32;           // LUT 字索引
    uint32_t blkbit = 1U << (blknum % 32);  // 字内位偏移
    return s->blk_lut[blkword] & blkbit;    // 返回该块的 NS 配置
}
```

**BLK_LUT 每位含义：**
- `0` = 该块为 Secure — 只允许 Secure 事务通过
- `1` = 该块为 Non-Secure — 只允许 Non-Secure 事务通过

### 2.4 IOMMU 翻译核心

```c
// tz-mpc.c:425-453 — tz_mpc_translate()
```

翻译逻辑：
1. 构造 `IOMMUTLBEntry`，地址对齐到 blocksize 边界
2. 比较 `tz_mpc_cfg_ns(addr)` 与 `iommu_idx == IOMMU_IDX_NS`
3. **匹配**：`ret.target_as = &s->downstream_as`（放行到下游内存）
4. **不匹配**：`ret.target_as = &s->blocked_io_as`（导向阻塞处理）

```c
// tz-mpc.c:445
ok = tz_mpc_cfg_ns(s, addr) == (iommu_idx == IOMMU_IDX_NS);
ret.target_as = ok ? &s->downstream_as : &s->blocked_io_as;
```

### 2.5 属性到索引映射

```c
// tz-mpc.c:455-465 — tz_mpc_attrs_to_index()
static int tz_mpc_attrs_to_index(IOMMUMemoryRegion *iommu, MemTxAttrs attrs)
{
    // unspecified 属性视为 Secure（如 ROM 加载时）
    return (attrs.unspecified || attrs.secure) ? IOMMU_IDX_S : IOMMU_IDX_NS;
}
```

### 2.6 违规处理

```c
// tz-mpc.c:363-386 — tz_mpc_handle_block()
```

被阻塞的事务处理：
1. 首次违规：捕获地址到 `INT_INFO1`，记录 `HNONSEC`/`CFG_NS` 到 `INT_INFO2`
2. 触发 IRQ（`int_stat |= R_INT_STAT_IRQ_MASK`）
3. 后续违规在 Guest 清除中断前不再捕获
4. 根据 `CTRL.SEC_RESP`：`MEMTX_ERROR`（总线错误）或 `MEMTX_OK`（RAZ/WI）

### 2.7 realize 与 IOMMU 注册

```c
// tz-mpc.c:494-530 — tz_mpc_realize()
```

实现流程：
1. 获取下游内存区域大小
2. 调用 `memory_region_init_iommu()` 创建上游 IOMMU 区域（类型 `TYPE_TZ_MPC_IOMMU_MEMORY_REGION`）
3. `blocksize` = IOMMU 最小页大小（通常 4KB）
4. 创建 `blocked_io_as` 用于阻塞事务的 RAZ/WI 处理
5. 初始化 LUT（默认全 0 = 全 Secure）

---

## 3. TrustZone 外设保护控制器（TZ-PPC）

### 3.1 访问检查

```c
// tz-ppc.c:80-103 — tz_ppc_check()
static bool tz_ppc_check(TZPPC *s, int n, MemTxAttrs attrs)
{
    // 三层检查：
    // 1. nonsec_mask 抑制安全检查（跳过该端口的安全过滤）
    // 2. cfg_nonsec 与 attrs.secure 不匹配 → 阻塞
    // 3. 用户模式访问且 cfg_ap == 0 → 阻塞
    if ((attrs.secure == s->cfg_nonsec[n] && !(s->nonsec_mask & (1 << n))) ||
        (attrs.user && !s->cfg_ap[n])) {
        s->irq_status = true;
        tz_ppc_update_irq(s);
        return false;
    }
    return true;
}
```

**PPC vs MPC 的区别：**

| 特性 | MPC | PPC |
|------|-----|-----|
| 保护对象 | 内存区域（按块） | 外设端口（按端口） |
| 实现方式 | IOMMU（地址翻译） | 代理读写（转发/阻塞） |
| 粒度 | 块级（4KB~） | 端口级（整个外设） |
| 安全配置 | BLK_LUT 位图 | cfg_nonsec[]/cfg_ap[] |
| 用户态检查 | 无 | cfg_ap 控制用户态访问 |

### 3.2 PPC 代理读写

```c
// tz-ppc.c:105-130
static MemTxResult tz_ppc_read(void *opaque, hwaddr addr, uint64_t *pdata,
                               unsigned size, MemTxAttrs attrs)
{
    if (!tz_ppc_check(s, n, attrs)) {
        if (s->cfg_sec_resp) return MEMTX_ERROR;  // 总线错误
        else { *pdata = 0; return MEMTX_OK; }     // RAZ（Read As Zero）
    }
    // 通过：转发到下游外设
    data = address_space_ldub/lduw_le/ldl_le(as, addr, attrs, &res);
}
```

### 3.3 PPC 端口模型

每个 PPC 支持 `TZ_NUM_PORTS` 个端口，每端口独立的安全配置：
- `cfg_nonsec[n]` — 端口安全属性（0=Secure, 1=NonSecure）
- `cfg_ap[n]` — 用户态访问权限（0=禁止, 1=允许）
- `nonsec_mask` — 位掩码，每位控制对应端口是否跳过安全检查

---

## 4. TrustZone 主设备安全控制器（TZ-MSC）

### 4.1 MSCAction 决策

```c
// tz-msc.c:66-71
typedef enum MSCAction {
    MSCBlockAbort,       // 阻塞并产生总线错误
    MSCBlockRAZWI,       // 阻塞但 RAZ/WI
    MSCAllowSecure,      // 允许，标记为 Secure 事务
    MSCAllowNonSecure,   // 允许，标记为 NonSecure 事务
} MSCAction;
```

### 4.2 IDAU 集成

```c
// tz-msc.c:73-122 — tz_msc_check()
static MSCAction tz_msc_check(TZMSC *s, hwaddr addr)
{
    IDAUInterfaceClass *iic = IDAU_INTERFACE_GET_CLASS(s->idau);
    iic->check(ii, addr, &idau_region, &idau_exempt, &idau_ns, &idau_nsc);

    if (idau_exempt)
        return s->cfg_nonsec ? MSCAllowNonSecure : MSCAllowSecure;
    if (idau_ns)
        return MSCAllowNonSecure;           // NS 区域 → 始终 NS
    if (!s->cfg_nonsec)
        return MSCAllowSecure;              // Secure 主机访问 Secure 区 → OK
    // NS 主机访问 Secure 区 → 阻塞
    return s->cfg_sec_resp ? MSCBlockAbort : MSCBlockRAZWI;
}
```

**MSC 特点：** 通过 IDAU（Implementation Defined Attribution Unit）接口判断地址的安全属性，适用于 Cortex-M 系列的安全地址映射。

---

## 5. 安全内存架构

### 5.1 virt 机器的双地址空间

```c
// virt.c:2649-2661
if (vms->secure) {
    secure_sysmem = g_new(MemoryRegion, 1);
    memory_region_init(secure_sysmem, ..., "secure-memory", UINT64_MAX);
    // Secure 视图 = 系统内存（低优先级）+ Secure 专有设备（高优先级）
    memory_region_add_subregion_overlap(secure_sysmem, 0, sysmem, -1);
}
```

**架构设计：**
```
secure-memory (容器, 最高优先级)
├── secure-ram (virt.secure-ram, 高优先级)
├── secure-flash (高优先级)
├── secure-uart (高优先级)
└── sysmem (系统内存, 优先级 -1, 包含所有 NS 设备)
    ├── RAM
    ├── MMIO 设备
    └── ...
```

- **Secure 世界**使用 `secure_sysmem` 地址空间 → 能看到 Secure 专有设备 + NS 设备
- **NonSecure 世界**使用 `sysmem` 地址空间 → 只能看到 NS 设备
- `memory_region_add_subregion_overlap(..., -1)` 确保 NS 设备在 Secure 地址空间中作为"背景"

### 5.2 Secure RAM

```c
// virt.c:2091-2103 — create_secure_ram()
memory_region_init_ram(secram, NULL, "virt.secure-ram", size, &error_fatal);
memory_region_add_subregion(secure_sysmem, base, secram);
```

Secure RAM 只映射到 `secure_sysmem`，NS 世界无法访问。

### 5.3 PSCI Conduit 选择

```c
// virt.c:2676-2682
if (vms->secure && firmware_loaded) {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_DISABLED;  // 固件自己实现 PSCI
} else if (vms->virt) {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_SMC;       // 有 EL2 用 SMC
} else {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_HVC;       // 否则用 HVC
}
```

---

## 6. MMU 安全态处理

### 6.1 S1Translate 安全上下文

```c
// ptw.c:22-90 — S1Translate 结构
typedef struct S1Translate {
    ARMMMUIdx in_mmu_idx;       // 使用哪个 TTBR/TCR
    ARMMMUIdx in_ptw_idx;       // PTW 读取使用的 mmuidx

    ARMSecuritySpace in_space;  // 本次遍历的安全空间
    ARMSecuritySpace cur_space; // 可被 NSTable 降级到 NonSecure
    ARMSecuritySpace out_space; // 翻译结果的安全空间

    bool in_debug;
    bool in_at;
    bool in_s1_is_el0;
    uint8_t in_prot_check;
    ...
} S1Translate;
```

**三个安全空间字段的含义：**
- `in_space`：初始安全空间（由 CPU 当前状态决定）
- `cur_space`：遍历过程中可被页表的 NSTable 位降级
- `out_space`：最终翻译结果的安全空间

### 6.2 NSTable 降级

当 Stage-1 页表中的 Table Descriptor 设置了 NSTable=1 时：
- `cur_space` 从 Secure 降级为 NonSecure
- 后续级别的页表查找将使用 NonSecure 的物理地址空间
- `in_ptw_idx` 也相应切换到 NonSecure 的 Stage-2 或物理索引

### 6.3 Stage-2 安全空间选择

```c
// ptw.c:193-224 — ptw_idx_for_stage_2()
```

根据 `arm_security_space_below_el3()` 选择 Stage-2 PTW 的物理索引：
- Secure → `ARMMMUIdx_Phys_S`
- NonSecure → `ARMMMUIdx_Phys_NS`
- Realm → `ARMMMUIdx_Phys_Realm`

### 6.4 TTBR Banking

```c
// ptw.c 中的 regime_ttbr()
```

不同异常级别和安全状态使用不同的 TTBR：
- EL1 Secure → `TTBR0_EL1` / `TTBR1_EL1`（Secure bank）
- EL1 NonSecure → `TTBR0_EL1` / `TTBR1_EL1`（NonSecure bank）
- EL3 → `TTBR0_EL3`（只有 TTBR0，无 TTBR1）
- EL2 → `TTBR0_EL2` / `TTBR1_EL2`（VHE 下有两个）

---

## 7. 中断安全路由

### 7.1 target_el_table 查找

```c
// helper.c:8309-8340 — target_el_table[2][2][2][2][2][4]
// 维度：[64bit EL3][SCR mask][SCR exec][HCR mask][Secure][Current EL]
```

ARM 中断路由由 SCR_EL3 和 HCR_EL2 控制：
- `SCR_EL3.IRQ` = 1 → IRQ 路由到 EL3
- `SCR_EL3.FIQ` = 1 → FIQ 路由到 EL3
- `SCR_EL3.EA` = 1 → 外部中止路由到 EL3
- `HCR_EL2.IMO/FMO/AMO` → 在非 EL3 时将中断路由到 EL2

```c
// helper.c:8369-8405 — arm_phys_excp_target_el()
uint32_t arm_phys_excp_target_el(CPUState *cs, uint32_t excp_idx,
                                 uint32_t cur_el, bool secure)
{
    // 从 SCR_EL3/HCR_EL2 提取路由位
    // 查表 target_el_table 获取目标 EL
    // 确保 target_el >= cur_el
}
```

### 7.2 GICv3 安全组

| 组 | 安全属性 | 默认路由 | 用途 |
|----|----------|----------|------|
| Group 0 | Secure | FIQ（EL3） | Secure 固件中断 |
| Group 1 Secure | Secure | IRQ（Secure EL1） | Secure OS 中断 |
| Group 1 Non-Secure | Non-Secure | IRQ（NS EL1/EL2） | 普通 OS 中断 |

```c
// arm_gicv3_dist.c:59-79
```

`GICD_IGROUPR` 和 `GICD_IGRPMODR` 寄存器控制每个中断的组分配：
- Group 0：`IGROUPR=0, IGRPMODR=0`
- Group 1 Secure：`IGROUPR=0, IGRPMODR=1`
- Group 1 NS：`IGROUPR=1`
- NS 访问看不到 Group 0/1S 的配置（RAZ/WI）

### 7.3 CPU 接口安全态

```c
// arm_gicv3_cpuif.c:40-106
```

GICv3 CPU 接口使用 `arm_is_secure_below_el3(env)` 选择安全 bank：
- `ICC_SRE_EL3` — 控制 EL3 的系统寄存器接口
- Secure 和 NonSecure 各有独立的优先级掩码（`ICC_PMR_EL1`）
- Secure 中断在 NS 世界中被自动抑制

---

## 8. 世界切换机制

### 8.1 SMC 进入 Monitor 模式（AArch32）

```c
// helper.c:9036-9076
case EXCP_SMC:
    new_mode = ARM_CPU_MODE_MON;     // 目标模式：Monitor
    addr = 0x08;                      // Monitor 向量表偏移
    mask = CPSR_A | CPSR_I | CPSR_F; // 屏蔽所有异步异常

// 进入 Monitor 时清除 SCR.NS → 强制 Secure 世界
if ((env->uncached_cpsr & CPSR_M) == ARM_CPU_MODE_MON) {
    env->cp15.scr_el3 &= ~SCR_NS;
}
```

### 8.2 寄存器 Banking（switch_mode）

```c
// helper.c:8280-8307 — switch_mode()
{
    // 1. FIQ 模式特殊处理：R8-R12 独立 bank
    if (old_mode == ARM_CPU_MODE_FIQ)
        memcpy(env->fiq_regs, env->regs + 8, 5 * sizeof(uint32_t));

    // 2. 保存旧模式的 R13/SPSR
    i = bank_number(old_mode);
    env->banked_r13[i] = env->regs[13];
    env->banked_spsr[i] = env->spsr;

    // 3. 恢复新模式的 R13/SPSR
    i = bank_number(mode);
    env->regs[13] = env->banked_r13[i];
    env->spsr = env->banked_spsr[i];

    // 4. R14 单独 banking（使用不同的 bank 号方案）
    env->banked_r14[r14_bank_number(old_mode)] = env->regs[14];
    env->regs[14] = env->banked_r14[r14_bank_number(mode)];
}
```

### 8.3 AArch64 SMC 处理

AArch64 下 SMC 异常导致跳转到 EL3：
1. 保存 PSTATE 到 `SPSR_EL3`
2. 保存 PC 到 `ELR_EL3`
3. 设置 `SCR_EL3.NS` 根据异常来源（NS 世界 SMC → NS=1 保持）
4. 跳转到 `VBAR_EL3 + offset`
5. EL3 固件（TF-A）根据 SMC 功能号分发

### 8.4 ERET 返回

从 EL3 返回时：
1. 恢复 PSTATE 从 `SPSR_EL3`
2. 恢复 PC 从 `ELR_EL3`
3. `SCR_EL3.NS` 决定返回到哪个世界
4. 地址空间自动切换（`secure_sysmem` ↔ `sysmem`）

---

## 9. 安全组件在 SoC 中的连接

### 9.1 典型连接拓扑

```
CPU (Secure/NonSecure)
  │
  ├─[Secure addr space]── TZ-MPC ──→ Secure SRAM/Flash
  │                         │
  │                         └─[blocked]─→ blocked_io_as (RAZ/WI 或 Bus Error)
  │
  ├─[NS addr space]──────→ Normal Memory
  │
  └─ GIC ─┬─ Group 0 (FIQ → EL3)
           ├─ Group 1S (IRQ → Secure EL1)
           └─ Group 1NS (IRQ → NS EL1/EL2)

TZ-PPC ──→ 外设端口 0..N
  │         ├─ UART (cfg_nonsec=0 → Secure only)
  │         ├─ GPIO (cfg_nonsec=1 → NS accessible)
  │         └─ ...
  │
  └─[blocked]─→ Bus Error 或 RAZ/WI

TZ-MSC ── IDAU ──→ 主设备安全属性判定
```

### 9.2 MPC/PPC/MSC 设备类型

```c
// TYPE_TZ_MPC = "tz-mpc"
// TYPE_TZ_MPC_IOMMU_MEMORY_REGION = "tz-mpc-iommu-memory-region"
// TYPE_TZ_PPC = "tz-ppc"
// TYPE_TZ_MSC = "tz-msc"
```

---

## 10. 完整安全事务路径

### 10.1 Secure CPU 访问 Secure 内存

```
1. CPU 发起内存访问（MemTxAttrs.secure=1）
2. MMU Stage-1 PTW（in_space=Secure, TTBR_EL1 Secure bank）
3. 翻译后地址进入 secure_sysmem 地址空间
4. 经过 TZ-MPC:
   - tz_mpc_attrs_to_index() → IOMMU_IDX_S
   - tz_mpc_translate(): cfg_ns=0, idx=S → 匹配 → downstream_as
5. 到达 Secure SRAM → 访问成功
```

### 10.2 NonSecure CPU 试图访问 Secure 内存

```
1. CPU 发起内存访问（MemTxAttrs.secure=0）
2. MMU Stage-1 PTW（in_space=NonSecure, TTBR_EL1 NS bank）
3. 翻译后地址进入 sysmem 地址空间
4. 经过 TZ-MPC:
   - tz_mpc_attrs_to_index() → IOMMU_IDX_NS
   - tz_mpc_translate(): cfg_ns=0, idx=NS → 不匹配 → blocked_io_as
5. tz_mpc_handle_block():
   - 记录 INT_INFO1(地址), INT_INFO2(HNONSEC=1, CFG_NS=0)
   - 触发 IRQ
   - CTRL.SEC_RESP=1 → MEMTX_ERROR（总线错误）
```

### 10.3 NS 外设访问通过 PPC

```
1. CPU 访问外设地址（MemTxAttrs.secure=0）
2. tz_ppc_check(s, port, attrs):
   - cfg_nonsec[port]=1（NS 端口）, attrs.secure=0 → 不等 → 通过
3. tz_ppc_read() 转发到下游 AddressSpace
4. 外设正常响应
```

---

## 11. 小结

| 组件 | 保护对象 | 实现方式 | 安全配置 | 违规响应 |
|------|----------|----------|----------|----------|
| **TZ-MPC** | 内存块 | IOMMU 翻译 | BLK_LUT 位图 | IRQ + 总线错误/RAZ |
| **TZ-PPC** | 外设端口 | 代理读写 | cfg_nonsec[]/cfg_ap[] | IRQ + 总线错误/RAZ |
| **TZ-MSC** | 主设备 | IDAU 查询 | cfg_nonsec | IRQ + 总线错误/RAZ |
| **MMU PTW** | 页表 | S1Translate | in_space/cur_space/out_space | 翻译错误 |
| **GICv3** | 中断 | 安全组 | IGROUPR/IGRPMODR | 路由到不同 EL |
| **virt 机器** | 地址空间 | 双 MemoryRegion | secure_sysmem 容器 | 不可见 |

**关键设计原则：**
1. MPC 用 IOMMU 索引（S/NS）实现地址翻译级的安全隔离
2. PPC 用代理读写实现端口级的安全过滤
3. PTW 中 `cur_space` 可被 NSTable 位动态降级
4. virt 机器用容器 + 优先级覆盖实现 Secure/NS 地址空间分离
5. 中断通过 GICv3 安全组 + SCR_EL3 路由位实现安全隔离
