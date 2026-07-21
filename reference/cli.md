# CLI reference (`atoshid`)

Complete command reference for the `atoshid` binary — the Atoshi L1 node and
client. Organized by area: [global conventions](#global-conventions), node &
admin, key management, then `tx` / `query` commands per module (Atoshi custom
modules first, then standard Cosmos SDK modules).

This page is a reference; for guided walkthroughs see [Node operation](./node-operation.md)
(run a node / validator), the [Governance runbook](./governance-runbook.md)
(proposals), and the [Energy module](../modules/01-energy.md).

> `atoshid` is a Cosmos SDK + Ethermint binary. Every `tx` broadcasts a Cosmos
> transaction; EVM interaction (MetaMask / web3.js) goes through the separate
> EVM JSON-RPC endpoint, not this CLI. Run any command with `--help` for the
> authoritative, version-matched flag list.

## Global conventions

Flags that apply to almost every command:

| Flag | Meaning |
|---|---|
| `--home` | Node/keyring home dir (e.g. `/data/atoshi/node`) |
| `--chain-id` | `atoshi_88288-1` (testnet) / `atoshi_88188-1` (mainnet) |
| `--node` | CometBFT RPC to talk to (e.g. `tcp://localhost:26657`) |
| `--keyring-backend` | `file` (password) for real use; `test` for dev only |
| `--from` | Key name or address that signs a `tx` |
| `--gas` / `--gas-prices` / `--fees` | Gas limit and price, e.g. `--gas 500000 --gas-prices 1000000000aatos` |
| `--output` / `-o` | `text` (default) or `json` |
| `-y` / `--yes` | Skip the interactive confirm prompt |

Denom is `aatos` (1 ATOS = 10^18 aatos). Network minimum gas price is
`1000000000aatos` (1 gwei). Keys use the `eth_secp256k1` algorithm, so every
account has both a `0x…` (EVM) and an `atoshi1…` (bech32) address.

---

## Node lifecycle

The daemon binary is `atoshid` (root command `Use: "atoshi"`, `Short: "Atoshi Daemon"`). The default home is `~/.atoshid`; the default account address prefix is `atoshi` and the default key algorithm is `eth_secp256k1`. Chain IDs follow the `atoshi_88388-1` form.

### `atoshid init`

```bash
atoshid init MONIKER --chain-id atoshi_88388-1 [-o] [--home ~/.atoshid]
```

Initializes the private validator, p2p, genesis, and application config files under `--home`. Takes exactly one positional `MONIKER`.

Key flags:
- `--chain-id` — genesis chain-id; if blank a random `atoshi_88388-<rand>` is generated.
- `-o, --overwrite` — overwrite an existing `genesis.json`.
- `--recover` — prompt for a bip39 mnemonic to recover the validator key instead of generating a new one.
- `--default-denom` — default bond denom written to genesis (defaults to the chain base denom `aatos`).
- `--home` — node home directory.

### `atoshid start`

```bash
atoshid start --minimum-gas-prices=0.0001aatos --json-rpc.api eth,txpool,personal,net,debug,web3 \
  --pruning nothing --home ~/.atoshid --chain-id atoshi_88388-1
```

Runs the full node (CometBFT consensus + EVM JSON-RPC + Cosmos gRPC/REST).

Key flags:
- `--minimum-gas-prices` — minimum gas price the node accepts (e.g. `0.0001aatos`).
- `--json-rpc.api` — comma-separated EVM JSON-RPC namespaces to enable (`eth,txpool,personal,net,debug,web3`).
- `--pruning` — state pruning strategy: `default` | `nothing` (keep all state, best for dev/archive) | `everything` | `custom` (with `--pruning-keep-recent` / `--pruning-interval`).
- `--home` — node home directory.
- `--chain-id` — chain ID to run.
- Other common flags: `--metrics`, `--log_level`, `--trace`, `--halt-height`, `--json-rpc.address`, `--api.enable`, `--grpc.address`.

### `atoshid status`

```bash
atoshid status [--node tcp://localhost:26657]
```

Queries the connected node's status (node info, latest/earliest block height, catching-up flag, validator info).

### `atoshid version`

```bash
atoshid version [--long] [-o json]
```

Prints the binary version; `--long` adds build tags, Go version, and the full dependency list.

### `atoshid config`

```bash
atoshid config set client chain-id atoshi_88388-1 --home ~/.atoshid
atoshid config get client keyring-backend
```

Manages the client/app/config TOML files (backed by confix). Subcommands include `get`, `set`, `view`, `migrate`, and `diff`. As used in `local_node.sh`, `config set client chain-id` and `config set client keyring-backend` write the client defaults.

## Genesis & network setup

Genesis-building commands are registered at the top level (there is no `atoshid genesis` parent group in this build).

### `atoshid keys add | list | show | export | import | delete`

```bash
atoshid keys add validator --keyring-backend file --algo eth_secp256k1 [--recover]
atoshid keys list --keyring-backend file
atoshid keys show validator -a --keyring-backend file
atoshid keys export validator --keyring-backend file
atoshid keys import validator key.asc --keyring-backend file
atoshid keys delete validator --keyring-backend file
```

Manages keys in the keyring (Atoshi uses Ethereum-style keys).

Key flags:
- `--keyring-backend` — `os` | `file` | `kwallet` | `pass` | `test`. `local_node.sh` uses `file` (password-protected).
- `--algo` — signing algorithm; use `eth_secp256k1` for Ethereum-compatible (0x) addresses.
- `--recover` — import from an existing bip39 mnemonic (prompted on stdin).
- `keys show -a` prints the address only; `--bech acc|val|cons` selects the address form.

### `atoshid add-genesis-account`

```bash
atoshid add-genesis-account ADDRESS_OR_KEY_NAME COIN... --keyring-backend file --home ~/.atoshid
# e.g. atoshid add-genesis-account validator 100000000000000000000000000aatos
```

Adds an account (address or keyring key name) with an initial balance to `genesis.json`.

Key flags:
- `--keyring-backend` — used to resolve a key name to an address.
- `--home` — node home directory.
- Vesting options: `--vesting-start-time`, `--clawback`, `--funder`, `--lockup`, `--vesting` (for clawback vesting accounts).

### `atoshid gentx`

```bash
atoshid gentx validator 1000000000000000000000aatos \
  --gas-prices 1000000000aatos --keyring-backend file --chain-id atoshi_88388-1 --home ~/.atoshid
```

Generates a signed genesis `MsgCreateValidator` transaction (self-delegation) and writes it to the gentxs directory.

Key flags: `--keyring-backend`, `--chain-id`, `--gas-prices`, `--moniker`, `--home`.

### `atoshid collect-gentxs`

```bash
atoshid collect-gentxs --home ~/.atoshid
```

Collects all gentx files from the gentxs directory and injects them into `genesis.json`.

### `atoshid validate-genesis`

```bash
atoshid validate-genesis [path/to/genesis.json] --home ~/.atoshid
```

Validates that `genesis.json` is well-formed and internally consistent; defaults to the home genesis file if no path is given.

### `atoshid testnet init-files | start`

```bash
atoshid testnet init-files --v 4 --output-dir ./.testnets --starting-ip-address 192.168.10.2
atoshid testnet start --v 4 --output-dir ./.testnets
```

`testnet` is a parent group for standing up multi-validator local testnets.
- `init-files` — writes `v` node directories (priv validator, genesis, config) for running each validator in a separate process (e.g. Docker Compose).
- `start` — launches an in-process multi-validator testnet.

Key flags: `--v` (number of validators, default 4), `-o, --output-dir` (default `./.testnets`), `--chain-id`, `--keyring-backend`, `--node-dir-prefix`, `--node-daemon-home`, `--starting-ip-address`, `--base-fee`, `--min-gas-price`; for `start`: `--enable-logging`, `--print-mnemonic`, `--rpc.address`, `--api.address`.

## CometBFT / consensus admin

In this build the consensus/node-key subcommands live under `atoshid tendermint` (the `comet` alias is not registered here).

### `atoshid tendermint show-node-id`

```bash
atoshid tendermint show-node-id --home ~/.atoshid
```

Prints the node's p2p ID (derived from `node_key.json`).

### `atoshid tendermint show-validator`

```bash
atoshid tendermint show-validator --home ~/.atoshid
```

Prints this node's consensus (validator) public key in bech32/JSON form.

### `atoshid tendermint show-address`

```bash
atoshid tendermint show-address --home ~/.atoshid
```

Prints the validator consensus address (`atoshivalcons...`).

### `atoshid tendermint version`

```bash
atoshid tendermint version
```

Prints the CometBFT, ABCI, block protocol, and p2p protocol versions.

### `atoshid tendermint unsafe-reset-all`

```bash
atoshid tendermint unsafe-reset-all --home ~/.atoshid
```

Resets the node to genesis: removes `data/` and blockstore/state, and resets `priv_validator_state.json` to height 0. Keeps keys (`node_key.json`, `priv_validator_key.json`) and config. Dangerous — only run on a node you intend to re-sync from scratch.

Related: `atoshid tendermint reset-state` (clears block/state stores only) and `atoshid tendermint bootstrap-state` (bootstrap CometBFT state from a state-sync snapshot).

## State & maintenance

### `atoshid export`

```bash
atoshid export [--height H] [--for-zero-height] [--jail-allowed-addrs ...] --home ~/.atoshid
```

Exports the full application state to a genesis JSON file (on stdout). `--height -1` (default) exports the latest committed height; `--for-zero-height` rewrites state for restarting a new chain at height 0.

### `atoshid rollback`

```bash
atoshid rollback [--hard] --home ~/.atoshid
```

Rolls the node back one block (CometBFT + app state), useful to recover from a bad block or a corrupted commit. `--hard` also removes the last block from the block store.

### `atoshid snapshots`

```bash
atoshid snapshots list
atoshid snapshots export --height H
atoshid snapshots dump <height> <format>
atoshid snapshots restore <height> <format>
atoshid snapshots load <archive-file>
atoshid snapshots delete <height> <format>
```

Manages state-sync snapshots. Subcommands: `list`, `export`, `dump`, `load`, `restore`, `delete`. (Node snapshot interval/keep-recent defaults are set to 5000 / 2 in this build.) A related `atoshid prune [pruning-method]` command prunes historical app state offline.

### `atoshid migrate`

```bash
atoshid migrate TARGET_VERSION GENESIS_FILE --chain-id atoshi_88388-1 --genesis-time 2022-04-01T17:00:00Z
```

Migrates a source `genesis.json` to a target version and prints the migrated genesis to stdout (does not modify the input file).

Key flags:
- `--chain-id` — override the chain-id in the output genesis.
- `--genesis-time` — override the genesis time (RFC3339).

Note: for non-mainnet chain IDs the version key is internally prefixed with `t` (testnet migration table).

### `versiondb` / `changeset` (rocksdb builds only)

```bash
atoshid changeset ...
```

Only compiled into the binary when built with the `rocksdb` tag. Exposes the Cronos versiondb change-set tooling (dump/restore/verify per-store change sets) for the memiavl/versiondb state backend. Not available in the default build.

## Atoshi-specific tools

### `migration-merkle` (standalone binary)

```bash
migration-merkle -in snapshot.csv -out ./migration
```

A standalone Go tool (package `cmd/migration-merkle`, built as its own `migration-merkle` binary — not an `atoshid` subcommand) that builds the Merkle snapshot for the Atoshi pre-mine migration airdrop.

It reads a CSV of `claimer_bech32,amount_uatos` rows (a leading `claimer`/`address` header row is skipped; `-in -` reads stdin) and writes into `--out`:
- `root.txt` — hex SHA-256 Merkle root, set on-chain as `Params.MigrationMerkleRoot` via a gov `MsgUpdateParams`.
- `proofs.json` — per-claimer leaf + Merkle proof, fed into `MsgClaimMigrationTokens`.

The hashing matches `x/tokenomics/keeper.verifyMerkleClaim`: length-prefixed double-SHA256 leaves (`sha256(sha256(uvarint(len)||claimer||uvarint(len)||amount))`) and sorted-pair OpenZeppelin-style parents; odd nodes are promoted unchanged.

Flags: `-in` (input CSV, `-` for stdin), `-out` (output directory, default `./migration`).

### `oracle-feeder` (off-chain service — not implemented in this repo)

```bash
# not available as an atoshid subcommand in this build
```

`cmd/oracle-feeder` is currently an empty placeholder. Per the design plan the oracle feeder is a separate off-chain service (its own `atoshi-oracle-feeder` repo, modeled on Ojo / Terra Classic price-feeders) that periodically pulls market prices/volume and submits `MsgReportPrice` from a whitelisted feeder key to the on-chain `x/oracle` module.

Until that service exists, price reporting is done directly with the standard tx command from an allowed feeder address, e.g.:

```bash
atoshid tx oracle report-price ... --from feeder --keyring-backend file --chain-id atoshi_88388-1
```

Feeder addresses are authorized via `x/oracle` `Params.allowed_feeders` (set in genesis for dev, or via governance in production).

---

# Module commands

Atoshi custom modules first, then standard Cosmos SDK modules.

## Energy — `x/energy`

### Transactions

```bash
atoshid tx energy delegate DELEGATEE AMOUNT DURATION [flags]   # lend TxEnergy to DELEGATEE; freezes backing ATOS in --from until expiry
atoshid tx energy undelegate DELEGATION_ID [flags]             # cancel one of your outbound delegations and reclaim the frozen ATOS
```

- `delegate`: AMOUNT is gas units (e.g. `200000`); DURATION is a Go duration string (`24h`, `720h`). Signer is `--from`.
- `undelegate`: only the original delegator may cancel; DELEGATION_ID is the uint64 id.
- Both accept the standard tx flags (`--from`, `--chain-id`, `--gas`, `--fees`, `--node`, ...).

### Queries

```bash
atoshid query energy params                                   # energy module parameters
atoshid query energy account ADDRESS                          # settled energy state + capacity ceilings for ADDRESS
atoshid query energy delegations ADDRESS [direction]          # active delegations (direction: out | in | all, default all)
atoshid query energy estimate-fee SIGNER GAS_LIMIT [flags]    # predict TxEnergy / DeployEnergy / ATOS cost for a tx right now
```

- `estimate-fee`: `--deploy` treats the tx as a contract deployment (draws DeployEnergy first).

## Tokenomics — `x/tokenomics`

### Transactions

```bash
atoshid tx tokenomics claim-miner-locked-reward [flags]                         # validator claims its unlocked locked-pool mining rewards
atoshid tx tokenomics claim-project-treasury-reward [flags]                     # project treasury claims unlocked project-pool funds
atoshid tx tokenomics claim-migration-tokens AMOUNT PROOF_HEX[,PROOF_HEX...]    # redeem pre-mine migration ATOS via a Merkle proof
```

- `claim-miner-locked-reward`: signer must be the validator's account key (derives the val-bech32 address from `--from`).
- `claim-project-treasury-reward`: signer must be `params.ProjectTreasuryAddress`.
- `claim-migration-tokens`: AMOUNT is the integer aatos allocation from the snapshot; PROOF_HEX is a comma-separated list of sibling hashes (hex, `0x` optional) ordered leaf-to-root.

### Queries

```bash
atoshid query tokenomics params                               # tokenomics parameters
atoshid query tokenomics release-status                       # price-unlock state machine (current tier, consecutive days, totals)
atoshid query tokenomics circulating-supply                   # live circulating ATOS supply
atoshid query tokenomics block-reward                         # current per-block miner reward and halving period
atoshid query tokenomics project-claimable                    # unclaimed project-pool funds
atoshid query tokenomics miner-locked-balance VALIDATOR_ADDR  # a validator's locked-pool accrued / claimable / claimed amounts
```

## Oracle — `x/oracle`

### Transactions

```bash
atoshid tx oracle report-price PRICE VOLUME_24H SOURCE [flags]   # submit a price report (restricted to whitelisted feeders)
```

- PRICE and VOLUME_24H are decimals; SOURCE is a label such as `uniswap_v3`. Feeder is taken from `--from`.

### Queries

```bash
atoshid query oracle params                        # oracle module parameters
atoshid query oracle current-price                 # latest reported ATOS price
atoshid query oracle twap [lookback-seconds]       # TWAP price (default lookback uses params.TWAPLookbackSeconds)
atoshid query oracle price-history [limit]          # recent price reports (limit defaults to all stored)
```

## ERC20 — `x/erc20`

The `erc20` module bridges native Cosmos coins and ERC20 tokens via registered token pairs.

### Transactions

```bash
atoshid tx erc20 convert-erc20 CONTRACT_ADDRESS AMOUNT [RECEIVER]
```
Convert an ERC20 token into its paired Cosmos coin. When the optional `RECEIVER` (bech32) is omitted, the coins are sent to the signer. `CONTRACT_ADDRESS` must be a valid hex ERC20 address. Standard `--from`, `--gas`, `--fees` tx flags apply.

### Queries

```bash
atoshid query erc20 token-pairs
```
List all registered token pairs. Supports pagination flags (`--page`, `--limit`, `--count-total`).

```bash
atoshid query erc20 token-pair TOKEN
```
Get a single registered token pair by `TOKEN` (either the ERC20 contract address or the Cosmos denom).

```bash
atoshid query erc20 params
```
Get the erc20 module parameters.

## EVM — `x/evm`

The `evm` module runs the Ethereum-compatible execution layer.

### Transactions

```bash
atoshid tx evm raw TX_HEX
```
Build and broadcast a Cosmos transaction from a raw signed Ethereum transaction hex string. Respects `--generate-only` (prints tx JSON) and `-y/--yes` (skip the interactive confirmation prompt).

### Queries

```bash
atoshid query evm storage ADDRESS KEY
```
Get the storage value at a given `KEY` for an account. Uses the latest height unless `--height` is provided.

```bash
atoshid query evm code ADDRESS
```
Get the deployed bytecode at an account address. Uses the latest height unless `--height` is provided.

```bash
atoshid query evm account ADDRESS
```
Get EVM account info (balance, nonce, codehash) for an address.

```bash
atoshid query evm params
```
Get the evm module parameter values.

```bash
atoshid query evm config
```
Get the evm configuration values.

## VESTING — `x/vesting`

The `vesting` module provides clawback-enabled vesting accounts with lockup and vesting schedules.

### Transactions

```bash
atoshid tx vesting create-clawback-vesting-account FUNDER_ADDRESS ENABLE_GOV_CLAWBACK
```
Create a clawback vesting account at the signer's address, designating `FUNDER_ADDRESS` as the account that may fund schedules. `ENABLE_GOV_CLAWBACK` (bool) toggles governance-initiated clawback.

```bash
atoshid tx vesting fund-vesting-account TO_ADDRESS
```
Fund a vesting account with a token allocation. Requires at least one of `--lockup` (path to unlocking periods file) or `--vesting` (path to vesting periods file); if both are given they must total the same amount. Coins are transferred from `--from`.

```bash
atoshid tx vesting clawback ADDRESS
```
Transfer unvested funds out of a ClawbackVestingAccount. Must be sent by the original funder (`--from`); use `--dest` to route coins to a destination other than the funder.

```bash
atoshid tx vesting update-vesting-funder VESTING_ACCOUNT_ADDRESS NEW_FUNDER_ADDRESS
```
Change the funder of an existing ClawbackVestingAccount. Must be requested by the original funder (`--from`).

```bash
atoshid tx vesting convert VESTING_ACCOUNT_ADDRESS
```
Convert a fully-vested ClawbackVestingAccount back to the chain's default account type. All coins must be vested for the conversion to succeed.

### Queries

```bash
atoshid query vesting balances ADDRESS
```
Get the locked, unvested, and vested token balances for a vesting account.

## FEEMARKET — `x/feemarket`

Query-only module implementing the EIP-1559-style dynamic base fee.

### Queries

```bash
atoshid query feemarket block-gas
```
Get the block gas used at a given block height (latest height if `--height` is omitted).

```bash
atoshid query feemarket base-fee
```
Get the base fee amount at a given block height (latest height if `--height` is omitted).

```bash
atoshid query feemarket params
```
Get the fee market parameter values.

## EPOCHS — `x/epochs`

Query-only module tracking recurring on-chain time epochs (e.g. `day`, `week`).

### Queries

```bash
atoshid query epochs epoch-infos
```
List all running epoch infos. Supports pagination flags (`--page`, `--limit`, `--count-total`).

```bash
atoshid query epochs current-epoch IDENTIFIER
```
Get the current epoch number for the given `IDENTIFIER` (e.g. `atoshid query epochs current-epoch week`).

---

## Bank — `x/bank`

### Transactions
```bash
atoshid tx bank send FROM_KEY TO_ADDR AMOUNT [flags]        # e.g. AMOUNT = 1000000000000000000aatos
atoshid tx bank multi-send FROM TO1 TO2 ... AMOUNT [flags]  # one sender, N recipients each receive AMOUNT
```
- `send` moves `AMOUNT` from the key `FROM_KEY` (name in keyring, or address) to `TO_ADDR`.
- `multi-send` splits the send across every listed recipient (each gets the full `AMOUNT`).
- Common flags: `--from`, `--fees 200000000000000000aatos` or `--gas-prices 1000000000aatos`, `--gas auto --gas-adjustment 1.4`, `--chain-id`, `--note`.

### Queries
```bash
atoshid query bank balances ADDRESS [--denom aatos]         # all balances (or one denom) held by ADDRESS
atoshid query bank balance ADDRESS DENOM                    # single-denom balance
atoshid query bank total [--denom aatos]                    # total supply (all denoms, or one)
atoshid query bank denom-metadata [--denom aatos]           # registered denom metadata
```

## Staking — `x/staking`

### Transactions
```bash
atoshid tx staking create-validator VALIDATOR_JSON [flags]                 # see node-operation.md for the full walkthrough
atoshid tx staking edit-validator [flags]                                  # --new-moniker --website --details --commission-rate
atoshid tx staking delegate VALIDATOR_ADDR AMOUNT [flags]                  # bond AMOUNT to a validator
atoshid tx staking unbond VALIDATOR_ADDR AMOUNT [flags]                    # begin unbonding (subject to unbonding period)
atoshid tx staking redelegate SRC_VALIDATOR DST_VALIDATOR AMOUNT [flags]   # move bonded stake between validators
atoshid tx staking cancel-unbond VALIDATOR_ADDR AMOUNT CREATION_HEIGHT [flags]  # re-bond a pending unbonding entry
```
- `create-validator`: creating a validator is documented end-to-end in **node-operation.md** (key setup, `VALIDATOR_JSON`, self-delegation, commission params). This entry is a pointer — do not run it standalone without that runbook.
- `cancel-unbond` needs the `CREATION_HEIGHT` of the unbonding entry (find it via `query staking unbonding-delegations`).
- Common flags: `--from`, `--fees`/`--gas-prices 1000000000aatos`, `--gas auto --gas-adjustment 1.4`, `--chain-id`.

### Queries
```bash
atoshid query staking validators                              # all validators (add --status Bonded to filter)
atoshid query staking validator VALIDATOR_ADDR                # one validator's details
atoshid query staking delegations DELEGATOR_ADDR              # all delegations by a delegator
atoshid query staking unbonding-delegations DELEGATOR_ADDR    # in-progress unbondings for a delegator
atoshid query staking pool                                    # bonded / not-bonded token pool
atoshid query staking params                                  # staking params (unbonding time, max validators, bond denom)
```

## Distribution — `x/distribution`

### Transactions
```bash
atoshid tx distribution withdraw-rewards VALIDATOR_ADDR [flags]  # withdraw rewards from one validator
atoshid tx distribution withdraw-all-rewards [flags]             # withdraw rewards from every validator you delegate to
atoshid tx distribution set-withdraw-addr WITHDRAW_ADDR [flags]  # redirect future reward withdrawals to another address
atoshid tx distribution fund-community-pool AMOUNT [flags]       # donate AMOUNT to the community pool
```
- `withdraw-rewards` accepts `--commission` to also pull a validator's accumulated commission.
- Common flags: `--from`, `--fees`/`--gas-prices 1000000000aatos`, `--chain-id`.

### Queries
```bash
atoshid query distribution rewards DELEGATOR_ADDR [VALIDATOR_ADDR]  # outstanding rewards (all validators, or one)
atoshid query distribution commission VALIDATOR_ADDR               # a validator's accumulated commission
atoshid query distribution community-pool                          # current community-pool balance
atoshid query distribution params                                  # distribution params (community tax, withdraw flags)
```

## Governance (v1) — `x/gov`

### Transactions
```bash
atoshid tx gov submit-proposal PROPOSAL_JSON [flags]        # see governance-runbook.md for the full proposal workflow
atoshid tx gov deposit PROPOSAL_ID AMOUNT [flags]           # add deposit to move a proposal into voting
atoshid tx gov vote PROPOSAL_ID OPTION [flags]              # OPTION = yes | no | no_with_veto | abstain
atoshid tx gov weighted-vote PROPOSAL_ID WEIGHTED_OPTIONS [flags]  # e.g. yes=0.6,no=0.3,abstain=0.1
atoshid tx gov cancel-proposal PROPOSAL_ID [flags]          # proposer cancels their own proposal before it ends
```
- `submit-proposal` (gov v1) takes a JSON file describing messages, deposit, title and summary; the complete drafting-to-execution flow is in **governance-runbook.md**. Legacy text/param proposals use `submit-legacy-proposal`.
- Common flags: `--from`, `--fees`/`--gas-prices 1000000000aatos`, `--chain-id`.

### Queries
```bash
atoshid query gov proposals                       # list proposals (filter with --status voting_period)
atoshid query gov proposal PROPOSAL_ID            # a single proposal's details
atoshid query gov votes PROPOSAL_ID               # all votes on a proposal
atoshid query gov vote PROPOSAL_ID VOTER_ADDR     # one voter's vote
atoshid query gov tally PROPOSAL_ID               # current tally of a proposal
atoshid query gov deposits PROPOSAL_ID            # all deposits on a proposal
atoshid query gov params                          # gov params (voting period, quorum, thresholds, min deposit)
```

## Slashing — `x/slashing`

### Transactions
```bash
atoshid tx slashing unjail [flags]                # unjail your validator after downtime (run with the validator's key)
```
- Must be signed by the jailed validator's operator key (`--from`); fails while still within the jail/downtime window.
- Common flags: `--from`, `--fees`/`--gas-prices 1000000000aatos`, `--chain-id`.

### Queries
```bash
atoshid query slashing signing-info VALIDATOR_CONS_PUBKEY   # signing info for one validator (consensus pubkey)
atoshid query slashing signing-infos                        # signing info for all validators
atoshid query slashing params                               # slashing params (signed-blocks window, min signed, slash fractions)
```

## Feegrant — `x/feegrant`

### Transactions
```bash
atoshid tx feegrant grant GRANTER GRANTEE [flags]     # allow GRANTEE to pay fees from GRANTER's account
atoshid tx feegrant revoke GRANTER GRANTEE [flags]    # revoke an existing fee allowance
```
- `grant` limits: `--spend-limit 1000000000000000000aatos`, `--expiration 2026-12-31T23:59:59Z`, `--period` / `--period-limit` for periodic allowances, `--allowed-messages` to restrict message types.
- A grantee then uses the allowance on any tx via `--fee-granter GRANTER_ADDR`.
- Common flags: `--from` (must be GRANTER), `--fees`/`--gas-prices 1000000000aatos`, `--chain-id`.

### Queries
```bash
atoshid query feegrant grants GRANTER GRANTEE      # a specific granter→grantee allowance
atoshid query feegrant grants-by-grantee GRANTEE   # all allowances a grantee has received
```

## Authz — `x/authz`

### Transactions
```bash
atoshid tx authz grant GRANTEE AUTHORIZATION_TYPE [flags]   # authorize GRANTEE to run messages on your behalf
atoshid tx authz revoke GRANTEE MSG_TYPE_URL [flags]        # revoke an authorization by message type URL
atoshid tx authz exec TX_JSON_FILE [flags]                  # grantee executes an authorized tx (--from GRANTEE)
```
- `grant` types: `send` (needs `--spend-limit`), `generic` (needs `--msg-type`, e.g. `/cosmos.staking.v1beta1.MsgDelegate`), or staking types `delegate`/`unbond`/`redelegate`. Add `--expiration` to time-box it.
- `exec` takes a generated tx JSON (e.g. from `--generate-only`) and runs it under the grantee's key.
- Common flags: `--from`, `--fees`/`--gas-prices 1000000000aatos`, `--chain-id`.

### Queries
```bash
atoshid query authz grants GRANTER GRANTEE [MSG_TYPE_URL]   # authorizations from a granter to a grantee
atoshid query authz grants-by-grantee GRANTEE              # all grants received by a grantee
atoshid query authz grants-by-granter GRANTER             # all grants issued by a granter
```

## IBC Transfer — `ibc-transfer`

### Transactions
```bash
atoshid tx ibc-transfer transfer SRC_PORT SRC_CHANNEL RECEIVER AMOUNT [flags]   # e.g. transfer transfer channel-0 <bech32> 1000000000000000000aatos
```
- Standard cross-chain token transfer over ICS-20; `SRC_PORT` is normally `transfer`.
- Timeout flags: `--packet-timeout-height` (e.g. `0-1000`) and/or `--packet-timeout-timestamp` (nanoseconds, default set relative to now); `--absolute-timeouts` to treat them as absolute.
- Common flags: `--from`, `--fees`/`--gas-prices 1000000000aatos`, `--chain-id`.

### Queries
```bash
atoshid query ibc-transfer denom-traces              # all IBC denom traces (base denom + path) known to the chain
atoshid query ibc-transfer denom-trace HASH          # resolve one ibc/<HASH> denom to its trace
atoshid query ibc-transfer params                    # transfer module params (send/receive enabled)
atoshid query ibc-transfer escrow-address CHANNEL_ID # escrow account holding funds for a channel
```

---

**Fees on all transactions.** Every `tx` command needs a fee. Use either `--fees <amount>aatos` (a fixed fee) or `--gas-prices <price>aatos` combined with `--gas auto --gas-adjustment 1.4`. The network minimum gas price is `1000000000aatos`, so `--gas-prices` must be at least that. The base denom is `aatos` (1 ATOS = 10^18 aatos).

---

## Related

- [Node operation](./node-operation.md) — run a node / validator
- [Governance runbook](./governance-runbook.md) — proposals, upgrades
- [Energy module](../modules/01-energy.md) · [Tokenomics](../modules/02-tokenomics.md) — module concepts behind these commands

---

*Last reviewed: 2026-07-22*
