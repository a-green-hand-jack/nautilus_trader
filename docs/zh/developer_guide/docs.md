# 文档风格

本指南概述了为 NautilusTrader 编写文档时应遵循的风格约定和最佳实践。

## 总体原则

- 我们倾向于简单而非复杂，简洁即是美。
- 我们倾向于简洁而又易读的行文和文档风格。
- 我们重视约定、风格、模式等方面的标准化。
- 文档应当能被不同技术背景的用户理解。

## 文档类型

大多数页面应属于以下四种类型之一
（参见 [Divio 文档系统](https://docs.divio.com/documentation-system/)）。
在单个页面中混合多种类型会使其更难阅读、更难维护。

| 类型                  | 目的                          | 所在目录          |
|------------------|----------------------------------|------------------|
| **教程（Tutorial）**     | 通过完整走一遍任务来进行教学  | `tutorials/`     |
| **操作指南（How-to guide）** | 解决某个具体问题         | `how_to/`        |
| **说明（Explanation）**  | 阐明设计与架构  | `concepts/`      |
| **参考（Reference）**    | 描述系统内部机制           | `api_reference/` |

有两个目录属于例外：`getting_started/` 是一条入门路径，将教程式的
分步引导与安装配置说明结合在一起；`integrations/` 页面将参考内容（能力、代码符号规则）
与操作指南内容（安装、配置）混合在一起，使每个交易所/平台页面自成一体。
不针对特定交易所/平台的独立操作指南内容应放在 `how_to/` 中。

### 选择正确的类型

- **这个页面是否引导新手完成一次学习体验？** 是教程。
- **它是否为已经了解系统的用户回答"我该如何……？"这样的问题？** 是操作指南。
- **它是否解释了某件事情为何以这种方式运作？** 是说明。
- **它是否罗列了类、配置字段、枚举或能力清单？** 是参考。

教程说的是"先做这个，然后做这个，再做这个"，路径由作者选定。
操作指南说的是"这是实现 X 的方法"，读者已经知道自己想要 X。
请将这两者区分开来：

- 教程不应假设读者具备预先知识。
- 操作指南不应教授背景概念。

当一种类型需要引用另一种类型的内容时，应链接过去而不是内联复制。例如，
配置 `TradingNodeConfig` 的操作指南应链接到关于字段定义的 API 参考，
而不是再次罗列这些字段。

## 语言与语气

- 尽可能使用主动语态（例如"配置适配器"而非"适配器应当被配置"）。
- 描述当前功能时使用现在时态。
- 只在描述计划中的功能时使用将来时态。
- 避免不必要的行话；技术术语首次出现时给出定义。
- 直接、简洁；避免使用"基本上"、"简单地"、"只是"这类填充词。
- 在列表中使用平行结构；各条目之间保持一致的语法模式。

## Markdown 表格

### 列对齐与间距

- 根据每列中最宽内容所需的空间，使用对称的列宽。
- 垂直对齐列分隔符（`|`），以提升可读性。
- 单元格内容周围保持一致的空格。

### 说明与描述

- 所有说明和描述都应以句号结尾。
- 说明应简洁而富有信息量。
- 使用句首字母大写（只有首字母和专有名词大写）。

### 示例

```markdown
| Order Type             | Spot | Margin | USDT Futures | Coin Futures | Notes                   |
|------------------------|------|--------|--------------|--------------|-------------------------|
| `MARKET`               | ✓    | ✓      | ✓            | ✓            |                         |
| `STOP_MARKET`          | -    | ✓      | ✓            | ✓            | Not supported for Spot. |
| `MARKET_IF_TOUCHED`    | -    | -      | ✓            | ✓            | Futures only.           |
```

### 支持情况标识

- 使用 `✓` 表示支持的功能。
- 使用 `-` 表示不支持的功能（不要使用 `✗` 或其他符号）。
- 为不支持的功能添加说明时，使用斜体强调：`*Not supported*`。
- 当原因很重要时，让不支持的说明更加具体：交易所层面的缺失使用
  `*Not supported by <venue>*`，适配器层面的缺失使用 `*Not currently implemented*`。
- 无需内容时，将单元格留空。

## 代码引用

- 对内联代码、方法名、类名以及配置选项使用反引号。
- 对多行示例使用代码块。
- 在引用代码位置时，使用 `file_path::function_name` 或 `file_path::ClassName`，
  而不要使用行号，因为行号会随代码变化而失效。

## 标题

我们遵循优先考虑可读性和无障碍性的现代文档约定：

- 页面主标题使用标题格式大小写（仅限一级标题 `#`）。
- 所有子标题（二级标题 `##` 及以下）使用句首字母大写格式。
- 无论标题层级如何，专有名词（产品名称、技术名称、公司名称、缩写词）始终大写。
- 使用正确的标题层级结构（不要跳级）。

这一约定与谷歌开发者文档、微软文档以及 Anthropic 文档等主流科技公司使用的行业标准保持一致，
能提升可读性、降低认知负担，并且对国际用户和屏幕阅读器更加友好。

### 示例

```markdown
# NautilusTrader Developer Guide

## Getting started with Python
## Using the Binance adapter
## REST API implementation
## WebSocket data streaming
## Testing with pytest
```

## 列表

- 无序列表使用连字符（`-`）作为项目符号；避免使用 `*` 或 `+`，以在整个项目中保持
  Markdown 风格一致。
- 只有在顺序有意义时才使用编号列表。
- 嵌套列表保持一致的缩进。
- 如果列表项是完整的句子，应以句号结尾。

## 链接与引用

- 使用具描述性的链接文字（避免使用"点击这里"或"此链接"）。
- 在合适的场合引用外部文档。
- 保持所有内部链接为相对路径且准确无误。

## 技术术语

- 能力矩阵应基于 Nautilus 的领域模型，而非特定交易所的术语。
- 必要时可在括号或说明中提及交易所特定的术语以便理解。
- 在整个文档中保持术语的一致性。

## 示例与代码样例

- 提供实用、可运行的示例。
- 包含必要的导入语句和上下文。
- 使用真实、贴近实际的变量名和值。
- 为示例中不易理解的部分添加注释。

## 提示框（Admonitions）

使用提示框来突出重要信息：

| 提示框类型   | 用途                                                       |
|--------------|---------------------------------------------------------------|
| `:::note`    | 起补充说明作用、能澄清但非必需的上下文信息。     |
| `:::info`    | 读者应了解的重要信息。          |
| `:::tip`     | 有帮助的建议或最佳实践。                        |
| `:::warning` | 潜在的陷阱或重要注意事项。                      |
| `:::danger`  | 可能导致数据丢失或系统故障的严重问题。 |

避免过度使用提示框；使用过多会削弱其效果。

## MDX 组件

文档站点（fumadocs）在所有 `.md` 文件中都提供了内置的 MDX 组件，无需导入即可使用。

### Tabs（选项卡）

对特定语言或变体的内容使用选项卡。Rust 排在 Python 之前，使 Rust 成为
默认（最左侧）选项卡。

对于代码示例，在连续的围栏代码块上添加 `tab="..."`：

```markdown
\`\`\`rust tab="Rust"
let params = Params::from([("close_position", true.into())]);
\`\`\`

\`\`\`python tab="Python"
strategy.submit_order(order, params={"close_position": True})
\`\`\`
```

对于表格或其他内容，用 `<Tabs>` 和 `<Tab>` 包裹每个变体。字段（Fields）表格中即使用了这种方式，
使每种语言各显示单独一列类型，而不是 Rust 和 Python 并排的两列。在内部内容的上方和下方各留一个
空行，以便 Markdown 正确渲染。

```markdown
<Tabs items={["Rust", "Python"]}>
<Tab value="Rust">

| Field           | Type           | Required/default | Notes                   |
|-----------------|----------------|------------------|-------------------------|
| `instrument_id` | `InstrumentId` | Required         | Stored as `id` in Rust. |

</Tab>
<Tab value="Python">

| Field           | Type           | Required/default | Notes |
|-----------------|----------------|------------------|-------|
| `instrument_id` | `InstrumentId` | Required         |       |

</Tab>
</Tabs>
```

### Steps（步骤）

对顺序性的操作步骤使用 `Steps` 和 `Step`。

```markdown
<Steps>
<Step>
Configure the adapter.
</Step>
<Step>
Start the trading node.
</Step>
</Steps>
```

### Accordions（折叠面板）

对可折叠内容使用 `Accordions` 和 `Accordion`。

```markdown
<Accordions>
<Accordion title="Advanced configuration">
Content here.
</Accordion>
</Accordions>
```

### Files（文件树）

对目录树可视化使用 `Files`、`Folder` 和 `File`。

```markdown
<Files>
<Folder name="src" defaultOpen>
<File name="main.rs" />
<File name="lib.rs" />
</Folder>
</Files>
```

### Cards（卡片）

对带链接的内容网格使用 `Cards` 和 `Card`。

```markdown
<Cards>
<Card title="Getting started" href="/latest/getting_started" />
<Card title="Concepts" href="/latest/concepts" />
</Cards>
```

### TypeTable

对参数或类型文档表格使用 `TypeTable`。

## 行长与换行

- 每行长度不超过约 100-120 个字符，以提升可读性并方便代码审查中的差异比较。
- 在自然断点处（逗号、连接词或短语之后）拆分长句。
- 尽量避免在新行开头出现孤立单词。
- 代码块和 URL 在必要时可以超出该行长限制。

## API 文档

- 清晰地记录参数和返回类型。
- 为复杂的 API 提供使用示例。
- 说明任何副作用或重要行为。
- 参数描述应简洁但完整。
