# 示例

本节提供了一系列可运行的示例，以帮助你了解如何使用 Tokio 的各种功能。这些示例旨在以最少的设置即可复制和运行。如需更全面的示例集，你可以浏览 [GitHub 上的 Tokio 仓库](https://github.com/tokio-rs/tokio/tree/master/examples)。

## TCP 回声服务器

这是一个基本的 TCP 回声服务器，它监听传入的连接，并将其接收到的任何数据发回。这是一个经典的示例，用于演示异步 I/O 和任务管理。

### 设置

首先，将必要的依赖项添加到你的 `Cargo.toml` 文件中。`full` 功能标志启用了所有公共的 Tokio API，这对于入门非常方便。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### 代码

现在，你可以在你的 `main.rs` 文件中使用以下代码：

```rust,no_run
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // 在循环中，从套接字读取数据并将其写回。
            loop {
                let n = match socket.read(&mut buf).await {
                    // 套接字已关闭
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // 将数据写回
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

此代码在 8080 端口上设置一个监听器。对于每个传入的连接，它会生成一个新的异步任务来处理通信。该任务将数据读入缓冲区，并将其写回到同一个套接字，直到连接关闭。

## 处理阻塞操作

异步任务不应直接执行阻塞操作，因为这会停止同一线程上其他任务的进展。对于阻塞或 CPU 密集型工作，请使用 `spawn_blocking` 函数将工作移至专用的线程池。

```rust,no_run
#[tokio::main]
async fn main() {
    // 这在核心线程上运行。

    let blocking_task = tokio::task::spawn_blocking(|| {
        // 这在阻塞线程上运行。
        // 在这里阻塞是可以的。
        // 例如，计算密集型任务或同步文件读取。
        "done"
    });

    // 我们可以像这样等待阻塞任务：
    // 如果阻塞任务发生 panic，下面的 unwrap 将传播该
    // panic。
    let result = blocking_task.await.unwrap();
    println!("Blocking task finished with result: {}", result);
}
```

`spawn_blocking` 函数接受一个闭包，并在 Tokio 运行时管理的独立线程池上执行它。这可以防止主异步调度程序被阻塞。对返回的 `JoinHandle` 进行 `await` 操作，允许异步任务等待阻塞操作完成而不会暂停线程。

## 进一步探索

对于更高级和真实世界的用例，以下资源提供了丰富的示例。

<x-cards data-columns="2">
  <x-card data-title="官方 Tokio 示例" data-icon="lucide:github" data-href="https://github.com/tokio-rs/tokio/tree/master/examples" data-cta="View on GitHub">
    官方 Tokio 仓库中的综合示例集合，涵盖了各种模块和功能。
  </x-card>
  <x-card data-title="Mini-Redis" data-icon="lucide:database" data-href="https://github.com/tokio-rs/mini-redis" data-cta="View on GitHub">
    一个更大、更“真实世界”的客户端-服务器应用程序示例，该应用使用 Tokio 构建，展示了更复杂的应用程序结构。
  </x-card>
</x-cards>