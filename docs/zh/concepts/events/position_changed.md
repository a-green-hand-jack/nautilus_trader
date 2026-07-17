# PositionChanged

`PositionChanged` 表示持仓已发生变化。`ExecutionEngine` 会在后续成交改变某个已开仓持仓
的数量或方向时发出该事件（参见[从成交到持仓](index.md#from-fill-to-position-the-causal-chain)）。
处理器：`on_position_changed`。

## 字段

`PositionChanged` 共享持仓事件的字段集合。完整字段矩阵（涵盖三种持仓事件）请参见
[持仓事件字段](index.md#position-event-fields)。以下是 `PositionChanged` 特有的字段：

| 字段              | Python 类型 | 说明                                            |
|-------------------|-------------|---------------------------------------------------------|
| `peak_qty`        | `Quantity`  | 该持仓曾达到的方向性峰值数量。                    |
| `avg_px_close`    | `float`     | 目前为止的平均平仓价格。                           |
| `realized_return` | `float`     | 该持仓已实现的收益率。                             |
| `realized_pnl`    | `Money`     | 该持仓已实现的盈亏。                               |
| `unrealized_pnl`  | `Money`     | 该持仓未实现的盈亏。                               |

只要持仓仍处于开仓状态，`closing_order_id` 就仍为 `None`。

## 示例

在策略处理器中读取该事件：

```python
def on_position_changed(self, event: PositionChanged) -> None:
    self.log.info(
        f"Changed {event.instrument_id} to {event.signed_qty} "
        f"(realized={event.realized_pnl})",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及从成交到持仓的因果链。
- [持仓（Positions）](../positions.md) - 持仓生命周期、聚合与盈亏。
- [订单（Orders）](../orders/) - 其成交用于开仓和平仓的订单。
