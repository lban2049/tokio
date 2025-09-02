# 示例

本节提供了一系列可运行的代码示例，帮助你理解如何使用 Tokio 的各种功能。这些示例旨在做到实用且易于理解，展示了常见的用例。

如需更全面的示例集，可以浏览 [GitHub 上的 Tokio 官方示例目录](https://github.com/tokio-rs/tokio/tree/master/examples)。

## TCP 回声服务器

一个简单而完整的 TCP 回声服务器，它监听传入的连接，并回显接收到的任何数据。这是构建网络应用程序的绝佳起点。

首先，确保你的 `Cargo.toml` 配置包含了必要的 Tokio 功能。建议使用 `full` 功能，以便轻松入门。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

以下是服务器的实现：

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

            // In a loop, read data from the socket and write the data back.
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

                // Write the data back
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

### 工作原理

1.  **`TcpListener::bind(...)`**：将一个新的 TCP 监听器绑定到指定地址。`.await` 会暂停执行，直到监听器成功绑定。
2.  **`listener.accept().await`**：在无限循环中，服务器等待新的传入连接。执行会暂停，直到建立连接。
3.  **`tokio::spawn(...)`**：对于每个新连接，都会生成一个新的异步任务。这使得服务器能够并发处理多个客户端，而不会阻塞主循环。
4.  **`socket.read(...)` 和 `socket.write_all(...)`**：在生成的任务内部，服务器重复地从套接字读取数据到缓冲区，然后将完全相同的数据写回套接字，从而实现“回声”效果。

## 处理阻塞操作

Tokio 的协作式调度器要求任务让出控制权，以便其他任务可以运行。然而，一些操作本质上是阻塞的，例如繁重的 CPU 计算或传统的同步文件 I/O。为了在不阻塞运行时的情况下处理这些操作，你应该使用 `tokio::task::spawn_blocking`。

此函数会将阻塞操作移至专用的线程池，从而允许主运行时继续处理其他异步任务。

```rust
#[tokio::main]
async fn main() {
    // This is running on a core thread.

    let blocking_task = tokio::task::spawn_blocking(|| {
        // This is running on a blocking thread.
        // Blocking here is ok.
        // For example, a heavy computation.
        std::thread::sleep(std::time::Duration::from_secs(1));
        "done"
    });

    // We can wait for the blocking task like this:
    // If the blocking task panics, the unwrap below will propagate the
    // panic.
    let result = blocking_task.await.unwrap();
    println!("Blocking task finished: {}", result);
}
```

### 工作原理

1.  传递给 `spawn_blocking` 的闭包在 Tokio 阻塞线程池中的一个单独线程上执行。
2.  这可以防止可能长时间运行的操作停止主调度器上其他异步任务的进程。
3.  主任务可以 `.await` `spawn_blocking` 返回的 `JoinHandle`，以便在计算完成后接收结果，而不会阻塞执行器。

## 更高级的示例

如需查看更大、更贴近实际的示例来演示如何使用 Tokio 构建完整的应用程序，请查阅以下资源。

<x-cards data-columns="2">
  <x-card data-title="Mini-Redis" data-icon="lucide:database" data-href="https://github.com/tokio-rs/mini-redis/">
    一个完整的异步 Redis 客户端和服务器。这是一个使用 Tokio 构建的真实世界应用的绝佳示例，展示了通道、共享状态和优雅关闭。
  </x-card>
  <x-card data-title="Official Examples" data-icon="lucide:book-open" data-href="https://github.com/tokio-rs/tokio/tree/master/examples">
    Tokio 官方仓库包含了各种各样的小型示例，每个示例都侧重于一个特定的功能，如网络、通道或计时器。
  </x-card>
</x-cards>