# ARM64 Branch Target Identification (BTI) 完整实现分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/77 |
| 分析对象 | FEAT_BTI (Branch Target Identification) 在 QEMU 中的完整实现 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §D5.4 (Branch Target Identification) |
| 核心文件 | `target/arm/tcg/translate-a64.c`, `target/arm/tcg/helper-a64.c` |
| 核心结论 | **QEMU 通过 TB 首指令静态检查 + guarded page 动态 probe 实现完整 BTI，包括 BTYPE 状态机、GP 页表位、SCTLR.BT 控制** |

---

## 1. BTI 架构概述

### 1.1 什么是 BTI

BTI (FEAT_BTI, Armv8.5) 是一种控制流完整性 (CFI) 机制，限制间接分支只能跳转到标记了 BTI 指令的位置：

```
间接分支 (BR/BLR/RET) → 目标页的 GP=1 (Guarded Page)
                       → 目标指令必须是允许的 landing pad
                       → 否则触发 Branch Target Exception
```

### 1.2 BTYPE 状态机

PSTATE.BTYPE (2 bits) 记录最近间接分支的类型：

| BTYPE | 设置场景 | 含义 |
|:-----:|---------|------|
| 0 | 顺序执行/直接分支/异常 | 无约束 |
| 1 | BR Xn (非 X16/X17) 到非 guarded page | 间接跳转 (JMP) |
| 2 | BLR Xn | 间接调用 (CALL) |
| 3 | BR Xn (非 X16/X17) 到 guarded page | 间接跳转到受保护页 |

### 1.3 Landing Pad 兼容性

在 Guarded Page 上，目标指令必须满足：

| 目标指令 | BTYPE=1 | BTYPE=2 | BTYPE=3 |
|---------|:-------:|:-------:|:-------:|
| BTI (无参数) | ❌ | ❌ | ❌ |
| BTI c | ✅ | ✅ | ❌ |
| BTI j | ✅ | ❌ | ✅ |
| BTI jc | ✅ | ✅ | ✅ |
| PACIASP/PACIBSP | ✅ | ✅ | ✅* |
| BRK/HLT | ✅ | ✅ | ✅ |
| 其他 | ❌ | ❌ | ❌ |

*: 当 SCTLR.BT=1 时，PACI*SP 不兼容 BTYPE=3

---

## 2. Guarded Page 检测

### 2.1 页表中的 GP 位

```c
// target/arm/ptw.c:2344
result->f.extra.arm.guarded = extract64(attrs, 50, 1); /* GP */
```

GP (Guarded Page) 是 Stage-1 页表描述符的 bit[50]：
- GP=1: 该页是 guarded page，间接分支到此页需要 BTI 检查
- GP=0: 非 guarded page，不检查

### 2.2 Stage-2 翻译保持 S1 的 GP 属性

```c
// ptw.c:3563-3644
bool s1_guarded;
s1_guarded = result->f.extra.arm.guarded;  // 保存 S1 结果
// ... 执行 S2 翻译 ...
result->f.extra.arm.guarded = s1_guarded;  // 恢复 S1 的 GP 标记
```

### 2.3 运行时 Guarded Page 检查

```c
// target/arm/tcg/helper-a64.c:1738
static bool is_guarded_page(CPUARMState *env, vaddr addr, uintptr_t ra)
{
#ifdef CONFIG_USER_ONLY
    return page_get_flags(addr) & PAGE_BTI;
#else
    CPUTLBEntryFull *full;
    void *host;
    int flags = probe_access_full(env, addr, 0, MMU_INST_FETCH,
                                  mmu_idx, false, &host, &full, ra);
    return full->extra.arm.guarded;
#endif
}
```

---

## 3. BTYPE 设置逻辑

### 3.1 BR Xn (间接跳转)

```c
// target/arm/tcg/translate-a64.c:1777
static void set_btype_for_br(DisasContext *s, int rn)
{
    if (dc_isar_feature(aa64_bti, s)) {
        if (rn == 16 || rn == 17) {
            // BR X16/X17: 总是 BTYPE=1 (linker veneer 约定)
            set_btype(s, 1);
        } else {
            // BR Xn (其他): 需要检查目标页是否 guarded
            // guarded → BTYPE=3, 非 guarded → BTYPE=1
            gen_helper_guarded_page_br(tcg_env, pc);
            s->btype = -1;  // 标记为运行时决定
        }
    }
}
```

### 3.2 BLR Xn (间接调用)

```c
// translate-a64.c:1792
static void set_btype_for_blr(DisasContext *s)
{
    if (dc_isar_feature(aa64_bti, s)) {
        // BLR 总是设置 BTYPE=2，无论目标页是否 guarded
        set_btype(s, 2);
    }
}
```

