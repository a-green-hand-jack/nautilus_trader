# Betfair v2

Betfair 的 Rust 适配器正在积极进行功能对齐工作。本页跟踪当前 Rust 的行为,以及从
[Betfair](betfair.md) 稳定指南切换过来的计划。

本页镜像了 [Betfair](betfair.md) 主要章节的顺序。当 Rust 适配器成为主要的 Betfair
路径后,这个文件可以经过小幅编辑替换 `betfair.md`,而不必完全重写。

## 范围

- 本页的信息来源:`crates/adapters/betfair`
- 当前的稳定指南:[Betfair](betfair.md)
- 本页目的:跟踪当前的 Rust 实现范围、已知差距,以及切换路径

## 当前 Rust 状态

| 领域                     | 当前 Rust 行为                                                                                        | 与今日 `betfair.md` 的差异                                        | 切换工作                                        |
|--------------------------|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|-----------------------------------------------------|
| 订单类型              | `MARKET` 仅支持 `AT_THE_CLOSE`;`LIMIT` 支持收盘时的 BSP 流程。                                  | 稳定指南在这方面仍是 Python 视角。                         | 决定最终的 Betfair 市价单模型。            |
| 批量操作         | 已实现 `SubmitOrderList` 与 `BatchCancelOrders`。                                                   | 稳定指南曾将这些标记为不支持。                       | 保留并推广。                                   |
| 对账范围     | `reconcile_market_ids_only` 使用 `reconcile_market_ids`;否则回退到 `stream_market_ids_filter`。 | 稳定指南将流过滤与对账视为相互独立。       | 决定 Rust 是否保留或移除这一耦合。      |
| 全量镜像缓存检查  | Rust 在启动时以及每次流重连时使用 `generate_mass_status()`;没有 `check_cache_against_order_image`。 | 稳定指南描述的是 Python 的全量镜像缓存检查。                 | 增加对齐支持,或将 Rust 路径记录为最终方案。 |
| 重连后暂停      | 在对账进行期间,`submit_order` 和 `submit_order_list` 会发出带 `STREAM_RECONCILING` 原因的 `OrderDenied`。 | Python 在重连期间会继续交易。                                    | 当 `betfair.md` 切换后,将其推广为 Rust 默认行为。 |
| 外部订单过滤 | `ignore_external_orders` 仅跳过没有 `rfo` 的 OCM 更新。                                               | Python 在全量镜像缓存检查期间也会用到它。                       | 决定最终的过滤行为。                    |
| 配置项              | 没有 `certs_dir`、没有 `instrument_config`,keep alive 固定,心跳值为必填。                          | 稳定指南仍在文档化 Python 的配置项。                   | 决定是增加对齐支持,还是直接采用 Rust 的方案。 |
| SSL 证书         | 流客户端目前将 `certs_dir=None` 硬编码。                                                                  | 稳定指南文档化了证书配置与 `BETFAIR_CERTS_DIR`。 | 增加支持,或从未来指南中移除。        |

## 订单能力

### 订单类型

| 订单类型             | 是否支持 | 备注                                                                       |
|------------------------|-----------|-----------------------------------------------------------------------------|
| `MARKET`               | ✓*        | Rust 仅支持 `AT_THE_CLOSE`,映射到 Betfair 的 `MARKET_ON_CLOSE`。 |
| `LIMIT`                | ✓         | Rust 支持普通限价单以及收盘时的 BSP 限价单。           |
| `STOP_MARKET`          | -         | 不支持。                                                              |
| `STOP_LIMIT`           | -         | 不支持。                                                              |
| `MARKET_IF_TOUCHED`    | -         | 不支持。                                                              |
| `LIMIT_IF_TOUCHED`     | -         | 不支持。                                                              |
| `TRAILING_STOP_MARKET` | -         | 不支持。                                                              |

### 有效期(Time in force)

| 有效期  | 是否支持 | 备注                                                        |
|----------------|-----------|--------------------------------------------------------------|
| `GTC`          | ✓         | 映射到 Betfair `PERSIST`。                                   |
| `DAY`          | ✓         | 映射到 Betfair `LAPSE`。                                     |
| `FOK`          | ✓         | 映射到 Betfair `FILL_OR_KILL`。                              |
| `IOC`          | ✓         | 映射到 `min_fill_size=0` 的 `FILL_OR_KILL`。               |
| `AT_THE_CLOSE` | ✓         | 用于 Betfair BSP 的 `LIMIT_ON_CLOSE` 与 `MARKET_ON_CLOSE`。 |

