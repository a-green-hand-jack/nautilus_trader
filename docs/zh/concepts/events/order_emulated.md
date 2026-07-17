# OrderEmulated

`OrderEmulated` 表示订单已被 Nautilus 系统置于模拟（emulated）状态。`ExecutionEngine`
将其应用到订单上，更新 `Cache`，并将其发布到 `MessageBus`。它会在 `OrderEmulator` 将订单
纳入本地模拟时触发。

状态转换：`INITIALIZED` -> `EMULATED`。处理器：`on_order_emulated`。

## 字段

`OrderEmulated` 仅携带[通用订单事件字段](index.md#common-order-event-fields)。在该事件
中，`venue_order_id` 和 `account_id` 均为 `None`，`reconciliation` 始终为 `False`，且
`ts_event` 等于 `ts_init`。

## 示例

在策略处理器中读取该事件：

```python
def on_order_emulated(self, event: OrderEmulated) -> None:
    self.log.info(f"Order {event.client_order_id} is now emulated locally")
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [模拟订单（Emulated orders）](../orders/emulated.md) - 本地模拟的生命周期。
