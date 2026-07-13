# `x/gov` — 链上治理

标准的 Cosmos SDK gov 模块是唯一有权变更链参数、并触发协调式软件升级的
权威。本章介绍 Atoshi 如何配置它、一个提案的生命周期,以及三个已经
在测试网上走完流程的真实提案 —— 其中包括一个值得研究的失败案例。

## 心智模型

本文档其他地方凡是标注"可由治理调节(governance-tunable)"的内容(能量
参数、代币经济学档位阈值、预言机喂价者、gov 自身),都意味着"只能通过一个
已通过的 gov 提案中的 `MsgUpdateParams` 或等价消息来变更"。除此之外别无他途;
基金会多签无法直接写入参数。

一个提案由两个前后叠加的周期组成:

```
submit ──▶ DEPOSIT_PERIOD ──▶ VOTING_PERIOD ──▶ PASSED / REJECTED / FAILED
            (top up min_deposit)  (validators vote)   (apply or refund)
```

如果在提交时 `initial_deposit ≥ min_deposit`,押金周期会被跳过,提案直接进入投票。

## 当前参数(proposal-2 之后)

在第二个"缩短治理周期"的提案落地后,参数如下:

| 参数 | 数值 | 备注 |
|---|---|---|
| `min_deposit` | 10,000,000 aatos(0.01 ATOS) | 发起人必须锁定这么多才能进入投票 |
| `expedited_min_deposit` | 50,000,000 aatos(0.05 ATOS) | 严格大于 `min_deposit`(链不变量) |
| `voting_period` | **30 分钟** | 为提升 QA 效率,从 48 小时下调 |
| `expedited_voting_period` | 15 分钟 | 用于紧急修复 |
| `max_deposit_period` | 15 分钟 | 押金可被补足的时长 |
| `quorum` | 33.4% | 需参与投票的绑定质押占比 |
| `threshold` | 50% | `yes / (yes + no + no_with_veto)` |
| `veto_threshold` | 33.4% | `no_with_veto / total` ≥ 此值 → REJECTED + 销毁 |
| `burn_vote_veto` | true | 因否决(veto)被否时销毁押金 |
| `burn_vote_quorum` | false | 因未达法定投票率被否时退还押金 |
| `burn_proposal_deposit_prevote` | false | 在投票前取消时退还押金 |

主网会把 `voting_period` 恢复到一个更稳妥的值(通常为 48–72 小时)。30 分钟的
设置仅限测试网 —— 它让 QA 能在一个工作日内、而不是一个工作周内跑完参数变更的完整周期。

## 生命周期

### 提交(Submit)

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

链在提交时会校验内层消息(签名者、类型),但**不会**校验所提议的参数本身。
无效参数会通过投票,只有在应用(apply)时才失败 —— 见下文的提案 1。

### 押金(Deposit)

如果 `initial_deposit < min_deposit`,提案会停留在 `DEPOSIT_PERIOD`。任何
账户都可以调用 `MsgDeposit{proposal_id, amount}` 来补足。一旦总额达到
`min_deposit`,提案会自动转入投票。

如果 `deposit_end_time` 已过而仍未达到阈值,提案会被丢弃,任何部分押金会被
退还(或销毁,取决于 `burn_proposal_deposit_prevote`)。

### 投票(Vote)

验证人(以及未把投票权委托出去的委托人)投出
`MsgVote{proposal_id, option}`,其中 option 为 `yes / no / abstain / no_with_veto`。
投票者可在 `voting_end_time` 之前随时更改投票 —— 以最后一次投票为准。

计票在 `voting_end_time` 发生:

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

### 应用(Apply)

对于一个 PASSED 的提案,链会执行每一条内层消息,如同它们是由 gov 模块账户
自身发出的一样。如果执行失败(参数无效、无关错误),提案会转入
`PROPOSAL_STATUS_FAILED`,并设置 `failed_reason`。押金仍会被退还。

## 案例研究 1 —— `Shorten governance periods`(FAILED)

这是 Atoshi 测试网上提交的第一个提案。意图是把 `voting_period` 从 48 小时
降到 30 分钟以提升 QA 效率。提交时带有如下参数:

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

**结果**:投票全票通过(40M ATOS 赞成,单一验证人)。然后在应用时:

```yaml
status: PROPOSAL_STATUS_FAILED
failed_reason: "expedited minimum deposit must be greater than minimum deposit: 5000000aatos"
```

