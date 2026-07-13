# Gas 与手续费经济学

本章把三个原本各自独立的组件串联成
"一笔交易花费多少"的完整图景:

- [`x/energy`](../modules/01-energy.md) —— 以基于持仓的容量模型
  覆盖 Cosmos 交易
- [`x/feemarket`](../modules/05-evm.md) —— 以 EIP-1559 基础费(base fee)+
  最低 gas 价格下限覆盖 EVM 交易
- 手续费最终结算进入的共享 bank 池

如果你只读一章经济学,就读这一章。

## 两条并行的手续费路径

一笔 Cosmos `MsgSend` 和一笔以太坊 `eth_sendRawTransaction` 在
`x/bank` 中转移完全相同的 ATOS,但它们通过不同的
ante 流水线支付不同的手续费:

```
                  Cosmos tx                            Ethereum tx
              (signed protobuf)                  (signed RLP, type 0/2)
                     │                                    │
                     ▼                                    ▼
     ┌─────────────────────────────┐    ┌─────────────────────────────┐
     │  EnergyDeductDecorator      │    │  MonoDecorator              │
     │                             │    │                             │
     │  1. Compute gas required    │    │  1. Check tx fee ≥          │
     │  2. Drain TxEnergyAccrued   │    │     feemarket.MinGasPrice × │
     │     up to gas_limit          │    │     gas_limit                │
     │  3. If shortfall remains:    │    │  2. Deduct fee from sender's │
     │     drain DelegatedInUsable  │    │     ATOS balance             │
     │  4. If still short:          │    │  3. Refund unused on success │
     │     charge ATOS at min(      │    │                              │
     │       user_offered_gas_price,│    │  (energy is bypassed for EVM,│
     │       params.insufficient_   │    │   intentionally)             │
     │       gas_price)              │    │                              │
     └─────────────────────────────┘    └─────────────────────────────┘
                     │                                    │
                     └──────────┬─────────────────────────┘
                                ▼
                       fee_collector module account
                       (later distributed to validators
                        via x/distribution)
```

这两条流水线**不**共享 gas 池,**不**共享定价逻辑,
也**不**共享能量计。它们只共享目的地
(`fee_collector`)和底层 ATOS denom。

## Cosmos 交易:能量 + ATOS 不足补缴

详见 [`x/energy`](../modules/01-energy.md)。摘要:

拥有合格余额 B 的用户具有:
- `TxEnergyCapacity = floor(B / 30,000 ATOS) × 50,000` 单位
- 在 `tx_energy_max_accrue_window`(默认 24h)内线性回填

当一笔 `gas_limit = G` 的 Cosmos 交易到来时:

1. 从自身能量中扣除 `min(G, TxEnergyAccrued − DelegatedOut)`
2. 若仍不足,从 `DelegatedInUsable`(向委托人借来的)中扣除
3. 若两者用尽后仍不足,按 `shortfall_gas × params.insufficient_gas_price` 收取 ATOS

`insufficient_gas_price` 默认为 `0.0021 aatos/gas`。一笔 100,000-gas 的
零能量转账花费 `100,000 × 0.0021 = 210 aatos ≈
2.1 × 10⁻¹⁶ ATOS`。即便是最坏情况的 Cosmos 转账也
基本免费。

若交易中的每条消息都在补贴白名单上,则整条
流水线被绕过,无论能量状态如何,该交易都免费。见
[modules/01-energy.md §补贴白名单](../modules/01-energy.md)。

## EVM 交易:标准 EIP-1559

EVM 路径使用 ethermint 的 `MonoDecorator` 和 `x/feemarket`
模块。能量被有意绕过。一笔典型的 EVM 交易
支付:

```
total_fee = gas_used × effective_gas_price

where:
  effective_gas_price = min(max_fee_per_gas,
                            base_fee + max_priority_fee_per_gas)
  base_fee adjusts per block per EIP-1559:
     base_fee_n+1 = base_fee_n × (1 + (gas_used_n - target_gas) / (target_gas × denominator))
```

当前 `x/feemarket` 参数:

