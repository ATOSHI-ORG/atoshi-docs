# `x/bank` —— 原生代币转账与余额

Cosmos SDK 的银行模块拥有 ATOS 余额与转账。Atoshi 基本上按原样使用它,
但增加了一个影响深远的钩子:来自能量模块的 `SendRestriction` 回调,
它让每个账户的能量快照与其实际银行余额保持同步。

## 心智模型

无论 ATOS 由哪个模块创建,它都存放在 `x/bank` 中。模块账户
(能量锁仓池、tokenomics 池、fee_collector、gov)
都在这里持有它们的余额。银行模块能看到链上的每一笔贷记和借记
——这正是它成为任何附带副作用的天然挂钩点的原因。

## Atoshi 改动了什么

基础 SDK 银行模块未作修改。唯一的 Atoshi 定制在应用装配层面:
在 `app/app.go` 中,我们在银行 keeper 上注册了一个额外的
`SendRestriction` 回调。

```go
app.BankKeeper.AppendSendRestriction(
    app.EnergyKeeper.SendRestriction,
)
```

能量 keeper 的 `SendRestriction` 会在每次发送时、在转账提交之前
被银行调用。它做两件事:

1. **快照刷新。** `from` 和 `to` 两个账户的
   `EnergyAccount.LastBalanceSnapshot` 都会被重置为其转账后的
   合格余额。这是 v20.1 交付的修复——没有这个钩子,
   接收方的快照会停留在其初始值,而累积的容量
   永远无法反映后来的流入。
2. **锁定 ATOS 强制约束。** 如果 `from` 存在未了结的能量
   委托,其银行余额中支撑这些委托的那一部分
   (在 `EnergyAccount.LockedAtos` 中跟踪)会被视为
   不可花费。试图发送超过 `balance - locked_atos` 的金额
   会导致转账被拒绝。

该钩子**不会**修改转账金额、参与方,或成功后的
后置状态。它只会:

- 作为副作用更新能量模块存储,以及
- 可以返回一个错误来拒绝转账。

## 状态

银行模块拥有:

- `BalancesPrefix` —— 每账户、每 denom 的余额
- `DenomMetadataKey` —— 每个 denom 的显示名称、小数位、描述
- `SupplyKey` —— 每个 denom 的总供应量
- 每 denom 的 send-enabled / receive-enabled 开关

ATOS 注册了如下 denom 元数据:

```yaml
base:     aatos          # atomic unit, on-chain wire format
display:  atos           # 18-decimal display unit
description: "Atoshi native token"
denom_units:
  - { denom: aatos, exponent: 0  }
  - { denom: atos,  exponent: 18 }
```

链的 `x/erc20` 模块还为 aatos 在确定性地址
`0xD4949664cD82660AaE99bEdc034a0deA8A0bd517` 上注册了一个 ERC20 对,
从而使 EVM 合约能够像操作任何其他 ERC20 那样与原生 ATOS 交互。

## 消息

标准的 SDK 银行消息——没有任何 Atoshi 特有的内容:

| Type URL | 用途 |
|---|---|
| `/cosmos.bank.v1beta1.MsgSend` | 两个账户间的转账 |
| `/cosmos.bank.v1beta1.MsgMultiSend` | 在单笔交易中进行一对多转账 |
| `/cosmos.bank.v1beta1.MsgUpdateParams` | 仅治理的参数更新 |
| `/cosmos.bank.v1beta1.MsgSetSendEnabled` | 仅治理的每 denom 发送开关 |

`MsgSend` 是钱包需要支持的、目前最常见的 Cosmos 消息。
它**不在**能量补贴白名单中——转账需消耗 gas,先用能量支付,
不足部分再用 ATOS 补齐。

## 查询

`/cosmos/bank/v1beta1/` 下的标准 REST 端点:

| 端点 | 返回 |
|---|---|
| `GET /balances/{addr}` | 一个账户持有的所有 denom |
| `GET /balances/{addr}/by_denom?denom=aatos` | 单个 denom 的余额 |
| `GET /supply` | 每个 denom 的总供应量(分页) |
| `GET /supply/by_denom?denom=aatos` | 特定 denom 的总量 |
| `GET /denoms_metadata` | 所有 denom 元数据记录 |
| `GET /params` | 模块参数(send-enabled 默认值) |

