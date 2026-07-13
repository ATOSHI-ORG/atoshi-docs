# `x/energy` — gas via holding, not paying

The energy module is the centerpiece of Atoshi's user-experience design. Instead of charging every transaction in scarce native tokens, the module gives each account a *capacity* that refills over time as long as the holder maintains a minimum ATOS balance. Transactions consume from this capacity first; only the shortfall is paid in ATOS.

This chapter is the canonical specification of the module: data model, accrual mathematics, delegation, consumption order, ante-handler integration, parameters, edge cases. If you only read one module chapter, read this one.

The three questions this chapter answers in full — each has a dedicated worked-example section below:

- **How does energy accumulate?** → [Accrual](#accrual-how-energy-accumulates)
- **How do I transfer (delegate) energy to another account?** → [Delegation](#delegation-lending-energy-to-another-account)
- **Once transferred, can anyone else touch that energy?** → [Why delegated energy is exclusive](#why-delegated-energy-cannot-be-used-by-anyone-else)

## Mental model

Hold ATOS → earn the right to transact for free, at a rate proportional to your holding.

Specifically:

- Holding **30,000 ATOS** unlocks **220,000 TxEnergy capacity**, refilled linearly over **24 hours**.
- Holding **1,000,000 ATOS** additionally unlocks **800,000 DeployEnergy capacity**, refilled linearly over **10 days**.
- Both are `uint64` counters tracked per account.
- TxEnergy can be **delegated** (lent) to another account for a fixed duration; the corresponding ATOS is locked as collateral. DeployEnergy cannot be delegated.

`TxEnergy` covers ordinary transactions (transfers, contract calls, delegations). `DeployEnergy` covers contract-deployment messages (Wasm instantiate / store-code). EVM `MsgEthereumTx` — including contract-creation EVM txs — is **not** handled here; see [The EVM boundary](#the-evm-boundary).

## State

### `EnergyAccount`

One record per account. Definition mirrors `proto/atoshi/energy/v1/energy.proto`:

```protobuf
message EnergyAccount {
  string address               = 1;   // bech32, the account this row describes
  uint64 tx_energy_accrued     = 2;   // current spendable TxEnergy (before adding delegated_in_usable)
  uint64 deploy_energy_accrued = 3;   // current spendable DeployEnergy
  string last_balance_snapshot = 4;   // eligible_balance (bank − locked) at last settle
  int64  last_updated_time     = 5;   // unix second of last settle
  uint64 delegated_out         = 6;   // TxEnergy currently lent to others
  uint64 delegated_in_usable   = 7;   // TxEnergy received from others, still usable
  string locked_atos           = 8;   // total ATOS currently in collateral lock
}
```

The four spendable / locked totals (`tx_energy_accrued`, `delegated_out`, `delegated_in_usable`, `locked_atos`) are denormalized for fast read access. The authoritative ledger of outstanding delegations lives in the separate `EnergyDelegation` table.

Usable TxEnergy for an account is always:

```
usable = (tx_energy_accrued − delegated_out) + delegated_in_usable
```

### `EnergyDelegation`

One record per active delegation:

```protobuf
message EnergyDelegation {
  uint64 id          = 1;   // monotonic, unique across all delegations
  string delegator   = 2;
  string delegatee   = 3;
  uint64 amount      = 4;   // TxEnergy units lent (gas units)
  string locked_atos = 5;   // collateral frozen in the locked module account
  int64  start_time  = 6;
  int64  expires_at  = 7;   // auto-released by EndBlocker at this time
  uint64 used        = 8;   // accumulated consumption attributed to this record
}
```

Two secondary indices accelerate common queries:

- by `delegatee` — used by the consume path to attribute burn to specific inbound delegations
- by `expires_at` (8 bytes big-endian, then 8 bytes id) — EndBlocker expiry sweep, O(expired) not O(total)

### Module account

- `energy_locked_pool` — holds all ATOS posted as delegation collateral. The delegator's bank balance is physically reduced by the locked amount (the ATOS is *moved*, not merely earmarked), which is exactly why locked ATOS cannot be transferred by anyone while a delegation is live.

## Capacity formula

Capacity is a step function of the account's **eligible balance** — the bank balance minus any ATOS locked out as delegation collateral:

```
eligible_balance = bank.GetBalance(address) − locked_atos
threshold_blocks = floor(eligible_balance / TxEnergyHoldingThreshold)
tx_energy_capacity = threshold_blocks × TxEnergyPerThreshold      (saturating on overflow)
```

With default params (`TxEnergyHoldingThreshold = 30,000 ATOS`, `TxEnergyPerThreshold = 220,000`):

| Eligible balance | Capacity |
|---|---|
| 0 – 29,999 ATOS | 0 |
| 30,000 – 59,999 ATOS | 220,000 |
| 60,000 – 89,999 ATOS | 440,000 |
| 1,000,000 ATOS | 7,260,000 |

The step function is deliberate: it makes the holding requirement unambiguous and easy to communicate ("hold 30,000 ATOS to transact for free"). A linear function would let attackers earn near-zero capacity by holding dust.

DeployEnergy is different: `DeployEnergyCapacity` (default 800,000) is a **constant per-account ceiling**, not a per-block multiplier. Any account holding ≥ `DeployHoldingThreshold` (1,000,000 ATOS) refills toward the same 800,000 cap; larger holdings only refill *faster*, they do not raise the cap. See the accrual section for the rate.

## Accrual: how energy accumulates

This is the first of the three headline questions. Energy is a bucket that refills toward its capacity at a fixed rate; you spend from the bucket, and holding ATOS keeps refilling it.

### Lazy settlement — no per-block sweep

Energy refills **lazily**. There is no scheduled job that touches every account every block. The chain stores `last_updated_time` and `last_balance_snapshot` per account and recomputes accrued energy only when the account is next touched. This routine is called `Settle` (`keeper/settle.go`) and runs:

- in the ante-handler, before fee deduction (mutating)
- at the start of every keeper message handler (mutating)
- inside queries as `SimulateSettle` (non-mutating — so external callers see live numbers, never stale stored ones)

Because settlement is idempotent within a block, you can never observe a stale energy value from outside the chain: the `Account` query settles before returning.

### The refill equation

For TxEnergy, on each settle:

```
elapsed = now − last_updated_time
cap     = TxEnergyCapacity(last_balance_snapshot)
add     = (cap × elapsed) / TxEnergyMaxAccrueWindow          // multiply-before-divide
new_accrued = min(tx_energy_accrued + add, cap)
```

`TxEnergyMaxAccrueWindow` defaults to 86,400 (24h). The multiply-before-divide ordering matters: a naive `(cap / window) × elapsed` loses precision — `220000 / 86400 = 2` per second discards the fractional 0.546, and for smaller caps it truncates to zero outright. All intermediate products use saturating arithmetic so a governance re-tune can never wrap `uint64`.

**Worked example (TxEnergy).** Alice holds 30,000 ATOS, so `cap = 220,000`. Rate = 220,000 / 86,400 ≈ **2.546 energy/second**.

- Start empty. After 1 hour (3,600 s): `add = 220,000 × 3,600 / 86,400 = 9,166`. Accrued ≈ 9,166.
- After 12 hours: `add = 220,000 × 43,200 / 86,400 = 110,000`. Accrued = 110,000 (half the cap).
- After 24 hours: accrued hits the 220,000 cap and stops. Holding longer does not overfill.

**Worked example (DeployEnergy).** DeployEnergy rate is `(units × DeployEnergyCapacity × elapsed) / (DeployRecoverDays × 86,400)` where `units = floor(eligible_balance / DeployHoldingThreshold)`.

- Bob holds exactly 1,000,000 ATOS → `units = 1`. Refill 0→800,000 takes the full 10 days.
- Carol holds 3,000,000 ATOS → `units = 3`. She refills toward the *same* 800,000 cap at 3× the rate, i.e. ~3.3 days from empty. The cap does not rise; only the speed does.

### Why `last_balance_snapshot` and not the live balance

The snapshot is what the balance was *during the period being settled*. Using the live balance to compute accrual over a period when the balance had changed would misattribute energy.

The bank send hook (`SendRestriction` → `ApplyBalanceChange`) keeps the snapshot honest on every transfer of the base denom:

1. `Settle` first — closes out accrual against the OLD snapshot.
2. Write `last_balance_snapshot` = the projected NEW eligible balance.
3. Cap `tx_energy_accrued` **down** to the new (possibly lower) capacity.

That third step (cap-down) is critical: an account that sells most of its ATOS must not keep a giant prefilled buffer to move to a fresh wallet for free transactions. Without it, capacity would only ever ratchet upward.

**Cap-down floors at `delegated_out`** (audit Issue 6). If a holder has lent energy out and then sells stake so the new cap falls below `delegated_out`, the cap-down stops at `delegated_out` rather than collapsing it — the lent energy is a promise already minted into the delegatee's `delegated_in_usable`, and cutting below it would break the symmetric accounting that undelegation relies on. Only the holder's *own* unused portion is cut.

### The Evmos send-order footgun (fixed 2026-06-30)

`ApplyBalanceChange` runs *before* the bank store write, so it must be handed the **projected** post-send balance, not read it back. The Evmos cosmos-sdk fork invokes `SendRestriction` **after** `subUnlockedCoins(from)` and **before** `addCoins(to)` — the opposite of upstream. So inside the hook:

- `bank.GetBalance(from)` already reflects the subtraction → do **not** subtract `moved` again.
- `bank.GetBalance(to)` is still pre-addition → **add** `moved` to project the post-receive balance.

The pre-fix code double-subtracted on the sender side, silently shaving one `moved` worth of eligible balance off the snapshot on *every* transfer — for a 30,000-ATOS movement that is exactly one threshold block, i.e. capacity dropped by 50,000 energy per send (the per-threshold capacity in effect at the time of the incident). This was the production "5万 ATOS 凭空消失" (50k ATOS vanishing) fingerprint on testnet account `0x30F288…`. The fix is documented inline in `keeper/send_restriction.go`.

## Consumption: the ante-handler pipeline

`x/energy/ante.EnergyDeductDecorator` replaces the SDK's stock `DeductFeeDecorator`. For every Cosmos tx:

1. Decode the tx, resolve the fee payer (honoring `--fee-granter` / `x/feegrant`), collect message type URLs and the gas limit. Reject a zero gas limit.
2. If **every** message in the tx is on the subsidized whitelist → `{Free: true}`, skip fee deduction entirely.
3. Otherwise call `Consume(payer, gasLimit, isDeploy, msgTypeUrls)` (`keeper/consume.go`), which drains energy in the order below and returns `ConsumeResult{EnergyDeducted, OwnDeducted, DelegatedDeducted, DeployEnergyUsed, ShortfallGas, Free, DelegationConsumptions}`.
4. If `ShortfallGas == 0` → validate priority, skip the fee transfer.
5. Otherwise → charge the ATOS shortfall to the `fee_collector` module account.

The **PostHandler** (`ante/post.go`) refunds reserved-but-unused energy after the messages run (the energy equivalent of a gas refund). A **pending-reservation marker** written in ante state guards against the case where `runMsgs` fails: BaseApp then discards the msg-context state and skips the PostHandler, so the refund would be lost — the EndBlocker sweeps leftover markers and refunds failed-tx reservations (audit Issue-1 round2).

### Consume order

For a **regular** tx:

1. Own energy: `tx_energy_accrued − delegated_out`.
2. Then the inbound-delegation pool: `delegated_in_usable`.
3. Anything still uncovered → `ShortfallGas`.

For a **deploy** tx: `deploy_energy_accrued` first, then the two TxEnergy buckets above, then `ShortfallGas`.

`delegated_out` is subtracted in step 1 because that energy has already been minted into the delegatees' `delegated_in_usable`; not subtracting it would let the delegator double-spend energy it has already lent.

### Refund order is LIFO by origin

The PostHandler refunds `reserved − actually_used`. It splits the refund by **origin** (audit Q1 / Issue 11):

- The **delegated** portion is refunded first, walking `DelegationConsumptions` in reverse (last-consumed first) and rolling back each delegation record's `used` in lock-step. Rolling unused energy back onto the same time-bounded grant keeps it intact instead of silently converting it to permanent own-energy on the delegatee.
- If a delegation was undelegated or swept between `Consume` and the refund, that slice is **lost, not redirected** to the signer's own bucket — the delegator was already debited for it inside `releaseDelegation`, so no counterparty is owed (audit Issue 11).
- Any remaining refund credits the signer's own `tx_energy_accrued`, capped at current capacity.

### Subsidized whitelist (always-free messages)

Defined in `types.DefaultParams().SubsidizedMsgTypeUrls`:

| Type URL | Purpose |
|---|---|
| `/atoshi.tokenomics.v1.MsgClaimMigrationTokens` | Legacy chain migration claim |
| `/atoshi.tokenomics.v1.MsgClaimMinerLockedReward` | Validator unlock claim |
| `/atoshi.tokenomics.v1.MsgClaimProjectTreasuryReward` | Treasury claim |
| `/atoshi.oracle.v1.MsgReportPrice` | Allow-listed feeder price report |
| `/atoshi.energy.v1.MsgDelegateEnergy` | Energy delegation |
| `/atoshi.energy.v1.MsgUndelegateEnergy` | Energy revocation |

Subsidization is global — even an account with zero balance and zero energy can call these free of charge. For claim messages the anti-abuse control is application-level authorization; for delegate/undelegate it is the ATOS collateral lock.

Delegate/undelegate **must** be subsidized: the ante-handler greedily reserves up to `gas_limit` of the signer's energy as fee coverage, which would empty the delegator's accrued energy *before* the delegate handler runs and tries to lock part of it. A tx that is entirely subsidized reserves nothing, so the handler sees the full accrued balance.

### Shortfall pricing

When energy is short:

```
charge = min(user_offered_gas_price, InsufficientGasPrice) × shortfall_gas
```

`InsufficientGasPrice` defaults to `0.0021 aatos/gas`. Capping the user's offered price prevents fee griefing — a user cannot pay a huge non-energy fee to push others out of the mempool; the chain charges at most the policy price. If the user offered zero fee the chain still charges at the policy price, otherwise the shortfall could be dodged with `tx.fee = 0`.

**Priority** is derived from the ATOS actually paid (`chargeAtos`), not the declared fee (audit Q1 round2). Otherwise an energy whale could declare a huge fee, pay almost nothing because energy covered the gas, and still jump the queue with no real economic stake in the slot.

## Delegation: lending energy to another account

This is the second headline question. "Energy transfer" on Atoshi is a **delegation** (a lend), not a permanent handover: you keep ownership of the underlying ATOS, you lock it as collateral, and the borrowed energy is bound to one recipient for a fixed window, after which it returns to you automatically.

Use cases: a relayer service that offers "send your ATOS for free" by accepting delegations from holders and paying gas on their behalf; or a parent account funding a child wallet's gas without sending it any tokens.

### How to do it — CLI

```bash
# Lend 200,000 TxEnergy to <delegatee> for 7 days.
# 200,000 gas ≈ 4 ERC20 transfers.
atoshid tx energy delegate <delegatee> 200000 168h --from mywallet

# Cancel one of your outbound delegations early and reclaim the frozen ATOS.
atoshid tx energy undelegate <delegation_id> --from mywallet
```

- `AMOUNT` is TxEnergy in gas units.
- `DURATION` is a Go duration string (`24h`, `168h`, `720h`). If omitted / set to `0` on the message, the protocol default of **7 days** (`604800 s`) applies.

The underlying messages are `MsgDelegateEnergy{delegator, delegatee, amount, duration_seconds}` and `MsgUndelegateEnergy{delegator, delegation_id}`. `DelegateEnergy` returns the new `delegation_id` and the exact `locked_atos`.

### Collateral: how much ATOS gets frozen

```
threshold_units = ceil(amount / TxEnergyPerThreshold)
locked_atos     = threshold_units × TxEnergyHoldingThreshold
```

You must lock the bank balance that *would have backed* the lent capacity — the TRON-style freeze model. This prevents lending energy you do not actually back with stake.

**Rounding is up, to whole threshold blocks.** With defaults, delegating any amount from 1 up to 220,000 energy locks the full **30,000 ATOS**. Delegating 220,001 locks **60,000 ATOS**. Wallet UIs must surface this — users will otherwise be surprised that lending "a little" energy freezes a whole block of ATOS.

### Delegate flow (keeper `Delegate`)

1. Reject `amount == 0`, `duration ≤ 0`, and `delegator == delegatee` (`ErrSelfDelegation`).
2. Compute `locked_atos` (ceil, as above).
3. `Settle` the delegator. Require free energy `(tx_energy_accrued − delegated_out) ≥ amount` (`ErrInsufficientEnergy`).
4. Require free bank balance `≥ locked_atos` (`ErrInsufficientBalance`).
5. Write the delegator counters **before** the bank move: `delegated_out += amount`, `locked_atos += locked`. (Pre-writing keeps the send hook's projected eligible balance stable, so a delegation is capacity-neutral — see the inline audit notes.)
6. `bank.SendCoinsFromAccountToModule(delegator → energy_locked_pool, locked)`.
7. `Settle` the delegatee, `delegated_in_usable += amount`.
8. Insert `EnergyDelegation{id, …, used: 0}`.

Non-obvious properties:

- The delegator's `tx_energy_accrued` is **not** decreased — `delegated_out` rises instead, and the usable formula `accrued − delegated_out + delegated_in_usable` handles the rest.
- A single delegatee can receive from many delegators; the incoming grants simply sum into one `delegated_in_usable` counter.

### Consume attribution — soonest-expiring first

When the delegatee spends from `delegated_in_usable`, the keeper attributes the burn across that account's inbound delegations **ordered by `expires_at` ascending** (soonest deadline first; tie-break on id for determinism). This was audit Issue 7: id-order attribution could burn a 30-day grant before a 30-minute one, wasting the short-tenor grant. Oldest-deadline-first extracts maximum value from each grant before it expires.

### Undelegate (early revocation) and automatic expiry

`MsgUndelegateEnergy` is callable **only by the original delegator**. It and the EndBlocker expiry sweep share one code path, `releaseDelegation`:

1. `Settle` the delegator. **Deduct the already-consumed `used` from the delegator's `tx_energy_accrued`** (audit Issue-2 round2 — see the security section below), then `delegated_out −= amount`, `locked_atos −= this delegation's locked`.
2. Refund the locked ATOS from `energy_locked_pool` back to the delegator.
3. `Settle` the delegatee, `delegated_in_usable −= remaining` (the unused part).
4. Delete the record and emit an event.

Already-consumed energy stays consumed — the delegatee got that value.

`SweepExpiredDelegations` runs in the **EndBlocker**, iterating the `by_expires_at` index (sorted, so O(expired) per block). Expired releases produce **block events, not a transaction** — `energy_delegation_expired` is *not* retrievable via `cosmos/tx/v1beta1/txs?query=…`. Wallets that need to detect "my delegation just expired" must subscribe to block events over WebSocket or poll the delegations query.

## Why delegated energy cannot be used by anyone else

This is the third headline question, and the security core of the design: **once you delegate energy, it is exclusively the delegatee's until it expires — not yours, not re-lendable, not reachable by any third party — and the ATOS that backs it is frozen.** Four independent mechanisms enforce this.

### 1. The delegator loses the use of it immediately

Delegating raises `delegated_out`, and every consume path computes the delegator's own available energy as `tx_energy_accrued − delegated_out`. The moment the delegation lands, that slice is subtracted from what the delegator can spend. The delegator cannot both lend energy *and* keep spending it.

### 2. Only the named delegatee can spend it

The lent `amount` is added solely to the *delegatee's* `delegated_in_usable`. No other account's state is touched. Energy is spent only from the signer's own account row, so a third party has no counter from which to draw the delegation — there is no bearer instrument, no transferable claim, nothing another address can present to spend it.

### 3. Delegated energy is not transitively re-delegatable

The delegate path requires free energy `tx_energy_accrued − delegated_out ≥ amount` — it looks **only** at the delegatee's *own* accrued energy and deliberately **excludes** `delegated_in_usable`. So a delegatee can *spend* borrowed energy on their own txs but cannot re-lend it onward. Delegation is exactly one hop deep. This stops an attacker from laundering a single locked stake through a chain of accounts to multiply usable energy.

### 4. The backing ATOS is frozen, and consumed energy is debited on release

Collateral is **physically moved** to `energy_locked_pool`, so it leaves the delegator's spendable bank balance entirely — no one, delegator included, can transfer or re-use it while the delegation is live. It also drops out of the delegator's eligible balance, so it stops generating fresh capacity for the duration. The ATOS returns only on undelegate or expiry.

The subtle attack this closes (audit Issue-2 round2): suppose release only shrank `delegated_out` and left `tx_energy_accrued` untouched. A delegator could `Delegate → let the delegatee burn it → Undelegate`, and because `delegated_out` fell back to zero while `tx_energy_accrued` still counted the burned units as own-available, the delegator would recover energy the delegatee had *already spent* — two transactions' worth of gas out of one accrued budget, repeatable without bound. The fix deducts `used` from the delegator's `tx_energy_accrued` on release, so energy the delegatee consumed is genuinely gone from the delegator too. Energy is conserved end to end: it is spent exactly once, by exactly one account.

## Estimation: previewing the cost of a tx

Before sending, a wallet can preview the split without broadcasting:

```bash
atoshid query energy estimate-fee <signer> <gas_limit> [--deploy]
# REST: GET /atoshi/energy/v1/estimate_fee?signer=&gas_limit=&is_deploy=
```

Response:

```
energy_used   uint64   // gas covered by accrued + delegated energy
shortfall_gas uint64   // gas requiring ATOS
atos_fee      string   // shortfall_gas × insufficient_gas_price
free          bool     // true if every msg is subsidized
```

It runs the same code path as the ante-handler (`EstimateConsume`) without committing state.

## Querying energy state

```bash
# Settled account state + current capacity ceilings.
atoshid query energy account <address>
#   settled.tx_energy_accrued      spendable TxEnergy
#   settled.deploy_energy_accrued  spendable DeployEnergy
#   settled.delegated_in_usable    lent to me by others
#   settled.delegated_out          I lent to others
#   settled.locked_atos            ATOS frozen by my outbound delegations
#   tx_energy_capacity             upper bound at my current balance
#   deploy_energy_capacity         constant DeployEnergy ceiling

# Active delegations. direction ∈ {out, in, all} (default all).
atoshid query energy delegations <address> [direction]

# Module parameters.
atoshid query energy params
```

REST equivalents live under `/atoshi/energy/v1/` (`account/{address}`, `delegations/{address}`, `params`, `estimate_fee`).

## Parameters

All governance-tunable via `MsgUpdateParams` (authority = gov module). Defaults in `types/helpers.go`:

| Param | Default | Notes |
|---|---|---|
| `energy_enabled` | `true` | Master switch; if false, the ante-handler falls back to standard SDK fee deduction. Accrued energy and delegations are preserved. |
| `tx_energy_holding_threshold` | 30,000 × 10^18 aatos | ATOS per capacity block |
| `tx_energy_per_threshold` | 220,000 | TxEnergy units per block |
| `tx_energy_max_accrue_window` | 86,400 (s) | Time to refill from 0 to cap |
| `deploy_holding_threshold` | 1,000,000 × 10^18 aatos | ATOS per deploy unit |
| `deploy_energy_capacity` | 800,000 | **Constant** DeployEnergy ceiling (not a per-block multiplier) |
| `deploy_recover_days` | 10 | Refill window at one unit; larger holdings refill proportionally faster |
| `insufficient_gas_price` | 0.0021 aatos/gas | Shortfall fee rate, also the ceiling on user-offered price |
| `subsidized_msg_type_urls` | (see list above) | Always-free messages |
| `privacy_relayer_whitelist` | empty | Reserved for privacy-module L2 quota mapping |

The protocol default delegation duration (`604800 s` = 7 days) is a code constant (`types.DefaultDelegationDurationSeconds`), applied by the msg server when a delegation is submitted with `duration_seconds == 0`; it is not a governance param.

## Events

| Event | Attributes | When |
|---|---|---|
| `energy_consumed` | `address`, `energy_used`, `shortfall_gas` | Once per tx, by the ante-handler |
| `energy_delegated` | `delegation_id`, `delegator`, `delegatee`, `amount`, `locked_atos` | On successful `MsgDelegateEnergy` |
| `energy_undelegated` | `delegation_id`, `delegator`, `delegatee`, `amount` | On successful `MsgUndelegateEnergy` |
| `energy_delegation_expired` | `delegation_id`, `delegator`, `delegatee`, `amount` | EndBlocker, when `expires_at < now` |

`energy_undelegated` and `energy_delegation_expired` share the `releaseDelegation` code path, so their attribute sets are identical — consumers distinguish them by event `type`. `energy_delegation_expired` is emitted at **block level**, not in a tx (see the expiry note above).

## The EVM boundary

EVM transactions (`/ethermint.evm.v1.MsgEthereumTx`), including contract-creation txs, are **not** processed by the energy module. The EVM ante chain (`MonoDecorator`) handles their gas in standard ATOS. `isContractDeployMsg` therefore only recognizes native Cosmos deploy messages; the previously-present `MsgEthereumTx` branch was unreachable and was removed. If and when EVM gas is bridged to energy, this is the seam that changes. See [Cosmos L1](../architecture/02-cosmos-l1.md).

## Edge cases and operator notes

**Balance drops below threshold after accrual.** The cap-down in `ApplyBalanceChange` clips `tx_energy_accrued` to the new lower capacity at the moment the balance changes (floored at `delegated_out`). Accrued energy above the new cap is forfeited.

**Delegator's free balance vs. locked collateral.** Can never invert — the collateral is *moved* to the module account, not earmarked in the user's balance, so it is bank-isolated. Staking slashing during the delegation can still hit non-locked stake, but the delegation collateral is safe.

**Two delegations to the same delegatee.** Compose — `delegated_in_usable` sums across all inbound grants. Each record is independent; when one expires only its share returns, the other(s) stay active. Burn is attributed soonest-expiring-first.

**Self-delegation.** Rejected with `ErrSelfDelegation`. It would be a no-op that only muddies bookkeeping.

**Race conditions.** All keeper operations run under the SDK's serial transaction model — no within-block concurrency. `Settle` is idempotent within a block.

## Related modules

- **`x/bank`** — capacity depends on bank balance via `EligibleBalance(addr) = bank − locked_atos`. The bank send hook (`SendRestriction`) calls `ApplyBalanceChange` so capacity reflects transfers immediately.
- **`x/tokenomics`** — claim messages are subsidized so users with no energy can still claim unlocks.
- **`x/oracle`** — `MsgReportPrice` is subsidized so feeders can post prices without holding ATOS.
- **`x/evm`** — bypassed by energy; EVM txs pay standard ATOS gas. See [The EVM boundary](#the-evm-boundary).

## Further reading

- [Gas and energy economics](../economics/03-gas-fee.md) — fee curves, parameter sensitivity, monetary analysis
- [Wallet integration handbook](../reference/wallet-integration.md) — displaying energy in a wallet UI
- [API reference](../reference/api-guide.md#energy) — all REST endpoints under `/atoshi/energy/v1/`

---

*Last reviewed: 2026-07-10*
