# 架构 —— 高层概览

本章介绍 Atoshi 完整的系统拓扑。`architecture/` 目录下的后续章节会深入讲解每一层。

## 30 秒速览

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  L1 — Atoshi Chain  (chain id 88288)                                  │
│  Cosmos SDK + Ethermint EVM, ~5s blocks, PoS validators               │
│                                                                       │
│  ┌─ Cosmos SDK modules ────────────────────────────────────────────┐ │
│  │  x/bank          ATOS native transfers                          │ │
│  │  x/staking       validator delegation, slashing                 │ │
│  │  x/oracle        ATOS/USD price feed (allow-listed feeders)     │ │
│  │  x/tokenomics    release schedule driven by oracle + halving    │ │
│  │  x/energy        per-account TxEnergy accrual + delegation      │ │
│  │  x/feemarket     EIP-1559-style base fee tracking (EVM only)    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌─ EVM (x/evm) ──────────────────────────────────────────────────┐  │
│  │  Standard ERC20 / contract execution                           │  │
│  │  L1 bridge contract (Polygon CDK)                              │  │
│  │  Block reward distribution hook                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  Endpoints:  REST :1317   EVM JSON-RPC :8545   TM RPC :26657          │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
                                    ↕  (Polygon CDK Bridge)
┌──────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  L2 — Atoshi zkEVM  (chain id 67890)                                  │
│  Polygon CDK validium / rollup, ~2s blocks, centralized sequencer     │
│                                                                       │
│  ┌─ EVM contracts ────────────────────────────────────────────────┐  │
│  │  wATOS              wrapped ATOS, ERC20                        │  │
│  │  Shield             Pedersen-Poseidon commitment pool          │  │
│  │  EnergySettlement   meta-tx + L1 energy bridge                 │  │
│  │  TokenRegistry      curated token list for privacy pool        │  │
│  │  Poseidon           on-chain Poseidon hash precompile          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  Endpoints:  EVM JSON-RPC :8123                                       │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

## 在架构层面让 Atoshi 与众不同的三件事

### 1. EVM 在 L1 上运行于 Cosmos SDK 链*内部*

Atoshi 并不是一条*外挂*了 EVM rollup 的 Cosmos 链。这里的 EVM 就是 `x/evm`,一个与 bank、oracle、energy、tokenomics 等模块参与同一个区块、同一个状态机、同一套共识的模块。这意味着:

- 一笔 EVM 的 `eth_sendRawTransaction` 与一笔 Cosmos 的 `MsgSend` 可以在同一个区块中执行,读取彼此的状态,并把事件写入同一个事件索引。
- ATOS 余额存储在 `x/bank` 中;`x/evm` 通过读取 `bank.GetBalance(...)` 把余额暴露给 EVM。在 L1 上不存在任何包装形式的表示。
- 用户的 `0x...` 地址和 `atoshi1...` 地址是链上同一个账户,从任一路径都可寻址。参见 [账户](./05-accounts.md)。

这就是 Ethermint 模式,再叠加上我们自己的定制模块。

### 2. 能量经济学在 L1 上只覆盖 Cosmos 一侧

`x/energy` 模块挂接到 SDK 的 ante-handler(`x/energy/ante`)。它在扣除手续费*之前*拦截每一笔 Cosmos 交易:

1. 计算该交易所需的 gas(标准 SDK 逻辑)。
2. 在 `gas_limit` 范围内消耗签名者的 `TxEnergyAccrued`。
3. 如果能量足以覆盖:将手续费清零,跳过标准的手续费扣除流程。
4. 如果能量不足:按 `params.insufficient_gas_price` 计算 ATOS 差额,并通过 bank 模块收取。

EVM 交易走的是 ethermint 的 `MonoDecorator`,它以 ATOS 自行完成 gas 核算 —— 能量在这里是被有意绕过的。原因在于分层:把能量移植进 EVM 执行路径需要改动 `ApplyMessage`,侵入性很强。v1 保持了边界清晰;未来版本可能会重新审视这一点。

实践含义:**要享受免费交易,请使用 Cosmos 消息。** EVM gas 始终以 ATOS 计价。参见 [Gas 与能量经济学](../economics/03-gas-fee.md)。

