# 5分钟 Futures 监控 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现一个 Binance USD-M Futures 5分钟监控系统：运行时通过当前 Binance routed WebSocket 订阅行情和交易时段，启动、重启、断线后用 REST 补齐历史和缺口，只用已收盘 `15m / 1h` K 线和可成交价格产生正式交易信号。

**Architecture:** 系统采用 event-driven 架构：REST 负责 bootstrap 和 backfill，WebSocket `/market` 负责 kline、markPrice、tradingSession，WebSocket `/public` 负责 bookTicker，strategy engine 只在 closed candle、session gate、高周期状态和 executable price 都有效时运行正式信号判断。V1 目标是保守、可解释、可复盘：宁可少推，也不要推提前、逆势、区间中间、非交易时段或不可成交的低质量信号。

**Tech Stack:** Python 3.11+，Binance USD-M Futures REST API，Binance USD-M Futures WebSocket Market Streams，SQLite，本地 dry-run 输出，后续可接 Telegram 或其他 notifier。

---

## 1. 结论

这套策略适合实现成代码，而且更适合 **WebSocket 主数据源 + REST 补洞** 的方式。

核心边界必须写死：

- **WebSocket 必须按当前 Binance routed endpoint 拆分：`/market` 订阅 kline、markPrice、tradingSession，`/public` 订阅 bookTicker。**
- **REST 只做启动初始化、重启恢复、断线 backfill、health/debug snapshot。**
- **未收盘 K 线只用于展示观察状态，不允许触发正式信号。**
- **正式信号只来自已收盘 `15m / 1h`，并且必须通过日线方向、4h 位置、TradFi session、短线结构、RR、executable price 可执行性和去重检查。**

这比纯 REST 每 5 分钟扫更有价值：你能实时看到价格是否已经走远，同时不会因为盘中假突破、假跌破和未收盘反抽产生误报。

## 2. 产品目标

这个系统不是泛化 signal bot。它真正要解决的是：

1. 让你不用每 5 分钟手动扫 7 个标的。
2. 价格变化足够实时，不因为只看 REST 收盘数据而感觉滞后。
3. 交易信号足够克制，不因为 WebSocket 太实时而抢跑。
4. 每条推送都能解释：为什么现在推，依据哪根已收盘 K，实时价是否还可入场。

产品侧优先级：

```text
过滤噪音 > 抢速度
可解释 > 多推送
已收盘确认 > 盘中形态
顺日线趋势 > 小周期漂亮K线
```

V1 的正确状态是：好机会能推，边缘机会少推，普通逆势和区间中间单不推。

## 3. 监控范围

### 3.1 标的

7 个 Binance USD-M Futures symbol：

| Symbol | 当前 `contractType` | Session 处理 |
| --- | --- | --- |
| `SOLUSDT` | `PERPETUAL` | 24/7 crypto perpetual |
| `PAXGUSDT` | `PERPETUAL` | 24/7 crypto perpetual |
| `CLUSDT` | `TRADIFI_PERPETUAL` | commodity session-aware |
| `SPCXUSDT` | `TRADIFI_PERPETUAL` | U.S. equity session-aware |
| `SOXLUSDT` | `TRADIFI_PERPETUAL` | U.S. equity session-aware |
| `MUUSDT` | `TRADIFI_PERPETUAL` | U.S. equity session-aware |
| `SKHYNIXUSDT` | `TRADIFI_PERPETUAL` | Korean equity session-aware |

我已用 Binance `GET /fapi/v1/exchangeInfo` 验证过，截至本方案编写时，这 7 个 symbol 都存在于 USD-M Futures，其中 5 个是 `TRADIFI_PERPETUAL`。实现不能把所有 symbol 都当 24/7 crypto perpetual。

实现时仍要在启动阶段校验 `exchangeInfo`：

- symbol 存在且状态可交易：启用。
- symbol 不存在、下线、不可交易：禁用该 symbol，并在 health/status 中明确展示原因。
- 不允许静默跳过，否则用户会误以为系统正在监控。

### 3.2 周期

每个 symbol 使用 4 个周期：

| 周期 | 用途 |
| --- | --- |
| `1d` | 日线方向，计算 `EMA20` |
| `4h` | 区间位置、关键支撑阻力、前高前低 |
| `1h` | 主结构和稳健触发确认 |
| `15m` | 快速确认和精确入场 |

额外实时数据：

- `markPrice`：展示和参考价格，不作为可成交性判断的唯一依据。
- `bookTicker`：正式信号的可执行价格来源；做多用 best ask，做空用 best bid。
- `tradingSession`：TradFi perpetual 的交易时段来源；保留 Binance raw session type，并归一化成 actionability 状态。

## 4. 外部接口依据

实现以 Binance 官方文档为准：

