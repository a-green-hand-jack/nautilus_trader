# 运行实盘交易（Rust）

`LiveNode` 通过适配器客户端连接到真实的交易场所和数据源。本指南以 OKX 为例，
完整讲解一次实盘交易的搭建过程。

有关实盘交易架构和状态对账的背景信息，请参见[实盘交易](../concepts/live.md)
概念指南。有关项目搭建和特性标志，请参见 [Rust](../concepts/rust.md#project-setup)
概念指南。

## 依赖

将 live crate、你所需的交易场所适配器以及配套 crate 添加到 `Cargo.toml`：

```toml
[dependencies]
nautilus-common = "0.60"
nautilus-live = "0.60"
nautilus-model = "0.60"
nautilus-okx = "0.60"
nautilus-trading = { version = "0.60", features = ["examples"] }

anyhow = "1"
dotenvy = "0.15"
log = "0.4"
tokio = { version = "1", features = ["full"] }
```

## 构建节点

`LiveNode` 使用构建器模式。请为你的交易场所添加数据和执行客户端工厂
（factory），配置日志，然后进行构建。

```rust
use log::LevelFilter;
use nautilus_common::{enums::Environment, logging::logger::LoggerConfig};
use nautilus_live::node::LiveNode;
use nautilus_model::identifiers::{AccountId, TraderId};
use nautilus_okx::{
    common::enums::OKXInstrumentType,
    config::{OKXDataClientConfig, OKXExecClientConfig},
    factories::{OKXDataClientFactory, OKXExecutionClientFactory},
};

let trader_id = TraderId::from("TESTER-001");
let account_id = AccountId::from("OKX-001");

let data_config = OKXDataClientConfig::builder()
    .instrument_types(vec![OKXInstrumentType::Swap])
    .build();

let exec_config = OKXExecClientConfig::builder()
    .trader_id(trader_id)
    .account_id(account_id)
    .instrument_types(vec![OKXInstrumentType::Swap])
    .build();

let log_config = LoggerConfig {
    stdout_level: LevelFilter::Info,
    ..Default::default()
};

let mut node = LiveNode::builder(trader_id, Environment::Live)?
    .with_name("MY-NODE-001".to_string())
    .with_logging(log_config)
    .add_data_client(
        None,
        Box::new(OKXDataClientFactory::new()),
        Box::new(data_config),
    )?
    .add_exec_client(
        None,
        Box::new(OKXExecutionClientFactory::new()),
        Box::new(exec_config),
    )?
    .with_reconciliation(false) // 为简化示例而设置；生产环境请启用
    .with_delay_post_stop_secs(5)
    .build()?;
```

:::warning
出于示例简化的考虑，本示例禁用了对账功能。在生产环境中，请移除
`.with_reconciliation(false)`，以便引擎在启动时使缓存状态与交易场所保持一致。
详情请参见[执行对账（Execution reconciliation）](../concepts/live.md#execution-reconciliation)。
:::

## 添加策略并运行

```rust
use nautilus_model::{identifiers::InstrumentId, types::Quantity};
use nautilus_trading::examples::strategies::{
    GridMarketMaker, GridMarketMakerConfig,
};

let mut config = GridMarketMakerConfig::builder()
    .instrument_id(InstrumentId::from("ETH-USDT-SWAP.OKX"))
    .max_position(Quantity::from("0.10"))
    .num_levels(3)
    .grid_step_bps(100)
    .skew_factor(0.5)
    .requote_threshold_bps(10)
    .expire_time_secs(8)
    .on_cancel_resubmit(true)
    .build();

// OKX 不接受客户端订单 ID 中包含连字符
config.base.use_hyphens_in_client_order_ids = false;

let strategy = GridMarketMaker::new(config);

node.add_strategy(strategy)?;
node.run().await?;
```

节点会持续运行，直到被中断（Ctrl+C）或以编程方式停止。

## 环境变量

OKX 从环境变量中读取 API 凭据。可以使用配合 `dotenvy` 的 `.env` 文件，
也可以直接在 shell 中设置：

```bash
export OKX_API_KEY="your_api_key"
export OKX_API_SECRET="your_api_secret"
export OKX_API_PASSPHRASE="your_passphrase"
```

若要进行模拟交易（demo trading），请在两个配置构建器上都设置
`.environment(OKXEnvironment::Demo)`，并使用来自 OKX 的模拟交易 API 凭据。

每个适配器所需的环境变量都记录在对应交易场所的
[集成指南](../integrations/)中。

## 异步运行时

`LiveNode::run()` 是异步方法，需要 Tokio 运行时。请在主函数上使用
`#[tokio::main]`：

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenvy::dotenv().ok();

    // ... 节点搭建 ...

    node.run().await?;
    Ok(())
}
```

## 适配器示例

大多数适配器都附带了可运行的示例，包含数据测试器和执行测试器：

| 适配器             | 示例目录                                  |
|---------------------|----------------------------------------------------|
| Architect AX        | `crates/adapters/architect_ax/examples/`           |
| Betfair             | `crates/adapters/betfair/examples/`                |
| Binance             | `crates/adapters/binance/examples/`                |
| BitMEX              | `crates/adapters/bitmex/examples/`                 |
| Blockchain          | `crates/adapters/blockchain/examples/`             |
| Bybit               | `crates/adapters/bybit/examples/`                  |
| Coinbase            | `crates/adapters/coinbase/examples/`               |
| Databento           | `crates/adapters/databento/examples/`              |
| Deribit             | `crates/adapters/deribit/examples/`                |
| Derive              | `crates/adapters/derive/examples/`                 |
| dYdX                | `crates/adapters/dydx/examples/`                   |
| Hyperliquid         | `crates/adapters/hyperliquid/examples/`            |
| Interactive Brokers | `crates/adapters/interactive_brokers/examples/`    |
| Kraken              | `crates/adapters/kraken/examples/`                 |
| Lighter             | `crates/adapters/lighter/examples/`                |
| OKX                 | `crates/adapters/okx/examples/`                    |
| Polymarket          | `crates/adapters/polymarket/examples/`             |
| Sandbox             | `crates/adapters/sandbox/examples/`                |
| Tardis              | `crates/adapters/tardis/examples/`                 |
</content>
