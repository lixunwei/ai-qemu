# ARM 内核加载与启动流程深度分析

> 文档编号：83  
> 分析目标：hw/arm/boot.c 的内核加载、bootloader 桩代码、CPU 复位、DTB 放置  
> 源码版本：QEMU 11.0.50  
> 核心文件：hw/arm/boot.c (1310行)、include/hw/arm/boot.h (240行)

---

## 一、概述

`hw/arm/boot.c` 是 QEMU ARM 通用的内核加载和启动基础设施。它负责：
- 加载 Linux 内核（ELF / Image / uImage / 压缩格式）
- 加载 initrd（ramdisk）
- 生成或加载设备树（DTB）
- 在 RAM 底部写入 bootloader 桩代码
- 配置 CPU 复位处理器
- 处理多核 SMP 启动（PSCI 或自旋循环）

---

## 二、两种启动模式

```c
// hw/arm/boot.c:1208
if (!info->kernel_filename || info->firmware_loaded) {
    arm_setup_firmware_boot(cpu, info);   // 固件启动
} else {
    arm_setup_direct_kernel_boot(cpu, info);  // 直接内核启动
}
```

### 2.1 固件启动模式（arm_setup_firmware_boot）

触发条件：使用 `-bios` 或 `-drive if=pflash` 加载了固件

```
CPU 从 0x0 (Flash) 开始执行 → UEFI/固件初始化 →
  通过 fw_cfg 获取内核/initrd → 固件加载并跳转内核
```

内核/initrd 通过 fw_cfg 暴露给固件：
```c
load_image_to_fw_cfg(fw_cfg, FW_CFG_KERNEL_SIZE, FW_CFG_KERNEL_DATA,
                     info->kernel_filename, try_decompressing_kernel);
load_image_to_fw_cfg(fw_cfg, FW_CFG_INITRD_SIZE, FW_CFG_INITRD_DATA,
                     info->initrd_filename, false);
fw_cfg_add_string(fw_cfg, FW_CFG_CMDLINE_DATA, info->kernel_cmdline);
```

### 2.2 直接内核启动模式（arm_setup_direct_kernel_boot）

触发条件：使用 `-kernel` 且没有加载固件

```
QEMU 将 bootloader桩 + kernel + initrd + DTB 写入 RAM →
  CPU 从 bootloader桩 开始 → 跳转到内核入口
```

---

## 三、直接内核启动的内存布局

### 3.1 AArch64 内存布局

```
RAM base (0x4000_0000 for virt)
│
├─ 0x0000: Bootloader 桩代码 (≤4KB)
│   └─ ldr x0, dtb_addr; ldr x4, kernel_entry; br x4
│
├─ 0x0008_0000 (KERNEL64_LOAD_ADDR): 内核 Image
│   └─ 或按 Image header 中的 text_offset 放置
│   └─ 如果 text_offset < 4KB 则偏移 2MB 避免覆盖 bootloader
│
├─ initrd 位置 = max(RAM/2 或 128MB, 内核末尾)
│   └─ 页对齐
│
├─ DTB 位置 = ALIGN_UP(initrd末尾, 2MB)
│   └─ AArch64 内核映射 2MB 对齐的 FDT 区域
│
└─ RAM 末尾
```

### 3.2 AArch32 内存布局

```
RAM base
│
├─ 0x0000: Bootloader 桩代码
│   └─ mov r0, #0; ldr r1, board_id; ldr r2, dtb/atags; ldr pc, entry
│
├─ 0x0100 (KERNEL_ARGS_ADDR): ATAGS（无 DTB 时）
│
├─ 0x0001_0000 (KERNEL_LOAD_ADDR): 32位内核 zImage
│
├─ initrd 位置（同上）
│
├─ DTB 位置 = ALIGN_UP(initrd末尾, 4KB)
│
└─ RAM 末尾
```

---

## 四、Bootloader 桩代码

### 4.1 AArch64 Bootloader

```c
static const ARMInsnFixup bootloader_aarch64[] = {
    { 0x580000c0 }, /* ldr x0, arg       ; x0 = DTB 地址 */
    { 0xaa1f03e1 }, /* mov x1, xzr       ; x1 = 0 */
    { 0xaa1f03e2 }, /* mov x2, xzr       ; x2 = 0 */
    { 0xaa1f03e3 }, /* mov x3, xzr       ; x3 = 0 */
    { 0x58000084 }, /* ldr x4, entry     ; x4 = 内核入口 */
    { 0xd61f0080 }, /* br x4             ; 跳转内核 */
    { 0, FIXUP_ARGPTR_LO },     /* arg低32位 */
    { 0, FIXUP_ARGPTR_HI },     /* arg高32位 */
    { 0, FIXUP_ENTRYPOINT_LO }, /* entry低32位 */
    { 0, FIXUP_ENTRYPOINT_HI }, /* entry高32位 */
    { 0, FIXUP_TERMINATOR }
};
```

