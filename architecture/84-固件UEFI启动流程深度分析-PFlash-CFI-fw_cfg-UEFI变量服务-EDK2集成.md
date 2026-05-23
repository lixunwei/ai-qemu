# 固件/UEFI 启动流程深度分析

> 文档编号：84  
> 分析目标：pflash CFI Flash、fw_cfg 固件配置接口、UEFI 变量服务、EDK2 集成  
> 源码版本：QEMU 11.0.50  
> 核心文件：hw/block/pflash_cfi01.c、hw/nvram/fw_cfg.c、hw/uefi/var-service-*.c、hw/arm/virt.c

---

## 一、概述

当 QEMU virt 机型使用 `-bios` 或 `-drive if=pflash` 加载固件（如 EDK2/UEFI）时，启动流程与直接内核启动（doc 83）完全不同：

```
CPU 从 Flash 地址 0x0 开始执行 → UEFI 固件自身初始化 →
  通过 fw_cfg 获取内核/initrd/cmdline → UEFI 加载内核 →
  ExitBootServices() → 跳转内核入口
```

涉及三个关键子系统：
1. **PFlash CFI01**：模拟 NOR Flash，存放固件映像
2. **fw_cfg**：QEMU→Guest 数据传递接口
3. **UEFI 变量服务**：运行时变量存储（EFI Variable）

---

## 二、PFlash CFI01 — NOR Flash 模拟

### 2.1 设备概述

PFlash CFI01 模拟 Intel 兼容的 CFI（Common Flash Interface）NOR Flash。在 virt 机型中，Flash 占据内存映射最低 128MB（0x0000_0000 ~ 0x0800_0000），分为两个各 64MB 的设备：

```
0x0000_0000 ┬─ flash0 (64MB) ─ 安全世界固件（UEFI Code）
            │   sector-length: 256KB
            │   仅在 secure_sysmem 可见（有 TrustZone 时）
0x0400_0000 ┼─ flash1 (64MB) ─ UEFI 变量存储
            │   NonSecure + Secure 均可见
0x0800_0000 ┴─ Flash 区域结束
```

### 2.2 设备创建

```c
// hw/arm/virt.c:1573-1601
static PFlashCFI01 *virt_flash_create1(VirtMachineState *vms, ...) {
    DeviceState *dev = qdev_new(TYPE_PFLASH_CFI01);
    qdev_prop_set_uint64(dev, "sector-length", 256 * KiB);
    qdev_prop_set_uint8(dev, "width", 4);         // 总线宽度 4 字节
    qdev_prop_set_uint8(dev, "device-width", 2);   // 芯片宽度 2 字节
    qdev_prop_set_uint16(dev, "id0", 0x89);        // Intel 制造商 ID
    qdev_prop_set_uint16(dev, "id1", 0x18);        // 设备 ID
    ...
}

vms->flash[0] = virt_flash_create1(vms, "virt.flash0", "pflash0");
vms->flash[1] = virt_flash_create1(vms, "virt.flash1", "pflash1");
```

### 2.3 内存映射

```c
// hw/arm/virt.c:1620-1638
static void virt_flash_map(VirtMachineState *vms, ...) {
    hwaddr flashsize = vms->memmap[VIRT_FLASH].size / 2;  // 64MB each
    hwaddr flashbase = vms->memmap[VIRT_FLASH].base;       // 0x0

    // flash0: 仅安全世界可见
    virt_flash_map1(vms->flash[0], flashbase, flashsize, secure_sysmem);
    // flash1: 所有世界可见
    virt_flash_map1(vms->flash[1], flashbase + flashsize, flashsize, sysmem);
}
```

### 2.4 PFlashCFI01 结构体

