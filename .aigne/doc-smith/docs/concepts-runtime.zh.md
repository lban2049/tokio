# 运行时

Tokio 运行时是驱动 Rust 中异步应用程序的引擎。与同步程序不同，异步代码需要一个专用环境来管理任务、处理 I/O 事件和协调基于时间的操作。Tokio `Runtime` 将所有这些必要的服务捆绑在一起提供。

概括来说，运行时负责：

-   **一个 I/O 事件循环 (驱动程序):** 这是与操作系统事件队列（例如 Linux 上的 `epoll`、macOS 上的 `kqueue` 或 Windows 上的 `IOCP`）交互的核心组件，用于驱动 I/O 资源，并在任务可以继续执行时通知它们。
-   **一个任务调度器:** 该组件管理一个轻量级的、非阻塞的任务池，决定在任何给定时间在哪个线程上运行哪个任务。
-   **一个计时器:** 它提供了在指定持续时间后调度工作运行的能力，从而实现了诸如休眠、间隔和超时等功能。

虽然您可以手动配置和管理 `Runtime` 实例，但大多数应用程序都以 `#[tokio::main]` 宏开始，该宏可以方便地设置一个默认的运行时。

## Runtime 架构

Tokio 运行时协调多个组件以高效执行异步代码。了解此架构有助于针对特定需求配置运行时以及诊断性能问题。

```d2
direction: down

"Application Code": { shape: rectangle }
os: "操作系统\n(epoll, kqueue, IOCP)": { shape: cloud }

"Tokio Runtime": {
  shape: package
  grid-columns: 1
  grid-gap: 50

  ts: "任务调度器" {
    wt: "工作线程"
  }
  drivers: "资源驱动程序" {
    grid-columns: 2
    io: "I/O 驱动程序"
    timer: "计时器"
  }
  bp: "阻塞池"
}

# Connections
"Application Code" -> "Tokio Runtime".ts: "tokio::spawn()"
"Application Code" -> "Tokio Runtime".bp: "tokio::task::spawn_blocking()"

"Tokio Runtime".ts -> "Tokio Runtime".ts.wt: "分发任务"
"Tokio Runtime".ts -> "Tokio Runtime".drivers: "轮询事件"
"Tokio Runtime".drivers.io <-> os: "I/O 事件"
```

## 调度器类型

Tokio 提供不同的调度策略，允许您为应用程序的工作负载选择最合适的策略。

### 多线程调度器

这是默认且最常用的调度器。它利用一个工作线程池（通常每个 CPU 核心一个线程），并采用工作窃取（work-stealing）策略来保持所有线程处于繁忙状态。当一个线程的本地队列中没有任务时，它会从其他更繁忙的线程“窃取”任务。这种方法对于大多数可以从并行中受益的服务器端应用程序和工作负载是理想的。

它通过 `Runtime::new()` 或 `Builder::new_multi_thread()` 默认启用。

```rust
use tokio::runtime;

// 创建一个具有默认设置的多线程运行时。
let rt = runtime::Runtime::new().unwrap();

rt.block_on(async {
    println!("Running on the multi-thread scheduler!");
});
```

### 当前线程调度器

当前线程调度器在创建运行时的线程上执行所有任务。它是一个单线程执行器。这种调度器适用于需要运行异步代码但不需要多线程的场景，例如在资源受限的环境中，或将异步运行时嵌入到更大型的现有应用程序中。

要使用它，您必须使用 `Builder` 来构造它。

```rust
use tokio::runtime;

// 创建一个单线程运行时。
let rt = runtime::Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();

// 在当前线程上运行该运行时。
rt.block_on(async {
    println!("Running on the current-thread scheduler!");
});
```

## 创建和配置运行时

您可以使用默认设置创建运行时，也可以使用 `Builder` 对其进行广泛的自定义。

### 简单方式：`#[tokio::main]`

对于大多数应用程序，`#[tokio::main]` 属性宏是启动运行时的最简单方法。它将一个 `async fn main()` 转换为一个同步的 `fn main()`，该函数会初始化一个 `Runtime` 并执行该 future。

```rust,no_run
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}
```

### 手动方式：`Runtime::new()` 和 `Builder`

为了获得更多控制权，您可以手动构建一个运行时。当您需要配置线程数、启用特定驱动程序或设置生命周期钩子时，这是必需的。

`Builder` 提供了一个流式 API 用于配置。请记住，在使用 `Builder` 时，用于 I/O 和时间的资源驱动程序默认是禁用的，必须使用 `enable_io()`、`enable_time()` 或便捷的 `enable_all()` 等方法显式启用。

**常用配置选项**

| Method                   | Description                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| `worker_threads(usize)`  | 为多线程调度器设置工作线程的数量。                                                       |
| `max_blocking_threads(usize)` | 为阻塞操作设置线程池中的最大线程数。                                                     |
| `thread_name(String)`    | 为派生的工作线程设置自定义名称，便于调试。                                               |
| `enable_all()`           | 同时启用 I/O 和时间驱动程序。                                                            |
| `enable_io()`            | 为网络、文件系统等启用 I/O 驱动程序。                                                    |
| `enable_time()`          | 为休眠、间隔和超时启用时间驱动程序。                                                     |
| `thread_stack_size(usize)` | 设置工作线程的堆栈大小。                                                                 |
| `thread_keep_alive(Duration)` | 为阻塞池中的空闲线程设置自定义超时时间。                                                 |

**示例：构建自定义运行时**

```rust
use tokio::runtime::Builder;
use std::time::Duration;

fn main() {
    // 构建一个自定义运行时
    let runtime = Builder::new_multi_thread()
        .worker_threads(4) // 使用 4 个工作线程
        .thread_name("my-tokio-worker")
        .thread_keep_alive(Duration::from_millis(100))
        .enable_all() // 启用 I/O 和时间驱动程序
        .build()
        .unwrap();

    // 使用该运行时阻塞等待主 future
    runtime.block_on(async {
        println!("Running on a custom-configured runtime!");
    });
}
```

## 运行时关闭

当 `Runtime` 实例被丢弃（dropped）时，运行时会关闭。在关闭期间，运行时会尝试优雅地停止所有派生的工作。丢弃 `Runtime` 的线程将阻塞，直到关闭过程完成。

-   **对于异步任务：**任务会运行到下一个让出点（yield point）（`.await`），此时它们将被丢弃。
-   **对于阻塞任务：**使用 `spawn_blocking` 派生的任务会运行至完成。

因为等待所有工作完成可能需要不确定的时间，Tokio 提供了其他关闭方法：

-   `shutdown_timeout(duration)`: 等待指定的时间以停止工作。如果达到超时，剩余的工作和线程将被泄露，函数返回。
-   `shutdown_background()`: 启动关闭而不等待。这等效于 `shutdown_timeout(Duration::from_nanos(0))`，并且在异步上下文中丢弃运行时很有用。

## 深入阅读

本页提供了 Tokio 运行时的概念性概述。有关详细的配置选项和方法，请参阅 [Runtime API 参考](./api-runtime.md)。