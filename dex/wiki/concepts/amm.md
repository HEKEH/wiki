---
title: AMM 与流动性池
date: 2026-05-18
tags: [amm, liquidity, uniswap]
sources: []
---

# AMM 与流动性池

AMM（Automated Market Maker，自动做市商）是主流 DEX 的核心定价和交易机制。

## 从传统做市说起

传统金融中，做市商同时挂买价和卖价，赚差价。比如：

- 买价 1.00 USDC/ETH  — 做市商愿意以这个价买入
- 卖价 1.02 USDC/ETH  — 做市商愿意以这个价卖出

这需要做市商持续报价和管理库存。在链上，这太贵也太慢。

## AMM 的核心想法

**用数学公式代替做市商的报价。**

最经典的是 Uniswap V2 的**恒定乘积公式**：

```text
x × y = k
```

- x = 池中代币 A 的数量
- y = 池中代币 B 的数量
- k = 常数（swap 过程中不变，添加/移除流动性时才变化）

### 为什么 k 是常数？

这不是自然规律，而是**智能合约强制执行的规则**。每次 swap 时合约检查 `新x × 新y >= 旧k`，不满足就 revert。

固定 k 带来的效果：买得越多越贵，自然抑制大额交易，防止池子被掏空。如果 k 不固定（比如允许 x+y=常数），大额交易几乎不滑点，池子会被套利者抽干。

### 直观理解

想象一个池子里有 100 ETH 和 200,000 USDC：

```text
100 × 200,000 = 20,000,000 (k)

如果有人买入 1 ETH：
池子变成 (100-1) ETH 和 (200,000 + Δ) USDC
99 × (200,000 + Δ) = 20,000,000
Δ ≈ 2,020.20 USDC

所以买 1 ETH 要花约 2,020 USDC（不是 2,000）
```

**买入越多，价格越贵**——这就是滑点的来源。

## 价格 = 两种代币的比例

```text
价格 = y / x = 200,000 / 100 = 2,000 USDC/ETH
```

每次 swap 改变池子比例，价格就自动更新。不需要任何人工报价。

## 流动性提供者（LP）

池子里的钱从哪来？**流动性提供者（Liquidity Provider）** 存入等值的两种代币：

```text
存入 1 ETH + 2,000 USDC → 获得 LP Token（代表你在池子里的份额）
```

LP 的收益来源：

- **交易手续费**：每笔 swap 收 0.05%–1%（Uniswap V2 固定 0.3%）
- LP 按份额分配手续费

LP 的风险：

- **无常损失（Impermanent Loss）**：池子比例变化时，LP 的资产价值可能低于单纯持有

## 滑点（Slippage）

你看到的报价和实际成交价之间的差距。

```text
池子：100 ETH / 200,000 USDC → 显示价格 2,000 USDC/ETH

你想买 10 ETH（池子 10% 的 ETH）：
实际需要支付 ≈ 22,222 USDC
平均价格 ≈ 2,222 USDC/ETH
滑点 = (2,222 - 2,000) / 2,000 = 11.1%
```

**池子越深（流动性越大），同样交易量的滑点越小。** 这就是流动性重要的原因。

DEX 前端通常让你设置滑点容忍度（如 0.5%），超过就回滚交易。

## Uniswap V3 的改进：集中流动性

V2 中 LP 的资金均匀分布在整个价格区间（0 到 ∞），大部分资金闲置。

V3 允许 LP 选择价格区间：

```text
V2:  资金分布 ████████████████████████████████ (全价格区间)
V3:  资金分布         ████████████             (选定区间)
                       ↑ 集中在这里
```

优点：资金效率大幅提升（同量资金可支持更大交易量）
代价：LP 需要主动管理区间，价格超出区间时资金停止赚取手续费

## 代码：查询池子状态

```javascript
// Uniswap V2 风格
const pair = new Contract(PAIR_ADDRESS, [
  "function getReserves() view returns (uint112, uint112, uint32)",
  "function token0() view returns (address)",
  "function token1() view returns (address)"
], provider);

const [reserve0, reserve1] = await pair.getReserves();
const price = Number(reserve1) / Number(reserve0);
```

## 详见

- [[concepts/dex-overview]] — DEX 整体架构
- [[concepts/gas]] — 交易费用机制
- [[concepts/erc20]] — 代币标准与授权
