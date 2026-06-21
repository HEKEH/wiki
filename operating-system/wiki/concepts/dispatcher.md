---
title: "调度器与分派器（Scheduler & Dispatcher）"
date: 2026-06-20
tags: [调度器, 分派器, 分派延迟, 上下文切换, 甘特图]
sources: ["core-dumped-operating-sytems-theory/The Fancy Algorithms That Make Your Computer Feel Smoother.en.srt"]
---

# 调度器与分派器（Scheduler & Dispatcher）

CPU 调度由两个分工明确的组件协作完成：

- **调度器(scheduler)**：按调度策略管理就绪队列，**决定**下一个该用 CPU 的进程（"决策者"）。
- **分派器(dispatcher)**：**执行**调度器的决定（"执行者"），即真正完成 [[concepts/context-switch]]。

## 分派器做什么

当运行进程进入 IO 等待、CPU 空出时，分派器：

1. 把当前 CPU 状态**存入被中断进程的 PCB**（运行进程虽不入队，但系统始终保留对其 PCB 的引用），
   再把该 PCB 放入相应队列（如等待队列）。
2. 取**就绪队头**的 PCB，认定其为新运行进程，**恢复其寄存器与程序计数器**，重新分配 CPU，让它从
   被中断处继续。
3. 若处理器支持，在交还控制权前**切回用户态**。

## 分派延迟（dispatch latency）

分派器本身也要消耗 CPU 来执行上下文切换，这段不可避免的延迟叫**分派延迟**——从停下一个进程到启动
另一个进程所花的时间。因为**每次上下文切换都调用它**，它必须尽量快，是操作系统中**被高度优化**的
组件之一。由于它是隐式的，画图时通常不在进程之间显式标出分派延迟。

> 调度相关的图常用**甘特图(Gantt chart)** 的变体：一种条形图，展示各参与进程 CPU 突发的起止时间。

## 相关

- [[concepts/cpu-scheduling]] —— 调度总览与标准
- [[concepts/context-switch]] —— 分派器执行的核心动作（及其纯软件做不到的难题）
- [[concepts/process-states]] —— 分派器据状态转移搬运 PCB
- [[sources/cpu-scheduling-algorithms]] —— 来源
