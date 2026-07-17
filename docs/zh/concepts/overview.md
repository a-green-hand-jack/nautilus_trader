# 概览（Overview）

## 简介

NautilusTrader 是一个开源的、生产级的、Rust 原生（Rust-native）引擎，
用于构建多资产、多交易场所（multi-venue）的交易系统。

该系统在单一事件驱动（event-driven）架构中涵盖研究、确定性仿真
（deterministic simulation）和实盘执行，其中 Python 作为策略逻辑、
配置和编排的控制平面（control plane）。

这种分离既提供了编译型交易引擎的性能与安全性，又保留了 Python 在
系统组合和策略开发方面的灵活性。对于关键任务型工作负载，交易系统也可以
完全用 Rust 编写。

相同的执行语义（execution semantics）和确定性时间模型在研究系统和
实盘系统中都同样适用。策略可以从研究环境无需修改代码即可部署到生产环境，
实现了研究与实盘的一致性（research-to-live parity），减少了通常会带来
部署风险的差异。

NautilusTrader 与资产类别无关（asset-class-agnostic）。任何提供 REST API
或 WebSocket 数据源的交易场所都可以通过模块化适配器进行集成。当前的集成
涵盖加密货币交易所（CEX 和 DEX）、传统市场（外汇、股票、期货、期权）
以及博彩交易所。

## 特性

