# BitMEX

BitMEX(Bitcoin Mercantile Exchange)成立于 2014 年,是一个加密货币
衍生品交易平台,提供现货、永续合约、传统期货、预测市场以及其他高级
交易产品。该集成支持接入 BitMEX 的实时市场数据与订单执行。

## 概览

该适配器以 Rust 实现,并提供可选的 Python 绑定,便于在基于 Python 的
工作流中使用。它不需要外部 BitMEX 客户端库;核心组件被编译为静态库,
并在构建期间自动链接。

## 示例

可以在[这里](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/bitmex/)找到实时示例脚本。

## 组件

本指南假设交易者需要同时配置实时市场数据源和交易执行。BitMEX 适配器
包含多个组件,可以组合使用,也可以单独使用,视具体使用场景而定。

- `BitmexHttpClient`:底层 HTTP API 连接。
- `BitmexWebSocketClient`:底层 WebSocket API 连接。
- `BitmexInstrumentProvider`:标的解析与加载功能。
- `BitmexDataClient`:市场数据流管理器。
- `BitmexExecutionClient`:账户管理与交易执行网关。
- `BitmexDataClientFactory`:BitMEX 数据客户端工厂(供交易节点构建器
  使用)。
- `BitmexExecutionClientFactory`:BitMEX 执行客户端工厂(供交易节点
  构建器使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),不必直接与这些底层
组件打交道。
:::

## BitMEX 文档

BitMEX 为用户提供了详尽的文档:

