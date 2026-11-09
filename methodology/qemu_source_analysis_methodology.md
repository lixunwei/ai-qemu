# QEMU 源码分析方法论（精要版）

> **版本**: QEMU 11.0.50 | **文档性质**: 方法论（所有 QEMU 分析任务的默认起点）
> **提炼基础**: 本仓库 ~194 篇分析文档的实践（architecture/tcg/arm64/arm/device-model/memory/monitor/network/debug/practice 十个域）
> **关联文档**: [practice/INDEX.md](../practice/INDEX.md)（动态分析方法）、[80-文档覆盖度分析与缺失规划.md](../80-文档覆盖度分析与缺失规划.md)（任务选择依据）
> **姊妹方法论**: Linux 内核树 `darren/methodology/source_analysis_methodology.md`（通用大型 C 项目框架，本文是其 QEMU 特化精要版）

---

## 目录

- [一、核心哲学](#一核心哲学)
- [二、QEMU 特化分析框架](#二qemu-特化分析框架)
- [三、QEMU 模式速查库](#三qemu-模式速查库)
- [四、工具链](#四工具链)
- [五、本仓库文档规范](#五本仓库文档规范)
- [六、一页纸总结](#六一页纸总结)
- [附录 A: QEMU 10/11 头文件重组对照](#附录-a-qemu-1011-头文件重组对照)
- [附录 B: 动态分析方法索引](#附录-b-动态分析方法索引)

---

## 一、核心哲学

### 1.1 分析的本质

源码分析不是**翻译代码为自然语言**，而是**重建设计者的思维模型**——
回答 WHY（为什么这样设计）和 HOW（取舍如何权衡），而非只复述 WHAT。

### 1.2 五个核心问题（QEMU 版）

对任何 QEMU 子系统，首先回答：

| # | 问题 | QEMU 语境下的具体化 |
|---|------|---------------------|
| 1 | 它解决什么问题？ | 仿真的哪个环节？（翻译 / 设备 / 生命周期 / 迁移...） |
| 2 | 核心抽象是什么？ | **QOM 类型（TypeInfo）+ ops 结构体**（DeviceClass/CPUClass/AccelClass...） |
| 3 | 典型执行路径是什么？ | 沿**执行上下文**（vCPU 循环 / main loop / coroutine / BH）从入口到完成 |
| 4 | 并发如何处理？ | **BQL 边界 + iothread + coroutine** —— 谁持锁、谁让出、谁回踢 |
| 5 | 设计取舍是什么？ | 性能（TB 缓存/无锁 fast path）vs 正确性（迁移一致性）vs 复杂度 |

### 1.3 三个层次

| 层次 | 目标 | QEMU 产出 |
|------|------|-----------|
| **L1 架构** | 系统分解、模块边界 | QOM 类型树 + 数据流图 + 线程模型图 |
| **L2 机制** | 关键路径执行逻辑 | 调用链 + 状态机 + MemoryRegion 拓扑 |
| **L3 实现** | 优化、边界、并发细节 | 代码注解 + fast/slow path 对比 + 锁分析 |

**原则：由 L1→L2→L3 逐层深入。**

### 1.4 QEMU vs 内核的分析视角差异

写过内核分析文档的读者，注意两套心智模型的转换：

| 维度 | Linux 内核 | QEMU |
|------|-----------|------|
| 面向对象 | ops 结构体（约定俗成） | **QOM 统一类型系统**（TypeInfo/class/instance 三层） |
| 并发单元 | CPU/中断/软中断上下文 | **线程**（main loop/vCPU/iothread）+ 协程 + BH |
| 主同步机制 | spinlock/mutex/RCU | **BQL**（粗粒度大锁）+ 细粒度锁 + RCU |
| "中断"类比 | softirq/tasklet | **Bottom Half（BH）/ 定时器**（由事件循环驱动） |
| 地址空间 | 物理内存 + 页表 | **MemoryRegion 树**（FlatView 分发，纯软件构造） |
| 性能敏感热点 | 系统调用/中断路径 | **TB 翻译缓存 / softmmu TLB / 内存分发** |
| 规范依据 | 内核自身文档/邮件列表 | **架构规范**（ARM DDI 0487、GIC IHI 0069）+ QEMU docs/ |
| 版本考古 | 对照高版本树（如 v7.1） | **本仓库 git log**（无并行树；注意头文件重组，见附录 A） |

---

## 二、QEMU 特化分析框架

### Phase 0: 定位与边界

**目标：确定分析范围、入口点、文件规模。**

QEMU 顶层源码布局（11.0.50）：

| 目录 | 内容 | 对应本仓库目录 |
|------|------|----------------|
| `qom/` | 对象模型核心（~2861 行） | architecture/（QOM 文档） |
| `system/` | main loop、runstate、memory 核心（原 softmmu/） | architecture/、memory/ |
| `accel/tcg/` `accel/kvm/` | 加速器：翻译执行循环 | architecture/（KVM）、tcg/ |
| `target/arm/tcg/` | ARM 指令翻译前端 | arm64/tcg/、arm/ |
| `hw/` | 设备模型（machine/virtio/pci/...） | device-model/、arm64/gic/ |
| `block/` `migration/` | 块层、热迁移 | device-model/、architecture/ |
| `monitor/` `qapi/` | HMP/QMP 管理面 | monitor/ |
| `net/` `chardev/` `ui/` | 网络/字符设备/显示 | network/、architecture/ |
| `iothread.c`（树根） | iothread 实现 | architecture/（事件循环） |

**定界三板斧**：

```bash
# 1. 找 QOM 类型定义（一个子系统 = 一组 TYPE_* 宏 + TypeInfo）
grep -rn "TYPE_.*_DEVICE\|TYPE_.*_CPU" hw/arm/ | head
grep -rn "static const TypeInfo" hw/ | grep -i virtio

# 2. 找注册入口（模块自注册机制）
grep -n "type_init\|module_obj" hw/arm/virt.c
#    hw/arm/virt.c:4085  type_init(machvirt_machine_init)

# 3. 找观测点（每目录的 trace-events 是免费的 tracepoint 清单）
cat hw/arm/trace-events accel/tcg/trace-events
```

**文件规模预判**：`wc -l` 核心文件；>3000 行（如 system/memory.c ~3759 行）
需按区域分段读，先锁定热点函数。

### Phase 1: 架构鸟瞰 —— QOM 类型树优先

**内核方法论说"数据结构优先"；QEMU 的对应原则是"QOM 类型树优先"。**

QOM 三层结构（`include/qom/object.h:435` TypeInfo 定义）：

```
TypeInfo（静态描述，注册期）
  ├── name / parent            ← 类型树的边
  ├── instance_size / instance_init / instance_finalize   ← 对象层（每实例一份）
  ├── class_size / class_init / class_base_init           ← 类层（类型一份，vtable）
  └── interfaces               ← 多接口（如 TYPE_USER_CREATABLE）

ObjectClass（类层）──继承链──> Object（实例）
   ↑ 派生类 class_init 里 override 父类方法指针 = C 版虚函数重写
```

分析动作：

1. 画出目标子系统的**类型继承树**（`parent` 字段串起来）
2. 对每个类型列三件事：class_init 覆盖了哪些方法、instance_init 初始化
   什么状态、注册了哪些属性（`object_property_add_*`）
3. 找 **ops 结构体**（DeviceClass/CPUClass/AccelClass/...）——它们是
   QEMU 的策略模式分发表，`grep -rn "\.foo = "` 找所有实现
4. MemoryRegion 拓扑：设备的 `Ops` 回调（read/write）挂在哪棵
   address-space 子树上（Phase 2 追 dispatch 路径用）

**类型树 + ops 分发表 + MemoryRegion 拓扑 = QEMU 的 L1 架构图三件套。**

### Phase 2: 关键路径追踪 —— 按执行上下文选入口

QEMU 没有统一的"入口"，**先问这段逻辑跑在哪个执行上下文**：

```
┌─────────────────────────────────────────────────────────────┐
│ 上下文 A: vCPU 线程（每 vCPU 一个，TCG 或 KVM）              │
│   TCG:  cpu_exec() [accel/tcg/cpu-exec.c:1019]              │
│         └─ tb 查找(softmmu TLB) → 未命中 → 前端解码        │
│            [target/arm/tcg/translate.c] → TCG IR →          │
│            后端codegen [tcg/<arch>/] → 缓存 TranslationBlock │
│   KVM:  kvm_cpu_exec() [accel/kvm/kvm-all.c:3439]           │
│         └─ KVM_RUN ioctl → 处理 exit_reason 分发表           │
├─────────────────────────────────────────────────────────────┤
│ 上下文 B: main loop 线程（持 BQL）                           │
│   qemu_main_loop → main_loop_wait → glib/AioContext poll    │
│   └─ 事件回调: fd handler / BH(qemu_bh_schedule) / 定时器    │
├─────────────────────────────────────────────────────────────┤
│ 上下文 C: iothread 线程（专用 AioContext，不持 BQL）         │
│   └─ dataplane 设备的 I/O 路径（virtio-blk 数据面等）        │
├─────────────────────────────────────────────────────────────┤
│ 上下文 D: coroutine（协程，协作式，属某 AioContext）         │
│   └─ 块层请求处理 / migrate 线程的协作切换                    │
└─────────────────────────────────────────────────────────────┘
```

**追踪规则**：

1. 入口函数确定后，调用链展开 **3-5 层**足够；wrapper/trivial 跳过
2. **函数指针跳转必须标注具体 impl**：`dc->realize` → `virtio_device_realize`
   （这类间接调用是 QEMU 的常态，漏标即断链）
3. 标记三类关键点：**跨上下文切换点**（BQL 获取/释放、`aio_co_wake`、
   BH schedule）、**fast/slow path 分岔**（TLB 命中/未命中）、
   **错误路径**（`error_setg` 之后的清理）
4. guest 触发的路径（如 MMIO 写）写清**从 guest 指令到回调**的完整
   链：guest store → TLB miss/softmmu → `memory_region_dispatch_write`
   [include/system/memory.h] → `mr->ops->write` → 设备回调

### Phase 3: 并发模型分析

QEMU 的并发正确性 = 回答三个问题：**谁持 BQL？谁裸奔？怎么跨线程交接？**

```
1. BQL 边界（bql_lock/bql_unlock，include/qemu/main-loop.h:369）
   ── 哪些路径假定持 BQL（大多数 hw/ 设备模型）？
   ── 哪些路径刻意不持（dataplane、TCG vCPU、KVM run 循环）？

2. vCPU × 设备的竞态面
   ── KVM 循环在 KVM_RUN ioctl 前后释放/重取 BQL（kvm-all.c:3450/:3561），
      MMIO exit 走 kvm_handle_io/address_space_rw 时持 BQL；
      TCG vCPU 执行已翻译代码时不持 BQL → 访问设备状态需
      atomics/rcu 或临时回 BQL

3. 协作式协程
   ── coroutine 内不能阻塞（会卡死整个 AioContext）
   ── qemu_co_queue / aio_co_wake 的交接语义

4. RCU（include/qemu/rcu.h）
   ── 读侧（vCPU 对 mr 拓扑/内存分布的引用）+ synchronize_rcu 回收
```

**锁层次图照画**（谁嵌谁），违反顺序 = 死锁风险，与内核方法论一致。

### Phase 4: 设计取舍与演化

QEMU 无并行版本树，**演化考古用本仓库 git**：

```bash
git log -S "bql_lock" --oneline            # 符号何时引入/改名
git log -L :cpu_exec:accel/tcg/cpu-exec.c  # 函数演化史
git log --grep "virtio" --since=2024.01    # 主题检索
```

对每个"不显然"的设计，问"为什么不……"，从 commit message 找证据，
按 决策/背景/选项/结论/代价 五段记录。

---

## 三、QEMU 模式速查库

### 3.1 QOM 三件套（一切皆对象）

| 件 | 位置 | 分析要点 |
|----|------|----------|
| TypeInfo | include/qom/object.h:435 | 类型树边 + init/finalize 生命周期 |
| ObjectClass | qom/object.c | vtable；class_init 是"重写虚函数"的地方 |
| 属性/接口 | object_property_add_* / interfaces | 动态属性 = 管理面（QMP 可见）；接口 = 横切能力 |

### 3.2 设备模型三件套 + VMState

```
一个典型设备 = QOM 类型（hw/xxx）
             + MemoryRegion（一到多个，挂 address-space，Ops.read/write）
             + qemu_irq（电平/边沿，经 GPIO/IRQ 总线连 GIC 等）
             + VMStateDescription（include/migration/vmstate.h:227，
               迁移时的状态序列化清单 —— 看它就知道哪些状态是"活"的）
```

**VMStateDescription 是设备分析的捷径**：它列出全部需要跨迁移存活的
状态字段，等于设备的"有效状态清单"。

### 3.3 TCG 双阶段 + TB 缓存 + softmmu

- 前端：`target/arm/tcg/translate.c`（guest 指令 → TCG IR）
- 后端：`tcg/<host>/`（TCG IR → host 指令）
- 缓存：TranslationBlock（include/qemu/typedefs.h），pc/flags 哈希查找
- softmmu TLB：guest 虚地址 → MR 快速分发（加速 guest 访存）
- MTTCG：多 vCPU 并行翻译执行的同步（tcg/tcg/mttcg 相关文档）

### 3.4 事件循环 / AioContext / coroutine / BH

| 概念 | 定义处 | 一句话 |
|------|--------|--------|
| AioContext | include/qemu/aio.h:201 | 事件循环实例（fd + BH + 定时器 + 协程） |
| main loop | system/main.c | 全局默认循环，持 BQL |
| iothread | iothread.c（树根） | 专用线程 + 专用 AioContext（dataplane） |
| coroutine | include/qemu/coroutine*.h | 协作式用户态线程，块层主力 |
| BH | aio_bh_new/schedule | "软中断"类比，跨线程踢事件循环 |

### 3.5 trace-events 观测面

每目录一份 `trace-events`（如 `hw/arm/trace-events`、
`accel/tcg/trace-events`），tracetool 生成代码。**分析任何子系统前
先 cat 它的 trace-events**——事件名即关键路径节点清单，动态实测时
（见 practice/102）直接开这些点。

---

## 四、工具链

```
语义查询: clangd（树根 compile_commands.json 已存在）> cscope > readtags > grep
符号速查: grep TYPE_*/TypeInfo/trace-events（QOM 风格 grep 命中率高）
历史考古: git log -S / -L / --grep（本仓库即上游 qemu 历史）
规范对照: ARM DDI 0487（ARMv8-A）、GIC IHI 0069 —— spec-verify/ 的对照法
动态实测: GDB stub / tracing / perf → practice/INDEX.md
```

| 需求 | 命令/工具 |
|------|-----------|
| "这个设备类型定义在哪?" | `grep -rn "TYPE_FOO" hw/ include/` |
| "TypeInfo 注册了什么?" | `grep -n "static const TypeInfo" <file>` + 读 class_init |
| "ops 的所有实现?" | `grep -rn "\.realize = \|\.read = " hw/` |
| "guest 访存怎么到设备?" | 从 `memory_region_dispatch_read` [include/system/memory.h:2617] 逆/顺推 |
| "这行为什么存在?" | `git log -S "该表达式" -- <file>` |

---

## 五、本仓库文档规范

### 5.1 编号与索引（强约定）

- 文档**全局唯一编号**（00-113+，跨目录不重复）；补充专项用 100+ 段
- 每目录 `INDEX.md` 登记；顶层 `README.md` 维护统计与阅读路线
- 新文档落盘 checklist：

```
□ 编号未占用（grep 编号于 README/各 INDEX）
□ 头部信息完整（版本 11.0.50 / 分析日期 / 核心源码 / 关联文档）
□ 所在目录 INDEX.md 添加条目
□ README.md 对应表格 + 篇数统计更新
□ commit: doc(scope): 中文摘要   （scope ∈ 目录名）
□ push 到 origin (ai-qemu.git)
```

### 5.2 双头部格式

**分析类**（architecture/tcg/memory/...）：

```markdown
# 中文标题 — 副标题 (QEMU 11.0.50)

> **版本**: QEMU 11.0.50
> **分析日期**: YYYY-MM-DD
> **重点架构**: ...
> **核心源码**: ...
> **关联文档**: ...

---
## 目录
## 1. 编号章节...
```

**实践类**（practice/）：

```markdown
# Doc NNN: 标题

## 文档信息
- 组件: ...
- 源码版本: QEMU 11.0.50
- 分析日期: ...
- 归档目录: practice/

## 目录
...
```

### 5.3 引用与图表

- 源码引用：`[path:line]` 或反引号路径（**落盘前对照源码复核**；
  注意头文件重组，见附录 A——旧资料路径 ≠ 当前路径）
- 表格 markdown；架构图/调用链/状态机用 ASCII box drawing
- 结论前置；中文描述 + 英文术语；避免翻译腔

---

## 六、一页纸总结

```
┌────────────────────────────────────────────────────────────┐
│              QEMU 源码分析方法论精要                          │
├────────────────────────────────────────────────────────────┤
│ 1. 定位   grep TYPE_*/type_init/trace-events 三板斧         │
│ 2. 鸟瞰   QOM 类型树 + ops 分发表 + MemoryRegion 拓扑        │
│ 3. 追踪   先定执行上下文（vCPU/main loop/iothread/coroutine）│
│           调用链 3-5 层，函数指针标注 impl，标记跨上下文切换点 │
│ 4. 并发   BQL 边界 + vCPU 竞态面 + 协程不阻塞 + RCU 回收      │
│ 5. 取舍   git log 考古 + 规范(DDI/IHI)对照                   │
│ 6. 写作   双头部格式 + [path:line] 复核 + INDEX/README 更新   │
│                                                            │
│ 工具: clangd > cscope > grep;  git log -S/-L                │
│ 产出: 让读者"不看源码就理解设计"的文档                        │
└────────────────────────────────────────────────────────────┘
```

---

## 附录 A: QEMU 10/11 头文件重组对照

本树（11.0.50）经历了大规模路径重组，**旧文档/旧 commit/网络资料中的
路径需按下表换算**（对照分析法时尤其注意）：

| 旧路径/符号（≤QEMU 9） | 新路径/符号（本树 11.0.50） |
|------------------------|------------------------------|
| `include/exec/memory.h` | `include/system/memory.h` |
| `memory.c` | `system/memory.c` |
| `include/block/aio.h` | `include/qemu/aio.h` |
| `include/hw/qdev-core.h` | `include/hw/core/qdev.h` |
| `softmmu/`（vl.c 等） | `system/`（main.c 等） |
| `qemu_mutex_lock_iothread()` | `bql_lock()`（include/qemu/main-loop.h:369） |

验证方法：`git log --follow -1 -- <新路径>` 可看到 rename 历史。

## 附录 B: 动态分析方法索引

静态分析（本文）配动态实测，方法见 practice/ 8 篇：

| Doc | 主题 |
|-----|------|
| 102 | 监测与追踪（trace events / simple、ftrace 后端） |
| 103 | 调试系统（GDB stub / core dump） |
| 104/105 | QEMU 基础/高级用法 |
| 106 | GDB 远程调试完全指南（RSP 协议） |
| 110 | GDB 调 Linux 内核工作流（与内核树实践联动） |
| 111 | 性能分析与 Profiling（perf 集成） |
| 112 | 开发者指南（添加设备/指令的实操） |

---

*本文为方法论文档，不占用文档全局编号；修订遵循 `doc(methodology):` 提交风格。*
