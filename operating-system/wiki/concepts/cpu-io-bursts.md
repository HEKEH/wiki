---
title: "CPU 突发与 IO 突发（CPU/IO Bursts）"
date: 2026-06-20
tags: [CPU突发, IO突发, CPU密集型, IO密集型, 护航效应, 冯诺依曼]
sources: ["core-dumped-operating-sytems-theory/The Fancy Algorithms That Make Your Computer Feel Smoother.en.srt"]
---

# CPU 突发与 IO 突发（CPU/IO Bursts）

运行中的程序**并不连续使用 CPU**。一个耗时 2 分钟的任务绝不是把 CPU 占满 2 分钟——读写文件、等
按键或网络请求等 **IO 操作**需要调用操作系统、与硬件交互，其耗时取决于 IO 硬件而非 CPU，期间
CPU 对该进程而言是空闲的。于是程序的执行被切成两类时段：

- **CPU 突发(CPU burst)**：程序占用 CPU 计算的时段。
- **IO 突发(IO burst)**：程序等待 IO 完成的时段。

每个进程都是 CPU 突发 → IO 突发 → CPU 突发 → … 交替，最后以一次"终止执行"的系统调用结束。在
**冯·诺依曼架构**（占现代计算机 99% 以上）中，这些空隙是**常态而非例外**。

## 突发时长分布：调度的根本依据

数十年测量表明 CPU 突发时长有规律的分布：**绝大多数是短突发，长突发很少但概率不为零**。这是设计
[[concepts/scheduling-algorithms]] 的关键观察——进程大量时间在等 IO，所以当一个进程进入 IO 突发
时，理想做法是把 CPU 让给另一个不在等 IO 的进程，**尽量让 CPU 保持忙碌**。因此调度器应只把 CPU
分配到**当前 CPU 突发结束**为止，而非到进程完全终止（"first come first served" 更精确的说法其实是
"first CPU burst come first served"）。

## CPU 密集型 vs IO 密集型

据突发特征把进程分两类：

- **IO 密集型(IO-bound)**：大量短 CPU 突发，性能受 **IO 子系统**速度制约。最常见。交互式进程也以
  很短的 CPU 突发为特征（但二者不完全等同）。
- **CPU 密集型(CPU-bound)**：少量长 CPU 突发，受 **CPU** 速度制约。

## 护航效应（convoy effect）

在 FCFS 下，一个 CPU 密集型进程会长时间霸占 CPU。期间其它 IO 密集型进程做完 IO 后全堆在就绪队列
里干等，IO 设备闲置；等大进程让出、小进程飞快跑完又回去做 IO，CPU 又空了下来——大进程再回来独占。
所有进程为一个大进程让路，这就是**护航效应**，导致 CPU 与设备利用率双低。让短进程先走（如 SJF）能
缓解它。

## 相关

- [[concepts/cpu-scheduling]] —— 调度总览与标准
- [[concepts/scheduling-algorithms]] —— SJF 据预测的下一突发长度调度
- [[concepts/process-states]] —— IO 突发对应"等待态"
- [[sources/cpu-scheduling-algorithms]] —— 来源
