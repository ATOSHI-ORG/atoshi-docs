# Supply and circulation

This chapter explains where ATOS comes from, where it goes, and why the curve looks the way it does. The numerical schedule (block reward amounts, halving timing) is in [Release schedule](./02-release-schedule.md); this chapter is the conceptual layer.

## Top-line numbers

| Quantity | Amount | Source |
|---|---|---|
| **Maximum supply** | 10,000,000,000 ATOS (10 B) | Hard-coded in `x/tokenomics` |
| **Genesis circulating** | 1,459,982,490.2 ATOS (~14.6%) | Immediate distribution at genesis |
| **Locked at genesis** | 5,839,929,960.8 ATOS (~58.4%) | Miner + project pools, released over time |
| **Reserved for migration** | ~2,700,087,549 ATOS (~27%) | Old-chain holders can claim |
| **Atomic unit** | 1 ATOS = 10^18 aatos | EVM-style precision |

The 20% / 80% genesis split (immediate vs locked) reflects a "ship working software" philosophy: enough ATOS in circulation at day one to bootstrap an active market, but most of the supply released against ongoing contribution (mining) and adoption (price-tier-driven releases).

## Three supply taps

ATOS supply enters circulation through exactly three mechanisms:

### 1. Block reward (continuous, validator-paid)

Every block, the `BeginBlocker` of `x/tokenomics` mints `current_reward` aatos and credits it to the proposer. Of this amount:

- **20% is immediately distributed** (sent to the proposer's withdraw address and counted against `total_immediate_distributed`).
- **80% is locked** in the miner reward pool and released through the tier-driven schedule (see #3).

The reward halves on a schedule. Initial reward at genesis: 19,819 ATOS / block. The halving period is 1,051,200 blocks (~4 years at 5-second blocks). The series 19,819 → 9,909.5 → 4,954.75 → ... continues until the per-block reward rounds to zero.

### 2. Migration claims (one-shot per holder)

Users who held the predecessor token can submit a Merkle proof to `MsgClaimMigrationTokens` and receive ATOS to a freshly derived address. Claims are bounded by the migration allocation (`reserved_for_migration` above). Claims are recorded in `total_migration_claimed`; once the allocation is fully claimed, further proofs fail. Unclaimed migration ATOS at the cutoff date sweep into the project treasury.

### 3. Tier-driven unlock (oracle-priced)

This is the novel one. The chain tracks four tiers `T0 < T1 < T2 < T3` defined by ATOS/USD price bands (configurable). The oracle module continuously reports a TWAP price; the tokenomics module reads that price each block and checks which tier we're in.

For each tier, the schedule specifies:
- An amount of ATOS to release from the locked pool to the **miner pool** per "tier event."
- An amount to release to the **project treasury** per tier event.
- A "consecutive days required" trigger — staying at or above the tier for this many days produces one event.

When the price *drops* below a tier for one full day, the consecutive counter resets. This makes the supply curve responsive to demand: sustained price strength accelerates issuance into the miner economy; sustained weakness pauses it. (Detailed tier table and event amounts in [Release schedule](./02-release-schedule.md).)

Released miner-pool ATOS is then distributed through validator claim messages (`MsgClaimMinerLockedReward`); project-pool ATOS is held in the treasury and disbursed by governance (`MsgClaimProjectTreasuryReward`).

## Where supply does NOT come from

There is no continuous inflation outside of block reward. The chain does not mint to fund the treasury through inflation hooks (unlike Cosmos Hub's standard `x/mint`). Specifically:

- `x/staking` does NOT mint inflation rewards. Validator income comes from block reward (#1) and tx fees only.
- The treasury does NOT mint. It receives tier-event releases (#3) and a fraction of project-treasury rewards.
- The bridge does NOT mint at L1. wATOS on L2 is created 1:1 against locked L1 ATOS in the bridge contract.

This means **circulating supply at any time = genesis circulating + block rewards distributed so far + migration claims + tier-unlocked claims**. There are no hidden taps.

## Where supply goes (sinks)

ATOS leaves circulation through:

### Burn (rare)

The chain currently does NOT have an EIP-1559-style base-fee burn. Shortfall fees from the energy ante-handler go to the standard `fee_collector` module account, not burned. Future governance may introduce a burn fraction; today the supply curve is monotonically non-decreasing.

### Locked (counted as "not circulating" but on-chain)

- **Validator self-bond** (`x/staking`) — visible in bank but bonded.
- **Energy delegation collateral** (`x/energy/energy_locked` module account) — held during active delegations.
- **Bridge locked** (L1 bridge contract holdings) — backing wATOS on L2.

These are all visible on-chain via standard module-account balance queries; the `circulating_supply` query subtracts them.

## The circulating-supply query

```
GET /atoshi/tokenomics/v1/circulating_supply
```

Returns the aatos amount that meets the chain's definition of "circulating" — total supply minus locked vesting, minus the unreleased portion of miner + project pools, minus migration unclaimed. This is the value to use for market-cap dashboards and exchange listings.

The exact subtraction is:

```
circulating = total_minted
            - total_miner_locked
            - total_project_locked
            - unclaimed_migration
            - bridge_locked
            - energy_delegation_locked
```

Each component is read directly from its source-of-truth keeper, not approximated.

## Why this design

We make several specific bets in this supply model:

1. **Tier-driven release ties issuance to demand.** A chain that issues a fixed schedule into a thin market depresses price. Atoshi releases more supply when the market is hot (price-tier sustained) and pauses when the market cools. This is the inverse of the typical "halving when price would benefit from it" pattern.

2. **80% locked at genesis aligns with PoS security.** Bonded stake earns block reward (the immediate 20%) and gradually unlocks miner ATOS over years. Validators are incentivized to operate the chain consistently to harvest the locked stream.

3. **Migration claims reward early holders.** ATOS reserved for migration is non-trivial. A direct claim path (no centralized airdrop) ensures the prior community ports forward.

4. **No inflation hook means clean accounting.** Anyone can compute the supply curve given the genesis state and observed tier events; nothing is opaque.

The model has tradeoffs: in particular, the supply curve is harder to forecast than a fixed schedule. The chain provides the `release_status` and `block_reward` queries so participants can always see the current state.

## Reading the supply curve

Three queries cover the practical needs:

| Query | Returns |
|---|---|
| `GET /atoshi/tokenomics/v1/release_status` | Current tier, consecutive days at tier, totals for miner / project / immediate / locked |
| `GET /atoshi/tokenomics/v1/circulating_supply` | One number, aatos |
| `GET /atoshi/tokenomics/v1/block_reward` | Per-block mint at current halving period |

Combined with the oracle price feed, these are enough to reconstruct the full supply state and project near-term issuance.

---

*Last reviewed: 2026-05-21*
