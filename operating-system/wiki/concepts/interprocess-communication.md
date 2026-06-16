---
title: "进程间通信（IPC）"
date: 2026-06-16
tags: [进程间通信, IPC, 协作进程, 进程隔离, 模块化]
sources: ["core-dumped-operating-sytems-theory/IPC： To Share Memory Or To Send Messages.en.srt"]
---

# 进程间通信（IPC）

**进程**是程序运行所处的整个上下文：可执行代码 + CPU 状态 + 分配的内存区域 + 打开的文件列表
+ IO 设备等资源。操作系统默认让进程相互**隔离**（见 [[concepts/mode-bit-and-operational-modes]]
与特权指令机制），但很多场景下进程需要协作。

## 独立进程 vs 协作进程

- **独立进程（independent）**——不与其他进程共享数据，互不影响。
- **协作进程（cooperating）**——需要与其他进程交互；离不开 IPC 机制。

促成协作的理由：

- **共享数据**——共享数据的进程显然在协作。
- **计算加速**——在支持并行的系统上拆分任务、同时执行，缩短总耗时。
- **模块化**——把系统功能拆到不同进程（如 Chromium 的多进程架构，见 [[concepts/shared-memory-ipc]]）。

## 两种基本模型

| | [[concepts/shared-memory-ipc]] | [[concepts/message-passing-ipc]] |
|---|---|---|
| 地址空间 | 共享一块区域 | 保持隔离 |
| 系统调用 | 仅建立/附加时需要 | 每次收发都需要 |
| 速度 | 极快（≈直接内存访问） | 较慢（含系统调用开销） |
| 责任 | 数据格式、同步由进程自负 | 操作系统中介，更安全 |
| 跨机器 | 否 | 可以（套接字 / RPC） |

> 共享内存更快，但消息传递更易用、更安全；99% 的场景下消息传递已足够。

## 相关

- [[concepts/shared-memory-ipc]]
- [[concepts/message-passing-ipc]]
- [[concepts/threads]] —— 进程内协作的轻量替代；跨进程线程共享数据仍需 IPC
- [[concepts/system-calls]] —— 两种模型都依赖系统调用进入内核
- [[sources/ipc-shared-memory-vs-message-passing]] —— 来源
