# Privacy — shielded transactions on L2

Atoshi's privacy system lives entirely on L2 (the Polygon CDK zkEVM). It is implemented as a set of EVM contracts (`Shield`, `EnergySettlement`, `TokenRegistry`, `Verifier`) plus Groth16 zk-circuits compiled against the BN254 curve. This chapter is the module overview; the deep mechanics are in [`privacy/`](../privacy/).

## Mental model

A standard EVM transfer reveals (sender, recipient, amount). A shielded transfer reveals only (amount, possibly recipient). The sender and the history of the funds are hidden.

This is achieved by:

1. **Commitment scheme.** Each "note" is committed as `commit = Poseidon(amount, owner_pubkey, nullifier_secret)`. Commitments are added to an append-only Merkle tree (the "shield pool").
2. **Nullifier scheme.** To spend a note, the owner reveals `nullifier = Poseidon(secret)` — a tag that proves the note has been consumed without revealing which one it was. The chain stores spent nullifiers; double-spend is impossible.
3. **Zero-knowledge proofs.** All state transitions (deposit, transfer, withdrawal) are proven with a Groth16 zkSNARK that checks the relation between old commitments, new commitments, nullifiers, and balances.

Atoshi's flavor adds two specifics:

- **Hybrid asset model.** Notes can hold native L2 wATOS or a curated whitelist of L2 ERC20 tokens (governed by `TokenRegistry`). Cross-asset shielded transfer is supported but each asset has its own sub-tree.
- **Energy-settled gas.** The `EnergySettlement` contract on L2 talks to the L1 `x/energy` module via meta-transaction so that committed L1 holders pay zero gas for privacy operations on L2.

![Dual-mode transaction computing layer — transparent (default) vs private (user-activated)](../assets/whitepaper/privacy-dual-mode.png)

