# Overview

Tokio is a runtime for writing reliable, asynchronous, and slim network applications with the Rust programming language without compromising speed.

It is an event-driven, non-blocking I/O platform for writing asynchronous applications. At a high level, it provides a few major components that are essential for building robust and performant systems.

<x-cards data-columns="3">
  <x-card data-title="Fast" data-icon="lucide:gauge-circle">
    Tokio's zero-cost abstractions give you bare-metal performance, ensuring your applications are as fast as possible.
  </x-card>
  <x-card data-title="Reliable" data-icon="lucide:shield-check">
    By leveraging Rust's ownership, type system, and concurrency model, Tokio helps you reduce bugs and ensure thread safety.
  </x-card>
  <x-card data-title="Scalable" data-icon="lucide:bar-chart-big">
    Tokio has a minimal footprint and handles backpressure and cancellation naturally, allowing your applications to scale efficiently.
  </x-card>
</x-cards>

## Core Components

Tokio is built around a few key components that provide the foundation for asynchronous applications:

*   **Tools for Asynchronous Tasks**: Primitives for managing task lifecycle, including synchronization and communication between tasks (channels, mutexes), and utilities for handling time (timeouts, sleeps, intervals).
*   **Asynchronous I/O APIs**: A rich set of APIs for non-blocking I/O, including TCP and UDP sockets, filesystem operations, and process and signal management.
*   **The Runtime**: A runtime for executing asynchronous code, which includes a multi-threaded, work-stealing task scheduler, an I/O driver backed by the operating system's event queue (like epoll, kqueue, or IOCP), and a high-performance timer.

## A Tour of Tokio

Tokio's functionality is organized into several modules, each serving a distinct purpose. Here’s a brief tour of the major APIs.

### Working With Tasks

Asynchronous programs in Rust are built around lightweight, non-blocking units of execution called tasks. Tokio provides powerful tools for managing them.

- **[`tokio::task`](./api-task.md)**: Contains the core tools for working with tasks, such as the `spawn` function for scheduling new tasks on the runtime.
- **[`tokio::sync`](./api-sync.md)**: Provides synchronization primitives for tasks, including channels (`oneshot`, `mpsc`, `watch`, `broadcast`) for communication and a non-blocking `Mutex` for protecting shared data.
- **[`tokio::time`](./api-time.md)**: Offers utilities for tracking time, enabling you to set timeouts, sleep for a specified duration, or repeat operations at intervals.

### Asynchronous I/O

Tokio provides a comprehensive suite of modules for performing asynchronous input and output operations.

- **[`tokio::io`](./api-io.md)**: The foundation of Tokio's I/O, providing the core `AsyncRead` and `AsyncWrite` traits.
- **[`tokio::net`](./api-net.md)**: Contains non-blocking versions of TCP, UDP, and Unix Domain Sockets for network programming.
- **[`tokio::fs`](./api-fs.md)**: Offers asynchronous APIs for filesystem I/O, similar to the standard library's `std::fs`.
- **[`tokio::signal`](./api-signal.md)** and **[`tokio::process`](./api-process.md)**: Allow for asynchronous handling of OS signals and management of child processes.

### The Runtime

The Tokio runtime is responsible for executing asynchronous tasks. While most applications can start with the simple `#[tokio::main]` macro, the [`tokio::runtime`](./api-runtime.md) module provides powerful APIs for fine-grained configuration and management of the runtime when more control is needed.

### CPU-bound tasks and blocking code

Tokio is designed for I/O-bound applications and uses a few threads to handle many concurrent tasks. Code that performs long-running, CPU-intensive computations without awaiting can block a thread, preventing other tasks from running. To handle this, Tokio provides `spawn_blocking`, which moves the blocking or CPU-bound computation to a dedicated thread pool, ensuring the main async scheduler remains responsive.

```rust A blocking task example icon=logos:rust
#[tokio::main]
async fn main() {
    // This is running on a core thread.

    let blocking_task = tokio::task::spawn_blocking(|| {
        // This is running on a blocking thread.
        // Blocking here is ok.
    });

    // We can wait for the blocking task to complete.
    blocking_task.await.unwrap();
}
```

## Ready to Dive In?

This overview has introduced the core ideas behind Tokio. The best way to learn is by doing. Head over to our [Getting Started](./getting-started.md) guide to set up your first Tokio application in minutes.