# 配置（Configuration）

NautilusTrader 在整个平台中使用类型化的配置结构体（typed configuration structs）。
每个组件（数据客户端、执行客户端、引擎、策略）都有一个
专用的配置结构体来控制其行为。

## 设计原则

### 默认值在配置边界处解析

配置结构体为始终具有合理默认值的字段携带具体的值。
超时、重试次数、退避延迟（backoff delays）和心跳间隔是像
`u64` 或 `u32` 这样内置默认值的普通类型。下游代码接收到的是已解析的值，
不需要重复默认值处理逻辑。

### `Option` 表示语义上的“缺失”，而不是“使用默认值”

`Option<T>` 字段仅在 `None` 具有真实含义时出现：某个功能被关闭、
某个回溯窗口（lookback window）无边界，或某个值在运行时从环境中继承。
如果一个字段总是会解析为一个具体的值，它就不会被包装在 `Option` 中。

这一区分使配置语义在类型上可见。一个普通的 `u64` 字段
总是有一个值。一个 `Option<u64>` 字段可能缺失，使用它的代码
会据此进行分支处理。

### 默认值的单一事实来源

每个配置结构体都使用 `bon::Builder`，通过 `#[builder(default = value)]`
注解在一处定义默认值。`Default` 实现委托给构建器
（`Self::builder().build()`），因此不存在可能失步的第二份默认值副本。

### 配置解码在遇到未知字段时会失败

配置解码在遇到未知字段时会快速失败。Nautilus 将多余的键视为
错误，而不是无害的输入。这可以在节点或客户端以错误设置启动之前，
捕获拼写错误、配置重命名后遗留的旧名称以及复制粘贴错误。

## Python 配置

Python 配置类（msgspec 结构体）对可选参数接受 `None`。
对于普通的 `T` 字段，`None` 表示“使用默认值”。对于 `Option<T>` 字段，
`None` 保留该字段的可选语义（禁用、无边界等）。

所有 Python 配置类都继承自 `NautilusConfig`，它会在底层的
`msgspec.Struct` 上设置 `forbid_unknown_fields=True`。未知的键现在会在
解码期间引发 `msgspec.ValidationError`。

```python
from nautilus_trader.adapters.bybit.config import BybitDataClientConfig

# All defaults: 60s timeout, 3 retries, etc.
config = BybitDataClientConfig()

# Override just the timeout
config = BybitDataClientConfig(http_timeout_secs=30)

# Disable instrument status polling
config = BybitDataClientConfig(instrument_status_poll_secs=None)
```

## Rust 配置

所有配置结构体都派生自 [`bon::Builder`](https://bon-rs.com)，它会生成
一个类型安全的构建器（builder），并对必填字段进行编译期检查。带有
`#[builder(default = value)]` 的字段可以在构建器调用中省略，将
使用其声明的默认值。有三种等价的方式来构造一个配置：

使用 Serde 反序列化的 Rust 配置结构体还设置了
`#[serde(deny_unknown_fields)]`。未知的键现在会导致反序列化失败，而不是被忽略。

```rust
// Builder: only set what differs from defaults
let config = BybitDataClientConfig::builder()
    .http_timeout_secs(30)
    .build();

// Struct literal with default spread
let config = BybitDataClientConfig {
    http_timeout_secs: 30,
    ..Default::default()
};

// Full defaults
let config = BybitDataClientConfig::default();
```

对于未指定的字段，以上三种方式产生的结果完全相同。

## 常见配置字段

大多数适配器配置共享一组常见字段：

| 字段                                | 类型   | 默认值   | 用途                       |
|------------------------------------|--------|---------|-------------------------------|
| `http_timeout_secs`                | `u64`  | 60      | REST 请求超时时间。         |
| `max_retries`                      | `u32`  | 3       | 最大重试次数。       |
| `retry_delay_initial_ms`           | `u64`  | 1,000   | 初始退避延迟。         |
| `retry_delay_max_ms`               | `u64`  | 10,000  | 最大退避延迟。         |
| `heartbeat_interval_secs`          | `u64`  | 因适配器而异 | WebSocket 保活（keepalive）间隔。 |
| `recv_window_ms`                   | `u64`  | 因适配器而异 | 签名请求的过期窗口。 |
| `update_instruments_interval_mins` | 因适配器而异 | 因适配器而异 | 定期刷新金融工具信息。 |

特定于适配器的字段（速率限制、轮询间隔、保证金模式）
记录在各自适配器的集成指南中。

## 引擎配置

引擎配置（`LiveExecEngineConfig`、`DataEngineConfig` 等）遵循相同的
模式。诸如 `reconciliation`、`inflight_check_interval_ms` 和
`open_check_threshold_ms` 之类的字段是带有构建器默认值的普通类型。真正可选的
功能则使用 `Option<T>`：

```python
from nautilus_trader.config import LiveExecEngineConfig

config = LiveExecEngineConfig(
    reconciliation=True,
    open_check_interval_secs=30.0,       # Enable open order polling
    open_check_lookback_mins=60,         # Look back 60 minutes
    # position_check_interval_secs=None  # Disabled by default
)
```
