# 永续合约（Perpetual Contract）

`PerpetualContract` 表示适用于各资产类别的通用永续期货合约。当某交易场所提供的永续互换未被专门建模为 `CryptoPerpetual` 时，可使用该类型。

示例包括非加密货币永续合约以及交易场所特定的合成互换（synthetic swap）。

## 字段

| 字段                   | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-----------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`       | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`          | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `underlying`          | `Ustr`             | `str`              | 必需             | 标的资产或参考市场。                       |
| `asset_class`         | `AssetClass`       | `AssetClass`       | 必需             | 标的资产的资产类别。                       |
| `base_currency`       | `Option<Currency>` | `Currency \| None` | `None`           | 基础货币，反向合约时必需。                 |
| `quote_currency`      | `Currency`         | `Currency`         | 必需             | 用于报价的货币。                           |
| `settlement_currency` | `Currency`         | `Currency`         | 必需             | 用于结算盈亏和手续费的货币。               |
| `is_inverse`          | `bool`             | `bool`             | 必需             | 若计价/计数量方式为反向，则为 True。       |
| `price_precision`     | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `size_precision`      | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                   |
| `price_increment`     | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_increment`      | `Quantity`         | `Quantity`         | 必需             | 最小有效数量步长。                         |
| `multiplier`          | `Quantity`         | `Quantity`         | `1`              | 合约乘数。                                 |
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

- `PerpetualContract` 的工具类别为 `Swap`（互换）。
- 它没有生效时间戳或到期时间戳。
- 反向合约要求提供基础货币。
- 线性合约通常以计价货币结算。
- 对于基础资产为某种货币的加密货币永续合约，请使用 `CryptoPerpetual`。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::AssetClass,
    identifiers::{InstrumentId, Symbol},
    instruments::PerpetualContract,
    types::{Currency, Price, Quantity},
};
use rust_decimal_macros::dec;
use ustr::Ustr;

let eurusd_perp = PerpetualContract::builder()
    .instrument_id(InstrumentId::from("EURUSD-PERP.AX"))
    .raw_symbol(Symbol::from("EURUSD-PERP"))
    .underlying(Ustr::from("EURUSD"))
    .asset_class(AssetClass::FX)
    .base_currency(Currency::from("EUR"))
    .quote_currency(Currency::from("USD"))
    .settlement_currency(Currency::from("USD"))
    .is_inverse(false)
    .price_precision(5)
    .size_precision(0)
    .price_increment(Price::from("0.00001"))
    .size_increment(Quantity::from("1"))
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
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import PerpetualContract
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

eurusd_perp = PerpetualContract(
    instrument_id=InstrumentId.from_str("EURUSD-PERP.AX"),
    raw_symbol=Symbol("EURUSD-PERP"),
    underlying="EURUSD",
    asset_class=AssetClass.FX,
    quote_currency=Currency.from_str("USD"),
    settlement_currency=Currency.from_str("USD"),
    is_inverse=False,
    price_precision=5,
    size_precision=0,
    price_increment=Price.from_str("0.00001"),
    size_increment=Quantity.from_int(1),
    ts_event=0,
    ts_init=0,
    base_currency=Currency.from_str("EUR"),
    margin_init=Decimal("0.03"),
    margin_maint=Decimal("0.03"),
    maker_fee=Decimal("0.00002"),
    taker_fee=Decimal("0.00002"),
)
```

## 适配器

创建或使用 `PerpetualContract` 的代表性适配器包括：

- [Architect AX](../../integrations/architect_ax.md)，用于交易场所定义的永续合约。

## 相关指南

- [加密货币永续合约（Crypto Perpetual）](crypto_perpetual.md) 介绍了加密货币永续期货。
- [数据（Data）](../data/) 介绍了标记价格、指数价格以及资金费率更新。
</content>
