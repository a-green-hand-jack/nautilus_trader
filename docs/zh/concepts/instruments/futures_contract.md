# 期货合约（Futures Contract）

`FuturesContract` 表示一份有到期日、在交易所交易的期货合约，具有明确的标的资产、生效时间、到期时间、货币、乘数和手数规模。

示例包括股指期货、商品期货、利率期货和外汇期货。

## 字段

| 字段               | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`      | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `asset_class`     | `AssetClass`       | `AssetClass`       | 必需             | 标的资产的资产类别。                       |
| `exchange`        | `Option<Ustr>`     | `str \| None`      | `None`           | 已知情况下的交易所 MIC 或交易场所代码。    |
| `underlying`      | `Ustr`             | `str`              | 必需             | 标的资产、指数或产品。                     |
| `activation_ns`   | `UnixNanos`        | `int`              | 必需             | 合约生效时间戳。                           |
| `expiration_ns`   | `UnixNanos`        | `int`              | 必需             | 合约到期时间戳。                           |
| `currency`        | `Currency`         | `Currency`         | 必需             | 报价及结算货币。                           |
| `price_precision` | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `price_increment` | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_precision`  | `u8`               | `int`              | `0`              | 期货以整数份合约交易。                     |
| `size_increment`  | `Quantity`         | `Quantity`         | `1`              | 最小合约数量步长。                         |
| `multiplier`      | `Quantity`         | `Quantity`         | 必需             | 合约乘数。                                 |
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

- `FuturesContract` 的工具类别为 `Future`（期货）。
- 它绝不是反向的。成本货币、结算货币和计价货币均使用 `currency`。
- 它以整数份合约交易，数量精度为 `0`，数量步长为 `1`。
- 对于标的资产和结算货币可能不同的有到期日加密货币期货，请使用 `CryptoFuture`。

## 示例

```rust tab="Rust"
use chrono::{TimeZone, Utc};
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::AssetClass,
    identifiers::{InstrumentId, Symbol},
    instruments::FuturesContract,
    types::{Currency, Price, Quantity},
};
use ustr::Ustr;

let activation = Utc.with_ymd_and_hms(2021, 9, 10, 0, 0, 0).unwrap();
let expiration = Utc.with_ymd_and_hms(2021, 12, 17, 0, 0, 0).unwrap();

let esz21 = FuturesContract::builder()
    .instrument_id(InstrumentId::from("ESZ21.GLBX"))
    .raw_symbol(Symbol::from("ESZ21"))
    .asset_class(AssetClass::Index)
    .exchange(Ustr::from("XCME"))
    .underlying(Ustr::from("ES"))
    .activation_ns(UnixNanos::from(activation.timestamp_nanos_opt().unwrap() as u64))
    .expiration_ns(UnixNanos::from(expiration.timestamp_nanos_opt().unwrap() as u64))
    .currency(Currency::from("USD"))
    .price_precision(2)
    .price_increment(Price::from("0.25"))
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
from nautilus_trader.model import FuturesContract
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

esz21 = FuturesContract(
    instrument_id=InstrumentId.from_str("ESZ21.GLBX"),
    raw_symbol=Symbol("ESZ21"),
    asset_class=AssetClass.INDEX,
    underlying="ES",
    activation_ns=pd.Timestamp("2021-09-10", tz="UTC").value,
    expiration_ns=pd.Timestamp("2021-12-17", tz="UTC").value,
    currency=Currency.from_str("USD"),
    price_precision=2,
    price_increment=Price.from_str("0.25"),
    multiplier=Quantity.from_int(1),
    lot_size=Quantity.from_int(1),
    ts_event=0,
    ts_init=0,
    exchange="XCME",
)
```

## 适配器

创建或使用 `FuturesContract` 的代表性适配器包括：

- [Databento](../../integrations/databento.md)，用于期货参考数据和市场数据。
- [Interactive Brokers](../../integrations/ib.md)，用于上市期货合约。

## 相关指南

- [连续期货（Continuous Futures）](../continuous_futures.md) 介绍了经过换月调整的期货序列。
- [加密货币期货（Crypto Future）](crypto_future.md) 介绍了有到期日的加密货币期货合约。
</content>