| 参数 | 值 |
|---|---|
| `min_gas_price` | 10⁹ aatos/gas(1 gwei)—— 下限 |
| `base_fee`(初始) | 1 gwei |
| `base_fee_change_denominator` | 8(标准以太坊) |
| `elasticity_multiplier` | 2(区块可达目标的 2×) |
| `min_gas_multiplier` | 0.5(至少按 gas_limit 的 50% 收取) |
| `no_base_fee` | false(启用 EIP-1559) |

通过 EVM 的原生 ATOS 转账:

| 成本项 | 值 |
|---|---|
| gas_limit | 21,000 |
| base_fee | 1 gwei(空闲时) |
| effective_fee | 21,000 × 1 gwei = 0.000021 ATOS |

一笔 ERC20 转账:

| 成本项 | 值 |
|---|---|
| gas_limit | ~65,000 |
| effective_fee | 65,000 × 1 gwei = 0.000065 ATOS |

这些价格比 Cosmos 路径高出几个数量级,但按绝对值
仍可忽略不计。该定价与钱包和 SDK 对任何 EVM 链的
预期一致。

## 选择正确的路径

| 情景 | 推荐路径 | 原因 |
|---|---|---|
| bech32 地址间的 ATOS 转账,持有者拥有 ≥ 30k ATOS | Cosmos `MsgSend` | ATOS 手续费为零(能量覆盖) |
| 向 0x... 地址转 ATOS;发送方用 MetaMask | EVM `eth_sendRawTransaction` | 保持 UX 连续性;成本可忽略 |
| ERC20 / 封装代币转账 | EVM(唯一选项) | Cosmos 看不到仅 ERC20 的资产 |
| 合约调用(DEX、借贷等) | EVM | Cosmos 无法直接调度 EVM 合约 |
| 验证人委托、治理投票 | Cosmos | 部分 Cosmos 消息没有 EVM 对应 |
| L1 → L2 跨链桥存款 | EVM(桥合约是 EVM) | 唯一路径 |

钱包通常通过同一套 UX 暴露两条路径,根据
收款地址格式自动选择(bech32 → Cosmos,
0x... → EVM)。

## 手续费流向何处:验证人分配

两条路径都把收取的手续费路由进 `fee_collector` 模块账户。
在每个区块结束时,SDK 的 `x/distribution` 模块清扫该
余额,并按投票权比例分配给活跃验证人,
扣除社区池税(当前 2%)。

验证人再按各自公示的佣金率与其委托人分享。
委托人通过 `MsgWithdrawDelegatorReward` 领取。

区块奖励发行(见
[economics/04-block-rewards.md](./04-block-rewards.md))也支付进
同一 fee_collector 流程,因此验证人无需区分"手续费
收入"和"发行收入" —— 两者都流经同一机制。

## 销毁 vs 累积

Atoshi 当前对交易手续费**零销毁**。所有收取的
ATOS 都归验证人和社区池。不存在
以太坊 EIP-1559 基础费(base fee)销毁的对应机制。

治理未来可通过对 `x/distribution` 执行 `MsgUpdateParams`
引入销毁比例(具体做法是将一部分路由到 burner 模块
账户)。这将是一项审慎的货币政策选择,伴随若干
影响:

- **利**:对供应形成通缩压力,尤其在高交易
  量期间
- **弊**:同时降低验证人收入,而这恰恰与高流量
  期间为保障安全所需的方向相反

当前"无销毁"设计假设:
- 区块奖励发行是主导性的供应效应;交易手续费
  微不足道
- 验证人收入的稳定性比供应通缩更重要

## 实例

### 示例 A:持有 60,000 ATOS 的持有者通过 Cosmos 向朋友转 100 ATOS

```
balance:   60,000 ATOS
capacity:  100,000 energy
accrued:   ~50,000 energy (assuming partially refilled)
delegated_out / in_usable: 0

Tx: MsgSend, ~100,000 gas

EnergyDeductDecorator:
  available_own = 50,000 − 0 = 50,000
  shortfall = 100,000 − 50,000 = 50,000 gas
  charge_atos = 50,000 × 0.0021 = 105 aatos
       ≈ 1.05 × 10⁻¹⁶ ATOS

Total user cost: 100 ATOS transferred + 105 aatos fee
```

### 示例 B:同一持有者、同一转账,但走 EVM