```c
// hw/block/pflash_cfi01.c:60-91
struct PFlashCFI01 {
    SysBusDevice parent_obj;
    BlockBackend *blk;           // 后端块设备（持久存储）
    uint32_t nb_blocs;           // 扇区数
    uint64_t sector_len;         // 扇区大小（256KB）
    uint8_t bank_width;          // 总线宽度
    uint8_t device_width;        // 芯片宽度
    uint8_t wcycle;              // 写周期状态（0=读模式）
    uint8_t cmd;                 // 当前命令
    uint8_t status;              // 状态寄存器
    uint16_t ident0..3;          // 设备识别 ID
    uint8_t cfi_table[0x52];     // CFI 查询表
    MemoryRegion mem;            // ROM 设备内存区域
    void *storage;               // 实际数据存储
    unsigned char *blk_bytes;    // 块写缓冲区
};
```

### 2.5 Flash 命令状态机

PFlash 使用 ROM Device 模式（`memory_region_init_rom_device`）——默认情况下读操作直接从 RAM 返回数据（ROMD 模式），写操作触发 MMIO 回调进入命令处理：

```c
// 命令状态机（Intel Flash 协议）
switch (cmd) {
    case 0x10/0x40: /* 单字节编程 */
    case 0x20:      /* 扇区擦除 — memset(sector, 0xFF) */
    case 0x50:      /* 清除状态位 */
    case 0x60:      /* 块锁定/解锁 */
    case 0x70:      /* 读状态寄存器 */
    case 0x90:      /* 读设备 ID */
    case 0x98:      /* CFI 查询 — 返回 cfi_table[] */
    case 0xe8:      /* 缓冲写入 */
    case 0xff:      /* 返回读数组模式（ROMD） */
}
```

**状态转换**：
```
读模式(ROMD=true) ──写命令──→ I/O模式(ROMD=false) ──0xFF──→ 读模式
       ↑                            │
       └────────────────────────────┘
```

### 2.6 Realize 流程

```c
// hw/block/pflash_cfi01.c:796-869
static void pflash_cfi01_realize(DeviceState *dev, Error **errp) {
    total_len = pfl->sector_len * pfl->nb_blocs;
    
    // 1. 创建 ROM 设备内存区域（支持 ROMD 切换）
    memory_region_init_rom_device(&pfl->mem, ..., total_len);
    pfl->storage = memory_region_get_ram_ptr(&pfl->mem);
    
    // 2. 如果有后端块设备，读取内容到 storage
    if (pfl->blk) {
        blk_check_size_and_read_all(pfl->blk, dev, pfl->storage, total_len);
    }
    
    // 3. 填充 CFI 查询表
    pflash_cfi01_fill_cfi_table(pfl);
    
    // 4. 初始化状态
    pfl->status = 0x80;  // WSM Ready
    pfl->cmd = 0x00;     // READ_ARRAY
}
```

---

## 三、virt_firmware_init — 固件加载入口

```c
// hw/arm/virt.c:1685-1733
static bool virt_firmware_init(VirtMachineState *vms,
                               MemoryRegion *sysmem,
                               MemoryRegion *secure_sysmem) {
    // 1. 将 -drive if=pflash 映射到 flash 设备属性
    for (i = 0; i < 2; i++) {
        pflash_cfi01_legacy_drive(vms->flash[i], drive_get(IF_PFLASH, 0, i));
    }
    
    // 2. 映射 flash 到地址空间
    virt_flash_map(vms, sysmem, secure_sysmem);
    
    // 3. 检查固件加载方式
    pflash_blk0 = pflash_cfi01_get_blk(vms->flash[0]);
    bios_name = MACHINE(vms)->firmware;  // -bios 参数
    
    if (bios_name) {
        // -bios: 将固件镜像加载到 flash0 的内存区域
        mr = sysbus_mmio_get_region(SYS_BUS_DEVICE(vms->flash[0]), 0);
        load_image_mr(fname, mr);
    }
    
    return pflash_blk0 || bios_name;  // 有固件就返回 true
}
```

**两种固件指定方式**：

