# Doc 110: QEMU+GDB 调试 Linux 内核完整工作流

## 文档信息
- **组件**: QEMU ARM64 系统模式, GDB 内核调试, Linux 编译, Rootfs 制作
- **源码版本**: QEMU 11.0.50, Linux 6.x
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [环境准备](#1-环境准备)
2. [编译 Linux 内核 (ARM64)](#2-编译-linux-内核-arm64)
3. [制作 Rootfs](#3-制作-rootfs)
4. [QEMU 启动配置](#4-qemu-启动配置)
5. [GDB 连接与内核调试](#5-gdb-连接与内核调试)
6. [内核模块调试](#6-内核模块调试)
7. [常用调试场景](#7-常用调试场景)
8. [高级技巧](#8-高级技巧)
9. [脚本自动化](#9-脚本自动化)
10. [常见问题排查](#10-常见问题排查)

---

## 1. 环境准备

### 1.1 宿主机工具链

```bash
# Ubuntu/Debian:
sudo apt install -y \
    gcc-aarch64-linux-gnu \
    binutils-aarch64-linux-gnu \
    gdb-multiarch \
    qemu-system-arm \
    debootstrap \
    libncurses-dev \
    flex bison bc \
    libssl-dev libelf-dev \
    cpio

# 或使用自编译 QEMU (推荐, 调试 QEMU 本身也方便):
cd /path/to/qemu
mkdir build && cd build
../configure --target-list=aarch64-softmmu --enable-debug
make -j$(nproc)
```

### 1.2 目录结构建议

```
~/kernel-debug/
├── linux/              # Linux 内核源码
├── rootfs/             # 根文件系统
├── rootfs.img          # rootfs 磁盘镜像
├── scripts/            # 启动/调试脚本
└── modules/            # 自定义内核模块
```

---

## 2. 编译 Linux 内核 (ARM64)

### 2.1 获取源码

```bash
git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

### 2.2 配置 (调试优化)

```bash
# 基础配置:
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig

# 开启调试选项:
./scripts/config --enable CONFIG_DEBUG_INFO           # 调试信息
./scripts/config --enable CONFIG_DEBUG_INFO_DWARF5    # DWARF5 (推荐)
./scripts/config --enable CONFIG_GDB_SCRIPTS          # GDB 辅助脚本
./scripts/config --enable CONFIG_FRAME_POINTER        # 帧指针 (精确回溯)
./scripts/config --disable CONFIG_RANDOMIZE_BASE      # 禁用 KASLR!
./scripts/config --enable CONFIG_KGDB                 # KGDB 支持
./scripts/config --enable CONFIG_KGDB_SERIAL_CONSOLE
./scripts/config --enable CONFIG_MAGIC_SYSRQ          # SysRq
./scripts/config --enable CONFIG_DEBUG_KERNEL
./scripts/config --enable CONFIG_PROVE_LOCKING        # 锁调试
./scripts/config --enable CONFIG_DEBUG_MUTEXES
./scripts/config --enable CONFIG_DEBUG_SPINLOCK
./scripts/config --enable CONFIG_STACKTRACE
./scripts/config --enable CONFIG_DEBUG_PAGEALLOC      # 页面调试
./scripts/config --enable CONFIG_KASAN                # 内存错误检测
./scripts/config --enable CONFIG_UBSAN                # 未定义行为检测

# VirtIO 驱动 (QEMU virt 平台需要):
./scripts/config --enable CONFIG_VIRTIO_BLK
./scripts/config --enable CONFIG_VIRTIO_NET
./scripts/config --enable CONFIG_VIRTIO_CONSOLE
./scripts/config --enable CONFIG_HW_RANDOM_VIRTIO
./scripts/config --enable CONFIG_DRM_VIRTIO_GPU

# 9P 文件系统 (共享宿主目录):
./scripts/config --enable CONFIG_NET_9P
./scripts/config --enable CONFIG_NET_9P_VIRTIO
./scripts/config --enable CONFIG_9P_FS

# 确保优化级别为 -O1 或 -Og (可选, 调试更友好):
# 在 Makefile 中: KBUILD_CFLAGS 添加 -Og 替换 -O2
# 注意: -O0 会导致内核不可启动!

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
```

### 2.3 编译

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# 输出:
# arch/arm64/boot/Image     ← 未压缩内核镜像 (GDB 需要)
# vmlinux                   ← ELF 文件 (GDB 符号源)
```

### 2.4 关键提示: nokaslr

**必须**在内核命令行添加 `nokaslr`, 否则 GDB 符号地址不匹配!

---

## 3. 制作 Rootfs

### 3.1 方式 A: BusyBox 最小 Rootfs (快速)

```bash
# 下载 BusyBox:
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xf busybox-1.36.1.tar.bz2 && cd busybox-1.36.1

# 配置 (静态编译):
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- install

# 制作 initramfs:
cd _install
mkdir -p proc sys dev etc/init.d
cat > init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
echo "Boot successful!"
exec /bin/sh
EOF
chmod +x init
find . | cpio -H newc -o | gzip > ../rootfs.cpio.gz
```

### 3.2 方式 B: Debootstrap 完整 Rootfs

```bash
# 创建 rootfs 目录:
sudo debootstrap --arch=arm64 --foreign bookworm rootfs/

# 创建磁盘镜像:
dd if=/dev/zero of=rootfs.img bs=1M count=2048
mkfs.ext4 rootfs.img
mkdir -p /mnt/rootfs
sudo mount rootfs.img /mnt/rootfs
sudo cp -a rootfs/* /mnt/rootfs/

# 配置 (在 chroot 中):
sudo chroot /mnt/rootfs /debootstrap/debootstrap --second-stage
sudo chroot /mnt/rootfs passwd root  # 设置密码
sudo chroot /mnt/rootfs bash -c 'echo "root:root" | chpasswd'

# 配置 getty:
sudo chroot /mnt/rootfs systemctl enable serial-getty@ttyAMA0.service

# 清理:
sudo umount /mnt/rootfs
```

### 3.3 方式 C: Buildroot (推荐生产用)

```bash
git clone https://github.com/buildroot/buildroot.git
cd buildroot
make qemu_aarch64_virt_defconfig
make -j$(nproc)
# 输出: output/images/rootfs.ext4
```

---

## 4. QEMU 启动配置

### 4.1 使用 initramfs (最简单)

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu max \
    -m 2G \
    -smp 4 \
    -kernel linux/arch/arm64/boot/Image \
    -initrd rootfs.cpio.gz \
    -append "console=ttyAMA0 nokaslr" \
    -nographic \
    -s -S
```

### 4.2 使用磁盘镜像

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu max \
    -m 4G \
    -smp 4 \
    -kernel linux/arch/arm64/boot/Image \
    -drive file=rootfs.img,format=raw,if=virtio \
    -append "root=/dev/vda rw console=ttyAMA0 nokaslr" \
    -nographic \
    -netdev user,id=net0,hostfwd=tcp::2222-:22 \
    -device virtio-net-pci,netdev=net0 \
    -s -S
```

### 4.3 使用 9P 共享目录

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu max \
    -m 2G \
    -kernel linux/arch/arm64/boot/Image \
    -initrd rootfs.cpio.gz \
    -append "console=ttyAMA0 nokaslr" \
    -nographic \
    -virtfs local,path=/path/to/shared,mount_tag=host0,security_model=mapped-xattr \
    -s -S

# Guest 中挂载:
# mount -t 9p -o trans=virtio host0 /mnt
```

### 4.4 关键参数说明

| 参数 | 说明 |
|------|------|
| `-M virt` | ARM64 virt 平台 (GIC + 通用定时器) |
| `-cpu max` | 开启所有 CPU 特性 |
| `-s` | 等价于 `-gdb tcp::1234` |
| `-S` | 启动后暂停, 等待 GDB 连接 |
| `nokaslr` | **必须!** 禁用地址随机化 |
| `-nographic` | 串口输出到终端 |
| `-smp N` | 多核调试 |

---

## 5. GDB 连接与内核调试

### 5.1 启动 GDB

```bash
# 终端 2:
gdb-multiarch linux/vmlinux

# 或使用内核 GDB 脚本:
gdb-multiarch -ex "add-auto-load-safe-path linux/scripts/gdb" linux/vmlinux
```

### 5.2 连接与基本操作

```gdb
(gdb) target remote :1234

# 加载内核 GDB 辅助脚本 (如果内核编译时启用了):
(gdb) source linux/scripts/gdb/vmlinux-gdb.py

# 设置断点:
(gdb) break start_kernel
(gdb) continue

# 命中后:
(gdb) bt                           # 调用栈
(gdb) info threads                 # 所有 CPU
(gdb) print init_task              # 打印 init_task 结构
```

### 5.3 内核辅助命令 (vmlinux-gdb.py)

```gdb
# 进程列表:
(gdb) lx-ps

# 当前 dmesg:
(gdb) lx-dmesg

# 查看 slab 缓存:
(gdb) lx-slabinfo

# 模块列表:
(gdb) lx-lsmod

# 内核符号查找:
(gdb) lx-symbols

# 设备树:
(gdb) lx-device-list-bus platform

# 定时器列表:
(gdb) lx-timerlist

# 查看特定进程:
(gdb) lx-task-by-pid 1
```

### 5.4 源码级调试

```gdb
# 设置源码搜索路径:
(gdb) directory linux/

# 按函数名设断点:
(gdb) break do_page_fault
(gdb) break schedule
(gdb) break __arm64_sys_openat

# 按文件行号:
(gdb) break kernel/sched/core.c:6000

# 条件断点:
(gdb) break do_page_fault if address == 0xdead0000

# 单步进入:
(gdb) step    # 源码级单步 (进入函数)
(gdb) next    # 源码级单步 (不进入)
(gdb) finish  # 执行完当前函数

# 打印变量:
(gdb) print current->comm          # 当前进程名
(gdb) print current->pid           # 当前 PID
(gdb) print *current->mm           # 内存描述符
```

---

## 6. 内核模块调试

### 6.1 编译模块

```c
// modules/hello.c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello, kernel debug!\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye!\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

```makefile
# modules/Makefile
obj-m += hello.o
KDIR := /path/to/linux

all:
	make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
	     -C $(KDIR) M=$(PWD) modules
```

### 6.2 加载模块并调试

```bash
# 将模块传入 Guest (通过 9P 或 scp):
# Guest 中:
insmod /mnt/hello.ko
```

### 6.3 GDB 中加载模块符号

```gdb
# 方法 1: 使用 lx-symbols 自动加载
(gdb) lx-symbols /path/to/modules/
# 自动检测已加载模块并添加符号

# 方法 2: 手动加载
# 先在 Guest 中获取模块加载地址:
# cat /sys/module/hello/sections/.text
# 假设输出: 0xffff800000a00000

(gdb) add-symbol-file modules/hello.ko 0xffff800000a00000

# 现在可以设断点:
(gdb) break hello_init
(gdb) break hello_exit
```

### 6.4 模块调试技巧

```gdb
# 在模块 init 之前设断点:
(gdb) break do_init_module
(gdb) continue
# 命中后, 此时模块已加载但 init 未执行
# 加载符号后再 break 模块函数

# 打印模块内全局变量:
(gdb) print my_module_var

# 模块数据结构:
(gdb) print *((struct module *)0xffff800000a00000)
```

---

## 7. 常用调试场景

### 7.1 追踪内核崩溃 (Panic/Oops)

```gdb
# 在 panic 处设断点:
(gdb) break panic
(gdb) break die
(gdb) continue

# 崩溃后检查:
(gdb) bt                        # 完整调用栈
(gdb) info locals               # 局部变量
(gdb) print/x $elr_el1          # 异常返回地址 (通过 monitor)
(gdb) monitor info registers    # 所有系统寄存器

# 反向追踪 (如果启用了 replay):
(gdb) reverse-continue
```

### 7.2 调试调度器

```gdb
(gdb) break schedule
(gdb) continue

# 查看进程切换:
(gdb) print prev->comm
(gdb) print next->comm
(gdb) print prev->state

# 查看运行队列:
(gdb) print rq->nr_running
(gdb) print rq->cfs.nr_running
```

### 7.3 调试页错误

```gdb
(gdb) break do_page_fault
(gdb) continue

# 检查故障信息:
(gdb) print/x address           # 故障地址
(gdb) print/x esr               # ESR 值
(gdb) print current->comm       # 触发进程
(gdb) print *current->mm->pgd   # 页表根
```

### 7.4 调试中断处理

```gdb
# GIC 中断入口:
(gdb) break gic_handle_irq
(gdb) continue

# 查看中断号:
(gdb) print irqnr
(gdb) monitor info irq          # 全局中断统计
```

### 7.5 调试系统调用

```gdb
# 特定系统调用:
(gdb) break __arm64_sys_openat
(gdb) continue

# 检查参数 (ARM64 调用约定: x0-x7):
(gdb) print/s (char*)$x1        # 文件路径 (第2个参数)
(gdb) print/x $x2               # flags
```

### 7.6 调试锁竞争

```gdb
# 死锁检测:
(gdb) break lockdep_rcu_suspicious_read_check
(gdb) break lock_acquire

# 检查所有 CPU 的调用栈:
(gdb) thread apply all bt

# 查看特定锁:
(gdb) print *lock               # spinlock 状态
(gdb) print lock->owner          # mutex 持有者
```

---

## 8. 高级技巧

### 8.1 物理内存检查

```gdb
# 切换到物理内存模式:
(gdb) monitor qemu.PhyMemMode 1

# 检查页表:
(gdb) x/gx 0x40000000          # 物理地址直接访问
(gdb) x/8gx 0x40001000         # PGD/PUD/PMD/PTE

# 恢复虚拟模式:
(gdb) monitor qemu.PhyMemMode 0
```

### 8.2 设备状态检查

```gdb
# 通过 monitor 检查设备:
(gdb) monitor info qtree       # 设备树
(gdb) monitor info mtree       # 内存映射
(gdb) monitor info irq         # 中断
(gdb) monitor info pci         # PCI 设备

# GIC 状态:
(gdb) monitor info pic         # (如果支持)
```

### 8.3 使用 Watchpoint 追踪数据修改

```gdb
# 监视全局变量:
(gdb) watch jiffies            # jiffies 变化时停止
(gdb) watch *(int*)0xffff800080300000  # 指定地址

# 限制条件:
(gdb) watch current->flags if current->pid == 100
```

### 8.4 打印内核数据结构

```gdb
# 使用 pretty-printer (需要 vmlinux-gdb.py):
(gdb) print *(struct task_struct *)$x0
(gdb) print *(struct mm_struct *)current->mm
(gdb) print *(struct vm_area_struct *)vma

# 遍历链表:
(gdb) lx-list-check init_task.tasks
```

### 8.5 多核调试

```gdb
# 查看所有 CPU:
(gdb) info threads
(gdb) thread apply all bt      # 所有 CPU 调用栈

# 切换到指定 CPU:
(gdb) thread 3                 # 切到 CPU#2
(gdb) bt                       # 该 CPU 的栈

# 只在特定 CPU 设断点:
(gdb) break schedule thread 1  # 只在 CPU#0
```

### 8.6 Early Boot 调试

```gdb
# QEMU 启动时暂停 (-S), GDB 连接后:
(gdb) break *0x40000000        # ARM64 默认入口
(gdb) c

# 或从内核解压后:
(gdb) break primary_entry      # 内核入口
(gdb) break __primary_switched # MMU 开启后
(gdb) break start_kernel       # C 代码入口
```

---

## 9. 脚本自动化

### 9.1 启动脚本 (run-qemu.sh)

```bash
#!/bin/bash
KERNEL=linux/arch/arm64/boot/Image
ROOTFS=rootfs.cpio.gz
QEMU=qemu-system-aarch64

$QEMU \
    -M virt,gic-version=3 \
    -cpu max \
    -m 4G \
    -smp 4 \
    -kernel $KERNEL \
    -initrd $ROOTFS \
    -append "console=ttyAMA0 nokaslr earlyprintk loglevel=8" \
    -nographic \
    -s -S \
    "$@"
```

### 9.2 GDB 初始化脚本 (.gdbinit)

```gdb
# ~/.gdbinit 或项目级 .gdbinit
set architecture aarch64
set confirm off
set pagination off

# 连接:
target remote :1234

# 加载内核辅助:
add-auto-load-safe-path /path/to/linux/scripts/gdb
source /path/to/linux/scripts/gdb/vmlinux-gdb.py

# 常用断点:
break start_kernel
break panic
break die

# 自定义命令:
define lk
    info threads
    thread apply all bt 5
end
document lk
    显示所有 CPU 的简短调用栈
end

# 继续到 start_kernel:
continue
```

### 9.3 调试脚本 (debug.sh)

```bash
#!/bin/bash
# 一键启动调试环境

# 终端 1: 启动 QEMU
tmux new-session -d -s debug './run-qemu.sh'

# 等待 QEMU 启动:
sleep 1

# 终端 2: 启动 GDB
tmux split-window -h 'gdb-multiarch -x .gdbinit linux/vmlinux'

tmux attach -t debug
```

---

## 10. 常见问题排查

### 10.1 GDB 符号不匹配

**症状**: `(gdb) bt` 显示 `<unknown>` 或地址错误

**原因**: KASLR 启用, 内核加载到随机地址

**解决**:
```
内核命令行添加: nokaslr
```

### 10.2 断点不命中

**可能原因**:
1. 函数被内联 → 使用 `break file.c:line` 替代
2. 符号被优化掉 → 降低优化级别或使用 `hbreak`
3. 地址不正确 → 检查 `info symbol <addr>`

### 10.3 单步进入汇编

**症状**: `step` 跳到汇编代码

**解决**: 使用 `next` (不进入) 或 `finish` (跳出)

### 10.4 "Cannot access memory"

**可能原因**:
1. 目标地址未映射 → 使用物理内存模式
2. VM 未暂停 → 确保在停止状态操作

### 10.5 连接后无响应

**检查**:
```bash
# 确认 QEMU 在监听:
ss -tlnp | grep 1234

# 确认 GDB 架构正确:
(gdb) show architecture  # 应为 aarch64
```

### 10.6 模块符号加载失败

**步骤**:
```gdb
# 确认模块已在 Guest 中加载:
(gdb) lx-lsmod

# 手动获取 .text 段地址:
# Guest: cat /sys/module/XXX/sections/.text

# 加载:
(gdb) add-symbol-file path/to/module.ko 0x<text_addr>
```

### 10.7 -Og 编译问题

某些内核版本/架构下 `-Og` 不能正常启动:
- 降级使用默认 `-O2` (大多数变量仍可观察)
- 关键函数使用 `__attribute__((optimize("O0")))` 单独降优化

---

## 附录: 推荐阅读顺序

| 步骤 | 对应文档 |
|------|----------|
| 1. 理解 GDB Stub 架构 | debug/00 |
| 2. RSP 命令与用法 | practice/106 |
| 3. 源码级执行路径 | debug/109 |
| 4. ARM64 调试寄存器 | debug/53 |
| 5. TCG 执行循环 | arm64/tcg/44 |
| 6. 断点实现细节 | debug/18 |

---

*文档结束*
