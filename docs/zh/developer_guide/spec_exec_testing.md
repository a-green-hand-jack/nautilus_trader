# 执行测试规范

本节使用 `ExecTester` 策略定义了一套严谨的测试矩阵，用于验证适配器的执行功能。
Python（`nautilus_trader.test_kit.strategies.tester_exec`）和
Rust（`nautilus_testkit::testers`）都提供了 `ExecTester`。每个测试用例通过带前缀的
ID（例如 TC-E01）标识，并按功能分组。

**每个适配器都必须通过与其所支持能力相匹配的那部分测试子集。**

测试从简单（单个市价单）逐步过渡到复杂（组合单、修改链、拒绝处理）。通过第 1-5 组测试的
适配器被认为达到了基线合规标准。应先使用 [数据测试规范](spec_data_testing.md) 验证数据连通性。

适配器特有的行为（某个交易场所如何模拟市价单、如何处理有效期选项等）应记录在该适配器
自己的指南中，而不是本文档中。每个适配器指南都应包含一份能力矩阵，展示它支持哪些订单类型、
有效期选项、动作和标志。

## 前置条件

在运行执行测试之前：

- 拥有有效 API 凭据的演示/测试网账户（推荐，非必需）。
- 账户为测试的金融工具和数量提供了充足的保证金。
- 目标金融工具可用，且能够通过金融工具提供者加载。
- 已设置环境变量：`{VENUE}_API_KEY`、`{VENUE}_API_SECRET`（或沙盒变体）。
- 如果该交易场所提供演示/测试网模式，请使用为该环境创建的凭据。演示环境和生产环境的
  API key 通常是各自独立、不可互换的；使用错误的凭据会产生身份验证错误
  （例如 HTTP 401）。
- 绕过风险引擎（`LiveRiskEngineConfig(bypass=True)`）以避免干扰。
- 启用对账以验证状态一致性。

**Python 节点设置**：

旧版示例仍使用 `nautilus_trader.live.node.TradingNode`，但新的、由 Rust 支撑的
PyO3 适配器应优先使用 `nautilus_trader.live.LiveNode`。当你需要在节点构建之前
注册适配器客户端工厂时，使用 `LiveNode.builder(...)`。

```python
from nautilus_trader.common import Environment
from nautilus_trader.live import LiveExecEngineConfig, LiveNode, LiveRiskEngineConfig
from nautilus_trader.model import TraderId

node = (
    LiveNode.builder("TESTER-001", TraderId("TESTER-001"), Environment.SANDBOX)
    .with_risk_engine_config(LiveRiskEngineConfig(bypass=True))
    .with_exec_engine_config(LiveExecEngineConfig(reconciliation=True))
    .add_exec_client(None, adapter_exec_client_factory, exec_client_config)
    .build()
)

node.add_strategy_from_config(importable_strategy_config)
# 注册其余组件，然后启动或运行
```

**Rust 节点设置**（参考：`crates/adapters/{adapter}/examples/node_exec_tester.rs`）：

```rust
use nautilus_testkit::testers::{ExecTester, ExecTesterConfig};
use nautilus_trading::strategy::StrategyConfig;

let tester_config = ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(order_qty)
    .build()?;
let tester = ExecTester::new(tester_config);
node.add_strategy(tester)?;
node.run().await?;
```

## 基础冒烟测试

一个可以随时运行的快速健全性检查，例如在适配器变更之后或开发迭代之间运行。该测试工具
在启动时用一个市价单开仓，下一个买单和一个卖单的 post-only 限价单，等待 30 秒，然后停止
（取消未结订单并平仓）。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.001"),
    open_position_on_start_qty=Decimal("0.001"),
    enable_limit_buys=True,
    enable_limit_sells=True,
    use_post_only=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.001"))
    .open_position_on_start_qty(dec!(0.001))
    .enable_limit_buys(true)
    .enable_limit_sells(true)
    .use_post_only(true)
    .build()?
```

**预期行为：**

1. 启动时：市价单成交，开仓一个持仓。
2. 在距离最优买/卖价 `tob_offset_ticks`（默认 500 个 tick）的位置下两个限价单。
3. 策略空闲 30 秒。检查日志中是否有错误、被拒绝的订单或断开连接。
4. 停止时：取消未结的限价单，用市价单平仓。

**通过标准：** 日志中没有错误，持仓干净地开仓和平仓，限价单被交易场所确认。

---

以下每一组都以一份汇总表开始，随后是详细的测试卡片。测试 ID 使用带间隔的编号，
以便插入新用例而无需重新编号。

---

## 第 1 组：市价单

测试市价单提交和成交。市价单应立即执行。

| TC     | 名称                          | 描述                                         | 何时跳过           |
|--------|-------------------------------|-----------------------------------------------------|---------------------|
| TC-E01 | 市价买单 - 提交并成交  | 通过市价买单开多仓。                  | 不支持市价单时。   |
| TC-E02 | 市价卖单 - 提交并成交 | 通过市价卖单开空仓。                | 不支持市价单时。   |
| TC-E03 | 带 IOC 有效期的市价单     | 显式使用 IOC 有效期的市价单。    | 不支持 IOC 时。             |
| TC-E04 | 带 FOK 有效期的市价单     | 显式使用 FOK 有效期的市价单。    | 不支持 FOK 时。             |
| TC-E05 | 带报价货币数量的市价单   | 使用计价货币数量的市价单。         | 不支持报价数量时。  |
| TC-E06 | 通过市价单平仓     | 在停止时用市价单平掉未结持仓。 | 不支持市价单时。   |

### TC-E01：市价买单 - 提交并成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，市场数据在流动，没有未结持仓。 |
| **操作**         | ExecTester 通过 `open_position_on_start_qty` 开多仓。     |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 持仓以 side=LONG 开立，数量与配置匹配，成交价格在市场区间内，`AccountState` 已更新。 |
| **何时跳过**      | 适配器不支持市价单时。                                |

**注意事项：**

- 一些适配器将市价单模拟为激进的限价 IOC 订单（请查阅适配器指南）。
- 从策略的角度看，无论交易场所的实现机制如何，事件序列应保持一致。
- 成交价格应处于近期买卖价差范围内。
- 部分成交是有效的；请验证累计成交数量与订单数量相符。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(1, 2))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .build()?
```

### TC-E02：市价卖单 - 提交并成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，市场数据在流动，没有未结持仓。 |
| **操作**         | ExecTester 通过负值的 `open_position_on_start_qty` 开空仓。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 持仓以 side=SHORT 开立，数量与配置匹配，成交价格在市场区间内。 |
| **何时跳过**      | 适配器不支持市价单或做空时。               |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("-0.01"),
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(-1, 2))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .build()?
```

### TC-E03：带 IOC 有效期的市价单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，市场数据在流动。             |
| **操作**         | 以 `open_position_time_in_force=IOC` 开仓。                  |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 与 TC-E01 相同；订单上显式设置了 IOC 有效期。            |
| **何时跳过**      | 不支持 IOC 时。                                                        |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("0.01"),
    open_position_time_in_force=TimeInForce.IOC,
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(1, 2))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .open_position_time_in_force(TimeInForce::Ioc)
    .build()?
```

### TC-E04：带 FOK 有效期的市价单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，市场数据在流动。             |
| **操作**         | 以 `open_position_time_in_force=FOK` 开仓。                  |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 与 TC-E01 相同；订单上显式设置了 FOK 有效期。            |
| **何时跳过**      | 不支持 FOK 时。                                                        |

**注意事项：**

