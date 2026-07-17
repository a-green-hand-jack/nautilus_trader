# Derive

Derive(前身为 Lyra)是一个去中心化衍生品交易场所,提供欧式期权和现金结算的
永续互换,是链上最大的期权市场之一。交易通过 Derive Chain 上每个用户专属的
智能合约钱包完成,因此抵押品始终由用户自行托管,而订单则通过场所的订单簿撮合。

Derive Chain 是一个结算到以太坊的 Optimistic Rollup。订单在链下撮合,在链上
结算,将订单簿撮合与自我托管结合起来。订单通过针对某个子账户(subaccount)的
会话密钥(session key)以 EIP-712 类型化数据签名进行授权,这使得签名密钥与
钱包所有者相互独立,用户可以在不移动资金的情况下轮换或撤销访问权限。

## 示例

Rust 示例测试程序位于
[`crates/adapters/derive/examples/`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/adapters/derive/examples/)。

## 概览

Derive 适配器以 Rust 实现,位于 `crates/adapters/derive`。它对外暴露:

- `DeriveHttpClient`:与 `api.lyra.finance`(主网)或 `api-demo.lyra.finance`(测试网)的底层 REST 连接。
- `DeriveWebSocketClient`:JSON-RPC WebSocket 传输,包含订阅跟踪、重连以及签名下单(WebSocket 交易 API)。
- `DeriveInstrumentProvider`:按币种获取并缓存标的。
- `DeriveDataClient`:实时市场数据客户端。
- `DeriveDataClientFactory`:供实时节点构建器使用的数据客户端工厂。
- `DeriveExecutionClient`:用于签名下单、取消、查询与报告流程的实时执行客户端。
- `DeriveExecutionClientFactory`:供实时节点构建器使用的执行客户端工厂。

执行流程使用针对 Derive Chain 各操作模块合约的 EIP-712 类型化数据签名。

## Derive 文档

