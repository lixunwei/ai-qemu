# ARM64 系统寄存器与 CP 访问深度分析：ARMCPRegInfo 框架、MRS/MSR 翻译、cpregs 哈希表、EL 银行与访问控制

> 基于 QEMU 11.0.50 源码分析，涵盖 ARM 系统寄存器（协处理器）完整子系统：
> ARMCPRegInfo 结构体（name/cp/crn/crm/opc0-2 编码、CPState AA32/AA64/BOTH、
> ARM_CP_* 22 个标志位、access 权限 PL0-PL3_RW、fieldoffset/bank_fieldoffsets 直接偏移+银行偏移、
> accessfn/readfn/writefn/resetfn 四回调、secure/fgt/nv2_redirect_offset 安全/陷入/NV2 字段）、
> cpregs 哈希表（GHashTable 键为 ENCODE_AA64_CP_REG 32 位编码、
> add_cpreg_to_hashtable 银行解析+AA32 端序调整+别名标记、get_arm_cp_reginfo O(1) 查找）、
> MRS/MSR AArch64 翻译（handle_sys 编码→查表→cp_access_ok→accessfn/fgt→NV2/VHE 重定向→
> gen_helper_get/set_cp_reg64 运行时回调、trans_SYS/MRS 入口）、
> MRC/MCR AArch32 翻译（do_coproc_insn→ENCODE_CP_REG 查表→同样访问检查链）、
> CP15 状态存储（CPUARMState.cp15 union 银行结构 sctlr_el[4]/ttbr0_el[4]/tcr_el[4]/mair_el[4]/vbar_el[4]、
> Secure/NS 双银行 sctlr_s/ns/ttbr0_s/ns）、
> 访问控制（cp_access_ok 编译时 EL 权限检查、HELPER(access_check_cp_reg) 运行时 accessfn+HSTR_EL2+FGT 三级检查、
> CPAccessResult TRAP_EL1/2/3 陷入路由）、
> EL 切换与寄存器上下文（arm_cpu_do_interrupt_aarch64 异常入口+SVE 状态调整、
> exception_return 异常返回+rebuild_hflags、SCR_EL3 安全状态切换→TLB 刷新）、
> FGT 精细陷入（FGTBit 编码 idx+bitpos+rev+nxs、fgt_read[]/fgt_write[]/fgt_exec[] 位图）、
> NV2 嵌套虚拟化（nv2_redirect_offset→VNCR_EL2 内存重定向、NV2_REDIR_NV1 条件选择）、
> VHE 重定向（vhe_redir_to_el2/el01、E2H+EL2→FOO_EL2 寄存器映射）。

---

## 目录

