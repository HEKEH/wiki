---
title: Gas 机制
date: 2026-05-18
tags: [gas, ethereum, basics]
sources: []
---

# Gas 机制

以太坊上的每次写入操作都要付费，这笔费用叫 **Gas**。

## 为什么需要 Gas？

区块链是一个全球计算机，每个节点都要执行你的交易。如果没有费用：

- 人们可以无限发送垃圾交易，把网络撑爆
- 节点没有经济激励去运行

Gas 本质上是**计算资源的定价机制**：你用了多少算力，就付多少钱。

## Gas 的组成

```
交易费 = Gas Used × Gas Price
```

| 概念 | 说明 |
| ---- | ---- |
| **Gas Limit** | 你愿意付出的最大 gas 单位数（类似"加满油箱"） |
| **Gas Used** | 实际消耗的 gas 单位数（类似"实际用了多少油"） |
| **Gas Price** | 每单位 gas 的价格，单位是 gwei |

### Gwei 是什么？

```
1 ETH = 10^9 Gwei = 10^18 Wei
```

Gwei 是 gas 报价的常用单位。比如 gas price = 20 gwei，意思是每单位 gas 值 0.00000002 ETH。

## 实际例子

假设你要做一次 ETH 转账：

- Gas Used：21,000（简单转账的实际消耗是固定的）
- Gas Limit：通常也设为 21,000（与 Gas Used 一致即可）
- Gas Price：20 gwei
- 实际费用：21,000 × 20 gwei = 420,000 gwei = 0.00042 ETH

> 如果是合约调用，Gas Limit 可能设 200,000，但 Gas Used 可能只有 150,000。你只付实际消耗的部分，多余的退回。

如果是 DEX swap，gas used 通常在 150,000–300,000 之间（因为合约逻辑更复杂）。

## 交易失败也要付费？

**是的。** 即使交易 revert（失败），节点已经执行了计算并记录了失败结果，所以 gas 费不退还。

这是 DEX 开发中 `staticCall` 重要的原因——它让你在不花钱的情况下模拟交易是否能成功。参见 [[concepts/provider-signer]]。

## EIP-1559 之后的费用结构

2021 年伦敦升级后，费用分为两部分：

| 部分 | 给谁 | 说明 |
| ---- | ---- | ---- |
| **Base Fee** | 销毁 | 网络自动调节，你无法控制 |
| **Priority Fee（小费）** | 验证者 | 你出价越高，越快被打包 |

```javascript
// ethers.js 中发送交易时指定
tx = await signer.sendTransaction({
  to: address,
  value: parseEther("0.1"),
  maxPriorityFeePerGas: parseUnits("2", "gwei"),  // 小费
  maxFeePerGas: parseUnits("30", "gwei"),          // 愿付最高单价
});
```

## DEX 开发中的 Gas 注意事项

1. **swap 前模拟**：用 `staticCall` 预检，避免花钱买失败
2. **设置合理 slippage**：gas 波动可能导致交易时价格已变
3. **Gas 影响收益**：小额 swap 时，gas 可能吃掉大部分利润
4. **批量操作**：合约层面合并多次调用可省 gas

## 详见

- [[concepts/ethereum-basics]] — 以太坊基础概念
- [[concepts/dex-overview]] — DEX 概览
- [[concepts/amm]] — AMM 与流动性池
- [[concepts/provider-signer]] — staticCall 模拟交易
- [[libraries/ethers-js]] — ethers.js 中的 gas 设置
