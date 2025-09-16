# Networking

Tokio provides asynchronous, non-blocking networking primitives for TCP, UDP, and Unix Domain Sockets. These components are designed to be efficient, reliable, and easy to use, allowing you to build high-performance network applications.

This guide covers the fundamental networking types available in Tokio. For more specialized I/O resources, you can use `AsyncFd`.

## TCP (Transmission Control Protocol)

TCP is a connection-oriented protocol that provides reliable, ordered, and error-checked delivery of a stream of bytes. Tokio offers two primary types for TCP communication: `TcpListener` for servers and `TcpStream` for clients.

### `TcpListener`

A `TcpListener` is a TCP socket server used to accept incoming connections. You can create a listener by binding it to a local address.

```rust TCP Server Example icon=logos:rust
use tokio::net::TcpListener;
use std::io;

async fn process_socket<T>(socket: T) {
    // Business logic for handling the connection
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

The `accept` method waits for a new connection and returns a `TcpStream` along with the client's address.

### `TcpStream`

A `TcpStream` represents a TCP connection between a local and a remote socket. It can be created by connecting to a remote endpoint or by accepting a connection from a `TcpListener`.

```rust TCP Client Example icon=logos:rust
use tokio::net::TcpStream;
use tokio::io::AsyncWriteExt;
use std::error::Error;

#[tokio::main]
asyn fn main() -> Result<(), Box<dyn Error>> {
    // Connect to a peer
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;

    // Write some data
    stream.write_all(b"hello world!").await?;

    Ok(())
}
```

`TcpStream` implements the `AsyncRead` and `AsyncWrite` traits, allowing you to read from and write to the stream asynchronously.

### Advanced Configuration with `TcpSocket`

For cases where you need to configure a socket before it is bound or connected, Tokio provides the `TcpSocket` type. This allows you to set socket options like `SO_REUSEADDR`.

```rust Configuring a TCP Socket icon=logos:rust
use tokio::net::TcpSocket;
use std::io;

#[tokio::main]
asyn fn main() -> io::Result<()> {
    let addr = "127.0.0.1:8080".parse().unwrap();

    let socket = TcpSocket::new_v4()?;
    
    // On Unix, this allows the socket to be quickly rebound.
    // On Windows, it allows rebinding to an in-use socket.
    socket.set_reuseaddr(true)?;
    
    socket.bind(addr)?;

    let listener = socket.listen(1024)?;
    # drop(listener);

    Ok(())
}
```

## UDP (User Datagram Protocol)

UDP is a connectionless protocol that provides a datagram-based service. Unlike TCP, it does not guarantee delivery, ordering, or error checking. Tokio's `UdpSocket` can be used to send and receive UDP datagrams.

### `UdpSocket`

A `UdpSocket` can be used in two main ways:

1.  **One-to-many**: Bind the socket and use `send_to` and `recv_from` to communicate with multiple remote peers.
2.  **One-to-one**: `connect` the socket to a single remote address and use `send` and `recv` for communication.

Here is an example of a simple UDP echo server:

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

Since its methods take `&self`, a `UdpSocket` can be wrapped in an `Arc<UdpSocket>` to be shared across multiple tasks for concurrent sending and receiving.

## Unix Domain Sockets (UDS)

For inter-process communication (IPC) on Unix-like systems, Tokio provides asynchronous Unix Domain Sockets. They behave similarly to their TCP/UDP counterparts but operate on a local filesystem path instead of an IP address and port.

-   **`UnixListener`** and **`UnixStream`**: Provide a reliable, stream-oriented connection, similar to TCP.
-   **`UnixDatagram`**: Provides a connectionless, datagram-oriented service, similar to UDP.

## Platform-Specific Networking

Tokio also includes support for other platform-specific networking types:

-   **`tokio::net::unix::pipe`**: For working with FIFO pipes on Unix systems.
-   **`tokio::net::windows::named_pipe`**: For working with Named Pipes on Windows.

These primitives enable you to build a wide range of robust and efficient network services. 

Next, let's explore how to perform [asynchronous Filesystem operations](./io-fs.md).