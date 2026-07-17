# 订单簿（Order Book）

NautilusTrader 提供了一个用 Rust 实现的高性能订单簿，能够为受支持的
公共订单簿类型维护完整的簿状态。`OrderBook` 是用于跟踪公共市场深度的
主要组件，而 `OwnOrderBook` 则单独跟踪你自己的订单，从而支持
显示真实可用流动性的过滤视图。

:::note
本指南记录的是 Rust API。这些类型也可以通过 PyO3 绑定
（`nautilus_pyo3.OrderBook`、`nautilus_pyo3.OwnOrderBook`）从 Python 中使用。由
`cache.order_book()` 返回的 v1 遗留 Cython `OrderBook`
（`nautilus_trader.model.book.OrderBook`）拥有类似但不完全相同的接口。
差异请参阅 API 参考文档。
:::

## 订单簿类型

`OrderBook` 实例针对每个金融工具维护，同时适用于回测和实盘交易：

- `L3_MBO`：三级（Level 3）按订单聚合的市场数据（market-by-order，MBO）。跟踪
  每个价格档位上的每一笔订单，以订单 ID 为键。
- `L2_MBP`：二级（Level 2）按价格聚合的市场数据（market-by-price，MBP）。将
  订单按价格档位聚合（每个价格一条记录）。
- `L1_MBP`：一级（Level 1）按价格聚合的市场数据（MBP）顶档数据，也称为
  最优买卖价（best bid and offer，BBO）。仅捕获最优价格。

:::note
报价、成交和 K 线数据（`QuoteTick`、`TradeTick` 和 `Bar`）也可以驱动
`L1_MBP` 订单簿。
:::

## 订阅订单簿数据

策略和 Actor 通过以下方法订阅订单簿更新。
订阅方法和处理器属于 Python 策略/Actor 层：

```python
# Incremental book deltas
self.subscribe_order_book_deltas(instrument_id)

# Aggregated depth snapshots (up to 10 levels)
self.subscribe_order_book_depth(instrument_id)

# Full book snapshots at a timed interval
self.subscribe_order_book_at_interval(instrument_id, interval_ms=1000)
```

每种订阅类型都会将数据传递给相应的处理器：

```python
def on_order_book_deltas(self, deltas: OrderBookDeltas) -> None:
    ...

def on_order_book_depth(self, depth: OrderBookDepth10) -> None:
    ...

def on_order_book(self, order_book: OrderBook) -> None:
    ...
```

## 访问订单簿

`OrderBook` 暴露了顶档（top-of-book）访问方法：

```rust
let best_bid: Option<Price> = book.best_bid_price();
let best_ask: Option<Price> = book.best_ask_price();
let spread: Option<f64> = book.spread();
let midpoint: Option<f64> = book.midpoint();
```

## 分析方法

`OrderBook` 支持市场深度分析和执行模拟：

```rust
// Average fill price for a given quantity
let avg_px = book.get_avg_px_for_quantity(quantity, OrderSide::Buy);

// Average price and quantity for a target exposure (notional)
let (price, qty, exposure) =
    book.get_avg_px_qty_for_exposure(target_exposure, OrderSide::Buy);

// Cumulative quantity available at or better than a price
let qty = book.get_quantity_for_price(price, OrderSide::Buy);

// Quantity at a specific price level only
let qty = book.get_quantity_at_level(price, OrderSide::Buy, 2);

// Simulate fills against the book
let fills: Vec<(Price, Quantity)> = book.simulate_fills(&order);

// All crossed levels regardless of order quantity
let levels = book.get_all_crossed_levels(OrderSide::Buy, price, 2);
```

## 完整性检查

`book_check_integrity` 函数会校验订单簿状态是否与其类型
保持一致：

- **L1_MBP**：每一侧不得超过一个档位。
- **L2_MBP**：每个价格档位不得超过一笔订单。
- **L3_MBO**：没有结构性约束（任意档位可有任意数量的订单）。
- **所有类型**：最优买价不得超过最优卖价（即不能出现交叉盘）。锁定盘
  （买价 == 卖价）被视为有效。

这些检查在应用差量（delta）时会在内部运行。传入差量的金融工具 ID
也会与订单簿自身的金融工具 ID 进行校验，不匹配时返回
`BookIntegrityError::InstrumentMismatch`。

## 美化打印

`OrderBook` 和 `OwnOrderBook` 都提供了 `pprint` 方法，可将订单簿
渲染为一个人类可读的表格：

```rust
book.pprint(5, None);
book.pprint(5, Some(Decimal::new(1, 2))); // group_size = 0.01
```

`group_size` 参数将价格档位归并为更粗粒度的分组，适用于
最小报价单位（tick size）很小的金融工具。输出是一个格式化的表格，
买价在左侧，价格在中间，卖价在右侧。

## 自有订单簿（Own order book）

`OwnOrderBook` 将你自己的挂单与公共订单簿分开跟踪。做市及其他
报价策略用它来估算在扣除自身订单后，每个价格档位上的可用流动性。

当启用 `manage_own_order_books` 时，执行引擎会维护自有订单簿。
随着订单事件改变状态，缓存会更新已存在的自有订单簿。符合条件的订单
需要有价格，且有效期不为 `IOC` 或 `FOK`。即使某个订单原本不符合
跟踪条件，终止事件仍可能清理已存在的自有订单簿条目。

