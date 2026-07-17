# 基于 K 线的执行

K 线（bar）数据为每个时间段提供了市场活动的概要信息，包含四个关键价格（假设 K 线按成交明细
（trade）聚合）：

- **开盘价（Open）**：开盘价格（首笔成交）
- **最高价（High）**：期间内的最高成交价格
- **最低价（Low）**：期间内的最低成交价格
- **收盘价（Close）**：收盘价格（末笔成交）

虽然这为我们提供了价格走势的概览，但与粒度更细的数据相比，我们会丢失一些重要信息：

- 我们不知道市场在该时间段内触及最高价与最低价的先后顺序。
- 我们无法确切知道价格在该时间段内何时发生变化。
- 我们不知道该期间内实际发生的成交序列。

正因如此，尽管存在上述限制，Nautilus 仍会通过一套系统来处理 K 线数据，尽力维持尽可能真实但保守的市场
行为。从本质上讲，该平台始终维护一个订单簿（order book）模拟——即便你提供的是粒度较低的数据，例如
报价（quotes）、成交明细（trades）或 K 线（bars），模拟出的订单簿也只会具有顶层（top level）数据。

:::warning
在使用 K 线进行执行模拟时（默认在交易场所配置中通过 `bar_execution=True` 启用），Nautilus 严格要求
每根 K 线的初始化时间戳（`ts_init`）代表其**收盘时间**。这样可以确保按时间顺序进行准确处理，防止
前视偏差（look-ahead bias），并使市场更新（开 -> 高 -> 低 -> 收）与该 K 线完成的时刻保持一致。

事件时间戳（`ts_event`）可以代表 K 线的开盘时间或收盘时间：

- 若 `ts_event` 表示**收盘**时间，处理 K 线时应确保 `ts_init_delta=0`（默认）。
- 若 `ts_event` 表示**开盘**时间，应将 `ts_init_delta` 设置为等于该 K 线的持续时长，以将
  `ts_init` 移到收盘时刻。

:::

## K 线时间戳约定

如果你的数据源以**开盘时间**为 K 线打上时间戳（部分数据提供商常见此做法），你需要确保 `ts_init`
被设置为收盘时间，以获得正确的执行模拟结果。有两种处理方式：

**方式一：调整数据时间戳（推荐）**

- 使用适配器（adapter）特定的配置项，例如 `bars_timestamp_on_close=True`（例如用于 Bybit 或
  Databento 适配器），在数据摄取（ingestion）阶段自动处理该问题。
- 对于自定义数据，请在加载前手动按 K 线时长偏移时间戳（例如，对 `1-MINUTE` K 线加上 1 分钟）。
- 这种方式最为清晰，因为数据本身反映的就是收盘时间。

**方式二：使用 `ts_init_delta` 参数**

- 在调用 `BarDataWrangler.process()` 时，将 `ts_init_delta` 设置为该 K 线以纳秒表示的持续时长
  （例如，1 分钟 K 线设为 `60_000_000_000`）。
- Wrangler 会计算 `ts_init = ts_event + ts_init_delta`，从而将执行时机移到收盘时刻。
- 当你无法或不希望修改源数据的时间戳时，请使用此方式。

请务必用一小段样本数据验证你所使用数据的时间戳约定，以避免模拟结果出现偏差。时间戳处理不当会导致
前视偏差和不真实的回测结果。

## K 线数据处理

即便你提供的是 K 线数据，Nautilus 仍会像真实交易场所那样，为每个金融工具维护一个内部订单簿。

1. **时间处理：**
   - Nautilus 使用 `ts_init` 来确定执行时机，而 `ts_init` 必须代表该 K 线的收盘时间。这代表着
     该 K 线完全形成、聚合完成的那一刻。
   - 事件时间戳（`ts_event`）代表数据事件发生的时刻，根据你的数据源不同，可能与 `ts_init`
     不同：
     - 若你的 K 线以**收盘**时间打上时间戳（推荐的默认做法），在 `BarDataWrangler` 中使用
       `ts_init_delta=0`，使 `ts_init = ts_event`。
     - 若你的 K 线以**开盘**时间打上时间戳，应将 `ts_init_delta` 设置为该 K 线以纳秒表示的持续
       时长（例如，1 分钟 K 线为 60_000_000_000），以将 `ts_init` 移到收盘时刻。
   - 该平台按照 `ts_init` 对事件进行排序，从而防止回测中出现前视偏差。

:::note[K 线执行的例外情况]
在以下情况下，K 线**不会**用于执行处理（也不会更新订单簿）：

- **内部聚合的 K 线**：`AggregationSource.INTERNAL` 的 K 线会被跳过，以避免重复处理由已处理过的
  tick 数据派生出的 K 线。
- **非 L1 订单簿类型**：当交易场所的 `book_type` 配置为 `L2_MBP` 或 `L3_MBO` 时，K 线数据在执行
  处理中会被忽略，因为 K 线仅源自最优挂单价格（top-of-book）。

在这些情况下，K 线仍会被发送给策略用于分析与决策，但不会触发订单撮合或更新模拟订单簿。
:::

2. **价格处理：**
   - 平台会将每根 K 线的 OHLC 价格转换为一系列市场更新事件。
   - 默认情况下，更新顺序为：开 -> 高 -> 低 -> 收
     （可通过 `bar_adaptive_high_low_ordering` 进行配置）。
   - 若你提供多个时间周期的数据（例如同时提供 1 分钟和 5 分钟 K 线），
     平台会使用粒度更细的数据以获得最高精度。

