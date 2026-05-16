# ARM 内存模型与内存序深度分析

## 文档信息
- **QEMU 版本**: 11.0.50
- **分析目标**: ARM 内存屏障、独占访问、原子操作、MTE/PAC/BTI 在 QEMU 中的完整实现
- **核心源文件**:
  - `target/arm/tcg/translate-a64.c` — 屏障/独占/原子/MTE/BTI 指令翻译
  - `include/tcg/tcg-mo.h` — TCG 内存序定义
  - `tcg/tcg-op.c` — tcg_gen_mb() 屏障生成
  - `target/arm/tcg/mte_helper.c` — MTE 标签检查
  - `target/arm/tcg/pauth_helper.c` — PAC 指针认证
  - `target/arm/ptw.c` — 页表遍历与内存属性
  - `target/arm/helper.c` — MMU 索引选择

---

## 第一部分：ARM 内存屏障指令

### 1. DMB/DSB 实现

```c
// translate-a64.c:2243-2261 — trans_DSB_DMB()
static bool trans_DSB_DMB(DisasContext *s, arg_DSB_DMB *a)
{
    TCGBar bar;

    switch (a->types) {
    case 1: /* MBReqTypes_Reads — 读屏障 */
        bar = TCG_BAR_SC | TCG_MO_LD_LD | TCG_MO_LD_ST;
        break;
    case 2: /* MBReqTypes_Writes — 写屏障 */
        bar = TCG_BAR_SC | TCG_MO_ST_ST;
        break;
    default: /* MBReqTypes_All — 完全屏障 */
        bar = TCG_BAR_SC | TCG_MO_ALL;
        break;
    }
    tcg_gen_mb(bar);
    return true;
}
```

**DMB/DSB 在 QEMU 中等价处理**：
- DMB (Data Memory Barrier)：仅保证数据访问顺序
- DSB (Data Synchronization Barrier)：确保所有先前操作完成
- QEMU TCG 将两者映射到相同的 `tcg_gen_mb()`

### 2. DSB nXS（ARMv8.7）

```c
// translate-a64.c:2263-2269
static bool trans_DSB_nXS(DisasContext *s, arg_DSB_nXS *a)
{
    tcg_gen_mb(TCG_BAR_SC | TCG_MO_ALL);  // 与普通DSB相同处理
    return true;
}
```

### 3. ISB 实现

```c
// translate-a64.c:2272-2281
static bool trans_ISB(DisasContext *s, arg_ISB *a)
{
    // ISB 不是内存屏障，而是指令同步
    // 结束当前TB，确保后续指令重新取指
    reset_btype(s);
    gen_goto_tb(s, 0, 4);  // 跳转到下一条指令（新TB）
    return true;
}
```

**ISB 关键行为**：不生成 `tcg_gen_mb()`，而是终止 Translation Block，强制后续指令在新 TB 中翻译。这确保了：
- 系统寄存器修改生效
- 自修改代码正确执行
- 中断在 TB 边界处理

### 4. SB（Speculation Barrier）

```c
// translate-a64.c:2284-2296
static bool trans_SB(DisasContext *s, arg_SB *a)
{
    // TCG 没有推测屏障操作码，用 MB + TB结束代替
    tcg_gen_mb(TCG_MO_ALL | TCG_BAR_SC);
    gen_goto_tb(s, 0, 4);
    return true;
}
```

---

## 第二部分：TCG 内存序模型

### 5. TCG 内存序位定义

```c
// include/tcg/tcg-mo.h:34-46
// 内存操作序
#define TCG_MO_LD_LD  (1 << 0)  // Load→Load 序
#define TCG_MO_ST_LD  (1 << 1)  // Store→Load 序
#define TCG_MO_LD_ST  (1 << 2)  // Load→Store 序
#define TCG_MO_ST_ST  (1 << 3)  // Store→Store 序
#define TCG_MO_ALL    (0xf)      // 所有组合

// 屏障类型
#define TCG_BAR_LDAQ  (1 << 4)  // Load-Acquire
#define TCG_BAR_STRL  (1 << 5)  // Store-Release
#define TCG_BAR_SC    (1 << 6)  // Sequential Consistency
```

