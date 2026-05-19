# ARM64 Cache 一致性与 TLB 广播规范验证

> QEMU 11.0.50 vs ARM DDI 0487 M.b — DC/IC/TLBI/PoC/PoU/多核一致性对照

## 1. 验证概述

本文对 QEMU 11.0.50 的 AArch64 TCG 实现进行“规范语义 vs 模拟语义”对照，重点检查：

- Cache 维护指令：`DC *` / `IC *`
- TLB 维护指令：`TLBI *`
- PoC / PoU / PoP / PoDP 抽象点
- 多核 cache 一致性与广播
- 自修改代码 / JIT / DMA / 持久化内存场景

对照规范基线：

- **DDI 0487 M.b §B2.4**：PoC / PoU / 一致性点 / cache 维护的可见性语义
- **DDI 0487 M.b §D8.14**：`DC` / `IC` 指令语义
- **DDI 0487 M.b §D8.10**：`TLBI` 指令语义

### 1.1 总结结论

QEMU 在这里的**最大架构性简化**确实是：**没有真实的 cache hierarchy**。它不是“近似真实 cache”，而是把大多数 cache 语义降格成：

1. **数据 cache 维护基本不建模**：除 `DC ZVA` / `DC GZVA` 以及 `DC CVAP` / `DC CVADP` 的有限特例外，大多数 `DC` 指令在系统模拟里是 NOP（`target/arm/helper.c:3484-3541`, `target/arm/helper.c:5350-5363`）。
2. **指令 cache 维护在 system mode 基本不建模**：`IC IALLU` / `IC IALLUIS` / `IC IVAU` 在 system emulation 下都是 NOP；只有 user mode 下 `IC IVAU` 会主动失效 TB（`target/arm/helper.c:3416-3441`, `target/arm/helper.c:3484-3507`）。
3. **TLBI 必须真实工作**：QEMU 的 softmmu TLB 必须刷新，因此 `TLBI` 实现相对完整；但它经常**过度失效**、**忽略 ASID/leaf-only/outer-shareable 区分**，并把很多变体折叠为同一路径（`target/arm/tcg/tlb-insns.c:317-523`, `target/arm/tcg/tlb-insns.c:843-1015`, `accel/tcg/cputlb.c:663-845`）。
4. **多核一致性天然成立**：所有 vCPU 共享同一份 host RAM，没有私有 D-cache / LLC / snoop 过滤器，因此很多硬件上会失败的 guest 程序，在 QEMU 上会“意外正确”。

### 1.2 对 Guest 的核心影响

这类简化对大多数普通 guest 软件透明，因为：

- 大多数应用并不显式发 `DC` / `IC` / `TLBI`；
- Linux/固件只要不依赖 cache 脏数据丢失、非一致 DMA、精确 TLB 广播延迟，功能往往都能跑；
- QEMU 还通过 TB 失效机制，额外帮 guest 掩盖了很多 I-cache 一致性问题（`system/physmem.c:3142-3145`, `accel/tcg/tb-maint.c:1183-1206`）。

但从**规范验证**角度，这正是差异来源：

- QEMU **不适合验证**“cache 维护缺失是否会出错”；
- QEMU **不适合验证**“non-coherent DMA 驱动是否正确执行 cache clean/invalidate”；
- QEMU **不适合验证**“自修改代码是否真的满足 `DC CVAU -> DSB -> IC IVAU -> DSB -> ISB` 序列”；
- QEMU **不适合验证**“TLBI 是否只影响某个 ASID / 某个 leaf / 某个 shareability 域”。

---

## 2. Cache 维护操作 (DC 指令)

## 2.1 QEMU 的总体策略：权限/陷入语义保留，cache 数据语义大多丢弃

QEMU 对 `DC` 指令的实现不是“完全忽略”。更准确地说：

- **访问控制仍然检查**：`aa64_cacheop_poc_access()` / `access_tocu()` / `aa64_zva_access()` 仍实现了 EL0/EL1/EL2 访问权限和 trap 条件，比如 `SCTLR_EL1.UCI`、`HCR_EL2.TPCP`、`HCR_EL2.TOCU`、`HCR_EL2.TDZ` 等（`target/arm/helper.c:3145-3225`）。
- **真正的数据 cache 行状态不建模**：没有 dirty line、clean line、exclusive/shared line、eviction、writeback buffer 等对象。

这意味着：**异常路径更接近硬件，数据路径却远比硬件简单**。

## 2.2 `DC CIVAC / CVAC / CVAU / IVAC / CISW / CSW / ISW`：系统模拟里基本都是 NOP

QEMU 在 AArch64 cpreg 表中直接把下列指令注册为 `ARM_CP_NOP`：

- `DC_IVAC`
- `DC_ISW`
- `DC_CVAC`
- `DC_CSW`
- `DC_CVAU`
- `DC_CIVAC`
- `DC_CISW`

对应源码见 `target/arm/helper.c:3510-3541`。

### 2.2.1 与规范的差异

按 DDI 0487 M.b：

- `DC CVAC` / `CIVAC`：要求把脏数据清到 **PoC**；
- `DC CVAU`：要求把数据清到 **PoU**；
- `DC IVAC`：要求以 **PoC** 为边界失效数据 cache 行；若该行是脏的且未写回，**可能导致数据丢失**；
- `DC CISW / CSW / ISW`：按 set/way 对 cache 层级做清洗/失效，常用于低功耗、secure switch、cache 关闭等管理路径。

而在 QEMU：

- 没有数据 cache 层级；
- 没有“脏而未回写”的行；
- 没有 set/way 结构；
- 没有 PoC/PoU 不同位置上的缓存副本。

因此这些指令**只能退化为“检查权限后无副作用”**。

### 2.2.2 最关键的可观察差异：`DC IVAC` 不会丢数据

这是最重要的 guest 可观察差异之一。

