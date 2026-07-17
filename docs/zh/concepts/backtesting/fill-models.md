# 成交模型

成交模型（Fill Model）用于模拟回测过程中订单执行的动态特性。它们解决了一个根本性的挑战：*即使拥有
完美的历史市场数据，我们也无法完全模拟订单在实时环境中与其他市场参与者的交互方式*。

基础的 `FillModel` 提供了用于模拟队列位置（queue position）和滑点（slippage）的概率参数。子类可以
重写 `get_orderbook_for_fill_simulation()` 方法，以生成合成订单簿，实现更复杂的流动性建模。

## 滑点与价差处理

在使用不同类型的数据进行回测时，Nautilus 针对滑点和价差（spread）模拟实现了特定的处理逻辑：

对于 L2（逐档，market-by-price）或 L3（逐单，market-by-order）数据，滑点通过以下方式实现高精度模拟：

- 按实际订单簿档位成交订单。
- 依次在每个价格档位撮合可用数量。
- 保持真实的订单簿深度冲击（每笔成交都会体现）。

对于 L1 类型的数据（例如 L1 订单簿、成交明细、报价、K 线），滑点通过 `FillModel` 处理：

**逐笔滑点**（`prob_slippage`）：

- 在使用带有已配置 `FillModel` 的 L1 订单簿时，适用于每一笔成交。
- 影响所有订单类型（市价单、限价单、止损单等）。
- 触发时，会使成交价格朝不利于订单方向的一侧移动一个最小变动价位（tick）。
- 示例：当 `prob_slippage=0.5` 时，一笔 BUY（买入）订单有 50% 的概率以高于最优卖价一个 tick 的
  价格成交。

:::note
在使用 K 线数据进行回测时，请注意价格信息粒度的降低会影响滑点机制。为获得最真实的回测结果，建议在
可行的情况下使用更高粒度的数据源，例如 L2 或 L3 订单簿数据。
:::

## 模拟行为如何随数据类型变化

`FillModel` 的行为会根据所使用的订单簿类型而有所不同：

**L2/L3 订单簿数据**

在拥有完整订单簿深度的情况下，`FillModel` 只专注于通过 `prob_fill_on_limit` 模拟限价单的队列位置。
订单簿本身会根据每个价格档位的可用流动性，自然地处理滑点。

- `prob_fill_on_limit` 生效——模拟队列位置。
- `prob_slippage` 不使用——真实的订单簿深度决定价格冲击。