| 方式 | 命令行 | 效果 |
|------|--------|------|
| `-bios` | `-bios QEMU_EFI.fd` | 将文件直接加载到 flash0 MemoryRegion |
| `-drive if=pflash` | `-drive file=code.fd,if=pflash,format=raw,readonly=on` | flash0 使用块设备后端（支持持久写入） |

---

## 四、固件启动对 PSCI 和 CPU 配置的影响

### 4.1 PSCI 禁用

```c
// hw/arm/virt.c:2676-2682
if (vms->secure && firmware_loaded) {
    // 有安全固件 → PSCI 由固件自己实现（如 TF-A）
    vms->psci_conduit = QEMU_PSCI_CONDUIT_DISABLED;
} else if (vms->virt) {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_SMC;
} else {
    vms->psci_conduit = QEMU_PSCI_CONDUIT_HVC;
}
```

### 4.2 ACPI GED 设备

```c
// hw/arm/virt.c:2912-2919
if (aarch64 && firmware_loaded && virt_is_acpi_enabled(vms)) {
    // UEFI 固件启动 → 使用 ACPI GED（Generic Event Device）
    vms->acpi_dev = create_acpi_ged(vms);
} else {
    // 直接内核启动 → 使用 GPIO 设备
    create_gpio_devices(vms, VIRT_GPIO, sysmem);
}
```

### 4.3 firmware_boot 路径

```c
// hw/arm/boot.c:1115-1171
static void arm_setup_firmware_boot(ARMCPU *cpu, struct arm_boot_info *info) {
    // DTB 放到 RAM 基地址，供固件拾取
    if (have_dtb(info)) {
        info->dtb_start = info->loader_start;
    }
    
    if (info->kernel_filename) {
        // 有 -kernel 时通过 fw_cfg 传递给固件
        FWCfgState *fw_cfg = fw_cfg_find();
        load_image_to_fw_cfg(fw_cfg, FW_CFG_KERNEL_SIZE,
                             FW_CFG_KERNEL_DATA, info->kernel_filename, ...);
        load_image_to_fw_cfg(fw_cfg, FW_CFG_INITRD_SIZE,
                             FW_CFG_INITRD_DATA, info->initrd_filename, false);
        fw_cfg_add_string(fw_cfg, FW_CFG_CMDLINE_DATA, info->kernel_cmdline);
    }
    
    // CPU 从地址 0x0 启动（Flash 起始地址）
    // env->boot_info 不设置 → do_cpu_reset() 不修改 PC
}
```

---

## 五、fw_cfg — 固件配置接口

### 5.1 概述

fw_cfg 是 QEMU 专有的 Host→Guest 数据传递机制。它不模拟真实硬件，而是提供一个简单的 key-value 接口，固件通过 MMIO 读取 QEMU 注入的数据。

### 5.2 内存映射

```
VIRT_FW_CFG = 0x0902_0000, size = 0x18

偏移      寄存器           功能
0x00      Data             数据读取（8位宽，按字节顺序读取）
0x08      Selector         选择 key（16位写入）
0x10      DMA Address      DMA 控制（64位，高32+低32）
```

### 5.3 数据项

```c
// 预定义 key（hw/nvram/fw_cfg.c:80-107）
FW_CFG_SIGNATURE    = 0x0000  // "QEMU" 魔术值
FW_CFG_ID           = 0x0001  // 版本标识
FW_CFG_NB_CPUS      = 0x0005  // CPU 数量
FW_CFG_KERNEL_SIZE  = 0x0008  // 内核大小
FW_CFG_KERNEL_DATA  = 0x0011  // 内核镜像数据
FW_CFG_INITRD_SIZE  = 0x000b  // initrd 大小
FW_CFG_INITRD_DATA  = 0x0012  // initrd 数据
FW_CFG_CMDLINE_SIZE = 0x0014  // 命令行长度
FW_CFG_CMDLINE_DATA = 0x0015  // 命令行字符串
FW_CFG_FILE_DIR     = 0x0019  // 动态文件目录
```

### 5.4 访问协议

