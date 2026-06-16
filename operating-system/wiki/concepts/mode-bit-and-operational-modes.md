---
title: "模式位与 CPU 运行模式"
date: 2026-06-16
tags: [模式位, 内核态, 用户态, 特权指令, cpu]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# 模式位与 CPU 运行模式

**模式位（mode bit）**是 CPU 内部的一个单比特硬件寄存器，用来区分哪个软件被允许执行
**特权指令**。它是让操作系统得以掌控机器的硬件基石。

## 工作原理

- **模式位 = 1 → 特权态 / 内核态。** CPU 可执行指令集中的*任何*指令，包括特权指令。
- **模式位 = 0 → 受限态 / 用户态。** 特权指令被禁止。

CPU 用组合逻辑电路译码指令。对普通指令（算术、数据搬移、分支、循环）而言，模式位没有影响
——它们在任一模式下都能运行。对**特权指令**而言，模式位决定能否执行。

## 特权指令

为可信软件保留的指令子集，包括：

- 直接操控 I/O 设备（网卡/显卡、硬盘、外设）。
- 操控 [[concepts/memory-management-unit]]（MMU）。
- 重定义 [[concepts/interrupts]] 的处理方式。
- 控制硬件定时器（见 [[concepts/preemptive-vs-cooperative-os]]）。

如果用户态下的进程试图执行特权指令，硬件可被配置为立即触发一次中断，把控制权交给操作
系统——后者通常会终止这个违规程序。

## 命名

特权态是给操作系统内核用的，故称**内核态（kernel mode）**；受限态是给用户程序用的，故称
**用户态（user mode）**。

## 谁来翻转模式位？

只有 [[concepts/interrupts]] 能把模式位翻转为 1，且中断会跳到一个固定内存位置。谁控制了那个
位置上的代码，谁就控制了机器——所以操作系统确保*自己*被装载在那里。详见
[[concepts/interrupts]] 与 [[concepts/system-calls]]。

## 相关

- [[concepts/interrupts]] —— 翻转模式位的机制
- [[concepts/system-calls]] —— 为用户程序提供的受控模式切换
- [[concepts/memory-management-unit]] —— 由特权指令保护
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
