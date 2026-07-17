# 编写 Actor（Rust）

Actor 接收市场数据、自定义数据/信号以及系统事件，但不负责管理订单。本指南
将带你构建一个 `SpreadMonitor`，它订阅报价（quote）数据，并记录买卖价差
（bid-ask spread）的日志。

有关 actor、trait 以及处理器（handler）分发机制的背景知识，请参见
[Actors](../concepts/actors.md) 和 [Rust](../concepts/rust.md) 概念指南。

## 定义结构体

一个 actor 拥有一个 `DataActorCore` 以及它所需的任何状态。该核心（core）存储
actor 的运行时状态。用户代码通常通过 `DataActor` facade 方法来访问该状态，例如：

- `clock()`
- `cache()`
- `config()`
- `actor_id()`
- `trader_id()`
- 各类订阅方法

```rust
use nautilus_common::{nautilus_actor, actor::{DataActor, DataActorConfig, DataActorCore}};
use nautilus_model::{data::QuoteTick, identifiers::{ActorId, InstrumentId}};

pub struct SpreadMonitor {
    core: DataActorCore,
    instrument_id: InstrumentId,
}
```

## 实现构造函数

创建一个带有 actor ID 的 `DataActorConfig`，然后将其传给 `DataActorCore::new`。
配置字段都使用 `Option` 并带有默认值，因此 `..Default::default()` 可以覆盖
除 actor ID 之外的所有字段。

```rust
impl SpreadMonitor {
    pub fn new(instrument_id: InstrumentId) -> Self {
        let config = DataActorConfig {
            actor_id: Some(ActorId::from("SPREAD_MON-001")),
            ..Default::default()
        };
        Self {
            core: DataActorCore::new(config),
            instrument_id,
        }
    }
}
```

## 接入核心并实现 Debug

`nautilus_actor!` 宏将 actor 的 `DataActorCore` 字段与运行时契约连接起来。
默认情况下，它会委托给一个名为 `core` 的字段；如需使用不同的字段名，可传入
第二个参数。普通的回调不会调用生成的原生（native）访问器；请在 `self` 上使用
`DataActor` facade 方法。

运行时注册使用了泛化的 `Actor` 和 `Component` 实现。该宏提供了原生运行时的
接线逻辑；`Debug` 需要手动实现或派生（derive）。

```rust
nautilus_actor!(SpreadMonitor);

impl std::fmt::Debug for SpreadMonitor {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("SpreadMonitor").finish()
    }
}
```

## 实现 DataActor trait

重写处理器方法以接收数据。所有处理器都有默认的空实现（no-op），因此你只需
重写自己需要的部分。每个处理器返回 `anyhow::Result<()>`。

```rust
impl DataActor for SpreadMonitor {
    fn on_start(&mut self) -> anyhow::Result<()> {
        self.subscribe_quotes(self.instrument_id, None, None);
        Ok(())
    }

    fn on_quote(&mut self, quote: &QuoteTick) -> anyhow::Result<()> {
        let spread = quote.ask_price.as_f64() - quote.bid_price.as_f64();
        log::info!("Spread: {spread:.5}");
        Ok(())
    }
}
```

`subscribe_quotes` 可以通过 `DataActor` trait 直接在 `self` 上调用。完整的
可用处理器列表请参见[处理器方法表](../concepts/rust.md#handler-methods)。

## 原生运行时访问

默认情况下请使用公共的 `DataActor` facade。只有在 facade 方法无法满足需求
的明确原生专用访问场景下，才需要添加 `DataActorNative`。以下只读属性
在 facade 上是可用的：

- `config()`
- `actor_id()`
- `trader_id()`
- `is_registered()`

[Rust 原生 trait](../concepts/rust.md#native-traits)一节涵盖了原生 trait
的适用性矩阵，以及以下方法表：

- [`DataActorNative` 方法](../concepts/rust.md#dataactornative-methods)

这些类型不会跨越 Python 边界，因此可移植（portable）的 actor 应当使用
诸如以下的 facade 方法：

- `clock()`
- `cache()`

## 注册 actor

使用 `BacktestEngine`：

```rust
let actor = SpreadMonitor::new(instrument_id);
engine.add_actor(actor)?;
```

使用 `LiveNode`：

```rust
let actor = SpreadMonitor::new(instrument_id);
node.add_actor(actor)?;
```

## 守卫（Guard）安全

当系统将消息分发给你的 actor 时，它会从注册表中获取一个短生命周期的
`ActorRef` 守卫。你不需要直接管理这些守卫。如果你编写的代码需要在回调
中访问其他 actor，请遵循以下规则：

- 每次都通过 ID 查找 actor；不要缓存 `ActorRef`。
- 在作用域结束前释放该守卫；切勿将其保存到某个字段中。
- 切勿在跨越 `.await` 点的情况下持有该守卫。

`DataActorCore` 上的订阅方法通过捕获 actor ID 并在回调闭包内部执行查找，
正确地处理了这一点。完整的线程与注册表模型请参见
[运行时不变式（Runtime invariants）](../developer_guide/rust.md#runtime-invariants)。

## 完整示例

参见
[`BookImbalanceActor`](https://github.com/nautechsystems/nautilus_trader/tree/develop/crates/trading/src/examples/actors/imbalance)，
这是一个更完整的 actor 示例，它跟踪按金融工具划分的状态，并在停止时打印摘要。
</content>
