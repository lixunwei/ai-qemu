# ARM64 Debug/Breakpoint/Watchpoint 与 RAS 寄存器深度分析

## 1. 概述

ARM64 调试架构提供硬件断点（Breakpoint）、数据观察点（Watchpoint）、单步执行和调试通信通道等功能。QEMU 在 TCG 模式下完整实现了这些调试机制，包括寄存器定义、匹配逻辑和异常路由。此外，QEMU 实现了最小化的 RAS（Reliability, Availability, Serviceability）框架，仅提供错误报告寄存器而不实现错误记录。

**关键源文件：**
- `target/arm/debug_helper.c` — 调试寄存器定义与访问控制（~525行）
- `target/arm/tcg/debug.c` — TCG 调试异常处理与 BP/WP 匹配（~770行）
- `target/arm/helper.c` — RAS 寄存器、MDCR_EL2/EL3 定义

---

## 2. 调试访问控制 — 三级陷阱体系

### 2.1 access_tdosa — OS 锁相关寄存器

```c
// debug_helper.c:21-36
// 控制 OSLAR_EL1, OSLSR_EL1, OSDLR_EL1 等 OS 级调试锁寄存器
// 陷阱条件:
//   EL < 2 且 (MDCR_EL2.TDOSA=1 或 MDCR_EL2.TDE=1 或 HCR_EL2.TGE=1)
//     → CP_ACCESS_TRAP_EL2
//   EL < 3 且 MDCR_EL3.TDOSA=1
//     → CP_ACCESS_TRAP_EL3
```

### 2.2 access_tda — 通用调试寄存器

```c
// debug_helper.c:63-78
// 控制所有 DBGBVRn/BCRn、DBGWVRn/WCRn、MDSCR_EL1 等
// 陷阱条件:
//   EL < 2 且 (MDCR_EL2.TDA=1 或 MDCR_EL2.TDE=1 或 HCR_EL2.TGE=1)
//     → CP_ACCESS_TRAP_EL2
//   EL < 3 且 MDCR_EL3.TDA=1
//     → CP_ACCESS_TRAP_EL3
```

### 2.3 access_tdcc — 调试通信通道

```c
// debug_helper.c:97-121
// 控制 DBGDTR_EL0、DBGDTRRX/TX_EL0、MDCCSR_EL0 等
// 额外检查:
//   EL0 时 MDSCR_EL1.TDCC=1 → CP_ACCESS_TRAP_EL1
//   EL < 2 时 MDCR_EL2.TDCC=1（需 FEAT_FGT）→ CP_ACCESS_TRAP_EL2
//   MDCR_EL3.TDCC=1（需 FEAT_FGT）→ CP_ACCESS_TRAP_EL3
```

---

## 3. 核心调试寄存器

### 3.1 MDSCR_EL1 — 监控调试系统控制

```c
// debug_helper.c:193-199
{ .name = "MDSCR_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 2, .opc1 = 0, .crn = 0, .crm = 2, .opc2 = 2,
  .access = PL1_RW, .accessfn = access_tda,
  .fgt = FGT_MDSCR_EL1,
  .fieldoffset = offsetof(CPUARMState, cp15.mdscr_el1) },

// 关键位域:
//   MDE  [15] — 监控调试事件使能（断点/观察点全局开关）
//   SS   [0]  — 软件单步使能
//   TDCC [12] — EL0 调试通信通道陷阱
//   KDE  [13] — 内核调试事件使能
```

### 3.2 MDCR_EL2 / MDCR_EL3 — 调试陷阱路由

```c
// helper.c:6830-6836 — MDCR_EL2
// 陷阱位: TDA, TDOSA, TDE, TDCC, TPM, TPMCR, HPMN 等
// TDE=1: 所有调试异常路由到 EL2

// helper.c:3637-3643 — MDCR_EL3
// 陷阱位: TDA, TDOSA, TDCC, TPM, SCCD, SDD 等
// SDD: 安全调试禁用
```

