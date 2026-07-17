# 仓位（Positions）

本指南解释 NautilusTrader 中仓位的工作方式，包括其生命周期、从订单成交
聚合的过程、盈亏计算，以及净额（netting）OMS 配置中仓位快照
这一重要概念。

## 概览

仓位表示对市场中某个特定金融工具的一个未平仓敞口。仓位是跟踪
交易表现和风险的基础，因为它们聚合了针对某个特定金融工具的所有成交，
并持续计算诸如未实现盈亏、平均入场价格和总敞口等指标。

系统在订单成交时会自动创建仓位，并跟踪其从开仓到平仓的整个过程。
该平台通过其 OMS（订单管理系统）配置，同时支持净额（netting）和
对冲（hedging）两种仓位管理风格。

## 仓位生命周期

### 创建

系统在首次成交时开立仓位：

- **NETTING OMS**：在某个金融工具首次成交时开立（每个金融工具一个仓位）。
- **HEDGING OMS**：在某个新 `position_id` 首次成交时开立（每个金融工具可有多个仓位）。

一个仓位跟踪：

- 开仓订单和成交详情。
- 入场方向（`LONG` 或 `SHORT`）。
- 初始数量和平均价格。
- 初始化和开仓的时间戳。

:::tip
你可以在 actor/策略内部通过 `self.cache.position(position_id)` 或
`self.cache.positions(instrument_id=instrument_id)` 从缓存中访问仓位。
:::

### 更新

随着更多成交的发生，仓位会：

- 聚合买入和卖出成交的数量。
- 重新计算平均入场价格和平均出场价格。
- 更新峰值数量（达到的最大敞口）。
- 跟踪所有关联的订单 ID 和成交 ID。
- 按币种累积手续费。

### 关闭

当净数量变为零（`FLAT`）时，一个仓位关闭。关闭时：

