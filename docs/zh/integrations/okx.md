# OKX

OKX 成立于 2017 年,是一个提供现货、保证金、永续互换、期货、期权、
价差以及事件合约交易的加密货币交易所。该集成支持接入 OKX 的实时市场
数据与订单执行。

## 概览

该适配器以 Rust 编写,并提供可选的 Python 绑定以支持 Python
工作流。它不需要外部 OKX 客户端库。核心组件被编译为静态库,并在构建
期间自动链接。

## 示例

实时示例脚本位于
[examples/live/okx](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/okx/)。

### 产品支持

| 产品         | 标的来源            | 数据 | 执行 | 备注                                     |
|-----------------|------------------------------|------|------|--------------------------------------------|
| 现货            | `public/instruments`         | 是  | 是  | 现货交易对。                       |
| 保证金          | `public/instruments`         | 是  | 是  | 带保证金或杠杆的现货标的。 |
| 永续互换 | `public/instruments`         | 是  | 是  | 正向和反向合约。             |
| 期货         | `public/instruments`         | 是  | 是  | 到期期货合约。                  |
| 期权         | `public/instruments`         | 是  | 是  | 限价类订单执行。              |
| 价差         | `sprd/spreads`               | 是  | 是  | 快照、报价、成交,通过 business WS 提供。 |
| 事件合约 | `event-contract/*` 端点 | 是  | 是  | 解析为 Nautilus `BinaryOption`。        |

相关 OKX 文档:

