# OrderRejected

`OrderRejected` 表示订单已被交易场所拒绝。`ExecutionEngine` 将其应用到订单上，更新
`Cache`，并将其发布到 `MessageBus`。它会在交易场所拒绝已提交的订单时触发。

状态转换：`SUBMITTED` -> `REJECTED`。处理器：`on_order_rejected`。

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderRejected` 还携带：

| 字段            | Python 类型 | 是否必需/默认值 | 说明                                                                    |
|-----------------|-------------|------------------|----------------------------------------------------------------------------------|
| `reason`        | `str`       | 必需             | 订单被拒绝的原因。                                                        |
| `due_post_only` | `bool`      | `False`          | 是否因为该订单为“只做挂单”（post-only）且会立即以吃单方（taker）方式成交而被拒绝。 |

在该事件中，`account_id` 会被填充，`venue_order_id` 为 `None`，`reconciliation` 携带
真实值。

## 示例

在策略处理器中读取该事件：

```python
def on_order_rejected(self, event: OrderRejected) -> None:
    self.log.warning(f"Order {event.client_order_id} rejected: {event.reason}")
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