- [Binance USD-M Futures WebSocket Market Streams](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams)
- [Kline/Candlestick Streams](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Kline-Candlestick-Streams)
- [Mark Price Stream](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Mark-Price-Stream)
- [Kline REST Data](https://developers.binance.com/docs/derivatives/usds-margined-futures/market-data/rest-api/Kline-Candlestick-Data)

- [Trading Session Stream](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Trading-Session-Stream)

已确认的关键接口点：

- Binance USD-M Futures WebSocket 已升级为 routed endpoints：`/public`、`/market`、`/private`。
- Legacy `wss://fstream.binance.com/stream?...` 不作为本方案实现依据；Binance change log 已说明 legacy URL decommission date 为 2026-04-23。
- market base：`wss://fstream.binance.com/market`
- public base：`wss://fstream.binance.com/public`
- market combined stream：`wss://fstream.binance.com/market/stream?streams=<stream1>/<stream2>`
- public combined stream：`wss://fstream.binance.com/public/stream?streams=<stream1>/<stream2>`
- kline stream：`<symbol>@kline_<interval>`，URL PATH 为 `/market`
- kline 消息字段 `k.x` 表示这根 K 是否 closed
- kline stream 会持续推送当前正在形成的 K 线
- REST kline endpoint：`GET /fapi/v1/klines`
- REST kline 用 open time 唯一标识一根 K
- mark price stream：`<symbol>@markPrice@1s`，URL PATH 为 `/market`
- trading session stream：`tradingSession`，URL PATH 为 `/market`，raw type 包括 `PRE_MARKET`、`REGULAR`、`AFTER_MARKET`、`OVERNIGHT`、`NO_TRADING`
- book ticker stream：`<symbol>@bookTicker`，URL PATH 为 `/public`
- mark price REST snapshot：`GET /fapi/v1/premiumIndex`
- book ticker REST snapshot：`GET /fapi/v1/ticker/bookTicker`
- trading schedule REST snapshot：`GET /fapi/v1/tradingSchedule`

## 5. 总体架构

```text
REST bootstrap
  -> exchangeInfo 校验 symbol
  -> 拉历史 closed klines
  -> 拉 mark price / book ticker snapshot
  -> 初始化 EMA20、ATR15、4h range、pivots、结构状态

WebSocket runtime
  -> /market 更新 kline、mark price、tradingSession
  -> /public 更新 bid/ask
  -> 更新 forming candle
  -> 收到 k.x=true 时写入 closed candle
  -> closed 15m/1h 触发 gated strategy evaluation

Strategy engine
  -> session gate
  -> higher timeframe barrier
  -> hard filters
  -> daily trend
  -> 4h location / key levels
  -> signal type classification
  -> short-structure confirmation
  -> risk/reward
  -> executable price actionability
  -> dedupe/cooldown

Notifier / Status
  -> 正式信号
  -> 观察状态
  -> health / stale / disabled symbol
```

关键原则：

- `x=false` 的 WebSocket kline 是实时状态，不是信号依据。
- `x=true` 的 WebSocket kline 才能进入 closed candle storage。
- REST backfill 补出来的历史 K 线默认只更新状态；只有满足恢复补发规则时，才允许发 `[恢复后确认]`。

## 6. 数据流

### 6.1 WebSocket 订阅

V1 必须按 Binance routed endpoint 拆成两条 combined stream，不能把 `/market` 和 `/public` stream 混在同一个 URL。

`/market` connection：

```text
wss://fstream.binance.com/market/stream?streams=
  solusdt@kline_15m/
  solusdt@kline_1h/
  solusdt@kline_4h/
  solusdt@kline_1d/
  solusdt@markPrice@1s/
  ...
  tradingSession
```

`/public` connection：

```text
wss://fstream.binance.com/public/stream?streams=
  solusdt@bookTicker/
  paxgusdt@bookTicker/
  ...
```

7 个 symbol 总量：

- `/market`：28 个 kline streams + 7 个 mark price streams + 1 个 `tradingSession` stream
- `/public`：7 个 book ticker streams

这不是为了架构漂亮，而是 Binance 当前路由规则要求。实现里要把每个 stream 的 expected route 写成测试，避免之后又误拼回 legacy `/stream`。

### 6.2 REST 启动初始化

启动流程：

1. 调 `exchangeInfo` 校验 7 个 symbol。
2. 保存每个 symbol 的 `contractType`、`underlyingType`、`underlyingSubType` 和 session group。
3. TradFi symbol 拉 `GET /fapi/v1/tradingSchedule`，初始化 session window。
4. 每个 symbol 拉 `1d / 4h / 1h / 15m` 历史 kline。
5. 丢弃 REST 返回里尚未收盘的最新 K。
6. 拉 mark price snapshot。
7. 拉 book ticker snapshot。
8. 初始化指标、结构状态和 session 状态。
9. 建立 `/market` 和 `/public` WebSocket 连接。
10. 所有启用 symbol 的关键数据齐备后，系统才进入 healthy。

建议历史 kline 数量：

| 周期 | limit | 目的 |
| --- | ---: | --- |
| `1d` | 80 | EMA20 和趋势上下文 |
| `4h` | 120 | range、pivot、S/R |
| `1h` | 120 | 主结构和触发上下文 |
| `15m` | 200 | 短线结构和 ATR15 |

### 6.3 断线与重启 backfill

本地记录每个 `(symbol, interval)` 最后一根已收盘 K 的 `open_time_ms`。

WebSocket 重连后：

1. 暂停受影响 symbol 的正式信号推送。
2. 记录 `disconnect_started_at_ms` 和 `reconnect_completed_at_ms`。
3. 用 REST 从最后一根 closed candle 的 open time 往后拉。
4. 插入缺失的 closed candles。
5. 对 TradFi symbol 刷新 `tradingSchedule`，并等待 `tradingSession` 恢复。
6. 重新计算指标和结构状态。
7. 按恢复补发规则处理 backfill candle。
8. 恢复正常 signal evaluation。

不能假设 WebSocket 重连期间没有漏 K。REST 是恢复一致性的来源。

恢复补发规则：

- backfill candle 默认只更新状态，不发旧信号。
- 只有 `basis_close_time_ms >= disconnect_started_at_ms` 的 closed `15m / 1h` 才允许重新评估。
- 同一 `dedupe_key + basis_open_time_ms` 未推送过，才允许补发。
- 补发前必须重新通过 session gate、higher timeframe barrier、RR 和 executable price actionability。
- 通过时信号标题标记为 `[恢复后确认]`。
- 如果策略确认但 executable price 已错过，只发观察消息 `[恢复后已错过]`，不带 entry/SL/TP，避免误导为当前可入场。

### 6.4 时间口径

存储：

- 使用 Binance UTC millisecond timestamp。

展示：

- 统一展示北京时间。

每条正式信号必须带：

- 检查时间，北京时间。
- 当前 mark price。
- executable price：做多为 ask，做空为 bid。
- bid/ask spread。
- TradFi symbol 当前 raw session type 和 actionable status。
- 最近已收 `15m` 时间和 close。
- 最近已收 `1h` 时间和 close。

这样用户能区分：

- 现在价格已经走到哪里。
- 策略判断依据是哪根已收盘 K。
- 信号是否仍然可执行。

## 7. 存储设计

V1 用 SQLite 就够，重点是简单、可审计、方便恢复。

候选、观察、正式信号要分开存。原因是 rejected candidate、missed setup、observation 很多时候没有完整 entry/SL/TP/RR；如果硬塞进一张 `signal_events` 且字段全是 NOT NULL，后续实现会被迫伪造价格，复盘也会失真。

### 7.1 `candles`

```sql
CREATE TABLE candles (
  symbol TEXT NOT NULL,
  interval TEXT NOT NULL,
  open_time_ms INTEGER NOT NULL,
  close_time_ms INTEGER NOT NULL,
  open_decimal TEXT NOT NULL,
  high_decimal TEXT NOT NULL,
  low_decimal TEXT NOT NULL,
  close_decimal TEXT NOT NULL,
  volume_decimal TEXT NOT NULL,
  is_closed INTEGER NOT NULL,
  source TEXT NOT NULL,
  updated_at_ms INTEGER NOT NULL,
  PRIMARY KEY (symbol, interval, open_time_ms)
);
```

规则：

- 只持久化 closed candle。
- `source` 可取 `rest_bootstrap`、`rest_backfill`、`websocket_closed`。
- forming candle 放内存，不写成 closed candle。

### 7.2 `live_prices`

```sql
CREATE TABLE live_prices (
  symbol TEXT PRIMARY KEY,
  mark_price_decimal TEXT,
  bid_price_decimal TEXT,
  ask_price_decimal TEXT,
  spread_pct_decimal TEXT,
  mark_event_time_ms INTEGER,
  book_event_time_ms INTEGER,
  updated_at_ms INTEGER NOT NULL
);
```

规则：

- mark price 是展示和风控参考价格。
- bid/ask 是正式信号 actionability 的价格来源。
- 做多 executable price = `ask_price`。
- 做空 executable price = `bid_price`。
- `spread_pct = (ask_price - bid_price) / mid_price`。
- book ticker 过期或 spread 过宽时，不允许正式信号。
- 计算层必须使用 `Decimal`，存储层使用 decimal string，不能用 SQLite `REAL` 作为复盘和 RR 的最终依据。

### 7.3 `symbol_sessions`

```sql
CREATE TABLE symbol_sessions (
  symbol TEXT PRIMARY KEY,
  contract_type TEXT NOT NULL,
  raw_session_type TEXT,
  actionable_session_status TEXT NOT NULL,
  session_open_time_ms INTEGER,
  session_close_time_ms INTEGER,
  schedule_source TEXT NOT NULL,
  updated_at_ms INTEGER NOT NULL
);
```

规则：

- `PERPETUAL` crypto symbol 可视为 `raw_session_type = 'TRADING_24_7'`，`actionable_session_status = 'ACTIONABLE'`。
- `TRADIFI_PERPETUAL` 必须由 `tradingSession` stream 和 `tradingSchedule` REST snapshot 驱动。
- `raw_session_type` 保存 Binance 原始值，不自行改名。
- `actionable_session_status` 只允许 `ACTIONABLE`、`NOT_ACTIONABLE`、`UNKNOWN`。
- raw session type 归一化规则：
  - `REGULAR` => `ACTIONABLE`
  - `PRE_MARKET` => `NOT_ACTIONABLE`
  - `AFTER_MARKET` => `NOT_ACTIONABLE`
  - `OVERNIGHT` => `NOT_ACTIONABLE`
  - `NO_TRADING` => `NOT_ACTIONABLE`
  - missing / stale / unrecognized => `UNKNOWN`
- V1 只有 `ACTIONABLE` 才允许正式信号。`PRE_MARKET`、`AFTER_MARKET`、`OVERNIGHT` 暂不发正式信号，避免流动性/价差和图表结构不稳定。

### 7.4 `signal_candidates`

```sql
CREATE TABLE signal_candidates (
  id TEXT PRIMARY KEY,
  symbol TEXT NOT NULL,
  direction TEXT NOT NULL,
  signal_type TEXT NOT NULL,
  dedupe_key TEXT NOT NULL,
  key_level_decimal TEXT,
  basis_interval TEXT NOT NULL,
  basis_open_time_ms INTEGER NOT NULL,
  basis_close_time_ms INTEGER NOT NULL,
  status TEXT NOT NULL,
  rejected_reason TEXT,
  reason_json TEXT NOT NULL,
  created_at_ms INTEGER NOT NULL,
  updated_at_ms INTEGER NOT NULL,
  UNIQUE(dedupe_key, basis_open_time_ms)
);
```

规则：

- 记录 candidate、rejected、missed、confirmed-but-not-alerted。
- `entry/SL/TP/RR` 不放在 candidate 表里，因为很多候选在 risk 阶段前就会被拒绝。
- `dedupe_key + basis_open_time_ms` 用于 backfill 补发和重复推送控制。

### 7.5 `signal_alerts`

```sql
CREATE TABLE signal_alerts (
  id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL,
  symbol TEXT NOT NULL,
  direction TEXT NOT NULL,
  signal_type TEXT NOT NULL,
  dedupe_key TEXT NOT NULL,
  key_level_decimal TEXT NOT NULL,
  entry_price_decimal TEXT NOT NULL,
  executable_price_decimal TEXT NOT NULL,
  executable_side TEXT NOT NULL,
  bid_price_decimal TEXT,
  ask_price_decimal TEXT,
  spread_pct_decimal TEXT,
  stop_loss_decimal TEXT NOT NULL,
  tp1_decimal TEXT NOT NULL,
  tp2_decimal TEXT NOT NULL,
  rr_tp1_entry_decimal TEXT NOT NULL,
  rr_tp2_entry_decimal TEXT NOT NULL,
  rr_tp1_executable_decimal TEXT NOT NULL,
  rr_tp2_executable_decimal TEXT NOT NULL,
  basis_interval TEXT NOT NULL,
  basis_open_time_ms INTEGER NOT NULL,
  basis_close_time_ms INTEGER NOT NULL,
  basis_15m_open_time_ms INTEGER,
  basis_1h_open_time_ms INTEGER,
  mark_price_decimal TEXT NOT NULL,
  status TEXT NOT NULL,
  reason_json TEXT NOT NULL,
  created_at_ms INTEGER NOT NULL,
  UNIQUE(dedupe_key, basis_open_time_ms)
);
```

规则：

- 只有正式推送才进入 `signal_alerts`。
- 每条正式信号必须保存 reason chain，方便复盘为什么推。
- `executable_side` 做多为 `ASK`，做空为 `BID`。
- `rr_*_entry` 记录确认 K close 视角。
- `rr_*_executable` 记录用户当前按 ask/bid 入场视角，正式推送必须以后者达标。
- `basis_interval + basis_open_time_ms` 是正式信号去重和恢复补发的硬约束。
- `basis_15m_open_time_ms` / `basis_1h_open_time_ms` 记录信号依据，不允许只有实时价。
- `status` V1 可用：`alerted`、`expired`、`invalidated`、`tp1_reached`、`tp2_reached`。

### 7.6 Decimal / tick 规则

所有价格、数量、RR、spread 计算：

- 计算层使用 `Decimal`。
- 从 Binance 读取的价格和数量保持原始 string 转 `Decimal`。
- 存储层保存 decimal string，不用 SQLite `REAL` 保存最终价格。
- 下单/展示相关价格按 `exchangeInfo` 的 `tickSize` 量化。
- 如果后续需要更高性能，可以额外存 `price_ticks INTEGER`，但 V1 不需要提前引入双存储。

### 7.7 `system_state`

```sql
CREATE TABLE system_state (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at_ms INTEGER NOT NULL
);
```

用途：

- 记录 last closed candle time。
- 记录 disconnect_started_at_ms。
- 记录 reconnect count。
- 记录 last backfill time。
- 记录 notifier 状态。

## 8. 策略判断顺序

必须按下面顺序判断：

1. 数据新鲜度和 symbol 可用性。
2. Session gate。
3. Higher timeframe barrier。
4. 三不铁律 hard filters。
5. 日线趋势方向。
6. 4h 位置和关键位。
7. 信号类型候选。
8. 短线结构确认。
9. RR 检查。
10. Executable price actionability 检查。
11. 去重和 cooldown。

原因：不能让小周期漂亮 K 线绕过大方向、位置和风控。

### 8.1 Session gate

正式信号必须满足：

- symbol `status = TRADING`。
- `contractType = PERPETUAL`：按 24/7 处理，但仍要检查 WebSocket/REST 数据 freshness。
- `contractType = TRADIFI_PERPETUAL`：必须满足当前 `actionable_session_status = ACTIONABLE`。
- `actionable_session_status = NOT_ACTIONABLE` 或 `UNKNOWN`：不发正式信号，只允许观察状态。

TradFi symbol 的 K 线 freshness 不能按 24/7 固定间隔硬判。非交易时段没有新 closed candle 是正常状态，不是 stale。

### 8.2 Higher Timeframe Barrier

closed `15m / 1h` 触发 strategy evaluation 前，必须确认所有高周期上下文已经不落后。

对 trigger candle：

```text
trigger_close_time_ms = basis candle close time
```

定义：

```text
expected_closed_close_time(interval, trigger_close_time_ms)
```

含义：

- 输入是 trigger candle 的 close time。
- 输出是“截至该 trigger close time，理论上应该已经 closed 的指定高周期 candle close_time”。
- 如果 trigger close time 正好等于高周期 close time，返回该高周期 close time。
- 如果 trigger close time 落在高周期内部，返回上一个已完成高周期 close time。

示例：

```text
trigger = 12:00:00, interval=1h => expected 1h close = 12:00:00
trigger = 12:15:00, interval=1h => expected 1h close = 12:00:00
trigger = 16:00:00, interval=4h => expected 4h close = 16:00:00
trigger = 16:15:00, interval=4h => expected 4h close = 16:00:00
```

评估前检查：

- latest closed `4h` close time must be `>= expected_closed_close_time('4h', trigger_close_time_ms)`。
- latest closed `1d` close time must be `>= expected_closed_close_time('1d', trigger_close_time_ms)`。
- 如果 trigger interval 是 `15m`，latest closed `1h` close time must be `>= expected_closed_close_time('1h', trigger_close_time_ms)`。

如果 barrier 不满足：

- 等待同一 WebSocket batch 中的高周期 message 到达。
- 最多延迟 `3s`。
- 仍不满足时，用 REST snapshot 补齐高周期 closed candle。
- 补齐失败则拒绝本次正式评估，记录 `rejected_reason = 'HIGHER_TIMEFRAME_NOT_READY'`。

这样避免 4h/1d 同时收盘时，低周期先评估而使用旧日线或旧 4h 区间。

必须测试三类边界：

- `15m` candle close time 正好等于 `1h` close time。
- `15m` candle close time 正好等于 `4h` close time。
- `15m` candle close time 落在 `1h / 4h` 内部的非边界时刻。

## 9. 三不铁律

### 9.1 不做逆突破单

如果最近 `1-2` 根已收盘 `15m / 1h` 已经明显突破并且有 follow-through，就不允许反着推。

V1 定义：

- 上破：close 超过关键位 `breakout_distance`。
- 下破：close 跌破关键位 `breakout_distance`。
- follow-through：下一根同周期 closed candle 继续收在突破方向关键位外。

### 9.2 不做趋势对抗单

信号方向和日线趋势相反时，默认拒绝。

唯一例外：

- 满足“逆日线极端急插针例外，仅scalp”全部条件。

### 9.3 不做区间中间单

V1 用最近 `30` 根已收盘 `4h` 的 high/low 定义区间。区间位置用 `mark_price` 计算，因为它是行情展示和风险参考价；正式入场可执行性仍然只看 bid/ask。

```text
range_high = max(high)
range_low = min(low)
range_position = (mark_price - range_low) / (range_high - range_low)
```

过滤规则：

```text
0.40 <= range_position <= 0.60 => 普通信号拒绝
```

急插针例外要求在 4h 极端区：

```text
range_position <= 0.20 or range_position >= 0.80
```

## 10. 日线趋势

EMA 输入：

- `EMA20` 使用 daily close 计算。

趋势分类：

```text
daily_close > EMA20 * 1.002 => daily_bull
daily_close < EMA20 * 0.998 => daily_bear
otherwise => daily_neutral
```

V1 行为：

- `daily_bull`：只优先做多，除急插针 scalp 例外外不做空。
- `daily_bear`：只优先做空，除急插针 scalp 例外外不做多。
- `daily_neutral`：不推正式方向信号，只允许输出观察状态。

`0.2%` buffer 的目的不是追求指标精致，而是避免价格贴 EMA20 时频繁切换方向。

## 11. 关键位定义

V1 关键位来源：

1. 最近 `30` 根 closed `4h` 的 range high / range low。
2. confirmed `4h` pivot high / pivot low。
3. 靠近 4h 边沿时，参考最近 `1h` swing high / low。

`4h` pivot 定义：

- pivot high：某根 closed `4h` 的 high 高于左侧 `2` 根和右侧 `2` 根的 high。
- pivot low：某根 closed `4h` 的 low 低于左侧 `2` 根和右侧 `2` 根的 low。

接近关键位：

```text
near_level_distance = min(0.3% of price, 0.25 * ATR15)
```

有效突破：

```text
breakout_distance = max(0.15% of price, 0.20 * ATR15)
```

`ATR15`：

- 使用最近 `20` 根 closed `15m` 的 true range 平均值。
- 只用来让阈值适配波动，不作为独立交易指标。

### 11.1 Swing、平台、起涨点 / 起跌点公式

为避免主观实现，V1 统一使用以下结构定义。

`15m` swing high：

- candle `i` 的 high 高于左侧 `2` 根和右侧 `2` 根 closed `15m` 的 high。

`15m` swing low：

- candle `i` 的 low 低于左侧 `2` 根和右侧 `2` 根 closed `15m` 的 low。

`1h` swing high/low：

- 同样用 left/right `2` 根 closed `1h` 确认。

小平台 high：

```text
platform_high = max(high of latest N closed 15m candles)
N = 5
platform_range = platform_high - min(low of same N candles)
platform is valid when platform_range <= 1.2 * ATR15
```

小平台 low：

```text
platform_low = min(low of latest N closed 15m candles)
N = 5
platform_range = max(high of same N candles) - platform_low
platform is valid when platform_range <= 1.2 * ATR15
```

起涨点：

- 做空 TP 候选使用。
- 在 entry 前最近 `20` 根 closed `15m` 内，找到最后一个 confirmed swing low。
- 如果没有 confirmed swing low，使用最近 `20` 根 closed `15m` 的最低 low。

起跌点：

- 做多 TP 候选使用。
- 在 entry 前最近 `20` 根 closed `15m` 内，找到最后一个 confirmed swing high。
- 如果没有 confirmed swing high，使用最近 `20` 根 closed `15m` 的最高 high。

Tie-breaker：

- 同方向多个候选 level 同时满足时，优先选择距离 entry 最近且 RR 达标的 level 作为 TP1。
- TP2 选择 TP1 之后距离 entry 最近且 RR 达到 `2.0R` 的更远结构位。
- 如果多个 level 价格相同或距离差小于 `0.05R`，优先级为：4h range edge > 4h pivot > 1h swing > 15m swing/platform。

### 11.2 支撑 / 压力候选公式

顺势回调/反弹必须先命中明确的支撑/压力候选，不能只靠“感觉到位”。

做多 support candidates：

1. 4h range low。
2. 最近 confirmed 4h pivot low。
3. 最近 confirmed 1h swing low。
4. 有效 15m platform low。

做空 pressure candidates：

1. 4h range high。
2. 最近 confirmed 4h pivot high。
3. 最近 confirmed 1h swing high。
4. 有效 15m platform high。

候选过滤：

- 做多只保留 `level <= trigger_close` 的 support candidate。
- 做空只保留 `level >= trigger_close` 的 pressure candidate。
- 同类多个候选时，选距离 trigger close 最近的 level。
- 如果两个候选距离差小于 `0.05 * ATR15`，优先级为：4h range edge > 4h pivot > 1h swing > 15m platform。

靠近支撑：

```text
min(abs(trigger_low - support_level), abs(trigger_close - support_level)) <= near_level_distance
```

支撑未被有效跌破：

```text
no closed 15m candle after first touch has close < support_level - breakout_distance
```

靠近压力：

```text
min(abs(trigger_high - pressure_level), abs(trigger_close - pressure_level)) <= near_level_distance
```

压力未被有效突破：

```text
no closed 15m candle after first touch has close > pressure_level + breakout_distance
```

`first touch`：

- 从最新 trigger candle 往前最多看 `12` 根 closed `15m`。
- 第一根 low/close 进入支撑 near-level 阈值，或 high/close 进入压力 near-level 阈值，即为 first touch。
- 如果最近 `12` 根内没有 first touch，顺势回调/反弹信号拒绝。

## 12. K 线质量定义

只允许用 closed candle。

```text
range = high - low
body = abs(close - open)
upper_wick = high - max(open, close)
lower_wick = min(open, close) - low
```

强阳：

- `close > open`
- `body / range >= 0.55`

强阴：

- `close < open`
- `body / range >= 0.55`

明显长上影：

- `upper_wick >= body * 1.2`
- `close <= low + range * 0.50`

明显长下影：

- `lower_wick >= body * 1.2`
- `close >= low + range * 0.50`

急插针 wick：

- 做多急插针：`lower_wick / range >= 0.55`
- 做空急插针：`upper_wick / range >= 0.55`

Bullish engulfing：

- current candle close > open。
- previous candle close < previous open。
- current close >= previous open。
- current open <= previous close。

Bearish engulfing：

- current candle close < open。
- previous candle close > previous open。
- current close <= previous open。
- current open >= previous close。

`range = 0` 的 K 线不参与 K 线质量判断。

## 13. 信号 A：顺势回调 / 反弹入场

这是主模式。

### 13.1 日线多头做多

必须同时满足：

1. 日线为 `daily_bull`。
2. 按 11.2 命中 support candidate，且支撑未被有效跌破。
3. 不是 4h 区间中间。
4. closed `15m / 1h` 确认回调结束。
5. 短线结构恢复上行。
6. RR 达标。
7. executable price 仍可执行。

短线结构确认至少满足其一：

- 回踩支撑不破后，closed `15m` 收盘价站上 `platform_high + breakout_distance`。
- 出现 higher low + higher high：最近两个 confirmed `15m` swing lows 抬高，且最新 closed `15m` close 突破两者之间的 swing high。
- 强阳或 bullish engulfing 后，下一根 closed `15m` close 高于 signal candle high。
- closed `1h` close 站上最近 confirmed `1h` swing high。

如果只是单根阳线反抽，不推。

### 13.2 日线空头做空

必须同时满足：

1. 日线为 `daily_bear`。
2. 按 11.2 命中 pressure candidate，且压力未被有效突破。
3. 不是 4h 区间中间。
4. closed `15m / 1h` 确认反弹转弱。
5. 短线结构恢复下行。
6. RR 达标。
7. executable price 仍可执行。

短线结构确认至少满足其一：

- higher low 被破坏：closed `15m` close 跌破最近 confirmed `15m` higher low 的 low。
- 出现 lower high + lower low：最近两个 confirmed `15m` swing highs 降低，且最新 closed `15m` close 跌破两者之间的 swing low。
- closed `15m` close 跌破 `platform_low - breakout_distance`，随后最多 `2` 根 closed `15m` 的 high 不重新站上 `platform_low`，视为反抽不过。
- failed breakout 已收盘确认：价格盘中突破阻力，但 closed `15m / 1h` close 回到关键位内侧，且下一根 closed candle 继续向下收在关键位内侧。

如果低点还在持续抬升、趋势线没破，就不能因为“到了压力”提前空。

## 14. 信号 B：突破追随

这是为 SOL 这类强突破补的例外模式，但仍然必须顺日线。

### 14.1 做多突破追随

必须满足：

1. 日线为 `daily_bull`。
2. 有效突破 `4h` 前高或区间上沿。
3. 通过 `15m` 快速确认或 `1h` 稳健确认。
4. 无明显长上影被打回。
5. RR 和 executable price actionability 通过。

`15m` 快速确认：

- 连续 `2` 根 closed `15m` 都收在关键位上方，且距离至少达到 `breakout_distance`。
- 第 `2` 根是强阳。
- 第 `2` 根没有明显长上影。

`1h` 稳健确认：

- 至少 `1` 根 closed `1h` 收在关键位上方，且距离至少达到 `breakout_distance`。
- 这根 `1h` 是阳线。
- follow-through 干净，没有明显长上影。

### 14.2 做空突破追随

镜像处理：

1. 日线为 `daily_bear`。
2. 有效跌破 `4h` 前低或区间下沿。
3. `15m` 双收盘确认或 `1h` 收盘确认。
4. 无明显长下影拉回。
5. RR 和 executable price actionability 通过。

`15m` 快速确认：

- 连续 `2` 根 closed `15m` 都收在关键位下方，且距离至少达到 `breakout_distance`。
- 第 `2` 根是强阴。
- 第 `2` 根没有明显长下影。

`1h` 稳健确认：

- 至少 `1` 根 closed `1h` 收在关键位下方，且距离至少达到 `breakout_distance`。
- 这根 `1h` 是阴线。
- follow-through 干净，没有明显长下影。

## 15. 信号 C：逆日线极端急插针例外

这是唯一允许的逆势例外。

推送文案必须明确写：

```text
逆日线极端急插针例外，仅scalp
```

### 15.1 做多 scalp 例外

必须全部满足：

1. 日线不是多头，正常规则下不该做多。
2. 价格明显刺破关键支撑。
3. closed `15m / 1h` 触发 K 快速收回关键位上方。
4. 触发 K 满足急插针 wick，且为强阳或 bullish engulfing。
5. lower wick 占整根 K 范围至少 `55%`。
6. 刺破距离至少 `max(0.3% of price, 0.5 * ATR15)`。
7. 位置在 4h 下沿极端区，`range_position <= 0.20`。
8. 只定义为 scalp。
9. 按 executable price 重算后，`rr_tp1_executable >= 1.5`。

### 15.2 做空 scalp 例外

镜像处理：

1. 日线不是空头，正常规则下不该做空。
2. 价格明显刺破关键压力。
3. closed `15m / 1h` 触发 K 快速收回关键位下方。
4. 触发 K 满足急插针 wick，且为强阴或 bearish engulfing。
5. upper wick 占整根 K 范围至少 `55%`。
6. 刺破距离至少 `max(0.3% of price, 0.5 * ATR15)`。
7. 位置在 4h 上沿极端区，`range_position >= 0.80`。
8. 只定义为 scalp。
9. 按 executable price 重算后，`rr_tp1_executable >= 1.5`。

普通双底、双顶、逆势反抽、逆势抄底摸顶全部过滤。

## 16. Entry / SL / TP / 可执行性

### 16.1 Entry

V1 使用确认 K 的 close 作为 entry candidate：

- 顺势回调 / 反弹：确认回调结束或反弹转弱的 closed `15m / 1h` close。
- 突破追随：完成确认的 closed candle close。
- 急插针例外：触发 K close。

正式推送必须同时展示：

- mark price：行情参考。
- executable price：做多用 ask，做空用 bid。
- spread：用于判断当前盘口是否可执行。

正式信号的 actionability 一律使用 executable price，不使用 mark price。

### 16.2 Stop Loss

做多：

```text
SL = recent_structure_low - max(0.1% of entry, 0.2 * ATR15)
```

做空：

```text
SL = recent_structure_high + max(0.1% of entry, 0.2 * ATR15)
```

`recent_structure_low/high` 必须来自 closed `15m / 1h`，不能用未收盘 K 的极值。

### 16.3 TP

TP 候选先用 `entry_price` 计算结构可行性；正式推送前必须再用 `executable_price` 重算真实 RR。

做多 TP 候选，从近到远：

1. entry 上方最近 confirmed `15m` swing high。
2. 起跌点。
3. entry 上方最近 confirmed `1h` swing high。
4. entry 上方最近 `4h` pivot high。
5. 4h range high。

做空 TP 候选，从近到远：

1. entry 下方最近 confirmed `15m` swing low。
2. 起涨点。
3. entry 下方最近 confirmed `1h` swing low。
4. entry 下方最近 `4h` pivot low。
5. 4h range low。

TP 选择：

- TP1 先选择第一个真实结构位且 `entry_rr_tp1 >= 1.5R` 的候选。
- TP2 先选择 TP1 之后第一个真实结构位且 `entry_rr_tp2 >= 2.0R` 的候选。
- 如果 TP1 和 TP2 选到同一价格，拒绝信号。
- 如果所有候选都不达标，拒绝信号。

Entry close 视角 RR：

```text
entry_risk = abs(entry_price - stop_loss)
entry_rr_tp1 = abs(tp1 - entry_price) / entry_risk
entry_rr_tp2 = abs(tp2 - entry_price) / entry_risk
```

Executable price 视角 RR：

```text
actual_risk = abs(executable_price - stop_loss)
rr_tp1_executable = abs(tp1 - executable_price) / actual_risk
rr_tp2_executable = abs(tp2 - executable_price) / actual_risk
```

正式推送要求：

```text
rr_tp1_executable >= 1.5
rr_tp2_executable >= 2.0
```

如果 entry close 视角达标但 executable price 视角不达标，不推正式信号，只允许输出 `[观察中] 盘口价导致RR不足`。不要为了凑 RR 编一个没有结构意义的目标。

### 16.4 Executable price 可执行性

因为 WebSocket 价格可能已经离确认 K close 很远，正式推送前要检查是否还能按盘口入场。

基础 freshness：

- book ticker event time 距离当前时间不能超过 `5s`。
- mark price event time 距离当前时间不能超过 `10s`。
- `spread_pct <= min(0.10%, 0.10R / executable_price)`，否则认为盘口过宽。

做多：

- `executable_price = ask_price`。
- 如果 ask 已经比 entry 高出 `0.5R` 以上，不推正式入场。
- 如果 ask 距离 TP1 已经小于 `0.25R`，不推正式入场。
- 如果按 ask 重算的 `rr_tp1_executable < 1.5` 或 `rr_tp2_executable < 2.0`，不推正式入场。

做空：

- `executable_price = bid_price`。
- 如果 bid 已经比 entry 低出 `0.5R` 以上，不推正式入场。
- 如果 bid 距离 TP1 已经小于 `0.25R`，不推正式入场。
- 如果按 bid 重算的 `rr_tp1_executable < 1.5` 或 `rr_tp2_executable < 2.0`，不推正式入场。

可以输出观察状态：

```text
信号已确认，但 executable price 已经偏离 entry，等待回踩，不作为当前入场推送。
```

但不能把错过的行情包装成可入场信号。

## 17. 推送格式

正式信号必须包含：

- signal type
- symbol
- direction
- 检查时间，北京时间
- mark price
- executable price，做多显示 ask，做空显示 bid
- bid/ask spread
- TradFi symbol 当前 raw session type 和 actionable status
- 最近已收 `15m` 时间和 close
- 最近已收 `1h` 时间和 close
- 日线趋势和 EMA20 关系
- 关键 `4h` level 和 range position
- entry
- SL
- TP1 / TP2
- RR
- 失效条件
- 简短 reason chain

示例：

```text
[正式信号] SOLUSDT 做多 - 突破追随
检查时间: 2026-07-03 15:30:02 北京时间
实时价: 150.42 mark
可执行价: 149.95 ask, spread=0.02%
Session: raw=TRADING_24_7, actionable=ACTIONABLE
15m依据: 2026-07-03 15:15 close=149.88
1h依据: 2026-07-03 15:00 close=149.10

日线: close 在 EMA20 上方，多头
4h位置: 突破区间上沿 148.90，range position=0.86
触发: 连续2根15m收在关键位上方，第2根强阳且无明显上影

Entry: 149.88
SL: 147.90
TP1: 153.10 (entry 1.63R, executable 1.54R)
TP2: 154.20 (entry 2.18R, executable 2.10R)
失效: 15m收回148.90下方，或 executable price 已偏离 entry 超过0.5R
```

观察状态不能长得像交易信号：

```text
[观察中] SOLUSDT 正在测试4h区间上沿，但15m还没收盘确认。
```

## 18. 去重和 cooldown

去重 key：

```text
symbol + direction + signal_type + rounded_key_level
```

`rounded_key_level`：

- 优先使用 `exchangeInfo` 的 `tickSize`。
- `rounded_key_level = round(key_level / bucket_size) * bucket_size`。
- `bucket_size = max(tickSize * 10, ATR15 * 0.05)`。
- 所有 rounding 使用 `Decimal`，最终以 normalized decimal string 写入 dedupe key。

数据库硬约束：

- `signal_candidates` 必须有 `UNIQUE(dedupe_key, basis_open_time_ms)`。
- `signal_alerts` 必须有 `UNIQUE(dedupe_key, basis_open_time_ms)`。

V1 cooldown：

- 同一个 key，`60` 分钟内只推一次。
- 如果价格重新回到结构内，并重新完成确认，才允许新信号。

状态流：

```text
candidate -> confirmed -> alerted -> expired
candidate -> rejected
candidate -> missed_observation
alerted -> invalidated
alerted -> tp1_reached
alerted -> tp2_reached
```

V1 不需要完整自动持仓系统，但要记录足够状态避免重复推送。

## 19. Health 和异常处理

### 19.1 Health 输出

至少包含：

- market WebSocket connected
- public WebSocket connected
- last market WebSocket event time
- last public WebSocket event time
- 每个 symbol 最近 closed `15m` 时间
- 每个 symbol 最近 closed `1h` 时间
- 每个 symbol 最近 mark price 时间
- 每个 symbol 最近 book ticker 时间
- 每个 symbol 当前 contract type
- 每个 TradFi symbol 当前 raw session type
- 每个 TradFi symbol 当前 actionable session status
- 每个 TradFi symbol 当前 session close time
- enabled / disabled symbol 数量
- reconnect count
- last REST backfill time
- notifier status

### 19.2 stale data 规则

symbol 标记为 stale：

- mark price 超过 `10s` 没更新。
- book ticker 超过 `5s` 没更新。
- 对 `PERPETUAL` crypto symbol：closed `15m` 比预期 close 晚超过 `20m`。
- 对 `PERPETUAL` crypto symbol：closed `1h` 比预期 close 晚超过 `70m`。
- 对 `TRADIFI_PERPETUAL`：只有 `actionable_session_status = ACTIONABLE` 时才按 candle close 延迟判 stale。
- 对 `TRADIFI_PERPETUAL`：`actionable_session_status = NOT_ACTIONABLE` 时没有新 candle 不算 stale，但该 symbol 处于 not actionable。
- trading session stream 超过 `60s` 没更新且 REST schedule 无法确认当前 session 时，session unknown，不发正式信号。

stale symbol 不允许发正式信号。

### 19.3 错误处理

边界错误必须显式可见：

- REST 失败：记录错误，health 展示，backoff retry。
- WebSocket disconnect：重连，REST backfill 后再恢复信号。
- `/market` 或 `/public` 任何一条连接断开：该连接相关数据视为 stale；正式信号暂停。
- WebSocket message 格式异常：记录 stream name，跳过该消息。
- notifier 失败：保存未发送 alert，后续 retry。
- symbol 不可用：禁用 symbol，health 展示原因。
- trading session unknown：暂停 TradFi symbol 正式信号，不影响 crypto symbol。

不允许 catch 后只 log 不影响状态。数据不确定时，宁可不推。

## 20. 配置建议

`config/strategy.yaml`：

```yaml
symbols:
  SOLUSDT:
    expected_contract_type: PERPETUAL
  PAXGUSDT:
    expected_contract_type: PERPETUAL
  CLUSDT:
    expected_contract_type: TRADIFI_PERPETUAL
  SPCXUSDT:
    expected_contract_type: TRADIFI_PERPETUAL
  SOXLUSDT:
    expected_contract_type: TRADIFI_PERPETUAL
  MUUSDT:
    expected_contract_type: TRADIFI_PERPETUAL
  SKHYNIXUSDT:
    expected_contract_type: TRADIFI_PERPETUAL

intervals:
  trend: 1d
  range: 4h
  structure: 1h
  trigger: 15m

trend:
  ema_period: 20
  ema_buffer_pct: 0.002

range:
  lookback_4h: 30
  middle_min: 0.40
  middle_max: 0.60
  extreme_low_max: 0.20
  extreme_high_min: 0.80

levels:
  pivot_left: 2
  pivot_right: 2
  platform_lookback_15m: 5
  platform_max_range_atr_mult: 1.2
  near_level_pct: 0.003
  near_level_atr_mult: 0.25
  breakout_pct: 0.0015
  breakout_atr_mult: 0.20

candle_quality:
  strong_body_ratio: 0.55
  rejection_wick_body_mult: 1.2
  spike_wick_range_ratio: 0.55

atr:
  interval: 15m
  period: 20

risk:
  stop_buffer_pct: 0.001
  stop_buffer_atr_mult: 0.20
  tp1_min_rr: 1.5
  tp2_min_rr: 2.0
  max_executable_entry_drift_r: 0.5
  min_tp1_remaining_r: 0.25
  max_book_ticker_age_seconds: 5
  max_mark_price_age_seconds: 10
  max_spread_pct: 0.001

dedupe:
  cooldown_minutes: 60

session:
  tradifi_requires_trading_session: true
  trading_session_stale_seconds: 60
  actionable_raw_session_types:
    - REGULAR
  non_actionable_raw_session_types:
    - PRE_MARKET
    - AFTER_MARKET
    - OVERNIGHT
    - NO_TRADING
  unknown_session_blocks_formal_signal: true

backfill:
  allow_recovery_alerts: true
  recovery_requires_basis_after_disconnect: true
  recovery_alert_prefix: "[恢复后确认]"

evaluation:
  higher_timeframe_barrier_delay_seconds: 3

decimal:
  use_decimal_for_calculation: true
  store_decimal_as_text: true
```

这些参数是策略定义，不是随手调的工程配置。改这些值会直接改变用户收到什么信号。

## 21. 代码结构建议

如果当前仓库作为实现仓库，建议新增：

```text
src/futures_monitor/
  __init__.py
  app.py
  config.py
  binance_rest.py
  binance_ws.py
  candles.py
  indicators.py
  levels.py
  sessions.py
  pricing.py
  strategy.py
  risk.py
  storage.py
  notifier.py
  health.py

tests/
  test_config.py
  test_candles.py
  test_indicators.py
  test_levels.py
  test_sessions.py
  test_pricing.py
  test_strategy_filters.py
  test_strategy_breakout.py
  test_strategy_spike_exception.py
  test_risk.py
  test_dedupe.py
  test_higher_timeframe_barrier.py
  test_reconnect_backfill.py
  test_websocket_parsing.py
  test_storage.py
  test_health.py
```

职责：

- `config.py`：加载并校验策略配置。
- `binance_rest.py`：REST bootstrap、snapshot、backfill。
- `binance_ws.py`：WebSocket 连接、解析、重连。
- `candles.py`：标准 candle model，closed/forming 边界。
- `indicators.py`：EMA20、ATR15。
- `levels.py`：4h range、pivot、key levels。
- `sessions.py`：contract type、tradingSession、tradingSchedule、session gate。
- `pricing.py`：mark price、bookTicker、executable price、spread/freshness。
- `strategy.py`：hard filters、信号分类、reason chain。
- `risk.py`：entry、SL、TP、RR、executable price actionability。
- `storage.py`：SQLite persistence。
- `notifier.py`：推送格式、发送、失败 retry。
- `health.py`：freshness、stale、disabled symbol、运行状态。

## 22. 实施任务拆分

### Task 1：项目骨架和配置

**Files:**

- Create: `pyproject.toml`
- Create: `config/strategy.yaml`
- Create: `src/futures_monitor/config.py`
- Create: `tests/test_config.py`

- [ ] 定义 Python package 和依赖。
- [ ] 写入 7 个 symbol 和 V1 参数。
- [ ] 写入每个 symbol 的 expected `contractType`。
- [ ] 写入 WebSocket route 规则：kline/markPrice/tradingSession 走 `/market`，bookTicker 走 `/public`。
- [ ] 写入 raw session type 到 actionable status 的映射。
- [ ] 写入 Decimal/tickSize 计算规则。
- [ ] 实现配置加载和必填项校验。
- [ ] 测试缺失配置会明确失败。

### Task 2：Candle / Indicator / Level

**Files:**

- Create: `src/futures_monitor/candles.py`
- Create: `src/futures_monitor/indicators.py`
- Create: `src/futures_monitor/levels.py`
- Create: `tests/test_candles.py`
- Create: `tests/test_indicators.py`
- Create: `tests/test_levels.py`

- [ ] 实现标准 candle model。
- [ ] 确保 forming candle 不能被当作 closed candle。
- [ ] 实现 daily close EMA20。
- [ ] 实现 ATR15。
- [ ] 实现 4h range position。
- [ ] 实现 pivot high / pivot low。
- [ ] 实现 `15m`/`1h` swing high / swing low。
- [ ] 实现 5 根 `15m` 小平台 high/low。
- [ ] 实现起涨点 / 起跌点和 TP tie-breaker。
- [ ] 实现 support/pressure candidates 和 near-level first touch 规则。

### Task 3：Session 和 Pricing

**Files:**

- Create: `src/futures_monitor/sessions.py`
- Create: `src/futures_monitor/pricing.py`
- Create: `tests/test_sessions.py`
- Create: `tests/test_pricing.py`

- [ ] 解析 `exchangeInfo` 的 `contractType`。
- [ ] `PERPETUAL` crypto symbol 标记为 `TRADING_24_7`。
- [ ] `TRADIFI_PERPETUAL` 必须读取 `tradingSchedule` 和 `tradingSession`。
- [ ] 保存 Binance raw session type。
- [ ] 将 `REGULAR` 归一化为 `ACTIONABLE`。
- [ ] 将 `PRE_MARKET` / `AFTER_MARKET` / `OVERNIGHT` / `NO_TRADING` 归一化为 `NOT_ACTIONABLE`。
- [ ] unknown session 阻断正式信号。
- [ ] 做多 executable price 使用 ask。
- [ ] 做空 executable price 使用 bid。
- [ ] book ticker 过期或 spread 过宽时阻断正式信号。
- [ ] 所有价格计算使用 `Decimal`，存储使用 decimal string。

### Task 4：REST bootstrap / backfill

**Files:**

- Create: `src/futures_monitor/binance_rest.py`
- Create: `tests/test_reconnect_backfill.py`

- [ ] 用 `exchangeInfo` 校验 symbol。
- [ ] 校验当前 `contractType` 和配置里的 expected type 一致，不一致时 health 报警。
- [ ] 拉 `tradingSchedule` 初始化 TradFi session。
- [ ] 拉各周期历史 klines。
- [ ] 丢弃未收盘 REST candle。
- [ ] 拉 mark price 和 book ticker snapshot。
- [ ] reconnect 后按 last closed open time 补齐缺口。
- [ ] backfill candle 默认只更新状态。
- [ ] 只有 `basis_close_time_ms >= disconnect_started_at_ms` 且 dedupe 未命中时，才允许恢复补发。

### Task 5：WebSocket runtime

**Files:**

- Create: `src/futures_monitor/binance_ws.py`
- Create: `tests/test_websocket_parsing.py`

- [ ] 构造 `/market/stream` URL，包含 kline、markPrice、tradingSession。
- [ ] 构造 `/public/stream` URL，包含 bookTicker。
- [ ] 测试不会生成 legacy `/stream` URL。
- [ ] 解析 kline、mark price、book ticker event。
- [ ] 解析 tradingSession event。
- [ ] `k.x=false` 只更新 forming state。
- [ ] `k.x=true` 写入 closed candle 并触发 strategy evaluation。
- [ ] 重连后先 REST backfill，再恢复信号。

### Task 6：Strategy engine

**Files:**

- Create: `src/futures_monitor/strategy.py`
- Create: `tests/test_strategy_filters.py`
- Create: `tests/test_strategy_breakout.py`
- Create: `tests/test_strategy_spike_exception.py`
- Create: `tests/test_higher_timeframe_barrier.py`

- [ ] 实现 daily trend 分类。
- [ ] 实现 session gate。
- [ ] 实现 higher timeframe barrier。
- [ ] 实现三不铁律。
- [ ] 实现顺势回调 / 反弹信号。
- [ ] 实现突破追随信号。
- [ ] 实现极端急插针 scalp 例外。
- [ ] 测试普通双底 / 双顶不会单独触发信号。

### Task 7：Risk / Actionability / Dedupe

**Files:**

- Create: `src/futures_monitor/risk.py`
- Create: `tests/test_risk.py`
- Create: `tests/test_dedupe.py`

- [ ] 用确认 K close 计算 entry candidate。
- [ ] 用 closed `15m / 1h` 结构点加 buffer 放 SL。
- [ ] 选择有结构意义的 TP1 / TP2。
- [ ] RR 不达标时拒绝。
- [ ] 用 executable price 重算 `rr_tp1_executable` / `rr_tp2_executable`。
- [ ] ask/bid executable price 已经偏离时拒绝正式入场推送。
- [ ] spread 过宽时拒绝正式信号。
- [ ] 实现 60 分钟 cooldown。

### Task 8：Storage / Alert / Health

**Files:**

- Create: `src/futures_monitor/storage.py`
- Create: `src/futures_monitor/notifier.py`
- Create: `src/futures_monitor/health.py`
- Create: `tests/test_storage.py`
- Create: `tests/test_health.py`

- [ ] 持久化 closed candles、live prices、symbol sessions、signal candidates、signal alerts、system state。
- [ ] 格式化正式信号。
- [ ] 正式信号展示 mark price、executable price、spread、session。
- [ ] 格式化观察状态，避免看起来像交易信号。
- [ ] notifier 失败时保存并 retry。
- [ ] health 输出 market/public WebSocket freshness、session、reconnect、disabled symbol。

### Task 9：Dry-run 验证

**Files:**

- Create: `scripts/run_monitor.py`
- Create: `scripts/dump_status.py`
- Create: `docs/five-minute-monitor-runbook.md`

- [ ] dry-run 启动，不发真实交易推送。
- [ ] 验证 7 个 symbol 都收到 mark price。
- [ ] 验证 7 个 symbol 都收到 book ticker。
- [ ] 验证 TradFi symbol 有 session 状态。
- [ ] 验证 4 个周期都能收到 closed candle。
- [ ] 人为重启，确认 REST backfill 补齐缺口。
- [ ] 抽样比对生成的 reason chain 和实际图表。

## 23. 测试策略

实现前就应该准备 fixture，避免策略变成“肉眼觉得差不多”。

最低测试集：

1. WebSocket route 生成：kline/markPrice/tradingSession 必须走 `/market/stream`，bookTicker 必须走 `/public/stream`。
2. `exchangeInfo` contract type 解析：`TRADIFI_PERPETUAL` 不按 24/7 处理。
3. `tradingSchedule` raw session type 保存原始值：`REGULAR / PRE_MARKET / AFTER_MARKET / OVERNIGHT / NO_TRADING`。
4. `REGULAR` 归一化为 `ACTIONABLE`，其他已知 raw type 归一化为 `NOT_ACTIONABLE`。
5. `NOT_ACTIONABLE` / `UNKNOWN` session 阻断正式信号。
6. 非交易时段 TradFi candle 不更新时不误报 stale。
7. Decimal 计算和 decimal string 存储，不用 SQLite REAL 作为价格复盘依据。
8. EMA20 趋势分类。
9. 4h range position 和中间区域过滤。
10. pivot high / pivot low。
11. 5 根 `15m` platform high/low。
12. `15m` / `1h` swing high/low。
13. support/pressure candidate near-level 和 first touch。
14. strong candle、rejection wick、bullish/bearish engulfing。
15. PAXG 类“到了压力但结构没转弱”拒绝。
16. SOL 类顺日线突破追随通过。
17. 普通双底 / 双顶不触发逆势单。
18. 极端急插针 scalp 例外通过。
19. TP1 / TP2 entry RR 不够时拒绝。
20. executable RR 不够时拒绝正式信号。
21. ask/bid executable price 已经跑远时拒绝正式信号。
22. book ticker stale 或 spread 过宽时拒绝正式信号。
23. `k.x=false` 永不触发正式信号。
24. 4h/1d 同时收盘时，higher timeframe barrier 不满足会延迟或 REST 补齐。
25. `expected_closed_close_time` 覆盖 15m=1h 边界、15m=4h 边界、非边界三类测试。
26. reconnect 期间缺失 closed candle 后 REST backfill 能补齐。
27. backfill 只补发 `basis_close_time_ms >= disconnect_started_at_ms` 且 dedupe 未命中的恢复信号。

上线前至少跑一轮 dry-run：

- 持续运行至少一个完整 `1h` 周期。
- 检查所有 symbol 的 WebSocket price freshness。
- 检查 `/market` 和 `/public` 两条连接分别健康。
- 检查 TradFi symbol 在 `NO_TRADING` 时不会发正式信号。
- 检查 closed `15m / 1h` 是否按预期入库。
- 人工复核至少 3 条 rejected reason 和 1 条 candidate reason。

## 24. 需要你 review 的策略参数

这些值建议先按 V1 保守版定下来：

| 参数 | V1 默认值 |
| --- | --- |
| Daily EMA20 neutral buffer | `0.2%` |
| 4h range lookback | `30` 根 closed `4h` |
| 区间中间过滤 | `40%-60%` |
| 极端区 | `<20%` 或 `>80%` |
| Pivot confirmation | left/right `2` candles |
| Near-level threshold | `min(0.3%, 0.25 ATR15)` |
| Breakout threshold | `max(0.15%, 0.20 ATR15)` |
| Strong candle body ratio | `55%` |
| Rejection wick threshold | `1.2x body` |
| Spike wick ratio | `55%` |
| Spike distance | `max(0.3%, 0.5 ATR15)` |
| Stop buffer | `max(0.1%, 0.2 ATR15)` |
| Max live entry drift | `0.5R` |
| Min TP1 remaining | `0.25R` |
| Max book ticker age | `5s` |
| Max mark price age | `10s` |
| Max spread | `0.10%` |
| Dedupe cooldown | `60 minutes` |
| Higher timeframe barrier delay | `3s` |
| Backfill recovery window | `basis_close_time >= disconnect_started_at` |
| Actionable raw session type | `REGULAR` |
| Non-actionable raw session types | `PRE_MARKET / AFTER_MARKET / OVERNIGHT / NO_TRADING` |
| Dedupe level bucket | `max(tickSize * 10, ATR15 * 0.05)` |

重点 review：

1. `daily_neutral` 是否完全不推正式信号。
2. 4h range lookback 用 `30` 根是否太短或太长，尤其是 TradFi session-aware symbol。
3. 突破确认阈值 `0.15% / 0.20 ATR15` 是否太严。
4. executable price 偏离 `0.5R` 后不推，是否符合你的交易习惯。
5. missed setup 是否要发观察消息，还是直接静默。
6. `PRE_MARKET / AFTER_MARKET / OVERNIGHT / NO_TRADING` 是否都不发正式信号。

## 25. 开放问题

1. V1 notifier 是先 console/file dry-run，还是直接接 Telegram？
2. `daily_neutral` 是否完全禁止正式信号？
3. missed but confirmed setup 要不要推观察消息？
4. 非 `REGULAR` session 期间要不要推“非正式交易时段”状态消息，还是只体现在 health？
5. TP1 到后“锁半仓、SL 提保本”是只写在文案里，还是后续需要做成状态跟踪？

## 26. Definition of Done

V1 完成标准：

- WebSocket 使用 `/market/stream` 和 `/public/stream`，不使用 legacy `/stream`。
- `/market` 能为所有启用 symbol 持续更新 kline、mark price，并维护 tradingSession。
- `/public` 能为所有启用 symbol 持续更新 book ticker。
- REST 能在启动、重启、断线后补齐 closed candles。
- TradFi symbol 保存 Binance raw session type，并归一化为 `ACTIONABLE / NOT_ACTIONABLE / UNKNOWN`。
- TradFi symbol 只有 raw session type 为 `REGULAR` 时才允许正式信号。
- 正式信号使用 executable price 做 actionability：做多 ask，做空 bid。
- 正式信号用 executable price 重算 actual RR，`rr_tp1_executable >= 1.5` 且 `rr_tp2_executable >= 2.0`。
- 价格、RR、spread 计算使用 Decimal，存储使用 decimal string。
- 任何正式信号都不使用 unclosed candle。
- 推送包含北京时间、mark price、executable price、spread、session、最近已收 `15m`、最近已收 `1h`。
- 同一收盘时刻不会因为 4h/1d message 晚到而用旧高周期状态评估。
- 三不铁律能过滤逆突破、趋势对抗、4h 中间区域。
- 顺势回调/反弹必须命中公式化 support/pressure candidate。
- 顺势回调 / 反弹、突破追随、极端急插针例外都有测试覆盖。
- RR 和 executable price actionability 有测试覆盖。
- 同类信号不会重复刷屏。
- stale data 和 disabled symbol 在 health 中可见。