### 6. tcg_gen_mb() 实现

```c
// tcg/tcg-op.c:289-306
void tcg_gen_mb(TCGBar bar)
{
    // 仅在并行执行时生成屏障指令
    if (parallel_cpus) {
        tcg_gen_op1(INDEX_op_mb, bar);
    }
}
```

**关键优化**：
- **单线程 TCG**：所有屏障都是 NOP（`parallel_cpus = false`）
- **MTTCG（多线程 TCG）**：生成实际的 `INDEX_op_mb` 操作，由后端翻译为主机屏障指令

### 7. ARM 屏障到 TCG 映射

| ARM 指令 | TCG 屏障 | 效果 |
|----------|---------|------|
| DMB LD | `TCG_BAR_SC \| TCG_MO_LD_LD \| TCG_MO_LD_ST` | 读→读/写序 |
| DMB ST | `TCG_BAR_SC \| TCG_MO_ST_ST` | 写→写序 |
| DMB SY/ISH/OSH | `TCG_BAR_SC \| TCG_MO_ALL` | 完全序 |
| DSB * | 同 DMB（QEMU简化）| 完全序 |
| ISB | TB 结束 | 指令同步 |
| SB | `TCG_MO_ALL \| TCG_BAR_SC` + TB 结束 | 推测屏障 |

---

## 第三部分：Load-Acquire / Store-Release

### 8. STLR（Store-Release）

```c
// translate-a64.c:3526-3549
static bool trans_STLR(DisasContext *s, arg_stlr *a)
{
    // Store-Release: 屏障在存储之前
    tcg_gen_mb(TCG_MO_ALL | TCG_BAR_STRL);  // Release屏障
    // ... 执行存储 ...
    do_gpr_st(s, ...);
    return true;
}
```

### 9. LDAR（Load-Acquire）

```c
// translate-a64.c:3552-3572
static bool trans_LDAR(DisasContext *s, arg_stlr *a)
{
    // Load-Acquire: 屏障在加载之后
    // ... 执行加载 ...
    do_gpr_ld(s, ...);
    tcg_gen_mb(TCG_MO_ALL | TCG_BAR_LDAQ);  // Acquire屏障
    return true;
}
```

### 10. Acquire/Release 语义

```
Store-Release (STLR):
  所有先前的读写 ──┐
                     ├── 屏障 (STRL)
  STLR 存储操作   ──┘

Load-Acquire (LDAR):
  LDAR 加载操作   ──┐
                     ├── 屏障 (LDAQ)
  所有后续的读写 ──┘
```

### 11. LDAPR（Load-AcquirePC，FEAT_LRCPC）

```c
// translate-a64.c:4164-4263
// LDAPR 实现与 LDAR 相同（加载后加Acquire屏障）
// QEMU 中 LDAPR 和 LDAR 的区别在架构上：
// LDAPR 有更弱的序（RCpc vs RCsc），但 QEMU 统一实现
```

---

## 第四部分：独占访问（LDXR/STXR）

### 12. 独占加载：gen_load_exclusive()

```c
// translate-a64.c:3241-3284
static void gen_load_exclusive(DisasContext *s, int rt, int rt2, int rn,
                                int size, bool is_pair)
{
    s->is_ldex = true;
    dirty_addr = cpu_reg_sp(s, rn);
    clean_addr = gen_mte_check1(s, dirty_addr, ...);  // MTE 检查

    if (is_pair) {
        // LDXP: 加载128位到两个寄存器
        tcg_gen_qemu_ld_i64(cpu_exclusive_val, clean_addr, ...);
        // 或 128位: tcg_gen_qemu_ld_i128(...)
    } else {
        // LDXR: 加载单个值
        tcg_gen_qemu_ld_i64(cpu_exclusive_val, clean_addr, ...);
        tcg_gen_mov_i64(cpu_reg(s, rt), cpu_exclusive_val);
    }
    // 记录独占地址
    tcg_gen_mov_i64(cpu_exclusive_addr, clean_addr);
}
```

