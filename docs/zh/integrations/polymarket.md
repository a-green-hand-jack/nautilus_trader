# Polymarket

Polymarket 成立于 2020 年,是一个去中心化预测市场平台,允许交易者通过
买卖结果代币来对事件结果进行投机。

NautilusTrader 通过 Polymarket 的中央限价订单簿(CLOB)API,提供了
针对该场所的数据和执行集成。

本页文档描述的是 V2 集成。该适配器以 Rust 实现,并通过 PyO3 在
`nautilus_trader.adapters.polymarket` 中暴露给 Python;因此数据、
执行、签名和 WebSocket 操作在 Rust 和 Python 中行为一致。

NautilusTrader 支持多种 Polymarket 签名类型用于订单签名,在
NautilusTrader 处理签名和订单准备的同时,为不同的钱包配置提供了灵活性。

## 安装

安装带有 Polymarket 支持的 NautilusTrader:

```bash
uv pip install "nautilus_trader[polymarket]"
```

从源码构建并带上所有附加组件(包括 Polymarket):

```bash
uv sync --all-extras
```

## 示例

维护中的 V2 示例可在以下位置找到:Rust 版本位于
[`crates/adapters/polymarket/examples`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/adapters/polymarket/examples),
Python 版本位于
[`python/examples/polymarket`](https://github.com/nautechsystems/nautilus_trader/tree/develop/python/examples/polymarket)。

## 二元期权

[二元期权(binary option)](https://en.wikipedia.org/wiki/Binary_option)
是一种奇异期权合约类型,交易者对一个是/否命题的结果进行押注。如果
预测正确,交易者获得固定支付;否则一无所获。NautilusTrader 将
Polymarket 的结果代币表示为 `BinaryOption` 标的。

Polymarket 使用 **pUSD** 作为交易的抵押代币,详情[见下文](#pusd)。

## Polymarket 文档

Polymarket 为不同受众提供了以下资源:

- [Polymarket Learn](https://learn.polymarket.com/):面向用户的教育
  内容和指南,帮助理解平台以及如何使用它。
- [Polymarket CLOB API](https://docs.polymarket.com/trading/orders/overview):
  面向与 Polymarket CLOB API 交互的开发者的技术文档。

## 概览

本指南假设交易者需要同时配置实时市场数据源和交易执行。Polymarket
集成适配器包含多个组件,可以组合使用,也可以单独使用,视具体使用
场景而定。

- `PolymarketWebSocketClient`:底层 WebSocket API 连接(基于 Rust
  实现的 Nautilus `WebSocketClient` 基类构建)。
- `PolymarketInstrumentProvider`:面向 `BinaryOption` 标的的解析与
  加载功能。
- `PolymarketDataClient`:市场数据流管理器。
- `PolymarketExecutionClient`:交易执行网关。
- `PolymarketDataClientFactory`:Polymarket 数据客户端工厂(供实时
  节点构建器使用)。
- `PolymarketExecutionClientFactory`:Polymarket 执行客户端工厂
  (供实时节点构建器使用)。

:::note
大多数用户只需为实时交易节点定义配置(如下所示),不必直接与这些底层
组件打交道。
:::

## pUSD

**pUSD** 是 Polymarket 上用于交易的抵押代币。它是 Polygon 上一个
标准的 ERC-20 代币,由 USDC 支持。

代理合约地址是 Polygon 上的
[0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB](https://polygonscan.com/address/0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB)。
直接链上注资会通过
[CollateralOnramp](https://docs.polymarket.com/resources/contracts)
将 Polygon 上的 USDC.e(跨链桥接的 USDC)包装为 pUSD。Bridge API 也
可以从其他链存入受支持的资产,转换后记入 pUSD。

## 钱包与账户

要通过 NautilusTrader 与 Polymarket 交互,你需要一个兼容
**Polygon** 的钱包(例如 MetaMask)。

### 签名类型

Polymarket 为订单签名和验证支持多种签名类型:

| 签名类型 | 钱包类型                    | 说明                                                              | 使用场景                                                                                                   |
|----------------|--------------------------------|----------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `0`            | EOA(外部持有账户) | 由直接控制私钥的钱包生成的标准 EIP712 签名。 | **默认。** 直接钱包连接(MetaMask、硬件钱包等)。                                 |
| `1`            | Email/Magic 钱包代理       | 面向基于邮箱账户(Magic Link)的智能合约钱包。             | 与 Email/Magic 账户关联的 Polymarket 代理钱包。需要 `funder` 地址。                          |
| `2`            | 浏览器钱包代理           | 面向浏览器钱包的改良版 Gnosis Safe(1-of-1 多签)。              | 与浏览器钱包关联的 Polymarket 代理钱包。支持 UI 验证。需要 `funder` 地址。      |
| `3`            | 存款钱包                 | 面向新 API 用户的 ERC-1271 存款钱包流程。                          | 需要存款钱包 `funder`;API 凭证仍绑定到签名者。                               |

:::note
另见 Polymarket 文档中的
[代理钱包](https://docs.polymarket.com/developers/proxy-wallet)一节,
了解更多关于签名类型和代理钱包基础设施的详情。
:::

NautilusTrader 默认使用签名类型 0(EOA),但可以通过 `signature_type`
配置参数配置为使用任何受支持的签名类型。

使用环境变量时,每个 trader 实例只支持一个钱包地址;也可以通过配置
多个 `PolymarketExecutionClient` 实例来支持多个钱包。

:::note
请确保你的钱包已注入 **pUSD**,否则在提交订单时会遇到 "not enough
balance or allowance" 的 API 错误。
:::

### 为 Polymarket 合约设置授权额度

在开始交易之前,你需要确保你的钱包已为 Polymarket 的智能合约设置了
授权额度(allowances)。你可以通过运行位于
`nautilus_trader/adapters/polymarket/scripts/set_allowances.py` 的
脚本来完成此操作。

该脚本改编自 @poly-rodr 创建的一个
[gist](https://gist.github.com/poly-rodr/44313920481de58d5a3f6d1f8226bd5e)。

:::note
每个 EOA 钱包运行一次相应的授权命令,之后当 Polymarket 变更所需合约
时再重新运行。
:::

:::warning
[Polymarket 将于 2026 年 7 月 17 日 00:00 UTC(10:00 AEST)停用
CLOB v1 Neg Risk Adapter](https://docs.polymarket.com/changelog)。
现有钱包不会自动批准其 v2 替代版本。对于每个现有钱包,请在截止日期
之前运行你所部署环境使用的 Python 脚本或 Rust 二进制程序。这些命令
不会撤销旧的 v1 授权;请在审查完任何遗留流程后,将撤销操作作为一次
单独的链上操作处理。对于给定钱包,一次只运行一个授权命令。
:::

该脚本自动化了为 Polymarket 合约批准必要授权额度的过程。它为 pUSD
抵押代币和 Conditional Token Framework(CTF)合约设置授权,使
Polymarket CLOB Exchange 能够操作你的资金。

运行脚本之前,请确保满足以下前置条件:

- 安装 web3 Python 包:`uv pip install "web3==7.12.1"`。
- 拥有一个已注入部分 POL(用于支付 gas 费)的兼容 **Polygon** 的
  钱包。
- 在你的 shell 中设置以下环境变量:
  - `POLYGON_PRIVATE_KEY`:你兼容 **Polygon** 钱包的私钥。
  - `POLYGON_PUBLIC_KEY`:你兼容 **Polygon** 钱包的公钥。

准备好这些之后,脚本会:

- 为 Polymarket 抵押代币合约批准可能的最大 pUSD 额度(使用
  `MAX_UINT256` 值)。
- 为 CTF 合约设置授权,使其能够为交易目的操作你的账户。

:::note
你也可以在脚本中调整授权额度,而不使用 `MAX_UINT256`,以
**pUSD** 的*分数单位*指定金额,不过这一用法尚未经过测试。
:::

在运行脚本之前,请确保你的私钥和公钥已正确存储在环境变量中。以下是
在终端会话中设置变量的示例:

```bash
export POLYGON_PRIVATE_KEY="YOUR_PRIVATE_KEY"
export POLYGON_PUBLIC_KEY="YOUR_PUBLIC_KEY"
```

使用以下命令运行脚本:

```bash
python nautilus_trader/adapters/polymarket/scripts/set_allowances.py
```

对于 Rust v2 适配器,设置 `POLYMARKET_PK`,然后运行:

```bash
cargo run -p nautilus-polymarket --bin polymarket-set-allowances
```

两个命令都会批准当前位于
`0xadA2005600Dec949baf300f4C6120000bDB6eAab` 的 Neg Risk Adapter。
两个命令默认都使用 `https://polygon.drpc.org`;设置 `POLYGON_RPC_URL`
可使用其他 Polygon RPC 端点。

### 脚本详解

该脚本执行以下操作:

- 通过 RPC URL(<https://polygon.drpc.org>)连接到 Polygon 网络。
- 签名并发送一笔交易,为 Polymarket 合约批准最大 pUSD 授权额度。
- 为 CTF 合约设置授权,使其能够代表你管理 Conditional Tokens。
- 对 Polymarket CLOB Exchange、Neg Risk CTF Exchange 以及当前的
  Neg Risk adapter 重复上述授权流程。

这使 Polymarket 能够在执行交易时操作你的资金,并确保与 CLOB
Exchange 的顺畅集成。

## API keys

要在 Polymarket 上交易,你需要生成 API 凭证。请按以下步骤操作:

1. 确保设置了以下环境变量:
   - `POLYMARKET_PK`:用于签署交易的私钥。
   - `POLYMARKET_FUNDER`:**Polygon** 网络上用于为 Polymarket 交易
     提供资金的钱包地址(公钥)。

2. 使用以下命令运行脚本:

   ```bash
   python nautilus_trader/adapters/polymarket/scripts/create_api_key.py
   ```

脚本会生成并打印 API 凭证,你应该将其保存到以下环境变量中:

- `POLYMARKET_API_KEY`
- `POLYMARKET_API_SECRET`
- `POLYMARKET_PASSPHRASE`

这些随后可用于 Polymarket 客户端配置:

- `PolymarketDataClientConfig`
- `PolymarketExecClientConfig`

## 配置

在设置 NautilusTrader 以配合 Polymarket 使用时,正确配置必要的参数
(尤其是私钥)至关重要。

**关键参数**:

- `private_key`:用于签署订单的钱包私钥。其解释方式取决于你的
  `signature_type` 配置。如果配置中未显式提供,会自动从
  `POLYMARKET_PK` 环境变量读取。
- `funder`:用于为交易提供资金的 **pUSD** 注资钱包地址。如果未提供,
  会从 `POLYMARKET_FUNDER` 环境变量读取。
- API 凭证:你需要提供以下 API 凭证才能与 Polymarket CLOB 交互:
  - `api_key`:如果未提供,会从 `POLYMARKET_API_KEY` 环境变量读取。
  - `api_secret`:如果未提供,会从 `POLYMARKET_API_SECRET` 环境变量
    读取。
  - `passphrase`:如果未提供,会从 `POLYMARKET_PASSPHRASE` 环境变量
    读取。
  API 凭证由用于 L2 认证的私钥签名者创建。对于 `POLY_1271`,存款
  钱包仍是 `funder`,但它不是 L2 认证地址。
- `auto_load_missing_instruments`(默认 `True`):控制针对不在缓存中
  的标的的订阅和请求命令,是否触发通过 Gamma API 进行的即时加载。
  禁用时,订阅一个未缓存的标的会返回错误。参见
  [运行时标的加载](#运行时标的加载)。
- `auto_load_debounce_ms`(默认 `100`):并发的自动加载请求被合并为
  一次批量 Gamma 调用的时间窗口(毫秒)。

:::tip
建议使用环境变量来管理凭证。
:::

## 订单能力

与传统交易所相比,Polymarket 作为一个预测市场,其订单类型和指令的
集合更为有限。

### 订单类型

| 订单类型             | 二元期权 | 备注                                                                     |
|------------------------|----------------|---------------------------------------------------------------------------|
| `MARKET`               | ✓              | **BUY 订单需要以计价货币计量数量**,SELL 订单需要以基础货币计量数量。 |
| `LIMIT`                | ✓              |                                                                           |
| `STOP_MARKET`          | -              | *Polymarket 不支持*。                                            |
| `STOP_LIMIT`           | -              | *Polymarket 不支持*。                                            |
| `MARKET_IF_TOUCHED`    | -              | *Polymarket 不支持*。                                            |
| `LIMIT_IF_TOUCHED`     | -              | *Polymarket 不支持*。                                            |
| `TRAILING_STOP_MARKET` | -              | *Polymarket 不支持*。                                            |

### 数量语义

Polymarket 根据订单类型*和*方向,以不同方式解释订单数量:

- **限价单**将 `quantity` 解释为条件代币的数量(基础单位)。
- **市价 SELL** 订单同样使用基础单位数量。
- **市价 BUY** 订单将 `quantity` 解释为以 **pUSD** 计的计价货币
  名义金额。

因此,以基础货币计量的数量提交的市价买单,其实际执行的规模会远远
超出预期。

提交市价 BUY 订单时,请在订单上设置 `quote_quantity=True`。适配器
会在发布到 CLOB 之前,将计价货币金额(pUSD)转换为已签名的基础单位
份额数量。Polymarket 执行客户端会拒绝以基础货币计量的市价买单,以
防止意外成交。

```python
# 以计价货币计量数量的市价 BUY(花费 10 pUSD)
order = strategy.order_factory.market(
    instrument_id=instrument_id,
    order_side=OrderSide.BUY,
    quantity=instrument.make_qty(10.0),
    time_in_force=TimeInForce.IOC,  # 映射为 Polymarket FAK
    quote_quantity=True,  # 解释为 pUSD 名义金额
)
strategy.submit_order(order)
```

### 执行指令

| 指令   | 二元期权 | 备注                                                |
|---------------|----------------|------------------------------------------------------|
| `post_only`   | ✓              | 仅支持带 `GTC` 或 `GTD` 的限价单。 |
| `reduce_only` | -              | *Polymarket 不支持*。                       |

### 有效期选项

Polymarket 将 `POST /order` 中的字段称为 `orderType`。在
NautilusTrader 中,它映射为 `TimeInForce`。有效组合取决于 Nautilus
订单类型:

| Nautilus 有效期 | Polymarket `orderType` | Nautilus 订单范围 | 备注 |
|--------------|------------------------|----------------------|-------|
| `GTC`        | `GTC`                  | 仅限 `LIMIT`         | 撤销前有效;挂在订单簿上。 |
| `GTD`        | `GTD`                  | 仅限 `LIMIT`         | 指定日期前有效;挂单直到过期、成交或取消。 |
| `FOK`        | `FOK`                  | `LIMIT` 或 `MARKET`  | 立即全部成交,否则取消整笔订单。 |
| `IOC`        | `FAK`                  | `LIMIT` 或 `MARKET`  | 立即成交可用数量,取消剩余部分。 |

:::note
Polymarket 使用 `FAK`(Fill-And-Kill)来表示 NautilusTrader 所称的
`IOC`(立即成交或取消)语义。Polymarket 文档将 `FOK` 和 `FAK`
归类为市价订单类型,而 `GTC` 和 `GTD` 是限价订单类型。对于
Nautilus 的 `MARKET` 订单,适配器仅接受 `IOC` 和 `FOK`;`GTC` 和
`GTD` 仅对挂单型 `LIMIT` 订单有效。
:::

:::note
一个可成交订单(任何 `FOK`/`FAK` 订单,或一个会吃单的 `BUY`)的
名义价值必须至少为 **1 pUSD**,否则场所会以
`invalid amount for a marketable BUY order … min size: $1` 拒绝它。
挂单型 `GTC`/`GTD` 限价单只受 5 份最小数量的限制。
:::

:::note
场所会将 `GTD` 到期报告为一个 `OrderCanceled` 事件(而非
`OrderExpired`),且 Polymarket 内部有一个约一分钟的过期缓冲,因此
`GTD` 订单实际挂单时间会比请求的时长少约一分钟,场所才会将其取消。
:::

### 高级订单功能

| 功能            | 二元期权 | 备注                              |
|--------------------|----------------|-------------------------------------|
| 订单修改 | -              | 仅提供取消功能。   |
| 括号单/OCO 订单 | -              | *Polymarket 不支持。*     |
| 冰山订单     | -              | *Polymarket 不支持。*     |

### 批量操作

| 操作    | 二元期权 | 备注                                                                                                                           |
|--------------|----------------|-----------------------------------------------------------------------------------------------------------------------------------|
| 批量提交 | ✓              | 适配器使用 `POST /orders` 处理独立的限价单批量(每次请求最多 15 笔订单)。参见 [批量提交](#批量提交)。 |
| 批量修改 | -              | *Polymarket 不支持*。                                                                                                  |
| 批量取消 | ✓              | 适配器使用 `DELETE /orders`。                                                                                             |

#### 批量提交

`SubmitOrderList` 命令会被路由到 Polymarket 的 `POST /orders` 端点。
该端点每次请求最多接受 15 笔订单(`BATCH_ORDER_LIMIT`);更大的列表
会被拆分为连续的 15 笔一组的分块。

- 只有 `LIMIT` 订单会被批量处理。列表中的 `MARKET` 订单会被路由到
  单笔订单路径,该路径会签署一个可成交订单,并根据 Nautilus 的
  `time_in_force` 以 `FAK` 或 `FOK` 提交。
- `reduce_only` 订单、`quote_quantity` 订单,以及带市价有效期
  (`IOC` 或 `FOK`)的 `post_only` 会在提交前被拒绝。
- 单笔符合条件的订单会走 `POST /order` 路径,从而保留单笔订单的重试
  语义;批量路径有意禁用了重试,因为场所没有暴露幂等性密钥。
- 如果批量响应遗漏了某条腿,该订单会保持在已提交状态等待对账。
  适配器会注册已签名订单的预期哈希,以便后续的 WebSocket 事件和
  取消操作仍能解析回本地订单。响应遗漏本身无法证明场所已拒绝该订单。
- `BatchCancelOrders` 会一次性分发给 `DELETE /orders`。

### 提交错误处理

Polymarket 的公开文档将
[`POST /order`](https://docs.polymarket.com/api-reference/trade/post-a-new-order)
成功响应描述为包含 `success`、`orderID`、`status` 和 `errorMsg`,
并将 [API 错误](https://docs.polymarket.com/resources/error-codes)
记录为结构化错误响应。它没有将无状态的客户端异常或传输失败记录为
场所拒绝。

只有当响应证明该订单未被接受时(例如 `success=false`、一个已记录的
订单处理错误,或另一个不可重试的客户端/API 错误),适配器才会拒绝
该订单。传输失败、超时、模糊的重试耗尽、无状态的
`PolyApiException`、格式错误的响应,以及服务端故障,都会使订单保持
在已提交状态。批量端点会将被拒绝的一条腿报告为 `success=true`、
`orderID` 为空,原因在 `errorMsg` 中(例如场所无法接受的裸卖单):
适配器会以场所给出的原因拒绝该条腿。没有 `orderID` 也没有原因的一条
腿会保持在已提交状态等待对账。

一旦任何单笔订单提交尝试出现模糊结果,后续的重试错误就无法证明首次
尝试失败。因此,即使后续尝试返回了一个客户端错误(例如订单已存在),
适配器仍会保持该订单为已提交状态。

在适配器发送 `POST /order` 之前发生的失败,会发出 `OrderDenied`,
而非 `OrderRejected`。这包括为调整市价 BUY 手续费所需的 pUSD 余额
查询失败的情况。

当拒绝原因报告一个 post-only 订单会吃单时,`OrderRejected` 事件会
设置 `due_post_only=true`,以便策略能够将其与其他场所拒绝区分开。

对于未知结果,适配器会尽可能从已签名的 EIP-712 订单中推导出预期的
Polymarket 订单哈希,并将其缓存为 `VenueOrderId`。后续的 WebSocket
订单事件(或对账报告)随后会关联到本地的 `ClientOrderId`,而不会
变成外部订单。

以计价货币计量数量的市价 BUY 订单,在未知结果路径上仍会应用已签名的
计价货币到基础货币的数量更新。在提交结果未知期间请求的取消会被延迟,
直到已知预期的场所订单 ID,成交跟踪会在该 ID 下注册。

### 仓位管理

| 功能          | 二元期权 | 备注                             |
|------------------|----------------|------------------------------------|
| 查询仓位  | ✓              | 来自 Polymarket Data API 的当前用户仓位。 |
| 仓位模式    | -              | 仅限二元结果仓位。    |
| 杠杆控制 | -              | 无可用杠杆。            |
| 保证金模式      | -              | 无保证金交易。            |

### 订单查询

| 功能              | 二元期权 | 备注                          |
|----------------------|----------------|--------------------------------|
| 查询未成交订单    | ✓              | 仅限活跃订单。            |
| 查询订单历史  | ✓              | 有限的历史数据。       |
| 订单状态更新 | ✓              | 实时订单状态变化。 |
| 成交历史        | ✓              | 执行与成交报告。    |

### 条件单

| 功能            | 二元期权 | 备注                               |
|--------------------|----------------|-------------------------------------|
| 订单列表        | -              | 存在独立的订单批量,但没有关联的条件单语义。 |
| OCO 订单         | -              | *Polymarket 不支持*。      |
| 括号单     | -              | *Polymarket 不支持*。      |
| 条件单 | -              | *Polymarket 不支持*。      |

### 精度限制

Polymarket 根据最小变动单位和 `orderType` 强制执行不同的精度约束。

**二元期权标的**通常最多支持 6 位小数的数量(最小变动单位为
0.0001),但**市价单(`FAK` 和 `FOK`)有更严格的精度要求**:

- **市价订单类型(`FAK` 和 `FOK`):**
  - 直接的 maker 金额被限制在**2 位小数**。
  - 计算得出的 taker 金额使用市场最小变动单位精度加上两位数量小数。
  - 以 `FAK` 或 `FOK` 提交的限价单也必须满足更严格的市价单金额校验。
    场所会拒绝对挂单型订单有效、但对该市价单类型无效的数值。

- **挂单型限价订单类型(`GTC` 和 `GTD`):** 精度更灵活,基于市场
  最小变动单位。

### 最小变动单位精度层级

| 最小变动单位 | 价格小数位 | 数量小数位 | 金额小数位 |
|-----------|----------------|---------------|-----------------|
| 0.1       | 1              | 2             | 3               |
| 0.01      | 2              | 2             | 4               |
| 0.001     | 3              | 2             | 5               |
| 0.0001    | 4              | 2             | 6               |

:::note

- 适配器在签名之前会校验最小变动单位。对于以 `FAK` 或 `FOK` 提交的
  限价单,CLOB 仍然是订单类型特定金额精度的权威来源。
- 适配器会在签名之前拒绝超出当前市场 `tick_size` 到 `1 - tick_size`
  范围的限价。
- 市价单精度限制包括卖出数量的两位小数,以及由最小变动单位推导的
  计算金额边界。
- 在市场条件下(尤其是当市场变得单边时),最小变动单位可能会动态
  变化。

:::

### 最小变动单位变化处理

当某个市场的最小变动单位发生变化时(`tick_size_change` WebSocket
事件),旧的订单簿档位在新网格上可能失效(例如,`0.505` 符合
`0.001` 的最小变动单位,但不符合 `0.01` 的最小变动单位)。为了将
旧网格价格排除在新纪元之外,适配器将该变化视为一次订单簿纪元切换:

1. 发布带有新 `price_increment` 和 `price_precision` 的更新后
   `BinaryOption`。
2. 丢弃该标的的本地订单簿。
3. 将该标的标记为等待全新快照。
4. 在快照到达之前丢弃增量的 `price_change` 订单簿更新。
5. 从快照重新填充订单簿,并恢复正常处理。

成交 tick 和标的更新流程不受影响。报价处理遵循
`drop_quotes_missing_side`:启用时,报价 tick 要求同时存在买价和
卖价;禁用时,缺失的一侧使用 Polymarket 边界价格加零数量。适配器
可以通过从每次 `price_change` 中读取 `best_bid` 和 `best_ask`,在
间隙期间保持报价持续流转。

## 成交

Polymarket 上的成交可以有以下状态:

- `MATCHED`:成交已匹配,并已发送给执行者服务。执行者将其作为一笔
  交易提交给 Exchange 合约。
- `MINED`:观察到该成交已被打包上链,但尚未建立最终性阈值。
- `CONFIRMED`:成交已达到较强的概率性最终性,且成功。
- `RETRYING`:成交交易已失败(回滚或重组),运营方正在重试/重新提交。
- `FAILED`:成交已失败,不再重试。

一旦某笔成交首次被匹配,后续的状态更新会通过用户 WebSocket 到达。
执行适配器会在 `MATCHED` 时发出一个 `OrderFilled`。它将 `MINED` 和
`RETRYING` 视为结算更新,不会发出另一个成交。`CONFIRMED` 记录最终性
并刷新账户。如果成交达到 `FAILED`,适配器会为每一笔本地已应用的成交
发出一个 `OrderFillVoided`,并刷新账户。该修正不会重新挂出失败的
数量,但会保留任何已在运行中的 maker 订单剩余部分。一个执行完成的
订单会变为 `VOIDED`。已匹配的 WebSocket 成交会在 `OrderFilled` 事件
的 `info` 字段中保留原始成交字段。

### 成交 ID 推导

Polymarket 在 `last_trade_price` 市场数据事件上不发布成交 ID。
适配器通过 Rust 的 `determine_trade_id` 函数,使用 FNV-1a 从资产
ID、方向、价格、数量和时间戳中推导出一个确定性的 `TradeId`。对于
执行成交,taker 报告在 REST 对账和用户 WebSocket 中都使用场所的
成交 `id`,因此同一笔成交在不同来源之间能够去重。一笔 maker 成交
可能会成交用户不止一个挂单,因此 maker 报告会将场所成交 ID 与
maker 场所订单 ID 结合使用。相同的场所事件在多次回放中会产生相同的
成交 ID。对于历史 Data API 成交,加载器使用
`{transactionHash[-24:]}-{asset[-4:]}-{seq:06d}` 来区分同一笔交易中
的多笔成交。

## 手续费

Polymarket 使用公式 `fee = C * feeRate * p * (1 - p)`,其中 C 是
成交份额数,p 是份额价格。手续费在 p = 0.50 时达到峰值,并向两端
对称递减。只有 taker 支付手续费;maker 不支付。

| 类别        | Taker `feeRate` | Maker `feeRate` | Maker 返佣 |
|-----------------|-----------------|-----------------|--------------|
| 加密货币          | 0.072           | 0               | 20%          |
| 体育          | 0.03            | 0               | 25%          |
| 金融         | 0.04            | 0               | 25%          |
| 政治         | 0.04            | 0               | 25%          |
| 经济学       | 0.05            | 0               | 25%          |
| 文化         | 0.05            | 0               | 25%          |
| 天气         | 0.05            | 0               | 25%          |
| 其他 / 综合 | 0.05            | 0               | 25%          |
| 提及类        | 0.04            | 0               | 25%          |
| 科技         | 0.04            | 0               | 25%          |
| 地缘政治     | 0               | 0               | -            |

手续费以 USDC 计算,四舍五入到小数点后 5 位,并在协议成交时应用。
收取的最小手续费为 0.00001 USDC;更小的手续费会四舍五入为零。

:::note
最新费率参见 Polymarket 的
[手续费](https://docs.polymarket.com/trading/fees)文档。
:::

### 回测手续费模型

对于回测,适配器提供了 `PolymarketFeeModel`(一个
`nautilus_trader.backtest.models.FeeModel` 子类),它应用上述 taker
手续费公式,并根据市场类别为被动的 maker 成交计入推断出的返佣。
Polymarket 在加密货币市场支付 20% 的 maker 返佣,在其他启用手续费的
类别(体育、金融、政治、经济学、文化、天气、科技、提及类、其他)
支付 25%,每日从各市场的返佣池中分发。地缘政治市场免手续费,没有
返佣,该模型对其返回零。

```python
from nautilus_trader.adapters.polymarket.fee_model import PolymarketFeeModel

# 默认:启用 maker 返佣
fee_model = PolymarketFeeModel()

# 或用于仅 taker 的策略
fee_model = PolymarketFeeModel(maker_rebates_enabled=False)
```

该模型也可以通过 `ImportableFeeModelConfig` 和
`PolymarketFeeModelConfig`,经由 `BacktestVenueConfig.fee_model`
进行配置。Maker 返佣份额的推断首先使用标的的类别标签,若标签缺失,
则回退到文档记录的按类别手续费率。

## 对账

Polymarket API 会根据查询方式,返回所有**活跃**(未成交)订单,或
按 Polymarket 订单 ID(`venue_order_id`)返回特定订单。Polymarket
的执行对账流程如下:

- 为 Polymarket 报告的所有具有活跃(未成交)订单的标的生成订单报告。
- 从 Polymarket Data API 报告的当前用户仓位生成仓位报告。
- 将这些报告与 Nautilus 执行状态进行比较。
- 生成缺失的订单,使 Nautilus 执行状态与 Polymarket 报告的仓位保持
  一致。

Polymarket 不会直接返回已不再活跃的订单。当某个订单的终态 WebSocket
更新被遗漏时,V2 适配器会从成交历史中恢复其缓存的单笔订单。只有
`CONFIRMED` 的成交才会计入恢复的成交;待处理和失败的结算状态不会。

全量状态对账会将每份订单报告与其场所成交报告配对。它会先应用真实
成交以保留成交 ID 和手续费,然后仅推断达到场所报告状态所需的任何
剩余数量。REST 订单报告将已匹配数量限制在本地已应用成交与已认证
`CONFIRMED` 成交历史两者中较大者,因此待处理的结算不会产生推断出的
成交。当场所报告的已匹配数量超过本地订单和 WebSocket 成交跟踪器所
含数量时,运行时订单检查会获取已确认的成交历史。未配对的成交报告
保留正常的仅成交路径。

### 从成交中恢复单笔订单

`/data/order/{id}` 只返回活跃订单,因此一个 `Filled` 或 `Canceled`
的订单会返回一个空响应。为避免引擎将本地的 `ACCEPTED` 订单解析为
`REJECTED`(这会丢弃已在场所发生的成交),`generate_order_status_report`
会回退到按场所订单 ID 过滤的 `/data/trades`。缓存的订单通过
`client_order_id` 解析,如果只知道场所 ID,则回退到缓存的
`venue_order_id` 索引。恢复以缓存的订单为键;如果没有缓存订单,
恢复会推迟给引擎,而不是仅凭成交历史合成一个外部订单:

- 有缓存订单 + 恢复的成交覆盖了缓存的数量(在 `DUST_SNAP_THRESHOLD`
  容差内,以应对 CLOB 分币级别的最小变动单位截断):返回 `Filled`。
  引擎会通过推断成交对账任何超出缓存 `filled_qty` 的差额。
- 有缓存订单 + 恢复的成交比缓存数量少了超过 dust 容差:返回
  `Canceled`,并附带恢复出的 `filled_qty`。引擎的 CANCELED 分支会
  在缓存的 `filled_qty` 处转换订单,因此在这种罕见的部分取消场景
  下,任何仅通过 REST(而非 WS)到达的新恢复成交都不会被应用。
  相比让订单卡在未成交状态,更倾向于关闭它;如果这种场景下需要精确
  的成交元数据,可以手动查阅场所的成交历史。
- 有缓存订单,无成交记录:返回 `Canceled`,带
  `cancel_reason="ORDER_NOT_FOUND_AT_VENUE"`。
- 有缓存订单,存在任何 `MATCHED`、`MINED` 或 `RETRYING` 状态的
  成交:单笔订单查询会保留本地已应用的已匹配数量,而终态 REST 恢复
  会等待 `CONFIRMED` 或 `FAILED`。
- 无缓存订单(无论是否存在成交):返回 `None`;由引擎的
  未在场所找到路径解析本地条目。

对于被 `GET /orders` 遗漏的已匹配订单,批量的未成交订单检查无法使用
此回退机制。在默认的 `open_check_open_only=true` 下,引擎会将这些
缓存订单保持未成交状态,等待后续对账。在
`open_check_open_only=false` 下,缺失订单重试可能在其待处理结算
确认之前就将该订单标记为已拒绝。单笔订单查询或下一次启动对账会从
已确认的成交历史中恢复已结算的数量。

## 成交数量规范化

Polymarket 的传输层金额使用 6 位小数的定点尾数。市价 SELL 签名会将
以份额计的 `makerAmount` 截断为 2 位小数,而市价 BUY 的计价货币转换
可能会在已注册和已成交数量之间留下几个微份额的漂移。这两种效应在
绝对份额层面都是固定的,因此适配器使用
`DUST_SNAP_THRESHOLD = 0.01` 份额。达到或超过该阈值的仍视为真实的
部分成交或超额成交。

| 方向 | 来源                                         | 适配器行为                              |
|-----------|------------------------------------------------|------------------------------------------------|
| 超额成交  | 市价 BUY 计价货币转换(微份额)      | 将成交向下钳制至 `submitted_qty`              |
| 成交不足 | 已签名或场所数量截断(`< 0.01`)  | 规范化原子 FOK;取消 FAK 剩余部分  |

终态数量规范化由挂单型 maker 订单的 `MATCHED` 订单更新触发,或对于
原子 FOK 订单则直接由确认的 taker 成交触发。它会发出一个对账用的
`OrderUpdated`,将订单数量降为累计场所成交量。它不会发出成交,也不会
改变仓位、余额或手续费。

IOC 映射为场所的 FAK。一旦某笔 taker 成交得到确认,`original_size`
和 `size_matched` 之间的任何正差值都是场所已经杀掉的未成交剩余部分。
因此适配器会在真实成交之后发出 `OrderCanceled`,而不是规范化数量或
让订单保持部分成交状态。当某个 `MATCHED` 的 FAK 订单有
`size_matched < original_size` 时,REST 报告应用相同规则。当确认的
成交在提交响应之前到达时,同样的终态处理会在缓冲的成交清空之后运行。
缓冲的 `Canceled`、`Expired` 或 `Rejected` 报告优先。

`FillReport.commission` 始终反映场所报告的数量,而非钳制后的数量。
这几个最小单位(ulp)级别的差异在 pUSD 中是次微分级别的。

成交跟踪器以 `venue_order_id` 为键,在订单被接受时注册,因此在另一
个会话中下达的订单的成交报告会原样通过。`DUST_SNAP_THRESHOLD` 不能
按策略配置;它位于 `nautilus_polymarket::common::consts` 中。

## WebSocket

`PolymarketWebSocketClient` 构建在 Rust 实现的高性能 Nautilus
`WebSocketClient` 基类之上。

### 数据

数据适配器会随着标的被请求而动态打开 `market` 订阅。目前它使用一个
市场 WebSocket 连接。`ws_max_subscriptions` 配置字段是存在的,但
V2 尚未强制执行它,也不会跨连接分片订阅。

一个 `price_change` 载荷可能包含多个资产交错的更新。适配器会按标的
对更新进行分组,为每个标的发布一批原子的订单簿增量,而报价处理则
保持在场所载荷的原始顺序。

### 运行时标的加载

Polymarket 列出了数千个活跃市场,并且新市场在一天中随时可能出现,
因此在启动时预加载完整的市场宇宙很少可行。数据适配器会按需自动加载
缺失的标的,使策略能够订阅不在缓存中的市场:

- 当策略对一个未缓存的标的发出 `subscribe_quote_ticks`、
  `subscribe_trade_ticks`、`subscribe_order_book_deltas` 或
  `request_instrument` 时,适配器会注册该请求,并等待
  `auto_load_debounce_ms`(默认 100 毫秒),以便并发请求能够合并。
- 然后它会发出一次批量的 Gamma API 调用。超过 Gamma
  `condition_ids` 查询上限(约 100)的批次会被拆分为多次调用并合并。
- 一旦标的加载完成,它们会被发布给数据引擎(填充缓存),延迟的订阅
  会原子性地打开其 WebSocket 订阅。如果某个策略在自动加载进行期间
  取消订阅,不会看到一个虚假打开的订阅。

此功能默认启用。可以通过在 `PolymarketDataClientConfig` 上设置
`auto_load_missing_instruments=False` 来禁用。要改为在启动时预加载
一组已知的市场,请在 `PolymarketInstrumentProviderConfig` 上提供
`load_ids`、`event_slugs`、`market_slugs` 或 `event_slug_builder`。

新铸造的市场会经历一个持续数分钟的 CLOB 生效窗口期,在此期间 Gamma
报告 `active=true`,但 `GET /markets/{cid}` 会返回 404,或返回带有
空 `token_id` 字符串的 200。适配器将这些情况归类为瞬态,并以带抖动
的有界指数退避重试自动加载。可以通过 `auto_load_max_retries`
(默认 12)、`auto_load_retry_delay_initial_secs`(默认 5.0)和
`auto_load_retry_delay_max_secs`(默认 15.0)调整节奏;默认值将
重试窗口限制在约 3 分钟内。设置 `auto_load_max_retries=0` 可禁用
重试。5 分钟的市场(例如涨跌加密货币市场)可能在场所完成生效之前
就已到期,请为此预留预算或提高上限。重试预算耗尽后,一个在 Gamma
上仍然缺失的条件会被记录为终态遗漏,调用方必须在市场变为可用后
重新订阅。

### 市场解析事件

Rust 数据客户端在 `condition_id` 级别跟踪 Polymarket 敞口,以便当
场所解析该市场时,YES 和 NO 两条腿能够同时平仓。仓位事件会将已开仓
的 Polymarket 二元期权标的添加到内部观察列表中。一旦某个被观察的
条件到期,数据客户端会等待 `resolve_poll_grace_secs`,然后每隔
`resolve_poll_interval_secs` 轮询 Gamma,直到该条件解析完成,或经过
`resolve_poll_max_wait_secs`。

解析使用严格的赢家推断:

- Gamma 必须返回一个已关闭的二元市场,恰好包含两个代币 ID、两个
  结果,以及二元的 `outcomePrices` 形态。
- 如果 Gamma 未对该条件提供严格的结果,客户端会回退到 CLOB 的
  `GET /markets/{condition_id}`,并使用 `tokens[].winner`。
- 非二元、模糊、格式错误或仍未解析的载荷会被跳过。它们会保留在观察
  列表中,直到轮询窗口超时,或一次手动请求解析它们。

当客户端应用一次解析时,它会为每条被跟踪的腿发出一个 `InstrumentStatus`
关闭事件和一个 `InstrumentClose` 事件。获胜的腿以 `1` 关闭,失败的
腿以 `0` 关闭。关闭类型为 `InstrumentCloseType.ContractExpired`。
该事件会关闭 Nautilus 层面的敞口,不会在链上兑现代币或索取资金。

同一套应用路径处理 WebSocket 的 `market_resolved` 事件、自动轮询和
手动请求。经过 `resolve_poll_max_wait_secs` 之后,自动轮询会暂停
被观察的条件,并记录以供手动恢复。手动请求随后仍可以重试该条件。

#### 手动解析请求

使用带 `PolymarketResolveRequest` 数据类型的 `request_data()` 来
强制进行一次解析检查。该请求接受以下任意参数:

| 参数            | 类型                 | 说明 |
|------------------|----------------------|-------------|
| `condition_id`   | `str`                | 解析一个 Polymarket 条件。 |
| `condition_ids`  | `str` 或 `list[str]` | 解析一个或多个 Polymarket 条件。 |
| `instrument_ids` | `str` 或 `list[str]` | 解析 Polymarket 标的 ID;其他场所会被忽略。 |

如果请求省略了所有选择器,客户端会使用观察列表。启用自动轮询时,
回退会选择已暂停或已超时的条目。禁用自动轮询时,会选择所有已到期
的合格条目,以便运维人员能够手动运行恢复流程。

响应载荷是一个具有以下字典结构的自定义数据:

| 键                          | 含义 |
|------------------------------|---------|
| `requested_condition_ids`    | 该请求检查的去重后条件 ID。 |
| `fetched_markets`            | 批量查询中返回的 Gamma 市场。 |
| `resolved_markets`           | 具有严格 Gamma 结果或成功 CLOB 回退结果的条件。 |
| `skipped_non_binary_markets` | 因非二元或模糊解析形态而被跳过的 Gamma 市场。 |
| `clob_fallback_successes`    | 通过 CLOB 回退路径解析的条件。 |
| `emitted_condition_ids`      | 至少发出一次 `InstrumentClose` 的条件。 |
| `failed_condition_ids`       | Gamma 和 CLOB 查询均失败的条件。 |
| `used_watchlist_fallback`    | 该请求是否从观察列表中选择了条件。 |
| `timed_out_watchlist`        | 回退选择期间看到的已超时观察列表条目。 |
| `error`                      | 如果发生错误,首个摘要错误信息。 |

兑现是一个独立的账户或执行工作流。请勿扩展数据客户端的解析路径来
索取资金;它只将市场结果关闭事件发布到 Nautilus 中。

### 运行时清理标的

Polymarket 按需自动加载标的,因此随着市场解析、新市场出现、策略在
事件之间轮转,长期运行的会话会不断使缓存增长。使用
`cache.purge_instrument` 来丢弃策略不再跟踪的市场。该调用会移除
该标的的记录,以及所有以其为键的缓存拥有的映射(订单簿、报价、
成交、K 线)。

```python
class PolymarketHousekeeping(Strategy):
    def on_position_closed(self, event: PositionClosed) -> None:
        # 一旦仓位关闭且你不再关心该市场,就丢弃它。
        instrument_id = event.instrument_id
        self.unsubscribe_quote_ticks(instrument_id)
        self.unsubscribe_order_book_deltas(instrument_id)
        self.cache.purge_instrument(instrument_id)
```

Polymarket 上常见的触发条件:

- 市场已解析,不再产生新的成交。
- 事件结束,策略轮转出其市场。
- 策略轮转一个固定大小的观察列表,并丢弃最旧的条目。

清理操作会跳过任何仍有非终态订单(已初始化、已提交、已接受、已模拟、
已释放或飞行中)或非关闭仓位的标的,因此无需与执行客户端协调即可
安全调用。活跃的 WebSocket 订阅属于数据引擎。如果你不再想要更新,
请在清理之前取消订阅。

缓存还暴露了 `purge_order`、`purge_position`、
`purge_closed_orders`、`purge_closed_positions` 和
`purge_account_events`,用于裁剪已关闭的执行状态。对于长期运行的
Polymarket 节点,建议从 `LiveExecEngineConfig` 调度批量清理(15
分钟间隔、60 分钟缓冲是一个合理的默认值)。完整列表参见
[Cache: 清理缓存数据](../concepts/cache.md#purging-cached-data)。

:::warning
何时不再需要某个标的由调用方决定。清理一个仍被其他 actor、策略或
引擎依赖的标的,会导致标的查找缺失,并丢失市场数据历史。
:::

### 执行

执行适配器为订单和成交事件保持一个 `user` 频道连接,并根据交易过程中
遇到的标的按需管理市场订阅。

适配器支持动态的 WebSocket 订阅和取消订阅操作。已匹配的 WebSocket
成交及其修正,会从缓存的订单历史中恢复,并在重连之间去重。如果某笔
成交在其标的可用之前到达,适配器会将其排除在去重状态之外。一次
重新投递的事件或后续的 REST 对账,可以在标的加载完成后应用它。
对于一个完全成交的订单,终态数量规范化会等待该订单
`associate_trades` 列表中的每一个成交 ID 都确认之后,才将订单数量
降为其实际成交量。如果某笔已确认的成交是在 WebSocket 出现间隙后
通过 REST 恢复的,对账会应用相同的仅按订单进行的规范化。如果某个
`MATCHED` 的 WebSocket 更新省略了 `associate_trades`,适配器不会
推断结算已经最终确定;下一次 REST 对账会在该成交达到 `CONFIRMED`
之后恢复剩余部分。

### 订阅限制

Polymarket 目前的速率限制文档中没有公布 WebSocket 订阅上限。V2
配置暴露了默认值为 200 的 `ws_max_subscriptions`,但 Rust 客户端
目前不强制执行该值,也不会创建额外的连接。它会在一个市场连接上,
在一次 `subscribe` 请求中发送所提供的所有资产 ID。

请不要依赖此设置来进行订阅分片。在实现连接分片之前,大规模市场
策略应保持在经过运维验证的场所限制之下。

## 速率限制

Polymarket 通过 Cloudflare 限流强制执行速率限制。超出限制时,请求
会在滑动窗口上被限流。持续超出仍可能表现为 HTTP 429 响应或临时
封锁。

### 部分 REST 限制

Polymarket 会随时间调整这些配额。截至 2026-07-10,官方限制为:

| 端点                            | 突发(10秒) | 持续(10分钟) | 备注                                      |
|-------------------------------------|-------------|--------------------|--------------------------------------------|
| 通用速率限制               | 15,000      | -                  | 全局记录的速率限制。              |
| 健康检查(`/ok`)                | 100         | -                  | 健康检查端点。                           |
| CLOB 通用                        | 9,000       | -                  | CLOB 各端点的聚合总数。           |
| CLOB `POST /order`                  | 5,000       | 120,000            | 单笔提交。                       |
| CLOB `POST /orders`                 | 2,000       | 21,000             | 批量提交(每次请求最多 15 笔订单)。 |
| CLOB `DELETE /order`                | 5,000       | 120,000            | 单笔取消。                       |
| CLOB `DELETE /orders`               | 2,000       | 15,000             | 批量取消。                              |
| CLOB `DELETE /cancel-all`           | 250         | 6,000              | 取消所有订单。                         |
| CLOB `DELETE /cancel-market-orders` | 1,500       | 21,000             | 取消某个市场的订单。              |
| CLOB `GET /balance-allowance`       | 200         | -                  | 余额和授权额度查询。             |
| CLOB API key 端点              | 100         | -                  | 密钥管理。                               |
| Gamma 通用                       | 4,000       | -                  | Gamma 各端点的聚合总数。          |
| Gamma `/markets`                    | 300         | -                  | 市场元数据。                           |
| Gamma `/events`                     | 500         | -                  | 事件元数据。                            |
| Data 通用                        | 1,000       | -                  | Data API 各端点的聚合总数。       |
| Data `/trades`                      | 200         | -                  | 成交历史。                             |
| Data `/positions`                   | 150         | -                  | 当前仓位。                            |

### WebSocket 限制

WebSocket 配额不在已发布的 REST 速率限制表中。V2 适配器暴露了
`ws_max_subscriptions`(默认 200),但尚未强制执行该上限或对连接
进行分片。

:::warning
超出 Polymarket 速率限制会触发 Cloudflare 限流。请求会以滑动窗口
排队,而非立即被拒绝,但持续超出仍可能导致 HTTP 429 响应或临时
封锁。
:::

### 遗留的 V1 加载器速率限制

下方描述的 Python `PolymarketDataLoader` 属于遗留的 V1 适配器,不属于
V2 集成的一部分。此内容仅作为迁移参考保留。

`PolymarketDataLoader` 在使用默认 HTTP 客户端时内置了速率限制。
请求默认自动限流为每分钟 100 次。这是 NautilusTrader 的默认值,
而非 Polymarket 当前公布的限制。当前的 Rust HTTP 客户端也默认带有
保守的每分钟 100 次请求的配额。

在跨多个市场获取大范围日期数据时:

- 共享同一个 `http_client` 实例的多个加载器会自动协调速率限制。
- 若需更高吞吐量,可传入一个调整过配额的自定义 `http_client`。
- 加载器不对 429 错误实现自动重试,如有需要请自行实现退避。

:::info
关于最新的速率限制详情,参见 Polymarket 官方文档:
<https://docs.polymarket.com/api-reference/rate-limits>
:::

## 限制与注意事项

目前已知以下限制:

- 不支持 reduce-only 订单。
- 批量提交(`POST /orders`)每次请求最多接受 15 笔订单;适配器会将
  更大的 `SubmitOrderList` 命令拆分为连续的 15 笔一组的分块。
- 适配器未实现 Polymarket 的已认证心跳自动取消端点。
- 仓位报告会省略低于 0.01 份额的余额。请勿将一份被省略的报告视为
  该 dust 仓位为空仓的证明;低于最小值的剩余部分无法通过 CLOB 的
  五份最小下单数量退出。因此仓位对账会容忍最多 0.009999 份额的
  差异,并对 0.01 份额或更多的差异进行对账。

## 配置

Rust struct 和 PyO3 类暴露相同的 V2 客户端配置。唯一仅限 Rust 的
字段是 `PolymarketDataClientConfig` 上的编程式 `filters` 和
`new_market_filter` trait 对象。

### 数据客户端选项

类/结构体:`PolymarketDataClientConfig`。

| 选项                                        | 默认值   | 说明 |
|-----------------------------------------------|-----------|-------------|
| `instrument_config`                           | `None`    | 启动范围,以 `PolymarketInstrumentProviderConfig` 传入。 |
| `base_url_http`、`base_url_ws`                | `None`    | 覆盖 CLOB HTTP 或 WebSocket 端点。 |
| `base_url_gamma`、`base_url_data_api`         | `None`    | 覆盖 Gamma 或 Data API 端点。 |
| `base_url_rtds`                               | `None`    | 覆盖 RTDS 端点。 |
| `http_timeout_secs`、`ws_timeout_secs`        | `60`、`30` | HTTP 与 WebSocket 超时时间(秒)。 |
| `ws_max_subscriptions`                        | `200`     | 已配置的上限;V2 目前不强制执行或分片。 |
| `update_instruments_interval_mins`            | `60`      | 标的目录刷新间隔;传入 `None` 可禁用。 |
| `subscribe_new_markets`                       | `false`   | 订阅新市场发现事件。 |
| `drop_quotes_missing_side`                    | `true`    | 丢弃不同时包含买价和卖价的报价。 |
| `new_market_fetch_max_concurrency`            | `8`       | 限制从发现事件并发获取市场的数量。 |
| `auto_load_missing_instruments`               | `true`    | 为受支持的请求和订阅加载未知标的。 |
| `auto_load_debounce_ms`                       | `100`     | 合并并发的自动加载请求。 |
| `auto_load_max_retries`                       | `12`      | 重试瞬时的 CLOB 生效遗漏;`0` 禁用重试。 |
| `auto_load_retry_delay_initial_secs`          | `5.0`     | 自动加载的初始重试延迟。 |
| `auto_load_retry_delay_max_secs`              | `15.0`    | 自动加载的最大重试延迟。 |
| `resolve_poll_enabled`                        | `true`    | 为已到期的被观察条件轮询解析结果。 |
| `resolve_poll_interval_secs`                  | `30`      | 解析轮询间隔。 |
| `resolve_poll_grace_secs`                     | `10`      | 到期后开始轮询前的延迟。 |
| `resolve_poll_max_wait_secs`                  | `1800`    | 经过此等待时间后暂停自动轮询。 |
| `transport_backend`                           | `Sockudo` | WebSocket 传输实现。 |

### 执行客户端选项

类/结构体:`PolymarketExecClientConfig`。

| 选项                                           | 默认值                 | 说明 |
|--------------------------------------------------|-------------------------|-------------|
| `trader_id`                                      | 默认 `TraderId`      | 该客户端注册的 trader 标识符。 |
| `account_id`                                     | `POLYMARKET-001`        | 该执行客户端的账户标识符。 |
| `private_key`                                    | `POLYMARKET_PK`         | EIP-712 签名密钥。 |
| `api_key`、`api_secret`、`passphrase`            | 环境变量   | CLOB L2 身份验证凭证。 |
| `funder`                                         | `POLYMARKET_FUNDER`     | 注资钱包;代理和存款钱包签名要求它与签名地址不同。 |
| `signature_type`                                 | `Eoa`                   | `Eoa`、`PolyProxy`、`PolyGnosisSafe` 或 `Poly1271`。 |
| `base_url_http`、`base_url_ws`、`base_url_data_api` | `None`                | 覆盖相应的生产环境端点。 |
| `http_timeout_secs`                              | `60`                    | HTTP 超时时间(秒)。 |
| `max_retries`                                    | `3`                     | 单笔订单提交和取消请求的重试次数。 |
| `retry_delay_initial_ms`                         | `1000`                  | 初始重试延迟。 |
| `retry_delay_max_ms`                             | `10000`                 | 最大重试延迟。 |
| `ack_timeout_secs`                               | `5`                     | 为订单/成交确认处理保留;目前未应用。 |
| `transport_backend`                              | `Sockudo`               | WebSocket 传输实现。 |

批量提交永远不会重试,因为 Polymarket 没有暴露幂等性密钥。代理签名
客户端如果 `funder` 不存在或与签名地址相同,会在构造时失败。

### 标的提供者选项

在数据客户端配置的 `instrument_config` 上传入
`PolymarketInstrumentProviderConfig`。

| 选项               | 默认值 | 说明 |
|----------------------|---------|-------------|
| `load_all`           | `false` | 在启动时加载完整的场所目录。 |
| `load_ids`           | `None`  | 加载确切的 Nautilus 标的 ID。 |
| `filters`            | `None`  | Gamma 查询键/值过滤器。 |
| `event_slugs`        | `None`  | 在启动时解析所列事件的所有市场。 |
| `market_slugs`       | `None`  | 在启动时加载所列的 Gamma 市场 slug。 |
| `event_slug_builder` | `None`  | Rust 实现的涨跌事件 slug 生成器。 |
| `log_warnings`       | `true`  | 发出提供者警告。 |
| `use_gamma_markets`  | `false` | 兼容性字段,在 V2 中没有额外行为。 |

#### 事件 slug 生成器

Rust Python v2 适配器将 Python 视为配置、工厂和用户策略的边界。
提供者、数据和执行操作都运行在 Rust 中。因此,`event_slug_builder`
接受一个 Rust 实现的 `PolymarketUpDownEventSlugConfig`;它不接受
Python 可调用对象路径。

在不下载完整场所目录的情况下,用它来获得可预测的 Polymarket
涨跌(Up/Down)事件 slug。该生成器为配置窗口内对齐的周期,发出符合
`{asset}-updown-{interval_mins}m-{unix_timestamp}` 模式的 slug。

```python
from nautilus_trader.adapters.polymarket import PolymarketInstrumentProviderConfig
from nautilus_trader.adapters.polymarket import PolymarketUpDownEventSlugConfig

instrument_config = PolymarketInstrumentProviderConfig(
    event_slug_builder=PolymarketUpDownEventSlugConfig(
        assets=["btc"],
        interval_mins=5,
        periods=3,
        start_offset_periods=0,
    ),
)
```

对于自定义事件模式,请传入显式的 `event_slugs`、直接传入
`market_slugs`,或添加一个 Rust 过滤器或生成器。Rust v2 适配器会
拒绝 Python 可调用形式的 `event_slug_builder` 值,以确保适配器操作
在实盘交易期间不会跨越到 Python。

## 遗留的 V1 历史数据加载

:::warning
以下 `PolymarketDataLoader` API、脚本和回测示例属于遗留的 V1
Python 适配器。它们不在 V2 集成的支持范围内,也未经过 V2 适配器
测试的验证。它们导入的符号未由 V2 PyO3 包导出,因此这些示例无法在
仅安装了 V2 的环境中运行。
:::

`PolymarketDataLoader` 提供了用于获取和解析历史市场数据的方法,
供研究和回测使用。该加载器与多个 Polymarket API 集成,以提供所需
数据。

:::note
所有数据获取方法都是**异步**的,必须使用 `await` 调用。加载器可以
选择性地接受一个 `http_client` 参数用于依赖注入(便于测试)。
:::

### 数据来源

加载器从三个主要来源获取数据:

1. **Polymarket Gamma API** - 市场元数据、标的详情以及活跃市场
   列表。
2. **Polymarket CLOB API** - 用于构建标的的市场详情。
3. **Polymarket Data API** - 历史成交和当前用户仓位。

当前的加载器**不**提供 CLOB 价格历史时间序列或订单簿历史快照的
辅助方法。

### 方法命名约定

加载器提供了两种访问 Polymarket API 的方式:

| 前缀    | 类型             | 使用场景                                                               |
|-----------|------------------|--------------------------------------------------------------------------|
| `query_*` | 静态方法   | 在没有标的实例的情况下探索 API。无需加载器实例。      |
| `fetch_*` | 实例方法 | 使用已配置的加载器获取数据。使用加载器的 HTTP 客户端。 |

**使用 `query_*`** 的场景是:在确定具体标的之前,你想要探索市场、
发现事件,或获取元数据:

```python
# 不需要加载器:直接查询 API
market = await PolymarketDataLoader.query_market_by_slug("some-market")
event = await PolymarketDataLoader.query_event_by_slug("some-event")
```

**使用 `fetch_*`** 的场景是:你已经有了一个加载器实例,想要使用其
已配置的 HTTP 客户端获取数据(以便跨多次调用协调速率限制):

```python
loader = await PolymarketDataLoader.from_market_slug("some-market")

# 所有 fetch 调用共享加载器的 HTTP 客户端
markets = await loader.fetch_markets(active=True, limit=100)
events = await loader.fetch_events(active=True)
details = await loader.fetch_market_details(condition_id)
```

### 查找市场

使用提供的工具脚本发现活跃市场:

```bash
# 列出所有活跃市场
python nautilus_trader/adapters/polymarket/scripts/active_markets.py

# 专门列出 BTC 和 ETH 涨跌市场
python nautilus_trader/adapters/polymarket/scripts/list_updown_markets.py
```

### 基本用法

创建加载器的推荐方式是使用工厂类方法,它会自动处理所有 API 调用和
标的创建:

```python
import asyncio

from nautilus_trader.adapters.polymarket import PolymarketDataLoader

async def main():
    # 从市场 slug 创建加载器(推荐)
    loader = await PolymarketDataLoader.from_market_slug("gta-vi-released-before-june-2026")

    # 加载器已就绪,标的和 token_id 已设置
    print(loader.instrument)
    print(loader.token_id)

asyncio.run(main())
```

对于包含多个市场的事件(例如温度区间),使用 `from_event_slug`:

```python
# 返回一个加载器列表,该事件下每个市场对应一个加载器
loaders = await PolymarketDataLoader.from_event_slug("highest-temperature-in-nyc-on-january-26")
```

#### 已解析市场的前瞻保护

当为一个在回测构建时已经解析的市场构造加载器时,场所载荷中包含
答案(`closed`、`closedTime`、`umaResolutionStatus`、每个代币的
`winner`)。因此,一个在 `on_start` 中读取
`cache.instrument(...).info` 的策略,可能在模拟运行之前就看到结果。

向任一工厂方法传入 `sanitize_info=True`,可以在构造标的之前,从
`instrument.info` 中去除这些字段。被去除的部分会作为
`resolution_metadata` 存放在加载器上,供事后分析(结算盈亏、
Brier 评分)使用,而不会泄漏到模拟中:

```python
loader = await PolymarketDataLoader.from_market_slug(
    "some-resolved-market",
    sanitize_info=True,
)

assert "closed" not in loader.instrument.info
assert loader.resolution_metadata["closed"] is True
```

### 发现市场和事件

使用 `fetch_markets()` 和 `fetch_events()` 以编程方式发现可用的
市场:

```python
loader = await PolymarketDataLoader.from_market_slug("any-market")

# 列出活跃市场
markets = await loader.fetch_markets(active=True, closed=False, limit=100)
for market in markets:
    print(f"{market['slug']}: {market['question']}")

# 列出活跃事件
events = await loader.fetch_events(active=True, limit=50)
for event in events:
    print(f"{event['slug']}: {event['title']}")

# 获取某个特定事件下的所有市场
event_markets = await loader.get_event_markets("highest-temperature-in-nyc-on-january-26")
```

对于无需创建加载器的快速探索,使用静态的 `query_*` 方法(参见上方的
[方法命名约定](#方法命名约定))。

### 获取成交历史

`load_trades()` 便捷方法一步完成获取和解析历史成交:

```python
import pandas as pd

# 加载所有可用成交
trades = await loader.load_trades()

# 或按时间范围过滤(客户端过滤)
end = pd.Timestamp.now(tz="UTC")
start = end - pd.Timedelta(hours=24)

trades = await loader.load_trades(
    start=start,
    end=end,
)
```

或者,你也可以使用更底层的方法分别获取和解析:

```python
condition_id = loader.condition_id

# 从 Polymarket Data API 获取原始成交
raw_trades = await loader.fetch_trades(condition_id=condition_id)

# 解析为 NautilusTrader 的 TradeTick
trades = loader.parse_trades(raw_trades)
```

成交数据来源于 [Polymarket Data API](https://data-api.polymarket.com/trades),
它提供包括价格、数量、方向和链上交易哈希在内的真实执行数据。

:::note
在活跃度高的市场上,公开的 Data API 会限制基于偏移量的分页。达到
此上限时,加载器会发出一条 `RuntimeWarning`,并返回截至该上限已获取
的成交,而非中止加载。如果你需要完整覆盖一个交易活跃的市场,请使用
其他历史数据来源。
:::

### 完整的回测示例

完整示例参见 `examples/backtest/polymarket_simple_quoter.py`:

```python
import asyncio
from decimal import Decimal

from nautilus_trader.adapters.polymarket import POLYMARKET_VENUE
from nautilus_trader.adapters.polymarket import PolymarketDataLoader
from nautilus_trader.backtest.config import BacktestEngineConfig
from nautilus_trader.backtest.engine import BacktestEngine
from nautilus_trader.examples.strategies.ema_cross_long_only import EMACrossLongOnly
from nautilus_trader.examples.strategies.ema_cross_long_only import EMACrossLongOnlyConfig
from nautilus_trader.model.currencies import pUSD
from nautilus_trader.model.data import BarType
from nautilus_trader.model.enums import AccountType
from nautilus_trader.model.enums import OmsType
from nautilus_trader.model.identifiers import TraderId
from nautilus_trader.model.objects import Money

async def run_backtest():
    # 初始化加载器并获取市场数据
    loader = await PolymarketDataLoader.from_market_slug("gta-vi-released-before-june-2026")
    instrument = loader.instrument

    # 从 Polymarket Data API 加载历史成交
    trades = await loader.load_trades()

    # 配置并运行回测
    config = BacktestEngineConfig(trader_id=TraderId("BACKTESTER-001"))
    engine = BacktestEngine(config=config)

    engine.add_venue(
        venue=POLYMARKET_VENUE,
        oms_type=OmsType.NETTING,
        account_type=AccountType.CASH,
        base_currency=pUSD,
        starting_balances=[Money(10_000, pUSD)],
    )

    engine.add_instrument(instrument)
    engine.add_data(trades)

    bar_type = BarType.from_str(f"{instrument.id}-100-TICK-LAST-INTERNAL")
    strategy_config = EMACrossLongOnlyConfig(
        instrument_id=instrument.id,
        bar_type=bar_type,
        trade_size=Decimal("20"),
    )

    strategy = EMACrossLongOnly(config=strategy_config)
    engine.add_strategy(strategy=strategy)
    engine.run()

    # 显示结果
    print(engine.trader.generate_account_report(POLYMARKET_VENUE))

# 运行回测
asyncio.run(run_backtest())
```

**运行完整示例**:

```bash
python examples/backtest/polymarket_simple_quoter.py
```

### 辅助函数

适配器提供了用于操作 Polymarket 标识符的工具函数:

```python
from nautilus_trader.adapters.polymarket import get_polymarket_instrument_id

# 从 Polymarket 标识符创建 NautilusTrader InstrumentId
instrument_id = get_polymarket_instrument_id(
    condition_id="0xcccb7e7613a087c132b69cbf3a02bece3fdcb824c1da54ae79acc8d4a562d902",
    token_id="8441400852834915183759801017793514978104486628517653995211751018945988243154"
)
```

## 贡献

:::info
如需了解更多功能或为 Polymarket 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
