# OrderBookDelta

`OrderBookDelta` 表示订单簿（order book）的一次变化，是 Nautilus 内置数据类型中
粒度最细的订单簿数据类型，支持 Nautilus 用于增量订单簿更新的以下几种订单簿类型：

- `L3_MBO`：三级逐单（market-by-order，MBO）数据。
- `L2_MBP`：二级逐价（market-by-price，MBP）数据。
- `L1_MBP`：一级逐价（market-by-price，MBP）盘口顶部（top-of-book）数据。

源数据流和目标 `BookType` 决定了某个 delta 携带的粒度。

当交易场所或数据提供商发布增量订单簿变化，且 Nautilus 需要在本地维护订单簿状态时，
使用该数据类型。

## 字段

| 字段           | Rust 类型      | Python 类型        | 是否必需/默认值 | 说明                                      |
|-----------------|----------------|--------------------|------------------|--------------------------------------------|
| `instrument_id` | `InstrumentId` | `InstrumentId`     | 必需             | 订单簿发生变化的金融工具。         |
| `action`        | `BookAction`   | `BookAction`       | 必需             | `ADD`、`UPDATE`、`DELETE` 或 `CLEAR`。     |
| `order`         | `BookOrder`    | `BookOrder`        | 必需             | 价格、数量、方向以及订单 ID 等负载信息。   |
| `flags`         | `u8`           | `int`              | 必需             | 用于事件元数据的 `RecordFlag` 位字段。 |
| `sequence`      | `u64`          | `int`              | 必需             | 交易场所序列号，若无则为零。  |
| `ts_event`      | `UnixNanos`    | `int`              | 必需             | 事件时间戳（纳秒）。            |
| `ts_init`       | `UnixNanos`    | `int`              | 必需             | 初始化时间戳（纳秒）。   |

## BookOrder 字段

`order` 字段包含该 delta 的 `BookOrder` 负载。

| 字段      | Rust 类型         | Python 类型 | 说明                                |
|------------|-------------------|-------------|--------------------------------------|
| `side`     | `OrderSide`       | `OrderSide` | 订单方向。                          |
| `price`    | `Price`           | `Price`     | 订单价格。                         |
| `size`     | `Quantity`        | `Quantity`  | 订单数量。                         |
| `order_id` | `OrderId` (`u64`) | `int`       | 源数据流携带的订单 ID。 |

空/默认订单使用 `NO_ORDER_SIDE`、零价格、零数量以及 `order_id` 为零。

## BookAction 取值

| Rust 变体             | Python 变体 | 值 | 含义                                |
|----------------------|----------------|-------|----------------------------------------|
| `BookAction::Add`    | `ADD`          | `1`   | 向订单簿添加一个订单。             |
| `BookAction::Update` | `UPDATE`       | `2`   | 更新订单簿中已存在的订单。 |
| `BookAction::Delete` | `DELETE`       | `3`   | 删除订单簿中已存在的订单。 |
| `BookAction::Clear`  | `CLEAR`        | `4`   | 清空订单簿状态。           |

## 行为

- `ADD` 和 `UPDATE` 类型的 delta 要求订单数量为正数。
- `CLEAR` 类型的 delta 会重置订单簿状态，并使用空订单。
- `flags` 携带事件边界和快照相关的元数据。参见
  [Delta 标志位与事件边界](index.md#delta-flags-and-event-boundaries)。
- Rust 提供 `OrderBookDelta::clear(...)`；Python 用户可以使用 `BookAction.CLEAR`
  以及空订单或默认订单来构造清空类型的 delta。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::{BookOrder, OrderBookDelta},
    enums::{BookAction, OrderSide, RecordFlag},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};

let delta = OrderBookDelta::new(
    InstrumentId::from("ETHUSDT-PERP.BINANCE"),
    BookAction::Add,
    BookOrder::new(
        OrderSide::Buy,
        Price::from("2500.10"),
        Quantity::from("3.5"),
        12_345,
    ),
    RecordFlag::F_LAST as u8,
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
from nautilus_trader.model.data import OrderBookDelta
from nautilus_trader.model.enums import BookAction
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import RecordFlag

delta = OrderBookDelta(
    instrument_id=InstrumentId.from_str("ETHUSDT-PERP.BINANCE"),
    action=BookAction.ADD,
    order=BookOrder(
        OrderSide.BUY,
        Price.from_str("2500.10"),
        Quantity.from_str("3.5"),
        12_345,
    ),
    flags=RecordFlag.F_LAST,
    sequence=42,
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [OrderBookDeltas](order_book_deltas.md) 介绍了批量处理 delta。
- [Order books](../order_book.md) 解释了订单簿类型和本地订单簿状态。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
