# 合成工具（Synthetics）

合成金融工具（Synthetic instruments）是在本地定义的金融工具，其价格由其他金融工具推导而来。
它们可以组合来自一个交易场所或多个交易场所的成分工具，并以标准的
Nautilus 金融工具形式暴露结果，使用合成交易场所代码 `SYNTH`。

合成金融工具适用于：

- 使 `Actor` 和 `Strategy` 组件能够订阅报价或成交数据流。
- 根据推导出的价格触发模拟订单（emulated orders）。
- 从合成报价或成交构建 K 线。

合成金融工具不能被直接交易。它们仅存在于平台本地，
作为分析工具使用。未来，Nautilus 可能会支持基于合成金融工具行为
交易其成分金融工具。

## 公式语言

每个合成金融工具都定义了一个推导公式。Nautilus 使用其内置的
数值表达式引擎对该公式求值，并将最终的数值结果转换为合成金融工具的
`Price`。

### 支持的语法

公式可以直接引用成分 `InstrumentId` 值，包括含有 `/` 和 `-` 的 ID。

| 结构           | 示例                                        | 说明                                                                 |
|---------------------|------------------------------------------------|-----------------------------------------------------------------------|
| 成分引用 | `BTCUSDT.BINANCE`                              | 使用原始 `InstrumentId` 文本。                                      |
| 成分引用 | `AUD/USD.SIM`                                  | 含有 `/` 的 ID 是有效的。                                         |
| 成分引用 | `ETH-USDT-SWAP.OKX`                            | 含有 `-` 的 ID 是有效的。                                         |
| 数值字面量     | `1`、`0.5`、`1.2e-3`                           | 按照 `f64` 语义求值。                                       |
| 布尔字面量     | `true`、`false`                                | 用于条件表达式和逻辑表达式。                           |
| 括号         | `(a + b) / 2`                                  | 使用括号来改变运算优先级。                                 |
| 一元运算符     | `-x`、`!flag`                                  | 一元 `-` 对数字取负。一元 `!` 对布尔值取反。                |
| 二元运算符    | `+ - * / % ^`、`== !=`、`< <= > >=`、`&& \|\|` | 算术运算符作用于数值。逻辑运算符作用于布尔值。                 |
| 局部变量赋值    | `spread = a - b; spread / 2`                   | 语句从左到右依次执行。公式必须以一个值结尾。 |
| 注释            | `// line`、`/* block */`                       | 注释会被忽略。                                                  |

:::note
新的公式应使用原始 `InstrumentId` 值。为了向后兼容，将成分 ID 中的
`-` 替换为 `_` 的公式仍然会被接受。
:::

### 运算符优先级

表达式引擎按照以下顺序（从最高优先级到最低优先级）对运算符求值：

| 级别   | 运算符            | 说明                                                        |
|---------|----------------------|--------------------------------------------------------------|
| 最高 | `^`                  | 幂运算。右结合。                           |
|         | 一元 `-`、一元 `!` | `-2 ^ 2` 的求值结果等价于 `-(2 ^ 2)`。                            |
|         | `*`、`/`、`%`        | 乘法、除法和取模。                        |
|         | `+`、`-`             | 加法和减法。                                    |
|         | `<`、`<=`、`>`、`>=` | 数值比较。                                         |
|         | `==`、`!=`           | 相等和不等比较。两侧必须是相同的类型。 |
| 最低  | `&&`、`\|\|`         | 布尔运算符。                                           |

赋值不是表达式运算符。使用 `;` 分隔各条语句，并让最后一条
语句成为合成金融工具希望产生的值。

### 内置函数

| 函数 | 签名                              | 说明                                                |
|----------|----------------------------------------|------------------------------------------------------|
| `abs`    | `abs(x)`                               | 绝对值。                                      |
| `ceil`   | `ceil(x)`                              | 向上取整。                                             |
| `floor`  | `floor(x)`                             | 向下取整。                                             |
| `round`  | `round(x)`                             | 使用 Rust `f64` 规则四舍五入到最近的整数。 |
| `min`    | `min(x1, x2, ...)`                     | 接受一个或多个数值参数。               |
| `max`    | `max(x1, x2, ...)`                     | 接受一个或多个数值参数。               |
| `if`     | `if(condition, when_true, when_false)` | 条件必须是布尔值。两个分支的类型必须一致。只会对被选中的分支求值。 |

### 类型规则

- 成分输入是数值类型。
- 算术运算符要求操作数为数值类型，并返回数值结果。
- `<`、`<=`、`>`、`>=` 要求操作数为数值类型，并返回布尔结果。
- `==` 和 `!=` 接受任意匹配的类型（两侧均为数值或均为布尔），并返回布尔
  结果。
- `&&`、`||` 以及一元 `!` 要求操作数为布尔类型。
- `&&` 和 `||` 采用短路求值。右侧仅在需要时才会求值。
- 局部变量必须先赋值后使用。
- 局部变量名必须以字母或 `_` 开头，后续可使用字母、数字或 `_`。
- 公式的最终结果必须是数值。以赋值结尾或产生布尔结果的公式对于
  合成金融工具来说是无效的。

### 限制

表达式引擎在编译期强制执行以下限制。超出限制的公式会在构造时
产生明确的错误。