- FOK 要求整个数量能够立即成交，否则该订单会被取消。
- 使用较小的测试数量，使订单簿深度足以完全成交。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("0.01"),
    open_position_time_in_force=TimeInForce.FOK,
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(1, 2))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .open_position_time_in_force(TimeInForce::Fok)
    .build()?
```

### TC-E05：带报价货币数量的市价单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，适配器支持报价货币数量。 |
| **操作**         | 以 `use_quote_quantity=True` 开仓，数量以计价货币表示。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 订单以计价货币数量提交；成交数量以基础货币表示。 |
| **何时跳过**      | 适配器不支持报价数量订单时。                        |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("100.0"),  # 计价货币金额
    open_position_on_start_qty=Decimal("100.0"),
    use_quote_quantity=True,
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("100"))
    .open_position_on_start_qty(Decimal::from(100))
    .use_quote_quantity(true)
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .build()?
```

### TC-E06：停止时通过市价单平仓

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自 TC-E01 或 TC-E02 的未结持仓。                                   |
| **操作**         | 停止策略；ExecTester 通过市价单平仓。        |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`（平仓单）。 |
| **通过标准**  | 持仓已平（净数量 = 0），没有剩余的未结订单。          |
| **何时跳过**      | 适配器不支持市价单时。                                |

**注意事项：**

- 该测试自然地承接 TC-E01 或 TC-E02，属于同一次会话的一部分。
- `close_positions_on_stop=True` 是默认值。
- 平仓单应处于该持仓的相反方向。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("0.01"),
    close_positions_on_stop=True,
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(1, 2))
    .close_positions_on_stop(true)
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .build()?
```

---

## 第 2 组：限价单

测试限价单提交、确认，以及在各种有效期选项下的行为。

| TC     | 名称                       | 描述                                      | 何时跳过          |
|--------|----------------------------|--------------------------------------------------|--------------------|
| TC-E10 | 限价买单 GTC              | 在最优买价以下下 GTC 限价买单，验证被接受。  | 从不跳过。             |
| TC-E11 | 限价卖单 GTC             | 在最优卖价以上下 GTC 限价卖单，验证被接受。 | 从不跳过。             |
| TC-E12 | 限价买卖单对    | 同时下两侧订单，验证都被接受。 | 从不跳过。             |
| TC-E13 | 限价 IOC 激进成交  | 以激进价格下限价 IOC，预期成交。      | 不支持 IOC 时。            |
| TC-E14 | 限价 IOC 被动不成交  | 远离市场价下限价 IOC，预期取消。       | 不支持 IOC 时。            |
| TC-E15 | 限价 FOK 成交             | 以激进价格下限价 FOK，预期成交。      | 不支持 FOK 时。            |
| TC-E16 | 限价 FOK 不成交          | 远离市场价下限价 FOK，预期取消。       | 不支持 FOK 时。            |
| TC-E17 | 限价 GTD                  | 带到期时间的限价单，验证被接受。         | 不支持 GTD 时。            |
| TC-E18 | 限价 GTD 到期           | 等待 GTD 到期，验证 `OrderExpired`。      | 不支持 GTD 时。            |
| TC-E19 | 限价 DAY                  | 带 DAY 有效期的限价单，验证被接受。             | 不支持 DAY 时。            |

### TC-E10：限价买单 GTC - 提交并接受

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 在 `best_bid - tob_offset_ticks` 处下限价买单。        |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 订单以正确的价格、数量、side=BUY、TIF=GTC 在交易场所处于未结状态。 |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- `tob_offset_ticks`（默认 500）会使该订单远离市场，以避免意外成交。
- 验证该订单以 `OrderStatus.ACCEPTED` 出现在缓存中。
- 该订单应保持未结状态，直到被显式取消。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .build()?
```

### TC-E11：限价卖单 GTC - 提交并接受

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 在 `best_ask + tob_offset_ticks` 处下限价卖单。       |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 订单以正确的价格、数量、side=SELL、TIF=GTC 在交易场所处于未结状态。 |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(false)
    .enable_limit_sells(true)
    .build()?
```

### TC-E12：限价买卖单对

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 同时下一个限价买单和一个限价卖单。                     |
| **事件序列** | 两个独立的序列：每个都是 `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。 |
| **通过标准**  | 两个订单都在交易场所处于未结状态，买单在买价以下，卖单在卖价以上。              |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(true)
    .build()?
```

### TC-E13：限价 IOC 激进成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 以等于或高于最优卖价（激进价格）提交一个限价买 IOC 订单。    |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 订单立即成交；持仓已开立。                              |
| **何时跳过**      | 适配器不支持 IOC 有效期时。                                      |

**注意事项：**

- 该测试需要手动创建订单或使用适配器特有的配置，因为 ExecTester 默认下达限价单时使用
  GTC 有效期。
- 未立即成交的 IOC 订单会被交易场所取消。

### TC-E14：限价 IOC 被动 - 不成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 以远低于市场（被动价格）的价格提交一个限价买 IOC 订单。          |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderCanceled`。 |
| **通过标准**  | 订单立即被交易场所取消，没有成交。                   |
| **何时跳过**      | 适配器不支持 IOC 有效期时。                                      |

**注意事项：**

- 交易场所应取消未成交的 IOC 订单；验证 `OrderCanceled` 事件（而不是 `OrderExpired`）。

### TC-E15：限价 FOK 成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动，订单簿深度充足。 |
| **操作**         | 以激进价格、且数量在最优档位深度内提交一个限价买 FOK 订单。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 订单在单次成交事件中完全成交。                         |
| **何时跳过**      | 适配器不支持 FOK 有效期时。                                      |

**注意事项：**

- FOK 要求整个数量可成交；使用较小的数量以确保订单簿深度充足。

### TC-E16：限价 FOK 不成交

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 以被动价格（远低于市场价）提交一个限价买 FOK 订单。           |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderCanceled`。 |
| **通过标准**  | 订单立即被交易场所取消，没有成交。                   |
| **何时跳过**      | 适配器不支持 FOK 有效期时。                                      |

### TC-E17：限价 GTD - 提交并接受

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 下一个设置了 `order_expire_time_delta_mins` 的限价买单（例如 60 分钟）。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 订单以 GTD 有效期和正确的到期时间戳被接受。              |
| **何时跳过**      | 适配器不支持 GTD 有效期时。                                      |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    order_expire_time_delta_mins=60,
    enable_limit_buys=True,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .order_expire_time_delta_mins(60)
    .build()?
```

### TC-E18：限价 GTD 到期

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自 TC-E17 的未结 GTD 限价单（或使用一个很短的到期时间）。         |
| **操作**         | 等待 GTD 到期时间流逝。                                |
| **事件序列** | `OrderExpired`。                                                        |
| **通过标准**  | 订单转为已到期状态；收到 `OrderExpired` 事件。    |
| **何时跳过**      | 适配器不支持 GTD 有效期时。                                      |

**注意事项：**

- 使用较短的 `order_expire_time_delta_mins`（例如 1-2 分钟）以避免长时间等待。
- 一些交易场所可能会将到期报告为取消；验证适配器是否将其映射为 `OrderExpired`。

### TC-E19：限价 DAY - 提交并接受

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，市场处于交易时段内。      |
| **操作**         | 提交带 DAY 有效期的限价买单。                                         |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 订单以 DAY 有效期被接受；将在交易日结束时被自动取消。 |
| **何时跳过**      | 适配器不支持 DAY 有效期时。                                      |

**注意事项：**

- DAY 订单在 24/7 交易的加密货币交易场所与传统市场上的行为可能不同。
- 验证在交易时段之外提交时的行为（如果适用）。

