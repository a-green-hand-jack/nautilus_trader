# 测试

我们的自动化测试是这个交易平台的可执行规范。一个健康的测试套件记录了预期行为，
让贡献者能够放心地进行重构，并在问题到达生产环境之前捕获回归。测试还兼具活文档的
作用，能够阐明复杂的流程，并提供快速的 CI 反馈，使问题尽早暴露。

该测试套件涵盖以下几类：

- 单元测试（Unit tests）
- 集成测试（Integration tests）
- 验收测试（Acceptance tests）
- 性能测试（Performance tests）
- 基于属性的测试（Property-based tests）
- 模糊测试（Fuzzing）
- 内存泄漏测试（Memory leak tests）

## 测试策略

测试与运行时契约构成同一个设计体系。[设计契约（Design by contract）](rust.md#design-by-contract)
这条阶梯尽可能地将不变量推入类型系统；下面的测试阶梯则通过更大的输入空间和更丰富的
执行模型，逐步升级来应对剩余的未知情况。每一层都在下一层无法触及的输入或执行状态上
扩展了覆盖范围。

并非每个模块都需要用到每一种技术。在添加测试或 `debug_assert!` 语句之前，
可以用本节内容来决定哪些层级适用。

### 机制阶梯

运行时契约的内容涵盖在 [Rust 指南](rust.md#design-by-contract) 中：优先使用类型系统，
接下来在 API 边界使用 `nautilus_core::correctness` 中的 `check_*`，然后对内部不变量使用
`debug_assert!`，最后对健全性关键或始终启用的检查使用 `assert!`。

测试层级遵循一个并行的升级路径。从能够证明所需内容的最低层级开始；只有当下一层
不再检测到回归、或者输入空间超出了手工挑选的用例范围时，才向上升级。

| 层级                    | 触发条件                                                                               |
|--------------------------|---------------------------------------------------------------------------------|
| 单元测试                | 单个函数或状态转换只有一小组可枚举的用例。           |
| 参数化测试        | 相同的形态在离散输入间重复出现（订单方向、状态、金融工具）。 |
| 基于属性的测试      | 某个不变量必须对一整类无法凭直觉枚举的输入都成立。   |
| 集成测试         | 多个模块通过真实（非模拟）引擎或运行时相互交互。        |
| 模糊测试                | 不可信或对抗性的字节流经过解析器、解码器或线协议处理器。 |
| 规范验收测试     | 行为依赖于一份真实交易场所的契约（参见 `spec_exec_testing.md`）。        |
| 确定性仿真测试 | 正确性依赖于任务调度、超时或墙钟顺序。       |
| 形式化验证      | 一个纯函数具有清晰的不变量和有限的输入空间，值得为其证明。   |

形式化验证这一档是有待实现的目标：目前工作区中还没有落地任何 Kani 或 Prusti 验证工具。
这一行记录的是采用某种验证器时的升级触发条件，而不是当前的强制要求。

### 投影规则

模块的形态决定了哪些层级值得投入。并非每个模块都需要走完整个阶梯。应在模块粒度而非
crate 粒度上应用该规则：一个适配器 crate 可能同时包含纯粹的解析器和 I/O 密集型的客户端
循环，每一行适用于其中不同的部分。

| 模块形态                        | 适用的层级                             | 示例                                |
|-------------------------------------|-----------------------------------------------|-----------------------------------------|
| 纯函数，具有清晰不变量     | 单元、参数化、属性、模糊测试            | 对账内核、投资组合数学计算 |
| 纯函数，没有明确声明的不变量 | 单元、参数化、属性、模糊测试            | 编解码器、适配器解析器、格式化器    |
| 有状态，同步                | 单元、参数化、针对状态转换的属性测试 | 缓存、订单簿                    |
| 有状态，异步                    | 单元、集成、确定性仿真测试   | 实盘引擎、执行管理器         |
| I/O 密集型，交易场所契约           | 集成、规范验收测试、边界模糊测试   | 适配器客户端循环                    |

### 何时不应添加覆盖率

- 只在测试能够触及的位置添加 `debug_assert!`。发布构建会剥离该检查，因此一个从未被
  执行到的断言没有任何信号价值。一个有针对性的单元测试就算作一个测试驱动；proptest
  或模糊测试驱动能进一步放大这一信号。
- 当某个不变量涵盖一整类输入时，优先使用 proptest 而不是手写的边界情况测试。
  针对已知交易场所异常情况的有针对性单元测试仍然是有效的，也可以作为收缩后的
  反例（counterexample）的回归复现用例。
- 不要把一个已有的实时规范验收卡片重复写成集成测试，而应链接过去。
- 不要用断言语言或框架自身保证的测试来凑覆盖率数字（例如 `Some(..)` 之后再断言
  `Option::is_some`，`push` 之后再断言 `Vec::len`）。

### DST（确定性仿真测试）就绪度

确定性仿真测试（DST）要求运行时不存在环境层面的不确定性。在将某个模块提升为支持
DST 运行之前，请验证以下几点：

- 时间、任务、运行时和信号相关的原语都通过 `nautilus_common::live::dst` 路由，
  而不是直接使用 `tokio`。墙钟读取通过 `nautilus_core::time` 中的接缝（seam）进行，
  而不是在调用点直接使用 `SystemTime::now()`。
- 依赖迭代顺序的状态映射使用 `IndexMap` 或 `IndexSet`，而不是默认的哈希集合。
- 控制平面路径上的每一个 `tokio::select!` 都设置了 `biased`，以固定轮询顺序。
- 没有任何 `Instant::now()`、`SystemTime::now()`、`tokio::signal::ctrl_c`、
  `std::thread::spawn` 或 `tokio::task::spawn_blocking` 的调用绕开了这条接缝。
  阻塞线程和操作系统线程原语会以与读取环境时钟同样的方式破坏 madsim 的确定性。
- 与重放相关的 ID（`trade_id`、`venue_order_id`）都是其输入的纯函数；参见
  `crates/execution/src/reconciliation/ids.rs`。其他对账路径上的临时性事件 UUID
  不需要具有确定性。

`crates/common/src/live/dst.rs` 中的 `surface` 探针只固定了重新导出的接口形态；
它并不检查调用方是否确实使用了该接缝。这一点的强制执行依靠代码评审。每当有新的
异步模块进入工作区，或现有模块获得了新的控制平面调度逻辑时，都应进行审计。

## 基于属性的测试

属性测试验证某项逻辑对*所有*有效输入都成立，而不仅仅是手工挑选的示例。
我们在 Rust 中使用 [`proptest`](https://altsysrq.github.io/proptest-book/intro.html)
来强制实施不变量。

- **使用场景**：核心领域类型（`Price`、`Quantity`、`UnixNanos`）、会计引擎、
  撮合引擎，以及状态机。
- **不变量示例**：
  - 往返序列化：`parse(to_string(value)) == value`
  - 逆运算：`(A + B) - B == A`
  - 传递性：`若 A < B 且 B < C，则 A < C`

## 模糊测试

模糊测试向系统引入非结构化或恶意的数据，以验证系统能够优雅地失败。

- **使用场景**：网络边界、交易所数据解析器（JSON、FIX、WebSocket 数据流），
  以及复杂的状态机。
- **目标**：在遇到格式错误的数据时，系统返回 `Result::Err`，绝不 panic、
  挂起或泄漏内存。

在构建或修改核心类型时，编写属性测试来覆盖数学边界情况。

性能测试有助于对性能关键型组件进行迭代演进。

使用 [pytest](https://docs.pytest.org)（我们的主要测试运行器）运行测试。
使用参数化测试和 fixture（例如 `@pytest.mark.parametrize`）来避免重复代码，
提升清晰度。

## 运行测试

### v1 旧版 Python 测试

v1 旧版测试套件位于仓库根目录的 `tests/` 下，测试的是基于 Cython 的包。
在仓库根目录下：

```bash
make pytest
# 或
uv run --active --no-sync pytest --new-first --failed-first
```

### Python 测试

Python 测试套件位于 `python/tests/` 下，测试的是由 Rust 支撑的 PyO3 包。
它需要一个已构建的扩展模块（`make build-debug-v2`），并使用其位于
`python/.venv/` 下的独立虚拟环境。

对于 v2 路径下新的实盘适配器示例和文档，优先使用 `nautilus_trader.live.LiveNode`。
`nautilus_trader.live.node.TradingNode` 仍是根级 `tests/` 套件和较旧示例所使用的
旧版 v1/Cython 运行时。

```bash
make pytest-v2
```

该 Makefile 目标会将某些测试模块隔离到独立的 pytest 进程中，以避免全局 Rust 状态
冲突。请使用 `make pytest-v2`，而不要直接调用 pytest。

本地运行 `make pytest-v2` 使用的是通过 `make build-debug-v2` 构建的调试版扩展。
CI 中的 `build-v2` 测试的是发布版 wheel。
不要在 `python/tests/` 中编写在进程内用 `pytest.raises(BaseException)` 之类宽泛捕获
方式探测 Rust panic 路径的测试用例。这类测试针对调试版构建可能看起来通过，但针对
发布版 wheel 会导致解释器中止。对于容易导致中止的 PyO3 或 FFI 方法，请验证 Python
签名和参数名，或者将该调用隔离到子进程中执行。

对于性能测试：

```bash
make test-performance
# 或
uv run --active --no-sync pytest tests/performance_tests --benchmark-disable-gc --codspeed
```

`--benchmark-disable-gc` 标志可防止垃圾回收干扰测试结果。请单独运行性能测试
（不要与单元测试混在一起），以避免相互干扰。

### Rust 测试

```bash
make cargo-test
# 或
cargo nextest run --workspace --features "arrow,ffi,python,high-precision,streaming,defi" --cargo-profile nextest --lib --tests
```

#### 使用可选特性进行测试

使用 `EXTRA_FEATURES` 来包含诸如 `capnp` 或 `hypersync` 之类的可选特性：

```bash
# 使用 capnp 特性进行测试
make cargo-test EXTRA_FEATURES="capnp"

# 使用多个特性进行测试
make cargo-test EXTRA_FEATURES="capnp hypersync"

# hypersync 的旧版简写方式
make cargo-test HYPERSYNC=true

# 使用特性测试特定的 crate
make cargo-test-crate-nautilus-serialization FEATURES="capnp"
```

### IDE 集成

- **PyCharm**：右键点击测试目录或文件 -> "Run pytest"。
- **VS Code**：使用 Python Test Explorer 扩展。

## 测试风格

### 通用

- 根据测试所验证的内容来命名测试函数；无需在名称中编码预期的断言内容。
- 当能够澄清设置、场景或预期结果时，添加文档字符串。
- **尽可能对断言进行分组**：先完成所有的设置/执行步骤，然后一起进行断言，
  以避免"执行—断言—再执行"这种不良模式。
- 在测试内部使用 `unwrap`、`expect` 或直接的 `panic!`/`assert` 调用；在这里，
  清晰简洁比防御性错误处理更重要。
- 不要通过捕获日志输出来对日志消息进行断言。在测试中捕获日志是脆弱的做法，
  因为日志记录器是全局状态，测试执行顺序是不确定的，并且当日志措辞发生变化时
  这类断言就会失效。请转而验证该日志消息所反映的可观察行为（返回值、状态变化、
  副作用）。

### Python 测试（`python/tests/`）

使用 **pytest 风格的自由函数和 fixture**，不要使用测试类。

- 将每个测试写成独立的 `def test_*()` 函数。
- 对共享的设置（金融工具、引擎实例、数据）使用 `@pytest.fixture`。
  当需要清理（teardown）时（例如 `engine.dispose()`），优先使用 `yield` fixture。
- 使用 `@pytest.mark.parametrize` 覆盖多个输入，而不必重复测试主体。
- 从 `nautilus_trader.model` 导入模型类型，而不是从
  `nautilus_trader.core.nautilus_pyo3` 导入。
- 测试提供者（provider）位于 `python/tests/providers.py` 中。使用
  `TestInstrumentProvider` 和 `TestDataProvider` 来获取常用的金融工具和数据。
- 对依赖尚未完成的功能的测试，使用
  `@pytest.mark.skip(reason="WIP: <description>")` 标记，而不要直接删除它们。

### v1 旧版 Python 测试（`tests/`）

v1 旧版测试套件混用了测试类和自由函数。添加到该套件的新测试可以遵循任一模式，
但对于新文件，更推荐使用带 fixture 的自由函数。

### Rust

关于 Rust 特有的测试约定（模块结构、`#[rstest]`、参数化），
请参见 [Rust 指南](rust.md#testing-conventions)。

## 等待异步效果完成

当需要等待后台工作完成时，优先使用轮询辅助函数
`nautilus_trader.test_kit.functions` 中的 `await eventually(...)`
以及 `nautilus_common::testing` 中的 `wait_until_async(...)`，而不是使用任意的
sleep。它们能更快地暴露失败，并减少 CI 中的不稳定性，因为它们会在条件满足时
立即停止，或者在超时时给出有用的错误信息。

## Mock（模拟对象）

优先使用返回固定值的手写桩（stub），而不是使用 mock 框架。只有当你需要断言
调用次数/参数，或需要模拟复杂的状态变化时，才使用 `MagicMock`。避免 mock
你实际正在测试的对象本身。

## 代码覆盖率

我们使用 `coverage` 生成覆盖率报告，并发布到 [codecov](https://about.codecov.io/)。

在不牺牲适当的错误处理、也不对架构造成"测试引发的损害"的前提下，力求较高的覆盖率。

有些分支在不修改生产行为的情况下是无法测试的。例如，一个防御性 if-else 代码块中的
最终条件可能只在遇到意外值时才会触发；请保留这些检查，以便未来的变更在需要时
能够触及它们。

设计时的异常情况也可能难以测试，因此 100% 的覆盖率并不是目标。

## 排除代码覆盖率

当测试是多余的时，我们使用 `pragma: no cover` 注释来
[从覆盖率中排除代码](https://coverage.readthedocs.io/en/coverage-4.3.3/excluding.html)。
典型的例子包括：

- 断言某个抽象方法在被调用时抛出 `NotImplementedError`。
- 断言某个 if-else 代码块中无法测试的最终条件检查（如上所述）。

这类测试的维护成本很高，因为它们必须跟随重构而变化，却提供不了多少价值。
应保持抽象方法的具体实现拥有完整的覆盖率。当 `pragma: no cover` 不再适用时应将其移除，
并将其使用限制在上述情形中。

## 调试 Rust 测试

使用默认的测试配置来调试 Rust 测试。

要运行带调试符号的完整测试套件以供后续使用，请运行 `make cargo-test-debug`
而不是 `make cargo-test`。

在 IntelliJ IDEA 中，请调整参数化 `#[rstest]` 用例的运行配置，使其读作
`test --package nautilus-model --lib data::bar::tests::test_get_time_bar_start::case_1`
（去掉 `-- --exact`，并追加 `::case_n`，其中 `n` 从 1 开始）。这个变通方法与
[此处](https://github.com/rust-lang/rust-analyzer/issues/8964#issuecomment-871592851)
所解释的行为一致。

在 VS Code 中你可以直接选择要调试的具体测试用例。

## Python + Rust 混合调试

这个工作流让你能够在 VS Code 内的 Jupyter notebook 中同时调试 Python 和 Rust 代码。

### 设置

安装以下 VS Code 扩展：Rust Analyzer、CodeLLDB、Python、Jupyter。

### 第 0 步：使用调试符号编译 `nautilus_trader`

   ```bash
   cd nautilus_trader && make build-debug-pyo3
   ```

### 第 1 步：设置调试配置

```python
from nautilus_trader.test_kit.debug_helpers import setup_debugging

setup_debugging()
```

该命令会创建所需的 VS Code 调试配置，并为 Python 调试器启动一个 `debugpy` 服务器。

默认情况下，`setup_debugging()` 期望在 `nautilus_trader` 根目录的上一级找到
`.vscode` 文件夹。如果你的工作区布局不同，请调整目标位置。

### 第 2 步：设置断点

- **Python 断点**：在 VS Code 中的 Python 源文件里设置。
- **Rust 断点**：在 VS Code 中的 Rust 源文件里设置。

### 第 3 步：启动混合调试

1. 在 VS Code 中选择 **"Debug Jupyter + Rust (Mixed)"** 配置。
2. 开始调试（F5）或点击绿色运行箭头。
3. Python 和 Rust 调试器都会附加到你的 Jupyter 会话中。

### 第 4 步：执行代码

运行调用 Rust 函数的 Jupyter notebook 单元格。调试器会在 Python 和 Rust 代码的断点处停下。

### 可用配置

`setup_debugging()` 会创建以下 VS Code 配置：

- **`Debug Jupyter + Rust (Mixed)`** - 用于 Jupyter notebook 的混合调试。
- **`Jupyter Mixed Debugging (Python)`** - 仅针对 notebook 的 Python 调试。
- **`Rust Debugger (for Jupyter debugging)`** - 仅针对 notebook 的 Rust 调试。

### 示例

打开并运行示例 notebook：`debug_mixed_jupyter.ipynb`。

### 参考资料

- [PyO3 调试](https://pyo3.rs/v0.25.1/debugging.html?highlight=deb#debugging-from-jupyter-notebooks)

## 数据类型测试

每种数据类型都会流经该平台的多个层级。下表展示了现有类型在哪些位置进行测试，
以便新类型能够遵循相同的模式。

### 测试层级矩阵

| 层级                  | 位置                                    | 覆盖内容                                             |
|------------------------|---------------------------------------------|--------------------------------------------------------------------------|
| DataEngine 订阅   | `crates/data/tests/engine.rs`               | 引擎正确处理订阅/取消订阅命令。 |
| DataEngine 发布     | `crates/data/tests/engine.rs`               | 引擎将已发布的数据路由到消息总线。           |
| DataActor 订阅    | `crates/common/src/actor/tests.rs`          | actor 通过类型化发布订阅并接收数据。      |
| DataActor 取消订阅  | `crates/common/src/actor/tests.rs`          | actor 在取消订阅后停止接收数据。              |
| PyO3 actor 分发    | `crates/common/src/python/actor.rs`         | Rust 处理器分发到 Python 的 `on_*` 方法。           |
| Python Actor 订阅 | `tests/unit_tests/common/test_actor.py`     | Python actor 完成订阅；命令计数递增。         |
| Python Actor 取消订阅     | `tests/unit_tests/common/test_actor.py`     | Python actor 完成取消订阅；订阅列表清空。       |
| 回测客户端        | `nautilus_trader/backtest/data_client.pyx`  | 回测客户端重写基类的订阅/取消订阅方法。      |
| 适配器实盘测试     | `docs/developer_guide/spec_data_testing.md` | 实盘数据验收测试（DataTester）。                   |

### 各数据类型的覆盖情况

下表展示了各数据类型在每个层级上的测试覆盖情况。
在添加新类型时可以将其作为检查清单使用。

| 数据类型           | 引擎 | Actor（Rust） | PyO3 分发 | Actor（Python） | 回测客户端 | 适配器规范 |
|---------------------|--------|--------------|---------------|----------------|-----------------|--------------|
| `InstrumentAny`     | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `OrderBookDeltas`   | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `OrderBook`         | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `QuoteTick`         | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `TradeTick`         | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `Bar`               | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `MarkPriceUpdate`   | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `IndexPriceUpdate`  | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `FundingRateUpdate` | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `InstrumentStatus`  | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `InstrumentClose`   | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `OptionGreeks`      | ✓      | ✓            | ✓             | ✓              | ✓               | ✓            |
| `OptionChainSlice`  | -      | ✓            | ✓             | ✓              | -               | ✓            |
| `CustomData`        | ✓      | ✓            | ✓             | ✓              | ✓               | -            |

`OptionChainSlice` 是由 DataEngine 的 `OptionChainManager` 根据每个金融工具的希腊值
（greeks）和报价订阅组装而成。它没有自己独立的引擎订阅命令或回测客户端重写方法。

### 添加新的数据类型

在引入新的数据类型时，请在每个层级都添加测试：

1. **DataEngine**（`crates/data/tests/engine.rs`）：添加 `test_execute_subscribe_<type>` 和
   `test_execute_unsubscribe_<type>` 测试。遵循现有订阅测试的模式：注册客户端、
   构建命令、调用 `engine.execute`、断言订阅列表。

2. **DataActor Rust**（`crates/common/src/actor/tests.rs`）：
   - 为 `TestDataActor` 添加 `received_<type>: Vec<Type>` 字段。
   - 在 `DataActor` trait 实现中实现 `on_<type>` 处理器。
   - 添加 `test_subscribe_and_receive_<type>` 和 `test_unsubscribe_<type>` 测试。
   - 对于使用 `TypedHandler` 路由的类型，使用类型化的发布函数
     （`msgbus::publish_<type>`），而不是 `publish_any`。

3. **PyO3 actor 分发**（`crates/common/src/python/actor.rs`）：
   - 添加调用 `py_self.call_method1("on_<type>", ...)` 的 `dispatch_on_<type>` 方法。
   - 在 `DataActor` trait 实现中添加调用该分发方法的 `on_<type>`。
   - 在 `#[pymethods]` 代码块中添加 `#[pyo3(name = "on_<type>")]` 方法。
   - 将 `on_<type>` 添加到 `RustTestDataActor` 包装器和内联 Python 测试类中。
   - 添加处理器测试和分发测试。

4. **Python Actor**（`tests/unit_tests/common/test_actor.py`）：
   - 添加 `test_subscribe_<type>` 和 `test_unsubscribe_<type>` 测试。
   - 断言在订阅之后 `actor.subscribed_<type>()` 返回预期条目，
     在取消订阅之后返回空列表。

5. **回测客户端**（`nautilus_trader/backtest/data_client.pyx`）：如果基类
   `MarketDataClient` 对该方法抛出 `NotImplementedError`，则重写
   `subscribe_<type>` 和 `unsubscribe_<type>`。

6. **文档**：在 `actors.md` 的回调表格、`strategies.md` 的处理器签名、`adapters.md`
   的订阅方法存根，以及 `spec_data_testing.md` 的测试卡片中添加对应条目。

:::tip
在全部六个层级中搜索一个已有类型（例如 `instrument_close` 或 `funding_rate`），
可以找到上述模式的具体示例。
:::
