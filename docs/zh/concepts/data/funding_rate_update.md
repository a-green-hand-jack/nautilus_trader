# FundingRateUpdate

`FundingRateUpdate` 表示某个永续合约（perpetual swap）金融工具的当前资金费率
（funding rate）。当交易场所发布资金费率区间和下一次资金费用结算时间时，该类型
也可以包含这些信息。

## 字段

| 字段             | Rust 类型             | Python 类型      | 是否必需/默认值 | 说明                                    |
|-------------------|-----------------------|------------------|------------------|------------------------------------------|
| `instrument_id`   | `InstrumentId`        | `InstrumentId`   | 必需             | 该资金费率对应的永续合约工具。       |
| `rate`            | `Decimal`             | `Decimal`        | 必需             | 当前资金费率。                    |
| `interval`        | `Option<u16>`         | `int \| None`    | `None`           | 资金费率结算周期（分钟）。             |
| `next_funding_ns` | `Option<UnixNanos>`   | `int \| None`    | `None`           | 下一次资金费用结算时间戳（纳秒）。   |
| `ts_event`        | `UnixNanos`           | `int`            | 必需             | 事件时间戳（纳秒）。          |
| `ts_init`         | `UnixNanos`           | `int`            | 必需             | 初始化时间戳（纳秒）。 |

## 行为

- 相等性判断和哈希计算使用金融工具 ID、费率、结算周期以及下一次结算时间。
- 资金费率属于参考数据，并不表示已经发生了资金支付。
- 仅在交易场所实际发布时才使用 `interval` 和 `next_funding_ns`。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{data::FundingRateUpdate, identifiers::InstrumentId};
use rust_decimal::Decimal;

let funding = FundingRateUpdate::new(
    InstrumentId::from("BTCUSDT-PERP.BINANCE"),
    Decimal::new(1, 4),
    Some(480),
    Some(UnixNanos::from(1_000_008_000)),
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from decimal import Decimal

from nautilus_trader.model import FundingRateUpdate
from nautilus_trader.model import InstrumentId

funding = FundingRateUpdate(
    instrument_id=InstrumentId.from_str("BTCUSDT-PERP.BINANCE"),
    rate=Decimal("0.0001"),
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
    interval=480,
    next_funding_ns=1_000_008_000,
)
```

## 相关指南

- [MarkPriceUpdate](mark_price_update.md) 介绍了衍生品的标记价格（mark price）。
- [IndexPriceUpdate](index_price_update.md) 介绍了指数参考价格。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
