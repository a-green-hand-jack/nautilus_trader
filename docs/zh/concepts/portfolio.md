# 投资组合（Portfolio）

Portfolio 是管理和跟踪交易节点或回测中所有活跃策略持仓的中央枢纽。
它汇总来自多个金融工具的仓位数据，提供你持仓、风险敞口和整体绩效的
统一视图。

## 货币转换

Portfolio 支持对盈亏和敞口计算进行自动货币转换，
使你能够以自己偏好的货币查看结果。这在跨多个具有不同成本货币的
金融工具进行交易，或管理具有不同基础货币的多个账户时特别有用。

### 支持的转换

以下投资组合查询支持货币转换：

- `realized_pnl()` / `realized_pnls()` - 将已实现盈亏转换为目标货币。
- `unrealized_pnl()` / `unrealized_pnls()` - 将未实现盈亏转换为目标货币。
- `total_pnl()` / `total_pnls()` - 将总盈亏转换为目标货币。
- `net_exposure()` / `net_exposures()` - 将净敞口转换为目标货币。

所有方法都接受一个可选的 `target_currency` 参数，用于指定期望的输出
货币。

### 单账户行为

在查询单个账户且未指定 `target_currency` 时，Portfolio 会
自动将值转换为该账户的基础货币：

```python
# Returns exposure in the account's base currency (e.g., USD)
exposure = portfolio.net_exposures(venue=BINANCE, account_id=account_id)
```

### 多账户行为

在同时查询多个账户时，行为取决于你查询的是所有金融工具
（`net_exposures()`）还是单个金融工具（`net_exposure()`）：

**对于 `net_exposures()`（所有金融工具）：**

- **相同的基础货币**：自动转换为共同的基础货币。
- **不同的基础货币**：返回一个包含多种货币的字典，每种货币都
  转换为对应账户的基础货币。如需单一货币的结果，请提供 `target_currency`。

**对于 `net_exposure()`（跨账户的单个金融工具）：**

- **不同的基础货币**：除非你提供了 `target_currency`，否则返回 `None`。

```python
# Scenario 1: Multiple accounts, all with USD base currency
exposures = portfolio.net_exposures(venue=BINANCE)
# Returns {USD: Money(...)}

# Scenario 2: Multiple accounts with different base currencies (USD and EUR)
exposures = portfolio.net_exposures(venue=BINANCE)
# Returns {USD: Money(...), EUR: Money(...)}

# Force single currency across accounts
exposures = portfolio.net_exposures(venue=BINANCE, target_currency=USD)
# Returns {USD: Money(...)}
```

### 转换失败

当提供了 `target_currency` 但货币转换失败时，行为取决于
方法类型：

- **返回单一值的方法**（`realized_pnl`、`unrealized_pnl`、`total_pnl`、`net_exposure`）：
  返回 `None` 并记录一条错误日志，以防止得到错误的数值。
- **返回字典的方法**（`realized_pnls`、`unrealized_pnls`、`total_pnls`、`net_exposures`）：
  省略转换失败的金融工具，但返回转换成功的结果。

:::warning
在使用 `target_currency` 进行跨货币聚合时，必须有可用的汇率数据。
:::

### 转换使用的价格类型

在将敞口转换为目标货币时，Portfolio 会根据仓位构成
使用不同的价格类型：

- **全部为多头仓位**：使用 `BID` 价格（对多头敞口而言更保守）。
- **全部为空头仓位**：使用 `ASK` 价格（对空头敞口而言更保守）。
- **多空混合仓位**：使用 `MID` 价格（多空同时存在时更中性）。

这确保了转换能够反映现实的市场条件：你会以买价平掉多头仓位，
以卖价回补空头仓位。对于多空混合的仓位，中间价定价提供了一种
中性的估值。

如果在投资组合配置中启用了 `use_mark_xrates`，`MARK` 价格将替代
`MID` 价格，用于混合仓位和一般转换。

## 权益与按市值计价（Mark-to-market）

Portfolio 提供了用于持续投资组合估值和已记录快照的拉取式（pull-style）
查询。按币种划分的结果使用相应账户的基础货币或原生成本货币。

| 方法                             | 返回内容                                                |
|------------------------------------|--------------------------------------------------------|
| `mark_values(venue, account_id)`   | 未平仓仓位的带符号 MTM（按市值计价）总额。                  |
| `equity(venue, account_id)`        | 结合余额和仓位估值的总权益。 |
| `build_snapshot(account_id)`       | 账户级别的 MTM 总额和估值元数据。        |
| `snapshots(account_id)`            | 按发出顺序记录的账户快照。                |
| `missing_price_instruments(venue)` | 当前被标记为无法估值的金融工具。          |

