# 发行计划与减半曲线

本章是 [Tokenomics 模块](../modules/02-tokenomics.md)的数字化配套说明。
它用具体数字和实例逐一列举每一种供应解锁机制,
让分析师无需阅读 keeper 代码即可构建自己的模型。

## 两台释放引擎

ATOS 通过两个并行流程解锁:

1. **区块奖励(持续)** —— 每块铸造,约每 4 年减半
2. **档位释放(条件)** —— 当价格/成交量档位条件持续满足时,
   从矿工池 + 项目池解锁的整块代币

创世流通供应量是固定的。除此之外的一切都来自
这两台引擎。

## 区块奖励:减半表 {#block-reward-halving-table}

每块奖励由区块高度确定性给出。默认参数
(权威来源:`x/tokenomics/types/helpers.go`,`DefaultParams`):

| 参数 | 值 |
|---|---|
| `initial_block_reward` | 19,819 × 10^18 aatos(= 19,819 ATOS) |
| `halving_interval_blocks` | 25,228,800 blocks |
| 出块时间(设计) | 5 seconds |
| 每年区块数 | 6,307,200(= 31,536,000 s ÷ 5 s) |
| 隐含减半周期 | 25,228,800 ÷ 6,307,200 = **~4.0 years** |

> **设计出块时间 vs. 实测出块时间。** `halving_interval_blocks`、
> `price_check_epoch_blocks`(17,280 = "每天一次")以及本章通篇的
> "约 4 年"和"30 天"数字,全部假设 **5 秒**的
> 设计目标。截至 2026-07,线上各链出块更快 ——
> 主网(`atoshi_88188-1`)约 3.5 s,测试网(`atoshi_88288-1`)约 3.4 s ——
> 因此按当前速率,一个减半周期约 2.8 年即走完,
> 而"每日"价格检查周期大约每 16.8 h 触发一次。
> 每周期的 **ATOS 数额不受影响**(它们基于区块数,
> 所以周期 0 仍发行约 500 B)。仅挂钟时长发生变化。
> 计划是把 `timeout_commit` 恢复到 5 秒目标,使
> 基于区块数的参数映射到其预期的日历语义;
> 或者由治理将基于区块数的参数重新调整到实测速率。
> 下文数字采用 5 秒设计基准。

区块奖励取自**矿工池**(1 万亿 ATOS,占固定 10 万亿供应的 10%)。
发行为比特币式:每个约 4 年周期
发行恰好为上一周期的一半,因此该级数是一个
收敛于但绝不超过 1 万亿池上限的无穷等比和。

### 各周期奖励

| 周期 | 年份 | 每块奖励 | 本周期铸造 | 累计 | 占矿工池比例 |
|---|---|---|---|---|---|
| 0 | 0–4 | 19,819 | ~500 B ATOS | 500 B | 50.00% |
| 1 | 4–8 | 9,909.5 | ~250 B | 750 B | 75.00% |
| 2 | 8–12 | 4,954.75 | ~125 B | 875 B | 87.50% |
| 3 | 12–16 | 2,477.375 | ~62.5 B | 937.5 B | 93.75% |
| 4 | 16–20 | 1,238.6875 | ~31.25 B | 968.75 B | 96.88% |
| ... | ... | ... | ... | → 1 T | → 100% |

周期 0 铸造 `19,819 × 25,228,800 ≈ 499.96 B ATOS` —— 矿工池的一半。
累计级数收敛于完整的 **1 万亿 ATOS** 矿工池
而永不耗尽,因此验证人在数十年内保持激励,
同时新发行趋近于零。

![1 万亿矿工池的比特币式减半发行](../assets/whitepaper/halving-emission.png)

### 每块拆分:20% 即时 + 80% 锁定

每块铸造的 19,819 ATOS 中(`immediate_reward_bps = 2000`,
`locked_reward_bps = 8000`):

- **3,963.8 ATOS**(20%)→ 手续费收集器,再经 `x/distribution`(扣除 2% 社区税后)支付给验证人 + 委托人 —— 可立即花费
- **15,855.2 ATOS**(80%)→ 按投票权计入验证人的 `MinerLockedBalance.locked_amount` —— 汇集直至条件档位释放触发;**不**支付给委托人

因此按 5 秒出块(= 17,280 blocks/day),每天:

- 即时分配:17,280 × 3,963.8 ≈ **68.5 M ATOS / day**(按抵押份额分摊给所有验证人)
- 锁定累积:17,280 × 15,855.2 ≈ **274.0 M ATOS / day**

## 档位释放:价格/成交量阈值

由档位 N 的 `price_base` × `tier_multiplier^N` 定义。

默认参数:

```
price_base      = 0.15 USD       (T0 price floor)
volume_base     = 150,000 USD    (T0 volume floor)
tier_multiplier = 1.1
```

因此:

