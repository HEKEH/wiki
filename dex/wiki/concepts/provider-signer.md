---
title: Provider 与 Signer
date: 2026-05-18
tags: [provider, signer, ethereum, architecture]
sources: [ethers-js/Ethers-Getting-Started.md]
---

# Provider 与 Signer

以太坊交互的两层架构，由 ethers.js 明确分离。

## Provider（只读层）

- 查询账户余额、区块号、交易详情
- 读取事件日志
- 执行 `call`（不改变状态的合约调用）
- 实现：`BrowserProvider`（MetaMask）、`JsonRpcProvider`（RPC 节点）

## Signer（写入层）

- 封装私钥，签名交易和消息
- 私钥位置：内存（`Wallet`）或 IPC 代理（MetaMask 等钱包插件）
- 所有状态变更操作必须通过 Signer

## 为什么分离？

Web3.js 的 Provider 同时承担读写，私钥可能在只读场景中暴露。Ethers 的分离模型：

1. 只读操作不需要私钥 → 更安全
2. `staticCall` 可用 Provider 模拟写入 → 零成本预检
3. 钱包插件（MetaMask）天然映射为 Signer → 权限边界清晰

## DEX 场景映射

| 操作     | 层       | 示例                            |
| -------- | -------- | ------------------------------- |
| 查价格   | Provider | `contract.getAmountsOut()`      |
| 查余额   | Provider | `contract.balanceOf()`          |
| Approve  | Signer   | `contract.approve()`            |
| Swap     | Signer   | `contract.swap()`               |
| 模拟 Swap | Provider | `contract.swap.staticCall()`   |

## 详见

- [[concepts/ethereum-basics]] — 以太坊基础（私钥、地址）
- [[concepts/erc20]] — ERC-20 与 Approve（Signer 场景）
- [[concepts/gas]] — Gas 机制（staticCall 省 gas）
- [[concepts/wallet-types]] — 钱包类型与方案
- [[libraries/ethers-js]] — ethers.js 库概览
- [[libraries/sdk-comparison]] — EVM SDK 对比
- [[entities/metamask]] — MetaMask 热钱包
- [[sources/ethers-getting-started]] — 官方文档摘要
