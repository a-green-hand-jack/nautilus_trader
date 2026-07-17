# 报告（Reports）

本指南解释 `ReportProvider` 类提供的投资组合分析和报告功能，
以及这些报告如何用于盈亏核算和回测后分析。

## 概览

NautilusTrader 中的 `ReportProvider` 类从交易数据生成结构化的分析报告，
将原始的订单、成交、仓位和账户状态转换为 pandas DataFrame，
以便进行分析和可视化。这些报告帮助你评估策略表现、
分析执行质量，并核实盈亏核算。

报告可以通过两种方式生成：

- **Trader 辅助方法**（推荐）：便捷的方法，例如 `trader.generate_orders_report()`。
- **直接使用 ReportProvider**：以便更精细地控制数据选择和过滤。

报告在回测和实盘交易环境中提供一致的分析结果，
从而实现可靠的绩效评估和策略比较。

## 可用的报告

`ReportProvider` 类提供了几种静态方法，用于从交易数据生成报告。
每份报告都返回一个具有特定列和索引的 pandas DataFrame，便于分析。

### 订单报告

生成所有订单的完整视图：

```python
# Using Trader helper method (recommended)
orders_report = trader.generate_orders_report()

# Or using ReportProvider directly
from nautilus_trader.analysis import ReportProvider

orders = cache.orders()
orders_report = ReportProvider.generate_orders_report(orders)
```

**返回 `pd.DataFrame`。主要列包括：**

| 列             | 说明                                             |
|--------------------|-----------------------------------------------------|
| `client_order_id`  | 索引 - 唯一订单标识符。                        |
| `instrument_id`    | 交易的金融工具。                                     |
| `strategy_id`      | 创建该订单的策略。                                |
| `trader_id`        | 交易者标识符。                                     |
| `account_id`       | 账户标识符（如果已分配）。                       |
| `venue_order_id`   | 交易场所分配的订单 ID（如果已接受）。                  |
| `side`             | BUY 或 SELL。                                            |
| `type`             | MARKET、LIMIT 等。                                     |
| `status`           | 当前订单状态。                                   |
| `quantity`         | 原始订单数量（字符串）。                       |
| `filled_qty`       | 已成交数量（字符串）。                                 |
| `price`            | 限价（取决于订单类型）。                     |
| `avg_px`           | 平均成交价格（如果已成交）。                         |
| `time_in_force`    | 有效期指令。                              |
| `ts_init`          | 订单初始化时间戳（Unix 纳秒）。      |
| `ts_last`          | 最后更新的时间戳（Unix 纳秒）。               |

其他列因订单类型而异（例如止损单的 `trigger_price`，GTD
订单的 `expire_time`）。完整字段列表请参阅 `Order.to_dict()`。

### 订单成交汇总报告

提供已成交订单的摘要（每个订单一行）：

```python
# Using Trader helper method (recommended)
fills_report = trader.generate_order_fills_report()

# Or using ReportProvider directly
orders = cache.orders()
fills_report = ReportProvider.generate_order_fills_report(orders)
```

该报告仅包含 `filled_qty > 0` 的订单，其列与订单报告相同，
但只保留了已执行的订单。请注意，该报告中的 `ts_init` 和 `ts_last`
会转换为 datetime 对象，以便于分析。

### 成交报告

详细描述各个成交事件（每笔成交一行）：

```python
# Using Trader helper method (recommended)
fills_report = trader.generate_fills_report()

# Or using ReportProvider directly
orders = cache.orders()
fills_report = ReportProvider.generate_fills_report(orders)
```

**返回 `pd.DataFrame`。主要列包括：**

| 列             | 说明                              |
|--------------------|------------------------------------------|
| `client_order_id`  | 索引 - 订单标识符。                |
| `trade_id`         | 唯一的成交 ID。            |
| `venue_order_id`   | 交易场所分配的订单 ID。                 |
| `instrument_id`    | 交易的金融工具。                      |
| `strategy_id`      | 创建该订单的策略。         |
| `account_id`       | 账户标识符。                      |
| `position_id`      | 关联的仓位 ID（如适用）。  |
| `order_side`       | BUY 或 SELL。                             |
| `order_type`       | 订单类型（MARKET、LIMIT 等）。        |
| `last_px`          | 成交执行价格（字符串）。           |
| `last_qty`         | 成交执行数量（字符串）。           |
| `currency`         | 该笔成交的货币。                    |
| `liquidity_side`   | MAKER 或 TAKER。                          |
| `commission`       | 手续费金额和货币。          |
| `ts_event`         | 成交时间戳（datetime）。               |
| `ts_init`          | 初始化时间戳（datetime）。     |

完整字段列表请参阅 `OrderFilled.to_dict()`。

### 仓位报告

仓位分析，包括快照：

```python
# Using Trader helper method (recommended)
# Automatically includes snapshots for NETTING OMS
positions_report = trader.generate_positions_report()

# Or using ReportProvider directly
positions = cache.positions()
snapshots = cache.position_snapshots()  # For NETTING OMS
positions_report = ReportProvider.generate_positions_report(
    positions=positions,
    snapshots=snapshots
)
```

