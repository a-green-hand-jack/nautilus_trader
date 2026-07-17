# Stop-Limit（止损限价单）

`FIX OrdType <40>=4`（Stop Limit）

*Stop-Limit* 订单是一种条件订单，一旦被触发，将立即以指定价格下达一笔 *Limit* 订单。

## 使用场景

当你既需要止损触发，又需要对最差可接受成交价设置上限时，使用 *Stop-Limit* 订单，例如保护性离场单，或不愿以超出某一价格成交的突破入场单。
其优势在于释放出的 *Limit* 订单具有价格保护。与 *Stop-Market* 相比，其核心风险代价在于：如果市场跳空穿越了触发价和限价，订单可能完全无法成交，导致持仓失去保护。

## 示例

在以下示例中，我们在 Currenex 外汇 ECN 上创建一笔 *Stop-Limit* 订单，一旦市场达到 1.30010 USD 的触发价，即以 1.3000 USD 的限价买入 50,000 GBP，有效期至 2022 年 6 月 6 日中午（UTC）：

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::{OrderSide, TimeInForce, TriggerType},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};

let order = self.order().stop_limit(
    InstrumentId::from("GBP/USD.CURRENEX"),
    OrderSide::Buy,
    Quantity::from(50_000),
    Price::from("1.30000"),
    Price::from("1.30010"),
    Some(TriggerType::BidAsk), // optional (default DEFAULT)
    Some(TimeInForce::Gtd),    // optional (default GTC)
    Some(UnixNanos::from(1_654_516_800_000_000_000_u64)), // 2022-06-06T12:00:00 UTC
    Some(true),                // post_only (default false)
    Some(false),               // reduce_only (default false)
    None,                      // quote_quantity (default false)
    None,                      // display_qty
    None,                      // emulation_trigger
    None,                      // trigger_instrument_id
    None,                      // exec_algorithm_id
    None,                      // exec_algorithm_params
    None,                      // tags
    None,                      // client_order_id
);
```

```python tab="Python"
import pandas as pd
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model.enums import TriggerType
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model.orders import StopLimitOrder

order: StopLimitOrder = self.order_factory.stop_limit(
    instrument_id=InstrumentId.from_str("GBP/USD.CURRENEX"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(50_000),
    price=Price.from_str("1.30000"),
    trigger_price=Price.from_str("1.30010"),
    trigger_type=TriggerType.BID_ASK,  # <-- optional (default DEFAULT)
    time_in_force=TimeInForce.GTD,  # <-- optional (default GTC)
    expire_time=pd.Timestamp("2022-06-06T12:00"),
    post_only=True,  # <-- optional (default False)
    reduce_only=False,  # <-- optional (default False)
    tags=None,  # <-- optional (default None)
)
```

更多详情请参见 [`StopLimitOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.stop_limit.StopLimitOrder)。

## 相关指南

- [订单（Orders）](index.md#trigger-type) - 触发类型及其他执行指令。
- [模拟订单（Emulated orders）](emulated.md) - 在不原生支持的交易场所上模拟条件订单。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
