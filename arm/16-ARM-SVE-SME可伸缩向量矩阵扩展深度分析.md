# ARM SVE/SME 可伸缩向量与矩阵扩展深度分析

## 文档信息
- **QEMU 版本**: 11.0.50
- **分析目标**: SVE (Scalable Vector Extension) 和 SME (Scalable Matrix Extension) 在 QEMU 中的完整实现
- **核心源文件**:
  - `target/arm/cpu.h` — SVE/SME 状态定义
  - `target/arm/helper.c` — ZCR/SMCR 寄存器、VL 计算、陷阱控制
  - `target/arm/tcg/translate-a64.c` — SVE/SME 访问检查
  - `target/arm/tcg/translate-sve.c` — SVE 指令翻译
  - `target/arm/tcg/sve_helper.c` — SVE 辅助函数实现
  - `target/arm/cpu64.c` — SVE VL finalize 与 CPU 属性

---

## 第一部分：SVE 架构概述

### 1. SVE 核心概念

SVE (Scalable Vector Extension) 是 ARMv8.2 引入的可伸缩向量扩展：

- **可变向量长度 (VL)**：128 到 2048 位，128 位递增
- **VQ (Vector Quadwords)**：VL / 128，即 128 位单元数（1-16）
- **32 个 Z 寄存器**：每个 VL 位宽
- **16 个 P 谓词寄存器** + 1 个 FFR (First Fault Register)：每个 VL/8 位
- **实现定义 VL**：同一 ISA 代码可在不同 VL 硬件上运行

### 2. SME 核心概念

SME (Scalable Matrix Extension) 是 ARMv9.2 引入的矩阵扩展：

- **ZA 矩阵存储**：SVL × SVL 字节方阵
- **Streaming SVE 模式 (PSTATE.SM)**：使用 SVL 而非 VL 的特殊执行模式
- **PSTATE.ZA**：控制 ZA 存储是否可访问
- **SVL (Streaming Vector Length)**：Streaming 模式的向量长度，可与 SVE VL 不同

---

## 第二部分：CPUARMState 中的 SVE/SME 状态

### 3. VFP/SVE 寄存器存储

```c
// cpu.h:674-702 — vfp 结构体包含 SVE 状态
struct {
    ARMVectorReg zregs[32];      // 32个Z寄存器，每个最大2048位

    /* FFR 存储为 pregs[16] 方便统一处理 */
#define FFR_PRED_NUM 16
    ARMPredicateReg pregs[17];   // 16个P寄存器 + 1个FFR
    ARMPredicateReg preg_tmp;    // 临时谓词暂存

    // ... 浮点状态 ...

    uint64_t zcr_el[4];          // ZCR_EL[1-3]（SVE VL控制）
    uint64_t smcr_el[4];         // SMCR_EL[1-3]（SME SVL控制）
} vfp;
```

**关键点**：

- `ARMVectorReg` (cpu.h:170-172)：固定分配 ARM_MAX_VQ × 16 字节（最大 256 字节 = 2048 位）
- 无论实际 VL 多大，每个 Z 寄存器始终占用最大空间
- FFR 不是独立存储，而是复用 `pregs[16]`

### 4. SVCR 寄存器（SM/ZA 状态位）

```c
// cpu.h:280-315
uint64_t svcr;  // PSTATE.{SM,ZA} 在SVCR对应位上
```

SVCR 包含两个关键位：
- **SM (bit 0)**：Streaming SVE 模式使能
- **ZA (bit 1)**：ZA 矩阵存储使能

### 5. ZA 矩阵存储

```c
// cpu.h:725-754 — za_state 结构体
struct {
    uint64_t zt0[512 / 64] QEMU_ALIGNED(16);  // SME2 ZT0（512位）

    /* ZA 存储 — 256×256 字节数组
     * ZA[N] 在 za[N] 的最低有效字节中
     * SVL 为 X 时只能访问底部 X 行×X 列
     */
    ARMVectorReg za[ARM_MAX_VQ * 16];          // 最大16×16=256行
} za_state;
```

**ZA 存储布局**：
- ZA 是 SVL×SVL 字节方阵
- 元素大小为 esz 的 Tile T 的第 N 行在 `ZA[T + N * esz]`
- Tile 不连续存储 — 行在 ZA 数组中交错

### 6. 独占监视器状态

