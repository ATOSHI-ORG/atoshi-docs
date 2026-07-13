# L2 上的能量结算 —— 无 gas 隐私交易

L1 的 [`x/energy`](../modules/01-energy.md) 模块为持有者补贴 Cosmos
交易费用。自然的扩展方向就是把同样的福利带到 L2 隐私操作上 ——
这类操作若不加处理,大多会因为 Groth16 证明验证而消耗约 280k EVM gas。

本章说明 L2 上的 **EnergySettlement** 合约如何实现这座桥梁:
持有者的 L1 能量额度可以为其 L2 隐私交易付费,而持有者本人
无需签署任何 L1 交易。

## 为什么需要它

L2 上一次朴素的隐私交互是这样的:

```
User → MetaMask → eth_sendRawTransaction
                  Shield.deposit(commitment, asset, amount)
                  Gas: ~280,000 × 1 gwei = 0.00028 ATOS
```

从绝对数值来看这笔成本可以忽略不计,但它存在三个严重的 UX
问题:

1. **热钱包必须持有 gas。** 一个要屏蔽存入 wATOS 的用户既需要 wATOS,
   又需要少量 L2 gas 代币。新用户两者都没有。
2. **隐私泄露。** 支付 gas 的签名地址会在链上公开地与 deposit
   事件绑定在一起。即便承诺本身隐藏了 note,链上分析仍能以概率 1
   将"钱包 X 为 deposit Y 支付了 gas"关联起来。
3. **交易卡死。** 如果用户钱包在隐私流程中途耗尽 gas
   (例如要做 3 笔 deposit,但 gas 只够 2 笔),第三笔就会失败,
   用户不得不重新充值。

EnergySettlement 通过引入一个**元交易中继器**同时解决这三个问题:
该中继器持有 gas,代表用户签署 L2 交易,并通过桥向用户的 L1
能量账户计费。

## 架构

```
                    L1 (Atoshi Cosmos chain)
        ┌─────────────────────────────────────────────┐
        │ x/energy                                     │
        │   ↑ user delegates energy → relayer L1 addr  │
        │   ↑ relayer's DelegatedInUsable grows        │
        └─────────────────────────────────────────────┘
                              ↑
                              │ (bridge / Cosmos query)
                              │
                    L2 (Polygon zkEVM)
        ┌─────────────────────────────────────────────┐
        │ EnergySettlement.sol                         │
        │   • acceptSignedTx(user_sig, target, calldata)│
        │   • Verifies user_sig against the calldata    │
        │   • Forwards to Shield (or any target)        │
        │   • Records gas used per user (off-chain        │
        │     reconciled against L1 energy delegation)  │
        ├─────────────────────────────────────────────┤
        │ Shield.sol (target)                          │
        │   • deposit / transfer / withdraw            │
        │   • Doesn't care that caller is the relayer  │
        └─────────────────────────────────────────────┘
                              ↑
                              │ tx submitted by relayer EOA
                              │ (relayer pays L2 gas in wATOS)
                              │
                          Off-chain relayer service
                          (foundation-operated in v1)
```

三个参与方:

1. **用户** —— 拥有 L1 能量额度,希望在不支付 L2 gas 的情况下与 L2 Shield 交互
2. **中继器** —— 运营一个持有 wATOS 用作 gas 的 L2 EOA;在 v1 中由基金会运行
3. **EnergySettlement 合约** —— 验证用户授权,执行调用

经济模型:用户将 L1 能量委托给中继器的 L1 地址。
随后中继器每赞助一笔 L2 交易,就(在账务意义上)"花费"这份能量。
他们实际支付的 wATOS gas 会在后文所述的结算流程中得到补偿。

## 端到端存入流程

一个屏蔽存入 50 wATOS 的用户:

```
Step 1 — User: delegate L1 energy to the relayer (one-time setup)
  L1 tx: MsgDelegateEnergy {
    delegator: user_atoshi1...,
    delegatee: relayer_atoshi1...,
    amount:    1_000_000,      // energy units
    duration:  7 * 24 * 3600   // 7 days
  }
  Effect on L1:
    user.DelegatedOut          += 1,000,000
    user.LockedAtos            += 600,000 ATOS (collateral)
    relayer.DelegatedInUsable  += 1,000,000

Step 2 — User: prepare an L2 deposit
  Compute commitment = Poseidon(amount, owner_pubkey, secret)
  Sign EIP-712 typed data:
    {
      target:    Shield_address,
      calldata:  abi.encode(deposit(commitment, wATOS, 50))
      nonce:     user_l2_nonce,
      deadline:  now + 5 min
    }
  Signature is from the user's L2 private key.

Step 3 — User: POST to relayer's HTTP API
  POST https://relayer.atoshi.org/sponsor
  body: { typedData, signature, l1_address (for billing) }

Step 4 — Relayer: validate + submit
  Verifies:
    - signature recovers to user's L2 address
    - L1 address has DelegatedInUsable > estimated_gas
    - target is in the allow-list (Shield, etc.)
    - deadline not expired
  Submits to L2:
    EnergySettlement.acceptSignedTx(
      typedData, signature, gas_estimate
    )

Step 5 — EnergySettlement contract: execute
  Re-verifies signature on-chain (cheap, EIP-712).
  Calls target.call(calldata).
  Records event: SponsoredCall {user, l1_address, gas_used, target}.
  Returns success/failure to relayer.

Step 6 — Off-chain settlement (batch, every few hours)
  Relayer accumulates {l1_address, gas_used} pairs.
  Submits a single L1 tx debiting each user's energy:
    For each user: Consume(user_l1, gas_used) via a special precompile
    or via a Cosmos message.
  Relayer recovers their wATOS spend from the protocol treasury.
```

