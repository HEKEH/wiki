---
title: "CPU 调度（CPU Scheduling）"
date: 2026-06-20
tags: [CPU调度, 调度标准, 响应时间, 吞吐量, 周转时间, 等待时间, 抢占, 饥饿]
sources: ["core-dumped-operating-sytems-theory/The Fancy Algorithms That Make Your Computer Feel Smoother.en.srt"]
---

# CPU 调度（CPU Scheduling）

**CPU 调度**是操作系统根据特定标准，决定**某一时刻把 CPU 分给哪个执行单位**的技术。它是任何
操作系统最重要的组件之一：基准测试把 CPU 占用打到 100% 时系统仍可用，正是因为通用操作系统
**优先保证响应时间**，让交互式应用在重负载下也流畅。

> "把 CPU 分配给进程"在底层意味着让 CPU 的**程序计数器(PC)** 指向该进程的文本段；换进程即修改
> PC，这一过程是[[concepts/context-switch]]。进入队列的并非进程本身，而是其 **PCB**
> （见 [[concepts/process-states]]）。

## 调度标准

不同系统侧重不同标准，常需折中平衡：

- **CPU 利用率(utilization)**：让 CPU 尽量忙碌。但"忙"若都花在上下文切换上则无意义。
- **吞吐量(throughput)**：单位时间完成的进程数。
- **周转时间(turnaround time)**：从进程创建到完成的总时间 = 就绪等待 + CPU 执行 + IO 时间。
- **等待时间(waiting time)**：进程在就绪队列中"就绪却拿不到 CPU"的时间——白白浪费的时间，最小化
  它很关键（[[concepts/scheduling-algorithms]] 中 SJF 可证明最小化平均等待时间）。
- **响应时间(response time)**：从提交请求到产生**第一个响应**的时间（不是完成时间）。它直接决定
  交互系统的**流畅感**，尤其在多任务时——通用桌面系统最看重它。

## 抢占式 vs 非抢占式

当一个更"该跑"的进程进入就绪队列、而另一进程正在运行时：

- **抢占式(preemptive)**：操作系统**可以打断**正在运行的进程。
- **非抢占式(non-preemptive)**：绝不打断，新进程必须等运行进程让出 CPU。

抢占是现代通用系统的默认，因为非抢占无法对付不含系统调用的**死循环**（进程永不交还 CPU，其它进程
全部冻结，恶意软件还能借此垄断 CPU）。抢占必须有硬件**定时器**支持——详见
[[concepts/preemptive-vs-cooperative-os]]。

## 饥饿与老化

调度策略导致某进程在就绪队列中**无限等待**即**饥饿(starvation)**。它是**策略本身**的后果，而非
单看进程突发长短：SJF 和优先级调度都可能饿死长突发/低优先级进程。解法是**老化(aging)**——逐渐
提高长期等待进程的优先级。详见 [[concepts/scheduling-algorithms]]。

## 相关

- [[concepts/cpu-io-bursts]] —— CPU 突发/IO 突发模型；调度优化的根本依据
- [[concepts/process-states]] —— 新建/就绪/运行/等待/终止；队列为何是核心数据结构
- [[concepts/dispatcher]] —— 调度器（决策）与分派器（执行）的分工
- [[concepts/scheduling-algorithms]] —— FCFS / SJF / RR / 优先级 / 多级（反馈）队列
- [[concepts/preemptive-vs-cooperative-os]] —— 定时器中断如何让抢占成为可能
- [[sources/cpu-scheduling-algorithms]] —— 来源
