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

第五条线索（来自 [[sources/hardware-assists-context-switch]]）：回到 [[concepts/interrupts]] 的
"保存/恢复状态"两步，把它落实为 [[concepts/context-switch]]——保存被中断进程的寄存器、载入下一个
进程的寄存器。核心洞见是这一步**无法仅靠软件完成**：中断会在任何处理指令执行前就**覆写 PC**，没有
指令能抢先保住它。于是这是个罕见的"**硬件支持软件**"情形——架构用**多寄存器组**(普通组/特权组随模式
切换)、**硬件自动压入 PC/SP**、或 **TSS 全硬件切换**来兜底。被保存的状态最终落进进程/线程的
`task`(PCB)，与 [[concepts/threads]]、[[concepts/preemptive-vs-cooperative-os]] 的定时器中断闭环。

第六条线索（来自 [[sources/cpu-scheduling-algorithms]]）：定时器中断让 OS 能抢占之后，真正的问题是
**该把 CPU 给谁**——即 [[concepts/cpu-scheduling]]。根本依据是 [[concepts/cpu-io-bursts]]：进程的
执行是 CPU 突发与 IO 突发交替，且**短突发占绝大多数**，故调度的对象是"下一个 CPU 突发"而非整个
进程，目标是趁某进程等 IO 时把 CPU 让给别人、并优先保证**响应时间**。进程在
[[concepts/process-states]]（就绪/运行/等待…）间循环，除运行态外都在队列里等，决策由调度器做、
执行（上下文切换）由 [[concepts/dispatcher]] 做。[[concepts/scheduling-algorithms]] 是一条"每个修复
前一个缺陷"的演进：FCFS(护航效应) → SJF(最优但需指数平均预测、且会饥饿) → 轮转(抢占+时间片，回到
[[concepts/preemptive-vs-cooperative-os]] 的定时器) → 优先级(+老化防饥饿) → 多级队列 → 多级反馈
队列(据行为自动升降级)。这正解释了引言的悖论：CPU 满载时桌面仍流畅——交互/IO 密集型进程突发短、
长留高优先级队列，CPU 重活用剩余周期。最后回扣 [[concepts/threads]]：现代系统调度的是线程而非进程。

第七条线索（来自 [[sources/race-conditions]]）：抢占（第六条的定时器中断）与线程共享地址空间
（第三条）合在一起，暴露出并发的**阴暗面**——[[concepts/race-conditions]]。两大根因：①**多数操作
不是原子的**（加法、字符串拷贝、`count++` 都是多条 CPU 指令）；②**抢占时机不可预测**（时间片到期，
或 UI/网络/其他定时器的中断随时切走线程）。于是出现单写多读的**脏读**（拷到半新半旧的乱串）与多写的
**丢失更新**（计数器被陈旧值覆写）。诡异之处在于它**极其罕见**（8500 万请求里 14 次）、非确定、难复现。
天真的"普通变量当锁"也会失败（读-比较-行动本身非原子），Peterson 等纯软件方案依赖脆弱假设——健壮解法
需硬件同步原语，回扣 [[concepts/concurrency]] 末节"同步要做进硬件"，留作下一条线索。

## 开放问题

- 真实架构（x86 的 0–3 环、ARM 的异常等级）如何映射到这个简化的模式位模型？
- 有哪些现代缓解措施能缩小有 bug 的内核态代码的"爆炸半径"（例如 eBPF 沙箱、用户态驱动、
  Windows 在 CrowdStrike 事件后的改动）？
- 现代真实调度器如何超越多级反馈队列？（Linux 的 CFS/EEVDF、Windows 的优先级+提升、BSD 的 ULE；
  源材料预告了后续会讲。）以及简化为 FIFO 的 IO 队列背后真正的 **IO 调度器**如何工作？
- 防止竞态条件的**同步原语**具体如何工作（互斥锁、信号量、原子指令如 CAS/test-and-set、内存屏障）？
  为何说它们要落到硬件层面？（源材料预告了后续"线程同步"一集。）