### 13. 独占存储：gen_store_exclusive()

```c
// translate-a64.c:3286-3400
static void gen_store_exclusive(DisasContext *s, int rd, int rt, int rt2,
                                 int rn, int size, int is_pair)
{
    // 算法:
    // if (exclusive_addr == addr && exclusive_val == [addr]) {
    //     [addr] = Rt;
    //     Rd = 0;  // 成功
    // } else {
    //     Rd = 1;  // 失败
    // }
    // exclusive_addr = -1;

    // 1. 地址匹配检查
    tcg_gen_brcond_i64(TCG_COND_NE, clean_addr, cpu_exclusive_addr, fail_label);

    // 2. 原子比较交换
    tcg_gen_atomic_cmpxchg_i64(tmp, cpu_exclusive_addr,
                                cpu_exclusive_val, cpu_reg(s, rt), ...);

    // 3. 检查是否成功
    tcg_gen_setcond_i64(TCG_COND_NE, tmp, tmp, cpu_exclusive_val);
    // Rd = 0(成功) 或 1(失败)
}
```

### 14. 独占监视器状态

```c
// cpu.h:704-713
uint64_t exclusive_addr;   // 独占监视地址（-1 表示无效）
uint64_t exclusive_val;    // 独占加载的值
uint64_t exclusive_high;   // LDXP 高64位
```

### 15. LDXR/STXR 变体

```c
// translate-a64.c:3502-3596
// STXR: 基本独占存储
trans_STXR: if (a->lasr) tcg_gen_mb(STRL); gen_store_exclusive(...);

// LDXR: 基本独占加载
trans_LDXR: gen_load_exclusive(...); if (a->lasr) tcg_gen_mb(LDAQ);

// STXP: 独占存储对
trans_STXP: if (a->lasr) tcg_gen_mb(STRL); gen_store_exclusive(..., true);

// LDXP: 独占加载对
trans_LDXP: gen_load_exclusive(..., true); if (a->lasr) tcg_gen_mb(LDAQ);
```

**lasr 标志**：当为 `STLXR/LDAXR` 时，附加 Release/Acquire 语义。

### 16. CLREX：清除独占监视

```c
// 翻译时: exclusive_addr = -1
// 清除独占监视器，任何后续 STXR 必定失败
```

---

## 第五部分：LSE 原子操作

### 17. CAS/CASP（Compare-And-Swap）

```c
// translate-a64.c:3599-3618
// CAS: 使用 tcg_gen_atomic_cmpxchg_i64()
// CASP: 使用 tcg_gen_atomic_cmpxchg_i128()

// 原子比较交换：if ([addr] == expected) [addr] = new;
```

### 18. 原子 Fetch 操作

| ARM 指令 | TCG 操作 | 语义 |
|----------|---------|------|
| LDADD | `tcg_gen_atomic_fetch_add` | 原子加并返回旧值 |
| LDCLR | `tcg_gen_atomic_fetch_and` (取反) | 原子清位 |
| LDEOR | `tcg_gen_atomic_fetch_xor` | 原子异或 |
| LDSET | `tcg_gen_atomic_fetch_or` | 原子置位 |
| LDSMAX | `tcg_gen_atomic_fetch_smax` | 有符号最大值 |
| LDSMIN | `tcg_gen_atomic_fetch_smin` | 有符号最小值 |
| LDUMAX | `tcg_gen_atomic_fetch_umax` | 无符号最大值 |
| LDUMIN | `tcg_gen_atomic_fetch_umin` | 无符号最小值 |
| SWP | `tcg_gen_atomic_xchg` | 原子交换 |