### 3.3 OSLAR / OSLSR / OSDLR — OS 调试锁

```c
// debug_helper.c:256-274
// OSLAR_EL1: 写入锁定/解锁 OS Lock
//   写 1: 锁定（OSLSR.OSLK=1）→ 限制调试寄存器访问
//   写 0: 解锁
// OSLSR_EL1: 只读状态（resetvalue=10, 即 OSLK=0, OSLM=1）
// OSDLR_EL1: 双重锁（Dummy 实现，Linux 32-bit 会读取）
```

### 3.4 DBGDTR / DBGDTRRX / DBGDTRTX — 调试通信通道

```c
// debug_helper.c:222-235
// 全部实现为 ARM_CP_CONST = 0（RAZ/WI）
// QEMU 不实现外部调试接口（External Debug Channel）
// 但保留寄存器定义以避免 Guest SIGILL
```

---

## 4. 断点寄存器（Breakpoint）

### 4.1 寄存器定义

```c
// debug_helper.c:477-499 — 动态注册循环
for (i = 0; i < brps; i++) {
    // DBGBVRn_EL1: 断点地址值
    { .name = "DBGBVR%d_EL1", .opc0 = 2, .opc1 = 0, .crn = 0, .crm = i, .opc2 = 4,
      .access = PL1_RW, .accessfn = access_tda,
      .fgt = FGT_DBGBVRN_EL1,
      .fieldoffset = offsetof(CPUARMState, cp15.dbgbvr[i]),
      .writefn = dbgbvr_write },
    
    // DBGBCRn_EL1: 断点控制
    { .name = "DBGBCR%d_EL1", .opc0 = 2, .opc1 = 0, .crn = 0, .crm = i, .opc2 = 5,
      .access = PL1_RW, .accessfn = access_tda,
      .fgt = FGT_DBGBCRN_EL1,
      .fieldoffset = offsetof(CPUARMState, cp15.dbgbcr[i]),
      .writefn = dbgbcr_write },
}
```

### 4.2 DBGBCRn 控制位

```
DBGBCRn_EL1:
  E     [0]    — 断点使能
  PMC   [2:1]  — 特权匹配控制 (EL0/EL1)
  BAS   [8:5]  — 字节地址选择（16位指令断点）
  HMC   [13]   — 高特权模式控制（EL2/EL3）
  SSC   [15:14] — 安全状态控制
  LBN   [19:16] — 链接断点编号
  BT    [23:20] — 断点类型
    0: 地址匹配（未链接）    1: 地址匹配（已链接）
    2: 上下文 ID 匹配       4/5: 地址不匹配（保留 AArch64）
```

### 4.3 写回调

```c
// debug_helper.c:371-400
// dbgbvr_write(): raw_write + hw_breakpoint_update(cpu, i)
// dbgbcr_write(): 
//   BAS[3]=BAS[2], BAS[1]=BAS[0]（只读拷贝位）
//   raw_write + hw_breakpoint_update(cpu, i)
```

---

## 5. 观察点寄存器（Watchpoint）

### 5.1 寄存器定义

```c
// debug_helper.c:501-523 — 动态注册循环
for (i = 0; i < wrps; i++) {
    // DBGWVRn_EL1: 观察点地址值
    { .fieldoffset = offsetof(CPUARMState, cp15.dbgwvr[i]),
      .writefn = dbgwvr_write },
    
    // DBGWCRn_EL1: 观察点控制
    { .fieldoffset = offsetof(CPUARMState, cp15.dbgwcr[i]),
      .writefn = dbgwcr_write },
}
```

### 5.2 DBGWCRn 控制位

