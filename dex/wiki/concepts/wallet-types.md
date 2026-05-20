---
title: 钱包类型与方案
date: 2026-05-18
tags: [wallet, metamask, embedded-wallet, walletconnect, account-abstraction]
sources: []
---

# 钱包类型与方案

DEX 前端连接用户的方式不止 MetaMask 一种，不同方案在安全性、易用性、控制权之间做不同取舍。

## 四种主要模式

### 1. 浏览器插件钱包

通过 `window.ethereum` 注入，代码层面互通。

| 钱包 | 特点 |
| --- | --- |
| MetaMask | 最广泛使用，生态最成熟 |
| Rabby Wallet | 安全提示更强，签名前显示风险警告 |
| Coinbase Wallet | Coinbase 出品，L2 支持好 |
| OKX Wallet（插件版） | 支持 60+ 链，内置跨链桥，DeFi 功能丰富 |
| Phantom | 最初 Solana 生态，现已支持 EVM |

开发者无需区分，代码完全一样：

```javascript
const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();
```

安全模型：私钥在浏览器沙盒中，用户自托管。详见 [[entities/metamask]]。

### 2. 嵌入式钱包（Embedded Wallet）

用户不需要装插件，用邮箱/社交账号登录即可交易。降低门槛，适合新用户。

| 方案 | 原理 | 用户体验 |
| --- | --- | --- |
| Privy | 社交登录 + MPC 分片 | 像 Web2 一样登录 |
| Magic Link | 邮件登录，私钥服务端托管 | 收邮件 → 点链接 → 登录 |
| Web3Auth | OAuth 登录 + 门限签名 | Google/Twitter 登录 |
| Sequence | 智能合约钱包 + 社交恢复 | 游戏场景友好 |
| OKX Web 端 | 账号登录，私钥加密存储在 OKX 服务器 | 登录即可交易，无需装插件，登录密码 + 2FA 解锁 |

```javascript
// Privy 示例：嵌入式钱包集成
import { PrivyProvider } from "@privy-io/react-auth";

function App() {
  return (
    <PrivyProvider appId="your-app-id">
      <YourDexApp />
    </PrivyProvider>
  );
}
```

安全模型：私钥被拆分成分片，分布在你和服务商之间（半托管）。

#### MPC 分片恢复机制

以 Web3Auth（tKey）为例，私钥被拆成 3 个分片，任意 2 个即可恢复签名能力：

```text
私钥 = 认证分片 + 设备分片 + 备份分片（2/3 门限）
       │           │           │
       │           │           └── 备份分片（助记词/云存储/社交恢复）
       │           └── 设备分片（存在当前设备本地）
       └── 认证分片（由社交账号 OAuth 令牌推导）
```

**新设备恢复流程：**

1. 登录社交账号 → 获得认证分片
2. 新设备没有设备分片 → 需要第三个分片补位
3. 通过备份助记词 / 云端恢复（iCloud/Google Drive）/ 社交恢复 → 获得备份分片
4. 认证分片 + 备份分片 → MPC 签名 ✓
5. 新设备生成新的设备分片，以后只需认证分片 + 设备分片

不同方案的实现差异：

| 方案 | 分片策略 | 新设备恢复方式 |
| --- | --- | --- |
| **Web3Auth** | OAuth 分片 + 设备分片 + 备份分片（2/3 门限） | 登录社交账号 + 备份分片 |
| **Privy** | 认证令牌加密存储私钥 | 重新登录同一社交账号即可 |
| **Magic Link** | 私钥完全由 Magic 服务器管理（托管） | 邮件验证身份后服务器解密 |

**核心价值：用邮箱/社交账号替代必须安全保管的助记词。** MetaMask 丢助记词 = 资产永久丢失；嵌入式钱包可通过社交恢复找回。

### 3. WalletConnect（手机扫码）

手机钱包通过扫码连接电脑上的 DEX，在手机上确认签名。

```
电脑浏览器（DEX 网页）──扫码连接──→ 手机钱包 App
                                      │
                                   在手机上确认签名
                                   私钥始终在手机上
```

支持的钱包：Rainbow、Trust Wallet、MetaMask Mobile、OKX App 等。

安全模型：私钥在手机端，电脑只发请求。比电脑上的插件钱包略安全（手机被黑概率更低），但本质还是热钱包。

### 4. 账户抽象（ERC-4337）

把账户从私钥绑定的 EOA 变成智能合约，实现传统钱包无法做到的功能：

| 功能 | 说明 |
| --- | --- |
| Gas 代付 | 项目方代付 gas，用户不需要持有 ETH |
| 社交恢复 | 好友可以帮你找回钱包，不再依赖助记词 |
| 批量操作 | approve + swap 一笔交易完成 |
| 权限管理 | 设置每日转账限额、session key |

代表项目：Safe、Biconomy、Alchemy Smart Wallet、Coinbase Smart Wallet。

## 安全性对比

| 模式 | 私钥控制 | 被黑风险 | 忘记密码 | 适合人群 |
| --- | --- | --- | --- | --- |
| 浏览器插件 | 用户完全控制 | 电脑被黑则失守 | 不可恢复，靠助记词 | 有经验的用户 |
| 嵌入式钱包 | 半托管（MPC 分片） | 邮箱 + 服务商同时被攻破 | 可通过邮箱/社交恢复 | 新用户、小额 |
| WalletConnect | 用户完全控制（手机端） | 手机被黑则失守 | 不可恢复，靠助记词 | 习惯手机操作的用户 |
| [[entities/hardware-wallet\|硬件钱包]] | 用户完全控制（离线芯片） | 物理设备被盗 + PIN 被破解 | 助记词备份恢复 | 大额资产存储 |

## DEX 前端开发的现状

现代 DEX 前端通常同时支持多种连接方式：

```javascript
// 典型连接选项
const connectors = [
  injected(),           // MetaMask 等浏览器插件
  walletConnect(),      // 手机扫码
  coinbaseWallet(),     // Coinbase Wallet
  // + Privy/Magic Link 等嵌入式方案
];
```

## 详见

- [[entities/metamask]] — MetaMask 详解
- [[entities/hardware-wallet]] — 硬件钱包详解
- [[concepts/ethereum-basics]] — 私钥与地址基础