```c
// cpu.h:704-713
uint64_t exclusive_addr;   // 独占加载的地址
uint64_t exclusive_val;    // 独占加载的值
uint64_t exclusive_high;   // LDXP 高64位（来自高地址）
```

### 7. PAC 密钥存储

```c
// cpu.h:715-721
struct {
    ARMPACKey apia;   // 指令地址认证 A
    ARMPACKey apib;   // 指令地址认证 B
    ARMPACKey apda;   // 数据地址认证 A
    ARMPACKey apdb;   // 数据地址认证 B
    ARMPACKey apga;   // 通用认证
} keys;
```

---

## 第三部分：SVE 向量长度管理

### 8. ZCR_ELx 寄存器定义

```c
// helper.c:4759-4777 — zcr_reginfo 数组
static const ARMCPRegInfo zcr_reginfo[] = {
    { .name = "ZCR_EL1", .opc0 = 3, .opc1 = 0, .crn = 1, .crm = 2, .opc2 = 0,
      .access = PL1_RW, .type = ARM_CP_SVE,
      .fieldoffset = offsetof(CPUARMState, vfp.zcr_el[1]),
      .writefn = zcr_write },
    { .name = "ZCR_EL2", .opc0 = 3, .opc1 = 4, ... },
    { .name = "ZCR_EL3", .opc0 = 3, .opc1 = 6, ... },
};
```

每个 ZCR_ELx 的低 4 位（LEN 字段）设置该 EL 允许的最大 VQ-1。

### 9. 有效 VL 计算：sve_vqm1_for_el_sm()

```c
// helper.c:4695-4731 — 核心VL计算
uint32_t sve_vqm1_for_el_sm(CPUARMState *env, int el, bool sm)
{
    ARMCPU *cpu = env_archcpu(env);
    uint64_t *cr = env->vfp.zcr_el;     // SVE: 用zcr
    uint32_t map = cpu->sve_vq.map;      // SVE: 硬件支持的VL位图
    uint32_t len = ARM_MAX_VQ - 1;       // 从最大值开始

    if (sm) {
        cr = env->vfp.smcr_el;           // Streaming: 用smcr
        map = cpu->sme_vq.map;
    }

    // 逐级取最小值（嵌套限制）
    if (el <= 1 && !el_is_in_host(env, el))
        len = MIN(len, 0xf & (uint32_t)cr[1]);  // ZCR_EL1
    if (el <= 2 && arm_is_el2_enabled(env))
        len = MIN(len, 0xf & (uint32_t)cr[2]);  // ZCR_EL2
    if (arm_feature(env, ARM_FEATURE_EL3))
        len = MIN(len, 0xf & (uint32_t)cr[3]);  // ZCR_EL3

    // 从支持位图中选择不超过len的最大VQ
    map &= MAKE_64BIT_MASK(0, len + 1);
    if (map != 0)
        return 31 - clz32(map);    // 最高置位位

    // SME-only CPU 不在Streaming模式：VL=128
    assert(sm);
    return ctz32(cpu->sme_vq.map);
}
```

**VL 计算逻辑**：
1. 取 ZCR_EL1/EL2/EL3 的 LEN 字段最小值
2. 用硬件支持位图 `sve_vq.map` 掩码
3. 选择不超过限制的最大有效 VQ
4. 高 EL 的 ZCR 总能限制低 EL 的 VL

### 10. ZCR 写入处理：zcr_write()

```c
// helper.c:4738-4757
static void zcr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    int old_len = sve_vqm1_for_el(env, cur_el);

    raw_write(env, ri, value & 0xf);  // 只有低4位有效

    new_len = sve_vqm1_for_el(env, cur_el);
    if (new_len < old_len) {
        aarch64_sve_narrow_vq(env, new_len + 1);  // VL缩小→清除高位
    }
}
```

**关键**：当 VL 缩小时，必须清除不可访问的寄存器高位（安全要求）。

### 11. CPU 属性配置：arm_cpu_sve_finalize()

```c
// cpu64.c:63-120 — SVE VL 验证与配置
void arm_cpu_sve_finalize(ARMCPU *cpu, Error **errp)
{
    uint32_t vq_map = cpu->sve_vq.map;       // 显式启用的VL
    uint32_t vq_init = cpu->sve_vq.init;     // 已初始化的VL
    uint32_t vq_supported = cpu->sve_vq.supported; // 硬件支持的VL
    // ...
}
```

