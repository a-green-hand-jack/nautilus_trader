# PositionOpened

`PositionOpened` 表示持仓已被开立。`ExecutionEngine` 会在某笔成交创建新持仓时发出该事
件（参见[从成交到持仓](index.md#from-fill-to-position-the-causal-chain)）。处理器：
`on_position_opened`。

## 字段

`PositionOpened` 共享持仓事件的字段集合。完整字段矩阵（涵盖三种持仓事件）请参见
[持仓事件字段](index.md#position-event-fields)。以下是 `PositionOpened` 特有的字段：

| 字段           | Python 类型    | 说明                                    |
|----------------|----------------|--------------------------------------------------|
| `entry`        | `OrderSide`    | 开立该持仓的入场订单方向。                |
| `side`         | `PositionSide` | 当前持仓方向（`LONG` 或 `SHORT`）。       |
| `quantity`     | `Quantity`     | 当前开仓数量。                             |
| `avg_px_open`  | `float`        | 平均开仓价格。                             |
| `realized_pnl` | `Money`        | 该持仓已实现的盈亏。                       |

开仓时，`closing_order_id` 为 `None`，`avg_px_close` 和 `realized_return` 为零。

## 示例

在策略处理器中读取该事件：

```python
def on_position_opened(self, event: PositionOpened) -> None:
    self.log.info(
        f"Opened {event.side} {event.quantity} {event.instrument_id} "
        f"@ {event.avg_px_open}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及从成交到持仓的因果链。
- [持仓（Positions）](../positions.md) - 持仓生命周期、聚合与盈亏。
- [订单（Orders）](../orders/) - 其成交用于开仓和平仓的订单。
