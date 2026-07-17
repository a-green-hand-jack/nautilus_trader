# OrderTriggered

`OrderTriggered` 表示订单已在交易场所被触发。`ExecutionEngine` 将其应用到订单上，更新
`Cache`，并将其发布到 `MessageBus`。它会在限价类条件订单（`StopLimit`、
`LimitIfTouched` 或 `TrailingStopLimit`）满足触发条件时触发。

状态转换：`ACCEPTED` -> `TRIGGERED`。处理器：`on_order_triggered`。

## 字段

`OrderTriggered` 仅携带[通用订单事件字段](index.md#common-order-event-fields)。在该事
件中，`venue_order_id` 和 `account_id` 通常会被填充，但也可能为 `None`，
`reconciliation` 携带真实值。

## 示例

在策略处理器中读取该事件：

```python
def on_order_triggered(self, event: OrderTriggered) -> None:
    self.log.info(f"Order {event.client_order_id} triggered")
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
