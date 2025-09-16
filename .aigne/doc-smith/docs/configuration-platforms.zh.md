# 平台支持

本文档概述了 Tokio 官方支持的平台，以及针对 WebAssembly (WASM) 等环境的特殊注意事项。

## 保证支持

Tokio 保证为以下平台提供支持，这意味着它们会得到积极的测试和维护。Tokio 的未来版本将继续支持这些平台，但最低版本要求（例如 Linux 上的 libc 版本或 Android 上的 API 级别）可能会随时间推移而改变。

- Linux
- Windows
- Android（API 级别 21 及更高版本）
- macOS
- iOS
- FreeBSD

## 其他平台

除了保证支持的平台外，Tokio 也致力于在底层 `mio` crate 支持的所有平台上运行。如需更详尽的列表，请参阅 [mio 的平台文档](https://crates.io/crates/mio#platforms)。

然而，我们不保证为这些额外平台提供支持，并且在未来的版本中可能会停止支持。请注意，Wine 被视为与 Windows 不同的平台；有关 Wine 支持的详细信息，请查阅 `mio` 文档。

## WASM 支持

Tokio 为 WebAssembly (`WASM`) 平台提供有限但专用的支持。

### 稳定支持

当使用不带 `tokio_unstable` 标志的稳定工具链时，`WASM` 支持以下功能：

| 功能 | 描述 |
| :--- | :--- |
| `sync` | 同步原语，如通道和互斥锁。 |
| `macros` | 过程宏，例如 `#[tokio::main]`。 |
| `io-util` | 核心 I/O 实用工具 trait 和函数。 |
| `rt` | 用于调度任务的单线程运行时。 |
| `time` | 用于基于时间的操作（如 `sleep` 和 `interval`）的实用工具。 |

启用任何不在此列表中的功能（包括 `full` 功能）将导致编译失败。

**重要注意事项：**

- `time` 模块仅在提供计时器支持的 `WASM` 目标（例如 `wasm32-wasi`）上才能正常工作。在没有计时器支持的平台上调用计时函数将引发 panic。
- 如果 `WASM` 上的 Tokio 运行时进入无限期空闲状态，它将立即 panic，而不是永久阻塞。在没有计时器支持的平台上，这意味着运行时永远不能处于空闲状态。

### 非稳定版 WASM 支持

通过启用 `tokio_unstable` 配置标志，您可以获得对 `WASM` 的额外、非稳定版支持。

这使得在 `wasm32-wasi` 目标上可以使用 `tokio::net` 模块。然而，由于 `WASI` 标准目前的限制，不支持从 `WASM` 内部创建新的套接字。因此，必须使用 `FromRawFd` trait，通过宿主环境提供的文件描述符来创建网络套接字。