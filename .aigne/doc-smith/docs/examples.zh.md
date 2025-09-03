# 示例

本节提供了一系列可运行的代码示例，用于演示 Tokio 的各种功能和用例。这些示例切合实用，可作为您开发自己应用程序的起点。

<x-cards data-columns="1">
  <x-card data-title="TCP Echo Server" data-icon="lucide:server">
    这是一个异步 TCP 服务器的基础示例，它会回显从客户端接收到的任何数据。这是理解基本网络 I/O 的绝佳起点。
  </x-card>
  <x-card data-title="Mini-Redis" data-icon="lucide:database">
    这是一个更大型的“真实世界”示例，实现了一个简化的 Redis 服务器。它演示了如何构建完整的应用程序、管理共享状态以及处理客户端命令。
  </x-card>
</x-cards>

## TCP Echo 服务器

简单的 TCP echo 服务器是演示异步 I/O 的经典方法。服务器监听一个套接字，接受传入的连接，并为每个连接读取数据，然后将数据写回同一个套接字。

### 依赖

要运行此示例，您需要在 `Cargo.toml` 文件中启用必要的功能。使用 `full` 功能标志是上手最简单的方式。

```toml
tokio = { version = "1", features = ["full"] }
```

### 服务器代码

以下代码实现了完整的 echo 服务器：

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

            // 循环中，从套接字读取数据并将数据写回。
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

### 工作原理

1.  **`TcpListener::bind`**：服务器首先将 `TcpListener` 绑定到本地地址（`127.0.0.1:8080`）。`.await` 关键字会暂停执行，直到监听器成功绑定。
2.  **`listener.accept()`**：服务器进入一个循环，调用 `listener.accept().await` 等待传入的连接。当客户端连接时，`accept` 会返回一个新的 `TcpSocket` 和客户端的地址。
3.  **`tokio::spawn`**：为了并发处理多个客户端，会为每个已接受的连接生成一个新任务。`socket` 会被移入这个新任务中。
4.  **读/写循环**：在生成的任务内部，一个循环会持续从套接字读取数据到缓冲区。如果读取成功（`Ok(n)` 且 `n > 0`），则会将相同的数据（`&buf[0..n]`）写回套接字。如果客户端关闭连接，`read` 会返回 `Ok(0)`，任务随之终止。

## 高级示例：Mini-Redis

如需更详尽的真实世界示例，请参阅 [mini-redis 代码库](https://github.com/tokio-rs/mini-redis/)。该项目是使用 Tokio 构建的 Redis 服务器和客户端的异步、简化实现。

它是学习如何构建更大型 Tokio 应用程序的绝佳资源，并演示了以下概念：

- 管理跨任务的共享可变状态。
- 帧处理（Framing），即将字节流解析为消息序列的过程。
- 优雅关闭。
- 实现协议的客户端和服务器端。

## 更多示例

更多涵盖 Tokio 各种功能的示例，可以在 [Tokio GitHub 代码库的 examples 目录](https://github.com/tokio-rs/tokio/tree/master/examples)中找到。这些示例简明地演示了特定的 API，是学习的宝贵资源。