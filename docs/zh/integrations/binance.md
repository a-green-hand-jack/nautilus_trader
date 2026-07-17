# Binance

Binance 成立于 2017 年,按日交易量以及加密资产和加密衍生品的未平仓
合约计,是最大的加密货币交易所之一。

NautilusTrader 同时提供 Python 和 Rust 两种语言的 Binance 集成。
Rust 适配器支持下方列出的所有产品类型,并包含额外功能(会在文中
标注)。Python 适配器支持相同的产品类型。

支持的产品:

- **Binance Spot**(包括 Binance US)
- **Binance USDT 保证金合约**(永续合约与交割合约)
- **Binance 币本位合约**(永续合约与交割合约)

## 示例

- [Python 实时示例](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/binance/)
- [Rust 现货示例](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/adapters/binance/examples/spot/)
- [Rust 合约示例](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/adapters/binance/examples/futures/)

## 概览

Binance 适配器包含多个组件,可以组合使用,也可以单独使用:

- `BinanceHttpClient`:底层 HTTP API 连接。
- `BinanceWebSocketClient`:底层 WebSocket API 连接。
- `BinanceInstrumentProvider`:标的解析与加载。
- `BinanceSpotDataClient` / `BinanceFuturesDataClient`:市场数据流
  管理器。
- `BinanceSpotExecutionClient` / `BinanceFuturesExecutionClient`:
  账户管理与交易执行网关。
- `BinanceLiveDataClientFactory`:Binance 数据客户端工厂(供交易
  节点构建器使用)。
- `BinanceLiveExecClientFactory`:Binance 执行客户端工厂(供交易
  节点构建器使用)。

:::note
大多数用户只需配置一个实时交易节点(如下所示),不必直接与这些底层
组件打交道。
:::

### 产品支持

| 产品类型                            | 是否支持 | 备注                              |
|-----------------------------------------|-----------|-------------------------------------|
| 现货市场(含 Binance US)         | ✓         |                                    |
| 保证金账户(全仓与逐仓)      | -         | *尚未实现。* 计划在 v2 中提供。 |
| USDT 保证金合约(永续 & 交割) | ✓         |                                    |
| 币本位合约                   | ✓         |                                    |

:::note
保证金账户功能(借币、还币、逐仓保证金管理)尚未实现。Python 适配器
不会添加保证金支持。完整的保证金交易支持计划在 v2 中提供。
:::

:::info
每个 Binance 客户端实例只处理一种产品类型。Rust 配置使用单数形式的
`product_type` 字段,实时工厂从一份配置创建一个数据或执行客户端。
要在同一节点中同时运行现货和合约,请为不同产品配置具有不同 ID 的
独立客户端,例如 `BINANCE_SPOT` 和 `BINANCE_FUTURES`,然后在策略
订阅或下单时传入对应的 `client_id`。Python 适配器使用不同的配置字段
名称,但 `examples/live/binance/binance_spot_and_futures_market_maker.py`
展示了相同的多客户端 ID 路由模式。
:::

## 数据类型

该集成包含若干自定义数据类型:

- `BinanceFuturesTicker`:合约 24 小时 ticker 数据,包含价格和统计
  信息。
- `BinanceBar`:带附加成交量指标的 K 线数据,供历史和实时使用。
- `BinanceFuturesMarkPriceUpdate`:Binance 合约的标记价格更新。
- `BinanceFuturesLiquidation`:来自 `forceOrder` 数据流的合约强平
  事件。

完整定义参见 Binance [API 参考](/docs/python-api-latest/adapters/binance.html)。

## 符号规则

对现货和合约,尽可能使用 Binance 原生符号。由于 NautilusTrader 支持
多场所交易,它必须区分作为现货交易对的 `BTCUSDT` 和作为永续期货合约
的 `BTCUSDT`(Binance 对两者使用相同的符号)。

Nautilus 会为所有永续符号附加 `-PERP` 后缀。例如,Binance Futures 的
`BTCUSDT` 永续合约在 Nautilus 中变为 `BTCUSDT-PERP`。

## 订单能力

以下表格详细列出了 Binance 各账户类型支持的订单类型、执行指令和
有效期选项。

### 订单类型

| 订单类型             | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                              |
|------------------------|------|--------|--------------|--------------|------------------------------------|
| `MARKET`               | ✓    | -      | ✓            | ✓            | 支持以计价货币计量数量:仅限现货。 |
| `LIMIT`                | ✓    | -      | ✓            | ✓            |                         |
| `STOP_MARKET`          | -    | -      | ✓            | ✓            | 仅限合约。           |
| `STOP_LIMIT`           | ✓    | -      | ✓            | ✓            |                         |
| `MARKET_IF_TOUCHED`    | -    | -      | ✓            | ✓            | 仅限合约。           |
| `LIMIT_IF_TOUCHED`     | ✓    | -      | ✓            | ✓            |                         |
| `TRAILING_STOP_MARKET` | -    | -      | ✓            | ✓            | 仅限合约。           |

### 执行指令

| 指令   | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                 |
|---------------|------|--------|--------------|--------------|---------------------------------------|
| `post_only`   | ✓    | -      | ✓            | ✓            | 见下方限制。               |
| `reduce_only` | -    | -      | ✓            | ✓            | 仅限合约;在对冲模式下禁用。 |

#### Post-only 限制

只有*限价*类订单类型支持 `post_only`。

| 订单类型               | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                               |
|--------------------------|------|--------|--------------|--------------|-----------------------------------------------------|
| `LIMIT`                  | ✓    | -      | ✓            | ✓            | 现货使用 `LIMIT_MAKER`,合约使用 `GTX` 有效期。 |
| `STOP_LIMIT`             | -    | -      | ✓            | ✓            | 仅限合约。                                       |

### 有效期

| 有效期 | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                      |
|---------------|------|--------|--------------|--------------|--------------------------------------------|
| `GTC`         | ✓    | -      | ✓            | ✓            | 撤销前有效。                        |
| `GTD`         | ✓*   | -      | ✓            | ✓            | *现货会转换为 GTC,并伴随警告。   |
| `FOK`         | ✓    | -      | ✓            | ✓            | 全部成交或取消。                              |
| `IOC`         | ✓    | -      | ✓            | ✓            | 立即成交或取消。                       |

### 高级订单功能

| 功能            | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                        |
|--------------------|------|--------|--------------|--------------|----------------------------------------------|
| 订单修改 | ✓    | -      | ✓            | ✓            | 仅限 `LIMIT` 订单的价格和数量修改。  |
| OCO 订单         | ✓    | -      | -            | -            | 现货 OCO 通过 `orderList/oco` 提交。      |
| 括号单     | -    | -      | -            | -            | *计划中*。目前会在提交时被拒绝。   |
| 冰山订单     | ✓    | -      | ✓            | ✓            | 大额订单拆分为可见部分。    |

### 批量操作

| 操作          | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                        |
|--------------------|------|--------|--------------|--------------|----------------------------------------------|
| 批量提交       | ✓    | -      | ✓            | ✓            | 逐笔提交订单(无批量 API 调用)。 |
| 批量修改       | -    | -      | -            | -            | 未实现。                             |
| 批量取消       | -*   | -      | ✓            | ✓            | *现货回退为逐笔取消。      |

#### 全部取消行为

当策略调用 `cancel_all_orders()` 时,适配器会包含未成交和飞行中
(SUBMITTED)状态的订单,以便适配器也能取消 Binance 尚未确认的订单。

