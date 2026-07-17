# 数据测试规范

本节使用 `DataTester` actor 定义了一套严谨的测试矩阵，用于验证适配器的数据功能。
Python（`nautilus_trader.test_kit.strategies.tester_data`）和
Rust（`nautilus_testkit::testers`）都提供了 `DataTester`。每个测试用例通过带前缀的
ID（例如 TC-D01）标识，并按功能分组。

**每个适配器都必须通过与其所支持的数据类型相匹配的那部分测试子集。**

各测试组按照从最基础到最派生的数据顺序排列：先是金融工具和原始订单簿数据，
然后是报价、成交、K 线（bar）和衍生品数据。通过第 1-4 组测试的适配器被认为达到了
基线数据合规标准。

适配器特有的数据行为（自定义频道、限流、快照语义等）应记录在该适配器自己的指南中，
而不是本文档中。

## 前置条件

在运行数据测试之前：

- 目标金融工具可用，且能够通过金融工具提供者加载。
- 当交易场所要求对所测数据进行身份验证时，通过环境变量设置好 API 凭据
  （`{VENUE}_API_KEY`、`{VENUE}_API_SECRET`）。
- 如果该交易场所提供演示/测试网（demo/testnet）模式，请使用为该环境创建的凭据。
  演示环境和生产环境的 API key 通常是各自独立、不可互换的；使用错误的凭据会产生
  身份验证错误（例如 HTTP 401）。

**Python 节点设置**：

旧版示例仍使用 `nautilus_trader.live.node.TradingNode`，但新的、由 Rust 支撑的
PyO3 适配器应优先使用 `nautilus_trader.live.LiveNode`。当你需要在节点构建之前
注册适配器客户端工厂时，使用 `LiveNode.builder(...)`。

```python
from nautilus_trader.common import Environment
from nautilus_trader.live import LiveDataEngineConfig, LiveNode
from nautilus_trader.model import TraderId

node = (
    LiveNode.builder("TESTER-001", TraderId("TESTER-001"), Environment.SANDBOX)
    .with_data_engine_config(
        LiveDataEngineConfig(time_bars_build_with_no_updates=False)
    )
    .add_data_client(None, adapter_data_client_factory, data_client_config)
    .build()
)

node.add_actor_from_config(importable_actor_config)
# 注册其余组件，然后启动或运行
```

**Rust 节点设置**（参考：`crates/adapters/{adapter}/examples/node_data_tester.rs`）：

```rust
use nautilus_testkit::testers::{DataTester, DataTesterConfig};

let tester_config = DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_quotes(true)
    .build()?;
let tester = DataTester::new(tester_config);
node.add_actor(tester)?;
node.run().await?;
```

以下每一组都以一份汇总表开始，随后是详细的测试卡片。测试 ID 使用带间隔的编号，
以便插入新用例而无需重新编号。

---

## 第 1 组：金融工具

在测试市场数据流之前，验证金融工具的加载和订阅。

| TC      | 名称                        | 描述                                          | 何时跳过            |
|---------|-----------------------------|------------------------------------------------------|----------------------|
| TC-D01  | 请求金融工具         | 加载某交易场所的所有金融工具。                    | 从不跳过。               |
| TC-D02  | 订阅金融工具        | 订阅金融工具更新。                     | 无金融工具订阅时。   |
| TC-D03  | 加载特定金融工具    | 按 ID 加载单个金融工具。                          | 从不跳过。               |

### TC-D01：请求金融工具

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接。                                                     |
| **操作**         | DataTester 在启动时请求该交易场所的所有金融工具。            |
| **事件序列** | `on_instruments` 回调接收到金融工具列表。                    |
| **通过标准**  | 至少收到一个金融工具；每个都具有有效的代码、价格精度和数量增量。 |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    request_instruments=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .request_instruments(true)
    .build()?
```

### TC-D02：订阅金融工具

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 订阅金融工具更新。                           |
| **事件序列** | `on_instrument` 回调接收到金融工具。                          |
| **通过标准**  | 收到的金融工具具有正确的 `instrument_id` 及有效字段。        |
| **何时跳过**      | 适配器不支持金融工具订阅时。                     |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_instrument=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_instrument(true)
    .build()?
```

