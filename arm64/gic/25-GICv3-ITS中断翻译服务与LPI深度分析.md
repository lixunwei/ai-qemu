# GICv3 ITS 中断翻译服务与 LPI 深度分析

## 1. 概述

本文深度分析 QEMU 11.0.50 中 GICv3 ITS（Interrupt Translation Service）和 LPI（Locality-specific Peripheral Interrupt）的实现。ITS 是 GICv3 的关键组件，负责将 MSI/MSI-X 消息翻译为 LPI 中断并路由到目标 CPU。覆盖 ITS 命令队列处理、设备/集合/中断翻译表管理、LPI pending/配置表、以及完整的 MSI→ITS→LPI→CPU 投递流程。

**关键源文件：**
- `hw/intc/arm_gicv3_its.c` — ITS 主实现（命令处理、表管理、翻译）
- `hw/intc/arm_gicv3_its_common.c` — ITS 公共代码（MMIO、MSI 入口）
- `hw/intc/arm_gicv3_redist.c` — 重分发器 LPI 处理（pending 表、优先级）
- `include/hw/intc/arm_gicv3_its_common.h` — ITS 寄存器和数据结构定义
- `include/hw/intc/arm_gicv3_common.h` — LPI 起始 ID 等公共定义
- `hw/intc/arm_gicv3_its_kvm.c` — KVM 直通 ITS

---

## 2. ITS 架构

### 2.1 ITS 在 GICv3 中的位置

```
PCIe 设备 ──→ MSI/MSI-X 写入 ──→ GITS_TRANSLATER
                                       │
                                       ↓
                               ┌───────────────┐
                               │     ITS       │
                               │  DeviceID +   │
                               │  EventID      │
                               │      ↓        │
                               │  设备表(DT)   │
                               │      ↓        │
                               │  ITT 翻译表   │
                               │      ↓        │
                               │  INTID + ICID │
                               │      ↓        │
                               │  集合表(CT)   │
                               │      ↓        │
                               │  目标 CPU     │
                               └───────┬───────┘
                                       │
                                       ↓
                              GICR[target CPU]
                              设置 LPI pending
                                       │
                                       ↓
                               CPU 接口 → CPU
```

### 2.2 QOM 类型与初始化

```c
// arm_gicv3_its.c:1922-2037 — ITS 设备模型
// TYPE_ARM_GICV3_ITS = "arm-gicv3-its"
// 属性：gicv3 (link), num-cpu (uint32)
// realize → gicv3_arm_its_realize()
//   注册 MMIO 区域（控制 + 翻译）
//   初始化命令队列和表描述符
```

### 2.3 核心数据结构

```c
// arm_gicv3_its_common.h:28-85 — ITS 寄存器偏移
#define GITS_CTLR        0x0000
#define GITS_IIDR        0x0004
#define GITS_CBASER      0x0080  // 命令队列基址
#define GITS_CWRITER     0x0088  // 写指针
#define GITS_CREADR      0x0090  // 读指针
#define GITS_BASER       0x0100  // 表基址寄存器 (8个)
#define GITS_TRANSLATER  0x0040  // 翻译寄存器（MSI 写入点）

// arm_gicv3_its.c:44-69 — 内存表条目结构
typedef struct DTEntry {     // 设备表条目
    bool valid;
    unsigned size;           // ITT 大小
    uint64_t ittaddr;        // ITT 基址
} DTEntry;

typedef struct CTEntry {     // 集合表条目
    bool valid;
    uint32_t rdbase;         // 目标重分发器索引
} CTEntry;

typedef struct ITEntry {     // 中断翻译表条目
    bool valid;
    int inttype;             // PHYSICAL 或 VIRTUAL
    uint32_t intid;          // LPI INTID
    uint32_t doorbell;       // vLPI doorbell
    uint32_t icid;           // 集合 ID
    uint32_t vpeid;          // 虚拟 PE ID
} ITEntry;

typedef struct VTEntry {     // 虚拟 PE 表条目
    bool valid;
    unsigned vptsize;
    uint32_t rdbase;
    uint64_t vptaddr;
} VTEntry;
```

---

## 3. ITS 表管理

### 3.1 GITS_BASER 寄存器

