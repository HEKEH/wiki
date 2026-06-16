---
title: "内核态中的驱动与第三方代码"
date: 2026-06-16
tags: [驱动, 内核态, 安全, 蓝屏, 反作弊]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# 内核态中的驱动与第三方代码

操作系统开发者无法为世上每一种硬件设备都写代码（尤其是操作系统发布之后才出现的设备），于是
硬件厂商提供**驱动（drivers）**——控制特定硬件的软件。由于控制硬件需要特权指令，驱动必须
运行在**内核态**（[[concepts/mode-bit-and-operational-modes]]）。

## 核心风险

内核态代码无法被隔离——所有内核态代码共享同一个地址空间（[[concepts/memory-management-unit]]）：

- 写错地址的驱动会破坏操作系统的关键数据。
- 崩溃的驱动会把整个操作系统拖垮 → Windows 上的**蓝屏死机（BSOD）**。

## 驱动之外

第三方内核态代码也被用于反作弊和恶意软件检测，因为（a）它绕过了系统调用开销、运行更快，
（b）它能直接观察硬件和其他进程：

- **反作弊**——用户态的游戏无法分辨输入是否来自被禁用的、映射成键盘的手柄；内核态反作弊可
  直接监控硬件。
- **恶意软件检测**——扫描网卡、检查流量，并读取运行中进程的地址空间以排查可疑行为。

## 当它出问题时

- **[[entities/crowdstrike]]**——一次损坏的配置更新使内核态软件崩溃，进而拖垮了全球的操作
  系统。
- **[[entities/vanguard]]**——Riot Games 的反作弊软件，即使不玩游戏也运行在内核态，通过其
  母公司 [[entities/tencent]] 引发隐私担忧。

## 相关

- [[concepts/mode-bit-and-operational-modes]]
- [[concepts/memory-management-unit]]
- [[concepts/system-calls]]
- [[analysis/detecting-kernel-corruption]] —— 操作系统能否检测这种破坏？
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