在真实硬件上，`DC IVAC` 的语义不是“从内存重新加载”，而是“丢弃 cache 行”。如果这行里有尚未 clean 到 PoC 的脏数据，那么 invalidate 以后，**新写入的数据可能被丢掉**。这正是许多 DMA/驱动/裸机代码必须先 `clean` 再 `invalidate` 的原因。

在 QEMU 上：

- guest store 直接写入 host RAM；
- 没有“只存在于 D-cache、尚未写回内存”的副本；
- `DC IVAC` 又是 NOP。

所以 guest 即使错误地把 `DC IVAC` 当成“刷新最新数据”的工具，也**不会暴露出真实硬件上的数据丢失风险**。

### 2.2.3 对 DMA 驱动验证的直接影响

这会直接掩盖以下真实 bug：

1. **CPU 写 buffer 给设备读**，驱动漏掉 `DC CVAC/CIVAC`：
   - 真机：设备可能读到旧数据；
   - QEMU：设备通常直接看到 host RAM 中的新数据。
2. **设备写 buffer 给 CPU 读**，驱动漏掉 `DC IVAC`：
   - 真机（non-coherent DMA）：CPU 可能继续从 D-cache 读旧值；
   - QEMU：CPU 直接读到内存中的新值。
3. **错误使用 `DC IVAC` 覆盖未 clean 的脏数据**：
   - 真机：可能丢数据；
   - QEMU：不会丢。

所以：**QEMU 无法有效检验 non-coherent DMA 驱动的 cache maintenance 正确性**。

## 2.3 `DC ZVA`：QEMU 会真正清零内存，而不是 NOP

`DC ZVA` 是少数被真正实现的 `DC` 指令。翻译时，`translate-a64.c` 对 `ARM_CP_DC_ZVA` 调用 `gen_helper_dc_zva()`（`target/arm/tcg/translate-a64.c:2996-3012`）；helper 中实际执行 `memset(mem, 0, blocklen)`（`target/arm/tcg/helper-a64.c:793-835`）。

### 2.3.1 这与规范在哪些点一致

QEMU 至少保留了以下规范要点：

- 使用 `DCZID_EL0.BS` 决定清零块大小，helper 里 `blocklen = 4 << get_dczid_bs(...)`（`target/arm/tcg/helper-a64.c:799-800`, `target/arm/cpu.h:1188-1197`）；
- 地址会对齐到块边界（`target/arm/tcg/helper-a64.c:799-800`）；
- 对普通内存会真正写零；
- 对 I/O 情况会退化成逐字节写（`target/arm/tcg/helper-a64.c:821-828`）；
- 访问权限由 `aa64_zva_access()` 和 `DCZID_EL0.DZP` 控制（`target/arm/helper.c:3199-3239`）。

### 2.3.2 但仍与真实硬件不同

helper 的注释明确说明：QEMU **不实现** `DC ZVA` 对 Device memory 的对齐 fault / attribute 相关行为，且“匹配 QEMU 一贯不实现 alignment fault 和 memory attribute handling 的风格”（`target/arm/tcg/helper-a64.c:793-797`）。

因此：

- 真机上某些 `DC ZVA` 使用方式可能 fault；
- QEMU 上可能直接清零成功。

### 2.3.3 `DC GZVA` / MTE 路径

`translate-a64.c` 对 `DC_GZVA` 的处理是：

1. 调 `dc_zva` 清零数据块；
2. 若 MTE tag access 激活，再写 tag（`target/arm/tcg/translate-a64.c:3033-3048`）。

因此在 QEMU 里，`DC GZVA` 的语义更像：**“数据块清零 + tag helper 更新”**，而不是围绕真实 cache/tag array 的微架构操作。

## 2.4 `DC CVAP / CVADP`：并非纯 NOP，但只提供非常弱的持久化近似

QEMU 为 `DC_CVAP` / `DC_CVADP` 专门提供了 `dccvap_writefn()`，并在 cpreg 中绑定到 `.writefn = dccvap_writefn`（`target/arm/helper.c:5315-5347`, `target/arm/helper.c:5349-5363`）。

该 helper 做的事情是：

1. 读取 `CTR_EL0.DminLine` 算出 cache line 大小（`target/arm/helper.c:5319-5323`）；
2. 对输入地址按 line 对齐（`target/arm/helper.c:5321-5324`）；
3. 找到 host 地址与对应 `MemoryRegion`；
4. 调 `memory_region_writeback()`（`target/arm/helper.c:5332-5340`）。

`memory_region_writeback()` 本身又只是：

- 若 `mr->dirty_log_mask` 非零，则调用 `memory_region_msync()`；
- 否则不做任何事（`include/system/memory.h:2040-2047`, `system/memory.c:2400-2408`）。

### 2.4.1 这与规范的距离

按规范：

- `DC CVAP` 应 clean 到 **Point of Persistence**；
- `DC CVADP` 应 clean 到 **Point of Deep Persistence**。

QEMU 的语义更接近：

- “如果底层 RAM block 当前打开了 dirty logging，就对 host backing 做一次 `msync` 风格 writeback；否则什么也不保证。”

所以它并没有建立真实的：

- PoP / PoDP 分层；
- ADR/eADR / NVDIMM 持久化队列；
- 与 `DSB`/powerfail model 联动的 durability ordering。

### 2.4.2 实际判断

因此对持久化内存 guest 来说：

- `DC CVAP/CVADP` 在 QEMU 上**比普通 DC NOP 更强一点**；
- 但它仍不足以验证真实硬件上的 persistence correctness；
- 特别是“断电后哪些写真正持久化”的问题，QEMU 几乎没有可比性。

---

## 3. Instruction Cache 维护 (IC 指令)

## 3.1 system emulation：`IC IALLU / IALLUIS / IVAU` 全部不建模为真实 I-cache

QEMU 在 `target/arm/helper.c` 中写得非常直接：

> Instruction cache ops. All of these except `IC IVAU` NOP because we don't emulate caches.

见 `target/arm/helper.c:3484-3486`。

具体注册结果：