Derive 在 [docs.derive.xyz](https://docs.derive.xyz) 发布了 API 文档。建议
将其与本指南结合参考以获取更多细节。

## 产品

| 产品类型           | 是否支持 | 备注                                                                |
|------------------------|-----------|------------------------------------------------------------------------|
| ERC-20 现货            | ✓         | 以 USDC 计价的交易对,如 `ETH-USDC`;解析为 `CurrencyPair`。      |
| 永续互换        | ✓         | 以 USDC 现金结算,按币种上市,如 `ETH-PERP`。 |
| 期权(看涨/看跌) | ✓         | 使用 `{CURRENCY}-{EXPIRY}-{STRIKE}-{C|P}` 格式的欧式期权。   |

## 符号规则

Derive 标的使用场所原生符号,并附加场所后缀 `.DERIVE`:

- 现货:`ETH-USDC.DERIVE`(基础货币、计价货币)。
- 永续:`ETH-PERP.DERIVE`、`BTC-PERP.DERIVE`。
- 期权:`ETH-20260626-3000-C.DERIVE`(币种、到期日、行权价、类型)。

符号中以连字符分隔的第一段是标的币种。数据提供者会按币种调用一次
`public/get_instruments`,因此当启用 `auto_load_missing_instruments`
(默认启用)时,订阅一个新币种会触发一次惰性的 REST 获取。

适配器根据场所的 `instrument_type`(`perp`、`option`、`erc20`)进行路由,
而非依据符号后缀,因此现货交易对不需要特殊的符号解析。现货复用与永续和
期权相同的 Trade 模块签名路径;仓库内 `crates/adapters/derive/test_data/spot/`
下的测试夹具捕获了解析器和执行路径所依赖的现货标的、订单簿、ticker 与
成交字段格式。

:::warning
现货交易的实盘验证程度低于永续和期权。测试网可以接受并取消一个最小数量为
`0.1 ETH` 的被动 `ETH-USDC` 限价单,主网的下单/取消也已手动验证过。公开的
现货成交频道(`trades.erc20.ETH`、`trades.ETH-USDC`)能够成功订阅,但成交量
可能较低,因此预期成交帧会比较稀疏。
:::

## 环境

通过任一客户端配置中的 `DeriveEnvironment` 枚举来配置环境。

| 环境 | 配置                       | REST                            | WebSocket                        |
|-------------|------------------------------|----------------------------------|-----------------------------------|
| 主网     | `DeriveEnvironment::Mainnet` | `https://api.lyra.finance`      | `wss://api.lyra.finance/ws`      |
| 测试网     | `DeriveEnvironment::Testnet` | `https://api-demo.lyra.finance` | `wss://api-demo.lyra.finance/ws` |

测试网是一条独立的链,拥有自己的会话密钥与余额;主网与测试网的 API key
不可互换。公开市场数据(订单簿、ticker、成交)不需要凭证。

两个网络的 EIP-712 协议常量(`DOMAIN_SEPARATOR`、`ACTION_TYPEHASH`、各操作
模块地址)都内置在 `crates/adapters/derive/src/common/consts.rs` 中,并与
Derive 的[协议常量参考文档](https://docs.derive.xyz/reference/protocol-constants)
保持同步跟踪。`DeriveExecClientConfig::domain_separator`、`action_typehash`
与 `trade_module_address` 支持按实例覆盖,覆盖值优先于内置值。

## 测试网入门

Derive 在 Web 应用中将演示环境称为 “testnet”,而在 API 主机名中称为 “demo”。
本指南使用 “testnet” 以与仪表盘及我们的 `DeriveEnvironment::Testnet` 枚举保持
一致。达到执行客户端可以提交已签名订单状态所需的步骤:

1. **登录测试网仪表盘。** 打开
   [testnet.derive.xyz](https://testnet.derive.xyz),连接一个 EVM 钱包
   (MetaMask、WalletConnect、社交登录等)。这是授权下方智能合约钱包的
   所有者 EOA。
2. **注册 Derive Chain 智能合约钱包。** 首次登录会在 Derive 测试网链上部署一个
   每用户专属的智能合约钱包。在 “Developers” -> “Derive Wallet” 下显示的地址
   即为客户端使用的 `wallet_address`(以及 `X-LYRAWALLET` 请求头),它与你
   刚连接的 EOA 是不同的地址。
3. **创建子账户。** 在该钱包下开设一个子账户(Standard Margin 是最简单的
   测试交易模式)。这个整数 id 即为客户端签名每次 `private/order` 请求所
   使用的 `subaccount_id`。
4. **生成会话密钥。** 在 “Developers” -> “Session Keys” 下,创建一个作用域为
   该子账户的会话密钥,并复制原始的 secp256k1 私钥。这即为 `session_key`
   值;它不会离开客户端,并在 `Debug` 输出中被脱敏。会话密钥可以在同一
   面板中轮换或撤销。
5. **通过水龙头为子账户注资。** 测试网仪表盘提供一个 USDC 水龙头,可以
   滴发测试抵押品。将其存入子账户,使链上余额显示非零抵押品;在子账户
   的保证金不足以支持所需下单数量之前,API 会拒绝订单。
6. **设置环境变量。** 导出客户端在测试网模式下读取的三个值(或在
   `DeriveExecClientConfig` 中直接传入,配置字段的优先级更高):

   ```bash
   export DERIVE_TESTNET_WALLET_ADDRESS="0x..."  # Derive Chain 智能合约钱包
   export DERIVE_TESTNET_SESSION_PRIVATE_KEY="0x..."  # secp256k1 会话密钥私钥
   export DERIVE_TESTNET_SUBACCOUNT_ID="12345"  # 整数子账户 id
   ```

### 最小资金要求

该场所没有固定的最小值。撮合引擎接受任何满足子账户对应仓位初始保证金
要求的订单。可将以下值作为最小可行测试的实际下限:

- **冒烟测试(仅提交与取消,不成交):** 任意正的 USDC 余额即可覆盖已签名
  下单流程。
- **完成一次 `ETH-PERP` 成交往返:** 需预留最坏情况下经滑点调整后的名义
  金额,加上初始保证金缓冲。以单份合约按 $3500、场所约 10% 初始保证金
  计算,大约需要 $350 抵押品加 $400 缓冲。首次成交测试保持约 $1000 USDC
  的余额较为宽裕。
- **期权:** 期权的初始保证金要求高于永续。可调用 `public/get_instrument`
  获取该期权信息,用合约规模乘以标记价格,再加上期权特定的初始保证金
  (可在标的响应中查看),据此确定存款规模。

注资后,可使用 `private/get_subaccount` 端点确认针对计划提交订单的
`initial_margin`/`maintenance_margin` 余量;适配器的 `query_account` 命令
会将该快照作为 `AccountState` 事件发出,以便策略层据此判断是否可以交易。

## 主网入门

主网入门流程与测试网类似,只是面向生产环境仪表盘,且使用真实资金。

1. **登录主网仪表盘。** 打开 [derive.xyz](https://derive.xyz),连接 EVM
   所有者钱包(MetaMask、WalletConnect、社交登录等)。首次登录会部署你的
   Derive Chain 智能合约钱包。
2. **复制钱包地址。** 在 “Developers” -> “Derive Wallet” 下复制智能合约钱包
   地址。这即为客户端签名所用的 `wallet_address`;它与你登录所用的 EOA
   **不同**。可在 Derive Chain 浏览器上验证该地址是否带有合约代码(EOA 没有)。
3. **创建或选择子账户。** 在该钱包下开设一个子账户(Standard Margin 是最
   简单的模式;只有在理解全仓保证金语义后,才切换到 Portfolio Margin)。
   这个整数 id 即为 `subaccount_id`。
4. **生成主网会话密钥。** 在 “Developers” -> “Session Keys” 下,创建一个
   作用域为该子账户的会话密钥,并复制原始的 secp256k1 私钥。会话密钥可以
   在同一面板中轮换或撤销;对于探索性测试运行,建议使用短期有效的密钥。
5. **为子账户注资。** 通过仪表盘的存款流程,向子账户存入 USDC(或其他
   受支持的抵押品)。提交前请通过 `private/get_subaccount`(或适配器的
   `query_account`)确认 `collaterals_value` 和 `initial_margin` 余量
   足以覆盖计划下的订单。
6. **设置环境变量。** 导出三个主网值(或在 `DeriveExecClientConfig`
   中直接传入,配置字段的优先级更高):

   ```bash
   export DERIVE_WALLET_ADDRESS="0x..."  # Derive Chain 智能合约钱包
   export DERIVE_SESSION_PRIVATE_KEY="0x..."  # secp256k1 会话密钥私钥
   export DERIVE_SUBACCOUNT_ID="12345"  # 整数子账户 id
   ```

   三个 Rust 示例(`node_data_tester`、`node_exec_tester`、
   `node_delta_neutral`)都通过 `const DERIVE_ENVIRONMENT: DeriveEnvironment =
   DeriveEnvironment::Testnet;` 字面量固定网络;若要使用真实资金运行,
   请将其切换为 `DeriveEnvironment::Mainnet`。生产部署通过
   `DeriveDataClientConfig::environment` / `DeriveExecClientConfig::environment`
   选择网络。

## 能力

### 市场数据

| 能力                     | 是否支持 | 备注                                                                   |
|--------------------------------|-----------|---------------------------------------------------------------------------|
| 请求标的(REST)      | ✓         | `public/get_instrument`;将单个标的加载到本地缓存。     |
| 请求全部标的(REST) | ✓         | `public/get_instruments`;为每个币种打捞有效行。        |
| 标的订阅        | -         | *不支持。* 使用配置的 REST 刷新间隔。              |
| 订单簿增量(L2_MBP)     | ✓         | 频道:`orderbook.{instrument}.{group}.{depth}`。                      |
| 订单簿深度 10(L2_MBP)    | ✓         | 使用相同订单簿频道,`depth=10`。                                |
| 按间隔的订单簿         | -         | *不支持。* 需在本地根据增量维护间隔订单簿。           |
| 订单簿快照(REST)     | -         | *不支持。* 该适配器未暴露此功能。                                |
| 历史订单簿增量(REST)  | -         | *不支持。* 该适配器未暴露此功能。                                |
| 报价(`ticker_slim`)         | ✓         | 频道:`ticker_slim.{instrument}.{interval}`。                         |
| 报价快照(REST)          | ✓         | 一次性调用 `public/get_tickers`;发出单条 `QuoteTick`。              |
| 历史报价(REST)       | -         | *不支持。* 该场所仅暴露 ticker 快照。                               |
| 成交                         | ✓         | 频道:`trades.{instrument_type}.{currency}`。                         |
| 历史成交(REST)       | ✓         | 按时间顺序去重;`limit` 保留最新的成交记录。      |
| K 线 / OHLC(REST)             | ✓         | 已收盘的分钟、小时、日、周 K 线,按桶收盘时间打上时间戳。        |
| K 线 / OHLC(WS)               | -         | *不支持。* 该场所没有 K 线订阅频道。          |
| 标记价格流              | ✓         | 派生自 `ticker_slim`;与报价订阅共享。              |
| 指数价格流             | ✓         | 派生自 `ticker_slim`;与报价订阅共享。              |
| 资金费率流            | ✓         | 派生自永续 ticker 上的 `perp_details.funding_rate`。               |
| 资金费率历史(REST)    | ✓         | 按时间顺序,针对永续合约;`limit` 保留最新的有效行。    |
| 标的状态              | -         | *不支持。* Ticker 载荷中包含 `is_active`。                   |
| 标的收盘              | -         | *不支持。* 期权结算仅通过 REST 提供。                   |
| 期权 Greeks                  | ✓         | 派生自期权 ticker 上的 `option_pricing`。                        |
| 期权链                   | ✓         | 由报价和 Greeks 聚合而成;`public/get_tickers` 初始化平值(ATM)。 |

`request_instrument` 会针对请求的 `InstrumentId` 调用 `public/get_instrument`,
并在发出响应之前缓存返回的定义。缓存的标的携带后续报价、成交、订单簿和
K 线解析所使用的精度与最小变动单位字段。

标的加载会将场所错误 `12001` 视为该受影响产品类型的空结果,因此某个币种
没有永续、期权或现货上市不会阻塞其其他产品的加载。无效的标的行会被记录
并跳过,同时有效行会继续加载。

历史请求使用 `public/get_trade_history`、`public/get_tradingview_chart_data`
以及 `public/get_funding_rate_history`。K 线的 `end` 边界仍按场所对应桶的
起始时间来选取。响应会省略任何收盘时间晚于请求时间的桶,包括场所返回的
仍在形成中的桶。

Derive 通过同一个 `orderbook.{instrument}.{group}.{depth}` 频道系列暴露
订单簿增量和深度 10 快照。`subscribe_book_deltas` 会将快照增量以
`OrderBookDeltas` 发布,而 `subscribe_book_depth10` 固定 `depth=10` 并发布
`OrderBookDepth10` 快照。

### 执行

下单、取消、修改、查询与报告生成均使用 Derive 的 EIP-712 自托管签名流程。
下单写操作(`private/order`、`private/cancel`、`private/cancel_all`、
`private/replace`)通过 WebSocket 交易 API 在同一个已认证会话上发送,
该会话也通过私有频道(`{subaccount_id}.orders`、`{subaccount_id}.trades`、
`{subaccount_id}.balances`)流式推送账户、订单、成交与余额状态。无论使用
哪种传输方式,已签名的 EIP-712 消息体都相同。

:::note
HTTP 下单端点仍保留在 `DeriveHttpClient` 中,供工具和测试使用,但实时
执行客户端会将所有写操作通过 WebSocket 交易 API 路由。报告生成、账户
刷新与标的查询仍使用 REST。
:::

永续、期权与 ERC-20 现货交易对都使用 Derive 的 Trade 模块。现货没有单独的
签名路径,对账时也将现货标的与其他标的类别一视同仁,唯有下文所述的
reduce-only 保护是例外。

适配器支持普通的 `private/order` 请求:带 `GTC`、`IOC` 或 `FOK` 有效期的
`LIMIT` 与 `MARKET` 订单。它还支持 Derive 的触发单,用于下方列出的
Nautilus 原生止损与触及型订单类型。不受支持的 Nautilus 订单类型会在签名
之前被拒绝,因此无法在场所成交。

市价单在提交前需要一个已缓存的报价。在异步提交任务解析出标的之后,它会
刷新当前 ticker 快照,并根据该刷新后的报价推导出已签名的滑点限价
`limit_price`。

#### 条件单

Derive 的触发单使用仅限 WebSocket 的 `private/trigger_order` 端点,而非
普通的 `private/order` 端点。该场所会以 `order_status=untriggered` 状态
存储这些订单,直到其触发工作进程提交已签名的子订单。因此对账会同时读取
`private/get_open_orders` 与 `private/get_trigger_orders`。

Derive 主网要求触发单签名的有效期在场所时间的 30 到 90 天之间。适配器
以固定的 31 天有效期签名触发单;`signature_expiry_secs` 仍控制普通的
`private/order` 与 `private/replace` 写操作,且必须大于场所要求的
300 秒最小值。

| Nautilus 订单类型 | 是否支持 | Derive `order_type` | Derive `trigger_type` | 备注                         |
|---------------------|-----------|---------------------|-----------------------|-------------------------------|
| `StopMarket`        | ✓         | `market`            | `stoploss`            | 使用触发价格作为限价边界。  |
| `StopLimit`         | ✓         | `limit`             | `stoploss`            | 同时发送限价和触发价格。 |
| `MarketIfTouched`   | ✓         | `market`            | `takeprofit`          | 使用触发价格作为限价边界。  |
| `LimitIfTouched`    | ✓         | `limit`             | `takeprofit`          | 同时发送限价和触发价格。 |
| `MarketToLimit`     | -         | -                   | -                     | *Derive 不支持*。    |
| 跟踪止损      | -         | -                   | -                     | *Derive 不支持*。    |
| TWAP / 算法单 / RFQ   | -         | -                   | -                     | *该适配器未暴露*。 |

适配器将 Nautilus 的 `TriggerType::Default` 与 `TriggerType::MarkPrice` 映射
为 Derive 的 `trigger_price_type=mark`。根据 Derive 目前的错误码参考文档,
指数价与最新成交价触发类型尚不受支持,因此 `IndexPrice`、`LastPrice`、
`BidAsk` 以及其他触发价格类型会在签名之前被本地拒绝。

Derive 错误 `11054` 表明触发单不能被替换,也不能替换其他订单。因此适配器
会以 `OrderModifyRejected` 事件拒绝针对触发单的 Nautilus 修改请求;若要
更新触发单,需取消后重新提交。

Derive 会校验触发价格所在的方向,若触发价未按预期方向越过当前价格,会以
错误 `11051` 拒绝。触发价格在订单签名时即已固定,因此对于价格波动快或
价位较高的标的,过窄的偏移量可能会在场所收到订单之前就漂移到错误的一侧。
应将触发偏移设置得足以充分覆盖提交期间的预期价格波动(以 `ETH-PERP` 为例,
应为几十美元,而非几美分);偏移过窄会产生虚假的 `11051` 拒绝。

#### 执行指令

| 指令   | 是否支持 | Derive 取值  | 备注                                                         |
|---------------|-----------|---------------|-----------------------------------------------------------------|
| `post_only`   | ✓         | `post_only`   | 需要 `GTC`;若会吃单则拒绝。    |
| `reduce_only` | ✓         | `reduce_only` | 仅限永续与期权,仅市价或 `IOC`/`FOK`;现货不支持。   |

#### 有效期

Derive 将 `gtc`、`post_only`、`fok` 和 `ioc` 记录为其 `time_in_force` 取值。
适配器会在签名之前拒绝在 Derive 中没有对应值的 Nautilus 值。Derive 将
post-only 作为一种 `time_in_force` 取值暴露,因此 `post_only` 不能与
`IOC` 或 `FOK` 组合使用。

| 有效期  | 是否支持 | Derive 取值 | 备注                      |
|----------------|-----------|--------------|----------------------------|
| `GTC`          | ✓         | `gtc`        | 撤销前有效。        |
| `IOC`          | ✓         | `ioc`        | 立即成交或取消。       |
| `FOK`          | ✓         | `fok`        | 全部成交或取消。              |
| `GTD`          | -         | -            | *Derive 不支持*。 |
| `DAY`          | -         | -            | *Derive 不支持*。 |
| `AT_THE_OPEN`  | -         | -            | *Derive 不支持*。 |
| `AT_THE_CLOSE` | -         | -            | *Derive 不支持*。 |

#### 现货 reduce-only 订单

Derive 现货没有仓位概念,因此 reduce-only 现货订单永远无法“减少”任何仓位。
场所始终会以错误 `11025` 拒绝此类订单;当适配器已知某标的为现货时,会
避免这次无谓的往返请求。缓存的现货标的会直接以 `OrderDenied` 拒绝;惰性
解析出的现货标的会在提交时以 `OrderRejected` 拒绝。

永续与期权的 reduce-only 订单仍会到达场所,其结果取决于该子账户当时的
仓位状态。`derive-flatten` 程序只会平掉衍生品仓位,永远不会处理现货,
因为清空现货余额会将基础资产倾销为另一种计价货币。

Derive 仅对市价单或非挂单型限价单(`IOC`/`FOK`)承认 `reduce_only`。带
`reduce_only` 的挂单型 `GTC` 或 post-only 限价单会被场所以错误
`11024 Reduce only not supported with this time in force` 拒绝。因此,在
Derive 上,一个止盈腿为 reduce-only `GTC` 限价单的 Nautilus 括号订单
(bracket order)无法挂单:入场腿和止损腿可以提交,但止盈腿会被拒绝。
如果目标是 Derive,请使用 reduce-only 的 `IOC`/`FOK` 平仓单,或使用非
reduce-only 的止盈单。

#### 订单拒绝语义

会改变状态的写操作(`submit_order`、`modify_order`、`cancel_order`)通过
WebSocket 交易 API 发送一次,不会重放。适配器根据 WebSocket 请求的结果
来区分终态处理与模糊处理。对于场所明确失败的情况,它会发出终态拒绝事件
(`OrderRejected`、`OrderModifyRejected`、`OrderCancelRejected`):

- 已签名操作被拒绝,如参数无效、保证金不足或未知订单。
- 场所业务错误码,如 `11009 Zero liquidity`。
- Post-only 吃单拒绝(`11008 Post only order cannot cross the market`),
  以带 `due_post_only=true` 的 `OrderRejected` 报告。
- 速率限制响应(`-32000 Rate limit exceeded`),此时网关在撮合引擎看到
  该请求之前就已拒绝。

对于到达场所的 post-only 订单,若发生吃单,Derive 会以 JSON-RPC `11008`
及消息 `Post only order cannot cross the market` 拒绝。适配器会将该终态
拒绝标记为 `due_post_only=true`;若某条 WebSocket/订单报告拒绝携带相同
原因,跟踪中的订单路径也会应用相同的分类。对于本地拒绝的、不受支持的
post-only + IOC/FOK 组合,不会标记 `due_post_only`,因为它们并不代表
场所层面的吃单拒绝。

对于结果模糊的写操作,适配器不会发出终态事件,而是让 WebSocket 对账或
后续状态报告去最终确定状态。这类模糊情形被刻意限定得很窄:

- `-32603`,通用的 JSON-RPC 内部错误。
- 无法解码的响应(该操作可能已被处理)。
- 请求超时、重连期间丢失的响应,以及传输层错误。

这一区分保护了订单生命周期的两端:错误的终态拒绝可能让引擎将一个实际
存活的订单当作已拒绝处理;错误的模糊判定则可能让一个未成功下达的订单
永远悬挂在 `Submitted` 状态,因为不会有任何 WebSocket 帧到达。

## 订阅参数

`subscribe_book_deltas` 与 `subscribe_book_depth10` 接受以下 `subscribe_params` 键:

| 键      | 类型   | 默认值 | 允许取值              |
|----------|--------|---------|----------------------|
| `group`  | string | `"1"`   | `"1"`、`"10"`、`"100"` |
| `depth`  | string | `"10"`  | `"1"`、`"10"`、`"20"`、`"100"` |

`subscribe_quotes` 接受:

| 键        | 类型   | 默认值  | 允许取值           |
|------------|--------|----------|-------------------|
| `interval` | string | `"1000"` | `"100"`、`"1000"` |

未知取值会在订阅时被拒绝。

### 共享的 ticker 订阅

报价、标记价格、指数价格、资金费率与期权 Greeks 都派生自同一个
`ticker_slim.{instrument}.{interval}` WebSocket 订阅。适配器对底层 WS
订阅调用进行引用计数:某标的第一个被订阅的数据流会打开该频道,最后一个
取消订阅会关闭它。因此,以第一次订阅时的 `interval` 为准;随后以不同
interval 订阅的数据流会共享已有频道。

标记价格、指数价格、资金费率和期权 Greeks 均从 ticker 载荷中读取字段。
完整的 ticker 结构和精简的 `SlimEnvelope` 结构都携带这些字段,因此派生
数据流在两种结构下都能正常工作:`mark_price` 和 `index_price` 是必需的
(缺失这些字段的帧会解析失败并被记录,而不是被静默丢弃),而
`funding_rate` 和 `option_pricing` 是可选的,仅在相关标的类别下存在。
报价数据流始终可用,因为买卖价在两种结构中都存在。

资金费率仅对永续合约有意义,期权 Greeks 仅对期权有意义。为某个标的的
类别订阅不匹配的数据流(例如为期权订阅资金费率)会被接受,WebSocket
订阅也会正常打开,但解析器不会为该数据流返回任何事件,因为场所载荷中
缺少相关字段(非永续合约没有 `perp_details`,非期权没有
`option_pricing`)。在订阅衍生品特定的数据流之前,请先确认标的类别。

## 配置

### 数据客户端配置选项

类/结构体:`DeriveDataClientConfig`。

| 选项                             | 默认值   | 说明 |
|------------------------------------|-----------|-------------|
| `base_url_rest`                    | `None`    | REST 基础 URL 的覆盖值。 |
| `base_url_ws`                      | `None`    | WebSocket 基础 URL 的覆盖值。 |
| `proxy_url`                        | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `environment`                      | `Mainnet` | 网络选择(Python 中为 `MAINNET` 或 `TESTNET`)。 |
| `http_timeout_secs`                | `10`      | REST 请求超时时间(秒)。 |
| `ws_timeout_secs`                  | `30`      | WebSocket 连接与空闲超时时间(秒)。 |
| `update_instruments_interval_mins` | `60`      | 标的刷新间隔(分钟)。 |
| `currencies`                       | `[]`      | 连接时批量加载的币种。为空表示按需惰性加载。 |
| `include_expired`                  | `false`   | 是否包含 `public/get_instruments` 中已过期的期权行。 |
| `auto_load_missing_instruments`    | `true`    | 在订阅或请求命令之前惰性加载未知标的。 |
| `transport_backend`                | `Sockudo` | 启用 `transport-sockudo` 时使用的 WebSocket 传输后端。 |

### 执行客户端配置选项

类/结构体:`DeriveExecClientConfig`。

| 选项                             | 默认值   | 说明 |
|------------------------------------|-----------|-------------|
| `wallet_address`                   | `None`    | Derive Chain 智能合约钱包地址。缺省时回退到下方环境变量。 |
| `session_key`                      | `None`    | secp256k1 会话密钥私钥。缺省时回退到下方环境变量。 |
| `subaccount_id`                    | `None`    | Derive 子账户 id。缺省时回退到下方环境变量。 |
| `base_url_rest`                    | `None`    | REST 基础 URL 的覆盖值。 |
| `base_url_ws`                      | `None`    | WebSocket 基础 URL 的覆盖值。 |
| `proxy_url`                        | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `environment`                      | `Mainnet` | 网络选择(Python 中为 `MAINNET` 或 `TESTNET`)。 |
| `http_timeout_secs`                | `10`      | REST 请求超时时间(秒)。 |
| `max_retries`                      | `3`       | 可恢复读取以及明确非写入路径的重试次数。 |
| `retry_delay_initial_ms`           | `100`     | 初始重试延迟(毫秒)。 |
| `retry_delay_max_ms`               | `5000`    | 最大重试延迟(毫秒)。 |
| `max_fee_per_contract`             | 必填  | 每份合约的正数 USDC 手续费上限,会被签入每笔订单。 |
| `domain_separator`                 | `None`    | 可选的 EIP-712 domain separator 覆盖值。 |
| `action_typehash`                  | `None`    | 可选的 EIP-712 action typehash 覆盖值。 |
| `trade_module_address`             | `None`    | 可选的 Trade 模块合约地址覆盖值。 |
| `signature_expiry_secs`            | `600`     | 下单/替换的存活时间;必须大于 300 秒。触发单固定使用 31 天存活时间。 |
| `market_order_slippage_bps`        | `50`      | 市价单限价的滑点边界。 |
| `max_matching_requests_per_second` | `None`    | 撮合引擎写请求(创建/取消/替换)的最大速率(每秒)。缺省时默认为 Trader 等级的 1 次限制;做市商账户可调高。 |
| `transport_backend`                | `Sockudo` | 启用 `transport-sockudo` 时使用的 WebSocket 传输后端。 |

当构建时禁用 `transport-sockudo` 特性时,默认传输会回退为 `Tungstenite`。

`wallet_address`、`session_key` 与 `subaccount_id` 在未设置时,会回退到以下
环境变量:

| 字段            | 主网变量             | 测试网变量                     |
|------------------|------------------------------|--------------------------------------|
| `wallet_address` | `DERIVE_WALLET_ADDRESS`      | `DERIVE_TESTNET_WALLET_ADDRESS`      |
| `session_key`    | `DERIVE_SESSION_PRIVATE_KEY` | `DERIVE_TESTNET_SESSION_PRIVATE_KEY` |
| `subaccount_id`  | `DERIVE_SUBACCOUNT_ID`       | `DERIVE_TESTNET_SUBACCOUNT_ID`       |

会话密钥是注册在该钱包上、用于 API 签名的 secp256k1 私钥。`session_key`
字段在 `Debug` 输出和 Python `repr` 中会被脱敏。

### Python v2 实时节点

以 Rust 为底层的 Python v2 节点使用 `LiveNode.builder(...)`,并传入具体的
工厂实例。执行工厂需要 `DeriveExecFactoryConfig`,它将 trader 与账户标识符
与底层的 `DeriveExecClientConfig` 一起包装。

```python
from decimal import Decimal

from nautilus_trader.adapters.derive import DeriveDataClientConfig
from nautilus_trader.adapters.derive import DeriveDataClientFactory
from nautilus_trader.adapters.derive import DeriveEnvironment
from nautilus_trader.adapters.derive import DeriveExecClientConfig
from nautilus_trader.adapters.derive import DeriveExecFactoryConfig
from nautilus_trader.adapters.derive import DeriveExecutionClientFactory
from nautilus_trader.common import Environment
from nautilus_trader.live import LiveNode
from nautilus_trader.model import AccountId
from nautilus_trader.model import TraderId

trader_id = TraderId("TESTER-001")

data_config = DeriveDataClientConfig(
    environment=DeriveEnvironment.TESTNET,
    currencies=["ETH", "BTC"],
)

exec_config = DeriveExecClientConfig(
    environment=DeriveEnvironment.TESTNET,
    max_fee_per_contract=Decimal("1000"),
)

exec_factory_config = DeriveExecFactoryConfig(
    trader_id,
    AccountId("DERIVE-001"),
    exec_config,
)

node = (
    LiveNode.builder("DERIVE-001", trader_id, Environment.LIVE)
    .add_data_client(None, DeriveDataClientFactory(), data_config)
    .add_exec_client(None, DeriveExecutionClientFactory(), exec_factory_config)
    .build()
)
```

不要直接将 `DeriveExecClientConfig` 传给 `add_exec_client`;Derive 执行工厂
需要经过包装的 `DeriveExecFactoryConfig`,才能使用正确的 trader 与账户
标识符创建 `ExecutionClientCore`。

### Rust 数据客户端

```rust
use nautilus_derive::{
    common::enums::DeriveEnvironment,
    config::DeriveDataClientConfig,
};

let config = DeriveDataClientConfig {
    environment: DeriveEnvironment::Testnet,
    currencies: vec!["ETH".to_string(), "BTC".to_string()],
    ..Default::default()
};
```

值得关注的字段:

- `currencies`:连接时批量加载哪些币种。为空表示按订阅惰性加载。
- `include_expired`:是否包含 `public/get_instruments` 中已过期的期权行。
- `auto_load_missing_instruments`:当某标的未知时,在订阅时惰性加载。
- `update_instruments_interval_mins`:REST 刷新间隔(默认 60 分钟)。
- `http_timeout_secs`、`ws_timeout_secs`:传输超时设置。

### Rust 执行客户端

```rust
use nautilus_derive::{
    common::enums::DeriveEnvironment,
    config::DeriveExecClientConfig,
};
use rust_decimal::Decimal;

let config = DeriveExecClientConfig {
    wallet_address: Some("0x...".to_string()),
    session_key: Some("0x...".to_string()),
    subaccount_id: Some(1),
    environment: DeriveEnvironment::Testnet,
    max_fee_per_contract: Some(Decimal::from(1000)),
    ..Default::default()
};
```

`max_fee_per_contract` 为必填项,且必须大于零。如果该字段缺失或非正数,
执行客户端会在创建场所客户端之前就构造失败。

## 已知限制

- `request_instruments` 要求 `DeriveDataClientConfig::currencies` 中至少配置
  一个币种;场所的 `public/get_instruments` 端点是按币种作用域的,适配器
  不会枚举整个币种全集。
- `data_client.rs` 集成测试断言的是记录到的 REST 调用集合,而非顺序,
  因为 `fetch_instrument_definitions` 通过 `tokio::try_join!` 并行发出
  `perp` 和 `option` 请求。
- 场所不会推送标的状态、标的收盘或 K 线订阅;ticker 载荷携带保证金参数
  和 `is_active`,K 线仅通过 REST 提供。
- 场所未暴露订单簿快照 REST 端点,以及历史订单簿增量/历史报价端点。
  参见上方能力表。
- 自 2025 年 12 月 1 日起,Derive 官方 REST 文档已将 `public/get_ticker`
  标记为弃用,推荐使用 `public/get_tickers`。适配器在报价快照和期权链
  远期价格初始化中均使用 `public/get_tickers`。
