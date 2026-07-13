# Governance Runbook

面向运维人员、可直接复制粘贴的操作指南,讲解如何在 Atoshi L1 上**提交并通过链上提案**:参数变更、软件升级以及各类紧急开关。若想了解 `x/gov` 的概念模型(阈值、计票数学、为何这样设计),请阅读[治理模块章节](../modules/09-governance.md);本页则是"我究竟该怎么做"的配套实操手册。

> 下文所有地址、home 路径和服务器名称均为**占位符**。请替换为你自己的值。切勿把真实的 keyring 密码粘贴进脚本或文档中。

每个片段中都会用到的约定:

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

## 1. 你需要哪一类提案?

| 目标 | 提案消息 | 是否需要新二进制文件? |
|---|---|---|
| 修改模块参数(gov 周期、feemarket 下限、energy 参数……) | 该模块对应的 `MsgUpdateParams` | 否——通过后的下一个区块即生效 |
| 发布代码变更(缺陷修复、新功能、迁移) | `MsgSoftwareUpgrade` | 是——每个节点都在升级高度切换二进制文件 |
| 取消已排定的升级 | `MsgCancelUpgrade` | 否 |
| 从社区资金池划拨资金 | `MsgCommunityPoolSpend` | 否 |

所有提案的提交方式都相同:一份携带一条或多条 `messages` 的**提案 JSON**,外加一笔 `deposit`,通过 `tx gov submit-proposal` 提交。

每条 `MsgUpdateParams` / `MsgSoftwareUpgrade` 内部的 `authority` 字段必须是 **gov 模块账户**(唯一被允许执行已通过提案的地址)。执行一次即可获取:

```bash
AUTHORITY=$($BIN query auth module-account gov --home $NODE_HOME --node $NODE -o json \
  | jq -r '.account.value.address // .account.base_account.address')
echo "authority: $AUTHORITY"      # e.g. atoshi1......  (deterministic per chain)
```

---

## 2. 押金机制(为何重要)

提案在进入投票前有两道门槛:必须达到 `min_deposit`,并且必须在 `max_deposit_period` 之内达到。

- 如果你在 `submit-proposal` 时提供的 **`deposit` 已经满足 `min_deposit`**,则提案会**跳过押金期,直接进入 `VOTING_PERIOD`**。这几乎总是你在测试网上想要的效果。
- 如果押金不足,提案会停留在 `DEPOSIT_PERIOD`,直到他人补足(或到期后押金按参数退还/销毁)。

押金是防垃圾提案的抵押品:如果提案达到法定人数(quorum),押金会被**退还**,并在特定条件下被销毁(`burn_vote_quorum` / `burn_proposal_deposit_prevote`)。因此把 `deposit` 设为**等于 `min_deposit`**,即可一步进入投票:

```bash
# read the current minimum
$BIN query gov params --home $NODE_HOME --node $NODE | grep -A3 min_deposit
```

---

## 3. 生命周期与时间线

```
submit ──▶ (deposit ≥ min_deposit?) ──▶ VOTING_PERIOD ──▶ tally ──▶ PASSED ──▶ apply
   │              │ no                        │               │
   │              ▼                           │               ▼
   │         DEPOSIT_PERIOD                   │        (upgrade) plan written,
   │                                          │         node halts at height
   └─ deposit refunded on quorum              ▼
                                       quorum + threshold checks
```

在 `voting_end_time` 进行计票,当**以下全部**成立时通过:

- 投票率(turnout)≥ `quorum`(默认 `0.334`)
- `yes / (yes+no+veto)` > `threshold`(默认 `0.5`)
- 否决(veto)占比 < `veto_threshold`(默认 `0.334`)

**时间线计算。** 总耗时 ≈ `voting_period`(仅在押金不足时才另加 `deposit_period`)。对于软件升级,还要加上等到 `upgrade-height` 的时间。

