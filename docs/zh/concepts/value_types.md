# 数值类型（Value Types）

NautilusTrader 提供了用于表示核心交易概念的专用数值类型：
`Price`、`Quantity` 和 `Money`。这些类型内部使用定点算术（fixed-point arithmetic）
来实现跨不同平台和环境的高性能、确定性计算。

## 概览

| 类型       | 用途                                  | 有符号 | 货币 |
|------------|------------------------------------------|--------|----------|
| `Quantity` | 交易规模、订单数量、仓位。   | 否     | -        |
| `Price`    | 市场价格、报价、价格档位。     | 是    | -        |
| `Money`    | 货币金额、盈亏、账户余额。 | 是    | 是       |

## 不可变性

所有数值类型都是**不可变的**。一旦构造完成，值就不能被更改。
运算不会改变原始对象。

```python
from nautilus_trader.model.objects import Quantity

qty1 = Quantity(100, precision=0)
qty2 = Quantity(50, precision=0)

# This creates a NEW Quantity; qty1 and qty2 are unchanged
result = qty1 + qty2

print(qty1)    # 100
print(qty2)    # 50
print(result)  # 150
```

这种设计带来了多项好处：

- **线程安全**：不可变值可以在线程之间安全共享，无需同步。
- **可预测性**：值不会意外变化，使调试更容易。
- **可哈希性**：不可变类型可以用作字典的键，或放入集合中。

## 算术运算

数值类型支持标准算术运算符（`+`、`-`、`*`、`/`、`%`、`//`）
和一元运算符（`-`、`+`、`abs`）。返回类型取决于运算符
和操作数的类型。

### 同类型二元运算

同一数值类型之间的加法和减法返回该类型本身，保留了
领域含义（一个价格加上一个价格仍然是价格）：

| 运算             | 结果     |
|-----------------------|------------|
| `Quantity + Quantity` | `Quantity` |
| `Quantity - Quantity` | `Quantity` |
| `Price + Price`       | `Price`    |
| `Price - Price`       | `Price`    |
| `Money + Money`       | `Money`    |
| `Money - Money`       | `Money`    |

```python
from nautilus_trader.model.objects import Price

price1 = Price(100.50, precision=2)
price2 = Price(0.25, precision=2)

result = price1 + price2  # Returns Price(100.75, precision=2)
print(type(result))       # <class 'Price'>
```

同一类型的两个值之间的乘法、除法、整除和取模运算返回 `Decimal`：

| 运算             | 结果    |
|-----------------------|-----------|
| `Price * Price`       | `Decimal` |
| `Price / Price`       | `Decimal` |
| `Price // Price`      | `Decimal` |
| `Price % Price`       | `Decimal` |

`Quantity` 和 `Money` 也遵循相同的模式。

这些运算不会返回原始类型，因为结果具有不同的量纲含义。价格乘以价格
得到的是“价格的平方”，而不再是价格。数量除以数量得到的是一个
无量纲比值，而不是数量。返回 `Decimal` 使单位的变化变得明确，
防止结果被误认为是具有原始单位的值。

### 一元运算

一元运算符会在结果对该类型有效的情况下保留数值类型：

| 运算    | `Price`   | `Quantity` | `Money`   |
|--------------|-----------|------------|-----------|
| `-x`（取负）   | `Price`   | `Decimal`  | `Money`   |
| `+x`（取正）   | `Price`   | `Quantity` | `Money`   |
| `abs(x)`     | `Price`   | `Quantity` | `Money`   |
| `int(x)`     | `int`     | `int`      | `int`     |
| `float(x)`   | `float`   | `float`    | `float`   |
| `round(x)`   | `Decimal` | `Decimal`  | `Decimal` |

`Quantity.__neg__` 返回 `Decimal` 而不是 `Quantity`，因为 `Quantity`
是无符号的，无法表示负值。

```python
from nautilus_trader.model.objects import Price, Quantity, Money
from nautilus_trader.model.currencies import USD

price = Price(100.50, precision=2)
print(-price)            # -100.50
print(type(-price))      # <class 'Price'>

money = Money(-50.00, USD)
print(abs(money))        # 50.00 USD
print(type(abs(money)))  # <class 'Money'>

qty = Quantity(10, precision=0)
print(+qty)              # 10
print(type(+qty))        # <class 'Quantity'>
```

### 混合类型运算

