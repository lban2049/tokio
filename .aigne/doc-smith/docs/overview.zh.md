# 概述

Tokio 是一个用于使用 Rust 编程语言编写可靠、异步和轻量级应用的运行时。它提供了一个事件驱动的非阻塞 I/O 平台，这对于构建高性能网络应用至关重要。

<x-cards data-columns="3">
  <x-card data-title="快速" data-icon="lucide:rocket">
    Tokio 的零成本抽象为你提供裸机性能，确保你的应用快速高效。
  </x-card>
  <x-card data-title="可靠" data-icon="lucide:shield-check">
    通过利用 Rust 的所有权、类型系统和并发模型，Tokio 有助于减少错误并确保线程安全。
  </x-card>
  <x-card data-title="可扩展" data-icon="lucide:scaling">
    Tokio 的资源占用极小，能自然地处理背压和取消，让你的应用能够有效扩展。
  </x-card>
</x-cards>

## 核心组件

从宏观上看，Tokio 提供了几个主要组件，它们是异步应用的构建模块：

*   **异步任务工具**：Tokio 提供了一个强大的工具包来管理并发操作。这包括创建任务，使用通道和互斥锁等同步原语，以及处理休眠、间隔和超时等基于时间的操作。这些对于在异步环境中管理控制流至关重要。更多详情，请参阅 [任务与调度](./tasks-scheduling.md) 指南。

*   **异步 I/O API**：通过一套全面的 API 执行非阻塞输入和输出。Tokio 支持 TCP、UDP 和 Unix 域套接字，以及异步文件系统操作、子进程管理和操作系统信号处理。在 [异步 I/O](./io.md) 部分深入了解这些功能。

*   **Tokio 运行时**：运行时是执行异步代码的引擎。它包含一个多线程、工作窃取的任务调度器，一个由操作系统事件队列（如 epoll、kqueue 或 IOCP）支持的 I/O 驱动，以及一个高性能计时器。在 [运行时](./tasks-scheduling-runtime.md) 文档中了解如何配置和管理它。

## 快速示例

这是一个基础的 TCP 回显服务器，演示了这些组件如何协同工作。它监听传入的连接，并将接收到的任何数据写回客户端。

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

此示例展示了 Tokio 的几个关键功能：

-   `#[tokio::main]`: 一个用于启动 Tokio 运行时并执行 `async` 主函数的宏。
-   `TcpListener`: 一个用于接受传入连接的异步 TCP 监听器。
-   `tokio::spawn`: 一个为每个连接创建新异步任务的函数，允许服务器并发处理多个客户端。
-   `AsyncReadExt` 和 `AsyncWriteExt`: 在套接字上提供异步 `read` 和 `write` 方法的 Trait。

## 后续步骤

现在你对 Tokio 提供的功能有了宏观的了解，可以开始构建了。请前往 [入门](./getting-started.md) 指南，查看关于设置你的第一个 Tokio 应用的分步教程。