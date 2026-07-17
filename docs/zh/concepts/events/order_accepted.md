# OrderAccepted

`OrderAccepted` 表示订单已被交易场所接受。`ExecutionEngine` 将其应用到订单上，更新
`Cache`，并将其发布到 `MessageBus`。它会在交易场所确认该订单已被接收且有效时触发（通常
对应 FIX 协议中的 `NEW` OrdStatus）。

状态转换：`SUBMITTED` -> `ACCEPTED`。处理器：`on_order_accepted`。

## 字段

`OrderAccepted` 仅携带[通用订单事件字段](index.md#common-order-event-fields)。在该事件
中，`venue_order_id` 和 `account_id` 会被填充，`reconciliation` 携带真实值（默认为
`False`）。

## 示例

在策略处理器中读取该事件：

```python
def on_order_accepted(self, event: OrderAccepted) -> None:
    self.log.info(
        f"Order {event.client_order_id} accepted as {event.venue_order_id}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
