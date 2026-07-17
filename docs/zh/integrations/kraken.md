# Kraken

Kraken 提供跨广泛数字资产品类的现货和衍生品交易。该集成连接到 Kraken Pro,
支持 Kraken Spot 和 Kraken Derivatives(Futures)的实时市场数据接入与
订单执行。

## 概览

该适配器以 Rust 实现,并带有 Python 绑定,便于在基于 Python 的工作流中
使用。它不需要外部 Kraken 客户端库;核心组件被编译为静态库,并在构建
期间自动链接。

本指南假设交易者需要同时配置实时市场数据源和交易执行。Kraken 适配器
包含多个组件,可以单独使用,也可以组合使用,视具体使用场景而定。

- `KrakenSpotRawHttpClient` 和 `KrakenFuturesRawHttpClient`:底层 HTTP
  API 连接。
- `KrakenSpotHttpClient` 和 `KrakenFuturesHttpClient`:带标的缓存和对账
  支持的更高层 HTTP 客户端。
- `KrakenInstrumentProvider`:标的解析与加载功能。
- `KrakenDataClient`:市场数据流管理器。
- `KrakenExecutionClient`:账户管理与交易执行网关。
- `KrakenDataClientFactory`:Kraken 数据客户端工厂(供交易节点构建器
  使用)。
- `KrakenExecutionClientFactory`:Kraken 执行客户端工厂(供交易节点
  构建器使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),不必直接与这些底层
组件打交道。
:::

## 示例

可以在 [examples/live/kraken] 目录中找到实时示例脚本。

[examples/live/kraken]: https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/kraken/

## Kraken 文档

Kraken 为用户提供了详细文档:

- [Kraken API 文档](https://docs.kraken.com/api/)
- [Kraken Spot REST API](https://docs.kraken.com/api/docs/guides/spot-rest-intro)
- [Kraken Futures REST API](https://docs.kraken.com/api/docs/futures-api)

请将本 NautilusTrader 集成指南与 Kraken 官方文档结合参考。

## 产品

Kraken 支持两大主要产品类别:

| 产品类型             | 是否支持 | 备注                                                     |
|--------------------------|-----------|-------------------------------------------------------------|
| 现货                     | ✓         | 支持保证金的标准加密货币交易对。        |
| 期货(永续)      | ✓         | 反向(`PI_`)和以 USD 计保证金(`PF_`)的永续互换。 |
| 期货(到期/灵活)     | ✓         | 固定到期(`FI_`)和灵活(`FF_`)合约。        |

:::note
**每个客户端仅支持一种产品类型**:每个 Kraken 数据或执行客户端都配置为
单一的 `product_type`(`SPOT` 或 `FUTURES`);单个客户端不能横跨这两个
市场。
:::

## K 线流

### 支持的周期

Kraken 适配器支持通过 WebSocket 为现货市场提供实时 K 线(OHLC)数据流。
支持以下周期:

| 周期   | BarType 规格 |
|------------|-----------------------|
| 1 分钟   | `1-MINUTE-LAST`       |
| 5 分钟  | `5-MINUTE-LAST`       |
| 15 分钟 | `15-MINUTE-LAST`      |
| 30 分钟 | `30-MINUTE-LAST`      |
| 1 小时     | `1-HOUR-LAST`         |
| 4 小时    | `4-HOUR-LAST`         |
| 1 天      | `1-DAY-LAST`          |
| 1 周     | `1-WEEK-LAST`         |
| 15 天    | `15-DAY-LAST`         |

:::note
**期货限制**:Kraken Futures 不支持通过 WebSocket 进行 K 线数据流传输。
请改用 `request_bars()` 获取历史 K 线数据。
:::

### K 线发出延迟

Kraken 的 WebSocket OHLC 频道会在每笔成交时推送*当前*(未完成)K 线的
更新。与某些交易所(例如 Binance)不同,Kraken 没有提供 “is_closed”
指标来表明某根 K 线是否已完成。

为避免发出部分/未完成的 K 线,适配器会缓冲当前 K 线,只在下一根 K 线
周期开始时(即收到带有新 `interval_begin` 时间戳的消息时)才发出它。
这意味着:

- K 线的发出会有最多一个 K 线周期的延迟。
- 对于 1 分钟 K 线,最大延迟约为 1 分钟。
- 发出的 K 线数据是完整且最终确定的。

我们选择这种方式而非基于计时器的发出方式,原因是:

- 基于计时器的发出可能会错过 K 线收盘前的最后一次更新。
- Kraken 的更新并不保证准确地在周期边界到达。
- 缓冲以延迟为代价,保证了数据的完整性。

:::warning
如果 K 线延迟对你的策略有影响,可以考虑使用成交 tick 数据,并使用
`BarAggregator` 在本地聚合 K 线。
:::

:::tip
对于大多数使用场景,我们建议使用 `INTERNAL` K 线聚合方式(订阅成交并
在本地聚合 K 线),而非 `EXTERNAL` 交易所提供的 K 线:

- K 线在完成时立即发出,没有缓冲延迟。
- 各交易所的行为一致,简化了多场所策略的开发。

:::

## 符号规则

### 比特币符号格式(BTC 与 XBT)

Kraken 在其不同的 API 中使用不同的比特币符号约定:

| 市场  | 符号格式 | 示例            | 备注                                       |
|---------|---------------|--------------------|---------------------------------------------|
| 现货    | `BTC`         | `BTC/USD.KRAKEN`   | 适配器在加载时将 XBT 规范化为 BTC。 |
| 期货 | `XBT`         | `PI_XBTUSD.KRAKEN` | 使用 Kraken 原生的 XBT 格式。            |

:::note
Kraken 的 REST API 对比特币返回 `XBT`(遵循 ISO 4217 中对超国家货币的
惯例),但其 WebSocket v2 API 要求使用 `BTC` 格式。适配器在加载标的时
会自动将现货符号规范化为 `BTC`,无论 XBT 出现在基础货币(例如
`XBT/USD` 变为 `BTC/USD`)还是计价货币(例如 `ETH/XBT` 变为
`ETH/BTC`)中。期货仍保留 Kraken 原生的 `XBT` 格式。
:::

### 现货市场

NautilusTrader 对 Kraken 现货标的符号使用 ISO 4217-A3 格式,从而在
不同交易所之间提供标准化的表示。适配器在内部处理向 Kraken 原生格式的
转换。

**标的 ID 格式:**

```python
InstrumentId.from_str("BTC/USD.KRAKEN")   # 现货 BTC/USD
InstrumentId.from_str("ETH/USD.KRAKEN")   # 现货 ETH/USD
InstrumentId.from_str("SOL/USD.KRAKEN")   # 现货 SOL/USD
InstrumentId.from_str("BTC/USDT.KRAKEN")  # 现货 BTC/USDT
InstrumentId.from_str("ETH/BTC.KRAKEN")   # 现货 ETH/BTC(从 ETH/XBT 规范化而来)
```

### 期货市场

Kraken 期货标的使用带前缀的特定命名约定:

- `PI_` - 永续反向合约(例如 `PI_XBTUSD`)
- `PF_` - 永续固定保证金合约(例如 `PF_XBTUSD`)
- `FI_` - 固定到期反向合约(例如 `FI_XBTUSD_230929`)
- `FF_` - 灵活期货合约

**标的 ID 格式:**

```python
InstrumentId.from_str("PI_XBTUSD.KRAKEN")  # 永续反向 BTC
InstrumentId.from_str("PI_ETHUSD.KRAKEN")  # 永续反向 ETH
InstrumentId.from_str("PF_XBTUSD.KRAKEN")  # 永续固定保证金 BTC
```

## 数据能力

### 订阅(实时)

| 数据类型              | 现货 | 期货 | 备注                                  |
|------------------------|------|---------|----------------------------------------|
| `QuoteTick`            | ✓    | ✓       | 派生自 ticker 频道。           |
| `TradeTick`            | ✓    | ✓       |                                        |
| `OrderBookDeltas`      | ✓    | ✓       | 现货 L2/L3 和期货 L2 更新。     |
| `OrderBookDepth10`     | -    | -       | 使用深度为 `10` 的 `OrderBookDeltas`。 |
| `Bar`                  | ✓    | -       | 现货 WS OHLC 频道。参见 K 线章节。 |
| `MarkPriceUpdate`      | -    | ✓       | 来自期货 ticker 数据流。              |
| `IndexPriceUpdate`     | -    | ✓       | 来自期货 ticker 数据流。              |
| `FundingRateUpdate`    | -    | ✓       | 仅限永续合约。                       |
| `InstrumentStatus`     | ✓    | ✓       | Python 适配器轮询标的刷新。 |

### 请求(历史)

| 数据类型              | 现货 | 期货 | 备注                                  |
|------------------------|------|---------|----------------------------------------|
| `TradeTick`            | ✓    | ✓       |                                        |
| `Bar`                  | ✓    | ✓       |                                        |
| `OrderBook`(快照) | ✓    | ✓       | 通过 HTTP 深度端点。               |
| `FundingRateUpdate`    | -    | ✓       | 客户端按开始/结束/limit 过滤。 |

## L3 订单簿(按单/市场逐单)

Kraken 通过 `wss://ws-l3.kraken.com/v2` 上的 WebSocket v2 `level3`
频道暴露现货的逐单订单簿数据。这提供了场所订单 ID、按订单的数量,以及
真正的增量事件(`add`、`modify`、`delete`)。适配器将每个场所订单 ID
哈希为 NautilusTrader 使用的 `u64` 类型 `BookOrder.order_id` 字段。

### 前置条件

L3 订阅需要现货 API 凭证,因为 Kraken 的 `level3` 频道是需要认证的。
在 `KrakenDataClientConfig` 中设置,或通过 `KRAKEN_SPOT_API_KEY` 和
`KRAKEN_SPOT_API_SECRET` 设置:

```python
from nautilus_trader.adapters.kraken.config import KrakenDataClientConfig

config = KrakenDataClientConfig(
    api_key="YOUR_KEY",
    api_secret="YOUR_SECRET",
)
```

然后使用 `book_type=BookType.L3_MBO` 订阅:

```python
from nautilus_trader.model.enums import BookType

await client.subscribe_book_deltas(
    instrument_id=instrument_id,
    book_type=BookType.L3_MBO,
    depth=1000,  # 有效值:10、100、1000
)
```

有效的深度值为 `10`、`100` 和 `1000`。`depth` 为 `0` 时会使用 `1000`。

### CRC32 校验和验证

默认情况下,适配器会在 Kraken 提供 CRC32 校验和时,对每个 L3 快照和
更新进行校验。若不匹配,它会发出一个 `Clear` 增量,清除本地 L3 状态,
刷新认证令牌,并重新订阅,以便 Kraken 发送一个全新的快照。要在基准
测试中禁用校验:

```python
config = KrakenDataClientConfig(
    api_key="...",
    api_secret="...",
    validate_l3_checksum=False,
)
```

### 存储建议

`OrderBookDelta` 的 Arrow schema 中已经携带了 `order_id: u64`,因此
L3 数据在 `ParquetDataCatalog` 中的存储方式与 L2 相同。L3 每个标的
产生的事件数量远多于 L2。推荐设置:

- 较小的分块大小(例如 `chunk_size=50_000`),以加快并行读取速度。
- 在 catalog 配置中启用 `zstd` 压缩。
- 使用按标的的路径分区(默认已启用)。

## 订单能力

### 订单类型

| 订单类型             | 现货 | 期货 | 备注                                         |
|------------------------|------|---------|-----------------------------------------------|
| `MARKET`               | ✓    | ✓       | 以市场价立即执行。          |
| `LIMIT`                | ✓    | ✓       | 以指定价格或更优价格执行。       |
| `STOP_MARKET`          | ✓    | ✓       | 条件市价单(止损)。         |
| `MARKET_IF_TOUCHED`    | ✓    | ✓       | 条件市价单(止盈)。       |
| `STOP_LIMIT`           | ✓    | ✓       | 条件限价单(止损限价)。    |
| `LIMIT_IF_TOUCHED`     | ✓    | ✓       | 映射为带 `limit_price` 的 `take_profit`。     |
| `TRAILING_STOP_MARKET` | ✓    | -       | 带 `trailing_offset` 的跟踪止损。         |
| `TRAILING_STOP_LIMIT`  | ✓    | -       | 带 `limit_offset` 的跟踪止损限价单。      |

### 有效期

| 有效期 | 现货 | 期货 | 备注                                               |
|---------------|------|---------|-----------------------------------------------------|
| `GTC`         | ✓    | ✓       | 撤销前有效。                                 |
| `GTD`         | ✓    | -       | 指定日期前有效(仅现货,需要 `expire_time`)。 |
| `IOC`         | ✓    | ✓       | 立即成交或取消。                                |
| `FOK`         | ✓    | -       | 仅限现货限价单。                             |

:::note
**市价单** 本身就是立即执行的,不支持有效期设置。`IOC` 仅适用于
限价类订单。
:::

### 执行指令

| 指令      | 现货 | 期货 | 备注                                                                |
|------------------|------|---------|------------------------------------------------------------------------|
| `post_only`      | ✓    | ✓       | 适用于限价单。                                          |
| `reduce_only`    | ✓    | ✓       | 现货要求 `spot_account_type=Margin`(仅限保证金订单)。       |
| `quote_quantity` | ✓    | -       | 仅限现货。以计价货币计量的数量(`viqc`)。                        |
| `display_qty`    | ✓    | -       | 仅限现货。冰山订单(`displayvol`)。                            |

### 触发类型

条件单(止损、止盈、跟踪止损)在现货上支持触发价格参考:

| 触发类型  | 现货 | 期货 | 备注                                      |
|---------------|------|---------|--------------------------------------------|
| `LAST_PRICE`  | ✓    | ✓       | 默认。最新成交价。                |
| `INDEX_PRICE` | ✓    | ✓       | 更广泛的市场指数价格。                |
| `MARK_PRICE`  | -    | ✓       | 仅限期货。                              |

:::note
适配器会在提交时拒绝不受支持的触发类型(例如 `BID_ASK`),而不是
静默地进行强制转换。
:::

### 批量操作

| 操作    | 现货 | 期货 | 备注                                                   |
|--------------|------|---------|-----------------------------------------------------------|
| 批量提交 | ✓    | ✓       | 现货以 15 笔为一批。期货以 10 笔为一批。         |
| 批量修改 | -    | ✓       | 仅限期货 HTTP 辅助方法。执行时发送单条命令。  |
| 批量取消 | ✓    | ✓       | 自动以 50 笔为一批分块处理。                         |

:::note
**取消所有订单**:

- 不支持按订单方向过滤;无论方向如何,所有订单都会被取消。
- 现货:取消所有交易对上的所有未成交订单。
- 期货:需要提供 `instrument_id`;只取消该符号的订单。

:::

### 仓位管理

| 功能          | 现货 | 期货 | 备注                                                   |
|------------------|------|---------|-----------------------------------------------------------|
| 查询仓位  | ✓    | ✓       | 现货保证金通过 `OpenPositions`;现货现金模式为选择性启用。      |
| 仓位模式    | -    | -       | 每个标的一个仓位。                         |
| 杠杆控制 | ✓    | ✓       | 现货分档;按订单的 `params={"leverage": N}`。         |
| 保证金模式      | ✓    | ✓       | 现货/期货全仓保证金;不支持现货逐仓保证金。     |

### 订单查询

| 功能              | 现货 | 期货 | 备注                                        |
|----------------------|------|---------|-----------------------------------------------|
| 查询未成交订单    | ✓    | ✓       | 列出所有活跃订单。                      |
| 查询订单历史  | ✓    | ✓       | 带分页的历史订单数据。       |
| 订单状态更新 | ✓    | ✓       | 通过 WebSocket 实时更新订单状态变化。 |
| 成交历史        | ✓    | ✓       | 执行与成交报告。                  |

### 条件单

| 功能             | 现货 | 期货 | 备注                                    |
|---------------------|------|---------|------------------------------------------|
| 订单列表         | -    | -       | *不支持*。                         |
| OCO 订单          | -    | -       | *不支持*。                         |
| 括号单      | -    | -       | *不支持*。                         |
| 条件单  | ✓    | ✓       | 止损和止盈订单。             |

## 订单路由(现货)

现货执行客户端默认通过 Kraken 已认证的 WebSocket v2 交易频道路由
`submit_order`、`modify_order`、`cancel_order` 和
`submit_order_list`,当 WebSocket 处于非活跃状态时回退到 REST。在
`KrakenExecClientConfig` 上设置 `use_ws_trade=False`,可将所有订单
操作都路由至 REST。

### 通过 REST 路由的订单形态

某些现货订单形态始终通过 REST 路由。它们分为两类:Kraken WS v2 API
完全不支持的形态,以及 WS API 支持但该适配器尚未编码实现的形态。

**Kraken WS v2 的限制:**

| 形态                     | 原因                                                       |
|---------------------------|--------------------------------------------------------------|
| 不受支持的触发类型 | `triggers.reference` 只接受 `last` 和 `index`。        |
| 混合符号订单列表  | `batch_add` 要求所有订单使用同一个符号。                 |

**该适配器尚未编码实现的形态(后续工作,目前走 REST):**

| 形态                       | 备注                                                                                |
|-----------------------------|--------------------------------------------------------------------------------------|
| `FOK` 有效期         | 可编码为 `FOK` 有效期,但构建器路由到 REST。                   |
| 跟踪止损 / 跟踪止损限价单  | 可通过 `triggers.price` + `triggers.price_type` 编码,但构建器路由到 REST。 |
| 冰山单(`display_qty`)     | 可编码为 `order_type: "iceberg"` + `display_qty`,但构建器路由到 REST。   |
| 计价货币数量订单       | 以计价货币数量买入市价单映射为 `cash_order_qty`;目前路由到 REST。                    |

按调用传入的 `params={"use_ws_trade": False}` 覆盖会强制单条命令走
REST,无论配置的默认值如何。可以在 `SubmitOrder`、`ModifyOrder`、
`CancelOrder` 或 `SubmitOrderList` 上设置该参数。

### WebSocket 请求超时

当一次 WebSocket 往返超过 `ws_request_timeout_secs`(默认 `5`)时,
分发器会将该命令的结果视为未知,并使订单保持在当前的飞行中状态:

- 提交 / batch_add:分发器可能通过同一个 WebSocket 发送一个尽力而为的
  补偿性 `cancel_order`,以避免延迟的场所接受结果变成一个孤儿订单。
- 修改:订单保持在 `PENDING_UPDATE`。
- 取消:订单保持在 `PENDING_CANCEL`。

该超时本身不会发出 `OrderRejected`、`OrderModifyRejected` 或
`OrderCancelRejected`。如果场所实际接受了该命令,恢复路径是 WebSocket
订单更新或实时执行对账引擎(`open_check_interval_secs`)。

:::tip
将 `ws_request_timeout_secs` 设置得明显高于你观察到的往返延迟(默认值
`5` 大约是典型延迟的 25 倍),以便该超时只在真正的网络故障下触发。
:::

### WebSocket 订单路由选项

`KrakenExecClientConfig` 暴露:

| 选项                    | 默认值 | 说明                                                   |
|---------------------------|---------|-----------------------------------------------------------|
| `use_ws_trade`            | `True`  | 当交易频道处于活跃状态时,通过 WS 路由订单。         |
| `ws_request_timeout_secs` | `5`     | 在将命令结果标记为未知之前的 WS 往返超时时间。 |

## 对账

Kraken 适配器为现货和期货市场都提供了对账能力,允许交易者在启动时或
运行期间将本地状态与交易所状态同步。

### 现货对账

**订单状态报告:**

- 未成交订单:获取所有当前活跃的订单。
- 已关闭订单:通过分页支持获取历史订单。
- 时间范围查询:支持按开始/结束时间戳过滤。

**成交报告:**

- 成交历史:通过分页获取执行历史。
- 时间范围查询:支持按开始/结束时间戳过滤。
- 所有成交类型:市价、限价和条件单成交。

**保证金仓位报告**(当 `spot_account_type=Margin` 时):

- 未平仓仓位:从 `POST /0/private/OpenPositions` 获取,并按
  (交易对、方向)聚合为 `PositionStatusReport` 条目。
- 合成 FLAT 清理:如果本地缓存中存在一个不再出现在场所上的未平仓
  现货保证金仓位(Kraken 会在 `OpenPositions` 中省略已平仓的仓位),
  适配器会在下一次仓位检查节拍时发出一份合成的 FLAT 报告,使引擎
  对账为已平仓。
- 保证金余额:`POST /0/private/TradeBalance` 会与账户状态刷新一并
  调用;已用保证金填充 `MarginBalance.initial`,其余指标流入
  `AccountState.info`(参见现货保证金交易)。

### 期货对账

**订单状态报告:**

- 未成交订单:获取所有当前活跃的期货订单。
- 历史订单:当 `open_only=False` 时,获取已关闭和已成交的订单。
- 订单事件:通过 `/api/history/v2/orders` 端点获取完整的订单生命周期
  历史。

**成交报告:**

- 成交历史:获取所有执行报告。
- 时间过滤:客户端按开始/结束时间戳过滤(解析 RFC3339 时间戳)。
- 所有成交类型:带手续费信息的 maker 和 taker 成交。

**仓位状态报告:**

- 未平仓仓位:获取所有活跃的期货仓位。
- 实时数据:包含未实现资金费用、均价和仓位规模。

:::note
**期货时间过滤**:Kraken Futures 的成交端点不支持服务端的时间范围
过滤。适配器通过解析 `fillTime` 字段并与请求的开始/结束时间戳比较,
实现客户端的过滤。
:::

### 现货仓位报告(现金模式)

在现金模式下,Kraken 适配器可以选择性地将现货标的的钱包余额报告为
仓位状态报告。此功能默认禁用,必须通过配置显式启用。保证金模式的
账户应保持此功能禁用,并依赖 `OpenPositions`(参见现货保证金交易)。

**工作原理:**

- 启用后,钱包余额会被转换为 `PositionStatusReport` 对象。
- 正余额会被报告为 `LONG` 仓位。
- 仅报告与已配置计价货币匹配的标的(默认:`USDT`)。
- 这可以避免当同一资产存在多种计价货币(例如 BTC/USD、BTC/USDT、
  BTC/EUR)时产生重复报告。

**配置:**

```python
exec_clients={
    KRAKEN: {
        "use_spot_position_reports": True,
        "spot_positions_quote_currency": "USDT",  # 默认值
    },
}
```

:::warning
**请谨慎使用**:如果你的策略并非设计用来处理现货仓位,启用现货仓位
报告可能导致意外行为。例如,一个预期用于平仓的策略可能会尝试卖出你
钱包中的持仓。
:::

## 现货保证金交易

Kraken Spot 在部分交易对上支持杠杆交易。各交易对的可用性和有效的杠杆
分档由 Kraken 在标的端点上以 `AssetPairInfo.leverage_buy` 和
`leverage_sell` 公布;适配器在标的加载时缓存这些信息,并在下单之前
校验所请求的分档。保证金交易通过 `spot_account_type` 按执行客户端启用,
并可通过按订单的 `leverage` 参数设置。

### 配置

```python
from nautilus_trader.adapters.kraken import KrakenExecClientConfig
from nautilus_trader.model.enums import AccountType

exec_clients = {
    KRAKEN: KrakenExecClientConfig(
        spot_account_type=AccountType.MARGIN,
        default_leverage=3,             # 可选的配置级默认值
        margin_balance_asset="ZGBP",    # 可选的摘要展示资产
    ),
}
```

`margin_balance_asset` 仅控制 Kraken `TradeBalance` 端点返回的账户
摘要指标(权益、可用保证金、已用保证金等)的计价单位。来自
`OpenPositions` 的每个仓位数据始终以该交易对的计价货币表示。

### 按订单的杠杆

可以通过 `params` 在单笔订单上覆盖已配置的默认值:

```python
order = strategy.order_factory.limit(
    instrument_id=BTC_USD,
    order_side=OrderSide.BUY,
    quantity=Quantity.from_str("0.01"),
    price=Price.from_str("50000.00"),
    params={"leverage": 5},
)
```

适配器会在提交之前,针对该交易对的 `AssetPairInfo.leverage_buy` /
`leverage_sell` 校验所请求的分档;无效的分档会产生一个 `OrderDenied`
事件,永远不会到达场所。

### Reduce-only

保证金订单可以携带 `reduce_only=True`;如果没有匹配的仓位存在,
Kraken 会拒绝该订单。现金订单会忽略该标志。

### 账户状态

当 `spot_account_type=Margin` 时,适配器会调用 Kraken 的
`TradeBalance` 端点,并将结果呈现在两个位置:

- `MarginBalance.initial`:已用保证金(`m`)。
- `AccountState.info` 字典:完整的 `TradeBalance` 快照:
  - `equity`:净权益
  - `free_margin`:权益减去已用保证金
  - `unrealized_pnl`:未平仓仓位的盈亏
  - `margin_level`:有仓位时的权益 / 已用保证金(%)
  - `trade_balance`:存入的抵押品
  - `equivalent_balance`:合并货币的钱包等值
  - `cost_basis`、`valuation`、`unexecuted_value`、`used_margin`:原始
    `TradeBalance` 字段
  - `asset`:解析出的计价资产(例如 `USD`、`GBP`)

每次账户状态刷新都会发出一条 INFO 日志:

```text
Margin metrics: equity=1234.56 GBP, free_margin=1100.00, unrealized_pnl=12.34
```

策略通过 `account_state.info["equity"]` 等方式读取这些值。

### 仓位对账

未平仓的现货保证金仓位会在每次 `position_check_interval_secs` 节拍
时,通过 `POST /0/private/OpenPositions` 呈现。在场所上已关闭、但在
本地缓存中仍显示为未平仓的仓位,会在下一次扫描时被对账为 FLAT。
此路径独立于 `use_spot_position_reports`(后者是基于钱包推导的,
仅限现金模式)。

## 资金费率

适配器从
[Ticker](https://docs.kraken.com/api/docs/futures-api/websocket/ticker)
WebSocket 数据流接收资金费率数据,该数据流为永续期货提供
`relative_funding_rate` 和 `next_funding_rate_time`。

对于 Kraken,`FundingRateUpdate` 上的 `interval` 字段为 `None`,
因为 ticker 数据流不包含资金费率间隔字段,Kraken API 文档也未指定
固定的资金结算周期。

## 速率限制

适配器实现了自动速率限制,以符合 Kraken 的 API 要求。

| 端点类型         | 限制(请求/秒) | 备注                                |
|-----------------------|----------------------|--------------------------------------|
| 现货 REST(全局)    | 5                    | 现货 API 的全局速率限制。      |
| 期货 REST(全局) | 5                    | 期货 API 的全局速率限制。   |

:::info
Kraken 使用基于计数器的速率限制系统,限制取决于账户等级:

- **Starter 等级**:计数器上限 15,每秒衰减 -0.33
- **Intermediate 等级**:计数器上限 20,每秒衰减 -0.5
- **Pro 等级**:计数器上限 20,每秒衰减 -1

Ledger/成交历史调用会使计数器 +2;其他调用 +1。
:::

:::warning
Kraken 可能会临时封锁超出速率限制的 IP 地址。适配器会在接近限制时
自动排队请求。
:::

### 对账间隔建议

执行引擎的 `open_check_interval_secs` 和 `position_check_interval_secs`
设置会产生持续的 REST API 负载,可能耗尽 Kraken 基于计数器的速率限制,
在 Starter 等级上尤其如此,因为该等级的计数器每秒只衰减 0.33。每次
未成交订单检查会产生 1-3 次 REST 调用(每次 +1 或 +2 计数器),在
较短的间隔下,计数器会在其能够衰减之前溢出,从而导致
`EAPI:Rate limit exceeded` 错误。

针对 Kraken 的推荐设置:

```python
exec_engine=LiveExecEngineConfig(
    reconciliation=True,
    open_check_interval_secs=30.0,    # Starter 等级最少 30 秒
    position_check_interval_secs=120.0,  # 2 分钟
)
```

计数器衰减更快的高等级账户可以使用更短的间隔。如果在日志中看到
`EAPI:Rate limit exceeded` 错误,请增加这些间隔,或降低适配器配置中的
`max_requests_per_second`。

## 配置

每个客户端的产品类型通过 `product_type` 选项指定。

### 数据客户端配置选项

| 选项                    | 默认值   | 说明                                                    |
|---------------------------|-----------|------------------------------------------------------------|
| `product_type`            | `SPOT`    | 该客户端的产品类型(`SPOT` 或 `FUTURES`)。            |
| `environment`             | `LIVE`    | 交易环境(`LIVE` 或 `DEMO`);`DEMO` 仅限期货。 |
| `api_key`                 | `None`    | API key;省略时从环境变量加载。       |
| `api_secret`              | `None`    | API secret;省略时从环境变量加载。    |
| `base_url`                | `None`    | Kraken REST 基础 URL 的覆盖值。                         |
| `ws_public_url`           | `None`    | 公开 WebSocket URL 的覆盖值。                         |
| `ws_private_url`          | `None`    | 私有 WebSocket URL 的覆盖值。                        |
| `ws_l3_url`               | `None`    | 现货 L3 WebSocket URL 的覆盖值。                        |
| `validate_l3_checksum`    | `True`    | 校验 Kraken 现货 L3 校验和,不匹配时重新同步。      |
| `proxy_url`               | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。          |
| `timeout_secs`            | `30`      | HTTP 请求超时时间(秒)。                               |
| `heartbeat_interval_secs` | `30`      | WebSocket 心跳间隔(秒)。                       |
| `ws_idle_timeout_ms`      | `10000`   | 现货 v2 WebSocket 的空闲超时;`0` 表示禁用。          |
| `max_requests_per_second` | `None`    | 覆盖速率限制;默认为 5 请求/秒。                       |
| `transport_backend`       | `Sockudo` | WebSocket 传输后端。                                   |

### 执行客户端配置选项

| 选项                          | 默认值   | 说明                                                           |
|---------------------------------|-----------|-------------------------------------------------------------------------|
| `api_key`                       | 必填  | Kraken API key。                                                       |
| `api_secret`                    | 必填  | Kraken API secret。                                                    |
| `product_type`                  | `SPOT`    | 该客户端的产品类型(`SPOT` 或 `FUTURES`)。                   |
| `environment`                   | `LIVE`    | 交易环境(`LIVE` 或 `DEMO`);`DEMO` 仅限期货。        |
| `base_url`                      | `None`    | Kraken REST 基础 URL 的覆盖值。                                |
| `ws_url`                        | `None`    | Kraken WebSocket URL 的覆盖值。                                |
| `proxy_url`                     | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。                 |
| `timeout_secs`                  | `30`      | HTTP 请求超时时间(秒)。                                      |
| `heartbeat_interval_secs`       | `30`      | WebSocket 心跳间隔(秒)。                              |
| `max_requests_per_second`       | `None`    | 覆盖速率限制;默认为 5 请求/秒。                              |
| `spot_account_type`             | `CASH`    | 现货交易的账户类型;`MARGIN` 启用杠杆和相关报告。 |
| `default_leverage`              | `None`    | 设置时以 `"N:1"` 形式发送的默认现货保证金杠杆。                |
| `use_spot_position_reports`     | `False`   | 将钱包余额报告为仓位;仅限现金模式。                  |
| `spot_positions_quote_currency` | `"USDT"`  | 现货钱包仓位报告的计价货币过滤器。               |
| `margin_balance_asset`          | `None`    | `TradeBalance` 的摘要资产;`None` 默认为 `ZUSD`。          |
| `transport_backend`             | `Sockudo` | WebSocket 传输后端。                                          |

对于现货保证金交易,当订单没有按订单设置的杠杆参数时,会应用
`default_leverage`。`margin_balance_asset` 仅改变 `TradeBalance` 摘要
的计价单位;每个仓位的数据仍以该交易对的计价货币表示。

### 演示环境设置

要使用 Kraken Futures 演示环境(模拟交易)进行测试:

1. 在 [https://demo-futures.kraken.com](https://demo-futures.kraken.com)
   注册并生成 API 凭证。
2. 使用你的演示凭证设置环境变量:
   - `KRAKEN_FUTURES_DEMO_API_KEY`
   - `KRAKEN_FUTURES_DEMO_API_SECRET`
3. 使用 `environment=KrakenEnvironment.DEMO` 和
   `product_type=KrakenProductType.FUTURES` 配置适配器。

```python
from nautilus_trader.adapters.kraken import KRAKEN
from nautilus_trader.adapters.kraken import KrakenEnvironment
from nautilus_trader.adapters.kraken import KrakenProductType

config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        KRAKEN: {
            "environment": KrakenEnvironment.DEMO,
            "product_type": KrakenProductType.FUTURES,
        },
    },
    exec_clients={
        KRAKEN: {
            "environment": KrakenEnvironment.DEMO,
            "product_type": KrakenProductType.FUTURES,
        },
    },
)
```

### 生产环境配置

最常见的用法是配置一个实时 `TradingNode`,使其包含 Kraken 数据客户端
和执行客户端。请在客户端配置中添加 `KRAKEN` 部分:

```python
from nautilus_trader.adapters.kraken import KRAKEN
from nautilus_trader.adapters.kraken import KrakenEnvironment
from nautilus_trader.adapters.kraken import KrakenProductType
from nautilus_trader.live.node import TradingNode

config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        KRAKEN: {
            "environment": KrakenEnvironment.LIVE,
            "product_type": KrakenProductType.SPOT,
        },
    },
    exec_clients={
        KRAKEN: {
            "environment": KrakenEnvironment.LIVE,
            "product_type": KrakenProductType.SPOT,
        },
    },
)
```

然后,创建一个 `TradingNode` 并添加客户端工厂:

```python
from nautilus_trader.adapters.kraken import KRAKEN
from nautilus_trader.adapters.kraken import KrakenDataClientFactory
from nautilus_trader.adapters.kraken import KrakenExecutionClientFactory
from nautilus_trader.live.node import TradingNode

# 使用配置实例化实时交易节点
node = TradingNode(config=config)

# 向节点注册客户端工厂
node.add_data_client_factory(KRAKEN, KrakenDataClientFactory)
node.add_exec_client_factory(KRAKEN, KrakenExecutionClientFactory)

# 最后构建节点
node.build()
```

### API 凭证

向 Kraken 客户端提供凭证有两种方式。可以将对应的 `api_key` 和
`api_secret` 值传给配置对象,或设置以下环境变量:

| 环境变量             | 说明                              |
|----------------------------------|------------------------------------------|
| `KRAKEN_SPOT_API_KEY`            | Kraken Spot 实盘交易的 API key。    |
| `KRAKEN_SPOT_API_SECRET`         | Kraken Spot 实盘交易的 API secret。 |
| `KRAKEN_FUTURES_API_KEY`         | Kraken Futures 实盘 API key。             |
| `KRAKEN_FUTURES_API_SECRET`      | Kraken Futures 实盘 API secret。          |
| `KRAKEN_FUTURES_DEMO_API_KEY`    | Kraken Futures(演示)的 API key。       |
| `KRAKEN_FUTURES_DEMO_API_SECRET` | Kraken Futures(演示)的 API secret。    |

:::note
**演示环境**:只有 Kraken Futures 提供演示环境
(`https://demo-futures.kraken.com`),可在不使用真实资金的情况下
测试。Kraken Spot 没有演示或测试网环境。
:::

:::tip
建议使用环境变量来管理凭证。
:::

启动交易节点时,你会立即收到关于凭证是否有效以及是否具有交易权限的
确认。

## 贡献

:::info
如需了解更多功能或为 Kraken 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
