# The Runtime

The Tokio runtime is the engine that powers asynchronous Rust applications. While async code in Rust provides the syntax for non-blocking operations, it requires a runtime to actually execute the futures, manage tasks, and handle I/O events. The Tokio runtime provides all the necessary services for building robust, high-performance network applications.

At a high level, the runtime bundles together several key components:

- An **I/O event loop**, often called the driver, which interfaces with the operating system's event queue (like `epoll`, `kqueue`, or `IOCP`).
- A **task scheduler** that manages the execution of numerous, lightweight asynchronous tasks.
- A **timer** for scheduling work to run at a future time, enabling functionality like timeouts and intervals.
- A dedicated **thread pool** for offloading blocking, CPU-bound operations to prevent them from stalling the event loop.

For most applications, the `#[tokio::main]` macro is the simplest way to start the runtime. However, Tokio also provides a powerful `Builder` for fine-grained configuration. This section explores the runtime's architecture, configurations, and execution model.

### Runtime Architecture

The components of the Tokio runtime work together to execute your asynchronous code efficiently.

```d2
direction: down

"Application Code" {
  "async fn main() {}"
  "tokio::spawn(...)"
}

"Tokio Runtime" {
  style.fill: "#f0f8ff"
  "Scheduler (Multi-thread or Current-thread)"
  "Driver" : {
    "I/O Poller (epoll, kqueue, etc.)"
    "Timer"
  }
  "Blocking Thread Pool"
}

"Application Code" -> "Tokio Runtime"."Scheduler": "Spawns tasks"
"Tokio Runtime"."Scheduler" -> "Tokio Runtime"."Driver": "Polls for events"
"Tokio Runtime"."Scheduler" -> "Tokio Runtime"."Blocking Thread Pool": "Delegates blocking work"
"Tokio Runtime"."Driver" -> "Tokio Runtime"."Scheduler": "Wakes tasks on I/O/Time events"

```

## Usage

There are two primary ways to interact with the Tokio runtime: using the `#[tokio::main]` macro for simplicity, or building and managing a `Runtime` instance manually for greater control.

### Simple Usage with `#[tokio::main]`

The easiest way to get started is by annotating your `main` function. This macro creates a default multi-threaded runtime, starts it, and runs the `async` main function within it.

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];
            loop {
                let n = match socket.read(&mut buf).await {
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

### Manual Usage with `Runtime::new()`

For more control, you can create a `Runtime` instance yourself. The `block_on` method starts the runtime and blocks the current thread until the provided future completes.

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
        // ... same logic as above ...
    });

    Ok(())
}
```

## Scheduler Configurations

Tokio provides two scheduler types, each suited for different use cases.

<x-cards data-columns="2">
  <x-card data-title="Multi-Thread Scheduler" data-icon="lucide:cpu">
    This is the default scheduler. It uses a work-stealing strategy across a pool of worker threads, typically one for each CPU core. It is ideal for most applications, especially those with high concurrency and I/O-bound workloads.
  </x-card>
  <x-card data-title="Current-Thread Scheduler" data-icon="lucide:user-round">
    This scheduler runs all tasks on the single thread that created it. It is lighter than the multi-threaded scheduler and is useful for scenarios where only one thread is needed, or for embedding Tokio into an existing single-threaded application.
  </x-card>
</x-cards>

You can select the scheduler using the `Builder`.

```rust
use tokio::runtime::Builder;

// Create a multi-threaded runtime
let multi_thread_rt = Builder::new_multi_thread()
    .enable_all()
    .build()
    .unwrap();

// Create a single-threaded runtime
let current_thread_rt = Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();
```

## Configuring the Runtime with `Builder`

The `tokio::runtime::Builder` provides a flexible way to configure every aspect of the runtime before it's created. You can chain methods to customize thread counts, drivers, and scheduler behavior.

| Method | Description |
|---|---|
| `new_multi_thread()` | Creates a builder for the multi-threaded, work-stealing scheduler. |
| `new_current_thread()` | Creates a builder for the single-threaded scheduler. |
| `enable_all()` | Enables both the I/O and time drivers. |
| `enable_io()` | Enables the I/O driver for networking, file system, etc. |
| `enable_time()` | Enables the timer driver for sleeps, intervals, and timeouts. |
| `worker_threads(usize)` | Sets the number of worker threads for the multi-thread scheduler. |
| `max_blocking_threads(usize)` | Sets the maximum number of threads for the blocking task pool. |
| `thread_name(String)` | Sets the name for spawned worker threads. |
| `thread_keep_alive(Duration)`| Sets the idle timeout for threads in the blocking pool. |

Here is an example of a custom configuration:

```rust
use tokio::runtime::Builder;
use std::time::Duration;

let runtime = Builder::new_multi_thread()
    .worker_threads(4)
    .thread_name("my-tokio-worker")
    .thread_stack_size(3 * 1024 * 1024)
    .thread_keep_alive(Duration::from_secs(60))
    .enable_all()
    .build()
    .unwrap();

runtime.block_on(async {
    println!("Hello from a custom-configured runtime!");
});
```

## Execution Behavior

Tokio's schedulers are designed for fairness and efficiency. While the exact scheduling algorithm is an implementation detail, the high-level behavior is important to understand.

- **Fairness**: Tokio guarantees that if the total number of tasks does not grow indefinitely and no task blocks a worker thread, every woken task will eventually be scheduled to run.
- **Spurious Wakeups**: A task may occasionally be polled even if its waker has not been called. Your code should not rely on wakeups being perfectly precise.

### Scheduler Details
- **Multi-Threaded**: Each worker thread has its own local queue of tasks. When a worker's local queue is empty, it will first check a global queue for new tasks and then attempt to "steal" tasks from other workers' local queues. This work-stealing approach helps ensure that all threads stay busy and work is distributed evenly.
- **Current-Thread**: This scheduler uses a simpler model with a local and global queue. It prefers tasks from its local queue to minimize synchronization overhead but periodically polls the global queue to ensure fairness.

## Shutting Down the Runtime

The runtime shuts down when the `Runtime` value is dropped. During shutdown, the runtime attempts to gracefully stop all spawned work.

- **Async tasks** (`tokio::spawn`): These tasks run until their next yield point (`.await`) and are then dropped. They are not guaranteed to run to completion.
- **Blocking tasks** (`spawn_blocking`): These tasks run until they complete.

The `drop` implementation will block the current thread until all work has stopped, which can be indefinite. For situations where you cannot block forever, you can use `shutdown_timeout(duration)` or `shutdown_background()`.

```rust
use tokio::runtime::Runtime;
use std::time::Duration;

let runtime = Runtime::new().unwrap();

runtime.spawn(async {
    // Some long-running task
});

// Shutdown the runtime, waiting at most 100ms for tasks to stop.
runtime.shutdown_timeout(Duration::from_millis(100));
```

---

Now that you understand the core concepts of the runtime, you can explore how to manage individual units of work in the [Tasks & Scheduling](./concepts-tasks.md) section or dive into the detailed configuration options in the [API Reference](./api-runtime.md).