---

## 第 3 组：止损单和条件单

测试止损单和条件单类型。这些订单会在交易场所挂单，直到触发条件被满足。
支持交易场所原生条件单的适配器还应验证，未结的触发单在重启对账中也能正确出现，
而不仅仅是在正常的未结订单端点中。

| TC     | 名称                   | 描述                                           | 何时跳过           |
|--------|------------------------|---------------------------------------------------------|---------------------|
| TC-E20 | StopMarket 买单         | 在卖价以上的止损买单，验证被接受。                  | 不支持 `STOP_MARKET` 时。   |
| TC-E21 | StopMarket 卖单        | 在买价以下的止损卖单，验证被接受。                 | 不支持 `STOP_MARKET` 时。   |
| TC-E22 | StopLimit 买单          | 带触发价 + 限价的止损限价买单。            | 不支持 `STOP_LIMIT` 时。    |
| TC-E23 | StopLimit 卖单         | 带触发价 + 限价的止损限价卖单。           | 不支持 `STOP_LIMIT` 时。    |
| TC-E24 | MarketIfTouched 买单    | 在买价以下的 MIT 买单。                                    | 不支持 `MIT` 时。           |
| TC-E25 | MarketIfTouched 卖单   | 在卖价以上的 MIT 卖单。                                   | 不支持 `MIT` 时。           |
| TC-E26 | LimitIfTouched 买单     | 带触发价 + 限价的 LIT 买单。                   | 不支持 `LIT` 时。           |
| TC-E27 | LimitIfTouched 卖单    | 带触发价 + 限价的 LIT 卖单。                  | 不支持 `LIT` 时。           |

### TC-E20：StopMarket 买单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 在当前卖价之上下一个止损市价买单。             |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 止损单以正确的触发价和 side=BUY 在交易场所被接受。  |
| **何时跳过**      | 适配器不支持 `StopMarket` 订单时。                          |

**注意事项：**

- 触发价应比当前卖价高 `stop_offset_ticks`。
- 该订单不应立即触发（触发价高于市场价）。
- 对于拥有长期存活的触发签名的交易场所，验证触发单签名的到期使用的是该交易场所的
  触发单窗口，而不是普通的订单到期时间。
- 验证触发和成交需要市场发生变动，这在测试期间可能不会发生。
- 接受之后，重启或强制对账，并验证当交易场所将触发单保存在独立端点时，
  该订单仍以未结订单报告的形式出现。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=False,
    enable_stop_buys=True,
    enable_stop_sells=False,
    stop_order_type=OrderType.STOP_MARKET,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .enable_stop_buys(true)
    .enable_stop_sells(false)
    .stop_order_type(OrderType::StopMarket)
    .build()?
```

### TC-E21：StopMarket 卖单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 在当前买价之下下一个止损市价卖单。            |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 止损单以正确的触发价和 side=SELL 在交易场所被接受。 |
| **何时跳过**      | 适配器不支持 `StopMarket` 订单时。                          |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=False,
    enable_stop_buys=False,
    enable_stop_sells=True,
    stop_order_type=OrderType.STOP_MARKET,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .enable_stop_buys(false)
    .enable_stop_sells(true)
    .stop_order_type(OrderType::StopMarket)
    .build()?
```

### TC-E22：StopLimit 买单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 下一个止损限价买单，触发价高于卖价并设置限价偏移量。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 止损限价单以正确的触发价、限价和 side=BUY 被接受。 |
| **何时跳过**      | 适配器不支持 `StopLimit` 订单时。                           |

**注意事项：**

- 需要设置 `stop_limit_offset_ticks`，用于限价相对触发价的偏移量。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=False,
    enable_stop_buys=True,
    enable_stop_sells=False,
    stop_order_type=OrderType.STOP_LIMIT,
    stop_limit_offset_ticks=50,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .enable_stop_buys(true)
    .enable_stop_sells(false)
    .stop_order_type(OrderType::StopLimit)
    .stop_limit_offset_ticks(50)
    .build()?
```

### TC-E23：StopLimit 卖单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 下一个触发价低于买价的止损限价卖单。      |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 止损限价单以正确的触发价、限价和 side=SELL 被接受。 |
| **何时跳过**      | 适配器不支持 `StopLimit` 订单时。                           |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=False,
    enable_stop_buys=False,
    enable_stop_sells=True,
    stop_order_type=OrderType.STOP_LIMIT,
    stop_limit_offset_ticks=50,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .enable_stop_buys(false)
    .enable_stop_sells(true)
    .stop_order_type(OrderType::StopLimit)
    .stop_limit_offset_ticks(50)
    .build()?
```

### TC-E24：MarketIfTouched 买单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 下一个触发价低于当前买价的 MIT 买单（逢低买入）。             |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | MIT 订单以正确的触发价在交易场所被接受。                |
| **何时跳过**      | 适配器不支持 `MarketIfTouched` 订单时。                     |

### TC-E25：MarketIfTouched 卖单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 下一个触发价高于当前卖价的 MIT 卖单（逢高卖出）。         |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | MIT 订单以正确的触发价在交易场所被接受。                |
| **何时跳过**      | 适配器不支持 `MarketIfTouched` 订单时。                     |

### TC-E26：LimitIfTouched 买单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 下一个触发价低于买价、且带限价偏移量的 LIT 买单。           |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | LIT 订单以正确的触发价和限价被接受。         |
| **何时跳过**      | 适配器不支持 `LimitIfTouched` 订单时。                      |

### TC-E27：LimitIfTouched 卖单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | 下一个触发价高于卖价、且带限价偏移量的 LIT 卖单。          |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | LIT 订单以正确的触发价和限价被接受。         |
| **何时跳过**      | 适配器不支持 `LimitIfTouched` 订单时。                      |

---

## 第 4 组：订单修改

测试订单修改（修订）和撤单重下工作流。

| TC    | 名称                         | 描述                                         | 何时跳过                   |
|-------|------------------------------|-----------------------------------------------------|-----------------------------|
| TC-E30 | 修改限价买单价格       | 将未结的限价买单修订为新价格。                  | 不支持修改时。          |
| TC-E31 | 修改限价卖单价格      | 将未结的限价卖单修订为新价格。                 | 不支持修改时。          |
| TC-E32 | 撤单重下限价买单     | 取消并以新价格重新提交限价买单。         | 从不跳过。                      |
| TC-E33 | 撤单重下限价卖单    | 取消并以新价格重新提交限价卖单。        | 从不跳过。                      |
| TC-E34 | 修改止损触发价    | 修订止损单的触发价。                     | 不支持修改或不支持止损单时。       |
| TC-E35 | 撤单重下止损单    | 取消并以新触发价重新提交止损单。      | 不支持止损单时。             |
| TC-E36 | 修改被拒绝              | 在不支持修改的适配器上进行修改。                      | 适配器支持修改时。    |

### TC-E30：修改限价买单价格

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自 TC-E10 的未结 GTC 限价买单。                                        |
| **操作**         | 随着市场变动，ExecTester 将限价买单修改为新价格
（`modify_orders_to_maintain_tob_offset=True`）。 |
| **事件序列** | `OrderPendingUpdate` -> `OrderUpdated`。                                 |
| **通过标准**  | 记录带有新价格的 `OrderUpdated` 事件；订单退出 `PendingUpdate`。 |
| **何时跳过**      | 适配器不支持订单修改时。                           |

**注意事项：**

