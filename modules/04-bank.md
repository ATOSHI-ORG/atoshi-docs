# `x/bank` — native token transfers and balances

The Cosmos SDK bank module owns ATOS balances and transfers. Atoshi uses it
mostly as-is but adds one consequential hook: a `SendRestriction` callback
from the energy module that keeps every account's energy snapshot in sync
with its actual bank balance.

## Mental model

ATOS lives in `x/bank` regardless of which module created it. Module
accounts (energy locked-pool, tokenomics pools, fee_collector, gov)
all hold their balances here. The bank module sees every credit and debit
on the chain — which is why it's the natural place to hook for any
secondary side-effect.

## What Atoshi changed

The base SDK bank module is unmodified. The only Atoshi customization is at
the application wiring level: in `app/app.go`, we register an extra
`SendRestriction` callback on the bank keeper.

```go
app.BankKeeper.AppendSendRestriction(
    app.EnergyKeeper.SendRestriction,
)
```

The energy keeper's `SendRestriction` is called by bank on every send,
before the transfer commits. It does two things:

1. **Snapshot refresh.** Both `from` and `to` accounts have their
   `EnergyAccount.LastBalanceSnapshot` reset to their post-transfer
   eligible balance. This is the fix shipped in v20.1 — without this hook
   the receiver's snapshot stayed at its initial value and accrued
   capacity never reflected later inflows.
2. **Locked-ATOS enforcement.** If `from` has outstanding energy
   delegations, the portion of their bank balance that backs those
   delegations (tracked in `EnergyAccount.LockedAtos`) is treated as
   non-spendable. Attempting to send more than `balance - locked_atos`
   results in a transfer rejection.

The hook does NOT modify the transfer amount, the participants, or the
post-state on success. It only:

- updates energy module storage as a side effect, and
- can return an error to reject the transfer.

## State

The bank module owns:

- `BalancesPrefix` — per-account, per-denom balances
- `DenomMetadataKey` — display names, decimals, descriptions for each denom
- `SupplyKey` — total supply per denom
- Per-denom send-enabled / receive-enabled toggles

ATOS is registered with denom metadata:

```yaml
base:     aatos          # atomic unit, on-chain wire format
display:  atos           # 18-decimal display unit
description: "Atoshi native token"
denom_units:
  - { denom: aatos, exponent: 0  }
  - { denom: atos,  exponent: 18 }
```

The chain's `x/erc20` module additionally registers an ERC20 pair for
aatos at deterministic address `0xD4949664cD82660AaE99bEdc034a0deA8A0bd517`,
so EVM contracts can interact with native ATOS like any other ERC20.

## Messages

Standard SDK bank messages — nothing Atoshi-specific:

| Type URL | Purpose |
|---|---|
| `/cosmos.bank.v1beta1.MsgSend` | Transfer between two accounts |
| `/cosmos.bank.v1beta1.MsgMultiSend` | One-to-many transfer in a single tx |
| `/cosmos.bank.v1beta1.MsgUpdateParams` | gov-only param update |
| `/cosmos.bank.v1beta1.MsgSetSendEnabled` | gov-only per-denom send toggle |

`MsgSend` is by far the most common Cosmos message wallets need to support.
It is NOT in the energy subsidized whitelist — transfers cost gas, paid in
energy first and ATOS shortfall if needed.

## Queries

Standard REST endpoints under `/cosmos/bank/v1beta1/`:

| Endpoint | Returns |
|---|---|
| `GET /balances/{addr}` | All denoms held by an account |
| `GET /balances/{addr}/by_denom?denom=aatos` | Single denom balance |
| `GET /supply` | Total supply per denom (paginated) |
| `GET /supply/by_denom?denom=aatos` | Specific denom total |
| `GET /denoms_metadata` | All denom metadata records |
| `GET /params` | Module params (send-enabled defaults) |