```c
// arm_gicv3_its_common.h:287-305 — GITS_BASER 字段
// 每个 GITS_BASER 描述一种表的物理位置和格式：
// - BASER[0]: 设备表 (Device Table)
// - BASER[1]: 集合表 (Collection Table)
// - BASER[2]: vPE 表（GICv4）
// 字段：
//   Physical_Address: 表基址
//   Entry_Size: 条目大小
//   Indirect: 是否二级表
//   Page_Size: 4K/16K/64K
//   Size: 表大小（页数）
```

### 3.2 平坦 vs 二级表地址计算

```c
// arm_gicv3_its.c:129-172 — table_entry_addr()
static uint64_t table_entry_addr(GICv3ITSState *s, TableDesc *td,
                                 uint32_t idx, MemTxResult *res)
{
    if (!td->indirect) {
        // 平坦表：直接计算
        return td->base_addr + idx * td->entry_sz;
    }

    // 二级表：
    // 1. 计算 L1 索引
    l2idx = idx / (td->page_sz / L1TABLE_ENTRY_SIZE);

    // 2. 读取 L1 条目（8 字节，包含 L2 页基址 + Valid 位）
    l2 = ldq_le(base_addr + l2idx * L1TABLE_ENTRY_SIZE);
    if (!(l2 & L2_TABLE_VALID_MASK)) return -1;  // 无效

    // 3. 计算 L2 内偏移
    num_l2_entries = td->page_sz / td->entry_sz;
    return (l2 & ADDR_MASK) + (idx % num_l2_entries) * td->entry_sz;
}
```

### 3.3 表参数提取

```c
// arm_gicv3_its.c:1415-1538 — extract_table_params() / extract_cmdq_params()
// ITS 使能时从 GITS_BASER 解析表配置：
//   base_addr: 物理基址
//   num_entries: 最大条目数
//   entry_sz: 条目大小
//   page_sz: 页大小
//   indirect: 是否间接（二级）
// 命令队列从 GITS_CBASER 解析：
//   base_addr, num_entries (队列深度)
```

---

## 4. ITS 命令队列

### 4.1 命令队列结构

```
Guest 内存中的环形缓冲区：
  GITS_CBASER: 基址 + 大小
  GITS_CWRITER: Guest 写指针（Guest 写入新命令后更新）
  GITS_CREADR: ITS 读指针（ITS 处理完命令后更新）

每条命令: 32 字节 (4 × uint64_t)
  cmdpkt[0]: 命令码 + 操作数低位
  cmdpkt[1-3]: 操作数高位
```

### 4.2 命令处理主循环

```c
// arm_gicv3_its.c:1246-1408 — process_cmdq()
static void process_cmdq(GICv3ITSState *s)
{
    if (!(s->ctlr & ENABLED)) return;

    wr_offset = GITS_CWRITER.OFFSET;
    rd_offset = GITS_CREADR.OFFSET;

    while (wr_offset != rd_offset) {
        // 1. DMA 读取 32 字节命令
        hostmem = address_space_map(as, cq.base_addr + offset, ...);
        // 内存故障 → 设置 STALLED 位，停止处理
        if (!hostmem) { CREADR.STALLED = 1; break; }

        cmd = cmdpkt[0] & CMD_MASK;

        // 2. 分派命令
        switch (cmd) {
        case GITS_CMD_INT:     result = process_its_cmd(s, cmdpkt, INTERRUPT);
        case GITS_CMD_CLEAR:   result = process_its_cmd(s, cmdpkt, CLEAR);
        case GITS_CMD_DISCARD: result = process_its_cmd(s, cmdpkt, DISCARD);
        case GITS_CMD_MAPD:    result = process_mapd(s, cmdpkt);
        case GITS_CMD_MAPC:    result = process_mapc(s, cmdpkt);
        case GITS_CMD_MAPTI:   result = process_mapti(s, cmdpkt, false);
        case GITS_CMD_MAPI:    result = process_mapti(s, cmdpkt, true);
        case GITS_CMD_INV:     result = process_inv(s, cmdpkt);
        case GITS_CMD_INVALL:  /* 全重分发器 LPI 刷新 */
        case GITS_CMD_MOVI:    result = process_movi(s, cmdpkt);
        case GITS_CMD_MOVALL:  result = process_movall(s, cmdpkt);
        case GITS_CMD_SYNC:    /* NOP: QEMU 同步执行 */
        // GICv4 虚拟化命令：
        case GITS_CMD_VMAPTI:  result = process_vmapti(s, cmdpkt, false);
        case GITS_CMD_VMAPP:   result = process_vmapp(s, cmdpkt);
        case GITS_CMD_VMOVI:   result = process_vmovi(s, cmdpkt);
        case GITS_CMD_VINVALL: result = process_vinvall(s, cmdpkt);
        }

        // 3. 推进读指针或停滞
        if (result != CMD_STALL) {
            rd_offset = (rd_offset + 1) % num_entries;
            CREADR.OFFSET = rd_offset;
        } else {
            CREADR.STALLED = 1;  // 命令失败，停止队列
            break;
        }
    }
}
```