| 限制            | 值 | 说明                                                    |
|------------------|-------|------------------------------------------------------------------|
| 栈深度      | 32    | 求值栈上中间值的最大数量。 |
| 局部变量  | 16    | 不同局部变量名的最大数量。               |

对于任何实际的定价公式来说，这些限制都是相当宽松的。8 个成分的加权和
的峰值栈深度为 3，且不使用局部变量。

### 示例

```python
# Simple spread
formula = "BTCUSDT.BINANCE - ETHUSDT.BINANCE"

# Average of two FX pairs
formula = "(AUD/USD.SIM + NZD/USD.SIM) / 2"

# Reuse an intermediate value
formula = "spread = BTCUSDT.BINANCE - ETHUSDT.BINANCE; spread / 2"

# Conditional output
formula = "if(BTCUSDT.BINANCE > ETHUSDT.BINANCE, BTCUSDT.BINANCE, ETHUSDT.BINANCE)"
```

## 创建合成金融工具

在定义新的合成金融工具之前，请确保所有成分金融工具已经存在于
缓存中。

以下示例通过 actor 或策略创建了一个合成金融工具。该合成工具
表示 Binance 上比特币和以太坊现货价格之间的简单价差。该示例假定
`BTCUSDT.BINANCE` 和 `ETHUSDT.BINANCE` 已存在于缓存中。

```python
from nautilus_trader.model.instruments import SyntheticInstrument

btcusdt_binance_id = InstrumentId.from_str("BTCUSDT.BINANCE")
ethusdt_binance_id = InstrumentId.from_str("ETHUSDT.BINANCE")

synthetic = SyntheticInstrument(
    symbol=Symbol("BTC-ETH:BINANCE"),
    price_precision=8,
    components=[
        btcusdt_binance_id,
        ethusdt_binance_id,
    ],
    formula=f"{btcusdt_binance_id} - {ethusdt_binance_id}",
    ts_event=self.clock.timestamp_ns(),
    ts_init=self.clock.timestamp_ns(),
)

self._synthetic_id = synthetic.id
self.add_synthetic(synthetic)
self.subscribe_quote_ticks(self._synthetic_id)
```

:::note
上面示例中的合成金融工具 `instrument_id` 为 `{symbol}.SYNTH`，即
`BTC-ETH:BINANCE.SYNTH`。
:::

## 更新公式

你可以随时更新合成工具的公式。

```python
synthetic = self.cache.synthetic(self._synthetic_id)

new_formula = "(BTCUSDT.BINANCE + ETHUSDT.BINANCE) / 2"
synthetic.change_formula(new_formula)

self.update_synthetic(synthetic)
```

## 触发金融工具 ID

你可以根据合成价格触发模拟订单。在以下示例中，一旦合成价格
达到触发条件，一个合成金融工具就会释放一个模拟订单。

```python
order = self.strategy.order_factory.limit(
    instrument_id=ETHUSDT_BINANCE.id,
    order_side=OrderSide.BUY,
    quantity=Quantity.from_str("1.5"),
    price=Price.from_str("30000.00000000"),
    emulation_trigger=TriggerType.DEFAULT,
    trigger_instrument_id=self._synthetic_id,
)

self.strategy.submit_order(order)
```

## 性能

公式在构造时编译一次，并在每次收到新的成分价格 tick 时求值。
表达式引擎采用“一次编译、多次求值”（compile-once/eval-many）的架构，
使用零分配的 f64 栈，因此求值给 tick 处理路径带来的开销可以忽略不计。

在 Apple M4 Pro、rustc 1.94.1、release 配置（opt-level 3）下测得：

### 求值（热路径）

| 公式模式                         | 耗时  |
|-----------------------------------------|-------|
| `(A + B) / 2.0`                         | 12 ns |
| `A * 0.4 + B * 0.3 + C * 0.2 + D * 0.1` | 18 ns |
| `if(A > B, A - B, B - A)`               | 12 ns |
| `spread = A - B; mid = ...; mid + ...`  | 19 ns |
| `max(min(A, B * 20), abs(A - B))`       | 15 ns |

### 求值扩展性（加权和）

| 成分数量 | 耗时  |
|------------|-------|
| 2          | 14 ns |
| 4          | 18 ns |
| 8          | 28 ns |

### 编译（冷路径）

| 公式模式    | 耗时   |
|--------------------|--------|
| 简单平均值     | 675 ns |
| 4 输入加权   | 1.4 us |
| 条件表达式        | 1.0 us |
| 带局部变量        | 1.3 us |
| 带连字符的 ID     | 755 ns |

## 错误处理

Nautilus 在每个边界处都会对合成金融工具进行校验。公式编译会拒绝
未知符号、类型错误和容量溢出。求值会在数值到达公式之前拒绝
错误的输入数量以及非有限值的价格（NaN、Infinity）。

有关输入要求和异常的说明，请参阅
[`SyntheticInstrument` API 参考文档](/docs/python-api-latest/model/instruments.html#nautilus_trader.model.instruments.synthetic.SyntheticInstrument)。

## 相关指南

- [金融工具（Instruments）](instruments/) - 金融工具定义以及特定于交易场所的金融工具类型。
- [数据（Data）](data/) - 引用金融工具的市场数据类型。
- [订单（Orders）](orders/) - 订单可以使用合成金融工具 ID 作为模拟触发条件。
