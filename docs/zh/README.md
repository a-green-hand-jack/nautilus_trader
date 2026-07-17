# <img src="https://github.com/nautechsystems/nautilus_trader/raw/develop/assets/nautilus-trader-logo.png" width="500">

[![codecov](https://codecov.io/gh/nautechsystems/nautilus_trader/branch/master/graph/badge.svg?token=DXO9QQI40H)](https://codecov.io/gh/nautechsystems/nautilus_trader)
[![codspeed](https://img.shields.io/endpoint?url=https://codspeed.io/badge.json)](https://codspeed.io/nautechsystems/nautilus_trader)
![pythons](https://img.shields.io/pypi/pyversions/nautilus_trader)
![pypi-version](https://img.shields.io/pypi/v/nautilus_trader)
![pypi-format](https://img.shields.io/pypi/format/nautilus_trader?color=blue)
[![Downloads](https://img.shields.io/pepy/dt/nautilus-trader?color=blue)](https://pepy.tech/projects/nautilus-trader)
[![Discord](https://img.shields.io/badge/Discord-%235865F2.svg?logo=discord&logoColor=white)](https://discord.gg/NautilusTrader)

| 分支      | 版本                                                                                                                                                                                                                     | 状态                                                                                                                                                                                            |
| :-------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `master`  | [![version](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Fnautechsystems%2Fnautilus_trader%2Fmaster%2Fversion.json)](https://packages.nautechsystems.io/simple/nautilus-trader/index.html)  | [![build](https://github.com/nautechsystems/nautilus_trader/actions/workflows/build.yml/badge.svg?branch=master)](https://github.com/nautechsystems/nautilus_trader/actions/workflows/build.yml)  |
| `nightly` | [![version](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Fnautechsystems%2Fnautilus_trader%2Fnightly%2Fversion.json)](https://packages.nautechsystems.io/simple/nautilus-trader/index.html) | [![build](https://github.com/nautechsystems/nautilus_trader/actions/workflows/build.yml/badge.svg?branch=nightly)](https://github.com/nautechsystems/nautilus_trader/actions/workflows/build.yml) |
| `develop` | [![version](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Fnautechsystems%2Fnautilus_trader%2Fdevelop%2Fversion.json)](https://packages.nautechsystems.io/simple/nautilus-trader/index.html) | [![build](https://github.com/nautechsystems/nautilus_trader/actions/workflows/build.yml/badge.svg?branch=develop)](https://github.com/nautechsystems/nautilus_trader/actions/workflows/build.yml) |

| 平台               | Rust   | Python    |
| :----------------- | :----- | :-------- |
| `Linux (x86_64)`   | 1.97.0 | 3.12-3.14 |
| `Linux (ARM64)`    | 1.97.0 | 3.12-3.14 |
| `macOS (ARM64)`    | 1.97.0 | 3.12-3.14 |
| `Windows (x86_64)` | 1.97.0 | 3.12-3.14 |

- **文档**: <https://nautilustrader.io/docs/>
- **网站**: <https://nautilustrader.io>
- **支持**: [support@nautilustrader.io](mailto:support@nautilustrader.io)

## 简介

NautilusTrader 是一个开源、生产级、以 Rust 为核心构建的多资产、多交易场所（venue）交易系统引擎。

该系统在单一的事件驱动架构中涵盖研究、确定性模拟（deterministic simulation）与实盘执行，Python 作为策略逻辑、配置和编排（orchestration）的控制平面。

这种分离既提供了编译型交易引擎的性能与安全性，又保留了 Python 在系统组合与策略开发方面的灵活性。
对于关键任务（mission-critical）的工作负载，交易系统也可以完全用 Rust 编写。

无论在研究环境还是实盘系统中，都运行着相同的执行语义和确定性时间模型。策略从研究阶段部署到生产环境无需修改代码，
从而实现研究与实盘的一致性（research-to-live parity），减少了通常会引入部署风险的差异。

NautilusTrader 与资产类别无关（asset-class-agnostic）。任何提供 REST API 或 WebSocket 数据流的交易场所都可以通过模块化适配器（adapter）进行集成。当前集成涵盖加密货币交易所
（中心化交易所 CEX 和去中心化交易所 DEX）、传统市场（外汇、股票、期货、期权），以及博彩交易所。

![nautilus-trader](https://github.com/nautechsystems/nautilus_trader/raw/develop/assets/nautilus-trader.png "nautilus-trader")

## 特性

- **快速**：Rust 内核搭配 [mimalloc](https://github.com/microsoft/mimalloc) 分配器，并使用 [tokio](https://crates.io/crates/tokio) 实现异步网络通信。
- **可靠**：由 Rust 保证类型安全与线程安全，并可选配 Redis 支持的状态持久化。
- **可移植**：可在 Linux、macOS 和 Windows 上运行，也可使用 Docker 部署。
- **灵活**：模块化适配器可集成任意 REST API 或 WebSocket 数据流。
- **高级功能**：支持 `IOC`、`FOK`、`GTC`、`GTD`、`DAY`、`AT_THE_OPEN`、`AT_THE_CLOSE` 等有效期类型（time in force）、高级订单类型和条件触发。支持 `post-only`、`reduce-only` 及冰山单（iceberg）等执行指令。支持包括 `OCO`、`OUO`、`OTO` 在内的连带订单（contingency order）。
- **可定制**：可使用用户自定义组件，或利用 [缓存（cache）](https://nautilustrader.io/docs/latest/concepts/cache) 与 [消息总线（message bus）](https://nautilustrader.io/docs/latest/concepts/message_bus) 从零搭建完整系统。
- **回测**：可同时使用报价（quote tick）、成交（trade tick）、K线（bar）、订单簿（order book）以及自定义数据，以纳秒级精度对多个交易场所、多个金融工具、多个策略进行回测（backtest）。
- **实盘**：研究与实盘部署使用完全相同的策略实现。
- **多场所**：可在多个交易场所上同时运行做市（market-making）和跨场所策略。
- **AI 训练**：引擎速度足以支撑 AI 交易智能体（RL/ES）的训练。

![nautilus](https://github.com/nautechsystems/nautilus_trader/raw/develop/assets/nautilus-art.png "nautilus")

> *nautilus - 源自古希腊语，意为“水手”，和 naus，意为“船”。*
>
> *鹦鹉螺（nautilus）的贝壳由若干模块化腔室组成，其生长因子近似于对数螺旋。
> 这一理念可以转化为设计与架构上的美学表达。*

## 为什么选择 NautilusTrader？

交易策略研究通常在 Python 中以向量化方式进行，而生产环境中的交易系统则通常
使用编译型语言，以事件驱动架构单独实现。

NautilusTrader 消除了这种分离。

以 Rust 为核心构建的引擎为研究和实盘执行提供了确定性的事件驱动运行时，而 Python
则作为控制平面。相同的架构、执行语义和时间模型在两种环境下运行，使策略能够
从研究阶段直接迁移到生产环境，而无需重新实现。

Python 绑定通过 [PyO3](https://pyo3.rs) 为以 Rust 为核心的 v2 运行时提供支持。
在 v2 发布候选（release-candidate）阶段，旧版基于 Cython 的 v1 内核仍会继续得到支持。
安装时不需要 Rust 工具链。

本项目承诺遵循 [健全性宣言（Soundness Pledge）](https://raphlinus.github.io/rust/2020/01/18/soundness-pledge.html)：

> “本项目的宗旨是不存在健全性（soundness）缺陷。
> 开发者将尽力避免此类问题，并欢迎大家协助分析与修复。”

> [!NOTE]
>
> **MSRV：** NautilusTrader 大量依赖于 Rust 语言及编译器的改进。
> 因此，最低支持 Rust 版本（MSRV）通常与最新稳定版 Rust 保持一致。

## 集成

NautilusTrader 采用模块化设计，通过*适配器（adapter）*将交易场所和数据提供商的原始 API
转换为统一接口和规范化的领域模型，从而实现互联互通。

目前支持以下集成；详情请参见 [docs/integrations/](https://nautilustrader.io/docs/latest/integrations/)：

| 名称                                                                         | ID                    | 类型                    | 状态                                                  | 文档                                       |
| :--------------------------------------------------------------------------- | :-------------------- | :---------------------- | :-------------------------------------------------------| :----------------------------------------- |
| [AX Exchange](https://architect.exchange)                                    | `AX`                  | 永续合约交易所           | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/architect_ax.md) |
| [Betfair](https://betfair.com)                                               | `BETFAIR`             | 体育博彩交易所           | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/betfair.md)      |
| [Binance](https://binance.com)                                               | `BINANCE`             | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/binance.md)      |
| [BitMEX](https://www.bitmex.com)                                             | `BITMEX`              | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/bitmex.md)       |
| [Bybit](https://www.bybit.com)                                               | `BYBIT`               | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/bybit.md)        |
| [Coinbase](https://coinbase.com)                                             | `COINBASE`            | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/coinbase.md)     |
| [Databento](https://databento.com)                                           | `DATABENTO`           | 数据提供商               | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/databento.md)    |
| [Deribit](https://www.deribit.com)                                           | `DERIBIT`             | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/deribit.md)      |
| [Derive](https://www.derive.xyz)                                             | `DERIVE`              | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/derive.md)       |
| [dYdX](https://dydx.exchange/)                                               | `DYDX`                | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/dydx.md)         |
| [Hyperliquid](https://hyperliquid.xyz)                                       | `HYPERLIQUID`         | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/hyperliquid.md)  |
| [Interactive Brokers](https://www.interactivebrokers.com)                    | `INTERACTIVE_BROKERS` | 券商（多交易场所）       | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/ib.md)           |
| [Kraken](https://kraken.com)                                                 | `KRAKEN`              | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/kraken.md)       |
| [Lighter](https://lighter.xyz)                                               | `LIGHTER`             | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/lighter.md)      |
| [OKX](https://okx.com)                                                       | `OKX`                 | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/okx.md)          |
| [Polymarket](https://polymarket.com)                                         | `POLYMARKET`          | 预测市场 (DEX)           | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/polymarket.md)   |
| [Tardis](https://tardis.dev)                                                 | `TARDIS`              | 加密货币数据提供商       | ![status](https://img.shields.io/badge/stable-green)    | [指南](docs/integrations/tardis.md)       |

- **ID**：该集成适配器客户端使用的默认客户端 ID。
- **类型**：集成的类型（通常为交易场所类型）。

### 状态

- `planned`：计划中，尚未开始开发。
- `building`：正在构建中，可能尚不可用。
- `beta`：已完成到最基本可用的状态，处于 beta 测试阶段。
- `stable`：功能集与 API 已趋于稳定，该集成已经过开发者和用户的合理程度测试（可能仍存在少量缺陷）。

更多详情请参见 [集成（Integrations）](https://nautilustrader.io/docs/latest/integrations/) 文档。

## 路线图

[路线图（Roadmap）](/ROADMAP.md) 概述了 NautilusTrader 的战略方向。
当前的优先事项包括完善以 Rust 为核心的内核、改进文档，以及提升代码的易用性（ergonomics）。

该开源项目专注于面向个人及小团队量化交易者的单节点回测和实盘交易。
UI 仪表盘、分布式编排以及内置的 AI/ML 工具不在项目范围内，以便将精力集中在核心引擎和生态系统的可持续发展上。

新的集成提案应先以 RFC issue 的形式讨论其适用性，然后再提交 PR。
相关指南请参见 [社区贡献的集成](/ROADMAP.md#community-contributed-integrations)。

## 安全

[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/nautechsystems/nautilus_trader/badge)](https://scorecard.dev/viewer/?uri=github.com/nautechsystems/nautilus_trader)

安全是 NautilusTrader 项目的首要事项，我们重视那些帮助发现并修复漏洞的人所做的工作。
我们在开发和发布的整个生命周期中应用分层控制措施，包括签名发布、持续的漏洞管理，
以及透明的开发实践：

- **源代码与审查控制**：CODEOWNERS 机制对关键基础设施、依赖清单和锁定文件进行把关；
  受保护分支要求提交经过签名且通过 CI；发布标签（tag）不可变；Rust 依赖仅来自 crates.io。
- **依赖引入管控**：锁定文件通过加密校验和固定每一个依赖版本，第三方 Python 包仅从
  wheel 安装，新依赖及工具版本需经过发布冷静期后方可采用，cargo-vet 对 Rust 依赖来源
  进行审计，许可证检查确保符合 LGPL-3.0-or-later 兼容性要求。
- **扫描与模糊测试**：Gitleaks 密钥泄露扫描和 Zizmor Actions 审计在 pre-commit 阶段运行；
  CodeQL 在针对 `master` 的 PR 及推送到 `nightly` 时运行；cargo-audit、cargo-deny、
  cargo-vet、OSV Scanner 和 pip-audit 在涉及审计的 PR 及每日计划任务中运行；
  cargo-fuzz 针对选定的适配器和签名相关代码面进行模糊测试。
- **构建与发布完整性**：GitHub Actions 固定到具体的提交 SHA，CI runner 通过出站流量
  白名单加固，Python 制品携带 SLSA 构建溯源信息，容器镜像通过 Sigstore 进行无密钥签名
  并附带经过认证的 SPDX SBOM，PyPI 和 crates.io 的发布使用 OIDC 可信发布
  （Trusted Publishing），且仅限于受保护的 `release` 环境，该环境从不运行 PR 或
  fork 中的代码。
- **运行时加密**：TLS 及大多数运行时加密使用
  [aws-lc-rs](https://github.com/aws/aws-lc-rs)（AWS-LC 的 Rust 绑定），
  Ed25519 签名通过 [ed25519-dalek](https://github.com/dalek-cryptography/curve25519-dalek) 实现。

上方的 OpenSSF Scorecard 徽章只是一种自动化的仓库健康度信号；它是对人工审查、
CI 加固以及安全审计的补充，而非替代。

### 报告漏洞

请通过
[GitHub Security Advisories](https://github.com/nautechsystems/nautilus_trader/security/advisories/new)
私下报告，或发送邮件至 <security@nautechsystems.io>（如需可提供 PGP 密钥）。我们会在
48 小时内确认收到报告，并在 30 天内修复严重漏洞。

一份用心的漏洞报告需要花费实实在在的时间和精力。我们对此深表感激，除非报告者希望匿名，
否则我们会在相应的安全公告和发布说明中对报告者予以致谢。

[安全策略（Security Policy）](SECURITY.md) 详细说明了范围、协同披露流程以及逐步的
发布验证方法。[发布安全架构（Release Security Architecture）](docs/developer_guide/release_security.md)
端到端地描述了发布供应链。完整策略请参见
[负责任披露（Responsible Disclosure）](https://nautilustrader.io/security/responsible-disclosure/) 和
[供应链安全（Supply Chain Security）](https://nautilustrader.io/security/supply-chain/) 政策；
CI/CD 安全相关内容记录在 [.github/OVERVIEW.md](.github/OVERVIEW.md#security) 中。

## 版本控制与发布

> [!WARNING]
>
> **NautilusTrader 仍处于积极开发阶段**。部分功能可能尚不完整，尽管 API 正在
> 变得更加稳定，但版本之间仍可能出现破坏性变更（breaking change）。
> 我们会尽力在发布说明中以**尽力而为（best-effort basis）**的方式记录这些变更。

我们的目标是遵循**每两周一次的发布节奏**，但实验性功能或较大的功能可能会导致延迟。

### 分支

我们致力于让所有分支保持稳定、可通过构建。

- `master`：反映最新已发布版本的源代码；推荐用于生产环境。
- `nightly`：`develop` 分支的每日快照，用于早期测试；在 **UTC 时间 14:00** 及必要时合并。
- `develop`：面向贡献者和功能开发的活跃开发分支。

> [!NOTE]
>
> v2 发布候选系列是向**版本 2.x 的稳定 API**过渡的阶段。
> 一旦达到这一里程碑，我们计划针对任何 API 变更实施正式的弃用（deprecation）流程。
> 这种方式让我们目前能够保持较快的开发节奏。

## 精度模式

NautilusTrader 为其核心值类型（`Price`、`Quantity`、`Money`）支持两种精度模式，
二者在内部位宽和最大小数精度上有所不同。

- **高精度**：128 位整数，最多支持 16 位小数精度，取值范围更大。
- **标准精度**：64 位整数，最多支持 9 位小数精度，取值范围较小。

> [!NOTE]
>
> 默认情况下，官方 Python wheel 在 Linux 和 macOS 上以高精度（128 位）模式发布。
> 在 Windows 上，仅提供标准精度（64 位）的 Python wheel，因为 MSVC 的 C/C++ 前端
> 不支持 `__int128`，导致 Cython/FFI 层无法处理 128 位整数。
>
> 对于纯 Rust crate，由于 Rust 通过软件模拟支持 `i128`/`u128`，高精度模式可在所有平台
> （包括 Windows）上使用。除非显式启用 `high-precision` 特性标志（feature flag），
> 否则默认使用标准精度。

更多详情请参见 [安装指南](https://nautilustrader.io/docs/latest/getting_started/installation)。

**Rust 特性标志**：要在 Rust 中启用高精度模式，请在 Cargo.toml 中添加 `high-precision` 特性：

```toml
[dependencies]
nautilus_model = { version = "*", features = ["high-precision"] }
```

## 安装

我们建议使用最新的受支持 Python 版本，并在虚拟环境中安装
[nautilus_trader](https://pypi.org/project/nautilus_trader/) 以隔离依赖。

**目前支持两种安装方式**：

1. 从 PyPI *或* Nautech Systems 包索引安装预构建的二进制 wheel。
2. 从源代码构建。

> [!TIP]
>
> 我们强烈建议使用 [uv](https://docs.astral.sh/uv) 包管理器搭配“原生（vanilla）” CPython 进行安装。
>
> Conda 及其他 Python 发行版*可能*可以运行，但官方并不支持。

### 从 PyPI 安装

使用 Python 的 pip 包管理器从 PyPI 安装最新的二进制 wheel（或 sdist 包）：

```bash
pip install -U nautilus_trader
```

若要测试 PyPI 上的 v2 发布候选 wheel：

```bash
pip install -U nautilus_trader --pre
```

v2 发布候选 wheel 使用 `2.0.0rcN` 版本号，旨在供社区测试，为最终的 `2.0.0` 版本
发布做准备。我们不建议在生产环境（例如涉及真实资金的实盘交易）中使用发布候选版本。

针对特定集成（例如 `betfair`、`docker`、`ib`、`polymarket`、`visualization`），
可安装可选依赖作为“extras”：

```bash
pip install -U "nautilus_trader[docker,ib]"
```

完整的可用 extras 列表请参见
[安装指南](https://nautilustrader.io/docs/latest/getting_started/installation#extras)。

### 从 Nautech Systems 包索引安装

Nautech Systems 包索引（`packages.nautechsystems.io`）遵循 [PEP-503](https://peps.python.org/pep-0503/) 标准，
托管了 `nautilus_trader` 的稳定版和开发版二进制 wheel。
这使用户可以安装最新的稳定发布版本，也可以安装预发布版本用于测试。

#### 稳定版 wheel

稳定版 wheel 对应 `nautilus_trader` 在 PyPI 上的正式发布版本，使用标准版本号规则。

安装最新的稳定发布版本：

```bash
pip install -U nautilus_trader --index-url=https://packages.nautechsystems.io/simple
```

> [!TIP]
>
> 如果希望 pip 在需要时自动回退到 PyPI，请使用 `--extra-index-url` 而非 `--index-url`。

#### 开发版 wheel

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

> [!WARNING]
>
> 我们不建议在生产环境（例如涉及真实资金的实盘交易）中使用开发版 wheel。

#### 安装命令

默认情况下，pip 会安装最新的稳定发布版本。添加 `--pre` 标志后，pip 会考虑
预发布版本（包括开发版 wheel）。

安装当前可用的最新预发布版本（包括开发版 wheel）：

```bash
pip install -U nautilus_trader --pre --index-url=https://packages.nautechsystems.io/simple
```

安装特定的开发版 wheel（例如 2025 年 10 月 26 日的 `1.221.0a20251026`）：

```bash
pip install nautilus_trader==1.221.0a20251026 --index-url=https://packages.nautechsystems.io/simple
```

#### 可用版本

你可以在[包索引](https://packages.nautechsystems.io/simple/nautilus-trader/index.html)上
查看 `nautilus_trader` 的所有可用版本。

以编程方式获取并列出可用版本：

```bash
curl -s https://packages.nautechsystems.io/simple/nautilus-trader/index.html | sed -n 's/.*<a href="\([^"]*\)".*/\1/p' | awk -F'#' '{print $1}' | sort
```

> [!NOTE]
>
> 在 Linux 上，安装二进制 wheel 之前，请使用 `ldd --version` 确认 glibc 版本，
> 并确保其报告的版本号为 **2.35** 或更高。

#### 分支更新

- `develop` 分支 wheel（`.dev`）：每次合并提交都会持续构建并发布。
- `nightly` 分支 wheel（`a`）：当我们在 **UTC 时间 14:00** 自动将 `develop` 分支合并时
  （若有变更），每日构建并发布。

#### 保留策略

- `develop` 分支 wheel（`.dev`）：我们仅保留最近一次的构建。
- `nightly` 分支 wheel（`a`）：我们仅保留最近的 30 次构建。

#### 验证构建溯源

CI/CD 流水线为所有发布的制品生成加密认证：

- Python wheel 和源代码分发包（PyPI、GitHub Releases、Nautech Systems 包索引）：
  [SLSA](https://slsa.dev/) 构建溯源。
- Docker 镜像（`ghcr.io/nautechsystems/nautilus_trader`、`ghcr.io/nautechsystems/jupyterlab`）：
  无密钥的 [cosign](https://github.com/sigstore/cosign) 签名以及 SPDX SBOM 认证。

两者均通过 [Sigstore](https://www.sigstore.dev/) 签发，并绑定到特定的提交 SHA，
因此验证可以确保制品由官方 NautilusTrader GitHub Actions 工作流生成，且自生成以来未被篡改。

具体的验证命令请参见 `SECURITY.md` 中的 [验证发布版本](SECURITY.md#verifying-releases)。

> [!NOTE]
>
> 验证 Python 制品需要 [GitHub CLI](https://cli.github.com/)（`gh`），
> 验证 Docker 镜像需要 [cosign](https://github.com/sigstore/cosign)。
> 来自 `develop` 和 `nightly` 分支的开发版 wheel 同样经过认证。

### 从源代码安装

如果你先按照 `pyproject.toml` 中指定的方式安装构建依赖，就可以使用 pip 从源代码安装。

1. 安装 [rustup](https://rustup.rs/)（Rust 工具链安装器）：
   - Linux 和 macOS：

       ```bash
       curl https://sh.rustup.rs -sSf | sh
       ```

   - Windows：
       - 下载并安装 [`rustup-init.exe`](https://win.rustup.rs/x86_64)
       - 通过 [Build Tools for Visual Studio 2022](https://visualstudio.microsoft.com/visual-cpp-build-tools/) 安装“使用 C++ 的桌面开发”
   - 验证（任意系统）：
       在终端会话中运行：`rustc --version`

2. 在当前 shell 中启用 `cargo`：
   - Linux 和 macOS：

       ```bash
       source $HOME/.cargo/env
       ```

   - Windows：
     - 启动一个新的 PowerShell

3. 安装 [clang](https://clang.llvm.org/)（LLVM 的 C 语言前端）：
   - Linux（同时会安装 [lld](https://lld.llvm.org/)，用作 Rust 链接器以加快构建速度）：

       ```bash
       sudo apt-get install clang lld
       ```

   - macOS：

       ```bash
       xcode-select --install
       ```

   - Windows：
       1. 在 [Build Tools for Visual Studio 2022](https://visualstudio.microsoft.com/visual-cpp-build-tools/) 中添加 Clang：
          - 开始 | Visual Studio Installer | 修改 | 勾选 C++ Clang tools for Windows (latest) | 修改
       2. 在当前 shell 中启用 `clang`：

          ```powershell
          [System.Environment]::SetEnvironmentVariable('path', "C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Tools\Llvm\x64\bin\;" + $env:Path,"User")
          ```

   - 验证（任意系统）：
       在终端会话中运行：`clang --version`

4. 安装 uv（详情请参见 [uv 安装指南](https://docs.astral.sh/uv/getting-started/installation)）：

    - Linux 和 macOS：

        ```bash
        curl -LsSf https://astral.sh/uv/install.sh | sh
        ```

    - Windows（PowerShell）：

        ```powershell
        irm https://astral.sh/uv/install.ps1 | iex
        ```

5. 使用 `git` 克隆源代码，并在项目根目录下安装：

    ```bash
    git clone --branch develop --depth 1 https://github.com/nautechsystems/nautilus_trader
    cd nautilus_trader
    uv sync --all-extras
    ```

> [!NOTE]
>
> `--depth 1` 标志仅获取最新提交，实现更快、更轻量的克隆。

6. 为 PyO3 编译设置环境变量（仅限 Linux 和 macOS）。在 `uv sync` 之后，从仓库根目录
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

> [!NOTE]
>
> `LD_LIBRARY_PATH` 的设置仅针对 Linux，macOS 不需要。
>
> 使用 `uv` 安装的 Python 运行 `make cargo-test` 时需要设置 `PYTHONHOME` 变量。
> 若未设置，依赖 PyO3 的测试可能无法定位到 Python 运行时。

其他选项及更多详情请参见 [安装指南](https://nautilustrader.io/docs/latest/getting_started/installation)。

## Redis

在 NautilusTrader 中使用 [Redis](https://redis.io) 是**可选的**，仅当将其配置为
[缓存（cache）](https://nautilustrader.io/docs/latest/concepts/cache)数据库或
[消息总线（message bus）](https://nautilustrader.io/docs/latest/concepts/message_bus) 的后端时才需要。
更多详情请参见[安装指南](https://nautilustrader.io/docs/latest/getting_started/installation#redis)中的 **Redis** 部分。

## Makefile

项目提供了一个 `Makefile`，用于自动化大多数开发过程中的安装和构建任务。部分目标（target）包括：

- `make install`：以 `release` 构建模式安装所有依赖分组和 extras。
- `make install-debug`：与 `make install` 相同，但使用 `debug` 构建模式。
- `make install-deps`：安装 Python 依赖，但不构建软件包。
- `make build`：以 `release` 构建模式（默认）运行构建脚本。
- `make build-debug`：以 `debug` 构建模式运行构建脚本。
- `make build-wheel`：以 `release` 模式使用 uv 构建 wheel 格式。
- `make build-wheel-debug`：以 `debug` 模式使用 uv 构建 wheel 格式。
- `make cargo-test`：使用 `cargo-nextest` 运行所有 Rust crate 测试。
- `make clean`：删除构建产物、缓存及构建目录。
- `make distclean`：**注意** 当以 `FORCE=1` 运行时，会删除所有未加入 git 索引的产物，
  包括未执行过 `git add` 的源文件。
- `make docs`：使用 Sphinx 构建 HTML 文档。
- `make pre-commit`：对所有文件运行 pre-commit 检查。
- `make ruff`：使用 `pyproject.toml` 配置对所有文件运行 ruff（自动修复）。
- `make pytest`：使用 `pytest` 运行所有测试。
- `make test-performance`：使用 [codspeed](https://codspeed.io) 运行性能测试。

> [!TIP]
>
> 运行 `make help` 查看所有可用 make 目标的说明文档。

> [!TIP]
>
> 有关运行基础设施集成测试的说明，请参见
> [crates/infrastructure/TESTS.md](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/infrastructure/TESTS.md) 文件。

## 示例

指标（indicator）和策略可以用 Python、Cython 或 Rust 编写。对于性能和延迟敏感的
应用场景，我们推荐使用 Rust。以下是一些示例：

- 用 Python 编写的[指标](/nautilus_trader/examples/indicators/ema_python.py)示例。
- 用 Cython 实现的[指标](/nautilus_trader/indicators/)。
- 用 Python 编写的[策略](/nautilus_trader/examples/strategies/)示例。
- 直接使用 `BacktestEngine` 的[回测](/examples/backtest/)示例。

## Docker

Docker 容器使用以下变体标签（tag）构建：

- `nautilus_trader:latest` 安装了最新的发布版本。
- `nautilus_trader:nightly` 安装了 `nightly` 分支的最新头部提交。
- `jupyterlab:latest` 安装了最新的发布版本，并附带 `jupyterlab` 及配套数据的
  回测示例 notebook。
- `jupyterlab:nightly` 安装了 `nightly` 分支的最新头部提交，并附带 `jupyterlab` 及
  配套数据的回测示例 notebook。

可以通过以下方式拉取容器镜像：

```bash
docker pull ghcr.io/nautechsystems/<image_variant_tag> --platform linux/amd64
```

可以运行以下命令启动回测示例容器：

```bash
docker pull ghcr.io/nautechsystems/jupyterlab:nightly --platform linux/amd64
docker run -p 8888:8888 ghcr.io/nautechsystems/jupyterlab:nightly
```

然后在浏览器中打开以下地址：

```bash
http://127.0.0.1:8888/lab
```

> [!WARNING]
>
> 示例中使用了 `log_level="ERROR"`，因为 Nautilus 的日志输出超过了 Jupyter 的
> stdout 速率限制，在较低日志级别下会导致 notebook 挂起。

## 开发

对于这个混合了 Rust、Python 和 Cython 的代码库，我们致力于提供尽可能愉悦的开发体验。
更多有用信息请参见[开发者指南（Developer Guide）](https://nautilustrader.io/docs/latest/developer_guide/)。

> [!TIP]
>
> 修改 Rust 或 Cython 代码后运行 `make build-debug` 进行编译，这是最高效的开发流程。

在修改 v2 PyO3 绑定、stub 注解或封装的 Rust 文档后，需要从仓库根目录重新生成
Python 制品：

```bash
make py-stubs-v2
```

需要提交该目标修改产生的 `.pyi` 文件以及 PyO3 封装的文档注释。详情请参见
[生成的 Python 制品](docs/developer_guide/rust.md#generated-python-artifacts)。

### 使用 Rust 进行测试

[cargo-nextest](https://nexte.st) 是 NautilusTrader 的标准 Rust 测试运行器。
它的主要优势在于将每个测试隔离在独立进程中运行，通过避免相互干扰来确保测试的可靠性。

可以通过以下命令安装 cargo-nextest：

```bash
cargo install cargo-nextest
```

> [!TIP]
>
> 使用 `make cargo-test` 运行 Rust 测试，该命令会使用高效配置的 **cargo-nextest**。

## 贡献

感谢你考虑为 NautilusTrader 做出贡献！我们欢迎任何有助于改进项目的帮助。如果你有
改进或修复缺陷的想法，第一步是在 GitHub 上开一个 [issue](https://github.com/nautechsystems/nautilus_trader/issues)
与团队讨论。这有助于确保你的贡献与项目目标保持一致，并避免重复劳动。

在开始工作之前，请务必查阅项目路线图中概述的[开源范围](/ROADMAP.md#open-source-scope)，
以了解哪些内容在范围之内，哪些不在。

准备好开始贡献后，请务必遵循
[CONTRIBUTING.md](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)
文件中列出的指南。这包括签署贡献者许可协议（CLA），以确保你的贡献可以被纳入项目。

> [!NOTE]
>
> Pull request 应针对 `develop` 分支（默认分支）提交。新功能和改进都会先集成到该分支，之后再发布。

再次感谢你对 NautilusTrader 的关注！我们期待审阅你的贡献，并与你一起改进这个项目。

## 社区

欢迎加入我们在 [Discord](https://discord.gg/NautilusTrader) 上的用户和贡献者社区，
交流讨论并及时了解 NautilusTrader 的最新公告和功能。无论你是想贡献代码的开发者，
还是只想进一步了解这个平台，都欢迎加入我们的 Discord 服务器。

> [!WARNING]
>
> NautilusTrader 不发行、不推广、也不认可任何加密货币代币。任何声称与此相反的说法或
> 通讯均未经授权，属于虚假信息。
>
> NautilusTrader 的所有官方更新和通讯将仅通过以下渠道发布：<https://nautilustrader.io>、
> 我们的 [GitHub](https://github.com/nautechsystems)、我们的 [Discord 服务器](https://discord.gg/NautilusTrader)，
> 或我们经过验证的 X（Twitter）账号：[@NautilusTrader](https://x.com/NautilusTrader)。
>
> 如发现任何可疑活动，请向相应平台举报，并通过 <info@nautechsystems.io> 与我们联系。

## 许可证

NautilusTrader 的源代码在 GitHub 上以
[GNU 宽通用公共许可证 v3.0（LGPL-3.0）](https://www.gnu.org/licenses/lgpl-3.0.en.html)发布。
欢迎为该项目做出贡献，但需要完成标准的
[贡献者许可协议（CLA）](https://github.com/nautechsystems/nautilus_trader/blob/develop/CLA.md)。

---

NautilusTrader™ 由 Nautech Systems 开发和维护，该公司专注于开发高性能交易系统。
更多信息请访问 <https://nautilustrader.io>。

使用本软件须遵守[免责声明（Disclaimer）](https://nautilustrader.io/legal/disclaimer/)。

© 2015-2026 Nautech Systems Pty Ltd. 保留所有权利。

![nautechsystems](https://github.com/nautechsystems/nautilus_trader/raw/develop/assets/ns-logo.png "nautechsystems")
<img src="https://github.com/nautechsystems/nautilus_trader/raw/develop/assets/ferris.png" width="128">
</content>
