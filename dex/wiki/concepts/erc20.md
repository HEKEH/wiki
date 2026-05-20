---
title: ERC-20 与 Approve 机制
date: 2026-05-18
tags: [erc20, token, approve, ethereum, basics]
sources: []
---

# ERC-20 与 Approve 机制

ERC-20 是以太坊上最常用的代币标准。DEX 上交易的绝大多数代币都是 ERC-20。

## ERC-20 是什么？

一个**接口规范**——所有 ERC-20 代币都必须实现以下方法：

| 方法 | 类型 | 说明 |
| ---- | ---- | ---- |
| `name()` | view | 代币名称，如 "USD Coin" |
| `symbol()` | view | 代币符号，如 "USDC" |
| `decimals()` | view | 精度，通常 18（同 ETH）或 6（USDC/USDT） |
| `totalSupply()` | view | 总发行量 |
| `balanceOf(address)` | view | 查询某地址的余额 |
| `transfer(to, amount)` | 写入 | 直接转账 |
| `approve(spender, amount)` | 写入 | 授权他人使用你的代币 |
| `allowance(owner, spender)` | view | 查询已授权额度 |
| `transferFrom(from, to, amount)` | 写入 | 被授权者代为转账 |

## 为什么 ETH 不需要 Approve，ERC-20 需要？

这是新人最容易困惑的地方：

**ETH 是原生资产**，转账是区块链底层操作，`sendTransaction` 直接把 ETH 从 A 移到 B。

**ERC-20 是合约管理的资产**，你的"余额"只是合约存储中的一个数字。当你想用 DEX swap 时，是 DEX 合约要**代替你**操作你的代币——它需要先获得你的授权。

## Approve 机制详解

这是 DEX 交互中最核心、也是最容易出错的环节：

```
第 1 步：Approve（授权）
你 → ERC-20 合约："我允许 Uniswap 合约动用我 1000 USDC"
                   contract.approve(uniswapAddress, parseUnits("1000", 6))

第 2 步：Swap（交易）
你 → Uniswap 合约："帮我把 100 USDC 换成 ETH"
                   Uniswap 合约内部调用 → USDC.transferFrom(你的地址, 池子地址, 100 USDC)
```

如果没有先 approve，swap 会失败并 revert，你白白损失 gas 费。

## Approve 的安全注意事项

| 风险 | 说明 | 建议 |
| ---- | ---- | ---- |
| **无限授权** | approve 金额设为最大值，合约可随时转走所有该代币 | 只授权需要的金额 |
| **恶意合约** | approve 给钓鱼合约，代币被盗 | 只 approve 你确认安全的合约 |
| **授权残留** | swap 后授权额度仍有剩余 | 交易后可调用 `approve(spender, 0)` 清零 |

## decimals 为什么重要？

不同代币精度不同，计算时必须考虑：

| 代币 | decimals | 1 个代币 = 内部表示 |
| ---- | -------- | ------------------- |
| WETH | 18 | 1000000000000000000 |
| USDC | 6 | 1000000 |
| USDT | 6 | 1000000 |
| WBTC | 8 | 100000000 |

```javascript
// 正确：用 parseUnits 自动处理精度
amount = parseUnits("100", await contract.decimals());

// 错误：假设所有代币都是 18 位精度
amount = parseUnits("100", 18);  // USDC/USDT 会出错！
```

## 代码示例：完整的 DEX swap 流程

```javascript
// 1. 连接
const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

// 2. 实例化合约
const usdc = new Contract(USDC_ADDRESS, ERC20_ABI, signer);
const router = new Contract(ROUTER_ADDRESS, ROUTER_ABI, signer);

// 3. Approve
const amount = parseUnits("100", 6);
let tx = await usdc.approve(ROUTER_ADDRESS, amount);
await tx.wait();  // 等待上链

// 4. Swap
tx = await router.swapExactTokensForTokens(amount, minOut, path, to, deadline);
await tx.wait();
```

## 详见

- [[concepts/smart-contract]] — 智能合约基础
- [[concepts/dex-overview]] — DEX 中的 swap 流程
- [[concepts/amm]] — AMM 与流动性池（ERC-20 代币的定价场景）
- [[concepts/provider-signer]] — 读写分离与 staticCall
- [[concepts/token-standards]] — 其他代币标准（ERC-721/1155 等）与跨链技术栈
