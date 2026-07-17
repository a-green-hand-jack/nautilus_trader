# dYdX

dYdX 是最大的加密货币衍生品去中心化交易所之一。该集成支持接入 dYdX v4
的实时市场数据与订单执行,dYdX v4 运行在自己的 Cosmos SDK 应用专属
区块链(dYdX Chain)上,使用 CometBFT 共识。订单簿和撮合引擎作为验证者
进程的一部分运行在链上。订单以 Cosmos 交易的形式通过 gRPC 提交,并在
每个区块结算。一个 Indexer 服务对外提供用于市场数据和账户状态的 REST
和 WebSocket API。

这是一个 Rust 实现、带 Python 绑定的适配器。

## 安装

:::note
无需额外的安装附加组件。该适配器以 Rust 实现,在构建期间会自动编译进
核心 `nautilus_trader` 包中。
:::

## 示例

可以在[这里](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/dydx/)找到实时示例脚本。

## 概览

该适配器以 Rust 实现,通过 PyO3 提供 Python 绑定。它直接集成了 dYdX 的
Indexer API(REST/WebSocket)用于市场数据,以及 gRPC 用于 Cosmos SDK
交易提交,无需依赖外部客户端库。

### 产品支持

| 产品类型      | 数据源 | 交易 | 备注                                  |
|-------------------|-----------|---------|------------------------------------------|
| 永续期货 | ✓         | ✓       | 所有永续合约均以 USDC 结算。       |
| 现货              | -         | -       | dYdX 在 Solana 上提供现货;该适配器不支持。 |
| 期权           | -         | -       | *dYdX 上不提供*。               |

:::note
该适配器仅支持永续期货。所有市场均以 USD 报价,以 USDC 结算。
:::

## 链架构

与暴露单一 REST/WebSocket API 的中心化交易所(CEX)不同,dYdX v4 运行在
自己专属的 **Cosmos SDK 应用专属区块链**之上。这意味着每笔交易都是一笔
需要经过共识的 Cosmos 交易,适配器必须管理序列号、gas,以及基于区块
高度的过期机制。

### 传输层

适配器通过三个独立的传输层进行通信:

```
                         ┌─────────────────────────────────────────────┐
                         │              dYdX v4 Chain                  │
                         │                                             │
 ┌──────────┐  HTTP      │   ┌──────────────────────┐                  │
 │          │───────────►│   │  Indexer (read-only) │                  │
 │          │  WebSocket │   │  - REST API          │                  │
 │ Nautilus │───────────►│   │  - Streaming API     │                  │
 │ Adapter  │            │   └──────────────────────┘                  │
 │          │  gRPC      │   ┌──────────────────────┐                  │
 │          │───────────►│   │  Validator (write)   │                  │
 └──────────┘            │   │  - Cosmos Tx submit  │                  │
                         │   │  - Sequence mgmt     │                  │
                         │   └──────────────────────┘                  │
                         └─────────────────────────────────────────────┘
```

| 层     | 目标    | 方向  | 用途                                              |
|-----------|-----------|------------|------------------------------------------------------|
| HTTP      | Indexer   | 只读  | 标的元数据、历史数据、账户状态。 |
| WebSocket | Indexer   | 只读  | 实时市场数据、订单/成交/仓位更新。  |
| gRPC      | Validator | 写入      | 下单、取消以及批量操作。 |

### 基于区块的结算

dYdX 区块大约每 **~0.5 秒**产生一个(实际时间会有变化)。适配器包含一个
`BlockTimeMonitor`,它跟踪从 WebSocket 数据流观察到的区块时间,以动态
估算 `seconds_per_block`。该估算值用于将基于时间的订单过期,转换为
短期订单所需的区块高度偏移量。

## 架构

dYdX v4 适配器包含多个组件,可以组合使用,也可以单独使用:

- `DydxHttpClient`:面向 Indexer REST API 查询的 Rust 实现 HTTP 客户端。
- `DydxWebSocketClient`:面向实时市场数据与账户更新的 Rust 实现
  WebSocket 客户端。
- `DydxGrpcClient`:面向 Cosmos SDK 交易提交的 Rust 实现 gRPC 客户端。
- `DydxInstrumentProvider`:标的解析与加载功能。
- `DydxDataClient`:市场数据流管理器。
- `DydxExecutionClient`:账户管理与交易执行网关。
- `DydxDataClientFactory`:dYdX v4 数据客户端工厂(供交易节点构建器
  使用)。
- `DydxExecutionClientFactory`:dYdX v4 执行客户端工厂(供交易节点
  构建器使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),不必直接与这些底层
组件打交道。
:::

:::warning[首次账户激活]
dYdX v4 交易账户(子账户 0)只有在钱包首次存款或交易之后才会创建。
在此之前,每次 gRPC/Indexer 查询都会返回 `NOT_FOUND`,因此
`DydxExecutionClient.connect()` 会失败。

