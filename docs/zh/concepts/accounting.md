# 账户（Accounting）

会计（accounting）子系统为平台交互的每个账户跟踪余额、保证金和
盈亏。本指南涵盖数据模型、策略使用的查询 API，
以及适配器作者为了在各交易场所之间保持一致性而必须遵循的约定。

它同样适用于回测和实盘交易。有关回测专属配置
（起始余额、按交易场所选择保证金模型），请参阅
[回测（Backtesting）](backtesting/)。

## 账户类型

当你将某个交易场所接入引擎（用于实盘交易或回测）时，
你通过 `account_type` 选择三种会计模式之一：

| 账户类型 | 典型使用场景                                 | 引擎锁定的内容                                                     |
| ------------ | ------------------------------------------------ | ------------------------------------------------------------------------- |
| Cash（现金）         | 现货交易（例如 BTC/USDT、股票）            | 每一笔待处理订单将要开立的仓位的名义价值。             |
| Margin（保证金）       | 衍生品或任何允许使用杠杆的产品  | 每笔订单的初始保证金加上未平仓仓位的维持保证金。 |
| Betting（投注）      | 体育博彩、庄家做市                       | 交易场所要求的赌注；不涉及杠杆。                                 |

### 现金账户

现金账户全额结算交易；不存在杠杆，因此也不存在保证金的概念。
锁定余额反映了为待处理订单预留的名义金额。

### 保证金账户