- `IC_IALLUIS` -> `ARM_CP_NOP`（`target/arm/helper.c:3487-3491`）
- `IC_IALLU` -> `ARM_CP_NOP`（`target/arm/helper.c:3492-3496`）
- `IC_IVAU` ->
  - `CONFIG_USER_ONLY` 下用 `ic_ivau_write()`；
  - system emulation 下 `ARM_CP_NOP`（`target/arm/helper.c:3497-3507`）

这意味着：**在 qemu-system-aarch64 的 TCG system mode 中，guest 执行的 IC 维护指令本身并不驱动 TB 失效。**

## 3.2 那为什么自修改代码通常还是能工作？

因为 system mode 里另有一条更强的捷径：**物理内存被写脏时，QEMU 会直接使对应 TB 失效**。

关键路径：

- `system/physmem.c` 中，当 dirty mask 包含 `DIRTY_MEMORY_CODE` 时，调用 `tb_invalidate_phys_range(NULL, addr, addr + length - 1)`（`system/physmem.c:3142-3145`）；
- `tb_invalidate_phys_range()` 遍历覆盖范围内的 TB，并执行 `tb_phys_invalidate__locked(tb)`（`accel/tcg/tb-maint.c:1183-1206`, `accel/tcg/tb-maint.c:1130-1160`）。

也就是说，**QEMU 依赖“代码页被写了”这一事实来失效翻译块，而不是依赖 guest 明确执行 `IC IVAU`。**

### 3.2.1 这与真实硬件的根本不同

在真机上：

- D-cache 与 I-cache 不一定自动保持 data-to-instruction coherence；
- 写代码到 D-cache 后，I-cache 可能仍执行旧指令；
- 所以需要 `DC CVAU -> DSB ISH -> IC IVAU -> DSB ISH -> ISB`。

在 QEMU system mode：

- 代码写入 RAM 通常直接导致 TB 被废弃；
- 下一次执行会重新翻译，天然看到新字节；
- 即使 guest 漏掉 `IC IVAU`，也常常照样工作。

**这会显著高估 guest 自修改代码/JIT 的可移植性。**

## 3.3 user mode：`IC IVAU` 被专门实现为 TB 范围失效

在 `CONFIG_USER_ONLY` 下，QEMU 对 `IC IVAU` 提供了特殊实现，注释明确说明这是为了兼容“W^X 下双映射 JIT”场景：写映射不可执行、执行映射不可写，导致 QEMU 无法从“执行页被写”自动探测代码变化（`target/arm/helper.c:3416-3423`）。

`ic_ivau_write()` 的逻辑是：

- 从 `CTR_EL0` 取 I-cache line 大小；
- 把传入地址按 line 对齐成 `[start_address, end_address]`；
- 调 `tb_invalidate_phys_range(env_cpu(env), start_address, end_address)`（`target/arm/helper.c:3424-3441`）。

与此同时，`target/arm/cpu.c` 还在 user mode 特意**清掉** `CTR_EL0.DIC`，让遵守这些位的软件/JIT 不敢假设“无需执行 I-cache invalidate”（`target/arm/cpu.c:1881-1890`）。

### 3.3.1 这里的语义仍不是“真实 I-cache”

即使在 user mode：

- `IC IVAU` 触发的是 TB 失效；
- 不是模拟某个 PoU 上的 I-cache 行失效；
- 没有 I-cache capacity / alias / VIPT/PIPT 冲突模型。

所以它是**功能兼容实现**，不是**微架构兼容实现**。

## 3.4 `IC IALLUIS` 的多核广播在 system mode 中并不存在

规范里 `IC IALLUIS` 具有 inner-shareable 范围上的广播/可见性含义；但在 QEMU system mode 中它就是 NOP（`target/arm/helper.c:3487-3491`）。

所以：

- QEMU 不建模 I-cache invalidate 广播；
- 更准确地说，QEMU 通过“共享 RAM + TB 被写时直接失效”绕开了这个问题；
- 这和“真实硬件上通过 shareability 域传播 I-cache maintenance”不是一回事。

---

## 4. TLB 维护操作 (TLBI 指令)

TLBI 是 QEMU 在本主题中**实现最认真**的一部分，因为 softmmu TLB 必须正确刷新，否则 guest 页表修改根本无法工作。

但“能工作”不等于“与规范一一对应”。QEMU 在这里的主要特征是：

- 支持大量 TLBI opcode；
- 对 IS 广播做跨 vCPU 刷新；
- 但经常**扩大失效范围**；
- 经常**忽略 ASID / last-level-only / inner vs outer shareable 区别**；
- 某些 OS 变体直接折叠到 IS，个别变体甚至注册成 NOP。

## 4.1 广播语义：IS 变体确实会跨 vCPU 刷新

`target/arm/tcg/tlb-insns.c` 开头直接写明：

- “IS variants of TLB operations must affect all cores”（`target/arm/tcg/tlb-insns.c:49-56`）。

实现方式：

- `TLBIALLIS` / `VMALLE1IS` 等使用 `tlb_flush_all_cpus_synced()` 或 `tlb_flush_by_mmuidx_all_cpus_synced()`（`target/arm/tcg/tlb-insns.c:49-80`, `target/arm/tcg/tlb-insns.c:317-400`）；
- `VAE1IS` / `IPAS2E1IS` 等使用 `tlb_flush_page*_all_cpus_synced()`（`target/arm/tcg/tlb-insns.c:434-523`）；
- range 版本用 `tlb_flush_range_by_mmuidx_all_cpus_synced()`（`target/arm/tcg/tlb-insns.c:888-1012`）。

底层 `cputlb.c` 中：

- `tlb_flush_by_mmuidx_all_cpus_synced()` 会先对其他 CPU `async_run_on_cpu()`，再对当前 CPU `async_safe_run_on_cpu()`（`accel/tcg/cputlb.c:423-435`）；
- 注释说明 “safe work + loop exit” 会形成同步点，保证恢复执行前这些工作都完成（`accel/tcg/cputlb.c:350-367`，`cpu-common.c:327-338`）。

### 判断

