# 运行时

Tokio 运行时是驱动异步 Rust 应用程序的引擎。虽然 Rust 中的异步代码提供了非阻塞操作的语法，但它需要一个运行时来实际执行 future、管理任务和处理 I/O 事件。Tokio 运行时为构建健壮、高性能的网络应用程序提供了所有必要的服务。

从高层次来看，运行时捆绑了几个关键组件：

- 一个**I/O 事件循环**，通常称为驱动，它与操作系统的事件队列（如 `epoll`、`kqueue` 或 `IOCP`）交互。
- 一个**任务调度器**，用于管理大量轻量级异步任务的执行。
- 一个**计时器**，用于调度将来要运行的工作，从而实现超时和间隔等功能。
- 一个专用的**线程池**，用于分流阻塞的、CPU 密集型操作，以防止它们拖慢事件循环。

对于大多数应用程序，`#[tokio::main]` 宏是启动运行时的最简单方法。然而，Tokio 还提供了一个功能强大的 `Builder` 用于进行细粒度配置。本节将探讨运行时的架构、配置和执行模型。

### 运行时架构

Tokio 运行时的各个组件协同工作，以高效地执行你的异步代码。

```d2
direction: down

"应用程序代码" {
  "async fn main() {}"
  "tokio::spawn(...)"
}

"Tokio 运行时" {
  style.fill: "#f0f8ff"
  "调度器（多线程或当前线程）"
  "驱动" : {
    "I/O 轮询器 (epoll, kqueue 等)"
    "计时器"
  }
  "阻塞线程池"
}

"应用程序代码" -> "Tokio 运行时"."调度器": "生成任务"
"Tokio 运行时"."调度器" -> "Tokio 运行时"."驱动": "轮询事件"
"Tokio 运行时"."调度器" -> "Tokio 运行时"."阻塞线程池": "委托阻塞工作"
"Tokio 运行时"."驱动" -> "Tokio 运行时"."调度器": "在 I/O/时间事件上唤醒任务"

```

## 用法

与 Tokio 运行时交互主要有两种方式：为简单起见使用 `#[tokio::main]` 宏，或者为了更强的控制力而手动构建和管理 `Runtime` 实例。

### 使用 #[tokio::main] 的简单用法

最简单的入门方法是为 `main` 函数添加注解。这个宏会创建一个默认的多线程运行时，启动它，并在其中运行 `async` main 函数。

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];
            loop {
                let n = match socket.read(&mut buf).await {
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

### 使用 Runtime::new() 的手动用法

为了获得更多控制权，你可以自己创建一个 `Runtime` 实例。`block_on` 方法会启动运行时并阻塞当前线程，直到提供的 future 完成。

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建运行时
    let rt = Runtime::new()?;

    // 生成根任务
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
        // ... 与上面相同的逻辑 ...
    });

    Ok(())
}
```

## 调度器配置

Tokio 提供了两种调度器类型，每种都适用于不同的用例。

<x-cards data-columns="2">
  <x-card data-title="多线程调度器" data-icon="lucide:cpu">
    这是默认的调度器。它在一组工作线程（通常每个 CPU 核心一个）中使用工作窃取策略。它非常适合大多数应用程序，特别是那些具有高并发和 I/O 密集型工作负载的应用程序。
  </x-card>
  <x-card data-title="当前线程调度器" data-icon="lucide:user-round">
    该调度器在创建它的单个线程上运行所有任务。它比多线程调度器更轻量，适用于只需要一个线程的场景，或将 Tokio 嵌入到现有的单线程应用程序中。
  </x-card>
</x-cards>

你可以使用 `Builder` 来选择调度器。

```rust
use tokio::runtime::Builder;

// 创建一个多线程运行时
let multi_thread_rt = Builder::new_multi_thread()
    .enable_all()
    .build()
    .unwrap();

// 创建一个单线程运行时
let current_thread_rt = Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();
```

## 使用 Builder 配置运行时

`tokio::runtime::Builder` 提供了一种灵活的方式，在创建运行时之前配置其各个方面。你可以链式调用方法来自定义线程数、驱动和调度器行为。

| Method | Description |
|---|---|
| `new_multi_thread()` | 为多线程、工作窃取调度器创建一个构建器。 |
| `new_current_thread()` | 为单线程调度器创建一个构建器。 |
| `enable_all()` | 同时启用 I/O 和时间驱动。 |
| `enable_io()` | 启用用于网络、文件系统等的 I/O 驱动。 |
| `enable_time()` | 启用用于休眠、间隔和超时的计时器驱动。 |
| `worker_threads(usize)` | 为多线程调度器设置工作线程的数量。 |
| `max_blocking_threads(usize)` | 为阻塞任务池设置最大线程数。 |
| `thread_name(String)` | 为生成的工作线程设置名称。 |
| `thread_keep_alive(Duration)`| 为阻塞池中的线程设置空闲超时时间。 |

以下是一个自定义配置的示例：

```rust
use tokio::runtime::Builder;
use std::time::Duration;

let runtime = Builder::new_multi_thread()
    .worker_threads(4)
    .thread_name("my-tokio-worker")
    .thread_stack_size(3 * 1024 * 1024)
    .thread_keep_alive(Duration::from_secs(60))
    .enable_all()
    .build()
    .unwrap();

runtime.block_on(async {
    println!("来自自定义配置运行时的问候！");
});
```

## 执行行为

Tokio 的调度器旨在实现公平和高效。虽然确切的调度算法是实现细节，但了解其高层行为很重要。

- **公平性**：Tokio 保证，如果任务总数不会无限增长，并且没有任务阻塞工作线程，那么每个被唤醒的任务最终都会被调度运行。
- **虚假唤醒**：即使任务的 waker 没有被调用，它也可能偶尔被轮询。你的代码不应依赖于唤醒的绝对精确性。

### 调度器详情
- **多线程**：每个工作线程都有自己的本地任务队列。当一个工作线程的本地队列为空时，它会首先检查全局队列中是否有新任务，然后尝试从其他工作线程的本地队列中“窃取”任务。这种工作窃取方法有助于确保所有线程都保持繁忙，并且工作被均匀分配。
- **当前线程**：该调度器使用一个更简单的模型，包含一个本地队列和一个全局队列。它优先处理本地队列中的任务以最小化同步开销，但会定期轮询全局队列以确保公平性。

## 关闭运行时

当 `Runtime` 值被丢弃时，运行时会关闭。在关闭期间，运行时会尝试优雅地停止所有已生成的工作。

- **异步任务** (`tokio::spawn`)：这些任务会运行到下一个屈服点 (`.await`)，然后被丢弃。不保证它们会运行到完成。
- **阻塞任务** (`spawn_blocking`)：这些任务会一直运行直到完成。

`drop` 的实现会阻塞当前线程，直到所有工作都停止，这可能会无限期地持续下去。对于不能永远阻塞的情况，你可以使用 `shutdown_timeout(duration)` 或 `shutdown_background()`。

```rust
use tokio::runtime::Runtime;
use std::time::Duration;

let runtime = Runtime::new().unwrap();

runtime.spawn(async {
    // 一些长时间运行的任务
});

// 关闭运行时，最多等待 100 毫秒让任务停止。
runtime.shutdown_timeout(Duration::from_millis(100));
```

---

既然你已经了解了运行时的核心概念，你可以在 [任务与调度](./concepts-tasks.md) 部分探索如何管理单个工作单元，或在 [API 参考](./api-runtime.md) 中深入了解详细的配置选项。