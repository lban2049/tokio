# 运行时

异步 Rust 应用程序需要一个运行时来为其提供动力。Tokio 运行时为执行异步任务提供了必要的服务，包括一个 I/O 事件循环（驱动程序）、一个任务调度器和一个计时器。

概括来说，运行时负责：

- **一个 I/O 事件循环**，称为驱动程序，它驱动 I/O 资源并将 I/O 事件分派给依赖它们的任务。
- **一个调度器**，用于执行使用这些 I/O 资源的任务。
- **一个计时器**，用于调度在设定的时间段后运行的工作。

`tokio::runtime::Runtime` 类型将所有这些服务捆绑在一起，允许它们被一同启动、配置和关闭。虽然你可以手动配置 `Runtime`，但大多数用户会从 `#[tokio::main]` 宏入手，该宏会自动创建和管理一个运行时。

## 使用 #[tokio::main] 快速入门

对于大多数应用程序而言，`#[tokio::main]` 属性宏是启动 Tokio 运行时的最简单方式。它会使用默认设置建立一个多线程运行时，并在该运行时上执行被装饰的 `async fn main` 函数。

```rust 一个简单的 TCP 回显服务器 icon=logos:rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write it back.
            loop {
                let n = match socket.read(&mut buf).await {
                    // socket closed
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

## 手动管理运行时

为获得更多控制权，你可以直接创建和管理 `Runtime` 实例。当需要自定义其配置或将 Tokio 运行时嵌入到更大型的同步应用程序中时，这种方式非常有用。

`Runtime::block_on` 方法是在运行时上执行异步任务的入口点。它会阻塞当前线程，直到提供的 future 完成为止。

下面是同一个 TCP 回显服务器，但使用了手动配置的运行时：

```rust 手动创建运行时 icon=logos:rust
use tokio::runtime::Runtime;
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt  = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await?;

        loop {
            let (mut socket, _) = listener.accept().await?;

            tokio::spawn(async move {
                let mut buf = [0; 1024];

                // In a loop, read data from the socket and write it back.
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
    })
}
```

## 运行时调度器

Tokio 提供了不同的任务调度策略以满足各种应用程序的需求。你可以使用 `Builder` API 或 `#[tokio::main]` 宏的 `flavor` 参数来选择调度器。

### 多线程调度器

多线程调度器使用线程池和工作窃取 (work-stealing) 策略来并发执行任务。默认情况下，它会为每个 CPU 核心生成一个工作线程，使其成为大多数应用程序的理想选择。

这是默认的调度器。你可以像这样手动创建它：

```rust 创建多线程运行时 icon=logos:rust
use tokio::runtime::Builder;

let runtime = Builder::new_multi_thread()
    .worker_threads(4) // 配置 4 个工作线程
    .enable_all()      // 启用 I/O 和时间驱动程序
    .build()
    .unwrap();
```

### 当前线程调度器

当前线程调度器在创建运行时的那个线程上执行所有任务。它是一个单线程执行器，对于有特定线程要求的应用程序或将 Tokio 嵌入到现有单线程事件循环中的场景非常有用。

```rust 创建当前线程运行时 icon=logos:rust
use tokio::runtime::Builder;

let runtime = Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();
```

## 使用 Builder 进行配置

`tokio::runtime::Builder` 提供了一个全面的 API 用于微调运行时。你可以配置工作线程、阻塞线程、线程名称、栈大小，以及启用或禁用特定的运行时驱动程序。

```rust 自定义运行时配置 icon=logos:rust
use tokio::runtime::Builder;
use std::time::Duration;

let runtime = Builder::new_multi_thread()
    .worker_threads(4)
    .max_blocking_threads(512)
    .thread_name("my-tokio-worker")
    .thread_stack_size(3 * 1024 * 1024)
    .thread_keep_alive(Duration::from_secs(60))
    .enable_all()
    .build()
    .unwrap();
```

关键配置选项包括：

