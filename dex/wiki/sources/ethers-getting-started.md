---
title: Ethers.js Getting Started — 源文摘要
date: 2026-05-18
tags: [ethers, ethereum, javascript, sdk]
sources: [ethers-js/Ethers-Getting-Started.md]
---

# Ethers.js Getting Started

来源：[ethers.js v6 官方文档](https://docs.ethers.org/v6/getting-started/)

## 核心对象

| 对象            | 角色     | 说明                                                                    |
| --------------- | -------- | ----------------------------------------------------------------------- |
| **Provider**    | 只读连接 | 查询链上状态（余额、区块、交易、日志），执行 `call`                     |
| **Signer**      | 写入操作 | 封装账户交互，私钥可在内存（Wallet）或 IPC 层（MetaMask）               |
| **Contract**    | 合约交互 | 元类，运行时根据 ABI 动态生成方法；连接 Provider 可读，连接 Signer 可写 |
| **Transaction** | 状态变更 | 需付 gas 费，revert 也付费                                              |
| **Receipt**     | 交易回执 | 包含区块号、实际费用、事件、成功/失败状态                                |

## 连接方式

### MetaMask / 注入 Provider

```javascript
let signer = null;
let provider;

if (window.ethereum == null) {
  console.log("MetaMask not installed; using read-only defaults");
  provider = ethers.getDefaultProvider();
} else {
  provider = new ethers.BrowserProvider(window.ethereum);
  signer = await provider.getSigner();
}
```

### JSON-RPC

```javascript
provider = new ethers.JsonRpcProvider(url);
signer = await provider.getSigner();
```

## 单位转换

以太坊内部使用整数（wei），显示时需要转换为人类可读格式：

```javascript
eth = parseEther("1.0");              // 1000000000000000000n
feePerGas = parseUnits("4.5", "gwei"); // 4500000000n
formatEther(eth);                      // '1.0'
formatUnits(feePerGas, "gwei");        // '4.5'
```

| 函数                         | 用途                   |
| ---------------------------- | ---------------------- |
| `parseEther("1.0")`          | ETH → wei              |
| `formatEther(wei)`           | wei → ETH 字符串       |
| `parseUnits("4.5", "gwei")`  | 带单位 → 最小单位      |
| `formatUnits(val, unit)`     | 最小单位 → 带单位字符串 |

## 查询链上状态

```javascript
await provider.getBlockNumber();                     // 23929462
balance = await provider.getBalance("ethers.eth");   // 4085267032476673080n
formatEther(balance);                                // '4.08526703247667308'
await provider.getTransactionCount("ethers.eth");    // 2
```

## 发送交易

```javascript
tx = await signer.sendTransaction({
  to: "ethers.eth",
  value: parseEther("1.0")
});
receipt = await tx.wait();
```

## 合约交互

### ABI（应用二进制接口）

ABI 定义了如何编码请求和解码结果。有两种常见格式：

- **JSON ABI**：Solidity 编译器输出的标准格式
- **人类可读 ABI**：直接用 Solidity 签名，手写更方便

```javascript
// 人类可读 ABI 格式
const abi = [
  "function decimals() view returns (uint8)",
  "function symbol() view returns (string)",
  "function balanceOf(address a) view returns (uint)"
];
```

只需包含你要用到的方法和事件，其余可省略。

### 只读方法（view / pure）

```javascript
const abi = [
  "function decimals() view returns (uint8)",
  "function symbol() view returns (string)",
  "function balanceOf(address a) view returns (uint)"
];
const contract = new Contract("dai.tokens.ethers.eth", abi, provider);

sym = await contract.symbol();                    // 'DAI'
decimals = await contract.decimals();             // 18n
balance = await contract.balanceOf("ethers.eth"); // 4000000000000000000000n
formatUnits(balance, decimals);                   // '4000.0'
```

### 状态变更方法

```javascript
const abi = ["function transfer(address to, uint amount)"];
const contract = new Contract("dai.tokens.ethers.eth", abi, signer);

const amount = parseUnits("1.0", 18);
tx = await contract.transfer("ethers.eth", amount);
await tx.wait();
```

### staticCall（模拟调用）

用 Provider 模拟状态变更调用，不实际发交易、不花 gas：

```javascript
const abi = ["function transfer(address to, uint amount) returns (bool)"];
const contract = new Contract("dai.tokens.ethers.eth", abi, provider);

const amount = parseUnits("1.0", 18);
await contract.transfer.staticCall("ethers.eth", amount); // true

// 以其他身份模拟
const other = new VoidSigner("0x643aA0A61eADCC9Cc202D1915D942d35D005400C");
const contractAsOther = contract.connect(other.connect(provider));
await contractAsOther.transfer.staticCall("ethers.eth", amount); // true
```

`VoidSigner` 是一个没有私钥的 Signer，用于模拟特定地址的调用。

### 事件监听

```javascript
const abi = ["event Transfer(address indexed from, address indexed to, uint amount)"];
const contract = new Contract("dai.tokens.ethers.eth", abi, provider);

// 监听指定事件，参数自动解构
contract.on("Transfer", (from, to, _amount, event) => {
  const amount = formatEther(_amount, 18);
  console.log(`${from} => ${to}: ${amount}`);
  event.removeListener();  // 移除当前监听
});

// 使用过滤器函数
contract.on(contract.filters.Transfer, (from, to, amount, event) => {});

// 过滤特定条件：只监听转给 ethers.eth 的 Transfer
filter = contract.filters.Transfer(null, "ethers.eth");
contract.on(filter, (from, to, amount, event) => {});

// 监听所有事件
contract.on("*", (event) => {});
```

### 查询历史事件

```javascript
const abi = ["event Transfer(address indexed from, address indexed to, uint amount)"];
const contract = new Contract("dai.tokens.ethers.eth", abi, provider);

// 查询最近 100 个区块内的 Transfer 事件
const filter = contract.filters.Transfer;
const events = await contract.queryFilter(filter, -100);
events.length; // 319

// events[0] 示例结构：
// EventLog {
//   address: '0x6B175474E89094C44Da98b954EedeAC495271d0F',
//   args: Result(3) [
//     '0xC2e9F25Be6257c210d7Adf0D4Cd6E3E881ba25f8',
//     '0xfBd4cdB413E45a52E2C8312f670e9cE67E794C37',
//     370136547842778449664n
//   ],
//   blockHash: '0x9d343d0880968c61e2285ebbeecf02a73e535010cdcef97c4823a69bb78e57f8',
//   blockNumber: 23929363,
//   ...
// }

// 带条件过滤：查询转给 ethers.eth 的 Transfer
const filteredEvents = await contract.queryFilter(
  contract.filters.Transfer(null, "ethers.eth")
);
```

> 大范围区块查询可能被后端截断或报错，取决于 RPC 提供商。

## 消息签名

私钥可以签名任意数据，用于链下认证（如登录）：

```javascript
// 创建钱包（从测试 ID 生成私钥）
signer = new Wallet(id("test"));
// Wallet {
//   address: '0xC08B5542D177ac6686946920409741463a15dDdB',
//   provider: null
// }

// 签名消息
message = "sign into ethers.org?";
sig = await signer.signMessage(message);
// '0xefc6e1d2f21bb22b1013d05ecf1f06fd73cdcb34388111e4deec58605f366706...'

// 验证签名 → 恢复签名者地址
verifyMessage(message, sig);
// '0xC08B5542D177ac6686946920409741463a15dDdB'
```

更高级的协议（如 EIP-712、EIP-2612 permit）允许私钥授权他人代转代币，由第三方支付 gas。

## 与 Web3.js 的关键区别

Ethers 将 Provider（只读）和 Signer（写入）明确分离，而 Web3.js 的 Provider 同时承担读写。详见 [[libraries/sdk-comparison]]。

## 详见

- [[libraries/ethers-js]] — ethers.js 库概览与 DEX 用途
- [[concepts/provider-signer]] — Provider/Signer 架构概念
