# DST

**确定性仿真测试（Deterministic simulation testing，DST）** 在一个由种子（seed）控制的运行时下运行
NautilusTrader，使得对时序敏感的执行行为可以从单个整数按位（bitwise）
重现。本指南解释了什么是 DST、NautilusTrader 如何支持它、这种支持提供了
哪些保证，以及这些保证的边界在哪里。

目标是提供一份外部用户和审计人员可以核实的公开契约：NautilusTrader
所声称的确定性由源代码层面的证据支撑，并由一个在持续集成中运行的
预提交钩子在提交时强制执行。

## 简介

### 什么是 DST

DST 是一种针对并发系统的测试技术。单个种子完全决定了一次执行，
包括任务调度、定时器触发和随机值。使用相同种子、相同二进制文件和
相同配置的两次运行会产生完全相同的可观察行为。当某个属性测试失败时，
该种子就是复现方式：相同的种子每次都能重放该失败。

异步运行时中的调度决策来自环境中的进程状态：任务唤醒顺序、
定时器分辨率、线程调度、哈希种子。这些都不受测试装置控制，
这就是为什么在 CI 中偶尔出现一次的竞争条件通常很难按需复现。DST
用一个带种子的伪随机序列取代了这些环境来源，因此交错执行顺序成为
该种子的函数。