### 4.3 命令返回值

```c
// CMD_CONTINUE_OK:  成功，推进读指针
// CMD_CONTINUE:     参数错误但不停滞（跳过），推进读指针
// CMD_STALL:        内存故障，停滞队列
```

---

## 5. ITS 命令详解

### 5.1 MAPD — 映射设备

```c
// arm_gicv3_its.c:822-848 — process_mapd()
// 创建或删除设备表条目
// 输入: DeviceID, ITT 基址, ITT 大小, Valid 位
// 效果: DT[DeviceID] = {valid, ittaddr, size}
```

### 5.2 MAPC — 映射集合

```c
// arm_gicv3_its.c:761-786 — process_mapc()
// 创建或删除集合表条目
// 输入: ICID (集合ID), 目标 CPU (RDbase), Valid 位
// 效果: CT[ICID] = {valid, rdbase}
```

### 5.3 MAPTI/MAPI — 映射中断翻译

```c
// arm_gicv3_its.c:579-647 — process_mapti()
// 创建中断翻译条目
// 输入: DeviceID, EventID, pINTID (LPI), ICID
// MAPI: pINTID = EventID（EventID 即 INTID）
// MAPTI: pINTID 显式指定
// 效果: ITT[DeviceID][EventID] = {intid, icid, inttype=PHYSICAL}
```

### 5.4 INT — 触发中断

```c
// arm_gicv3_its.c:556-577 — process_its_cmd(cmd=INTERRUPT)
// 软件命令触发中断（等价于 TRANSLATER 写入）
// 流程: DeviceID + EventID → lookup ITT → INTID + ICID
//       → lookup CT → target CPU → gicv3_redist_process_lpi()
```

### 5.5 INV — 失效单个 LPI 缓存

```c
// arm_gicv3_its.c:1186-1240 — process_inv()
// 通知重分发器重新读取 LPI 配置表中指定 LPI 的条目
// 用于 Guest 修改 LPI 优先级/使能后通知 GIC
// 效果: gicv3_redist_inv_lpi(cs, intid)
```

### 5.6 INVALL — 失效全部 LPI 缓存

```c
// arm_gicv3_its.c:1352-1364 — INVALL 处理
// 全部重分发器重新扫描 LPI：
for (i = 0; i < s->gicv3->num_cpu; i++) {
    gicv3_redist_update_lpi(&s->gicv3->cpu[i]);
}
```

### 5.7 MOVI — 移动中断

```c
// arm_gicv3_its.c:885-931 — process_movi()
// 将 LPI 从一个集合移到另一个（改变目标 CPU）
// 流程: 查找 ITT 条目 → 更新 ICID → 移动 pending 状态
//       gicv3_redist_mov_lpi(src, dest, intid)
```

### 5.8 MOVALL — 移动所有 LPI

```c
// arm_gicv3_its.c:850-883 — process_movall()
// 将一个 CPU 的所有 pending LPI 移到另一个 CPU
// 用于 CPU 热迁移场景
// 效果: gicv3_redist_movall_lpis(src, dest)
```

### 5.9 GICv4 虚拟化命令

| 命令 | 函数 | 行号 | 功能 |
|------|------|------|------|
| VMAPTI | `process_vmapti` | 649-726 | 映射虚拟中断翻译 |
| VMAPP | `process_vmapp` | 966-1010 | 映射虚拟 PE |
| VMOVP | `process_vmovp` | 1056-1082 | 移动虚拟 PE |
| VMOVI | `process_vmovi` | 1084-1161 | 移动虚拟中断 |
| VINVALL | `process_vinvall` | 1163-1184 | 失效虚拟 LPI |

