# OrderBookDepth10

`OrderBookDepth10` 表示一个固定深度的订单簿更新，最多包含 10 个买盘（bid）
档位和 10 个卖盘（ask）档位。当交易场所发布自包含的深度快照（而非增量 delta）
时，该类型十分有用。

## 字段

| 字段           | Rust 类型          | Python 类型       | 是否必需/默认值 | 说明                                      |
|-----------------|--------------------|-------------------|------------------|--------------------------------------------|
| `instrument_id` | `InstrumentId`     | `InstrumentId`    | 必需             | 该订单簿所对应的金融工具。      |
| `bids`          | `[BookOrder; 10]`  | `list[BookOrder]` | 必需             | 恰好 10 个买盘档位。                     |
| `asks`          | `[BookOrder; 10]`  | `list[BookOrder]` | 必需             | 恰好 10 个卖盘档位。                     |
| `bid_counts`    | `[u32; 10]`        | `list[int]`       | 必需             | 每个买盘档位的订单数量。        |
| `ask_counts`    | `[u32; 10]`        | `list[int]`       | 必需             | 每个卖盘档位的订单数量。        |
| `flags`         | `u8`               | `int`             | 必需             | 用于事件元数据的 `RecordFlag` 位字段。 |
| `sequence`      | `u64`              | `int`             | 必需             | 交易场所序列号，若无则为零。  |
| `ts_event`      | `UnixNanos`        | `int`             | 必需             | 事件时间戳（纳秒）。            |
| `ts_init`       | `UnixNanos`        | `int`             | 必需             | 初始化时间戳（纳秒）。   |

## 行为

- Rust 和 PyO3 Python 构造函数要求恰好提供 10 个买盘档位、10 个卖盘档位、
  10 个买盘数量以及 10 个卖盘数量。
- 对于不可用的档位，使用空/默认订单并将数量置为零。
- 该类型不能与增量的 `OrderBookDelta` 数据流互换使用。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::{BookOrder, OrderBookDepth10, DEPTH10_LEN},
    enums::OrderSide,
    identifiers::InstrumentId,
    types::{Price, Quantity},
};

let mut bids = [BookOrder::default(); DEPTH10_LEN];
let mut asks = [BookOrder::default(); DEPTH10_LEN];
bids[0] = BookOrder::new(OrderSide::Buy, Price::from("2500.10"), Quantity::from("3.5"), 1);
asks[0] = BookOrder::new(OrderSide::Sell, Price::from("2500.20"), Quantity::from("2.0"), 2);

let depth = OrderBookDepth10::new(
    InstrumentId::from("ETHUSDT-PERP.BINANCE"),
    bids,
    asks,
    [1; DEPTH10_LEN],
    [1; DEPTH10_LEN],
    0,
    42,
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model.data import BookOrder
from nautilus_trader.model.data import OrderBookDepth10
from nautilus_trader.model.enums import OrderSide

bids = [
    BookOrder(
        OrderSide.BUY,
        Price.from_str(f"{2500.10 - i * 0.10:.2f}"),
        Quantity.from_str("3.5"),
        i + 1,
    )
    for i in range(10)
]
asks = [
    BookOrder(
        OrderSide.SELL,
        Price.from_str(f"{2500.20 + i * 0.10:.2f}"),
        Quantity.from_str("2.0"),
        i + 11,
    )
    for i in range(10)
]

depth = OrderBookDepth10(
    instrument_id=InstrumentId.from_str("ETHUSDT-PERP.BINANCE"),
    bids=bids,
    asks=asks,
    bid_counts=[1] * 10,
    ask_counts=[1] * 10,
    flags=0,
    sequence=42,
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [QuoteTick](quote_tick.md) 介绍了由深度数据派生的盘口顶部数据。
- [Order books](index.md#order-books) 解释了订单簿状态。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
