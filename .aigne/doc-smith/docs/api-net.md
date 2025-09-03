# Networking

This module provides asynchronous TCP, UDP, and Unix socket bindings for Tokio. These components are designed to feel similar to their counterparts in the Rust standard library but are non-blocking and integrate with the Tokio runtime.

## Overview of Networking Primitives

Tokio's `net` module is organized by protocol, offering a range of types for different communication needs.

```d2
direction: down

"tokio::net": {
  shape: package
  grid-columns: 3
  grid-gap: 50

  "TCP (Connection-Oriented)": {
    shape: rectangle
    "TcpListener": "Accepts incoming connections"
    "TcpStream": "Represents a TCP stream"
    "TcpSocket": "Low-level socket configuration"
  }

  "UDP (Connectionless)": {
    shape: rectangle
    "UdpSocket": "Sends/receives datagrams"
  }

  "IPC (Unix-like Systems)": {
    shape: rectangle
    "UnixListener": "Stream-based listener"
    "UnixStream": "Stream connection"
    "UnixDatagram": "Datagram socket"
    "Pipes": "FIFO pipes"
  }
}
```

Below is a quick guide to the primary components available for building your networking protocols.

<x-cards data-columns="2">
  <x-card data-title="TCP" data-icon="lucide:server">
    For reliable, stream-oriented communication. Includes `TcpListener` for accepting connections and `TcpStream` for data transfer.
  </x-card>
  <x-card data-title="UDP" data-icon="lucide:send">
    For connectionless, datagram-based communication. `UdpSocket` can be used to send and receive data to and from multiple remotes.
  </x-card>
  <x-card data-title="Unix Domain Sockets" data-icon="lucide:box">
    For inter-process communication (IPC) on Unix-like systems. Provides stream (`UnixListener`, `UnixStream`) and datagram (`UnixDatagram`) variants.
  </x-card>
  <x-card data-title="Pipes" data-icon="lucide:workflow">
    For platform-specific pipe-based communication, such as FIFO pipes on Unix and Named Pipes on Windows.
  </x-card>
</x-cards>

## TCP (Transmission Control Protocol)

TCP provides a reliable, ordered, and error-checked stream of bytes between applications. It's the foundation for many internet protocols like HTTP and FTP.

-   **`TcpListener`**: An asynchronous listener for accepting incoming TCP connections.
-   **`TcpStream`**: Represents a TCP connection between a local and remote socket. It can be split into owned read and write halves.
-   **`TcpSocket`**: A lower-level utility for creating and configuring a TCP socket before it starts listening or connecting.

### Example: TCP Echo Server

Here is a simple example of a TCP echo server that accepts a connection and echoes back any data it receives.

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
                    // Return value of `Ok(0)` signifies that the remote has
                    // closed the connection.
                    Ok(0) => return,
                    Ok(n) => {
                        // Copy the data back to the socket
                        if socket.write_all(&buf[..n]).await.is_err() {
                            // Unexpected error. Just exit the task.
                            return;
                        }
                    }
                    Err(_) => {
                        // Unexpected error. Just exit the task.
                        return;
                    }
                }
            }
        });
    }
}
```

## UDP (User Datagram Protocol)

UDP is a connectionless protocol that offers a simple but unreliable datagram service. It's suitable for applications where speed is critical and some data loss is acceptable, like gaming or voice chat.

The `UdpSocket` type can be used in two primary ways:

1.  **One-to-many**: A single socket bound to an address can send and receive datagrams to and from many different remote peers using `send_to` and `recv_from`.
2.  **One-to-one**: A socket can be `connect`ed to a single remote peer, allowing the use of `send` and `recv` for communication, which filters incoming packets to only that address.

### Example: One-to-Many UDP Echo Server

This server binds to an address and echoes back any received datagram to its original sender.

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

### Sharing a `UdpSocket`

Because its methods take `&self` instead of `&mut self`, a `UdpSocket` can be safely shared across multiple tasks for concurrent reads and writes by wrapping it in an `Arc<UdpSocket>`.

## Unix Domain Sockets (UDS)

Available only on Unix-like systems, Unix Domain Sockets facilitate inter-process communication (IPC) on the same machine. They behave similarly to TCP streams but use filesystem paths for addressing instead of IP addresses and ports.

-   **`UnixListener`** and **`UnixStream`**: Provide a stream-oriented connection, similar to TCP.
-   **`UnixDatagram`**: Provides a datagram-based socket, similar to UDP.
-   **`UnixSocket`**: A lower-level utility for creating and configuring a Unix socket.

## Platform-Specific Networking

Tokio also includes modules for networking features specific to certain operating systems.

-   **Windows**: The `tokio::net::windows` module provides support for Named Pipes.
-   **Unix**: The `tokio::net::unix` module contains UDS types as well as support for FIFO pipes via `tokio::net::unix::pipe`.

## Utilities

### DNS Resolution

The `lookup_host` function provides an asynchronous way to perform DNS resolution.

### Address Handling

The `ToSocketAddrs` trait is used by networking types to convert various address representations into one or more `SocketAddr` instances.

---

With these networking primitives, you can build a wide variety of applications. For more hands-on examples, check out the [Examples](./examples.md) section. To understand the underlying I/O operations, see the [I/O API Reference](./api-io.md).