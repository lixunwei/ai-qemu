# ARM64 Crypto 扩展指令翻译深度分析

> 文档编号：87  
> 分析目标：AES/SHA/SM3/SM4 密码学扩展的解码、翻译和 helper 实现  
> 源码版本：QEMU 11.0.50  
> 核心文件：target/arm/tcg/crypto_helper.c (679行)、translate-a64.c (crypto节)、a64.decode (crypto节)

---

## 一、概述

ARM64 密码学扩展（Cryptographic Extension）提供硬件加速的对称/哈希算法原语。QEMU 通过 TCG helper 函数在软件层面精确模拟这些指令的行为。

### 1.1 特性分组

| FEAT ID | ID_AA64ISAR0 字段 | 指令集 | 说明 |
|---------|-------------------|--------|------|
| FEAT_AES | AES ≠ 0 | AESE, AESD, AESMC, AESIMC | AES 加密/解密 |
| FEAT_PMULL | AES > 1 | PMULL/PMULL2 | 多项式乘法（GF(2^128)） |
| FEAT_SHA1 | SHA1 ≠ 0 | SHA1C, SHA1P, SHA1M, SHA1H, SHA1SU0/SU1 | SHA-1 哈希 |
| FEAT_SHA256 | SHA2 ≠ 0 | SHA256H, SHA256H2, SHA256SU0/SU1 | SHA-256 哈希 |
| FEAT_SHA512 | SHA2 > 1 | SHA512H, SHA512H2, SHA512SU0/SU1 | SHA-512 哈希 |
| FEAT_SHA3 | SHA3 ≠ 0 | EOR3, RAX1, XAR, BCAX | SHA-3/Keccak 辅助 |
| FEAT_SM3 | SM3 ≠ 0 | SM3SS1, SM3TT1A/B, SM3TT2A/B, SM3PARTW1/W2 | 国密 SM3 哈希 |
| FEAT_SM4 | SM4 ≠ 0 | SM4E, SM4EKEY | 国密 SM4 加密 |
| SVE2-AES | — | SVE AESE/AESD/AESMC/AESIMC | SVE2 向量化 AES |
| SVE2-SM4 | — | SVE SM4E/SM4EKEY | SVE2 向量化 SM4 |

### 1.2 特性检测

```c
// target/arm/cpu-features.h
static inline bool isar_feature_aa64_aes(const ARMISARegisters *id) {
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, AES) != 0;
}
static inline bool isar_feature_aa64_sha256(const ARMISARegisters *id) {
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, SHA2) != 0;
}
static inline bool isar_feature_aa64_sm3(const ARMISARegisters *id) {
    return FIELD_EX64_IDREG(id, ID_AA64ISAR0, SM3) != 0;
}
// ... 类似模式
```

---

## 二、解码层

### 2.1 a64.decode 中的密码学指令

```
# AES 指令（2-register crypto）
AESE            01001110 00 10100 00100 10 ..... .....  @r2r_q1e0
AESD            01001110 00 10100 00101 10 ..... .....  @r2r_q1e0
AESMC           01001110 00 10100 00110 10 ..... .....  @rr_q1e0
AESIMC          01001110 00 10100 00111 10 ..... .....  @rr_q1e0

# SHA-1 指令（3-register crypto）
SHA1C           0101 1110 000 ..... 000000 ..... .....  @rrr_q1e0
SHA1P           0101 1110 000 ..... 000100 ..... .....  @rrr_q1e0
SHA1M           0101 1110 000 ..... 001000 ..... .....  @rrr_q1e0
SHA1SU0         0101 1110 000 ..... 001100 ..... .....  @rrr_q1e0

# SHA-256 指令
SHA256H         0101 1110 000 ..... 010000 ..... .....  @rrr_q1e0
SHA256H2        0101 1110 000 ..... 010100 ..... .....  @rrr_q1e0
SHA256SU1       0101 1110 000 ..... 011000 ..... .....  @rrr_q1e0

# SM3/SM4 指令
SM3PARTW1       1100 1110 011 ..... 110000 ..... .....  @rrr_q1e0
SM3PARTW2       1100 1110 011 ..... 110001 ..... .....  @rrr_q1e0
SM4EKEY         1100 1110 011 ..... 110010 ..... .....  @rrr_q1e0
SM4E            1100 1110 110 00000 100001 ..... .....  @r2r_q1e0
SM3SS1          1100 1110 010 ..... 0 ..... ..... ..... @rrrr_q1e3

# SHA-3 辅助指令
EOR3            1100 1110 000 ..... 0 ..... ..... ..... @rrrr_q1e3
BCAX            1100 1110 001 ..... 0 ..... ..... ..... @rrrr_q1e3
RAX1            1100 1110 011 ..... 100011 ..... .....  @rrr_q1e3
XAR             1100 1110 100 rm:5 imm:6 rn:5 rd:5
```

