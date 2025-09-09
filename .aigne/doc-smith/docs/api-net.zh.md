# 网络

该模块为 Tokio 提供了异步 TCP、UDP 和 Unix 套接字绑定。这些组件的设计旨在提供熟悉的使用体验，它们的功能与标准库的网络类型相似，但采用的是非阻塞的异步方式。

无论你是在构建 Web 服务器、远程 API 的客户端，还是点对点应用程序，该模块都包含了网络通信所需的基础工具。

要深入了解异步 I/O 模式，请参阅 [I/O API 文档](./api-io.md)。

## 核心组件

`net` 模块围绕主要网络协议进行组织：

<x-cards data-columns="3">
  <x-card data-title="TCP" data-icon="lucide:arrow-right-left">
    用于可靠的、面向连接的通信。包括用于接受连接的 `TcpListener` 和用于数据传输的 `TcpStream`。
  </x-card>
  <x-card data-title="UDP" data-icon="lucide:move-diagonal">
    用于使用数据报进行快速、无连接的通信。主要类型是 `UdpSocket`。
  </x-card>
  <x-card data-title="Unix Sockets" data-icon="lucide:server">
    用于在类 Unix 系统上进行高效的进程间通信。包括流、数据报和管道类型。
  </x-card>
</x-cards>

## TCP (传输控制协议)

TCP 提供可靠、有序且经过错误校验的字节流交付。它是大多数常见应用协议（如 HTTP、FTP 和 SMTP）的基础。

### `TcpListener`

`TcpListener` 用于接受传入的 TCP 连接。将其绑定到套接字地址后，你可以通过循环来处理新的客户端连接。

```rust icon=logos:rust TCP Echo Server
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use std::io;

async fn process_socket(mut socket: TcpStream) -> io::Result<()> {
    let mut buf = [0; 1024];
    loop {
        let n = socket.read(&mut buf).await?;
        if n == 0 { return Ok(()); }
        socket.write_all(&buf[0..n]).await?;
    }
}

#[tokio::main]
asyn fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            if let Err(e) = process_socket(socket).await {
                println!("failed to process socket; error = {:?}", e);
            }
        });
    }
}
```

### `TcpStream`

`TcpStream` 表示本地和远程套接字之间的 TCP 连接。你可以通过 `TcpStream::connect` 连接到远程主机，或通过从 `TcpListener` 接受连接来创建 `TcpStream`。

`TcpStream` 提供了读写数据的方法，并可以被拆分为读半部分和写半部分，以便在不同的任务中使用。

### `TcpSocket`

对于高级配置，`TcpSocket` 允许你在开始监听连接或连接到远程对等方之前创建和配置套接字。这对于设置诸如 `SO_REUSEADDR` 之类的套接字选项非常有用。

## UDP (用户数据报协议)

UDP 是一种无连接协议，通过发送称为数据报的独立数据包来提供低延迟通信。它不保证交付、顺序或防止重复，因此适用于游戏、流媒体或 DNS 等应用。

### `UdpSocket`

`UdpSocket` 类型主要有两种使用方式：

1.  **一对多 (One-to-Many)：** 将套接字绑定到本地地址，并使用 `send_to` 和 `recv_from` 与多个远程对等方通信。这是服务器应用程序的理想选择。
2.  **一对一 (One-to-One)：** 绑定后，将套接字 `connect` 到单个远程地址。这样就可以使用更简单的 `send` 和 `recv` 方法，这对于客户端应用程序通常更为方便。

#### 示例：一对多回显服务器

该服务器监听数据报，并将其回显给发送方地址。

```rust icon=logos:rust UDP Bind Example
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

#### 示例：一对一连接的客户端

此示例连接到一个特定的远程对等方，并使用 `send` 和 `recv` 进行通信。

```rust icon=logos:rust UDP Connect Example
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

#### 共享 `UdpSocket`

由于 `UdpSocket` 上的所有 I/O 方法都接受 `&self` 参数，你可以安全地将套接字包装在 `Arc<UdpSocket>` 中，以便在多个任务间共享，从而实现并发发送和接收，而无需使用 `Mutex`。

```rust icon=logos:rust Sharing UdpSocket with Arc
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

## Unix 域套接字

Unix 域套接字 (UDS) 可用于类 Unix 平台，以促进同一主机上的进程间通信 (IPC)。它们的行为与 TCP/UDP 套接字类似，但使用文件系统路径而非 IP 地址和端口。对于本地通信，它们可以提供更好的性能和安全性。

*   **`UnixListener` 和 `UnixStream`**：提供面向连接的流套接字，类似于 `TcpListener` 和 `TcpStream`。
*   **`UnixDatagram`**：提供无连接的数据报套接字，用于发送单个消息，类似于 `UdpSocket`。
*   **`tokio::net::unix::pipe`**：提供 FIFO 管道的功能。

## 平台特定的网络功能

Tokio 还包含特定于某些操作系统的网络类型。

*   **Windows 命名管道**：在 Windows 上，`tokio::net::windows::named_pipe` 提供了与命名管道通信的 API，这是该平台上一种常见的 IPC 机制。

## 实用工具

### `lookup_host`

你可以使用 `lookup_host` 函数执行异步 DNS 查询。该函数将主机字符串解析为一个或多个套接字地址。

```rust icon=logos:rust DNS Lookup
use tokio::net;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let addrs = net::lookup_host("tokio.rs:80").await?;

    for addr in addrs {
        println!("socket address is {}", addr);
    }

    Ok(())
}
```

### `ToSocketAddrs` Trait

许多网络函数（如 `TcpStream::connect` 和 `UdpSocket::bind`）都泛型于 `ToSocketAddrs` trait。这使你可以传递各种地址类型，例如 `SocketAddr`、`(IpAddr, u16)` 元组或字符串（如 `"127.0.0.1:8080"`），系统将为你解析这些地址。