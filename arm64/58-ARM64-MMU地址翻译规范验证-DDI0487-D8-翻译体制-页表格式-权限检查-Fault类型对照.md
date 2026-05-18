# ARM64 MMU 地址翻译规范验证：DDI 0487 Chapter D8 交叉校验（翻译体制 / 页表格式 / 权限检查 / Fault 类型对照）

## 1. 概述

本文目的是把现有 QEMU ARM64 MMU/PTW 分析文档，与 ARM Architecture Reference Manual for A-profile architecture（DDI 0487 M.b）中 **Chapter D8 Address Translation**、以及 **Chapter D7 Memory system architecture basics** 做一次逐项交叉验证，确认哪些结论与规范一致，哪些属于 QEMU 实现层面的简化，哪些需要勘误或补充边界条件。

### 1.1 验证对象

已核对文档：

- `darren/arm64/11-MMU-TLB深度分析.md`
- `darren/arm64/30-ARM64-MMU系统寄存器与页表遍历深度分析.md`
- `darren/arm64/38-ARM64内存管理深度分析-页表遍历-TLB-Stage2翻译与属性合并.md`
- `darren/arm64/49-ARM64-页表遍历PTW深度分析-Stage1-Stage2翻译-权限检查-Fault处理-安全属性传播.md`
- `darren/arm64/51-ARM64-内存属性与缓存一致性深度分析-MAIR-TCR属性编码-Device-Normal类型-缓存维护指令.md`

### 1.2 规范依据

本文主要依据 JSONL 中抽取的以下规范条目：

- **DDI 0487 §D8.1** Address translation
- **DDI 0487 §D8.1.1** Translation granules
- **DDI 0487 §D8.1.2** Translation regimes
- **DDI 0487 §D8.1.3** Relationship between translation regimes and implemented Exception levels
- **DDI 0487 §D8.1.4** The system registers relevant to MMU operation
- **DDI 0487 §D8.2 ~ §D8.2.4** Translation process / Translation table walk / TTBR selection
- **DDI 0487 §D8.2.8 ~ §D8.2.10** 4KB/16KB/64KB granule walk properties
- **DDI 0487 §D8.3 / §D8.3.1** Translation table descriptor formats
- **DDI 0487 §D8.4 ~ §D8.4.4** Memory access control / Stage-1 / Stage-2 permissions
- **DDI 0487 §D8.5 / §D8.5.1 / §D8.5.2** AF / dirty state hardware update
- **DDI 0487 §D8.6 ~ §D8.6.7** MAIR / MemAttr / Shareability / FWB
- **DDI 0487 §D8.15 / §D8.15.1 / §D8.15.4 / §D8.15.5** Memory aborts / MMU fault types / checking sequence / priority
- **DDI 0487 §D7.1 / §D7.1.2 / §D7.8.6 / §D7.8.7** memory system basics / memory attributes / permission checking / abort exceptions

### 1.3 QEMU 参考实现

本次对照同时复核了 QEMU 代码：

- `target/arm/ptw.c`
- `target/arm/internals.h`

重点函数/数据结构：

- `get_phys_addr_lpae()`
- `get_phys_addr_twostage()`
- `check_s2_mmu_setup()`
- `get_S1prot()`
- `get_S2prot()`
- `combine_cacheattrs()`
- `ARMFaultType`
- `ARMMMUFaultInfo`

### 1.4 结论先行

总体上，现有文档对 **QEMU 的 VMSAv8-64 PTW 主路径** 描述是扎实的，尤其是：

- TTBR/TCR/VTCR/MAIR 的取值来源；
- `get_phys_addr_lpae()` 的主循环；
- `get_S1prot()` / `get_S2prot()` 的权限求值；
- Stage-1 + Stage-2 权限交集；
- `combine_cacheattrs()` 的大方向；
- QEMU 对 ASID / cacheability / shareability 的简化。

但和规范逐条对照后，仍有几类需要修正：

1. **把规范中的“翻译体制（translation regime）”讲得过于 EL 级别化**，忽略 Security state / Realm / Root 的影响；
2. **把页表格式讲成固定的 `SH[9:8] / OA[47:12]` 模式**，未充分覆盖 `TCR_ELx.DS` / `VTCR_EL2.DS` 使能后 `OA[51:50]` 占用 `descriptor[9:8]` 的情况；
3. **把 4KB/16KB/64KB 粒度的 lookup level 数量讲成静态结论**，未充分覆盖 4KB + DS=1 的 `level -1`、16KB Stage-2 拼接表等边界；
4. **把 QEMU 的 `ARMFaultType` 内部枚举直接当成架构定义的 MMU fault 列表**，这与 DDI 0487 §D8.15.1 不一致；
5. **FWB 模式下 Stage-2 `MemAttr` 的解释被过度简化**，与 DDI 0487 §D8.6.6 和 QEMU `combined_attrs_fwb()` 的实现都不完全一致。

---

## 2. 翻译体制验证（DDI 0487 §D8.1）

### 2.1 架构定义的 translation regime 不是“只有 EL1/EL2/EL3”

**DDI 0487 §D8.1.2** 明确指出，translation regime 由以下因素共同决定：

- 当前 Security state
- 当前 Exception level
- 是否实现 EL2 / EL3
- `HCR_EL2` 设置
- 实现的特性（如 VHE / RME 等）

规范定义的不只是“EL1、EL2、EL3 三套页表”，而是更细粒度的 regime，例如：

- Non-secure EL1&0 translation regime
- Secure EL1&0 translation regime
- Realm EL1&0 translation regime
- Non-secure EL2 translation regime
- Secure EL2 translation regime
- Realm EL2 translation regime
- Non-secure EL2&0 translation regime
- Secure EL2&0 translation regime
- Realm EL2&0 translation regime
- EL3 translation regime

因此：

