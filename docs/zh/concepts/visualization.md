# 可视化（Visualization）

NautilusTrader 通过一个基于 Plotly 构建的可扩展可视化系统，提供交互式
HTML 分析报告（tearsheet），用于分析回测结果。你可以用最少的代码生成报告，
并添加自定义图表和主题。

## 概览

可视化系统由三部分组成：

1. **图表注册表（Chart Registry）** - 解耦的图表定义，可以用自定义可视化进行扩展。
2. **主题系统（Theme System）** - 内置及自定义主题带来的一致样式。
3. **配置（Configuration）** - 用于声明式指定渲染内容及展示方式。

所有可视化输出都是自包含的 HTML 文件，可以在任何现代浏览器中查看、
与相关人员共享，或存档以供日后参考。

:::note
该可视化系统需要 `visualization` 附加组件。它会安装用于 DataFrame
处理的 Pandas、用于交互式图形的 Plotly，以及用于静态图像导出的 Kaleido：

```bash
uv pip install "nautilus_trader[visualization]"
```

:::

## 分析报告（Tearsheets）

分析报告（tearsheet）是一份将多个图表和统计数据合并到单一交互式可视化
视图中的绩效报告。分析报告在完成一次回测运行之后生成，
对策略表现提供即时的可视化反馈。

### 快速开始

使用默认设置生成一份分析报告：

```python
from nautilus_trader.analysis import create_tearsheet
from nautilus_trader.backtest.engine import BacktestEngine

# After running your backtest
engine.run()

# Generate tearsheet
create_tearsheet(
    engine=engine,
    output_path="backtest_results.html",
)
```

这会生成一个包含所有默认图表的 HTML 文件，使用亮色主题和自动布局。
在浏览器中打开 `backtest_results.html`，即可查看交互式分析报告。

### 自定义

控制显示哪些图表以及它们的样式：

```python
from nautilus_trader.analysis import TearsheetConfig
from nautilus_trader.analysis import TearsheetDrawdownChart
from nautilus_trader.analysis import TearsheetEquityChart
from nautilus_trader.analysis import TearsheetRunInfoChart
from nautilus_trader.analysis import TearsheetStatsTableChart

config = TearsheetConfig(
    charts=[
        TearsheetRunInfoChart(),
        TearsheetStatsTableChart(),
        TearsheetEquityChart(),
        TearsheetDrawdownChart(),
    ],
    theme="nautilus_dark",
    height=2000,
)

create_tearsheet(
    engine=engine,
    output_path="custom_tearsheet.html",
    config=config,
)
```

### 货币过滤

对于多币种回测，可以将统计数据过滤到某种特定货币：

```python
from nautilus_trader.model.currencies import USD

create_tearsheet(
    engine=engine,
    output_path="usd_only.html",
    currency=USD,  # Currency object, shows only USD statistics
)
```

当 `currency` 为 `None`（默认值）时，所有货币的统计数据会在分析报告中
分别展示。基于收益率的图表只有在各账户共享同一货币时，才会从账户报告中
重建；对于多币种回测，请传入 `currency`，以便收益率图表使用所选定的货币。

## 可用图表

分析报告可以包含以下任意组合的内置图表：

| 图表名称         | 类型         | 说明                                              |
|--------------------|--------------|----------------------------------------------------------|
| `run_info`         | 表格        | 运行元数据和账户余额。                       |
| `stats_table`      | 表格        | 绩效统计（盈亏、收益率、一般性指标）。  |
| `equity`           | 折线图         | 随时间变化的累计收益率，可选基准比较。    |
| `drawdown`         | 面积图         | 相对于峰值权益的回撤百分比。                    |
| `monthly_returns`  | 热力图      | 按年组织的月度投资组合收益率百分比。  |
| `distribution`     | 直方图    | 各个收益率值的分布。                |
| `rolling_sharpe`   | 折线图         | 60 天滚动夏普比率。                             |
| `yearly_returns`   | 柱状图          | 年度收益率百分比。                             |
| `bars_with_fills`  | 蜡烛图  | K 线价格（OHLC），叠加显示订单成交标记。  |

