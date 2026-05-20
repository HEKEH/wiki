# DEX Knowledge Base — Activity Log

Append-only chronological record of wiki activity.

## [2026-05-18] ingest | Ethers.js Getting Started

- 源文件：`raw/ethers-js/Ethers-Getting-Started.md`
- 新建页面：`sources/ethers-getting-started`、`libraries/ethers-js`、`concepts/provider-signer`
- 更新：`home.md`、`index.md`
- 定制：`CLAUDE.md`（定义 DEX 域、5 个分类目录）

## [2026-05-18] ingest | 新人基础概念补充

- 新建 5 个概念页面：
  - `concepts/ethereum-basics` — 区块链/以太坊基础（地址、私钥、交易、测试网）
  - `concepts/gas` — Gas 机制（费用构成、EIP-1559、DEX 注意事项）
  - `concepts/smart-contract` — 智能合约（Solidity、ABI、读写模式）
  - `concepts/erc20` — ERC-20 与 Approve 机制（授权流程、安全注意事项、decimals）
  - `concepts/dex-overview` — DEX 概览（AMM vs 订单簿、swap 流程、合约架构）
  - `concepts/amm` — AMM 与流动性池（恒定乘积、滑点、集中流动性）
- 更新：`home.md`（添加学习路径）、`index.md`（注册新页面）

## [2026-05-18] ingest | MetaMask 与硬件钱包

- 新建 2 个实体页面：
  - `entities/metamask` — MetaMask 热钱包（安全模型、沙盒隔离、代码使用、局限）
  - `entities/hardware-wallet` — 硬件钱包（工作原理、热钱包 vs 冷钱包对比、主流产品、DEX 集成）
- 更新：`index.md`（注册新页面）

## [2026-05-18] ingest | 钱包类型与方案

- 新建页面：`concepts/wallet-types` — 浏览器插件/嵌入式钱包/WalletConnect/账户抽象四种模式对比
- 更新：`index.md`

## [2026-05-18] ingest | 代币标准、跨链技术栈、SDK 对比

- 新建 2 个页面：
  - `concepts/token-standards` — 代币标准（ERC-20/721/1155/4626）、其他链同类标准、EVM vs 非 EVM 技术栈差异、同链跨协议复用规律
  - `libraries/sdk-comparison` — EVM SDK 对比（ethers.js vs viem vs web3.js）、web3.js ≠ @solana/web3.js 说明
- 更新：`concepts/erc20`（添加交叉链接）、`concepts/ethereum-basics`（添加 EVM/非 EVM 说明）、`index.md`

## [2026-05-18] ingest | 补全源文代码与 MPC 恢复机制

- 重写 `sources/ethers-getting-started`：补全链上查询、发送交易、合约只读/写入/staticCall/事件监听/历史事件查询/消息签名的完整代码示例，补全 ABI 格式说明、EventLog 结构、VoidSigner 模拟
- 重写 `libraries/ethers-js`：补全安装导入、连接链、查询状态、发送交易、合约交互（创建/只读/写入/staticCall/事件/历史查询）、单位转换、消息签名的完整代码
- 更新 `concepts/wallet-types`：补充 MPC 分片恢复机制（Web3Auth 2/3 门限、新设备恢复流程、Privy/Magic Link 差异）
