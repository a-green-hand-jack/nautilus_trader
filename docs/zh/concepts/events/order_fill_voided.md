# OrderFillVoided

`OrderFillVoided` 记录了此前报告的成交的全部或部分内容不再具有经济效力（economic
effect）。`ExecutionEngine` 会将该修正应用到订单和持仓上，然后刷新投资组合的持仓与
PnL 缓存，之后再将其发布到 `MessageBus`。交易场所适配器（venue adapter）会从其权威的
账户端点刷新账户余额。

该修正会就地更新缓存中的持仓聚合数据。它不会合成 `PositionChanged` 或
`PositionClosed`；策略会在修正后的缓存状态可用之后收到 `OrderFillVoided`。

修正并非反向成交。它保留了原始成交的身份标识，因此重放（replay）、对账
（reconciliation）以及策略审计历史都能直接反映交易场所的操作。

处理器：`on_order_fill_voided`。

## 数量与状态行为

`voided_qty` 和 `commission_voided` 是针对所引用的 `trade_id` 的累计值。数量和手续费
的修正不能减少。后续的修订可以增大这两个值中的任意一个，或在相同数量下改变
`is_reopened`。重复的、过期的以及超额作废的修正都会被拒绝。

默认情况下，一次修正不会使被修正的数量重新变为可执行状态：

- 已完全成交的订单会变为终态 `VOIDED`，即使仍有部分有效成交数量存续。
- 部分成交的订单会保留原本仍在运作中（working）的剩余部分。其状态取决于存续的有效成交，
  其剩余数量（leaves）不包含未重新开放（non-reopened）的作废数量。
- 已取消或已过期的订单会保持其终态。
- 若某次修正设置了 `is_reopened=true`，则被修正的数量也会重新回到运作中的剩余数量。
  在没有有效成交存续时，订单会变为 `ACCEPTED`；若仍有部分数量存续，则变为
  `PARTIALLY_FILLED`。

如果 Nautilus 从未应用过所引用的那笔成交，该事件会记录权威的订单级修正，但不会逆转持仓
或账户的风险敞口。

:::note
该模式（schema）会追加此事件与状态，而不改变现有记录。较旧的 v2 读取器无法识别新增的
值，因此在消费者读取修正后的数据流或数据目录（catalog data）之前，应先升级它们。
:::

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderFillVoided` 还携带：

| 字段                 | Python 类型                  | 是否必需/默认值 | 说明                                                     |
|---------------------|-------------------------------|------------------|---------------------------------------------------------|
| `correction_id`     | `str`                          | 必需             | 该次修正修订版本的标识。                                 |
| `trade_id`          | `TradeId`                      | 必需             | 原始交易场所成交 ID。                                    |
| `voided_qty`        | `Quantity`                     | 必需             | 该笔成交累计的无效数量。                                 |
| `commission_voided` | `Money` 或 `None`              | `None`           | 该笔成交累计的手续费修正。                               |
| `order_side`        | `OrderSide`                     | 必需             | 原始成交的方向。                                          |
| `order_type`        | `OrderType`                     | 必需             | 原始订单的类型。                                          |
| `last_px`           | `Price`                         | 必需             | 原始成交的价格。                                          |
| `currency`          | `Currency`                      | 必需             | 原始成交价格的货币。                                      |
| `liquidity_side`    | `LiquiditySide`                 | 必需             | 原始成交的流动性方向。                                    |
| `position_id`       | `PositionId` 或 `None`          | `None`           | 与原始成交关联的持仓 ID。                                 |
| `reason`            | `str` 或 `None`                 | `None`           | 交易场所或对账过程给出的修正原因。                         |
| `info`              | `dict[str, str]` 或 `None`      | `None`           | 额外的交易场所修正元数据。                                 |
| `is_reopened`       | `bool`                          | `False`          | 交易场所是否证明该订单再次可执行。                         |

## 示例

```python
def on_order_fill_voided(self, event: OrderFillVoided) -> None:
    self.log.warning(
        f"Corrected {event.trade_id}: voided={event.voided_qty} "
        f"reopened={event.is_reopened}",
    )
```

## 相关指南

- [执行（Execution）](../execution.md) - 修正的应用与发布顺序。
- [OrderFilled](order_filled.md) - 原始的成交事件。
- [订单（Orders）](../orders/) - 订单状态与状态流。