For ATOS specifically, the `bank.balance` value excludes nothing — locked
delegation collateral has already been transferred OUT to the energy
locked-pool module account, so it isn't present in the user's bank entry.
Wallets should NOT subtract `EnergyAccount.LockedAtos` from `bank.balance`
to display "spendable balance" because that would double-count.

## Send restriction in detail

The `SendRestriction` callback signature:

```go
func (k Keeper) SendRestriction(
    ctx context.Context,
    from, to sdk.AccAddress,
    amt sdk.Coins,
) (sdk.AccAddress, error)
```

For each ATOS transfer:

```
1. Skip if amount doesn't include base denom (aatos).
2. Read both accounts' EnergyAccount records.
3. Compute the new eligible balance for `from`:
       new_from_eligible = bank.GetBalance(from, aatos) - amt - from.LockedAtos
   If new_from_eligible < 0:
       return error: insufficient unlocked balance
4. Run `energy.Settle(from)` against the OLD snapshot, then
   set from.LastBalanceSnapshot to the new eligible balance, then
   call `cap-down accrued to new capacity`.
5. Same for `to`: settle, set new snapshot, cap-down accrued.
6. Return (to, nil) — the transfer proceeds.
```

The `from` energy account is touched on outflows, the `to` energy account
on inflows. Both sides stay in sync without separate hooks.

## Edge cases

### Multi-coin sends

`MsgSend` can carry multiple coin denoms. The restriction only inspects
the `aatos` portion. Non-ATOS coins (ibc tokens, ERC20-bridged tokens)
have no impact on energy state.

### Self-send

A send from an address to itself (`from == to`) still passes through the
restriction. Settle is called twice on the same account; the second call
is a no-op because `LastUpdatedTime` already matches the current block.
The snapshot writes are also idempotent. No special-casing needed.

### Module account transfers

Internal transfers (e.g. tokenomics → fee_collector) flow through the
same hook. Module accounts don't have an `EnergyAccount` record by
default, so the hook is a no-op for them — but it's still safe to call.

### Slashing

When a validator is slashed, bank moves coins out of their bonded amount
into the community pool. This is treated like any other transfer and
does fire `SendRestriction`. Since validator operator accounts don't
typically hold energy delegations against their own self-bond, the
locked-ATOS check passes trivially. If they did, the chain would refuse
to slash beyond their unlocked balance.

### Gas cost

`SendRestriction` adds 2 reads + 2 writes to the KVStore per transfer
(one each for `from` and `to`). At ~3 gas/byte read, ~30 gas/byte write,
this adds roughly 1,500–3,000 gas per `MsgSend`. Negligible compared to
the message's base cost (~70k–100k gas).

## Parameters

| Name | Default | Notes |
|---|---|---|
| `default_send_enabled` | true | Globally enable bank sends |
| `send_enabled` (per-denom) | `[]` | Override for specific denoms; empty means "use default" |

The chain ships with all denoms enabled. Governance can disable a denom
temporarily (e.g. for an emergency halt of a stolen token) via
`MsgSetSendEnabled`.

## Events

Standard SDK events:

| Event | Attributes |
|---|---|
| `transfer` | `recipient`, `sender`, `amount` |
| `coin_spent` | `spender`, `amount` |
| `coin_received` | `receiver`, `amount` |
| `message` | `action`, `sender`, `module` |

Wallet transfer history can be reconstructed from these via
`/cosmos/tx/v1beta1/txs?query=transfer.sender=...` plus a second query
on `transfer.recipient=...` and a client-side merge (Tendermint
tx-search does not support `OR`).

## Related modules

- [`x/energy`](./01-energy.md) — registers the `SendRestriction` hook; consumes
  energy first when transfer is sent
- [`x/erc20`](#) — exposes aatos as ERC20 to EVM contracts at a fixed address
- [`x/evm`](./05-evm.md) — reads bank balances when servicing
  `eth_getBalance` calls; balances are unified across both interfaces

---

*Last reviewed: 2026-06-10*