### TC-D03：加载特定金融工具

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接。                                                     |
| **操作**         | 通过金融工具提供者按 `InstrumentId` 加载某个特定的金融工具。 |
| **事件序列** | 加载后金融工具在缓存中可用。                              |
| **通过标准**  | 金融工具已加载，且 ID、价格精度、数量增量和交易规则均正确。 |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- 该测试直接测试金融工具提供者的 `load` / `load_async` 方法。
- 验证该金融工具已被缓存，并可通过 `self.cache.instrument(instrument_id)` 获取。

---

## 第 2 组：订单簿

测试订单簿订阅模式和快照请求。

| TC      | 名称                           | 描述                                        | 何时跳过              |
|---------|--------------------------------|----------------------------------------------------|------------------------|
| TC-D10  | 订阅订单簿深度变化          | 流式接收 `OrderBookDeltas` 更新。                  | 不支持订单簿时。       |
| TC-D11  | 按间隔订阅订单簿     | 周期性的 `OrderBook` 快照。                    | 不支持订单簿时。       |
| TC-D12  | 订阅订单簿深度           | `OrderBookDepth10` 快照。                      | 不支持订单簿深度时。         |
| TC-D13  | 请求订单簿快照          | 一次性的订单簿快照请求。                    | 不支持订单簿快照时。      |
| TC-D14  | 从深度变化数据构建的托管订单簿       | 根据深度变化数据流构建本地订单簿。                | 不支持订单簿时。       |
| TC-D15  | 请求历史订单簿深度变化数据 | 历史订单簿深度变化数据请求。                    | 不支持历史深度变化数据时。  |

### TC-D10：订阅订单簿深度变化

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 订阅订单簿深度变化数据。                            |
| **事件序列** | 在 `on_order_book_deltas` 中收到 `OrderBookDeltas` 事件。           |
| **通过标准**  | 收到的深度变化数据具有有效的金融工具 ID；至少一条深度变化数据包含买/卖更新。 |
| **何时跳过**      | 适配器不支持订单簿数据时。                              |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_book_deltas=True,
    book_type=BookType.L2_MBP,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_book_deltas(true)
    .book_type(BookType::L2_MBP)
    .build()?
```

### TC-D11：按间隔订阅订单簿

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 订阅周期性的订单簿快照。                |
| **事件序列** | 在配置的间隔时间内，`on_order_book` 中收到 `OrderBook` 事件。 |
| **通过标准**  | 收到含买/卖档位的订单簿快照；更新以大致配置的间隔到达。 |
| **何时跳过**      | 适配器不支持订单簿数据时。                              |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_book_at_interval=True,
    book_type=BookType.L2_MBP,
    book_depth=10,
    book_interval_ms=1000,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_book_at_interval(true)
    .book_type(BookType::L2_MBP)
    .book_depth(10)
    .book_interval_ms(1000)
    .build()?
```

### TC-D12：订阅订单簿深度

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 订阅 `OrderBookDepth10` 快照。                 |
| **事件序列** | 在 `on_order_book_depth` 中收到 `OrderBookDepth10` 事件。           |
| **通过标准**  | 收到的深度快照最多包含 10 档买/卖价位；价格顺序正确。 |
| **何时跳过**      | 适配器不支持订单簿深度订阅时。                     |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_book_depth=True,
    book_type=BookType.L2_MBP,
    book_depth=10,
)
```

**Rust 配置：** 尚未支持。订单簿深度订阅在 Rust 版 `DataTester` 中仍是待办事项。

### TC-D13：请求订单簿快照

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 请求一次性的订单簿快照。                    |
| **事件序列** | 通过历史数据回调收到订单簿快照。                   |
| **通过标准**  | 快照中包含具有有效价格和数量的买/卖档位。          |
| **何时跳过**      | 适配器不支持订单簿快照请求时。                       |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    request_book_snapshot=True,
    book_depth=10,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .request_book_snapshot(true)
    .book_depth(10)
    .build()?
```

### TC-D14：从深度变化数据构建的托管订单簿

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，订单簿深度变化数据流已开启。           |
| **操作**         | DataTester 以 `manage_book=True` 订阅深度变化数据；从该数据流构建本地订单簿。 |
| **事件序列** | `OrderBookDeltas` 被应用到本地 `OrderBook`；订单簿按配置的深度记录日志。 |
| **通过标准**  | 本地订单簿能根据深度变化数据正确构建；买价档位递减，卖价档位递增；初始快照后订单簿非空。 |
| **何时跳过**      | 适配器不支持订单簿数据时。                              |

**注意事项：**

