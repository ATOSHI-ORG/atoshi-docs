# Burn and recycle mechanisms

ATOS supply is monotonically non-decreasing in v1. There is no
EIP-1559-style base-fee burn, no token-burning gov action, no
deflationary mechanism. This chapter documents the current state,
explains why, and outlines what burn paths could look like if
governance decides to add them later.

## Current state: zero burn

Track every aatos through the chain and you'll find one of two
destinations:

```
                       Where ATOS goes
              ┌──────────────────────────────┐
              │                              │
              │   Account balance            │
              │   (spendable by holder)      │
              │                              │
              │   Module account balance     │
              │   (held by:                  │
              │     - fee_collector          │
              │     - bonded_tokens_pool     │
              │     - not_bonded_tokens_pool │
              │     - distribution_module    │
              │     - tokenomics_*_pool      │
              │     - energy_locked          │
              │     - shield contract (L2)   │
              │     - bridge escrow (L1)     │
              │   )                          │
              │                              │
              └──────────────────────────────┘
                          │
                          └─── never destroyed
```

A spent transaction fee goes from sender's account into `fee_collector`,
then out to validators and the community pool. The total ATOS supply
doesn't change.

A burned-deposit governance veto burns the proposer's deposit — but
those go to the community pool, not destroyed. Even slashed validator
stake is moved (to the community pool or burned at the bank level
depending on params), but Atoshi's current `slash_fraction_*` go to
distribution.

## Why no burn (current rationale)

Three reasons:

### 1. Block reward is the dominant supply flow

Daily block reward emission ≈ 10⁹ ATOS-equivalent (at current testnet
params; mainnet will be lower). Daily tx fees ≈ 10⁻³ ATOS. Burning all
fees would barely register against issuance — symbolic deflation, real
inflation.

For Ethereum, EIP-1559 burn matters because gas fees are large
(hundreds of millions of dollars/year). For Atoshi, fees are tiny by
design (the energy system explicitly subsidizes them), so a fee burn
has no monetary policy effect.

### 2. Validator income shouldn't be diluted

Burning fees reduces validator revenue. Validators on Atoshi already
have unusual income mechanics (80% locked, contingent on tier release).
Adding a burn on top would worsen validator economics during low-tier
periods, exactly when they need stable income most.

### 3. Predictable supply curve > deflation theater

Atoshi's supply curve is already endogenous to demand (tier engine).
Adding burn introduces a second endogenous variable, making the supply
forecast harder. Investors prefer "explainable monotonic growth" to
"complex dynamics with occasional deflationary periods."

## What kinds of burn could be added later

Each of these is a future governance proposal, not a v1 feature. They
exist as design space, not as commitment.

### Option A: fixed-fraction fee burn (EIP-1559 style)

Modify `x/distribution` (or insert a hook between `fee_collector` and
validator payout) to burn a fixed fraction of incoming fees:

```
burn_fraction = 0.5         // governance-tunable
fee_to_burn   = collected_fees × burn_fraction
fee_to_validators = collected_fees × (1 - burn_fraction)
```

**Pros**: Standard pattern, well-understood. Aligns with Ethereum mental
model. Aware investors expect it.

**Cons**: Reduces validator revenue. At current low fee levels, effect on
supply is negligible. Possibly justifies itself only if Atoshi reaches
mainnet-Ethereum-level activity.

### Option B: dynamic burn (tied to tier release)

When tier engine releases supply, burn a fraction to offset:

```
tier_release = circulating × release_percentage_bps / 10000
burn         = tier_release × burn_offset_fraction
net_release  = tier_release - burn
```

**Pros**: Cap on net supply growth, even during T3 sustained periods.
Reduces analyst worry about "runaway tier-release dilution."

**Cons**: Diminishes the tier-release reward to validators. The whole
point of the tier engine is to reward sustained price strength —
burning offsets the reward.

### Option C: shielded-pool exit burn

When a privacy withdrawal claims from `Shield.sol` on L2, burn a small
percentage:

```
withdraw_amount × privacy_exit_fee_bps → burned (or sent to community pool)
```

**Pros**: Self-funding for privacy infrastructure. Could fund the
relayer service or future ZK proving costs. Doesn't affect Cosmos /
validator economics.

**Cons**: Reduces user economic incentive to use privacy. Privacy is
already opt-in; further taxing it slows adoption.

