# Trailing-Stop-Market（跟踪止损市价单）

`FIX OrdType <40>=3`（Stop）+ 跟踪挂钩

*Trailing-Stop-Market* 订单是一种条件订单，其止损触发价会以固定偏移量跟踪所定义的市场价格。一旦触发，将立即下达一笔 *Market* 订单。

## 使用场景

使用 *Trailing-Stop-Market* 订单可以在锁定盈利的同时让持仓继续运行：触发价会以固定偏移量跟踪有利方向的行情变动，仅在行情反转时才会触发，无需手动调整。
其优势是动态保护结合触发后的成交确定性。代价在于偏移量的选择需要权衡：偏移过窄会带来反复止损（whipsaw）风险，偏移过宽则会让出更多盈利，
并且在剧烈反转行情中，市价成交仍可能出现滑点。

## 示例

在以下示例中，我们在 Binance Futures 交易所创建一笔 *Trailing-Stop-Market* 订单，卖出 10 份以 COIN_M 结算的 ETHUSD-PERP 永续期货合约，
在 5,000 USD 激活，随后以最新成交价的 1%（以基点表示）作为偏移量进行跟踪：

```rust tab="Rust"
use nautilus_model::{
    enums::{OrderSide, TimeInForce, TrailingOffsetType, TriggerType},
    identifiers::InstrumentId,
    types::{Price, Quantity},
};
use rust_decimal::Decimal;
use ustr::Ustr;

let order = self.order().trailing_stop_market(
    InstrumentId::from("ETHUSD-PERP.BINANCE"),
    OrderSide::Sell,
    Quantity::from(10),
    Decimal::from(100),                    // trailing_offset
    Some(TrailingOffsetType::BasisPoints), // optional (default PRICE)
    Some(Price::from("5000")),             // activation_price
    None,                                  // trigger_price (materializes from the offset on the first trail)
    Some(TriggerType::LastPrice),          // optional (default DEFAULT)
    Some(TimeInForce::Gtc),                // optional (default GTC)
    None,                                  // expire_time
    Some(true),                            // reduce_only (default false)
    None,                                  // quote_quantity (default false)
    None,                                  // display_qty
    None,                                  // emulation_trigger
    None,                                  // trigger_instrument_id
    None,                                  // exec_algorithm_id
    None,                                  // exec_algorithm_params
    Some(vec![Ustr::from("TRAILING_STOP-1")]), // tags
    None,                                  // client_order_id
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
from nautilus_trader.model.orders import TrailingStopMarketOrder

order: TrailingStopMarketOrder = self.order_factory.trailing_stop_market(
    instrument_id=InstrumentId.from_str("ETHUSD-PERP.BINANCE"),
    order_side=OrderSide.SELL,
    quantity=Quantity.from_int(10),
    activation_price=Price.from_str("5_000"),
    trigger_type=TriggerType.LAST_PRICE,  # <-- optional (default DEFAULT)
    trailing_offset=Decimal(100),
    trailing_offset_type=TrailingOffsetType.BASIS_POINTS,
    time_in_force=TimeInForce.GTC,  # <-- optional (default GTC)
    expire_time=None,  # <-- optional (default None)
    reduce_only=True,  # <-- optional (default False)
    tags=["TRAILING_STOP-1"],  # <-- optional (default None)
)
```

如果 `activation_price` 和 `trigger_price` 都被省略，则订单会在当前市场立即激活，其触发价将在首次跟踪更新时根据 `trailing_offset` 生成。

更多详情请参见 [`TrailingStopMarketOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.trailing_stop_market.TrailingStopMarketOrder)。

## 相关指南

- [订单（Orders）](index.md#trigger-offset-type) - 触发类型与跟踪偏移类型。
- [模拟订单（Emulated orders）](emulated.md) - 在不原生支持的交易场所上模拟跟踪止损单。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
