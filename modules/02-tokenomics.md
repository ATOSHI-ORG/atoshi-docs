# `x/tokenomics` — supply emission, halving, tier-driven release

This module governs how ATOS enters circulation. It owns the block-reward minting hook, the locked miner pool, the project treasury pool, the migration pool, and the price-tier release machine that decides when locked ATOS unlocks.

## Mental model

There are three reservoirs of ATOS at genesis:

```
┌─────────────────────────────────────────────────────────────┐
│  Total supply (10 B ATOS, hard cap)                          │
│                                                              │
│  ┌──────────────┬──────────────────┬────────────────────┐   │
│  │ Migration    │ Miner pool       │ Project pool       │   │
│  │ pool (1%)    │ (10%)            │ (89%)              │   │
│  └──────────────┴──────────────────┴────────────────────┘   │
│         │              │                     │               │
│         ▼              ▼                     ▼               │
│   Old-chain        Block reward         Treasury releases    │
│   holders         (proposer mint)       (gov-controlled)     │
│   claim by         + locked              + tier-driven       │
│   Merkle proof     unlock streak         unlock streak       │
└─────────────────────────────────────────────────────────────┘
```

Three paths move ATOS from a reservoir into circulation:

1. **Block reward** — every block, mint `current_reward` aatos. 20% goes immediately to the proposer through `x/distribution`; 80% accumulates in a per-validator locked balance that the validator can later claim *but only after the tier-streak unlocks it*.
2. **Migration claim** — pre-mine token holders submit `MsgClaimMigrationTokens` with a Merkle proof. Verified once, paid once.
3. **Tier release** — the EndBlocker watches the on-chain price (from `x/oracle`). If the price has stayed at or above tier-N's threshold for `consecutive_days_required` epochs, it releases `release_percentage_bps` of the relevant pool. A drop below tier resets the streak.

The novelty is #3: supply is endogenous to market price.

## State

### `ReleaseState`

One global record. Tracks where we are in the tier-release machine.

```
message ReleaseState {
  uint32 current_tier              = 1;   // T0..T3
  uint32 consecutive_days          = 2;   // streak at or above current_tier
  int64  last_check_block          = 3;
  int64  last_check_time_unix      = 4;
  string total_miner_released      = 5;
  string total_project_released    = 6;
  string total_immediate_distributed = 7;
  string total_miner_locked        = 8;
}
```

### `BlockRewardState`

```
message BlockRewardState {
  string current_reward     = 1;
  uint64 period             = 2;   // 0 before first halving
  int64  last_halving_block = 3;
}
```

### `MinerLockedBalance`

One row per validator (operator address).

```
message MinerLockedBalance {
  string validator_address = 1;
  string locked_amount     = 2;   // aatos accumulated, not yet claimable
  string claimable_amount  = 3;   // aatos unlocked by tier release
}
```

The split between `locked_amount` and `claimable_amount` is important: validators accumulate locked rewards every block, but cannot claim them until the tier engine unlocks a fraction. Unlocks are proportional to each validator's share of total locked.

### Module accounts

- `tokenomics_miner_pool` — 80% locked share at the chain level before per-validator attribution
- `tokenomics_project_pool` — project treasury funds
- `tokenomics_migration_pool` — migration airdrop allocation

## Block reward emission

`BeginBlocker` mints the per-block reward and splits it.

### Halving

Halvings are deterministic by block height:

```
period         = floor((current_height - genesis_height) / halving_interval_blocks)
current_reward = initial_block_reward / 2^period
```

Default params: `initial_block_reward = 19,819 ATOS`, `halving_interval_blocks = 1,051,200` (~4 years at 5s blocks).

| Period | Reward per block |
|---|---|
| 0 (year 0–4) | 19,819 |
| 1 (year 4–8) | 9,909.5 |
| 2 (year 8–12) | 4,954.75 |
| 3 (year 12–16) | 2,477.375 |

Minting is capped by the 10 B total supply hard cap — once cumulative emission would exceed the cap, the chain caps it to the remainder.

### Split: 20% immediate / 80% locked

Per block:

