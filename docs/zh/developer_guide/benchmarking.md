# 基准测试

本文档是编写和运行 NautilusTrader 基准测试的实践参考手册，
涵盖了工具细节、目录结构、示例代码、本地运行方式以及火焰图（flamegraph）性能剖析。

关于策略层面的内容（我们对什么进行基准测试、何时进行、以何种严谨程度进行，
以及它如何与 CI 结合），请参见仓库根目录的 [`/BENCHMARKING.md`](../../BENCHMARKING.md)。

---

## 工具概览

NautilusTrader 使用两个互补的 Rust 基准测试框架：

| 框架                                                          | 测量内容                                    | 优先使用场景                                    |
|--------------------------------------------------------------|-------------------------------------------|------------------------------------------------------|
| [**Criterion**](https://docs.rs/criterion/latest/criterion/) | 带置信区间的墙钟时间（wall-clock time）     | 任何 ≥ 100 纳秒的场景；绝对测量；对比测量。 |
| [**iai**](https://docs.rs/iai/latest/iai/)                   | 已退休 CPU 指令数（通过 Cachegrind）        | 亚 100 纳秒级函数；CI 回归检测。         |

大多数热路径代码可以同时从两者中受益。Criterion 提供面向用户的可见数字；
iai 提供无噪声的回归信号。

:::note
iai 是确定性的（不受系统噪声影响），但结果与所在机器相关。请将其用于 CI 内部的回归检测，
而不是用于跨机器的比较。
:::

---

## 目录结构

每个 crate 都在本地的 `benches/` 文件夹中保存其基准测试：

```text
crates/<crate_name>/
└── benches/
    ├── foo_criterion.rs
    └── foo_iai.rs
```

在 crate 的 `Cargo.toml` 中显式注册每个基准测试，以便
`cargo bench` 能够发现它：

```toml
[[bench]]
name = "foo_criterion"
path = "benches/foo_criterion.rs"
harness = false

[[bench]]
name = "foo_iai"
path = "benches/foo_iai.rs"
harness = false
```

要接入夜间（nightly）CI 性能工作流，请将该 crate 添加到工作区
`Makefile` 中的 `cargo-ci-benches` 任务中。

---

## 编写 Criterion 基准测试

1. **将设置代码放在计时循环之外。** 任何在迭代之间不发生变化的工作，都应放在周围代码中，
   或放在 `iter_batched_ref` 的 setup 闭包中，而不是放在传给 `iter` 的函数体中。
2. **将输入用 `black_box` 包裹**，以防止优化器将其折叠掉。
3. **在有可变操作的基准测试中使用 `iter_batched_ref`。** 它会将输入的 `Drop` 排除在计时区域之外，
   否则对于拥有大型结构体的基准测试，`Drop` 往往会主导整个测量结果。
4. **为按规模参数化的分组添加 `Throughput::Elements(n)`**，以便
   Criterion 能报告每元素的吞吐量。
5. **注释测试意图。** 说明该基准测试所衡量的内容（热路径、最坏情况、
   缓存冷启动情况等），以便未来的读者理解回归意味着什么。

```rust
use std::hint::black_box;

use criterion::{BatchSize, BenchmarkId, Criterion, Throughput, criterion_group, criterion_main};

const SIZES: &[usize] = &[10, 100, 1_000];

fn bench_my_op(c: &mut Criterion) {
    let mut group = c.benchmark_group("module/my_op");

    for &n in SIZES {
        group.throughput(Throughput::Elements(n as u64));
        group.bench_with_input(BenchmarkId::from_parameter(n), &n, |b, &n| {
            b.iter_batched_ref(
                || populate(n),
                |state| state.run(black_box(n)),
                BatchSize::SmallInput,
            );
        });
    }

    group.finish();
}

criterion_group!(benches, bench_my_op);
criterion_main!(benches);
```

---

## 编写 iai 基准测试

`iai` 要求函数不接受任何参数。请保持函数体小巧，
使指令计数具有意义，也避免函数外部的变化"泄漏"进测量结果。

```rust
use std::hint::black_box;

fn bench_add() -> i64 {
    let a = black_box(123);
    let b = black_box(456);
    a + b
}

iai::main!(bench_add);
```

在运行之间存在变化的设置代码（分配、随机性、系统调用）
会以误导性的方式抬高指令计数。iai 最适合纯粹、无分配的函数。

---

## 本地运行基准测试

| 目标                                | 命令                                                              |
|-------------------------------------|----------------------------------------------------------------------|
| 一个 crate 中的所有基准测试            | `cargo bench -p nautilus-execution`                                  |
| 一个基准测试模块                    | `cargo bench -p nautilus-execution --bench matching_core`            |
| 按名称模式匹配的某个具体基准测试     | `cargo bench -p nautilus-execution --bench matching_core -- iterate` |
| 快速冒烟测试（低采样数）             | `cargo bench ... -- --quick`                                         |
| 所有 CI 跟踪的基准测试                | `make cargo-ci-benches`                                              |

Criterion 会将 HTML 报告写入 `target/criterion/`。打开
`target/criterion/report/index.html`。该报告包含每个基准测试的小提琴图（violin plot）、
置信区间，以及与上一次运行保存的基线进行的对比。

---

## 生成火焰图

`cargo-flamegraph` 会为单个基准测试生成一份采样式调用栈剖析结果（call-stack profile）。
当某个基准测试出现回归但不清楚具体是哪个内部调用导致时，这个工具非常有用。

1. 每台机器安装一次：

   ```bash
   cargo install flamegraph
   ```

2. 使用 `bench` profile 运行某个特定的基准测试：

   ```bash
   cargo flamegraph --bench matching -p nautilus-common --profile bench
   ```

3. 在浏览器中打开 `flamegraph.svg` 并放大查看热路径。

### Linux

必须要有 `perf` 可用。在 Debian/Ubuntu 上：

```bash
sudo apt install linux-tools-common linux-tools-$(uname -r)
```

如果 `perf_event_paranoid` 阻止了运行：

```bash
sudo sh -c 'echo 1 > /proc/sys/kernel/perf_event_paranoid'
```

通常设置为 `1` 就足够了。之后请将其改回 `2`（默认值），
或通过 `/etc/sysctl.conf` 进行持久化配置。

### macOS

`DTrace` 需要 root 权限，因此 `cargo flamegraph` 必须使用 `sudo` 运行。

:::warning
使用 `sudo` 运行会在 `target/` 中生成属于 root 的文件，导致后续的 `cargo` 命令出现
权限错误。你可能需要手动删除这些属于 root 的文件，或运行 `sudo cargo clean`。
:::

```bash
sudo cargo flamegraph --bench matching -p nautilus-common --profile bench
```

`bench` profile 保留了完整的调试符号，因此火焰图能渲染出可读的函数名，
而不会使生产环境的二进制文件（仍使用 `panic = "abort"` 并通过 `[profile.release]` 构建）变得臃肿。

> **注意** 基准测试二进制文件使用工作区 `Cargo.toml` 中定义的自定义 `[profile.bench]`
> 编译。该 profile 继承自 `release`，并设置了 `debug = "full"`，从而在保留完整优化的同时，
> 也保留了调试符号，使得 `cargo flamegraph` 或 `perf` 等工具能够生成可读的调用栈信息。

---

## 模板

可直接复制使用的起始模板文件位于 [`docs/dev_templates/`](../dev_templates/)：

- **Criterion**：[`criterion_template.rs`](../dev_templates/criterion_template.rs)
- **iai**：[`iai_template.rs`](../dev_templates/iai_template.rs)

将模板文件复制到目标 crate 的 `benches/` 目录下，调整导入和分组名称，
在 `Cargo.toml` 中注册，然后即可开始测量。
