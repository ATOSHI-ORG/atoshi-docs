# `x/energy` — gas via holding, not paying

The energy module is the centerpiece of Atoshi's user-experience design. Instead of charging every transaction in scarce native tokens, the module gives each account a *capacity* that refills over time as long as the holder maintains a minimum ATOS balance. Transactions consume from this capacity first; only the shortfall is paid in ATOS.

This chapter is the canonical specification of the module: data model, accrual mathematics, delegation, consumption order, ante-handler integration, parameters, edge cases. If you only read one module chapter, read this one.

## Mental model

Hold ATOS → earn the right to transact for free, at a rate proportional to your holding.

Specifically:

- Holding **30,000 ATOS** unlocks **50,000 TxEnergy capacity**, refilled linearly over **24 hours**.
- Holding **1,000,000 ATOS** additionally unlocks **800,000 DeployEnergy capacity**, refilled linearly over **10 days**.
- Both are uint64 counters tracked per account.
- Both can be delegated to other accounts for a fixed duration; the corresponding ATOS is locked as collateral.

`TxEnergy` covers ordinary transactions (transfers, contract calls, delegations). `DeployEnergy` covers contract deployment messages (Wasm instantiate / store-code, EVM creation tx wrapped as a Cosmos msg).

## State

### `EnergyAccount`

One record per account. Definition mirrors `x/energy/types/energy.proto`:

```
message EnergyAccount {
  string  address                = 1;   // bech32, the account this row describes
  uint64  tx_energy_accrued      = 2;   // current TxEnergy held by the account
  uint64  deploy_energy_accrued  = 3;   // current DeployEnergy held by the account
  string  last_balance_snapshot  = 4;   // ATOS balance at last settle, used for capacity
  int64   last_updated_time      = 5;   // unix time of last settle
  uint64  delegated_out          = 6;   // TxEnergy currently lent to others
  uint64  delegated_in_usable    = 7;   // TxEnergy received from others, still usable
  string  locked_atos            = 8;   // total ATOS currently in collateral lock
}
```

The four spendable / locked totals (`tx_energy_accrued`, `delegated_out`, `delegated_in_usable`, `locked_atos`) are denormalized for fast read access. The authoritative ledger of outstanding delegations lives in the separate `EnergyDelegation` table.

### `EnergyDelegation`

One record per active delegation:

```
message EnergyDelegation {
  uint64 id              = 1;   // monotonic, unique across all delegations
  string delegator       = 2;
  string delegatee       = 3;
  uint64 amount          = 4;   // TxEnergy units lent
  string locked_atos     = 5;   // collateral held in the locked module account
  int64  start_time      = 6;
  int64  expires_at      = 7;   // auto-released by EndBlocker at this time
  uint64 used            = 8;   // accumulated consumption by the delegatee
}
```

Two secondary indices accelerate common queries:
- by `delegator` — wallet "my outgoing delegations" view
- by `expires_at` (8 bytes big-endian, then 8 bytes id) — EndBlocker sweep

### Module accounts

- `energy_locked` — holds all ATOS posted as delegation collateral. Module account `atoshi12ham4nq8y2rpqs63wcuh4adnjwev6qavrc2wp5`.

## Capacity formula

Capacity is a step function of the account's bank balance (locked-out collateral excluded):

```
eligible_balance = bank.GetBalance(address) - locked_atos
threshold_blocks = floor(eligible_balance / TxEnergyHoldingThreshold)

tx_energy_capacity = threshold_blocks × TxEnergyPerThreshold
```

With default params (`TxEnergyHoldingThreshold = 30,000 ATOS`, `TxEnergyPerThreshold = 50,000`):

| Eligible balance | Capacity |
|---|---|
| 0 – 29,999 ATOS | 0 |
| 30,000 – 59,999 ATOS | 50,000 |
| 60,000 – 89,999 ATOS | 100,000 |
| 1,000,000 ATOS | 1,650,000 |

The step function is deliberate: it makes the holding requirement unambiguous and easy to communicate ("hold 30,000 ATOS to transact for free"). A linear function would let attackers earn near-zero capacity by holding dust.

DeployEnergy uses the same shape but with `DeployHoldingThreshold = 1,000,000 ATOS` and capacity `800,000` per block.

## Accrual: how energy refills

Energy refills **lazily** — there is no scheduled job that touches every account every block. Instead, the chain remembers `last_updated_time` and `last_balance_snapshot` per account, and recomputes accrued energy when the account is next read or written. This pattern, called `Settle`, is invoked:

- by the ante-handler before fee deduction (mutating call)
- by every keeper message handler at the start (mutating call)
- by queries (non-mutating: `SimulateSettle`)

### The refill equation

For TxEnergy:

```
elapsed = now - last_updated_time
cap     = TxEnergyCapacity(last_balance_snapshot, params)
add     = (elapsed × cap) / TxEnergyMaxAccrueWindow

new_accrued = min(tx_energy_accrued + add, cap)
```

Default `TxEnergyMaxAccrueWindow` is 86,400 (24 hours). So a holder at 30,000 ATOS regains 50,000 / 86,400 ≈ 0.578 energy per second, until the cap.

For DeployEnergy, the per-second rate is `(units × capacity) / (DeployRecoverDays × 86,400)`, where `units = floor(balance / DeployHoldingThreshold)`. The formula is rearranged to multiply before dividing so that low-holding-but-positive accounts don't lose precision to integer truncation.

### Why `last_balance_snapshot` and not `current_balance`

The snapshot is what the balance was *during the period being settled*. If we used `current_balance` to compute capacity over a period when the balance had changed, we'd misattribute accrual.

The bank module hooks `OnBalanceChange`, which:
1. Calls `Settle` first — closes out accrual against the OLD snapshot.
2. Updates `last_balance_snapshot` to the NEW eligible balance.
3. Caps `tx_energy_accrued` down to the new (possibly lower) capacity.

That third step (cap-down) is critical: an account that sells most of its ATOS should not retain a giant prefilled buffer to be moved to a new wallet for free transactions. Without the cap-down, capacity would only ever ratchet upward.

## Consumption: the ante-handler pipeline

`x/energy/ante.EnergyDeductDecorator` replaces the SDK's stock `DeductFeeDecorator`. For every Cosmos tx:

1. Decode the tx, identify signers, gas limit, message type URLs.
2. Check the subsidized whitelist. If every message in the tx is subsidized, return `{Free: true}` and skip fee deduction entirely.
3. Otherwise, call `Consume(signer, gasLimit, isDeploy, msgTypeUrls)`. This:
   - For deploy txs: drains `DeployEnergyAccrued` first, then `TxEnergyAccrued`, then `DelegatedInUsable`.
   - For regular txs: drains `TxEnergyAccrued` (less `DelegatedOut`), then `DelegatedInUsable`.
   - Returns `ConsumeResult{EnergyDeducted, DeployEnergyUsed, ShortfallGas, Free}`.
4. If `ShortfallGas == 0`: skip the fee deduction.
5. Otherwise: compute the ATOS shortfall fee and call `bank.SendCoinsFromAccountToModule(signer, fee_collector, ...)`.

The post-handler runs after the message handlers and refunds any reserved energy that wasn't actually used (gas refund equivalent).

### Subsidized whitelist (always-free messages)

Defined in `x/energy/types.DefaultParams().SubsidizedMsgTypeUrls`:

| Type URL | Purpose |
|---|---|
| `/atoshi.tokenomics.v1.MsgClaimMigrationTokens` | Legacy chain migration claim |
| `/atoshi.tokenomics.v1.MsgClaimMinerLockedReward` | Validator unlock claim |
| `/atoshi.tokenomics.v1.MsgClaimProjectTreasuryReward` | Treasury claim |
| `/atoshi.oracle.v1.MsgReportPrice` | Allow-listed feeder price report |
| `/atoshi.energy.v1.MsgDelegateEnergy` | Energy delegation (collateral is the anti-spam) |
| `/atoshi.energy.v1.MsgUndelegateEnergy` | Energy revocation |

Subsidization is global — even an account with zero balance and zero energy can call these messages free of charge. The anti-abuse mechanism for delegation is the ATOS collateral lock; for claim messages it is application-level authorization (e.g. only the validator can claim their own locked reward).

### Shortfall pricing

If energy is short, the chain charges:

```
charge = min(user_offered_gas_price, params.InsufficientGasPrice) × shortfall_gas
```

`InsufficientGasPrice` defaults to `0.0021 aatos / gas`. The cap on the user's offered price prevents fee griefing: a user can't pay a non-energy fee 100x the standard rate to push other tx out of the mempool — the chain charges at most the policy price.

If the user offered zero fee, the chain still charges at the policy price. Otherwise the user could dodge the shortfall by setting `tx.fee = 0`.

## Delegation: lending energy to another account

