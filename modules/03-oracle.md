# `x/oracle` — ATOS/USD price feed

The oracle module provides an authoritative on-chain ATOS/USD price and 24h trading volume, sourced from a configurable set of allow-listed feeders. It is the input that drives `x/tokenomics`'s tier engine: without a fresh oracle reading, tier evolution pauses.

## Mental model

A small number of trusted "feeders" (foundation-operated and partner-operated) post price reports to the chain via `MsgReportPrice`. The module:

1. Validates the report (feeder allow-listed, deviation within bounds, source labeled).
2. Stores it in a bounded ring buffer (`PriceHistory`).
3. Exposes the latest report and a TWAP computed over `twap_lookback_seconds` (default 24h).

The chain does NOT run an internal AMM, does not aggregate from multiple sources internally, and does not rank feeders. The trust assumption is **allow-listed feeders are honest**; consumers (most importantly `x/tokenomics`) read the chain's stored price as ground truth.

Feeders are typically deployed off-chain bots that pull from external venues (Uniswap V3, project DEX, CEX APIs) and submit at a fixed cadence. The chain enforces `max_price_deviation_bps` between consecutive reports to prevent runaway moves.

## State

### `PriceData`

```
message PriceData {
  string price       = 1;   // USD, 18-decimal LegacyDec
  string volume_24h  = 2;   // USD, 18-decimal LegacyDec
  int64  timestamp   = 3;   // unix seconds, block time
  string feeder      = 4;   // bech32 of the reporting feeder
  string source      = 5;   // free-form label ("uniswap_v3", "static", etc.)
}
```

The ring buffer holds up to `MaxHistoryEntries` (compiled-in constant, default 1000) most recent reports.

### Current price

The "current" price exposed at `/atoshi/oracle/v1/price` is simply the most recent `PriceData` in the buffer. There is no on-chain consensus / median across multiple feeders within a block; the latest write wins.

If two feeders report in the same block, the order is determined by tx ordering — there is no deterministic preference. This is acceptable because all allow-listed feeders are expected to converge on the same source data.

## TWAP computation

TWAP at lookback `T` is computed as:

```
TWAP(T) = Σ (price_i × dt_i) / Σ dt_i
```

where the sum is over reports in `[now - T, now]`, and `dt_i` is the gap between report `i` and report `i+1` (or to `now` for the last). The formula is a standard time-weighted mean.

If no reports exist within the lookback window, the query returns the latest report unchanged with a flag (not yet implemented as a separate field but visible from `timestamp`). Consumers should check `now - timestamp > max_price_age_seconds` to detect stale data.

## Messages

### `MsgReportPrice`

```
feeder     string   // signer; must be in params.allowed_feeders
price      string   // 18-dec LegacyDec
volume_24h string   // 18-dec LegacyDec
source     string   // label
```

Validation:

1. `feeder` is in `params.allowed_feeders`.
2. `price > 0` and `volume_24h >= 0`.
3. Deviation from the latest stored price (if any) does not exceed `max_price_deviation_bps`. A deviation is `|new - prev| / prev × 10000`.
4. Source string is non-empty (advisory; no enforcement of values).

On success, appends a new `PriceData` to the ring buffer with the block timestamp. Emits `price_reported`.

Whitelisted by `x/energy` → free.

### `MsgUpdateParams`

Standard gov-only. Used to add/remove feeders, adjust deviation caps, or change the lookback window.

## Queries

| Endpoint | Returns |
|---|---|
| `GET /atoshi/oracle/v1/price` | Latest `PriceData` |
| `GET /atoshi/oracle/v1/twap?lookback_seconds=N` | `{twap_price, avg_volume}` |
| `GET /atoshi/oracle/v1/price_history?limit=N` | Up to N most recent reports |
| `GET /atoshi/oracle/v1/params` | Module params |

The `lookback_seconds` parameter is mandatory for TWAP and is independent of `params.twap_lookback_seconds` (which is just the default the tokenomics module uses internally).

## Parameters

| Name | Default | Notes |
|---|---|---|
| `allowed_feeders` | `[atoshi18d4ade...]` (foundation) at genesis | Add via governance |
| `max_price_age_seconds` | 3600 (1h) | Consumers' staleness threshold |
| `min_valid_reports` | 1 | Reserved for future multi-feeder consensus |
| `denom` | `aatos` | Token tracked |
| `twap_lookback_seconds` | 86400 (24h) | Default lookback hint |
| `max_price_deviation_bps` | 5000 (50%) | Per-report movement cap |

`max_price_deviation_bps` is intentionally loose (50%) because:

- During genesis bootstrap, the price may move sharply.
- A tighter cap (e.g. 5%) would let a stale or malicious feeder block legitimate moves by squatting on a stale price.

In production, governance can tighten this once price discovery stabilizes.

## Events

| Event | Attributes |
|---|---|
| `price_reported` | `feeder`, `price`, `volume_24h`, `source`, `timestamp` |
| `price_rejected` | `feeder`, `reason` (e.g. `deviation_too_high`) |

## Edge cases

### Multiple feeders, conflicting reports

The chain stores all of them. The latest one is "the" current price. TWAP averages across all. There is no rejection of "outlier" reports beyond the per-report deviation check — the system trusts the allow-list. If governance adds a malicious feeder, it can drift the price; remediation is removing the feeder.

### Feeder downtime

If all feeders go silent for longer than `max_price_age_seconds`, downstream consumers (especially `x/tokenomics` tier engine) treat the price as stale and pause their logic. The chain itself does not panic or halt — it just operates without fresh oracle data.

### Cross-source comparability

Different feeders may report from different venues (Uniswap V3 vs. centralized exchanges). Source labels (`uniswap_v3`, `gate`, etc.) are advisory; the chain does not differentiate them. Operationally, allow-listed feeders are expected to coordinate on which venue is canonical.

### TWAP behavior with sparse data

If only one report exists in the lookback window, TWAP equals that report's price (single sample, no time-weighting useful). If reports are clustered at the start of the window, the TWAP is biased toward old data — TWAP is not a perfect measure when feeder cadence is irregular. Feeders should aim for evenly spaced submissions (every 10–60 minutes is typical).

## Operational notes

- **Feeder bot**: a small Python/Go program that polls an external venue and submits `MsgReportPrice` once per minute. Reference implementation lives in the `atoshi-feeder` companion repo.
- **Gas cost**: the message is on the energy whitelist, so cost is zero. Feeders need any positive ATOS balance to satisfy the standard tx requirement (signature, sequence), but no fee is charged.
- **Multi-feeder operation**: foundation-operated, partner-operated, and (eventually) community-operated feeders submit independently. The chain does not coordinate them; off-chain communication can be used to align cadence.

## Related modules

- **`x/tokenomics`** — primary consumer. Tier engine pauses without fresh oracle data.
- **`x/energy`** — `MsgReportPrice` is subsidized, allowing feeders to operate without holding ATOS for gas.

---

*Last reviewed: 2026-05-21*
