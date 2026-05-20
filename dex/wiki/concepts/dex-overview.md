---
title: DEX 概览
date: 2026-05-18
tags: [dex, amm, defi, overview]
sources: []
---

# DEX 概览

DEX（Decentralized Exchange，去中心化交易所）是不依赖中心化中介的代币交易平台。

## CEX vs DEX

| 维度 | CEX（如 Binance） | DEX（如 Uniswap） |
| ---- | ----------------- | ----------------- |
| 资产控制 | 交易所保管你的币 | 你自己保管（自托管） |
| 身份验证 | 需要 KYC | 无需 KYC |
| 交易对手 | 撮合订单簿 | 与流动性池交易 |
| 上币门槛 | 交易所审核 | 任何人可创建池子 |
| 交易速度 | 毫秒级（链下） | 秒级（链上确认） |
| 信任模型 | 信任交易所不跑路 | 信任智能合约代码 |

> 核心区别：CEX 里你的币在交易所手里，DEX 里你的币始终在你自己的钱包里。

## DEX 的运作方式

主流 DEX 采用 **AMM（自动做市商）** 模型，而非传统订单簿：

```
传统交易所（CEX）：  买家 ←撮合→ 卖家     （需要对手方）
AMM 交易所（DEX）：  买家 ←交易→ 流动性池(Liquidity Pool)  （池子永远是对手方）
```

流动性池是一个智能合约，里面同时存放两种代币（如 ETH/USDC）。价格由池中两种代币的比例自动决定，不需要挂单和撮合。

## 一次 DEX Swap 的完整流程

```
用户                    ERC-20 合约              DEX 合约
  │                         │                       │
  ├─ 1. Approve ──────────→ │                       │
  │   "允许 DEX 动我的 USDC" │ 记录授权额度           │
  │                         │                       │
  ├─ 2. Swap ──────────────────────────────────────→│
  │   "用 100 USDC 换 ETH"   │                      │
  │                         │                       │
  │                         │← 3. transferFrom ──── │
  │                         │   DEX 取走你的 USDC    │
  │                         │                       │
  │← 4. 收到 ETH ───────────────────────────────────│
  │   DEX 将 ETH 转给你      │                       │
```

步骤 3 和 4 在同一笔交易内原子性完成——要么都成功，要么都失败。

## DEX 的类型

| 类型 | 代表 | 原理 |
| ---- | ---- | ---- |
| **恒定乘积 AMM** | Uniswap V2 | x × y = k，价格随池子比例变化 |
| **集中流动性 AMM** | Uniswap V3/V4 | LP 选择价格区间，资金效率更高 |
| **稳定币 AMM** | Curve | 针对价格接近 1:1 的资产优化 |
| **加权池 AMM** | Balancer | 支持多代币、自定义权重 |
| **订单簿 DEX** | dYdX, Serum | 传统挂单撮合模式，链上或链下订单簿 |
| **聚合器** | 1inch, MetaMask Swap | 聚合多个 DEX 寻找最优路径 |

## DEX 开发的核心合约

| 合约 | 职责 |
| ---- | ---- |
| **Factory** | 创建新的交易对/池子 |
| **Pair / Pool** | 管理流动资金，执行 swap |
| **Router** | 路由交易，处理多跳 swap、wrap/unwrap ETH |
| **Position Manager** | 管理 LP 的流动性仓位 |

用户通常只和 Router 交互，Router 内部调用 Pair/Pool 完成实际操作。

## 详见

- [[concepts/amm]] — AMM 机制与定价原理
- [[concepts/erc20]] — ERC-20 与 Approve 机制
- [[concepts/gas]] — 为什么交易要付费
- [[concepts/smart-contract]] — 智能合约基础
