# OrderReleased

`OrderReleased` 表示订单已从 `OrderEmulator` 中释放。`ExecutionEngine` 将其应用到订单
上，更新 `Cache`，并将其发布到 `MessageBus`。它会在模拟器的触发价格条件被满足、订单被
释放到交易场所时触发。

状态转换：`EMULATED` -> `RELEASED`。处理器：`on_order_released`。

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderReleased` 还携带：

| 字段              | Python 类型 | 是否必需/默认值 | 说明                                       |
|-------------------|-------------|------------------|-----------------------------------------------------|
| `released_price`  | `Price`     | 必需             | 使订单从模拟器中释放的价格。                        |

在该事件中，`venue_order_id` 和 `account_id` 均为 `None`，`reconciliation` 始终为
`False`，且 `ts_event` 等于 `ts_init`。

## 示例

在策略处理器中读取该事件：

```python
def on_order_released(self, event: OrderReleased) -> None:
    self.log.info(
        f"Order {event.client_order_id} released at {event.released_price}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [模拟订单（Emulated orders）](../orders/emulated.md) - 本地模拟的生命周期。
