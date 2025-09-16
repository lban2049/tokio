# 功能标志

Tokio 采用模块化设计，允许你仅包含所需的部分。这通过 `Cargo.toml` 文件中的一组功能标志进行管理，有助于最小化依赖项并减小最终编译的二进制文件体积。默认情况下，不启用任何功能。

## 使用建议

功能标志的配置方式通常取决于你是在构建应用程序还是库。

### 对于应用程序

构建应用程序时，最简单的入门方法通常是启用 Tokio 的所有功能。这能确保你能够使用全套工具，并且在构建过程中不会遇到组件缺失的问题。如果你不确定需要哪些功能，`full` 是一个安全且推荐的选择。

```toml Cargo.toml icon=simple-icons:cargotoolbox
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### 对于库

作为库的作者，你的目标应该是创建一个轻量级的 crate。为此，你应该只启用库所需的特定功能。这让库的用户可以避免引入不必要的依赖项。

例如，一个只需要派生任务和使用 TCP 流的库，可以启用 `rt` 和 `net` 功能：

```toml Cargo.toml icon=simple-icons:cargotoolbox
[dependencies]
tokio = { version = "1", features = ["rt", "net"] }
```

## 可用标志

以下是可用的功能标志及其所启用功能的详细列表。

| 功能标志 | 描述 |
| :--- | :--- |
| `full` | 启用除 `test-util` 和 `tracing` 之外的所有下述公共 API。推荐用于应用程序。 |
| `rt` | 启用 `tokio::spawn`、当前线程调度器以及相关的非调度器工具。 |
| `rt-multi-thread` | 为高并发应用程序启用强大的、多线程的、工作窃取（work-stealing）调度器。 |
| `macros` | 启用 `#[tokio::main]` 和 `#[tokio::test]` 过程宏，以方便地进行运行时设置。 |
| `sync` | 启用 `tokio::sync` 中的所有同步原语，例如通道（`mpsc`、`oneshot` 等）和 `Mutex`。 |
| `time` | 启用 `tokio::time` 中与时间相关的工具，包括 `sleep`、`interval` 和 `timeout`。 |
| `net` | 启用 `tokio::net` 中的网络原语，例如 `TcpStream`、`UdpSocket` 和 `UnixStream`。 |
| `fs` | 启用 `tokio::fs` 中的异步文件系统操作。 |
| `process` | 启用 `tokio::process` 中用于派生和管理子进程的类型。 |
| `signal` | 启用 `tokio::signal` 中处理操作系统信号（如 `SIGINT`）的功能。 |
| `io-util` | 启用 I/O 工具 trait 和组合器，提供与 `std::io` 对应的异步版本。 |
| `io-std` | 启用异步的 `Stdout`、`Stdin` 和 `Stderr`。 |
| `test-util` | 启用 Tokio 运行时的测试基础设施。不适用于生产环境。 |
| `parking_lot` | 内部使用 `parking_lot` crate 的同步原语作为潜在的优化。 |

**注意：** 核心 I/O trait `AsyncRead` 和 `AsyncWrite` 始终可用，无需启用任何功能标志。

## 不稳定功能

Tokio 还包含一些被视为不稳定的功能。它们的 API 不受语义化版本控制的约束，可能会在次要版本中发生变化。使用这些功能需要显式选择启用。

关于如何启用和使用这些功能的完整指南，请参阅[不稳定功能](./configuration-unstable.md)文档。