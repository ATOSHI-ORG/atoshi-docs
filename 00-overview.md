# Atoshi — Overview

## What Atoshi is

Atoshi is a **dual-layer public blockchain** designed around a simple thesis:

> Everyday users should not pay gas fees. Holding the chain's native token should *be* the gas.

Concretely, Atoshi consists of:

- **L1 (chain id 88288)** — a Cosmos SDK chain with a built-in EVM execution layer (via Ethermint). It runs the core economic modules: token release, price oracle, energy accounting, and the native bridge contract.
- **L2 (chain id 67890)** — a Polygon CDK zkEVM rollup that hosts the privacy pool and high-throughput smart contracts. L2 inherits L1's security via zero-knowledge proofs.

Both layers settle in **ATOS**, the native token. ATOS is bridgeable in both directions and is the only fee asset.

## What problem we solve

Public chains face a structural tension between three goals:

1. **Permissionless access** — anyone can transact.
2. **Spam resistance** — scarce blockspace must be priced.
3. **Approachable UX** — fees should not be a friction at every action.

Most chains solve (1) and (2) by sacrificing (3): gas pricing in volatile native tokens, requiring users to constantly top up. Atoshi takes a different stance: **price the right to transact in token-holding rather than token-spending**.

If you hold enough ATOS, your wallet *accrues* transaction capacity (we call it "energy") at a fixed rate, capped by your balance. Spending energy is free at the token level — only the underlying ATOS holding requirement is enforced.

A user holding 30,000 ATOS gets 50,000 gas units of free transaction capacity every 24 hours, refilling continuously. A user holding 1,000,000 ATOS additionally gets free contract-deployment capacity. Users who hold less, or who want to send more than their capacity, fall back to standard gas-priced transactions — but the floor is low.

## Why dual-layer

Cosmos SDK gives us:

- **Mature module composition** — bank, staking, governance, IBC, ethermint EVM, plus our custom modules (energy, oracle, tokenomics) are all proper SDK modules.
- **Validator-driven block production** — proof-of-stake from day one, no PoW transition needed.
- **Native account abstraction** — every account is both an EVM `0x...` address and a Cosmos `atoshi1...` bech32 address with one underlying secp256k1 key.

Polygon CDK gives us:

- **EVM-equivalent throughput** at L2 — privacy pools and high-frequency dApps live here.
- **Validity proofs** — every L2 block is verified by a zkSNARK on L1, so withdrawals are trust-minimized.
- **A canonical bridge** — `bridgeAsset` and `claimAsset` on the L1 bridge contract, mirrored on L2.

L1 stays small, opinionated, and modular; L2 absorbs the long tail of programmability without bloating consensus.

## Headline numbers

| Property | Value |
|---|---|
| Native token | ATOS |
| Atomic unit | 1 ATOS = 10^18 aatos (same precision as ETH) |
| L1 block time | ~5 seconds |
| L2 block time | ~2 seconds |
| L1 chain id | 88288 |
| L2 chain id | 67890 |
| Bech32 prefix | `atoshi` |
| Key scheme | ethsecp256k1 (Ethereum-compatible) |
| Hashing for tx signing | keccak256 |
| Halving cycle | every 1,051,200 blocks (~4 years at 5s blocks) |
| Genesis circulating | 1.46 B ATOS (immediate distribution) |
| Locked initial supply | 5.84 B ATOS (released over miner + project schedule) |

## Core design choices

These are the decisions that distinguish Atoshi. Each is detailed in its own chapter; this section is a guided tour.

### 1. Energy instead of gas

Standard EVM chains charge `gasPrice * gasUsed` in the native token for every transaction. Atoshi's L1 Cosmos chain replaces this with a two-tier model:

1. **Energy first.** The signer's account accrues `TxEnergy` continuously, capped at `floor(balance / 30,000 ATOS) × 50,000` units. Each transaction consumes from this pool.
2. **ATOS fallback.** If energy is insufficient, the chain charges the *shortfall* in ATOS at a fixed price (currently 0.0021 aatos per gas unit). Users who hold enough never see this.