- 记录平仓订单 ID。
- 计算从开仓到平仓的持续时间。
- 计算最终已实现盈亏。
- 在 `NETTING` OMS 中，当该仓位之后重新开仓时，引擎会对已关闭的状态
  进行快照以保留历史盈亏（参见[仓位快照](#position-snapshotting)）。

## 订单成交聚合

仓位聚合订单成交，以维护对市场敞口的准确视图。该聚合过程
同时处理交易活动的两个方向：

### 买入成交

当一个 BUY 订单成交时：

- 增加多头敞口或减少空头敞口。
- 更新开仓交易的平均入场价格。
- 更新平仓交易的平均出场价格。
- 计算任何已平仓部分的已实现盈亏。

### 卖出成交

当一个 SELL 订单成交时：

- 增加空头敞口或减少多头敞口。
- 更新开仓交易的平均入场价格。
- 更新平仓交易的平均出场价格。
- 计算任何已平仓部分的已实现盈亏。

### 净仓位计算

仓位维护一个 `signed_qty` 字段，表示净敞口：

- 正值表示 `LONG`（多头）仓位。
- 负值表示 `SHORT`（空头）仓位。
- 零表示 `FLAT`（已平仓）仓位。

```python
# Example: Position aggregation
# Initial BUY 100 units at $50
signed_qty = +100  # LONG position

# Subsequent SELL 150 units at $55
signed_qty = -50   # Now SHORT position

# Final BUY 50 units at $52
signed_qty = 0     # Position FLAT (closed)
```

## 仓位调整

仓位调整跟踪发生在正常订单成交之外的数量或盈亏变化，
以确保仓位数量准确反映真实的净资产头寸。系统会为这些场景生成
`PositionAdjusted` 事件。

### 基础货币手续费

在交易现货货币对（例如 BTC/USDT）或外汇现货时，以基础货币支付的
手续费会直接影响收到或交付的净数量：

- **开仓成交**：手续费从成交数量中扣除。买入 1.0 BTC，手续费
  0.001 BTC，最终得到 0.999 BTC 的多头仓位净数量。
- **平仓成交**：手续费会应用于 `signed_qty`，因为它影响实际库存。
  卖出一个 0.999 BTC 的 LONG 仓位，手续费为 0.000999 BTC，最终会使你
  处于 SHORT 0.000999 BTC 的状态，而不是 FLAT，因为你总共交出了
  0.999999 BTC。
- **翻转（Flips）**：手续费会影响翻转两侧的最终仓位规模。

:::note
基础货币手续费仅适用于现货货币对和外汇现货金融工具，且手续费
货币需与 `instrument.base_currency` 一致。对于其他金融工具，
手续费会单独跟踪，不影响仓位数量。
:::

### 资金费用

资金调整（Funding adjustments）跟踪永续期货的周期性资金费用支付，
不影响仓位数量。这些记录的 `quantity_change = None`，且可能包含对
盈亏的影响。

### 调整跟踪

所有调整都保留在仓位的事件历史中：

- `position.adjustments` 返回所有 `PositionAdjusted` 事件的列表。
- 每次调整都包含类型（`COMMISSION` 或 `FUNDING`）、数量变化和时间戳。
- 当仓位关闭并重新开仓时，调整历史会被清除。当事件被清除时，
  与被移除成交相关的手续费调整会被重新生成，而非手续费类的调整
  （例如资金费用）会被保留。

## OMS 类型与仓位管理

NautilusTrader 支持两种主要的 OMS 类型，它们从根本上影响仓位被
跟踪和管理的方式。还存在一个 `OmsType.UNSPECIFIED` 选项，它默认使用
组件所处的上下文。完整详情请参阅[执行指南](execution.md#order-management-system-oms)。

### `NETTING`（净额）

在 `NETTING` 模式下，某个金融工具的所有成交都被聚合为单一仓位：

- 每个金融工具 ID 一个仓位。
- 所有成交都汇入同一个仓位。
- 随着净数量的变化，仓位在 `LONG` 和 `SHORT` 之间翻转。
- 历史快照保留已关闭仓位的状态。

### `HEDGING`（对冲）

在 `HEDGING` 模式下，同一个金融工具可以同时存在多个仓位：

- 可以同时存在多个 `LONG` 和 `SHORT` 仓位。
- 每个仓位都有唯一的仓位 ID。
- 各仓位被独立跟踪。
- 各仓位之间不会自动净额。
- 已关闭的仓位仍保留在缓存历史中，但不会重新开仓；新的成交会
  创建新的仓位。

:::warning
使用 `HEDGING` 模式时，请注意保证金要求会增加，因为每个仓位
都独立占用保证金。有些交易场所可能不支持真正的对冲模式，
会自动对仓位进行净额处理。
:::

### 策略 OMS 与交易场所 OMS

该平台允许策略和交易场所使用不同的 OMS 配置：

| 策略 OMS | 交易场所 OMS | 行为                                                    |
|--------------|-----------|-------------------------------------------------------------|
| `NETTING`    | `NETTING` | 策略和交易场所两端都是每个金融工具一个仓位。  |
| `HEDGING`    | `HEDGING` | 两端都支持多个仓位。                |
| `NETTING`    | `HEDGING` | 交易场所跟踪多个仓位，Nautilus 维护单一仓位。  |
| `HEDGING`    | `NETTING` | 交易场所跟踪单一仓位，Nautilus 维护虚拟仓位。  |

:::tip
对于大多数交易场景，保持策略和交易场所的 OMS 类型一致会简化
仓位管理。覆盖配置主要适用于自营交易台，或需要与遗留系统
对接的场景。有关特定交易场所的 OMS 配置，请参阅[实盘指南](live.md)。
:::

## 仓位快照

仓位快照是 `NETTING` OMS 配置的一项重要功能，用于保留已关闭仓位的
状态，以实现准确的盈亏跟踪和报告。

### 为什么快照很重要

在 `NETTING` 系统中，当一个仓位关闭（变为 `FLAT`）之后又因新的交易
重新开仓时，仓位对象会被重置以跟踪新的敞口。如果没有快照，
上一个仓位周期的历史已实现盈亏就会丢失。

### 工作原理

当一个 `NETTING` 仓位关闭之后，又收到同一金融工具的新成交时，执行
引擎会在重置该仓位之前对其已关闭状态进行快照，保留以下内容：

- 最终数量和价格。
- 已实现盈亏。
- 所有成交事件。
- 手续费总额。

该快照以仓位 ID 为索引存储在缓存中。随后该仓位会为新周期重置，
而之前的快照仍然可以访问。Portfolio 会跨所有快照聚合盈亏，
以得到准确的总额。

:::note
这种历史快照机制不同于可选的仓位状态快照（`snapshot_positions`），
后者会周期性地记录未平仓仓位的状态用于遥测。有关
`snapshot_positions` 和 `snapshot_positions_interval_secs` 设置，请参阅
[实盘指南](live.md)。
:::

### 示例场景

```python
# NETTING OMS Example
# Cycle 1: Open LONG position
BUY 100 units at $50   # Position opens
SELL 100 units at $55  # Position closes, PnL = $500
# Snapshot taken preserving $500 realized PnL

# Cycle 2: Open SHORT position
SELL 50 units at $54   # Position reopens (SHORT)
BUY 50 units at $52    # Position closes, PnL = $100
# Snapshot taken preserving $100 realized PnL

# Total realized PnL = $500 + $100 = $600 (from snapshots)
```

如果没有快照，则只有最近一个周期的盈亏可用，会导致
报告和分析出现错误。

## 盈亏计算

NautilusTrader 提供的盈亏计算会考虑金融工具的规格和
市场惯例。

### 已实现盈亏

在仓位部分或全部平仓时计算：

```python
# For standard instruments
realized_pnl = (exit_price - entry_price) * closed_quantity * multiplier

# For inverse instruments (side-aware)
# LONG: realized_pnl = closed_quantity * multiplier * (1/entry_price - 1/exit_price)
# SHORT: realized_pnl = closed_quantity * multiplier * (1/exit_price - 1/entry_price)
```

引擎会根据仓位方向自动应用正确的公式。

### 未实现盈亏

使用未平仓仓位的当前市场价格计算。`price` 参数可接受任意
参考价格（买价、卖价、中间价、最新价或标记价）：

```python
position.unrealized_pnl(last_price)  # Using last traded price
position.unrealized_pnl(bid_price)   # Conservative for LONG positions
position.unrealized_pnl(ask_price)   # Conservative for SHORT positions
```

无论提供什么价格，对于 `FLAT` 仓位都返回 `Money(0, cost_currency)`。

### 总盈亏

结合已实现和未实现部分：

```python
total_pnl = position.total_pnl(current_price)
# Returns realized_pnl + unrealized_pnl
```

### 货币方面的考虑

- 盈亏以金融工具的成本货币计算：线性合约为报价货币，反向合约为
  基础货币，数量对冲（quanto）合约为结算货币。
- 对于外汇，成本货币通常是报价货币。
- Portfolio 会按金融工具、以成本货币聚合已实现盈亏。
- 多币种汇总需要在 Position 类之外进行转换。

## 手续费和成本

仓位会跟踪所有交易成本：

- 手续费按币种累积。
- 每次成交的手续费都会加到累计总额中。
- 支持多种手续费货币。
- 仅当手续费以仓位的成本货币计价时，已实现盈亏才会包含手续费。
- 其他货币的手续费会单独跟踪，可能需要转换。

```python
commissions = position.commissions()
# Returns list[Money] with aggregated commission totals per currency

notional = position.notional_value(current_price)
# Returns Money in quote (linear), base (inverse), or settlement currency (quanto)
```

**限制：**

- 如果反向金融工具未设置 `base_currency`，会引发 panic。

## 仓位属性与状态

### 标识符

- `id`：唯一的仓位标识符。
- `instrument_id`：被交易的金融工具。
- `account_id`：持有该仓位的账户。
- `trader_id`：拥有该仓位的交易者。
- `strategy_id`：管理该仓位的策略。
- `opening_order_id`：开仓时的客户端订单 ID。
- `closing_order_id`：平仓时的客户端订单 ID。

### 仓位状态

- `side`：当前仓位方向（`LONG`、`SHORT` 或 `FLAT`）。
- `entry`：当前未平仓仓位的方向（`LONG` 对应 `Buy`，`SHORT` 对应 `Sell`）。当仓位翻转方向时会更新。
- `quantity`：当前的仓位绝对规模。
- `signed_qty`：带符号的仓位规模（`LONG` 为正，`SHORT` 为负）。
- `peak_qty`：仓位生命周期内达到的最大数量。
- `is_open`：仓位当前是否处于未平仓状态。
- `is_closed`：仓位是否已关闭（`FLAT`）。
- `is_long`：仓位方向是否为 `LONG`。
- `is_short`：仓位方向是否为 `SHORT`。

### 定价与估值

- `avg_px_open`：平均入场价格。
- `avg_px_close`：平仓时的平均出场价格。
- `realized_pnl`：已实现盈亏。
- `realized_return`：以小数表示的已实现收益率（例如 0.05 表示 5%）。
- `quote_currency`：金融工具的报价货币。
- `base_currency`：基础货币（如适用）。
- `settlement_currency`：用于盈亏结算的货币。

### 金融工具规格

- `multiplier`：合约乘数。
- `price_precision`：价格的小数精度。
- `size_precision`：数量的小数精度。
- `is_inverse`：金融工具是否为反向合约。

### 时间戳

- `ts_init`：仓位被初始化的时间。
- `ts_opened`：仓位被开仓的时间。
- `ts_last`：最后一次更新的时间戳。
- `ts_closed`：仓位被关闭的时间。
- `duration_ns`：从开仓到平仓的持续时间（纳秒）。

### 关联数据

- `symbol`：该金融工具的代码。
- `venue`：交易场所。
- `client_order_ids`：与该仓位关联的所有客户端订单 ID。
- `venue_order_ids`：与该仓位关联的所有交易场所订单 ID。
- `trade_ids`：来自交易场所的所有成交 ID。
- `events`：应用于该仓位的所有订单成交事件。
- `event_count`：已应用的成交事件总数。
- `last_event`：最近一次的成交事件。
- `last_trade_id`：最近一次的成交 ID。

:::info
有关完整的类型信息和详细的属性文档，请参阅 Position 的
[API 参考文档](/docs/python-api-latest/model/position.html#nautilus_trader.model.position.Position)。
:::

## 事件与跟踪

仓位维护着完整的事件历史：

- 所有订单成交事件都按时间顺序存储。
- 会跟踪关联的客户端订单 ID。
- 来自交易场所的成交 ID 会被保留。
- 事件计数表示已应用的成交总数。

这些历史数据支持：

- 详细的仓位分析。
- 交易核对（reconciliation）。
- 绩效归因（performance attribution）。
- 审计追踪。

:::tip
使用 `position.events` 来访问完整的成交历史，以进行核对。
`position.trade_ids` 属性有助于与经纪商对账单进行匹配。
有关核对的最佳实践，请参阅[执行指南](execution.md)。
:::

## 数值精度

仓位计算对盈亏和平均价格的运算使用 64 位浮点数（`f64`）算术。
虽然定点类型（`Price`、`Quantity`、`Money`）在配置的小数位数上保留了
精确精度，但内部计算会转换为 `f64`，以兼顾性能和溢出安全性。

### 设计理由

该平台在仓位计算中使用 `f64`，以在性能和精度之间取得平衡：

- 浮点运算比任意精度算术显著更快。
- 即使使用 128 位整数，原始整数乘法也可能溢出。
- 每次计算都从精确的定点值开始，避免了误差的累积。
- IEEE-754 双精度提供约 15 位十进制数字的精度。

### 已验证的精度特性

测试确认 `f64` 算术在典型交易场景下能保持精度：

- 标准金额：对于标准货币中 ≥ 0.01 的金额，不存在精度损失。
- 高精度金融工具：9 位小数的加密货币价格可保持在 1e-6 的误差容限内。
- 连续成交：100 次成交未显示漂移（手续费精度可达 1e-10）。
- 极端价格：可处理从 0.00001 到 99,999.99999 的范围而不溢出。
- 往返交易：以相同价格开仓和平仓会产生精确的盈亏（仅受手续费影响）。

有关实现细节，请参阅 `crates/model/src/position.rs` 中的
`test_position_pnl_precision_*` 测试。

:::note
对于需要精确十进制算术的合规或审计追踪场景，可以考虑使用外部库中的
`Decimal` 类型。低于 `f64` epsilon（约 1e-15）的极小金额可能会四舍五入为零。
这对具有标准货币精度（通常为 2-9 位小数）的现实交易场景没有影响。
:::

## 与其他组件的集成

仓位与几个关键组件交互：

- **Portfolio**：跨金融工具和策略聚合仓位。
- **ExecutionEngine**：根据成交创建和更新仓位。
- **Cache**：存储仓位状态和快照。
- **RiskEngine**：监控仓位限额和敞口。

:::note
不会为价差（spread）金融工具创建仓位。虽然或有订单（contingent orders）
仍然可以为价差触发，但它们的运行不与仓位关联。引擎将价差金融工具
与常规仓位分开处理。
:::

## 小结

仓位是跟踪交易活动和绩效的核心。理解仓位如何聚合成交、
计算盈亏，以及处理不同的 OMS 配置，对于构建交易策略至关重要。
仓位快照在 `NETTING` 模式下提供了准确的历史跟踪，而事件历史
支持详细的分析和核对。

## 相关指南

- [事件（Events）](events/) - 成交如何产生仓位事件。
- [订单（Orders）](orders/) - 创建和修改仓位的订单。
- [执行（Execution）](execution.md) - 更新仓位的成交处理。
- [投资组合（Portfolio）](portfolio.md) - 投资组合层面的仓位聚合。
