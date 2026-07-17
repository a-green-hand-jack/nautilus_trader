# TradeTick

`TradeTick` 表示交易场所发生的一次已执行成交或撮合事件。它携带成交价格、
成交数量、主动方（aggressor side）以及交易场所的成交标识符。

## 字段

| 字段            | Rust 类型       | Python 类型     | 是否必需/默认值 | 说明                                    |
|------------------|-----------------|-----------------|------------------|------------------------------------------|
| `instrument_id`  | `InstrumentId`  | `InstrumentId`  | 必需             | 该成交所对应的金融工具。                |
| `price`          | `Price`         | `Price`         | 必需             | 成交价格。                          |
| `size`           | `Quantity`      | `Quantity`      | 必需             | 成交数量。                       |
| `aggressor_side` | `AggressorSide` | `AggressorSide` | 必需             | 买方主动、卖方主动，或无主动方。          |
| `trade_id`       | `TradeId`       | `TradeId`       | 必需             | 交易场所分配的撮合 ID。                 |
| `ts_event`       | `UnixNanos`     | `int`           | 必需             | 事件时间戳（纳秒）。          |
| `ts_init`        | `UnixNanos`     | `int`           | 必需             | 初始化时间戳（纳秒）。 |

## 行为

- `size` 必须为正数。
- 信息驱动型 K 线（information-driven bar）需要 `TradeTick` 数据，因为它们要
  使用 `aggressor_side`。
- 成交 K 线（trade bar）使用 `LAST` 价格类型。
- 当交易场所提供了成交 ID 时，`trade_id` 对于同一交易场所事件应保持稳定不变。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::TradeTick,
    enums::AggressorSide,
    identifiers::{InstrumentId, TradeId},
    types::{Price, Quantity},
};

let trade = TradeTick::new(
    InstrumentId::from("BTCUSDT.BINANCE"),
    Price::from("65000.10"),
    Quantity::from("0.25"),
    AggressorSide::Buyer,
    TradeId::from("123456789"),
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import TradeId
from nautilus_trader.model import TradeTick
from nautilus_trader.model.enums import AggressorSide

trade = TradeTick(
    instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
    price=Price.from_str("65000.10"),
    size=Quantity.from_str("0.25"),
    aggressor_side=AggressorSide.BUYER,
    trade_id=TradeId("123456789"),
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [Bar](bar.md) 介绍了成交到 K 线的聚合方式。
- [信息驱动型 K 线](index.md#information-driven-bars) 解释了主动方的使用方式。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
