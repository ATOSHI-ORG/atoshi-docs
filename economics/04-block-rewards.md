# Block rewards and validator income

This chapter is the validator-economics companion to
[`x/tokenomics`](../modules/02-tokenomics.md) and [Release schedule](./02-release-schedule.md).
It walks through how every aatos that a validator earns flows through
the chain, from block production to claimable balance.

## The two income channels

A validator's total income decomposes into:

```
total_income = immediate_share + locked_share_claimable_so_far + tx_fee_share

where:
  immediate_share        = Σ over blocks proposed (20% of current_reward)
  locked_share_claimable = Σ over tier-release events × (this validator's locked share / total locked)
  tx_fee_share           = Σ over blocks × (fee_collector amount × validator's stake share)
```

The first two are issuance income (newly minted ATOS). The third is
transaction fee income (no minting). In a low-volume chain, **issuance
dominates** by orders of magnitude.

## Immediate share — the 20% block reward

When a validator proposes a block, `BeginBlocker` of `x/tokenomics` mints
`current_reward` ATOS. Of that:

- `immediate_reward_bps = 2000` (20%) flows through `x/distribution`'s
  standard validator reward path
- `locked_reward_bps = 8000` (80%) accumulates in the validator's
  `MinerLockedBalance.locked_amount`

The 20% is distributed by `x/distribution` like any other validator reward:

1. Validator's `commission_rate` is taken off the top
2. Remainder is split across delegators proportional to their stake
3. Delegators claim via `MsgWithdrawDelegatorReward`
4. The validator self-claims commission via `MsgWithdrawValidatorCommission`

This is **standard Cosmos behavior** — no Atoshi-specific logic. The
only Atoshi-specific detail is that the proposer is the SOLE recipient
of the 20% (other validators in the same block only earn through their
voting-power share of fees, not block reward).

### Per-block math

At default params with current testnet conditions (proxy for mainnet):

```
current_reward    = 19,819 ATOS / block
immediate_share   = 19,819 × 0.20 = 3,963.8 ATOS / block
locked_share      = 19,819 × 0.80 = 15,855.2 ATOS / block (to proposer's locked balance)
```

With 28,800 blocks/day (at 3-second blocks):

```
Total daily immediate distribution = 28,800 × 3,963.8 = ~114.2 M ATOS / day
Total daily locked accumulation    = 28,800 × 15,855.2 = ~456.6 M ATOS / day
```

These numbers spread across all validators in proportion to their share
of block production over time. With 4 validators of equal stake:

```
Per-validator daily immediate = 28.5 M ATOS / day
Per-validator daily locked    = 114.2 M ATOS / day (in locked_amount, NOT claimable yet)
```

At $0.15/ATOS that's $4.3 M / day immediate per validator. Numbers
this large reflect the early-phase issuance design; expect them to shrink
once the chain matures (via reduction in `initial_block_reward` or
adjustment to `halving_interval_blocks`).

## Locked share — the 80% that waits for tier release

The locked share doesn't go directly to the validator's account. It
accumulates in the chain-level `tokenomics_miner_pool` module account,
with per-validator attribution tracked in:

```
MinerLockedBalance {
  validator_address: string
  locked_amount:     string  // not yet claimable
  claimable_amount:  string  // unlocked by tier release, ready for MsgClaimMinerLockedReward
}
```

### How locked becomes claimable

Tier release events ([Release schedule](./02-release-schedule.md))
unlock a fraction of the miner pool:

```
At each release event:
  released_to_miner_pool = circulating_supply × release_percentage_bps / 10000 × miner_release_share_bps / 10000
                        = circulating_supply × 0.05 × 0.50
                        = 2.5% of circulating supply

For each validator v:
  validator_share = v.locked_amount / Σ all validators' locked_amount
  v.claimable_amount += released_to_miner_pool × validator_share
  v.locked_amount    -= released_to_miner_pool × validator_share
```

