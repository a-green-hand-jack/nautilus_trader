# 编写策略（Rust）

策略是在 actor 基础上扩展了订单管理功能。本指南将带你构建一个最简化的策略，
该策略订阅报价数据并提交市价单。请先阅读
[编写 Actor（Rust）](write_rust_actor.md)。

有关策略概念和订单管理的背景知识，请参见
[Strategies](../concepts/strategies.md) 和 [Rust](../concepts/rust.md) 概念指南。

## 定义结构体

策略存储一个 `StrategyCore` 字段用于运行时接线。常规的策略逻辑不会直接使用
该字段；请在 `self` 上使用 facade 方法。

```rust
use nautilus_common::actor::DataActor;
use nautilus_model::{
    data::QuoteTick,
    enums::OrderSide,
    identifiers::{InstrumentId, StrategyId},
    types::Quantity,
};
use nautilus_trading::{nautilus_strategy, strategy::{Strategy, StrategyConfig, StrategyCore}};

pub struct MyStrategy {
    core: StrategyCore,
    instrument_id: InstrumentId,
    trade_size: Quantity,
}
```

## 实现构造函数

`StrategyConfig` 接受一个 `strategy_id` 和一个 `order_id_tag`。该标签会被
附加到该策略产生的所有客户端订单 ID 上，从而在多个策略交易同一金融工具时
避免冲突。

```rust
impl MyStrategy {
    pub fn new(instrument_id: InstrumentId) -> Self {
        let config = StrategyConfig {
            strategy_id: Some(StrategyId::from("MY_STRAT-001")),
            order_id_tag: Some("001".to_string()),
            ..Default::default()
        };
        Self {
            core: StrategyCore::new(config),
            instrument_id,
            trade_size: Quantity::from("1.0"),
        }
    }
}
```

## 接入核心并实现 Debug

`nautilus_strategy!` 宏生成注册和 `Strategy` trait 实现所使用的原生运行时
接线逻辑。默认情况下，它会委托给一个名为 `core` 的字段；如需使用不同的字段
名，可传入第二个参数。该宏不会让你的策略或其 `StrategyCore` 解引用
（deref）到运行时内部结构。它还会添加 `config()` 方法，返回传递给
`StrategyCore::new` 的 `StrategyConfig`。

运行时注册使用了需要原生接线和 `Debug` 的泛化 `Actor` 和 `Component`
实现。该宏提供了原生接线逻辑；`Debug` 需要手动实现或派生。

```rust
nautilus_strategy!(MyStrategy);

impl std::fmt::Debug for MyStrategy {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("MyStrategy").finish()
    }
}
```

## 实现 DataActor trait

数据处理方式与 actor 中相同。在 `on_start` 中订阅，在处理器中响应。

```rust
impl DataActor for MyStrategy {
    fn on_start(&mut self) -> anyhow::Result<()> {
        self.subscribe_quotes(self.instrument_id, None, None);
        Ok(())
    }

    fn on_quote(&mut self, quote: &QuoteTick) -> anyhow::Result<()> {
        let order = self.order().market(
            self.instrument_id,
            OrderSide::Buy,
            self.trade_size,
            None, None, None, None, None, None, None,
        );
        self.submit_order(order, None, None, None)?;
        Ok(())
    }
}
```

`self.order()` 用于构建订单和订单列表。可用方法包括：

- `market`
- `limit`
- `stop_market`
- `stop_limit`
- `market_to_limit`
- `market_if_touched`
- `limit_if_touched`
- `trailing_stop_market`
- `trailing_stop_limit`
- `bracket`
- `create_list`
- `generate_client_order_id`
- `generate_order_list_id`

`submit_order` 可以通过该宏生成的 `Strategy` trait 实现直接在 `self` 上调用。

## 原生运行时访问

在策略逻辑中请使用公共 facade：

- `clock()`
- `cache()`
- `order()`
- `portfolio()`
- `strategy_id()`
- `Strategy` 上的订单管理方法

常规的策略代码不会导入 `DataActorNative` 或 `StrategyNative`，也不会调用
诸如以下的原生句柄（native handle）：

- `core()`
- `core_mut()`
- `strategy_core()`
- `strategy_core_mut()`
- `order_factory()`
- `order_factory_rc()`
- `portfolio_rc()`

这些原生句柄暴露的是借用（borrowed）的运行时状态，仅在引擎、运行时、注册、
PyO3、testkit 或明确面向延迟敏感场景的原生代码中使用。
[Rust 原生 trait](../concepts/rust.md#native-traits)一节涵盖了原生 trait
的适用性矩阵，以及以下方法表：

- [`DataActorNative` 方法](../concepts/rust.md#dataactornative-methods)
- [`StrategyNative` 方法](../concepts/rust.md#strategynative-methods)

## 重写 Strategy 钩子方法

如需重写 `Strategy` trait 方法（例如订单或仓位事件处理器），请将其放在一个
代码块中传入。该宏会自动生成内部接线逻辑；请将 `DataActor` 处理器保留在
单独的 `impl DataActor` 代码块中。

```rust
nautilus_strategy!(MyStrategy, {
    fn on_order_rejected(&mut self, event: OrderRejected) {
        log::warn!("Order rejected: {}", event.reason);
    }
});
```

## 订单管理方法

`Strategy` trait 提供以下 facade 方法：

| 方法                | 作用                                          |
|-----------------------|-------------------------------------------------|
| `submit_order`        | 向交易场所提交一笔新订单。                |
| `submit_order_list`   | 提交一组连带订单（contingent order）。             |
| `modify_order`        | 修改价格、数量或触发价格。       |
| `modify_orders`       | 修改同一金融工具下的多个订单。 |
| `cancel_order`        | 取消特定订单。                |
| `cancel_orders`       | 取消经过筛选的一组订单。             |
| `cancel_all_orders`   | 取消某个金融工具的所有订单。            |
| `close_position`      | 以市价单方式平掉某个仓位。           |
| `close_all_positions` | 平掉所有未平仓仓位。                     |

## 完整示例

- [`EmaCross`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/trading/src/examples/strategies/ema_cross)：
  搭配指标集成的双 EMA 交叉策略。
- [`GridMarketMaker`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/trading/src/examples/strategies/grid_mm)：
  带有可配置档位和重新报价（requoting）功能的网格做市策略。
</content>
