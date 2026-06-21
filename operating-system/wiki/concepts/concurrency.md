---
title: "并发（Concurrency）"
date: 2026-06-16
tags: [并发, CPU 调度, 多任务, 阻塞效应, CPU 利用率, 并行]
sources: ["core-dumped-operating-sytems-theory/Why Are Threads Needed On Single Core Processors.en.srt"]
---

# 并发（Concurrency）

**并发**指操作系统在单个 CPU 上**快速轮换**多个执行单位的访问权，快到让用户产生"同时运行"的
错觉。它与 **CPU 调度**一起，造就了流畅的多任务体验。

## 并发的两个目的

1. **多任务**(广为人知)——让多个程序看起来同时运行。
2. **最大化 CPU 利用率**(不那么显眼)——进程"是执行中的程序"，但不等于它总在用 CPU。当一个
   进程**等待 IO** 而无法执行指令时，把 CPU 留给它就是浪费;不如把 CPU **填补**给另一个就绪的
   执行单位。并发就是用"能用 CPU 的任务"去填满"用不上 CPU 的任务"留下的空隙。

> 关键:本概念**即使在单核处理器上也成立**——单核并发不是"真正同时跑"，而是**消除空闲、填满
> 缝隙**。真正"同时跑"需要多核，那叫**并行**——见 [[concepts/parallelism]]。可以有并发而没有并行。

## 阻塞效应（Blocking Effect）

当一个任务因为另一个(本应独立的)任务正在运行而无法执行，就称为**阻塞效应**。典型场景:服务器
顺序地 `accept → process(昂贵的磁盘 IO) → respond`，多个请求几乎同时到达时，后来的请求要苦等
前一个处理完;时间线上出现大片 CPU **闲置的灰色空隙**=被浪费的计算时间(详见
[[analysis/multiprocess-vs-multithreaded-server]])。

## 如何获得进程内并发

单个进程只有一个程序计数器，无法让进程内两段代码并发。两条路径:

- **多进程 + [[concepts/interprocess-communication]]**——把代码拆到不同进程;但 IPC 不直观、
  创建进程开销大、内存占用高。
- **[[concepts/threads]]**——给进程内每个并发实体各一个程序计数器与 CPU 状态;轻量、共享地址
  空间。这是现代主流做法。

## 同步的必要性

并发任务共享内存(尤其是堆)时,若一个读、另一个同时写同一块内存,结果可能是灾难性的
([[concepts/race-conditions]])。因此需要**同步**机制;其重要性之高，以至于相关原语直接在**硬件**
层面实现(属另一主题)。

## 相关

- [[concepts/threads]] —— 实现进程内并发的执行单位
- [[concepts/race-conditions]] —— 共享内存并发不加同步时的灾难性后果
- [[concepts/parallelism]] —— 多核上把并发升级为真正同时执行
- [[concepts/interprocess-communication]] —— 进程间协作的另一条路径
- [[concepts/preemptive-vs-cooperative-os]] —— 操作系统靠定时器中断强制轮换，使并发可被抢占
- [[analysis/multiprocess-vs-multithreaded-server]] —— 阻塞与两种并发方案的实例
- [[sources/why-threads-single-core]] —— 来源
