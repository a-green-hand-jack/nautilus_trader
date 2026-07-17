# 期权价差（Option Spread）

`OptionSpread` 表示交易所定义的、包含多条腿的期权策略。交易场所将该策略作为一个具有自身符号、最小报价单位、到期日和执行规则的单一工具发布。

示例包括上市的垂直价差（vertical spread）、日历价差以及其他期权策略。

## 字段

| 字段               | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`      | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `asset_class`     | `AssetClass`       | `AssetClass`       | 必需             | 标的策略的资产类别。                       |
| `exchange`        | `Option<Ustr>`     | `str \| None`      | `None`           | 已知情况下的交易所 MIC 或交易场所代码。    |
| `underlying`      | `Ustr`             | `str`              | 必需             | 标的资产、期货或指数。                     |
| `strategy_type`   | `Ustr`             | `str`              | 必需             | 交易场所策略类型，例如垂直价差。           |
| `activation_ns`   | `UnixNanos`        | `int`              | 必需             | 策略生效时间戳。                           |
| `expiration_ns`   | `UnixNanos`        | `int`              | 必需             | 策略到期时间戳。                           |
| `currency`        | `Currency`         | `Currency`         | 必需             | 权利金报价及结算货币。                     |
| `price_precision` | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `price_increment` | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_precision`  | `u8`               | `int`              | `0`              | 期权价差以整数份合约交易。                 |
| `size_increment`  | `Quantity`         | `Quantity`         | `1`              | 最小合约数量步长。                         |
| `multiplier`      | `Quantity`         | `Quantity`         | 必需             | 策略乘数。                                 |
| `lot_size`        | `Quantity`         | `Quantity`         | 必需             | 取整后的手数或合约手规模。                 |
| `margin_init`     | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 初始保证金率。                             |
| `margin_maint`    | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 维持保证金率。                             |
| `maker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 挂单方费率。负值表示返佣。                 |
| `taker_fee`       | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 吃单方费率。负值表示返佣。                 |
| `max_quantity`    | `Option<Quantity>` | `Quantity \| None` | `None`           | 最大订单数量。                             |
| `min_quantity`    | `Option<Quantity>` | `Quantity \| None` | `1`              | 最小订单数量。                             |
| `max_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最大有效报价或订单价格。                   |
| `min_price`       | `Option<Price>`    | `Price \| None`    | `None`           | 最小有效报价或订单价格。                   |
| `tick_scheme`     | `Option<Ustr>`     | `str \| None`      | `None`           | 所注册的可变最小报价单位方案名称。         |
| `info`            | `Option<Params>`   | `dict \| None`     | `None`           | 适配器元数据。                             |
| `ts_event`        | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`         | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `OptionSpread` 的工具类别为 `OptionSpread`。
- 交易场所将该价差发布为单一可交易工具。
- 它以整数份合约交易，数量精度为 `0`，数量步长为 `1`。
- 若适配器提供了交易场所特定的分腿细节，将其存储在 `info` 字段中。

## 示例

```rust tab="Rust"
use chrono::{TimeZone, Utc};
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::AssetClass,
    identifiers::{InstrumentId, Symbol},
    instruments::OptionSpread,
    types::{Currency, Price, Quantity},
};
use ustr::Ustr;

let activation = Utc.with_ymd_and_hms(2023, 11, 6, 20, 54, 7).unwrap();
let expiration = Utc.with_ymd_and_hms(2024, 2, 23, 22, 59, 0).unwrap();

let sr3_spread = OptionSpread::builder()
    .instrument_id(InstrumentId::from("UD:U$: GN 2534559.GLBX"))
    .raw_symbol(Symbol::from("UD:U$: GN 2534559"))
    .asset_class(AssetClass::FX)
    .exchange(Ustr::from("XCME"))
    .underlying(Ustr::from("SR3"))
    .strategy_type(Ustr::from("GN"))
    .activation_ns(UnixNanos::from(activation.timestamp_nanos_opt().unwrap() as u64))
    .expiration_ns(UnixNanos::from(expiration.timestamp_nanos_opt().unwrap() as u64))
    .currency(Currency::from("USD"))
    .price_precision(2)
    .price_increment(Price::from("0.01"))
    .multiplier(Quantity::from("1"))
    .lot_size(Quantity::from("1"))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
import pandas as pd

from nautilus_trader.model import AssetClass
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import OptionSpread
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

sr3_spread = OptionSpread(
    instrument_id=InstrumentId.from_str("UD:U$: GN 2534559.GLBX"),
    raw_symbol=Symbol("UD:U$: GN 2534559"),
    asset_class=AssetClass.FX,
    underlying="SR3",
    strategy_type="GN",
    activation_ns=pd.Timestamp("2023-11-06T20:54:07", tz="UTC").value,
    expiration_ns=pd.Timestamp("2024-02-23T22:59:00", tz="UTC").value,
    currency=Currency.from_str("USD"),
    price_precision=2,
    price_increment=Price.from_str("0.01"),
    multiplier=Quantity.from_int(1),
    lot_size=Quantity.from_int(1),
    ts_event=0,
    ts_init=0,
    exchange="XCME",
)
```

## 适配器

创建或使用 `OptionSpread` 的代表性适配器包括：

- [Databento](../../integrations/databento.md)，用于上市期权价差市场。
- [Interactive Brokers](../../integrations/ib.md)，用于交易所定义的期权策略。

## 相关指南

- [期权合约（Option Contract）](option_contract.md) 介绍了单腿期权合约。
- [期权（Options）](../options.md) 介绍了期权数据、希腊字母以及期权链订阅。
</content>
