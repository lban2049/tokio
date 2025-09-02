# Networking

This module provides asynchronous TCP, UDP, and Unix socket bindings for Tokio, enabling the development of high-performance networking applications. These components are designed to be analogous to their counterparts in the standard library but operate in a non-blocking manner, integrating seamlessly with the Tokio runtime.

## Overview

Tokio's networking primitives cover the most common protocols and inter-process communication (IPC) mechanisms. Here's a visual breakdown of the key components and their relationships:

```d2
direction: down

"Network Application": {
  "TCP Server": {
    "TcpListener" -> "TcpStream" : accepts
  }
  
  "TCP Client": {
    "TcpStream"
  }

  "UDP Peer": {
    "UdpSocket"
  }
  
  "Unix Domain Server": {
    "UnixListener" -> "UnixStream" : accepts
  }

  "Unix Domain Client": {
    "UnixStream"
  }
}

"Remote Peers": {
  "Remote TCP Peer 1"
  "Remote TCP Peer 2"
  "Remote UDP Peer A"
  "Remote UDP Peer B"
}

"Local Processes": {
  "Process A"
  "Process B"
}

"Network Application"."TCP Client"."TcpStream" <-> "Remote Peers"."Remote TCP Peer 1": TCP Connection
"Network Application"."TCP Server"."TcpStream" <-> "Remote Peers"."Remote TCP Peer 2": TCP Connection
"Network Application"."UDP Peer"."UdpSocket" <-> "Remote Peers"."Remote UDP Peer A": UDP Datagrams
"Network Application"."UDP Peer"."UdpSocket" <-> "Remote Peers"."Remote UDP Peer B": UDP Datagrams
"Network Application"."Unix Domain Client"."UnixStream" <-> "Local Processes"."Process A": IPC
"Network Application"."Unix Domain Server"."UnixStream" <-> "Local Processes"."Process B": IPC
```

## Core Components

The `tokio::net` module is organized by protocol. Here are the primary types you will work with:

<x-cards data-columns="2">
  <x-card data-title="TCP Sockets" data-icon="lucide:arrow-right-left">
    Provides `TcpListener` to accept incoming stream connections and `TcpStream` for communication over TCP. Ideal for reliable, connection-oriented protocols like HTTP.
  </x-card>
  <x-card data-title="UDP Sockets" data-icon="lucide:move-diagonal">
    Provides `UdpSocket` for connectionless, datagram-based communication over UDP. Suitable for applications where speed is preferred over reliability, like gaming or streaming.
  </x-card>
  <x-card data-title="Unix Domain Sockets" data-icon="lucide:server">
    Stream and datagram sockets for inter-process communication (IPC) on Unix-based systems. Includes `UnixListener`, `UnixStream`, and `UnixDatagram`.
  </x-card>
  <x-card data-title="Pipes" data-icon="lucide:pipeline">
    Platform-specific pipes for IPC. This includes `tokio::net::unix::pipe` for FIFO pipes on Unix and `tokio::net::windows::named_pipe` on Windows.
  </x-card>
</x-cards>

## TCP (Transmission Control Protocol)

TCP provides reliable, ordered, and error-checked delivery of a stream of bytes. Tokio offers two primary types for TCP networking:

-   **`TcpListener`**: An asynchronous version of `std::net::TcpListener`. It is used to listen for and accept incoming TCP connections.
-   **`TcpStream`**: An asynchronous TCP stream between a local and a remote socket. It implements `AsyncRead` and `AsyncWrite` for sending and receiving data.
-   **`TcpSocket`**: A lower-level socket that allows for configuration (e.g., setting `SO_REUSEADDR`) before it is used to connect or listen.

## UDP (User Datagram Protocol)

UDP is a connectionless protocol that provides a datagram-based communication service. It prioritizes speed and low latency over reliability.

### Usage Patterns

A `UdpSocket` can be used in two primary ways:

1.  **One-to-Many (Unconnected)**: Bind a socket to an address and use `send_to` and `recv_from` to communicate with multiple remote peers. This is the typical use case for a server that handles many clients.

    *Example: An echo server handling multiple clients.*
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

2.  **One-to-One (Connected)**: Associate a socket with a single remote address using `connect`. Once connected, you can use the more convenient `send` and `recv` methods, and the socket will only send to and receive from that specific peer.

    *Example: An echo client connected to a single server.*
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

### Concurrency and Splitting

Unlike `TcpStream`, `UdpSocket` does not have a `split` method. However, since its send and receive methods take `&self`, a single socket can be shared across multiple tasks using `Arc<UdpSocket>`.

*Example: Concurrent sending and receiving using an `Arc`.*
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

## Unix Domain Sockets

For inter-process communication (IPC) on Unix-like systems, Tokio provides asynchronous Unix domain sockets. They behave like TCP sockets but operate on a local filesystem path instead of an IP address and port.

-   **`UnixListener`** and **`UnixStream`**: For connection-oriented, stream-based communication, similar to TCP.
-   **`UnixDatagram`**: For connectionless, datagram-based communication, similar to UDP.
-   **`UnixSocket`**: A lower-level socket for advanced configuration.

These types are only available on Unix platforms.

## Custom I/O Resources

For I/O resources not natively available in `tokio::net`, such as raw sockets or other platform-specific handles, you can use [`AsyncFd`](./api-io.md) to integrate them with the Tokio runtime. This allows you to perform non-blocking I/O operations on any file descriptor that can be monitored by the operating system's event queue (like epoll, kqueue, or IOCP).

---

Now that you have an overview of Tokio's networking capabilities, you can explore the [asynchronous I/O traits and helpers](./api-io.md) that power them, or dive into [synchronization primitives](./api-sync.md) for managing state in your network application.