**PIO 模式**（慢，兼容）：
```
1. Guest 写 Selector = key    → 选中数据项
2. Guest 读 Data 多次         → 按字节顺序返回数据，offset 自动递增
```

**DMA 模式**（快，批量）：
```
1. Guest 构造 FWCfgDmaAccess 结构（address, length, control）
2. Guest 写 DMA Address = 结构的物理地址
3. QEMU 读取结构，执行 SELECT/READ/WRITE/SKIP 操作
4. QEMU 清除 control 字段通知完成
```

### 5.5 DMA 传输实现

```c
// hw/nvram/fw_cfg.c:334-439
static void fw_cfg_dma_transfer(FWCfgState *s) {
    // 读取 guest 提交的 DMA 描述符
    dma_memory_read(s->dma_as, dma_addr, &dma, sizeof(dma));
    
    // SELECT: 选择数据项
    if (dma.control & FW_CFG_DMA_CTL_SELECT) {
        fw_cfg_select(s, dma.control >> 16);
    }
    
    // READ: QEMU → Guest 数据传输
    if (dma.control & FW_CFG_DMA_CTL_READ) {
        dma_memory_write(s->dma_as, dma.address,
                        &e->data[s->cur_offset], len);
    }
    
    // 完成：清除 control
    stl_be_dma(s->dma_as, dma_addr + offsetof(control), 0);
}
```

### 5.6 动态文件接口

除预定义 key 外，fw_cfg 支持动态注册"文件"（`fw_cfg_add_file()`），固件通过 `FW_CFG_FILE_DIR` 获取文件目录，然后按 key 读取内容。常见文件：

| 文件名 | 内容 |
|--------|------|
| `etc/smbios/smbios-tables` | SMBIOS 表 |
| `etc/smbios/smbios-anchor` | SMBIOS 入口点 |
| `etc/acpi/tables` | ACPI 表 |
| `etc/acpi/rsdp` | ACPI RSDP |
| `etc/e820` | 内存映射 |

---

## 六、UEFI 变量服务

### 6.1 概述

QEMU 11.0 新增了 `uefi-vars` 设备，直接在 QEMU 中实现 UEFI 运行时变量服务，无需固件自己管理 NOR Flash 变量存储。

### 6.2 设备类型

| 类型 | 用途 |
|------|------|
| `TYPE_UEFI_VARS_SYSBUS` | ARM sysbus 设备（virt 机型用） |
| `TYPE_UEFI_VARS_X64` | x86 ISA 设备 |

### 6.3 MMIO 寄存器

```c
// include/hw/uefi/var-service-api.h:17-26
UEFI_VARS_REG_MAGIC               = 0x00  // 16位，magic=0xef1
UEFI_VARS_REG_CMD_STS              = 0x02  // 16位，命令/状态
UEFI_VARS_REG_BUFFER_SIZE          = 0x04  // 32位，缓冲区大小
UEFI_VARS_REG_DMA_BUFFER_ADDR_LO   = 0x08  // 32位，DMA 地址低
UEFI_VARS_REG_DMA_BUFFER_ADDR_HI   = 0x0c  // 32位，DMA 地址高
UEFI_VARS_REG_PIO_BUFFER_TRANSFER  = 0x10  // 8-64位，PIO 传输
UEFI_VARS_REG_PIO_BUFFER_CRC32C    = 0x18  // 32位，CRC32C 校验
UEFI_VARS_REG_FLAGS                = 0x1c  // 32位，标志
```

### 6.4 命令流程

```
Guest (UEFI Runtime):
  1. 写 BUFFER_ADDR = MM Communicate 缓冲区物理地址
  2. 写 CMD_STS = UEFI_VARS_CMD_DMA_MM
  
QEMU (var-service-core.c:uefi_vars_cmd_mm):
  3. 读取 guest 缓冲区中的 MM Communicate 头
  4. 解析 GUID 和操作（GetVariable/SetVariable/...）
  5. 在内部数据库中执行操作
  6. 将结果写回 guest 缓冲区
  7. 设置 CMD_STS = SUCCESS
```