- 需要市场发生变动才能触发 ExecTester 的订单维护逻辑。
- 当订单价格偏离目标最优档位偏移量时，会触发修改。
- 验证 `OrderUpdated` 日志显示预期的价格。如果该事件从未到达，订单会保持在
  `PendingUpdate` 状态，测试工具会停止修改它。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=False,
    modify_orders_to_maintain_tob_offset=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .modify_orders_to_maintain_tob_offset(true)
    .build()?
```

### TC-E31：修改限价卖单价格

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自 TC-E11 的未结 GTC 限价卖单。                                       |
| **操作**         | 随着市场变动，ExecTester 将限价卖单修改为新价格。           |
| **事件序列** | `OrderPendingUpdate` -> `OrderUpdated`。                                 |
| **通过标准**  | 记录带有新价格的 `OrderUpdated` 事件；订单退出 `PendingUpdate`。 |
| **何时跳过**      | 适配器不支持订单修改时。                           |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=True,
    modify_orders_to_maintain_tob_offset=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(false)
    .enable_limit_sells(true)
    .modify_orders_to_maintain_tob_offset(true)
    .build()?
```

### TC-E32：撤单重下限价买单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 未结的 GTC 限价买单。                                                    |
| **操作**         | 随着市场变动，ExecTester 取消并以新价格重新提交限价买单。 |
| **事件序列** | `OrderPendingCancel` -> `OrderCanceled` -> `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。 |
| **通过标准**  | 原订单被取消，新订单以更新后的价格被接受。          |
| **何时跳过**      | 从不跳过（撤单重下始终可用）。                            |

**注意事项：**

- 当适配器不支持原生修改时，这是通用的替代方案。
- 缓存中存在两个不同的订单：已取消的原订单和新的替换订单。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=False,
    cancel_replace_orders_to_maintain_tob_offset=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .cancel_replace_orders_to_maintain_tob_offset(true)
    .build()?
```

### TC-E33：撤单重下限价卖单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 未结的 GTC 限价卖单。                                                   |
| **操作**         | ExecTester 取消并以新价格重新提交限价卖单。              |
| **事件序列** | `OrderPendingCancel` -> `OrderCanceled` -> `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。 |
| **通过标准**  | 原订单被取消，新订单以更新后的价格被接受。          |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=False,
    enable_limit_sells=True,
    cancel_replace_orders_to_maintain_tob_offset=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(false)
    .enable_limit_sells(true)
    .cancel_replace_orders_to_maintain_tob_offset(true)
    .build()?
```

### TC-E34：修改止损触发价

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自 TC-E20 或 TC-E22 的未结止损单。                                 |
| **操作**         | 随着市场变动，ExecTester 修改止损触发价
（`modify_stop_orders_to_maintain_offset=True`）。 |
| **事件序列** | `OrderPendingUpdate` -> `OrderUpdated`。                                 |
| **通过标准**  | 记录带有新触发价的 `OrderUpdated` 事件；订单退出 `PendingUpdate`。 |
| **何时跳过**      | 适配器不支持原生止损修改，或不支持止损单时。 |

**注意事项：**

- 一些交易场所允许限价单修改，但拒绝触发单替换。对于这些适配器，
  跳过 TC-E34，改为运行 TC-E35。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_stop_buys=True,
    modify_stop_orders_to_maintain_offset=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_stop_buys(true)
    .modify_stop_orders_to_maintain_offset(true)
    .build()?
```

### TC-E35：撤单重下止损单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 未结的止损单。                                                       |
| **操作**         | ExecTester 取消并以新触发价重新提交止损单。            |
| **事件序列** | `OrderPendingCancel` -> `OrderCanceled` -> `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。 |
| **通过标准**  | 原止损单被取消，新止损单以更新后的触发价被接受。    |
| **何时跳过**      | 不支持止损单时。                                                 |

**注意事项：**

- 对于不支持原生触发单替换的交易场所，这是必需的路径。
- 在新的止损单被接受之后，重启或强制对账，并验证恰好只有一个当前触发单存在。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_stop_buys=True,
    cancel_replace_stop_orders_to_maintain_offset=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_stop_buys(true)
    .cancel_replace_stop_orders_to_maintain_offset(true)
    .build()?
```

### TC-E36：修改被拒绝

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 未结的限价单，适配器不支持修改。                     |
| **操作**         | 尝试修改该订单（以编程方式，而不是通过 ExecTester 的自动维护）。 |
| **事件序列** | `OrderModifyRejected`。                                                 |
| **通过标准**  | 修改尝试导致带有原因说明的 `OrderModifyRejected` 事件；原订单保持不变。 |
| **何时跳过**      | 适配器支持订单修改时。                                   |

**注意事项：**

- 该测试测试的是适配器的拒绝路径，而不是 ExecTester 的撤单重下逻辑。
- 拒绝原因应说明不支持修改。

---

## 第 5 组：订单取消

测试订单取消工作流。

| TC    | 名称                       | 描述                                          | 何时跳过            |
|-------|----------------------------|------------------------------------------------------|----------------------|
| TC-E40 | 取消单个限价单  | 取消一个未结的限价单。                          | 从不跳过。               |
| TC-E41 | 停止时全部取消         | 策略停止时取消所有未结订单（默认）。     | 从不跳过。               |
| TC-E42 | 停止时逐一取消 | 停止时逐一取消订单。                    | 从不跳过。               |
| TC-E43 | 停止时批量取消       | 停止时通过批量 API 取消订单。                 | 不支持批量取消时。     |
| TC-E44 | 取消已取消的订单    | 取消一个非未结状态的订单。                          | 从不跳过。               |

### TC-E40：取消单个限价单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自 TC-E10 或 TC-E11 的未结 GTC 限价单。                            |
| **操作**         | 停止策略；ExecTester 取消未结的限价单。            |
| **事件序列** | `OrderPendingCancel` -> `OrderCanceled`。                                |
| **通过标准**  | 订单状态转为 CANCELED；没有剩余的未结订单。        |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- `cancel_orders_on_stop=True`（默认值）会在策略停止时触发取消。
- 验证 `OrderCanceled` 事件包含正确的 `venue_order_id`。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=False,
    cancel_orders_on_stop=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .cancel_orders_on_stop(true)
    .build()?
```

### TC-E41：停止时全部取消

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 多个未结订单（来自 TC-E12 的限价买单 + 限价卖单）。             |
| **操作**         | 以 `cancel_orders_on_stop=True`（默认值）停止策略。         |
| **事件序列** | 对每个订单：`OrderPendingCancel` -> `OrderCanceled`。                |
| **通过标准**  | 所有未结订单都被取消；没有剩余的未结订单。                    |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=True,
    cancel_orders_on_stop=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(true)
    .cancel_orders_on_stop(true)
    .build()?
```

### TC-E42：停止时逐一取消

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 多个未结订单。                                                  |
| **操作**         | 以 `use_individual_cancels_on_stop=True` 停止。                       |
| **事件序列** | 对每个订单单独发出 `OrderPendingCancel` -> `OrderCanceled`。      |
| **通过标准**  | 每个订单被单独取消；所有订单都到达 CANCELED 状态。    |
| **何时跳过**      | 从不跳过。                                                                 |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=True,
    use_individual_cancels_on_stop=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(true)
    .use_individual_cancels_on_stop(true)
    .build()?
```

### TC-E43：停止时批量取消

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 多个未结订单，适配器支持批量取消。                   |
| **操作**         | 以 `use_batch_cancel_on_stop=True` 停止。                             |
| **事件序列** | 对所有订单批量发出 `OrderPendingCancel` -> `OrderCanceled`。           |
| **通过标准**  | 所有订单通过单次批量请求被取消；全部到达 CANCELED 状态。 |
| **何时跳过**      | 适配器不支持批量取消时。                                 |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=True,
    use_batch_cancel_on_stop=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(true)
    .use_batch_cancel_on_stop(true)
    .build()?