| 档位 | 价格阈值 | 成交量阈值 |
|---|---|---|
| T0 | 0.150 USD | 150,000 USD/24h |
| T1 | 0.165 USD | 165,000 USD/24h |
| T2 | 0.1815 USD | 181,500 USD/24h |
| T3 | 0.1997 USD | 199,650 USD/24h |

要处于"档位 N",来自 `x/oracle` 的价格 **与** 成交量必须
同时越过档位 N 的阈值。预言机上报的 TWAP 价格与
24 小时成交量按每个周期进行比对。

![带递增价格/成交量档位的条件代币释放](../assets/whitepaper/conditional-release.png)

### 连续计数引擎

默认 `consecutive_days_required = 30`。每隔 `price_check_epoch_blocks`
(默认 17,280,按 5 秒出块约 1 天;若主网使用
不同出块时间需重新核算),EndBlocker 执行:

```
new_tier = highest tier N where (twap >= threshold_N AND volume >= vol_threshold_N)

if new_tier == current_tier:
    consecutive_days += 1
elif new_tier > current_tier:
    current_tier = new_tier
    consecutive_days = 1
elif new_tier < current_tier:
    current_tier = new_tier
    consecutive_days = 0    # streak reset

if consecutive_days == consecutive_days_required:
    trigger_release()
    # consecutive_days stays at threshold; next release in another 30 days
```

连续计数在释放时**不**重置 —— 一旦在某档位持续达标,
释放就每 30 天持续发生,直到价格/成交量回落。

### 每次事件的释放数额

`release_percentage_bps = 500`(5%),即流通供应量的 5%。

这 5% 中:
- 50% 来自矿工池(50% × 5% = 流通量的 2.5%)
- 50% 来自项目池

因此若触发时流通供应量为 200 B ATOS(首日流通量是
100 B 迁移池加上累积的即时区块奖励):

| 池 | 本次事件释放 |
|---|---|
| 矿工 | 5 B ATOS |
| 项目国库 | 5 B ATOS |

已释放的矿工 ATOS 按各验证人 `locked_amount` 份额比例
计入其 `claimable_amount`。项目 ATOS 进入
`project_treasury_address`,通过 `MsgClaimProjectTreasuryReward` 领取。

### 实例:T1 强度持续

假设:

- 周期开始时流通量:200 B ATOS
- 价格在 30 个周期内保持 0.17 USD(高于 T1 的 0.165)
- 成交量在 30 个周期内持续高于 165k USD/24h

第 30 天(首次释放):
- 5% × 200 B = 10 B ATOS 释放
- 5 B 进入矿工池 → 分配给验证人
- 5 B 进入项目池 → 存入国库

释放后流通量:约 210 B ATOS(已释放数额现在可花费)。
下一个 5% 将在再过 30 个周期后触发。

第二次释放(假设连续计数保持):
- 5% × 210 B = 10.5 B ATOS

以此类推。每次释放都是*当前*流通量的 5%,而非
首日流通量 —— 因此随着累计发行复利式增长,绝对
数额也随之增大,始终受剩余矿工池 + 项目池限额约束。

### 若价格在连续计数中途下跌

第 20 天:价格跌至 0.155 USD(低于 T1 的 0.165 但仍高于 T0 的 0.150)。

```
new_tier = T0
current_tier was T1
→ current_tier = T0
→ consecutive_days = 0
```

连续计数完全重置。第 50 天(在 T0 再持续 30 天后):
首次 T0 释放触发。熊市名副其实地暂停了高档位供应。

若价格在 T0 连续计数中途回升到 T1:

```
new_tier = T1
current_tier was T0 (with consecutive_days = N)
→ current_tier = T1
→ consecutive_days = 1
```

T1 连续计数从 1 重新开始。牛市重获档位但丢失了此前的
T0 进度。

## 迁移领取

一次性的供应增量。总限额:`migration_pool_total =
100,000,000,000 ATOS`(1000 亿,= 10 万亿总供应量的 1%)。
该池提供首日流通供应量,并按 1:100 映射兑现每一份
ERC-20 / 应用内余额。

旧版 ETH ATOS 代币的持有者通过针对创世设定的
`migration_merkle_root` 提交 Merkle 证明进行领取。领取受以下约束:

- 池余额(领取不能超过池中所有)
- 每个领取者的 Merkle 叶(每个领取者只能领取其叶对应的数额,一次)
- 时间窗口(`migration_claim_end_time_unix`)

窗口关闭后,未领取的数额归入项目池。

## 累计供应预测

将曲线拆为确定性部分(与价格无关)和
内生部分(需求驱动)。

**确定性部分** —— 迁移池 + 即时区块奖励,覆盖
周期 0(5 秒设计基准下的第 0–4 年):

| 来源 | 周期 0 对流通量的贡献 |
|---|---|
| 迁移池(首日流通量) | 100 B ATOS(大部分在第 1 年领取) |
| 铸造的区块奖励(来自矿工池) | ~500 B(池的一半) |
| —— 即时 20%(流通) | ~100 B(~25 B/year) |
| —— 锁定 80%(仅经档位释放流通) | ~400 B 已累积,尚未流通 |