3. **执行：**
   - 当你下单时，订单会像在真实交易场所一样，与模拟订单簿进行交互。
   - 对于 MARKET（市价）订单，会以当前模拟市场价格加上任何已配置的延迟进行执行。
   - 对于在市场中挂单的 LIMIT（限价）订单，只要该 K 线的任一价格触及或穿越了限价，就会成交。
   - 撮合引擎会随着 OHLC 价格的变化持续处理订单，而不是等待整根 K 线完整形成后再处理。

## OHLC 价格模拟

在回测执行期间，每根 K 线都会被转换为一个由四个价格点组成的序列：

1. 开盘价
2. 最高价 *（高/低之间的顺序可配置，参见下方的 `bar_adaptive_high_low_ordering`。）*
3. 最低价
4. 收盘价

该 K 线的成交量会**均等地拆分**到这四个价格点（各占 25%），其余部分加入收盘价的成交中以保持总
成交量不变。在极端情况下，若该 K 线的成交量除以 4 后小于该金融工具的最小 `size_increment`，则每个
价格点会使用最小 `size_increment`，以确保市场活动的有效性（例如，CME 集团交易所为 1 手合约）。

这些价格点的排列顺序可以通过配置交易场所时的 `bar_adaptive_high_low_ordering` 参数来控制。

Nautilus 支持两种 K 线处理模式：

1. **固定顺序**（`bar_adaptive_high_low_ordering=False`，默认）
   - 按照固定顺序处理每根 K 线：`开 -> 高 -> 低 -> 收`。
   - 简单且具有确定性的方式。

2. **自适应顺序**（`bar_adaptive_high_low_ordering=True`）
   - 根据 K 线结构估计可能的价格路径：
     - 若开盘价更接近最高价：按 `开 -> 高 -> 低 -> 收` 处理。
     - 若开盘价更接近最低价：按 `开 -> 低 -> 高 -> 收` 处理。
   - [相关研究](https://gist.github.com/stefansimik/d387e1d9ff784a8973feca0cde51e363)
     表明，这种方式在预测正确的高/低顺序方面能达到约 75%-85% 的准确率（相比之下，固定顺序在
     统计学上的准确率约为 50%）。
   - 当止盈和止损位在同一根 K 线内同时出现时，这一顺序尤为重要：它决定了哪个订单先成交。

以下示例展示了如何为一个交易场所配置自适应 K 线顺序，包括账户设置：

```python
from nautilus_trader.backtest.engine import BacktestEngine
from nautilus_trader.model.enums import OmsType, AccountType
from nautilus_trader.model import Money, Currency

# 初始化回测引擎
engine = BacktestEngine()

# 添加一个启用自适应 K 线顺序并配置了必要账户设置的交易场所
engine.add_venue(
    venue=venue,  # 你的 Venue 标识符，例如 Venue("BINANCE")
    oms_type=OmsType.NETTING,
    account_type=AccountType.CASH,
    starting_balances=[Money(10_000, Currency.from_str("USDT"))],
    bar_adaptive_high_low_ordering=True,  # 启用 K 线高/低价格的自适应排序
)
```

## 订单提交时机

第 N 根 K 线的 OHLC 序列会在 `on_bar(N)` 触发之前处理完毕。如果没有配置 `LatencyModel`
（延迟模型），在 `on_bar` 中提交的订单会立即结算，并与当前订单簿撮合，而该订单簿的顶层反映的是第 N
根 K 线的收盘价。

为交易场所附加一个 `LatencyModel`，可以推迟订单的有效到达时间。在仅有 K 线数据、且中间没有其他计时器
事件的情况下，订单会在下一根 K 线的 OHLC 扫描之后结算，因此成交价格即为该 K 线的收盘价（若延迟超过了
K 线间隔，则可能是更晚的 K 线的收盘价）。粒度更细的数据（报价、成交明细）或 K 线之间由计时器驱动的
结算，可以使订单更早地被处理，其成交依据是当时所处状态的订单簿：

```python
from nautilus_trader.backtest.models import LatencyModel

engine.add_venue(
    venue=venue,
    oms_type=OmsType.NETTING,
    account_type=AccountType.CASH,
    starting_balances=[Money(10_000, Currency.from_str("USDT"))],
    latency_model=LatencyModel(base_latency_nanos=1_000_000_000),  # 1 秒
)
```

:::note
平台并未提供原生的“下一根 K 线开盘价（next-bar-open）”执行模式。K 线的 `ts_init` 即为其收盘时间戳，
因此只有当该 K 线到达时才能知道其开盘价。若要以前一根 K 线产生的信号在该开盘价成交，则需要前视信息，
这是不允许的。
:::

## 内部 K 线聚合时机

在从 tick 数据内部聚合时间 K 线时，数据引擎会使用计时器（timer）在时间间隔边界处收盘 K 线。当数据恰好
在 K 线收盘时间戳到达时，会出现一种时序边界情况：计时器可能会在处理该边界数据之前就已触发。

可以在 `DataEngineConfig` 中配置 `time_bars_build_delay` 来延迟 K 线收盘的计时器：

```python
from nautilus_trader.config import BacktestEngineConfig
from nautilus_trader.data.config import DataEngineConfig

config = BacktestEngineConfig(
    data_engine=DataEngineConfig(
        time_bars_build_delay=1,  # 微秒
    ),
)
```

:::tip
一个较小的延迟（1 微秒）可确保边界数据在 K 线收盘之前被处理。当 tick 数据集中出现在整数时间间隔
时间戳附近时，这一设置会很有用。
:::

:::note
仅影响内部聚合的 K 线（`AggregationSource.INTERNAL`）。
:::
