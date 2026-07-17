# MarkPriceUpdate

`MarkPriceUpdate` 表示某个金融工具的当前标记价格（mark price）。交易场所最常在
衍生品的保证金计算（margining）、强平检查（liquidation checks）和未实现盈亏
（unrealized PnL）计算中使用标记价格。

## 字段

| 字段           | Rust 类型      | Python 类型    | 是否必需/默认值 | 说明                                    |
|-----------------|----------------|----------------|------------------|------------------------------------------|
| `instrument_id` | `InstrumentId` | `InstrumentId` | 必需             | 该标记价格对应的金融工具。           |
| `value`         | `Price`        | `Price`        | 必需             | 当前标记价格。                      |
| `ts_event`      | `UnixNanos`    | `int`          | 必需             | 事件时间戳（纳秒）。          |
| `ts_init`       | `UnixNanos`    | `int`          | 必需             | 初始化时间戳（纳秒）。 |

## 行为

- 收到标记价格后会按金融工具进行缓存。
- 回测（backtest）可以输入标记价格，使保证金和盈亏行为与那些将参考价格与
  成交单独发布的交易场所保持一致。
- 目录（catalog）在存储标记价格时会同时保存金融工具 ID 和价格精度元数据。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::MarkPriceUpdate,
    identifiers::InstrumentId,
    types::Price,
};

let mark = MarkPriceUpdate::new(
    InstrumentId::from("BTCUSDT-PERP.BINANCE"),
    Price::from("65000.10"),
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import MarkPriceUpdate
from nautilus_trader.model import Price

mark = MarkPriceUpdate(
    instrument_id=InstrumentId.from_str("BTCUSDT-PERP.BINANCE"),
    value=Price.from_str("65000.10"),
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [IndexPriceUpdate](index_price_update.md) 介绍了指数参考价格。
- [FundingRateUpdate](funding_rate_update.md) 介绍了永续合约的资金费率元数据。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
