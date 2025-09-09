# The Runtime

The Tokio runtime is the engine that powers asynchronous applications in Rust. It provides the essential services needed to execute async tasks, manage I/O, and handle time-based events. While you can write `async fn` in Rust, you need a runtime to actually run them.

At a high level, the Tokio runtime bundles together:

*   An **I/O event loop**, often called the driver, which interacts with the operating system's I/O APIs (like epoll, kqueue, or IOCP).
*   A **task scheduler** that manages the execution of many lightweight, non-blocking tasks concurrently on a few OS threads.
*   A **timer** for scheduling work to run at a future time, powering features like `tokio::time::sleep` and timeouts.

Most users interact with the runtime through the `#[tokio::main]` macro, which sets up a default runtime and executes the annotated `async fn`. For more advanced control, you can build and manage a `Runtime` instance manually.

## Usage Patterns

There are two primary ways to start using the Tokio runtime.

### The `#[tokio::main]` Macro

For most applications, the simplest way to get started is with the `#[tokio::main]` attribute macro. This macro transforms an `async fn main()` into a synchronous `fn main()` that initializes a `Runtime` instance and runs the future to completion.

```rust main.rs icon=logos:rust
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
                        println!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    println!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

Within the context of a running runtime, you can spawn additional tasks using [`tokio::spawn`](./api-task.md).

### Manual Runtime Management

If you need more control over the runtime's configuration, you can build and manage it yourself using the [`Runtime`](./api-runtime.md) struct. The `block_on` method is the entry point, which runs a future until it completes, blocking the current thread.

```rust main.rs icon=logos:rust
use tokio::runtime::Runtime;
use tokio::net::TcpListener;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt  = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await?;
        println!("Listening on: {}", listener.local_addr()?);
        // ... application logic ...
        Ok(())
    })
}
```

## Scheduler Configurations

Tokio offers different scheduling strategies to suit various application needs. You can select a scheduler using the [`Builder`](./api-runtime.md).

<x-cards>
  <x-card data-title="Multi-Thread Scheduler" data-icon="lucide:cpu">
    This is the default scheduler. It uses a thread pool with a work-stealing strategy to execute tasks across multiple CPU cores. It's the best choice for most applications, especially network services that handle many concurrent connections.
  </x-card>
  <x-card data-title="Current-Thread Scheduler" data-icon="lucide:user">
    This scheduler provides a single-threaded executor. All tasks are created and executed on the same thread that the runtime is started on. It's useful for scenarios where you need to run async code but don't require multi-threading, or when dealing with `!Send` futures in conjunction with a `LocalSet`.
  </x-card>
</x-cards>

## Customizing the Runtime with `Builder`

The [`Builder`](./api-runtime.md) provides a fluent API to configure every aspect of the runtime before it's created.

Here are some of the most common configuration options:

| Method | Description |
|---|---|
| `worker_threads(n)` | Sets the number of worker threads for the multi-thread scheduler. Defaults to the number of CPU cores. |
| `max_blocking_threads(n)` | Sets the upper limit for threads in the blocking pool, used for `spawn_blocking`. Defaults to 512. |
| `enable_all()` | Enables both the I/O and time drivers. Required when building a runtime manually to use networking or time features. |
| `enable_io()` | Enables the I/O driver for networking, filesystem, etc. |
| `enable_time()` | Enables the time driver for sleeps, intervals, and timeouts. |
| `thread_name("name")` | Sets a custom name for worker threads, which is useful for debugging. |
| `thread_stack_size(bytes)` | Sets the stack size for worker threads. |

Here's an example of building a custom multi-threaded runtime:

```rust builder_example.rs icon=logos:rust
use tokio::runtime::Builder;

fn main() {
    // Build a runtime with 4 worker threads, a custom thread name,
    // and both I/O and time drivers enabled.
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("my-tokio-worker")
        .thread_stack_size(3 * 1024 * 1024)
        .enable_all()
        .build()
        .unwrap();

    runtime.block_on(async {
        println!("Hello from the custom runtime!");
    });
}
```

## Runtime Shutdown

Shutting down the runtime happens when the `Runtime` instance is dropped. The shutdown process blocks the thread until spawned work is stopped.

- **Asynchronous tasks** spawned with `spawn` run until their next yield point (`.await`), at which point they are dropped.
- **Blocking tasks** spawned with `spawn_blocking` will run to completion.

Because the default `drop` implementation can block indefinitely, Tokio provides two alternative shutdown methods for cases where you need more control:

- `shutdown_timeout(duration)`: Waits for a specified duration for work to complete. If the timeout is reached, threads and tasks are leaked, but the calling thread is unblocked.
- `shutdown_background()`: A shorthand for `shutdown_timeout` with a zero duration. It initiates shutdown and immediately returns, without waiting for tasks to stop.

## Execution Behavior and Fairness

Tokio's scheduler provides a fairness guarantee: if the total number of tasks doesn't grow infinitely and no task blocks a thread, every task is guaranteed to be scheduled eventually. This prevents task starvation.

However, the specific order of execution is not guaranteed. The scheduler may poll some tasks more frequently than others. It may also perform *spurious wakeups*, where a task is polled even if its `Waker` has not been called. Your `Future` implementations should be robust against this behavior.

The multi-threaded scheduler employs a work-stealing strategy. Each worker has a local queue of tasks. When a worker's queue is empty, it will try to steal tasks from other workers' queues to ensure all threads stay busy.