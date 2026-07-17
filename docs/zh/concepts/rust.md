# Rust

Nautilus 在 `crates/` 目录下拥有一套完整的 Rust 实现。
你可以编写 actor、策略，运行回测，并在无需 Python 的情况下进行实盘交易。
领域模型在所有路径之间共享，v2 PyO3 路径直接在 Rust 引擎上
运行 Python 策略。

:::warning
Rust API 正在积极开发中。方法签名和 trait 要求可能会在版本之间发生变化。
:::

## 系统实现

Nautilus 有三种实现。理解每种实现的定位有助于你为自己的用例选择合适的一种。

- **v1 传统版本**：位于 `nautilus_trader/` 下的 Cython/Python 类。
  功能最全面，组件覆盖面最广。
- **v2 Rust**：位于 `crates/` 下的纯 Rust 实现。无需 Python 即可运行。
- **v2 PyO3**：Python 用户组件（actor、策略）通过 PyO3 绑定运行在
  Rust 核心之上。结合了 Python 的便捷性与 Rust 引擎的性能。

### 能力对比表

| 组件             | v1 传统版本（Cython） | v2 Rust        | v2 PyO3（Python 运行于 Rust 之上） |
|-----------------------|--------------------|----------------|--------------------------|
| Strategy              | ✓                  | ✓              | ✓                        |
| Actor                 | ✓                  | ✓              | ✓                        |
| DataEngine            | ✓                  | ✓              | ✓                        |
| ExecutionEngine       | ✓                  | ✓              | ✓                        |
| RiskEngine            | ✓                  | ✓              | ✓                        |
| BacktestEngine        | ✓                  | ✓              | ✓                        |
| BacktestNode          | ✓                  | ✓              | ✓                        |
| LiveNode              | ✓                  | ✓              | ✓                        |
| OrderEmulator         | ✓                  | ✓              | ✓                        |
| Matching engine       | ✓                  | ✓              | ✓                        |
| Portfolio             | ✓                  | ✓              | ✓                        |
| Accounts              | ✓                  | ✓              | ✓                        |
| Cache                 | ✓                  | ✓              | ✓                        |
| MessageBus            | ✓                  | ✓              | ✓                        |
| Data catalog          | ✓                  | ✓              | ✓                        |
| Indicators            | ✓                  | ✓              | ✓                        |
| Exec algorithms       | TWAP               | TWAP           | TWAP                     |
| Controller            | ✓                  | -              | ✓                        |
| Tearsheets            | ✓                  | -              | ✓                        |
| Config serialization  | ✓                  | -              | -                        |

### 适配器

| 适配器             | v1 传统版本（Cython） | v2 Rust | v2 PyO3 |
|---------------------|--------------------|---------|---------|
| Architect AX        | ✓                  | ✓       | ✓       |
| Betfair             | ✓                  | ✓       | ✓       |
| Binance             | ✓                  | ✓       | ✓       |
| BitMEX              | ✓                  | ✓       | ✓       |
| Blockchain          | -                  | ✓       | ✓       |
| Bybit               | ✓                  | ✓       | ✓       |
| Coinbase            | -                  | ✓       | ✓       |
| Databento           | ✓                  | ✓       | ✓       |
| Deribit             | ✓                  | ✓       | ✓       |
| Derive              | -                  | ✓       | ✓       |
| dYdX                | ✓                  | ✓       | ✓       |
| Hyperliquid         | ✓                  | ✓       | ✓       |
| Interactive Brokers | ✓                  | ✓       | ✓       |
| Kraken              | ✓                  | ✓       | ✓       |
| Lighter             | -                  | ✓       | ✓       |
| OKX                 | ✓                  | ✓       | ✓       |
| Polymarket          | ✓                  | ✓       | ✓       |
| Sandbox             | ✓                  | ✓       | ✓       |
| Tardis              | ✓                  | ✓       | ✓       |

### 如何选择

- **v1 传统版本**目前是最完整的。如果你需要 Controller 或
  配置序列化，请使用它。
- **v2 Rust** 在没有 Python 运行时的情况下提供原生性能。所有核心交易
  功能都可用。适用于对延迟敏感的部署，或偏好编译型语言的团队。
- **v2 PyO3**：Python 用户组件（actor、策略）运行在 Rust 核心引擎上，
  在数据处理和执行方面具有 Rust 性能，同时保留了 Python 的编写体验。

## 项目设置

