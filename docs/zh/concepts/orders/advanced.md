# 高级订单

以下指南应结合券商或交易场所（例如 Interactive Brokers）针对这些订单类型、列表/分组以及执行指令提供的具体文档一起阅读。

## 订单列表（Order lists）

多个关联订单的组合，或更大批量的订单，可以通过一个公共的 `order_list_id` 分组为一个列表。此列表中包含的订单彼此之间可能存在也可能不存在关联关系，
这取决于订单本身的构造方式以及它们被路由到的具体交易场所。

列表中的所有订单必须共享同一个交易场所（venue）。订单可以针对该交易场所上不同的金融工具（例如货币对、跨期价差、多腿组合的各个腿）；
目标交易场所是否接受混合工具批次则取决于具体交易场所。列表的 `instrument_id` 取自第一笔订单，作为代表性值；需要按订单获取金融工具的下游消费者会单独解析每笔订单。

混合工具列表的注意事项：

- 交易前逐单检查（价格/数量精度、GTD）使用每笔订单自身的金融工具。
- 累计风险检查（可用余额、最小/最大名义价值、减仓敞口、逐单行情查询）使用列表的代表性金融工具。对于混合列表，这只是单一工具的约束，而非按工具的精确检查。
- 类似 `cache.order_lists(instrument_id=...)` 的缓存查询会按代表性 `instrument_id` 过滤；包含其他金融工具的列表不会匹配对那些其他工具的查询。
- 当提供了 `position_id` 时，执行引擎会拒绝混合工具列表（无论 OMS 如何，一个持仓只属于单一金融工具）。
- 适配器（Adapter）的 `submit_order_list` 实现各不相同。有些会按腿逐单迭代，并针对交易场所 API 解析每笔订单自身的 `instrument_id`；
  另一些仍会围绕列表的代表性 `instrument_id` 构建批量请求，从而可能对非首笔订单造成错误路由。请将混合工具列表视为适配器相关（adapter-specific）的功能；
  在依赖此功能前，请验证目标适配器的具体行为。目前，在用户空间处理多腿路由的回测和自定义策略代码仍是最稳妥的方式。

## 关联类型（Contingency types）

- **OTO（One-Triggers-Other，一触发另一）** – 父订单一旦成交，即自动下达一个或多个子订单。
  - *全触发模式（Full-trigger model）*：子订单**仅在父订单完全成交后**才被释放。此模式常见于大多数零售股票/期权券商（例如 Schwab、Fidelity、TD Ameritrade）以及多数现货加密货币交易场所（Binance、Coinbase）。
  - *部分触发模式（Partial-trigger model）*：子订单**按每次部分成交按比例释放**。此模式用于专业级平台，例如 Interactive Brokers、大多数期货/外汇 OMS 以及 Kraken Pro。

- **OCO（One-Cancels-Other，一取消另一）** – 两个（或多个）关联的存续订单，其中一个成交后会取消其余订单。

- **OUO（One-Updates-Other，一更新另一）** – 两个（或多个）关联的存续订单，其中一个成交后会减少其余订单的未成交数量。

:::info
这些关联类型对应 ContingencyType FIX 标签 <1385> <https://www.onixs.biz/fix-dictionary/5.0.sp2/tagnum_1385.html>。
:::

### One-Triggers-Other（OTO）

一笔 OTO 订单包含两部分：

1. **父订单** – 立即提交至撮合引擎。
2. **子订单** – 在触发条件满足之前一直保持*离簿（off-book）*状态。

#### 触发模式

| 触发模式             | 子订单何时被释放？                                                                                            |
|---------------------|--------------------------------------------------------------------------------------------------------------|
| **全触发（Full trigger）**    | 当父订单的累计成交数量等于其原始数量时（即*完全*成交时）。                                          |
| **部分触发（Partial trigger）** | 父订单每次部分成交后立即触发；子订单数量匹配已成交的数量，并随进一步成交而增加。                    |

:::info
NautilusTrader 默认的回测交易场所对 OTO 订单使用*部分触发模式*。
如需选择使用*全触发模式*，请为交易场所设置 `oto_trigger_mode="FULL"`（例如通过 `BacktestVenueConfig`）。
:::

**在生产环境中处理部分触发：**

如果你的策略需要全触发语义，但交易场所或回测引擎使用的是部分触发模式：

1. 提交不带关联子订单的父订单。
2. 订阅父订单的 `OrderFilled` 事件。
3. 仅在确认父订单完全成交后，才提交子订单（止损、止盈）。
4. 使用 `order.is_closed` 和 `order.filled_qty == order.quantity` 来验证是否完全成交。

> **为什么这一区分很重要**
> *全触发* 存在一个风险窗口：在剩余数量成交之前，任何已部分成交的仓位都处于没有保护性离场单的暴露状态。
> *部分触发* 通过确保每笔已执行的批次立即拥有其关联的止损/止盈单来降低这一风险，但代价是产生更多的订单流量和更新操作。

一笔 OTO 订单可以在交易场所支持的任意资产类型上使用（例如股票入场配合期权对冲、期货入场配合 OCO 括号订单、加密货币现货入场配合止盈/止损）。

