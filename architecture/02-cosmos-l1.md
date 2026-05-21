# Cosmos SDK L1 with EVM

> **Status: draft outline.** This chapter will detail the L1 chain in full. Sections marked TODO are stubs to be filled in next iteration.

## What L1 looks like at runtime

A Cosmos SDK v0.50 chain with ethermint EVM module, ABCI++, ~5s blocks, PoS validators. Genesis configures a foundation-operated validator set; permissionless validator joining via standard `MsgCreateValidator`.

## Module list (alphabetical)

| Module | Source | Purpose |
|---|---|---|
| `x/auth` | SDK | Account / sequence / pubkey storage; ethermint `EthAccount` extension |
| `x/authz` | SDK | Grant-based authorization (unused in v1 but available) |
| `x/bank` | SDK | ATOS transfers, balance queries, supply tracking |
| `x/distribution` | SDK | Validator commission + block-reward routing |
| `x/energy` | atoshi | Holding-based gas system. See [Energy module](../modules/01-energy.md) |
| `x/epochs` | evmos | Period-based hook scheduler (used by inflation/erc20 in evmos; minimal use here) |
| `x/erc20` | evmos | Native-token ↔ ERC20 conversion path |
| `x/evm` | ethermint | EVM execution layer |
| `x/feemarket` | ethermint | EIP-1559 base-fee tracking for EVM |
| `x/gov` | SDK | On-chain governance proposals |
| `x/ibc` | SDK | Inter-blockchain communication (passive in v1) |
| `x/inflation` | evmos | DISABLED in atoshi — no minting via inflation |
| `x/oracle` | atoshi | ATOS/USD price feed. See [Oracle module](../modules/03-oracle.md) |
| `x/slashing` | SDK | Validator misbehavior penalties |
| `x/staking` | SDK | Bonding / delegation / unbonding |
| `x/tokenomics` | atoshi | Block reward emission + tier-driven release. See [Tokenomics module](../modules/02-tokenomics.md) |
| `x/upgrade` | SDK | Coordinated chain upgrades |

## Ante-handler pipeline

The ante-handler is where atoshi diverges most from a vanilla Cosmos chain. The pipeline (in order):

1. `SetUpContextDecorator` — gas meter initialization
2. `RejectExtensionOptionsDecorator` — security: reject unknown tx extensions
3. `MempoolFeeDecorator` — applies only outside the energy path; minimum gas price check
4. `ValidateBasicDecorator` — message-level validation
5. `TxTimeoutHeightDecorator` — `timeout_height` check
6. `ValidateMemoDecorator` — memo length cap
7. `ConsumeTxSizeGasDecorator` — gas charged for tx size in bytes
8. **`EnergyDeductDecorator`** — atoshi's replacement for `DeductFeeDecorator`. See [Energy module §Consumption](../modules/01-energy.md#consumption-the-ante-handler-pipeline).
9. `SetPubKeyDecorator`, `ValidateSigCountDecorator`, `SigGasConsumeDecorator`, `SigVerificationDecorator` — signature handling
10. `IncrementSequenceDecorator` — sequence increment

For EVM transactions (`MsgEthereumTx`-wrapped), a separate `MonoDecorator` from ethermint takes over and does NOT consult the energy module. See [Gas economics](../economics/03-gas-fee.md) for the implications.

## Block lifecycle

- **BeginBlock**: `x/tokenomics` mints block reward, distributes 20% immediate share to proposer, accumulates 80% in miner-locked pool. Then checks tier conditions and possibly emits a tier-release event. `x/distribution` does validator commission accounting.
- **DeliverTx** loop: each tx flows through ante-handler then message handlers.
- **EndBlock**: `x/energy.SweepExpiredDelegations` releases expired delegations. `x/staking` processes unbonding queue. `x/slashing` updates downtime counters.

## TODO sections

- L1 endpoints summary (REST, RPC, EVM JSON-RPC) — overlaps with [API reference](../reference/api-guide.md)
- Genesis state structure and validator key handling
- Upgrade workflow with cosmosvisor
- Snapshot + statesync configuration
- Sentry-validator architecture recommendations

---

*Last reviewed: 2026-05-21*