**多策略安全性**:当多个策略交易同一标的时,适配器会将请求策略拥有的
订单与该标的的所有订单进行比较。如果该策略拥有全部订单,则使用单次
的全部取消 API 调用。否则,会发送按策略的取消请求(常规订单批量
取消,算法单逐笔取消),以避免影响其他策略。

**合约算法单**:条件单类型(`STOP_MARKET`、`STOP_LIMIT`、
`TAKE_PROFIT`、`TAKE_PROFIT_MARKET`、`TRAILING_STOP_MARKET`)需要
不同的取消端点。适配器会自动将它们路由到正确的端点。一旦某个算法单
触发并变为常规订单,它就会使用标准的取消端点。

**使用的端点**:

| 账户类型 | 常规订单                  | 算法单(批量)              | 算法单(逐笔)    |
|--------------|---------------------------------|-----------------------------------|------------------------------|
| 现货/保证金  | `DELETE /api/v3/openOrders`     | 不适用                              | 不适用                         |
| USDT 合约 | `DELETE /fapi/v1/allOpenOrders` | `DELETE /fapi/v1/algoOpenOrders` | `DELETE /fapi/v1/algoOrder` |
| 币本位合约 | `DELETE /dapi/v1/allOpenOrders` | `DELETE /dapi/v1/algoOpenOrders` | `DELETE /dapi/v1/algoOrder` |

### 仓位管理

| 功能             | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                       |
|---------------------|------|--------|--------------|--------------|---------------------------------------------|
| 查询仓位     | -    | -      | ✓            | ✓            | 实时仓位更新。                 |
| 仓位模式       | -    | -      | ✓            | ✓            | 单向模式与对冲模式(仓位 ID)。       |
| 杠杆控制    | -    | -      | ✓            | ✓            | 按符号动态调整杠杆。     |
| 保证金模式         | -    | -      | ✓            | ✓            | 按符号选择全仓或逐仓保证金。        |

### 风险事件

| 功能              | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                       |
|----------------------|------|--------|--------------|--------------|---------------------------------------------|
| 强平处理 | -    | -      | ✓            | ✓            | 交易所强制平仓。          |
| ADL 处理         | -    | -      | ✓            | ✓            | 自动减仓事件。                   |

Binance Futures 可能因风险事件触发交易所生成的订单:

- **强平**:当保证金不足以维持仓位时,Binance 会以破产价格强制平仓。
  这些订单的客户端 ID 以 `autoclose-` 开头。
- **ADL(自动减仓)**:当保险基金耗尽时,Binance 会平掉盈利仓位以
  覆盖损失。这些订单使用客户端 ID 前缀 `adl_autoclose`。
- **结算(USDT-M)**:资金/保证金结算订单的客户端 ID 以
  `settlement_autoclose-` 开头。
- **交割(COIN-M)**:到期的交割合约会自动平仓,客户端 ID 以
  `delivery_autoclose-` 开头。
- **保险基金**:保险基金接管使用状态 `NEW_INSURANCE`(在公开
  changelog 中已弃用,但在传输层仍能观察到)。

适配器通过客户端 ID 模式检测这些特殊订单类型(在检查执行类型之前
进行),然后:

1. 记录带订单详情的警告日志以供监控。
2. 生成带正确成交详情和 TAKER 流动性方向的 `FillReport`。
3. 生成用于对账的 `OrderStatusReport`。

上游参考资料:

