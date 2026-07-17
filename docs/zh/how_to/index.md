# How-to 指南

针对常见任务的目标导向型示例（recipe）。每篇指南都假设你已经熟悉 Nautilus 的
相关概念，并专注于实现某个具体目标。

刚接触 Nautilus？请先从[快速入门](../getting_started/)路径和[教程](../tutorials/)开始。

## 数据工作流

| 指南                                                   | 说明                                            |
|:------------------------------------------------------|:-----------------------------------------------|
| [加载外部数据][loading_external_data]                  | 将 CSV 数据加载到 Parquet 数据目录中。            |
| [搭配 Databento 使用数据目录][data_catalog_databento]  | 使用 Databento 市场数据搭建数据目录。              |

## 实盘交易

| 指南                                                           | 说明                                                     |
|:--------------------------------------------------------------|:----------------------------------------------------------|
| [配置实盘交易节点](configure_live_trading)                     | 设置 TradingNodeConfig、执行引擎与交易场所。               |
| [Lighter 入门指南](get_started_lighter)                        | 从 Rust 或 Python v2 启动 Lighter。                        |

## Rust

| 指南                                                       | 说明                                                    |
|:----------------------------------------------------------|:-------------------------------------------------------|
| [编写 Actor（Rust）](write_rust_actor)                     | 构建带有订阅和处理器的数据 actor。                        |
| [编写策略（Rust）](write_rust_strategy)                    | 构建带有订单管理功能的策略。                              |
| [运行回测（Rust）](run_rust_backtest)                      | 使用 BacktestEngine 或搭配数据目录的 BacktestNode。      |
| [运行实盘交易（Rust）](run_rust_live_trading)               | 使用 LiveNode 连接交易场所。                              |

[loading_external_data]: https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/how_to/loading_external_data.py
[data_catalog_databento]: https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/how_to/data_catalog_databento.py
</content>
