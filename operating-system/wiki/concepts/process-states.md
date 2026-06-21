---
title: "进程状态与 PCB（Process States）"
date: 2026-06-20
tags: [进程状态, PCB, 就绪队列, 等待队列, 进程控制块, 队列]
sources: ["core-dumped-operating-sytems-theory/The Fancy Algorithms That Make Your Computer Feel Smoother.en.srt"]
---

# 进程状态与 PCB（Process States）

进程**不是**传统意义上的对象或结构体实例，而是"计算机运行所处的**上下文**"。操作系统用一个叫
**进程控制块(PCB, Process Control Block)** 的结构来**表示**进程，记录其**状态、程序计数器、
寄存器**等管理数据。每创建一个进程就生成一个 PCB——进入调度队列的正是 PCB，而非进程本身。

> 在 Linux 中进程与线程统一为 `task` 结构（见 [[concepts/threads]]）；被中断时的寄存器状态最终
> 存入 PCB/`task`（见 [[concepts/context-switch]]）。

## 五种状态

- **新建(new)**：程序刚启动，可执行文件还在载入内存。
- **就绪(ready)**：已载入并分配好资源，**等待 CPU 时间**。
- **运行(running)**：CPU 已分配给它（同一核心同一时刻只有一个进程在此态）。
- **等待(waiting)**：进程发出 IO 等系统调用、自愿交还 CPU，等待事件发生或 IO 完成。注意 IO 调用
  不只为请求资源（文件、内存），也可单纯**等待事件**（按键、网络请求）。完成后回到**就绪**态——
  **不会立即恢复运行**，因为期间 CPU 多半已给了别人，它必须等轮到自己。
- **终止(terminated)**：多数用户程序最终到此。退出后操作系统还要释放内存等资源，故某些系统进一步
  细分"**正在终止**"与"**已完全释放**"。

状态名因系统而异，但代表的状态各系统皆有。进程一生大多在 **就绪 ↔ 运行 ↔ 等待** 三态间循环；
web 服务器等设计为永不终止、长留此循环。

## 为何队列是核心数据结构

除**运行态**外，其它状态（尤其等待态，但就绪态同样在等）的进程都在等待。任一时刻可能有数百个进程
在等待，用**队列**管理其执行顺序既合理又高效——这正是 CPU 调度以队列为基础数据结构的原因。但队列
不适用于运行进程：一个核心同一时刻只能运行一个进程，运行进程不入队（仅保留对其 PCB 的引用，供
[[concepts/dispatcher]] 在中断时存盘）。

## 相关

- [[concepts/cpu-scheduling]] —— 调度总览
- [[concepts/dispatcher]] —— 分派器据状态转移搬运 PCB
- [[concepts/cpu-io-bursts]] —— IO 突发 → 等待态；CPU 突发 → 运行态
- [[concepts/context-switch]] —— 保存/恢复落在 PCB 中的状态
- [[sources/cpu-scheduling-algorithms]] —— 来源