**用户配置方式**：
- `sve-max-vq=N`：设置最大 VQ（隐式启用所有 ≤N 的 VL）
- `sve128=on/off`、`sve256=on/off` ... `sve2048=on/off`：逐个控制
- 所有小于最大已启用的 2 次幂 VL 会自动启用
- KVM 模式下自动启用所有支持的未初始化 VL

---

## 第四部分：SVE 陷阱与访问控制

### 12. sve_exception_el()：SVE 陷阱判断

```c
// helper.c:4598-4641
int sve_exception_el(CPUARMState *env, int el)
{
    // 第一层：CPACR_EL1.ZEN（EL0/EL1 控制）
    if (el <= 1 && !el_is_in_host(env, el)) {
        switch (FIELD_EX64(env->cp15.cpacr_el1, CPACR_EL1, ZEN)) {
        case 0: case 2: return 1;  // 陷入EL1
        case 1: if (el == 0) return 1; break;  // 仅EL0陷入
        case 3: break;  // 不陷入
        }
    }

    // 第二层：CPTR_EL2（EL2 控制）
    if (el <= 2 && arm_is_el2_enabled(env)) {
        if (E2H) {
            // VHE格式：CPTR_EL2.ZEN
            switch (CPTR_EL2.ZEN) { ... }
        } else {
            // 非VHE：CPTR_EL2.TZ
            if (CPTR_EL2.TZ) return 2;
        }
    }

    // 第三层：CPTR_EL3.EZ（EL3 控制，负逻辑）
    if (arm_feature(env, ARM_FEATURE_EL3) && !CPTR_EL3.EZ)
        return 3;

    return 0;  // 不陷入
}
```

### 13. sme_exception_el()：SME 陷阱判断

```c
// helper.c:4647-4690 — 结构与 sve_exception_el() 对称
int sme_exception_el(CPUARMState *env, int el)
{
    // CPACR_EL1.SMEN → 陷入EL1
    // CPTR_EL2.SMEN/TSM → 陷入EL2
    // CPTR_EL3.ESM → 陷入EL3（负逻辑）
}
```

### 14. 翻译阶段访问检查：sve_access_check()

```c
// translate-a64.c:1508-1538
bool sve_access_check(DisasContext *s)
{
    if (dc_isar_feature(aa64_sme, s)) {
        if (s->pstate_sm) {
            ret = sme_enabled_check(s);      // Streaming模式→检查SME
        } else if (dc_isar_feature(aa64_sve, s)) {
            goto continue_sve;               // 普通SVE路径
        } else {
            ret = sme_sm_enabled_check(s);   // SME-only CPU
        }
        if (ret) ret = nonstreaming_check(s); // 非Streaming指令检查
        return ret;
    }

continue_sve:
    if (s->sve_excp_el) {
        gen_exception_insn_el(s, 0, EXCP_UDEF,
                              syn_sve_access_trap(), s->sve_excp_el);
        return false;                        // 生成陷阱
    }
    return fp_access_check(s);               // 还需FP访问检查
}
```

**访问检查层次**：
1. SME CPU 在 Streaming 模式：走 SME 检查路径
2. 普通 SVE：检查 `sve_excp_el`（已在翻译上下文初始化时计算）
3. 最后还要通过 FP 访问检查（CPACR.FPEN）

---

## 第五部分：SME 实现

### 15. SMCR_ELx 寄存器

```c
// helper.c:4868-4894 — smcr_write()
static void smcr_write(CPUARMState *env, const ARMCPRegInfo *ri, uint64_t value)
{
    uint64_t valid_mask = R_SMCR_LEN_MASK | R_SMCR_FA64_MASK;

    if (cpu_isar_feature(aa64_sme2, env_archcpu(env)))
        valid_mask |= R_SMCR_EZT0_MASK;  // SME2: EZT0位

    raw_write(env, ri, value & valid_mask);

    // SVL缩小时清除高位（与ZCR相同逻辑）
    if (new_len < old_len)
        aarch64_sve_narrow_vq(env, new_len + 1);
}
```

SMCR 额外字段：
- **FA64**：允许 Streaming 模式下执行非 Streaming SVE 指令
- **EZT0** (SME2)：启用 ZT0 寄存器访问

### 16. Streaming SVE 模式切换：aarch64_set_svcr()

