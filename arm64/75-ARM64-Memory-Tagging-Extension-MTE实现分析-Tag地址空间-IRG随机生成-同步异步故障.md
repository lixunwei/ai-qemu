# ARM64 Memory Tagging Extension (MTE) 完整实现分析

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档编号 | arm64/75 |
| 分析对象 | FEAT_MTE (Memory Tagging Extension) 在 QEMU 中的完整实现 |
| QEMU 版本 | 11.0.50 |
| 参考规范 | ARM DDI 0487 M.b §D8.1 (Allocation Tags), §D8.2 (Tag Checking) |
| 核心文件 | `target/arm/tcg/mte_helper.c` (1171 行) |
| 核心结论 | **QEMU 实现完整的 MTE 包括独立的 tag 地址空间、4-bit tag 存储、同步/异步故障、IRG 随机生成、DC_ZVA/MOPS 集成** |

---

## 1. MTE 架构概述

### 1.1 什么是 MTE

MTE (FEAT_MTE, Armv8.5) 为每 16 字节内存关联一个 4-bit **Allocation Tag**，同时在指针的 bit[59:56] 嵌入一个 4-bit **Logical Tag**。访存时硬件比较两者，不匹配则报告错误。

```
指针结构:
┌───────┬──────┬─────────────────────────────────────────────┐
│ 63:60 │59:56 │ 55:0 (虚拟地址，受 TBI 控制)                  │
│ sign  │ TAG  │ virtual address                              │
└───────┴──────┴─────────────────────────────────────────────┘

内存标签存储:
┌────────────────────────────────────────────────────────────┐
│ 每 16 字节 (TAG_GRANULE) → 1 个 4-bit tag                   │
│ 每字节存储 2 个 tag (高低 nibble)                             │
│ tag 地址 = phys_addr >> (LOG2_TAG_GRANULE + 1) = addr >> 5  │
└────────────────────────────────────────────────────────────┘
```

### 1.2 MTE 操作分类

| 类别 | 指令 | 功能 |
|------|------|------|
| Tag 生成 | IRG | 生成随机 tag 插入指针 |
| Tag 算术 | ADDG/SUBG | 指针偏移 + tag 推进 |
| Tag 查询 | GMI | 从指针提取 tag 到位掩码 |
| Tag 读写 | LDG/STG/ST2G/STZGM/STGM/LDGM | 加载/存储 allocation tag |
| 检查 | 隐式 (每次内存访问) | 比较 logical tag 与 allocation tag |

---

## 2. Tag 存储架构 (QEMU 实现)

### 2.1 独立的 Tag 地址空间

QEMU 为 MTE 创建独立的内存地址空间：

```c
// target/arm/cpu.h:2371
typedef enum ARMASIdx {
    ARMASIdx_NS = 0,    // Non-secure 普通内存
    ARMASIdx_S = 1,     // Secure 普通内存
    ARMASIdx_TagNS = 2, // Non-secure Tag 内存
    ARMASIdx_TagS = 3,  // Secure Tag 内存
} ARMASIdx;
```

### 2.2 Tag 内存初始化 (virt 机器)

```c
// hw/arm/virt.c:2081
static void create_tag_ram(MemoryRegion *tag_sysmem,
                           hwaddr base, hwaddr size, const char *name)
{
    MemoryRegion *tagram = g_new(MemoryRegion, 1);
    // Tag 内存大小 = 普通内存 / 32 (每 16 字节占 4 bit = 1/32)
    memory_region_init_ram(tagram, NULL, name, size / 32, &error_fatal);
    // Tag 地址 = 物理地址 / 32
    memory_region_add_subregion(tag_sysmem, base / 32, tagram);
}
```

关键设计：
- Tag 地址空间大小 = `UINT64_MAX / 32`
- 每段 RAM 对应的 tag RAM 大小 = `RAM_size / 32`
- 映射关系: `tag_addr = phys_addr / 32` (即 `phys_addr >> 5`)

### 2.3 Tag 存储格式