具体到 ATOS,`bank.balance` 的值不排除任何东西——锁定的
委托抵押品已经被转移**出去**到了能量锁仓池模块账户,
因此它不出现在用户的银行条目中。
钱包**不应**从 `bank.balance` 中减去 `EnergyAccount.LockedAtos`
来显示"可花费余额",因为那样会重复计算。

## 发送限制详解

`SendRestriction` 回调签名:

```go
func (k Keeper) SendRestriction(
    ctx context.Context,
    from, to sdk.AccAddress,
    amt sdk.Coins,
) (sdk.AccAddress, error)
```

对于每一笔 ATOS 转账:

```
1. Skip if amount doesn't include base denom (aatos).
2. Read both accounts' EnergyAccount records.
3. Compute the new eligible balance for `from`:
       new_from_eligible = bank.GetBalance(from, aatos) - amt - from.LockedAtos
   If new_from_eligible < 0:
       return error: insufficient unlocked balance
4. Run `energy.Settle(from)` against the OLD snapshot, then
   set from.LastBalanceSnapshot to the new eligible balance, then
   call `cap-down accrued to new capacity`.
5. Same for `to`: settle, set new snapshot, cap-down accrued.
6. Return (to, nil) — the transfer proceeds.
```

`from` 能量账户在流出时被触及,`to` 能量账户
在流入时被触及。两侧无需单独的钩子即可保持同步。

## 边界情形

### 多币种发送

`MsgSend` 可以携带多个币种 denom。该限制只检查
`aatos` 部分。非 ATOS 的币种(ibc 代币、ERC20 跨桥代币)
对能量状态没有影响。

### 自转账

从一个地址向其自身发送(`from == to`)仍会经过
该限制。Settle 会在同一账户上被调用两次;第二次调用
是无操作的,因为 `LastUpdatedTime` 已经与当前区块匹配。
快照写入同样是幂等的。无需特殊处理。

### 模块账户转账

内部转账(例如 tokenomics → fee_collector)也流经
同一个钩子。模块账户默认没有 `EnergyAccount` 记录,
因此钩子对它们是无操作的——但调用它仍然是安全的。

### 罚没(Slashing)

当一个验证人被罚没时,银行会将币从其绑定金额中转出
到社区池。这被当作任何其他转账一样处理,
且**会**触发 `SendRestriction`。由于验证人操作者账户通常
不会针对其自身的自绑定持有能量委托,因此
锁定 ATOS 检查会轻松通过。如果它们确实持有,链会拒绝
罚没超过其未锁定余额的部分。

### Gas 成本

`SendRestriction` 每笔转账向 KVStore 增加 2 次读 + 2 次写
(`from` 和 `to` 各一次)。按读 ~3 gas/字节、写 ~30 gas/字节计算,
每笔 `MsgSend` 大约增加 1,500–3,000 gas。相比该消息的
基础成本(~70k–100k gas)可以忽略不计。

## 参数

| 名称 | 默认值 | 说明 |
|---|---|---|
| `default_send_enabled` | true | 全局启用银行发送 |
| `send_enabled`(每 denom) | `[]` | 针对特定 denom 的覆盖;为空表示"使用默认值" |

链发布时所有 denom 均已启用。治理可以通过
`MsgSetSendEnabled` 临时禁用某个 denom(例如为紧急冻结一个被盗代币)。

## 事件

标准 SDK 事件:

| 事件 | 属性 |
|---|---|
| `transfer` | `recipient`、`sender`、`amount` |
| `coin_spent` | `spender`、`amount` |
| `coin_received` | `receiver`、`amount` |
| `message` | `action`、`sender`、`module` |

钱包的转账历史可以通过
`/cosmos/tx/v1beta1/txs?query=transfer.sender=...` 加上对
`transfer.recipient=...` 的第二次查询,再在客户端合并来重建
(Tendermint 的 tx-search 不支持 `OR`)。

## 相关模块

- [`x/energy`](./01-energy.md) —— 注册 `SendRestriction` 钩子;发送转账时
  优先消耗能量
- [`x/erc20`](#) —— 在固定地址将 aatos 以 ERC20 形式暴露给 EVM 合约
- [`x/evm`](./05-evm.md) —— 在处理 `eth_getBalance` 调用时
  读取银行余额;余额在两种接口间统一

---

*最后审阅:2026-06-10*
