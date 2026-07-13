# `x/energy` —— 通过持有获得 gas,而非付费

energy 模块是 Atoshi 用户体验设计的核心。它不再让每笔交易都用稀缺的原生代币来支付,而是为每个账户提供一份*容量*(capacity):只要持有者维持最低的 ATOS 余额,该容量便会随时间不断回填。交易优先消耗这份容量,只有不足部分才用 ATOS 支付。

本章是该模块的权威规范:数据模型、累积数学、委托、消耗顺序、ante 处理器集成、参数、边界情况。如果你只读一章模块文档,那就读这一章。

本章将完整回答以下三个问题——每个问题下方都有专门的实例演算章节:

- **能量是如何累积的?** → [累积](#accrual-how-energy-accumulates)
- **我如何把能量转移(委托)给另一个账户?** → [委托](#delegation-lending-energy-to-another-account)
- **一旦转移出去,别人还能动用那份能量吗?** → [为什么委托出去的能量具有排他性](#why-delegated-energy-cannot-be-used-by-anyone-else)

## 思维模型

持有 ATOS → 获得免费交易的权利,速率与持有量成正比。

具体来说:

- 持有 **30,000 ATOS** 可解锁 **220,000 TxEnergy 容量**,在 **24 小时**内线性回填。
- 持有 **1,000,000 ATOS** 还额外解锁 **800,000 DeployEnergy 容量**,在 **10 天**内线性回填。
- 二者都是按账户跟踪的 `uint64` 计数器。
- TxEnergy 可以在固定期限内**委托**(出借)给另一个账户;对应的 ATOS 会被锁定为抵押。DeployEnergy 不可委托。

`TxEnergy` 覆盖普通交易(转账、合约调用、委托)。`DeployEnergy` 覆盖合约部署类消息(Wasm instantiate / store-code)。EVM `MsgEthereumTx`(包括创建合约的 EVM 交易)**不**由本模块处理;参见 [EVM 边界](#the-evm-boundary)。

## 状态

### `EnergyAccount`

每个账户一条记录。定义与 `proto/atoshi/energy/v1/energy.proto` 一致:

```protobuf
message EnergyAccount {
  string address               = 1;   // bech32, the account this row describes
  uint64 tx_energy_accrued     = 2;   // current spendable TxEnergy (before adding delegated_in_usable)
  uint64 deploy_energy_accrued = 3;   // current spendable DeployEnergy
  string last_balance_snapshot = 4;   // eligible_balance (bank − locked) at last settle
  int64  last_updated_time     = 5;   // unix second of last settle
  uint64 delegated_out         = 6;   // TxEnergy currently lent to others
  uint64 delegated_in_usable   = 7;   // TxEnergy received from others, still usable
  string locked_atos           = 8;   // total ATOS currently in collateral lock
}
```

四个可花费 / 锁定总额(`tx_energy_accrued`、`delegated_out`、`delegated_in_usable`、`locked_atos`)做了反规范化处理,以便快速读取。委托余额的权威账本存放在单独的 `EnergyDelegation` 表中。

一个账户的可用 TxEnergy 始终为:

```
usable = (tx_energy_accrued − delegated_out) + delegated_in_usable
```

### `EnergyDelegation`

每个活跃委托一条记录:

```protobuf
message EnergyDelegation {
  uint64 id          = 1;   // monotonic, unique across all delegations
  string delegator   = 2;
  string delegatee   = 3;
  uint64 amount      = 4;   // TxEnergy units lent (gas units)
  string locked_atos = 5;   // collateral frozen in the locked module account
  int64  start_time  = 6;
  int64  expires_at  = 7;   // auto-released by EndBlocker at this time
  uint64 used        = 8;   // accumulated consumption attributed to this record
}
```

两个二级索引用于加速常见查询:

- 按 `delegatee` —— 消耗路径用它把消耗归属到具体的入向委托
- 按 `expires_at`(8 字节大端序,再接 8 字节 id)—— EndBlocker 到期清理,复杂度为 O(expired) 而非 O(total)

### 模块账户

- `energy_locked_pool` —— 持有所有作为委托抵押提交的 ATOS。委托方的银行余额会被物理扣减掉锁定的数额(ATOS 是被*转移*了,而非仅仅打上标记),这正是为什么委托存续期间任何人都无法转移这些被锁定的 ATOS。

## 容量公式

容量是账户**合格余额**(eligible balance,即银行余额减去作为委托抵押锁定出去的 ATOS)的阶梯函数:

```
eligible_balance = bank.GetBalance(address) − locked_atos
threshold_blocks = floor(eligible_balance / TxEnergyHoldingThreshold)
tx_energy_capacity = threshold_blocks × TxEnergyPerThreshold      (saturating on overflow)
```

采用默认参数(`TxEnergyHoldingThreshold = 30,000 ATOS`、`TxEnergyPerThreshold = 220,000`)时:

| 合格余额 | 容量 |
|---|---|
| 0 – 29,999 ATOS | 0 |
| 30,000 – 59,999 ATOS | 220,000 |
| 60,000 – 89,999 ATOS | 440,000 |
| 1,000,000 ATOS | 7,260,000 |

采用阶梯函数是有意为之:它让持有要求清晰无歧义、易于传达("持有 30,000 ATOS 即可免费交易")。若采用线性函数,攻击者只需持有微量代币就能获得近乎为零的容量。

DeployEnergy 则不同:`DeployEnergyCapacity`(默认 800,000)是一个**恒定的按账户上限**,而非按块的乘数。任何持有 ≥ `DeployHoldingThreshold`(1,000,000 ATOS)的账户都朝着同一个 800,000 上限回填;持有更多只会*更快*回填,不会抬高上限。回填速率见累积一节。

## 累积:能量如何积累 {#accrual-how-energy-accumulates}

这是三个头号问题中的第一个。能量就像一个水桶,以固定速率朝其容量回填;你从桶里花费,而持有 ATOS 会持续为它回填。

### 惰性结算 —— 没有逐块扫描

能量以**惰性**方式回填。没有任何定时任务会每块触碰每个账户。链上按账户存储 `last_updated_time` 与 `last_balance_snapshot`,仅在账户下次被触碰时才重新计算累积能量。这套例程称为 `Settle`(`keeper/settle.go`),它在以下时机运行:

- 在 ante 处理器中,扣费之前(会改写状态)
- 在每个 keeper 消息处理器的开头(会改写状态)
- 在查询中以 `SimulateSettle` 形式运行(不改写状态——因此外部调用方看到的是实时数值,绝不会是陈旧的存储值)

由于结算在同一块内是幂等的,你从链外永远观察不到陈旧的能量值:`Account` 查询会在返回前先结算。

### 回填方程

对于 TxEnergy,每次结算时:

```
elapsed = now − last_updated_time
cap     = TxEnergyCapacity(last_balance_snapshot)
add     = (cap × elapsed) / TxEnergyMaxAccrueWindow          // multiply-before-divide
new_accrued = min(tx_energy_accrued + add, cap)
```

`TxEnergyMaxAccrueWindow` 默认为 86,400(24 小时)。先乘后除的运算顺序很关键:朴素的 `(cap / window) × elapsed` 会损失精度——`220000 / 86400 = 2`(每秒),丢掉了 0.546 的小数部分;而对更小的容量则会直接被整数运算截断为零。所有中间乘积都使用饱和运算,因此治理层的重新调参永远不会让 `uint64` 回绕溢出。

**实例演算(TxEnergy)。** Alice 持有 30,000 ATOS,故 `cap = 220,000`。速率 = 220,000 / 86,400 ≈ **2.546 能量/秒**。

- 从空开始。1 小时(3,600 秒)后:`add = 220,000 × 3,600 / 86,400 = 9,166`。累积 ≈ 9,166。
- 12 小时后:`add = 220,000 × 43,200 / 86,400 = 110,000`。累积 = 110,000(容量的一半)。
- 24 小时后:累积触及 220,000 上限并停止。持有更久也不会溢满。

**实例演算(DeployEnergy)。** DeployEnergy 速率为 `(units × DeployEnergyCapacity × elapsed) / (DeployRecoverDays × 86,400)`,其中 `units = floor(eligible_balance / DeployHoldingThreshold)`。

- Bob 恰好持有 1,000,000 ATOS → `units = 1`。从 0 回填到 800,000 需要整整 10 天。
- Carol 持有 3,000,000 ATOS → `units = 3`。她朝着*同一个* 800,000 上限以 3 倍速率回填,即从空开始约 3.3 天。上限不会升高,只是速度更快。

### 为什么用 `last_balance_snapshot` 而不是实时余额

快照记录的是*正在被结算的那段时间内*的余额。如果用实时余额去计算一段余额已经变化过的时期的累积,就会错误归属能量。

银行发送钩子(`SendRestriction` → `ApplyBalanceChange`)在每次基础 denom 转账时保持快照的准确性:

1. 先 `Settle` —— 针对旧快照结清累积。
2. 写入 `last_balance_snapshot` = 预计的新合格余额。
3. 将 `tx_energy_accrued` **向下**封顶到新的(可能更低的)容量。

第三步(向下封顶)至关重要:一个卖掉大部分 ATOS 的账户,绝不能保留一个巨大的预填充缓冲区,再转移到全新钱包去做免费交易。若无此步,容量便只会单向棘轮上升。

**向下封顶以 `delegated_out` 为下限**(审计 Issue 6)。如果持有者已经把能量出借出去,随后卖出质押导致新容量低于 `delegated_out`,那么向下封顶会停在 `delegated_out` 处,而不会将其压垮——出借出去的能量是一份已经铸入受托方 `delegated_in_usable` 的承诺,若压到其下方将破坏取消委托所依赖的对称记账。只有持有者*自己*未使用的那部分会被削减。

### Evmos 发送顺序陷阱(已于 2026-06-30 修复)

`ApplyBalanceChange` 在银行 store 写入*之前*运行,因此必须传入**预计的**发送后余额,而不能回读。Evmos 的 cosmos-sdk 分叉在 `subUnlockedCoins(from)` **之后**、`addCoins(to)` **之前**调用 `SendRestriction`——与上游相反。所以在钩子内部:

- `bank.GetBalance(from)` 已经反映了扣减 → **不要**再次减去 `moved`。
- `bank.GetBalance(to)` 仍是加数之前的状态 → **加上** `moved` 以推算收款后余额。

修复前的代码在发送方一侧重复扣减,在*每一笔*转账中悄悄地从快照里削掉一份 `moved` 的合格余额——对于一笔 30,000 ATOS 的划转,那恰好是一个阈值区块,即每次发送容量下降 50,000 能量(即当时生效的每档容量)。这就是测试网账户 `0x30F288…` 上生产环境 "5万 ATOS 凭空消失" 的特征表现。修复方案在 `keeper/send_restriction.go` 中有内联文档说明。

## 消耗:ante 处理器流水线 {#consumption-the-ante-handler-pipeline}

`x/energy/ante.EnergyDeductDecorator` 取代了 SDK 原生的 `DeductFeeDecorator`。对于每一笔 Cosmos 交易:

1. 解码交易,解析手续费支付方(尊重 `--fee-granter` / `x/feegrant`),收集消息类型 URL 和 gas 上限。拒绝为零的 gas 上限。
2. 如果交易中的**每一条**消息都在补贴白名单上 → `{Free: true}`,完全跳过扣费。
3. 否则调用 `Consume(payer, gasLimit, isDeploy, msgTypeUrls)`(`keeper/consume.go`),它按下述顺序耗用能量并返回 `ConsumeResult{EnergyDeducted, OwnDeducted, DelegatedDeducted, DeployEnergyUsed, ShortfallGas, Free, DelegationConsumptions}`。
4. 若 `ShortfallGas == 0` → 校验优先级,跳过手续费转账。
5. 否则 → 将 ATOS 不足部分收取到 `fee_collector` 模块账户。

**PostHandler**(`ante/post.go`)在消息执行完后退还预留但未使用的能量(等同于 gas 退款的能量版本)。ante 状态中写入的**待处理预留标记**(pending-reservation marker)防范了 `runMsgs` 失败的情形:此时 BaseApp 会丢弃消息上下文状态并跳过 PostHandler,退款本会丢失——EndBlocker 会清扫遗留的标记并退还失败交易的预留(审计 Issue-1 round2)。

### 消耗顺序

对于**普通**交易:

1. 自有能量:`tx_energy_accrued − delegated_out`。
2. 然后是入向委托池:`delegated_in_usable`。
3. 仍未覆盖的部分 → `ShortfallGas`。

对于**部署**交易:先 `deploy_energy_accrued`,再是上面两个 TxEnergy 桶,然后是 `ShortfallGas`。

第 1 步中减去 `delegated_out`,是因为那部分能量已经铸入了受托方的 `delegated_in_usable`;若不减去,委托方就能重复花费它已经出借的能量。

### 退款顺序按来源 LIFO

PostHandler 退还 `reserved − actually_used`。它按**来源**拆分退款(审计 Q1 / Issue 11):

- **委托**部分优先退还,逆序遍历 `DelegationConsumptions`(最后消耗的最先退)并同步回滚每条委托记录的 `used`。把未使用的能量退回到同一份有时限的授予上,能保持其完整,而不是悄悄把它转成受托方永久的自有能量。
- 如果某份委托在 `Consume` 和退款之间被取消或清扫掉了,那一份将**丢失,而不会被重定向**到签名方自己的桶里——委托方在 `releaseDelegation` 内部已经为其扣了账,因此不欠任何对手方(审计 Issue 11)。
- 剩余的退款计入签名方自己的 `tx_energy_accrued`,以当前容量为上限。

### 补贴白名单(始终免费的消息)

定义于 `types.DefaultParams().SubsidizedMsgTypeUrls`:

| Type URL | 用途 |
|---|---|
| `/atoshi.tokenomics.v1.MsgClaimMigrationTokens` | 旧链迁移领取 |
| `/atoshi.tokenomics.v1.MsgClaimMinerLockedReward` | 验证人解锁领取 |
| `/atoshi.tokenomics.v1.MsgClaimProjectTreasuryReward` | 项目金库领取 |
| `/atoshi.oracle.v1.MsgReportPrice` | 白名单喂价方价格上报 |
| `/atoshi.energy.v1.MsgDelegateEnergy` | 能量委托 |
| `/atoshi.energy.v1.MsgUndelegateEnergy` | 能量撤销 |

补贴是全局性的——即便一个余额为零、能量为零的账户也能免费调用这些消息。对于领取类消息,反滥用控制在应用层授权;对于委托 / 取消委托,则由 ATOS 抵押锁定来把关。

委托 / 取消委托**必须**被补贴:ante 处理器会贪婪地预留至多 `gas_limit` 的签名方能量作为手续费覆盖,而这会在委托处理器运行并试图锁定其中一部分*之前*就掏空委托方已累积的能量。一笔完全被补贴的交易不预留任何东西,因此处理器能看到全部已累积余额。

### 不足部分定价

当能量不足时:

```
charge = min(user_offered_gas_price, InsufficientGasPrice) × shortfall_gas
```

`InsufficientGasPrice` 默认为 `0.0021 aatos/gas`。为用户报价设上限可防止手续费敲诈——用户无法通过支付一笔巨额的非能量手续费把别人挤出内存池;链最多按政策价收取。若用户报价为零手续费,链仍按政策价收取,否则就能用 `tx.fee = 0` 来逃避不足部分。

**优先级**由实际支付的 ATOS(`chargeAtos`)推导,而非声明的手续费(审计 Q1 round2)。否则一个能量巨鲸可以声明一笔巨额手续费,却因为能量覆盖了 gas 而几乎不付钱,还能在对该区块槽位没有真实经济利益的情况下插队。

## 委托:把能量借给另一个账户 {#delegation-lending-energy-to-another-account}

这是第二个头号问题。Atoshi 上的"能量转移"是一种**委托**(出借),而非永久移交:你保留底层 ATOS 的所有权,把它锁定为抵押,借出的能量在固定时间窗内绑定给单一接收方,期满后自动归还给你。

使用场景:一个中继(relayer)服务通过接受持有者的委托并代其支付 gas,来提供"免费发送你的 ATOS"服务;或者一个父账户为子钱包的 gas 出资,而无需向其发送任何代币。

### 如何操作 —— CLI

```bash
# Lend 200,000 TxEnergy to <delegatee> for 7 days.
# 200,000 gas ≈ 4 ERC20 transfers.
atoshid tx energy delegate <delegatee> 200000 168h --from mywallet

# Cancel one of your outbound delegations early and reclaim the frozen ATOS.
atoshid tx energy undelegate <delegation_id> --from mywallet
```

- `AMOUNT` 是以 gas 单位表示的 TxEnergy。
- `DURATION` 是 Go 的时长字符串(`24h`、`168h`、`720h`)。如果在消息中省略 / 设为 `0`,则应用协议默认值 **7 天**(`604800 s`)。

底层消息为 `MsgDelegateEnergy{delegator, delegatee, amount, duration_seconds}` 和 `MsgUndelegateEnergy{delegator, delegation_id}`。`DelegateEnergy` 会返回新的 `delegation_id` 和确切的 `locked_atos`。

### 抵押:会冻结多少 ATOS

```
threshold_units = ceil(amount / TxEnergyPerThreshold)
locked_atos     = threshold_units × TxEnergyHoldingThreshold
```

你必须锁定*本应支撑*所借容量的那笔银行余额——TRON 风格的冻结模型。这防止出借你实际上没有质押支撑的能量。

**向上取整,以整数阈值区块计。** 采用默认值时,委托从 1 到 220,000 之间的任意数量能量都会锁定整整 **30,000 ATOS**。委托 220,001 则锁定 **60,000 ATOS**。钱包 UI 必须把这一点呈现出来——否则用户会惊讶于出借"一点点"能量竟冻结了整整一个区块的 ATOS。

### 委托流程(keeper `Delegate`)

1. 拒绝 `amount == 0`、`duration ≤ 0` 以及 `delegator == delegatee`(`ErrSelfDelegation`)。
2. 计算 `locked_atos`(如上,向上取整)。
3. `Settle` 委托方。要求空闲能量 `(tx_energy_accrued − delegated_out) ≥ amount`(`ErrInsufficientEnergy`)。
4. 要求空闲银行余额 `≥ locked_atos`(`ErrInsufficientBalance`)。
5. 在银行转移**之前**写入委托方计数器:`delegated_out += amount`、`locked_atos += locked`。(预先写入使发送钩子推算的合格余额保持稳定,因此一次委托对容量是中性的——参见内联审计注释。)
6. `bank.SendCoinsFromAccountToModule(delegator → energy_locked_pool, locked)`。
7. `Settle` 受托方,`delegated_in_usable += amount`。
8. 插入 `EnergyDelegation{id, …, used: 0}`。

非显而易见的特性:

- 委托方的 `tx_energy_accrued` **不**减少——而是 `delegated_out` 上升,可用公式 `accrued − delegated_out + delegated_in_usable` 处理其余部分。
- 单个受托方可以接收来自多个委托方的委托;入向授予只是简单地汇总进一个 `delegated_in_usable` 计数器。

### 消耗归属 —— 最先到期者优先

当受托方从 `delegated_in_usable` 花费时,keeper 会把消耗归属到该账户的入向委托上,**按 `expires_at` 升序**排列(最近的截止期优先;为确定性以 id 破平)。这源自审计 Issue 7:按 id 顺序归属可能在一份 30 分钟的授予之前先烧掉一份 30 天的授予,浪费了短期授予。最早截止优先能在每份授予到期前从中榨取最大价值。

### 取消委托(提前撤销)与自动到期

`MsgUndelegateEnergy` **仅可由原委托方调用**。它与 EndBlocker 的到期清理共享一条代码路径 `releaseDelegation`:

1. `Settle` 委托方。**从委托方的 `tx_energy_accrued` 中扣除已消耗的 `used`**(审计 Issue-2 round2——见下文安全一节),然后 `delegated_out −= amount`、`locked_atos −= 本次委托的 locked`。
2. 把锁定的 ATOS 从 `energy_locked_pool` 退还给委托方。
3. `Settle` 受托方,`delegated_in_usable −= remaining`(未使用部分)。
4. 删除记录并发出事件。

已消耗的能量仍旧消耗掉——受托方拿到了那份价值。

`SweepExpiredDelegations` 在 **EndBlocker** 中运行,遍历 `by_expires_at` 索引(已排序,故每块 O(expired))。到期释放产生的是**区块事件,而非交易**——`energy_delegation_expired` **无法**通过 `cosmos/tx/v1beta1/txs?query=…` 检索到。需要检测"我的委托刚刚到期"的钱包必须通过 WebSocket 订阅区块事件,或轮询委托查询。

## 为什么委托出去的能量别人无法使用 {#why-delegated-energy-cannot-be-used-by-anyone-else}

这是第三个头号问题,也是本设计的安全核心:**一旦你委托出能量,在它到期之前就专属于受托方——不再是你的、不可再出借、不可被任何第三方触及——而支撑它的 ATOS 处于冻结状态。** 有四套独立机制来强制执行这一点。

### 1. 委托方立即失去对它的使用权

委托会抬高 `delegated_out`,而每条消耗路径都把委托方自己的可用能量算作 `tx_energy_accrued − delegated_out`。委托一落地,那一份就从委托方能花费的额度中被减去。委托方无法既出借能量*又*继续花它。

### 2. 只有被指名的受托方能花费它

借出的 `amount` 仅被加到*受托方的* `delegated_in_usable`。不触碰任何其他账户的状态。能量只从签名方自己的账户行花费,因此第三方没有可从中支取委托的计数器——不存在无记名工具、不存在可转让的债权、也没有任何别的地址能出示来花费它。

### 3. 委托能量不可传递地再委托

委托路径要求空闲能量 `tx_energy_accrued − delegated_out ≥ amount`——它**只**看受托方*自己*已累积的能量,并刻意**排除** `delegated_in_usable`。因此受托方可以把借来的能量*花*在自己的交易上,却无法把它继续转借出去。委托恰好只有一跳深。这阻止攻击者把单一的锁定质押通过一串账户"洗"出来以倍增可用能量。

### 4. 支撑的 ATOS 被冻结,已消耗的能量在释放时被扣账

抵押被**物理转移**到 `energy_locked_pool`,因此它完全离开了委托方可花费的银行余额——委托存续期间任何人(包括委托方在内)都无法转移或再利用它。它同时退出委托方的合格余额,故在此期间停止产生新的容量。ATOS 仅在取消委托或到期时归还。

它所堵住的微妙攻击(审计 Issue-2 round2):假设释放时只缩减 `delegated_out` 而不动 `tx_energy_accrued`。委托方就能 `Delegate → 让受托方烧掉它 → Undelegate`,由于 `delegated_out` 归零而 `tx_energy_accrued` 仍把已烧掉的单位计为自有可用,委托方就能收回受托方*已经花掉*的能量——用一份累积预算掏出两笔交易的 gas,且可无限重复。修复方案在释放时从委托方的 `tx_energy_accrued` 中扣除 `used`,因此受托方消耗掉的能量对委托方来说也确实没了。能量端到端守恒:它恰好被花费一次,由恰好一个账户花费。

## 估算:预览一笔交易的成本

发送之前,钱包可以在不广播的情况下预览拆分:

```bash
atoshid query energy estimate-fee <signer> <gas_limit> [--deploy]
# REST: GET /atoshi/energy/v1/estimate_fee?signer=&gas_limit=&is_deploy=
```

响应:

```
energy_used   uint64   // gas covered by accrued + delegated energy
shortfall_gas uint64   // gas requiring ATOS
atos_fee      string   // shortfall_gas × insufficient_gas_price
free          bool     // true if every msg is subsidized
```

它运行与 ante 处理器相同的代码路径(`EstimateConsume`),但不提交状态。

## 查询能量状态

```bash
# Settled account state + current capacity ceilings.
atoshid query energy account <address>
#   settled.tx_energy_accrued      spendable TxEnergy
#   settled.deploy_energy_accrued  spendable DeployEnergy
#   settled.delegated_in_usable    lent to me by others
#   settled.delegated_out          I lent to others
#   settled.locked_atos            ATOS frozen by my outbound delegations
#   tx_energy_capacity             upper bound at my current balance
#   deploy_energy_capacity         constant DeployEnergy ceiling

# Active delegations. direction ∈ {out, in, all} (default all).
atoshid query energy delegations <address> [direction]

# Module parameters.
atoshid query energy params
```

对应的 REST 端点位于 `/atoshi/energy/v1/` 下(`account/{address}`、`delegations/{address}`、`params`、`estimate_fee`)。

## 参数

全部可通过 `MsgUpdateParams` 由治理调整(authority = gov 模块)。默认值见 `types/helpers.go`:

| 参数 | 默认值 | 说明 |
|---|---|---|
| `energy_enabled` | `true` | 总开关;若为 false,ante 处理器回退到标准 SDK 扣费。已累积的能量和委托被保留。 |
| `tx_energy_holding_threshold` | 30,000 × 10^18 aatos | 每个容量区块所需的 ATOS |
| `tx_energy_per_threshold` | 220,000 | 每个区块的 TxEnergy 单位数 |
| `tx_energy_max_accrue_window` | 86,400 (s) | 从 0 回填到上限的时间 |
| `deploy_holding_threshold` | 1,000,000 × 10^18 aatos | 每个 deploy 单位所需的 ATOS |
| `deploy_energy_capacity` | 800,000 | **恒定**的 DeployEnergy 上限(非按块乘数) |
| `deploy_recover_days` | 10 | 单个单位下的回填窗口;持有更多按比例更快回填 |
| `insufficient_gas_price` | 0.0021 aatos/gas | 不足部分手续费率,也是用户报价的上限 |
| `subsidized_msg_type_urls` | (见上方列表) | 始终免费的消息 |
| `privacy_relayer_whitelist` | empty | 预留给隐私模块 L2 配额映射 |

协议默认委托时长(`604800 s` = 7 天)是一个代码常量(`types.DefaultDelegationDurationSeconds`),当提交的委托 `duration_seconds == 0` 时由消息服务器应用;它不是治理参数。

## 事件

| 事件 | 属性 | 触发时机 |
|---|---|---|
| `energy_consumed` | `address`, `energy_used`, `shortfall_gas` | 每笔交易一次,由 ante 处理器发出 |
| `energy_delegated` | `delegation_id`, `delegator`, `delegatee`, `amount`, `locked_atos` | `MsgDelegateEnergy` 成功时 |
| `energy_undelegated` | `delegation_id`, `delegator`, `delegatee`, `amount` | `MsgUndelegateEnergy` 成功时 |
| `energy_delegation_expired` | `delegation_id`, `delegator`, `delegatee`, `amount` | EndBlocker,当 `expires_at < now` 时 |

`energy_undelegated` 和 `energy_delegation_expired` 共享 `releaseDelegation` 代码路径,故它们的属性集完全相同——消费方通过事件 `type` 加以区分。`energy_delegation_expired` 在**区块层级**发出,而非在交易中(见上文到期注释)。

## EVM 边界 {#the-evm-boundary}

EVM 交易(`/ethermint.evm.v1.MsgEthereumTx`),包括创建合约的交易,**不**由 energy 模块处理。EVM ante 链(`MonoDecorator`)以标准 ATOS 处理它们的 gas。因此 `isContractDeployMsg` 只识别原生 Cosmos 部署消息;此前存在的 `MsgEthereumTx` 分支不可达,已被移除。如果将来把 EVM gas 桥接到能量,这里就是需要改动的接缝。参见 [Cosmos L1](../architecture/02-cosmos-l1.md)。

## 边界情况与运维注意事项

**累积后余额跌破阈值。** `ApplyBalanceChange` 中的向下封顶会在余额变化的那一刻把 `tx_energy_accrued` 裁剪到新的更低容量(以 `delegated_out` 为下限)。超出新上限的累积能量被没收。

**委托方的空闲余额 vs. 锁定抵押。** 二者永远不会倒挂——抵押被*转移*到模块账户,而非在用户余额中打标记,因此它在银行层面被隔离。委托期间的质押罚没(slashing)仍可能命中未锁定的质押,但委托抵押是安全的。

**对同一受托方的两份委托。** 会叠加——`delegated_in_usable` 汇总所有入向授予。每条记录相互独立;当一份到期时只有它那一份归还,其余仍活跃。消耗按最先到期者优先归属。

**自委托。** 以 `ErrSelfDelegation` 拒绝。它会是一个只会搅乱记账的空操作。

**竞态条件。** 所有 keeper 操作都在 SDK 的串行交易模型下运行——没有块内并发。`Settle` 在同一块内是幂等的。

## 相关模块

- **`x/bank`** —— 容量通过 `EligibleBalance(addr) = bank − locked_atos` 依赖于银行余额。银行发送钩子(`SendRestriction`)调用 `ApplyBalanceChange`,使容量立即反映转账。
- **`x/tokenomics`** —— 领取类消息被补贴,以便没有能量的用户仍可领取解锁。
- **`x/oracle`** —— `MsgReportPrice` 被补贴,使喂价方无需持有 ATOS 即可上报价格。
- **`x/evm`** —— 被能量模块绕过;EVM 交易支付标准 ATOS gas。参见 [EVM 边界](#the-evm-boundary)。

## 延伸阅读

- [Gas 与能量经济学](../economics/03-gas-fee.md) —— 手续费曲线、参数敏感性、货币分析
- [钱包集成手册](../reference/wallet-integration.md) —— 在钱包 UI 中展示能量
- [API 参考](../reference/api-guide.md#energy) —— `/atoshi/energy/v1/` 下的所有 REST 端点

---

*最后审阅:2026-07-10*
