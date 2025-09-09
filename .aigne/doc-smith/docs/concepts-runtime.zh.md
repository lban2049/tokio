# 运行时

Tokio 运行时是驱动 Rust 异步应用程序的引擎。它提供了执行异步任务、管理 I/O 和处理基于时间的事件所需的基本服务。虽然你可以在 Rust 中编写 `async fn`，但你需要一个运行时来实际运行它们。

从高层次来看，Tokio 运行时捆绑了以下几个部分：

*   一个 **I/O 事件循环**，通常称为驱动程序，它与操作系统的 I/O API（如 epoll、kqueue 或 IOCP）进行交互。
*   一个 **任务调度器**，它在少数几个操作系统线程上并发管理许多轻量级、非阻塞任务的执行。
*   一个用于安排工作在未来某个时间运行的 **计时器**，为 `tokio::time::sleep` 和超时等功能提供支持。

大多数用户通过 `#[tokio::main]` 宏与运行时进行交互，该宏会设置一个默认的运行时并执行被注解的 `async fn`。若想进行更高级的控制，你可以手动构建和管理一个 `Runtime` 实例。

## 使用模式

有两种主要的方式来开始使用 Tokio 运行时。

### `#[tokio::main]` 宏

对于大多数应用程序来说，最简单的入门方式是使用 `#[tokio::main]` 属性宏。这个宏会将一个 `async fn main()` 转换成一个同步的 `fn main()`，后者会初始化一个 `Runtime` 实例并运行 future 直至完成。

```rust main.rs icon=logos:rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // 在一个循环中，从套接字读取数据并将数据写回。
            loop {
                let n = match socket.read(&mut buf).await {
                    // 套接字已关闭
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        println!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    println!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

在一个正在运行的运行时上下文中，你可以使用 [`tokio::spawn`](./api-task.md) 来派生额外的任务。

### 手动管理运行时

如果你需要对运行时的配置进行更多控制，可以使用 [`Runtime`](./api-runtime.md) 结构体来自己构建和管理它。`block_on` 方法是入口点，它会运行一个 future 直到完成，并在此期间阻塞当前线程。

```rust main.rs icon=logos:rust
use tokio::runtime::Runtime;
use tokio::net::TcpListener;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建运行时
    let rt  = Runtime::new()?;

    // 派生根任务
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await?;
        println!("Listening on: {}", listener.local_addr()?);
        // ... 应用程序逻辑 ...
        Ok(())
    })
}
```

## 调度器配置

Tokio 提供了不同的调度策略以适应各种应用程序的需求。你可以使用 [`Builder`](./api-runtime.md) 来选择一个调度器。

<x-cards>
  <x-card data-title="多线程调度器" data-icon="lucide:cpu">
    这是默认的调度器。它使用一个带有工作窃取（work-stealing）策略的线程池，在多个 CPU 核心上执行任务。对于大多数应用程序，尤其是处理许多并发连接的网络服务，这是最佳选择。
  </x-card>
  <x-card data-title="当前线程调度器" data-icon="lucide:user">
    该调度器提供一个单线程执行器。所有任务都在启动运行时的同一线程上创建和执行。它适用于需要运行异步代码但不需要多线程的场景，或者在与 `LocalSet` 结合处理 `!Send` future 时非常有用。
  </x-card>
</x-cards>

## 使用 `Builder` 自定义运行时

[`Builder`](./api-runtime.md) 提供了一个流式 API，用于在创建运行时之前配置其各个方面。

以下是一些最常见的配置选项：

| Method | Description |
|---|---|
| `worker_threads(n)` | 为多线程调度器设置工作线程的数量。默认为 CPU 核心数。 |
| `max_blocking_threads(n)` | 为阻塞池中的线程设置上限，用于 `spawn_blocking`。默认为 512。 |
| `enable_all()` | 同时启用 I/O 和时间驱动程序。在手动构建运行时以使用网络或时间功能时是必需的。 |
| `enable_io()` | 启用用于网络、文件系统等的 I/O 驱动程序。 |
| `enable_time()` | 启用用于休眠、间隔和超时的时钟驱动程序。 |
| `thread_name("name")` | 为工作线程设置自定义名称，这对于调试很有用。 |
| `thread_stack_size(bytes)` | 设置工作线程的栈大小。 |

以下是构建自定义多线程运行时的示例：

```rust builder_example.rs icon=logos:rust
use tokio::runtime::Builder;

fn main() {
    // 构建一个拥有 4 个工作线程、一个自定义线程名、
    // 并同时启用 I/O 和时间驱动程序的运行时。
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("my-tokio-worker")
        .thread_stack_size(3 * 1024 * 1024)
        .enable_all()
        .build()
        .unwrap();

    runtime.block_on(async {
        println!("Hello from the custom runtime!");
    });
}
```

## 运行时关闭

当 `Runtime` 实例被丢弃（drop）时，运行时就会关闭。关闭过程会阻塞线程，直到所有派生的工作都停止。

- 使用 `spawn` 派生的**异步任务**会一直运行到它们的下一个让出点（yield point）（`.await`），此时它们将被丢弃。
- 使用 `spawn_blocking` 派生的**阻塞任务**将运行至完成。

由于默认的 `drop` 实现可能会无限期阻塞，因此对于需要更多控制权的场景，Tokio 提供了两种替代的关闭方法：

- `shutdown_timeout(duration)`: 等待指定的时间以让工作完成。如果达到超时时间，线程和任务会被泄漏，但调用线程会被解除阻塞。
- `shutdown_background()`: 是 `shutdown_timeout` 零时长的简写。它会启动关闭过程并立即返回，而不会等待任务停止。

## 执行行为与公平性

Tokio 的调度器提供公平性保证：如果任务总数不会无限增长，并且没有任务阻塞线程，那么每个任务都保证最终会被调度。这可以防止任务饥饿。

然而，具体的执行顺序是不保证的。调度器可能会比其他任务更频繁地轮询某些任务。它也可能执行*虚假唤醒*（spurious wakeups），即即使任务的 `Waker` 没有被调用，任务也被轮询。你的 `Future` 实现应该对这种行为具有鲁棒性。

多线程调度器采用工作窃取（work-stealing）策略。每个工作线程都有一个本地的任务队列。当一个工作线程的队列为空时，它会尝试从其他工作线程的队列中窃取任务，以确保所有线程都保持繁忙。