### 订单生命周期

`OwnOrderBook` 跟踪订单的整个生命周期。订单在提交时或从核对
（reconciliation）中具体化时被添加，随着状态变化事件到达而更新，
并在关闭时被移除。更新涵盖已接受、待更新、待撤销、部分成交、
完全成交、已撤销、已过期、已拒绝以及已否决等订单模型所支持的各种状态。

每个 `OwnBookOrder` 携带以下信息：

- `client_order_id`：用于将自有订单簿与缓存状态进行核对的客户端订单 ID。
- `venue_order_id`：如果已分配，则为交易场所订单 ID。
- `side`、`price` 和 `size`：订单方向、价格和剩余（leaves）数量。
- `order_type` 和 `time_in_force`：供过滤和诊断使用的订单类型元数据。
- `status`：当前订单状态，例如 `SUBMITTED`、`ACCEPTED` 或 `PENDING_CANCEL`。
- `ts_last`：应用于该自有订单簿订单的最新订单事件的时间戳。
- `ts_accepted`：订单被交易场所接受的时间戳。
- `ts_submitted`：订单被提交的时间戳。
- `ts_init`：订单被初始化的时间戳。

这些字段使得过滤视图可以按状态和接受时间来包含或排除自有订单
（参见[状态和时间过滤](#status-and-time-filtering)）。

### 审计

`audit_open_orders` 方法根据一组有效的客户端订单 ID 对自有订单簿
进行核对。任何不在给定集合中的自有订单簿订单都会被移除，并作为
审计错误记录下来。`Cache::audit_own_order_books` 会从未平仓订单和
在途订单构建这一集合，因此已提交的订单不会在正常的交易场所延迟
窗口内被移除。实盘系统可以通过自有订单簿审计间隔（own-books audit interval）
定期运行该审计。

### 查询

```rust
// Check if a specific order is tracked
let in_book = own_book.is_order_in_book(&client_order_id);

// Get all tracked order IDs per side
let bid_ids = own_book.bid_client_order_ids();
let ask_ids = own_book.ask_client_order_ids();

// Aggregated quantities per price level
let bid_qty = own_book.bid_quantity(None, None, None, None, None);
let ask_qty = own_book.ask_quantity(None, None, None, None, None);

// Pretty print
own_book.pprint(5, None);
```

### 过滤视图

从公共订单簿中减去你自己的订单，得到净可用流动性：

```rust
// Filtered maps of price -> quantity (own orders subtracted)
let net_bids = book.bids_filtered_as_map(Some(10), Some(&own_book), None, None, None);
let net_asks = book.asks_filtered_as_map(Some(10), Some(&own_book), None, None, None);

// Full filtered OrderBook with all analysis methods available
let filtered = book.filtered_view(Some(&own_book), Some(10), None, None, None);
let avg_px = filtered.get_avg_px_for_quantity(quantity, OrderSide::Buy);
```

`filtered_view` 方法返回一个减去了你自己订单数量的新 `OrderBook`，
使你可以在净订单簿上使用完整的分析方法集合（`spread`、`midpoint`、
`get_avg_px_for_quantity` 等）。

### 状态和时间过滤

过滤视图支持针对自有订单的可选状态和基于时间的过滤：

```rust
let status = Some(AHashSet::from([OrderStatus::Accepted]));

// Only subtract ACCEPTED orders (ignore SUBMITTED, PENDING_CANCEL, etc.)
let filtered = book.filtered_view(Some(&own_book), None, status, None, None);
```

`accepted_buffer_ns` 参数提供了一个宽限期：设置该参数后，只有
满足 `ts_accepted + buffer <= now` 的订单才会被纳入。这样可以排除
那些最近才被接受、可能尚未出现在公共订单簿数据流中的订单。该缓冲期
适用于 `ts_accepted` 字段，与订单状态无关。可以与状态过滤器结合使用，
以同时排除未被接受的订单。

```rust
// Only subtract orders accepted at least 500ms ago
let filtered = book.filtered_view(
    Some(&own_book),
    None,
    None,
    Some(500_000_000),
    Some(clock.timestamp_ns()),
);
```

## 二元市场（Binary markets）

对于二元/预测市场（例如 Polymarket），金融工具具有两个互补的
方向（YES 和 NO），其价格之和为 1.0。在 NO 一侧以 0.40 报出的买价，
在经济上等价于在 YES 一侧以 0.60 报出的卖价。

`OwnOrderBook::combined_with_opposite` 方法处理这种转换，
将来自双方的订单合并为单一视图：

```rust
let yes_own = own_yes_book
    .cloned()
    .unwrap_or_else(|| OwnOrderBook::new(yes_instrument_id));

let no_own = own_no_book
    .cloned()
    .unwrap_or_else(|| OwnOrderBook::new(no_instrument_id));

// Merge NO-side orders with parity price transform (1 - price)
let combined = yes_own.combined_with_opposite(&no_own).unwrap();

// Filter the public YES book using the combined own book
let filtered = book.filtered_view(Some(&combined), None, None, None, None);
```

该转换的工作方式如下：

- NO 一侧价格为 P 的卖价，在合并后的订单簿中变为价格为 1 - P 的买价。
- NO 一侧价格为 P 的买价，在合并后的订单簿中变为价格为 1 - P 的卖价。

这样就得到了你在市场双方自身流动性的完整图景。
