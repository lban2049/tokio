# Overview

Tokio is a runtime for writing reliable, asynchronous, and slim applications with the Rust programming language. It is an event-driven, non-blocking I/O platform that provides the building blocks for writing network applications without compromising speed, reliability, or scalability.

<x-cards>
  <x-card data-title="Fast" data-icon="lucide:rocket">
    Tokio's zero-cost abstractions give you bare-metal performance, ensuring your application runs as efficiently as possible.
  </x-card>
  <x-card data-title="Reliable" data-icon="lucide:shield-check">
    By leveraging Rust's ownership, type system, and concurrency model, Tokio helps you write thread-safe code and reduce common programming bugs.
  </x-card>
  <x-card data-title="Scalable" data-icon="lucide:line-chart">
    With a minimal footprint, Tokio handles backpressure and cancellation naturally, allowing your applications to scale efficiently under load.
  </x-card>
</x-cards>

## Core Components

At a high level, Tokio provides a few major components that form the foundation for asynchronous applications in Rust.

```d2
direction: down

"Your Application Code" -> "Tokio Runtime": "Runs on"

"Tokio Runtime": {
  shape: cloud
  "Task Scheduler": "Manages and executes tasks"
  "I/O Reactor": "Interfaces with OS events"
  "Timer": "Provides time-based events"
}

"Tokio Runtime"."I/O Reactor" <-> "OS Event Queue (epoll, kqueue, IOCP)": "Non-blocking I/O"
```

*   **A Runtime for Asynchronous Code**: Tokio provides a multi-threaded, work-stealing task scheduler for executing asynchronous tasks. It includes an I/O driver backed by the operating system's event queue (e.g., epoll, kqueue, IOCP) and a high-performance timer.

*   **Tools for Asynchronous Tasks**: It offers a rich set of tools for working with asynchronous tasks, including [synchronization primitives](./concepts-synchronization.md) like channels and mutexes, and utilities for managing [time](./concepts-timers.md) such as sleeps, intervals, and timeouts.

*   **APIs for Asynchronous I/O**: A comprehensive set of APIs for performing non-blocking [I/O](./concepts-io.md), including TCP, UDP, and Unix sockets, as well as filesystem, process, and signal management.

## A Quick Example

Here is a basic TCP echo server that demonstrates some of Tokio's core features. First, add Tokio as a dependency with the `full` feature flag in your `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

Then, you can write the server code:

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
This example binds a `TcpListener` to an address, and for each incoming connection, it spawns a new asynchronous task to handle reading data from the socket and writing it back.

## Next Steps

This overview provides a high-level look at Tokio's purpose and components. To start building your own applications, head over to the [Getting Started](./getting-started.md) guide.