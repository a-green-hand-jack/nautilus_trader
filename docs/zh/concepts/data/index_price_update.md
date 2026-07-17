# IndexPriceUpdate

`IndexPriceUpdate` 表示衍生品市场使用的外部指数价格。交易场所通常会基于指数价格
推导标记价格（mark price）、资金费率计算或结算行为。

## 字段

| 字段           | Rust 类型      | Python 类型    | 是否必需/默认值 | 说明                                    |
|-----------------|----------------|----------------|------------------|------------------------------------------|
| `instrument_id` | `InstrumentId` | `InstrumentId` | 必需             | 该指数价格对应的金融工具。          |
| `value`         | `Price`        | `Price`        | 必需             | 当前指数价格。                     |
| `ts_event`      | `UnixNanos`    | `int`          | 必需             | 事件时间戳（纳秒）。          |
| `ts_init`       | `UnixNanos`    | `int`          | 必需             | 初始化时间戳（纳秒）。 |

## 行为

- 指数价格属于参考数据，并不表示已经发生了成交。
- 永续合约和期货交易场所可能同时发布标记价格和指数价格。
- 目录（catalog）在存储指数价格时会同时保存金融工具 ID 和价格精度元数据。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::IndexPriceUpdate,
    identifiers::InstrumentId,
    types::Price,
};

let index = IndexPriceUpdate::new(
    InstrumentId::from("BTCUSDT-PERP.BINANCE"),
    Price::from("64995.50"),
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from nautilus_trader.model import IndexPriceUpdate
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price

index = IndexPriceUpdate(
    instrument_id=InstrumentId.from_str("BTCUSDT-PERP.BINANCE"),
    value=Price.from_str("64995.50"),
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [MarkPriceUpdate](mark_price_update.md) 介绍了标记价格。
- [FundingRateUpdate](funding_rate_update.md) 介绍了永续合约的资金费率元数据。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