当与其他数值类型进行运算时，结果类型遵循 Python 的
[数值塔（numeric tower）](https://docs.python.org/3/library/numbers.html)约定。
总体原则是运算会朝更通用的类型扩展：`float` 运算返回
`float`，而 `int` 和 `Decimal` 运算为保留精度返回 `Decimal`。

这适用于全部六种二元运算符（`+`、`-`、`*`、`/`、`//`、`%`），
并且在两个方向上都成立（`值 op 标量` 和 `标量 op 值`）：

| 左操作数 | 右操作数 | 结果类型 |
|--------------|---------------|-------------|
| 数值类型   | `int`         | `Decimal`   |
| 数值类型   | `float`       | `float`     |
| 数值类型   | `Decimal`     | `Decimal`   |
| `int`        | 数值类型    | `Decimal`   |
| `float`      | 数值类型    | `float`     |
| `Decimal`    | 数值类型    | `Decimal`   |

```python
from decimal import Decimal
from nautilus_trader.model.objects import Quantity

qty = Quantity(100, precision=0)

# Quantity + int -> Decimal
result1 = qty + 50
print(type(result1))  # <class 'decimal.Decimal'>

# Quantity + float -> float
result2 = qty + 50.5
print(type(result2))  # <class 'float'>

# Quantity + Decimal -> Decimal
result3 = qty + Decimal("50")
print(type(result3))  # <class 'decimal.Decimal'>
```

## 精度处理

每个数值类型都存储一个精度字段，表示小数位数。
精度在构造时设定，且不可变。不存在“未指定”的精度。

### 定点表示

数值类型在内部以按全局固定精度缩放后的整数存储
（例如高精度模式下为 10^16），而不是浮点数。`precision`
字段记录构造时使用的小数位数，用于控制显示格式和
序列化，但底层的原始值始终使用全局比例。

```python
from nautilus_trader.model.objects import Price

p1 = Price(1.23, precision=2)   # displays as "1.23"
p2 = Price(1.230, precision=3)  # displays as "1.230"

p1 == p2  # True: same underlying value
str(p1)   # "1.23"
str(p2)   # "1.230"
```

**精度控制的是显示方式，而非身份（identity）。** 两个小数值相同但
精度不同的价格是相等的。`precision` 字段决定了字符串格式以及
显示多少位小数，但相等性判断基于底层的数值。

**市场数据序列化使用精度元数据。** 当市场数据类型（报价、
成交、订单簿差量）写入 Parquet 或 Arrow 格式时，精度会存储在
文件元数据中，以便正确解码数值。同一个文件内的所有市场数据值
必须使用相同的精度。

:::note
如果某个交易场所更改了金融工具的最小报价单位（tick size，从而改变了其
精度），那么变更前后写入的数据文件将具有不同的精度元数据，不应
合并到同一个文件中。
:::

关于金融工具级别的精度如何约束有效的价格和数量，请参阅
金融工具指南中的[精度（Precision）](instruments/index.md#precision)部分。

### 算术精度

在对不同精度的值进行算术运算时，结果使用操作数中
精度较大的那个。

```python
from nautilus_trader.model.objects import Price

price1 = Price(100.5, precision=1)    # 1 decimal place
price2 = Price(0.125, precision=3)    # 3 decimal places

result = price1 + price2
print(result)            # 100.625
print(result.precision)  # 3 (max of 1 and 3)
```

## 特定类型的约束

### Quantity

`Quantity` 表示非负数量。尝试创建负数量，
或用较小的数量减去较大的数量，都会引发错误：

```python
from nautilus_trader.model.objects import Quantity

# This raises ValueError: Quantity cannot be negative
qty = Quantity(-100, precision=0)

# This also raises ValueError
qty1 = Quantity(50, precision=0)
qty2 = Quantity(100, precision=0)
result = qty1 - qty2  # Would be -50, which is invalid
```

### Money

`Money` 值包含一个货币。`Money` 值之间的加法和减法
要求货币必须一致：

```python
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USD, EUR

usd_amount = Money(100.00, USD)
eur_amount = Money(50.00, EUR)

# This works - same currency
result = usd_amount + Money(25.00, USD)

# This raises ValueError - currency mismatch
result = usd_amount + eur_amount
```

## 常见模式

### 累加数值

由于数值类型是不可变的，累加需要通过重新赋值来实现：

```python
from nautilus_trader.model.objects import Money
from nautilus_trader.model.currencies import USD

total = Money(0.00, USD)
amounts = [Money(100.00, USD), Money(50.00, USD), Money(25.00, USD)]

for amount in amounts:
    total = total + amount  # Reassign to new Money instance

print(total)  # 175.00 USD
```

### 转换为其他类型

数值类型提供了转换方法：

```python
from nautilus_trader.model.objects import Price

price = Price(123.456, precision=3)

# Convert to Decimal (preserves precision)
decimal_value = price.as_decimal()

# Convert to float
float_value = price.as_double()

# Convert to string
string_value = str(price)  # "123.456"
```

### 从字符串创建

从字符串表示解析出数值类型：

```python
from nautilus_trader.model.objects import Quantity, Price, Money

qty = Quantity.from_str("100.5")
price = Price.from_str("99.95")
money = Money.from_str("1000.00 USD")
```
