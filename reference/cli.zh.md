# CLI 命令参考(`atoshid`)

`atoshid` 二进制文件的完整命令参考——即 Atoshi L1 节点与
客户端。按领域组织:[全局约定](#全局约定)、节点与
管理、密钥管理,然后是各模块的 `tx` / `query` 命令(先是 Atoshi 自定义
模块,再是标准 Cosmos SDK 模块)。

本页是一份参考手册;若需引导式操作教程,请参阅 [节点操作](./node-operation.md)
(运行节点 / 验证人)、[治理操作手册](./governance-runbook.md)
(提案)以及 [能量模块](../modules/01-energy.md)。

> `atoshid` 是一个 Cosmos SDK + Ethermint 二进制文件。每个 `tx` 都会广播一笔 Cosmos
> 交易;EVM 交互(MetaMask / web3.js)通过独立的
> EVM JSON-RPC 端点进行,而非本 CLI。对任何命令运行 `--help` 可获取
> 权威且与版本匹配的完整参数列表。

## 全局约定

几乎适用于每条命令的参数:

| Flag | 含义 |
|---|---|
| `--home` | 节点 / keyring 主目录(例如 `/data/atoshi/node`) |
| `--chain-id` | `atoshi_88288-1`(测试网)/ `atoshi_88188-1`(主网) |
| `--node` | 要连接的 CometBFT RPC(例如 `tcp://localhost:26657`) |
| `--keyring-backend` | 实际使用用 `file`(需密码);;`test` 仅用于开发 |
| `--from` | 为 `tx` 签名的密钥名称或地址 |
| `--gas` / `--gas-prices` / `--fees` | gas 上限与价格,例如 `--gas 500000 --gas-prices 1000000000aatos` |
| `--output` / `-o` | `text`(默认)或 `json` |
| `-y` / `--yes` | 跳过交互式确认提示 |

denom 为 `aatos`(1 ATOS = 10^18 aatos)。网络最低 gas 价格为
`1000000000aatos`(1 gwei)。密钥使用 `eth_secp256k1` 算法,因此每个
账户同时拥有 `0x…`(EVM)和 `atoshi1…`(bech32)两种地址。

---

## 节点生命周期

守护进程二进制文件为 `atoshid`(根命令 `Use: "atoshi"`,`Short: "Atoshi Daemon"`)。默认主目录为 `~/.atoshid`;默认账户地址前缀为 `atoshi`,默认密钥算法为 `eth_secp256k1`。链 ID 遵循 `atoshi_88388-1` 形式。

### `atoshid init`

```bash
atoshid init MONIKER --chain-id atoshi_88388-1 [-o] [--home ~/.atoshid]
```

在 `--home` 下初始化私有验证人、p2p、创世及应用配置文件。仅接受一个位置参数 `MONIKER`。

关键参数:
- `--chain-id` — 创世链 ID;若留空则生成一个随机的 `atoshi_88388-<rand>`。
- `-o, --overwrite` — 覆盖已有的 `genesis.json`。
- `--recover` — 提示输入 bip39 助记词以恢复验证人密钥,而非生成新密钥。
- `--default-denom` — 写入创世的默认绑定 denom(默认为链基础 denom `aatos`)。
- `--home` — 节点主目录。

### `atoshid start`

```bash
atoshid start --minimum-gas-prices=0.0001aatos --json-rpc.api eth,txpool,personal,net,debug,web3 \
  --pruning nothing --home ~/.atoshid --chain-id atoshi_88388-1
```

运行全节点(CometBFT 共识 + EVM JSON-RPC + Cosmos gRPC/REST)。

关键参数:
- `--minimum-gas-prices` — 节点接受的最低 gas 价格(例如 `0.0001aatos`)。
- `--json-rpc.api` — 以逗号分隔、要启用的 EVM JSON-RPC 命名空间(`eth,txpool,personal,net,debug,web3`)。
- `--pruning` — 状态裁剪策略:`default` | `nothing`(保留全部状态,最适合开发 / 归档)| `everything` | `custom`(配合 `--pruning-keep-recent` / `--pruning-interval`)。
- `--home` — 节点主目录。
- `--chain-id` — 运行的链 ID。
- 其他常用参数:`--metrics`、`--log_level`、`--trace`、`--halt-height`、`--json-rpc.address`、`--api.enable`、`--grpc.address`。

### `atoshid status`

```bash
atoshid status [--node tcp://localhost:26657]
```

查询所连接节点的状态(节点信息、最新 / 最早区块高度、追赶标志、验证人信息)。

### `atoshid version`

```bash
atoshid version [--long] [-o json]
```

打印二进制文件版本;`--long` 会额外输出构建标签、Go 版本以及完整的依赖列表。

### `atoshid config`

```bash
atoshid config set client chain-id atoshi_88388-1 --home ~/.atoshid
atoshid config get client keyring-backend
```

管理 client/app/config TOML 文件(底层由 confix 支持)。子命令包括 `get`、`set`、`view`、`migrate` 和 `diff`。如 `local_node.sh` 中所用,`config set client chain-id` 和 `config set client keyring-backend` 用于写入客户端默认值。

## 创世与组网

创世构建命令注册在顶层(此构建中没有 `atoshid genesis` 父级分组)。

### `atoshid keys add | list | show | export | import | delete`

```bash
atoshid keys add validator --keyring-backend file --algo eth_secp256k1 [--recover]
atoshid keys list --keyring-backend file
atoshid keys show validator -a --keyring-backend file
atoshid keys export validator --keyring-backend file
atoshid keys import validator key.asc --keyring-backend file
atoshid keys delete validator --keyring-backend file
```

管理 keyring 中的密钥(Atoshi 使用以太坊风格的密钥)。

关键参数:
- `--keyring-backend` — `os` | `file` | `kwallet` | `pass` | `test`。`local_node.sh` 使用 `file`(受密码保护)。
- `--algo` — 签名算法;对以太坊兼容(0x)地址使用 `eth_secp256k1`。
- `--recover` — 从已有的 bip39 助记词导入(在 stdin 上提示输入)。
- `keys show -a` 仅打印地址;`--bech acc|val|cons` 选择地址形式。

### `atoshid add-genesis-account`

```bash
atoshid add-genesis-account ADDRESS_OR_KEY_NAME COIN... --keyring-backend file --home ~/.atoshid
# e.g. atoshid add-genesis-account validator 100000000000000000000000000aatos
```

向 `genesis.json` 添加一个带初始余额的账户(地址或 keyring 密钥名称)。

关键参数:
- `--keyring-backend` — 用于将密钥名称解析为地址。
- `--home` — 节点主目录。
- 归属(vesting)选项:`--vesting-start-time`、`--clawback`、`--funder`、`--lockup`、`--vesting`(用于可回收归属账户)。

### `atoshid gentx`

```bash
atoshid gentx validator 1000000000000000000000aatos \
  --gas-prices 1000000000aatos --keyring-backend file --chain-id atoshi_88388-1 --home ~/.atoshid
```

生成一笔已签名的创世 `MsgCreateValidator` 交易(自委托),并写入 gentxs 目录。

关键参数:`--keyring-backend`、`--chain-id`、`--gas-prices`、`--moniker`、`--home`。

### `atoshid collect-gentxs`

```bash
atoshid collect-gentxs --home ~/.atoshid
```

收集 gentxs 目录中的所有 gentx 文件,并将其注入 `genesis.json`。

### `atoshid validate-genesis`

```bash
atoshid validate-genesis [path/to/genesis.json] --home ~/.atoshid
```

校验 `genesis.json` 格式正确且内部一致;若未提供路径,则默认使用主目录中的创世文件。

### `atoshid testnet init-files | start`

```bash
atoshid testnet init-files --v 4 --output-dir ./.testnets --starting-ip-address 192.168.10.2
atoshid testnet start --v 4 --output-dir ./.testnets
```

`testnet` 是用于搭建多验证人本地测试网的父级分组。
- `init-files` — 写出 `v` 个节点目录(私有验证人、创世、配置),以便在各自独立的进程中运行每个验证人(例如 Docker Compose)。
- `start` — 启动一个进程内的多验证人测试网。

关键参数:`--v`(验证人数量,默认 4)、`-o, --output-dir`(默认 `./.testnets`)、`--chain-id`、`--keyring-backend`、`--node-dir-prefix`、`--node-daemon-home`、`--starting-ip-address`、`--base-fee`、`--min-gas-price`;对 `start`:`--enable-logging`、`--print-mnemonic`、`--rpc.address`、`--api.address`。

## CometBFT / 共识管理

在此构建中,共识 / 节点密钥相关子命令位于 `atoshid tendermint` 下(此处未注册 `comet` 别名)。

### `atoshid tendermint show-node-id`

```bash
atoshid tendermint show-node-id --home ~/.atoshid
```

打印节点的 p2p ID(由 `node_key.json` 派生)。

### `atoshid tendermint show-validator`

```bash
atoshid tendermint show-validator --home ~/.atoshid
```

以 bech32/JSON 形式打印本节点的共识(验证人)公钥。

### `atoshid tendermint show-address`

```bash
atoshid tendermint show-address --home ~/.atoshid
```

打印验证人共识地址(`atoshivalcons...`)。

### `atoshid tendermint version`

```bash
atoshid tendermint version
```

打印 CometBFT、ABCI、区块协议以及 p2p 协议的版本。

### `atoshid tendermint unsafe-reset-all`

```bash
atoshid tendermint unsafe-reset-all --home ~/.atoshid
```

将节点重置回创世状态:移除 `data/` 及区块存储 / 状态,并将 `priv_validator_state.json` 重置到高度 0。保留密钥(`node_key.json`、`priv_validator_key.json`)和配置。危险操作——仅在你打算从头重新同步的节点上运行。

相关命令:`atoshid tendermint reset-state`(仅清除区块 / 状态存储)以及 `atoshid tendermint bootstrap-state`(从状态同步快照引导 CometBFT 状态)。

## 状态与维护

### `atoshid export`

```bash
atoshid export [--height H] [--for-zero-height] [--jail-allowed-addrs ...] --home ~/.atoshid
```

将完整应用状态导出为创世 JSON 文件(输出到 stdout)。`--height -1`(默认)导出最新已提交高度;`--for-zero-height` 会重写状态,以便在高度 0 处重启一条新链。

### `atoshid rollback`

```bash
atoshid rollback [--hard] --home ~/.atoshid
```

将节点回滚一个区块(CometBFT + 应用状态),用于从坏块或损坏的提交中恢复。`--hard` 还会从区块存储中移除最后一个区块。

### `atoshid snapshots`

```bash
atoshid snapshots list
atoshid snapshots export --height H
atoshid snapshots dump <height> <format>
atoshid snapshots restore <height> <format>
atoshid snapshots load <archive-file>
atoshid snapshots delete <height> <format>
```

管理状态同步快照。子命令:`list`、`export`、`dump`、`load`、`restore`、`delete`。(此构建中节点快照间隔 / 保留最近数量默认分别为 5000 / 2。)相关的 `atoshid prune [pruning-method]` 命令可离线裁剪历史应用状态。

### `atoshid migrate`

```bash
atoshid migrate TARGET_VERSION GENESIS_FILE --chain-id atoshi_88388-1 --genesis-time 2022-04-01T17:00:00Z
```

将源 `genesis.json` 迁移到目标版本,并将迁移后的创世打印到 stdout(不修改输入文件)。

关键参数:
- `--chain-id` — 覆盖输出创世中的 chain-id。
- `--genesis-time` — 覆盖创世时间(RFC3339)。

注意:对于非主网链 ID,版本键会在内部加上 `t` 前缀(测试网迁移表)。

### `versiondb` / `changeset`(仅限 rocksdb 构建)

```bash
atoshid changeset ...
```

仅在使用 `rocksdb` 标签构建时才会编入二进制文件。为 memiavl/versiondb 状态后端暴露 Cronos versiondb 变更集工具(按各存储 dump/restore/verify 变更集)。默认构建中不可用。

## Atoshi 专用工具

### `migration-merkle`(独立二进制文件)

```bash
migration-merkle -in snapshot.csv -out ./migration
```

一个独立的 Go 工具(包 `cmd/migration-merkle`,构建为自己的 `migration-merkle` 二进制文件——不是 `atoshid` 子命令),用于为 Atoshi 预挖迁移空投构建 Merkle 快照。

它读取 `claimer_bech32,amount_uatos` 行的 CSV(会跳过开头的 `claimer`/`address` 表头行;`-in -` 读取 stdin),并向 `--out` 写入:
- `root.txt` — 十六进制 SHA-256 Merkle 根,通过治理 `MsgUpdateParams` 在链上设置为 `Params.MigrationMerkleRoot`。
- `proofs.json` — 每个 claimer 的叶子 + Merkle 证明,输入到 `MsgClaimMigrationTokens`。

其哈希方式与 `x/tokenomics/keeper.verifyMerkleClaim` 一致:长度前缀的双重 SHA256 叶子(`sha256(sha256(uvarint(len)||claimer||uvarint(len)||amount))`)以及排序对的 OpenZeppelin 风格父节点;奇数节点原样上提。

参数:`-in`(输入 CSV,`-` 表示 stdin)、`-out`(输出目录,默认 `./migration`)。

### `oracle-feeder`(链下服务——本仓库尚未实现)

```bash
# not available as an atoshid subcommand in this build
```

`cmd/oracle-feeder` 目前是一个空占位符。按设计规划,预言机 feeder 是一个独立的链下服务(拥有自己的 `atoshi-oracle-feeder` 仓库,参照 Ojo / Terra Classic 价格 feeder),它周期性地拉取市场价格 / 成交量,并从白名单 feeder 密钥向链上 `x/oracle` 模块提交 `MsgReportPrice`。

在该服务存在之前,价格上报直接通过标准 tx 命令、从被允许的 feeder 地址完成,例如:

```bash
atoshid tx oracle report-price ... --from feeder --keyring-backend file --chain-id atoshi_88388-1
```

Feeder 地址通过 `x/oracle` 的 `Params.allowed_feeders` 授权(开发环境在创世中设置,生产环境通过治理设置)。

---

# 模块命令

先是 Atoshi 自定义模块,再是标准 Cosmos SDK 模块。

## Energy(能量)— `x/energy`

### 交易

```bash
atoshid tx energy delegate DELEGATEE AMOUNT DURATION [flags]   # lend TxEnergy to DELEGATEE; freezes backing ATOS in --from until expiry
atoshid tx energy undelegate DELEGATION_ID [flags]             # cancel one of your outbound delegations and reclaim the frozen ATOS
```

- `delegate`:AMOUNT 是 gas 单位(例如 `200000`);DURATION 是 Go 时长字符串(`24h`、`720h`)。签名者为 `--from`。
- `undelegate`:仅原始委托人可取消;DELEGATION_ID 是 uint64 id。
- 二者均接受标准 tx 参数(`--from`、`--chain-id`、`--gas`、`--fees`、`--node`……)。

### 查询

```bash
atoshid query energy params                                   # energy module parameters
atoshid query energy account ADDRESS                          # settled energy state + capacity ceilings for ADDRESS
atoshid query energy delegations ADDRESS [direction]          # active delegations (direction: out | in | all, default all)
atoshid query energy estimate-fee SIGNER GAS_LIMIT [flags]    # predict TxEnergy / DeployEnergy / ATOS cost for a tx right now
```

- `estimate-fee`:`--deploy` 将该 tx 视为合约部署(优先动用 DeployEnergy)。

## Tokenomics(代币经济)— `x/tokenomics`

### 交易

```bash
atoshid tx tokenomics claim-miner-locked-reward [flags]                         # validator claims its unlocked locked-pool mining rewards
atoshid tx tokenomics claim-project-treasury-reward [flags]                     # project treasury claims unlocked project-pool funds
atoshid tx tokenomics claim-migration-tokens AMOUNT PROOF_HEX[,PROOF_HEX...]    # redeem pre-mine migration ATOS via a Merkle proof
```

- `claim-miner-locked-reward`:签名者必须是验证人的账户密钥(从 `--from` 派生 val-bech32 地址)。
- `claim-project-treasury-reward`:签名者必须是 `params.ProjectTreasuryAddress`。
- `claim-migration-tokens`:AMOUNT 是快照中的整数 aatos 分配额;PROOF_HEX 是以逗号分隔的兄弟哈希列表(十六进制,`0x` 可选),按叶子到根的顺序排列。

### 查询

```bash
atoshid query tokenomics params                               # tokenomics parameters
atoshid query tokenomics release-status                       # price-unlock state machine (current tier, consecutive days, totals)
atoshid query tokenomics circulating-supply                   # live circulating ATOS supply
atoshid query tokenomics block-reward                         # current per-block miner reward and halving period
atoshid query tokenomics project-claimable                    # unclaimed project-pool funds
atoshid query tokenomics miner-locked-balance VALIDATOR_ADDR  # a validator's locked-pool accrued / claimable / claimed amounts
```

## Oracle(预言机)— `x/oracle`

### 交易

```bash
atoshid tx oracle report-price PRICE VOLUME_24H SOURCE [flags]   # submit a price report (restricted to whitelisted feeders)
```

- PRICE 和 VOLUME_24H 是小数;SOURCE 是标签,例如 `uniswap_v3`。Feeder 取自 `--from`。

### 查询

```bash
atoshid query oracle params                        # oracle module parameters
atoshid query oracle current-price                 # latest reported ATOS price
atoshid query oracle twap [lookback-seconds]       # TWAP price (default lookback uses params.TWAPLookbackSeconds)
atoshid query oracle price-history [limit]          # recent price reports (limit defaults to all stored)
```

## ERC20 — `x/erc20`

`erc20` 模块通过已注册的代币对在原生 Cosmos coin 与 ERC20 代币之间架起桥梁。

### 交易

```bash
atoshid tx erc20 convert-erc20 CONTRACT_ADDRESS AMOUNT [RECEIVER]
```
将 ERC20 代币转换为其配对的 Cosmos coin。当省略可选的 `RECEIVER`(bech32)时,coin 会发送给签名者。`CONTRACT_ADDRESS` 必须是有效的十六进制 ERC20 地址。适用标准的 `--from`、`--gas`、`--fees` tx 参数。

### 查询

```bash
atoshid query erc20 token-pairs
```
列出所有已注册的代币对。支持分页参数(`--page`、`--limit`、`--count-total`)。

```bash
atoshid query erc20 token-pair TOKEN
```
按 `TOKEN`(ERC20 合约地址或 Cosmos denom)获取单个已注册代币对。

```bash
atoshid query erc20 params
```
获取 erc20 模块参数。

## EVM — `x/evm`

`evm` 模块运行以太坊兼容的执行层。

### 交易

```bash
atoshid tx evm raw TX_HEX
```
从原始已签名的以太坊交易十六进制字符串构建并广播一笔 Cosmos 交易。遵循 `--generate-only`(打印 tx JSON)和 `-y/--yes`(跳过交互式确认提示)。

### 查询

```bash
atoshid query evm storage ADDRESS KEY
```
获取账户在给定 `KEY` 处的存储值。除非提供 `--height`,否则使用最新高度。

```bash
atoshid query evm code ADDRESS
```
获取账户地址上已部署的字节码。除非提供 `--height`,否则使用最新高度。

```bash
atoshid query evm account ADDRESS
```
获取某地址的 EVM 账户信息(余额、nonce、codehash)。

```bash
atoshid query evm params
```
获取 evm 模块参数值。

```bash
atoshid query evm config
```
获取 evm 配置值。

## VESTING(归属(vesting))— `x/vesting`

`vesting` 模块提供支持回收(clawback)、带锁定与归属计划的归属账户。

### 交易

```bash
atoshid tx vesting create-clawback-vesting-account FUNDER_ADDRESS ENABLE_GOV_CLAWBACK
```
在签名者地址处创建一个可回收归属账户,指定 `FUNDER_ADDRESS` 为可为计划注资的账户。`ENABLE_GOV_CLAWBACK`(布尔)切换是否允许由治理发起的回收。

```bash
atoshid tx vesting fund-vesting-account TO_ADDRESS
```
以代币分配为一个归属账户注资。需要至少提供 `--lockup`(解锁周期文件路径)或 `--vesting`(归属周期文件路径)之一;若二者都提供,则总额必须相同。coin 从 `--from` 转出。

```bash
atoshid tx vesting clawback ADDRESS
```
将尚未归属的资金从 ClawbackVestingAccount 中转出。必须由原始 funder(`--from`)发送;使用 `--dest` 可将 coin 转到 funder 以外的目标地址。

```bash
atoshid tx vesting update-vesting-funder VESTING_ACCOUNT_ADDRESS NEW_FUNDER_ADDRESS
```
更改现有 ClawbackVestingAccount 的 funder。必须由原始 funder(`--from`)发起。

```bash
atoshid tx vesting convert VESTING_ACCOUNT_ADDRESS
```
将一个已完全归属的 ClawbackVestingAccount 转换回链的默认账户类型。所有 coin 必须已归属,转换才能成功。

### 查询

```bash
atoshid query vesting balances ADDRESS
```
获取某归属账户的已锁定、未归属和已归属代币余额。

## FEEMARKET(费用市场)— `x/feemarket`

仅供查询的模块,实现 EIP-1559 风格的动态 base fee。

### 查询

```bash
atoshid query feemarket block-gas
```
获取给定区块高度处已用的区块 gas(若省略 `--height` 则为最新高度)。

```bash
atoshid query feemarket base-fee
```
获取给定区块高度处的 base fee 数额(若省略 `--height` 则为最新高度)。

```bash
atoshid query feemarket params
```
获取费用市场参数值。

## EPOCHS(Epochs)— `x/epochs`

仅供查询的模块,跟踪循环性的链上时间 epoch(例如 `day`、`week`)。

### 查询

```bash
atoshid query epochs epoch-infos
```
列出所有正在运行的 epoch 信息。支持分页参数(`--page`、`--limit`、`--count-total`)。

```bash
atoshid query epochs current-epoch IDENTIFIER
```
获取给定 `IDENTIFIER` 的当前 epoch 编号(例如 `atoshid query epochs current-epoch week`)。

---

## Bank(银行)— `x/bank`

### 交易
```bash
atoshid tx bank send FROM_KEY TO_ADDR AMOUNT [flags]        # e.g. AMOUNT = 1000000000000000000aatos
atoshid tx bank multi-send FROM TO1 TO2 ... AMOUNT [flags]  # one sender, N recipients each receive AMOUNT
```
- `send` 将 `AMOUNT` 从密钥 `FROM_KEY`(keyring 中的名称或地址)转到 `TO_ADDR`。
- `multi-send` 将发送拆分给列出的每个接收者(每个都收到完整的 `AMOUNT`)。
- 常用参数:`--from`、`--fees 200000000000000000aatos` 或 `--gas-prices 1000000000aatos`、`--gas auto --gas-adjustment 1.4`、`--chain-id`、`--note`。

### 查询
```bash
atoshid query bank balances ADDRESS [--denom aatos]         # all balances (or one denom) held by ADDRESS
atoshid query bank balance ADDRESS DENOM                    # single-denom balance
atoshid query bank total [--denom aatos]                    # total supply (all denoms, or one)
atoshid query bank denom-metadata [--denom aatos]           # registered denom metadata
```

## Staking(质押)— `x/staking`

### 交易
```bash
atoshid tx staking create-validator VALIDATOR_JSON [flags]                 # see node-operation.md for the full walkthrough
atoshid tx staking edit-validator [flags]                                  # --new-moniker --website --details --commission-rate
atoshid tx staking delegate VALIDATOR_ADDR AMOUNT [flags]                  # bond AMOUNT to a validator
atoshid tx staking unbond VALIDATOR_ADDR AMOUNT [flags]                    # begin unbonding (subject to unbonding period)
atoshid tx staking redelegate SRC_VALIDATOR DST_VALIDATOR AMOUNT [flags]   # move bonded stake between validators
atoshid tx staking cancel-unbond VALIDATOR_ADDR AMOUNT CREATION_HEIGHT [flags]  # re-bond a pending unbonding entry
```
- `create-validator`:创建验证人的完整流程记录在 **node-operation.md** 中(密钥设置、`VALIDATOR_JSON`、自委托、佣金参数)。此条目仅为指引——不要在缺少该操作手册的情况下单独运行。
- `cancel-unbond` 需要解绑条目的 `CREATION_HEIGHT`(可通过 `query staking unbonding-delegations` 查找)。
- 常用参数:`--from`、`--fees`/`--gas-prices 1000000000aatos`、`--gas auto --gas-adjustment 1.4`、`--chain-id`。

### 查询
```bash
atoshid query staking validators                              # all validators (add --status Bonded to filter)
atoshid query staking validator VALIDATOR_ADDR                # one validator's details
atoshid query staking delegations DELEGATOR_ADDR              # all delegations by a delegator
atoshid query staking unbonding-delegations DELEGATOR_ADDR    # in-progress unbondings for a delegator
atoshid query staking pool                                    # bonded / not-bonded token pool
atoshid query staking params                                  # staking params (unbonding time, max validators, bond denom)
```

## Distribution(分配)— `x/distribution`

### 交易
```bash
atoshid tx distribution withdraw-rewards VALIDATOR_ADDR [flags]  # withdraw rewards from one validator
atoshid tx distribution withdraw-all-rewards [flags]             # withdraw rewards from every validator you delegate to
atoshid tx distribution set-withdraw-addr WITHDRAW_ADDR [flags]  # redirect future reward withdrawals to another address
atoshid tx distribution fund-community-pool AMOUNT [flags]       # donate AMOUNT to the community pool
```
- `withdraw-rewards` 接受 `--commission` 以同时提取验证人已累积的佣金。
- 常用参数:`--from`、`--fees`/`--gas-prices 1000000000aatos`、`--chain-id`。

### 查询
```bash
atoshid query distribution rewards DELEGATOR_ADDR [VALIDATOR_ADDR]  # outstanding rewards (all validators, or one)
atoshid query distribution commission VALIDATOR_ADDR               # a validator's accumulated commission
atoshid query distribution community-pool                          # current community-pool balance
atoshid query distribution params                                  # distribution params (community tax, withdraw flags)
```

## Governance (v1)(治理)— `x/gov`

### 交易
```bash
atoshid tx gov submit-proposal PROPOSAL_JSON [flags]        # see governance-runbook.md for the full proposal workflow
atoshid tx gov deposit PROPOSAL_ID AMOUNT [flags]           # add deposit to move a proposal into voting
atoshid tx gov vote PROPOSAL_ID OPTION [flags]              # OPTION = yes | no | no_with_veto | abstain
atoshid tx gov weighted-vote PROPOSAL_ID WEIGHTED_OPTIONS [flags]  # e.g. yes=0.6,no=0.3,abstain=0.1
atoshid tx gov cancel-proposal PROPOSAL_ID [flags]          # proposer cancels their own proposal before it ends
```
- `submit-proposal`(gov v1)接受一个描述消息、押金、标题和摘要的 JSON 文件;从起草到执行的完整流程见 **governance-runbook.md**。旧版文本 / 参数提案使用 `submit-legacy-proposal`。
- 常用参数:`--from`、`--fees`/`--gas-prices 1000000000aatos`、`--chain-id`。

### 查询
```bash
atoshid query gov proposals                       # list proposals (filter with --status voting_period)
atoshid query gov proposal PROPOSAL_ID            # a single proposal's details
atoshid query gov votes PROPOSAL_ID               # all votes on a proposal
atoshid query gov vote PROPOSAL_ID VOTER_ADDR     # one voter's vote
atoshid query gov tally PROPOSAL_ID               # current tally of a proposal
atoshid query gov deposits PROPOSAL_ID            # all deposits on a proposal
atoshid query gov params                          # gov params (voting period, quorum, thresholds, min deposit)
```

## Slashing(惩罚)— `x/slashing`

### 交易
```bash
atoshid tx slashing unjail [flags]                # unjail your validator after downtime (run with the validator's key)
```
- 必须由被封禁验证人的操作者密钥(`--from`)签名;在封禁 / 停机窗口期内运行会失败。
- 常用参数:`--from`、`--fees`/`--gas-prices 1000000000aatos`、`--chain-id`。

### 查询
```bash
atoshid query slashing signing-info VALIDATOR_CONS_PUBKEY   # signing info for one validator (consensus pubkey)
atoshid query slashing signing-infos                        # signing info for all validators
atoshid query slashing params                               # slashing params (signed-blocks window, min signed, slash fractions)
```

## Feegrant(手续费授权)— `x/feegrant`

### 交易
```bash
atoshid tx feegrant grant GRANTER GRANTEE [flags]     # allow GRANTEE to pay fees from GRANTER's account
atoshid tx feegrant revoke GRANTER GRANTEE [flags]    # revoke an existing fee allowance
```
- `grant` 限制:`--spend-limit 1000000000000000000aatos`、`--expiration 2026-12-31T23:59:59Z`、用于周期性授权的 `--period` / `--period-limit`、用于限制消息类型的 `--allowed-messages`。
- grantee 之后可在任意 tx 上通过 `--fee-granter GRANTER_ADDR` 使用该授权额度。
- 常用参数:`--from`(必须是 GRANTER)、`--fees`/`--gas-prices 1000000000aatos`、`--chain-id`。

### 查询
```bash
atoshid query feegrant grants GRANTER GRANTEE      # a specific granter→grantee allowance
atoshid query feegrant grants-by-grantee GRANTEE   # all allowances a grantee has received
```

## Authz(授权(authz))— `x/authz`

### 交易
```bash
atoshid tx authz grant GRANTEE AUTHORIZATION_TYPE [flags]   # authorize GRANTEE to run messages on your behalf
atoshid tx authz revoke GRANTEE MSG_TYPE_URL [flags]        # revoke an authorization by message type URL
atoshid tx authz exec TX_JSON_FILE [flags]                  # grantee executes an authorized tx (--from GRANTEE)
```
- `grant` 类型:`send`(需要 `--spend-limit`)、`generic`(需要 `--msg-type`,例如 `/cosmos.staking.v1beta1.MsgDelegate`),或质押类型 `delegate`/`unbond`/`redelegate`。添加 `--expiration` 可为其设置时限。
- `exec` 接受一个生成的 tx JSON(例如来自 `--generate-only`),并在 grantee 的密钥下运行它。
- 常用参数:`--from`、`--fees`/`--gas-prices 1000000000aatos`、`--chain-id`。

### 查询
```bash
atoshid query authz grants GRANTER GRANTEE [MSG_TYPE_URL]   # authorizations from a granter to a grantee
atoshid query authz grants-by-grantee GRANTEE              # all grants received by a grantee
atoshid query authz grants-by-granter GRANTER             # all grants issued by a granter
```

## IBC Transfer(IBC 跨链转账)— `ibc-transfer`

### 交易
```bash
atoshid tx ibc-transfer transfer SRC_PORT SRC_CHANNEL RECEIVER AMOUNT [flags]   # e.g. transfer transfer channel-0 <bech32> 1000000000000000000aatos
```
- 基于 ICS-20 的标准跨链代币转账;`SRC_PORT` 通常为 `transfer`。
- 超时参数:`--packet-timeout-height`(例如 `0-1000`)和 / 或 `--packet-timeout-timestamp`(纳秒,默认相对于当前时间设置);`--absolute-timeouts` 将其视为绝对值。
- 常用参数:`--from`、`--fees`/`--gas-prices 1000000000aatos`、`--chain-id`。

### 查询
```bash
atoshid query ibc-transfer denom-traces              # all IBC denom traces (base denom + path) known to the chain
atoshid query ibc-transfer denom-trace HASH          # resolve one ibc/<HASH> denom to its trace
atoshid query ibc-transfer params                    # transfer module params (send/receive enabled)
atoshid query ibc-transfer escrow-address CHANNEL_ID # escrow account holding funds for a channel
```

---

**所有交易的手续费。** 每条 `tx` 命令都需要手续费。可使用 `--fees <amount>aatos`(固定手续费),或将 `--gas-prices <price>aatos` 与 `--gas auto --gas-adjustment 1.4` 结合使用。网络最低 gas 价格为 `1000000000aatos`,因此 `--gas-prices` 至少须为该值。基础 denom 为 `aatos`(1 ATOS = 10^18 aatos)。

---

## 相关

- [节点操作](./node-operation.md) — 运行节点 / 验证人
- [治理操作手册](./governance-runbook.md) — 提案、升级
- [能量模块](../modules/01-energy.md) · [代币经济](../modules/02-tokenomics.md) — 这些命令背后的模块概念

---

*最后审阅:2026-07-22*
