# OrderInitialized

`OrderInitialized` 表示订单已被初始化。`ExecutionEngine` 将其应用到订单上，更新
`Cache`，并将其发布到 `MessageBus`。它是携带足够信息的种子事件（seed event），可用于
将订单通过网络发送出去并原样重建。

在本地作为新订单的种子事件被创建。处理器：`on_order_initialized`。

## 字段

除[通用订单事件字段](index.md#common-order-event-fields)外，`OrderInitialized` 还携带：

| 字段                     | Python 类型                       | 是否必需/默认值 | 说明                                                        |
|--------------------------|------------------------------------|------------------|--------------------------------------------------------------------|
| `side`                   | `OrderSide`                         | 必需             | 订单方向（以 `event.side` 暴露）。                          |
| `order_type`             | `OrderType`                         | 必需             | 订单类型。                                                    |
| `quantity`               | `Quantity`                          | 必需             | 订单数量。                                                    |
| `time_in_force`          | `TimeInForce`                       | 必需             | 订单的有效期方式（time in force）。                           |
| `post_only`              | `bool`                              | 必需             | 该订单是否仅提供流动性（作为挂单方成交，make a market）。     |
| `reduce_only`            | `bool`                              | 必需             | 该订单是否携带“只减仓”（reduce-only）执行指令。               |
| `quote_quantity`         | `bool`                              | 必需             | 订单数量是否以报价货币（quote currency）计价。                |
| `options`                | `dict[str, str]`                    | 必需             | 用于特定订单参数的订单初始化选项。                             |
| `emulation_trigger`      | `TriggerType`                       | `NO_TRIGGER`     | 用于本地订单模拟的市场价格触发条件。                           |
| `trigger_instrument_id`  | `InstrumentId` 或 `None`            | 必需             | 模拟触发所使用的金融工具 ID（默认为 `instrument_id`）。       |
| `contingency_type`       | `ContingencyType`                   | 必需             | 订单的联动类型（contingency type）。                          |
| `order_list_id`          | `OrderListId` 或 `None`             | 必需             | 与该订单关联的订单列表 ID。                                    |
| `linked_order_ids`       | `list[ClientOrderId]` 或 `None`     | 必需             | 关联的客户端订单 ID。                                          |
| `parent_order_id`        | `ClientOrderId` 或 `None`           | 必需             | 该订单的父级客户端订单 ID。                                    |
| `exec_algorithm_id`      | `ExecAlgorithmId` 或 `None`         | 必需             | 该订单使用的执行算法 ID。                                      |
| `exec_algorithm_params`  | `dict[str, Any]` 或 `None`          | 必需             | 执行算法的参数。                                                |
| `exec_spawn_id`          | `ClientOrderId` 或 `None`           | 必需             | 生成该订单的执行算法的主客户端订单 ID。                        |
| `tags`                   | `list[str]` 或 `None`               | 必需             | 该订单的自定义用户标签。                                        |

在该事件中，`venue_order_id` 和 `account_id` 均为 `None`，且 `ts_event` 等于
`ts_init`。此处 `reconciliation` 属性始终返回 `False`，即使是在对账过程中重建的订单也
是如此；后续的订单事件，例如 [`OrderAccepted`](order_accepted.md)，会携带真实值。

## 示例

在策略处理器中读取该事件：

```python
def on_order_initialized(self, event: OrderInitialized) -> None:
    self.log.info(
        f"Initialized {event.order_type} {event.side} "
        f"{event.quantity} {event.instrument_id}",
    )
```

## 相关指南

- [事件（Events）](index.md) - 事件分类、分发以及通用订单事件字段。
- [订单（Orders）](../orders/) - 订单类型与状态机。
