# `x/evm` — Ethereum-compatible execution

Atoshi's L1 hosts a full Ethereum Virtual Machine inside the Cosmos SDK
state machine, via the `x/evm` module from the ethermint stack. EVM
transactions execute in the same blocks as Cosmos transactions and read
the same bank balances — there is no wrapped representation of ATOS at L1.

## Mental model

Two distinct transaction encodings, one shared state:

```
                  Cosmos tx                 Ethereum tx
              (signed protobuf)         (signed RLP, type 0/2)
                     │                          │
                     ▼                          ▼
       ┌─────────────────────────┐  ┌──────────────────────────┐
       │ ante.EnergyDeductDecor. │  │ ante.MonoDecorator       │
       │ (uses energy, then ATOS)│  │ (uses ATOS gas only)     │
       └─────────────────────────┘  └──────────────────────────┘
                     │                          │
                     ▼                          ▼
            ┌────────────────────────────────────────────┐
            │            Shared state machine             │
            │   x/bank balances · x/staking · x/energy    │
            │              x/evm · x/erc20                │
            └────────────────────────────────────────────┘
```

A wallet user with one private key can send either a Cosmos `MsgSend` or
an Ethereum `eth_sendRawTransaction` to the same destination address —
the resulting bank state update is identical. Tooling (MetaMask, Keplr,
ethers.js, atoshid CLI) is fully cross-compatible.

## What Atoshi changed

`x/evm` is taken from ethermint with these adaptations:

- **Chain id registry**. `x/evm/types/denom.go` registers the three valid
  Atoshi chain ids (`atoshi_88188`, `atoshi_88288`, `atoshi_88388`); any
  other chain id panics at startup. This protects against accidentally
  starting the binary against a foreign chain's genesis.
- **Base denom**: aatos (mainnet, testnet, testing all unified after a
  bug fix that previously had testnet set to `ataatos`, breaking
  `eth_getBalance`).
- **ERC20 auto-pair**: aatos is registered as an ERC20 token pair at
  deterministic address `0xD4949664cD82660AaE99bEdc034a0deA8A0bd517`,
  enabling EVM contracts to call `transferFrom` / `approve` on native ATOS.
- **Precompile registry**: standard ethermint precompiles (staking,
  distribution, ICS-20) at `0x0000…0800-0805` for cross-module EVM
  interactions.

## Why energy bypasses EVM (and what that means for users)

The `x/energy.EnergyDeductDecorator` ante-handler **only intercepts
Cosmos messages**. EVM transactions go through ethermint's
`MonoDecorator`, which does its own gas accounting and is intentionally
not modified.

Practical consequences:

- A wallet user paying gas on an EVM transfer pays standard ATOS gas at
  the prevailing `feemarket.MinGasPrice` (currently 1 gwei = 10⁹ aatos/gas).
  A 21,000-gas native transfer costs ~0.000021 ATOS.
- The same user could send the same ATOS through Cosmos `MsgSend` and
  pay zero ATOS fee (energy covers it) — as long as their holding gives
  them enough energy capacity.

We chose this boundary intentionally for v1:

- The ethermint `MonoDecorator` is well-tested and modifying it adds risk
- Wallets and SDKs in the EVM ecosystem expect predictable gas-pricing
  semantics, not a "free if you hold enough" model that doesn't map onto
  EIP-1559
- Integrating energy into EVM execution requires touching `ApplyMessage`
  and would invalidate years of Ethereum security work

A future Atoshi version may relax this. For now: **Cosmos route = energy
benefits; EVM route = standard gas.**

## State

`x/evm` stores:

- Code hash → bytecode (large blobs, content-addressed)
- Account → code hash (mapping from EVM address to deployed contract code)
- Storage slots: `address || slot_key` → 32-byte value (the EVM SSTORE storage)
- Transient state (intra-tx, cleared at FinalizeBlock)
- Genesis params (precompile list, allow/deny access controls)

EOA balances are NOT stored in `x/evm` — they're read live from `x/bank`
when an EVM opcode requests `balance(addr)`.

## Messages

```
MsgEthereumTx {
  data: bytes                       // RLP-encoded signed Ethereum tx
}

MsgUpdateParams {                   // gov-only
  authority: string
  params: Params { ... }
}
```

`MsgEthereumTx` wraps a standard Ethereum-signed transaction in a Cosmos
message envelope so it can be carried through the same mempool / ABCI
pipeline as other messages. The Cosmos signer is empty; the EVM signer
recovered from the RLP signature is what authorizes the transaction.

A few legacy / management messages exist in the type space
(`MsgCreateContract`, `MsgEthCallEntryPoint`) but are not used in
practice — clients always submit through `eth_sendRawTransaction` which
results in a `MsgEthereumTx`.

## EVM JSON-RPC

The chain exposes a standard Ethereum JSON-RPC server on port 8545. All
read methods work — `eth_getBalance`, `eth_getStorageAt`,
`eth_getTransactionByHash`, `eth_getLogs`, `eth_call`, `eth_estimateGas`,
`eth_chainId`, etc.

Write methods are limited to `eth_sendRawTransaction`. Methods that
imply chain-side key management (`eth_sendTransaction`,
`personal_sign`) are disabled — wallets must sign locally and submit raw.