| 场景 | 到"可以开始测试"的实际耗时 |
|---|---|
| 参数变更,`voting_period=60s`,立即提交并投票 | 约 60–90 秒 |
| 软件升级,`voting_period=60s`,`upgrade-height = now + ~1000 blocks`(约 2 秒/块时 ≈ 33 分钟) | 约 35 分钟 |
| 默认 `voting_period=48h` | ≥ 2 天 |

在一条全新网络上,默认的 `voting_period` 很长(数小时到数天)。你在测试网上通常要做的**第一件事**就是缩短它(见 §5)——但这第一个提案本身仍必须熬过*当前*(较长的)周期。若要彻底避免 2 天的等待,你也可以改为 `export` 创世文件,编辑 `app_state.gov.params`,再从编辑后的创世文件重启(仅当你掌控所有验证人 / 反正要重置时才可行)。

---

## 4. 查询当前的 gov 参数

```bash
$BIN query gov params --home $NODE_HOME --node $NODE
```

典型输出(默认值):

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

> `MsgUpdateParams` 是**整体覆盖**——该消息会替换整个 params 结构体。请始终从当前值出发,只修改你确实想改的字段;缺失的字段会被重置为零/空。先查询,再编辑,然后提交。

---

## 5. 实操示例——缩短治理周期(参数变更)

目标:缩短 `voting_period` / 押金期,让 QA 能够快速迭代。这是最典型的 `MsgUpdateParams` 流程。

### 5.1 编写提案 JSON

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

`deposit`(`10000000aatos`)等于 `min_deposit`,因此提案会立即进入投票。

### 5.2 提交

```bash
$BIN tx gov submit-proposal /tmp/shorten-gov.json --from <proposer-key> $COMMON $TXFLAGS | tee /tmp/submit.json
sleep 5   # wait for inclusion
```

> 使用 `--broadcast-mode sync` 时,提交响应经常显示 `"code":0` 且 `"height":"0"`——这仅表示"已被接受进入内存池(mempool)",**并非**"已执行"。请务必用 `query tx <hash>` 确认(见 §5.6)。

### 5.3 获取提案 id

```bash
PID=$($BIN query gov proposals --reverse --limit 1 --home $NODE_HOME --node $NODE -o json | jq -r '.proposals[0].id')
echo "proposal: $PID"
```

### 5.4 投票——单个验证人

```bash
$BIN tx gov vote $PID yes --from <validator-key> $COMMON $TXFLAGS
```

### 5.5 投票——多验证人网络

每个验证人都从**其自己的 home 目录和密钥**投票;法定人数按**质押权重**计算,因此你需要足够多参与投票的验证人才能越过 `quorum`。在每台验证人机器上运行:

```bash
# node0
$BIN tx gov vote $PID yes --from val0 --home /data/atoshi/node0 --node $NODE --chain-id $CHAINID --keyring-backend file $TXFLAGS
# node1
$BIN tx gov vote $PID yes --from val1 --home /data/atoshi/node1 --node $NODE --chain-id $CHAINID --keyring-backend file $TXFLAGS
# … one per validator
```

### 5.6 确认投票确实上链

由于提交时的 `code:0` 仅表示"已被接受",请验证每一笔投票交易都已提交上链:

```bash
$BIN query tx <vote-tx-hash> --node $NODE -o json | jq '{height, code, raw_log}'
# want: code == 0, a real height, empty raw_log
```

检查已记录的投票和实时计票:

```bash
$BIN query gov vote $PID <voter-address> --node $NODE
$BIN query gov tally $PID --node $NODE
# yes_count should ≈ the total stake of the validators who voted yes
```

### 5.7 投票结束后,确认已通过并生效

```bash
$BIN query gov proposal $PID --node $NODE -o json | jq '{status, failed_reason, final_tally_result}'
# want: status == PROPOSAL_STATUS_PASSED, failed_reason empty

$BIN query gov params --node $NODE | grep -E "voting_period|max_deposit_period|quorum"
# want: the new values (e.g. voting_period: 1800s)
```

### 5.8 一键脚本

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

## 6. 实操示例——软件升级(`MsgSoftwareUpgrade`)