```

### TC-E44：取消已取消的订单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 一个先前已取消的订单（来自 TC-E40）。                             |
| **操作**         | 再次尝试取消同一个订单。                                |
| **事件序列** | `OrderCancelRejected`。                                                 |
| **通过标准**  | 取消尝试被拒绝；收到带有原因说明的 `OrderCancelRejected` 事件。 |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- 该测试测试的是适配器对无效取消请求的错误处理。
- 拒绝原因应说明该订单不处于可取消状态。

---

## 第 6 组：组合单

测试组合单提交（入场单 + 止盈单 + 止损单）。

| TC    | 名称                          | 描述                                       | 何时跳过            |
|-------|-------------------------------|-----------------------------------------------------|----------------------|
| TC-E50 | 组合买单                   | 限价买入入场 + 限价卖出止盈 + 止损卖出。   | 不支持组合单时。  |
| TC-E51 | 组合卖单                  | 限价卖出入场 + 限价买入止盈 + 止损买入。    | 不支持组合单时。  |
| TC-E52 | 组合入场成交激活止盈/止损  | 验证入场成交后止盈/止损变为激活状态。      | 不支持组合单时。  |
| TC-E53 | 带 post-only 入场的组合单  | 入场单使用 post-only 标志。                  | 不支持组合单或 post-only 时。    |

### TC-E50：组合买单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 提交一个组合单：限价买入入场 + 止盈卖单 + 止损卖单。 |
| **事件序列** | 入场单：`OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`；止盈和止损：`OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。 |
| **通过标准**  | 创建并接受三个订单：入场单在买价以下，止盈单在卖价以上，止损单在入场价以下。 |
| **何时跳过**      | 适配器不支持组合单时。                               |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_brackets=True,
    bracket_entry_order_type=OrderType.LIMIT,
    bracket_offset_ticks=500,
    enable_limit_buys=True,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_brackets(true)
    .bracket_entry_order_type(OrderType::Limit)
    .bracket_offset_ticks(500)
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .build()?
```

### TC-E51：组合卖单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 提交组合单：限价卖出入场 + 止盈买单 + 止损买单。        |
| **事件序列** | 与 TC-E50 相同模式，但方向为卖出。                              |
| **通过标准**  | 卖出方向创建并接受三个订单。                        |
| **何时跳过**      | 适配器不支持组合单时。                               |

### TC-E52：组合入场成交激活止盈/止损

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | TC-E50 中入场单已成交的组合单。                     |
| **操作**         | 入场单成交；验证条件止盈和止损单被激活。       |
| **事件序列** | 入场：`OrderFilled`；止盈和止损从条件状态转为激活状态。  |
| **通过标准**  | 入场成交之后，止盈和止损单在交易场所处于活跃状态。              |
| **何时跳过**      | 适配器不支持组合单时。                               |

**注意事项：**

- 这需要入场单真正成交，可能需要使用激进定价。
- 止盈/止损的激活机制因交易场所而异（一些立即激活，一些使用 OCA 分组）。

### TC-E53：带 post-only 入场的组合单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器支持组合单和 post-only。                               |
| **操作**         | 提交带 `use_post_only=True` 的组合单（应用于入场单和止盈单）。    |
| **事件序列** | 与 TC-E50 相同，入场单带有 post-only 标志。                           |
| **通过标准**  | 入场单和止盈单以 post-only（挂单方）身份被接受；止损单不是 post-only。 |
| **何时跳过**      | 不支持组合单或不支持 post-only 时。                            |

---

## 第 7 组：订单标志

测试订单级别的标志和特殊参数。

| TC    | 名称                 | 描述                                            | 何时跳过            |
|-------|----------------------|--------------------------------------------------------|----------------------|
| TC-E60 | PostOnly 被接受    | 带 post-only 的限价单，下在远离最优档位的位置。            | 不支持 post-only 时。        |
| TC-E61 | 平仓时使用 ReduceOnly  | 使用 reduce-only 标志平仓。                  | 不支持 reduce-only 时。      |
| TC-E62 | 显示数量     | 可见数量小于总量的冰山订单。               | 不支持显示数量时。  |
| TC-E63 | 自定义订单参数  | 通过 `order_params` 传递适配器特有的参数。            | 不适用（N/A）。                 |

### TC-E60：PostOnly 被接受

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 以被动价格下一个带 `use_post_only=True` 的限价买单。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 订单作为挂单方（maker）订单被接受；post-only 标志被交易场所确认。 |
| **何时跳过**      | 适配器不支持 post-only 标志时。                               |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=False,
    use_post_only=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .use_post_only(true)
    .build()?
```

### TC-E61：平仓时使用 ReduceOnly

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 未结持仓（来自 TC-E01）。                                           |
| **操作**         | 以 `reduce_only_on_stop=True` 停止策略；平仓单使用 reduce-only 标志。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`（带 reduce-only）。 |
| **通过标准**  | 平仓单带有 reduce-only 标志；持仓完全平仓。             |
| **何时跳过**      | 适配器不支持 reduce-only 标志时。                             |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("0.01"),
    reduce_only_on_stop=True,
    close_positions_on_stop=True,
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(1, 2))
    .reduce_only_on_stop(true)
    .close_positions_on_stop(true)
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .build()?
```

