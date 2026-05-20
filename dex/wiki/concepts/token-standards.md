---
title: 代币标准与跨链技术栈
date: 2026-05-18
tags: [erc20, erc721, erc1155, token-standards, cross-chain, evm, solana]
sources: []
---

# 代币标准与跨链技术栈

## 以太坊代币标准

ERC-20 不是唯一标准，以太坊上有多种代币标准用于不同场景：

| 标准 | 用途 | 例子 |
| ---- | ---- | ---- |
| **ERC-20** | 同质化代币（可互换） | USDT、UNI、DAI |
| **ERC-721** | 非同质化代币（NFT，每个唯一） | CryptoPunks、BAYC |
| **ERC-1155** | 同质化 + 非同质化混合（批量转移） | 游戏道具 |
| **ERC-4626** | 代币化金库标准（自动生息） | Yearn Vault、Aave aToken |
| **ERC-777** | ERC-20 增强版（支持钩子函数） | 较少使用 |

**同质化 vs 非同质化**：你的 1 USDC 和我的 1 USDC 完全等价（同质化），但你的 NFT 和我的 NFT 完全不同（非同质化）。

## 其他链的同类标准

不同链有自己的代币标准，功能类似但协议不同：

| 链 | 同质化代币标准 | 与 ERC-20 兼容？ |
| ---- | ---- | ---- |
| **Ethereum** | ERC-20 | — |
| **BSC（币安链）** | BEP-20 | 兼容 |
| **Polygon / Arbitrum** | 兼容 ERC-20 | 兼容（L2） |
| **Avalanche** | 兼容 ERC-20 | 兼容（EVM 链） |
| **Solana** | SPL Token | 不兼容，完全不同 |
| **Tron** | TRC-20 | 不兼容，但 API 风格相似 |

## EVM 链 vs 非 EVM 链

**EVM 链**（以太坊、Polygon、BSC、Arbitrum、Avalanche 等）执行以太坊虚拟机，合约和 SDK 可直接复用。**非 EVM 链**（Solana、Tron）架构完全不同，需要换技术栈。

| 维度 | EVM 链 | Solana | Tron |
| ---- | ---- | ---- | ---- |
| 合约语言 | Solidity | Rust | Solidity（编译器不同） |
| SDK | ethers.js / viem / web3.js | @solana/web3.js | tronweb |
| 代币标准 | ERC-20 | SPL Token | TRC-20 |
| 签名算法 | secp256k1 | ed25519 | secp256k1 |
| 钱包 | MetaMask 等通用 | Phantom | TronLink |

> **@solana/web3.js 和 web3.js 是两个完全不同的库**，只是名字碰巧都叫 web3.js。前者由 Solana Labs 维护，只支持 Solana；后者由 ChainSafe 维护，只支持 EVM。

## 核心规律

**同链跨协议 = 只换地址和 ABI，技术栈复用：**

```javascript
// 同一套 ethers.js，调用不同协议只是换地址和 ABI
const uniswapRouter = new Contract(UNISWAP_ADDRESS, UNISWAP_ABI, signer);
const curvePool = new Contract(CURVE_ADDRESS, CURVE_ABI, signer);
const aavePool = new Contract(AAVE_ADDRESS, AAVE_ABI, signer);
```

**跨链 = 换技术栈，代码无法复用：**

```javascript
// EVM
import { ethers } from "ethers";

// Solana（完全不同的包和 API）
import { Connection, PublicKey } from "@solana/web3.js";

// Tron（又是另一个包）
import TronWeb from "tronweb";
```

目前没有任何 SDK 同时支持 EVM 和非 EVM 链。

## 对 DEX 开发的影响

| 场景 | 做法 |
| ---- | ---- |
| 只做 EVM 生态 DEX | ethers.js/viem 一套技术栈覆盖所有 EVM 链和协议 |
| 需要支持 Solana | 独立一套 @solana/web3.js 项目 |
| 跨链聚合 | 多套技术栈 + 跨链桥/聚合协议（如 1inch、LI.FI） |

## 详见

- [[concepts/erc20]] — ERC-20 与 Approve 机制
- [[concepts/smart-contract]] — 智能合约基础
- [[libraries/ethers-js]] — ethers.js 概览
- [[libraries/sdk-comparison]] — EVM SDK 对比（ethers.js vs viem vs web3.js）
