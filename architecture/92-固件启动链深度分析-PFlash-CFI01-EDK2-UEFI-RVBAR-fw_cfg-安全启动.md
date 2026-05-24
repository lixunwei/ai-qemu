# Doc 92: 固件启动链深度分析 — PFlash/CFI01/EDK2/UEFI/RVBAR/fw_cfg/安全启动

## 目录

1. [概述与架构定位](#1-概述与架构定位)
2. [virt 机型内存布局中的 Flash 区域](#2-virt-机型内存布局中的-flash-区域)
3. [PFlash CFI01 设备模型](#3-pflash-cfi01-设备模型)
4. [PFlash 设备创建与映射](#4-pflash-设备创建与映射)
5. [固件加载路径：-pflash vs -bios](#5-固件加载路径-pflash-vs--bios)
6. [CPU 复位向量与 RVBAR 机制](#6-cpu-复位向量与-rvbar-机制)
7. [Firmware Boot 模式 vs Direct Kernel Boot](#7-firmware-boot-模式-vs-direct-kernel-boot)
8. [fw_cfg 设备：固件与 QEMU 的数据通道](#8-fw_cfg-设备固件与-qemu-的数据通道)
9. [EDK2 UEFI 固件集成](#9-edk2-uefi-固件集成)
10. [UEFI 变量存储（第二 PFlash）](#10-uefi-变量存储第二-pflash)
11. [安全启动与 TrustZone 集成](#11-安全启动与-trustzone-集成)
12. [PFlash ROM/IO 模式切换优化](#12-pflash-romio-模式切换优化)
13. [CFI 命令状态机详解](#13-cfi-命令状态机详解)
14. [PFlash 与 BlockBackend 的持久化](#14-pflash-与-blockbackend-的持久化)
15. [FDT 中的 Flash 描述](#15-fdt-中的-flash-描述)
16. [完整启动时序](#16-完整启动时序)
17. [与真实硬件的对比分析](#17-与真实硬件的对比分析)
18. [总结](#18-总结)

---

## 1. 概述与架构定位

QEMU ARM64 virt 机型的固件启动链模拟了真实 ARM 平台从上电到固件执行的完整流程。核心组件包括：

- **PFlash CFI01**：Intel 命令集 NOR Flash 设备模型，存放 UEFI 固件代码和变量
- **RVBAR (Reset Vector Base Address Register)**：决定 CPU 复位后的第一条指令地址
- **fw_cfg**：QEMU 与固件之间的数据传递通道
- **EDK2**：TianoCore UEFI 固件实现，QEMU 默认提供的 ARM64 固件

```
┌─────────────────────────────────────────────────────────┐
│                    用户命令行                              │
│  -pflash code.fd  -pflash vars.fd  [-kernel Image]      │
└─────────────┬───────────────────────────┬───────────────┘
              │                           │
              ▼                           ▼
┌─────────────────────┐    ┌─────────────────────────┐
│ virt_firmware_init() │    │   arm_load_kernel()     │
│  ├─legacy_drive()   │    │  ├─firmware_boot path   │
│  ├─virt_flash_map() │    │  └─fw_cfg expose kernel │
│  └─load_image_mr()  │    └─────────────────────────┘
└─────────┬───────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│           Physical Address Space (PA)                     │
│                                                           │
│  0x0000_0000 ┌──────────────────┐ ← Flash0 (Code)       │
│              │   UEFI Firmware   │   64MB (secure only)   │
│  0x0400_0000 ├──────────────────┤ ← Flash1 (Vars)       │
│              │   NVRAM/Vars      │   64MB (all worlds)    │
│  0x0800_0000 ├──────────────────┤ ← GIC/Periph          │
│              │     ...           │                        │
│  0x0902_0000 │   fw_cfg MMIO    │                        │
│              │     ...           │                        │
│  0x4000_0000 ├──────────────────┤ ← RAM                  │
│              │   System Memory   │                        │
└─────────────────────────────────────────────────────────┘
```

---

## 2. virt 机型内存布局中的 Flash 区域

```c
// hw/arm/virt.c:173-175
static const MemMapEntry base_memmap[] = {
    /* Space up to 0x8000000 is reserved for a boot ROM */
    [VIRT_FLASH] = { 0, 0x08000000 },  // 128MB total
    ...
};
```

**关键设计决策**：

| 属性 | 值 | 说明 |
|------|-----|------|
| 基地址 | 0x0000_0000 | CPU 复位向量默认指向此处 |
| 总大小 | 128 MB (0x0800_0000) | 等分为两块各 64MB |
| Flash0 | 0x0000_0000 ~ 0x03FF_FFFF | 固件代码 (secure only) |
| Flash1 | 0x0400_0000 ~ 0x07FF_FFFF | UEFI 变量 (所有 world) |
| 扇区大小 | 256 KiB | VIRT_FLASH_SECTOR_SIZE |

Flash 放在地址 0 的设计与真实 ARM 平台一致：CPU 复位后 PC 从 RVBAR 取值，默认为 0，直接执行 Flash 中的固件。

---

## 3. PFlash CFI01 设备模型

### 3.1 数据结构

```c
// hw/block/pflash_cfi01.c:60-91
struct PFlashCFI01 {
    SysBusDevice parent_obj;

    BlockBackend *blk;           // 后端存储（文件/块设备）
    uint32_t nb_blocs;           // 块数量
    uint64_t sector_len;         // 扇区大小（256KB）
    uint8_t bank_width;          // 总线宽度（4字节）
    uint8_t device_width;        // 单设备宽度（2字节）
    uint8_t max_device_width;    // 最大设备宽度
    uint32_t features;           // PFLASH_BE | PFLASH_SECURE
    uint8_t wcycle;              // 写周期状态机
    bool ro;                     // 只读标志
    uint8_t cmd;                 // 当前命令
    uint8_t status;              // 状态寄存器
    uint16_t ident0~ident3;      // 设备 ID
    uint8_t cfi_table[0x52];     // CFI 查询表
    uint64_t counter;            // 块写计数器
    uint32_t writeblock_size;    // 写块大小
    MemoryRegion mem;            // MMIO 区域（ROM device）
    char *name;                  // 设备名称
    void *storage;               // Flash 内容的 RAM 缓存
    unsigned char *blk_bytes;    // 块写缓冲区
    uint32_t blk_offset;         // 块写偏移（-1表示非写入状态）
};
```

### 3.2 设备属性

```c
// hw/block/pflash_cfi01.c:892-927
static const Property pflash_cfi01_properties[] = {
    DEFINE_PROP_DRIVE("drive", ...),          // 后端块设备
    DEFINE_PROP_UINT32("num-blocks", ...),    // 块数量
    DEFINE_PROP_UINT64("sector-length", ...), // 扇区大小
    DEFINE_PROP_UINT8("width", ...),          // 总线宽度
    DEFINE_PROP_UINT8("device-width", ...),   // 设备宽度
    DEFINE_PROP_BIT("big-endian", ...),       // 大端
    DEFINE_PROP_BIT("secure", ...),           // Secure 属性
    DEFINE_PROP_UINT16("id0"~"id3", ...),    // 设备 ID
    DEFINE_PROP_STRING("name", ...),          // 名称
};
```

### 3.3 初始化流程 (pflash_cfi01_realize)

```c
// hw/block/pflash_cfi01.c:796-869
static void pflash_cfi01_realize(DeviceState *dev, Error **errp)
{
    total_len = pfl->sector_len * pfl->nb_blocs;

    // 1. 创建 ROM Device 类型的 MemoryRegion
    memory_region_init_rom_device(&pfl->mem, ..., total_len, errp);

    // 2. 获取 RAM 指针作为 storage
    pfl->storage = memory_region_get_ram_ptr(&pfl->mem);
    sysbus_init_mmio(SYS_BUS_DEVICE(dev), &pfl->mem);

    // 3. 如果有后端块设备，读入全部内容
    if (pfl->blk) {
        blk_check_size_and_read_all(pfl->blk, dev, pfl->storage, total_len);
    }

    // 4. 初始化状态机和 CFI 表
    pfl->wcycle = 0;
    pfl->cmd = 0x00;        // READ_ARRAY
    pfl->status = 0x80;     // WSM Ready
    pflash_cfi01_fill_cfi_table(pfl);
}
```

**关键点**：`memory_region_init_rom_device()` 创建的区域具有双重性质：
- 默认 ROMD 模式：读操作直接访问 RAM（零开销，如同 ROM）
- 写操作或命令模式：通过 ops 回调处理

---

## 4. PFlash 设备创建与映射

### 4.1 创建 (virt_flash_create1)

```c
// hw/arm/virt.c:1573-1596
static PFlashCFI01 *virt_flash_create1(VirtMachineState *vms,
                                        const char *name,
                                        const char *alias_prop_name)
{
    DeviceState *dev = qdev_new(TYPE_PFLASH_CFI01);
    qdev_prop_set_uint64(dev, "sector-length", VIRT_FLASH_SECTOR_SIZE); // 256KB
    qdev_prop_set_uint8(dev, "width", 4);         // 32-bit bus
    qdev_prop_set_uint8(dev, "device-width", 2);  // 16-bit device
    qdev_prop_set_bit(dev, "big-endian", false);   // Little-endian
    qdev_prop_set_uint16(dev, "id0", 0x89);       // Intel manufacturer
    qdev_prop_set_uint16(dev, "id1", 0x18);       // Intel device
    object_property_add_alias(OBJECT(vms), alias_prop_name,
                              OBJECT(dev), "drive"); // 绑定 -drive 接口
    return PFLASH_CFI01(dev);
}
```

### 4.2 映射 (virt_flash_map)

```c
// hw/arm/virt.c:1620-1639
static void virt_flash_map(VirtMachineState *vms, MemoryRegion *sysmem,
                           MemoryRegion *secure_sysmem)
{
    hwaddr flashsize = vms->memmap[VIRT_FLASH].size / 2;  // 64MB each
    hwaddr flashbase = vms->memmap[VIRT_FLASH].base;      // 0x0

    // Flash0: 固件代码 → 仅映射到 secure_sysmem
    virt_flash_map1(vms->flash[0], flashbase, flashsize, secure_sysmem);

    // Flash1: 变量存储 → 映射到 sysmem（所有 world 可见）
    virt_flash_map1(vms->flash[1], flashbase + flashsize, flashsize, sysmem);
}
```

**安全隔离设计**：
- `flash[0]`（代码）仅对 Secure world 可见 → 防止 Non-Secure 篡改固件
- `flash[1]`（变量）对所有 world 可见 → UEFI Runtime Services 需要 NS 访问 NVRAM

---

## 5. 固件加载路径：-pflash vs -bios

### 5.1 virt_firmware_init() 完整流程

```c
// hw/arm/virt.c:1685-1733
static bool virt_firmware_init(VirtMachineState *vms,
                               MemoryRegion *sysmem,
                               MemoryRegion *secure_sysmem)
{
    // Step 1: 绑定 legacy -drive if=pflash
    for (i = 0; i < 2; i++) {
        pflash_cfi01_legacy_drive(vms->flash[i], drive_get(IF_PFLASH, 0, i));
    }

    // Step 2: 映射 Flash 到地址空间
    virt_flash_map(vms, sysmem, secure_sysmem);

    // Step 3: 检查 -bios 选项
    pflash_blk0 = pflash_cfi01_get_blk(vms->flash[0]);
    bios_name = MACHINE(vms)->firmware;

    if (bios_name) {
        if (pflash_blk0) {
            error_report("Cannot use both -bios and -drive if=pflash");
            exit(1);
        }
        // -bios fallback: 直接加载到 flash0 的 MemoryRegion
        fname = qemu_find_file(QEMU_FILE_TYPE_BIOS, bios_name);
        mr = sysbus_mmio_get_region(SYS_BUS_DEVICE(vms->flash[0]), 0);
        load_image_mr(fname, mr);
    }

    return pflash_blk0 || bios_name;  // 是否有固件
}
```

### 5.2 两种固件加载方式对比

| 特性 | `-drive if=pflash` | `-bios` |
|------|-------------------|---------|
| 加载方式 | BlockBackend 全量读入 | load_image_mr() 直接写入 region |
| 持久化 | 写入可落盘到文件 | 无持久化（只读加载） |
| UEFI 变量 | 支持（pflash1 独立文件） | 不支持变量持久化 |
| 典型用法 | 生产部署 | 快速测试 |
| 支持两块 Flash | 是 | 仅 flash0 |

### 5.3 典型命令行

```bash
# 完整 UEFI 启动（代码 + 变量持久化）
qemu-system-aarch64 -M virt \
    -drive if=pflash,format=raw,file=AAVMF_CODE.fd,readonly=on \
    -drive if=pflash,format=raw,file=AAVMF_VARS.fd

# 简化 BIOS 启动（无变量持久化）
qemu-system-aarch64 -M virt -bios AAVMF_CODE.fd

# UEFI + 内核（固件通过 fw_cfg 加载内核）
qemu-system-aarch64 -M virt \
    -drive if=pflash,format=raw,file=AAVMF_CODE.fd \
    -drive if=pflash,format=raw,file=AAVMF_VARS.fd \
    -kernel Image -initrd initrd.img -append "root=/dev/vda"
```

---

## 6. CPU 复位向量与 RVBAR 机制

### 6.1 ARM 规范定义

根据 ARM Architecture Reference Manual (Armv9.6)：

> When a Cold or Warm reset is deasserted, execution starts at an IMPLEMENTATION DEFINED address. 
> The RVBAR associated with the highest implemented Exception level (RVBAR_EL1, RVBAR_EL2, or RVBAR_EL3) 
> holds the address at which the PE starts execution.

RVBAR_EL3 特性：
- 只读寄存器
- 仅当 EL3 是最高实现 EL 时存在
- 复位值为 IMPLEMENTATION DEFINED（通常由硬件配置输入决定）
- 复位时 PC 从 RVBAR 采样

### 6.2 QEMU 实现

```c
// target/arm/cpu.c:405-414
if (arm_feature(env, ARM_FEATURE_EL3)) {
    env->pstate = PSTATE_MODE_EL3h;
} else if (arm_feature(env, ARM_FEATURE_EL2)) {
    env->pstate = PSTATE_MODE_EL2h;
} else {
    env->pstate = PSTATE_MODE_EL1h;
}

/* Sample rvbar at reset. */
env->cp15.rvbar = cpu->rvbar_prop;
env->pc = env->cp15.rvbar;
```

```c
// target/arm/cpu.c:1509-1510
object_property_add_uint64_ptr(obj, "rvbar", &cpu->rvbar_prop, OBJ_PROP_FLAG_READWRITE);
```

**QEMU 中 RVBAR 的默认值为 0**，正好对应 Flash 的基地址。这就是为什么 CPU 复位后自动执行 Flash 中的固件。

### 6.3 RVBAR 系统寄存器注册

```c
// target/arm/helper.c:6872-6875
{ .name = "RVBAR_EL3", .state = ARM_CP_STATE_AA64,
  .opc0 = 3, .opc1 = 6, .crn = 12, .crm = 0, .opc2 = 1,
  .access = PL3_R,  // 只读
  .fieldoffset = offsetof(CPUARMState, cp15.rvbar) },
```

**与真实硬件对比**：
- 真实硬件：RVBAR 由硬件引脚/配置决定，上电后不可修改
- QEMU：通过 `rvbar` 属性在创建 CPU 时设置，默认为 0

---

## 7. Firmware Boot 模式 vs Direct Kernel Boot

### 7.1 arm_load_kernel() 分支选择

```c
// hw/arm/boot.c:1208-1212
if (!info->kernel_filename || info->firmware_loaded) {
    arm_setup_firmware_boot(cpu, info);  // 固件模式
} else {
    arm_setup_direct_kernel_boot(cpu, info);  // 直接内核模式
}
```

**判定逻辑**：有固件时（即使同时指定了 -kernel），走固件启动路径。

### 7.2 Firmware Boot 路径

```c
// hw/arm/boot.c:1115-1171
static void arm_setup_firmware_boot(ARMCPU *cpu, struct arm_boot_info *info)
{
    // 1. DTB 放到 RAM 基址供固件获取
    if (have_dtb(info)) {
        info->dtb_start = info->loader_start;
    }

    // 2. 如果同时指定了 kernel，通过 fw_cfg 暴露给固件
    if (info->kernel_filename) {
        FWCfgState *fw_cfg = fw_cfg_find();
        load_image_to_fw_cfg(fw_cfg, FW_CFG_KERNEL_SIZE,
                            FW_CFG_KERNEL_DATA, info->kernel_filename, ...);
        load_image_to_fw_cfg(fw_cfg, FW_CFG_INITRD_SIZE,
                            FW_CFG_INITRD_DATA, info->initrd_filename, false);
        // cmdline 也通过 fw_cfg 传递
    }

    // 3. 关键：不设置 boot_info → do_cpu_reset() 不修改 PC
    //    CPU 将从 RVBAR（即 Flash 基址）开始执行
}
```

### 7.3 do_cpu_reset() 行为

```c
// hw/arm/boot.c:655-741
static void do_cpu_reset(void *opaque)
{
    cpu_reset(cs);  // 基础复位（PC = RVBAR = 0）

    if (info) {
        if (!info->is_linux) {
            // 非 Linux：跳转到 entry point
            cpu_set_pc(cs, info->entry);
        } else {
            // Linux：设置适当 EL，跳到 loader_start
            arm_emulate_firmware_reset(cs, target_el);
            cpu_set_pc(cs, info->loader_start);
        }
    }
    // info == NULL 时（firmware boot）：PC 保持为 RVBAR = 0
    //  → CPU 执行 Flash 中的固件代码
}
```

---

## 8. fw_cfg 设备：固件与 QEMU 的数据通道

### 8.1 设备创建

```c
// hw/arm/virt.c:1735-1755
static FWCfgState *create_fw_cfg(const VirtMachineState *vms, AddressSpace *as)
{
    hwaddr base = vms->memmap[VIRT_FW_CFG].base;  // 0x0902_0000
    fw_cfg = fw_cfg_init_mem_dma(base + 8, base, 8, base + 16, as);
    fw_cfg_add_i16(fw_cfg, FW_CFG_NB_CPUS, ms->smp.cpus);
    // FDT 节点：compatible = "qemu,fw-cfg-mmio"
}
```

### 8.2 fw_cfg 传递的数据

| Key | 内容 | 用途 |
|-----|------|------|
| FW_CFG_NB_CPUS | CPU 数量 | 固件 SMP 初始化 |
| FW_CFG_KERNEL_SIZE/DATA | 内核镜像 | UEFI 直接加载内核 |
| FW_CFG_INITRD_SIZE/DATA | initrd | 配合内核启动 |
| FW_CFG_CMDLINE_SIZE/DATA | cmdline | 内核命令行 |

### 8.3 EDK2 读取 fw_cfg 的流程

EDK2 中的 `QemuFwCfgLib` 驱动通过 MMIO 访问 fw_cfg：
1. 写 selector 寄存器选择 key
2. 从 data 寄存器读取内容
3. 支持 DMA 大批量传输

---

## 9. EDK2 UEFI 固件集成

### 9.1 QEMU 提供的 ARM 固件文件

```
pc-bios/edk2-aarch64-code.fd.bz2   # AArch64 UEFI 代码（virt 机型）
pc-bios/edk2-arm-code.fd.bz2       # AArch32 UEFI 代码
pc-bios/edk2-arm-vars.fd.bz2       # UEFI 变量模板
```

### 9.2 描述文件

```json
// pc-bios/descriptors/60-edk2-aarch64.json
{
  "description": "UEFI firmware for ARM64 virtual machines",
  "interface-types": ["uefi"],
  "mapping": {
    "device": "flash",
    "executable": {
      "filename": "edk2-aarch64-code.fd",
      "format": "raw"
    },
    "nvram-template": {
      "filename": "edk2-arm-vars.fd",
      "format": "raw"
    }
  },
  "targets": [{"architecture": "aarch64", "machines": ["virt-*"]}]
}
```

### 9.3 EDK2 启动阶段

```
┌─────────────────────────────────────────────────────┐
│ SEC (Security)  → 最早执行，初始化临时 RAM            │
│     ↓                                                │
│ PEI (Pre-EFI)  → 初始化主存，发现硬件                │
│     ↓                                                │
│ DXE (Driver Execution) → 加载所有驱动                 │
│     ↓                                                │
│ BDS (Boot Device Selection) → 选择启动设备            │
│     ↓                                                │
│ OS Loader → 加载 Linux/Windows                       │
└─────────────────────────────────────────────────────┘
```

在 QEMU virt 机型中：
- **SEC 阶段**：从 Flash 地址 0 开始执行
- **PEI 阶段**：通过 fw_cfg 检测内存大小，初始化 GIC
- **DXE 阶段**：加载 VirtIO 驱动、PCI 枚举、网络栈
- **BDS 阶段**：检查 fw_cfg 中是否有 kernel，或从磁盘引导

---

## 10. UEFI 变量存储（第二 PFlash）

### 10.1 双 Flash 布局

```
flash[0] (0x0000_0000 ~ 0x03FF_FFFF): UEFI 固件代码
    - 通常 readonly
    - 映射到 secure_sysmem（仅 secure world 可见）
    - 包含 SEC/PEI/DXE 核心代码

flash[1] (0x0400_0000 ~ 0x07FF_FFFF): UEFI 变量存储
    - 可读写
    - 映射到 sysmem（所有 world 可见）
    - 存储：启动顺序、Secure Boot 密钥、OS 变量
    - 通过 CFI 命令编程/擦除
```

### 10.2 变量持久化机制

```
Guest UEFI Runtime → 写变量
    ↓ CFI Write command (0x40 single byte / 0xE8 buffer write)
    ↓ pflash_write() 状态机处理
    ↓ pflash_data_write() → 修改 pfl->storage
    ↓ pflash_update() → blk_pwrite() 写回块设备
    ↓ 宿主文件系统落盘
```

### 10.3 变量区的 CFI 操作

UEFI Runtime Services 通过标准 CFI 命令操作 Flash：
1. **擦除**：发送 0x20 + 0xD0 确认 → 256KB 扇区清零为 0xFF
2. **编程**：发送 0x40 + 数据 → 单字节写入
3. **缓冲写**：发送 0xE8 + 计数 + 数据 + 0xD0 确认 → 批量写入

---

## 11. 安全启动与 TrustZone 集成

### 11.1 Secure 模式启用条件

```c
// hw/arm/virt.c:2649-2661
vms->secure = !vms->tcg_disabled;  // 仅 TCG 支持 secure
if (vms->secure) {
    // 创建独立的 secure 地址空间
    secure_sysmem = g_new(MemoryRegion, 1);
    memory_region_init(secure_sysmem, ..., "secure-memory", UINT64_MAX);
    // 非 secure 视图作为低优先级子区域
    memory_region_add_subregion_overlap(secure_sysmem, 0, sysmem, -1);
}
```

### 11.2 固件加载对 PSCI 的影响

```c
// hw/arm/virt.c:2666-2682
if (vms->secure && firmware_loaded) {
    // 有安全固件 → 禁用 QEMU 内置 PSCI（由固件自己实现）
    vms->psci_conduit = QEMU_PSCI_CONDUIT_DISABLED;
} else if (vms->virt) {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_SMC;
} else {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_HVC;
}
```

**设计逻辑**：如果用户提供了安全固件（如 ATF/TF-A），说明固件会自己处理 PSCI 请求，QEMU 不应拦截 SMC/HVC。

### 11.3 安全 Flash 的访问控制

```c
// hw/block/pflash_cfi01.c:661-687
static MemTxResult pflash_mem_read_with_attrs(void *opaque, hwaddr addr,
                                              uint64_t *value, unsigned len,
                                              MemTxAttrs attrs)
{
    if ((pfl->features & (1 << PFLASH_SECURE)) && !attrs.secure) {
        // 非 secure 访问 secure flash → 只能读数据（不能执行命令）
        *value = pflash_data_read(opaque, addr, len, be);
    } else {
        *value = pflash_read(opaque, addr, len, be);
    }
}

static MemTxResult pflash_mem_write_with_attrs(...)
{
    if ((pfl->features & (1 << PFLASH_SECURE)) && !attrs.secure) {
        return MEMTX_ERROR;  // 非 secure 完全禁止写入
    }
    pflash_write(opaque, addr, value, len, be);
}
```

### 11.4 Secure Memory 布局

```
Secure Address Space (secure_sysmem):
├── Flash0 @ 0x0000_0000 (secure only, priority 0)
├── Secure MEM @ 0x0E00_0000 (16MB, priority 0)
├── Secure GPIO @ 0x090B_0000
└── sysmem @ 0x0 (priority -1, fallback)

Normal Address Space (sysmem):
├── Flash1 @ 0x0400_0000 (vars, accessible)
├── GIC @ 0x0800_0000
├── UART/RTC/fw_cfg
├── VirtIO MMIO
├── PCIe
└── RAM @ 0x4000_0000
```

---

## 12. PFlash ROM/IO 模式切换优化

### 12.1 ROM Device 机制

`memory_region_init_rom_device()` 创建的 MemoryRegion 有两种访问模式：

| 模式 | 读行为 | 写行为 | 性能 |
|------|--------|--------|------|
| ROMD (默认) | 直接访问 RAM | 触发 ops 回调 | 极高（翻译块可内联） |
| IO mode | 触发 ops 回调 | 触发 ops 回调 | 较低（每次 MMIO trap） |

### 12.2 模式切换时机

```c
// 进入 IO mode（首次写命令时）
// hw/block/pflash_cfi01.c:470-473
static void pflash_write(PFlashCFI01 *pfl, hwaddr offset, ...)
{
    if (!pfl->wcycle) {
        /* Set the device in I/O access mode */
        memory_region_rom_device_set_romd(&pfl->mem, false);
    }
    ...
}

// 返回 ROMD（READ_ARRAY 命令或复位）
// hw/block/pflash_cfi01.c:653-657
mode_read_array:
    memory_region_rom_device_set_romd(&pfl->mem, true);
    pfl->wcycle = 0;
    pfl->cmd = 0x00;
```

### 12.3 性能影响

在 UEFI 固件执行期间：
1. **代码执行（flash[0]）**：几乎始终在 ROMD 模式 → TCG 可直接从 RAM 读取指令，性能接近内存执行
2. **变量操作（flash[1]）**：擦除/写入时短暂进入 IO mode → 操作完成后自动回到 ROMD
3. **对比真实硬件**：真实 NOR Flash 读取延迟 ~100ns vs DRAM ~10ns，QEMU 的 ROMD 优化消除了这个差异

---

## 13. CFI 命令状态机详解

### 13.1 状态转换图

```
                    ┌─────────────────────────────┐
                    │     wcycle = 0              │
                    │     ROMD = true             │
                    │     cmd = READ_ARRAY (0x00) │
                    └────────────┬────────────────┘
                                 │ 任何写命令
                                 │ ROMD → false
                                 ▼
              ┌──────────────────────────────────────────┐
              │              wcycle = 0                    │
              │  ┌────┬────┬────┬────┬────┬────┬────┐   │
              │  │0x10│0x20│0x50│0x60│0x70│0x90│0xE8│   │
              │  │Prog│Eras│Clr │Lock│Stat│ID  │Buf │   │
              │  └──┬─┴──┬─┴──┬─┴────┴──┬─┴──┬─┴──┬─┘   │
              └─────┼────┼────┼─────────┼────┼────┼──────┘
                    │    │    │         │    │    │
                    ▼    ▼    │         │    │    ▼
             wcycle=1  wc=1  ROMD=true wc=0  wc=0  wcycle=1
              (data)  (conf)  reset    done  done  (count)
                │       │                           │
                ▼       ▼                           ▼
            program  confirm                    wcycle=2
            + update  0xD0                      (data×N)
            + ROMD    erase                         │
              true    + ROMD                        ▼
                       true                     wcycle=3
                                                (confirm)
                                                  0xD0
                                                 flush
                                                + ROMD true
```

### 13.2 支持的命令集

| 命令 | 代码 | 功能 | 写周期数 |
|------|------|------|----------|
| Read Array | 0xFF/0x00 | 回到正常读模式 | 0 |
| Single Byte Program | 0x10/0x40 | 编程单字节 | 2 |
| Block Erase | 0x20 + 0xD0 | 擦除整个扇区 | 2 |
| Clear Status | 0x50 | 清除状态位 | 1 |
| Block Lock | 0x60 + 0xD0/0x01 | 块锁定/解锁 | 2 |
| Read Status | 0x70 | 读状态寄存器 | 1 |
| Read ID | 0x90 | 读设备 ID | 1 |
| CFI Query | 0x98 | 读 CFI 表 | 1 |
| Buffer Write | 0xE8 + N + data + 0xD0 | 批量编程 | 4 |

---

## 14. PFlash 与 BlockBackend 的持久化

### 14.1 数据流

```
Guest CPU Write → pflash_write() 状态机
    → pflash_data_write(): 修改 pfl->storage (RAM)
    → pflash_update(): blk_pwrite() → BlockBackend → 文件
```

### 14.2 块写缓冲区 (Buffer Write)

```c
// hw/block/pflash_cfi01.c:404-431
static void pflash_blk_write_start(PFlashCFI01 *pfl, hwaddr offset)
{
    // 备份原始数据到缓冲区
    pfl->blk_offset = offset & ~(pfl->writeblock_size - 1);
    memcpy(pfl->blk_bytes, pfl->storage + pfl->blk_offset, pfl->writeblock_size);
}

static void pflash_blk_write_flush(PFlashCFI01 *pfl)
{
    // 提交：缓冲区 → storage → 块设备
    memcpy(pfl->storage + pfl->blk_offset, pfl->blk_bytes, pfl->writeblock_size);
    pflash_update(pfl, pfl->blk_offset, pfl->writeblock_size);
}

static void pflash_blk_write_abort(PFlashCFI01 *pfl)
{
    // 回滚：丢弃缓冲区
    pfl->blk_offset = -1;
}
```

### 14.3 热迁移状态

```c
// hw/block/pflash_cfi01.c:114-130
static const VMStateDescription vmstate_pflash = {
    .name = "pflash_cfi01",
    .fields = (const VMStateField[]) {
        VMSTATE_UINT8(wcycle, ...),   // 写状态机位置
        VMSTATE_UINT8(cmd, ...),      // 当前命令
        VMSTATE_UINT8(status, ...),   // 状态寄存器
        VMSTATE_UINT64(counter, ...), // 块写计数器
    },
    .subsections = { &vmstate_pflash_blk_write, ... },
};
```

Flash 内容本身不迁移（由块设备层处理），只迁移命令状态机状态。

---

## 15. FDT 中的 Flash 描述

### 15.1 非安全模式（单节点）

```dts
/flash@0 {
    compatible = "cfi-flash";
    reg = <0x0 0x0 0x0 0x4000000   /* flash0: 0~64MB */
           0x0 0x4000000 0x0 0x4000000>;  /* flash1: 64~128MB */
    bank-width = <4>;
};
```

### 15.2 安全模式（双节点）

```dts
/secflash@0 {
    compatible = "cfi-flash";
    reg = <0x0 0x0 0x0 0x4000000>;
    bank-width = <4>;
    status = "disabled";         /* Non-secure 不可见 */
    secure-status = "okay";     /* Secure world 可用 */
};

/flash@4000000 {
    compatible = "cfi-flash";
    reg = <0x0 0x4000000 0x0 0x4000000>;
    bank-width = <4>;
};
```

---

## 16. 完整启动时序

```
时间轴 →

T0: QEMU 初始化
    ├── machvirt_init()
    │   ├── virt_flash_create()     → 创建 flash[0], flash[1]
    │   ├── virt_firmware_init()    → 绑定 drive, 映射, 加载固件
    │   ├── create_fw_cfg()         → 创建 fw_cfg 设备
    │   └── arm_load_kernel()       → 注册 reset handler
    │       └── arm_setup_firmware_boot() → kernel/initrd → fw_cfg
    │
T1: 系统复位 (qemu_system_reset)
    ├── pflash_cfi01_system_reset() → cmd=0, ROMD=true
    ├── arm_cpu_reset()
    │   ├── pstate = EL3h (最高实现 EL)
    │   ├── env->cp15.rvbar = cpu->rvbar_prop (= 0)
    │   └── env->pc = 0  ← 指向 Flash 基址
    └── do_cpu_reset() → info==NULL, PC 不修改
    
T2: CPU 开始执行
    ├── PC = 0x0000_0000 (Flash0 内容)
    ├── Flash 处于 ROMD 模式 → 直接从 RAM 读指令
    └── EDK2 SEC Phase 开始执行
    
T3: EDK2 SEC → PEI
    ├── 初始化临时 RAM (SRAM 或 Flash 内嵌)
    ├── 发现物理内存
    └── 通过 fw_cfg 获取配置
    
T4: EDK2 PEI → DXE
    ├── 初始化主存
    ├── 安装 HOB (Hand-Off Block)
    └── 加载 DXE Core
    
T5: EDK2 DXE
    ├── 枚举 PCI 设备
    ├── 加载 VirtIO 驱动
    ├── 初始化网络/存储
    ├── 安装 UEFI Runtime Services
    └── 初始化 Variable Store (flash[1])
    
T6: EDK2 BDS → OS Boot
    ├── 检查 fw_cfg 是否有 kernel
    │   ├── 有 → 直接加载 kernel/initrd/cmdline
    │   └── 无 → 搜索启动设备 (Boot####)
    └── ExitBootServices() → 控制权交给 OS
```

---

## 17. 与真实硬件的对比分析

### 17.1 Flash 设备对比

| 特性 | 真实 NOR Flash | QEMU PFlash CFI01 |
|------|---------------|-------------------|
| 读延迟 | ~100ns | 0（ROMD，等同 RAM） |
| 编程延迟 | ~200μs/word | 瞬时完成 |
| 擦除延迟 | ~1s/block | 瞬时完成 |
| 磨损均衡 | 10万次擦写限制 | 无限制 |
| CFI 兼容 | 标准规范 | 部分实现（Intel P30 模拟） |
| 时序模拟 | N/A | 不模拟（注释：does not support timings） |
| Suspend/Resume | 支持 | 不实现 |
| 数据保护 | 硬件 OTP/Lock | 不实现 |

### 17.2 启动流程对比

| 阶段 | 真实平台 | QEMU virt |
|------|---------|-----------|
| 复位向量 | 硬件引脚决定 RVBAR | rvbar 属性默认 0 |
| Boot ROM | 芯片内置不可修改 | 无（直接执行 Flash） |
| ATF BL1 | 从 Boot ROM 跳转 | 可选（secure + firmware） |
| ATF BL2 | 加载 BL31/BL33 | 可选（自定义固件） |
| UEFI (BL33) | 由 ATF 加载 | 直接从 Flash 执行 |
| 变量存储 | 独立 SPI/NOR Flash | flash[1] 同一设备 |

### 17.3 安全启动对比

| 特性 | 真实平台 | QEMU |
|------|---------|------|
| 安全启动 ROM | 不可篡改 | Flash0 映射到 secure view |
| RoT (Root of Trust) | 硬件 fuse | 无（信任 Flash 内容） |
| 安全存储 | TrustZone + Crypto | 仅地址空间隔离 |
| PSCI 实现 | ATF BL31 | QEMU 内置（或禁用由固件接管） |
| 时钟/电源控制 | SCP firmware | 无（PSCI 简化为状态切换） |

### 17.4 简化与差异总结

1. **无 Boot ROM**：真实芯片有内置 ROM（不可修改），QEMU 直接从 Flash 启动
2. **无时序模拟**：Flash 擦除/编程瞬时完成，不影响功能正确性
3. **无磨损模型**：无限擦写，不模拟坏块
4. **简化安全链**：无 OTP/fuse/TrustZone 硬件密钥
5. **RVBAR 可配置**：真实硬件由引脚决定，QEMU 通过属性设置
6. **fw_cfg 旁路**：真实平台无此通道，QEMU 用来简化固件发现

---

## 18. 总结

QEMU ARM64 virt 机型的固件启动链是一个精心设计的模拟系统：

**架构层面**：
- 两块 PFlash CFI01 设备分别承载固件代码和 UEFI 变量
- Flash 放在物理地址 0，配合 RVBAR 默认值实现零配置启动
- ROM Device 优化确保固件执行性能接近原生

**安全层面**：
- Secure/Non-Secure 地址空间隔离保护固件代码
- 固件存在时可禁用 QEMU 内置 PSCI，让固件自行处理
- Flash 访问属性检查阻止非授权写入

**灵活性**：
- 支持 -pflash（完整 UEFI + 变量持久化）和 -bios（简化测试）
- fw_cfg 允许固件动态获取 kernel/initrd/cmdline
- 可以部署自定义固件（ATF + UEFI）或使用 QEMU 内置 EDK2

**与硬件的关键差异**：
- 无 Boot ROM、无时序、无磨损 → 功能正确但不适合时序/磨损相关测试
- fw_cfg 是 QEMU 特有通道 → 需要固件支持（EDK2 已内置）
- 安全链简化 → 不适合验证真实 Secure Boot 密钥链

---

## 附录 A：关键源码文件索引

| 文件 | 行号 | 功能 |
|------|------|------|
| hw/block/pflash_cfi01.c:60-91 | PFlashCFI01 结构体 |
| hw/block/pflash_cfi01.c:461-658 | pflash_write() 命令状态机 |
| hw/block/pflash_cfi01.c:796-869 | pflash_cfi01_realize() 初始化 |
| hw/block/pflash_cfi01.c:871-890 | pflash_cfi01_system_reset() |
| hw/block/pflash_cfi01.c:661-687 | secure 访问控制 |
| hw/arm/virt.c:173-175 | VIRT_FLASH memmap 定义 |
| hw/arm/virt.c:1571-1602 | virt_flash_create() |
| hw/arm/virt.c:1604-1639 | virt_flash_map() secure 分离 |
| hw/arm/virt.c:1685-1733 | virt_firmware_init() 主流程 |
| hw/arm/virt.c:1735-1755 | create_fw_cfg() |
| hw/arm/virt.c:2663-2682 | firmware_loaded → PSCI conduit |
| hw/arm/boot.c:655-741 | do_cpu_reset() |
| hw/arm/boot.c:1115-1171 | arm_setup_firmware_boot() |
| hw/arm/boot.c:1173-1265 | arm_load_kernel() 分支 |
| target/arm/cpu.c:405-414 | RVBAR 采样与 PC 设置 |
| target/arm/cpu.c:1509-1510 | rvbar 属性注册 |
| target/arm/helper.c:6872-6875 | RVBAR_EL3 系统寄存器 |

## 附录 B：命令行快速参考

```bash
# 标准 UEFI 启动
qemu-system-aarch64 -M virt,secure=on -cpu cortex-a72 -m 2G \
    -drive if=pflash,format=raw,file=AAVMF_CODE.fd,readonly=on \
    -drive if=pflash,format=raw,file=AAVMF_VARS.fd \
    -drive file=disk.qcow2,if=virtio

# UEFI + 直接内核加载（通过 fw_cfg）
qemu-system-aarch64 -M virt -cpu max -m 4G \
    -drive if=pflash,format=raw,file=AAVMF_CODE.fd,readonly=on \
    -drive if=pflash,format=raw,file=AAVMF_VARS.fd \
    -kernel Image -initrd rootfs.cpio.gz -append "console=ttyAMA0"

# 简化 BIOS 模式
qemu-system-aarch64 -M virt -cpu max -m 1G \
    -bios AAVMF_CODE.fd -nographic

# 自定义复位向量
qemu-system-aarch64 -M virt -cpu max,rvbar=0x80000000 -m 1G \
    -device loader,file=firmware.bin,addr=0x80000000
```