所以对于“**IS 需要广播到其他 PE**”这一点，QEMU 的功能语义是存在的；但它是：

- **软件队列 + 线程同步点**；
- 不是硬件 snoop/filter/fabric 上的传播；
- 没有传播延迟、拥塞、局部可见窗口等微架构现象。

## 4.2 非 IS 变体：可能只刷本 CPU，也可能被 `HCR_EL2.FB` 升级成广播

`tlb_force_broadcast()` 明确实现了：若当前在 EL1 且 `HCR_EL2.FB` 生效，则把本应本地的 TLBI 提升成全 CPU 广播（`target/arm/tcg/tlb-insns.c:82-90`）。

例如：

- `TLBI_VMALLE1`：本地 `tlb_flush_by_mmuidx()`，但 `HCR_FB` 时升为 `all_cpus_synced()`（`target/arm/tcg/tlb-insns.c:326-337`）；
- `TLBI_VAE1` / `IPAS2E1` 也有同样逻辑（`target/arm/tcg/tlb-insns.c:445-463`, `target/arm/tcg/tlb-insns.c:501-513`）。

这与架构“FB 强制 broadcast”的方向一致，但仍是**软件级同步**。

## 4.3 `VMALLE1 / VMALLE1IS / VMALLE1OS`：基本支持，但 shareability 域被压扁

- `VMALLE1IS` -> `tlbi_aa64_vmalle1is_write()` -> `tlb_flush_by_mmuidx_all_cpus_synced(mask)`（`target/arm/tcg/tlb-insns.c:317-324`, `target/arm/tcg/tlb-insns.c:634-639`）
- `VMALLE1` -> `tlbi_aa64_vmalle1_write()`（`target/arm/tcg/tlb-insns.c:326-337`, `target/arm/tcg/tlb-insns.c:670-675`）
- `VMALLE1OS` 直接绑定到 **同一个** `vmalle1is_write`（`target/arm/tcg/tlb-insns.c:1160-1165`）

这意味着：

- QEMU **不区分 Inner Shareable vs Outer Shareable** 的实际硬件传播域；
- `OS` 变体被折叠成与 `IS` 相同的“全 vCPU 同步刷新”。

对功能正确性通常无害，但从规范一致性看，它显然比真实硬件更粗糙。

## 4.4 `VAE1 / VAE1IS`：支持按 VA 刷，但经常不是“规范级精确”

实现：

- 提取页地址 `pageaddr = sextract64(value << 12, 0, 56)`；
- 结合 regime/TBI 算出有效位宽 `bits`；
- 调 `tlb_flush_page_bits_by_mmuidx(...)` 或全 CPU 版本（`target/arm/tcg/tlb-insns.c:434-463`）。

这说明 QEMU 至少做了：

- VA 级别失效；
- 考虑不同 MMU regime；
- 考虑 TBI 对地址匹配位宽的影响（`target/arm/tcg/tlb-insns.c:270-315`）。

### 4.4.1 但它把多个 opcode 合并处理

源码注释明确写明：`tlbi_aa64_vae1_write()` “Currently handles all of VAE1, VAAE1, VAALE1 and VALE1, since we don't support flush-for-specific-ASID-only or flush-last-level-only.”（`target/arm/tcg/tlb-insns.c:448-453`）。

也就是说：

- `VAE1`、`VAAE1`、`VAALE1`、`VALE1` 在 QEMU 里被合并；
- **ASID-only** 限定被丢弃；
- **last-level-only** 限定被丢弃。

这对 guest 的直接影响是：

- QEMU 会**过度失效**；
- 真机上只影响某 ASID / 某层 leaf 的场景，在 QEMU 上可能退化成“更多条目都没了”；
- 这通常不会导致功能错误，但会掩盖 guest 对 TLBI 精度的依赖。

## 4.5 `ASIDE1 / ASIDE1IS / ASIDE1OS`：ASID 语义实际上被抹掉了

这是最明显的 TLBI 规范差异之一。

在 cpreg 注册中：

- `TLBI_ASIDE1IS` 直接绑定 `tlbi_aa64_vmalle1is_write`（`target/arm/tcg/tlb-insns.c:646-651`）；
- `TLBI_ASIDE1` 直接绑定 `tlbi_aa64_vmalle1_write`（`target/arm/tcg/tlb-insns.c:682-687`）；
- `TLBI_ASIDE1OS` 也绑定 `tlbi_aa64_vmalle1is_write`（`target/arm/tcg/tlb-insns.c:1172-1177`）。

也就是说：**ASID 字段根本没有被用来缩小失效范围**，而是直接退化成“整个 EL1&0 regime 全刷”。

### 判断

- 真机：`ASIDE1*` 只应影响特定 ASID；
- QEMU：直接接近 `VMALLE1*`。

这会让依赖“ASID 局部失效”的 guest 看起来也能工作，但验证不到以下问题：

- ASID 复用前是否做了足够精确的 invalidate；
- 多地址空间切换时，是否错误保留了旧 ASID 的 TLB 项；
- 软件是否过度依赖“只刷某 ASID 而不刷别人”。

## 4.6 `VMALLS12E1 / VMALLS12E1IS / VMALLS12E1OS`：退化成 `ALLE1*`

注册表显示：

- `TLBI_VMALLS12E1IS` -> `tlbi_aa64_alle1is_write`（`target/arm/tcg/tlb-insns.c:718-721`）
- `TLBI_VMALLS12E1` -> `tlbi_aa64_alle1is_write`（注意这里甚至不是 `alle1_write`，而是 IS handler，`target/arm/tcg/tlb-insns.c:734-737`）
- `TLBI_VMALLS12E1OS` -> `tlbi_aa64_alle1is_write`（`target/arm/tcg/tlb-insns.c:1216-1219`）

而 `alle1_tlbmask()` 明确包含：

- EL1 stage-1 TLB 项；
- `Stage2` 和 `Stage2_S`（`target/arm/helper.c:404-420`）。

因此 QEMU 的实现语义是：