```c
// helper.c:4832-4860
void aarch64_set_svcr(CPUARMState *env, uint64_t new, uint64_t mask)
{
    uint64_t change = (env->svcr ^ new) & mask;
    if (change == 0) return;

    env->svcr ^= change;

    // SM 切换 → ResetSVEState（清除所有Z/P/FFR）
    if (change & R_SVCR_SM_MASK) {
        arm_reset_sve_state(env);
    }

    // ZA 使能 → 清零ZA存储
    // 禁用时不清零（存储不可访问，值无关）
    if (change & new & R_SVCR_ZA_MASK) {
        memset(&env->za_state, 0, sizeof(env->za_state));
    }

    arm_rebuild_hflags(env);  // 重建翻译标志
}
```

**SM 切换的关键影响**：
1. 所有 Z 寄存器清零
2. 所有 P 寄存器（含 FFR）清零
3. FPSR 重置为 0x0800009f
4. 不影响 ZA 存储

### 17. arm_reset_sve_state()

```c
// helper.c:4823-4830
static void arm_reset_sve_state(CPUARMState *env)
{
    memset(env->vfp.zregs, 0, sizeof(env->vfp.zregs));  // 清除32个Z寄存器
    memset(env->vfp.pregs, 0, sizeof(env->vfp.pregs));   // 清除P0-P15+FFR
    vfp_set_fpsr(env, 0x0800009f);                        // 重置FPSR
}
```

---

## 第六部分：SVE/SME 与 EL 切换

### 18. aarch64_sve_change_el()：异常 EL 变化处理

```c
// helper.c:10073-10128
void aarch64_sve_change_el(CPUARMState *env, int old_el, int new_el, bool el0_a64)
{
    // 无SVE/SME则直接返回
    if (!aa64_sve && !aa64_sme) return;

    // FP在任一EL被禁用则跳过
    if (fp_exception_el(env, old_el) || fp_exception_el(env, new_el)) return;

    old_a64 = old_el ? arm_el_is_aa64(env, old_el) : el0_a64;
    new_a64 = new_el ? arm_el_is_aa64(env, new_el) : el0_a64;

    // AArch64↔AArch32 切换且 SM=1 → 完全重置SVE
    sm = FIELD_EX64(env->svcr, SVCR, SM);
    if (old_a64 != new_a64 && sm) {
        arm_reset_sve_state(env);
        return;
    }

    // 计算新旧EL的有效VL
    old_len = old_a64 ? sve_vqm1_for_el_sm_ena(env, old_el, sm) : 0;
    new_len = new_a64 ? sve_vqm1_for_el_sm_ena(env, new_el, sm) : 0;

    // VL缩小 → 清除不可访问的高位
    if (new_len < old_len)
        aarch64_sve_narrow_vq(env, new_len + 1);
}
```

**EL 切换场景**：

| 场景 | 行为 |
|------|------|
| EL0(aa64,VQ=4) → EL1(aa64,VQ=2) | narrow_vq 清除 Q2-Q3 |
| EL1(aa64,SM=1) → EL0(aa32) | 完全 reset SVE state |
| EL2(aa64,VQ=4) → EL0(aa32) → EL1(aa64,VQ=1) | 进入 aa32 时 narrow 为 0，回 EL1 保持 |

### 19. aarch64_sve_narrow_vq()

当新 VL < 旧 VL 时调用：
- 清除每个 Z 寄存器中超出新 VL 的字节
- 清除每个 P 寄存器中超出新 VL/8 的字节
- 确保不会泄露高 EL 的数据到低 EL

---

## 第七部分：SVE 指令翻译与辅助函数

### 20. translate-sve.c：SVE 指令翻译

`target/arm/tcg/translate-sve.c` 是 SVE 指令解码/翻译的核心文件：

- 每条 SVE 指令对应一个 `trans_*` 函数
- 开头调用 `sve_access_check(s)` 检查访问权限
- 使用 `gen_helper_sve_*` 调用辅助函数
- VL 依赖操作通过运行时 `tcg_env` 中的 VL 动态处理

### 21. sve_helper.c：SVE 辅助函数

```c
// sve_helper.c:45-117 — 谓词测试
void HELPER(sve_predtest1)(void *vd, void *vg, uint32_t words)
void HELPER(sve_predtest)(void *vd, void *vg, uint32_t words)
```

