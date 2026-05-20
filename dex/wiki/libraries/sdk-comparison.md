---
title: EVM SDK 对比
date: 2026-05-18
tags: [ethers, viem, web3js, sdk, evm, comparison]
sources: []
---

# EVM SDK 对比

EVM 生态有三个主要的 JavaScript SDK：ethers.js、viem、web3.js。新项目应选 ethers.js 或 viem。

## 对比总览

| 维度 | ethers.js v6 | viem | web3.js v4 |
| ---- | ---- | ---- | ---- |
| **设计哲学** | Provider/Signer 分离 | 轻量、模块化 | Provider 合一（读写不分） |
| **TypeScript** | 原生支持，类型完善 | 原生支持，类型最严格 | v4 改善，但仍弱于前两者 |
| **包体积** | 较小 | 最小 | 较大 |
| **链上数值类型** | BigInt（安全） | BigInt（安全） | v3 用 BN/Number（易溢出），v4 改善 |
| **社区趋势** | 当前主流 | 新项目增长快 | 老项目维护用，新项目少选 |
| **与前端框架集成** | 独立使用 | 和 wagmi 深度配合 | 独立使用 |

## Provider / Signer 设计差异

这是最本质的架构区别：

```javascript
// web3.js：Provider 同时承担读写
const web3 = new Web3(provider);
await web3.eth.getBalance(address);        // 读
await web3.eth.sendTransaction({...});     // 写
// 同一个对象，读写不分

// ethers.js：读写明确分离
const provider = new BrowserProvider(window.ethereum);  // 只读
const signer = await provider.getSigner();               // 写入
await provider.getBalance(address);                       // 用 provider 读
await signer.sendTransaction({...});                      // 用 signer 写

// viem：类似 ethers，Client + Wallet 分离
const publicClient = createPublicClient({ transport: http() });  // 只读
const walletClient = createWalletClient({ transport: custom() }); // 写入
```

ethers.js 和 viem 的分离设计更安全——只读操作不会意外触发写入，私钥不会暴露给不需要它的场景。详见 [[concepts/provider-signer]]。

## 选型建议

| 场景 | 推荐 | 原因 |
| ---- | ---- | ---- |
| 新项目（React） | **viem + wagmi** | wagmi 提供 React Hooks，开发体验最好 |
| 新项目（非 React） | **ethers.js** | 成熟稳定，文档丰富 |
| 老项目维护 | web3.js v4 | 渐进迁移，不必重写 |
| 需要最大兼容性 | **ethers.js** | 生态最广，教程最多 |

## 注意：web3.js ≠ @solana/web3.js

这两个是**完全不同的库**，只是名字碰巧相同：

| | web3.js | @solana/web3.js |
| ---- | ---- | ---- |
| NPM 包名 | `web3` | `@solana/web3.js` |
| 维护团队 | ChainSafe | Solana Labs |
| 支持的链 | 仅 EVM | 仅 Solana |

没有任何一个 SDK 同时支持 EVM 和 Solana。详见 [[concepts/token-standards]]。

## 详见

- [[libraries/ethers-js]] — ethers.js 概览与 DEX 用途
- [[concepts/provider-signer]] — Provider/Signer 架构
- [[concepts/token-standards]] — 跨链技术栈差异
