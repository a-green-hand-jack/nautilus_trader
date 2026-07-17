# 差价合约（Cfd）

`Cfd` 表示跟踪某标的资产的差价合约（Contract for Difference），但并不转移标的资产的所有权。交易场所定义计价货币、精度、步长、限制、保证金和手续费。

示例包括外汇、股票、指数和商品上的差价合约。

## 字段

| 字段              | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`      | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `asset_class`     | `AssetClass`       | `AssetClass`       | 必需             | 标的资产的资产类别。                       |
| `base_currency`   | `Option<Currency>` | `Currency \| None` | `None`           | 该差价合约跟踪某货币时的基础货币。         |
| `quote_currency`  | `Currency`         | `Currency`         | 必需             | 用于报价和计价的货币。                     |
| `price_precision` | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `size_precision`  | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                   |
| `price_increment` | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_increment`  | `Quantity`         | `Quantity`         | 必需             | 最小有效数量步长。                         |
| `lot_size`        | `Option<Quantity>` | `Quantity \| None` | `None`           | 取整后的手数或标准手规模。                 |
| `max_quantity`    | `Option<Quantity>` | `Quantity \| None` | `None`           | 最大订单数量。                             |
| `min_quantity`    | `Option<Quantity>` | `Quantity \| None` | `None`           | 最小订单数量。                             |
| `max_notional`    | `Option<Money>`    | `Money \| None`    | `None`           | 最大订单名义价值。                         |
| `min_notional`    | `Option<Money>`    | `Money \| None`    | `None`           | 最小订单名义价值。                         |
| `max_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最大有效报价或订单价格。                   |
| `min_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最小有效报价或订单价格。                   |
| `margin_init`     | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 初始保证金率。                             |
| `margin_maint`    | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 维持保证金率。                             |
| `maker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 挂单方费率。负值表示返佣。                 |
| `taker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 吃单方费率。负值表示返佣。                 |
| `tick_scheme`     | `Option<Ustr>`     | `str \| None`      | `None`           | 所注册的可变最小报价单位方案名称。         |
| `info`            | `Option<Params>`   | `dict \| None`     | `None`           | 适配器元数据。                             |
| `ts_event`        | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`         | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `Cfd` 的工具类别为 `Cfd`。
- 它绝不是反向的，乘数为 1。
- 它没有生效时间戳、到期时间戳、行权价或期权类型。
- 当某交易场所同时提供现金工具和差价合约时，请使用来源市场类型加以区分。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::AssetClass,
    identifiers::{InstrumentId, Symbol},
    instruments::Cfd,
    types::{Currency, Price, Quantity},
};
use rust_decimal_macros::dec;

let audusd = Cfd::builder()
    .instrument_id(InstrumentId::from("AUDUSD.OANDA"))
    .raw_symbol(Symbol::from("AUD/USD"))
    .asset_class(AssetClass::FX)
    .base_currency(Currency::from("AUD"))
    .quote_currency(Currency::from("USD"))
    .price_precision(5)
    .size_precision(0)
    .price_increment(Price::from("0.00001"))
    .size_increment(Quantity::from("1"))
    .lot_size(Quantity::from("1000"))
    .margin_init(dec!(0.03))
    .margin_maint(dec!(0.03))
    .maker_fee(dec!(0.00002))
    .taker_fee(dec!(0.00002))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
from decimal import Decimal

from nautilus_trader.model import AssetClass
from nautilus_trader.model import Cfd
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

audusd = Cfd(
    instrument_id=InstrumentId.from_str("AUDUSD.OANDA"),
    raw_symbol=Symbol("AUD/USD"),
    asset_class=AssetClass.FX,
    quote_currency=Currency.from_str("USD"),
    price_precision=5,
    price_increment=Price.from_str("0.00001"),
    size_precision=0,
    size_increment=Quantity.from_int(1),
    ts_event=0,
    ts_init=0,
    base_currency=Currency.from_str("AUD"),
    lot_size=Quantity.from_int(1000),
    margin_init=Decimal("0.03"),
    margin_maint=Decimal("0.03"),
    maker_fee=Decimal("0.00002"),
    taker_fee=Decimal("0.00002"),
)
```

## 适配器

创建或使用 `Cfd` 的代表性适配器包括：

- [Interactive Brokers](../../integrations/ib.md)，用于差价合约。

## 相关指南

- [货币对（Currency Pair）](currency_pair.md) 介绍了现金外汇和加密货币现货对。
- [商品（Commodity）](commodity.md) 介绍了现货商品工具。
</content>