### 6.5 在 virt 机型中的注册

```c
// hw/arm/virt.c:3832
machine_class_allow_dynamic_sysbus_dev(mc, TYPE_UEFI_VARS_SYSBUS);
```

通过 `-device uefi-vars-sysbus` 添加，或由 EDK2 固件通过设备树发现。

---

## 七、Flash DT 节点

### 7.1 无 TrustZone（单节点）

```dts
/flash@0 {
    compatible = "cfi-flash";
    reg = <0x0 0x0 0x0 0x4000000     /* flash0: 64MB */
           0x0 0x4000000 0x0 0x4000000>; /* flash1: 64MB */
    bank-width = <4>;
};
```

### 7.2 有 TrustZone（双节点）

```dts
/secflash@0 {
    compatible = "cfi-flash";
    reg = <0x0 0x0 0x0 0x4000000>;
    bank-width = <4>;
    status = "disabled";        /* NonSecure 看不到 */
    secure-status = "okay";     /* Secure 可访问 */
};

/flash@4000000 {
    compatible = "cfi-flash";
    reg = <0x0 0x4000000 0x0 0x4000000>;
    bank-width = <4>;
};
```

---

## 八、完整启动时序

### 8.1 UEFI 固件启动

```
QEMU 启动
    │
    ├─ virt_firmware_init()
    │     ├─ 创建 flash0/flash1 → 映射到 0x0
    │     ├─ 加载固件到 flash0（-bios 或 -drive if=pflash）
    │     └─ return true (firmware_loaded)
    │
    ├─ PSCI conduit = DISABLED（如果 secure=on）
    │
    ├─ create_fw_cfg() → 0x0902_0000
    │     └─ 注册 KERNEL_DATA/INITRD_DATA/CMDLINE_DATA
    │
    ├─ arm_load_kernel() → arm_setup_firmware_boot()
    │     └─ 不设置 boot_info → CPU 从 0x0 启动
    │
    ▼
CPU 复位: do_cpu_reset()
    │
    ├─ cpu_reset()
    ├─ boot_info == NULL → 不修改 PC
    └─ PC = 0x0（Flash 起始地址）
    
UEFI 固件执行
    │
    ├─ SEC Phase: 从 Flash 0x0 开始
    ├─ PEI Phase: 初始化内存、发现硬件
    ├─ DXE Phase: 加载驱动
    │     ├─ 通过 fw_cfg DMA 读取 KERNEL_DATA
    │     ├─ 通过 fw_cfg 读取 INITRD_DATA
    │     └─ 通过 fw_cfg 读取 CMDLINE_DATA
    ├─ BDS Phase: 选择启动项
    │     └─ 使用 Linux Boot Protocol 或 EFI Stub
    └─ ExitBootServices() → 跳转内核
```

### 8.2 QEMU 命令行示例

```bash
# 基本 UEFI 启动（EDK2）
qemu-system-aarch64 -M virt -cpu cortex-a72 -m 2G \
  -drive file=QEMU_EFI.fd,if=pflash,format=raw,readonly=on \
  -drive file=efivars.fd,if=pflash,format=raw \
  -kernel Image -initrd rootfs.cpio -append "console=ttyAMA0"

# 或用 -bios
qemu-system-aarch64 -M virt -cpu cortex-a72 -m 2G \
  -bios QEMU_EFI.fd \
  -kernel Image -initrd rootfs.cpio
```

**注意**：`-bios` 不支持变量持久化（flash0 只读加载），`-drive if=pflash` 支持持久化（flash1 可写）。

---

## 九、与真实硬件对比