用户签署一笔 L1 交易(委托),以及每次被赞助的操作对应一条 L2 EIP-712
消息。每笔 L2 操作都不需要对应一笔 L1 交易。中继器支付 L2 gas,
并通过 L1 能量账务获得补偿。

## 为什么这样能保护隐私

三个属性至关重要:

### 隐私属性 1:中继器 ≠ 用户

被赞助调用在 L2 上的 `tx.origin` 是中继器的 EOA。进入 Shield 的
`msg.sender` 是 EnergySettlement 合约。**这笔交易中用户的 L2 EOA
从不出现在链上。**

区块浏览器看到的是"中继器进行了 deposit"或"EnergySettlement
合约进行了 deposit",而不是"用户 X 进行了 deposit"。无论由谁调用,
Shield.sol 中的承诺都是一样的。

### 隐私属性 2:签名是私密的

EIP-712 签名会包含在给 `acceptSignedTx` 的 calldata 中,但合约会:

- 恢复出签名者(用户的 L2 地址)
- 验证签名者与类型化数据中的 `from` 字段匹配
- 仅将恢复出的地址记录在一个内部映射中,而**不**记录到事件里

签名本身是明文 calldata;分析者**确实能**从中推导出用户的 L2 地址。
但是:

- 该 L2 地址是一次性的(钱包会为隐私操作生成全新地址)
- 签名只是授权的证明,而非实际存入金额或接收方的证明
  (后者都在承诺里)
- "L2 地址 → 承诺"这一关联,与用户直接签署一次 Shield 调用时
  已经公开的关联完全相同

因此这一属性是"不比直接调用更差"—— 不是严格的改进,
但也不是倒退。

### 隐私属性 3:L1 ↔ L2 不可关联性

L1 委托声明"用户 X 委托给中继器 Y"—— 在 L1 上是公开的。

L2 被赞助调用声明"中继器 Y 赞助了某个用户的 deposit"—— 在 L2 上
公开,但**不会透露是哪个 L1 用户**。

最后的 L1 结算批次声明"用户 [X, Z, W] 消耗了 [a, b, c] gas"——
在 L1 上公开,**但不会透露哪笔 L2 deposit 对应哪个用户**。

分析者看到的是:
- L1:用户 X 向中继器 Y 委托了 1M 能量
- L2:中继器 Y 本周做了 100 笔被赞助的 deposit
- L1:中继器 Y 本周结算了 300 个用户账户(X 是其中之一)

他们无法确定中继器 Y 的 100 笔 deposit 中哪一笔来自 X。
"用户 X 的可能 deposit"集合就是该时段内全部被赞助交易量构成的集合
—— 即匿名集。

**中继器赞助的交易量越大,匿名集就越大。** 每天做 10 笔 deposit
的中继器提供的隐私很弱;每天 10,000 笔则提供强隐私。这就是那条
根本性质:隐私随采用率而扩展。

## 结算账务细节

上文的链下对账步骤(Step 6)是信任敏感的部分。可以有不同的设计:

### v1 设计(中心化中继器 + ZK 收据)

中继器由基金会运营,并被信任会诚实地向用户计费。对于每次被赞助的
调用,EnergySettlement 都会发出一个带索引的事件,由中继器进行聚合。
每天一次:

1. 中继器将所有 `(user, gas)` 对的 Merkle 根发布到一笔 L1 交易
2. 任何人都可以通过重放 L2 事件来验证该根
3. 中继器根据该根触发 L1 能量扣减

如果中继器作弊(多计费或凭空捏造用户),用户可以在 L1 上通过提交
一个相互矛盾的 Merkle 证明来发起挑战。

### v2 设计(链上可证明结算)

每个结算批次一个 ZK 证明,用以证明:

- 批次中所有 `(user, gas)` 对都对应 L2 上真实的
  EnergySettlement 事件
- gas 总和与中继器的 L2 wATOS 支出相符(可验证)
- 每个用户都签署了对应的交易

这消除了对中继器的信任假设,但增加了证明成本。
v2 正在设计中;v1 先行发布以验证 UX。

### 边界外情况:gas 超支

如果一次被赞助的调用消耗的 gas **多于**预估(例如网络拥堵改变了
base fee,或者用户的调用在部分执行后回滚),中继器自行承担差额。
链上的 EnergySettlement 合约会对每次调用强制执行一个最大 gas
上限来限定这一风险。

如果一次被赞助的调用消耗的 gas **少于**预估(很常见 —— 高估是安全
的默认做法),未使用的部分会在结算中退还给中继器,而非用户。

