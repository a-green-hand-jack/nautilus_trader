# Lighter 入门指南

Lighter 可通过 v2 Rust 引擎使用。你既可以在纯 Rust 项目中使用它，也可以通过 PyO3
绑定从 Python v2 中使用——该绑定将相同的 Rust 数据和执行客户端暴露给 Python 版的
`LiveNode`。

最简捷的路径是先从公开数据入手。等数据订阅工作正常后，再添加执行凭据，然后
添加一个能够提交订单的策略。

## 选择搭建路径

| 路径            | 适用场景                                            | 第一步                                       |
|:------------|:----------------------------------------------------|:-------------------------------------------|
| 纯 Rust     | 你想要一个没有 Python 运行时的编译型应用。               | 复制 Rust 快速启动示例。                       |
| Python v2   | 你想要在 Rust 引擎上运行 Python 脚本。                  | 运行 Python v2 数据测试器（data tester）。       |
| RWA 示例    | 你想要 Databento 信号数据搭配 Lighter 交易。            | 阅读组合式做市教程。                            |

从以下文件开始：

- Rust 快速启动：`examples/quickstarts/lighter-rust-data-client/`。
- Python v2 数据测试器：`python/examples/lighter/data_tester.py`。
- RWA 教程：[组合式做市教程][lighter-rwa-composite-mm]。

Rust 和 Python v2 两条路径都使用以下组件：

- `LighterDataClientConfig` 用于选择主网（mainnet）或测试网（testnet）以及可选的传输设置。
- `LighterExecClientConfig` 新增了交易者/账户 ID，并解析所需凭据。
- `LighterDataClientFactory` 和 `LighterExecutionClientFactory` 负责向 `LiveNode` 注册客户端。
- `DataTester` 和 `ExecTester` 在你编写自定义策略之前，提供了冒烟测试用的 actor。

## 纯 Rust 起始示例

将快速启动示例复制到你自己的工作区：

```bash
cp -R examples/quickstarts/lighter-rust-data-client ~/lighter-rust-data-client
cd ~/lighter-rust-data-client
cargo run
```

这段代码会构建一个 `LiveNode`，注册 Lighter 数据客户端，添加一个
`DataTester`，并连接到测试网的公开数据流。使用 Ctrl+C 停止。

核心设置使用构建器（builder）模式，会为你自动填充可选默认值：

```rust
let data_config = LighterDataClientConfig::builder()
    .environment(LighterEnvironment::Testnet)
    .build();

let mut node = LiveNode::builder(trader_id, Environment::Live)?
    .with_name("LIGHTER-DATA-STARTER-001".to_string())
    .add_data_client(
        None,
        Box::new(LighterDataClientFactory::new()),
        Box::new(data_config),
    )?
    .build()?;
```

数据链路正常工作之后，可在调用 `.build()` 之前向构建器添加执行客户端：

```rust
let exec_config = LighterExecClientConfig::builder()
    .trader_id(trader_id)
    .account_id(account_id)
    .environment(LighterEnvironment::Testnet)
    .build();

let mut node = LiveNode::builder(trader_id, Environment::Live)?
    .with_name("LIGHTER-EXEC-STARTER-001".to_string())
    .add_data_client(
        None,
        Box::new(LighterDataClientFactory::new()),
        Box::new(data_config),
    )?
    .add_exec_client(
        None,
        Box::new(LighterExecutionClientFactory::new()),
        Box::new(exec_config),
    )?
    .build()?;
```

如需执行交易，请在连接之前设置对应的环境变量：

```bash
export LIGHTER_TESTNET_ACCOUNT_INDEX="123456"
export LIGHTER_TESTNET_API_KEY_INDEX="0"
export LIGHTER_TESTNET_API_SECRET="your-lighter-api-secret"
```

主网环境请使用 `LIGHTER_ACCOUNT_INDEX`、`LIGHTER_API_KEY_INDEX` 和
`LIGHTER_API_SECRET`。

## Python v2 起始示例

Python v2 通过 PyO3 使用 Rust 引擎。请在源代码检出目录之外安装 Python v2
开发版 wheel，或在运行以下示例之前从源代码构建 v2 软件包。详见
[Python v2 安装说明][python-v2-install]。

在已安装 Python v2 的源代码检出目录中：

```bash
cd python
.venv/bin/python examples/lighter/data_tester.py --lighter-environment testnet
```

该命令会构建节点并随即退出。传入 `--run` 参数以建立连接：

```bash
.venv/bin/python examples/lighter/data_tester.py \
    --lighter-environment testnet \
    --instrument BTC-PERP.LIGHTER \
    --run
```

该 Python 脚本与 Rust 版的设置相对应：

```python
builder = LiveNode.builder(
    "LIGHTER-DATA-TESTER-001",
    TraderId.from_str("TESTER-001"),
    Environment.LIVE,
).add_data_client(
    None,
    LighterDataClientFactory(),
    LighterDataClientConfig(environment=LighterEnvironment.TESTNET),
)
```

只有在数据测试器正常工作之后，再使用执行测试器：

```bash
.venv/bin/python examples/lighter/exec_tester.py \
    --lighter-environment testnet \
    --instrument DOGE-PERP.LIGHTER
```

与数据测试器类似，该命令会构建节点并随即退出。传入 `--run` 参数以在
模拟运行（dry-run）模式下建立连接，再加上 `--live-orders` 才会提交真实订单。

## 转向编写策略

上述起始路径已经验证了客户端接线（wiring）、数据订阅和凭据查找是否正常工作。
下一步是用策略替换掉测试器：

- 使用[编写策略（Rust）](write_rust_strategy.md)编写纯 Rust 策略。
- 使用 `python/examples/lighter/nvda_composite_mm.py` 实现 Python v2 版的
  节点接线，搭配内置的 Rust `CompositeMarketMaker` 策略。
- 当你需要完整的 Databento 信号搭建时，请使用
  [在 Lighter RWA 上进行组合式做市][lighter-rwa-composite-mm]。

:::warning
当你将 `DRY_RUN` 设为 `false` 时，Rust 执行示例可以提交真实订单；当你传入
`--run --live-orders` 时，Python 执行示例可以提交真实订单。请先在测试网上
运行，或使用能被接受的最小下单数量，并在运行前确认好金融工具、环境、账户
索引、API key 索引以及私钥。
:::

如需紧急清仓，可运行 `cargo run --bin lighter-flatten -p nautilus-lighter`，
该命令会取消未成交订单并平掉配置的 Lighter 账户中的仓位。使用前请仔细检查，
因为它会扫描整个账户，在标准的每分钟 60 次请求的配额限制下可能耗时数分钟，
并且当该账户涉及更广泛的敞口（exposure）时，会影响不止一个策略或市场。

[lighter-rwa-composite-mm]: ../tutorials/lighter_rwa_composite_mm.md
[python-v2-install]: ../getting_started/installation.md#python-v2-branch-development-wheels
</content>
