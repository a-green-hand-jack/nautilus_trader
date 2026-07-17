# 希腊值（Greeks）

Nautilus 提供两条处理期权希腊值（option Greeks，即期权价格对市场变量变化的
敏感度）的路径：

1. **交易场所提供的希腊值（Rust/PyO3）**：通过 `OptionGreeks` 数据类型和期权链
   聚合系统，从 Deribit、Bybit、OKX 等交易场所流式获取实时希腊值。
2. **本地希腊值计算器（Cython/Python 和 Rust/PyO3）**：`GreeksCalculator`
   类根据缓存的市场数据计算 Black-Scholes 希腊值，支持投资组合聚合、
   冲击情景（shock scenarios）和 beta 加权。

两条路径既可以独立使用，也可以组合使用。交易场所提供的希腊值通过数据订阅
系统到达，无需本地计算。本地计算器则覆盖了不推送希腊值的交易场所、回测，
以及自定义调整（冲击、beta 加权、百分比希腊值）。

## 交易场所提供的希腊值（Rust/PyO3）

### OptionGreeks

`OptionGreeks` 类型表示由交易场所提供的、针对单一期权合约的敏感度。
它是一个 Rust 原生类型，通过 PyO3 暴露给 Python。

| 字段              | 类型               | 说明                                         |
|--------------------|--------------------|-----------------------------------------------------|
| `instrument_id`    | `InstrumentId`     | 这些希腊值所属的期权合约。          |
| `convention`       | `GreeksConvention` | 希腊值的计价单位约定（Numeraire convention）。                     |
| `delta`            | `float`            | 期权价格相对于标的每单位变化的变化率。 |
| `gamma`            | `float`            | delta 相对于标的每单位变化的变化率。        |
| `vega`             | `float`            | 隐含波动率变化 1% 时的敏感度。   |
| `theta`            | `float`            | 每日时间衰减（dV/dt / 365.25）。                  |
| `rho`              | `float`            | 对利率变化的敏感度。                  |
| `mark_iv`          | `float` 或 None    | 标记隐含波动率。                            |
| `bid_iv`           | `float` 或 None    | 买价隐含波动率。                            |
| `ask_iv`           | `float` 或 None    | 卖价隐含波动率。                            |
| `underlying_price` | `float` 或 None    | 计算时使用的标的价格。            |
| `open_interest`    | `float` 或 None    | 该合约的未平仓合约量（open interest）。                     |
| `ts_event`         | `int`              | 该事件的 UNIX 时间戳（纳秒）。          |
| `ts_init`          | `int`              | 初始化时的 UNIX 时间戳（纳秒）。      |

从 actor 或策略中订阅：

```python
self.subscribe_option_greeks(instrument_id, client_id=ClientId("DERIBIT"))
```

处理更新：

```python
def on_option_greeks(self, greeks: OptionGreeks) -> None:
    self.log.info(f"delta={greeks.delta:.4f} gamma={greeks.gamma:.6f}")
```

有关完整的订阅 API（包括期权链聚合、行权价范围过滤和快照模式），
请参阅[期权（Options）](options.md)指南。

### 持久化与回放

`OptionGreeks` 是 `Data` 枚举的原生成员，因此它会持久化到数据 catalog，
并在回测中作为内置市场数据（而非自定义数据）进行回放。写入和查询
使用标准的 catalog API：

```python
catalog.write_data(greeks)               # greeks: list[OptionGreeks]
greeks = catalog.query(data_cls=OptionGreeks)
```

在回放期间，已持久化的希腊值会通过与实时数据相同的 `on_option_greeks`
处理器到达已订阅的 actor 或策略。它们也会驱动期权链聚合：当一个策略
订阅了某个 `OptionChainSlice` 时，回测数据引擎会将已回放的 `OptionGreeks`
与该期权金融工具已回放的 `QuoteTick` 顶档报价（BBO）更新进行合并。
`underlying_price` 字段用于确定 ATM，而 `delta` 支持通过
`StrikeRange.delta(target, tolerance)` 进行基于 delta 的行权价选择。

### 核心模式与自定义数据的区别

原生的 `OptionGreeks` 字段是标准的核心模式（core schema）：五个标准
希腊值（`delta`、`gamma`、`vega`、`theta`、`rho`）加上隐含波动率、标的价格、
未平仓合约量和计价单位约定。这些字段名是稳定的。