- [USDT-M `ORDER_TRADE_UPDATE`](https://developers.binance.com/docs/derivatives/usds-margined-futures/user-data-streams/Event-Order-Update)
- [COIN-M `ORDER_TRADE_UPDATE`](https://developers.binance.com/docs/derivatives/coin-margined-futures/user-data-streams/Event-Order-Update)

当订单尚未在缓存中时,执行引擎会从运行时状态报告中创建外部订单。
这涵盖了首次出现的交易所生成订单(实盘强平或 ADL 事件的典型场景)。
引擎会将该订单分配给任何通过 `external_order_claims` 声明了该标的
的策略,或默认分配给 `EXTERNAL` 策略。

#### 手续费估算

当 Binance 在成交事件中省略手续费字段(`N`/`n`)时,Rust 适配器会
使用计价货币,按 `default_taker_fee * qty * price` 估算手续费。
这仅适用于 USD-M 正向合约。COIN-M 反向合约会回退使用零手续费,
因为正向公式没有考虑合约规模。请在 `BinanceExecClientConfig` 上
配置 `default_taker_fee` 以匹配你的手续费等级(默认:0.0004 /
0.04%)。

#### 对冲模式仓位 ID

启用 `use_position_ids`(默认)时,交易所生成的成交报告会包含一个
从标的和仓位方向推导出的 `venue_position_id`(例如
`ETHUSDT-PERP.BINANCE-LONG`)。对于 Binance 的双向仓位,请保持
此选项启用。仅当使用 `OmsType.HEDGING` 的虚拟仓位、由引擎管理仓位
身份时,才将 `use_position_ids` 设为 false。

对于使用 Rust 适配器、处于双向仓位模式的合约账户,请设置
`oms_type=OmsType::Hedging`。其 Python 绑定使用 `OmsType.HEDGING`。
Rust 适配器默认使用 `OmsType::Netting` 处理单向仓位模式。请保持
`use_position_ids` 启用,以跟踪 Binance 独立的多空两侧。

:::note
状态报告和成交报告会作为单一的 `OrderWithFills` 执行报告捆绑发出。
引擎会从状态报告中创建外部订单,然后应用真实成交,保留场所的
`trade_id` 和 `commission`。任何未被捆绑成交覆盖的剩余数量,会以
状态报告的 `avg_px` 推断出的成交进行平仓。
:::

### 订单查询

| 功能             | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                       |
|---------------------|------|--------|--------------|--------------|---------------------------------------------|
| 查询未成交订单   | ✓    | ✓      | ✓            | ✓            | 列出所有活跃订单。                     |
| 查询订单历史 | ✓    | ✓      | ✓            | ✓            | 历史订单数据。                      |
| 订单状态更新| ✓    | ✓      | ✓            | ✓            | 实时订单状态变化。              |
| 成交历史       | ✓    | ✓      | ✓            | ✓            | 执行与成交报告。                 |

### 条件单

| 功能             | 现货 | 保证金 | USDT 合约 | 币本位合约 | 备注                                        |
|---------------------|------|--------|--------------|--------------|------------------------------------------------|
| 订单列表         | ✓    | -      | ✓            | ✓            | 现货 OCO 列表;合约独立批量。 |
| OCO 订单          | ✓    | -      | -            | -            | 仅限现货,通过 `orderList/oco`。              |
| 括号单      | -    | -      | -            | -            | *计划中*。目前会在提交时被拒绝。   |
| 条件单  | ✓    | ✓      | ✓            | ✓            | 止损单和触及市价单。           |

### 订单参数

在调用 `Strategy.submit_order`(Python)或在 `SubmitOrder` 命令上
设置 `Params`(Rust)时,通过传入 `params` 字典来自定义单个订单。
Binance 执行客户端识别以下参数:

| 参数        | 类型   | 账户类型     | 说明 |
|------------------|--------|-------------------|-------------|
| `price_match`    | `str`  | USDT/币本位合约 | 设置 Binance 的某个 `priceMatch` 模式(见下方价格匹配一节),将价格选择委托给交易所。不能与 `post_only` 或冰山(`display_qty`)指令组合使用。 |
| `close_position` | `bool` | USDT/币本位合约 | 触发时平掉整个仓位(见下方平仓一节)。仅对 `StopMarket` 和 `MarketIfTouched` 订单有效。不能与 `reduce_only` 组合使用。 |

### 价格匹配

Binance Futures 通过 `priceMatch` 参数支持 BBO(最优买卖价)价格
匹配,将价格选择委托给交易所。限价单会动态加入订单簿的最优价格,
而无需指定确切的价格档位。

使用 `price_match` 时,你提交一个带参考价格(用于本地风险检查)的
限价单,Binance 会根据当前市场状态和价格匹配模式确定实际的挂单价格。

#### 有效的价格匹配取值

| 取值         | 行为                                                       |
|---------------|------------------------------------------------------------------|
| `OPPONENT`    | 加入订单簿对手方一侧的最优价格。          |
| `OPPONENT_5`  | 加入对手方一侧价格,但允许最多 5 个 tick 的偏移。  |
| `OPPONENT_10` | 加入对手方一侧价格,但允许最多 10 个 tick 的偏移。 |
| `OPPONENT_20` | 加入对手方一侧价格,但允许最多 20 个 tick 的偏移。 |
| `QUEUE`       | 加入同一侧的最优价格(保持挂单方)。             |
| `QUEUE_5`     | 加入同一侧队列,但偏移最多 5 个 tick。             |
| `QUEUE_10`    | 加入同一侧队列,但偏移最多 10 个 tick。            |
| `QUEUE_20`    | 加入同一侧队列,但偏移最多 20 个 tick。            |

:::info
更多详情参见[官方文档](https://developers.binance.com/docs/derivatives/usds-margined-futures/trade/rest-api)。
:::

#### 事件顺序

当一个订单以 `price_match` 提交时:

1. Nautilus 向 Binance 发送该订单,携带 `priceMatch` 参数,但在
   API 请求中省略限价。
2. Binance 接受该订单,并确定实际的挂单价格。
3. Nautilus 生成一个 `OrderAccepted` 事件。
4. 如果 Binance 接受的价格与参考价格不同,Nautilus 会生成一个带
   实际挂单价格的 `OrderUpdated` 事件。
5. 此时 Nautilus 缓存中的订单价格与 Binance 接受的价格一致。

#### 示例

```python
order = strategy.order_factory.limit(
    instrument_id=InstrumentId.from_str("BTCUSDT-PERP.BINANCE"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(1),
    price=Price.from_str("65000"),  # 用于本地风险检查的参考价格
)

strategy.submit_order(
    order,
    params={"price_match": "QUEUE"},
)
```

:::note
如果 Binance 以不同价格(例如 64,995.50)接受该订单,你会先收到一个
`OrderAccepted` 事件,随后收到一个带新价格的 `OrderUpdated` 事件。
:::

### 平仓

Binance Futures 条件单支持 `closePosition`,触发时会平掉整个仓位。
Binance 会在触发时根据当前仓位规模,在服务端解析数量。

与 `reduce_only` 不同,`closePosition` 会适应仓位规模的变化,并且
当仓位通过其他方式被平掉时,Binance 会自动取消该订单。

在 `StopMarket` 或 `MarketIfTouched` 订单上,通过 `params` 字典传入
`close_position`。不能与 `reduce_only` 组合使用。

```rust tab="Rust"
let params = Params::from([("close_position", true.into())]);
let cmd = SubmitOrder::new(order).with_params(params);
```

```python tab="Python"
strategy.submit_order(order, params={"close_position": True})
```

:::info
当设置了 `close_position` 时,Nautilus 会在 API 请求中省略
`quantity` 和 `reduceOnly`。订单数量仅用于本地风险检查。
:::

### 跟踪止损

对于 Binance 上的跟踪止损市价单:

- 使用 `activation_price`(可选)来指定跟踪机制何时激活。
- 省略时,Binance 使用提交时的当前市场价格。
- 使用 `trailing_offset` 表示回调率(以基点计)。

:::warning
不要对跟踪止损订单使用 `trigger_price`:这会导致失败并报错。请改用
`activation_price`。
:::

## Link & Trade

对于每一笔通过 Binance Rust 适配器下达的订单,NautilusTrader 的
集成 ID 都会自动作为前缀添加到所有系统生成的客户端订单 ID 中。这
通过 Binance 的
[Link and Trade](https://developers.binance.com/docs/binance_link/link-and-trade)
计划,提供了透明的订单归属,无需任何用户配置。

适配器使用确定性的双向编码,将出站的 `ClientOrderId` 值压缩为紧凑
格式,以适配 Binance `newClientOrderId` 的 36 字符限制,并在传入的
订单事件到达策略之前,将其解码回原始 ID。这一转换是完全透明的:
策略在任何时候都只会看到其原始的 `ClientOrderId` 值。

:::note
该集成 ID 前缀适用于所有订单操作,包括提交、修改、取消和状态查询。
在增加此支持之前下达的订单,会通过透传解码优雅地处理。
:::

:::info
此功能目前仅在 Rust 适配器中可用。用户可以通过在订单上传入自定义
`client_order_id`,或移除编码调用并重新编译来选择退出。这两种方式
都没有技术上的限制。
:::

### 解码客户端订单 ID

直接查询 Binance(REST API、Web UI 或你自己的 HTTP 代码)时,
`clientOrderId` 字段包含的是编码后的形式。两个工具函数可以恢复出
原始的 Nautilus `ClientOrderId`:

```python
from nautilus_trader.adapters.binance import (
    decode_binance_futures_client_order_id,
    decode_binance_spot_client_order_id,
)

# 来自 Binance REST 响应或 Web UI 的编码 ID
encoded = "x-TD67BGP9-T0A4b1H2vj50H"
original = decode_binance_spot_client_order_id(encoded)
# -> "O-20260305-120000-001-001-100"

# 合约版本
encoded_futures = "x-aHRE4BCj-U2xK9mPqR7sT1vW3y"
original_futures = decode_binance_futures_client_order_id(encoded_futures)
```

没有 broker 前缀的字符串会原样通过,因此可以安全地对任何
`clientOrderId` 值调用这两个函数。

:::note
领域层 HTTP 客户端(`BinanceSpotHttpClient`、
`BinanceFuturesHttpClient`)在返回 `OrderStatusReport` 等 Nautilus
类型时会自动解码。只有在适配器之外工作时(直接 REST 查询、Binance
Web UI,或原始场所模型),才需要手动解码。
:::

## 订单簿

订单簿可以维护为完整深度或部分深度。WebSocket 数据流的更新频率在
现货和合约之间有所不同,Nautilus 使用可用的最高频率:

- **现货 SBE 增量深度**:25ms
- **现货 JSON 增量深度**:100ms
- **合约**:0ms(不限速)

每个交易者实例、每个标的仅支持一个订单簿。当数据流订阅不同时,
Binance 数据客户端使用最新的订单簿数据订阅(增量或快照)。

订单簿快照重建会在以下情况触发:

- 首次订阅订单簿数据时。
- 数据 WebSocket 重连时。

事件顺序如下:

- 增量开始被缓冲。
- 请求并等待快照。
- 将快照响应解析为 `OrderBookDeltas`。
- 快照增量被发送给 `DataEngine`。
- 遍历已缓冲的增量,丢弃序列号不大于快照中最后一个增量的部分。
- 停止缓冲增量。
- 剩余的增量被发送给 `DataEngine`。

:::note
这一快照+缓冲的流程适用于没有显式深度的合约和现货 `BookDeltas`
订阅。现货的部分深度订阅会提供自包含的前 N 档快照。参见
[现货市场数据模式](#现货市场数据模式)。
:::

## Binance 数据差异

`QuoteTick` 上的 `ts_event` 字段在现货和合约之间有所不同。现货不
提供事件时间戳,因此适配器使用 `ts_init`(意味着 `ts_event` 和
`ts_init` 相同)。

## Binance 特有数据

你可以订阅 Binance 特有的数据流,只要它们可用。

:::note
K 线、标记价格、指数价格和资金费率可以通过 Rust 适配器以常规方式
订阅。下方的自定义数据订阅针对的是 Python 适配器。
:::

Binance USD-M 的标记价格载荷可能包含一个 `ap` 移动平均字段。Rust
适配器会解析这一原始场所字段,但不会将其作为领域数据或 Binance
自定义数据发出;Nautilus 的标记价格订阅从同一个数据流中发出标记
价格、指数价格和资金费率更新。

### `BinanceFuturesTicker`

为特定的合约标的订阅 24 小时 ticker 统计数据:

```python
from nautilus_trader.core import nautilus_pyo3 as pyo3

client_id = pyo3.ClientId.from_str("BINANCE")

self.subscribe_data(
    data_type=pyo3.DataType(
        "BinanceFuturesTicker",
        {"instrument_id": "BTCUSDT-PERP.BINANCE"},
    ),
    client_id=client_id,
)
```

适配器订阅该标的的 `@ticker` 数据流,并以
`metadata={"instrument_id": "<instrument_id>"}` 发出
`BinanceFuturesTicker` 自定义数据。Ticker 自定义数据要求提供
`instrument_id`;不支持全市场 ticker 订阅。

### `BinanceFuturesMarkPriceUpdate`

从你的 actor 或策略中订阅 `BinanceFuturesMarkPriceUpdate`
(包含资金费率信息):

```python
from nautilus_trader.adapters.binance import BinanceFuturesMarkPriceUpdate
from nautilus_trader.model import DataType
from nautilus_trader.model import ClientId

# 在你的 `on_start` 方法中
self.subscribe_data(
    data_type=DataType(BinanceFuturesMarkPriceUpdate, metadata={"instrument_id": self.instrument.id}),
    client_id=ClientId("BINANCE"),
)
```

接收到的 `BinanceFuturesMarkPriceUpdate` 对象会传给你的 `on_data`
方法。请检查其类型,因为该方法处理所有自定义/通用数据。

```python
from nautilus_trader.core import Data

def on_data(self, data: Data):
    # 首先检查数据类型
    if isinstance(data, BinanceFuturesMarkPriceUpdate):
        # 在此处理数据
```

### `BinanceFuturesLiquidation`

订阅以下任一种强平更新:

- 特定标的(`<symbol>@forceOrder`),或
- 所有符号(`!forceOrder@arr`),省略 `instrument_id` 即可。

```python
from nautilus_trader.core import nautilus_pyo3 as pyo3

client_id = pyo3.ClientId.from_str("BINANCE")

# 特定标的
self.subscribe_data(
    data_type=pyo3.DataType(
        "BinanceFuturesLiquidation",
        {"instrument_id": "BTCUSDT-PERP.BINANCE"},
    ),
    client_id=client_id,
)

# 全市场(无 instrument_id 元数据)
self.subscribe_data(
    data_type=pyo3.DataType("BinanceFuturesLiquidation"),
    client_id=client_id,
)
```

对于特定标的的订阅,`CustomData.data_type` 包含
`metadata={"instrument_id": "<instrument_id>"}`。对于全市场订阅,
该数据类型没有元数据。

当两种模式同时被订阅时,全市场订阅优先。适配器会在全市场模式活跃时
暂停按符号的强平数据流,并在取消订阅全市场后恢复活跃的按符号数据流。

## 资金费率

Rust 适配器通过 `subscribe_funding_rates` 将 `FundingRateUpdate`
作为一等数据类型发出。数据来自
[标记价格数据流](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Mark-Price-Stream)
WebSocket 端点,该端点在提供标记价格和指数价格的同时,也提供当前
资金费率和下一次资金结算时间。三种订阅
(`subscribe_mark_prices`、`subscribe_index_prices`、
`subscribe_funding_rates`)共享同一个带引用计数订阅管理的
`@markPrice@1s` 数据流。

历史资金费率可以通过 `request_funding_rates` 获取,它会查询
[获取资金费率历史](https://developers.binance.com/docs/derivatives/usds-margined-futures/market-data/rest-api/Get-Funding-Rate-History)
REST 端点(USD-M 使用 `GET /fapi/v1/fundingRate`,COIN-M 使用
`GET /dapi/v1/fundingRate`)。每行历史数据都映射为一条
`FundingRateUpdate`,其 `ts_event` 设为资金结算时间。由于该端点
不提供该信息,历史行的 `next_funding_ns` 字段为 `None`。

Python 适配器通过 `BinanceFuturesMarkPriceUpdate` 自定义数据订阅
暴露资金费率数据(参见下方的 [Binance 特有数据](#binance-特有数据))。

对于 Binance,`FundingRateUpdate` 上的 `interval` 字段为 `None`,
因为标记价格数据流和资金费率历史端点都不包含资金结算间隔字段。
Binance 通过
[获取资金费率信息](https://developers.binance.com/docs/derivatives/usds-margined-futures/market-data/rest-api/Get-Funding-Rate-Info)
REST 端点暴露了 `fundingIntervalHours`,但适配器未使用它。

## 标的状态轮询

:::info[仅限 Rust 适配器]
此功能在 Rust 数据客户端(`LiveNode`)中可用。Python 数据客户端不会
轮询状态变化。
:::

适配器会定期轮询 Binance 的 `exchangeInfo`,以检测标的交易状态的
变化。当某个符号在状态之间转换时(例如从 Trading 变为 Halt,或对于
即将到期的合约从 Trading 变为 Delivering),适配器会发出一个
`InstrumentStatus` 事件。

轮询间隔默认为 3600 秒(60 分钟),可以在数据客户端配置中通过
`instrument_status_poll_secs` 配置。设为 `0` 可完全禁用轮询。

首次连接时,适配器会从交易所信息响应中初始化其状态缓存,不发出事件。
只有后续检测到状态变化的轮询才会发出 `InstrumentStatus` 事件。如果
某个符号从交易所信息中消失(例如下市或合约到期后),适配器会发出
`NotAvailableForTrading`。

### 状态映射

#### 现货

| Binance 状态     | MarketStatusAction         |
|--------------------|----------------------------|
| Trading            | Trading                    |
| EndOfDay           | Close                      |
| Halt               | Halt                       |
| Break              | Pause                      |
| NonRepresentable   | NotAvailableForTrading     |

#### 合约(USD-M)

| Binance 状态     | MarketStatusAction         |
|--------------------|----------------------------|
| Trading            | Trading                    |
| PendingTrading     | PreOpen                    |
| PreTrading         | PreOpen                    |
| PostTrading        | PostClose                  |
| EndOfDay           | Close                      |
| Halt               | Halt                       |
| AuctionMatch       | Cross                      |
| Break              | Pause                      |

#### 合约(COIN-M)

| Binance 状态     | MarketStatusAction         |
|--------------------|----------------------------|
| Trading            | Trading                    |
| PendingTrading     | PreOpen                    |
| PreDelivering      | PreClose                   |
| Delivering         | Close                      |
| Delivered          | Close                      |
| PreSettle          | PreClose                   |
| Settling           | Close                      |
| Close              | Close                      |
| PreDelisting       | PreClose                   |
| Delisting          | Suspend                    |
| Down               | NotAvailableForTrading     |

:::note
只跟踪连接时处于可交易状态的标的。以非交易状态开始的符号(例如连接
时已暂停)不会出现在标的缓存中,因此不会监控它们的状态转换。
:::

## 速率限制

Binance 使用基于时间间隔的速率限制系统,请求权重按固定的时间窗口
(每分钟,在 :00 秒时重置)跟踪。每个 API 端点都有指定的权重成本,
总权重使用量按 IP 地址跟踪。

### 全局权重限制

以下是所有端点共享的主要限制:

| 账户类型 | 权重限制 | 时间间隔 |
|--------------|--------------|----------|
| 现货/保证金  | 6,000        | 1 分钟 |
| 合约      | 2,400        | 1 分钟 |

### 端点权重成本

某些端点每次请求的权重成本更高:

| 端点                  | 权重 | 备注                                  |
|---------------------------|--------|----------------------------------------|
| `/api/v3/order`           | 1      | 现货下单。                  |
| `/api/v3/allOrders`       | 20     | 现货历史订单(开销较大)。    |
| `/api/v3/klines`          | 2+     | 随 `limit` 参数变化。         |
| `/fapi/v1/order`          | 1      | 合约下单。               |
| `/fapi/v1/algoOrder`      | 0      | 使用订单计数限制。               |
| `/fapi/v1/allOrders`      | 20     | 合约历史订单(开销较大)。 |
| `/fapi/v1/commissionRate` | 20     | 合约手续费率查询。         |
| `/fapi/v1/klines`         | 5+     | 随 `limit` 参数变化。         |

USD-M 合约的 `POST /fapi/v1/algoOrder` 会从 `X-MBX-ORDER-COUNT-10S`
和 `X-MBX-ORDER-COUNT-1M` 中各消耗 `1`。Binance 对该端点不收取 IP
请求权重;适配器仍会将其作为本地节流模型的一部分,通过全局桶排队
处理。

### WebSocket API 限制

WebSocket API(用于用户数据流)与 REST API 共享相同的权重配额:

| 限制类型       | 数值  | 备注                                 |
|------------------|--------|---------------------------------------|
| 请求权重   | 共享 | 计入 REST API 权重配额。 |
| 握手        | 5      | 每次连接尝试的权重成本。   |
| Ping/pong 帧 | 5/秒  | 最大 ping/pong 速率。               |

### 适配器行为

适配器使用令牌桶速率限制器来近似 Binance 基于时间间隔的限制。这在
保持正常操作吞吐量的同时,降低了违反配额的风险。

对于具有动态权重的端点(例如 `/klines` 随 `limit` 参数变化),
适配器每次调用消耗一个令牌。大范围历史数据请求可能需要手动调节节奏。
监控 `X-MBX-USED-WEIGHT-*` 响应头以跟踪实际用量。

:::warning
超出允许的权重时,Binance 会返回 HTTP 429。反复违反会触发临时 IP
封禁(对于屡犯者,从 2 分钟升级到 3 天)。
:::

:::info
获取最新的速率限制,请查询 `/api/v3/exchangeInfo`(现货)或
`/fapi/v1/exchangeInfo`(合约),或参见:

- [现货 API 限制](https://developers.binance.com/docs/binance-spot-api-docs/rest-api/limits)
- [合约 API 限制](https://developers.binance.com/docs/derivatives/usds-margined-futures/general-info)

:::

## 配置

:::note
下方的配置表描述的是 **Python 适配器**。Rust 适配器使用
`BinanceDataClientConfig` 和 `BinanceExecClientConfig`,字段名称
不同。Rust 配置选项的权威列表参见 Rust 源码
`crates/adapters/binance/src/config.rs`。
:::

### 数据客户端配置选项

| 选项                             | 默认值   | 说明 |
|------------------------------------|-----------|-------------|
| `venue`                            | `BINANCE` | 注册客户端时使用的场所标识符。 |
| `api_key`                          | `None`    | Binance API key;省略时从环境变量加载。 |
| `api_secret`                       | `None`    | Binance API secret;省略时从环境变量加载。 |
| `key_type`                         | `HMAC`    | **已弃用**:密钥类型现在会根据 API secret 格式自动检测。仅在需要强制使用 `RSA` 时才需要。 |
| `account_type`                     | `SPOT`    | 数据端点的账户类型(现货、保证金、USDT 合约、币本位合约)。 |
| `base_url_http`                    | `None`    | HTTP REST 基础 URL 的覆盖值。 |
| `base_url_ws`                      | `None`    | WebSocket 基础 URL 的覆盖值。 |
| `proxy_url`                        | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `us`                                | `False`   | 为 `True` 时,将请求路由到 Binance US 端点。 |
| `environment`                      | `None`    | Binance 环境:`LIVE`、`TESTNET` 或 `DEMO`。为 `None` 时默认为 `LIVE`。 |
| `update_instruments_interval_mins` | `60`      | 标的目录刷新间隔(分钟)。 |
| `use_agg_trade_ticks`              | `False`   | 为 `True` 时,订阅聚合成交 tick 而非原始成交。合约 WebSocket 订阅始终使用 `@aggTrade`,不受此标志影响。 |
| `spot_market_data_mode`            | `Sbe`     | *仅限 Rust。* 现货市场数据传输方式(`Sbe` 或 `Json`)。参见[现货市场数据模式](#现货市场数据模式)。 |
| `instrument_status_poll_secs`      | `3600`    | *仅限 Rust。* 检测标的状态变化的交易所信息轮询间隔(秒)。设为 `0` 可禁用。 |
| `transport_backend`                | `Sockudo` | *仅限 Rust。* WebSocket 传输后端。 |

### 执行客户端配置选项

| 选项                                  | 默认值   | 说明 |
|-----------------------------------------|-----------|-------------|
| `venue`                                 | `BINANCE` | 注册客户端时使用的场所标识符。 |
| `api_key`                               | `None`    | Binance API key;省略时从环境变量加载。 |
| `api_secret`                            | `None`    | Binance API secret;省略时从环境变量加载。 |
| `key_type`                              | `HMAC`    | **已弃用**:密钥类型现在会根据 API secret 格式自动检测。仅在需要强制使用 `RSA`(仅限数据客户端,执行不支持 RSA)时才需要。 |
| `account_type`                          | `SPOT`    | 下单的账户类型(现货、保证金、USDT 合约、币本位合约)。 |
| `base_url_http`                         | `None`    | HTTP REST 基础 URL 的覆盖值。 |
| `base_url_ws`                           | `None`    | WebSocket API 基础 URL 的覆盖值。 |
| `base_url_ws_stream`                    | `None`    | WebSocket 数据流 URL 的覆盖值(合约用户数据事件推送)。 |
| `proxy_url`                             | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `us`                                    | `False`   | 为 `True` 时,将请求路由到 Binance US 端点。 |
| `environment`                           | `None`    | Binance 环境:`LIVE`、`TESTNET` 或 `DEMO`。为 `None` 时默认为 `LIVE`。 |
| `use_gtd`                               | `True`    | 为 `False` 时,将 GTD 订单重映射为 GTC,由本地管理到期。 |
| `use_reduce_only`                       | `True`    | 为 `True` 时,将 `reduce_only` 指令透传给 Binance。 |
| `use_position_ids`                      | `True`    | 启用 Binance 对冲仓位 ID;虚拟对冲时设为 `False`。 |
| `use_trade_lite`                        | `False`   | 使用包含推导手续费的 TRADE_LITE 执行事件。 |
| `treat_expired_as_canceled`             | `False`   | 为 `True` 时,将 `EXPIRED` 执行类型视为 `CANCELED`。 |
| `recv_window_ms`                        | `5,000`   | 已签名 REST 请求的接收窗口(毫秒)。 |
| `max_retries`                           | `None`    | 下单/取消/修改调用的最大重试次数。 |
| `retry_delay_initial_ms`                | `None`    | 重试尝试之间的初始延迟(毫秒)。 |
| `retry_delay_max_ms`                    | `None`    | 重试尝试之间的最大延迟(毫秒)。 |
| `futures_leverages`                     | `None`    | 合约账户中 `BinanceSymbol` 到初始杠杆的映射。 |
| `futures_margin_types`                  | `None`    | `BinanceSymbol` 到合约保证金类型(逐仓/全仓)的映射。 |
| `use_ws_trading`                        | `True`    | 使用 WebSocket 交易 API 进行订单操作(现货和 USD-M 合约)。为 `False` 时使用 HTTP。 |
| `oms_type`                              | `None`    | *仅限 Rust。* 对于处于双向仓位模式的合约账户设为 `Hedging`;`None` 使用 `Netting`。 |
| `default_taker_fee`                     | `0.0004`  | 用于对交易所生成的成交(强平、ADL、结算)进行手续费估算的默认 taker 手续费率。 |
| `bnfcr_currency`                        | `USDT`    | USD-M 合约积分交易模式:`BNFCR` 余额和手续费解析为的货币。参见 [合约积分交易模式(BNFCR)](#合约积分交易模式bnfcr)。 |
| `log_rejected_due_post_only_as_warning` | `True`    | 为 `True` 时,将 post-only 拒绝记录为警告;否则记录为错误。 |
| `transport_backend`                     | `Sockudo` | *仅限 Rust。* WebSocket 传输后端。 |

最常见的用法是配置一个包含 Binance 数据和执行客户端的实时
`TradingNode`。在你的客户端配置中添加一个 `BINANCE` 部分:

```python
from nautilus_trader.adapters.binance import BINANCE
from nautilus_trader.live.node import TradingNode

config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        BINANCE: {
            "api_key": "YOUR_BINANCE_API_KEY",
            "api_secret": "YOUR_BINANCE_API_SECRET",
            "account_type": "spot",  # {spot, usdt_futures, coin_futures}
            "base_url_http": None,  # 使用自定义端点覆盖
            "base_url_ws": None,  # 使用自定义端点覆盖
            "us": False,  # 客户端是否用于 Binance US
        },
    },
    exec_clients={
        BINANCE: {
            "api_key": "YOUR_BINANCE_API_KEY",
            "api_secret": "YOUR_BINANCE_API_SECRET",
            "account_type": "spot",  # {spot, usdt_futures, coin_futures}
            "base_url_http": None,  # 使用自定义端点覆盖
            "base_url_ws": None,  # 使用自定义端点覆盖
            "us": False,  # 客户端是否用于 Binance US
        },
    },
)
```

然后,创建一个 `TradingNode` 并添加客户端工厂:

```python
from nautilus_trader.adapters.binance import BINANCE
from nautilus_trader.adapters.binance import BinanceLiveDataClientFactory
from nautilus_trader.adapters.binance import BinanceLiveExecClientFactory
from nautilus_trader.live.node import TradingNode

# 使用配置实例化实时交易节点
node = TradingNode(config=config)

# 向节点注册客户端工厂
node.add_data_client_factory(BINANCE, BinanceLiveDataClientFactory)
node.add_exec_client_factory(BINANCE, BinanceLiveExecClientFactory)

# 最后构建节点
node.build()
```

### 合约积分交易模式(BNFCR)

Binance Futures Credits Trading Mode 是一种欧盟监管模式,在该模式下,
USD-M 合约钱包、保证金、盈亏和手续费均以 `BNFCR` 计价:一种与美元
1:1 挂钩、取代稳定币余额的内部积分单位。由于 `BNFCR` 不是可交易
资产,适配器将其映射到 `bnfcr_currency` 执行配置选项(默认
`USDT`),使账户余额和手续费能够与所交易合约结算所用的稳定币对账。
交易以 USDC 计保证金的永续合约时,请将 `bnfcr_currency` 设为
`USDC`。任何其他无法识别的合约资产都会被注册为通用加密货币,而非
失败。

### 现货市场数据模式

`spot_market_data_mode`(Rust `BinanceDataClientConfig`)选择现货
数据传输方式。它仅影响现货;合约不受影响。

| 模式   | 凭证        | 报价       |
|--------|--------------------|--------------|
| `Sbe`  | Ed25519(必需) | `bestBidAsk` |
| `Json` | 无(公开)      | `bookTicker` |

`Sbe`(默认)使用 Binance Simple Binary Encoding 数据流,需要
Ed25519 密钥(参见[密钥类型](#密钥类型));客户端在没有这些密钥的
情况下拒绝连接。`Json` 使用无需凭证的公开数据流。完整的现货
`BookDeltas` 订阅在 `Sbe` 模式下使用 25ms 的 SBE 增量深度数据流,
在 `Json` 模式下使用 100ms 的公开 JSON 增量深度数据流,并配合 REST
快照同步。显式的深度订阅使用部分订单簿快照(参见[订单簿](#订单簿))。

:::note
在 Python 中,通过 `nautilus_trader.core.nautilus_pyo3.binance` 上
的 `BinanceSpotMarketDataMode` 暴露;传统 Python 适配器配置中没有此
选项。
:::

### 密钥类型

Binance 支持三种 API 密钥类型:**Ed25519**、**HMAC-SHA256** 和
**RSA**。适配器会根据你的 API secret 格式自动检测密钥类型,因此无需
额外配置。

**强烈推荐使用 Ed25519。** Binance 推荐使用 Ed25519,因其具有更优的
性能和安全性。NautilusTrader 的未来版本将只支持 Ed25519。

| 密钥类型 | 数据客户端 | 执行客户端 | 状态 |
|----------|--------------|-------------------|--------|
| Ed25519  | ✓            | ✓                 | **推荐** |
| HMAC     | ✓            | ✓                 | 已弃用,将在未来版本中移除。 |
| RSA      | ✓            | -                 | 已弃用,执行不支持。 |

:::tip
现在就切换到 Ed25519 密钥。生成一个 Ed25519 密钥对并将其注册到
Binance。参见下方的[生成 Ed25519 密钥](#生成-ed25519-密钥)。
:::

:::note
Ed25519 密钥必须以未加密的 PEM 格式(base64 编码的 ASN.1/DER)提供。
该实现会自动从 DER 结构中提取 32 字节的种子。不支持加密(带密码保护)
的 PEM 密钥。如果你的密钥已加密,请先解密:
`openssl pkey -in encrypted.pem -out decrypted.pem`
:::

#### 生成 Ed25519 密钥

**方式 1:OpenSSL(推荐)**

```bash
# 生成私钥(PKCS#8 PEM 格式)
openssl genpkey -algorithm ed25519 -out binance_ed25519_private.pem

# 提取公钥
openssl pkey -in binance_ed25519_private.pem -pubout -out binance_ed25519_public.pem
```

**方式 2:Binance Key Generator**

从发行页面下载
[Binance Asymmetric Key Generator](https://github.com/binance/asymmetric-key-generator)
并运行它以生成密钥对。

**在 Binance 上注册**

1. 登录 Binance,进入 **Profile** -> **API Management**
2. 点击 **Create API**,选择 **Self-generated**
3. 粘贴你公钥文件的内容(包括 `-----BEGIN PUBLIC KEY-----` 头尾标记)
4. 配置权限(启用 Spot & Margin Trading 等)

**在 NautilusTrader 中使用**

将私钥设置为你的 API secret:

```bash
export BINANCE_API_KEY="your-api-key-from-binance"
export BINANCE_API_SECRET="$(cat binance_ed25519_private.pem)"
```

或在配置中直接传入 PEM 内容。

:::warning
妥善保管你的私钥。切勿分享或将其提交到版本控制中。
:::

### API 凭证

将凭证直接传给配置对象,或设置相应的环境变量(各环境的变量参见
[环境](#环境))。

:::tip
所有客户端都使用 Ed25519 密钥。HMAC 密钥对数据和执行客户端仍然
有效,但 Ed25519 提供更好的性能,并将在未来版本中成为唯一支持的
密钥类型。参见[密钥类型](#密钥类型)。
:::

:::warning
`BINANCE_ED25519_*` 和 `BINANCE_*_ED25519_*` 环境变量已针对
现货/保证金移除。对于合约,它们已被弃用,将在未来版本中移除。请将
其重命名为 `BINANCE_API_KEY` / `BINANCE_API_SECRET`(Ed25519 密钥
现在会被自动检测)。
:::

交易节点启动时,你会收到关于凭证是否有效以及是否具有交易权限的确认。

### 账户类型

使用 `BinanceAccountType` 枚举设置 `account_type`:

- `SPOT`
- `USDT_FUTURES`(以 USDT 或 BUSD 稳定币作为抵押品)
- `COIN_FUTURES`(以其他加密货币作为抵押品)

:::note
枚举中存在 `MARGIN` 和 `ISOLATED_MARGIN` 账户类型,但保证金交易
尚未实现。参见[产品支持](#产品支持)。
:::

### 基础 URL 覆盖

可以覆盖 HTTP REST 和 WebSocket API 的默认基础 URL。这在配置 API
集群,或 Binance 提供了专用端点时很有用。

### Binance US

在配置中设置 `us=True` 以使用 Binance US 端点(默认为 `False`)。
US 账户可用的所有功能与标准 Binance 表现一致。

### 环境

Binance 提供三种交易环境,各自拥有独立的 API 凭证和端点。
`environment` 配置选项用于选择使用哪一种。

| 环境 | 配置                  | 说明                                                            |
|-------------|-------------------------|------------------------------------------------------------------------|
| **Live**    | `environment="LIVE"`    | 使用真实资金的生产交易(默认)。                          |
| **Demo**    | `environment="DEMO"`    | 使用模拟现货和合约资金的模拟交易。                    |
| **Testnet** | `environment="TESTNET"` | 传统的现货和合约测试网络。                                  |

#### Live(生产环境)

用于真实资金实盘交易的默认环境。使用你的主 Binance 账户凭证。

```python
config = BinanceExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    account_type=BinanceAccountType.SPOT,
    # environment=BinanceEnvironment.LIVE (默认)
)
```

| 变量             | 说明         |
|----------------------|---------------------|
| `BINANCE_API_KEY`    | 实盘 API key。       |
| `BINANCE_API_SECRET` | 实盘 API secret。    |

#### 模拟交易(Demo)

在生产基础设施上使用模拟资金进行练习交易。模拟账户使用与你实盘账户
相同的 Binance 登录信息,但使用虚拟余额交易。

**如何获取模拟凭证:**

1. 在 [binance.com/en/demo-trading](https://www.binance.com/en/demo-trading)
   登录。
2. 进入 **API Management**,创建一个模拟 API key。
3. 模拟密钥适用于现货和合约的模拟端点。

| 端点       | URL                           |
|----------------|-------------------------------|
| 现货 HTTP      | `demo-api.binance.com`        |
| 现货 WS        | `demo-stream.binance.com`     |
| USD-M HTTP     | `demo-fapi.binance.com`       |
| USD-M WS       | `demo-fstream.binance.com`    |
| COIN-M HTTP    | `demo-dapi.binance.com`       |
| COIN-M WS      | `demo-dstream.binance.com`    |

```python
config = BinanceExecClientConfig(
    api_key="YOUR_DEMO_API_KEY",
    api_secret="YOUR_DEMO_API_SECRET",
    account_type=BinanceAccountType.SPOT,
    environment=BinanceEnvironment.DEMO,
)
```

| 变量                  | 说明      |
|---------------------------|------------------|
| `BINANCE_DEMO_API_KEY`    | 模拟 API key。    |
| `BINANCE_DEMO_API_SECRET` | 模拟 API secret。 |

#### Testnet

一个拥有自己用户账户、余额和订单簿的传统测试网络。对于新的模拟交易
设置,建议优先使用 `environment=BinanceEnvironment.DEMO`。现货测试网
仍位于 `testnet.binance.vision`;合约测试网端点可能会通过 Demo
Trading 基础设施路由。

**如何获取现货测试网凭证:**

1. 前往 [testnet.binance.vision](https://testnet.binance.vision/)。
2. 使用 GitHub 登录。
3. 生成一个 API key(HMAC、RSA 或 Ed25519)。

**合约测试网:** 使用 `BinanceEnvironment.TESTNET` 的现有配置仍可
继续工作,但新的合约测试应使用 `BinanceEnvironment.DEMO`。

```python
config = BinanceExecClientConfig(
    api_key="YOUR_TESTNET_API_KEY",
    api_secret="YOUR_TESTNET_API_SECRET",
    account_type=BinanceAccountType.SPOT,
    environment=BinanceEnvironment.TESTNET,
)
```

| 变量                             | 说明                                        |
|--------------------------------------|----------------------------------------------------|
| `BINANCE_TESTNET_API_KEY`            | 现货测试网 API key。                              |
| `BINANCE_TESTNET_API_SECRET`         | 现货测试网 API secret。                           |
| `BINANCE_FUTURES_TESTNET_API_KEY`    | 合约测试网 API key。                           |
| `BINANCE_FUTURES_TESTNET_API_SECRET` | 合约测试网 API secret。                        |

:::note
测试网凭证与你的实盘账户完全独立。市场数据和流动性与生产环境不同。
:::

### 聚合成交

Binance 提供聚合成交数据端点,作为成交的另一种数据来源。与默认的
成交端点不同,聚合成交端点可以返回 `start_time` 和 `end_time` 之间
的所有 tick。

设置 `use_agg_trade_ticks=True` 以使用聚合成交(默认为 `False`)。

:::note
对于合约(USD-M 和 COIN-M),WebSocket 成交订阅始终使用
`@aggTrade`。Binance 只在合约 WebSocket 上发布聚合成交;传统的
`@trade` 数据流未被记录文档,已被静默。HTTP 的
`request_trade_ticks` 路径仍遵循 `use_agg_trade_ticks`。
:::

### 手续费率查询

默认情况下,Binance Futures 标的根据你的 VIP 等级使用手续费分档表。
对于拥有负 maker 手续费的做市商账户,或需要精确费率时,可以启用
按符号的手续费率查询:

```python
from nautilus_trader.adapters.binance import BinanceInstrumentProviderConfig

instrument_provider=BinanceInstrumentProviderConfig(
    load_all=True,
    query_commission_rates=True,  # 按符号查询准确费率
)
```

启用后,适配器会在标的加载期间,为每个符号并行查询 Binance 的
`/fapi/v1/commissionRate` 端点。适用于:

- 拥有负 maker 手续费的做市商账户。
- 拥有自定义手续费安排的账户。
- 用于盈亏计算的精确手续费率。

适配器使用带速率限制的并行请求(每分钟 120 次请求,考虑到该端点
的权重为 20)。如果查询失败,会回退到手续费分档表。

### 解析器警告

某些 Binance 标的如果包含超出平台处理能力的字段值,则无法解析为
Nautilus 对象。这些标的会被跳过,并记录一条警告。

要抑制这些警告:

```python
from nautilus_trader.config import InstrumentProviderConfig

instrument_provider=InstrumentProviderConfig(
    load_all=True,
    log_warnings=False,
)
```

### 合约对冲模式

Binance Futures 对冲模式允许在同一标的上同时持有多头和空头仓位。

以下步骤适用于 Python 适配器。对于 Rust 适配器(包括其 Python
绑定),在 Binance 上配置对冲模式,并在 Rust 中将 `oms_type` 设为
`OmsType::Hedging`,或在 Python 中设为 `OmsType.HEDGING`。保持
`use_position_ids` 启用,以跟踪场所的多空两侧。

使用对冲模式:

1. 在启动策略之前,在 Binance 上配置对冲模式。
2. 在 `BinanceExecClientConfig` 中设置 `use_reduce_only=False`
   (默认为 `True`)。

    ```python
    from nautilus_trader.adapters.binance import BINANCE

    config = TradingNodeConfig(
        ...,  # 省略
        data_clients={
            BINANCE: BinanceDataClientConfig(
                api_key=None,  # 'BINANCE_API_KEY' 环境变量
                api_secret=None,  # 'BINANCE_API_SECRET' 环境变量
                account_type=BinanceAccountType.USDT_FUTURES,
                base_url_http=None,  # 使用自定义端点覆盖
                base_url_ws=None,  # 使用自定义端点覆盖
            ),
        },
        exec_clients={
            BINANCE: BinanceExecClientConfig(
                api_key=None,  # 'BINANCE_API_KEY' 环境变量
                api_secret=None,  # 'BINANCE_API_SECRET' 环境变量
                account_type=BinanceAccountType.USDT_FUTURES,
                base_url_http=None,  # 使用自定义端点覆盖
                base_url_ws=None,  # 使用自定义端点覆盖
                use_reduce_only=False,  # 对冲模式下必须禁用
            ),
        }
    )
    ```

3. 提交订单时,在 `position_id` 中使用 `LONG` 或 `SHORT` 后缀来
   表明仓位方向。

    ```python
    class EMACrossHedgeMode(Strategy):
        ...,  # 省略
        def buy(self) -> None:
            order: MarketOrder = self.order_factory.market(
                instrument_id=self.instrument_id,
                order_side=OrderSide.BUY,
                quantity=self.instrument.make_qty(self.trade_size),
                # time_in_force=TimeInForce.FOK,
            )

            # LONG 后缀会被 Binance 适配器识别为多头仓位。
            position_id = PositionId(f"{self.instrument_id}-LONG")
            self.submit_order(order, position_id)

        def sell(self) -> None:
            order: MarketOrder = self.order_factory.market(
                instrument_id=self.instrument_id,
                order_side=OrderSide.SELL,
                quantity=self.instrument.make_qty(self.trade_size),
                # time_in_force=TimeInForce.FOK,
            )
            # SHORT 后缀会被 Binance 适配器识别为空头仓位。
            position_id = PositionId(f"{self.instrument_id}-SHORT")
            self.submit_order(order, position_id)
    ```

### COIN-M / USD-M 架构

Binance 的 COIN-M 合约(CM / DAPI)和 USD-M 合约(UM / FAPI)共享
统一架构。本节涵盖对适配器的相关影响。

完整详情参见
[重要的 CM-UM 集成通知](https://developers.binance.com/docs/derivatives/coin-margined-futures/Important-CM-UM-Integration-Notice)。

#### WebSocket 数据流

市场数据流载荷在 `<symbol>@aggTrade`、`<symbol>@ticker`、
`<symbol>@bookTicker`、`<symbol>@depth<levels>`、
`<symbol>@miniTicker` 以及所有 `!*@arr` 数据流上都包含 `st`
(符号类型:`1` = UM,`2` = CM)。UM 侧的单符号数据流还在
`<symbol>@bookTicker`、`<symbol>@depth<levels>`、
`<symbol>@miniTicker` 和 `<symbol>@rpiDepth` 上包含 `ps`
(pair symbol)。

适配器使用 `msgspec`(Python)和 `serde`(Rust)进行 JSON 解码,
两者默认都会忽略未知字段。这些字段会被静默丢弃。

全市场数组数据流(`!ticker@arr`、`!miniTicker@arr`、
`!bookTicker`、`!forceOrder@arr`、`!contractInfo`)在 `fstream` 和
`dstream` 上都推送合并后的 UM + CM 内容。

#### REST 与 WebSocket API

- 下单和修改的确认响应不包含 `avgPrice` / `cumQuote` /
  `cumBase`。适配器从用户数据流中获取成交信息。查询端点
  (`GET /{f,d}api/v1/order`、`userTrades`)仍会返回这些字段。
- `PUT /dapi/v1/order`(COIN-M 修改)要求同时提供 `price` 和
  `quantity`。适配器的 `_modify_order` 会发送这两个字段,并回退
  到缓存订单的值。
- COIN-M 条件单(STOP、TAKE_PROFIT 等)使用
  `/dapi/v1/algoOrder` 端点。适配器会将所有合约条件单路由到算法单
  API。
- 使用无效符号调用 `GET /dapi/v1/openOrders` 会返回错误 `-1121`。

#### 速率限制池

UM 和 CM 每个 IP 共享同一个速率限制池(每分钟 2400 权重,每分钟
1200 笔订单,每 10 秒 300 笔订单)。适配器为 UM 和 CM 创建独立的
HTTP 客户端实例,各自拥有自己的速率限制器。如果一个节点同时驱动
UM 和 CM 客户端,合计流量可能超出共享的服务端预算。

#### dualSidePosition

UM 和 CM 共享相同的 `dualSidePosition` 设置。在任一方修改它都会
影响双方。在切换该设置之前,请确保 UM 和 CM 都没有未成交订单或
仓位。

## 贡献

:::info
如需为 Binance 适配器贡献代码,参见
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
