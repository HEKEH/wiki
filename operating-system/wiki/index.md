# 操作系统 — 索引

所有 wiki 页面的目录，按类别组织。每次 ingest 时更新。

## 概览

- [[home]] — 顶层综合与导航

## 概念

- [[concepts/mode-bit-and-operational-modes]] — 区分内核态与用户态的单比特寄存器；为特权指令把关。
- [[concepts/interrupts]] — 暂停进程、跳转到处理程序、并把模式位翻转为内核态的信号。
- [[concepts/context-switch]] — 保存被中断进程、载入下一个进程的寄存器状态；PC 被中断覆写使纯软件无法保存，需多寄存器组/TSS 等硬件支持。
- [[concepts/system-calls]] — 操作系统库函数，通过软件中断安全进入内核态；含优缺点。
- [[concepts/syscall-abi]] — 内核如何得知用户程序的意图：系统调用号 + 寄存器传参 + 系统调用表分发 + 用户指针校验。
- [[concepts/application-binary-interface]] — 二进制层约定（寄存器用法、传参方式、指针宽度、文件格式）；API 定义源码层、ABI 定义二进制层。
- [[concepts/executable-file-format]] — 可执行文件不只含指令还含数据/元数据；OS 据格式加载；ELF（Linux）vs PE（Windows）。
- [[concepts/memory-management-unit]] — 限制 CPU 内存访问的硬件；只能通过特权指令操控。
- [[concepts/page-fault]] — MMU 在地址翻译时触发的同步异常；按需调页的基础，也是 MMU 把非法访问转为异常的机制。
- [[concepts/preemptive-vs-cooperative-os]] — 定时器中断让操作系统能强制收回 CPU；协作式为何被淘汰。
- [[concepts/kernel-mode-drivers]] — 驱动/反作弊/杀软为何运行在内核态，以及共享地址空间的风险（蓝屏）。
- [[concepts/interprocess-communication]] — 进程为何/如何协作；共享内存 vs 消息传递两种模型对比。
- [[concepts/shared-memory-ipc]] — 进程共享一块内存区域；建立后免系统调用、极快，但数据格式与同步需自负。
- [[concepts/message-passing-ipc]] — 经内核邮箱/端口收发消息；地址空间隔离、可跨机器（套接字/RPC），但每次收发都要系统调用。
- [[concepts/concurrency]] — 单 CPU 快速轮换执行单位的错觉；两个目的（多任务 + 填满 IO 空隙）、阻塞效应；单核也成立。
- [[concepts/threads]] — 进程内可独立调度的最基本执行单位；共享地址空间、独占 CPU 状态与栈；线程不是函数。
- [[concepts/parallelism]] — 多核上真正同时执行；并发 vs 并行、最多 n 个线程并行、声明并发即可移植、数据并行 vs 任务并行。
- [[concepts/cpu-scheduling]] — CPU 调度总览：五大标准（利用率/吞吐量/周转/等待/响应）、抢占 vs 非抢占、饥饿与老化。
- [[concepts/cpu-io-bursts]] — CPU 突发/IO 突发交替模型；突发时长分布、CPU 密集型 vs IO 密集型、护航效应。
- [[concepts/process-states]] — 新建/就绪/运行/等待/终止五态与 PCB；队列为何是调度的核心数据结构。
- [[concepts/dispatcher]] — 调度器（决策）vs 分派器（执行上下文切换）；分派延迟、甘特图。
- [[concepts/scheduling-algorithms]] — 六大算法演进：FCFS → SJF（指数平均预测）→ 轮转 → 优先级（老化）→ 多级队列 → 多级反馈队列。
- [[concepts/race-conditions]] — 共享数据并发的核心缺陷：操作非原子 + 抢占时机不可预测；单写多读脏读、多写丢失更新、普通变量当锁为何失败。
- [[concepts/memory-protection]] — 进程间地址空间隔离：基址+界限寄存器在地址总线拦截每次访存、门控读写使能、越界触发段错误；纯软件为何不可行、上下文切换要更新边界。

## 实体

- [[entities/crowdstrike]] — 内核态杀软，一次损坏的更新拖垮了全球操作系统。
- [[entities/vanguard]] — Riot Games 的内核态反作弊；隐私担忧。
- [[entities/tencent]] — Riot Games 的中国母公司；Vanguard 隐私担忧的根源。

## 分析

- [[analysis/detecting-kernel-corruption]] — 操作系统为何基本无法检测内核态对合法数据的破坏，以及页保护、KASAN、PatchGuard、VBS/HVCI 等尽力而为的补救机制。
- [[analysis/multiprocess-vs-multithreaded-server]] — 头像服务器例子：阻塞效应，以及多进程 vs 多线程两种并发方案的内存/速度/隔离权衡。
- [[analysis/why-applications-are-os-specific]] — 应用为何不仅与架构、也与操作系统绑定：系统调用集合 + ABI + 可执行文件格式 + 运行时四层不兼容。

## 来源

- [[sources/single-bit-mode-bit-os-integrity]] — Core Dumped 视频（已清洗）：模式位、特权指令、中断、系统调用、定时器与内核态驱动如何让操作系统保持掌控。
- [[sources/ipc-shared-memory-vs-message-passing]] — Core Dumped 视频（已清洗）：进程间通信的两种模型——共享内存与消息传递（含 Chromium、Mach 端口、套接字、性能权衡）。
- [[sources/why-threads-single-core]] — Core Dumped 视频（已清洗）：并发的两个目的、阻塞效应、线程如何在单进程内实现并发，以及 Linux 的 task 抽象。
- [[sources/threads-on-multicore]] — Core Dumped 视频（已清洗）：并发的极限与三种解法、多核术语、并发 vs 并行、最多 n 个线程并行、数据并行 vs 任务并行。
- [[sources/why-apps-are-os-specific]] — Core Dumped 视频（已清洗）：应用为何与操作系统绑定——系统调用集合（fork/exec vs CreateProcess）、ABI、可执行文件格式（ELF/PE）、运行时。
- [[sources/hardware-assists-context-switch]] — Core Dumped 视频（已清洗）：上下文切换为何纯软件做不到（PC 被中断覆写），以及多寄存器组、硬件自动压栈、TSS 三种硬件支持。
- [[sources/cpu-scheduling-algorithms]] — Core Dumped 视频（已清洗）：CPU 调度从 FCFS 到多级反馈队列的完整演进，含 CPU/IO 突发、进程状态、调度器/分派器、各项调度标准与饥饿/老化。
- [[sources/race-conditions]] — Core Dumped 视频（已清洗）：竞态条件——操作非原子、抢占带来的不可预测时机、共享字符串脏读与质数计数器丢失更新、为何普通标志位与 Peterson 软件方案不可靠。
- [[sources/why-programs-cant-access-memory]] — Core Dumped 视频（已清洗）：程序为何无法访问彼此内存——地址空间神圣性、纯软件为何开销过大、基址/界限寄存器 + 地址总线拦截 + 比较器/AND 门 + 门控读写使能 + 段错误，以及上下文切换/特权寄存器的角色（预告分页）。
