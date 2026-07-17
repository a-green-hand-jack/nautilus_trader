# 代币化资产（Tokenized Asset）

`TokenizedAsset` 表示一种类似现货的代币，用于跟踪加密货币交易场所上的另一项资产。当交易场所提供的是代币，但其经济价值参照的是外部资产时（例如代币化股票、代币化基金或类似工具），可使用该类型。

示例包括加密货币交易场所上的代币化股票或 ETF 符号。

## 字段

| 字段               | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`      | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `asset_class`     | `AssetClass`       | `AssetClass`       | 必需             | 经济资产分类。                             |
| `base_currency`   | `Currency`         | `Currency`         | 必需             | 代币化资产或基础代币。                     |
| `quote_currency`  | `Currency`         | `Currency`         | 必需             | 用于为该代币定价的货币。                   |
| `price_precision` | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `size_precision`  | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                   |
| `price_increment` | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_increment`  | `Quantity`         | `Quantity`         | 必需             | 最小有效数量步长。                         |
| `ts_event`        | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`         | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |
| `isin`            | `Option<Ustr>`     | `str \| None`      | `None`           | 已知情况下的国际证券识别码（ISIN）。       |
| `multiplier`      | `Quantity`         | `Quantity`         | `1`              | 合约乘数。                                 |
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

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `TokenizedAsset` 的工具类别为 `Spot`（现货）。
- 它绝不是反向的，且其成本货币为计价货币。
- 当该代币参照某上市证券时，可携带 `isin` 字段。
- 它没有生效时间戳、到期日、行权价或期权类型。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::AssetClass,
    identifiers::{InstrumentId, Symbol},
    instruments::TokenizedAsset,
    types::{Currency, Price, Quantity},
};
use rust_decimal_macros::dec;

let aaplx = TokenizedAsset::builder()
    .instrument_id(InstrumentId::from("AAPLx/USD.KRAKEN"))
    .raw_symbol(Symbol::from("AAPLxUSD"))
    .asset_class(AssetClass::Equity)
    .base_currency(Currency::get_or_create_crypto("AAPLx"))
    .quote_currency(Currency::from("USD"))
    .price_precision(2)
    .size_precision(4)
    .price_increment(Price::from("0.01"))
    .size_increment(Quantity::from("0.0001"))
    .min_quantity(Quantity::from("0.0001"))
    .maker_fee(dec!(-0.0002))
    .taker_fee(dec!(0.001))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
from decimal import Decimal

from nautilus_trader.model import AssetClass
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol
from nautilus_trader.model import TokenizedAsset

aaplx = TokenizedAsset(
    instrument_id=InstrumentId.from_str("AAPLx/USD.KRAKEN"),
    raw_symbol=Symbol("AAPLxUSD"),
    asset_class=AssetClass.EQUITY,
    base_currency=Currency.from_str("AAPLx"),
    quote_currency=Currency.from_str("USD"),
    price_precision=2,
    size_precision=4,
    price_increment=Price.from_str("0.01"),
    size_increment=Quantity.from_str("0.0001"),
    ts_event=0,
    ts_init=0,
    min_quantity=Quantity.from_str("0.0001"),
    maker_fee=Decimal("-0.0002"),
    taker_fee=Decimal("0.001"),
)
```

## 适配器

创建或使用 `TokenizedAsset` 的代表性适配器包括：

- [Kraken](../../integrations/kraken.md)，用于交易场所提供的代币化资产。

## 相关指南

- [货币对（Currency Pair）](currency_pair.md) 介绍了普通的加密货币现货对。
- [股票（Equity）](equity.md) 介绍了上市现金股票。
</content>
