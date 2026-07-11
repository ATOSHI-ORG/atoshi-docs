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

Per-block reward is deterministic by block height. Default params
(source of truth: `x/tokenomics/types/helpers.go`, `DefaultParams`):

| Param | Value |
|---|---|
| `initial_block_reward` | 19,819 × 10^18 aatos (= 19,819 ATOS) |
| `halving_interval_blocks` | 25,228,800 blocks |
| Block time (design) | 5 seconds |
| Blocks per year | 6,307,200 (= 31,536,000 s ÷ 5 s) |
| Implied halving period | 25,228,800 ÷ 6,307,200 = **~4.0 years** |

> **Design vs. observed block time.** `halving_interval_blocks`,
> `price_check_epoch_blocks` (17,280 = "once a day"), and the "~4-year"
> and "30-day" figures throughout this chapter all assume the **5-second**
> design target. As of 2026-07, the live chains commit blocks faster —
> ~3.5 s on mainnet (`atoshi_88188-1`) and ~3.4 s on testnet
> (`atoshi_88288-1`) — so at the current rate a halving epoch elapses in
> ~2.8 years and the "daily" price-check epoch fires roughly every 16.8 h.
> The per-cycle **ATOS amounts are unaffected** (they are block-count
> based, so cycle 0 still emits ~500 B). Only wall-clock durations move.
> The plan is to bring `timeout_commit` back to the 5-second target so the
> block-count parameters map to their intended calendar semantics;
> alternatively governance can re-tune the block-count params to the
> observed rate. Numbers below use the 5-second design basis.

The block reward is drawn from the **Miner Pool** (1 trillion ATOS, 10% of
the fixed 10-trillion supply). Emission is Bitcoin-style: each ~4-year
cycle emits exactly half of the previous cycle, so the series is an
infinite geometric sum that converges to — but never exceeds — the
1-trillion pool cap.

### Reward per period

| Cycle | Years | Reward / block | Minted this cycle | Cumulative | % of Miner Pool |
|---|---|---|---|---|---|
| 0 | 0–4 | 19,819 | ~500 B ATOS | 500 B | 50.00% |
| 1 | 4–8 | 9,909.5 | ~250 B | 750 B | 75.00% |
| 2 | 8–12 | 4,954.75 | ~125 B | 875 B | 87.50% |
| 3 | 12–16 | 2,477.375 | ~62.5 B | 937.5 B | 93.75% |
| 4 | 16–20 | 1,238.6875 | ~31.25 B | 968.75 B | 96.88% |
| ... | ... | ... | ... | → 1 T | → 100% |

Cycle 0 mints `19,819 × 25,228,800 ≈ 499.96 B ATOS` — half the Miner Pool.
The cumulative series converges to the full **1 trillion ATOS** Miner Pool
without ever depleting it, so validators stay incentivised for decades
while new issuance trends toward zero.

![Bitcoin-style halving emission of the 1-trillion Miner Pool](../assets/whitepaper/halving-emission.png)

### Per-block split: 20% immediate + 80% locked

Of the 19,819 ATOS minted each block (`immediate_reward_bps = 2000`,
`locked_reward_bps = 8000`):

- **3,963.8 ATOS** (20%) → fee collector, then paid via `x/distribution` (less the 2% community tax) to validators + delegators — immediately spendable
- **15,855.2 ATOS** (80%) → validators' `MinerLockedBalance.locked_amount` by voting power — pooled until a conditional tier release fires; **not** paid to delegators

So daily, at 5-second blocks (= 17,280 blocks/day):

- Immediate distribution: 17,280 × 3,963.8 ≈ **68.5 M ATOS / day** (across all validators by stake share)
- Locked accumulation: 17,280 × 15,855.2 ≈ **274.0 M ATOS / day**

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

![Conditional token release with escalating price/volume tiers](../assets/whitepaper/conditional-release.png)

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

So if circulating supply at trigger time is 200 B ATOS (day-one float is
the 100 B Migration Pool plus accumulated immediate block rewards):

| Pool | Released this event |
|---|---|
| Miner | 5 B ATOS |
| Project treasury | 5 B ATOS |

Released miner ATOS goes into validators' `claimable_amount` proportional
to their `locked_amount` share. Project ATOS goes to the
`project_treasury_address`, claimable via `MsgClaimProjectTreasuryReward`.

### Worked example: sustained T1 strength

Suppose:

