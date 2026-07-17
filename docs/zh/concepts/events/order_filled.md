# OrderFilled

`OrderFilled` 表示订单已在交易所成交。`ExecutionEngine` 将其应用到订单上，更新
`Cache`，并将其发布到 `MessageBus`。它会在交易场所报告该订单的部分或全部执行成交时触
发，并进而驱动持仓事件的产生。

状态转换：`ACCEPTED` -> `FILLED` / `PARTIALLY_FILLED`。处理器：`on_order_filled`。

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderFilled` 还携带：

| 字段             | Python 类型              | 是否必需/默认值 | 说明                                                              |
|------------------|---------------------------|------------------|----------------------------------------------------------------------------|
| `trade_id`       | `TradeId`                  | 必需             | 成交撮合 ID（由交易场所分配）。                                    |
| `position_id`    | `PositionId` 或 `None`     | 必需             | 与该成交关联的持仓 ID（由交易场所分配）。                          |
| `order_side`     | `OrderSide`                 | 必需             | 执行成交的订单方向。                                                |
| `order_type`     | `OrderType`                 | 必需             | 执行成交的订单类型。                                                |
| `last_qty`       | `Quantity`                  | 必需             | 本次执行的成交数量。                                                |
| `last_px`        | `Price`                     | 必需             | 本次执行的成交价格（并非平均价格）。                                |
| `currency`       | `Currency`                  | 必需             | 成交价格的货币。                                                    |
| `commission`     | `Money`                     | 必需             | 成交手续费。                                                        |
| `liquidity_side` | `LiquiditySide`             | 必需             | 执行成交的流动性方向（`MAKER`、`TAKER` 或 `NO_LIQUIDITY_SIDE`）。   |
| `info`           | `dict[str, object]`         | `None`           | 额外的成交信息（省略时会被强制转换为 `{}`）。                       |

在该事件中，`venue_order_id` 和 `account_id` 会被填充，`reconciliation` 携带真实值。

## 示例

在策略处理器中读取该事件：

```python
def on_order_filled(self, event: OrderFilled) -> None:
    self.log.info(
        f"Filled {event.last_qty} @ {event.last_px} "
        f"({event.liquidity_side}) commission={event.commission}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [持仓（Positions）](../positions.md) - 由成交创建和修改的持仓。
- [订单（Orders）](../orders/) - 订单类型与状态机。