**Linux AArch64 启动协议要求**：
- x0 = DTB 物理地址
- x1 = x2 = x3 = 0
- 跳转到 Image 入口
- MMU 关闭、D-Cache 关闭

### 4.2 AArch32 Bootloader

```c
static const ARMInsnFixup bootloader[] = {
    { 0xe28fe004 }, /* add lr, pc, #4    ; 返回地址 (board_setup) */
    { 0xe51ff004 }, /* ldr pc, [pc, #-4] ; 跳 board_setup */
    { 0, FIXUP_BOARD_SETUP },
    { 0xe3a00000 }, /* mov r0, #0        */
    { 0xe59f1004 }, /* ldr r1, [pc, #4]  ; r1 = board_id */
    { 0xe59f2004 }, /* ldr r2, [pc, #4]  ; r2 = ATAGS/DTB */
    { 0xe59ff004 }, /* ldr pc, [pc, #4]  ; 跳内核 */
    { 0, FIXUP_BOARDID },
    { 0, FIXUP_ARGPTR_LO },
    { 0, FIXUP_ENTRYPOINT_LO },
    { 0, FIXUP_TERMINATOR }
};
```

### 4.3 SMP 从核 Bootloader（非 PSCI 模式）

```c
static const ARMInsnFixup smpboot[] = {
    /* 启用 GIC CPU Interface */
    { 0xe59f2028 }, /* ldr r2, gic_cpu_if */
    { 0xe3a01001 }, /* mov r1, #1 */
    { 0xe5821000 }, /* str r1, [r2]      ; GICC_CTLR.Enable=1 */
    { 0xe3a010ff }, /* mov r1, #0xff */
    { 0xe5821004 }, /* str r1, [r2, 4]   ; GICC_PMR=0xff */
    /* 自旋等待 */
    { 0, FIXUP_DSB },   /* dsb */
    { 0xe320f003 }, /* wfi */
    { 0xe5901000 }, /* ldr r1, [r0]      ; 读 boot 寄存器 */
    { 0xe1110001 }, /* tst r1, r1 */
    { 0x0afffffb }, /* beq <wfi>         ; 为0继续等 */
    { 0xe12fff11 }, /* bx r1             ; 跳到入口 */
};
```

**注意**：使用 PSCI 时不需要此 smpboot 代码，从核通过 PSCI CPU_ON 直接启动到指定入口。

---

## 五、内核镜像加载

### 5.1 加载优先级

```
1. ELF 格式  → arm_load_elf()
   ├─ 支持 AArch64 (EM_AARCH64) 和 AArch32 (EM_ARM)
   ├─ 自动检测大小端
   └─ BE32 格式需要字节交换

2. uImage 格式 → load_uimage_as()
   └─ U-Boot 包装格式

3. AArch64 raw Image → load_aarch64_image()
   ├─ 支持 gzip 压缩
   ├─ 检查 ARM64 magic ("ARM\x64" at offset 56)
   ├─ 读取 text_offset 和 image_size 头字段
   └─ 根据 text_offset 放置（避开 bootloader 桩）

4. AArch32 raw Image → load_image_targphys_as()
   └─ 直接加载到 KERNEL_LOAD_ADDR (0x10000)
```

### 5.2 AArch64 Image Header 解析

```c
// Image header (Documentation/arm64/booting.txt):
// Offset 8:  text_offset (LE 64-bit)
// Offset 16: image_size  (LE 64-bit)
// Offset 56: magic "ARM\x64"

if (memcmp(buffer + 56, "ARM\x64", 4) == 0) {
    kernel_load_offset = le64(hdrvals[0]);  // text_offset
    kernel_size = le64(hdrvals[1]);          // image_size
    
    // 避免覆盖 bootloader 桩（位于 RAM base）
    if (kernel_load_offset < BOOTLOADER_MAX_SIZE) {
        kernel_load_offset += 2 * MiB;
    }
}
entry = mem_base + kernel_load_offset;
```

---

## 六、initrd 和 DTB 放置策略

### 6.1 initrd 放置

```c
// 策略：RAM/2 或 128MB 中取小值，但不能覆盖内核
info->initrd_start = info->loader_start +
    MIN(info->ram_size / 2, 128 * MiB);

// 如果内核末尾超过此位置，推后
if (image_high_addr) {
    info->initrd_start = MAX(info->initrd_start, image_high_addr);
}
info->initrd_start = TARGET_PAGE_ALIGN(info->initrd_start);
```

### 6.2 DTB 放置

