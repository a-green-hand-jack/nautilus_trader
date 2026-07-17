# 概念（Concepts）

这些指南介绍了 NautilusTrader 的核心组件、架构和设计。

## 概览（Overview）

平台的主要特性和预期使用场景。

## 架构（Architecture）

支撑该平台的原则、结构和设计。

## Actor

`Actor` 是与交易系统交互的基础组件。
涵盖其能力和实现细节。

## 策略（Strategies）

如何使用 `Strategy` 组件实现交易策略。

## 金融工具（Instruments）

可交易资产和合约的金融工具定义。

## 合成工具（Synthetics）

用户自定义的金融工具，其价格通过对成分金融工具价格计算数值表达式得出。

## 连续期货（Continuous Futures）

通过显式的展期表（roll table）将连续的期货合约拼接为一个经过调整的 K 线（bar）序列，
包括四种调整模式、请求与订阅流程，以及以 K 线中点为界的展期边界策略。

## 数值类型（Value Types）

平台中使用的不可变数值类型（`Price`、`Quantity`、`Money`），
包括它们的算术行为、精度处理和特定类型的约束。

## 数据（Data）

交易领域中的内置数据类型，以及如何使用自定义数据。

## 事件（Events）

驱动系统运行的事件类型：订单事件、仓位事件、账户
事件和时间事件。涵盖处理器分发、从订单成交到仓位事件的因果链，
以及订单到仓位的追踪。

## 事件溯源（Event Sourcing）

用于记录影响状态的消息的持久化事件存储日志，包括捕获边界、
关联头（correlation headers）、重放模式、恢复锚点以及验证器行为。

## 期权（Options）

期权金融工具类型、由交易场所（venue）提供的希腊值（Greeks）流式推送、
带行权价范围过滤的期权链订阅，以及快照聚合。

## 希腊值（Greeks）

来自两条路径的期权希腊值（delta、gamma、vega、theta）：通过 Rust/PyO3 的
`OptionGreeks` 类型获取由交易场所提供的实时希腊值，以及使用本地
`GreeksCalculator` 进行 Black-Scholes 计算，支持冲击情景（shock scenarios）、
beta 加权和投资组合聚合。

## 自定义数据（Custom Data）

自定义数据系统在 Python 和 Rust 中的工作方式：注册、持久化、
Arrow 编码，以及通过 actor 和策略进行的运行时路由。

## 订单簿（Order Book）

高性能订单簿、自有订单跟踪、用于净流动性的过滤视图，以及二元市场支持。

## 执行（Execution）

同时跨多个策略和交易场所（每个实例）进行交易执行和订单管理，
包括涉及的组件以及执行消息（命令和事件）的流转。

## 订单（Orders）

可用的订单类型、支持的执行指令、高级订单类型以及模拟订单（emulated orders）。

## 仓位（Positions）

仓位生命周期、从订单成交聚合仓位、盈亏（PnL）计算，以及用于净额（netting）
OMS 配置的仓位快照。

## 缓存（Cache）

`Cache` 是所有交易相关数据的中央内存存储。
涵盖其能力和最佳实践。

## 消息总线（Message Bus）

`MessageBus` 支持组件之间的解耦消息传递，支持点对点、
发布/订阅以及请求/响应模式。

## 账户（Accounting）

账户类型（现金、保证金、投注型），`AccountBalance` 和 `MarginBalance`
数据模型，按金融工具与账户整体的保证金作用域区别，策略查询
API，内置保证金模型，以及各实盘交易场所间的适配器约定。

## 投资组合（Portfolio）

`Portfolio` 跟踪所有策略和金融工具的仓位，提供持仓、
风险敞口和绩效的统一视图。

## 报告（Reports）

执行报告、投资组合分析、盈亏核算以及回测后分析。

## 日志（Logging）

用 Rust 实现的高性能日志系统，适用于回测和实盘交易。

## 回测（Backtesting）

回测 API、数据与交易场所设置、执行时序、成交模拟、
账户、资金和保证金配置。

## 可视化（Visualization）

用于分析回测结果的交互式分析报告（tearsheet），包括图表、主题、
自定义选项，以及通过可扩展的图表注册表实现的自定义可视化。

## 配置（Configuration）

配置结构体在 Python 和 Rust 中的工作方式：默认值解析、`T` 与
`Option<T>` 的约定、构建器模式，以及适配器和引擎共享的常见字段。

## 实盘交易（Live Trading）

无需修改代码即可将回测策略部署到实时环境，以及回测与
实盘交易之间的关键差异。

## 插件（Plugins）

`nautilus-plugin` crate：插件产物身份、清单契约，以及
独立编译的 Rust cdylib 的 C-ABI 边界类型。

## 适配器（Adapters）

为数据提供方和交易场所开发集成适配器的要求与最佳实践。

## Rust

直接使用 `crates/` 实现，在纯 Rust 中编写 actor、策略，
并运行回测和实盘交易。

## 确定性仿真测试（Deterministic Simulation Testing, DST）

可重放种子（seed-replayable）执行的确定性契约、实现该契约的源代码级接缝（seams）、
强制执行的预提交钩子（pre-commit hook），以及已知的适用范围边界。

:::note
如果这些指南与 API 参考文档之间存在差异，以 API 参考文档为准。
:::
