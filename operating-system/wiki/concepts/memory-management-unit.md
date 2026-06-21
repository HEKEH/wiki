---
title: "内存管理单元（MMU）"
date: 2026-06-16
tags: [mmu, 内存, 地址空间, 保护]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# 内存管理单元（MMU）

**MMU** 是一个硬件组件，用来限制 CPU 可访问的物理内存区域。它只能通过**特权指令**操控，因此
只有内核态（[[concepts/mode-bit-and-operational-modes]]）下的代码才能控制它。

## 为什么重要

- **完全的内存控制**——因为操作系统控制 MMU，它可以读写物理内存的任何地方，包括用户程序的地址空间。
- **保护中断处理程序**——操作系统利用 MMU 阻止用户进程访问存放 [[concepts/interrupts]] 处理
  程序的内存，使恶意进程无法覆盖处理程序来夺取内核态。
- **共享的内核地址空间**——所有内核态代码（包括驱动）共享同一个地址空间，因此一次错误写入就能破坏操作系统的关键数据。见 [[concepts/kernel-mode-drivers]]。

## 相关

- [[concepts/memory-protection]] —— MMU 的入门版原型：基址+界限寄存器的连续内存保护
- [[concepts/mode-bit-and-operational-modes]]
- [[concepts/interrupts]]
- [[concepts/kernel-mode-drivers]]
- [[concepts/page-fault]] —— MMU 翻译失败/越权时触发的异常
- [[analysis/detecting-kernel-corruption]] —— MMU 为何只能拦非法访问、拦不住合法地址的破坏
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
