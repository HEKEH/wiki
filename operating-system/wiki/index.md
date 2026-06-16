# 操作系统 — 索引

所有 wiki 页面的目录，按类别组织。每次 ingest 时更新。

## 概览

- [[home]] — 顶层综合与导航

## 概念

- [[concepts/mode-bit-and-operational-modes]] — 区分内核态与用户态的单比特寄存器；为特权指令把关。
- [[concepts/interrupts]] — 暂停进程、跳转到处理程序、并把模式位翻转为内核态的信号。
- [[concepts/system-calls]] — 操作系统库函数，通过软件中断安全进入内核态；含优缺点。
- [[concepts/syscall-abi]] — 内核如何得知用户程序的意图：系统调用号 + 寄存器传参 + 系统调用表分发 + 用户指针校验。
- [[concepts/memory-management-unit]] — 限制 CPU 内存访问的硬件；只能通过特权指令操控。
- [[concepts/page-fault]] — MMU 在地址翻译时触发的同步异常；按需调页的基础，也是 MMU 把非法访问转为异常的机制。
- [[concepts/preemptive-vs-cooperative-os]] — 定时器中断让操作系统能强制收回 CPU；协作式为何被淘汰。
- [[concepts/kernel-mode-drivers]] — 驱动/反作弊/杀软为何运行在内核态，以及共享地址空间的风险（蓝屏）。
- [[concepts/interprocess-communication]] — 进程为何/如何协作；共享内存 vs 消息传递两种模型对比。
- [[concepts/shared-memory-ipc]] — 进程共享一块内存区域；建立后免系统调用、极快，但数据格式与同步需自负。
- [[concepts/message-passing-ipc]] — 经内核邮箱/端口收发消息；地址空间隔离、可跨机器（套接字/RPC），但每次收发都要系统调用。
- [[concepts/concurrency]] — 单 CPU 快速轮换执行单位的错觉；两个目的（多任务 + 填满 IO 空隙）、阻塞效应；单核也成立。
- [[concepts/threads]] — 进程内可独立调度的最基本执行单位；共享地址空间、独占 CPU 状态与栈；线程不是函数。

## 实体

- [[entities/crowdstrike]] — 内核态杀软，一次损坏的更新拖垮了全球操作系统。
- [[entities/vanguard]] — Riot Games 的内核态反作弊；隐私担忧。
- [[entities/tencent]] — Riot Games 的中国母公司；Vanguard 隐私担忧的根源。

## 分析

- [[analysis/detecting-kernel-corruption]] — 操作系统为何基本无法检测内核态对合法数据的破坏，以及页保护、KASAN、PatchGuard、VBS/HVCI 等尽力而为的补救机制。
- [[analysis/multiprocess-vs-multithreaded-server]] — 头像服务器例子：阻塞效应，以及多进程 vs 多线程两种并发方案的内存/速度/隔离权衡。

## 来源

- [[sources/single-bit-mode-bit-os-integrity]] — Core Dumped 视频（已清洗）：模式位、特权指令、中断、系统调用、定时器与内核态驱动如何让操作系统保持掌控。
- [[sources/ipc-shared-memory-vs-message-passing]] — Core Dumped 视频（已清洗）：进程间通信的两种模型——共享内存与消息传递（含 Chromium、Mach 端口、套接字、性能权衡）。
- [[sources/why-threads-single-core]] — Core Dumped 视频（已清洗）：并发的两个目的、阻塞效应、线程如何在单进程内实现并发，以及 Linux 的 task 抽象。
