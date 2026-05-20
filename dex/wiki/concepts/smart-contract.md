---
title: 智能合约
date: 2026-05-18
tags: [smart-contract, solidity, ethereum, basics]
sources: []
---

# 智能合约

智能合约是部署在以太坊上的**程序**，代码和数据都存在链上，一旦部署就不可修改（除非设计了可升级模式）。

## 核心特性

| 特性 | 说明 |
| ---- | ---- |
| **确定性** | 相同输入永远产生相同输出，所有节点执行结果一致 |
| **不可篡改** | 部署后代码无法修改，规则透明可审计 |
| **自动执行** | 条件满足时自动执行，不需要中间人 |
| **公开可见** | 任何人可以读取合约代码和状态 |

## 用 Solidity 编写

Solidity 是以太坊智能合约最主流的语言：

```solidity
// 一个极简的代币合约
contract SimpleToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "余额不足");
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

## ABI — 合约的"使用说明书"

你不需要读懂合约的 Solidity 源码才能使用它。**ABI（Application Binary Interface）** 定义了合约对外暴露的方法、参数和返回值。

```javascript
// 人类可读的 ABI 格式
const abi = [
  "function balanceOf(address owner) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 amount)"
];
```

有了 ABI + 合约地址，ethers.js 就能帮你调用合约：

```javascript
const contract = new Contract(address, abi, provider);  // 只读
const contract = new Contract(address, abi, signer);    // 可读写
```

## 合约交互的两种模式

```
                    ┌─── view / pure ──→ 不花 gas，本地执行 ──→ 立即返回结果
                    │
读取链上数据 ───────┤
                    │
                    └─── call（eth_call）→ 不花 gas，向节点请求 ──→ 返回结果

                    ┌─── 非 view/pure ──→ 花 gas，广播交易 ──→ 等待上链 ──→ 拿到回执
                    │
改变链上状态 ───────┤
                    │
                    └─── staticCall ──→ 不花 gas，模拟执行 ──→ 检查是否会 revert
```

## 常见的合约类型（DEX 相关）

| 类型 | 作用 | 例子 |
| ---- | ---- | ---- |
| **ERC-20** | 代币标准 | USDT、UNI、DAI |
| **ERC-721** | NFT 标准 | CryptoPunks |
| **DEX Pair** | 交易对合约 | Uniswap V2 Pair |
| **DEX Router** | 路由合约 | Uniswap V2 Router02 |
| **Vault** | 资金池 | Curve Vault |

## 详见

- [[concepts/erc20]] — ERC-20 代币标准与 Approve 机制
- [[concepts/dex-overview]] — DEX 中的合约架构
- [[concepts/provider-signer]] — Provider/Signer 与合约交互
