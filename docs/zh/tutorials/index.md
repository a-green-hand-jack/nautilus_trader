# 教程

演示特定功能和工作流程的分步指南。

:::info
大多数 Python 教程都是 docs
[tutorials 目录](https://github.com/nautechsystems/nautilus_trader/tree/develop/docs/tutorials)
中的 Jupytext percent 格式文件。
您可以直接将它们作为脚本运行，也可以使用 Jupytext 将其作为笔记本打开。
Rust 教程使用页面上展示的命令。
:::

:::tip

- **Latest（最新版）**：基于 `master` 分支构建的文档，对应稳定版本。
  参见 <https://nautilustrader.io/docs/latest/tutorials/>。
- **Nightly（每夜构建版）**：基于 `nightly` 分支构建的文档，对应实验性功能。
  参见 <https://nautilustrader.io/docs/nightly/tutorials/>。

:::

## 推荐学习顺序

刚接触 NautilusTrader？建议按以下顺序学习：

1. [快速入门](../getting_started/quickstart) - 使用合成数据在五分钟内运行您的第一个回测
2. [回测（低层级 API）](../getting_started/backtest_low_level) - 直接使用
   `BacktestEngine`，配合真实市场数据和执行算法
3. [回测（高层级 API）](../getting_started/backtest_high_level) - 使用
   `BacktestNode` 和 Parquet 数据目录进行配置驱动的回测
4. [加载外部数据][loading_external_data] - 将 CSV 或其他外部数据
   加载到 `ParquetDataCatalog` 中（操作指南）
5. [使用 FX Bar 数据回测][backtest_fx_bars] - 结合展期利息模拟的
   外汇 Bar 数据回测
6. 从下方选择一个主题相关的教程

## 回测

| 教程                                                                                 | 描述                                    | 数据          |
|:------------------------------------------------------------------------------------|:-----------------------------------------------|:--------------|
| [使用 FX Bar 数据回测][backtest_fx_bars]                                       | 在外汇 Bar 上进行 EMA 交叉策略并模拟展期。 | 内置数据       |
| [使用订单簿深度数据回测（币安）][backtest_orderbook_binance]         | 基于深度数据的订单簿失衡策略。   | 用户自备 |
| [使用订单簿深度数据回测（Bybit）][backtest_orderbook_bybit]             | 基于深度数据的订单簿失衡策略。   | 用户自备 |

## 数据工作流

有关面向具体任务的数据处理方案，请参见[操作指南](../how_to/)：

| 指南                                                                               | 描述                                       | 数据              |
|:------------------------------------------------------------------------------------|:--------------------------------------------------|:------------------|
| [加载外部数据][loading_external_data]                                      | 将外部数据加载到 `ParquetDataCatalog`。 | 用户自备     |
| [使用 Databento 建立数据目录][data_catalog_databento]                               | 使用 Databento schema 建立数据目录。          | Databento API 密钥 |

## 策略模式

| 教程                                                                            | 描述                                       | 数据              |
|:------------------------------------------------------------------------------------|:--------------------------------------------------|:------------------|
| [基于代理外汇数据的均值回归（AX Exchange）](fx_mean_reversion_ax)             | 在 EURUSD-PERP 上使用布林带（Bollinger Band）均值回归。     | TrueFX 代理数据      |
| [黄金永续合约订单簿失衡（AX Exchange）](gold_book_imbalance_ax)               | 在 XAU-PERP 上进行订单簿失衡交易。                 | Databento API 密钥 |
| [基于死人开关的网格做市（BitMEX）](grid_market_maker_bitmex)     | 在 XBTUSD 上进行带服务器端安全保护的网格做市。        | Tardis.dev        |
| [基于短期订单的链上网格做市（dYdX）](grid_market_maker_dydx) | dYdX v4 永续合约上的网格做市。                    | 用户自备     |

## 期权

| 教程                                                                            | 描述                                       | 数据              |
|:------------------------------------------------------------------------------------|:--------------------------------------------------|:------------------|
| [期权数据与希腊值（Bybit）](options_data_bybit)                               | 流式获取希腊值和期权链快照。         | 实时 API          |
| [Delta 中性期权策略（Bybit）](delta_neutral_options_bybit)               | 使用永续合约进行 Delta 对冲的空头宽跨式策略。      | 实时 API          |
| [Delta 中性期权策略（Derive）](delta_neutral_options_derive)             | 带权利金入场的 Derive ETH 宽跨式对冲策略。    | 实时 API          |

## Rust

| 教程                                                                                    | 描述                                          | 数据                |
|:--------------------------------------------------------------------------------------------|:-----------------------------------------------------|:--------------------|
| [订单簿失衡回测（Betfair）](backtest_book_imbalance_betfair)                        | 在 Betfair L2 数据上运行订单簿失衡 Actor。             | 用户自备       |
| [基于 Databento EQUS NVDA 的 Lighter RWA 组合做市](lighter_rwa_composite_mm) | 在 NVDA-PERP 上进行信号偏斜做市。                       | Databento + Lighter |
| [Hurst/VPIN 方向性策略（Kraken Futures）](hurst_vpin_kraken)                       | 在 PF_XBTUSD 上基于机制过滤的知情流策略。 | Tardis.dev          |

[backtest_fx_bars]: https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/tutorials/backtest_fx_bars.py
[backtest_orderbook_binance]: https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/tutorials/backtest_orderbook_binance.py
[backtest_orderbook_bybit]: https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/tutorials/backtest_orderbook_bybit.py
[loading_external_data]: https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/how_to/loading_external_data.py
[data_catalog_databento]: https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/how_to/data_catalog_databento.py
