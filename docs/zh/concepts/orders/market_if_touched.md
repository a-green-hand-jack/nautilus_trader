# Market-If-Touched（触价市价单）

`FIX OrdType <40>=J`（Market If Touched）

*Market-If-Touched* 订单是一种条件订单，一旦被触发，将立即下达一笔 *Market* 订单。此订单类型常用于在止损价位建立新持仓，
或为现有持仓获利了结——针对多头持仓作为 SELL 单，或针对空头持仓作为 BUY 单。

## 使用场景

当目标价格被触及时，若你需要以成交确定性来采取行动，可使用 *Market-If-Touched* 订单，例如在价格回调至某一水平时入场，或在目标价位止盈。
它的行为类似于反方向的止损单（在当前市场之下买入或之上卖出），并在触发时转换为 *Market* 订单。其代价与任何市价成交相同：触及价并非成交价，
在快速变动的市场中成交可能出现滑点。

## 示例

在以下示例中，我们在 Binance Futures 交易所创建一笔 *Market-If-Touched* 订单，以 10,000 USDT 的触发价卖出 10 份 ETHUSDT-PERP 永续期货合约，有效期至另行通知：

```rust tab="Rust"
use nautilus_model::{
    enums::{OrderSide, TimeInForce, TriggerType},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};
use ustr::Ustr;

let order = self.order().market_if_touched(
    InstrumentId::from("ETHUSDT-PERP.BINANCE"),
    OrderSide::Sell,
    Quantity::from(10),
    Price::from("10000.00"),
    Some(TriggerType::LastPrice),    // optional (default DEFAULT)
    Some(TimeInForce::Gtc),          // optional (default GTC)
    None,                            // expire_time
    Some(false),                     // reduce_only (default false)
    None,                            // quote_quantity (default false)
    None,                            // emulation_trigger
    None,                            // trigger_instrument_id
    None,                            // exec_algorithm_id
    None,                            // exec_algorithm_params
    Some(vec![Ustr::from("ENTRY")]), // tags
    None,                            // client_order_id
);
```

```python tab="Python"
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model.enums import TriggerType
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Price
from nautilus_trader.model import Quantity
from nautilus_trader.model.orders import MarketIfTouchedOrder

order: MarketIfTouchedOrder = self.order_factory.market_if_touched(
    instrument_id=InstrumentId.from_str("ETHUSDT-PERP.BINANCE"),
    order_side=OrderSide.SELL,
    quantity=Quantity.from_int(10),
    trigger_price=Price.from_str("10_000.00"),
    trigger_type=TriggerType.LAST_PRICE,  # <-- optional (default DEFAULT)
    time_in_force=TimeInForce.GTC,  # <-- optional (default GTC)
    expire_time=None,  # <-- optional (default None)
    reduce_only=False,  # <-- optional (default False)
    tags=["ENTRY"],  # <-- optional (default None)
)
```

更多详情请参见 [`MarketIfTouchedOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.market_if_touched.MarketIfTouchedOrder)。

## 相关指南

- [订单（Orders）](index.md#trigger-type) - 触发类型及其他执行指令。
- [模拟订单（Emulated orders）](emulated.md) - 在不原生支持的交易场所上模拟条件订单。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
