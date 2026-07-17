# 加密货币期权价差（Crypto Option Spread）

`CryptoOptionSpread` 表示交易所定义的加密货币期权组合策略。交易场所将该策略作为一个具有自身符号、策略类型、精度、步长和到期日的单一工具发布。

示例包括加密货币衍生品交易场所上市的 BTC 或 ETH 期权组合（combo）。

## 字段

| 字段                   | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-----------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`       | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`          | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `underlying`          | `Currency`         | `Currency`         | 必需             | 该策略所跟踪的加密资产。                   |
| `quote_currency`      | `Currency`         | `Currency`         | 必需             | 用于报出权利金的货币。                     |
| `settlement_currency` | `Currency`         | `Currency`         | 必需             | 用于结算盈亏和手续费的货币。               |
| `is_inverse`          | `bool`             | `bool`             | 必需             | 若计价/计数量方式为反向，则为 True。       |
| `strategy_type`       | `Ustr`             | `str`              | 必需             | 交易场所策略类型，例如垂直价差（vertical）。 |
| `activation_ns`       | `UnixNanos`        | `int`              | 必需             | 策略生效时间戳。                           |
| `expiration_ns`       | `UnixNanos`        | `int`              | 必需             | 策略到期时间戳。                           |
| `price_precision`     | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `size_precision`      | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                   |
| `price_increment`     | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_increment`      | `Quantity`         | `Quantity`         | 必需             | 最小有效数量步长。                         |
| `multiplier`          | `Quantity`         | `Quantity`         | `1`              | 策略乘数。                                 |
| `lot_size`            | `Quantity`         | `Quantity`         | `1`              | 取整后的手数或标准手规模。                 |
| `max_quantity`        | `Option<Quantity>` | `Quantity \| None` | `None`           | 最大订单数量。                             |
| `min_quantity`        | `Option<Quantity>` | `Quantity \| None` | `None`           | 最小订单数量。                             |
| `max_notional`        | `Option<Money>`    | `Money \| None`    | `None`           | 最大订单名义价值。                         |
| `min_notional`        | `Option<Money>`    | `Money \| None`    | `None`           | 最小订单名义价值。                         |
| `max_price`           | `Option<Price>`    | `Price \| None`    | `None`           | 最大有效报价或订单价格。                   |
| `min_price`           | `Option<Price>`    | `Price \| None`    | `None`           | 最小有效报价或订单价格。                   |
| `margin_init`         | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 初始保证金率。                             |
| `margin_maint`        | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 维持保证金率。                             |
| `maker_fee`           | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 挂单方费率。负值表示返佣。                 |
| `taker_fee`           | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 吃单方费率。负值表示返佣。                 |
| `tick_scheme`         | `Option<Ustr>`     | `str \| None`      | `None`           | 所注册的可变最小报价单位方案名称。         |
| `info`                | `Option<Params>`   | `dict \| None`     | `None`           | 适配器元数据。                             |
| `ts_event`            | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`             | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `CryptoOptionSpread` 的资产类别为 `Cryptocurrency`，工具类别为 `OptionSpread`。
- 交易场所将该价差发布为单一可交易工具。
- 根据所设定的货币不同，该策略可以是线性、反向或量化的。
- 若适配器提供了交易场所特定的分腿细节，将其存储在 `info` 字段中。

## 示例

```rust tab="Rust"
use chrono::{TimeZone, Utc};
use nautilus_core::UnixNanos;
use nautilus_model::{
    identifiers::{InstrumentId, Symbol},
    instruments::CryptoOptionSpread,
    types::{Currency, Price, Quantity},
};
use rust_decimal_macros::dec;
use ustr::Ustr;

let activation = Utc.with_ymd_and_hms(2026, 5, 12, 0, 0, 0).unwrap();
let expiration = Utc.with_ymd_and_hms(2026, 5, 19, 8, 0, 0).unwrap();

let btc_spread = CryptoOptionSpread::builder()
    .instrument_id(InstrumentId::from("BTC-CS-19MAY26-70000_75000.DERIBIT"))
    .raw_symbol(Symbol::from("BTC-CS-19MAY26-70000_75000"))
    .underlying(Currency::from("BTC"))
    .quote_currency(Currency::from("USD"))
    .settlement_currency(Currency::from("BTC"))
    .is_inverse(false)
    .strategy_type(Ustr::from("CS"))
    .activation_ns(UnixNanos::from(activation.timestamp_nanos_opt().unwrap() as u64))
    .expiration_ns(UnixNanos::from(expiration.timestamp_nanos_opt().unwrap() as u64))
    .price_precision(4)
    .size_precision(1)
    .price_increment(Price::from("0.0001"))
    .size_increment(Quantity::from("0.1"))
    .multiplier(Quantity::from("1"))
    .min_quantity(Quantity::from("0.1"))
    .maker_fee(dec!(0.0003))
    .taker_fee(dec!(0.0003))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
from decimal import Decimal

import pandas as pd

from nautilus_trader.model import CryptoOptionSpread
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

BTC = Currency.from_str("BTC")
USD = Currency.from_str("USD")

btc_spread = CryptoOptionSpread(
    instrument_id=InstrumentId.from_str("BTC-CS-19MAY26-70000_75000.DERIBIT"),
    raw_symbol=Symbol("BTC-CS-19MAY26-70000_75000"),
    underlying=BTC,
    quote_currency=USD,
    settlement_currency=BTC,
    is_inverse=False,
    strategy_type="CS",
    activation_ns=pd.Timestamp("2026-05-12T00:00:00", tz="UTC").value,
    expiration_ns=pd.Timestamp("2026-05-19T08:00:00", tz="UTC").value,
    price_precision=4,
    size_precision=1,
    price_increment=Price.from_str("0.0001"),
    size_increment=Quantity.from_str("0.1"),
    min_quantity=Quantity.from_str("0.1"),
    maker_fee=Decimal("0.0003"),
    taker_fee=Decimal("0.0003"),
    ts_event=0,
    ts_init=0,
)
```

## 适配器

创建或使用 `CryptoOptionSpread` 的代表性适配器包括：

- [Deribit](../../integrations/deribit.md)，用于加密货币期权组合。
- [OKX](../../integrations/okx.md)，用于加密货币期权价差市场。

## 相关指南

- [加密货币期权（Crypto Option）](crypto_option.md) 介绍了单腿的加密货币期权。
- [期权价差（Option Spread）](option_spread.md) 介绍了非加密货币期权价差。
</content>
