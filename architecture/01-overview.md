# Architecture — high-level overview

This chapter introduces Atoshi's full system topology. Subsequent chapters in `architecture/` go deep on each layer.

## The 30-second picture

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

## Three things that make Atoshi different at the architecture level

### 1. EVM lives *inside* the Cosmos SDK chain at L1

Atoshi is not a Cosmos chain *with* an EVM rollup attached. The EVM is `x/evm`, a module that participates in the same block, same state machine, same consensus as the bank, oracle, energy, and tokenomics modules. This means:

- An EVM `eth_sendRawTransaction` and a Cosmos `MsgSend` can execute in the same block, read each other's state, and emit events into the same event index.
- ATOS balances are stored in `x/bank`; `x/evm` exposes them to the EVM by reading `bank.GetBalance(...)`. There is no wrapped representation at L1.
- A user's `0x...` address and `atoshi1...` address are the same on-chain account, addressable from either path. See [accounts](./05-accounts.md).

This is the Ethermint pattern, with our custom modules layered alongside.

### 2. Energy economics span only the Cosmos side at L1

The `x/energy` module hooks into the SDK ante-handler (`x/energy/ante`). It intercepts every Cosmos transaction *before* fee deduction:

1. Compute gas required by the tx (standard SDK).
2. Drain the signer's `TxEnergyAccrued` up to `gas_limit`.
3. If energy covers it: zero out the fee, skip the standard fee deduction path.
4. If energy is short: compute the ATOS shortfall at `params.insufficient_gas_price` and charge it through the bank module.

EVM transactions go through the Ethermint `MonoDecorator` instead, which does its own gas accounting in ATOS — energy is intentionally bypassed there. The reason is layering: porting energy into the EVM execution path requires touching `ApplyMessage`, which is intrusive. v1 keeps the boundary clean; future versions may revisit.

Practical implication: **for free transactions, use Cosmos messages.** EVM gas is always priced in ATOS. See [Gas and energy economics](../economics/03-gas-fee.md).

### 3. L2 is for things you don't want on L1

Anything privacy-related, anything that benefits from cheap EVM gas, anything experimental, lives on L2. Specifically:

- The shield pool — large state, frequent updates, ZK-verifier-heavy. Not appropriate for L1 blockspace.
- Application-specific contracts — DEX, lending, NFT marketplaces. Anyone can deploy on L2 without governance approval.
- High-frequency contract loops — game state, on-chain mechanics.

L1 is curated: only governance-approved modules live there. L2 is permissionless.

## Data flow: an example

To make this concrete, here is a single user journey — Alice transfers 100 ATOS to Bob, then bridges 50 ATOS to L2 to use the shield pool:

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

This three-step flow exercises both layers, both message types (Cosmos + EVM), the bridge, and the privacy pool. Each component is detailed in its own chapter.

## Validators, sequencers, and trust assumptions

| Layer | Block producer | Trust model | Slashing |
|---|---|---|---|
| L1 | Bonded PoS validators | Honest 2/3 majority by stake | Double-sign + downtime |
| L2 | Sequencer (centralized v1) | Liveness only; safety inherited from L1 via ZK proof | None at L2; L1 enforces validity |
| Bridge | L1 bridge contract + L1 validators | L1 consensus security | N/A |

In v1, the L2 sequencer is operated by the foundation. Censorship resistance is provided by the **force-exit** mechanism: any L2 user can submit a `forceClaimAsset` transaction directly to the L1 bridge to exit their funds without relying on the sequencer. The path is documented in [Bridge](./04-bridge.md).

## What's next

Continue reading in this order if you want depth:

1. [Cosmos L1 — modules, ante pipeline, state layout](./02-cosmos-l1.md)
2. [Polygon L2 — sequencer, prover, deposit/withdrawal cycles](./03-polygon-l2.md)
3. [Bridge — bridgeAsset and claimAsset semantics](./04-bridge.md)
4. [Accounts — ethermint key derivation and address mapping](./05-accounts.md)

Or jump straight to a module chapter under [`modules/`](../modules/).

---

*Last reviewed: 2026-05-21*