- 托管订单簿会将每一条深度变化数据应用到由该 actor 维护的 `OrderBook` 实例上。
- 使用 `book_levels_to_print` 来控制日志的详细程度。

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_book_deltas=True,
    manage_book=True,
    book_type=BookType.L2_MBP,
    book_levels_to_print=10,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_book_deltas(true)
    .manage_book(true)
    .book_type(BookType::L2_MBP)
    .build()?
```

### TC-D15：请求历史订单簿深度变化数据

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 请求历史订单簿深度变化数据。                      |
| **事件序列** | 通过回调收到历史深度变化数据。                               |
| **通过标准**  | 收到的深度变化数据具有有效的时间戳和订单簿动作。                |
| **何时跳过**      | 适配器不支持历史订单簿深度变化数据请求时。               |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    request_book_deltas=True,
)
```

**Rust 配置：** 尚未支持。历史订单簿深度变化数据请求在 Rust 版 `DataTester` 中仍是待办事项。

---

## 第 3 组：报价

测试报价 tick 订阅和历史请求。

| TC      | 名称                      | 描述                                     | 何时跳过              |
|---------|---------------------------|-------------------------------------------------|------------------------|
| TC-D20  | 订阅报价          | 验证启动后 `QuoteTick` 事件是否流动。     | 从不跳过。                 |
| TC-D21  | 请求历史报价 | 请求历史报价 tick。                 | 不支持历史报价时。  |

### TC-D20：订阅报价

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 在启动时订阅报价。                              |
| **事件序列** | 在 `on_quote_tick` 中收到 `QuoteTick` 事件。                        |
| **通过标准**  | 至少收到一条 `QuoteTick`，具有有效的买/卖价格和数量；买价 < 卖价。 |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_quotes=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_quotes(true)
    .build()?
```

### TC-D21：请求历史报价

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 请求历史报价 tick。                            |
| **事件序列** | 通过 `on_historical_data` 回调收到历史报价。          |
| **通过标准**  | 收到的报价具有有效的时间戳、买/卖价格和数量。       |
| **何时跳过**      | 适配器不支持历史报价请求时。                    |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    request_quotes=True,
    requests_start_delta=pd.Timedelta(hours=1),
)
```

---

## 第 4 组：成交

测试成交 tick 订阅和历史请求。

| TC     | 名称                      | 描述                                     | 何时跳过              |
|--------|---------------------------|-------------------------------------------------|------------------------|
| TC-D30 | 订阅成交          | 验证启动后 `TradeTick` 事件是否流动。     | 从不跳过。                 |
| TC-D31 | 请求历史成交 | 请求历史成交 tick。                 | 不支持历史成交时。  |

### TC-D30：订阅成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 在启动时订阅成交。                              |
| **事件序列** | 在 `on_trade_tick` 中收到 `TradeTick` 事件。                        |
| **通过标准**  | 至少收到一条 `TradeTick`，具有有效的价格、数量和主动方（aggressor side）。 |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_trades=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_trades(true)
    .build()?
```

### TC-D31：请求历史成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 请求历史成交 tick。                            |
| **事件序列** | 通过 `on_historical_data` 回调收到历史成交。          |
| **通过标准**  | 收到的成交具有有效的时间戳、价格、数量和成交 ID。   |
| **何时跳过**      | 适配器不支持历史成交请求时。                    |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    request_trades=True,
    requests_start_delta=pd.Timedelta(hours=1),
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .request_trades(true)
    .build()?
```

---

## 第 5 组：K 线（bar）

测试 K 线（bar）订阅和历史请求。

| TC      | 名称                    | 描述                                       | 何时跳过           |
|---------|-------------------------|-----------------------------------------------------|---------------------|
| TC-D40  | 订阅 K 线          | 验证启动后 `Bar` 事件是否流动。             | 不支持 K 线时。     |
| TC-D41  | 请求历史 K 线 | 请求历史 OHLCV K 线数据。                    | 不支持历史 K 线时。 |

### TC-D40：订阅 K 线

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，K 线类型已配置。             |
| **操作**         | DataTester 为配置的 `BarType` 订阅 K 线。              |
| **事件序列** | 在 `on_bar` 中收到 `Bar` 事件。                                     |
| **通过标准**  | 至少收到一条 `Bar`，OHLCV 数值有效；最高价 >= 最低价，最高价 >= 开盘价，最高价 >= 收盘价。 |
| **何时跳过**      | 适配器不支持 K 线订阅时。                            |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    bar_types=[BarType.from_str("BTCUSDT-PERP.VENUE-1-MINUTE-LAST-EXTERNAL")],
    subscribe_bars=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .bar_types(vec![bar_type])
    .subscribe_bars(true)
    .build()?