Nautilus 的 crate 发布在
[crates.io](https://crates.io/crates/nautilus-backtest) 上。将它们添加到你的
`Cargo.toml`：

```toml
[dependencies]
nautilus-backtest = "0.60"
nautilus-common = "0.60"
nautilus-execution = "0.60"
nautilus-model = { version = "0.60", features = ["stubs"] }
nautilus-trading = { version = "0.60", features = ["examples"] }

anyhow = "1"
log = "0.4"
```

对于实盘交易，添加 live crate 以及你所交易场所对应的适配器：

```toml
[dependencies]
nautilus-live = "0.60"
nautilus-okx = "0.60"
```

要跟踪最新的开发分支，请将所有 Nautilus 依赖指向同一个 git 源，
以避免 crates.io 版本和 git 版本之间出现类型不匹配：

```toml
[dependencies]
nautilus-backtest = { git = "https://github.com/nautechsystems/nautilus_trader.git", branch = "develop" }
nautilus-common = { git = "https://github.com/nautechsystems/nautilus_trader.git", branch = "develop" }
nautilus-execution = { git = "https://github.com/nautechsystems/nautilus_trader.git", branch = "develop" }
nautilus-model = { git = "https://github.com/nautechsystems/nautilus_trader.git", branch = "develop", features = ["stubs"] }
nautilus-trading = { git = "https://github.com/nautechsystems/nautilus_trader.git", branch = "develop", features = ["examples"] }
```

支持的最低 Rust 版本（MSRV）为 **1.97.0**。

### 功能标志

| 标志             | Crate               | 效果                                                        |
|------------------|---------------------|-----------------------------------------------------------------|
| `high-precision` | `nautilus-model`    | 16 位定点精度（默认为 9 位）。加密货币场景下必需。 |
| `stubs`          | `nautilus-model`    | 测试用的金融工具桩（`audusd_sim` 等）。                   |
| `examples`       | `nautilus-trading`  | 示例策略（`EmaCross`、`GridMarketMaker`）。           |
| `streaming`      | `nautilus-backtest` | 通过 `BacktestNode` 进行基于 catalog 的数据流式传输。              |
| `defi`           | `nautilus-model`    | DeFi 数据类型。隐含启用 `high-precision`。                    |

:::tip
标准的 9 位精度可以处理大多数传统金融的金融工具。对于价格可能有很多
小数位（例如 `0.00000001`）的加密货币交易场所，请启用 `high-precision`。
:::

### 内存分配器

Python wheel 和 `nautilus` CLI 对 Rust 分配使用
[mimalloc](https://crates.io/crates/mimalloc)。一个 Rust 二进制文件会自行
选择其分配器，因此请在你自己的项目中添加 mimalloc 以保持一致：

```toml
[dependencies]
mimalloc = "0.1"
```

```rust
use mimalloc::MiMalloc;

#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;
```

默认的系统分配器也能正常工作，但回测吞吐量会明显下降，
在 Windows 上尤为明显，其分配器开销可能达到热循环运行时间的一半。
背景信息请参阅[架构指南](architecture.md#memory-allocation)。

## Actor

一个 actor 接收市场数据、自定义数据/信号和系统事件，但不管理订单。
实现 `DataActor` trait，并使用 `nautilus_actor!` 将你的 `DataActorCore`
字段接入运行时契约。你的类型实现或派生 `Debug`；该宏提供原生运行时接线。
用户代码通常使用 `DataActor` 外观（facade）方法进行订阅、缓存访问和
时钟访问。

### 处理器方法

重写 `DataActor` trait 上的任意处理器以接收对应的数据或事件。
所有处理器都有默认的无操作实现，因此你只需重写所需的部分。

| 处理器                | 接收内容                  |
|------------------------|---------------------------|
| `on_start`             | actor 已启动。            |
| `on_stop`              | actor 已停止。            |
| `on_quote`             | `QuoteTick`               |
| `on_trade`             | `TradeTick`               |
| `on_bar`               | `Bar`                     |
| `on_book_deltas`       | `OrderBookDeltas`         |
| `on_book`              | `OrderBook`（按间隔） |
| `on_instrument`        | `InstrumentAny`           |
| `on_mark_price`        | `MarkPriceUpdate`         |
| `on_index_price`       | `IndexPriceUpdate`        |
| `on_funding_rate`      | `FundingRateUpdate`       |
| `on_option_greeks`     | `OptionGreeks`            |
| `on_option_chain`      | `OptionChainSlice`        |
| `on_instrument_status` | `InstrumentStatus`        |
| `on_order_filled`      | `OrderFilled`             |
| `on_order_canceled`    | `OrderCanceled`           |
| `on_time_event`        | `TimeEvent`               |

有关分步演练，请参阅[编写一个 Actor（Rust）](../how_to/write_rust_actor.md)操作指南。
完整示例请参阅
[`BookImbalanceActor`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/trading/src/examples/actors/imbalance)。

## 策略

一个策略在 actor 的基础上扩展了订单管理。实现 `DataActor` 以处理数据，
并使用 `nautilus_strategy!` 将你的 `StrategyCore` 字段接入策略运行时契约。
`StrategyCore` 存储了运行时策略状态；常规策略逻辑通过 `self` 上的外观
方法访问它。运行时注册需要该宏生成的原生接线，但常规策略逻辑使用
`Strategy` 方法以及 `self` 上的外观方法。

### 订单管理

`Strategy` trait 通过外观提供订单方法：

| 方法                | 操作                                    |
|-----------------------|-------------------------------------------|
| `submit_order`        | 向交易场所提交一个新订单。          |
| `submit_order_list`   | 提交一组或有（contingent）订单。       |
| `modify_order`        | 修改价格、数量或触发价格。 |
| `cancel_order`        | 撤销某个特定订单。                  |
| `cancel_orders`       | 撤销一组经过过滤的订单。            |
| `cancel_all_orders`   | 撤销某个金融工具的所有订单。      |
| `close_position`      | 用市价单平仓一个仓位。     |
| `close_all_positions` | 平掉所有未平仓仓位。                 |

`OrderApi`（通过 `self.order()` 访问）用于构建订单和订单列表：

- `generate_client_order_id`
- `generate_order_list_id`
- `market`
- `limit`
- `stop_market`
- `stop_limit`
- `market_to_limit`
- `market_if_touched`
- `limit_if_touched`
- `trailing_stop_market`
- `trailing_stop_limit`
- `bracket`
- `create_list`

### 核心接线宏

Rust 中的 actor、策略和执行算法都将其运行时核心保存为一个结构体字段。
这些宏告诉 trait 该字段位于何处。

| 宏                                          | 核心字段               | 生成内容                       |
|------------------------------------------------|--------------------------|----------------------------------|
| `nautilus_actor!(Type)`                        | `DataActorCore`          | 运行时接线。                 |
| `nautilus_strategy!(Type)`                     | `StrategyCore`           | 运行时接线和 `Strategy`。  |
| `nautilus_execution_algorithm!(Type, { ... })` | `ExecutionAlgorithmCore` | 运行时接线和算法。   |

这些宏期望有一个名为 `core` 的字段；如有需要，可以将字段名作为第二个
参数传入。它们不会使 actor、策略或 `StrategyCore` 解引用到运行时内部结构。
执行算法宏接受一个 `on_order()` 实现块，因为该方法定义了该算法所需的
订单处理逻辑。
常规代码使用诸如以下的外观方法：

- `actor_id()`
- `trader_id()`
- `is_registered()`
- `config()`
- `strategy_id()`
- `clock()`
- `cache()`
- `order()`
- `portfolio()`

### 原生 trait

默认情况下使用外观方法：

- `actor_id()`
- `trader_id()`
- `is_registered()`
- `config()`
- `strategy_id()`
- `clock()`
- `cache()`
- `order()`
- `portfolio()`

`DataActorNative`、`StrategyNative` 和 `ExecutionAlgorithmNative`
用于该外观之下的仅限原生代码的访问。本节记录的是引擎、运行时以及
显式的对延迟敏感的原生 Rust 代码，而不是可移植的编写路径。

| 编写路径            | 需要原生 trait？   | 常规 API                          |
|---------------------------|------------------|--------------------------------------|
| 原生 Rust 二进制文件        | 仅在需要时 | `Strategy` 和 `DataActor` 外观。 |
| 从 Python 启动的 Rust | 仅在需要时 | 与原生 Rust 相同。                |
| Python 编写的组件 | 否               | 只使用外观。                             |

原生 trait 暴露了借用的核心状态、`Rc<RefCell<_>>` 以及运行时引用。
仅当原生 Rust 代码有意为某个明确的、对延迟敏感的路径接受这些借用规则时，
才使用它们。引擎、运行时、注册、PyO3 和测试工具包代码在需要访问 actor
核心、策略核心或执行算法核心时，可以导入 `DataActorNative`、
`StrategyNative` 或 `ExecutionAlgorithmNative`。不要在常规的可移植 actor、
策略或执行算法逻辑，或 Python 编写的组件中使用它们，因为这些类型不会
跨越 Python 边界。

`ExecutionAlgorithmCore` 拥有一个 `DataActorCore`，但它不会解引用到该
类型。常规的执行算法逻辑应使用 `id()`、`actor_id()`、`trader_id()`、
`clock()` 和 `cache()`。只有当代码需要原生执行算法状态时，才使用
`ExecutionAlgorithmNative`。

选择最小范围的原生句柄，并将每次借用的作用域控制在最小范围内。
使用 `order()` 进行常规的策略订单构造。只有当原生代码需要原始的
可变工厂借用时，才使用 `order_factory()`。

#### `DataActorNative` 方法

| 原生方法 | 返回形态             | 使用时机                        |
|---------------|--------------------------|---------------------------------|
| `core()`      | `&DataActorCore`         | 读取 actor 内部状态。           |
| `core_mut()`  | `&mut DataActorCore`     | 修改 actor 内部状态。         |
| `clock_mut()` | `RefMut<'_, dyn Clock>`  | 需要可变的时钟借用。    |
| `clock_rc()`  | `Rc<RefCell<dyn Clock>>` | 存储或传递共享时钟。 |
| `cache_ref()` | `Ref<'_, Cache>`         | 需要短生命周期的缓存读取。    |
| `cache_rc()`  | `Rc<RefCell<Cache>>`     | 修改、存储或传递缓存。   |

#### `StrategyNative` 方法

| 原生方法         | 返回形态                 | 使用时机                          |
|-----------------------|------------------------------|------------------------------------|
| `strategy_core()`     | `&StrategyCore`              | 读取策略内部状态。          |
| `strategy_core_mut()` | `&mut StrategyCore`          | 修改策略内部状态。        |
| `order_factory()`     | `RefMut<'_, OrderFactory>`   | 需要原始的可变工厂借用。  |
| `order_factory_rc()`  | `Rc<RefCell<OrderFactory>>`  | 存储或传递该工厂。        |
| `portfolio_rc()`      | `Rc<RefCell<Portfolio>>`     | 存储或传递投资组合。      |

#### `ExecutionAlgorithmNative` 方法

| 原生方法               | 返回形态                   | 使用时机                              |
|-----------------------------|---------------------------------|----------------------------------------|
| `exec_algorithm_core()`     | `&ExecutionAlgorithmCore`      | 读取执行算法内部状态。   |
| `exec_algorithm_core_mut()` | `&mut ExecutionAlgorithmCore`  | 修改执行算法内部状态。 |

有关分步演练，请参阅[编写一个策略（Rust）](../how_to/write_rust_strategy.md)操作指南。
完整示例请参阅
[`EmaCross`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/trading/src/examples/strategies/ema_cross)
和
[`GridMarketMaker`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/trading/src/examples/strategies/grid_mm)。

### 运行 Rust 组件

Rust 策略和 actor 可以通过两条路径运行。以下示例使用策略，
但同样的模式也适用于通过 `add_actor`（纯 Rust）和 `add_builtin_actor`
（从 Python）添加的内置 actor。

#### 纯 Rust

用 Rust 编写你的策略和 `main` 函数，然后使用 `cargo build` 构建一个
独立的二进制文件。这条路径不需要 Python 运行时。

```rust
let strategy = GridMarketMaker::new(config);
node.add_strategy(strategy)?;
node.run().await?;
```

完整演练请参阅[运行实盘交易（Rust）](../how_to/run_rust_live_trading.md)。

#### 从 Python 使用内置示例

将类型名和配置传给 `add_builtin_strategy`，即可从 Python 注册一个
内置的示例策略。这条路径的存在是为了在 Rust 和 Python 的文档、示例
和测试之间对捆绑的示例策略代码保持单一数据源。它不是添加原生策略的
一流扩展路径。对于自定义的原生组件，请使用纯 Rust。

```python
from nautilus_trader.core.nautilus_pyo3.trading import GridMarketMakerConfig

config = GridMarketMakerConfig(
    instrument_id=InstrumentId.from_str("BTC-USDT-SWAP.OKX"),
    max_position=Quantity.from_str("10.0"),
    trade_size=Quantity.from_str("0.1"),
    num_levels=5,
    grid_step_bps=15,
)

node.add_builtin_strategy("GridMarketMaker", config)
```

内置的策略配置：

| 配置                         | 策略                 |
|--------------------------------|--------------------------|
| `CompositeMarketMakerConfig`   | `CompositeMarketMaker`   |
| `DeltaNeutralVolConfig`        | `DeltaNeutralVol`        |
| `EmaCrossConfig`               | `EmaCross`               |
| `ExecTesterConfig`             | `ExecTester`             |
| `GridMarketMakerConfig`        | `GridMarketMaker`        |
| `HurstVpinDirectionalConfig`   | `HurstVpinDirectional`   |

对于示例和测试所使用的 actor，`add_builtin_actor` 遵循相同的
仅限捆绑组件的规则。

内置的 actor 配置（通过 `add_builtin_actor`）：

| 配置                     | Actor                 |
|----------------------------|-----------------------|
| `BookImbalanceActorConfig` | `BookImbalanceActor`  |
| `DataTesterConfig`         | `DataTester`          |

## 回测

有关两种 API 带注释的演练，请参阅
[运行回测（Rust）](../how_to/run_rust_backtest.md)操作指南。

### `BacktestEngine`（低级 API）

构造引擎、添加交易场所和金融工具、加载数据、注册策略，然后运行。
完整可运行的示例：

```bash
cargo run -p nautilus-backtest --features examples --example engine-ema-cross
```

源代码：
[`crates/backtest/examples/engine_ema_cross.rs`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/backtest/examples/engine_ema_cross.rs)

### `BacktestNode`（高级 API）

从 `ParquetDataCatalog` 加载数据，并支持以可配置的分块大小进行流式传输。
需要在 `nautilus-backtest` 上启用 `streaming` 功能。完整可运行的示例：

```bash
cargo run -p nautilus-backtest --features examples,streaming --example node-ema-cross
```

源代码：
[`crates/backtest/examples/node_ema_cross.rs`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/backtest/examples/node_ema_cross.rs)

## 实盘交易

有关带注释的演练，请参阅
[运行实盘交易（Rust）](../how_to/run_rust_live_trading.md)操作指南。

`LiveNode` 通过适配器客户端连接到真实的交易场所和数据源。构建器
模式配置数据客户端和执行客户端，然后调用 `run()` 启动异步事件循环。
每个适配器都提供自己的工厂和配置类型。

| 适配器             | 示例                                                |
|---------------------|--------------------------------------------------------|
| Architect AX        | `crates/adapters/architect_ax/examples/`               |
| Betfair             | `crates/adapters/betfair/examples/`                    |
| Binance             | `crates/adapters/binance/examples/`                    |
| BitMEX              | `crates/adapters/bitmex/examples/`                     |
| Blockchain          | `crates/adapters/blockchain/examples/`                 |
| Bybit               | `crates/adapters/bybit/examples/`                      |
| Coinbase            | `crates/adapters/coinbase/examples/`                   |
| Databento           | `crates/adapters/databento/examples/`                  |
| Deribit             | `crates/adapters/deribit/examples/`                    |
| Derive              | `crates/adapters/derive/examples/`                     |
| dYdX                | `crates/adapters/dydx/examples/`                       |
| Hyperliquid         | `crates/adapters/hyperliquid/examples/`                |
| Interactive Brokers | `crates/adapters/interactive_brokers/examples/`        |
| Kraken              | `crates/adapters/kraken/examples/`                     |
| Lighter             | `crates/adapters/lighter/examples/`                    |
| OKX                 | `crates/adapters/okx/examples/`                        |
| Polymarket          | `crates/adapters/polymarket/examples/`                 |
| Sandbox             | `crates/adapters/sandbox/examples/`                    |
| Tardis              | `crates/adapters/tardis/examples/`                     |

大多数适配器都包含 `node_data_tester.rs` 和 `node_exec_tester.rs`
示例。这些示例针对实盘交易场所测试数据请求、流式传输和订单执行。

## 相关指南

- [编写一个 Actor（Rust）](../how_to/write_rust_actor.md) - 分步 actor 演练。
- [编写一个策略（Rust）](../how_to/write_rust_strategy.md) - 分步策略演练。
- [运行回测（Rust）](../how_to/run_rust_backtest.md) - BacktestEngine 和 BacktestNode 的用法。
- [运行实盘交易（Rust）](../how_to/run_rust_live_trading.md) - LiveNode 设置和交易场所连接。
- [架构（Architecture）](architecture.md) - 系统设计以及数据/执行流转。
- [Actor](actors.md) - Actor 概念（同时适用于 Python 和 Rust）。
- [策略（Strategies）](strategies.md) - 策略概念和处理器参考。
- [事件（Events）](events/) - 事件类型和处理器分发。
- [回测（Backtesting）](backtesting/) - 回测概念和撮合引擎行为。