所有 LSE 原子操作支持 Acquire/Release 后缀（A/L/AL）。

---

## 第六部分：内存类型与属性

### 19. ARM MMU 索引选择

```c
// helper.c:9957-10013 — arm_mmu_idx_el()
// 根据当前 EL、安全状态、HCR_EL2 设置选择 ARMMMUIdx
// 决定使用哪个翻译 Regime (EL1&0 / EL2 / EL2&0 / EL3)
```

### 20. Device/Normal 内存区分

```c
// ptw.c:574-600
// S1_attrs_are_device() — Stage 1 属性是否为Device
// S2_attrs_are_device() — Stage 2 属性是否为Device
```

### 21. MAIR 内存属性间接

```c
// ptw.c:2340
result->cacheattrs.attrs = extract64(mair, attrindx * 8, 8);
// 从MAIR寄存器提取8位内存属性（attrindx来自页表描述符）
```

MAIR 编码：
- `0x00`：Device-nGnRnE（最严格 Device）
- `0x04`：Device-nGnRE
- `0xFF`：Normal, Write-Back（最宽松 Normal）

### 22. Stage 1/2 属性合并

```c
// ptw.c:3413-3464 — combine_cacheattrs()
// 合并 Stage 1 和 Stage 2 翻译的内存属性
// Device 类型取更严格的一方
// Normal 类型合并缓存策略
```

### 23. MMU 禁用时的默认属性

```c
// ptw.c:3470-3551
// SCTLR.M = 0 时:
// - 数据访问默认为 Device-nGnRnE（最严格）
// - 指令访问默认为 Normal（可缓存）
```

---

## 第七部分：MTE（内存标签扩展）

### 24. MTE 检查翻译生成

```c
// translate-a64.c:257-349
TCGv_i64 gen_mte_check1(DisasContext *s, TCGv_i64 addr, ...)
{
    // 如果 MTE 激活，在加载/存储前插入标签检查
    gen_helper_mte_check(ret, tcg_env, desc, addr);
    return ret;  // 返回清理后的地址
}
```

### 25. MTE 运行时检查

```c
// mte_helper.c:882-902
uint64_t HELPER(mte_check)(CPUARMState *env, uint32_t desc, uint64_t ptr)
{
    // 1. 先检查对齐（优先级高于标签故障）
    unsigned align = FIELD_EX32(desc, MTEDESC, ALIGN);
    if (unlikely(ptr & align_mask)) {
        arm_cpu_do_unaligned_access(...);  // 对齐异常
    }

    // 2. 标签检查
    return mte_check(env, desc, ptr, GETPC());
}
```

### 26. MTE 标签故障处理

```c
// mte_helper.c:601
void mte_check_fail(CPUARMState *env, uint32_t desc, uint64_t fault, ...)
{
    // 根据 SCTLR.TCF 设置:
    // 00: 无操作（标签检查禁用）
    // 01: 同步异常
    // 10: 异步累积（设置TFSR位）
    // 11: 未定义
}
```

### 27. DC ZVA 的 MTE 检查

```c
// mte_helper.c:922
uint64_t HELPER(mte_check_zva)(CPUARMState *env, uint32_t desc, uint64_t ptr)
// DC ZVA 有特殊的标签检查路径：
// 必须检查整个零化块的所有标签粒度
```

---

## 第八部分：PAC（指针认证）

### 28. PAC 密钥与算法

```c
// pauth_helper.c:490-538 — PAC 签名函数
uint64_t HELPER(pacia)(CPUARMState *env, uint64_t x, uint64_t y)
{
    if (!pauth_key_enabled(env, el, SCTLR_EnIA)) return x;  // 未启用
    pauth_check_trap(env, el, GETPC());                       // 陷阱检查
    return pauth_addpac(env, x, y, &env->keys.apia, false);  // 签名
}

// 类似: HELPER(pacib) 用 apib 密钥
//        HELPER(pacda) 用 apda 密钥，data=true
//        HELPER(pacdb) 用 apdb 密钥，data=true
```

