# 博彩工具（Betting Instrument）

`BettingInstrument` 表示体育或博彩市场中的一个选项（selection）。它携带赛事（event）、赛事系列（competition）、市场（market）以及选项（selection）的元数据，使 Nautilus 能够将该选项当作一个带有价格、数量、限制、保证金和手续费的工具来处理。

示例包括 Betfair 的赛事赔率（match-odds）选项以及让分市场（handicap market）选项。

## 字段

| 字段                  | Rust 类型           | Python 类型         | 是否必需/默认值 | 说明                                       |
|----------------------|--------------------|--------------------|------------------|------------------------------------------|
| `instrument_id`      | `InstrumentId`     | `InstrumentId`     | 必需             | 在 Rust 中存储为 `id`。                   |
| `raw_symbol`         | `Symbol`           | `Symbol`           | 必需             | 原生或生成的交易场所符号。                 |
| `event_type_id`      | `u64`              | `int`              | 必需             | 赛事类型标识符。                           |
| `event_type_name`    | `Ustr`             | `str`              | 必需             | 赛事类型名称，例如某项运动。               |
| `competition_id`     | `u64`              | `int`              | 必需             | 赛事系列标识符。                           |
| `competition_name`   | `Ustr`             | `str`              | 必需             | 赛事系列名称。                             |
| `event_id`           | `u64`              | `int`              | 必需             | 赛事标识符。                               |
| `event_name`         | `Ustr`             | `str`              | 必需             | 赛事名称。                                 |
| `event_country_code` | `Ustr`             | `str`              | 必需             | 赛事所属国家代码。                         |
| `event_open_date`    | `UnixNanos`        | `int`              | 必需             | 赛事开盘时间。                             |
| `betting_type`       | `Ustr`             | `str`              | 必需             | 交易场所发布的投注类型。                   |
| `market_id`          | `Ustr`             | `str`              | 必需             | 市场标识符。                               |
| `market_name`        | `Ustr`             | `str`              | 必需             | 市场名称。                                 |
| `market_type`        | `Ustr`             | `str`              | 必需             | 市场类型，例如赛事赔率。                   |
| `market_start_time`  | `UnixNanos`        | `int`              | 必需             | 市场开始时间。                             |
| `selection_id`       | `u64`              | `int`              | 必需             | 选项或参赛方标识符。                       |
| `selection_name`     | `Ustr`             | `str`              | 必需             | 选项或参赛方名称。                         |
| `selection_handicap` | `f64`              | `float`            | 必需             | 让分市场对应的让分值。                     |
| `currency`           | `Currency`         | `Currency`         | 必需             | 报价及结算货币。                           |
| `price_precision`    | `u8`               | `int`              | 必需             | 价格允许的小数位数。                       |
| `size_precision`     | `u8`               | `int`              | 必需             | 订单数量允许的小数位数。                   |
| `price_increment`    | `Price`            | `Price`            | 必需             | 价格步长，通常由 tick scheme 设定。        |
| `size_increment`     | `Quantity`         | `Quantity`         | 必需             | 最小数量步长。                             |
| `max_quantity`       | `Option<Quantity>` | `Quantity \| None` | `None`           | 最大订单数量。                             |
| `min_quantity`       | `Option<Quantity>` | `Quantity \| None` | `None`           | 最小订单数量。                             |
| `max_notional`       | `Option<Money>`    | `Money \| None`    | `None`           | 最大订单名义价值。                         |
| `min_notional`       | `Option<Money>`    | `Money \| None`    | `None`           | 最小订单名义价值。                         |
| `max_price`          | `Option<Price>`    | `Price \| None`    | `None`           | 最大有效报价或订单价格。                   |
| `min_price`          | `Option<Price>`    | `Price \| None`    | `None`           | 最小有效报价或订单价格。                   |
| `margin_init`        | `Option<Decimal>`  | `Decimal \| None`  | `1`              | 初始保证金率。                             |
| `margin_maint`       | `Option<Decimal>`  | `Decimal \| None`  | `1`              | 维持保证金率。                             |
| `maker_fee`          | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 挂单方费率。负值表示返佣。                 |
| `taker_fee`          | `Option<Decimal>`  | `Decimal \| None`  | `0`              | 吃单方费率。负值表示返佣。                 |
| `tick_scheme`        | `Option<Ustr>`     | `str \| None`      | `None`           | 所注册的可变最小报价单位方案名称。         |
| `info`               | `Option<Params>`   | `dict \| None`     | `None`           | 适配器元数据。                             |
| `ts_event`           | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的事件时间戳。                 |
| `ts_init`            | `UnixNanos`        | `int`              | 必需             | 以纳秒为单位的初始化时间戳。               |

