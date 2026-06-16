---
title: "CrowdStrike"
date: 2026-06-16
tags: [实体, 安全, 内核态, 事故]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# CrowdStrike

一款安装在全球数百万台企业设备上、运行于**内核态**（[[concepts/kernel-mode-drivers]]）的
恶意软件检测软件。

## 事故

一次更新损坏了一个配置文件（将其填满了零）。文件损坏后，软件崩溃——又因为它运行在内核态，
把操作系统也一起拖垮了。结果：全球范围的混乱，数千台设备蓝屏死机，企业损失数十亿美元。

> 勘误：来源视频称事故发生在"九月"，但现实中这次 CrowdStrike Falcon 故障发生在
> **2024 年 7 月 19 日**。

这是 [[concepts/kernel-mode-drivers]] 核心风险的教科书式案例：内核态代码共享同一地址空间、
无法被隔离，因此它的失败就是整个系统的失败。

## 相关

- [[concepts/kernel-mode-drivers]]
- [[entities/vanguard]] —— 另一个内核态的反面教材
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