所有图表都在图表注册表中注册，并通过 `TearsheetConfig.charts` 中的图表
对象进行配置（每个图表对象对应一个内置图表名称）。

### 运行信息表

`run_info` 图表显示了关于该次回测运行的关键元数据：

- 运行 ID、开始时间、结束时间
- 回测时间段（起止日期）
- 处理的总迭代次数
- 事件、订单和仓位计数
- 账户起始和结束余额（按币种）

默认情况下，该表格出现在左上角位置。

### 绩效统计表

`stats_table` 图表以以下几个部分展示绩效指标：

- **盈亏统计**（按币种）：总盈亏、胜率、盈利因子等。
- **收益率统计**：夏普比率、索提诺比率、最大回撤等。
- **一般性统计**：总交易次数、平均交易持续时间等。

默认情况下，该表格出现在右上角位置。

### 权益曲线

`equity` 图表绘制了回测期间的累计收益率。当向 `create_tearsheet()`
提供了 `benchmark_returns` 时，基准会被叠加显示以便比较。

```python
import pandas as pd

# Load benchmark returns (e.g., from a market index)
# Index should be datetime, aligned with strategy returns timeframe
benchmark_returns = pd.read_csv("sp500_returns.csv", index_col=0, parse_dates=True)["return"]

create_tearsheet(
    engine=engine,
    output_path="with_benchmark.html",
    benchmark_returns=benchmark_returns,
    benchmark_name="S&P 500",
)
```

基准序列会按原样绘制；请确保索引与你策略的收益率日期对齐，
以获得准确的比较结果。

### 月度和年度收益率

`monthly_returns` 和 `yearly_returns` 图表默认使用复利（时间加权）收益率：
每个单元格衡量的是该周期的收益相对于周期开始时滚动余额的比例，
各周期复利累加为总收益率。

设置 `compounding=False` 可报告相对于固定初始资本计算的简单、
非复利收益率。此时每个单元格衡量的是该周期收益占起始资本的百分比，
因此各周期是相加而不是复利累加为总收益率。这是名义收益率，
适用于以固定规模交易并提取利润的固定资本策略约定。

```python
config = TearsheetConfig(
    charts=[
        TearsheetMonthlyReturnsChart(compounding=False),
        TearsheetYearlyReturnsChart(compounding=False),
    ],
)
create_tearsheet(engine=engine, config=config)
```

独立的 `create_monthly_returns_heatmap()` 和 `create_yearly_returns()`
函数接受相同的 `compounding` 参数。为了让非复利图表真实反映固定资本，
应以固定数量而不是当前权益的一部分来确定仓位规模；否则随着滚动余额
的增长，之后的周期会被夸大。

## 主题

主题控制图表的视觉样式，包括颜色、字体和背景。
NautilusTrader 提供了四种内置主题：

| 主题名称      | 说明                                    | 使用场景                       |
|-----------------|------------------------------------------------|--------------------------------|
| `plotly_white`  | 简洁的亮色主题，配深灰色表头。      | 默认，适用于专业报告。 |
| `plotly_dark`   | 深色背景，配标准 Plotly 颜色。   | 弱光环境。        |
| `nautilus`      | 使用 NautilusTrader 品牌配色的亮色主题。  | 官方亮色模式。           |
| `nautilus_dark` | 使用青绿/青色标志性配色的深色主题。    | 官方深色模式。           |

### 选择主题

在 `TearsheetConfig` 中指定主题：

```python
config = TearsheetConfig(theme="nautilus_dark")
create_tearsheet(engine=engine, config=config)
```

### 自定义主题

注册一个自定义主题，以在所有可视化中保持一致的品牌风格：

```python
from nautilus_trader.analysis import register_theme

register_theme(
    name="corporate",
    template="plotly_white",  # Base Plotly template
    colors={
        "primary": "#003366",      # Navy blue
        "positive": "#2e8b57",     # Sea green
        "negative": "#c41e3a",     # Cardinal red
        "neutral": "#808080",      # Gray
        "background": "#ffffff",   # White
        "grid": "#e5e5e5",         # Light gray
        # Optional table colors (defaults will be provided if omitted)
        "table_section": "#e5e5e5",
        "table_row_odd": "#f8f8f8",
        "table_row_even": "#ffffff",
        "table_text": "#000000",
    }
)

# Use the custom theme
config = TearsheetConfig(theme="corporate")
```

