---
title: "缺页异常（Page Fault）"
date: 2026-06-16
tags: [缺页异常, mmu, 虚拟内存, 异常, 按需调页, 段错误, 页表]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# 缺页异常（Page Fault）

> **范围说明**：源视频 [[sources/single-bit-mode-bit-os-integrity]] 只提到 MMU 限制内存访问，
> 并未展开缺页异常。本页的具体机制（present 位、错误码、CR2、按需调页等）属源视频之外的
> 工程知识，作为概念补充归档。

缺页异常是 **CPU 的一种同步异常**——不是 [[concepts/interrupts]] 里那种由外部设备发来的
异步中断，而是**在执行某条访存指令的当下、由 MMU 当场触发**的。

## 背景：虚拟地址与页表

程序使用虚拟地址，[[concepts/memory-management-unit]] 通过**页表**把虚拟地址翻译为物理地址。
每个页表项（PTE）含两类关键位：

- **present / 有效位**：该页此刻是否在物理内存、是否已建立映射。
- **权限位**：可读 / 可写 / 可执行、用户态可否访问（user/supervisor）。

CPU 访问虚拟地址时 MMU 查页表，**只要翻译或权限检查不通过就抛出缺页异常**。

## 两类原因

1. **页不在场**（present = 0）：尚未映射、被换出到磁盘（swap）、或延迟分配（demand paging /
   懒分配）尚未落盘。
2. **权限违规**（页在场但越权）：写只读页、执行 NX（不可执行）页、用户态访问内核页等。

## 检测机制链

检测由**硬件在地址翻译那一刻完成**，而非软件轮询：

1. **MMU 硬件**翻译时发现违规 → 触发 CPU 异常（x86 上为 `#PF`，向量号 14）。
2. 属 **fault 型异常**：出错指令地址被压栈，便于处理后**重新执行该指令**。
3. CPU 给出诊断信息：
   - **错误码**：区分"不在场 vs 权限违规""读 vs 写""用户态 vs 内核态""取指"等；
   - x86 上**出错的虚拟地址存入 CR2 寄存器**。
4. CPU 自动切到内核态，经中断描述符表（IDT）中 `#PF` 表项跳进内核的**缺页处理程序**——
   即 [[concepts/interrupts]] 那套"跳到固定位置 + 翻转模式位"的机制。

## 处理程序的判定

读 CR2（出错地址）+ 错误码，判断本次缺页是良性还是非法：

- **良性（虚拟内存正常运转）**：按需调页/换入、写时复制（COW）、栈自动增长 → 修好 PTE →
  返回并**重跑该指令**，对程序透明。
- **非法**：出错地址不属于任何合法内存区（VMA），或确实越权 →
  - 用户态：投递 **SIGSEGV**（段错误）；
  - 内核态：**oops / panic / 蓝屏（BSOD）**。

> 关键区分：**"缺页异常"是机制**，多数良性、是虚拟内存的工作基础；**"段错误"只是其中一种
> 非法结局**。两者不是一回事。

## 与内核完整性的关系

缺页异常正是 [[analysis/detecting-kernel-corruption]] 中"MMU 把非法访问转化为异常"的那个
异常。但它也暴露了 MMU 的局限：**写到一个合法、可写的内核地址去覆盖关键数据并不产生缺页
异常**，因此这类破坏检测不到。

## 相关

- [[concepts/memory-management-unit]] —— 触发缺页异常的硬件与页表
- [[concepts/memory-protection]] —— 越权访问转为异常/段错误的前身（基址+界限版）
- [[concepts/interrupts]] —— 异常经 IDT 进入内核态的同一机制
- [[analysis/detecting-kernel-corruption]] —— 缺页异常能/不能检测到什么
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
