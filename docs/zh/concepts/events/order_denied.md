# OrderDenied

`OrderDenied` 表示订单已被 Nautilus 系统拒绝提交（denied）。`ExecutionEngine` 将其应用
到订单上，更新 `Cache`，并将其发布到 `MessageBus`。它会在一个本应有效的订单因为例如风险
限制或不受支持的功能而无法被提交时触发。

状态转换：`INITIALIZED` -> `DENIED`。处理器：`on_order_denied`。

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderDenied` 还携带：

| 字段     | Python 类型 | 是否必需/默认值 | 说明               |
|----------|-------------|------------------|----------------------------|
| `reason` | `str`       | 必需             | 订单被拒绝提交的原因。       |

在该事件中，`venue_order_id` 和 `account_id` 均为 `None`，`reconciliation` 始终为
`False`，且 `ts_event` 等于 `ts_init`。

## 示例

在策略处理器中读取该事件：

```python
def on_order_denied(self, event: OrderDenied) -> None:
    self.log.warning(f"Order {event.client_order_id} denied: {event.reason}")
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [执行（Execution）](../execution.md) - 风险检查与订单被拒绝提交的原因。
- [订单（Orders）](../orders/) - 订单类型与状态机。
