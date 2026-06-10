# `x/gov` — on-chain governance

The standard Cosmos SDK gov module is the only authority capable of mutating
chain parameters and triggering coordinated software upgrades. This chapter
covers how Atoshi configures it, the lifecycle of a proposal, and three real
proposals that have already passed through the testnet — including a failure
case worth studying.

## Mental model

Anything that says "governance-tunable" elsewhere in these docs (energy
parameters, tokenomics tier thresholds, oracle feeders, gov itself) means
"only changeable via a `MsgUpdateParams` or equivalent inside a passed gov
proposal." There is no other path; foundation multisigs cannot directly write
to params.

A proposal has two periods stacked on top of each other:

```
submit ──▶ DEPOSIT_PERIOD ──▶ VOTING_PERIOD ──▶ PASSED / REJECTED / FAILED
            (top up min_deposit)  (validators vote)   (apply or refund)
```

If `initial_deposit ≥ min_deposit` at submit time, the deposit period is
skipped and the proposal goes directly to voting.

## Current parameters (post-proposal-2)

After the second gov-shortening proposal landed, parameters are:

| Param | Value | Notes |
|---|---|---|
| `min_deposit` | 10,000,000 aatos (0.01 ATOS) | Sponsor must lock this much to enter voting |
| `expedited_min_deposit` | 50,000,000 aatos (0.05 ATOS) | Strictly > `min_deposit` (chain invariant) |
| `voting_period` | **30 minutes** | Reduced from 48 hours for QA velocity |
| `expedited_voting_period` | 15 minutes | For urgent fixes |
| `max_deposit_period` | 15 minutes | How long deposit can be topped up |
| `quorum` | 33.4% | Of bonded stake, must participate |
| `threshold` | 50% | `yes / (yes + no + no_with_veto)` |
| `veto_threshold` | 33.4% | `no_with_veto / total` ≥ this → REJECTED + burn |
| `burn_vote_veto` | true | Deposit burned on veto rejection |
| `burn_vote_quorum` | false | Deposit refunded on quorum failure |
| `burn_proposal_deposit_prevote` | false | Deposit refunded on pre-voting cancel |

Mainnet will revert `voting_period` to a more sober value (typically 48–72
hours). The 30-minute setting is testnet-only — it lets QA cycle through
parameter changes in a working day instead of a working week.

## Lifecycle

### Submit

```
MsgSubmitProposal {
  messages: [<inner Msg, e.g. MsgUpdateParams>]
  initial_deposit: [{denom, amount}]
  proposer: bech32
  metadata: string
  title: string
  summary: string
}
```

The chain validates the inner message at submit time (signers, types) but
does NOT validate the proposed parameters themselves. Invalid params will
pass voting and only fail on apply — see Proposal 1 below.

### Deposit

If `initial_deposit < min_deposit` the proposal sits in `DEPOSIT_PERIOD`. Any
account can call `MsgDeposit{proposal_id, amount}` to top it up. Once total
reaches `min_deposit`, the proposal transitions to voting automatically.

If `deposit_end_time` passes without the threshold being met, the proposal
is dropped and any partial deposits are refunded (or burned, depending on
`burn_proposal_deposit_prevote`).

### Vote

