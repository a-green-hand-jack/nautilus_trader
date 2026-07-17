# 运行回测（Rust）

Nautilus 提供了两种用于回测的 Rust API：`BacktestEngine`（低阶）和
`BacktestNode`（高阶，支持数据目录流式读取）。本指南将介绍这两种方式。

有关回测概念、成交模型（fill model）以及撮合引擎行为的背景知识，请参见
[回测（Backtesting）](../concepts/backtesting/)概念指南。有关项目搭建和特性
标志，请参见 [Rust](../concepts/rust.md#project-setup) 概念指南。

## 依赖

在 `Cargo.toml` 中添加以下内容。`streaming` 和 `nautilus-persistence` 这两项
仅在使用高阶 `BacktestNode` API 时才需要。

```toml
[dependencies]
nautilus-backtest = { version = "0.60", features = ["streaming"] }
nautilus-execution = "0.60"
nautilus-model = { version = "0.60", features = ["stubs"] }
nautilus-persistence = "0.60"
nautilus-trading = { version = "0.60", features = ["examples"] }

ahash = "0.8"
anyhow = "1"
tempfile = "3"
ustr = "1"
```

如果你只需要低阶的 `BacktestEngine`，可以去掉 `streaming`、
`nautilus-persistence`、`tempfile` 和 `ustr`。

## BacktestEngine（低阶 API）

低阶 API 提供直接控制：你需要自行构建引擎、添加交易场所和金融工具、在内存中
加载数据、注册策略，然后运行。

### 1. 创建引擎

```rust
use nautilus_backtest::{config::BacktestEngineConfig, engine::BacktestEngine};

let mut engine = BacktestEngine::new(BacktestEngineConfig::default())?;
```

### 2. 添加交易场所

`SimulatedVenueConfig` 使用 `bon::Builder`：只需要设置必填字段，其余设置都会
回退到文档中说明的默认值。`build()` 会校验配置并返回 `ConfigResult`，因此
需要传播（propagate）或直接解包该结果。

```rust
use nautilus_backtest::config::SimulatedVenueConfig;
use nautilus_model::{
    enums::{AccountType, BookType, OmsType},
    identifiers::Venue,
    types::Money,
};

engine.add_venue(
    SimulatedVenueConfig::builder()
        .venue(Venue::from("SIM"))
        .oms_type(OmsType::Hedging)
        .account_type(AccountType::Margin)
        .book_type(BookType::L1_MBP)
        .starting_balances(vec![Money::from("1_000_000 USD")])
        .build()?,
)?;
```

可以通过链式调用其他 setter 来覆盖任意默认值，例如
`.reject_stop_orders(false)` 或 `.allow_cash_borrowing(true)`。

### 3. 添加金融工具和数据

```rust
use nautilus_model::instruments::{
    Instrument, InstrumentAny, stubs::audusd_sim,
};

let instrument = InstrumentAny::CurrencyPair(audusd_sim());
let instrument_id = instrument.id();
engine.add_instrument(&instrument)?;

let quotes = generate_quotes(instrument_id); // 你自己的数据加载函数
engine.add_data(quotes, None, true, true)?;
```

### 4. 注册策略并运行

```rust
use nautilus_model::types::Quantity;
use nautilus_trading::examples::strategies::EmaCross;

let strategy = EmaCross::new(
    instrument_id,
    Quantity::from("100000"),
    10, // 快速 EMA 周期
    20, // 慢速 EMA 周期
);

engine.add_strategy(strategy)?;
engine.run(None, None, None, false)?;
```

### 运行完整示例

```bash
cargo run -p nautilus-backtest --features examples --example engine-ema-cross
```

源码：
[`crates/backtest/examples/engine_ema_cross.rs`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/backtest/examples/engine_ema_cross.rs)

## BacktestNode（高阶 API）

高阶 API 从 `ParquetDataCatalog` 中加载数据，并以可配置的块大小（chunk size）
进行流式读取。需要为 `nautilus-backtest` 启用 `streaming` 特性。

### 1. 将数据写入数据目录

```rust
use nautilus_model::instruments::{
    Instrument, InstrumentAny, stubs::audusd_sim,
};
use nautilus_persistence::backend::catalog::ParquetDataCatalog;
use tempfile::TempDir;

let instrument = InstrumentAny::CurrencyPair(audusd_sim());
let instrument_id = instrument.id();
let quotes = generate_quotes(instrument_id);

let temp_dir = TempDir::new()?;
let catalog_path = temp_dir.path().to_str()
    .context("temp dir path is not valid UTF-8")?
    .to_string();
let catalog = ParquetDataCatalog::new(
    temp_dir.path(), None, None, None, None,
);

catalog.write_instruments(vec![instrument])?;
catalog.write_to_parquet(&quotes, None, None, None)?;
```

### 2. 配置运行

```rust
use nautilus_backtest::config::{
    BacktestDataConfig, BacktestRunConfig, BacktestVenueConfig, NautilusDataType,
};
use nautilus_model::enums::{AccountType, BookType, OmsType};

let venue_config = BacktestVenueConfig::builder()
    .name("SIM")
    .oms_type(OmsType::Hedging)
    .account_type(AccountType::Margin)
    .book_type(BookType::L1_MBP)
    .starting_balances(vec!["1_000_000 USD".to_string()])
    .build()?;

let data_config = BacktestDataConfig::builder()
    .data_type(NautilusDataType::QuoteTick)
    .catalog_path(catalog_path)
    .instrument_id(instrument_id)
    .build()?;

let run_config = BacktestRunConfig::builder()
    .id("ema-cross-run".to_string())
    .venues(vec![venue_config])
    .data(vec![data_config])
    .chunk_size(100)
    .build()?;
```

### 3. 构建、添加策略并运行

```rust
use nautilus_backtest::node::BacktestNode;
use nautilus_model::types::Quantity;
use nautilus_trading::examples::strategies::EmaCross;

let mut node = BacktestNode::new(vec![run_config])?;
node.build()?;

let engine = node.get_engine_mut("ema-cross-run")
    .context("engine not found for run config ID")?;
let strategy = EmaCross::new(
    instrument_id,
    Quantity::from("100000"),
    10,
    20,
);
engine.add_strategy(strategy)?;

node.run()?;
```

### 运行完整示例

```bash
cargo run -p nautilus-backtest --features examples,streaming --example node-ema-cross
```

源码：
[`crates/backtest/examples/node_ema_cross.rs`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/backtest/examples/node_ema_cross.rs)
</content>
