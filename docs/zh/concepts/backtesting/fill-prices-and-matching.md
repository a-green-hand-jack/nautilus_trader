# 成交价格与撮合

## 成交建模理念

NautilusTrader 在回测过程中将历史订单簿和成交数据视为**不可变的（immutable）**。市场中实际发生的
一切都会被原样保留，与记录时完全一致。成交（fill）永远不会修改底层的订单簿状态。

这弥补了学术文献中的一处空白：大多数研究关注的是订单簿会实际演变的实盘市场动态。而使用冻结快照进行
历史回测，则是一个截然不同的工程问题：在数据不会因我们的订单而发生变化的情况下，我们该如何模拟出
真实的成交？

**设计选择：**

- **不可变的历史数据：** 订单簿和成交数据永远不会被修改。
- **可选的消耗跟踪：** 当 `liquidity_consumption=True` 时，引擎会按价格档位跟踪已消耗的流动性，
  以防止重复成交。配置详情参见 [订单簿不可变性](#order-book-immutability)。
- **可复现的结果：** 固定的 `random_seed` 会锁定概率性成交模型所使用的伪随机数生成器（PRNG）。
  同进程内的重复运行预计会得到一致的结果；跨进程的重复运行在极少数情况下可能因成交模型之外的哈希
  排序效应而有所不同。

## 成交价格的确定

撮合引擎根据订单类型、订单簿类型和市场状态来确定成交价格。

### L2/L3 订单簿数据

在拥有完整订单簿深度的情况下，成交由实际的订单簿模拟决定：

| 订单类型                | 成交价格                                                     |
| ------------------------ | -------------------------------------------------------------- |
| `MARKET`                 | 沿订单簿逐档撮合，在每个价格档位成交（吃单方）。                |
| `MARKET_TO_LIMIT`        | 沿订单簿逐档撮合，在每个价格档位成交（吃单方）。                |
| `LIMIT`                  | 撮合时使用订单自身的限价（挂单方）。                            |
| `STOP_MARKET`            | 触发时沿订单簿逐档撮合。                                        |
| `STOP_LIMIT`             | 触发并撮合时使用订单自身的限价。                                |
| `MARKET_IF_TOUCHED`      | 触发时沿订单簿逐档撮合。                                        |
| `LIMIT_IF_TOUCHED`       | 触发时使用订单自身的限价。                                      |
| `TRAILING_STOP_MARKET`   | 激活并触发时沿订单簿逐档撮合。                                  |
| `TRAILING_STOP_LIMIT`    | 激活、触发并撮合时使用订单自身的限价。                          |

在使用 L2/L3 数据时，若最优档位的流动性不足，市价类订单可能会跨多个价格档位部分成交。限价类订单
触发后会作为挂单，若市场未到达其限价，则可能一直无法成交。`MARKET_TO_LIMIT` 会先以吃单方身份成交
一部分数量，剩余数量则以其首次成交价格作为限价挂单。

### L1 订单簿数据（报价、成交明细、K 线）

在只有最优挂单数据的情况下，使用相同的订单簿模拟机制，但仅有单一档位：

| 订单类型                | BUY 成交价格 | SELL 成交价格 |
| ------------------------ | ------------- | -------------- |
| `MARKET`                 | 最优卖价      | 最优买价       |
| `MARKET_TO_LIMIT`        | 最优卖价      | 最优买价       |
| `LIMIT`                  | 限价          | 限价           |
| `STOP_MARKET`            | 最优卖价      | 最优买价       |
| `STOP_LIMIT`             | 限价          | 限价           |
| `MARKET_IF_TOUCHED`      | 最优卖价      | 最优买价       |
| `LIMIT_IF_TOUCHED`       | 限价          | 限价           |
| `TRAILING_STOP_MARKET`   | 最优卖价      | 最优买价       |
| `TRAILING_STOP_LIMIT`    | 限价          | 限价           |

在使用 L1 数据时，模拟订单簿每一侧只有单一价格档位。订单以该档位的可用数量成交。若某订单在耗尽最优
挂单流动性后仍有剩余数量，市价单及可成交的限价类订单会滑一个 tick 以成交剩余部分。

对于 K 线数据而言，当该 K 线在其高/低价处理过程中穿越触发价时，`STOP_MARKET` 和
`TRAILING_STOP_MARKET` 订单可能会以触发价而非最优买/卖价成交。详情参见
[使用 K 线数据时止损单的成交行为](#stop-order-fill-behavior-with-bar-data)。

:::note
成交模型可以改变这些成交价格。有关配置执行模拟的详情，请参见 [成交模型](fill-models.md)。
:::

### 订单类型语义

- **市价执行：** 以当前市场价格（买价/卖价）成交。这模拟了真实交易所的行为，即此类订单在触发后以
  最优可用价格成交。例外情况：使用 K 线数据时，在高/低价处理过程中被触发的 `STOP_MARKET` 和
  `TRAILING_STOP_MARKET` 订单会以触发价格成交（见下文）。
- **限价执行：** 撮合时以订单的限价成交。可提供价格保证，但若市场未到达该限价，则可能无法成交。

### 使用 K 线数据时止损单的成交行为

在仅使用 K 线数据（无 tick 数据）进行回测时，撮合引擎会针对 `STOP_MARKET` 和
`TRAILING_STOP_MARKET` 订单区分以下两种情形：

**跳空情形**（K 线开盘价越过触发价）：当某根 K 线的开盘价跳空越过触发价时，止损单会立即触发，并
以市场价格（即开盘价）成交。这模拟了真实交易所中止损市价单在跳空时无价格保证的行为。

示例——触发价为 100 的 SELL `STOP_MARKET`：

- 上一根 K 线收于 105。
- 下一根 K 线开于 90（隔夜跳空下跌）。
- 止损单在开盘时触发，并以 90 成交。

**穿越情形**（K 线穿越触发价）：当某根 K 线正常开盘，随后其最高价或最低价穿越了触发价时，止损单
会以触发价成交。由于我们只有 OHLC 数据，我们假设市场是平滑地穿越触发价的，因此该订单应当在此处
成交。

示例——触发价为 100 的 SELL `STOP_MARKET`：

- K 线开于 102（没有跳空）。
- K 线最低价达到 98，穿越了触发价 100。
- 止损单以 100（触发价）成交。

这种行为在有序市场波动期间限定了潜在的滑点，同时仍能准确模拟跳空时的滑点。若需要 tick 级别的精度，
请使用报价或成交明细数据，而非 K 线数据。

## 价格保护

价格保护（Price protection）定义了一个由交易所计算出的价格边界，以防止可成交订单以过于激进的价格
成交。它模拟了 Binance 和 CME 等交易所针对市价单和止损市价单实施的保护机制。

**配置：**

```python
from nautilus_trader.backtest.config import BacktestVenueConfig

venue_config = BacktestVenueConfig(
    name="BINANCE",
    oms_type="NETTING",
    account_type="MARGIN",
    starting_balances=["100_000 USDT"],
    price_protection_points=100,  # 100 个点 = 对于 2 位小数的金融工具偏移 1.00
)
```

**工作原理：**

撮合引擎根据成交时刻当前的最优买/卖价计算保护边界：

- **BUY（买入）订单：** `protection_price = ask + (points × price_increment)`
- **SELL（卖出）订单：** `protection_price = bid - (points × price_increment)`

引擎会过滤掉超出保护边界的成交。举例来说，当 `price_protection_points=100`、且金融工具的
`price_increment=0.01` 时：

- 最优卖价为 1001.00。
- 保护价格 = 1001.00 + (100 × 0.01) = 1002.00。
- 一笔 BUY 市价单只会以 ≤ 1002.00 的价格成交。
- 1003.00 或更高价位的流动性会被过滤掉，导致订单部分成交。

**触发时机语义：**

引擎在成交时刻而非订单提交时刻计算保护边界：

- **市价单：** 保护边界在订单处理时立即计算。
- **止损市价单：** 保护边界在止损触发时，依据触发那一刻的买/卖价计算。

这种设计允许即使订单簿另一侧为空，止损单也能被提交，因为引擎会在止损触发时才计算保护边界。

**受影响的订单类型：**

- `MARKET`
- `STOP_MARKET`

限价单不受影响，因为它们本身已经定义了价格边界。

:::note
将 `price_protection_points=0` 设置为禁用价格保护（默认行为）。
:::

## 订单簿不可变性

在回测期间，历史订单簿数据是不可变的。当你的订单与订单簿流动性成交时，订单簿状态保持不变。这保留了
历史数据的完整性。

撮合引擎可以选择性地使用**按档位的消耗跟踪（per-level consumption tracking）**，以防止重复成交，同时
在有新流动性到达时仍允许成交。此行为由 `liquidity_consumption` 配置项控制。

**配置：**

```python
from nautilus_trader.backtest.config import BacktestVenueConfig

venue_config = BacktestVenueConfig(
    name="SIM",
    oms_type="NETTING",
    account_type="CASH",
    starting_balances=["100_000 USD"],
    liquidity_consumption=True,  # 启用消耗跟踪（默认：False）
)
```

- `liquidity_consumption=False`（默认）：每次迭代都会独立地依据完整的订单簿流动性进行成交。
  这种方式更简单，假设你只是一个小规模参与者，你的订单不会对可用流动性产生实质性影响。
- `liquidity_consumption=True`：按价格档位跟踪已消耗的流动性，防止同一份显示的流动性产生多次
  成交。当该档位到达新数据时会重置。

**消耗跟踪的工作原理（启用时）：**

对于每个价格档位，引擎维护以下信息：

- `original_size`：开始跟踪时订单簿的数量。
- `consumed`：已针对该档位成交的数量。

处理一笔成交时：

1. 检查订单簿当前该档位的数量是否与 `original_size` 相符。
2. 如果不同（表示有新数据到达），重置该条目：`original_size = current_size`，
   `consumed = 0`。
3. 计算 `available = original_size - consumed`。
4. 成交完成后，将 `consumed` 增加相应的成交数量。

**示例：**

1. 订单簿显示卖价 100.00 处有 100 单位。引擎记录：`(original=100, consumed=0)`。
2. 你的 BUY 订单成交 30 单位。引擎更新：`(original=100, consumed=30)`。可用数量 = 70。
3. 另一笔 BUY 订单尝试成交 50 单位。可用数量为 70，因此可以成交 50。`(original=100,
   consumed=80)`。
4. 一条增量数据将卖价 100.00 的数量更新为 120 单位。引擎重置：`(original=120, consumed=0)`。
5. 新订单现在可以针对全新的 120 单位进行成交。

**在 L1 数据上被动挂单（限价单）的成交：**

在 L1 数据下（报价、成交明细、K 线），订单簿每一侧只有一个价格档位。当市场价格穿越一笔被动
（挂单方，MAKER）限价单的价格时，引擎必须决定在耗尽显示的流动性之后，如何处理剩余的订单数量。

| `liquidity_consumption` | 市场穿越被动限价单时的行为                                                    |
| ------------------------ | --------------------------------------------------------------------------------- |
| `False`（默认）          | 以限价成交整笔订单。假设市场价格波动意味着存在足够的流动性。                       |
| `True`                   | 仅针对已显示的流动性成交。订单剩余部分保持挂单状态，等待后续成交。                 |

**示例场景**（`liquidity_consumption=True`）：

1. 报价显示卖价 100.10，数量为 50 单位。
2. 你以 100.05 的价格提交一笔 BUY LIMIT（买入限价）订单，数量 1000 单位（被动挂单，低于卖价）。
3. 下一次报价显示卖价 100.00，数量为 30 单位（市场穿越了你的限价）。
4. 订单针对已显示的流动性成交 30 单位。剩余 970 单位仍处于挂单状态。
5. 下一次报价显示卖价 99.95，数量为 200 单位。
6. 订单再成交 200 单位。剩余 770 单位仍处于挂单状态。
7. 每当有新流动性在被穿越的价格档位出现时，成交会持续进行。

这种行为提供了保守的成交模拟：你的订单只会针对数据中实际观察到的流动性成交，而不是从价格波动中
推断出流动性。

**成交明细流动性：**

成交明细（trade tick）可以作为某一价格档位存在可成交流动性的证据。当某笔成交发生在当前订单簿中未
体现的价格时，引擎可以将该成交的数量作为可用流动性（前提是启用了相应的消耗跟踪规则）。

**成交消耗预置（Trade consumption seeding）：**

在使用 L2/L3 订单簿数据、且某笔成交明细触发了订单撮合（例如触发了一笔挂单的止损单）时，该笔成交本身
已经消耗了订单簿中的流动性。在为被触发的订单模拟成交之前，引擎会用该笔成交所消耗的数量预先填充消耗
映射表。这可以防止被触发的订单再次针对触发该次成交时已经消耗掉的流动性成交。对于 L1 订单簿，此项
预置会被跳过，因为该成交明细已经直接更新了唯一的最优挂单档位。

举例来说，若订单簿最优卖价处有 10 单位，且一笔数量为 8 的 BUY 成交触发了一笔数量为 5 的止损市价
BUY 单，则该止损单在最优卖价处只能看到 2 单位可用（10 - 8），其余 3 单位必须在下一价格档位成交。
若没有这一预置机制，该止损单会被错误地在最优卖价处全部成交 5 单位。

引擎使用时间戳保护机制以避免重复计数：如果订单簿最近一次更新的时间戳（`ts_last`）晚于该笔成交的
事件时间（`ts_event`），则会跳过预置流程。这适用于像 Binance 这样深度增量数据先于对应成交明细
到达的交易所——此时订单簿已经反映了被消耗的流动性，额外的预置反而会对成交造成过度惩罚。

:::note
成交模型可以增加更为复杂的执行动态，包括：

- 基于订单规模的可变滑点。
- 更复杂的队列位置建模。

:::

### 已知局限性

**档位内部无队列位置：** 消耗跟踪只能确定某一档位*还剩多少*流动性，而无法建模你的订单相对于其他
参与者*位于队列中的位置*。请使用 `prob_fill_on_limit` 以概率方式模拟队列位置。

**由成交驱动的成交是机会性的：** 当成交明细表明某一未在订单簿中出现的价格存在流动性时，引擎会将其
作为成交依据。然而，这代表的是当时短暂存在的流动性，未必能反映持续可用的流动性。

## 精度要求与不变量

撮合引擎在整个成交流程中强制执行严格的精度不变量（invariants），以确保数据完整性。所有价格和数量都
必须符合金融工具所配置的精度（`price_precision` 和 `size_precision`）。不匹配会立即抛出
`RuntimeError`，以防止成交数量被静默损坏。

| 数据/操作      | 字段                            | 所需精度                     | 校验位置                     |
| --------------- | -------------------------------- | ------------------------------ | ------------------------------ |
| `QuoteTick`     | `bid_price`、`ask_price`         | `instrument.price_precision`   | `process_quote_tick`           |
| `QuoteTick`     | `bid_size`、`ask_size`           | `instrument.size_precision`    | `process_quote_tick`           |
| `TradeTick`     | `price`                          | `instrument.price_precision`   | `process_trade_tick`           |
| `TradeTick`     | `size`                           | `instrument.size_precision`    | `process_trade_tick`           |
| `Bar`           | `open`、`high`、`low`、`close`   | `instrument.price_precision`   | `process_bar`                  |
| `Bar`           | `volume`（基础货币单位）         | `instrument.size_precision`    | `process_bar`                  |
| `Order`         | `quantity`                       | `instrument.size_precision`    | `process_order`                |
| `Order`         | `price`                          | `instrument.price_precision`   | `process_order`                |
| `Order`         | `trigger_price`                  | `instrument.price_precision`   | `process_order`                |
| `Order`         | `activation_price`\*             | `instrument.price_precision`   | `process_order`                |
| Order update    | `quantity`                       | `instrument.size_precision`    | `update_order`                 |
| Order update    | `price`、`trigger_price`         | `instrument.price_precision`   | `update_order`                 |
| Fill            | `fill_qty`                       | `instrument.size_precision`    | `apply_fills`、`fill_order`    |
| Fill            | `fill_px`                        | `instrument.price_precision`   | `apply_fills`                  |

\*`activation_price` 在订单提交后不可修改。

:::warning
`Bar.volume` 必须以**基础货币单位（base currency units）**表示。部分数据提供商报告的是以计价货币
（quote-currency）计算的成交量；请在加载前转换为基础货币单位（除以价格，或使用提供商特定的字段）。
:::

:::tip
如果遇到精度不匹配错误，请将你的数据与金融工具对齐：

```python
# 将价格/数量对齐到金融工具的精度
price = instrument.make_price(raw_price)
qty = instrument.make_qty(raw_qty)
```

同时请确认以下几点：

1. 金融工具的定义与你的数据源的精度相匹配。
2. 数据在加载过程中未被意外地四舍五入或截断。
3. 自定义数据加载器保留了原始的精度元数据。

:::