```
DBGWCRn_EL1:
  E     [0]     — 观察点使能
  PAC   [2:1]   — 特权访问控制
  LSC   [4:3]   — 加载/存储控制 (01=读, 10=写, 11=读写)
  BAS   [12:5]  — 字节地址选择
  HMC   [13]    — 高特权模式控制
  SSC   [15:14] — 安全状态控制
  LBN   [19:16] — 链接断点编号
  WT    [20]    — 是否链接到断点
  MASK  [28:24] — 地址掩码（0=使用 BAS, 3-31=对齐掩码）
```

### 5.3 hw_watchpoint_update() — QEMU 内部映射

```c
// tcg/debug.c:546-633
void hw_watchpoint_update(ARMCPU *cpu, int n)
{
    // 1. 清除旧的 QEMU 观察点
    // 2. 检查 WCR.E（未使能则返回）
    // 3. 根据 LSC 设置 flags（BP_MEM_READ/WRITE/ACCESS）
    // 4. 计算监控范围:
    //    MASK != 0: 对齐区域（最大 2GB）
    //    MASK == 0: BAS 字节选择（最大 8 字节）
    // 5. 调用 cpu_watchpoint_insert() 注册到 QEMU 核心
}
// 将 ARM 硬件观察点映射为 QEMU 软件观察点
// QEMU 每次内存访问时检查地址匹配
```

### 5.4 hw_breakpoint_update() — 断点类型处理

```c
// tcg/debug.c:652-736
void hw_breakpoint_update(ARMCPU *cpu, int n)
{
    // BT (断点类型) 分类:
    //   0/1: 地址匹配 → cpu_breakpoint_insert()
    //   2:   上下文 ID 匹配 → LOG_UNIMP（未实现）
    //   4/5: 地址不匹配 → LOG_UNIMP（未实现）
    //   8/10: VMID 匹配 → LOG_UNIMP（未实现）
    //   3/9/11: 链接上下文 → 不生成事件（由链接者处理）
    
    // BAS 处理:
    //   0b0000 → 禁用    0b0011 → addr
    //   0b1100 → addr+2  0b1111 → addr
}
```

---

## 6. 断点/观察点匹配逻辑

### 6.1 bp_wp_matches() — 核心匹配函数

```c
// tcg/debug.c:255-352
static bool bp_wp_matches(ARMCPU *cpu, int n, bool is_wp)
{
    // 1. 检查命中标志（WP: BP_WATCHPOINT_HIT, BP: PC 匹配）
    // 2. 解析控制位: PAC, HMC, SSC
    // 3. 安全状态过滤:
    //    SSC=1/3: 仅非安全匹配    SSC=2: 仅安全匹配
    // 4. EL 过滤:
    //    EL2/3: 需 HMC=1
    //    EL1: 需 PAC[0]=1    EL0: 需 PAC[1]=1
    // 5. 链接检查: WT=1 时调用 linked_bp_matches()
    //    链接断点必须同时命中才触发
    // 6. LDRT/STRT 特殊处理: 非特权访问指令视为 EL0
}
```

### 6.2 check_watchpoints() / arm_debug_check_breakpoint()

```c
// tcg/debug.c:354-420
// 全局前置条件:
//   1. MDSCR_EL1.MDE [15] = 1（调试事件使能）
//   2. arm_generate_debug_exceptions(env) = true
// 断点额外检查:
//   3. 单步异常优先于断点（SS active-pending 时抑制 BP）
//   4. PC 对齐错误优先于断点
// 遍历所有 N 个 BP/WP 调用 bp_wp_matches()
```

---

## 7. 调试异常路由

### 7.1 arm_debug_target_el() — 目标 EL 决定

```c
// tcg/debug.c:18-41
static int arm_debug_target_el(CPUARMState *env)
{
    // M-profile: 固定 EL1
    // 路由到 EL2 条件: HCR_EL2.TGE=1 或 MDCR_EL2.TDE=1
    // 路由到 EL3 条件: AArch32 EL3 + Secure 状态
    // 默认: EL1
}
```

### 7.2 raise_exception_debug() — 异常生成

