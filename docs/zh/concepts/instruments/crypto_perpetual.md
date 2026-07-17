# 加密货币永续合约（Crypto Perpetual）

`CryptoPerpetual` 表示加密货币永续期货合约，也称为永续互换（perpetual swap）。它没有到期日，跟踪某加密货币基础资产，并以加密货币、稳定币或交易场所定义的其他结算货币进行结算。

示例包括 `ETHUSDT-PERP.BINANCE`、`XBTUSD.BITMEX` 以及 `BTC-USD-SWAP.OKX`。

## 字段

| 字段                   | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|-----------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`       | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`          | `Symbol`           | `Symbol`           | 必需             | 原生交易场所符号。                         |
| `base_currency`       | `Currency`         | `Currency`         | 必需             | 基础加密资产。                             |
| `quote_currency`      | `Currency`         | `Currency`         | 必需             | 价格计价货币。                             |
| `settlement_currency` | `Currency`         | `Currency`         | 必需             | 用于结算盈亏和手续费的货币。               |
| `is_inverse`          | `bool`             | `bool`             | 必需             | 若计价/计数量方式为反向，则为 True。       |
| `price_precision`     | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `size_precision`      | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                   |
| `price_increment`     | `Price`            | `Price`            | 必需             | 最小有效价格步长。                         |
| `size_increment`      | `Quantity`         | `Quantity`         | 必需             | 最小有效数量步长。                         |
| `ts_event`            | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`             | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |
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

*注：Python 构造函数使用 `instrument_id`；Rust 将同一值存储为 `id`。*

## 行为特征

- `CryptoPerpetual` 的资产类别为 `Cryptocurrency`，工具类别为 `Swap`（互换）。
- 它没有生效时间戳或到期时间戳。
- 线性合约通常设置 `is_inverse=False`，并以计价货币结算。
- 反向合约设置 `is_inverse=True`，通常以基础货币结算。
- 量化合约以第三种货币结算，该货币既不同于基础货币，也不同于计价货币。
- 对于反向合约，成本货币为基础货币；对于量化合约，成本货币为结算货币；其余情况下则为计价货币。

:::note
资金费率支付（funding payment）并不是该工具上的字段。它们以数据形式（例如 `FundingRateUpdate`）到达，并引用相应的工具 ID。
:::

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    identifiers::{InstrumentId, Symbol},
    instruments::{CryptoPerpetual, InstrumentAny},
    types::{Currency, Money, Price, Quantity},
};
use rust_decimal_macros::dec;

let ethusdt_perp = CryptoPerpetual::builder()
    .instrument_id(InstrumentId::from("ETHUSDT-PERP.BINANCE"))
    .raw_symbol(Symbol::from("ETHUSDT"))
    .base_currency(Currency::from("ETH"))
    .quote_currency(Currency::from("USDT"))
    .settlement_currency(Currency::from("USDT"))
    .is_inverse(false)
    .price_precision(2)
    .size_precision(3)
    .price_increment(Price::from("0.01"))
    .size_increment(Quantity::from("0.001"))
    .max_quantity(Quantity::from("10000.000"))
    .min_quantity(Quantity::from("0.001"))
    .min_notional(Money::from("10.00 USDT"))
    .max_price(Price::from("15000.00"))
    .min_price(Price::from("1.00"))
    .margin_init(dec!(1.0))
    .margin_maint(dec!(0.35))
    .maker_fee(dec!(0.0002))
    .taker_fee(dec!(0.0004))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();

let instrument = InstrumentAny::CryptoPerpetual(ethusdt_perp);
```

```python tab="Python"
from decimal import Decimal

from nautilus_trader.model import CryptoPerpetual
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Money
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol

ETH = Currency.from_str("ETH")
USDT = Currency.from_str("USDT")

ethusdt_perp = CryptoPerpetual(
    instrument_id=InstrumentId.from_str("ETHUSDT-PERP.BINANCE"),
    raw_symbol=Symbol("ETHUSDT"),
    base_currency=ETH,
    quote_currency=USDT,
    settlement_currency=USDT,
    is_inverse=False,
    price_precision=2,
    size_precision=3,
    price_increment=Price.from_str("0.01"),
    size_increment=Quantity.from_str("0.001"),
    ts_event=0,
    ts_init=0,
    max_quantity=Quantity.from_str("10000.000"),
    min_quantity=Quantity.from_str("0.001"),
    min_notional=Money(10.00, USDT),
    max_price=Price.from_str("15000.00"),
    min_price=Price.from_str("1.00"),
    margin_init=Decimal("1.0"),
    margin_maint=Decimal("0.35"),
    maker_fee=Decimal("0.0002"),
    taker_fee=Decimal("0.0004"),
)
```

## 适配器

创建或使用 `CryptoPerpetual` 的代表性适配器包括：

- [Binance](../../integrations/binance.md)，用于 USD-M 和 COIN-M 永续期货。
- [BitMEX](../../integrations/bitmex.md)，用于反向及线性永续合约。
- [Bybit](../../integrations/bybit.md)，用于线性及反向永续产品。
- [dYdX](../../integrations/dydx.md)，用于永续市场。
- [Hyperliquid](../../integrations/hyperliquid.md)，用于永续市场。
- [Kraken](../../integrations/kraken.md)，用于期货交易场所的永续市场。
- [OKX](../../integrations/okx.md)，用于互换市场。
- [Tardis](../../integrations/tardis.md)，用于加密货币永续合约元数据。

## 相关指南

- [数据（Data）](../data/) 介绍了标记价格（mark price）、指数价格（index price）以及资金费率更新。
- [期权（Options）](../options.md) 介绍了期权特有的工具类型。
- [执行（Execution）](../execution.md) 说明了订单到达交易场所前的精度和名义价值检查。
</content>
