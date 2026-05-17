# ARM64 异常级别（EL）状态管理综合导航

> 基于 QEMU 11.0.50 源码分析，整合 8 篇深度分析文档的核心内容。
> 本文为**导航式概览**：每个主题提取精华要点，并指向详细文档。

---

## 目录

1. [全景架构：EL 状态管理五大维度](#1-全景架构el-状态管理五大维度)
2. [文档地图：8 篇深度分析的定位与关系](#2-文档地图8-篇深度分析的定位与关系)
3. [CPU 状态表示与 EL 判定](#3-cpu-状态表示与-el-判定)
4. [异常进入：从 ELx 到 ELy 的完整路径](#4-异常进入从-elx-到-ely-的完整路径)
5. [异常返回：ERET 的完整路径](#5-异常返回eret-的完整路径)
6. [各 EL 下指令执行差异全景](#6-各-el-下指令执行差异全景)
7. [TCG 翻译器如何感知 EL 变化](#7-tcg-翻译器如何感知-el-变化)
8. [EL1：内核态执行环境](#8-el1内核态执行环境)
9. [EL2：Hypervisor 执行环境](#9-el2hypervisor-执行环境)
10. [EL3：Secure Monitor 执行环境](#10-el3secure-monitor-执行环境)
11. [安全状态转换与世界切换](#11-安全状态转换与世界切换)
12. [核心数据结构速查](#12-核心数据结构速查)
13. [关键函数速查表](#13-关键函数速查表)
14. [阅读路线推荐](#14-阅读路线推荐)

---

## 1. 全景架构：EL 状态管理五大维度

ARM64 的 EL0-EL3 切换不仅改变"特权级别"，而是触发五个维度的连锁变化：

```
                    EL 切换的五大影响维度
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║  ① 指令可用性与陷阱                                       ║
║     └── 特权指令（ERET/HVC/SMC）、FP/SVE/SME 陷阱、       ║
║         系统寄存器访问权限、Cache/TLB 操作权限              ║
║                                                          ║
║  ② MMU 域切换                                             ║
║     └── 翻译表基址（TTBR）、权限检查规则、                  ║
║         Stage-1/2 体制、PAN/UAO 控制                       ║
║                                                          ║
║  ③ PSTATE 状态                                            ║
║     └── DAIF 异常屏蔽、CurrentEL、SPSel、                  ║
║         PAN/TCO/SSBS/ALLINT 等特性位                       ║
║                                                          ║
║  ④ 寄存器组                                               ║
║     └── SP_ELx 切换、ELR_ELx/SPSR_ELx 保存恢复、          ║
║         SVE 向量长度（ZCR_ELx.LEN）                        ║
║                                                          ║
║  ⑤ TCG 翻译缓存                                           ║
║     └── hflags 重建 → TB 键变化 → TB 链断裂 →              ║
║         新 TB 使用新 EL 上下文翻译                          ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

**核心原则**：每次 EL 切换后，QEMU 调用 `arm_rebuild_hflags()` 重建翻译标志，新的 hflags 作为 TB 键的一部分，确保不同 EL 的代码永远不会复用同一个 Translation Block。

> **详见**：[41-EL切换TCG翻译变化](41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md) §1 架构概述

---

## 2. 文档地图：8 篇深度分析的定位与关系

```
                    文档关系图
                                            
  ┌─────────────────────────────────────────────────────────┐
  │            异常处理总流程                                  │
  │  ┌──────────────────────────────────────────────┐        │
  │  │  52: 异常入口/VBAR/PSTATE/ERET 完整流程        │        │
  │  │  35: EL 状态切换 + PSTATE 管理                 │        │
  │  └──────────┬────────────────────┬──────────────┘        │
  │             │                    │                        │
  │    ┌────────▼────────┐  ┌───────▼──────────┐            │
  │    │ 指令执行差异      │  │ TCG 翻译变化      │            │
  │    │ 07: Trap 机制全景 │  │ 41: hflags/TB 键  │            │
  │    │ 36: EL 感知翻译   │  │     链断裂/寄存器  │            │
  │    └────────┬─────────┘  └───────┬──────────┘            │
  │             │                    │                        │
  │    ┌────────▼────────────────────▼──────────┐            │
  │    │            特定 EL 深度分析               │            │
  │    │  39: EL3/Secure — SMC/ERET/安全切换      │            │
  │    │  40: EL1↔EL2 — HVC/VHE/Stage-2/NV      │            │
  │    │  37: 安全状态 — SCR/HCR/中断路由/四域模型 │            │
  │    └──────────────────────────────────────────┘            │
  └─────────────────────────────────────────────────────────┘
```

| # | 文档 | 大小 | 核心主题 | 适合解答的问题 |
|---|------|------|----------|----------------|
| **07** | [不同EL下指令执行差异](07-不同EL下指令执行差异深度分析.md) | 41KB | Trap 机制全景：WFI/WFE、SVC/HVC/SMC、FP/SVE/SME、ERET、系统寄存器、FGT | "某个指令在不同 EL 行为不同？" |
| **35** | [EL状态切换-异常进入返回](35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md) | 24KB | CPUARMState/PSTATE/异常进入/ERET/路由 | "异常进入时 PSTATE 怎么变？" |
| **36** | [不同EL下指令执行流变化](36-ARM64不同EL下指令执行流变化深度分析.md) | 19KB | hflags/DisasContext/MMU regime/PAN/SVE VL | "翻译器怎么知道当前是哪个 EL？" |
| **37** | [安全状态转换-SCR/HCR联动](37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md) | 24KB | 四域安全模型/SCR_EL3/HCR_EL2/中断路由 | "Secure 和 Non-secure 怎么切换？" |
| **39** | [EL3-Secure世界切换](39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md) | 27KB | SMC→EL3 完整路径/Monitor 执行/ERET 返回 | "SMC 怎么进 EL3？EL3 的执行环境？" |
| **40** | [EL1-EL2交互](40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md) | 25KB | HVC/VHE/Stage-2/NV 嵌套虚拟化 | "Hypervisor 怎么拦截 Guest？" |
| **41** | [EL切换TCG翻译变化](41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md) | 26KB | 59 个 TBFLAG 位域/TB 键/链断裂/寄存器组 | "EL 切换对 TCG 有什么影响？" |
| **52** | [异常处理完整流程](52-ARM64-异常处理完整流程深度分析-同步异步异常入口-VBAR向量表-PSTATE保存恢复-异常返回.md) | 34KB | 同步/异步异常全流程/EC 编码/路由表/ERET | "异常从触发到进入向量表的完整路径？" |

---

## 3. CPU 状态表示与 EL 判定

### 3.1 CPUARMState 中的 EL 相关字段

```c
// target/arm/cpu.h:270-318
struct CPUARMState {
    uint64_t pstate;          // CurrentEL[3:2]、SPSel、IL 等（非缓存部分）
    bool     aarch64;         // PSTATE.nRW 反转：true=AArch64

    // 条件标志单独缓存（避免位操作）
    uint32_t CF, VF, NF, ZF;
    uint64_t daif;            // DAIF 异常屏蔽位（独立缓存）
    uint32_t btype;           // BTI 分支类型

    // EL 专用寄存器组
    uint64_t elr_el[4];       // ELR_EL1/2/3（索引 1-3）
    uint64_t sp_el[4];        // SP_EL0/1/2/3
    uint64_t banked_spsr[8];  // SPSR 数组（索引对应不同异常模式）
};
```

### 3.2 EL 判定函数

```c
// target/arm/internals.h:489-515
static inline int arm_current_el(CPUARMState *env) {
    if (is_a64(env)) {
        return extract32(env->pstate, 2, 2);  // PSTATE.CurrentEL[3:2]
    }
    // AArch32: 通过 CPSR 模式位推断
}
```

**关键点**：PSTATE 被拆分存储——NZCV、DAIF、BTYPE 独立缓存以提高 TCG 生成代码效率，`pstate_read()`/`pstate_write()` 负责汇聚/拆分。

> **详见**：[35-EL状态切换](35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md) §1-2

---

## 4. 异常进入：从 ELx 到 ELy 的完整路径

### 4.1 十一步异常入口时序

```
 1. 确定目标 EL        arm_phys_excp_target_el() — 6 维查表
 2. SVE 长度切换       aarch64_sve_change_el(cur→new)
 3. VBAR 偏移计算      vbar_el[new] + 来源偏移(0/0x200/0x400/0x600) + 类型偏移(0/0x80/0x100/0x180)
 4. ESR 写入          esr_el[new] = syndrome（仅同步异常）
 5. 保存 SP           aarch64_save_sp(env, cur_el)
 6. 保存 ELR          elr_el[new] = PC
 7. 保存 SPSR         banked_spsr[idx] = pstate_read()
 8. 设置新 PSTATE      DAIF=1111, CurrentEL=new, SPSel=1(h), PAN/TCO/SSBS 按规则
 9. 恢复 SP           aarch64_restore_sp(env, new_el)
10. 重建 hflags       arm_rebuild_hflags(env) → 新 TB 键
11. 跳转              env->pc = VBAR + offset
```

### 4.2 SVC: EL0→EL1 实例

```
用户态 SVC #0
    │
    ▼ TCG: trans_SVC → EXCP_SWI(syndrome=syn_aa64_svc)
    │
    ▼ 运行时: arm_cpu_do_interrupt_aarch64(EXCP_SWI)
    ├── target_el = 1
    ├── addr = VBAR_EL1 + 0x400(从低 EL) + 0x000(Sync)
    ├── 保存: SPSR_EL1=PSTATE, ELR_EL1=PC, SP→sp_el[0]
    ├── 新 PSTATE: EL1h, DAIF=1111, PAN=SCTLR.SPAN?
    ├── SP = sp_el[1]
    └── PC = VBAR_EL1 + 0x400
```

### 4.3 VBAR 向量偏移表

| 异常来源 | 同步 | IRQ | FIQ | SError |
|----------|------|-----|-----|--------|
| 同 EL, SP_EL0 (t) | +0x000 | +0x080 | +0x100 | +0x180 |
| 同 EL, SP_ELx (h) | +0x200 | +0x280 | +0x300 | +0x380 |
| 低 EL, AArch64 | +0x400 | +0x480 | +0x500 | +0x580 |
| 低 EL, AArch32 | +0x600 | +0x680 | +0x700 | +0x780 |

### 4.4 异常目标 EL 路由

物理异常的目标 EL 由 `target_el_table[5][2][2][2][2][4]` 六维查表实现：

```
维度: [是否有 EL2][SCR.IRQ][SCR.FIQ][SCR.EA][HCR.IMO/FMO/AMO][异常类型]
     → 目标 EL (1/2/3)
```

同步异常的目标 EL 通常是执行异常指令的直接上级，但 HCR_EL2 的陷阱位可将 EL1 异常重定向到 EL2。

> **详见**：[52-异常处理完整流程](52-ARM64-异常处理完整流程深度分析-同步异步异常入口-VBAR向量表-PSTATE保存恢复-异常返回.md) §3-8，[35-EL状态切换](35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md) §3-6

---

## 5. 异常返回：ERET 的完整路径

### 5.1 十二步异常返回时序

```
 1. 保存当前 SP        aarch64_save_sp(env, cur_el)
 2. 清除独占监视器      arm_clear_exclusive(env)
 3. 读取 SPSR          banked_spsr[idx] → spsr
 4. 解析目标 EL        el_from_spsr(spsr) → new_el
 5. 合法性检查          EL/nRW/TGE/RME/GCS 多项验证
 6. 恢复 PSTATE        pstate_write(spsr)
 7. SS 处理            单步调试状态更新
 8. 恢复 SP            aarch64_restore_sp(env, new_el)
 9. 重建 hflags        helper_rebuild_hflags_a64(env, new_el)
10. TBI 地址处理       56 位截断或符号扩展
11. 恢复 PC            env->pc = ELR_ELx
12. SVE 长度切换       aarch64_sve_change_el(cur→new)
```

### 5.2 ERET: EL1→EL0 实例

```
内核 ERET
    │
    ▼ TCG: trans_ERET → gen_helper_exception_return + DISAS_EXIT
    │
    ▼ helper_exception_return():
    ├── SP → sp_el[1]
    ├── spsr = SPSR_EL1 → new_el=0
    ├── 验证: new_el(0) ≤ cur_el(1) ✓
    ├── pstate_write(spsr) → PSTATE 恢复(EL0t, DAIF 恢复)
    ├── SP = sp_el[0]
    ├── hflags 重建: mmu_idx = E10_0
    └── PC = ELR_EL1 (TBI 处理)
```

### 5.3 非法异常返回

以下情况触发 **Illegal Exception Return**（跳到目标地址但 PSTATE 设 IL=1，后续任何数据处理指令 UNDEF）：

- 目标 EL > 当前 EL
- 目标 EL=2 但 EL2 未实现
- 目标 EL=3 但从非 EL3 返回
- 返回到 AArch32 但当前 EL 不支持 AArch32

> **详见**：[52-异常处理完整流程](52-ARM64-异常处理完整流程深度分析-同步异步异常入口-VBAR向量表-PSTATE保存恢复-异常返回.md) §12-13，[35-EL状态切换](35-ARM64异常级别EL状态切换深度分析-异常进入返回与PSTATE管理.md) §7-8

---

## 6. 各 EL 下指令执行差异全景

### 6.1 指令差异的三个维度

```
1. 静态可用性  —— 指令在某 EL 不存在（UNDEF）
   例：ERET 在 EL0 → unallocated_encoding

2. 静态 Trap   —— 编译时已知的 Trap（嵌入 TB flags）
   例：FP 指令在 fp_excp_el != 0 时 → EXCP_UDEF

3. 动态 Trap   —— 运行时检查控制寄存器
   例：MSR 写 SCTLR_EL1 在 HCR_EL2.TVM=1 时 → CP_ACCESS_TRAP_EL2
```

### 6.2 分层陷阱控制模型

```
┌─────────────────────────────────────────┐
│  EL3 (SCR_EL3 / CPTR_EL3 / MDCR_EL3)  │  ← 控制所有 EL 的陷阱
├─────────────────────────────────────────┤
│  EL2 (HCR_EL2 / CPTR_EL2 / MDCR_EL2)  │  ← 控制 EL0/EL1 的陷阱
├─────────────────────────────────────────┤
│  EL1 (SCTLR_EL1 / CPACR_EL1)          │  ← 控制 EL0 的陷阱
├─────────────────────────────────────────┤
│  EL0 (用户态 — 被所有高 EL 控制)        │
└─────────────────────────────────────────┘

检查顺序: EL1 → EL2 → EL3（低优先级先检查，第一个匹配即生效）
```

### 6.3 关键指令在各 EL 的行为对照表

| 指令 | EL0 | EL1 | EL2 | EL3 |
|------|-----|-----|-----|-----|
| **SVC** | → EL1 | → EL1(自身) | → EL2(自身) | → EL3(自身) |
| **HVC** | UNDEF | → EL2 | → EL2(自身) | → EL2 |
| **SMC** | UNDEF | → EL3 (或 EL2 若 TSC) | → EL3 | → EL3(自身) |
| **ERET** | UNDEF | → 目标 EL | → 目标 EL | → 目标 EL |
| **WFI** | Trap(若 TWI) | Trap(若 TWI) | 执行 | 执行 |
| **WFE** | Trap(若 TWE) | Trap(若 TWE) | 执行 | 执行 |
| **MRS/MSR** | 受限子集 | EL1 寄存器 | +EL2 寄存器 | +EL3 寄存器 |
| **FP/SIMD** | CPACR→CPTR→CPTR | CPACR→CPTR | CPTR_EL2→CPTR_EL3 | CPTR_EL3 |
| **SVE** | 同 FP + ZCR 检查 | + ZCR_EL1 | + ZCR_EL2 | + ZCR_EL3 |
| **DC ZVA** | EL0 权限检查 | 执行 | 执行 | 执行 |
| **TLBI** | UNDEF | EL1 TLB | + EL2 TLB | + EL3 TLB |
| **AT** | UNDEF | S1 翻译 | +S12 翻译 | +S1E3 翻译 |

### 6.4 QEMU 的两层实现架构

```
┌──────────────────────────────────────────────────┐
│                  翻译时（编译时）                    │
│                                                    │
│  DisasContext 字段          TB flags 预计算         │
│  ├─ current_el             ← PSTATE              │
│  ├─ fp_excp_el             ← fp_exception_el()   │
│  ├─ sve_excp_el            ← sve_exception_el()  │
│  ├─ trap_eret              ← FEAT_FGT            │
│  └─ fgt_svc                ← FEAT_FGT            │
│                                                    │
│  检查方式: 直接判断字段值                             │
│  结果: 生成异常 或 跳过指令翻译                       │
├──────────────────────────────────────────────────┤
│                  运行时（Helper）                   │
│                                                    │
│  ├─ check_wfx_trap()       WFI/WFE 三层检查       │
│  ├─ pre_hvc()/pre_smc()    HVC/SMC disable 检查   │
│  ├─ access_check_cp_reg()  系统寄存器访问控制       │
│  │   ├─ cp_access_ok()     静态位矩阵              │
│  │   ├─ ri->accessfn()     动态检查回调            │
│  │   └─ FGT 位检查         细粒度 trap            │
│  └─ CPAccessResult → raise_exception()            │
└──────────────────────────────────────────────────┘
```

> **详见**：[07-指令执行差异](07-不同EL下指令执行差异深度分析.md) 全文（25 节），[36-指令执行流变化](36-ARM64不同EL下指令执行流变化深度分析.md) §2-7

---

## 7. TCG 翻译器如何感知 EL 变化

### 7.1 hflags：96 位 EL 敏感缓存

```c
// target/arm/cpu.h:185-188
typedef struct CPUARMTBFlags {
    uint32_t flags;    // 低 32 位（TBFLAG_ANY 共享字段）
    uint64_t flags2;   // 高 64 位（TBFLAG_A64 或 TBFLAG_A32 专用字段）
} CPUARMTBFlags;
```

**59 个位域**分为两组：

| 组别 | 位域数 | 关键字段 | 影响 |
|------|--------|----------|------|
| TBFLAG_ANY (flags) | 14 | MMUIDX(4b)、FPEXC_EL(2b)、ALIGN_MEM | 所有模式共享 |
| TBFLAG_A64 (flags2) | 45 | SVEEXC_EL、UNPRIV、ATA、NV/NV1/NV2、MTE、GCS、TRAP_ERET、BTI、PAUTH | AArch64 专用 |

### 7.2 hflags 重建流程

```
rebuild_hflags_a64(env, el, fp_el, mmu_idx)
    │
    ├── ① TBI (Top Byte Ignore)     — TCR_ELx 控制
    ├── ② SVE/SME 状态               — ZCR/SMCR + CPTR 联合
    ├── ③ SCTLR 位域                 — WXN/IESB/ATA/BT/EPAN
    ├── ④ PAuth (指针认证)           — APD/API 控制
    ├── ⑤ BTI (分支目标识别)         — SCTLR.BT0/BT1
    ├── ⑥ UNPRIV (LDTR/STTR)        — 仅 EL1+
    ├── ⑦ FGT (Fine-Grained Trap)   — FEAT_FGT
    ├── ⑧ NV (嵌套虚拟化)           — HCR_NV/NV1/NV2
    ├── ⑨ MTE (内存标记)            — SCTLR/HCR 联合
    └── ⑩ GCS (Guarded Call Stack)  — GCSCR/GCSCRE0
```

### 7.3 TB 键与链断裂

```c
// arm_get_tb_cpu_state() — 生成 TB 查找键
tb_key = (PC, hflags.flags, hflags.flags2)
```

EL 切换 → hflags 变化 → TB 键不同 → **TB 缓存 miss** → 翻译新 TB

```c
// gen_goto_tb() — TB 链接决策
if (use_goto_tb(s, dest)) {
    tcg_gen_goto_tb(n);      // 直接跳转（同 EL 内优化）
} else {
    gen_a64_update_pc(s, dest - s->pc_curr);
    tcg_gen_exit_tb(NULL, 0); // 退出到主循环（EL 切换时）
}
```

**DISAS_EXIT / DISAS_NORETURN**：所有改变 EL 的指令（SVC/HVC/SMC/ERET）都使用这些退出码，强制退出当前 TB，回到执行循环重新查找。

### 7.4 EL0 vs EL1 的 hflags 差异

| hflag 字段 | EL0 (E10_0) | EL1 (E10_1) | 原因 |
|-----------|-------------|-------------|------|
| MMUIDX | E10_0 | E10_1 / E10_1_PAN | MMU regime 不同 |
| UNPRIV | 0 | 1 (有 LDTR) | EL1 的 LDTR 使用 EL0 权限 |
| FPEXC_EL | 可能 ≠0 | 通常 0 | EL0 FP 可被 EL1 陷阱 |
| BT | SCTLR.BT0 | SCTLR.BT1 | BTI 不同控制位 |
| ATA/ATA0 | — | ATA 不同 | MTE 标签检查差异 |
| NV/NV1/NV2 | 0 | 可能 1 | 仅 EL1 受嵌套虚拟化影响 |
| TRAP_ERET | 0 | 可能 1 | 仅 EL1 的 ERET 可被 FGT 陷阱 |

> **详见**：[41-EL切换TCG翻译变化](41-ARM64-EL切换TCG翻译变化深度分析-hflags位域全景-TB键与链断裂-寄存器组切换与行为效应.md) §2-8，[36-指令执行流变化](36-ARM64不同EL下指令执行流变化深度分析.md) §1

---

## 8. EL1：内核态执行环境

EL1 是操作系统内核运行的异常级别。进入 EL1 后的关键状态变化：

### 8.1 进入 EL1 时的状态变化

| 变化项 | 旧值(EL0) | 新值(EL1) |
|--------|-----------|-----------|
| CurrentEL | 0 | 1 |
| SPSel | 0 (SP_EL0) | 1 (SP_EL1) |
| DAIF | 用户值 | 全屏蔽 (1111) |
| PAN | 无效 | SCTLR_EL1.SPAN=0 → PAN=1 |
| MMU regime | E10_0 (用户态) | E10_1 (内核态，可能 +PAN) |
| 可访问寄存器 | 用户子集 | + SCTLR_EL1/TTBR0_EL1/VBAR_EL1 等 |
| SP | SP_EL0 (用户栈) | SP_EL1 (内核栈) |
| 陷阱来源 | 受 EL1 控制 | 受 EL2/EL3 控制 |

### 8.2 EL1 特有的指令能力

- **ERET**：返回 EL0
- **系统寄存器**：读写 `SCTLR_EL1`、`TTBR0/1_EL1`、`TCR_EL1`、`VBAR_EL1`、`ESR_EL1`、`FAR_EL1` 等
- **LDTR/STTR**：以 EL0 权限访问内存（UNPRIV 操作，用于 `copy_from_user`/`copy_to_user`）
- **DC/IC/TLBI**：Cache/TLB 维护操作（受 EL2 HACR/HCR 陷阱控制）
- **AT**：地址翻译指令（S1E0R/S1E0W/S1E1R/S1E1W）

### 8.3 EL1 受到的 EL2 陷阱控制

当 EL2 启用时，HCR_EL2 的位可陷阱 EL1 的大量操作：

| HCR 位 | 效果 |
|--------|------|
| TVM | EL1 写 SCTLR/TTBR/TCR/MAIR 等 → Trap EL2 |
| TRVM | EL1 读 上述寄存器 → Trap EL2 |
| TSC | EL1 执行 SMC → Trap EL2 |
| TWI/TWE | EL1 执行 WFI/WFE → Trap EL2 |
| IMO/FMO/AMO | EL1 看到的 IRQ/FIQ/SError 实际来自虚拟中断 |
| TGE | 所有 EL0/EL1 异常直接到 EL2 |

> **详见**：[40-EL1-EL2交互](40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md) §3-7

---

## 9. EL2：Hypervisor 执行环境

### 9.1 进入 EL2 的路径

1. **HVC**：EL1 主动调用 Hypercall → EL2
2. **陷阱**：EL1 执行被 HCR_EL2 陷阱的操作 → EL2（如 WFI、MSR、SMC）
3. **物理异常路由**：IRQ/FIQ/SError 路由到 EL2（HCR.IMO/FMO/AMO=1）

### 9.2 EL2 的特殊执行特性

| 特性 | 说明 |
|------|------|
| **VHE (E2H=1)** | EL2 复用 EL1 寄存器名（SCTLR_EL2→SCTLR_EL1 别名），Host OS 直接运行在 EL2 |
| **Stage-2 MMU** | EL2 控制 VTTBR_EL2 管理 Guest 的 IPA→PA 翻译 |
| **独立 MMU regime** | 非 VHE 时使用 E2 索引（TTBR0_EL2 + VTCR_EL2） |
| **嵌套虚拟化 (NV)** | HCR_EL2.NV/NV1/NV2 让 Guest Hypervisor 在 EL1 运行时的 EL2 操作被 EL2 陷阱 |

### 9.3 VHE 寄存器重定向

当 HCR_EL2.E2H=1 且 TGE=1 时：

```
"EL1" 寄存器名         实际访问
SCTLR_EL1         →   SCTLR_EL2
TTBR0_EL1         →   TTBR0_EL2
TCR_EL1           →   TCR_EL2
MAIR_EL1          →   MAIR_EL2
VBAR_EL1          →   VBAR_EL2
CNTP_*_EL0        →   CNTHP_*_EL2
```

这使得 Linux Host 无需修改即可运行在 EL2（KVM 的 VHE 模式）。

> **详见**：[40-EL1-EL2交互](40-ARM64-EL1-EL2交互深度分析-HVC陷入-VHE重定向-Stage2控制与嵌套虚拟化.md) §1, §4-7

---

## 10. EL3：Secure Monitor 执行环境

### 10.1 进入 EL3 的路径

```
EL1 执行 SMC #imm
    │
    ├── trans_SMC() 翻译检查
    │   └── EL0 → UNDEF; EL1+ → gen_helper_pre_smc()
    │
    ├── pre_smc 陷阱决策
    │   ├── 无 EL3 + 非 PSCI → UNDEF
    │   ├── HCR.TSC + NS EL1 → 陷入 EL2（不到 EL3）
    │   ├── SCR.SMD + 非 PSCI → UNDEF
    │   └── 通过 → EXCP_SMC
    │
    └── arm_cpu_do_interrupt_aarch64(EXCP_SMC, target_el=3)
        ├── VBAR_EL3 + 来源偏移 + 类型偏移
        ├── PSTATE: DAIF=1111, EL3h, AArch64
        ├── SP = SP_EL3
        ├── MMU regime: E3（最高特权，Security State 固定）
        └── hflags 重建: mmu_idx=E3
```

### 10.2 EL3 的执行环境特点

| 特性 | 说明 |
|------|------|
| **安全状态不可变** | EL3 始终在 Secure 域执行（与 SCR.NS 无关） |
| **SCR_EL3 控制权** | 写 SCR_EL3.NS 切换低 EL 的世界（触发 TLB 刷新） |
| **独立 MMU regime** | E3 索引，使用 MAIR_EL3/TCR_EL3/TTBR0_EL3 |
| **无 Stage-2** | EL3 不受 Stage-2 翻译 |
| **最高陷阱级别** | CPTR_EL3/MDCR_EL3 可陷阱所有 EL 的操作 |
| **RME 扩展** | SCR_EL3.NSE 与 NS 组合确定四域: Root/Secure/Non-secure/Realm |

### 10.3 EL3 ERET 返回时的安全状态切换

```
EL3 ERET
    │
    ├── 读 SPSR_EL3 → 目标 EL & PSTATE
    ├── SCR_EL3.NS 决定目标世界
    │   ├── NS=0 → Secure (EL1_S)
    │   ├── NS=1 → Non-secure (EL1_NS)
    │   └── NS=1,NSE=1 → Realm (FEAT_RME)
    ├── pstate_write(spsr) → 恢复
    ├── hflags 重建 → arm_mmu_idx_el(new_el) 基于 SCR.NS 确定安全状态
    └── PC = ELR_EL3
```

> **详见**：[39-EL3-Secure世界切换](39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md) §1-15

---

## 11. 安全状态转换与世界切换

### 11.1 四域安全模型

```c
// target/arm/internals.h
typedef enum ARMSecuritySpace {
    ARMSS_Secure      = 0,  // Secure 世界
    ARMSS_NonSecure   = 1,  // Non-secure 世界
    ARMSS_Root        = 2,  // Root（EL3 专用，FEAT_RME）
    ARMSS_Realm       = 3,  // Realm（FEAT_RME）
} ARMSecuritySpace;
```

判定公式：
```
安全域 = f(arm_current_el(), SCR_EL3.NS, SCR_EL3.NSE)
  EL3:           → Root (FEAT_RME) 或 Secure
  NS=0, NSE=0:  → Secure
  NS=1, NSE=0:  → Non-secure
  NS=1, NSE=1:  → Realm
```

### 11.2 SCR_EL3 关键控制位

| 位 | 名称 | 功能 |
|----|------|------|
| 0 | NS | 低 EL 安全/非安全选择 |
| 1-3 | IRQ/FIQ/EA | 异常路由到 EL3 |
| 7 | SMD | 禁用 SMC |
| 10 | RW | 低 EL AArch64/AArch32 |
| 18 | EEL2 | 使能 Secure EL2 |
| 62 | NSE | RME 扩展位 |

### 11.3 世界切换的 TLB 刷新

`scr_write()` 检测 NS 位变化时，刷新全部 12 种 MMU 索引的 TLB：

```
E10_0, E10_1, E10_1_PAN,          // Stage-1 EL0/EL1
E20_0, E20_2, E20_2_PAN,          // Stage-1 EL0/EL2 (VHE)
E2,                                // Stage-1 EL2 (非 VHE)
Stage2, Stage2_S,                  // Stage-2
E3,                                // EL3
Phys_NS, Phys_S                   // 物理地址
```

> **详见**：[37-安全状态转换](37-ARM64安全状态转换深度分析-SCR_EL3-HCR_EL2联动-中断路由与异常级别安全域.md) §1-4, [39-EL3-Secure世界切换](39-ARM64-EL3-Secure世界切换深度分析-SMC异常入口-Monitor执行-ERET返回与安全状态转换.md) §8, §11

---

## 12. 核心数据结构速查

### CPUARMState 中 EL 相关字段

| 字段 | 类型 | 用途 |
|------|------|------|
| `pstate` | uint64_t | CurrentEL, SPSel, IL, SS, PAN, UAO 等 |
| `aarch64` | bool | 执行状态 (true=AArch64) |
| `daif` | uint64_t | DAIF 异常屏蔽位 (缓存) |
| `elr_el[4]` | uint64_t[] | 异常链接寄存器 |
| `sp_el[4]` | uint64_t[] | 栈指针组 |
| `banked_spsr[8]` | uint64_t[] | 保存的程序状态 |
| `cp15.scr_el3` | uint64_t | SCR_EL3 安全配置 |
| `cp15.hcr_el2` | uint64_t | HCR_EL2 虚拟化配置 |
| `cp15.sctlr_el[4]` | uint64_t[] | 系统控制寄存器 |
| `cp15.vbar_el[4]` | uint64_t[] | 向量基址寄存器 |

### PSTATE 模式编码

| 编码 | 模式 | 含义 |
|------|------|------|
| 0x0 | EL0t | EL0, SP_EL0 |
| 0x4 | EL1t | EL1, SP_EL0 |
| 0x5 | EL1h | EL1, SP_EL1 |
| 0x8 | EL2t | EL2, SP_EL0 |
| 0x9 | EL2h | EL2, SP_EL2 |
| 0xC | EL3t | EL3, SP_EL0 |
| 0xD | EL3h | EL3, SP_EL3 |

---

## 13. 关键函数速查表

### 异常入口/返回

| 函数 | 文件:行 | 用途 |
|------|---------|------|
| `arm_cpu_do_interrupt` | helper.c:9469-9530 | 异常入口总分发 |
| `arm_cpu_do_interrupt_aarch64` | helper.c:9198-9428 | AArch64 异常入口 |
| `arm_phys_excp_target_el` | helper.c:8369-8421 | 物理异常目标 EL |
| `target_el_table` | helper.c:8309-8364 | 6 维路由查表 |
| `HELPER(exception_return)` | helper-a64.c:622-785 | ERET 运行时 |
| `el_from_spsr` | helper-a64.c:584-620 | SPSR → 目标 EL |
| `pstate_read/write` | cpu.h:1607-1626 | PSTATE 汇聚/拆分 |
| `aarch64_save_sp/restore_sp` | internals.h:565-581 | SP 组切换 |

### hflags / TB

| 函数 | 文件:行 | 用途 |
|------|---------|------|
| `rebuild_hflags_a64` | hflags.c:240-504 | AArch64 hflags 构建 |
| `arm_rebuild_hflags` | hflags.c:506-573 | hflags 总入口 |
| `arm_get_tb_cpu_state` | hflags.c:620-695 | TB 键生成 |
| `gen_goto_tb` | translate-a64.c:535-564 | TB 链接决策 |

### 指令陷阱

| 函数 | 文件:行 | 用途 |
|------|---------|------|
| `fp_exception_el` | helper.c | FP 陷阱目标 EL |
| `sve_exception_el` | helper.c | SVE 陷阱目标 EL |
| `check_wfx_trap` | op_helper.c | WFI/WFE 三层陷阱 |
| `pre_hvc/pre_smc` | op_helper.c | HVC/SMC 陷阱决策 |
| `access_check_cp_reg` | translate-a64.c | 系统寄存器访问检查 |
| `arm_hcr_el2_eff` | helper.c | HCR_EL2 有效值 |

### 安全状态

| 函数 | 文件:行 | 用途 |
|------|---------|------|
| `arm_security_space` | internals.h | 当前安全域 |
| `arm_security_space_below_el3` | internals.h | EL3 以下安全域 |
| `scr_write` | helper.c | SCR_EL3 写入+TLB 刷新 |

---

## 14. 阅读路线推荐

### 路线 A：异常处理机制（入门）

```
35 (EL状态切换/PSTATE)
  → 52 (异常入口/VBAR/返回完整流程)
    → 07 §1-4 (SVC/HVC/SMC 指令)
```

### 路线 B：EL 指令差异（进阶）

```
07 (指令Trap全景，25节)
  → 36 (hflags/DisasContext/MMU/PAN/SVE)
    → 41 §2-5 (59个TBFLAG位域详解)
```

### 路线 C：虚拟化（EL1↔EL2）

```
40 (HVC/VHE/Stage-2/NV)
  → 36 §4-5 (HCR陷阱/MMU regime)
    → 41 §11-12 (SCTLR/HCR对翻译影响)
```

### 路线 D：安全世界切换（EL3/TrustZone）

```
37 (四域模型/SCR/HCR联动)
  → 39 (SMC→EL3完整路径/Monitor/ERET)
    → 52 §3 (目标EL路由表)
```

### 路线 E：TCG 内部实现

```
41 (hflags全景/TB键/链断裂/寄存器组)
  → 36 §1-2 (hflags重建/DisasContext)
    → 07 §23 (TB flags预计算)
```

---

## 源文件索引

| 文件 | 核心内容 |
|------|----------|
| `target/arm/cpu.h` | CPUARMState 结构、PSTATE 定义、TBFLAG 位域 |
| `target/arm/internals.h` | arm_current_el()、save/restore_sp、安全状态判定 |
| `target/arm/helper.c` | 异常入口、target_el_table、系统寄存器访问函数 |
| `target/arm/helper-a64.c` | ERET helper、el_from_spsr |
| `target/arm/tcg/hflags.c` | rebuild_hflags_a64、arm_get_tb_cpu_state |
| `target/arm/tcg/translate-a64.c` | 指令翻译、DisasContext、gen_goto_tb |
| `target/arm/tcg/op_helper.c` | WFI/WFE/HVC/SMC helper |
| `target/arm/cpregs.h` | ARMCPRegInfo 权限标志 |
| `target/arm/syndrome.h` | EC 编码、syndrome 构造函数 |
| `target/arm/cpu-irq.c` | 中断路由、arm_excp_unmasked |

---

## 交叉引用

- [47-系统寄存器与CP访问](47-ARM64-系统寄存器与CP访问深度分析-ARMCPRegInfo框架-MRS-MSR翻译-cpregs哈希表-EL银行与访问控制.md) — 系统寄存器框架
- [44-TCG执行循环](44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md) — cpu_exec 主循环
- [43-TCG-TLB](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — TLB 与内存访问
- [48-安全扩展TrustZone](48-ARM64-安全扩展TrustZone深度分析-SCR_EL3-Secure-NS世界切换-安全状态隔离.md) — TrustZone 完整分析

---

> 文档生成时间基于 QEMU 11.0.50 源码，综合 8 篇深度分析文档精华。
> 共引用文档总计 212KB，本导航浓缩为核心要点与阅读路线。