- 现有文档把 QEMU 的 `ARMMMUIdx_E10_* / E2 / E20_* / SE3` 解释成“EL1&0 / EL2 / EL2&0 / EL3 四套翻译体制”，**作为 QEMU 软件实现摘要是对的**；
- 但若把它当成 **架构层面的完整 regime 定义**，则不够完整，因为规范还区分 Secure / Non-secure / Realm / Root 输出空间。

### 2.2 只有 EL1&0 regime 支持 Stage-2

**DDI 0487 §D8.1.2** 明确：

> Only the EL1&0 translation regimes support a stage 2 translation.

这与 QEMU 主实现吻合：

- Stage-2 用于把 **EL1&0 Stage-1 的输出 IPA** 再翻译为 PA；
- `get_phys_addr_twostage()` 先做 Stage-1（VA→IPA），再做 Stage-2（IPA→PA）；
- Stage-2 也会参与 **Stage-1 页表遍历过程中的 descriptor fetch**（即 S1 page-table walk 访问页表页时，若 EL2 使能，也要过 S2）。

这与现有文档 38 / 49 的主线描述一致。

### 2.3 EL1&0 双 VA range 与 EL2/EL3 单 VA range

**DDI 0487 §D8.2.4** 规定：当 stage-1 regime 支持 two VA ranges 时：

- `TTBR0_ELx` 指向低地址范围；
- `TTBR1_ELx` 指向高地址范围；
- `TCR_ELx.T0SZ / T1SZ` 配置低/高范围大小；
- `VA[55]` 用于选择 TTBR0 还是 TTBR1；
- 不落在 low/high range 的地址会产生 **level 0 Translation fault**。

QEMU 实现与此一致：

- `aa64_va_parameters()` 解码 `select`；
- `regime_ttbr()` 按 `param.select` 选择 TTBR0/1；
- `get_phys_addr_lpae()` 对地址上方位进行合法性检查，不匹配时 `goto do_translation_fault`；
- `param.epd` 生效时直接产生 Translation fault。

### 2.4 EL2 使能后，EL1&0 的页表基址是 IPA，不再是 PA

这是规范里非常重要、但文档里容易被“寄存器基址”话术冲淡的一点。

**DDI 0487 §D8.2.3** 明确：

- 若 EL2 未实现或未使能，则 EL1&0 Stage-1 的 `TTBR_ELx` 基址是 **PA**；
- 若 EL2 已使能，则 **EL1&0 Stage-1 的页表基址是 IPA**；
- 此时 Stage-1 table walk 对页表页的访问本身也要做 Stage-2 translation。

ASCII 图如下：

```text
EL2 disabled:
  VA --S1 walk--> PA
      TTBR0/1 holds PA of translation table

EL2 enabled:
  VA --S1 walk--> IPA --S2 walk--> PA
      TTBR0/1 holds IPA of S1 translation table
      descriptor fetch during S1 walk is also subject to S2
```

这和 QEMU 的 `S1_ptw_translate()`、`fi->s1ptw`、`HCR_EL2.PTW` 保护逻辑完全一致。现有文档 38 / 49 的主线是正确的，但建议把“**EL2 打开后，EL1 页表基址语义从 PA 变成 IPA**”单独强调出来，因为这是 `DDI 0487 §D8.2.3` 的核心规范点。

### 2.5 QEMU 对 translation regime 的抽象与规范的对应关系

| QEMU 视角 | 规范视角 | 结论 |
|---|---|---|
| `ARMMMUIdx_E10_*` | EL1&0 translation regime（还受 Security state 细分） | ✓ 主抽象正确 |
| `ARMMMUIdx_E2` | EL2 translation regime | ✓ |
| `ARMMMUIdx_E20_*` | EL2&0 translation regime（VHE） | ✓ |
| `ARMMMUIdx_SE3` | EL3 translation regime | ✓ |
| `ARMMMUIdx_Stage2` / `Stage2_S` | EL1&0 regime 的 stage-2 translation | ✓ |
| “regime 只有 4 个” | 规范定义还区分 NS/S/Realm/Root | ✗ 过度简化 |

### 2.6 本节结论

- 文档 11/30/38/49 对 **QEMU 代码中的 regime 选择** 解释基本正确；
- 若上升到 **DDI 0487 架构定义**，则应补充：regime 还受 Security state / Realm / Root 影响；
- 文档应强调：**EL1&0 Stage-1 的页表基址在 EL2 开启时是 IPA**，这不是细节，而是 DDI 0487 §D8.2.3 的关键语义。

---

## 3. 页表格式验证

### 3.1 颗粒大小与 lookup level 数量（DDI 0487 §D8.1.1, §D8.2.8~§D8.2.10）

规范给出的高层结论：

- **4KB granule**：page size = 4KB，最多可用 **5 个 lookup level**（4KB + DS=1 时可出现 `level -1`）
- **16KB granule**：page size = 16KB，最多 **4 个 lookup level**
- **64KB granule**：page size = 64KB，最多 **3 个 lookup level**

QEMU 对应实现：

- `stride = arm_granule_bits(param.gran) - 3`
  - 4KB → 9
  - 16KB → 11
  - 64KB → 13
- Stage-1 起始级别：`level = 4 - (inputsize - 4) / stride`
- Stage-2 起始级别：`check_s2_mmu_setup()`
- Block 合法级别：`lpae_block_desc_valid()`

QEMU `lpae_block_desc_valid()`：

```text
4KB : block valid at level 1/2, and level 0 only when DS=1
16KB: block valid at level 2, and level 1 only when DS=1
64KB: block valid at level 2, and level 1 only when PA max = 52 bits
```

这与规范方向一致：lookup level 是否可用、某级是否允许 block descriptor，确实依赖 granule、DS/LPA、PA size 等条件，而不是“固定 L0~L3”。

### 3.2 文档中“L0-L3 四级页表”的说法不够完整

现有文档普遍把 AArch64 LPAE 图示为：

```text
L0 -> L1 -> L2 -> L3
```

