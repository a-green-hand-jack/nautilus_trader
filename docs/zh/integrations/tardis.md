# Tardis

Tardis 为加密货币市场提供细粒度数据,包括逐笔订单簿快照与更新、成交、
未平仓合约、资金费率、期权摘要,以及主流加密交易所的强平数据。

NautilusTrader 集成了 Tardis API、Tardis Machine WebSocket 服务器以及
Tardis CSV 格式。该适配器的能力包括:

- `TardisCSVDataLoader`:将 Tardis 格式的 CSV 文件读取为 Nautilus
  数据,提供批量和内存高效的流式两种路径。
- `TardisMachineClient`:从 Tardis Machine 流式获取实时或历史回放数据,
  并将消息转换为 Nautilus 数据。
- `TardisHttpClient`:从 Tardis HTTP API 请求标的元数据,并将其解析为
  Nautilus 标的定义。
- `TardisDataClient`:为 Tardis Machine 数据流提供实时数据客户端。
- `TardisInstrumentProvider`:从 Tardis 元数据 API 加载标的定义。
- **数据管道函数**:从 Tardis Machine 回放历史数据,并写入 Nautilus
  Parquet catalog 文件。

:::info
Nautilus 标的元数据调用需要 `TARDIS_API_KEY`。Tardis Machine 对于
每月免费首日之外的历史日期,使用 `TM_API_KEY`。另见
[环境变量](#环境变量)。
:::

## 概览

该适配器以 Rust 实现,并提供可选的 Python 绑定。它不需要任何外部
Tardis 客户端库依赖。

:::info
`tardis` **不需要**任何额外的安装步骤。该适配器的核心组件被编译为
静态库,并在构建期间链接。
:::

## Tardis 文档

Tardis 提供了详尽的用户[文档](https://docs.tardis.dev/)。建议将本
NautilusTrader 集成指南与 Tardis 官方文档结合参考。

## 支持的格式

Tardis 提供*标准化*的市场数据,这是一种在所有支持的交易所之间保持一致
的统一格式。这种标准化使单一解析器能够处理来自任何
[Tardis 支持交易所](#场所)的数据。该适配器不支持交易所原生的 Tardis
市场数据格式。

NautilusTrader 支持以下标准化的 Tardis Machine 格式。字段 schema 请
参见官方的 [Tardis 数据类型参考](https://docs.tardis.dev/tardis-machine/data-types)。

| Tardis 格式       | Nautilus 数据类型                                                |
|:--------------------|:------------------------------------------------------------------|
| `book_change`       | `OrderBookDelta`                                                  |
| `book_snapshot_*`   | `OrderBookDepth10` 或 `OrderBookDeltas`                           |
| `quote`             | `QuoteTick`                                                       |
| `quote_10s`         | `QuoteTick`                                                       |
| `trade`             | `Trade`                                                           |
| `trade_bar_*`       | `Bar`                                                             |
| `instrument`        | `CurrencyPair`、`CryptoFuture`、`CryptoPerpetual`、`CryptoOption` |
| `derivative_ticker` | `FundingRateUpdate`                                               |
| `option_summary`    | `OptionGreeks`;可选地从 BBO 字段生成 `QuoteTick`              |
| `disconnect`        | *不适用*                                                  |

**备注:**

- Tardis 文档将 `quote` 记录为 `book_snapshot_1_0ms` 的别名。
- Tardis 文档将 `quote_10s` 记录为 `book_snapshot_1_10s` 的别名。
- `quote`、`quote_10s` 以及单档快照都会被解析为 `QuoteTick`。
- 当值发生变化时,Rust 数据客户端还会从 `derivative_ticker` 消息中
  发出标记价格和指数价格更新。
- Tardis 的 `option_summary` 消息包含最优买卖价字段。Nautilus 始终将
  该数据流映射为 `OptionGreeks`;将 `extract_bbo_as_quotes` 设为
  `true` 可同时从这些 BBO 字段发出 `QuoteTick`。

:::info
另请参见 Tardis 的
[Tardis Machine 快速入门](https://docs.tardis.dev/tardis-machine/quickstart)。
:::

## K 线

适配器将 Tardis 的成交 K 线周期和后缀转换为 Nautilus 的 `BarType`,
包括以下内容:

| Tardis 后缀 | 含义         | Nautilus K 线聚合 |
|:--------------|:----------------|:-------------------------|
| `ms`          | 毫秒    | `MILLISECOND`            |
| `s`           | 秒         | `SECOND`                 |
| `m`           | 分钟         | `MINUTE`                 |
| `ticks`       | tick 数量 | `TICK`                   |
| `vol`         | 成交量大小     | `VOLUME`                 |

## 符号规则与规范化

Tardis 集成通过一致地规范化符号,确保与 NautilusTrader 各加密货币交易所
适配器的兼容性。通常情况下,NautilusTrader 使用 Tardis 提供的原生交易所
命名约定。对于某些交易所,原始符号会按以下方式调整,以符合 Nautilus
符号规范化规则:

### 通用规则

- 所有符号都转换为大写。
- 对某些交易所,市场类型后缀会以连字符附加。
- 原始交易所符号保留在 Nautilus 标的定义的 `raw_symbol` 字段中。

### 交易所特定的规范化

- **Binance**:Nautilus 会为所有永续符号附加后缀 `-PERP`。
- **Bybit**:Nautilus 使用产品类别后缀,包括 `-SPOT`、`-LINEAR`、
  `-INVERSE` 和 `-OPTION`。
- **dYdX**:Nautilus 会为所有永续符号附加后缀 `-PERP`。
- **Gate.io**:Nautilus 会为所有永续符号附加后缀 `-PERP`。

各交易所的详细符号文档:

- [Binance 符号规则](./binance.md#符号规则)
- [Bybit 符号规则](./bybit.md#符号规则)
- [dYdX 符号规则](./dydx.md#符号规则)

## 场所

Tardis 上的部分交易所被划分为多个场所。下表列出了 Nautilus 场所与对应
Tardis 交易所之间的映射关系:

| Nautilus 场所          | Tardis 交易所                                    |
|:------------------------|:------------------------------------------------------|
| `ASCENDEX`              | `ascendex`                                            |
| `BINANCE`               | `binance`、`binance-dex`、`binance-futures`、`binance-options` |
| `BINANCE_DELIVERY`      | `binance-delivery`(*币本位合约*)        |
| `BINANCE_US`            | `binance-us`                                          |
| `BITFINEX`              | `bitfinex`、`bitfinex-derivatives`                    |
| `BITFLYER`              | `bitflyer`                                            |
| `BITGET`                | `bitget`、`bitget-futures`                            |
| `BITMEX`                | `bitmex`                                              |
| `BITNOMIAL`             | `bitnomial`                                           |
| `BITSTAMP`              | `bitstamp`                                            |
| `BLOCKCHAIN_COM`        | `blockchain-com`                                      |
| `BYBIT`                 | `bybit`、`bybit-options`、`bybit-spot`                |
| `COINBASE`              | `coinbase`                                            |
| `COINBASE_INTX`         | `coinbase-international`                              |
| `COINFLEX`              | `coinflex`(*用于历史研究*)                |
| `CRYPTO_COM`            | `crypto-com`                                          |
| `CRYPTOFACILITIES`      | `cryptofacilities`                                    |
| `DELTA`                 | `delta`                                               |
| `DERIBIT`               | `deribit`                                             |
| `DYDX`                  | `dydx`                                                |
| `DYDX_V4`               | `dydx-v4`                                             |
| `FTX`                   | `ftx`、`ftx-us`(*历史研究*)               |
| `GATE_IO`               | `gate-io`、`gate-io-futures`                          |
| `GEMINI`                | `gemini`                                              |
| `HITBTC`                | `hitbtc`                                              |
| `HUOBI`                 | `huobi`、`huobi-dm`、`huobi-dm-linear-swap`、`huobi-dm-options` |
| `HUOBI_DELIVERY`        | `huobi-dm-swap`                                       |
| `HYPERLIQUID`           | `hyperliquid`                                         |
| `KRAKEN`                | `kraken`                                              |
| `KUCOIN`                | `kucoin`、`kucoin-futures`                            |
| `MANGO`                 | `mango`                                               |
| `OKCOIN`                | `okcoin`                                              |
| `OKEX`                  | `okex`、`okex-futures`、`okex-options`、`okex-spreads`、`okex-swap` |
| `PHEMEX`                | `phemex`                                              |
| `POLONIEX`              | `poloniex`                                            |
| `SERUM`                 | `serum`(*历史研究*)                       |
| `STAR_ATLAS`            | `star-atlas`                                          |
| `UPBIT`                 | `upbit`                                               |
| `WOO_X`                 | `woo-x`                                               |

Tardis 还提供了历史遗留的 Binance 交易所,如
`binance-european-options` 和 `binance-jersey`。

## 环境变量

Tardis 和 NautilusTrader 使用以下环境变量。

- `TM_API_KEY`:Tardis Machine 的 API key。
- `TARDIS_API_KEY`:NautilusTrader Tardis 客户端的 API key。
- `TARDIS_MACHINE_WS_URL`(可选):`TardisMachineClient` 的 WebSocket
  URL。
- `TARDIS_BASE_URL`(可选):NautilusTrader 中 `TardisHttpClient` 的
  基础 URL。
- `NAUTILUS_PATH`(可选):包含用于回放输出的 `catalog/` 子目录的
  父目录。

Tardis 标的元数据 API 需要 bearer token 授权,面向活跃的 pro 和
business 订阅用户开放。

## 运行 Tardis Machine 历史回放

[Tardis Machine 服务器](https://docs.tardis.dev/tardis-machine/quickstart)
是一个可在本地运行的服务器,内置数据缓存。它通过 HTTP 和 WebSocket API
提供逐笔历史数据以及整合后的实时加密货币市场数据。

你可以使用 Python 或 Rust,对历史数据执行完整的 Tardis Machine
WebSocket 回放,并以 Nautilus Parquet 格式输出结果。由于该功能以 Rust
实现,无论从 Python 还是 Rust 运行,性能都保持一致。

端到端的 `run_tardis_machine_replay` 数据管道函数使用指定的
[配置](#配置)执行以下步骤:

- 连接到 Tardis Machine 服务器。
- 从 Tardis 标的元数据 API 请求并解析所有必要的标的定义。
- 从 Tardis Machine 流式获取指定时间范围内所有请求的标的和数据类型。
- 针对每个标的、数据类型和日期(UTC),生成一个与 catalog 兼容的
  `.parquet` 文件。
- 断开与 Tardis Machine 服务器的连接,终止程序。

**文件命名约定**

文件按每天、每个标的写入一个文件,使用 ISO 8601 时间戳范围:

- **格式**:`{start_timestamp}_{end_timestamp}.parquet`
- **示例**:`2023-10-01T00-00-00-000000000Z_2023-10-01T23-59-59-999999999Z.parquet`
- **结构**:`data/{data_type}/{instrument_id}/{filename}`

该格式与 Nautilus 数据 catalog 的查询、合并和管理兼容。

:::note
每月的第一天数据可以在无需 Tardis Machine API key 的情况下请求。其他
日期则需要 `TM_API_KEY`。
:::

该流程针对直接输出到 Nautilus Parquet 数据 catalog 进行了优化。请将
`NAUTILUS_PATH` 设置为包含 `catalog/` 子目录的父目录。Parquet 文件会
按数据类型和标的写入 `<NAUTILUS_PATH>/catalog/data/` 下的子目录中。

如果未指定 `output_path` 且未设置 `NAUTILUS_PATH`,输出将默认写入
当前工作目录。

### 流程

首先,确保 `tardis-machine` docker 容器正在运行。使用以下命令:

```bash
docker run -p 8000:8000 -p 8001:8001 -e "TM_API_KEY=YOUR_API_KEY" -d tardisdev/tardis-machine
```

此命令启动的 `tardis-machine` 服务器没有持久化的本地缓存,这可能影响
性能。为获得更好的回放性能,请使用持久化卷运行它。

### 配置

接下来,确保准备好一个可用的配置 JSON 文件。

**配置 JSON 字段**

- `tardis_ws_url`(`str | null`):Tardis Machine WebSocket URL。默认为
  `TARDIS_MACHINE_WS_URL`。
- `normalize_symbols`(`bool | null`):应用 Nautilus 符号规范化。
  默认为 `true`。
- `output_path`(`str | null`):Parquet 数据的输出目录。默认为
  `NAUTILUS_PATH`,再默认为当前工作目录。
- `book_snapshot_output`(`"deltas" | "depth10" | null`):快照的输出
  格式。默认为 `"deltas"`。
- `extract_bbo_as_quotes`(`bool | null`):同时从 Tardis Machine
  `option_summary` 消息中的最优买卖价字段写入 `QuoteTick` 数据。
  默认为 `false`。
- `compression`(`"zstd" | "snappy" | "uncompressed" | null`):Parquet
  压缩编解码器。默认为 `"zstd"` 3 级。
- `proxy_url`(`str | null`):Tardis HTTP 请求的代理 URL。默认不使用
  代理。
- `options`(`JSON[]`):必需的回放请求选项对象。

`crates/adapters/tardis/bin/example_config.json` 提供了一个配置文件
示例:

```json
{
  "tardis_ws_url": "ws://localhost:8001",
  "output_path": null,
  "options": [
    {
      "exchange": "bitmex",
      "symbols": [
        "xbtusd",
        "ethusd"
      ],
      "data_types": [
        "trade"
      ],
      "from": "2019-10-01",
      "to": "2019-10-02"
    }
  ]
}
```

### 订单簿快照输出

`book_snapshot_output` 配置项控制 Tardis 的 `book_snapshot_*` 消息
如何转换和存储。

| 取值     | Nautilus 类型      | 输出目录     | 说明                           |
|:----------|:-------------------|:---------------------|:--------------------------------------|
| `deltas`  | `OrderBookDeltas`  | `order_book_deltas/` | 价格档位更新。                  |
| `depth10` | `OrderBookDepth10` | `order_book_depths/` | 最多 10 档的快照。 |

**何时使用每种格式:**

- **`deltas`(默认)**:当你需要重建订单簿状态,或将快照与
  `book_change` 数据结合使用时使用。每个价格档位都成为一条独立的
  增量记录。
- **`depth10`**:当策略需要周期性的深度快照时使用。每个快照是单独一条
  记录,超过 10 档的快照只保留前 10 档。

**避免文件被覆盖:**

当为同一标的和日期范围同时下载 `book_snapshot_*` 和 `book_change`
数据时,`depth10` 会将快照写入 `order_book_depths/`,避免覆盖
`order_book_deltas/`。

带显式格式的配置示例:

```json
{
  "tardis_ws_url": "ws://localhost:8001",
  "book_snapshot_output": "depth10",
  "options": [
    {
      "exchange": "binance-futures",
      "symbols": ["btcusdt"],
      "data_types": ["book_snapshot_5_100ms", "book_change"],
      "from": "2024-01-01",
      "to": "2024-01-02"
    }
  ]
}
```

### Option summary 的 BBO 提取

在请求 Tardis Machine 的 `option_summary` 数据、且回测同时需要期权
BBO 报价时,请将 `extract_bbo_as_quotes` 设为 `true`。Nautilus 仍会从
每条 `option_summary` 消息中写入 `OptionGreeks`。当所有最优买卖价字段
都存在且数量有效时,还会为同一标的和时间戳写入 `QuoteTick`。

此选项仅适用于 Tardis Machine 的 `option_summary` 回放和数据流消息。
它不会改变 Tardis CSV 的加载行为。

```json
{
  "tardis_ws_url": "ws://localhost:8001",
  "extract_bbo_as_quotes": true,
  "options": [
    {
      "exchange": "deribit",
      "symbols": ["BTC-28JUN24-70000-C"],
      "data_types": ["option_summary"],
      "from": "2024-01-01",
      "to": "2024-01-02"
    }
  ]
}
```

### Python 回放

要在 Python 中运行回放,请创建类似如下的脚本:

```python
import asyncio
from pathlib import Path

from nautilus_trader.core import nautilus_pyo3


async def run():
    config_filepath = Path("YOUR_CONFIG_FILEPATH")
    await nautilus_pyo3.run_tardis_machine_replay(str(config_filepath.resolve()))


if __name__ == "__main__":
    asyncio.run(run())
```

### Rust 回放

要在 Rust 中运行回放,请创建类似如下的二进制程序:

```rust
use std::path::PathBuf;

use nautilus_adapters::tardis::replay::run_tardis_machine_replay_from_config;

#[tokio::main]
async fn main() {
    nautilus_common::logging::ensure_logging_initialized();

    let config_filepath = PathBuf::from("YOUR_CONFIG_FILEPATH");
    run_tardis_machine_replay_from_config(&config_filepath).await;
}
```

日志默认级别为 INFO。要启用调试日志,请导出以下环境变量:

```bash
export NAUTILUS_LOG=debug
```

`crates/adapters/tardis/bin/example_replay.rs` 提供了一个可运行的
示例二进制程序。

也可以使用 cargo 运行:

```bash
cargo run --bin tardis-replay <path_to_your_config>
```

### 期权链回测 catalog

期权链回测在 Tardis 回放已将数据写入 Nautilus catalog 之后开始。回测
加载器在运行期间不会请求缺失的 Tardis 数据,因此 catalog 必须包含:

- 来自 Tardis 标的元数据 API 的期权标的。
- 来自单档期权订单簿快照、报价数据或 `option_summary` BBO 提取的
  `QuoteTick` 数据。
- 来自 Tardis `option_summary` 消息的 `OptionGreeks` 数据。

在 `BacktestDataConfig` 列表中,针对同一批期权标的 ID 同时使用
`QuoteTick` 和 `OptionGreeks`。期权链管理器会将回放的 BBO 和 Greeks
聚合为 `OptionChainSlice` 快照。使用 `snapshot_interval_ms=None` 进行
原始发布,或设置一个毫秒间隔以发布抽稀后的快照。

策略可以按虚实度(相对 ATM 或 ATM 百分比行权价范围)、按 delta(使用
`StrikeRange.delta(target, tolerance)`),或按固定行权价(使用
`StrikeRange.fixed([...])`)来选择合约。回测中的期权订单撮合是由报价
驱动的:可成交订单作为 taker 与对手方 BBO 撮合,而被动限价单可以在
后续 BBO 更新穿过限价时作为 maker 成交。

请在模拟场所上使用结构化的手续费模型(如 `CappedOptionFeeModel` 或
`TieredNotionalOptionFeeModel`)显式配置期权手续费。没有自动的 Tardis
交易所到手续费模型的映射。

### 期权链 CSV catalog 转换

对于来自可下载 Tardis CSV 文件的历史期权链,使用
`TardisCSVDataLoader.convert_options_chain_csv(...)` 将
`options_chain` 行转换为 Nautilus catalog 数据。此路径不会调用
Tardis Machine 或标的元数据 API,因此当你已经拥有 Tardis CSV 文件,
或希望从已下载的数据中启动一个无需 API key 的 catalog 时,这非常有用。

该转换器会为每一个已选行写入 `OptionGreeks`。在默认的
`extract_bbo_as_quotes=True` 设置下,完整的最优买卖价行也会写入
`QuoteTick`。对于期权链回测,请保持该选项启用:仅含 greeks 的
catalog 不提供报价,因此链管理器无法为没有 BBO 数据的行权价发布填充
完整的 `OptionChainSlice` 快照。

标的推导目前支持 Deribit 期权。对于其他期权场所,请在转换前设置
`write_instruments=False`,并在回测前通过其他来源加载标的。对非
Deribit 文件保留该选项启用,可能在数据文件已写入 catalog 之后失败。
请按时间顺序传入每日的 `options_chain` CSV 路径。`underlyings` 过滤器
匹配符号前缀,例如 `["BTC-"]`。设置 `snapshot_interval_ms` 可在每个
输入文件中,为每个标的在每个间隔内只保留最后一行;使用 `None` 则写入
每一个已选行。抽稀时,每个文件内的行必须按 `local_timestamp` 排序。

请在加载器上提供显式的 `price_precision` 和 `size_precision`,以获得
确定性的报价元数据。推断出的精度可能随着后续行的读取而增加,因此
文件较早写入的数据可能保留较低精度的元数据。

```python
from pathlib import Path

from nautilus_trader.adapters.tardis import TardisCSVDataLoader


loader = TardisCSVDataLoader(
    price_precision=4,
    size_precision=1,
)
loader.convert_options_chain_csv(
    filepaths=[Path("deribit_options_chain_2020-06-08.csv")],
    catalog_path=Path("catalog"),
    underlyings=["BTC-"],
    snapshot_interval_ms=60_000,
)
```

## 加载 Tardis CSV 数据

可以使用 Python 或 Rust 加载 Tardis 格式的 CSV 数据。加载器从磁盘读取
CSV 文本数据,并将其解析为 Nautilus 数据。由于加载器以 Rust 实现,
无论从 Python 还是 Rust 运行,性能都保持一致。

你还可以为 `load_*` 函数和方法指定 `limit` 参数,以控制加载的最大
行数。

:::note
加载混合标的的 CSV 文件由于精度要求而具有挑战性,不推荐使用。请改用
单标的的 CSV 文件。

`load_options_chain`、`stream_options_chain` 和
`convert_options_chain_csv` 方法是例外:Tardis 的 `options_chain`
文件本身就是混合标的的链文件,这些路径会按标的分别跟踪精度。不过
仍建议提供显式精度以获得确定性输出。
:::

### 在 Python 中加载 CSV 数据

可以使用 `TardisCSVDataLoader` 在 Python 中加载 Tardis 格式的 CSV
数据。加载数据时,可以选择性地指定标的 ID、价格精度和数量精度。提供
标的 ID 可以提高加载性能。如果省略,价格和数量精度会从 CSV 中推断
得出,但建议提供显式值以获得确定性输出,对于大文件尤其如此。

要加载数据,请创建类似如下的脚本:

```python
from pathlib import Path

from nautilus_trader.adapters.tardis import TardisCSVDataLoader
from nautilus_trader.model import InstrumentId


instrument_id = InstrumentId.from_str("BTC-PERPETUAL.DERIBIT")
loader = TardisCSVDataLoader(
    price_precision=1,
    size_precision=0,
    instrument_id=instrument_id,
)

filepath = Path("YOUR_CSV_DATA_PATH")
limit = None

deltas = loader.load_deltas(filepath, limit=limit)
```

### 在 Rust 中加载 CSV 数据

可以使用 `crates/adapters/tardis/src/csv/mod.rs` 中的加载函数,在
Rust 中加载 Tardis 格式的 CSV 数据。加载数据时,可以选择性地指定标的
ID、价格精度和数量精度。提供标的 ID 可以提高加载性能。如果省略,
价格和数量精度会从 CSV 中推断得出,但建议提供显式值以获得确定性输出。

完整示例请参见 `crates/adapters/tardis/bin/example_csv.rs`。

要加载数据,可以使用类似如下的代码:

```rust
use std::path::Path;

use nautilus_adapters::tardis;
use nautilus_model::identifiers::InstrumentId;

#[tokio::main]
async fn main() {
    // 可选地指定精度和 CSV 文件路径
    let price_precision = Some(1);
    let size_precision = Some(0);
    let filepath = Path::new("YOUR_CSV_DATA_PATH");

    // 可选地指定标的 ID 和/或 limit
    let instrument_id = InstrumentId::from("BTC-PERPETUAL.DERIBIT");
    let limit = None;

    // 视你的工作流,可考虑向上传播解析错误
    let _deltas = tardis::csv::load_deltas(
        filepath,
        price_precision,
        size_precision,
        Some(instrument_id),
        limit,
    )
    .unwrap();
}
```

## 流式加载 Tardis CSV 数据

为了对大型 CSV 文件进行内存高效处理,Tardis 集成可以按可配置的分块
加载和处理数据,而非一次性将整个文件加载进内存。这对于处理数 GB 大小
的 CSV 文件、同时不耗尽系统内存非常有用。

Python 的流式功能适用于以下高数据量的 CSV 类型:

- 订单簿增量(`stream_deltas`)。
- 报价 tick(`stream_quotes`)。
- 成交 tick(`stream_trades`)。
- 订单簿深度快照(`stream_depth10`)。
- 期权链行(`stream_options_chain`)。

Rust 还为这些 CSV 类型提供了流式函数,外加批量增量和资金费率。

### 在 Python 中流式加载 CSV 数据

`TardisCSVDataLoader` 提供了以迭代器形式产出数据分块的流式方法。每个
方法都接受一个 `chunk_size` 参数,控制每个分块读取的记录数量:

```python
from nautilus_trader.adapters.tardis import TardisCSVDataLoader
from nautilus_trader.model import InstrumentId

instrument_id = InstrumentId.from_str("BTC-PERPETUAL.DERIBIT")
loader = TardisCSVDataLoader(
    price_precision=1,
    size_precision=0,
    instrument_id=instrument_id,
)

filepath = Path("large_trades_file.csv")
chunk_size = 100_000  # 每个分块处理 100,000 条记录(默认值)

# 分块流式读取成交 tick
for chunk in loader.stream_trades(filepath, chunk_size):
    print(f"Processing chunk with {len(chunk)} trades")
    # 处理每个分块 —— 内存中只有当前这个分块
    for trade in chunk:
        # 在此处添加你的处理逻辑
        pass
```

### 流式加载订单簿数据

对于订单簿数据,增量和深度快照都可以流式读取:

```python
# 流式读取订单簿增量
for chunk in loader.stream_deltas(filepath):
    print(f"Processing {len(chunk)} deltas")
    # 处理增量分块

# 流式读取 depth10 快照(指定档位:5 或 25)
for chunk in loader.stream_depth10(filepath, levels=5):
    print(f"Processing {len(chunk)} depth snapshots")
    # 处理深度分块
```

### 流式加载报价数据

报价数据也可以类似地流式读取:

```python
# 流式读取报价 tick
for chunk in loader.stream_quotes(filepath):
    print(f"Processing {len(chunk)} quotes")
    # 处理报价分块
```

### 内存效率优势

流式方式提供了显著的内存效率优势:

- **可控的内存占用**:一次只有一个分块加载进内存。
- **可扩展的处理能力**:可以处理大于可用内存的文件。
- **可配置的分块大小**:根据你系统的内存和性能需求调整
  `chunk_size`(默认 100,000)。

:::warning
在使用流式加载配合精度推断时,推断出的精度可能与整体加载文件时有所
不同。精度推断在分块边界内进行,不同分块可能包含精度要求不同的数值。
若要获得确定性的精度行为,请提供显式的 `price_precision` 和
`size_precision` 参数。
:::

### 在 Rust 中流式加载 CSV 数据

底层的流式功能以 Rust 实现,可以直接使用:

```rust
use std::path::Path;

use nautilus_adapters::tardis::csv::stream_trades;
use nautilus_model::identifiers::InstrumentId;

#[tokio::main]
async fn main() {
    let filepath = Path::new("large_trades_file.csv");
    let chunk_size = 100_000;
    let price_precision = Some(1);
    let size_precision = Some(0);
    let instrument_id = Some(InstrumentId::from("BTC-PERPETUAL.DERIBIT"));

    // 分块流式读取成交
    let stream = stream_trades(
        filepath,
        chunk_size,
        price_precision,
        size_precision,
        instrument_id,
    ).unwrap();

    for chunk_result in stream {
        match chunk_result {
            Ok(chunk) => {
                println!("Processing chunk with {} trades", chunk.len());
                // 处理分块
            }
            Err(e) => {
                eprintln!("Error processing chunk: {}", e);
                break;
            }
        }
    }
}
```

## 请求标的定义

可以使用 `TardisHttpClient` 在 Python 和 Rust 中请求标的定义。该
客户端与
[Tardis 标的元数据 API](https://docs.tardis.dev/api/instruments-metadata-api)
交互,请求并将标的元数据解析为 Nautilus 标的。

`TardisHttpClient` 构造函数接受可选参数 `api_key`、`base_url`、
`timeout_secs`、`normalize_symbols` 和 `proxy_url`。

该客户端提供了获取特定 `instrument`,或某个交易所上所有
`instruments` 的方法。请使用 Tardis 的小写连字符交易所 ID,例如
`binance-futures`。

:::note
需要具备标的元数据 API 访问权限的 `TARDIS_API_KEY`。
:::

### 在 Python 中请求标的

要在 Python 中请求标的定义,请创建类似如下的脚本:

```python
import asyncio

from nautilus_trader.core import nautilus_pyo3


async def run():
    http_client = nautilus_pyo3.TardisHttpClient()

    instrument = await http_client.instrument("bitmex", "xbtusd")
    print(f"Received: {instrument}")

    instruments = await http_client.instruments("bitmex")
    print(f"Received: {len(instruments)} instruments")


if __name__ == "__main__":
    asyncio.run(run())
```

### 在 Rust 中请求标的

要在 Rust 中请求标的定义,可以使用类似如下的代码。完整示例请参见
`crates/adapters/tardis/bin/example_http.rs`。

```rust
use nautilus_tardis::{
    enums::TardisExchange,
    http::client::TardisHttpClient,
};

#[tokio::main]
async fn main() {
    nautilus_common::logging::ensure_logging_initialized();

    let client = TardisHttpClient::new(None, None, None, true, None).unwrap();

    // Tardis 标的定义
    let resp = client
        .instruments_info(TardisExchange::Bitmex, Some("XBTUSD"), None)
        .await;
    println!("Received: {resp:?}");

    // Nautilus 标的定义
    let resp = client
        .instruments(
            TardisExchange::Bitmex,
            Some("XBTUSD"),
            None,
            None,
            None,
            None,
            None,
            None,
        )
        .await;
    println!("Received: {resp:?}");
}
```

## 标的提供者

`TardisInstrumentProvider` 通过 HTTP 标的元数据 API 从 Tardis 请求并
解析标的定义。由于存在多个
[Tardis 支持的交易所](#场所),在加载全部标的时,必须使用
`InstrumentProviderConfig` 过滤出所需的场所:

```python
from nautilus_trader.config import InstrumentProviderConfig

# 参见支持的场所 https://nautilustrader.io/docs/nightly/integrations/tardis#venues
venues = {"BINANCE", "BYBIT"}
filters = {"venues": frozenset(venues)}
instrument_provider_config = InstrumentProviderConfig(load_all=True, filters=filters)
```

你也可以按常规方式加载特定的标的定义:

```python
from nautilus_trader.config import InstrumentProviderConfig

instrument_ids = [
    InstrumentId.from_str("BTCUSDT-PERP.BINANCE"),  # 使用 'binance-futures' 交易所
    InstrumentId.from_str("BTCUSDT.BINANCE"),  # 使用 'binance' 交易所
]
instrument_provider_config = InstrumentProviderConfig(load_ids=instrument_ids)
```

### 期权交易所过滤

当未提供 `instrument_type` 过滤器,或该过滤器不包含 `"option"` 时,
标的提供者会过滤掉期权专属的交易所,例如 `binance-options`、
`binance-european-options`、`bybit-options`、`okex-options` 和
`huobi-dm-options`。

要显式加载期权标的,请在 `instrument_type` 过滤器中包含 `"option"`:

```python
from nautilus_trader.config import InstrumentProviderConfig

venues = {"BINANCE", "BYBIT"}
filters = {
    "venues": frozenset(venues),
    "instrument_type": {"option"},  # 显式请求期权
}
instrument_provider_config = InstrumentProviderConfig(load_all=True, filters=filters)
```

这一过滤机制可以避免在不需要期权交易所时,对其发起不必要的 API 调用。

:::note
所有订阅都要求标的存在于缓存中。为简单起见,建议为你打算订阅的场所
加载全部标的。
:::

## 实时数据客户端

`TardisDataClient` 将 Tardis Machine 集成到一个正在运行的
NautilusTrader 系统中。Python 实时数据客户端将标准订阅转换为 Tardis
Machine 数据流,用于:

- `OrderBookDelta`(来自 Tardis 的 L2 粒度,包括增量或全深度快照)
- `QuoteTick`
- `TradeTick`
- `Bar`(使用 [Tardis 支持的 K 线聚合方式](#k-线)的成交 K 线)
- `FundingRateUpdate`(来自 derivative_ticker 消息)

当 `book_snapshot_output` 为 `depth10` 时,已配置的 Tardis Machine
回放/流选项也可以发出 `OrderBookDepth10`。Tardis Machine 回放路径和
catalog 写入器支持来自 `option_summary` 的 `OptionGreeks`。设置
`extract_bbo_as_quotes` 也可以从这些 `option_summary` 消息中的最优
买卖价字段发出 `QuoteTick`。

### 数据 WebSocket

主 `TardisMachineClient` 数据 WebSocket 管理在初始连接阶段(由
`ws_connection_delay_secs` 指定的时长内)收到的所有数据流订阅。对于
在此期间之后发起的任何额外订阅,会创建一个新的 `TardisMachineClient`。
这使得主 WebSocket 能够在单一数据流中处理大量启动订阅。

当通过 `ws_connection_delay_secs` 设置了初始订阅延迟时,取消这些数据
流中的任何订阅都不会将该订阅从 Tardis Machine 数据流中移除,因为
Tardis 不支持选择性取消订阅。该组件仍会取消消息总线发布的订阅。

在任何初始延迟期之后发起的所有订阅都表现正常,在请求时会完全取消
Tardis Machine 数据流的订阅。

:::tip
如果你预期会频繁地订阅和取消订阅数据,请将 `ws_connection_delay_secs`
设置为零。这会为每个初始订阅创建一个新客户端,使每个客户端都能在
取消订阅时单独关闭。
:::

## 成交 ID 推导

成交 tick 使用 Tardis 消息或 CSV 行中场所提供的成交 ID 作为
`TradeId`。当场所省略了成交 ID(在某些交易所上为空字符串或 null)时,
WebSocket 解析器和 CSV 解析器都会回退到对符号、时间戳、价格、数量和
方向进行确定性 FNV-1a 哈希计算。相同的场所事件在多次回放中会产生
相同的成交 ID,从而保持下游去重的正确性。

## 限制与注意事项

目前已知以下限制和注意事项:

- `TardisDataClient` 不支持历史报价和成交请求。历史外部 `Bar` 请求
  使用 Tardis Machine 回放,需要基于日期的回放窗口。对于 catalog
  工作流,建议优先使用 `run_tardis_machine_replay`。

## 贡献

:::info
如需了解更多功能或为 Tardis 适配器贡献代码,请参阅我们的
[贡献指南](https://github.com/nautechsystems/nautilus_trader/blob/develop/CONTRIBUTING.md)。
:::