### 3.3 guarded_page_br Helper

```c
// helper-a64.c:1768
void HELPER(guarded_page_br)(CPUARMState *env, vaddr pc)
{
    // BR Xn (非 X16/X17):
    // 目标是 guarded page → BTYPE=3 (严格检查)
    // 目标非 guarded page → BTYPE=1 (宽松检查)
    env->btype = is_guarded_page(env, pc, GETPC()) ? 3 : 1;
}
```

### 3.4 RET 指令

RET 不设置 BTYPE (等同于 BR X30 但不触发 BTI 检查)：
- RET 被视为函数返回，不需要 BTI landing pad

---

## 4. BTI 检查实现

### 4.1 翻译时检查 (TB 首指令)

```c
// translate-a64.c:10832
if (dc_isar_feature(aa64_bti, s)) {
    if (s->base.num_insns == 1) {
        // TB 首指令可能有非零 BTYPE

        if (s->btype != 0
            && !btype_destination_ok(insn, s->bt, s->btype)) {
            // 指令不兼容 → 需要检查是否在 guarded page
            gen_helper_guarded_page_check(tcg_env);
        }
    } else {
        // 非首指令: BTYPE 必须为 0 (已被 reset)
        tcg_debug_assert(s->btype == 0);
    }
}
```

### 4.2 btype_destination_ok (静态检查)

```c
// translate-a64.c:10617
static bool btype_destination_ok(uint32_t insn, bool bt, int btype)
{
    if ((insn & 0xfffff01fu) == 0xd503201fu) {
        /* HINT space */
        switch (extract32(insn, 5, 7)) {
        case 0b011001: /* PACIASP */
        case 0b011011: /* PACIBSP */
            // SCTLR.BT=1 时不兼容 BTYPE=3
            return !bt || btype != 3;
        case 0b100000: /* BTI (无参数) */
            return false;  // 不兼容任何 BTYPE
        case 0b100010: /* BTI c */
            return btype != 3;  // 不兼容 BTYPE=3
        case 0b100100: /* BTI j */
            return btype != 2;  // 不兼容 BTYPE=2
        case 0b100110: /* BTI jc */
            return true;  // 兼容所有
        }
    } else {
        switch (insn & 0xffe0001fu) {
        case 0xd4200000u: /* BRK */
        case 0xd4400000u: /* HLT */
            return true;  // 断点/调试优先
        }
    }
    return false;  // 不兼容
}
```

### 4.3 guarded_page_check (运行时确认)

```c
// helper-a64.c:1754
void HELPER(guarded_page_check)(CPUARMState *env)
{
    // 只有当页确实是 guarded 时才触发异常
    if (is_guarded_page(env, env->pc, 0)) {
        raise_exception(env, EXCP_UDEF, syn_btitrap(env->btype),
                        exception_target_el(env));
    }
    // 非 guarded page: 不检查，正常执行
}
```

### 4.4 两阶段检查设计

```
阶段 1 (翻译时，静态):
  BTYPE ≠ 0 且指令不在兼容列表中
  → 需要进一步确认

阶段 2 (运行时，动态):
  确认当前页是否为 guarded page (查 TLB/页表 GP 位)
  → GP=1: 触发 Branch Target Exception
  → GP=0: 正常执行 (非保护页不检查)
```

这种设计优化了性能：大部分代码在非 guarded page 上，运行时检查后不触发异常。

---

## 5. BTYPE 生命周期

### 5.1 BTYPE 存储

```c
// target/arm/cpu.h:279, 2504
// BTYPE 存储在 env->btype (不在 PSTATE 中缓存)
// TB flag 中携带 BTYPE: FIELD(TBFLAG_A64, BTYPE, 10, 2)
```

### 5.2 BTYPE 传播

```
1. 间接分支设置 BTYPE (1, 2, 或 3)
2. BTYPE 嵌入到目标 TB 的 flags 中
3. 目标 TB 首指令检查 BTYPE
4. 首指令执行后 reset_btype(s) → BTYPE=0
5. TB 中后续指令 BTYPE 始终为 0
```

### 5.3 reset_btype

```c
// translate-a64.c:178
static void reset_btype(DisasContext *s)
{
    if (s->btype != 0) {
        set_btype_raw(0);  // 写 env->btype = 0
        s->btype = 0;
    }
}

// 在以下位置调用 reset:
// - 目标 TB 首指令兼容时 (合法 landing pad)
// - 大多数非分支指令执行后
// 在 translate_insn 末尾: if (s->btype > 0) reset_btype(s);
```

---

## 6. SCTLR.BT 控制

### 6.1 BT 位的作用

