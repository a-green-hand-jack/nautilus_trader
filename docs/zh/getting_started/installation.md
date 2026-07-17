# 安装

NautilusTrader 正式支持 Python 3.12-3.14，适用于以下 64 位平台：

| 操作系统               | 支持版本            | CPU 架构           |
|------------------------|--------------------|-------------------|
| Linux (Ubuntu)         | 22.04 及更高版本    | x86_64            |
| Linux (Ubuntu)         | 22.04 及更高版本    | ARM64             |
| macOS                  | 15.0 及更高版本     | ARM64             |
| Windows Server         | 2022 及更高版本     | x86_64            |

:::note
NautilusTrader 可能也能在其他平台上运行，但只有上表列出的平台是开发者常规使用并在 CI 中测试的。
:::

持续的 CI 覆盖来自我们构建所用的 GitHub Actions runner：

- `Linux (Ubuntu)` 构建目前固定使用 `ubuntu-22.04`，以便在 `ubuntu-latest` 持续更新的
  同时保持 glibc 2.35 的兼容性。
- `macOS (ARM64)` 构建运行在 `macos-latest` 上，因此支持范围会随该 runner 镜像的更新而变化。
- `Windows (x86_64)` 构建目前固定使用 `windows-2022`，以保持工具链稳定。

在 Linux 上，请先使用 `ldd --version` 确认 glibc 版本，并确保其报告的版本号为 2.35
或更高，然后再继续操作。