```c
// AArch64：2MB 对齐（内核会映射 2MB 对齐的 FDT 区域）
// AArch32：4KB 对齐（避免 initrd 末尾页面污染）
info->dtb_start = QEMU_ALIGN_UP(initrd_start + initrd_size, align);
```

### 6.3 位置示例（virt 机型，2GB RAM）

```
0x4000_0000: RAM base
0x4000_0000: Bootloader 桩 (40 bytes)
0x4008_0000: Kernel Image (~30MB)
0x4200_0000: initrd (~50MB, at 128MB mark)
0x4540_0000: DTB (~64KB, 2MB aligned after initrd)
0xBFFF_FFFF: RAM end (2GB)
```

---

## 七、CPU 复位处理器（do_cpu_reset）

每次系统复位时为每个 CPU 调用：

```c
static void do_cpu_reset(void *opaque) {
    cpu_reset(cs);  // 基础硬件复位
    
    if (!info) return;  // 固件启动：从 0x0 开始
    
    if (!info->is_linux) {
        // 非 Linux（裸机/RTOS）：直接跳到 entry
        cpu_set_pc(cs, info->entry);
    } else {
        // Linux 启动
        int target_el = has_el2 ? 2 : 1;
        
        // AArch32 安全启动可从 EL3
        if (aarch32 && has_el3 && secure_boot) target_el = 3;
        
        // 模拟固件设置（SCR_EL3/HCR_EL2 等）
        arm_emulate_firmware_reset(cs, target_el);
        
        if (is_primary_cpu) {
            // 主 CPU：PC = bootloader 桩地址（RAM base）
            cpu_set_pc(cs, info->loader_start);
        } else {
            // 从 CPU：调用 secondary_cpu_reset_hook
            // （PSCI 模式下不调用，从核处于 powered-off）
            info->secondary_cpu_reset_hook(cpu, info);
        }
    }
    
    arm_rebuild_hflags(env);  // TCG：重建翻译标志
}
```

---

## 八、DTB 加载与修改

### 8.1 arm_load_dtb()

```c
int arm_load_dtb(hwaddr addr, const struct arm_boot_info *binfo,
                 hwaddr addr_limit, AddressSpace *as, MachineState *ms,
                 ARMCPU *cpu)
{
    // 1. 获取 DTB（用户 -dtb 或 board 的 get_dtb 回调）
    // 2. 修改 /memory 节点（RAM 大小和地址）
    // 3. 添加 /chosen 节点
    //    - bootargs = kernel_cmdline
    //    - linux,initrd-start/end
    //    - stdout-path
    //    - kaslr-seed（随机化）
    // 4. 添加 PSCI 节点（如果启用）
    // 5. 添加 NUMA 信息
    // 6. 调用 board 的 modify_dtb 回调
    // 7. 写入 guest RAM
}
```

### 8.2 PSCI DT 节点生成

```c
static void fdt_add_psci_node(void *fdt, ARMCPU *armcpu) {
    qemu_fdt_add_subnode(fdt, "/psci");
    // compatible: "arm,psci-1.0", "arm,psci-0.2", "arm,psci"
    qemu_fdt_setprop_string(fdt, "/psci", "method", "hvc"/"smc");
    qemu_fdt_setprop_cell(fdt, "/psci", "cpu_suspend", ...);
    qemu_fdt_setprop_cell(fdt, "/psci", "cpu_off", ...);
    qemu_fdt_setprop_cell(fdt, "/psci", "cpu_on", ...);
    qemu_fdt_setprop_cell(fdt, "/psci", "migrate", ...);
}
```

---

## 九、arm_boot_info 关键字段

```c
struct arm_boot_info {
    // 输入参数（由 board 填写）
    uint64_t ram_size;              // RAM 大小
    hwaddr loader_start;            // RAM 起始地址（bootloader桩放这里）
    int psci_conduit;               // PSCI 调用通道
    bool firmware_loaded;           // 是否加载了固件
    bool secure_boot;               // 安全启动
    ARMCPU *primary_cpu;            // 主 CPU
    
    // 回调函数
    void *(*get_dtb)(...);          // 获取 DTB（board 生成）
    void (*modify_dtb)(...);        // DTB 修改钩子
    void (*write_secondary_boot)(); // 从核启动代码
    void (*secondary_cpu_reset_hook)(); // 从核复位钩子
    void (*write_board_setup)();    // 板级初始化代码
    
    // 内部状态（arm_load_kernel 填写）
    const char *kernel_filename;    // 从 MachineState 复制
    const char *kernel_cmdline;
    const char *initrd_filename;
    const char *dtb_filename;
    hwaddr entry;                   // 内核入口地址
    hwaddr initrd_start;            // initrd 加载地址
    hwaddr initrd_size;
    hwaddr dtb_start;               // DTB 加载地址
    int is_linux;                   // 是否为 Linux
};
```