这对 **常见 48-bit VA / 4KB granule / DS=0** 场景是对的，但对 DDI 0487 全量语义不够完整：

- **4KB + DS=1** 时，规范允许出现 **level -1**；
- **Stage-2** 还可能出现 **concatenated translation tables**（DDI 0487 §D8.2.2）；
- **16KB / 64KB** 的起始 lookup level 与 `SL0`/`SL2`/`T0SZ` 组合有关；
- 某些 lookup level 的 block descriptor 可能是 reserved/invalid。

因此，文档里凡是把页表级数写成“固定四级”的，应理解为“常见实现路径”，不能当成 DDI 0487 的完整归纳。

### 3.3 Translation table walk 规范顺序（DDI 0487 §D8.2.1）

规范顺序可总结为：

```text
VA/IPA
  -> select TTBR
  -> validate TTBR range / enable state
  -> check TTBR base address size
  -> fetch descriptor atomically
  -> descriptor valid/type check
  -> if table desc: accumulate hierarchical attrs and continue
  -> if block/page desc: form OA, then do AF / permission / alignment / dirty handling
```

QEMU `get_phys_addr_lpae()` 的主要步骤与此匹配：

```text
1. aa64_va_parameters()
2. select TTBR
3. EPD / TxSZ legality check
4. compute start level
5. extract TTBR base addr
6. walk descriptor loop
7. AF / DBM handling
8. attrs extraction + tableattrs merge
9. permission check
10. shareability / MAIR / final PA
```

### 3.4 Descriptor 基本格式验证（DDI 0487 §D8.3 / §D8.3.1）

规范核心点：

- `descriptor[0] = 0` → invalid descriptor
- `descriptor[0] = 1` → valid descriptor
- `level < 3` 时：
  - `bit[1] = 0` → Block descriptor
  - `bit[1] = 1` → Table descriptor
- `level == 3` 时：
  - `bit[1] = 1` → Page descriptor
  - `bit[1] = 0` → reserved, treated as invalid

这与 QEMU 判断一致：

```c
if (!(descriptor & 1) ||
    (!(descriptor & 2) && !lpae_block_desc_valid(...))) {
    goto do_translation_fault;
}
```

即：

- `Valid=0` → Translation fault
- 在不允许 block 的层级上出现 `bit1=0` → Translation fault

### 3.5 Table descriptor 的层级属性传播（hierarchical controls）

**DDI 0487 §D8.2.1** 与 **§D8.4** 都强调，Table descriptor 可以把一部分属性传播到下级 lookup。

QEMU 的 `tableattrs |= extract64(descriptor, 59, 5)` 收集的正是：

- `NSTable`
- `APTable`
- `UXNTable`
- `PXNTable`

然后在最终 `attrs` 合并时：

- `UXN/PXN` 下传；
- `APTable` 对 `AP[2:1]` 施加限制；
- `HPD=1` 时，除 `NSTable` 外层级权限被禁用。

这与 DDI 0487 §D8.4 “hierarchical access permissions” 的精神一致。

### 3.6 DS=1 时 descriptor 位图不能再按旧格式理解

这是现有文档最容易漏掉的一点。

在很多旧式图示中，descriptor 低属性位写成：

```text
[9:8] SH
[7:6] AP / S2AP
[5]   NS
[4:2] AttrIndx / MemAttr
```

但 **DDI 0487 §D8.6.2 / §D8.6.7** 与 QEMU 实现都说明：

- 当 `TCR_ELx.DS = 1`（或 `VTCR_EL2.DS = 1`）时，
- **descriptor bits[9:8] 会被重定义为 `OA[51:50]`**，
- 此时 shareability 不再来自 descriptor，而来自 `TCR_ELx.SHx` / `VTCR_EL2.SH0`。

QEMU 代码也明确写了：

```c
if (param.ds) {
    result->cacheattrs.shareability = param.sh;
} else {
    result->cacheattrs.shareability = extract32(attrs, 8, 2);
}
```

因此，文档 38 / 49 中把 `SH[9:8]` 作为固定字段的图，只对 **DS=0 的常见路径** 成立；若覆盖 DDI 0487 D8 全部语义，则需要补充 DS=1 例外。

### 3.7 Page table walk ASCII 对照图

```text
                 +------------------------------+
Input Address -->| range check / TTBR select    |
                 +--------------+---------------+
                                |
                                v
                        +---------------+
                        | TTBR base addr |
                        +-------+-------+
                                |
                     level n    v
                 +----------------------+
                 | fetch descriptor     |
                 | (atomic 8-byte read) |
                 +----------+-----------+
                            |
            +---------------+----------------+
            |                                |
            v                                v
   bit0=0 or invalid block level       valid descriptor
   -> Translation fault                     |
                                            |
                                 +----------+----------+
                                 | bit1=1 && level<3 ? |
                                 +----+-----------+----+
                                      |           |
                                    yes           no
                                      |           |
                                      v           v
                          Table descriptor   Block/Page descriptor
                          - collect attrs    - form OA
                          - next level       - AF / permission
                                             - alignment / dirty
                                             - final attrs
```

### 3.8 本节结论

- 文档 30 / 38 / 49 对 **descriptor 有效位、table/block/page 基本判断** 是正确的；
- 对 **常见 4KB + 48-bit VA + DS=0** 场景的页表遍历描述也基本正确；
- 需要补充或修正的点：
  - 4KB granule 在 DS=1 时可出现 `level -1`；
  - Stage-2 可使用 concatenated tables；
  - `SH[9:8]` 并非在所有情形下都是 shareability 字段；
  - block/page 支持层级依赖 granule 与 DS/LPA，不能写成固定常量表。

---

## 4. 权限检查验证

### 4.1 Stage-1 权限（DDI 0487 §D8.4.1）

规范层面，Stage-1 permission 控制的是：

- Priv Read / Priv Write
- Unpriv Read / Unpriv Write
- Priv Execute / Unpriv Execute
- 以及 WXN / UWNX 一类“可写不可执行”约束

