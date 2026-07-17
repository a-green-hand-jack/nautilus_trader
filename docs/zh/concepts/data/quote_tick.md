# QuoteTick

`QuoteTick` 表示某个金融工具的盘口顶部（top-of-book）买卖报价。它携带某一
特定事件时间点上的最优买价和买量，以及最优卖价和卖量。

## 字段

| 字段           | Rust 类型      | Python 类型    | 是否必需/默认值 | 说明                                    |
|-----------------|----------------|----------------|------------------|------------------------------------------|
| `instrument_id` | `InstrumentId` | `InstrumentId` | 必需             | 该报价所对应的金融工具。                |
| `bid_price`     | `Price`        | `Price`        | 必需             | 最优买价。                          |
| `ask_price`     | `Price`        | `Price`        | 必需             | 最优卖价。                          |
| `bid_size`      | `Quantity`     | `Quantity`     | 必需             | 最优买价上可成交的数量。      |
| `ask_size`      | `Quantity`     | `Quantity`     | 必需             | 最优卖价上可成交的数量。      |
| `ts_event`      | `UnixNanos`    | `int`          | 必需             | 事件时间戳（纳秒）。          |
| `ts_init`       | `UnixNanos`    | `int`          | 必需             | 初始化时间戳（纳秒）。 |

## 行为

- 买价和卖价必须使用相同的精度。
- 买量和卖量必须使用相同的精度。
- `extract_price(PriceType.BID | ASK | MID)` 返回所请求的价格基准。
- 报价 K 线（quote bar）可以使用 `BID`、`ASK` 或 `MID` 价格类型。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::QuoteTick,
    identifiers::InstrumentId,
    types::{Price, Quantity},
};

let quote = QuoteTick::new(
    InstrumentId::from("AUD/USD.SIM"),
    Price::from("0.65000"),
    Price::from("0.65002"),
    Quantity::from("1000000"),
    Quantity::from("1200000"),
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model import QuoteTick

quote = QuoteTick(
    instrument_id=InstrumentId.from_str("AUD/USD.SIM"),
    bid_price=Price.from_str("0.65000"),
    ask_price=Price.from_str("0.65002"),
    bid_size=Quantity.from_int(1_000_000),
    ask_size=Quantity.from_int(1_200_000),
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [OrderBookDepth10](order_book_depth10.md) 介绍了包含盘口顶部档位的固定深度快照。
- [K 线与聚合](index.md#bars-and-aggregation) 介绍了报价到 K 线的聚合方式。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
