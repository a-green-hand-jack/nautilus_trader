# 回测 API 与重复运行

## 选择 API 级别

在以下情况下，建议使用**低层级** API：

- 你的整个数据流可以在可用机器资源（例如内存）范围内完整处理。
- 你不希望将数据以 Nautilus 专用的 Parquet 格式存储。
- 你有特定需求或偏好，希望以原始格式（例如 CSV、二进制等）保留原始数据。
- 你需要对 `BacktestEngine` 进行细粒度控制，例如在相同数据集上重复运行回测，同时替换组件
  （例如 actors 或策略）或调整参数配置。

在以下情况下，建议使用**高层级** API：

- 你的数据流超出可用内存，需要以批次形式流式加载数据。
- 你希望获得 `ParquetDataCatalog` 在存储 Nautilus 专用 Parquet 格式数据时的性能与便利性。
- 你重视通过传递配置对象来定义和管理跨多个引擎的多次回测运行所带来的灵活性和功能性。

## 低层级 API

低层级 API 以 `BacktestEngine` 为核心，输入通过 Python 脚本手动初始化并添加。实例化的
`BacktestEngine` 可以接受以下内容：

- `Data` 对象列表，会根据 `ts_init` 自动排序为单调递增顺序。
- 多个交易场所（venue），需手动初始化。
- 多个 actors，需手动初始化并添加。
- 多个执行算法，需手动初始化并添加。

这种方式为回测过程提供了细致的控制，允许你手动配置每个组件。

### 高效加载大规模数据集

在跨多个金融工具处理大量数据时，加载数据的方式会显著影响性能。

#### 性能考量

默认情况下，当 `sort=True`（默认值）时，`BacktestEngine.add_data()` 会在每次调用时对整个数据流
（现有数据 + 新添加的数据）进行排序。这意味着：

- 第一次调用 100 万条 K 线：对 100 万条排序。
- 第二次调用 100 万条 K 线：对 200 万条排序。
- 第三次调用 100 万条 K 线：对 300 万条排序。
- 依此类推……

在为多个金融工具加载数据时，这种对不断增大的数据集重复排序的方式可能成为性能瓶颈。

#### 优化策略

**策略一：将排序推迟到最后（推荐用于多个金融工具的情况）**

```python
from nautilus_trader.backtest.engine import BacktestEngine

engine = BacktestEngine()

# 设置交易场所与金融工具
engine.add_venue(...)
engine.add_instrument(instrument1)
engine.add_instrument(instrument2)
engine.add_instrument(instrument3)

# 加载所有数据时不进行排序
engine.add_data(instrument1_bars, sort=False)
engine.add_data(instrument2_bars, sort=False)
engine.add_data(instrument3_bars, sort=False)

# 最后统一排序一次——效率更高！
engine.sort_data()

# 现在运行回测
engine.add_strategy(strategy)
engine.run()
```

**策略二：先收集数据，再一次性批量添加**

```python
# 先收集所有数据
all_bars = []
all_bars.extend(instrument1_bars)
all_bars.extend(instrument2_bars)
all_bars.extend(instrument3_bars)

# 一次性添加并排序
engine.add_data(all_bars, sort=True)
```

**策略三：对超大数据集使用流式 API**

对于无法完整放入内存的数据集，有两种流式处理方式：

**自动分块（chunking）**——提供一个按批次产生数据的生成器（generator）。引擎会在单次 `run()`
调用期间惰性地拉取数据块：

```python
def data_generator():
    # 依次产出数据块（每个数据块是一个 Data 对象的列表）
    yield load_chunk_1()
    yield load_chunk_2()
    yield load_chunk_3()

engine.add_data_iterator(
    data_name="my_data_stream",
    generator=data_generator(),
)

engine.run()  # 数据块按需消费
```

**手动分块**——由你自己加载并逐批运行。这也是 `BacktestNode` 内部使用的模式，能让你完全掌控批次
边界：

```python
engine.add_strategy(strategy)

for batch in data_batches:
    engine.add_data(batch)
    engine.run(streaming=True)
    engine.clear_data()

engine.end()  # 收尾：刷新剩余计时器，停止引擎，生成结果
```

:::note
在流式模式下，每个批次的数据耗尽后，计时器的推进会停止。安排在最后一个数据点之后的计时器（例如 K
线聚合的时间间隔）会被推迟，直到有更多数据到达或调用 `end()`，后者会将上一次 `run()` 调用的
`end` 边界之前的内容全部刷新完成。
:::

:::tip[性能影响]
对于包含 10 个金融工具、每个 100 万根 K 线的回测：

- 每次调用都排序：约进行 10 次规模递增的排序（100 万、200 万、300 万……直到 1000 万条）。
- 最后统一排序一次：仅进行 1 次针对 1000 万条数据的排序。

对于大规模数据集，延迟排序的方式可以**显著提升速度**。
:::

### 数据加载约定

`BacktestEngine` 强制执行重要的不变量（invariants）以确保数据完整性：

**要求：**

- 在调用 `run()` 之前，所有数据都必须已经排序。
- 使用 `sort=False` 时，你**必须**在运行前调用 `sort_data()`。
- 引擎会对此进行校验，若检测到未排序数据会抛出 `RuntimeError`。
- 多次调用 `sort_data()` 是安全的（幂等操作）。