### TC-E62：显示数量（冰山）

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，适配器支持显示数量。                  |
| **操作**         | 下一个 `order_display_qty` 小于 `order_qty` 的限价单。              |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 订单以设置的显示数量被接受；订单簿上只可见显示数量。 |
| **何时跳过**      | 适配器不支持显示数量/冰山单时。            |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("1.0"),
    order_display_qty=Decimal("0.1"),
    enable_limit_buys=True,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("1.0"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .order_display_qty(Quantity::from("0.1"))
    .build()?
```

### TC-E63：自定义订单参数

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，适配器接受额外的参数。              |
| **操作**         | 下一个带有包含适配器特有参数的 `order_params` 字典的订单。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。               |
| **通过标准**  | 订单被接受；适配器特有参数原样传递给交易场所。   |
| **何时跳过**      | 不适用（取决于具体适配器）。                                                |

**注意事项：**

- `order_params` 字典对 ExecTester 是不透明的，会原样传递给适配器。
- 请查阅具体适配器的指南以了解支持的参数。

---

## 第 8 组：拒绝处理

测试适配器是否正确处理和报告订单拒绝。

| TC     | 名称                    | 描述                                      | 何时跳过        |
|--------|-------------------------|--------------------------------------------------|------------------|
| TC-E70 | PostOnly 拒绝      | 会跨越价差的 post-only 订单。     | 不支持 post-only 时。    |
| TC-E71 | ReduceOnly 拒绝    | 没有可减仓位的 reduce-only 订单。    | 不支持 reduce-only 时。  |
| TC-E72 | 不支持的订单类型  | 提交适配器不支持的订单类型。      | 从不跳过。           |
| TC-E73 | 不支持的有效期         | 提交带有不支持有效期的订单。     | 从不跳过。           |
| TC-E74 | 提交结果不明确   | 提交时的传输、超时或发送失败。   | 无 mock 路径时。    |
| TC-E75 | 取消结果不明确   | 取消时的传输、超时或发送失败。   | 不支持取消时。       |
| TC-E76 | 修改结果不明确   | 修改时的传输、超时或发送失败。   | 不支持修改时。       |
| TC-E77 | 批量结果不明确    | 没有逐订单结果的整批失败。   | 不支持批量时。        |
| TC-E78 | 逐订单批量拒绝  | 批量响应中带有明确的逐订单拒绝。 | 不支持批量时。        |

TC-E74 到 TC-E78 在下方被统一说明，因为它们通常需要一个模拟的 HTTP 或
WebSocket 边界，而不是一个实盘交易场所。

### 结果不明确的失败

这些用例证明了当交易场所结果未知时，适配器的请求失败不会转变为终止性的拒绝事件。
通过标准还定义了本地准备失败的例外情况：当一个命令已知未被发送、且可以归因于
单个取消或修改命令时，适配器可以发出对应的拒绝事件。

**通过标准：**

- 由传输错误、超时、WebSocket 发送失败、重试耗尽或响应解析失败导致的提交失败，
  不会发出 `OrderRejected`。
- 由传输错误、超时、WebSocket 发送失败、重试耗尽或整体请求服务器失败导致的取消失败，
  不会发出 `OrderCancelRejected`。
- 由传输错误、超时、WebSocket 发送失败、重试耗尽或整体请求服务器失败导致的修改失败，
  不会发出 `OrderModifyRejected`。
- 当适配器能够将失败归因于某个取消命令时，能证明该命令无法发送的本地取消准备失败，
  可以发出 `OrderCancelRejected`。
- 当适配器能够将失败归因于某个修改命令时，能证明该命令无法发送的本地修改准备失败，
  可以发出 `OrderModifyRejected`。
- 当交易场所没有返回逐订单结果时，整批请求失败不会为每个订单发出一个拒绝事件。
- 明确的逐订单交易场所拒绝仍然会发出带有交易场所原因的对应拒绝事件。

该订单保持在适当的在途状态，直到某个交易场所更新、查询结果或对账流程解析它。

### TC-E70：PostOnly 拒绝

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，报价在流动。                  |
| **操作**         | ExecTester 在订单簿的错误一侧下一个 post-only 订单
（`test_reject_post_only=True`），导致其跨越价差。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderRejected`。               |
| **通过标准**  | 交易场所拒绝订单；`OrderRejected.due_post_only=true`；原因说明了 post-only 违规。 |
| **何时跳过**      | 适配器不支持 post-only 标志时。                               |

**注意事项：**

- ExecTester 的 `test_reject_post_only` 模式故意为该订单定价，使其发生跨越。
- 一些交易场所可能会部分成交而不是拒绝；行为因交易场所而异。
- 对于 post-only 跨越导致的拒绝，发出 `OrderRejected` 的适配器应设置
  `due_post_only=true`，以便策略能将其与其他交易场所拒绝区分开来。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    enable_limit_buys=True,
    enable_limit_sells=False,
    use_post_only=True,
    test_reject_post_only=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .enable_limit_buys(true)
    .enable_limit_sells(false)
    .use_post_only(true)
    .test_reject_post_only(true)
    .build()?
```

### TC-E71：ReduceOnly 拒绝

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，该金融工具没有未结持仓。                |
| **操作**         | 在没有可减仓位的情况下，ExecTester 通过 `test_reject_reduce_only=True` 和
`open_position_on_start_qty` 以 `reduce_only=True` 开仓。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderRejected`。               |
| **通过标准**  | 订单被拒绝；`OrderRejected` 事件的原因说明了 reduce-only 违规。 |
| **何时跳过**      | 适配器不支持 reduce-only 标志时。                             |

**注意事项：**

- `test_reject_reduce_only` 标志只适用于通过 `open_position_on_start_qty` 提交的
  开仓市价单。
- 在运行此测试之前，验证该金融工具之前没有持仓。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("0.01"),
    test_reject_reduce_only=True,
    enable_limit_buys=False,
    enable_limit_sells=False,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(1, 2))
    .test_reject_reduce_only(true)
    .enable_limit_buys(false)
    .enable_limit_sells(false)
    .build()?
```

### TC-E72：不支持的订单类型

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，该订单类型不在适配器支持的集合中。          |
| **操作**         | 提交适配器不支持的订单类型。                     |
| **事件序列** | `OrderDenied`（提交前由适配器拒绝）。                   |
| **通过标准**  | 订单在到达交易场所之前被拒绝；`OrderDenied` 事件带有原因。   |
| **何时跳过**      | 从不跳过（每个适配器都有可测试的不支持订单类型）。             |

**注意事项：**

- `OrderDenied` 发生在适配器层面，在订单到达交易场所之前。
- 这与来自交易场所的 `OrderRejected` 不同。
- 通过配置适配器不支持的止损单类型来测试。

### TC-E73：不支持的有效期

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，该有效期不在适配器支持的集合中。                 |
| **操作**         | 提交带有适配器不支持的有效期的订单。               |
| **事件序列** | `OrderDenied`（提交前由适配器拒绝）。                   |
| **通过标准**  | 订单在到达交易场所之前被拒绝；`OrderDenied` 事件带有原因。   |
| **何时跳过**      | 从不跳过（每个适配器都有可测试的不支持有效期选项）。             |

**注意事项：**

- 与 TC-E72 类似，但针对有效期选项。
- 用适配器未映射的 Nautilus 枚举中的有效期值进行测试。

---

## 第 9 组：生命周期（启动/停止）

测试策略在启动和停止时的生命周期行为和状态管理。

| TC     | 名称                        | 描述                                            | 何时跳过            |
|--------|-----------------------------|--------------------------------------------------------|----------------------|
| TC-E80 | 启动时开仓      | 策略启动时立即开仓。      | 不支持市价单时。    |
| TC-E81 | 停止时取消订单       | 策略停止时取消所有未结订单。             | 从不跳过。               |
| TC-E82 | 停止时平仓     | 策略停止时平掉未结持仓。               | 不支持市价单时。    |
| TC-E83 | 停止时取消订阅         | 策略停止时取消数据流订阅。           | 不支持取消订阅时。    |
| TC-E84 | 对账未结订单       | 对账来自之前会话的现有未结订单。    | 从不跳过。               |
| TC-E85 | 对账已成交订单     | 对账来自之前会话的、先前已成交的订单。| 从不跳过。               |
| TC-E86 | 对账未结多头       | 对账现有的未结多头持仓。                  | 从不跳过。               |
| TC-E87 | 对账未结空头        | 对账现有的未结空头持仓。                 | 从不跳过。               |

### TC-E80：启动时开仓

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，金融工具已加载，没有现存持仓。            |
| **操作**         | 策略以设置了 `open_position_on_start_qty` 启动。                 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 启动时开仓；市价单在限价单维护开始之前提交并成交。 |
| **何时跳过**      | 适配器不支持市价单时。                                |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    open_position_on_start_qty=Decimal("0.01"),
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .open_position_on_start_qty(Decimal::new(1, 2))
    .build()?
```

### TC-E81：停止时取消订单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自策略会话的未结限价单。                           |
| **操作**         | 以 `cancel_orders_on_stop=True`（默认值）停止策略。         |
| **事件序列** | 对每个未结订单：`OrderPendingCancel` -> `OrderCanceled`。           |
| **通过标准**  | 所有属于该策略的未结订单在停止时都被取消。                       |
| **何时跳过**      | 从不跳过。                                                                 |