Use case: a Relayer service offers "send your ATOS for free" by accepting energy delegations from holders, then pays gas on the holders' behalf. Or: a parent account funds a child wallet's gas without sending tokens.

### Delegate flow

```
MsgDelegateEnergy {
  delegator        string  // signer
  delegatee        string  // recipient
  amount           uint64  // TxEnergy units
  duration_seconds int64   // lock period
}
```

Keeper logic (simplified from `keeper/delegation.go`):

1. Compute collateral: `locked = ceil(amount / TxEnergyPerThreshold) × TxEnergyHoldingThreshold`.
2. Settle delegator. Require `(tx_energy_accrued - delegated_out) >= amount`.
3. Require free ATOS balance ≥ `locked`.
4. `bank.SendCoinsFromAccountToModule(delegator, energy_locked, locked)`.
5. `delegator.DelegatedOut += amount`
   `delegator.LockedAtos += locked`
   `delegator.LastBalanceSnapshot = bank.GetBalance(delegator)` ← drops by `locked`.
6. Settle delegatee, then `delegatee.DelegatedInUsable += amount`.
7. Insert `EnergyDelegation{id, delegator, delegatee, amount, locked, start, expires_at, used=0}`.

**Important non-obvious behavior**:

- Collateral is rounded **up** to whole `TxEnergyHoldingThreshold` blocks. Delegating any amount up to 50,000 energy locks 30,000 ATOS (the full block). Delegating 50,001 locks 60,000. UI must communicate this — users will be surprised otherwise.
- The delegator's `tx_energy_accrued` is **not** decreased. Their `delegated_out` is increased instead. The usable energy formula stays `accrued - delegated_out + delegated_in_usable`.
- Delegated energy is **NOT** transitively delegatable. The delegatee can spend it but cannot re-delegate, because the delegate path checks `accrued - delegated_out` (excluding `delegated_in_usable`).

### Consume order with delegations

When the delegatee spends gas, the ante-handler drains:

1. The delegatee's own `tx_energy_accrued - delegated_out`.
2. Then `delegated_in_usable` (across all incoming delegations — there's a single counter, no per-delegation accounting in the consume path).

The `delegated_in_usable` counter shrinks as the delegatee consumes; the keeper does not attempt to assign consumption to a specific delegation record's `used` field on every tx (that would be costly). Instead, periodic reconciliation or the expiry sweep figures out how the consumption attributes back.

### Undelegate (early revocation)

`MsgUndelegateEnergy{delegator, delegation_id}` — only callable by the original delegator. Frees the collateral, removes unused energy from the delegatee, deletes the record. Already-consumed energy stays consumed (the delegatee got the value).

### Automatic expiry

`SweepExpiredDelegations` runs in the EndBlocker. It iterates the `by_expires_at` secondary index (sorted, so O(expired) per block, not O(total)) and releases each expired delegation via the same code path as undelegate. The released event has type `energy_delegation_expired` instead of `energy_undelegated`.

**This means**: expired releases produce no transaction. They are EndBlocker events, not indexable by `cosmos/tx/v1beta1/txs?query=...`. Wallets that want to detect "my delegation just expired" must either subscribe to block events via WebSocket or poll the delegations query.

## Estimation: previewing the cost of a tx

Before sending a tx, a wallet can call:

```
GET /atoshi/energy/v1/estimate_fee?signer=&gas_limit=&is_deploy=
```

Returns:

```
{
  energy_used:   uint64   // gas covered by energy
  shortfall_gas: uint64   // gas requiring ATOS
  atos_fee:      string   // shortfall × insufficient_gas_price
  free:          bool     // true if every msg is subsidized
}
```

Use this to display per-tx cost in the wallet. The chain side just runs the same code path as the ante-handler without committing state.

## Parameters

All governance-tunable. Default values in `types/helpers.go`:

| Param | Default | Notes |
|---|---|---|
| `energy_enabled` | true | Master switch; if false, falls back to standard SDK fee deduction |
| `tx_energy_holding_threshold` | 30,000 × 10^18 aatos | ATOS per capacity block |
| `tx_energy_per_threshold` | 50,000 | TxEnergy units per block |
| `tx_energy_max_accrue_window` | 86,400 (sec) | Refill from 0 to cap takes this long |
| `deploy_holding_threshold` | 1,000,000 × 10^18 aatos | ATOS per deploy capacity block |
| `deploy_energy_capacity` | 800,000 | DeployEnergy per block (NOT per-threshold-times-blocks; this IS the cap) |
| `deploy_recover_days` | 10 | Refill window |
| `insufficient_gas_price` | 0.0021 aatos / gas | Shortfall fee rate, also floor for user offers |
| `subsidized_msg_type_urls` | (see list above) | Always-free messages |
| `privacy_relayer_whitelist` | empty | Reserved for future privacy module integration |

Note: `deploy_energy_capacity` is named confusingly. It is the maximum capacity for an account at exactly one threshold block; total capacity is `units × deploy_energy_capacity`. The name is preserved for backward compatibility with genesis state.

## Events

Emitted as standard SDK events on the tx (or block, for `_expired`):

| Event | Attributes | When |
|---|---|---|
| `energy_consumed` | `address`, `energy_used`, `shortfall_gas` | Once per tx, by the ante-handler |
| `energy_delegated` | `delegation_id`, `delegator`, `delegatee`, `amount`, `locked_atos` | On successful `MsgDelegateEnergy` |
| `energy_undelegated` | `delegation_id`, `delegator`, `delegatee`, `amount` | On successful `MsgUndelegateEnergy` |
| `energy_delegation_expired` | `delegation_id`, `delegator`, `delegatee`, `amount` | EndBlocker, when `expires_at < now` |

`energy_undelegated` and `energy_delegation_expired` share the keeper code path (`releaseDelegation`), so their attribute sets are identical. Consumers distinguish them by event `type`. Important: `energy_delegation_expired` is emitted at block level, not in a tx — it is not retrievable via `cosmos/tx/v1beta1/txs?query=...`.

## Edge cases and operator notes

### What if balance drops below threshold after accrual?

The cap-down step in `OnBalanceChange` clips `tx_energy_accrued` to the new (lower) capacity at the moment the balance changes. Already-accrued energy above the new cap is forfeited.

### What if a delegator's free balance falls below their locked collateral?

It can't — the collateral is *moved* to the module account, not earmarked in the user's balance. The locked ATOS is bank-isolated. Slashing during the delegation period can still hit non-locked stake, but the delegation collateral is safe.

### Can two delegations to the same delegatee compose?

Yes. The delegatee's `delegated_in_usable` is the sum across all incoming delegations. Each contributor's record is independent — when one expires, only its share returns; the other(s) remain active.

### What stops me from delegating to myself?

The keeper rejects `delegator == delegatee` with `ErrSelfDelegation`. Self-delegation would be a no-op (lock ATOS, lend yourself energy, take it back when you call consume), but the keeper rejects it to keep the bookkeeping clean.

### Why is collateral the ATOS that "would have backed" the lent capacity?

Imagine you hold 30,000 ATOS (50,000 energy capacity) and delegate 10,000 energy to me. If you didn't lock any ATOS, you would still have 50,000 capacity AND have given me an extra 10,000 — net 60,000 units of capacity from 30,000 ATOS, an infinite loop of you delegating and reclaiming. Locking the ATOS makes your real capacity drop while the delegation is active, balancing the books.

The "ceil" in the collateral formula errs on the side of over-locking; over-collateralization is safe (extra ATOS unlocks on release), but under-collateralization breaks the invariant.

### Race conditions

All keeper operations execute under the SDK's serial transaction model — there is no within-block concurrency. `Settle` is idempotent in the same block. The lazy-settle pattern means you cannot observe stale energy values from outside the chain: the `Account` query path calls `SimulateSettle` before returning.

## Related modules

- **`x/bank`** — energy capacity depends on bank balance via `EligibleBalance(addr)`. The bank send hook calls `energy.OnBalanceChange` so capacity reflects balance changes immediately.
- **`x/tokenomics`** — claim messages (`MsgClaimMigrationTokens`, etc.) are subsidized so users with no energy can still claim their unlocks.
- **`x/oracle`** — `MsgReportPrice` is subsidized so feeders can post prices without holding ATOS.
- **`x/evm`** — currently bypassed by energy. EVM txs always pay standard ATOS gas. See [Cosmos L1](../architecture/02-cosmos-l1.md) for the boundary.

## Further reading

- [Gas and energy economics](../economics/03-gas-fee.md) — fee curves, sensitivity to parameter changes, monetary analysis
- [Wallet integration handbook](../reference/wallet-integration.md) — how to display energy in a wallet UI
- [API reference](../reference/api-guide.md#energy) — all REST endpoints under `/atoshi/energy/v1/`

---

*Last reviewed: 2026-05-21*
