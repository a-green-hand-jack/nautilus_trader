# 环境搭建

对于开发环境，我们推荐使用 PyCharm *Professional* 版 IDE，因为它能够解析 Cython 语法。
你也可以使用带有 Cython 扩展的 Visual Studio Code 作为替代方案。

[uv](https://docs.astral.sh/uv) 是处理所有 Python 虚拟环境和依赖的首选工具。

[prek](https://github.com/j178/prek) 用于在提交（commit）时自动运行各种 pre-commit 检查、
自动格式化工具和代码检查（lint）工具。

NautilusTrader 越来越多地使用 [Rust](https://www.rust-lang.org)，因此你的系统上也应该安装 Rust
（参见[安装指南](https://www.rust-lang.org/tools/install)）。

序列化模式（schema）编译需要 [Cap'n Proto](https://capnproto.org/)。所需的版本在仓库根目录的
`tools.toml` 中指定。Ubuntu 默认提供的软件包版本通常过旧，因此你可能需要从源码安装（见下文）。

:::info
NautilusTrader *必须* 能在 **Linux、macOS 和 Windows** 上编译和运行。请始终牢记可移植性
（使用 `std::path::Path`，在 shell 脚本中避免使用 Bash 特有的语法等）。
:::

## 搭建步骤

以下步骤适用于类 UNIX 系统，且只需完成一次。

### 快速搭建

在新的 Linux 或 macOS 开发机器上，可以使用这条精简的搭建路径。下方的详细章节
会解释每一步骤并涵盖其他替代方案。

首先安装平台工具：

```bash tab="Ubuntu"
sudo apt-get update
sudo apt-get install -y build-essential clang lld curl git make pkg-config
```

```bash tab="macOS"
xcode-select --install
```

然后克隆仓库并安装固定版本的项目工具：

```bash
git clone --branch develop https://github.com/nautechsystems/nautilus_trader
cd nautilus_trader

curl https://sh.rustup.rs -sSf | sh
source "$HOME/.cargo/env"

curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

cargo install cargo-binstall --locked
make install-tools
./scripts/install-capnp.sh

uv sync --all-groups --all-extras
source .venv/bin/activate

export PYO3_PYTHON="$PWD/.venv/bin/python"

if [ "$(uname -s)" = "Linux" ]; then
  PYTHON_LIB_DIR="$("$PYO3_PYTHON" -c 'import sysconfig; print(sysconfig.get_config_var("LIBDIR"))')"
  export LD_LIBRARY_PATH="$PYTHON_LIB_DIR${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
fi

export PYTHONHOME="$("$PYO3_PYTHON" -c 'import sys; print(sys.base_prefix)')"

prek install
make build-debug
```

Windows 用户应遵循[安装指南](../getting_started/installation.md#from-source)中源码安装部分的步骤，
然后再使用本指南中对应的命令。

### 1. 安装依赖

按照[安装指南](../getting_started/installation.md)搭建项目，但对最后一条命令做出修改，
以安装开发和测试依赖：

```bash tab="uv"
uv sync --active --all-groups --all-extras
```

```bash tab="make"
make install
```

如果你正在频繁地开发和迭代，那么以 debug 模式编译通常就足够了，并且比完全优化的构建
*快得多*。要以 debug 模式安装，请使用：

```bash
make install-debug
```

### 2. 安装开发工具

NautilusTrader 会固定每一个开发工具的版本，以便所有贡献者和 CI 使用完全相同的版本。
一个 Makefile 目标就能安装整套工具：

```bash
make install-tools
```

这会安装：

- **Cargo 命令行工具**，版本固定于 `Cargo.toml` 的 `[workspace.metadata.tools]` 中：
  `cargo-audit`、`cargo-deny`、`cargo-edit`、`cargo-fuzz`、`cargo-llvm-cov`、`cargo-machete`、
  `cargo-nextest`、`cargo-vet`、`flamegraph`、`lychee`。
- **预编译二进制文件**，版本固定于 `tools.toml` 中：`prek`（pre-commit 运行器）和
  `osv-scanner`（漏洞扫描器）。
- **uv**，同步为 `pyproject.toml` 所要求的版本。

Cap'n Proto 的版本也固定于 `tools.toml`，但安装方式不同；参见下方的
[Cap'n Proto](#capn-proto) 一节。

模糊测试（fuzz）目标在运行时还需要 Rust nightly 工具链，因为 `cargo-fuzz` 使用了
`libfuzzer-sys` 以及不稳定的编译器标志：

```bash
rustup toolchain install nightly
```

#### 一次性前置条件：cargo-binstall

`make install-tools` 使用 [`cargo-binstall`](https://github.com/cargo-bins/cargo-binstall)
以预编译二进制文件的形式获取 `prek`，而不是从源码编译。请在每台机器上安装一次
`cargo-binstall`：

```bash
cargo install cargo-binstall --locked
```

这是一次性步骤。之后再运行 `make install-tools` 会复用已安装的 `cargo-binstall`。

#### 版本信息的唯一权威来源

仓库的清单文件（manifest）是依赖和工具版本的权威来源。除非没有基于清单文件读取版本的方式，
否则不要将当前版本号复制到文档、运行器镜像或脚本中。

| 源文件或配置段                              | 定义内容                                               |
|----------------------------------------------|-------------------------------------------------------|
| `rust-toolchain.toml`                        | Rust 工具链。                                       |
| `Cargo.toml` 和 `Cargo.lock`                 | Rust 工作区依赖及精确解析结果。     |
| `Cargo.toml` 中的 `[workspace.metadata.tools]` | 可通过 Cargo 安装的开发工具。                  |
| `pyproject.toml` 和 `python/pyproject.toml`  | Python 依赖、支持的 Python 版本范围以及 uv。  |
| `uv.lock` 和 `python/uv.lock`                | 精确的 Python 依赖解析结果。                  |
| `tools.toml`                                 | 没有原生清单文件的外部命令行工具和二进制文件。 |

`tools.toml` 中固定版本的外部工具包括 `prek`、`pip-audit`、`pypi-attestations`、
`osv-scanner` 和 `capnp`。

Makefile 通过 `scripts/cargo-tool-version.sh`、`scripts/tool-version.sh` 和
`scripts/uv-version.sh` 读取这些版本，因此只需在源文件中更新版本号即可，无需再改动其他地方。
要检查已固定的 cargo 工具版本与 crates.io 上的最新版本相比是否过期，请运行：

```bash
make outdated
```

### 3. 设置 pre-commit

设置 pre-commit 钩子，之后它会在每次提交时自动运行：

```bash
prek install
```

在提交 pull request 之前，请在本地运行格式化和代码检查（lint）套件，以确保 CI 能一次通过：

```bash
make format
make pre-commit
```

确保 Rust 编译器报告**零错误**——损坏的构建会拖慢所有人的进度。

### 4. 配置环境变量

**Rust/PyO3 必需（Linux 和 macOS）**：当在 Linux 或 macOS 上使用通过 `uv` 安装的 Python 时，
请在 `uv sync` 之后从仓库根目录设置以下环境变量：

```bash
# 为 PyO3 设置 Python 可执行文件路径
export PYO3_PYTHON="$PWD/.venv/bin/python"

# 仅限 Linux：为 uv 管理的 Python 运行时设置库路径
PYTHON_LIB_DIR="$("$PYO3_PYTHON" -c 'import sysconfig; print(sysconfig.get_config_var("LIBDIR"))')"
export LD_LIBRARY_PATH="$PYTHON_LIB_DIR${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

# 设置 Python home 路径（Rust 测试需要）
export PYTHONHOME="$("$PYO3_PYTHON" -c 'import sys; print(sys.base_prefix)')"
```

:::note
`LD_LIBRARY_PATH` 的导出仅限 Linux 需要，在 macOS 或 Windows 上不需要。

- `PYO3_PYTHON` 告诉 PyO3 使用哪个 Python 解释器，从而减少不必要的重新编译。
- 当使用 `uv` 安装的 Python 运行 `make cargo-test` 时，`PYTHONHOME` 是必需的。
  如果没有设置，依赖 PyO3 的测试可能无法定位 Python 运行时。

:::

要验证你的环境配置是否正确：

```bash
python -c "import sys; print('Python:', sys.executable, sys.version)"
echo "PYO3_PYTHON: $PYO3_PYTHON"
echo "PYTHONHOME: $PYTHONHOME"
```

## 依赖管理

Python 依赖由 [uv](https://docs.astral.sh/uv) 管理。`pyproject.toml` 中的 `[tool.uv]` 部分
强制实施了三项供应链安全设置：

- **`required-version`**：所有开发者和 CI 使用相同的 uv 版本。该版本由
  `scripts/uv-version.sh` 提取，供 Makefile、CI 和 Docker 构建使用。如果本地的 uv 版本
  与固定版本不一致，`uv lock`/`uv sync` 会失败并报错
  `Required uv version ... does not match the running version ...`。运行 `make update-uv`
  可安装固定版本（或按照 uv 自身给出的 `uv self update <version>` 提示操作）。v2 存根目标
  在运行 `uv` 之前会检查 `python/pyproject.toml` 中的固定版本；参见
  [生成的 Python 产物](rust.md#generated-python-artifacts)。
- **`exclude-newer = "3 days"`**：`uv lock` 会忽略最近 3 天内发布的软件包版本。
  这为社区留出了时间来发现并隔离被入侵的发布版本，避免其进入锁文件。该值接受
  RFC 3339 时间戳（`"2026-03-30T00:00:00Z"`）、易读格式的时长（`"3 days"`、`"1 week"`、
  `"24 hours"`），或 ISO 8601 时长格式（`"P3D"`、`"P1W"`、`"PT24H"`）。uv 0.11.8+ 会将
  易读/ISO 格式以 `exclude-newer-span` 的形式存储在 `uv.lock` 中，并同时写入一个哨兵式的
  `exclude-newer` 时间戳以保持向后兼容；本仓库中的两份锁文件都使用这种格式。
- **`no-build-package`**：显式列出 `uv.lock` 中锁定的每一个第三方软件包。`uv` 拒绝从源码
  构建列表中的任何软件包。在正常操作下 uv 优先使用预编译的 wheel，因此该设置通常不会
  触发；只有当列表中的某个软件包不再为目标平台发布 wheel 时才会触发，此时 `uv lock` 会
  直接失败，而不是悄悄地从 sdist 构建。本地工作区包有意不包含在该列表中，因为它必须由
  工作区自身的构建后端来构建。该列表通过 `scripts/check-no-build-packages.sh` 与
  `uv.lock` 保持同步，该脚本同时也作为 pre-commit 钩子，在 `uv.lock` 或 `pyproject.toml`
  发生变化时运行。

### 绕过冷却期

当需要立即引入某个安全补丁或关键 bug 修复时，可以在命令行上覆盖 `exclude-newer`。
所有形式都接受时间戳、易读格式的时长或 ISO 时长；针对单个包的覆盖还额外支持 `false`，
以完全豁免该包的冷却期。

```bash
# 缩短单个包的冷却期（易读格式的时长）
uv lock --exclude-newer-package "somepackage=1 day"

# 将单个包固定到一个绝对的截止时间
uv lock --exclude-newer-package "somepackage=2026-03-30T00:00:00Z"

# 完全豁免某个包的冷却期
uv lock --exclude-newer-package "somepackage=false"

# 对整个解析过程禁用冷却期
uv lock --exclude-newer "0 seconds"
```

命令行标志只会覆盖此次调用中的 `pyproject.toml` 配置值。后续运行的配置保持不变。

### 更新 uv

要更新固定的 uv 版本，请同时修改 `pyproject.toml` 和 `python/pyproject.toml` 中的
`required-version`，然后更新 `.pre-commit-config.yaml` 中的 `rev` 使其保持一致。
运行 `make update-uv` 在本地安装新固定的版本。

## 构建

对 `.rs`、`.pyx` 或 `.pxd` 文件做出任何更改之后，可以通过运行以下命令重新编译：

```bash tab="uv"
uv run --no-sync python build.py
```

```bash tab="make"
make build
```

如果你正在频繁地开发和迭代，那么以 debug 模式编译通常就足够了，并且比完全优化的构建
*快得多*。要以 debug 模式编译，请使用：

```bash
make build-debug
```

## Cap'n Proto

序列化模式（schema）编译需要 [Cap'n Proto](https://capnproto.org/)。
所需版本定义在仓库根目录的 `tools.toml` 中。

请为你的平台安装正确的版本：

```bash tab="Script (Linux/macOS)"
./scripts/install-capnp.sh
```

```bash tab="macOS (Homebrew)"
brew install capnp
```

```bash tab="Linux (source)"
CAPNP_VERSION=$(bash scripts/tool-version.sh capnp)
cd ~
wget https://capnproto.org/capnproto-c++-${CAPNP_VERSION}.tar.gz
tar xzf capnproto-c++-${CAPNP_VERSION}.tar.gz
cd capnproto-c++-${CAPNP_VERSION}
./configure
make -j$(nproc)
sudo make install
sudo ldconfig
```

```bash tab="Windows (Chocolatey)"
choco install capnproto
```

验证已安装的版本与 `tools.toml` 中的一致：

```bash
capnp --version
```

安装脚本会确保安装的是固定版本。如果 Homebrew 或 Chocolatey 提供的是较旧版本，
请从源码安装，或参见
[Cap'n Proto 安装指南](https://capnproto.org/install.html)。

## 更快的构建

cranelift 后端能大幅减少开发、测试和 IDE 检查时的构建时间。不过 cranelift 目前只在
nightly 工具链上可用，且需要额外的配置。安装 nightly 工具链：

```
rustup install nightly
rustup override set nightly
rustup component add rust-analyzer # 安装 nightly 版的 lsp
rustup override set stable # 重置回 stable
```

在工作区的 `Cargo.toml` 中为 dev 和 testing profile 启用 nightly 特性并使用 "cranelift" 后端。
你可以使用 `git apply <patch>` 应用下方的补丁，并在推送变更之前使用
`git apply -R <patch>` 撤销它。

:::warning
不要提交这些改动。cranelift 补丁仅用于本地开发，如果被推送会破坏 CI。
:::

```
diff --git a/Cargo.toml b/Cargo.toml
index 62b78cd8d0..beb0800211 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -1,3 +1,6 @@
+# This line needs to come before anything else in Cargo.toml
+cargo-features = ["codegen-backend"]
+
 [workspace]
 resolver = "2"
 members = [
@@ -140,6 +143,7 @@ lto = false
 panic = "unwind"
 incremental = true
 codegen-units = 256
+codegen-backend = "cranelift"

 [profile.test]
 opt-level = 0
@@ -150,11 +154,13 @@ strip = false
 lto = false
 incremental = true
 codegen-units = 256
+codegen-backend = "cranelift"

 [profile.nextest]
 inherits = "test"
 debug = false # Improves compile times
 strip = "debuginfo" # Improves compile times
+codegen-backend = "cranelift"

 [profile.release]
 opt-level = 3
```

在运行 `make build-debug` 之类的命令时传入 `RUSTUP_TOOLCHAIN=nightly`，并将其加入到所有
[rust analyzer 配置](#rust-analyzer-settings)中，以获得更快的构建和 IDE 检查速度。

## 服务

你可以使用位于 `.docker` 目录下的 `docker-compose.yml` 文件来引导 Nautilus 工作环境。
这将启动以下服务：

```bash
docker-compose up -d
```

如果你只想运行特定的服务（例如 `postgres`），可以用以下命令启动：

```bash
docker-compose up -d postgres
```

所用到的服务包括：

- `postgres`：Postgres 数据库，根用户为 `POSTGRES_USER`（默认为 `postgres`），
  `POSTGRES_PASSWORD` 默认为 `pass`，`POSTGRES_DB` 默认为 `postgres`。
- `redis`：Redis 服务器。
- `pgadmin`：用于数据库管理和维护的 PgAdmin4。

:::info
请仅将其用作开发环境。生产环境请使用正确且更安全的配置方式。
:::

服务启动后，你必须使用 `psql` 命令行工具登录，以创建 `nautilus` Postgres 数据库。
可以运行以下命令，并输入 docker 服务配置中的 `POSTGRES_PASSWORD`：

```bash
psql -h localhost -p 5432 -U postgres
```

以 `postgres` 管理员身份登录后，运行 `CREATE DATABASE` 命令，指定目标数据库名（这里使用 `nautilus`）：

```
psql (16.2, server 15.2 (Debian 15.2-1.pgdg110+1))
Type "help" for help.

postgres=# CREATE DATABASE nautilus;
CREATE DATABASE

```

## Nautilus CLI 开发者指南

## 简介

Nautilus CLI 是一个用于与 NautilusTrader 生态系统交互的命令行工具。
它提供了用于管理 PostgreSQL 数据库以及处理各种交易操作的命令。

:::warning
在使用 GNOME 桌面的 Linux 系统上，`nautilus` 命令通常指向 GNOME 文件管理器
（`/usr/bin/nautilus`）。安装 NautilusTrader CLI 之后，你可能需要通过以下方式之一
确保 Cargo 安装的二进制文件优先生效：

- 在 shell 配置中添加别名：`alias nautilus="$HOME/.cargo/bin/nautilus"`
- 使用完整路径：`~/.cargo/bin/nautilus`
- 确保 `~/.cargo/bin` 在 `PATH` 中出现在 `/usr/bin` 之前

:::

:::note
Nautilus CLI 命令仅在类 UNIX 系统上受支持。
:::

## 安装

你可以使用以下 Makefile 目标来安装 Nautilus CLI，其内部使用了 `cargo install`。
假设 Rust 的 `cargo` 已正确配置，这会将 nautilus 二进制文件放入系统的 PATH 中。

```bash
make install-cli
```

## 命令

你可以运行 `nautilus --help` 查看 CLI 的结构和可用的命令组：

### 数据库

这些命令用于引导 PostgreSQL 数据库。要使用它们，你需要提供正确的连接配置，
可以通过命令行参数，也可以通过位于根目录或当前工作目录下的 `.env` 文件提供。

- `--host` 或 `POSTGRES_HOST` 用于设置数据库主机
- `--port` 或 `POSTGRES_PORT` 用于设置数据库端口
- `--user` 或 `POSTGRES_USERNAME` 用于设置根管理员（通常是 postgres 用户）
- `--password` 或 `POSTGRES_PASSWORD` 用于设置根管理员的密码
- `--database` 或 `POSTGRES_DATABASE` 用于同时指定数据库**名称，以及对该数据库拥有权限
  的新用户**（例如，如果你提供 `nautilus` 作为值，将会创建一个名为 nautilus 的新用户，
  密码使用 `POSTGRES_PASSWORD` 中的值，并以该用户作为所有者引导创建 `nautilus` 数据库）。

`.env` 文件示例

```
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=pass
POSTGRES_DATABASE=nautilus
```

命令列表如下：

1. `nautilus database init`：会引导创建 schema、角色，以及位于 `schema` 根目录下的所有
   sql 文件（如 `tables.sql`）。
2. `nautilus database drop`：会删除目标 Postgres 数据库中的所有表、角色和数据。

## Rust analyzer 设置

Rust analyzer 是一个流行的 Rust 语言服务器，并有针对多种 IDE 的集成方案。建议配置
rust analyzer 使其具有与 `make build-debug` 相同的环境变量，以获得更快的编译速度。
下面提供了在 VSCode 和 Astro Nvim 中经过测试的配置。更多信息请参见
[PR](https://github.com/nautechsystems/nautilus_trader/pull/2524) 或 rust analyzer
[配置文档](https://rust-analyzer.github.io/book/configuration.html)。

```json tab="VSCode"
{
    "rust-analyzer.restartServerOnConfigChange": true,
    "rust-analyzer.linkedProjects": [
        "Cargo.toml"
    ],
    "rust-analyzer.cargo.features": "all",
    "rust-analyzer.check.workspace": false,
    "rust-analyzer.check.extraEnv": {
        "VIRTUAL_ENV": "<path-to-your-virtual-environment>/.venv",
        "CC": "clang",
        "CXX": "clang++"
    },
    "rust-analyzer.cargo.extraEnv": {
        "VIRTUAL_ENV": "<path-to-your-virtual-environment>/.venv",
        "CC": "clang",
        "CXX": "clang++"
    },
    "rust-analyzer.runnables.extraEnv": {
        "VIRTUAL_ENV": "<path-to-your-virtual-environment>/.venv",
        "CC": "clang",
        "CXX": "clang++"
    },
    "rust-analyzer.check.features": "all",
    "rust-analyzer.testExplorer": true
}
```

```lua tab="Neovim (AstroLSP)"
config = {
  rust_analyzer = {
    settings = {
      ["rust-analyzer"] = {
        restartServerOnConfigChange = true,
        linkedProjects = { "Cargo.toml" },
        cargo = {
          features = "all",
          extraEnv = {
            VIRTUAL_ENV = "<path-to-your-virtual-environment>/.venv",
            CC = "clang",
            CXX = "clang++",
          },
        },
        check = {
          workspace = false,
          command = "check",
          features = "all",
          extraEnv = {
            VIRTUAL_ENV = "<path-to-your-virtual-environment>/.venv",
            CC = "clang",
            CXX = "clang++",
          },
        },
        runnables = {
          extraEnv = {
            VIRTUAL_ENV = "<path-to-your-virtual-environment>/.venv",
            CC = "clang",
            CXX = "clang++",
          },
        },
        testExplorer = true,
      },
    },
  },
}
```
