# Limit-If-Touched（触价限价单）

`FIX OrdType <40>` 无专用值（通常以 `4` Stop Limit 加一个有利的触发价发送）

*Limit-If-Touched* 订单是一种条件订单，一旦被触发，将立即以指定价格下达一笔 *Limit* 订单。

## 使用场景

使用 *Limit-If-Touched* 订单，可以让一笔价格受保护的订单仅在触发价被触及后才被激活，例如在价格接近目标位时才激活止盈 *Limit* 单，
而不是提前挂单。其优势在于条件激活与限定成交价格相结合。与 *Stop-Limit* 类似，其代价是：若价格在触发后穿越了限价，订单可能无法成交。

## 示例

在以下示例中，我们在 Binance Futures 交易所创建一笔 *Limit-If-Touched* 订单，以 30,100 USDT 的限价买入 5 份 BTCUSDT-PERP 永续期货合约
（一旦市场达到 30,150 USDT 的触发价），有效期至 2022 年 6 月 6 日中午（UTC）：

```rust tab="Rust"
use nautilus_core::UnixNanos;
use nautilus_model::{
    enums::{OrderSide, TimeInForce, TriggerType},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};
use ustr::Ustr;

let order = self.order().limit_if_touched(
    InstrumentId::from("BTCUSDT-PERP.BINANCE"),
    OrderSide::Buy,
    Quantity::from(5),
    Price::from("30100"),
    Price::from("30150"),
    Some(TriggerType::LastPrice), // optional (default DEFAULT)
    Some(TimeInForce::Gtd),       // optional (default GTC)
    Some(UnixNanos::from(1_654_516_800_000_000_000_u64)), // 2022-06-06T12:00:00 UTC
    Some(true),                   // post_only (default false)
    Some(false),                  // reduce_only (default false)
    None,                         // quote_quantity (default false)
    None,                         // display_qty
    None,                         // emulation_trigger
    None,                         // trigger_instrument_id
    None,                         // exec_algorithm_id
    None,                         // exec_algorithm_params
    Some(vec![Ustr::from("TAKE_PROFIT")]), // tags
    None,                         // client_order_id
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
from nautilus_trader.model.orders import LimitIfTouchedOrder

order: LimitIfTouchedOrder = self.order_factory.limit_if_touched(
    instrument_id=InstrumentId.from_str("BTCUSDT-PERP.BINANCE"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(5),
    price=Price.from_str("30_100"),
    trigger_price=Price.from_str("30_150"),
    trigger_type=TriggerType.LAST_PRICE,  # <-- optional (default DEFAULT)
    time_in_force=TimeInForce.GTD,  # <-- optional (default GTC)
    expire_time=pd.Timestamp("2022-06-06T12:00"),
    post_only=True,  # <-- optional (default False)
    reduce_only=False,  # <-- optional (default False)
    tags=["TAKE_PROFIT"],  # <-- optional (default None)
)
```

更多详情请参见 [`LimitIfTouchedOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.limit_if_touched.LimitIfTouchedOrder)。

## 相关指南

- [订单（Orders）](index.md#trigger-type) - 触发类型及其他执行指令。
- [模拟订单（Emulated orders）](emulated.md) - 在不原生支持的交易场所上模拟条件订单。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
