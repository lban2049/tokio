# Networking

This module provides asynchronous TCP, UDP, and Unix socket bindings for Tokio. It contains networking types similar to those in the standard library, designed for building high-performance networking protocols.

## Organization

The `tokio::net` module is organized by protocol, with specific types for handling different kinds of network communication. The primary components are:

```d2
direction: down

tokio-net: {
  shape: package
  label: "tokio::net"
  grid-columns: 2
  grid-gap: 50

  TCP: {
    shape: package
    "TcpListener": "Accepts incoming TCP connections"
    "TcpStream": "A TCP stream between two peers"
    "TcpSocket": "For configuring TCP sockets"
  }

  UDP: {
    shape: package
    "UdpSocket": "A UDP socket"
  }

  Unix: {
    shape: package
    label: "Unix (Unix-only)"
    "UnixListener": "Accepts Unix stream connections"
    "UnixStream": "A Unix stream between two peers"
    "UnixDatagram": "A Unix datagram socket"
    "pipe": "FIFO pipes"
  }

  Windows: {
    shape: package
    label: "Windows (Windows-only)"
    "named_pipe": "Named Pipes"
  }

  "Utilities": {
    shape: package
    "lookup_host": "DNS resolution"
    "ToSocketAddrs": "Trait for address conversion"
  }
}
```

<x-cards data-columns="2">
  <x-card data-title="TCP" data-icon="lucide:arrow-right-left">
    `TcpListener` and `TcpStream` provide functionality for reliable, connection-oriented communication over TCP.
  </x-card>
  <x-card data-title="UDP" data-icon="lucide:move-diagonal">
    `UdpSocket` provides functionality for connectionless communication over UDP.
  </x-card>
  <x-card data-title="Unix Sockets" data-icon="lucide:server">
    `UnixListener`, `UnixStream`, and `UnixDatagram` enable communication over Unix Domain Sockets on Unix-like systems.
  </x-card>
  <x-card data-title="Platform-Specific" data-icon="lucide:box">
    Includes Unix pipes (`tokio::net::unix::pipe`) and Windows Named Pipes (`tokio::net::windows::named_pipe`).
  </x-card>
</x-cards>

For I/O resources not directly available in `tokio::net`, you can use `AsyncFd` from the `tokio::io::unix` module.

## UdpSocket

A UDP socket is connectionless, meaning it can communicate with many different remote peers without establishing a dedicated connection. There are two primary ways to use `UdpSocket`:

1.  **One-to-many**: Use `bind` to create a socket, then use `send_to` and `recv_from` to communicate with various remote addresses.
2.  **One-to-one**: Use `connect` to associate the socket with a single remote address, allowing you to use the simpler `send` and `recv` methods.

Unlike TCP streams, `UdpSocket` does not have a `split` method. Instead, it can be wrapped in an `Arc<UdpSocket>` and cloned to be shared across multiple tasks, as its methods take `&self`.

### Example: One-to-Many Echo Server

This example demonstrates a simple echo server that receives a datagram from any client and sends it back to the original address.

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

### Example: One-to-One (Connected) Echo Server

After binding, you can `connect` the socket to a specific peer. This filters incoming packets to only that address and sets it as the default destination for sends.

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

### Example: Sharing a Socket with `Arc`

To handle concurrent sending and receiving, you can wrap the `UdpSocket` in an `Arc` and clone it for different tasks. This example uses a channel to pass received messages to a sending task.

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