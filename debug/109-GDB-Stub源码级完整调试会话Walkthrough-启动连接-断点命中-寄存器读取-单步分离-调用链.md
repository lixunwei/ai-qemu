# Doc 109: GDB Stub 源码级完整调试会话 Walkthrough

## 文档信息
- **组件**: GDB Stub, RSP 协议引擎, 断点执行路径, 系统模式调试
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **归档目录**: debug/

---

## 目录
1. [概述](#1-概述)
2. [Phase 1: GDB Server 启动](#2-phase-1-gdb-server-启动)
3. [Phase 2: GDB 连接与能力协商](#3-phase-2-gdb-连接与能力协商)
4. [Phase 3: 设置断点](#4-phase-3-设置断点)
5. [Phase 4: 继续执行与断点命中](#5-phase-4-继续执行与断点命中)
6. [Phase 5: 读取寄存器](#6-phase-5-读取寄存器)
7. [Phase 6: 单步执行](#7-phase-6-单步执行)
8. [Phase 7: 分离与终止](#8-phase-7-分离与终止)
9. [完整调用链图](#9-完整调用链图)
10. [数据流层次模型](#10-数据流层次模型)
11. [关键状态转换](#11-关键状态转换)
12. [源码索引](#12-源码索引)

---

## 1. 概述

本文追踪一次 **完整的 GDB 调试会话** 在 QEMU 源码中的精确执行路径。场景:

```
qemu-system-aarch64 -M virt -cpu max -kernel Image -s -S
                                                      ↑    ↑
                                            tcp::1234  暂停等待
```

GDB 端操作:
```
(gdb) target remote :1234        ← Phase 2
(gdb) break *0xffff800080000000  ← Phase 3
(gdb) continue                   ← Phase 4 (命中后自动停止)
(gdb) info registers             ← Phase 5
(gdb) stepi                      ← Phase 6
(gdb) detach                     ← Phase 7
```

---

## 2. Phase 1: GDB Server 启动

### 2.1 命令行解析

```
启动: qemu-system-aarch64 ... -s -S
```

源码: `system/vl.c:1281-1307`

```c
// -s 等价于 -gdb tcp::1234
// -S 设置 autostart = 0 (启动后暂停)

case QEMU_OPTION_gdb:
    // 记录为 DEV_GDB 类型设备
    add_device_config(DEV_GDB, optarg);  // optarg = "tcp::1234"
    break;

case QEMU_OPTION_s:
    add_device_config(DEV_GDB, "tcp::" DEFAULT_GDBSTUB_PORT);
    break;
```

### 2.2 GDB Server 初始化

源码: `gdbstub/system.c:333-411`

```c
gdbserver_start(const char *device):
    │
    ├─ 1. qemu_chr_new_noreplay("gdb", device)
    │      // 创建 chardev 后端 (TCP socket)
    │      // device = "tcp::1234,server=on,wait=off"
    │      └─ chardev/char-socket.c:1344-1427
    │         tcp_chr_open() → qmp_chardev_open_socket_server()
    │         → qio_net_listener_open_sync()  // 绑定端口, 开始监听
    │
    ├─ 2. qemu_chr_fe_init(&gdbserver_system_state.chr, chr)
    │      // 初始化前端接口
    │
    ├─ 3. qemu_chr_fe_set_handlers(&...chr,
    │         gdb_chr_can_receive,    // 可接收回调
    │         gdb_chr_receive,        // 数据到达回调
    │         gdb_chr_event,          // 事件回调 (连接/断开)
    │         NULL, NULL, NULL, true)
    │
    └─ 4. 此时 server socket 已在监听 tcp:1234
           等待 GDB 客户端连接...
```

### 2.3 GDB 客户端连接

当 GDB 执行 `target remote :1234` 时, TCP 连接建立:

```
chardev/char-socket.c:957-967
tcp_chr_accept():
    │
    └─ 触发 CHR_EVENT_OPENED 事件
       │
       └─ gdbstub/system.c:85-105
          gdb_chr_event(CHR_EVENT_OPENED):
              │
              ├─ 设置 gdbserver_state.init = true
              ├─ 为当前第一个 CPU 标记为 attached
              └─ vm_stop(RUN_STATE_PAUSED)  // 如果 VM 在运行则暂停
```

**状态转换**: `Server Listening` → `GDB Connected, VM Paused`

---

## 3. Phase 2: GDB 连接与能力协商

### 3.1 GDB 发送 qSupported

GDB 连接后首先发送:
```
$qSupported:multiprocess+;swbreak+;hwbreak+;...#xx
```

### 3.2 数据接收: RSP 包组装

源码: `gdbstub/gdbstub.c:2330-2485`

```c
gdb_read_byte(uint8_t ch):
    // 状态机: 逐字节组装 RSP 包
    switch (gdbserver_state.state):
        case RS_IDLE:
            if (ch == '$')  → state = RS_GETLINE  // 包开始
            if (ch == 0x03) → gdb_handle_break()  // Ctrl-C 中断
            
        case RS_GETLINE:
            if (ch == '#')  → state = RS_CHKSUM1  // 校验和开始
            else → 累积到 line_buf[]
            // 处理转义 (0x7d) 和 RLE (0x2a)
            
        case RS_CHKSUM1:
            记录校验高位 → state = RS_CHKSUM2
            
        case RS_CHKSUM2:
            验证校验和
            if (valid) → 发送 '+' (ACK)
                       → gdb_handle_packet(line_buf)  // 分发!
            else       → 发送 '-' (NAK, 请重传)
```

### 3.3 命令分发

源码: `gdbstub/gdbstub.c:2062-2195`

```c
gdb_handle_packet(const char *line_buf):
    switch (line_buf[0]):
        case 'q':   // 查询命令
            handle_query_packet(line_buf + 1)
                │
                └─ 匹配 "Supported" → handle_query_supported()
        case 'v':   // v 命令族
            handle_v_commands(line_buf + 1)
        case 'g':   // 读所有寄存器
            handle_read_all_regs()
        case 'G':   // 写所有寄存器
            handle_write_all_regs()
        case 'm':   // 读内存
            handle_read_mem()
        case 'Z':   // 插入断点
            handle_insert_bp()
        case 'c':   // 继续
            handle_continue()
        case 's':   // 单步
            handle_step()
        case 'D':   // 分离
            handle_detach()
        case 'k':   // 终止
            handle_kill()
        ...
```

### 3.4 qSupported 响应

源码: `gdbstub/gdbstub.c:1678-1722`

```c
handle_query_supported():
    // 构建响应字符串:
    gdb_fmt_reply("PacketSize=%x", MAX_PACKET_LENGTH);
    gdb_fmt_reply(";qXfer:features:read+");
    gdb_fmt_reply(";vContSupported+");
    gdb_fmt_reply(";multiprocess+");
    
    // 条件特性:
    if (replay_mode != REPLAY_MODE_NONE):
        gdb_fmt_reply(";ReverseStep+;ReverseContinue+");
    
    // User-mode 特有:
    if (user_mode):
        gdb_fmt_reply(";qXfer:auxv:read+");
        gdb_fmt_reply(";QCatchSyscalls+");
        gdb_fmt_reply(";qXfer:siginfo:read+");
```

### 3.5 XML 目标描述请求

GDB 接着发送 `qXfer:features:read:target.xml:0,fff`:

源码: `gdbstub/gdbstub.c:1725-1772`

```c
handle_query_xfer_features():
    │
    ├─ 解析 annex (文件名), offset, length
    ├─ gdb_get_xml(annex)
    │      └─ 对于 ARM64 主 CPU:
    │         返回动态生成的 target.xml, 内含:
    │         <xi:include href="aarch64-core.xml"/>
    │         <xi:include href="aarch64-fpu.xml"/>
    │         <xi:include href="system-registers.xml"/>  (动态)
    │         <xi:include href="sve-registers.xml"/>     (动态, 如果有 SVE)
    │         ...
    └─ gdb_put_packet() 分片返回 XML 内容
```

初始化时 CPU 注册 XML:

源码: `gdbstub/gdbstub.c:487-616` + `target/arm/gdbstub64.c:883-933`

```c
gdb_init_cpu(cpu):
    // ARM64:
    aarch64_cpu_register_gdb_regs_for_features(cpu):
        gdb_register_coprocessor(cs, aarch64_fpu_gdb_get_reg, ..., "aarch64-fpu.xml")
        if (sve): gdb_register_coprocessor(cs, ..., "sve-registers.xml")  // 动态
        if (sme): gdb_register_coprocessor(cs, ..., "sme-registers.xml")
        gdb_register_coprocessor(cs, ..., "aarch64-pauth.xml")
        gdb_register_coprocessor(cs, ..., "system-registers.xml")  // 动态
```

**状态转换**: `VM Paused` → `GDB 已获知 CPU 架构, 等待用户命令`

---

## 4. Phase 3: 设置断点

### 4.1 GDB 发送 Z0 包

```
$Z0,ffff800080000000,4#xx
 ↑  ↑                 ↑
 │  地址               长度 (ARM64 指令 = 4 字节)
 软件断点类型
```

### 4.2 断点插入调用链

```
gdbstub/gdbstub.c:1166-1188
handle_insert_bp(line_buf):
    │
    ├─ 解析: type=0, addr=0xffff800080000000, len=4
    │
    └─ gdb_breakpoint_insert(cpu, type, addr, len)
       │
       └─ gdbstub/system.c:640-647
          gdb_breakpoint_insert():
              │
              ├─ type == GDB_BREAKPOINT_SW (0):
              │      cpu_breakpoint_insert(cpu, addr, BP_GDB, NULL)
              │
              ├─ type == GDB_BREAKPOINT_HW (1):
              │      cpu_breakpoint_insert(cpu, addr, BP_GDB | BP_HW, NULL)
              │
              └─ type == GDB_WATCHPOINT_* (2/3/4):
                     cpu_watchpoint_insert(cpu, addr, len, type, NULL)
```

### 4.3 TCG 断点注册

源码: `accel/tcg/tcg-accel-ops.c:133-162`

```c
tcg_insert_breakpoint(cpu, addr, ...):
    │
    ├─ cpu_breakpoint_insert(cpu, addr, BP_GDB, &bp)
    │      // 在 cpu->breakpoints QTAILQ 中添加 CPUBreakpoint 节点
    │      // 标记该地址对应的 TB 为无效 (需要重翻译)
    │
    └─ tb_flush(cpu)  // 可能需要刷新包含该地址的 TB
```

### 4.4 断点在执行循环中的检查

源码: `accel/tcg/cpu-exec.c:328, 682-683`

```c
// TB 翻译时:
translator_loop():
    // 每条指令翻译前检查:
    if (cpu_breakpoint_test(cpu, pc, BP_GDB | BP_ANY)):
        // 在此指令处生成 TB 退出
        // 设置 EXCP_DEBUG 异常
        gen_exception_debug()

// 执行循环中:
cpu_exec_loop():
    ...
    if (cpu->exception_index == EXCP_DEBUG):
        cpu_handle_guest_debug(cpu)
        // → gdb_handlesig() → 发送停止回复给 GDB
```

### 4.5 ARM64 断点匹配

源码: `target/arm/tcg/debug.c:652-736`

```c
arm_debug_check_breakpoint(env, pc):
    // 遍历 DBGBCR/DBGBVR (如果使用硬件断点):
    for (int i = 0; i < max_hw_bps; i++):
        if (bp_enabled(env->cp15.dbgbcr[i])):
            if (bp_match(env->cp15.dbgbvr[i], pc)):
                return true;
    
    // GDB 软件断点由 cpu->breakpoints 列表匹配
```

**响应**: QEMU 发送空回复 `$OK#9a` 确认断点设置成功。

---

## 5. Phase 4: 继续执行与断点命中

### 5.1 GDB 发送 continue

```
$vCont;c#xx     (或简单的 $c#xx)
```

### 5.2 vCont 解析

源码: `gdbstub/gdbstub.c:735-875`

```c
gdb_handle_vcont(line_buf):
    │
    ├─ 解析 "c" → 所有线程继续
    │   或 "c:1" → 线程 1 继续
    │   或 "s:1;c" → 线程 1 单步, 其他继续
    │
    ├─ 构建 newstates[] 数组:
    │   newstates[cpu_index] = 'c' (continue) 或 's' (step)
    │
    └─ gdb_continue_partial(newstates)
```

### 5.3 恢复 VM 执行

源码: `gdbstub/system.c:558-603`

```c
gdb_continue_partial(newstates):
    │
    ├─ for each CPU:
    │      if newstates[i] == 'c':
    │          cpu_resume(cpu)           // 恢复 CPU
    │      elif newstates[i] == 's':
    │          cpu_single_step(cpu, sstep_flags)  // 设置单步
    │          cpu_resume(cpu)
    │
    └─ vm_start()  // 恢复虚拟时钟, 唤醒 vCPU 线程
```

简单 `c` 命令路径:

源码: `gdbstub/system.c:547-553`

```c
gdb_continue():
    vm_start()
    // → qemu_clock_enable(QEMU_CLOCK_VIRTUAL, true)
    // → resume_all_vcpus()
    // → 各 vCPU 线程从 qemu_wait_io_event() 唤醒
    // → 进入 cpu_exec() 主循环
```

### 5.4 断点命中

执行循环运行到 `0xffff800080000000` 时:

```c
// accel/tcg/cpu-exec.c (简化):
cpu_exec_loop():
    tb = tb_lookup(cpu, pc=0xffff800080000000);
    // TB 翻译时已在该地址插入 EXCP_DEBUG 退出
    
    cpu_tb_exec(cpu, tb):
        // 执行到断点指令 → 触发 EXCP_DEBUG
        cpu->exception_index = EXCP_DEBUG;
        return;
    
    // 回到循环:
    cpu_handle_exception(cpu):
        // exception_index == EXCP_DEBUG
        → return true  // 退出 cpu_exec
```

### 5.5 停止通知发送

源码: `gdbstub/system.c:122-217`

```c
// VM state 变化回调 (注册在 gdbserver_start 时):
gdb_vm_state_change(running=false, state=RUN_STATE_DEBUG):
    │
    ├─ gdb_set_stop_cpu(cpu)
    │      // 记录当前停止的 CPU 为 c_cpu 和 g_cpu
    │
    ├─ 构建停止回复:
    │   type = GDB_SIGNAL_TRAP (5)
    │   snprintf(buf, "T%02x", type)  // "T05"
    │   // 附加 thread-id:
    │   append "thread:p01.01;"       // 进程1, 线程1
    │
    └─ gdb_put_packet(buf)
       // 发送: $T05thread:p01.01;#xx
```

GDB 收到 `T05` 后显示:
```
Program received signal SIGTRAP, Trace/breakpoint trap.
0xffff800080000000 in start_kernel ()
```

**状态转换**: `VM Running` → `Breakpoint Hit` → `VM Paused, GDB Notified`

---

## 6. Phase 5: 读取寄存器

### 6.1 GDB 发送 g 包

```
$g#67
```

### 6.2 读取所有寄存器

源码: `gdbstub/gdbstub.c:1346-1363`

```c
handle_read_all_regs():
    │
    ├─ 获取当前 g_cpu (之前 gdb_set_stop_cpu 设置的)
    │
    ├─ for (reg_id = 0; reg_id < cpu->gdb_num_regs; reg_id++):
    │      len = gdb_read_register(cpu, reg_id, mem_buf)
    │      // 将值转为 hex 追加到响应
    │
    └─ gdb_put_packet(hex_buf)
       // 返回所有寄存器的十六进制拼接
```

### 6.3 寄存器读取分发

源码: `gdbstub/gdbstub.c:529-560`

```c
gdb_read_register(cpu, reg_id, buf):
    │
    ├─ if (reg_id < cpu->gdb_num_core_regs):
    │      // 核心寄存器 → CPU class 方法
    │      cc->gdb_read_register(cpu, buf, reg_id)
    │      └─ ARM64: aarch64_cpu_gdb_read_register()
    │
    └─ else:
           // 扩展寄存器组 → 注册的 coprocessor handler
           遍历 cpu->gdb_regs[]:
               if reg_id in [base_reg, base_reg + num_regs):
                   gdb_reg->get_reg(env, buf, reg_id - base_reg)
                   // 如 FP: aarch64_fpu_gdb_get_reg()
                   // 如 SVE: aarch64_sve_gdb_get_reg()
                   // 如 System: arm_gdb_get_sysreg()
```

### 6.4 ARM64 核心寄存器读取

源码: `target/arm/gdbstub64.c:35-55`

```c
aarch64_cpu_gdb_read_register(cpu, buf, reg):
    switch (reg):
        case 0 ... 30:  // x0-x30
            return gdb_get_reg64(buf, env->xregs[reg]);
        
        case 31:  // sp
            return gdb_get_reg64(buf, env->xregs[31]);
        
        case 32:  // pc
            return gdb_get_reg64(buf, env->pc);
        
        case 33:  // cpsr/pstate
            return gdb_get_reg32(buf, pstate_read(env));
```

### 6.5 响应格式

```
$<x0_hex><x1_hex>...<x30_hex><sp_hex><pc_hex><cpsr_hex>#xx
  64-bit   64-bit              64-bit  64-bit  32-bit

// 例: x0=0, x1=0x40000000, ..., pc=0xffff800080000000, cpsr=0x3c5
```

---

## 7. Phase 6: 单步执行

### 7.1 GDB 发送单步命令

```
$vCont;s:p1.1#xx    (单步线程 p1.1)
或简单: $s#xx
```

### 7.2 单步设置

源码: `gdbstub/gdbstub.c:1366-1374`

```c
handle_step():
    │
    ├─ cpu_single_step(gdbserver_state.c_cpu, gdbserver_state.sstep_flags)
    │      // sstep_flags = SSTEP_ENABLE | SSTEP_NOIRQ | SSTEP_NOTIMER
    │      // 设置 cpu->singlestep_enabled = flags
    │
    └─ gdb_continue()  // 恢复执行 (只会执行一条指令)
```

### 7.3 TCG 单步机制

设置 `cpu->singlestep_enabled` 后, TCG 翻译行为改变:

```c
// tcg/translator.c — translator_loop():
tb_cflags(tb) |= CF_SINGLE_STEP;  // TB 编译标志

// 前端翻译每条指令后:
if (dc->base.singlestep_enabled):
    gen_exception(EXCP_DEBUG)  // 每条指令后生成异常
    dc->base.is_jmp = DISAS_NORETURN  // TB 只包含一条指令
```

效果: **TB 最多翻译 1 条 Guest 指令**, 执行后立即触发 EXCP_DEBUG。

### 7.4 单步完成与通知

```c
// 执行一条指令后:
cpu->exception_index = EXCP_DEBUG
→ cpu_handle_exception() 退出 cpu_exec()
→ gdb_vm_state_change(RUN_STATE_DEBUG)
→ 清除单步: cpu_single_step(cpu, 0)     // system.c:208-217
→ gdb_put_packet("T05thread:p01.01;")   // 发送停止通知
```

GDB 显示:
```
0xffff800080000004 in start_kernel+4 ()
```

---

## 8. Phase 7: 分离与终止

### 8.1 Detach (分离)

GDB 发送: `$D#44`

源码: `gdbstub/gdbstub.c:1021-1059`

```c
handle_detach():
    │
    ├─ 解析进程 ID (多进程模式: D;pid)
    │
    ├─ gdb_breakpoint_remove_all(cpu)
    │      // 删除该进程所有 GDB 设置的断点
    │      for each breakpoint with BP_GDB flag:
    │          cpu_breakpoint_remove(cpu, addr, BP_GDB)
    │
    ├─ gdb_put_packet("OK")
    │
    ├─ process->attached = false
    │
    └─ if (所有进程都已分离):
           gdb_disable_syscalls()
           gdb_continue()  // 恢复 VM 运行!
```

### 8.2 Kill (终止)

GDB 发送: `$k#6b`

源码: `gdbstub/gdbstub.c:2118-2123`

```c
case 'k':
    gdb_exit(0)      // 发送 W00 退出通知
    gdb_qemu_exit(0) // 退出 QEMU 进程
```

### 8.3 退出清理

源码: `gdbstub/system.c:421-439`

```c
gdb_exit(int code):
    │
    ├─ snprintf(buf, "W%02x", code)  // "W00"
    ├─ gdb_put_packet(buf)           // 通知 GDB
    │
    └─ qemu_chr_fe_deinit(&gdbserver_system_state.chr, true)
       // 关闭 chardev 前端
       // 释放 socket 资源
```

**状态转换**: `VM Paused` → `Breakpoints Removed` → `VM Running` (detach) 或 `QEMU Exit` (kill)

---

## 9. 完整调用链图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GDB Client (用户)                             │
└───────────────┬───────────────────────────────────────────┬─────────┘
                │ TCP :1234                                  │
┌───────────────▼───────────────────────────────────────────▼─────────┐
│                     Chardev Layer (char-socket.c)                     │
│  tcp_chr_accept() → gdb_chr_receive() → gdb_chr_event()             │
└───────────────┬─────────────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────────────┐
│                   RSP Protocol Engine (gdbstub.c)                     │
│                                                                       │
│  gdb_read_byte() → 包组装 → gdb_handle_packet() → 命令分发          │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │ Query:  handle_query_supported/xfer_features/rcmd           │     │
│  │ Exec:   handle_continue/step/vcont                          │     │
│  │ Regs:   handle_read_all_regs/read_register                  │     │
│  │ Memory: handle_read_mem/write_mem                           │     │
│  │ BP:     handle_insert_bp/remove_bp                          │     │
│  │ Thread: handle_set_thread/query_thread_info                 │     │
│  └─────────────────────────────────────────────────────────────┘     │
└───────────────┬─────────────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────────────┐
│                System Mode Layer (system.c)                           │
│                                                                       │
│  gdb_breakpoint_insert/remove() — 断点管理                           │
│  gdb_continue/continue_partial() — VM 恢复                           │
│  gdb_vm_state_change()           — 停止通知                           │
│  gdb_exit()                      — 退出清理                           │
└───────────────┬─────────────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────────────┐
│                    CPU / Accel Layer                                   │
│                                                                       │
│  cpu_breakpoint_insert/remove()  — 断点记录                           │
│  cpu_single_step()               — 单步标志                           │
│  vm_start() / vm_stop()          — VM 状态控制                        │
│  cpu_exec() → TB lookup → execute → EXCP_DEBUG                       │
└───────────────┬─────────────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────────────┐
│               ARM64 Target Layer (gdbstub64.c / debug.c)              │
│                                                                       │
│  aarch64_cpu_gdb_read/write_register()  — 寄存器访问                  │
│  arm_debug_check_breakpoint()           — ARM 断点匹配                │
│  arm_debug_excp_handler()               — 调试异常处理                │
│  aarch64_cpu_register_gdb_regs()        — 寄存器组注册                │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 10. 数据流层次模型

### 10.1 下行 (GDB → QEMU 执行)

```
GDB 命令 "break *0x..."
    │
    ▼ TCP 字节流
$Z0,ffff800080000000,4#xx
    │
    ▼ char-socket 接收
gdb_chr_receive(buf, len)
    │
    ▼ RSP 解析
gdb_read_byte() × N → 完整包
    │
    ▼ 命令分发
gdb_handle_packet("Z0,...")
    │
    ▼ 断点逻辑
handle_insert_bp() → gdb_breakpoint_insert()
    │
    ▼ CPU 操作
cpu_breakpoint_insert(cpu, 0xffff800080000000, BP_GDB)
    │
    ▼ TB 失效
tb_jmp_cache_clear_page() / tb_flush()
```

### 10.2 上行 (QEMU 事件 → GDB 通知)

```
TB 执行到断点地址
    │
    ▼ 异常产生
cpu->exception_index = EXCP_DEBUG
    │
    ▼ 执行循环退出
cpu_exec() returns
    │
    ▼ VM 暂停
vm_stop(RUN_STATE_DEBUG)
    │
    ▼ 状态变化回调
gdb_vm_state_change()
    │
    ▼ 构建停止包
"T05thread:p01.01;"
    │
    ▼ RSP 封装
gdb_put_packet() → "$T05thread:p01.01;#xx"
    │
    ▼ chardev 发送
qemu_chr_fe_write() → TCP 字节流
    │
    ▼
GDB 收到停止通知, 显示断点命中
```

---

## 11. 关键状态转换

### 11.1 GDB Server 状态

```
                ┌──────────┐
                │ Inactive │
                └────┬─────┘
                     │ gdbserver_start()
                     ▼
                ┌──────────┐
                │ Listening│ (等待连接)
                └────┬─────┘
                     │ tcp_chr_accept()
                     ▼
                ┌──────────┐
          ┌────▶│ Connected│◀───┐
          │     │ (Paused) │    │
          │     └────┬─────┘    │
          │          │ c/vCont  │ BP hit / step done
          │          ▼          │
          │     ┌──────────┐   │
          │     │ Running  │───┘
          │     └────┬─────┘
          │          │ D (detach)
          │          ▼
          │     ┌──────────┐
          │     │ Detached │ (VM 继续运行, GDB 已断开)
          │     └──────────┘
          │
          │ k (kill)
          ▼
    ┌──────────┐
    │   Exit   │
    └──────────┘
```

### 11.2 CPU 调试状态

```
┌───────────────┐     cpu_single_step(flags)     ┌────────────────┐
│ Normal Exec   │ ──────────────────────────────▶ │ Single-Step    │
│ (多指令 TB)   │                                 │ (1指令 TB)     │
└───────┬───────┘                                 └───────┬────────┘
        │ BP hit                                          │ 1 instr done
        ▼                                                 ▼
┌───────────────┐                                 ┌────────────────┐
│ EXCP_DEBUG    │                                 │ EXCP_DEBUG     │
│ (暂停, 通知)  │                                 │ (暂停, 通知)   │
└───────────────┘                                 └────────────────┘
```

---

## 12. 源码索引

| 阶段 | 文件 | 行号 | 函数 |
|------|------|------|------|
| 启动 | `system/vl.c` | 1281-1307 | 命令行解析 -gdb/-s |
| 启动 | `gdbstub/system.c` | 333-411 | `gdbserver_start()` |
| 连接 | `chardev/char-socket.c` | 957-967 | `tcp_chr_accept()` |
| 连接 | `gdbstub/system.c` | 85-105 | `gdb_chr_event()` |
| 包解析 | `gdbstub/gdbstub.c` | 2330-2485 | `gdb_read_byte()` |
| 分发 | `gdbstub/gdbstub.c` | 2062-2195 | `gdb_handle_packet()` |
| 协商 | `gdbstub/gdbstub.c` | 1678-1722 | `handle_query_supported()` |
| XML | `gdbstub/gdbstub.c` | 1725-1772 | `handle_query_xfer_features()` |
| 断点 | `gdbstub/gdbstub.c` | 1166-1188 | `handle_insert_bp()` |
| 断点 | `gdbstub/system.c` | 640-647 | `gdb_breakpoint_insert()` |
| 断点 | `accel/tcg/tcg-accel-ops.c` | 133-162 | `tcg_insert_breakpoint()` |
| 继续 | `gdbstub/gdbstub.c` | 735-875 | `gdb_handle_vcont()` |
| 继续 | `gdbstub/system.c` | 547-603 | `gdb_continue()/partial()` |
| 停止 | `gdbstub/system.c` | 122-217 | `gdb_vm_state_change()` |
| 寄存器 | `gdbstub/gdbstub.c` | 1346-1363 | `handle_read_all_regs()` |
| 寄存器 | `gdbstub/gdbstub.c` | 529-560 | `gdb_read_register()` |
| 寄存器 | `target/arm/gdbstub64.c` | 35-55 | `aarch64_cpu_gdb_read_register()` |
| 单步 | `gdbstub/gdbstub.c` | 1366-1374 | `handle_step()` |
| 单步 | `gdbstub/system.c` | 558-603 | `gdb_continue_partial()` |
| 分离 | `gdbstub/gdbstub.c` | 1021-1059 | `handle_detach()` |
| 退出 | `gdbstub/system.c` | 421-439 | `gdb_exit()` |
| ARM调试 | `target/arm/tcg/debug.c` | 652-736 | `arm_debug_check_breakpoint()` |
| ARM调试 | `target/arm/tcg/debug.c` | 488-507 | `arm_debug_excp_handler()` |

---

*文档结束*
