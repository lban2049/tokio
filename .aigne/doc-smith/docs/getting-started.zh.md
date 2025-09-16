# 入门指南

本指南将引导你在几分钟内从零开始运行一个 Tokio 应用程序。我们将介绍如何设置项目以及如何构建一个可并发处理多个连接的简单 TCP 回显服务器。

## 项目设置

首先，你需要在 `Cargo.toml` 文件中添加 `tokio` crate 作为依赖项。

在编写应用程序时，我们建议通过 `full` 标志启用所有功能。这可以确保你在构建过程中能够访问 Tokio 的所有 API，而不会遇到障碍。

将以下内容添加到你的 `Cargo.toml` 中：

```toml Cargo.toml icon=logos:rust
[dependencies]
tokio = { version = "1", features = ["full"] }
```

## 一个基本的 TCP 回显服务器

让我们构建一个简单的服务器，它能接受传入的 TCP 连接，并将其接收到的任何数据发回。这是一个经典的例子，展示了 Tokio 的核心功能：异步 I/O 和并发任务管理。

创建一个新文件 `src/main.rs` 并添加以下代码：

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

## 代码解读

让我们来分析一下代码中发生了什么：

1.  **`#[tokio::main]`**：这是一个宏，用于设置 Tokio 运行时。它将 `async fn main()` 转换为一个同步的 `main` 函数，该函数会初始化运行时并执行其中的异步代码。

2.  **`TcpListener::bind("127.0.0.1:8080").await?`**：我们创建一个 `TcpListener` 并将其绑定到 8080 端口。这是一个异步操作，因此我们使用 `.await` 等待其完成。

3.  **`listener.accept().await?`**：`loop` 循环持续调用 `accept()`。每次调用 `accept()` 都会异步等待一个新的传入连接。当客户端连接时，它会返回一个新的 `TcpStream`（即我们的 `socket`）和客户端的地址。

4.  **`tokio::spawn(async move { ... })`**：为了并发处理多个客户端，我们为每个传入的连接生成一个新的异步任务。`tokio::spawn` 函数接受一个 `async` 代码块，并在 Tokio 运行时上运行它，而不会阻塞主循环。这使得 `loop` 循环可以立即返回并等待下一个连接。

5.  **`socket.read(...)` 和 `socket.write_all(...)`**：在生成的任务内部，我们重复地从客户端读取数据到缓冲区，然后将相同的数据写回客户端。这就是“回显”逻辑。这两个都是异步操作，因此它们都用 `.await` 标记。

## 运行服务器

现在，你可以从终端运行该应用程序：

```sh
cargo run
```

服务器将启动并监听 `127.0.0.1:8080` 上的连接。

要进行测试，请打开一个新的终端窗口，并使用像 `telnet` 或 `netcat` 这样的工具进行连接：

```sh
telnet 127.0.0.1 8080
```

连接后，你输入的任何内容都将被服务器回显给你。你甚至可以打开多个终端窗口并同时连接，以观察并发处理的实际效果。

## 后续步骤

恭喜！你已成功使用 Tokio 构建并运行了你的第一个异步应用程序。你已经了解了如何设置项目、执行非阻塞 I/O 以及使用任务处理并发操作。

要深入了解 Tokio 如何管理这些并发操作，请参阅 [任务与调度](./tasks-scheduling.md) 指南。