```c
// target/arm/cpu.h:2686
#define LOG2_TAG_GRANULE 4   // 每个 tag 管理 16 字节
#define TAG_GRANULE      16  // (1 << 4)

// 每字节存 2 个 tag (little-endian nibbles):
//   byte[addr >> 5] 的 bit[3:0] = 低 16 字节的 tag
//   byte[addr >> 5] 的 bit[7:4] = 高 16 字节的 tag
```

---

## 3. Tag 地址解析 (allocation_tag_mem_probe)

### 3.1 核心函数

```c
// target/arm/tcg/mte_helper.c:61
uint8_t *allocation_tag_mem_probe(CPUARMState *env, int ptr_mmu_idx,
                                   uint64_t ptr, ...)
```

### 3.2 System 模式流程

```
guest 虚拟地址
    │
    ▼
probe_access_full() ─────── TLB 查找，获取物理地址 + MemAttr
    │
    ▼
检查 full->extra.arm.pte_attrs == 0xf0 ?
    │ (0xf0 = Tagged Normal Memory)
    │ 否 → return NULL (不检查 tag)
    ▼
计算 tag_paddr = ptr_paddr >> (LOG2_TAG_GRANULE + 1) = ptr_paddr >> 5
    │
    ▼
选择 tag 地址空间:
    secure ? ARMASIdx_TagS : ARMASIdx_TagNS
    │
    ▼
address_space_translate(tag_as, tag_paddr, ...) ─── 查找 tag 内存区域
    │
    ▼
return memory_region_get_ram_ptr(mr) + xlat ─── 返回 host 指针
```

### 3.3 页属性判定

页表遍历后的 MemAttr 存储在 TLB 条目中：

```c
// target/arm/tcg/tlb_helper.c:367
res.f.extra.arm.pte_attrs = res.cacheattrs.attrs;
```

- `pte_attrs == 0xf0`: Tagged Normal Memory (Inner WB, Outer WB, Tagged)
- 其他值: Non-tagged memory → MTE 检查跳过

### 3.4 User 模式流程

```c
// CONFIG_USER_ONLY 路径:
// 1. 检查 PAGE_ANON | PAGE_MTE flags
// 2. 从 page_get_target_data() 获取 tag 存储
// 3. 返回 tags[index]
```

---

## 4. Tag 检查机制

### 4.1 翻译时 MTE 检查生成

```c
// target/arm/tcg/translate-a64.c:300
static TCGv_i64 gen_mte_check1_mmuidx(DisasContext *s, TCGv_i64 addr, ...)
{
    if (tag_checked && s->mte_active[is_unpriv]) {
        // 构建 MTEDESC 描述符
        desc = FIELD_DP32(desc, MTEDESC, MIDX, core_idx);
        desc = FIELD_DP32(desc, MTEDESC, TBI, s->tbid);
        desc = FIELD_DP32(desc, MTEDESC, TCMA, s->tcma);
        desc = FIELD_DP32(desc, MTEDESC, WRITE, is_write);
        desc = FIELD_DP32(desc, MTEDESC, ALIGN, memop_alignment_bits(memop));
        desc = FIELD_DP32(desc, MTEDESC, SIZEM1, memop_size(memop) - 1);

        // 生成 helper 调用
        gen_helper_mte_check(ret, tcg_env, tcg_constant_i32(desc), addr);
        return ret;  // 返回清理后的地址
    }
    return clean_data_tbi(s, addr);  // MTE 未激活，只清理 TBI
}
```

### 4.2 MTEDESC 描述符

```c
// target/arm/internals.h:1555
FIELD(MTEDESC, MIDX,   0,  4)  // MMU index
FIELD(MTEDESC, TBI,    4,  2)  // TBI[1:0] per address range
FIELD(MTEDESC, TCMA,   6,  2)  // TCMA[1:0] per address range
FIELD(MTEDESC, WRITE,  8,  1)  // 是否为写操作
FIELD(MTEDESC, ALIGN,  9,  3)  // 对齐要求
FIELD(MTEDESC, SIZEM1, 12, 20) // 访问大小 - 1
```

### 4.3 运行时检查 (mte_probe_int)

