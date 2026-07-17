# 开发者指南

关于开发和扩展 NautilusTrader，或者为该项目贡献代码的指南。

NautilusTrader 采用 **Rust 核心 + Python 绑定** 的架构：

- **Rust** 负责网络通信、数据解析、订单撮合等性能关键型操作。
- **Python** 提供面向用户的 API，用于策略开发、配置管理和系统集成。
- **PyO3** 在两者之间架起桥梁，以极低的开销将 Rust 的功能暴露给 Python。

这种方式将 Python 的简洁性和生态系统优势与 Rust 的性能和内存安全性结合在了一起。

## 目录

- [环境搭建](environment_setup.md)
- [设计原则](design_principles.md)
- [编码规范](coding_standards.md)
- [Rust](rust.md)
- [Python](python.md)
- [测试](testing.md)
- [测试数据集](test_datasets.md)
- [文档风格](docs.md)
- [发布说明](releases.md)
- [发布安全架构](release_security.md)
- [适配器](adapters.md)
- [数据测试规范](spec_data_testing.md)
- [执行测试规范](spec_exec_testing.md)
- [基准测试](benchmarking.md)
- [FFI 内存契约](ffi.md)