在 QEMU 的 direct-permission 路径里，这被折叠成：

- `AP[2:1]` → user/privileged 的读写基权限；
- `XN` / `PXN` → 执行权限；
- `SCTLR_WXN`、PAN / EPAN、Security-space rule → 额外限制。

现有文档 11 / 38 / 49 对 `get_S1prot()` 的总体说明是正确的。

#### 4.1.1 AP[2:1] 的常见解释

对 AArch64 direct permission，常见映射可以写成：

| AP[2:1] | EL1/EL2/EL3 | EL0 |
|---|---|---|
| 00 | RW | 无访问 |
| 01 | RW | RW |
| 10 | RO | 无访问 |
| 11 | RO | RO |

QEMU 也是先把 `ap` 转换为 `user_rw` / `prot_rw`，再交给 `get_S1prot()` 做 PAN/WXN/XN/PXN 叠加。

#### 4.1.2 Hierarchical permission 的作用

**DDI 0487 §D8.4** 指出，除了 block/page descriptor 的 direct permission 外，若启用 hierarchical permission，还会受到 Table descriptor 的限制。

QEMU 体现为：

- `APTable` 收紧下级 `AP`；
- `UXNTable/PXNTable` 强制下级不可执行；
- `HPD=1` 时禁用除 `NSTable` 外的大部分层级权限传播。

这与文档 38 / 49 的解释一致。

### 4.2 Stage-2 权限（DDI 0487 §D8.4.2）

规范层面，Stage-2 permission 的概念名称比 QEMU 代码更抽象：

- RO / WO / RW / MRO
- uX / pX / puX
- direct permission 或 indirect permission (`S2PIE`)

QEMU 在 VMSAv8-64 direct-permission 路径中，用更直接的编码：

- `S2AP[1:0]` → 读/写许可
- `XN`（或 FEAT_XNX 下的两位 XN）→ 执行许可

`get_S2prot()` 的行为：

| S2AP[1:0] | 结果 |
|---|---|
| 00 | no access |
| 01 | read-only |
| 10 | write-only |
| 11 | read-write |

执行权限：

- 无 FEAT_XNX 时：`XN[1]==0` 才可能执行；
- 有 FEAT_XNX/TTS2UXN 时，QEMU 支持 4 态：
  - `0`：EL0/EL1 都可执行
  - `1`：仅 EL0 可执行
  - `2`：都不可执行
  - `3`：仅 EL1 可执行

这和 **DDI 0487 §D8.4.4** 所说的“如果某一级把区域判定为 execute-never，则该级产生 Permission fault”相一致。

### 4.3 Stage-1 + Stage-2 权限组合（DDI 0487 §D8.4.3）

规范非常明确：

1. 若 Stage-1 不允许，则产生 **Stage-1 Permission fault**；
2. 若 Stage-1 允许但 Stage-2 不允许，则产生 **Stage-2 Permission fault**；
3. 两级都允许，才允许访问。

也就是说，最终许可语义是 **交集**，但 fault reporting 有优先级。

QEMU 两个层面都体现了这个原则：

- `get_phys_addr_twostage()` 中：
  - `result->f.prot = s1_prot & result->s2prot;`
- `ptw.c` 中还有对 fault 顺序的专门注释：
  - alignment fault caused by memory type
  - permission fault
  - stage-2 fault on the memory access

因此，文档 49 中“**S1 和 S2 权限取交集**”这个结论是正确的，但若要完全符合 DDI 0487，还应补一句：

> “最终权限是交集；一旦 fault，需要按 `DDI 0487 §D8.4.3` / `§D8.15.5` 的优先级决定究竟报告 Stage-1 还是 Stage-2 Permission fault。”

### 4.4 执行权限并不等于数据权限

**DDI 0487 §D8.4.4** 明确：

- XN / UXN / PXN 只限制 instruction fetch，不直接阻止 data access；
- 若当前 EL 对该区域 execute-never，则 speculative fetch 也被禁止；
- 如果尝试执行，则在“第一个判定 execute-never 的 translation stage”产生 Permission fault。

QEMU `get_S1prot()` / `get_S2prot()` 的行为与此一致：

- 数据权限先由 AP/S2AP 得出；
- 执行权限再根据 XN/PXN/WXN/PAN 等额外裁剪；
- 对于 Security-space 不允许的 instruction fetch，只去掉 `PAGE_EXEC`，随后再以 permission fault 形式体现。

### 4.5 PAN / EPAN / WXN 属于 QEMU 对架构控制位的真实建模

虽然用户目标主要是 D8，但现有文档把以下点写出来是有价值的，而且与规范并不冲突：

- `PAN`：privileged data access 不能访问 user-permitted page
- `EPAN/PAN3`：如果 EL0 有 execute permission 也可能触发 PAN 限制
- `SCTLR_WXN`：写权限页面不可执行

这些不是“QEMU 私货”，而是 QEMU 在 `get_S1prot()` 里把架构规则软件化实现。

### 4.6 权限检查 ASCII 图

```text
Descriptor attrs
   |
   +--> AP / S2AP ----------> data R/W base perms
   |
   +--> XN / UXN / PXN -----> exec restriction
   |
   +--> APTable / XNTable --> hierarchical restriction
   |
   +--> PAN / EPAN / WXN ---> regime-level restriction
   |
   v
final S1 prot / final S2 prot
   |
   +--> final data perm = S1 ∩ S2
   +--> exec fault at first stage that says XN
```

### 4.7 本节结论

- 文档 11 / 38 / 49 对 `get_S1prot()`、`get_S2prot()` 的主叙述是可靠的；
- 最需要补充的是：
  - hierarchical permission 受 `HPD` 影响；
  - S1 与 S2 的关系不是“只取交集”这么简单，还涉及 **fault priority**；
  - execute-never fault 是在 **第一个宣告 XN 的 stage** 上报。

---

## 5. Fault 类型验证

