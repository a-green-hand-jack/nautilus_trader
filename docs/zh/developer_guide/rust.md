# Rust

[Rust](https://www.rust-lang.org/learn) 提供了该平台关键任务核心所需的类型系统、所有权模型和
可预测的性能。安全 Rust（safe Rust）能在编译期防止数据竞争和许多内存错误。使用 `unsafe` 的代码
必须声明并遵守编译器无法检查的不变量。

## Cargo 清单约定

- 在 `[dependencies]` 中，先按字母顺序列出内部 crate（`nautilus-*`）。空一行，
  按字母顺序列出必需的外部依赖，再空一行，按字母顺序列出可选依赖。将内联注释与其所属的
  依赖项保持在一起。
- 为每一个构建 Python 产物的 `extension-module` 特性列表添加 `"python"`。将其与
  `"pyo3/extension-module"` 放在相邻位置，使完整的 Python 技术栈一目了然。
- 当清单将适配器单独分组时，将 `# Adapters` 代码块直接放在内部 crate 列表下方。
- 在 `[dev-dependencies]` 和 `[build-dependencies]` 部分之前始终留一个空行。
- 当特性或依赖集发生变化时，在相关清单之间保持相同的布局。
- 对 `bin/` 源文件使用 snake_case 文件名，例如 `bin/ws_data.rs`，并在每个 `[[bin]]`
  部分中使用这些路径。
- 保持 `[[bin]] name` 条目为 kebab-case，例如 `name = "hyperliquid-ws-data"`。

## 版本管理指南

- 对共享依赖使用工作区继承（例如 `serde = { workspace = true }`）。
- 只对不属于工作区的、crate 专属的依赖直接固定版本。
- 将工作区提供的依赖排在仅限该 crate 使用的依赖之前，以便审查继承关系。
- 保持相关依赖之间的版本对齐：`capnp`/`capnpc`（精确版本）、`arrow`/`parquet`（major.minor）、
  `datafusion`/`object_store`，以及 `dydx-proto`/`prost`/`tonic`。pre-commit 会强制执行这一点。
- 仅限适配器使用的依赖应放在工作区 `Cargo.toml` 的 "Adapter dependencies" 部分。
  pre-commit 会阻止核心 crate 使用这些依赖。

## 特性标志约定

- 优先使用可累加（additive）的特性标志。启用某个特性不应破坏现有功能。
- 使用能说明所启用能力的具描述性标志名称。
- 在 crate 级别的文档中记录每一个特性，以便使用者了解他们在切换什么。
- 常见模式：
  - `high-precision`：将定点数值类型从 64 位切换为 128 位整数底层实现。
  - `default = []`：保持默认值最小化。
  - `python`：启用 Python 绑定。
  - `extension-module`：构建一个 Python 扩展模块（始终包含 `python`）。
  - `ffi`：启用 C FFI 绑定。
  - `stubs`：暴露测试用的桩（stub）。

## 构建配置

为避免不必要的重新构建，请在相关目标之间对齐 Cargo 特性、profile 和标志。Cargo 会根据特性、
profile 和标志对构建产物进行索引。不匹配会产生独立的产物，并可能导致大量重新编译。

### 主要目标

Makefile 和已变更 crate 脚本是目标范围和特性的权威来源。不要把它们的特性列表复制到新文档中。

| 目标                   | 范围                              | 特性来源                    |
|--------------------------|------------------------------------|-----------------------------------|
| `make cargo-test`        | 工作区库和测试。     | Makefile 中的 `CARGO_FEATURES`。 |
| Pre-commit Clippy        | 已变更 crate 的库和测试。 | `scripts/clippy-changed.sh`。      |
| `make check-all-targets` | 整个工作区，包括示例。 | `CARGO_FEATURES` 加上 `examples`。 |

这些目标默认使用 `nextest` Cargo profile。当为相同的代码面添加新目标时，请匹配其 profile 和
特性，以便 Cargo 能够复用已编译的产物。

### 文档构建

文档使用 `make docs-rust` 单独构建，该命令运行：

```bash
cargo +nightly doc --all-features --no-deps --workspace
```

该目标使用 nightly 和 `--all-features`，因此它不会与测试和 lint 目标共享全部构建产物。

### 独立目标（Python 扩展构建）

| 目标        | Profile   | 特性来源          |
|---------------|-----------|-------------------------|
| `build`       | `release` | `_set_feature_flags()`。 |
| `build-debug` | `dev`     | `_set_feature_flags()`。 |

这两个目标都运行 `build.py`。Python 扩展构建需要 `extension-module`，因此它们使用不同的
特性集，并创建独立的 Cargo 产物。

### 应避免的重新构建触发因素

以下任何一种不匹配都会导致完全重新构建：

- 不同的特性组合（例如 `--features "a,b"` 与 `--features "a,c"`）。
- 不同的 `--no-default-features` 使用方式（启用/禁用默认特性）。
- 不同的 profile（例如 `dev` 与 `nextest` 与 `release`）。

当添加或更改一个构建目标时，如果该目标覆盖相同的代码，请与测试和 lint 分组保持匹配。

### 生成的 FFI 绑定与精度模式

当启用 `ffi` 特性时，`nautilus-model` 的构建脚本会重新生成
`nautilus_trader/core/includes/model.h` 和 `nautilus_trader/core/rust/model.pxd`。
这些文件编码了生成的 C/Cython 绑定是否使用高精度。已提交的生成文件使用高精度。
本地在编译带有 `ffi` 特性的 `nautilus-model` 的 cargo 命令，应当要么包含
`high-precision` 特性，要么避免重新生成这些文件。

使用 `BASE_FEATURES` 的 Make 目标（例如 `make build-debug-v2`）已经包含了
`high-precision`。漂移风险主要来自启用了 `ffi` 却没有对齐特性集的临时 cargo 命令。

对于不包含完整对齐特性集的窄范围检查，使用 Rust 特性。在命令中保留环境变量覆盖，
以防一个陈旧的 shell 值强制使用标准精度绑定：

```bash
env HIGH_PRECISION=true cargo check -p nautilus-model --features ffi,python,high-precision
```

在提交与 FFI 相关的工作之前，验证这些生成的文件没有发生漂移：

```bash
git diff -- nautilus_trader/core/includes/model.h nautilus_trader/core/rust/model.pxd
```

如果它们仅仅因为某个命令在未开启高精度的情况下运行而发生了变化，请使用
`HIGH_PRECISION=true` 重新运行该 cargo 命令。不要手动编辑生成的文件。

## 模块组织

- 保持模块专注于单一职责。
- 在定义子模块时，使用 `mod.rs` 作为模块根。
- 优先使用相对扁平的层级结构，而不是深层嵌套，以保持路径可管理。
- 使用尽可能窄的可见性范围。工作区禁止不可达的 `pub` 项。
- 在 crate 根中重新导出有意公开的 API。
- 除非某个窄范围的局部导入能实质性地提升清晰度，否则将导入语句保持在文件或模块顶部。
- 将私有函数和类型放在其调用方之下。在适配器模块中，将主要的客户端结构体和实现放在
  顶部，随后是私有的路由类型和函数。

## 代码风格与约定

### 文件头要求

所有手写的 Rust 文件都必须包含标准化的版权声明头。生成的文件例外，必须保留其生成器的
文件头。

```rust
// -------------------------------------------------------------------------------------------------
//  Copyright (C) 2015-2026 Nautech Systems Pty Ltd. All rights reserved.
//  https://nautechsystems.io
//
//  Licensed under the GNU Lesser General Public License Version 3.0 (the "License");
//  You may not use this file except in compliance with the License.
//  You may obtain a copy of the License at https://www.gnu.org/licenses/lgpl-3.0.en.html
//
//  Unless required by applicable law or agreed to in writing, software
//  distributed under the License is distributed on an "AS IS" BASIS,
//  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//  See the License for the specific language governing permissions and
//  limitations under the License.
// -------------------------------------------------------------------------------------------------
```

:::info[自动化强制执行]
`check_copyright_year.sh` pre-commit 钩子会验证版权声明头中包含当前年份。
:::

### 代码格式化

运行 `make format` 时，`rustfmt` 会格式化导入语句。它会将标准库、外部 crate 和本地导入分组，
并对每一组按字母顺序排序。

请遵循以下间距规则：

- 函数之间（包括测试函数）留一个空行。
- 每个文档注释（`///` 或 `//!`）上方留一个空行。
- 独立的 `if`、`match`、`for`、`while` 和 `loop` 表达式上方留一个空行。
- spawn 调用上方留一个空行。

当表达式是某个代码块的开头、延续上一行的操作、或带有附加的注释或属性时，控制流和 spawn
规则不适用。`check-formatting-rs` pre-commit 钩子会强制执行这些例外情况。

#### 字符串格式化

优先使用内联格式字符串，而不是位置参数：

```rust
anyhow::bail!("Failed to subtract {n} months from {datetime}");
```

避免使用位置参数：

```rust
anyhow::bail!("Failed to subtract {} months from {}", n, datetime);
```

这样能使消息更具可读性、更能自我说明，在存在多个变量时尤其如此。

### 类型限定

在代码中限定类型时遵循以下约定：

- **anyhow**：完全限定其宏和结果类型，例如 `anyhow::bail!` 和 `anyhow::Result<T>`。
- **Nautilus 领域类型**：导入 `Symbol`、`InstrumentId`、`Price` 等类型，然后不带 crate
  前缀地使用它们。
- **Tokio**：完全限定其类型和函数，例如 `tokio::spawn` 和 `tokio::time::timeout`。
- **std::fmt**：导入 `Debug` 和 `Display`，但完全限定 `std::fmt::Formatter` 和
  `std::fmt::Result`。在手写的 `Debug` 实现中使用 `debug_struct(stringify!(TypeName))`。
- **Nautilus 宏**：导入 `nautilus_actor!` 和 `nautilus_strategy!`，然后不带 crate
  前缀地调用它们。

```rust
use nautilus_model::identifiers::Symbol;

pub fn process_symbol(symbol: Symbol) -> anyhow::Result<()> {
    if !symbol.is_valid() {
        anyhow::bail!("Invalid symbol: {symbol}");
    }

    tokio::spawn(async move {
        // Process symbol asynchronously
    });

    Ok(())
}
```

:::info[自动化强制执行]
`check_anyhow_usage.sh` pre-commit 钩子会自动强制执行这些 anyhow 约定。
:::

### 日志记录

- 完全限定 `log` 宏，使后端一目了然，例如 `log::debug!` 和 `log::info!`。
- 消息以大写字母开头，且不以句号结尾。
- 连接生命周期、客户端生命周期、重连、对账以及批量状态汇总保持在 `INFO` 级别。
- 订阅细节、每笔订单确认、金融工具计数、身份验证以及 WebSocket 内部细节保持在
  `DEBUG` 级别。
- 除非是函数的第一行，否则在日志调用上方留一个空行。
- 不要在生产库代码中直接写入 stdout 或 stderr，也不要调用 `std::process::exit`。
  二进制文件、示例、基准测试、测试、适配器、CLI 和 testkit 属于例外，因为直接的进程
  控制是它们职责的一部分。

:::info[自动化强制执行]
`check_logging_conventions.sh` 钩子会强制执行宏限定、句尾无句号、直接输出以及
进程退出方面的规则。
:::

### 错误处理

在 API 边界处选择错误类型：

| 边界                              | 返回类型                         |
|---------------------------------------|-------------------------------------|
| 可复用的库或领域 API。       | 一个类型化的 `Result<T, E>`。             |
| 应用或适配器编排。 | `anyhow::Result<T>`。                |
| 公共输入校验。              | 适用时使用 `CorrectnessResult<T>`。 |

- 当调用方需要检查或从失败中恢复时，使用 `thiserror` 定义类型化的错误。
- 使用 `?` 进行错误传播。
- 将错误模式和闭包绑定为 `e`，而不是 `err` 或 `error`。
- 对于返回 `anyhow::Result` 的函数，优先使用 `anyhow::bail!` 进行提前返回：

  ```rust
  pub fn process_value(value: i32) -> anyhow::Result<i32> {
      if value < 0 {
          anyhow::bail!("Value cannot be negative: {value}");
      }

      Ok(value * 2)
  }
  ```

- 在需要一个错误值的地方（例如 `ok_or_else`），使用 `anyhow::anyhow!`。
- 不要在错误或断言中使用 `", got"`。根据上下文使用 `", was"`、`", received"` 或
  `", found"`。
- `.context()` 消息以小写文本开头，使链式错误读起来自然，除非消息以专有名词或缩写词开头：

  ```rust
  parse_timestamp(value).context("failed to parse timestamp")?;
  connect().context("BitMEX websocket did not become active")?;
  ```

:::info[自动化强制执行]
`check_error_conventions.sh` 钩子会强制执行错误变量命名规范。`check_anyhow_usage.sh`
钩子会强制执行限定导入以及在提前返回时使用 `anyhow::bail!`。
:::

### 异步模式

使用一致的异步模式：

- 使用不带 `async` 后缀的自然函数名。
- 完全限定 Tokio 的类型和函数。使用 `std::time::Duration`，而不是其 Tokio 重新导出版本。
- 让 Tokio 保持在同步核心 crate 依赖之外。`common` crate 将 Tokio 声明为可选依赖。
- 当取消操作可能留下部分工作或改变某个不变量时，为取消安全性编写文档。
- 当背压（back pressure）重要时，使用 `tokio_stream` 或 `futures::Stream`。
- 在网络和长时间运行的操作边界处应用超时。避免叠加冗余的超时。
- 遵循 [DST 确定性契约](../concepts/dst.md#determinism-contract)：将时钟、随机值、
  任务派生和网络访问路由通过项目的接缝（seam），并在 DST 路径上的 `tokio::select!`
  代码块中使用 `biased;`。

### 适配器运行时模式

`crates/adapters/` 下的适配器 crate 使用共享的运行时，以便来自 Python 线程的调用不依赖于
线程局部的 Tokio 上下文：

- **派生任务**：在生产环境的适配器代码中，使用 `get_runtime().spawn()` 而不是
  `tokio::spawn()`。

  ```rust
  use nautilus_common::live::get_runtime;

  get_runtime().spawn(async move {
      run_client().await;
  });
  ```

- **导入重新导出**：使用 `live::get_runtime`，而不是 `live::runtime::get_runtime`。

- **桥接同步代码**：当同步的适配器代码调用一个异步函数时，使用
  `get_runtime().block_on()`：

  ```rust
  fn sync_method(&self) -> anyhow::Result<()> {
      get_runtime().block_on(self.async_implementation())
  }
  ```

- **在首次使用前安装自定义运行时**：拥有 `main()` 的 Rust 原生二进制文件可以在
  `LiveNode::build()` 或任何适配器/客户端使用之前调用 `set_runtime()`。使用
  `tokio::runtime::Builder::new_multi_thread().enable_all()` 构建自定义运行时；
  当前线程运行时以及没有 I/O 或定时器驱动的运行时不满足适配器的假设条件。
  如果启用了 `python` 特性，请在构建运行时之前准备好 Python，或者保留默认的
  初始化器。

- **在测试中使用测试运行时**：`#[tokio::test]` 下的代码拥有自己的运行时上下文，
  因此 `tokio::spawn()` 能正常工作。该强制执行钩子会跳过测试文件和测试模块。

:::info[自动化强制执行]
`check_tokio_usage.sh` 钩子会强制执行适配器中共享运行时和导入限制的规则。
:::

### 属性模式

匹配附近类型所使用的 derive 顺序。将相关的 PyO3 和 stub 属性保持相邻，将运行时的 PyO3
路径与公共 stub 路径分开。

```rust
#[repr(C)]
#[derive(Clone, Copy, Hash, PartialEq, Eq, PartialOrd, Ord)]
#[cfg_attr(
    feature = "python",
    pyo3::pyclass(module = "nautilus_trader.core.nautilus_pyo3.model", from_py_object)
)]
#[cfg_attr(
    feature = "python",
    pyo3_stub_gen::derive::gen_stub_pyclass(module = "nautilus_trader.model")
)]
pub struct Symbol(Ustr);
```

对于具有大量 derive 属性的枚举：

```rust
#[repr(C)]
#[derive(
    Copy,
    Clone,
    Debug,
    Display,
    Hash,
    PartialEq,
    Eq,
    PartialOrd,
    Ord,
    AsRefStr,
    FromRepr,
    EnumIter,
    EnumString,
)]
#[strum(ascii_case_insensitive)]
#[strum(serialize_all = "SCREAMING_SNAKE_CASE")]
#[cfg_attr(
    feature = "python",
    pyo3::pyclass(
        frozen,
        eq,
        eq_int,
        module = "nautilus_trader.core.nautilus_pyo3.model.enums",
        from_py_object,
        rename_all = "SCREAMING_SNAKE_CASE",
    )
)]
#[cfg_attr(
    feature = "python",
    pyo3_stub_gen::derive::gen_stub_pyclass_enum(module = "nautilus_trader.model")
)]
pub enum AccountType {
    /// An account with unleveraged cash assets only.
    Cash = 1,
    /// An account which facilitates trading on margin, using account assets as collateral.
    Margin = 2,
}
```

### 类型桩注解

Python 类型桩（`.pyi` 文件）是使用
[pyo3-stub-gen](https://github.com/Jij-Inc/pyo3-stub-gen) 从 Rust 源代码生成的。
每一个暴露给 Python 的类型和函数都需要一个与之匹配的桩注解，以使生成的桩与绑定保持同步。

**注解类型：**

| PyO3 构造    | 桩注解                                  |
|-------------------|--------------------------------------------------|
| `#[pyclass]`      | `pyo3_stub_gen::derive::gen_stub_pyclass`        |
| 枚举 `#[pyclass]` | `pyo3_stub_gen::derive::gen_stub_pyclass_enum`   |
| `#[pymethods]`    | `pyo3_stub_gen::derive::gen_stub_pymethods`      |
| `#[pyfunction]`   | `pyo3_stub_gen::derive::gen_stub_pyfunction`     |

**放置规则：**

- 在结构体和枚举上，使用 `#[cfg_attr(feature = "python", ...)]`，并将该桩注解直接放在
  `pyo3::pyclass` 属性下方。
- 在 `#[pymethods]` impl 代码块上，将 `#[pyo3_stub_gen::derive::gen_stub_pymethods]`
  直接放在 `#[pymethods]` 下方。
- 在函数上，将桩注解放在 `#[pyfunction]` 正上方、任何文档注释之后。完全限定该路径，
  而不是导入它。

```rust
/// Converts a list of `Bar` into Arrow IPC bytes.
#[pyo3_stub_gen::derive::gen_stub_pyfunction(module = "nautilus_trader.serialization")]
#[pyfunction(name = "bars_to_arrow")]
pub fn py_bars_to_arrow(data: Vec<Bar>) -> PyResult<Py<PyBytes>> {
    // ...
}
```

```rust
#[pymethods]
#[pyo3_stub_gen::derive::gen_stub_pymethods]
impl AccountState {
    #[staticmethod]
    #[pyo3(name = "from_dict")]
    pub fn py_from_dict(values: &Bound<'_, PyDict>) -> PyResult<Self> {
        // ...
    }
}
```

**模块参数：** 设置 `module = "nautilus_trader.<package>"`，使其与 Python 导入该类型所在的
包相匹配。对于模型类型使用 `nautilus_trader.model`，对于序列化函数使用
`nautilus_trader.serialization`。

**Cargo.toml：** 将 `pyo3-stub-gen` 添加为可选依赖，并将其包含在 `python` 特性列表中：

```toml
[features]
python = ["pyo3", "pyo3-stub-gen"]

[dependencies]
pyo3-stub-gen = { workspace = true, optional = true }
```

### 生成的 Python 产物

v2 Python 界面提交了两类生成的产物：

- 位于 `python/nautilus_trader/**/*.pyi` 下的 Python 类型桩。
- 位于 `crates/**/src/python/**/*.rs` 下的 PyO3 包装器文档注释。

用一条命令运行生成器：

```bash
make py-stubs-v2
```

在更改任何暴露给 Python 的 Rust 界面之后运行它：`#[pyclass]`、`#[pymethods]`、
`#[pyfunction]`、桩注解、被包装的核心项上的文档注释，或适配器特性接线。将该目标
所改变的每一个生成的 `.pyi` 文件和包装器文档注释都与源代码变更一起提交。当已提交的
输出与重新生成的结果不一致时，CI 会失败。`make build-debug-v2` 也会重新生成这些产物，
但当你只需要桩和文档字符串时，使用 `make py-stubs-v2`。

`crates/**/src/python/**` 下的包装器 `///` 文档是由 `python/generate_docstrings.py`
根据核心 Rust 项的文档生成的。不要手动编辑它们。编辑核心文档，然后运行
`make py-stubs-v2`。该同步过程应用以下转换：

- `# Errors` 和 `# Safety` 部分会原样复制。
- `# Panics` 部分在到达 Python API 之前会被丢弃。
- 文档内链接会被剥离。
- 使用 `::` 书写的 Rust 路径会变成 Python 风格的 `.` 路径。

v2 目标使用 `python/pyproject.toml` 中通过 `required-version = "==0.11.28"` 固定的
uv 版本。如果你本地的 `uv` 版本不同，`make sync-v2`、`make py-stubs-v2` 和
`make build-debug-v2` 会在同步之前失败，并给出所需版本和更新命令。运行预检脚本打印出的
`uv self update --version ...` 命令，或在 `PATH` 中前置一个匹配的 `uv` 二进制文件。

桩生成必须编译与 wheel 构建所暴露的相同的可选 Python 界面。`python/generate_stubs.py`
在运行 cargo 之前会剥离 `extension-module`，因此仅由 wheel 构建中的 `extension-module`
启用的特性，必须在该脚本中显式追加。Interactive Brokers 就应用了这一规则，
追加了 `nautilus-interactive-brokers/gateway`，从而使 `DockerizedIBGateway` 和
`ContainerStatus` 保留在生成的桩中。

后处理器负责处理 `py_` 前缀剥离、`@property`/`@staticmethod`/`@classmethod` 装饰、
关键字转义、去重和 ruff 格式化。

### 构造函数模式

一致地使用 `new()` 与 `new_checked()` 约定：

```rust
/// Creates a new [`Symbol`] instance with correctness checking.
///
/// # Errors
///
/// Returns an error if `value` is not a valid string.
///
/// # Notes
///
/// PyO3 requires a `Result` type for proper error handling and stacktrace printing in Python.
pub fn new_checked<T: AsRef<str>>(value: T) -> CorrectnessResult<Self> {
    // Implementation
}

/// Creates a new [`Symbol`] instance.
///
/// # Panics
///
/// Panics if `value` is not a valid string.
pub fn new<T: AsRef<str>>(value: T) -> Self {
    Self::new_checked(value).expect_display(FAILED)
}
```

对于 `CorrectnessResult` 上的 `.expect_display()` 消息，始终使用 `FAILED` 常量，
并导入提供它的 trait：

```rust
use nautilus_core::correctness::{CorrectnessResult, CorrectnessResultExt, FAILED};
```

#### 用于多可选参数构造函数的流式构建器

以可选字段为主导的、拥有大型构造函数的类型（`instruments` 领域类型）也暴露一个流式的
`bon` 构建器，这样调用方只需设置他们需要的字段，而不必传入一长串 `None`。
将 `#[bon::bon]` 放在固有 impl 上，并添加一个委托给 `new_checked` 的构建器方法，
从而保持单一的、经过验证的构造路径：

```rust
#[bon::bon]
impl CryptoPerpetual {
    // new_checked / new as above

    /// Returns a fluent builder for a [`CryptoPerpetual`] instance.
    ///
    /// # Errors
    ///
    /// Returns an error if any input validation fails (see [`CryptoPerpetual::new_checked`]).
    #[builder(start_fn = builder, finish_fn = build)]
    pub fn build_checked(/* same parameters as new_checked */) -> CorrectnessResult<Self> {
        Self::new_checked(/* forward verbatim */)
    }
}
```

调用方写作 `CryptoPerpetual::builder().instrument_id(..)..build()?`。必需的
（非 `Option`）参数会由 bon 的类型状态（typestate）在编译期强制检查；`Option`
参数可以省略，并会像 `new_checked` 那样精确地应用其默认值。`build()` 返回与
`new_checked` 相同的 `CorrectnessResult`，因此每一项正确性检查仍然会运行。
与仅限测试使用的事件规范不同，这个构建器存在于生产类型上，随生产构建一同发布，
并返回一个 `Result` 而不是直接返回值。保留 `new()` 和 `new_checked()`；
这个构建器是附加性的。

### 类型转换模式

对字符串解析使用 `FromStr`，对可能失败的转换使用 `TryFrom`。只为不会失败的转换实现
`From`。

```rust
impl FromStr for Symbol {
    type Err = SymbolParseError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        // parsing logic
    }
}
```

一些现有的领域类型对无效输入实现了会 panic 的泛型 `From<T: AsRef<str>>`。请将这些视为
兼容性表面，不要把这种模式复制到新的 API 中。当输入尚未经过验证时，使用 `FromStr`、
`.parse()` 或 `TryFrom`。

### 领域数值类型

对离散的金融数量使用小数值：

| 值                                    | 类型及构造方式                                      |
|------------------------------------------|--------------------------------------------------------------|
| 价格或数量。                       | `Price::from_decimal_dp` 或 `Quantity::from_decimal_dp`。   |
| 资金、费用、保证金或余额。          | `Decimal`，然后使用 `Money::from_decimal` 或 `Money::zero`。    |
| 连续的信号比率或时序曲线。 | 当小数精度在该领域没有意义时使用 `f64`。        |

在测试中，将 `.as_decimal()` 与 `dec!(value)` 进行比较。不要将金融值转换为 `f64` 用于
断言。旧版的浮点数构造函数仍留存在代码库中，但在新代码中应使用小数路径。

### 常量与命名约定

对常量使用带有描述性名称的 SCREAMING_SNAKE_CASE：

```rust
/// Number of nanoseconds in one second.
pub const NANOSECONDS_IN_SECOND: u64 = 1_000_000_000;

/// Bar specification for 1-minute last price bars.
pub const BAR_SPEC_1_MINUTE_LAST: BarSpecification = BarSpecification {
    step: NonZero::new(1).unwrap(),
    aggregation: BarAggregation::Minute,
    price_type: PriceType::Last,
};
```

### 哈希集合

根据确定性、信任边界、性能和访问模式来选择哈希集合：

| 需求                                      | 集合                     |
|--------------------------------------------------|--------------------------------|
| 需要可观察的插入顺序迭代。            | `IndexMap` 或 `IndexSet`。      |
| 无需可观察迭代顺序的热路径查找。   | `AHashMap` 或 `AHashSet`。      |
| 不受信任的键或面向网络的边界。     | `HashMap` 或 `HashSet`。        |
| 外部 API 要求使用标准集合。     | `HashMap` 或 `HashSet`。        |
| 简单的、非关键性存储。                    | `HashMap` 或 `HashSet`。        |

#### 迭代顺序的确定性

`AHash` 会为每个进程随机化其哈希器，因此其迭代顺序在不同运行之间会有所不同。当迭代结果
会影响可观察状态时（包括发出的事件、返回的序列、随机数消耗或下游效应），使用
`IndexMap` 或 `IndexSet`。

```rust
use indexmap::{IndexMap, IndexSet};

let mut commissions: IndexMap<Currency, Money> = IndexMap::new();
let mut subscribed: IndexSet<InstrumentId> = IndexSet::new();
```

`check-dst-conventions` 钩子会在经过审计的 DST 路径上强制执行这一规则。请参照
[DST 确定性契约](../concepts/dst.md#determinism-contract) 审查其他位置。

#### 性能

对于查找密集的热路径、且迭代顺序不可观察时，使用 `AHashMap` 或 `AHashSet`：

```rust
use ahash::{AHashMap, AHashSet};

let mut symbols: AHashSet<Symbol> = AHashSet::new();
let mut prices: AHashMap<InstrumentId, Price> = AHashMap::new();
```

`AHashMap` 是非加密安全的。当不受信任的键使得抗哈希洪泛（hash-flooding）能力成为
安全边界的一部分时，不要使用它。

基准测试位于 `crates/core/benches/hash_map.rs`。在做出依赖硬件或工具链细节的断言之前，
请重新运行这些基准测试。

对于从 `IndexMap` 中删除元素：

- 当插入顺序必须保持稳定时，使用 `shift_remove`。
- 当顺序不再重要时，使用 `swap_remove`。

### 线程安全的哈希映射模式

`Arc<AHashMap<K, V>>` 支持共享读取，不支持修改。除非该值提供内部可变性，否则安全 Rust
会拒绝通过 `Arc` 进行修改。

| 访问模式                               | 集合                                                     |
|----------------------------------------------|------------------------------------------------------------------|
| 单线程读写。            | `AHashMap<K, V>`。                                              |
| 共享，构造后不可变。        | `Arc<AHashMap<K, V>>`。                                         |
| 共享读写，各自访问独立的键。 | `Arc<DashMap<K, V>>`。                                          |
| 存在跨键不变量的共享状态。      | `Arc<RwLock<AHashMap<K, V>>>` 或 `Arc<Mutex<AHashMap<K, V>>>`。 |

当某个操作必须原子性地更新多个条目时，选择 `RwLock` 或 `Mutex`。不要在 `.await`
点之间持有 `DashMap` 的守卫（guard）。

### 共享可变性存储

从 Cython 移植过来的代码经常在修改一个值之前将其从容器中克隆出来。这种模式会产生
静默的过期数据：一旦另一条代码路径对该条目应用了事件，本地克隆就会立即与规范条目发生
分歧。

只有当以下三个条件同时成立时，才使用 `Rc<RefCell<T>>`（单线程）或 `Arc<RwLock<T>>`
（多线程）存储：

- 该值在插入后会被修改。
- 多个持有者需要观察彼此的写入。
- 一个句柄必须超出容器的借用作用域而存续。

`Cache` 中的 orders 在内部使用这种形态来进行按键（per-key）的借用跟踪。存储类型为
`AHashMap<ClientOrderId, SharedCell<OrderAny>>`；这种智能指针的泄漏被保留在内部。
公共访问器返回隐藏了这一点的、作用域限定的新类型：`Cache::order` 返回
`OrderRef<'_>`（读借用），`Cache::order_mut` 返回 `OrderRefMut<'_>`（独占写借用，
需要 `&mut Cache`），而 `Cache::order_owned` 在某个值必须跨越边界时返回一个所拥有的
`OrderAny` 快照。当某个订单缺失属于错误情况时，使用 `Cache::try_order` 或
`Cache::try_order_owned`；它们返回 `OrderLookupError`，而不是强迫每个调用方构建
临时的"未找到"错误。引擎会在分发事件之前释放借用，并在事件之后重新读取缓存以获取
最新状态，从而使分发保持为一个干净的事务边界。

`Cache::order_mut` 需要 `&mut Cache`，这意味着接收到 `CacheView`（该视图只暴露不可变
缓存借用）的策略和适配器无法访问它。订单修改被保留给直接持有缓存的数据引擎和执行
引擎；类型系统强制执行这一契约。

在其他情况下，优先使用更简单的形态：

- 主要用于读取且只设置一次：`Rc<T>` 或 `Arc<T>`（无内部可变性）。
- 拥有的快照对调用方就足够了：存储 `T`，在读取时克隆。
- 单一所有者，不进行修改：普通字段。

在采用 `Rc<RefCell<T>>` 之前值得权衡的成本：

- 每次访问都要付出运行时借用检查的代价。
- 智能指针类型会在写入边界处泄漏。
- 误用会在运行时 panic，而不是编译失败。
- `Rc<RefCell<T>>` 是 `!Send`；跨线程存储需要 `Arc<RwLock<T>>`
  （或在读取很少时使用 `Arc<Mutex<T>>`）。

**决策树：**

1. 可变、多观察者，且句柄需要超出容器借用而存续？
   - 单线程：`Rc<RefCell<T>>`。
   - 多线程：`Arc<RwLock<T>>`（或在读取很少时使用 `Arc<Mutex<T>>`）。
2. 主要用于读取且只设置一次：`Rc<T>` 或 `Arc<T>`。
3. 拥有的快照就足够了：存储 `T`，在读取时克隆。
4. 单一所有者，不进行修改：普通字段。

### 重新导出模式

按字母顺序组织重新导出，并放在 lib.rs 文件的末尾：

```rust
pub use crate::{
    nanos::UnixNanos,
    time::AtomicTime,
    uuid::UUID4,
};
```

### 文档标准

文档注释使用陈述语气："Returns the account ID"（返回账户 ID），而不是
"Return the account ID"。

#### 章节标题大小写

Rustdoc 章节标题使用标题格式大小写（Title Case），与 Rust 标准库的约定一致：

- `# Examples`
- `# Errors`
- `# Panics`
- `# Safety`
- `# Notes`
- `# Thread Safety`
- `# Feature Flags`

#### 模块级文档

为公共模块以及契约不明显的模块添加模块级文档。不要为私有的叶子模块添加样板文档。

```rust
//! Functions for correctness checks similar to the *design by contract* philosophy.
//!
//! This module provides validation checking of function or method conditions.
//!
//! A condition is a predicate which must be true just prior to the execution of
//! some section of code - for correct behavior as per the design specification.
```

对于带有特性标志的模块，请清晰地记录它们：

```rust
//! # Feature flags
//!
//! This crate provides feature flags to control source code inclusion during compilation,
//! depending on the intended use case:
//!
//! - `ffi`: Enables the C foreign function interface (FFI) from [cbindgen](https://github.com/mozilla/cbindgen).
//! - `python`: Enables Python bindings from [PyO3](https://pyo3.rs).
//! - `extension-module`: Builds as a Python extension module (used with `python`).
//! - `stubs`: Enables type stubs for use in testing scenarios.
```

#### 字段文档

当相邻的字段都有文档时，为公共字段添加文档。使用句尾句号，并在该类型内部保持文档密度
一致。不要给私有字段添加文档注释；将重要的上下文放在类型级别的文档中。

```rust
pub struct Currency {
    /// The currency code as an alpha-3 string (e.g., "USD", "EUR").
    pub code: Ustr,
    /// The currency decimal precision.
    pub precision: u8,
    /// The ISO 4217 currency code.
    pub iso4217: u16,
    /// The full name of the currency.
    pub name: Ustr,
    /// The currency type, indicating its category (e.g. Fiat, Crypto).
    pub currency_type: CurrencyType,
}
```

#### 函数文档

为所有公共函数记录：

- 目的和行为。
- 当输入用法从类型和名称中不明显时，说明输入用法。
- 当函数返回 `Result` 时，说明错误情况。
- 当函数可能 panic 时，说明 panic 情况。

```rust
/// Returns a reference to the `AccountBalance` for the specified currency, or `None` if absent.
///
/// # Panics
///
/// Panics if `currency` is `None` and `self.base_currency` is `None`.
pub fn base_balance(&self, currency: Option<Currency>) -> Option<&AccountBalance> {
    // Implementation
}
```

#### 错误与 panic 文档格式

对于单行的错误和 panic 文档，使用句首字母大写格式：

```rust
/// Returns a reference to the `AccountBalance` for the specified currency, or `None` if absent.
///
/// # Errors
///
/// Returns an error if the currency conversion fails.
///
/// # Panics
///
/// Panics if `currency` is `None` and `self.base_currency` is `None`.
pub fn base_balance(&self, currency: Option<Currency>) -> anyhow::Result<Option<&AccountBalance>> {
    // Implementation
}
```

对于多行的错误和 panic 文档，使用带句尾句号的项目符号列表：

```rust
/// Calculates the unrealized profit and loss for the position.
///
/// # Errors
///
/// Returns an error if:
/// - The market price for the instrument cannot be found.
/// - The conversion rate calculation fails.
/// - Invalid position state is encountered.
///
/// # Panics
///
/// This function panics if:
/// - The instrument ID is invalid or uninitialized.
/// - Required market data is missing from the cache.
/// - Internal state consistency checks fail.
pub fn calculate_unrealized_pnl(&self, market_price: Price) -> anyhow::Result<Money> {
    // Implementation
}
```

#### 安全性文档格式

使用 `# Safety` 部分声明调用方对某个 unsafe 函数的义务。在每一个 unsafe 操作正上方
放置一条 `SAFETY:` 注释，解释该操作为何满足这些义务。

```rust
/// Creates a new instance from raw components without validation.
///
/// # Safety
///
/// The caller must ensure that all input parameters are valid and properly initialized.
pub unsafe fn from_raw_parts(ptr: *const u8, len: usize) -> Self {
    // SAFETY: The caller guarantees that `ptr` is valid for `len` bytes.
    let data = unsafe { std::slice::from_raw_parts(ptr, len) };
    Self { data }
}
```

## Python 绑定

Python 绑定使用 [PyO3](https://pyo3.rs)，使用户无需 Rust 工具链即可在 Python 中直接导入
NautilusTrader 的 crate。

### PyO3 命名约定

通过 PyO3 暴露 Rust 函数和类型时：

- 为 Rust 函数符号加上 `py_` 前缀。
- 使用 `#[pyo3(name = "...")]` 来发布不带前缀的 Python 函数名。
- 为面向 Python 的包装类型命名时加上 `Py` 前缀，并在发布给 Python 时去掉该前缀。
  为包装器所拥有的底层状态保留一个 `PyTypeInner` 名称；不要将其暴露为 pyclass。
- 公共适配器桩元数据使用 `nautilus_trader.adapters.<adapter_name>`。运行时模块路径
  可以使用 `nautilus_trader.core.nautilus_pyo3.<adapter_name>`。
- 使用 `nautilus_core::python` 中的 `to_pyvalue_err`、`to_pytype_err`、
  `to_pyruntime_err`、`to_pykey_err`、`to_pyexception` 和 `to_pynotimplemented_err`
  转换标准 Python 异常。

```rust
#[pyo3(name = "do_something")]
pub fn py_do_something() -> PyResult<()> {
    // ...
}
```

:::info[自动化强制执行]
`check_pyo3_conventions.sh` pre-commit 钩子会强制执行 PyO3 函数的 `py_` 前缀规则。
:::

### PyO3 枚举约定

暴露给 Python 的枚举应使用以下 `pyclass` 属性：

- `frozen`：枚举是不可变的值类型。
- `eq, eq_int`：允许与其他枚举实例以及整数判别值进行相等比较。
- `rename_all = "SCREAMING_SNAKE_CASE"`：统一 Python 变体名称的风格。
- `from_py_object`：允许从 Python 对象进行转换。

:::warning[不要在 `eq_int` 枚举上使用 `hash` pyclass 属性]
PyO3 自动生成的 `__hash__` 使用 Rust 的 `DefaultHasher`，其产生的值与 Python 对等效
整数使用 `hash()` 得到的值不同。由于 `eq_int` 使得 `MyEnum.VARIANT == 1` 成立，
哈希契约（`a == b` 意味着 `hash(a) == hash(b)`）会被违反。因此应改为提供一个手动的
`__hash__`，直接返回判别值：
:::

```rust
#[pymethods]
impl MyEnum {
    const fn __hash__(&self) -> isize {
        *self as isize
    }
}
```

### 测试约定

- 使用 `mod tests` 作为标准测试模块名称，除非你需要专门做隔离划分。
- 使用 `#[rstest]` 而不是 `#[test]`，即使是非参数化测试也是如此。
- 对非参数化的异步测试使用 `#[tokio::test]`。
- 在测试模块和仅测试用的文件上保留 `#[cfg(test)]`。不要将测试行为添加到生产代码中。
- 将 JSON 固定装置存储在 crate 的 `test_data/` 目录下，并使用 `include_str!` 加载它们。
- 通过 `.as_decimal()` 和 `dec!(value)` 比较价格、数量和资金。
- 不要使用 Arrange、Act、Assert 分隔注释。

:::info[自动化强制执行]
`check_testing_conventions.sh` pre-commit 钩子会强制执行使用 `#[rstest]` 而非
`#[test]` 的规则。
:::

#### 参数化测试

一致地使用 `rstest` 属性，对于参数化测试：

```rust
#[rstest]
#[case("AUDUSD", false)]
#[case("AUD/USD", false)]
#[case("CL.FUT", true)]
fn test_symbol_is_composite(#[case] input: &str, #[case] expected: bool) {
    let symbol = Symbol::new(input);
    assert_eq!(symbol.is_composite(), expected);
}
```

#### 测试规范（bon 构建器）

对于具有大量构造参数的事件，规范的测试构建器是一个与该事件定义在一起的流式规范，
位于 `events/<event>/spec/<name>.rs` 下（参考实现见
`crates/model/src/events/order/spec/filled.rs`）。使用
`#[cfg(any(test, feature = "stubs"))]` 对该规范模块进行门控，使其对 crate 内的测试
以及通过 `stubs` 特性按需选用的下游 crate 可用，但在生产构建中被编译剔除。规范不得
被生产代码引用。

为什么使用自定义规范而不是带有 `builder(default)` 的 `derive_builder::Builder`：
后者绕过了生产构造函数，因此后续添加的不变量不会被测试所执行。而规范则在每次
`build()` 时都通过生产构造函数进行漏斗式（funnel）传递。

结构：

- 使用 `finish_fn = into_spec` 派生 `bon::Builder`，使生成的完成方法不会与自定义的
  `build()` 冲突。
- 用字面量或 `TestDefault::test_default()` 调用，将每一个必需字段标记为
  `#[builder(default = ...)]`。将可选字段保留为不带默认值的 `Option<T>`，使调用方
  要么显式设置它们，要么接受 `None`。
- 将事件 ID 字段默认设置为 `crate::stubs` 中的 `test_uuid()`。这样能在无需调用方
  管理状态的情况下产生独立、可复现的 UUID。
- 在生成的构建器上实现 `build()`，使其调用 `into_spec()` 并通过生产构造函数
  （例如 `OrderFilled::new`）进行转发。返回类型是事件本身，而不是 `Result`，
  因为规范的默认值在构造时就是有效的。

调用方用法：

```rust
let fill = OrderFilledSpec::builder()
    .last_qty(Quantity::from(50_000))
    .trade_id(TradeId::from("TRADE-1"))
    .build();
```

只覆盖测试所关心的字段；其余字段使用规范默认值。不要在 `build()` 之后写
`.unwrap()`。

确定性：在 `cargo nextest` 下，每个测试都在一个全新的进程中运行，因此每线程的
UUID 序列会自动重置。在普通的 `cargo test` 下，如果某个测试需要跨多次抽取比较
UUID 序列，请在该测试开始时调用 `crate::stubs` 中的 `reset_test_uuid_rng()`。

在规范模块中用一个测试固定规范的默认值，使任何字段的意外漂移都能在此处暴露出来，
而不是在下游测试中表现为静默的行为变化。

#### 基于属性的测试

对基于属性的测试使用 `proptest` crate。将这些测试放在一个独立的 `property_tests`
模块中（而不是放在 `mod tests` 内部），以将确定性的单元测试与随机化的属性测试
分开：

```rust
#[cfg(test)]
mod property_tests {
    use proptest::prelude::*;
    use rstest::rstest;

    use super::*;

    fn my_strategy() -> impl Strategy<Value = MyType> {
        prop_oneof![
            Just(MyType::VariantA),
            Just(MyType::VariantB),
        ]
    }

    fn value_strategy() -> impl Strategy<Value = f64> {
        prop_oneof![
            -1000.0..1000.0,
            Just(0.0),
        ]
    }

    proptest! {
        #[rstest]
        fn prop_construction_roundtrip(
            value in value_strategy(),
            variant in my_strategy()
        ) {
            // The constructed value must preserve `value` and `variant`.
        }
    }
}
```

约定：

- 将模块命名为 `property_tests`，与 `mod tests` 分开。
- 导入 `proptest::prelude::*` 和 `rstest::rstest`。
- 定义返回 `impl Strategy<Value = T>` 的策略函数。
- 使用 `prop_oneof!` 将值范围与边界情况相结合。
- 使用 `prop_filter_map` 过滤无效的组合。
- 测试名称以 `prop_` 为前缀。
- 在 `proptest!` 内部为每个测试标记 `#[rstest]`。

#### 测试命名

使用能说明场景的具描述性测试名称：

```rust
fn test_sma_with_no_inputs()
fn test_sma_with_single_input()
fn test_symbol_is_composite()
```

### 方框式横幅注释

不要使用方框式的横幅或分隔注释。如果代码需要视觉上的分隔，考虑将其拆分为独立的模块或
文件。请改用：

- 能传达意图的清晰函数名。
- 用于逻辑分组的模块结构（`mod tests { mod fixtures { } }`）。
- 用于对相关方法进行分组的 impl 代码块。
- 用于语义化文档的文档注释（`///`）。
- IDE 的导航和代码折叠功能。

应避免的模式：

```rust
// ============================================================================
// Some Section
// ============================================================================

// ========== Test Fixtures ==========
```

## Rust-Python 内存管理

`Py<T>` 拥有对某个 Python 对象的引用。`Py::clone_ref` 和
`nautilus_core::python::clone_py_object` 会在与解释器绑定期间增加 Python 引用计数。
它们提供了安全的克隆，但不会打破引用循环。

通常不需要额外的 `Arc<Py<T>>`，因为 `Py<T>` 本身已经提供了共享所有权。去掉这个 `Arc`
能简化所有权，但如果该 Python 对象反过来引用了 Rust 所有者，循环仍然存在。

使用与该关系相匹配的所有权形态：

| 关系                                      | 模式                                          |
|---------------------------------------------------|--------------------------------------------------|
| Rust 拥有一个没有反向引用的 Python 对象。 | 存储 `Py<T>`，使用 `clone_py_object` 克隆。  |
| 一个 pyclass 拥有其他 Python 对象。              | 实现 `__traverse__` 和 `__clear__`。        |
| 该引用不能延长其目标的存活时间。     | 使用 Python 弱引用。                     |
| 所有权跨越线程。                        | 在进行 Python API 调用之前获取解释器。 |

对于 pyclass，`__traverse__` 必须访问每一个所拥有的 Python 引用，而 `__clear__`
必须清除可能参与循环的可变引用。不要在 `__traverse__` 中与解释器绑定；PyO3 在垃圾
回收器遍历对象期间禁止这样做。参见
[PyO3 垃圾回收器集成](https://pyo3.rs/v0.29.0/class/protocols.html#garbage-collector-integration)。

## 设计契约

设计契约（Design by contract）声明了一个函数与其调用方之间的义务：

- **前置条件（Preconditions）**：函数要求调用方满足的条件。
- **后置条件（Postconditions）**：函数作为回报所保证的内容。
- **不变量（Invariants）**：其类型在各次调用之间维持的属性。

优先使用类型系统。所有权、生命周期、`Send`/`Sync`、`Result`/`Option`、穷尽匹配、
新类型（newtype）和可见性在编译期以零运行时成本编码了大多数契约。仅在类型系统无法
表达的地方才使用运行时检查。

对于大多数前置条件，使用 `nautilus_core::correctness` 模块：它是该项目的设计契约机制，
应作为默认选择。`check_*` 函数（`check_predicate_true`、`check_valid_string_ascii`、
`check_positive_u64`、`check_in_range_inclusive_f64`、`check_equal_usize`、
`check_key_in_map` 等）返回一个类型化的 `CorrectnessResult<()>`，其 `CorrectnessError`
变体命名了每一种违规情况。将 `new_checked()`（可能失败，返回 `CorrectnessResult`）
与一个通过 `.expect_display(FAILED)` 在失败时 panic 的 `new()` 包装器配对，
用于经过验证的类型；这就是 [构造函数模式](#constructor-patterns) 约定，
它会产生以 `Condition failed: ...` 为前缀的 panic 消息。

对于正确性模块无法建模的*内部*不变量——字段关系、单调序列、CAS 后置条件、
编解码往返、可证明的范围内索引，以及依赖受信任的上游验证的内部辅助函数的前置条件——
使用 `debug_assert!`（以及 `debug_assert_eq!`/`_ne!`）。发布构建会剥离该检查，
因此绝不要将 `debug_assert!` 用于公共 API 的输入。对于 `unsafe` 代码，对健全性关键的
前置条件（空指针、对齐、来源合法性）使用始终启用的 `assert!`，并将 `debug_assert!`
保留给由设计保证成立的热路径前置条件。

选择使用哪种机制：

| 情况                                      | 使用                                               |
|------------------------------------------------|-----------------------------------------------------|
| 公共 API 前置条件。                       | `nautilus_core::correctness` 中的 `check_*`。      |
| 经过验证的构造函数。                         | `new_checked()` 和 `new()`。                      |
| 可恢复的解析、I/O 或网络错误。      | `Result<T, DomainError>`。                         |
| 编译器无法证明的内部不变量。  | `debug_assert!`。                                  |
| 始终启用的内部不变量。                  | `assert!`。                                        |
| 健全性关键的 unsafe 前置条件。        | `assert!`（始终启用）。                            |
| 由设计保证的热路径 unsafe 前置条件。 | `debug_assert!` 加上一份文档化的 `Safety` 说明。 |

风格：

- `debug_assert!` 消息以 `Invariant:` 为前缀，声明其应遵循的正向规则，而不是失败情况：
  `debug_assert!(next > last, "Invariant: time is strictly monotonic across CAS")`。
- `Condition failed: ...`（来自 `FAILED` 常量）表示调用方提供的输入违反了条件；
  `Invariant: ...` 表示内部契约存在 bug。
- 在最先假设某个不变量成立的位置放置断言。当某个不变量在一个热循环中始终成立时，
  在边界处断言一次即可，不要放在循环内部。

## 常见反模式

- 避免在热路径中使用 `.clone()`；优先使用借用或通过 `Arc` 实现的共享所有权。
- 避免在生产代码中使用 `.unwrap()`。应传播或映射可恢复的错误。当锁中毒（lock
  poisoning）代表不可恢复的程序状态时，解包（unwrap）是可以接受的。
- 当 `&str` 足够时避免使用 `String`，在热路径上尤其如此。
- 避免暴露内部可变性。将锁和 `RefCell` 隐藏在安全的 API 之后。
- 避免大型错误变体。当大型负载会显著增加 `Result<T, E>` 的大小时，用 Box 包装它们。

## Unsafe Rust

当 FFI 和底层存储需要安全 Rust 无法表达的契约时，NautilusTrader 会使用 `unsafe`。
每一个 unsafe 操作都将一项特定的证明义务从编译器转移到代码及其审查者身上。
参见 Rust 参考文档中的
[被视为未定义的行为](https://doc.rust-lang.org/stable/reference/behavior-considered-undefined.html)。

### 安全策略

任何使用 unsafe Rust 的地方都必须遵循以下策略：

- 为每一个 unsafe 函数提供一个 `# Safety` 部分，声明调用方的完整义务。
- 在每一个 unsafe 操作上方放置一条 `SAFETY:` 注释，解释为何其前置条件成立。
- 为 unsafe 代码周围的可观察行为添加有针对性的测试。测试支持但不能建立健全性证明。
- 每一个暴露 FFI 符号的 crate 都启用 `#![deny(unsafe_op_in_unsafe_fn)]`。
  即使在 `unsafe fn` 内部，每一次指针解引用或其他 unsafe 操作也必须包装在它自己的
  `unsafe { ... }` 代码块中。
- 对于跨越 FFI 边界的原始向量，遵循
  [FFI 内存契约](ffi.md)。外部代码成为该分配的所有者，必须恰好调用一次匹配的
  `vec_drop_*` 函数。

### unsafe 代码的分类

代码库出于以下目的使用 unsafe Rust：

- 操作原始指针的 FFI 边界。参见 [FFI](ffi.md)。
- 具有强制别名和生命周期不变量的 `UnsafeCell` 存储。
- 其完整可达状态满足 trait 契约的 unsafe `Send` 或 `Sync` 实现。

### unsafe Send/Sync 要求

`Send` 和 `Sync` 有不同的义务：

| Trait  | 安全代码可以                                              |
|--------|--------------------------------------------------------|
| `Send` | 将该值的所有权转移到另一个线程。             |
| `Sync` | 在线程之间共享 `&T`。                                |

一个 unsafe 实现必须使每一种被允许的安全用法都是健全的。这一证明必须覆盖所有可达状态、
别名、回调、泛型参数、安全方法、克隆和析构。文档无法将这一义务转移给安全代码的调用方。

如果移动或共享某个线程亲和（thread-affine）类型可能导致未定义行为，就不要为它实现
`Send` 或 `Sync`。可以让该类型保持非线程安全，将其状态替换为线程安全的原语，
或者暴露一个由通道或原子操作支撑的独立命令句柄。

### 纵深防御

在 unsafe 代码依赖某些不变量的地方，添加防御机制：

- 在转换类型之前进行验证，例如使用 `TypeId`。
- 使用 RAII 守卫（guard）在返回和 panic 路径上进行清理。
- 当失败会破坏健全性时，使用始终启用的检查。
- 仅将调试断言用作对由设计已经保证成立的不变量的诊断手段。它们无法强制执行健全性，
  因为发布构建会移除它们。

### 运行时不变量

若干核心子系统依赖运行时不变量，而不是编译期保证。下面前三项契约由测试进行验证。
守卫使用规则则通过约定强制执行。任何涉及 `UnsafeCell`、注册表、`unsendable` 或
实盘节点线程处理的 PR，都应确认不变量测试仍然通过。

#### 线程局部注册表

actor 注册表、组件注册表和消息总线各自使用 `thread_local!` 存储。在一个线程上注册的
对象在另一个线程上永远不可见。实盘节点的事件循环运行在单个线程上，所有的注册表和消息
总线访问都发生在该线程上。

`LiveNodeHandle` 是唯一预期的跨线程控制界面。它使用 `Arc<AtomicBool>` 来发出停止信号，
使用 `Arc<AtomicU8>` 来表示状态，两者都使用 `Ordering::Relaxed`。

#### actor 注册表与组件注册表

两个注册表都在线程局部的映射中存储 `Rc<UnsafeCell<dyn Trait>>`，但在如何处理别名访问
方面有所不同：

| 属性          | actor 注册表                     | 组件注册表                 |
|-------------------|------------------------------------|------------------------------------|
| 别名          | 允许（多个守卫）。 | 禁止（`BorrowGuard` + 集合）。   |
| 可重入访问 | 是，回调需要它。       | 否，生命周期操作是顺序执行的。  |
| 错误处理    | 查找失败时 panic 或返回 `None`。 | 出错时返回 `anyhow::Result`。 |
| 守卫类型          | `ActorRef<T>`（基于 Rc）。           | 栈局部的 `BorrowGuard`。         |

actor 注册表选择了可重入访问而不是防止别名，因为消息处理器经常需要回调注册表来查找
其他 actor。组件注册表可以强制执行严格的别名限制，因为生命周期操作
（启动、停止、重置、释放）是非可重入的。

#### `ActorRef` 使用规则

`ActorRef` 守卫必须：

- 在单个同步作用域内获取和释放。
- 绝不存储在结构体字段中。
- 绝不跨越 `.await` 点持有。
- 绝不发送到另一个线程。

规范的模式是在闭包中捕获某个 actor 的 `Ustr` ID，并在每次回调触发时查找该 actor：

```rust
let actor_id = actor.actor_id().inner();
let handler = TypedHandler::from(move |quote: &QuoteTick| {
    if let Some(mut actor) = try_get_actor_unchecked::<MyActor>(&actor_id) {
        actor.handle_quote(quote);
    }
});
```

## 工具配置

本仓库将标准 Rust 工具与项目专属的 pre-commit 检查结合在一起：

| 领域                       | 权威来源                                                 |
|----------------------------|-------------------------------------------------------------------|
| 格式化和导入。    | `rustfmt.toml`。                                                 |
| 工作区 lint。           | `Cargo.toml` 和 `clippy.toml`。                                 |
| Rust 布局约定。   | `.pre-commit-hooks/check_formatting_rs.sh`。                     |
| Nautilus 类型约定。 | `.pre-commit-hooks/check_nautilus_conventions.sh`。              |
| Tokio 和 DST 使用规范。       | `check_tokio_usage.sh` 和 `check_dst_conventions.sh` 钩子。    |
| PyO3 绑定。             | `.pre-commit-hooks/check_pyo3_conventions.sh`。                  |

每个工作区 crate 都通过 `[lints] workspace = true` 继承工作区的 lint。当抑制
`missing_panics_doc` 或 `missing_errors_doc` 时，包含一条 `reason`，解释该 lint
为何不适用：

```rust
#[allow(clippy::missing_panics_doc, reason = "mutex poisoning is not expected")]
```

使用 `cbindgen` 为 FFI 生成 C 头文件。不要直接编辑生成的头文件。

## Rust 版本管理

该项目通过 `rust-toolchain.toml` 固定了一个具体的 Rust 版本。

安装固定的工具链并验证当前激活的覆盖版本：

```bash
rustup toolchain install "$(bash scripts/rust-toolchain.sh)"
rustup show active-toolchain
```

如果 pre-commit 在本地通过但在 CI 中失败，请清除 prek 缓存并重新运行：

```bash
prek clean
make pre-commit
```

这些命令会恢复 CI 所使用的 Rust 和 Clippy 版本。

## Cap'n Proto 序列化

`nautilus-serialization` crate 提供可选的 Cap'n Proto 序列化功能。该特性保持可选启用，
以便标准构建不需要该编译器。

### 安装 Cap'n Proto

在处理模式（schema）之前安装 Cap'n Proto 编译器。所需版本在仓库根目录的 `tools.toml`
中指定。

参见 [环境搭建](environment_setup.md#capn-proto) 获取平台特定的说明。

:::warning
Ubuntu 默认的 `capnproto` 软件包版本过旧。Linux 用户必须从源码安装。
:::

验证安装：

```bash
capnp --version
```

该版本必须与 `tools.toml` 中的版本匹配。

### 模式开发工作流

模式文件位于 `crates/serialization/schemas/capnp/`：

- `common/`：基础类型、标识符和枚举。
- `commands/`：交易命令。
- `events/`：订单和持仓事件。
- `data/`：市场数据类型。

在修改模式时：

1. 编辑相应子目录中的 `.capnp` 模式文件。
2. 重新生成 Rust 绑定：

   ```bash
   make regen-capnp
   ```

3. 审查变更：

   ```bash
   git diff crates/serialization/generated/capnp
   ```

4. 如有需要，更新 `crates/serialization/src/capnp/conversions.rs` 中的转换逻辑。
5. 运行测试：

   ```bash
   make cargo-test EXTRA_FEATURES="capnp"
   ```

### 生成的代码

生成的 Rust 文件被提交到 `crates/serialization/generated/capnp/` 中，供 docs.rs
使用以及进行漂移审查。docs.rs 构建使用这些文件，因为其环境中没有 Cap'n Proto 编译器。

带有 `capnp` 特性的正常构建仍需要固定版本的编译器。`build.rs` 会将模式编译到
`OUT_DIR` 中；`make regen-capnp` 会将该输出复制到已提交的目录中。

### 验证模式一致性

在提交模式变更之前，确保生成的文件是最新的：

```bash
make check-capnp-schemas
```

该目标会：

1. 如果未安装 `capnp`，则跳过并给出警告，这在本地开发中是可以接受的。
2. 如果重新生成时出错（例如版本不匹配），则失败。
3. 重新生成模式，如果生成的文件与已提交的版本不同，则失败。

CI 会自动运行此检查以捕获漂移（CI 中始终已安装 capnp）。

### 使用 capnp 特性进行测试

- 运行工作区测试：`make cargo-test EXTRA_FEATURES="capnp"`。
- 运行序列化 crate 测试：`make cargo-test-crate-nautilus-serialization`。

### 模式演进指南

在演进模式时：

- **仅做累加性变更**：在末尾添加新字段。
- **绝不删除字段**：在注释中标记已废弃的字段。
- **绝不复用字段编号**：即使在废弃之后也不行。
- **测试往返兼容性**：确保新旧版本能够互操作。

Cap'n Proto 的演进规则允许在不破坏二进制兼容性的情况下更改模式，但你必须遵循这些约束
才能维持向前/向后兼容性。

## 参考资料

- [The Rustonomicon](https://doc.rust-lang.org/nomicon/)。
- [Rust Reference: Unsafety](https://doc.rust-lang.org/stable/reference/unsafety.html)。
- [Safe bindings in Rust](https://www.abubalay.com/blog/2020/08/22/safe-bindings-in-rust)。
- [Rust and C interoperability](https://www.chromium.org/Home/chromium-security/memory-safety/rust-and-c-interoperability/)。