Important: validators who produce more blocks (over time) accumulate
larger `locked_amount`, so they get a larger SHARE of each release.
Block production rewards are roughly proportional to stake (because
CometBFT selects proposers by stake-weighted randomness), so locked
share is roughly proportional to stake — but with random variation
that smooths out over time.

### Validator claims locked

Once `claimable_amount > 0`, the validator's operator calls:

```
MsgClaimMinerLockedReward {
  validator_address: "atoshivaloper1..."
}
```

Signer must be the operator account. Transfers `claimable_amount` ATOS
from the miner pool module account to the validator's withdraw address.
Resets `claimable_amount` to zero; `locked_amount` continues accumulating
from future blocks.

This message is on the [energy subsidized whitelist](../modules/01-energy.md):
the validator pays zero gas to claim — they can claim immediately
even from a zero-balance account.

## Transaction fee share

Fees collected by both the energy ante-handler (Cosmos shortfall) and
the EVM ante-handler (gas) accumulate in the `fee_collector` module
account. The SDK's `x/distribution` sweeps it every block and splits
across active validators in proportion to their voting power.

At current low chain volume, this number is tiny:

```
Average tx fee = 100,000 gas × 0.0021 aatos/gas (Cosmos route) ≈ 0
Average tx fee = 21,000 gas × 1 gwei (EVM route)               = 0.000021 ATOS

Daily volume = 100 tx (estimate)
Daily fee revenue total = 100 × 0.000021 = 0.0021 ATOS
```

At $0.15/ATOS, that's **$0.000315 per day** in fees across the entire
validator set. Essentially noise. **Block reward dominates by 8+ orders
of magnitude.**

This will only change if chain volume rises to mainnet-Ethereum levels
(thousands of tx/s sustained), at which point fees might become
non-trivial. Until then, validators essentially earn from issuance only.

## End-to-end validator PnL (example)

A validator with 25% of total stake, running for one year, sustained
T1 price (1 release event every 30 days at 5% of circulating supply):

### Immediate income

```
Yearly blocks proposed = 25% × 28,800 × 365 = 2,628,000 blocks
Immediate / block = 3,963.8 ATOS
Yearly immediate = 2,628,000 × 3,963.8 = 10.4 B ATOS
```

(These numbers are clearly too large; they reflect the unadjusted block
reward formula. In practice mainnet will scale this down via parameter
tuning before launch.)

### Locked accumulation

```
Locked / block to this validator = 15,855.2 ATOS × (locked accumulated by them) / pool total
```

This compounds nonlinearly — each release reduces both their locked AND
the pool, but the rate of accumulation also depends on what fraction of
new blocks they propose. Easiest to model with simulation.

### Tier release claimable

```
T1 release event yields 2.5% of circulating to the miner pool
Validator's claim = 2.5% × circulating × (their locked / total locked)
For a 25% stake validator: roughly 25% of the 2.5% = 0.625% of circulating
```

So per tier-release event, this validator can claim ~0.625% of circulating
supply. At 2 B circulating, that's 12.5 M ATOS per release.

### Yearly summary (rough order of magnitude)

```
Immediate           ~10 B ATOS  (oversize, expect mainnet rescaling)
Locked claimed      ~150 M ATOS (12 releases × 12.5 M)
Fees                ~0
Total               ~10.15 B ATOS / year
                    (worst case, sustained T1 across all 12 months)
```

A 25%-stake validator earning 10 B ATOS in a year clearly indicates the
block-reward parameter needs to be retuned before mainnet — current
testnet numbers are placeholder values for testing the mechanism. The
mainnet `initial_block_reward` will be governance-set to something
reasonable (likely two orders of magnitude smaller) before launch.

## Validator economics in the bear case

Same validator, but T0 sustained (no tier releases):

```
Immediate           ~10 B ATOS
Locked accumulated  ~40 B ATOS (sits in the pool, unclaimable)
Fees                ~0
Total claimed       ~10 B ATOS / year
Unclaimed potential ~40 B ATOS (waits for tier resume)
```

