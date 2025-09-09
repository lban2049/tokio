# 概述

Tokio 是一个运行时，用于通过 Rust 编程语言编写可靠、异步且轻量的网络应用程序，同时不影响速度。

它是一个用于编写异步应用程序的事件驱动、非阻塞 I/O 平台。总体而言，它提供了构建健壮且高性能的系统所必需的几个主要组件。

<x-cards data-columns="3">
  <x-card data-title="快速" data-icon="lucide:gauge-circle">
    Tokio 的零成本抽象为你提供裸机性能，确保你的应用程序运行速度尽可能快。
  </x-card>
  <x-card data-title="可靠" data-icon="lucide:shield-check">
    通过利用 Rust 的所有权、类型系统和并发模型，Tokio 帮助你减少错误并确保线程安全。
  </x-card>
  <x-card data-title="可扩展" data-icon="lucide:bar-chart-big">
    Tokio 占用极少的资源，并能自然地处理背压和取消，使你的应用程序能够高效扩展。
  </x-card>
</x-cards>

## 核心组件

Tokio 围绕几个关键组件构建，这些组件为异步应用程序提供了基础：

*   **异步任务工具**：用于管理任务生命周期的原语，包括任务间的同步和通信（通道、互斥锁），以及处理时间的实用工具（超时、休眠、间隔）。
*   **异步 I/O API**：一套丰富的用于非阻塞 I/O 的 API，包括 TCP 和 UDP 套接字、文件系统操作以及进程和信号管理。
*   **运行时**：一个用于执行异步代码的运行时，它包括一个多线程、工作窃取的任务调度器，一个由操作系统事件队列（如 epoll、kqueue 或 IOCP）支持的 I/O 驱动程序，以及一个高性能计时器。

## Tokio 概览

Tokio 的功能被组织成几个模块，每个模块都有不同的用途。以下是主要 API 的简要介绍。

### 使用任务

Rust 中的异步程序是围绕称为任务的轻量级、非阻塞执行单元构建的。Tokio 提供了强大的工具来管理它们。

- **[`tokio::task`](./api-task.md)**：包含处理任务的核心工具，例如用于在运行时上调度新任务的 `spawn` 函数。
- **[`tokio::sync`](./api-sync.md)**：为任务提供同步原语，包括用于通信的通道（`oneshot`、`mpsc`、`watch`、`broadcast`）和用于保护共享数据的非阻塞 `Mutex`。
- **[`tokio::time`](./api-time.md)**：提供用于跟踪时间的实用工具，使你能够设置超时、休眠指定时长或按间隔重复操作。

### 异步 I/O

Tokio 提供了一套全面的模块，用于执行异步输入和输出操作。

- **[`tokio::io`](./api-io.md)**：Tokio I/O 的基础，提供核心的 `AsyncRead` 和 `AsyncWrite` trait。
- **[`tokio::net`](./api-net.md)**：包含用于网络编程的 TCP、UDP 和 Unix 域套接字的非阻塞版本。
- **[`tokio::fs`](./api-fs.md)**：提供用于文件系统 I/O 的异步 API，类似于标准库的 `std::fs`。
- **[`tokio::signal`](./api-signal.md)** 和 **[`tokio::process`](./api-process.md)**：允许异步处理操作系统信号和管理子进程。

### 运行时

Tokio 运行时负责执行异步任务。虽然大多数应用程序可以从简单的 `#[tokio::main]` 宏开始，但当需要更多控制时，[`tokio::runtime`](./api-runtime.md) 模块提供了强大的 API，用于对运行时进行细粒度配置和管理。

### CPU 密集型任务和阻塞代码

Tokio 专为 I/O 密集型应用程序设计，使用少量线程处理大量并发任务。执行长时间、CPU 密集型计算而没有 `await` 的代码会阻塞线程，从而阻止其他任务运行。为了处理这种情况，Tokio 提供了 `spawn_blocking`，它将阻塞或 CPU 密集型计算移至专用的线程池，从而确保主异步调度器保持响应。

```rust A blocking task example icon=logos:rust
#[tokio::main]
async fn main() {
    // 这在核心线程上运行。

    let blocking_task = tokio::task::spawn_blocking(|| {
        // 这在阻塞线程上运行。
        // 在这里阻塞是可以的。
    });

    // 我们可以等待阻塞任务完成。
    blocking_task.await.unwrap();
}
```

## 准备好深入了解了吗？

本概述介绍了 Tokio 背后的核心思想。最好的学习方式是实践。请前往我们的[入门指南](./getting-started.md)，在几分钟内设置你的第一个 Tokio 应用程序。
