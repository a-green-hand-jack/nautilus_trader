# Trailing-Stop-Limit（跟踪止损限价单）

`FIX OrdType <40>=4`（Stop Limit）+ 跟踪挂钩

*Trailing-Stop-Limit* 订单是一种条件订单，其止损触发价会以固定偏移量跟踪所定义的市场价格。一旦触发，将立即以定义好的价格
（该价格在触发前也会随市场变动而更新）下达一笔 *Limit* 订单。

## 使用场景

当你既希望获得跟踪止损的动态跟随特性，又希望对成交价设置上限时，使用 *Trailing-Stop-Limit* 订单。其优势是跟踪保护与价格控制相结合。
代价是 *Stop-Limit* 的跟踪版本所固有的问题：在快速反转行情中，被释放的 *Limit* 单可能无法成交，导致持仓仍处于敞口状态。

## 示例

在以下示例中，我们在 Currenex 外汇 ECN 上创建一笔 *Trailing-Stop-Limit* 订单，用 USD 以 0.71000 USD 的限价买入 1,250,000 AUD，
在 0.72000 USD 激活，随后以 0.00100 USD 的止损偏移量跟踪当前卖价，有效期至另行通知：

```rust tab="Rust"
use nautilus_model::{
    enums::{OrderSide, TimeInForce, TrailingOffsetType, TriggerType},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};
use rust_decimal_macros::dec;
use ustr::Ustr;

let order = self.order().trailing_stop_limit(
    InstrumentId::from("AUD/USD.CURRENEX"),
    OrderSide::Buy,
    Quantity::from(1_250_000),
    Price::from("0.71000"),          // limit price
    dec!(0.00050),                   // limit_offset
    dec!(0.00100),                   // trailing_offset
    Some(TrailingOffsetType::Price), // optional (default PRICE)
    Some(Price::from("0.72000")),    // activation_price
    None,                            // trigger_price (materializes from the offset on the first trail)
    Some(TriggerType::BidAsk),       // optional (default DEFAULT)
    Some(TimeInForce::Gtc),          // optional (default GTC)
    None,                            // expire_time
    Some(false),                     // post_only (default false)
    Some(true),                      // reduce_only (default false)
    None,                            // quote_quantity (default false)
    None,                            // display_qty
    None,                            // emulation_trigger
    None,                            // trigger_instrument_id
    None,                            // exec_algorithm_id
    None,                            // exec_algorithm_params
    Some(vec![Ustr::from("TRAILING_STOP")]), // tags
    None,                            // client_order_id
);
```

```python tab="Python"
import pandas as pd
from decimal import Decimal
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model.enums import TriggerType
from nautilus_trader.model.enums import TrailingOffsetType
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model.orders import TrailingStopLimitOrder

order: TrailingStopLimitOrder = self.order_factory.trailing_stop_limit(
    instrument_id=InstrumentId.from_str("AUD/USD.CURRENEX"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(1_250_000),
    price=Price.from_str("0.71000"),
    activation_price=Price.from_str("0.72000"),
    trigger_type=TriggerType.BID_ASK,  # <-- optional (default DEFAULT)
    limit_offset=Decimal("0.00050"),
    trailing_offset=Decimal("0.00100"),
    trailing_offset_type=TrailingOffsetType.PRICE,
    time_in_force=TimeInForce.GTC,  # <-- optional (default GTC)
    expire_time=None,  # <-- optional (default None)
    reduce_only=True,  # <-- optional (default False)
    tags=["TRAILING_STOP"],  # <-- optional (default None)
)
```

更多详情请参见 [`TrailingStopLimitOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.trailing_stop_limit.TrailingStopLimitOrder)。

## 相关指南

- [订单（Orders）](index.md#trigger-offset-type) - 触发类型与跟踪偏移类型。
- [模拟订单（Emulated orders）](emulated.md) - 在不原生支持的交易场所上模拟跟踪止损单。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