主题系统会根据 `background` 和 `grid` 颜色，自动为 `table_*` 颜色提供
合理的默认值，从而确保与在引入表格专属颜色之前注册的主题向后兼容。

## 配置

`TearsheetConfig` 类为分析报告的生成提供了声明式的控制：

```python
from nautilus_trader.analysis import GridLayout
from nautilus_trader.analysis import TearsheetConfig
from nautilus_trader.analysis import TearsheetDrawdownChart
from nautilus_trader.analysis import TearsheetEquityChart
from nautilus_trader.analysis import TearsheetStatsTableChart

config = TearsheetConfig(
    charts=[
        TearsheetEquityChart(),
        TearsheetDrawdownChart(),
        TearsheetStatsTableChart(),
    ],
    theme="nautilus_dark",
    title="Q4 2024 Strategy Performance",
    height=1800,
    include_benchmark=True,
    benchmark_name="SPY",
    layout=GridLayout(
        rows=2,
        cols=2,
        heights=[0.60, 0.40],
        vertical_spacing=0.08,
        horizontal_spacing=0.12,
    ),
)
```

### 配置参数

| 参数           | 类型                   | 默认值          | 说明                         |
|---------------------|------------------------|------------------|--------------------------------------|
| `charts`            | `list[TearsheetChart]` | 内置图表        | 要包含的图表，按顺序排列。        |
| `theme`             | `str`                  | `"plotly_white"` | 用于样式的主题名称。             |
| `layout`            | `GridLayout`           | `None`           | 自定义子图网格布局。         |
| `title`             | `str`                  | 自动生成   | 分析报告的标题。                    |
| `include_benchmark` | `bool`                 | `True`           | 在提供了基准数据时是否显示基准。       |
| `benchmark_name`    | `str`                  | `"Benchmark"`    | 基准的显示名称。         |
| `height`            | `int`                  | `1500`           | 以像素为单位的总高度。             |
| `show_logo`         | `bool`                 | `True`           | 为未来的 logo 渲染保留。 |

当 `layout` 为 `None` 时，网格尺寸和每行高度会根据图表数量自动计算。
对于 8 个图表（默认值），使用 4x2 网格，各行高度为
`[0.50, 0.22, 0.16, 0.12]`，为顶部一行的表格提供更多空间。

## 自定义图表

注册表模式允许你添加自定义图表。图表是将轨迹渲染到 Plotly figure 对象上
的函数。

### 注册一个自定义图表

```python
from nautilus_trader.analysis.tearsheet import register_chart
import plotly.graph_objects as go

def my_custom_chart(returns, output_path=None, title="Custom Chart", theme="plotly_white"):
    """
    Create a custom visualization.

    This function signature matches the built-in chart functions for consistency.
    """
    from nautilus_trader.analysis.themes import get_theme

    theme_config = get_theme(theme)

    # Create your visualization
    fig = go.Figure()
    fig.add_trace(go.Scatter(
        x=returns.index,
        y=returns.cumsum(),
        mode="lines",
        name="Custom Metric",
        line={"color": theme_config["colors"]["primary"]},
    ))

    fig.update_layout(
        title=title,
        template=theme_config["template"],
        xaxis_title="Date",
        yaxis_title="Value",
    )

    if output_path:
        fig.write_html(output_path)

    return fig

# Register the chart for standalone use (via `get_chart()` / `list_charts()`)
register_chart("my_custom", my_custom_chart)
```

### 分析报告集成

如需将图表集成到分析报告并正确进行网格布局，请使用
`register_tearsheet_chart`。与 `register_chart`（注册一个返回自己 figure 的
独立函数）不同，分析报告渲染器直接将轨迹绘制到一个共享子图网格的
单元格上，因此其函数签名接受目标 `fig` 以及要渲染到的 `row` 和 `col`。

