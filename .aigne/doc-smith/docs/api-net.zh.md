# 网络

该模块为 Tokio 提供了异步 TCP、UDP 和 Unix 套接字绑定。它包含与标准库中类似的联网类型，专为构建高性能网络协议而设计。

## 组织结构

`tokio::net` 模块按协议组织，包含用于处理不同类型网络通信的特定类型。主要组件如下：

```d2
direction: down

tokio-net: {
  shape: package
  label: "tokio::net"
  grid-columns: 2
  grid-gap: 50

  TCP: {
    shape: package
    "TcpListener": "接受传入的 TCP 连接"
    "TcpStream": "两个对等点之间的 TCP 流"
    "TcpSocket": "用于配置 TCP 套接字"
  }

  UDP: {
    shape: package
    "UdpSocket": "一个 UDP 套接字"
  }

  Unix: {
    shape: package
    label: "Unix（仅限 Unix）"
    "UnixListener": "接受 Unix 流连接"
    "UnixStream": "两个对等点之间的 Unix 流"
    "UnixDatagram": "一个 Unix 数据报套接字"
    "pipe": "FIFO 管道"
  }

  Windows: {
    shape: package
    label: "Windows（仅限 Windows）"
    "named_pipe": "命名管道"
  }

  "Utilities": {
    shape: package
    "lookup_host": "DNS 解析"
    "ToSocketAddrs": "用于地址转换的 Trait"
  }
}
```

<x-cards data-columns="2">
  <x-card data-title="TCP" data-icon="lucide:arrow-right-left">
    `TcpListener` 和 `TcpStream` 提供了通过 TCP 进行可靠的、面向连接的通信的功能。
  </x-card>
  <x-card data-title="UDP" data-icon="lucide:move-diagonal">
    `UdpSocket` 提供了通过 UDP 进行无连接通信的功能。
  </x-card>
  <x-card data-title="Unix 套接字" data-icon="lucide:server">
    `UnixListener`、`UnixStream` 和 `UnixDatagram` 能够在类 Unix 系统上通过 Unix 域套接字进行通信。
  </x-card>
  <x-card data-title="平台特定" data-icon="lucide:box">
    包括 Unix 管道 (`tokio::net::unix::pipe`) 和 Windows 命名管道 (`tokio::net::windows::named_pipe`)。
  </x-card>
</x-cards>

对于 `tokio::net` 中未直接提供的 I/O 资源，你可以使用 `tokio::io::unix` 模块中的 `AsyncFd`。

## UdpSocket

UDP 套接字是无连接的，这意味着它可以在不建立专用连接的情况下与许多不同的远程对等方通信。使用 `UdpSocket` 主要有两种方式：

1.  **一对多**：使用 `bind` 创建一个套接字，然后使用 `send_to` 和 `recv_from` 与不同的远程地址通信。
2.  **一对一**：使用 `connect` 将套接字与单个远程地址关联，这样你就可以使用更简单的 `send` 和 `recv` 方法。

与 TCP 流不同，`UdpSocket` 没有 `split` 方法。但由于其方法接受 `&self`，因此可以将其包装在 `Arc<UdpSocket>` 中，并通过克隆在多个任务间共享。

### 示例：一对多回声服务器

此示例演示了一个简单的回声服务器，它从任意客户端接收一个数据报，并将其发回原地址。

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

### 示例：一对一（已连接）回声服务器

绑定后，你可以将套接字 `connect` 到一个特定的对等方。这会将传入的数据包过滤为仅来自该地址的数据包，并将其设置为发送操作的默认目标地址。

```rust,no_run
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

### 示例：使用 `Arc` 共享套接字

为了处理并发的发送和接收，你可以将 `UdpSocket` 包装在 `Arc` 中，并为不同的任务克隆它。本示例使用一个通道将接收到的消息传递给发送任务。

```rust,no_run
use tokio::{net::UdpSocket, sync::mpsc};
use std::{io, net::SocketAddr, sync::Arc};

#[tokio::main]
async fn main() -> io::Result<()> {
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