- `worker_threads(usize)`: 设置多线程调度器的工作线程数量。
- `max_blocking_threads(usize)`: 设置阻塞池中的最大线程数，用于通过 `spawn_blocking` 运行同步代码。
- `thread_name(&str)` or `thread_name_fn(Fn)`: 为工作线程设置静态或动态生成的名称。
- `thread_stack_size(usize)`: 设置工作线程的栈大小。
- `enable_io()`: 启用 I/O 驱动程序（用于网络、文件系统等）。
- `enable_time()`: 启用时间驱动程序（用于 `sleep`、`interval`、`timeout`）。
- `enable_all()`: 启用 I/O 和时间驱动程序的简写方式。

## 运行时句柄

`Handle` 是一个轻量级、可克隆的 Tokio 运行时引用。它允许你从任何上下文中与运行时交互，例如生成任务，即使是在非 Tokio 管理的线程中。

你可以通过两种主要方式获取句柄：

1.  **From an existing `Runtime`**: `let handle = runtime.handle();`
2.  **From within a task**: `let handle = Handle::current();`

使用 `Handle` 是授予应用程序其他部分访问运行时权限的首选方式，而无需让它们拥有 `Runtime` 本身的所有权。与 `Arc<Runtime>` 不同，`Handle` 不会阻止运行时关闭。

```rust 从另一个线程生成任务 icon=logos:rust
use tokio::runtime::{Runtime, Handle};
use std::thread;
use std::time::Duration;

let runtime = Runtime::new().unwrap();
let handle = runtime.handle().clone();

let job = thread::spawn(move || {
    // 使用句柄在这个新线程中运行异步代码。
    handle.block_on(async {
        println!("Hello from a blocking context!");
    });
});

job.join().unwrap();
```

## 核心操作

<x-cards>
  <x-card data-title="block_on" data-icon="lucide:log-in">
    运行时的入口点。它接收一个 future 并阻塞当前线程，直到该 future 完成，然后返回其输出。
  </x-card>
  <x-card data-title="spawn" data-icon="lucide:send">
    生成一个新的异步任务，并为其返回一个 `JoinHandle`。该任务与运行时上的其他任务并发运行。
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:brick-wall">
    在专用的线程池上运行阻塞函数，以防止其阻塞异步调度器。
  </x-card>
  <x-card data-title="enter" data-icon="lucide:arrow-right-left">
    进入运行时上下文。这允许你使用像 `tokio::spawn` 这样的上下文感知函数，或从非异步代码中创建 I/O 类型。
  </x-card>
</x-cards>

## 关闭

当 `Runtime` 值被丢弃 (dropped) 时，运行时会关闭。此过程涉及停止所有工作线程并尝试优雅地终止已生成的任务。默认情况下，drop 实现将无限期等待通过 `spawn_blocking` 生成的所有阻塞任务完成。

对于需要更多控制权的场景，`Runtime` 提供了两种显式的关闭方法：

- `shutdown_timeout(duration)`: 关闭运行时，最多等待指定的时间让所有工作停止。任何未及时停止的线程都会被泄露。
- `shutdown_background()`: 立即关闭运行时，不等待已生成的工作停止。这对于从另一个异步上下文中丢弃运行时很有用，但可能导致资源泄露。

```rust 定时关闭 icon=logos:rust
use tokio::runtime::Runtime;
use std::time::Duration;

let runtime = Runtime::new().unwrap();

runtime.spawn(async {
    // 一些长时间运行的任务
    tokio::time::sleep(Duration::from_secs(10)).await;
});

// 即使任务尚未完成，运行时也将在 100 毫秒后关闭。
runtime.shutdown_timeout(Duration::from_millis(100));
```

在对运行时有了扎实的理解后，你就可以管理更复杂的异步操作了。要了解有关创建和管理并发任务的更多信息，请继续阅读下一部分。

---

接下来，让我们更深入地探讨如何创建和管理任务。

<x-card data-title="下一步：生成和管理任务" data-icon="lucide:arrow-right" data-href="/tasks-scheduling/spawning" data-cta="阅读更多">
  学习如何使用 `spawn` 和 `JoinHandle` 创建和管理并发任务。
</x-card>