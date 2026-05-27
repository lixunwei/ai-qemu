# Doc 94: CPU 热插拔深度分析 — 拓扑预分配/GIC Redistributor/ACPI GED/PSCI 启动

## 目录

1. [概述与架构定位](#1-概述与架构定位)
2. [CPU 热插拔槽位预分配](#2-cpu-热插拔槽位预分配)
3. [GICv3 Redistributor 容量规划](#3-gicv3-redistributor-容量规划)
4. [QMP 接口与 device_add 流程](#4-qmp-接口与-device_add-流程)
5. [virt 机型热插拔回调链](#5-virt-机型热插拔回调链)
6. [ACPI CPU 热插拔控制器](#6-acpi-cpu-热插拔控制器)
7. [GED 事件通知机制](#7-ged-事件通知机制)
8. [Guest 侧 ACPI 扫描流程](#8-guest-侧-acpi-扫描流程)
9. [CPU 热插拔与 PSCI 的协作](#9-cpu-热插拔与-psci-的协作)
10. [CPU 拓扑验证](#10-cpu-拓扑验证)
11. [ACPI 表中的 CPU 描述](#11-acpi-表中的-cpu-描述)
12. [vCPU 线程创建 (TCG/KVM)](#12-vcpu-线程创建-tcgkvm)
13. [热拔除 (Unplug)](#13-热拔除-unplug)
14. [完整热插拔时序](#14-完整热插拔时序)
15. [与真实硬件对比](#15-与真实硬件对比)
16. [总结](#16-总结)

---

## 1. 概述与架构定位

ARM64 virt 机型支持 CPU 热插拔，允许在虚拟机运行时动态添加/移除 vCPU。其核心设计理念是**预分配**：

- GIC Redistributor 空间在机器初始化时按 `max_cpus` 预留
- CPU 槽位 (possible_cpus) 在启动时全部定义
- 热插拔时只需"填充"已预留的槽位，无需修改硬件拓扑

```
┌─────────────────────────────────────────────────────────┐
│                 CPU 热插拔架构                            │
│                                                          │
│  QMP: device_add                                         │
│       │                                                  │
│       ▼                                                  │
│  ┌──────────────┐   ┌──────────────┐                    │
│  │ CPU Object   │──▶│ arm_cpu_     │                    │
│  │ Create/Init  │   │ realizefn()  │                    │
│  └──────────────┘   └──────┬───────┘                    │
│                             │                            │
│       ┌─────────────────────┼────────────────┐           │
│       ▼                     ▼                ▼           │
│  ┌─────────┐      ┌──────────────┐   ┌───────────┐     │
│  │ vCPU    │      │ Machine      │   │ GIC slot  │     │
│  │ Thread  │      │ Hotplug CB   │   │ (预分配)   │     │
│  └─────────┘      └──────┬───────┘   └───────────┘     │
│                           │                              │
│                           ▼                              │
│                    ┌──────────────┐                      │
│                    │ ACPI CPU     │                      │
│                    │ plug_cb()    │                      │
│                    └──────┬───────┘                      │
│                           │                              │
│                           ▼                              │
│                    ┌──────────────┐                      │
│                    │  GED IRQ     │──▶ Guest ACPI scan   │
│                    │  (SCI)       │    └─▶ CPU online    │
│                    └──────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

---

## 2. CPU 热插拔槽位预分配

### 2.1 virt_possible_cpu_arch_ids()

```c
// hw/arm/virt.c:3397-3449
static const CPUArchIdList *virt_possible_cpu_arch_ids(MachineState *ms)
{
    // 为 max_cpus 预分配槽位数组
    ms->possible_cpus = g_malloc0(sizeof(CPUArchIdList) +
                                  sizeof(CPUArchId) * ms->smp.max_cpus);
    ms->possible_cpus->len = ms->smp.max_cpus;

    for (int i = 0; i < ms->smp.max_cpus; i++) {
        CPUArchId *slot = &ms->possible_cpus->cpus[i];
        // 根据拓扑计算 socket/cluster/core/thread
        slot->props.socket_id = ...;
        slot->props.cluster_id = ...;
        slot->props.core_id = ...;
        slot->props.thread_id = ...;
        slot->arch_id = arm_cpu_mp_affinity(i, ...);  // MPIDR
        slot->type = ms->cpu_type;
    }
    return ms->possible_cpus;
}
```

### 2.2 槽位状态

| 字段 | 含义 |
|------|------|
| `slot->cpu` | NULL = 空槽（可热插） / 非NULL = 已占用 |
| `slot->arch_id` | MPIDR affinity 值 |
| `slot->props` | socket/cluster/core/thread ID |
| `slot->type` | CPU 类型字符串（如 "cortex-a72"） |

### 2.3 初始 CPU vs 热插 CPU

```
启动时：cpus=4, max_cpus=8
  slots[0..3].cpu = 已创建的 CPU 对象
  slots[4..7].cpu = NULL（可热插槽位）
```

---

## 3. GICv3 Redistributor 容量规划

### 3.1 Redistributor 区域布局

```c
// hw/arm/virt.c:185-186 (base_memmap)
[VIRT_GIC_REDIST] = { 0x080A0000, 0x00F60000 },  // 约 15.4MB

// hw/arm/virt.c:234-236 (extended_memmap, highmem)
[VIRT_HIGH_GIC_REDIST2] = ...  // 第二区域（highmem-redists=on）
```

### 3.2 容量计算

```c
// include/hw/arm/virt.h:231-239
static inline int virt_gicv3_redist_region_count(int smp_cpus)
{
    // 每个 redistributor 占 2×64KB = 128KB
    // 基础区域 (0xF60000 / 0x20000) = 123 CPUs
    // 超过 123 需要第二区域 (highmem)
}

// hw/arm/virt.c:1171-1185
// virt_max_cpus 受 redistributor 容量限制
// 若 max_cpus > 基础容量且未启用 highmem-redists，报错退出
```

### 3.3 容量限制

| 配置 | 最大 CPU 数 | 说明 |
|------|------------|------|
| 默认（基础区域） | 123 | 0xF60000 / 0x20000 |
| highmem-redists=on | 512 | 加上第二区域 |
| mc->max_cpus | 512 | virt 机型硬限制 |

### 3.4 预分配设计

GIC 在机器初始化时就为 `max_cpus` 个 CPU 分配所有 redistributor 状态：

```c
// hw/intc/arm_gicv3_common.c:439-489
static void arm_gicv3_common_realize(...)
{
    // 为所有可能的 CPU 分配 redistributor
    s->cpu = g_new0(GICv3CPUState, s->num_cpu);  // num_cpu = max_cpus
    for (i = 0; i < s->num_cpu; i++) {
        // 绑定 CPU 对象（启动时存在的）
        // 热插 CPU 的槽位暂时为空
    }
}
```

---

## 4. QMP 接口与 device_add 流程

### 4.1 查询可用槽位

```json
// QMP 命令
{ "execute": "query-hotpluggable-cpus" }

// 响应示例
[
  { "type": "cortex-a72-arm-cpu",
    "vcpus-count": 1,
    "props": { "socket-id": 0, "cluster-id": 0, "core-id": 4, "thread-id": 0 }
  },
  ...
]
```

### 4.2 热插 CPU

```json
// QMP 命令
{ "execute": "device_add",
  "arguments": {
    "driver": "cortex-a72-arm-cpu",
    "socket-id": 0,
    "cluster-id": 0,
    "core-id": 4,
    "thread-id": 0
  }
}
```

### 4.3 device_add 内部流程

```
device_add (QMP)
  → qdev_device_add()
    → object_new(cpu_type)           // 创建 CPU 对象
    → object_property_set()          // 设置拓扑属性
    → qdev_realize()                 // 触发 realize
      → arm_cpu_realizefn()          // ARM CPU 初始化
        → cpu_exec_realizefn()       // 执行引擎初始化
      → hotplug_handler_plug()       // 触发机器热插拔回调
```

---

## 5. virt 机型热插拔回调链

### 5.1 回调注册

```c
// hw/arm/virt.c (machine class init)
mc->get_hotplug_handler = virt_machine_get_hotplug_handler;
hc->pre_plug = virt_machine_device_pre_plug_cb;
hc->plug = virt_machine_device_plug_cb;
hc->unplug_request = virt_machine_device_unplug_request_cb;
hc->unplug = virt_machine_device_unplug_cb;
```

### 5.2 pre_plug 回调

```c
// hw/arm/virt.c:3496-3576
static void virt_machine_device_pre_plug_cb(HotplugHandler *hotplug_dev,
                                            DeviceState *dev, Error **errp)
{
    if (object_dynamic_cast(OBJECT(dev), TYPE_CPU)) {
        // 1. 验证 CPU 类型匹配
        // 2. 查找对应的 possible_cpus 槽位
        // 3. 验证拓扑属性 (socket/cluster/core/thread)
        // 4. 检查槽位未被占用
        // 5. 设置 CPU 的 mp-affinity (MPIDR)
    }
}
```

### 5.3 plug 回调

```c
// hw/arm/virt.c:3578-3621
static void virt_machine_device_plug_cb(HotplugHandler *hotplug_dev,
                                        DeviceState *dev, Error **errp)
{
    if (object_dynamic_cast(OBJECT(dev), TYPE_CPU)) {
        // 1. 将 CPU 对象记录到 possible_cpus 槽位
        slot->cpu = OBJECT(dev);

        // 2. 如果有 ACPI 设备，通知 ACPI 控制器
        if (vms->acpi_dev) {
            acpi_cpu_plug_cb(hotplug_dev, &vms->cpuhp_state, dev, errp);
        }
    }
}
```

---

## 6. ACPI CPU 热插拔控制器

### 6.1 数据结构

```c
// include/hw/acpi/cpu.h:24-40
typedef struct AcpiCpuStatus {
    DeviceState *cpu;         // 关联的 CPU 设备（NULL=不存在）
    uint64_t arch_id;         // MPIDR
    bool is_inserting;        // 正在插入标志
    bool is_removing;         // 正在移除标志
    bool fw_remove;           // 固件请求移除
    uint32_t ost_event;       // _OST 事件码
    uint32_t ost_status;      // _OST 状态码
} AcpiCpuStatus;

typedef struct CPUHotplugState {
    MemoryRegion ctrl_reg;    // MMIO 控制寄存器
    uint32_t selector;        // 当前选中的 CPU 索引
    AcpiCpuStatus *devs;      // 每个 CPU 的状态数组
    uint32_t dev_count;       // CPU 总数 (max_cpus)
    bool is_enabled;          // 热插拔是否启用
} CPUHotplugState;
```

### 6.2 初始化

```c
// hw/acpi/cpu.c:215-234
void cpu_hotplug_hw_init(MemoryRegion *as, Object *owner,
                         CPUHotplugState *state, hwaddr base_addr)
{
    // 为 max_cpus 分配状态数组
    state->devs = g_new0(AcpiCpuStatus, state->dev_count);

    // 标记已存在的 CPU
    for (i = 0; i < state->dev_count; i++) {
        state->devs[i].cpu = possible_cpus->cpus[i].cpu;
    }

    // 注册 MMIO 控制区域
    memory_region_init_io(&state->ctrl_reg, owner, &cpu_hotplug_ops, ...);
    memory_region_add_subregion(as, base_addr, &state->ctrl_reg);
}
```

### 6.3 plug 回调

```c
// hw/acpi/cpu.c:250-265
void acpi_cpu_plug_cb(HotplugHandler *hotplug_dev,
                      CPUHotplugState *cpu_st,
                      DeviceState *dev, Error **errp)
{
    // 找到对应槽位
    AcpiCpuStatus *cdev = get_cpu_status(cpu_st, dev);

    // 标记为已插入
    cdev->cpu = dev;
    cdev->is_inserting = true;

    // 发送 ACPI 事件（通过 GED）
    acpi_send_event(DEVICE(hotplug_dev), ACPI_CPU_HOTPLUG_STATUS);
}
```

---

## 7. GED 事件通知机制

### 7.1 GED (Generic Event Device) 架构

ARM 平台不使用 x86 的 SCI（GPE）机制，而是通过 GED 发送热插拔事件：

```c
// hw/acpi/generic_event_device.c:325-358
void acpi_ged_send_event(AcpiGedState *s, AcpiGedEventType event)
{
    // event 映射：
    //   ACPI_CPU_HOTPLUG_STATUS → ACPI_GED_CPU_HOTPLUG_EVT

    // 设置事件标志
    s->events[event] = true;

    // 触发 GED 中断 (SPI → GIC → Guest)
    qemu_irq_pulse(s->irq);
}
```

### 7.2 GED 中断到 Guest 的路径

```
acpi_cpu_plug_cb()
  → acpi_send_event(ACPI_CPU_HOTPLUG_STATUS)
    → acpi_ged_send_event(GED_CPU_HOTPLUG_EVT)
      → qemu_irq_pulse(ged_irq)
        → GIC SPI → Guest kernel ACPI subsystem
          → 执行 GED _EVT 方法
            → 调用 \_SB.CPUS.CSCN (CPU Scan)
```

### 7.3 GED AML 生成

```c
// hw/acpi/generic_event_device.c:117-119
// GED 事件处理方法中：
//   If (CPU hotplug event):
//     Notify(\_SB.CPUS, CSCN)  // 触发 CPU 扫描
```

---

## 8. Guest 侧 ACPI 扫描流程

### 8.1 build_cpus_aml() 生成的 AML

```c
// hw/acpi/cpu.c:444-724
void build_cpus_aml(...)
{
    // 创建 \_SB.CPUS 设备
    //   包含 N 个子设备 C000..CNNN

    // 每个 CPU 子设备的 _STA 方法：
    //   读取 MMIO selector → 检查 is_enabled
    //   返回 0xF (present+enabled) 或 0x0 (absent)

    // CSCN (CPU Scan) 方法：
    //   遍历所有 CPU 槽位
    //   检查 is_inserting / is_removing 标志
    //   对变化的 CPU 发送 Notify()

    // _OST 方法：
    //   写入 ost_event/ost_status
    //   触发 qapi_event_send_acpi_device_ost()
}
```

### 8.2 Guest 扫描序列

```
1. GED 中断触发
2. ACPI subsystem 执行 GED _EVT → 识别为 CPU 事件
3. 调用 \_SB.CPUS.CSCN
4. CSCN 遍历 selector 0..max_cpus-1：
   - 读 MMIO：检查 is_inserting 标志
   - 若 is_inserting：
     a. Notify(CPU_device, 1)  // Device Check
     b. 清除 is_inserting
5. Guest OS 收到 Notify → 评估 _STA → 返回 0xF
6. Guest OS 调用 cpu_up() 启动新 CPU
7. Guest OS 调用 _OST 报告成功
```

---

## 9. CPU 热插拔与 PSCI 的协作

### 9.1 新 CPU 的启动路径

热插入的 CPU 在 Guest OS 看来是"powered off"状态，需要通过 PSCI CPU_ON 唤醒：

```
Guest OS 检测到新 CPU (ACPI):
  → cpu_up(cpu_id)
    → smp_ops.cpu_boot(cpu_id)
      → PSCI CPU_ON(MPIDR, entry_point, context_id)
        → SMC/HVC 陷入 QEMU
          → arm_handle_psci_call() → CPU_ON
            → arm_set_cpu_on() → 目标 CPU 开始执行
```

### 9.2 FDT/ACPI 中的 enable-method

```c
// hw/arm/virt.c:674-677
// 当 PSCI 启用时，CPU 节点声明：
//   enable-method = "psci"  (FDT)
//   或 MADT GICC 中设置 PSCI 标志 (ACPI)
```

### 9.3 热插 CPU 的初始状态

```
热插入后的 CPU 状态：
  - halted = 1
  - power_state = PSCI_OFF
  - 不执行任何指令
  - 等待 PSCI CPU_ON 唤醒
```

---

## 10. CPU 拓扑验证

### 10.1 拓扑层次

```
Socket → Cluster → Core → Thread
  │         │        │       │
  │         │        │       └── thread-id (SMT)
  │         │        └────────── core-id
  │         └─────────────────── cluster-id
  └───────────────────────────── socket-id
```

### 10.2 pre_plug 验证内容

```c
// hw/arm/virt.c (virt_machine_device_pre_plug_cb)
验证项：
1. CPU 类型与机器配置匹配
2. socket-id 在有效范围内 (0..sockets-1)
3. cluster-id 在有效范围内 (0..clusters-1)
4. core-id 在有效范围内 (0..cores-1)
5. thread-id 在有效范围内 (0..threads-1)
6. 计算出的槽位索引未被占用 (slot->cpu == NULL)
7. 设置 MPIDR (mp-affinity) 给新 CPU
```

### 10.3 NUMA 节点分配

```c
// hw/core/machine.c:759-870
void machine_set_cpu_numa_node(MachineState *ms, ...)
{
    // 根据 socket/core/thread 匹配 possible_cpus 槽位
    // 将 NUMA node-id 分配给匹配的槽位
    // 验证：
    //   - 所有指定的拓扑字段都在范围内
    //   - 精确匹配（不允许模糊匹配）
}
```

---

## 11. ACPI 表中的 CPU 描述

### 11.1 MADT (Multiple APIC Description Table)

```c
// hw/arm/virt-acpi-build.c:1016-1055
static void build_madt(...)
{
    // 为每个 CPU 生成 GICC 结构
    for (i = 0; i < ms->smp.cpus; i++) {
        // Type 11: GIC CPU Interface Structure
        //   CPU Interface Number
        //   ACPI Processor UID
        //   MPIDR
        //   Flags: Enabled | Performance Interrupt Mode
        //   Parking Protocol Version (if applicable)
        //   GIC CPU Interface Address
        //   GICV/GICH/VGIC Maintenance Interrupt
    }

    // GICR (Redistributor) 区域描述
    // 告诉 guest redistributor 的 MMIO 范围
}
```

### 11.2 SRAT (System Resource Affinity Table)

```c
// hw/arm/virt-acpi-build.c:804-830
static void build_srat(...)
{
    // 使用 possible_cpu_arch_ids (max_cpus) 生成
    // 每个 CPU 的 NUMA affinity 条目
    // 包括尚未存在的热插槽位
}
```

### 11.3 PPTT (Processor Properties Topology Table)

描述 CPU 拓扑层次关系，Guest 用于识别 cache 共享和拓扑结构。

---

## 12. vCPU 线程创建 (TCG/KVM)

### 12.1 通用路径

```c
// system/cpus.c:709-730
void qemu_init_vcpu(CPUState *cpu)
{
    // 设置基本属性
    cpu->nr_cores = ...;
    cpu->nr_threads = ...;

    // 调用加速器创建线程
    cpus_accel->create_vcpu_thread(cpu);

    // 等待线程就绪
    while (!cpu->created) {
        qemu_cond_wait(&qemu_cpu_cond, &qemu_global_mutex);
    }
}
```

### 12.2 TCG 模式

- 单线程 TCG (icount)：所有 CPU 共享一个线程，round-robin 调度
- 多线程 TCG (MTTCG)：每个 CPU 独立线程

### 12.3 KVM 模式

- 每个 CPU 一个内核线程
- `kvm_init_vcpu()` → `ioctl(KVM_CREATE_VCPU)` 创建内核态 vCPU
- 热插时同样走此路径

---

## 13. 热拔除 (Unplug)

### 13.1 请求拔除

```c
// hw/arm/virt.c:3661-3673
static void virt_machine_device_unplug_request_cb(...)
{
    if (object_dynamic_cast(OBJECT(dev), TYPE_CPU)) {
        // 通知 ACPI 控制器
        acpi_cpu_unplug_request_cb(hotplug_dev, &vms->cpuhp_state, dev, errp);
    }
}
```

### 13.2 ACPI 拔除流程

```c
// hw/acpi/cpu.c:267-293
void acpi_cpu_unplug_request_cb(...)
{
    cdev->is_removing = true;
    acpi_send_event(ACPI_CPU_HOTPLUG_STATUS);
    // Guest 收到后执行 cpu_down()
}

void acpi_cpu_unplug_cb(...)
{
    cdev->cpu = NULL;
    cdev->is_removing = false;
    // CPU 对象销毁
}
```

### 13.3 Guest 侧拔除序列

```
1. GED 中断 → CSCN 扫描发现 is_removing
2. Notify(CPU_device, 3)  // Eject Request
3. Guest OS: cpu_down(cpu_id)
   → 迁移进程/中断到其他 CPU
   → 停止 CPU 调度
   → PSCI CPU_OFF (可选)
4. Guest OS: _OST(成功)
5. QEMU 完成拔除：销毁 CPU 对象/线程
```

---

## 14. 完整热插拔时序

```
T0: 机器初始化 (cpus=4, max_cpus=8)
    ├── virt_possible_cpu_arch_ids(): 创建 8 个槽位
    ├── create_gic(): 为 8 个 CPU 分配 redistributor
    ├── 创建 CPU 0-3, 槽位 0-3 标记已占用
    └── 槽位 4-7 保持空（cpu=NULL）

T1: 用户执行 query-hotpluggable-cpus
    └── 返回槽位 4-7 的属性（空闲可用）

T2: 用户执行 device_add (core-id=4)
    ├── object_new("cortex-a72-arm-cpu")
    ├── 设置 socket-id=0, cluster-id=0, core-id=4, thread-id=0
    └── qdev_realize() 触发

T3: CPU Realize
    ├── arm_cpu_realizefn()
    │   ├── 初始化寄存器/特性
    │   ├── cpu_exec_realizefn()
    │   └── qemu_init_vcpu() → 创建 vCPU 线程
    └── CPU 进入 halted/PSCI_OFF 状态

T4: Machine Hotplug Callbacks
    ├── virt_machine_device_pre_plug_cb()
    │   ├── 验证拓扑属性
    │   ├── 查找槽位 4
    │   └── 设置 MPIDR
    ├── virt_machine_device_plug_cb()
    │   ├── slot[4].cpu = new_cpu
    │   └── acpi_cpu_plug_cb()
    │       ├── cdev->is_inserting = true
    │       └── acpi_send_event(CPU_HOTPLUG)
    └── acpi_ged_send_event() → GED IRQ

T5: Guest ACPI 处理
    ├── GED ISR → _EVT 方法
    ├── 调用 \_SB.CPUS.CSCN
    ├── 扫描到 slot[4].is_inserting = true
    ├── Notify(C004, DeviceCheck)
    └── Guest OS 收到通知

T6: Guest OS 启动新 CPU
    ├── 评估 _STA → 返回 0xF (present+enabled)
    ├── cpu_up(4)
    │   └── PSCI CPU_ON(mpidr=0x4, entry=secondary_entry)
    │       → SMC → QEMU → arm_set_cpu_on()
    │       → 目标 CPU: reset + set PC + PSCI_ON
    ├── 新 CPU 开始执行 secondary_entry
    ├── 加入调度器
    └── _OST(成功) → QEMU 记录

T7: 新 CPU 正常运行
    └── 可接收中断、运行进程
```

---

## 15. 与真实硬件对比

### 15.1 机制对比

| 特性 | 真实硬件 (ACPI/PSCI) | QEMU virt |
|------|---------------------|-----------|
| CPU 发现 | MADT/PPTT 预描述 | 同（max_cpus 槽位） |
| 热插通知 | Platform SCI/GED | GED IRQ (模拟) |
| CPU 启动 | PSCI CPU_ON | PSCI CPU_ON (模拟) |
| GIC 扩展 | 硬件 redistributor | 预分配 (无动态扩展) |
| 电源管理 | 真实上下电 | 状态标志切换 |
| 拓扑感知 | PPTT + MPIDR | 同 |
| 缓存一致性 | 硬件保证 | 不模拟 |

### 15.2 关键差异

1. **GIC 不动态扩展**：真实平台的 GIC redistributor 可能按需映射；QEMU 全部预分配
2. **无真实电源操作**：热插只是"使能已分配的资源"，无上电时序
3. **MADT 静态**：QEMU 的 MADT 在启动时生成，只包含初始 CPU 数量的条目（但 SRAT 包含全部 max_cpus）
4. **无固件参与**：真实平台可能需要 SCP/SCMI 协调 CPU 上电；QEMU 直接操作
5. **内存一致性**：真实热插可能涉及 NUMA 距离变化；QEMU 是预设的

### 15.3 限制

- 不支持 CPU 类型混合（所有 CPU 必须同类型）
- 不支持跨 NUMA 节点动态迁移 CPU
- KVM 模式下热拔除支持有限（取决于内核版本）
- max_cpus 不能超过 GIC redistributor 容量

---

## 16. 总结

### 设计哲学

ARM64 virt 的 CPU 热插拔采用**"预留资源 + 延迟填充"**策略：
- 启动时按 max_cpus 预留所有 GIC/ACPI/拓扑资源
- 热插时仅需"激活"已预留的槽位
- 避免了运行时修改 GIC 拓扑或重建 ACPI 表的复杂性

### 核心路径

```
device_add → realize → pre_plug验证 → plug回调
  → ACPI标记插入 → GED中断 → Guest扫描
    → PSCI CPU_ON → CPU开始执行
```

### 关键设计约束

1. `max_cpus` 决定了 GIC redistributor 区域大小（不可运行时修改）
2. CPU 拓扑（socket/cluster/core/thread）在启动时固定
3. 热插 CPU 通过 PSCI 启动（与冷启动的 secondary CPU 相同路径）
4. ACPI GED 是 ARM 平台的统一事件通知机制

---

## 附录 A：关键源码文件索引

| 文件 | 行号 | 功能 |
|------|------|------|
| hw/arm/virt.c:3397-3449 | virt_possible_cpu_arch_ids() 槽位预分配 |
| hw/arm/virt.c:3496-3576 | pre_plug 回调（拓扑验证） |
| hw/arm/virt.c:3578-3621 | plug 回调（激活槽位） |
| hw/arm/virt.c:3661-3686 | unplug_request/unplug 回调 |
| hw/arm/virt.c:1171-1185 | GIC redistributor 容量检查 |
| hw/acpi/cpu.c:215-234 | cpu_hotplug_hw_init() |
| hw/acpi/cpu.c:250-265 | acpi_cpu_plug_cb() |
| hw/acpi/cpu.c:267-293 | acpi_cpu_unplug_request/cb() |
| hw/acpi/cpu.c:444-724 | build_cpus_aml() ACPI 命名空间 |
| hw/acpi/generic_event_device.c:325-358 | acpi_ged_send_event() |
| hw/intc/arm_gicv3_common.c:439-489 | GIC redistributor 预分配 |
| hw/intc/arm_gicv3_common.c:321-369 | Redistributor MMIO 区域创建 |
| hw/arm/virt-acpi-build.c:1016-1055 | build_madt() GICC 生成 |
| hw/arm/virt-acpi-build.c:804-830 | build_srat() NUMA affinity |
| hw/core/machine.c:707-731 | machine_query_hotpluggable_cpus() |
| hw/core/machine.c:759-870 | machine_set_cpu_numa_node() |
| system/cpus.c:709-730 | qemu_init_vcpu() 线程创建 |
| include/hw/acpi/cpu.h:24-40 | AcpiCpuStatus/CPUHotplugState |

## 附录 B：命令行与 QMP 参考

```bash
# 启动时预留热插槽位
qemu-system-aarch64 -M virt,gic-version=3 \
    -smp cpus=4,maxcpus=8,sockets=1,clusters=1,cores=8,threads=1 \
    -cpu cortex-a72 -m 4G -kernel Image

# QMP: 查询可热插 CPU
echo '{"execute":"query-hotpluggable-cpus"}' | qmp-shell

# QMP: 热插入 CPU (core-id=4)
echo '{"execute":"device_add","arguments":{"driver":"cortex-a72-arm-cpu","core-id":4}}' | qmp-shell

# QMP: 请求热拔除
echo '{"execute":"device_del","arguments":{"id":"cpu4"}}' | qmp-shell

# 使用 highmem redistributor（支持 >123 CPU）
qemu-system-aarch64 -M virt,gic-version=3,highmem-redists=on \
    -smp cpus=4,maxcpus=256 -cpu max -m 8G
```