Energy regenerates linearly over 24 hours regardless of holding (as long as the holding threshold is met). It cannot be transferred at the token level, but can be *delegated* — an account can lend energy to another for a fixed duration, by locking the corresponding ATOS as collateral. This enables sponsored-transaction services without giving up custody.

Subsidized message types (e.g. `MsgDelegateEnergy`, `MsgUndelegateEnergy`, on-chain claim messages) bypass even the energy check — they are completely free. See [Energy module](./modules/01-energy.md).

### 2. Token release driven by on-chain price

ATOS supply unlocks on a schedule that depends on real ATOS/USD price observed by the on-chain oracle. The chain tracks four tiers (T0–T3); each tier corresponds to a price band. Sustained presence in a higher tier accelerates the release of miner and project-treasury allocations. Time spent below a tier rolls back the counter.

This couples token issuance to demand — the supply curve is endogenous, not pre-determined. See [Tokenomics module](./modules/02-tokenomics.md) and [Release schedule](./economics/02-release-schedule.md).

### 3. Native zkEVM privacy

L2 hosts a **shield pool** contract: users deposit ATOS or wATOS, get a Pedersen-Poseidon commitment, and can spend that commitment privately by submitting a Groth16 proof. Withdrawals reveal only the destination address and the amount — sender history is hidden.

Privacy interactions optionally settle their gas through L1 energy via the `EnergySettlement` contract, so private transactions can also be free for committed holders. See [Privacy modules](./modules/07-privacy.md) and the deep-dive in [`privacy/`](./privacy/).

### 4. Ethermint key compatibility

A single secp256k1 private key gives the user:

- An EVM `0x...` address (`keccak256(pubkey)[12:32]`)
- A Cosmos `atoshi1...` bech32 address (`bech32_encode("atoshi", <same 20 bytes>)`)

Both addresses refer to the same on-chain account. Wallets only need to manage one key. Tx signing for Cosmos messages uses `keccak256(SignDoc)` rather than vanilla Cosmos's `sha256`, but otherwise the SDK pipeline is unchanged. See [Accounts](./architecture/05-accounts.md).

## What's NOT in scope

We are deliberate about what Atoshi does and does not try to be:

- **Not a memecoin platform.** Asset launches on L2 are fine, but there's no built-in launchpad and we don't subsidize creators.
- **Not a privacy-by-default chain.** Privacy is opt-in via the L2 shield pool. The base L1/L2 ledgers are transparent.
- **Not IBC-first.** The chain enables IBC because Cosmos SDK does, but it is not a primary integration target in v1.
- **Not a DeFi hub.** L1 stays minimal; L2 is permissionless EVM, but core dev focus is on the privacy + energy primitives, not on AMMs or lending.

## How to read these docs

The documentation is organized by **layer of abstraction**, not by file:

- **00 overview** (this page) → the why
- **architecture/** → the how, at a system level
- **modules/** → the how, per Cosmos SDK module (one file per module, with proto schemas, params, message-level behavior, edge cases)
- **economics/** → the why for token-level decisions
- **privacy/** → ZK constructions, threat model, circuit details
- **reference/** → integrator-facing API + handbook material

Each module chapter follows the same template: purpose → state → messages → queries → events → params → edge cases → related modules. If you've read one, you know how to navigate the others.

## Status

| Item | Status | Notes |
|---|---|---|
| L1 mainnet | testnet only | chain id 88288, public testnet endpoint live |
| L2 mainnet | testnet only | chain id 67890; bridge claim path being hardened |
| Energy module | live | params governance-tunable |
| Privacy pool | testnet-only, Phase 2 | circuits audited but not in mainnet wallet UI yet |
| Bridge claim (L2→L1) | partial | deposit works, claim flow in active development |

---

*Last reviewed: 2026-05-21*