```c
// mte_helper.c:779
static int mte_probe_int(CPUARMState *env, uint32_t desc, uint64_t ptr, ...)
{
    // 1. TBI 检查: 如果 TBI 未使能，不检查
    if (!tbi_check(desc, bit55)) return -1;

    // 2. 从指针提取 logical tag (bit[59:56])
    ptr_tag = allocation_tag_from_addr(ptr);  // extract64(ptr, 56, 4)

    // 3. TCMA 检查: tag 为 0x0 或 0xF 时可能跳过
    if (tcma_check(desc, bit55, ptr_tag)) return 1;

    // 4. 计算需要检查的 tag 数量
    tag_count = ((tag_last - tag_first) / TAG_GRANULE) + 1;

    // 5. 获取 tag 内存指针
    mem1 = allocation_tag_mem(env, mmu_idx, ptr, type, ...);

    // 6. 逐 nibble 比较 (checkN)
    n = checkN(mem1, ptr & TAG_GRANULE, ptr_tag, tag_count);

    // 7. 比较结果
    if (n == tag_count) return 1;  // 全部匹配
    *fault = tag_first + n * TAG_GRANULE; // 第一个失败位置
    return 0;  // 失败
}
```

### 4.4 checkN 算法

```c
// mte_helper.c:683
static int checkN(uint8_t *mem, int odd, int cmp, int count)
{
    cmp *= 0x11;  // 复制 tag 到两个 nibble
    diff = *mem++ ^ cmp;

    while (1) {
        // 检查偶数 nibble
        if ((diff) & 0x0f) break;
        if (++n == count) break;
        // 检查奇数 nibble
        if ((diff) & 0xf0) break;
        if (++n == count) break;
        diff = *mem++ ^ cmp;
    }
    return n;  // 成功匹配的 tag 数
}
```

---

## 5. Tag Check 故障处理

### 5.1 SCTLR_ELx.TCF 控制

```c
// mte_helper.c:601
void mte_check_fail(CPUARMState *env, uint32_t desc, uint64_t dirty_ptr, ...)
{
    // 从 SCTLR_ELx 获取 TCF (Tag Check Fault) 模式
    tcf = extract64(sctlr, el == 0 ? 38 : 40, 2);

    switch (tcf) {
    case 0: // 不报告 (已在 TB flag 中优化掉)
        g_assert_not_reached();
    case 1: // 同步异常
        mte_sync_check_fail(env, desc, dirty_ptr, ra);
        break;
    case 2: // 异步标志
        mte_async_check_fail(env, dirty_ptr, ra, arm_mmu_idx, el);
        break;
    case 3: // 写=异步, 读=同步 (Armv8.7 MTE3)
        if (FIELD_EX32(desc, MTEDESC, WRITE))
            mte_async_check_fail(...);
        else
            mte_sync_check_fail(...);
        break;
    }
}
```

### 5.2 同步故障

```c
static void mte_sync_check_fail(CPUARMState *env, uint32_t desc,
                                uint64_t dirty_ptr, uintptr_t ra)
{
    env->exception.vaddress = dirty_ptr;
    // 生成 Data Abort syndrome (DFSC = 0x11 = Tag Check Fault)
    syn = syn_data_abort_no_iss(el != 0, 0, 0, 0, 0, is_write, 0x11);
    raise_exception_ra(env, EXCP_DATA_ABORT, syn, exception_target_el(env), ra);
}
```

### 5.3 异步故障

```c
static void mte_async_check_fail(CPUARMState *env, uint64_t dirty_ptr, ...)
{
    // 根据地址范围设置 TFSR_ELx 的对应位
    select = extract64(dirty_ptr, 55, 1);  // 高/低地址范围
    env->cp15.tfsr_el[el] |= 1 << select; // 设置 TF0/TF1

    // User-only: 触发 cpu_exit 以便尽快报告
    cpu_exit(env_cpu(env));
}
```

---

## 6. Tag 生成 (IRG 指令)

### 6.1 伪随机 Tag 生成

