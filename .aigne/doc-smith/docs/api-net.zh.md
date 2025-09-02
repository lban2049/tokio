# 网络

该模块为 Tokio 提供了异步的 TCP、UDP 和 Unix 套接字绑定，从而支持开发高性能网络应用程序。这些组件在设计上与其在标准库中的对应部分类似，但以非阻塞方式运行，与 Tokio 运行时无缝集成。

## 概述

Tokio 的网络原语涵盖了最常见的协议和进程间通信 (IPC) 机制。以下是关键组件及其关系的直观分解：

```d2
direction: down

"网络应用程序": {
  "TCP 服务器": {
    "TcpListener" -> "TcpStream" : 接受
  }
  
  "TCP 客户端": {
    "TcpStream"
  }

  "UDP 对等端": {
    "UdpSocket"
  }
  
  "Unix 域服务器": {
    "UnixListener" -> "UnixStream" : 接受
  }

  "Unix 域客户端": {
    "UnixStream"
  }
}

"远程对等端": {
  "远程 TCP 对等端 1"
  "远程 TCP 对等端 2"
  "远程 UDP 对等端 A"
  "远程 UDP 对等端 B"
}

"本地进程": {
  "进程 A"
  "进程 B"
}

"网络应用程序"."TCP 客户端"."TcpStream" <-> "远程对等端"."远程 TCP 对等端 1": TCP 连接
"网络应用程序"."TCP 服务器"."TcpStream" <-> "远程对等端"."远程 TCP 对等端 2": TCP 连接
"网络应用程序"."UDP 对等端"."UdpSocket" <-> "远程对等端"."远程 UDP 对等端 A": UDP 数据报
"网络应用程序"."UDP 对等端"."UdpSocket" <-> "远程对等端"."远程 UDP 对等端 B": UDP 数据报
"网络应用程序"."Unix 域客户端"."UnixStream" <-> "本地进程"."进程 A": IPC
"网络应用程序"."Unix 域服务器"."UnixStream" <-> "本地进程"."进程 B": IPC
```

## 核心组件

`tokio::net` 模块按协议进行组织。以下是您将使用的主要类型：

<x-cards data-columns="2">
  <x-card data-title="TCP 套接字" data-icon="lucide:arrow-right-left">
    提供 `TcpListener` 用于接受传入的流连接，以及 `TcpStream` 用于通过 TCP 进行通信。非常适合像 HTTP 这样可靠的、面向连接的协议。
  </x-card>
  <x-card data-title="UDP 套接字" data-icon="lucide:move-diagonal">
    提供 `UdpSocket` 用于通过 UDP 进行无连接的、基于数据报的通信。适用于速度优先于可靠性的应用，如游戏或流媒体。
  </x-card>
  <x-card data-title="Unix 域套接字" data-icon="lucide:server">
    用于在基于 Unix 的系统上进行进程间通信 (IPC) 的流和数据报套接字。包括 `UnixListener`、`UnixStream` 和 `UnixDatagram`。
  </x-card>
  <x-card data-title="管道" data-icon="lucide:pipeline">
    用于 IPC 的平台特定管道。这包括用于 Unix 上 FIFO 管道的 `tokio::net::unix::pipe` 和 Windows 上的 `tokio::net::windows::named_pipe`。
  </x-card>
</x-cards>

## TCP (传输控制协议)

TCP 提供可靠、有序且经过错误校验的字节流传输。Tokio 为 TCP 网络提供了两种主要类型：

-   **`TcpListener`**：`std::net::TcpListener` 的异步版本。用于监听和接受传入的 TCP 连接。
-   **`TcpStream`**：本地和远程套接字之间的异步 TCP 流。它实现了 `AsyncRead` 和 `AsyncWrite` 用于发送和接收数据。
-   **`TcpSocket`**：一个较低级别的套接字，允许在用于连接或监听之前进行配置（例如，设置 `SO_REUSEADDR`）。