```c
// hflags.c:332
if (cpu_isar_feature(aa64_bti, env_archcpu(env))) {
    if (sctlr & (el == 0 ? SCTLR_BT0 : SCTLR_BT1)) {
        DP_TBFLAG_A64(flags, BT, 1);
    }
}
```

- `SCTLR_ELx.BT = 0`: PACIASP/PACIBSP 兼容所有 BTYPE (包括 3)
- `SCTLR_ELx.BT = 1`: PACIASP/PACIBSP 不兼容 BTYPE=3

### 6.2 设计意义

BT=0 (默认): 允许 PAC 保护的函数作为间接分支目标
- 因为 `PACIASP` 已经提供了认证保护
- 等价于 `BTI jc` 的效果

BT=1 (严格): 要求显式 BTI 指令
- 更严格的 CFI 策略

---

## 7. X16/X17 特殊处理

### 7.1 为什么特殊

```c
// translate-a64.c:1781
if (rn == 16 || rn == 17) {
    set_btype(s, 1);  // 总是 BTYPE=1，不检查 guarded page
}
```

- X16/X17 是 linker/PLT veneer 约定的 scratch 寄存器
- `BR X16` / `BR X17` 不需要目标有 `BTI j`
- 只需要 `BTI c` 或 `BTI jc` (BTYPE=1 兼容)
- 这避免了对所有 PLT stub 添加 BTI 的需求

---

## 8. BTI 指令编码

### 8.1 BTI 在 HINT 空间中

```
BTI    = 0xd503241f  (HINT #32)
BTI c  = 0xd503245f  (HINT #34)
BTI j  = 0xd503249f  (HINT #36)
BTI jc = 0xd50324df  (HINT #38)
```

所有 BTI 变体在不支持 BTI 的 CPU 上执行为 NOP (HINT 空间的后向兼容设计)。

### 8.2 兼容性矩阵总结

| | BTI | BTI c | BTI j | BTI jc | PACI*SP (BT=0) | PACI*SP (BT=1) |
|--|:---:|:-----:|:-----:|:------:|:--------------:|:--------------:|
| BTYPE=1 (BR Xn→non-GP) | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| BTYPE=2 (BLR) | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ |
| BTYPE=3 (BR Xn→GP) | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |

---

## 9. 与 ARM 规范的一致性评估

### 9.1 实现完整度

| 规范要求 | QEMU 状态 | 说明 |
|---------|:--------:|------|
| PSTATE.BTYPE (2-bit) | ✅ | env->btype + TB flag |
| Guarded Page (GP bit) | ✅ | PTE bit[50] → full->extra.arm.guarded |
| BTI/BTI c/BTI j/BTI jc | ✅ | btype_destination_ok |
| BR X16/X17 特殊 | ✅ | BTYPE=1 无条件 |
| BLR 总是 BTYPE=2 | ✅ | set_btype_for_blr |
| BR→GP = BTYPE=3 | ✅ | guarded_page_br helper |
| RET 不设 BTYPE | ✅ | trans_RET 无 set_btype |
| SCTLR.BT 控制 | ✅ | TB flag BT |
| PACIASP/PACIBSP 兼容 | ✅ | !bt \|\| btype != 3 |
| BRK/HLT 优先 | ✅ | return true |
| Branch Target Exception | ✅ | syn_btitrap(btype) |
| S2 保持 S1 GP | ✅ | s1_guarded 保存/恢复 |

### 9.2 优化设计

| 优化 | 说明 |
|------|------|
| 两阶段检查 | 翻译时静态筛选 + 运行时 GP 确认 |
| TB flag BTYPE | 避免每条指令都检查 |
| BTYPE reset in-TB | 首指令后清零，后续无开销 |
| X16/X17 快速路径 | 不需要运行时 probe |

### 9.3 与真实硬件的差异

| 方面 | 真实硬件 | QEMU |
|------|---------|------|
| GP 检查时机 | 分支执行时并行检查 TLB GP 位 | helper 调用 probe TLB |
| BTYPE 存储 | PSTATE 硬件位 | env->btype 软件变量 |
| BTI 指令解码 | 流水线首级并行检查 | TB 首指令翻译时静态检查 |
| 性能影响 | 几乎零开销 | 非 X16/X17 的 BR 需 helper 调用 |

### 9.4 总结

QEMU 的 BTI 实现**与规范完全一致**，通过巧妙的两阶段设计（翻译时静态 + 运行时动态）平衡了正确性和性能。核心创新在于：
- 利用 TB 边界天然对应间接分支目标
- 只在 TB 首指令执行 BTI 检查
- 大部分情况下（非 guarded page）运行时检查 fast-path 返回