多头贡献正的名义敞口，空头贡献负的名义敞口。持平（flat）
仓位会被跳过。

### 权益公式

权益结合了账户余额和未平仓仓位的估值，第二项的计算方式
因账户类型而异：

- **没有基础货币的现金账户**：以 `balances_total` 为起点。对于
  该账户持有的仓位，如果余额中已经持有该资产，且该金融工具的成本货币
  与其基础货币不同，则不再额外添加该基础资产的按市值计价数值。对于反向合约
  以及没有被某个已入账余额资产所代表的仓位，会添加其按市值计价数值。
- **有基础货币的现金账户和投注型账户**：
  `balances_total + Σ mark_value(未平仓仓位)`。
- **保证金账户**：`balances_total + Σ unrealized_pnl(未平仓仓位)`。

`mark_values()` 始终返回未平仓仓位的毛值，包括已经存在于
多币种现金余额中的资产。“只计一次”规则意味着 `equity()` 和权益
快照对每个非反向的基础资产要么计为余额、要么计为按市值计价数值，
二者不会重复计算。保证金路径使用的是驱动 `unrealized_pnls()` 的
同一个缓存未实现盈亏管道。

### 价格回退

估值会按以下顺序向 `Cache` 请求价格，一旦命中即停止：

1. 标记价格（Mark price），前提是 `PortfolioConfig` 中的 `use_mark_prices=true`（v2 默认值）且已缓存标记价格。
2. 与仓位方向相符的报价：多头用 `BID`，空头用 `ASK`。
3. 最新成交价格。
4. 最近缓存的 K 线收盘价（在 `bar_updates=true` 时填充）。

在 v2 中，设置 `use_mark_prices=false` 可跳过标记价格这一档，直接从与仓位方向相符的报价开始。

如果上述四种都无法得到当前价格，Portfolio 会沿用该金融工具和仓位方向
最后一个有效的价格。下一次快照会将该金融工具列入
`stale_instruments`。如果该仓位从未有过有效价格，它会被归入
缺失价格跟踪器，列入 `unpriced_instruments`，并从汇总中排除。

### 基础货币转换

当 `convert_to_account_base_currency=true`（默认值）且账户设置了
`base_currency` 时，成本货币的值会使用 `Cache.get_xrate()` 提供的
`MID` 汇率转换为基础货币。当 `use_mark_xrates=true` 时，会优先使用
`Cache.get_mark_xrate()` 提供的缓存标记汇率，若不可用则回退到 `MID`。
此时输出字典只有一个与基础货币对应的键。

当 `convert_to_account_base_currency=false`，或账户没有 `base_currency`
时，结果将以每个仓位的原生成本货币为键，且不会应用任何
汇率转换。

如果所需转换缺少当前汇率，Portfolio 会沿用最后一个有效的
汇率，并将其来源货币列入 `stale_currencies`。如果从未获得过有效
汇率，该仓位会被视为无法估值，并通过缺失价格跟踪器标记出来，
而不是被静默地以 1.0 的汇率估值。

### 快照估值元数据

`PortfolioSnapshot.total_equity` 始终提供按币种划分的 MTM 明细。
当启用了基础货币转换且账户设置了基础货币时，
`base_currency_equity` 会以该货币提供总权益标量值。当转换被
禁用，或账户没有基础货币时，该值为 `None`。

当快照使用了沿用的价格或汇率，或排除了从未拥有全部
所需估值输入的仓位时，`is_stale` 为 true。相关字段标明了
原因：

- `stale_instruments`：使用沿用价格估值的金融工具。
- `stale_currencies`：使用沿用汇率转换的来源货币。
- `unpriced_instruments`：由于从未获得完整有效估值而被排除的金融工具。

调用 `build_snapshot(account_id)` 可按需获取一份样本。调用
`snapshots(account_id)` 可读取有界的已记录序列。这些方法
在 Rust 的 Portfolio 和 Strategy API 中可用，也在 Python v2 版本的
Portfolio 绑定中可用。

### 缺失价格跟踪

该跟踪器为每个按账户过滤的查询范围以及未过滤的交易场所范围
维护最新的缺失集合。`missing_price_instruments(venue)` 返回它们
在整个交易场所范围内的并集。每次观察结果在同一范围再次运行之前
都是权威的；一个经过过滤的结果不会宣称此前一个未过滤的结果
已被解决。它有两种可观察的行为：

- 当某个金融工具从“没有任何范围报告它缺失”过渡到“至少一个范围报告
  它缺失”时，会触发一次警告日志，而不是在此后每次调用时都触发。
  一旦每个报告范围都观察到恢复，未来再次出现缺失会重新触发警告。
