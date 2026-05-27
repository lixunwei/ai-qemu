# TCG 翻译引擎 (Tiny Code Generator)

TCG 核心子系统：IR 设计、前端翻译、优化 Pass、后端代码生成、多线程、TLB、Plugin。

| # | 文档 | 主题 |
|---|------|------|
| 00 | [TCG深度分析](00-TCG深度分析.md) | TCG 整体架构概览 |
| 01 | [TCG优化递次](01-TCG优化递次深度分析.md) | 优化 Pass 层次与执行顺序 |
| 02 | [TCG-IR设计与前端翻译](02-TCG-IR设计与前端翻译深度分析.md) | IR 操作码设计、前端 API |
| 03 | [TCG后端代码生成与TB管理](03-TCG后端代码生成与TB管理深度分析.md) | 后端发射、TB 缓存管理 |
| 04 | [MTTCG多线程翻译](04-MTTCG多线程翻译深度分析.md) | 多线程并行翻译与执行 |
| 05 | [Softmmu-TLB与内存访问](05-Softmmu-TLB与内存访问深度分析.md) | TLB 快慢路径、内存访问模拟 |
| 06 | [TCG-Plugin系统](06-TCG-Plugin系统深度分析.md) | Plugin API、回调注入 |
| 07 | [Linux-user用户模式翻译](07-Linux-user用户模式翻译深度分析.md) | 用户态翻译、系统调用转换 |
| 08 | [TCG加速器综合导航](08-TCG加速器子系统综合导航-IR-前端-后端-优化-MTTCG-TLB-Plugin-Linux-user.md) | TCG 子系统交叉索引 |
| 08b | [TCG后端(IR/寄存器/缓存)](08-TCG后端深度分析-IR生成寄存器分配与代码缓存.md) | IR 生成、寄存器分配、代码缓存 |
| 09 | [TCG深入(优化/向量/TLB)](09-TCG深入分析-优化遍向量指令与Softmmu-TLB机制.md) | 优化遍、向量指令、TLB 机制 |
| 12 | [多线程TCG(MTTCG)](12-多线程TCG深度分析-MTTCG并行执行TB失效与内存屏障.md) | 并行执行、TB 失效、内存屏障 |
| 13 | [TCG前端翻译(解码/IR/优化)](13-TCG前端翻译深度分析-指令解码IR生成与优化Pass.md) | 指令解码、IR 生成、优化 Pass |
| 14 | [TCG后端(AArch64/寄存器/TLB)](14-TCG后端代码生成深度分析-AArch64后端寄存器分配与TLB慢路径.md) | AArch64 后端、寄存器分配、TLB 慢路径 |
| 78 | [TCG前端IR生成与优化Pass](78-TCG前端IR生成与优化Pass深度分析-操作码体系-常量折叠-拷贝传播-活性分析-寄存器分配.md) | 操作码体系、常量折叠、拷贝传播、活性分析、寄存器分配 |
| 107 | [TCG与真实JIT编译器对比](107-TCG与真实JIT编译器对比分析-寄存器分配-优化Pass-代码缓存-V8-LLVM-HotSpot-LuaJIT.md) | TCG vs V8/LLVM/HotSpot/LuaJIT: IR设计、优化深度、寄存器分配、代码缓存、投机优化、分层编译 |
