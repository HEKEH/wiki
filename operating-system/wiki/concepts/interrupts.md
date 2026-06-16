---
title: "中断"
date: 2026-06-16
tags: [中断, cpu, 上下文切换, 中断处理程序]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# 中断

**中断（interrupt）**是发送给 CPU 的信号，表示发生了某个事件。它会让处理器暂停当前进程，
跳到一个固定内存位置，那里存放着处理该事件的代码。中断是翻转
[[concepts/mode-bit-and-operational-modes]] 进入内核态的机制。

## 中断来源

- **硬件**——例如按键、鼠标移动，或硬件定时器到期（见
  [[concepts/preemptive-vs-cooperative-os]]）。
- **软件**——例如 `int` 指令，这正是 [[concepts/system-calls]] 进入内核的方式。

## 处理例程的四个步骤

1. **保存**被中断进程的状态。
2. **处理**中断（运行处理例程）。
3. **恢复**被中断进程的状态。
4. **中断返回**——跳回并恢复被中断的进程。

## 中断的两个效果

1. 程序计数器跳到处理程序所在的内存位置。
2. 模式位被自动置为 **1**（内核态），使处理程序以特权方式运行。

## 为什么这很微妙

原则上任何可执行实体都可能位于中断处理程序所在的位置，从而获得内核态。因此操作系统会
（a）确保自己的代码被装载在那里，并（b）利用 [[concepts/memory-management-unit]] 阻止用户
进程访问或覆盖处理程序内存。特权指令让操作系统能重定义 CPU 跳转的*目标位置*，使处理程序
可以安全地放在内存的任何地方。

## 相关

- [[concepts/mode-bit-and-operational-modes]]
- [[concepts/system-calls]]
- [[concepts/preemptive-vs-cooperative-os]]
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