```c
// mte_helper.c:209
uint64_t HELPER(irg)(CPUARMState *env, uint64_t rn, uint64_t rm)
{
    uint16_t exclude = extract32(rm | env->cp15.gcr_el1, 0, 16);
    int rrnd = extract32(env->cp15.gcr_el1, 16, 1);  // GCR_EL1.RRND
    int start = extract32(env->cp15.rgsr_el1, 0, 4);  // 当前 seed tag
    int seed = extract32(env->cp15.rgsr_el1, 8, 16);  // 16-bit LFSR

    // 4 轮 LFSR 生成 4-bit 随机偏移
    for (i = offset = 0; i < 4; ++i) {
        int top = (extract32(seed, 5, 1) ^ extract32(seed, 3, 1) ^
                   extract32(seed, 2, 1) ^ extract32(seed, 0, 1));
        seed = (top << 15) | (seed >> 1);
        offset |= top << i;
    }

    // 从 start+offset 开始跳过 excluded tags
    rtag = choose_nonexcluded_tag(start, offset, exclude);

    // 更新 RGSR_EL1 状态
    env->cp15.rgsr_el1 = rtag | (seed << 8);

    // 将 tag 嵌入地址 bit[59:56]
    return address_with_allocation_tag(rn, rtag);
}
```

### 6.2 相关寄存器

| 寄存器 | 功能 |
|--------|------|
| GCR_EL1 | bit[15:0]: exclude mask; bit[16]: RRND |
| RGSR_EL1 | bit[3:0]: 当前 tag; bit[23:8]: LFSR seed |
| TFSR_EL1 | 异步 tag check 故障标志 |

---

## 7. TB Flag MTE 优化

### 7.1 MTE_ACTIVE 判定

```c
// target/arm/tcg/hflags.c:402
if (cpu_isar_feature(aa64_mte, env_archcpu(env))) {
    if (allocation_tag_access_enabled(env, el, sctlr)) {  // ATA 使能
        if (tbid                             // TBI 使能
            && !(env->pstate & PSTATE_TCO)   // Tag Check Override 未设置
            && (sctlr & SCTLR_TCF))          // TCF != 0 (会报告故障)
        {
            DP_TBFLAG_A64(flags, MTE_ACTIVE, 1);
        }
    }
}
```

### 7.2 优化效果

- `MTE_ACTIVE = 0`: 翻译时不生成 `gen_helper_mte_check` 调用 → 零开销
- `MTE_ACTIVE = 1`: 每个内存访问生成 helper 调用进行 tag 检查

### 7.3 TCO (Tag Check Override)

当 `PSTATE.TCO = 1` 时：
- MTE_ACTIVE 被清除
- 所有内存访问跳过 tag 检查
- 用于 memcpy 等不关心 tag 的批量操作

---

## 8. 特殊指令的 MTE 集成

### 8.1 DC_ZVA (零化缓存行)

```c
// mte_helper.c:922
uint64_t HELPER(mte_check_zva)(CPUARMState *env, uint32_t desc, uint64_t ptr)
{
    // 对齐到 DCZ block (通常 64 字节)
    // 比较整个 block 的 tag (可用位运算一次比较多个)
    switch (log2_tag_bytes) {
    case 1: // 64 bytes → 4 tags → 16 bits
        mem_tag = cpu_to_le16(*(uint16_t *)mem);
        ptr_tag *= 0x1111u;  // 复制 tag 到 4 个 nibble
        break;
    ...
    }
    if (mem_tag == ptr_tag) goto done;
    // 失败: 定位第一个不匹配的 granule
    i = ctz64(mem_tag ^ ptr_tag) >> 4;
}
```

### 8.2 FEAT_MOPS (Memory Operations)

```c
// mte_helper.c:1025
uint64_t mte_mops_probe(CPUARMState *env, uint64_t ptr, uint64_t size, ...)
{
    // CPYP/CPYM/CPYE 等大块内存操作的 tag 检查
    // 使用 true probe (不触发异常)
    mem = allocation_tag_mem_probe(env, mmu_idx, ptr, ..., true, 0);
    if (!mem) return size;  // 无 tag → 全部通过
    n = checkN(mem, ptr & TAG_GRANULE, ptr_tag, tag_count);
    // 返回可安全访问的字节数
}
```

### 8.3 LDGM/STGM (批量 Tag 操作)

