# OrderPendingCancel

`OrderPendingCancel` 表示 `CancelOrder` 命令已被发送到交易场所。`ExecutionEngine` 将其
应用到订单上，更新 `Cache`，并将其发布到 `MessageBus`。它会在系统发出取消请求并等待
交易场所确认时触发。

状态转换：`ACCEPTED` -> `PENDING_CANCEL`。处理器：`on_order_pending_cancel`。

## 字段

`OrderPendingCancel` 仅携带[通用订单事件字段](index.md#common-order-event-fields)。在
该事件中，`venue_order_id` 和 `account_id` 通常会被填充，但也可能为 `None`，
`reconciliation` 携带真实值。

## 示例

在策略处理器中读取该事件：

```python
def on_order_pending_cancel(self, event: OrderPendingCancel) -> None:
    self.log.info(f"Cancel pending for {event.client_order_id}")
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