- 确实会把 stage1 + stage2 一起刷掉；
- 但 shareability / 非 shareability / opcode 微差异被压平；
- `VMALLS12E1` 非 IS 版本也被直接映射到广播型 handler。

这比规范要求更“猛”。

## 4.7 `IPAS2E1 / IPAS2E1IS`：支持按 IPA stage-2 刷，但 OS 变体并不完整

`tlbi_aa64_ipas2e1_write()` / `tlbi_aa64_ipas2e1is_write()`：

- 提取 IPA 对应页地址；
- 根据 NS/SEL2 状态选择 `Stage2` 或 `Stage2_S` mask；
- 对应调用本地或全 CPU 的 `tlb_flush_page_by_mmuidx*`（`target/arm/tcg/tlb-insns.c:488-523`）。

所以 `IPAS2E1` / `IPAS2E1IS` 本身是有实现的。

但在 `tlbios_reginfo` 中：

- `TLBI_IPAS2E1OS`
- `TLBI_RIPAS2E1OS`
- `TLBI_IPAS2LE1OS`
- `TLBI_RIPAS2LE1OS`

都被注册为 `ARM_CP_NOP`（`target/arm/tcg/tlb-insns.c:1220-1231`）。

### 判断

因此对 FEAT 下的 Outer Shareable IPA TLBI 来说，QEMU 这一版并不是“统一建模成 IS”，而是**有些 OS 变体根本没做**。这是一个需要明确标出的规范偏差。

## 4.8 FEAT_TLBIRANGE：支持 range，但仍有多处折叠和过度失效

QEMU 实现了 range TLBI：

- `tlbi_aa64_get_range()` 解析 granule、num、scale、base（`target/arm/tcg/tlb-insns.c:843-885`）；
- `do_rvae_write()` 调 `tlb_flush_range_by_mmuidx*`（`target/arm/tcg/tlb-insns.c:888-907`）；
- `RVAE1/RVAE2/RVAE3/RIPAS2E1` 等 range 指令均接入这一路径（`target/arm/tcg/tlb-insns.c:910-1012`）。

### 4.8.1 但语义仍被压平

注释再次明确写出：

- `RVAE1` 同时处理 `RVAE1/RVAAE1/RVAALE1/RVALE1`（`target/arm/tcg/tlb-insns.c:914-919`）；
- `RVAE1IS` 同时处理 `IS/OS` 以及 `AA/LE` 变体，因为“不支持 specific-ASID-only、last-level-only、inner/outer specific flushes”（`target/arm/tcg/tlb-insns.c:929-937`）。

也就是说：

- **range 支持存在**；
- 但**精度比规范弱得多**。

### 4.8.2 底层还会继续“退化成更粗的 flush”

`accel/tcg/cputlb.c` 的 `tlb_flush_range_locked()` / `tlb_flush_range_by_mmuidx()` 还会根据范围大小和匹配位宽，进一步退化：

- 若 `bits < TARGET_PAGE_BITS`，直接退化成 `tlb_flush_by_mmuidx()`，即全 flush（`accel/tcg/cputlb.c:774-777`, `accel/tcg/cputlb.c:812-815`）；
- 若范围足够小且所有位都显著，则退化成 page flush（`accel/tcg/cputlb.c:780-785`, `accel/tcg/cputlb.c:817-823`）；
- 若 mask/len 不合适，或遇到 large page，也可能强制 full flush（`accel/tcg/cputlb.c:671-699`）。

所以 QEMU 的 range TLBI 更准确的说法是：**支持 range opcode，但允许内部用更粗粒度 invalidate 近似实现**。

## 4.9 DSB 后 TLBI 可见性：QEMU 有同步点，但没有真实传播时间

硬件规范强调：TLBI 的全局可见性通常需要配合 `DSB` 才能保证。QEMU 没有真实互连传播过程，因此它的“可见性”主要依赖：

- `all_cpus_synced` 路径把远端 CPU 工作排队；
- 当前 CPU 上使用 `async_safe_run_on_cpu()` 创建同步点（`accel/tcg/cputlb.c:423-435`, `accel/tcg/cputlb.c:803-844`, `cpu-common.c:327-338`）。

这意味着：

- 从功能角度，QEMU 能近似保证“恢复执行后，相关 CPU 的软 TLB 已经被刷”；
- 但它不会体现真实硬件里：
  - TLBI 广播在 fabric 上的传播时间；
  - 某核心先看到、某核心后看到的短暂窗口；
  - DSB/ISH/OSH 与 shareability 域交互的性能/时序代价。

因此：**QEMU 验证的是“结果 eventually 正确”，不是“时序和域传播是否与硬件一致”。**

---

## 5. PoC / PoU / 一致性点模型

## 5.1 在 QEMU 中，PoC / PoU / 主存几乎退化为同一个东西

按 DDI 0487 M.b §B2.4：

- **PoC**：所有 agent（CPU、DMA、其他主设备）看到一致数据的点；
- **PoU**：某个 PE 的 I-side 和 D-side 统一可见的点；
- **PoP / PoDP**：持久化相关的更深层可见性/持久性点。

在真实 SoC 中，这些点通常不相同，因为存在：

- 私有 L1 I/D cache；
- 共享 LLC；
- snoop coherent fabric；
- writeback 队列；
- 持久化控制器。

而在 QEMU 中：

- guest load/store 主要落到共享 host RAM；
- 没有每 vCPU 私有 D-cache 副本；
- 没有 guest 可见 I-cache 内容，只存在 TB；
- TB 也不是 cache hierarchy 的一部分，而是翻译缓存。

因此**PoC/PoU 在大多数功能路径上被压扁成“guest RAM + TB 失效规则”**。

## 5.2 `CTR_EL0` / `CLIDR_EL1` / `CCSIDR`：会报告“像真的 cache”，但语义并不真

QEMU 仍然给 guest 暴露 cache 相关寄存器。

例如 `-cpu max` 的 AArch64 TCG init：

