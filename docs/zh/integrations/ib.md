# Interactive Brokers

Interactive Brokers(IB)是一个交易平台,提供跨广泛金融标的的市场
接入,包括股票、期权、期货、货币、债券、基金和加密货币。NautilusTrader
提供了一个适配器,使用其
[Trader Workstation(TWS)API](https://ibkrcampus.com/ibkr-api-page/trader-workstation-api/)
与 IB 集成。v1 遗留路径使用 Python 的
[ibapi](https://github.com/nautechsystems/ibapi) 包,而 v2 路径以
Rust 实现,位于 `crates/adapters/interactive_brokers`,并为 Python
v2 提供 PyO3 绑定。

TWS API 是 IB 独立交易应用程序(TWS 和 IB Gateway)的接口。两者都可以
从 IB 网站下载。如果你尚未安装 TWS 或 IB Gateway,请参考
[初始设置](https://ibkrcampus.com/ibkr-api-page/trader-workstation-api/#tws-download)
指南。在 NautilusTrader 中,你将通过 `InteractiveBrokersClient` 与
其中一个应用建立连接。

或者,你也可以从 IB Gateway 的
[docker 化版本](https://github.com/gnzsnz/ib-gateway-docker)开始,
这在托管云平台上部署交易策略时特别有用。这需要在你的机器上安装
[Docker](https://www.docker.com/),以及
[docker](https://pypi.org/project/docker/) Python 包,
NautilusTrader 已将其作为附加包便捷地包含在内。

:::note
独立的 TWS 和 IB Gateway 应用程序需要在启动时手动输入用户名、密码和
交易模式(实盘或模拟)。docker 化版本的 IB Gateway 会以编程方式处理
这些步骤。
:::

## 安装

安装带有 Interactive Brokers(和 Docker)支持的 NautilusTrader:

```bash
uv pip install "nautilus_trader[ib,docker]"
```

从源码构建并带上所有附加组件(包括 IB 和 Docker):

```bash
uv sync --all-extras
```

:::note
由于 IB 不为 `ibapi` 提供 wheel 包,NautilusTrader 对其进行了
[重新打包](https://pypi.org/project/nautilus-ibapi/),以在 PyPI 上
发布。
:::

## 示例

- Rust 实时节点测试器:
  [`crates/adapters/interactive_brokers/examples/`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/adapters/interactive_brokers/examples/)
- Python 实时示例:
  [`examples/live/interactive_brokers/`](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/interactive_brokers/)

## 快速入门

在实现你的交易策略之前,请确保 TWS(Trader Workstation)或 IB
Gateway 正在运行。你可以使用你的凭证登录这两个独立应用程序中的
任意一个,或通过 `DockerizedIBGateway` 以编程方式连接。

:::warning
在连接 NautilusTrader 之前,请将 TWS 或 IB Gateway 配置为以 UTC
返回市场数据时间戳。此设置必须由用户在 TWS/IB Gateway 中启用,因为
NautilusTrader 设计为使用 UTC 时间戳工作。
:::

### 连接方式

有两种主要方式连接到 Interactive Brokers:

1. **连接到现有的 TWS 或 IB Gateway 实例**
2. **使用 docker 化的 IB Gateway(推荐用于自动化部署)**

### 默认端口

Interactive Brokers 根据应用程序和交易模式使用不同的默认端口:

| 应用程序 | 模拟交易 | 实盘交易 |
|-------------|---------------|--------------|
| TWS         | 7497          | 7496         |
| IB Gateway  | 4002          | 4001         |

### 建立到现有网关或 TWS 的连接

连接到一个预先存在的网关或 TWS 时,请在
`InteractiveBrokersDataClientConfig` 和
`InteractiveBrokersExecClientConfig` 中都指定 `ibg_host` 和
`ibg_port` 参数:

```python
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersDataClientConfig
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersExecClientConfig

# TWS 模拟交易示例(默认端口 7497)
data_config = InteractiveBrokersDataClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,
    ibg_client_id=1,
)

exec_config = InteractiveBrokersExecClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,
    ibg_client_id=1,
    account_id="DU123456",  # 你的模拟交易账户 ID
)
```

### 建立到 Docker 化 IB Gateway 的连接

对于自动化部署,建议使用 docker 化网关。在两个客户端配置中都提供
`dockerized_gateway`,传入一个 `DockerizedIBGatewayConfig` 实例。
`ibg_host` 和 `ibg_port` 参数不需要,因为它们会被自动管理。

```python
from nautilus_trader.adapters.interactive_brokers.config import DockerizedIBGatewayConfig
from nautilus_trader.adapters.interactive_brokers.gateway import DockerizedIBGateway

gateway_config = DockerizedIBGatewayConfig(
    username="your_username",  # 或设置 TWS_USERNAME 环境变量
    password="your_password",  # 或设置 TWS_PASSWORD 环境变量
    trading_mode="paper",      # "paper" 或 "live"
    read_only_api=True,        # 设为 False 以允许订单执行
    timeout=300,               # 启动超时时间(秒)
)

# 启动可能需要一些时间,尤其是首次启动
gateway = DockerizedIBGateway(config=gateway_config)
gateway.start()

# 确认已登录
print(gateway.is_logged_in(gateway.container))

# 检查日志
print(gateway.container.logs())
```

### 环境变量

要向 Interactive Brokers Gateway 提供凭证,可以将 `username` 和
`password` 传给 `DockerizedIBGatewayConfig`,或设置以下环境变量:

- `TWS_USERNAME`:你的 IB 账户用户名。
- `TWS_PASSWORD`:你的 IB 账户密码。
- `TWS_ACCOUNT`:你的 IB 账户 ID(作为 `account_id` 的回退值)。

### 连接管理

该适配器包含连接管理功能:

- **自动重连**:通过 `IB_MAX_CONNECTION_ATTEMPTS` 环境变量配置重试
  次数。
- **连接超时**:通过 `connection_timeout` 参数调整超时时间(默认:
  300 秒)。
- **连接看门狗**:监控连接健康状况,并在需要时自动触发重连。
- **优雅的错误处理**:通过错误分类处理各种连接场景。
- **瞬时数据农场容忍**:瞬时的 IB 数据农场状态通知(代码
  2103/2105 “broken”,随后是 2104/2106 “OK”,常见于每晚的网关
  重启期间)只会降级受影响的数据流,并在该农场恢复后通过重新订阅
  恢复。它们**不会**拆除套接字连接或订单/执行通道。

#### 客户端 ID 分配

IB 按客户端 ID 限定未成交订单的可见范围:给定的客户端 ID 只能看到、
也只能管理它自己下达的订单。分配 ID 时请牢记这一点:

- **为每个客户端预留一个 ID 段。** 在网关重启期间重连时,配置的
  客户端 ID 仍可能被尚未释放的上一个会话占用(IB 错误 326)。
  适配器会退避,并首先重试*同一个*已配置的 ID;只有当冲突持续
  存在时,才会回退到一个有界范围内附近的 ID(确定性递增,而非
  随机)。为每个客户端分配一个连续的 ID 段(例如 `1-5`、`6-10`、
  `11-15`),以便回退始终保持在该客户端的范围内,永远不会与其他
  进程冲突。
- **订单隔离并非在任何时候都有保证。** 当在回退 ID 下运行时,按
  客户端 ID 的订单隔离不再成立,适配器会在这种情况下重新获取所有
  未成交订单。请勿将按客户端 ID 的订单隔离当作一个硬性保证依赖。

## 概览

Interactive Brokers 适配器提供了与 IB TWS API 的集成。该适配器包含
几个主要组件:

### 核心组件

- **`InteractiveBrokersClient`**:使用 `ibapi` 执行 TWS API 请求的
  中央客户端。管理连接、处理错误,并协调所有 API 交互。
- **`InteractiveBrokersDataClient`**:连接到网关,以流式获取市场
  数据,包括报价、成交和 K 线。
- **`InteractiveBrokersExecutionClient`**:处理账户信息、订单管理
  和交易执行。
- **`InteractiveBrokersInstrumentProvider`**:获取和管理标的定义,
  包括对期权链和期货链的支持。
- **`HistoricInteractiveBrokersClient`**:提供获取标的和历史数据的
  方法,适用于回测和研究。

### 支持组件

- **`DockerizedIBGateway`**:为自动化部署管理 docker 化的 IB
  Gateway 实例。
- **配置类**:为所有组件提供配置选项。
- **工厂类**:创建并使用必要的依赖项配置客户端实例。

### 支持的资产类别

该适配器支持通过 Interactive Brokers 交易所有主要资产类别:

- **股票**:股票、ETF 和股票期权。
- **固定收益**:债券和债券基金。
- **衍生品**:期货、期权和权证。
- **外汇**:现货外汇和外汇远期。
- **加密货币**:比特币、以太坊及其他数字资产。
- **商品**:实物商品和商品期货。
- **指数**:指数产品和指数期权。

## Interactive Brokers 客户端

`InteractiveBrokersClient` 是 IB 适配器的核心组件,负责管理一系列
功能。这些功能包括建立和维护连接、处理 API 错误、执行交易,以及
收集各类数据,如市场数据、合约/标的数据和账户详情。

`InteractiveBrokersClient` 被拆分为专门的 mixin 类,每个类负责一项
特定职责。

### 客户端架构

该客户端使用基于 mixin 的架构,每个 mixin 处理 IB API 的一个特定
方面:

#### 连接管理(`InteractiveBrokersClientConnectionMixin`)

- 建立并维护到 TWS/Gateway 的套接字连接。
- 处理连接超时和重连逻辑。
- 管理连接状态和健康监控。
- 通过 `IB_MAX_CONNECTION_ATTEMPTS` 环境变量支持可配置的重连尝试
  次数。

#### 错误处理(`InteractiveBrokersClientErrorMixin`)

- 处理所有 API 错误和警告。
- 按类型对错误进行分类(客户端错误、连接问题、请求错误)。
- 处理订阅和请求特定的错误场景。
- 提供错误日志记录和调试信息。

#### 账户管理(`InteractiveBrokersClientAccountMixin`)

- 获取账户信息和余额。
- 管理仓位数据和组合更新。
- 处理多账户场景。
- 处理与账户相关的通知。

#### 合约/标的管理(`InteractiveBrokersClientContractMixin`)

- 获取合约详情和规格。
- 处理标的搜索和查找。
- 管理合约验证。
- 支持复杂标的类型(期权链、期货链)。

#### 市场数据管理(`InteractiveBrokersClientMarketDataMixin`)

- 处理实时和历史市场数据订阅。
- 处理报价、成交和 K 线数据。
- 管理市场数据类型设置(实时、延迟、冻结)。
- 处理逐笔数据和市场深度。

#### 订单管理(`InteractiveBrokersClientOrderMixin`)

- 处理下单、修改和取消。
- 处理订单状态更新和执行报告。
- 管理订单验证和错误处理。
- 支持复杂订单类型和条件。

### 关键特性

- **异步操作**:所有操作都使用 Python 的 asyncio 完全异步实现。
- **错误处理**:错误分类与处理。
- **连接弹性**:带可配置重试逻辑的自动重连。
- **消息处理**:面向高吞吐量场景的高效消息队列处理。
- **状态管理**:对连接、订阅和请求进行正确的状态跟踪。

:::tip
要排查 TWS API 入站消息问题,可以考虑从
`InteractiveBrokersClient._process_message` 方法开始,它是处理从
API 接收到的所有消息的主要入口。
:::

## 符号规则

`InteractiveBrokersInstrumentProvider` 支持三种构造 `InstrumentId`
实例的方法,可以通过
`InteractiveBrokersInstrumentProviderConfig` 中的
`symbology_method` 枚举进行配置。

### 符号规则方法

#### 1. 简化符号规则(`IB_SIMPLIFIED`)- 默认

当 `symbology_method` 设置为 `IB_SIMPLIFIED`(默认设置)时,系统
使用直观、易读的符号规则:

**按资产类别的格式规则:**

- **外汇**:`{symbol}/{currency}.{exchange}`
  - 示例:`EUR/USD.IDEALPRO`
- **股票**:`{localSymbol}.{primaryExchange}`
  - localSymbol 中的空格替换为连字符
  - 示例:`BF-B.NYSE`、`SPY.ARCA`
- **期货**:`{localSymbol}.{exchange}`
  - 单个合约使用一位数年份
  - 示例:`ESM4.CME`、`CLZ7.NYMEX`
- **连续期货**:`{symbol}.{exchange}`
  - 代表近月合约,自动展期
  - 示例:`ES.CME`、`CL.NYMEX`
- **期货期权(FOP)**:`{localSymbol}.{exchange}`
  - 格式:`{symbol}{month}{year} {right}{strike}`
  - 示例:`ESM4 C4200.CME`
- **期权**:`{localSymbol}.{exchange}`
  - localSymbol 中的所有空格都被移除
  - 示例:`AAPL230217P00155000.SMART`
- **指数**:`^{localSymbol}.{exchange}`
  - 示例:`^SPX.CBOE`、`^NDX.NASDAQ`
- **债券**:`{localSymbol}.{exchange}`
  - 示例:`912828XE8.SMART`
- **加密货币**:`{symbol}/{currency}.{exchange}`
  - 示例:`BTC/USD.PAXOS`、`ETH/USD.PAXOS`

#### 2. 原始符号规则(`IB_RAW`)

将 `symbology_method` 设置为 `IB_RAW` 会强制执行与 IB API 中定义
字段直接对应的更严格解析规则。此方法在所有地区和标的类型上提供
最大的兼容性:

**格式规则:**

- **CFD**:`{localSymbol}={secType}.IBCFD`
- **商品**:`{localSymbol}={secType}.IBCMDTY`
- **其他类型默认**:`{localSymbol}={secType}.{exchange}`

**示例:**

- `IBUS30=CFD.IBCFD`
- `XAUUSD=CMDTY.IBCMDTY`
- `EUR.USD=CASH.IDEALPRO`
- `AAPL=STK.SMART`

此配置确保了明确的标的标识,并支持来自任何地区的标的,尤其是那些
简化解析可能失败的非标准符号规则的标的。

### MIC 场所转换

该适配器支持将 Interactive Brokers 交易所代码转换为市场标识代码
(MIC),以实现标准化的场所标识:

#### `convert_exchange_to_mic_venue`

设为 `True` 时,适配器会自动将 IB 交易所代码转换为对应的 MIC 代码:

```python
instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    convert_exchange_to_mic_venue=True,  # 启用 MIC 转换
    symbology_method=SymbologyMethod.IB_SIMPLIFIED,
)
```

**MIC 转换示例:**

- `CME` -> `XCME`(芝加哥商业交易所)
- `NASDAQ` -> `XNAS`(纳斯达克股票市场)
- `NYSE` -> `XNYS`(纽约证券交易所)
- `LSE` -> `XLON`(伦敦证券交易所)

#### `symbol_to_mic_venue`

符号前缀到 MIC 场所的覆盖映射。在场所解析中**最先**应用,独立于
`convert_exchange_to_mic_venue`。当某个合约的符号匹配已配置的前缀
时,会使用该 MIC 场所;否则解析会使用交易所(如果
`convert_exchange_to_mic_venue` 为 True,则可能还会进行 MIC
转换)。对于交易所为 SMART 的 OPT 合约(例如 SPX -> XCBO),以及
与 databento 风格标的 ID 对齐时非常有用。

```python
instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    symbol_to_mic_venue={
        "SPX": "XCBO",  # 交易所为 SMART 的 OPT -> XCBO
        "ES": "XCME",   # 所有 ES 期货/期权都使用 CME MIC
        "SPY": "ARCX",  # SPY 特别使用 ARCA
    },
)
# convert_exchange_to_mic_venue 可以为 True 或 False;symbol_to_mic_venue 会先被应用
```

#### 场所解析与 `_process_contract_details`

通过 `IBContract` 加载标的时,提供者会将 `venue=None` 传给
`_process_contract_details`,因此每个合约详情都会获得自己的场所
(通过 `symbol_to_mic_venue`、validExchanges 和 MIC 转换)。传入
单个场所字符串的调用方,所有详情仍会得到同一个场所。当你有混合或
SMART 路由的结果时,请传入 `venue=None` 以获得按详情的解析。

### 支持的标的格式

该适配器基于 Interactive Brokers 的合约规格支持多种标的格式:

#### 期货月份代码

- **F** = 一月,**G** = 二月,**H** = 三月,**J** = 四月
- **K** = 五月,**M** = 六月,**N** = 七月,**Q** = 八月
- **U** = 九月,**V** = 十月,**X** = 十一月,**Z** = 十二月

#### 按资产类别支持的交易所

**期货交易所:**

- `CME`、`CBOT`、`NYMEX`、`COMEX`、`KCBT`、`MGE`、`NYBOT`、`SNFE`

**期权交易所:**

- `SMART`(IB 的智能路由)

**外汇交易所:**

- `IDEALPRO`(IB 的外汇平台)

**加密货币交易所:**

- `PAXOS`(IB 的加密货币平台)

**CFD/商品交易所:**

- `IBCFD`、`IBCMDTY`(IB 的内部路由)

### 选择合适的符号规则方法

- **使用 `IB_SIMPLIFIED`**(默认)适用于大多数使用场景 —— 提供
  简洁、易读的标的 ID
- **使用 `IB_RAW`**,当处理复杂的国际标的,或简化解析失败时
- **启用 `convert_exchange_to_mic_venue`**,当你需要标准化的 MIC
  场所代码以满足合规性或数据一致性要求时

## 标的与合约

在 Interactive Brokers 中,一个 NautilusTrader `Instrument` 对应
一个 IB [Contract](https://ibkrcampus.com/ibkr-api-page/trader-workstation-api/#contracts)。
该适配器处理两种类型的合约表示:

### 合约类型

#### 基本合约(`IBContract`)

- 包含基本的合约标识字段
- 用于合约搜索和基本操作
- 无法直接转换为 NautilusTrader `Instrument`

#### 合约详情(`IBContractDetails`)

- 包含合约信息,包括:
  - 支持的订单类型
  - 交易时间和日历
  - 保证金要求
  - 价格增量和乘数
  - 市场数据权限
- 可以转换为 NautilusTrader `Instrument`
- 是交易操作所必需的

### 合约发现

要搜索合约信息,请使用
[IB 合约信息中心](https://pennies.interactivebrokers.com/cstools/contract_info/)。

### 加载标的

加载标的有两种主要方法:

Interactive Brokers 不支持使用 `load_all=True` 加载完整的 IB 标的
全集。请为节点启动时所需的标的配置 `load_ids` 或
`load_contracts`,或在订阅其市场数据之前显式请求某个标的。

#### 1. 使用 `load_ids`(推荐)

使用 `symbology_method=SymbologyMethod.IB_SIMPLIFIED`(默认)配合
`load_ids`,可获得简洁、直观的标的标识:

对于外汇标的,请使用斜杠分隔的符号,例如 `EUR/USD.IDEALPRO`。带点的
本地符号形式属于原始符号规则,例如 `EUR.USD=CASH.IDEALPRO`。

```python
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersInstrumentProviderConfig
from nautilus_trader.adapters.interactive_brokers.config import SymbologyMethod

instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    symbology_method=SymbologyMethod.IB_SIMPLIFIED,
    load_ids=frozenset([
        "EUR/USD.IDEALPRO",    # 外汇
        "SPY.ARCA",            # 股票
        "ESM24.CME",           # 期货
        "BTC/USD.PAXOS",       # 加密货币
        "^SPX.CBOE",           # 指数
    ]),
)
```

#### 2. 使用 `load_contracts`(用于复杂标的)

对于期权链/期货链等复杂场景,使用带 `IBContract` 实例的
`load_contracts`:

```python
from nautilus_trader.adapters.interactive_brokers.common import IBContract

# 加载特定到期日的期权链
options_chain_expiry = IBContract(
    secType="IND",
    symbol="SPX",
    exchange="CBOE",
    build_options_chain=True,
    lastTradeDateOrContractMonth='20240718',
)

# 加载某个日期范围内的期权链
options_chain_range = IBContract(
    secType="IND",
    symbol="SPX",
    exchange="CBOE",
    build_options_chain=True,
    min_expiry_days=0,
    max_expiry_days=30,
)

# 加载期货链
futures_chain = IBContract(
    secType="CONTFUT",
    exchange="CME",
    symbol="ES",
    build_futures_chain=True,
)

instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    load_contracts=frozenset([
        options_chain_expiry,
        options_chain_range,
        futures_chain,
    ]),
)
```

### 按资产类别的 IBContract 示例

```python
from nautilus_trader.adapters.interactive_brokers.common import IBContract

# 股票
IBContract(secType='STK', exchange='SMART', primaryExchange='ARCA', symbol='SPY')
IBContract(secType='STK', exchange='SMART', primaryExchange='NASDAQ', symbol='AAPL')

# 债券
IBContract(secType='BOND', secIdType='ISIN', secId='US03076KAA60')
IBContract(secType='BOND', secIdType='CUSIP', secId='912828XE8')

# 单个期权
IBContract(secType='OPT', exchange='SMART', symbol='SPY',
           lastTradeDateOrContractMonth='20251219', strike=500, right='C')

# 期权链(加载所有行权价/到期日)
IBContract(secType='STK', exchange='SMART', primaryExchange='ARCA', symbol='SPY',
           build_options_chain=True, min_expiry_days=10, max_expiry_days=60)

# CFD
IBContract(secType='CFD', symbol='IBUS30')
IBContract(secType='CFD', symbol='DE40EUR', exchange='SMART')

# 单个期货
IBContract(secType='FUT', exchange='CME', symbol='ES',
           lastTradeDateOrContractMonth='20240315')

# 期货链(加载所有到期日)
IBContract(secType='CONTFUT', exchange='CME', symbol='ES', build_futures_chain=True)

# 期货期权(FOP)- 单个
IBContract(secType='FOP', exchange='CME', symbol='ES',
           lastTradeDateOrContractMonth='20240315', strike=4200, right='C')

# 期货期权链(加载所有行权价/到期日)
IBContract(secType='CONTFUT', exchange='CME', symbol='ES',
           build_options_chain=True, min_expiry_days=7, max_expiry_days=60)

# 外汇
IBContract(secType='CASH', exchange='IDEALPRO', symbol='EUR', currency='USD')
IBContract(secType='CASH', exchange='IDEALPRO', symbol='GBP', currency='JPY')

# 加密货币
IBContract(secType='CRYPTO', symbol='BTC', exchange='PAXOS', currency='USD')
IBContract(secType='CRYPTO', symbol='ETH', exchange='PAXOS', currency='USD')

# 指数
IBContract(secType='IND', symbol='SPX', exchange='CBOE')
IBContract(secType='IND', symbol='NDX', exchange='NASDAQ')

# 商品
IBContract(secType='CMDTY', symbol='XAUUSD', exchange='SMART')
```

### 高级配置选项

```python
# 使用自定义交易所的期权链
IBContract(
    secType="STK",
    symbol="AAPL",
    exchange="SMART",
    primaryExchange="NASDAQ",
    build_options_chain=True,
    options_chain_exchange="CBOE",  # 期权使用 CBOE 而非 SMART
    min_expiry_days=7,
    max_expiry_days=45,
)

# 特定月份的期货链
IBContract(
    secType="CONTFUT",
    exchange="NYMEX",
    symbol="CL",  # 原油
    build_futures_chain=True,
    min_expiry_days=30,
    max_expiry_days=180,
)
```

### 连续期货

对于连续期货合约(使用 `secType='CONTFUT'`),适配器仅使用符号和
场所创建标的 ID:

```python
# 连续期货示例
IBContract(secType='CONTFUT', exchange='CME', symbol='ES')  # -> ES.CME
IBContract(secType='CONTFUT', exchange='NYMEX', symbol='CL') # -> CL.NYMEX

# 启用 MIC 场所转换后
instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    convert_exchange_to_mic_venue=True,
)
# 结果:
# ES.XCME(而非 ES.CME)
# CL.XNYM(而非 CL.NYMEX)
```

**连续期货与单个期货的对比:**

- **连续**:`ES.CME` —— 代表近月合约,自动展期
- **单个**:`ESM4.CME` —— 特定的 2024 年 3 月合约

:::note
当使用 `build_options_chain=True` 或 `build_futures_chain=True`
时,应为标的合约指定 `secType` 和 `symbol`。适配器会自动发现并加载
指定到期范围内的所有相关衍生品合约。
:::

## 期权价差

Interactive Brokers 通过 BAG 合约支持期权价差,将多条期权腿组合为
单个可交易标的。NautilusTrader 提供了创建、加载和交易期权价差的
支持。

### 创建期权价差标的 ID

期权价差使用 `new_generic_spread_id()` 函数创建,该函数将各条期权腿
与其各自的比率组合在一起:

```python
from nautilus_trader.model.identifiers import InstrumentId, new_generic_spread_id

# 创建各个期权标的 ID
call_leg = InstrumentId.from_str("SPY C400.SMART")
put_leg = InstrumentId.from_str("SPY P390.SMART")

# 创建 1:1 的看涨价差(买入看涨、卖出看涨)
call_spread_id = new_generic_spread_id([
    (call_leg, 1),   # 买入 1 份合约
    (put_leg, -1),   # 卖出 1 份合约
])

# 创建 1:2 的比率价差
ratio_spread_id = new_generic_spread_id([
    (call_leg, 1),   # 买入 1 份合约
    (put_leg, 2),    # 买入 2 份合约
])
```

### 动态价差加载

期权价差必须先被请求,然后才能交易或订阅市场数据。使用
`request_instrument()` 方法动态加载价差标的:

```python
# 在你策略的 on_start 方法中
def on_start(self):
    # 请求价差标的
    self.request_instrument(spread_id)

def on_instrument(self, instrument):
    # 处理已加载的价差标的
    self.log.info(f"Loaded spread: {instrument.id}")

    # 现在你可以订阅市场数据
    self.subscribe_quote_ticks(instrument.id)

    # 并下单
    order = self.order_factory.market(
        instrument_id=instrument.id,
        order_side=OrderSide.BUY,
        quantity=instrument.make_qty(1),
        time_in_force=TimeInForce.DAY,
    )
    self.submit_order(order)
```

### 价差交易要求

1. **先加载各条腿**:确保各条独立期权腿在创建价差之前已可用。
2. **请求价差标的**:交易之前使用 `request_instrument()` 加载价差。
3. **订阅市场数据**:价差加载完成后请求报价 tick。
4. **下单**:价差可用之后,可以使用任意订单类型。

## 历史数据与回测

`HistoricInteractiveBrokersClient` 提供了从 Interactive Brokers
获取历史数据的方法,用于回测和研究。

### 支持的数据类型

- **K 线数据**:带时间、tick 和成交量聚合的 OHLCV K 线。
- **Tick 数据**:具有微秒精度的成交 tick 和报价 tick。
- **标的数据**:完整的合约规格和交易规则。

### 历史数据客户端

```python
from nautilus_trader.adapters.interactive_brokers.historical.client import HistoricInteractiveBrokersClient
from ibapi.common import MarketDataTypeEnum

# 初始化客户端
client = HistoricInteractiveBrokersClient(
    host="127.0.0.1",
    port=7497,
    client_id=1,
    market_data_type=MarketDataTypeEnum.DELAYED_FROZEN,  # 若无订阅则使用延迟数据
    log_level="INFO"
)

# 连接到 TWS/Gateway
await client.connect()
```

### 获取标的

#### 基本标的获取

```python
from nautilus_trader.adapters.interactive_brokers.common import IBContract

# 定义合约
contracts = [
    IBContract(secType="STK", symbol="AAPL", exchange="SMART", primaryExchange="NASDAQ"),
    IBContract(secType="STK", symbol="MSFT", exchange="SMART", primaryExchange="NASDAQ"),
    IBContract(secType="CASH", symbol="EUR", currency="USD", exchange="IDEALPRO"),
]

# 请求标的定义
instruments = await client.request_instruments(contracts=contracts)
```

#### 期权链获取与 catalog 存储

你可以在策略中使用 `request_instruments` 下载整个期权链,并附带
`update_catalog=True` 将数据保存到 catalog 的好处:

```python
# 在你策略的 on_start 方法中
def on_start(self):
    self.request_instruments(
        venue=IB_VENUE,
        update_catalog=True,
        params={
            "ib_contracts": (
                # SPY 期权
                {
                    "secType": "STK",
                    "symbol": "SPY",
                    "exchange": "SMART",
                    "primaryExchange": "ARCA",
                    "build_options_chain": True,
                    "min_expiry_days": 7,
                    "max_expiry_days": 30,
                },
                # QQQ 期权
                {
                    "secType": "STK",
                    "symbol": "QQQ",
                    "exchange": "SMART",
                    "primaryExchange": "NASDAQ",
                    "build_options_chain": True,
                    "min_expiry_days": 7,
                    "max_expiry_days": 30,
                },
                # ES 期货期权
                {
                    "secType": "CONTFUT",
                    "exchange": "CME",
                    "symbol": "ES",
                    "build_options_chain": True,
                    "min_expiry_days": 0,
                    "max_expiry_days": 60,
                },
                # SPX 指数期权
                {
                    "secType": "IND",
                    "symbol": "SPX",
                    "exchange": "CBOE",
                    "build_options_chain": True,
                    "min_expiry_days": 0,
                    "max_expiry_days": 5,
                },
                # ES 期货链和期货期权
                {
                    "secType": "CONTFUT",
                    "exchange": "CME",
                    "symbol": "ES",
                    "build_futures_chain": True,
                    "build_options_chain": True,
                    "min_expiry_days": 0,
                    "max_expiry_days": 2,
                },
                # ESTX50 指数期权(Eurex)
                {
                    "secType": "IND",
                    "exchange": "EUREX",
                    "symbol": "ESTX50",
                    "build_options_chain": True,
                    "min_expiry_days": 0,
                    "max_expiry_days": 2,
                },
            ),
        },
    )
```

### 获取历史 K 线

```python
import datetime

# 请求历史 K 线
bars = await client.request_bars(
    bar_specifications=[
        "1-MINUTE-LAST",    # 1 分钟 K 线,使用最新成交价
        "5-MINUTE-MID",     # 5 分钟 K 线,使用中间价
        "1-HOUR-LAST",      # 1 小时 K 线,使用最新成交价
        "1-DAY-LAST",       # 日 K 线,使用最新成交价
    ],
    start_date_time=datetime.datetime(2023, 11, 1, 9, 30),
    end_date_time=datetime.datetime(2023, 11, 6, 16, 30),
    tz_name="America/New_York",
    contracts=contracts,
    use_rth=True,  # 仅限正常交易时段
    timeout=120,   # 请求超时时间(秒)
)
```

### 获取历史 tick 数据

```python
# 请求历史 tick 数据(使用 tick_type="TRADES" 或 "BID_ASK" 获取报价 tick)
ticks = await client.request_ticks(
    tick_type="TRADES",
    start_date_time=datetime.datetime(2023, 11, 6, 9, 30),
    end_date_time=datetime.datetime(2023, 11, 6, 16, 30),
    tz_name="America/New_York",
    contracts=contracts,
    use_rth=True,
    timeout=120,
)
```

### K 线规格

该适配器支持多种 K 线规格:

#### 基于时间的 K 线

- `"1-SECOND-LAST"`、`"5-SECOND-LAST"`、`"10-SECOND-LAST"`、
  `"15-SECOND-LAST"`、`"30-SECOND-LAST"`
- `"1-MINUTE-LAST"`、`"2-MINUTE-LAST"`、`"3-MINUTE-LAST"`、
  `"5-MINUTE-LAST"`、`"10-MINUTE-LAST"`、`"15-MINUTE-LAST"`、
  `"20-MINUTE-LAST"`、`"30-MINUTE-LAST"`
- `"1-HOUR-LAST"`、`"2-HOUR-LAST"`、`"3-HOUR-LAST"`、
  `"4-HOUR-LAST"`、`"8-HOUR-LAST"`
- `"1-DAY-LAST"`、`"1-WEEK-LAST"`、`"1-MONTH-LAST"`

#### 价格类型

- `LAST` - 最新成交价
- `MID` - 买卖中间价
- `BID` - 买价
- `ASK` - 卖价

### 完整示例

```python
import asyncio
import datetime
from nautilus_trader.adapters.interactive_brokers.common import IBContract
from nautilus_trader.adapters.interactive_brokers.historical.client import HistoricInteractiveBrokersClient
from nautilus_trader.persistence.catalog import ParquetDataCatalog


async def download_historical_data():
    # 初始化客户端
    client = HistoricInteractiveBrokersClient(
        host="127.0.0.1",
        port=7497,
        client_id=5,
    )

    # 连接
    await client.connect()
    await asyncio.sleep(2)  # 让连接稳定下来

    # 定义合约
    contracts = [
        IBContract(secType="STK", symbol="AAPL", exchange="SMART", primaryExchange="NASDAQ"),
        IBContract(secType="CASH", symbol="EUR", currency="USD", exchange="IDEALPRO"),
    ]

    # 请求标的
    instruments = await client.request_instruments(contracts=contracts)

    # 请求历史 K 线
    bars = await client.request_bars(
        bar_specifications=["1-HOUR-LAST", "1-DAY-LAST"],
        start_date_time=datetime.datetime(2023, 11, 1, 9, 30),
        end_date_time=datetime.datetime(2023, 11, 6, 16, 30),
        tz_name="America/New_York",
        contracts=contracts,
        use_rth=True,
    )

    # 请求 tick 数据
    ticks = await client.request_ticks(
        tick_type="TRADES",
        start_date_time=datetime.datetime(2023, 11, 6, 14, 0),
        end_date_time=datetime.datetime(2023, 11, 6, 15, 0),
        tz_name="America/New_York",
        contracts=contracts,
    )

    # 保存到 catalog
    catalog = ParquetDataCatalog("./catalog")
    catalog.write_data(instruments)
    catalog.write_data(bars)
    catalog.write_data(ticks)

    print(f"Downloaded {len(instruments)} instruments")
    print(f"Downloaded {len(bars)} bars")
    print(f"Downloaded {len(ticks)} ticks")

    # 断开连接
    await client.disconnect()

# 运行示例
if __name__ == "__main__":
    asyncio.run(download_historical_data())
```

### 数据限制

请留意 Interactive Brokers 历史数据的以下限制:

- **速率限制**:IB 对历史数据请求强制执行速率限制
- **数据可用性**:历史数据的可用性因标的和订阅等级而异
- **市场数据权限**:部分数据需要特定的市场数据订阅
- **时间范围**:最大回溯周期因 K 线大小和标的类型而异

### 最佳实践

1. **使用延迟数据**:对于回测,`MarketDataTypeEnum.DELAYED_FROZEN`
   通常已经足够
2. **批量请求**:尽可能在单次请求中组合多个标的
3. **处理超时**:为大型数据请求设置合适的超时值
4. **遵守速率限制**:在请求之间添加延迟以避免触及速率限制
5. **验证数据**:回测前始终检查数据质量和完整性

:::warning
Interactive Brokers 强制执行节奏限制;过多的历史数据或订单请求会
触发节奏违规,IB 可能会禁用该 API 会话数分钟。
:::

## 实盘交易

使用 Interactive Brokers 进行实盘交易,需要设置一个同时包含
`InteractiveBrokersDataClient` 和 `InteractiveBrokersExecutionClient`
的 `TradingNode`。这些客户端依赖
`InteractiveBrokersInstrumentProvider` 进行标的管理。

### 架构概览

实盘交易设置由三个主要组件构成:

1. **InstrumentProvider**:管理标的定义和合约详情
2. **DataClient**:处理实时市场数据订阅
3. **ExecutionClient**:管理订单、仓位和账户信息

### InstrumentProvider 配置

`InteractiveBrokersInstrumentProvider` 提供对 IB 金融标的数据的
访问。它支持加载单个标的、期权链和期货链。

#### 基本配置

```python
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersInstrumentProviderConfig
from nautilus_trader.adapters.interactive_brokers.config import SymbologyMethod
from nautilus_trader.adapters.interactive_brokers.common import IBContract

instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    symbology_method=SymbologyMethod.IB_SIMPLIFIED,
    build_futures_chain=False,  # 若要获取期货链,设为 True
    build_options_chain=False,  # 若要获取期权链,设为 True
    min_expiry_days=10,         # 衍生品的最小到期天数
    max_expiry_days=60,         # 衍生品的最大到期天数
    convert_exchange_to_mic_venue=False,  # 使用 MIC 代码进行场所映射
    cache_validity_days=1,      # 标的数据缓存 1 天
    load_ids=frozenset([
        # 使用简化符号规则的单个标的
        "EUR/USD.IDEALPRO",     # 外汇
        "BTC/USD.PAXOS",        # 加密货币
        "SPY.ARCA",             # 股票 ETF
        "V.NYSE",               # 单只股票
        "ESM4.CME",             # 期货合约(单位数年份)
        "^SPX.CBOE",            # 指数
    ]),
    load_contracts=frozenset([
        # 使用 IBContract 的复杂标的
        IBContract(secType='STK', symbol='AAPL', exchange='SMART', primaryExchange='NASDAQ'),
        IBContract(secType='CASH', symbol='GBP', currency='USD', exchange='IDEALPRO'),
    ]),
)
```

#### 衍生品的高级配置

```python
# 期权链和期货链的配置
advanced_config = InteractiveBrokersInstrumentProviderConfig(
    symbology_method=SymbologyMethod.IB_SIMPLIFIED,
    build_futures_chain=True,   # 启用期货链加载
    build_options_chain=True,   # 启用期权链加载
    min_expiry_days=7,          # 加载 7 天以上到期的合约
    max_expiry_days=90,         # 加载 90 天内到期的合约
    load_contracts=frozenset([
        # 加载 SPY 期权链
        IBContract(
            secType='STK',
            symbol='SPY',
            exchange='SMART',
            primaryExchange='ARCA',
            build_options_chain=True,
        ),
        # 加载 ES 期货链
        IBContract(
            secType='CONTFUT',
            exchange='CME',
            symbol='ES',
            build_futures_chain=True,
        ),
    ]),
)
```

#### 过滤证券类型

使用 `filter_sec_types` 忽略特定的 IB `secType` 值。任何 `secType`
匹配该 frozenset 中条目的合约都会被跳过,并记录一条警告(例如
`WAR` 或 `IOPT` 等不受支持的类型):

```python
instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    load_ids=frozenset(["SPY.ARCA"]),
    filter_sec_types=frozenset({"WAR", "IOPT"}),  # 排除不受支持的资产类型
)
```

### 与外部数据提供商的集成

Interactive Brokers 适配器可以与其他数据提供商配合使用,以增强市场
数据覆盖范围。使用多个数据源时:

- 在各提供商之间使用一致的符号规则方法
- 考虑使用 `convert_exchange_to_mic_venue=True` 以实现标准化的
  场所标识
- 确保标的缓存管理得当,以避免冲突

### 数据客户端配置

`InteractiveBrokersDataClient` 与 IB 交互,以流式获取和检索实时
市场数据。连接后,它会配置
[市场数据类型](https://ibkrcampus.com/ibkr-api-page/trader-workstation-api/#delayed-market-data),
并根据 `InteractiveBrokersInstrumentProviderConfig` 设置加载标的。

#### 支持的数据类型

- **报价 Tick**:实时买卖价格和数量
- **成交 Tick**:实时成交价格和成交量
- **K 线数据**:实时 OHLCV K 线(1 秒到 1 天间隔)
- **市场深度**:二档订单簿数据(如果可用)

#### 市场数据类型

Interactive Brokers 支持多种市场数据类型:

- `REALTIME`:实时市场数据(需要市场数据订阅)
- `DELAYED`:延迟 15-20 分钟的数据(大多数市场免费)
- `DELAYED_FROZEN`:不更新的延迟数据(适用于测试)
- `FROZEN`:最后已知的实时数据(市场休市时)

#### 基本数据客户端配置

```python
from nautilus_trader.adapters.interactive_brokers.config import IBMarketDataTypeEnum
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersDataClientConfig

data_client_config = InteractiveBrokersDataClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,  # TWS 模拟交易端口
    ibg_client_id=1,
    use_regular_trading_hours=True,  # 股票仅使用正常交易时段
    market_data_type=IBMarketDataTypeEnum.DELAYED_FROZEN,  # 使用延迟数据
    ignore_quote_tick_size_updates=False,  # 包含仅数量的更新
    instrument_provider=instrument_provider_config,
    connection_timeout=300,  # 5 分钟
    request_timeout_secs=60,      # 1 分钟
)
```

#### 高级数据客户端配置

```python
# 面向生产环境的实时数据配置
production_data_config = InteractiveBrokersDataClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=4001,  # IB Gateway 实盘交易端口
    ibg_client_id=1,
    use_regular_trading_hours=False,  # 包含延长交易时段
    market_data_type=IBMarketDataTypeEnum.REALTIME,  # 实时数据
    ignore_quote_tick_size_updates=True,  # 减少 tick 数量
    handle_revised_bars=True,  # 处理 K 线修订
    instrument_provider=instrument_provider_config,
    dockerized_gateway=dockerized_gateway_config,  # 如果使用 Docker
    connection_timeout=300,
    request_timeout_secs=60,
)
```

### 数据客户端配置选项

| 选项                          | 默认值                                         | 说明 |
|---------------------------------|-------------------------------------------------|-------------|
| `instrument_provider`           | `InteractiveBrokersInstrumentProviderConfig()`  | 控制启动时加载哪些合约的标的提供者设置。 |
| `ibg_host`                      | `127.0.0.1`                                     | TWS/IB Gateway 的主机名或 IP。 |
| `ibg_port`                      | `None`                                          | TWS/IB Gateway 的端口(TWS 为 `7497`/`7496`,IBG 为 `4002`/`4001`)。 |
| `ibg_client_id`                 | `1`                                             | 连接 TWS/IB Gateway 时使用的唯一客户端标识符。 |
| `use_regular_trading_hours`     | `True`                                          | 为 `True` 时,请求限定在正常交易时段内的 K 线。 |
| `market_data_type`              | `REALTIME`                                      | 市场数据流类型(`REALTIME`、`DELAYED`、`DELAYED_FROZEN` 等)。 |
| `ignore_quote_tick_size_updates`| `False`                                         | 为 `True` 时,抑制仅数量变化的报价 tick。 |
| `handle_revised_bars`           | `False`                                         | 为 `True` 时,处理来自 IB 的 K 线修订(K 线可能在首次发布后被更新)。 |
| `dockerized_gateway`            | `None`                                          | 用于容器化设置的可选 `DockerizedIBGatewayConfig`。 |
| `connection_timeout`            | `300`                                           | 等待初始 API 连接的时间(秒)。 |
| `request_timeout_secs`          | `60`                                            | 历史数据请求超时前的等待时间(秒)。 |

#### 备注

- **`use_regular_trading_hours`**:为 `True` 时,仅在正常交易时段
  请求数据。主要影响股票的 K 线数据。
- **`ignore_quote_tick_size_updates`**:为 `True` 时,过滤掉仅数量
  变化(而非价格)的报价 tick,减少数据量。
- **`handle_revised_bars`**:为 `True` 时,处理来自 IB 的 K 线修订
  (K 线可能在首次发布后被更新)。
- **`connection_timeout`**:等待初始连接建立的最长时间。
- **`request_timeout_secs`**:等待历史数据请求的最长时间。

### 执行客户端配置选项

| 选项                                  | 默认值                                         | 说明 |
|-----------------------------------------|-------------------------------------------------|-------------|
| `instrument_provider`                   | `InteractiveBrokersInstrumentProviderConfig()`  | 控制启动时加载哪些合约的标的提供者设置。 |
| `ibg_host`                              | `127.0.0.1`                                     | TWS/IB Gateway 的主机名或 IP。 |
| `ibg_port`                              | `None`                                          | TWS/IB Gateway 的端口(TWS 为 `7497`/`7496`,IBG 为 `4002`/`4001`)。 |
| `ibg_client_id`                         | `1`                                             | 连接 TWS/IB Gateway 时使用的唯一客户端标识符。 |
| `account_id`                            | `None`                                          | Interactive Brokers 账户标识符(回退到 `TWS_ACCOUNT` 环境变量)。 |
| `dockerized_gateway`                    | `None`                                          | 用于容器化设置的可选 `DockerizedIBGatewayConfig`。 |
| `connection_timeout`                    | `300`                                           | 等待初始 API 连接的时间(秒)。 |
| `request_timeout_secs`                  | `60`                                            | 等待请求响应(合约详情等)的时间(秒)。 |
| `fetch_all_open_orders`                 | `False`                                         | 为 `True` 时,拉取每一个 API 客户端 ID 的未成交订单(而非仅本会话)。 |
| `track_option_exercise_from_position_update` | `False`                                    | 为 `True` 时,订阅实时仓位更新以检测期权行权。 |

### 执行客户端配置

`InteractiveBrokersExecutionClient` 处理交易执行、订单管理、账户
信息和仓位跟踪。它提供订单生命周期管理和实时账户更新。

#### 支持的功能

- **订单管理**:下单、修改和取消
- **订单类型**:市价、限价、止损、止损限价、跟踪止损等
- **账户信息**:实时余额和保证金更新
- **仓位跟踪**:实时仓位更新和盈亏
- **成交报告**:执行报告和成交通知
- **风险管理**:交易前风险检查和仓位限额

#### 支持的订单类型

该适配器支持大多数 Interactive Brokers 订单类型:

- **市价单**:`OrderType.MARKET`
- **限价单**:`OrderType.LIMIT`
- **止损单**:`OrderType.STOP_MARKET`
- **止损限价单**:`OrderType.STOP_LIMIT`
- **触及市价单**:`OrderType.MARKET_IF_TOUCHED`
- **触及限价单**:`OrderType.LIMIT_IF_TOUCHED`
- **跟踪止损市价单**:`OrderType.TRAILING_STOP_MARKET`
- **跟踪止损限价单**:`OrderType.TRAILING_STOP_LIMIT`
- **收盘市价单**:带 `TimeInForce.AT_THE_CLOSE` 的 `OrderType.MARKET`
- **收盘限价单**:带 `TimeInForce.AT_THE_CLOSE` 的 `OrderType.LIMIT`

#### 有效期选项

- **当日订单**:`TimeInForce.DAY`
- **撤销前有效**:`TimeInForce.GTC`
- **立即成交或取消**:`TimeInForce.IOC`
- **全部成交或取消**:`TimeInForce.FOK`
- **指定日期前有效**:`TimeInForce.GTD`
- **开盘时**:`TimeInForce.AT_THE_OPEN`
- **收盘时**:`TimeInForce.AT_THE_CLOSE`

#### 批量操作

| 操作          | 是否支持 | 备注                                        |
|--------------------|-----------|----------------------------------------------|
| 批量提交       | ✓         | 在单次请求中提交多个订单。    |
| 批量修改       | ✓         | 在单次请求中修改多个订单。    |
| 批量取消       | ✓         | 在单次请求中取消多个订单。    |

#### 仓位管理

| 功能              | 是否支持 | 备注                                        |
|--------------------|-----------|----------------------------------------------|
| 查询仓位     | ✓         | 实时仓位更新。                  |
| 仓位模式       | ✓         | 净额与分开的多空仓位。       |
| 杠杆控制    | ✓         | 账户级别的保证金要求。          |
| 保证金模式         | ✓         | 组合保证金与单笔保证金。             |

#### 订单查询

| 功能              | 是否支持 | 备注                                        |
|--------------------|-----------|----------------------------------------------|
| 查询未成交订单   | ✓         | 列出所有活跃订单。                      |
| 查询订单历史 | ✓         | 历史订单数据。                       |
| 订单状态更新| ✓         | 实时订单状态变化。              |
| 成交历史       | ✓         | 执行与成交报告。                 |

#### 条件单

| 功能              | 是否支持 | 备注                                        |
|--------------------|-----------|----------------------------------------------|
| 订单列表         | ✓         | 原子性的多订单提交。               |
| OCO 订单          | ✓         | 一取消另一,支持可自定义的 OCA 类型(1、2、3)。 |
| 括号单      | ✓         | 父子订单关系。 |
| 条件单  | ✓         | 高级订单条件和触发。     |

#### 基本执行客户端配置

```python
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersExecClientConfig
from nautilus_trader.config import RoutingConfig

exec_client_config = InteractiveBrokersExecClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,  # TWS 模拟交易端口
    ibg_client_id=1,
    account_id="DU123456",  # 你的 IB 账户 ID(模拟或实盘)
    instrument_provider=instrument_provider_config,
    connection_timeout=300,
    routing=RoutingConfig(default=True),  # 通过该客户端路由所有订单
)
```

#### 高级执行客户端配置

```python
# 使用 docker 化网关的生产环境配置
production_exec_config = InteractiveBrokersExecClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=4001,  # IB Gateway 实盘交易端口
    ibg_client_id=1,
    account_id=None,  # 将使用 TWS_ACCOUNT 环境变量
    instrument_provider=instrument_provider_config,
    dockerized_gateway=dockerized_gateway_config,
    connection_timeout=300,
    routing=RoutingConfig(default=True),
)
```

#### 账户 ID 配置

`account_id` 参数至关重要,必须与登录 TWS/Gateway 的账户匹配:

```python
# 选项 1:直接在配置中指定
exec_config = InteractiveBrokersExecClientConfig(
    account_id="DU123456",  # 模拟交易账户
    # ... 其他参数
)

# 选项 2:使用环境变量
import os
os.environ["TWS_ACCOUNT"] = "DU123456"
exec_config = InteractiveBrokersExecClientConfig(
    account_id=None,  # 将使用 TWS_ACCOUNT 环境变量
    # ... 其他参数
)
```

#### 订单参数

执行适配器在下单、订单列表提交和修改订单命令上支持
`params["exchange"]`。使用它可以为当前订单覆盖 IB 合约交易所以进行
路由,同时保留缓存的标的合约:

```python
self.submit_order(order, params={"exchange": "IEX"})
```

未设置 `exchange`,或将其设为空字符串,则使用缓存的合约交易所。

#### 订单标签与高级功能

该适配器通过订单标签支持 IB 特有的订单参数:

```python
from nautilus_trader.adapters.interactive_brokers.common import IBOrderTags

# 创建带 IB 特定参数的订单
order_tags = IBOrderTags(
    allOrNone=True,           # 全部成交或不成交
    ocaGroup="MyGroup1",      # 一取消所有组
    ocaType=1,                # 带阻塞取消
    activeStartTime="20240315 09:30:00 EST",  # GTC 激活时间
    activeStopTime="20240315 16:00:00 EST",   # GTC 失效时间
    goodAfterTime="20240315 09:35:00 EST",    # 生效起始时间
)

# 将标签应用于订单
order = order_factory.limit(
    instrument_id=instrument.id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(100),
    price=instrument.make_price(100.0),
    tags=[order_tags.value],
)
```

#### OCA(一取消所有)订单

该适配器通过使用 `IBOrderTags` 的显式配置提供对 OCA 订单的支持:

### 基本 OCA 配置

所有 OCA 功能都必须通过 `IBOrderTags` 显式配置:

```python
from nautilus_trader.adapters.interactive_brokers.common import IBOrderTags

# 创建 OCA 配置
oca_tags = IBOrderTags(
    ocaGroup="MY_OCA_GROUP",
    ocaType=1,  # 类型 1:带阻塞的全部取消(推荐)
)

# 应用于括号单
bracket_order = order_factory.bracket(
    instrument_id=instrument.id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(100),
    tp_price=instrument.make_price(110.0),
    sl_trigger_price=instrument.make_price(90.0),
    tp_tags=[oca_tags.value],  # 必须显式添加 OCA 标签
    sl_tags=[oca_tags.value],  # 必须显式添加 OCA 标签
)
```

### 高级 OCA 配置

你可以使用 `IBOrderTags` 指定不同的 OCA 类型和行为:

```python
from nautilus_trader.adapters.interactive_brokers.common import IBOrderTags

# 创建自定义 OCA 配置
custom_oca_tags = IBOrderTags(
    ocaGroup="MY_CUSTOM_GROUP",
    ocaType=2,  # 使用类型 2:带阻塞的减少
)

# 应用于单个订单
order = order_factory.limit(
    instrument_id=instrument.id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(100),
    price=instrument.make_price(100.0),
    tags=[custom_oca_tags.value],
)
```

### OCA 类型

Interactive Brokers 支持三种 OCA 类型:

| 类型 | 名称 | 行为 | 使用场景 |
|------|------|----------|----------|
| **1** | 带阻塞的全部取消 | 取消所有剩余订单,并带有阻塞保护 | **默认** —— 最安全的选项,防止超额成交 |
| **2** | 带阻塞的减少 | 按比例减少剩余订单,并带有阻塞保护 | 部分成交且带超额成交保护 |
| **3** | 不带阻塞的减少 | 按比例减少剩余订单,不带阻塞保护 | 最快执行,超额成交风险更高 |

#### 同一 OCA 组中的多个订单

```python
# 创建多个具有相同 OCA 组的订单
oca_tags = IBOrderTags(
    ocaGroup="MULTI_ORDER_GROUP",
    ocaType=3,  # 使用类型 3:不带阻塞的减少
)

order1 = order_factory.limit(
    instrument_id=instrument.id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(50),
    price=instrument.make_price(99.0),
    tags=[oca_tags.value],
)

order2 = order_factory.limit(
    instrument_id=instrument.id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(50),
    price=instrument.make_price(101.0),
    tags=[oca_tags.value],
)
```

### OCA 配置要求

OCA 功能**仅**通过显式配置提供:

1. **需要 IBOrderTags** —— OCA 设置必须在订单标签中显式指定
2. **无自动检测** —— `ContingencyType.OCO` 和 `ContingencyType.OUO`
   不会自动创建 OCA 组
3. **手动配置** —— 所有 OCA 组和类型都必须手动指定

### 条件单

该适配器通过 `IBOrderTags` 中的 `conditions` 参数支持 Interactive
Brokers 的条件单。条件单允许你指定在订单被传输或取消之前必须满足的
条件。

#### 支持的条件类型

- **价格条件**:基于特定标的的价格变动触发
- **时间条件**:在特定日期和时间触发
- **成交量条件**:基于成交量阈值触发
- **执行条件**:当某个特定标的发生成交时触发
- **保证金条件**:基于账户保证金水平触发
- **百分比变化条件**:基于价格百分比变化触发

#### 基本条件单示例

```python
from nautilus_trader.adapters.interactive_brokers.common import IBOrderTags

# 创建一个价格条件:当 SPY 超过 $250 时触发
price_condition = {
    "type": "price",
    "conId": 265598,  # SPY 合约 ID
    "exchange": "SMART",
    "isMore": True,  # 价格高于阈值时触发
    "price": 250.00,
    "triggerMethod": 0,  # 默认触发方式
    "conjunction": "and",
}

# 创建带条件的订单标签
order_tags = IBOrderTags(
    conditions=[price_condition],
    conditionsCancelOrder=False,  # 条件满足时传输订单
)

# 应用于订单
order = order_factory.limit(
    instrument_id=instrument.id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(100),
    price=instrument.make_price(251.00),
    tags=[order_tags.value],
)
```

#### 带逻辑运算的多个条件

```python
# 使用 AND/OR 逻辑创建多个条件
conditions = [
    {
        "type": "price",
        "conId": 265598,
        "exchange": "SMART",
        "isMore": True,
        "price": 250.00,
        "triggerMethod": 0,
        "conjunction": "and",  # 与下一条件进行 AND 运算
    },
    {
        "type": "time",
        "time": "20250315-09:30:00",
        "isMore": True,
        "conjunction": "or",  # 与下一条件进行 OR 运算
    },
    {
        "type": "volume",
        "conId": 265598,
        "exchange": "SMART",
        "isMore": True,
        "volume": 10000000,
        "conjunction": "and",
    },
]

order_tags = IBOrderTags(
    conditions=conditions,
    conditionsCancelOrder=False,
)
```

#### 条件参数

**价格条件:**

- `conId`:要监控标的的合约 ID
- `exchange`:要监控的交易所(例如 "SMART"、"NASDAQ")
- `isMore`:True 表示 >=,False 表示 <=
- `price`:价格阈值
- `triggerMethod`:0=默认,1=DoubleBidAsk,2=Last,3=DoubleLast,
  4=BidAsk,7=LastBidAsk,8=MidPoint

**时间条件:**

- `time`:UTC 格式的时间字符串 "YYYYMMDD-HH:MM:SS"(例如
  "20250315-09:30:00")
- `isMore`:True 表示该时间之后,False 表示该时间之前

**成交量条件:**

- `conId`:要监控标的的合约 ID
- `exchange`:要监控的交易所
- `isMore`:True 表示 >=,False 表示 <=
- `volume`:成交量阈值

**执行条件:**

- `symbol`:要监控成交的符号
- `secType`:证券类型(例如 "STK"、"OPT"、"FUT")
- `exchange`:要监控的交易所

**保证金条件:**

- `percent`:保证金缓冲百分比阈值
- `isMore`:True 表示 >=,False 表示 <=

**百分比变化条件:**

- `conId`:要监控标的的合约 ID
- `exchange`:要监控的交易所
- `isMore`:True 表示 >=,False 表示 <=
- `changePercent`:百分比变化阈值

#### 完整示例:所有条件类型

```python
# 展示全部 6 种支持的条件类型的示例
from nautilus_trader.adapters.interactive_brokers.common import IBOrderTags

# 1. 价格条件 - 当 ES 期货 > 6000 时触发
price_condition = {
    "type": "price",
    "conId": 495512563,  # ES 期货合约 ID
    "exchange": "CME",
    "isMore": True,
    "price": 6000.0,
    "triggerMethod": 0,
    "conjunction": "and",
}

# 2. 时间条件 - 在特定时间触发
time_condition = {
    "type": "time",
    "time": "20250315-09:30:00",  # UTC 格式
    "isMore": True,
    "conjunction": "and",
}

# 3. 成交量条件 - 当成交量 > 100,000 时触发
volume_condition = {
    "type": "volume",
    "conId": 495512563,
    "exchange": "CME",
    "isMore": True,
    "volume": 100000,
    "conjunction": "and",
}

# 4. 执行条件 - 当 SPY 发生成交时触发
execution_condition = {
    "type": "execution",
    "symbol": "SPY",
    "secType": "STK",
    "exchange": "SMART",
    "conjunction": "and",
}

# 5. 保证金条件 - 当保证金缓冲 > 75% 时触发
margin_condition = {
    "type": "margin",
    "percent": 75,
    "isMore": True,
    "conjunction": "and",
}

# 6. 百分比变化条件 - 当价格变化 > 5% 时触发
percent_change_condition = {
    "type": "percent_change",
    "conId": 495512563,
    "exchange": "CME",
    "changePercent": 5.0,
    "isMore": True,
    "conjunction": "and",
}

# 使用任意条件组合
order_tags = IBOrderTags(
    conditions=[price_condition, time_condition],  # 多个条件
    conditionsCancelOrder=False,  # 条件满足时传输
)
```

#### 订单行为

设置 `conditionsCancelOrder` 来控制条件满足时的行为:

- `False`:条件满足时传输订单
- `True`:条件满足时取消订单

#### 实现说明

- **全部 6 种条件类型均已完整支持**,并在实盘 Interactive
  Brokers 订单中经过测试
- **价格条件**能够正常工作,尽管 ibapi 库中存在一个已知 bug,即
  `PriceCondition.__str__` 被错误地装饰为一个属性
- **时间条件**使用带连字符分隔符的 UTC 格式
  (`YYYYMMDD-HH:MM:SS`)以确保可靠解析
- **逻辑连接**允许使用 “and”/“or” 运算符组合复杂条件

### 完整的交易节点配置

搭建一个完整的交易环境需要配置一个包含所有必要组件的
`TradingNodeConfig`。以下是不同场景的示例。

#### 模拟交易配置

```python
import os
from nautilus_trader.adapters.interactive_brokers.common import IB
from nautilus_trader.adapters.interactive_brokers.common import IB_VENUE
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersDataClientConfig
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersExecClientConfig
from nautilus_trader.adapters.interactive_brokers.config import InteractiveBrokersInstrumentProviderConfig
from nautilus_trader.adapters.interactive_brokers.config import IBMarketDataTypeEnum
from nautilus_trader.adapters.interactive_brokers.config import SymbologyMethod
from nautilus_trader.adapters.interactive_brokers.factories import InteractiveBrokersLiveDataClientFactory
from nautilus_trader.adapters.interactive_brokers.factories import InteractiveBrokersLiveExecClientFactory
from nautilus_trader.config import LiveDataEngineConfig
from nautilus_trader.config import LoggingConfig
from nautilus_trader.config import RoutingConfig
from nautilus_trader.config import TradingNodeConfig
from nautilus_trader.live.node import TradingNode

# 标的提供者配置
instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    symbology_method=SymbologyMethod.IB_SIMPLIFIED,
    load_ids=frozenset([
        "EUR/USD.IDEALPRO",
        "GBP/USD.IDEALPRO",
        "SPY.ARCA",
        "QQQ.NASDAQ",
        "AAPL.NASDAQ",
        "MSFT.NASDAQ",
    ]),
)

# 数据客户端配置
data_client_config = InteractiveBrokersDataClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,  # TWS 模拟交易
    ibg_client_id=1,
    use_regular_trading_hours=True,
    market_data_type=IBMarketDataTypeEnum.DELAYED_FROZEN,
    instrument_provider=instrument_provider_config,
)

# 执行客户端配置
exec_client_config = InteractiveBrokersExecClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,  # TWS 模拟交易
    ibg_client_id=1,
    account_id="DU123456",  # 你的模拟交易账户
    instrument_provider=instrument_provider_config,
    routing=RoutingConfig(default=True),
)

# 交易节点配置
config_node = TradingNodeConfig(
    trader_id="PAPER-TRADER-001",
    logging=LoggingConfig(log_level="INFO"),
    data_clients={IB: data_client_config},
    exec_clients={IB: exec_client_config},
    data_engine=LiveDataEngineConfig(
        time_bars_timestamp_on_close=False,  # IB 标准:使用 K 线开盘时间
        validate_data_sequence=True,         # 丢弃乱序的 K 线
    ),
    timeout_connection=90.0,
    timeout_reconciliation=5.0,
    timeout_portfolio=5.0,
    timeout_disconnection=5.0,
    timeout_post_stop=2.0,
)

# 创建并配置交易节点
node = TradingNode(config=config_node)
node.add_data_client_factory(IB, InteractiveBrokersLiveDataClientFactory)
node.add_exec_client_factory(IB, InteractiveBrokersLiveExecClientFactory)
node.build()

if __name__ == "__main__":
    try:
        node.run()
    finally:
        node.dispose()
```

当 `validate_data_sequence=True` 时,请通过 `request_bars()` 的
`callback` 订阅实时 K 线,以便数据流仅在历史数据加载完成后才开始;
参见
[使用 K 线:请求与订阅](../concepts/data/index.md#working-with-bars-request-vs-subscribe)。

## 使用 Docker 化网关进行实盘交易

```python
from nautilus_trader.adapters.interactive_brokers.config import DockerizedIBGatewayConfig

# Docker 化网关配置
dockerized_gateway_config = DockerizedIBGatewayConfig(
    username=os.environ.get("TWS_USERNAME"),
    password=os.environ.get("TWS_PASSWORD"),
    trading_mode="live",  # "paper" 或 "live"
    read_only_api=False,  # 允许订单执行
    timeout=300,
)

# 带 docker 化网关的数据客户端
data_client_config = InteractiveBrokersDataClientConfig(
    ibg_client_id=1,
    use_regular_trading_hours=False,  # 包含延长交易时段
    market_data_type=IBMarketDataTypeEnum.REALTIME,
    instrument_provider=instrument_provider_config,
    dockerized_gateway=dockerized_gateway_config,
)

# 带 docker 化网关的执行客户端
exec_client_config = InteractiveBrokersExecClientConfig(
    ibg_client_id=1,
    account_id=os.environ.get("TWS_ACCOUNT"),  # 实盘账户 ID
    instrument_provider=instrument_provider_config,
    dockerized_gateway=dockerized_gateway_config,
    routing=RoutingConfig(default=True),
)

# 实盘交易节点配置
config_node = TradingNodeConfig(
    trader_id="LIVE-TRADER-001",
    logging=LoggingConfig(log_level="INFO"),
    data_clients={IB: data_client_config},
    exec_clients={IB: exec_client_config},
    data_engine=LiveDataEngineConfig(
        time_bars_timestamp_on_close=False,
        validate_data_sequence=True,
    ),
)
```

### 多客户端配置

对于高级设置,你可以配置多个用途不同的客户端:

```python
# 使用不同客户端 ID 的独立数据和执行客户端
data_client_config = InteractiveBrokersDataClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,
    ibg_client_id=1,  # 数据客户端使用 ID 1
    market_data_type=IBMarketDataTypeEnum.REALTIME,
    instrument_provider=instrument_provider_config,
)

exec_client_config = InteractiveBrokersExecClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,
    ibg_client_id=2,  # 执行客户端使用 ID 2
    account_id="DU123456",
    instrument_provider=instrument_provider_config,
    routing=RoutingConfig(default=True),
)
```

### 面向不同账户的多个 IB 执行客户端

NautilusTrader 支持同时使用多个 Interactive Brokers 执行客户端,
每个客户端连接到不同的 IB 账户。这在你需要使用多个账户交易时非常
有用,例如:

- 为不同策略使用独立的账户
- 同时运行模拟交易和实盘交易账户
- 同一 IB 登录下的多个托管账户

要配置多个 IB 执行客户端,请在 `exec_clients` 字典中提供多个带唯一
键的条目。每个条目指定不同的 `account_id`:

```python
from nautilus_trader.adapters.interactive_brokers.config import (
    InteractiveBrokersDataClientConfig,
    InteractiveBrokersExecClientConfig,
    InteractiveBrokersInstrumentProviderConfig,
    SymbologyMethod,
    IBMarketDataTypeEnum,
)
from nautilus_trader.live.config import TradingNodeConfig, RoutingConfig, LoggingConfig
from nautilus_trader.model.identifiers import AccountId, Venue, ClientId

# 共享的标的提供者配置
instrument_provider_config = InteractiveBrokersInstrumentProviderConfig(
    symbology_method=SymbologyMethod.IB_SIMPLIFIED,
)

# 数据客户端(在所有账户之间共享)
data_client_config = InteractiveBrokersDataClientConfig(
    ibg_host="127.0.0.1",
    ibg_port=7497,
    ibg_client_id=1,
    market_data_type=IBMarketDataTypeEnum.REALTIME,
    instrument_provider=instrument_provider_config,
)

# 多个 IB 执行客户端的配置
config_node = TradingNodeConfig(
    trader_id="MULTI-ACCOUNT-001",
    logging=LoggingConfig(log_level="INFO"),

    # 跨账户共享的单一数据客户端
    data_clients={
        "IB": data_client_config,
    },

    # 每个账户一个的多个执行客户端
    exec_clients={
        # 第一个账户:模拟交易账户
        "IB-PAPER": InteractiveBrokersExecClientConfig(
            ibg_host="127.0.0.1",
            ibg_port=7497,
            ibg_client_id=2,  # 唯一的 IB API 客户端 ID
            account_id="DU123456",  # 模拟交易账户 ID
            instrument_provider=instrument_provider_config,
            routing=RoutingConfig(default=False),  # 非默认
        ),

        # 第二个账户:实盘交易账户
        "IB-LIVE": InteractiveBrokersExecClientConfig(
            ibg_host="127.0.0.1",
            ibg_port=7497,
            ibg_client_id=3,  # 唯一的 IB API 客户端 ID
            account_id="U987654",  # 实盘账户 ID
            instrument_provider=instrument_provider_config,
            routing=RoutingConfig(default=True),  # 设为默认
        ),

        # 第三个账户:另一个托管账户
        "IB-ACCOUNT3": InteractiveBrokersExecClientConfig(
            ibg_host="127.0.0.1",
            ibg_port=7497,
            ibg_client_id=4,  # 唯一的 IB API 客户端 ID
            account_id="U456789",  # 另一个账户 ID
            instrument_provider=instrument_provider_config,
            routing=RoutingConfig(default=False),
        ),
    },
)
```

**多个 IB 执行客户端的要点:**

1. **唯一的键**:`exec_clients` 中的每个条目都必须有一个唯一的键
   (例如 `"IB-PAPER"`、`"IB-LIVE"`)。该键会成为该客户端的
   `account_issuer`。

2. **唯一的客户端 ID**:每个执行客户端都必须使用不同的
   `ibg_client_id`(2、3、4 等)。IB Gateway/TWS 要求每个 API
   连接都使用唯一的客户端 ID。

3. **账户 ID**:每个执行客户端都必须指定不同的 `account_id`,
   与登录到 IB Gateway/TWS 的账户匹配。

4. **账户标识符**:系统会创建类似以下形式的 `AccountId` 实例:
   - `AccountId("IB-PAPER-DU123456")`
   - `AccountId("IB-LIVE-U987654")`
   - `AccountId("IB-ACCOUNT3-U456789")`

5. **路由**:订单和查询会根据以下条件自动路由到正确的执行客户端:
   - 命令中显式的 `client_id`
   - `account_id` 的 issuer(用于 `QueryAccount` 命令,或设置了
     account_id 的订单)
   - 默认客户端(如果某个客户端标记为
     `routing=RoutingConfig(default=True)`)

6. **组合查询**:查询组合属性时,你可以指定:
   - `account_id` 用于特定账户的查询:
     `portfolio.realized_pnls(account_id=AccountId("IB-PAPER-DU123456"))`
   - `venue` 用于跨具有该场所的所有账户的聚合查询:
     `portfolio.realized_pnls(venue=Venue("IB-PAPER"))`

**示例:在策略中使用多个 IB 执行客户端:**

```python
from nautilus_trader.model.identifiers import AccountId, ClientId
from nautilus_trader.trading.strategy import Strategy

class MultiAccountStrategy(Strategy):
    """使用多个 IB 账户的示例策略。"""

    def on_start(self):
        # 定义账户 ID 以便引用
        self.paper_account = AccountId("IB-PAPER-DU123456")
        self.live_account = AccountId("IB-LIVE-U987654")

        # 查询模拟账户余额
        paper_account_state = self.cache.account(self.paper_account)
        if paper_account_state:
            self.log.info(f"Paper account balance: {paper_account_state.balance_total()}")

        # 查询实盘账户余额
        live_account_state = self.cache.account(self.live_account)
        if live_account_state:
            self.log.info(f"Live account balance: {live_account_state.balance_total()}")

    def submit_order_to_paper(self, order):
        """向模拟交易账户提交订单。"""
        self.submit_order(order, client_id=ClientId("IB-PAPER"))

    def submit_order_to_live(self, order):
        """向实盘交易账户提交订单。"""
        self.submit_order(order, client_id=ClientId("IB-LIVE"))

    def check_paper_pnl(self, instrument_id):
        """检查模拟账户的已实现盈亏。"""
        pnl = self.portfolio.realized_pnl(
            instrument_id=instrument_id,
            account_id=self.paper_account
        )
        return pnl

    def check_live_pnl(self, instrument_id):
        """检查实盘账户的已实现盈亏。"""
        pnl = self.portfolio.realized_pnl(
            instrument_id=instrument_id,
            account_id=self.live_account
        )
        return pnl
```

**示例:使用多个 IB 客户端查询账户信息:**

```python
from nautilus_trader.model.identifiers import AccountId

# 查询特定账户
paper_account = cache.account(AccountId("IB-PAPER-DU123456"))
live_account = cache.account(AccountId("IB-LIVE-U987654"))

# 使用 account_id 查询账户(推荐方式)
paper_account_by_id = cache.account(AccountId("IB-PAPER-DU123456"))

# 备选方式:使用 account_id 参数查询账户(同样有效)
paper_account_via_account_id = cache.account_for_venue(
    account_id=AccountId("IB-PAPER-DU123456")
)

# 按账户查询组合属性
paper_realized_pnl = portfolio.realized_pnl(
    instrument_id=instrument_id,
    account_id=AccountId("IB-PAPER-DU123456")
)

# 查询跨所有 IB 账户聚合的组合属性
# 注意:这会聚合所有具有相同场所的账户
all_ib_realized_pnl = portfolio.realized_pnls(venue=Venue("IB"))
```

### 运行交易节点

```python
def run_trading_node():
    """运行带有正确错误处理的交易节点。"""
    node = None
    try:
        # 创建并构建节点
        node = TradingNode(config=config_node)
        node.add_data_client_factory(IB, InteractiveBrokersLiveDataClientFactory)
        node.add_exec_client_factory(IB, InteractiveBrokersLiveExecClientFactory)
        node.build()

        # 在此添加你的策略
        # node.trader.add_strategy(YourStrategy())

        # 运行节点
        node.run()

    except KeyboardInterrupt:
        print("Shutting down...")
    except Exception as e:
        print(f"Error: {e}")
    finally:
        if node:
            node.dispose()

if __name__ == "__main__":
    run_trading_node()
```

### 附加配置选项

#### 环境变量

设置以下环境变量以便于配置:

```bash
export TWS_USERNAME="your_ib_username"
export TWS_PASSWORD="your_ib_password"
export TWS_ACCOUNT="your_account_id"
export IB_MAX_CONNECTION_ATTEMPTS="5"  # 可选:限制重连尝试次数
```

#### 日志配置

```python
# 增强的日志配置
logging_config = LoggingConfig(
    log_level="INFO",
    log_level_file="DEBUG",
    log_file_format="json",  # 用于结构化日志的 JSON 格式
    log_component_levels={
        "InteractiveBrokersClient": "DEBUG",
        "InteractiveBrokersDataClient": "INFO",
        "InteractiveBrokersExecutionClient": "INFO",
    },
)
```

更多示例可以在这里找到:<https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/interactive_brokers>

## 故障排查

### 常见连接问题

#### 连接被拒绝

- **原因**:TWS/Gateway 未运行,或端口错误
- **解决方法**:验证 TWS/Gateway 是否正在运行,并检查端口配置
- **默认端口**:TWS(7497/7496),IB Gateway(4002/4001)

#### 身份验证错误

- **原因**:凭证不正确,或账户未登录
- **解决方法**:验证用户名/密码,并确保账户已登录到 TWS/Gateway

#### 客户端 ID 冲突

- **原因**:多个客户端使用相同的客户端 ID
- **解决方法**:为每个连接使用唯一的客户端 ID

#### 市场数据权限

- **原因**:市场数据订阅不足
- **解决方法**:测试时使用 `IBMarketDataTypeEnum.DELAYED_FROZEN`,
  或订阅所需的数据流

### 错误码

Interactive Brokers 使用特定的错误码。常见的包括:

- **200**:未找到证券定义
- **201**:订单被拒绝 —— 原因如下
- **202**:订单已取消
- **300**:无法通过 ticker ID 找到 EId
- **354**:请求的市场数据未订阅
- **2104**:市场数据农场连接正常
- **2106**:HMDS 数据农场连接正常

### 性能优化

#### 减少数据量

```python
# 通过忽略仅数量的更新减少报价 tick 数量
data_config = InteractiveBrokersDataClientConfig(
    ignore_quote_tick_size_updates=True,
    # ... 其他配置
)
```

#### 连接管理

```python
# 设置合理的超时值
config = InteractiveBrokersDataClientConfig(
    connection_timeout=300,  # 5 分钟
    request_timeout_secs=60,      # 1 分钟
    # ... 其他配置
)
```

#### 内存管理

- 为你的策略使用合适的 K 线大小
- 限制同时订阅的数量
- 考虑使用历史数据进行回测,而非实时数据

### 最佳实践

#### 安全

- 切勿在源代码中硬编码凭证
- 对敏感信息使用环境变量
- 在开发和测试中使用模拟交易
- 对仅数据应用设置 `read_only_api=True`

#### 开发工作流

1. **从模拟交易开始**:始终先用模拟交易测试
2. **使用延迟数据**:开发时使用 `DELAYED_FROZEN` 市场数据
3. **实现适当的错误处理**:优雅地处理连接丢失和 API 错误
4. **监控日志**:为调试启用适当的日志级别
5. **测试重连**:测试你的策略在连接中断期间的行为

#### 生产部署

- 对自动化部署使用 docker 化网关
- 实现适当的监控和告警
- 建立日志聚合与分析
- 只在必要时使用实时数据订阅
- 实现熔断机制和仓位限额

#### 订单管理

- 提交前始终验证订单
- 实现适当的仓位规模控制
- 为你的策略使用合适的订单类型
- 监控订单状态并处理拒绝
- 为订单操作实现超时处理

### 调试技巧

#### 启用调试日志

```python
logging_config = LoggingConfig(
    log_level="DEBUG",
    log_component_levels={
        "InteractiveBrokersClient": "DEBUG",
    },
)
```

#### 监控连接状态

```python
# 在策略中检查连接状态
if not self.data_client.is_connected:
    self.log.warning("Data client disconnected")
```

#### 验证标的

```python
# 确保在交易之前已加载标的
instruments = self.cache.instruments()
if not instruments:
    self.log.error("No instruments loaded")
```

### 支持与资源

- **IB API 文档**:[TWS API 指南](https://ibkrcampus.com/ibkr-api-page/trader-workstation-api/)
- **NautilusTrader 示例**:[GitHub 示例](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/interactive_brokers)
- **IB 合约搜索**:[合约信息中心](https://pennies.interactivebrokers.com/cstools/contract_info/)
- **市场数据订阅**:[IB 市场数据](https://www.interactivebrokers.com/en/pricing/market-data-pricing.php)

## 贡献

:::info
如需了解更多功能或为 Interactive Brokers 适配器贡献代码,请参阅
我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
