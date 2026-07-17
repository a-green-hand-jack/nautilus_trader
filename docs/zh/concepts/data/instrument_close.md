# InstrumentClose

`InstrumentClose` 表示某个金融工具在交易场所的收盘价事件，用于表示交易日
（session）结束时的收盘或合约到期时的收盘事件。

## 字段

| 字段           | Rust 类型             | Python 类型           | 是否必需/默认值 | 说明                                    |
|-----------------|-----------------------|-----------------------|------------------|------------------------------------------|
| `instrument_id` | `InstrumentId`        | `InstrumentId`        | 必需             | 发生收盘的金融工具。                 |
| `close_price`   | `Price`               | `Price`               | 必需             | 收盘价或结算价。             |
| `close_type`    | `InstrumentCloseType` | `InstrumentCloseType` | 必需             | `END_OF_SESSION`（交易日结束）或 `CONTRACT_EXPIRED`（合约到期）。  |
| `ts_event`      | `UnixNanos`           | `int`                 | 必需             | 事件时间戳（纳秒）。          |
| `ts_init`       | `UnixNanos`           | `int`                 | 必需             | 初始化时间戳（纳秒）。 |

## 行为

- 交易日结束收盘（end-of-session close）提供当日交易时段级别的收盘价。
- 合约到期收盘（contract-expiry close）标记有到期日的合约的到期事件。
- 收盘价属于参考数据；并不表示在该价格上真的发生了成交。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::InstrumentClose,
    enums::InstrumentCloseType,
    identifiers::InstrumentId,
    types::Price,
};

let close = InstrumentClose::new(
    InstrumentId::from("ESM4.XCME"),
    Price::from("5325.25"),
    InstrumentCloseType::EndOfSession,
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from nautilus_trader.model import InstrumentClose
from nautilus_trader.model import InstrumentCloseType
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price

close = InstrumentClose(
    instrument_id=InstrumentId.from_str("ESM4.XCME"),
    close_price=Price.from_str("5325.25"),
    close_type=InstrumentCloseType.END_OF_SESSION,
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [InstrumentStatus](instrument_status.md) 介绍了金融工具状态事件。
- [Instruments](../instruments/) 介绍了金融工具的定义。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
