# DEX Knowledge Base — Home

去中心化交易所技术生态知识库，由 LLM 增量构建与维护。

## Quick Links

- [[index]] — 所有页面目录
- [[log]] — 活动日志

## 学习路径（新人推荐）

按以下顺序阅读，从零基础到理解 DEX 开发：

1. [[concepts/ethereum-basics]] — 区块链与以太坊基础概念
2. [[concepts/gas]] — 为什么交易要收费
3. [[concepts/smart-contract]] — 智能合约是什么
4. [[concepts/erc20]] — 代币标准与 Approve 机制
5. [[concepts/provider-signer]] — Provider/Signer 读写分离
6. [[concepts/wallet-types]] — 钱包类型与方案（插件/嵌入式/硬件）
7. [[concepts/dex-overview]] — DEX 整体架构与交易流程
8. [[concepts/amm]] — AMM 定价与流动性池
9. [[concepts/token-standards]] — 代币标准族与跨链技术栈
10. [[libraries/ethers-js]] — ethers.js 开发库
11. [[libraries/sdk-comparison]] — EVM SDK 对比（ethers.js vs viem vs web3.js）

## Current State

知识库聚焦于 **从零到 DEX 开发** 的基础知识链，已覆盖从以太坊基础到 AMM 机制的完整学习路径。

### 已覆盖

- 以太坊基础（地址、私钥、交易、测试网、EVM/非EVM 链）
- Gas 机制（费用构成、EIP-1559、DEX 中的注意事项）
- 智能合约（Solidity、ABI、读写模式）
- ERC-20 与 Approve 机制（授权流程、安全注意事项、decimals）
- Provider/Signer 架构
- DEX 概览（AMM vs 订单簿、swap 流程、合约架构）
- AMM 与流动性池（恒定乘积、滑点、集中流动性）
- 代币标准族（ERC-20/721/1155/4626）与跨链技术栈差异
- 钱包类型（浏览器插件/嵌入式/WalletConnect/账户抽象）
- MetaMask 热钱包（安全模型、沙盒隔离、局限）
- 硬件钱包（Ledger/Trezor、冷存储原理）
- ethers.js v6 入门
- EVM SDK 对比（ethers.js vs viem vs web3.js）
- Ethers.js Getting Started 官方文档摘要

### 待覆盖

- Uniswap V2/V3/V4 合约详解
- 前端框架集成（wagmi、viem、RainbowKit）
- 聚合器与路由（1inch、0x）
- 无常损失的量化分析
- MEV 与三明治攻击

## Open Questions

- ethers.js vs viem：现代 DEX 前端应选哪个？
- Uniswap V4 的 Hooks 架构对 DEX 开发范式的影响？
