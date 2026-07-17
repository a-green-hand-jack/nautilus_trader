# OrderUpdated

`OrderUpdated` 表示订单已在交易场所被更新。`ExecutionEngine` 将其应用到订单上，更新
`Cache`，并将其发布到 `MessageBus`。它会在交易场所确认了对数量、价格或触发价格的修改
时触发。

状态转换：`PENDING_UPDATE` -> 之前的状态（例如 `ACCEPTED`）。处理器：
`on_order_updated`。

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderUpdated` 还携带：

| 字段                | Python 类型        | 是否必需/默认值 | 说明                                                 |
|---------------------|---------------------|------------------|---------------------------------------------------------------|
| `quantity`          | `Quantity`           | 必需             | 订单当前的数量。                                       |
| `price`             | `Price` 或 `None`    | 必需             | 订单当前的价格。                                       |
| `trigger_price`     | `Price` 或 `None`    | 必需             | 订单当前的触发价格。                                   |
| `is_quote_quantity` | `bool`               | `False`          | 订单数量是否以报价货币计价。                           |

在该事件中，`venue_order_id` 和 `account_id` 通常会被填充，但也可能为 `None`，
`reconciliation` 携带真实值。

## 示例

在策略处理器中读取该事件：

```python
def on_order_updated(self, event: OrderUpdated) -> None:
    self.log.info(
        f"Order {event.client_order_id} updated: "
        f"qty={event.quantity} price={event.price}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
