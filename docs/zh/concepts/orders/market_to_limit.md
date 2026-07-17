# Market-To-Limit（市价转限价单）

`FIX OrdType <40>=K`（Market With Left Over as Limit）

*Market-To-Limit* 订单以市价单的形式按当前最优价格提交。若订单部分成交，系统将取消剩余部分，并以已成交价格重新提交为一笔 *Limit* 订单。

## 使用场景

当你希望立即获取最优价格上的可用流动性，而不希望以更差的价格扫掉更深层级的订单簿时，使用 *Market-To-Limit* 订单：这在流动性稀薄的订单簿中，
或对于希望获得最优价格但不希望产生扫单市场冲击的大额订单来说很有帮助。其优势是能够立即以最优价格成交，剩余部分则以 *Limit* 单挂在该价位，而不是继续追价。
代价是若市场随后走开，未成交的剩余部分可能一直无法成交。

## 示例

在以下示例中，我们在 Interactive Brokers [IdealPro](https://ibkr.info/node/1708) 外汇 ECN 上创建一笔 *Market-To-Limit* 订单，用 JPY 买入 200,000 USD：

```rust tab="Rust"
use nautilus_model::{
    enums::{OrderSide, TimeInForce},
    identifiers::InstrumentId,
    types::Quantity,
};

let order = self.order().market_to_limit(
    InstrumentId::from("USD/JPY.IDEALPRO"),
    OrderSide::Buy,
    Quantity::from(200_000),
    Some(TimeInForce::Gtc), // optional (default GTC)
    None,                   // expire_time
    Some(false),            // reduce_only (default false)
    None,                   // quote_quantity (default false)
    None,                   // display_qty (default full display)
    None,                   // exec_algorithm_id
    None,                   // exec_algorithm_params
    None,                   // tags
    None,                   // client_order_id
);
```

```python tab="Python"
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Quantity
from nautilus_trader.model.orders import MarketToLimitOrder

order: MarketToLimitOrder = self.order_factory.market_to_limit(
    instrument_id=InstrumentId.from_str("USD/JPY.IDEALPRO"),
    order_side=OrderSide.BUY,
    quantity=Quantity.from_int(200_000),
    time_in_force=TimeInForce.GTC,  # <-- optional (default GTC)
    reduce_only=False,  # <-- optional (default False)
    display_qty=None,  # <-- optional (default None which indicates full display)
    tags=None,  # <-- optional (default None)
)
```

更多详情请参见 [`MarketToLimitOrder` API 参考](/docs/python-api-latest/model/orders.html#nautilus_trader.model.orders.market_to_limit.MarketToLimitOrder)。

## 相关指南

- [订单（Orders）](index.md) - 订单概念、执行指令与订单工厂。
- [执行（Execution）](../execution.md) - 订单如何送达交易场所以及成交如何处理。
</content>
