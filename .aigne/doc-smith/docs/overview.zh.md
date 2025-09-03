# 概述

Tokio 是一个用于使用 Rust 编程语言编写可靠、异步和轻量化应用程序的运行时。它是一个事件驱动、非阻塞的 I/O 平台，提供了构建快速、可扩展和安全的网络应用程序所需的工具。

<x-cards data-columns="3">
  <x-card data-title="快速" data-icon="lucide:rocket">
    Tokio 的零成本抽象提供了裸机性能，确保您的应用程序快速且高效。
  </x-card>
  <x-card data-title="可靠" data-icon="lucide:shield-check">
    通过利用 Rust 的所有权、类型系统和并发模型，Tokio 有助于减少程序错误并确保线程安全。
  </x-card>
  <x-card data-title="可扩展" data-icon="lucide:scaling">
    Tokio 占用资源极少，并能自然地处理背压和取消，使您能够构建可扩展的服务。
  </x-card>
</x-cards>

## 核心组件

宏观来看，Tokio 提供了几个主要组件，它们构成了异步应用程序的基础：

```d2
direction: down

"应用程序逻辑" {
  shape: rectangle
}

"Tokio 运行时" {
  shape: package
  grid-columns: 1

  "核心 API" {
    shape: rectangle
    grid-columns: 2

    "任务管理" {
      label: "任务与同步"
      shape: class
    }
    "I/O 原语" {
      label: "异步 I/O"
      shape: class
    }
    "时间工具" {
      label: "计时器与超时"
      shape: class
    }
  }

  "内部引擎" {
    shape: rectangle
    grid-columns: 2
    
    "调度器" {
      label: "任务调度器\n(工作窃取)"
      shape: hexagon
    }

    "驱动器" {
      label: "I/O 驱动器\n(反应器)"
      shape: hexagon
    }
  }

  "操作系统" {
      label: "操作系统事件\n(epoll, kqueue, IOCP)"
      shape: cylinder
  }
}

"应用程序逻辑" -> "Tokio 运行时"."核心 API": "使用"
"Tokio 运行时"."核心 API" -> "Tokio 运行时"."内部引擎": "依赖"
"Tokio 运行时"."内部引擎".Driver -> "OS": "由...支持"

```

*   **处理异步任务的工具**：这包括用于生成和管理任务的原语、用于同步的通道和互斥锁，以及像超时和休眠这样用于处理时间的实用工具。
*   **异步 I/O 的 API**：Tokio 提供了用于 TCP 和 UDP 的非阻塞套接字，以及用于文件系统操作、进程管理和信号处理的实用工具。
*   **用于执行异步代码的运行时**：这包括一个多线程、工作窃取的任务调度器，一个由操作系统事件队列（例如 epoll、kqueue、IOCP）支持的 I/O 驱动器（也称为反应器），以及一个高性能计时器。

## 快速一瞥

下面是一个使用 Tokio 构建的基础 TCP 回声服务器。首先，在您的 `Cargo.toml` 文件中添加 Tokio 作为依赖项，并启用 `full` 特性标志：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

然后，您可以在 `main.rs` 文件中编写服务器逻辑：

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

            // 在循环中，从套接字读取数据，然后将数据写回。
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

这个例子演示了 Tokio 各组件如何协同工作：`#[tokio::main]` 宏设置运行时，`TcpListener` 提供异步网络套接字，`tokio::spawn` 调度一个新任务来并发处理每个传入的连接。

## 后续步骤

本概述介绍了 Tokio 背后的核心理念。要开始构建您的第一个应用程序，请继续阅读“入门”指南。

<x-card data-title="入门" data-icon="lucide:play-circle" data-href="/getting-started" data-cta="开始构建">
  一份关于如何设置新 Tokio 项目的分步指南，内容包括安装和一个简单可行的 TCP 回声服务器示例。
</x-card>