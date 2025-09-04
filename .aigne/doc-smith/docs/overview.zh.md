# 概述

Tokio 是一个事件驱动、非阻塞的 I/O 平台，用于使用 Rust 编程语言编写异步应用程序。它提供了构建快速、可靠和可扩展的网络应用程序所需的运行时和工具，同时不影响速度。

Tokio 的核心建立在几个关键原则之上：

*   **快速**：零成本抽象提供了裸机般的性能。
*   **可靠**：它利用 Rust 的所有权、类型系统和并发模型来减少错误并确保线程安全。
*   **可扩展**：占用资源少，能自然地处理背压和取消，使其适用于任何规模的应用程序。

## 核心组件

Tokio 提供了几个构成异步应用程序基础的主要组件：

*   **多线程任务调度器**：一个工作窃取调度器，用于在多个 CPU 核心上高效地执行异步任务。
*   **I/O 驱动程序 (Reactor)**：由操作系统的事件队列（如 epoll、kqueue 或 IOCP）支持，此组件驱动异步 I/O 操作。
*   **异步 API**：一套丰富的 API，用于处理异步任务、I/O、计时和同步，包括 TCP/UDP 套接字、文件系统操作、定时器和通道。

下图展示了这些组件如何交互的高层视图：

```d2
direction: down

"User Application": {
  shape: rectangle
  label: "你的应用程序代码"
}

"Tokio Runtime": {
  shape: package
  label: "Tokio 运行时"
  grid-columns: 1

  "Scheduler": {
    label: "任务调度器"
    shape: rectangle
  }

  "Driver": {
    label: "I/O 驱动程序和计时器"
    shape: rectangle
  }
}

"OS": {
  shape: cylinder
  label: "操作系统\n(事件队列、套接字、文件)"
}

"User Application" -> "Tokio Runtime"."Scheduler": "生成异步任务"
"Tokio Runtime"."Scheduler" <-> "Tokio Runtime"."Driver": "轮询任务就绪状态"
"Tokio Runtime"."Driver" <-> "OS": "注册 I/O 事件"

```

## 快速一览

为了让你感受一下 Tokio 代码的样子，下面提供一个基本的 TCP 回声（echo）服务器。它会监听传入的连接，并将收到的任何数据直接写回客户端。

首先，在你的 `Cargo.toml` 文件中将 Tokio 添加为依赖项，并启用 `full` 功能标志：

```toml
tokio = { version = "1", features = ["full"] }
```

然后，你可以编写服务器逻辑：

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

            // 在一个循环中，从套接字读取数据，然后再写回去。
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

此示例展示了 Tokio 的几个关键部分：用于启动运行时的 `#[tokio::main]` 宏、异步 TCP 套接字（`TcpListener`、通过 `socket` 得到的 `TcpStream`），以及使用 `tokio::spawn` 生成并发任务。

## 接下来做什么？

<x-cards>
  <x-card data-title="开始使用" data-icon="lucide:rocket" data-href="/getting-started">
    按照分步指南，设置你的第一个 Tokio 项目并运行一个可行的示例。
  </x-card>
  <x-card data-title="核心概念" data-icon="lucide:book-open" data-href="/concepts">
    深入了解 Tokio 的基本概念，例如任务、I/O 和运行时。
  </x-card>
</x-cards>