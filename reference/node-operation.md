# Node Operation

How to build the Atoshi L1 binary, run a node (single or multi-validator), connect to a live network, and — where the validator set is open to you — register as a validator.

This guide covers the **Cosmos SDK L1** (`atoshid`) only. The Polygon CDK zkEVM L2 has its own operational runbook and is out of scope here.

Governance operations (submitting proposals, coordinating a software upgrade) live in the companion [Governance runbook](./governance-runbook.md).

---

## 1. Networks and public endpoints

Atoshi runs two public L1 networks. Each is a Cosmos SDK chain with native EVM support, so every network exposes **both** a Cosmos-side surface (REST + CometBFT RPC + gRPC) and an Ethereum-side surface (EVM JSON-RPC / WebSocket).

The chain id encodes the EVM numeric chain id: `atoshi_<evmChainId>-<epoch>`. Use the numeric part in MetaMask / web3.js.

### Production (mainnet)

| Service | URL | Notes |
|---|---|---|
| L1 RPC (EVM JSON-RPC) | `https://rpc.atoshi.org` | web3.js / MetaMask / wallet connect. Cosmos chain id `atoshi_88188-1`, **EVM chain id `88188`** |
| REST (Cosmos LCD) | `https://rpc.atoshi.org/rest-api` | Cosmos-side REST gateway (`/cosmos/...`, `/atoshi/...`); the LCD listens on `:1317` behind the proxy |
| CometBFT RPC | `https://rpc.atoshi.org/rpc` | Tendermint/CometBFT JSON-RPC (`:26657`) — `block`, `block_search`, `tx_search`, `status` |
| Explorer | `https://explorer.atoshi.org` | L1 block explorer |
| Wallet history API | `https://explorer.atoshi.org/example-api` | Address transaction history service hosted alongside the explorer |
| Archive node (P2P) | `archive.atoshi.org` | Public archive node offering P2P for state sync / peering (see [§5](#5-join-a-live-network-full-or-archive-node)) |

### Testnet

| Service | URL | Notes |
|---|---|---|
| L1 RPC (EVM JSON-RPC) | `https://rpc-testnet.atoshi.org` | Cosmos chain id `atoshi_88288-1`, **EVM chain id `88288`** |
| REST (Cosmos LCD) | `https://rpc-testnet.atoshi.org/rest-api` | |
| CometBFT RPC | `https://rpc-testnet.atoshi.org/rpc` | |
| Explorer | `https://explorer-testnet.atoshi.org` | |
| Archive node (P2P) | `tm-rpc-testnet.atoshi.org` | Public archive node offering P2P |

> **On the `/rpc` path.** Some older internal notes label `…/rpc` as "gRPC". It is not — it is the **CometBFT RPC** (port `26657`). The wallet uses it for `block_search` / `block_results` to read block-level events (e.g. `energy_delegation_expired`) that the REST gateway does not surface. Native gRPC (`:9090`) is not exposed publicly.

### Default local port map

When you run a node yourself (`atoshid start`), the default listeners are:

| Port | Service |
|---|---|
| `26657` | CometBFT RPC |
| `26656` | P2P |
| `1317` | Cosmos REST (LCD) |
| `9090` | Cosmos gRPC |
| `8545` | EVM JSON-RPC |
| `8546` | EVM WebSocket |

---

## 2. Build the binary

```bash
git clone <atoshi-chain-repo>
cd atoshi-chain
make build          # produces ./build/atoshid
./build/atoshid version --long   # confirm the version tag, e.g. v20.1
```

Install it on `PATH` (optional but assumed below):

```bash
sudo cp build/atoshid /usr/local/bin/atoshid
```

Key parameters used throughout:

- **Base denom:** `aatos` (1 ATOS = 10^18 aatos)
- **Key algorithm:** `eth_secp256k1` (Ethereum-compatible keys; required so the same key yields both a `0x…` and an `atoshi1…` address)
- **Keyring backend:** use `file` (password-protected) for anything real; `test` (no password) is dev-only

---

## 3. Run a single node (local dev)

The fastest path is the repo's `local_node.sh`, which initializes a one-validator dev chain (chain id `atoshi_88388-1`), funds dev accounts, and starts the node with EVM JSON-RPC enabled:

```bash
cd atoshi-chain
./local_node.sh            # first run: inits genesis + keys, then starts
```

On start it prints the local endpoints (`26657` / `8545` / `1317` / `9090`) and the dev key list.

### What the script does (do it manually)

If you want to understand or customize it, the sequence is:

```bash
BINARY=atoshid
CHAINID="atoshi_88388-1"
MONIKER="my-node"
HOMEDIR="$HOME/.atoshid"
KEYRING="file"
KEYALGO="eth_secp256k1"
DENOM="aatos"

# 1. init node home + genesis skeleton
$BINARY init "$MONIKER" -o --chain-id "$CHAINID" --home "$HOMEDIR"

# 2. create (or recover) a validator key
$BINARY keys add validator --keyring-backend "$KEYRING" --algo "$KEYALGO" --home "$HOMEDIR"
#   …or recover from mnemonic:
#   echo "<mnemonic>" | $BINARY keys add validator --recover --keyring-backend "$KEYRING" --algo "$KEYALGO" --home "$HOMEDIR"

# 3. fund the validator (and any dev accounts) in genesis
$BINARY add-genesis-account "$($BINARY keys show validator -a --keyring-backend "$KEYRING" --home "$HOMEDIR")" \
  100000000000000000000000000$DENOM --keyring-backend "$KEYRING" --home "$HOMEDIR"

# 4. create the genesis self-delegation tx and fold it into genesis
$BINARY gentx validator 1000000000000000000000$DENOM \
  --gas-prices 1000000000$DENOM --keyring-backend "$KEYRING" --chain-id "$CHAINID" --home "$HOMEDIR"
$BINARY collect-gentxs --home "$HOMEDIR"

# 5. sanity check
$BINARY validate-genesis --home "$HOMEDIR"

# 6. start
$BINARY start \
  --log_level info \
  --minimum-gas-prices=0.0001$DENOM \
  --json-rpc.api eth,txpool,personal,net,debug,web3 \
  --home "$HOMEDIR" --chain-id "$CHAINID"
```

`add-genesis-account` amounts are in `aatos`; `100000000000000000000000000aatos` = 100,000,000 ATOS.

---

## 4. Run a multi-node / multi-validator network

For a private test network with N validators, the shape is: **one operator builds genesis by collecting every validator's `gentx`, distributes the finished genesis, and all nodes start peered to each other.**

### 4.1 Each validator: init locally and produce a gentx

On every validator machine `i` (with its own `$HOMEDIR` = e.g. `/data/atoshi/node<i>`):

```bash
atoshid init "node<i>" -o --chain-id "$CHAINID" --home "$HOMEDIR"
atoshid keys add "val<i>" --keyring-backend file --algo eth_secp256k1 --home "$HOMEDIR"
```

One machine is the **genesis coordinator**. Every validator sends the coordinator (a) its genesis account address + starting balance and (b) its `gentx` file.

### 4.2 Coordinator: assemble genesis

```bash
# add every validator's account to genesis (repeat per validator)
atoshid add-genesis-account <val0-addr> 10000000000000000000000000aatos --home "$HOMEDIR"
atoshid add-genesis-account <val1-addr> 10000000000000000000000000aatos --home "$HOMEDIR"
# … etc

# collect every validator's gentx JSON into $HOMEDIR/config/gentx/, then:
atoshid collect-gentxs --home "$HOMEDIR"
atoshid validate-genesis --home "$HOMEDIR"
```

The coordinator now has the final `config/genesis.json`. **Distribute this exact file to every node** (`scp` it into each `$HOMEDIR/config/`). All nodes must share byte-identical genesis or their app hashes will diverge.

### 4.3 Wire up peers

Get each node's P2P id:

```bash
atoshid comet show-node-id --home "$HOMEDIR"     # prints <node-id>
```

The full peer address is `<node-id>@<host-ip>:26656`. On every node, set the others as persistent peers in `config/config.toml`:

```toml
# $HOMEDIR/config/config.toml
persistent_peers = "<id0>@<ip0>:26656,<id1>@<ip1>:26656,<id2>@<ip2>:26656"
```

### 4.4 Start every node

Start all nodes (bring them up close together so they can form quorum):

```bash
atoshid start \
  --log_level info \
  --minimum-gas-prices=0.0001aatos \
  --json-rpc.api eth,txpool,personal,net,debug,web3 \
  --home "$HOMEDIR" --chain-id "$CHAINID"
```

Verify the set is producing blocks and agreeing:

```bash
# on each node — heights climb, app hashes match across nodes, catching_up=false
curl -s http://localhost:26657/status | jq '.result.sync_info | {latest_block_height, latest_app_hash, catching_up}'
```

> **Reset caution.** Wiping a node means removing `data/` and resetting `config/priv_validator_state.json`; `atoshid tendermint unsafe-reset-all --home "$HOMEDIR"` does both and preserves `priv_validator_key.json` / `node_key.json`. If you keep the **same `genesis.json`**, block 1 keeps the original `genesis_time` (block 1's timestamp is the genesis time, not the restart time) — the true restart moment shows on block 2. See the [Governance runbook](./governance-runbook.md) for how this interacts with upgrade heights.

---

## 5. Join a live network (full or archive node)

To sync a node against **mainnet** or **testnet** without being a validator, peer with the public **archive node**, which is what Atoshi exposes for outside P2P.

> **Our own validator nodes are not open for direct peering.** External nodes connect through the archive node:
> - Mainnet: `archive.atoshi.org`
> - Testnet: `tm-rpc-testnet.atoshi.org`

### 5.1 Init with the right chain id and genesis

```bash
CHAINID="atoshi_88288-1"          # testnet; use atoshi_88188-1 for mainnet
HOMEDIR="/data/atoshi/fullnode"

atoshid init "my-fullnode" -o --chain-id "$CHAINID" --home "$HOMEDIR"

# Replace the freshly-generated genesis with the network's genesis.
# Fetch it from the CometBFT RPC of the network you are joining:
curl -s "https://rpc-testnet.atoshi.org/rpc/genesis" | jq '.result.genesis' > "$HOMEDIR/config/genesis.json"
atoshid validate-genesis --home "$HOMEDIR"
```

(If the genesis is too large to return in one RPC call, obtain it from the operator or the archive node's `config/genesis.json`.)

### 5.2 Point at the archive node for peers

Get the archive node's P2P id from the operator, or from its CometBFT RPC `status` if exposed (`.result.node_info.id`). Then set it as a persistent peer:

```toml
# $HOMEDIR/config/config.toml
persistent_peers = "<archive-node-id>@archive.atoshi.org:26656"     # mainnet
# testnet:
# persistent_peers = "<archive-node-id>@tm-rpc-testnet.atoshi.org:26656"
```

### 5.3 (Optional) State sync for a fast start

If the archive node has state-sync snapshots enabled, set `[statesync] enable = true`, `rpc_servers` to the archive RPC (two entries required — you can list the same URL twice), and a recent `trust_height` / `trust_hash` (read them from the archive RPC `/block`). Otherwise the node replays from block 1 (slower but always works).

### 5.4 Start and watch it catch up

```bash
atoshid start --home "$HOMEDIR" --chain-id "$CHAINID" \
  --minimum-gas-prices=0.0001aatos \
  --json-rpc.api eth,txpool,personal,net,debug,web3

curl -s http://localhost:26657/status | jq '.result.sync_info.catching_up'   # false when synced
```

A synced full node gives you your own private REST / CometBFT RPC / EVM JSON-RPC endpoints — useful for wallets, indexers, or dApps that should not depend on the public gateway.

---

## 6. Become a validator

> **Current policy:** Atoshi's active validator set is **not open** for external registration on either mainnet or testnet today. Anyone can run a **full / archive node** (§5) and sync, but joining the active set requires coordination with the core team. The steps below are the standard mechanism and will apply when validator onboarding opens.

Once you have a fully-synced node (§5) and a funded key:

```bash
# 1. get your validator's consensus public key (from your synced node)
atoshid comet show-validator --home "$HOMEDIR"    # prints the {"@type":"/cosmos.crypto.ed25519.PubKey",...} JSON

# 2. write a validator spec
cat > validator.json <<'JSON'
{
  "pubkey": <output of show-validator>,
  "amount": "1000000000000000000000aatos",
  "moniker": "my-validator",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
JSON

# 3. submit the create-validator tx (signed by your funded key)
atoshid tx staking create-validator validator.json \
  --from <your-key> \
  --chain-id "$CHAINID" \
  --keyring-backend file --home "$HOMEDIR" \
  --gas 500000 --gas-prices 1000000000aatos -y
```

Verify:

```bash
atoshid query staking validators --home "$HOMEDIR" -o json | jq '.validators[] | {moniker: .description.moniker, tokens, status}'
```

Your node must stay online and signing; if it misses too many blocks it will be jailed (recover with `atoshid tx slashing unjail`). Validator economics — commission, rewards, slashing — are covered in [Staking](../modules/08-staking.md) and [Block rewards](../economics/04-block-rewards.md).

---

## Related

- [Governance runbook](./governance-runbook.md) — proposals, parameter changes, software upgrades
- [Cosmos SDK L1 with EVM](../architecture/02-cosmos-l1.md) — architecture of what you're running
- [Staking](../modules/08-staking.md) — validator economics
- [API reference](./api-guide.md) — REST + JSON-RPC surface exposed by a node

---

*Last reviewed: 2026-07-11*