**安全保证：**

- 数据列表在内部始终会被复制，以防止外部修改影响引擎状态。
- 将数据列表传递给 `add_data()` 之后，你可以安全地清空或修改这些列表。
- 使用 `sort=True` 添加数据后，数据会立即可用于回测。

这种设计既确保了数据完整性，又能够为大规模数据集实现性能优化。

## 高层级 API

高层级 API 以 `BacktestNode` 为核心，负责编排多个 `BacktestEngine` 实例的管理，每个实例由一个
`BacktestRunConfig` 定义。多个配置可以打包为一个列表，由该节点在一次运行中统一处理。

每个 `BacktestRunConfig` 对象由以下部分组成：

- `BacktestDataConfig` 对象列表。
- `BacktestVenueConfig` 对象列表。
- `ImportableActorConfig` 对象列表。
- `ImportableStrategyConfig` 对象列表。
- `ImportableExecAlgorithmConfig` 对象列表。
- 一个可选的 `ImportableControllerConfig` 对象。
- 一个可选的 `BacktestEngineConfig` 对象，若未指定则使用默认配置。

## 出错时关闭

设置 `BacktestEngineConfig.shutdown_on_error=True`，使得 Rust 层的错误日志能够终止回测运行。Rust
日志系统会记录内核启动后第一条发出的 `log::error!`，并在回测循环下一次检查关闭状态时，将该触发条件
转换为一条 `ShutdownSystem` 命令。

关闭请求遵循正常的回测停止路径：先停止交易器（trader）和引擎，然后返回截至关闭时刻已收集的回测结果。
它不会中止进程。有关最终 `on_stop` 及命令结算行为的详细信息，请参见
[关闭语义（shutdown semantics）](execution-flow.md#shutdown-semantics)。

```python
from nautilus_trader.backtest import BacktestEngineConfig

config = BacktestEngineConfig(shutdown_on_error=True)
```

被组件过滤器或 `bypass_logging=True` 抑制的错误日志仍然会触发关闭请求。当新的内核运行启动时，触发条件
会被清除并重新启用，因此一个进程可以在不重新初始化日志系统的情况下运行另一次回测。出错关闭机制关注的是
Rust 的 `log` 记录，而非 Python 的 `logging.error(...)` 调用。

## 重复运行

在进行多次回测运行时，理解组件如何重置以避免出现意外行为非常重要。

### 重置 BacktestEngine

`.reset()` 方法会将引擎状态和已加载组件的状态恢复到其**初始值**。它会保留已加载的组件、数据、金融工具
和交易场所的注册信息。

**会被重置的内容：**

- 所有交易状态（订单、持仓、账户余额）。
- 已加载的 actors、策略和执行算法会被原地重置。
- 引擎计数器和时间戳。

**会保留的内容：**

- 通过 `.add_data()` 添加的数据（使用 `.clear_data()` 移除）。
- 金融工具（必须与保留的数据相匹配）。
- 交易场所配置。
- 已加载的 actors、策略和执行算法。

**金融工具处理：**

对于 `BacktestEngine`，默认情况下金融工具会在重置后保留（因为数据会保留，而金融工具必须与数据匹配）。
这一行为通过默认 `BacktestEngineConfig` 中的 `CacheConfig.drop_instruments_on_reset=False`
进行配置。

### 多次回测运行的方式

运行多次回测主要有两种方式：

#### 使用 BacktestNode 用于生产环境

高层级 API 专为使用不同配置进行多次回测运行而设计：

```python
from nautilus_trader.backtest.node import BacktestNode
from nautilus_trader.config import BacktestRunConfig

# 定义多个运行配置
configs = [
    BacktestRunConfig(...),  # 运行 1
    BacktestRunConfig(...),  # 运行 2
    BacktestRunConfig(...),  # 运行 3
]

# 执行所有运行
node = BacktestNode(configs=configs)
results = node.run()
```

每次运行都会获得一个具有全新状态的引擎——无需调用 reset()。

#### 使用 BacktestEngine.reset

若要通过低层级 API 实现细粒度控制：

```python
from nautilus_trader.backtest.engine import BacktestEngine

engine = BacktestEngine()

# 只需设置一次
engine.add_venue(...)
engine.add_instrument(ETHUSDT)
engine.add_data(data)

# 运行 1
engine.add_strategy(strategy1)
engine.run()

# 重置后使用相同已加载的策略运行第 2 次
engine.reset()
engine.run()

# 重置后使用不同的策略运行第 3 次
engine.reset()
engine.clear_strategies()
engine.add_strategy(strategy2)
engine.run()
```

:::note
对于 `BacktestEngine`，金融工具和数据默认会在重置后保留，这使得参数优化变得十分简便。
:::

:::tip[最佳实践]

- **用于生产回测：** 使用带配置对象的 `BacktestNode`。
- **用于参数优化：** 使用 `BacktestEngine.reset()` 保留数据和金融工具，然后在添加替代策略实例之前
  调用 `clear_strategies()`。
- **用于快速实验：** 两种方式都可行——根据具体使用场景选择。

:::
