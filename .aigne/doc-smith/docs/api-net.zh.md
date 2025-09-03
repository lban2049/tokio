# 网络

该模块为 Tokio 提供了异步的 TCP、UDP 和 Unix 套接字绑定。这些组件在设计上与 Rust 标准库中的对应部分相似，但它们是非阻塞的，并与 Tokio 运行时集成。

## 网络原语概述

Tokio 的 `net` 模块按协议组织，为不同的通信需求提供了一系列类型。

```d2
direction: down

"tokio::net": {
  shape: package
  grid-columns: 3
  grid-gap: 50

  "TCP (面向连接)": {
    shape: rectangle
    "TcpListener": "接受传入连接"
    "TcpStream": "表示一个 TCP 流"
    "TcpSocket": "底层套接字配置"
  }

  "UDP (无连接)": {
    shape: rectangle
    "UdpSocket": "发送/接收数据报"
  }

  "IPC (类 Unix 系统)": {
    shape: rectangle
    "UnixListener": "基于流的监听器"
    "UnixStream": "流连接"
    "UnixDatagram": "数据报套接字"
    "Pipes": "FIFO 管道"
  }
}
```

以下是可用于构建网络协议的主要组件的快速指南。

<x-cards data-columns="2">
  <x-card data-title="TCP" data-icon="lucide:server">
    用于可靠的、面向流的通信。包括用于接受连接的 `TcpListener` 和用于数据传输的 `TcpStream`。
  </x-card>
  <x-card data-title="UDP" data-icon="lucide:send">
    用于无连接的、基于数据报的通信。`UdpSocket` 可用于向多个远程端发送数据和接收来自它们的数据。
  </x-card>
  <x-card data-title="Unix 域套接字" data-icon="lucide:box">
    用于类 Unix 系统上的进程间通信 (IPC)。提供流 (`UnixListener`, `UnixStream`) 和数据报 (`UnixDatagram`) 两种变体。
  </x-card>
  <x-card data-title="管道" data-icon="lucide:workflow">
    用于特定平台的基于管道的通信，例如 Unix 上的 FIFO 管道和 Windows 上的命名管道。
  </x-card>
</x-cards>

## TCP (传输控制协议)

TCP 在应用程序之间提供可靠、有序且经过错误校验的字节流。它是许多互联网协议（如 HTTP 和 FTP）的基础。

-   **`TcpListener`**：一个用于接受传入 TCP 连接的异步监听器。
-   **`TcpStream`**：表示本地和远程套接字之间的 TCP 连接。它可以被拆分为拥有的读取和写入两半。
-   **`TcpSocket`**：一个用于在 TCP 套接字开始监听或连接之前创建和配置它的底层工具。

### 示例：TCP 回显服务器

这是一个简单的 TCP 回显服务器示例，它接受连接并回显接收到的任何数据。

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = vec![0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    // `Ok(0)` 的返回值表示远程端已经
                    // 关闭了连接。
                    Ok(0) => return,
                    Ok(n) => {
                        // 将数据复制回套接字
                        if socket.write_all(&buf[..n]).await.is_err() {
                            // 意外错误。直接退出任务。
                            return;
                        }
                    }
                    Err(_) => {
                        // 意外错误。直接退出任务。
                        return;
                    }
                }
            }
        });
    }
}
```

## UDP (用户数据报协议)

UDP 是一种无连接协议，提供简单但不可靠的数据报服务。它适用于速度至关重要且可以接受少量数据丢失的应用，如游戏或语音聊天。

`UdpSocket` 类型主要有两种使用方式：

1.  **一对多**：一个绑定到某个地址的套接字可以使用 `send_to` 和 `recv_from` 向许多不同的远程对等端发送和接收数据报。
2.  **一对一**：一个套接字可以 `connect` 到单个远程对等端，从而可以使用 `send` 和 `recv` 进行通信，这会将传入的数据包过滤为仅来自该地址的数据包。

### 示例：一对多 UDP 回显服务器

该服务器绑定到一个地址，并将接收到的任何数据报回显给其原始发送方。

```rust,no_run
use tokio::net::UdpSocket;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let sock = UdpSocket::bind("0.0.0.0:8080").await?;
    let mut buf = [0; 1024];
    loop {
        let (len, addr) = sock.recv_from(&mut buf).await?;
        println!("{:?} bytes received from {:?}", len, addr);

        let len = sock.send_to(&buf[..len], addr).await?;
        println!("{:?} bytes sent", len);
    }
}
```

### 共享 `UdpSocket`

由于其方法接收的是 `&self` 而不是 `&mut self`，因此可以通过将 `UdpSocket` 包装在 `Arc<UdpSocket>` 中，在多个任务之间安全地共享以进行并发读写。

## Unix 域套接字 (UDS)

Unix 域套接字仅在类 Unix 系统上可用，它有助于在同一台机器上进行进程间通信 (IPC)。它们的行为类似于 TCP 流，但使用文件系统路径进行寻址，而不是 IP 地址和端口。

-   **`UnixListener`** 和 **`UnixStream`**：提供面向流的连接，类似于 TCP。
-   **`UnixDatagram`**：提供基于数据报的套接字，类似于 UDP。
-   **`UnixSocket`**：一个用于创建和配置 Unix 套接字的底层工具。

## 平台特定的网络

Tokio 还包含用于特定操作系统网络功能的模块。

-   **Windows**：`tokio::net::windows` 模块提供对命名管道的支持。
-   **Unix**：`tokio::net::unix` 模块包含 UDS 类型以及通过 `tokio::net::unix::pipe` 对 FIFO 管道的支持。

## 工具

### DNS 解析

`lookup_host` 函数提供了一种执行 DNS 解析的异步方式。

### 地址处理

网络类型使用 `ToSocketAddrs` trait 将各种地址表示形式转换为一个或多个 `SocketAddr` 实例。

---

借助这些网络原语，您可以构建各种各样的应用程序。要获取更多实践示例，请查看 [示例](./examples.md) 部分。要了解底层的 I/O 操作，请参阅 [I/O API 参考](./api-io.md)。