- 设置 `CLIDR`、多级 `CCSIDR`，描述 64KB L1 D/I、1MB L2、2MB L3（`target/arm/tcg/cpu64.c:1163-1178`）；
- 又把 `CLIDR_EL1.LOUIS/LOUU` 清零（`target/arm/tcg/cpu64.c:1208-1215`）；
- 并把 `CTR_EL0.IDC` / `DIC` 置 1，明确告诉 guest：“data-to-instruction / instruction-to-guest coherence 不需要 cache maintenance”（`target/arm/tcg/cpu64.c:1217-1225`）。

这是一种非常 QEMU 化的组合：

- 一方面，guest 看到看似正常的 cache line size / hierarchy；
- 另一方面，QEMU 又通过 `DIC/IDC=1` 告诉 guest “别真的按普通硬件去维护 cache”。

### 5.2.1 这会造成什么结果

- 依赖 `CTR_EL0.DIC/IDC` 的现代软件，可能主动跳过部分维护序列；
- 依赖 `CTR_EL0.DminLine` / `IminLine` 的软件，仍会据此做 line 对齐；
- 依赖 `CLIDR/CCSIDR` 去推断“真实 cache 拓扑/层级行为”的软件，在 QEMU 上得不到真实答案。

## 5.3 `DCZID_EL0` / cache line size：QEMU 只在少数 helper 中真正使用

QEMU 多个 CPU 型号设置 `DCZID_EL0.BS`，比如常见 CPU 为 `4`（即 64B），个别是 256B/512B（`target/arm/tcg/cpu64.c:75`, `221-222`, `520`, `1393-1394` 等）。

`DCZID_EL0` 的读取通过 `aa64_dczid_read()` 动态合成 `DZP` 位（`target/arm/helper.c:3227-3239`）。

这个 block size 主要真实影响：

- `DC ZVA` / `DC GZVA` 的清零块大小（`target/arm/tcg/helper-a64.c:799-800`）；
- MTE 相关 zero/tag helper；
- 某些 line 对齐逻辑。

但它**不会**驱动真实 D-cache line 的存在、竞争、替换、写回。

### 结论

所以 QEMU 的 cache 类型寄存器更像：

- **ABI/guest 软件兼容信息**；
- 不是**cache 微架构真值**。

---

## 6. 多核 Cache 一致性

## 6.1 真实 ARM：依赖 coherent interconnect + cache protocol

真实 ARM 多核一致性通常依赖：

- 私有 cache + 共享 cache 组合；
- MESI/MOESI 等状态机；
- snoop/filter/directory；
- store buffer、writeback queue、speculation、line bouncing；
- shareability 域与 barrier 组合。

QEMU 没有这些对象。

## 6.2 QEMU：多核“天然一致”，因为大家直接共享一份 RAM

对 QEMU 来说：

- vCPU 写内存，本质上就是修改共享 RAM；
- 另一个 vCPU 下一次 load 就能看到；
- 不存在“还留在前一个 vCPU 私有 D-cache 里，尚未到 PoC”的阶段；
- 不存在 cache line ownership 转移；
- 不存在 false sharing 导致的 line ping-pong。

这就是为什么多核 cache 一致性在 QEMU 中会“自动满足”。

## 6.3 这会掩盖哪些真实硬件问题

### 6.3.1 store buffer / 写暂不可见

真机上，一个核刚做完 store，不代表另一个核立刻看到；还可能需要 barrier 或 coherence 完成。QEMU 通常把写变成“立即进入共享内存状态”。

结果：

- 某些漏 barrier 的 lock-free/driver/firmware 代码，在 QEMU 上更容易“看起来没问题”；
- 但这更多是内存模型层面的掩盖，与 cache hierarchy 缺失叠加在一起。

### 6.3.2 false sharing / line bouncing

真机上，两个 CPU 频繁写同一 cache line 的不同字，会导致 line ownership 抖动和严重性能下降。QEMU 没有 guest cache line 所有权概念，因此无法复现。

### 6.3.3 cache maintenance 广播延迟

真机上：

- `IC IALLUIS` / `TLBI ... IS` / `DC` 广播需要在 shareability 域传播；
- 传播要时间，还会受互连与核状态影响。

QEMU 上：

- 广播退化成“把工作投递给所有 vCPU，然后等同步点”；
- 没有 fabric latency。

## 6.4 结论

QEMU 的多核一致性模型可以概括为：

> **功能上比真实 ARM 更强、更同步、更少中间态。**

这对跑通 guest 很友好，但对发现“遗漏 cache maintenance / barrier / shareability 假设”的问题很不友好。

---

## 7. 自修改代码与 JIT 序列

## 7.1 规范要求的标准序列

按 ARM 规范，自修改代码/JIT 在通用场景下通常需要：

1. 先把新指令字节写入内存；
2. `DC CVAU`（清到 PoU）；
3. `DSB ISH`；
4. `IC IVAU`（I-cache invalidate to PoU）；
5. `DSB ISH`；
6. `ISB`。

这个序列的目的，是把数据侧写入与取指侧可见性联系起来。

## 7.2 QEMU system mode：很多步骤即使省略，也常常“照样跑”

原因不是 QEMU 正确模拟了该序列，而是：

- `DC CVAU` 是 NOP（`target/arm/helper.c:3528-3532`）；
- `IC IVAU` 在 system mode 也是 NOP（`target/arm/helper.c:3497-3507`）；
- 但代码页被写后，TB 会在物理页层面失效（`system/physmem.c:3142-3145`, `accel/tcg/tb-maint.c:1183-1206`）；
- 下一次执行会重新翻译，直接看到新代码。

因此在 QEMU system mode 中，以下 guest 错误很容易被掩盖：

- 漏掉 `DC CVAU`；
- 漏掉 `IC IVAU`；
- 漏掉中间 `DSB`；
- 只做 `ISB` 就跳去执行新代码。

### 结论

**QEMU system mode 更接近“写代码页 = 自动让新代码生效”，而不是“必须满足架构规定的 D-cache/I-cache 协调序列”。**