```c
// tcg/debug.c:47-61
void raise_exception_debug(CPUARMState *env, uint32_t excp, uint32_t syndrome)
{
    int debug_el = arm_debug_target_el(env);
    // 断言: debug_el >= cur_el（不会降级异常）
    // 同 EL 时设置 syndrome EC 位
    raise_exception(env, excp, syndrome, debug_el);
}
```

### 7.3 arm_debug_excp_handler() — 异常入口

```c
// tcg/debug.c:464-508
void arm_debug_excp_handler(CPUState *cs)
{
    // 观察点命中:
    //   设置 exception.fsr = arm_debug_exception_fsr()
    //   设置 exception.vaddress = wp->hitaddr
    //   raise_exception_debug(EXCP_DATA_ABORT, syn_watchpoint())
    
    // 断点命中:
    //   GDB 断点优先（返回给 GDB）
    //   非 CPU 断点忽略（单步产生的内部异常）
    //   CPU 断点:
    //     fsr = arm_debug_exception_fsr()
    //     vaddress = 0（UNKNOWN，清零避免信息泄露）
    //     raise_exception_debug(EXCP_PREFETCH_ABORT, syn_breakpoint())
}
```

---

## 8. 调试异常使能条件

### 8.1 AArch64 调试异常生成

```c
// tcg/debug.c:63-92 — aa64_generate_debug_exceptions()
// EL3: 仅当 MDSCR_EL1.KDE=1 且 OS 未双重锁定
// EL2: debug_el >= 2 时使能（MDCR_EL2.TDE 控制）
// EL1: debug_el >= 1 时使能
// EL0: 总是使能（如果 debug_el >= 1）
// D 位（PSTATE.D）抑制所有调试异常
```

---

## 9. RAS 寄存器 — 最小化实现

### 9.1 设计决策

```c
// helper.c:4545-4567
// QEMU 实现"最小 RAS"：零个错误记录（Error Record）
// 所有 ERX* 寄存器（ERXADDR/ERXCTLR/ERXFR/ERXMISC*/ERXSTATUS）
// 和 ERRSELR_EL1 均 UNDEFINED（不定义 = 未实现）
// UNDEF 优先级高于 FGT 陷阱，无需注册 FGT 位
```

### 9.2 已实现的 RAS 寄存器

```c
// helper.c:4568-4586 — minimal_ras_reginfo[]

// DISR_EL1: 延迟中断状态寄存器
{ .name = "DISR_EL1", .opc0 = 3, .opc1 = 0, .crn = 12, .crm = 1, .opc2 = 1,
  .access = PL1_RW,
  .readfn = disr_read, .writefn = disr_write },
  // 记录延迟的 SError 异常信息

// ERRIDR_EL1: 错误记录 ID
{ .name = "ERRIDR_EL1", .opc0 = 3, .opc1 = 0, .crn = 5, .crm = 3, .opc2 = 0,
  .access = PL1_R, .type = ARM_CP_CONST, .resetvalue = 0 },
  // 返回 0: 表示零个错误记录

// VDISR_EL2: 虚拟 DISR（Hypervisor 注入虚拟 SError）
{ .name = "VDISR_EL2", .opc0 = 3, .opc1 = 4, .crn = 12, .crm = 1, .opc2 = 1,
  .access = PL2_RW },

// VSESR_EL2: 虚拟 SError 异常综合征
{ .name = "VSESR_EL2", .opc0 = 3, .opc1 = 4, .crn = 5, .crm = 2, .opc2 = 3,
  .access = PL2_RW },
```

### 9.3 RAS 特性标志

```c
// helper.c:732-734 — aa64_ras: FEAT_RAS (ARMv8.2)
// helper.c:753-755 — aa64_doublefault: FEAT_DoubleFault
// ERRIDR_EL1 = 0 → 无错误记录
// DISR_EL1 支持延迟 SError 报告
// VDISR/VSESR 支持虚拟化 SError 注入
```

---

## 10. QEMU 实现特点与限制

### 10.1 完整实现