- **快速**：Rust 核心，使用 [tokio](https://crates.io/crates/tokio) 实现异步网络通信。
- **可靠**：由 Rust 保证类型安全和线程安全，并支持可选的基于 Redis 的状态持久化。
- **可移植**：可在 Linux、macOS 和 Windows 上运行，支持使用 Docker 部署。
- **灵活**：模块化适配器可集成任意 REST API 或 WebSocket 数据源。
- **高级功能**：有效期（Time in force）类型 `IOC`、`FOK`、`GTC`、`GTD`、`DAY`、`AT_THE_OPEN`、`AT_THE_CLOSE`，高级订单类型和条件触发。执行指令 `post-only`、`reduce-only` 和冰山单（icebergs）。或有订单（Contingency orders），包括 `OCO`、`OUO`、`OTO`。
- **可定制**：支持用户自定义组件，或使用[缓存（cache）](cache.md)和[消息总线（message bus）](message_bus.md)从零搭建整个系统。
- **回测**：可同时对多个交易场所、金融工具和策略进行回测，使用具有纳秒级分辨率的历史报价 tick、成交 tick、K 线（bar）、订单簿及自定义数据。
- **实盘**：研究环境和实盘部署使用完全相同的策略实现。
- **多交易场所**：可同时跨多个交易场所运行做市和跨场所策略。
- **AI 训练**：引擎速度足以训练 AI 交易代理（RL/ES）。

## 为什么选择 NautilusTrader？

交易策略研究通常在 Python 中以向量化方式进行，而生产级交易系统
则单独使用编译型语言以事件驱动架构构建。

NautilusTrader 消除了这种分离。

Rust 原生核心为研究和实盘执行都提供了一个确定性的事件驱动运行时，
而 Python 则作为控制平面。相同的架构、执行语义和时间模型在两种环境中
都同样适用，使策略能够从研究环境迁移到生产环境而无需重新实现。

Python 绑定通过 [PyO3](https://pyo3.rs) 提供，目前正在从 Cython 逐步
迁移过来。安装时无需 Rust 工具链。

## 使用场景

该软件包主要有三种使用场景：

- 在历史数据上对交易系统进行回测（`backtest`）。
- 使用实时数据和虚拟执行模拟交易系统（`sandbox`）。
- 在真实账户或模拟账户上实盘部署交易系统（`live`）。

该代码库提供了一个框架，用于构建实现上述目标的软件层。
默认的 `backtest` 和 `live` 系统实现分别位于对应名称的子包中。
`sandbox` 环境可以使用 sandbox 适配器构建。

:::note

- 所有示例都将使用这些默认的系统实现。
- 我们将交易策略视为端到端交易系统的子组件，这些系统
  包含应用层和基础设施层。

:::

## 分布式

该平台可集成到更大的分布式系统中。
几乎所有的配置和领域对象（domain objects）都使用 JSON、MessagePack 或
Apache Arrow（Feather）进行序列化，以便通过网络通信。

## 通用核心

通用系统核心被所有节点[环境上下文（environment contexts）](architecture.md#environment-contexts)
（`backtest`、`sandbox` 和 `live`）所使用。用户定义的 `Actor`、`Strategy`
和 `ExecAlgorithm` 组件在这些环境上下文中都以一致的方式进行管理。

## 回测

可以直接将数据提供给 `BacktestEngine`，或通过更高层级的 `BacktestNode`
和 `ParquetDataCatalog`，然后以纳秒级分辨率将数据送入系统运行。

## 实盘交易

`TradingNode` 从多个数据客户端和执行客户端摄取数据和事件，支持
模拟/纸上交易账户和真实账户。通过在单个
[事件循环（event loop）](https://docs.python.org/3/library/asyncio-eventloop.html)
上异步运行提供高性能，并可选择使用 [uvloop](https://github.com/MagicStack/uvloop)
实现（适用于 Linux 和 macOS）以进一步提升吞吐量。

## 领域模型

该平台提供了一个交易领域模型，包含各种数值类型，例如
`Price` 和 `Quantity`，以及更复杂的实体，例如 `Order` 和 `Position` 对象，
它们用于聚合多个事件以确定状态。

## 时间戳

所有时间戳均以 UTC 纳秒精度表示。

时间戳字符串遵循 ISO 8601（RFC 3339）格式，小数精度为 9 位（纳秒）
或 3 位（毫秒）（但大多数情况下为纳秒），且始终保留所有位数，包括尾随零。
这些格式可以在日志消息以及对象的调试/显示输出中看到。

时间戳字符串由以下部分组成：

- 始终包含完整的日期部分：`YYYY-MM-DD`。
- 日期与时间部分之间使用 `T` 分隔符。
- 始终为纳秒精度（9 位小数），或在某些情况下（例如 GTD 到期时间）为毫秒精度（3 位小数）。
- 始终使用 `Z` 后缀表示 UTC 时区。

示例：`2024-01-05T15:30:45.123456789Z`

完整规范请参阅 [RFC 3339：互联网上的日期和时间](https://datatracker.ietf.org/doc/html/rfc3339)。

## UUID

该平台使用通用唯一标识符（UUID）第 4 版（RFC 4122）作为唯一标识符。
我们的高性能实现在从字符串解析时使用 `uuid` crate 进行正确性验证，
确保输入的 UUID 符合规范。

一个有效的 UUID v4 由以下部分组成：

- 32 个十六进制数字，分为 5 组显示。
- 各组之间使用连字符分隔：`8-4-4-4-12` 格式。
- 版本 4 标识（由第三组以“4”开头表示）。
- RFC 4122 变体标识（由第四组以“8”、“9”、“a”或“b”开头表示）。

示例：`2d89666b-1a1e-4a75-b193-4eb3b454c757`

完整规范请参阅 [RFC 4122：通用唯一标识符（UUID）URN 命名空间](https://datatracker.ietf.org/doc/html/rfc4122)。

## 数据类型

以下市场数据类型可以进行历史请求，也可以在交易场所/数据提供方支持并
在集成适配器中实现的情况下订阅为实时流：

- `OrderBookDelta`（单条订单簿变更）
- `OrderBookDeltas`（容器类型）
- `OrderBookDepth10`（每一侧固定 10 档深度）
- `QuoteTick`
- `TradeTick`
- `Bar`
- `Instrument`
- `InstrumentStatus`
- `InstrumentClose`

以下 `PriceType` 选项可用于 K 线聚合（bar aggregation）：

- `BID`
- `ASK`
- `MID`
- `LAST`

## K 线聚合（Bar aggregations）

以下 `BarAggregation` 方法可用：

- `MILLISECOND`
- `SECOND`
- `MINUTE`
- `HOUR`
- `DAY`
- `WEEK`
- `MONTH`
- `YEAR`
- `TICK`
- `VOLUME`
- `VALUE`（即美元 K 线，Dollar bars）
- `RENKO`（基于价格的砖形图）
- `TICK_IMBALANCE`
- `TICK_RUNS`
- `VOLUME_IMBALANCE`
- `VOLUME_RUNS`
- `VALUE_IMBALANCE`
- `VALUE_RUNS`

所有列出的聚合方式均已实现用于内部聚合。
信息驱动型（information-driven）聚合需要 `TradeTick` 数据。

价格类型和 K 线聚合方式可以通过 `BarSpecification` 与任意 >= 1 的步长
（step sizes）以任意方式组合。这使得可以为实盘交易聚合出其他形式的 K 线。

## 账户类型

以下账户类型在实盘和回测环境中均可用：

- `Cash` 单币种（基础货币）
- `Cash` 多币种
- `Margin` 单币种（基础货币）
- `Margin` 多币种
- `Betting` 单币种

## 订单类型

以下订单类型可用（如果交易场所支持）：

- `MARKET`
- `LIMIT`
- `STOP_MARKET`
- `STOP_LIMIT`
- `MARKET_TO_LIMIT`
- `MARKET_IF_TOUCHED`
- `LIMIT_IF_TOUCHED`
- `TRAILING_STOP_MARKET`
- `TRAILING_STOP_LIMIT`

## 数值类型

以下数值类型依据编译时使用的
[精度模式（precision mode）](../getting_started/installation.md#precision-mode)，
由 128 位或 64 位原始整数值支撑：

- `Price`
- `Quantity`
- `Money`

### 高精度模式（128 位）

当 `high-precision` 功能标志被**启用**时（默认），数值使用以下规格：

| 类型         | 原始存储    | 最大精度       | 最小值               | 最大值              |
|:-------------|:------------|:--------------|:--------------------|:-------------------|
| `Price`      | `i128`      | 16            | -17,014,118,346,046 | 17,014,118,346,046 |
| `Money`      | `i128`      | 16            | -17,014,118,346,046 | 17,014,118,346,046 |
| `Quantity`   | `u128`      | 16            | 0                   | 34,028,236,692,093 |

### 标准精度模式（64 位）

当 `high-precision` 功能标志被**禁用**时，数值使用以下规格：

| 类型         | 原始存储    | 最大精度       | 最小值               | 最大值              |
|:-------------|:------------|:--------------|:--------------------|:-------------------|
| `Price`      | `i64`       | 9             | -9,223,372,036      | 9,223,372,036      |
| `Money`      | `i64`       | 9             | -9,223,372,036      | 9,223,372,036      |
| `Quantity`   | `u64`       | 9             | 0                   | 18,446,744,073     |