| 方面 | QEMU virt | 真实 ARM 服务器 |
|------|-----------|-----------------|
| Flash 类型 | CFI01 (Intel) 模拟 | SPI NOR/NAND Flash |
| Flash 大小 | 2×64MB (128MB) | 通常 32-64MB |
| Flash 地址 | 0x0 (固定) | SoC 决定 |
| UEFI 固件 | EDK2 (开源) | 厂商定制 UEFI |
| Secure Boot | 可选模拟 | 硬件 eFuse + ROM |
| 变量存储 | flash1 或 uefi-vars 设备 | Flash 分区 |
| fw_cfg | QEMU 专有接口 | 不存在 |
| SMBIOS | fw_cfg 传递 | 固件自行构建 |
| ACPI 表 | QEMU 生成→fw_cfg | 固件自行构建 |
| PSCI | QEMU 内置或禁用 | TF-A (BL31) 实现 |

### 9.1 关键差异

1. **fw_cfg 是 QEMU 特有的**：真实硬件不存在这个接口。EDK2 专门为 QEMU 编译了 fw_cfg 驱动来读取内核和 ACPI 表。

2. **CFI Flash 模拟简化**：
   - 真实 NOR Flash 有复杂的时序要求（编程/擦除需要微秒到毫秒级等待）
   - QEMU 的 pflash 是即时完成的——`memset(sector, 0xFF)` 立即返回
   - 没有磨损均衡、坏块管理等

3. **安全启动链不完整**：
   - 真实系统：ROM → BL1 → BL2 → BL31 (TF-A) → BL33 (UEFI)
   - QEMU：直接从 Flash 0x0 启动 UEFI，跳过 TF-A（除非明确配置 secure=on 并加载 TF-A 固件）

4. **UEFI 变量服务**：
   - 真实系统：变量通过 SMM 或 TrustZone 安全隔离
   - QEMU：`uefi-vars` 设备直接在 hypervisor 层实现，或直接写 flash1

---

## 十、源文件索引

| 文件 | 行数 | 关键内容 |
|------|------|----------|
| `hw/block/pflash_cfi01.c` | 1038 | CFI Flash 模拟：命令状态机、读写、CFI 查询 |
| `hw/nvram/fw_cfg.c` | 1304 | fw_cfg 接口：key-value 存储、DMA 传输、MMIO |
| `hw/uefi/var-service-core.c` | 328 | UEFI 变量服务核心：MM Communicate 分发 |
| `hw/uefi/var-service-sysbus.c` | 126 | sysbus 封装（ARM 用） |
| `include/hw/uefi/var-service-api.h` | 48 | MMIO 寄存器定义 |
| `include/hw/block/flash.h` | 53 | PFlash 类型声明 |
| `hw/arm/virt.c:1573-1733` | — | flash 创建/映射/FDT/固件加载 |
| `hw/arm/virt.c:2663-2682` | — | firmware_loaded 对 PSCI 的影响 |
| `hw/arm/boot.c:1115-1171` | — | arm_setup_firmware_boot() |

| 函数 | 位置 | 职责 |
|------|------|------|
| `virt_firmware_init()` | virt.c:1685 | 固件加载入口 |
| `virt_flash_create()` | virt.c:1598 | 创建 flash0/flash1 |
| `virt_flash_map()` | virt.c:1620 | 映射 flash 到地址空间 |
| `virt_flash_fdt()` | virt.c:1641 | 生成 flash DT 节点 |
| `pflash_cfi01_realize()` | pflash_cfi01.c:796 | Flash 设备初始化 |
| `pflash_write()` | pflash_cfi01.c:461 | Flash 命令状态机 |
| `pflash_read()` | pflash_cfi01.c:262 | Flash 读取处理 |
| `create_fw_cfg()` | virt.c:1735 | 创建 fw_cfg 设备 |
| `fw_cfg_dma_transfer()` | fw_cfg.c:334 | DMA 传输处理 |
| `fw_cfg_init_mem_dma()` | fw_cfg.c:1091 | 初始化带 DMA 的 fw_cfg |
| `arm_setup_firmware_boot()` | boot.c:1115 | 固件启动模式配置 |
| `uefi_vars_cmd_mm()` | var-service-core.c:62 | UEFI 变量命令分发 |
