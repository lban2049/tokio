# 快速入门

本指南将引导你使用 Tokio 设置一个新项目，并构建一个简单的异步 TCP 回声服务器。读完本指南后，你将拥有一个可以处理多个并发连接的应用程序。

## 1. 添加 Tokio 作为依赖项

首先，创建一个新的 Rust 项目并将 Tokio 添加到你的依赖项中。最简单的入门方法是在 `Cargo.toml` 文件中通过 `full` 功能标志启用所有功能。这可以确保在你构建应用程序时，可能需要的所有 API 都可用。

**Cargo.toml**
```toml
tokio = { version = "1", features = ["full"] }
```

## 2. 编写一个异步 TCP 回声服务器

接下来，我们来为服务器编写代码。此应用程序将在 `127.0.0.1:8080` 上监听传入的 TCP 连接。对于每个连接，它会从套接字读取数据，然后将相同的数据写回给客户端。

**main.rs**
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

            // 在一个循环中，从套接字读取数据并将数据写回。
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

## 3. 运行应用程序

代码准备就绪后，你可以使用 Cargo 运行服务器：

```sh
cargo run
```

要测试该服务器，你可以使用 `netcat` 或 `telnet` 等工具来连接。打开一个新的终端窗口并运行：

```sh
telnet 127.0.0.1 8080
```

你在 `telnet` 会话中输入的任何内容都将由服务器回显。

## 4. 工作原理

让我们简要分析一下代码的工作内容：

-   `#[tokio::main]`: 这是一个宏，它将 `async fn main()` 转换成一个同步的 `main` 函数，该函数会初始化一个 Tokio 运行时并执行异步代码。
-   `TcpListener::bind("127.0.0.1:8080").await?`: 我们创建一个 `TcpListener` 来监听传入的连接。这是一个异步操作，因此我们使用 `.await` 等待它完成。
-   `listener.accept().await?`: `accept` 方法等待新的连接。当连接建立时，它会返回一个新的 `TcpStream`（即我们的 `socket`）和对端的地址。`loop` 循环确保服务器可以持续不断地接受新连接。
-   `tokio::spawn(async move { ... })`: 对于每个传入的连接，我们都会生成一个新的异步任务。这使得服务器能够并发处理多个连接。主任务可以立即返回以接受新连接，而生成的任务则处理其特定的连接。
-   `socket.read(&mut buf).await` 和 `socket.write_all(...).await`: 这些是异步 I/O 操作。它们从套接字读取数据到缓冲区，并将缓冲区的内容写回套接字。`.await` 关键字会暂停任务直到操作完成，而不会阻塞整个线程，从而允许其他任务运行。

## 后续步骤

你已经成功使用 Tokio 构建了你的第一个异步应用程序！为了更深入地理解刚刚用到的概念，我们建议你浏览 [核心概念](./concepts.md) 文档。