### 3. L2 承载那些你不希望放在 L1 上的东西

任何与隐私相关的、任何能从廉价 EVM gas 中获益的、任何实验性的东西,都放在 L2 上。具体来说:

- 隐私池 —— 状态庞大、更新频繁、大量依赖 ZK 验证器。不适合占用 L1 区块空间。
- 应用专用合约 —— DEX、借贷、NFT 市场。任何人都可以在 L2 上部署,无需治理批准。
- 高频合约循环 —— 游戏状态、链上机制。

L1 是经过精选的:只有治理批准的模块才能在此运行。L2 是无需许可的。

## 数据流:一个示例

为了把上述内容讲具体,这里给出一段完整的用户旅程 —— Alice 向 Bob 转账 100 ATOS,然后把 50 ATOS 跨到 L2 以使用隐私池:

```
1. Alice signs MsgSend(from=alice, to=bob, amount=100 ATOS)
       │
       │   Cosmos transaction, signed with ethsecp256k1 + keccak256
       │
       ▼
   L1 mempool → block N
       │
       │   AnteHandler: drains 100k gas worth of TxEnergy from Alice
       │   Bank module: deducts 100 ATOS from alice, credits bob
       │
       ▼
   Event: cosmos.bank.v1beta1.EventSend
       │
       ▼
   Alice's wallet sees: balance -100 ATOS, +0 ATOS gas (energy covered it)


2. Alice signs an EVM tx calling bridge.bridgeAsset(...)
       │
       │   Standard Ethereum-style tx, type 0 (legacy)
       │
       ▼
   L1 mempool → block N+1
       │
       │   x/evm: executes the bridgeAsset call, locks 50 ATOS in bridge contract
       │   EVM gas: paid in ATOS at 2 gwei × 250k gas = 0.0005 ATOS
       │
       ▼
   Event: BridgeEvent(destination=L2, amount=50, to=alice)
       │
       ▼
   ~30s later: L2 sequencer observes the global exit root, mints 50 wATOS
       on L2 to Alice's address (same 0x... since same key).
       │
       ▼
   Alice's L2 balance: +50 wATOS


3. Alice deposits 50 wATOS into the shield pool on L2
       │
       │   L2 EVM tx, type 0 only (sequencer hard limit)
       │
       ▼
   Shield contract: takes wATOS, emits commitment hash
       │
       ▼
   Alice's wallet stores the commitment + nullifier secret locally.
       The on-chain ledger no longer associates this 50 ATOS with Alice.
```

这个三步流程贯穿了两层、两种消息类型(Cosmos + EVM)、跨层桥以及隐私池。每个组件都会在各自的章节中详细展开。

## 验证人、定序器与信任假设

| 层 | 出块方 | 信任模型 | 惩罚(slashing) |
|---|---|---|---|
| L1 | 已质押的 PoS 验证人 | 按质押量计的诚实 2/3 多数 | 双签 + 掉线 |
| L2 | 定序器(v1 中中心化) | 仅保证活性;安全性通过 ZK 证明从 L1 继承 | L2 上无惩罚;由 L1 强制保证有效性 |
| 跨层桥 | L1 桥合约 + L1 验证人 | L1 共识安全性 | 不适用 |

在 v1 中,L2 定序器由基金会运营。抗审查性由**强制退出(force-exit)**机制提供:任何 L2 用户都可以直接向 L1 桥合约提交一笔 `forceClaimAsset` 交易来退出自己的资金,而无需依赖定序器。该路径记录在 [跨层桥](./04-bridge.md) 中。

## 接下来

如果你想深入了解,请按以下顺序继续阅读:

1. [Cosmos L1 —— 模块、ante 流水线、状态布局](./02-cosmos-l1.md)
2. [Polygon L2 —— 定序器、证明器、存款/提款周期](./03-polygon-l2.md)
3. [跨层桥 —— bridgeAsset 与 claimAsset 语义](./04-bridge.md)
4. [账户 —— ethermint 密钥派生与地址映射](./05-accounts.md)

或者直接跳到 [`modules/`](../modules/) 下的某一篇模块章节。

---

*最后审阅:2026-05-21*