我们建议使用最新的受支持 Python 版本，并在虚拟环境中安装
[nautilus_trader](https://pypi.org/project/nautilus_trader/) 以隔离依赖。

**目前支持两种安装方式**：

1. 从 PyPI *或* Nautech Systems 包索引安装预构建的二进制 wheel。
2. 从源代码构建。

:::tip
我们强烈建议使用 [uv](https://docs.astral.sh/uv) 包管理器搭配“原生（vanilla）” CPython 进行安装。

Conda 及其他 Python 发行版*可能*可以运行，但官方并不支持。
:::

## 从 PyPI 安装

从 PyPI 安装最新的 [nautilus_trader](https://pypi.org/project/nautilus_trader/) 二进制 wheel（或 sdist 包）：

```bash
uv pip install nautilus_trader
```

### Python v2 发布候选 wheel

Python v2 是位于 `python/` 目录下的 Rust + PyO3 软件包。在最终 v2 验证完成之前，
发布候选 wheel 会以 `2.0.0rcN` 版本号发布到 PyPI。

```bash
uv pip install --pre nautilus_trader
```

由于这些 wheel 是预发布构建，因此需要加上 `--pre` 标志。安装后的导入名称
仍然是 `nautilus_trader`。

请在 NautilusTrader 源代码检出目录之外运行此命令。仓库根目录使用了
`exclude-newer` uv 策略以确保开发的可复现性，这可能会过滤掉刚发布的 v2 wheel。
在源代码检出目录内，请改用[从源代码构建 Python v2](#8-build-python-v2-from-source)。

当前 v2 wheel 面向 Python 3.12-3.14。当你需要本地的 Rust 修改、调试构建，
或者所需的平台 wheel 尚不可用时，请从源代码构建。

## Extras

针对特定集成，可以安装可选依赖作为“extras”：

- `betfair`：Betfair 适配器（集成）依赖。
- `docker`：在配合 Interactive Brokers 适配器使用 IB 网关（gateway）时，用于 Docker 相关依赖。
- `ib`：Interactive Brokers 适配器（集成）依赖。
- `polymarket`：Polymarket 适配器（集成）依赖。
- `visualization`：基于 Plotly 的交互式绩效报告（tearsheet）和图表。

安装带有指定 extras 的包：

```bash
uv pip install "nautilus_trader[docker,ib]"
```

## 从 Nautech Systems 包索引安装

Nautech Systems 包索引（`packages.nautechsystems.io`）遵循 [PEP-503](https://peps.python.org/pep-0503/) 标准，
托管了 `nautilus_trader` 的稳定版和开发版二进制 wheel。
这使用户可以安装最新的稳定发布版本，也可以安装预发布版本用于测试。

### 稳定版 wheel

稳定版 wheel 对应 `nautilus_trader` 在 PyPI 上的正式发布版本，使用标准版本号规则。

安装最新的稳定发布版本：

```bash
uv pip install nautilus_trader --index-url=https://packages.nautechsystems.io/simple
```

:::tip
如果希望 uv 在需要时自动回退到 PyPI，请使用 `--extra-index-url` 而非 `--index-url`。
:::

### 开发版 wheel

开发版 wheel 分别从 `nightly` 和 `develop` 分支发布，让用户可以在稳定版发布前
提前测试新特性和修复。

这一流程也有助于节省计算资源，并方便获取 CI 流水线中经过测试的精确二进制文件，
同时遵循 [PEP-440](https://peps.python.org/pep-0440/) 版本号规范：

- `develop` wheel 使用版本格式 `dev{date}+{build_number}`（例如 `1.208.0.dev20241212+7001`）。
- `nightly` wheel 使用版本格式 `a{date}`（alpha 版）（例如 `1.208.0a20241212`）。

| 平台               | Nightly | Develop |
| :----------------- | :------ | :------ |
| `Linux (x86_64)`   | ✓       | ✓       |
| `Linux (ARM64)`    | ✓       | -       |
| `macOS (ARM64)`    | ✓       | -       |
| `Windows (x86_64)` | ✓       | -       |

**注意**：来自 `develop` 分支的开发版 wheel 仅针对 Linux x86_64 发布。
Windows、macOS 和 Linux ARM64 的构建在 nightly 计划任务中运行，以保持 CI 反馈速度。

:::warning
我们不建议在生产环境（例如涉及真实资金的实盘交易）中使用开发版 wheel。
:::

### 安装命令

默认情况下，uv 会安装最新的稳定发布版本。添加 `--pre` 标志后，uv 会考虑
预发布版本（包括开发版 wheel）。

安装当前可用的最新预发布版本（包括开发版 wheel）：

```bash
uv pip install nautilus_trader --pre --index-url=https://packages.nautechsystems.io/simple
```

安装特定的开发版 wheel（例如 2025 年 9 月 12 日的 `1.221.0a20250912`）：

```bash
uv pip install nautilus_trader==1.221.0a20250912 --index-url=https://packages.nautechsystems.io/simple
```

### Python v2 分支开发版 wheel

面向 v2 Rust + PyO3 软件包的分支开发版 wheel，会发布到与 `develop` 和 `nightly`
不同的独立 v2 索引：

```bash
uv pip install --pre --index-url=https://packages.nautechsystems.io/v2/simple/ nautilus-trader
```

安装后的导入名称仍然是 `nautilus_trader`。请在 NautilusTrader 源代码检出目录之外
运行此命令，以避免仓库的 `exclude-newer` uv 策略过滤掉刚发布的 v2 wheel。
当你需要本地的 Rust 修改、调试构建，或者所需的平台 wheel 尚不可用时，请从源代码构建。

### 可用版本

你可以在[包索引](https://packages.nautechsystems.io/simple/nautilus-trader/index.html)上
查看 `nautilus_trader` 的所有可用版本。

以编程方式请求并列出可用版本：

```bash
curl -s https://packages.nautechsystems.io/simple/nautilus-trader/index.html | grep -oP '(?<=<a href=")[^"]+(?=")' | awk -F'#' '{print $1}' | sort
```

### 分支更新

- `develop` 分支 wheel（`.dev`）：每次合并提交都会持续构建并发布。
- `nightly` 分支 wheel（`a`）：当我们在 **UTC 时间 14:00** 自动将 `develop` 分支合并时
  （若有变更），每日构建并发布。

### 保留策略

- `develop` 分支 wheel（`.dev`）：我们仅保留最近一次的构建。
- `nightly` 分支 wheel（`a`）：我们仅保留最近的 30 次构建。

### 验证构建溯源

CI/CD 流水线为所有发布的制品生成加密认证：

- Python wheel 和源代码分发包（PyPI、GitHub Releases、Nautech Systems 包索引）：
  [SLSA](https://slsa.dev/) 构建溯源。
- Docker 镜像（`ghcr.io/nautechsystems/nautilus_trader`、`ghcr.io/nautechsystems/jupyterlab`）：
  无密钥的 [cosign](https://github.com/sigstore/cosign) 签名以及 SPDX SBOM 认证。

两者均通过 [Sigstore](https://www.sigstore.dev/) 签发，并绑定到特定的提交 SHA，
因此验证可以确保制品由官方 NautilusTrader GitHub Actions 工作流生成，且自生成以来未被篡改。

具体的验证命令请参见 `SECURITY.md` 中的
[验证发布版本](https://github.com/nautechsystems/nautilus_trader/blob/develop/SECURITY.md#verifying-releases)。

:::note
验证 Python 制品需要 [GitHub CLI](https://cli.github.com/)（`gh`），
验证 Docker 镜像需要 [cosign](https://github.com/sigstore/cosign)。
来自 `develop` 和 `nightly` 分支的开发版 wheel 同样经过认证。
:::

## 从源代码安装

如果你先按照 `pyproject.toml` 中指定的方式安装构建依赖，就可以使用 pip 从源代码安装。

### 1. 安装 rustup

安装 [rustup](https://rustup.rs/)（Rust 工具链安装器）：

```bash tab="Linux/macOS"
curl https://sh.rustup.rs -sSf | sh
```

```powershell tab="Windows"
# 从 https://win.rustup.rs/x86_64 下载并安装 rustup-init.exe
# 同时通过 Build Tools for Visual Studio 2022 安装“使用 C++ 的桌面开发”
```

验证：`rustc --version`

### 2. 启用 cargo

在当前 shell 中启用 `cargo`：

```bash tab="Linux/macOS"
source $HOME/.cargo/env
```

```powershell tab="Windows"
# 启动一个新的 PowerShell 会话
```

### 3. 安装 clang

安装 [clang](https://clang.llvm.org/)（LLVM 的 C 语言前端）。在 Linux 上，此操作
还会安装 [lld](https://lld.llvm.org/)，并将其配置为 Rust 链接器以加快构建速度：

```bash tab="Linux"
sudo apt-get install clang lld
```

```powershell tab="Windows"
# 1. 通过 Visual Studio Installer 添加 Clang：
#    修改 > C++ Clang tools for Windows (latest) > 修改
# 2. 添加到 PATH：
[System.Environment]::SetEnvironmentVariable('path', "C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Tools\Llvm\x64\bin\;" + $env:Path,"User")
```

验证：`clang --version`

### 4. 安装 uv

安装 [uv](https://docs.astral.sh/uv/getting-started/installation)：

```bash tab="Linux/macOS"
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```powershell tab="Windows"
irm https://astral.sh/uv/install.ps1 | iex
```

### 5. 克隆并安装

使用 `git` 克隆源代码，并在项目根目录下安装：

```bash
git clone --branch develop --depth 1 https://github.com/nautechsystems/nautilus_trader
cd nautilus_trader
uv sync --all-extras
```

对于开发主机和 CI runner 镜像，在安装固定版本工具之前，请参见
[版本的单一可信来源](../developer_guide/environment_setup.md#single-source-of-truth-for-versions)。

:::note
`--depth 1` 标志仅获取最新提交，实现更快、更轻量的克隆。
:::

### 6. 安装 Cap'n Proto（用于开发）

如果你计划启用 `capnp` Rust 特性、重新生成序列化 schema，或从事序列化相关代码的
开发工作，请安装 [Cap'n Proto](https://capnproto.org/)。可以在 Linux 或 macOS 上
使用仓库脚本安装 `tools.toml` 中指定的固定版本：

```bash
./scripts/install-capnp.sh
```

验证：`capnp --version`

:::note
Cap'n Proto 属于开发依赖，安装预构建 wheel 时不需要。
:::

### 7. 设置环境变量

为 PyO3 编译设置环境变量（仅限 Linux 和 macOS）。在 `uv sync` 之后，从仓库根目录
运行以下命令：

```bash
# 为 PyO3 设置 Python 可执行文件路径
export PYO3_PYTHON="$PWD/.venv/bin/python"

# 仅限 Linux：为 uv 管理的 Python 运行时设置库路径
PYTHON_LIB_DIR="$("$PYO3_PYTHON" -c 'import sysconfig; print(sysconfig.get_config_var("LIBDIR"))')"
export LD_LIBRARY_PATH="$PYTHON_LIB_DIR${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

# 使用 uv 安装的 Python 运行 Rust 测试时需要设置
export PYTHONHOME="$("$PYO3_PYTHON" -c 'import sys; print(sys.base_prefix)')"
```

:::note
`LD_LIBRARY_PATH` 的设置仅针对 Linux，macOS 不需要。

使用 `uv` 安装的 Python 运行 `make cargo-test` 时需要设置 `PYTHONHOME` 变量。
若未设置，依赖 PyO3 的测试可能无法定位到 Python 运行时。
:::

### 8. 从源代码构建 Python v2

此路径会从 `python/` 目录构建 PyO3 软件包，并将其安装到 `python/.venv` 中。
适用于在 NautilusTrader 源代码检出目录中、你所在平台没有可用的 v2 开发版 wheel，
或需要本地 Rust 修改的场景。

在仓库根目录下：

```bash
make build-debug-v2
```

该目标会同步 `python/.venv`，使用 maturin 构建 Rust 扩展，并重新生成 Python
类型 stub。它使用 `target-v2/` 存放 Cargo 构建产物，因此不会与 `target/` 中的
传统 Python 构建产生冲突。

在 v2 环境中运行一个 Python v2 示例：

```bash
cd python
.venv/bin/python examples/lighter/data_tester.py --lighter-environment testnet
```

有关直接命令和测试目标，请参见 [Python v2 README][python-v2-readme]。

[python-v2-readme]: https://github.com/nautechsystems/nautilus_trader/blob/develop/python/README.md

## 从 GitHub Release 安装

要从 GitHub 安装二进制 wheel，请先前往
[最新发布版本](https://github.com/nautechsystems/nautilus_trader/releases/latest)。
下载与你的操作系统和 Python 版本相匹配的 `.whl` 文件，然后运行：

```bash
uv pip install <file-name>.whl
```

## 版本控制与发布

NautilusTrader 仍处于积极开发阶段。部分功能可能尚不完整，尽管 API 正在
变得更加稳定，但版本之间仍可能出现破坏性变更。
我们会尽力在发布说明中以**尽力而为**的方式记录这些变更。

我们的目标是遵循**每两周一次的发布节奏**，但实验性功能或较大的功能可能会导致延迟。

只有在你准备好应对这些变化的情况下，才应使用 NautilusTrader。

## Redis

在 NautilusTrader 中使用 [Redis](https://redis.io) 是**可选的**，仅当将其配置为
缓存数据库或[消息总线](../concepts/message_bus.md)的后端时才需要。

:::info
支持的最低 Redis 版本为 6.2（[streams](https://redis.io/docs/latest/develop/data-types/streams/)
功能需要该版本）。
:::

若要快速搭建，我们建议使用 [Redis Docker 容器](https://hub.docker.com/_/redis/)。
你可以在 `.docker` 目录中找到示例配置，或运行以下命令启动一个容器：

```bash
docker run -d --name redis -p 6379:6379 redis:latest
```

该命令将会：

- 如果尚未下载，则从 Docker Hub 拉取最新版本的 Redis。
- 以分离模式（`-d`）运行容器。
- 将容器命名为 `redis`，便于引用。
- 在默认端口 6379 上暴露 Redis，使其可在你的机器上被 NautilusTrader 访问。

管理 Redis 容器：

- 使用 `docker start redis` 启动它。
- 使用 `docker stop redis` 停止它。

:::tip
我们建议使用 [Redis Insight](https://redis.io/insight/) 作为图形界面工具，
以便高效地可视化和调试 Redis 数据。
:::

## 精度模式

NautilusTrader 为其核心值类型（`Price`、`Quantity`、`Money`）支持两种精度模式，
二者在内部位宽和最大小数精度上有所不同。

- **高精度**：128 位整数，最多支持 16 位小数精度，取值范围更大。
- **标准精度**：64 位整数，最多支持 9 位小数精度，取值范围较小。

:::note
默认情况下，官方 Python wheel 在 Linux 和 macOS 上以高精度（128 位）模式发布。
在 Windows 上，仅提供标准精度（64 位）的 Python wheel，因为 MSVC 的 C/C++ 前端
不支持 `__int128`，导致 Cython/FFI 层无法处理 128 位整数。

对于纯 Rust crate，由于 Rust 通过软件模拟支持 `i128`/`u128`，高精度模式可在所有平台
（包括 Windows）上使用。除非显式启用 `high-precision` 特性标志，否则默认使用标准精度。
:::

性能上的权衡在于：标准精度在典型回测中大约快 3%-5%，但小数精度较低，
可表示的取值范围也更小。

:::note
两种模式的性能基准测试对比正在进行中，尚未发布。
:::

### 构建配置

精度模式由以下方式决定：

- 编译期间设置 `HIGH_PRECISION` 环境变量，**和/或**
- 显式启用 `high-precision` Rust 特性标志。

```bash tab="High-precision (128-bit)"
export HIGH_PRECISION=true
make install-debug
```

```bash tab="Standard-precision (64-bit)"
export HIGH_PRECISION=false
make install-debug
```

### Rust 特性标志

要在 Rust 中启用高精度（128 位）模式，请在 `Cargo.toml` 中添加 `high-precision` 特性：

```toml
[dependencies]
nautilus_core = { version = "*", features = ["high-precision"] }
```

:::info
更多详情请参见[值类型（Value Types）](../concepts/overview.md#value-types)相关说明。
:::
</content>
