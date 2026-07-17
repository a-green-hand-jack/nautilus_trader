# Databento

NautilusTrader 包含一个针对 [Databento](https://databento.com/) API
以及 [Databento Binary Encoding(DBN)](https://databento.com/docs/standards-and-conventions/databento-binary-encoding)
格式数据的适配器。Databento 仅是一个市场数据提供商。该适配器不包含
执行客户端,但你可以将其与沙盒配合用于模拟执行。你也可以将
Databento 数据与 Interactive Brokers 执行配合使用,或为加密货币交易
计算传统资产类别信号。

该适配器支持:

- 从 DBN 文件加载历史数据,并解码为 Nautilus 对象,用于回测或
  catalog 存储。
- 请求解码为 Nautilus 对象的历史数据,用于实盘交易和回测。
- 订阅解码为 Nautilus 对象的实时数据流,用于实盘交易和沙盒环境。

:::tip
[Databento](https://databento.com/signup) 为新注册用户提供 $125 的
免费数据额度。目前 Databento 允许将这些额度用于历史数据,或用于订阅
方案的首月费用。

只要谨慎地发出请求,这足以覆盖测试和评估需求。在请求数据之前,请先
检查 [/metadata.get_cost](https://databento.com/docs/api-reference-historical/metadata/metadata-get-cost)
端点。
:::

## 概览

该适配器使用 [databento-rs](https://crates.io/crates/databento) crate,
即 Databento 的官方 Rust 客户端库。

:::info
你不需要单独安装 `databento`。适配器会被编译为静态库,并在构建期间
自动链接。
:::

以下适配器类可供使用:

- `DatabentoDataLoader`:从文件加载 DBN 数据。
- `DatabentoInstrumentProvider`:通过 Databento HTTP API 获取最新或
  历史标的定义。
- `DatabentoHistoricalClient`:通过 Databento HTTP API 获取历史市场
  数据。
- `DatabentoLiveClient`:通过 Databento 的原始 TCP API 订阅实时数据流。
- `DatabentoDataClient`:面向实时交易节点的 `LiveMarketDataClient`
  实现。

:::info
大多数用户只需配置一个实时交易节点(下文有介绍),不必直接使用这些
组件。
:::

## 示例

参见[实时示例](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/databento/)。

Rust 示例位于
[`crates/adapters/databento/examples/`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/adapters/databento/examples/)。
运行时,数据测试器会订阅所配置标的的实时报价和成交:

```bash
cargo run --example databento-data-tester --package nautilus-databento
```

## Databento 文档

参见 [Databento 新用户指南](https://databento.com/docs/quickstart/new-user-guides)。
请将其与本集成指南结合参考。

## Databento Binary Encoding(DBN)

Databento Binary Encoding(DBN)是一种针对标准化市场数据的高速消息
编码与存储格式。[DBN 规范](https://databento.com/docs/standards-and-conventions/databento-binary-encoding)
包含一个自描述的元数据头,以及一组固定的结构体定义,用于标准化市场
数据的规范化方式。

该适配器将 DBN 数据解码为 Nautilus 对象。同一个 Rust 解码器处理:

- 从磁盘加载并解码 DBN 文件。
- 实时解码历史和实时数据。

## 支持的 schema

NautilusTrader 支持以下 Databento schema:

| Databento schema                                                              | Nautilus 数据类型                | 说明                     |
|:------------------------------------------------------------------------------|:----------------------------------|:--------------------------------|
| [MBO](https://databento.com/docs/schemas-and-data-formats/mbo)                | `OrderBookDelta`                  | 按单市场数据(L3)。           |
| [MBP_1](https://databento.com/docs/schemas-and-data-formats/mbp-1)            | `(QuoteTick, TradeTick \| None)`  | 按价格市场数据(L1)。           |
| [MBP_10](https://databento.com/docs/schemas-and-data-formats/mbp-10)          | `OrderBookDepth10`                | 市场深度(L2)。              |
| [BBO_1S](https://databento.com/docs/schemas-and-data-formats/bbo-1s)          | `QuoteTick`                       | 1 秒最优买卖价。        |
| [BBO_1M](https://databento.com/docs/schemas-and-data-formats/bbo-1m)          | `QuoteTick`                       | 1 分钟最优买卖价。        |
| [CMBP_1](https://databento.com/docs/schemas-and-data-formats/cmbp-1)          | `(QuoteTick, TradeTick \| None)`  | 跨场所整合的 MBP。 |
| [CBBO_1S](https://databento.com/docs/schemas-and-data-formats/cbbo-1s)        | `QuoteTick`                       | 整合的 1 秒最优买卖价。      |
| [CBBO_1M](https://databento.com/docs/schemas-and-data-formats/cbbo-1m)        | `QuoteTick`                       | 整合的 1 分钟最优买卖价。      |
| [TCBBO](https://databento.com/docs/schemas-and-data-formats/tcbbo)            | `(QuoteTick, TradeTick)`          | 按成交采样的整合最优买卖价。 |
| [TBBO](https://databento.com/docs/schemas-and-data-formats/tbbo)              | `(QuoteTick, TradeTick)`          | 按成交采样的最优买卖价。   |
| [TRADES](https://databento.com/docs/schemas-and-data-formats/trades)          | `TradeTick`                       | 成交 tick。                    |
| [OHLCV_1S](https://databento.com/docs/schemas-and-data-formats/ohlcv-1s)      | `Bar`                             | 1 秒 K 线。                  |
| [OHLCV_1M](https://databento.com/docs/schemas-and-data-formats/ohlcv-1m)      | `Bar`                             | 1 分钟 K 线。                  |
| [OHLCV_1H](https://databento.com/docs/schemas-and-data-formats/ohlcv-1h)      | `Bar`                             | 1 小时 K 线。                    |
| [OHLCV_1D](https://databento.com/docs/schemas-and-data-formats/ohlcv-1d)      | `Bar`                             | 日 K 线。                     |
| [DEFINITION](https://databento.com/docs/schemas-and-data-formats/definition)  | `Instrument`(多种类型)      | 标的定义。         |
| [IMBALANCE](https://databento.com/docs/schemas-and-data-formats/imbalance)    | `DatabentoImbalance`              | 集合竞价失衡数据。         |
| [STATISTICS](https://databento.com/docs/schemas-and-data-formats/statistics)  | `DatabentoStatistics`             | 市场统计数据。              |
| [STATUS](https://databento.com/docs/schemas-and-data-formats/status)          | `InstrumentStatus`                | 市场状态更新。          |

:::note
Databento 还文档化了参考 schema,包括公司行动、调整因子和证券主数据。
该适配器目前只映射上表列出的 schema 到 Nautilus 数据类型。每日
Databento OHLCV 使用 `ohlcv-1d`。官方结算价格和未平仓合约来自
`statistics` schema,而非 OHLCV K 线。
:::

:::info
不受支持的 `instrument_class` 取值(`'I'` 指数、`'B'` 债券)的标的
定义会被跳过并记录一条警告,而不会中止整批处理。Nautilus 无法映射
货币的外汇现货定义也会被跳过。发出指数的发布方包括
CGIF.TITANIUM(110)、IEX Options(108)和 MEMX MX2(109)。如果你
需要对这些进行 Nautilus 建模,请提交一个 issue。

`stat_type` 取值超出已建模范围(目前为 1-20)的统计消息也会被跳过并
记录一条警告。这包括超出用于持久化的 `u8` Arrow 列宽度的场所特定
取值 `VenueSpecificVolume1`(10001)和 `VenueSpecificPrice1`
(10002)。
:::

### Schema 选择考虑

- **TBBO 和 TCBBO**:按成交采样的数据流,将每笔成交与其效果*生效前*
  紧邻的 BBO 配对。当你需要与同期报价对齐的成交、且不想管理两个流时,
  使用它们。
- **MBP-1 和 CMBP-1(L1)**:事件级更新,仅在成交事件发生时发出成交。
  当你需要完整的一档事件流水记录时选择它们。若需要报价与成交对齐,
  优先使用 TBBO 或 TCBBO。
- **MBP-10(L2)**:带成交的前 10 档。当你需要深度感知但不需要完整
  MBO 数据的策略时使用。每档包含订单数。Databento 订单簿深度订阅
  仅支持 `depth=10`。
- **MBO(L3)**:用于队列位置建模和精确订单簿重建的按单事件。为获得
  正确的重放上下文,应在节点初始化时就开始订阅。
- **BBO_1S/BBO_1M 和 CBBO_1S/CBBO_1M**:以固定间隔(1 秒或 1 分钟)
  采样的一档更新。适配器仅对这些 schema 发出 `QuoteTick`。适用于
  监控、价差以及低成本信号,不适合用于微观结构研究。
- **TRADES**:仅成交。可配合 MBP-1(`include_trades=True`)使用,
  或使用 TBBO 或 TCBBO 获取带报价上下文的成交。
- **OHLCV**:由成交聚合而成的 K 线。适用于较大时间框架的分析。设置
  `bars_timestamp_on_close=True` 以使用收盘时间戳。日 K 线使用
  `ohlcv-1d`;官方结算和未平仓合约请使用 `statistics`。
- **失衡和统计数据**:场所运营数据。通过 `subscribe_data` 配合携带
  `instrument_id` 元数据的 `DataType` 订阅。
- **状态**:场所交易状态更新。通过 `subscribe_instrument_status`
  订阅。

:::tip
整合类 schema(CMBP_1、CBBO_1S、CBBO_1M、TCBBO)会跨多个场所聚合数据。
适用于跨场所分析。
:::

:::info
另请参见 Databento [Schema 与数据格式](https://databento.com/docs/schemas-and-data-formats)指南。
:::

## 数据集可用性与选择

Databento 数据集 ID 与 Nautilus 场所标识符是相互独立的。该适配器支持
上面列出的 schema,但每个 Databento 数据集会暴露自己的子集。在向实盘
配置中添加新的数据集或 schema 之前,请先检查元数据端点:

```bash
databento_auth="$(printf '%s:' "$DATABENTO_API_KEY" | base64 | tr -d '\n')"

curl --header "Authorization: Basic ${databento_auth}" \
  "https://hist.databento.com/v0/metadata.list_schemas?dataset=EQUS.MINI"

curl --header "Authorization: Basic ${databento_auth}" \
  "https://hist.databento.com/v0/metadata.list_unit_prices?dataset=EQUS.MINI"

curl --header "Authorization: Basic ${databento_auth}" \
  "https://hist.databento.com/v0/metadata.get_cost" \
  --data-urlencode "dataset=EQUS.MINI" \
  --data-urlencode "symbols=AAPL" \
  --data-urlencode "stype_in=raw_symbol" \
  --data-urlencode "schema=bbo-1s" \
  --data-urlencode "start=2026-06-24T14:30:00Z" \
  --data-urlencode "end=2026-06-24T14:31:00Z"
```

对于两个常用的评估数据集:

- `GLBX.MDP3` 是面向 CME、CBOT、NYMEX 和 COMEX 期货、期货期权及价差
  的 CME Globex MDP 3.0 数据集。它支持 MBO、MBP-1、MBP-10、TBBO、
  成交、BBO 间隔、OHLCV、定义、统计和状态。它不暴露整合类股票
  schema(`cmbp-1`、`cbbo-*` 或 `tcbbo`)。
- `EQUS.MINI` 是 Databento 的美国股票 Mini 数据集。它是一个匿名化了
  各成分场所的衍生聚合一档数据集。它支持 MBP-1、TBBO、成交、BBO
  间隔、OHLCV 和定义。它不支持 MBO、MBP-10、失衡、统计、状态或
  整合类 schema。

对于美国股票 Mini 数据集的标的,使用 `EQUS` 作为 Nautilus 场所:
`AAPL.EQUS`、`MSFT.EQUS` 等等。内置的场所到数据集映射会将 `EQUS`
路由到 `EQUS.MINI`。除非你通过 `venue_dataset_map` 覆盖它们,否则
`XNAS` 和 `XNYS` 等场所代码指的是各自专属的数据集。

:::warning
如果你将 `XNAS` 之类的场所覆盖为 `EQUS.MINI`,请保持下游标的 ID 的
一致性。Mini 记录携带整合的 `EQUS` 发布方标识,在没有显式
`instrument_id` 的情况下进行文件或历史解码会发出 `*.EQUS` 标识符。
:::

成本取决于 schema、符号和时间范围。对于探索性使用,建议从较窄的时间
范围、`definition`、`bbo-1s`、`bbo-1m` 或 `trades` 开始,并在拉取
历史时间序列数据之前调用 `metadata.get_cost`。当组合 schema
(如 `mbp-1` 或 `tbbo`)已经携带策略所需的数据时,应避免重复订阅
报价和成交。

## 实时订阅的 schema 选择

Nautilus 订阅方法与 Databento schema 的映射关系如下:

| Nautilus 订阅方法    | 默认 schema | 可用的 Databento schema                                                  | Nautilus 数据类型 |
|:--------------------------------|:---------------|:-----------------------------------------------------------------------------|:-------------------|
| `subscribe_quote_ticks()`       | `mbp-1`        | `mbp-1`、`bbo-1s`、`bbo-1m`、`cmbp-1`、`cbbo-1s`、`cbbo-1m`、`tbbo`、`tcbbo` | `QuoteTick`        |
| `subscribe_trade_ticks()`       | `trades`       | `trades`、`tbbo`、`tcbbo`、`mbp-1`、`cmbp-1`                                 | `TradeTick`        |
| `subscribe_order_book_depth()`  | `mbp-10`       | `mbp-10`                                                                     | `OrderBookDepth10` |
| `subscribe_order_book_deltas()` | `mbo`          | `mbo`                                                                        | `OrderBookDeltas`  |
| `subscribe_bars()`              | 因情况而异         | `ohlcv-1s`、`ohlcv-1m`、`ohlcv-1h`、`ohlcv-1d`                               | `Bar`              |

:::warning
“可用的 Databento schema” 列出的是该 Nautilus 订阅方法所支持的适配器
可选项。所选数据集也必须支持该 schema。例如,`EQUS.MINI` 无法提供
`mbo`、`mbp-10`、`statistics` 或 `status`。
:::

:::note
下方示例假设处于 `Strategy` 或 `Actor` 上下文中,`self` 具有订阅方法。
导入所需类型:

```python
from nautilus_trader.adapters.databento import DATABENTO_CLIENT_ID
from nautilus_trader.model import BarType
from nautilus_trader.model.enums import BookType
from nautilus_trader.model.identifiers import InstrumentId
```

:::

### 报价订阅(MBP 和 L1)

```python
# 默认 MBP-1 报价(可能包含成交)
self.subscribe_quote_ticks(instrument_id, client_id=DATABENTO_CLIENT_ID)

# 显式指定 MBP-1 schema
self.subscribe_quote_ticks(
    instrument_id=instrument_id,
    params={"schema": "mbp-1"},
    client_id=DATABENTO_CLIENT_ID,
)

# 1 秒 BBO 快照(适配器仅发出 QuoteTick)
self.subscribe_quote_ticks(
    instrument_id=instrument_id,
    params={"schema": "bbo-1s"},
    client_id=DATABENTO_CLIENT_ID,
)

# 跨场所整合报价
self.subscribe_quote_ticks(
    instrument_id=instrument_id,
    params={"schema": "cbbo-1s"},  # 或使用 "cmbp-1" 获取整合 MBP
    client_id=DATABENTO_CLIENT_ID,
)

# 按成交采样的 BBO(包含报价和成交)
self.subscribe_quote_ticks(
    instrument_id=instrument_id,
    params={"schema": "tbbo"},  # 在消息总线上接收 QuoteTick 和 TradeTick
    client_id=DATABENTO_CLIENT_ID,
)
```

### 成交订阅

```python
# 仅成交 tick
self.subscribe_trade_ticks(instrument_id, client_id=DATABENTO_CLIENT_ID)

# 来自 MBP-1 数据流的成交(仅在发生成交事件时)
self.subscribe_trade_ticks(
    instrument_id=instrument_id,
    params={"schema": "mbp-1"},
    client_id=DATABENTO_CLIENT_ID,
)

# 按成交采样的数据(包含成交时刻的报价)
self.subscribe_trade_ticks(
    instrument_id=instrument_id,
    params={"schema": "tbbo"},  # 也在成交事件时提供报价
    client_id=DATABENTO_CLIENT_ID,
)
```

### 订单簿深度订阅(MBP 和 L2)

```python
# 订阅前 10 档市场深度
self.subscribe_order_book_depth(
    instrument_id=instrument_id,
    depth=10  # 自动选择 MBP-10 schema
)

# 对于 Databento,depth 参数必须为 10
# 接收 OrderBookDepth10 更新
```

### 订单簿增量订阅(MBO 和 L3)

```python
# 订阅完整的按单订单簿更新
self.subscribe_order_book_deltas(
    instrument_id=instrument_id,
    book_type=BookType.L3_MBO  # 使用 MBO schema
)

# 在节点启动时进行 MBO 订阅,以便 Databento 可以从会话开始重放
```

### K 线订阅

```python
# 订阅 1 分钟 K 线(自动使用 ohlcv-1m schema)
self.subscribe_bars(
    bar_type=BarType.from_str(f"{instrument_id}-1-MINUTE-LAST-EXTERNAL")
)

# 订阅 1 秒 K 线(自动使用 ohlcv-1s schema)
self.subscribe_bars(
    bar_type=BarType.from_str(f"{instrument_id}-1-SECOND-LAST-EXTERNAL")
)

# 订阅小时 K 线(自动使用 ohlcv-1h schema)
self.subscribe_bars(
    bar_type=BarType.from_str(f"{instrument_id}-1-HOUR-LAST-EXTERNAL")
)

# 订阅日 K 线(自动使用 ohlcv-1d schema)
self.subscribe_bars(
    bar_type=BarType.from_str(f"{instrument_id}-1-DAY-LAST-EXTERNAL")
)
```

### 自定义数据类型订阅

失衡和统计数据需要使用通用的 `subscribe_data` 方法:

```python
from nautilus_trader.adapters.databento import DATABENTO_CLIENT_ID
from nautilus_trader.adapters.databento import DatabentoImbalance
from nautilus_trader.adapters.databento import DatabentoStatistics
from nautilus_trader.model import DataType

# 订阅失衡数据
self.subscribe_data(
    data_type=DataType(DatabentoImbalance, metadata={"instrument_id": instrument_id}),
    client_id=DATABENTO_CLIENT_ID,
)

# 订阅统计数据
self.subscribe_data(
    data_type=DataType(DatabentoStatistics, metadata={"instrument_id": instrument_id}),
    client_id=DATABENTO_CLIENT_ID,
)
```

标的状态使用专用的状态订阅 API:

```python
# 订阅标的状态更新
self.subscribe_instrument_status(
    instrument_id=instrument_id,
    client_id=DATABENTO_CLIENT_ID,
)
```

## 标的 ID 与符号规则

Databento 市场数据包含一个 `instrument_id` 字段:大多数情况下是由
发布方分配的数字 ID,若发布方未提供,则由 Databento 合成。Databento
只保证该 ID 在给定的一天内是唯一的。这与 Nautilus 的 `InstrumentId`
不同,后者是由符号和场所以句点分隔组成的字符串:
`"{symbol}.{venue}"`。

解码器将 Databento 的 `raw_symbol` 映射到 Nautilus 的 `symbol`。
发布方 ID 通过 `publishers.json` 映射到默认的 Nautilus 场所。订阅时
提供的 `InstrumentId` 元数据也可以在市场数据到达之前,预先填充符号到
场所的映射。

Databento 使用*数据集 ID* 来标识数据集,这与场所标识符是独立的。
详情参见 [Databento 数据集命名约定](https://databento.com/docs/api-reference-historical/basics/datasets)。

对于历史请求和实时订阅,适配器会将每个 `InstrumentId` 的 Nautilus
符号部分作为 Databento 符号发送,并从该字符串推断 `stype_in`:

- 以 `.FUT` 或 `.OPT` 结尾的符号使用 Databento 母символ规则,例如
  `ES.FUT.XCME`。
- 最后一部分为数字的三段式符号使用连续合约符号规则,例如
  `ES.c.0.GLBX`。
- 全数字符号使用 Databento 的 `instrument_id` 符号规则。
- 所有其他符号使用原始符号规则,例如 `ESZ6.XCME` 或 `AAPL.EQUS`。

一次请求或订阅中的所有符号必须使用相同的符号规则类型。可以将
`AAPL.EQUS` 与 `MSFT.EQUS` 批量在一起,或将 `ES.FUT.XCME` 与
`NQ.FUT.XCME` 批量在一起,但不要在一次 Databento 请求中混合使用原始
符号和母符号。

对于 CME Globex MDP 3.0(`GLBX.MDP3`),默认发布方映射到 `GLBX` 场所。
当 `use_exchange_as_venue=True` 时,定义消息可以用标的的交易所 MIC
覆盖 `GLBX`:

- `CBCM`:XCME-XCBT 跨交易所价差
- `NYUM`:XNYM-DUMX 跨交易所价差
- `XCBT`:芝加哥期货交易所(CBOT)
- `XCEC`:商品交易中心(COMEX)
- `XCME`:芝加哥商业交易所(CME)
- `XFXS`:CME FX Link 价差
- `XNYM`:纽约商业交易所(NYMEX)

:::info
其他场所 MIC 可在
[metadata.list_publishers](https://databento.com/docs/api-reference-historical/metadata/metadata-list-publishers)
端点响应的 `venue` 字段中找到。
:::

## 时间戳

Databento 数据包含以下时间戳字段:

- `ts_event`:撮合引擎收到该消息的时间戳,自 UNIX 纪元以来的纳秒数。
- `ts_in_delta`:撮合引擎发送该消息的时间戳,相对于 `ts_recv` 的
  纳秒偏移(位于其之前)。
- `ts_recv`:捕获服务器收到该消息的时间戳,自 UNIX 纪元以来的纳秒数。
- `ts_out`:Databento 发送该消息的时间戳(仅限实时)。

Nautilus 数据(按照 `Data` 契约)至少需要两个时间戳:

- `ts_event`:该数据事件发生时的 UNIX 时间戳(纳秒)。
- `ts_init`:该数据实例被创建时的 UNIX 时间戳(纳秒)。

报价和成交类 schema 将 Databento 的 `ts_recv` 映射到 Nautilus 的
`ts_event`,因为它更可靠,且按 Databento 符号单调递增。K 线使用 DBN
的 K 线区间时间戳;`bars_timestamp_on_close` 控制 Nautilus K 线使用
区间的开盘还是收盘时间戳。`InstrumentStatus` 使用解码后状态消息中的
状态事件时间戳。`DatabentoImbalance` 和 `DatabentoStatistics` 会保留
Databento 的时间戳字段,因为它们是适配器专属类型。

:::info
详情参见以下 Databento 文档:

- [Databento 标准与约定 - 时间戳](https://databento.com/docs/standards-and-conventions/common-fields-enums-types#timestamps)
- [Databento 时间戳指南](https://databento.com/docs/architecture/timestamping-guide)

:::

## 数据类型

本节将 Databento schema 映射到 Nautilus 数据类型。

:::info
参见 Databento [schema 与数据格式](https://databento.com/docs/schemas-and-data-formats)。
:::

### 标的定义

Databento 对所有标的类别使用单一 schema。解码器将每种类别映射为
适当的 Nautilus `Instrument` 类型。

| Databento 标的类别 | 代码 | Nautilus 标的类型 |
|----------------------------|------|--------------------------|
| 股票                      | `K`  | `Equity`                 |
| 期货                     | `F`  | `FuturesContract`        |
| 看涨期权                       | `C`  | `OptionContract`         |
| 看跌期权                        | `P`  | `OptionContract`         |
| 期货价差              | `S`  | `FuturesSpread`          |
| 期权价差              | `T`  | `OptionSpread`           |
| 混合价差               | `M`  | `OptionSpread`           |
| 外汇现货                    | `X`  | `CurrencyPair`           |
| 指数                      | `I`  | 尚不可用        |
| 债券                       | `B`  | 尚不可用        |

### 期权到期时间修正

OPRA 期权定义(数据集 `OPRA.PILLAR`)携带的到期时间只精确到日期级别:
时刻部分被清零为 UTC 午夜。因此,一个在纽约时间 16:00 到期的期权,
会被标记为纽约时间前一晚,这会让撮合引擎在其最后交易时段之前就将该
合约视为已到期。加载器默认将这类 UTC 午夜的 OPRA 到期时间修正为纽约
时间 16:00,而其他所有数据集(以及任何已携带日内时间的到期时间,例如
CME Globex)保持不变。

可以使用 `expiration_overrides` 覆盖默认值,或按标的资产分别设置时间。
它将数据集映射到一个从标的资产符号到时间的映射,其中保留键
`default` 设置整个数据集范围的时间:

```python
loader.from_dbn_file(
    path,
    expiration_overrides={
        "OPRA.PILLAR": {"default": "16:00", "SPX": "09:30"},
    },
)
```

时间使用交易所本地时区的 `HH:MM` 或 `HH:MM:SS`(OPRA 为纽约时区)。
只有内置修正规则的数据集(目前是 `OPRA.PILLAR`)可以调整;未知或
没有规则的数据集(例如 `GLBX.MDP3`)会抛出 `ValueError`。该修正是
按期权的标的资产进行键控的,因此无法区分共享同一标的资产但结算时间
不同的系列(例如 AM 结算的 SPX 与 PM 结算的 SPXW);请设置与你所
加载合约匹配的时间。

### 价格精度

Databento 的原始价格是按 1e-9 缩放的定点整数。适配器根据定义消息中
标的的最小变动单位推导价格精度。

对于实时数据流,数据流处理器维护一个按标的的精度映射,随着
`InstrumentDefMsg` 记录到达而填充。市场数据处理器按以下顺序解析精度:

1. Databento 记录 `instrument_id` 对应的 `InstrumentDefMsg` 元数据。
2. Python 订阅路径传入的缓存标的精度。
3. 传给直接实时客户端的显式 `price_precisions`。
4. USD 的默认精度 2。

回退映射按符号映射之后的 Databento 记录 `instrument_id` 进行键控,
因此在定义元数据到达之前,母符号、连续合约以及其他非原始符号规则的
请求仍可以使用缓存或显式精度。

对于具有非标准最小变动单位的标的(例如带有 1/256 分数 tick 的国债
期货),**标的定义必须先于市场数据到达**,才能获得正确的精度。请在
市场数据订阅之前或同时,为你的标的订阅 `DEFINITION` schema。

对于历史请求和基于文件的加载,精度按以下顺序针对每条记录解析:

1. 调用时传入的显式 `price_precision` 参数。
2. 通过加载定义(文件加载器上的 `load_instruments`、历史客户端上的
   `get_range_instruments`)或显式调用
   `set_price_precision(symbol, precision)` 填充的按符号缓存。

Python 数据客户端会在每次请求之前,从标的提供者向历史客户端缓存注入
数据,因此已加载的标的无需额外配置。当精度无法解析时,加载会以明确
的错误失败,而不是静默地回退到 USD 精度。

:::tip
Python 适配器会在市场数据之前自动订阅标的定义,并将缓存的标的精度
作为回退传入,因此精度映射无需额外配置即可填充。若直接使用 Rust
客户端,请在市场数据之前订阅 `DEFINITION` schema,或传入显式的精度
回退值。
:::

### MBO(按单市场数据)

MBO 是 Databento 提供的粒度最细的数据,代表完整的订单簿深度。部分
消息包含成交数据。解码器会生成一个 `OrderBookDelta`,以及一个可选的
`TradeTick`。

实时客户端会缓冲 MBO 消息,直到看到 `F_LAST` 标志,然后将一个
`OrderBookDeltas` 容器传给处理器。

在重放启动序列期间,客户端也会将订单簿快照缓冲为 `OrderBookDeltas`。

### MBP-1(按价格市场数据,一档)

MBP-1 代表一档报价和成交。部分消息携带成交数据。解码器会生成一个
`QuoteTick`,当消息为成交时还会生成一个 `TradeTick`。

### TBBO 和 TCBBO(带成交的一档数据)

TBBO 和 TCBBO 在每条消息中同时提供报价和成交数据。这两种 schema 都
会为每条消息发出 `QuoteTick` 和 `TradeTick`,比分开订阅报价和成交
更高效。TCBBO 提供跨场所的整合数据。

#### 成交 ID 推导(CMBP-1 和 TCBBO)

CMBP-1 和 TCBBO schema 不发布原生的成交标识符。解码器通过对标的 ID、
`ts_event`、`ts_recv`、价格、数量和成交发起方进行 FNV-1a 哈希计算,
推导出一个确定性的 `TradeId`。相同的场所事件在多次回放中会产生相同
的成交 ID,因此下游去重保持有效。两笔字段完全相同但逻辑上不同的
成交会发生哈希碰撞;这与场所本身无法区分它们的情况相符。

### OHLCV(K 线聚合)

Databento 对 K 线消息按区间的**开盘**时间打时间戳。默认情况下,
解码器会将 K 线的 `ts_event` 规范化为区间的**收盘**时间:即原始
`ts_event` 加上该区间长度。`ts_init` 使用实时接收时间,或在历史/
文件加载且未提供显式初始化时间戳时使用收盘时间。将
`bars_timestamp_on_close` 设为 `False` 可将 K 线的 `ts_event` 打为
区间开盘时间戳。

### 失衡和统计数据

`imbalance` 和 `statistics` schema 没有内置的 Nautilus 等价类型。
适配器在 Rust 中定义了 `DatabentoImbalance` 和 `DatabentoStatistics`。

PyO3 绑定在 Python 中暴露了这些类型。它们的属性是 PyO3 对象,可能
无法与期望 Cython 类型的方法配合使用。PyO3 到 Cython 的转换方法参见
API 参考文档。

将 PyO3 的 `Price` 转换为 Cython 的 `Price`:

```python
price = Price.from_raw(pyo3_price.raw, pyo3_price.precision)
```

请求和订阅这些类型需要使用通用的 `subscribe_data` 方法。为
`AAPL.XNAS` 订阅 `imbalance`:

```python
from nautilus_trader.adapters.databento import DATABENTO_CLIENT_ID
from nautilus_trader.adapters.databento import DatabentoImbalance
from nautilus_trader.model import DataType

instrument_id = InstrumentId.from_str("AAPL.XNAS")
self.subscribe_data(
    data_type=DataType(DatabentoImbalance, metadata={"instrument_id": instrument_id}),
    client_id=DATABENTO_CLIENT_ID,
)
```

为 `ES.FUT` 母符号(所有活跃的 E-mini 标普 500 期货)请求一段有界
范围的 `statistics`。在进行真实的历史数据拉取之前,请使用
Databento 的 Historical
[`metadata.get_cost`](https://databento.com/docs/api-reference-historical/metadata/metadata-get-cost)
端点:

```python
from nautilus_trader.adapters.databento import DATABENTO_CLIENT_ID
from nautilus_trader.adapters.databento import DatabentoStatistics
from nautilus_trader.model import DataType

instrument_id = InstrumentId.from_str("ES.FUT.GLBX")
metadata = {
    "instrument_id": instrument_id,
    "start": "2024-03-06",
    "end": "2024-03-07",
}
self.request_data(
    data_type=DataType(DatabentoStatistics, metadata=metadata),
    client_id=DATABENTO_CLIENT_ID,
)
```

### Catalog 持久化

这两种类型都支持 Arrow 序列化以用于 catalog 存储。导入适配器包时,
Arrow 序列化器会自动注册。

#### 写入 catalog

```python
from nautilus_trader.adapters.databento import DatabentoDataLoader
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.persistence.catalog import ParquetDataCatalog

catalog = ParquetDataCatalog.from_env()
loader = DatabentoDataLoader()

imbalances = loader.from_dbn_file(
    path="aapl-imbalance.dbn.zst",
    instrument_id=InstrumentId.from_str("AAPL.XNAS"),
    as_legacy_cython=False,  # Databento 专属类型需要此设置
)

catalog.write_data(imbalances)
```

#### 从 catalog 读取

```python
from nautilus_trader.adapters.databento import DatabentoImbalance

results = catalog.query(DatabentoImbalance, identifiers=["AAPL.XNAS"])

for imbalance in results:
    print(imbalance.ref_price)  # DatabentoImbalance 字段
```

:::warning
Catalog 持久化支持写入和查询这些类型,但目前尚不支持通过
`BacktestNode` 或 `BacktestEngine` 对它们进行流式处理。对于需要使用
失衡或统计数据的回测,请直接查询 catalog,并在策略或分析代码中处理
结果。
:::

#### 在 Rust 中编码和解码

`nautilus_databento::arrow` 模块提供了 Arrow record batch 的编码和
解码功能。需要启用 `arrow` 特性标志。

```rust
use nautilus_databento::arrow::imbalance::{
    decode_imbalance_batch,
    imbalance_to_arrow_record_batch,
};

let batch = imbalance_to_arrow_record_batch(imbalances)?;

let metadata = batch.schema().metadata().clone();
let decoded = decode_imbalance_batch(&metadata, batch)?;
```

`statistics` 模块遵循相同的模式,使用 `decode_statistics_batch` 和
`statistics_to_arrow_record_batch`。

## 性能考虑

使用 DBN 数据进行回测有两种方式:

- 将数据以 DBN(`.dbn.zst`)文件存储,每次运行时都解码为 Nautilus
  对象。
- 将 DBN 文件一次性转换为 Nautilus 对象,并写入数据 catalog
  (Nautilus Parquet 格式)。

DBN 解码器是经过优化的 Rust 实现,但一次性写入 catalog 能带来最佳的
回测性能。

[DataFusion](https://arrow.apache.org/datafusion/) 能以高吞吐量从
磁盘流式读取 Nautilus Parquet 数据,比每次运行都解码 DBN 至少快一个
数量级。

:::note
性能基准测试正在开发中。
:::

对于实时数据,从数据流处理器到 Nautilus 的解码后传递是有意设计为
无界的。这可以防止慢速消费者拖慢数据流路径;处于内存压力下的进程
应该失败,而不是阻塞实时解码。

## 加载 DBN 数据

`DatabentoDataLoader` 类用于加载 DBN 文件,并将记录转换为 Nautilus
对象。两个主要用途:

- 将数据传给 `BacktestEngine.add_data` 用于回测。
- 将数据写入 `ParquetDataCatalog`,以便与 `BacktestNode` 一起流式
  处理。

### 将 DBN 数据传给 BacktestEngine

加载 DBN 数据并传给 `BacktestEngine`。该引擎需要一个标的。本示例
使用 `TestInstrumentProvider`(从 DBN 文件解析出的标的同样适用)。
该数据涵盖了纳斯达克上一个月的 TSLA 成交数据:

```python
# 添加标的
TSLA_NASDAQ = TestInstrumentProvider.equity(symbol="TSLA")
engine.add_instrument(TSLA_NASDAQ)

# 将数据解码为 Cython 对象
loader = DatabentoDataLoader()
trades = loader.from_dbn_file(
    path=TEST_DATA_DIR / "databento" / "temp" / "tsla-xnas-20240107-20240206.trades.dbn.zst",
    instrument_id=TSLA_NASDAQ.id,
)

# 添加数据
engine.add_data(trades)
```

### 将 DBN 数据写入 ParquetDataCatalog

加载 DBN 数据并写入 `ParquetDataCatalog`。设置
`as_legacy_cython=False` 以解码为 PyO3 对象。

### 加载标的

**重要**:在将市场数据加载进 catalog 之前,请先从 DEFINITION schema
文件加载标的定义。Catalog 在能够存储市场数据之前需要标的信息。
市场数据文件本身不包含标的定义。

```python
# 初始化 catalog 接口
# (会使用 NAUTILUS_PATH 环境变量作为路径)
catalog = ParquetDataCatalog.from_env()

loader = DatabentoDataLoader()

# 步骤 1:首先加载标的定义
# 从 Databento 获取你所需标的的 DEFINITION schema 文件
instruments = loader.from_dbn_file(
    path=TEST_DATA_DIR / "databento" / "temp" / "tsla-xnas-definition.dbn.zst",
    as_legacy_cython=False,  # 使用 PyO3 以获得最佳性能
)

# 将标的写入 catalog
catalog.write_data(instruments)

# 步骤 2:现在加载并写入市场数据
instrument_id = InstrumentId.from_str("TSLA.XNAS")

# 将成交解码为 PyO3 对象
trades = loader.from_dbn_file(
    path=TEST_DATA_DIR / "databento" / "temp" / "tsla-xnas-20240107-20240206.trades.dbn.zst",
    instrument_id=instrument_id,
    as_legacy_cython=False,  # 这是写入 catalog 的一项优化
)

# 写入市场数据
catalog.write_data(trades)
```

#### 加载多种数据类型用于回测

始终在市场数据之前加载标的:

```python
from nautilus_trader.adapters.databento.loaders import DatabentoDataLoader
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.persistence.catalog import ParquetDataCatalog

catalog = ParquetDataCatalog.from_env()
loader = DatabentoDataLoader()

# 步骤 1:从 DEFINITION 文件加载标的定义
instruments = loader.from_dbn_file(
    path="equity-definitions.dbn.zst",
    as_legacy_cython=False,
)
catalog.write_data(instruments)

# 步骤 2:加载市场数据(MBO、成交、报价等)
instrument_id = InstrumentId.from_str("AAPL.XNAS")

# 加载 MBO 订单簿增量
deltas = loader.from_dbn_file(
    path="aapl-mbo.dbn.zst",
    instrument_id=instrument_id,  # 可选,但能提升性能
    as_legacy_cython=False,
)
catalog.write_data(deltas)

# 加载成交
trades = loader.from_dbn_file(
    path="aapl-trades.dbn.zst",
    instrument_id=instrument_id,
    as_legacy_cython=False,
)
catalog.write_data(trades)

# 验证标的已在 catalog 中
print(catalog.instruments())  # 显示已加载的标的
```

:::tip
调用 `catalog.instruments()` 进行验证。空列表意味着你需要先加载
DEFINITION 文件。
:::

:::info
通过 Databento API 或 CLI,为你的符号和日期范围下载 DEFINITION
schema 文件。详情参见
[Databento 文档](https://databento.com/docs/api-reference-historical/timeseries/timeseries-get-range)。
:::

:::info
另请参见[数据概念指南](../concepts/data/)。
:::

### 历史加载器选项

`from_dbn_file` 的参数:

- `instrument_id`:通过跳过符号规则查找来加快解码速度。
- `price_precision`:应用于每条读取记录的覆盖值。若省略,加载器会
  从其缓存(由 `load_instruments` 或 `set_price_precision` 填充)
  中按符号解析精度;若无法解析,加载会失败。
- `include_trades`:对于 MBP-1/CMBP-1 schema,`True` 会在存在成交
  数据时同时发出 `QuoteTick` 和 `TradeTick`。
- `as_legacy_cython`:对于 IMBALANCE/STATISTICS schema,应设为
  `False`(必需),或为获得更好的 catalog 写入性能设置为 `False`。

:::warning
IMBALANCE 和 STATISTICS schema 要求 `as_legacy_cython=False`
(仅限 PyO3 类型)。设为 `True` 会抛出 `ValueError`。
:::

### 加载整合数据

整合类 schema 会跨多个场所聚合数据:

```python
# 加载整合的 MBP-1 报价
loader = DatabentoDataLoader()
cmbp_quotes = loader.from_dbn_file(
    path="consolidated.cmbp-1.dbn.zst",
    instrument_id=InstrumentId.from_str("AAPL.XNAS"),
    include_trades=True,  # 若可用,同时包含报价和成交
    as_legacy_cython=True,
)

# 加载整合的 BBO 报价
cbbo_quotes = loader.from_dbn_file(
    path="consolidated.cbbo-1s.dbn.zst",
    instrument_id=InstrumentId.from_str("AAPL.XNAS"),
    as_legacy_cython=False,  # 使用 PyO3 以获得更好的性能
)

# 加载带报价和成交的 TCBBO(按成交采样的整合 BBO)
# include_trades=True 加载报价,include_trades=False 加载成交
tcbbo_quotes = loader.from_dbn_file(
    path="consolidated.tcbbo.dbn.zst",
    instrument_id=InstrumentId.from_str("AAPL.XNAS"),
    include_trades=True,  # 加载报价
    as_legacy_cython=True,
)

tcbbo_trades = loader.from_dbn_file(
    path="consolidated.tcbbo.dbn.zst",
    instrument_id=InstrumentId.from_str("AAPL.XNAS"),
    include_trades=False,  # 加载成交
    as_legacy_cython=True,
)
```

:::tip
避免为同一标的同时订阅 TBBO/TCBBO 和独立的成交数据流。这些 schema
已经包含成交数据。重复订阅会浪费成本,并产生重复数据。
:::

## 实时客户端架构

`DatabentoDataClient` 封装了其他 Databento 适配器类。每个数据集使用
两个 `DatabentoLiveClient` 实例:

- 一个用于 MBO(订单簿增量)实时数据流
- 一个用于所有其他实时数据流

:::warning
请在节点启动时为某数据集完成所有 MBO 订阅,以便从会话开始重放。
客户端会将启动之后的订阅记录为错误并忽略它们。

此限制不适用于其他 schema。
:::

一个单一的 `DatabentoHistoricalClient` 同时为
`DatabentoInstrumentProvider` 和 `DatabentoDataClient` 提供历史请求
服务。

## 配置

在你的 `TradingNode` 客户端配置中添加一个 `DATABENTO` 部分。请加载
特定的标的;适配器不支持对 Databento 数据集使用
`load_all=True`,因为一个数据集可能包含数百万条定义。

```python
from nautilus_trader.adapters.databento import DATABENTO
from nautilus_trader.config import InstrumentProviderConfig
from nautilus_trader.config import TradingNodeConfig
from nautilus_trader.model.identifiers import InstrumentId

instrument_ids = [
    InstrumentId.from_str("ESZ6.XCME"),  # GLBX.MDP3
    # InstrumentId.from_str("AAPL.EQUS"),  # EQUS.MINI
]

config = TradingNodeConfig(
    data_clients={
        DATABENTO: {
            "api_key": None,  # 'DATABENTO_API_KEY' 环境变量
            "http_gateway": None,  # 默认 HTTP 历史网关的覆盖值
            "live_gateway": None,  # 默认原始 TCP 实时网关的覆盖值
            "instrument_provider": InstrumentProviderConfig(
                load_ids=frozenset(instrument_ids),
            ),
            "instrument_ids": instrument_ids,  # 启动时加载的定义
            "parent_symbols": {"GLBX.MDP3": {"ES.FUT"}},  # 可选的定义树
        },
    },
)
```

创建 `TradingNode` 并注册工厂:

```python
from nautilus_trader.adapters.databento.factories import DatabentoLiveDataClientFactory
from nautilus_trader.live.node import TradingNode

# 使用配置创建实时交易节点
node = TradingNode(config=config)

# 向节点注册客户端工厂
node.add_data_client_factory(DATABENTO, DatabentoLiveDataClientFactory)

# 构建节点
node.build()
```

### 配置参数

| 选项                    | 默认值 | 说明                                                           |
|---------------------------|---------|-------------------------------------------------------------------------|
| `api_key`                 | `None`  | Databento API secret;回退到 `DATABENTO_API_KEY`。              |
| `http_gateway`            | `None`  | 历史 HTTP 端点覆盖值,主要用于测试。                  |
| `live_gateway`            | `None`  | 实时 TCP 端点覆盖值,主要用于测试。                         |
| `instrument_provider`     | 默认 | 提供者设置;请使用 `load_ids`,而非 `load_all=True`。               |
| `use_exchange_as_venue`   | `True`  | 对 GLBX 定义使用交易所 MIC 作为场所。                         |
| `timeout_initial_load`    | `15.0`  | 每个数据集的定义加载超时时间(秒)。                      |
| `mbo_subscriptions_delay` | `3.0`   | 开始 MBO/L3 数据流之前的延迟(秒)。                     |
| `bars_timestamp_on_close` | `True`  | 对 `ts_event` 使用 K 线收盘时间;`False` 使用开盘时间。                 |
| `reconnect_timeout_mins`  | `10`    | 重试窗口(分钟);`None` 表示无限期重试。                 |
| `venue_dataset_map`       | `None`  | 覆盖场所到数据集的映射。                                   |
| `parent_symbols`          | `None`  | 按数据集预加载母符号定义树。                           |
| `instrument_ids`          | `None`  | 启动时预加载的定义。                                    |

:::tip
使用环境变量管理凭证。
:::

### 连接稳定性

实时客户端会在以下情况下自动重连:

- **网络中断**:临时的连接性问题。
- **网关重启**:Databento 计划中的实时网关重启。参见
  [维护计划](https://databento.com/docs/api-reference-live/basics#maintenance-schedule)。
- **市场休市**:会话在非交易时段结束。

#### 重连策略

退避策略取决于超时配置:

**带超时**(默认 10 分钟):

- 指数退避,上限为 **60 秒**。
- 模式:1s、2s、4s、8s、16s、32s、60s、60s,依此类推(带抖动)。
- 在超时窗口内快速重连。

**不带超时**(`reconnect_timeout_mins=None`):

- 指数退避,上限为 **10 分钟**。
- 模式:1s、2s、4s、8s、16s、32s、64s、128s、256s、512s、600s、
  600s,依此类推(带抖动)。
- 适用于需要熬过隔夜休市和计划维护的无人值守系统。

所有重连都包含:

- **抖动**:随机延迟(最多 1 秒),防止同时发生的重连风暴。
- **自动重新订阅**:重连后恢复所有活跃订阅。
- **周期重置**:每次成功的会话(> 60 秒)都会重置超时计时器。

单独的取消订阅请求会记录一条警告并被忽略,因为 Databento 实时会话
不支持细粒度的取消订阅。要从实时网关移除某个订阅,需要停止该会话。

#### 超时配置

`reconnect_timeout_mins` 参数控制客户端尝试重连的时长:

**默认(10 分钟)**:适用于大多数使用场景。

- 处理瞬时网络问题。
- 熬过计划中的网关重启。
- 市场休市时,过夜会停止重试。
- 更长时间的中断需要人工干预。

:::warning
设置 `reconnect_timeout_mins=None` 会导致无限期重试。仅在需要熬过
隔夜市场休市的无人值守系统中使用。这可能会掩盖持续存在的配置或
身份验证问题。
:::

#### 计划维护

Databento 按以下计划重启实时网关(所有客户端都会断开连接):

| 数据集            | 重启时间      |
|--------------------|-------------------|
| CME Globex         | 周六 CT 02:15 |
| 所有 ICE 场所     | 周日 UTC 09:45  |
| 所有其他数据集 | 周日 UTC 10:30  |

默认的 10 分钟超时能够覆盖典型的重启。对于无人值守系统,请使用
`reconnect_timeout_mins=None` 或更长的值。详情参见
[Databento 维护计划](https://databento.com/docs/api-reference-live/basics/maintenance-schedule)。

## 贡献

:::info
如需贡献代码,参见
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