### Option D: slashing → burn (not distribute)

Standard Cosmos slashes go to the community pool; on Atoshi, they could
be burned outright. Currently params are configured to distribute.

```
ParamChange: slashing_destination = "burn"
```

**Pros**: Stronger deterrent for misbehavior (validators don't see
slashed stake recovered into community pool that they have governance
votes on). Symbolic deflation.

**Cons**: Slashing is rare (counted in dozens per year across all
chains). Effect on supply is negligible. Net practical change: minimal.

## Recycle: where "burned-ish" amounts already go

Even without literal burn, several mechanisms remove ATOS from
individual circulation by moving it to module accounts. These are
"recycling," not "burning":

| Flow | Source | Destination | Released back? |
|---|---|---|---|
| Validator commission | Block reward + fees | Validator's account | Already in circulation |
| Community pool | 2% of fees + slashed amounts | `distribution_community_pool` | Released via gov spend proposals |
| Energy delegation collateral | User's bank balance | `energy_locked` module | Returned at delegation expiry |
| Bridge L1 escrow | User's bank balance | bridge contract holdings | Released on L2-to-L1 withdrawal |
| Tokenomics locked miner pool | Block reward | `tokenomics_miner_pool` | Released via tier engine |
| Tokenomics locked project pool | Genesis | `tokenomics_project_pool` | Released via tier engine + gov treasury withdrawals |
| Tokenomics unclaimed migration | Genesis | `tokenomics_migration_pool` | Sweeps into project pool after deadline |

The `circulating_supply` query excludes these (see [supply.md](./01-supply.md)).
From a market-cap perspective, these amounts are effectively non-circulating
until released. **The mechanism that turns "non-circulating" into
"circulating" is gradual and gov-controlled.**

## Comparison: peer chains

| Chain | Burn mechanism | Active? | Annual burn (estimate) |
|---|---|---|---|
| Ethereum | EIP-1559 base fee | Active since 2021 | ~3 M ETH burned cumulatively |
| BNB Chain | Quarterly burn from exchange profits | Active | ~$1 B / year |
| Polygon | EIP-1559 burn | Active | Small, network usage dependent |
| Cosmos Hub | None (community pool only) | None | 0 |
| Osmosis | Some token sinks | Marginal | < 0.1% supply / year |
| **Atoshi v1** | **None** | **None** | **0** |

Atoshi's "no burn" choice puts it in the Cosmos lineage — most BFT
chains don't burn either. EVM chains (Ethereum, BNB, Polygon) tend
to burn because their fee revenue is large enough to matter.

If/when Atoshi reaches a transaction volume where fee revenue becomes
material (~10 USD/day across all validators), a burn proposal becomes
worth considering.

## Operational notes

For analysts modeling Atoshi supply, "burn" line items are zero. The
fully-diluted supply equals the cumulative emission given the chain's
parameters and observed tier history.

For wallets, no burn means no asymmetry between "what I receive" and
"what was sent" — transfers move ATOS quantity-preserving. Display
"sent" / "received" amounts as equal.

For validators, no burn means tx fee revenue maps 1:1 to claimable
amount. After commission + distribution math, every aatos of fees ends
up in someone's account.

## Future considerations

The next time a major param change touches monetary policy (probably
around mainnet launch or first major upgrade), this question will likely
re-open. The proposal will need to address:

- Which burn mechanism (A / B / C / D, or hybrid)?
- What `burn_fraction` or `burn_offset_fraction`?
- How to communicate the supply-curve change to existing holders?
- Does it require a coordinated chain upgrade (yes, if it touches
  `x/distribution` or `x/tokenomics`)?

The chain code IS prepared for this: the `fee_collector` flow has
intermediary steps where a burn hook can be inserted. Adding a burn
mechanism is a contained refactor, not a deep redesign.

But there's no urgency. The current zero-burn equilibrium is internally
consistent and matches the chain's design philosophy of stable validator
income + endogenous supply expansion.

## Related

- [Supply overview](./01-supply.md) — three pools + sinks
- [Block rewards](./04-block-rewards.md) — where issuance flows
- [Release schedule](./02-release-schedule.md) — tier engine for supply expansion
- [Gas and fee economics](./03-gas-fee.md) — where fees come from and where they go

---

*Last reviewed: 2026-06-10*
