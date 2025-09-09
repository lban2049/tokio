# Networking

This module provides asynchronous TCP, UDP, and Unix socket bindings for Tokio. These components are designed to feel familiar, mirroring the functionality of the standard library's networking types but in a non-blocking, asynchronous way.

Whether you're building a web server, a client for a remote API, or a peer-to-peer application, this module contains the foundational tools you'll need for network communication.

For a deeper dive into asynchronous I/O patterns, see the [I/O API documentation](./api-io.md).

## Core Components

The `net` module is organized around the primary networking protocols:

<x-cards data-columns="3">
  <x-card data-title="TCP" data-icon="lucide:arrow-right-left">
    For reliable, connection-oriented communication. Includes `TcpListener` for accepting connections and `TcpStream` for data transfer.
  </x-card>
  <x-card data-title="UDP" data-icon="lucide:move-diagonal">
    For fast, connectionless communication using datagrams. The primary type is `UdpSocket`.
  </x-card>
  <x-card data-title="Unix Sockets" data-icon="lucide:server">
    For efficient inter-process communication on Unix-like systems. Includes stream, datagram, and pipe types.
  </x-card>
</x-cards>

## TCP (Transmission Control Protocol)

TCP provides reliable, ordered, and error-checked delivery of a stream of bytes. It's the foundation for most common application protocols like HTTP, FTP, and SMTP.

### `TcpListener`

A `TcpListener` is used to accept incoming TCP connections. After binding it to a socket address, you can loop over it to handle new client connections.

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

A `TcpStream` represents a TCP connection between a local and a remote socket. You can create one by either connecting to a remote host with `TcpStream::connect` or by accepting a connection from a `TcpListener`.

`TcpStream` provides methods for reading and writing data and can be split into a read half and a write half to be used in different tasks.

### `TcpSocket`

For advanced configuration, `TcpSocket` allows you to create and configure a socket before you start listening for connections or connect to a remote peer. This is useful for setting socket options like `SO_REUSEADDR`.

## UDP (User Datagram Protocol)

UDP is a connectionless protocol that offers low-latency communication by sending individual packets called datagrams. It doesn't guarantee delivery, ordering, or duplicate protection, making it suitable for applications like gaming, streaming, or DNS.

### `UdpSocket`

The `UdpSocket` type can be used in two primary ways:

1.  **One-to-Many:** Bind a socket to a local address and use `send_to` and `recv_from` to communicate with multiple remote peers. This is ideal for server applications.
2.  **One-to-One:** After binding, `connect` the socket to a single remote address. This allows you to use the simpler `send` and `recv` methods, which is often more convenient for client applications.

#### Example: One-to-Many Echo Server

This server listens for datagrams and echoes them back to the sender's address.

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

#### Example: One-to-One Connected Client

This example connects to a specific remote peer and communicates using `send` and `recv`.

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

#### Sharing a `UdpSocket`

Since all I/O methods on `UdpSocket` take `&self`, you can safely wrap the socket in an `Arc<UdpSocket>` to share it across multiple tasks for concurrent sending and receiving without needing a `Mutex`.

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

## Unix Domain Sockets

Available on Unix-like platforms, Unix Domain Sockets (UDS) facilitate inter-process communication (IPC) on the same host. They behave similarly to TCP/UDP sockets but use filesystem paths instead of IP addresses and ports. They can offer better performance and security for local communication.

*   **`UnixListener` and `UnixStream`**: Provide a connection-oriented stream socket, analogous to `TcpListener` and `TcpStream`.
*   **`UnixDatagram`**: Provides a connectionless datagram socket for sending individual messages, analogous to `UdpSocket`.
*   **`tokio::net::unix::pipe`**: Provides functionality for FIFO pipes.

## Platform-Specific Networking

Tokio also includes networking types that are specific to certain operating systems.

*   **Windows Named Pipes**: On Windows, `tokio::net::windows::named_pipe` provides an API for communicating with named pipes, a common mechanism for IPC on that platform.

## Utilities

### `lookup_host`

To perform an asynchronous DNS query, you can use the `lookup_host` function. It resolves a host string to one or more socket addresses.

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

Many networking functions, like `TcpStream::connect` and `UdpSocket::bind`, are generic over the `ToSocketAddrs` trait. This allows you to pass various address types, such as a `SocketAddr`, an `(IpAddr, u16)` tuple, or a string like `"127.0.0.1:8080"`, which will be resolved for you.
