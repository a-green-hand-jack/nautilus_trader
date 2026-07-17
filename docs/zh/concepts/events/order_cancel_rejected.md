# OrderCancelRejected

`OrderCancelRejected` 表示交易场所拒绝了 `CancelOrder` 命令。`ExecutionEngine` 将其应用
到订单上，更新 `Cache`，并将其发布到 `MessageBus`。它会在交易场所拒绝取消请求时触发。

状态转换：`PENDING_CANCEL` -> 之前的状态（例如 `ACCEPTED`）。处理器：
`on_order_cancel_rejected`。

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderCancelRejected` 还携
带：

| 字段     | Python 类型 | 是否必需/默认值 | 说明                       |
|----------|-------------|------------------|-----------------------------------|
| `reason` | `str`       | 必需             | 订单取消被拒绝的原因。             |

在该事件中，`venue_order_id` 和 `account_id` 通常会被填充，但也可能为 `None`，
`reconciliation` 携带真实值。

## 示例

在策略处理器中读取该事件：

```python
def on_order_cancel_rejected(self, event: OrderCancelRejected) -> None:
    self.log.warning(
        f"Cancel rejected for {event.client_order_id}: {event.reason}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