保证金账户支持需要抵押品的金融工具，例如期货或
带杠杆的加密永续合约。它们跟踪账户余额，为未平仓订单和仓位
预留保证金，并对每个金融工具应用可配置的杠杆。保证金按
两种作用域跟踪；参见下方的[保证金作用域](#margin-scopes)。

**关键术语**：

- **杠杆（Leverage）**：相对于账户权益放大敞口。更高的杠杆
  同时提高了潜在收益和风险。
- **初始保证金（Initial margin）**：提交订单时预留的抵押品。
- **维持保证金（Maintenance margin）**：维持一个未平仓仓位所需的最低抵押品。
- **锁定余额（Locked balance）**：作为抵押品预留的资金，不可用于新的订单。

:::note
Reduce-only（只减仓）订单不会计入现金账户的 `balance_locked`，
也不会增加保证金账户的初始保证金，因为它们只能减少敞口。
:::

### 投注账户

投注账户是为需要押注一笔金额以赢取或损失固定赔付的交易场所
（预测市场、体育博彩公司）而设计的专用账户。引擎只锁定
交易场所要求的赌注；不适用杠杆和保证金。

## 余额模型

一个 `AccountBalance` 以同一种货币持有三个值：

- `total`：交易场所报告的总余额数字（钱包余额、净清算价值，
  或保证金余额，取决于交易场所）。
- `locked`：为未平仓订单和仓位预留的金额。
- `free`：可用于新订单的金额（`total - locked`）。

不变式 `total == locked + free` 在货币精度上必须始终成立。

Python 中的 `AccountBalance(total, locked, free)` 构造函数要求
一次性提供全部三个字段。用 Rust 编写的适配器代码还有两个额外的
派生构造函数，用于集中强制该不变式；当交易场所只报告三个值中的
两个时，应优先使用它们，而不是 `AccountBalance::new`：

| Rust 辅助函数                             | 使用场景                                                                    |
| --------------------------------------- | ------------------------------------------------------------------------------ |
| `AccountBalance::from_total_and_locked` | 交易场所报告 total 和 locked；`free` 是派生的，并被限定在 `[0, total]` 范围内。 |
| `AccountBalance::from_total_and_free`   | 交易场所报告 total 和 free；`locked` 是派生的，并被限定范围。                 |
| `AccountBalance::new`                   | 三个值均已知且一致（测试、直通场景）。       |

当 `total >= 0` 时，这些辅助函数会将派生字段限定在 `[0, total]` 范围内，
因此因交易场所舍入导致的瞬时超出永远不会使账户处于损坏状态。

## 货币与估值契约

会计数值在成功进行显式转换之前，会保留其来源货币。这可以防止一个
有效的数字被贴上错误的货币标签，或将一个不可用的值被当作零处理。

| 值                        | 货币契约                                              |
|------------------------------|----------------------------------------------------------------|
| 金融工具成本货币     | 反向合约用基础货币，数量对冲（quanto）合约用结算货币，其余情况用报价货币。  |
| 仓位盈亏                 | 仓位开仓时记录的金融工具成本货币。     |
| 计算得出的锁定和保证金 | 各计算金额自身的货币，独立进行转换。    |
| 投资组合汇总         | 原生分组，或在转换成功后使用账户基础货币。 |

聚合只会合并兼容的 `Money` 值。没有基础货币的账户会保留各自独立的
原生货币分组。单一金融工具的已实现盈亏查询会返回“不可用”，
而不是合并不同的货币。

会计和估值路径还遵循以下规则：

- 无效或无法表示的名义金额、盈亏、手续费、锁定余额和保证金结果
  会产生错误、返回不可用值，或标记为未估值状态。它们不会用零来替代。
  已实现盈亏重新计算失败时，也会清除之前缓存的结果。
- 对于没有基础货币的多币种现金账户，`equity()` 会将一个已入账的
  非反向基础资产只计算一次。`mark_values()` 仍然是一个毛仓位价值查询，
  会包含该资产。
- MTM 快照会区分沿用的过期输入与从未拥有完整估值数据的仓位。
  过期价格元数据只覆盖有未平仓状态的金融工具与仓位方向组合。

有关权益公式、价格和汇率选择、快照元数据以及缺失价格的查询范围，
请参阅[投资组合（Portfolio）](portfolio.md#equity-and-mark-to-market)。

## 保证金作用域

一个 `MarginBalance` 有四个字段：`initial`、`maintenance`、`currency`，
以及一个用于选择两种作用域之一的 `Optional[InstrumentId]`。

### 按金融工具作用域

`MarginBalance.instrument_id` 被设置为一个具体的金融工具。适用于：

- 逐仓保证金（每个仓位独立的抵押品）。
- 回测或计算得出的保证金，此时 `AccountsManager` 会根据每个金融工具的
  未平仓订单和仓位在本地推导出保证金。

### 账户级作用域

`MarginBalance.instrument_id` 为 `None`。该条目以其 `currency`
（抵押品货币）为键。适用于报告每种抵押品货币单一汇总值的
全仓保证金（cross-margin）交易场所。一个交易场所可能会发出一条
账户级条目（单一抵押品的全仓保证金），也可能发出多条
（每种抵押品币种一条）。

两种作用域在同一个 `MarginAccount` 上共存于各自独立的内部存储中。
一个 `AccountState` 事件可能携带任一作用域或两种作用域的条目，
`MarginAccount.apply()` 会根据 `instrument_id` 是否被设置，
将每个条目路由到正确的存储。

:::note
`MarginAccount.apply()` 会用传入事件**替换**两个存储的内容。它
不会与之前的状态合并。发出部分快照的适配器必须在每次更新时
包含所有当前有效的保证金条目，否则这些条目会在下一次完整快照到来之前
被丢弃。余额列表同样会被替换。
:::

## 策略查询 API

使用与交易场所报告形态相匹配的查询方式。如果交易场所报告
按金融工具的保证金，就按 `InstrumentId` 查询。如果它报告
账户级的保证金，就按 `Currency` 查询。

| 你想要的值的作用域            | 使用                                                                                             |
| -------------------------------------- | ----------------------------------------------------------------------------------------------- |
| 按金融工具的保证金（逐仓）       | `margin(id)` / `margin_init(id)` / `margin_maint(id)`                                           |
| 单一抵押品的账户级保证金 | `margin_for_currency(ccy)` / `margin_init_for_currency(ccy)` / `margin_maint_for_currency(ccy)` |
| 跨两种作用域的合计            | `total_margin_init(ccy)` / `total_margin_maint(ccy)`                                            |

单点查询在条目不存在时返回 `None`；合计查询始终
返回一个 `Money`（如果没有任何匹配，则为该货币的零值）。

:::note
下面列出的名称是 `MarginAccount` 上的 Python / Cython API。使用
`nautilus-model` crate 的 Rust 策略会调用 `account_margin(&currency)`、
`account_initial_margin(&currency)`、`account_maintenance_margin(&currency)`、
`total_initial_margin(currency)` 和 `total_maintenance_margin(currency)`：
按 `Option<InstrumentId>` 进行相同的划分，只是方法名不同。
:::

### 按金融工具的查询（`MarginAccount`）

- `margin(instrument_id) -> MarginBalance | None`
- `margin_init(instrument_id) -> Money | None`
- `margin_maint(instrument_id) -> Money | None`
- `margins() -> dict[InstrumentId, MarginBalance]`（所有按金融工具的条目）
- `margins_init() -> dict[InstrumentId, Money]`
- `margins_maint() -> dict[InstrumentId, Money]`

这些方法只能看到按金融工具的存储。在全仓保证金交易场所上，
它们会返回空字典或 `None`。请使用下面的账户级查询。

### 账户级查询（`MarginAccount`）

- `margin_for_currency(currency) -> MarginBalance | None`
- `margin_init_for_currency(currency) -> Money | None`
- `margin_maint_for_currency(currency) -> Money | None`
- `account_margins() -> dict[Currency, MarginBalance]`（所有账户级条目）
- `account_margins_init() -> dict[Currency, Money]`
- `account_margins_maint() -> dict[Currency, Money]`

### 合计（`MarginAccount`）

这些方法会对给定货币在按金融工具和账户级条目上进行求和：

- `total_margin_init(currency) -> Money`
- `total_margin_maint(currency) -> Money`

当策略在一个可能同时出现两种作用域的交易场所上交易时
（例如逐仓仓位与全仓保证金抵押品并存），这些方法很有用。

### 清除账户级条目

- `clear_account_margin(currency)` 移除给定抵押品货币的账户级条目，
  并触发余额重新计算。对应按金融工具条目的方法是
  `clear_margin(instrument_id)`。

这些是系统方法；适配器代码通过 `MarginAccount.apply()` 隐式调用它们。
策略通常不需要直接使用它们。

### 投资组合级查询

保证金查询：

- `portfolio.margins_init(venue=..., account_id=...) -> dict[InstrumentId, Money]`
- `portfolio.margins_maint(venue=..., account_id=...) -> dict[InstrumentId, Money]`

这些方法与 `MarginAccount.margins_init` / `margins_maint` 相对应，
只返回按金融工具的条目。对于全仓保证金交易场所上的账户级数据，
请通过 `portfolio.account(venue).margin_init_for_currency(ccy)` 直接查询该账户。

盈亏、敞口、按市值计价和权益查询都接受 `venue` 和一个可选的
`account_id`，用于限定多账户交易场所的范围：

- `portfolio.unrealized_pnls(venue=..., account_id=...) -> dict[Currency, Money]`
- `portfolio.realized_pnls(venue=..., account_id=...) -> dict[Currency, Money]`
- `portfolio.total_pnls(venue=..., account_id=...) -> dict[Currency, Money]`
- `portfolio.net_exposures(venue=..., account_id=...) -> dict[Currency, Money]`
- `portfolio.mark_values(venue=..., account_id=...) -> dict[Currency, Money]`
- `portfolio.equity(venue=..., account_id=...) -> dict[Currency, Money]`
- `portfolio.missing_price_instruments(venue) -> list[InstrumentId]`

有关权益公式、价格回退链、基础货币转换行为，以及只警告一次的
缺失价格跟踪器，请参阅[投资组合指南](portfolio.md#equity-and-mark-to-market)。

### 示例演算

单一抵押品全仓保证金（一条账户级条目）：

```python
usdc_margin = margin_account.margin_init_for_currency(USDC)
usdc_total = margin_account.total_margin_init(USDC)
```

按币种的全仓保证金（每种抵押品货币一条条目）：

```python
for ccy, margin_balance in margin_account.account_margins().items():
    print(ccy, margin_balance.initial, margin_balance.maintenance)
```

## 保证金模型

NautilusTrader 为计算路径（回测，以及以 `calculate_account_state=True`
运行以进行核对的实盘策略）提供了灵活的保证金计算模型。
交易场所报告的保证金会直接流入 `_account_margins` 或 `_margins`，
不经过任何模型。

### 概览

不同的交易场所对杠杆的处理方式不同：

- **传统经纪商**（盈透证券、TD Ameritrade）：无论杠杆如何，都使用固定的保证金百分比。
- **加密货币交易所**（Binance 等）：杠杆可能会降低保证金要求。

两种内置模型都使用金融工具的 `margin_init` 和 `margin_maint` 字段，
将保证金计算为名义金额的百分比。它们的区别仅在于杠杆是否会
降低预留金额。对于具有真正逐合约固定保证金的交易场所（CME / ICE），
请设置 `instrument.margin_init` 和 `margin_maint`，使该百分比
换算出所需的美元金额，或者实现一个[自定义模型](#custom-models)。

### HEDGING 模式下的净额化

在 `OmsType.HEDGING` 下，每笔成交都会开立自己的 `Position`，
因此一个账户可能持有同一金融工具的多个未平仓子仓位。账户管理器
（accounts manager）会按 `ts_opened` 顺序，将这些子仓位净额化为一个
假想的 NETTING 仓位，然后在得到的净带符号数量和平均开仓价格上
运行一次保证金模型。

这一重演过程遵循与 `Position.apply` 相同的规则：同方向的成交
产生按数量加权的平均开仓价格，反方向的成交按现有均价部分平仓，
而跨越零点的成交会使剩余部分采用发生翻转的那笔成交的价格。
共享同一个 `ts_opened` 的子仓位按 `(ts_opened, position_id)` 顺序
合并，因此结果不依赖于缓存的迭代顺序。

对于相同的成交序列，HEDGING 账户和 NETTING 账户计算出的
维持保证金相同；该要求会随净经济敞口而变化。

### 可用模型

#### `StandardMarginModel`

使用固定百分比，不进行杠杆除法，与传统经纪商行为一致。

```python
# Fixed percentages - leverage ignored
margin = notional * instrument.margin_init
```

- 初始保证金：`notional_value * instrument.margin_init`
- 维持保证金：`notional_value * instrument.margin_maint`

**使用场景**：传统经纪商（盈透证券）、具有固定保证金要求的外汇经纪商。

#### `LeveragedMarginModel`

将保证金要求除以杠杆。

```python
# Leverage reduces margin requirements
adjusted_notional = notional / leverage
margin = adjusted_notional * instrument.margin_init
```

- 初始保证金：`(notional_value / leverage) * instrument.margin_init`
- 维持保证金：`(notional_value / leverage) * instrument.margin_maint`

**使用场景**：随杠杆降低保证金的加密货币交易所，以及杠杆会影响
保证金要求的交易场所。

### 默认行为

`MarginAccount` 默认使用 `LeveragedMarginModel`。可通过编程方式覆盖：

```python
from nautilus_trader.backtest.models import LeveragedMarginModel
from nautilus_trader.backtest.models import StandardMarginModel
from nautilus_trader.test_kit.stubs.execution import TestExecStubs

account = TestExecStubs.margin_account()

# Traditional broker behavior
account.set_margin_model(StandardMarginModel())

# Or the leveraged model (default)
account.set_margin_model(LeveragedMarginModel())
```

### 演算示例：EUR/USD

- **金融工具**：EUR/USD
- **数量**：100,000 EUR
- **价格**：1.10000
- **名义金额**：$110,000
- **杠杆**：50 倍
- **`instrument.margin_init`**：3%

| 模型     | 计算            | 结果 | 百分比 |
| --------- | ---------------------- | ------ | ---------- |
| Standard  | $110,000 × 0.03        | $3,300 | 3.00%      |
| Leveraged | ($110,000 ÷ 50) × 0.03 | $66    | 0.06%      |

在一个 10,000 美元的账户上：标准模型会阻止该笔交易；杠杆模型
则允许它。

### 自定义模型

继承 `MarginModel`，并通过 `MarginModelConfig` 接收配置：

```python
from decimal import Decimal

from nautilus_trader.backtest.config import MarginModelConfig
from nautilus_trader.backtest.models import MarginModel
from nautilus_trader.model.objects import Money


class RiskAdjustedMarginModel(MarginModel):
    def __init__(self, config: MarginModelConfig) -> None:
        self.risk_multiplier = Decimal(str(config.config.get("risk_multiplier", 1.0)))
        self.use_leverage = config.config.get("use_leverage", False)

    def calculate_margin_init(self, instrument, quantity, price, leverage, use_quote_for_inverse=False):
        notional = instrument.notional_value(quantity, price, use_quote_for_inverse)

        if self.use_leverage:
            adjusted = notional.as_decimal() / leverage
        else:
            adjusted = notional.as_decimal()

        margin = adjusted * instrument.margin_init * self.risk_multiplier
        return Money(margin, instrument.quote_currency)

    def calculate_margin_maint(self, instrument, side, quantity, price, leverage, use_quote_for_inverse=False):
        return self.calculate_margin_init(instrument, quantity, price, leverage, use_quote_for_inverse)
```

有关通过 `BacktestVenueConfig` 和 `MarginModelConfig` 进行回测范围内
保证金模型配置的信息，请参阅[回测（Backtesting）](backtesting/accounts-and-margin.md#margin-models)
中的保证金模型部分。

## 适配器约定

实盘适配器将交易场所的响应转换为 `AccountBalance` 和
`MarginBalance` 实例。适配器作者必须遵循以下约定：

### 构建 `AccountBalance`

优先使用派生辅助函数，以便在中心位置统一强制执行限定范围和
`total == locked + free` 不变式。手动计算三个字段并传给
`AccountBalance::new` 仅适用于三个值都已经是权威值的直通路径
（例如测试）。

### 构建 `MarginBalance`

选择与交易场所报告方式相匹配的作用域：

| 交易场所报告的内容                                  | 作用域          | 使用方式                                                  |
| ---------------------------------------------- | -------------- | ---------------------------------------------------------- |
| 按金融工具（逐仓仓位）            | 按金融工具 | `MarginBalance::new(initial, maint, Some(id))`             |
| 每种抵押品单一汇总值（全仓保证金） | 账户级   | `MarginBalance::new(initial, maint, None)`                 |
| 多个汇总值，每种抵押品一个        | 账户级   | 每种货币一个 `MarginBalance`，`instrument_id=None` |

:::note
不会使用合成的 `ACCOUNT.{VENUE}` 或 `ACCOUNT-{COIN}.{VENUE}`
`InstrumentId` 占位符。账户级条目携带 `instrument_id=None`，
以 `currency` 为键。
:::

## 相关指南

- [回测（Backtesting）](backtesting/)：起始余额、`MarginModelConfig`，以及
  特定于回测的账户设置。
- [投资组合（Portfolio）](portfolio.md)：投资组合级别的盈亏、敞口和货币
  转换。
- [仓位（Positions）](positions.md)：仓位生命周期、聚合和盈亏。
- [适配器（Adapters）](adapters.md)：适配器作者的要求和最佳实践。
