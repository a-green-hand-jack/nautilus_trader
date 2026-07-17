# Stop-Market（止损市价单）

`FIX OrdType <40>=3`（Stop）

*Stop-Market* 订单是一种条件订单，一旦被触发，将立即下达一笔 *Market* 订单。此订单类型常用作止损单以限制损失——针对多头持仓作为 SELL 单，或针对空头持仓作为 BUY 单。

## 使用场景

当价格水平被突破后你需要成交确定性时，使用 *Stop-Market* 订单，例如保护性止损或突破入场单。由于触发后会转换为 *Market* 订单，持仓几乎总能被开立或平仓。
代价在于触发价并非成交价：在快速变动或跳空的市场中，成交可能远远偏离止损价，因此它是以价格确定性换取成交确定性（与 *Stop-Limit* 相反）。

## 示例

在以下示例中，我们在 Binance Spot/Margin 交易所创建一笔 *Stop-Market* 订单，以 100,000 USDT 的触发价卖出 1 枚 BTC，有效期至另行通知：

```rust tab="Rust"
use nautilus_model::{
    enums::{OrderSide, TimeInForce, TriggerType},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};

let order = self.order().stop_market(
    InstrumentId::from("BTCUSDT.BINANCE"),
    OrderSide::Sell,
    Quantity::from(1),
    Price::from("100000"),
    Some(TriggerType::LastPrice), // optional (default DEFAULT)
    Some(TimeInForce::Gtc),       // optional (default GTC)
    None,                         // expire_time
    Some(false),                  // reduce_only (default false)
    None,                         // quote_quantity (default false)
    None,                         // display_qty
    None,                         // emulation_trigger
    None,                         // trigger_instrument_id
    None,                         // exec_algorithm_id
    None,                         // exec_algorithm_params
    None,                         // tags
    None,                         // client_order_id
);
```

```python tab="Python"
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model.enums import TriggerType
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model.orders import StopMarketOrder

order: StopMarketOrder = self.order_factory.stop_market(
    instrument_id=InstrumentId.from_str("BTCUSDT.BINANCE"),
    order_side=OrderSide.SELL,
    quantity=Quantity.from_int(1),
    trigger_price=Price.from_int(100_000),
    trigger_type=TriggerType.LAST_PRICE,  # <-- optional (default DEFAULT)
    time_in_force=TimeInForce.GTC,  # <-- optional (default GTC)
    expire_time=None,  # <-- optional (default None)
    reduce_only=False,  # <-- optional (default False)
    tags=None,  # <-- optional (default None)
)
```

更多详情请参见 [`StopMarketOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.stop_market.StopMarketOrder)。

## 相关指南

- [订单（Orders）](index.md#trigger-type) - 触发类型及其他执行指令。
- [模拟订单（Emulated orders）](emulated.md) - 在不原生支持的交易场所上模拟条件订单。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
