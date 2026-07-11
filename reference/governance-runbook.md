# Governance Runbook

An operator-facing, copy-paste guide to **submitting and passing on-chain proposals** on Atoshi L1: parameter changes, software upgrades, and the emergency switches. For the conceptual model of `x/gov` (thresholds, tally math, why it's designed this way) read the [Governance module chapter](../modules/09-governance.md); this page is the "how do I actually do it" companion.

> All addresses, home paths, and server names below are **placeholders**. Substitute your own. Never paste a real keyring passphrase into a script or a doc.

Conventions used in every snippet:

```bash
BIN=atoshid
NODE_HOME=/data/atoshi/node0          # your node's home dir
CHAINID=atoshi_88288-1                # testnet; mainnet is atoshi_88188-1
NODE=tcp://localhost:26657
KEYRING=file                          # use file (password) for anything real
COMMON="--home $NODE_HOME --node $NODE --chain-id $CHAINID --keyring-backend $KEYRING"
TXFLAGS="--gas 500000 --gas-prices 1000000000aatos -o json --yes"
```

---

## 1. What kind of proposal do you need?

| Goal | Proposal message | Needs new binary? |
|---|---|---|
| Change module parameters (gov periods, feemarket floor, energy params, …) | `MsgUpdateParams` for that module | No — takes effect the block after it passes |
| Ship code changes (bug fix, new feature, migration) | `MsgSoftwareUpgrade` | Yes — every node swaps the binary at the upgrade height |
| Cancel a scheduled upgrade | `MsgCancelUpgrade` | No |
| Move funds from the community pool | `MsgCommunityPoolSpend` | No |

Everything is submitted the same way: a **proposal JSON** carrying one or more `messages`, plus a `deposit`, submitted with `tx gov submit-proposal`.

The `authority` field inside every `MsgUpdateParams` / `MsgSoftwareUpgrade` must be the **gov module account** (the only address allowed to execute a passed proposal). Fetch it once:

```bash
AUTHORITY=$($BIN query auth module-account gov --home $NODE_HOME --node $NODE -o json \
  | jq -r '.account.value.address // .account.base_account.address')
echo "authority: $AUTHORITY"      # e.g. atoshi1......  (deterministic per chain)
```

---

## 2. The deposit mechanic (why it matters)

A proposal has two gates before voting: it must reach `min_deposit`, and it must do so within `max_deposit_period`.

- If your `submit-proposal` **`deposit` already meets `min_deposit`**, the proposal **skips the deposit period and goes straight to `VOTING_PERIOD`**. This is what you almost always want on testnet.
- If it's short, the proposal sits in `DEPOSIT_PERIOD` until others top it up (or it expires and the deposit is refunded/burned per params).

The deposit is anti-spam collateral: it is **refunded** if the proposal reaches quorum, and can be burned under certain conditions (`burn_vote_quorum` / `burn_proposal_deposit_prevote`). So set `deposit` **equal to `min_deposit`** to one-shot into voting:

```bash
# read the current minimum
$BIN query gov params --home $NODE_HOME --node $NODE | grep -A3 min_deposit
```

---

## 3. Lifecycle and timing

```
submit ──▶ (deposit ≥ min_deposit?) ──▶ VOTING_PERIOD ──▶ tally ──▶ PASSED ──▶ apply
   │              │ no                        │               │
   │              ▼                           │               ▼
   │         DEPOSIT_PERIOD                   │        (upgrade) plan written,
   │                                          │         node halts at height
   └─ deposit refunded on quorum              ▼
                                       quorum + threshold checks
```

Tally at `voting_end_time` passes when **all** hold:

- turnout ≥ `quorum` (default `0.334`)
- `yes / (yes+no+veto)` > `threshold` (default `0.5`)
- veto share < `veto_threshold` (default `0.334`)

**Timing math.** Total time ≈ `voting_period` (+ `deposit_period` only if under-funded). For a software upgrade, add the wait until `upgrade-height`.

| Scenario | Wall-clock to "ready to test" |
|---|---|
| Param change, `voting_period=60s`, submit+vote immediately | ~60–90 s |
| Software upgrade, `voting_period=60s`, `upgrade-height = now + ~1000 blocks` (~33 min at ~2 s/block) | ~35 min |
| Default `voting_period=48h` | ≥ 2 days |

On a fresh network the default `voting_period` is long (hours to days). The **first thing** you usually do on a test network is shorten it (§5) — but that first proposal itself must sit through the *current* (long) period. To avoid the 2-day wait entirely, you can instead `export` genesis, edit `app_state.gov.params`, and restart from the edited genesis (only viable when you control all validators / are resetting anyway).

---

## 4. Query the current gov parameters

```bash
$BIN query gov params --home $NODE_HOME --node $NODE
```

Typical output (defaults):

```yaml
voting_period: 172800s            # 2 days
expedited_voting_period: 86400s   # 1 day
max_deposit_period: 172800s
min_deposit:
  - denom: aatos
    amount: "10000000"            # this is the number to match in your `deposit`
quorum: "0.334"
threshold: "0.5"
veto_threshold: "0.334"
```

> `MsgUpdateParams` is a **full overwrite** — the message replaces the entire params struct. Always start from the current values and change only the fields you mean to; a missing field resets to zero/empty. Query first, edit, then submit.

---

## 5. Worked example — shorten governance periods (param change)

Goal: cut `voting_period` / deposit period down so QA can iterate. This is the canonical `MsgUpdateParams` flow.

### 5.1 Write the proposal JSON

```bash
cat > /tmp/shorten-gov.json <<JSON
{
  "messages": [
    {
      "@type": "/cosmos.gov.v1.MsgUpdateParams",
      "authority": "$AUTHORITY",
      "params": {
        "min_deposit":               [{"denom": "aatos", "amount": "10000000"}],
        "max_deposit_period":        "900s",
        "voting_period":             "1800s",
        "quorum":                    "0.334000000000000000",
        "threshold":                 "0.500000000000000000",
        "veto_threshold":            "0.334000000000000000",
        "min_initial_deposit_ratio": "0.000000000000000000",
        "proposal_cancel_ratio":     "0.500000000000000000",
        "proposal_cancel_dest":      "",
        "expedited_voting_period":   "900s",
        "expedited_threshold":       "0.667000000000000000",
        "expedited_min_deposit":     [{"denom": "aatos", "amount": "5000000"}],
        "burn_vote_quorum":          false,
        "burn_proposal_deposit_prevote": false,
        "burn_vote_veto":            true,
        "min_deposit_ratio":         "0.010000000000000000"
      }
    }
  ],
  "metadata": "shorten gov periods for QA",
  "deposit": "10000000aatos",
  "title": "Shorten governance periods (testnet)",
  "summary": "Reduce voting_period / deposit period for faster QA cycles."
}
JSON
```

The `deposit` (`10000000aatos`) equals `min_deposit`, so the proposal enters voting immediately.

### 5.2 Submit

```bash
$BIN tx gov submit-proposal /tmp/shorten-gov.json --from <proposer-key> $COMMON $TXFLAGS | tee /tmp/submit.json
sleep 5   # wait for inclusion
```

> The submit response often shows `"code":0` with `"height":"0"` when using `--broadcast-mode sync` — that only means "accepted into mempool", **not** "executed". Always confirm with `query tx <hash>` (see §5.6).

### 5.3 Get the proposal id

```bash
PID=$($BIN query gov proposals --reverse --limit 1 --home $NODE_HOME --node $NODE -o json | jq -r '.proposals[0].id')
echo "proposal: $PID"
```

### 5.4 Vote — one validator

```bash
$BIN tx gov vote $PID yes --from <validator-key> $COMMON $TXFLAGS
```

### 5.5 Vote — multi-validator network

Every validator votes from **its own home dir and key**; quorum is by **stake weight**, so you need enough voting validators to clear `quorum`. Run on each validator machine:

```bash
# node0
$BIN tx gov vote $PID yes --from val0 --home /data/atoshi/node0 --node $NODE --chain-id $CHAINID --keyring-backend file $TXFLAGS
# node1
$BIN tx gov vote $PID yes --from val1 --home /data/atoshi/node1 --node $NODE --chain-id $CHAINID --keyring-backend file $TXFLAGS
# … one per validator
```

### 5.6 Confirm votes actually landed

Because `code:0` at submit only means "accepted", verify each vote tx committed:

```bash
$BIN query tx <vote-tx-hash> --node $NODE -o json | jq '{height, code, raw_log}'
# want: code == 0, a real height, empty raw_log
```

Check a recorded vote and the live tally:

```bash
$BIN query gov vote $PID <voter-address> --node $NODE
$BIN query gov tally $PID --node $NODE
# yes_count should ≈ the total stake of the validators who voted yes
```

### 5.7 After voting ends, confirm it passed and applied

```bash
$BIN query gov proposal $PID --node $NODE -o json | jq '{status, failed_reason, final_tally_result}'
# want: status == PROPOSAL_STATUS_PASSED, failed_reason empty

$BIN query gov params --node $NODE | grep -E "voting_period|max_deposit_period|quorum"
# want: the new values (e.g. voting_period: 1800s)
```

### 5.8 One-shot script

```bash
cat > /tmp/run-gov.sh <<'BASH'
set -e
BIN=atoshid
HF="--home /data/atoshi/node0"; NF="--node tcp://localhost:26657"
CF="--chain-id atoshi_88288-1"; KF="--keyring-backend file"
TXF="-o json --yes --gas 500000 --gas-prices 1000000000aatos"

AUTH=$($BIN query auth module-account gov $HF $NF -o json | jq -r '.account.value.address // .account.base_account.address')
echo "authority: $AUTH"

sed "s|__AUTH__|$AUTH|g" > /tmp/shorten-gov.json <<JSON
{ "messages":[{ "@type":"/cosmos.gov.v1.MsgUpdateParams","authority":"__AUTH__",
  "params":{
    "min_deposit":[{"denom":"aatos","amount":"10000000"}],
    "max_deposit_period":"900s","voting_period":"60s",
    "quorum":"0.334000000000000000","threshold":"0.500000000000000000",
    "veto_threshold":"0.334000000000000000","min_initial_deposit_ratio":"0.000000000000000000",
    "proposal_cancel_ratio":"0.500000000000000000","proposal_cancel_dest":"",
    "expedited_voting_period":"30s","expedited_threshold":"0.667000000000000000",
    "expedited_min_deposit":[{"denom":"aatos","amount":"5000000"}],
    "burn_vote_quorum":false,"burn_proposal_deposit_prevote":false,
    "burn_vote_veto":true,"min_deposit_ratio":"0.010000000000000000" } }],
  "metadata":"shorten gov for QA","deposit":"10000000aatos",
  "title":"Shorten governance periods (testnet)","summary":"Faster QA cycles." }
JSON

$BIN tx gov submit-proposal /tmp/shorten-gov.json --from <proposer-key> $HF $NF $CF $KF $TXF
sleep 5
PID=$($BIN query gov proposals --reverse --limit 1 $HF $NF -o json | jq -r '.proposals[0].id'); echo "PID=$PID"
$BIN tx gov vote $PID yes --from <validator-key> $HF $NF $CF $KF $TXF
sleep 75
$BIN query gov proposal $PID $HF $NF -o json | jq '{status, final_tally_result}'
$BIN query gov params $HF $NF | grep -E "voting_period|max_deposit_period|quorum"
BASH
bash /tmp/run-gov.sh
# expect: status: PROPOSAL_STATUS_PASSED and voting_period: 60s
```

---

## 6. Worked example — software upgrade (`MsgSoftwareUpgrade`)

Use this when the change is **code**, not just params (e.g. a keeper bug fix plus a one-time state migration). The whole validator set halts at a chosen height and restarts on the new binary, which runs the upgrade handler.

> A software upgrade is **mandatory-coordinate**: every validator must swap the binary at the same height or the chain stalls / forks. Announce the height and binary hash to all validators before submitting.

### 6.1 Prepare the binary and pick a height

```bash
# 1. current height
$BIN status --home $NODE_HOME | jq -r '.SyncInfo.latest_block_height'
# 2. choose an upgrade height with buffer (e.g. current + ~1000 blocks ≈ 30 min)
# 3. build the new binary and distribute to EVERY node
cd atoshi-chain && make build      # -> ./build/atoshid  (must report the new version)
scp build/atoshid <user>@<server>:/opt/atoshi/upgrades/<version>/atoshid
```

### 6.2 Submit the upgrade proposal

Named upgrades map to an `UpgradeName` registered in `app/upgrades/<version>/`. The name in the proposal **must exactly match** the registered name or the handler won't run.

```bash
$BIN tx upgrade software-upgrade <version> \
  --title "<version>: <short description>" \
  --summary "<what it changes and why>" \
  --upgrade-height <chosen-height> \
  --upgrade-info '{"binaries":{"linux/amd64":"file:///opt/atoshi/upgrades/<version>/atoshid"}}' \
  --deposit 10000000aatos \
  --no-validate \
  --from <proposer-key> $COMMON $TXFLAGS
```

Then vote and confirm exactly as in §5.3–5.7. After it passes, the plan is written to the upgrade keeper and the chain will halt at `<chosen-height>`.

### 6.3 Apply at the upgrade height

At the height, every node **panics on purpose**:

```
ERR UPGRADE "<version>" NEEDED at height: <chosen-height>
```

**Option A — cosmovisor (recommended).** Pre-stage the binary; cosmovisor swaps and restarts automatically:

```bash
mkdir -p $NODE_HOME/cosmovisor/upgrades/<version>/bin
cp build/atoshid $NODE_HOME/cosmovisor/upgrades/<version>/bin/
# cosmovisor detects the on-chain plan and switches at the height — no manual stop/start
```

**Option B — manual.** Swap and restart on every node:

```bash
systemctl stop atoshid
cp /opt/atoshi/upgrades/<version>/atoshid /usr/local/bin/atoshid
systemctl start atoshid
```

On restart the node runs the upgrade handler. For a state-migration upgrade you'll see the handler's log line (e.g. an "energy snapshot refresh complete, accounts_refreshed=N" style message for the energy snapshot migration that shipped in v20.1 — see [Governance module, Case study 3](../modules/09-governance.md#case-study-3--v201-software-upgrade-passed)).

### 6.4 Verify the fix landed

Query whatever the upgrade was supposed to change. For the v20.1 energy-snapshot migration, for example:

```bash
$BIN query energy account <affected-account> --home $NODE_HOME
# last_balance_snapshot should equal the account's real current bank balance,
# and tx_energy_capacity should equal floor(balance / 30000) * 50000
```

---

## 7. Worked example — feemarket minimum gas price (param change)

**Problem.** `x/feemarket` ships with `DefaultMinGasPrice = 0`. The EIP-1559 base-fee controller lets the base fee decay toward `MinGasPrice` when blocks are empty — with a floor of 0, base fee can fall all the way to zero on a quiet network, so gas effectively becomes free and spam becomes cheap.

**Fix.** Raise the floor to a small non-zero value. A reasonable floor is **1 gwei = 1,000,000,000 aatos/gas** — a 21,000-gas transfer then costs `21000 × 1e9 = 2.1e13 aatos = 0.000021 ATOS`: negligible for users, but no longer zero.

This is independent of the energy module's `InsufficientGasPrice` (`0.0021 aatos/gas`, the shortfall price when energy runs out) — different knob, different module.

```bash
# 1. read current feemarket params — you must resupply every field (full overwrite)
$BIN query feemarket params --home $NODE_HOME --node $NODE

# 2. write the proposal, changing only min_gas_price (and base_fee floor if desired)
cat > /tmp/feemarket-mingas.json <<JSON
{
  "messages": [
    {
      "@type": "/ethermint.feemarket.v1.MsgUpdateParams",
      "authority": "$AUTHORITY",
      "params": {
        "no_base_fee": false,
        "base_fee_change_denominator": 8,
        "elasticity_multiplier": 2,
        "enable_height": "0",
        "base_fee": "1000000000",
        "min_gas_price": "1000000000.000000000000000000",
        "min_gas_multiplier": "0.500000000000000000"
      }
    }
  ],
  "metadata": "",
  "deposit": "10000000aatos",
  "title": "Set feemarket MinGasPrice to 1 gwei",
  "summary": "Prevent base fee from decaying to 0 under low traffic. Floor = 1 gwei (~0.000021 ATOS per 21k-gas transfer)."
}
JSON

# 3. submit + vote + confirm (as in §5)
$BIN tx gov submit-proposal /tmp/feemarket-mingas.json --from <proposer-key> $COMMON $TXFLAGS
```

Double-check the non-changed fields against step 1's output before submitting — a wrong `base_fee_change_denominator` or `elasticity_multiplier` silently changes fee dynamics for the whole chain.

---

## 8. Decision note — disabling the energy module (`EnergyEnabled = false`)

`energy_enabled` is a param on `x/energy`, flippable with a `MsgUpdateParams` proposal. It is a **global kill switch**. Before proposing it, understand exactly what changes — this was analyzed against the code:

**What happens when it's off:**

1. **Ante handler** — `Consume` returns the full gas as `ShortfallGas`, so every tx pays gas in ATOS at the shortfall price (or the EIP-1559 fee). Users with enough ATOS succeed; users without fail — i.e. behaviour identical to a chain that never had energy.
2. **Transfers** (`MsgSend` / EVM) — unaffected functionally; they never cost energy anyway. They just get billed in ATOS gas, which is more expensive for the user than the energy path.
3. **Snapshots keep updating** — `SendRestriction` → `ApplyBalanceChange` does **not** check `energy_enabled`, so per-account snapshots keep accruing while the module is off. Turning it back on resumes cleanly with no data loss. Disabling is "stop using", not "destroy state".
4. **Subsidized messages stay free** — the whitelist check runs *before* the `energy_enabled` check, so oracle price reports, energy delegate/undelegate, and tokenomics claims remain free even while off.
5. **Queries** — `Account` still returns (accruing) data; `Params` shows `energy_enabled: false`; `EstimateFee` returns `shortfall_gas = gas_limit` (tells the wallet "bill the whole thing as ATOS gas").
6. **EndBlocker** — delegation expiry sweeping stops while off. Any delegation that "expires" during the off-window is not released until the module is re-enabled, at which point the backlog is swept.

**Wallet must adapt.** If the module is disabled without the wallet knowing, it will keep showing stale energy UI ("energy: 0" plus "insufficient energy"). Wallets should poll `Params` and branch:

```
if (!params.energy_enabled) {
    // hide energy balance / "my energy" entry
    // hide "delegate energy" / "reclaim energy" buttons
    // on the send-confirm screen, show the ATOS gas fee, not "energy consumed: N"
    // for fee estimation, use eth_estimateGas instead of energy EstimateFee
}
```

**Recommendation.** **Do not disable it during normal testnet operation.** Keep it as a mainnet emergency-stop only, and if you ever must flip it, notify the wallet team first so they ship the `energy_enabled` UI branch. Disabling is safe for data but strictly worse UX (users pay gas instead of spending energy).

---

## 9. Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Proposal stuck in `DEPOSIT_PERIOD` | `deposit` < `min_deposit` | Resubmit with `deposit` = `min_deposit`, or top up with `tx gov deposit` |
| `PROPOSAL_STATUS_REJECTED` despite yes votes | Quorum not met — not enough stake voted | Ensure enough validators (by weight) vote before `voting_end_time` |
| Upgrade height reached, chain halts, never resumes | Binary not swapped, or `UpgradeName` mismatch, or wrong binary | Every node must run the matching new binary; the proposal name must equal the registered `UpgradeName` |
| `MsgUpdateParams` passed but other params changed unexpectedly | Full-overwrite semantics — a field was omitted | Always start from `query … params` and edit; never hand-write a partial params block |
| Submit shows `code:0` but nothing happened | `code:0` = accepted to mempool, not executed | Verify with `query tx <hash>` for a real height + `code:0` |

---

## Related

- [Governance module](../modules/09-governance.md) — thresholds, tally math, real case studies
- [Node operation](./node-operation.md) — building the binary, running validators
- [Energy module](../modules/01-energy.md) — what `energy_enabled` and the energy params control
- [Gas and energy economics](../economics/03-gas-fee.md) — fee curves and parameter sensitivity

---

*Last reviewed: 2026-07-11*
