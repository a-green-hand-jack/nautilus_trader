# 策略（Strategies）

策略继承自 `Strategy` 类，并实现其逻辑所需要的方法。

**能力**：

- 全部 `Actor` 能力。
- 订单管理。

**与 Actor 的关系**：
`Strategy` 类继承自 `Actor`，这意味着策略可以访问全部 actor 功能，
外加订单管理能力。

:::tip
我们建议在深入策略开发之前先阅读一遍[Actor](actors.md)指南。
:::

策略可以被添加到任意[环境上下文](architecture.md#environment-contexts)中的
Nautilus 系统里，系统一启动，策略就会根据其逻辑开始发送命令和接收事件。

利用数据摄取、事件处理和订单管理这些构建模块（下文讨论），
你可以构建任意类型的策略，包括方向型、动量型、再平衡型、配对交易、
做市等等。

有关所有可用方法，请参阅 [`Strategy` API 参考文档](/docs/python-api-latest/trading.html)。

一个 Nautilus 交易策略主要由两部分组成：

- 策略实现本身，通过继承 `Strategy` 类来定义。
- *可选的*策略配置，通过继承 `StrategyConfig` 类来定义。

:::tip
一旦定义了一个策略，相同的源代码就可以同时用于回测和实盘交易。
:::

策略的主要能力包括：

- 历史数据请求。
- 实时数据流订阅。
- 设置时间提醒或定时器。
- 缓存访问。
- 投资组合访问。
- 创建和管理订单及仓位。

:::info Rust 实现
Rust 策略的编写者需要实现他们所需的 `DataActor` 回调，并使用
`nautilus_strategy!` 生成 `Strategy` 实现，然后在 `self` 上调用诸如
`clock()`、`cache()`、`order()` 和 `portfolio()` 之类的外观方法。
`DataActorNative` 是仅限原生代码对运行时接线和 actor 核心状态的访问；
`StrategyNative` 暴露了策略借用的状态，例如订单工厂、订单管理器和
投资组合访问。仅在同一二进制文件内的性能敏感路径或内部运行时接线场景中
导入它们。
:::

## 策略实现

一个交易策略继承自 `Strategy`，因此你必须定义一个构造函数。
至少要初始化基类：

```python
from nautilus_trader.trading.strategy import Strategy

class MyStrategy(Strategy):
    def __init__(self) -> None:
        super().__init__()  # <-- the superclass must be called to initialize the strategy
```

从这里开始，你可以根据需要实现处理器，以便根据状态转换和事件执行相应的操作。

:::warning
不要在 `__init__` 构造函数中（即注册之前）调用诸如 `clock` 和 `logger`
之类的组件。这是因为系统的时钟和日志子系统此时尚未初始化。
:::

### 处理器

处理器是 `Strategy` 类上的方法，根据事件或状态变化执行相应操作。
这些方法使用 `on_*` 前缀。根据你的策略需要，实现其中的任意或全部方法。

针对类似的事件类型存在多个处理器，以便让你控制处理粒度。
你可以用一个专门的处理器响应某个特定事件，也可以用一个通用处理器处理
一系列相关事件（使用类似 switch 语句的逻辑）。系统会按从最具体到最一般
的顺序依次调用这些处理器。

#### 状态性操作

生命周期状态变化会触发以下这些处理器。建议：

- 使用 `on_start` 方法初始化你的策略（例如获取金融工具、订阅数据）。
- 使用 `on_stop` 方法执行清理任务（例如撤销未平仓订单、平掉未平仓仓位、取消数据订阅）。

```python
def on_start(self) -> None:
def on_stop(self) -> None:
def on_resume(self) -> None:
def on_reset(self) -> None:
def on_dispose(self) -> None:
def on_degrade(self) -> None:
def on_fault(self) -> None:
def on_save(self) -> dict[str, bytes]:  # Returns user-defined dictionary of state to be saved
def on_load(self, state: dict[str, bytes]) -> None:
```

#### 数据处理

这些处理器接收数据更新，包括内置的市场数据和用户自定义的数据。

```python
from nautilus_trader.core import Data
from nautilus_trader.model import OrderBook
from nautilus_trader.model import Bar
from nautilus_trader.model import QuoteTick
from nautilus_trader.model import TradeTick
from nautilus_trader.model import OrderBookDeltas
from nautilus_trader.model import InstrumentClose
from nautilus_trader.model import InstrumentStatus
from nautilus_trader.model import OptionChainSlice
from nautilus_trader.model import OptionGreeks
from nautilus_trader.model.instruments import Instrument

def on_order_book_deltas(self, deltas: OrderBookDeltas) -> None:
def on_order_book(self, order_book: OrderBook) -> None:
def on_quote_tick(self, tick: QuoteTick) -> None:
def on_trade_tick(self, tick: TradeTick) -> None:
def on_bar(self, bar: Bar) -> None:
def on_instrument(self, instrument: Instrument) -> None:
def on_instrument_status(self, data: InstrumentStatus) -> None:
def on_instrument_close(self, data: InstrumentClose) -> None:
def on_option_greeks(self, greeks: OptionGreeks) -> None:
def on_option_chain(self, chain: OptionChainSlice) -> None:
def on_historical_data(self, data: Data) -> None:
def on_data(self, data: Data) -> None:  # Custom data passed to this handler
def on_signal(self, signal: Data) -> None:  # Custom signals passed to this handler
```

#### 订单管理

这些处理器接收与订单相关的事件。
`OrderEvent` 类型的消息会按以下顺序传递给处理器：

1. 特定处理器（例如 `on_order_accepted`、`on_order_rejected` 等）
2. `on_order_event(...)`
3. `on_event(...)`

```python
from nautilus_trader.model.events import OrderAccepted
from nautilus_trader.model.events import OrderCanceled
from nautilus_trader.model.events import OrderCancelRejected
from nautilus_trader.model.events import OrderDenied
from nautilus_trader.model.events import OrderEmulated
from nautilus_trader.model.events import OrderEvent
from nautilus_trader.model.events import OrderExpired
from nautilus_trader.model.events import OrderFilled
from nautilus_trader.model.events import OrderInitialized
from nautilus_trader.model.events import OrderModifyRejected
from nautilus_trader.model.events import OrderPendingCancel
from nautilus_trader.model.events import OrderPendingUpdate
from nautilus_trader.model.events import OrderRejected
from nautilus_trader.model.events import OrderReleased
from nautilus_trader.model.events import OrderSubmitted
from nautilus_trader.model.events import OrderTriggered
from nautilus_trader.model.events import OrderUpdated

def on_order_initialized(self, event: OrderInitialized) -> None:
def on_order_denied(self, event: OrderDenied) -> None:
def on_order_emulated(self, event: OrderEmulated) -> None:
def on_order_released(self, event: OrderReleased) -> None:
def on_order_submitted(self, event: OrderSubmitted) -> None:
def on_order_rejected(self, event: OrderRejected) -> None:
def on_order_accepted(self, event: OrderAccepted) -> None:
def on_order_canceled(self, event: OrderCanceled) -> None:
def on_order_expired(self, event: OrderExpired) -> None:
def on_order_triggered(self, event: OrderTriggered) -> None:
def on_order_pending_update(self, event: OrderPendingUpdate) -> None:
def on_order_pending_cancel(self, event: OrderPendingCancel) -> None:
def on_order_modify_rejected(self, event: OrderModifyRejected) -> None:
def on_order_cancel_rejected(self, event: OrderCancelRejected) -> None:
def on_order_updated(self, event: OrderUpdated) -> None:
def on_order_filled(self, event: OrderFilled) -> None:
def on_order_event(self, event: OrderEvent) -> None:  # All order event messages are eventually passed to this handler
```

#### 仓位管理

这些处理器接收与仓位相关的事件。
`PositionEvent` 类型的消息会按以下顺序传递给处理器：

1. 特定处理器（例如 `on_position_opened`、`on_position_changed` 等）
2. `on_position_event(...)`
3. `on_event(...)`

```python
from nautilus_trader.model.events import PositionChanged
from nautilus_trader.model.events import PositionClosed
from nautilus_trader.model.events import PositionEvent
from nautilus_trader.model.events import PositionOpened

def on_position_opened(self, event: PositionOpened) -> None:
def on_position_changed(self, event: PositionChanged) -> None:
def on_position_closed(self, event: PositionClosed) -> None:
def on_position_event(self, event: PositionEvent) -> None:  # All position event messages are eventually passed to this handler
```

#### 通用事件处理

该处理器最终会接收到达该策略的所有事件消息，包括那些没有其他特定处理器的消息。

```python
from nautilus_trader.core.message import Event

def on_event(self, event: Event) -> None:
```

#### 处理器示例

以下示例展示了一个典型的 `on_start` 处理器方法实现（取自示例
EMA 交叉策略）。这里我们可以看到：

- 指标被注册以接收 K 线更新。
- 请求历史数据（用于给指标预热）。
- 订阅实时数据。

在实盘交易中，缓存检查很重要。直接订阅假定该金融工具已经由金融工具
提供方配置或先前的金融工具请求加载。

```python
def on_start(self) -> None:
    """
    Actions to be performed on strategy start.
    """
    self.instrument = self.cache.instrument(self.instrument_id)
    if self.instrument is None:
        self.log.error(f"Could not find instrument for {self.instrument_id}")
        self.stop()  # Transitions strategy to STOPPED state
        return

    # Register the indicators for updating
    self.register_indicator_for_bars(self.bar_type, self.fast_ema)
    self.register_indicator_for_bars(self.bar_type, self.slow_ema)

    # Get historical data and subscribe to live data
    self.request_bars(
        self.bar_type,
        callback=lambda _: self.subscribe_bars(self.bar_type),
    )
    self.subscribe_quote_ticks(self.instrument_id)
```

实时 K 线是通过 `request_bars()` 的 `callback` 订阅的，因此该数据流仅在
历史数据加载完成后才开始；关于这一点在 `validate_data_sequence=True` 下
为何重要，请参阅[使用 K 线：请求 vs. 订阅](data/index.md#working-with-bars-request-vs-subscribe)。

### 时钟和定时器

策略可以访问一个 `Clock`，该时钟提供了多种方法用于创建不同的时间戳，
以及设置触发 `TimeEvent` 的时间提醒或定时器。

有关所有可用方法，请参阅 [`Clock` API 参考文档](/docs/python-api-latest/common.html)。

#### 当前时间戳

虽然有多种方式可以获取当前时间戳，但这里以两种常用方法作为示例：

获取当前 UTC 时间戳，作为带时区信息的 `pd.Timestamp`：

```python
import pandas as pd


now: pd.Timestamp = self.clock.utc_now()
```

获取当前 UTC 时间戳，作为自 UNIX 纪元以来的纳秒数：

```python
unix_nanos: int = self.clock.timestamp_ns()
```

#### 时间提醒

可以设置时间提醒，在指定的提醒时间将一个 `TimeEvent` 分发给
`on_event` 处理器。在实盘环境中，这可能会有几微秒的轻微延迟。

以下示例设置了一个从当前时间起一分钟后触发的时间提醒：

```python
import pandas as pd

# Fire a TimeEvent one minute from now
self.clock.set_time_alert(
    name="MyTimeAlert1",
    alert_time=self.clock.utc_now() + pd.Timedelta(minutes=1),
)
```

#### 定时器

可以设置连续的定时器，按固定间隔生成 `TimeEvent`，直到该定时器过期或
被取消。

以下示例设置了一个每分钟触发一次的定时器，立即开始：

```python
import pandas as pd

# Fire a TimeEvent every minute
self.clock.set_timer(
    name="MyTimer1",
    interval=pd.Timedelta(minutes=1),
)
```

### 缓存访问

交易者的中央 `Cache` 存储数据和执行对象（订单、仓位等）。
提供了许多带过滤功能的方法。以下是一些基础使用场景。

#### 获取数据

以下示例从缓存中获取数据（假设已经分配了某个金融工具 ID 属性）。
如果请求的数据不可用，这些方法返回 `None`。

```python
last_quote = self.cache.quote_tick(self.instrument_id)
last_trade = self.cache.trade_tick(self.instrument_id)
last_bar = self.cache.bar(bar_type)
```

#### 获取执行对象

以下示例展示了如何从缓存中获取单个订单和仓位对象：

```python
order = self.cache.order(client_order_id)
position = self.cache.position(position_id)
```

有关所有可用方法，请参阅 [`Cache` API 参考文档](/docs/python-api-latest/cache.html)。

### 投资组合访问

交易者的中央 `Portfolio` 提供账户和仓位信息。
以下是可用方法的一个大致概览。

#### 账户和仓位信息

```python
import decimal

from nautilus_trader.accounting.accounts.base import Account
from nautilus_trader.model import Venue
from nautilus_trader.model import Currency
from nautilus_trader.model import Money
from nautilus_trader.model import InstrumentId

def account(self, venue: Venue) -> Account

def balances_locked(self, venue: Venue) -> dict[Currency, Money]
def margins_init(self, venue: Venue) -> dict[Currency, Money]
def margins_maint(self, venue: Venue) -> dict[Currency, Money]
def unrealized_pnls(self, venue: Venue) -> dict[Currency, Money]
def realized_pnls(self, venue: Venue) -> dict[Currency, Money]
def net_exposures(self, venue: Venue) -> dict[Currency, Money]

def unrealized_pnl(self, instrument_id: InstrumentId) -> Money
def realized_pnl(self, instrument_id: InstrumentId) -> Money
def net_exposure(self, instrument_id: InstrumentId) -> Money
def net_position(self, instrument_id: InstrumentId) -> decimal.Decimal

def is_net_long(self, instrument_id: InstrumentId) -> bool
def is_net_short(self, instrument_id: InstrumentId) -> bool
def is_flat(self, instrument_id: InstrumentId) -> bool
def is_completely_flat(self) -> bool
```

有关所有可用方法，请参阅 [`Portfolio` API 参考文档](/docs/python-api-latest/portfolio.html)。

#### 报告和分析

`Portfolio` 还暴露了一个 `PortfolioAnalyzer`，它接受灵活数量的数据
（以适应不同的回溯窗口）。该分析器跟踪并生成绩效指标和统计数据。

请参阅 [`PortfolioAnalyzer` API 参考文档](/docs/python-api-latest/analysis.html)
和[投资组合统计](portfolio.md#portfolio-statistics)指南。

### 交易命令

以下交易命令可用于订单管理。
有关系统内完整的流转过程，另请参阅[执行（Execution）](../concepts/execution.md)指南。

#### 提交订单

`Strategy` 基类为每个策略都便捷地提供了一个 `OrderFactory`，
减少了创建不同 `Order` 对象所需的样板代码量（不过如果交易者更喜欢，
仍然可以直接使用 `Order.__init__(...)` 构造函数初始化这些对象）。

一条 `SubmitOrder` 或 `SubmitOrderList` 命令会流向哪个组件进行执行，
取决于以下几点：

- 如果指定了 `emulation_trigger`，该命令会*首先*被发送到 `OrderEmulator`。
- 如果指定了 `exec_algorithm_id`（且没有 `emulation_trigger`），该命令会
  *首先*被发送到相应的 `ExecAlgorithm`。
- 否则，该命令会*首先*被发送到 `RiskEngine`。

以下示例提交了一个用于模拟的 `LIMIT` BUY 订单（参见
[模拟订单（Emulated Orders）](orders/emulated.md)）：

```python
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TriggerType
from nautilus_trader.model.orders import LimitOrder


def buy(self) -> None:
    """
    Users simple buy method (example).
    """
    order: LimitOrder = self.order_factory.limit(
        instrument_id=self.instrument_id,
        order_side=OrderSide.BUY,
        quantity=self.instrument.make_qty(self.trade_size),
        price=self.instrument.make_price(5000.00),
        emulation_trigger=TriggerType.LAST_PRICE,
    )

    self.submit_order(order)
```

:::info
你可以同时指定订单模拟和一个执行算法。在这种情况下，该订单会先被发送到
`OrderEmulator`，一旦被释放，就会路由到 `ExecAlgorithm`。
:::

以下示例向一个 TWAP 执行算法提交了一个 `MARKET` BUY 订单：

```python
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model import ExecAlgorithmId


def buy(self) -> None:
    """
    Users simple buy method (example).
    """
    order: MarketOrder = self.order_factory.market(
        instrument_id=self.instrument_id,
        order_side=OrderSide.BUY,
        quantity=self.instrument.make_qty(self.trade_size),
        time_in_force=TimeInForce.FOK,
        exec_algorithm_id=ExecAlgorithmId("TWAP"),
        exec_algorithm_params={"horizon_secs": 20, "interval_secs": 2.5},
    )

    self.submit_order(order)
```

#### 撤销订单

订单可以被单独撤销、批量撤销，或撤销某个金融工具的全部订单
（可选按方向过滤）。

如果该订单已经*关闭*或已经处于待撤销状态，则会记录一条警告日志。

如果该订单当前处于*未平仓*状态，其状态将变为 `PENDING_CANCEL`。

一条 `CancelOrder`、`CancelAllOrders` 或 `BatchCancelOrders` 命令会流向
哪个组件进行执行，取决于以下几点：

- 如果该订单当前处于模拟状态，该命令会*首先*被发送到 `OrderEmulator`。
- 如果指定了 `exec_algorithm_id`（且没有 `emulation_trigger`），并且
  该订单在本地系统内仍处于活跃状态，该命令会*首先*被发送到相应的
  `ExecAlgorithm`。
- 否则，该订单会*首先*被发送到 `ExecutionEngine`。

:::info
在该命令离开策略之后，任何受管理的 GTD 定时器也会被取消。
:::

以下展示了如何撤销单个订单：

```python
self.cancel_order(order)
```

以下展示了如何批量撤销订单：

```python
from nautilus_trader.model.orders import Order


my_order_list: list[Order] = [order1, order2, order3]
self.cancel_orders(my_order_list)
```

以下展示了如何撤销所有订单：

```python
self.cancel_all_orders()
```

#### 修改订单

当处于模拟状态或在交易场所上处于*未平仓*状态（如果支持）时，
订单可以被单独修改。

如果该订单已经*关闭*或已经处于待撤销状态，则会记录一条警告日志。
如果该订单当前处于*未平仓*状态，其状态将变为 `PENDING_UPDATE`。

:::warning
必须至少有一个值与原始订单不同，该命令才有效。
:::

一条 `ModifyOrder` 命令会流向哪个组件进行执行，取决于以下几点：

- 如果该订单当前处于模拟状态，该命令会*首先*被发送到 `OrderEmulator`。
- 否则，该订单会*首先*被发送到 `RiskEngine`。

:::info
一旦某个订单处于执行算法的控制之下，策略就无法直接修改它（只能撤销）。
:::

以下展示了如何修改一个当前在交易场所*未平仓*的 `LIMIT` BUY 订单的数量：

```python
from nautilus_trader.model import Quantity


new_quantity: Quantity = Quantity.from_int(5)
self.modify_order(order, new_quantity)
```

:::info
价格和触发价格也可以被修改（在模拟状态下，或交易场所支持的情况下）。
:::

#### 市场退出

`market_exit()` 方法提供了一种优雅的方式，用于平掉某个策略的所有仓位
并撤销所有订单。退出完成后策略仍会继续运行，如果需要，允许你之后
重新进入仓位。

```python
self.market_exit()
```

市场退出流程：

1. 撤销该策略的所有未平仓和在途订单。
2. 用市价单平掉所有未平仓仓位。
3. 周期性检查（按 `market_exit_interval_ms`），直到所有订单都已解决且
   仓位已平仓。
4. 一旦持平，或达到 `market_exit_max_attempts` 之后，调用
   `post_market_exit()`。

有两个钩子可用于自定义逻辑：

- `on_market_exit()` - 在退出流程开始时调用。
- `post_market_exit()` - 在退出流程完成时调用。

```python
class MyStrategy(Strategy):
    def on_market_exit(self) -> None:
        self.log.info("Beginning market exit...")

    def post_market_exit(self) -> None:
        self.log.info("Market exit complete")
```

在市场退出期间，非仅减仓订单会被自动拒绝。对于订单列表，如果列表中
任何一个订单是非仅减仓的，整个列表都会被拒绝，以保留列表语义
（例如具有相互依赖关系的括号订单）。

要检查是否正在进行退出（例如为了跳过下单逻辑），可以使用
`is_exiting()`：

```python
def on_quote_tick(self, tick: QuoteTick) -> None:
    if self.is_exiting():
        return  # Skip order logic during exit
    # ... normal order logic
```

要在策略停止时自动执行市场退出，请设置 `manage_stop=True`：

```python
config = StrategyConfig(manage_stop=True)
```

使用该选项时，调用 `stop()` 会先执行一次市场退出，然后在持平之后
再停止该策略。

`StrategyConfig` 中的配置选项：

- `manage_stop`（默认：False）- 如果为 True，`stop()` 会在停止之前执行一次市场退出。
- `market_exit_interval_ms`（默认：100）- 退出完成检查之间的间隔。
- `market_exit_max_attempts`（默认：100）- 在完成退出之前的最大检查次数。
- `market_exit_time_in_force`（默认：None/GTC）- 用于平仓市价单的有效期。
- `market_exit_reduce_only`（默认：True）- 平仓市价单是否应为仅减仓。

## 策略配置

一个独立的配置类为策略在何处以及如何被实例化提供了充分的灵活性。
配置可以通过网络序列化，从而支持分布式回测和远程实盘交易。

这是可选启用的。你可以跳过配置，直接向你的策略构造函数传递参数。
如果你希望支持分布式回测或远程实盘交易，请定义一个配置。

以下是一个配置示例：

```python
from decimal import Decimal
from nautilus_trader.config import StrategyConfig
from nautilus_trader.model import Bar, BarType
from nautilus_trader.model import InstrumentId
from nautilus_trader.trading.strategy import Strategy


# Configuration definition
class MyStrategyConfig(StrategyConfig):
    instrument_id: InstrumentId   # example value: "ETHUSDT-PERP.BINANCE"
    bar_type: BarType             # example value: "ETHUSDT-PERP.BINANCE-15-MINUTE[LAST]-EXTERNAL"
    fast_ema_period: int = 10
    slow_ema_period: int = 20
    trade_size: Decimal
    order_id_tag: str


# Strategy definition
class MyStrategy(Strategy):
    def __init__(self, config: MyStrategyConfig) -> None:
        # Always initialize the parent Strategy class
        # After this, configuration is stored and available via `self.config`
        super().__init__(config)

        # Custom state variables
        self.time_started = None
        self.count_of_processed_bars: int = 0

    def on_start(self) -> None:
        self.time_started = self.clock.utc_now()    # Remember time, when strategy started
        self.subscribe_bars(self.config.bar_type)   # See how configuration data are exposed via `self.config`

    def on_bar(self, bar: Bar):
        self.count_of_processed_bars += 1           # Update count of processed bars


# Instantiate configuration with specific values. By setting:
#   - InstrumentId - we parameterize the instrument the strategy will trade.
#   - BarType - we parameterize bar-data, that strategy will trade.
config = MyStrategyConfig(
    instrument_id=InstrumentId.from_str("ETHUSDT-PERP.BINANCE"),
    bar_type=BarType.from_str("ETHUSDT-PERP.BINANCE-15-MINUTE[LAST]-EXTERNAL"),
    trade_size=Decimal(1),
    order_id_tag="001",
)

# Pass configuration to our trading strategy.
strategy = MyStrategy(config=config)
```

通过 `self.config` 访问配置值。
这清晰地划分了以下两者：

- 配置数据（通过 `self.config` 访问）：
  - 包含定义策略工作方式的初始设置。
  - 例如：`self.config.trade_size`、`self.config.instrument_id`

- 策略状态变量（作为直接属性）：
  - 跟踪策略的任何自定义状态。
  - 例如：`self.time_started`、`self.count_of_processed_bars`

这种划分使代码更易于理解和维护。

:::note
虽然通常定义一个只交易单一金融工具的策略是合理的，但单个策略能够
处理的金融工具数量仅受限于机器资源。
:::

### 受管理的 GTD 过期

策略可以为有效期为 GTD（*Good 'till Date*，有效期至指定日期）的订单
管理过期时间。如果交易所/经纪商不支持这种有效期选项，或者出于任何原因
你希望策略来管理此事，这个功能会很有用。

要使用此选项，请向你的 `StrategyConfig` 传入 `manage_gtd_expiry=True`。
当一个有效期为 GTD 的订单被提交时，策略会自动启动一个内部时间提醒。
一旦到达该内部 GTD 时间提醒，该订单就会被撤销（如果尚未*关闭*）。

一些交易场所（例如 Binance Futures）支持 GTD 有效期，因此在使用
`managed_gtd_expiry` 时，为避免冲突，你应该将执行客户端配置中的
`use_gtd` 设置为 `False`。

### 多个策略

如果你打算使用不同的配置（例如交易不同的金融工具）运行同一个策略的
多个实例，则每个实例都需要一个唯一的策略 ID 和订单 ID 标签。

如果未提供 `strategy_id`，平台会根据策略类名和一个订单 ID 标签构建
策略 ID。可以通过 `order_id_tag` 提供该标签；否则注册过程会分配下一个
数字标签，从 `000` 开始。例如，上面的配置会得到策略 ID
`MyStrategy-001`。

如果同时提供了 `strategy_id` 和 `order_id_tag`，Rust 会将该标签追加到
运行时策略 ID 上，除非该 ID 已经以该标签结尾。例如，
`strategy_id=MyStrategy-PRIMARY` 加上 `order_id_tag=ABC` 会变成
`MyStrategy-PRIMARY-ABC`。
如果省略了 `order_id_tag`，Rust 会使用 `strategy_id` 中按连字符分隔的
最后一部分作为订单 ID 标签。

:::note
该平台内置了安全措施：如果两个策略共享重复的策略 ID，在注册过程中会
引发一个 `RuntimeError`，指出该策略 ID 已经被注册。
:::

之所以这样设计，是因为系统必须能够识别各种命令和事件属于哪个策略。
订单 ID 标签还能确保同一个交易者生成的客户端订单 ID 在各策略之间保持
唯一。

:::info Rust 实现
Rust 将 `StrategyConfig` 视为不可变的构造输入。运行时的 `StrategyId`
携带订单 ID 标签，与 Python/Cython 的行为一致。这使 actor 注册、
客户端订单 ID 生成、订单列表 ID 生成和仓位 ID 生成都通过
`strategy_id.get_tag()` 保持一致。

如果省略了 `strategy_id`，`order_id_tag` 会覆盖生成的后缀，
例如 `MyStrategy-ABC`。
:::

更多详情请参阅 [`StrategyId` API 参考文档](/docs/python-api-latest/model/identifiers.html)。

## 相关指南

- [Actor](actors.md) - 策略所扩展的基类。
- [事件（Events）](events/) - 事件类型和处理器分发。
- [订单（Orders）](orders/) - 从策略进行订单类型管理。
- [回测（Backtesting）](backtesting/) - 使用历史数据测试策略。