### 29. PAC 密钥类型

| 密钥 | 用途 | SCTLR 使能位 |
|------|------|-------------|
| APIAKey | 指令地址认证 A | SCTLR.EnIA |
| APIBKey | 指令地址认证 B | SCTLR.EnIB |
| APDAKey | 数据地址认证 A | SCTLR.EnDA |
| APDBKey | 数据地址认证 B | SCTLR.EnDB |
| APGAKey | 通用认证 | 始终可用 |

### 30. PACGA（通用认证）

```c
// pauth_helper.c:530-538
uint64_t HELPER(pacga)(CPUARMState *env, uint64_t x, uint64_t y)
{
    uint64_t pac = pauth_computepac(env, x, y, env->keys.apga);
    return pac & 0xffffffff00000000ull;  // 只返回高32位
}
```

### 31. AUT* 认证验证

认证失败时（签名不匹配）：
- 在指针高位插入错误码
- 后续使用该指针会触发地址异常
- 不直接抛出异常（延迟检测）

---

## 第九部分：BTI（分支目标标识）

### 32. BTI 检查入口

```c
// translate-a64.c:10832-10849
// 在 AArch64 翻译初始化时检查 BTI
// s->btype: 来自 PSTATE.BTYPE（上一次分支设置的类型）
// s->bt: 来自 SCTLR_ELx.BT
```

### 33. btype_destination_ok()

```c
// translate-a64.c:10602-10651
static bool btype_destination_ok(uint32_t insn, bool bt, int btype)
{
    // 保护页上的合法分支目标:
    if ((insn & 0xfffff01fu) == 0xd503201fu) {
        switch (extract32(insn, 5, 7)) {
        case 0b011001: /* PACIASP */
        case 0b011011: /* PACIBSP */
            return !bt || btype != 3;  // BT=1时不兼容btype=3
        case 0b100000: /* BTI */
            return false;              // 纯BTI不兼容任何btype
        case 0b100010: /* BTI c */
            return btype != 3;         // 不兼容btype=3(间接跳转)
        case 0b100100: /* BTI j */
            return btype != 2;         // 不兼容btype=2(间接调用)
        case 0b100110: /* BTI jc */
            return true;               // 兼容所有btype
        }
    } else {
        // BRK/HLT: 优先处理断点
        switch (insn & 0xffe0001fu) {
        case 0xd4200000u: /* BRK */
        case 0xd4400000u: /* HLT */
            return true;
        }
    }
    return false;
}
```

### 34. BTI 兼容性矩阵

| 目标指令 | btype=1(CALL) | btype=2(JMP) | btype=3(RET等) |
|---------|:---:|:---:|:---:|
| BTI | ✗ | ✗ | ✗ |
| BTI c | ✓ | ✓ | ✗ |
| BTI j | ✓ | ✗ | ✓ |
| BTI jc | ✓ | ✓ | ✓ |
| PACIASP/PACIBSP | ✓ | ✓ | BT=0:✓ BT=1:✗ |

### 35. PSTATE.BTYPE 设置

| 分支类型 | BTYPE 值 |
|---------|---------|
| BL/BLR（间接调用）| 1 |
| BR（间接跳转）| 2 |
| BLR/BR（寄存器间接）| 3 |
| B/CBZ/TBZ（直接分支）| 0（不设置）|

---

## 第十部分：MTTCG 内存模型

### 36. 单线程 vs 多线程 TCG