**返回 `pd.DataFrame`。主要列包括：**

| 列             | 说明                              |
|--------------------|------------------------------------------|
| `position_id`      | 索引 - 唯一仓位标识符。      |
| `instrument_id`    | 交易的金融工具。                      |
| `strategy_id`      | 管理该仓位的策略。      |
| `trader_id`        | 交易者标识符。                       |
| `account_id`       | 账户标识符。                       |
| `opening_order_id` | 开仓订单 ID。       |
| `closing_order_id` | 平仓订单 ID。       |
| `entry`            | 入场方向（BUY 或 SELL）。                |
| `side`             | 仓位方向（LONG、SHORT 或 FLAT）。    |
| `quantity`         | 当前仓位规模。                   |
| `peak_qty`         | 达到的最大规模。                     |
| `avg_px_open`      | 平均入场价格。                     |
| `avg_px_close`     | 平均出场价格（如已平仓）。          |
| `commissions`      | 已支付手续费列表。                |
| `realized_pnl`     | 已实现盈亏。                    |
| `realized_return`  | 收益率百分比。                     |
| `ts_init`          | 仓位初始化时间戳。       |
| `ts_opened`        | 开仓时间戳（datetime）。            |
| `ts_last`          | 最后更新时间戳。                   |
| `ts_closed`        | 平仓时间戳（datetime 或 NA）。      |
| `duration_ns`      | 仓位持续时间（纳秒）。        |
| `is_snapshot`      | 是否为历史快照。   |

### 账户报告

跟踪账户余额和保证金随时间的变化：

```python
# Using Trader helper method (recommended)
# Requires venue parameter
from nautilus_trader.model.identifiers import Venue
venue = Venue("BINANCE")
account_report = trader.generate_account_report(venue)

# Or using ReportProvider directly
account = cache.account(account_id)
account_report = ReportProvider.generate_account_report(account)
```

**返回 `pd.DataFrame`。列包括：**

| 列          | 说明                                |
|-----------------|--------------------------------------------|
| `ts_event`      | 索引 - 账户状态变化的时间戳。 |
| `account_id`    | 账户标识符。                        |
| `account_type`  | 账户类型（例如 SPOT、MARGIN）。      |
| `base_currency` | 该账户的基础货币。             |
| `total`         | 总余额金额（字符串）。             |
| `free`          | 可用余额（字符串）。                 |
| `locked`        | 被订单锁定的余额（字符串）。         |
| `currency`      | 该余额的货币。                    |
| `reported`      | 该余额是否由交易场所报告。     |
| `margins`       | 保证金信息（列表，如适用）。  |
| `info`          | 其他特定于交易场所的信息。     |

每一行代表一条余额记录；持有多种货币的账户，每次账户状态事件
会产生多行记录。

## 盈亏核算注意事项

准确的盈亏核算需要仔细考虑以下几个因素：

### 基于仓位的盈亏

- **已实现盈亏**：在仓位部分或全部平仓时计算。
- **未实现盈亏**：使用当前价格按市值计价。
- **手续费影响**：仅当手续费以仓位的成本货币计价时才会计入。

:::warning
盈亏计算取决于 OMS 类型。在 `NETTING` OMS 中，当仓位重新开仓时，
仓位快照会保留历史盈亏。为了准确计算总盈亏，务必在报告中包含快照。
在 `HEDGING` OMS 中，由于每个仓位都有唯一的 ID 且从不重新开仓，
不使用快照。
:::

### 多币种核算

在处理多种货币时：

- 每个仓位以其成本货币跟踪盈亏：线性合约为报价货币，反向合约为
  基础货币，数量对冲（quanto）合约为结算货币。
- 投资组合汇总需要货币转换。
- 手续费货币可能与仓位的成本货币不同。

```python
# Accessing PnL across positions
for position in positions:
    realized = position.realized_pnl  # In the position's cost currency
    unrealized = position.unrealized_pnl(last_price)

    # Handle multi-currency aggregation (illustrative)
    # Note: Currency conversion requires user-provided exchange rates
    if realized.currency != base_currency:
        # Apply conversion rate from your data source
        # rate = get_exchange_rate(realized.currency, base_currency)
        # realized_converted = realized.as_double() * rate
        pass
```

### 快照方面的考虑

对于 `NETTING` OMS：

```python
from nautilus_trader.model.objects import Money

# Include snapshots for complete PnL (per currency)
pnl_by_currency = {}

# Add PnL from current positions
for position in cache.positions(instrument_id=instrument_id):
    if position.realized_pnl:
        currency = position.realized_pnl.currency
        if currency not in pnl_by_currency:
            pnl_by_currency[currency] = 0.0
        pnl_by_currency[currency] += position.realized_pnl.as_double()

# Add PnL from historical snapshots
for snapshot in cache.position_snapshots(instrument_id=instrument_id):
    if snapshot.realized_pnl:
        currency = snapshot.realized_pnl.currency
        if currency not in pnl_by_currency:
            pnl_by_currency[currency] = 0.0
        pnl_by_currency[currency] += snapshot.realized_pnl.as_double()

# Create Money objects for each currency
total_pnls = [Money(amount, currency) for currency, amount in pnl_by_currency.items()]
```