---

## 6. MSI → ITS → LPI 完整流程

### 6.1 GITS_TRANSLATER 写入

```c
// arm_gicv3_its.c:1552-1576 — gicv3_its_translation_write()
static MemTxResult gicv3_its_translation_write(void *opaque, hwaddr offset,
                                               uint64_t data, unsigned size,
                                               MemTxAttrs attrs)
{
    // MSI 写入 GITS_TRANSLATER:
    //   data = EventID
    //   attrs.requester_id = DeviceID（PCIe BDF）
    if (offset == GITS_TRANSLATER && (s->ctlr & ENABLED)) {
        do_process_its_cmd(s, attrs.requester_id, data, NONE);
    }
}
```

### 6.2 翻译流程

```c
// arm_gicv3_its.c:514-554 — do_process_its_cmd()
static ItsCmdResult do_process_its_cmd(GICv3ITSState *s, uint32_t devid,
                                       uint32_t eventid, ItsCmdType cmd)
{
    // 1. 查找 ITT 条目：DeviceID → DT → ITT → ITEntry
    cmdres = lookup_ite(s, devid, eventid, &ite, &dte);

    // 2. 确定 irqlevel（INT/TRANSLATER=1, CLEAR/DISCARD=0）
    irqlevel = (cmd == CLEAR || cmd == DISCARD) ? 0 : 1;

    // 3. 按类型分派
    switch (ite.inttype) {
    case ITE_INTTYPE_PHYSICAL:
        // 物理 LPI → 查集合表 → 投递到目标 CPU
        return process_its_cmd_phys(s, &ite, irqlevel);
    case ITE_INTTYPE_VIRTUAL:
        // 虚拟 LPI → 查 vPE 表 → 投递到目标 vPE
        return process_its_cmd_virt(s, &ite, irqlevel);
    }
}

// arm_gicv3_its.c:465-477 — process_its_cmd_phys()
static ItsCmdResult process_its_cmd_phys(GICv3ITSState *s, const ITEntry *ite,
                                         int irqlevel)
{
    // 查集合表：ICID → CTEntry → rdbase（目标 CPU）
    lookup_cte(s, ite->icid, &cte);
    // 投递到目标 CPU 的重分发器
    gicv3_redist_process_lpi(&s->gicv3->cpu[cte.rdbase], ite->intid, irqlevel);
}
```

### 6.3 完整 MSI→LPI 端到端流程

```
1. PCIe 设备发起 MSI-X 写入                     [设备 DMA]
   写 GITS_TRANSLATER: data=EventID, attrs.requester_id=DeviceID

2. ITS 接收翻译请求                              [arm_gicv3_its.c:1552-1576]
   gicv3_its_translation_write()
     → do_process_its_cmd(devid, eventid, NONE)

3. ITS 查设备表                                  [arm_gicv3_its.c:213-240]
   DT[DeviceID] → DTEntry{ittaddr, size}

4. ITS 查中断翻译表                              [arm_gicv3_its.c:249-279]
   ITT[EventID] → ITEntry{intid, icid, inttype}

5. ITS 查集合表                                  [arm_gicv3_its.c:180-205]
   CT[ICID] → CTEntry{rdbase}

6. ITS 投递到重分发器                            [arm_gicv3_its.c:465-477]
   process_its_cmd_phys()
     → gicv3_redist_process_lpi(&cpu[rdbase], intid, 1)

7. 重分发器处理 LPI                              [arm_gicv3_redist.c:897-911]
   gicv3_redist_process_lpi()
     检查 ENABLE_LPIS、INTID 范围
     → gicv3_redist_lpi_pending(cs, irq, 1)

8. 更新 LPI pending 表                           [arm_gicv3_redist.c:869-895]
   gicv3_redist_lpi_pending()
     写 Guest 内存中的 pending 表位
     比较优先级 → 更新 hpplpi
     → gicv3_redist_update(cs)
       → gicv3_cpuif_update(cs)

9. CPU 接口信号 CPU                              [arm_gicv3_cpuif.c:1046-1102]
   gicv3_cpuif_update()
     LPI 始终为 G1NS → IRQ 或 FIQ（按安全状态）
     → qemu_set_irq(parent_irq, 1)

10. CPU 应答                                     [arm_gicv3_cpuif.c:1285-1311]
    CPU 读 ICC_IAR1_EL1
      icc_activate_irq(cs, intid)
        → gicv3_redist_lpi_pending(cs, irq, 0)  // 清除 pending
      返回 LPI INTID

11. CPU EOI                                      [arm_gicv3_cpuif.c:1645-1714]
    写 ICC_EOIR1_EL1
      icc_drop_prio() + icc_deactivate_irq()
```