FoundationDB 从大约 2009 年开始将这一模式应用于一个生产级分布式数据库；
在 Rust 生态系统中，[madsim](https://github.com/madsim-rs/madsim) 拦截了
`tokio` 原语，提供了一个确定性调度器。

DST 所针对的漏洞，正是那些逃过单元测试、集成测试、属性测试和验收测试的
漏洞：通道唤醒顺序、关闭时的排空竞争、启动时序、核对顺序、恢复路径的
正确性。这些都涉及其他测试层无法穷举覆盖，但确定性调度器可以系统性
探索的交错情形。

### 本指南涵盖的内容

NautilusTrader 的 DST 支持分为两部分：

- **契约**：运行时在种子控制的执行下保证了什么，以及在何种条件下保证。
- **强制执行**：实现该契约的源代码层面接缝（seams），以及维持这些接缝
  到位的预提交钩子。

## 目标

- 对 NautilusTrader 运行时的范围内部分提供**种子可复现的执行**。
- **诚实的范围界定**。该契约列出了它涵盖的内容以及不涵盖的内容。
  不会静默回退到真实挂钟时间或无种子的 RNG；削弱该保证的条件都被
  明确列举出来。
- **在源代码层面强制执行**。一个预提交钩子会使向 DST 路径添加禁止模式的
  提交失败，因此该契约的成立不依赖于审查人员的注意力。
- **最小必要的插桩**。这些接缝仅在契约要求的地方才将时间、任务调度和
  随机性路由到一个确定性来源；其他一切照常运行。

## 方法

`madsim` 只对通过其别名子模块（`time`、`task`、`runtime`、`signal`）
路由的 `tokio` 原语进行确定化处理。挂钟读取、单调时钟读取、RNG 抽取、
哈希遍历以及 `select!` 轮询完全绕过了 `tokio`，需要它们自己的接缝。
第 1 层将这些别名子模块替换为 `madsim`；第 2 层提供这些接缝。

### 第 1 层：运行时替换

在 `nautilus-common` 的 `simulation` Cargo 功能下，当设置了
`RUSTFLAGS="--cfg madsim"` 时，四个 `tokio` 子模块会被路由到 `madsim`：

- `time`（定时器、间隔、单调 `Instant`）。
- `task`（异步任务的生成和合并）。
- `runtime`（运行时构建器和句柄）。
- `signal`（进程信号，例如 `ctrl_c`；重新导出可用，调用点的采用情况
  是部分的，详见[范围边界](#signal-handling)中的说明）。

这些重新导出位于 `nautilus_common::live::dst` 中。`time`、`task` 和
`runtime` 的 DST 路径调用点从这个模块导入，而不是直接从 `tokio` 导入，
因此切换该功能标志会在一个地方为已完全路由的原语切换异步运行时。
在正常构建中，这些重新导出解析为真实的 `tokio`。在 `simulation` +
`cfg(madsim)` 下，它们解析为 `madsim` 的确定性对应实现。

`tokio` 提供的其他一切（`sync`、`io`、作为宏的 `select!`、`fs`、`net`）
无条件地使用真实的 `tokio`。传递依赖 crate（`tokio-tungstenite`、
`tokio-rustls`、`reqwest`）不受影响。

### 第 2 层：非确定性替换

异步运行时之外的非确定性会被重定向到明确的接缝：

- **挂钟读取**通过 `nautilus_core::time::duration_since_unix_epoch`。
  在仿真下，这会路由到 `madsim::time::TimeHandle::try_current()`，
  为订单和成交时间戳保留了 Unix 纪元语义。当在 madsim 运行时之外调用时
  （例如普通的 `#[rstest]` 测试主体），它会回退到 `SystemTime::now()`，
  在 `cfg(madsim)` 下该调用会被 libc 拦截到正常构建所使用的同一个真实
  系统调用。在仿真下，生产路径总是运行在一个运行时内部，因此它们
  会持续接收虚拟时间。
- **单调时钟读取**通过 `nautilus_common::live::dst::time::Instant`。
  该类型在正常构建下解析为 `tokio::time::Instant`（以便兼容
  `tokio::test(start_paused)` 测试辅助工具），在仿真下解析为
  `madsim::time::Instant`。
- **网络本地的单调读取**通过 `nautilus_network::dst::time`。该 crate 在
  依赖图中位于 `nautilus-common` 之下，暴露了一个具有相同语义的本地
  重新导出模块。
- 核对管理器和订单撮合引擎中的**哈希遍历顺序**使用 `IndexMap` 和
  `IndexSet`，而不是 `AHashMap` 和 `AHashSet`。`AHash` 每个进程会随机化
  其哈希器；在顺序驱动下游事件发布或带种子的 `FillModel` RNG 被消费的
  顺序的场景下，需要按插入顺序遍历。
- **`tokio::select!` 轮询顺序**在 DST 路径上的每个生产环境调用点都使用
  `biased;` 修饰符。非 biased 的 `select!` 会以一个未被拦截的 RNG 选定的
  顺序轮询各分支。

## 确定性契约

在下述条件下，由 `(seed, binary hash, configuration hash)` 标识的一次运行，
在相同平台上会产生按位相同的：

1. 异步任务的调度顺序。
2. 定时器触发（虚拟单调时钟和虚拟挂钟）。
3. 来自 `madsim::rand` 的 RNG 输出。
4. `tokio` 原语上的通道投递顺序。

### 所需条件

只有在以下条件全部成立时，该契约才成立：

1. `simulation` Cargo 功能处于激活状态，且设置了
   `RUSTFLAGS="--cfg madsim"`。两者缺一不可。该功能激活确定性运行时；
   该 cfg 标志激活 `madsim` 在 libc 层面对 `clock_gettime` 和 `getrandom`
   的拦截。只满足其中之一会静默回退到真实的 `tokio`，在不报错的情况下
   破坏确定性。
2. DST 路径上的每一个 `tokio::select!` 调用点都使用 `biased;` 修饰符。
3. 单调时钟读取通过 DST 接缝路由（`nautilus_common::live::dst::time` 或
   `nautilus_network::dst::time`），而不是直接使用
   `std::time::Instant::now`。
4. 挂钟时间读取通过 `nautilus_core::time::duration_since_unix_epoch` 路由。
5. 随机性通过 `madsim::rand` 路由。`rand::thread_rng`、`rand::rng()`、
   `fastrand`、`getrandom` 和 `OsRng` 不会被拦截。
6. 对遍历顺序敏感的集合使用 `IndexMap` 或 `IndexSet`，而不是
   `AHashMap` 或 `AHashSet`。
7. 在仿真下，`tokio::task::LocalSet` 的构造被 cfg 排除在外。`madsim`
   不提供 `LocalSet`；`spawn_local` 无需它也能工作。
8. `tokio::task::spawn_blocking` 调用点被 cfg 排除或移除。一次阻塞调用
   会逃离确定性调度器。

## 静态强制执行

静态强制执行分为两层：

- `clippy.toml` 和 `[workspace.lints.clippy]` 中的 Clippy 策略会阻止那些
  在整个工作区 DST 契约下无效的 API：直接调用
  `getrandom::{fill,u32,u64}` 以及 `tokio::task::LocalSet`。
- 一个名为 `check-dst-conventions` 的预提交钩子强制执行了 Clippy 无法
  清晰表达的、限定范围的、感知路径和 cfg 的结构性检查。

该钩子位于 `.pre-commit-hooks/check_dst_conventions.sh`，是标准预提交
套件的一部分，并在持续集成中运行。规则 1 到 6 覆盖 16 个范围内的
工作区 crate。原始 Tokio 接口规则覆盖了 madsim 构建路径上的九个 crate。
当检测到以下任一情况时，该钩子会使提交失败：

- 直接读取 `std::time::Instant::now()`、`SystemTime::now()` 或
  `chrono::Utc::now()`，包括当所在文件从 `std::time` 导入了该类型，
  或从 `chrono` 导入了 `Utc` 时的裸调用形式。
- 未经 cfg 排除的原始 RNG 使用（`rand::thread_rng`、`rand::rng()`、
  `fastrand::`、`getrandom::`、`OsRng`）或 `Uuid::new_v4()`。
- `tokio::select!` 代码块的前三行中缺少 `biased;`。
- `std::thread::spawn`、`std::thread::Builder::new` 或
  `tokio::task::spawn_blocking` 调用，且前面缺少
  `#[cfg(test)]`、`#[cfg(not(madsim))]` 或
  `#[cfg(not(all(feature = "simulation", madsim)))]` 属性。
- 在对遍历顺序敏感、且位于 DST 路径上的文件中使用了 `AHashMap` 或
  `AHashSet`。完整的文件集合正在审核中；目前的强制执行覆盖了
  `crates/live/src/manager.rs` 和
  `crates/execution/src/matching_engine/engine.rs`，并会随着更多文件被
  审核而扩展。
- 绕过 `nautilus_network::net` 的直接 `tokio::net::TcpStream::connect` /
  `tokio::net::TcpListener::bind` 调用。该接缝在正常构建下重新导出
  `tokio::net` 类型，在 `turmoil` 功能下切换为 `turmoil::net`，因此所有
  TCP 入口点共享同一个受 cfg 控制的切换点。
- 在 madsim 构建路径的生产代码中直接使用
  `tokio::{time,task,runtime,signal}` 路径。调用方应通过
  `nautilus_common::live::dst` 路由这些模块；该接缝的定义本身、
  进程范围的真实 Tokio 运行时以及测试基础设施是明确的例外。

该钩子支持两种例外形式：

- 在特定行上的一个内联 `// dst-ok` 标记，通常附带一个简短的原因
  （例如，不影响状态的仅用于日志的挂钟计时）。
- 该钩子脚本本身内的一个小型文件级白名单，用于在代码库审计中被归类为
  “保持不变”的站点（缓存模块中的日志计时、DeFi 模块中的进度报告）。

测试文件、位于 `tests/`、`python/` 和 `ffi/` 目录下的文件，以及内联
`#[cfg(test)]` 模块内的代码行都被排除在外，因为它们不属于 DST 路径。

### 范围内的 crate

该钩子适用于 `nautilus-live` 传递闭包中的 16 个工作区 crate：

- `analysis`、`common`、`core`、`cryptography`、`data`、`execution`、
  `indicators`、`live`、`model`、`network`、`persistence`、`portfolio`、
  `risk`、`serialization`、`system`、`trading`。

适配器 crate 和基础设施 crate（Redis、Postgres）不在范围内。它们要
进入 DST 路径，需要在此之前进行单独的 DST 适用性审计。

## 网络种子浸泡测试（Seed soaks）

`nautilus-network` 的 Turmoil 测试使用两层：

- 固定种子测试在每晚的测试套件中运行。这些测试涵盖连接、重连、
  分区、重连期间关闭、退避期间关闭，以及重复的服务端断开场景，
  均使用可复现的种子。
- 一个被忽略的重连浸泡测试会遍历 Turmoil 种子，直到被停止，或者直到
  运行了 `NAUTILUS_TURMOIL_SOAK_COUNT` 个种子为止。当启用了
  `transport-sockudo` 时，每个种子会先运行 Tungstenite WebSocket 后端，
  再运行 Sockudo 后端，因此两种后端都能看到相同的调度搜索路径。

以持续方式运行浸泡测试：

```bash
scripts/soak-network-turmoil.sh
```

运行有限次数的浸泡测试：

```bash
env NAUTILUS_TURMOIL_SOAK_COUNT=100 scripts/soak-network-turmoil.sh
```

该浸泡测试使用确定性种子遍历、随机节点顺序、随机化的链路延迟、
重复的服务端断开、重连状态循环，以及精确的应用消息顺序检查。
它不会启用 Turmoil 的 `fail_rate`：对于 TCP，这会在没有重传模型的情况下
破坏链路，从而夸大对订单保序测试而言的客户端投递契约。

这些 Turmoil 测试针对模拟网络运行，不限定于 Linux。若干真实的本地
套接字和 WebSocket 单元测试使用 `target_os = "linux"` 以保证 CI 的
稳定性，因此 macOS 本地运行不会执行该主机的 TCP 覆盖测试。在将完整的
网络测试集视为已覆盖之前，请使用 macOS 进行 Turmoil 种子遍历测试，
并使用 Linux CI 或 Linux 工作站。

## 实现说明

DST 审计在本代码库中产生的具体变更。在调查某段代码路径是否位于
DST 路径上、以及它当前如何路由时，可以以此作为起点。

### 遍历顺序接缝

生产环境中将 `AHashMap` / `AHashSet` 改为 `IndexMap` / `IndexSet` 的位置，
因为其遍历顺序在 DST 路径上是可观察的：

- **撮合引擎**（`crates/execution/src/matching_engine/engine.rs`）：
  九个字段（`execution_bar_types`、`execution_bar_deltas`、`account_ids`、
  `cached_filled_qty`、`bid_consumption`、`ask_consumption`、`queue_ahead`、
  `queue_excess`、`queue_pending`）。遍历式删除使用 `.shift_remove()`。
  关闭了 [#3914](https://github.com/nautechsystems/nautilus_trader/issues/3914)。
- **核对管理器**（`crates/live/src/manager.rs`）：由钩子强制执行，
  另外还有 `ReconciliationResult.orders` 和 `ReconciliationResult.fills`。
- **Account trait**（`crates/model/src/accounts/`）：`balances`、
  `balances_total`、`balances_free`、`balances_locked`、`starting_balances`
  的返回值。`BaseAccount` 和 `MarginAccount` 上的存储字段是 `IndexMap`。
- **仓位事件**（`crates/model/src/position.rs`）：`Position::commissions`
  改为 `IndexMap`（在 `events/position/snapshot.rs` 中通过 `.values()` 消费）。
- **投资组合聚合**（`crates/portfolio/src/portfolio.rs`）：
  `unrealized_pnls`、`realized_pnls`、`net_positions` 的存储；
  `accumulate_mark_values` 构建 `IndexMap<Currency, f64>`。
- **数据引擎**（`crates/data/src/engine/`）：`book_snapshot_counts`、
  `bar_aggregators`、`BookSnapshotInfos`。遍历式删除使用
  `.shift_remove()`。
- **执行引擎**（`crates/execution/src/engine/`）：`ExecutionEngine.clients`，
  以及 `get_clients_for_orders()` 中的 `client_ids` / `venues` 累加器。
- **交易算法**（`crates/trading/src/algorithm/core.rs`）：
  `strategy_event_handlers`（驱动有序的 `msgbus::unsubscribe_*` 扇出）。
- **分析器**（`crates/analysis/src/analyzer.rs`）：`account_balances`、
  `account_balances_starting`。
- **缓存 API**（`crates/common/src/cache/mod.rs`）：`get_orders_for_ids`
  和 `get_positions_for_ids` 在返回其 `Vec` 结果之前按
  `client_order_id` / `position_id` 排序。存储仍保留在
  `AHashSet` 上（集合语义）。

范围内 crate 中剩余的 `AHashMap` / `AHashSet` 位置，要么仅用于查找、
要么位于并发共享所有权包装器（`Arc<DashMap>`、`AtomicMap`）之后，
要么被送入可交换聚合中。任何新出现的、会驱动可观察遍历顺序的
范围内位置，都是每个区域审计所防范的一次回归。

### 时间接缝

仍存留在 DST 路径上的 `Instant::now` / `SystemTime::now` 调用点，
要么位于 `#[cfg(test)]` 内，要么在钩子中被文件级白名单允许，
要么携带一个附带原因的内联 `// dst-ok` 标记：

- `crates/common/src/testing.rs:81,108` `wait_until` / `wait_until_async`
- `crates/execution/src/engine/mod.rs:822,847` 初始化日志计时
- `crates/common/src/cache/mod.rs:569,904,3895` 日志和审计计时（文件级白名单）
- `crates/model/src/defi/reporting.rs:59,123` 进度日志（文件级白名单）
- `crates/core/src/time.rs` 接缝定义位置（文件级白名单）

`chrono::Utc::now` 在范围内 crate 中被该钩子禁止。剩余的调用点是
日志桥接和写入器（在“日志运行在真实 OS 线程上”中被划出范围）。
过去 `crates/core/src/datetime.rs::is_within_last_24_hours` 辅助函数
会从非日志路径调用到 `chrono::Utc::now`；现在它改为通过
`nautilus_core::time::nanos_since_unix_epoch()` 路由，并直接以 `u64`
纳秒进行比较。

### 随机性接缝

DST 路径上生产环境的 RNG 位置：

- `crates/core/src/uuid.rs::UUID4::new()` 在仿真下于 madsim 运行时内部
  调用时，通过 `madsim::rand::thread_rng()` 路由，在运行时之外
  （以及在正常构建下）回退到 `rand::rng()`。在仿真下，生产路径总是
  运行在一个运行时内部，因此它们会消费带种子的字节；在 `cfg(madsim)`
  下的普通 `#[rstest]` 测试则使用主机 RNG。可从 `nautilus-common` 和
  `nautilus-risk` 中的订单和事件工厂到达。
- `crates/execution/src/models/fill.rs::default_std_rng()` 以相同方式路由。
  当未提供种子时，由 `ProbabilisticFillState::new()` 调用。提供了种子时，
  `StdRng::seed_from_u64` 在构造上就是确定性的。
- `crates/execution/src/matching_engine/ids_generator.rs:167,179` 在
  `use_random_ids` 路径中使用 `nautilus_core::UUID4::new()`。默认的 ID
  方案（`{venue}-{raw_id}-{count}`）不需要它即为确定性的。

带标记的允许项：`crates/network/src/backoff.rs:105` 用于重连抖动，
标记为 `// dst-ok`（传输层）。

### Tokio 子模块拆分

`madsim` 对 `time`、`task`、`runtime` 和 `signal` 进行了别名替换。
其他 tokio 子模块（`sync`、`io`、`select!`、`fs`、`net`）在仿真下仍保留
真实 tokio。进一步扩展该替换将需要针对被垫片替换后的
`tokio::net::TcpStream` 重新构建 `tokio-tungstenite`、`tokio-rustls`
和 `reqwest`，审计认为这样做侵入性太强，因此排除在外。

范围内直接接触真实 `tokio::net` / `tokio::io` 的位置：

- `crates/network/src/net.rs:37` 重新导出
  `tokio::net::{TcpListener, TcpStream}`
- `crates/network/src/socket/client.rs:46,356`
  `tokio::io::{AsyncReadExt, AsyncWriteExt}`
- `crates/network/src/tls.rs:22` `tokio::io::{AsyncRead, AsyncWrite}`
- `crates/network/src/websocket/types.rs:26,29` 为
  `MaybeTlsStream<tokio::net::TcpStream>` 设置别名

即使在仿真下，这些也运行在真实的套接字上。`tokio::sync` 上的通道投递
顺序仍然是确定性的，因为发送方和接收方任务是由 madsim 执行器调度的，
即使通道实现本身是真实的。

### 原始线程逃逸规则

该钩子的规则 4 禁止原始线程生成，除以下三种逃逸情况外：

- `#[cfg(test)]` 测试模块。
- `#[cfg(not(madsim))]` 或
  `#[cfg(not(all(feature = "simulation", madsim)))]` 的生产环境位置
  （例如日志写入器线程）。
- 一个内联的 `// dst-ok` 标记。

`madsim` 不支持 `tokio::task::LocalSet` 和
`tokio::task::spawn_blocking`。代码库审计在范围内的 crate 中未发现
这两者的任何生产环境位置；新出现的位置必须带有 cfg 排除或
`// dst-ok` 标记。

### 仿真下的日志测试

在仿真下，日志写入器线程被 cfg 排除在外；在 `cfg(madsim)` 下，
日志事件会被丢弃。初始化文件日志写入器的测试要么会挂起，要么会针对
一个空日志文件进行断言，因此受影响的子模块在模块边界被排除：

- `crates/common/src/logging/logger.rs::tests::serial_tests`（八个测试）。
- `crates/common/src/logging/macros.rs::tests`（两个测试）。

`logger.rs::tests::sim_tests::test_init_under_madsim_skips_writer_thread_and_forces_bypass`
在仿真下运行，并固定了这一被排除的行为。

## 范围边界

该契约有意保持狭窄。以下削弱情况是明确说明的，而不是疏漏。

### Python 不在 DST 范围内

DST 运行在一个原生 Rust 测试装置下。在一次 DST 运行期间不会启动任何
Python 解释器。`crates/*/src/python/` 下的 PyO3 绑定、`ffi/` 目录，
以及 `nautilus_trader/` 下的 Python 包，都作为一项策略被排除在该契约之外，
而不是作为一个弱点。任何只能从 Python 调用路径到达的代码都不在范围内；
任何从原生 DST 测试装置可到达的 Rust 路径都必须满足该契约，
即使相同的类型也被导出给了 Python。

`check-dst-conventions` 钩子通过在范围内的 crate 中跳过 `/python/` 和
`/ffi/` 路径来编码这一策略。这些路径背后的时钟、RNG 和线程调用点
不适用于该契约。

DST 的主要目标是 Rust 引擎自身的可靠性：订单生命周期、核对、撮合、
风险和执行状态机。用户策略的确定性重放是一个之后才实现的次要目标，
一旦策略是用 Rust 编写的，或通过一个 Rust 原生测试装置运行，
这个目标就会变得可用。与此同时，一个调用 `time.time()`、发出任意
网络请求或依赖线程调度的 Python 策略，其命令流可能在不同运行之间
发生变化；Rust 核心会确定性地处理这个变化的数据流，但从 Python 入口点
开始的端到端重放并不受保证。

### 限定于特定平台

`madsim` 对 `clock_gettime` 和 `getrandom` 的 libc 覆盖是特定于平台的。
不声称跨平台的按位可复现性。一个能在 Linux x86_64 上复现某个故障的
种子，可能无法在 macOS aarch64 上复现。

### 未被别名替换的依赖会静默逃逸

任何通过未被别名替换的路径（直接的 `libc` 调用、绕过 `std::net`、
使用 `fastrand` 或 `OsRng` 的 crate）访问操作系统的依赖，都会在不
引发错误的情况下逃离仿真器。范围内的 crate 已经过审计；适配器 crate
和基础设施 crate 在进入 DST 路径之前需要各自单独的审计。

### 传输层 I/O 未被仿真

`tokio-tungstenite`、`tokio-rustls`、`reqwest`、`redis` 和 `sqlx` 内部
使用真实的 `tokio`。在仿真下，WebSocket 和 HTTP I/O 运行在真实的网络上。
这是有意为之的：最初的目标是订单生命周期的确定性，而不是传输层故障
注入。传输层的确定性将需要目前尚不存在的按 crate 划分的 `madsim` 垫片。

驱动真实本地套接字的测试模块（`crates/network/src/socket/client.rs::tests`、
`::rust_tests`；`crates/network/src/websocket/client.rs::tests`、
`::rust_tests`；`crates/network/tests/websocket_proxy.rs`）在
`all(feature = "simulation", madsim)` 下被 cfg 排除在外，因为它们的
生产代码路径会到达 `dst::time::*`（madsim 时间原语），当从
`#[tokio::test]` 运行时调用时会 panic。重试测试模块
（`crates/network/src/retry.rs::tests`、`::proptest_tests`）在仿真下运行：
每个测试属性都通过 `cfg_attr` 在 `#[tokio::test(start_paused = true)]` 和
`#[madsim::test]` 之间切换，时间读取和 sleep 通过 `crate::dst::time` 路由，
显式的虚拟时间推进通过一个受 `cfg` 控制的 `advance_clock` 辅助函数进行，
因此同一个测试主体可以覆盖两种运行时。

### 信号处理

`nautilus_common::live::dst::signal` 暴露了一个被路由的 `ctrl_c`
重新导出。`crates/live/src/node.rs` 的运行循环通过它路由，因此在
`cfg(madsim)` 下，由 `ctrl_c` 驱动的节点关闭可以通过测试代码经由
`madsim::runtime::Handle::send_ctrl_c` 注入。适配器二进制文件的入口点
仍然直接调用 `tokio::signal::ctrl_c`，仍处于范围之外。

### 日志运行在真实的 OS 线程上

日志子系统通过 `std::thread::Builder` 生成一个写入器线程，并使用
`std::sync::mpsc`。在仿真下，该线程不会被生成，日志事件会被丢弃。
日志输出在确定性契约之外：该写入器只写入，从不读取或修改仿真状态。

### 适配器

适配器 crate 不在初始 DST 契约的范围内。每个适配器都有自己的一套
`chrono::Utc::now`、`SystemTime::now`、`Uuid::new_v4` 以及传输层调用点。
一个要进入 DST 路径的适配器，必须先针对直接的时钟、RNG 和传输使用
进行审计，其行为才能被该契约覆盖。

### 进程级全局惰性状态在首次调用时会消耗 RNG 字节

该契约在单次运行时运行内成立。有几个进程全局的惰性初始化会在首次
调用时消耗 RNG 字节，这对于在同一进程内运行两次带种子的执行并比较
其轨迹的测试装置来说很重要。

- `Ustr::from()` 内部化器在首次使用时分配并为其内部映射播种。
- `ahash::RandomState` 在首次创建实例时通过 `getrandom`（在
  `cfg(madsim)` 下被 `madsim` 拦截）为自身播种。

单运行时测试（每个种子调用一次测试主体、全新进程）不受影响：
该消耗是带种子执行的一部分，能够确定性地复现。

在同一进程内两次调用测试主体以对比轨迹的同种子相等性测试框架，
会看到不同运行之间的偏移。第一次运行需要支付惰性初始化的成本，
消耗 RNG 字节；第二次运行继承了预热后的状态，从 RNG 序列的不同
偏移量开始。

变通方法：在比较之前，在运行时之外预热进程全局状态，例如在进程
启动时调用一次 `Ustr::from("")` 并构造一个 `ahash::RandomState`。
这样两次运行都会以预热后的状态开始，并以相同的方式消耗 RNG 序列。

### 适配器工厂不再暴露 `Rc<RefCell<Cache>>`

提交 `f0ea66da15`（“通过 `CacheView` 标准化适配器缓存访问”）将
`DataClientFactory::create` 和 `ExecutionClientFactory::create` 上的可变
`Rc<RefCell<Cache>>` 参数替换为一个 `CacheView`。`CacheView` 暴露了
`borrow()` 用于读取访问，但不暴露内部的 `Rc` 句柄。

这阻碍了需要在工厂内部内联构造 `OrderMatchingEngine` 的 DST 风格
测试装置工厂，因为 `OrderMatchingEngine::new` 仍然接受
`Rc<RefCell<Cache>>`，而没有公共访问器可以从 `CacheView` 恢复该句柄。

变通方法：

- 将测试装置消费者固定在 `f0ea66da15` 之前的提交上。
- 重构测试装置，使其在工厂之外（内核 `Cache` 句柄仍然可以到达的地方）
  构建 `OrderMatchingEngine`，并将构造好的引擎传入客户端。

一个更长期的解决方案，要么是在 `CacheView` 上添加一个用于访问内部句柄
的访问器，要么是提供一个接受 `CacheView` 的替代 `OrderMatchingEngine`
构造函数。这两者目前都不在代码树中。

## 与其他测试层的关系

DST 是对现有测试的补充；它不会取代其中的任何一层。

| 层级                   | 覆盖范围                                               | 与 DST 的关系                                |
|-------------------------|------------------------------------------------------|-------------------------------------------------|
| 单元测试              | 纯逻辑、计算、解析器、转换器。     | 不变。                                      |
| 集成测试       | 组件交互、I/O 边界。               | 不变。DST 与之并行运行，而不是取代它。 |
| 基于属性的测试    | 输入域上的不变式（解析器、往返转换）。 | 不变。                                      |
| 验收测试        | 端到端的回测和实盘场景。              | 不变。                                      |
| 确定性仿真（DST） | 异步时序、调度、恢复正确性。      | 增加了可通过种子重放的探索能力。               |

DST 的独特价值在于异步并发与状态机正确性的交集。诸如“关闭时的某条消息
在特定唤醒顺序下被丢弃”或“遍历顺序反转时某个核对事件丢失”之类的漏洞，
正是它所针对的目标类别。对于其他情况，现有的测试层才是正确的工具。

## 现状

截至本代码库的当前状态：

- 第 1 层（运行时替换）已实现。`nautilus_common::live::dst` 为
  `time`、`task`、`runtime` 和 `signal` 暴露了已路由的重新导出。
  `time`、`task` 和 `runtime` 的生产环境调用点通过该接缝路由；
  信号调用点的采用情况是部分的（参见“范围边界”下的“信号处理”）。
- 第 2 层（非确定性替换）已在 16 个范围内的 crate 中实现。已经存在
  针对挂钟时间、单调时间、随机性和遍历顺序的接缝。审计的收尾工作和
  剩余的允许调用点已在“实现说明”中列举。
- 通过 `check-dst-conventions` 进行的静态强制执行在预提交和 CI 中处于
  激活状态。该钩子覆盖了具有支撑意义的条件；在有充分理由的情况下，
  `// dst-ok` 标记约定允许按行例外。
- 在 `cfg(madsim)` 下的构建和测试冒烟检查通过 `dst` 工作流
  （`.github/workflows/dst.yml`，调用 `make cargo-test-sim`）运行。
  它以 `--features simulation` 编译范围内的 crate，并运行当前所有
  兼容仿真的测试。消费 `nautilus-model` 类型的 crate
  （`nautilus-common`、`nautilus-execution`）还会以
  `--features "simulation,high-precision"` 运行第二条流程，
  以便在两种定点宽度（`QuantityRaw` / `PriceRaw` 分别为 `u64` 与
  `u128`）下都执行接缝路由的代码路径。
  - 所有兼容仿真的 `nautilus-common` 测试。这条流程在编译时传播了
    `nautilus-core/simulation`，因此套件中的每个测试都会选中显式的
    `wall_clock_now` cfg 分支。普通的 `#[rstest]` 测试运行在 madsim
    运行时之外，会通过该接缝的 `SystemTime::now()` 回退路径（这与
    madsim 的 libc 垫片在运行时之外所采用的路径相同）。`LiveClock`
    测试模块被 cfg 排除在外，因为它的普通 `#[rstest]` 用例会在没有
    madsim 运行时的情况下启动 `LiveTimer` 任务，而且大多数用例还会
    阻塞等待挂钟时间推进。该流程中的
    `live::dst::tests::test_dst_wall_clock_advances_with_virtual_time`
    测试使用 `#[madsim::test]`，并断言 `nanos_since_unix_epoch` 会随着
    `madsim::time::sleep` 而推进，因此虚拟挂钟行为在该公共流程上得到了
    端到端的验证。
  - `nautilus-live` 的启动核对超时回归测试。该测试运行在一个 madsim
    运行时下，验证一个待处理的整体状态请求会到达配置的超时时间，
    报告预期的错误，并清理该节点，而不是进入一个真实的 Tokio 定时器。
  - `nautilus-network` 的全部测试（受传输层约束的测试模块在源代码层面
    被排除）。包括针对 sleep / timeout 虚拟时间和限速器的接缝固定测试，
    以及在虚拟时间下测试退避计时的重试套件。
  - `nautilus-execution` 的全部测试。撮合引擎、成交模型和执行引擎的
    状态机在确定性调度器下、使用带种子的 RNG 运行。
  - `nautilus-core` 中的跨 crate 接缝固定测试（`wall_clock_now` 虚拟
    时间）。每条流程都以其自身 crate 的 `--features simulation` 运行，
    并在适用的情况下使用 `#[madsim::test]`，因此显式的 cfg 分支和
    虚拟时间都得到了验证。

  综合起来，这可以捕获受 cfg 控制的 DST 接缝中的偏移，并在确定性调度器下
  执行范围内的状态机；但目前尚未端到端地验证确定性本身。
- 端到端的运行时验证（对范围内某段代码路径进行同种子差异对比）
  不在本代码库的范围内。结构性条件（规则 1 到规则 7）已被强制执行；
  “某个种子能在多次运行之间重现相同的可观察行为”这一说法，从接缝设计
  来看是可信的，但目前尚未通过一个回归检查门加以验证。

## 延伸阅读

- `.pre-commit-hooks/check_dst_conventions.sh` 完整定义了七条强制执行
  规则，并记录了 `// dst-ok` 标记约定。
- 外部参考资料：[FoundationDB 测试理念](https://apple.github.io/foundationdb/testing.html)、
  [TigerBeetle 仿真测试博客文章](https://tigerbeetle.com/blog/)，
  以及关于确定性运行时的 [madsim 代码库](https://github.com/madsim-rs/madsim)。
