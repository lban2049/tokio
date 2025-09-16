# Overview

Tokio is a runtime for writing reliable, asynchronous, and slim applications with the Rust programming language. It provides an event-driven, non-blocking I/O platform that's essential for building high-performance network applications.

<x-cards data-columns="3">
  <x-card data-title="Fast" data-icon="lucide:rocket">
    Tokio's zero-cost abstractions give you bare-metal performance, ensuring your applications are fast and efficient.
  </x-card>
  <x-card data-title="Reliable" data-icon="lucide:shield-check">
    By leveraging Rust's ownership, type system, and concurrency model, Tokio helps reduce bugs and ensure thread safety.
  </x-card>
  <x-card data-title="Scalable" data-icon="lucide:scaling">
    With a minimal footprint, Tokio handles backpressure and cancellation naturally, allowing your applications to scale effectively.
  </x-card>
</x-cards>

## Core Components

At a high level, Tokio provides a few major components that are the building blocks for asynchronous applications:

*   **Tools for Asynchronous Tasks**: Tokio provides a powerful toolkit for managing concurrent operations. This includes spawning tasks, using synchronization primitives like channels and mutexes, and handling time-based operations such as sleeps, intervals, and timeouts. These are crucial for managing control flow in an async environment. For more details, see the [Tasks & Scheduling](./tasks-scheduling.md) guide.

*   **APIs for Asynchronous I/O**: Perform non-blocking input and output with a comprehensive set of APIs. Tokio includes support for TCP, UDP, and Unix Domain Sockets, as well as asynchronous filesystem operations, child process management, and OS signal handling. Dive deeper into these features in the [Asynchronous I/O](./io.md) section.

*   **The Tokio Runtime**: The runtime is the engine that executes your asynchronous code. It includes a multi-threaded, work-stealing task scheduler, an I/O driver backed by the operating system's event queue (like epoll, kqueue, or IOCP), and a high-performance timer. Learn how to configure and manage it in [The Runtime](./tasks-scheduling-runtime.md) documentation.

## A Quick Example

Here is a basic TCP echo server that demonstrates how these components work together. It listens for incoming connections and simply writes any data it receives back to the client.

```rust A Simple TCP Echo Server icon=logos:rust
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

This example showcases several key Tokio features:

-   `#[tokio::main]`: A macro to start the Tokio runtime and execute the `async` main function.
-   `TcpListener`: An asynchronous TCP listener to accept incoming connections.
-   `tokio::spawn`: A function to spawn a new asynchronous task for each connection, allowing the server to handle multiple clients concurrently.
-   `AsyncReadExt` and `AsyncWriteExt`: Traits that provide asynchronous `read` and `write` methods on the socket.

## Next Steps

Now that you have a high-level understanding of what Tokio offers, you're ready to start building. Head over to the [Getting Started](./getting-started.md) guide for a step-by-step tutorial on setting up your first Tokio application.