---

## 7. LPI 配置与 Pending 表

### 7.1 LPI 配置表（GICR_PROPBASER）

```c
// arm_gicv3_redist.c:99-127 — update_for_one_lpi()
// LPI 配置表中每个条目 1 字节：
//   bit[0]:   Enable（是否使能）
//   bit[7:2]: Priority（优先级，6 位有效）

static void update_for_one_lpi(GICv3CPUState *cs, int irq,
                               uint64_t ctbase, bool ds, PendingIrq *hpp)
{
    // 从 Guest 内存读取配置字节
    lpite = read_from(ctbase + (irq - 8192));

    if (!(lpite & LPI_CTE_ENABLED)) return;  // 未使能

    if (ds) prio = lpite & PRIORITY_MASK;
    else    prio = ((lpite & PRIORITY_MASK) >> 1) | 0x80;  // NS 优先级移位

    // LPI 始终为 G1NS
    if (prio < hpp->prio) {
        hpp->irq = irq;
        hpp->prio = prio;
        hpp->grp = GICV3_G1NS;
    }
}
```

### 7.2 LPI Pending 表（GICR_PENDBASER）

```c
// arm_gicv3_redist.c:72-83 — pending 表位操作
// 每个 LPI 1 位，表示 pending 状态
// 表基址由 GICR_PENDBASER 指定
// set_pending_table_bit(cs, baddr, irq, level)
//   读取 → 修改位 → 写回 Guest 内存
```

### 7.3 LPI 重新扫描

```c
// arm_gicv3_redist.c:837-867 — gicv3_redist_update_lpi_only()
void gicv3_redist_update_lpi_only(GICv3CPUState *cs)
{
    // 全量扫描 LPI pending 表：
    // 对每个 pending LPI：
    //   读取配置表获取优先级和使能位
    //   与当前 hpplpi 比较
    //   更新最高优先级 LPI
    if (!(GICR_CTLR & ENABLE_LPIS)) return;

    lpipt_baddr = GICR_PENDBASER & PHYADDR_MASK;
    lpict_baddr = GICR_PROPBASER & PHYADDR_MASK;
    update_for_all_lpis(cs, lpipt_baddr, lpict_baddr, idbits, ds, &cs->hpplpi);
}
```

### 7.4 LPI ID 范围

```c
// arm_gicv3_common.h:40
#define GICV3_LPI_INTID_START 8192

// LPI INTID 范围：8192 ~ (2^(IDbits+1) - 1)
// IDbits 由 GICR_PROPBASER.IDbits 和 GICD_TYPER.IDbits 的最小值决定
// arm_gicv3_its.c:96-106 — intid_in_lpi_range() 检查范围
```

---

## 8. LPI 特殊处理

### 8.1 LPI vs SPI/PPI/SGI 对比

| 特性 | SPI (32-1019) | PPI (16-31) | SGI (0-15) | LPI (8192+) |
|------|-------------|-------------|------------|-------------|
| 状态存储 | GICD 位图 | GICR 寄存器 | GICR 寄存器 | Guest 内存表 |
| 配置存储 | GICD 寄存器 | GICR 寄存器 | GICR 寄存器 | Guest 内存表 |
| 组 | 可配置 | 可配置 | 可配置 | **始终 G1NS** |
| 触发模式 | 边沿/电平 | 边沿/电平 | 边沿 | **始终边沿** |
| Active 位 | 有 | 有 | 有 | **无** |
| EOI 行为 | 清 active | 清 active | 清 active | 清 pending 表位 |
| 路由 | GICD_IROUTER | Per-CPU | Per-CPU | ITS 集合表 |

### 8.2 LPI 无 Active 状态