关键类别：
- **谓词逻辑运算**：`sve_and_pppp`、`sve_orr_pppp` 等
- **向量算术**：`sve_add_zpzz`、`sve_mul_zpzz` 等
- **加载/存储**：连续/聚集/散射访问
- **缩减操作**：`sve_addv`、`sve_sminv` 等

---

## 第八部分：KVM SVE/SME 支持

### 22. KVM SVE VL 发现

```c
// kvm.c:247-309（概要）
// 使用 KVM_ARM_VCPU_SVE 特性启用 SVE
// 通过 KVM_REG_ARM64_SVE_VLS 读取主机支持的 VL 列表
```

### 23. KVM SVE 寄存器保存/恢复

```c
// kvm.c:2131-2148, 2314-2332
// 使用 KVM_REG_ARM64_SVE_ZREG(n, i) 保存/恢复 Z 寄存器
// 使用 KVM_REG_ARM64_SVE_PREG(n) 保存/恢复 P 寄存器
// 使用 KVM_REG_ARM64_SVE_FFR(i) 保存/恢复 FFR
```

---

## 第九部分：SVE/SME 陷阱控制总表

### 24. SVE 陷阱控制层次

| 控制位 | 位置 | 控制范围 | 值含义 |
|--------|------|----------|--------|
| CPACR_EL1.ZEN | CPACR_EL1[17:16] | EL0/EL1 | 00/10=陷入EL1, 01=仅EL0陷入, 11=不陷 |
| CPTR_EL2.TZ | CPTR_EL2[8] | ≤EL2 (非VHE) | 1=陷入EL2 |
| CPTR_EL2.ZEN | CPTR_EL2[17:16] | ≤EL2 (VHE) | 同CPACR格式 |
| CPTR_EL3.EZ | CPTR_EL3[8] | 全局 | 0=陷入EL3（负逻辑）|

### 25. SME 陷阱控制层次

| 控制位 | 位置 | 控制范围 | 值含义 |
|--------|------|----------|--------|
| CPACR_EL1.SMEN | CPACR_EL1[25:24] | EL0/EL1 | 同ZEN格式 |
| CPTR_EL2.TSM | CPTR_EL2[12] | ≤EL2 (非VHE) | 1=陷入EL2 |
| CPTR_EL2.SMEN | CPTR_EL2[25:24] | ≤EL2 (VHE) | 同CPACR格式 |
| CPTR_EL3.ESM | CPTR_EL3[12] | 全局 | 0=陷入EL3（负逻辑）|

---

## 第十部分：SVE VL 配置与验证流程

### 26. 完整 VL 生命周期

```
用户配置 (-cpu max,sve-max-vq=4,sve128=on,sve256=on,sve512=on)
    │
    ▼
arm_cpu_sve_finalize()  [cpu64.c:63]
    │ ├── 验证 sve<N> 属性与 sve-max-vq 一致性
    │ ├── 自动启用2次幂VL
    │ └── 生成 sve_vq.map 位图
    │
    ▼
运行时 VL 计算
    │
sve_vqm1_for_el_sm()  [helper.c:4695]
    │ ├── 读取 ZCR_EL1/2/3.LEN
    │ ├── 取最小值
    │ └── 与 sve_vq.map 位图匹配最大有效VQ
    │
    ▼
EL 切换 VL 调整
    │
aarch64_sve_change_el()  [helper.c:10073]
    │ ├── 检查 AArch64↔AArch32 + SM → reset
    │ ├── 比较新旧 EL 的有效 VL
    │ └── VL 缩小 → narrow_vq
    │
    ▼
SVE 指令执行
    │
sve_access_check()  [translate-a64.c:1508]
    ├── 检查陷阱条件
    └── 通过后执行向量操作
```

---

## 第十一部分：SME ZA 存储模型

### 27. ZA 数组布局

```
ZA[0]   = [ byte0, byte1, byte2, ..., byte(SVL-1) ]
ZA[1]   = [ byte0, byte1, byte2, ..., byte(SVL-1) ]
...
ZA[SVL-1] = [ byte0, byte1, byte2, ..., byte(SVL-1) ]
```

### 28. Tile 映射

对于元素大小 esz 字节的 Tile T：
- Tile T 的第 N 行（水平切片）= `ZA[T + N * esz]`
- 例如 32 位 (esz=4) Tile 0：行 0=ZA[0], 行 1=ZA[4], 行 2=ZA[8]...
- 例如 32 位 Tile 1：行 0=ZA[1], 行 1=ZA[5], 行 2=ZA[9]...

