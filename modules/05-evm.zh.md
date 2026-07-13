# `x/evm` —— 以太坊兼容的执行

Atoshi 的 L1 通过来自 ethermint 技术栈的 `x/evm` 模块,在 Cosmos SDK
状态机内部承载了一个完整的以太坊虚拟机。EVM
交易与 Cosmos 交易在同一批区块中执行,并读取
相同的银行余额——在 L1 上不存在 ATOS 的封装表示。

## 心智模型

两种不同的交易编码,一套共享状态:

```
                  Cosmos tx                 Ethereum tx
              (signed protobuf)         (signed RLP, type 0/2)
                     │                          │
                     ▼                          ▼
       ┌─────────────────────────┐  ┌──────────────────────────┐
       │ ante.EnergyDeductDecor. │  │ ante.MonoDecorator       │
       │ (uses energy, then ATOS)│  │ (uses ATOS gas only)     │
       └─────────────────────────┘  └──────────────────────────┘
                     │                          │
                     ▼                          ▼
            ┌────────────────────────────────────────────┐
            │            Shared state machine             │
            │   x/bank balances · x/staking · x/energy    │
            │              x/evm · x/erc20                │
            └────────────────────────────────────────────┘
```

一个持有单个私钥的钱包用户,既可以向同一个目标地址发送 Cosmos 的
`MsgSend`,也可以发送以太坊的 `eth_sendRawTransaction`——
产生的银行状态更新是完全相同的。相关工具(MetaMask、Keplr、
ethers.js、atoshid CLI)是完全跨兼容的。

## Atoshi 改动了什么

`x/evm` 取自 ethermint,并做了如下适配:

- **Chain id 注册表**。`x/evm/types/denom.go` 注册了三个有效的
  Atoshi chain id(`atoshi_88188`、`atoshi_88288`、`atoshi_88388`);任何
  其他 chain id 都会在启动时 panic。这可以防止意外地
  用某个外部链的创世来启动该二进制。
- **Base denom**:aatos(主网、测试网、testing 在一次
  bug 修复后已全部统一;此前测试网被设置为 `ataatos`,导致
  `eth_getBalance` 出错)。
- **ERC20 自动配对**:aatos 在确定性地址
  `0xD4949664cD82660AaE99bEdc034a0deA8A0bd517` 上注册为一个 ERC20 代币对,
  使 EVM 合约能够对原生 ATOS 调用 `transferFrom` / `approve`。
- **Precompile 注册表**:标准 ethermint 预编译合约(staking、
  distribution、ICS-20)位于 `0x0000…0800-0805`,用于跨模块的 EVM
  交互。

## 为什么能量绕过 EVM(以及这对用户意味着什么)

`x/energy.EnergyDeductDecorator` 这个 ante-handler **只拦截
Cosmos 消息**。EVM 交易经由 ethermint 的
`MonoDecorator`,它有自己的一套 gas 记账,并且被有意
不加修改。

实际后果:

- 一个在 EVM 转账上支付 gas 的钱包用户,按当前的
  `feemarket.MinGasPrice`(目前为 1 gwei = 10⁹ aatos/gas)支付标准的 ATOS gas。
  一笔 21,000-gas 的原生转账花费约 0.000021 ATOS。
- 同一个用户可以通过 Cosmos 的 `MsgSend` 发送同样的 ATOS,并
  支付零 ATOS 手续费(由能量覆盖)——只要其持仓给了
  他足够的能量容量。

我们在 v1 中有意选择了这条边界:

- ethermint 的 `MonoDecorator` 经过充分测试,修改它会增加风险
- EVM 生态中的钱包和 SDK 期望可预测的 gas 定价
  语义,而不是一个无法映射到 EIP-1559 的
  "持有足够就免费"的模型
- 将能量集成进 EVM 执行需要触碰 `ApplyMessage`,
  并且会使多年的以太坊安全工作作废