```

### TC-D41：请求历史 K 线

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，K 线类型已配置。             |
| **操作**         | DataTester 为配置的 `BarType` 请求历史 K 线。        |
| **事件序列** | 通过回调收到历史 K 线。                                 |
| **通过标准**  | 收到的 K 线具有有效的 OHLCV 数值，且时间戳递增。        |
| **何时跳过**      | 适配器不支持历史 K 线请求时。                      |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    bar_types=[BarType.from_str("BTCUSDT-PERP.VENUE-1-MINUTE-LAST-EXTERNAL")],
    request_bars=True,
    requests_start_delta=pd.Timedelta(hours=1),
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .bar_types(vec![bar_type])
    .request_bars(true)
    .build()?
```

---

## 第 6 组：衍生品数据

测试衍生品特有的数据流：标记价格、指数价格和资金费率。

| TC     | 名称                             | 描述                                 | 何时跳过             |
|--------|----------------------------------|-----------------------------------------------|-----------------------|
| TC-D50 | 订阅标记价格            | `MarkPriceUpdate` 事件。                   | 非衍生品时。     |
| TC-D51 | 订阅指数价格           | `IndexPriceUpdate` 事件。                  | 非衍生品时。     |
| TC-D52 | 订阅资金费率          | `FundingRateUpdate` 事件。                 | 非永续合约时。      |
| TC-D53 | 请求历史资金费率 | 历史资金费率数据。               | 非永续合约时。      |

### TC-D50：订阅标记价格

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，衍生品金融工具已加载。                       |
| **操作**         | DataTester 订阅标记价格更新。                           |
| **事件序列** | 在 `on_mark_price` 中收到 `MarkPriceUpdate` 事件。                  |
| **通过标准**  | 至少收到一条 `MarkPriceUpdate`，具有有效的金融工具 ID 和标记价格。 |
| **何时跳过**      | 金融工具不是衍生品，或适配器不提供标记价格时。 |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_mark_prices=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_mark_prices(true)
    .build()?
```

### TC-D51：订阅指数价格

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，衍生品金融工具已加载。                       |
| **操作**         | DataTester 订阅指数价格更新。                          |
| **事件序列** | 在 `on_index_price` 中收到 `IndexPriceUpdate` 事件。                |
| **通过标准**  | 至少收到一条 `IndexPriceUpdate`，具有有效的金融工具 ID 和指数价格。 |
| **何时跳过**      | 金融工具不是衍生品，或适配器不提供指数价格时。 |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_index_prices=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_index_prices(true)
    .build()?
```

### TC-D52：订阅资金费率

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，永续合约金融工具已加载。                        |
| **操作**         | DataTester 订阅资金费率更新。                         |
| **事件序列** | 在 `on_funding_rate` 中收到 `FundingRateUpdate` 事件。              |
| **通过标准**  | 至少收到一条 `FundingRateUpdate`，具有有效的金融工具 ID 和费率。 |
| **何时跳过**      | 金融工具不是永续合约，或适配器不提供资金费率时。 |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_funding_rates=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_funding_rates(true)
    .build()?
```

### TC-D53：请求历史资金费率

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，永续合约金融工具已加载。                        |
| **操作**         | DataTester 请求历史资金费率（默认回溯 7 天）。 |
| **事件序列** | 通过回调收到历史资金费率。                        |
| **通过标准**  | 收到的资金费率具有有效的时间戳和费率值。          |
| **何时跳过**      | 金融工具不是永续合约，或适配器不支持历史资金费率请求时。 |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    request_funding_rates=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .request_funding_rates(true)
    .build()?
```

---

## 第 7 组：金融工具状态

测试金融工具状态和收盘事件订阅。

| TC     | 名称                        | 描述                                    | 何时跳过             |
|--------|-----------------------------|--------------------------------------------------|-----------------------|
| TC-D60 | 订阅金融工具状态 | `InstrumentStatus` 事件。                     | 不支持状态时。    |
| TC-D61 | 订阅金融工具收盘  | `InstrumentClose` 事件。                      | 不支持收盘时。     |