Rust 目前也接受处于 `AT_THE_OPEN` 模式下的 `LIMIT` 订单,并将其通过 Betfair
`LIMIT_ON_CLOSE` 指令路由。请将其视为当前行为,而非已定型的公开契约。

### 批量操作

| 操作    | 是否支持 | 备注                                      |
|--------------|-----------|--------------------------------------------|
| 批量提交 | ✓         | 通过 `SubmitOrderList` 实现。     |
| 批量修改 | -         | 不支持。                             |
| 批量取消 | ✓         | 通过 `BatchCancelOrders` 实现。   |

## 执行控制流

启动流程:

1. 连接 HTTP 客户端并获取初始账户资金。
2. 从缓存的订单中初始化 OCM 状态。
3. 连接 Betfair 执行流并订阅订单更新。
4. 通过 `listCurrentOrders` 生成启动时的完整状态(mass status)。
5. 将订单与成交报告对账进执行引擎。

每次流重连时,都会在最近的时间窗口内运行同样的完整状态对账,期间适配器会
暂停会增加敞口的新指令,直到对账分发完成。参见
[重连后对账](#重连后对账)。

当前 Rust 备注:

- `stream_market_ids_filter` 过滤实时的 OCM 更新。
- `reconcile_market_ids_only=True` 使用显式的 `reconcile_market_ids`。
- 当 `reconcile_market_ids_only=False` 且未设置 `reconcile_market_ids` 时,Rust 目前会
  回退使用 `stream_market_ids_filter` 进行启动时对账。
- Rust 尚未实现 Python 的 `check_cache_against_order_image` 全量镜像缓存检查。
- `ignore_external_orders=True` 目前仅跳过没有 `rfo` 的 OCM 更新。

## 会话管理与重连

Betfair 会话每 12-24 小时过期一次。Rust 适配器通过三种机制自动处理会话恢复:

| 机制           | 触发条件                           | 动作                                                               |
|---------------------|-----------------------------------|----------------------------------------------------------------------|
| 定期保活 | 每 10 小时一次。                   | 刷新会话令牌,推送给所有流监听通道。              |
| 保活回退 | 保活返回 `LoginFailed`。 | 通过 `reconnect()` 完全重新登录,将新令牌推送给各个流。        |
| 流重连    | 断开后收到 `Connection` 消息。  | 先尝试保活,若 `LoginFailed` 则回退为重新登录,更新鉴权。 |

保活期间的瞬时错误(网络超时、5xx 响应)会被记录并跳过。原有的会话令牌
会被保留,下一次保活间隔会重试。只有 `LoginFailed` 错误(会话过期)才会
触发完全重新登录。

数据客户端与执行客户端运行相同的重连逻辑。每个客户端都会启动:

- 一个 **保活任务**,定期刷新会话,并将更新后的鉴权字节推送给流监听通道。
- 一个 **重连处理器**,在流重连后监听 `Connection` 消息,刷新会话,并推送新令牌。

流客户端将鉴权字节保存在 `tokio::sync::watch` 通道中。`post_reconnection`
闭包会在每次 TCP 重连时从该通道读取,因此无论是保活任务还是重连处理器
刷新的令牌,都会在下一次连接尝试时被读取。

数据客户端的重连处理器在赛事流(race stream)处于活跃状态时,也会更新
赛事流的鉴权信息。

## 重连后对账

当 Betfair 执行流重连时,适配器会假定缓存在这段间隙期间可能与场所状态出现
偏差(尤其是成交可能在重连流镜像到达之前就已完成并从未匹配簿中滚落)。因此
在允许策略增加新敞口之前,它会在最近的时间窗口内运行一次完整状态对账。

| 步骤 | 触发条件                                        | 动作                                                                                                          |
|------|------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| 1    | 流断开后收到的第二条 `Connection` 消息。 | OCM 处理器置位 `pending_resync` 和 `is_reconciling`,向后台任务发送重连信号。      |
| 2    | 重连任务收到信号。                   | 重新置位 `is_reconciling`,确保排队中的第二次重连在其自身迭代期间也会暂停。                    |
| 3    | 重连任务主体。                   | 刷新会话、更新流鉴权、调用 `getAccountFunds`,并调用 `listCurrentOrders` 获取订单与成交。 |
| 4    | 完整状态构建完成。                     | 以 `ExecutionReport::MassStatus` 分发,供引擎对账进缓存。                            |
| 5    | 迭代结束。                                | 清除 `is_reconciling`。若迭代失败也会清除(fail-open,与 Nautilus 的其余部分保持一致)。  |

在 `is_reconciling` 置位期间:

- `submit_order` 与 `submit_order_list` 会发出原因为
  `STREAM_RECONCILING: post-reconnect reconciliation in progress, retry once it completes`
  的 `OrderDenied`。
- `cancel_order`、`batch_cancel_orders` 与 `modify_order` 会不受影响地正常通过,
  使策略始终可以在这一窗口期间减少敞口。
- 间隙期内到达的缓冲 OCM 会在下一次策略指令时,通过
  `process_pending_resync`(独立的 `pending_resync` 标志)被清空处理。

如果客户端在对账仍在进行中时断开连接,`clear_resync_state` 会清除
`is_reconciling`,使随后的连接/提交周期能够干净地重新开始。

完整状态获取的回溯窗口为 `stream_gap_recovery_lookback_mins`
(默认 `10`)。该值应充分覆盖预期的最长重连时长,以确保在间隙期中途
完成的成交仍能被捕获。

## 报价机制与定价

Betfair 使用分层报价机制,不同价格区间对应不同的最小变动单位:

| 价格区间    | 最小变动单位 |
|----------------|-----------|
| 1.01 - 2.00    | 0.01      |
| 2.00 - 3.00    | 0.02      |
| 3.00 - 4.00    | 0.05      |
| 4.00 - 6.00    | 0.10      |
| 6.00 - 10.00   | 0.20      |
| 10.00 - 20.00  | 0.50      |
| 20.00 - 30.00  | 1.00      |
| 30.00 - 50.00  | 2.00      |
| 50.00 - 100.00 | 5.00      |
| 100.00 - 1000  | 10.00     |

最小价格为 1.01,最大价格为 1000.00。

## 订单修改

- 价格与数量不能原子性地同时变更;这需要分别操作。
- 价格修改使用 `ReplaceOrders`(取消 + 以新价格提交新订单)。
- 数量减少使用带 `size_reduction` 参数的 `CancelOrders`。
- 不支持增加数量;需提交新订单代替。

一次替换操作会同时为原订单生成一个取消事件,并为替换订单生成一个已接受事件。
适配器会跟踪待处理的替换,以抑制合成的取消事件。

## 订单流成交处理

执行客户端处理来自 Betfair Exchange Streaming API 的订单更新。两个配置项
控制更新的过滤方式:

- `stream_market_ids_filter`:在市场层面过滤(提前退出,静默跳过)。
- `ignore_external_orders`:在订单层面过滤(跳过没有 `rfo` 的 OCM 更新)。

### 成交处理

适配器在处理来自流的成交时,处理了若干边缘情况:

- **增量成交**:Betfair 报告的是累计已匹配数量。适配器通过跟踪每个订单
  上次已知的已成交数量来计算增量成交。
- **累计撤销(void)**:Betfair 的 `sv` 是累计值。适配器将增量按“最新优先”的
  顺序分配给已知的、已在本地应用的成交批次,并发出累计的 `OrderFillVoided` 修正。
- **重连安全**:首次看到的快照会为其累计撤销状态设定初值,而不会逆转
  Nautilus 从未应用过的敞口。
- **终态处置**:Betfair 的撤销订单仍保持 `EXECUTION_COMPLETE`,将订单映射为
  `VOIDED`,且永远不会设置 `is_reopened`。
- **超额成交保护**:会拒绝会导致超过订单数量的成交。
- **竞态条件**:当流成交在 HTTP 订单响应之前到达时,适配器会立即缓存场所
  订单 ID,以确保订单能够正确匹配。
- **网络错误恢复**:当 HTTP 订单提交因网络错误(超时、连接重置)失败时,
  订单可能实际上仍已在场所下达。适配器会将订单保持在 SUBMITTED 状态,
  并保留客户订单引用,以便流在重连时能够确认该订单。API 明确拒绝的错误
  (Betfair 明确拒绝)会立即拒绝。
- **间隙窗口成交**:在流断开期间完成并从未匹配簿滚落的成交,会通过
  重连后的完整状态对账恢复;参见 [重连后对账](#重连后对账)。

## 速率限制

适配器使用独立的速率限制桶,使账户状态轮询与对账不会限制订单下达:

| 桶  | 默认值 | 端点                                       |
|---------|---------|-------------------------------------------------|
| 通用 | 5/s     | 账户状态、对账、保活。      |
| 订单  | 20/s    | `placeOrders`、`replaceOrders`、`cancelOrders`。 |

订单状态与成交报告查询在会话错误后刷新会话并重试一次。`TOO_MANY_REQUESTS`
错误会在 5 秒延迟后重试。

## 市场版本价格保护

当 `use_market_version=True` 时,每个订单请求都会携带适配器最后看到的市场
版本号。如果在 Betfair 处理该订单时市场已推进超过该版本,Betfair 会让该
投注失效(lapse),而不会将其与已变化的订单簿撮合。

适配器从 instrument 的 `info` 字典中读取市场版本,该字典由 Exchange
Streaming API 的 `MarketDefinition` 更新填充。在收到首个 `MarketDefinition`
之前提交的订单不包含版本信息。

## 自定义数据类型

Rust 适配器通过市场流和赛事流发出与 Python 适配器相同的自定义数据类型。
所有自定义数据在订阅市场后都会自动流转。

| 类型                       | 流 | 说明                                       |
|----------------------------|--------|---------------------------------------------------|
| `BetfairTicker`            | 市场 | 最新成交价、成交量、BSP 指标。 |
| `BetfairStartingPrice`     | 市场 | 收盘后已实现的 BSP。                  |
| `BetfairSequenceCompleted` | 市场 | 标记一个市场变更序列的结束。            |
| `BetfairOrderVoided`       | 订单  | 已撤销订单的详情(撤销数量、价格、方向)。  |
| `BetfairRaceRunnerData`    | 赛事   | 每个参赛者的实时 GPS 跟踪数据(TPD)。               |
| `BetfairRaceProgress`      | 赛事   | 分段用时、跑位顺序、起跳数据。        |

赛事数据需要 Total Performance Data(TPD)覆盖以及具备 TPD 权限的 Betfair
API key。通过 `subscribe_race_data=True` 启用。

## 多节点部署

当多个交易节点共享同一个 Betfair 账户、处理不同市场时:

1. 将 `stream_market_ids_filter` 设置为仅包含该节点的市场。
2. 将 `ignore_external_orders=True` 设置为抑制关于其他节点订单的警告。
3. 将 `reconcile_market_ids_only=True` 设置为限制对账范围。

## 当前 Rust 配置

### 数据客户端配置

| 选项                              | 默认值  | 备注                                         |
|-------------------------------------|----------|-----------------------------------------------|
| `account_currency`                  | 必填 | Betfair 账户货币。                     |
| `username`                          | `None`   | 回退到 `BETFAIR_USERNAME`。             |
| `password`                          | `None`   | 回退到 `BETFAIR_PASSWORD`。             |
| `app_key`                           | `None`   | 回退到 `BETFAIR_APP_KEY`。              |
| `proxy_url`                         | `None`   | HTTP 请求可选的代理 URL。         |
| `request_rate_per_second`           | `5`      | 通用 HTTP 速率限制。                      |
| `default_min_notional`              | `None`   | 可选的最小名义金额覆盖值。                   |
| `event_type_ids`                    | `None`   | 可选的导航过滤器。                   |
| `event_type_names`                  | `None`   | 可选的导航过滤器。                   |
| `event_ids`                         | `None`   | 可选的导航过滤器。                   |
| `country_codes`                     | `None`   | 可选的导航过滤器。                   |
| `market_types`                      | `None`   | 可选的导航过滤器。                   |
| `market_ids`                        | `None`   | 可选的导航过滤器。                   |
| `min_market_start_time`             | `None`   | 可选的导航过滤器。                   |
| `max_market_start_time`             | `None`   | 可选的导航过滤器。                   |
| `stream_host`                       | `None`   | 可选的流主机覆盖值。                |
| `stream_port`                       | `None`   | 可选的流端口覆盖值。                |
| `stream_heartbeat_ms`               | `5,000`  | Rust 中目前为必填。                       |
| `stream_idle_timeout_ms`            | `60,000` | 重连前的空闲超时时间。                |
| `stream_reconnect_delay_initial_ms` | `2,000`  | 初始重连延迟。                      |
| `stream_reconnect_delay_max_ms`     | `30,000` | 最大重连延迟。                      |
| `stream_use_tls`                    | `True`   | 流连接使用 TLS。            |
| `stream_conflate_ms`                | `None`   | 显式的合流(conflation)设置。                  |
| `subscription_delay_secs`           | `3`      | 首次市场订阅前的延迟。   |
| `subscribe_race_data`               | `False`  | 订阅 RCM 更新。                     |

Rust 尚未暴露 `certs_dir` 或 `instrument_config`。Rust 还使用固定的
36,000 秒保活间隔。

### 执行客户端配置

| 选项                              | 默认值       | 备注                                                  |
|-------------------------------------|---------------|--------------------------------------------------------|
| `trader_id`                         | `TRADER-001`  | 客户端核心使用的 Trader ID。                         |
| `account_id`                        | `BETFAIR-001` | 客户端核心使用的账户 ID。                         |
| `account_currency`                  | `GBP`         | Betfair 账户货币。                              |
| `username`                          | `None`        | 回退到 `BETFAIR_USERNAME`。                      |
| `password`                          | `None`        | 回退到 `BETFAIR_PASSWORD`。                      |
| `app_key`                           | `None`        | 回退到 `BETFAIR_APP_KEY`。                       |
| `proxy_url`                         | `None`        | HTTP 请求可选的代理 URL。                  |
| `request_rate_per_second`           | `5`           | 通用 HTTP 速率限制。                               |
| `order_request_rate_per_second`     | `20`          | 订单端点速率限制。                             |
| `stream_host`                       | `None`        | 可选的流主机覆盖值。                         |
| `stream_port`                       | `None`        | 可选的流端口覆盖值。                         |
| `stream_heartbeat_ms`               | `5,000`       | Rust 中目前为必填。                                |
| `stream_idle_timeout_ms`            | `60,000`      | 重连前的空闲超时时间。                         |
| `stream_reconnect_delay_initial_ms` | `2,000`       | 初始重连延迟。                               |
| `stream_reconnect_delay_max_ms`     | `30,000`      | 最大重连延迟。                               |
| `stream_use_tls`                    | `True`        | 流连接使用 TLS。                     |
| `stream_market_ids_filter`          | `None`        | 可选的实时 OCM 市场过滤器。                       |
| `ignore_external_orders`            | `False`       | 仅跳过没有 `rfo` 的 OCM 更新。                  |
| `calculate_account_state`           | `True`        | 目前在 Rust 中用于控制定期账户状态轮询。    |
| `request_account_state_secs`        | `300`         | 账户资金轮询间隔。                    |
| `reconcile_market_ids_only`         | `False`       | 当为 `True` 时,使用 `reconcile_market_ids`。               |
| `reconcile_market_ids`              | `None`        | 显式指定启动时对账的市场 ID。            |
| `use_market_version`                | `False`       | 在下单和替换请求中附带市场版本号。   |
| `stream_gap_recovery_lookback_mins` | `10`          | 重连后完整状态对账的回溯窗口。 |

Rust 尚未暴露 `certs_dir` 或 `instrument_config`。

## 切换计划

在 Rust 适配器成为主要 Betfair 路径之前,将本页作为过渡跟踪文档。

切换时:

1. 决定 Rust 是保留当前的对账过滤行为,还是与 Python 的拆分方式对齐。
2. 决定 Rust 是否增加证书配置以及其他 Python 配置字段。
3. 决定 Rust 是保留仅支持 BSP 的 `MARKET` 订单,还是增加 Python 的激进限价路径。
4. 将本文件提升为 `betfair.md`。
5. 将任何剩余的 Python 专属说明移入一个简短的历史遗留说明或发行说明中。