未来的 Atoshi 版本可能会放宽这一点。目前:**Cosmos 路径 = 能量
优惠;EVM 路径 = 标准 gas。**

## 状态

`x/evm` 存储:

- 代码哈希 → 字节码(大块数据,按内容寻址)
- 账户 → 代码哈希(从 EVM 地址到已部署合约代码的映射)
- 存储槽:`address || slot_key` → 32 字节值(EVM 的 SSTORE 存储)
- 瞬态状态(交易内,在 FinalizeBlock 时清除)
- 创世参数(预编译列表、allow/deny 访问控制)

EOA 余额**不**存储在 `x/evm` 中——当某个 EVM opcode 请求
`balance(addr)` 时,它们从 `x/bank` 实时读取。

## 消息

```
MsgEthereumTx {
  data: bytes                       // RLP-encoded signed Ethereum tx
}

MsgUpdateParams {                   // gov-only
  authority: string
  params: Params { ... }
}
```

`MsgEthereumTx` 将一笔标准的以太坊签名交易包裹在一个 Cosmos
消息信封中,使其能够像其他消息一样经过相同的 mempool / ABCI
流水线传输。Cosmos 签名者为空;从 RLP 签名中恢复出的
EVM 签名者才是授权该交易的主体。

类型空间中存在少数遗留 / 管理类消息
(`MsgCreateContract`、`MsgEthCallEntryPoint`),但在实践中
不被使用——客户端总是通过 `eth_sendRawTransaction` 提交,
其结果是一个 `MsgEthereumTx`。

## EVM JSON-RPC

该链在 8545 端口上暴露一个标准的以太坊 JSON-RPC 服务器。所有
读方法都可用——`eth_getBalance`、`eth_getStorageAt`、
`eth_getTransactionByHash`、`eth_getLogs`、`eth_call`、`eth_estimateGas`、
`eth_chainId` 等。

写方法仅限于 `eth_sendRawTransaction`。那些
隐含链侧密钥管理的方法(`eth_sendTransaction`、
`personal_sign`)被禁用——钱包必须在本地签名并提交原始交易。

在 `app.toml` 中配置后,通过 `eth_subscribe` 的订阅可在同一端口上
经由 WebSocket 工作。

## Gas

EVM 侧的 gas 定价由独立的 `x/feemarket`
模块治理(EIP-1559 base fee + 最低 gas 价格)。关键参数:

| 参数 | 当前值 | 说明 |
|---|---|---|
| `min_gas_price` | 10⁹ aatos/gas(1 gwei) | 下限;链拒绝低于此值的交易 |
| `base_fee` | 每区块动态 | EIP-1559 base fee,根据区块饱和度调整 |
| `base_fee_change_denominator` | 8 | 标准以太坊取值 |
| `elasticity_multiplier` | 2 | 区块可达目标 gas 用量的 2 倍 |
| `min_gas_multiplier` | 0.5 | 即使实际用量更低,也至少按 gas_limit 的 50% 收费 |
| `no_base_fee` | false | EIP-1559 已启用 |

调用 `eth_gasPrice` 的钱包应将结果视为一个推荐的
gas 价格,但需回退到一个合理的最小值(≥ 1 gwei),以防
因近期活动量低而返回的值为 0。

## EVM ↔ Cosmos 交叉场景

| 动作 | 执行位置 | 结算于 |
|---|---|---|
| 通过 MetaMask 转账原生 ATOS | EVM | bank(aatos 余额更新) |
| 通过 atoshid `tx bank send` 转账原生 ATOS | Cosmos | bank(相同的余额更新) |
| 转账某个 Cosmos 原生 denom 的 ERC20 | 经 `x/erc20` 自动配对的 EVM | bank(denom 余额更新) |
| 通过 MetaMask 质押 | EVM 预编译 `0x0000…0800` | x/staking |
| 桥接 L1 → L2 | EVM(bridgeAsset 调用) | 桥合约持有托管 |