Subscription via `eth_subscribe` works over WebSocket on the same port
when configured in `app.toml`.

## Gas

Gas pricing on the EVM side is governed by the separate `x/feemarket`
module (EIP-1559 base fee + min gas price). Key parameters:

| Param | Current | Notes |
|---|---|---|
| `min_gas_price` | 10⁹ aatos/gas (1 gwei) | Floor; the chain refuses tx below this |
| `base_fee` | dynamic per block | EIP-1559 base fee, adjusts based on block fullness |
| `base_fee_change_denominator` | 8 | Standard Ethereum value |
| `elasticity_multiplier` | 2 | Block can be up to 2× target gas usage |
| `min_gas_multiplier` | 0.5 | Charge at least 50% of gas_limit even if usage was lower |
| `no_base_fee` | false | EIP-1559 enabled |

Wallets calling `eth_gasPrice` should treat the result as a recommended
gas price but fall back to a sensible minimum (≥ 1 gwei) in case the
returned value is 0 due to low recent activity.

## EVM ↔ Cosmos crossover scenarios

| Action | Where it executes | Settled in |
|---|---|---|
| Native ATOS transfer via MetaMask | EVM | bank (aatos balance update) |
| Native ATOS transfer via atoshid `tx bank send` | Cosmos | bank (same balance update) |
| ERC20 transfer of a Cosmos-native denom | EVM via `x/erc20` auto-pair | bank (denom balance update) |
| Staking via MetaMask | EVM precompile `0x0000…0800` | x/staking |
| Bridge L1 → L2 | EVM (bridgeAsset call) | bridge contract holds escrow |

`x/erc20` is the lynchpin for cross-mode access: every native Cosmos
denom (IBC tokens, the chain's own ATOS) gets an automatically-deployed
ERC20 contract at a deterministic address, callable from any EVM context.

## Edge cases

### Type 0 vs Type 2 tx

The chain accepts both legacy (type 0) and EIP-1559 (type 2)
transactions. L2 currently accepts only type 0 due to a sequencer
configuration — see [modules/06-bridge.md](./06-bridge.md). Wallets that
want cross-layer compatibility should default to type 0.

### nonce semantics

EVM nonce is read from `x/auth.Account.Sequence` — the same counter
incremented by Cosmos messages. So a wallet sending mixed Cosmos +
EVM transactions sees a single monotonic counter.

This avoids a class of footguns where two parallel nonce spaces (one
for Cosmos, one for EVM) could lead to gaps and stuck transactions.

### Failed EVM tx in a successful block

If an EVM transaction reverts (out-of-gas, REVERT opcode, contract
panic), gas is still consumed and charged. The block proceeds normally
and the tx is included with `status: 0x0`. Wallets querying
`eth_getTransactionReceipt` should always check `status` — it's
possible to "successfully include" a failing transaction.

### Disabled access control

`evm.params.access_control` allows governance to limit who can call
`eth_call` (READ) or deploy contracts (CREATE). The chain currently runs
with `ACCESS_TYPE_PERMISSIONLESS` for both. If governance ever switches
to allow-list mode, expected wallets will need to be added.

## Events

EVM events surface in two parallel forms:

1. **EVM logs** — readable via `eth_getLogs`, `eth_getTransactionReceipt`,
   standard EVM tooling. Indexed by topic.
2. **Cosmos events** — same data also emitted as `cosmos.evm.v1.*` typed
   events, queryable via Tendermint tx-search at
   `/cosmos/tx/v1beta1/txs?query=...`.

Both views show the same logs; choose based on which client you're
using.

## Parameters

| Name | Default | Notes |
|---|---|---|
| `allow_unprotected_txs` | false | If true, accepts EIP-155-unprotected (pre-Spurious Dragon) txs |
| `evm_denom` | "aatos" | Native gas denom |
| `access_control.call.access_type` | PERMISSIONLESS | `eth_call` allow list |
| `access_control.create.access_type` | PERMISSIONLESS | Contract deployment allow list |
| `evm_channels` | `["channel-10", "channel-31", "channel-83"]` | IBC channels authorized to receive EVM-callable assets |
| `extra_eips` | `["ethereum_3855"]` | Activates EIP-3855 (`PUSH0`) and similar opt-in EIPs |
| `active_static_precompiles` | 8 entries | Precompile contracts at `0x100` (modexp), `0x400` (bls), `0x800-0x805` (Cosmos integration) |

## Related modules

- [`x/bank`](./04-bank.md) — backs EOA balances; reads/writes go through here
- [`x/erc20`](#) — bridges Cosmos denoms to ERC20 contracts
- [`x/feemarket`](#) — owns EVM gas pricing
- [`x/energy`](./01-energy.md) — does NOT cover EVM transactions (intentional)
- [`x/evidence`, `x/staking`](#) — surfaced through precompiles for EVM
  contract interaction

## Further reading

- [Ethermint documentation](https://docs.evmos.org/protocol/concepts/chain-id)
- [Wallet integration handbook](../reference/wallet-integration.md) — practical
  examples of building and signing both Cosmos and Ethereum transactions

---

*Last reviewed: 2026-06-10*
