# Node Operation

如何构建 Atoshi L1 二进制文件、运行节点(单验证人或多验证人)、连接到线上网络,以及——在验证人集合对你开放的前提下——注册成为验证人。

本指南**仅**覆盖 **Cosmos SDK L1**(`atoshid`)。Polygon CDK zkEVM L2 有其自己的运维手册,不在本文范围内。

治理操作(提交提案、协调软件升级)见配套的[治理运维手册](./governance-runbook.md)。

---

## 1. 网络与公共端点

Atoshi 运行两个公共 L1 网络。每个都是带原生 EVM 支持的 Cosmos SDK 链,因此每个网络都会**同时**暴露一套 Cosmos 侧接口(REST + CometBFT RPC + gRPC)和一套 Ethereum 侧接口(EVM JSON-RPC / WebSocket)。

chain id 中编码了 EVM 数字链 id:`atoshi_<evmChainId>-<epoch>`。在 MetaMask / web3.js 中使用其数字部分。

### 生产环境(主网)

| 服务 | URL | 说明 |
|---|---|---|
| L1 RPC(EVM JSON-RPC) | `https://rpc.atoshi.org` | web3.js / MetaMask / wallet connect。Cosmos chain id `atoshi_88188-1`,**EVM chain id `88188`** |
| REST(Cosmos LCD) | `https://rpc.atoshi.org/rest-api` | Cosmos 侧 REST 网关(`/cosmos/...`、`/atoshi/...`);LCD 在代理后面监听 `:1317` |
| CometBFT RPC | `https://rpc.atoshi.org/rpc` | Tendermint/CometBFT JSON-RPC(`:26657`)——`block`、`block_search`、`tx_search`、`status` |
| 区块浏览器 | `https://explorer.atoshi.org` | L1 区块浏览器 |
| 钱包历史 API | `https://explorer.atoshi.org/example-api` | 与区块浏览器一同托管的地址交易历史服务 |
| 归档节点(P2P) | `archive.atoshi.org` | 公共归档节点,提供 P2P 用于状态同步 / 对等连接(见 [§5](#5-join-a-live-network-full-or-archive-node)) |

### 测试网

| 服务 | URL | 说明 |
|---|---|---|
| L1 RPC(EVM JSON-RPC) | `https://rpc-testnet.atoshi.org` | Cosmos chain id `atoshi_88288-1`,**EVM chain id `88288`** |
| REST(Cosmos LCD) | `https://rpc-testnet.atoshi.org/rest-api` | |
| CometBFT RPC | `https://rpc-testnet.atoshi.org/rpc` | |
| 区块浏览器 | `https://explorer-testnet.atoshi.org` | |
| 归档节点(P2P) | `tm-rpc-testnet.atoshi.org` | 提供 P2P 的公共归档节点 |

> **关于 `/rpc` 路径。** 一些较早的内部笔记把 `…/rpc` 标注为 "gRPC"。它不是——它是 **CometBFT RPC**(端口 `26657`)。钱包用它做 `block_search` / `block_results`,以读取 REST 网关不暴露的区块级事件(例如 `energy_delegation_expired`)。原生 gRPC(`:9090`)不对公众暴露。

### 默认本地端口映射

当你自己运行节点(`atoshid start`)时,默认监听器为:

| 端口 | 服务 |
|---|---|
| `26657` | CometBFT RPC |
| `26656` | P2P |
| `1317` | Cosmos REST(LCD) |
| `9090` | Cosmos gRPC |
| `8545` | EVM JSON-RPC |
| `8546` | EVM WebSocket |

---

## 2. 构建二进制文件

```bash
git clone <atoshi-chain-repo>
cd atoshi-chain
make build          # produces ./build/atoshid
./build/atoshid version --long   # confirm the version tag, e.g. v20.1
```

将其安装到 `PATH`(可选,但下文假定已安装):

```bash
sudo cp build/atoshid /usr/local/bin/atoshid
```

全文用到的关键参数:

- **基础面额(denom):** `aatos`(1 ATOS = 10^18 aatos)
- **密钥算法:** `eth_secp256k1`(Ethereum 兼容密钥;必须使用它,同一密钥才能同时产生 `0x…` 和 `atoshi1…` 两种地址)
- **Keyring 后端:** 对任何真实用途使用 `file`(带密码保护);`test`(无密码)仅供开发使用

---

## 3. 运行单节点(本地开发)

最快的途径是仓库的 `local_node.sh`,它会初始化一条单验证人开发链(chain id `atoshi_88388-1`)、给开发账户注资,并启动启用了 EVM JSON-RPC 的节点:

```bash
cd atoshi-chain
./local_node.sh            # first run: inits genesis + keys, then starts
```

启动时它会打印本地端点(`26657` / `8545` / `1317` / `9090`)以及开发密钥列表。

### 脚本做了什么(手动执行)

如果你想理解或定制它,其步骤如下:

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

`add-genesis-account` 的金额单位是 `aatos`;`100000000000000000000000000aatos` = 100,000,000 ATOS。

---

## 4. 运行多节点 / 多验证人网络

对于一个有 N 个验证人的私有测试网络,其形态是:**由一位运维者收集每个验证人的 `gentx` 来构建创世文件,分发完成后的创世文件,所有节点相互对等连接后启动。**

### 4.1 每个验证人:本地初始化并生成 gentx

在每台验证人机器 `i` 上(各有其自己的 `$HOMEDIR`,例如 `/data/atoshi/node<i>`):

```bash
atoshid init "node<i>" -o --chain-id "$CHAINID" --home "$HOMEDIR"
atoshid keys add "val<i>" --keyring-backend file --algo eth_secp256k1 --home "$HOMEDIR"
```

其中一台机器担任**创世协调者**。每个验证人向协调者发送 (a) 其创世账户地址 + 初始余额,以及 (b) 其 `gentx` 文件。

### 4.2 协调者:组装创世文件

```bash
# add every validator's account to genesis (repeat per validator)
atoshid add-genesis-account <val0-addr> 10000000000000000000000000aatos --home "$HOMEDIR"
atoshid add-genesis-account <val1-addr> 10000000000000000000000000aatos --home "$HOMEDIR"
# … etc

# collect every validator's gentx JSON into $HOMEDIR/config/gentx/, then:
atoshid collect-gentxs --home "$HOMEDIR"
atoshid validate-genesis --home "$HOMEDIR"
```

现在协调者已拥有最终的 `config/genesis.json`。**将这份完全一致的文件分发到每个节点**(用 `scp` 拷入各自的 `$HOMEDIR/config/`)。所有节点必须共享逐字节一致的创世文件,否则它们的 app hash 会发散。

### 4.3 配置对等节点

获取每个节点的 P2P id:

```bash
atoshid comet show-node-id --home "$HOMEDIR"     # prints <node-id>
```

完整的对等节点地址为 `<node-id>@<host-ip>:26656`。在每个节点上,把其他节点设为 `config/config.toml` 中的持久对等节点(persistent peers):

```toml
# $HOMEDIR/config/config.toml
persistent_peers = "<id0>@<ip0>:26656,<id1>@<ip1>:26656,<id2>@<ip2>:26656"
```

### 4.4 启动每个节点

启动所有节点(让它们紧接着一起启动,以便形成法定人数):

```bash
atoshid start \
  --log_level info \
  --minimum-gas-prices=0.0001aatos \
  --json-rpc.api eth,txpool,personal,net,debug,web3 \
  --home "$HOMEDIR" --chain-id "$CHAINID"
```

验证该集合正在出块且达成一致:

```bash
# on each node — heights climb, app hashes match across nodes, catching_up=false
curl -s http://localhost:26657/status | jq '.result.sync_info | {latest_block_height, latest_app_hash, catching_up}'
```

> **重置注意事项。** 抹除一个节点意味着删除 `data/` 并重置 `config/priv_validator_state.json`;`atoshid tendermint unsafe-reset-all --home "$HOMEDIR"` 会同时完成这两件事,并保留 `priv_validator_key.json` / `node_key.json`。如果你保留**同一份 `genesis.json`**,那么区块 1 仍会保留原始的 `genesis_time`(区块 1 的时间戳就是创世时间,而非重启时间)——真正的重启时刻会显示在区块 2 上。关于这与升级高度如何相互作用,参见[治理运维手册](./governance-runbook.md)。

---

## 5. 加入线上网络(全节点或归档节点) {#5-join-a-live-network-full-or-archive-node}

若要在不担任验证人的情况下,针对**主网**或**测试网**同步一个节点,请与公共**归档节点**对等连接——这是 Atoshi 对外暴露 P2P 的方式。

> **我们自己的验证人节点不开放直接对等连接。** 外部节点通过归档节点接入:
> - 主网:`archive.atoshi.org`
> - 测试网:`tm-rpc-testnet.atoshi.org`

### 5.1 用正确的 chain id 和创世文件初始化

```bash
CHAINID="atoshi_88288-1"          # testnet; use atoshi_88188-1 for mainnet
HOMEDIR="/data/atoshi/fullnode"

atoshid init "my-fullnode" -o --chain-id "$CHAINID" --home "$HOMEDIR"

# Replace the freshly-generated genesis with the network's genesis.
# Fetch it from the CometBFT RPC of the network you are joining:
curl -s "https://rpc-testnet.atoshi.org/rpc/genesis" | jq '.result.genesis' > "$HOMEDIR/config/genesis.json"
atoshid validate-genesis --home "$HOMEDIR"
```

(如果创世文件太大,无法在单次 RPC 调用中返回,请向运维者或从归档节点的 `config/genesis.json` 获取。)

### 5.2 指向归档节点获取对等连接

从运维者处获取归档节点的 P2P id,或在其 CometBFT RPC `status` 暴露的情况下从中获取(`.result.node_info.id`)。然后把它设为持久对等节点:

```toml
# $HOMEDIR/config/config.toml
persistent_peers = "<archive-node-id>@archive.atoshi.org:26656"     # mainnet
# testnet:
# persistent_peers = "<archive-node-id>@tm-rpc-testnet.atoshi.org:26656"
```

### 5.3 (可选)用状态同步实现快速启动

如果归档节点启用了状态同步(state-sync)快照,请设置 `[statesync] enable = true`,将 `rpc_servers` 设为归档 RPC(需要两个条目——你可以把同一个 URL 列两次),以及一个较新的 `trust_height` / `trust_hash`(从归档 RPC 的 `/block` 读取)。否则节点会从区块 1 开始重放(较慢,但总是可行)。

### 5.4 启动并观察它追上进度

```bash
atoshid start --home "$HOMEDIR" --chain-id "$CHAINID" \
  --minimum-gas-prices=0.0001aatos \
  --json-rpc.api eth,txpool,personal,net,debug,web3

curl -s http://localhost:26657/status | jq '.result.sync_info.catching_up'   # false when synced
```

一个已同步的全节点会为你提供属于你自己的私有 REST / CometBFT RPC / EVM JSON-RPC 端点——对于不应依赖公共网关的钱包、索引器或 dApp 很有用。

---

## 6. 成为验证人

> **当前政策:** 目前 Atoshi 的活跃验证人集合在主网和测试网上都**不开放**外部注册。任何人都可以运行**全节点 / 归档节点**(见 §5)并进行同步,但加入活跃集合需要与核心团队协调。下方步骤是标准机制,待验证人接入开放后即适用。

一旦你有了一个完全同步的节点(见 §5)和一个已注资的密钥:

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

验证:

```bash
atoshid query staking validators --home "$HOMEDIR" -o json | jq '.validators[] | {moniker: .description.moniker, tokens, status}'
```

你的节点必须保持在线并签名;如果它错过太多区块,就会被监禁(jailed)(用 `atoshid tx slashing unjail` 恢复)。验证人经济学——佣金、奖励、罚没——在[质押](../modules/08-staking.md)和[区块奖励](../economics/04-block-rewards.md)中讲解。

---

## 相关阅读

- [治理运维手册](./governance-runbook.md)——提案、参数变更、软件升级
- [带 EVM 的 Cosmos SDK L1](../architecture/02-cosmos-l1.md)——你所运行系统的架构
- [质押](../modules/08-staking.md)——验证人经济学
- [API 参考](./api-guide.md)——节点暴露的 REST + JSON-RPC 接口

---

*最后审阅:2026-07-11*
