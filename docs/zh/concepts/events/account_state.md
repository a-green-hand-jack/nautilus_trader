# AccountState

`AccountState` 携带账户余额与保证金的快照。它会在以下情况触发：交易场所报告了账户更新
（通过执行客户端），或者 `Portfolio` 在持仓更新后重新计算账户状态（适用于启用了
`calculate_account_state` 的保证金账户）。`Portfolio` 会在内部订阅这些事件，以维护风险
敞口与余额跟踪。

`is_reported` 标志用于区分交易场所报告的快照与系统计算得出的快照。

## 字段

| 字段            | Python 类型             | 是否必需/默认值 | 说明                                                          |
|-----------------|-------------------------|------------------|---------------------------------------------------------------|
| `account_id`    | `AccountId`              | 必需             | 账户 ID（对应交易场所）。                                      |
| `account_type`  | `AccountType`            | 必需             | 账户类型（`CASH`、`MARGIN` 或 `BETTING`）。                    |
| `base_currency` | `Currency` 或 `None`     | 必需             | 账户基础货币（多币种账户为 `None`）。                          |
| `is_reported`   | `bool`                   | 必需             | 该状态是否由交易所报告（否则为系统计算）。                     |
| `balances`      | `list[AccountBalance]`   | 必需             | 账户余额（可以为空）。                                          |
| `margins`       | `list[MarginBalance]`    | 必需             | 保证金余额（可以为空）。                                        |
| `info`          | `dict[str, object]`      | 必需             | 额外的、具体实现相关的账户信息。                                |
| `event_id`      | `UUID4`                  | 必需             | 事件 ID。                                                       |
| `ts_event`      | `int`                    | 必需             | 事件发生时的 UNIX 时间戳（纳秒）。                              |
| `ts_init`       | `int`                    | 必需             | 对象初始化时的 UNIX 时间戳（纳秒）。                            |

## 示例

账户状态通常是通过 `Portfolio` 而非专用的处理器来使用的：

```python
from nautilus_trader.model import Venue

# Account state is tracked by the portfolio; query it by venue
account = self.portfolio.account(Venue("BINANCE"))
self.log.info(f"Account state: {account}")
```

## 相关指南

- [事件（Events）](index.md) - 事件分类与分发。
- [会计（Accounting）](../accounting.md) - 账户类型、余额与保证金模型。
- [投资组合（Portfolio）](../portfolio.md) - 账户状态如何驱动风险敞口与余额跟踪。