由于不存在唯一完整的希腊值形态，特定于交易场所或模型的值，例如
`vanna`、`volga`、`charm`、校准输入或波动率曲面元数据，应归入
[自定义数据](custom_data.md)，而不是原生类型。可选的交易场所字段是
可为空的。用于解释这些值所必需的字段，例如 `convention`，是不可为空的，
并携带默认值。

### 底层 Rust 类型

核心 Rust 实现位于 `crates/model/src/data/greeks.rs`：

- `OptionGreekValues`：一个包含 `delta`、`gamma`、`vega`、`theta`、`rho`
  字段的简单结构体。实现了 `Add` 和 `Mul<f64>` 用于聚合。
- `OptionGreeks`（位于 `crates/model/src/data/option_chain.rs`）：包装了
  `OptionGreekValues`，并附加 `instrument_id`、`convention`、隐含波动率字段
  以及时间戳。实现了 `Deref<Target = OptionGreekValues>`，因此你可以
  直接访问希腊值字段。
- `HasGreeks` trait：提供一个返回 `OptionGreekValues` 的 `greeks()` 方法。
  由 `OptionGreekValues` 和 `OptionGreeks` 共同实现。

### Black-Scholes 函数（Rust/PyO3）

从 `crates/model/src/data/greeks.rs` 暴露给 Python 的底层定价函数：

```python
from nautilus_trader.model import (
    black_scholes_greeks,
    imply_vol,
    imply_vol_and_greeks,
    refine_vol_and_greeks,
)

# Compute Greeks given known volatility
result = black_scholes_greeks(s=100.0, r=0.05, b=0.0, vol=0.20, is_call=True, k=100.0, t=0.25)
# result.delta, result.gamma, result.vega, result.theta, result.price, result.vol

# Imply volatility from market price, then compute Greeks
result = imply_vol_and_greeks(s=100.0, r=0.05, b=0.0, is_call=True, k=100.0, t=0.25, price=5.0)

# Refine volatility from a starting vol estimate (faster convergence)
result = refine_vol_and_greeks(s=100.0, r=0.05, b=0.0, is_call=True, k=100.0, t=0.25,
                                target_price=5.0, initial_vol=0.18)
```

这些函数返回的 `BlackScholesGreeksResult` 包含：`price`、`vol`、
`delta`、`gamma`、`vega`、`theta` 和 `itm_prob`。

**约定：**

- Vega 按 0.01 缩放（对应波动率变化 1 个百分点的敏感度）。
- Theta 按 1/365.25 缩放（每日衰减）。
- 美式期权在计算希腊值时按欧式期权定价。

## 本地希腊值计算器

### GreeksCalculator

位于 `nautilus_trader/model/greeks.pyx` 中的传统 Cython `GreeksCalculator`
类根据缓存的市场数据计算 Black-Scholes 希腊值。`nautilus_trader.common.GreeksCalculator`
还暴露了一个 PyO3 版本的计算器，用于 v2 运行时接口。
两种计算器都使用缓存和时钟，且都可以从 actor 或策略中访问。

```python
from nautilus_trader.model.greeks import GreeksCalculator  # legacy Cython

# v2 PyO3: from nautilus_trader.common import GreeksCalculator

# Typically created in on_start()
calculator = GreeksCalculator(cache=self.cache, clock=self.clock)
```

#### 单一金融工具的希腊值

以数量为 1 计算单一金融工具（期权或标的）的希腊值：

```python
greeks = calculator.instrument_greeks(
    instrument_id=option_id,
    flat_interest_rate=0.0425,  # used if no yield curve in cache
)
# Both surfaces return GreeksData or None while market data is warming up.
```

两种计算器都会：

1. 在缓存中查找该金融工具及其标的。
2. 获取当前价格（优先使用 MID，回退到 LAST）。
3. 从缓存中查找收益率曲线（缺失时回退到 `flat_interest_rate`）。
4. 使用 `imply_vol_and_greeks` 从市场价格推算隐含波动率。
5. 返回一个包含全部计算值的 `GreeksData` 对象。

价格缺失时返回 `None`，这使策略可以将预热阶段视为正常的无操作路径处理。
v2 PyO3 接口对于设置错误（例如缺失金融工具定义）仍然会引发 Python 异常。

对于非期权金融工具（期货、股票），计算器返回一个 `delta=1`（或经过 beta
加权的 delta）且没有 gamma/vega/theta 的 `GreeksData`。

