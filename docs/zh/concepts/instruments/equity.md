# 股票（Equity）

`Equity` 表示上市股票、ETF 或类似的现金市场证券。Nautilus 使用该类型表示以整数单位交易、以单一货币报价、且没有合约到期日的工具。

示例包括 `AAPL.XNAS`、`MSFT.XNAS` 以及交易场所特定的 ETF 符号。

## 字段

| 字段               | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`      | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `currency`        | `Currency`         | `Currency`         | 必需             | 报价及结算货币。                           |
| `price_precision` | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `price_increment` | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `lot_size`        | `Option<Quantity>` | `Quantity \| None` | `None`           | 标准手（board lot）或整股手数规模。         |
| `ts_event`        | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`         | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |
| `isin`            | `Option<Ustr>`     | `str \| None`      | `None`           | 已知情况下的国际证券识别码（ISIN）。       |
| `max_quantity`    | `Option<Quantity>` | `Quantity \| None` | `None`           | 最大订单数量。                             |
| `min_quantity`    | `Option<Quantity>` | `Quantity \| None` | `None`           | 最小订单数量。                             |
| `max_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最大有效报价或订单价格。                   |
| `min_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最小有效报价或订单价格。                   |
| `margin_init`     | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 初始保证金率。                             |
| `margin_maint`    | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 维持保证金率。                             |
| `maker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 挂单方费率。负值表示返佣。                 |
| `taker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 吃单方费率。负值表示返佣。                 |
| `tick_scheme`     | `Option<Ustr>`     | `str \| None`      | `None`           | 所注册的可变最小报价单位方案名称。         |
| `info`            | `Option<Params>`   | `dict \| None`     | `None`           | 适配器元数据。                             |

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `Equity` 的资产类别为 `Equity`（股票），工具类别为 `Spot`（现货）。
- 数量精度始终为零，因此订单使用整股数量。
- 乘数和数量步长均为 1。
- 它没有基础货币、到期日、行权价、期权类型或反向计价标志。
- 只有当交易场所公布价格限制时，才使用价格限制字段。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    identifiers::{InstrumentId, Symbol},
    instruments::Equity,
    types::{Currency, Price, Quantity},
};
use ustr::Ustr;

let aapl = Equity::builder()
    .instrument_id(InstrumentId::from("AAPL.XNAS"))
    .raw_symbol(Symbol::from("AAPL"))
    .isin(Ustr::from("US0378331005"))
    .currency(Currency::from("USD"))
    .price_precision(2)
    .price_increment(Price::from("0.01"))
    .lot_size(Quantity::from("100"))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
from nautilus_trader.model import Currency
from nautilus_trader.model import Equity
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

aapl = Equity(
    instrument_id=InstrumentId.from_str("AAPL.XNAS"),
    raw_symbol=Symbol("AAPL"),
    currency=Currency.from_str("USD"),
    price_precision=2,
    price_increment=Price.from_str("0.01"),
    ts_event=0,
    ts_init=0,
    isin="US0378331005",
    lot_size=Quantity.from_int(100),
)
```

## 适配器

创建或使用 `Equity` 的代表性适配器包括：

- [Databento](../../integrations/databento.md)，用于上市美国股票和 ETF。
- [Interactive Brokers](../../integrations/ib.md)，用于上市股票合约。

## 相关指南

- [数据（Data）](../data/) 说明了引用工具的市场数据。
- [值类型（Value types）](../value_types.md) 介绍了 `Price`、`Quantity` 和 `Money`。
</content>
