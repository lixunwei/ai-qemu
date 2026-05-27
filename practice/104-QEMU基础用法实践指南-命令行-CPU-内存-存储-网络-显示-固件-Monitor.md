# Doc 104: QEMU 基础用法实践指南

## 文档信息
- **组件**: 命令行, 基础配置, 常用操作
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [安装与编译](#1-安装与编译)
2. [基本启动](#2-基本启动)
3. [CPU 配置](#3-cpu-配置)
4. [内存配置](#4-内存配置)
5. [存储配置](#5-存储配置)
6. [网络配置](#6-网络配置)
7. [显示配置](#7-显示配置)
8. [固件与引导](#8-固件与引导)
9. [Monitor 基础](#9-monitor-基础)
10. [常用场景](#10-常用场景)

---

## 1. 安装与编译

### 1.1 编译安装

```bash
# 依赖 (Ubuntu/Debian):
sudo apt install build-essential ninja-build python3-venv \
    libglib2.0-dev libpixman-1-dev libslirp-dev \
    libgtk-3-dev libspice-server-dev libusb-1.0-0-dev

# 配置 (ARM64 系统模式):
mkdir build && cd build
../configure --target-list=aarch64-softmmu \
    --enable-kvm --enable-vnc --enable-gtk \
    --enable-spice --enable-slirp

# 编译:
make -j$(nproc)

# 安装:
sudo make install
```

### 1.2 最小编译

```bash
# 仅 ARM64 TCG 模拟:
../configure --target-list=aarch64-softmmu --disable-kvm
make -j$(nproc)

# ARM64 用户模式:
../configure --target-list=aarch64-linux-user
make -j$(nproc)
```

---

## 2. 基本启动

### 2.1 最小命令

```bash
# 启动 ARM64 virt 机器:
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a72 \
    -m 2G \
    -kernel Image \
    -append "console=ttyAMA0" \
    -nographic
```

### 2.2 命令行结构

```
qemu-system-aarch64 [选项]

标准选项:
  -h, -help              帮助
  -version               版本
  -machine TYPE          机器类型 (virt)
  -cpu MODEL             CPU 型号
  -smp N                 CPU 数量
  -m SIZE                内存大小

加速:
  -accel kvm|tcg         加速器
  -enable-kvm            KVM 快捷方式

存储:
  -drive ...             块设备
  -hda/-hdb/-hdc/-hdd    IDE 硬盘
  -cdrom FILE            光驱

网络:
  -nic ...               网络设备
  -netdev ...            网络后端

显示:
  -display TYPE          显示方式
  -vnc DISPLAY           VNC 服务
  -nographic             无图形 (串口输出)

引导:
  -kernel FILE           内核镜像
  -initrd FILE           初始 ramdisk
  -append STRING         内核参数
  -bios FILE             BIOS/UEFI 固件
  -pflash FILE           Flash 存储

调试:
  -s                     GDB 服务 (tcp::1234)
  -S                     启动时暂停
  -d ITEMS               调试日志
  -D FILE                日志文件

监控:
  -monitor DEV           HMP monitor
  -qmp DEV               QMP monitor
```

---

## 3. CPU 配置

### 3.1 CPU 型号

```bash
# 查看支持的 CPU:
qemu-system-aarch64 -cpu help

# 常用 CPU:
-cpu cortex-a53         # ARMv8.0-A 低功耗
-cpu cortex-a57         # ARMv8.0-A 高性能
-cpu cortex-a72         # ARMv8.0-A 高性能
-cpu cortex-a76         # ARMv8.2-A
-cpu neoverse-n1        # ARMv8.2-A 服务器
-cpu neoverse-v1        # ARMv8.4-A 高性能
-cpu max                # 启用所有支持的特性
-cpu host               # KVM: 透传宿主 CPU (需 -accel kvm)
```

### 3.2 多核配置

```bash
# 简单多核:
-smp 4                  # 4 核

# 详细拓扑:
-smp 8,sockets=2,clusters=1,cores=2,threads=2

# 可热插 CPU:
-smp 2,maxcpus=8        # 启动 2 核, 最多支持 8 核
```

### 3.3 CPU 特性开关

```bash
# 启用/禁用特性:
-cpu max,sve=on,sve128=on,sve256=on,sve512=off
-cpu cortex-a72,pauth=off

# KVM 下:
-cpu host,pmu=on
```

---

## 4. 内存配置

### 4.1 基本配置

```bash
-m 2G                   # 2GB RAM
-m 4096M               # 4096MB RAM
-m 512M,maxmem=8G,slots=4  # 支持热插到 8GB
```

### 4.2 NUMA 拓扑

```bash
-m 4G \
-smp 4 \
-numa node,memdev=m0,cpus=0-1,nodeid=0 \
-numa node,memdev=m1,cpus=2-3,nodeid=1 \
-object memory-backend-ram,size=2G,id=m0 \
-object memory-backend-ram,size=2G,id=m1
```

### 4.3 大页内存

```bash
-object memory-backend-memfd,id=mem0,size=4G,hugetlb=on,hugetlbsize=2M \
-machine memory-backend=mem0
```

---

## 5. 存储配置

### 5.1 磁盘镜像创建

```bash
# 创建 qcow2 镜像:
qemu-img create -f qcow2 disk.qcow2 40G

# 创建预分配镜像:
qemu-img create -f qcow2 -o preallocation=full disk.qcow2 40G

# 创建 raw 镜像:
qemu-img create -f raw disk.raw 40G

# 镜像信息:
qemu-img info disk.qcow2
```

### 5.2 块设备配置

```bash
# VirtIO 块设备 (推荐):
-drive file=disk.qcow2,format=qcow2,if=none,id=hd0 \
-device virtio-blk-pci,drive=hd0

# 简化写法:
-drive file=disk.qcow2,format=qcow2,if=virtio

# SCSI:
-device virtio-scsi-pci,id=scsi0 \
-drive file=disk.qcow2,format=qcow2,if=none,id=hd0 \
-device scsi-hd,bus=scsi0.0,drive=hd0

# 只读:
-drive file=disk.qcow2,format=qcow2,if=virtio,readonly=on

# CD-ROM:
-drive file=install.iso,format=raw,if=none,id=cd0,media=cdrom \
-device virtio-blk-pci,drive=cd0
```

### 5.3 快照

```bash
# 启用快照模式 (不修改原始镜像):
-drive file=disk.qcow2,format=qcow2,if=virtio,snapshot=on

# 运行时快照 (HMP):
(qemu) savevm snapshot1       # 保存
(qemu) loadvm snapshot1       # 恢复
(qemu) delvm snapshot1        # 删除
(qemu) info snapshots         # 列出

# 命令行快照管理:
qemu-img snapshot -l disk.qcow2           # 列出
qemu-img snapshot -c snap1 disk.qcow2     # 创建
qemu-img snapshot -d snap1 disk.qcow2     # 删除
```

---

## 6. 网络配置

### 6.1 用户模式网络 (默认)

```bash
# 默认网络 (NAT, 无需 root):
-nic user

# 端口转发:
-nic user,hostfwd=tcp::2222-:22,hostfwd=tcp::8080-:80

# 带 model:
-nic user,model=virtio-net-pci,hostfwd=tcp::2222-:22
```

### 6.2 TAP 网络

```bash
# TAP 设备 (需 root 或配置权限):
-nic tap,model=virtio-net-pci,ifname=tap0,script=no,downscript=no

# 使用辅助程序:
-nic tap,model=virtio-net-pci,helper=/usr/lib/qemu/qemu-bridge-helper
```

### 6.3 详细配置

```bash
# 分离 netdev 和 device:
-netdev user,id=net0,hostfwd=tcp::2222-:22 \
-device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:56

# 多网卡:
-netdev user,id=net0 -device virtio-net-pci,netdev=net0 \
-netdev user,id=net1 -device virtio-net-pci,netdev=net1
```

### 6.4 无网络

```bash
-nic none
```

---

## 7. 显示配置

### 7.1 无图形 (串口)

```bash
# 最常用于服务器/CI:
-nographic
# 等价于: -display none -serial mon:stdio
# 退出: Ctrl-A X
```

### 7.2 VNC

```bash
-vnc :0                        # 监听 5900
-vnc :0,password=on            # 需密码 (HMP 设置)
-vnc 0.0.0.0:5                 # 监听所有 IP, 端口 5905
-vnc unix:/tmp/vnc.sock        # Unix socket
```

### 7.3 GTK 窗口

```bash
-display gtk                   # 基本窗口
-display gtk,gl=on             # OpenGL 加速
-display gtk,zoom-to-fit=on    # 自适应窗口大小
```

### 7.4 Spice

```bash
-spice port=5930,disable-ticketing=on
-display spice-app             # 自动启动客户端
```

---

## 8. 固件与引导

### 8.1 直接内核引导

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a72 \
    -m 2G \
    -kernel Image \
    -initrd initramfs.img \
    -append "root=/dev/vda2 console=ttyAMA0" \
    -drive file=rootfs.qcow2,if=virtio \
    -nographic
```

### 8.2 UEFI 引导

```bash
# 使用 AAVMF (ARM UEFI):
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a72 \
    -m 4G \
    -pflash /usr/share/AAVMF/AAVMF_CODE.fd \
    -pflash /var/lib/libvirt/qemu/nvram/vm_VARS.fd \
    -drive file=disk.qcow2,if=virtio \
    -nographic

# 或只读 BIOS (不保存变量):
-bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd
```

### 8.3 引导顺序

```bash
# 优先硬盘, 其次 CD:
-boot order=cd

# 菜单:
-boot menu=on
```

---

## 9. Monitor 基础

### 9.1 访问方式

```bash
# 方式 1: 串口复用 (nographic 时 Ctrl-A C 切换)
-nographic
# Ctrl-A C: 切换 console ↔ monitor
# Ctrl-A X: 退出 QEMU

# 方式 2: telnet
-monitor telnet:127.0.0.1:4444,server,nowait

# 方式 3: Unix socket (QMP)
-qmp unix:/tmp/qmp.sock,server,nowait

# 方式 4: stdio
-monitor stdio
```

### 9.2 常用 HMP 命令

```
# 系统控制:
(qemu) system_reset            # 重启
(qemu) system_powerdown        # 关机
(qemu) quit                    # 退出 QEMU
(qemu) stop / cont             # 暂停/恢复

# 设备热插拔:
(qemu) device_add virtio-net-pci,netdev=net1
(qemu) device_del dev1

# 快照:
(qemu) savevm snap1
(qemu) loadvm snap1

# 信息查询:
(qemu) info status
(qemu) info cpus
(qemu) info block
(qemu) info network
```

### 9.3 QMP 基础

```bash
# 连接:
nc -U /tmp/qmp.sock

# 初始化:
{"execute": "qmp_capabilities"}

# 查询:
{"execute": "query-status"}
{"execute": "query-block"}
{"execute": "query-cpus-fast"}

# 控制:
{"execute": "stop"}
{"execute": "cont"}
{"execute": "system_reset"}
{"execute": "quit"}
```

---

## 10. 常用场景

### 10.1 开发内核

```bash
qemu-system-aarch64 \
    -M virt -cpu max -m 4G -smp 4 \
    -kernel arch/arm64/boot/Image \
    -initrd rootfs.cpio.gz \
    -append "console=ttyAMA0 nokaslr" \
    -nographic \
    -s -S    # GDB 调试
```

### 10.2 运行发行版 (UEFI)

```bash
qemu-system-aarch64 \
    -M virt -cpu max -m 8G -smp 4 \
    -accel kvm \
    -pflash AAVMF_CODE.fd \
    -pflash VARS.fd \
    -drive file=ubuntu.qcow2,if=virtio \
    -nic user,hostfwd=tcp::2222-:22 \
    -display gtk
```

### 10.3 CI/测试 (无头)

```bash
qemu-system-aarch64 \
    -M virt -cpu max -m 2G \
    -kernel Image -initrd initrd.img \
    -append "console=ttyAMA0 panic=1" \
    -nographic -no-reboot \
    -serial mon:stdio
```

### 10.4 用户模式运行程序

```bash
# 直接运行 ARM64 可执行文件 (需 binfmt 或直接调用):
qemu-aarch64 -L /usr/aarch64-linux-gnu ./hello

# 带 strace:
qemu-aarch64 -d strace ./hello

# GDB 调试:
qemu-aarch64 -g 1234 ./hello
# 另一终端: gdb-multiarch ./hello -ex "target remote :1234"
```

### 10.5 网络实验室

```bash
# 两台 VM 互联:
# VM1:
qemu-system-aarch64 ... \
    -netdev socket,id=net0,listen=:1234 \
    -device virtio-net-pci,netdev=net0

# VM2:
qemu-system-aarch64 ... \
    -netdev socket,id=net0,connect=127.0.0.1:1234 \
    -device virtio-net-pci,netdev=net0
```

---

## 附录: Ctrl-A 快捷键 (nographic 模式)

| 快捷键 | 功能 |
|--------|------|
| `Ctrl-A C` | 切换 serial ↔ monitor |
| `Ctrl-A X` | 退出 QEMU |
| `Ctrl-A S` | 保存磁盘数据 |
| `Ctrl-A T` | 切换时间戳 |
| `Ctrl-A B` | 发送 Break (SysRq) |
| `Ctrl-A H` | 帮助 |

---

*文档结束*