在启动实时 `TradingNode` 之前,请从同一网络(主网/测试网)的同一钱包
存入任意正数量的 USDC 或其他受支持的抵押品。等交易完成最终确认后
(几个区块),重启节点,客户端就能顺利连接。
:::

## 故障排查

### `StatusCode.NOT_FOUND` 账户未找到

**原因:** 该钱包/子账户从未被注资,因此在链上尚不存在。

**修复方法:**

1. 向正确网络上的子账户 0 存入任意正数量的 USDC。
2. 等待最终确认(主网大约 30 秒,测试网更长)。
3. 重启 `TradingNode`;连接现在应该能够成功。

:::tip
在无人值守部署中,请将 `connect()` 调用包裹在一个指数退避循环中,
让客户端持续重试,直到存款出现为止。
:::

## 符号规则

dYdX 对永续期货合约使用特定的符号约定。

### 符号格式

格式:`{Base}-USD-PERP`

dYdX 上所有的永续合约都:

- 以 USD 报价
- 以 USDC 结算
- 在 Nautilus 中使用 `.DYDX` 场所后缀

示例:

- `BTC-USD-PERP.DYDX` - 比特币永续期货
- `ETH-USD-PERP.DYDX` - 以太坊永续期货
- `SOL-USD-PERP.DYDX` - Solana 永续期货

在策略中订阅:

```python
InstrumentId.from_str("BTC-USD-PERP.DYDX")
InstrumentId.from_str("ETH-USD-PERP.DYDX")
```

:::info
添加 `-PERP` 后缀是为了与其他适配器保持一致,并为未来扩展做准备。
虽然 dYdX 目前只支持永续合约,但这种命名约定为将来可能扩展到其他产品
类型留出了空间。
:::

## 订单能力

dYdX 支持带有完整订单类型和执行功能集的永续期货交易。Rust 适配器会
根据有效期和到期时间自动将订单分类为短期或长期,无需手动标记。

### 订单类型

| 订单类型             | 永续 | 备注                                              |
|------------------------|------------|------------------------------------------------------|
| `MARKET`               | ✓          | 以最优可得价格立即执行。       |
| `LIMIT`                | ✓          |                                                    |
| `STOP_MARKET`          | ✓          | 止损条件单,始终为长期订单。     |
| `STOP_LIMIT`           | ✓          | 条件单,始终为长期订单。               |
| `MARKET_IF_TOUCHED`    | ✓          | 止盈市价单,价格触及时触发。 |
| `LIMIT_IF_TOUCHED`     | ✓          | 止盈限价单,价格触及时触发。  |
| `TRAILING_STOP_MARKET` | -          | *不支持*。                                   |

### 执行指令

| 指令   | 永续 | 备注                                                                                |
|---------------|------------|--------------------------------------------------------------------------------------|
| `post_only`   | ✓          | 支持 LIMIT、STOP_LIMIT 和 LIMIT_IF_TOUCHED 订单。定价会吃单的 post-only 订单会被场所**先接受、随即立即取消**(而不是带原因拒绝)。 |
| `reduce_only` | ✓          | 适用于所有订单类型。dYdX 将其作为**成交时的钳制**来强制执行,而非下单时的前置条件:一个针对无仓位提交的 reduce-only 订单仍会正常成交。 |

### 有效期选项

| 有效期 | 永续 | 备注                                                                      |
|---------------|------------|----------------------------------------------------------------------------|
| `GTC`         | ✓          | 撤销前有效。                                                        |
| `GTD`         | ✓          | 指定日期前有效。场所将过期报告为一个取消事件;适配器在订单的 `expire_time` 已过后,将其映射为 `OrderExpired`(而非 `OrderCanceled`)。 |
| `IOC`         | ✓          | 立即成交或取消。                                                       |
| `FOK`         | -          | *已被 dYdX v4 废弃*。链会以 `code=48` 拒绝 FOK 订单;适配器在本地生成 `OrderDenied`,不会广播该交易。 |
| `DAY`         | -          | *不支持*。适配器在本地生成 `OrderDenied`,不会广播该交易。 |

### 高级订单功能

