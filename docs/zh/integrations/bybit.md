# Bybit

Bybit 成立于 2018 年,按加密资产和加密衍生品的日交易量和未平仓合约
计,是最大的加密货币交易所之一。该集成支持接入 Bybit 的实时市场数据
与订单执行。

## 示例

可以在[这里](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/bybit/)找到实时示例脚本。

## 概览

本指南假设交易者需要同时配置实时市场数据源和交易执行。Bybit 适配器
包含多个组件,可以组合使用,也可以单独使用,视具体使用场景而定。

- `BybitHttpClient`:底层 HTTP API 连接。
- `BybitWebSocketClient`:底层 WebSocket API 连接。
- `BybitInstrumentProvider`:标的解析与加载功能。
- `BybitDataClient`:市场数据流管理器。
- `BybitExecutionClient`:账户管理与交易执行网关。
- `BybitLiveDataClientFactory`:Bybit 数据客户端工厂(供交易节点
  构建器使用)。
- `BybitLiveExecClientFactory`:Bybit 执行客户端工厂(供交易节点
  构建器使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),不必直接与这些底层
组件打交道。
:::

## Bybit 文档

Bybit 在 [Bybit 帮助中心](https://www.bybit.com/en/help-center)为用户
提供了详尽的文档。建议将本 NautilusTrader 集成指南与 Bybit 官方文档
结合参考。

## 产品

产品(product)是一组相关标的类型的统称。

:::note
在 Bybit v5 API 中,产品也称为 `category`。
:::

Bybit 支持以下产品类型:

| 产品类型                | 是否支持 | 备注                                    |
|-----------------------------|-----------|------------------------------------------|
| 现货加密货币       | ✓         | 支持保证金的原生现货市场。 |
| 正向永续合约  | ✓         | 以 USDT/USDC 计保证金的永续互换。      |
| 正向期货合约    | ✓         | 交割结算的正向期货。         |
| 反向永续合约 | ✓         | 以币计保证金的永续互换。           |
| 反向期货合约   | ✓         | 以币计保证金的交割期货。          |
| 期权合约            | ✓         | 以 USDT 结算的欧式期权。           |

## 符号规则

为了区分 Bybit 上不同的产品类型,Nautilus 使用特定的产品类别后缀来
标记符号:

- `-SPOT`:现货加密货币
- `-LINEAR`:永续和期货合约
- `-INVERSE`:反向永续和反向期货合约
- `-OPTION`:期权合约

必须将这些后缀附加到 Bybit 原始符号字符串之后,以标识标的 ID 对应的
具体产品类型。例如:

- 以太坊/泰达币现货交易对使用 `-SPOT` 标识,例如 `ETHUSDT-SPOT`。
- BTCUSDT 永续期货合约使用 `-LINEAR` 标识,例如 `BTCUSDT-LINEAR`。
- BTCUSD 反向永续期货合约使用 `-INVERSE` 标识,例如
  `BTCUSD-INVERSE`。
- 一个以 USDT 结算的 BTC 看跌期权:`BTC-27MAR26-70000-P-USDT-OPTION`。
- 一个以 USDC 结算的 ETH 看涨期权:`ETH-28FEB25-2800-C-OPTION`。

Bybit 的期权符号对以 USDT 结算的合约会包含结算货币(例如
`BTC-27MAR26-70000-P-USDT`),对以 USDC 结算的合约则会省略(例如
`ETH-28FEB25-2800-C`)。适配器会在 API 返回的任何符号后附加
`-OPTION`。

## 标的加载

Bybit 数据和执行客户端使用通用的 `instrument_provider` 配置。请配置
它,以便在策略订阅市场数据或下单之前加载标的。订阅操作不会请求缺失的
标的定义。

```python
from nautilus_trader.adapters.bybit import BybitProductType
from nautilus_trader.adapters.bybit.config import BybitDataClientConfig
from nautilus_trader.config import InstrumentProviderConfig

BybitDataClientConfig(
    instrument_provider=InstrumentProviderConfig(load_all=True),
    product_types=(BybitProductType.SPOT,),
)
```

当你只需要一组已知的标的时,使用 `load_ids`:

```python
from nautilus_trader.adapters.bybit import BybitProductType
from nautilus_trader.adapters.bybit.config import BybitDataClientConfig
from nautilus_trader.config import InstrumentProviderConfig
from nautilus_trader.model.identifiers import InstrumentId

BybitDataClientConfig(
    instrument_provider=InstrumentProviderConfig(
        load_ids=frozenset([InstrumentId.from_str("BTCUSDT-SPOT.BYBIT")]),
    ),
    product_types=(BybitProductType.SPOT,),
)
```

配置的 `product_types` 必须与每个标的 ID 中的产品后缀相匹配。

## 环境

Bybit 提供三种交易环境。请在客户端配置中通过 `environment` 枚举配置
对应的环境。

| 环境  | 配置                                | 说明                                                      |
|--------------|---------------------------------------|------------------------------------------------------------------|
| **Mainnet**  | `BybitEnvironment.MAINNET`            | 使用真实资金的生产交易。                              |
| **Demo**     | `BybitEnvironment.DEMO`               | 在主网基础设施上使用模拟资金进行练习交易。 |
| **Testnet**  | `BybitEnvironment.TESTNET`            | 用于开发和集成测试的独立测试网络。   |

### Mainnet(生产环境)

用于真实资金实盘交易的默认环境。

```python
from nautilus_trader.adapters.bybit import BybitEnvironment

config = BybitExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    environment=BybitEnvironment.MAINNET,
)
```

环境变量:`BYBIT_API_KEY`、`BYBIT_API_SECRET`

### 模拟交易(Demo)

模拟交易使用 Bybit 的主网基础设施,配合模拟资金。可以从
[Bybit 模拟交易页面](https://www.bybit.com/en/demo-trading)创建
demo API key。

```python
from nautilus_trader.adapters.bybit import BybitEnvironment

config = BybitExecClientConfig(
    api_key="YOUR_DEMO_API_KEY",
    api_secret="YOUR_DEMO_API_SECRET",
    environment=BybitEnvironment.DEMO,
)
```

环境变量:`BYBIT_DEMO_API_KEY`、`BYBIT_DEMO_API_SECRET`

:::warning
**Demo 环境的限制:**

- 模拟交易**不支持** WebSocket 交易 API。NautilusTrader 在 demo 模式
  下会自动使用 HTTP REST API 处理订单操作。
- 原生的 TP/SL 以及期权参数(`order_iv`、`mmp`)在新订单上可以通过
  HTTP 创建订单端点在 demo 中使用。
- 自定义 TP/SL 触发价格 `tp_trigger_price` 和 `sl_trigger_price` 在
  demo 中不受支持(设置它们的订单会被拒绝);创建订单端点无法携带
  这些字段。
- Demo 私有数据流使用 `wss://stream-demo.bybit.com`,但公开市场数据
  使用 Bybit 主网的公开数据流 `wss://stream.bybit.com`。

:::

### 测试网

用于开发和集成测试的独立测试网络。

```python
from nautilus_trader.adapters.bybit import BybitEnvironment

config = BybitExecClientConfig(
    api_key="YOUR_TESTNET_API_KEY",
    api_secret="YOUR_TESTNET_API_SECRET",
    environment=BybitEnvironment.TESTNET,
)
```

环境变量:`BYBIT_TESTNET_API_KEY`、`BYBIT_TESTNET_API_SECRET`

:::note
测试网支持包括 WebSocket 交易 API 在内的所有交易功能。它使用与主网
完全独立的基础设施,因此市场数据和流动性与生产环境有很大不同。
:::

当 `environment=BybitEnvironment.TESTNET` 时,适配器会自动解析 Bybit
文档中记录的测试网端点:

- REST API:`https://api-testnet.bybit.com`
- 公开 WebSocket:`wss://stream-testnet.bybit.com/v5/public/{spot|linear|inverse|option}`
- 私有 WebSocket:`wss://stream-testnet.bybit.com/v5/private`
- 交易 WebSocket:`wss://stream-testnet.bybit.com/v5/trade`

### 测试网设置

设置 Bybit 测试网账户和凭证:

1. 在桌面浏览器中打开 [testnet.bybit.com](https://testnet.bybit.com)。
2. 创建一个独立的测试网账户,或登录你现有的测试网账户。
3. 从 **Assets -> Assets Overview -> Request Test Coins** 请求测试
   币,以便账户拥有可用于测试的余额。
4. 打开
   [testnet.bybit.com/app/user/api-management](https://testnet.bybit.com/app/user/api-management)
   中的 **API Management**。
5. 点击 **Create New Key**。
6. 根据你的使用场景选择所需权限。
7. 完成 2FA 提示,复制 API key 和 secret。
8. 在你的 shell 中导出凭证:

   ```bash
   export BYBIT_TESTNET_API_KEY="YOUR_TESTNET_API_KEY"
   export BYBIT_TESTNET_API_SECRET="YOUR_TESTNET_API_SECRET"
   ```

Bybit 当前的测试网指南还提到:

- API key 在网站上创建,而非在移动应用中。
- 新用户注册后的前 48 小时内可能无法创建 API key。
- 测试网与主网是分离的。请勿向测试网账户存入真实资金。
- Bybit 目前记录的测试网账户设置流程需要通过桌面浏览器完成。

## 订单能力

Bybit 提供灵活的触发类型组合,支持更广泛的 Nautilus 订单类型。下方
列出的所有订单类型都可以*既*作为入场单*也*作为出场单使用,跟踪止损
除外(它使用与仓位相关的 API)。

### 订单类型

| 订单类型             | 现货 | 正向 | 反向 | 期权 | 备注                             |
|------------------------|------|--------|---------|--------|-----------------------------------|
| `MARKET`               | ✓    | ✓      | ✓       | ✓      | 支持以计价货币计量数量。          |
| `LIMIT`                | ✓    | ✓      | ✓       | ✓      |                                   |
| `STOP_MARKET`          | ✓    | ✓      | ✓       | -      | *期权不支持*。      |
| `STOP_LIMIT`           | ✓    | ✓      | ✓       | -      | *期权不支持*。      |
| `MARKET_IF_TOUCHED`    | ✓    | ✓      | ✓       | -      | *期权不支持*。      |
| `LIMIT_IF_TOUCHED`     | ✓    | ✓      | ✓       | -      | *期权不支持*。      |
| `TRAILING_STOP_MARKET` | -    | ✓      | ✓       | -      | *现货/期权不支持*。 |

### 执行指令

| 指令   | 现货 | 正向 | 反向 | 期权 | 备注                             |
|---------------|------|--------|---------|--------|-----------------------------------|
| `post_only`   | ✓    | ✓      | ✓       | ✓      | 仅支持 `LIMIT` 订单。 |
| `reduce_only` | -    | ✓      | ✓       | ✓      | *现货不支持*。         |

### 有效期

| 有效期 | 现货 | 正向 | 反向 | 期权 | 备注                        |
|---------------|------|--------|---------|--------|------------------------------|
| `GTC`         | ✓    | ✓      | ✓       | ✓      | 撤销前有效。          |
| `GTD`         | -    | -      | -       | -      | *不支持*。             |
| `FOK`         | ✓    | ✓      | ✓       | ✓      | 全部成交或取消。                |
| `IOC`         | ✓    | ✓      | ✓       | ✓      | 立即成交或取消。         |

### 高级订单功能

| 功能            | 现货 | 正向 | 反向 | 期权 | 备注                                  |
|--------------------|------|--------|---------|--------|----------------------------------------|
| 订单修改 | ✓    | ✓      | ✓       | ✓      | 价格和数量修改。       |
| 括号单/OCO 订单 | ✓    | ✓      | ✓       | -      | 仅限 UI;API 用户需自行实现。 |
| 冰山订单     | ✓    | ✓      | ✓       | -      | 每个账户最多 10 个,每个符号最多 1 个。      |

### 批量操作

| 操作          | 现货 | 正向 | 反向 | 期权 | 备注                                     |
|--------------------|------|--------|---------|--------|-------------------------------------------|
| 批量提交       | ✓    | ✓      | ✓       | ✓      | 在单次请求中提交多个订单。 |
| 批量修改       | ✓    | ✓      | ✓       | ✓      | 在单次请求中修改多个订单。 |
| 批量取消       | ✓    | ✓      | ✓       | ✓      | 在单次请求中取消多个订单。 |

### 仓位管理

| 功能             | 现货 | 正向 | 反向 | 期权 | 备注                                    |
|---------------------|------|--------|---------|--------|------------------------------------------|
| 查询仓位     | -    | ✓      | ✓       | ✓      | 实时仓位更新。              |
| 仓位模式       | -    | ✓      | ✓       | -      | 期权仅支持单向。                |
| 杠杆控制    | -    | ✓      | ✓       | -      | 不适用于期权。              |
| 保证金模式         | -    | ✓      | ✓       | ✓      | 全仓、逐仓或组合保证金。    |

#### 对冲模式(双向持仓,BothSides)

Bybit 仅对 USDT 正向永续合约接受 `BOTH_SIDES`。对于其他产品类型,
请配置为 `MERGED_SINGLE`,或将其从 `position_mode` 中省略。按符号
进行配置:

```python
from nautilus_trader.adapters.bybit import BybitPositionMode

config = BybitExecClientConfig(
    ...,
    position_mode={"ETHUSDT-LINEAR": BybitPositionMode.BOTH_SIDES},
)
```

连接时,适配器会为每个条目调用 `/v5/position/switch-mode`,然后为
每个订单推导 `positionIdx`:开仓 BUY -> `1`(多头),开仓 SELL ->
`2`(空头),reduce-only SELL -> `1`,reduce-only BUY -> `2`。Bybit
在其 V5 [切换仓位模式](https://bybit-exchange.github.io/docs/v5/position/position-mode)
和 [下单](https://bybit-exchange.github.io/docs/v5/order/create-order#request-parameters)
API 中对此有说明:`mode=3` 启用 Both Sides,对冲模式的订单需要
`positionIdx`。

带 `positionIdx=0`(单向/Merged Single 模式)的订单和报告不携带场所
仓位 ID。对于对冲模式的索引 `1` 和 `2`,适配器将报告映射为以
`-LONG` 和 `-SHORT` 结尾的场所仓位 ID,并在 Bybit 执行消息不包含
`positionIdx` 时,将同一 ID 携带到成交上。

要覆盖此行为,通过 `params` 传入 `position_idx`:

```python
params={"position_idx": 1}  # 0 单向,1 多头,2 空头
```

### 风险事件

| 功能                   | 现货 | 正向 | 反向 | 期权 | 备注                                     |
|---------------------------|------|--------|---------|--------|-------------------------------------------|
| 强平处理      | -    | ✓      | ✓       | ✓      | 接管成交会被标记为交易所生成。 |
| ADL 处理              | -    | ✓      | ✓       | ✓      | 自动减仓成交会被标记并记录日志。   |
| ADL 排名警告         | -    | ✓      | ✓       | ✓      | 当 `adlRankIndicator >= 4` 时记录仓位报告日志。 |

Bybit 会以 `execType` 设置为以下值的方式发出场所发起的成交:

- `AdlTrade`:自动减仓执行。当保险基金无法覆盖损失时,系统会选择一个
  盈利的对手方仓位,用于平掉抵押不足的对手方仓位。
- `BustTrade`:强平接管。当保证金耗尽后,强平引擎接管了该仓位。
- `Delivery`:USDC 期货交割。
- `Settle`:反向期货结算。

适配器会将每种情况标记为交易所生成,并记录一条包含执行 ID、符号、
方向、数量和价格的警告日志。这些成交通过正常的 `FillReport` 路径
流转;由于这些订单携带空的 `orderLinkId`,执行引擎会将它们视为外部
订单,并通过 `external_order_claims`(或默认的 `EXTERNAL` 策略)
进行分配。

Bybit 还通过 `adlRankIndicator` 字段在仓位更新中发布 ADL 排名。范围
是 0(空仓/无仓位)到 5(下一个被减仓)。每当未平仓仓位的排名达到
4 或更高时,适配器都会记录一条警告日志,以便你能在场所强制平仓之前
做出反应。

上游参考资料:

- [V5 `execType` 取值](https://bybit-exchange.github.io/docs/v5/enum#exectype)
- [V5 `createType` 取值](https://bybit-exchange.github.io/docs/v5/enum#createtype)
- [强平机制](https://www.bybit.com/en/help-center/article/Liquidation-Process-Derivatives-Trading)
- [自动减仓机制](https://www.bybit.com/en/help-center/article/Auto-Deleveraging-ADL-Derivatives-Trading)

### 订单查询

| 功能             | 现货 | 正向 | 反向 | 期权 | 备注                                   |
|---------------------|------|--------|---------|--------|-----------------------------------------|
| 查询未成交订单   | ✓    | ✓      | ✓       | ✓      | 列出所有活跃订单。                 |
| 查询订单历史 | ✓    | ✓      | ✓       | ✓      | 历史订单数据。                  |
| 订单状态更新| ✓    | ✓      | ✓       | ✓      | 实时订单状态变化。          |
| 成交历史       | ✓    | ✓      | ✓       | ✓      | 执行与成交报告。             |

### 条件单

| 功能             | 现货 | 正向 | 反向 | 期权 | 备注                                   |
|---------------------|------|--------|---------|--------|-----------------------------------------|
| 订单列表         | ✓    | ✓      | ✓       | ✓      | 通过 WebSocket 以批量方式提交。     |
| OCO 订单          | ✓    | ✓      | ✓       | -      | 仅限 UI;API 用户需自行实现。  |
| 括号单      | ✓    | ✓      | ✓       | -      | 仅限 UI;API 用户需自行实现。  |
| 条件单  | ✓    | ✓      | ✓       | -      | 止损单和触及限价单。       |

### 订单参数

下单时可以使用 `params` 字典自定义各个订单:

| 参数          | 类型                   | 说明                                                             |
|--------------------|------------------------|-------------------------------------------------------------------------|
| `is_leverage`      | `bool`                 | 仅限现货。启用保证金交易(借币)。默认:`False`。        |
| `take_profit`      | `str` 或 `float`       | 止盈触发价格。为订单附加原生止盈。                    |
| `stop_loss`        | `str` 或 `float`       | 止损触发价格。为订单附加原生止损。                    |
| `tp_trigger_by`    | `str`                  | 止盈触发类型:`"LastPrice"`、`"IndexPrice"` 或 `"MarkPrice"`。       |
| `sl_trigger_by`    | `str`                  | 止损触发类型:`"LastPrice"`、`"IndexPrice"` 或 `"MarkPrice"`。       |
| `tp_order_type`    | `str`                  | 止盈执行类型:`"Market"` 或 `"Limit"`。默认:`"Market"`。        |
| `sl_order_type`    | `str`                  | 止损执行类型:`"Market"` 或 `"Limit"`。默认:`"Market"`。        |
| `tp_limit_price`   | `str` 或 `float`       | 当 `tp_order_type` 为 `"Limit"` 时的止盈限价。                   |
| `sl_limit_price`   | `str` 或 `float`       | 当 `sl_order_type` 为 `"Limit"` 时的止损限价。                   |
| `tp_trigger_price` | `str` 或 `float`       | 自定义止盈触发价格(覆盖 `take_profit`)。                      |
| `sl_trigger_price` | `str` 或 `float`       | 自定义止损触发价格(覆盖 `stop_loss`)。                        |
| `close_on_trigger` | `bool`                 | TP/SL 触发时平仓。默认:`False`。               |
| `position_idx`     | `int`                  | 对冲模式的仓位索引。参见 [对冲模式](#对冲模式双向持仓bothsides)。     |
| `bbo_side_type`    | `str`                  | 正向/反向 BBO 方向:`"Queue"` 或 `"Counterparty"`。                 |
| `bbo_level`        | `str` 或 `int`         | 正向/反向 BBO 订单簿档位:`"1"` 到 `"5"`。                     |

:::note
在 demo 中,原生 TP/SL 参数通过 HTTP 创建订单端点路由,但有一个例外:
自定义触发价格 `tp_trigger_price` 和 `sl_trigger_price` 不受支持,
因为该端点无法携带这些字段,设置它们的订单会被拒绝。`is_leverage`
参数仅适用于现货产品。参见
[Bybit 的 isLeverage 文档](https://bybit-exchange.github.io/docs/v5/order/create-order#request-parameters)。
:::

当设置了 `bbo_side_type` 和 `bbo_level` 时,Nautilus 会发送 Bybit 的
`bboSideType` 和 `bboLevel` 字段,并在 API 请求中省略订单价格。BBO
订单支持正向和反向的限价单、止损限价单以及触及限价单。

#### 示例:带原生 TP/SL 的订单

```python
order = strategy.order_factory.limit(
    instrument_id=InstrumentId.from_str("BTCUSDT-LINEAR.BYBIT"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_str("0.01"),
    price=Price.from_str("60000.0"),
    params={
        "take_profit": "65000.0",
        "stop_loss": "58000.0",
        "tp_trigger_by": "LastPrice",
        "sl_trigger_by": "LastPrice",
    },
)
strategy.submit_order(order)
```

#### 示例:BBO 订单

```python
order = strategy.order_factory.limit(
    instrument_id=InstrumentId.from_str("BTCUSDT-LINEAR.BYBIT"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_str("0.01"),
    price=Price.from_str("60000.0"),
    params={"bbo_side_type": "Queue", "bbo_level": 1},
)
strategy.submit_order(order)
```

#### 示例:现货保证金交易

```python
# 提交一个启用保证金的现货订单
order = strategy.order_factory.market(
    instrument_id=InstrumentId.from_str("BTCUSDT-SPOT.BYBIT"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_str("0.1"),
    params={"is_leverage": True}  # 为该订单启用保证金
)
strategy.submit_order(order)
```

:::note
如果参数中未设置 `is_leverage=True`,现货订单会使用你的可用余额,
不会借入资金,即使你的 Bybit 账户已启用自动借币。
:::

有关使用订单参数(包括 `is_leverage`)的完整示例,参见
[bybit_exec_tester.py](https://github.com/nautechsystems/nautilus_trader/tree/develop/examples/live/bybit/bybit_exec_tester.py)
示例。

## 现货保证金借贷与还款

NautilusTrader 提供自动化的现货保证金借款还款功能,以防止在 Bybit
上平掉空头仓位后继续产生利息。

### 背景

当以启用保证金的方式(`is_leverage=True`)进行现货交易时,Bybit 会
在你执行空头仓位时自动借入代币。然而,在你平掉空头仓位(BUY 订单
成交)之后,借入的代币**不会自动归还**——它们会继续按小时计息,直到
被手动归还。如果不处理,这可能导致相当可观的利息成本。

### 自动还款(推荐)

NautilusTrader 会在现货标的的 BUY 订单成交后立即自动归还现货保证金
借款。该功能通过 `auto_repay_spot_borrows` 配置标志**默认启用**。

**工作原理:**

1. 当某个现货 BUY 订单成交时,执行客户端会自动尝试归还该币种的任何
   未偿还借款。
2. 归还操作使用 Bybit 的 `no-convert-repay` 端点,该端点会归还全部
   未偿还的借款金额。
3. 如果归还失败(例如 API 错误),会记录错误,但不会导致执行客户端
   崩溃。
4. 在 Bybit 的 UTC 停摆窗口期间(见下文),归还操作会被自动跳过。

**示例:**

```python
from nautilus_trader.adapters.bybit import BybitExecClientConfig

config = BybitExecClientConfig(
    api_key="YOUR_API_KEY",
    api_secret="YOUR_API_SECRET",
    product_types=[BybitProductType.SPOT],
    auto_repay_spot_borrows=True,  # 默认值为 True
)
```

### 手动保证金操作

策略可以通过 `query_account` 配合 `BybitMarginAction` 枚举,直接控制
保证金借入和还款:

| 操作                                | 说明                      |
|---------------------------------------|----------------------------------|
| `BybitMarginAction.BORROW`            | 为保证金交易借入资金。 |
| `BybitMarginAction.REPAY`             | 归还借入的资金。            |
| `BybitMarginAction.GET_BORROW_AMOUNT` | 查询当前借款金额。   |

#### 借入

```python
self.query_account(
    account_id=self.account_id,
    params={"action": BybitMarginAction.BORROW, "coin": "USDT", "amount": 1000},
)
```

#### 归还

```python
# 归还指定金额
self.query_account(
    account_id=self.account_id,
    params={"action": BybitMarginAction.REPAY, "coin": "USDT", "amount": 500},
)

# 全部归还(省略 amount)
self.query_account(
    account_id=self.account_id,
    params={"action": BybitMarginAction.REPAY, "coin": "USDT"},
)
```

#### 查询借款金额

```python
self.query_account(
    account_id=self.account_id,
    params={"action": BybitMarginAction.GET_BORROW_AMOUNT, "coin": "USDT"},
)
```

:::note
`account_id` 可以从 `self.portfolio.account(BYBIT_VENUE).id` 获取,
或在策略初始化期间通过配置存储。
:::

#### 接收结果

结果作为自定义数据发布到消息总线上。在你的策略中订阅以接收它们:

```python
from nautilus_trader.adapters.bybit import BybitMarginAction
from nautilus_trader.adapters.bybit import BybitMarginBorrowResult
from nautilus_trader.adapters.bybit import BybitMarginRepayResult
from nautilus_trader.adapters.bybit import BybitMarginStatusResult
from nautilus_trader.model.data import DataType


class MyStrategy(Strategy):
    def on_start(self):
        self.subscribe_data(DataType(BybitMarginBorrowResult))
        self.subscribe_data(DataType(BybitMarginRepayResult))
        self.subscribe_data(DataType(BybitMarginStatusResult))

    def on_data(self, data):
        if isinstance(data, BybitMarginBorrowResult):
            if data.success:
                self.log.info(f"Borrowed {data.amount} {data.coin}")
            else:
                self.log.error(f"Borrow failed: {data.message}")
        elif isinstance(data, BybitMarginRepayResult):
            if data.success:
                self.log.info(f"Repaid {data.amount or 'all'} {data.coin}")
            else:
                self.log.error(f"Repay failed: {data.message}")
        elif isinstance(data, BybitMarginStatusResult):
            self.log.info(f"Borrow amount for {data.coin}: {data.borrow_amount}")
```

### UTC 停摆窗口

Bybit 每天在 **04:00-05:30 UTC** 期间会阻止 `no-convert-repay` 操作,
用于处理利息计算。NautilusTrader 会自动检测这一窗口,并跳过还款尝试,
转而记录一条警告。

在停摆窗口期间,任何 BUY 订单成交都会触发类似如下的警告:

```
Skipping borrow repayment for BTC due to Bybit blackout window (04:00-05:30 UTC daily). Will need manual repayment.
```

**重要提示:** 如果你的 BUY 订单在停摆窗口期间成交,你需要在
05:30 UTC 之后手动归还借款以停止计息,或者等待下一次在停摆窗口之外
发生的 BUY 订单成交。

### 配置选项

| 选项                    | 类型   | 默认值 | 说明                                                                 |
|---------------------------|--------|---------|-----------------------------------------------------------------------------|
| `auto_repay_spot_borrows` | `bool` | `True`  | 若为 `True`,在 BUY 订单成交后自动归还现货保证金借款。防止借入代币持续计息。归还操作在停摆窗口期间会被跳过。 |

### 重要说明

- 自动还款仅在**现货 BUY 订单**上触发,不适用于衍生品。
- 还款使用 `no-convert-repay` 端点,默认归还全部未偿还借款。
- 该功能会妥善处理 API 错误,记录失败但不会崩溃。
- 除非你的 Bybit 账户启用了自动借币,否则在开设空头仓位之前仍需手动
  借入资金。

### 现货交易限制

由于现货仓位不在场所侧被跟踪,以下限制适用于现货产品:

- *不支持* `reduce_only` 订单。
- *不支持*跟踪止损订单。

### 期权交易

Bybit 上市了以 BTC 和 ETH 为标的、以 USDT 或 USDC 结算的欧式期权。
适配器使用 `CryptoOption` 标的类型和 `-OPTION` 符号后缀。完整符号
格式参见[符号规则章节](#符号规则)。

#### 期权数据

适配器通过 WebSocket ticker 频道支持实时期权市场数据:

| 数据类型                  | 说明                                              |
|----------------------------|----------------------------------------------------------|
| 报价(买/卖)           | 每个期权合约的最优买卖价格和数量。   |
| Greeks                     | Delta、gamma、vega、theta,以及买/卖/标记隐含波动率。         |
| 标记价格                 | 每个期权合约的交易所标记价格。            |
| 指数价格                | 标的指数价格。                                  |
| 标的(远期)价格 | 用于确定平值(ATM)的按到期日远期价格。    |
| 未平仓合约              | 每个合约的未平仓合约数。                              |
| 订单簿增量          | 来自期权订单簿数据流的 L2 MBP 更新。         |

订阅按标的的 Greeks,或将它们聚合为带 ATM 相对行权价过滤的期权链
快照。订阅模式参见[期权概念指南](../concepts/options.md),分步演示
参见[期权数据教程](../tutorials/options_data_bybit.md)。NautilusTrader
根据 Bybit 按合约的期权市场数据在本地构建期权链视图。

期权没有 K 线(kline)数据。Bybit 不为该产品类型提供 K 线数据流。

#### 期权订单参数

除标准订单参数外,期权订单还接受:

| 参数  | 类型             | 说明                                              |
|------------|------------------|------------------------------------------------------------|
| `order_iv` | `str` 或 `float` | 按隐含波动率(而非价格)下单或修改订单。 |
| `mmp`      | `bool`           | 为该订单启用做市商保护(Market Maker Protection)。            |

这些参数通过 `SubmitOrder` 上的 `params` 传入。在主网上,它们通过
WebSocket 交易频道流转;在 demo 中,它们通过 HTTP 创建订单端点路由。
demo 模式不支持按 `order_iv` 修改现有订单。

#### 期权交易限制

- 按隐含波动率修改订单(`order_iv`)以及其他仅限 WS 交易的功能在
  demo 模式下不受支持。
- 杠杆不可配置。期权买方支付权利金;卖方缴纳保证金。
- 仓位模式仅支持单向。不支持对冲模式。
- 不支持条件单类型(`STOP_MARKET`、`STOP_LIMIT`、
  `MARKET_IF_TOUCHED`、`LIMIT_IF_TOUCHED`)。
- 不支持交易止损单(仓位上的 TP/SL)。
- 资金费率不适用于期权。
- 期权要求使用统一交易账户(UTA)。

### 跟踪止损

Bybit 上的跟踪止损在场所侧没有客户端订单 ID(不过有一个
`venue_order_id`)。这是因为跟踪止损与某个标的的净额仓位相关联。
在 Bybit 上使用跟踪止损时,请注意以下几点:

- 可以使用 `reduce_only` 指令
- 当与某个跟踪止损相关联的仓位被平掉时,该跟踪止损会在场所侧被自动
  “停用”(关闭)。
- 无法查询尚未开启的跟踪止损订单(在开启之前,`venue_order_id` 是
  未知的)。
- 可以在图形界面中手动调整触发价格,这会更新对应的 Nautilus 订单。

## 资金费率

适配器从
[Linear Ticker](https://bybit-exchange.github.io/docs/v5/websocket/public/ticker#linear-inverse-perpetual-response)
WebSocket 数据流接收资金费率数据。Bybit 在 ticker 更新中提供
`fundingIntervalHour` 字段,适配器用它来填充 `FundingRateUpdate`
上的 `interval` 字段。

适配器会为每个符号缓存最后一次已知的 `fundingIntervalHour`,以便
部分 ticker 更新(可能省略该字段)仍能携带正确的间隔值。

对于历史资金费率请求,适配器会从连续的资金费率时间戳中计算间隔。

## 速率限制

每次 HTTP 调用都会消耗全局令牌桶,以及任何按键的配额。当用量超出某个
桶时,请求会被自动排队,因此很少需要手动限流。

| 键 / 端点            | 限制(请求/秒) | 备注                                              |
|---------------------------|-----------------------|-----------------------------------------------------|
| `bybit:global`            | 120                  | 交易所全局每 5 秒 600 次请求的上限。               |
| `/v5/market/kline`        | 20                   | 历史数据扫描的限流略低于全局值。 |
| `/v5/market/trades`       | 24                   | 与全局配额一致。                          |
| `/v5/order/create`        | 10                   | 标准下单。                          |
| `/v5/order/cancel`        | 10                   | 单笔订单取消。                         |
| `/v5/order/create-batch`  | 5                    | 批量下单端点。                         |
| `/v5/order/cancel-batch`  | 5                    | 批量取消端点。                      |
| `/v5/order/cancel-all`    | 2                    | 全部取消,遵循 Bybit 的指导。         |

:::warning
当超出速率限制时,Bybit 会以错误码 `10016` 响应,如果请求持续无退避
地继续发送,可能会临时封锁该 IP。
:::

:::info
关于速率限制的更多详情,参见官方文档:<https://bybit-exchange.github.io/docs/v5/rate-limit>。
:::

### 数据客户端

如果未指定产品类型,则会加载并提供全部产品类型。

### 执行客户端

适配器会根据已配置的产品类型自动确定账户类型:

- **仅现货**:使用启用了借贷支持的 `CASH` 账户类型
- **衍生品或混合产品**:使用 `MARGIN` 账户类型(UTA —— 统一交易账户)

这使你能够在单个统一交易账户中同时交易现货和衍生品,这也是大多数
Bybit 用户使用的标准账户类型。

:::info
**统一交易账户(UTA)与现货保证金交易**

由于 Bybit 将新用户引导至该账户类型,大多数 Bybit 用户现在都拥有
统一交易账户(UTA)。经典账户被视为遗留类型。

对于 UTA 账户上的现货保证金交易:

- 借贷**不会自动启用**——需要显式的 API 配置
- 要通过 API 使用现货保证金,必须在参数中以 `is_leverage=True`
  提交订单(参见 [Bybit 文档](https://bybit-exchange.github.io/docs/v5/order/create-order#request-parameters))
- 如果你的 Bybit 账户启用了自动借贷/自动还款,场所会自动为这些保证金
  订单借入/归还资金
- 若未启用自动借贷,你需要通过 Bybit 的界面手动管理借贷

**重要提示**:Nautilus Bybit 适配器对现货订单默认使用
`is_leverage=False`,意味着除非你显式启用,否则它们不会使用保证金。
:::

## 手续费货币逻辑

理解 Bybit 如何确定交易手续费的货币,对于准确的记账和仓位跟踪非常
重要。手续费货币规则在现货和衍生品产品之间有所不同。

### 现货交易手续费

对于现货交易,手续费货币取决于订单方向以及该手续费是否为返佣
(maker 订单的负手续费):

#### 正常手续费(正值)

- **BUY 订单**:手续费以**基础货币**计收(例如,BTCUSDT 收取 BTC)
- **SELL 订单**:手续费以**计价货币**计收(例如,BTCUSDT 收取 USDT)

#### Maker 返佣(负手续费)

当 maker 手续费为负值(返佣)时,货币逻辑会**反转**:

- **带 maker 返佣的 BUY 订单**:返佣以**计价货币**支付(例如,
  BTCUSDT 支付 USDT)
- **带 maker 返佣的 SELL 订单**:返佣以**基础货币**支付(例如,
  BTCUSDT 支付 BTC)

:::note
**Taker 订单永远不会出现反转逻辑**,即使 maker 手续费率为负。Taker
手续费始终遵循正常的手续费货币规则。
:::

#### 示例:BTCUSDT 现货

- **以 taker 身份买入 1 BTC(0.1% 手续费)**:支付 0.001 BTC 手续费
- **以 taker 身份卖出 1 BTC(0.1% 手续费)**:支付等值 USDT 手续费
- **以 maker 身份买入 1 BTC(-0.01% 返佣)**:收到 USDT 返佣(反转)
- **以 maker 身份卖出 1 BTC(-0.01% 返佣)**:收到 BTC 返佣(反转)

### 衍生品交易手续费

对于所有衍生品产品(LINEAR、INVERSE、OPTION),手续费始终以**结算
货币**计收:

| 产品类型 | 结算货币                   | 手续费货币 |
|--------------|---------------------------------------|--------------|
| LINEAR       | USDT(通常)                      | USDT         |
| INVERSE      | 基础币种(例如,BTCUSD 为 BTC)      | 基础币种    |
| OPTION       | USDT                                  | USDT         |

### 手续费计算

当 WebSocket 执行消息未提供确切的手续费金额(`execFee`)时,适配器
按以下方式计算手续费:

#### 现货产品

- **BUY 订单**:`fee = base_quantity × fee_rate`
- **SELL 订单**:`fee = notional_value × fee_rate`
  (其中 `notional_value = quantity × price`)

#### 衍生品

- 所有衍生品:`fee = notional_value × fee_rate`

### 官方文档

有关 Bybit 手续费结构和货币规则的完整详情,参见:

- [Bybit WebSocket 私有执行](https://bybit-exchange.github.io/docs/v5/websocket/private/execution)
- [Bybit 现货手续费货币说明](https://bybit-exchange.github.io/docs/v5/enum#spot-fee-currency-instruction)

## 配置

必须在配置中为每个客户端指定产品类型。

### 数据客户端配置选项

| 选项                             | 默认值   | 说明 |
|------------------------------------|-----------|-------------|
| `api_key`                          | `None`    | API key;省略时从对应的环境变量加载。 |
| `api_secret`                       | `None`    | API secret;省略时从对应的环境变量加载。 |
| `product_types`                    | `None`    | 要启用的 `BybitProductType` 值序列;为 `None` 时加载所有产品。 |
| `instrument_provider`              | 默认   | 标的加载配置。订阅之前请使用 `load_all=True` 或 `load_ids`。 |
| `environment`                      | `None`    | Bybit 环境枚举。使用 `BybitEnvironment.MAINNET`、`BybitEnvironment.DEMO` 或 `BybitEnvironment.TESTNET`。 |
| `base_url_http`                    | `None`    | REST 基础 URL 的覆盖值。 |
| `proxy_url`                        | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `update_instruments_interval_mins` | `60`      | 标的目录刷新间隔(分钟)。 |
| `recv_window_ms`                   | `5,000`   | 已签名 REST 请求的接收窗口(毫秒)。 |
| `bars_timestamp_on_close`          | `True`    | K 线以区间的收盘(`True`)还是开盘(`False`)打时间戳。 |
| `max_retries`                      | `None`    | REST/WebSocket 恢复的最大重试次数。 |
| `retry_delay_initial_ms`           | `None`    | 重试之间的初始延迟(毫秒)。 |
| `retry_delay_max_ms`               | `None`    | 重试之间的最大延迟(毫秒)。 |
| `transport_backend`                | `Sockudo` | WebSocket 传输后端。 |

### 执行客户端配置选项

| 选项                                  | 默认值   | 说明 |
|-----------------------------------------|-----------|-------------|
| `api_key`                               | `None`    | API key;省略时从对应的环境变量加载。 |
| `api_secret`                            | `None`    | API secret;省略时从对应的环境变量加载。 |
| `product_types`                         | `None`    | 要启用的 `BybitProductType` 值序列(执行时现货不能与衍生品混合)。 |
| `instrument_provider`                   | 默认   | 标的加载配置。下单之前请使用 `load_all=True` 或 `load_ids`。 |
| `environment`                           | `None`    | Bybit 环境枚举。使用 `BybitEnvironment.MAINNET`、`BybitEnvironment.DEMO` 或 `BybitEnvironment.TESTNET`。 |
| `base_url_http`                         | `None`    | REST 基础 URL 的覆盖值。 |
| `base_url_ws_private`                   | `None`    | 私有 WebSocket 基础 URL 的覆盖值。 |
| `base_url_ws_trade`                     | `None`    | 交易 WebSocket 基础 URL 的覆盖值。 |
| `proxy_url`                             | `None`    | HTTP 与 WebSocket 传输的可选代理 URL。 |
| `use_gtd`                               | `False`   | 为 `True` 时,将 GTD 订单重映射为 GTC(Bybit 缺乏原生 GTD 支持)。 |
| `use_ws_execution_fast`                 | `False`   | 订阅低延迟执行数据流。 |
| `use_http_batch_api`                    | `False`   | 使用 Bybit 的 HTTP 批量交易 API(已弃用)。 |
| `use_spot_position_reports`             | `False`   | 为 `True` 时,将现货钱包余额报告为仓位。 |
| `auto_repay_spot_borrows`               | `True`    | 在 BUY 订单完全成交后自动归还现货保证金借款(仅限现货)。 |
| `repay_queue_interval_secs`             | `1.0`     | 处理现货借款还款队列的间隔(秒)。 |
| `ignore_uncached_instrument_executions` | `False`   | 忽略尚未缓存标的的执行消息。 |
| `max_retries`                           | `None`    | 下单/取消/修改调用的最大重试次数。 |
| `retry_delay_initial_ms`                | `None`    | 重试之间的初始延迟(毫秒)。 |
| `retry_delay_max_ms`                    | `None`    | 重试之间的最大延迟(毫秒)。 |
| `recv_window_ms`                        | `5,000`   | 已签名 REST 请求的接收窗口(毫秒)。 |
| `ws_trade_timeout_secs`                 | `5.0`     | 等待交易 WebSocket 确认的超时时间(秒)。 |
| `ws_auth_timeout_secs`                  | `5.0`     | 等待认证 WebSocket 确认的超时时间(秒)。 |
| `futures_leverages`                     | `None`    | `BybitSymbol` 到杠杆设置的映射。 |
| `position_mode`                         | `None`    | `BybitSymbol` 到仓位模式的映射。参见 [对冲模式](#对冲模式双向持仓bothsides)。 |
| `margin_mode`                           | `None`    | 账户的保证金模式设置。 |
| `transport_backend`                     | `Sockudo` | WebSocket 传输后端。 |

最常见的用法是配置一个实时 `TradingNode`,使其包含 Bybit 数据客户端
和执行客户端。为此,请在客户端配置中添加 `BYBIT` 部分:

```python
from nautilus_trader.adapters.bybit import BYBIT
from nautilus_trader.adapters.bybit import BybitEnvironment
from nautilus_trader.adapters.bybit import BybitProductType
from nautilus_trader.live.node import TradingNode
from nautilus_trader.live.node import TradingNodeConfig

config = TradingNodeConfig(
    ...,  # 省略
    data_clients={
        BYBIT: {
            "api_key": "YOUR_BYBIT_API_KEY",
            "api_secret": "YOUR_BYBIT_API_SECRET",
            "base_url_http": None,  # 使用自定义端点覆盖
            "environment": BybitEnvironment.MAINNET,
            "product_types": [BybitProductType.LINEAR],
        },
    },
    exec_clients={
        BYBIT: {
            "api_key": "YOUR_BYBIT_API_KEY",
            "api_secret": "YOUR_BYBIT_API_SECRET",
            "base_url_http": None,  # 使用自定义端点覆盖
            "environment": BybitEnvironment.MAINNET,
            "product_types": [BybitProductType.LINEAR],
        },
    },
)
```

然后,创建一个 `TradingNode` 并添加客户端工厂:

```python
from nautilus_trader.adapters.bybit import BYBIT
from nautilus_trader.adapters.bybit import BybitLiveDataClientFactory
from nautilus_trader.adapters.bybit import BybitLiveExecClientFactory
from nautilus_trader.live.node import TradingNode

# 使用配置实例化实时交易节点
node = TradingNode(config=config)

# 向节点注册客户端工厂
node.add_data_client_factory(BYBIT, BybitLiveDataClientFactory)
node.add_exec_client_factory(BYBIT, BybitLiveExecClientFactory)

# 最后构建节点
node.build()
```

### API 凭证

向 Bybit 客户端提供凭证有两种方式。可以将对应的 `api_key` 和
`api_secret` 值传给配置对象,或设置以下环境变量:

对于 Bybit 实盘客户端,可以设置:

- `BYBIT_API_KEY`
- `BYBIT_API_SECRET`

对于 Bybit demo 客户端,可以设置:

- `BYBIT_DEMO_API_KEY`
- `BYBIT_DEMO_API_SECRET`

对于 Bybit 测试网客户端,可以设置:

- `BYBIT_TESTNET_API_KEY`
- `BYBIT_TESTNET_API_SECRET`

:::tip
建议使用环境变量来管理凭证。
:::

启动交易节点时,你会立即收到关于凭证是否有效以及是否具有交易权限的
确认。

## 贡献

:::info
如需了解更多功能或为 Bybit 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
