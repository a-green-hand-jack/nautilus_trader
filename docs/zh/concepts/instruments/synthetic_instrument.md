# 合成工具（Synthetic Instrument）

`SyntheticInstrument` 表示一种本地工具，其价格来自对其他工具应用某个公式计算得出。它适用于价差、篮子（basket）、比率以及其他应当以工具形式出现在系统中的衍生价格。

示例包括 `(BTC.BINANCE + LTC.BINANCE) / 2.0`，以及由成分工具（component instrument）价格构建的比率型价差对。

## 字段

| 字段               | Rust 类型            | Python 类型            | 是否必需/默认值 | 说明                                         |
|-------------------|---------------------|----------------------|------------------|---------------------------------------------|
| `symbol`          | `Symbol`            | `Symbol`             | 必需             | 与交易场所 `SYNTH` 一起使用的合成符号。      |
| `id`              | `InstrumentId`      | `InstrumentId`       | 推导得出         | 由 `symbol.SYNTH` 组成的工具 ID。            |
| `price_precision` | `u8`                | `int`                | 必需             | 合成价格允许的小数位数。                     |
| `price_increment` | `Price`             | `Price`              | 推导得出         | 根据精度得出的最小价格步长。                 |
| `components`      | `Vec<InstrumentId>` | `list[InstrumentId]` | 必需             | 该公式所使用的成分工具。                     |
| `formula`         | `String`            | `str`                | 必需             | 基于成分工具 ID 的数值表达式。               |
| `ts_event`        | `UnixNanos`         | `int`                | 必需             | 以纳秒为单位的事件时间戳。                   |
| `ts_init`         | `UnixNanos`         | `int`                | 必需             | 以纳秒为单位的初始化时间戳。                 |

*注：Python 根据 `symbol` 和 `SYNTH` 交易场所构建工具 ID。Rust 将同一值存储为 `id`。*

## 行为特征

- `SyntheticInstrument` 是 Nautilus 本地的，并不代表某个交易场所上可下单的市场。
- 它始终使用合成交易场所 `SYNTH`。
- Python 端要求至少提供两个成分工具 ID。
- 在该对象生效之前，公式必须能够针对所提供的成分标识符成功编译。
- 它没有交易场所限制、保证金、手续费、订单簿或适配器特定元数据。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    identifiers::{InstrumentId, Symbol},
    instruments::SyntheticInstrument,
};

let synthetic = SyntheticInstrument::new(
    Symbol::from("BTC-LTC"),
    2,
    vec![
        InstrumentId::from("BTC.BINANCE"),
        InstrumentId::from("LTC.BINANCE"),
    ],
    "(BTC.BINANCE + LTC.BINANCE) / 2.0",
    UnixNanos::default(),
    UnixNanos::default(),
);
```

```python tab="Python"
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Symbol
from nautilus_trader.model import SyntheticInstrument

synthetic = SyntheticInstrument(
    symbol=Symbol("BTC-LTC"),
    price_precision=2,
    components=[
        InstrumentId.from_str("BTC.BINANCE"),
        InstrumentId.from_str("LTC.BINANCE"),
    ],
    formula="(BTC.BINANCE + LTC.BINANCE) / 2.0",
    ts_event=0,
    ts_init=0,
)
```

## 适配器

`SyntheticInstrument` 仅限本地使用。它从成分工具推导价格，而这些成分工具可能来自系统中已加载的任意适配器。

## 相关指南

- [合成工具（Synthetics）](../synthetics.md) 介绍了公式推导的工具和合成 K 线（synthetic bar）。
- [数据（Data）](../data/) 说明了引用工具的市场数据。
</content>
