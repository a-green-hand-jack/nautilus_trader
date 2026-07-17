# 指数工具（Index Instrument）

`IndexInstrument` 表示一个参考指数，例如股票指数、波动率指数或基准价格序列。它携带精度和步长元数据，使 Nautilus 能够一致地存储和传递价格，但它本身并不是可直接交易的合约。

示例包括 `SPX.XCBO`、`VIX.XCBO` 以及交易场所特定的参考指数。

## 字段

| 字段               | Rust 类型        | Python 类型    | 是否必需/默认值 | 说明                                       |
|-------------------|------------------|----------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`   | `InstrumentId` | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`      | `Symbol`         | `Symbol`       | 必需             | 原生交易场所符号。                         |
| `currency`        | `Currency`       | `Currency`     | 必需             | 报价数值所参照的货币。                     |
| `price_precision` | `u8`             | `int`          | 必需             | 价格允许的小数位数。                       |
| `size_precision`  | `u8`             | `int`          | 必需             | 数量允许的小数位数。                       |
| `price_increment` | `Price`          | `Price`        | 必需             | 最小有效价格步长。                         |
| `size_increment`  | `Quantity`       | `Quantity`     | 必需             | 最小有效数量步长。                         |
| `ts_event`        | `UnixNanos`      | `int`          | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`         | `UnixNanos`      | `int`          | 必需             | 以纳秒为单位的初始化时间戳。               |
| `tick_scheme`     | `Option<Ustr>`   | `str \| None`  | `None`           | 所注册的可变最小报价单位方案名称。         |
| `info`            | `Option<Params>` | `dict \| None` | `None`           | 适配器元数据。                             |

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `IndexInstrument` 的资产类别为 `Index`（指数），工具类别为 `Spot`（现货）。
- 它是一个参考工具，不应用于提交订单。
- 它没有限制、保证金、手续费、合约乘数、到期日或结算货币。
- 对于以指数为标的的可交易衍生品，应使用期权或期货类型。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    identifiers::{InstrumentId, Symbol},
    instruments::IndexInstrument,
    types::{Currency, Price, Quantity},
};

let spx = IndexInstrument::builder()
    .instrument_id(InstrumentId::from("SPX.XCBO"))
    .raw_symbol(Symbol::from("SPX"))
    .currency(Currency::from("USD"))
    .price_precision(2)
    .size_precision(0)
    .price_increment(Price::from("0.01"))
    .size_increment(Quantity::from("1"))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
from nautilus_trader.model import Currency
from nautilus_trader.model import IndexInstrument
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

spx = IndexInstrument(
    instrument_id=InstrumentId.from_str("SPX.XCBO"),
    raw_symbol=Symbol("SPX"),
    currency=Currency.from_str("USD"),
    price_precision=2,
    size_precision=0,
    price_increment=Price.from_str("0.01"),
    size_increment=Quantity.from_str("1"),
    ts_event=0,
    ts_init=0,
)
```

## 适配器

创建或使用 `IndexInstrument` 的代表性适配器包括：

- [Interactive Brokers](../../integrations/ib.md)，用于参考指数。
- [Databento](../../integrations/databento.md)，用于参考数据源。

## 相关指南

- [期权合约（Option Contract）](option_contract.md) 介绍了以指数为标的的上市期权。
- [期货合约（Futures Contract）](futures_contract.md) 介绍了指数期货。
</content>
