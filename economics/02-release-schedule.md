# Release schedule and halving curves

This chapter is the numerical companion to [Tokenomics module](../modules/02-tokenomics.md).
It enumerates every supply unlock mechanism with concrete numbers and
worked examples, so analysts can build their own models without reading
keeper code.

## The two release engines

ATOS unlocks through two parallel processes:

1. **Block reward (continuous)** — per-block mint that halves every ~4 years
2. **Tier release (conditional)** — chunks unlocked from miner + project pools
   when price/volume tier conditions sustain

Genesis circulating supply is fixed. Everything beyond that comes from
these two engines.

## Block reward: halving table

Per-block reward is deterministic by block height. Default params:

| Param | Value |
|---|---|
| `initial_block_reward` | 19,819 × 10^18 aatos (= 19,819 ATOS) |
| `halving_interval_blocks` | 1,051,200 blocks |
| Block time | ~3 seconds (testnet observed; mainnet expected similar) |
| Implied period | ~36 days per halving epoch at 3 seconds/block |

Wait — `halving_interval_blocks = 1,051,200` ≠ 4 years at 3 seconds.
At 3-second block time, 1,051,200 × 3 = 3,153,600 seconds = **36.5 days**,
not 4 years. The original design assumed 5-second blocks (the historical
Cosmos default), which would give ~60 days.

This is governance-tunable. If mainnet block time stabilizes near 3 seconds,
`halving_interval_blocks` should be re-tuned (proposed value: 42,048,000
for a 4-year halving at 3-second blocks). Below we assume the original
4-year intent for compatibility with prior public materials.

### Reward per period (assuming 4-year halving epoch)

| Period | Years | Reward / block | Total ATOS minted (this period) |
|---|---|---|---|
| 0 | 0–4 | 19,819 | ~2,083,000,000 ATOS |
| 1 | 4–8 | 9,909.5 | ~1,041,500,000 |
| 2 | 8–12 | 4,954.75 | ~520,750,000 |
| 3 | 12–16 | 2,477.375 | ~260,375,000 |
| 4 | 16–20 | 1,238.6875 | ~130,187,500 |
| ... | ... | ... | ... |

Cumulative emission converges to ~4.16 B ATOS (= initial × 2). Combined
with genesis circulating (~1.46 B), this caps total emission via block
reward at ~5.62 B, well below the 10 B hard cap. The remaining ~4.4 B
flows through tier releases.

### Per-block split: 20% immediate + 80% locked

Of the 19,819 ATOS minted each block:

- **3,963.8 ATOS** (20%) → proposer's withdraw address through `x/distribution` — immediately spendable
- **15,855.2 ATOS** (80%) → proposer's `MinerLockedBalance.locked_amount` — pooled until tier release

So daily, at 3-second blocks (= 28,800 blocks):

- Immediate distribution: 28,800 × 3,963.8 = ~114.2 M ATOS / day (across all validators by stake share)
- Locked accumulation: 28,800 × 15,855.2 = ~456.6 M ATOS / day

(These numbers assume mainnet block time matches testnet. Real mainnet may differ.)

## Tier release: price/volume thresholds

Defined by `price_base` × `tier_multiplier^N` for tier N.

Default params:

```
price_base      = 0.15 USD       (T0 price floor)
volume_base     = 150,000 USD    (T0 volume floor)
tier_multiplier = 1.1
```

So:

| Tier | Price threshold | Volume threshold |
|---|---|---|
| T0 | 0.150 USD | 150,000 USD/24h |
| T1 | 0.165 USD | 165,000 USD/24h |
| T2 | 0.1815 USD | 181,500 USD/24h |
| T3 | 0.1997 USD | 199,650 USD/24h |

To be "at tier N", BOTH price AND volume from `x/oracle` must clear
tier-N's threshold simultaneously. The oracle's reported TWAP price and
24h volume are compared per epoch.

### Streak engine

Default `consecutive_days_required = 30`. Each `price_check_epoch_blocks`
(default 17,280 ≈ 1 day at 5-second blocks; recheck if mainnet uses
different block time), the EndBlocker:

```
new_tier = highest tier N where (twap >= threshold_N AND volume >= vol_threshold_N)

if new_tier == current_tier:
    consecutive_days += 1
elif new_tier > current_tier:
    current_tier = new_tier
    consecutive_days = 1
elif new_tier < current_tier:
    current_tier = new_tier
    consecutive_days = 0    # streak reset

if consecutive_days == consecutive_days_required:
    trigger_release()
    # consecutive_days stays at threshold; next release in another 30 days
```

The streak does NOT reset on release — once you're sustained at a tier,
releases happen every 30 days continuously until price/volume drop.

### Release amount per event

`release_percentage_bps = 500` (5%) of circulating supply.

Of that 5%:
- 50% from miner pool (50% × 5% = 2.5% of circulating)
- 50% from project pool

So if circulating supply at trigger time is 2 B ATOS:

| Pool | Released this event |
|---|---|
| Miner | 50 M ATOS |
| Project treasury | 50 M ATOS |

Released miner ATOS goes into validators' `claimable_amount` proportional
to their `locked_amount` share. Project ATOS goes to the
`project_treasury_address`, claimable via `MsgClaimProjectTreasuryReward`.

### Worked example: sustained T1 strength

Suppose:

- Circulating at start of period: 2 B ATOS
- Price stays at 0.17 USD (above T1's 0.165) for 30 days
- Volume sustained above 165k USD/24h for 30 days

Day 30 (first release):
- 5% × 2 B = 100 M ATOS released
- 50 M to miner pool → distributed to validators
- 50 M to project pool → in treasury

Circulating after release: ~2.05 B ATOS (the released amount is now
spendable). Tier release of NEXT 5% would happen on day 60.

Day 60 (second release, assuming streak sustained):
- 5% × 2.05 B = 102.5 M ATOS

And so on. Each release is 5% of the *current* circulating, not the
genesis circulating — so the absolute amount slowly grows as cumulative
issuance compounds.

### What if price drops mid-streak

Day 20: price drops to 0.155 USD (below T1's 0.165 but still above T0's 0.150).

```
new_tier = T0
current_tier was T1
→ current_tier = T0
→ consecutive_days = 0
```

The streak fully resets. Day 50 (after 30 more days at T0): first T0
release fires. The bear market literally pauses high-tier supply.

If the price recovers to T1 mid-T0-streak:

```
new_tier = T1
current_tier was T0 (with consecutive_days = N)
→ current_tier = T1
→ consecutive_days = 1
```

T1 streak restarts from 1. Bull-market regains tier but loses prior
T0 progress.

## Migration claims

A one-shot supply addition. Total cap: `migration_pool_total = 100,000,000 ATOS`
(1% of total supply, 10x the genesis figure given in supply.md — confirm
with deployment if you need exact number).

Holders of the legacy ETH ATOS token claim via Merkle proof against the
genesis-set `migration_merkle_root`. Claims are bounded by:

- Pool balance (can't claim more than the pool has)
- Per-claimer Merkle leaf (each claimer can only claim their leaf's amount, once)
- Time window (`migration_claim_end_time_unix`)

After the window closes, unclaimed amount sweeps into the project pool.

## Cumulative supply forecast

Stitching the pieces together. Assumes:
- Block reward: ~2.08 B in period 0 (years 0-4 if 4-year halving)
- Tier releases: 5%/event × variable number of events
- Migration: 100 M one-shot, mostly claimed in year 1

A best-case bullish scenario (sustained T2-T3 for years 1-3):

| Year | Block reward | Tier release | Cumulative emission | Cumulative supply |
|---|---|---|---|---|
| 0 (genesis) | — | — | 0 | 1.46 B |
| 1 | 521 M | 150 M (3 events × ~50 M) | 671 M | 2.13 B |
| 2 | 521 M | 200 M (4 events) | 1.39 B | 2.85 B |
| 3 | 521 M | 250 M (4-5 events) | 2.16 B | 3.62 B |
| 4 | 521 M | 200 M | 2.88 B | 4.34 B |
| 5–8 (halving 1) | 1.04 B | ~600 M | 4.52 B | 5.98 B |
| 9–12 (halving 2) | 521 M | ~400 M | 5.44 B | 6.90 B |
| ... | ... | ... | ... | ... |
| ∞ (asymptote) | ~4.16 B | bounded by pools | ~8.5 B | ~10.0 B |

A bear scenario (price stays at T0 for years 1-3):

| Year | Block reward | Tier release | Cumulative supply |
|---|---|---|---|
| 0–4 | 2.08 B | ~600 M (T0 only, 5% × ~24 events × ~50 M) | 4.14 B |

The block reward path is deterministic; the tier release path is endogenous.

## Reading the current state

Three queries:

```
GET /atoshi/tokenomics/v1/release_status
{
  "state": {
    "current_tier":              "1",
    "consecutive_days":          "15",
    "last_check_block":          "...",
    "total_miner_released":      "...",   // cumulative ATOS unlocked from miner pool
    "total_project_released":    "...",
    "total_immediate_distributed": "...", // cumulative 20% share paid to proposers
    "total_miner_locked":        "..."   // cumulative 80% share accumulated
  }
}
```

```
GET /atoshi/tokenomics/v1/circulating_supply
{ "circulating_supply": "..." }
```

```
GET /atoshi/tokenomics/v1/block_reward
{ "current_reward": "...", "period": "0" }
```

Together they tell you: where we are in the halving epoch, what tier we're
at, how many days into the streak, how much has been released so far,
how much is in flight.

## Practical implications

### For validators

Block production = 20% immediate revenue + accumulating 80% locked share.
Locked share only liquefies on tier release. **Sustained price/volume
strength is in your direct interest** — without it, your share of locked
rewards never claims.

If a validator misbehaves (slashing) or stops producing (downtime), they
keep already-accumulated locked but stop accumulating new. Locked balance
isn't slashed unless explicitly enabled.

### For long-term holders

Holding through bear markets is rewarded: when tier resumes, the supply
release accelerates issuance into validator economics, which reflects in
energy capacity (via released ATOS going into circulation, raising your
relative threshold count) and validator commission.

### For analysts

Build your supply model with:

```
supply(t) = genesis +
            block_reward_cumulative(t) +
            tier_release_cumulative(t) +
            migration_claims_to_date(t)

block_reward_cumulative(t):
    deterministic given (block_time, halving_interval_blocks, initial_block_reward)

tier_release_cumulative(t):
    requires modeling the price/volume scenario
    Σ over events: 5% × circulating_at_event_time

migration_claims_to_date(t):
    bounded by 100 M, mostly closed by year 1
```

The chain provides every number via REST queries — no scraping needed.

## Sensitivity to block time

The block-time assumption matters. If real mainnet block time is 3 seconds
but `halving_interval_blocks` stays at 1,051,200, halvings come every 36
days instead of 4 years. Per-block reward stays the same, but year-0
emission is `28,800 × 24 × 19,819 × 36.5 / 365` × 12 ≈ much smaller than
intended.

If governance wants to maintain the "4-year halving" semantics, the
parameter must be adjusted:

```
halving_interval_blocks = ⌊4 × 365.25 × 86400 / block_time_seconds⌋
                        = 42,048,000 if block_time = 3 sec
                        = 25,228,800 if block_time = 5 sec  (original)
```

This is a pending governance decision (see [governance case studies](../modules/09-governance.md)).
The chain's tokenomics.params.halving_interval_blocks default is the
5-second-block assumption; if testnet sustains 3-second blocks, expect a
proposal to retune.

## Related

- [Tokenomics module](../modules/02-tokenomics.md) — mechanism + state spec
- [Supply overview](./01-supply.md) — pools + sinks at a glance
- [Block rewards detail](./04-block-rewards.md) — validator-side accounting
- [Oracle module](../modules/03-oracle.md) — price feed that drives tier engine

---

*Last reviewed: 2026-06-10*