### 2.2 解码格式说明

- `@r2r_q1e0`: 2 寄存器操作，Q=1（128-bit），元素大小=0
- `@rrr_q1e0`: 3 寄存器操作，Q=1
- `@rrrr_q1e3`: 4 寄存器操作（SM3SS1/EOR3/BCAX）
- `@crypto3i`: 3 寄存器 + 2-bit 立即数（SM3TT 系列）
- 所有密码学指令固定操作 128-bit（Q=1）

---

## 三、翻译层

### 3.1 统一翻译模式 — TRANS_FEAT

```c
// target/arm/tcg/translate-a64.c:5457
TRANS_FEAT(AESE, aa64_aes, do_gvec_op3_ool, a, 0, gen_helper_crypto_aese)
TRANS_FEAT(AESD, aa64_aes, do_gvec_op3_ool, a, 0, gen_helper_crypto_aesd)
TRANS_FEAT(AESMC, aa64_aes, do_gvec_op2_ool, a, 0, gen_helper_crypto_aesmc)
```

`TRANS_FEAT` 宏展开为：
1. 检查 `dc_isar_feature(aa64_aes, s)` — 特性是否启用
2. 检查 `fp_access_check(s)` — FP/NEON 是否可访问
3. 调用 `do_gvec_op3_ool()` 生成 helper 调用

### 3.2 翻译辅助函数

```c
// do_gvec_op3_ool: 3 操作数 out-of-line helper
//   生成: gen_helper_crypto_xxx(vd, vn, vm, desc)
//   desc 编码 oprsz=16, maxsz=vec_reg_size

// do_gvec_op2_ool: 2 操作数 out-of-line helper
//   生成: gen_helper_crypto_xxx(vd, vm, desc)
```

### 3.3 特殊翻译 — SM3SS1

```c
static bool trans_SM3SS1(DisasContext *s, arg_SM3SS1 *a) {
    // SM3SS1 足够简单，直接用 TCG ops 内联翻译（不需要 helper）
    // Vd[127:96] = ROL32(Vn[127:96], 12) + Vm[127:96] + Va[127:96]
    // 然后 ROL32 结果 7 位
    read_vec_element_i32(s, tcg_op1, a->rn, 3, MO_32);  // element[3] = bits[127:96]
    read_vec_element_i32(s, tcg_op2, a->rm, 3, MO_32);
    read_vec_element_i32(s, tcg_op3, a->ra, 3, MO_32);
    
    tcg_gen_rotri_i32(tcg_res, tcg_op1, 20);  // ROL(x,12) = ROR(x,32-12=20)
    tcg_gen_add_i32(tcg_res, tcg_res, tcg_op2);
    tcg_gen_add_i32(tcg_res, tcg_res, tcg_op3);
    tcg_gen_rotri_i32(tcg_res, tcg_res, 25);  // ROL(x,7) = ROR(x,25)
    
    clear_vec(s, a->rd);  // 清零整个 128-bit 向量
    write_vec_element_i32(s, tcg_res, a->rd, 3, MO_32);  // 只写 [127:96]
}
```

### 3.4 SVE2 密码学指令

```c
// target/arm/tcg/translate-sve.c:7793
TRANS_FEAT_NONSTREAMING(AESMC, aa64_sve2_aes, gen_gvec_ool_zz, ...)
TRANS_FEAT_NONSTREAMING(AESE, aa64_sve2_aes, gen_gvec_ool_arg_zzz, ...)
TRANS_FEAT_NONSTREAMING(SM4E, aa64_sve2_sm4, gen_gvec_ool_arg_zzz, ...)
```