- [BitMEX API Explorer](https://www.bitmex.com/app/restAPI) - 交互式 API 文档。
- [BitMEX API 文档](https://www.bitmex.com/app/apiOverview) - 完整的 API 参考。
- [BitMEX 交易所规则](https://www.bitmex.com/exchange-rules) - 官方交易所规则与法规。
- [合约指南](https://www.bitmex.com/app/contract) - 详细的合约规格。
- [现货交易指南](https://www.bitmex.com/app/spotGuide) - 现货交易概览。
- [永续合约指南](https://www.bitmex.com/app/perpetualContractsGuide) - 永续互换详解。
- [期货合约指南](https://www.bitmex.com/app/futuresGuide) - 传统期货信息。

建议将本 NautilusTrader 集成指南与 BitMEX 官方文档结合参考。

## 产品支持

| 产品类型      | 数据源 | 交易 | 备注                                               |
|-------------------|-----------|---------|-----------------------------------------------------|
| 现货              | ✓         | ✓       | 交易对有限,与衍生品共用统一钱包。     |
| 永续互换   | ✓         | ✓       | 提供反向和正向合约。             |
| 股票永续合约  | -         | -       | *尚不支持*。目前仅在测试网上提供。     |
| 期货           | ✓         | ✓       | 传统的固定到期日合约。             |
| Quanto 期货    | ✓         | ✓       | 以不同于标的资产的货币结算。      |
| 预测市场| ✓         | ✓       | 基于事件的合约,0-100 定价,以 USDT 结算。 |
| 期权           | -         | -       | *BitMEX 不提供*。                           |

:::note
BitMEX 已停止其期权产品,专注于核心的衍生品和现货业务。
:::

### 现货交易

- 直接的代币/币种交易,即时结算。
- 主要交易对包括 XBT/USDT、ETH/USDT、ETH/XBT。
- 其他山寨币交易对(LINK、SOL、UNI、APE、AXS、BMEX 对 USDT)。

### 衍生品

- **永续合约**:反向(例如 XBTUSD)和正向(例如 ETHUSDT)。
- **传统期货**:固定到期日合约。
- **Quanto 期货**:以不同于标的资产的货币结算的合约。
- **预测市场**:基于事件的衍生品(例如 P_FTXZ26、P_SBFJAILZ26),
  允许交易者对涉及加密货币、金融及其他事件的结果进行投机。无杠杆,
  定价范围 0-100,以 USDT 结算。
- **股票永续合约**:基于股票的永续合约(例如 SPYUSDT、CRCLUSDT)。
  *目前仅在测试网上提供;该适配器尚不支持。*

### 标的类型代码(CFI)

BitMEX 使用遵循 ISO 10962 标准的 CFI(金融工具分类)代码。该适配器
识别以下标的类型代码:

| 代码     | 类型                 | 状态      | 说明                                     |
|----------|----------------------|-------------|-------------------------------------------------|
| `FFWCSX` | 永续合约   | 支持   | 基于加密货币的永续互换(例如 XBTUSD)。    |
| `FFWCSF` | 外汇永续         | 支持   | 基于外汇的永续合约。                   |
| `FFCCSX` | 期货              | 支持   | 有固定到期日的日历期货。         |
| `FFICSX` | 预测市场    | 支持   | 基于事件的预测合约。               |
| `IFXXXP` | 现货                 | 支持   | 现货交易对。                             |
| `FFSCSX` | 股票永续      | 不支持 | 基于股票/股权的永续合约。仅限测试网。    |
| `SRMCSX` | 掉期利率            | 不支持 | 基于收益率的掉期产品(历史遗留)。         |
| `MR****` | 指数                | 参考   | BitMEX 指数(不可交易,仅作价格参考)。  |

更多详情参见 [BitMEX Typ 值](https://support.bitmex.com/hc/en-gb/articles/6299296145565-What-are-the-Typ-Values-for-Instrument-endpoint)。

## 符号规则

BitMEX 对其交易符号使用特定的命名约定。理解这一约定对于正确识别和交易
标的至关重要。

### 符号格式

BitMEX 符号通常遵循以下模式:

- **现货交易对**:基础货币 + 计价货币(例如 `XBT/USDT`、`ETH/USDT`)。
- **永续合约**:基础货币 + 计价货币(例如 `XBTUSD`、`ETHUSD`)。
- **期货合约**:基础货币 + 到期代码(例如 `XBTM24`、`ETHH25`)。
- **Quanto 合约**:非美元结算合约的特殊命名。
- **预测市场**:`P_` 前缀 + 事件标识符 + 到期代码(例如
  `P_POWELLK26`、`P_FTXZ26`)。

:::info
BitMEX 使用 `XBT` 而非 `BTC` 作为比特币的符号。这遵循 ISO 4217 货币
代码标准,其中 “X” 表示非国家货币。XBT 和 BTC 指的是同一种资产 ——
比特币。
:::

### 到期代码

期货合约使用标准的期货月份代码:

- `F` = 一月
- `G` = 二月
- `H` = 三月
- `J` = 四月
- `K` = 五月
- `M` = 六月
- `N` = 七月
- `Q` = 八月
- `U` = 九月
- `V` = 十月
- `X` = 十一月
- `Z` = 十二月

后接年份(例如 `24` 表示 2024 年,`25` 表示 2025 年)。

### NautilusTrader 标的 ID

在 NautilusTrader 内部,BitMEX 标的直接使用 BitMEX 原生符号,并结合
场所标识符来标识:

```python
from nautilus_trader.model.identifiers import InstrumentId

# 现货交易对(注意:符号中没有斜杠)
spot_id = InstrumentId.from_str("XBTUSDT.BITMEX")  # XBT/USDT 现货
eth_spot_id = InstrumentId.from_str("ETHUSDT.BITMEX")  # ETH/USDT 现货

# 永续合约
perp_id = InstrumentId.from_str("XBTUSD.BITMEX")  # 比特币永续(反向)
linear_perp_id = InstrumentId.from_str("ETHUSDT.BITMEX")  # 以太坊永续(正向)

# 期货合约(2024 年 6 月)
futures_id = InstrumentId.from_str("XBTM24.BITMEX")  # 2024 年 6 月到期的比特币期货

# 预测市场合约
prediction_id = InstrumentId.from_str("P_XBTETFV23.BITMEX")  # 2023 年 10 月到期的比特币 ETF SEC 批准预测
```

:::note
NautilusTrader 中的 BitMEX 现货符号不包含 BitMEX UI 中出现的斜杠(/)。
使用 `XBTUSDT` 而非 `XBT/USDT`。
:::

### 数量缩放

BitMEX 以*合约*为单位报告现货和衍生品数量。每份合约对应的实际资产
规模是交易所特定的,并公布在标的定义中:

- `lotSize` – 可交易的最小合约数量。
- `underlyingToPositionMultiplier` – 每单位标的资产对应的合约数量。

例如,SOL/USDT 现货标的(`SOLUSDT`)的 `lotSize = 1000`、
`underlyingToPositionMultiplier = 10000`,意味着一份合约代表
`1 / 10000 = 0.0001` SOL,最小订单(`lotSize * contract_size`)为
`0.1` SOL。适配器目前直接根据这些字段推导合约规模,并相应地缩放入站
市场数据和出站订单,因此 Nautilus 中的数量始终以基础货币单位(SOL、
ETH 等)表示。

关于这些字段的详情,参见 BitMEX API 文档:
<https://www.bitmex.com/app/apiOverview#Instrument-Properties>。

## 订单能力

BitMEX 集成支持以下订单类型和执行功能。

### 订单类型

| 订单类型             | 是否支持 | 备注                                         |
|------------------------|-----------|-----------------------------------------------|
| `MARKET`               | ✓         | 以当前市场价格立即执行。不支持以计价货币计量数量。 |
| `LIMIT`                | ✓         | 仅以指定价格或更优价格执行。   |
| `STOP_MARKET`          | ✓         | 支持(设置 `trigger_price`)。              |
| `STOP_LIMIT`           | ✓         | 支持(设置 `price` 和 `trigger_price`)。  |
| `MARKET_IF_TOUCHED`    | ✓         | 支持(设置 `trigger_price`)。              |
| `LIMIT_IF_TOUCHED`     | ✓         | 支持(设置 `price` 和 `trigger_price`)。  |
| `TRAILING_STOP_MARKET` | ✓         | 支持(设置 `trailing_offset`)。仅限价格偏移类型。 |
| `TRAILING_STOP_LIMIT`  | ✓         | 支持(设置 `price` 和 `trailing_offset`)。仅限价格偏移类型。 |

### 执行指令

| 指令   | 是否支持 | 备注                                                                             |
|---------------|-----------|-----------------------------------------------------------------------------------|
| `post_only`   | ✓         | 通过 `LIMIT` 订单上的 `ParticipateDoNotInitiate` 执行指令支持。 |
| `reduce_only` | ✓         | 通过 `ReduceOnly` 执行指令支持。                                 |

:::note
会吃单的 post-only 订单会被 BitMEX 取消,而非拒绝。该集成会将这些
呈现为带 `due_post_only=True` 的拒绝,以便策略能够一致地处理它们。
:::

### 触发类型

BitMEX 为以下订单类型支持多种参考价格来评估止损/条件单触发:

- `STOP_MARKET`
- `STOP_LIMIT`
- `MARKET_IF_TOUCHED`
- `LIMIT_IF_TOUCHED`

请根据你的策略和/或风险偏好选择合适的触发类型。

| 参考价格 | Nautilus `TriggerType` | BitMEX 取值  | 备注                                                                           |
|-----------------|------------------------|---------------|---------------------------------------------------------------------------------|
| 最新成交      | `LAST_PRICE`           | `LastPrice`   | BitMEX 默认;基于最新成交价触发。                              |
| 标记价格      | `MARK_PRICE`           | `MarkPrice`   | 对许多止损场景推荐使用,可减少因价格插针导致的止损。 |
| 指数价格     | `INDEX_PRICE`          | `IndexPrice`  | 跟踪外部指数;对某些合约有用。                           |

- 如果未提供 `trigger_type`,BitMEX 会使用其场所默认值(`LastPrice`)。
- 这些触发参考由交易所评估;订单在触发前会一直挂在场所上。

**示例**:

```python
from nautilus_trader.model.enums import TriggerType

order = self.order_factory.stop_market(
    instrument_id=instrument_id,
    order_side=order_side,
    quantity=qty,
    trigger_price=trigger,
    trigger_type=TriggerType.MARK_PRICE,  # 使用 BitMEX 标记价格作为参考
)
```

`examples/live/bitmex/bitmex_exec_tester.py` 中的 `ExecTester` 示例
配置也展示了如何设置
`stop_trigger_type=TriggerType.MARK_PRICE`。

### 跟踪止损

BitMEX 支持跟踪止损订单,当市场向有利方向移动时,会自动调整止损价格。
适配器将 `TRAILING_STOP_MARKET` 和 `TRAILING_STOP_LIMIT` 订单映射为
带 `TrailingStopPeg` 价格类型的 BitMEX 挂钩订单(pegged orders);
限价变体还会额外携带限价 `price`。

**限制:**

- 仅支持 `PRICE` 跟踪偏移类型(绝对价格偏移,而非基点或 tick)。
- 偏移方向由系统自动处理:卖出止损使用负偏移,买入止损使用正偏移。
- 触发类型可以与跟踪止损结合使用以获得额外控制。

**示例**:

```python
from nautilus_trader.model.enums import TrailingOffsetType

order = self.order_factory.trailing_stop_market(
    instrument_id=instrument_id,
    order_side=OrderSide.SELL,
    quantity=qty,
    trailing_offset=Decimal("100"),  # 100 美元的跟踪偏移
    trailing_offset_type=TrailingOffsetType.PRICE,
    trigger_type=TriggerType.LAST_PRICE,  # 可选
)
```

:::note
BitMEX 会随市场变化定期更新跟踪止损价格。当市场朝触发水平移动时,
止损价格会被冻结。当前更新频率的详情参见
[BitMEX API 文档](https://www.bitmex.com/app/perpetualContractsGuide)。
:::

### 挂钩订单(Pegged orders)

BitMEX 支持自动跟踪参考价格的挂钩订单(BBO)。适配器通过
`submit_order` 上的 `params` 字典支持挂钩订单,该字典会在交易所侧将
订单类型覆盖为 `Pegged`。

| 挂钩价格类型 | 说明                                                      |
|----------------|------------------------------------------------------------------|
| `PrimaryPeg`   | 挂钩最优买价(买入)或最优卖价(卖出)。                   |
| `MarketPeg`    | 挂钩对手方(买入挂钩最优卖价,卖出挂钩最优买价)。 |
| `MidPricePeg`  | 挂钩买卖中间价。                       |
| `LastPeg`      | 挂钩最新成交价。                                   |

**要求**:

- 底层订单必须是 `LIMIT` 订单。其他订单类型会被拒绝。
- `peg_price_type` 为必填;`peg_offset_value` 为可选(默认为 0)。
- `peg_offset_value` 可以为负数(例如卖方偏移)或分数。

**示例**:

```python
# 以零偏移挂钩最优买价(BBO)
order = self.order_factory.limit(
    instrument_id=instrument_id,
    order_side=OrderSide.BUY,
    quantity=qty,
    price=price,  # LIMIT 订单要求提供,但会被挂钩覆盖
)
self.submit_order(order, params={"peg_price_type": "PrimaryPeg", "peg_offset_value": "0"})

# 以 -0.5 的偏移挂钩中间价
self.submit_order(order, params={"peg_price_type": "MidPricePeg", "peg_offset_value": "-0.5"})
```

:::note
构造 `LimitOrder` 时仍需要提供 `price` 字段,但 BitMEX 对挂钩订单会
忽略它,而是持续跟踪参考价格加偏移量。
:::

### 有效期

| 有效期  | 是否支持 | 备注                                               |
|----------------|-----------|-----------------------------------------------------|
| `GTC`          | ✓         | 撤销前有效(默认)。                       |
| `GTD`          | -         | *BitMEX 不支持*。                          |
| `FOK`          | ✓         | 全部成交或取消 —— 整单成交或取消。       |
| `IOC`          | ✓         | 立即成交或取消 —— 允许部分成交。         |
| `DAY`          | ✓         | 于 00:00 UTC 过期(BitMEX 交易日边界)。 |

:::note
`DAY` 订单在 UTC 时间凌晨 0 点过期,这标志着 BitMEX 交易日的边界
(当日交易时段的结束)。完整详情参见
[BitMEX 交易所规则](https://www.bitmex.com/exchange-rules) 和
[API 文档](https://www.bitmex.com/api/explorer/)。
:::

### 高级订单功能

| 功能            | 是否支持 | 备注                                                                    |
|--------------------|-----------|--------------------------------------------------------------------------|
| 订单修改 | ✓         | 修改价格、数量和触发价格。                               |
| 括号单     | ✓         | 使用 `contingency_type` 和 `linked_order_ids`。                           |
| 冰山订单     | ✓         | 使用 `display_qty`。                                                       |
| 跟踪止损     | ✓         | 使用 `trailing_offset`。仅限价格偏移类型。                           |
| 挂钩订单      | ✓         | 使用带 `peg_price_type` 的 `params`。参见 [挂钩订单](#挂钩订单pegged-orders)。 |

### 批量操作

| 操作          | 是否支持 | 备注                                       |
|--------------------|-----------|---------------------------------------------|
| 批量提交       | -         | *BitMEX 不支持*。                  |
| 批量修改       | -         | *BitMEX 不支持*。                  |
| 批量取消       | ✓         | 在单次请求中取消多个订单。 |

### 仓位管理

| 功能             | 是否支持 | 备注                                              |
|---------------------|-----------|-----------------------------------------------------|
| 查询仓位     | ✓         | 通过 REST 和 WebSocket 进行实时仓位更新。 |
| 全仓保证金        | ✓         | 默认保证金模式。                               |
| 逐仓保证金     | ✓         |                                                    |

### 订单查询

| 功能              | 是否支持 | 备注                                        |
|----------------------|-----------|----------------------------------------------|
| 查询未成交订单    | ✓         | 列出所有活跃订单。                      |
| 查询订单历史  | ✓         | 历史订单数据。                       |
| 订单状态更新 | ✓         | 通过 WebSocket 实时更新订单状态变化。 |
| 成交历史        | ✓         | 执行与成交报告。                  |

### 强平与 ADL 处理

BitMEX 通过 `execution` 频道上的 `execType` 字段暴露强制平仓成交:

| `execType`    | 含义                                                      |
|---------------|--------------------------------------------------------------|
| `Trade`       | 正常执行(用户或 taker 发起)。                  |
| `Liquidation` | 仓位被强平引擎强制平仓。BitMEX 对自动减仓和对手方强平成交都使用该代码。 |
| `Bankruptcy`  | 账户破产;仓位与保险基金平仓。 |
| `Settlement`  | 计划中的合约结算。                               |
| `Funding`     | 未平仓仓位的资金费用结算。                        |

适配器通过标准 `FillReport` 路径路由 `Liquidation` 和 `Bankruptcy`,
并在破产执行时记录一条警告。BitMEX 的公开 API **不会**在 `execType`
中区分自动减仓与对手方强平;两者都显示为 `Liquidation`。通常可以通过
零手续费,以及本地缓存中缺少匹配的订单(引擎会为其创建一个外部订单)
来识别一个 ADL 平仓的仓位。

上游参考资料:

- [`/execution` 字段定义](https://support.bitmex.com/hc/en-gb/articles/6205689858077--execution-field-definitions)
- [自动减仓概览](https://support.bitmex.com/hc/en-gb/articles/18589621443357-What-is-Auto-Deleveraging)
- [强平概览](https://support.bitmex.com/hc/en-gb/articles/360003188434-Liquidations)

## 市场数据

- 订单簿增量:仅限 `L2_MBP`;`depth` 为 0(全量订单簿)或 25。
- 订单簿深度 10 快照:通过 `orderBook10` 频道提供固定的 10 档。
- 通过 WebSocket 支持报价、成交和标的更新。
- 在适用的情况下支持资金费率、标记价格和指数价格。
- REST 请求:
  - 带可选深度的当前 L2 订单簿快照。
  - 带可选 `start`、`end` 和 `limit` 过滤器的成交 tick(每次调用最多
    1,000 条结果)。
  - 外部聚合最新价的时间 K 线(`1m`、`5m`、`1h`、`1d`),包括可选的
    部分区间。
  - 带可选 `start`、`end` 和 `limit` 过滤器的资金费率。

:::note
BitMEX 表格 REST 端点的分页大小因端点而异。资金费率请求以 500 行为
一页进行分页,直到达到请求的 limit 或时间范围为止;成交 tick 和
K 线请求目前每次请求返回一个场所页,最多允许 1,000 行。
:::

### 成交 ID 推导

成交 tick 和成交使用场所提供的 `trdMatchID`(UUID)作为 `TradeId`。
当场所省略 `trdMatchID`(分桶成交或某些执行类型)时,执行路径会回退
使用场所的 `execID`;市场数据解析器会回退到对符号、`ts_event`、
价格、数量和方向进行确定性 FNV-1a 哈希计算。相同的场所事件在多次
回放中会产生相同的成交 ID,从而保持下游去重的正确性。

## 连接管理

### HTTP Keep-Alive

BitMEX 适配器使用 HTTP keep-alive 以获得最佳性能:

- **连接池**:连接会被自动池化并复用。
- **Keep-alive 超时**:90 秒(与 BitMEX 服务端超时匹配)。
- **自动重连**:失败的连接会被自动重新建立。
- **SSL 会话缓存**:减少后续请求的握手开销。

该配置通过维护持久连接、避免为每个请求都重新建立连接的开销,确保与
BitMEX 服务器之间的低延迟通信。

### 请求身份验证与过期

BitMEX 使用 `api-expires` 请求头进行请求身份验证,以防止重放攻击:

- 已签名的请求包含一个 `api-expires` UNIX 时间戳,设置为
  `recv_window_ms / 1000` 秒之后(默认 10 秒)。
- 一旦该时间戳已过,BitMEX 会拒绝任何请求,因此请将延迟控制在你配置的
  窗口内。

## 资金费率

适配器从
[Funding](https://www.bitmex.com/app/wsAPI#Funding)
WebSocket 数据流接收资金费率数据。BitMEX 在每条消息中返回一个
`fundingInterval` 日期时间字段,适配器读取其中的小时和分钟,以计算
`FundingRateUpdate` 上的 `interval` 字段。

## 速率限制

BitMEX 实现了双层速率限制系统:

### REST 限制

- **突发限制**:已认证用户每秒 10 次请求(适用于下单、修改和取消
  端点)。
- **滚动分钟限制**:已认证用户每分钟 120 次请求(未认证用户每分钟
  30 次请求)。
- **订单上限**:每个符号 200 个未成交订单和 10 个止损订单;超出这些
  上限会触发交易所侧的拒绝。

适配器使用配置的 `max_requests_per_second` 和 `max_requests_per_minute`
值在本地强制执行这些配额。

### WebSocket 限制

- 连接请求:遵循交易所指导(目前每个 IP 每秒 3 个连接)。
- 私有数据流需要身份验证;如果超出限制,适配器会自动重连。

:::warning
超出 BitMEX 速率限制会返回 HTTP 429,并可能触发临时 IP 封禁;持续的
4xx/5xx 错误可能延长封禁期。
:::

### 可配置的速率限制

如果你的账户限制与默认值不同,可以配置速率限制:

| 参数                  | 默认值(已认证) | 默认值(未认证) | 说明                                         |
|----------------------------|-------------------------|---------------------------|-----------------------------------------------------|
| `max_requests_per_second`  | 10                      | 10                        | 每秒最大请求数(突发限制)。          |
| `max_requests_per_minute`  | 120                     | 30                        | 每分钟最大请求数(滚动窗口)。       |

:::info
关于速率限制的更多详情,参见
[BitMEX API 关于速率限制的文档](https://www.bitmex.com/app/restAPI#Limits)。
:::

:::warning
**取消广播器速率限制注意事项**

取消广播器(当 `canceller_pool_size > 1` 时)会将每个取消请求并行
分发给多个独立的 HTTP 客户端。每个客户端维护自己的速率限制器,这意味
着实际的请求速率会被池大小成倍放大。

**示例**:当 `canceller_pool_size=3`、`max_requests_per_second=10`
时,单次取消操作会消耗**3 次请求**(每个客户端一次),如果快速取消,
可能达到**每秒 30 次请求**。

由于 BitMEX 是**在账户级别**(而非按连接)强制执行速率限制的,该
广播器可能会使你超出交易所默认的每秒 10 次突发限制以及每分钟 120 次
滚动窗口限制。

**缓解方法**:按比例(除以 `canceller_pool_size`)降低
`max_requests_per_second` 和 `max_requests_per_minute`,或调整池
大小本身(参见 [取消广播器配置](#配置-2))。未来版本可能会支持在
池内共享速率限制器。
:::

### 速率限制响应头

BitMEX 通过响应头暴露当前配额:

- `x-ratelimit-limit`:当前窗口内允许的总请求数。
- `x-ratelimit-remaining`:在被限流之前的剩余请求数。
- `x-ratelimit-reset`:配额重置的 UNIX 时间戳。
- `retry-after`:收到 429 响应后需要等待的秒数。

## 提交广播器

BitMEX 执行客户端包含一个提交广播器,通过并行请求分发,为市价单和
限价单以目标价格被接受提供更高的保证,以更低的最小延迟换取重复提交的
风险。

### 概念

下单操作对时间非常敏感 —— 当策略决定进场时,任何延迟都可能导致错过
机会或不利的定价。提交广播器通过以下方式解决这一问题:

- **并行分发**:提交请求会同时广播给多个独立的 HTTP 客户端实例。
- **首个成功即短路**:第一个成功的响应获胜,将接受的延迟降至最低。
- **共享 client_order_id**:所有传输通道使用相同的
  `client_order_id`。BitMEX 会以 “duplicate clOrdID” 拒绝重复提交
  (被跟踪为预期拒绝)。
- **延迟与重复的权衡**:以接受潜在重复成交的风险(如果多个传输通道在
  被拒绝之前都成功了)为代价,换取更低的最小延迟和更高的接受保证。

这种架构通过在多条网络路径上并行化,降低了订单被接受的最小延迟。

### 用法

提交广播器需要主动启用,通过下单时的 `submit_tries` 参数控制。默认
情况下,订单通过单个 HTTP 客户端提交。启用广播:

```python
# 单次提交(默认行为)
self.submit_order(order)

# 广播给 2 个并行 HTTP 客户端以实现冗余
self.submit_order(order, params={"submit_tries": 2})

# 广播给 3 个并行 HTTP 客户端(建议的最大值)
self.submit_order(order, params={"submit_tries": 3})
```

**要点**:

- `submit_tries` 必须是正整数。
- 只有当 `submit_tries > 1` 时才会进行广播。默认提交通过单个 HTTP
  客户端进行。
- 如果 `submit_tries` 超过 `submitter_pool_size`,会被限制在池大小
  以内,并记录一条警告。
- 所有传输通道使用相同的 `client_order_id`;BitMEX 会将重复请求作为
  预期拒绝处理。

### 健康监控

广播器池中的每个 HTTP 客户端都维护健康指标:

- 成功的提交会将某个客户端标记为健康。
- 失败的请求会递增错误计数器。
- 后台健康检查会定期验证客户端的连接性。
- 状态下降的客户端仍会被跟踪并保留在池中,以维持容错能力。

广播器暴露包括总提交数、成功提交数、失败提交数以及预期拒绝数在内的
指标,用于运维监控和调试。

#### 跟踪的指标

| 指标                   | 类型   | 说明                                                                                                           |
|--------------------------|--------|-----------------------------------------------------------------------------------------------------------------------|
| `total_submits`          | `u64`  | 已发起的提交操作总数。                                                                          |
| `successful_submits`     | `u64`  | 成功收到 BitMEX 确认的提交操作数量。                                   |
| `failed_submits`         | `u64`  | 池中所有 HTTP 客户端都失败的提交操作数量(没有健康客户端或所有请求都失败)。    |
| `expected_rejects`       | `u64`  | 检测到的预期拒绝模式数量(例如,来自并行提交的重复 clOrdID)。                   |
| `healthy_clients`        | `usize`| 池中当前健康的 HTTP 客户端数量(通过近期健康检查的客户端)。                        |
| `total_clients`          | `usize`| 池中配置的 HTTP 客户端总数(`submitter_pool_size`)。                                          |

这些指标可以通过 `SubmitBroadcaster` 实例上的 `get_metrics()` 方法
以编程方式访问。

### 配置

提交广播器通过执行客户端配置进行配置:

| 选项                 | 默认值 | 说明                                                                               |
|------------------------|---------|-------------------------------------------------------------------------------------------|
| `submitter_pool_size`  | `None`  | HTTP 客户端池的大小。`None` 解析为 1(单客户端,无冗余)。 |
| `submitter_proxy_urls` | `None`  | 用于提交广播器路径多样性的可选代理 URL 列表。 |

**配置示例**:

```python
from nautilus_trader.adapters.bitmex.config import BitmexExecClientConfig

exec_config = BitmexExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    submitter_pool_size=3,  # 建议的冗余池大小
)
```

:::tip
对于没有更高速率限制的 HFT 策略,请权衡使用提交广播器的优势与可能
触及速率限制的风险,因为每个客户端拥有独立的速率限制配额。
默认的 `submitter_pool_size=None` 会禁用该广播器。建议的
`submitter_pool_size=3` 设置会将每个提交请求广播给 3 个并行 HTTP
客户端以实现容错,这会为每次提交操作消耗 3 倍的速率限制配额,但能
针对网络或交易所问题提供更高的保证。
:::

广播器会在执行客户端连接时自动启动,断开连接时自动停止。只有当
`submit_tries > 1` 时,提交操作才会通过广播器路由;默认提交直接使用
单个 HTTP 客户端。

## 取消广播器

BitMEX 执行客户端包含一个取消广播器,通过并行请求分发,提供容错的
订单取消能力。

### 概念

取消操作对时间非常敏感 —— 当策略决定取消某个订单时,任何延迟或失败
都可能导致意外成交、滑点或不希望出现的仓位敞口。取消广播器通过以下
方式解决这一问题:

- **并行分发**:取消请求会同时广播给多个独立的 HTTP 客户端实例。
- **首个成功即短路**:第一个成功的响应获胜,其余进行中的请求会立即
  中止。
- **容错**:如果某个 HTTP 客户端遇到网络问题、DNS 故障或连接超时,
  池中的其他客户端会继续处理。
- **幂等成功处理**:表明订单已经被取消的响应(例如 “orderID not
  found” 或类似的幂等状态)会被视为成功而非失败,避免不必要的错误
  传播。

这种架构确保单一网络路径故障或慢速连接不会阻塞取消操作,提升了实盘
交易中风险管理和仓位控制的可靠性。

### 健康监控

广播器池中的每个 HTTP 客户端都维护健康指标:

- 成功的取消操作会将某个客户端标记为健康。
- 失败的请求会递增错误计数器。
- 后台健康检查会定期验证客户端的连接性。
- 状态下降的客户端仍会被跟踪并保留在池中,以维持容错能力。

广播器暴露包括总取消数、成功取消数、失败取消数、预期拒绝数(已取消的
订单)以及幂等成功数在内的指标,用于运维监控和调试。

#### 跟踪的指标

| 指标                   | 类型   | 说明                                                                                                           |
|--------------------------|--------|-----------------------------------------------------------------------------------------------------------------------|
| `total_cancels`          | `u64`  | 已发起的取消操作总数(包括单个、批量和全部取消请求)。                        |
| `successful_cancels`     | `u64`  | 成功收到 BitMEX 确认的取消操作数量。                                   |
| `failed_cancels`         | `u64`  | 池中所有 HTTP 客户端都失败的取消操作数量(没有健康客户端或所有请求都失败)。    |
| `expected_rejects`       | `u64`  | 检测到的预期拒绝模式数量(例如,post-only 订单拒绝)。                                    |
| `idempotent_successes`   | `u64`  | 幂等成功响应的数量(订单已取消、订单未找到、因状态无法取消)。     |
| `healthy_clients`        | `usize`| 池中当前健康的 HTTP 客户端数量(通过近期健康检查的客户端)。                        |
| `total_clients`          | `usize`| 池中配置的 HTTP 客户端总数(`canceller_pool_size`)。                                          |

这些指标可以通过 `CancelBroadcaster` 实例上的 `get_metrics()` 方法
以编程方式访问。

### 配置

取消广播器通过执行客户端配置进行配置:

| 选项                 | 默认值 | 说明                                                                               |
|------------------------|---------|-------------------------------------------------------------------------------------------|
| `canceller_pool_size`  | `None`  | HTTP 客户端池的大小。`None` 解析为 1(单客户端,无冗余)。 |
| `canceller_proxy_urls` | `None`  | 用于取消广播器路径多样性的可选代理 URL 列表。 |

**配置示例**:

```python
from nautilus_trader.adapters.bitmex.config import BitmexExecClientConfig

exec_config = BitmexExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    canceller_pool_size=3,  # 建议的冗余池大小
)
```

:::tip
对于没有更高速率限制的 HFT 策略,请权衡使用取消广播器的优势与可能
触及速率限制的风险,因为每个客户端拥有独立的速率限制配额。
默认的 `canceller_pool_size=None` 会禁用该广播器。建议的
`canceller_pool_size=3` 设置会将每个取消请求广播给 3 个并行 HTTP
客户端以实现容错,这会为每次取消操作消耗 3 倍的速率限制配额,但能
针对网络或交易所问题提供更高的保证。
:::

广播器会在执行客户端连接时自动启动,断开连接时自动停止。所有取消操作
(`cancel_order`、`cancel_all_orders`、`batch_cancel_orders`)都会
自动通过该广播器路由,策略代码无需任何改动。

## 死人开关(Dead man's switch)

该适配器支持 BitMEX 的
[死人开关](https://www.bitmex.com/app/restAPI#OrdercancelAllAfter)
(`cancelAllAfter`),它作为应对连接故障的安全网,提供自动订单取消
功能。

### 工作原理

启用后,BitMEX 上会设置一个服务端计时器。如果该计时器过期而未被
刷新,BitMEX 会取消账户上**所有**未成交订单。适配器通过发送周期性的
心跳请求来保持计时器存活。如果适配器失去连接(网络故障、进程崩溃等),
心跳会停止,BitMEX 会在配置的超时之后取消这些订单。

流程:

1. **连接**时,适配器调用 `POST /api/v1/order/cancelAllAfter`,携带
   配置的超时时间(毫秒),以启动服务端计时器。
2. 一个后台任务会以 `timeout / 4`(最小 1 秒)的**刷新间隔**发送
   相同的请求,以在计时器过期前不断重置它。
3. **断开连接**时,适配器会等待后台心跳任务完全关闭,然后调用
   `timeout=0` 的 `cancelAllAfter` 以**解除**服务端计时器。

例如,超时设为 60 秒时,适配器每 15 秒发送一次心跳。如果连续四次
心跳失败(即连接丢失 60 秒),BitMEX 会取消所有未成交订单。

### 断开连接时的顺序

在断开连接期间解除死人开关需要谨慎排序。解除请求(`timeout=0`)应是
到达 BitMEX 的最后一次 `cancelAllAfter` 调用。如果某个正在进行中的
心跳请求在解除请求之后才被处理,它会重新启动服务端计时器,即使适配器
已经优雅地断开连接,订单仍可能在超时过期后被意外取消。

适配器在两种实现中都缓解了这一问题:

- **Rust**:心跳任务会被立即停止(中止 + 等待),因此断开连接不会
  停下来等待某个 sleep 或 HTTP 超时结束。解除请求会在该任务退出之后
  发送。
- **Python**:心跳任务会被取消并等待,确保该协程在解除请求发送之前
  完全展开退出。

在强制停止场景中(例如通过 `stop()` 关闭进程),心跳任务会在不解除
死人开关的情况下被中止。这是有意为之的,因为当进程意外退出时,服务端
计时器提供了所需的安全行为。

:::note
每次心跳消耗一个 REST 速率限制令牌。60 秒的超时大约会从每分钟
120 次的预算中消耗约 4 次请求。
:::

### 配置

通过在执行客户端配置上设置 `deadmans_switch_timeout_secs` 来启用
死人开关:

```python
from nautilus_trader.adapters.bitmex.config import BitmexExecClientConfig

exec_config = BitmexExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    deadmans_switch_timeout_secs=60,  # 连接丢失 60 秒后取消所有订单
)
```

启用后,适配器会在连接时记录:

```
Starting dead man's switch: timeout=60s, refresh_interval=15s
```

在断开连接时记录:

```
Disarming dead man's switch
```

:::tip
建议的起始超时值为 **60 秒**。更短的超时提供更快的保护,但对瞬时
网络抖动更敏感。更长的超时对短暂中断更宽容,但在真正的故障期间会让
订单暴露更长时间。
:::

:::warning
死人开关适用于账户上**所有**未成交订单,而不仅仅是适配器下达的订单。
如果其他系统在同一账户上下单,启用死人开关也会影响这些订单。
:::

## 配置

### API 凭证

BitMEX API 凭证可以直接在配置中提供,也可以通过环境变量提供:

- `BITMEX_API_KEY`:生产环境的 BitMEX API key。
- `BITMEX_API_SECRET`:生产环境的 BitMEX API secret。
- `BITMEX_TESTNET_API_KEY`:测试网的 BitMEX API key。
- `BITMEX_TESTNET_API_SECRET`:测试网的 BitMEX API secret。

生成 API key:

1. 登录你的 BitMEX 账户。
2. 导航到 Account & Security -> API Keys。
3. 创建一个具有适当权限的新 API key。
4. 对于测试网,使用 [testnet.bitmex.com](https://testnet.bitmex.com)。

:::note
**测试网 API 端点**:

- REST API:`https://testnet.bitmex.com/api/v1`
- WebSocket:`wss://ws.testnet.bitmex.com/realtime`

当配置了 `environment=BitmexEnvironment.TESTNET` 时,适配器会自动将
请求路由到正确的端点。
:::

### 数据客户端配置选项

BitMEX 数据客户端提供以下配置选项:

| 选项                             | 默认值   | 说明 |
|------------------------------------|-----------|-------------|
| `api_key`                          | `None`    | 可选的 API key;若为 `None`,从 `environment` 选定的环境加载。 |
| `api_secret`                       | `None`    | 可选的 API secret;若为 `None`,从 `environment` 选定的环境加载。 |
| `environment`                      | `None`    | 环境枚举(`MAINNET` 或 `TESTNET`)。 |
| `base_url_http`                    | `None`    | REST 基础 URL 的覆盖值(默认为生产环境)。 |
| `base_url_ws`                      | `None`    | WebSocket 基础 URL 的覆盖值(默认为生产环境)。 |
| `http_timeout_secs`                | `60`      | 应用于 HTTP 调用的请求超时时间。 |
| `max_retries`                      | `3`       | HTTP 调用的最大重试次数。 |
| `retry_delay_initial_ms`           | `1,000`   | 重试之间的初始退避延迟(毫秒)。 |
| `retry_delay_max_ms`               | `10,000`  | 重试之间的最大退避延迟(毫秒)。 |
| `recv_window_ms`                   | `10,000`  | 已签名请求的过期窗口(毫秒)。参见 [请求身份验证](#请求身份验证与过期)。 |
| `update_instruments_interval_mins` | `None`    | 标的目录刷新的间隔(分钟)。`None` 表示禁用定期刷新。 |
| `max_requests_per_second`          | `10`      | 适配器为 REST 调用强制执行的突发速率限制。 |
| `max_requests_per_minute`          | `120`     | 适配器为 REST 调用强制执行的滚动分钟速率限制。 |
| `proxy_url`                        | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `transport_backend`                | `Sockudo` | WebSocket 传输后端。 |

### 执行客户端配置选项

BitMEX 执行客户端提供以下配置选项:

| 选项                         | 默认值   | 说明 |
|--------------------------------|-----------|-------------|
| `api_key`                      | `None`    | 可选的 API key;若为 `None`,从 `environment` 选定的环境加载。 |
| `api_secret`                   | `None`    | 可选的 API secret;若为 `None`,从 `environment` 选定的环境加载。 |
| `environment`                  | `None`    | 环境枚举(`MAINNET` 或 `TESTNET`)。 |
| `base_url_http`                | `None`    | REST 基础 URL 的覆盖值(默认为生产环境)。 |
| `base_url_ws`                  | `None`    | WebSocket 基础 URL 的覆盖值(默认为生产环境)。 |
| `http_timeout_secs`            | `60`      | 应用于 HTTP 调用的请求超时时间。 |
| `max_retries`                  | `3`       | HTTP 调用的最大重试次数。 |
| `retry_delay_initial_ms`       | `1,000`   | 重试之间的初始退避延迟(毫秒)。 |
| `retry_delay_max_ms`           | `10,000`  | 重试之间的最大退避延迟(毫秒)。 |
| `recv_window_ms`               | `10,000`  | 已签名请求的过期窗口(毫秒)。参见 [请求身份验证](#请求身份验证与过期)。 |
| `max_requests_per_second`      | `10`      | 适配器为 REST 调用强制执行的突发速率限制。 |
| `max_requests_per_minute`      | `120`     | 适配器为 REST 调用强制执行的滚动分钟速率限制。 |
| `deadmans_switch_timeout_secs` | `None`    | 死人开关的超时时间(秒)。`None` 表示禁用。参见 [死人开关](#死人开关dead-mans-switch)。 |
| `canceller_pool_size`          | `None`    | 取消广播器池中的 HTTP 客户端数量。`None` 解析为 1。参见 [取消广播器](#取消广播器)。 |
| `submitter_pool_size`          | `None`    | 提交广播器池中的 HTTP 客户端数量。`None` 解析为 1。参见 [提交广播器](#提交广播器)。 |
| `proxy_url`                    | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `submitter_proxy_urls`         | `None`    | 用于提交广播器路径多样性的可选代理 URL 列表。 |
| `canceller_proxy_urls`         | `None`    | 用于取消广播器路径多样性的可选代理 URL 列表。 |
| `transport_backend`            | `Sockudo` | WebSocket 传输后端。 |

### 配置示例

典型的实盘交易 BitMEX 配置同时包含测试网和主网选项:

```python
from nautilus_trader.adapters.bitmex.config import BitmexDataClientConfig
from nautilus_trader.adapters.bitmex.config import BitmexExecClientConfig
from nautilus_trader.core.nautilus_pyo3 import BitmexEnvironment

# 使用环境变量(推荐)
testnet_data_config = BitmexDataClientConfig(
    environment=BitmexEnvironment.TESTNET,
)

# 使用显式凭证
mainnet_data_config = BitmexDataClientConfig(
    api_key="YOUR_API_KEY",  # 或使用 os.getenv("BITMEX_API_KEY")
    api_secret="YOUR_API_SECRET",  # 或使用 os.getenv("BITMEX_API_SECRET")
    environment=BitmexEnvironment.MAINNET,
)

mainnet_exec_config = BitmexExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    environment=BitmexEnvironment.MAINNET,
)
```

## 交易注意事项

### 条件单

BitMEX 执行适配器将 Nautilus 条件单列表映射为交易所原生的
`clOrdLinkID`/`contingencyType` 机制。当引擎提交
`ContingencyType::Oco` 或 `ContingencyType::Oto` 订单时,适配器:

- 在 BitMEX 上创建/维护关联的订单组,使子止损单和目标单继承父订单
  状态。
- 传播订单列表更新和取消,使条件单同伴与当前仓位状态保持一致。
- 呈现带有适当条件单元数据的执行报告,使策略层面的跟踪无需额外的手动
  接线。

适配器不会将 Nautilus 的 `ContingencyType::Ouo` 映射到 BitMEX。对于
带入场、止损和止盈的括号单流程,BitMEX 能够原生地将 OTO 入场关联到
条件单激活步骤,但止损腿与止盈腿之间的互相取消或更新行为需要在策略
层面模拟实现。在设计策略时,请继续使用 Nautilus 的
`OrderList`/`ContingencyType` 抽象,但不要依赖适配器为条件出场腿提供
OUO 配对。

### 合约规格

- **反向合约**:以加密货币结算(例如 XBTUSD 以 XBT 结算)。
- **正向合约**:以稳定币结算(例如 ETHUSDT 以 USDT 结算)。
- **合约规模**:因标的而异,请仔细检查规格。
- **最小变动单位**:因合约而异。

### 保证金要求

- 初始保证金要求因合约和市场条件而异。
- 维持保证金通常低于初始保证金。
- 当维持保证金要求未满足时会发生强平。
- BitMEX 支持逐仓保证金和全仓保证金两种模式。
- 可根据仓位规模调整风险限额,详见
  [交易所规则](https://www.bitmex.com/exchange-rules)。

### 手续费

- **Maker 手续费**:提供流动性通常为负(返佣)。
- **Taker 手续费**:吃单收取正手续费。
- **资金费率**:每 8 小时应用于永续合约。
- **预测市场手续费**:Maker 0.00%,Taker 0.25%(不允许杠杆)。

## 贡献

:::info
如需了解更多功能或为 BitMEX 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
