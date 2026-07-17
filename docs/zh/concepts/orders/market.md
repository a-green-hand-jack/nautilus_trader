# Market（市价单）

`FIX OrdType <40>=1`

*Market* 订单是交易者要求以最优可用价格立即成交指定数量的指令。你还可以指定多种有效期类型选项，并指明该订单是否仅用于减少某一持仓。

## 使用场景

当成交本身比确切价格更重要时，使用 *Market* 订单：紧急降低风险、进入快速变动的流动性市场，或穿越点差过窄的市场（等待的代价高于点差本身）。
其优势是近乎确定的即时成交。代价是没有价格保护：你需要支付点差并可能在稀薄或快速变动的市场中承受滑点，因此它更适合流动性充足的金融工具，而非流动性差的工具。

## 示例

在以下示例中，我们在 Interactive Brokers [IdealPro](https://ibkr.info/node/1708) 外汇 ECN 上创建一笔 *Market* 订单，用 USD 买入 100,000 AUD：

```rust tab="Rust"
use nautilus_model::{
    enums::{OrderSide, TimeInForce},
    identifiers::InstrumentId,
    types::Quantity,
};
use ustr::Ustr;

let order = self.order().market(
    InstrumentId::from("AUD/USD.IDEALPRO"),
    OrderSide::Buy,
    Quantity::from(100_000),
    Some(TimeInForce::Ioc),          // optional (default GTC)
    Some(false),                     // reduce_only (default false)
    None,                            // quote_quantity (default false)
    None,                            // exec_algorithm_id
    None,                            // exec_algorithm_params
    Some(vec![Ustr::from("ENTRY")]), // tags
    None,                            // client_order_id (auto-generated if None)
);
```

```python tab="Python"
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Quantity
from nautilus_trader.model.orders import MarketOrder

order: MarketOrder = self.order_factory.market(
    instrument_id=InstrumentId.from_str("AUD/USD.IDEALPRO"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(100_000),
    time_in_force=TimeInForce.IOC,  # <-- optional (default GTC)
    reduce_only=False,  # <-- optional (default False)
    tags=["ENTRY"],  # <-- optional (default None)
)
```

更多详情请参见 [`MarketOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.market.MarketOrder)。

## 相关指南

- [订单（Orders）](index.md) - 订单概念、执行指令与订单工厂。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
