# Examples

This section provides a collection of working code examples that demonstrate various Tokio features and use cases. These examples are designed to be practical and can serve as a starting point for your own applications.

<x-cards data-columns="1">
  <x-card data-title="TCP Echo Server" data-icon="lucide:server">
    A fundamental example of an asynchronous TCP server that echoes back any data it receives from a client. This is a great starting point for understanding basic network I/O.
  </x-card>
  <x-card data-title="Mini-Redis" data-icon="lucide:database">
    A larger, 'real-world' example of a simplified Redis server. It demonstrates structuring a complete application, managing shared state, and handling client commands.
  </x-card>
</x-cards>

## TCP Echo Server

A simple TCP echo server is a classic way to demonstrate asynchronous I/O. The server listens on a socket, accepts incoming connections, and for each connection, it reads data and writes it back to the same socket.

### Dependencies

To run this example, you need to enable the necessary features in your `Cargo.toml` file. The `full` feature flag is the easiest way to get started.

```toml
tokio = { version = "1", features = ["full"] }
```

### Server Code

The following code implements the complete echo server:

```rust,no_run
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write the data back.
            loop {
                let n = match socket.read(&mut buf).await {
                    // socket closed
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // Write the data back
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

### How It Works

1.  **`TcpListener::bind`**: The server starts by binding a `TcpListener` to a local address (`127.0.0.1:8080`). The `.await` keyword pauses execution until the listener is successfully bound.
2.  **`listener.accept()`**: The server enters a loop, calling `listener.accept().await` to wait for incoming connections. When a client connects, `accept` returns a new `TcpSocket` and the client's address.
3.  **`tokio::spawn`**: To handle multiple clients concurrently, a new task is spawned for each accepted connection. The `socket` is moved into this new task.
4.  **Read/Write Loop**: Inside the spawned task, a loop continuously reads data from the socket into a buffer. If the read is successful (`Ok(n)` where `n > 0`), the same data (`&buf[0..n]`) is written back to the socket. If the client closes the connection, `read` returns `Ok(0)`, and the task terminates.

## Advanced Example: Mini-Redis

For a more substantial, real-world example, see the [mini-redis repository](https://github.com/tokio-rs/mini-redis/). This project is an asynchronous, simplified implementation of a Redis server and client built with Tokio.

It's an excellent resource for learning how to structure a larger Tokio application and demonstrates concepts such as:

- Managing shared, mutable state across tasks.
- Framing, which is the process of parsing a stream of bytes into a sequence of messages.
- Graceful shutdown.
- Implementing both the client and server sides of a protocol.

## More Examples

More examples covering a wide range of Tokio's features can be found in the [examples directory of the Tokio GitHub repository](https://github.com/tokio-rs/tokio/tree/master/examples). These provide concise demonstrations of specific APIs and are a valuable resource for learning.