**冲击情景**：对标的价格、波动率或时间施加假设性变化：

```python
greeks = calculator.instrument_greeks(
    instrument_id=option_id,
    spot_shock=10.0,            # +10 points on underlying
    vol_shock=0.02,             # +2% absolute vol increase
    time_to_expiry_shock=1/365, # roll forward one day
)
```

**波动率更新**：从一个缓存的起始点细化隐含波动率，以加快收敛速度：

```python
greeks = calculator.instrument_greeks(
    instrument_id=option_id,
    update_vol=True,        # use cached vol as starting point
    cache_greeks=True,      # store result for next iteration
)
```

**beta 加权希腊值**：以某个指数为基准表示 delta 和 gamma：

```python
greeks = calculator.instrument_greeks(
    instrument_id=option_id,
    index_instrument_id=InstrumentId.from_str("SPX.CBOE"),
    beta_weights={underlying_id: 1.15},
    percent_greeks=True,
)
```

**时间加权 vega**：在不同到期日之间归一化 vega：

```python
greeks = calculator.instrument_greeks(
    instrument_id=option_id,
    vega_time_weight_base=30,  # normalize to 30-day vega
)
```

#### 投资组合希腊值

跨所有符合过滤条件的未平仓仓位聚合希腊值：

```python
portfolio = calculator.portfolio_greeks(
    underlyings=["AAPL", "MSFT"],
    venue=Venue("CBOE"),
    strategy_id=StrategyId("DELTA_HEDGE-001"),
    flat_interest_rate=0.0425,
    index_instrument_id=InstrumentId.from_str("SPX.CBOE"),
    beta_weights=beta_dict,
    percent_greeks=True,
)
# Returns PortfolioGreeks: pnl, price, delta, gamma, vega, theta
```

过滤条件：

- `underlyings`：符号前缀列表（例如 `["AAPL"]` 匹配 AAPL 股票及所有
  AAPL 期权）。
- `venue`：限定为单个交易场所。
- `instrument_id`：限定为单个金融工具。
- `strategy_id`：限定为单个策略。
- `side`：按仓位方向过滤（LONG、SHORT）。
- `greeks_filter`：一个可调用对象，接受每个仓位的 `PortfolioGreeks`；返回
  `True` 表示包含该仓位。

### GreeksData

在传统的 Python 接口上，`GreeksData` 是一个 Python 自定义数据类
（`@customdataclass`），携带单个金融工具希腊值计算的完整上下文。它扩展了
`Data`，并支持 Arrow 序列化、缓存存储和 catalog 持久化。v2/PyO3 接口
从 Rust 暴露了相同的核心字段。

| 字段               | 类型            | 说明                                            |
|---------------------|-----------------|--------------------------------------------------------|
| `instrument_id`     | `InstrumentId`  | 该金融工具。                                        |
| `is_call`           | `bool`          | 看涨期权为 True，看跌期权为 False。                          |
| `strike`            | `float`         | 行权价。                                        |
| `expiry`            | `int`           | 到期日期，以 YYYYMMDD 整数表示。                       |
| `expiry_in_days`    | `int`           | 距到期的天数。                                          |
| `expiry_in_years`   | `float`         | 距到期的年数（天数 / 365.25）。                       |
| `multiplier`        | `float`         | 合约乘数。                                   |
| `quantity`          | `float`         | 仓位数量（从 `instrument_greeks` 得出时始终为 1）。 |
| `underlying_price`  | `float`         | 计算中使用的标的价格。                  |
| `interest_rate`     | `float`         | 使用的利率。                                   |
| `cost_of_carry`     | `float`         | 持有成本（r - 股息率；期货为 0）。     |
| `vol`               | `float`         | 隐含波动率。                                     |
| `pnl`               | `float`         | 相对于仓位入场的盈亏（如提供了仓位信息）。 |
| `price`             | `float`         | 模型价格。                                          |
| `delta`             | `float`         | Delta。                                                 |
| `gamma`             | `float`         | Gamma。                                                 |
| `vega`              | `float`         | Vega（波动率变化 1% 对应的 dV）。                             |
| `theta`             | `float`         | Theta（每日衰减）。                             |
| `itm_prob`          | `float`         | 价内（in‑the‑money）概率。                                  |

