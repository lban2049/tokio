# 入门指南

本指南将引导你完成首个 Tokio 应用的设置。我们将从创建一个新的 Rust 项目开始，将 Tokio 添加为依赖项，然后构建一个简单的 TCP 回声服务器，该服务器会将其接收到的任何数据发回。

### 设置项目

首先，我们使用 Cargo 创建一个新的 Rust 项目：

```bash Create a new project icon=lucide:terminal
cargo new my-tokio-app
cd my-tokio-app
```

接下来，在 `Cargo.toml` 文件中添加 `tokio` crate 作为依赖项。我们将使用 `full` 功能标志启用所有功能。这是最简单的入门方式，可以确保你所需的所有 API 都可用。

```toml Cargo.toml icon=lucide:file-text
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### 编写回声服务器

现在，将 `main.rs` 文件的内容替换为以下代码。该程序将设置一个在 `127.0.0.1:8080` 上监听的服务器，并为每个传入的连接读取数据，然后将相同的数据写回客户端。

```rust main.rs icon=logos:rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // 在循环中，从套接字读取数据并将数据写回。
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

我们来分解一下这段代码：

-   `#[tokio::main]`: 这是一个宏，它将 `async fn main()` 转换为一个同步的 `main()` 函数，该函数会初始化一个 Tokio 运行时并执行异步代码。
-   `TcpListener::bind("127.0.0.1:8080").await?`: 我们创建一个绑定到指定地址的 TCP 监听器。使用 `.await` 关键字是因为绑定是一个异步操作。
-   `loop { ... }`: 服务器进入一个循环，以持续接受新的连接。
-   `listener.accept().await?`: 这会异步地等待一个新的入站连接。当连接建立时，它会返回一个包含套接字和对端地址的元组。
-   `tokio::spawn(async move { ... });`: 对于每个连接，都会生成一个新任务。这使得服务器能够并发处理多个连接。`move` 关键字将 `socket` 的所有权转移给新任务。
-   `socket.read(&mut buf).await`: 在任务内部，我们从套接字读取数据到缓冲区。这是另一个异步操作，所以我们使用 `.await` 等待它完成。
-   `socket.write_all(&buf[0..n]).await`: 我们将刚刚读取的数据写回套接字，从而实现“回声”效果。

### 运行服务器

代码就绪后，你可以在终端中运行服务器：

```bash Run the application icon=lucide:terminal
cargo run
```

服务器现已运行。要进行测试，请打开一个新的终端窗口，并使用 `netcat` 或 `telnet` 等工具连接到它。

```bash Test with netcat icon=lucide:terminal
nc 127.0.0.1 8080
```

连接后，输入任意消息并按回车键，你将看到同样的消息被回显回来。要停止服务器，请返回第一个终端并按 `Ctrl+C`。

恭喜！你已经使用 Tokio 构建了你的第一个异步应用。

### 后续步骤

既然你已经运行了一个基础应用，就可以开始深入了解 Tokio 的基本构建块了。

<x-card data-title="核心概念" data-icon="lucide:puzzle" data-href="/concepts" data-cta="探索概念">
深入了解任务、I/O、状态管理和运行时本身，以理解 Tokio 的底层工作原理。
</x-card>