### TC-D60：订阅金融工具状态

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 订阅金融工具状态更新。                    |
| **事件序列** | 在 `on_instrument_status` 中收到 `InstrumentStatus` 事件。          |
| **通过标准**  | 收到的状态事件具有有效的 `MarketStatusAction`（例如 `Trading`）。 |
| **何时跳过**      | 适配器不支持金融工具状态订阅时。              |

**注意事项：**

- 状态事件可能只在状态发生变化时触发（例如从交易暂停 -> 恢复交易）。
- 在正常交易时段内，订阅时可能会收到一次 `Trading` 状态。

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_instrument_status=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_instrument_status(true)
    .build()?
```

### TC-D61：订阅金融工具收盘

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载。                                  |
| **操作**         | DataTester 订阅金融工具收盘事件。                      |
| **事件序列** | 在 `on_instrument_close` 中收到 `InstrumentClose` 事件。            |
| **通过标准**  | 收到具有有效收盘价格和收盘类型的收盘事件。            |
| **何时跳过**      | 适配器不支持金融工具收盘订阅时。               |

**注意事项：**

- 对于传统市场，收盘事件通常在收盘时触发。
- 对于 24/7 交易的加密货币交易场所，除非适配器自行合成一个每日收盘，
  否则该事件可能不会触发。

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_instrument_close=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_instrument_close(true)
    .build()?
```

---

## 第 8 组：期权希腊值

测试期权希腊值和期权链订阅。

| TC     | 名称                        | 描述                                    | 何时跳过              |
|--------|-----------------------------|--------------------------------------------------|------------------------|
| TC-D62 | 订阅期权希腊值     | 单个金融工具的 `OptionGreeks` 数据。   | 不支持希腊值时。     |
| TC-D63 | 订阅期权链      | 某系列的 `OptionChainSlice` 快照。     | 不支持期权链时。      |

### TC-D62：订阅期权希腊值

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，期权金融工具已加载。                           |
| **操作**         | DataTester 订阅期权希腊值更新。                        |
| **事件序列** | 在 `on_option_greeks` 中收到 `OptionGreeks` 事件。                  |
| **通过标准**  | 收到具有有效 delta、gamma、vega、theta 值的希腊值。           |
| **何时跳过**      | 适配器不支持期权希腊值订阅时。                  |

**注意事项：**

- 希腊值仅对期权类金融工具可用。
- 值取决于该交易场所的定价模型，可能在每次报价变化时都会更新。
- 一些交易场所（Bybit、Deribit）按金融工具订阅；OKX 按金融工具族订阅，
  并过滤出所请求的金融工具。
- 当交易场所不提供 `rho` 时（Bybit、OKX），其值可能为零。
- `underlying_price` 和 `open_interest` 可能因交易场所频道不同而为 `None`。

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_option_greeks=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_option_greeks(true)
    .build()?
```

### TC-D63：订阅期权链

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，该系列的期权金融工具已加载。           |
| **操作**         | DataTester 订阅某系列的期权链快照。          |
| **事件序列** | 在 `on_option_chain` 中收到 `OptionChainSlice` 快照。            |
| **通过标准**  | 期权链快照包含与该系列匹配的金融工具的希腊值。     |
| **何时跳过**      | 适配器不支持期权链订阅时。                   |

**注意事项：**

- 期权链订阅由 DataEngine 管理，DataEngine 会在内部为每个金融工具创建报价和
  希腊值订阅。
- 相对于平值（ATM）的行权价范围需要在订阅开始前进行远期价格自举（bootstrap）。
- 尚不能通过 `DataTesterConfig` 进行配置；需要手动设置 actor，
  使用 `subscribe_option_chain` 和 `OptionSeriesId`。

---

## 第 9 组：生命周期

测试 actor 生命周期行为：取消订阅处理和自定义参数。

| TC     | 名称                    | 描述                                        | 何时跳过            |
|--------|-------------------------|------------------------------------------------------|----------------------|
| TC-D70 | 停止时取消订阅     | 在 actor 停止时取消订阅数据流。         | 不支持取消订阅时。    |
| TC-D71 | 自定义订阅参数 | 适配器特有的订阅参数。          | 不适用（N/A）。 |
| TC-D72 | 自定义请求参数   | 适配器特有的请求参数。               | 不适用（N/A）。 |

### TC-D70：停止时取消订阅

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 已有活跃的数据订阅（报价、成交、订单簿）。                      |
| **操作**         | 使用 `can_unsubscribe=True`（默认值）停止该 actor。                  |
| **事件序列** | 数据订阅被移除；不再收到数据事件。           |
| **通过标准**  | 顺利完成取消订阅；日志中没有错误；停止之后没有数据事件。       |
| **何时跳过**      | 适配器不支持取消订阅时。                                  |

**Python 配置：**

```python
DataTesterConfig(
    instrument_ids=[instrument_id],
    subscribe_quotes=True,
    subscribe_trades=True,
    can_unsubscribe=True,
)
```

**Rust 配置：**

```rust
DataTesterConfig::builder()
    .client_id(client_id)
    .instrument_ids(vec![instrument_id])
    .subscribe_quotes(true)
    .subscribe_trades(true)
    .can_unsubscribe(true)
    .build()?