### 5.1 架构定义的 MMU fault 类型只有 8 类（DDI 0487 §D8.15.1）

规范明确写道：

> All of the following MMU fault types are supported, and there are no other MMU fault types.

D8.15.1 架构化 MMU fault type 只有：

1. Alignment fault on a data access
2. Translation fault
3. Address size fault
4. Synchronous External abort on a translation table walk
5. Access flag fault
6. Permission fault
7. TLB conflict abort
8. Granule Protection Check fault (GPC fault)

因此，把 QEMU 的 `ARMFaultType` 枚举成员总数，直接说成“规范定义的 fault 类型”，是错误的。

### 5.2 QEMU 的 `ARMFaultType` 是“内部故障分类超集”

`target/arm/internals.h` 中 `ARMFaultType` 包含：

- `ARMFault_Translation`
- `ARMFault_AddressSize`
- `ARMFault_AccessFlag`
- `ARMFault_Permission`
- `ARMFault_Alignment`
- `ARMFault_SyncExternalOnWalk`
- `ARMFault_TLBConflict`
- `ARMFault_GPCFOnWalk`
- `ARMFault_GPCFOnOutput`

以及很多 **并不属于 D8.15.1 架构化 MMU fault 列表** 的项目，例如：

- `ARMFault_Debug`
- `ARMFault_AsyncExternal`
- `ARMFault_AsyncParity`
- `ARMFault_SyncParity`
- `ARMFault_Background`
- `ARMFault_Domain`
- `ARMFault_ICacheMaint`
- `ARMFault_QEMU_NSCExec`
- `ARMFault_QEMU_SFault`

所以正确表述应是：

> `ARMFaultType` 是 QEMU 为实现 ARM 异常/翻译路径而定义的 **软件内部 fault taxonomy**；其中只有一部分与 DDI 0487 §D8.15.1 的 MMU fault 一一对应。

### 5.3 Translation fault（DDI 0487 §D8.15.1.1）

规范定义了 Translation fault 的典型来源：

- descriptor `bit[0]==0`（invalid descriptor）
- 在不支持 block 的 lookup level 上出现 block encoding
- 地址不落在 TTBR0/TTBR1 配置的 VA range
- `EPD` 阻止对应 TTBR 的 page table walk
- 某些保留/非法的 stage-2 起始级别配置

QEMU 对应点：

- invalid descriptor / invalid block level → `goto do_translation_fault`
- `param.epd` → `goto do_translation_fault`
- gap between TTBR0/TTBR1 ranges → `goto do_translation_fault`
- `check_s2_mmu_setup()` 返回非法 → `goto do_translation_fault`

这部分与规范一致度很高。

### 5.4 Address size fault（DDI 0487 §D8.15.1.2）

规范定义：若以下地址含有超出配置 OA size 的非零高位，则产生 Address size fault：

- `TTBR_ELx` 基址
- translation table entry 中的 OA
- 最终 translation OA

QEMU 对应点：

- TTBR 基址越界：
  - `descaddr >> outputsize` → `ARMFault_AddressSize`
- descriptor 中 OA 越界：
  - `descaddr >> outputsize` → `ARMFault_AddressSize`
- `get_phys_addr_disabled()` 也会对物理直通地址做 address-size 检查。

### 5.5 Access flag fault（DDI 0487 §D8.5.1 / §D8.15.4）

规范顺序是：

- 若 AF hardware management 未实现或未启用，且 descriptor.Af = 0 → Access flag fault；
- 若 AF hardware management 启用，则硬件尝试更新 AF，这个更新本身可能引发 **stage-2 Permission fault** 或 **Synchronous External abort**。

QEMU 对应：

- `!(descriptor & (1 << 10)) && !param.ha` → `ARMFault_AccessFlag`
- `param.ha=1` 时会走 `arm_casq_ptw()` 做原子更新；
- 更新 descriptor 的原子写若失败/被重写，会重启处理流程。

这与规范非常吻合。

### 5.6 Permission fault（DDI 0487 §D8.4.3 / §D8.4.4 / §D8.15.4）

权限 fault 的来源包括：

- Stage-1 AP/PXN/UXN/WXN/PAN 不允许
- Stage-2 S2AP/XN 不允许
- Stage-1 page-table walk 落到 Stage-2 Device memory 且 `HCR_EL2.PTW=1`
- AF/dirty 状态硬件更新本身在 Stage-2 上不被允许
- Security-space rule 导致 instruction fetch 不允许

QEMU 中这些都统一落成 `ARMFault_Permission`，符合“架构 fault class 相同，但 fault origin 不同”的规范处理模式。

### 5.7 Alignment fault 的优先级必须放在“属性已知之后”

这点很重要。

**DDI 0487 §D8.15.4** 给出的 fault checking sequence 是：

- 先做一般 alignment check；
- 走 TTBR / descriptor / AF；
- 取出 OA 与 memory attributes 后；
- **再检查 output memory type 需要的 alignment**；
- 然后才是 OA space permission check。

QEMU 在 `get_phys_addr_lpae()` 中也明确注释：

```text
correct ordering:
- Alignment fault caused by the memory type
- Permission fault
- A stage 2 fault on the memory access
```

即：

- 只有先知道是 Device memory，才能决定是否强制 alignment fault；
- 因此这种 alignment check 不可能在最开始就做完。

文档 49 已提到 Alignment fault，但建议明确区分：

- **一般输入地址对齐 fault**
- **由最终 memory type（尤其 Device）触发的 alignment fault**

### 5.8 Fault priority（DDI 0487 §D8.15.5）

D8.15.5 明确给出单一 translation stage 下 fault 的优先顺序，前面几项尤其关键：

1. Stage-1 的 alignment fault（非 memory-type 引起）
2. Translation fault
3. TTBR address size fault
4. Stage-2 fault during Stage-1 table walk
5. ...
6. ...
7. ...
8. translation fault on level-1 entry
9. address size fault on level-1 entry
10. stage-2 fault on level-0 memory access during Stage-1 walk
...

