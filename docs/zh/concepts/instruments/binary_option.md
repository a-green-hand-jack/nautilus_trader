# 二元期权（Binary Option）

`BinaryOption` 表示一种二元结果工具，根据某一条件是否成立而结算为固定的收益。它可用于建模预测市场（prediction market）、二元期权，或交易场所特定的是/否合约。

示例包括预测市场结果以及二元事件合约。

## 字段

| 字段              | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                        |
|-------------------|--------------------|--------------------|------------------|--------------------------------------------|
| `instrument_id`   | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                    |
| `raw_symbol`      | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                          |
| `asset_class`     | `AssetClass`       | `AssetClass`       | 必需             | 结果市场的资产类别。                        |
| `currency`        | `Currency`         | `Currency`         | 必需             | 报价及结算货币。                            |
| `activation_ns`   | `UnixNanos`        | `int`              | 必需             | 合约生效时间戳。                            |
| `expiration_ns`   | `UnixNanos`        | `int`              | 必需             | 合约到期时间戳。                            |
| `price_precision` | `u8`               | `int`              | 必需             | 价格允许的小数位数。                        |
| `size_precision`  | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                    |
| `price_increment` | `Price`            | `Price`            | 必需             | 最小有效价格步长。                          |
| `size_increment`  | `Quantity`         | `Quantity`         | 必需             | 最小有效数量步长。                          |
| `outcome`         | `Option<Ustr>`     | `str \| None`      | `None`           | 交易场所提供时的结果标签。                  |
| `description`     | `Option<Ustr>`     | `str \| None`      | `None`           | 供人阅读的市场描述。                        |
| `max_quantity`    | `Option<Quantity>` | `Quantity \| None` | `None`           | 最大订单数量。                              |
| `min_quantity`    | `Option<Quantity>` | `Quantity \| None` | `None`           | 最小订单数量。                              |
| `max_notional`    | `Option<Money>`    | `Money \| None`    | `None`           | 最大订单名义价值。                          |
| `min_notional`    | `Option<Money>`    | `Money \| None`    | `None`           | 最小订单名义价值。                          |
| `max_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最大有效报价或订单价格。                    |
| `min_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最小有效报价或订单价格。                    |
| `margin_init`     | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 初始保证金率。                              |
| `margin_maint`    | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 维持保证金率。                              |
| `maker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 挂单方费率。负值表示返佣。                  |
| `taker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 吃单方费率。负值表示返佣。                  |
| `tick_scheme`     | `Option<Ustr>`     | `str \| None`      | `None`           | 所注册的可变最小报价单位方案名称。          |
| `info`            | `Option<Params>`   | `dict \| None`     | `None`           | 适配器元数据。                              |
| `ts_event`        | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                  |
| `ts_init`         | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。                |

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `BinaryOption` 的工具类别为 `BinaryOption`。
- 它绝不是反向的，乘数和手数均为 1。
- 许多交易场所将二元结果的报价范围设为 0 到 1，但具体的允许价格区间和最小报价单位由交易场所定义。
- `outcome` 和 `description` 为该合约提供了便于人类阅读的上下文信息。

## 示例

```rust tab="Rust"
use chrono::{TimeZone, Utc};
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::AssetClass,
    identifiers::{InstrumentId, Symbol, Venue},
    instruments::BinaryOption,
    types::{Currency, Price, Quantity},
};
use rust_decimal_macros::dec;
use ustr::Ustr;

let raw_symbol = Symbol::from(
    "0x12a0cb60174abc437bf1178367c72d11f069e1a3add20b148fb0ab4279b772b2-92544998123698303655208967887569360731013655782348975589292031774495159624905",
);
let expiration = Utc.with_ymd_and_hms(2024, 1, 1, 0, 0, 0).unwrap();

let yes_outcome = BinaryOption::builder()
    .instrument_id(InstrumentId::new(raw_symbol, Venue::from("POLYMARKET")))
    .raw_symbol(raw_symbol)
    .asset_class(AssetClass::Alternative)
    .currency(Currency::from("USDC"))
    .activation_ns(UnixNanos::default())
    .expiration_ns(UnixNanos::from(expiration.timestamp_nanos_opt().unwrap() as u64))
    .price_precision(3)
    .size_precision(2)
    .price_increment(Price::from("0.001"))
    .size_increment(Quantity::from("0.01"))
    .outcome(Ustr::from("Yes"))
    .description(Ustr::from("Will the outcome of this market be 'Yes'?"))
    .min_quantity(Quantity::from("5"))
    .maker_fee(dec!(0))
    .taker_fee(dec!(0))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
from decimal import Decimal

import pandas as pd

from nautilus_trader.model import AssetClass
from nautilus_trader.model import BinaryOption
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol
from nautilus_trader.model import Venue

raw_symbol = Symbol(
    "0x12a0cb60174abc437bf1178367c72d11f069e1a3add20b148fb0ab4279b772b2-92544998123698303655208967887569360731013655782348975589292031774495159624905",
)
price_increment = Price.from_str("0.001")
size_increment = Quantity.from_str("0.01")

yes_outcome = BinaryOption(
    instrument_id=InstrumentId(raw_symbol, Venue("POLYMARKET")),
    raw_symbol=raw_symbol,
    asset_class=AssetClass.ALTERNATIVE,
    currency=Currency.from_str("USDC"),
    activation_ns=0,
    expiration_ns=pd.Timestamp("2024-01-01", tz="UTC").value,
    price_precision=price_increment.precision,
    size_precision=size_increment.precision,
    price_increment=price_increment,
    size_increment=size_increment,
    min_quantity=Quantity.from_int(5),
    maker_fee=Decimal(0),
    taker_fee=Decimal(0),
    outcome="Yes",
    description="Will the outcome of this market be 'Yes'?",
    ts_event=0,
    ts_init=0,
)
```

## 适配器

创建或使用 `BinaryOption` 的代表性适配器包括：

- [Hyperliquid](../../integrations/hyperliquid.md)，用于二元及预测风格的市场。
- [OKX](../../integrations/okx.md)，用于交易场所定义的二元结果产品。
- [Polymarket](../../integrations/polymarket.md)，用于预测市场结果。

## 相关指南

- [订单簿（Order Book）](../order_book.md) 介绍了二元市场订单簿的行为。
- [数据（Data）](../data/) 说明了引用工具的市场数据。
</content>