*注：Python 会根据交易场所、市场、选项以及让分等字段自动构建工具 ID 和原生符号。Rust 中则直接以 `instrument_id` 和 `raw_symbol` 传入。*

## 行为特征

- `BettingInstrument` 的资产类别为 `Alternative`（另类资产），工具类别为 `SportsBetting`（体育博彩）。
- 每个选项或参赛方都被建模为独立的工具。
- 博彩工具通常使用已注册的 tick scheme 来定义有效的赔率步长。
- 保证金默认设为 1，因为下注通常需要预留全部本金。

## 示例

```rust tab="Rust"
use chrono::{TimeZone, Utc};
use nautilus_core::UnixNanos;
use nautilus_model::{
    identifiers::{InstrumentId, Symbol},
    instruments::BettingInstrument,
    types::{Currency, Money, Price, Quantity},
};
use rust_decimal_macros::dec;
use ustr::Ustr;

let event_open = Utc.with_ymd_and_hms(2022, 2, 7, 23, 30, 0).unwrap();
let market_start = Utc.with_ymd_and_hms(2022, 2, 7, 23, 30, 0).unwrap();

let selection = BettingInstrument::builder()
    .instrument_id(InstrumentId::from("1-123456789.BETFAIR"))
    .raw_symbol(Symbol::from("1-123456789"))
    .event_type_id(6423)
    .event_type_name(Ustr::from("American Football"))
    .competition_id(12_282_733)
    .competition_name(Ustr::from("NFL"))
    .event_id(29_678_534)
    .event_name(Ustr::from("NFL"))
    .event_country_code(Ustr::from("GB"))
    .event_open_date(UnixNanos::from(event_open.timestamp_nanos_opt().unwrap() as u64))
    .betting_type(Ustr::from("ODDS"))
    .market_id(Ustr::from("1-123456789"))
    .market_name(Ustr::from("AFC Conference Winner"))
    .market_type(Ustr::from("SPECIAL"))
    .market_start_time(UnixNanos::from(market_start.timestamp_nanos_opt().unwrap() as u64))
    .selection_id(50214)
    .selection_name(Ustr::from("Kansas City Chiefs"))
    .selection_handicap(0.0)
    .currency(Currency::from("GBP"))
    .price_precision(2)
    .size_precision(2)
    .price_increment(Price::from("0.01"))
    .size_increment(Quantity::from("0.01"))
    .max_quantity(Quantity::from("1000"))
    .min_quantity(Quantity::from("1"))
    .max_notional(Money::from("10000 GBP"))
    .min_notional(Money::from("10 GBP"))
    .max_price(Price::from("100.00"))
    .min_price(Price::from("1.00"))
    .margin_init(dec!(1))
    .margin_maint(dec!(1))
    .maker_fee(dec!(0))
    .taker_fee(dec!(0))
    .ts_event(UnixNanos::default())
    .ts_init(UnixNanos::default())
    .build()
    .unwrap();
```

```python tab="Python"
import pandas as pd

from nautilus_trader.model import BettingInstrument
from nautilus_trader.model import Currency
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Money
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import Symbol
from nautilus_trader.model import Venue

GBP = Currency.from_str("GBP")

selection = BettingInstrument(
    instrument_id=InstrumentId(Symbol("1-123456789-50214"), Venue("BETFAIR")),
    raw_symbol=Symbol("1-123456789-50214"),
    event_type_id=6423,
    event_type_name="American Football",
    competition_id=12282733,
    competition_name="NFL",
    event_id=29678534,
    event_name="NFL",
    event_country_code="GB",
    event_open_date=pd.Timestamp("2022-02-07 23:30:00+00:00").value,
    betting_type="ODDS",
    market_id="1-123456789",
    market_name="AFC Conference Winner",
    market_type="SPECIAL",
    market_start_time=pd.Timestamp("2022-02-07 23:30:00+00:00").value,
    selection_id=50214,
    selection_name="Kansas City Chiefs",
    selection_handicap=0.0,
    currency=GBP,
    price_precision=2,
    size_precision=2,
    price_increment=Price.from_str("0.01"),
    size_increment=Quantity.from_str("0.01"),
    min_notional=Money(1, GBP),
    ts_event=0,
    ts_init=0,
)
```

## 适配器

创建或使用 `BettingInstrument` 的代表性适配器包括：

- [Betfair](../../integrations/betfair.md)，用于体育博彩市场。
- [Betfair v2](../../integrations/betfair_v2.md)，用于体育博彩市场。

## 相关指南

- [账务处理（Accounting）](../accounting.md) 介绍了博彩账户的行为。
- [数据（Data）](../data/) 说明了引用工具的市场数据。
</content>
