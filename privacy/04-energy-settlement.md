# Energy settlement on L2 — gasless private transactions

The L1 [`x/energy`](../modules/01-energy.md) module subsidizes Cosmos
transactions for holders. The natural extension is to bring the same
benefit to L2 privacy operations — most of which would otherwise cost
~280k EVM gas due to Groth16 proof verification.

This chapter explains how the **EnergySettlement** contract on L2
implements that bridge: a holder's L1 energy capacity can pay for their
L2 private transactions without the holder ever signing an L1 tx.

## Why this is needed

A naive privacy interaction on L2 looks like:

```
User → MetaMask → eth_sendRawTransaction
                  Shield.deposit(commitment, asset, amount)
                  Gas: ~280,000 × 1 gwei = 0.00028 ATOS
```

That cost is negligible in absolute terms, but it has three serious UX
problems:

1. **Hot wallet must hold gas.** A user shielding wATOS needs both wATOS
   AND a small amount of L2 gas token. New users have neither.
2. **Privacy leak.** The signing address that paid gas is publicly tied
   to the deposit event. Even though the commitment itself hides the
   note, on-chain analysis can map "wallet X paid gas on deposit Y" with
   probability 1.
3. **Stuck transactions.** If the user's wallet runs out of gas mid-
   privacy-flow (e.g. doing 3 deposits and only enough gas for 2), the
   third one fails and they need to top up.

EnergySettlement solves all three by introducing a **meta-transaction
relayer** that holds the gas, signs the L2 transaction on the user's
behalf, and bills the user's L1 energy account through the bridge.

## Architecture

```
                    L1 (Atoshi Cosmos chain)
        ┌─────────────────────────────────────────────┐
        │ x/energy                                     │
        │   ↑ user delegates energy → relayer L1 addr  │
        │   ↑ relayer's DelegatedInUsable grows        │
        └─────────────────────────────────────────────┘
                              ↑
                              │ (bridge / Cosmos query)
                              │
                    L2 (Polygon zkEVM)
        ┌─────────────────────────────────────────────┐
        │ EnergySettlement.sol                         │
        │   • acceptSignedTx(user_sig, target, calldata)│
        │   • Verifies user_sig against the calldata    │
        │   • Forwards to Shield (or any target)        │
        │   • Records gas used per user (off-chain        │
        │     reconciled against L1 energy delegation)  │
        ├─────────────────────────────────────────────┤
        │ Shield.sol (target)                          │
        │   • deposit / transfer / withdraw            │
        │   • Doesn't care that caller is the relayer  │
        └─────────────────────────────────────────────┘
                              ↑
                              │ tx submitted by relayer EOA
                              │ (relayer pays L2 gas in wATOS)
                              │
                          Off-chain relayer service
                          (foundation-operated in v1)
```

Three actors:

1. **User** — has L1 energy capacity, wants to interact with L2 Shield without paying L2 gas
2. **Relayer** — operates an L2 EOA with wATOS for gas; foundation-run in v1
3. **EnergySettlement contract** — verifies user authorization, executes the call

The economic model: user delegates L1 energy to the relayer's L1 address.
The relayer then "spends" that energy (in accounting terms) for every L2
transaction they sponsor. The wATOS gas they actually pay is reimbursed
later through a settlement process described below.

## End-to-end deposit flow

A user shielding 50 wATOS:

```
Step 1 — User: delegate L1 energy to the relayer (one-time setup)
  L1 tx: MsgDelegateEnergy {
    delegator: user_atoshi1...,
    delegatee: relayer_atoshi1...,
    amount:    1_000_000,      // energy units
    duration:  7 * 24 * 3600   // 7 days
  }
  Effect on L1:
    user.DelegatedOut          += 1,000,000
    user.LockedAtos            += 600,000 ATOS (collateral)
    relayer.DelegatedInUsable  += 1,000,000

Step 2 — User: prepare an L2 deposit
  Compute commitment = Poseidon(amount, owner_pubkey, secret)
  Sign EIP-712 typed data:
    {
      target:    Shield_address,
      calldata:  abi.encode(deposit(commitment, wATOS, 50))
      nonce:     user_l2_nonce,
      deadline:  now + 5 min
    }
  Signature is from the user's L2 private key.

Step 3 — User: POST to relayer's HTTP API
  POST https://relayer.atoshi.org/sponsor
  body: { typedData, signature, l1_address (for billing) }

Step 4 — Relayer: validate + submit
  Verifies:
    - signature recovers to user's L2 address
    - L1 address has DelegatedInUsable > estimated_gas
    - target is in the allow-list (Shield, etc.)
    - deadline not expired
  Submits to L2:
    EnergySettlement.acceptSignedTx(
      typedData, signature, gas_estimate
    )

Step 5 — EnergySettlement contract: execute
  Re-verifies signature on-chain (cheap, EIP-712).
  Calls target.call(calldata).
  Records event: SponsoredCall {user, l1_address, gas_used, target}.
  Returns success/failure to relayer.

Step 6 — Off-chain settlement (batch, every few hours)
  Relayer accumulates {l1_address, gas_used} pairs.
  Submits a single L1 tx debiting each user's energy:
    For each user: Consume(user_l1, gas_used) via a special precompile
    or via a Cosmos message.
  Relayer recovers their wATOS spend from the protocol treasury.
```