`GreeksData` 通过其 `to_portfolio_greeks()` 方法扩展到投资组合级别，
该方法将所有值乘以合约的 `multiplier`。`*` 运算符应用仓位数量：

```python
position_greeks = signed_qty * instrument_greeks  # returns PortfolioGreeks
```

### PortfolioGreeks

`PortfolioGreeks` 是 `portfolio_greeks()` 返回的聚合结果。它支持
加法（`+`）以合并仓位，也支持标量乘法（`*`）以进行缩放：

| 字段   | 类型    | 说明            |
|---------|---------|------------------------|
| `pnl`   | `float` | 汇总盈亏。         |
| `price` | `float` | 汇总模型价值。 |
| `delta` | `float` | 投资组合 delta。       |
| `gamma` | `float` | 投资组合 gamma。       |
| `vega`  | `float` | 投资组合 vega。        |
| `theta` | `float` | 投资组合 theta。       |

### YieldCurveData

`YieldCurveData` 存储一条利率或股息收益率曲线。`GreeksCalculator`
按货币代码（用于利率）或按标的金融工具 ID（用于股息收益率）
从缓存中查找曲线。

```python
from nautilus_trader.model.greeks_data import YieldCurveData
import numpy as np

curve = YieldCurveData(
    ts_event=0,
    ts_init=0,
    curve_name="USD",
    tenors=np.array([0.25, 0.5, 1.0, 2.0]),
    interest_rates=np.array([0.04, 0.042, 0.045, 0.048]),
)

# Callable: interpolates rate for a given tenor
rate = curve(0.75)  # quadratic interpolation
```

## 在两条路径之间做选择

| 判断标准                    | 交易场所提供（`OptionGreeks`）        | 本地计算器（`GreeksCalculator`）    |
|------------------------------|----------------------------------------|------------------------------------------|
| 计算                  | 由交易场所完成                      | 本地 Black‑Scholes                      |
| 延迟                      | 随市场数据一同到达               | 按需计算                       |
| 支持的交易场所                       | Deribit、Bybit、OKX                    | 任何拥有期权金融工具的交易场所        |
| 冲击情景              | 不支持                          | 支持标的、波动率和时间冲击                      |
| 投资组合聚合        | 手动（遍历 `OptionChainSlice`）    | 通过 `portfolio_greeks()` 内置支持        |
| Beta 加权               | 不支持                          | 内置支持                                 |
| 回测支持             | 通过已记录的 `OptionGreeks` 数据       | 从任意时间点的缓存价格计算  |
| 可用的希腊值             | delta、gamma、vega、theta、rho、IV、OI | delta、gamma、vega、theta、itm_prob、vol |
| 数据类型                    | `OptionGreeks`（Rust/PyO3）             | `GreeksData` / `PortfolioGreeks`         |

## 希腊值定义

作为参考，以下是 Nautilus 计算的希腊值：

| 希腊值      | 符号 | 定义                                                                    |
|------------|--------|-------------------------------------------------------------------------------|
| Delta      | `d`    | 期权价格相对于标的价格的一阶导数（dV/dS）。    |
| Gamma      | `g`    | 期权价格相对于标的价格的二阶导数（d2V/dS2）。 |
| Vega       | `v`    | 隐含波动率变化 1 个百分点时的敏感度（dV/dVol）。   |
| Theta      | `t`    | 每日时间衰减：每个日历日期权价格的变化（dV/dt / 365.25）。   |
| Rho        | `r`    | 对无风险利率变化的敏感度（dV/dr）。               |
| ITM 概率   | -      | 期权到期时处于价内的概率：P(ϕS_T > ϕK)，其中看涨期权 ϕ = 1，看跌期权 ϕ = -1。 |

## 示例

代码库中提供了完整可运行的示例：

- `examples/live/bybit/bybit_option_greeks.py`：订阅 Bybit 提供的希腊值。
- `examples/live/deribit/deribit_option_greeks.py`：订阅 Deribit 提供的希腊值。
- `examples/live/okx/okx_option_greeks.py`：订阅 OKX 提供的希腊值。

## 相关指南

- [期权（Options）](options.md) - 期权金融工具、期权链订阅和行权价过滤。
- [数据（Data）](data/) - 内置数据类型、自定义数据和订阅模型。
- [Actor](actors.md) - 订阅和处理器参考。
- [策略（Strategies）](strategies.md) - 策略实现和处理器方法。