SVE2 版本的关键区别：
- 操作宽度 = 当前 VL（可能 > 128-bit）
- helper 内部以 128-bit 块循环处理
- `NONSTREAMING`：不允许在 SME streaming 模式下执行

---

## 四、Helper 实现

### 4.1 AES — SubBytes + ShiftRows + AddRoundKey

```c
// target/arm/tcg/crypto_helper.c:50
void HELPER(crypto_aese)(void *vd, void *vn, void *vm, uint32_t desc) {
    intptr_t i, opr_sz = simd_oprsz(desc);
    
    for (i = 0; i < opr_sz; i += 16) {   // 每 128-bit 块
        AESState *st = (AESState *)(vn + i);
        AESState *rk = (AESState *)(vm + i);
        AESState t;
        
        // ARM AESE: AddRoundKey → SubBytes → ShiftRows
        // 注意：ARM 的 AESE 先做 AddRoundKey，再做 SB+SR
        // 而标准 AES 的 SB+SR+MC+AK 顺序不同
        t.v = st->v ^ rk->v;                    // AddRoundKey
        aesenc_SB_SR_AK(ad, &t, &aes_zero, false); // SubBytes + ShiftRows (key=0)
    }
    clear_tail(vd, opr_sz, simd_maxsz(desc));  // 清零高位
}

void HELPER(crypto_aesmc)(void *vd, void *vm, uint32_t desc) {
    // AESMC: 仅做 MixColumns
    aesenc_MC(ad, st, false);
}
```

**关键实现细节**：
- ARM 的 AESE = AddRoundKey + SubBytes + ShiftRows（与标准 AES 轮次顺序不同）
- 调用 `include/crypto/aes-round.h` 中的底层原语
- 当宿主机支持 AES 硬件加速时（`HAVE_AES_ACCEL=true`），使用宿主机 AES 指令

### 4.2 SHA-1 实现

```c
// 三个逻辑函数
static uint32_t cho(x, y, z) { return (x & (y ^ z)) ^ z; }         // Ch(x,y,z)
static uint32_t par(x, y, z) { return x ^ y ^ z; }                  // Parity
static uint32_t maj(x, y, z) { return (x & y) | ((x | y) & z); }   // Maj(x,y,z)

// SHA1C/SHA1P/SHA1M: 4 轮迭代
static void crypto_sha1_3reg(rd, rn, rm, desc, fn) {
    for (i = 0; i < 4; i++) {
        t = fn(&d);                              // 选择 Ch/Par/Maj
        t += ROL32(d[0], 5) + n[0] + m[i];     // 标准 SHA-1 公式
        // 状态寄存器移位
        n[0] = d[3]; d[3] = d[2]; d[2] = ROR32(d[1], 2); d[1] = d[0]; d[0] = t;
    }
}

// SHA1SU0: 消息调度第一步 (W[i-16] ^ W[i-14] ^ W[i-8])
// SHA1SU1: 消息调度第二步 (ROL1)
// SHA1H: ROR32(Vm[0], 2)，其余元素清零
```

### 4.3 SHA-256 实现

```c
// 压缩函数
static uint32_t S0(x) { return ROR32(x,2) ^ ROR32(x,13) ^ ROR32(x,22); }
static uint32_t S1(x) { return ROR32(x,6) ^ ROR32(x,11) ^ ROR32(x,25); }
static uint32_t s0(x) { return ROR32(x,7) ^ ROR32(x,18) ^ (x >> 3); }
static uint32_t s1(x) { return ROR32(x,17) ^ ROR32(x,19) ^ (x >> 10); }

void HELPER(crypto_sha256h)(vd, vn, vm, desc) {
    // 4 轮 SHA-256 压缩
    for (i = 0; i < 4; i++) {
        t = Ch(n[0], n[1], n[2]) + n[3] + Σ1(n[0]) + m[i];
        // 更新工作变量
        n[3]=n[2]; n[2]=n[1]; n[1]=n[0]; n[0]=d[3]+t;
        t += Maj(d[0], d[1], d[2]) + Σ0(d[0]);
        d[3]=d[2]; d[2]=d[1]; d[1]=d[0]; d[0]=t;
    }
}

// SHA256SU0/SU1: 消息扩展
```