## UDP (用户数据报协议)

UDP 是一种无连接协议，提供基于数据报的通信服务。它优先考虑速度和低延迟，而不是可靠性。

### 使用模式

`UdpSocket` 主要有两种使用方式：

1.  **一对多 (未连接)**：将套接字绑定到一个地址，并使用 `send_to` 和 `recv_from` 与多个远程对等端通信。这是服务器处理多个客户端的典型用例。

    *示例：一个处理多个客户端的回显服务器。*
    ```rust
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

2.  **一对一 (已连接)**：使用 `connect` 将套接字与单个远程地址关联。连接后，您可以使用更方便的 `send` 和 `recv` 方法，该套接字将只与该特定对等端进行发送和接收。

    *示例：一个连接到单个服务器的回显客户端。*
    ```rust
    use tokio::net::UdpSocket;
    use std::io;
    
    #[tokio::main]
    async fn main() -> io::Result<()> {
        let sock = UdpSocket::bind("0.0.0.0:8080").await?;
    
        let remote_addr = "127.0.0.1:59611";
        sock.connect(remote_addr).await?;
        let mut buf = [0; 1024];
        loop {
            let len = sock.recv(&mut buf).await?;
            println!("{:?} bytes received from {:?}", len, remote_addr);
    
            let len = sock.send(&buf[..len]).await?;
            println!("{:?} bytes sent", len);
        }
    }
    ```

### 并发与拆分

与 `TcpStream` 不同，`UdpSocket` 没有 `split` 方法。然而，由于其发送和接收方法接受 `&self`，单个套接字可以使用 `Arc<UdpSocket>` 在多个任务之间共享。

*示例：使用 `Arc` 进行并发发送和接收。*
```rust
use tokio::{net::UdpSocket, sync::mpsc};
use std::{io, net::SocketAddr, sync::Arc};

#[tokio::main]
asyn fn main() -> io::Result<()> {
    let sock = UdpSocket::bind("0.0.0.0:8080".parse::<SocketAddr>().unwrap()).await?;
    let r = Arc::new(sock);
    let s = r.clone();
    let (tx, mut rx) = mpsc::channel::<(Vec<u8>, SocketAddr)>(1_000);

    tokio::spawn(async move {
        while let Some((bytes, addr)) = rx.recv().await {
            let len = s.send_to(&bytes, &addr).await.unwrap();
            println!("{:?} bytes sent", len);
        }
    });

    let mut buf = [0; 1024];
    loop {
        let (len, addr) = r.recv_from(&mut buf).await?;
        println!("{:?} bytes received from {:?}", len, addr);
        tx.send((buf[..len].to_vec(), addr)).await.unwrap();
    }
}
```

## Unix 域套接字

对于类 Unix 系统上的进程间通信 (IPC)，Tokio 提供了异步 Unix 域套接字。它们的行为类似于 TCP 套接字，但操作的是本地文件系统路径，而不是 IP 地址和端口。

-   **`UnixListener`** 和 **`UnixStream`**：用于面向连接的、基于流的通信，类似于 TCP。
-   **`UnixDatagram`**：用于无连接的、基于数据报的通信，类似于 UDP。
-   **`UnixSocket`**：用于高级配置的较低级别套接字。

这些类型仅在 Unix 平台上可用。

## 自定义 I/O 资源

对于 `tokio::net` 中未原生提供的 I/O 资源，例如原始套接字或其他平台特定的句柄，您可以使用 [`AsyncFd`](./api-io.md) 将它们与 Tokio 运行时集成。这使您能够对任何可以被操作系统事件队列（如 epoll、kqueue 或 IOCP）监视的文件描述符执行非阻塞 I/O 操作。

---

现在您已经对 Tokio 的网络功能有了大致了解，可以继续探索为其提供支持的[异步 I/O 特征和辅助工具](./api-io.md)，或深入了解用于在网络应用程序中管理状态的[同步原语](./api-sync.md)。