而在最终 block/page 命中后，还要遵守 `D8.15.4` 给出的局部顺序：

- alignment fault caused by memory type
- permission fault
- stage-2 fault on the memory access

QEMU 在 `ptw.c` 用显式注释和代码位置维护了这一顺序，属于实现上很贴规范的一部分。

### 5.9 Spec fault vs QEMU fault 对照

| 规范 MMU fault | QEMU fault enum | 说明 |
|---|---|---|
| Translation fault | `ARMFault_Translation` | ✓ 直接对应 |
| Address size fault | `ARMFault_AddressSize` | ✓ |
| Access flag fault | `ARMFault_AccessFlag` | ✓ |
| Permission fault | `ARMFault_Permission` | ✓ |
| Alignment fault | `ARMFault_Alignment` | ✓ |
| Sync External abort on walk | `ARMFault_SyncExternalOnWalk` | ✓ |
| TLB conflict abort | `ARMFault_TLBConflict` | ✓ |
| GPC fault | `ARMFault_GPCFOnWalk/GPCFOnOutput` | QEMU 进一步区分 walk/output |
| 非 MMU fault（Debug/Async/Domain 等） | `ARMFault_Debug` 等 | ✗ 不是 D8.15.1 的 MMU fault 类型 |

### 5.10 本节结论

- 文档 11 / 49 对 fault 触发点分析总体正确；
- 但必须把“**QEMU 内部 fault enum**”与“**DDI 0487 架构化 MMU fault**”明确区分；
- 真正的架构 MMU fault 类型只有 D8.15.1 列出的 8 类。

---

## 6. 内存属性验证（DDI 0487 §D7 / §D8.6）

### 6.1 D7 的基本结论与文档 51 基本一致

**DDI 0487 §D7.1.2** 总结内存属性：

- **Device**：Outer Shareable，Non-cacheable
- **Normal**：可 Non-shareable / Inner Shareable / Outer Shareable，cacheability 可为 NC / WT / WB

这和文档 51 的基础分类一致。

### 6.2 Stage-1 AttrIndx -> MAIR_ELx（DDI 0487 §D8.6.1）

Stage-1 descriptor 中：

- 常规情况下 `AttrIndx[2:0]` 选 `MAIR_ELx.Attr<n>`；
- 若实现 FEAT_AIE，还可能通过 `AttrIndx[3]` 切换 `MAIR2_ELx`。

QEMU 代码：

```c
attrindx = extract32(attrs, 2, 3);
mair = (param.aie && extract64(attrs, 59, 1)
        ? env->cp15.mair2_el[el]
        : env->cp15.mair_el[el]);
result->cacheattrs.attrs = extract64(mair, attrindx * 8, 8);
```

因此：

- 文档 51 对 `AttrIndx -> MAIR_ELx` 的主描述是对的；
- 文档 30 把 `高4位=0 => Device，否则 Normal` 作为简化规则，也能覆盖绝大多数场景；
- 但若以 DDI 0487 全量语义计，还应补充 FEAT_AIE / FEAT_XS 的扩展边界。

### 6.3 Stage-1 shareability（DDI 0487 §D8.6.2）

规范规定：

- 对 **Normal cacheable memory**，`SH[1:0]` 编码 shareability：
  - `00` Non-shareable
  - `10` Outer Shareable
  - `11` Inner Shareable
- 若最终结果是 **Device** 或 **Normal Non-cacheable**，则 effective shareability 为 **Outer Shareable**。
- 若 `DS=1`，descriptor `bits[9:8]` 不再是 SH，而是 `OA[51:50]`；此时 shareability 来自 `TCR_ELx.SHx`。

QEMU `combine_cacheattrs()` 完全体现了“最终 Device 或 0x44（Normal NC）强制 Outer Shareable”的规则：

```c
if ((ret.attrs & 0xf0) == 0 || ret.attrs == 0x44) {
    ret.shareability = 2;
}
```

所以：

- 文档 51 “Device / NC 强制 Outer Shareable”是正确的；
- 文档 38 / 49 若把 `SH[9:8]` 写成始终存在的字段，则需要补上 `DS=1` 例外。

### 6.4 Stage-2 MemAttr（DDI 0487 §D8.6.5 / §D8.6.6）

Stage-2 与 Stage-1 最大不同点：

- Stage-1 通过 `AttrIndx -> MAIR_ELx` 间接取属性；
- Stage-2 用 descriptor 中的 `MemAttr[3:0]`（或 FWB enabled 下的重解释）直接编码。

#### 6.4.1 FWB disabled

**DDI 0487 §D8.6.5** 指出：

- `MemAttr[3:0] = 0000..0011` → 四种 Device subtype
- `0101` → Normal NC/NC
- `0110` → Outer NC + Inner WT
- `0111` → Outer NC + Inner WB
- `1000..1111` 还对应更多 WT/WB 组合

也就是说，**FWB disabled 的 Stage-2 MemAttr 并不是简单“2 bit cacheability”**，而是完整 4-bit 编码表。

#### 6.4.2 FWB enabled

**DDI 0487 §D8.6.6** 更复杂：

- `MemAttr[2]==0` 时，`MemAttr[1:0]` 用于 Device subtype；
- 其余编码与 Stage-1 memory type/cacheability 组合后，得到 resultant memory type；
- 是否保留 S1 cache allocation hint，也取决于 S1 原本是不是 cacheable。

QEMU `combined_attrs_fwb()` 的实现与规范精神一致：

- `s2.attrs == 7` → use stage-1 attrs
- `s2.attrs == 6` → force Normal WB
- `s2.attrs == 5` → Device 保留，否则强制 0x44（Normal NC）
- `s2.attrs in 0..3` → force Device subtype
- reserved → QEMU 退化成 Device

所以，文档 38 里把 FWB 写成：

