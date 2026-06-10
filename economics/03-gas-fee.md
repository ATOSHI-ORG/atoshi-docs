# Gas and fee economics

This chapter ties together three otherwise-separate components into the
single "what does a transaction cost" picture:

- [`x/energy`](../modules/01-energy.md) — covers Cosmos transactions with a
  holding-based capacity model
- [`x/feemarket`](../modules/05-evm.md) — covers EVM transactions with
  EIP-1559 base fee + min-gas-price floor
- The shared bank pool that fees ultimately settle into

If you only read one economics chapter, read this one.

## Two parallel fee paths

A Cosmos `MsgSend` and an Ethereum `eth_sendRawTransaction` move the
exact same ATOS in `x/bank`, but they pay different fees through
different ante pipelines:

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

The two pipelines do NOT share gas pools, do NOT share pricing logic,
and do NOT share the energy meter. They share only the destination
(`fee_collector`) and the underlying ATOS denom.

## Cosmos transactions: energy + ATOS shortfall

Detailed in [`x/energy`](../modules/01-energy.md). Summary:

A user with eligible balance B has:
- `TxEnergyCapacity = floor(B / 30,000 ATOS) × 50,000` units
- Refilling linearly over `tx_energy_max_accrue_window` (24h default)

When a Cosmos tx with `gas_limit = G` arrives:

1. Drain `min(G, TxEnergyAccrued − DelegatedOut)` from own energy
2. If still short, drain from `DelegatedInUsable` (borrowed from delegators)
3. If still short after both, charge `shortfall_gas × params.insufficient_gas_price` in ATOS

`insufficient_gas_price` defaults to `0.0021 aatos/gas`. A 100,000-gas
transfer with zero energy costs `100,000 × 0.0021 = 210 aatos ≈
2.1 × 10⁻¹⁶ ATOS`. Even the worst-case Cosmos transfer is
essentially free.

If every message in the tx is on the subsidized whitelist, this entire
pipeline is bypassed and the tx is free regardless of energy state. See
[modules/01-energy.md §Subsidized whitelist](../modules/01-energy.md).

## EVM transactions: standard EIP-1559

The EVM path uses the ethermint `MonoDecorator` and the `x/feemarket`
module. Energy is intentionally bypassed. A typical EVM transaction
pays:

```
total_fee = gas_used × effective_gas_price

where:
  effective_gas_price = min(max_fee_per_gas,
                            base_fee + max_priority_fee_per_gas)
  base_fee adjusts per block per EIP-1559:
     base_fee_n+1 = base_fee_n × (1 + (gas_used_n - target_gas) / (target_gas × denominator))
```

Current `x/feemarket` parameters:

| Param | Value |
|---|---|
| `min_gas_price` | 10⁹ aatos/gas (1 gwei) — floor |
| `base_fee` (initial) | 1 gwei |
| `base_fee_change_denominator` | 8 (standard Ethereum) |
| `elasticity_multiplier` | 2 (block can be 2× target) |
| `min_gas_multiplier` | 0.5 (charge at least 50% of gas_limit) |
| `no_base_fee` | false (EIP-1559 enabled) |

A native ATOS transfer via EVM:

| Cost item | Value |
|---|---|
| gas_limit | 21,000 |
| base_fee | 1 gwei (when idle) |
| effective_fee | 21,000 × 1 gwei = 0.000021 ATOS |

An ERC20 transfer:

| Cost item | Value |
|---|---|
| gas_limit | ~65,000 |
| effective_fee | 65,000 × 1 gwei = 0.000065 ATOS |

These prices are orders of magnitude higher than the Cosmos path but
still negligible in absolute terms. The pricing matches what wallets
and SDKs expect from any EVM chain.

## Choosing the right path

| Scenario | Recommended path | Why |
|---|---|---|
| ATOS transfer between bech32 addresses, holder has ≥ 30k ATOS | Cosmos `MsgSend` | Zero ATOS fee (energy covers it) |
| ATOS transfer to a 0x... address; sender uses MetaMask | EVM `eth_sendRawTransaction` | UX continuity; cost is negligible |
| ERC20 / wrapped token transfer | EVM (only option) | Cosmos doesn't see ERC20-only assets |
| Contract call (DEX, lending, etc.) | EVM | Cosmos can't dispatch EVM contracts directly |
| Validator delegation, governance vote | Cosmos | Some Cosmos messages have no EVM equivalent |
| L1 → L2 bridge deposit | EVM (bridge contract is EVM) | Only path |

Wallets typically expose both paths via the same UX, choosing
automatically based on the recipient address format (bech32 → Cosmos,
0x... → EVM).

## Where fees go: validator distribution

Both paths route collected fees into the `fee_collector` module account.
At the end of each block, the SDK's `x/distribution` module sweeps that
balance and distributes it across active validators in proportion to
voting power, minus a community-pool tax (currently 2%).

