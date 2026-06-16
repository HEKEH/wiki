---
title: "Vanguard（Riot Games 反作弊）"
date: 2026-06-16
tags: [实体, 反作弊, 内核态, 隐私]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# Vanguard

Riot Games 的反作弊软件，用于《英雄联盟》《无畏契约》等游戏。它运行在**内核态**
（[[concepts/kernel-mode-drivers]]）。

## 担忧

- 发布之初曾导致数百名玩家蓝屏死机。
- 玩 Riot 游戏必须安装它（对反作弊而言合理），但它**即使在你不玩游戏时也运行在内核态**，
  意味着它*可能*监控计算机上的一切。
- Riot 由 [[entities/tencent]] 拥有——一家很可能受政府监管的中国公司。考虑到 Vanguard 遍布
  全球数百万台计算机，这引发了隐私担忧。（来源中并未提及实际滥用的证据。）

## 相关

- [[concepts/kernel-mode-drivers]]
- [[entities/tencent]]
- [[entities/crowdstrike]]
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