> The diagram frames privacy as a per-transaction user toggle. On Atoshi, the transparent mode is the ordinary L1/L2 ledger; the private mode is the L2 shield pool described below. The zk-SNARK / stealth-address / commitment techniques it references map to the concrete circuits in the [Contracts](#contracts) and [Circuits](#circuits) sections.

## Three operations

### 1. Shield (deposit)

```
Public:  amount, sender_pubkey, asset
Private: nullifier_secret

User does:
  commit = Poseidon(amount, sender_pubkey, nullifier_secret)
  proof  = Groth16(circuit_shield)(amount, sender, secret → commit)
  call Shield.deposit(asset, amount, commit, proof)

Contract checks:
  - proof is valid
  - sender transfers `amount` of `asset` into the pool
  - append commit to the Merkle tree, emit MerkleUpdate
```

After this transaction, on-chain analysis can see "this user deposited X tokens", but cannot link the deposit to subsequent withdrawals or transfers. The Note (commit, secret, position in tree) is stored client-side by the user's wallet.

### 2. Private transfer

A shielded transfer takes one or more input notes and produces one or more output notes:

```
Public:  asset, encrypted_recipient_payloads, new_commitments[], nullifiers[]
Private: input notes, output amounts, output owner pubkeys, fresh secrets

User does:
  For each input note: compute nullifier
  For each output note: compute commit
  proof = Groth16(circuit_transfer)(inputs, outputs → balanced)
  call Shield.transfer(nullifiers, new_commits, proof)

Contract checks:
  - all input nullifiers not previously spent
  - proof asserts Σ inputs == Σ outputs
  - mark nullifiers spent, append new commits
```

The recipient learns of the new note by:
- ECDH-encrypted payload posted in tx data, decryptable by recipient's view key
- Or: out-of-band sharing (the sender hands the note to the recipient through a side channel)

Both modes are supported by the SDK. Default is encrypted on-chain (no side-channel needed).

### 3. Unshield (withdraw)

```
Public:  amount, recipient_address, asset, nullifier
Private: input note, secret, Merkle path

User does:
  proof = Groth16(circuit_unshield)(input → nullifier, amount, recipient)
  call Shield.withdraw(asset, amount, recipient, nullifier, proof)

Contract checks:
  - nullifier not spent
  - proof asserts the input note exists in tree and has `amount` for `asset`
  - mark nullifier spent
  - transfer `amount` of `asset` to recipient
```

On-chain analysis sees "X tokens went to recipient" but cannot trace which deposit funded it.

## Contracts

| Contract | Address (L2 testnet) | Role |
|---|---|---|
| `Shield` | `0x81fAA0D0579c82d6b77FD759C198B507180E59E9` | Pool, deposit/transfer/withdraw, Merkle tree, nullifier set |
| `EnergySettlement` | `0xB515a4a438c168cf34F1ABEEa40a835a39af5625` | Meta-tx + L1 energy bridge for gasless private txs |
| `TokenRegistry` | `0xC79bd646541DBBC54e6e4A349D44e19C33b31aF5` | Curated list of supported assets |
| `Poseidon` | `0xC1d3Bb5B7b9f4f097e7cD0126608D498A2986DAe` | Library: Poseidon hash for the BN254 curve |
| `Verifier` (per circuit) | Embedded in Shield.sol | Groth16 verifier for shield / transfer / unshield circuits |

## Circuits

Compiled from Circom 2.x, proving system Groth16, curve BN254. Sources in [`atoshi-privacy-circuits`](https://github.com/atoshi-chain/atoshi-privacy-circuits).

| Circuit | Purpose | Public inputs | Private inputs |
|---|---|---|---|
| `shield.circom` | Prove that `commit` is a valid Poseidon hash of (`amount`, `owner_pubkey`, `secret`) | `amount`, `owner_pubkey`, `commit` | `secret` |
| `transfer.circom` | Prove input notes belong to caller, outputs balance, no double-spend | `merkle_root`, `nullifiers[]`, `new_commits[]` | `input_notes`, `output_amounts`, `output_owners`, `new_secrets`, `merkle_paths` |
| `unshield.circom` | Prove note ownership and withdrawal to specific recipient | `merkle_root`, `nullifier`, `amount`, `recipient`, `asset` | `input_note`, `secret`, `merkle_path` |

Trusted setup: Powers of Tau ceremony from Hermez (audited), plus per-circuit phase-2 contribution. Verification keys committed to the repo and embedded in `Shield.sol`.

## Threat model summary

What we hide:

- Transaction sender within the shielded pool
- Which deposit funded a given withdrawal
- The graph of shielded transfers (no link between notes)
- Note amounts (visible only at deposit and withdrawal)

What we do NOT hide:

- Total deposits and total withdrawals (visible as ERC20 transfers in/out of `Shield`)
- Aggregate pool size per asset
- The fact that a user has a balance in the pool (the deposit tx itself is on-chain)
- Timing patterns (large deposits followed by similar-amount withdrawals can be loosely correlated)

The privacy guarantee is **anonymity within the anonymity set** — i.e. once N users have deposited similar amounts, any one of them can withdraw without their specific deposit being identified. Larger pools give stronger privacy. The pool is per-asset and per-amount-bucket; the SDK enforces standard denominations (1 / 10 / 100 / 1000 ATOS) to maximize anonymity set overlap.

Detailed analysis in [`privacy/01-threat-model.md`](../privacy/01-threat-model.md).

## Energy-settled gas (the killer feature)

By default, L2 transactions cost ATOS gas — including the Groth16 proof verification, which is ~280k gas per call. A user shielding 10 ATOS would pay measurable gas. This is bad UX.

`EnergySettlement` solves it via a meta-transaction pattern:

1. User produces a private tx and signs it with their L2 address.
2. Instead of submitting directly, user posts the signed tx + their L1 bech32 address to `EnergySettlement`.
3. `EnergySettlement` accepts the tx, executes the shield call on the user's behalf, and bills L1 energy via a designated relayer that batches and submits proof-of-execution back to L1.
4. The relayer is reimbursed (in L2 gas) by the user's L1 energy delegation.

Effectively: a user who delegates 1M energy from L1 to the relayer can do thousands of private transfers without paying L2 gas directly. The relayer's economic model is `delegated_energy / cost_per_op`.

This bridges the L1 energy economics into L2 privacy operations. See [`privacy/04-energy-settlement.md`](../privacy/04-energy-settlement.md) for details.

## Status (testnet)

- Shield deposit: working
- Shield transfer: working
- Shield unshield: working
- EnergySettlement: deployed; relayer integration partial
- Token registry: foundation-curated list of 5 assets initially
- Wallet UI integration: planned for Phase 2 (Q4 2026); CLI tools available for testing today

The user-facing wallet (Atoshi Mobile) does NOT expose privacy operations in v1. The SDK is available for early-adopter developers and integrators.

## Related modules

- **`x/evm` (L1)** — bridge deposits go through here before reaching L2.
- **`x/energy` (L1)** — backs the energy used by `EnergySettlement`.
- **Bridge** — moves ATOS L1 ↔ L2 wATOS, which is the pool's primary asset.

## Further reading

- [Threat model](../privacy/01-threat-model.md) — what we protect against and what we don't
- [Shield pool deep-dive](../privacy/02-shield-pool.md) — Merkle tree implementation, nullifier set, amount buckets
- [ZK circuits](../privacy/03-zk-circuits.md) — circom source walkthrough, trusted setup details
- [Energy settlement](../privacy/04-energy-settlement.md) — meta-tx flow, relayer economics

---

*Last reviewed: 2026-05-21*