### 根因

Cosmos SDK gov 强制执行一个不变量:`expedited_min_deposit > min_deposit`。
其直觉在于,"加急(expedited)"应当比普通提案更昂贵,因为它是一条绕过了
正常冷却期的快速通道。

而这个提案把它们设反了(5M < 10M),违反了该不变量。

### 链做对了什么

- 校验运行在应用时,而非提交时。提案进入了投票,验证人看到了参数,依然投了
  赞成。链仍然拒绝应用这些坏掉的参数。**投票者无法通过 gov 意外地把链搞砖。**
- 押金被自动退还(FAILED 状态不会销毁押金,除非同时触发了
  `burn_vote_veto`,而那需要真实的否决票)。

### 教训

**在提交提案前,务必先在本地把参数跑一遍 `Validate()`**,尤其是对于
那些捆绑了许多不变量的链(能量、代币经济学、gov 各自都有自己的不变量)。一个快捷做法:

```bash
atoshid simulate tx ...   # validates without broadcasting
```

常见不变量的清单见 [`reference/faq.md`](../reference/faq.md)。

## 案例研究 2 —— `Shorten governance periods v2`(PASSED)

用相同的载荷重新提交,但将 `expedited_min_deposit` 提高到
`50,000,000 aatos`(`min_deposit` 的 5 倍)。

```json
{
  "expedited_min_deposit": [{"denom": "aatos", "amount": "50000000"}],
  ...
}
```

**结果**:

```yaml
status: PROPOSAL_STATUS_PASSED
final_tally_result:
  yes_count: 40000000000000000000000000   # 40M ATOS, 100% of stake
```

`gov.params.voting_period` 立即从 `48h0m0s` 变为 `30m0s`。后续所有提案
(包括 v20.1 升级)都使用了这个 30 分钟的窗口。

### 教训

在修正不变量之后,重试在机制上就是同一个提案。链没有"修订一个失败提案"的
概念 —— 每次提交都是一个全新的、独立的流程,拥有新的提案 id。这是有意为之的:
修订会使先前的投票失效。

## 案例研究 3 —— `v20.1` 软件升级(PASSED)

这是运维上最复杂的一种提案类型。它会触发一次对每个验证人二进制文件的
协调式停机换装(halt-and-swap)。

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

### 为什么这类升级必须协调进行

此次修复是把 `energy.SendRestriction` 接入 bank —— 这是一个影响共识的
状态转换。如果验证人 A 在区块 N 运行新二进制,而验证人 B 运行旧二进制,
它们会计算出不同的 `MsgSend` 状态变更,产生分歧,然后链就会分叉。

SDK 用一个内置的守卫来强制执行这一点。当一个带有 v20.1 处理器的二进制
试图在一条已设置但尚未触发 v20.1 计划的链上运行时,链会拒绝继续:

```
ERR BINARY UPDATED BEFORE TRIGGER! UPGRADE "v20.1" -
    in binary but not executed on chain. Downgrade your binary
```

这是有意设计的 —— Cosmos 强制运维者一直运行旧二进制,直到升级高度。完整
序列如下:

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

### 我们的 v20.1 处理器做了什么

