# Home

本知识库的顶层综合与导航。

## 关于

本 wiki 由 LLM 代理增量构建和维护。原始来源不可变；wiki 是 LLM 的图层——摘要、实体页、
概念页、交叉引用以及不断演进的综合。

## 快速链接

- [[index]] — 所有 wiki 页面的目录
- [[log]] — 按时间顺序的活动记录

## 当前状态

焦点：硬件与软件如何协作，让操作系统内核保持掌控。

核心洞见（来自 [[sources/single-bit-mode-bit-os-integrity]]）：CPU 中的一个**模式位**把
**内核态**（可执行特权指令）与**用户态**（不可执行）区分开。用户程序只能通过**中断**切入
内核态——这正是**系统调用**的工作方式（软件中断把控制权交给操作系统）。一个硬件**定时器
中断**让操作系统成为*抢占式*，使失控的循环无法饿死系统。驱动以及部分第三方软件（反作弊、
恶意软件检测）也运行在内核态，共享同一个地址空间——强大但危险（见 [[entities/crowdstrike]]、
[[entities/vanguard]]）。

**概念图：** [[concepts/mode-bit-and-operational-modes]] →
[[concepts/interrupts]] → [[concepts/system-calls]]；
[[concepts/memory-management-unit]]；[[concepts/preemptive-vs-cooperative-os]]；
[[concepts/kernel-mode-drivers]]。

第二条线索（来自 [[sources/ipc-shared-memory-vs-message-passing]]）：进程默认相互隔离，
协作需要 **IPC**。[[concepts/shared-memory-ipc]] 让进程共享一块内存区域——建立后免系统调用、
极快，但数据格式与同步要进程自负；[[concepts/message-passing-ipc]] 经内核邮箱/端口收发消息——
地址空间保持隔离、甚至可跨机器（套接字/RPC），代价是每次收发都要系统调用。两条线索都落在
[[concepts/system-calls]] 这个进出内核的关口上。

第三条线索（来自 [[sources/why-threads-single-core]]）：从"进程"下探到**执行单位**本身。
[[concepts/concurrency]] 让单 CPU 快速轮换任务，既做多任务、也**填满进程等 IO 时的空隙**
（单核亦然）。但单进程只有一个程序计数器，无法内部并发；解法是 [[concepts/threads]]——给每个
内部执行实体各一个 PC 与 CPU 状态，共享进程地址空间。线程比"每客户端一进程"更省内存、创建更快
（见 [[analysis/multiprocess-vs-multithreaded-server]]），代价是共享地址空间带来的隔离/同步问题。
Linux 用统一的 `task` 结构同时表示进程与线程。

续集（来自 [[sources/threads-on-multicore]]）把这条线索推到**多核**：并发有极限(任务太多时
重获 CPU 的间隔变长)，蛮力解法是加核心。多核让并发升级为 [[concepts/parallelism]]——线程被
分派到独立核心上**真正同时运行**。并发≠并行(并发让所有任务有进展，并行让任务真正同时；单核
只有并发)，这正是现代 OS 以线程为基本执行单位的根本原因。n 核 → **最多** n 个线程并行(还要与
其他进程竞争)；程序员只需声明并发，可移植性交给 OS。并行分**数据并行**(同操作、分数据)与
**任务并行**(分操作、同/异数据)，且加速比通常小于核心数。

第四条线索（来自 [[sources/why-apps-are-os-specific]]）：从 [[concepts/system-calls]] 这个关口
引出一个更大的问题——**为何应用程序不仅与架构、也与操作系统绑定**（综合见
[[analysis/why-applications-are-os-specific]]）。即便同一 CPU，四层约定任何一层不对齐都会失败：
①每个 OS 有**自己的系统调用集合**（连语义都不同，如 `fork`+`exec` vs `CreateProcess`）；
②[[concepts/application-binary-interface]]——调用号所在寄存器、传参方式、指针宽度等二进制约定
（[[concepts/syscall-abi]] 是其系统调用实例）；③[[concepts/executable-file-format]]（ELF vs PE）；
④运行时依赖。可移植性这条线把 system-calls 的"平台依赖"缺点展开成了完整图景。

## 开放问题

- 真实架构（x86 的 0–3 环、ARM 的异常等级）如何映射到这个简化的模式位模型？
- 有哪些现代缓解措施能缩小有 bug 的内核态代码的"爆炸半径"（例如 eBPF 沙箱、用户态驱动、
  Windows 在 CrowdStrike 事件后的改动）？
