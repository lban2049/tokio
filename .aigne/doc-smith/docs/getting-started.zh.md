# 入门指南

本指南将引导你逐步设置一个新的 Tokio 项目，并构建一个简单可用的 TCP 回显服务器。读完本指南后，你将拥有一个可运行的异步应用程序。

## 1. 设置项目

首先，你需要一个新的 Rust 项目。如果没有，可以使用 Cargo 创建：

```bash
cargo new my-tokio-app
cd my-tokio-app
```

接下来，将 `tokio` crate 添加为 `Cargo.toml` 文件中的依赖项。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

我们启用了 `full` 特性标志，以包含所有公共 API。建议应用程序这样做，以确保你可以使用所有必要的工具，而无需在构建过程中指定单个特性。

## 2. 编写回显服务器

现在，将 `src/main.rs` 的内容替换为以下代码，以创建 TCP 回显服务器。

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // 在一个循环中，从套接字读取数据，然后将数据写回。
            loop {
                let n = match socket.read(&mut buf).await {
                    // 套接字关闭
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

### 代码解析

- **`#[tokio::main]`**：这是一个宏，它将 `async fn main` 转换为一个同步的 `main` 函数，该函数会初始化一个 Tokio 运行时并执行异步代码。
- **`TcpListener::bind("127.0.0.1:8080").await?`**：此行代码创建一个绑定到指定地址的 `TcpListener`。它会异步等待监听器成功创建。
- **`listener.accept().await?`**：`accept` 方法等待新的传入连接。当连接建立时，它会返回一个新的 `TcpSocket` 和对端的地址。
- **`tokio::spawn(async move { ... })`**：此函数会生成一个新的异步任务。服务器在各自的任务中并发处理每个传入的连接，从而允许它同时管理多个客户端。`move` 关键字将 `socket` 的所有权转移给新任务。
- **`socket.read(&mut buf).await`**：此代码从套接字读取数据到缓冲区 `buf` 中。`.await` 会暂停任务，直到有数据可用。
- **`socket.write_all(&buf[0..n]).await`**：此代码将刚从缓冲区读取的数据写回套接字，从而将其回显给客户端。

## 3. 运行应用程序

代码准备就绪后，你可以使用 Cargo 运行服务器：

```bash
cargo run
```

服务器现在正在运行并等待传入连接。要测试它，请打开一个新的终端窗口，并使用 `netcat` 或 `telnet` 等工具连接到它：

```bash
telnet 127.0.0.1 8080
```

连接后，输入任何消息并按回车键。服务器会将消息回显给你。要停止服务器，可以在其运行的终端中使用 `Ctrl+C`。

## 后续步骤

恭喜！你已经成功使用 Tokio 构建了你的第一个异步应用程序。要继续你的学习之旅，可以探索支持 Tokio 的核心概念或浏览更多示例。

<x-cards data-columns="2">
  <x-card data-title="核心概念" data-icon="lucide:puzzle" data-href="/concepts">
    深入了解 Tokio 的基本组件，包括任务、异步 I/O、同步和运行时。
  </x-card>
  <x-card data-title="示例" data-icon="lucide:lightbulb" data-href="/examples">
    浏览一系列可运行的代码示例，这些示例演示了 Tokio 的各种功能和常见用例。
  </x-card>
</x-cards>