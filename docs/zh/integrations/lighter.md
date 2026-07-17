# Lighter

[Lighter](https://lighter.xyz) 是一个面向现货和永续期货的去中心化中央限价订单簿
交易所。该场所通过以太坊零知识 Rollup 进行结算,而撮合与排序在链下进行。

NautilusTrader 的 Lighter 适配器由 `nautilus-lighter` crate 实现。它提供 Rust
数据与执行客户端、带类型的 REST 与 WebSocket 模型,以及针对该场所 Schnorr /
ECgFp5 签名流程的内置 L2 交易签名器。

## 概览

该适配器由以下主要组件构成:

- `LighterRawHttpClient`:面向公共和账户端点的底层 REST 客户端。
- `LighterHttpClient`:领域客户端,将标的、成交、订单簿、订单和账户状态解析为
  Nautilus 模型类型。
- `LighterWebSocketClient`:支持重连的 WebSocket 客户端,用于公开市场和私有
  账户数据流。
- `LighterDataClient`:面向标的、成交、报价和 L2 MBP 订单簿的 Nautilus 数据客户端。
- `LighterExecutionClient`:面向账户流、下单、修改、取消以及对账报告的
  Nautilus 执行客户端。
- `LighterDataClientFactory` 与 `LighterExecutionClientFactory`:实时节点的
  工厂接入。

Python 层的暴露面刻意保持精简。Python 扩展只暴露配置、环境选择、工厂类以及
integrator 撤销功能;数据和执行客户端通过 Rust trait 层使用。

## 示例

该适配器提供 Python v2 和 Rust 实时节点示例。Python 示例位于
[`python/examples/lighter/`](https://github.com/nautechsystems/nautilus_trader/tree/develop/python/examples/lighter/),
默认执行“空跑”:构建节点、注册测试器,除非传入 `--run` 参数,否则会直接退出。

```bash
cd python
.venv/bin/python examples/lighter/data_tester.py --lighter-environment testnet
.venv/bin/python examples/lighter/exec_tester.py --lighter-environment testnet
```

传入 `--run` 以连接到 Lighter。执行测试器默认保持在 `dry_run` 模式,除非同时
传入 `--live-orders`。

```bash
cd python
.venv/bin/python examples/lighter/data_tester.py \
    --lighter-environment mainnet \
    --instrument BTC-PERP.LIGHTER \
    --run
.venv/bin/python examples/lighter/exec_tester.py \
    --lighter-environment mainnet \
    --instrument DOGE-PERP.LIGHTER \
    --run
```

Rust 示例位于 `crates/adapters/lighter/examples/`。数据测试器在运行时即会连接。
执行测试器运行时同样会连接,但默认 `DRY_RUN = true`;仅当你打算提交真实订单时,
才在源码中将其改为 `false`:

```bash
cargo run --example lighter-data-tester --package nautilus-lighter --features examples
cargo run --example lighter-exec-tester --package nautilus-lighter --features examples
```

:::warning
示例可能连接到实盘场所。启用实盘下单流的执行示例,若指向一个已注资的主网
账户,可能会实际提交订单。运行前请仔细检查所选标的、数量与环境。
:::

对于紧急账户清理,`cargo run --bin lighter-flatten -p nautilus-lighter`
会为配置的 Lighter 账户取消所有未成交订单并平掉所有仓位。它会扫描所有已注册
的市场,因此在标准的 60 请求/分钟 REST 配额下,运行可能需要几分钟。由于它是
账户级别而非策略级别的操作,请先检查活跃账户和仓位。

## 产品支持

| 产品类型      | 数据源 | 交易 | 备注                                                        |
|-------------------|-----------|---------|--------------------------------------------------------------|
| 现货              | ✓         | ✓       | 使用 Lighter 市场索引 2048-4094 的现货市场。         |
| 永续期货 | ✓         | ✓       | 使用 Lighter 市场索引 0-254 的线性永续市场。 |
| 到期期货     | -         | -       | *不支持*。                                             |
| 期权           | -         | -       | *不支持*。                                             |

## 限制

当前适配器的范围有意窄于该场所完整的交易能力范围:

- 未实现分组订单列表、OCO/OTO 分组、括号单、TWAP 以及冰山显示数量。
- 订单列表提交与批量取消会依次通过 WebSocket 拆分为独立的交易。两种操作
  均限制在每条命令最多 15 笔交易。
- 分组场所订单不在范围内:批量提交不使用 `CreateGroupedOrders`,也不提供
  原子性的 OCO/OTO 或括号单分组。
- `CancelAllOrders` 使用所请求标的的已缓存未成交订单。适配器不使用 Lighter
  原生的账户级全取消交易,因为它可能影响不相关的市场。
- 现货交易支持市价单和限价单。条件止损/止盈订单仅限于永续市场。
- 账户状态与仓位报告来自私有 WebSocket 数据流。`query_account` 和仓位状态
  生成会重放最新的缓存流状态。
- 未限定范围的订单对账被限制在已配置或已观察到的活跃市场内,以避免在标准
  REST 配额下对整个场所进行全量扫描。
- 历史成交请求使用公开的 `recentTrades` 端点,无需凭证。

## 符号规则

Lighter 通过数字 `market_index` 值来标识市场。适配器从
`GET /api/v1/orderBookDetails` 初始化映射关系,然后将场所原始符号转换为
Nautilus `InstrumentId`。

| 场所产品      | Nautilus 符号格式        | 示例                 | 备注                        |
|--------------------|-------------------------------|--------------------------|------------------------------|
| 永续期货  | `{BASE}-PERP.LIGHTER`         | `BTC-PERP.LIGHTER`      | 原始场所符号 `BTC`。      |
| 现货               | `{BASE}/{QUOTE}-SPOT.LIGHTER` | `ETH/USDC-SPOT.LIGHTER` | 原始场所符号 `ETH/USDC`。 |

后缀用于消歧现货与永续挂牌。现货符号保留了场所报价的交易对,而出站请求会
去掉后缀,改用缓存的 `market_index`。

## 环境

| 环境 | REST URL                              | WebSocket URL                              | Chain ID |
|-------------|---------------------------------------|--------------------------------------------|----------|
| 主网     | `https://mainnet.zklighter.elliot.ai` | `wss://mainnet.zklighter.elliot.ai/stream` | 304      |
| 测试网     | `https://testnet.zklighter.elliot.ai` | `wss://testnet.zklighter.elliot.ai/stream` | 300      |

在数据和执行配置中使用 `LighterEnvironment::Mainnet` 或
`LighterEnvironment::Testnet`。私有网关或本地测试夹具可使用 URL 覆盖。

## Integrator 归属标注

已提交的创建和修改订单交易,会在 Lighter 的 `L2TxAttributes` 中携带
NautilusTrader integrator 账户索引。这有助于我们评估该集成的实际使用情况,
并据此确定后续维护的优先级。Maker 和 taker 的 integrator 手续费均设为零,
因此该归属标注不会增加任何交易成本。

在这些属性能够附加到订单之前,Lighter 要求先获得 `ApproveIntegrator` 批准。
在启动期间,执行客户端会为配置的 L2 账户提交所需的**零手续费**批准。

### 撤销批准

在停用适配器时,可使用撤销操作进行清理。它发送相同的 `ApproveIntegrator`
交易,但将 `approval_expiry = 0` 且所有最大手续费均设为零;下一次执行客户端
启动时会记录一个新的零手续费批准。

```bash
export LIGHTER_API_KEY_INDEX=5
export LIGHTER_API_SECRET=REPLACE_ME
export LIGHTER_ACCOUNT_INDEX=123456
cargo run -p nautilus-lighter --bin lighter-integrator-revoke           # 主网
cargo run -p nautilus-lighter --bin lighter-integrator-revoke testnet   # 测试网
```

脚本源码:
[`crates/adapters/lighter/bin/integrator_revoke.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/lighter/bin/integrator_revoke.rs)。

```python
# Python(PyO3 绑定)- 读取与 Rust 程序相同的环境变量
from nautilus_trader.adapters.lighter import revoke_lighter_integrator
from nautilus_trader.adapters.lighter import LighterEnvironment

await revoke_lighter_integrator()                            # 主网(默认)
await revoke_lighter_integrator(LighterEnvironment.TESTNET)  # 测试网
```

Rust 脚本会打印该操作的摘要,并在签名或发送之前暂停,等待按下 Enter 键;
如果摘要中有任何内容看起来不对,请在此之前使用 `Ctrl+C` 中止。Python 绑定
不会提示确认:调用前请自行检查当前的环境变量。

## 数据订阅

| 数据类型            | 订阅         | 快照 | 历史 | Nautilus 类型       | 备注                                                    |
|----------------------|--------------|----------|-------|---------------------|----------------------------------------------------------|
| 标的元数据  | 缓存重放 | ✓        | -     | `InstrumentAny`     | 从 `orderBookDetails` 加载。                          |
| 成交 tick          | ✓            | -        | ✓     | `TradeTick`         | WebSocket 成交;公开的 `recentTrades` REST 历史记录。    |
| 报价 tick          | ✓            | -        | -     | `QuoteTick`         | 最优买卖价 ticker 流。                          |
| 订单簿增量    | ✓            | ✓        | -     | `OrderBookDeltas`   | 仅限 `L2_MBP`。                                           |
| 订单簿深度 10   | ✓            | -        | -     | `OrderBookDepth10`  | 来自维护中订单簿的实时前 10 档视图;无 REST 快照。 |
| 订单簿快照 | -            | ✓        | -     | `OrderBook`         | REST 快照,最大深度 250。                                |
| 标记价格          | ✓            | -        | -     | `MarkPriceUpdate`   | 永续市场统计数据流。                                |
| 指数价格         | ✓            | -        | -     | `IndexPriceUpdate`  | 市场与现货统计数据流。                           |
| 资金费率        | ✓            | -        | ✓     | `FundingRateUpdate` | 当前估计值以及 REST 每小时历史记录。                            |
| K 线                 | ✓            | -        | ✓     | `Bar`               | WebSocket K 线数据流;REST 历史用于回填。                           |
| 标的状态    | REST         | ✓        | -     | `InstrumentStatus`  | `active` / `inactive` 快照。                         |

订单簿增量与深度 10 订阅仅接受 `BookType::L2_MBP`。其他订单簿类型会在
订阅之前返回错误。

WebSocket 订单簿只从 `subscribed/order_book` 初始化。如果在该快照之前
到达 `update/order_book`,适配器会将其丢弃,并等待真正的快照,因为增量
更新不包含完整的可见订单簿。

深度 10 订阅使用与增量相同的 WebSocket `order_book` 流。适配器会在每次
接受快照或增量更新之后,发出一个刷新后的前 10 档视图。

K 线订阅使用该场所的 `candle/{market_id}/{resolution}` WebSocket 频道。
Lighter 每约 500 毫秒批量推送一次仍在进行中的 K 线更新;适配器只在
K 线起始时间戳前进时才发出 Nautilus `Bar`,因此消费者每个已收盘周期只会
看到一个事件。进行中的缓存会在重连和取消订阅时被清除。

该数据流支持 `1m`、`5m`、`15m`、`30m`、`1h`、`4h`、`12h` 和 `1d`。
`1w` 仅通过 `request_bars` 提供 REST 支持;订阅 `1-WEEK` K 线类型会返回错误。

REST K 线历史会省略开盘价、最高价、最低价或收盘价缺失、为 null、为零
或为负数的场所间隙行。这些行无法构成有效的 Nautilus K 线,但不会阻止
后续有效行的加载。

标的状态订阅会在有缓存时重放最新的 `orderBookDetails` 状态,否则会获取
一次 REST 快照。Lighter 未提供 WebSocket 状态变化数据流。

资金费率订阅使用 `market_stats.current_funding_rate`,这是 Lighter 对
即将到来的结算的估计值。历史资金费率请求使用 `1h` 精度的
`/api/v1/fundings`,并将已结算的行映射为 `interval=60` 的
`FundingRateUpdate`。REST 的 `direction` 字段控制符号:`long` 保持为正,
因为多头向空头支付,而 `short` 映射为负,因为空头向多头支付。适配器不会
将账户特定的 `positionFunding` 载荷用于公开资金费率历史。

成交订阅使用公开的 WebSocket 成交流。历史成交请求使用公开的
`/api/v1/recentTrades` 端点,无需凭证;适配器会将请求限制在场所单次调用
的上限内,并将返回的 tick 过滤到请求的时间范围。

### 不受支持的数据请求

`request_quotes` 未实现。Lighter 通过 WebSocket 的 `ticker` 数据流暴露
最优买卖价数据,但适配器可用的 REST 端点不提供带时间戳的报价快照或报价
历史,无法安全地映射为 `QuoteTick`。

`request_book_depth` 未实现。已记录的 REST 订单簿端点不提供
`OrderBookDepth10.ts_event` 所需的场所事件时间戳;请使用
`subscribe_book_depth10` 获取实时深度 10 数据流,或使用
`request_book_snapshot` 获取 REST `OrderBook` 快照。

## 订单能力

### 订单标识

Lighter 使用数字场所订单索引以及调用方提供的 `client_order_index`。
适配器从 Nautilus 的 `ClientOrderId` 派生出 Lighter 的
`client_order_index`,并维护一个本地映射,以便私有 WebSocket 报告能够
恢复原始的客户端订单 ID。由于数字客户端索引可能会在历史场所订单之间被
复用,适配器要求映射的场所订单 ID 匹配后才恢复 Nautilus 客户端订单 ID。
未知的历史索引会使用唯一的场所订单 ID 作为其外部客户端订单 ID。

查询路径使用数字场所订单 ID 来查询活跃或终态历史。在该 ID 已知之前,
Nautilus 客户端订单 ID 可以通过其派生的客户端索引来查询活跃订单。仅凭
客户端索引进行的查询不会搜索终态历史,并且重复的活跃匹配会因结果模糊
而失败。

### 订单类型

| 订单类型             | 永续 | 现货 | 备注                                                   |
|------------------------|------------|------|-----------------------------------------------------------|
| `MARKET`               | ✓          | ✓    | 上限基于缓存的对手方报价 + 滑点推导而来。      |
| `LIMIT`                | ✓          | ✓    | 需要限价。                                 |
| `STOP_MARKET`          | ✓          | -    | 仅限永续;上限基于 `trigger_price` + 滑点推导而来。 |
| `STOP_LIMIT`           | ✓          | -    | 仅限永续;映射为 Lighter 止损限价单。      |
| `MARKET_IF_TOUCHED`    | ✓          | -    | 仅限永续;上限基于 `trigger_price` + 滑点推导而来。 |
| `LIMIT_IF_TOUCHED`     | ✓          | -    | 仅限永续;映射为 Lighter 止盈限价单。    |
| `MARKET_TO_LIMIT`      | -          | -    | *不支持*。                                        |
| `TRAILING_STOP_MARKET` | -          | -    | *不支持*。                                        |
| `TRAILING_STOP_LIMIT`  | -          | -    | *不支持*。                                        |
| `TWAP`                 | -          | -    | *不支持*;没有对应的 Nautilus 映射。                   |

条件单类型仅适用于永续市场。现货条件单会在本地被拒绝,因为 Lighter 会在
场所层面拒绝它们。条件单类型必须包含 `trigger_price`。若缺少触发价,
`STOP_MARKET` 和 `MARKET_IF_TOUCHED` 会被提前拒绝;若触发价在标的的
价格精度下被截断为 `0` 个 tick,所有条件单类型都会被拒绝。

Lighter 的市价类订单在传输报文中需要一个最差可接受的 `price` 字段。
适配器会自动推导该值:`MARKET` 订单读取缓存的对手方 `QuoteTick`
(买入读取卖价,卖出读取买价);`STOP_MARKET` 和 `MARKET_IF_TOUCHED`
使用订单的 `trigger_price`。基准值会按 `market_order_slippage_bps`
(默认 50 基点 = 0.5%)进行放宽,并按标的价格精度保守取整(买入向上取整,
卖出向下取整)。如果在策略订阅报价之前提交 `MARKET` 订单,会以明确的
错误信息被拒绝。可通过 `SubmitOrder.params["market_order_slippage_bps"]`
按订单覆盖该值。

### 条件单

| 功能                         | 永续 | 现货 | 备注                                                  |
|---------------------------------|------------|------|----------------------------------------------------------|
| 止损市价单                | ✓          | -    | `STOP_MARKET` 映射为 Lighter 的 `STOP_LOSS`。             |
| 止损限价单                 | ✓          | -    | `STOP_LIMIT` 映射为 Lighter 的 `STOP_LOSS_LIMIT`。        |
| 止盈市价单              | ✓          | -    | `MARKET_IF_TOUCHED` 映射为 Lighter 的 `TAKE_PROFIT`。     |
| 止盈限价单               | ✓          | -    | `LIMIT_IF_TOUCHED` 映射为 `TAKE_PROFIT_LIMIT`。        |
| 触发价                   | ✓          | -    | 每种受支持的条件单都需要。        |
| 触发价类型              | -          | -    | *不支持*;没有触发来源选择器。                    |
| 分组订单列表             | -          | -    | *不支持*。                                       |
| OCO / OTO 订单                | -          | -    | *不支持*。                                       |
| 括号单                  | -          | -    | *不支持*。                                       |
| `CreateGroupedOrders`           | -          | -    | *不支持*;订单列表使用独立交易。      |

### 订单选项

| 选项           | 永续 | 现货 | 备注                                                                      |
|------------------|------------|------|------------------------------------------------------------------------------|
| `post_only`      | ✓          | ✓    | 映射为 Lighter 的仅挂单有效期。                                 |
| `reduce_only`    | ✓          | -    | 透传给 `CreateOrder`;仅用于减少现有仓位。  |
| `quote_quantity` | -          | -    | *不支持*;请改为提交基础货币数量。                             |
| `display_qty`    | -          | -    | *不支持*;Lighter 未暴露冰山显示数量字段。        |

### 适配器订单参数

| 参数                                      | 永续 | 现货 | 备注                                               |
|--------------------------------------------|------------|------|-----------------------------------------------------|
| `market_order_slippage_bps`                | ✓          | ✓    | 覆盖市价类订单上限的配置默认值。 |
| 通过 `SubmitOrder.params` 传入 `post_only`   | -          | -    | *不支持*;请使用 Nautilus 订单标志。       |
| 通过 `SubmitOrder.params` 传入 `reduce_only` | -          | -    | *不支持*;请使用 Nautilus 订单标志。       |

### 有效期

| 有效期  | 永续 | 现货 | 备注                                                                        |
|----------------|------------|------|------------------------------------------------------------------------------|
| `GTC`          | ✓          | ✓    | 限价类使用 `GoodTillTime`;市价类使用 `IOC`。                    |
| `DAY`          | ✓          | ✓    | 限价类和条件单使用一个正的订单到期时间。              |
| `GTD`          | ✓          | ✓    | 提供的到期时间必须在提交时起 5 分钟到 30 天之间。                |
| `IOC`          | ✓          | ✓    | 普通 `MARKET`/`LIMIT` 使用到期时间 `0`;条件限价单使用触发到期时间。 |
| `FOK`          | -          | -    | *不支持*。                                                            |
| `AT_THE_OPEN`  | -          | -    | *不支持*。                                                            |
| `AT_THE_CLOSE` | -          | -    | *不支持*。                                                            |

对于 `MARKET`、`STOP_MARKET` 和 `MARKET_IF_TOUCHED`,适配器会将报文中的
有效期映射为 Lighter 的 `ImmediateOrCancel`,因为场所会拒绝以
`GoodTillTime` 发送的市价类订单。普通 `MARKET` 订单设置
`OrderExpiry = 0`。条件市价单(`STOP_MARKET` 和 `MARKET_IF_TOUCHED`)
保留一个正的 `OrderExpiry`,使触发器能够挂起,只有在触发之后,报文中的
`ImmediateOrCancel` 才会生效。条件市价单无法表示 Nautilus 的 `IOC`,
因此适配器会在本地以明确错误拒绝它。条件限价单(`STOP_LIMIT` 和
`LIMIT_IF_TOUCHED`)可以使用 Nautilus 的 `IOC`:触发器以正的
`OrderExpiry` 挂起,而在触发之后,子限价单使用 Lighter 的
`ImmediateOrCancel`。

当没有提供显式的 GTD 到期时间时,限价类的 `GTC`、`DAY` 和 `GTD` 订单
默认使用当前时间加 28 天。条件单的 `GTC`、`DAY` 以及限价类 `IOC` 使用
相同的默认到期时间。对于这些有效期,场所会拒绝将 `-1` 作为有效的到期
时间。Lighter 要求订单到期时间戳距离提交时至少 5 分钟、最多 30 天,
因此超出该场所窗口的实时 GTD 订单会在本地被拒绝。该最小值检查包含
1 秒的签名与传输余量。

### 执行指令

| 指令   | 永续 | 现货 | 备注                                                        |
|---------------|------------|------|----------------------------------------------------------------|
| `post_only`   | ✓          | ✓    | 覆盖有效期,并发送 Lighter 的 `PostOnly`。              |
| `reduce_only` | ✓          | -    | 用于对现有衍生品仓位进行减仓的标志。    |

在限价类订单上使用 `post_only`。适配器不会合成仅挂单的市价单。实盘主网
测试确认,`reduce_only=true` 可用于平掉永续仓位。无效的 reduce-only 开仓
可能会被 Lighter 丢弃而不产生场所订单报告;适配器会将其对账为
`INFLIGHT_TIMEOUT`,而不是场所提供的拒绝原因。

### 高级订单功能

| 功能              | 永续 | 现货 | 备注                                                       |
|----------------------|------------|------|-------------------------------------------------------------|
| 订单修改   | ✓          | ✓    | 修改活跃订单的数量、价格和触发价格。  |
| 括号单       | -          | -    | *不支持*。                                            |
| 冰山订单       | -          | -    | *不支持*。                                            |
| 跟踪止损       | -          | -    | *不支持*。                                            |
| 挂钩订单        | -          | -    | *不支持*。                                            |
| TWAP 订单          | -          | -    | *不支持*;没有对应的 Nautilus 映射。                       |
| 杠杆更新      | ✓          | -    | 仅限永续;提交一笔已签名的 `UpdateLeverage` 交易。            |
| 原生全部取消    | -          | -    | *不支持*;适配器按标的限定全部取消的范围。  |
| 死人开关 (Dead man's switch)   | -          | -    | *不支持*。                                            |

### 订单操作

| 操作           | 永续 | 现货 | 备注                                                           |
|---------------------|------------|------|-------------------------------------------------------------------|
| 提交订单        | ✓          | ✓    | 通过 WebSocket 发送一笔已签名的 `L2CreateOrder` 交易。      |
| 提交订单列表   | ✓          | ✓    | 依次拆分为最多 15 笔独立创建交易。  |
| 修改订单        | ✓          | ✓    | 发送一笔已签名的 `ModifyOrder`;报告可能会重新声明接受状态。      |
| 取消订单        | ✓          | ✓    | 发送一笔已签名的 `L2CancelOrder` 交易。                     |
| 取消全部订单   | ✓          | ✓    | 遍历所请求标的的缓存未成交订单。       |
| 设置杠杆        | ✓          | -    | 仅限永续;提交一笔已签名的 `UpdateLeverage` 交易。                |
| 批量取消订单 | ✓          | ✓    | 依次拆分为最多 15 笔独立取消交易。  |
| 查询订单         | ✓          | ✓    | 需要凭证以及 REST 查询。                           |
| 查询账户       | ✓          | ✓    | 重放最新的私有 WebSocket 账户状态。             |
| 全量状态         | ✓          | ✓    | 限定在 WS 和 REST 报告中账户活跃的市场内。     |

场所原生的 `CancelAllOrders` 交易是账户级别的。适配器有意按标的取消已
缓存的未成交订单,以避免影响不相关的市场。

`SubmitOrderList` 和 `BatchCancelOrders` 会通过基于哈希关联的 WebSocket
`sendTx` 路径,按顺序对每笔子交易进行签名并发送。适配器只有在前一笔子
交易的发送完成后,才会分配下一个 nonce。因此每笔交易都会获得正常的
确认、拒绝和 nonce 恢复处理。这种拆分并非原子操作:它不会创建分组场所
订单,也不提供 OCO/OTO 或括号单语义。

`UpdateLeverage` 通过
`LighterExecutionClient::update_leverage(instrument_id, initial_margin_fraction, margin_mode)`
暴露。`initial_margin_fraction` 以场所 tick 为单位(1e-4 的比例):
`500` 表示 5% 初始保证金(20 倍杠杆),`1000` 表示 10%(10 倍杠杆),
以此类推。

`UpdateLeverage`、`CancelAllOrders`、带 integrator 属性的修改订单,以及
条件创建订单,都已与官方 Lighter v1.1.2 签名器进行了字节级别的一致性校验。

### 订单查询与对账

| 功能              | 永续 | 现货 | 备注                                                        |
|----------------------|------------|------|----------------------------------------------------------------|
| 查询未成交订单    | ✓          | ✓    | 按市场限定范围的 REST `accountActiveOrders`。                 |
| 查询订单历史  | ✓          | ✓    | 带游标分页的 REST `accountInactiveOrders`。         |
| 订单状态更新 | ✓          | ✓    | 私有 WebSocket 订单流以及状态报告。         |
| 成交历史        | ✓          | ✓    | REST `trades`;账户历史需要凭证。 |
| 成交报告         | ✓          | ✓    | REST 与私有 WebSocket 成交载荷。                   |
| 仓位报告     | ✓          | -    | 仅限永续;重放缓存的仓位流。            |
| 账户状态        | ✓          | ✓    | 重放缓存的合并账户状态快照。            |
| 全量状态          | ✓          | ✓    | 综合订单、成交与缓存仓位。                    |

已认证的非活跃订单与成交分页会拒绝重复的游标,并在 1,000 页后停止。
成交对账在多次调用间保持可重复性,同时会抑制已通过实时 WebSocket 流发出
的成交。历史订单与成交报告仅将映射的客户端索引绑定到与之匹配的场所订单
ID,因此被复用的数字索引不会将不相关的生命周期合并在一起。

## 账户与仓位管理

已认证的执行客户端会订阅以下私有数据流:

- `account_all_orders`:订单状态报告。
- `account_all_trades`:成交报告。
- `account_all_positions`:仓位快照。
- `account_all_assets`:按资产的余额快照(现货余额加永续抵押品)。
- `user_stats`:永续账户保证金汇总(抵押品和可用余额)。

适配器将 `account_all_assets` 和 `user_stats` 合并为单一的账户状态,
并仅在两个数据流都已推送首帧之后才发出该状态。

执行客户端在连接前需要凭证,因为私有账户流和 nonce 刷新是强制性的。
可以在没有凭证的情况下构造客户端,但除非 `private_key`、`account_index`
和 `api_key_index` 都能解析出来,否则实盘执行不会连接。

永续仓位以净额模式报告:每个市场一个仓位。现货余额通过账户资产状态
而非仓位报告传递。
每一帧 `account_all_positions` 都被视为一个场所快照。如果新的一帧
省略了先前已缓存的市场,适配器会为该标的发出一份平仓仓位报告;一个空的
`positions` 映射会清空所有已缓存的永续仓位,并为每个市场发出平仓报告。
若该帧中存在无法映射市场或无法解析仓位的行,则会按市场 ID 保留:已解析
的行仍会更新缓存,而既未出现在已解析行中、也未出现在被跳过市场 ID 中的
已缓存市场仍会被平仓。

| 功能                 | 永续 | 现货 | 备注                                                        |
|-------------------------|------------|------|----------------------------------------------------------------|
| 账户余额        | ✓          | ✓    | 合并资产 + `user_stats`,查询时从缓存重放。  |
| 仓位快照      | ✓          | -    | 仅限永续;`account_all_positions` 数据流。                   |
| 净额仓位       | ✓          | -    | 每个永续市场一个 Nautilus 仓位。                   |
| 全仓保证金            | ✓          | -    | 通过 `LighterPositionMarginMode::Cross` 透传。           |
| 逐仓保证金         | ✓          | -    | 通过 `LighterPositionMarginMode::Isolated` 透传。        |
| 杠杆更新        | ✓          | -    | 已签名的 `UpdateLeverage` 交易。                         |
| 现货保证金 / 借贷 | -          | -    | *不支持*。                                             |
| 存款 / 取款  | -          | -    | 请使用场所工具或交易适配器之外的 Lighter API。 |

## 强平与自动减仓(ADL)处理

| 事件或字段              | 是否支持 | 备注                                                             |
|-----------------------------|---------|---------------------------------------------------------------------|
| 强平成交          | ✓       | 账户成交行可解析为成交,没有特殊事件类型。     |
| 减仓(ADL)成交           | ✓       | 账户成交行可解析为成交,没有特殊事件类型。     |
| 强平价格报告 | -       | *不支持*;报告中省略该字段。                         |
| ADL 事件流            | -       | *不支持*。                                                  |

## 资金费率

永续的 `market_stats` 帧会发出 `MarkPriceUpdate`、`IndexPriceUpdate` 和
`FundingRateUpdate` 事件。现货的 `spot_market_stats` 帧会发出
`IndexPriceUpdate` 事件。

历史资金费率请求使用公开的 `/api/v1/fundings` 端点,针对已结算的每小时
行发出 `FundingRateUpdate` 响应。适配器会对请求范围进行跨页分页,因此
较宽的时间窗口不会被截断到单页;显式的 `limit` 仍会限制返回的行数。

## 账户等级

Lighter 为每个账户分配一个等级,用以决定延迟、速率限制和手续费。
Standard 是零手续费的默认等级;更高等级需要在场所主动选择加入,并以
手续费换取更低的延迟和更高的吞吐量。执行客户端在连接时(通过
`GET /api/v1/account`)检测等级,并以蓝色日志记录,对于该适配器尚未
识别的等级也会记录原始的 `account_type` 代码。这一检测仅具备信息性质:
适配器绝不会自行提高速率限制,因为更高的场所限制需要向 Lighter 注册
调用方 IP,因此更高的等级本身并不能保证更高的限制已对你的连接生效。

| 等级     | 延迟(maker / taker) | REST 加权限额 | `sendTx` 限额       | 手续费(maker / taker)      | 备注                                   |
|----------|-------------------------|---------------------|----------------------|---------------------------|------------------------------------------|
| Standard | 200 ms / 300 ms         | 60 请求/分钟          | 60 请求/分钟           | 0 / 0                     | 默认的零手续费等级。                  |
| Premium  | 0 ms / 140-200 ms       | 24,000 加权请求/分钟      | 4,000-40,000 请求/分钟 | 0.28-0.40 / 1.96-2.80 基点 | 最低延迟;随质押的 LIT 数量提升。 |
| Plus     | 200 ms / 300 ms         | 120,000 加权请求/分钟     | 8,000 请求/分钟        | 0.5 / 0.5 基点             | 提高限额,延迟保持标准水平。        |
| Builder  | -                       | 240,000 加权请求/分钟     | -                     | -                         | 最高的 REST 吞吐量。                |

Premium 等级的延迟、手续费和 `sendTx` 吞吐量会随质押的 LIT 数量而变化,
具体规则也可能调整;请参见 Lighter 文档获取当前数据。若要实际使用更高
等级的限额,需向 Lighter 注册调用方 IP,并显式设置配额(参见
[速率限制](#速率限制))。

## 速率限制

Lighter 对 IP 地址和 L1 地址都施加了速率限制。执行客户端在连接时检测
账户等级并记录日志(参见 [账户等级](#账户等级)),但不会自动提高限额。
默认情况下,两个客户端都使用较为保守的标准账户配额。若要使用更高等级的
吞吐量,需向 Lighter 注册调用方 IP,并在客户端配置中显式设置配额:

- `rest_quota_per_min`:REST 读取桶的配额,单位为每分钟请求数。未设置时
  保持 60 请求/分钟。数据客户端和执行客户端均可用。
- `sendtx_quota_per_min`:交易配额,单位为每分钟请求数,计量在与读取
  独立的单独桶中。未设置时保持标准的 60 请求/分钟,与
  `rest_quota_per_min` 无关。仅限执行客户端。

场所文档中标注了更高等级的 REST 配额权重。适配器的 REST 限流器不对
按端点的权重建模;它通过一个共享的 REST 桶和一个路由桶,为每次 HTTP
调用消耗一个令牌。当将 `rest_quota_per_min` 设置得高于标准默认值时,
应使用针对你计划调用的端点组合的有效请求速率,而非原始的加权配额。
例如,Lighter 将 `/api/v1/trades` 和 `/api/v1/recentTrades` 的权重
记录为 600,因此 24,000 加权请求/分钟的 premium 限额相当于每分钟
40 次这类调用。

该场所会将同一账户在两种传输方式下的交易计入同一个桶中。执行客户端使用
一个跨 WebSocket `sendTx`(包括订单列表和取消拆分)以及用于启动时
integrator 批准的 HTTP `sendTx` 的共享限流器来强制执行
`sendtx_quota_per_min`。直接调用公开的底层 HTTP `sendTxBatch` API 时,
也会使用同一个限流器。

数据客户端和执行客户端在每个场所 URL 上共享一个 WebSocket 消息限流器,
因此它们的合计发送速率会遵守场所的单 IP 上限。它会将订阅、取消订阅和
重新订阅等非交易控制帧,按场所文档记录的 200 条消息/分钟的速率进行
节流。订阅分发还受到一个闭环飞行中门限的额外限制:一旦未确认数量达到
其上限(35),数据流处理器会暂停排队的订阅请求,使其保持在 Lighter
每 IP 50 条消息的飞行中上限之下,同时确保启动和重连时的订阅能够顺利
展开。仅靠速率上限无法约束这一点,因为未确认数量跟踪的是场所确认延迟,
而非发送速率。`sendTx` 不计入 WebSocket 客户端消息桶。

| 范围                                  | 场所限制                 | 适配器行为                                     |
|----------------------------------------|-----------------------------|--------------------------------------------------------|
| REST,标准账户                 | 60 请求/分钟                  | 默认值;设置 `rest_quota_per_min` 以覆盖。       |
| REST,premium 账户                  | 24,000 加权请求/分钟     | 记录日志;设置 `rest_quota_per_min` 以使用。          |
| REST,plus 账户                     | 120,000 加权请求/分钟    | 记录日志;设置 `rest_quota_per_min` 以使用。          |
| REST,builder 账户                  | 240,000 加权请求/分钟    | 记录日志;设置 `rest_quota_per_min` 以使用。          |
| `sendTx` / `sendTxBatch`,standard     | 60 请求/分钟                  | 执行下单使用 WebSocket `sendTx`。             |
| `sendTx` / `sendTxBatch`,plus         | 8,000 请求/分钟                  | 设置 `sendtx_quota_per_min` 以使用。                   |
| `sendTx` / `sendTxBatch`,premium      | 4,000-40,000 请求/分钟        | 设置 `sendtx_quota_per_min`(随质押的 LIT 数量提升)。 |
| 默认交易类型限制         | 40 请求/分钟                  | 适用于未被交易量配额覆盖的交易类型。     |
| `L2UpdateLeverage` 交易限制   | 40 请求/分钟                  | 与 `update_leverage` 相关。                       |
| 挂单数量                         | 500/账户,16/市场      | 场所限制;适配器不会预先计数。          |
| 活跃订单数量                          | 1,500/账户,1,000/市场 | 场所限制;适配器不会预先计数。          |

来自官方文档的常见 REST 端点权重:

| 端点组                         | 权重 | 适配器行为                                |
|----------------------------------------|--------|---------------------------------------------------|
| `sendTx`、`sendTxBatch`、`nextNonce`   | 6      | 交易调用使用交易限流器;`nextNonce` 使用 REST。 |
| `accountInactiveOrders`                | 100    | 适配器为每次 HTTP 调用计入一个 REST 令牌。    |
| `trades`、`recentTrades`               | 600    | 适配器为每次 HTTP 调用计入一个 REST 令牌。    |
| 其他端点                        | 300    | 适配器为每次 HTTP 调用计入一个 REST 令牌。    |

| 端点或传输方式                  | 限制      | 备注                                              |
|----------------------------------------|------------|------------------------------------------------------|
| `/api/v1/trades`                       | 100 行   | 适配器在此上限下对账时进行分页。      |
| `/api/v1/accountInactiveOrders`        | 100 行   | 适配器在此上限下遵循 `next_cursor`。         |
| `/api/v1/orderBookOrders`              | 250 档 | 快照深度会被限制在场所上限之内。        |
| `/api/v1/candles`                      | 500 行   | 适配器将 REST K 线分页限制在此场所最大值。 |
| `/api/v1/fundings`                     | 100 行   | 适配器在此场所上限下对资金费率进行分页。 |
| WebSocket 连接                  | 255 / IP   | 场所限制。                                       |
| WebSocket 订阅数 / 连接   | 500        | 场所限制。                                       |
| WebSocket 唯一账户数 / 连接 | 500        | 场所限制。                                       |
| WebSocket 连接数 / 分钟         | 255        | 场所限制。                                       |
| WebSocket 客户端消息数 / 分钟     | 200        | 适配器在此上限下节流非交易控制帧。   |
| WebSocket 飞行中消息数            | 50         | 场所上限;订阅使用一个 35 帧的闭环。 |
| `sendTxBatch` 批量大小               | 15 笔交易     | 底层 API 限制;拆分上限同样为 15。        |
| WebSocket 保活                    | 2 分钟  | 适配器每 30 秒发送一次心跳。         |
| WebSocket 出站命令队列       | 无上限 | 在写入前进行节流;没有队列深度上限。   |

Premium 交易量配额是针对 `L2CreateOrder`、`L2CancelAllOrders`、
`L2ModifyOrder` 和 `L2CreateGroupedOrders` 的一个独立场所限制。适配器
不会检查剩余配额;若策略依赖 premium 或 plus 等级的限额,请使用场所
账户工具。当前的配额规则和补充计算方式,请参见 Lighter 的
[交易量配额](https://apidocs.lighter.xyz/docs/volume-quota-program)文档。

## 交易量配额与无成交挂单

Lighter 的交易量配额对报价刷新型策略的影响与传输层速率限制不同。
创建、修改、原生全部取消,以及分组订单交易都会消耗交易量配额,而场所会
根据已完成的交易量以及该计划提供的任何免费额度来补充配额。一个反复在
中间价附近报价但没有成交的做市冒烟测试,即使 WebSocket 和 `sendTx`
速率限流器工作正常,也可能耗尽配额。

对于 Lighter 上的实盘测试,不要将无限期的无成交报价当作一种中性的
连接性检查。建议采用较慢的单边报价、更宽的刷新阈值、测试网覆盖,或者
一个有意获取一定量小额成交以补充所消耗配额的有界策略。

## 连接管理

WebSocket 客户端每 30 秒发送一次心跳,并以从 250 毫秒到 30 秒的指数
退避方式重连。私有账户订阅使用 Lighter 认证令牌,最大存活时间为 8 小时。
适配器会铸造有效期为 7 小时的令牌,每 6 小时轮换一次,并重新订阅账户
频道。透明重连也会通知轮换任务铸造新令牌,并在 WebSocket 客户端开始
重放已跟踪的订阅之后重新订阅。

在执行客户端重连时,适配器会通过 `GET /api/v1/nextNonce` 启动一次
nonce 基线刷新。在该 HTTP 刷新进行期间,新签名交易的发送不会被阻塞。

在一个会话内,适配器在本地管理交易 nonce:场所确认会推进分配窗口,
明确的拒绝或发送前失败可能会回滚最新的 nonce,而陈旧状态会触发从
`GET /api/v1/nextNonce` 的重新同步。可能已到达场所的结果会保留其待定
的 nonce 和订单身份,以便通过 WebSocket 或对账进行恢复。

`LighterExecutionClient::connect()` 在返回之前,会等待每个账户流
(`account_all_orders`、`account_all_trades`、`account_all_positions`、
`account_all_assets`、`user_stats`)最多 30 秒以推送其首帧。Lighter 没有
用于账户或仓位状态的 REST 端点,因此 WebSocket 帧是唯一的真实来源:
更早返回会让策略与场所的初始状态产生竞态,并发现场所订单 ID 查询表或
仓位缓存为空。该门限会在每次连接尝试开始时清除任何上一会话的仓位和
账户缓存,以便重连周期观察到的是新会话的帧,而非陈旧数据。
相比之下,透明的 WebSocket 重连不会重新进入 `connect()`:它们会保留
缓存的仓位,直到下一帧 `account_all_positions` 到达,然后应用相同的
仓位快照替换规则。

## API 凭证

Lighter 的签名需要以下全部三个凭证值:

- 账户索引:数字形式的 Lighter 账户标识符。
- API key 索引:数字形式的 API key 槽位。请使用 Lighter 分配的用户
  创建的 key 索引,避免使用保留的低位索引,且不要使用 `255`;它是
  `apikeys` 查询的哨兵值,而非签名密钥。
- API 私钥:40 字节的十六进制私钥,可带或不带 `0x` 前缀。

配置值优先。若某个配置字段缺失,或 API 私钥为空(空字符串或仅含空白),
会回退到所选环境对应的环境变量。

| 环境 | API key 索引                   | API 私钥              | 账户索引                   |
|-------------|---------------------------------|-------------------------------|---------------------------------|
| 主网     | `LIGHTER_API_KEY_INDEX`         | `LIGHTER_API_SECRET`         | `LIGHTER_ACCOUNT_INDEX`         |
| 测试网     | `LIGHTER_TESTNET_API_KEY_INDEX` | `LIGHTER_TESTNET_API_SECRET` | `LIGHTER_TESTNET_ACCOUNT_INDEX` |

执行客户端会拒绝不完整的凭证。数据客户端可以在没有凭证的情况下运行:
它的订阅和 REST 请求(标的、订单簿、成交、K 线、资金费率)全部使用
公开端点。

## 配置

### 数据客户端配置选项

| 选项                             | 默认值   | 说明                                         |
|------------------------------------|-----------|-----------------------------------------------------|
| `base_url_http`                    | `None`    | 可选的 REST URL 覆盖值。                         |
| `base_url_ws`                      | `None`    | 可选的 WebSocket URL 覆盖值。                    |
| `proxy_url`                        | `None`    | HTTP 与 WebSocket 的可选代理 URL。          |
| `environment`                      | `Mainnet` | `LighterEnvironment::Mainnet` 或 `Testnet`。         |
| `account_index`                    | `None`    | 可选的工厂字段;公开数据调用不使用它。 |
| `api_key_index`                    | `None`    | 可选的工厂字段;公开数据调用不使用它。 |
| `private_key`                      | `None`    | 可选的工厂字段;公开数据调用不使用它。 |
| `http_timeout_secs`                | `60`      | HTTP 请求超时时间(秒)。                    |
| `ws_timeout_secs`                  | `30`      | WebSocket 连接与重连超时时间。      |
| `update_instruments_interval_mins` | `60`      | 标的元数据刷新间隔(分钟)。    |
| `rest_quota_per_min`               | `None`    | REST 配额覆盖值;未设置时保持 60 请求/分钟。        |
| `transport_backend`                | 默认   | WebSocket 传输后端。                        |

### 执行客户端配置选项

| 选项                      | 默认值       | 说明                                                |
|-----------------------------|---------------|------------------------------------------------------------|
| `trader_id`                 | `TRADER-001`  | Nautilus trader 标识符。                                |
| `account_id`                | `LIGHTER-001` | 该场所对应的 Nautilus 账户标识符。                 |
| `account_index`             | `None`        | Lighter 账户索引。                                     |
| `api_key_index`             | `None`        | Lighter API key 槽位。                                      |
| `private_key`               | `None`        | 用于认证和 L2 交易签名的十六进制私钥。       |
| `base_url_http`             | `None`        | 可选的 REST URL 覆盖值。                                 |
| `base_url_ws`               | `None`        | 可选的 WebSocket URL 覆盖值。                           |
| `proxy_url`                 | `None`        | HTTP 与 WebSocket 的可选代理 URL。                 |
| `environment`               | `Mainnet`     | `LighterEnvironment::Mainnet` 或 `Testnet`。                |
| `http_timeout_secs`         | `60`          | HTTP 请求超时时间(秒)。                           |
| `ws_timeout_secs`           | `30`          | WebSocket 连接与重连超时时间。             |
| `market_order_slippage_bps` | `50`          | `MARKET` / `STOP_MARKET` / `MIT` 的滑点上限(基点)。   |
| `rest_quota_per_min`        | `None`        | REST 配额覆盖值;未设置时保持 60 请求/分钟。               |
| `sendtx_quota_per_min`      | `None`        | 交易配额覆盖值;未设置时保持 60 请求/分钟。        |
| `transport_backend`         | 默认       | WebSocket 传输后端。                               |

### 配置示例

```rust
use nautilus_lighter::{
    common::enums::LighterEnvironment,
    config::{LighterDataClientConfig, LighterExecClientConfig},
};

let data_config = LighterDataClientConfig::builder()
    .environment(LighterEnvironment::Testnet)
    .build();

let exec_config = LighterExecClientConfig::builder()
    .trader_id(trader_id)
    .account_id(account_id)
    .environment(LighterEnvironment::Testnet)
    .build();
```

以上执行配置会从匹配的测试网环境变量中解析凭证。可直接设置
`account_index`、`api_key_index` 和 `private_key` 来覆盖环境变量查找。
使用 `LiveExecEngineConfig.reconciliation_instrument_ids` 将对账范围
限定到特定的 Nautilus 标的。对于历史较长的账户,还应在
`LiveExecEngineConfig` 上设置 `reconciliation_lookback_mins`,以将非活跃
订单和成交的重放限制在策略所需的时间窗口内。

## 官方文档

- 快速入门:<https://apidocs.lighter.xyz/docs/get-started>
- 交易与签名:<https://apidocs.lighter.xyz/docs/trading>
- API keys:<https://apidocs.lighter.xyz/docs/api-keys>
- 速率限制:<https://apidocs.lighter.xyz/docs/rate-limits>
- 交易量配额:<https://apidocs.lighter.xyz/docs/volume-quota-program>
- 数据结构、常量与错误:<https://apidocs.lighter.xyz/docs/data-structures-constants-and-errors>
- REST OpenAPI:<https://raw.githubusercontent.com/elliottech/lighter-python/main/openapi.json>
- WebSocket 参考:<https://apidocs.lighter.xyz/docs/websocket-reference>

## 贡献

:::info
如需了解更多功能或为 Lighter 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