```
Tx: eth_sendRawTransaction, native transfer, gas_limit=21,000

MonoDecorator:
  fee = 21,000 × 1 gwei = 2.1 × 10⁻⁵ ATOS

Total user cost: 100 ATOS transferred + 2.1 × 10⁻⁵ ATOS fee
```

本例中 EVM 路径贵约 200,000×。但按绝对值仍
可忽略。原则是:若收款方是 0x 地址且钱包默认
使用 MetaMask,直接用 EVM。

### 示例 C:新用户持有 0 ATOS,但有 1,000 ATOS 的借入能量

```
balance:           1,000 ATOS  (below 30,000 threshold → capacity 0)
accrued:           0
delegated_in_usable: 50,000  (delegated to user by a relayer)

Tx: MsgSend, 100,000 gas

EnergyDeductDecorator:
  available_own = 0 − 0 = 0
  drain delegated_in_usable: takes 50,000
  remaining shortfall = 100,000 − 50,000 = 50,000 gas
  user offered fee = 0
  → fall back to params.insufficient_gas_price
  charge_atos = 50,000 × 0.0021 = 105 aatos
```

该用户仍能发送交易,因为他们借到了能量以覆盖
一半 gas,链会自动收取剩余部分。这
就是"relayer"模式:一个赞助者向众多用户委托能量,
使他们能以低于阈值的余额进行交易。

## 规模化的手续费经济学

假设:
- 平均每笔交易在 1 gwei 下支付 100,000 gas → 0.0001 ATOS / tx
- 链持续处理 10 tx/s → 864,000 tx/day
- 每日手续费收入:86.4 ATOS

按假设的 $0.15 ATOS 价格,那是全体验证人合计每天
约 $13 的手续费收入。以以太坊标准看微乎其微 —— 在此
交易量下仅靠手续费无法维持验证人经济。

这正是为何区块奖励(见
[economics/04-block-rewards.md](./04-block-rewards.md))是
主导性的验证人收入来源。只有当交易量达到
以太坊主网级别(200+ tx/s)时,手续费才变得可观。

## 与同类链的对比

| 链 | Gas 定价 | 手续费销毁 | 亚美分转账? |
|---|---|---|---|
| Ethereum mainnet | EIP-1559 基础费 + 小费 | 基础费销毁 | 否(~$0.50–$10) |
| BNB Smart Chain | 固定 gas 价格 | 无 | 勉强(~$0.05) |
| Polygon PoS | EIP-1559 + 小费 | 基础费销毁 | 是(~$0.001) |
| Polygon zkEVM | EIP-1559 + 小费 | 基础费销毁 | 是(~$0.001) |
| **Atoshi L1(Cosmos 路径)** | **能量 + 不足下限** | **无** | **是**(通常为 0) |
| **Atoshi L1(EVM 路径)** | **EIP-1559 + 1 gwei 下限** | **无** | **是**(~$0.000001) |

Atoshi 的独特设计在于能量路径,而非 EVM 路径。EVM
出于兼容性有意采用"标准以太坊 gas 经济学"。
有趣的问题是:有多少用户会发现 Cosmos 路径?

## 治理杠杆

三个参数占主导。全部可通过 `MsgUpdateParams` 提案调整。

| 杠杆 | 模块 | 调高的影响 |
|---|---|---|
| `energy.tx_energy_holding_threshold` | x/energy | 免费交易的门槛更高(当前 30,000 ATOS) |
| `energy.insufficient_gas_price` | x/energy | ATOS 不足补缴费更贵(当前 0.0021 aatos/gas) |
| `feemarket.min_gas_price` | x/feemarket | EVM 交易下限更高(当前 1 gwei) |

近期将 `feemarket.min_gas_price` 从 0 提升到 1 gwei 的治理
提案是典型案例。变更前的行为
(`min_gas_price = 0`)使钱包显示"gas price = 0",因为
`eth_gasPrice` 字面返回 `0x0`,打破了用户预期。

## 相关

- [Energy 模块](../modules/01-energy.md) —— 能量系统的完整规范
- [EVM 模块](../modules/05-evm.md) —— EVM gas 流水线详解
- [区块奖励](./04-block-rewards.md) —— 手续费收入与发行的交汇点
- [治理案例研究](../modules/09-governance.md) —— 包含 feemarket
  手续费下限提案

---

*最后审阅:2026-06-10*