---

## 十、Fixup 机制

Bootloader 桩使用 Fixup 机制在写入 ROM 时填入运行时值：

```c
typedef enum {
    FIXUP_NONE = 0,          // 不修改
    FIXUP_TERMINATOR,        // 结束标记
    FIXUP_BOARDID,           // 填入 board_id (r1)
    FIXUP_BOARD_SETUP,       // 填入 board_setup 地址
    FIXUP_ARGPTR_LO/HI,     // 填入 DTB/ATAGS 地址
    FIXUP_ENTRYPOINT_LO/HI, // 填入内核入口地址
    FIXUP_GIC_CPU_IF,        // 填入 GIC CPU Interface 地址
    FIXUP_BOOTREG,           // 填入 boot 寄存器地址
    FIXUP_DSB,               // 填入正确的 DSB 指令
} FixupType;
```

---

## 十一、启动流程完整时序

```
QEMU 命令行解析
    │
    ├─ -kernel Image -initrd rootfs.cpio -dtb virt.dtb -append "..."
    │
    ▼
machvirt_init()
    │
    ├─ ... (创建 CPU、设备) ...
    │
    └─ arm_load_kernel()
          │
          ├─ 注册 do_cpu_reset() 到 system reset
          │
          ├─ arm_setup_direct_kernel_boot()
          │     │
          │     ├─ 加载内核 (ELF → uImage → Image → raw)
          │     │     └─ 确定 entry point
          │     │
          │     ├─ 加载 initrd
          │     │     └─ 放在 min(RAM/2, 128MB) 之后
          │     │
          │     ├─ 计算 DTB 位置 (initrd后 2MB对齐)
          │     │
          │     ├─ 写入 bootloader桩 到 RAM base
          │     │     └─ 填入 DTB 地址和内核入口
          │     │
          │     └─ 设置 PSCI (conduit + secondary powered-off)
          │
          └─ (machine_done 回调中)
                │
                └─ arm_load_dtb()
                      ├─ 修改 /memory, /chosen
                      ├─ 添加 /psci 节点
                      └─ 写入 guest RAM

系统开始执行
    │
    ├─ do_cpu_reset() 被调用
    │     ├─ cpu_reset()
    │     ├─ arm_emulate_firmware_reset(target_el)
    │     └─ cpu_set_pc(loader_start)  // RAM base
    │
    └─ CPU 开始执行 bootloader桩
          │
          ├─ x0 = DTB 地址
          ├─ x1 = x2 = x3 = 0
          └─ br kernel_entry
                │
                └─ Linux 内核开始执行
```

---

## 十二、与真实硬件启动对比

| 方面 | QEMU 直接启动 | 真实硬件 |
|------|---------------|----------|
| 引导加载器 | 40 字节桩代码 | UEFI/U-Boot (MB级) |
| 内核位置 | QEMU 直接写入 RAM | 固件从存储读取 |
| DTB 来源 | QEMU 生成或用户提供 | 固件/bootloader 提供 |
| CPU 初始状态 | `arm_emulate_firmware_reset()` | 真实固件配置 |
| 多核启动 | PSCI（QEMU模拟）或自旋桩 | PSCI（TF-A实现）|
| 内核解压 | QEMU 在加载时解压 | 内核自解压或 bootloader |
| 安全启动 | 可选 secure_boot 标志 | Secure Boot 签名验证链 |

---

## 十三、源文件索引

| 函数 | 行号 | 职责 |
|------|------|------|
| `arm_load_kernel()` | 1173 | 主入口：注册 reset handler、选择启动模式 |
| `arm_setup_direct_kernel_boot()` | 892 | 直接内核启动：加载 kernel/initrd/DTB |
| `arm_setup_firmware_boot()` | 1115 | 固件启动：通过 fw_cfg 传递 kernel |
| `do_cpu_reset()` | 655 | CPU 复位：设置 EL/PC/endianness |
| `arm_load_elf()` | 757 | ELF 格式加载 |
| `load_aarch64_image()` | 816 | AArch64 raw Image 加载（含 gzip） |
| `arm_write_bootloader()` | 139 | bootloader 桩写入（带 fixup） |
| `default_write_secondary()` | 190 | 非 PSCI SMP 从核自旋代码 |
| `arm_load_dtb()` | (调用) | DTB 加载与修改 |
| `fdt_add_psci_node()` | 366 | PSCI DT 节点生成 |
| `arm_boot_address_space()` | 50 | 确定启动地址空间（S/NS） |
| `arm_emulate_firmware_reset()` | (cpu.c) | 模拟固件 EL3/EL2 设置 |