### 4.4 SHA-512 实现

```c
// 64-bit 版本的压缩函数
static uint64_t S0_512(x) { return ROR64(x,28) ^ ROR64(x,34) ^ ROR64(x,39); }
static uint64_t S1_512(x) { return ROR64(x,14) ^ ROR64(x,18) ^ ROR64(x,41); }
static uint64_t s0_512(x) { return ROR64(x,1) ^ ROR64(x,8) ^ (x >> 7); }
static uint64_t s1_512(x) { return ROR64(x,19) ^ ROR64(x,61) ^ (x >> 6); }

// SHA512H: 2 轮 SHA-512 压缩（128-bit 寄存器每次处理 2 个 64-bit 字）
void HELPER(crypto_sha512h)(vd, vn, vm, desc) {
    d1 += Σ1(vm[1]) + Ch(vm[1], vn[0], vn[1]);
    d0 += Σ1(d1 + vm[0]) + Ch(d1 + vm[0], vm[1], vn[0]);
}
```

### 4.5 SM3 实现（国密哈希）

```c
// SM3PARTW1: 消息扩展第一步
// W'[i] = W[i] ^ W[i+4] ^ ROL(W[i+10], 17)
// 然后: result = x ^ ROL(x,17) ^ ROL(x,9)  （P1 置换）

// SM3PARTW2: 消息扩展第二步
// d[i] ^= n[i] ^ ROL(m[i], 25)

// SM3TT1A/1B/2A/2B: 压缩函数（4种变体）
static void crypto_sm3tt(rd, rn, rm, desc, opcode) {
    imm2 = simd_data(desc);  // 选择 rm 的哪个 32-bit 元素
    
    if (opcode == 0 || opcode == 2)
        t = par(d[3], d[2], d[1]);  // Parity (rounds 0-15)
    else if (opcode == 1)
        t = maj(d[3], d[2], d[1]);  // Majority (rounds 0-15)
    else
        t = cho(d[3], d[2], d[1]);  // Choice (rounds 16-63)
    
    t += d[0] + m[imm2];
    // TT1: t += n[3] ^ ROR(d[3], 20)
    // TT2: t += n[3]; t ^= ROL(t,9) ^ ROL(t,17)  （P0 置换）
}

// SM3SS1: 内联 TCG 翻译（无 helper）
// result = ROL(ROL(Vn[3], 12) + Vm[3] + Va[3], 7)
```

### 4.6 SM4 实现（国密对称加密）

```c
// SM4E: 一轮 SM4 加密（4 轮迭代）
static void do_crypto_sm4e(rd, rn, rm) {
    d = rn;  // 输入状态
    for (i = 0; i < 4; i++) {
        t = d[(i+1)%4] ^ d[(i+2)%4] ^ d[(i+3)%4] ^ m[i];  // 轮密钥混合
        t = sm4_subword(t);  // S-Box 替换（256 字节查表）
        // L 线性变换
        d[i] ^= t ^ ROL(t,2) ^ ROL(t,10) ^ ROL(t,18) ^ ROL(t,24);
    }
}

// SM4EKEY: 密钥扩展（L' 变换不同）
static void do_crypto_sm4ekey(rd, rn, rm) {
    for (i = 0; i < 4; i++) {
        t = d[(i+1)%4] ^ d[(i+2)%4] ^ d[(i+3)%4] ^ m[i];
        t = sm4_subword(t);
        d[i] ^= t ^ ROL(t,13) ^ ROL(t,23);  // L' 线性变换
    }
}
```

SM4 S-Box 定义在 `crypto/sm4.c` (256 字节常量表)。

### 4.7 SHA-3 辅助指令

