---
title: MetaMask
date: 2026-05-18
tags: [metamask, wallet, browser-extension, hot-wallet]
sources: []
---

# MetaMask

最广泛使用的以太坊浏览器钱包插件，由 ConsenSys 开发。

## 核心功能

| 功能 | 说明 |
| --- | --- |
| **账户管理** | 生成和存储私钥、管理多个账户 |
| **链上交互** | 通过 `window.ethereum` 向网页暴露 Provider 和 Signer |
| **交易签名** | 弹窗让用户确认每笔交易 |
| **多链支持** | Ethereum、Polygon、BSC、Arbitrum 等 EVM 链 |
| **代币管理** | 自动或手动添加 ERC-20/ERC-721 代币显示 |

## 安全模型

MetaMask 是**热钱包**——私钥存在联网设备的浏览器中。

```
网页 ←── window.ethereum（受限通信接口）──→ MetaMask 插件（独立沙盒）
                                              │
                                              私钥加密存储在 localStorage
                                              解锁密码由用户设置
```

三层防护：

1. **沙盒隔离**：网页无法直接访问私钥，只能通过接口发请求
2. **密码加密**：私钥加密后存储，需要解锁密码才能使用
3. **弹窗确认**：每笔交易、每次签名都要用户手动确认

## 代码中的使用

```javascript
// 检测是否安装
if (window.ethereum == null) {
  // 未安装，提示用户安装 MetaMask
}

// 创建 Provider 和 Signer
const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();  // 首次调用会弹窗请求授权

// 请求切换网络
await window.ethereum.request({
  method: "wallet_switchEthereumChain",
  params: [{ chainId: "0x1" }]  // 切到主网
});
```

## 安全边界与局限

| 前提 | 安全 | 不安全 |
| --- | --- | --- |
| 电脑未被入侵 | 沙盒隔离 + 弹窗确认足够安全 | — |
| 电脑被黑客控制 | — | 键盘记录密码、读 localStorage、篡改交易内容 |

**MetaMask 的安全前提是"你的电脑是安全的"。** 大额资产应使用 [[entities/hardware-wallet]]。

## 与 DEX 开发的关系

几乎所有 DEX 前端的第一步就是连接 MetaMask：

```javascript
// DEX 标准连接模式
const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();
const address = await signer.getAddress();
```

## 详见

- [[entities/hardware-wallet]] — 硬件钱包：更高安全级别的方案
- [[concepts/wallet-types]] — 钱包类型与方案总览
- [[concepts/provider-signer]] — Provider/Signer 架构
- [[concepts/ethereum-basics]] — 私钥、地址基础概念