:::warning
在回测期间，历史订单簿是不可变的。成交后订单簿深度**不会**被扣减。默认情况下
（`liquidity_consumption=False`），同一份流动性在一次迭代中可以被重复消耗多次。启用
`liquidity_consumption=True` 以按价格档位跟踪已消耗的流动性。当该档位到达新数据时，消耗记录会
重置。详情参见 [订单簿不可变性](fill-prices-and-matching.md#order-book-immutability)。
:::

**L1 订单簿数据**

在只有最优买卖价可用的情况下，`FillModel` 提供了额外的模拟：

- `prob_fill_on_limit` 生效——模拟队列位置。
- `prob_slippage` 生效——由于缺乏真实深度信息，模拟基础的价格冲击。

**K 线/报价/成交明细数据**

在使用粒度更低的数据时，行为与 L1 相同：

- `prob_fill_on_limit` 生效——模拟队列位置。
- `prob_slippage` 生效——模拟基础的价格冲击。

## 重要注意事项

- **部分成交：** 使用 L2/L3 数据时，成交受限于每个价格档位的可用流动性。使用 L1 数据时，整笔订单
  数量会在唯一可用的档位上全部成交。
- **消耗跟踪：** 有关防止重复成交的详情，请参见
  [订单簿不可变性](fill-prices-and-matching.md#order-book-immutability)。

## 可用的成交模型

| 模型                          | 描述                                                     | 使用场景                                     |
| ----------------------------- | -------------------------------------------------------- | --------------------------------------------- |
| `FillModel`                   | 具有概率性成交/滑点参数的基础模型。                        | 简单的队列位置与滑点模拟。                     |
| `BestPriceFillModel`          | 以最优价格成交，流动性不限。                                | 以乐观方式测试基本策略逻辑。                   |
| `OneTickSlippageFillModel`    | 强制所有订单产生恰好一个最小变动价位的滑点。                | 保守的滑点测试。                               |
| `TwoTierFillModel`            | 10 手以最优价格成交，其余部分以差一个 tick 的价格成交。    | 基础的市场深度模拟。                           |
| `ThreeTierFillModel`          | 按 50/30/20 的手数分布在三个价格档位成交。                  | 更真实的深度模拟。                             |
| `ProbabilisticFillModel`      | 50% 概率以最优价格成交，50% 概率产生一个 tick 的滑点。      | 随机化的执行质量。                             |
| `SizeAwareFillModel`          | 根据订单规模不同（≤10 手 vs >10 手）采用不同的执行方式。   | 与规模相关的市场冲击。                         |
| `LimitOrderPartialFillModel`  | 每次触及价位时，最多成交 5 手。                             | 通过部分成交模拟队列位置。                     |
| `MarketHoursFillModel`        | 在低流动性时段扩大价差。                                    | 会话感知（session-aware）的执行。              |
| `VolumeSensitiveFillModel`    | 流动性基于近期成交量。                                      | 与成交量相适应的深度模拟。                     |
| `CompetitionAwareFillModel`   | 仅有部分可见流动性可供成交。                                | 多参与者竞争场景。                             |

## 配置成交模型

**使用带概率参数的基础 FillModel：**

```python
from nautilus_trader.backtest.config import BacktestVenueConfig
from nautilus_trader.backtest.config import ImportableFillModelConfig

venue_config = BacktestVenueConfig(
    name="SIM",
    oms_type="NETTING",
    account_type="CASH",
    starting_balances=["100_000 USD"],
    fill_model=ImportableFillModelConfig(
        fill_model_path="nautilus_trader.backtest.models:FillModel",
        config_path="nautilus_trader.backtest.config:FillModelConfig",
        config={
            "prob_fill_on_limit": 0.2,    # 价格匹配时限价单成交的概率
            "prob_slippage": 0.5,         # 产生 1 个 tick 滑点的概率（仅 L1 数据）
            "random_seed": 42,            # 可选：设置以获得可复现的结果
        },
    ),
)
```

**使用订单簿模拟模型：**

```python
from nautilus_trader.backtest.config import BacktestVenueConfig
from nautilus_trader.backtest.config import ImportableFillModelConfig

venue_config = BacktestVenueConfig(
    name="SIM",
    oms_type="NETTING",
    account_type="CASH",
    starting_balances=["100_000 USD"],
    fill_model=ImportableFillModelConfig(
        fill_model_path="nautilus_trader.backtest.models:ThreeTierFillModel",
    ),
)
```

## 概率参数

**prob_fill_on_limit**（默认值：`1.0`）

通过控制限价单在其价格档位被触及（但未被穿越）时成交的概率，来模拟队列位置。

- `0.0`：触及价位时永不成交（位于队列末尾）。
- `0.5`：50% 的概率成交（位于队列中间）。
- `1.0`：触及价位时总是成交（位于队列最前）。

**prob_slippage**（默认值：`0.0`）

模拟每笔成交的价格滑点。仅适用于 L1 类型数据（报价、成交明细、K 线），因为这类数据缺乏真实的深度
信息。作为吃单方（taker）执行时，影响所有订单类型。

- `0.0`：无滑点（以最优价格成交）。
- `0.5`：50% 的概率产生一个 tick 的滑点。
- `1.0`：总是产生一个 tick 的滑点。

## 订单簿模拟模型

这些模型重写 `get_orderbook_for_fill_simulation()` 方法，以生成代表预期市场流动性的合成订单簿。
撮合引擎会依据该合成订单簿完成订单成交。

**工作原理：**

1. 在处理一笔成交之前，撮合引擎会调用 `get_orderbook_for_fill_simulation()`。
2. 若该模型返回了合成订单簿，则成交会依据该订单簿的流动性执行。
3. 若该模型返回 `None`，则应用标准的成交逻辑。

:::note
当自定义成交模型提供了模拟订单簿时，`liquidity_consumption`（流动性消耗）跟踪机制**不会**被应用。
自定义成交模型应自行管理其返回的订单簿中的流动性模拟。流动性消耗跟踪仅影响内置的成交逻辑（即当
`get_orderbook_for_fill_simulation()` 返回 `None` 时）。
:::

**示例：ThreeTierFillModel**

该模型创建了一个流动性分布在三个价格档位的订单簿：

- 最优价格处 50 手
- 差一个 tick 的价格处 30 手
- 差两个 tick 的价格处 20 手

一笔 100 手的市价订单将在每个档位分别部分成交，从而体现出真实的价格冲击。

**创建自定义成交模型：**

```python
from nautilus_trader.backtest.models import FillModel
from nautilus_trader.model.book import OrderBook, BookOrder
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.core.rust.model import BookType

class MyCustomFillModel(FillModel):
    def get_orderbook_for_fill_simulation(
        self,
        instrument,
        order,
        best_bid,
        best_ask,
    ):
        book = OrderBook(
            instrument_id=instrument.id,
            book_type=BookType.L2_MBP,
        )

        # 根据你自己的市场模型添加自定义流动性
        # ...

        return book
```