```c
// arm_gicv3_cpuif.c:1158-1187 — icc_activate_irq()
if (irq >= GICV3_LPI_INTID_START) {
    // LPI: 没有 active 位，直接清除 pending
    gicv3_redist_lpi_pending(cs, irq, 0);
    // APR 位仍然会被设置（用于优先级追踪）
}
```

---

## 9. vLPI 与 GICv4 虚拟化

### 9.1 虚拟 LPI 流程

```c
// arm_gicv3_its.c:479-504 — process_its_cmd_virt()
// ITEntry.inttype == ITE_INTTYPE_VIRTUAL 时：
// 1. 查 vPE 表：vpeid → VTEntry{rdbase, vptaddr}
// 2. 投递到重分发器的虚拟 LPI 处理
gicv3_redist_process_vlpi(&cpu[vte.rdbase], ite->intid,
                          vte.vptaddr << 16, ite->doorbell, irqlevel);
```

### 9.2 vLPI Pending 与 Doorbell

```c
// arm_gicv3_redist.c:1024-1085 — gicv3_redist_process_vlpi()
// 如果目标 vPE 当前在此 CPU 上运行：
//   直接更新虚拟 pending 表
//   触发虚拟 CPU 接口更新
// 否则：
//   设置 vLPI pending 位
//   如果有 doorbell → 触发 doorbell 中断唤醒 Hypervisor
```

---

## 10. KVM 直通 ITS

```c
// arm_gicv3_its_kvm.c:45-67 — kvm_its_send_msi()
// KVM 模式下，MSI 直接通过 KVM ioctl 注入
// 绕过 QEMU 软件 ITS 翻译
// 用于硬件 GICv3 直通场景

// arm_gicv3_its_kvm.c:136-198 — 状态保存/恢复
// 迁移时保存/恢复 ITS 寄存器和表状态
```

---

## 11. 小结

| 组件 | 关键函数 | 源文件:行号 |
|------|---------|-----------|
| **MSI 入口** | `gicv3_its_translation_write` | arm_gicv3_its.c:1552-1576 |
| **翻译核心** | `do_process_its_cmd` | arm_gicv3_its.c:514-554 |
| **物理 LPI 投递** | `process_its_cmd_phys` | arm_gicv3_its.c:465-477 |
| **虚拟 LPI 投递** | `process_its_cmd_virt` | arm_gicv3_its.c:479-504 |
| **命令队列** | `process_cmdq` | arm_gicv3_its.c:1246-1408 |
| **表地址计算** | `table_entry_addr` | arm_gicv3_its.c:129-172 |
| **MAPD** | `process_mapd` | arm_gicv3_its.c:822-848 |
| **MAPC** | `process_mapc` | arm_gicv3_its.c:761-786 |
| **MAPTI** | `process_mapti` | arm_gicv3_its.c:579-647 |
| **MOVI** | `process_movi` | arm_gicv3_its.c:885-931 |
| **INV/INVALL** | `process_inv` | arm_gicv3_its.c:1186-1240 |
| **LPI 处理** | `gicv3_redist_process_lpi` | arm_gicv3_redist.c:897-911 |
| **LPI pending** | `gicv3_redist_lpi_pending` | arm_gicv3_redist.c:869-895 |
| **LPI 扫描** | `gicv3_redist_update_lpi_only` | arm_gicv3_redist.c:837-867 |
| **LPI 配置读取** | `update_for_one_lpi` | arm_gicv3_redist.c:99-127 |
| **LPI 移动** | `gicv3_redist_mov_lpi` | arm_gicv3_redist.c:924-1022 |

**核心设计原则：**
1. **三级翻译**：DeviceID→设备表→ITT→集合表→目标 CPU，灵活支持 PCIe 拓扑
2. **Guest 内存表**：LPI 配置和 pending 直接存储在 Guest 内存中，避免 GIC 寄存器爆炸
3. **命令队列异步**：Guest 通过环形缓冲区批量提交命令（QEMU 同步执行简化实现）
4. **增量更新**：单个 LPI 变化时优先增量比较（`gicv3_redist_check_lpi_priority`），避免全表扫描
5. **SYNC 为 NOP**：QEMU 同步执行所有命令，无需显式同步
6. **LPI 无 Active 位**：简化状态机，EOI 直接清 pending 表
