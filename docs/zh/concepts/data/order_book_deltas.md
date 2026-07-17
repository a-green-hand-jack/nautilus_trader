# OrderBookDeltas

`OrderBookDeltas` 将一批非空的 `OrderBookDelta` 记录归为一组，这些记录属于同一
逻辑订单簿事件。它能够在适配器（adapter）一次性接收或产生多个订单簿变化时，
降低每条消息的处理开销。

## 字段

| 字段           | Rust 类型             | Python 类型            | 是否必需/默认值 | 说明                                     |
|-----------------|-----------------------|------------------------|------------------|--------------------------------------------|
| `instrument_id` | `InstrumentId`        | `InstrumentId`         | 必需             | 订单簿发生变化的金融工具。        |
| `deltas`        | `Vec<OrderBookDelta>` | `list[OrderBookDelta]` | 必需             | 非空的 delta 批次。                |
| `flags`         | `u8`                  | `int`                  | 取自最后一个 delta  | 最后一个 delta 的标志位。                          |
| `sequence`      | `u64`                 | `int`                  | 取自最后一个 delta  | 最后一个 delta 的序列号。               |
| `ts_event`      | `UnixNanos`           | `int`                  | 取自最后一个 delta  | 最后一个 delta 的事件时间戳。               |
| `ts_init`       | `UnixNanos`           | `int`                  | 取自最后一个 delta  | 最后一个 delta 的初始化时间戳。      |

## 行为

- 该批次必须至少包含一个 delta。
- 批次的元数据与最后一个 delta 保持一致。
- 若最后一个 delta 结束了一组逻辑事件，则应携带 `F_LAST` 标志位。
- 快照批次通常以 `CLEAR` 类型的 delta 开始，并以携带 `F_SNAPSHOT | F_LAST`
  标志位的 delta 结束。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::{BookOrder, OrderBookDelta, OrderBookDeltas},
    enums::{BookAction, OrderSide, RecordFlag},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};

let instrument_id = InstrumentId::from("ETHUSDT-PERP.BINANCE");
let bid = OrderBookDelta::new(
    instrument_id,
    BookAction::Add,
    BookOrder::new(OrderSide::Buy, Price::from("2500.10"), Quantity::from("3.5"), 1),
    0,
    41,
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
let ask = OrderBookDelta::new(
    instrument_id,
    BookAction::Add,
    BookOrder::new(OrderSide::Sell, Price::from("2500.20"), Quantity::from("2.0"), 2),
    RecordFlag::F_LAST as u8,
    42,
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);

let deltas = OrderBookDeltas::new(instrument_id, vec![bid, ask]);
```

```python tab="Python"
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model.data import BookOrder
from nautilus_trader.model.data import OrderBookDelta
from nautilus_trader.model.data import OrderBookDeltas
from nautilus_trader.model.enums import BookAction
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import RecordFlag

instrument_id = InstrumentId.from_str("ETHUSDT-PERP.BINANCE")
bid = OrderBookDelta(
    instrument_id=instrument_id,
    action=BookAction.ADD,
    order=BookOrder(
        OrderSide.BUY,
        Price.from_str("2500.10"),
        Quantity.from_str("3.5"),
        1,
    ),
    flags=0,
    sequence=41,
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
ask = OrderBookDelta(
    instrument_id=instrument_id,
    action=BookAction.ADD,
    order=BookOrder(
        OrderSide.SELL,
        Price.from_str("2500.20"),
        Quantity.from_str("2.0"),
        2,
    ),
    flags=RecordFlag.F_LAST,
    sequence=42,
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)

deltas = OrderBookDeltas(instrument_id, [bid, ask])
```

## 相关指南

- [OrderBookDelta](order_book_delta.md) 介绍了其中包含的更新类型。
- [Order books](../order_book.md) 解释了支持的订单簿状态。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