```text
MemAttr[3:2] = 01 -> 强制 NC
MemAttr[3:2] = 10 -> 强制 WT
MemAttr[3:2] = 11 -> 强制 WB
MemAttr[3:2] = 00 -> 使用 S1 属性
```

这是 **不准确的过度简化**：

- 它没有体现 Device subtype；
- 没有体现 `s2.attrs==5/6/7` 与 S1 原始属性的组合逻辑；
- 也没有体现 FWB enabled 的结果仍然依赖 Stage-1 memory type。

### 6.5 QEMU 会“计算属性”，但不会“模拟 cache hierarchy”

这点现有文档 11 / 30 / 51 写得很好。

QEMU `ptw.c` 注释直说：

- 忽略 `SH/ORGN/IRGN` 对真实缓存行为的影响；
- 但仍把属性存入 `result->cacheattrs`；
- 用于 Device memory 判定、Stage-1/Stage-2 属性合并、TLB 元数据、部分 KVM/系统行为接口。

所以更准确的表述应是：

> QEMU **不模拟硬件 cache/shareability 行为本身**，但 **并不忽略属性元数据的计算与传播**。

这比“QEMU 忽略 cacheability/shareability”更完整。

### 6.6 MAIR 常见编码验证

文档 51 中以下常见编码与规范一致：

| 编码 | 含义 |
|---|---|
| `0x00` | Device-nGnRnE |
| `0x04` | Device-nGnRE |
| `0x08` | Device-nGRE |
| `0x0C` | Device-GRE |
| `0x44` | Normal Non-cacheable |
| `0xFF` | Normal Write-Back R/W Allocate |
| `0xF0` | Tagged Normal Write-Back（MTE 相关） |

注意：并非所有字节组合都可随意解释；规范中还存在 reserved / constrained unpredictable / FEAT_XS / FEAT_AIE 特例。

### 6.7 本节结论

- 文档 51 是五篇里与 D7/D8 一致度最高的一篇之一；
- 最需要修正的是：
  - 不能把 FWB enabled 的 `MemAttr` 讲成简单的 2-bit cache mode；
  - 不能忽略 `DS=1` 时 `SH` 位域挪用；
  - “QEMU 忽略属性”应改写为“QEMU 不模拟真实 cache 行为，但保留属性计算与传播”。

---

## 7. QEMU 实现差异

以下内容不是“规范错误”，而是 **QEMU 相对 DDI 0487 的有意简化或实现范围选择**。

### 7.1 不建模真实 ASID/VMID 精细化 TLB 行为

QEMU 注释明确说明：

- 不实现 ASID-like TLB tagging；
- `TCR.A1`、TTBR ASID 字段不会像硬件那样精确选择性命中；
- ASID 改变时往往直接 flush TLB。

这与真实硬件的 ASID/VMID 行为相比是简化。

### 7.2 不模拟 cacheability / shareability 对访存次序和缓存层次的真实影响

QEMU 会计算 `cacheattrs`，但不会像硬件一样：

- 创建真正的 Inner/Outer cache hierarchy；
- 区分 coherent / non-coherent transaction；
- 依据 shareability 域做真实缓存一致性协议；
- 让绝大多数 cache maintenance 指令产生物理 cache 效果。

这点和 D7 的 system-level memory model 是显著差异。

### 7.3 `ARMFaultType` 是实现超集，不是架构 fault list

规范只定义 8 类 MMU fault；QEMU 还把异步异常、debug、M-profile 专用项等放在统一 fault enum 中。这是工程实现上的方便性抽象。

### 7.4 对 IMPLEMENTATION DEFINED 行为作了固定选择

例如 `TxSZ` 越界时，规范允许 implementation-defined 行为；QEMU 选择：

- 直接按 Translation fault 路径处理（且注释说明这是明确选择）。

这类点应被看作“QEMU 选定了一种允许的实现行为”，不是架构冲突。

### 7.5 主要覆盖的是 VMSAv8-64 PTW 主路径

**DDI 0487 §D8.1** 还讨论 VMSAv9-128 translation system；但本组文档与 QEMU `ptw.c` 现有主分析，实际焦点仍是 **VMSAv8-64 64-bit descriptor** 路径。

因此若文档写“完整覆盖 D8”，就会过头；更准确的说法应是：

> “完整分析了 QEMU 当前主实现所覆盖的 VMSAv8-64 PTW 路径，并对 D8 中相关规则进行了交叉验证。”

### 7.6 Cache maintenance 在 TCG 下大多为 NOP

文档 51 已指出：

- 大多数 `IC` / `DC` cache maintenance 指令在 QEMU TCG 中是 NOP；
- `DC ZVA` 等少数指令有真实语义。

这与规范允许的架构语义并不矛盾，但说明 QEMU 并不试图完整实现硬件 cache subsystem。

---

## 8. 既有文档勘误（✓/✗ 对照）

> 说明：下表的 ✓/✗ 评价标准不是“这篇文档是否有价值”，而是“该具体表述是否能直接对齐 DDI 0487 D7/D8”。很多 ✗ 实际上只是“常见路径成立，但缺边界条件”。

