# OrderSubmitted

`OrderSubmitted` 表示订单已被系统提交到交易场所。`ExecutionEngine` 将其应用到订单上，
更新 `Cache`，并将其发布到 `MessageBus`。它会在系统将订单发送到交易场所并等待确认时
触发。

状态转换：`INITIALIZED` / `RELEASED` -> `SUBMITTED`。处理器：`on_order_submitted`。

## 字段

`OrderSubmitted` 仅携带[通用订单事件字段](index.md#common-order-event-fields)。在该事
件中，`account_id` 会被填充，`venue_order_id` 尚未被分配（`None`），`reconciliation`
始终为 `False`。

## 示例

在策略处理器中读取该事件：

```python
def on_order_submitted(self, event: OrderSubmitted) -> None:
    self.log.info(f"Order {event.client_order_id} submitted ({event.account_id})")
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