The validator earns the immediate share regardless of tier. The 80%
locked share is contingent — they accumulate it but can only claim when
the chain's price tier sustains. This is the **alignment mechanism**:
validators are paid for chain success, not just for block production.

## Slashing and effects

Standard Cosmos slashing applies:

| Offense | Penalty |
|---|---|
| Double-sign | 5% of bonded stake (`slash_fraction_double_sign`) |
| Downtime (50%+ blocks missed in 100-block window) | 1% of bonded stake (`slash_fraction_downtime`) |

Slashing affects:
- **Bonded stake**: reduced
- **Self-bond and delegator stake**: proportionally reduced
- **Locked rewards (MinerLockedBalance)**: NOT slashed in current params

A validator that double-signs loses 5% of their bonded stake but keeps
their accumulated locked rewards. Whether to also slash locked rewards
is a future governance decision — pros (stronger alignment) vs cons
(don't punish validators who've stopped misbehaving but still have old
locked balances accumulated).

## Delegator vs validator perspective

Delegators get:
- Their share of the immediate 20% (after validator commission)
- Their share of tx fees (after commission)
- **Zero of the locked 80%** — that flows entirely to validators' MinerLockedBalance

If a delegator wants to participate in the locked share, they need to:
- Run their own validator, OR
- Find a validator with low commission AND large locked balance and trust they'll redistribute

The chain doesn't enforce locked-share distribution to delegators. This
is a deviation from standard Cosmos behavior. Reasoning: the 80% lock
is a validator-alignment mechanism, not a delegator reward. Delegators
already get yield via the immediate share.

This DOES change validator selection dynamics: delegators may prefer
high-commission validators with strong locked-share accumulation over
low-commission newcomers.

## Operational queries

```
GET /atoshi/tokenomics/v1/miner_locked_balance/{validator_address}
{
  "balance": {
    "validator_address": "atoshivaloper1...",
    "locked_amount":     "... aatos",      // not yet claimable
    "claimable_amount":  "... aatos"       // ready to MsgClaimMinerLockedReward
  }
}
```

Validators check this regularly to know when to claim.

```
GET /atoshi/tokenomics/v1/release_status
```

Returns chain-wide state including totals. Validators use it to see
recent tier-release activity (signals whether claimable balances likely
increased).

## Comparison with peer Cosmos chains

| Chain | Block reward source | Locked / vested ? | Tier-conditional ? |
|---|---|---|---|
| Cosmos Hub | x/mint (inflation) | No, instantly claimable | No |
| Osmosis | x/mint + epochs | Liquidity mining is locked | No |
| Sei | x/mint | No | No |
| Stride | x/mint | No (but small) | No |
| **Atoshi** | **x/tokenomics (fixed schedule)** | **Yes (80% locked per block)** | **Yes (tier-driven release)** |

The Atoshi design is more conservative than typical Cosmos
"distribute-everything-immediately" patterns. The intent is to align
validator and chain success over years, not over single blocks.

## Future tuning

Likely governance proposals before mainnet:

1. **Reduce `initial_block_reward`** — current value implies oversize
   issuance; expect 10-100× reduction.
2. **Adjust `halving_interval_blocks`** to match real block time (4-year
   epoch needs ~42 M blocks at 3-second blocks, not 1 M).
3. **Possibly add slashing of locked rewards** — strengthens alignment
   but is contentious.
4. **Consider opening up locked share to delegators** — improves
   delegator economics, possibly at cost of validator security alignment.

None of these are urgent. The current testnet parameters work for
mechanism testing; mainnet parameters will be set in a coordinated
governance event.

## Related

- [Tokenomics module](../modules/02-tokenomics.md) — mechanism + state spec
- [Release schedule](./02-release-schedule.md) — supply curve details
- [Supply overview](./01-supply.md) — three pools + sinks
- [Governance case studies](../modules/09-governance.md) — how to propose param changes

---

*Last reviewed: 2026-06-10*
