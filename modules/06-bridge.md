# Bridge — moving assets between L1 and L2

The bridge is not a Cosmos SDK module. It is a set of EVM contracts
deployed on the L1 EVM execution layer (`x/evm`), forked from the
Polygon zkEVM v1 stack. The same contracts are mirrored on L2, and the
two halves coordinate to move ATOS (and other ERC20 assets) between
the layers.

## Mental model

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

L1 holds the canonical token reserves. L2 mints "wrapped" representations
when assets cross into it, and burns them when assets cross back out.
Every L2 batch is proven on L1 with a zkSNARK, so the bridge can trust
L2 state without trusting the sequencer.

## What's deployed

These are the contracts that ship with a working Atoshi bridge. Each one
is an OpenZeppelin TransparentUpgradeableProxy with the implementation
behind it.

| Contract | Role |
|---|---|
| `PolygonZkEVMDeployer` | CREATE2 factory; first thing deployed, used to deterministically place every other contract |
| `PolygonZkEVMBridgeV2` (L1) | Receives `bridgeAsset` calls, locks assets in escrow, emits `BridgeEvent` |
| `PolygonZkEVMBridgeV2` (L2) | Mirror on L2; mints/burns wrapped tokens, processes claims |
| `PolygonRollupManager` (L1) | Holds the list of L2 rollups, accepts state-root submissions from the sequencer/aggregator |
| `PolygonZkEVMGlobalExitRootV2` (L1) | Computes the unified exit-root Merkle tree across rollups + L1 |
| `PolygonZkEVMGlobalExitRootL2` (L2) | Mirror on L2 for cross-rollup exit verification |
| `FflonkVerifier` | Groth16+FFLONK proof verification; called by RollupManager |
| `PolygonZkEVMTimelock` | Governance gate for upgrading any of the above |

Source: [`atoshi-zkevm-contracts`](https://github.com/atoshi-chain/atoshi-zkevm-contracts).
The repo is a curated fork of `0xPolygonHermez/zkevm-contracts` with chain
id and admin address customizations.

Testnet (`atoshi_88288-1`) bridge addresses are documented in the
[wallet integration handbook](../reference/wallet-integration.md).
Mainnet addresses will be published after deployment by the governance
multisig + Timelock.

## Two flows: deposit and withdrawal

### L1 → L2 deposit

A user wants to use ATOS on L2 (e.g. interact with the shield pool).

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

Claim is permissionless: anyone (with the right SMT proof) can call
`claimAsset` on the user's behalf. Most wallets do this themselves; for
gasless flow, a relayer can sponsor.

### L2 → L1 withdrawal

Symmetric and reversed:

```
1. User on L2 calls L2 bridge.bridgeAsset(0, userL1, amount, token, ...)
2. L2 bridge burns the wrapped tokens, emits BridgeEvent on L2.
3. Sequencer batches the L2 state, posts root to L1 with ZK proof.
4. ~30s after L1 proof verification, user/relayer calls L1 bridge.claimAsset
   with the SMT proof from L2.
5. L1 bridge releases N ATOS from escrow to userL1.
```

Withdrawal latency is dominated by L2 batch finality (depends on
sequencer cadence + proof cost) — typically 10–30 minutes in our
configuration.

## Force-exit (censorship resistance)

The above flows depend on the L2 sequencer to include and prove user
withdrawals. If the sequencer goes offline or censors, users would be
unable to exit.

The bridge implements **force-exit**: any L1 user can submit a
`forceBatch` transaction directly to L1's `PolygonZkEVM` contract,
including their own withdrawal payload. The sequencer is obligated to
include this batch in the next N L1 blocks or face liquidation/slashing.
If no sequencer responds, governance can rotate to a backup sequencer.

This is the mechanism that makes the centralized-sequencer v1 acceptable
— users can always exit unilaterally.

## Withdrawal latency budget

For a typical L2 → L1 withdrawal:

| Stage | Time |
|---|---|
| User L2 tx finalized in a batch | 1 batch (~30s) |
| Sequencer posts batch to L1 + verifies proof | 5–15 min (depends on prover queue) |
| Global exit root reflected on L1 | ~30s after proof verified |
| User submits claim on L1 | 0s (manual) |
| **Total** | **~10–30 min** |

For deposits (L1 → L2), the budget is shorter (~30s–2min) because L2
finality is faster and there's no proof step in this direction.

## Gas costs

Approximate gas costs at current rates:

| Operation | L1 cost (gas) | L2 cost (gas) | At 1 gwei (ATOS) |
|---|---|---|---|
| `bridgeAsset` deposit (L1) | ~250,000 | — | 0.00025 ATOS |
| `claimAsset` deposit (L2) | — | ~280,000 | 0 (L2 has no fee in test) |
| `bridgeAsset` withdrawal (L2) | — | ~280,000 | 0 |
| `claimAsset` withdrawal (L1) | ~350,000 | — | 0.00035 ATOS |

Deposit total: ~0.00025 ATOS. Withdrawal total: ~0.00035 ATOS.
Independent of amount.

## Wrapped tokens on L2

When an ERC20 token first crosses into L2, the L2 bridge deploys a
canonical wrapped contract at a deterministic address. Subsequent
deposits of the same token reuse this address. The wrapped contract
implements standard ERC20 plus `permit` and `bridgeBurn`.

Native ATOS gets wrapped as **wATOS** at a known address on L2. The
shield pool exclusively accepts wATOS (not native L2 gas token), so a
deposit flow looks like: user has ATOS on L1 → bridge to L2 → user has
wATOS on L2 → user deposits into shield.

## Sequencer and prover (v1 model)

| Role | Operator |
|---|---|
| Sequencer | Foundation-operated, single node |
| Aggregator / Prover | Foundation-operated |
| Force-exit fallback | Anyone (permissionless) |

This is the standard Polygon CDK v1 trust model. Sequencer cannot steal
funds (proofs would fail) but can censor by refusing to include certain
transactions. Force-exit on L1 is the unconditional fallback.

Future versions plan to decentralize sequencing via the Polygon AggLayer
shared sequencer protocol.

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| L1 deposit succeeded, no L2 mint | Forgot to call `claimAsset` on L2 | Bridges are not auto-claim; user (or relayer) must complete the claim |
| `claimAsset` reverts with "global exit root invalid" | L2 batch with the deposit hasn't been proved yet | Wait for the proof step, retry |
| Sequencer not responding | L2 deposits/withdrawals queue up | Submit `forceBatch` on L1 |
| Token transferred to wrong network id | Funds locked but unspendable in destination | Bridge expects `destinationNetwork=1` for L2; L1 is `0` |
| L2 type-2 (EIP-1559) tx rejected | "transaction type not supported" | L2 sequencer accepts only type 0; build a legacy tx |

## Related modules

- [`x/evm`](./05-evm.md) — bridge contracts are EVM contracts on L1 EVM
- [`x/bank`](./04-bank.md) — the bridge escrows ATOS via standard bank transfer
- [Privacy module](./07-privacy.md) — shield pool consumes wATOS on L2

## Further reading

- [Polygon CDK architecture](https://docs.polygon.technology/cdk/)
- [Bridge deployment guide](https://github.com/atoshi-chain/atoshi-zkevm-contracts/blob/main/BRIDGE_DEPLOY.md)

---

*Last reviewed: 2026-06-10*