## 回测后分析

回测完成后，可以通过结果统计和生成的报告进行分析。

### 访问回测结果

```python
# After backtest run
engine.run(start=start_time, end=end_time)

# Access result statistics
result = engine.get_result()

# Generate reports from the backtest engine
fills_report = engine.generate_fills_report()
venue = engine.list_venues()[0]
account_report = engine.generate_account_report(venue=venue)

# Or access data directly for custom analysis
orders = engine.cache.orders()
positions = engine.cache.positions()
snapshots = engine.cache.position_snapshots()
```

### 投资组合统计

回测结果提供绩效指标：

```python
# Access backtest result statistics
result = engine.get_result()

# Get different categories of statistics
stats_pnls = result.stats_pnls
stats_returns = result.stats_returns
stats_general = result.stats_general
```

:::info
有关可用统计指标的详细信息，请参阅
[投资组合指南](portfolio.md#portfolio-statistics)。该指南涵盖：

- 内置统计指标类别（基于盈亏、收益率、仓位、订单）。
- 投资组合报告的上下文。

:::

### 可视化

NautilusTrader 通过 Plotly 提供交互式分析报告（tearsheet）和图表：

```python
from nautilus_trader.analysis import create_tearsheet

# After backtest run
engine.run()

# Generate interactive HTML tearsheet
create_tearsheet(engine, output_path="tearsheet.html")
```

这会创建一份交互式 HTML 报告，包含：

- 权益曲线
- 回撤分析
- 月度收益率热力图
- 绩效统计表
- 收益率分布

如需更精细的控制，可以生成单独的图表：

```python
import pandas as pd

from nautilus_trader.analysis import create_equity_curve

returns = pd.Series(
    [0.01, -0.005, 0.002],
    index=pd.date_range("2024-01-01", periods=3, tz="UTC"),
)
fig = create_equity_curve(returns, title="My Strategy Equity")
fig.show()  # Display in browser
fig.write_image("equity.png")  # Export to PNG (requires kaleido)
```

安装可视化依赖：

```bash
uv pip install "nautilus_trader[visualization]"
```

## 报告生成模式

### 实盘交易

在实盘交易期间，定期生成报告：

```python
import pandas as pd

class ReportingActor(Actor):
    def on_start(self):
        # Schedule periodic reporting
        self.clock.set_timer(
            name="generate_reports",
            interval=pd.Timedelta(minutes=30),
            callback=self.generate_reports
        )

    def generate_reports(self, event):
        # Generate and log reports
        positions_report = self.trader.generate_positions_report()

        # Save or transmit report
        positions_report.to_csv(f"positions_{event.ts_event}.csv")
```

### 绩效分析

对于回测分析：

```python
import pandas as pd

# Run the backtest
engine.run(start=start_time, end=end_time)

# Collect results
positions_closed = engine.cache.positions_closed()
result = engine.get_result()
stats_pnls = result.stats_pnls
stats_returns = result.stats_returns
stats_general = result.stats_general

# Create summary dictionary
results = {
    "total_positions": len(positions_closed),
    "pnl_total": stats_pnls.get("USD", {}).get("PnL (total)"),
    "sharpe_ratio": stats_returns.get("Sharpe Ratio (252 days)"),
    "profit_factor": stats_general.get("Profit Factor"),
    "win_rate": stats_general.get("Win Rate"),
}

# Display results
results_df = pd.DataFrame([results])
print(results_df.T)  # Transpose for vertical display
```

:::info
报告是从内存中的数据结构生成的。对于大规模分析或长期运行的系统，
可以考虑将报告持久化到数据库中以便高效查询。有关持久化选项，
请参阅[缓存指南](cache.md)。
:::

## 与其他组件的集成

`ReportProvider` 与若干系统组件协同工作：

- **Cache**：报告所需全部交易数据（订单、仓位、账户）的来源。
- **Portfolio**：使用报告进行绩效分析和指标计算。
- **BacktestEngine**：使用报告进行回测后分析和可视化。
- **仓位快照**：在 `NETTING` OMS 中，准确的盈亏报告需要用到仓位快照。

## 小结

`ReportProvider` 将订单、成交、仓位和账户状态生成为结构化的
DataFrame，用于分析和可视化。为了在 `NETTING` OMS 中获得准确的
总盈亏，在生成报告时请包含仓位快照。

## 相关指南

- [可视化（Visualization）](visualization.md) - 来自回测结果的交互式分析报告和图表。
- [投资组合（Portfolio）](portfolio.md) - 投资组合统计和绩效指标。
- [回测（Backtesting）](backtesting/) - 运行会生成报告的回测。
- [缓存（Cache）](cache.md) - 存储报告所需数据的缓存系统。
