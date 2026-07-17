# Deribit

Deribit 成立于 2016 年,是一个专注于期权、期货、永续、现货和组合标的
的加密货币衍生品交易所。按成交量计,它是最大的加密期权交易所之一,也是
加密衍生品交易的领先平台。

该集成支持接入 Deribit 的实时市场数据与订单执行。

## 概览

该适配器以 Rust 实现,并提供可选的 Python 绑定,供基于 Python 的
工作流使用。Deribit 在 HTTP 和 WebSocket 两种传输方式上都使用
JSON-RPC 2.0。订阅和实时数据优先使用 WebSocket。

Deribit 官方 API 参考文档可在 [docs.deribit.com](https://docs.deribit.com/) 找到。

Deribit 适配器包含多个组件,视具体使用场景而定,可以组合使用,也可以
单独使用:

- `DeribitHttpClient`:底层 HTTP API 连接(HTTP 上的 JSON-RPC)。
- `DeribitWebSocketClient`:底层 WebSocket API 连接(WebSocket 上的
  JSON-RPC)。
- `DeribitInstrumentProvider`:标的解析与加载功能。
- `DeribitDataClient`:市场数据流管理器。
- `DeribitExecutionClient`:账户管理与交易执行网关。
- `DeribitDataClientFactory`:Deribit 数据客户端工厂(供实时节点
  构建器使用)。
- `DeribitExecutionClientFactory`:Deribit 执行客户端工厂(供实时
  节点构建器使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),不必直接与这些底层
组件打交道。
:::

### 产品支持

| 产品类型      | 数据源 | 交易 | 备注                                          |
|-------------------|-----------|---------|------------------------------------------------|
| 永续期货 | ✓         | ✓       | 通过 `DeribitProductType.FUTURE` 加载。       |
| 到期期货     | ✓         | ✓       | 通过 `DeribitProductType.FUTURE` 加载。       |
| 期权           | ✓         | ✓       | 通过 `DeribitProductType.OPTION` 加载。       |
| 现货              | ✓         | ✓       | 通过 `DeribitProductType.SPOT` 加载。         |
| 期货组合     | ✓         | ✓       | 通过 `DeribitProductType.FUTURE_COMBO` 加载。 |
| 期权组合     | ✓         | ✓       | 通过 `DeribitProductType.OPTION_COMBO` 加载。 |

## 符号规则

Deribit 对不同标的类型使用特定的符号约定。引用标的 ID 时,都应包含
`.DERIBIT` 后缀(例如,BTC 永续为 `BTC-PERPETUAL.DERIBIT`)。

### 数量单位

Nautilus 的数量映射到 Deribit 的 `amount` 字段,而非可选的
`contracts` 字段。Deribit 以 USD 为单位报告永续合约和反向期货的数量,
以标的基础货币报告期权和正向期货的数量。Deribit 的 `contract_size`
字段用于在 `amount` 和合约数量之间转换;适配器不会将其再次作为
Nautilus 的乘数应用。

### 永续期货

格式:`{Currency}-PERPETUAL`

示例:

- `BTC-PERPETUAL` - 比特币永续互换。
- `ETH-PERPETUAL` - 以太坊永续互换。

在策略中订阅 BTC 永续:

```python
InstrumentId.from_str("BTC-PERPETUAL.DERIBIT")
```

### 到期期货

格式:`{Currency}-{DDMMMYY}`

示例:

- `BTC-25DEC26` - 2026 年 12 月 25 日到期的比特币期货。
- `ETH-26MAR27` - 2027 年 3 月 26 日到期的以太坊期货。

```python
InstrumentId.from_str("BTC-25DEC26.DERIBIT")
```

### 期权

格式:`{Currency}-{DDMMMYY}-{Strike}-{Type}`

示例:

- `BTC-25DEC26-100000-C` - 比特币看涨期权,行权价 $100,000,2026 年
  12 月 25 日到期。
- `BTC-25DEC26-80000-P` - 比特币看跌期权,行权价 $80,000,2026 年
  12 月 25 日到期。
- `ETH-26MAR27-4000-C` - 以太坊看涨期权,行权价 $4,000,2027 年 3 月
  26 日到期。

其中:

- `C` = 看涨期权。
- `P` = 看跌期权。

```python
InstrumentId.from_str("BTC-25DEC26-100000-C.DERIBIT")
```

### 现货

格式:`{BaseCurrency}_{QuoteCurrency}`

示例:

- `BTC_USDC` - 比特币兑 USDC。
- `ETH_USDC` - 以太坊兑 USDC。

```python
InstrumentId.from_str("BTC_USDC.DERIBIT")
```

### 期货组合

格式:`{Currency}-FS-{LegA}_{LegB}`

腿(legs)可以是到期期货或永续合约(在组合名称内表示为 `PERP`,尽管
独立标的的名称是 `BTC-PERPETUAL`)。组合的到期日与其最早到期的腿相同。

示例:

- `BTC-FS-25DEC26_PERP` - 2026 年 12 月合约与永续合约之间的日历价差。
- `BTC-FS-26MAR27_25DEC26` - 两个到期期货之间的跨月价差。

```python
InstrumentId.from_str("BTC-FS-25DEC26_PERP.DERIBIT")
```

适配器将期货组合建模为 `CryptoFuturesSpread`,以美元计价为各腿之间的
价差,结算货币为加密货币,`is_inverse` 根据上游的 `instrument_type`
设置。

### 期权组合

格式:`{Currency}-{Strategy}-{DDMMMYY}-{Strikes}`

策略代码包括 CS(牛熊价差)、PS(看跌价差)、STRG(宽跨式)、
STRD(跨式)、BOX(箱式)和 RR(风险逆转)。行权价部分用 `_` 分隔
多个行权价。

示例:

- `BTC-CS-25DEC26-70000_75000` - 2026 年 12 月 25 日到期的 70k / 75k
  看涨价差。
- `BTC-STRG-26MAR27-72000_80000` - 2027 年 3 月 26 日到期的 72k / 80k
  宽跨式。
- `BTC-STRD-26MAR27-77000` - 2027 年 3 月 26 日到期的 77k 跨式。
- `BTC-BOX-26MAR27-58000_60000` - 2027 年 3 月 26 日到期的 58k / 60k
  箱式。

```python
InstrumentId.from_str("BTC-STRG-26MAR27-72000_80000.DERIBIT")
```

适配器将期权组合建模为 `CryptoOptionSpread`,按 Deribit 的反向期权
惯例以基础货币计价;小数形式的 `size_increment`(例如 `0.1`)会被
端到端保留。

## 活跃交易的到期日

Deribit 通过 `public/get_expirations` HTTP 端点暴露当前活跃交易的
到期日。期权链加载器可以使用高层 HTTP 客户端刷新活跃期权系列,而无需
扫描每一个标的。

```rust tab="Rust"
use nautilus_deribit::http::models::DeribitCurrency;

let expirations = client
    .request_option_expirations(DeribitCurrency::BTC)
    .await?;
```

```python tab="Python"
from nautilus_trader.adapters.deribit import DeribitCurrency
from nautilus_trader.adapters.deribit import DeribitHttpClient

client = DeribitHttpClient()
expirations = await client.request_option_expirations(DeribitCurrency.BTC)
```

这个高层方法仅返回期权到期日。对于更底层的 Rust 请求,请使用
`GetExpirationsParams` 调用 `client.inner().get_expirations(...)`。
对于 `BTC` 这样的具体货币,Deribit 返回按货币分组的结果;对于
`currency=any`,返回直接按类型分组的结果;适配器同时处理这两种格式。

## 组合标的

当 `product_types` 包含期货组合或期权组合枚举值时,标的提供者会加载
组合。在 Python 中,使用 `DeribitProductType.FUTURE_COMBO` 或
`DeribitProductType.OPTION_COMBO`。Deribit 在 `/public/get_combos`
上暴露每个活跃组合的腿构成,并在标准的
`/public/get_instruments?kind=option_combo|future_combo` 响应中提供
组合的交易元数据(最小变动单位、合约规模、到期日、最小交易数量)。

### 成交发布

Deribit 会将每笔组合成交发布两次:

- 在组合的成交频道上(`trades.{combo_name}.{interval}`):父级成交
  以及一个描述各腿成交的 `legs[]` 数组。
- 在每条腿的成交频道上(`trades.{leg_instrument}.{interval}`):该腿
  的一个独立成交,标记有指向父级的 `combo_id` 和 `combo_trade_id`。

因此,订阅普通期权或期货的用户会在其现有成交流上看到源自组合的成交,
而订阅组合本身的用户则会看到组合层面的成交。适配器不会将组合父消息
拆分成额外的腿 tick;它会将上游的父消息和各腿消息作为独立的
`TradeTick` 转发给各自对应的 `InstrumentId`,因此同时订阅组合和某个
标的腿的用户,针对该笔组合成交会看到一条组合 tick 加一条腿 tick,
而不是针对同一标的的重复 tick。

要让 Deribit 数据客户端在订阅组合成交的同时,也打开真正的腿成交频道,
请将 `params={"subscribe_combo_legs": True}` 传给
`subscribe_trade_ticks`。取消订阅该组合成交流时,Nautilus 也会关闭
该选项所打开的腿订阅。

Deribit 已经按腿发布了大宗交易(block trades)和 Block RFQ,因此
适配器通过标准的 1:1 成交路径转发它们。参见
[成交 ID 来源](#成交-id-来源)了解大宗和 RFQ 来源的成交在生成的
`TradeTick` 上是如何标记的。

### 历史组合成交

标准的按标的成交端点接受组合标的名称。要在一次调用中扫描给定产品类型
的所有组合,请通过 `DeribitHttpClient::inner()` 使用
`get_last_trades_by_currency`:

```rust
use nautilus_deribit::http::{
    models::{DeribitCurrency, DeribitProductType},
    query::GetLastTradesByCurrencyParams,
};

let params = GetLastTradesByCurrencyParams::builder()
    .currency(DeribitCurrency::BTC)
    .kind(DeribitProductType::FutureCombo)
    .count(50_u32)
    .include_old(true)
    .build()?;
let resp = client.inner().get_last_trades_by_currency(params).await?;
```

每个返回的 `DeribitPublicTrade` 携带 `legs: Option<Vec<DeribitTradeLeg>>`,
以及用于关联各腿成交的 `combo_id` 和 `combo_trade_id` 字段。

## 成交 ID 来源

当成交源自 Block RFQ、大宗交易或组合时,适配器发出的公开
`TradeTick` 会在场所成交 ID 前加上前缀。需要将这些成交与普通成交区分
开的策略,可以对 `TradeTick.trade_id` 的前缀进行模式匹配。原始的
Deribit `trade_id` 会保留在前缀之后,因此与 Deribit 自身 ID 的对账
只需去除前缀即可。

| 前缀       | 来源字段     | 含义                                                        |
|--------------|------------------|----------------------------------------------------------------|
| `RFQ-`       | `block_rfq_id`   | 成交源自 Block RFQ。                             |
| `BLK-`       | `block_trade_id` | 成交是非 RFQ 的大宗交易。                                  |
| `COMBO-`     | `combo_id`       | 父级成交源自组合标的的按腿成交。 |
| *无前缀* | (以上皆非)  | 标准成交。                                                |

当多个标签同时存在时的优先级:`RFQ-` > `BLK-` > `COMBO-`。Block RFQ
本身在 Deribit 上就是大宗交易,因此 RFQ 标签优先;以大宗交易方式执行
的组合成交标记为 `BLK-`,因为大宗交易流程是更重要的对账信号。

这仅适用于公开成交(`TradeTick`)。`FillReport.trade_id` 未受影响,
因此针对 `get_user_trades_*` 的对账仍然有效。

:::note
这是一个单向约定。此版本之前捕获的回放数据没有前缀。跨版本存储和比较
`trade_id` 字符串的策略,应在新数据一侧去除前缀,或仅对已知在升级后
捕获的数据按前缀过滤。
:::

## 订单簿订阅

Deribit 提供两种类型的订单簿数据流,各自适用于不同的场景。

### 原始数据流(逐笔)

原始频道将每一次更新作为单独的消息推送。订阅原始订单簿,你会收到订单簿
中每一次插入、更新或删除的通知。

- 需要已认证的连接(防止滥用的保护措施)。
- 当你需要 HFT 或做市所需的每一档价格变动时使用。
- 消息量较高。

### 聚合数据流(批量)

聚合频道以固定间隔(例如每 100ms)批量推送更新。这会将多次订单簿变化
归并到单条消息中。

- 无需身份验证即可使用。
- 适用于大多数使用场景。
- 消息量较低,更易处理。
- 未认证情况下的默认间隔:100ms。

### 订阅参数

Nautilus 适配器通过订阅参数支持这两种数据流类型:

| 参数  | 取值                 | 备注                                                                     |
|------------|------------------------|-----------------------------------------------------------------------------------|
| `interval` | `raw`、`100ms`、`agg2` | `agg2` 大约以 1 秒为间隔批量推送。`raw` 需要认证。          |
| `group`    | `none`、价格分组    | 默认:`none`。仅适用于分组的非原始订单簿频道。           |
| `depth`    | `1`、`10`、`20`        | 默认:`10`。分组订单簿频道每侧的价格档位数量。 |

数据客户端按以下方式选择订单簿间隔:

1. 提供 `params["interval"]` 时使用该值。
2. 当 WebSocket 连接已认证且未提供 interval 时,使用 `raw`。
3. 当连接未认证时,使用 Deribit 公开的 `100ms` 分组数据流。

```python
from nautilus_trader.model.identifiers import InstrumentId

instrument_id = InstrumentId.from_str("BTC-PERPETUAL.DERIBIT")

# 未配置 API 凭证时使用公开的 100ms 聚合数据流。
strategy.subscribe_order_book_deltas(instrument_id)

# 原始数据流。这也是未提供 interval 时已认证连接的默认值。
strategy.subscribe_order_book_deltas(
    instrument_id,
    params={"interval": "raw"},
)

# 在已认证连接上强制使用聚合数据流。
strategy.subscribe_order_book_deltas(
    instrument_id,
    params={"interval": "100ms", "depth": 10},
)
```

:::note
原始订单簿数据流需要已认证的 WebSocket 连接。订阅原始数据流之前,
请确保已配置 API 凭证。
:::

:::tip
对于大多数策略,100ms 聚合数据流以较低的消息开销提供了足够的粒度。
当你提供了凭证但不需要原始逐笔订单簿更新时,请设置
`params={"interval": "100ms"}`。
:::

### 序列号跳变恢复

适配器会跟踪每次订单簿更新上的 `change_id` / `prev_change_id` 序列号。
当检测到序列号跳变(消息丢失)时,适配器会自动:

1. 丢弃受影响标的所有传入的增量。
2. 取消订阅该订单簿频道。
3. 重新订阅以获取全新快照。
4. 快照到达后恢复正常处理。

在重新同步期间,策略不会收到陈旧或不完整的订单簿更新。

## 订单能力

以下是 Deribit 支持的订单类型、执行指令和有效期选项。

### 订单类型

| Nautilus 订单类型    | Deribit 订单类型 | 是否支持 | 备注                                   |
|------------------------|--------------------|-----------|-----------------------------------------|
| `MARKET`               | `market`           | ✓         | 以市场价格立即执行。    |
| `LIMIT`                | `limit`            | ✓         | 以指定价格或更优价格执行。 |
| `STOP_MARKET`          | `stop_market`      | ✓         | 触发时的条件市价单。    |
| `STOP_LIMIT`           | `stop_limit`       | ✓         | 触发时的条件限价单。     |
| `MARKET_IF_TOUCHED`    | `take_market`      | ✓         | 止盈式市价单。         |
| `LIMIT_IF_TOUCHED`     | `take_limit`       | ✓         | 止盈式限价单。          |
| `TRAILING_STOP_MARKET` | `trailing_stop`    | -         | *目前未实现*。            |
| `TRAILING_STOP_LIMIT`  | N/A                | -         | *Deribit 不支持*。             |
| `MARKET_TO_LIMIT`      | `market_limit`     | -         | *目前未实现*。            |

### 执行指令

| 指令   | 是否支持 | 备注                                                                            |
|---------------|-----------|----------------------------------------------------------------------------------|
| `post_only`   | ✓         | 如果会吃单,订单会被拒绝。使用 `reject_post_only=true`。 |
| `reduce_only` | ✓         | 订单只能用于减少现有仓位。                                      |

### 有效期

| 有效期 | 是否支持 | 备注                                                |
|---------------|-----------|------------------------------------------------------|
| `GTC`         | ✓         | 撤销前有效(`good_til_cancelled`)。           |
| `GTD`         | ✓         | 当日有效。于 UTC 8:00 到期(`good_til_day`)。 |
| `IOC`         | ✓         | 立即成交或取消(`immediate_or_cancel`)。         |
| `FOK`         | ✓         | 全部成交或取消(`fill_or_kill`)。                       |

Deribit 将有效期应用于限价类订单。对于 `MARKET`、`STOP_MARKET` 和
`MARKET_IF_TOUCHED` 订单,适配器会省略 `time_in_force`,因为 Deribit
会拒绝在市价类订单类型上传入该参数。

:::note
**Deribit 上的 GTD**:与其他允许任意到期时间的交易所不同,Deribit 的
`good_til_day` 总是于当天或次日的 UTC 8:00 到期。自定义到期时间会
被记录为警告,订单会使用交易所的固定到期行为。
:::

### 触发类型

条件单(止损单)支持不同的触发价格来源:

| 触发类型  | 是否支持 | 备注                                 |
|---------------|-----------|---------------------------------------|
| `last_price`  | ✓         | 使用最新成交价(默认)。 |
| `mark_price`  | ✓         | 使用标记价格。                  |
| `index_price` | ✓         | 使用标的指数价格。      |

```python
# 示例:使用标记价格触发的止损单
stop_order = order_factory.stop_market(
    instrument_id=instrument_id,
    order_side=OrderSide.SELL,
    quantity=Quantity.from_str("0.1"),
    trigger_price=Price.from_str("45000.0"),
    trigger_type=TriggerType.MARK_PRICE,  # 使用标记价格作为触发依据
)
strategy.submit_order(stop_order)
```

### 批量操作

| 操作                | 是否支持 | 备注                                                                    |
|--------------------------|-----------|--------------------------------------------------------------------------|
| 提交订单列表        | ✓         | 将每个订单作为独立的 Deribit 订单发送。没有原子性的场所批量操作。  |
| 按订单 ID 批量取消 | ✓         | 为每个场所订单 ID 发送独立的 `private/cancel` 请求。      |
| 按标的取消全部 | ✓         | 未提供方向过滤器时使用 `private/cancel_all_by_instrument`。 |
| 按方向过滤的取消全部 | ✓         | 在本地过滤缓存的未成交订单,然后取消每个匹配的订单。    |
| 批量修改             | -         | *目前未实现*:支持单笔订单修改。           |

### Post-only 行为

Deribit 提供两种 post-only 模式:

1. **价格调整(Deribit 默认)**:如果一个 post-only 订单会穿越价差并
   成交,Deribit 会自动将价格调整到价差内一个 tick 的位置。
2. **拒绝模式**:如果会穿越价差,订单会被立即拒绝。

Nautilus 适配器使用**拒绝模式**(`reject_post_only=true`)以获得
确定性行为。如果一个 post-only 订单会吃单,它会以错误码 `11054`
被拒绝,并发出一个 `due_post_only` 标志设为 `true` 的
`OrderRejected` 事件。

这使策略能够区分:

- 因违反 post-only 规则而被拒绝的订单(尝试吃单)。
- 因其他原因被拒绝的订单(保证金不足、价格无效等)。

### 订单修改

适配器使用 Deribit 原生的 `private/edit` 端点,而非取消再重下。这带来
若干优势:

| 优势                    | 说明                                                        |
|----------------------------|--------------------------------------------------------------------|
| 单次请求             | 比取消 + 新下单更快、延迟更低。           |
| 保留队列优先级 | 仅减少数量或保持相同价格时保留排队位置。 |
| 保留成交历史    | 部分成交仍关联到相同的订单 ID。                  |

**队列优先级规则:**

- **仅减少数量**:保留排队位置。
- **相同价格**:保留排队位置。
- **增加数量或更改价格**:失去排队位置(视为新订单)。

### 仓位管理

| 功能          | 是否支持 | 备注                                                             |
|------------------|-----------|-------------------------------------------------------------------|
| 查询仓位  | ✓         | 实时仓位更新。                                       |
| 仓位模式    | -         | *Deribit 不支持*:仅净额仓位模式。               |
| 杠杆控制 | -         | *Deribit 不支持*:没有直接的杠杆设置。           |
| 保证金模式      | -         | *目前未实现*:Deribit 暴露了账户保证金模式。 |

### 订单查询

| 功能              | 是否支持 | 备注                             |
|----------------------|-----------|-----------------------------------|
| 查询未成交订单    | ✓         | 列出所有活跃订单。           |
| 查询订单历史  | ✓         | 历史订单数据。            |
| 订单状态更新 | ✓         | 实时订单状态变化。    |
| 成交历史        | ✓         | 执行与成交报告。       |

### 条件单

| 功能                        | 是否支持 | 备注                                                              |
|--------------------------------|-----------|----------------------------------------------------------------------|
| 订单列表                    | ✓*        | 按顺序提交。适配器不提供原子性列表。 |
| 原生关联订单           | -         | *目前未实现*:Deribit 支持关联订单。       |
| OCO 订单                     | -         | *目前未实现*:Deribit 支持 OCO 关联。           |
| 括号单                 | -         | *目前未实现*:Deribit 支持 OTOCO 关联。         |
| 条件止损单        | ✓         | 止损市价单和止损限价单。                                 |
| 条件止盈单 | ✓         | 触及市价单和触及限价单。                     |

### 强平处理

Deribit 会为由强平触发的任何成交打标签。在 `user.trades` 数据流以及
`private/get_user_trades_*` 端点上,可选的 `liquidation` 字段指明
哪一方正在被强平:

| 取值  | 含义                       |
|--------|-------------------------------|
| `"M"`  | Maker 方被强平。    |
| `"T"`  | Taker 方被强平。    |
| `"MT"` | 双方都被强平。   |
| 缺失 | 正常的非强平成交。 |

适配器会为每一笔带强平标签的成交记录一条警告日志,包含标的、成交 ID、
订单 ID 和被强平的一方,然后通过正常管道发出 `FillReport`。Deribit
没有独立于强平 + 保险基金/组合保证金流程之外的 ADL 机制,因此没有
单独的 ADL 信号需要呈现。

上游参考资料:

- [`user.trades.{instrument_name}.{interval}` 频道](https://docs.deribit.com/#user-trades-instrument_name-interval)
- [强平文档](https://support.deribit.com/hc/en-us/articles/25944769313309-Liquidations)

## 资金费率

与大多数其他交易所按固定间隔结算不同,Deribit 是持续(每隔几秒)进行
资金结算的。对于 Deribit,`FundingRateUpdate` 上的 `interval` 字段
为 `None`,因为这种持续模型无法映射为一个离散周期。

## Deribit 特有数据

适配器从 Deribit 的 `deribit_volatility_index.{index_name}`
WebSocket 频道发出 `DeribitVolatilityIndex` 自定义数据。Deribit
提供诸如 `btc_usd` 和 `eth_usd` 之类的波动率指数数据流。

| 字段        | 类型    | 说明                                              |
|--------------|---------|----------------------------------------------------------|
| `index_name` | `str`   | Deribit 波动率指数名称,例如 `btc_usd`。    |
| `volatility` | `float` | 当前波动率指数值。                          |
| `ts_event`   | `int`   | 更新发生时的 UNIX 时间戳(纳秒)。  |
| `ts_init`    | `int`   | 该对象构建时的 UNIX 时间戳(纳秒)。 |

通过 `DataType(DeribitVolatilityIndex)` 从 actor 或策略中订阅。
必须提供 `index_name` 元数据键:

```python
from nautilus_trader.adapters.deribit import DeribitVolatilityIndex
from nautilus_trader.model import ClientId
from nautilus_trader.model.data import DataType

self.subscribe_data(
    data_type=DataType(DeribitVolatilityIndex, metadata={"index_name": "btc_usd"}),
    client_id=ClientId.from_str("DERIBIT"),
)
```

## 速率限制

Deribit 使用基于信用点(credit)和端点特定的速率限制。Deribit 官方
限制是权威依据,可能因端点、账户等级和当前场所政策而异。适配器增加了
本地令牌桶,以减少可避免的限流,但这不能替代 Deribit 自身的服务端
检查。

### HTTP 限制

| 桶 / 键       | 适配器桶          | 备注                                              |
|--------------------|--------------------------|----------------------------------------------------|
| `deribit:global`   | 20 请求/秒,突发 100   | 非撮合类 HTTP 请求的默认桶。     |
| `deribit:orders`   | 5 请求/秒,突发 20     | 面向底层客户端的撮合引擎 HTTP 桶。 |
| `deribit:account`  | 5 请求/秒,无突发     | 账户信息端点。                     |

### WebSocket 限制

| 操作             | 适配器桶        | 备注                                      |
|-----------------------|------------------------|--------------------------------------------|
| 订阅/取消订阅 | 3 请求/秒,突发 10   | 订阅操作。                   |
| 订单操作      | 5 请求/秒,突发 20   | 通过 WebSocket 的买、卖、编辑和取消。 |

:::note
Nautilus 适配器使用 WebSocket(而非 HTTP)提交订单以获得更低延迟。
订单操作受 `DERIBIT_WS_ORDER_QUOTA`(5 请求/秒,突发 20)速率限制。
:::

### 基于信用点系统的细节

Deribit 会持续补充非撮合引擎信用点。当前的公开文档列出的默认非撮合
引擎池如下:

**非撮合引擎请求:**

| 参数        | 数值              | 备注                           |
|------------------|--------------------|---------------------------------|
| 每次请求成本 | 500 信用点        | 每次 API 调用都会消耗信用点。 |
| 最大池     | 50,000 信用点     | 允许 100 次突发请求。       |
| 补充速率      | 10,000 信用点/秒 | 约每秒可持续 20 次请求。  |

**撮合引擎请求(默认等级):**

| 参数      | 数值          | 备注                            |
|----------------|----------------|-----------------------------------|
| 持续速率 | 5 请求/秒 | 持续速率限制。           |
| 突发容量 | 20 请求    | 触发限流前的最大突发量。 |

对于做市商和高交易量的交易者,可以根据 7 天交易量分档获得更高的撮合
引擎限额。

某些 Deribit 端点有更严格的按方法限制。例如,当前场所文档列出
`public/get_instruments` 为每秒 1 次请求,突发 50 次;订阅方法约为
每秒 3.3 次请求,突发 10 次。请将 `product_types` 限定在你所需的品类
范围内,并在实盘系统中避免反复进行完整的标的重新加载。

Nautilus 适配器实现了广泛的令牌桶速率限制器,配置如下:

- `DERIBIT_HTTP_REST_QUOTA`:20 请求/秒,突发 100(非撮合 HTTP)
- `DERIBIT_HTTP_ORDER_QUOTA`:5 请求/秒,突发 20(撮合引擎 HTTP)
- `DERIBIT_HTTP_ACCOUNT_QUOTA`:5 请求/秒,无突发(账户 HTTP)
- `DERIBIT_WS_ORDER_QUOTA`:5 请求/秒,突发 20(撮合引擎 WebSocket)
- `DERIBIT_WS_SUBSCRIPTION_QUOTA`:3 请求/秒,突发 10(订阅和取消订阅)

更多详情,参见
[速率限制文章](https://support.deribit.com/hc/en-us/articles/25944617523357-Rate-Limits)。

:::warning
当超出允许的配额时,Deribit 会返回错误码 `10028`
(too_many_requests)。反复违反可能导致临时限流。
:::

## 连接管理

### 平台限制

| 限制                                   | 当前 Deribit 指导值 |
|-----------------------------------------|--------------------------|
| 每个 API key 或登录的活跃会话数    | 16                       |
| 每个浏览器会话的 Web 应用连接数 | 2                         |

### 基于会话的身份验证

适配器为数据客户端和执行客户端使用**独立的 WebSocket 会话**,各自拥有
自己的认证范围:

| 客户端           | 会话名称         | 用途                                             |
|------------------|----------------------|-----------------------------------------------------|
| 数据客户端      | `nautilus-data`      | 市场数据订阅(原始数据流需要认证)。 |
| 执行客户端 | `nautilus-execution` | 订单操作(买入、卖出、编辑、取消)。         |

**身份验证流程:**

1. WebSocket 连接到 Deribit。
2. 客户端使用带会话范围的 `client_signature` grant 类型进行认证。
3. 令牌会在过期前刷新。
4. 重连时,重新认证会以指数退避方式重试(最多 3 次)。若全部尝试
   失败,只会恢复公开频道的订阅。

这种基于会话的方式带来以下优势:

- 每种客户端类型的令牌管理相互独立。
- 隔离的故障域(数据认证失败不会影响执行)。
- 在 Deribit 的会话日志中留下清晰的审计轨迹。

### 最佳实践

适配器遵循 Deribit 的
[推荐连接实践](https://support.deribit.com/hc/en-us/articles/25944603459613):

1. 使用 **WebSocket 订阅**获取实时数据,而非 REST 轮询,从而减少请求
   数量、降低延迟、减少速率限制消耗。
2. 在提供凭证时**对所有连接进行身份验证**。已认证用户能享受更高的
   速率限制,且不太容易被按 IP 限流。
3. **实现心跳**(默认 30 秒间隔),以维持连接健康并及早发现断连。
4. **自动处理重连**,包括重新认证和订阅恢复。

:::tip
即使只需要公开数据访问,也始终提供 API 凭证。已认证连接拥有更高的
速率限制,并且在高负载期间,Deribit 会在实施限制前联系已认证客户端。
:::

:::note
适配器默认使用 30 秒的心跳间隔。Deribit 要求 WebSocket 心跳间隔至少
为 10 秒。
:::

## 身份验证

Deribit 对私有端点使用带 HMAC-SHA256 签名的 API key 身份验证。

创建 API 凭证:

1. 登录你的 Deribit 账户 [deribit.com](https://www.deribit.com)
   (测试网请使用 [test.deribit.com](https://test.deribit.com))。
2. 导航到 **Account** -> **API**。
3. 点击 **Add new key** 并配置权限:
   - 启用 **read** 以访问市场数据
   - 启用 **trade** 以执行订单
   - 如需访问账户余额,启用 **wallet**
4. 记下你的 **Client ID**(API key)和 **Client Secret**(API secret)。

:::warning
妥善保管你的 API secret。切勿分享或将其提交到版本控制中。
:::

### API key 权限范围

Deribit 上的每个 API key 都分配有默认的访问范围,决定了最大权限。
[创建 API key](https://support.deribit.com/hc/en-us/articles/26268257333661)
时请配置适当的权限:

| 范围              | 所需场景                           |
|--------------------|----------------------------------------|
| `account:read`     | 账户信息、组合数据。   |
| `trade:read`       | 查看订单和仓位。             |
| `trade:read_write` | 下单、修改和取消订单。      |
| `wallet:read`      | 查看余额和交易历史。 |

**推荐的交易最低配置:** `account:read`、`trade:read_write`、`wallet:read`

:::tip
请遵循最小权限原则。对于仅需数据访问(市场数据,不交易)的场景,
创建一个不含 `trade:read_write` 的只读密钥。
:::

## 测试网

Deribit 提供了一个测试网环境,可以在不使用真实资金的情况下测试策略。
要使用测试网,请在客户端配置中设置
`environment=DeribitEnvironment.TESTNET`:

```python
from nautilus_trader.adapters.deribit import DeribitDataClientConfig
from nautilus_trader.adapters.deribit import DeribitEnvironment
from nautilus_trader.adapters.deribit import DeribitExecClientConfig
from nautilus_trader.adapters.deribit import DeribitProductType
from nautilus_trader.model import AccountId
from nautilus_trader.model import TraderId

product_types = [DeribitProductType.FUTURE]
trader_id = TraderId.from_str("TRADER-001")
account_id = AccountId.from_str("DERIBIT-001")

data_config = DeribitDataClientConfig(
    product_types=product_types,
    environment=DeribitEnvironment.TESTNET,
)

exec_config = DeribitExecClientConfig(
    trader_id=trader_id,
    account_id=account_id,
    product_types=product_types,
    environment=DeribitEnvironment.TESTNET,
)
```

启用测试网模式时:

- HTTP 请求使用 `https://test.deribit.com`。
- WebSocket 连接使用 `wss://test.deribit.com/ws/api/v2`。
- 从 `DERIBIT_TESTNET_API_KEY` 和 `DERIBIT_TESTNET_API_SECRET`
  环境变量加载凭证。

:::note
测试网 API key 与生产环境密钥是分开的。请通过测试网界面
[test.deribit.com](https://test.deribit.com) 专门为测试网创建
API key。
:::

## 配置

### 数据客户端配置选项

| 选项                             | 默认值    | 说明                                                        |
|------------------------------------|------------|--------------------------------------------------------------------|
| `api_key`                          | `None`     | Deribit API key。省略时从环境变量加载。    |
| `api_secret`                       | `None`     | Deribit API secret。省略时从环境变量加载。 |
| `product_types`                    | `[FUTURE]` | 要加载的产品类型。                                             |
| `environment`                      | `MAINNET`  | 环境枚举(`MAINNET` 或 `TESTNET`)。                         |
| `base_url_http`                    | `None`     | HTTP JSON-RPC 基础 URL 的覆盖值。                           |
| `base_url_ws`                      | `None`     | WebSocket 基础 URL 的覆盖值。                               |
| `proxy_url`                        | `None`     | HTTP 与 WebSocket 传输的可选代理 URL。              |
| `http_timeout_secs`                | `60`       | HTTP 调用的请求超时时间(秒)。                         |
| `max_retries`                      | `3`        | 可恢复错误的最大重试次数。                     |
| `retry_delay_initial_ms`           | `1,000`    | 重试前的初始延迟(毫秒)。                     |
| `retry_delay_max_ms`               | `10,000`   | 重试之间的最大延迟(毫秒)。                     |
| `heartbeat_interval_secs`          | `30`       | WebSocket 心跳间隔。                                      |
| `update_instruments_interval_mins` | `60`       | 标的刷新间隔(分钟)。                  |
| `auto_load_missing_instruments`    | `False`    | 在订阅时惰性加载未缓存的标的。                       |

#### 订阅时惰性加载

`subscribe_*` 命令在发送 WebSocket 订阅之前会先在本地缓存中查找该
标的,以便处理器能够解析传入的帧。使用默认设置
`auto_load_missing_instruments = False` 时,对未预加载(因为配置的
`product_types` 未包含它)的标的进行订阅,会预先返回一个错误,而不是
静默成功并在处理器中丢弃后续的帧。

将 `auto_load_missing_instruments` 设置为 `True`,可以在首次订阅时
改为通过 HTTP 获取该标的,填充 WebSocket 处理器缓存,然后转发该订阅。
HTTP 失败会被记录,并跳过该 WebSocket 订阅。

### 执行客户端配置选项

| 选项                   | 默认值    | 说明                                                        |
|--------------------------|------------|--------------------------------------------------------------------|
| `trader_id`              | 必填   | 用于生成报告和事件的 Nautilus trader ID。               |
| `account_id`             | 必填   | 用于生成报告和事件的 Nautilus 账户 ID。              |
| `api_key`                | `None`     | Deribit API key。省略时从环境变量加载。    |
| `api_secret`             | `None`     | Deribit API secret。省略时从环境变量加载。 |
| `product_types`          | `[FUTURE]` | 要加载的产品类型。                                             |
| `environment`            | `MAINNET`  | 环境枚举(`MAINNET` 或 `TESTNET`)。                         |
| `base_url_http`          | `None`     | HTTP JSON-RPC 基础 URL 的覆盖值。                           |
| `base_url_ws`            | `None`     | WebSocket 基础 URL 的覆盖值。                               |
| `proxy_url`              | `None`     | HTTP 与 WebSocket 传输的可选代理 URL。              |
| `http_timeout_secs`      | `60`       | HTTP 调用的请求超时时间(秒)。                         |
| `max_retries`            | `3`        | 可恢复错误的最大重试次数。                     |
| `retry_delay_initial_ms` | `1,000`    | 重试前的初始延迟(毫秒)。                     |
| `retry_delay_max_ms`     | `10,000`   | 重试之间的最大延迟(毫秒)。                     |

Rust 配置还暴露了 `transport_backend`。当启用 `transport-sockudo`
Cargo 特性时,默认值为 `Sockudo`,否则为 `Tungstenite`。Python 绑定
使用编译时的默认值。

### 生产环境配置

以下是一个使用 Deribit 数据和执行客户端的实时节点示例:

```python
from nautilus_trader.adapters.deribit import DeribitDataClientConfig
from nautilus_trader.adapters.deribit import DeribitDataClientFactory
from nautilus_trader.adapters.deribit import DeribitEnvironment
from nautilus_trader.adapters.deribit import DeribitExecClientConfig
from nautilus_trader.adapters.deribit import DeribitExecutionClientFactory
from nautilus_trader.adapters.deribit import DeribitProductType
from nautilus_trader.common import Environment
from nautilus_trader.live import LiveNode
from nautilus_trader.model import AccountId
from nautilus_trader.model import TraderId

product_types = [DeribitProductType.FUTURE]
trader_id = TraderId.from_str("TRADER-001")
account_id = AccountId.from_str("DERIBIT-001")

node = (
    LiveNode.builder("DERIBIT-001", trader_id, Environment.LIVE)
    .add_data_client(
        None,
        DeribitDataClientFactory(),
        DeribitDataClientConfig(
            product_types=product_types,
            environment=DeribitEnvironment.MAINNET,
            api_key=None,
            api_secret=None,
        ),
    )
    .add_exec_client(
        None,
        DeribitExecutionClientFactory(),
        DeribitExecClientConfig(
            trader_id=trader_id,
            account_id=account_id,
            product_types=product_types,
            environment=DeribitEnvironment.MAINNET,
            api_key=None,
            api_secret=None,
        ),
    )
    .build()
)
```

### API 凭证

向 Deribit 客户端提供凭证有多种方式。可以将对应的值传给配置对象,或
设置以下环境变量:

对于 Deribit 实盘(生产)客户端:

- `DERIBIT_API_KEY`
- `DERIBIT_API_SECRET`

对于 Deribit 测试网客户端:

- `DERIBIT_TESTNET_API_KEY`
- `DERIBIT_TESTNET_API_SECRET`

:::tip
建议使用环境变量来管理凭证。
:::

### 产品类型

`product_types` 配置项控制加载哪些 Deribit 产品系列。通过
`DeribitProductType` 枚举可用的选项:

- `DeribitProductType.FUTURE` - 永续和到期期货。
- `DeribitProductType.OPTION` - 看涨和看跌期权。
- `DeribitProductType.SPOT` - 现货交易对。
- `DeribitProductType.FUTURE_COMBO` - 期货价差标的。
- `DeribitProductType.OPTION_COMBO` - 期权价差标的。

加载多种产品类型的示例:

```python
from nautilus_trader.adapters.deribit import DeribitDataClientConfig
from nautilus_trader.adapters.deribit import DeribitProductType

config = DeribitDataClientConfig(
    product_types=[
        DeribitProductType.FUTURE,
        DeribitProductType.OPTION,
    ],
    # ... 其他配置
)
```

### 基础 URL 覆盖

可以为 HTTP 和 WebSocket API 覆盖默认的基础 URL:

| 环境 | HTTP URL                   | WebSocket URL                      |
|-------------|----------------------------|------------------------------------|
| 生产环境  | `https://www.deribit.com`  | `wss://www.deribit.com/ws/api/v2`  |
| 测试网     | `https://test.deribit.com` | `wss://test.deribit.com/ws/api/v2` |

## 服务器基础设施

Deribit 的撮合引擎位于 **Equinix LD4,英国斯劳(Slough)**。对于
延迟敏感型策略,可以考虑在伦敦附近托管。Deribit 直接为机构客户提供
托管和交叉连接选项。

对于大多数通过互联网连接的用户,适配器内置的重试逻辑、心跳监控和
自动重连处理能够提供可靠的连接性。

更多详情,参见
[服务器基础设施文章](https://support.deribit.com/hc/en-us/articles/25944617582877)。

## 贡献

:::info
如需了解更多功能或为 Deribit 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