```c
// 一次读/写整个 block 的 tag
// gm_blocksize 配置 block 大小: 32/64/128/256 字节
uint64_t HELPER(ldgm)(CPUARMState *env, uint64_t ptr, uint64_t xt)
{
    int gm_bs_bytes = 4 << env_archcpu(env)->gm_blocksize;
    // 直接按字节/半字/字/双字读取 tag 数据
}
```

---

## 9. 并发安全

### 9.1 原子 Tag 写入

```c
// mte_helper.c:308
static void store_tag1_parallel(uint64_t ptr, uint8_t *mem, int tag)
{
    int ofs = extract32(ptr, LOG2_TAG_GRANULE, 1) * 4;
    uint8_t old = qatomic_read(mem);
    while (1) {
        uint8_t new = deposit32(old, ofs, 4, tag);
        uint8_t cmp = qatomic_cmpxchg(mem, old, new);
        if (likely(cmp == old)) return;
        old = cmp;  // CAS 失败，重试
    }
}
```

为什么需要原子操作：
- 一个字节存储 2 个 tag (2 个 16-byte granule)
- 两个 vCPU 可能同时写同一字节的不同 nibble
- MTTCG 模式下必须使用 CAS 保证原子性

---

## 10. 与 ARM 规范的一致性评估

### 10.1 实现完整度

| 规范要求 | QEMU 状态 | 说明 |
|---------|:--------:|------|
| 4-bit allocation tag per 16B granule | ✅ | LOG2_TAG_GRANULE=4 |
| Logical tag in ptr[59:56] | ✅ | allocation_tag_from_addr: extract64(ptr,56,4) |
| IRG 随机 tag 生成 | ✅ | LFSR + exclude mask |
| LDG/STG/ST2G tag 读写 | ✅ | 完整实现 |
| LDGM/STGM/STZGM 批量操作 | ✅ | gm_blocksize 可配置 |
| 同步异常 (TCF=1) | ✅ | DFSC=0x11 |
| 异步标志 (TCF=2) | ✅ | TFSR_ELx.TF0/TF1 |
| 混合模式 (TCF=3, MTE3) | ✅ | 读同步+写异步 |
| TCMA (Tag Check Match All) | ✅ | tcma_check() |
| TCO (Tag Check Override) | ✅ | PSTATE.TCO → MTE_ACTIVE=0 |
| Tagged Normal Memory 属性 | ✅ | pte_attrs == 0xf0 |
| Separate tag address space | ✅ | ARMASIdx_TagNS/TagS |
| FEAT_MOPS integration | ✅ | mte_mops_probe/probe_rev/set_tags |
| DC_ZVA integration | ✅ | mte_check_zva |
| Migration dirty tracking | ✅ | physical_memory_set_dirty_flag |
| 并发原子性 | ✅ | store_tag1_parallel (CAS) |

### 10.2 QEMU 的 IMPDEF 选择

| 决定 | QEMU 实现 | 规范允许 |
|------|----------|---------|
| GCR_EL1.RRND=1 时的随机源 | 继续使用确定性 LFSR (强制非零 seed) | ✅ IMPDEF |
| gm_blocksize 默认值 | 可配置 (3-6) | ✅ IMPDEF |
| MMIO 页上的 tag 行为 | 返回 NULL (不检查), log 警告 | ✅ 合理 |

### 10.3 与真实硬件的差异

| 方面 | 真实硬件 | QEMU |
|------|---------|------|
| Tag 存储位置 | L2 cache / 专用 SRAM | 独立 AddressSpace + RAM |
| Tag 检查时机 | 与 L1 cache 访问同步 (零延迟) | helper 调用 (有开销) |
| LFSR 实现 | 硬件并行, 无可见延迟 | 软件模拟 4 轮 LFSR |
| 跨页 tag 访问 | 硬件自动处理 | 需要两次 probe |
| Tag 在 cache 中的存储 | 通常内嵌在 ECC 位中 | 独立内存映射 |

**总结**: QEMU 的 MTE 实现在功能上与规范**完全一致**，IMPDEF 选择合理。主要差异在于性能模型——真实硬件的 tag 检查与 cache 访问同步完成（近零额外延迟），而 QEMU 每次访存需要一个 helper 函数调用。这是功能模拟器的固有局限。