## 7.3 QEMU user mode：`IC IVAU` 变成 JIT 协议的一部分

在 user mode 下，如果 JIT 采用双映射（W^X），QEMU 不能靠“执行页被写脏”自动发现变化，所以它要求 guest/guest 程序显式执行 `IC IVAU`，该指令会触发 TB 范围失效（`target/arm/helper.c:3416-3441`）。

这让 user mode 下的行为比 system mode 更接近“JIT 需要自己宣告代码变更”。

但即便如此，QEMU 仍没有真实 I-cache，只是在模拟：

- “你告诉我这段代码变了”
- “我就把这段 TB 作废”。

## 7.4 验证建议

如果目的是验证 guest/JIT 是否真正遵守 ARM 的自修改代码规范，QEMU 结论只能是：

- **能验证功能是否大致可跑；**
- **不能验证缺少 `DC/IC/DSB/ISB` 中某一步时，真机会不会失败。**

---

## 8. DMA 一致性

## 8.1 真实硬件：non-coherent DMA 需要显式 cache maintenance

对 non-coherent DMA：

- CPU -> Device：要 `clean`，否则设备可能读到旧数据；
- Device -> CPU：要 `invalidate`，否则 CPU 可能继续读到旧 cache line；
- 若顺序错了，还可能把 CPU 脏数据覆盖掉或丢掉。

这就是 `DC CVAC/CIVAC/IVAC` 真正重要的地方。

## 8.2 QEMU：DMA 直接改共享内存，CPU 也直接从共享内存读

由于 QEMU 没有 guest D-cache hierarchy：

- 设备 DMA 写 guest RAM 后，CPU 往往马上就能看到；
- CPU 刚写给 DMA 的 buffer，也通常已经在 guest RAM 中；
- `DC IVAC/CVAC/CIVAC` 又是 NOP（`target/arm/helper.c:3510-3537`）。

所以在 QEMU 上：

- 漏掉 cache clean/invalidate 的 DMA 驱动，往往仍然“工作正常”；
- 真实硬件上那些“偶发旧数据”“偶发覆盖”“偶发包损坏/描述符错乱”的 bug 很难暴露。

## 8.3 最危险的误判

最危险的误判不是“QEMU 挂了”，而是：

> **驱动作者误以为自己的 non-coherent DMA 路径已经被验证过。**

实际上 QEMU 只能证明：

- 设备模型功能路径通；
- 内存共享路径通；
- 不能证明 cache maintenance 正确。

### 结论

对 DMA 一致性代码，QEMU 最多可作为：

- 功能联调环境；
- 不是 cache correctness 环境。

---

## 9. FEAT_DPB / 持久化

## 9.1 `DC CVAP / CVADP` 在 QEMU 中的真实价值

如前所述，QEMU 的 `DC CVAP/CVADP` 会进入 `dccvap_writefn()`，最终可能调用 `memory_region_writeback()` -> `memory_region_msync()`（`target/arm/helper.c:5315-5347`, `system/memory.c:2393-2408`）。

这说明它并非纯 NOP，而是有意给持久化/后端 RAM 提供某种“向宿主回写”的钩子。

## 9.2 但它并不等价于架构 PoP / PoDP

QEMU 缺少以下关键内容：

- 持久化层级（controller / ADR / eADR / deep persistence）；
- 掉电模型；
- `DSB` 与 durability ordering 的强绑定；
- cache line 在 persistence domain 间逐层推进的状态。

所以：

- `CVAP` 与 `CVADP` 在 QEMU 中几乎共用同一 helper（`target/arm/helper.c:5349-5363`）；
- 真实硬件上 PoP 与 PoDP 的区分，在这里没有被真正建立。

## 9.3 对 NVDIMM / pmem guest 的意义

因此，如果 guest 软件是：

- 文件系统 / 数据库 / 持久化日志；
- 依赖 `dc cvap` / `dc cvadp` + barrier 做 crash consistency；

那么 QEMU 可以帮助验证：

- 指令是否被 guest 正常执行；
- 基本路径是否联通；

但不能严肃验证：

- 断电后一致性；
- 深持久化边界；
- 真正的 durability guarantee。

---

## 10. 综合差异汇总表