| 功能 | 状态 | 说明 |
|------|------|------|
| 地址匹配断点 (BT=0/1) | ✅ | 映射为 QEMU cpu_breakpoint |
| 数据观察点 (读/写/读写) | ✅ | 映射为 QEMU cpu_watchpoint |
| BAS 字节选择 | ✅ | 断点/观察点均支持 |
| MASK 地址掩码 | ✅ | 观察点支持最大 2GB 范围 |
| 链接断点 (linked BP) | ✅ | linked_bp_matches() |
| 安全状态过滤 (SSC) | ✅ | Secure/Non-secure 匹配 |
| EL 过滤 (PAC/HMC) | ✅ | EL0-EL3 逐级控制 |
| 三级调试陷阱 (TDA/TDOSA/TDCC) | ✅ | EL1→EL2→EL3 |
| 调试异常路由 (TDE/TGE) | ✅ | arm_debug_target_el() |

### 10.2 未实现 / 简化

| 功能 | 状态 | 说明 |
|------|------|------|
| 上下文 ID 匹配断点 (BT=2) | ❌ | LOG_UNIMP |
| VMID 匹配断点 (BT=8/10) | ❌ | LOG_UNIMP |
| 地址不匹配断点 (BT=4/5) | ❌ | LOG_UNIMP |
| 外部调试通信通道 | ❌ | RAZ/WI |
| RAS 错误记录 (ERX*) | ❌ | UNDEFINED |
| 外部调试接口 (EDSCR 等) | ❌ | 不实现 |

---

## 11. 调试异常流程总览

```
Guest 执行 → 指令匹配/内存访问
  │
  ├── 断点命中路径:
  │   cpu_breakpoint_test(pc) → arm_debug_check_breakpoint()
  │     → MDSCR.MDE=1? → generate_debug_exceptions?
  │     → bp_wp_matches(n, false) → SSC/PAC/HMC/linked 检查
  │     → arm_debug_excp_handler()
  │       → raise_exception_debug(EXCP_PREFETCH_ABORT, syn_breakpoint)
  │         → arm_debug_target_el() → EL1/EL2/EL3
  │
  ├── 观察点命中路径:
  │   cpu_watchpoint 触发 → arm_debug_check_watchpoint()
  │     → check_watchpoints() → bp_wp_matches(n, true)
  │     → arm_debug_excp_handler()
  │       → raise_exception_debug(EXCP_DATA_ABORT, syn_watchpoint)
  │
  └── 寄存器访问陷阱路径:
      MSR/MRS DBGBVRn → access_tda()
        → MDCR_EL2.TDA/TDE/HCR.TGE → CP_ACCESS_TRAP_EL2
        → MDCR_EL3.TDA → CP_ACCESS_TRAP_EL3
```

---

## 12. 小结

| 组件 | 实现 |
|------|------|
| **调试陷阱** | 三套访问检查（TDOSA/TDA/TDCC）× 三级路由（EL1→EL2→EL3），MDCR_EL2.TDE 整体重路由 |
| **MDSCR_EL1** | MDE 全局开关 + SS 单步使能 + TDCC EL0 通信陷阱，guard 所有 BP/WP 匹配 |
| **断点** | 动态注册 N 个 DBGBVRn/BCRn，写回调触发 hw_breakpoint_update → cpu_breakpoint_insert |
| **观察点** | 动态注册 N 个 DBGWVRn/WCRn，支持 BAS 字节选择和 MASK 掩码（最大 2GB 范围） |
| **匹配逻辑** | bp_wp_matches() 统一框架：SSC 安全过滤→PAC/HMC EL 过滤→WT 链接断点→命中 |
| **异常路由** | arm_debug_target_el()：TGE/TDE→EL2，Secure AArch32→EL3，默认→EL1 |
| **RAS** | 最小实现：ERRIDR=0（零记录），DISR/VDISR/VSESR 支持 SError 报告/注入 |
