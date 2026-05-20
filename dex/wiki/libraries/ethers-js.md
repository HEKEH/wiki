---
title: Ethers.js
date: 2026-05-18
tags: [ethers, ethereum, javascript, sdk]
sources: [ethers-js/Ethers-Getting-Started.md]
---

# Ethers.js

小巧、完整的以太坊 JavaScript 库，v6 版本。

## 架构核心：Provider / Signer 分离

Ethers 最重要的设计决策是将链上交互分为两层：

- **Provider** — 只读，查询链上状态
- **Signer** — 写入，封装私钥操作

这比 Web3.js 的单一 Provider 模型更安全，私钥不会暴露给只读操作。详见 [[concepts/provider-signer]]。

## 安装与导入

```bash
npm install ethers
```

```javascript
// 整体导入
import { ethers } from "ethers";

// 按需导入
import { BrowserProvider, parseUnits } from "ethers";

// 子模块导入
import { HDNodeWallet } from "ethers/wallet";
```

## 连接链

```javascript
// MetaMask
const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

// JSON-RPC（自建节点或 INFURA 等服务）
const provider = new JsonRpcProvider("https://mainnet.infura.io/v3/YOUR_KEY");
const signer = await provider.getSigner();
```

## DEX 开发中的典型用途

| 场景             | 用法                                    |
| ---------------- | --------------------------------------- |
| 读取代币余额     | `contract.balanceOf(address)` via Provider |
| 查询池子状态     | `contract.getReserves()` via Provider   |
| 执行 swap        | `contract.swap(...)` via Signer + `tx.wait()` |
| 监听交易事件     | `contract.on("Swap", ...)`              |
| 模拟交易结果     | `contract.swap.staticCall(...)` 不消耗 gas |
| 查询历史 Swap    | `contract.queryFilter(filter, fromBlock)` |

## 查询链上状态

```javascript
await provider.getBlockNumber();                   // 当前区块号
const balance = await provider.getBalance(addr);   // BigInt (wei)
formatEther(balance);                              // → "4.085..."
await provider.getTransactionCount(addr);          // 交易计数（nonce）
```

## 发送交易

```javascript
const tx = await signer.sendTransaction({
  to: "0x...",
  value: parseEther("1.0")
});
const receipt = await tx.wait();  // 等待上链，获取回执
```

## 合约交互

### 创建合约实例

```javascript
// 人类可读 ABI（手写最方便）
const abi = [
  "function balanceOf(address) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 amount)"
];

// 只读连接
const contract = new Contract(address, abi, provider);

// 读写连接
const contract = new Contract(address, abi, signer);
```

### 只读调用

```javascript
const symbol = await contract.symbol();          // 'DAI'
const decimals = await contract.decimals();      // 18n
const balance = await contract.balanceOf(addr);  // 4000000000000000000000n
formatUnits(balance, decimals);                  // '4000.0'
```

### 状态变更调用

```javascript
const amount = parseUnits("1.0", 18);
const tx = await contract.transfer(toAddress, amount);
await tx.wait();  // 必须等待上链才算完成
```

### staticCall 模拟

```javascript
// 不花 gas，模拟执行，检查是否会 revert
const result = await contract.transfer.staticCall(toAddress, amount);

// 以其他身份模拟（VoidSigner 没有私钥，只模拟地址）
const other = new VoidSigner("0x...");
const contractAsOther = contract.connect(other.connect(provider));
await contractAsOther.transfer.staticCall(toAddress, amount);
```

### 事件监听

```javascript
// 监听 Transfer 事件，参数自动解构
contract.on("Transfer", (from, to, amount, event) => {
  console.log(`${from} => ${to}: ${formatEther(amount)}`);
  event.removeListener();  // 移除监听
});

// 过滤：只监听转给某地址的事件
const filter = contract.filters.Transfer(null, targetAddress);
contract.on(filter, (from, to, amount, event) => {});

// 监听所有事件
contract.on("*", (event) => {});
```

### 查询历史事件

```javascript
// 查询最近 100 个区块
const events = await contract.queryFilter(filter, -100);

// EventLog 结构
events[0].args;       // Result [from, to, amount]
events[0].blockNumber;
events[0].address;
```

## 单位转换速查

```javascript
parseEther("1.0")           // → 1000000000000000000n (wei)
formatEther(wei)             // → "1.0"
parseUnits("4.5", "gwei")   // → 4500000000n
formatUnits(val, "gwei")    // → "4.5"
```

## 消息签名

```javascript
// 签名
const sig = await signer.signMessage("Hello");

// 验证 → 恢复签名者地址
verifyMessage("Hello", sig);  // → '0xC08B...'
```

## 详见

- [[sources/ethers-getting-started]] — 官方入门文档完整摘要
- [[libraries/sdk-comparison]] — EVM SDK 对比