在没有任何档位释放的情况下,周期 0 结束时的流通量 ≈ 100 B
(迁移)+ 100 B(即时)= **~200 B** —— 约为 10 T 上限的 2%。

**内生部分** —— 条件档位引擎在每次档位持续事件时解锁*当前*
流通量的 5%,按 50/50 拆分给矿工/项目:

| 情景 | 流通量走势 |
|---|---|
| 熊市(价格从未持续达到 T0) | 保持在确定性路径附近(周期 0 结束时约 200 B);仅通过即时区块奖励增长 |
| 牛市(T1–T3 持续) | 每次事件增加 5%;流通量在数年内从数千亿复利增长至数万亿,逐步抽取矿工池(1 T)和项目池(8.9 T) |

总供应量收敛于但绝不超过 **10 万亿**
上限。区块奖励路径是确定性的;档位释放路径是
内生的(需求驱动),这正是该模型的全部要义。

## 读取当前状态

三个查询:

```
GET /atoshi/tokenomics/v1/release_status
{
  "state": {
    "current_tier":              "1",
    "consecutive_days":          "15",
    "last_check_block":          "...",
    "total_miner_released":      "...",   // cumulative ATOS unlocked from miner pool
    "total_project_released":    "...",
    "total_immediate_distributed": "...", // cumulative 20% share paid to proposers
    "total_miner_locked":        "..."   // cumulative 80% share accumulated
  }
}
```

```
GET /atoshi/tokenomics/v1/circulating_supply
{ "circulating_supply": "..." }
```

```
GET /atoshi/tokenomics/v1/block_reward
{ "current_reward": "...", "period": "0" }
```

三者合起来告诉你:我们处在减半周期的什么位置、当前处于
哪个档位、连续计数进行到第几天、迄今已释放多少、
有多少正在途中。

## 实际影响

### 对验证人

出块 = 20% 即时收入 + 累积的 80% 锁定份额。
锁定份额只有在档位释放时才变现。**价格/成交量
持续走强直接符合你的利益** —— 没有它,你的锁定
奖励份额永远无法领取。

若验证人作恶(被罚没)或停止出块(掉线),他们
保留已累积的锁定部分但停止累积新的。锁定余额
不会被罚没,除非明确启用。

### 对长期持有者

穿越熊市持有会获得回报:当档位恢复时,供应
释放会加速发行进入验证人经济,进而反映在
能量容量上(通过已释放 ATOS 进入流通,抬高你的
相对档位计数)和验证人佣金上。

### 对分析师

用以下方式构建你的供应模型:

```
supply(t) = genesis +
            block_reward_cumulative(t) +
            tier_release_cumulative(t) +
            migration_claims_to_date(t)

block_reward_cumulative(t):
    deterministic given (block_time, halving_interval_blocks, initial_block_reward)

tier_release_cumulative(t):
    requires modeling the price/volume scenario
    Σ over events: 5% × circulating_at_event_time

migration_claims_to_date(t):
    bounded by 100 B, mostly closed by year 1
```

链通过 REST 查询提供每一个数字 —— 无需抓取。

## 对出块时间的敏感性

`halving_interval_blocks = 25,228,800` 和 `price_check_epoch_blocks =
17,280` 是**区块数**,所以其日历含义取决于
线上出块时间。在 5 秒设计下它们分别代表"4 年"和"1 天"。

截至 2026-07,线上各链更快 —— **主网约 3.5 s,测试网约 3.4 s** ——
所以按当前速率:

```
halving epoch  ≈ 25,228,800 × 3.5 s ≈ 2.8 years   (design intent: 4 years)
price epoch    ≈ 17,280 × 3.5 s     ≈ 16.8 hours  (design intent: 24 hours)
30-epoch window ≈ 21 days                          (design intent: 30 days)
```

每块和每周期的 **ATOS 数额不受影响**(基于
区块数);仅挂钟时长变化。要恢复预期的日历
语义,要么把 `timeout_commit` 恢复到 5 秒目标(
当前计划),要么将基于区块数的参数重新调整到实测速率:

```
halving_interval_blocks = ⌊4 × 365.25 × 86400 / block_time_seconds⌋
                        = 25,228,800 at 5 s (current default)
                        ≈ 36,036,000 at 3.5 s
price_check_epoch_blocks = ⌊86400 / block_time_seconds⌋
                        = 17,280 at 5 s (current default)
                        ≈ 24,700 at 3.5 s
```

这是一项待定决策(见[治理运行手册](../reference/governance-runbook.md))。

## 相关

- [Tokenomics 模块](../modules/02-tokenomics.md) —— 机制 + 状态规范
- [供应概览](./01-supply.md) —— 池 + 汇出口一览
- [区块奖励详解](./04-block-rewards.md) —— 验证人侧的账目
- [Oracle 模块](../modules/03-oracle.md) —— 驱动档位引擎的价格喂价

---

*最后审阅:2026-07-12*