- [获取标的](https://www.okx.com/docs-v5/en/#public-data-rest-api-get-instruments)。
- [获取限价](https://www.okx.com/docs-v5/en/#public-data-rest-api-get-limit-price)。
- [获取价差(公开)](https://www.okx.com/docs-v5/en/#spread-trading-rest-api-get-spreads-public)。
- [价差交易下单](https://www.okx.com/docs-v5/en/#spread-trading-rest-api-place-order)。
- [事件合约系列](https://www.okx.com/docs-v5/en/#public-data-rest-api-get-series)。

:::note
**期权支持**:该适配器支持期权市场数据、场所提供的 Greeks
(`subscribe_option_greeks`),以及期权标的的订单执行。详情参见下方的
[期权交易](#期权交易)章节,订阅模式参见
[期权](../concepts/options.md)指南。
:::

:::info
**标的乘数**:对于衍生品(`SWAP`、`FUTURES`、`OPTION`),标的乘数
计算为 OKX 的 `ctMult` 和 `ctVal` 字段的乘积。这使仓位规模计算与
OKX 的合约规模和价值保持一致。
:::

:::info
**价格限制**:OKX 在 `public/instruments` 上为现货、保证金、互换和
期货标的提供 `initPxLmtPct`、`floatPxLmtPct` 和 `maxPxLmtPct`。
适配器将非空值保留在标的的 `info` 字段中,分别为
`okx_init_px_lmt_pct`、`okx_float_px_lmt_pct` 和
`okx_max_px_lmt_pct`。这些字段描述的是交易所波动带百分比,因此不会
被解析为静态的 Nautilus `min_price` 或 `max_price` 值。

当你需要从 OKX 的 `GET /api/v5/public/price-limit` 端点获取当前计算
出的买卖限价时,使用 `OKXHttpClient.request_price_limit(instrument_id)`。
OKX 文档中,期权和事件合约的百分比字段为空;适配器不会修改这些标的的
`info`。
:::

:::note
OKX 的金融产品端点,如 `/api/v5/finance/okusd/*`,不在 OKX 交易适配器
的范围内。
:::

OKX 适配器包含多个组件,可以单独使用,也可以组合使用:

- `OKXHttpClient`:底层 HTTP API 连接。
- `OKXWebSocketClient`:底层 WebSocket API 连接。
- `OKXInstrumentProvider`:标的解析与加载功能。
- `OKXDataClient`:市场数据流管理器。
- `OKXExecutionClient`:账户管理与交易执行网关。
- `OKXLiveDataClientFactory`:OKX 数据客户端工厂(供交易节点构建器
  使用)。
- `OKXLiveExecClientFactory`:OKX 执行客户端工厂(供交易节点构建器
  使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),不必直接与这些底层
组件打交道。
:::

## 符号规则

OKX 对不同标的类型使用特定的符号约定。在 Nautilus 中引用标的时,请
添加 `.OKX` 后缀,例如 `BTC-USDT.OKX`。

### 按标的类型划分的符号格式

#### SPOT(现货)

格式:`{BaseCurrency}-{QuoteCurrency}`

示例:

- `BTC-USDT` - 比特币兑 USDT(泰达币)
- `BTC-USDC` - 比特币兑 USDC
- `ETH-USDT` - 以太坊兑 USDT
- `SOL-USDT` - Solana 兑 USDT

在策略中订阅现货比特币 USD:

```python
InstrumentId.from_str("BTC-USDT.OKX")  # 以 USDT 计价的现货
InstrumentId.from_str("BTC-USDC.OKX")  # 以 USDC 计价的现货
```

#### SWAP(永续互换)

格式:`{BaseCurrency}-{QuoteCurrency}-SWAP`

示例:

- `BTC-USDT-SWAP` - 比特币永续互换(正向,以 USDT 计保证金)
- `BTC-USD-SWAP` - 比特币永续互换(反向,以币计保证金)
- `ETH-USDT-SWAP` - 以太坊永续互换(正向)
- `ETH-USD-SWAP` - 以太坊永续互换(反向)

正向合约与反向合约:

- **正向**(以 USDT 计保证金):使用如 USDT 等稳定币作为保证金。
- **反向**(以币计保证金):使用基础加密货币作为保证金。

#### FUTURES(到期期货)

格式:`{BaseCurrency}-{QuoteCurrency}-{YYMMDD}`

示例:

- `BTC-USD-251226` - 2025 年 12 月 26 日到期的比特币期货
- `ETH-USD-251226` - 2025 年 12 月 26 日到期的以太坊期货
- `BTC-USD-250328` - 2025 年 3 月 28 日到期的比特币期货

注意:期货通常是反向合约(以币计保证金)。

#### SPREADS(价差)

格式:`{Leg1InstrumentId}_{Leg2InstrumentId}`

示例:

- `BTC-USDT_BTC-USDT-SWAP` - BTC-USDT 现货与 BTC-USDT 永续互换之间的
  价差
- `ETH-USD-SWAP_ETH-USD-231229` - ETH-USD 永续互换与到期期货之间的
  价差

在数据客户端上设置 `load_spreads=True`,以从 OKX 的
[获取价差(公开)](https://www.okx.com/docs-v5/en/#spread-trading-rest-api-get-spreads-public)
端点加载实时的 OKX 价差标的。适配器将每个 OKX `sprdId` 映射为一个带
`.OKX` 场所后缀的 Nautilus 价差标的 ID。

关于价差标的的简要说明:

- 价差市场数据流在 OKX business WebSocket 上提供:报价
  (`sprd-bbo-tbt`)、成交(`sprd-public-trades`),以及 5 档订单簿
  快照(`sprd-books5`)。价差没有增量订单簿频道,因此每次
  `sprd-books5` 更新都是通过订单簿订阅传递的完整快照(标记为快照,
  而非增量 L2 增量)。
- 目前 OKX 实时价差发现返回现货、互换和期货腿的组合。
- 如果 OKX 通过同一个价差端点暴露期权腿组合定义,该解析器可以表示
  它们。
- OKX 期权 RFQ 和大宗交易工作流独立于 Nitro 价差订单簿 API,不通过该
  价差路径路由。

#### OPTIONS(期权)

格式:`{BaseCurrency}-{QuoteCurrency}-{YYMMDD}-{Strike}-{Type}`

示例:

- `BTC-USD-251226-100000-C` - 比特币看涨期权,行权价 $100,000,
  2025 年 12 月 26 日到期
- `BTC-USD-251226-100000-P` - 比特币看跌期权,行权价 $100,000,
  2025 年 12 月 26 日到期
- `ETH-USD-251226-4000-C` - 以太坊看涨期权,行权价 $4,000,2025 年
  12 月 26 日到期

其中:

- `C` = 看涨期权
- `P` = 看跌期权

#### EVENTS(事件合约)

OKX 事件合约标的 ID 使用 OKX 标的 API 返回的市场 ID。适配器将这些
市场表示为 Nautilus 的 `BinaryOption` 标的。

示例:

- `BTC-ABOVE-DAILY-260224-1600-65000` - 属于 `BTC-ABOVE-DAILY` 系列
  的事件合约市场。

### 常见问题

**问:如何订阅现货比特币 USD?**
答:以 USDT 计保证金的现货使用 `BTC-USDT.OKX`,以 USDC 计保证金的
现货使用 `BTC-USDC.OKX`。

**问:BTC-USDT-SWAP 和 BTC-USD-SWAP 有什么区别?**
答:`BTC-USDT-SWAP` 是正向永续合约(以 USDT 计保证金),而
`BTC-USD-SWAP` 是反向永续合约(以 BTC 计保证金)。

**问:如何知道该使用哪种合约类型?**
答:检查配置中的 `contract_types` 参数:

- 正向合约:`OKXContractType.LINEAR`。
- 反向合约:`OKXContractType.INVERSE`。

**问:如何加载事件合约?**
答:使用 `OKXInstrumentType.EVENTS`。要限定加载范围,通过
`instrument_families` 传入 OKX 的 `seriesId` 值,例如
`BTC-ABOVE-DAILY`。

## 订单能力

以下是 OKX 上正向永续互换产品支持的订单类型、执行指令和有效期选项。

### WebSocket 订单标识

OKX WebSocket 订单操作使用 `instIdCode`(一个数字标的标识符),而非
字符串形式的 `instId` 参数。适配器从启动期间获取的标的定义中解析
`instIdCode` 值,并在会话生命周期内缓存它们。如果标的缓存为空(例如
由于启动引导失败),下单会以明确的错误失败。

### 客户端订单 ID 要求

:::note
OKX 对客户端订单 ID 有特定要求:

- **不允许使用连字符**:OKX 不接受客户端订单 ID 中包含连字符
  (`-`)。
- 最大长度:32 个字符。
- 允许的字符:仅限字母和数字。

配置策略时,请确保设置:

```python
use_hyphens_in_client_order_ids=False
```

:::

### 订单类型

| 订单类型             | 正向永续互换 | 备注                                                         |
|------------------------|-----------------------|-----------------------------------------------------------------|
| `MARKET`               | ✓                     | 以市场价立即执行。支持以计价货币计量数量。 |
| `MARKET_TO_LIMIT`      | ✓                     | 市价单转换为 IOC 限价单。                          |
| `LIMIT`                | ✓                     | 以指定价格或更优价格执行。                       |
| `STOP_MARKET`          | ✓                     | 通过 OKX 算法单实现的条件市价单。             |
| `STOP_LIMIT`           | ✓                     | 通过 OKX 算法单实现的条件限价单。              |
| `MARKET_IF_TOUCHED`    | ✓                     | 通过 OKX 算法单实现的条件市价单。             |
| `LIMIT_IF_TOUCHED`     | ✓                     | 通过 OKX 算法单实现的条件限价单。              |
| `TRAILING_STOP_MARKET` | ✓                     | 通过 OKX 高级算法单实现的跟踪止损市价单。   |

:::info
**条件单**:`STOP_MARKET`、`STOP_LIMIT`、`MARKET_IF_TOUCHED`、
`LIMIT_IF_TOUCHED` 和 `TRAILING_STOP_MARKET` 都使用 OKX 算法单。
`TRAILING_STOP_MARKET` 路径使用 OKX 的高级算法单 API
(`move_order_stop`),取消时需要使用 `cancel-advance-algos` 端点。
:::

### 价差订单

OKX 价差标的使用独立的价差交易订单簿和 API 系列。执行客户端目前
通过 HTTP `/api/v5/sprd/*` 端点,按价差标的 ID(例如
`ETH-USD-SWAP_ETH-USD-231229.OKX`)路由价差订单。

适配器使用 OKX 的价差 REST 端点进行提交、取消、批量取消、订单状态
以及成交报告。它订阅 OKX business WebSocket 的
[`sprd-orders` 频道](https://www.okx.com/docs-v5/en/#spread-trading-websocket-private-channel-order-channel)
以获取实时的价差订单更新。

OKX 的 `sprd-orders` WebSocket 更新不包含手续费字段。从该频道发出的
实时价差成交报告使用零手续费;来自 REST
[`sprd/trades` 端点](https://www.okx.com/docs-v5/en/#spread-trading-rest-api-get-trades)
的历史和对账成交报告包含 OKX 手续费数据。

支持的价差订单指令:

- 带 GTC 有效期的 `LIMIT`。
- 带 IOC 有效期的 `LIMIT`。
- 带 post-only 执行的 `LIMIT`。

OKX 价差交易 API 路径不支持价差订单列表、条件单、FOK 有效期以及
修改请求。

相关 OKX 文档:

- [价差下单](https://www.okx.com/docs-v5/en/#spread-trading-rest-api-place-order)。
- [价差订单详情](https://www.okx.com/docs-v5/en/#spread-trading-rest-api-get-order-details)。
- [价差订单频道](https://www.okx.com/docs-v5/en/#spread-trading-websocket-private-channel-order-channel)。

### 现货保证金交易的数量语义

在使用现货保证金交易(`use_spot_margin=True`)时,OKX 根据订单方向
以不同方式解释订单数量:

- **限价单**将 `quantity` 解释为基础货币单位数量。
- **市价 SELL** 订单同样使用基础货币单位数量。
- **市价 BUY** 订单将 `quantity` 解释为计价货币名义金额(例如
  USDT)。

:::warning
**提交现货保证金市价 BUY 订单时**,请在订单上设置
`quote_quantity=True`(或预先计算以计价货币计量的金额)。OKX 执行
客户端会拒绝现货保证金以基础货币计量的市价买单,以防止意外成交。

**首次成交时**,订单数量会从计价货币数量自动更新为实际收到的基础
货币数量,反映实际执行的成交。
:::

```python
# 现货保证金市价 BUY,以计价货币计量数量(花费 100 USDT)
order = strategy.order_factory.market(
    instrument_id=instrument_id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(100.0),
    quote_quantity=True,  # 解释为 USDT 名义金额
)
strategy.submit_order(order)
```

### 执行指令

| 指令   | 正向永续互换 | 备注                  |
|---------------|-----------------------|-------------------------|
| `post_only`   | ✓                     | 仅适用于限价单。 |
| `reduce_only` | ✓                     | 仅适用于衍生品。  |

### 有效期

| 有效期 | 正向永续互换 | 备注                                             |
|---------------|-----------------------|-----------------------------------------------------|
| `GTC`         | ✓                     | 撤销前有效。                               |
| `FOK`         | ✓                     | 全部成交或取消。                                     |
| `IOC`         | ✓                     | 立即成交或取消。                              |
| `GTD`         | -                     | *OKX 没有原生的此有效期。*              |

:::note
**GTD(指定日期前有效)**:OKX 通过 `expTime` 支持请求过期,但那是
一种请求超时,而非原生的订单过期指令。

如果你需要 GTD 功能,请使用 Nautilus 的策略托管 GTD 功能。它通过在
指定的过期时间取消订单来处理订单过期。
:::

### 批量操作

| 操作          | 正向永续互换 | 备注                                     |
|--------------------|-----------------------|---------------------------------------------|
| 批量提交       | ✓                     | 在单次请求中提交多个订单。 |
| 批量修改       | ✓                     | 在单次请求中修改多个订单。 |
| 批量取消       | ✓                     | 在单次请求中取消多个订单。 |

### 仓位管理

| 功能           | 正向永续互换 | 备注                                                |
|-------------------|-----------------------|------------------------------------------------------|
| 查询仓位   | ✓                     | 实时仓位更新。                          |
| 仓位模式     | ✓                     | 净额模式与多空模式(见下文)。                  |
| 杠杆控制  | ✓                     | 按标的动态调整杠杆。          |
| 保证金模式       | ✓                     | 支持现金、逐仓和全仓模式。            |

#### 仓位模式

OKX 为衍生品交易支持两种仓位模式:

- **净额模式(Net mode)**(net netting):每个标的一个仓位。买卖订单
  相互对冲。这是大多数交易者的默认和推荐模式。
- **多空模式(Long/Short mode)**(hedging):同一标的的多头和空头
  仓位分开。该模式支持同时持有多头和空头敞口。

:::note
仓位模式必须通过 OKX 网页或应用界面配置,并作用于整个账户。适配器会
检测当前的仓位模式,并相应地处理仓位报告。
:::

### 交易模式与保证金配置

OKX 的统一账户系统对现货和衍生品交易支持不同的交易模式。适配器会
根据你的配置和标的类型确定正确的交易模式。

:::note
**重要**:请先通过 OKX 网页或应用界面配置账户模式。API 无法首次
设置账户模式。
:::

有关 OKX 账户模式和保证金的更多详情,参见
[OKX 账户模式文档](https://www.okx.com/docs-v5/en/#overview-account-mode)。

#### 交易模式概览

OKX 支持多种账户模式。对于订单,适配器会根据你的配置从 `cash`、
`isolated` 或 `cross` 三种交易模式中选择一种:

| 模式           | 用途                       | 杠杆 | 借贷 | 配置                         |
|----------------|--------------------------------|----------|-----------|----------------------------------------|
| **`cash`**     | 无杠杆的现货交易。 | -        | -         | 当 `use_spot_margin=False` 时的默认值。 |
| **`isolated`** | 现货保证金或衍生品。    | ✓        | ✓         | `margin_mode=ISOLATED`。               |
| **`cross`**    | 现货保证金或衍生品。    | ✓        | ✓         | `margin_mode=CROSS`。                  |

#### 基于配置的交易模式选择

适配器根据以下内容选择交易模式:

1. **标的类型**(`SPOT` 与其他 OKX 标的类型)。
2. **配置设置**(`SPOT` 使用 `use_spot_margin`,其他情况使用
   `margin_mode`)。

##### 对于 SPOT 交易

```python
# 无杠杆的简单 SPOT 交易(使用 'cash' 模式)
exec_clients={
    OKX: OKXExecClientConfig(
        instrument_types=(OKXInstrumentType.SPOT,),
        use_spot_margin=False,  # 默认值——简单的 SPOT
        # ... 其他配置
    ),
}

# 带保证金/杠杆的 SPOT 交易(使用 'isolated' 或 'cross' 模式)
exec_clients={
    OKX: OKXExecClientConfig(
        instrument_types=(OKXInstrumentType.SPOT,),
        use_spot_margin=True,  # 为 SPOT 启用保证金交易
        margin_mode=OKXMarginMode.ISOLATED,  # 或使用 CROSS 共享保证金
        # ... 其他配置
    ),
}
```

##### 对于非现货交易

```python
# 带逐仓保证金的衍生品(默认——使用 'isolated' 模式)
exec_clients={
    OKX: OKXExecClientConfig(
        instrument_types=(OKXInstrumentType.SWAP,),
        margin_mode=OKXMarginMode.ISOLATED,  # 或省略——ISOLATED 是默认值
        # ... 其他配置
    ),
}

# 带全仓保证金的衍生品(使用 'cross' 模式)
exec_clients={
    OKX: OKXExecClientConfig(
        instrument_types=(OKXInstrumentType.SWAP,),
        margin_mode=OKXMarginMode.CROSS,  # 跨所有仓位共享保证金
        # ... 其他配置
    ),
}
```

##### 对于混合 SPOT 和衍生品交易

当同时交易 SPOT 和衍生品标的时,适配器会根据所交易的标的,按订单
确定交易模式:

```python
# 混合 SPOT + SWAP 配置
exec_clients={
    OKX: OKXExecClientConfig(
        instrument_types=(OKXInstrumentType.SPOT, OKXInstrumentType.SWAP),
        use_spot_margin=True,           # 仅适用于 SPOT 订单
        margin_mode=OKXMarginMode.CROSS,  # 仅适用于 SWAP 订单
        # ... 其他配置
    ),
}
```

**工作原理:**

- **SPOT 订单**使用 `cross` 模式,因为 `use_spot_margin=True` 且
  `margin_mode=CROSS`。
- **SWAP 订单**使用 `cross` 模式,因为 `margin_mode=CROSS`。
- 每个订单都会根据其标的类型获得正确的 `tdMode`。
- 无需人工干预。

这支持跨标的类型、使用不同保证金配置进行交易的策略,例如:

- 现货-期货套利策略。
- 结合现货和永续互换的 delta 中性策略。
- 跨现货和衍生品市场的做市。

:::warning
**手动覆盖交易模式**:你可以通过 `params={"td_mode": "..."}` 按订单
覆盖交易模式。这会绕过适配器的选择,当该值与标的类型不匹配时(例如
现货标的使用 `isolated`),可能导致订单被拒绝。

请仅在无法通过配置满足的需求下使用手动覆盖。
:::

#### 基于配置方式的优势

- **类型安全**:配置在启动时就会被校验,在下任何订单之前。
- **自动化**:适配器根据标的类型和意图自动选择模式。
- **清晰**:字段名称清楚地表达了意图,例如 `use_spot_margin` 与
  `td_mode`。
- **安全**:不兼容的组合会在到达 OKX 之前被拒绝。
- **向后兼容**:默认值保留现有行为。

### 订单查询

| 功能              | 正向永续互换 | 备注                                     |
|----------------------|-----------------------|--------------------------------------------|
| 查询未成交订单    | ✓                     | 列出所有活跃订单。                   |
| 查询订单历史  | ✓                     | 历史订单数据。                    |
| 订单状态更新 | ✓                     | 实时订单状态变化。            |
| 成交历史        | ✓                     | 执行与成交报告。               |

### 条件单

| 功能            | 正向永续互换 | 备注                                 |
|--------------------|-----------------------|----------------------------------------|
| 订单列表        | ✓                     | 通过 WS 批量提交;仅限普通订单。    |
| OCO 订单         | ✓                     | 一取消另一(One-Cancels-Other)订单。             |
| 括号单     | ✓                     | 止损 + 止盈组合。 |
| 条件单 | ✓                     | 止损单和触及限价单。     |

#### 条件单架构

条件单(OKX 算法单)使用混合架构:

- **提交**:HTTP REST API(`/api/v5/trade/order-algo`)。
- **状态更新**:WebSocket business 端点(`/ws/v5/business`)上的
  `orders-algo` 频道。
- **取消**:带算法单 ID 跟踪的 HTTP REST API。

这一设计确保了:

- 通过 HTTP 立即确认提交。
- 通过 WebSocket 实时更新状态。
- 通过算法单 ID 映射进行适当的订单生命周期管理。

#### 支持的条件单类型

| 订单类型             | 触发类型     | 备注                                     |
|------------------------|-------------------|--------------------------------------------|
| `STOP_MARKET`          | 最新价、标记价、指数价 | 触发时以市价执行。          |
| `STOP_LIMIT`           | 最新价、标记价、指数价 | 触发时下达限价单。     |
| `MARKET_IF_TOUCHED`    | 最新价、标记价、指数价 | 价格触及时以市价执行。      |
| `LIMIT_IF_TOUCHED`     | 最新价、标记价、指数价 | 价格触及时下达限价单。 |
| `TRAILING_STOP_MARKET` | 最新价、标记价、指数价 | 带回调比率的跟踪止损。        |

#### 触发价格类型

条件单支持不同的触发价格来源:

- **最新价**(`TriggerType.LAST_PRICE`):使用最新成交价(默认)。
- **标记价**(`TriggerType.MARK_PRICE`):使用标记价格。
- **指数价**(`TriggerType.INDEX_PRICE`):使用标的指数价格。

```python
# 示例:使用标记价格触发的止损单
stop_order = order_factory.stop_market(
    instrument_id=instrument_id,
    order_side=OrderSide.SELL,
    quantity=Quantity.from_str("0.1"),
    trigger_price=Price.from_str("45000.0"),
    trigger_type=TriggerType.MARK_PRICE,  # 使用标记价格作为触发依据
)
strategy.submit_order(stop_order)
```

## 风险管理

### 强平与 ADL 事件处理

OKX 适配器检测交易所发起的风险管理事件:

- **强平订单**:当交易所强平某个仓位时,适配器会检测到该强平类别,
  并记录带订单详情的警告日志。这些订单仍会通过正常的订单和成交管道
  流转。
- **自动减仓(ADL)**:当 OKX 为对冲对手方的强平而平掉你的仓位时,
  适配器会检测并记录带仓位详情的 ADL 事件。

检测依据是订单记录上的 `category` 字段。已识别的取值有:

| `category`              | 含义                                              |
|-------------------------|------------------------------------------------------|
| `full_liquidation`      | 全仓强平。                           |
| `partial_liquidation`   | 部分强平。                        |
| `adl`                   | 自动减仓平仓。                             |
| `delivery`              | 到期时的合约交割。                     |
| `normal` / 其他取值 | 常规订单流程。                                  |

检测在以下两条路径上都会运行:

- WebSocket 的 `orders` 频道(实时订单/成交更新)。
- HTTP `GET /api/v5/trade/orders-history` 和
  `orders-history-archive`(用于对账和冷启动全量状态)。

:::info
**强平和 ADL 事件会以 WARNING 级别记录日志**,包含订单 ID、标的和
状态等详情。请将监控这些日志作为你风险管理流程的一部分。

适配器会处理这些交易所生成的订单,发出相关的 `OrderFilled` 事件,
并更新仓位。你的策略代码不需要单独的处理路径。
:::

上游参考资料:

- [订单频道与 `category` 字段](https://www.okx.com/docs-v5/en/#order-book-trading-trade-ws-order-channel)
- [自动减仓机制](https://www.okx.com/help/okx-contract-auto-deleveraging-adl)
- [强平机制](https://www.okx.com/help/introduction-to-liquidation)

## 期权交易

OKX 适配器支持交易期权(`OPTION` 标的类型),与其他衍生品相比存在
一些差异。OKX 期权是以标的加密货币结算的反向合约。完整的 API 详情
参见
[OKX 期权交易文档](https://www.okx.com/docs-v5/en/#order-book-trading-trade-post-place-order)。

### 支持的订单类型

仅支持限价类订单。OKX 不允许期权使用市价单。

| 订单类型 | 是否支持 | 备注                                             |
|------------|-----------|-----------------------------------------------------|
| `LIMIT`    | ✓         | 标准限价单。                             |
| `MARKET`   | -         | 在到达 API 之前会被适配器拒绝。  |

期权支持 FOK 和 IOC 有效期。OKX 对期权 FOK 订单使用专用的 `op_fok`
订单类型;适配器会自动处理这一映射。

期权不支持条件单/算法单(`STOP_MARKET`、`STOP_LIMIT`、
`MARKET_IF_TOUCHED`、`LIMIT_IF_TOUCHED`、`TRAILING_STOP_MARKET`),
会被拒绝。

### 定价模式

期权订单可以用三种互斥的方式定价。通过订单 `params` 传入定价模式:

| 模式  | 参数 | 说明                                             |
|-------|-----------|-----------------------------------------------------------|
| Price | (默认) | 以合约计价货币表示的标准限价。        |
| USD   | `px_usd`  | 以美元表示的价格。                                     |
| IV    | `px_vol`  | 以隐含波动率表示的价格(1.0 = 100%)。               |

```python
# 以美元定价
order = strategy.order_factory.limit(
    instrument_id=InstrumentId.from_str("BTC-USD-250328-50000-C.OKX"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(1),
    price=Price.from_str("0"),  # 占位符;px_usd 优先
    params={"px_usd": "100.5"},
)

# 以隐含波动率定价
order = strategy.order_factory.limit(
    instrument_id=InstrumentId.from_str("BTC-USD-250328-50000-C.OKX"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(1),
    price=Price.from_str("0"),  # 占位符;px_vol 优先
    params={"px_vol": "0.55"},
)
```

修改订单时,可以将相同的 `px_usd` 或 `px_vol` 参数传给修改命令,
以在原定价模式下修改价格。

### 期权 Greeks

OKX 在 `opt-summary` 频道上发布两套并行的 greek 值:

- **Black-Scholes(`BLACK_SCHOLES`)**:以美元计价的 Greeks。与
  Deribit 和 Bybit 适配器使用的惯例一致。
- **价格调整(`PRICE_ADJUSTED`)**:以标的币种单位计价的 Greeks。
  与 OKX 原生的合约惯例一致。

默认情况下,适配器在每个 `opt-summary` tick 上都发出这两种。每个
发出的 `OptionGreeks` 都携带一个 `convention` 字段,设置为
`GreeksConvention.BLACK_SCHOLES` 或 `GreeksConvention.PRICE_ADJUSTED`,
以便接收方按消息分支处理。

要缩小数据流范围,可在订阅时传入 `params["greeks_convention"]`:

- 单个字符串:`"BLACK_SCHOLES"` 或 `"PRICE_ADJUSTED"`(不区分大小写)。
- 字符串列表:`["BLACK_SCHOLES", "PRICE_ADJUSTED"]`。
- 省略:适配器会发出两种。

未知条目会记录一条警告并被跳过。如果请求的每一项都未知,适配器会
回退到发出两种。

```python
# 默认(两种惯例,接收方按消息分支)
self.subscribe_option_greeks(instrument_id)

def on_option_greeks(self, greeks: OptionGreeks) -> None:
    if greeks.convention == GreeksConvention.BLACK_SCHOLES:
        self._handle_bs(greeks)
    else:
        self._handle_pa(greeks)
```

```python
# 缩小到单一惯例
self.subscribe_option_greeks(
    instrument_id,
    params={"greeks_convention": "PRICE_ADJUSTED"},
)
```

```python
# 显式列表(当列出两者时等同于默认值)
self.subscribe_option_greeks(
    instrument_id,
    params={"greeks_convention": ["BLACK_SCHOLES", "PRICE_ADJUSTED"]},
)
```

:::note
数据引擎按 `instrument_id` 对期权 Greeks 订阅进行去重,因此如果同一
节点上的两个 actor 以不同的单一惯例订阅同一个标的,只有第一个会
到达适配器。第二个 actor 会得到第一个 actor 所选的惯例。变通方法:
任一 actor 都可以在不传 `params`(或传完整列表)的情况下订阅,以
接收两种数据流,并在本地根据 `greeks.convention` 进行过滤。
:::

### 仓位 Greeks

适配器从 OKX 仓位数据中暴露仓位级别的 Black-Scholes Greeks
(`delta_bs`、`gamma_bs`、`theta_bs`、`vega_bs`)。这些数据可通过
标准的仓位报告管道获取。

### 限制

- `reduce_only` 不适用于期权,会被自动移除。
- 仓位方向默认为 `Net`。

### 配置

期权需要 `instrument_families` 配置参数来限定要加载哪些标的资产:

```python
config = TradingNodeConfig(
    data_clients={
        OKX: OKXDataClientConfig(
            instrument_types=(OKXInstrumentType.OPTION,),
            instrument_families=("BTC-USD", "ETH-USD"),
            instrument_provider=InstrumentProviderConfig(load_all=True),
        ),
    },
    exec_clients={
        OKX: OKXExecClientConfig(
            instrument_types=(OKXInstrumentType.OPTION,),
            instrument_families=("BTC-USD", "ETH-USD"),
            margin_mode=OKXMarginMode.CROSS,
        ),
    },
)
```

## 事件合约

OKX 通过 `instType=EVENTS` 暴露预测市场合约。适配器将这些标的加载为
Nautilus 的 `BinaryOption` 标的,并将 OKX 元数据(如 `seriesId`、
`instCategory`、`instIdCode`、`state` 和 `ruleType`)保留在标的的
`info` 字段中。

### 加载事件合约标的

在数据或执行客户端配置中使用 `OKXInstrumentType.EVENTS`。
`instrument_families` 设置对应于事件合约的 OKX `seriesId` 值。当省略
`instrument_families` 时,适配器会先请求事件合约系列列表,然后为每个
系列请求标的。

```python
config = TradingNodeConfig(
    data_clients={
        OKX: OKXDataClientConfig(
            instrument_types=(OKXInstrumentType.EVENTS,),
            instrument_families=("BTC-ABOVE-DAILY",),
            instrument_provider=InstrumentProviderConfig(load_all=True),
        ),
    },
    exec_clients={
        OKX: OKXExecClientConfig(
            instrument_types=(OKXInstrumentType.EVENTS,),
            instrument_families=("BTC-ABOVE-DAILY",),
            margin_mode=OKXMarginMode.CROSS,
        ),
    },
)
```

### 事件合约市场数据

底层 HTTP 客户端暴露了 OKX 公开的事件合约发现端点:

- `request_event_contract_series`。
- `request_event_contract_events`。
- `request_event_contract_markets`。

底层 WebSocket 客户端通过 `subscribe_event_contract_markets` 和
`unsubscribe_event_contract_markets` 支持 `event-contract-markets`
频道。该频道发布市场状态和 floor-strike 生成更新,没有初始快照,
也不包含 `instId`,因此适配器将其作为原始场所 JSON 转发。

:::note
OKX 的标准市场数据端点为 `EVENTS` 返回的是 YES 一侧的数据。当策略
需要两种结果时,请从 YES 一侧价格推导出 NO 一侧价格。
:::

### 事件合约交易

提交事件合约订单时,通过订单 `params` 传入 OKX 事件结果:

```python
order = strategy.order_factory.limit(
    instrument_id=InstrumentId.from_str("BTC-ABOVE-DAILY-260224-1600-65000.OKX"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(1),
    price=Price.from_str("0.42"),
    params={"outcome": "yes"},
)
strategy.submit_order(order)
```

OKX 要求 `EVENTS` 订单必须提供 `outcome`。对于非 post-only 的事件
合约订单和修改操作,还要求提供 `speedBump=1`。适配器会在发送订单之前
校验 `outcome`,并在未提供时,为非 post-only 的事件订单将
`speedBump` 默认设为 `1`。

结算成交会以 OKX 订单类别 `delivery` 到达。适配器在实时订单更新和
对账期间会识别这一类别。

上游参考资料:

- [事件合约 REST 端点](https://www.okx.com/docs-v5/en/#public-data-rest-api-get-series)。
- [WS 频道](https://www.okx.com/docs-v5/en/#public-data-websocket-event-contract-markets-channel)。
- [下单请求字段](https://www.okx.com/docs-v5/en/#order-book-trading-trade-post-place-order)。

## 身份验证

要使用 OKX 适配器,请在你的 OKX 账户中创建 API 凭证:

1. 登录你的 OKX 账户,导航到 API 管理页面。
2. 创建一个具有交易和数据访问所需权限的新 API key。
3. 记录你的 API key、secret key 和口令(passphrase)。

你可以通过环境变量提供这些凭证:

```bash
export OKX_API_KEY="your_api_key"
export OKX_API_SECRET="your_api_secret"
export OKX_API_PASSPHRASE="your_passphrase"
```

或直接在配置中传入(生产环境不推荐)。

## 模拟交易

OKX 提供了一个模拟交易环境,可以在不使用真实资金的情况下测试策略。

### 设置模拟账户

1. 在 [okx.com](https://www.okx.com) 登录你的 OKX 账户。
2. 导航到 **Trade** > **Demo Trading**。
3. 在 Demo Trading 中进入 **Personal Center**。
4. 选择 **Demo Trading API** 并创建一个新的 API key。
5. 记录你的模拟 API key、secret key 和口令。

你可以通过环境变量提供模拟凭证:

```bash
export OKX_API_KEY="your_demo_api_key"
export OKX_API_SECRET="your_demo_api_secret"
export OKX_API_PASSPHRASE="your_demo_passphrase"
```

### 配置

在客户端配置中设置 `environment=OKXEnvironment.DEMO`:

```python
from nautilus_trader.core.nautilus_pyo3 import OKXEnvironment

config = TradingNodeConfig(
    data_clients={
        OKX: OKXDataClientConfig(
            environment=OKXEnvironment.DEMO,
            # ... 其他配置
        ),
    },
    exec_clients={
        OKX: OKXExecClientConfig(
            environment=OKXEnvironment.DEMO,
            # ... 其他配置
        ),
    },
)
```

启用模拟模式时:

- REST API 请求包含 `x-simulated-trading: 1` 请求头。
- WebSocket 连接使用模拟端点(`wspap.okx.com`)。

:::note
模拟 API key 与生产密钥是分开的。请通过 Demo Trading 界面为模拟交易
创建 API key。生产 API key 在模拟模式下不可用。
:::

## 区域端点

OKX 按区域提供不同的端点,API key 仅对其注册所在的区域有效(对另一
区域的端点使用该 key 会返回 `API key doesn't exist`)。设置
`region` 以选择正确的端点集:

| 区域   | 注册于 | REST            | WebSocket 主机       |
|----------|---------------|-----------------|----------------------|
| `GLOBAL` | `www.okx.com` | `www.okx.com`   | `ws.okx.com`         |
| `EEA`    | `my.okx.com`  | `eea.okx.com`   | `wseea.okx.com`      |
| `US`     | `app.okx.com` | `us.okx.com`    | `wsus.okx.com`       |

`region` 默认为 `GLOBAL`。例如,一个 EEA 账户:

```python
from nautilus_trader.core.nautilus_pyo3 import OKXRegion

config = TradingNodeConfig(
    data_clients={
        OKX: OKXDataClientConfig(
            region=OKXRegion.EEA,
            # ... 其他配置
        ),
    },
    exec_clients={
        OKX: OKXExecClientConfig(
            region=OKXRegion.EEA,
            # ... 其他配置
        ),
    },
)
```

`region` 选择区域默认值,并与 `environment` 结合以选择模拟主机
(例如 EEA 模拟为 `wseeapap.okx.com`)。显式的 `base_url_http` 和
`base_url_ws` 覆盖值始终优先于区域默认值。

## 资金费率

适配器从
[资金费率频道](https://www.okx.com/docs-v5/en/#public-data-websocket-funding-rate-channel)
WebSocket 数据流接收资金费率数据。OKX 在每条消息中都提供
`fundingTime` 和 `nextFundingTime`,适配器根据这两个值的差计算
`interval`。

对于历史资金费率请求,适配器根据
[获取资金费率历史](https://www.okx.com/docs-v5/en/#public-data-rest-api-get-funding-rate-history)
端点返回的连续资金费率时间戳计算间隔。

## 速率限制

适配器在对 REST 和 WebSocket 调用保持合理默认值的同时,强制执行 OKX
按端点的配额。

### REST 限制

- 内部全局桶:每秒 250 次请求。
- 端点特定的配额见下表,尽可能与 OKX 公布的限制一致。

### WebSocket 限制

- 建立连接:每秒 3 次请求(按 IP)。
- 订阅操作(订阅/取消订阅/登录):每个连接每小时 480 次请求。
- 订单操作桶见下表,尽可能与 OKX 公布的限制一致。

| 操作键  | 限制(请求/秒) | 备注                                                     |
|----------------|-----------------|-----------------------------------------------------------|
| `order`        | 30              | OKX 每 2 秒 60 次请求。                              |
| `cancel`       | 30              | OKX 每 2 秒 60 次请求。                              |
| `amend`        | 30              | OKX 每 2 秒 60 次请求。                              |
| `batch-order`  | 7               | OKX 每 2 秒 300 笔订单,按完整批次向下取整。 |
| `batch-cancel` | 7               | OKX 每 2 秒 300 笔订单,按完整批次向下取整。 |
| `batch-amend`  | 7               | OKX 每 2 秒 300 笔订单,按完整批次向下取整。 |
| `mass-cancel`  | 2               | OKX 每 2 秒 5 次请求,向下取整。                 |
| `algo-order`   | 10              | OKX 每 2 秒 20 次请求。                              |
| `algo-cancel`  | 1               | OKX 每 2 秒 20 笔订单,按完整批次向下取整。  |

:::warning
OKX 强制执行按端点和按账户的配额。超出配额会导致 HTTP 429 响应,
并对该 key 进行临时限流。
:::

| 键 / 端点                          | 限制(请求/秒) | 备注                                             |
|-----------------------------------------|-----------------|-----------------------------------------------------|
| `okx:global`                            | 250             | 适配器级别的共享桶。                      |
| `/api/v5/account/set-position-mode`     | 2               | OKX 每 2 秒 5 次请求,向下取整。         |
| `/api/v5/account/balance`               | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/account/trade-fee`             | 2               | OKX 每 2 秒 5 次请求,向下取整。         |
| `/api/v5/account/positions`             | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/account/positions-history`     | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/public/instruments`            | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/public/position-tiers`         | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/public/event-contract/series`  | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/public/event-contract/events`  | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/public/event-contract/markets` | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/public/opt-summary`            | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/public/price-limit`            | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/public/time`                   | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/public/mark-price`             | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/public/funding-rate-history`   | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/market/index-tickers`          | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/market/books`                  | 20              | OKX 每 2 秒 40 次请求。                      |
| `/api/v5/market/candles`                | 20              | OKX 每 2 秒 40 次请求。                      |
| `/api/v5/market/history-candles`        | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/market/history-trades`         | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/sprd/spreads`                  | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/sprd/order`                    | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/sprd/cancel-order`             | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/sprd/mass-cancel`              | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/sprd/orders-pending`           | 5               | OKX 每 2 秒 10 次请求。                      |
| `/api/v5/sprd/orders-history`           | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/sprd/trades`                   | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/trade/order`                   | 30              | OKX 每 2 秒 60 次请求。                      |
| `/api/v5/trade/cancel-batch-orders`     | 7               | OKX 每 2 秒 300 笔订单,向下取整。         |
| `/api/v5/trade/orders-pending`          | 30              | OKX 每 2 秒 60 次请求。                      |
| `/api/v5/trade/orders-history`          | 20              | OKX 每 2 秒 40 次请求。                      |
| `/api/v5/trade/fills`                   | 30              | OKX 每 2 秒 60 次请求。                      |
| `/api/v5/trade/order-algo`              | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/trade/cancel-algos`            | 1               | OKX 每 2 秒 20 笔订单。                        |
| `/api/v5/trade/cancel-advance-algos`    | 1               | 用于高级算法单取消的保守桶。     |
| `/api/v5/trade/amend-algos`             | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/trade/orders-algo-pending`     | 10              | OKX 每 2 秒 20 次请求。                      |
| `/api/v5/trade/orders-algo-history`     | 10              | OKX 每 2 秒 20 次请求。                      |

所有键都包含 `okx:global` 桶。URL 在进行速率限制之前会规范化为
去除查询字符串,因此带有不同过滤条件的请求会共享同一配额。

对于基于订单的取消配额,适配器使用假定为完整批量大小的请求级桶:
常规批量取消每次请求 20 笔订单,算法单取消每次请求 10 笔订单。OKX
目前公开的文档不再列出 `/api/v5/trade/cancel-advance-algos` 的速率
限制,但由于 HTTP 客户端仍可调用这个遗留路径,适配器仍保留了一个
端点特定的桶。

:::info
参见 [OKX 速率限制文档](https://www.okx.com/docs-v5/en/#rest-api-rate-limit)。
:::

## 配置

### 配置选项

OKX 数据客户端提供以下配置选项。此表描述的是 Rust/pyo3 v2 数据
客户端配置;Python v1 实时配置不暴露下方的订单簿健康监控字段。

#### 数据客户端

| 选项                             | 默认值                     | 说明                                  |
|------------------------------------|-----------------------------|----------------------------------------------|
| `instrument_types`                 | `(OKXInstrumentType.SPOT,)` | 要加载的 OKX 标的类型。                |
| `contract_types`                   | `None`                      | 要加载的合约样式。                     |
| `load_spreads`                     | `False`                     | 加载实时价差标的。               |
| `instrument_families`              | `None`                      | 品类或事件 `seriesId` 值。         |
| `base_url_http`                    | `None`                      | OKX REST 端点的覆盖值。          |
| `base_url_ws_public`               | `None`                      | 公开 WebSocket URL 的覆盖值。       |
| `base_url_ws_business`             | `None`                      | business WebSocket URL 的覆盖值。     |
| `api_key`                          | `None`                      | 未设置时回退到 `OKX_API_KEY`。      |
| `api_secret`                       | `None`                      | 未设置时回退到 `OKX_API_SECRET`。   |
| `api_passphrase`                   | `None`                      | 回退到 `OKX_API_PASSPHRASE`。          |
| `environment`                      | `None`                      | 环境枚举(`LIVE` 或 `DEMO`)。         |
| `region`                           | `None`                      | 区域枚举(`GLOBAL`、`EEA` 或 `US`)。      |
| `http_timeout_secs`                | `60`                        | REST 市场数据请求超时时间。            |
| `max_retries`                      | `3`                         | 可恢复 REST 错误的重试次数。  |
| `retry_delay_initial_ms`           | `1,000`                     | 重试之前的初始延迟。               |
| `retry_delay_max_ms`               | `10,000`                    | 最大指数退避延迟。           |
| `update_instruments_interval_mins` | `60`                        | 后台标的刷新间隔。      |
| `book_stale_check_interval_secs`   | `5`                         | 陈旧订单簿检查间隔。                   |
| `book_stale_threshold_secs`        | `30`                        | 触发陈旧订单簿警告前的空闲时间。       |
| `book_snapshot_timeout_secs`       | `3`                         | 重连后等待快照的时间。                |
| `vip_level`                        | `None`                      | 根据 VIP 等级启用更高深度的订单簿。      |
| `proxy_url`                        | `None`                      | 可选的 HTTP 与 WebSocket 代理 URL。       |
| `transport_backend`                | `Sockudo`                   | WebSocket 传输后端。                 |

将 `book_stale_check_interval_secs`、`book_stale_threshold_secs`
或 `book_snapshot_timeout_secs` 设为 `0` 可禁用对应的健康监控。
清淡的市场可能长时间没有订单簿更新;对于交易稀疏的标的,可以增加
`book_stale_threshold_secs`。

数据客户端支持的 `instrument_types` 取值为 `SPOT`、`MARGIN`、
`SWAP`、`FUTURES`、`OPTION` 和 `EVENTS`。

`instrument_families` 对 `OPTION` 是必需的,对 `FUTURES`、`SWAP`
和 `EVENTS` 是可选的,对 `SPOT` 和 `MARGIN` 会被忽略。对于
`EVENTS`,请传入 OKX 的 `seriesId` 值,例如 `BTC-ABOVE-DAILY`。
价差标的使用 `load_spreads` 而非 `instrument_types`,因为 OKX 通过
`/api/v5/sprd/spreads` 提供它们。

OKX 执行客户端提供以下配置选项:

#### 执行客户端

| 选项                            | 默认值                     | 说明                                 |
|-----------------------------------|-----------------------------|-----------------------------------------------|
| `instrument_types`                | `(OKXInstrumentType.SPOT,)` | 可交易的 OKX 标的类型。              |
| `contract_types`                  | `None`                      | 要加载的可交易合约样式。           |
| `load_spreads`                    | `False`                     | 加载实时价差标的。              |
| `instrument_families`             | `None`                      | 品类或事件 `seriesId` 值。        |
| `base_url_http`                   | `None`                      | OKX 交易 REST 端点的覆盖值。 |
| `base_url_ws_private`             | `None`                      | 私有 WebSocket URL 的覆盖值。     |
| `base_url_ws_business`            | `None`                      | business WebSocket URL 的覆盖值。    |
| `api_key`                         | `None`                      | 未设置时回退到 `OKX_API_KEY`。     |
| `api_secret`                      | `None`                      | 未设置时回退到 `OKX_API_SECRET`。  |
| `api_passphrase`                  | `None`                      | 回退到 `OKX_API_PASSPHRASE`。         |
| `environment`                     | `None`                      | 环境枚举(`LIVE` 或 `DEMO`)。        |
| `region`                          | `None`                      | 区域枚举(`GLOBAL`、`EEA` 或 `US`)。     |
| `margin_mode`                     | `None`                      | 保证金模式(`ISOLATED` 或 `CROSS`)。        |
| `use_spot_margin`                 | `False`                     | 启用现货式保证金或杠杆。      |
| `http_timeout_secs`               | `60`                        | REST 交易请求超时时间。               |
| `use_fills_channel`               | `False`                     | 订阅成交频道(VIP5+)。       |
| `use_mm_mass_cancel`              | `False`                     | 使用做市商批量取消端点。 |
| `max_retries`                     | `3`                         | 可恢复 REST 错误的重试次数。 |
| `retry_delay_initial_ms`          | `1,000`                     | 重试之前的初始延迟。              |
| `retry_delay_max_ms`              | `10,000`                    | 最大指数退避延迟。          |
| `use_spot_cash_position_reports`  | `False`                     | 从钱包生成 SPOT 现金仓位。  |
| `proxy_url`                       | `None`                      | 可选的 HTTP 与 WebSocket 代理 URL。      |
| `transport_backend`               | `Sockudo`                   | WebSocket 传输后端。                |

执行客户端支持的 `instrument_types` 取值为 `SPOT`、`MARGIN`、
`SWAP`、`FUTURES`、`OPTION` 和 `EVENTS`。

`instrument_families` 对执行客户端的含义与数据客户端相同。价差标的
使用 OKX 价差 ID 而非 `instrument_types`;需要在数据客户端上通过
`load_spreads=True` 加载它们,对于仅执行的 Python v1 节点,还需在
交易之前在执行客户端上加载。

### 手动端点覆盖

设置 `region`(参见 [区域端点](#区域端点))会自动选择正确的 EEA
或 US 端点,这是推荐的做法。下方显式的 `base_url_*` 覆盖仍可用于
代理、自定义路由,或某个区域未覆盖的端点;它们优先于 `region`
默认值。EEA 基础地址作为示例展示如下。

| 配置字段           | 生产环境基础地址                  | 模拟环境基础地址                     | WebSocket 路径    |
|------------------------|----------------------------|--------------------------------|-------------------|
| `base_url_http`        | `https://eea.okx.com`      | `https://eea.okx.com`         |                   |
| `base_url_ws_public`   | `wss://wseea.okx.com:8443` | `wss://wseeapap.okx.com:8443` | `/ws/v5/public`   |
| `base_url_ws_private`  | `wss://wseea.okx.com:8443` | `wss://wseeapap.okx.com:8443` | `/ws/v5/private`  |
| `base_url_ws_business` | `wss://wseea.okx.com:8443` | `wss://wseeapap.okx.com:8443` | `/ws/v5/business` |

对于 WebSocket 字段,将同一行的基础地址和路径拼接起来。

在数据客户端配置中使用 `base_url_ws_public`,在执行客户端配置中使用
`base_url_ws_private`。EEA 账户还必须在两个 v2 配置上都设置
`base_url_ws_business`,因为 v2 不会从公开或私有覆盖值推导出
business WebSocket URL。

Python v1 实时配置暴露的是 `base_url_ws`,而非拆分的 WebSocket
字段。对于这些配置,请在 `OKXDataClientConfig` 上将 `base_url_ws`
设置为公开的 EEA WebSocket URL,在 `OKXExecClientConfig` 上设置为
私有的 EEA WebSocket URL;每个客户端会从该值推导出自己的 business
WebSocket URL。

当前的官方端点列表参见
[OKX EEA API 文档](https://my.okx.com/docs-v5/en/)。

以下是使用 OKX 数据和执行客户端的实时交易节点配置示例:

```python
from nautilus_trader.adapters.okx import OKX
from nautilus_trader.adapters.okx import OKXDataClientConfig, OKXExecClientConfig
from nautilus_trader.adapters.okx.factories import OKXLiveDataClientFactory, OKXLiveExecClientFactory
from nautilus_trader.config import InstrumentProviderConfig, TradingNodeConfig
from nautilus_trader.core.nautilus_pyo3 import OKXContractType
from nautilus_trader.core.nautilus_pyo3 import OKXEnvironment
from nautilus_trader.core.nautilus_pyo3 import OKXInstrumentType
from nautilus_trader.core.nautilus_pyo3 import OKXMarginMode
from nautilus_trader.live.node import TradingNode

config = TradingNodeConfig(
    ...,
    data_clients={
        OKX: OKXDataClientConfig(
            api_key=None,           # 将使用 OKX_API_KEY 环境变量
            api_secret=None,        # 将使用 OKX_API_SECRET 环境变量
            api_passphrase=None,    # 将使用 OKX_API_PASSPHRASE 环境变量
            base_url_http=None,
            base_url_ws=None,
            environment=OKXEnvironment.LIVE,
            instrument_provider=InstrumentProviderConfig(load_all=True),
            instrument_types=(OKXInstrumentType.SWAP,),
            contract_types=(OKXContractType.LINEAR,),
        ),
    },
    exec_clients={
        OKX: OKXExecClientConfig(
            api_key=None,
            api_secret=None,
            api_passphrase=None,
            base_url_http=None,
            base_url_ws=None,
            environment=OKXEnvironment.LIVE,
            instrument_provider=InstrumentProviderConfig(load_all=True),
            instrument_types=(OKXInstrumentType.SWAP,),
            contract_types=(OKXContractType.LINEAR,),
        ),
    },
)
node = TradingNode(config=config)
node.add_data_client_factory(OKX, OKXLiveDataClientFactory)
node.add_exec_client_factory(OKX, OKXLiveExecClientFactory)
node.build()
```

## 贡献

:::info
如需了解更多功能或为 OKX 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
