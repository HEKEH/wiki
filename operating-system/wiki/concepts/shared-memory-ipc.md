---
title: "共享内存（IPC）"
date: 2026-06-16
tags: [共享内存, IPC, 地址空间, 竞态条件, 生产者消费者, Chromium]
sources: ["core-dumped-operating-sytems-theory/IPC： To Share Memory Or To Send Messages.en.srt"]
---

# 共享内存（IPC）

**共享内存**让多个进程直接共享一块内存空间，是 [[concepts/interprocess-communication]] 的
两种基本模型之一。

## 机制

操作系统默认通过特权指令强制进程隔离：进程读写他人地址空间会被立即终止。要共享内存，必须让
操作系统解除这一限制：

> 注：视频把隔离归因于"特权指令"，这是简化。工程上进程隔离主要由
> [[concepts/memory-management-unit]]（虚拟内存/页表）实现——每个进程有独立的地址翻译，越界
> 访问触发 [[concepts/page-fault]]；特权指令则用于保护操控 MMU 的能力本身（只有内核态可改页表）。

1. 用**系统调用**创建共享内存区域，它位于创建者进程的地址空间中。
2. 其他进程用系统调用把该区域**附加（attach）**到自己的地址空间。
3. 之后双方读写这块区域即可通信——**无需再走系统调用**。

**关键**：授予区域后，操作系统不再管理其内容。数据的组织方式、写入位置完全由进程约定，与操作
系统无关。

## 风险（责任落在程序员身上）

- **数据解释约定**——生产者写 8 位有符号数、消费者按 8 位无符号数读取，同样的比特含义不同；
  类型不符（如期望 32 位整数）或地址错误都会导致误解释，轻则失败、重则**未定义行为**。
- **竞态条件**——进程须自己避免同时写同一位置，否则出现 race condition（属线程同步话题）。

## 实例：Chromium 多进程架构

为防止网页 JavaScript 的 bug 拖垮浏览器，Chromium 用共享内存把任务分给三类进程：

- **browser**——管理 UI 与 IO，启动时只创建一次。
- **renderer**——处理网页内容（HTML / JS / 图片），每个标签页一个。
- **plug-in**——管理 Flash、QuickTime、PDF 阅读器等插件。

某标签页崩溃时只有它的 renderer 进程失败，其他标签页不受影响。其他例子：仿真软件、游戏引擎、
数据库管理系统、深度学习框架。

## 优势

系统调用只在创建区域和附加进程时需要；之后通信像访问自身地址空间一样快——基本等同于直接内存
访问。这是相对 [[concepts/message-passing-ipc]] 的主要性能优势。

## 相关

- [[concepts/interprocess-communication]]
- [[concepts/message-passing-ipc]] —— 另一种模型（更慢但更安全）
- [[concepts/system-calls]] —— 创建/附加共享区域所需
- [[sources/ipc-shared-memory-vs-message-passing]] —— 来源
