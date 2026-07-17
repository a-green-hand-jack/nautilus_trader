# 快速入门

## 1. 安装

搭建一个 Python 3.12-3.14 环境，并安装包：

```bash
pip install -U nautilus_trader
```

有关平台支持、源码构建和 Docker 镜像的信息，请参见[安装指南](installation)。

## 2. 运行快速启动示例

[快速启动（Quickstart）](quickstart)使用合成数据，在五分钟内完成你的第一个回测。
无需下载数据，无需搭建数据目录（catalog）。

所有入门教程都使用一个简单的 EMA 交叉策略，这是刻意为之——交易逻辑本身并非重点。
这些教程旨在讲解引擎的运行方式：数据加载、交易场所模拟、订单生命周期以及报告生成。
在理解了引擎的运作机制之后，[教程（tutorials）](../tutorials/)会介绍其他不同的策略
（均值回归、订单簿失衡、网格做市）。

## 3. 选择你的路径

- **回测**——先学习下面介绍的两个 API 层级，然后阅读[教程](../tutorials/)以了解策略模式的完整流程。
- **实盘交易**——请参见[配置实盘交易节点](../how_to/configure_live_trading.md) how-to 指南，
  以及[集成（Integrations）](../integrations/)以了解所支持的交易场所。
- **数据工作流**——请参见 [how-to 指南](../how_to/)以了解如何加载外部数据以及搭建 Parquet 数据目录。
- **构建适配器**——请参见[开发者指南](../developer_guide/)。

## 回测 API 层级

NautilusTrader 提供两个层级的回测 API：

| API 层级                                        | 入口点          | 适用场景                                                          |
|:-----------------------------------------------|:----------------|:------------------------------------------------------------------|
| [低阶 API](backtest_low_level)                  | `BacktestEngine`| 直接访问各组件，适合库开发                                          |
| [高阶 API](backtest_high_level)                 | `BacktestNode`  | 生产级工作流，更易于过渡到实盘交易（推荐）                            |

高阶 API 需要基于 Parquet 的数据目录。低阶 API 可以直接使用内存中的数据，但没有通往
实盘交易的路径。

:::warning[每个进程只运行一个节点]
由于存在全局单例状态，不支持在同一进程中并发运行多个 `BacktestNode` 或 `TradingNode`
实例。支持顺序执行，即在每次运行之间进行正确的资源释放（disposal）。

详情请参见[进程与线程](../concepts/architecture.md#processes-and-threads)。
:::

如需选择合适的 API 层级，请参见[回测（Backtesting）](../concepts/backtesting/)概念指南。

## 仓库中的示例

在线文档展示的只是示例的一个子集。要查看完整的示例集合，请参见 GitHub 上的仓库：

| 目录                                                                                                       | 内容                                                            |
|:---------------------------------------------------------------------------------------------------------|:--------------------------------------------------------------|
| [examples/](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples)                     | 可完整独立运行的 Python 示例                                     |
| [docs/tutorials/](../tutorials/)                                                                         | 展示常见工作流的教程                                              |
| [docs/concepts/](../concepts/)                                                                           | 带有代码片段的概念指南，用于说明关键特性                             |
| [nautilus_trader/examples/](../../nautilus_trader/examples/)                                              | 纯 Python 编写的策略、指标及执行算法（exec algo）示例               |
| [tests/unit_tests/](../../tests/unit_tests/)                                                             | 覆盖核心功能和边界情况的单元测试                                    |

## 在 Docker 中运行

使用独立的、已容器化的 Jupyter notebook 服务器是尝试 NautilusTrader 最快速的方式，无需
本地环境搭建。删除该容器会同时删除其中的所有数据。

```bash
# 拉取最新镜像
docker pull ghcr.io/nautechsystems/jupyterlab:nightly --platform linux/amd64

# 运行容器
docker run -p 8888:8888 ghcr.io/nautechsystems/jupyterlab:nightly
```

然后在浏览器中打开 <http://localhost:8888>。

:::warning
示例中使用了 `log_level="ERROR"`，因为 Nautilus 的日志输出超过了 Jupyter 的
stdout 速率限制，在较低日志级别下会导致 notebook 挂起。
:::
</content>
