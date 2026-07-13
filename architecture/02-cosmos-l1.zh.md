# 带 EVM 的 Cosmos SDK L1

> **状态:草稿提纲。** 本章将完整阐述 L1 链。标记为 TODO 的小节是待下一轮迭代补充的占位内容。

## L1 在运行时的样子

一条采用 Cosmos SDK v0.50 的链,搭载 ethermint EVM 模块、ABCI++、约 5 秒出块、PoS 验证人。创世时配置了一组由基金会运营的验证人集合;后续可通过标准的 `MsgCreateValidator` 无需许可地加入验证人。

## 模块列表(按字母排序)

| 模块 | 来源 | 用途 |
|---|---|---|
| `x/auth` | SDK | 账户 / 序列号 / 公钥存储;ethermint `EthAccount` 扩展 |
| `x/authz` | SDK | 基于授权的委托授权(v1 中未使用,但可用) |
| `x/bank` | SDK | ATOS 转账、余额查询、供应量跟踪 |
| `x/distribution` | SDK | 验证人佣金 + 区块奖励分发 |
| `x/energy` | atoshi | 基于持币的 gas 系统。参见 [Energy 模块](../modules/01-energy.md) |
| `x/epochs` | evmos | 基于周期的钩子调度器(在 evmos 中被 inflation/erc20 使用;此处用途极少) |
| `x/erc20` | evmos | 原生代币 ↔ ERC20 转换路径 |
| `x/evm` | ethermint | EVM 执行层 |
| `x/feemarket` | ethermint | 面向 EVM 的 EIP-1559 base-fee 跟踪 |
| `x/gov` | SDK | 链上治理提案 |
| `x/ibc` | SDK | 区块链间通信(v1 中处于被动状态) |
| `x/inflation` | evmos | 在 atoshi 中已禁用 —— 不通过通胀增发 |
| `x/oracle` | atoshi | ATOS/USD 价格预言机。参见 [Oracle 模块](../modules/03-oracle.md) |
| `x/slashing` | SDK | 验证人作恶惩罚 |
| `x/staking` | SDK | 质押 / 委托 / 解绑 |
| `x/tokenomics` | atoshi | 区块奖励发放 + 按档位驱动的释放。参见 [Tokenomics 模块](../modules/02-tokenomics.md) |
| `x/upgrade` | SDK | 协调式链升级 |

## Ante-handler 流水线

ante-handler 是 atoshi 相对于原生 Cosmos 链偏离最大的地方。流水线(按顺序):

1. `SetUpContextDecorator` —— gas 计量器初始化
2. `RejectExtensionOptionsDecorator` —— 安全:拒绝未知的交易扩展
3. `MempoolFeeDecorator` —— 仅在能量路径之外生效;最低 gas 价格检查
4. `ValidateBasicDecorator` —— 消息级校验
5. `TxTimeoutHeightDecorator` —— `timeout_height` 检查
6. `ValidateMemoDecorator` —— memo 长度上限
7. `ConsumeTxSizeGasDecorator` —— 按交易字节大小收取 gas
8. **`EnergyDeductDecorator`** —— atoshi 对 `DeductFeeDecorator` 的替代实现。参见 [Energy 模块 §消费](../modules/01-energy.md#consumption-the-ante-handler-pipeline)。
9. `SetPubKeyDecorator`、`ValidateSigCountDecorator`、`SigGasConsumeDecorator`、`SigVerificationDecorator` —— 签名处理
10. `IncrementSequenceDecorator` —— 序列号自增

对于 EVM 交易(以 `MsgEthereumTx` 包装),会改由 ethermint 的独立 `MonoDecorator` 接管,且**不**查询能量模块。相关影响参见 [Gas 经济学](../economics/03-gas-fee.md)。

## 区块生命周期

- **BeginBlock**:`x/tokenomics` 增发区块奖励,将其中 20% 的即时份额分发给出块者,把 80% 累积到矿工锁定池中。随后检查各档位条件,并可能发出一个档位释放事件。`x/distribution` 完成验证人佣金核算。
- **DeliverTx** 循环:每笔交易先流经 ante-handler,再进入消息处理器。
- **EndBlock**:`x/energy.SweepExpiredDelegations` 释放已过期的委托。`x/staking` 处理解绑队列。`x/slashing` 更新掉线计数器。

## TODO 小节

- L1 端点汇总(REST、RPC、EVM JSON-RPC)—— 与 [API 参考](../reference/api-guide.md) 有重叠
- 创世状态结构与验证人密钥处理
- 使用 cosmosvisor 的升级工作流
- 快照 + statesync 配置
- 哨兵-验证人(sentry-validator)架构建议

---

*最后审阅:2026-05-21*
