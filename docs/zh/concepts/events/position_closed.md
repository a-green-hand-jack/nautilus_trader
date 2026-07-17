# PositionClosed

`PositionClosed` 表示持仓已被平仓。`ExecutionEngine` 会在某笔成交将持仓平至零仓
（flatten）时发出该事件（参见[从成交到持仓](index.md#from-fill-to-position-the-causal-chain)）。
处理器：`on_position_closed`。

## 字段

`PositionClosed` 共享持仓事件的字段集合。完整字段矩阵（涵盖三种持仓事件）请参见
[持仓事件字段](index.md#position-event-fields)。以下是 `PositionClosed` 特有的字段：

| 字段               | Python 类型     | 说明                                            |
|--------------------|-----------------|---------------------------------------------------------|
| `closing_order_id` | `ClientOrderId` | 平掉该持仓的客户端订单 ID。                       |
| `avg_px_close`     | `float`         | 平均平仓价格。                                     |
| `realized_return`  | `float`         | 该持仓已实现的收益率。                             |
| `realized_pnl`     | `Money`         | 该持仓最终已实现的盈亏。                           |
| `duration_ns`      | `int`           | 总持仓时长（纳秒）。                               |
| `ts_closed`        | `int`           | 持仓平仓时的 UNIX 时间戳（纳秒）。                 |

平仓时，`side` 为 `FLAT`，`unrealized_pnl` 为零。

## 示例

在策略处理器中读取该事件：

```python
def on_position_closed(self, event: PositionClosed) -> None:
    self.log.info(
        f"Closed {event.instrument_id}: realized={event.realized_pnl} "
        f"return={event.realized_return}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及从成交到持仓的因果链。
- [持仓（Positions）](../positions.md) - 持仓生命周期、聚合与盈亏。
- [订单（Orders）](../orders/) - 其成交用于开仓和平仓的订单。
