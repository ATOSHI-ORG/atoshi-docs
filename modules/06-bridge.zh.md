# 桥 —— 在 L1 与 L2 之间转移资产

桥并不是一个 Cosmos SDK 模块。它是一组部署在
L1 EVM 执行层(`x/evm`)上的 EVM 合约,从
Polygon zkEVM v1 技术栈 fork 而来。相同的合约在 L2 上被镜像部署,
两半协同工作,在两层之间转移 ATOS(以及其他 ERC20 资产)。

## 心智模型

```
                            L1 (Atoshi chain, chain id 88288)
   ┌──────────────────────────────────────────────────────────────┐
   │  PolygonZkEVMBridgeV2  ←─ user calls bridgeAsset(amount,...)│
   │  PolygonRollupManager  ←─ posts L2 state roots               │
   │  PolygonZkEVMGlobalExitRootV2 ←─ tree of cross-layer roots   │
   │  FflonkVerifier        ←─ checks Groth16+FFLONK proofs       │
   └──────────────────────────────────────────────────────────────┘
                            │             ▲
                            │             │  zkSNARK proof of
                  L1 → L2   │             │  L2 batch validity
                  deposit   ▼             │
                            │             │
   ┌──────────────────────────────────────────────────────────────┐
   │                L2 (Atoshi zkEVM, chain id 67890)             │
   │  PolygonZkEVMBridgeV2 (L2 side, mirrored)                    │
   │  + native L2 wATOS, Shield privacy pool, EnergySettlement    │
   └──────────────────────────────────────────────────────────────┘
```

L1 持有权威的代币储备。当资产跨入 L2 时,L2 会铸造"封装的"
表示,并在资产跨出时销毁它们。
每一个 L2 batch 都通过 zkSNARK 在 L1 上被证明,因此桥可以信任
L2 状态而无需信任 sequencer。

## 部署了哪些内容

以下是一个可用的 Atoshi 桥所附带的合约。每一个
都是一个 OpenZeppelin TransparentUpgradeableProxy,其背后是实现合约。

| 合约 | 角色 |
|---|---|
| `PolygonZkEVMDeployer` | CREATE2 工厂;第一个被部署的合约,用于确定性地放置其他每一个合约 |
| `PolygonZkEVMBridgeV2`(L1) | 接收 `bridgeAsset` 调用,将资产锁入托管,发出 `BridgeEvent` |
| `PolygonZkEVMBridgeV2`(L2) | L2 上的镜像;铸造/销毁封装代币,处理领取(claim) |
| `PolygonRollupManager`(L1) | 持有 L2 rollup 列表,接受来自 sequencer/aggregator 的状态根提交 |
| `PolygonZkEVMGlobalExitRootV2`(L1) | 跨 rollup + L1 计算统一的退出根 Merkle 树 |
| `PolygonZkEVMGlobalExitRootL2`(L2) | L2 上用于跨 rollup 退出验证的镜像 |
| `FflonkVerifier` | Groth16+FFLONK 证明验证;由 RollupManager 调用 |
| `PolygonZkEVMTimelock` | 升级上述任一合约的治理关卡 |