### TC-E82：停止时平仓

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自策略会话的未结持仓。                               |
| **操作**         | 以 `close_positions_on_stop=True`（默认值）停止策略。       |
| **事件序列** | 平仓单：`OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled`。 |
| **通过标准**  | 所有属于该策略的持仓都被平仓；净持仓 = 0。                 |
| **何时跳过**      | 适配器不支持市价单时。                                |

### TC-E83：停止时取消订阅

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 活跃的数据订阅（报价、成交、订单簿）。                      |
| **操作**         | 以 `can_unsubscribe=True`（默认值）停止策略。               |
| **事件序列** | 数据订阅被移除。                                            |
| **通过标准**  | 停止之后不再收到数据事件；干净地断开连接。       |
| **何时跳过**      | 适配器不支持取消订阅时。                                  |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,
    order_qty=Decimal("0.01"),
    can_unsubscribe=True,
)
```

**Rust 配置：**

```rust
ExecTesterConfig::builder()
    .base(StrategyConfig {
        strategy_id: Some(strategy_id),
        ..Default::default()
    })
    .instrument_id(instrument_id)
    .client_id(client_id)
    .order_qty(Quantity::from("0.01"))
    .can_unsubscribe(true)
    .build()?
```

### TC-E84：对账未结订单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自之前会话、在交易场所上存在一个或多个未结限价单。       |
| **操作**         | 以 `reconciliation=True` 启动节点。                             |
| **事件序列** | 为每个未结订单生成 `OrderStatusReport`。                     |
| **通过标准**  | 每个未结订单都以正确的 `venue_order_id`、status=ACCEPTED、价格、数量、方向和订单类型加载到缓存中。 |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- 从之前的测试会话中保留未结的限价单（停止时不取消）。
- 使用 `external_order_claims` 认领该金融工具，使适配器为其进行对账。
- 验证对账后的订单数量与交易场所报告的数量相符。

### TC-E85：对账已成交订单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自之前会话、在交易场所上存在一个或多个已成交订单。           |
| **操作**         | 以 `reconciliation=True` 启动节点。                             |
| **事件序列** | 为每次历史成交生成 `FillReport`。                       |
| **通过标准**  | 每个已成交订单都以正确的 `venue_order_id`、status=FILLED、成交价格、成交数量和佣金加载到缓存中。 |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- 需要在之前会话中已成交的订单。
- 验证成交价格、数量和佣金与交易场所报告的值相符。
- 一些适配器可能只报告回溯窗口内的成交。

### TC-E86：对账未结多头持仓

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自之前会话、在交易场所上存在一个未结多头持仓。               |
| **操作**         | 以 `reconciliation=True` 启动节点。                             |
| **事件序列** | 为该多头持仓生成 `PositionStatusReport`。                |
| **通过标准**  | 持仓以正确的金融工具、side=LONG、数量和与交易场所相符的入场价加载到缓存中。 |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- 在之前的会话中开一个多头持仓，并在不平仓的情况下停止策略
  （`close_positions_on_stop=False`）。
- 验证对账后的持仓数量和平均入场价与交易场所相符。
- 对账之后，策略应能够管理或平掉该持仓。

### TC-E87：对账未结空头持仓

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自之前会话、在交易场所上存在一个未结空头持仓。              |
| **操作**         | 以 `reconciliation=True` 启动节点。                             |
| **事件序列** | 为该空头持仓生成 `PositionStatusReport`。               |
| **通过标准**  | 持仓以正确的金融工具、side=SHORT、数量和与交易场所相符的入场价加载到缓存中。 |
| **何时跳过**      | 从不跳过。                                                                 |

**注意事项：**

- 在之前的会话中开一个空头持仓，并在不平仓的情况下停止策略
  （`close_positions_on_stop=False`）。
- 验证对账后的持仓数量和平均入场价与交易场所相符。
- 对账之后，策略应能够管理或平掉该持仓。

---

## 第 10 组：期权交易

测试期权特有的执行行为。期权金融工具通常与线性衍生品有着不同的约束：交易场所可能会
限制订单类型、支持替代的定价模式，或不允许条件单。具体限制因交易场所而异；
请查阅适配器指南。

这些测试需要一个 `CryptoOption` 金融工具。使用一个具备合理流动性的价外（OTM）期权
以便成交。

| TC      | 名称                          | 描述                                                      | 何时跳过              |
|---------|-------------------------------|--------------------------------------------------------------------------------------|------------------------|
| TC-E90  | 期权限价买单              | 在期权金融工具上下一个限价买单。                       | 不支持期权时。    |
| TC-E91  | 期权限价卖单             | 在期权金融工具上下一个限价卖单。                      | 不支持期权时。    |
| TC-E92  | 带替代定价的限价单        | 通过 `order_params` 下一个带适配器特有定价的限价单。 | 不支持替代定价时。    |
| TC-E94  | 不支持的订单类型被拒绝 | 提交适配器对期权拒绝的订单类型。            | 不支持期权时。    |
| TC-E96  | 条件单被拒绝    | 在期权上提交止损/条件单；预期被拒绝。  | 不支持期权时。    |
| TC-E99  | 期权 FOK 限价单              | 在期权金融工具上下一个 FOK 限价单。                 | 不支持期权 FOK 时。        |
| TC-E100 | 取消期权订单           | 取消期权金融工具上的一个未结限价单。              | 不支持期权时。    |
| TC-E101 | 对账期权持仓     | 对账来自之前会话的未结期权持仓。          | 不支持期权时。    |

### TC-E90：期权限价买单

| 字段              | 值                                                                       |
|--------------------|-----------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，期权金融工具已加载，报价在流动。                |
| **操作**         | ExecTester 以被动价格在该期权上下一个限价买单。             |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。                    |
| **通过标准**  | 订单以正确的金融工具、方向、价格和数量被交易场所接受。 |
| **何时跳过**      | 适配器不支持期权交易时。                                   |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,  # CryptoOption 金融工具
    order_qty=Decimal("1"),
    enable_limit_buys=True,
    enable_limit_sells=False,
    tob_offset_ticks=500,
)
```

### TC-E91：期权限价卖单

| 字段              | 值                                                                       |
|--------------------|-----------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，期权金融工具已加载，报价在流动。                |
| **操作**         | ExecTester 以被动价格在该期权上下一个限价卖单。            |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。                    |
| **通过标准**  | 订单以正确的金融工具、方向、价格和数量被交易场所接受。 |
| **何时跳过**      | 适配器不支持期权交易时。                                   |

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,  # CryptoOption 金融工具
    order_qty=Decimal("1"),
    enable_limit_buys=False,
    enable_limit_sells=True,
    tob_offset_ticks=500,
)
```

### TC-E92：带替代定价的限价单

| 字段              | 值                                                               |
|--------------------|---------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，期权金融工具已加载。                        |
| **操作**         | 通过 `order_params` 下一个带适配器特有定价的限价单。 |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted`。            |
| **通过标准**  | 订单被接受；交易场所确认了该替代定价模式。    |
| **何时跳过**      | 适配器不支持期权的替代定价模式时。     |

**注意事项：**

- 当替代定价处于激活状态时，订单对象上的 `price` 字段可能是一个占位符。
  请查阅适配器指南以了解支持的参数键。
- 示例：OKX 支持 `px_usd`（美元价格）和 `px_vol`（隐含波动率）。
- 在交易场所响应中验证定价模式被正确反映。

**Python 配置：**

```python
ExecTesterConfig(
    instrument_id=instrument_id,  # CryptoOption 金融工具
    order_qty=Decimal("1"),
    enable_limit_buys=True,
    enable_limit_sells=False,
    order_params={"px_usd": "100.5"},  # 适配器特有的定价键
)
```