| 文档 | 原表述/主张 | 结论 | 说明 |
|---|---|---:|---|
| 11 | “支持 4KB/16KB/64KB 粒度，0-4 级页表” | ✗ | DDI 0487 §D8.2.8 说明 4KB granule 在 DS=1 时可到 `level -1`，不能仅概括为固定 0~4 级。 |
| 11 | “QEMU 忽略 shareability 和 cacheability 属性” | ✓ | 若理解为“不模拟真实缓存行为”则正确；但更完整表述应补充“仍保留 `cacheattrs` 计算与传播”。 |
| 30 | “EL1&0 和 VHE EL2&0 有两段地址空间，EL2/EL3 只有 TTBR0” | ✓ | 与 DDI 0487 §D8.2.4 和 QEMU `regime_has_2_ranges()` 一致。 |
| 30 | “QEMU 支持 4KB/16KB/64KB 页粒度和最多 52 位物理地址” | ✓ | 对 VMSAv8-64 + LPA/LPA2 范围内的 QEMU 主实现成立。 |
| 30 | 用固定图示表达 `descriptor[9:8]=SH` | ✗ | DDI 0487 §D8.6.2 / §D8.6.7：DS=1 时 `bits[9:8]` 为 `OA[51:50]`，SH 来自 `TCR/VTCR.SHx`。 |
| 38 | “无 FWB：取更强约束” | ✓ | 若“更强约束”理解为“更严格/更弱 cacheability 结果”则基本正确。 |
| 38 | “FWB: MemAttr[3:2] = 01/10/11 -> NC/WT/WB，00 -> 用 S1” | ✗ | 与 DDI 0487 §D8.6.6 及 QEMU `combined_attrs_fwb()` 都不完全一致，忽略 Device subtype 与 S1 属性依赖。 |
| 38 | “Device 内存总是 Outer Shareable” | ✓ | 对最终 resultant memory type 成立，符合 DDI 0487 §D8.6.2 / §D8.6.7 与 QEMU `combine_cacheattrs()`。 |
| 49 | “ARMFaultType 24 种故障类型 ...”若被理解为架构 MMU fault 列表 | ✗ | DDI 0487 §D8.15.1 明确架构 MMU fault 只有 8 类；QEMU enum 是实现超集。 |
| 49 | “S1 和 S2 权限取交集” | ✓ | 作为最终权限结果正确；但还应补充 Stage-1/Stage-2 Permission fault 的优先级。 |
| 49 | “合并页大小 = MAX(s1_lgpgsz, s2_lgpgsz)” | ✗ | QEMU 实际有 `TARGET_PAGE_BITS` 特判；并不是无条件 MAX。 |
| 49 | 描述符下层属性固定含 `SH[9:8]` | ✗ | 同样遗漏 `DS=1` 时 `OA[51:50]` 重用位域。 |
| 51 | “AttrIndx -> MAIR_ELx 取 8-bit 属性” | ✓ | 与 DDI 0487 §D8.6.1 及 QEMU 实现一致。 |
| 51 | “Device / Normal NC 最终视为 Outer Shareable” | ✓ | 与 DDI 0487 §D8.6.2 / §D8.6.7 一致。 |
| 51 | “QEMU 不模拟缓存，因此大多数 cache maintenance 为 NOP” | ✓ | 这是 QEMU 实现差异说明，不与规范冲突。 |

### 8.1 建议统一改写的几类表述

建议后续文档统一采用下面的写法，避免误导：

#### 建议改写 1：翻译体制

原式：

```text
ARM64 有 EL1&0、EL2、EL2&0、EL3 四种翻译体制
```

建议：

```text
QEMU 代码层面主要抽象为 EL1&0、EL2、EL2&0、EL3 四类 MMU index；
而 DDI 0487 架构层面的 translation regime 还区分 Security state / Realm / Root。
```

#### 建议改写 2：页表字段

原式：

```text
SH[9:8] 永远表示共享属性
```

建议：

```text
在 DS=0 的常见 VMSAv8-64 格式中，descriptor[9:8] 表示 SH；
在 DS=1 时，这两位被重定义为 OA[51:50]，shareability 改由 TCR/VTCR.SHx 提供。
```

#### 建议改写 3：Fault 类型

原式：

```text
ARMFaultType 列出了 ARM 规范的所有 MMU fault
```

建议：

```text
ARMFaultType 是 QEMU 内部 fault taxonomy；其中与 DDI 0487 §D8.15.1 对应的 MMU fault 只有一部分。
```

#### 建议改写 4：FWB

原式：

```text
FWB 就是 S2 直接指定 NC/WT/WB
```

建议：

```text
FWB enabled 后，Stage-2 MemAttr 的解释与 resultant memory type 仍然依赖 S1 memory type，
并包含 Device subtype 与 force-NC/force-WB 等分支，不能简化为单一 2-bit cache mode 表。
```

---

## 9. 总结

本次交叉验证的最终判断如下：

### 9.1 可以保留为“正确结论”的部分

- `TCR_ELx` / `TTBR_ELx` / `VTCR_EL2` / `VTTBR_EL2` 对页表遍历的控制关系；
- `aa64_va_parameters()`、`get_phys_addr_lpae()`、`get_phys_addr_twostage()` 的主流程；
- Stage-1/Stage-2 权限的 direct-permission 解释；
- `AttrIndx -> MAIR_ELx`、`MemAttr[3:0]`、`combine_cacheattrs()` 的大方向；
- QEMU 对 ASID / cacheability / shareability / cache maintenance 的简化说明。

### 9.2 必须补充的边界条件

- `translation regime` 受 Security state / Realm / Root 影响；
- EL2 使能后，EL1&0 Stage-1 的页表基址语义是 **IPA**；
- 4KB granule 在 DS=1 时可能出现 `level -1`；
- Stage-2 可能使用 concatenated translation tables；
- `DS=1` 时 descriptor `bits[9:8]` 不再表示 SH；
- fault reporting 需要同时考虑 **最终交集权限** 与 **fault priority**。

### 9.3 明确需要勘误的地方

- 不能把 `ARMFaultType` 直接当成 DDI 0487 架构 MMU fault 列表；
- 不能把 FWB 描述成“`MemAttr[3:2]` 直接映射成 NC/WT/WB”；
- 不能把所有 descriptor 图都画成固定 `SH[9:8]` 格式；
- 不能把页表级数永久写死为 L0~L3。

### 9.4 一句话总结

**现有文档对 QEMU ARM64 VMSAv8-64 PTW 主实现的解释总体是可信的；问题主要不在“主线错误”，而在“把常见实现路径说成了完整架构规则”。** 后续若要把这些文档升级为“规范级”参考资料，应以本文列出的 DDI 0487 §D8.x 勘误点为准补齐边界条件。