来源:[`atoshi-zkevm-contracts`](https://github.com/atoshi-chain/atoshi-zkevm-contracts)。
该仓库是 `0xPolygonHermez/zkevm-contracts` 的一个经过精选的 fork,做了 chain
id 和管理员地址的定制。

测试网(`atoshi_88288-1`)的桥地址记录在
[钱包集成手册](../reference/wallet-integration.md)中。
主网地址将在治理多签 + Timelock 完成部署后发布。

## 两条流程:存入与提取

### L1 → L2 存入

用户希望在 L2 上使用 ATOS(例如与 shield 池交互)。

```
1. User signs an EVM tx calling:
       bridge.bridgeAsset(
           destinationNetwork = 1,       // L2 network id
           destinationAddress = userL2,  // can be same as L1 address
           amount             = N,
           token              = 0x0,     // 0x0 = native ATOS, or ERC20 address
           forceUpdateGlobal  = true,
           permitData         = 0x       // empty for non-permit tokens
       )

2. The L1 bridge contract:
       - Receives N ATOS from user's account (msg.value for native, transferFrom for ERC20)
       - Locks them in its own contract balance (escrow)
       - Increments depositCount, emits BridgeEvent
       - Updates the global exit root (a Merkle tree of all bridge events)

3. The L2 sequencer observes the new global exit root,
   includes it in a L2 batch, produces a proof on L1.

4. ~30s–10m later, user calls (or a relayer calls) the L2 bridge:
       bridge.claimAsset(
           smtProofLocalExitRoot,    // Merkle path from BridgeEvent to local root
           smtProofRollupExitRoot,    // Merkle path to global root
           globalIndex,                // depositCount of the event
           mainnetExitRoot,
           rollupExitRoot,
           originNetwork = 0,
           originTokenAddress = 0x0,
           destinationNetwork = 1,
           destinationAddress = userL2,
           amount = N,
           metadata = 0x
       )

5. L2 bridge verifies the proof against the locally-stored root,
   marks this claim as consumed, and either:
       - Mints N L2-native wrapped ATOS to userL2 (first time), or
       - Releases N wrapped ATOS from L2 escrow (subsequent times)
```

领取(claim)是无许可的:任何人(只要持有正确的 SMT 证明)都可以代表用户调用
`claimAsset`。大多数钱包自己完成这一步;对于
无 gas 流程,relayer 可以代付。

### L2 → L1 提取

对称且反向:

```
1. User on L2 calls L2 bridge.bridgeAsset(0, userL1, amount, token, ...)
2. L2 bridge burns the wrapped tokens, emits BridgeEvent on L2.
3. Sequencer batches the L2 state, posts root to L1 with ZK proof.
4. ~30s after L1 proof verification, user/relayer calls L1 bridge.claimAsset
   with the SMT proof from L2.
5. L1 bridge releases N ATOS from escrow to userL1.
```

提取延迟主要由 L2 batch 的最终性决定(取决于
sequencer 节奏 + 证明成本)——在我们的配置中通常为 10–30 分钟。

## 强制退出(抗审查)

上述流程依赖 L2 sequencer 来包含并证明用户的
提取。如果 sequencer 下线或进行审查,用户将
无法退出。

桥实现了**强制退出(force-exit)**:任何 L1 用户都可以直接向 L1 的
`PolygonZkEVM` 合约提交一笔 `forceBatch` 交易,
其中包含自己的提取载荷。sequencer 有义务
在接下来的 N 个 L1 区块内包含这个 batch,否则将面临清算/罚没。
如果没有 sequencer 响应,治理可以轮换到一个备用 sequencer。

正是这个机制使得中心化 sequencer 的 v1 可以被接受
——用户总能单方面退出。

## 提取延迟预算

对于一次典型的 L2 → L1 提取:

| 阶段 | 时间 |
|---|---|
| 用户的 L2 交易在一个 batch 中最终确定 | 1 个 batch(~30s) |
| sequencer 将 batch 提交到 L1 + 验证证明 | 5–15 分钟(取决于 prover 队列) |
| 全局退出根反映到 L1 | 证明验证后 ~30s |
| 用户在 L1 上提交领取 | 0s(手动) |
| **合计** | **~10–30 分钟** |

对于存入(L1 → L2),预算更短(~30s–2min),因为 L2
最终性更快,而且这个方向上没有证明步骤。

## Gas 成本

按当前费率的大致 gas 成本:

| 操作 | L1 成本(gas) | L2 成本(gas) | 按 1 gwei(ATOS) |
|---|---|---|---|
| `bridgeAsset` 存入(L1) | ~250,000 | — | 0.00025 ATOS |
| `claimAsset` 存入(L2) | — | ~280,000 | 0(测试中 L2 无手续费) |
| `bridgeAsset` 提取(L2) | — | ~280,000 | 0 |
| `claimAsset` 提取(L1) | ~350,000 | — | 0.00035 ATOS |

存入合计:~0.00025 ATOS。提取合计:~0.00035 ATOS。
与金额无关。

## L2 上的封装代币

当一个 ERC20 代币首次跨入 L2 时,L2 桥会在一个
确定性地址上部署一个权威的封装合约。同一代币后续的
存入会复用此地址。该封装合约
实现标准 ERC20,外加 `permit` 和 `bridgeBurn`。

原生 ATOS 在 L2 上被封装为 **wATOS**,位于一个已知地址。
shield 池仅接受 wATOS(而非 L2 原生 gas 代币),因此一个
存入流程看起来是:用户在 L1 上有 ATOS → 桥接到 L2 → 用户拥有
L2 上的 wATOS → 用户将其存入 shield。

## Sequencer 与 prover(v1 模型)

| 角色 | 运营方 |
|---|---|
| Sequencer | 基金会运营,单节点 |
| Aggregator / Prover | 基金会运营 |
| 强制退出回退 | 任何人(无许可) |

这是标准的 Polygon CDK v1 信任模型。sequencer 无法窃取
资金(证明会失败),但可以通过拒绝包含某些
交易来进行审查。L1 上的强制退出是无条件的回退方案。

未来版本计划通过 Polygon AggLayer
共享 sequencer 协议来去中心化排序。

## 常见陷阱

| 陷阱 | 症状 | 解决办法 |
|---|---|---|
| L1 存入成功,但 L2 没有铸造 | 忘了在 L2 上调用 `claimAsset` | 桥不会自动领取;用户(或 relayer)必须完成领取 |
| `claimAsset` 回滚并报 "global exit root invalid" | 包含该存入的 L2 batch 尚未被证明 | 等待证明步骤,重试 |
| Sequencer 无响应 | L2 存入/提取排队积压 | 在 L1 上提交 `forceBatch` |
| 代币被转到了错误的 network id | 资金被锁定但在目标层无法花费 | 桥要求 L2 的 `destinationNetwork=1`;L1 为 `0` |
| L2 的 type-2(EIP-1559)交易被拒绝 | "transaction type not supported" | L2 sequencer 仅接受 type 0;构建一笔 legacy 交易 |

## 相关模块

- [`x/evm`](./05-evm.md) —— 桥合约是 L1 EVM 上的 EVM 合约
- [`x/bank`](./04-bank.md) —— 桥通过标准银行转账托管 ATOS
- [隐私模块](./07-privacy.md) —— shield 池在 L2 上消耗 wATOS

## 延伸阅读

- [Polygon CDK 架构](https://docs.polygon.technology/cdk/)
- [桥部署指南](https://github.com/atoshi-chain/atoshi-zkevm-contracts/blob/main/BRIDGE_DEPLOY.md)

---

*最后审阅:2026-06-10*
