# Coinbase

Coinbase 成立于 2012 年,是美国最大的受监管加密货币交易所之一,通过
Advanced Trade API 提供现货、永续互换和到期期货交易。该适配器通过一个
共享的执行客户端,支持在现货(Cash)和 CFM 衍生品(Margin)账户上进行
实时市场数据接入与订单执行,账户类型由工厂选择(参见
[执行范围](#执行范围))。

:::note
该适配器仅限 Rust 实现,供 v2 系统(以及 Rust `LiveNode`)使用。它不提供
传统的 Python `TradingNode` 集成;只有配置和枚举类型通过 PyO3 导出,以便
v2 的 Python 入口点能够构造它们。
:::

## 概览

Coinbase 适配器以 Rust 实现,供 v2 系统使用。该适配器不提供传统的 Python
`TradingNode` 集成;只有配置和枚举类型通过 PyO3 导出,以便 v2 入口点能够
从 Python 构造它们。

组件:

- `CoinbaseHttpClient`:双层 REST 客户端(原始端点方法 + 领域封装)。
- `CoinbaseWebSocketClient`:带 JWT 订阅认证的底层 WebSocket 连接。
- `CoinbaseInstrumentProvider`:标的解析与加载。
- `CoinbaseDataClient`:市场数据流管理器。
- `CoinbaseDataClientFactory`:数据客户端工厂。
- `CoinbaseExecutionClient`:执行客户端(现货或 CFM 衍生品;REST 下单
  + WS 数据流)。
- `CoinbaseExecutionClientFactory`:执行客户端工厂;现货与 CFM 衍生品
  通过配置上的 `account_type` 选择。

从 `nautilus_trader.core.nautilus_pyo3.coinbase` 可用的 PyO3 接口:

- `CoinbaseDataClientConfig`、`CoinbaseExecClientConfig`
- `CoinbaseEnvironment`、`CoinbaseMarginType`
- `COINBASE` 场所常量

## Coinbase 文档

Coinbase 为 Advanced Trade API 提供了以下文档:

- [REST API 参考](https://docs.cdp.coinbase.com/advanced-trade/reference)
- [WebSocket 频道](https://docs.cdp.coinbase.com/advanced-trade/docs/ws-channels)
- [API key 身份验证](https://docs.cdp.coinbase.com/coinbase-app/authentication-authorization/api-key-authentication)
- [速率限制](https://docs.cdp.coinbase.com/advanced-trade/docs/rate-limits)

建议将本 NautilusTrader 集成指南与 Coinbase 官方文档结合参考。

:::info
该适配器针对的是 Coinbase Advanced Trade API。独立的
[Coinbase International Exchange(INTX)](https://international.coinbase.com)
场所由专门的 `coinbase_intx` 适配器支持。
:::

## 产品

产品(product)是一组相关标的类型的统称。

支持以下产品类型:

| 产品类型        | 是否支持 | 备注                                                    |
|---------------------|-----------|------------------------------------------------------------|
| 现货                | ✓         | 以 USD、USDC 和 USDT 计价的现货交易对。                   |
| 永续合约 | ✓         | FCM 场所上以 USD 保证金的永续互换。           |
| 期货合约   | ✓         | 到期交割期货(nano BTC、nano ETH 等)。        |

## 符号规则

Coinbase 直接使用场所原生的 `product_id` 字段作为 Nautilus 符号。标的 ID
为 `{product_id}.COINBASE`。

| 产品          | 格式                             | 示例                           |
|------------------|------------------------------------|------------------------------------|
| 现货             | `{base}-{quote}`                   | `BTC-USD`、`ETH-USDC`、`SOL-USDT`。 |
| 永续        | `{contract_code}-{ddMMMyy}-CDE`    | `BIP-20DEC30-CDE`(BTC 永续)。      |
| 到期期货     | `{contract_code}-{ddMMMyy}-CDE`    | `BIT-24APR26-CDE`(BTC 2026 年 4 月到期)。  |

`-CDE` 后缀表示 Coinbase Derivatives Exchange(FCM 场所)。永续合约带有
交易所分配的远期到期日(例如 `20DEC30`),但根据是否存在持续的资金费率,
将其归类为 `CryptoPerpetual`。到期期货则归类为 `CryptoFuture`。

适配器从 API 元数据中按结构判定产品类型(`future_product_details.contract_expiry_type`;
当其为 `EXPIRING` 时,再依据 `future_product_details.funding_rate` 是否为
非空作为仅限永续合约的结构化信号);备用启发式方法会检查 `display_name`
中是否包含 `PERP` 或 `Perpetual` 子字符串。

完整 Nautilus 标的 ID 示例:

- `BTC-USD.COINBASE`(现货比特币/美元)。
- `ETH-USDC.COINBASE`(现货以太坊/USDC)。
- `BIP-20DEC30-CDE.COINBASE`(BTC 永续互换)。
- `BIT-24APR26-CDE.COINBASE`(BTC 到期期货,2026 年 4 月)。

### 别名产品(USDC 与 USD)

Coinbase 将同一交易对的 USDC 计价版本和 USD 计价版本合并到单一的撮合引擎
订单簿中,并通过 `GET /products` 的 `alias` 和 `alias_to` 字段暴露这种
关系:

```text
BTC-USD :  alias=""        alias_to=["BTC-USDC"]   # 规范形式
BTC-USDC:  alias="BTC-USD" alias_to=[]             # BTC-USD 的别名
```

当调用方使用别名一侧进行订阅或下单时,场所会在传输层将请求改写为规范
ID。适配器透明地处理这一点:它在启动时记录 `product_id -> alias` 映射,
在订阅和下单时发送规范 ID,在 WebSocket 客户端上注册反向映射,并在解析
之前将入站消息重新映射回调用方提供的 ID。

因此,一个只持有 USDC 的策略可以端到端地交易 `BTC-USDC.COINBASE`,
而无需引用规范的 `BTC-USD`。结算货币由提交的 `product_id` 决定,因此
在 `BTC-USDC.COINBASE` 上下的订单始终会借记或贷记 USDC 钱包。

## 环境

Coinbase 提供两种交易环境。请在客户端配置中通过 `environment` 字段
配置对应的环境。

| 环境 | `environment` 取值             | REST 基础 URL                      |
|-------------|---------------------------------|-------------------------------------|
| Live        | `CoinbaseEnvironment.LIVE`      | `https://api.coinbase.com`         |
| Sandbox     | `CoinbaseEnvironment.SANDBOX`   | `https://api-sandbox.coinbase.com` |

### Live(生产环境)

用于真实资金实盘交易的默认环境。

```python
config = CoinbaseExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    # environment=CoinbaseEnvironment.LIVE (默认)
)
```

环境变量:`COINBASE_API_KEY`、`COINBASE_API_SECRET`。

### Sandbox

根据 [Sandbox 文档](https://docs.cdp.coinbase.com/coinbase-app/advanced-trade-apis/sandbox),
这是一个用于集成调试的静态模拟测试环境。

```python
config = CoinbaseExecClientConfig(
    api_key="ANY_NON_EMPTY_STRING",   # 适配器构造函数要求
    api_secret="ANY_NON_EMPTY_STRING",
    environment=CoinbaseEnvironment.SANDBOX,
)
```

沙盒场所不强制要求身份验证,但 `CoinbaseExecutionClient::new` 仍要求
构造时提供这两个字段(或对应的环境变量)。

:::warning
**沙盒不是一个并行的交易场所:**

- 所有响应都是静态且预先定义的;没有实时市场或动态定价。
- 只有 Accounts 和 Orders 端点可用;其他资源不可用。
- 无需身份验证(也不会强制要求)。
- 自定义的 `X-Sandbox` 请求头可以触发预定义的错误场景。

请使用沙盒来验证客户端接线以及请求/响应格式;若要测试真实行为,请使用
生产环境(需谨慎处理真实资金)。
:::

## 身份验证

Coinbase Advanced Trade 使用 ES256 JWT 身份验证。每个 REST 请求和每次
WebSocket 订阅都会生成一个使用你的 EC 私钥签名的短期 JWT。适配器从环境
变量或配置字段解析凭证。

### 创建 API key

Coinbase 有多种 key 类型。该适配器要求使用签名算法为 **ECDSA**(而非
Ed25519)的 **Coinbase App Secret API key**。

<Steps>
<Step>
打开 CDP 门户的 API keys 页面:
[portal.cdp.coinbase.com/projects/api-keys](https://portal.cdp.coinbase.com/projects/api-keys)。
</Step>
<Step>
选择 **Secret API Keys** 标签页,点击 **Create API key**。
</Step>
<Step>
输入一个昵称(例如 `nautilus-trading`)。
</Step>
<Step>
展开 **API restrictions**,将权限设置为 **View** 和 **Trade**。
</Step>
<Step>
展开 **Advanced Settings**,将签名算法从 Ed25519 改为 **ECDSA**。此步骤
是必需的:Ed25519 密钥无法用于 Advanced Trade API。
</Step>
<Step>
点击 **Create API key**。从弹出的对话框中保存密钥名称和私钥。密钥名称
形如 `organizations/{org_id}/apiKeys/{key_id}`。私钥是 PEM 编码的 EC
密钥(SEC1 格式)。
</Step>
</Steps>

:::warning
Coinbase 不再自动下载密钥文件。请在关闭创建对话框之前,复制其中的值或
点击下载按钮。之后你将无法再次获取该私钥。
:::

:::info
不要使用来自 coinbase.com/settings/api 的传统 API key(UUID 格式,
HMAC-SHA256 签名)。那类密钥使用不同的认证方案(`CB-ACCESS-*` 请求头),
该适配器不支持。
:::

完整详情请参见 Coinbase 的
[API key 身份验证指南](https://docs.cdp.coinbase.com/coinbase-app/authentication-authorization/api-key-authentication)。

### 环境变量

| 变量              | 说明                                               |
|-----------------------|-----------------------------------------------------------|
| `COINBASE_API_KEY`    | 密钥名称(`organizations/{org_id}/apiKeys/{key_id}`)。     |
| `COINBASE_API_SECRET` | PEM 编码的 EC 私钥(完整的多行字符串)。      |

示例:

```bash
export COINBASE_API_KEY="organizations/abc-123/apiKeys/def-456"
export COINBASE_API_SECRET="$(cat ~/path/to/cdp_api_key.pem)"
```

:::tip
建议使用环境变量来管理凭证。
:::

### JWT 有效期

Coinbase JWT 在 120 秒后过期。根据
[WebSocket 概览](https://docs.cdp.coinbase.com/coinbase-app/advanced-trade-apis/websocket/websocket-overview),
每条经过身份验证的 WebSocket 消息(即每次订阅)都必须生成一个新的 JWT。
适配器会为每个已签名的 REST 请求以及每条已认证的订阅消息重新生成新的
JWT;无需手动轮换。

## 组合(Portfolios)

一个 Coinbase 账户持有一个或多个**组合(portfolios)**。每个组合拥有
自己的钱包(USD、USDC、BTC 等)、余额和订单范围。每个账户都有一个
`DEFAULT` 组合;用户可以创建额外的 `CONSUMER` 组合,以隔离策略、风险
或税务批次。

CDP API key **在创建时会绑定到单个组合**。除非明确指定其他组合,
否则每个已认证的请求(账户查询、下单、取消)都作用于该组合。

### 查找你的组合 UUID

运行适配器的已认证探测二进制程序;它会打印你的 CDP key 可见的组合、
绑定组合中的账户余额,以及一些参考性的 REST 调用:

```bash
cargo run --bin coinbase-http-private --package nautilus-coinbase
```

示例输出:

```
Found 1 portfolio(s)
  name=Default type=DEFAULT uuid=ca7244bc-21d1-5e4c-bfe5-80f208ac5723 deleted=false
Account has 3 balance(s)
  USDC total=100.00000000 USDC free=100.00000000 USDC locked=0.00000000 USDC
  AUD total=0.00 AUD free=0.00 AUD locked=0.00 AUD
  BTC total=0.00000000 BTC free=0.00000000 BTC locked=0.00000000 BTC
```

等效的 curl 调用(你需要先用自己的 CDP PEM 密钥签署自己的 ES256 JWT):

```bash
curl -H "Authorization: Bearer $JWT" \
  https://api.coinbase.com/api/v3/brokerage/portfolios
```

### 何时需要 `retail_portfolio_id`

Coinbase 的 `POST /orders` 端点默认路由到该 key 绑定的组合,因此单组合
账户不需要设置此字段。当以下任一条件成立时,请在
[`CoinbaseExecClientConfig`](#执行客户端配置选项)上设置该字段:

- 该账户持有多个组合,而你想针对非该 key 默认组合的组合进行交易。
- 场所以 `account is not available` 拒绝订单,且已排除下方的钱包
  诊断情形。

### 创建新组合

大多数用户不需要创建新组合;账户的默认组合开箱即用。仅当你想要以下
情形时,才在 [coinbase.com/portfolios](https://www.coinbase.com/portfolios)
上创建组合:

- 将 API 驱动的交易与手动零售活动分隔开。
- 在不同策略之间隔离风险或盈亏。
- 绕过受限的默认组合(例如 Vault)。

创建组合后,请先为其注资(从 coinbase.com 上默认组合的钱包转账),
然后再发送任何订单,否则场所会针对计价货币返回
`account is not available`。

### 排查 `account is not available`

场所会因多种不同原因返回此错误;可通过运行上述探测程序并检查组合钱包
列表来进行诊断。

| 症状                                                              | 可能原因                                                                                          | 解决方法                                                                                       |
|----------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 仅特定产品被拒绝(例如,只持有 USDC 时下 `BTC-USD`) | 该组合缺少该产品计价货币对应的钱包。USD 和 USDC 在 Coinbase 上是分开的,场所按提交的 `product_id` 路由订单,而非按规范别名路由。 | 针对你持有货币所对应的产品提交订单(例如,持有 USDC 时使用 `BTC-USDC`)。适配器会在内部解析数据侧的别名;无需修改配置。也可以通过 coinbase.com 为缺失的钱包注资,但如果你只持有一种货币,这并非必要。 |
| 所有产品的每笔订单都被拒绝                             | key 绑定到一个非默认组合,且未设置 `retail_portfolio_id`。                           | 在 `CoinbaseExecClientConfig` 上将 `retail_portfolio_id` 设置为目标组合 UUID。     |
| 非美国账户下 `*-USD` 产品被拒绝                     | 司法辖区限制(例如,澳大利亚账户不能交易以 USD 计价的交易对)。                                                                          | 改用本地可用的计价货币(USDC、AUD、EUR 等)而非 USD。                                                                       |
| key 轮换后立即被拒绝                                    | 新 key 是在与之前不同的组合中创建的。                                                                   | 更新 `retail_portfolio_id` 以匹配新 key 所在的组合,或转移资金。             |

## 订单能力

下表描述的是 Coinbase **场所**层面的订单能力范围。所提供的
[`CoinbaseExecutionClient`](#执行范围) 根据配置的 `account_type` 处理
现货或 CFM 衍生品。Coinbase 的订单能力在现货和衍生品之间有所不同
(永续合约与到期期货共享相同的 FCM 订单能力)。

### 执行范围

`CoinbaseExecutionClientFactory` 生成单一的 `CoinbaseExecutionClient`
类型。产品系列由 `CoinbaseExecClientConfig` 上的 `account_type` 字段
选择:

| `account_type`        | 启动标的加载                         | 账户状态来源                                      |
|-----------------------|-----------------------------------------------|-----------------------------------------------------------|
| `AccountType::Cash`   | 仅 `CoinbaseProductType::Spot`。             | `/accounts` REST 端点。                                |
| `AccountType::Margin` | `CoinbaseProductType::Future`(永续 + 到期)。 | CFM `balance_summary` REST + `futures_balance_summary` WS,加上来自 `cfm/positions` 的仓位报告。 |

其他账户类型在工厂创建时会被拒绝。由于该场所未暴露对冲模式,OMS 始终
为 `Netting`。

为防止跨账户串扰:

1. 连接时的标的启动加载被限定在配置的产品系列内;另一个系列的产品
   永远不会进入进程内缓存。
2. `submit_order` 会拒绝任何标的不在该缓存中的订单。
3. `generate_order_status_report(s)` 和 `generate_fill_reports` 会通过
   同一缓存对其输出进行后置过滤,因此一个同时拥有现货和衍生品活动的
   Coinbase 账户,不会通过单一客户端暴露另一范围的报告。

请为每个范围运行一个执行客户端;如果同一个 trader 需要同时进行现货和
CFM 活动,请实例化两个具有不同 `account_type` 值(以及不同
`account_id`)的客户端。

### 订单类型

该矩阵列出了通过 Nautilus 模型暴露的订单类型。右列显示适配器发出的
对应 `order_configuration` 键。本表未列出的 Coinbase 订单类型(TWAP、
Bracket、Scaled、SOR LIMIT IOC)记录在
[高级订单功能](#高级订单功能)中,并标注为适配器*尚不支持*。

| 订单类型             | 现货 | 永续 | 期货 | 传输层格式                                                  |
|------------------------|------|-----------|--------|-------------------------------------------------------------|
| `MARKET`               | ✓    | ✓         | ✓      | `market_market_ioc`(现货 + CFM);`market_market_fok`(仅 CFM) |
| `LIMIT`                | ✓    | ✓         | ✓      | `limit_limit_gtc` / `limit_limit_gtd` / `limit_limit_fok`   |
| `STOP_LIMIT`           | -    | ✓         | ✓      | `stop_limit_stop_limit_gtc` / `stop_limit_stop_limit_gtd`   |
| `STOP_MARKET`          | -    | -         | -      | *场所未暴露。*                                 |
| `MARKET_IF_TOUCHED`    | -    | -         | -      | *场所未暴露。*                                 |
| `LIMIT_IF_TOUCHED`     | -    | -         | -      | *场所未暴露。*                                 |
| `TRAILING_STOP_MARKET` | -    | -         | -      | *场所未暴露。*                                 |

### 执行指令

| 指令   | 现货 | 永续 | 期货 | 备注                                                              |
|---------------|------|-----------|--------|--------------------------------------------------------------------|
| `post_only`   | ✓    | ✓         | ✓      | 仅限 LIMIT GTC 和 LIMIT GTD。                                      |
| `reduce_only` | -    | ✓         | ✓      | 仅限衍生品。                                                  |

### 有效期

适配器接受该矩阵中列出的组合;未列出的组合会在提交时以
`"Unsupported TIF {tif} for {order_type}"` 拒绝。

| 订单类型   | GTC | GTD | IOC | FOK | 备注                                                          |
|--------------|-----|-----|-----|-----|----------------------------------------------------------------|
| `MARKET`     | ✓   | -   | ✓   | (✓) | GTC 被映射为 IOC;显式的 IOC 会被遵守。FOK 会构建场所的 `market_market_fok` 格式,但撮合引擎目前在现货上会以 `UNSUPPORTED_ORDER_CONFIGURATION` 拒绝它;仅可用于 CFM 衍生品。 |
| `LIMIT`      | ✓   | ✓   | -   | ✓   | GTD 需要 `expire_time`。LIMIT IOC *尚不支持*(参见 [SOR LIMIT IOC](#高级订单功能))。 |
| `STOP_LIMIT` | ✓   | ✓   | -   | -   | 需要 `trigger_price`。仅限衍生品。                    |

### 高级订单功能

| 功能            | 现货 | 永续 | 期货 | 备注                                                                              |
|--------------------|------|-----------|--------|------------------------------------------------------------------------------------|
| 订单修改 | ✓    | ✓         | ✓      | 仅限 GTC 类型(LIMIT、STOP_LIMIT、Bracket);其他类型使用取消重下。    |
| 括号单     | -    | -         | -      | *尚不支持。* 场所暴露了 `trigger_bracket_gtc` / `trigger_bracket_gtd`。  |
| OCO 订单         | -    | -         | -      | *场所未将其作为独立订单类型暴露。*                               |
| 冰山订单     | -    | -         | -      | *场所未暴露。*                                                        |
| TWAP 订单        | -    | -         | -      | *尚不支持。* 场所暴露了 `twap_limit_gtd`。                               |
| Scaled 订单      | -    | -         | -      | *尚不支持。* 场所暴露了 `scaled_limit_gtc`。                             |
| SOR LIMIT IOC      | -    | -         | -      | *尚不支持。* 场所为智能订单路由 LIMIT IOC 暴露了 `sor_limit_ioc`。 |

底层场所规范请参见
[创建订单参考](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/orders/create-order)
和 [编辑订单参考](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/orders/edit-order)。

### 仓位控制(衍生品)

| 控制项       | 备注                                                                |
|---------------|----------------------------------------------------------------------|
| 杠杆      | 按订单设置;默认 `1.0`。                                        |
| 保证金类型   | 按订单设置:全仓(默认)或逐仓。                          |
| 仓位模式 | 仅单向;未暴露对冲模式。                             |

### 批量操作

| 操作     | 备注                                                                                              |
|---------------|----------------------------------------------------------------------------------------------------|
| 批量提交  | 不支持。每笔订单对应一次 `Create Order` 请求。                                           |
| 批量修改  | 不支持。每次修改对应一次 `Edit Order` 请求。                                              |
| 批量取消  | `POST /api/v3/brokerage/orders/batch_cancel` 接受一个 `order_ids` 数组。未记录最大数量;响应中包含每笔订单的成功/失败状态。 |

### 订单查询

| 功能              | 现货 | 永续 | 期货 | 备注                                       |
|----------------------|------|-----------|--------|---------------------------------------------|
| 查询未成交订单    | ✓    | ✓         | ✓      | 列出所有活跃订单。                     |
| 查询订单历史  | ✓    | ✓         | ✓      | 带游标分页的历史订单数据。   |
| 订单状态更新 | ✓    | ✓         | ✓      | 通过 `user` 频道的实时状态变化。 |
| 成交历史        | ✓    | ✓         | ✓      | 执行与成交报告。                 |

### 现货交易限制

- 现货订单不支持 `reduce_only`(该指令适用于衍生品)。
- 不支持跟踪止损单。
- 现货不提供原生的止损限价单和括号单。
- 支持以计价货币计量的 MARKET 订单;LIMIT 订单以基础货币单位计量。

### 衍生品交易

Coinbase 衍生品通过 FCM(期货佣金商)场所交易。执行客户端通过与现货
相同的 `POST /orders` 端点提交订单;每笔订单的 `leverage` 和
`margin_type`(`CROSS` 或 `ISOLATED`)默认值来自
`CoinbaseExecClientConfig.default_leverage` 和 `default_margin_type`。
保证金余额通过 REST 的 `cfm/balance_summary` 端点(连接时快照、
`query_account`,以及 WebSocket 重连时)以及已认证的
`futures_balance_summary` WebSocket 频道更新。仓位报告来自 REST 的
`cfm/positions` 端点。

Coinbase 的 Advanced Trade API 在创建订单的 schema 中并未记录
`reduce_only` 字段,尽管场所的失败原因枚举承认这一概念。客户端在其
`submit_order` 签名中透传 `reduce_only` 以保持 API 一致性,仅当其为
`true` 时才在传输层包含该标志;若场所日后接受它,客户端无需任何改动。

#### 资金费率

适配器按 `derivatives_poll_interval_secs`(默认 15 秒)轮询 REST 的
`/products/{id}` 端点,当 FCM 的 `future_product_details` 载荷中存在
`funding_rate` 时,发出一条 `FundingRateUpdate`。资金费率周期从
`funding_interval` 字段解析(通常为 `"3600s"`,即每小时结算一次),
下一次结算时间戳来自 `funding_time`。Coinbase Advanced Trade 不会在
WebSocket 的 `ticker` 频道上发布 `funding_rate`,因此 REST 轮询是唯一
的实时来源。

历史资金费率请求(`DataTester` TC-D53)尚未实现;未来版本中可以由
同一个 REST products 端点提供,通过连续的资金费率时间戳推导出间隔。

#### 标的状态

`subscribe_instrument_status` 在首次订阅时加入 Coinbase 的 WebSocket
`status` 频道(该场所为所有产品发布一个统一的状态数据流),将传入的
事件过滤到已订阅的标的,并发出 `InstrumentStatus` 事件:`online` 对应
`MarketStatusAction::Trading`,`offline` 对应 `Halt`,`delisted` 对应
`Close`。报告空 `status` 字符串的期货产品对数据引擎没有信息价值,会被
跳过。当最后一个标的取消订阅时,该频道订阅会被取消。

#### 仓位对账

对于 Cash(现货)账户,客户端不返回任何仓位报告,因为 Coinbase 现货
没有仓位。对于 Margin 账户,仓位报告来自 REST 的 `cfm/positions`(列表)
和 `cfm/positions/{product_id}`(单个)端点,并会被后置过滤到启动时
的标的缓存。未成交订单和历史成交在连接时以及由 `LiveExecEngineConfig`
设置的标准对账间隔,通过 `generate_order_status_report(s)` 和
`generate_fill_reports` 从 REST 对账。

#### 成交去重

用户频道 WebSocket 在重连时可能会重放事件。执行客户端维护一个容量为
10,000 条的 FIFO 去重队列,以 `(venue_order_id, trade_id)` 为键,
丢弃任何合成交易 ID 与近期已见记录匹配的成交。累积状态映射的容量
限制与之相同,以防范在此客户端生命周期内从未收到终态事件的订单。
在非常长时间的断连之后(超出内存去重窗口),重放的成交可能会发出
重复的 `FillReport`;在这种情况下,策略应依赖 REST 对账来恢复规范状态。

## 执行客户端行为

本节记录 `CoinbaseExecutionClient` 如何将 Nautilus 订单命令和 Coinbase
场所事件转换为 Nautilus 执行事件。

:::warning
Coinbase 实盘执行尚未被批准作为 v2 切换的证据。用户频道报告的是订单
累积状态,不包含 Coinbase 的每笔成交 `trade_id`,因此其合成的实时成交
ID 与 REST 对账返回的场所 ID 不匹配。在两条路径使用相同的稳定成交身份
之前,适配器无法证明在启动对账窗口内成交的幂等性。
:::

### 订单提交

`submit_order` 直接根据 Nautilus 订单字段构建 Coinbase 的
`order_configuration` 结构:

- `MARKET` -> `market_market_ioc`。仅接受 `TimeInForce::Ioc` 和 `Gtc`
  (Nautilus 默认值);市价单上显式的 `Fok`、`Day` 或 `Gtd` 会在 HTTP
  调用之前被拒绝,以免调用方在不知情的情况下获得 IOC 语义。以 `Gtc`
  构建的 `MARKET` 订单在场所以 IOC 方式执行;需要严格回测/实盘一致性的
  策略应显式使用 `Ioc` 构造 `MarketOrder`。
- `LIMIT` GTC -> `limit_limit_gtc`,GTD -> `limit_limit_gtd`(需要
  `expire_time`),FOK -> `limit_limit_fok`。
- `STOP_LIMIT` GTC -> `stop_limit_stop_limit_gtc`,GTD ->
  `stop_limit_stop_limit_gtd`。止损方向根据订单方向推导(`Buy` ->
  `STOP_DIRECTION_STOP_UP`,`Sell` -> `STOP_DIRECTION_STOP_DOWN`)。
- `STOP_MARKET`、`MARKET_IF_TOUCHED`、`LIMIT_IF_TOUCHED` 以及跟踪止损
  变体均未被场所暴露。它们会以 `OrderRejected` 呈现,携带来自已派生的
  提交任务中 `build_order_configuration` 的错误信息(会先发出
  `OrderSubmitted`)。

HTTP 创建成功后,会发出携带 `success_response.order_id` 中返回的场所
订单 ID 的 `OrderAccepted`。若响应中 `success=false`,则会发出携带
格式化后场所失败原因的 `OrderRejected`。场所结果未知的 HTTP 失败会使
订单保持在飞行中状态,等待 WebSocket 更新、未成交订单轮询或对账处理。

### 订单修改

`modify_order` 使用带类型的 `EditOrderRequest` 向 `/orders/edit` 发起
POST 请求。Coinbase 将修改限制在 GTC 类型(LIMIT、STOP_LIMIT、
Bracket);其他订单类型必须使用取消重下。

Coinbase 的 `/orders/edit` 即使只修改其中一项,也要求同时提供 `price`
和 `size`;省略的 `size` 会被读作 0,并以 `INVALID_EDITED_SIZE` 或
`CANNOT_EDIT_TO_BELOW_FILLED_SIZE` 拒绝。执行客户端会从缓存的订单中
自动填充缺失字段,因此策略调用 `modify_order(price=X)` 时无需重复
提供当前数量。来自 `ModifyOrder` 命令的值优先;否则使用缓存订单当前的
`price` 和 `quantity`。

场所修改失败时,会以带类型的 `EditOrderResponse` 原因(优先使用
`edit_failure_reason`,回退到 `preview_failure_reason`)发出
`OrderModifyRejected`。场所结果未知的 HTTP 失败会使订单保持在
`PENDING_UPDATE`,直到某次更新、查询结果或对账将其解决。

### 取消

- `cancel_order` 发起一次单 ID 的 `batch_cancel`。明确的按订单场所失败
  会呈现为 `OrderCancelRejected`;场所结果未知的整体请求传输失败会使
  订单保持在 `PENDING_CANCEL`,等待对账处理。
- `cancel_all_orders` 通过 REST 列出未成交订单,不使用仅 `OPEN` 的过滤
  (因为 Coinbase 的 `OPEN` 过滤器会排除仍可取消的 `PENDING` 和
  `QUEUED` 订单),在本地过滤到
  `{Submitted, Accepted, Triggered, PendingUpdate, PartiallyFilled}`
  以及请求的方向,然后以每组 100 个的方式分批调用 `batch_cancel`。
  按订单的场所失败会发出 `OrderCancelRejected`;场所结果未知的整体
  请求失败会使受影响的订单等待对账处理。
- `batch_cancel_orders` 以相同方式分批,并将明确的按订单场所失败呈现为
  `OrderCancelRejected`。场所结果未知的传输失败会使受影响的订单等待
  对账处理。

### 用户 WebSocket 频道

`CoinbaseExecutionClient` 订阅 `user` 频道时不设置 `product_ids` 过滤,
并使用一个新生成的 JWT,将每个事件解析为 `OrderStatusReport`,并将其
送入执行事件流。Coinbase 报告的是每个订单的累积状态,而非按笔成交,
因此执行客户端会从累积增量中合成 `FillReport`。单笔成交价格通过
`(avg_now * qty_now - avg_prev * qty_prev) / delta_qty` 推导,因此
多笔成交的订单能够携带正确的成交价格,而非累积加权平均价格。在终态
更新(`CANCELLED`、`EXPIRED`、`FAILED`,此时场所将 `leaves_quantity`
清零)时,会恢复原始数量。

用户频道不会回显 `price`、`stop_price`、`trigger_type` 或
maker/taker 分类。执行客户端会在提交时按 `client_order_id` 缓存这些
信息,并在发出之前修补报告,因此对账器不会观察到
`Some(price) -> None` 的差异,`post_only` 成交也能正确标注为
`liquidity_side = Maker`。订单状态 `PENDING`、`QUEUED` 和 `OPEN` 都会
映射为 `OrderStatus::Accepted`,以避免当用户频道更新与 REST 的
`OrderAccepted` 事件产生竞态时出现虚假的状态倒退警告。

携带 `INVALID_LIMIT_PRICE_POST_ONLY`(或预览/新订单等价错误)的
`submit_order` 拒绝,会以 `due_post_only = true` 发出,以便策略能够
针对 post-only 吃单情况做出反应(通常是根据新的 TOB 重新报价)。

重连时,账户状态会通过 REST 重新获取,以恢复断连窗口期间的余额变化。
按订单的累积跟踪会在重连后持续保留,因此合成的成交增量仍然正确。

## 速率限制

Coinbase 为 Advanced Trade API 发布了以下限制:

| 范围                           | 限制                                                | 来源                                                |
|-----------------------------------|------------------------------------------------------|---------------------------------------------------------|
| WebSocket 连接             | 每个 IP 地址每秒 8 次                          | Advanced Trade WebSocket 速率限制                  |
| WebSocket 未认证消息    | 每个 IP 地址每秒 8 次                          | Advanced Trade WebSocket 速率限制                  |
| WebSocket 订阅截止时间      | 首条订阅消息必须在连接后 5 秒内到达,否则服务器会断开连接 | Advanced Trade WebSocket 概览 |
| 已认证 WebSocket JWT       | 120 秒;每条已认证的订阅消息都必须生成新的 JWT | Advanced Trade WebSocket 概览 |
| REST 每个 key 的配额                | 每个 API key 每小时 10,000 次请求(Coinbase App 通用策略) | Coinbase App 速率限制       |

超出 REST 限制时,Coinbase 会返回 HTTP `429`,并带有如下响应体:

```json
{
  "errors": [
    {
      "id": "rate_limit_exceeded",
      "message": "Too many requests"
    }
  ]
}
```

:::info
截至撰写本文时,Advanced Trade 专属的 REST 配额(每秒上限、按组合限制)
未在 Advanced Trade 文档中单独发布;上方的 Coinbase App 每小时配额是
目前记录最明确的数值。参考资料:
[REST 速率限制](https://docs.cdp.coinbase.com/advanced-trade/docs/rest-api-rate-limits/)、
[WebSocket 速率限制](https://docs.cdp.coinbase.com/advanced-trade/docs/ws-rate-limits)、
[Coinbase App 速率限制](https://docs.cdp.coinbase.com/coinbase-app/api-architecture/rate-limiting)。
:::

## 重连与重新订阅

WebSocket 客户端在重连时使用基数为 250ms、上限为 30s 的指数退避。重连
后,订阅会按创建顺序自动恢复。Coinbase 要求在连接后 5 秒内发送订阅
消息,否则服务器会断开连接;适配器会在 WebSocket 握手完成后立即发送
已排队的订阅。

对于已认证的频道(`user`,以及 Margin 客户端上的
`futures_balance_summary`),适配器会为每条订阅消息生成一个新的 JWT;
根据 Coinbase 文档,“你必须为每条发送的 websocket 消息生成不同的
JWT,因为 JWT 会在 120 秒后过期”。一旦某个订阅被接受,数据流会在
WebSocket 连接的生命周期内持续,无需进一步身份验证。

当执行客户端的 WebSocket 重连时,内部客户端会从头重建(而非依赖现有
连接的状态机),以确保即使前一个会话的 `Disconnect` 命令与关闭信号
发生竞态,也能获得一组全新的 `cmd_tx`/`out_rx`/信号三元组。按订单的
累积跟踪会在重连后持续保留,因此合成的成交增量仍然正确。

## 配置

### 数据客户端配置选项

| 选项                             | 默认值     | 说明                                                                       |
|------------------------------------|-------------|-------------------------------------------------------------------------------------|
| `api_key`                          | `None`      | 回退到 `COINBASE_API_KEY` 环境变量。                                         |
| `api_secret`                       | `None`      | 回退到 `COINBASE_API_SECRET` 环境变量。                                      |
| `base_url_rest`                    | `None`      | REST 基础 URL 的覆盖值。                                                   |
| `base_url_ws`                      | `None`      | 市场数据 WebSocket URL 的覆盖值。                                       |
| `proxy_url`                        | `None`      | HTTP 与 WebSocket 传输的可选代理 URL。                             |
| `environment`                      | `Live`      | `Live` 或 `Sandbox`。                                                              |
| `http_timeout_secs`                | `10`        | HTTP 请求超时时间(秒)。                                                   |
| `ws_timeout_secs`                  | `30`        | WebSocket 超时时间(秒)。                                                      |
| `update_instruments_interval_mins` | `60`        | 标的目录刷新间隔。                                  |
| `derivatives_poll_interval_secs`   | `15`        | 发出 `IndexPriceUpdate` 和 `FundingRateUpdate` 的 REST 轮询间隔。 |
| `transport_backend`                | `Sockudo`   | WebSocket 传输后端。                                                      |

### 执行客户端配置选项

| 选项                   | 默认值   | 说明                                                                                              |
|--------------------------|-----------|------------------------------------------------------------------------------------------------------------|
| `api_key`                | `None`    | 回退到 `COINBASE_API_KEY` 环境变量。                                                                |
| `api_secret`             | `None`    | 回退到 `COINBASE_API_SECRET` 环境变量。                                                             |
| `base_url_rest`          | `None`    | REST 基础 URL 的覆盖值。                                                                          |
| `base_url_ws`            | `None`    | 用户数据 WebSocket URL 的覆盖值。                                                                |
| `proxy_url`              | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。                                                    |
| `environment`            | `Live`    | `Live` 或 `Sandbox`。                                                                                     |
| `http_timeout_secs`      | `10`      | HTTP 请求超时时间(秒)。                                                                          |
| `max_retries`            | `3`       | HTTP 请求的最大重试次数。                                                                |
| `retry_delay_initial_ms` | `100`     | 初始重试延迟(毫秒)。                                                                      |
| `retry_delay_max_ms`     | `5000`    | 最大重试延迟(毫秒)。                                                                      |
| `account_type`           | `Cash`    | 现货为 `Cash`,CFM 衍生品为 `Margin`。参见 [执行范围](#执行范围)。                |
| `default_margin_type`    | `None`    | 应用于衍生品订单的默认 `CoinbaseMarginType`(`Cross` 或 `Isolated`)。在 Cash 上被忽略。     |
| `default_leverage`       | `None`    | 应用于衍生品订单的默认杠杆。在 Cash 上被忽略。                                         |
| `retail_portfolio_id`    | `None`    | CDP 零售组合 UUID。当 API key 绑定到非默认组合时为必填(否则场所会以 `account is not available` 拒绝订单)。参见 [组合](#组合-portfolios)。 |
| `transport_backend`      | `Sockudo` | WebSocket 传输后端。                                                                             |

配置可通过 PyO3 导出的类型从 Python 构造:

```python
from nautilus_trader.core.nautilus_pyo3 import CoinbaseDataClientConfig
from nautilus_trader.core.nautilus_pyo3 import CoinbaseExecClientConfig
from nautilus_trader.core.nautilus_pyo3 import CoinbaseEnvironment

data_config = CoinbaseDataClientConfig(
    api_key="YOUR_COINBASE_API_KEY",
    api_secret="YOUR_COINBASE_API_SECRET",
    environment=CoinbaseEnvironment.LIVE,
)

exec_config = CoinbaseExecClientConfig(
    api_key="YOUR_COINBASE_API_KEY",
    api_secret="YOUR_COINBASE_API_SECRET",
    environment=CoinbaseEnvironment.LIVE,
)
```

v2 系统直接从这些配置实例化 Rust 工厂;无需 Python 工厂接线。

## 已知限制

### 场所侧

- 订单修改仅限于 GTC 订单(LIMIT、STOP_LIMIT、Bracket);其他类型必须
  使用取消重下。
- OCO 订单未作为独立订单类型暴露。
- 跟踪止损、MARKET_IF_TOUCHED、LIMIT_IF_TOUCHED 以及冰山订单均未被
  场所暴露。
- 批量提交和批量修改不可用;仅提供批量取消。
- 沙盒是一个静态模拟环境(仅 Accounts 和 Orders 端点,响应预先定义,
  没有真实市场数据)。
- 用户频道 WebSocket 报告的是按订单的累积状态,而非按笔成交。执行
  客户端从累积增量中推导每笔成交的数量、价格和手续费;按笔的
  `trade_id` 由 `(venue_order_id, cumulative_quantity)` 合成。

### 适配器侧

- **实时路径与 REST 路径的稳定成交身份不同。** 用户频道不提供
  Coinbase 按笔成交的 `trade_id`,因此实时的 `FillReport` 使用由场所
  订单 ID 和累积数量合成的 ID。REST 对账使用场所的 `trade_id`。因此
  Coinbase 实盘执行尚未被批准作为 v2 切换的证据。
- **每个客户端仅支持一个产品系列。** 下单、修改、取消以及报告生成都
  被过滤到已配置的产品系列(`AccountType::Cash` 下为现货;
  `AccountType::Margin` 下为永续 + 到期期货)。标的不在启动时缓存
  范围内的订单会被拒绝。参见 [执行范围](#执行范围)。
- **Cash 账户的仓位报告始终为空。** Coinbase 现货没有仓位。衍生品
  (CFM)仓位报告来自 `cfm/positions`,仅出现在 Margin 客户端上。
- **用户频道更新省略 `price`、`stop_price` 和 `trigger_type`。**
  对于该客户端提交的订单,缺失的字段会从 `submit_order` 时填充的
  缓存中修补。对于外部订单(由其他进程或通过 Coinbase UI 提交),
  用户频道处理器会在首次看到该订单时通过获取
  `/orders/historical/{venue_order_id}` 来丰富报告,并缓存结果。
  该 REST 调用会为外部订单的首次用户频道更新增加延迟;后续更新使用
  缓存的丰富信息。
- **全部取消和批量取消的 REST 列表失败仅会被记录。** 如果列出未成交
  订单的 REST 调用失败,不会发出按订单的 `OrderCancelRejected`;订单
  会保持在 `PendingCancel`,直到下一次对账将其恢复。这与 Bybit
  适配器的模式相同。
- **新上市的产品需要重新连接才能交易。** 标的缓存在连接时填充;在此
  之后上市的产品不在缓存中,`submit_order` 会拒绝它们。
- **MARKET 订单默认使用 IOC。** 以 Nautilus 默认值
  `TimeInForce::Gtc` 构造的 `MarketOrder` 会在场所被映射为
  `market_market_ioc`。显式的 `TimeInForce::Ioc` 会被遵守;
  `TimeInForce::Fok` 会路由到 `market_market_fok`,但在现货上会在
  运行时被撮合引擎以 `UNSUPPORTED_ORDER_CONFIGURATION` 拒绝(该传输层
  格式在 API 规范中有记录,但仅在 CFM 衍生品上被接受)。`Day` 和
  `Gtd` 会在提交时被拒绝。

## 已认证的二进制程序

两个二进制程序可用于协助实盘验证和账户维护:

- `coinbase-http-private` 列出组合、打印钱包余额、为 `BTC-USD` 和
  `BTC-USDC` 运行 `/orders/preview`,并展示按产品的门控标志。这是接入
  新账户时建议首先运行的工具。
- `coinbase-cancel-all-open` 取消已认证 CDP key 上所有未成交订单。
  可用于在测试运行之间清理挂单。

两者都从环境变量读取 `COINBASE_API_KEY` 和 `COINBASE_API_SECRET`。

## 贡献

:::info
如需了解更多功能或为 Coinbase 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