### 29. ZA 访问控制

- PSTATE.ZA = 0：ZA 不可访问
- PSTATE.ZA = 1：ZA 可访问
- ZA 使能时清零（`aarch64_set_svcr` 中 `memset`）
- ZA 禁用时不清零（节省性能，反正不可访问）

---

## 第十二部分：SME Streaming 模式交互

### 30. Streaming 模式下的指令行为

| 指令类型 | 普通模式 (SM=0) | Streaming 模式 (SM=1) |
|----------|-----------------|----------------------|
| 普通 SVE | 使用 VL | SMCR.FA64=1 时可用，使用 SVL |
| Streaming-only SVE | 不可用 | 使用 SVL |
| NEON/FP | 正常执行 | 正常执行 |
| SME 外积指令 | 不可用 | 使用 SVL |

### 31. SM 切换时的寄存器清除

```
SMSTART SM (SM: 0→1):
  ├── arm_reset_sve_state()
  │   ├── zregs[0..31] = 0
  │   ├── pregs[0..16] = 0  (含FFR)
  │   └── FPSR = 0x0800009f
  └── arm_rebuild_hflags()

SMSTOP SM (SM: 1→0):
  └── 同上（SM 切换总是清除）

SMSTART ZA (ZA: 0→1):
  └── za_state = 0  (清零ZA+ZT0)

SMSTOP ZA (ZA: 1→0):
  └── 不清零（优化）
```

---

## 第十三部分：完整 SVE/SME 特性关系图

### 32. 特性依赖关系

```
SVE (ARMv8.2)
  ├── SVE2 (ARMv9.0)
  │   ├── SVE2-AES
  │   ├── SVE2-SM4
  │   ├── SVE2-SHA3
  │   └── SVE2-BITPERM
  └── ...

SME (ARMv9.2)
  ├── 依赖 SVE2（SME 需要 SVE2 基础设施）
  ├── PSTATE.SM / PSTATE.ZA
  ├── ZA 矩阵存储
  ├── SME2 (ARMv9.4)
  │   ├── ZT0 寄存器（512位查找表）
  │   └── 多向量指令
  └── FA64（全SVE指令在Streaming模式可用）
```

### 33. QEMU 中的 SVE/SME 限制

1. **TCG 模式**：完整 SVE/SME 软件模拟
2. **KVM 模式**：SVE 直通主机（需主机支持），SME KVM 支持较新
3. **VL 必须是 128 位倍数**：1-16 个 128 位 quadword
4. **迁移**：需要源/目标支持相同 VL 集合
5. **性能**：TCG SVE 向量操作比标量慢（辅助函数调用开销）

---

## 附录

### A. SVE/SME 相关关键源文件

| 文件 | 行数 | 内容 |
|------|------|------|
| `target/arm/cpu.h` | ~3500 | SVE/SME 状态定义、ARMVectorReg |
| `target/arm/helper.c` | ~10200 | ZCR/SMCR、VL 计算、陷阱控制、EL 切换处理 |
| `target/arm/tcg/translate-a64.c` | ~10000 | sve/sme_access_check、系统寄存器翻译 |
| `target/arm/tcg/translate-sve.c` | ~8000 | SVE 指令完整翻译 |
| `target/arm/tcg/sve_helper.c` | ~7000 | SVE 辅助函数（向量/谓词/加载存储） |
| `target/arm/cpu64.c` | ~1000 | arm_cpu_sve_finalize、VL 属性配置 |
| `target/arm/kvm.c` | ~2500 | KVM SVE 直通、VL 发现、寄存器保存恢复 |

### B. 关键数据结构大小

| 结构 | 最大大小 | 说明 |
|------|---------|------|
| ARMVectorReg | 256 字节 | 2048 位向量 |
| ARMPredicateReg | 32 字节 | 2048/8 位谓词 |
| vfp.zregs[32] | 8192 字节 | 32 个 Z 寄存器 |
| vfp.pregs[17] | 544 字节 | P0-P15 + FFR |
| za_state.za[] | 1048576 字节 | 256×256 字节 ZA (4096 × ARMVectorReg) |
| za_state.zt0 | 64 字节 | SME2 ZT0 (512 位) |
