---
title: "抢占式 vs 协作式操作系统"
date: 2026-06-16
tags: [调度, 抢占, 定时器中断, 协作式, 抢占式]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# 抢占式 vs 协作式操作系统

## 协作式（非抢占式）

操作系统依赖用户程序通过 [[concepts/system-calls]] 主动交还控制权。致命缺陷：一个含死循环且
内部无任何系统调用的程序会永远占用 CPU，饿死所有其他进程。正因如此，通用的协作式操作系统
已不再使用。

## 抢占式

光靠软件解决不了死循环问题——必须有硬件支持：一个内建于 CPU、只能通过特权指令控制的
**定时器（timer）**。

- 在把 CPU 交给某进程之前，操作系统设置定时器。
- 进程运行期间，定时器倒计时。
- 进程仍可通过系统调用自愿让出，但若某进程运行太久，定时器到期就触发一次硬件
  [[concepts/interrupts]]，强制把控制权交还操作系统。

这让内核态能精确决定每个进程获得多少 CPU 时间。再加上硬件会捕获用户态下的非法特权指令，
内核态便获得了对 **CPU 如何被使用** 的完全控制——这正是**抢占式（preemptive）**操作系统的
定义。

## 相关

- [[concepts/context-switch]] —— 定时器中断到期后真正切换进程的机制；协作式则退化为纯软件、各程序自保状态
- [[concepts/cpu-scheduling]] —— 抢占是通用调度的默认；轮转/优先级抢占都依赖此处的定时器
- [[concepts/scheduling-algorithms]] —— 轮转(RR)的时间片正是用定时器实现
- [[concepts/race-conditions]] —— 抢占的不可预测时机正是竞态条件的根源
- [[concepts/interrupts]]
- [[concepts/mode-bit-and-operational-modes]]
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
