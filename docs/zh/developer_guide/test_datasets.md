# 测试数据集

关于整理、存储和使用作为测试固定装置（fixture）的外部数据集的目标标准。
新的数据集应遵循这些标准。早于该策略制定的现有数据集记录在
[旧版数据集](#legacy-datasets) 一节中。

## 数据集分类

**小数据**（小于 1 MB）会直接连同 `metadata.json` 文件一起提交到
`tests/test_data/<source>/` 中。这些文件始终可用，无需网络访问。

**大数据**（大于 1 MB）以 Parquet 格式托管在 R2 测试数据存储桶（bucket）中。
`tests/test_data/large/checksums.json` 中记录了 SHA-256 校验和。
`ensure_test_data_exists()` 辅助函数会在首次使用时下载该文件并验证其完整性。

**用户自取数据（User-fetched data）** 用于当供应商许可、授权模型或访问控制不允许
NautilusTrader 通过公共仓库或公共 R2 存储桶重新分发数据的情况。在这种模式下，
仓库只存储清单、获取说明和转换代码。每个用户使用自己的供应商账户下载源数据，
并在本地进行转换。

在以下任一情况下应使用用户自取模式：

- 供应商要求每个用户拥有各自的账户、API key 或历史数据许可。
- 许可证允许内部使用，但没有明确允许重新分发衍生固定装置。
- 该数据集适合用于示例或按需选用的集成测试，但不适合默认 CI。

## 必需的元数据

每一个存储或重新分发具体产物的整理过的数据集，都必须包含一个至少包含以下内容的
`metadata.json`：

| 字段          | 描述                                                       |
|----------------|-------------------------------------------------------------------|
| `file`         | 数据集的文件名。                                          |
| `sha256`       | 该文件的 SHA-256 哈希值。                                         |
| `size_bytes`   | 文件大小（字节）。                                               |
| `original_url` | 原始源数据的下载 URL。                         |
| `licence`      | 许可条款及任何重新分发限制。                 |
| `added_at`     | 该数据集被整理时的 ISO 8601 时间戳。                  |

这些字段与 `scripts/curate-dataset.sh` 的输出相匹配。以下是为提供更丰富来源信息而
建议添加的额外字段：

| 字段           | 描述                                                      |
|-----------------|--------------------------------------------------------------------|
| `instrument`    | 涵盖的金融工具代码。                                    |
| `date`          | 涵盖的交易日期。                                    |
| `format`        | 存储格式（例如 "Nautilus OrderBookDelta Parquet"）。        |
| `original_file` | 转换前的原始供应商文件名。                                  |
| `parser`        | 转换时使用的解析器（例如 "itchy 0.3.4"）。            |

用户自取数据集在适用的地方使用相同的元数据字段。它们还应当包含：

| 字段                 | 描述                                                           |
|-----------------------|-------------------------------------------------------------------------|
| `distribution`        | 必须为 `"user-fetch"`。                                               |
| `fetch_method`        | 用户获取源数据的方式（API、网页门户、CLI 等）。   |
| `fetch_reference`     | 面向用户的下载流程的 URL 或文档参考。          |
| `auth`                | 所需的凭据或授权（如有）。                         |
| `transform_version`   | 用于构建最终文件的本地转换流水线的版本。   |
| `redistribution`      | 描述该数据集重新分发限制的简短说明。          |
| `public_mirror`       | 对于受限的供应商数据集，必须为 `false`。                       |

对于没有单一已提交或已镜像产物的用户自取数据集，`metadata.json` 中可以省略 `file`、
`sha256` 和 `size_bytes`。在这种情况下，`manifest.json` 中的 `target_files` 是本地
输出文件的权威来源。对于用户自取数据集，当确切的文件是按用户账户或按请求生成时，
`original_url` 可以指向供应商的下载入口，而不是确切的文件 URL。

其他元数据字段在适用时仍然是推荐使用的。特别是，`licence` 和 `added_at` 仍应为
用户自取数据集记录下来。

## 存储格式

新的数据集应存储为 **Nautilus Parquet**（而不是原始的供应商格式）。
这可以确保：

- 所有测试数据集之间数据类型保持一致。
- 测试时无需解析供应商格式。
- 在许可方面明确其衍生作品（derivative-work）状态。

使用 ZSTD 压缩（等级 3），行组大小为 100 万行。

用户自取数据集在完成本地转换步骤后，也应最终变成 Nautilus Parquet 格式。
原始供应商文件应保留在仓库之外，也不应放入公共 R2 存储桶。

## 命名约定

```
<source>_<instrument>_<date>_<datatype>.parquet
```

示例：

- `itch_AAPL_2019-01-30_deltas.parquet`
- `tardis_BTCUSDT_2020-09-01_depth10.parquet`
- `histdata_EURUSD.SIM_2020-01_quotes.parquet`

## 整理工作流

### 简单文件（单次下载）

使用 `scripts/curate-dataset.sh`：

```bash
scripts/curate-dataset.sh <slug> <filename> <download-url> <licence>
```

这会创建一个带版本号的目录（`v1/<slug>/`），其中包含该文件、
`LICENSE.txt`，以及包含上述必需字段的 `metadata.json`。

### 复杂流水线（解析 + 转换）

对于需要格式转换的数据集（例如从二进制 ITCH 转为 Parquet）：

1. 在 `crates/testkit/src/<source>/` 中编写一个整理函数，用 `#[cfg(test)]`
   或 `#[ignore]` 测试进行门控。
2. 该函数应完成：下载、解析、过滤、转换为 NautilusTrader 类型、写入 Parquet。
3. 将 Parquet 文件和 `metadata.json` 输出到本地目录。
4. 手动上传到 R2，然后将校验和添加到 `checksums.json` 中。

### 用户自取流水线（受限重新分发）

对于 NautilusTrader 无法重新分发的数据集：

1. 提交一份清单和 `metadata.json`，但不要提交真实的供应商数据或转换后的
   Parquet 输出。
2. 提供一个本地获取命令或辅助工具，使用用户自己的供应商凭据、授权或已购买的
   历史数据文件。
3. 在本地将供应商数据转换为 Nautilus Parquet。
4. 将转换后的文件存储到一个被 git 忽略的本地缓存路径中。
5. 让测试和示例按需选用（opt in）。当数据集缺失时它们应能干净地跳过。

新数据集的默认分发优先顺序为：

1. 直接提交的小数据。
2. 公共 R2 大数据。
3. 用户自取数据。

只有当前两种方案在供应商条款下不被允许时，才选择用户自取方式。

不要：

- 将受限的供应商数据集上传到公共 R2 存储桶。
- 在重新分发权利不明确的情况下，将真实的、由供应商衍生的 Parquet 文件提交到仓库。
- 让默认 CI 依赖供应商凭据或付费的历史数据访问权限。

如果许可证允许内部共享，你可以为内部 CI 或员工维护一个私有镜像。请将其视为
独立的运维路径，而不是公共测试数据标准的一部分。

## 添加新数据集

1. 按照上述工作流整理数据。
2. 编写包含所有必需字段的 `metadata.json`。
3. 对于小数据：提交到 `tests/test_data/<source>/`。
4. 对于大数据：将 Parquet 上传到 R2，并将校验和添加到
   `tests/test_data/large/checksums.json`。
5. 对于用户自取数据：只提交清单和获取说明。将源数据和转换后的数据都排除在
   仓库和公共 R2 存储桶之外。
6. 当需要共享的 testkit 访问时，将路径辅助函数添加到 `crates/testkit/src/common.rs`。
7. 编写使用该数据集的测试。

对于用户自取数据，推荐以下布局：

```text
tests/test_data/<source>/<slug>/
  metadata.json
  manifest.json
  README.md
```

使用 `tests/test_data/local/<source>/<slug>/` 作为生成产物的标准本地缓存路径。
当需要本地保留时，将原始的供应商下载文件放在同一缓存路径下的相邻 `vendor/` 目录中。

清单应当是机器可读且稳定的。它应捕获在另一台机器上复现获取和转换步骤所需的
最少信息。

`metadata.json` 是来源信息、许可和重新分发规则的权威来源。
`manifest.json` 是获取输入、命令、缓存位置和输出文件的权威来源。

推荐的清单字段：

| 字段               | 描述                                                        |
|---------------------|----------------------------------------------------------------------|
| `slug`              | 稳定的数据集标识符。                                         |
| `vendor`            | 供应商或交易场所名称。                                              |
| `source_type`       | `api`、`portal-download`、`purchased-archive` 等。                |
| `source_filters`    | 交易品种、事件 ID、市场 ID、日期范围或文件名。        |
| `target_files`      | 转换后预期得到的 Nautilus Parquet 输出文件。            |
| `cache_dir`         | 相对于 `tests/test_data/local/` 的本地输出位置。         |
| `fetch_command`     | 建议的命令或脚本入口点。                           |
| `transform_command` | 建议的本地转换命令。                                |
| `env`               | 所需的环境变量。                                    |
| `notes`             | 面向用户的简短操作说明。                                 |

依赖用户自取数据的测试应当：

- 与默认 CI 测试单独标记或分组。
- 当本地数据集不存在时，以清晰的消息跳过。
- 除非用户明确选用，否则避免网络访问。
- 复用一个稳定的本地缓存路径，使获取操作在每台机器上只发生一次。

对于基于 pytest 的测试，建议使用如下的保护语句：

```python
if not filepath.exists():
    pytest.skip(f"User-fetched test data not found: {filepath}")
```

对于需要手动准备数据集的 Rust 测试，如果该测试不打算在默认 CI 中运行，
建议使用 `#[ignore]`。

## 测试运行器的序列化

下载大数据文件的测试会在多个测试二进制文件之间共享目标路径。由于 `nextest`
会在独立进程中运行每个二进制文件，向同一路径并发下载可能会产生竞态。
`.config/nextest.toml` 中的 nextest 配置定义了一个 `large-data-tests` 分组，
将 `max-threads` 设置为 1，以对这些二进制文件进行序列化。

当添加一个新的、会下载大型共享文件的测试二进制文件时，请将其添加到该分组过滤器中：

```toml
[[profile.default.overrides]]
filter = 'binary(grid_mm_itch) | binary(orderbook_integration) | binary(your_new_binary)'
test-group = 'large-data-tests'
```

## 重新生成数据集

当模式（schema）变更使某个大型 Parquet 文件失效时，使用下方的整理测试从原始源数据
重新生成该文件。重新生成之后：

1. `sha256sum /tmp/<output_file>.parquet`
2. 用新的哈希值更新 `tests/test_data/large/checksums.json`。
3. 更新对应的 `metadata.json`（sha256、size_bytes）。
4. 将 Parquet 文件上传到 R2。
5. 提交 `checksums.json` 和 `metadata.json`（这也会使 CI 缓存失效）。

### ITCH AAPL L3 深度变化数据

来源：来自 NASDAQ EMI 的 `01302019.NASDAQ_ITCH50.gz`（约 4.4 GB）。

```bash
# 下载源文件（请保留一份本地副本，这是一个大文件）
wget -O ~/Downloads/01302019.NASDAQ_ITCH50.gz \
  "https://emi.nasdaq.com/ITCH/Nasdaq%20ITCH/01302019.NASDAQ_ITCH50.gz"

# 整理测试期望源文件位于 /tmp
ln -sf ~/Downloads/01302019.NASDAQ_ITCH50.gz /tmp/01302019.NASDAQ_ITCH50.gz

# 重新生成 parquet（输出：/tmp/itch_AAPL.XNAS_2019-01-30_deltas.parquet）
cargo test -p nautilus-testkit --lib test_curate_aapl_itch -- --ignored --nocapture
```

### Tardis Deribit BTC-PERPETUAL L2 深度变化数据

来源：来自 [Tardis](https://tardis.dev/) 的
`tardis_deribit_incremental_book_L2_2020-04-01_BTC-PERPETUAL.csv.gz`。每月首日的数据
可以作为免费样例获取（无需 API key）。

```bash
# 下载源文件（免费样例，无需 API key）
wget -O tests/test_data/large/tardis_deribit_incremental_book_L2_2020-04-01_BTC-PERPETUAL.csv.gz \
  "https://datasets.tardis.dev/v1/deribit/incremental_book_L2/2020/04/01/BTC-PERPETUAL.csv.gz"

# 重新生成 parquet（输出：/tmp/tardis_BTC-PERPETUAL.DERIBIT_2020-04-01_deltas.parquet）
cargo test -p nautilus-tardis test_curate_deribit_deltas -- --ignored --nocapture
```

## 教程测试数据

多个教程会加载用户提供的市场数据。`NAUTILUS_DATA_DIR` 环境变量会覆盖这些教程所使用的
基础数据路径。测试套件将该变量设置为 `tests/test_data/local/`，
以便教程针对本地存储的小样本文件运行。

### 目录布局

```text
tests/test_data/local/
  Binance/
    BTCUSDT_T_DEPTH_2022-11-01_depth_snap.csv
    BTCUSDT_T_DEPTH_2022-11-01_depth_update.csv
  Bybit/
    2024-12-01_XRPUSDT_ob500.data.zip
  HISTDATA/
    DAT_ASCII_EURUSD_T_202001.csv.gz
```

`tests/test_data/local/` 目录被 git 忽略。当数据缺失时，测试会自动跳过。

### 获取数据

**Binance 深度快照** 可从
[Binance 公共数据门户](https://data.binance.vision/) 获取。下载 2022-11-01 的
BTCUSDT T_DEPTH 文件，将快照和更新 CSV 文件放到 `tests/test_data/local/Binance/` 下。
对于测试而言，一部分行（例如前 10,000 行）就足够了。

**Bybit ob500 订单簿数据** 可从 Bybit CDN 获取：

```bash
curl -L "https://quote-saver.bycsi.com/orderbook/linear/XRPUSDT/2024-12-01_XRPUSDT_ob500.data.zip" \
  -o tests/test_data/local/Bybit/2024-12-01_XRPUSDT_ob500.data.zip
```

完整文件大约 360 MB。对于测试而言，提取前几百行并重新打包成一个更小的 zip 文件即可。

**HISTDATA 逐笔数据** 可从 [histdata.com](https://www.histdata.com/) 获取。
下载任意月份的 EUR/USD ASCII 逐笔数据，将 CSV（或 `.csv.gz`）文件放到
`tests/test_data/local/HISTDATA/` 下。

### 运行测试

```bash
pytest tests/docs_tests/test_tutorials.py::test_tutorial_with_local_data -v
```

当对应的数据子目录为空或缺失时，测试会以一条消息跳过。

## 旧版数据集

这些数据集早于本策略制定，使用的是没有 `metadata.json` 的原始供应商格式
（CSV/CSV.gz）。它们对现有测试仍然有效。新的数据集应遵循上面的 Parquet 标准。

| 数据集                       | 来源   | 格式           | 位置                  | 状态   |
|-------------------------------|----------|------------------|---------------------------|----------|
| Tardis Deribit L2 深度变化数据      | Tardis   | Parquet（大）  | `tests/test_data/large/`  | 已整理  |
| ITCH AAPL L3 深度变化数据           | NASDAQ   | Parquet（大）  | `tests/test_data/large/`  | 已整理  |
| HISTDATA EURUSD.SIM 报价数据    | HISTDATA | Parquet（大）  | `tests/test_data/large/`  | 已迁移 |
| Tardis Deribit L2             | Tardis   | CSV（已提交） | `tests/test_data/tardis/` | 旧版   |
| Tardis Binance 快照      | Tardis   | CSV.gz（大）   | `tests/test_data/large/`  | 旧版   |
| Tardis Bitmex 成交数据          | Tardis   | CSV.gz（大）   | `tests/test_data/large/`  | 旧版   |

原先的 `nautechsystems/nautilus_data` 目录对应于上述的 HISTDATA EURUSD.SIM Parquet
文件。原始的 HISTDATA CSV 文件仍属于用户自取数据。
