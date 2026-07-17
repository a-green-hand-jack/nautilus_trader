# Python

[Python](https://www.python.org/) 编程语言承担了 NautilusTrader 中大部分面向用户的代码。
Python 拥有丰富的库和框架生态系统，非常适合用于策略开发、数据分析和系统集成。

## 代码风格

### PEP-8

代码库总体上遵循 PEP-8 风格指南。一个值得注意的例外是：对于除集合类型以外的其他情况，
代码并不总是利用 Python 的真值判断（truthiness）来检查参数是否为 `None`。

按照 [Google Python 风格指南](https://google.github.io/styleguide/pyguide.html)的建议，
当有可能向函数或方法传入意外对象、导致真值判断产生意外结果（可能造成逻辑类型的 bug）时，
不建议使用真值判断来检查参数是否为/不是 `None`。

*"始终使用 if foo is None:（或 is not None）来检查 None 值。例如，测试一个默认值为
None 的变量或参数是否被设置为了其他值时。这个其他值可能在布尔上下文中恰好为假！"*

:::note
对于空集合的检查，使用真值判断（例如 `if not my_list:`），而不是显式与 `None` 或空值比较。
:::

我们欢迎所有关于代码库在没有明显理由的情况下偏离 PEP-8 的反馈。

### 类型提示

所有函数和方法签名*必须*包含类型注解：

```python
def __init__(self, config: EMACrossConfig) -> None:
def on_bar(self, bar: Bar) -> None:
def on_save(self) -> dict[str, bytes]:
def on_load(self, state: dict[str, bytes]) -> None:
```

**联合类型语法**：可选类型使用 PEP 604 的联合类型语法：

```python
# 推荐
def get_instrument(self, id: InstrumentId) -> Instrument | None:

# 避免
def get_instrument(self, id: InstrumentId) -> Optional[Instrument]:
```

**泛型类型**：对可复用的组件使用 `TypeVar`：

```python
T = TypeVar("T")
class ThrottledEnqueuer(Generic[T]):
```

### 文档字符串（Docstrings）

整个代码库都使用 [NumPy 文档字符串规范](https://numpydoc.readthedocs.io/en/latest/format.html)。
需要始终一致地遵循该规范，以确保文档能够正确构建。

**Python** 文档字符串应使用 **祈使语气** 编写——例如，*"Return a cached client."*
（返回一个缓存的客户端）。

这一约定与 Python 生态系统的主流风格保持一致，能让生成的文档对最终用户来说更自然。

#### 私有方法

不要为私有方法（以 `_` 为前缀）添加文档字符串：

- 文档字符串会生成面向公众的 API 文档。
- 为私有方法添加文档字符串会错误地暗示它们属于公共 API 的一部分。
- 私有方法是实现细节，并非面向最终用户设计的。

可以接受添加文档字符串的例外情况：

- 逻辑非常复杂、包含多个步骤或重要边界情况的方法。
- 由于复杂度较高、需要详细记录参数或返回值的方法。

当私有方法需要一些上下文说明（例如某个棘手的前置条件或副作用）时，优先在相关逻辑附近使用
简短的内联注释（`#`），而不是文档字符串。

### 属性 vs 方法（PyO3 绑定）

通过 PyO3 将 Rust 类型暴露给 Python 时，应根据调用处所传达的语义来选择使用 `#[getter]`
（属性）还是普通方法，而不是根据该值是否可变来决定：

- **属性（`#[getter]`）**：开销小、无副作用、类似属性的当前状态视图。标量字段、
  判断式（predicate）和轻量的派生值都属于这一类，即使它们会在对象的生命周期内发生变化。
  例如：`status`、`side`、`quantity`、`price`、`is_open`、`has_inputs`、
  `realized_pnl`、`venue_order_id`。
- **方法（不带 `#[getter]`）**：动作、修改操作、非平凡的计算工作、分配/拷贝、I/O，
  或任何需要接受参数的操作。
  例如：`apply(fill)`、`unrealized_pnl(price)`、`calculate_pnl(...)`。
- **灰色地带（优先使用方法）**：每次调用都会克隆或分配一个集合的 getter。
  使用方法能向调用方传达这一开销。
  例如：`events()`、`adjustments()`、`client_order_ids()`、`trade_ids()`。

## Python v2 实盘回调路由

Python v2 实盘节点维持一条运行时不变量：Tokio 工作线程在实盘交易期间不运行 Python 代码。

`LiveNode::py_run` 会在 Rust 异步运行时运行期间释放 GIL。必须触发 Python 的
工作线程侧工作，使用现有的实盘运行器事件通道，而不是在工作线程上调用 `Python::attach`。
定时器回调使用时间事件通道。运行器会在启动缓冲阶段和主 select 循环中排空该通道，
然后在实盘事件循环线程上执行回调。

这条路径是用于处理不可避免的用户 Python 回调工作的边界，而不是用来把适配器、
provider、数据或执行逻辑迁移到 Python 中的地方。Python v2 适配器模块负责配置 Rust
适配器并注册工厂；适配器操作由 Rust 拥有。如果工作线程侧的 Rust 工作需要一个 Python
回调，应通过一个属于实盘运行器的特定事件类型来路由。

在添加涉及 Python 的实盘代码时：

- 优先使用现有的运行器事件通道。
- 让回调函数体保持简短，因为它们会在实盘事件循环上同步运行。
- 不要在 Python v2 实盘交易的 Tokio 工作线程任务中调用 `Python::attach`。
- 不要为了适配回调路由而在 Python 中添加适配器业务逻辑。

旧版的 Cython `LiveClock` 回调走的是另一条独立的 FFI 路径。它们使用胶囊风格的
回调参数以兼容 v1，并且可以在没有实盘运行器发送方的情况下创建。在时间事件调度能够
在 v1 和 v2 之间统一之前，请让这套 ABI 保持独立。

### 测试命名

使用具描述性的名称来说明测试场景：

```python
def test_currency_with_negative_precision_raises_overflow_error(self):
def test_sma_with_no_inputs_returns_zero_count(self):
def test_sma_with_single_input_returns_expected_value(self):
```

### Ruff

代码库使用 [ruff](https://astral.sh/ruff) 进行代码检查（lint）。Ruff 的规则可以在
顶层的 `pyproject.toml` 中找到，忽略某项规则的理由通常会以注释形式给出。

## Cython（旧版）

:::note
本节涵盖了 `.pyx` 和 `.pxd` 文件的 Cython 约定。
:::

对于 `.pyx` 和 `.pxd` 文件，请确保所有返回 `void` 或原始 C 类型（如 `bint`、`int`、
`double`）的函数和方法在签名中都包含 `except *` 关键字。如果不这样做，Python 异常会被
静默忽略。

更多信息，请参见 [Cython 文档](https://cython.readthedocs.io/en/latest/index.html)。
