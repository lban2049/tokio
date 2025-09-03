# 入门指南

本指南将引导你使用 Tokio 创建一个新项目，并构建一个简单的 TCP 回显服务器。阅读完毕后，你将拥有一个可运行的异步应用程序。

## 1. 创建新项目

首先，我们使用 Cargo 创建一个新的 Rust 项目。

```bash
cargo new my-tokio-app
cd my-tokio-app
```

## 2. 添加 Tokio 作为依赖项

Tokio 是模块化的，不同的功能通过特性标志（feature flags）提供。为了方便入门，我们将使用 `full` 标志启用所有功能。

将以下行添加到你的 `Cargo.toml` 文件中：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

这能确保你在构建应用程序时可能需要的所有 API 都立即可用。

## 3. 编写代码

现在，我们来编写 TCP 回显服务器。打开 `src/main.rs` 文件，并将其内容替换为以下代码：

```rust,no_run
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Bind a listener to the address
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    println!("Server listening on port 8080");

    loop {
        // The second item contains the IP and port of the new connection.
        let (mut socket, _) = listener.accept().await?;

        // Spawn a new task to handle each connection.
        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write it back.
            loop {
                let n = match socket.read(&mut buf).await {
                    // Socket closed
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // Write the data back to the socket
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

我们来分解一下代码的执行过程：

-   `#[tokio::main]`: 这是一个宏，它将 `async fn main()` 函数转换为一个同步的 `main` 函数，该函数会初始化一个 Tokio 运行时并执行异步代码。
-   `TcpListener::bind("...").await?`: 这行代码创建一个 TCP 监听器并将其绑定到指定地址。由于绑定是一个异步操作，因此使用了 `.await` 关键字。
-   `listener.accept().await?`: 这行代码会异步等待新的传入连接。当连接建立后，它会返回一个元组，其中包含一个套接字和对端的地址。
-   `tokio::spawn(async move { ... })`: 这行代码创建一个新的异步任务。连接被移入此任务中并进行并发处理，这使得主循环可以继续接受新连接，而无需等待前一个连接处理完成。
-   `socket.read(&mut buf).await`: 这行代码从套接字读取数据到缓冲区中，并返回读取的字节数。如果返回 `Ok(0)`，则表示连接已被客户端关闭。
-   `socket.write_all(&buf[0..n]).await`: 这行代码将缓冲区中的数据写回套接字，从而将其回显给客户端。

## 4. 运行应用程序

现在你可以运行服务器了：

```bash
cargo run
```

你应该会看到输出 `Server listening on port 8080`。

要测试它，请打开一个新的终端窗口，并使用 `telnet` 或 `netcat` 等工具连接到服务器：

```bash
telnet 127.0.0.1 8080
```

你在 `telnet` 会话中输入的任何内容都将被服务器回显。要关闭连接，请在 `telnet` 中按 `Ctrl+]`，然后输入 `quit`。

## 后续步骤

恭喜！你已经成功使用 Tokio 构建了你的第一个异步应用程序。

为了更好地理解你刚才使用的组件以及 Tokio 背后的原理，建议深入学习其核心概念。

<x-card data-title="核心概念" data-icon="lucide:book-open" data-href="/concepts" data-cta="了解更多">
  探索构成 Tokio 运行时的基本概念和组件，为高级用法奠定坚实的基础。
</x-card>