- Circulating at start of period: 200 B ATOS
- Price stays at 0.17 USD (above T1's 0.165) for 30 epochs
- Volume sustained above 165k USD/24h for 30 epochs

Day 30 (first release):
- 5% × 200 B = 10 B ATOS released
- 5 B to miner pool → distributed to validators
- 5 B to project pool → in treasury

Circulating after release: ~210 B ATOS (the released amount is now
spendable). The NEXT 5% would fire another 30 epochs later.

Second release (assuming streak sustained):
- 5% × 210 B = 10.5 B ATOS

And so on. Each release is 5% of the *current* circulating, not the
day-one float — so the absolute amount grows as cumulative issuance
compounds, always bounded by the remaining Miner + Project pools.

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

A one-shot supply addition. Total cap: `migration_pool_total =
100,000,000,000 ATOS` (100 billion, = 1% of the 10-trillion total supply).
This pool seeds day-one circulating supply and honours every ERC-20 /
in-app balance at the 1:100 mapping.

Holders of the legacy ETH ATOS token claim via Merkle proof against the
genesis-set `migration_merkle_root`. Claims are bounded by:

- Pool balance (can't claim more than the pool has)
- Per-claimer Merkle leaf (each claimer can only claim their leaf's amount, once)
- Time window (`migration_claim_end_time_unix`)

After the window closes, unclaimed amount sweeps into the project pool.

## Cumulative supply forecast

Split the curve into a deterministic part (price-independent) and an
endogenous part (demand-driven).

**Deterministic part** — Migration Pool + immediate block rewards, over
cycle 0 (years 0–4 on the 5-second design basis):

| Source | Cycle 0 contribution to circulating |
|---|---|
| Migration Pool (day-one float) | 100 B ATOS (mostly claimed in year 1) |
| Block reward minted (from Miner Pool) | ~500 B (half the pool) |
| — immediate 20% (circulates) | ~100 B (~25 B/year) |
| — locked 80% (circulates only via tier release) | ~400 B accrued, not yet circulating |

Absent any tier release, circulating at the end of cycle 0 ≈ 100 B
(migration) + 100 B (immediate) = **~200 B** — about 2% of the 10 T cap.

**Endogenous part** — the conditional tier engine unlocks 5% of *current*
circulating per sustained-tier event, split 50/50 Miner/Project:

| Scenario | Circulating trajectory |
|---|---|
| Bear (price never sustains T0) | stays near the deterministic path (~200 B by end of cycle 0); grows only via immediate block rewards |
| Bull (T1–T3 sustained) | each event adds 5%; circulating compounds from hundreds of billions toward trillions over years, drawing down the Miner (1 T) and Project (8.9 T) pools |

Total supply converges toward — but never exceeds — the **10 trillion**
cap. The block-reward path is deterministic; the tier-release path is
endogenous (demand-driven), which is the whole point of the model.

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
    bounded by 100 B, mostly closed by year 1
```

The chain provides every number via REST queries — no scraping needed.

## Sensitivity to block time

`halving_interval_blocks = 25,228,800` and `price_check_epoch_blocks =
17,280` are **block counts**, so their calendar meaning depends on the
live block time. At the 5-second design they mean "4 years" and "1 day".

As of 2026-07 the live chains are faster — **~3.5 s mainnet, ~3.4 s
testnet** — so at the current rate:

```
halving epoch  ≈ 25,228,800 × 3.5 s ≈ 2.8 years   (design intent: 4 years)
price epoch    ≈ 17,280 × 3.5 s     ≈ 16.8 hours  (design intent: 24 hours)
30-epoch window ≈ 21 days                          (design intent: 30 days)
```

Per-block and per-cycle **ATOS amounts are unaffected** (block-count
based); only wall-clock durations move. To restore the intended calendar
semantics, either bring `timeout_commit` back to the 5-second target (the
current plan), or re-tune the block-count params to the observed rate:

```
halving_interval_blocks = ⌊4 × 365.25 × 86400 / block_time_seconds⌋
                        = 25,228,800 at 5 s (current default)
                        ≈ 36,036,000 at 3.5 s
price_check_epoch_blocks = ⌊86400 / block_time_seconds⌋
                        = 17,280 at 5 s (current default)
                        ≈ 24,700 at 3.5 s
```

This is a pending decision (see [governance runbook](../reference/governance-runbook.md)).

## Related

- [Tokenomics module](../modules/02-tokenomics.md) — mechanism + state spec
- [Supply overview](./01-supply.md) — pools + sinks at a glance
- [Block rewards detail](./04-block-rewards.md) — validator-side accounting
- [Oracle module](../modules/03-oracle.md) — price feed that drives tier engine

---

*Last reviewed: 2026-07-12*