```

### TC-D71：自定义订阅参数

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，且该适配器接受额外的订阅参数。 |
| **操作**         | 使用包含适配器特有参数的 `subscribe_params` 字典进行订阅。 |
| **事件序列** | 订阅建立，自定义参数生效。               |
| **通过标准**  | 数据在适配器特有参数生效的情况下正常流动。                 |
| **何时跳过**      | 不适用（取决于具体适配器）。                                                |

**注意事项：**

- `subscribe_params` 字典对 DataTester 是不透明的，会原样传递给适配器。
- 请查阅具体适配器的指南以了解支持的参数。

### TC-D72：自定义请求参数

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，且该适配器接受额外的请求参数。      |
| **操作**         | 使用包含适配器特有参数的 `request_params` 字典请求数据。 |
| **事件序列** | 请求在自定义参数生效的情况下得到满足。                      |
| **通过标准**  | 在适配器特有参数生效的情况下收到历史数据。   |
| **何时跳过**      | 不适用（取决于具体适配器）。                                                |

**注意事项：**

- `request_params` 字典对 DataTester 是不透明的，会原样传递给适配器。
- 请查阅具体适配器的指南以了解支持的参数。

---

## DataTester 配置参考

所有 `DataTesterConfig` 参数的快速参考。所展示的默认值针对的是 Python 配置。
注意：Rust 版 `DataTesterConfig` 构建器将 `manage_book` 默认设置为 `true`，
而 Python 版默认设置为 `False`。

| 参数                    | 类型              | 默认值         | 影响的分组 |
|------------------------------|-------------------|-----------------|----------------|
| `instrument_ids`             | list[InstrumentId]| *必填*      | 全部            |
| `client_id`                  | ClientId?         | None            | 全部            |
| `bar_types`                  | list[BarType]?    | None            | 5              |
| `subscribe_book_deltas`      | bool              | False           | 2              |
| `subscribe_book_depth`       | bool              | False           | 2              |
| `subscribe_book_at_interval` | bool              | False           | 2              |
| `subscribe_quotes`           | bool              | False           | 3              |
| `subscribe_trades`           | bool              | False           | 4              |
| `subscribe_mark_prices`      | bool              | False           | 6              |
| `subscribe_index_prices`     | bool              | False           | 6              |
| `subscribe_funding_rates`    | bool              | False           | 6              |
| `subscribe_bars`             | bool              | False           | 5              |
| `subscribe_instrument`       | bool              | False           | 1              |
| `subscribe_instrument_status`| bool              | False           | 7              |
| `subscribe_instrument_close` | bool              | False           | 7              |
| `subscribe_option_greeks`    | bool              | False           | 8              |
| `subscribe_params`           | dict?             | None            | 9              |
| `can_unsubscribe`            | bool              | True            | 9              |
| `request_instruments`        | bool              | False           | 1              |
| `request_book_snapshot`      | bool              | False           | 2              |
| `request_book_deltas`        | bool              | False           | 2              |
| `request_quotes`             | bool              | False           | 3              |
| `request_trades`             | bool              | False           | 4              |
| `request_bars`               | bool              | False           | 5              |
| `request_funding_rates`      | bool              | False           | 6              |
| `request_params`             | dict?             | None            | 9              |
| `requests_start_delta`       | Timedelta?        | 1 小时          | 3、4、5        |
| `book_type`                  | BookType          | L2_MBP          | 2              |
| `book_depth`                 | PositiveInt?      | None            | 2              |
| `book_interval_ms`           | PositiveInt       | 1000            | 2              |
| `book_levels_to_print`       | PositiveInt       | 10              | 2              |
| `manage_book`                | bool              | False           | 2              |
| `use_pyo3_book`              | bool              | False           | 2              |
| `log_data`                   | bool              | True            | 全部            |

---
