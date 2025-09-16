# 网络

Tokio 为 TCP、UDP 和 Unix 域套接字提供异步、非阻塞的网络原语。这些组件被设计为高效、可靠且易于使用，使你能够构建高性能的网络应用程序。

本指南涵盖了 Tokio 中可用的基本网络类型。对于更专门的 I/O 资源，你可以使用 `AsyncFd`。

## TCP (传输控制协议)

TCP 是一种面向连接的协议，提供可靠、有序且经过错误校验的字节流传输。Tokio 为 TCP 通信提供了两种主要类型：用于服务器的 `TcpListener` 和用于客户端的 `TcpStream`。

### `TcpListener`

`TcpListener` 是一个 TCP 套接字服务器，用于接受传入的连接。你可以通过将其绑定到本地地址来创建一个监听器。

```rust TCP Server Example icon=logos:rust
use tokio::net::TcpListener;
use std::io;

async fn process_socket<T>(socket: T) {
    // 处理连接的业务逻辑
    # drop(socket);
}

#[tokio::main]
asyn fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            process_socket(socket).await;
        });
    }
}
```

`accept` 方法等待新连接，并返回一个 `TcpStream` 以及客户端的地址。

### `TcpStream`

`TcpStream` 表示本地套接字和远程套接字之间的 TCP 连接。它可以通过连接到远程端点或接受来自 `TcpListener` 的连接来创建。

```rust TCP Client Example icon=logos:rust
use tokio::net::TcpStream;
use tokio::io::AsyncWriteExt;
use std::error::Error;

#[tokio::main]
asyn fn main() -> Result<(), Box<dyn Error>> {
    // 连接到一个对等节点
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;

    // 写入一些数据
    stream.write_all(b"hello world!").await?;

    Ok(())
}
```

`TcpStream` 实现了 `AsyncRead` 和 `AsyncWrite` 特性，允许你异步地从流中读取和写入数据。

### 使用 `TcpSocket` 进行高级配置

在某些情况下，你需要在套接字被绑定或连接之前对其进行配置，Tokio 为此提供了 `TcpSocket` 类型。这允许你设置诸如 `SO_REUSEADDR` 之类的套接字选项。

```rust Configuring a TCP Socket icon=logos:rust
use tokio::net::TcpSocket;
use std::io;

#[tokio::main]
asyn fn main() -> io::Result<()> {
    let addr = "127.0.0.1:8080".parse().unwrap();

    let socket = TcpSocket::new_v4()?;
    
    // 在 Unix 上，这允许套接字被快速重新绑定。
    // 在 Windows 上，这允许重新绑定到正在使用的套接字。
    socket.set_reuseaddr(true)?;
    
    socket.bind(addr)?;

    let listener = socket.listen(1024)?;
    # drop(listener);

    Ok(())
}
```

## UDP (用户数据报协议)

UDP 是一种无连接协议，提供基于数据报的服务。与 TCP 不同，它不保证交付、顺序或错误检查。Tokio 的 `UdpSocket` 可用于发送和接收 UDP 数据报。

### `UdpSocket`

`UdpSocket` 主要有两种使用方式：

1.  **一对多**：绑定套接字并使用 `send_to` 和 `recv_from` 与多个远程对等节点通信。
2.  **一对一**：`connect` 套接字到单个远程地址，并使用 `send` 和 `recv` 进行通信。

以下是一个简单的 UDP 回显服务器示例：

```rust UDP Echo Server icon=logos:rust
use tokio::net::UdpSocket;
use std::io;

#[tokio::main]
asyn fn main() -> io::Result<()> {
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

由于其方法接收 `&self`，`UdpSocket` 可以被包装在 `Arc<UdpSocket>` 中，以便在多个任务之间共享，从而实现并发发送和接收。

## Unix 域套接字 (UDS)

对于类 Unix 系统上的进程间通信 (IPC)，Tokio 提供了异步的 Unix 域套接字。它们的行为与其 TCP/UDP 对应物类似，但在本地文件系统路径上操作，而不是 IP 地址和端口。

-   **`UnixListener`** 和 **`UnixStream`**: 提供可靠的、面向流的连接，类似于 TCP。
-   **`UnixDatagram`**: 提供无连接的、面向数据报的服务，类似于 UDP。

## 平台特定的网络

Tokio 还支持其他平台特定的网络类型：

-   **`tokio::net::unix::pipe`**: 用于在 Unix 系统上处理 FIFO 管道。
-   **`tokio::net::windows::named_pipe`**: 用于在 Windows 上处理命名管道。

这些原语使你能够构建各种健壮且高效的网络服务。

接下来，让我们探讨如何执行[异步文件系统操作](./io-fs.md)。