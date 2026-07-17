# 商品（Commodity）

`Commodity` 表示黄金、白银、原油或其他以某种货币计价的实物资产的现货市场。它建模的是现货市场，而不是有到期日的期货合约。

示例包括 `XAUUSD.IDEALPRO` 以及交易场所特定的商品现金符号。

## 字段

| 字段              | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`      | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `asset_class`     | `AssetClass`       | `AssetClass`       | 必需             | 商品资产分类。                             |
| `quote_currency`  | `Currency`         | `Currency`         | 必需             | 用于为该商品定价的货币。                   |
| `price_precision` | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `size_precision`  | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                   |
| `price_increment` | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_increment`  | `Quantity`         | `Quantity`         | 必需             | 最小有效数量步长。                         |
| `ts_event`        | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`         | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |
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

- `Commodity` 的工具类别为 `Spot`（现货）。
- 它允许出现负价格：电力或原油等现货市场可能在零以下交易，`RiskEngine` 在订单提交和修改时都接受负价格。
- 它绝不是反向的，其成本货币为计价货币。
- 它没有生效时间戳、到期日、行权价、期权类型或结算货币字段。
- 对于有到期日的交易所上市商品期货，请使用 `FuturesContract`。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::AssetClass,
    identifiers::{InstrumentId, Symbol},
    instruments::Commodity,
    types::{Currency, Price, Quantity},
};

let gold = Commodity::builder()
    .instrument_id(InstrumentId::from("GOLD.COMEX"))
    .raw_symbol(Symbol::from("GOLD"))
    .asset_class(AssetClass::Commodity)
    .quote_currency(Currency::from("USD"))
    .price_precision(2)
    .size_precision(0)
    .price_increment(Price::from("0.01"))
    .size_increment(Quantity::from("1"))
    .lot_size(Quantity::from("1"))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
from nautilus_trader.model import AssetClass
from nautilus_trader.model import Commodity
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

gold = Commodity(
    instrument_id=InstrumentId.from_str("GOLD.COMEX"),
    raw_symbol=Symbol("GOLD"),
    asset_class=AssetClass.COMMODITY,
    quote_currency=Currency.from_str("USD"),
    price_precision=2,
    price_increment=Price.from_str("0.01"),
    size_precision=0,
    size_increment=Quantity.from_int(1),
    ts_event=0,
    ts_init=0,
    lot_size=Quantity.from_int(1),
)
```

## 适配器

创建或使用 `Commodity` 的代表性适配器包括：

- [Interactive Brokers](../../integrations/ib.md)，用于现货商品和金属合约。

## 相关指南

- [期货合约（Futures Contract）](futures_contract.md) 介绍了以商品为标的的有到期日期货。
- [数据（Data）](../data/) 说明了引用工具的市场数据。
</content>