- `immediate = current_reward × immediate_reward_bps / 10000` (default 20%) — sent to proposer through `x/distribution`. Immediately spendable.
- `locked = current_reward × locked_reward_bps / 10000` (default 80%) — added to the proposer's `MinerLockedBalance.locked_amount`. Not claimable yet.

The ratios are governance-tunable but must sum to 10000.

### Why 80% locked

Validators are paid eventually in proportion to their work, but the lockup creates two effects:

- **Skin in the game.** A validator who maliciously double-signs or goes offline loses access to a growing locked stash, not just the immediate share.
- **Coupling to tier mechanic.** The locked stash unlocks only when the chain's price tier sustains. So validators are incentivized for the chain's long-term price strength, not just short-term block production.

## Tier-driven release

The most distinctive piece of the module.

### Tier definitions

Four tiers indexed 0–3, parameterized by `price_base` and `tier_multiplier`:

```
threshold_T0 = price_base                          (e.g. $0.15)
threshold_T1 = price_base × tier_multiplier        (e.g. $0.165)
threshold_T2 = price_base × tier_multiplier^2      (e.g. $0.1815)
threshold_T3 = price_base × tier_multiplier^3      (e.g. $0.19965)
```

Similarly for `volume_base × tier_multiplier^n`. Both price AND volume must clear the tier's threshold.

### The streak engine

Every `price_check_epoch_blocks` (default ~24h worth of blocks), the EndBlocker reads the TWAP price and 24h volume from `x/oracle`, then:

1. Determine the *current achievable tier* (highest N where both thresholds satisfy).
2. Compare to `state.current_tier`:
   - **Equal**: `consecutive_days += 1`
   - **Higher**: `consecutive_days = 1`, `current_tier = new`
   - **Lower**: `consecutive_days = 0`, `current_tier = new`. Streak broken.
3. **If `consecutive_days == consecutive_days_required`** (default 14): trigger a release.

### Release event

A trigger releases `release_percentage_bps` (default 100, i.e. 1%) of *circulating supply* from each pool:

- From miner pool: `miner_release_share_bps × release` (default 5000 = 50%) → distributed across all validators' `MinerLockedBalance.claimable_amount` proportional to their current `locked_amount`.
- From project pool: `project_release_share_bps × release` (default 5000) → sent to the `project_treasury_address`, claimable via `MsgClaimProjectTreasuryReward`.

After releasing, `consecutive_days` does NOT reset. The next release happens after another `consecutive_days_required` epochs (so on a 14-day streak: release on day 14, then day 28, then 42, ...).

### Why tier-based

A fixed emission schedule disconnects supply from market health. The threshold-and-streak design avoids dumping supply on a brief price spike — only *sustained* presence triggers a release. Bear markets pause releases entirely; the locked stash waits for renewed strength.

## Messages

### `MsgClaimMigrationTokens`

```
delegator    string    // signer's bech32
amount       string    // aatos to claim
merkle_proof bytes[]   // SHA-256 Merkle proof against params.migration_merkle_root
```

Validates the proof against the configured root. On success, transfers `amount` aatos from `tokenomics_migration_pool` to the signer. Records the claim to prevent double-claim. Refused after `migration_claim_end_time_unix`.

Whitelisted by `x/energy` → free.

### `MsgClaimMinerLockedReward`

```
validator_address string   // operator addr (atoshivaloper1...)
```

Signer must be the operator account. Transfers the validator's `claimable_amount` from the miner pool module account to the validator's withdraw address. Resets `claimable_amount` to zero; `locked_amount` continues accumulating.

Whitelisted by `x/energy` → free.

### `MsgClaimProjectTreasuryReward`

```
authority string   // must equal params.project_treasury_address
to        string   // destination address
amount    string   // aatos
```

Signer must equal the configured treasury address. Pays out `amount` aatos. Treasury address itself is governance-controlled — typically a multisig.

Whitelisted by `x/energy` → free.

### `MsgUpdateParams`

Standard gov-only update. Mutating `migration_merkle_root` is allowed but requires governance vote.

## Queries

