# Doc 113: GDB x/xp 内存访问原理 — 虚拟地址与物理地址读取路径

## 文档信息
- **组件**: GDB Stub 内存访问, PhyMemMode, HMP x/xp, ARM64 页表翻译
- **源码版本**: QEMU 11.0.50
- **关键源文件**: `gdbstub/system.c`, `gdbstub/gdbstub.c`, `system/physmem.c`, `target/arm/ptw.c`, `monitor/hmp-cmds.c`
- **分析日期**: 2025-07
- **归档目录**: debug/

---

## 目录
1. [概述](#1-概述)
2. [整体架构图](#2-整体架构图)
3. [GDB 虚拟地址访问 (默认模式)](#3-gdb-虚拟地址访问-默认模式)
4. [GDB 物理地址访问 (PhyMemMode)](#4-gdb-物理地址访问-phymemmode)
5. [Monitor x 与 xp 命令](#5-monitor-x-与-xp-命令)
6. [ARM64 页表翻译细节](#6-arm64-页表翻译细节)
7. [完整调用链对比](#7-完整调用链对比)
8. [实践使用指南](#8-实践使用指南)
9. [源码索引](#9-源码索引)

---

## 1. 概述

GDB 通过 RSP 协议的 `m` 命令读取目标内存。在 QEMU 系统模式中，地址可以是 Guest 虚拟地址 (VA) 或物理地址 (PA)，两者的处理路径完全不同：

| 访问方式 | 地址类型 | 是否翻译 | 核心函数 |
|----------|----------|----------|----------|
| GDB `x` (默认) | 虚拟地址 | ✅ CPU 当前页表 | `cpu_memory_rw_debug()` |
| GDB `x` (PhyMemMode=1) | 物理地址 | ❌ 直接访问 | `cpu_physical_memory_read()` |
| Monitor `x` | 虚拟地址 | ✅ CPU 当前页表 | `cpu_memory_rw_debug()` |
| Monitor `xp` | 物理地址 | ❌ 直接访问 | `address_space_read()` |

---

## 2. 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        GDB 客户端                                │
│   (gdb) x/4gx 0xffff800080000000     ← 虚拟地址                │
│   (gdb) monitor qemu.PhyMemMode 1                              │
│   (gdb) x/4gx 0x40000000             ← 物理地址                │
└────────────────────────┬────────────────────────────────────────┘
                         │ RSP 'm' packet: m<addr>,<len>
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  gdbstub/gdbstub.c:1292  handle_read_mem()                      │
│  解析地址和长度, 调用 gdb_target_memory_rw_debug()               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  gdbstub/system.c:452  gdb_target_memory_rw_debug()             │
│                                                                  │
│  if (phy_memory_mode) {                                         │
│      cpu_physical_memory_read(addr, buf, len);  ─── 物理路径 ──┐│
│  } else {                                                       ││
│      cpu_memory_rw_debug(cpu, addr, buf, len);  ─── 虚拟路径 ──┤│
│  }                                                              ││
└──────────────┬──────────────────────────────────────────────┬───┘│
               │ 虚拟路径                                      │ 物理│
               ▼                                               ▼    │
┌──────────────────────────────┐  ┌────────────────────────────────┐
│ system/physmem.c:4030        │  │ 直接物理内存访问                 │
│ cpu_memory_rw_debug()        │  │ cpu_physical_memory_read()      │
│                              │  │ → address_space_read()          │
│ 循环每页:                     │  │ → MemoryRegion dispatch        │
│ 1. cpu_get_phys_page_debug() │  │   → RAM 或 MMIO 设备           │
│    → 页表遍历 (VA → PA)      │  └────────────────────────────────┘
│ 2. address_space_rw(PA)      │
│    → 用物理地址访问内存       │
└──────────────┬───────────────┘
               │ 页表翻译
               ▼
┌──────────────────────────────────────────────────────────────────┐
│  target/arm/ptw.c:3966                                           │
│  arm_cpu_get_phys_page_attrs_debug()                             │
│    → arm_cpu_get_phys_page(env, addr, attrs, arm_mmu_idx(env))  │
│      → get_phys_addr_gpc(env, &ptw, addr, ...)                  │
│        → 完整 ARM64 S1/S2 页表遍历                               │
│        → 返回物理地址 (或 -1 表示翻译失败)                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. GDB 虚拟地址访问 (默认模式)

### 3.1 RSP 协议入口

当 GDB 执行 `x/4gx 0xffff800080000000` 时，发送 RSP packet:
```
m ffff800080000000,20
```

### 3.2 handle_read_mem()

```c
// gdbstub/gdbstub.c:1292
static void handle_read_mem(GArray *params, void *user_ctx)
{
    // 解析地址和长度
    g_byte_array_set_size(gdbserver_state.mem_buf, len);

    if (gdb_target_memory_rw_debug(gdbserver_state.g_cpu,
                                    addr, buf, len, false) != 0) {
        gdb_put_packet("E14");  // 读取失败
        return;
    }
    // 成功: 转为 hex 返回
    gdb_memtohex(gdbserver_state.str_buf, buf, len);
    gdb_put_strbuf();
}
```

### 3.3 gdb_target_memory_rw_debug() — 核心分发

```c
// gdbstub/system.c:450-468
static int phy_memory_mode;  // 全局标志: 0=虚拟, 1=物理

int gdb_target_memory_rw_debug(CPUState *cpu, hwaddr addr,
                               uint8_t *buf, int len, bool is_write)
{
    if (phy_memory_mode) {
        // 物理地址模式: 直接访问, 不翻译
        if (is_write) {
            cpu_physical_memory_write(addr, buf, len);
        } else {
            cpu_physical_memory_read(addr, buf, len);
        }
        return 0;
    }

    // 虚拟地址模式: 需要页表翻译
    if (cpu->cc->memory_rw_debug) {
        return cpu->cc->memory_rw_debug(cpu, addr, buf, len, is_write);
    }
    return cpu_memory_rw_debug(cpu, addr, buf, len, is_write);
}
```

### 3.4 cpu_memory_rw_debug() — 按页翻译

```c
// system/physmem.c:4030-4063
int cpu_memory_rw_debug(CPUState *cpu, vaddr addr,
                        void *ptr, size_t len, bool is_write)
{
    hwaddr phys_addr;
    vaddr l, page;
    uint8_t *buf = ptr;

    cpu_synchronize_state(cpu);  // 确保 CPU 状态同步 (KVM 需要)

    while (len > 0) {
        int asidx;
        MemTxAttrs attrs;

        // 1. 对齐到页边界
        page = addr & TARGET_PAGE_MASK;

        // 2. 调用 CPU 特定的页表翻译
        phys_addr = cpu_get_phys_page_attrs_debug(cpu, page, &attrs);
        asidx = cpu_asidx_from_attrs(cpu, attrs);

        // 3. 翻译失败 → 返回错误
        if (phys_addr == -1)
            return -1;

        // 4. 计算页内偏移
        l = (page + TARGET_PAGE_SIZE) - addr;
        if (l > len) l = len;
        phys_addr += (addr & ~TARGET_PAGE_MASK);

        // 5. 用物理地址访问
        res = address_space_rw(cpu->cpu_ases[asidx].as,
                               phys_addr, attrs, buf, l, is_write);
        if (res != MEMTX_OK) return -1;

        len -= l;
        buf += l;
        addr += l;
    }
    return 0;
}
```

**关键设计点**:
- 按 4KB 页为单位翻译 (跨页访问会多次翻译)
- 使用 debug 模式翻译 (不触发 TLB 填充、不产生异常)
- `cpu_synchronize_state()` 确保 KVM vCPU 寄存器已同步到 QEMU

---

## 4. GDB 物理地址访问 (PhyMemMode)

### 4.1 切换模式

```
(gdb) monitor qemu.PhyMemMode 1    # 切换到物理地址模式
(gdb) x/4gx 0x40000000             # 此时地址被当作物理地址
(gdb) monitor qemu.PhyMemMode 0    # 切回虚拟地址模式
```

### 4.2 PhyMemMode 实现

```c
// gdbstub/gdbstub.c:1946-1974
// 查询当前模式:
{   .cmd = "qemu.PhyMemMode",
    .handler = handle_query_phy_mem_mode, }

// 设置模式:
{   .cmd = "qemu.PhyMemMode:",
    .handler = handle_set_phy_mem_mode, }

// gdbstub/system.c:490-509
static void handle_query_phy_mem_mode(...)
{
    g_string_printf(gdbserver_state.str_buf, "%d", phy_memory_mode);
}

static void handle_set_phy_mem_mode(...)
{
    if (val == '0') phy_memory_mode = 0;
    else            phy_memory_mode = 1;
}
```

### 4.3 物理访问路径

```
phy_memory_mode = 1 时:
gdb_target_memory_rw_debug()
  → cpu_physical_memory_read(addr, buf, len)
    → address_space_read(&address_space_memory, addr, ...)
      → flatview_read(fv, addr, ...)
        → MemoryRegion dispatch
          ├── RAM: 直接 memcpy 从 host 内存
          └── MMIO: 调用设备 read 回调
```

**特点**: 完全绕过 CPU 页表，地址直接作为系统物理地址空间中的地址使用。

---

## 5. Monitor x 与 xp 命令

### 5.1 x 命令 (虚拟地址)

```c
// monitor/hmp-cmds.c:692
void hmp_memory_dump(Monitor *mon, const QDict *qdict)
{
    vaddr addr = qdict_get_int(qdict, "addr");
    memory_dump(mon, count, format, size, addr, false);  // is_physical=false
}
```

### 5.2 xp 命令 (物理地址)

```c
// monitor/hmp-cmds.c:702
void hmp_physical_memory_dump(Monitor *mon, const QDict *qdict)
{
    hwaddr addr = qdict_get_int(qdict, "addr");
    memory_dump(mon, count, format, size, addr, true);   // is_physical=true
}
```

### 5.3 memory_dump() 分发

```c
// monitor/hmp-cmds.c:584-690
static void memory_dump(Monitor *mon, int count, int format, int wsize,
                        uint64_t addr, bool is_physical)
{
    while (len > 0) {
        if (is_physical) {
            // xp: 直接物理地址访问
            AddressSpace *as = cs ? cs->as : &address_space_memory;
            address_space_read(as, addr, MEMTXATTRS_UNSPECIFIED, buf, l);
        } else {
            // x: 通过 CPU 页表翻译
            cpu_memory_rw_debug(cs, addr, buf, l, 0);
        }
        // 格式化输出...
    }
}
```

---

## 6. ARM64 页表翻译细节

### 6.1 入口: arm_cpu_get_phys_page_attrs_debug()

```c
// target/arm/ptw.c:3966-3994
hwaddr arm_cpu_get_phys_page_attrs_debug(CPUState *cs, vaddr addr,
                                         MemTxAttrs *attrs)
{
    ARMCPU *cpu = ARM_CPU(cs);
    CPUARMState *env = &cpu->env;
    ARMMMUIdx mmu_idx = arm_mmu_idx(env);  // 获取当前 EL 的 MMU 索引

    hwaddr res = arm_cpu_get_phys_page(env, addr, attrs, mmu_idx);

    if (res != -1) {
        return res;
    }

    // 如果当前权限翻译失败, 尝试非特权模式
    // (处理 LDTR/STTR 等非特权访问指令)
    switch (mmu_idx) {
    case ARMMMUIdx_E10_1:
    case ARMMMUIdx_E10_1_PAN:
        return arm_cpu_get_phys_page(env, addr, attrs, ARMMMUIdx_E10_0);
    case ARMMMUIdx_E20_2:
    case ARMMMUIdx_E20_2_PAN:
        return arm_cpu_get_phys_page(env, addr, attrs, ARMMMUIdx_E20_0);
    default:
        return -1;
    }
}
```

### 6.2 核心翻译: arm_cpu_get_phys_page()

```c
// target/arm/ptw.c:3945-3964
static hwaddr arm_cpu_get_phys_page(CPUARMState *env, vaddr addr,
                                    MemTxAttrs *attrs, ARMMMUIdx mmu_idx)
{
    S1Translate ptw = {
        .in_mmu_idx = mmu_idx,
        .in_space = arm_mmu_idx_to_security_space(env, mmu_idx),
        .in_debug = true,       // ← 调试模式: 不触发异常!
        .in_at = true,          // ← AT (Address Translation) 操作
        .in_prot_check = 0,     // ← 不做权限检查
    };
    GetPhysAddrResult res = {};
    ARMMMUFaultInfo fi = {};

    // 完整 ARM64 页表遍历 (含 S1 + S2 + Granule Protection)
    bool ret = get_phys_addr_gpc(env, &ptw, addr, MMU_DATA_LOAD, 0, &res, &fi);
    *attrs = res.f.attrs;

    if (ret) {
        return -1;    // 翻译失败 (页未映射/权限不足)
    }
    return res.f.phys_addr;  // 返回物理地址
}
```

### 6.3 MMU 索引与 EL 的关系

```
arm_mmu_idx(env) 根据当前 CPU 状态返回:

EL0 (用户态):    ARMMMUIdx_E10_0 或 ARMMMUIdx_E20_0
EL1 (内核态):    ARMMMUIdx_E10_1 (或 _PAN)
EL2 (Hypervisor): ARMMMUIdx_E20_2 (VHE) 或 ARMMMUIdx_E2
EL3 (Secure):    ARMMMUIdx_E3
```

**重要含义**: GDB 的 `x` 命令使用的是 CPU **当前 EL** 的页表!
- 如果 CPU 停在 EL1 (内核), 访问的是内核页表
- 如果 CPU 停在 EL0 (用户), 访问的是用户进程页表
- 如果 CPU 停在 EL3, 使用 Secure 页表

### 6.4 debug 模式的特殊处理

设置 `.in_debug = true` 的效果:
1. **不触发 Data Abort** — 翻译失败返回 -1 而非注入异常
2. **不更新 TLB** — 不污染正常执行路径的 TLB 缓存
3. **不检查权限** (`.in_prot_check = 0`) — 即使页标记为不可读也能访问
4. **不触发 Watchpoint** — 不影响调试断点状态

---

## 7. 完整调用链对比

### 7.1 虚拟地址读取 (GDB `x` 默认)

```
GDB: x/gx 0xffff800080000000
  │
  ├─ RSP packet: "m ffff800080000000,8"
  │
  ▼
gdbstub.c:1292  handle_read_mem()
  │ addr=0xffff800080000000, len=8
  ▼
system.c:452  gdb_target_memory_rw_debug(cpu, 0xffff...000, buf, 8, false)
  │ phy_memory_mode == 0
  ▼
physmem.c:4030  cpu_memory_rw_debug(cpu, 0xffff...000, buf, 8, false)
  │ page = 0xffff800080000000 & ~0xFFF = 0xffff800080000000
  ▼
ptw.c:3966  arm_cpu_get_phys_page_attrs_debug(cpu, 0xffff800080000000, &attrs)
  │ mmu_idx = ARMMMUIdx_E10_1 (假设 CPU 在 EL1)
  ▼
ptw.c:3945  arm_cpu_get_phys_page(env, addr, attrs, E10_1)
  │ S1Translate { .in_debug=true, .in_at=true }
  ▼
ptw.c  get_phys_addr_gpc(env, &ptw, addr, MMU_DATA_LOAD, ...)
  │ 遍历 TTBR1_EL1 → L0/L1/L2/L3 页表
  │ 返回 phys_addr = 0x40000000 (示例)
  ▼
physmem.c:4053  address_space_rw(as, 0x40000000, attrs, buf, 8, false)
  │ → MemoryRegion 派发 → RAM memcpy
  ▼
返回 8 字节数据 → hex 编码 → RSP 回复
```

### 7.2 物理地址读取 (GDB PhyMemMode=1)

```
GDB: monitor qemu.PhyMemMode 1
GDB: x/gx 0x40000000
  │
  ├─ RSP packet: "m 40000000,8"
  │
  ▼
gdbstub.c:1292  handle_read_mem()
  │ addr=0x40000000, len=8
  ▼
system.c:452  gdb_target_memory_rw_debug(cpu, 0x40000000, buf, 8, false)
  │ phy_memory_mode == 1 ← 走物理路径!
  ▼
cpu_physical_memory_read(0x40000000, buf, 8)
  │ → address_space_read(&address_space_memory, 0x40000000, ...)
  │ → flatview 查找 MemoryRegion
  │ → RAM 直接 memcpy
  ▼
返回 8 字节数据
```

### 7.3 Monitor xp 命令

```
(qemu) xp/4gx 0x40000000
  │
  ▼
hmp-cmds.c:702  hmp_physical_memory_dump()
  │ is_physical = true
  ▼
hmp-cmds.c:584  memory_dump(..., is_physical=true)
  │
  ▼ (循环)
hmp-cmds.c:637  address_space_read(as, 0x40000000, ..., buf, 32)
  │ → 直接物理地址访问 (无页表翻译)
  ▼
格式化输出:
0000000040000000: 0x... 0x... 0x... 0x...
```

---

## 8. 实践使用指南

### 8.1 何时用虚拟地址 (GDB `x` 默认)

```gdb
# 调试内核代码时, 访问内核数据结构:
(gdb) x/4gx &init_task           # 内核虚拟地址
(gdb) x/s current->comm          # 进程名字符串
(gdb) x/20i $pc                  # 当前指令 (PC 是虚拟地址)
```

### 8.2 何时用物理地址

```gdb
# 场景 1: 检查页表本身 (页表项存储的是物理地址)
(gdb) monitor qemu.PhyMemMode 1
(gdb) x/gx 0x40001000            # 直接读 PGD 物理地址

# 场景 2: MMU 未开启时 (early boot)
# 此时虚拟地址 == 物理地址, 两种模式效果相同

# 场景 3: 设备 MMIO 区域
(gdb) monitor qemu.PhyMemMode 1
(gdb) x/wx 0x08000000            # GIC Distributor
(gdb) x/wx 0x09000000            # UART

# 场景 4: 验证物理内存映射
(gdb) monitor qemu.PhyMemMode 1
(gdb) x/gx 0x40000000            # RAM 起始 (virt 平台)
```

### 8.3 Monitor xp 的优势

```
(qemu) xp/4gx 0x40000000
```
- 不需要切换 PhyMemMode (不影响 GDB 后续操作)
- 适合在 HMP 中快速查看物理内存
- 可以在没有 CPU 上下文时访问 (如检查固件加载)

### 8.4 常见陷阱

| 问题 | 原因 | 解决 |
|------|------|------|
| `x` 返回 "Cannot access memory" | 虚拟地址未映射/翻译失败 | 检查页表或用 xp |
| PhyMemMode=1 后 `x` 看到乱数据 | 地址是虚拟的但被当物理用 | 切回 `PhyMemMode 0` |
| 不同线程/CPU 看到不同内容 | 不同进程有不同页表 | 切换 thread 后再 `x` |
| EL1 下看不到 EL0 用户空间 | TTBR0 映射在 E10_0 | debug 模式会 fallback 尝试 |

---

## 9. 源码索引

| 文件 | 行号 | 函数/变量 | 作用 |
|------|------|-----------|------|
| `gdbstub/gdbstub.c` | 1292 | `handle_read_mem()` | RSP 'm' 命令处理 |
| `gdbstub/gdbstub.c` | 1265 | `handle_write_mem()` | RSP 'M' 命令处理 |
| `gdbstub/gdbstub.c` | 1946 | `qemu.PhyMemMode` | 物理模式查询命令注册 |
| `gdbstub/gdbstub.c` | 1970 | `qemu.PhyMemMode:` | 物理模式设置命令注册 |
| `gdbstub/system.c` | 450 | `phy_memory_mode` | 全局模式标志 |
| `gdbstub/system.c` | 452 | `gdb_target_memory_rw_debug()` | 核心分发 (虚拟/物理) |
| `system/physmem.c` | 4030 | `cpu_memory_rw_debug()` | 虚拟地址按页翻译+访问 |
| `target/arm/ptw.c` | 3966 | `arm_cpu_get_phys_page_attrs_debug()` | ARM64 入口 |
| `target/arm/ptw.c` | 3945 | `arm_cpu_get_phys_page()` | ARM64 页表遍历封装 |
| `target/arm/ptw.c` | - | `get_phys_addr_gpc()` | 完整 S1/S2/GPC 翻译 |
| `target/arm/cpu.c` | 2439 | `.get_phys_page_attrs_debug` | CPU class 注册 |
| `monitor/hmp-cmds.c` | 584 | `memory_dump()` | Monitor x/xp 统一实现 |
| `monitor/hmp-cmds.c` | 692 | `hmp_memory_dump()` | Monitor `x` 入口 |
| `monitor/hmp-cmds.c` | 702 | `hmp_physical_memory_dump()` | Monitor `xp` 入口 |

---

*文档结束*