The user signs one L1 tx (the delegation) and one L2 EIP-712 message per
sponsored action. No L1 tx per L2 action. The relayer pays L2 gas and
gets reimbursed through L1 energy accounting.

## Why this preserves privacy

Three properties are critical:

### Privacy property 1: relayer ≠ user

The `tx.origin` on L2 for the sponsored call is the relayer's EOA. The
`msg.sender` into Shield is the EnergySettlement contract. **The user's
L2 EOA never appears on-chain for this transaction.**

Block explorers see "the relayer deposited" or "the EnergySettlement
contract deposited", not "user X deposited". The commitment in
Shield.sol is the same regardless of who called it.

### Privacy property 2: signature is private

The EIP-712 signature is included in the calldata to `acceptSignedTx`,
but the contract:

- Recovers the signer (the user's L2 address)
- Verifies the signer matches the typed-data's `from` field
- Records the recovered address ONLY in an internal mapping, NOT in events

The signature itself is plain calldata; an analyst CAN derive the user's
L2 address from it. But:

- The L2 address is one-time-use (wallets generate fresh addresses for
  privacy operations)
- The signature is only proof of authorization, not of the actual deposit
  amount or recipient (those are in the commitment)
- The link "L2 address → commitment" is the same link the user already
  publishes by signing a direct Shield call

So this property is "no worse than direct calling" — not a strict
improvement, but not a regression.

### Privacy property 3: L1 ↔ L2 unlinkability

The L1 delegation says "user X delegates to relayer Y" — public on L1.

The L2 sponsored call says "relayer Y sponsors some user's deposit" —
public on L2, but **does not reveal which L1 user**.

The L1 settlement batch at the end says "users [X, Z, W] consumed [a, b, c]
gas" — public on L1, **but does not reveal which L2 deposit corresponds
to which user**.

The analyst sees:
- L1: user X delegated 1M energy to relayer Y
- L2: relayer Y did 100 sponsored deposits this week
- L1: relayer Y settled 300 user accounts this week (X among them)

They cannot determine which of relayer Y's 100 deposits came from X.
The set of "user X's possible deposits" is the full sponsored-volume set
for that period — the anonymity set.

**The bigger the relayer's sponsored volume, the larger the anonymity
set.** A relayer doing 10 deposits/day gives weak privacy; 10,000/day
gives strong privacy. This is the fundamental property: privacy scales
with adoption.

## Settlement bookkeeping detail

The off-chain reconciliation step (Step 6 above) is the trust-sensitive
part. Different designs are possible:

### v1 design (centralized relayer + ZK receipt)

The relayer is foundation-operated and trusted to bill users honestly.
For each sponsored call, EnergySettlement emits an indexed event that
the relayer aggregates. Once a day:

1. Relayer publishes a Merkle root of all `(user, gas)` pairs to an L1
   tx
2. Anyone can verify the root by replaying the L2 events
3. Relayer triggers L1 energy debits based on the root

If the relayer cheats (over-bills or invents users), users can challenge
on L1 by submitting a contradicting Merkle proof.

### v2 design (proven on-chain settlement)

A ZK proof per settlement batch proves:

- All `(user, gas)` pairs in the batch correspond to real
  EnergySettlement events on L2
- Sum of gas matches the relayer's L2 wATOS expenditure (verifiable)
- Each user signed the corresponding transaction

This removes the trust assumption on the relayer but adds proving cost.
v2 is in design; v1 ships first to validate the UX.

### Out-of-bounds: gas overage

If a sponsored call uses MORE gas than estimated (e.g. due to network
congestion changing base fee, or the user's call reverting after partial
execution), the relayer eats the difference. The on-chain
EnergySettlement contract enforces a max-gas cap per call to bound this.

If a sponsored call uses LESS gas (common — overestimation is the safe
default), the unused portion is refunded to the relayer in the
settlement, not to the user.

## Economic model for the relayer

The relayer's PnL:

```
+  Sum of L1 energy debited from users (converted to "what would they have paid in ATOS gas")
−  Sum of L2 wATOS spent on gas
−  Operational costs (server, monitoring, multisig overhead)
=  Net margin
```

Since L1 energy is essentially free for holders (they're "spending"
something they're going to regenerate), the user-side cost is the
delegation collateral lock — not the energy itself. For the relayer:

- Energy debits come from delegations, which are bounded by user
  willingness to lock collateral
- L2 wATOS gas is bounded by Polygon zkEVM's actual gas market

The relayer breaks even if L1 energy → L2 wATOS exchange ratio is
neutral. The protocol can subsidize the relayer (e.g. via the project
treasury) to keep the service running during low-volume periods.

In v1, foundation operates the relayer and treats it as a marketing
expense — privacy-as-a-service for committed holders.

## API surface

### User-facing endpoints (HTTP, off-chain)

```
POST /sponsor
  body: {
    typed_data:    EIP712TypedData,
    signature:     bytes,
    l1_address:    atoshi1...
  }
  response: {
    l2_tx_hash:    "0x...",
    estimated_gas: 280_000,
    status:        "submitted" | "rejected"
  }
```

Returns the L2 tx hash the relayer submitted. User can monitor
acceptance via standard L2 RPC.

```
GET /quota?l1_address=atoshi1...
  response: {
    delegated_in_usable:  uint64,
    estimated_calls_left: uint32,
    expires_at:           int64
  }
```

So wallets can show "you have X privacy actions left in this delegation
period."

### EnergySettlement contract API (L2 EVM)

```
function acceptSignedTx(
    bytes32 typedDataHash,
    bytes calldata signature,
    address target,
    bytes calldata payload,
    uint256 maxGas
) external returns (bool success, bytes memory result);
```

`target` must be in the allow-list (Shield, etc.). `maxGas` bounds the
sub-call gas; relayer doesn't pay more than this even if the call would
have used more. Returns the call result.

Emits:

```
event SponsoredCall(
    address indexed signer,    // user's L2 address
    address indexed target,
    uint256 gasUsed
);
```

Off-chain indexer reads `SponsoredCall` events, maps signer → user's L1
address via a registry, and builds the settlement batch.

## Operational considerations

### Allow-list management

EnergySettlement's `target` allow-list is governance-controlled. Initially
only `Shield`. Adding more targets (e.g. a privacy-preserving DEX) requires
a gov proposal. This is the censorship vector: governance can refuse to
add a target.

### Front-running risk

Sponsored deposits include a commitment hash. A malicious relayer could
in theory front-run a deposit, but the commitment doesn't reveal the
note contents — there's nothing to front-run economically. Same for
transfers and withdrawals.

### Replay protection

Each EIP-712 signature includes a nonce + deadline. The
EnergySettlement contract maintains a `(signer, nonce)` mapping to reject
replays.

### Relayer downtime

If the relayer service is down, users can fall back to direct calls
(paying L2 gas themselves). The Shield contract doesn't require
sponsorship; sponsorship is purely a gas-payer abstraction.

This is important: **the privacy system is not dependent on the
relayer's liveness for safety**. Only for the gasless UX.

## Comparison with similar designs

| System | Gas payer | Privacy property | Trust |
|---|---|---|---|
| Direct Shield call | User's L2 EOA | User publicly tied to deposit event | Trustless |
| Tornado Cash relayer | Third-party relayer (paid out of withdrawal) | Anonymity within Tornado anonymity set | Trustless (smart contract enforces) |
| Aztec | Aggregator | Strong; aggregated proof hides individual deposits | Trustless |
| **Atoshi (this) v1** | **Foundation relayer** | **Anonymity within sponsored set** | **Trust foundation, but L1 challenge available** |
| **Atoshi v2** | **ZK-proven relayer** | **Same as v1** | **Trustless** |

Atoshi's distinguishing factor is the **L1 energy reimbursement model**
— users "pay" for privacy operations with capacity they already hold,
not with separate token expenditure. This is the natural extension of
the L1 design philosophy.

## Status (testnet)

- EnergySettlement contract: deployed at
  `0xB515a4a438c168cf34F1ABEEa40a835a39af5625` on L2 testnet
- Relayer service: foundation-operated, partial integration
- Wallet UI integration: planned for Phase 2 (Q4 2026)

## Related

- [Privacy overview module](../modules/07-privacy.md) — Shield contract spec
- [Energy module](../modules/01-energy.md) — L1 energy delegation
- [Shield pool deep-dive](./02-shield-pool.md) — for the operations being sponsored

---

*Last reviewed: 2026-06-10*
