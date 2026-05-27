# Doc 105: QEMU 高级用法实践指南

## 文档信息
- **组件**: 高级配置, 性能调优, 安全, 迁移, 设备直通, 自动化
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: practice/

---

## 目录
1. [KVM 加速与性能调优](#1-kvm-加速与性能调优)
2. [VirtIO 高性能配置](#2-virtio-高性能配置)
3. [设备直通 (VFIO)](#3-设备直通-vfio)
4. [热迁移](#4-热迁移)
5. [快照与备份](#5-快照与备份)
6. [安全配置](#6-安全配置)
7. [多核与 NUMA 优化](#7-多核与-numa-优化)
8. [块设备高级配置](#8-块设备高级配置)
9. [网络高级配置](#9-网络高级配置)
10. [TCG 高级选项](#10-tcg-高级选项)
11. [自动化与编排](#11-自动化与编排)
12. [确定性重放](#12-确定性重放)
13. [安全启动与加密](#13-安全启动与加密)
14. [调优检查清单](#14-调优检查清单)

---

## 1. KVM 加速与性能调优

### 1.1 启用 KVM

```bash
# 检查 KVM 支持:
ls /dev/kvm
cat /sys/module/kvm/parameters/*

# 基本 KVM:
qemu-system-aarch64 -M virt -accel kvm -cpu host ...

# KVM + 性能选项:
qemu-system-aarch64 -M virt \
    -accel kvm \
    -cpu host,pmu=on \
    -m 8G \
    ...
```

### 1.2 Dirty Ring (高性能脏页追踪)

```bash
# 启用 dirty ring (替代 dirty bitmap):
-accel kvm,dirty-ring-size=65536

# 优势:
# - per-vCPU ring buffer, 避免全局锁
# - 减少迁移时的 KVM_GET_DIRTY_LOG ioctl 开销
# - 适合大内存 VM
```

### 1.3 内存优化

```bash
# 大页:
-object memory-backend-memfd,id=mem0,size=8G,hugetlb=on,hugetlbsize=1G \
-machine memory-backend=mem0

# 共享内存 (用于 vhost-user):
-object memory-backend-memfd,id=mem0,size=8G,share=on \
-machine memory-backend=mem0

# 内存预分配 (避免缺页中断):
-object memory-backend-memfd,id=mem0,size=8G,prealloc=on \
-machine memory-backend=mem0
```

### 1.4 中断优化

```bash
# irqfd (中断绕过 QEMU 用户态):
# 默认启用 (VirtIO 设备自动使用)

# 验证 irqfd 状态:
(qemu) info kvm
# 应显示 irqfd: supported
```

---

## 2. VirtIO 高性能配置

### 2.1 VirtIO-blk 优化

```bash
# 多队列:
-device virtio-blk-pci,drive=hd0,num-queues=4,iothread=iot0 \
-object iothread,id=iot0

# I/O 线程 (从主循环卸载 I/O):
-object iothread,id=iot0 \
-device virtio-blk-pci,drive=hd0,iothread=iot0

# AIO + Direct I/O:
-drive file=disk.qcow2,format=qcow2,if=none,id=hd0,\
aio=io_uring,cache=none
```

### 2.2 VirtIO-net 优化

```bash
# 多队列:
-netdev tap,id=net0,queues=4,vhost=on \
-device virtio-net-pci,netdev=net0,mq=on,vectors=10

# vhost-net (内核态数据路径):
-netdev tap,id=net0,vhost=on \
-device virtio-net-pci,netdev=net0
```

### 2.3 vhost-user (用户态数据路径)

```bash
# 存储: vhost-user-blk
-chardev socket,id=char0,path=/tmp/vhost-blk.sock \
-device vhost-user-blk-pci,chardev=char0,num-queues=4

# 网络: vhost-user-net (配合 DPDK/OVS)
-chardev socket,id=char0,path=/tmp/vhost-net.sock \
-netdev vhost-user,id=net0,chardev=char0,queues=4 \
-device virtio-net-pci,netdev=net0,mq=on
```

---

## 3. 设备直通 (VFIO)

### 3.1 PCI 设备直通

```bash
# 准备 (绑定 VFIO 驱动):
echo "vfio-pci" > /sys/bus/pci/devices/0000:01:00.0/driver_override
echo 0000:01:00.0 > /sys/bus/pci/drivers_probe

# QEMU 配置:
qemu-system-aarch64 -M virt \
    -accel kvm -cpu host \
    -device vfio-pci,host=0000:01:00.0 \
    ...

# 带 IOMMU (SMMU):
-M virt,iommu=smmuv3 \
-device vfio-pci,host=0000:01:00.0,iommu_platform=on
```

### 3.2 GPU 直通

```bash
# NVIDIA GPU:
-device vfio-pci,host=0000:41:00.0,multifunction=on,x-vga=on \
-device vfio-pci,host=0000:41:00.1  # 音频功能

# 需要 IOMMU Group 中所有设备一起直通
```

### 3.3 网卡直通 (SR-IOV VF)

```bash
# 直通 SR-IOV Virtual Function:
-device vfio-pci,host=0000:03:10.0
# 性能接近裸机, 延迟 < 3μs
```

---

## 4. 热迁移

### 4.1 基本迁移

```bash
# 源端:
(qemu) migrate tcp:dest_host:4444

# 目的端 (先启动相同配置, 等待迁移):
qemu-system-aarch64 ... -incoming tcp:0:4444
```

### 4.2 QMP 迁移

```json
// 目的端:
{"execute": "migrate-incoming", "arguments": {"uri": "tcp:0:4444"}}

// 源端:
{"execute": "migrate", "arguments": {"uri": "tcp:dest:4444"}}
{"execute": "query-migrate"}
```

### 4.3 高级迁移选项

```bash
# Multifd (多通道并行传输):
(qemu) migrate_set_capability multifd on
(qemu) migrate_set_parameter multifd-channels 8

# Postcopy (先迁再拉):
(qemu) migrate_set_capability postcopy-ram on
(qemu) migrate
(qemu) migrate_start_postcopy  # 切换到 postcopy

# 压缩:
(qemu) migrate_set_capability compress on
(qemu) migrate_set_parameter compress-threads 8

# XBZRLE (增量压缩):
(qemu) migrate_set_capability xbzrle on
(qemu) migrate_set_parameter xbzrle-cache-size 256M

# 脏页速率限制:
(qemu) migrate_set_parameter max-bandwidth 1G
(qemu) migrate_set_parameter downtime-limit 300  # ms
```

### 4.4 迁移监控

```json
{"execute": "query-migrate"}
// 返回: status, ram (transferred/remaining/total/dirty-rate), 
//       expected-downtime, setup-time
```

---

## 5. 快照与备份

### 5.1 内部快照

```bash
# VM 运行中:
(qemu) savevm checkpoint1
(qemu) loadvm checkpoint1
(qemu) delvm checkpoint1
(qemu) info snapshots
```

### 5.2 外部快照 (QMP)

```json
// 磁盘快照:
{"execute": "blockdev-snapshot-sync",
 "arguments": {"device": "drive0",
               "snapshot-file": "/path/to/snap.qcow2",
               "format": "qcow2"}}

// 恢复: 使用快照文件启动
```

### 5.3 增量备份

```json
// 完整备份:
{"execute": "drive-backup",
 "arguments": {"device": "drive0",
               "target": "/backup/full.qcow2",
               "sync": "full",
               "format": "qcow2"}}

// 增量备份 (基于 dirty bitmap):
{"execute": "block-dirty-bitmap-add",
 "arguments": {"node": "drive0", "name": "bitmap0"}}
// ... 运行一段时间 ...
{"execute": "drive-backup",
 "arguments": {"device": "drive0",
               "target": "/backup/incr1.qcow2",
               "sync": "incremental",
               "bitmap": "bitmap0"}}
```

---

## 6. 安全配置

### 6.1 VNC 安全

```bash
# TLS 加密:
-object tls-creds-x509,id=tls0,dir=/etc/pki/qemu,endpoint=server \
-vnc :0,tls-creds=tls0

# SASL 认证:
-vnc :0,sasl=on
# 需配置 /etc/sasl2/qemu.conf

# 密码 + TLS:
-vnc :0,password=on,tls-creds=tls0
```

### 6.2 QMP 安全

```bash
# Unix socket (文件权限控制):
-qmp unix:/var/run/qemu/qmp.sock,server,nowait

# 不要使用 TCP 暴露 QMP!
```

### 6.3 沙箱

```bash
# 启用 seccomp 沙箱:
-sandbox on,obsolete=deny,elevateprivileges=deny,spawn=deny,resourcecontrol=deny
```

### 6.4 SELinux / AppArmor

```bash
# 使用 sVirt 标签:
-seclabel type=svirt_t,level=s0:c42,c123
```

---

## 7. 多核与 NUMA 优化

### 7.1 CPU 绑定

```bash
# 使用 taskset 绑定 vCPU 线程:
taskset -c 0-3 qemu-system-aarch64 -smp 4 ...

# 或运行时 (找到 vCPU 线程后绑定):
(qemu) info cpus
# → 获取线程 ID
taskset -cp 0 <tid_vcpu0>
taskset -cp 1 <tid_vcpu1>
```

### 7.2 NUMA 感知

```bash
# Guest NUMA 映射到 Host NUMA:
-object memory-backend-ram,size=4G,id=m0,host-nodes=0,policy=bind \
-object memory-backend-ram,size=4G,id=m1,host-nodes=1,policy=bind \
-numa node,memdev=m0,cpus=0-3,nodeid=0 \
-numa node,memdev=m1,cpus=4-7,nodeid=1

# NUMA 距离:
-numa dist,src=0,dst=1,val=20
```

### 7.3 IOThread 分离

```bash
# 将块 I/O 从 vCPU 线程分离:
-object iothread,id=iot0 \
-object iothread,id=iot1 \
-device virtio-blk-pci,drive=hd0,iothread=iot0 \
-device virtio-blk-pci,drive=hd1,iothread=iot1
```

---

## 8. 块设备高级配置

### 8.1 Cache 模式

| 模式 | Host Page Cache | 写入行为 | 适用 |
|------|----------------|----------|------|
| `none` | 绕过 | O_DIRECT | 最佳性能 (推荐) |
| `writethrough` | 使用 | 同步写 | 数据安全 |
| `writeback` | 使用 | 异步写 | 平衡 |
| `unsafe` | 使用 | 忽略 flush | 最快 (不安全) |

```bash
-drive file=disk.qcow2,cache=none,aio=io_uring ...
```

### 8.2 I/O 调度

```bash
# io_uring (Linux 5.1+, 推荐):
-drive file=disk.qcow2,aio=io_uring ...

# 线程池 (传统):
-drive file=disk.qcow2,aio=threads ...

# 原生 AIO:
-drive file=disk.raw,aio=native,cache=none ...
```

### 8.3 限速 (QoS)

```bash
# 限制 IOPS 和带宽:
-drive file=disk.qcow2,throttling.iops-total=1000,\
throttling.bps-total=100M

# 运行时调整:
{"execute": "block_set_io_throttle",
 "arguments": {"device": "drive0", "iops": 2000, "bps": 200000000}}
```

### 8.4 镜像链与 Backing

```bash
# 创建增量镜像:
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay.qcow2

# commit (合并到 base):
qemu-img commit overlay.qcow2

# rebase (更换 backing):
qemu-img rebase -b new_base.qcow2 -F qcow2 overlay.qcow2
```

---

## 9. 网络高级配置

### 9.1 多队列 TAP

```bash
# 创建多队列 TAP:
ip tuntap add dev tap0 mode tap multi_queue

qemu-system-aarch64 ... \
    -netdev tap,id=net0,ifname=tap0,queues=4,vhost=on \
    -device virtio-net-pci,netdev=net0,mq=on,vectors=10
```

### 9.2 macvtap (直连物理网络)

```bash
# 创建 macvtap:
ip link add link eth0 name macvtap0 type macvtap mode bridge

qemu-system-aarch64 ... \
    -netdev tap,id=net0,fd=3 3<>/dev/tap$(cat /sys/class/net/macvtap0/ifindex) \
    -device virtio-net-pci,netdev=net0
```

### 9.3 网络过滤

```bash
# 流量镜像:
-object filter-mirror,id=m0,netdev=net0,queue=rx,outdev=mirror0
-chardev socket,id=mirror0,host=192.168.1.100,port=9000

# 流量转储:
-object filter-dump,id=d0,netdev=net0,file=/tmp/traffic.pcap
```

---

## 10. TCG 高级选项

### 10.1 TCG 配置

```bash
# 多线程 TCG:
-accel tcg,thread=multi

# 单线程 TCG (确定性):
-accel tcg,thread=single

# TB 大小限制:
-accel tcg,tb-size=2048  # 代码缓存 2GB

# 指令计数模式 (确定性时间):
-icount shift=1,align=off,sleep=on
```

### 10.2 TCG 插件

```bash
# 指令计数:
-plugin libexeclog.so

# 内存追踪:
-plugin libmem.so

# 缓存模拟:
-plugin libcache.so,l1-isize=32768,l1-dsize=32768

# 自定义插件:
-plugin /path/to/my_plugin.so,arg1=val1
```

### 10.3 perf 集成

```bash
# perfmap (perf report 可识别 TB):
-accel tcg,perfmap=true

# jitdump (更详细的 JIT 信息):
-accel tcg,jitdump=true
```

---

## 11. 自动化与编排

### 11.1 QMP 自动化脚本

```python
#!/usr/bin/env python3
import json, socket

def qmp_command(sock, cmd, **args):
    msg = {"execute": cmd}
    if args:
        msg["arguments"] = args
    sock.sendall(json.dumps(msg).encode() + b'\n')
    return json.loads(sock.recv(65536))

# 连接
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('/tmp/qmp.sock')
s.recv(4096)  # greeting
qmp_command(s, 'qmp_capabilities')

# 查询
status = qmp_command(s, 'query-status')
print(f"VM status: {status['return']['status']}")

# 快照
qmp_command(s, 'human-monitor-command',
            **{'command-line': 'savevm checkpoint1'})
```

### 11.2 libvirt 集成

```xml
<!-- ARM64 VM 定义 -->
<domain type='kvm'>
  <name>arm64-vm</name>
  <memory unit='GiB'>8</memory>
  <vcpu>4</vcpu>
  <os>
    <type arch='aarch64' machine='virt'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/AAVMF/AAVMF_CODE.fd</loader>
  </os>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native'/>
      <source file='/var/lib/libvirt/images/vm.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
  </devices>
</domain>
```

### 11.3 Cloud-init

```bash
# 创建 cloud-init 配置:
cat > user-data <<EOF
#cloud-config
hostname: my-vm
users:
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-rsa AAAA...
EOF

# 生成 seed ISO:
cloud-localds seed.iso user-data

# 使用:
qemu-system-aarch64 ... \
    -drive file=ubuntu-cloud.qcow2,if=virtio \
    -drive file=seed.iso,if=virtio,format=raw
```

---

## 12. 确定性重放

### 12.1 录制

```bash
qemu-system-aarch64 -M virt -cpu max -m 2G \
    -icount shift=auto,rr=record,rrfile=replay.bin \
    -net none \
    -drive file=disk.qcow2,if=none,id=hd0,snapshot=on \
    -device virtio-blk-pci,drive=hd0 \
    -nographic
```

### 12.2 重放

```bash
qemu-system-aarch64 -M virt -cpu max -m 2G \
    -icount shift=auto,rr=replay,rrfile=replay.bin \
    -net none \
    -drive file=disk.qcow2,if=none,id=hd0,snapshot=on \
    -device virtio-blk-pci,drive=hd0 \
    -nographic \
    -s -S    # 可配合 GDB 反向调试
```

### 12.3 反向调试

```
(gdb) target remote :1234
(gdb) break panic
(gdb) continue
# 命中断点后:
(gdb) reverse-continue    # 反向运行
(gdb) reverse-step        # 反向单步
```

---

## 13. 安全启动与加密

### 13.1 UEFI 安全启动

```bash
# 使用带安全启动的固件:
-pflash /usr/share/AAVMF/AAVMF_CODE.secboot.fd \
-pflash VARS.secboot.fd

# 需要在 UEFI Shell 中导入 PK/KEK/db 证书
```

### 13.2 内存加密 (概念)

```bash
# AMD SEV (x86, 参考):
-object sev-guest,id=sev0,cbitpos=47,reduced-phys-bits=1 \
-machine memory-encryption=sev0

# ARM CCA Realm (未来):
# 待 QEMU 支持 RME realm 后可用
```

### 13.3 磁盘加密

```bash
# LUKS 加密镜像:
qemu-img create -f qcow2 --object secret,id=sec0,data=mypassword \
    -o encrypt.format=luks,encrypt.key-secret=sec0 encrypted.qcow2 40G

# 使用:
-object secret,id=sec0,data=mypassword \
-drive file=encrypted.qcow2,encrypt.key-secret=sec0
```

---

## 14. 调优检查清单

### 14.1 CPU 性能

| 项目 | 检查 | 推荐 |
|------|------|------|
| 加速器 | `info kvm` | 使用 KVM |
| CPU 型号 | `-cpu` | host (KVM) / max (TCG) |
| vCPU 绑定 | taskset | 绑定到物理核 |
| NUMA | `/proc/cmdline` | 对齐 host NUMA |

### 14.2 内存性能

| 项目 | 检查 | 推荐 |
|------|------|------|
| 大页 | `hugetlb=on` | 1G 或 2M 大页 |
| 预分配 | `prealloc=on` | 避免缺页 |
| 脏页追踪 | `dirty-ring-size` | 启用 Dirty Ring |
| NUMA 绑定 | `host-nodes` | 对齐 NUMA node |

### 14.3 存储性能

| 项目 | 检查 | 推荐 |
|------|------|------|
| 设备类型 | `-device` | virtio-blk-pci |
| Cache | `cache=` | none (O_DIRECT) |
| AIO | `aio=` | io_uring |
| IOThread | `iothread=` | 独立 IO 线程 |
| 多队列 | `num-queues=` | ≥ vCPU 数 |

### 14.4 网络性能

| 项目 | 检查 | 推荐 |
|------|------|------|
| 后端 | netdev | TAP + vhost |
| 多队列 | `queues=` | ≥ vCPU 数 |
| 卸载 | 设备特性 | 启用 GSO/GRO |
| 直通 | VFIO | SR-IOV VF |

### 14.5 快速调优模板

```bash
# 生产环境高性能 ARM64 VM:
qemu-system-aarch64 \
    -M virt,gic-version=3 \
    -accel kvm,dirty-ring-size=65536 \
    -cpu host,pmu=on \
    -smp 8,sockets=1,cores=8,threads=1 \
    -m 16G \
    -object memory-backend-memfd,id=mem0,size=16G,hugetlb=on,\
hugetlbsize=1G,prealloc=on,share=on \
    -machine memory-backend=mem0 \
    -object iothread,id=iot0 \
    -drive file=vm.qcow2,format=qcow2,if=none,id=hd0,\
cache=none,aio=io_uring \
    -device virtio-blk-pci,drive=hd0,iothread=iot0,num-queues=8 \
    -netdev tap,id=net0,ifname=tap0,vhost=on,queues=8 \
    -device virtio-net-pci,netdev=net0,mq=on,vectors=18 \
    -vnc :0 \
    -qmp unix:/var/run/qemu/qmp.sock,server,nowait
```

---

## 附录: 性能对比参考

| 配置 | 相对性能 | 延迟 |
|------|----------|------|
| TCG 单线程 | 1x (基准) | 高 |
| TCG 多线程 | 2-4x | 中 |
| KVM (默认) | 50-80x | 低 |
| KVM + 大页 | 60-90x | 极低 |
| KVM + VFIO 直通 | ~100x | 接近裸机 |

---

*文档结束*
