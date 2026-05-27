# Doc 112: QEMU 开发者指南 — 添加设备与指令

## 文档信息
- **组件**: QOM 设备模型, TCG 指令翻译, 构建系统, 测试框架
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [开发环境搭建](#1-开发环境搭建)
2. [QOM 设备模型概览](#2-qom-设备模型概览)
3. [添加 SysBus MMIO 设备 (完整示例)](#3-添加-sysbus-mmio-设备-完整示例)
4. [添加 PCI 设备 (完整示例)](#4-添加-pci-设备-完整示例)
5. [设备属性与配置](#5-设备属性与配置)
6. [中断与 DMA](#6-中断与-dma)
7. [添加 ARM64 指令 (完整示例)](#7-添加-arm64-指令-完整示例)
8. [TCG Helper 函数](#8-tcg-helper-函数)
9. [构建系统集成](#9-构建系统集成)
10. [Trace Events 添加](#10-trace-events-添加)
11. [测试编写](#11-测试编写)
12. [提交规范与流程](#12-提交规范与流程)

---

## 1. 开发环境搭建

### 1.1 编译 QEMU (调试模式)

```bash
git clone https://gitlab.com/qemu-project/qemu.git
cd qemu
mkdir build && cd build
../configure \
    --target-list=aarch64-softmmu,aarch64-linux-user \
    --enable-debug \
    --enable-debug-tcg \
    --enable-trace-backends=simple,ftrace \
    --enable-sanitizers
make -j$(nproc)
```

### 1.2 开发工具建议

```bash
# compile_commands.json (IDE 支持):
cd build
ninja -t compdb > compile_commands.json
ln -sf build/compile_commands.json ../

# 代码检查:
scripts/checkpatch.pl --no-tree -f hw/misc/mydevice.c

# 运行单个测试:
make check-qtest-aarch64
```

### 1.3 目录结构认知

```
hw/              ← 所有设备模型
├── misc/        ← 杂项设备 (最适合新设备起步)
├── arm/         ← ARM 平台板级
├── pci/         ← PCI 总线基础
├── virtio/      ← VirtIO 设备
target/
├── arm/
│   ├── tcg/     ← TCG 翻译器 (指令添加在此)
│   │   ├── translate-a64.c    ← AArch64 翻译
│   │   ├── a64.decode         ← 指令解码描述
│   │   └── helper-a64.c      ← Helper 实现
│   └── helper.c              ← 通用 Helper
include/
├── hw/          ← 设备头文件
└── qom/         ← QOM 框架 API
tests/
├── qtest/       ← 设备测试
└── tcg/         ← 指令测试
```

---

## 2. QOM 设备模型概览

### 2.1 核心概念

```
Object (基类)
└── DeviceState
    ├── SysBusDevice     ← 平台设备 (MMIO + IRQ)
    └── PCIDevice        ← PCI 设备
        └── VirtIOPCIProxy
```

### 2.2 设备生命周期

```
TypeInfo 注册 → type_register_static()
     ↓
class_init()     ← 设置类方法 (realize, reset, vmsd)
     ↓
instance_init()  ← 对象创建时 (早期初始化)
     ↓
realize()        ← 设备激活 (注册 MMIO, IRQ, 分配资源)
     ↓
reset()          ← 复位 (Guest 重启时)
     ↓
unrealize()      ← 热拔出时
```

### 2.3 关键 API

| API | 用途 |
|-----|------|
| `type_register_static()` | 注册设备类型 |
| `memory_region_init_io()` | 创建 MMIO 区域 |
| `sysbus_init_mmio()` | 注册到 SysBus |
| `sysbus_init_irq()` | 注册中断输出 |
| `qdev_init_gpio_out()` | GPIO 输出 |
| `pci_register_bar()` | PCI BAR 注册 |
| `device_class_set_props()` | 注册属性 |
| `qemu_set_irq()` | 触发中断 |

---

## 3. 添加 SysBus MMIO 设备 (完整示例)

### 3.1 设备描述

创建一个简单的"计数器"设备:
- 4 个寄存器: CTRL, COUNT, THRESHOLD, STATUS
- 当 COUNT >= THRESHOLD 时触发中断
- 通过 MMIO 访问

### 3.2 头文件: `include/hw/misc/my-counter.h`

```c
#ifndef HW_MY_COUNTER_H
#define HW_MY_COUNTER_H

#include "hw/sysbus.h"
#include "qom/object.h"

#define TYPE_MY_COUNTER "my-counter"
OBJECT_DECLARE_SIMPLE_TYPE(MyCounterState, MY_COUNTER)

struct MyCounterState {
    /* 父类 (必须第一个字段) */
    SysBusDevice parent_obj;

    /* 设备状态 */
    MemoryRegion mmio;
    qemu_irq irq;

    /* 寄存器 */
    uint32_t ctrl;       /* 0x00: 控制 (bit0=enable) */
    uint32_t count;      /* 0x04: 当前计数 */
    uint32_t threshold;  /* 0x08: 阈值 */
    uint32_t status;     /* 0x0C: 状态 (bit0=overflow) */

    /* 属性 */
    uint32_t freq_hz;
};

#endif
```

### 3.3 实现: `hw/misc/my-counter.c`

```c
#include "qemu/osdep.h"
#include "hw/misc/my-counter.h"
#include "hw/irq.h"
#include "hw/qdev-properties.h"
#include "migration/vmstate.h"
#include "qemu/log.h"
#include "qemu/module.h"
#include "trace.h"

/* 寄存器偏移 */
#define REG_CTRL       0x00
#define REG_COUNT      0x04
#define REG_THRESHOLD  0x08
#define REG_STATUS     0x0C

/* CTRL 位定义 */
#define CTRL_ENABLE    BIT(0)
#define CTRL_IRQ_EN    BIT(1)

/* STATUS 位定义 */
#define STATUS_OVERFLOW BIT(0)

static void my_counter_update_irq(MyCounterState *s)
{
    bool level = (s->status & STATUS_OVERFLOW) && (s->ctrl & CTRL_IRQ_EN);
    qemu_set_irq(s->irq, level);
}

static void my_counter_check_threshold(MyCounterState *s)
{
    if (s->count >= s->threshold && s->threshold > 0) {
        s->status |= STATUS_OVERFLOW;
        my_counter_update_irq(s);
    }
}

/* MMIO 读 */
static uint64_t my_counter_read(void *opaque, hwaddr addr, unsigned size)
{
    MyCounterState *s = MY_COUNTER(opaque);
    uint64_t val = 0;

    switch (addr) {
    case REG_CTRL:
        val = s->ctrl;
        break;
    case REG_COUNT:
        val = s->count;
        break;
    case REG_THRESHOLD:
        val = s->threshold;
        break;
    case REG_STATUS:
        val = s->status;
        break;
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "my-counter: bad read offset 0x%" HWADDR_PRIx "\n",
                      addr);
    }
    return val;
}

/* MMIO 写 */
static void my_counter_write(void *opaque, hwaddr addr, uint64_t val,
                             unsigned size)
{
    MyCounterState *s = MY_COUNTER(opaque);

    switch (addr) {
    case REG_CTRL:
        s->ctrl = val & 0x3;
        break;
    case REG_COUNT:
        s->count = val;
        my_counter_check_threshold(s);
        break;
    case REG_THRESHOLD:
        s->threshold = val;
        my_counter_check_threshold(s);
        break;
    case REG_STATUS:
        /* 写 1 清除 */
        s->status &= ~val;
        my_counter_update_irq(s);
        break;
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "my-counter: bad write offset 0x%" HWADDR_PRIx "\n",
                      addr);
    }
}

static const MemoryRegionOps my_counter_ops = {
    .read = my_counter_read,
    .write = my_counter_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl.min_access_size = 4,
    .impl.max_access_size = 4,
};

/* 设备属性 */
static Property my_counter_properties[] = {
    DEFINE_PROP_UINT32("freq-hz", MyCounterState, freq_hz, 1000000),
    DEFINE_PROP_END_OF_LIST(),
};

/* VMState (迁移/快照支持) */
static const VMStateDescription vmstate_my_counter = {
    .name = "my-counter",
    .version_id = 1,
    .minimum_version_id = 1,
    .fields = (const VMStateField[]) {
        VMSTATE_UINT32(ctrl, MyCounterState),
        VMSTATE_UINT32(count, MyCounterState),
        VMSTATE_UINT32(threshold, MyCounterState),
        VMSTATE_UINT32(status, MyCounterState),
        VMSTATE_END_OF_LIST()
    }
};

/* realize: 资源分配 */
static void my_counter_realize(DeviceState *dev, Error **errp)
{
    /* 复杂初始化放这里 (如定时器、DMA) */
}

/* instance_init: 对象创建 */
static void my_counter_init(Object *obj)
{
    MyCounterState *s = MY_COUNTER(obj);
    SysBusDevice *sbd = SYS_BUS_DEVICE(obj);

    /* 初始化 MMIO 区域 */
    memory_region_init_io(&s->mmio, obj, &my_counter_ops, s,
                          "my-counter", 0x1000);
    sysbus_init_mmio(sbd, &s->mmio);

    /* 初始化 IRQ 输出 */
    sysbus_init_irq(sbd, &s->irq);
}

/* reset */
static void my_counter_reset(DeviceState *dev)
{
    MyCounterState *s = MY_COUNTER(dev);
    s->ctrl = 0;
    s->count = 0;
    s->threshold = 0;
    s->status = 0;
    qemu_set_irq(s->irq, 0);
}

/* class_init: 类初始化 */
static void my_counter_class_init(ObjectClass *klass, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->realize = my_counter_realize;
    dc->reset = my_counter_reset;
    dc->vmsd = &vmstate_my_counter;
    device_class_set_props(dc, my_counter_properties);
    set_bit(DEVICE_CATEGORY_MISC, dc->categories);
}

/* 类型注册 */
static const TypeInfo my_counter_info = {
    .name          = TYPE_MY_COUNTER,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(MyCounterState),
    .instance_init = my_counter_init,
    .class_init    = my_counter_class_init,
};

static void my_counter_register_types(void)
{
    type_register_static(&my_counter_info);
}

type_init(my_counter_register_types)
```

### 3.4 使用设备

```bash
# 命令行:
qemu-system-aarch64 -M virt -cpu max -m 1G \
    -device my-counter,freq-hz=2000000

# 在设备树中 (virt 板级代码添加):
# 或者通过 -device 动态添加
```

---

## 4. 添加 PCI 设备 (完整示例)

### 4.1 PCI 设备框架 (参考 hw/misc/edu.c)

```c
#include "qemu/osdep.h"
#include "hw/pci/pci_device.h"
#include "hw/pci/msi.h"
#include "qom/object.h"

#define TYPE_MY_PCI "my-pci-dev"
OBJECT_DECLARE_SIMPLE_TYPE(MyPCIState, MY_PCI)

struct MyPCIState {
    PCIDevice parent_obj;
    MemoryRegion mmio;
    /* 设备寄存器 */
    uint32_t reg0;
};

static uint64_t my_pci_read(void *opaque, hwaddr addr, unsigned size)
{
    MyPCIState *s = opaque;
    switch (addr) {
    case 0x00: return s->reg0;
    default: return 0;
    }
}

static void my_pci_write(void *opaque, hwaddr addr, uint64_t val,
                         unsigned size)
{
    MyPCIState *s = opaque;
    switch (addr) {
    case 0x00:
        s->reg0 = val;
        /* 触发 MSI */
        if (msi_enabled(&s->parent_obj)) {
            msi_notify(&s->parent_obj, 0);
        }
        break;
    }
}

static const MemoryRegionOps my_pci_ops = {
    .read = my_pci_read,
    .write = my_pci_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
};

static void my_pci_realize(PCIDevice *pdev, Error **errp)
{
    MyPCIState *s = MY_PCI(pdev);

    /* 初始化 MMIO */
    memory_region_init_io(&s->mmio, OBJECT(s), &my_pci_ops, s,
                          "my-pci-mmio", 4096);
    pci_register_bar(pdev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &s->mmio);

    /* 启用 MSI */
    msi_init(pdev, 0, 1, true, false, errp);
}

static void my_pci_class_init(ObjectClass *klass, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    PCIDeviceClass *k = PCI_DEVICE_CLASS(klass);

    k->realize = my_pci_realize;
    k->vendor_id = PCI_VENDOR_ID_QEMU;  /* 0x1234 */
    k->device_id = 0x9999;
    k->revision = 0x01;
    k->class_id = PCI_CLASS_OTHERS;

    set_bit(DEVICE_CATEGORY_MISC, dc->categories);
}

static const TypeInfo my_pci_info = {
    .name          = TYPE_MY_PCI,
    .parent        = TYPE_PCI_DEVICE,
    .instance_size = sizeof(MyPCIState),
    .class_init    = my_pci_class_init,
    .interfaces    = (InterfaceInfo[]) {
        { INTERFACE_PCIE_DEVICE },
        { }
    },
};

static void my_pci_register(void) { type_register_static(&my_pci_info); }
type_init(my_pci_register)
```

### 4.2 PCI vs SysBus 对比

| 特性 | SysBus | PCI |
|------|--------|-----|
| 地址分配 | 固定 (设备树/板级) | 动态 (BAR 枚举) |
| 中断 | 直接连线 | INTx / MSI / MSI-X |
| DMA | 需手动设置 | PCI DMA API |
| 热插拔 | 一般不支持 | 支持 |
| 适用场景 | SoC 内嵌外设 | 标准外设卡 |

---

## 5. 设备属性与配置

### 5.1 属性类型

```c
static Property my_properties[] = {
    /* 整数 */
    DEFINE_PROP_UINT32("freq", MyState, freq, 1000000),
    DEFINE_PROP_UINT64("size", MyState, size, 4096),

    /* 布尔 */
    DEFINE_PROP_BOOL("dma-enabled", MyState, dma_enabled, true),

    /* 字符串 */
    DEFINE_PROP_STRING("name", MyState, name),

    /* 链接 (到其他设备) */
    DEFINE_PROP_LINK("chardev", MyState, chr, TYPE_CHARDEV, Chardev *),

    DEFINE_PROP_END_OF_LIST(),
};
```

### 5.2 命令行使用

```bash
-device my-counter,freq-hz=2000000
-device my-pci-dev,addr=03.0
-device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:56
```

---

## 6. 中断与 DMA

### 6.1 SysBus 中断

```c
/* 设备侧: 声明 + 触发 */
sysbus_init_irq(sbd, &s->irq);          // instance_init
qemu_set_irq(s->irq, 1);                // 触发 (边沿/电平)
qemu_set_irq(s->irq, 0);                // 清除

/* 板级侧: 连线到 GIC */
sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0,
                   qdev_get_gpio_in(gic, spi_num));
```

### 6.2 PCI MSI/MSI-X

```c
/* realize 中初始化: */
msi_init(pdev, 0, 1, true, false, errp);        // MSI (1 vector)
msix_init(pdev, 4, &s->mmio, 0, 0x2000,         // MSI-X (4 vectors)
          &s->mmio, 0, 0x3000, 0, errp);

/* 触发中断: */
msi_notify(pdev, vector_num);
msix_notify(pdev, vector_num);
```

### 6.3 DMA 操作

```c
/* PCI DMA 读写: */
pci_dma_read(pdev, guest_addr, buf, len);
pci_dma_write(pdev, guest_addr, buf, len);

/* 通用 DMA (SysBus): */
dma_memory_read(&address_space_memory, addr, buf, len, MEMTXATTRS_UNSPECIFIED);
dma_memory_write(&address_space_memory, addr, buf, len, MEMTXATTRS_UNSPECIFIED);
```

---

## 7. 添加 ARM64 指令 (完整示例)

### 7.1 指令翻译流程

```
a64.decode (声明格式)
     ↓  decodetree.py 生成
decode-a64.c.inc (pattern matching)
     ↓  调用
trans_XXX() 函数 (translate-a64.c)
     ↓  生成
TCG ops (tcg_gen_*)
     ↓  TCG 后端
Host 机器码
```

### 7.2 Decode 文件格式

**文件**: `target/arm/tcg/a64.decode`

```
# 格式: <指令名> <位模式> [参数提取]
# '.' = don't care, 数字 = 固定位, 字母 = 字段

# 现有示例 (MOV 系列):
MOVN      sf:1 00 100101 hw:2 imm:16 rd:5
MOVZ      sf:1 10 100101 hw:2 imm:16 rd:5
MOVK      sf:1 11 100101 hw:2 imm:16 rd:5
```

### 7.3 添加假想指令 "MYINST"

假设我们要添加一个自定义指令: `MYINST Rd, Rn, #imm` (将 Rn + imm*4 存入 Rd)

**步骤 1: 在 a64.decode 添加解码条目**

```
# 选择一个未使用的编码空间 (示例用)
MYINST    1 00 11111 00 imm:12 rn:5 rd:5
```

**步骤 2: 在 translate-a64.c 添加翻译函数**

```c
static bool trans_MYINST(DisasContext *s, arg_MYINST *a)
{
    TCGv_i64 tcg_rn, tcg_rd;

    /* 检查 CPU 特性 (如果需要) */
    if (!dc_isar_feature(aa64_myext, s)) {
        return false;  /* 未识别指令 → UNDEF */
    }

    tcg_rn = cpu_reg(s, a->rn);
    tcg_rd = cpu_reg(s, a->rd);

    /* 生成 TCG 操作: rd = rn + imm * 4 */
    tcg_gen_addi_i64(tcg_rd, tcg_rn, (int64_t)a->imm << 2);

    return true;
}
```

**步骤 3: 如果需要 Helper (复杂操作)**

```c
// target/arm/tcg/helper-a64.h (声明):
DEF_HELPER_FLAGS_2(myinst_complex, TCG_CALL_NO_RWG, i64, i64, i32)

// target/arm/tcg/helper-a64.c (实现):
uint64_t HELPER(myinst_complex)(uint64_t rn, uint32_t imm)
{
    /* 复杂计算 (TCG 无法直接表达的) */
    return some_complex_operation(rn, imm);
}

// translate-a64.c 中调用:
static bool trans_MYINST(DisasContext *s, arg_MYINST *a)
{
    TCGv_i64 result = tcg_temp_new_i64();
    gen_helper_myinst_complex(result, cpu_reg(s, a->rn),
                              tcg_constant_i32(a->imm));
    tcg_gen_mov_i64(cpu_reg(s, a->rd), result);
    return true;
}
```

### 7.4 现有指令参考: MOVN 翻译

```c
// target/arm/tcg/translate-a64.c:
static bool trans_MOVN(DisasContext *s, arg_movw *a)
{
    int pos = a->hw << 4;
    uint64_t imm = ~((uint64_t)a->imm << pos);
    if (!a->sf) {
        imm = (uint32_t)imm;
    }
    tcg_gen_movi_i64(cpu_reg(s, a->rd), imm);
    return true;
}
```

### 7.5 指令添加检查清单

- [ ] `a64.decode` 添加编码模式
- [ ] `translate-a64.c` 添加 `trans_XXX()`
- [ ] 如需 Helper: `helper-a64.h` 声明 + `helper-a64.c` 实现
- [ ] 如需 CPU Feature: `target/arm/cpu-features.h` 添加 isar 检查
- [ ] `tests/tcg/aarch64/` 添加测试
- [ ] 运行 `make check-tcg` 验证

---

## 8. TCG Helper 函数

### 8.1 何时需要 Helper

| 场景 | 用 TCG ops | 用 Helper |
|------|-----------|-----------|
| 简单算术 | ✅ `tcg_gen_add/sub/and/or` | ❌ |
| 位操作 | ✅ `tcg_gen_shl/shr/rotl` | ❌ |
| 浮点运算 | ❌ | ✅ `helper_vfp_*` |
| 系统寄存器访问 | ❌ | ✅ `helper_msr/mrs` |
| 异常触发 | ❌ | ✅ `helper_exception` |
| 内存屏障 | ❌ | ✅ `helper_dmb/dsb` |
| 复杂加密 | ❌ | ✅ `helper_crypto_*` |

### 8.2 Helper 声明语法

```c
// helper.h 中:
DEF_HELPER_FLAGS_N(name, flags, ret_type, arg1_type, ..., argN_type)

// flags:
//   TCG_CALL_NO_RWG     — 不读/写全局状态
//   TCG_CALL_NO_WG      — 不写全局状态
//   TCG_CALL_NO_SE      — 无副作用
//   TCG_CALL_PURE       — 纯函数 (NO_RWG + NO_SE)
//   0                   — 可能有任何副作用

// 类型: i32, i64, ptr, env, void
```

### 8.3 调用 Helper

```c
// 在 trans_* 中:
gen_helper_myop(ret, cpu_env, arg1, arg2);

// 注意: env 参数需要显式传递 (如果声明中有 env)
```

---

## 9. 构建系统集成

### 9.1 添加设备到 Meson

**`hw/misc/meson.build`** 添加:
```meson
system_ss.add(when: 'CONFIG_MY_COUNTER', if_true: files('my-counter.c'))
```

### 9.2 添加 Kconfig 条目

**`hw/misc/Kconfig`** 添加:
```
config MY_COUNTER
    bool
    default y
    depends on ARM
```

### 9.3 平台选择 (可选)

**`hw/arm/Kconfig`** (如果只在 virt 上用):
```
config ARM_VIRT
    ...
    select MY_COUNTER
```

### 9.4 指令/Target 代码

指令代码在 `target/arm/tcg/meson.build`:
```meson
arm_ss.add(files(
    'translate-a64.c',
    'helper-a64.c',
    ...
))
```

Decode 文件自动处理:
```meson
decodetree = generator(
    find_program(meson.project_source_root() / 'scripts/decodetree.py'),
    output: 'decode-@BASENAME@.c.inc',
    arguments: ['@INPUT@', '@EXTRA_ARGS@', '-o', '@OUTPUT@']
)
# a64.decode 自动生成 decode-a64.c.inc
```

---

## 10. Trace Events 添加

### 10.1 添加 trace-events 文件

在设备目录 (`hw/misc/trace-events`) 添加:
```
# my-counter events
my_counter_read(uint64_t addr, uint64_t val) "addr=0x%" PRIx64 " val=0x%" PRIx64
my_counter_write(uint64_t addr, uint64_t val) "addr=0x%" PRIx64 " val=0x%" PRIx64
my_counter_irq(int level) "level=%d"
```

### 10.2 在代码中使用

```c
#include "trace.h"

static uint64_t my_counter_read(void *opaque, hwaddr addr, unsigned size)
{
    ...
    trace_my_counter_read(addr, val);
    return val;
}
```

### 10.3 运行时启用

```bash
qemu-system-aarch64 ... --trace "my_counter_*"
```

---

## 11. 测试编写

### 11.1 QTest (设备测试)

**`tests/qtest/my-counter-test.c`**:

```c
#include "libqtest.h"

#define COUNTER_BASE 0x09000000  /* virt 平台分配 */

static void test_basic_read_write(void)
{
    QTestState *qts;

    qts = qtest_init("-M virt -device my-counter");

    /* 写 threshold */
    qtest_writel(qts, COUNTER_BASE + 0x08, 100);
    /* 读回验证 */
    g_assert_cmpuint(qtest_readl(qts, COUNTER_BASE + 0x08), ==, 100);

    /* 写 count, 触发 overflow */
    qtest_writel(qts, COUNTER_BASE + 0x04, 200);
    /* 检查 status */
    g_assert_cmpuint(qtest_readl(qts, COUNTER_BASE + 0x0C) & 1, ==, 1);

    qtest_quit(qts);
}

int main(int argc, char **argv)
{
    g_test_init(&argc, &argv, NULL);
    qtest_add_func("/my-counter/basic", test_basic_read_write);
    return g_test_run();
}
```

**`tests/qtest/meson.build`** 添加:
```meson
qtests_aarch64 = {
    'my-counter-test': [],
}
```

### 11.2 TCG 指令测试

**`tests/tcg/aarch64/test-myinst.c`**:

```c
#include <stdio.h>
#include <assert.h>

int main(void)
{
    uint64_t result;
    uint64_t input = 10;

    /* 内联汇编测试 MYINST */
    asm volatile(
        ".inst 0x..." /* MYINST x0, x1, #5 的编码 */
        : "=r"(result)
        : "r"(input)
    );

    /* 验证: result = input + 5*4 = 30 */
    assert(result == 30);
    printf("MYINST test passed!\n");
    return 0;
}
```

### 11.3 运行测试

```bash
# 设备测试:
make check-qtest-aarch64

# 指令测试:
make check-tcg

# 单个测试:
./build/tests/qtest/my-counter-test
```

---

## 12. 提交规范与流程

### 12.1 Commit Message 格式

```
hw/misc: Add my-counter device

Add a simple counter device with threshold-based interrupt
support for testing and educational purposes.

Features:
- 4 MMIO registers (CTRL/COUNT/THRESHOLD/STATUS)
- Configurable frequency property
- Level-triggered interrupt on threshold overflow
- VMState migration support

Signed-off-by: Your Name <your@email.com>
```

### 12.2 Patch 系列建议拆分

```
1/4  hw/misc: Add my-counter device header
2/4  hw/misc: Implement my-counter device
3/4  hw/arm/virt: Wire my-counter to virt platform
4/4  tests/qtest: Add my-counter test
```

### 12.3 检查清单

- [ ] `scripts/checkpatch.pl` 无警告
- [ ] `make -j$(nproc)` 编译通过
- [ ] `make check` 测试通过
- [ ] VMState 定义完整 (支持迁移)
- [ ] Trace events 添加
- [ ] 文档更新 (`docs/system/`)
- [ ] 使用 `qemu_log_mask(LOG_GUEST_ERROR, ...)` 报告错误

### 12.4 代码风格要点

```c
/* QEMU 编码风格: */
- 4 空格缩进 (不用 Tab)
- 函数名: 小写 + 下划线 (my_device_read)
- 类型名: 驼峰 (MyDeviceState)
- 宏: 全大写 (TYPE_MY_DEVICE)
- 大括号: K&R 风格 (函数换行, if/for 同行)
- 行宽: 80 字符
```

---

## 附录: 快速参考

### A. 最小设备模板

```c
/* 5 步创建最小设备: */
// 1. struct MyState { SysBusDevice parent; MemoryRegion mmio; };
// 2. static const MemoryRegionOps ops = { .read=..., .write=... };
// 3. instance_init: memory_region_init_io + sysbus_init_mmio
// 4. class_init: set realize/reset/vmsd/props
// 5. TypeInfo + type_register_static
```

### B. 文件创建清单

| 新设备 | 新指令 |
|--------|--------|
| `include/hw/xxx/my-dev.h` | `target/arm/tcg/a64.decode` (修改) |
| `hw/xxx/my-dev.c` | `target/arm/tcg/translate-a64.c` (修改) |
| `hw/xxx/meson.build` (修改) | `target/arm/tcg/helper-a64.h` (可选) |
| `hw/xxx/Kconfig` (修改) | `target/arm/tcg/helper-a64.c` (可选) |
| `hw/xxx/trace-events` (修改) | `tests/tcg/aarch64/test-xxx.c` |
| `tests/qtest/my-dev-test.c` | `target/arm/cpu-features.h` (可选) |

---

*文档结束*