```python
from nautilus_trader.analysis import TearsheetConfig
from nautilus_trader.analysis import TearsheetCustomChart
from nautilus_trader.analysis import TearsheetEquityChart
from nautilus_trader.analysis import TearsheetStatsTableChart
from nautilus_trader.analysis import register_tearsheet_chart

def _render_my_metric(fig, row, col, returns, theme_config, **kwargs):
    """
    Render custom metric directly onto a subplot.

    Parameters
    ----------
    fig : go.Figure
        The figure to add traces to.
    row : int
        Subplot row position.
    col : int
        Subplot column position.
    returns : pd.Series
        Strategy returns series supplied to the renderer.
    theme_config : dict
        Theme configuration dictionary.
    **kwargs : dict
        Additional parameters (stats_pnls, stats_returns, benchmark_returns, etc.).
    """
    metric_values = returns.rolling(30).std() * 100  # Example metric

    fig.add_trace(
        go.Scatter(
            x=returns.index,
            y=metric_values,
            mode="lines",
            name="30-Day Volatility",
            line={"color": theme_config["colors"]["neutral"]},
        ),
        row=row,
        col=col,
    )

    fig.update_xaxes(title_text="Date", row=row, col=col)
    fig.update_yaxes(title_text="Volatility (%)", row=row, col=col)

# Register for tearsheet use
register_tearsheet_chart(
    name="volatility",
    subplot_type="scatter",
    title="Rolling Volatility (30-day)",
    renderer=_render_my_metric,
)

# Now "volatility" can be used in TearsheetConfig.charts:
config = TearsheetConfig(
    charts=[
        TearsheetStatsTableChart(),
        TearsheetEquityChart(),
        TearsheetCustomChart(chart="volatility"),
    ],
)
```

该渲染函数会接收所有必要的数据（收益率、统计数据、主题配置），
并直接渲染到指定的子图位置上。

## 离线分析

对于你已经拥有预先计算好的统计数据、但没有 `BacktestEngine` 实例的情况，
可以使用更底层的 API：

```python
import pandas as pd

from nautilus_trader.analysis.tearsheet import create_tearsheet_from_stats

# Load precomputed data. The structure matches BacktestResult stats fields.
stats_pnls = {"USD": {"PnL (total)": 1500.0, "Win Rate": 0.55, ...}}  # Per-currency
stats_returns = {"Sharpe Ratio (252 days)": 1.2, "Max Drawdown": -0.15, ...}
stats_general = {"Avg Winner": 100.0, "Avg Loser": -50.0, ...}
returns = pd.Series(...)  # Daily returns with datetime index

create_tearsheet_from_stats(
    stats_pnls=stats_pnls,
    stats_returns=stats_returns,
    stats_general=stats_general,
    returns=returns,
    output_path="offline_analysis.html",
)
```

字典的键应与 `engine.get_result().stats_pnls`、
`engine.get_result().stats_returns` 和 `engine.get_result().stats_general`
返回的键一致。

这种方式适用于以下场景：

- 分析分别存储的多次回测运行结果。
- 使用预先计算好的指标比较不同策略。
- 与外部分析流水线集成。

## 最佳实践

### 图表选择

- 在探索性分析中使用默认图表，以查看所有可用指标。
- 当你已经明确知道哪些指标对你的策略重要时，再自定义图表。
- 移除不相关的图表，以减少视觉杂乱和文件大小。

### 主题使用

- 对于专业报告和演示，使用 `plotly_white`。
- 对于官方材料或弱光环境查看，使用 `nautilus_dark`。
- 根据内部规范或个人偏好创建自定义主题。

### 性能考量

- 分析报告的 HTML 文件将所有数据内联存储，对于长期回测可能达到数兆字节。
- 可以考虑为不同的分析时间范围分别生成独立的分析报告。
- 对于非常大的数据集，请使用单个图表函数，而不是完整的分析报告。

### 自定义统计指标集成

