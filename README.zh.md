# Atoshi 文档

本仓库是 **Atoshi Chain** 的官方权威技术文档。Atoshi Chain 是一条双层区块链，将基于 Cosmos SDK 的 L1(原生兼容 EVM)与基于 Polygon CDK 的 zkEVM L2 结合在一起,并围绕一套基于持币的能量模型设计,让长期持币者的日常交易体验如同免费一般。

如果你正在寻找:

| 读者 | 从这里开始 |
|---|---|
| 初次阅读者 | [00-overview.md](./00-overview.md) —— Atoshi 是什么、面向谁、有何创新 |
| 构建者 / 开发者 | [reference/api-guide.md](./reference/api-guide.md) —— REST + JSON-RPC 参考 |
| 钱包集成方 | [reference/wallet-integration.md](./reference/wallet-integration.md) |
| 研究员 / 分析师 | [economics/](./economics/) 与 [modules/](./modules/) |
| 运维方 / 验证人 | [architecture/](./architecture/) 与 [reference/node-operation.md](./reference/node-operation.md) |

## 目录

### 基础
- [00 —— 概览](./00-overview.md) —— 使命、设计目标、我们构建了什么
- [01 —— 高层架构](./architecture/01-overview.md) —— L1/L2/跨层桥的整体心智模型

### 架构
- [带 EVM 的 Cosmos SDK L1](./architecture/02-cosmos-l1.md)
- [Polygon CDK zkEVM L2](./architecture/03-polygon-l2.md)
- [L1 与 L2 之间的跨层桥](./architecture/04-bridge.md)
- [账户模型与密钥管理](./architecture/05-accounts.md)

### 模块(每一篇都是自成体系的章节)
- [Energy —— 靠持币而非付费获得 gas](./modules/01-energy.md)
- [Tokenomics —— 供应、释放、减半](./modules/02-tokenomics.md)
- [Oracle —— 链上 ATOS/USD 价格](./modules/03-oracle.md)
- [Bank —— ATOS 原生转账](./modules/04-bank.md)
- [EVM —— 兼容以太坊的执行环境](./modules/05-evm.md)
- [Bridge —— 跨层资产转移](./modules/06-bridge.md)
- [Privacy —— L2 上的隐私交易](./modules/07-privacy.md)
- [Staking —— 验证人经济学](./modules/08-staking.md)
- [Governance —— 链上提案及案例研究](./modules/09-governance.md)

### 经济学
- [供应与流通](./economics/01-supply.md)
- [释放计划与减半曲线](./economics/02-release-schedule.md)
- [Gas 与能量经济学](./economics/03-gas-fee.md)
- [区块奖励与验证人收入](./economics/04-block-rewards.md)
- [销毁与回收机制](./economics/05-burn-recycle.md)

### 隐私深度解析
- [威胁模型与设计目标](./privacy/01-threat-model.md)
- [隐私池 —— 存入、转账、提取](./privacy/02-shield-pool.md)
- [ZK 电路与证明系统](./privacy/03-zk-circuits.md)
- [隐私交易的能量结算](./privacy/04-energy-settlement.md)

### 参考
- [REST + JSON-RPC API 参考](./reference/api-guide.md)
- [钱包集成手册](./reference/wallet-integration.md)
- [节点运维指南](./reference/node-operation.md)
- [治理操作手册](./reference/governance-runbook.md) —— 提交提案、参数变更、软件升级
- [术语表](./reference/glossary.md)
- [常见问题](./reference/faq.md)

## 版本管理

本文档跟随 `atoshi-chain` 的 `main` 分支。当链发布主版本时,本仓库会打上与之匹配的版本标签。旧版文档保存在对应的标签中。

## 贡献

发现内容有误、过时或不清晰?提交一个 PR —— 每个页面页脚都有 `Last reviewed:` 日期,修改内容时请一并更新。若涉及较大的结构性改动,请先提一个 issue。

## 许可证

文档以 CC-BY-SA 4.0 发布。文档中内嵌的代码片段以 MIT 发布。
