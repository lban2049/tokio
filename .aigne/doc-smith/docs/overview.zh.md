# 概述

Tokio 是一个用于使用 Rust 编程语言编写可靠、异步和轻量级应用程序的运行时。它是一个事件驱动、非阻塞的 I/O 平台，为编写网络应用程序提供了基础构建块，而不会影响速度、可靠性或可伸缩性。

<x-cards>
  <x-card data-title="快速" data-icon="lucide:rocket">
    Tokio 的零成本抽象为你提供了裸机性能，确保你的应用程序尽可能高效地运行。
  </x-card>
  <x-card data-title="可靠" data-icon="lucide:shield-check">
    通过利用 Rust 的所有权、类型系统和并发模型，Tokio 帮助你编写线程安全的代码并减少常见的编程错误。
  </x-card>
  <x-card data-title="可伸缩" data-icon="lucide:line-chart">
    Tokio 占用空间极小，能自然地处理背压和取消，使你的应用程序能在负载下高效扩展。
  </x-card>
</x-cards>

## 核心组件

从高层次来看，Tokio 提供了几个主要组件，它们构成了 Rust 异步应用程序的基础。

```d2
direction: down

"你的应用程序代码" -> "Tokio 运行时": "运行于"

"Tokio 运行时": {
  shape: cloud
  "任务调度器": "管理并执行任务"
  "I/O 反应器": "与操作系统事件交互"
  "计时器": "提供基于时间的事件"
}

"Tokio 运行时"."I/O 反应器" <-> "操作系统事件队列 (epoll, kqueue, IOCP)": "非阻塞 I/O"
```

*   **异步代码的运行时**：Tokio 提供了一个多线程、工作窃取的任务调度器来执行异步任务。它包括一个由操作系统事件队列（例如 epoll、kqueue、IOCP）支持的 I/O 驱动程序和一个高性能计时器。

*   **异步任务的工具**：它提供了一套丰富的工具来处理异步任务，包括像通道和互斥锁这样的[同步原语](./concepts-synchronization.md)，以及用于管理[时间](./concepts-timers.md)的实用程序，例如休眠、间隔和超时。

*   **异步 I/O 的 API**：一套用于执行非阻塞 [I/O](./concepts-io.md) 的全面 API，包括 TCP、UDP 和 Unix 套接字，以及文件系统、进程和信号管理。

## 快速示例

以下是一个基本的 TCP 回显服务器，演示了 Tokio 的一些核心功能。首先，在你的 `Cargo.toml` 中添加 Tokio 作为依赖项，并启用 `full` 功能标志：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

然后，你可以编写服务器代码：

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

            // 在一个循环中，从套接字读取数据，然后再将数据写回。
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
此示例将一个 `TcpListener` 绑定到一个地址，并为每个传入的连接生成一个新的异步任务，以处理从套接字读取数据并将其写回的操作。

## 后续步骤

本概述宏观地介绍了 Tokio 的用途和组件。要开始构建你自己的应用程序，请前往[入门指南](./getting-started.md)。