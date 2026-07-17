# 配置实盘交易节点

设置 `TradingNode` 以实现与真实市场的实盘连接。有关实盘交易架构和状态对账
（reconciliation）的背景信息，请参见[实盘交易](../concepts/live.md)概念指南。

:::danger[不推荐在 Jupyter notebook 中运行实盘交易]
请勿在 Jupyter notebook 中运行实盘交易节点。事件循环冲突和运维风险使其不适合这样使用：

- Jupyter 运行着自己的 asyncio 事件循环，这与 `TradingNode` 的事件循环存在冲突。
- 类似 `nest_asyncio` 这样的变通方案不具备生产级质量。
- 单元格（cell）可能乱序执行，内核可能崩溃，状态可能丢失。
- Notebook 缺少生产交易所需的日志记录、监控和优雅停机（graceful shutdown）能力。

请将 Jupyter 用于回测、分析和实验。对于实盘交易，请以独立的 Python 脚本或服务的
形式运行节点。
:::

:::warning[每个进程只运行一个 TradingNode]
由于存在全局单例状态，不支持在同一进程中并发运行多个 `TradingNode` 实例。
可以向单个节点添加多个策略，或者在不同进程中运行多个节点以实现并行执行。

详情请参见[进程与线程](../concepts/architecture.md#processes-and-threads)。
:::

:::warning[不要阻塞事件循环]
在事件循环线程上运行的用户代码（策略回调、actor 处理器、`on_event` 方法）必须
快速返回。这一点对 Python 和 Rust 都适用。诸如模型推理、繁重计算或同步 I/O
等阻塞操作，会导致成交遗漏、数据过期和订单提交延迟。请将长时间运行的工作
交由执行器（executor）或独立的线程/进程处理。
:::

:::info[平台差异]
Windows 的信号处理方式与类 Unix 系统不同。如果你在 Windows 上运行，请阅读
[Windows 信号处理](#windows-signal-handling)一节，了解优雅停机行为和 Ctrl+C
（SIGINT）支持的相关指导。
:::

## TradingNodeConfig

`TradingNodeConfig` 继承自 `NautilusKernelConfig`，并新增了实盘专用的选项。
关于配置结构体如何处理默认值和 `Option<T>` 语义的背景信息，请参见
[配置（Configuration）](../concepts/configuration.md)概念指南。

```python
from nautilus_trader.config import TradingNodeConfig

config = TradingNodeConfig(
    trader_id="MyTrader-001",

    # 组件配置
    cache=CacheConfig(),
    message_bus=MessageBusConfig(),
    data_engine=LiveDataEngineConfig(),
    risk_engine=LiveRiskEngineConfig(),
    exec_engine=LiveExecEngineConfig(),
    portfolio=PortfolioConfig(),

    # 客户端配置
    data_clients={
        "BINANCE": BinanceDataClientConfig(),
    },
    exec_clients={
        "BINANCE": BinanceExecClientConfig(),
    },
)
```

### 核心配置参数

| 设置                      | 默认值        | 说明                                          |
|--------------------------|--------------|---------------------------------------------|
| `trader_id`              | "TRADER-001" | 唯一的交易者标识符（名称—标签格式）。            |
| `instance_id`            | `None`       | 可选的唯一实例标识符。                          |
| `timeout_connection`     | 60.0         | 连接超时时间（秒）。                            |
| `timeout_reconciliation` | 30.0         | 对账（reconciliation）超时时间（秒）。          |
| `timeout_portfolio`      | 10.0         | 投资组合初始化超时时间。                        |
| `timeout_disconnection`  | 10.0         | 断开连接超时时间。                              |
| `timeout_post_stop`      | 10.0         | 停止后清理超时时间。                            |

### 缓存数据库配置

以 Rust 为核心的实盘系统将缓存行为保留在 `CacheConfig` 中，将 Redis 连接设置
保留在 `RedisCacheConfig` 中。

```rust
use nautilus_common::{
    cache::{CacheConfig, database::CacheDatabaseFactory},
    enums::SerializationEncoding,
};
use nautilus_infrastructure::redis::cache::RedisCacheConfig;

let config = CacheConfig {
    encoding: SerializationEncoding::MsgPack,
    timestamps_as_iso8601: true,
    buffer_interval_ms: Some(100),
    flush_on_start: false,
    ..Default::default()
};

let database = RedisCacheConfig {
    host: Some("localhost".to_string()),
    port: Some(6379),
    username: Some("nautilus".to_string()),
    password: Some("pass".to_string()),
    connection_timeout: 2,
    response_timeout: 2,
    ..Default::default()
};

let cache_database = database
    .create(trader_id, instance_id, config.clone())
    .await?;
```

请在构建以 Rust 为核心的节点之后、启动之前挂载适配器。当 `exec_engine.load_cache`
启用时（这是默认设置），节点会在对账之前先从数据库恢复状态。

```rust
let node_config = LiveNodeConfig {
    trader_id,
    ..Default::default()
};
let mut node = LiveNode::build("LiveNode".to_string(), Some(node_config))?;
node.set_cache_database(cache_database)?;
node.run().await?;
```

将 `CacheConfig.flush_on_start` 设为 `true`，可以清空已挂载的后端，而不是从中恢复数据。
Python v2 版本的 `LiveNode` 目前尚未暴露直接注入缓存后端的能力。

### MessageBus 配置

消息总线的行为保留在 `MessageBusConfig` 中。Redis 连接设置位于
`RedisMessageBusConfig` 中，它通过 `MessageBusBackingFactory` 构建后端。

```rust
use nautilus_common::{
    enums::SerializationEncoding,
    msgbus::{backing::MessageBusBackingFactory, config::MessageBusConfig},
};
use nautilus_infrastructure::redis::msgbus::{RedisMessageBusConfig, RedisMessageBusFactory};

let config = MessageBusConfig {
    encoding: SerializationEncoding::Json,
    timestamps_as_iso8601: true,
    use_instance_id: false,
    types_filter: Some(vec!["QuoteTick".to_string(), "TradeTick".to_string()]),
    stream_per_topic: false,
    autotrim_mins: Some(30),
    heartbeat_interval_secs: Some(1),
    ..Default::default()
};

let backing = RedisMessageBusConfig {
    connection_timeout: 2,
    response_timeout: 2,
    ..Default::default()
};

let message_bus_backing = RedisMessageBusFactory::new(backing).create(
    trader_id,
    instance_id,
    config.clone(),
)?;
```

## 多交易场所配置

一个节点可以连接多个交易场所。以下示例同时为 Binance 配置了现货和期货市场：

```python
config = TradingNodeConfig(
    trader_id="MultiVenue-001",

    # 针对不同市场类型的多个数据客户端
    data_clients={
        "BINANCE_SPOT": BinanceDataClientConfig(
            account_type=BinanceAccountType.SPOT,
            environment=BinanceEnvironment.LIVE,
        ),
        "BINANCE_FUTURES": BinanceDataClientConfig(
            account_type=BinanceAccountType.USDT_FUTURES,
            environment=BinanceEnvironment.LIVE,
        ),
    },

    # 对应的执行客户端
    exec_clients={
        "BINANCE_SPOT": BinanceExecClientConfig(
            account_type=BinanceAccountType.SPOT,
            environment=BinanceEnvironment.LIVE,
        ),
        "BINANCE_FUTURES": BinanceExecClientConfig(
            account_type=BinanceAccountType.USDT_FUTURES,
            environment=BinanceEnvironment.LIVE,
        ),
    },
)
```

## ExecutionEngine 配置

`LiveExecEngineConfig` 控制订单处理、执行事件以及交易场所的对账。完整细节
请参见 [API 参考](/docs/python-api-latest/config.html#nautilus_trader.live.config.LiveExecEngineConfig)。

### 对账（Reconciliation）

恢复遗漏的订单和仓位事件，使系统状态与交易场所保持一致。

| 设置                             | 默认值   | 说明                                                                     |
|---------------------------------|---------|---------------------------------------------------------------------------------|
| `reconciliation`                | True    | 在启动时启用对账，使内部状态与交易场所保持一致。      |
| `reconciliation_lookback_mins`  | None    | 为对账未缓存的状态，向前追溯请求过去事件的时间范围（分钟）。   |
| `reconciliation_instrument_ids` | None    | 需要进行对账的金融工具 ID 列表。                                    |
| `filtered_client_order_ids`     | None    | 在对账过程中跳过的客户端订单 ID（用于处理交易场所侧的重复项）。     |

详情请参见[执行对账（Execution reconciliation）](../concepts/live.md#execution-reconciliation)。

### 订单过滤

控制系统处理哪些订单事件和报告，以防止多个交易节点之间产生冲突。

| 设置                                | 默认值   | 说明                                                                   |
|------------------------------------|---------|-------------------------------------------------------------------------------|
| `filter_unclaimed_external_orders` | False   | 丢弃未被认领的外部订单，使其不会影响策略。            |
| `filter_position_reports`          | False   | 丢弃仓位状态报告。当多个节点共用同一账户交易时很有用。   |

:::note[订单标记行为]
对账会按来源为订单打上标记：

- **`VENUE` 标记**：在交易场所处发现的外部订单（通过本系统之外的方式下单）。
- **`RECONCILIATION` 标记**：为对齐仓位差异而生成的合成订单。

启用 `filter_unclaimed_external_orders` 后，只有带 `VENUE` 标记的订单会被过滤。
带 `RECONCILIATION` 标记的订单永远不会被过滤，因此仓位对齐总能成功完成。
:::

### 持续对账（Continuous reconciliation）

持续对账通过检查在途（in-flight）订单、轮询未成交订单、检查仓位状态以及审计
自有订单簿，使运行时的执行状态在启动后保持一致。可通过以下设置配置该循环。
有关运行时状态转换规则、重试协调机制以及注意事项，请参见
[运行时检查（Runtime checks）](../concepts/live.md#runtime-checks)。

| 设置                                  | 默认值          | 说明                                                                                      |
|--------------------------------------|----------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `inflight_check_interval_ms`         | 2,000&nbsp;ms  | 检查在途订单状态的频率。设为 0 表示禁用。                                  |
| `inflight_check_threshold_ms`        | 5,000&nbsp;ms  | 在途订单触发交易场所状态检查前的等待时间。若为同机房部署，可适当调低。                |
| `inflight_check_retries`             | 5&nbsp;retries | 向交易场所验证在途订单的重试次数。                                                      |
| `open_check_interval_secs`           | None           | 向交易场所检查未成交订单的频率（秒）。None 或 0.0 表示禁用。建议：5-10 秒。 |
| `open_check_open_only`               | True           | 为 true 时仅查询未成交订单；为 false 时获取完整历史（较消耗资源）。          |
| `open_check_lookback_mins`           | 60&nbsp;min    | 订单状态轮询的回溯窗口（分钟）。仅处理该时间窗口内被修改过的订单。     |
| `open_check_threshold_ms`            | 5,000&nbsp;ms  | 在针对交易场所差异采取行动之前，距上次缓存事件所需的最短时间。                       |
| `open_check_missing_retries`         | 5&nbsp;retries | 在对符合条件的订单执行针对性的“未找到”解析之前的最大重试次数。                            |
| `max_single_order_queries_per_cycle` | 10             | 每个周期内单订单查询次数的上限，用于防止耗尽速率限额。                           |
| `single_order_query_delay_ms`        | 100&nbsp;ms    | 单订单查询之间的延迟（毫秒），用于避免触发速率限制。                                    |
| `reconciliation_startup_delay_secs`  | 10.0&nbsp;s    | 启动对账*完成后*到持续检查开始之前的延迟（秒）。                                   |
| `own_books_audit_interval_secs`      | None           | 将自有订单簿与公开订单簿进行审计比对的时间间隔（秒）。                        |
| `position_check_interval_secs`       | None           | 仓位一致性检查的时间间隔（秒）。发现差异时会查询遗漏的成交记录。None 表示禁用。建议：30-60 秒。 |
| `position_check_lookback_mins`       | 60&nbsp;min    | 在仓位出现差异时，查询成交报告的回溯窗口（分钟）。                     |
| `position_check_threshold_ms`        | 5,000&nbsp;ms  | 在针对仓位差异采取行动之前，距上次本地活动所需的最短时间。                  |
| `position_check_retries`             | 3&nbsp;retries | 引擎针对某个金融工具/账户的差异停止重试之前的最大尝试次数。超过该次数后会记录错误日志，且在该差异消失之前不再主动进行对账。 |

:::warning

- **`open_check_lookback_mins`**：不要将其调低于 60 分钟。过短的时间窗口会
  因订单超出查询范围而触发错误的“订单丢失”判定。
- **`open_check_threshold_ms`**：如果交易场所时间戳相对本地时钟存在延迟，应
  适当增大该值，以避免刚更新的订单被过早标记为丢失。
- **`reconciliation_startup_delay_secs`**：在生产环境中不要将其调低于 10 秒。
  这段延迟可以让系统在启动对账完成后有时间稳定下来，再开始持续检查。

:::

### 其他选项

| 设置                                | 默认值   | 说明                                                                                     |
|------------------------------------|---------|---------------------------------------------------------------------------------------------------------------------|
| `allow_overfills`                  | False   | 允许成交数量超过订单数量（会记录警告日志）。当对账与成交出现竞态时很有用。    |
| `generate_missing_orders`          | True    | 在对账过程中生成限价单以对齐仓位差异（策略为 `EXTERNAL`，标记为 `RECONCILIATION`）。 |
| `snapshot_orders`                  | False   | 在订单事件发生时进行订单快照。                                                           |
| `snapshot_positions`               | False   | 在仓位事件发生时进行仓位快照。                                                             |
| `snapshot_positions_interval_secs` | None    | 仓位快照之间的时间间隔（秒）。                                                                  |
| `debug`                            | False   | 为执行相关逻辑启用调试日志。                                                             |

### 内存管理

定期清理已关闭的订单、已平仓的仓位以及账户事件的内存缓存，从而在长期运行
或高频交易（HFT）场景下将内存占用控制在合理范围内。

| 设置                                    | 默认值 | 说明                                                                        |
|----------------------------------------|---------|--------------------------------------------------------------------------------|
| `purge_closed_orders_interval_mins`    | None    | 从内存中清理已关闭订单的频率（分钟）。建议：10-15 分钟。    |
| `purge_closed_orders_buffer_mins`      | None    | 订单关闭后需要经过多久（分钟）才会被清理。建议：60 分钟。    |
| `purge_closed_positions_interval_mins` | None    | 从内存中清理已平仓仓位的频率（分钟）。建议：10-15 分钟。 |
| `purge_closed_positions_buffer_mins`   | None    | 仓位平仓后需要经过多久（分钟）才会被清理。建议：60 分钟。  |
| `purge_account_events_interval_mins`   | None    | 从内存中清理账户事件的频率（分钟）。建议：10-15 分钟。   |
| `purge_account_events_lookback_mins`   | None    | 账户事件需要存在多久（分钟）才会被清理。建议：60 分钟。    |
| `purge_from_database`                  | False   | 同时从后端数据库（Redis/PostgreSQL）中删除。**请谨慎使用**。    |

设置某个时间间隔即可启用相应的清理循环；不设置则禁用该调度和删除行为。除非
`purge_from_database` 为 true，否则数据库记录不受影响。每个清理循环都会调用
[缓存（Cache）](../concepts/cache.md)中描述的缓存 API。

### 队列管理

| 设置                              | 默认值 | 说明                                                                     |
|----------------------------------|---------|---------------------------------------------------------------------------------|
| `qsize`                          | 100,000 | 内部队列缓冲区的大小。                                                 |
| `graceful_shutdown_on_exception` | False   | 在队列处理出现意外异常（而非用户代码异常）时优雅停机。 |

## 策略配置

完整的参数列表请参见 `StrategyConfig`
[API 参考](/docs/python-api-latest/config.html#nautilus_trader.trading.config.StrategyConfig)。

### 标识

| 设置           | 默认值 | 说明                                                   |
|----------------|---------|---------------------------------------------------------------|
| `strategy_id`  | None    | 唯一的策略标识符。                                   |
| `order_id_tag` | None    | 附加到该策略订单 ID 上的唯一标签。             |

### 订单管理

| 设置                         | 默认值 | 说明                                                                               |
|-----------------------------|---------|---------------------------------------------------------------------------------------------|
| `oms_type`                  | None    | 用于仓位 ID 和订单处理的[OMS 类型](../concepts/execution#oms-configuration)。 |
| `use_uuid_client_order_ids` | False   | 客户端订单 ID 使用 UUID4 值。                                                    |
| `external_order_claims`     | None    | 该策略负责认领的外部订单和对账活动所对应的金融工具 ID。    |
| `manage_contingent_orders`  | False   | 自动管理 OTO、OCO 和 OUO 连带订单（contingent order）。                                 |
| `manage_gtd_expiry`         | False   | 管理订单的 GTD（Good-Till-Date）过期时间。                                                       |

## Windows 信号处理

:::warning
Windows：asyncio 事件循环未实现 `loop.add_signal_handler`。因此，`TradingNode`
在 Windows 上无法通过 asyncio 接收 OS 信号。请使用 Ctrl+C（SIGINT）处理方式或
以编程方式关闭；在 Windows 上不应期望有对等的 SIGTERM 行为。
:::

在 Windows 上，asyncio 事件循环未实现 `loop.add_signal_handler`，因此类 Unix 的
信号集成方式不可用。`TradingNode` 在 Windows 上无法通过 asyncio 接收 OS 信号，
除非你主动介入，否则不会优雅停机。

推荐做法：

- 用 `try/except KeyboardInterrupt` 包裹 `run`，并在捕获后依次调用
  `node.stop()` 和 `node.dispose()`。Ctrl+C 会在主线程中触发
  `KeyboardInterrupt`，从而提供一条干净的资源释放路径。
- 以编程方式发布 `ShutdownSystem` 命令（或在某个 actor/组件中调用
  `shutdown_system(...)`），以触发相同的停机流程。

出现“inflight check loop task still pending”这条提示信息，是因为常规的优雅停机
流程未被触发。相关跟踪见
[#2785](https://github.com/nautechsystems/nautilus_trader/issues/2785)。

v2 版本的 `LiveNode` 在其 Rust 运行循环中会处理 Ctrl+C（SIGINT），在 Unix 上
还会处理 SIGTERM。Python v2 桥接层也会将 SIGINT 路由到相同的停机路径，
从而使运行器（runner）和任务能够干净地关闭。

Windows 上的示例模式：

```python
try:
    node.run()
except KeyboardInterrupt:
    pass
finally:
    try:
        node.stop()
    finally:
        node.dispose()
```
</content>
