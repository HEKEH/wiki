# 操作系统 — 活动日志

按时间顺序、仅追加的 wiki 活动记录。

## [2026-06-19] query | 多进程也能并行吗？不同进程用不同核

确认：进程当然能并行——并行的单位是任何可被调度的执行实体（Linux 里进程/线程都是 `task`、同一
调度器派发），进程间并行早于多线程普及。真正的区别是**进程间并行 vs 进程内并行**：线程的独到之
处不是"能并行"，而是"让同一进程内部也能并行"且代价更低（共享地址空间，省内存/创建快，代价是同步）。
在 [[concepts/parallelism]] 增补"并行的单位：进程也能并行吗？"一节，回链
[[analysis/multiprocess-vs-multithreaded-server]] 与 [[concepts/interprocess-communication]]。

## [2026-06-19] ingest | Core Dumped —《多核系统中的线程》

将原始 `.srt` 文字稿清洗为 [[sources/threads-on-multicore]]：删除时间戳、Brilliant / CodeCrafters
广告口播和口水话，修正语音识别错误（CodeCrafters、git/SQLite、the illusion 等）。原始来源
保持不变。第四次源 ingest，是《为什么单核处理器也需要线程》的续集。

新建 1 个概念页 [[concepts/parallelism]]（并发 vs 并行对比表、为何需要更多核心与三种解法、
最多 n 个线程并行、声明并发即可移植、数据并行 vs 任务并行、加速比 < 核心数）。从
[[concepts/threads]]（多核额外用途）与 [[concepts/concurrency]]（真正同时需多核）回链，并更新
index.md 与 home.md。

## [2026-06-16] ingest | Core Dumped —《为什么单核处理器也需要线程》

将原始 `.srt` 文字稿清洗为 [[sources/why-threads-single-core]]：删除时间戳、Brilliant 广告口播
和口水话，修正语音识别错误（scheduler、cooperate、pointer 等）。原始来源保持不变。第三次源 ingest。

拆解为 2 个概念页——[[concepts/concurrency]]（并发两个目的、阻塞效应、单核也成立、对比并行）、
[[concepts/threads]]（线程为何存在、共享/独占、线程不是函数、PCB 与 Linux task、主线程、动态创建）——
以及 1 个分析页 [[analysis/multiprocess-vs-multithreaded-server]]（头像服务器例子的两种并发方案
权衡）。从 [[concepts/interprocess-communication]] 回链，并更新 index.md 与 home.md。

## [2026-06-16] query | 邮箱和端口是一个东西吗？

回答后在 [[concepts/message-passing-ipc]] 增补"术语辨析：邮箱 / 端口"一节：同义层(邮箱=Mach
端口，仅名字不同)；Mach 端口额外带端口权限(receive/send rights)能力模型；套接字"端口"是 16 位
编号(服务复用)而非队列对象本身。标注了源材料 vs 外部工程知识。

## [2026-06-16] ingest | Core Dumped —《IPC：共享内存还是消息传递》

将原始 `.srt` 文字稿清洗为 [[sources/ipc-shared-memory-vs-message-passing]]：删除了时间戳、
JetBrains / CodeCrafters 广告口播和口水话；修正了语音识别错误（Mach OS、queue、kernel、
CodeCrafters、Rider）。原始来源保持不变。这是本知识库的第二次源 ingest。

将来源拆解为 3 个概念页——[[concepts/interprocess-communication]]（总览 + 两模型对比表）、
[[concepts/shared-memory-ipc]]（地址空间共享、生产者消费者风险、竞态条件、Chromium 多进程）、
[[concepts/message-passing-ipc]]（内核邮箱/send-receive、Mach 端口、套接字与客户端服务器、
系统调用开销）。从 [[concepts/system-calls]] 回链，并更新 index.md 与 home.md。

## [2026-06-16] ingest | Core Dumped —《处理器中的一个比特如何守护操作系统的完整性》

将原始 `.srt` 文字稿清洗为 [[sources/single-bit-mode-bit-os-integrity]]：删除了时间戳、
NeetCode 广告口播和口水话；修正了语音识别错误（NeetCode、`syscall`/`sysret`、Tencent、
MMU）。原始来源保持不变。这是本知识库的首次 ingest——在 CLAUDE.md 中设定了领域和类别。

将来源拆解为 6 个概念页——[[concepts/mode-bit-and-operational-modes]]、
[[concepts/interrupts]]、[[concepts/system-calls]]、[[concepts/memory-management-unit]]、
[[concepts/preemptive-vs-cooperative-os]]、[[concepts/kernel-mode-drivers]]——以及 3 个
实体页——[[entities/crowdstrike]]、[[entities/vanguard]]、[[entities/tencent]]。交叉链接了
所有页面，并更新了 index.md 与 home.md。

## [2026-06-16] lint | 全库改写为中文

将所有 wiki 页面（来源、概念、实体、home、index、log）从英文改写为中文，保留文件名/
wikilink 别名不变。技术术语保留英文（mode bit、MMU、`syscall`/`sysret`、BSOD 等）并附中文。

## [2026-06-16] query | 操作系统如何检测内核态代码破坏关键数据？

回答后归档为新分析页 [[analysis/detecting-kernel-corruption]]：核心结论是同级别基本无法
可靠检测（MMU 只能拦非法访问、拦不住对合法地址的覆盖），并梳理了页保护/W^X/SMEP-SMAP、
KASAN/Driver Verifier 红区、PatchGuard、VBS/HVCI 等补救机制。页面标注了源材料 vs 外部
工程知识。从 [[concepts/kernel-mode-drivers]] 与 [[concepts/memory-management-unit]] 回链，
并更新 index.md。

## [2026-06-16] query | 什么是缺页异常，如何检测到缺页异常

回答后归档为新概念页 [[concepts/page-fault]]：缺页异常是 MMU 在地址翻译时触发的同步异常
（present 位/权限位检查），经 IDT 进入内核处理；区分良性（按需调页/COW/栈增长）与非法
（SIGSEGV / panic / 蓝屏）。从 [[concepts/memory-management-unit]] 与
[[analysis/detecting-kernel-corruption]] 回链，并更新 index.md。页面标注了源材料 vs 外部
工程知识。

## [2026-06-16] query | 用户程序中断后，内核如何知道要执行哪个操作（如写硬盘）

回答后归档为新概念页 [[concepts/syscall-abi]]：系统调用号 + 寄存器传参（x86-64 ABI）+ 以调用号
查 `sys_call_table` 函数指针表分发 + `copy_from_user` 对用户指针做边界校验。补全了
[[concepts/system-calls]] 缺失的"内核如何得知意图"一环；从 system-calls 回链，并更新 index.md。
页面标注了源材料 vs 外部工程知识。
