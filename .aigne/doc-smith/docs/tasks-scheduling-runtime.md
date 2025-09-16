# The Runtime

Asynchronous Rust applications require a runtime to power them. The Tokio runtime provides the necessary services for executing asynchronous tasks, including an I/O event loop (the driver), a task scheduler, and a timer.

At a high level, the runtime is responsible for:

- **An I/O event loop**, called the driver, which drives I/O resources and dispatches I/O events to tasks that depend on them.
- **A scheduler** to execute tasks that use these I/O resources.
- **A timer** for scheduling work to run after a set period of time.

The `tokio::runtime::Runtime` type bundles all of these services, allowing them to be started, configured, and shut down together. While you can configure a `Runtime` manually, most users will start with the `#[tokio::main]` macro, which creates and manages a runtime automatically.

## Quick Start with `#[tokio::main]`

For most applications, the `#[tokio::main]` attribute macro is the simplest way to start a Tokio runtime. It sets up a multi-threaded runtime with default settings and executes the decorated `async fn main` on it.

```rust A simple TCP echo server icon=logos:rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write it back.
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

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

## Manual Runtime Management

For more control, you can create and manage a `Runtime` instance directly. This is useful when you need to customize its configuration or embed a Tokio runtime within a larger synchronous application.

The `Runtime::block_on` method is the entry point for executing an asynchronous task on the runtime. It blocks the current thread until the provided future completes.

Here is the same TCP echo server, but with a manually configured runtime:

```rust Manually creating a runtime icon=logos:rust
use tokio::runtime::Runtime;
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt  = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await?;

        loop {
            let (mut socket, _) = listener.accept().await?;

            tokio::spawn(async move {
                let mut buf = [0; 1024];

                // In a loop, read data from the socket and write it back.
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
    })
}
```

## Runtime Schedulers

Tokio offers different task scheduling strategies to suit various application needs. You can select a scheduler using the `Builder` API or the `flavor` argument of the `#[tokio::main]` macro.

### Multi-Thread Scheduler

The multi-thread scheduler uses a thread pool and a work-stealing strategy to execute tasks concurrently. It spawns one worker thread per CPU core by default, making it the ideal choice for most applications.

This is the default scheduler. You can create it manually like this:

```rust Creating a multi-threaded runtime icon=logos:rust
use tokio::runtime::Builder;

let runtime = Builder::new_multi_thread()
    .worker_threads(4) // Configure 4 worker threads
    .enable_all()      // Enable I/O and time drivers
    .build()
    .unwrap();
```

### Current-Thread Scheduler

The current-thread scheduler executes all tasks on the thread that creates the runtime. It's a single-threaded executor, which can be useful for applications with specific threading requirements or for embedding Tokio into an existing single-threaded event loop.

```rust Creating a current-thread runtime icon=logos:rust
use tokio::runtime::Builder;

let runtime = Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();
```

## Configuring with `Builder`

The `tokio::runtime::Builder` provides a comprehensive API for fine-tuning the runtime. You can configure worker threads, blocking threads, thread names, stack sizes, and enable or disable specific runtime drivers.

```rust Custom runtime configuration icon=logos:rust
use tokio::runtime::Builder;
use std::time::Duration;

let runtime = Builder::new_multi_thread()
    .worker_threads(4)
    .max_blocking_threads(512)
    .thread_name("my-tokio-worker")
    .thread_stack_size(3 * 1024 * 1024)
    .thread_keep_alive(Duration::from_secs(60))
    .enable_all()
    .build()
    .unwrap();
```

Key configuration options include:

- `worker_threads(usize)`: Sets the number of worker threads for the multi-thread scheduler.
- `max_blocking_threads(usize)`: Sets the maximum number of threads in the blocking pool for running synchronous code with `spawn_blocking`.
- `thread_name(&str)` or `thread_name_fn(Fn)`: Sets a static or dynamically generated name for worker threads.
- `thread_stack_size(usize)`: Sets the stack size for worker threads.
- `enable_io()`: Enables the I/O driver (for networking, filesystems, etc.).
- `enable_time()`: Enables the time driver (for `sleep`, `interval`, `timeout`).
- `enable_all()`: A shorthand to enable both I/O and time drivers.

## The Runtime Handle

A `Handle` is a lightweight, cloneable reference to a Tokio runtime. It allows you to interact with the runtime, such as spawning tasks, from any context, even from threads not managed by Tokio.

You can obtain a handle in two primary ways:

1.  **From an existing `Runtime`**: `let handle = runtime.handle();`
2.  **From within a task**: `let handle = Handle::current();`

Using a `Handle` is the preferred way to grant other parts of your application access to the runtime without giving them ownership of the `Runtime` itself. Unlike an `Arc<Runtime>`, a `Handle` does not prevent the runtime from shutting down.

```rust Spawning tasks from another thread icon=logos:rust
use tokio::runtime::{Runtime, Handle};
use std::thread;
use std::time::Duration;

let runtime = Runtime::new().unwrap();
let handle = runtime.handle().clone();

let job = thread::spawn(move || {
    // Use the handle to run async code in this new thread.
    handle.block_on(async {
        println!("Hello from a blocking context!");
    });
});

job.join().unwrap();
```

## Core Operations

<x-cards>
  <x-card data-title="block_on" data-icon="lucide:log-in">
    The entry point to the runtime. It takes a future and blocks the current thread until that future completes, returning its output.
  </x-card>
  <x-card data-title="spawn" data-icon="lucide:send">
    Spawns a new asynchronous task, returning a `JoinHandle` for it. The task runs concurrently with other tasks on the runtime.
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:brick-wall">
    Runs a blocking function on a dedicated thread pool, preventing it from blocking the asynchronous scheduler.
  </x-card>
  <x-card data-title="enter" data-icon="lucide:arrow-right-left">
    Enters the runtime context. This allows you to use context-aware functions like `tokio::spawn` or create I/O types from non-async code.
  </x-card>
</x-cards>

## Shutdown

The runtime shuts down when the `Runtime` value is dropped. This process involves stopping all worker threads and attempting to gracefully terminate spawned tasks. By default, the drop implementation will wait indefinitely for all blocking tasks spawned via `spawn_blocking` to complete.

For situations where you need more control, `Runtime` offers two explicit shutdown methods:

- `shutdown_timeout(duration)`: Shuts down the runtime, waiting at most for the specified duration for all work to stop. Any threads that don't stop in time are leaked.
- `shutdown_background()`: Shuts down the runtime immediately without waiting for spawned work to stop. This is useful for dropping a runtime from within another async context, but may lead to resource leaks.

```rust Timed shutdown icon=logos:rust
use tokio::runtime::Runtime;
use std::time::Duration;

let runtime = Runtime::new().unwrap();

runtime.spawn(async {
    // Some long-running task
    tokio::time::sleep(Duration::from_secs(10)).await;
});

// The runtime will shut down after 100ms, even if the task hasn't finished.
runtime.shutdown_timeout(Duration::from_millis(100));
```

With a solid understanding of the runtime, you are ready to manage more complex asynchronous operations. To learn more about creating and managing concurrent tasks, proceed to the next section.

---

Next, let's dive deeper into how tasks are created and managed.

<x-card data-title="Next: Spawning & Managing Tasks" data-icon="lucide:arrow-right" data-href="/tasks-scheduling/spawning" data-cta="Read More">
  Learn how to create and manage concurrent tasks using `spawn` and `JoinHandle`.
</x-card>