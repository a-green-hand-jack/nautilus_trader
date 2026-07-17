# 集成 (Integrations)

NautilusTrader 使用模块化的 *适配器 (adapters)* 来连接交易场所和数据提供商,将原始 API 转换为统一接口和标准化的领域模型。

目前支持以下集成:

| 名称                                                                          | ID                    | 类型                    | 状态                                                       | 文档                      |
| :--------------------------------------------------------------------------- | :-------------------- | :---------------------- | :------------------------------------------------------ | :----------------------- |
| [AX Exchange](https://architect.exchange)                                    | `AX`                  | 永续合约交易所            | ![status](https://img.shields.io/badge/stable-green)    | [指南](architect_ax.md)  |
| [Betfair](https://betfair.com)                                               | `BETFAIR`             | 体育博彩交易所            | ![status](https://img.shields.io/badge/stable-green)    | [指南](betfair.md)      |
| [Binance](https://binance.com)                                               | `BINANCE`             | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](binance.md)      |
| [Coinbase](https://coinbase.com)                                             | `COINBASE`            | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](coinbase.md)     |
| [BitMEX](https://www.bitmex.com)                                             | `BITMEX`              | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](bitmex.md)       |
| [Blockchain](blockchain.md)                                                  | `BLOCKCHAIN`          | DeFi 数据提供商          | ![status](https://img.shields.io/badge/stable-green)    | [指南](blockchain.md)   |
| [Bybit](https://www.bybit.com)                                               | `BYBIT`               | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](bybit.md)        |
| [Databento](https://databento.com)                                           | `DATABENTO`           | 数据提供商               | ![status](https://img.shields.io/badge/stable-green)    | [指南](databento.md)    |
| [Deribit](https://www.deribit.com)                                           | `DERIBIT`             | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](deribit.md)      |
| [Derive](https://www.derive.xyz)                                             | `DERIVE`              | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](derive.md)       |
| [dYdX](https://dydx.exchange/)                                               | `DYDX`                | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](dydx.md)         |
| [Hyperliquid](https://hyperliquid.xyz)                                       | `HYPERLIQUID`         | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](hyperliquid.md)  |
| [Lighter](https://lighter.xyz)                                               | `LIGHTER`             | 加密货币交易所 (DEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](lighter.md)      |
| [Interactive Brokers](https://www.interactivebrokers.com)                    | `INTERACTIVE_BROKERS` | 券商(多场所)            | ![status](https://img.shields.io/badge/stable-green)    | [指南](ib.md)           |
| [Kraken](https://kraken.com)                                                 | `KRAKEN`              | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](kraken.md)       |
| [OKX](https://okx.com)                                                       | `OKX`                 | 加密货币交易所 (CEX)     | ![status](https://img.shields.io/badge/stable-green)    | [指南](okx.md)          |
| [Polymarket](https://polymarket.com)                                         | `POLYMARKET`          | 预测市场 (DEX)           | ![status](https://img.shields.io/badge/stable-green)    | [指南](polymarket.md)   |
| [Tardis](https://tardis.dev)                                                 | `TARDIS`              | 加密货币数据提供商        | ![status](https://img.shields.io/badge/stable-green)    | [指南](tardis.md)       |

- **ID**:该集成适配器客户端的默认客户端 ID。
- **类型**:集成的类型(通常即场所类型)。

## 状态

- `planned`:计划中,将在未来开发。
- `building`:正在构建中,可能尚不可用。
- `beta`:已完成到最基本可用状态,处于 “beta” 测试阶段。
- `stable`:功能集与 API 已稳定,该集成已由开发者与用户测试到相当程度(仍可能存在一些 bug)。

## 实现目标

NautilusTrader 的主要目标是为多种集成提供统一的交易系统。为支持尽可能广泛的交易
策略,将优先支持 *标准* 功能:

- 请求历史市场数据。
- 流式获取实时市场数据。
- 对账执行状态。
- 提交带有标准执行指令的标准订单类型。
- 修改现有订单(如果交易所支持)。
- 取消订单。

每个集成的实现都力求满足以下标准:

- 底层客户端组件应尽可能贴近交易所 API 本身。
- 交易所的全部功能(在适用于 NautilusTrader 的范围内)应*最终*得到支持。
- 将添加交易所特定的数据类型,以支持相应功能,并返回用户可合理预期的返回类型。
- 交易所或 NautilusTrader 不支持的操作,在被调用时将记录为警告或错误日志。

::::warning[跟踪日志与凭证]

TRACE 级别日志可能包含原始的出站 WebSocket 载荷,其中对某些场所可能包含身份验证数据。
TRACE 仅应用于本地调试,分享 TRACE 日志前请先脱敏。

::::

## API 统一

所有集成都必须遵循 NautilusTrader 的系统 API,这需要进行规范化和标准化:

- 除非需要消歧(例如 Binance 现货 vs. Binance 期货),否则符号应使用场所原生的符号格式。
- 时间戳必须使用 UNIX 纪元纳秒。如果使用毫秒,字段/属性名称应显式以 `_ms` 结尾。