| Endpoint | Returns |
|---|---|
| `GET /atoshi/tokenomics/v1/release_status` | Full `ReleaseState` |
| `GET /atoshi/tokenomics/v1/circulating_supply` | Single aatos amount |
| `GET /atoshi/tokenomics/v1/block_reward` | `{current_reward, period}` |
| `GET /atoshi/tokenomics/v1/miner_locked_balance/{validator_address}` | Per-validator balance |
| `GET /atoshi/tokenomics/v1/project_claimable` | Treasury balance available |
| `GET /atoshi/tokenomics/v1/params` | Module params |

## Parameters

| Name | Default | Notes |
|---|---|---|
| `miner_pool_total` | 1.0 B ATOS (10%) | Hard cap |
| `project_pool_total` | 8.9 B ATOS (89%) | Hard cap |
| `migration_pool_total` | 0.1 B ATOS (1%) | Hard cap |
| `immediate_reward_bps` | 2000 (20%) | Must sum to 10000 with locked |
| `locked_reward_bps` | 8000 (80%) | |
| `halving_interval_blocks` | 1,051,200 (~4yr at 5s) | |
| `initial_block_reward` | 19,819 ATOS | |
| `price_base` | 0.15 USD | T0 threshold |
| `volume_base` | 150,000 USD | T0 volume threshold |
| `tier_multiplier` | 1.1 | Per-tier factor |
| `consecutive_days_required` | 14 | Streak to unlock |
| `release_percentage_bps` | 100 (1%) | Of circulating supply per event |
| `miner_release_share_bps` | 5000 | Split of release into miner pool |
| `project_release_share_bps` | 5000 | Split into project pool |
| `project_treasury_address` | (governance-set) | Multisig recommended |
| `price_check_epoch_blocks` | ~17,280 (24h at 5s) | |
| `migration_merkle_root` | (genesis-set) | SHA-256 |
| `migration_claim_end_time_unix` | T+1 year | Deadline |

## Events

| Event | When | Attributes |
|---|---|---|
| `block_reward` | Every block | `proposer`, `immediate`, `locked`, `period` |
| `tier_advance` | EndBlocker, tier up | `from_tier`, `to_tier`, `price`, `volume` |
| `tier_retreat` | EndBlocker, tier down | same |
| `release_triggered` | Streak reached threshold | `tier`, `miner_amount`, `project_amount`, `consecutive_days` |
| `migration_claimed` | `MsgClaimMigrationTokens` | `claimant`, `amount` |
| `miner_reward_claimed` | `MsgClaimMinerLockedReward` | `validator`, `amount` |
| `project_reward_claimed` | `MsgClaimProjectTreasuryReward` | `recipient`, `amount` |

## Edge cases

### What if oracle has no fresh price?

If the latest oracle report is older than `oracle.params.max_price_age_seconds`, the tier check is skipped for that epoch. The streak counter does NOT advance but is not reset either. The chain effectively pauses tier evolution until the price feed resumes.

### What if a validator unbonds with locked balance?

The locked balance remains attributable to the operator address after unbonding. Their `claimable_amount` continues growing with tier releases proportional to their *frozen* `locked_amount`. They can still call `MsgClaimMinerLockedReward`. The `locked_amount` stops accumulating once they no longer produce blocks.

### What if the migration period ends with unclaimed ATOS?

`migration_pool_total - total_migration_claimed` sweeps into the project pool, expanding `project_pool_total`. Subsequent migration claims fail.

### Why no inflation hook?

Other Cosmos chains use `x/mint` to inflate continuously. Atoshi uses fixed block reward + tier release, so total supply is bounded and predictable. Validator rewards come from block production + tx fees, not inflation.

## Related modules

- **`x/oracle`** — provides TWAP price and volume. Tier engine cannot advance without fresh data.
- **`x/energy`** — all claim messages are subsidized so users with zero balance can claim.
- **`x/distribution`** — receives the immediate 20% block reward, applies validator commission.
- **`x/staking`** — validators producing blocks accumulate locked rewards via this module.

---

*Last reviewed: 2026-05-21*