| 特性 | 单线程 TCG | MTTCG |
|------|-----------|-------|
| tcg_gen_mb() | NOP（不生成代码）| 生成主机屏障 |
| 独占访问 | 本地检查（无竞争）| 原子 CAS 操作 |
| LSE 原子操作 | 可简化为普通读写 | 真正的原子操作 |
| Device 内存 | 无特殊处理 | 需要序列化 |
| 数据竞争 | 不存在 | 可能存在 |

### 37. MTTCG 屏障映射到主机

TCG 后端将 `INDEX_op_mb` 映射到主机架构的屏障指令：
- **x86**：`mfence`（完全屏障）、`lock addl`
- **AArch64**：`dmb ish`、`dmb ishld`、`dmb ishst`
- **RISC-V**：`fence`

---

## 第十一部分：页表遍历与地址翻译

### 38. 顶层翻译流程

```
虚拟地址
  │
  ├── arm_mmu_idx_el() [helper.c:9957]
  │   └── 确定MMU Regime (EL1&0 / EL2 / EL3)
  │
  ├── get_phys_addr() [ptw.c]
  │   ├── Stage 1 翻译
  │   │   ├── 页表遍历（4级/3级）
  │   │   ├── 权限检查（AP/UXN/PXN）
  │   │   └── 提取内存属性（AttrIndx → MAIR）
  │   │
  │   ├── Stage 2 翻译（虚拟化时）
  │   │   ├── IPA → PA
  │   │   ├── S2权限检查
  │   │   └── S2内存属性
  │   │
  │   └── combine_cacheattrs() [ptw.c:3413]
  │       └── 合并S1/S2属性 → 最终属性
  │
  └── 物理地址 + 属性
```

### 39. MMU 禁用路径

```c
// ptw.c:3470-3551
// SCTLR.M = 0: 直接VA=PA
// 数据访问: Device-nGnRnE
// 指令访问: Normal
```

---

## 第十二部分：监视点内存访问

### 40. CPU 监视点检查

```c
// accel/tcg/watchpoint.c:54-128
cpu_watchpoint_address_matches()  // 地址匹配检查
cpu_check_watchpoint()            // 完整监视点检查
tb_check_watchpoint()             // TB级别检查
```

监视点与内存屏障独立运作：
- 屏障影响访问顺序，不影响监视点触发
- 监视点在 TLB 查找路径中检查
- 独占访问同样触发监视点

---

## 附录

### A. 内存序关键源文件

| 文件 | 行数 | 内容 |
|------|------|------|
| `target/arm/tcg/translate-a64.c` | ~10800 | 屏障/独占/原子/MTE/BTI 翻译 |
| `include/tcg/tcg-mo.h` | ~50 | TCG 内存序位定义 |
| `tcg/tcg-op.c` | ~3000 | tcg_gen_mb() 实现 |
| `target/arm/tcg/mte_helper.c` | ~1020 | MTE 标签检查实现 |
| `target/arm/tcg/pauth_helper.c` | ~620 | PAC 签名/验证 |
| `target/arm/ptw.c` | ~3600 | 页表遍历、内存属性 |
| `target/arm/helper.c` | ~10200 | MMU 索引、TLB |
| `accel/tcg/watchpoint.c` | ~130 | 监视点检查 |

### B. ARM 内存模型关键概念映射

| ARM 概念 | QEMU 实现 |
|----------|----------|
| DMB/DSB | `tcg_gen_mb()` + TCG_BAR_SC |
| ISB | TB 结束（`gen_goto_tb`）|
| LDAR/STLR | 加载/存储 + `tcg_gen_mb()` LDAQ/STRL |
| LDXR/STXR | `exclusive_addr/val` + `atomic_cmpxchg` |
| CAS/SWP | `tcg_gen_atomic_*` 系列 |
| MTE | `gen_helper_mte_check` + tag memory |
| PAC | `pauth_addpac()` / `pauth_auth()` |
| BTI | `btype_destination_ok()` 在 TB 开头检查 |
| Device memory | `S1_attrs_are_device()` 在 TLB 填充时识别 |