```c
// EOR3: Vd = Vn ^ Vm ^ Va（三输入异或）
// BCAX: Vd = Vn ^ (Vm & ~Va)（Bit Clear And XOR）
// RAX1: Vd = Vn ^ ROL64(Vm, 1)（Rotate And XOR）
// XAR:  Vd = ROR64(Vn ^ Vm, imm)（XOR And Rotate）

// RAX1 实现
void HELPER(crypto_rax1)(void *vd, void *vn, void *vm, uint32_t desc) {
    for (i = 0; i < opr_sz / 8; ++i) {
        d[i] = n[i] ^ rol64(m[i], 1);
    }
}
```

---

## 五、宿主机加速

### 5.1 AES 加速框架

```
include/crypto/aes-round.h          — 通用调度接口
host/include/aarch64/host/crypto/aes-round.h  — AArch64 宿主加速
host/include/x86_64/host/crypto/aes-round.h   — x86 AES-NI 加速
host/include/ppc64/host/crypto/aes-round.h    — POWER 加速
host/include/generic/host/crypto/aes-round.h  — 纯软件回退
```

### 5.2 AArch64 宿主加速实现

```c
// host/include/aarch64/host/crypto/aes-round.h
#ifdef __ARM_FEATURE_AES
# define HAVE_AES_ACCEL  true
#else
# define HAVE_AES_ACCEL  likely(cpuinfo & CPUINFO_AES)
#endif

// 使用宿主机 AES 指令加速 helper
#define aes_accel_aese   vaeseq_u8    // ARM NEON intrinsic
#define aes_accel_aesmc  vaesmcq_u8
```

**加速路径**：当 QEMU 运行在带有 AES 扩展的 AArch64 宿主上时，helper 中的 `aesenc_SB_SR_AK()` 调用会直接使用宿主机的 AESE/AESMC 指令，避免软件查表。

---

## 六、与硬件规范对比

### 6.1 AESE 指令语义

**ARM ARM 规范**：
```
AESE <Vd>.16B, <Vn>.16B
  operand = Vd EOR Vn       (AddRoundKey)
  result = SubBytes(ShiftRows(operand))
  Vd = result
```

**QEMU 实现**：
```c
t.v = st->v ^ rk->v;                         // AddRoundKey
aesenc_SB_SR_AK(ad, &t, &aes_zero, false);   // SubBytes + ShiftRows (key=0)
```

✅ **完全一致**：QEMU 通过将 AK=0 传入 `aesenc_SB_SR_AK` 实现了"只做 SB+SR 不做最后 AK"的效果。

### 6.2 SHA-256 压缩

**ARM ARM 规范**：SHA256H 执行 4 轮 SHA-256 压缩，输入/输出为 128-bit 向量。

**QEMU 实现**：精确复现了 NIST FIPS 180-4 定义的 Ch/Maj/Σ0/Σ1/σ0/σ1 函数和状态更新。

✅ **完全一致**。

### 6.3 SM3/SM4 国密算法

**GM/T 0002-2012 / GM/T 0004-2012 规范**：

QEMU 实现严格遵循国密标准：
- SM3 的 P0(x) = x ^ ROL(x,9) ^ ROL(x,17)
- SM3 的 P1(x) = x ^ ROL(x,15) ^ ROL(x,23)（消息扩展用，通过 PARTW1 helper）
- SM4 的 L(B) = B ^ ROL(B,2) ^ ROL(B,10) ^ ROL(B,18) ^ ROL(B,24)
- SM4 的 L'(B) = B ^ ROL(B,13) ^ ROL(B,23)（密钥扩展）

✅ **完全一致**。

### 6.4 注意事项

| 方面 | 硬件 | QEMU TCG |
|------|------|----------|
| 性能 | 单周期/few cycles | 软件模拟（慢 100-1000x） |
| 侧信道 | 恒定时间 | 查表可能有 cache 侧信道 |
| SVE2 向量宽度 | 真实 VL | 模拟任意 VL |
| 宿主机加速 | — | AES 可用宿主 AES-NI/CE 加速 |

---

## 七、数据流总结