1. [架构概述](#1-架构概述)
2. [ARMCPRegInfo 结构体](#2-armcpreginfo-结构体)
3. [ARM_CP_* 类型标志](#3-arm_cp-类型标志)
4. [CPState / CPSecureState / CPAccessRights](#4-cpstate--cpsecurestate--cpaccessrights)
5. [cpregs 哈希表](#5-cpregs-哈希表)
6. [MRS/MSR AArch64 翻译](#6-mrsmsr-aarch64-翻译)
7. [MRC/MCR AArch32 翻译](#7-mrcmcr-aarch32-翻译)
8. [CP15 状态存储与银行结构](#8-cp15-状态存储与银行结构)
9. [访问控制机制](#9-访问控制机制)
10. [EL 切换与寄存器上下文](#10-el-切换与寄存器上下文)
11. [FGT 精细陷入](#11-fgt-精细陷入)
12. [NV2 嵌套虚拟化重定向](#12-nv2-嵌套虚拟化重定向)
13. [VHE 寄存器重定向](#13-vhe-寄存器重定向)
14. [系统寄存器定义示例](#14-系统寄存器定义示例)
15. [完整 MRS 执行流程总结](#15-完整-mrs-执行流程总结)

---

## 1. 架构概述

QEMU 使用统一的 **ARMCPRegInfo** 框架管理所有 ARM 系统寄存器（AArch64 MRS/MSR）和协处理器寄存器（AArch32 MRC/MCR）：

```
定义阶段（启动时）:
  ARMCPRegInfo 数组 → define_arm_cp_regs()
    → add_cpreg_to_hashtable() → GHashTable cp_regs

翻译阶段（TB 编译时）:
  MRS X0, SCTLR_EL1
    → ENCODE_AA64_CP_REG(3,0,1,0,0) = key
    → get_arm_cp_reginfo(cp_regs, key) → ri
    → cp_access_ok(current_el, ri, isread) → 编译时权限
    → gen_helper_access_check_cp_reg()    → 运行时检查
    → gen_helper_get_cp_reg64(ri)         → 读取

执行阶段（TB 运行时）:
  helper_access_check_cp_reg → accessfn + HSTR_EL2 + FGT
  helper_get_cp_reg64 → ri->readfn 或 fieldoffset 直接读取
```

### 关键源文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `target/arm/cpregs.h` | 32-152 | ARM_CP_* 标志定义 |
| `target/arm/cpregs.h` | 266-375 | CPState/CPSecureState/CPAccessRights/CPAccessResult |
| `target/arm/cpregs.h` | 921-1049 | ARMCPRegInfo 结构体完整定义 |
| `target/arm/cpregs.h` | 1122-1126 | cp_access_ok 内联函数 |
| `target/arm/cpu.h` | 320-540 | CPUARMState.cp15 银行存储 |
| `target/arm/cpu.h` | 927 | GHashTable *cp_regs |
| `target/arm/helper.c` | 7523-7613 | add_cpreg_to_hashtable 核心 |
| `target/arm/helper.c` | 7989-8052 | define_arm_cp_regs/get_arm_cp_reginfo |
| `target/arm/tcg/translate-a64.c` | 2764-2960 | handle_sys MRS/MSR 翻译 |
| `target/arm/tcg/op_helper.c` | 802-900 | HELPER(access_check_cp_reg) |

---

## 2. ARMCPRegInfo 结构体

```c
// target/arm/cpregs.h:921-1049
struct ARMCPRegInfo {
    const char *name;             // 923: 寄存器名（调试用）

    // === 编码字段 ===
    uint8_t cp;                   // 942: 协处理器号（AArch64 用 0x13）
    uint8_t crn;                  // 943: CRn 字段
    uint8_t crm;                  // 944: CRm 字段
    uint8_t opc0;                 // 945: AArch64 op0
    uint8_t opc1;                 // 946: op1
    uint8_t opc2;                 // 947: op2

    // === 状态与类型 ===
    CPState state;                // 949: AA32/AA64/BOTH
    int type;                     // 951: ARM_CP_* 标志组合
    CPAccessRights access;        // 953: PL0-PL3_RW 权限
    CPSecureState secure;         // 955: Secure/NS/Both

    // === 陷入与重定向 ===
    FGTBit fgt;                   // 960: 精细陷入位编码
    uint32_t nv2_redirect_offset; // 966: NV2 内存重定向偏移
    uint32_t vhe_redir_to_el2;    // 972: VHE→EL2 重定向键
    uint32_t vhe_redir_to_el01;   // 980: EL02/EL12→EL0/EL1 回指

    // === 值与存储 ===
    uint64_t resetvalue;          // 986: 复位值（或 ARM_CP_CONST 值）
    ptrdiff_t fieldoffset;        // 993: offsetof(CPUARMState, field)
    ptrdiff_t bank_fieldoffsets[2]; // 1007: [0]=secure, [1]=non-secure

    // === 回调函数 ===
    CPAccessFn *accessfn;         // 1015: 运行时访问检查
    CPReadFn *readfn;             // 1021: 自定义读（NULL→fieldoffset）
    CPWriteFn *writefn;           // 1027: 自定义写（NULL→fieldoffset）
    CPReadFn *raw_readfn;         // 1034: 原始读（迁移/KVM）
    CPWriteFn *raw_writefn;       // 1042: 原始写
    CPResetFn *resetfn;           // 1048: 自定义复位
};
```

### 访问模式

- **fieldoffset 直接访问**：`readfn/writefn == NULL` 时，QEMU 直接从 `env + fieldoffset` 读写
- **回调访问**：提供 `readfn/writefn` 时，通过 helper 调用回调（如定时器、TLB 维护等副作用寄存器）
- **常量**：`ARM_CP_CONST` 时，读返回 `resetvalue`，写忽略

---

## 3. ARM_CP_* 类型标志

```c
// target/arm/cpregs.h:32-152
enum {
    // === 特殊寄存器类型（低 4 位互斥） ===
    ARM_CP_NOP       = 0x0001,  // 39: 读写忽略
    ARM_CP_WFI       = 0x0002,  // 41: WFI 指令
    ARM_CP_NZCV      = 0x0003,  // 43: NZCV 特殊处理
    ARM_CP_CURRENTEL = 0x0004,  // 45: CurrentEL
    ARM_CP_DC_ZVA    = 0x0005,  // 47: DC ZVA 缓存零化
    ARM_CP_GCSPUSHM  = 0x0008,  // 51: GCS 栈操作指令...

    // === 通用标志（位标志，可组合） ===
    ARM_CP_CONST            = 1<<4,   // 60: 常量寄存器
    ARM_CP_64BIT            = 1<<5,   // 62: AArch32 64 位
    ARM_CP_SUPPRESS_TB_END  = 1<<6,   // 67: 写后不终止 TB
    ARM_CP_OVERRIDE         = 1<<7,   // 73: 允许覆盖已有定义
    ARM_CP_ALIAS            = 1<<8,   // 80: 别名（不迁移）
    ARM_CP_IO               = 1<<9,   // 86: I/O 操作（终止 TB）
    ARM_CP_NO_RAW           = 1<<10,  // 93: 不支持原始访问
    ARM_CP_RAISES_EXC       = 1<<11,  // 99: 回调可能抛异常
    ARM_CP_NEWEL             = 1<<12, // 105: 写可能切换 EL
    ARM_CP_FPU               = 1<<13, // 110: 需 FPU 访问检查
    ARM_CP_SVE               = 1<<14, // 115: 需 SVE 访问检查
    ARM_CP_NO_GDB            = 1<<15, // 117: GDB 不可见
    ARM_CP_EL3_NO_EL2_UNDEF  = 1<<16, // 126: 无 EL2 时 UNDEF
    ARM_CP_EL3_NO_EL2_KEEP   = 1<<17, // 127: 无 EL2 时保留
    ARM_CP_EL3_NO_EL2_C_NZ   = 1<<18, // 128: 无 EL2 时常量非零
    ARM_CP_SME               = 1<<19, // 133: 需 SME 访问检查
    ARM_CP_NV2_REDIRECT      = 1<<20, // 138: NV2 重定向到 EL1
    ARM_CP_ADD_TLBI_NXS      = 1<<21, // 146: 自动注册 NXS 变体
    ARM_CP_NV_NO_TRAP        = 1<<22, // 151: NV 不陷入
};
```

---

## 4. CPState / CPSecureState / CPAccessRights

### CPState

```c
// cpregs.h:266-270
ARM_CP_STATE_AA32 = 0   // 仅 AArch32 可见
ARM_CP_STATE_AA64 = 1   // 仅 AArch64 可见
ARM_CP_STATE_BOTH = 2   // 两种状态均可见（自动生成两个哈希条目）
```

### CPSecureState

```c
// cpregs.h:283-287
ARM_CP_SECSTATE_BOTH = 0  // Secure 和 NS 各一个条目
ARM_CP_SECSTATE_S    = 1  // 仅 Secure
ARM_CP_SECSTATE_NS   = 2  // 仅 Non-Secure
```

### CPAccessRights — 位编码权限

```c
// cpregs.h:307-333
PL3_R = 0x80, PL3_W = 0x40   // EL3 读/写
PL2_R = 0x20|PL3_R, PL2_W = 0x10|PL3_W   // EL2（含 EL3）
PL1_R = 0x08|PL2_R, PL1_W = 0x04|PL2_W   // EL1（含 EL2/3）
PL0_R = 0x02|PL1_R, PL0_W = 0x01|PL1_W   // EL0（含 EL1/2/3）
```

权限检查：`(ri->access >> ((current_el * 2) + isread)) & 1`

### CPAccessResult

```c
// cpregs.h:335-375
CP_ACCESS_OK          = 0       // 允许
CP_ACCESS_TRAP_EL1    = 4|1     // 陷入 EL1
CP_ACCESS_TRAP_EL2    = 4|2     // 陷入 EL2
CP_ACCESS_TRAP_EL3    = 4|3     // 陷入 EL3
CP_ACCESS_UNDEFINED   = 8       // UNDEF（syndrome 0x0）
CP_ACCESS_EXLOCK      = 12      // GCS EXLOCK 异常
```

---

## 5. cpregs 哈希表

### 存储

```c
// target/arm/cpu.h:927
GHashTable *cp_regs;  // 每个 ARMCPU 实例一个哈希表
// 键: uint32_t 编码（ENCODE_AA64_CP_REG 或 ENCODE_CP_REG）
// 值: ARMCPRegInfo* 指针
```

### 注册流程

```c
// target/arm/helper.c:7989-7993
void define_arm_cp_regs_len(ARMCPU *cpu, const ARMCPRegInfo *regs, size_t len)
{
    for (size_t i = 0; i < len; ++i)
        define_one_arm_cp_reg(cpu, regs + i);
}
// → define_one_arm_cp_reg → add_cpreg_to_hashtable_aa32/aa64
```

### add_cpreg_to_hashtable 核心

```c
// target/arm/helper.c:7523-7613
static void add_cpreg_to_hashtable(ARMCPU *cpu, ARMCPRegInfo *r,
                                   CPState state, CPSecureState secstate,
                                   uint32_t key)
{
    // 1. 覆盖检查（ARM_CP_OVERRIDE）           7531-7535
    // 2. 银行字段解析:
    //    bank_fieldoffsets[0/1] → fieldoffset   7539-7548
    //    AA32 银行别名标记                       7549-7573
    // 3. AA32+BOTH 端序调整:
    //    大端 → fieldoffset += 4                7581-7586
    // 4. 特殊寄存器标记 NO_RAW                  7592-7594
    // 5. 更新 state/secure 为具体实例           7600-7601
    // 6. 插入哈希表                             7612
    g_hash_table_insert(cpu->cp_regs, key, r);
}
```

### 查找

```c
// target/arm/helper.c:8037-8039
const ARMCPRegInfo *get_arm_cp_reginfo(GHashTable *cpregs, uint32_t encoded_cp)
{
    return g_hash_table_lookup(cpregs, (gpointer)(uintptr_t)encoded_cp);
}
// O(1) 哈希查找
```

---

## 6. MRS/MSR AArch64 翻译

### handle_sys 主函数

```c
// target/arm/tcg/translate-a64.c:2764-2960
static void handle_sys(DisasContext *s, bool isread,
                       unsigned int op0, op1, op2, crn, crm, rt)
{
    // 1. 编码→哈希键                                    2768
    uint32_t key = ENCODE_AA64_CP_REG(op0, op1, crn, crm, op2);

    // 2. 查找寄存器                                     2769
    const ARMCPRegInfo *ri = get_arm_cp_reginfo(s->cp_regs, key);
    if (!ri) { gen_sysreg_undef(); return; }             // 2796-2804

    // 3. NV2 内存重定向检查                              2807-2820
    // 4. 编译时权限检查 cp_access_ok()                   2822-2860
    // 5. VHE 重定向 (E2H+EL2→FOO_EL2)                  2862-2881
    // 6. 运行时访问检查 (accessfn/fgt)                   2883-2892
    if (ri->accessfn || ri->fgt)
        gen_helper_access_check_cp_reg(key, syndrome, isread);
    // 7. FPU/SVE/SME 访问检查                           2901-2908
    // 8. NV trap/redirect 处理                          2911-2932
    // 9. NV2 内存访问                                   2934-...
    // 10. 实际读/写:
    //     ARM_CP_CONST → 直接加载常量
    //     readfn/writefn → gen_helper_get/set_cp_reg64
    //     fieldoffset → tcg_gen_ld/st_i64
}

// 3149-3152: 入口
static bool trans_SYS(DisasContext *s, arg_SYS *a) {
    handle_sys(s, a->l, a->op0, a->op1, a->op2, a->crn, a->crm, a->rt);
}
```

---

## 7. MRC/MCR AArch32 翻译

```c
// target/arm/tcg/translate.c:1719+
static void do_coproc_insn(DisasContext *s, int cpnum, bool isread,
                           int opc1, int crn, int crm, int opc2,
                           bool isrt, int rt, int rt2)
{
    // 类似 handle_sys 的流程:
    // ENCODE_CP_REG(cpnum, is64, ns, crn, crm, opc1, opc2) → key
    // get_arm_cp_reginfo → ri
    // cp_access_ok → accessfn → read/write
}

// 2346-2383: decodetree 入口
// trans_MCR → do_coproc_insn(isread=false)
// trans_MRC → do_coproc_insn(isread=true)
// trans_MCRR → 64 位变体
// trans_MRRC → 64 位变体
```

---

## 8. CP15 状态存储与银行结构

```c
// target/arm/cpu.h:320-540 — CPUARMState.cp15 结构

// === SCTLR — 系统控制寄存器 ===
union {                              // 332-340
    struct { _unused; sctlr_ns; hsctlr; sctlr_s; };
    uint64_t sctlr_el[4];           // [0]=unused, [1]=NS, [2]=Hyp, [3]=Sec
};

// === TTBR0 — 翻译表基址 0 ===
union {                              // 347-355
    struct { _unused; ttbr0_ns; _unused; ttbr0_s; };
    uint64_t ttbr0_el[4];           // [1]=NS, [3]=Sec
};

// === TTBR1 — 翻译表基址 1 ===
union {                              // 356-364
    struct { _unused; ttbr1_ns; _unused; ttbr1_s; };
    uint64_t ttbr1_el[4];
};

// === TCR — 翻译控制寄存器 ===
uint64_t tcr_el[4];                  // 368: [1..3]

// === MAIR — 内存属性 ===
union {                              // 451-470
    struct { _unused; mair0_ns+mair1_ns; _unused; mair0_s+mair1_s; };
    uint64_t mair_el[4];            // 469
};

// === VBAR — 向量基址 ===
union {                              // 472-480
    struct { _unused; vbar_ns; hvbar; vbar_s; };
    uint64_t vbar_el[4];
};

// === 调试寄存器 ===
uint64_t dbgbvr[16];                // 529: 断点值
uint64_t dbgbcr[16];                // 530: 断点控制
uint64_t dbgwvr[16];                // 531: 观察点值
uint64_t dbgwcr[16];                // 532: 观察点控制
uint64_t mdscr_el1;                 // 534: 调试系统控制
```

### 银行设计

union 结构实现了 **EL 索引访问**和 **Secure/NS 银行**的统一：
- `sctlr_el[1]` == `sctlr_ns` (Non-Secure EL1)
- `sctlr_el[3]` == `sctlr_s`  (Secure EL3 / Secure EL1)
- `sctlr_el[2]` == `hsctlr`   (EL2)

ARMCPRegInfo 使用 `bank_fieldoffsets[2]` 在注册时选择：
```c
{ .name = "SCTLR_EL1",
  .bank_fieldoffsets = { offsetof(...sctlr_s), offsetof(...sctlr_ns) }
}
// add_cpreg_to_hashtable 根据 secstate 选择 fieldoffset
```

---

## 9. 访问控制机制

### 编译时检查

```c
// target/arm/cpregs.h:1122-1126
static inline bool cp_access_ok(int current_el, const ARMCPRegInfo *ri, int isread)
{
    return (ri->access >> ((current_el * 2) + isread)) & 1;
}
// PL0_R 对应 bit 1, PL1_R 对应 bit 3, PL2_R 对应 bit 5, PL3_R 对应 bit 7
// 高 EL 权限自动包含低 EL（通过 OR 链）
```

### 运行时检查 — 三级优先级

```c
// target/arm/tcg/op_helper.c:802-900
HELPER(access_check_cp_reg)(env, key, syndrome, isread)
{
    // === 第一级: accessfn 回调 ===
    if (ri->accessfn)                                    // 813
        res = ri->accessfn(env, ri, isread);             // 814
    // 返回 CP_ACCESS_TRAP_EL2 等

    // === 第二级: HSTR_EL2 陷入 ===
    if (EL0 && !AArch64 && cp==15 && EL2 enabled)       // 835-837
        if (hstr_el2 & (1 << crn))                       // 847
            res = CP_ACCESS_TRAP_EL2;                    // 848

    // === 第三级: FGT 精细陷入 ===
    if (arm_fgt_active())                                // 858
        // 查找 fgt_read[idx] / fgt_write[idx] / fgt_exec[idx]
        // 检查 bitpos 位                                 860-875
        → CP_ACCESS_TRAP_EL2                             // 如果位设置
}
```

### accessfn 示例

```c
// target/arm/helper.c:298-356
// access_tvm_trvm → HCR_EL2.TVM/TRVM 陷入 VM 寄存器访问
// access_tsw → HCR_EL2.TSW 陷入缓存维护
// access_tacr → HCR_EL2.TACR 陷入辅助控制
// access_el3_aa32ns → 仅 AArch32 NS 模式访问
```

---

## 10. EL 切换与寄存器上下文

### 异常入口

```c
// target/arm/helper.c:9197-9245
void arm_cpu_do_interrupt_aarch64(CPUState *cs)
{
    // 保存 PSTATE → SPSR_ELx
    // 保存 PC → ELR_ELx
    // 设置新 PSTATE（关中断、设 EL、设 SP）
    // PC = VBAR_ELx + offset
    aarch64_sve_change_el(env, cur_el, new_el, ...);    // SVE/SME 状态调整
}
```

### 异常返回

```c
// target/arm/tcg/helper-a64.c:622-758
HELPER(exception_return)(env, new_pc, spsr)
{
    // 恢复 PSTATE（从 SPSR）
    // 恢复 PC
    // 安全状态检查
    aarch64_sve_change_el(env, cur_el, new_el, ...);    // SVE/SME 调整
    helper_rebuild_hflags_a64(env, new_el);              // 714/728: 重建翻译标志
}
```

### SVE EL 切换

```c
// target/arm/helper.c:10073-10102
void aarch64_sve_change_el(CPUARMState *env, int old_el, int new_el, ...)
{
    // 根据新 EL 的 ZCR_ELx.LEN 重新计算 SVE 向量长度
    // 可能截断高位向量数据
    // SME 状态类似处理
}
```

### 安全状态切换

```c
// target/arm/helper.c:822-835
// SCR_EL3 写入时:
// 如果 NS 位改变 → TLB 全刷新（安全/非安全 TLB 隔离）
```

---

## 11. FGT 精细陷入

### FGTBit 编码

```c
// target/arm/cpregs.h:377+
// FGT 字段编码: idx(哪个 FGT 寄存器) + bitpos(哪一位) + rev(反转) + nxs
// HFGRTR_EL2 / HFGWTR_EL2 — 读/写精细陷入
// HDFGRTR_EL2 / HDFGWTR_EL2 — 调试精细陷入
// HFGITR_EL2 — 指令精细陷入
```

### 运行时检查

```c
// op_helper.c:858-895
if (arm_fgt_active(env, current_el)) {
    unsigned int idx = FIELD_EX32(ri->fgt, FGT, IDX);
    unsigned int bitpos = FIELD_EX32(ri->fgt, FGT, BITPOS);
    // 读: fgt_read[idx], 写: fgt_write[idx], 执行: fgt_exec[idx]
    trapword = env->cp15.fgt_read[idx];
    if ((trapword >> bitpos) & 1)
        → CP_ACCESS_TRAP_EL2  // 陷入到 EL2
}
```

---

## 12. NV2 嵌套虚拟化重定向

```c
// translate-a64.c:2807-2820
if (s->nv2 && ri->nv2_redirect_offset) {
    // NV2_REDIR_NV1: 仅 HCR_EL2.NV1=1 时重定向
    // NV2_REDIR_NO_NV1: 仅 NV1=0 时重定向
    // 其他: 始终重定向
    → nv2_mem_redirect = true
}

// 重定向时：
// 寄存器访问变为内存访问：
// addr = VNCR_EL2 + nv2_redirect_offset
// 64 位原子加载/存储到该内存地址
```

这使得 L1 hypervisor 可以通过内存映射管理 L2 guest 的 EL2 寄存器。

---

## 13. VHE 寄存器重定向

```c
// translate-a64.c:2862-2881
if (ri->vhe_redir_to_el2 && s->current_el == 2 && s->e2h) {
    // EL2 + E2H → FOO_EL1 重定向到 FOO_EL2
    key = ri->vhe_redir_to_el2;
    ri = redirect_cpreg(s, key, isread);
}
else if (ri->vhe_redir_to_el01 && s->current_el >= 2) {
    if (!s->e2h) { UNDEF; return; }
    // FOO_EL12/FOO_EL02 → 回指 FOO_EL1/FOO_EL0
    key = ri->vhe_redir_to_el01;
    ri = redirect_cpreg(s, key, isread);
}
```

| 访问 | 条件 | 实际访问 |
|------|------|---------|
| SCTLR_EL1 | EL2+E2H | SCTLR_EL2 |
| TTBR0_EL1 | EL2+E2H | TTBR0_EL2 |
| SCTLR_EL12 | EL2+E2H | SCTLR_EL1（真正的 EL1） |
| SCTLR_EL12 | !E2H | UNDEF |

---

## 14. 系统寄存器定义示例

### SCTLR_EL1（银行寄存器）

```c
// target/arm/helper.c:7336-7347
{ .name = "SCTLR_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .opc1 = 0, .crn = 1, .crm = 0, .opc2 = 0,
  .access = PL1_RW,
  .fgt = FGT_SCTLR_EL1,
  .bank_fieldoffsets = { offsetof(CPUARMState, cp15.sctlr_s),
                         offsetof(CPUARMState, cp15.sctlr_ns) },
  .writefn = sctlr_write, .resetvalue = cpu->reset_sctlr,
}
```

### TTBR0_EL1（银行+NV2+FGT）

```c
// target/arm/helper.c:2850-2859
{ .name = "TTBR0_EL1", .state = ARM_CP_STATE_BOTH,
  .opc0 = 3, .opc1 = 0, .crn = 2, .crm = 0, .opc2 = 0,
  .access = PL1_RW,
  .fgt = FGT_TTBR0_EL1,
  .nv2_redirect_offset = 0x200 | NV2_REDIR_NV1,
  .bank_fieldoffsets = { offsetof(CPUARMState, cp15.ttbr0_s),
                         offsetof(CPUARMState, cp15.ttbr0_ns) },
  .writefn = vmsa_ttbr_write, .resetvalue = 0,
}
```

### arm_cp_write_ignore / arm_cp_read_zero

```c
// target/arm/helper.c:8042-8052
void arm_cp_write_ignore(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t v)
{ /* 写忽略 */ }

uint64_t arm_cp_read_zero(CPUARMState *env, const ARMCPRegInfo *ri)
{ return 0; /* 读零 */ }
```

---

## 15. 完整 MRS 执行流程总结

```
Guest: MRS X0, SCTLR_EL1    (op0=3, op1=0, crn=1, crm=0, opc2=0)

=== 翻译阶段 ===
handle_sys(isread=true)
  ① key = ENCODE_AA64_CP_REG(3,0,1,0,0)
  ② ri = get_arm_cp_reginfo(cp_regs, key)       → ARMCPRegInfo for SCTLR_EL1
  ③ cp_access_ok(1, ri, 1)                      → PL1_R → true
  ④ ri->accessfn != NULL || ri->fgt != 0
     → gen_helper_access_check_cp_reg(key, syn, 1)
  ⑤ VHE: if EL2+E2H → 重定向到 SCTLR_EL2
  ⑥ ri->readfn == sctlr_read? 或 fieldoffset?
     → gen_helper_get_cp_reg64(rt, ri_ptr)
     或 tcg_gen_ld_i64(rt, env, fieldoffset)

=== 执行阶段 ===
  ① helper_access_check_cp_reg:
     - accessfn(env, ri, true)                   → CP_ACCESS_OK
     - HSTR_EL2 检查（EL0 only）                 → 跳过
     - FGT: fgt_read[HFGRTR] & SCTLR_EL1 位     → 若设置则 TRAP_EL2
  ② helper_get_cp_reg64:
     - ri->readfn(env, ri) 或 *(env + fieldoffset)
     → 返回 sctlr_el[1] 值

=== 结果 ===
  X0 = env->cp15.sctlr_ns（或 sctlr_s，取决于安全状态）
```

---

## 交叉参考

- [46-ARM64-TCG插件与调试子系统深度分析](46-ARM64-TCG插件与调试子系统深度分析-PluginAPI-GDBStub-断点单步-ARM调试寄存器与Tracing.md) — ARM 调试寄存器 DBGBVR/BCR/WVR/WCR 写入回调
- [44-ARM64-TCG执行循环深度分析](44-ARM64-TCG执行循环深度分析-cpu_exec主循环-TB查找链接-中断异常-MTTCG多线程与icount.md) — TB 终止与 ARM_CP_SUPPRESS_TB_END/IO 影响
- [43-ARM64-TCG-softmmu-TLB深度分析](43-ARM64-TCG-softmmu-TLB深度分析-数据结构-快慢路径-页表遍历-TLBI指令与MMIO分发.md) — SCTLR/TCR/TTBR 写入触发 TLB 刷新

---

> 文档生成时间基于 QEMU 11.0.50 源码，commit 范围覆盖 v11.0.50 开发版本。