| 功能            | 永续 | 备注            |
|--------------------|------------|------------------|
| 订单修改 | -          | 不支持。dYdX 支持短期订单
[替换](https://docs.dydx.xyz/concepts/trading/limit-orderbook#replacements)
(相同 ID、更高的 GTB);尚未作为 `ModifyOrder` 暴露。 |
| 括号单/OCO 订单 | -          | *不支持*。 |
| 冰山订单     | -          | *不支持*。 |

### 批量操作

| 操作    | 永续 | 备注                                                                                                                  |
|--------------|------------|------------------------------------------------------------------------------------------------------------------------|
| 批量提交 | ✓          | 支持长期 `LIMIT` 订单。短期订单逐笔单独提交。                                   |
| 批量修改 | -          | *不支持*。                                                                                                       |
| 批量取消 | ✓          | 按类型拆分:短期订单使用 `MsgBatchCancel`(单次 gRPC 调用),长期订单使用批量的 `MsgCancelOrder`。 |

### 仓位管理

| 功能          | 永续 | 备注                         |
|------------------|------------|-------------------------------|
| 查询仓位  | ✓          | 实时仓位更新。   |
| 仓位模式    | -          | 仅净额模式(见下文)。     |
| 杠杆控制 | ✓          | 按市场设置杠杆。 |
| 保证金模式      | -          | 仅全仓保证金。            |

:::note
dYdX 在场所层面支持净额模式(每个标的一个仓位)。适配器目前仅在
`NETTING` 模式下运行。对冲模式支持计划在未来版本中提供。
:::

### 订单查询

| 功能              | 永续 | 备注                          |
|----------------------|------------|--------------------------------|
| 查询未成交订单    | ✓          | 列出所有活跃订单。        |
| 查询订单历史  | ✓          | 历史订单数据。         |
| 订单状态更新 | ✓          | 实时订单状态变化。 |
| 成交历史        | ✓          | 执行与成交报告。    |

### 条件单

| 功能            | 永续 | 备注                                            |
|--------------------|------------|--------------------------------------------------|
| 订单列表        | -          | *不支持*。                                 |
| OCO 订单         | -          | *不支持*。                                 |
| 括号单     | -          | *不支持*。                                 |
| 条件单 | ✓          | 止损、止盈市价单和止盈限价单。 |

### 权益分档限制

根据账户的权益分档(例如,标准分档为 10 个条件单),dYdX 对每个子账户
**同时挂起的条件单数量**设有固定上限。提交超出该上限的额外条件单会在
链上以 `code=10001` 被拒绝,并伴随形如
`Opening order would exceed equity tier limit of N` 的日志消息。在
提交更多条件单之前,请先取消现有的条件单,或将策略拆分到不同的子账户中。

### MIT 与 LIT 的相互转换

dYdX 协议使用单一的 `TAKE_PROFIT` 订单类型,包含一个价格
(`subticks`)和一个触发价格;它表现为触发时市价还是触发时限价,取决于
价格本身的隐含含义。适配器将 Nautilus 的 `MARKET_IF_TOUCHED` 提交为
一个止盈单,价格设置为 5% 的最坏情况穿价;而 `LIMIT_IF_TOUCHED` 提交
为一个以用户限价为价格的止盈单。这两种形式在 Indexer 中都返回为
`"type":"TAKE_PROFIT"`。在对账时,适配器会将解析出的限价与配置的穿价
容差进行比较,以恢复原始的 Nautilus 订单类型。如果价格处于预言机价格
的穿价带内,该订单会被对账为 `MARKET_IF_TOUCHED`;否则会被对账为
`LIMIT_IF_TOUCHED`。

### 强平与 ADL(减仓)处理

dYdX v4 应用两种顺序执行的风险机制:

1. **强平**在账户跌破维持保证金时运行。仓位会以与预言机价格有限的价差,
   与保险基金进行平仓。
2. **减仓(ADL)**在以下情形激活:强平无法完全恢复抵押率,或某次较大的
   预言机价格跳变使某个账户在单个步骤内变为负值。减仓会将抵押不足的
   仓位与随机选中的对冲账户进行平仓。

Indexer 通过每条 `Fill` 记录上的 `type` 字段(`DydxFillType`)暴露该
分类:

| `type`         | 含义                                               |
|----------------|-------------------------------------------------------|
| `LIMIT`        | 正常成交。                                          |
| `LIQUIDATED`   | 强平的 taker 方(抵押不足账户)。    |
| `LIQUIDATION`  | 强平的 maker 方(保险基金)。         |
| `DELEVERAGED`  | 减仓的 taker 方(ADL 平仓)。           |
| `OFFSETTING`   | 减仓的 maker 方(对冲账户)。    |

适配器会为每一笔强平/减仓成交记录一条警告日志,包含标的、方向、数量
和价格,然后通过正常路径发出 `FillReport`。
`DydxPerpetualPositionStatus::Liquidated` 会平掉对应的仓位报告。

上游参考资料:

- [强平机制](https://docs.dydx.xyz/concepts/trading/liquidations)
- [合约亏损机制(减仓)](https://help.dydx.trade/en/articles/166973-contract-loss-mechanisms-on-dydx-chain)

### 订单分类

dYdX 将每个订单分类为三种链上类别之一。Rust 适配器会根据有效期和到期
时间自动确定类别,无需手动配置。

| 类别        | 存放位置   | 到期方式            | 典型用途                                   |
|-----------------|-------------|-------------------|-----------------------------------------------|
| 短期      | 内存中   | 区块高度      | IOC/FOK,或 40 个区块内到期的订单。 |
| 长期       | 链上    | 时间戳(UTC)   | 到期时间超出短期窗口(约 20 秒,按约 0.5 秒/区块计算)的 GTC/GTD。 |
| 条件单     | 链上    | 时间戳(UTC)   | 止损和止盈触发单。           |

在协议层面,**所有 dYdX 订单都是限价单**。`MARKET` 订单类型是
Nautilus 提供的一种便利形式,适配器将其实现为一个定价远超订单簿的
激进 IOC 限价单。这意味着市价单遵循与限价单相同的
`Submitted > Accepted > Filled` 生命周期(成交之前应先收到一个
`OrderAccepted` 事件)。

关于短期与状态化(stateful)订单机制的完整协议层细节,参见
[dYdX 订单文档](https://docs.dydx.xyz/concepts/trading/orders)。

#### 短期订单

短期订单**仅存在于验证者内存中**,按区块高度过期(最多 40 个区块,
按约 0.5 秒/区块计算约为 20 秒)。它们是 dYdX 上最快的订单类型,因为
跳过了链上存储。

**特性**:

- **IOC 和 FOK 始终是短期订单**,无论其他参数如何
- 当到期时间落在动态短期窗口(`40 个区块 × seconds_per_block`)之内时,
  **GTD 订单**会被自动分类为短期
- 使用 Good-Til-Block(GTB)进行重放保护,而非 Cosmos SDK 序列号
- 可以**并发**广播(无信号量限制,使用缓存的序列号)
- 静默过期,不会产生取消事件
- 无法在单笔交易中批量处理(每笔交易一个 `MsgPlaceOrder`)

#### 长期订单

长期(状态化)订单**存储在链上**,按 UTC 时间戳过期。它们在过期或被
取消时会产生明确的取消事件。

**特性**:

- **GTC** 订单默认 90 天到期(协议上限为 95 天)
- **GTD** 订单使用用户提供的到期时间戳
- 需要正确的 Cosmos SDK 序列号管理(通过信号量序列化)
- 必须以递增序列号**串行**广播
- 可以在单笔交易中批量处理

#### 条件单

条件单(止损、止盈)**始终存储在链上**,由验证者根据价格条件触发。

**特性**:

- 始终使用基于时间戳的到期方式(GTC 默认 90 天,协议上限 95 天)
- 始终使用长期广播路径(通过信号量序列化)
- 包括 `StopMarket`、`StopLimit`、`TakeProfitMarket` 和
  `TakeProfitLimit`

#### 自动路由

适配器使用 `BlockTimeMonitor` 自动确定订单生命周期:

```
max_short_term_secs = SHORT_TERM_ORDER_MAXIMUM_LIFETIME (40) × seconds_per_block
```

如果订单距离到期的剩余时间在 `max_short_term_secs` 之内,则按短期
路由;否则按长期路由。无需手动配置。

#### MARKET 订单的实现方式

dYdX 没有原生的市价单类型。适配器将 `MARKET` 订单实现为定价如下的
激进 **IOC 限价单**:

- **买入**:`oracle_price × (1 + 0.05)`(高于预言机价格 5%)
- **卖出**:`oracle_price × (1 - 0.05)`(低于预言机价格 5%)

这个 5% 的滑点缓冲(`DEFAULT_MARKET_ORDER_SLIPPAGE = 0.05`)设定了
最坏情况下的价格(“穿价价格”)。由于订单是 IOC 的,未成交部分的滑点
不会被消耗。该缓冲刻意设置得较宽,以在波动条件下最大化成交概率。

### 客户端订单 ID 编码

dYdX 在链上要求使用 `u32` 类型的客户端 ID,而 Nautilus 使用基于字符串
的 `ClientOrderId` 值(例如 `O-20260220-031943-001-000-51`)。适配器
双向编码这些值,以便订单能够在重启后无需持久化状态即可完成对账。

对于标准的 O 格式(`O-YYYYMMDD-HHMMSS-TTT-SSS-CCC`),编码是确定性的:

| dYdX 字段        | 位数 | 内容                                           |
|-------------------|------|----------------------------------------------------|
| `client_id`       | 32   | `[trader:10][strategy:10][count:12]`(唯一键)。 |
| `client_metadata` | 32   | 自 2020-01-01 UTC 以来的秒数(时间戳)。          |

由于编码是确定性的,适配器可以将任何已对账的订单解码回其原始的
`ClientOrderId` 字符串,而无需数据库或映射文件。

非标准的 `ClientOrderId` 格式(自定义字符串、纯数字)会回退到基于
内存反向映射的顺序分配。这些 ID 只能在同一会话内解码。

#### 重启冲突预防

重启后,Nautilus 会根据已对账订单的数量重置内部订单计数器,该数量可能
低于上一个会话中使用过的最高计数器值(例如,如果某些订单已从 API
响应中过期)。这可能导致新订单产生与上一个会话某订单相同的
`client_id`,从而产生重复的场所订单 UUID。

适配器通过在对账过程中注册每个看到的 `client_id` 来防止这种情况。
如果新的 O 格式编码产生了一个已经被使用过的 `client_id`,编码器会记录
一条警告并回退到顺序分配。顺序分配同样会跳过任何已注册的值。

:::note
这一保护是自动的,不需要任何用户配置。警告日志
`[ENCODER] client_id ... collides with reconciled order` 仅供参考。
该订单仍会使用一个替代 ID 成功提交。
:::

## 广播与重试策略

### 短期广播

短期订单使用 Good-Til-Block(GTB)进行重放保护。链的 `ClobDecorator`
ante handler 会为短期消息跳过 Cosmos SDK 的序列号检查,因此:

- **无信号量限制**:广播完全并发
- **缓存的序列号**:无需递增或分配
- **不重试**:如果广播失败,立即失败
- 良性的取消错误会被视为成功(见下文)

### 长期广播

长期和条件单需要正确的 Cosmos SDK 序列号管理:

- **1 个许可的信号量**将所有长期广播串行化
- **指数退避**:500ms -> 1s -> 2s -> 4s(最多重试 5 次)
- **总预算 10 秒**,防止无限重试循环
- 遇到序列号不匹配时,会在重试前**从链上重新同步**序列号

### 序列号不匹配检测

| 错误码 | 来源               | 含义                                          |
|------------|----------------------|--------------------------------------------------|
| `code=32`  | Cosmos SDK           | 账户序列号不匹配                        |
| `code=104` | dYdX authenticator   | 签名验证失败(与序列号相关) |

两者都会通过 `RetryManager` 触发自动重新同步 + 重试。

### 良性的取消错误

在短期取消操作期间出现的以下错误会被视为**成功**:

| 错误码  | 含义                                                        |
|-------------|------------------------------------------------------------------|
| `code=19`   | 交易已存在于内存池缓存中(重复交易)            |
| `code=9`    | 取消在 memclob 中已存在,且 GoodTilBlock >= 当前值          |
| `code=3006` | 待取消的订单不存在(已成交/已过期/已取消) |

### 批量取消的分组

在取消多个订单时,适配器会按订单生命周期类型进行分组:

1. **短期订单**:通过 `broadcast_short_term()` 发送单个 `MsgBatchCancel`
2. **长期订单**:通过 `broadcast_with_retry()` 发送批量的
   `MsgCancelOrder` 消息

这确保了每一组都使用了适当的广播策略。

## 资金费率

dYdX 永续期货使用固定的 1 小时资金费率结算周期。适配器为 WebSocket 和
历史资金费率数据的所有 `FundingRateUpdate` 对象都设置 `interval` 为
`60`(分钟)。

## 速率限制

### gRPC 速率限制

适配器会对 gRPC 的 `broadcast_tx` 调用进行速率限制,以防止来自验证者
节点的 `ResourceExhausted`(429)错误。

| 设置                       | 默认值 | 说明                               |
|-------------------------------|---------|-------------------------------------------|
| `grpc_rate_limit_per_second`  | `4`     | 每秒最大 gRPC 广播请求数。设为 `None` 表示禁用。 |

### 提供商限制

已知公开 gRPC 提供商的速率限制:

| 提供商   | 限制              | 备注           |
|------------|--------------------|-----------------|
| Polkachu   | 300 请求/分钟(约 5/秒) |                 |
| KingNodes  | 250 请求/分钟(约 4.2/秒) |               |
| AutoStake  | 4 请求/秒            |                 |

默认值 4 请求/秒较为保守,适用于所有公开提供商。

### 多 gRPC URL 回退

执行配置的 `grpc_endpoint` 字段会覆盖主 gRPC 端点。这是一个
config-struct 字段,不是 Python `DydxExecClientConfig` 构造函数的参数。

当 `grpc_endpoint` 未设置时,适配器会为所选网络使用默认的公共节点,
并在公共验证者列表中内置回退机制。目前 Python 配置尚未暴露通过用户
配置进行显式多 URL 回退的能力。

## 价格与数量的量化

dYdX 对价格和数量使用基于整数的量化。适配器通过 `OrderMessageBuilder`
自动处理所有转换,但了解这些参数有助于调试。

### 市场参数

| 参数                      | 说明                                              |
|--------------------------------|----------------------------------------------------------|
| `atomic_resolution`            | 将人类可读的数量转换为量子(quantums)的指数  |
| `quantum_conversion_exponent`  | 将量子转换为代币的指数               |
| `step_base_quantums`           | 以量子表示的最小下单量步长                      |
| `subticks_per_tick`            | 每个 tick 内的价格粒度                     |

### 市价单定价

市价单使用带 5% 滑点缓冲的预言机价格(“穿价价格”):

- **买入**:`oracle_price × 1.05`
- **卖出**:`oracle_price × 0.95`

预言机价格从 Indexer 缓存,并定期刷新。

### 自动处理

所有价格和数量的量化都由 `OrderMessageBuilder` 自动处理。通过
Nautilus 下单时无需手动转换。

## 数据订阅

v4 适配器支持以下数据订阅:

| 数据类型            | 订阅 | 历史请求 | 备注                                           |
|----------------------|--------------|--------------------|-------------------------------------------------|
| 成交 tick          | ✓            | ✓                  |                                                 |
| 报价 tick          | ✓            | -                  | 从订单簿最优买卖价合成。        |
| 订单簿增量    | ✓            | -                  | 仅限 L2 深度。                                  |
| 订单簿快照 | -            | ✓                  | 通过 HTTP 请求获取一次性快照。             |
| K 线                 | ✓            | ✓                  | 支持的精度见下文。                |
| 标记价格          | ✓            | -                  | 通过 markets 频道。                            |
| 指数价格         | ✓            | -                  | 通过 markets 频道。                            |
| 资金费率        | ✓            | ✓                  | 实时通过 markets 频道,历史通过 HTTP。 |
| 标的状态    | ✓            | -                  | 通过 markets 频道。                            |

### 支持的 K 线精度

| 精度 | dYdX K 线 |
|------------|-------------|
| 1-MINUTE   | `1MIN`      |
| 5-MINUTE   | `5MINS`     |
| 15-MINUTE  | `15MINS`    |
| 30-MINUTE  | `30MINS`    |
| 1-HOUR     | `1HOUR`     |
| 4-HOUR     | `4HOURS`    |
| 1-DAY      | `1DAY`      |

## 子账户

dYdX 支持每个钱包地址拥有多个子账户,允许在单个钱包内隔离交易策略和
风险管理。

### 关键概念

- 每个钱包地址可以拥有多个编号子账户(0、1、2、...、127)。
- 子账户 0 是**默认**账户,首次存款时会自动创建。
- 每个子账户维护自己的:
  - 仓位
  - 未成交订单
  - 抵押品余额
  - 保证金要求

### 配置

在执行客户端配置中指定子账户编号:

```python
config = TradingNodeConfig(
    exec_clients={
        "DYDX": DydxExecClientConfig(
            subaccount_number=0,  # 默认子账户
        ),
    },
)
```

:::note
大多数用户会使用子账户 `0`(默认账户)。高级用户可以为不同的子账户
配置多个执行客户端,以实现策略隔离或风险隔离。
:::

## 测试网设置

dYdX 测试网(`dydx-testnet-4`)是主网的完整副本,用于在不承担真实资金
风险的情况下测试策略。当 `network=DydxNetwork.TESTNET` 时,所有默认
测试网端点会自动解析。

### 1. 创建测试网钱包

**方式 A:通过 dYdX 测试网 Web 应用(最简单)**

1. 打开 [v4.testnet.dydx.exchange](https://v4.testnet.dydx.exchange)
2. 使用 MetaMask、Keplr、Phantom 或 WalletConnect 连接
3. 系统会自动生成一个 dYdX 账户
4. 导出你的助记词:点击右上角地址,选择 “Export secret phrase”

**方式 B:使用现有的 secp256k1 私钥**

任何 32 字节的十六进制编码 secp256k1 私钥都可以使用。适配器会使用
Cosmos bech32 编码自动从该密钥派生出 `dydx1...` 地址。

### 2. 为测试网账户注资

必须先为某个子账户注资,适配器才能连接(参见
[首次账户激活](#架构))。

**通过测试网 Web 应用:**

在 [v4.testnet.dydx.exchange](https://v4.testnet.dydx.exchange) 上
点击存款/充值按钮,自动获取测试网 USDC。

**直接通过水龙头 API:**

```bash
# 为子账户 0 注资 2000 USDC
curl -X POST https://faucet.v4testnet.dydx.exchange/faucet/tokens \
  -H "Content-Type: application/json" \
  -d '{"address": "dydx1...", "subaccountNumber": 0, "amount": 2000}'

# 注资原生代币(用于 gas 费用)
curl -X POST https://faucet.v4testnet.dydx.exchange/faucet/native-token \
  -H "Content-Type: application/json" \
  -d '{"address": "dydx1..."}'
```

### 3. 设置环境变量

```bash
export DYDX_TESTNET_WALLET_ADDRESS="dydx1..."
export DYDX_TESTNET_PRIVATE_KEY="0x..."  # 十六进制编码,0x 前缀可选
```

### 4. 配置交易节点

在数据和执行客户端上都设置 `network=DydxNetwork.TESTNET`:

```python
from nautilus_trader.adapters.dydx import DydxNetwork

config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        DYDX: DydxDataClientConfig(
            wallet_address=None,  # 回退到 DYDX_TESTNET_WALLET_ADDRESS 环境变量
            instrument_provider=InstrumentProviderConfig(load_all=True),
            network=DydxNetwork.TESTNET,
        ),
    },
    exec_clients={
        DYDX: DydxExecClientConfig(
            wallet_address=None,  # 回退到 DYDX_TESTNET_WALLET_ADDRESS 环境变量
            private_key=None,     # 回退到 DYDX_TESTNET_PRIVATE_KEY 环境变量
            subaccount_number=0,
            instrument_provider=InstrumentProviderConfig(load_all=True),
            network=DydxNetwork.TESTNET,
        ),
    },
)
```

### 测试网端点

默认测试网端点会自动使用。如有需要,可通过执行配置上的
`http_endpoint`、`ws_endpoint` 或 `grpc_endpoint` config-struct 字段
覆盖(这些不是 Python 构造函数参数)。

| 服务   | 默认 URL                                          |
|-----------|------------------------------------------------------|
| HTTP      | `https://indexer.v4testnet.dydx.exchange`            |
| WebSocket | `wss://indexer.v4testnet.dydx.exchange/v4/ws`        |
| gRPC      | `https://test-dydx-grpc.kingnodes.com:443`(主用) |
| 水龙头    | `https://faucet.v4testnet.dydx.exchange`             |
| Web 应用   | `https://v4.testnet.dydx.exchange`                   |

### 主网端点

默认主网端点会自动使用。如有需要,可通过执行配置上的
`http_endpoint`、`ws_endpoint` 或 `grpc_endpoint` config-struct 字段
覆盖(这些不是 Python 构造函数参数)。

| 服务   | 默认 URL                                         |
|-----------|-----------------------------------------------------|
| HTTP      | `https://indexer.dydx.trade`                        |
| WebSocket | `wss://indexer.dydx.trade/v4/ws`                    |
| gRPC      | `https://dydx-ops-grpc.kingnodes.com:443`(主用) |

## 配置

通过交易节点配置来配置 dYdX 适配器。执行客户端支持凭证的环境变量
回退。数据客户端使用公开端点,不需要钱包凭证。

### 数据客户端配置选项

| 选项                    | 默认值   | 说明                                                                                 |
|---------------------------|-----------|-----------------------------------------------------------------------------------------------|
| `wallet_address`          | `None`    | 传统 Python 配置字段。公开数据客户端不使用钱包凭证。         |
| `network`                 | `None`    | `DydxNetwork.MAINNET` 或 `DydxNetwork.TESTNET`。                                             |
| `bars_timestamp_on_close` | `True`    | K 线的 `ts_event` 是否应为 K 线收盘时间。设为 `False` 使用场所原生的开盘时间。  |
| `base_url_http`           | `None`    | HTTP API 端点覆盖值。`None` 表示为所选网络选择默认值。            |
| `base_url_ws`             | `None`    | WebSocket 端点覆盖值。`None` 表示为所选网络选择默认值。           |
| `proxy_url`               | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。                                       |
| `max_retries`             | `3`       | REST / WebSocket 恢复的最大重试次数。                                       |
| `retry_delay_initial_ms`  | `100`     | 重试之间的初始延迟(毫秒)。                                               |
| `retry_delay_max_ms`      | `5,000`   | 重试之间的最大延迟(毫秒)。                                               |
| `transport_backend`       | `Sockudo` | WebSocket 传输后端。                                                                |

`base_url_http` 和 `base_url_ws` 是 config-struct 字段,不是 Python
`DydxDataClientConfig` 构造函数的参数。

### 执行客户端配置选项

| 选项                         | 默认值   | 说明                                                                                        |
|--------------------------------|-----------|----------------------------------------------------------------------------------------------------|
| `wallet_address`               | `None`    | dYdX 钱包地址。回退到 `DYDX_WALLET_ADDRESS` / `DYDX_TESTNET_WALLET_ADDRESS` 环境变量。  |
| `subaccount_number`            | `0`       | 子账户编号(0-127)。子账户 0 是默认值。                                            |
| `private_key`                  | `None`    | 用于签名的十六进制编码私钥。回退到 `DYDX_PRIVATE_KEY` / `DYDX_TESTNET_PRIVATE_KEY`。|
| `authenticator_ids`            | `None`    | 用于权限化 key 交易(机构场景)的 authenticator ID 列表。                     |
| `network`                      | `None`    | `DydxNetwork.MAINNET` 或 `DydxNetwork.TESTNET`。                                                    |
| `http_endpoint`                | `None`    | HTTP 客户端的自定义端点覆盖值。`None` 表示为所选网络选择默认值。         |
| `ws_endpoint`                  | `None`    | WebSocket 客户端的自定义端点覆盖值。`None` 表示为所选网络选择默认值。    |
| `grpc_endpoint`                | `None`    | gRPC 客户端的自定义端点覆盖值。`None` 表示为所选网络选择默认值。         |
| `proxy_url`                    | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。                                              |
| `max_retries`                  | `3`       | 提交/取消/修改订单操作的最大重试次数。                                  |
| `retry_delay_initial_ms`       | `1,000`   | 重试之间的初始延迟(毫秒)。                                                      |
| `retry_delay_max_ms`           | `10,000`  | 重试之间的最大延迟(毫秒)。                                                      |
| `grpc_rate_limit_per_second`   | `4`       | 每秒最大 gRPC 请求数。设为 `None` 表示禁用。                                        |
| `transport_backend`            | `Sockudo` | WebSocket 传输后端。                                                                       |

`http_endpoint`、`ws_endpoint` 和 `grpc_endpoint` 是 config-struct
字段,不是 Python `DydxExecClientConfig` 构造函数的参数。

### 基本设置

配置一个包含 dYdX 数据和执行客户端的实时 `TradingNode`:

```python
from nautilus_trader.adapters.dydx import DydxDataClientConfig
from nautilus_trader.adapters.dydx import DydxExecClientConfig
from nautilus_trader.adapters.dydx import DydxNetwork
from nautilus_trader.adapters.dydx.constants import DYDX
from nautilus_trader.config import InstrumentProviderConfig
from nautilus_trader.config import TradingNodeConfig

config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        DYDX: DydxDataClientConfig(
            wallet_address=None,  # 回退到环境变量
            instrument_provider=InstrumentProviderConfig(load_all=True),
            network=DydxNetwork.MAINNET,
        ),
    },
    exec_clients={
        DYDX: DydxExecClientConfig(
            wallet_address=None,  # 回退到环境变量
            private_key=None,     # 回退到环境变量
            subaccount_number=0,
            instrument_provider=InstrumentProviderConfig(load_all=True),
            network=DydxNetwork.MAINNET,
        ),
    },
)
```

然后,创建一个 `TradingNode` 并注册客户端工厂:

```python
from nautilus_trader.adapters.dydx import DydxDataClientFactory
from nautilus_trader.adapters.dydx import DydxExecutionClientFactory
from nautilus_trader.adapters.dydx.constants import DYDX
from nautilus_trader.live.node import TradingNode

node = TradingNode(config=config)

node.add_data_client_factory(DYDX, DydxDataClientFactory)
node.add_exec_client_factory(DYDX, DydxExecutionClientFactory)

node.build()
```

### API 凭证

凭证可以直接通过 Python 配置传入(`wallet_address`、`private_key`),
也可以根据所配置的 `network` 自动从环境变量解析。

#### 环境变量

| 变量                        | 网络  | 说明                                    |
|---------------------------------|----------|--------------------------------------------------|
| `DYDX_WALLET_ADDRESS`           | 主网  | Bech32 编码的钱包地址(`dydx1...`)。    |
| `DYDX_PRIVATE_KEY`              | 主网  | 用于签名的十六进制编码 secp256k1 私钥。 |
| `DYDX_TESTNET_WALLET_ADDRESS`   | 测试网  | 测试网钱包地址(`dydx1...`)。           |
| `DYDX_TESTNET_PRIVATE_KEY`      | 测试网  | 测试网私钥。                           |

#### 解析优先级

1. Python 配置中传入的值(如果非空)
2. 由 `network` 选择的环境变量

### 权限化 key 交易

#### 什么是 API Trading Keys

API Trading Keys 允许你将交易委托给一个独立的签名密钥,而无需分享
主钱包的助记词。该 API key 可以使用所有者全仓保证金账户中的全部可用
保证金进行交易,但不能提现或转移资产。

#### 创建 API key

1. 在 dYdX Web 应用中,导航到 **More > API Trading Keys**
2. 点击 **Generate New API Key**
3. 保存 **API Wallet Address** 和 **Private Key**(仅显示一次,
   dYdX 不会存储)
4. 点击 **Authorize API Key**(这会将该密钥作为 authenticator 注册到
   链上)
5. 该密钥现已激活,可用于交易

创建和管理 API key 的完整详情,参见
[dYdX API Trading Keys 指南](https://docs.dydx.xyz/concepts/trading/api-trading-keys)。

#### 适配器配置

有两种方式为使用 API Trading Key 配置适配器:

**自动解析(推荐):** 将该 API key 的私钥设为 `DYDX_PRIVATE_KEY`,
将所有者的钱包地址设为 `DYDX_WALLET_ADDRESS`。适配器会在连接期间检测
到不匹配,并自动向链上查询匹配的 authenticator ID。无需手动配置 ID。

```python
config = DydxExecClientConfig(
    wallet_address="dydx1owner...",   # 所有者账户(持有保证金)
    private_key="0xapikey...",         # API Trading Key 私钥
    # authenticator_ids 自动解析
)
```

**手动覆盖:** 如果你已知 authenticator ID(例如,来自 dYdX
TypeScript 客户端),可以直接传入以跳过自动解析:

```python
config = DydxExecClientConfig(
    wallet_address="dydx1owner...",
    private_key="0xapikey...",
    authenticator_ids=[1, 2],  # 跳过自动解析
)
```

:::note
API Trading Keys 仅适用于**全仓保证金**账户和全仓市场。不支持逐仓
保证金。
:::

## 订单簿

根据订阅方式,订单簿可以维护为完整深度,也可以只维护最优买卖价报价。
该场所不直接提供报价。相反,适配器订阅订单簿增量,并在最优买卖价的
价格或数量发生变化时,为 `DataEngine` 合成报价。仅支持 L2(MBP)订单
簿类型。

## 贡献

:::info
如需了解更多功能或为 dYdX 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