| 交易场所 / 适配器 ID                          | 资产类别                  | 子订单触发规则                                | 实用说明                                                          |
|------------------------------------------------|---------------------------|------------------------------------------------|---------------------------------------------------------------------|
| Binance / Binance Futures（`BINANCE`）        | 现货、永续期货             | **部分或全部** – 首次成交即触发。               | OTOCO/止盈止损子订单立即出现；需监控保证金使用情况。                |
| Bybit 现货（`BYBIT`）                         | 现货                       | **全部** – 完全成交后才下达子订单。             | 止盈止损预设仅在限价单完全成交后才激活。                             |
| Bybit 永续合约（`BYBIT`）                     | 永续期货                   | **部分和全部** – 可配置。                       | “部分持仓”模式会根据成交进度设定止盈止损数量。                       |
| Kraken 期货（`KRAKEN`）                       | 期货与永续合约             | **部分和全部** – 自动。                         | 子订单数量匹配每次部分成交。                                        |
| OKX（`OKX`）                                  | 现货、期货、期权           | **全部** – 附加止损单等待成交完成。             | 可单独添加持仓级别的止盈止损。                                      |
| Interactive Brokers（`INTERACTIVE_BROKERS`） | 股票、期权、外汇、期货     | **可配置** – OCA 可按比例分配。                 | `OcaType 2/3` 会减少剩余子订单的数量。                               |
| dYdX v4（`DYDX`）                             | 永续期货（DEX）            | 链上条件（数量精确）。                          | 止盈止损由预言机价格触发；不适用部分成交场景。                       |
| Polymarket（`POLYMARKET`）                    | 预测市场（DEX）            | 不适用。                                        | 高级关联逻辑完全在策略层处理。                                      |
| Betfair（`BETFAIR`）                          | 体育博彩                   | 不适用。                                        | 高级关联逻辑完全在策略层处理。                                      |

### One-Cancels-Other（OCO）

OCO 订单是一组关联订单，其中**任意**一个订单（全部成交*或部分成交*）都会触发对其余订单的尽力取消（best-efforts cancellation）。
两个订单同时存续；一旦其中一个开始成交，交易场所会尝试取消其余订单未成交的部分。

### One-Updates-Other（OUO）

OUO 订单是一组关联订单，其中一个订单的成交会立即*减少*其余订单的未成交数量。
两个订单同时并发存续，每次部分成交都会以尽力方式按比例更新其配对订单的剩余数量。

## 关联订单验证

在处理关联订单（OTO、OCO、OUO）时，请注意以下验证规则和错误场景：

**订单列表要求：**

- 关联订单组中的所有订单必须共享同一个 `order_list_id`。
- 父订单必须先于或与子订单同时提交。
- 子订单通过 `parent_order_id` 引用其父订单。

**修改规则：**

- 父订单在待处理状态下通常可以修改，但修改可能会级联影响子订单。
- 子订单在大多数交易场所可以独立修改，但需检查具体交易场所的行为。
- 取消父订单会取消所有关联的子订单。

**常见错误场景：**

| 场景 | 系统行为 |
|----------|-----------------|
| 子订单引用了不存在的父订单 | 订单被拒绝，返回 `INVALID_ORDER` 错误 |
| 父订单在子订单触发前被取消 | 子订单自动取消 |
| OCO 兄弟订单在取消传播前已成交 | 部分成交予以确认，剩余数量被取消 |
| 括号订单保证金不足 | 入场单可能成交，子订单单独被拒绝 |

:::warning
请始终在策略中处理 `OrderDenied` 和 `OrderRejected` 事件，尤其是对于关联订单，因为部分失败可能导致持仓失去保护。
:::

## 括号订单（Bracket orders）

括号订单是一种高级订单类型，允许交易者为一个持仓同时设置止盈和止损水平。这涉及下达一笔父订单（入场单）以及两笔子订单：一笔止盈 `LIMIT` 订单和一笔止损 `STOP_MARKET` 订单。
当父订单成交后，系统将下达子订单。若市场朝有利方向变动，止盈单将平仓；若市场朝不利方向变动，止损单将限制损失。

括号订单可以使用 [OrderFactory](/docs/python-api-latest/common.html#nautilus_trader.common.factories.OrderFactory) 轻松创建，该工厂支持多种订单类型、参数和指令。

在以下示例中，我们为一笔 *Market* 入场单（买入 10 份 ETHUSDT-PERP 合约）设置括号，止盈 *Limit* 单价格为 3,300 USDT，止损 *Stop-Market* 单触发价格为 2,800 USDT。
入场单默认使用 `MARKET`，止盈单默认使用 `LIMIT`，止损单默认使用 `STOP_MARKET`；止盈和止损两腿均为 `reduce_only`，并通过 `OUO` 关联关系相互关联：

```rust tab="Rust"
use nautilus_model::{
    enums::OrderSide,
    identifiers::InstrumentId,
    types::{Price, Quantity},
};

// `bracket()` returns a `bon` builder; finalize with `.call()`.
// The result is a `Vec<OrderAny>` ordered as [entry, stop-loss, take-profit].
let orders = self
    .order()
    .bracket()
    .instrument_id(InstrumentId::from("ETHUSDT-PERP.BINANCE"))
    .order_side(OrderSide::Buy)
    .quantity(Quantity::from(10))
    .tp_price(Price::from("3300.00"))         // take-profit LIMIT (default)
    .sl_trigger_price(Price::from("2800.00")) // stop-loss STOP_MARKET (default)
    .call();
```

```python tab="Python"
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model.orders import OrderList

bracket: OrderList = self.order_factory.bracket(
    instrument_id=InstrumentId.from_str("ETHUSDT-PERP.BINANCE"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(10),
    tp_price=Price.from_str("3300.00"),  # <-- take-profit LIMIT (default)
    sl_trigger_price=Price.from_str("2800.00"),  # <-- stop-loss STOP_MARKET (default)
)
```

:::warning
你应当留意持仓的保证金要求，因为为持仓设置括号订单将会占用更多的订单保证金。
:::

## 相关指南

- [订单（Orders）](index.md) - 订单概念、执行指令与订单工厂。
- [模拟订单（Emulated orders）](emulated.md) - 在不原生支持的交易场所上模拟订单类型。
- [执行（Execution）](../execution.md) - 订单执行与成交处理。
</content>
