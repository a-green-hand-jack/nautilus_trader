# 回测

回测（Backtesting）使用与实盘交易相同的核心系统组件，针对历史数据进行模拟交易：内置引擎、`Cache`、
[消息总线（MessageBus）](../message_bus.md)、`Portfolio`、[Actors](../actors.md)、
[策略（Strategies）](../strategies.md)、[执行算法（Execution Algorithms）](../execution.md)，以及
用户自定义模块。

`BacktestEngine` 处理历史数据流。当数据流耗尽时，引擎会生成结果和性能指标以供分析。NautilusTrader
为回测提供两种 API 级别：

| API 级别 | 使用场景                                                                 |
|----------|--------------------------------------------------------------------------|
| 高层级   | 你希望使用 `BacktestNode`、配置对象、数据目录（data catalog）以及批量运行。 |
| 低层级   | 你希望直接控制 `BacktestEngine` 并手动配置组件。                          |

## 阅读指南

生成的侧边栏可能会按字母顺序对这些页面排序。若要按顺序完整阅读本节内容，请遵循以下顺序：

| 步骤 | 页面                                                    | 用途                                              |
|------|----------------------------------------------------------|---------------------------------------------------|
| 1    | [API 与重复运行](apis-and-runs.md)                       | 选择 API 级别、加载数据并运行批量任务。            |
| 2    | [数据与交易场所](data-and-venues.md)                      | 将数据粒度与交易场所（venue）的 `book_type` 匹配。 |
| 3    | [执行流程](execution-flow.md)                             | 理解排序、计时器和成交 ID。                        |
| 4    | [成交价格与撮合](fill-prices-and-matching.md)             | 理解确定性的撮合行为。                             |
| 5    | [基于成交明细的执行](trade-execution.md)                  | 使用成交明细（trade tick）、主动方向和队列。       |
| 6    | [基于 K 线的执行](bar-execution.md)                       | 使用 K 线（bar）、OHLC 排序和 K 线时间戳。         |
| 7    | [成交模型](fill-models.md)                                | 配置滑点和概率性成交。                             |
| 8    | [账户与保证金](accounts-and-margin.md)                    | 配置资金、余额和保证金模型。                       |

## 相关指南

- [策略（Strategies）](../strategies.md) - 开发用于回测的策略。
- [可视化（Visualization）](../visualization.md) - 根据回测结果生成绩效报表（tearsheet）。
- [报告（Reports）](../reports.md) - 分析回测绩效数据。