| 主题 | ARM 规范预期 | QEMU 11.0.50 实现 | 主要差异/影响 | 关键源码 |
|---|---|---|---|---|
| `DC CVAC/CIVAC/CVAU` | clean 到 PoC/PoU | NOP | 不会把“脏 cache line”写回，因为根本没有此对象 | `target/arm/helper.c:3519-3537` |
| `DC IVAC` | invalidate 到 PoC，脏行可能丢数据 | NOP | 无法暴露 invalidate-before-clean 导致的数据丢失 | `target/arm/helper.c:3510-3514` |
| `DC CISW/CSW/ISW` | set/way 维护 cache 层级 | NOP | 无 set/way，无低功耗/安全切换相关 cache 行为 | `target/arm/helper.c:3515-3518`, `3524-3527`, `3538-3541` |
| `DC ZVA` | 按 DCZID block 清零 | 真正清零内存块 | 功能接近，但不做 device memory attribute/alignment 建模 | `target/arm/tcg/translate-a64.c:2996-3012`, `target/arm/tcg/helper-a64.c:793-835` |
| `DC GZVA` | 清零并处理 tag | 先 `dc_zva` 再 tag helper | 更像“内存清零 + tag 写入”，不是 cache/tag array 微架构 | `target/arm/tcg/translate-a64.c:3033-3048` |
| `DC CVAP/CVADP` | clean 到 PoP/PoDP | `memory_region_writeback`/`msync` 近似 | 仅弱持久化近似，无真实 PoP/PoDP | `target/arm/helper.c:5315-5363`, `system/memory.c:2400-2408` |
| `IC IALLU/IALLUIS` | I-cache invalidate（含 shareable 域语义） | system mode 下 NOP | 无真实 I-cache 广播；TB 不靠该指令失效 | `target/arm/helper.c:3487-3496` |
| `IC IVAU` | 按 VA invalidate to PoU | user mode 失效 TB；system mode NOP | system mode 中 guest 漏掉 `IC IVAU` 也常能跑 | `target/arm/helper.c:3416-3441`, `3497-3507` |
| system mode SMC/JIT | 需 `DC`/`IC`/`DSB`/`ISB` 序列 | 写代码页直接导致 TB 失效 | QEMU 高估 guest 自修改代码正确性 | `system/physmem.c:3142-3145`, `accel/tcg/tb-maint.c:1183-1206` |
| `TLBI VMALLE1*` | 按 shareability 域全刷 | 支持；`OS` 折叠到 `IS` | 不区分 inner/outer shareable 传播域 | `target/arm/tcg/tlb-insns.c:317-337`, `1159-1165` |
| `TLBI VAE1*` | 按 VA 精确刷 | 支持 VA 刷 | `VAE1/VAAE1/VALE1/VAALE1` 折叠，忽略 ASID/leaf-only | `target/arm/tcg/tlb-insns.c:445-463` |
| `TLBI ASIDE1*` | 仅按 ASID 失效 | 直接映射到 `VMALLE1*` | ASID 语义基本丢失，过度失效 | `target/arm/tcg/tlb-insns.c:646-651`, `682-687`, `1172-1177` |
| `TLBI VMALLS12E1*` | stage1+2 全刷 | 通过 `ALLE1*` mask 实现 | 功能上会刷 stage1+2，但 shareability/opcode 差异被压平 | `target/arm/tcg/tlb-insns.c:718-737`, `target/arm/helper.c:404-420` |
| `TLBI IPAS2E1/IPAS2E1IS` | IPA stage-2 按页刷 | 已实现 | 基本可用，但软件同步而非硬件广播 | `target/arm/tcg/tlb-insns.c:501-523` |
| `TLBI ...OS` | Outer Shareable 传播 | 很多变体折叠为 IS；部分 `IPAS2E1OS` 直接 NOP | Outer shareable 语义不真实 | `target/arm/tcg/tlb-insns.c:1159-1244` |
| FEAT_TLBIRANGE | 按范围高精度失效 | 支持 range opcode | 仍忽略 ASID/leaf-only/IS-OS 区分，底层可能退化 full flush | `target/arm/tcg/tlb-insns.c:843-1015`, `accel/tcg/cputlb.c:663-845` |
| DSB 后 TLBI 可见性 | 需架构同步保证 | `all_cpus_synced + safe work` | 有同步点，但无真实传播延迟/域开销 | `accel/tcg/cputlb.c:350-367`, `423-435`, `803-844`, `cpu-common.c:327-338` |
| PoC/PoU | 不同一致性点 | 基本压扁到共享 RAM/TB 模型 | cache maintenance 语义被显著弱化 | `target/arm/tcg/cpu64.c:1208-1225` |
| `CTR_EL0.DIC/IDC` | 反映 I/D coherence 属性 | `-cpu max` 置 1；user mode 还会清 DIC | 用寄存器显式告诉 guest“不要依赖真实 cache 维护” | `target/arm/tcg/cpu64.c:1217-1225`, `target/arm/cpu.c:1881-1890` |

---

## 11. 对 Guest 软件的影响

## 11.1 哪些东西在 QEMU 上“更容易成功”

### 11.1.1 漏掉 data cache clean/invalidate 的 DMA 驱动

由于没有真实 D-cache，guest 很容易误以为：

- `dma_map_single()` 前后的 cache maintenance 不是必须；
- non-coherent 设备也像 coherent 一样工作。

### 11.1.2 漏掉 `DC CVAU / IC IVAU / DSB / ISB` 的 JIT/SMC

在 system mode，TB 写脏即失效，导致很多不符合架构序列的实现仍然可执行。

### 11.1.3 依赖粗粒度 TLBI 也能跑的 MMU 代码

因为 QEMU 常常过度失效：

- 少刷一点本该不行的地方，也可能被“全刷”掩盖；
- 只刷某个 ASID 的 bug，可能因为 QEMU 实际把整个 regime 刷掉而被隐藏。

## 11.2 哪些东西在 QEMU 上“验证不出来”

### 11.2.1 `DC IVAC` 的破坏性语义

真机上可能丢数据；QEMU 上不会。

### 11.2.2 PoC / PoU 差异引起的取指/数据不同步

真机需要显式维护；QEMU 常常直接绕过。

### 11.2.3 outer-shareable / inner-shareable 的真实传播域

QEMU 多数情况下把 OS/IS 压平，或者直接忽略。

### 11.2.4 持久化域边界

`CVAP/CVADP` 在 QEMU 只给出极弱近似，无法验证 crash consistency 语义。

## 11.3 哪些东西仍然能在 QEMU 上验证

尽管差异显著，QEMU 仍然可以验证：

- guest 是否发出了正确的系统指令编码；
- trap/权限控制是否大致符合架构；
- 基本页表修改后，TLB 是否被刷新；
- 多 vCPU 下 TLBI 广播路径是否“功能上”传播到其他 vCPU；
- `DC ZVA` / MTE tag helper / `CVAP` 基本钩子是否可执行。

## 11.4 最终结论

如果把问题表述成一句话：

> **QEMU 把 ARM64 cache hierarchy 从“真实一致性对象”简化成了“共享内存 + TB/TLB 软件缓存”，因此对大多数 guest 是透明的，但会系统性掩盖 cache maintenance、non-coherent DMA、自修改代码序列、shareability 域和 persistence 相关的硬件差异。**

所以在规范验证上应把 QEMU 的结论分成两类：

1. **功能正确性结论**：某条指令/某个 guest 路径在 QEMU 上能跑通；
2. **硬件一致性结论**：不能仅凭 QEMU 推断真机也满足 DDI 0487 M.b 的 cache / TLBI / PoC / PoU / PoP 语义。

前者通常成立；后者在本主题下经常不成立。