当自定义图表与通过内置分析报告图表所使用的相同 `stats_pnls`、
`stats_returns` 和 `stats_general` 字典一起提供的统计数据配合使用时，
效果最好。对于实时的 `BacktestEngine` 使用场景，这些值来自
`engine.get_result()`；对于离线分析，可以将兼容的字典直接传给
`create_tearsheet_from_stats()`：

```python
stats_returns = {
    "Sharpe Ratio (252 days)": 1.2,
    "Custom Volatility Score": 0.42,
}
```

## API 级别

可视化系统提供了两个 API 级别：

### 高级 API

推荐用于大多数使用场景：

```python
create_tearsheet(engine=engine, config=config)
```

自动从 `BacktestEngine` 中提取数据，生成所有已配置的图表，
并生成一份完整的 HTML 分析报告。

### 低级 API

用于高级自定义或离线分析：

```python
create_tearsheet_from_stats(
    stats_pnls=stats_pnls,
    stats_returns=stats_returns,
    stats_general=stats_general,
    returns=returns,
    run_info=run_info,
    account_info=account_info,
    config=config,
)
```

对数据输入提供细粒度的控制，并支持对预先计算好的统计数据进行分析。

### 独立的图表函数

各个图表函数可以独立使用，用于生成单一用途的 HTML 可视化，
或用于自定义分析工作流的 Plotly figure。

#### 带成交标记的价格 K 线

`create_bars_with_fills` 函数生成一个叠加了订单成交的蜡烛图，
便于在价格走势中直观分析策略的执行情况。它既可以独立使用，
也可以纳入分析报告中：

```python
from nautilus_trader.analysis import create_bars_with_fills
from nautilus_trader.analysis import create_tearsheet
from nautilus_trader.analysis import TearsheetBarsWithFillsChart
from nautilus_trader.analysis import TearsheetConfig
from nautilus_trader.analysis import TearsheetEquityChart
from nautilus_trader.analysis import TearsheetStatsTableChart
from nautilus_trader.model.data import BarType

# Standalone usage
bar_type = BarType.from_str("ESM4.XCME-1-MINUTE-LAST-EXTERNAL")
fig = create_bars_with_fills(
    engine=engine,
    bar_type=bar_type,
    title="ES Futures - Entry/Exit Analysis",
)
fig.show()  # Display in Jupyter
fig.write_html("bars_with_fills.html")  # Or save to file

# Include in tearsheet
config = TearsheetConfig(
    charts=[
        TearsheetStatsTableChart(),
        TearsheetEquityChart(),
        TearsheetBarsWithFillsChart(
            bar_type="ESM4.XCME-1-MINUTE-LAST-EXTERNAL",
            title="Bars with Fills",
        ),
    ],
)
create_tearsheet(engine=engine, config=config)

# Multiple bars-with-fills charts in one tearsheet
config = TearsheetConfig(
    charts=[
        TearsheetStatsTableChart(),
        TearsheetEquityChart(),
        TearsheetBarsWithFillsChart(
            bar_type=f"{instrument.id}-5-MINUTE-MID-INTERNAL",
            title=f"Bars with Order Fills - {instrument.id}",
        ),
        TearsheetBarsWithFillsChart(
            bar_type=f"{other_instrument.id}-5-MINUTE-MID-INTERNAL",
            title=f"Bars with Order Fills - {other_instrument.id}",
        ),
    ],
)
create_tearsheet(engine=engine, config=config)
```

该可视化展示了 OHLC 价格走势的蜡烛图，并用三角形标记表示订单成交
（绿色向上三角形代表买入，红色向下三角形代表卖出）。需要额外配置
（例如 `bar_type`）的图表会直接在图表对象上接受这些参数
（例如 `TearsheetBarsWithFillsChart(bar_type=...)`）。

其他单独的图表函数包括 `create_equity_curve`、`create_drawdown_chart`、
`create_monthly_returns_heatmap` 等。完整列表请参阅 API 参考文档。

## 相关指南

- [回测（Backtesting）](backtesting/) - 了解如何运行会生成分析报告的回测。
- [报告（Reports）](reports.md) - 理解分析报告中所展示统计数据的底层原理。
- [投资组合（Portfolio）](portfolio.md) - 了解投资组合跟踪和绩效指标。
