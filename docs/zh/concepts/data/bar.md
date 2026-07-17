# Bar

`Bar` 表示特定 `BarType` 的 OHLCV 价格和成交量数据。K 线（bar）可以由交易场所或数据提供商
外部提供，也可以由报价（quote）或成交（trade）行情内部聚合而成，或由更小周期的 K 线聚合而成。

## 字段

| 字段         | Rust 类型   | Python 类型 | 是否必需/默认值 | 说明                                    |
|------------|-------------|-------------|------------------|------------------------------------------|
| `bar_type` | `BarType`   | `BarType`   | 必需             | 金融工具、聚合方式、价格类型和数据来源。 |
| `open`     | `Price`     | `Price`     | 必需             | K 线区间内的第一个价格。         |
| `high`     | `Price`     | `Price`     | 必需             | K 线区间内的最高价格。       |
| `low`      | `Price`     | `Price`     | 必需             | K 线区间内的最低价格。        |
| `close`    | `Price`     | `Price`     | 必需             | K 线区间内的最后一个价格。          |
| `volume`   | `Quantity`  | `Quantity`  | 必需             | 成交量或代表成交笔数的替代量。      |
| `ts_event` | `UnixNanos` | `int`       | 必需             | K 线事件时间戳（纳秒）。      |
| `ts_init`  | `UnixNanos` | `int`       | 必需             | 初始化时间戳（纳秒）。 |

## 行为

- `high` 必须大于或等于 `open`、`low` 和 `close`。
- `low` 必须小于或等于 `open` 和 `close`。
- `bar_type` 决定该 K 线是内部生成还是外部提供。
- 组合 K 线类型使用 `@` 语法来标识来源 K 线类型。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::{Bar, BarType},
    types::{Price, Quantity},
};

let bar = Bar::new(
    BarType::from("AUD/USD.SIM-1-MINUTE-LAST-EXTERNAL"),
    Price::from("0.65000"),
    Price::from("0.65010"),
    Price::from("0.64990"),
    Price::from("0.65005"),
    Quantity::from("1000000"),
    UnixNanos::from(1_000_000_000),
    UnixNanos::from(1_000_000_100),
);
```

```python tab="Python"
from nautilus_trader.model import Bar
from nautilus_trader.model import BarType
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity

bar = Bar(
    bar_type=BarType.from_str("AUD/USD.SIM-1-MINUTE-LAST-EXTERNAL"),
    open=Price.from_str("0.65000"),
    high=Price.from_str("0.65010"),
    low=Price.from_str("0.64990"),
    close=Price.from_str("0.65005"),
    volume=Quantity.from_int(1_000_000),
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [K 线与聚合](index.md#bars-and-aggregation) 介绍了聚合方法。
- [K 线类型](index.md#bar-types) 解释了 `BarType` 字符串语法。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