- 当某个交易场所变为持平（没有未平仓仓位）时，其跟踪器条目会被
  清除，以免过期的金融工具持续被标记。

调用 `missing_price_instruments(venue)` 可查看当前的集合。

:::tip
如果 `equity()` 的结果低于你的预期，请在深入排查计算逻辑之前先检查
`missing_price_instruments(venue)`。某个金融工具的报价、成交和 K 线
数据流为空，是造成静默缺口最常见的原因。
:::

### 交易场所和账户范围

`mark_values` 和 `equity` 接受一个可选的 `account_id`，用于将聚合
范围限定在单个账户内。当 `account_id=None` 时，结果会跨该交易场所上的
每个账户进行聚合。

按账户过滤的估值只核对该账户自身的观测结果，因此同一交易场所上
其他账户产生的标记不受影响。

## 投资组合统计

`crates/analysis/src/statistics` 中提供了多种内置的投资组合统计指标，
用于分析交易组合在回测和实盘交易中的表现。

这些统计指标大致分为以下几类：

- 基于盈亏的统计指标（按币种）
- 基于收益率的统计指标
- 基于仓位的统计指标
- 基于订单的统计指标

回测统计信息在一次运行结束后通过 `engine.get_result()` 暴露出来。

## 自定义统计指标

用于事后分析的自定义指标可以从报告、快照或仓位数据中计算，
并添加到传递给诸如 `create_tearsheet_from_stats()` 之类的可视化 API 的
字典中。

例如，根据已实现盈亏计算胜率：

```python
import pandas as pd


def calculate_win_rate(realized_pnls: pd.Series) -> float:
    if realized_pnls.empty:
        return 0.0

    winners = realized_pnls[realized_pnls > 0.0]
    return len(winners) / len(realized_pnls)
```

然后将该指标包含进离线分析报告（tearsheet）的输入中：

```python
stats_general = {
    "Win Rate": calculate_win_rate(realized_pnls),
}
```

:::tip
你的指标函数应能处理退化输入，例如空序列或数据不足的情况。
对于未知或无法计算的值返回 `None`，或在语义上合理时返回一个
合理的默认值，例如 `0.0`。
:::

## 收益率：仓位 vs 投资组合

分析器（analyzer）跟踪两个不同的收益率序列：

- **仓位收益率**（`analyzer.position_returns()`）衡量每个仓位的已实现收益率，
  这是一种相对于平均开仓价格、考虑方向的价格收益率。它反映了
  该金融工具在开仓和平仓之间的价格变动情况，与账户规模或
  杠杆无关。
- **投资组合收益率**（`analyzer.portfolio_returns()`）衡量账户总余额的
  每日百分比变化。在一个 10 万美元的账户上盈利 900 美元，
  当天的收益率大约会报告为 0.9%。

当分析器拥有跨越至少两个不同日历日的账户状态历史时，
它会自动计算投资组合收益率，并将其作为统计、分析报告
以及月度收益率热力图的主要序列。同一天内的多个快照
只算作一天，因此仅在日内交易不会产生投资组合收益率。当
投资组合收益率不可用时，会回退到仓位收益率。

便捷访问器 `analyzer.returns()` 解析这一优先级：如果存在
投资组合收益率则使用它，否则使用仓位收益率。

### 多币种账户

投资组合收益率需要单一币种的余额历史。当账户持有
多种货币的余额时，分析器无法生成单一的收益率序列，
会静默回退到仓位收益率。统计和分析报告图表会使用
`returns()` 所解析出的那个序列。

如果你需要多币种账户的投资组合级别收益率，请在计算百分比变化
之前，先在外部将余额转换为一种统一货币来计算。

### 按交易场所计算

在回测引擎中，分析器按交易场所运行（`engine.pyx`）。每个交易场所的
账户会产生自己的投资组合收益率序列。分析报告会跨所有已缓存的
账户进行聚合，为多交易场所回测生成一个综合的收益率序列。

## 回测分析

在一次回测运行结束后，引擎会将已实现盈亏、收益率、仓位和订单数据
传递给每个已注册的统计指标。任何输出都会显示在分析报告中
`Portfolio Performance` 标题下，分组如下：

- 已实现盈亏统计（按币种）
- 收益率统计（针对整个投资组合）
- 从仓位和订单数据中得出的一般性统计（针对整个投资组合）

## 相关指南

- [仓位（Positions）](positions.md) - 投资组合内的仓位跟踪。
- [报告（Reports）](reports.md) - 生成投资组合分析报告。
- [可视化（Visualization）](visualization.md) - 可视化投资组合表现。