当变更涉及的是**代码**而不仅仅是参数时(例如某个 keeper 的缺陷修复外加一次性的状态迁移),使用此方式。整个验证人集合会在选定高度停机,并在新二进制文件上重启,由后者运行升级处理器(upgrade handler)。

> 软件升级是**必须协调一致的**:每个验证人都必须在同一高度切换二进制文件,否则链会停滞 / 分叉。提交前请向所有验证人公布升级高度和二进制文件哈希。

### 6.1 准备二进制文件并选定高度

```bash
# 1. current height
$BIN status --home $NODE_HOME | jq -r '.SyncInfo.latest_block_height'
# 2. choose an upgrade height with buffer (e.g. current + ~1000 blocks ≈ 30 min)
# 3. build the new binary and distribute to EVERY node
cd atoshi-chain && make build      # -> ./build/atoshid  (must report the new version)
scp build/atoshid <user>@<server>:/opt/atoshi/upgrades/<version>/atoshid
```

### 6.2 提交升级提案

具名升级会映射到在 `app/upgrades/<version>/` 中注册的某个 `UpgradeName`。提案中的名称**必须与注册名称完全一致**,否则处理器不会运行。

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

随后完全按照 §5.3–5.7 进行投票和确认。提案通过后,升级计划会被写入 upgrade keeper,链将在 `<chosen-height>` 停机。

### 6.3 在升级高度执行升级

到达该高度时,每个节点都会**有意 panic**:

```
ERR UPGRADE "<version>" NEEDED at height: <chosen-height>
```

**方案 A——cosmovisor(推荐)。** 预先暂存好二进制文件;cosmovisor 会自动切换并重启:

```bash
mkdir -p $NODE_HOME/cosmovisor/upgrades/<version>/bin
cp build/atoshid $NODE_HOME/cosmovisor/upgrades/<version>/bin/
# cosmovisor detects the on-chain plan and switches at the height — no manual stop/start
```

**方案 B——手动。** 在每个节点上切换并重启:

```bash
systemctl stop atoshid
cp /opt/atoshi/upgrades/<version>/atoshid /usr/local/bin/atoshid
systemctl start atoshid
```

重启后节点会运行升级处理器。对于状态迁移类升级,你会看到处理器的日志行(例如 v20.1 中随附的 energy 快照迁移会输出类似 "energy snapshot refresh complete, accounts_refreshed=N" 的消息——参见[治理模块,案例研究 3](../modules/09-governance.md))。

### 6.4 验证修复已生效

查询本次升级本应改变的任何内容。例如对于 v20.1 的 energy 快照迁移:

```bash
$BIN query energy account <affected-account> --home $NODE_HOME
# last_balance_snapshot should equal the account's real current bank balance,
# and tx_energy_capacity should equal floor(balance / 30000) * 50000
```

---

## 7. 实操示例——feemarket 最低 gas 价格(参数变更)

**问题。** `x/feemarket` 出厂时 `DefaultMinGasPrice = 0`。EIP-1559 的基础费(base-fee)控制器会在区块为空时让基础费朝 `MinGasPrice` 衰减——当下限为 0 时,在冷清的网络上基础费可能一路跌到零,于是 gas 实际上变得免费,发起垃圾交易也变得廉价。

**修复。** 将下限抬高到一个较小的非零值。一个合理的下限是 **1 gwei = 1,000,000,000 aatos/gas**——这样一笔 21,000-gas 的转账花费为 `21000 × 1e9 = 2.1e13 aatos = 0.000021 ATOS`:对用户而言微不足道,但不再为零。

这与 energy 模块的 `InsufficientGasPrice`(`0.0021 aatos/gas`,即能量耗尽时的补差价格)相互独立——不同的旋钮,不同的模块。

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

提交前请对照第 1 步的输出复核未改动的字段——错误的 `base_fee_change_denominator` 或 `elasticity_multiplier` 会悄无声息地改变整条链的手续费动态。

---

## 8. 决策说明——禁用 energy 模块(`EnergyEnabled = false`)

`energy_enabled` 是 `x/energy` 上的一个参数,可通过 `MsgUpdateParams` 提案翻转。它是一个**全局急停开关**。在提议使用它之前,请弄清楚究竟会改变什么——以下是对照代码分析得出的结论:

**关闭后会发生什么:**

1. **Ante handler**——`Consume` 会把全部 gas 作为 `ShortfallGas` 返回,因此每笔交易都以补差价格(或 EIP-1559 手续费)用 ATOS 支付 gas。ATOS 充足的用户成功;不足的用户失败——即行为与从未有过 energy 的链完全一致。
2. **转账**(`MsgSend` / EVM)——功能上不受影响;它们本来就不消耗能量。只是改为用 ATOS gas 计费,对用户来说比走能量路径更贵。
3. **快照持续更新**——`SendRestriction` → `ApplyBalanceChange` **不会**检查 `energy_enabled`,因此在模块关闭期间,各账户的快照仍持续累积。重新开启时会干净地恢复,不会有数据丢失。禁用是"停止使用",而非"销毁状态"。
4. **受补贴的消息仍然免费**——白名单检查在 `energy_enabled` 检查*之前*运行,因此即便在关闭期间,预言机价格上报、energy 委托/取消委托以及代币经济领取仍保持免费。
5. **查询**——`Account` 仍会返回(持续累积的)数据;`Params` 显示 `energy_enabled: false`;`EstimateFee` 返回 `shortfall_gas = gas_limit`(告诉钱包"把整笔按 ATOS gas 计费")。
6. **EndBlocker**——关闭期间委托到期清扫会停止。任何在关闭窗口内"到期"的委托,直到模块重新启用后才会被释放,届时积压的部分会被一并清扫。

**钱包必须适配。** 如果模块在钱包不知情的情况下被禁用,钱包会继续显示陈旧的能量 UI("energy: 0"外加"insufficient energy")。钱包应轮询 `Params` 并分支处理:

```
if (!params.energy_enabled) {
    // hide energy balance / "my energy" entry
    // hide "delegate energy" / "reclaim energy" buttons
    // on the send-confirm screen, show the ATOS gas fee, not "energy consumed: N"
    // for fee estimation, use eth_estimateGas instead of energy EstimateFee
}
```

**建议。** **在正常的测试网运行期间不要禁用它。** 把它仅作为主网的紧急停机手段;如果你确实必须翻转它,请先通知钱包团队,以便他们发布 `energy_enabled` 的 UI 分支。禁用对数据是安全的,但 UX 严格更差(用户改为支付 gas,而不是消耗能量)。

---

## 9. 常见故障模式

| 现象 | 原因 | 修复 |
|---|---|---|
| 提案卡在 `DEPOSIT_PERIOD` | `deposit` < `min_deposit` | 用 `deposit` = `min_deposit` 重新提交,或用 `tx gov deposit` 补足 |
| 尽管有赞成票仍为 `PROPOSAL_STATUS_REJECTED` | 未达到法定人数——参与投票的质押不足 | 确保有足够(按权重计)的验证人在 `voting_end_time` 之前投票 |
| 到达升级高度,链停机,却再也不恢复 | 未切换二进制文件,或 `UpgradeName` 不匹配,或二进制文件有误 | 每个节点都必须运行匹配的新二进制文件;提案名称必须等于已注册的 `UpgradeName` |
| `MsgUpdateParams` 通过了,但其他参数意外改变 | 整体覆盖语义——某个字段被遗漏 | 始终从 `query … params` 出发再编辑;切勿手写残缺的 params 块 |
| 提交显示 `code:0` 但什么也没发生 | `code:0` = 已接受进入内存池,并非已执行 | 用 `query tx <hash>` 验证是否有真实高度且 `code:0` |

---

## 相关阅读

- [治理模块](../modules/09-governance.md)——阈值、计票数学、真实案例研究
- [节点运维](./node-operation.md)——构建二进制文件、运行验证人
- [energy 模块](../modules/01-energy.md)——`energy_enabled` 和各项 energy 参数控制什么
- [Gas 与能量经济学](../economics/03-gas-fee.md)——手续费曲线与参数敏感性

---

*最后审阅:2026-07-11*
