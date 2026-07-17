# AX Exchange

[AX Exchange](https://architect.exchange) 是全球首个针对传统底层资产类别提供永续期货
交易的中心化、受监管交易所。由 Architect Bermuda Ltd. 运营,并由
[百慕大金融管理局(BMA)](https://www.bma.bm/) 颁发牌照,AX 将加密货币式的
永续合约带入传统金融市场,涵盖外汇、贵金属、能源、股票指数和利率等品类。

该集成支持接入 AX Exchange 的实时市场数据以及订单执行。

## 示例

可以在[这里](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/architect_ax/)找到实时示例脚本。

## 概览

本指南假设交易者需要同时配置实时市场数据源和交易执行。AX Exchange
适配器包含多个组件,可以单独使用,也可以组合使用,视具体使用场景而定。

- `AxHttpClient`:底层 HTTP API 连接。
- `AxMdWebSocketClient`:市场数据 WebSocket 连接。
- `AxOrdersWebSocketClient`:订单 WebSocket 连接。
- `AxInstrumentProvider`:标的解析与加载功能。
- `AxDataClient`:市场数据流管理器。
- `AxExecutionClient`:账户管理与交易执行网关。
- `AxLiveDataClientFactory`:AX 数据客户端工厂(供交易节点构建器使用)。
- `AxLiveExecClientFactory`:AX 执行客户端工厂(供交易节点构建器使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),
不必直接与这些底层组件打交道。
:::

## AX Exchange 文档

AX Exchange 为用户提供了文档,可在
[Architect 文档站点](https://docs.architect.exchange/)查阅。
建议将本 NautilusTrader 集成指南与 AX Exchange 官方文档结合参考。

## 产品

AX Exchange 专注于传统资产类别上的永续期货合约。永续合约永不到期,
从而消除了标准期货所带来的展期成本。

| 资产类别      | 示例                           | 备注                        |
|------------------|------------------------------------|------------------------------|
| 外汇 | GBPUSD-PERP, EURUSD-PERP           | 主要及次要外汇货币对。    |
| 股票指数    | 股票指数永续合约            |                              |
| 贵金属           | XAU-PERP(黄金)、XAG-PERP(白银) | 贵金属永续合约。  |
| 能源           | 原油、天然气             | 能源商品永续合约。 |
| 利率   | SOFR、国债收益率             | 利率永续合约。             |

### 永续合约

永续合约(perpetual swap)是一种跟踪标的资产价格但没有到期日的衍生品。
与标准期货不同,它没有结算日期,从而消除了展期成本,并简化了仓位管理。
资金费率机制通过多空双方之间的定期结算,使合约价格与标的指数价格保持一致。
有关资金费率机制与合约规格的详情,请参阅
[Architect 文档](https://docs.architect.exchange/)。

AX 永续合约的特点:

- **以美元现金结算**:无实物交割。所有盈亏均以美元结算。
- **资金费率**:定期结算,使合约价格与标的价格保持一致。
- **乘数为 1**:每份合约代表对标的资产的一单位敞口。
- **仅支持整数合约**:不支持小数数量。
- **保证金**:开仓需要初始保证金;维持仓位需要维持保证金。

在 NautilusTrader 中,所有 AX 标的都表示为与资产类别无关的
`PerpetualContract` 永续合约类型。资产类别(外汇、商品、股票等)会
根据标的自动推断。该适配器使用 `MARGIN` 账户类型和 `NETTING` 订单管理模式。

## 符号规则

AX Exchange 使用简单直接的命名约定。所有标的均为永续期货,
在标的资产符号后附加 `-PERP` 后缀标识。

**格式**:`{SYMBOL}-PERP`

| 标的资产     | AX 符号      | Nautilus InstrumentId |
|----------------|----------------|-----------------------|
| GBP/USD        | `GBPUSD-PERP`  | `GBPUSD-PERP.AX`      |
| EUR/USD        | `EURUSD-PERP`  | `EURUSD-PERP.AX`      |
| 黄金           | `XAU-PERP`     | `XAU-PERP.AX`         |
| 白银         | `XAG-PERP`     | `XAG-PERP.AX`         |

场所标识符为 `AX`。构造 Nautilus `InstrumentId`:

```python
from nautilus_trader.model.identifiers import InstrumentId

instrument_id = InstrumentId.from_str("GBPUSD-PERP.AX")
```

## 环境

AX Exchange 提供两种交易环境。请在客户端配置中通过 `environment`
参数配置对应的环境。

| 环境    | 配置                                 | 说明                            |
|----------------|----------------------------------------|----------------------------------------|
| **沙盒(Sandbox)**    | `environment=AxEnvironment.SANDBOX`    | 使用模拟资金的测试环境。 |
| **生产环境(Production)** | `environment=AxEnvironment.PRODUCTION` | 使用真实资金的实盘交易。          |

### 沙盒

用于开发与测试的默认环境,使用模拟资金。
当 `environment=AxEnvironment.SANDBOX` 时,所有沙盒端点会自动解析。

#### 1. 创建沙盒账户

按照 [Architect 文档](https://docs.architect.exchange/)创建沙盒账户。
注册时需要邀请码。

#### 2. 创建 API key 并为账户注资

使用 AX 沙盒界面生成 API key,并向账户中存入模拟资金。
请妥善保管 `api_key` 与 `api_secret`。

#### 3. 设置环境变量

```bash
export AX_API_KEY="your-sandbox-api-key"
export AX_API_SECRET="your-sandbox-api-secret"
```

#### 4. 配置交易节点

```python
config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        AX: AxDataClientConfig(
            environment=AxEnvironment.SANDBOX,
            instrument_provider=InstrumentProviderConfig(load_all=True),
        ),
    },
    exec_clients={
        AX: AxExecClientConfig(
            environment=AxEnvironment.SANDBOX,
            instrument_provider=InstrumentProviderConfig(load_all=True),
        ),
    },
)
```

### 生产环境

用于使用真实资金进行实盘交易。需要一个经过验证的 AX Exchange 账户。

```python
config = AxExecClientConfig(
    environment=AxEnvironment.PRODUCTION,
)
```

:::warning
下单前请务必确认使用的环境是否正确。
默认使用沙盒环境,以防止意外的实盘交易。
:::

## 市场数据

该适配器通过 WebSocket 订阅提供实时市场数据,并通过 HTTP 端点
提供历史数据回填。

### 数据类型

| AX 数据           | Nautilus 数据类型   | 备注                                                              |
|-------------------|----------------------|--------------------------------------------------------------------|
| 订单簿(L1)   | `QuoteTick`          | 来自 L1 订单簿订阅的最优买卖盘(top-of-book)。                |
| 订单簿(L2)   | `OrderBookDelta`     | 聚合的价格档位。                                           |
| 订单簿(L3)   | `OrderBookDelta`     | 单笔订单数量。                                       |
| 成交            | `TradeTick`          | 来自纯成交 WebSocket 订阅的实时成交事件。     |
| 标记价格        | `MarkPriceUpdate`    | 从 L1 ticker 订阅中提取。                             |
| K 线      | `Bar`                | OHLCV 数据(仅总成交量,不区分买卖)。             |
| 资金费率     | `FundingRateUpdate`  | 通过 HTTP 轮询获取;轮询间隔可配置。                            |
| 标的状态 | `InstrumentStatus`   | 来自 L1 ticker 订阅的状态变化(开盘、暂停、收盘)。  |

:::note
AX Exchange 不支持历史报价(quote tick)请求。仅可通过 WebSocket
L1 订单簿订阅获取实时报价数据。
:::

### WebSocket 订阅行为

AX 市场数据 WebSocket 订阅对每个符号仅使用一条活跃流。适配器会选择
能够覆盖当前活跃 Nautilus 订阅的最小流:

- `subscribe_trades` 使用 AX 的 `level: "TRADES"`,仅推送成交打印(trade prints)。
- 仅订阅订单簿或仅订阅报价时,会设置 AX 的 `trades: false` 与 `ticker: false`,
  以抑制未被请求的成交和 ticker 事件。
- 标记价格和标的状态订阅需要 AX 的 ticker 事件,因此当任一数据类型处于活跃
  状态时,适配器会在 L1 流上启用 ticker 推送。
- 如果一个符号有多个 Nautilus 数据类型处于活跃状态,只有当所需的 AX 级别
  或推送标志发生变化时,适配器才会重新订阅。

AX 的发行说明中还提到了 ticker 事件中的预估资金费率,以及订单 WebSocket
的预估资金费率请求。目前 Nautilus 仅通过 HTTP 轮询暴露已结算的资金费率
更新;适配器不会解析或将场所的预估资金费率字段作为单独的 Nautilus
数据类型发出。

### HTTP API 行为

- `GET /tickers` 返回 limit/offset 分页元数据,支持 `limit`、`offset` 和 `sort`
  查询参数。
- `GET /ticker` 在顶层 `ticker` 响应字段下返回 ticker。
- `GET /orders` 返回游标分页元数据,支持 `order_id`、`order_ids`、`account_id`
  以及可选的时间戳过滤条件。
- `GET /transactions` 要求提供 `start_timestamp_ns` 与 `end_timestamp_ns`,
  且时间范围不超过 7 天。
- `GET /order-status` 对于被拒绝的订单可包含 `reject_reason` 与 `reject_message`。

### K 线周期

| 周期 | 说明 |
|----------|-------------|
| `1s`     | 1 秒    |
| `5s`     | 5 秒    |
| `1m`     | 1 分钟    |
| `5m`     | 5 分钟    |
| `15m`    | 15 分钟   |
| `1h`     | 1 小时      |
| `1d`     | 1 天      |

## 订单能力

AX Exchange 仅支持市价单和限价单类型。该场所没有原生的止损/条件单
(下单载荷中没有触发或订单类型字段)。

### 订单类型

| 订单类型             | 是否支持 | 备注                                              |
|------------------------|-----------|----------------------------------------------------|
| `MARKET`               | ✓         | 以最优可得价格立即执行。       |
| `LIMIT`                | ✓         | 以指定价格或更优价格执行。              |
| `STOP_LIMIT`           | -         | *AX Exchange 不支持*。                    |
| `LIMIT_IF_TOUCHED`     | -         | *AX Exchange 不支持*。                    |
| `STOP_MARKET`          | -         | *不支持*。                                   |
| `MARKET_IF_TOUCHED`    | -         | *不支持*。                                   |
| `TRAILING_STOP_MARKET` | -         | *不支持*。                                   |

### 执行指令

| 指令   | 是否支持 | 备注                                               |
|---------------|-----------|-----------------------------------------------------|
| `post_only`   | ✓         | 仅挂单(maker-only);若会吃单则拒绝。 |
| `reduce_only` | -         | *不支持*。                                    |

### 有效期

| 有效期 | 是否支持 | 备注                           |
|---------------|-----------|---------------------------------|
| `GTC`         | ✓         | 撤销前有效(Good Till Canceled)。             |
| `GTD`         | -         | *AX Exchange 不支持*。 |
| `DAY`         | ✓         | 当日交易时段内有效。 |
| `IOC`         | ✓         | 立即成交或取消(Immediate or Cancel)。            |
| `FOK`         | -         | *AX Exchange 不支持*。 |
| `AT_THE_OPEN` | -         | *AX Exchange 不支持*。 |
| `AT_THE_CLOSE`| -         | *AX Exchange 不支持*。 |

该场所已弃用 `DAY`,推荐改用 `GTC`。

### 高级订单功能

| 功能            | 是否支持 | 备注                                                              |
|--------------------|-----------|--------------------------------------------------------------------|
| 订单修改 | ✓         | 通过 `POST /replace-order` 原子替换。返回新的订单 ID。  |
| 取消订单       | ✓         | 单笔订单取消。                                         |
| 全部取消  | ✓         | 取消某标的的所有未成交订单。                          |
| 批量取消       | -         | *AX Exchange 不支持*。改为逐笔取消。   |
| 订单列表        | ✓         | 顺序提交(逐笔提交,非原子操作)。 |

### 仓位管理

| 功能          | 是否支持 | 备注                                |
|------------------|-----------|--------------------------------------|
| 查询仓位  | ✓         | 实时仓位更新。          |
| 仓位模式    | -         | 仅支持净额(netting)模式。                   |
| 全仓保证金     | ✓         | 所有标的的全仓保证金。 |

### 订单查询

| 功能              | 是否支持 | 备注                                                   |
|----------------------|-----------|---------------------------------------------------------|
| 查询未成交订单    | ✓         | 列出所有活跃订单。                                 |
| 查询单笔订单   | ✓         | 通过场所订单 ID 或客户端订单 ID 查询(任意订单状态)。 |
| 订单状态报告 | ✓         | 从未成交订单进行对账;见下方说明。        |
| 成交报告         | ✓         | 执行与成交历史。                             |

:::note
用于对账的订单状态报告是根据未成交订单端点生成的。已成交或已取消的
订单不包含在对账快照中。通过 `query_order` 进行的单笔订单查询使用
专用的 `/order-status` 端点,该端点适用于任意订单状态。
:::

## 身份验证

AX Exchange 使用 bearer token 认证方式:

1. 使用 API key 与 secret 通过 `/authenticate` 获取会话令牌。
2. 该会话令牌作为 bearer token 用于后续的 REST 与 WebSocket 请求。
3. 适配器请求有效期为一小时的会话令牌,并每 30 分钟刷新一次。
4. 刷新会更新 REST 认证信息,以及下一次 WebSocket 重连所使用的令牌,
   而不会中断当前连接。

## 配置

### 环境与端点

| 环境 | HTTP API(市场数据)                           | HTTP API(订单)                                   | 市场数据 WS                                   | 订单 WS                                            |
|-------------|--------------------------------------------------|-----------------------------------------------------|--------------------------------------------------|------------------------------------------------------|
| 沙盒     | `https://gateway.sandbox.architect.exchange/api` | `https://gateway.sandbox.architect.exchange/orders` | `wss://gateway.sandbox.architect.exchange/md/ws` | `wss://gateway.sandbox.architect.exchange/orders/ws` |
| 生产环境  | `https://gateway.architect.exchange/api`         | `https://gateway.architect.exchange/orders`         | `wss://gateway.architect.exchange/md/ws`         | `wss://gateway.architect.exchange/orders/ws`         |

:::info
订单管理相关的 HTTP 端点(下单、取消、订单状态)使用与市场数据端点
不同的基础 URL。适配器配置会自动处理这一点。
:::

### 数据客户端配置选项

| 选项                             | 默认值   | 说明                                                         |
|------------------------------------|-----------|-----------------------------------------------------------------------|
| `api_key`                          | `None`    | API key;省略时从 `AX_API_KEY` 环境变量加载。             |
| `api_secret`                       | `None`    | API secret;省略时从 `AX_API_SECRET` 环境变量加载。       |
| `environment`                      | `SANDBOX` | 交易环境(`SANDBOX` 或 `PRODUCTION`)。                    |
| `base_url_http`                    | `None`    | REST 基础 URL 的覆盖值。                                     |
| `base_url_ws_public`               | `None`    | 市场数据 WebSocket URL 的覆盖值。                         |
| `base_url_ws_private`              | `None`    | 订单 WebSocket URL 的覆盖值。                              |
| `proxy_url`                        | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。               |
| `http_timeout_secs`                | `60`      | REST 请求的超时时间(秒)。                                |
| `max_retries`                      | `3`       | REST 请求的最大重试次数。                           |
| `retry_delay_initial_ms`           | `1000`    | 重试之间的初始延迟(毫秒)。                       |
| `retry_delay_max_ms`               | `10000`   | 重试之间的最大延迟(毫秒,指数退避)。 |
| `heartbeat_interval_secs`          | `20`      | WebSocket 连接的心跳间隔(秒)。             |
| `recv_window_ms`                   | `5000`    | 签名请求的接收窗口(毫秒)。                  |
| `update_instruments_interval_mins` | `60`      | 标的目录刷新间隔(分钟)。            |
| `funding_rate_poll_interval_mins`  | `15`      | 资金费率轮询请求的间隔(分钟)。              |
| `transport_backend`                | `Sockudo` | WebSocket 传输后端。                                        |

### 执行客户端配置选项

| 选项                    | 默认值      | 说明                                                                     |
|---------------------------|--------------|-----------------------------------------------------------------------------------|
| `trader_id`               | `TRADER-001` | 客户端的 Trader ID。                                                       |
| `account_id`              | `AX-001`     | 用于执行事件与对账的账户 ID。                             |
| `api_key`                 | `None`       | API key;省略时从 `AX_API_KEY` 环境变量加载。                         |
| `api_secret`              | `None`       | API secret;省略时从 `AX_API_SECRET` 环境变量加载。                   |
| `environment`             | `SANDBOX`    | 交易环境(`SANDBOX` 或 `PRODUCTION`)。                                |
| `base_url_http`           | `None`       | REST 基础 URL 的覆盖值。                                                 |
| `base_url_orders`         | `None`       | 订单 REST 基础 URL 的覆盖值。                                          |
| `base_url_ws_private`     | `None`       | 订单 WebSocket URL 的覆盖值。                                          |
| `proxy_url`                | `None`       | HTTP 与 WebSocket 传输的可选代理 URL。                           |
| `http_timeout_secs`       | `60`         | REST 请求的超时时间(秒)。                                            |
| `max_retries`             | `3`          | REST 请求的最大重试次数。                                            |
| `retry_delay_initial_ms`  | `1000`       | 重试之间的初始延迟(毫秒)。                                   |
| `retry_delay_max_ms`      | `10000`      | 重试之间的最大延迟(毫秒,指数退避)。             |
| `heartbeat_interval_secs` | `30`         | WebSocket 连接的心跳间隔(秒)。                         |
| `recv_window_ms`          | `5000`       | 签名请求的接收窗口(毫秒)。                              |
| `cancel_on_disconnect`    | `false`      | 当订单 WebSocket 断开连接时,取消所有未成交订单。                   |
| `transport_backend`       | `Sockudo`    | WebSocket 传输后端。                                                    |

当启用 `transport-sockudo` Cargo 特性时,`transport_backend` 默认值为
`Sockudo`;当该特性被禁用时,会回退为 `Tungstenite`。

最常见的用法是配置一个实时 `TradingNode`,使其包含 AX Exchange
数据客户端和执行客户端。为此,请在客户端配置中添加 `AX` 部分:

```python
from nautilus_trader.adapters.architect_ax import AX
from nautilus_trader.adapters.architect_ax import AxDataClientConfig
from nautilus_trader.adapters.architect_ax import AxEnvironment
from nautilus_trader.adapters.architect_ax import AxExecClientConfig
from nautilus_trader.config import InstrumentProviderConfig
from nautilus_trader.config import TradingNodeConfig

config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        AX: AxDataClientConfig(
            environment=AxEnvironment.SANDBOX,
            instrument_provider=InstrumentProviderConfig(load_all=True),
        ),
    },
    exec_clients={
        AX: AxExecClientConfig(
            environment=AxEnvironment.SANDBOX,
            instrument_provider=InstrumentProviderConfig(load_all=True),
        ),
    },
)
```

然后,创建一个 `TradingNode` 并添加客户端工厂:

```python
from nautilus_trader.adapters.architect_ax import AX
from nautilus_trader.adapters.architect_ax import AxLiveDataClientFactory
from nautilus_trader.adapters.architect_ax import AxLiveExecClientFactory
from nautilus_trader.live.node import TradingNode

# 使用配置实例化实时交易节点
node = TradingNode(config=config)

# 向节点注册客户端工厂
node.add_data_client_factory(AX, AxLiveDataClientFactory)
node.add_exec_client_factory(AX, AxLiveExecClientFactory)

# 最后构建节点
node.build()
```

### API 凭证

向 AX Exchange 客户端提供凭证有两种方式。可以将对应的
`api_key` 与 `api_secret` 值传给配置对象,或设置以下环境变量:

- `AX_API_KEY`
- `AX_API_SECRET`

:::tip
建议使用环境变量来管理凭证。
:::

启动交易节点时,你会立即收到关于凭证是否有效以及是否具有交易权限的确认。

## 实现说明

- **仅支持整数合约**:AX Exchange 使用整数合约数量。不支持小数数量;
  适配器会在本地生成 `OrderDenied`。
- **速率限制**:适配器采用较为保守的每秒 10 次请求的速率限制,
  在收到速率限制响应时会自动进行指数退避。
- **市价单**:AX 不支持原生市价单。适配器使用预览端点确定穿价价格
  (take-through price),并提交一个激进的 IOC 限价单。
- **订单修改**:AX 通过 `POST /replace-order` 支持原子性的订单替换。
  适配器将 `modify_order` 映射到该端点。交易所会取消原订单并创建一个
  带有更新字段的新订单,返回新的订单 ID。
- **断线取消**:在执行客户端配置中设置 `cancel_on_disconnect=True`,
  可在订单 WebSocket 断开连接时让交易所取消所有未成交订单。
- **成交手续费**:来自 WebSocket 的实时成交事件不包含手续费数据。
  对于流式成交,手续费报告为零。在对账期间,REST 的 `/fills` 端点
  会提供准确的手续费信息。
- **成交对账窗口**:`/fills` 端点要求提供有界的时间范围,且最大跨度
  上限为 7 天。对账会请求最近七天的成交记录;超出该范围的成交不会
  被对账。
- **未成交的 IOC/FOK**:AX 会将未成交的即时订单报告为过期(expiry);
  适配器将其映射为 `OrderCanceled`,以匹配 NautilusTrader 的语义。

## 贡献

:::info
如需了解更多功能或为 AX Exchange 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