```
┌───────────────────────────────────────────────────────┐
│  guest 指令: AESE V0.16B, V1.16B                     │
├───────────────────────────────────────────────────────┤
│  1. 解码 (a64.decode)                                 │
│     匹配: 01001110 00 10100 00100 10 rd:5 rn:5       │
│     提取: rd, rn → arg_r2r_q1e0                      │
├───────────────────────────────────────────────────────┤
│  2. 翻译 (translate-a64.c)                            │
│     TRANS_FEAT(AESE, aa64_aes, do_gvec_op3_ool, ...)  │
│     → 检查 ID_AA64ISAR0.AES ≠ 0                     │
│     → 检查 fp_access_check()                         │
│     → 生成 TCG: call gen_helper_crypto_aese(vd,vn,vm)│
├───────────────────────────────────────────────────────┤
│  3. 执行 (crypto_helper.c)                            │
│     HELPER(crypto_aese):                              │
│       t = Vn XOR Vm (AddRoundKey)                    │
│       result = SubBytes(ShiftRows(t))                │
│       Vd = result; clear_tail()                      │
├───────────────────────────────────────────────────────┤
│  4. 底层原语 (include/crypto/aes-round.h)             │
│     if HAVE_AES_ACCEL:                               │
│       使用宿主机 AES 指令 (vaeseq_u8/AESENC)        │
│     else:                                            │
│       纯软件 S-Box 查表 + 行移位矩阵                │
└───────────────────────────────────────────────────────┘
```

---

## 八、源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| `target/arm/tcg/crypto_helper.c` | 679 | 所有 crypto helper 实现 |
| `target/arm/tcg/a64.decode` | 835-889 | AArch64 crypto 解码规则 |
| `target/arm/tcg/neon-dp.decode` | 151-491 | AArch32 crypto 解码规则 |
| `target/arm/tcg/sve.decode` | 1805-1814 | SVE2 crypto 解码规则 |
| `target/arm/tcg/translate-a64.c` | 5457-5540 | AArch64 crypto 翻译 |
| `target/arm/tcg/translate-sve.c` | 7793-7805 | SVE2 crypto 翻译 |
| `target/arm/tcg/translate-neon.c` | 933-948 | AArch32 crypto 翻译 |
| `target/arm/cpu-features.h` | 807-865 | crypto 特性检测函数 |
| `include/crypto/aes-round.h` | — | AES 原语接口+加速调度 |
| `host/include/aarch64/host/crypto/aes-round.h` | — | AArch64 宿主 AES 加速 |
| `host/include/x86_64/host/crypto/aes-round.h` | — | x86 AES-NI 加速 |
| `include/crypto/sm4.h` | — | SM4 S-Box + subword |
| `crypto/sm4.c` | — | SM4 S-Box 常量表 |

| Helper 函数 | 指令 | 说明 |
|-------------|------|------|
| `crypto_aese` | AESE | AES 加密轮 (XOR + SB + SR) |
| `crypto_aesd` | AESD | AES 解密轮 (XOR + ISB + ISR) |
| `crypto_aesmc` | AESMC | AES MixColumns |
| `crypto_aesimc` | AESIMC | AES InvMixColumns |
| `crypto_sha1c/p/m` | SHA1C/P/M | SHA-1 压缩 (Ch/Par/Maj 变体) |
| `crypto_sha1h` | SHA1H | SHA-1 哈希固定旋转 |
| `crypto_sha1su0/su1` | SHA1SU0/SU1 | SHA-1 消息调度 |
| `crypto_sha256h/h2` | SHA256H/H2 | SHA-256 压缩 |
| `crypto_sha256su0/su1` | SHA256SU0/SU1 | SHA-256 消息扩展 |
| `crypto_sha512h/h2` | SHA512H/H2 | SHA-512 压缩 |
| `crypto_sha512su0/su1` | SHA512SU0/SU1 | SHA-512 消息扩展 |
| `crypto_sm3partw1/w2` | SM3PARTW1/W2 | SM3 消息扩展 |
| `crypto_sm3tt1a/1b/2a/2b` | SM3TT* | SM3 压缩变体 |
| `crypto_sm4e` | SM4E | SM4 加密轮 |
| `crypto_sm4ekey` | SM4EKEY | SM4 密钥扩展 |
| `crypto_rax1` | RAX1 | SHA-3 旋转异或 |