## 中继器的经济模型

中继器的盈亏:

```
+  Sum of L1 energy debited from users (converted to "what would they have paid in ATOS gas")
−  Sum of L2 wATOS spent on gas
−  Operational costs (server, monitoring, multisig overhead)
=  Net margin
```

由于 L1 能量对持有者而言基本免费(他们"花费"的是终将再生的东西),
用户侧的成本是委托抵押品的锁定 —— 而不是能量本身。对中继器而言:

- 能量扣减来自委托,而委托受用户锁定抵押品意愿的限制
- L2 wATOS gas 受 Polygon zkEVM 实际 gas 市场的限制

当 L1 能量 → L2 wATOS 兑换比率为中性时,中继器收支平衡。协议可以
补贴中继器(例如通过项目金库),以在低交易量时段维持服务运转。

在 v1 中,基金会运营中继器,并将其视为一项营销开支 ——
为忠实持有者提供的隐私即服务。

## API 界面

### 面向用户的端点(HTTP,链下)

```
POST /sponsor
  body: {
    typed_data:    EIP712TypedData,
    signature:     bytes,
    l1_address:    atoshi1...
  }
  response: {
    l2_tx_hash:    "0x...",
    estimated_gas: 280_000,
    status:        "submitted" | "rejected"
  }
```

返回中继器所提交的 L2 交易哈希。用户可通过标准 L2 RPC 监控其被接受
的情况。

```
GET /quota?l1_address=atoshi1...
  response: {
    delegated_in_usable:  uint64,
    estimated_calls_left: uint32,
    expires_at:           int64
  }
```

这样钱包就能显示"你在本委托周期内还剩 X 次隐私操作"。

### EnergySettlement 合约 API(L2 EVM)

```
function acceptSignedTx(
    bytes32 typedDataHash,
    bytes calldata signature,
    address target,
    bytes calldata payload,
    uint256 maxGas
) external returns (bool success, bytes memory result);
```

`target` 必须在允许列表中(Shield 等)。`maxGas` 限定子调用的 gas;
即便该调用本会消耗更多,中继器支付的也不会超过这个值。返回调用结果。

发出:

```
event SponsoredCall(
    address indexed signer,    // user's L2 address
    address indexed target,
    uint256 gasUsed
);
```

链下索引器读取 `SponsoredCall` 事件,通过一个注册表将 signer 映射到
用户的 L1 地址,并据此构建结算批次。

## 运维考量

### 允许列表管理

EnergySettlement 的 `target` 允许列表由治理控制。初始仅包含
`Shield`。添加更多 target(例如一个隐私保护的 DEX)需要一份治理提案。
这就是审查向量:治理可以拒绝添加某个 target。

### 抢先交易风险

被赞助的 deposit 包含一个承诺哈希。理论上恶意中继器可以抢先执行
某笔 deposit,但承诺并不泄露 note 内容 —— 在经济上没有可抢先的东西。
transfer 和 withdraw 同理。

### 重放保护

每个 EIP-712 签名都包含一个 nonce + deadline。EnergySettlement
合约维护一个 `(signer, nonce)` 映射来拒绝重放。

### 中继器停机

如果中继器服务宕机,用户可以回退到直接调用(自行支付 L2 gas)。
Shield 合约并不要求赞助;赞助纯粹是一层 gas 支付者抽象。

这一点很重要:**隐私系统的安全性并不依赖于中继器的存活性**。
它只在无 gas 的 UX 上依赖中继器。

## 与类似设计的对比

| 系统 | Gas 支付者 | 隐私属性 | 信任 |
|---|---|---|---|
| 直接 Shield 调用 | 用户的 L2 EOA | 用户被公开地与 deposit 事件绑定 | 无需信任 |
| Tornado Cash 中继器 | 第三方中继器(从提现金额中支付) | 在 Tornado 匿名集内匿名 | 无需信任(由智能合约强制执行) |
| Aztec | 聚合器 | 强;聚合证明隐藏单笔 deposit | 无需信任 |
| **Atoshi(本方案) v1** | **基金会中继器** | **在被赞助交易集合内匿名** | **信任基金会,但可在 L1 发起挑战** |
| **Atoshi v2** | **ZK 可证明的中继器** | **与 v1 相同** | **无需信任** |

Atoshi 的独特之处在于 **L1 能量补偿模型** —— 用户用自己已经持有的
额度来"支付"隐私操作,而不是另外花费代币。这是 L1 设计哲学的自然
延伸。

## 状态(测试网)

- EnergySettlement 合约:已部署于 L2 测试网的
  `0xB515a4a438c168cf34F1ABEEa40a835a39af5625`
- 中继器服务:由基金会运营,已部分集成
- 钱包 UI 集成:计划于第 2 阶段(2026 年 Q4)进行

## 相关

- [隐私概览模块](../modules/07-privacy.md) —— Shield 合约规范
- [能量模块](../modules/01-energy.md) —— L1 能量委托
- [隐私池深入解析](./02-shield-pool.md) —— 被赞助的那些操作

---

*最后审阅:2026-06-10*