Validators (and delegators who haven't delegated their vote) cast
`MsgVote{proposal_id, option}` where option is `yes / no / abstain / no_with_veto`.
Voters can change their vote any time before `voting_end_time` — the latest
vote wins.

Tally happens at `voting_end_time`:

```
1. participation = (yes + no + abstain + no_with_veto) / total_bonded_stake
   if participation < quorum:
       → REJECTED, deposit refund/burn per burn_vote_quorum
       exit

2. veto_ratio = no_with_veto / (yes + no + abstain + no_with_veto)
   if veto_ratio ≥ veto_threshold:
       → REJECTED, deposit BURNED (burn_vote_veto = true)
       exit

3. yes_ratio = yes / (yes + no + no_with_veto)   ← abstain excluded
   if yes_ratio ≥ threshold:
       → PASSED, deposit refunded, apply inner messages
   else:
       → REJECTED, deposit refunded
```

### Apply

For a PASSED proposal, the chain executes each inner message as if it were
sent by the gov module account itself. If execution fails (invalid params,
unrelated error), the proposal transitions to `PROPOSAL_STATUS_FAILED` and
`failed_reason` is set. The deposit is still refunded.

## Case study 1 — `Shorten governance periods` (FAILED)

The first proposal submitted on Atoshi testnet. Intent was to reduce
`voting_period` from 48h to 30 minutes for QA velocity. Submitted with these
parameters:

```json
{
  "voting_period":          "1800s",
  "max_deposit_period":     "900s",
  "expedited_voting_period": "900s",
  "expedited_min_deposit":  [{"denom": "aatos", "amount": "5000000"}],
  "min_deposit":            [{"denom": "aatos", "amount": "10000000"}],
  ...
}
```

**Outcome**: voting passed unanimously (40M ATOS yes, single validator).
Then on apply:

```yaml
status: PROPOSAL_STATUS_FAILED
failed_reason: "expedited minimum deposit must be greater than minimum deposit: 5000000aatos"
```

### Root cause

Cosmos SDK gov enforces an invariant: `expedited_min_deposit > min_deposit`.
The intuition is that "expedited" should be more expensive than a normal
proposal because it's a fast-track path that bypasses the normal cooling
period.

This proposal set them backwards (5M < 10M), violating the invariant.

### What the chain did right

- Validation runs at apply time, not at submit time. The proposal entered
  voting, validators saw the params, voted yes anyway. The chain still
  refused to apply broken parameters. **Voters cannot accidentally brick
  the chain via gov.**
- The deposit was refunded automatically (FAILED status doesn't burn unless
  `burn_vote_veto` was also triggered, which requires actual veto votes).

### Lesson

**Always run params through `Validate()` locally before submitting a
proposal**, especially for chains that bundle many invariants (energy,
tokenomics, gov all have their own). A quick way:

```bash
atoshid simulate tx ...   # validates without broadcasting
```

A list of common invariants is in [`reference/faq.md`](../reference/faq.md).

## Case study 2 — `Shorten governance periods v2` (PASSED)

Resubmitted with the same payload but `expedited_min_deposit` raised to
`50,000,000 aatos` (5× `min_deposit`).

```json
{
  "expedited_min_deposit": [{"denom": "aatos", "amount": "50000000"}],
  ...
}
```

**Outcome**:

```yaml
status: PROPOSAL_STATUS_PASSED
final_tally_result:
  yes_count: 40000000000000000000000000   # 40M ATOS, 100% of stake
```

`gov.params.voting_period` immediately changed from `48h0m0s` to `30m0s`.
All subsequent proposals (including the v20.1 upgrade) used the 30-minute
window.

### Lesson

After fixing the invariant, retry is the same proposal mechanically. The
chain has no concept of "amending a failed proposal" — each submission is a
new, independent flow with a new proposal id. This is intentional:
amendments would invalidate prior votes.

## Case study 3 — `v20.1` software upgrade (PASSED)

The most operationally complex proposal type. Triggers a coordinated halt-
and-swap of every validator's binary.

```json
{
  "@type": "/cosmos.upgrade.v1beta1.MsgSoftwareUpgrade",
  "authority": "atoshi10d07y265gmmuvt4z0w9aw880jnsr700jxjttqu",
  "plan": {
    "name": "v20.1",
    "height": "1001100",
    "info": ""
  }
}
```

### Why this kind of upgrade is mandatory-coordinate

The fix was wiring `energy.SendRestriction` into bank — a consensus-affecting
state transition. If validator A runs new binary at block N while validator B
runs old binary, they will compute different `MsgSend` state changes,
diverge, and the chain forks.

The SDK enforces this with a built-in guard. When a binary with the v20.1
handler tries to run on a chain where the v20.1 plan is set but not yet
triggered, the chain refuses to proceed:

```
ERR BINARY UPDATED BEFORE TRIGGER! UPGRADE "v20.1" -
    in binary but not executed on chain. Downgrade your binary
```

This is by design — Cosmos forces operators to run the old binary right up to
the upgrade height. The complete sequence is:

```
1. proposal submitted, voted, PASSED
2. SetUpgradePlan(name="v20.1", height=1001100) recorded on-chain
3. Old binary continues producing blocks until 1001099
4. At height 1001100, old binary's BeginBlocker checks:
   "Do I have a handler for v20.1?"  → NO
   "Is plan.height reached?"        → YES
   → PANIC with "UPGRADE NEEDED" message, exit non-zero
5. Operators stop old binary, swap in new binary (which DOES have v20.1 handler), restart
6. New binary's BeginBlocker checks:
   "Plan v20.1 reached?"            → YES
   "Do I have a handler?"           → YES
   → Run handler (RefreshAllSnapshots, etc.), continue producing blocks
```

### What our v20.1 handler does

From [`app/upgrades/v20_1/upgrades.go`](https://github.com/atoshi-chain/atoshi/blob/main/app/upgrades/v20_1/upgrades.go):

1. Run standard module migrations (no-op for this release, no consensus
   versions bumped).
2. Call `energy.RefreshAllSnapshots(ctx)`:
   - Iterate every stored `EnergyAccount` in the energy module store
   - For each, reset `LastBalanceSnapshot` to the current bank balance
   - Cap `TxEnergyAccrued` down if it exceeds the new capacity
3. Log the count of refreshed accounts and return.

The fix was paired with a runtime code change: bank's send-restriction hook
is now wired to call `energy.OnBalanceChange` on every outbound transfer, so
future MsgSends keep snapshots in sync without needing another upgrade.

### Operational lessons learned

| Pitfall encountered | Lesson |
|---|---|
| First simulation attempt failed: `target_height + 20` was too tight; voting period (~600 blocks) consumed all the buffer | Use ≥ `voting_period_blocks + 1200 buffer` (~1 hour cushion for validator coordination) |
| First broadcast failed: `--fees 5000aatos` was below `min_gas_price` (1 gwei post-feemarket-update) | After any feemarket proposal, recompute fees using `gas × gas_price` |
| Single-validator simulation kept failing with `BINARY UPDATED BEFORE TRIGGER` | This error is correct SDK behavior — single-binary simulation of upgrades is intentionally impossible. Test the migration handler with a Go unit test, not a devnet rehearsal |
| Chain panicked with `nil pointer dereference at app/app.go:982` after the upgrade fired | This was a secondary panic masking the real `UPGRADE NEEDED` message — added a nil guard in `FinalizeBlock`'s defer to preserve real panic stacks |

The QA simulation script (`scripts/upgrade_devnet.sh`) was built during this
process and is useful for any future upgrade — it covers proposal syntax,
voting, plan scheduling. The handler execution itself can only be tested with
two binaries or a Go unit test.

## How to submit your own proposal

### Step 1: write JSON

```bash
cat > /tmp/myprop.json <<EOF
{
  "messages": [
    {
      "@type": "/<message type url>",
      "authority": "atoshi10d07y265gmmuvt4z0w9aw880jnsr700jxjttqu",
      ...
    }
  ],
  "metadata": "<short label>",
  "deposit": "10000000aatos",
  "title": "<short title>",
  "summary": "<longer description of intent and rationale>"
}
EOF
```

`metadata` is a free-form pointer (typically to a discussion thread on
GitHub or Commonwealth); it is NOT IPFS-validated by the chain.

### Step 2: simulate locally (recommended)

```bash
atoshid tx gov submit-proposal /tmp/myprop.json \
  --from validator0 --chain-id atoshi_88288-1 \
  --gas-prices 1000000000aatos --dry-run
```

This catches signature errors, fee miscalculation, and some param
validation. It does NOT catch invariants that only fail on apply.

### Step 3: submit

```bash
atoshid tx gov submit-proposal /tmp/myprop.json \
  --from validator0 --chain-id atoshi_88288-1 \
  --keyring-backend file --gas 500000 \
  --gas-prices 1000000000aatos -o json --yes \
  | tee /tmp/submit.json
```

Verify the tx hash actually landed:

```bash
sleep 5
atoshid query tx $(jq -r .txhash /tmp/submit.json) -o json | jq '{height, code, raw_log}'
```

### Step 4: vote

Wait for `PROPOSAL_STATUS_VOTING_PERIOD`, then:

```bash
PID=$(atoshid query gov proposals -o json | jq -r '[.proposals[].id | tonumber] | max | tostring')
atoshid tx gov vote $PID yes --from validator0 \
  --chain-id atoshi_88288-1 --keyring-backend file \
  --gas 200000 --gas-prices 1000000000aatos -o json --yes
```

### Step 5: monitor

```bash
# Real-time tally during voting
atoshid query gov tally $PID

# Final result after voting_end_time
atoshid query gov proposal $PID -o json \
  | jq '.proposal | {status, failed_reason, final_tally_result}'
```

A `PROPOSAL_STATUS_PASSED` with empty `failed_reason` means success.

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `code: 13 insufficient fee` | `--fees / --gas-prices` below `feemarket.MinGasPrice` | Use `--gas-prices 1000000000aatos` |
| `status: PROPOSAL_STATUS_FAILED` after vote pass | Invariant violation in inner message | Check `failed_reason`, fix params, resubmit |
| `status: PROPOSAL_STATUS_REJECTED` with all zero tallies | quorum not reached | Increase validator participation |
| Chain halts on software upgrade | Validators didn't swap binary in time | Each validator: stop old, swap, restart |
| `tx not found` | Submission was rejected at mempool; tx never on chain | Re-check `raw_log` from broadcast response |

## Related

- [`x/energy`](./01-energy.md) — parameters changeable via gov
- [`x/tokenomics`](./02-tokenomics.md) — release schedule changeable via gov
- [`x/oracle`](./03-oracle.md) — feeder allow-list changeable via gov
- [Cosmos SDK gov spec](https://docs.cosmos.network/v0.50/build/modules/gov)

---

*Last reviewed: 2026-06-10*
