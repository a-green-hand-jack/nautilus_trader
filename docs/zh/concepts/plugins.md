# 插件（Plugins）

`nautilus-plugin` crate 定义了 NautilusTrader 的插件（plug-in）产物契约（artifact contract）。它使一个独立编译的 Rust
`cdylib` 能够通过带版本信息的构建元数据（build metadata）和清单（manifest）标识自身，
并提供该边界处使用的 C-ABI 边界原语（boundary primitives）。它仅涵盖产物身份和边界类型；
不负责加载、注册或运行插件。

:::warning
插件 ABI 目前处于早期 alpha 阶段，契约尚不稳定。请将插件构建版本固定到匹配的
`nautilus-plugin` 版本。
:::

插件是一个导出单一 `nautilus_plugin_init` 入口符号的 Rust `cdylib`。
`nautilus_plugin!` 宏会生成该符号以及携带构建身份信息的静态清单：

```rust
nautilus_plugin::nautilus_plugin! {
    name: "example-plugin",
    vendor: "Nautech",
    version: env!("CARGO_PKG_VERSION"),
}
```

在产物的 `Cargo.toml` 中设置 `crate-type = ["cdylib"]`，并依赖匹配版本的
`nautilus-plugin`。有关边界类型和清单类型的详细信息，请参阅 [crate 文档](https://docs.rs/nautilus-plugin)。