摘自 [`app/upgrades/v20_1/upgrades.go`](https://github.com/atoshi-chain/atoshi/blob/main/app/upgrades/v20_1/upgrades.go):

1. 运行标准的模块迁移(本次发布为空操作,没有 bump 任何共识版本)。
2. 调用 `energy.RefreshAllSnapshots(ctx)`:
   - 遍历能量模块 store 中存储的每一个 `EnergyAccount`
   - 对每个账户,将 `LastBalanceSnapshot` 重置为当前 bank 余额
   - 如果 `TxEnergyAccrued` 超过了新的容量上限,则将其下调封顶
3. 记录被刷新账户的数量并返回。

此次修复搭配了一处运行时代码变更:bank 的 send-restriction 钩子现在被接入,
会在每一笔转出时调用 `energy.OnBalanceChange`,这样未来的 MsgSend 都能保持
快照同步,而无需再来一次升级。

### 运维中吸取的教训

| 遇到的坑 | 教训 |
|---|---|
| 首次模拟尝试失败:`target_height + 20` 太紧;投票期(约 600 个区块)吃光了全部缓冲 | 使用 ≥ `voting_period_blocks + 1200 buffer`(约 1 小时的缓冲,供验证人协调) |
| 首次广播失败:`--fees 5000aatos` 低于 `min_gas_price`(feemarket 更新后为 1 gwei) | 任何 feemarket 提案之后,用 `gas × gas_price` 重新计算手续费 |
| 单验证人模拟不断以 `BINARY UPDATED BEFORE TRIGGER` 失败 | 这个错误是正确的 SDK 行为 —— 对升级进行单二进制模拟在设计上就是不可能的。用 Go 单元测试来测试迁移处理器,而不是 devnet 彩排 |
| 升级触发后链以 `nil pointer dereference at app/app.go:982` panic | 这是一个掩盖了真正 `UPGRADE NEEDED` 消息的次生 panic —— 在 `FinalizeBlock` 的 defer 中加了一个 nil 守卫,以保留真实的 panic 堆栈 |

QA 模拟脚本(`scripts/upgrade_devnet.sh`)是在这个过程中构建的,对任何未来的
升级都很有用 —— 它涵盖了提案语法、投票、计划排期。处理器的执行本身只能用
两个二进制或一个 Go 单元测试来测试。

## 如何提交你自己的提案

### 第 1 步:编写 JSON

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

`metadata` 是一个自由格式的指针(通常指向 GitHub 或 Commonwealth 上的
讨论帖);它不会被链做 IPFS 校验。

### 第 2 步:本地模拟(推荐)

```bash
atoshid tx gov submit-proposal /tmp/myprop.json \
  --from validator0 --chain-id atoshi_88288-1 \
  --gas-prices 1000000000aatos --dry-run
```

这会捕获签名错误、手续费计算错误,以及部分参数校验。它**不会**捕获那些只在
应用时才失败的不变量。

### 第 3 步:提交

```bash
atoshid tx gov submit-proposal /tmp/myprop.json \
  --from validator0 --chain-id atoshi_88288-1 \
  --keyring-backend file --gas 500000 \
  --gas-prices 1000000000aatos -o json --yes \
  | tee /tmp/submit.json
```

确认 tx 哈希确实上链:

```bash
sleep 5
atoshid query tx $(jq -r .txhash /tmp/submit.json) -o json | jq '{height, code, raw_log}'
```

### 第 4 步:投票

等待 `PROPOSAL_STATUS_VOTING_PERIOD`,然后:

```bash
PID=$(atoshid query gov proposals -o json | jq -r '[.proposals[].id | tonumber] | max | tostring')
atoshid tx gov vote $PID yes --from validator0 \
  --chain-id atoshi_88288-1 --keyring-backend file \
  --gas 200000 --gas-prices 1000000000aatos -o json --yes
```

### 第 5 步:监控

```bash
# Real-time tally during voting
atoshid query gov tally $PID

# Final result after voting_end_time
atoshid query gov proposal $PID -o json \
  | jq '.proposal | {status, failed_reason, final_tally_result}'
```

一个 `PROPOSAL_STATUS_PASSED` 且 `failed_reason` 为空即表示成功。

## 常见失败模式

| 症状 | 原因 | 修复 |
|---|---|---|
| `code: 13 insufficient fee` | `--fees / --gas-prices` 低于 `feemarket.MinGasPrice` | 使用 `--gas-prices 1000000000aatos` |
| 投票通过后 `status: PROPOSAL_STATUS_FAILED` | 内层消息中违反了不变量 | 检查 `failed_reason`,修正参数,重新提交 |
| `status: PROPOSAL_STATUS_REJECTED` 且计票全为零 | 未达到法定投票率 | 提高验证人参与度 |
| 链在软件升级时停机 | 验证人未及时换装二进制 | 每个验证人:停旧、换装、重启 |
| `tx not found` | 提交在 mempool 就被拒绝;tx 从未上链 | 重新检查广播响应中的 `raw_log` |

## 相关

- [`x/energy`](./01-energy.md) —— 可经由 gov 变更的参数
- [`x/tokenomics`](./02-tokenomics.md) —— 可经由 gov 变更的释放计划
- [`x/oracle`](./03-oracle.md) —— 可经由 gov 变更的喂价者白名单
- [Cosmos SDK gov 规范](https://docs.cosmos.network/v0.50/build/modules/gov)

---

*最后审阅:2026-06-10*