### TC-E94：期权不支持的订单类型被拒绝

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，期权金融工具已加载。                           |
| **操作**         | 提交该交易场所对期权不支持的订单类型（例如市价单）。 |
| **事件序列** | 取决于适配器：`OrderDenied`（提交前）或 `OrderSubmitted` -> `OrderRejected`（提交后）。 |
| **通过标准**  | 订单不会成交。拒绝原因引用了该不支持的订单类型。 |
| **何时跳过**      | 适配器不支持期权时。                                      |

**注意事项：**

- 确切的拒绝时点因适配器而异。一些适配器在本地提交前拒绝；
  另一些提交后转发交易场所的拒绝。
- ExecTester 可以在期权金融工具上通过 `open_position_on_start_qty` 触发市价单。
  一些不支持的类型（例如 `MarketToLimit`）需要手动或编程方式提交。
- 测试适配器文档中记录的每一种不支持的类型。

### TC-E96：期权条件单被拒绝

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，期权金融工具已加载。                           |
| **操作**         | 在期权金融工具上提交一个条件单。                    |
| **事件序列** | 取决于适配器：`OrderDenied`（提交前）或 `OrderSubmitted` -> `OrderRejected`（提交后）。 |
| **通过标准**  | 订单不会成交。原因引用了不支持的条件单类型。 |
| **何时跳过**      | 适配器不支持期权，或适配器支持期权条件单时。 |

**注意事项：**

- 测试适配器文档中记录的、对期权不支持的每一种条件单类型
  （例如 `STOP_MARKET`、`STOP_LIMIT`、`MARKET_IF_TOUCHED`、`LIMIT_IF_TOUCHED`、
  `TRAILING_STOP_MARKET`）。
- ExecTester 可以在期权金融工具上通过 `enable_stop_buys`/`enable_stop_sells`
  加上 `stop_order_type` 触发条件单。

### TC-E99：期权 FOK 限价单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 适配器已连接，期权金融工具已加载，订单簿深度充足。    |
| **操作**         | 在期权金融工具上下一个带 `TimeInForce::Fok` 的限价单。   |
| **事件序列** | `OrderInitialized` -> `OrderSubmitted` -> `OrderAccepted` -> `OrderFilled` 或 `OrderCanceled`。 |
| **通过标准**  | 订单完全成交或被取消。没有部分成交。               |
| **何时跳过**      | 适配器不支持期权的 FOK 时。                              |

**注意事项：**

- 一些交易场所为期权 FOK 订单使用专用的订单类型（例如 OKX 使用
  `op_fok`）。适配器会透明地处理这种映射。
- 使用较小的数量和激进的定价，为正向用例获取一次成交。

### TC-E100：取消期权订单

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自 TC-E90 或 TC-E91 的未结限价单。                                |
| **操作**         | 取消该未结限价单。                                           |
| **事件序列** | `OrderPendingCancel` -> `OrderCanceled`。                                 |
| **通过标准**  | 订单被取消；不再出现在交易场所的未结订单中。         |
| **何时跳过**      | 适配器不支持期权时。                                      |

### TC-E101：对账期权持仓

| 字段              | 值                                                                  |
|--------------------|------------------------------------------------------------------------|
| **前置条件**   | 来自之前会话的未结期权持仓。                             |
| **操作**         | 以 `reconciliation=True` 启动节点。                             |
| **事件序列** | 为该期权持仓生成 `PositionStatusReport`。              |
| **通过标准**  | 持仓以正确的金融工具、方向、数量和入场价加载到缓存中。 |
| **何时跳过**      | 适配器不支持期权时。                                      |

**注意事项：**

- 在之前的会话中开一个期权持仓，并在不平仓的情况下停止
  （`close_positions_on_stop=False`）。
- 验证对账后的持仓与交易场所报告的状态相符。

---

## ExecTester 配置参考

所有 `ExecTesterConfig` 参数的快速参考。所展示的默认值针对的是 Python 配置；
Rust 构建器使用等效的默认值。

| 参数                                       | 类型              | 默认值         | 影响的分组 |
|-------------------------------------------------|-------------------|-----------------|----------------|
| `instrument_id`                                 | InstrumentId      | *必填*      | 全部            |
| `order_qty`                                     | Decimal           | *必填*      | 全部            |
| `order_display_qty`                             | Decimal?          | None            | 2、7           |
| `order_expire_time_delta_mins`                  | PositiveInt?      | None            | 2              |
| `order_params`                                  | dict?             | None            | 7、10          |
| `client_id`                                     | ClientId?         | None            | 全部            |
| `subscribe_quotes`                              | bool              | True            |                |
| `subscribe_trades`                              | bool              | True            |                |
| `subscribe_book`                                | bool              | False           |                |
| `book_type`                                     | BookType          | L2_MBP          |                |
| `book_depth`                                    | PositiveInt?      | None            |                |
| `book_interval_ms`                              | PositiveInt       | 1000            |                |
| `book_levels_to_print`                          | PositiveInt       | 10              |                |
| `open_position_on_start_qty`                    | Decimal?          | None            | 1、9           |
| `open_position_time_in_force`                   | TimeInForce       | GTC             | 1              |
| `enable_limit_buys`                             | bool              | True            | 2、4、5、6     |
| `enable_limit_sells`                            | bool              | True            | 2、4、5、6     |
| `enable_stop_buys`                              | bool              | False           | 3、4           |
| `enable_stop_sells`                             | bool              | False           | 3、4           |
| `limit_time_in_force`                           | TimeInForce?      | None            | 2、6           |
| `tob_offset_ticks`                              | PositiveInt       | 500             | 2、4           |
| `stop_order_type`                               | OrderType         | STOP_MARKET     | 3              |
| `stop_offset_ticks`                             | PositiveInt       | 100             | 3              |
| `stop_limit_offset_ticks`                       | PositiveInt?      | None            | 3              |
| `stop_time_in_force`                            | TimeInForce?      | None            | 3              |
| `stop_trigger_type`                             | TriggerType?      | None            | 3              |
| `enable_brackets`                               | bool              | False           | 6              |
| `bracket_entry_order_type`                      | OrderType         | LIMIT           | 6              |
| `bracket_offset_ticks`                          | PositiveInt       | 500             | 6              |
| `modify_orders_to_maintain_tob_offset`          | bool              | False           | 4              |
| `modify_stop_orders_to_maintain_offset`         | bool              | False           | 4              |
| `cancel_replace_orders_to_maintain_tob_offset`  | bool              | False           | 4              |
| `cancel_replace_stop_orders_to_maintain_offset` | bool              | False           | 4              |
| `use_post_only`                                 | bool              | False           | 2、6、7、8     |
| `use_quote_quantity`                            | bool              | False           | 1、7           |
| `emulation_trigger`                             | TriggerType?      | None            | 2、3           |
| `cancel_orders_on_stop`                         | bool              | True            | 5、9           |
| `close_positions_on_stop`                       | bool              | True            | 9              |
| `close_positions_time_in_force`                 | TimeInForce?      | None            | 9              |
| `reduce_only_on_stop`                           | bool              | True            | 7、9           |
| `use_individual_cancels_on_stop`                | bool              | False           | 5              |
| `use_batch_cancel_on_stop`                      | bool              | False           | 5              |
| `dry_run`                                       | bool              | False           |                |
| `log_data`                                      | bool              | True            |                |
| `test_reject_post_only`                         | bool              | False           | 8              |
| `test_reject_reduce_only`                       | bool              | False           | 8              |
| `can_unsubscribe`                               | bool              | True            | 9              |
