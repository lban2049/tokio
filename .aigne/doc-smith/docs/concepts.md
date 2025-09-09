# Core Concepts

Tokio is an event-driven, non-blocking I/O platform for writing asynchronous applications with Rust. To use it effectively, it's important to understand its fundamental components. This section provides a high-level overview of the key concepts that form the foundation of Tokio, giving you a solid base for building advanced, high-performance applications.

Each concept has a dedicated page for a more in-depth exploration. We recommend reading them in order to build your understanding progressively.

<x-cards data-columns="2">
  <x-card data-title="Tasks & Scheduling" data-icon="lucide:box" data-href="/concepts/tasks">
    Asynchronous programs in Rust are built around lightweight, non-blocking units of execution called tasks. Learn how Tokio spawns, runs, and manages these tasks.
  </x-card>
  <x-card data-title="Asynchronous I/O" data-icon="lucide:arrow-right-left" data-href="/concepts/io">
    Discover Tokio's non-blocking I/O primitives for networking (TCP, UDP), filesystem operations, and interacting with the operating system asynchronously.
  </x-card>
  <x-card data-title="Synchronization" data-icon="lucide:link" data-href="/concepts/synchronization">
    Explore primitives for communicating and sharing data between tasks, including channels (oneshot, mpsc, watch), mutexes, and barriers.
  </x-card>
  <x-card data-title="Timers" data-icon="lucide:timer" data-href="/concepts/timers">
    Understand the utilities for tracking time and scheduling work, such as setting timeouts, sleeping, or repeating operations at an interval.
  </x-card>
  <x-card data-title="The Runtime" data-icon="lucide:cpu" data-href="/concepts/runtime">
    Delve into the engine that executes asynchronous code, featuring a task scheduler, an I/O driver, and a high-performance timer.
  </x-card>
</x-cards>

## CPU-bound tasks and blocking code

Tokio excels at I/O-bound tasks by concurrently running many of them on a few threads. This is possible because I/O-bound tasks yield control back to the scheduler when they are waiting for I/O, allowing another task to run. However, code that performs long-running, CPU-intensive computations without awaiting can block the thread, preventing other tasks from running.

To handle this, Tokio provides a dedicated thread pool for blocking operations. You can run blocking code, such as CPU-bound computations or interacting with blocking file I/O, using the `spawn_blocking` function. This moves the work to a separate thread, ensuring the main runtime is not blocked.

```rust main.rs icon=logos:rust
#[tokio::main]
async fn main() {
    // This is running on a core thread.

    let blocking_task = tokio::task::spawn_blocking(|| {
        // This is running on a blocking thread.
        // Blocking here is ok.
        // For example, a CPU-intensive computation:
        let mut sum = 0;
        for i in 0..1_000_000_000 {
            sum += i;
        }
        sum
    });

    // We can wait for the blocking task to complete.
    let result = blocking_task.await.unwrap();
    println!("Blocking task finished with result: {}", result);
}
```

For managing pools of CPU-bound tasks, consider using a dedicated library like [Rayon](https://docs.rs/rayon). You can integrate it with Tokio by using a `oneshot` channel to send the result from the Rayon task back to an asynchronous Tokio task.

---

Now that you have an overview of the core components, a great next step is to dive into how Tokio manages concurrent operations. 

**Next**: [Tasks & Scheduling](./concepts-tasks.md)