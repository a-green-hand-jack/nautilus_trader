# OptionGreeks

`OptionGreeks` 表示交易场所提供的某个期权金融工具的希腊字母（Greeks）敏感度
指标和隐含波动率（implied volatility）。它是原生的 `Data` 枚举变体，可以被
记录、回放并通过目录（catalog）进行查询。

## 字段

| 字段              | Rust 类型             | Python 类型      | 是否必需/默认值 | 说明                                      |
|--------------------|-----------------------|------------------|------------------|--------------------------------------------|
| `instrument_id`    | `InstrumentId`        | `InstrumentId`   | 必需             | 该希腊字母对应的期权金融工具。          |
| `convention`       | `GreeksConvention`    | `GreeksConvention` | 默认值        | 数值所使用的计价（numeraire）约定。       |
| `greeks`           | `OptionGreekValues`   | 各自独立的浮点数字段  | 必需         | Delta、Gamma、Vega、Theta 和 Rho。        |
| `mark_iv`          | `Option<f64>`         | `float \| None`  | `None`           | 标记隐含波动率。                   |
| `bid_iv`           | `Option<f64>`         | `float \| None`  | `None`           | 买价隐含波动率。                   |
| `ask_iv`           | `Option<f64>`         | `float \| None`  | `None`           | 卖价隐含波动率。                   |
| `underlying_price` | `Option<f64>`         | `float \| None`  | `None`           | 用于计算的标的资产价格。 |
| `open_interest`    | `Option<f64>`         | `float \| None`  | `None`           | 已发布时的未平仓合约量（open interest）。              |
| `ts_event`         | `UnixNanos`           | `int`            | 必需             | 事件时间戳（纳秒）。            |
| `ts_init`          | `UnixNanos`           | `int`            | 必需             | 初始化时间戳（纳秒）。   |

## 行为

- 在 Rust 层面，`OptionGreeks` 会解引用（deref）到其核心的 `OptionGreekValues`。
- Python 构造函数接受 `delta`、`gamma`、`vega`、`theta` 以及可选的 `rho`
  作为独立的浮点数参数。
- 期权链（option chain）订阅使用 `underlying_price` 和各 delta 值来确定平值
  （ATM）以及基于 delta 的行权价窗口。

## 示例

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    data::{OptionGreekValues, OptionGreeks},
    enums::GreeksConvention,
    identifiers::InstrumentId,
};

let greeks = OptionGreeks {
    instrument_id: InstrumentId::from("BTC-20240628-65000-C.DERIBIT"),
    convention: GreeksConvention::PriceAdjusted,
    greeks: OptionGreekValues {
        delta: 0.51,
        gamma: 0.0002,
        vega: 12.5,
        theta: -3.2,
        rho: 0.1,
    },
    mark_iv: Some(0.55),
    bid_iv: Some(0.54),
    ask_iv: Some(0.56),
    underlying_price: Some(65_000.0),
    open_interest: Some(120.0),
    ts_event: UnixNanos::from(1_000_000_000),
    ts_init: UnixNanos::from(1_000_000_100),
};
```

```python tab="Python"
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import OptionGreeks

greeks = OptionGreeks(
    instrument_id=InstrumentId.from_str("BTC-20240628-65000-C.DERIBIT"),
    delta=0.51,
    gamma=0.0002,
    vega=12.5,
    theta=-3.2,
    rho=0.1,
    mark_iv=0.55,
    bid_iv=0.54,
    ask_iv=0.56,
    underlying_price=65_000.0,
    open_interest=120.0,
    ts_event=1_000_000_000,
    ts_init=1_000_000_100,
)
```

## 相关指南

- [Greeks](../greeks.md) 介绍了交易场所提供以及本地计算的希腊字母。
- [Options](../options.md#optiongreeks-data-type) 介绍了期权链订阅。
- [Python API 参考](/docs/python-api-latest/model/data.html) 列出了 Python 成员。