Validators in turn share with their delegators per the commission rate
each validator advertised. Delegators claim with `MsgWithdrawDelegatorReward`.

Block-reward emission (covered in
[economics/04-block-rewards.md](./04-block-rewards.md)) is paid into the
same fee_collector flow, so validators don't need to distinguish "fee
income" from "issuance income" — both flow through one mechanism.

## Burn vs accumulation

Atoshi currently has **zero burn** on transaction fees. All collected
ATOS goes to validators and the community pool. There is no equivalent
of Ethereum's EIP-1559 base-fee burn.

Governance can introduce a burn fraction in the future via `MsgUpdateParams`
on `x/distribution` (specifically by routing a portion to a burner module
account). This would be a deliberate monetary policy choice with
implications:

- **Pro**: deflationary pressure on supply, especially during high
  transaction volume
- **Con**: reduces validator revenue at the same time, which is the
  opposite of what's wanted for security during high-traffic periods

The current "no burn" design assumes:
- Block reward issuance is the dominant supply effect; transaction fees
  are minor
- Validator revenue stability is more important than supply deflation

## Worked examples

### Example A: Holder with 60,000 ATOS sends 100 ATOS to a friend via Cosmos

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

### Example B: Same holder, same transfer, but via EVM

```
Tx: eth_sendRawTransaction, native transfer, gas_limit=21,000

MonoDecorator:
  fee = 21,000 × 1 gwei = 2.1 × 10⁻⁵ ATOS

Total user cost: 100 ATOS transferred + 2.1 × 10⁻⁵ ATOS fee
```

The EVM path is ~200,000× more expensive in this example. Still
negligible in absolute terms. The principle: if the recipient is a 0x
address and the wallet defaults to MetaMask, just use EVM.

### Example C: New user with 0 ATOS but 1,000 ATOS borrowed energy

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

The user can still send the transaction because they had borrowed energy
to cover half the gas, and the chain auto-charges the remainder. This
is the "relayer" pattern: a sponsor delegates energy to many users so
they can transact with sub-threshold balances.

## Fee economics at scale

Suppose:
- Average tx pays 100,000 gas at 1 gwei → 0.0001 ATOS / tx
- Chain handles 10 tx/s sustained → 864,000 tx/day
- Daily fee revenue: 86.4 ATOS

At a hypothetical $0.15 ATOS price, that's ~$13 daily fee revenue
across all validators. Tiny by Ethereum standards — fees alone don't
sustain validator economics at this volume.

This is why block rewards (covered in
[economics/04-block-rewards.md](./04-block-rewards.md)) are the
dominant validator income source. Fees become significant only if
transaction volume reaches Ethereum-mainnet levels (200+ tx/s).

## Comparison with neighbors

| Chain | Gas pricing | Fee burn | Sub-cent transfers? |
|---|---|---|---|
| Ethereum mainnet | EIP-1559 base fee + tip | Base fee burned | No (~$0.50–$10) |
| BNB Smart Chain | Flat gas price | None | Marginal (~$0.05) |
| Polygon PoS | EIP-1559 + tip | Base fee burned | Yes (~$0.001) |
| Polygon zkEVM | EIP-1559 + tip | Base fee burned | Yes (~$0.001) |
| **Atoshi L1 (Cosmos route)** | **Energy + shortfall floor** | **None** | **Yes** (typical 0) |
| **Atoshi L1 (EVM route)** | **EIP-1559 + 1 gwei floor** | **None** | **Yes** (~$0.000001) |

Atoshi's distinctive design is the energy path, not the EVM path. EVM
is intentionally "standard Ethereum gas economics" for compatibility.
The interesting question is: how many users discover the Cosmos route?

## Governance levers

Three parameters dominate. All are tunable via `MsgUpdateParams` proposals.

| Lever | Module | Effect of raising |
|---|---|---|
| `energy.tx_energy_holding_threshold` | x/energy | Higher bar for free transactions (currently 30,000 ATOS) |
| `energy.insufficient_gas_price` | x/energy | More expensive ATOS-shortfall fees (currently 0.0021 aatos/gas) |
| `feemarket.min_gas_price` | x/feemarket | Higher floor on EVM transactions (currently 1 gwei) |

The recent governance proposal raising `feemarket.min_gas_price` from 0
to 1 gwei is the canonical example. The pre-change behavior
(`min_gas_price = 0`) made wallets display "gas price = 0" because
`eth_gasPrice` literally returned `0x0`, breaking user expectations.

## Related

- [Energy module](../modules/01-energy.md) — full spec of the energy system
- [EVM module](../modules/05-evm.md) — EVM gas pipeline detail
- [Block rewards](./04-block-rewards.md) — where fee revenue meets issuance
- [Gov case studies](../modules/09-governance.md) — including the feemarket
  fee-floor proposal

---

*Last reviewed: 2026-06-10*
