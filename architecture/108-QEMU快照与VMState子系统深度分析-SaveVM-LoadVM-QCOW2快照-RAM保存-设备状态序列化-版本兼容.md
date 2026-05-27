# Doc 108: QEMU 快照与 VMState 子系统深度分析

## 文档信息
- **组件**: Snapshot, VMState, SaveVM/LoadVM, QCOW2 快照, RAM 迁移
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: architecture/

---

## 目录
1. [子系统概述](#1-子系统概述)
2. [VMState 数据驱动序列化框架](#2-vmstate-数据驱动序列化框架)
3. [SaveVM/LoadVM 完整流程](#3-savevmloadvm-完整流程)
4. [RAM 保存与恢复](#4-ram-保存与恢复)
5. [Block 层快照支持](#5-block-层快照支持)
6. [ARM64 CPU 状态序列化](#6-arm64-cpu-状态序列化)
7. [设备状态序列化模式](#7-设备状态序列化模式)
8. [快照类型与使用](#8-快照类型与使用)
9. [QMP/HMP 命令接口](#9-qmphmp-命令接口)
10. [版本兼容机制](#10-版本兼容机制)
11. [设计模式与限制](#11-设计模式与限制)
12. [源码索引](#12-源码索引)

---

## 1. 子系统概述

### 1.1 快照的本质

QEMU 快照 = **VM 全量状态的持久化存储**, 包含:
- 所有 vCPU 的寄存器状态
- 所有 RAM 内容
- 所有设备的内部状态
- 磁盘镜像的某个时间点状态

### 1.2 架构分层

```
┌─────────────────────────────────────────────────────┐
│                 HMP/QMP 命令层                        │
│   savevm / loadvm / delvm / snapshot-save/load      │
├─────────────────────────────────────────────────────┤
│              SaveVM 协调层 (savevm.c)                 │
│   qemu_savevm_state() / qemu_loadvm_state()         │
├───────────────┬───────────────┬─────────────────────┤
│   RAM 保存    │   设备状态     │    Block 快照        │
│  (ram.c)      │  (vmstate.c)  │  (snapshot.c)       │
├───────────────┼───────────────┼─────────────────────┤
│  Dirty Bitmap │ VMStateDesc   │ QCOW2 Snapshot      │
│  Zero Detect  │ 字段遍历      │ Table               │
│  Compression  │ 版本控制      │ bdrv_snapshot_*     │
└───────────────┴───────────────┴─────────────────────┘
```

### 1.3 核心源文件

| 文件 | 职责 |
|------|------|
| `migration/savevm.c` | 快照协调, save/load 主流程 |
| `migration/vmstate.c` | VMState 序列化/反序列化引擎 |
| `migration/vmstate-types.c` | 类型处理器 (uint8/32/64/buffer 等) |
| `migration/ram.c` | RAM 页面保存/恢复 |
| `include/migration/vmstate.h` | VMStateDescription/Field 定义 |
| `include/migration/snapshot.h` | 快照 API 声明 |
| `block/snapshot.c` | Block 层快照统一封装 |
| `block/qcow2-snapshot.c` | QCOW2 内部快照实现 |
| `target/arm/machine.c` | ARM CPU 状态注册 |

---

## 2. VMState 数据驱动序列化框架

### 2.1 核心数据结构

#### VMStateDescription (源码: `vmstate.h:227-260`)

```c
struct VMStateDescription {
    const char *name;                    // 设备/状态名 (唯一标识)
    bool unmigratable;                   // 标记为不可迁移
    bool early_setup;                    // 早期设置标志
    int version_id;                      // 当前版本
    int minimum_version_id;              // 最低兼容版本
    MigrationPriority priority;          // 保存优先级

    // 生命周期回调:
    int (*pre_load)(void *opaque);       // 加载前准备
    int (*pre_save)(void *opaque);       // 保存前准备
    int (*post_load)(void *opaque, int version_id);  // 加载后修复
    int (*post_save)(void *opaque);      // 保存后清理

    // 条件判断:
    bool (*needed)(void *opaque);        // subsection 是否需要发送
    bool (*dev_unplug_pending)(void *opaque); // 热拔标志

    // 字段描述:
    const VMStateField *fields;          // 字段数组 (核心!)
    const VMStateDescription * const *subsections; // 子段落
};
```

#### VMStateField (源码: `vmstate.h:196-225`)

```c
struct VMStateField {
    const char *name;           // 字段名
    size_t offset;              // 在结构体中的偏移
    size_t size;                // 字段大小
    int num;                    // 数组元素数 (固定)
    size_t num_offset;          // 数组大小在结构体中的偏移 (动态)
    const VMStateInfo *info;    // 类型处理器 (read/write)
    enum VMStateFlags flags;    // 标志位
    const VMStateDescription *vmsd;  // 嵌套结构描述
    int version_id;             // 字段引入版本
    bool (*field_exists)(void *opaque, int version_id); // 条件存在
};
```

### 2.2 VMSTATE 宏体系

QEMU 通过宏系统实现声明式序列化:

```c
// 基本类型:
VMSTATE_UINT8(field, struct)          // uint8_t 字段
VMSTATE_UINT32(field, struct)         // uint32_t 字段
VMSTATE_UINT64(field, struct)         // uint64_t 字段
VMSTATE_INT32(field, struct)          // int32_t 字段
VMSTATE_BOOL(field, struct)           // bool 字段

// 数组:
VMSTATE_UINT32_ARRAY(field, struct, n)       // 固定大小数组
VMSTATE_VARRAY_UINT32(field, struct, n_field, ver, info, type) // 变长数组

// 结构体嵌套:
VMSTATE_STRUCT(field, struct, ver, vmsd, type)       // 嵌套结构
VMSTATE_STRUCT_ARRAY(field, struct, n, ver, vmsd, type)

// 指针:
VMSTATE_POINTER(field, struct, ver, info, type)

// 缓冲区:
VMSTATE_VBUFFER(field, struct, ver, NULL, size_field)
VMSTATE_BITMAP(field, struct, ver, size_field)

// 容器:
VMSTATE_QTAILQ_V(field, struct, ver, vmsd, type, next_field)
VMSTATE_GTREE_V(field, struct, ver, vmsd_key, vmsd_val)

// 定时器:
VMSTATE_TIMER_PTR(field, struct)

// 终止标记:
VMSTATE_END_OF_LIST()
```

源码: `vmstate.h:382-418, 549-818, 885-1015`

### 2.3 序列化引擎

#### 保存流程 (源码: `vmstate.c:567-787`)

```c
vmstate_save_vmsd_v(f, opaque, vmsd, vmdesc, version_id):
    1. 调用 pre_save(opaque)
    2. for each field in vmsd->fields:
        a. 检查 field_exists() (版本条件)
        b. 根据 flags 处理:
           - VMS_SINGLE: 保存单值
           - VMS_ARRAY: 遍历保存 n 个元素
           - VMS_POINTER: 解引用后保存
           - VMS_STRUCT: 递归调用 vmstate_save_vmsd_v()
           - VMS_VSTRUCT: 虚拟结构 (偏移计算)
        c. field->info->put(f, addr, size, field, vmdesc_loop)
    3. 处理 subsections:
        for each subsection where needed() == true:
            写入 QEMU_VM_SUBSECTION 标记
            vmstate_save_vmsd_v(f, opaque, subsection)
    4. 调用 post_save(opaque)
```

#### 加载流程 (源码: `vmstate.c:313-409`)

```c
vmstate_load_vmsd(f, opaque, vmsd, version_id):
    1. 版本检查: version_id > vmsd->version_id → "too new"
    2. 版本检查: version_id < vmsd->minimum_version_id → "too old"
    3. 调用 pre_load(opaque)
    4. for each field in vmsd->fields:
        a. 检查 field_exists()
        b. vmstate_handle_alloc() (动态分配)
        c. field->info->get(f, addr, size, field)  // 读取
    5. subsection_load() → 加载子段落
    6. 调用 post_load(opaque, version_id)
```

### 2.4 设备注册

```c
// 现代方式 (推荐): 在 DeviceClass 中声明
static const VMStateDescription vmstate_my_device = {
    .name = "my_device",
    .version_id = 2,
    .minimum_version_id = 1,
    .fields = (VMStateField[]) {
        VMSTATE_UINT32(reg_a, MyDeviceState),
        VMSTATE_UINT32(reg_b, MyDeviceState),
        VMSTATE_END_OF_LIST()
    }
};

// 注册: DeviceClass->vmsd = &vmstate_my_device;

// 手动注册 (旧式):
vmstate_register_with_alias_id(NULL, instance_id, &vmsd, opaque, ...);
// 源码: migration/savevm.c:925-977
```

---

## 3. SaveVM/LoadVM 完整流程

### 3.1 流格式 (Wire Format)

源码: `migration/savevm.h:19-33`

```
┌──────────────────────────────────────────┐
│ Magic: QEMU_VM_FILE_MAGIC (0x5145564d)   │
│ Version: QEMU_VM_FILE_VERSION            │
├──────────────────────────────────────────┤
│ Section: QEMU_VM_CONFIGURATION           │  ← VM 配置信息
├──────────────────────────────────────────┤
│ Section: QEMU_VM_SECTION_START (设备A)    │  ← 设备 A setup
│ Section: QEMU_VM_SECTION_PART  (设备A)    │  ← 设备 A iterate
│ ...                                       │
│ Section: QEMU_VM_SECTION_END   (设备A)    │  ← 设备 A complete
├──────────────────────────────────────────┤
│ Section: QEMU_VM_SECTION_FULL  (设备B)    │  ← 设备 B 一次性
├──────────────────────────────────────────┤
│ Section: QEMU_VM_VMDESCRIPTION            │  ← JSON 设备描述
├──────────────────────────────────────────┤
│ Section: QEMU_VM_EOF                      │  ← 结束标记
└──────────────────────────────────────────┘
```

Section 类型:
- `START` (1): 设备首次出现, 含名称/实例ID/版本
- `PART` (2): 设备增量数据 (iterate)
- `END` (3): 设备最终数据 (complete)
- `FULL` (4): 设备一次性完整数据
- `SUBSECTION` (5): VMState 子段落
- `VMDESCRIPTION` (6): JSON 格式设备描述
- `CONFIGURATION` (7): VM 配置
- `COMMAND` (8): 迁移命令

### 3.2 save_snapshot() 完整流程

源码: `migration/savevm.c:3202-3504`

```
save_snapshot(name, overwrite, vmstate, errp):
    │
    ├─ 1. migrate_can_snapshot()           // 检查是否可快照
    ├─ 2. bdrv_all_can_snapshot()          // 所有 block 设备支持快照
    ├─ 3. 处理同名快照 (overwrite 参数)
    ├─ 4. bdrv_all_find_vmstate_bs()       // 找到存储 vmstate 的设备
    ├─ 5. vm_stop(RUN_STATE_SAVE_VM)       // 暂停 VM ← 关键!
    │
    ├─ 6. qemu_savevm_state(f):            // 核心保存
    │      ├─ migrate_init()
    │      ├─ qemu_savevm_state_header()   // 写 magic + version
    │      ├─ qemu_savevm_state_do_setup() // 各设备 setup (RAM dirty track)
    │      ├─ loop: qemu_savevm_state_iterate()  // 增量发送 (RAM)
    │      ├─ qemu_savevm_state_complete_precopy() // 最终完成
    │      └─ qemu_savevm_state_cleanup()
    │
    ├─ 7. bdrv_all_create_snapshot(sn)     // 所有磁盘创建快照
    │      └─ 对 vmstate_bs: sn->vm_state_size = 实际写入大小
    │
    └─ 8. vm_resume() (如果之前在运行)
```

### 3.3 load_snapshot() 完整流程

源码: `migration/savevm.c:3398-3482`

```
load_snapshot(name, vmstate, errp):
    │
    ├─ 1. 查找快照名在所有设备中存在
    ├─ 2. bdrv_all_find_vmstate_bs()       // 找 vmstate 存储设备
    ├─ 3. 检查 sn.vm_state_size != 0      // 纯磁盘快照不能恢复 VM
    │
    ├─ 4. bdrv_all_goto_snapshot(sn)       // 所有磁盘回到快照点
    │
    └─ 5. qemu_loadvm_state(f):            // 核心加载
           ├─ qemu_savevm_state_blocked()  // 检查阻塞
           ├─ qemu_loadvm_state_header()   // 验证 magic + version
           ├─ qemu_loadvm_state_setup()    // 设备 setup
           ├─ cpu_synchronize_all_pre_loadvm() // CPU 同步
           └─ qemu_loadvm_state_main()     // 逐 section 加载
```

### 3.4 SaveVM Handler 注册与遍历

```c
// 全局 handler 列表:
static SaveState savevm_state = {
    .handlers = QTAILQ_HEAD_INITIALIZER(...)
};

// 遍历所有 handler 保存:
qemu_savevm_state_complete_precopy():
    QTAILQ_FOREACH(se, &savevm_state.handlers, entry) {
        if (se->ops->save_state)
            se->ops->save_state(f, se->opaque);   // 旧式
        else if (se->vmsd)
            vmstate_save_state(f, se->vmsd, ...);  // 新式 VMState
    }
```

---

## 4. RAM 保存与恢复

### 4.1 RAM 保存三阶段

源码: `migration/ram.c`

| 阶段 | 函数 | 功能 |
|------|------|------|
| Setup | `ram_save_setup()` (L3112-3210) | 初始化 dirty bitmap, 记录所有 RAM block |
| Iterate | `ram_save_iterate()` (L3255-3356) | 增量发送脏页 |
| Complete | `ram_save_complete()` (L3368-3443) | 最终同步所有剩余脏页 |

### 4.2 脏页追踪机制

```
┌─────────────────────────────────────────────┐
│            KVM / TCG 脏页追踪                │
│  (硬件 dirty bit 或软件 bitmap)              │
├─────────────────────────────────────────────┤
│        physical_memory_sync_dirty_bitmap()   │  ← 同步到 QEMU bitmap
│        (ram.c:940-1009)                      │
├─────────────────────────────────────────────┤
│        ram_find_and_save_block()             │  ← 找脏页并发送
│        migration_bitmap_clear_dirty()        │  ← 发送后清除标记
│        (ram.c:829-940)                       │
└─────────────────────────────────────────────┘
```

### 4.3 页面优化

#### 零页检测 (源码: `ram.c:1213-1253`)

```c
save_zero_page(rs, pss, offset):
    if (buffer_is_zero(p, TARGET_PAGE_SIZE)):
        // 只发送一个标记, 不发送 4K 数据
        // 恢复时填零
        return 1;  // 成功
```

#### 压缩 (可选)

- XBZRLE: 增量压缩 (XOR + ZRLE 编码)
- multifd + zlib/zstd: 多线程压缩传输

### 4.4 快照场景下的 RAM

对于本地快照 (savevm), RAM 保存相对简单:
- 无需增量迭代 (VM 已暂停)
- `ram_save_complete()` 一次性写出所有内存
- 零页检测仍有效 (减少存储大小)
- 数据写入 vmstate block device (通常是 QCOW2 文件)

---

## 5. Block 层快照支持

### 5.1 统一 API

源码: `block/snapshot.c:216-385`

```c
// 创建快照:
int bdrv_snapshot_create(BlockDriverState *bs,
                         QEMUSnapshotInfo *sn_info);

// 跳转到快照:
int bdrv_snapshot_goto(BlockDriverState *bs,
                       const char *snapshot_id, Error **errp);

// 删除快照:
int bdrv_snapshot_delete(BlockDriverState *bs,
                         const char *snapshot_id, Error **errp);
```

### 5.2 批量操作

源码: `block/snapshot.c:482-785`

```c
// 所有设备是否支持快照:
bool bdrv_all_can_snapshot(bool has_devices, ...);

// 所有设备创建快照:
int bdrv_all_create_snapshot(QEMUSnapshotInfo *sn, ...);

// 所有设备跳转到快照:
int bdrv_all_goto_snapshot(const char *name, ...);

// 所有设备删除快照:
int bdrv_all_delete_snapshot(const char *name, ...);

// 找到存储 VM state 的设备:
BlockDriverState *bdrv_all_find_vmstate_bs(...);
```

### 5.3 QCOW2 内部快照实现

源码: `block/qcow2-snapshot.c`

#### 快照表结构

```
QCOW2 文件布局:
┌─────────────────────┐
│ Header              │
├─────────────────────┤
│ L1 Table (active)   │  ← 当前活跃的映射
├─────────────────────┤
│ L2 Tables           │
├─────────────────────┤
│ Data Clusters       │
├─────────────────────┤
│ Snapshot Table      │  ← 快照元数据
│  ├─ Snapshot 1      │
│  │   ├─ name       │
│  │   ├─ L1 copy    │  ← 快照时刻的 L1 表副本
│  │   ├─ vm_state_size │  ← VM 状态数据大小
│  │   └─ disk_size   │
│  └─ Snapshot 2      │
├─────────────────────┤
│ VM State Data       │  ← qemu_savevm_state() 写入的字节流
└─────────────────────┘
```

#### 创建快照 (COW 语义)

```
bdrv_snapshot_create (qcow2):
    1. 复制当前 L1 表 → 快照的 L1 副本
    2. 所有引用的 cluster 增加引用计数
    3. 在 snapshot table 中添加条目
    4. 后续写操作触发 COW: 写新 cluster, 旧 cluster 属于快照
```

#### 恢复快照

```
bdrv_snapshot_goto (qcow2):
    1. 释放当前 L1 表引用的所有 cluster
    2. 将快照的 L1 副本恢复为活跃 L1
    3. 增加快照 cluster 的引用计数
    4. 重建 L2 cache
```

### 5.4 VM State 在 QCOW2 中的位置

- VM state 数据由 `qemu_savevm_state()` 写入 **vmstate block device**
- `bdrv_all_find_vmstate_bs()` 确定哪个 QCOW2 文件存储 vmstate
- 快照表中 `vm_state_size` 记录字节流长度
- 恢复时从该偏移读回并输入 `qemu_loadvm_state()`

源码: `block/snapshot.c:695-733` (`bdrv_all_create_snapshot`)

---

## 6. ARM64 CPU 状态序列化

### 6.1 vmstate_arm_cpu 主结构

源码: `target/arm/machine.c:1229-1328`

```c
const VMStateDescription vmstate_arm_cpu = {
    .name = "cpu",
    .version_id = 22,
    .minimum_version_id = 22,
    .pre_save = cpu_pre_save,
    .post_load = cpu_post_load,
    .fields = (const VMStateField[]) {
        // AArch32 通用寄存器:
        VMSTATE_UINT32_ARRAY(env.regs, ARMCPU, 16),

        // AArch64 通用寄存器:
        VMSTATE_UINT64_ARRAY(env.xregs, ARMCPU, 32),

        // 程序计数器:
        VMSTATE_UINT64(env.pc, ARMCPU),

        // 处理器状态:
        VMSTATE_UINT32(env.pstate, ARMCPU),

        // 异常信息:
        VMSTATE_STRUCT(env.exception, ARMCPU, 0, vmstate_exception, ...),

        // Banked 寄存器 (AArch32 各模式):
        VMSTATE_UINT32_ARRAY(env.banked_spsr, ARMCPU, 8),
        VMSTATE_UINT32_ARRAY(env.banked_r13, ARMCPU, 8),
        VMSTATE_UINT32_ARRAY(env.banked_r14, ARMCPU, 8),
        VMSTATE_UINT32_ARRAY(env.usr_regs, ARMCPU, 5),
        VMSTATE_UINT32_ARRAY(env.fiq_regs, ARMCPU, 5),

        // EL1/EL2/EL3 寄存器:
        VMSTATE_UINT64_ARRAY(env.elr_el, ARMCPU, 4),    // ELR_EL1/2/3
        VMSTATE_UINT64_ARRAY(env.sp_el, ARMCPU, 4),     // SP_EL0/1/2/3

        // CP 寄存器 (系统寄存器数组):
        VMSTATE_VARRAY_INT32(cpreg_vmstate_array, ARMCPU,
                             cpreg_vmstate_array_len, 0,
                             vmstate_info_uint64, uint64_t),

        // 定时器:
        VMSTATE_TIMER_PTR(gt_timer[GTIMER_PHYS], ARMCPU),
        VMSTATE_TIMER_PTR(gt_timer[GTIMER_VIRT], ARMCPU),

        // 电源状态:
        VMSTATE_UINT32(power_state, ARMCPU),

        VMSTATE_END_OF_LIST()
    },
    .subsections = (const VMStateDescription * const []) {
        &vmstate_vfp,           // VFP/NEON
        &vmstate_iwmmxt,        // iWMMXt (XScale)
        &vmstate_m,             // M-Profile
        &vmstate_thumb2ee,      // ThumbEE
        &vmstate_pmsav7,        // PMSAv7
        &vmstate_pmsav8,        // PMSAv8
        &vmstate_m_security,    // M-Profile Security
        &vmstate_sve,           // SVE
        &vmstate_serror,        // SError
        &vmstate_irq_line_state,
        &vmstate_za,            // SME ZA
        &vmstate_zt0,           // SME2 ZT0
        &vmstate_pstate64,      // AArch64 PSTATE (兼容拆分)
        NULL
    }
};
```

### 6.2 SVE 状态 (条件保存)

源码: `target/arm/machine.c:232-260`

```c
static bool sve_needed(void *opaque) {
    ARMCPU *cpu = opaque;
    return cpu_isar_feature(aa64_sve, cpu);  // 仅 SVE CPU 保存
}

static const VMStateDescription vmstate_sve = {
    .name = "cpu/sve",
    .needed = sve_needed,
    .fields = (VMStateField[]) {
        // z0-z31 高位 (超出 128-bit 的部分):
        VMSTATE_STRUCT_ARRAY(env.vfp.zregs, ARMCPU, 32, 0,
                             vmstate_zreg_hi_reg, ARMVectorReg),
        // p0-p15 谓词寄存器:
        VMSTATE_STRUCT_ARRAY(env.vfp.pregs, ARMCPU, 17, 0,
                             vmstate_preg_reg, ARMPredicateReg),
        VMSTATE_END_OF_LIST()
    }
};
```

### 6.3 VFP/NEON 状态

源码: `target/arm/machine.c:139-225`

```c
static const VMStateDescription vmstate_vfp = {
    .name = "cpu/vfp",
    .fields = (VMStateField[]) {
        // v0-v31 低 128-bit:
        VMSTATE_UINT64_ARRAY(env.vfp.zregs[0..31].d[0..1], ...),
        // FPSCR 拆分为 FPCR + FPSR (兼容):
        VMSTATE_UINT32(env.vfp.xregs[ARM_VFP_FPSID], ARMCPU),
        ...
        VMSTATE_END_OF_LIST()
    }
};
```

### 6.4 系统寄存器 (CP Reg) 保存策略

ARM64 有数百个系统寄存器, QEMU 使用动态数组方式:

```c
// pre_save 时:
cpu_pre_save():
    1. 遍历所有 CPRegInfo (哈希表)
    2. 过滤出需要迁移的 (ARM_CP_NO_RAW 除外)
    3. 排序后存入 cpreg_vmstate_array[]
    4. 设置 cpreg_vmstate_array_len

// 保存: VMSTATE_VARRAY_INT32 写出变长数组

// post_load 时:
cpu_post_load():
    1. 从 cpreg_vmstate_array[] 恢复所有系统寄存器
    2. 调用 write_raw 将值写回 CPRegInfo
    3. 重建 hflags, 刷新 TLB
```

---

## 7. 设备状态序列化模式

### 7.1 典型设备 VMState 示例

```c
// GIC (简化):
static const VMStateDescription vmstate_gicv3 = {
    .name = "arm_gicv3",
    .version_id = 1,
    .fields = (VMStateField[]) {
        VMSTATE_UINT32(num_irq, GICv3State),
        VMSTATE_UINT32(gicd_ctlr, GICv3State),
        VMSTATE_STRUCT_VARRAY_UINT32(cpu, GICv3State, num_cpu, 0,
                                      vmstate_gicv3_cpu, GICv3CPUState),
        VMSTATE_END_OF_LIST()
    }
};

// VirtIO 通用:
static const VMStateDescription vmstate_virtio = {
    .name = "virtio",
    .fields = (VMStateField[]) {
        VMSTATE_UINT32(device_id, VirtIODevice),
        VMSTATE_UINT8(status, VirtIODevice),
        VMSTATE_UINT8(isr, VirtIODevice),
        VMSTATE_UINT16(queue_sel, VirtIODevice),
        VMSTATE_STRUCT_VARRAY_UINT32(vq, VirtIODevice, nvectors, ...),
        VMSTATE_END_OF_LIST()
    }
};

// PL011 UART:
static const VMStateDescription vmstate_pl011 = {
    .name = "pl011",
    .fields = (VMStateField[]) {
        VMSTATE_UINT32(flags, PL011State),
        VMSTATE_UINT32(int_enabled, PL011State),
        VMSTATE_UINT32(int_level, PL011State),
        VMSTATE_UINT32_ARRAY(read_fifo, PL011State, PL011_FIFO_DEPTH),
        VMSTATE_END_OF_LIST()
    }
};
```

### 7.2 保存优先级

```c
typedef enum MigrationPriority {
    MIG_PRI_LOW,        // 普通设备
    MIG_PRI_DEFAULT,    // 默认
    MIG_PRI_GICV3_ITS, // GIC ITS (依赖 CPU)
    MIG_PRI_MAX,
} MigrationPriority;
```

高优先级设备先保存/后加载, 确保依赖关系正确。

### 7.3 Live 设备 (SaveVMHandlers)

某些设备需要 live migration 支持 (增量发送):

```c
static const SaveVMHandlers ram_handlers = {
    .save_setup     = ram_save_setup,      // 初始化
    .save_live_iterate = ram_save_iterate,  // 增量发送
    .save_live_complete_precopy = ram_save_complete, // 最终完成
    .load_state     = ram_load,            // 加载
    .is_active      = ram_is_active,
};
// 注册: register_savevm_live("ram", 0, &ram_handlers, NULL)
```

---

## 8. 快照类型与使用

### 8.1 内部快照 (Internal Snapshot)

```
特点:
- 存储在 QCOW2 文件内部
- 包含: 磁盘状态 + VM state (CPU/设备/RAM)
- 原子性: 创建时 VM 暂停
- 恢复: 完整回到过去状态
- 空间: COW, 只存储差异

使用:
(qemu) savevm my_snapshot
(qemu) loadvm my_snapshot
(qemu) delvm my_snapshot
(qemu) info snapshots
```

### 8.2 外部快照 (External Snapshot)

```
特点:
- 创建新 QCOW2 文件, 原文件变为 backing
- 不暂停 VM (live snapshot)
- 通常不含 VM state (disk-only)
- 适合备份/分支

使用:
{ "execute": "blockdev-snapshot-sync",
  "arguments": { "device": "drive0",
                 "snapshot-file": "/path/to/new.qcow2" } }
```

### 8.3 对比

| 特性 | 内部快照 | 外部快照 |
|------|----------|----------|
| VM state | ✅ 包含 | ❌ 通常不含 |
| 暂停 VM | ✅ 必须 | ❌ 可 live |
| 存储位置 | 同一文件 | 新文件 + backing |
| 恢复 VM | ✅ 完整恢复 | ❌ 只恢复磁盘 |
| 空间效率 | COW (内部) | COW (backing chain) |
| 嵌套 | 多个快照平级 | 形成链式结构 |
| 适用场景 | 调试/测试 | 备份/生产 |

### 8.4 blockdev-snapshot-sync vs blockdev-snapshot

| 命令 | 行为 |
|------|------|
| `blockdev-snapshot-sync` | 同步创建, 自动完成配置, 旧接口兼容 |
| `blockdev-snapshot` | 更底层, 需要预先 blockdev-add 目标, 更灵活 |

---

## 9. QMP/HMP 命令接口

### 9.1 HMP 命令

源码: `hmp-commands.hx:349-390`

| 命令 | 功能 |
|------|------|
| `savevm [name]` | 创建内部快照 |
| `loadvm name` | 恢复到内部快照 |
| `delvm name` | 删除内部快照 |
| `info snapshots` | 列出所有快照 |

### 9.2 QMP 命令

源码: `qapi/migration.json:2119-2248`

```json
// 保存快照:
{"execute": "snapshot-save",
 "arguments": {"job-id": "snap0", "tag": "my_snapshot",
               "vmstate": "drive0", "devices": ["drive0"]}}

// 加载快照:
{"execute": "snapshot-load",
 "arguments": {"job-id": "load0", "tag": "my_snapshot",
               "vmstate": "drive0", "devices": ["drive0"]}}

// 删除快照:
{"execute": "snapshot-delete",
 "arguments": {"job-id": "del0", "tag": "my_snapshot",
               "devices": ["drive0"]}}
```

### 9.3 命令与内部函数映射

| 命令 | 内部函数 | 源码位置 |
|------|----------|----------|
| savevm | `save_snapshot()` | `savevm.c:3202` |
| loadvm | `load_snapshot()` | `savevm.c:3398` |
| delvm | `delete_snapshot()` | `savevm.c:3492` |
| info snapshots | `bdrv_snapshot_list()` | `snapshot.c` |

---

## 10. 版本兼容机制

### 10.1 版本号

```c
.version_id = 22,         // 当前保存时使用的版本
.minimum_version_id = 22, // 能加载的最低版本
```

- 保存时总是写当前 `version_id`
- 加载时检查: `saved_version >= minimum_version_id`
- 允许新版本加载旧快照, 但旧版本不能加载新快照

### 10.2 Subsections (条件段落)

```c
static bool pauth_needed(void *opaque) {
    ARMCPU *cpu = opaque;
    return cpu_isar_feature(aa64_pauth, cpu);
}

static const VMStateDescription vmstate_pauth = {
    .name = "cpu/pauth",
    .needed = pauth_needed,    // 只在 PAC 启用时保存
    .fields = (VMStateField[]) { ... }
};
```

- `needed()` 返回 true 时才写入该段
- 加载时: 如果流中有该段就加载, 没有则跳过
- 实现向前兼容: 新特性以 subsection 方式添加

### 10.3 field_exists (字段条件)

```c
VMSTATE_UINT64_V(new_field, MyState, 3),  // 仅 version >= 3 时存在
```

- 低版本加载时跳过该字段
- 高版本保存时包含该字段

### 10.4 兼容性矩阵示例

```
Version 20 快照 → Version 22 QEMU: ✅ (22 >= min=22? 需看实际min)
Version 22 快照 → Version 20 QEMU: ❌ (22 > 20的version_id)
Version 22 快照 + pauth subsection → Version 22 QEMU (无PAC): ✅ (subsection 可选)
```

---

## 11. 设计模式与限制

### 11.1 设计模式

| 模式 | 说明 |
|------|------|
| **数据驱动** | 设备只声明结构, 通用引擎处理序列化 |
| **COW 语义** | QCOW2 快照不复制全部数据, 只引用 |
| **增量传输** | RAM 通过 dirty bitmap 只传脏页 |
| **条件保存** | subsection/field_exists 按需保存 |
| **版本演进** | version_id 机制支持平滑升级 |
| **优先级排序** | 设备间依赖通过 priority 处理 |
| **回调钩子** | pre/post save/load 处理复杂逻辑 |

### 11.2 限制

| 限制 | 说明 |
|------|------|
| **unmigratable 设备** | 任何标记 unmigratable 的设备会阻塞整个快照 |
| **磁盘一致性** | 所有 block 设备都必须支持 snapshot |
| **内存大小** | 大内存 VM 的快照需要大量存储和时间 |
| **纯磁盘快照** | `vm_state_size == 0` 的快照不能恢复 VM 运行状态 |
| **版本锁定** | 跨大版本的快照兼容性不保证 |
| **PSTATE 拆分** | ARM FPSCR 等需要复杂的兼容性处理 |
| **热插拔设备** | 动态添加的设备需要 `dev_unplug_pending` 处理 |
| **加密磁盘** | 加密 QCOW2 的快照支持有限制 |

### 11.3 与热迁移的关系

快照和热迁移共享大部分基础设施:

| 组件 | 快照 | 热迁移 |
|------|------|--------|
| VMState 引擎 | ✅ 共用 | ✅ 共用 |
| RAM 保存 | 一次性 complete | 多轮 iterate + complete |
| VM 暂停 | 保存前暂停 | 最后 downtime 暂停 |
| 目标 | 本地文件 | 网络 socket |
| dirty tracking | 只需一次 | 持续追踪 |
| postcopy | ❌ | ✅ (可选) |

---

## 12. 源码索引

| 文件 | 行数 | 关键内容 |
|------|------|----------|
| `include/migration/vmstate.h:196-260` | - | VMStateField + VMStateDescription |
| `include/migration/vmstate.h:382-1015` | - | VMSTATE_* 宏定义 |
| `migration/vmstate.c:313-409` | - | vmstate_load_vmsd() |
| `migration/vmstate.c:567-787` | - | vmstate_save_vmsd_v() |
| `migration/savevm.c:809-977` | - | 设备注册 (register_savevm_live / vmstate_register) |
| `migration/savevm.c:1848-1900` | - | qemu_savevm_state() 主流程 |
| `migration/savevm.c:3044-3074` | - | qemu_loadvm_state() 主流程 |
| `migration/savevm.c:3202-3504` | - | save/load/delete_snapshot() |
| `migration/ram.c:3112-3443` | - | RAM save setup/iterate/complete |
| `migration/ram.c:940-1253` | - | Dirty bitmap + zero page |
| `block/snapshot.c:216-785` | - | Block 层快照 API |
| `block/qcow2-snapshot.c:81-360` | - | QCOW2 快照表读写 |
| `target/arm/machine.c:1229-1328` | - | ARM CPU vmstate |
| `target/arm/machine.c:139-260` | - | VFP/SVE subsections |

---

*文档结束*
