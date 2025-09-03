# Overview

Tokio is a runtime for writing reliable, asynchronous, and slim applications with the Rust programming language. It is an event-driven, non-blocking I/O platform that provides the tools needed to build fast, scalable, and safe network applications.

<x-cards data-columns="3">
  <x-card data-title="Fast" data-icon="lucide:rocket">
    Tokio's zero-cost abstractions provide bare-metal performance, ensuring your application is fast and efficient.
  </x-card>
  <x-card data-title="Reliable" data-icon="lucide:shield-check">
    By leveraging Rust's ownership, type system, and concurrency model, Tokio helps reduce bugs and ensure thread safety.
  </x-card>
  <x-card data-title="Scalable" data-icon="lucide:scaling">
    Tokio has a minimal footprint and naturally handles backpressure and cancellation, allowing you to build scalable services.
  </x-card>
</x-cards>

## Core Components

At a high level, Tokio provides a few major components that form the foundation for asynchronous applications:

```d2
direction: down

"Application Logic" {
  shape: rectangle
}

"Tokio Runtime" {
  shape: package
  grid-columns: 1

  "Core APIs" {
    shape: rectangle
    grid-columns: 2

    "Task Management" {
      label: "Tasks & Synchronization"
      shape: class
    }
    "I/O Primitives" {
      label: "Asynchronous I/O"
      shape: class
    }
    "Time Utilities" {
      label: "Timers & Timeouts"
      shape: class
    }
  }

  "Internal Engine" {
    shape: rectangle
    grid-columns: 2
    
    "Scheduler" {
      label: "Task Scheduler\n(Work-stealing)"
      shape: hexagon
    }

    "Driver" {
      label: "I/O Driver\n(Reactor)"
      shape: hexagon
    }
  }

  "OS" {
      label: "Operating System Events\n(epoll, kqueue, IOCP)"
      shape: cylinder
  }
}

"Application Logic" -> "Tokio Runtime"."Core APIs": "Uses"
"Tokio Runtime"."Core APIs" -> "Tokio Runtime"."Internal Engine": "Relies on"
"Tokio Runtime"."Internal Engine".Driver -> "OS": "Backed by"

```

*   **Tools for working with asynchronous tasks**: This includes primitives for spawning and managing tasks, channels and mutexes for synchronization, and utilities like timeouts and sleeps for handling time.
*   **APIs for asynchronous I/O**: Tokio provides non-blocking sockets for TCP and UDP, as well as utilities for filesystem operations, process management, and signal handling.
*   **A runtime for executing asynchronous code**: This includes a multi-threaded, work-stealing task scheduler, an I/O driver (also called a reactor) backed by the operating system's event queue (e.g., epoll, kqueue, IOCP), and a high-performance timer.

## A Quick Look

Here is a basic TCP echo server built with Tokio. First, add Tokio as a dependency with the `full` feature flag in your `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

Then, you can write the server logic in your `main.rs` file:

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

This example demonstrates how Tokio's components work together: the `#[tokio::main]` macro sets up the runtime, `TcpListener` provides an asynchronous network socket, and `tokio::spawn` schedules a new task to handle each incoming connection concurrently.

## Next Steps

This overview has introduced the core ideas behind Tokio. To start building your first application, proceed to the Getting Started guide.

<x-card data-title="Getting Started" data-icon="lucide:play-circle" data-href="/getting-started" data-cta="Start Building">
  A step-by-step guide to setting up a new Tokio project, including installation and a simple, working TCP echo server example.
</x-card>
