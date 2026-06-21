# 操作系统 — 活动日志

按时间顺序、仅追加的 wiki 活动记录。

## [2026-06-19] query | 协作式 OS 怎么解决 PC/SP 丢失问题

核心区分：PC/SP 丢失是**异步中断(抢占)独有**的问题，协作式是**自愿/同步**让出——`call` 由调用约定
同步保存返回地址(PC)、yield 例程在交出控制权前用普通指令存 SP/寄存器，故此难题自始不存在；极简
无中断 CPU 更是没有异步中断。附带差别：协作式切点在函数边界、按 ABI 只需存 callee-saved 寄存器，
比抢占式存整套状态更轻，代价是依赖程序诚实让出。在 [[concepts/context-switch]] 增补"为何协作式
切换不受此难题困扰"一节。

## [2026-06-19] ingest | Core Dumped —《硬件如何在多任务时协助软件》

将原始 `.srt` 文字稿清洗为 [[sources/hardware-assists-context-switch]]：删除时间戳、Skillshare /
CodeCrafters 广告口播和口水话，修正语音识别错误（MOS 6502、SIMD 寄存器；"cisr" 疑为识别误差，
指 `iret` 类中断返回指令）。原始来源保持不变。第六次源 ingest。

新建 1 个概念页 [[concepts/context-switch]]（定义、为何必须做、纯软件做不到的根因=PC 被中断覆写、
两类硬件支持：多寄存器组 / 硬件自动压栈 / TSS 全硬件）。深化了 [[concepts/interrupts]] 的"保存/恢复
状态"两步并互链；从 [[concepts/preemptive-vs-cooperative-os]]（定时器触发切换 / 协作式纯软件）与
[[concepts/threads]]（状态存入 PCB/`task`）回链。更新 index.md 与 home.md（新增第五条线索）。

## [2026-06-19] ingest | Core Dumped —《为什么应用程序与操作系统绑定》

将原始 `.srt` 文字稿清洗为 [[sources/why-apps-are-os-specific]]：删除时间戳、Brilliant 广告口播
和口水话，修正语音识别错误（`syscall`、x86-64、系统调用表、`CreateProcess`、Macs 等）。原始来源
保持不变。第五次源 ingest。

新建 1 个分析页 [[analysis/why-applications-are-os-specific]]（四层不兼容：系统调用集合 / ABI /
可执行文件格式 / 运行时）与 2 个概念页——[[concepts/application-binary-interface]]（API vs ABI、
寄存器/传参/指针宽度约定）、[[concepts/executable-file-format]]（指令+数据+元数据、ELF vs PE）。
因本视频明确讲了"调用号+寄存器+系统调用表"总体机制，给 [[concepts/syscall-abi]] 补充了第二个来源
并放宽了范围说明；在 [[concepts/system-calls]] 的"平台依赖"缺点里展开 fork/exec vs CreateProcess
并回链。更新 index.md 与 home.md（新增第四条线索）。

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

## [2026-06-20] ingest | CPU 调度（让电脑更流畅的精妙算法）

来源 [[sources/cpu-scheduling-algorithms]]（Core Dumped 视频，已清洗）。新建 1 个来源页 + 5 个
概念页：[[concepts/cpu-scheduling]]（标准/抢占/饥饿总览）、[[concepts/cpu-io-bursts]]（CPU/IO 突发、
CPU 密集型 vs IO 密集型、护航效应）、[[concepts/process-states]]（五态 + PCB + 队列）、
[[concepts/dispatcher]]（调度器 vs 分派器、分派延迟）、[[concepts/scheduling-algorithms]]
（FCFS → SJF + 指数平均 → 轮转 → 优先级 + 老化 → 多级队列 → 多级反馈队列）。从
[[concepts/threads]]、[[concepts/context-switch]]、[[concepts/preemptive-vs-cooperative-os]]
回链，并更新 index.md 与 home.md（新增第六条线索 + 一条开放问题：现代真实调度器 CFS/EEVDF 与 IO 调度器）。
