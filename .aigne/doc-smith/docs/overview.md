# Overview

Tokio is an event-driven, non-blocking I/O platform for writing asynchronous applications with the Rust programming language. It provides the runtime and tools needed to build fast, reliable, and scalable network applications without compromising on speed.

At its core, Tokio is built on a few key principles:

*   **Fast**: Zero-cost abstractions provide bare-metal performance.
*   **Reliable**: It leverages Rust's ownership, type system, and concurrency model to reduce bugs and ensure thread safety.
*   **Scalable**: A minimal footprint, natural handling of backpressure, and cancellation make it suitable for applications of any size.

## Core Components

Tokio provides a few major components that form the foundation for asynchronous applications:

*   **A Multi-threaded Task Scheduler**: A work-stealing scheduler for executing asynchronous tasks efficiently across multiple CPU cores.
*   **An I/O Driver (Reactor)**: Backed by the operating system's event queue (like epoll, kqueue, or IOCP), this component drives asynchronous I/O operations.
*   **Asynchronous APIs**: A rich set of APIs for asynchronous tasks, I/O, timing, and synchronization, including TCP/UDP sockets, filesystem operations, timers, and channels.

Here is a high-level view of how these components interact:

```d2
direction: down

"User Application": {
  shape: rectangle
  label: "Your Application Code"
}

"Tokio Runtime": {
  shape: package
  label: "Tokio Runtime"
  grid-columns: 1

  "Scheduler": {
    label: "Task Scheduler"
    shape: rectangle
  }

  "Driver": {
    label: "I/O Driver & Timer"
    shape: rectangle
  }
}

"OS": {
  shape: cylinder
  label: "Operating System\n(Event Queue, Sockets, Files)"
}

"User Application" -> "Tokio Runtime"."Scheduler": "Spawns async tasks"
"Tokio Runtime"."Scheduler" <-> "Tokio Runtime"."Driver": "Polls tasks for readiness"
"Tokio Runtime"."Driver" <-> "OS": "Registers I/O events"

```

## A Quick Look

To get a feel for what Tokio code looks like, here is a basic TCP echo server. It listens for incoming connections and simply writes any data it receives back to the client.

First, add Tokio as a dependency with the `full` feature flag in your `Cargo.toml`:

```toml
tokio = { version = "1", features = ["full"] }
```

Then, you can write the server logic:

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

This example showcases several key parts of Tokio: the `#[tokio::main]` macro to start the runtime, asynchronous TCP sockets (`TcpListener`, `TcpStream` via `socket`), and spawning concurrent tasks with `tokio::spawn`.

## What's Next?

<x-cards>
  <x-card data-title="Getting Started" data-icon="lucide:rocket" data-href="/getting-started">
    Follow a step-by-step guide to set up your first Tokio project and run a working example.
  </x-card>
  <x-card data-title="Core Concepts" data-icon="lucide:book-open" data-href="/concepts">
    Dive deeper into the fundamental concepts of Tokio, such as tasks, I/O, and the runtime.
  </x-card>
</x-cards>