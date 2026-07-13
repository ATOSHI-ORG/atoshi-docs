# `x/oracle` — ATOS/USD 价格喂价

预言机模块在链上提供一个权威的 ATOS/USD 价格与 24 小时交易量,数据来源于一组可配置的白名单喂价者。它是驱动 `x/tokenomics` 分层引擎的输入:一旦没有新鲜的预言机读数,分层演进就会暂停。

## 心智模型

少量受信任的"喂价者"(基金会运营及合作方运营)通过 `MsgReportPrice` 向链上提交价格报告。该模块会:

1. 校验报告(喂价者在白名单内、偏离度在阈值内、来源已标注)。
2. 将其存入一个有界环形缓冲区(`PriceHistory`)。
3. 对外暴露最新报告,以及一个在 `twap_lookback_seconds`(默认 24 小时)窗口内计算的 TWAP。

该链**不**运行内部 AMM,不在内部聚合多个来源,也不对喂价者排名。信任假设是**白名单喂价者是诚实的**;消费方(最重要的是 `x/tokenomics`)将链上存储的价格作为事实真相来读取。

喂价者通常是部署在链下的机器人,从外部场所(Uniswap V3、项目方 DEX、CEX API)拉取数据并以固定节奏提交。链会在连续两次报告之间强制执行 `max_price_deviation_bps`,以防止价格失控。

## 状态

### `PriceData`

```
message PriceData {
  string price       = 1;   // USD, 18-decimal LegacyDec
  string volume_24h  = 2;   // USD, 18-decimal LegacyDec
  int64  timestamp   = 3;   // unix seconds, block time
  string feeder      = 4;   // bech32 of the reporting feeder
  string source      = 5;   // free-form label ("uniswap_v3", "static", etc.)
}
```

环形缓冲区最多保存 `MaxHistoryEntries`(编译期内置常量,默认 1000)条最近的报告。

### 当前价格

在 `/atoshi/oracle/v1/price` 暴露的"当前"价格,就是缓冲区中最新的一条 `PriceData`。链上不存在跨多个喂价者、在同一区块内做共识/中位数的机制;最后写入者胜出。

如果两个喂价者在同一区块内报告,顺序由交易排序决定——没有确定性的优先级。这是可接受的,因为所有白名单喂价者都被期望收敛到相同的来源数据上。

## TWAP 计算

回溯窗口为 `T` 的 TWAP 计算如下:

```
TWAP(T) = Σ (price_i × dt_i) / Σ dt_i
```

其中求和的范围是 `[now - T, now]` 内的报告,`dt_i` 是报告 `i` 与报告 `i+1` 之间的间隔(对于最后一条则计算到 `now`)。该公式是标准的时间加权平均。

如果回溯窗口内不存在任何报告,查询会原样返回最新的报告并附带一个标志(尚未实现为独立字段,但可通过 `timestamp` 观察到)。消费方应检查 `now - timestamp > max_price_age_seconds` 来检测过期数据。

## 消息

### `MsgReportPrice`

```
feeder     string   // signer; must be in params.allowed_feeders
price      string   // 18-dec LegacyDec
volume_24h string   // 18-dec LegacyDec
source     string   // label
```

校验:

1. `feeder` 在 `params.allowed_feeders` 中。
2. `price > 0` 且 `volume_24h >= 0`。
3. 相对于最新存储价格(如果有)的偏离度不超过 `max_price_deviation_bps`。偏离度为 `|new - prev| / prev × 10000`。
4. source 字符串非空(仅为提示;不对取值做强制约束)。

成功后,将一条带有区块时间戳的新 `PriceData` 追加到环形缓冲区。触发 `price_reported`。

由 `x/energy` 加入白名单 → 免费。

### `MsgUpdateParams`

标准的仅治理消息。用于增删喂价者、调整偏离度上限,或更改回溯窗口。

## 查询