`x/erc20` 是跨模式访问的关键枢纽:每一个原生 Cosmos
denom(IBC 代币、链自身的 ATOS)都会在一个确定性地址上
自动部署一个 ERC20 合约,可从任何 EVM 上下文中调用。

## 边界情形

### Type 0 与 Type 2 交易

该链接受 legacy(type 0)和 EIP-1559(type 2)两种
交易。由于 sequencer 的配置,L2 目前只接受 type 0
——参见 [modules/06-bridge.md](./06-bridge.md)。想要跨层
兼容的钱包应默认使用 type 0。

### nonce 语义

EVM nonce 从 `x/auth.Account.Sequence` 读取——即由 Cosmos 消息
递增的同一个计数器。因此一个混合发送 Cosmos +
EVM 交易的钱包看到的是单一的单调递增计数器。

这避免了一类隐患:两个并行的 nonce 空间(一个
用于 Cosmos,一个用于 EVM)可能导致间隙和交易卡住。

### 成功区块中失败的 EVM 交易

如果一笔 EVM 交易回滚(out-of-gas、REVERT opcode、合约
panic),gas 仍会被消耗并收取。区块照常推进,
交易以 `status: 0x0` 被包含。查询
`eth_getTransactionReceipt` 的钱包应始终检查 `status`——
一笔失败的交易是有可能被"成功包含"的。

### 禁用的访问控制

`evm.params.access_control` 允许治理限制谁可以调用
`eth_call`(READ)或部署合约(CREATE)。该链目前
以两者都为 `ACCESS_TYPE_PERMISSIONLESS` 运行。如果治理曾切换
到白名单模式,则需要添加预期的钱包。

## 事件

EVM 事件以两种并行形式呈现:

1. **EVM 日志** —— 可通过 `eth_getLogs`、`eth_getTransactionReceipt`
   及标准 EVM 工具读取。按 topic 建索引。
2. **Cosmos 事件** —— 同一份数据也以 `cosmos.evm.v1.*` 类型化
   事件的形式发出,可通过 Tendermint 的 tx-search 在
   `/cosmos/tx/v1beta1/txs?query=...` 查询。

两种视图展示的是相同的日志;根据你使用的客户端
来选择。

## 参数

| 名称 | 默认值 | 说明 |
|---|---|---|
| `allow_unprotected_txs` | false | 若为 true,接受 EIP-155 未保护(Spurious Dragon 之前)的交易 |
| `evm_denom` | "aatos" | 原生 gas denom |
| `access_control.call.access_type` | PERMISSIONLESS | `eth_call` 白名单 |
| `access_control.create.access_type` | PERMISSIONLESS | 合约部署白名单 |
| `evm_channels` | `["channel-10", "channel-31", "channel-83"]` | 被授权接收 EVM 可调用资产的 IBC 通道 |
| `extra_eips` | `["ethereum_3855"]` | 激活 EIP-3855(`PUSH0`)及类似的可选 EIP |
| `active_static_precompiles` | 8 条 | 位于 `0x100`(modexp)、`0x400`(bls)、`0x800-0x805`(Cosmos 集成)的预编译合约 |

## 相关模块

- [`x/bank`](./04-bank.md) —— 为 EOA 余额提供支撑;读/写都经过这里
- [`x/erc20`](#) —— 将 Cosmos denom 桥接到 ERC20 合约
- [`x/feemarket`](#) —— 拥有 EVM 的 gas 定价
- [`x/energy`](./01-energy.md) —— 不覆盖 EVM 交易(有意为之)
- [`x/evidence`、`x/staking`](#) —— 通过预编译暴露,供 EVM
  合约交互

## 延伸阅读

- [Ethermint 文档](https://docs.evmos.org/protocol/concepts/chain-id)
- [钱包集成手册](../reference/wallet-integration.md) —— 构建和签名
  Cosmos 与以太坊两种交易的实用示例

---

*最后审阅:2026-06-10*