| 端点 | 返回 |
|---|---|
| `GET /atoshi/oracle/v1/price` | 最新的 `PriceData` |
| `GET /atoshi/oracle/v1/twap?lookback_seconds=N` | `{twap_price, avg_volume}` |
| `GET /atoshi/oracle/v1/price_history?limit=N` | 最多 N 条最近的报告 |
| `GET /atoshi/oracle/v1/params` | 模块参数 |

`lookback_seconds` 参数对于 TWAP 是必填的,且独立于 `params.twap_lookback_seconds`(后者只是 tokenomics 模块内部使用的默认值)。

## 参数

| 名称 | 默认值 | 说明 |
|---|---|---|
| `allowed_feeders` | 创世时为 `[atoshi18d4ade...]`(基金会) | 通过治理添加 |
| `max_price_age_seconds` | 3600(1 小时) | 消费方的过期阈值 |
| `min_valid_reports` | 1 | 为未来多喂价者共识预留 |
| `denom` | `aatos` | 所追踪的代币 |
| `twap_lookback_seconds` | 86400(24 小时) | 默认回溯窗口提示 |
| `max_price_deviation_bps` | 5000(50%) | 单次报告的价格变动上限 |

`max_price_deviation_bps` 有意设置得较宽松(50%),原因是:

- 在创世引导期,价格可能剧烈波动。
- 更紧的上限(例如 5%)会让一个过期的或恶意的喂价者通过霸占一个陈旧价格来阻挡合法的价格变动。

在生产环境中,一旦价格发现趋于稳定,治理可以收紧此参数。

## 事件

| 事件 | 属性 |
|---|---|
| `price_reported` | `feeder`、`price`、`volume_24h`、`source`、`timestamp` |
| `price_rejected` | `feeder`、`reason`(例如 `deviation_too_high`) |

## 边界情形

### 多个喂价者,报告相互冲突

链会存储所有报告。最新的一条即为"当前"价格。TWAP 对所有报告求平均。除了单次报告的偏离度检查之外,系统不会拒绝"离群"报告——系统信任白名单。如果治理添加了一个恶意喂价者,它可以让价格漂移;补救办法是移除该喂价者。

### 喂价者停机

如果所有喂价者静默的时间超过 `max_price_age_seconds`,下游消费方(尤其是 `x/tokenomics` 分层引擎)会将价格视为过期并暂停其逻辑。链本身不会 panic 或停机——它只是在没有新鲜预言机数据的情况下运行。

### 跨来源可比性

不同喂价者可能从不同场所(Uniswap V3 与中心化交易所)报告。来源标签(`uniswap_v3`、`gate` 等)仅为提示;链不对它们加以区分。在运营层面,白名单喂价者被期望就哪个场所是权威来源达成协调。

### 稀疏数据下的 TWAP 行为

如果回溯窗口内只有一条报告,TWAP 等于该报告的价格(单个样本,时间加权无意义)。如果报告集中在窗口的起始处,则 TWAP 会偏向旧数据——当喂价节奏不规律时,TWAP 并非完美的度量。喂价者应力求提交间隔均匀(通常每 10–60 分钟一次)。

## 运营须知

- **喂价机器人**:一个小型的 Python/Go 程序,轮询外部场所并每分钟提交一次 `MsgReportPrice`。参考实现位于 `atoshi-feeder` 配套仓库中。
- **Gas 成本**:该消息在能量白名单中,因此成本为零。喂价者需要任意正数的 ATOS 余额来满足标准交易要求(签名、序列号),但不收取手续费。
- **多喂价者运营**:基金会运营、合作方运营,以及(最终的)社区运营喂价者独立提交。链不对它们进行协调;可以通过链下沟通来对齐节奏。

## 相关模块

- **`x/tokenomics`** —— 主要消费方。没有新鲜预言机数据时分层引擎会暂停。
- **`x/energy`** —— `MsgReportPrice` 受补贴,使喂价者无需持有 ATOS 用于 gas 即可运营。

---

*最后审阅:2026-05-21*
