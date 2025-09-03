# The Runtime

The Tokio runtime is the engine that powers asynchronous applications in Rust. Unlike synchronous programs, async code requires a dedicated environment to manage tasks, handle I/O events, and coordinate time-based operations. The Tokio `Runtime` provides all these necessary services bundled together.

At a high level, the runtime is responsible for:

-   **An I/O event loop (driver):** This is the core component that interfaces with the operating system's event queue (like `epoll` on Linux, `kqueue` on macOS, or `IOCP` on Windows) to drive I/O resources and notify tasks when they are ready to make progress.
-   **A task scheduler:** This component manages a pool of lightweight, non-blocking tasks, deciding which task to run on which thread at any given time.
-   **A timer:** This provides the ability to schedule work to run after a specific duration, enabling features like sleeps, intervals, and timeouts.

While you can manually configure and manage a `Runtime` instance, most applications start with the `#[tokio::main]` macro, which conveniently sets up a default runtime.

## Runtime Architecture

A Tokio runtime coordinates several components to execute asynchronous code efficiently. Understanding this architecture helps in configuring the runtime for specific needs and diagnosing performance issues.

```d2
direction: down

"Application Code": { shape: rectangle }
os: "Operating System\n(epoll, kqueue, IOCP)": { shape: cloud }

"Tokio Runtime": {
  shape: package
  grid-columns: 1
  grid-gap: 50

  ts: "Task Scheduler" {
    wt: "Worker Threads"
  }
  drivers: "Resource Drivers" {
    grid-columns: 2
    io: "I/O Driver"
    timer: "Timer"
  }
  bp: "Blocking Pool"
}

# Connections
"Application Code" -> "Tokio Runtime".ts: "tokio::spawn()"
"Application Code" -> "Tokio Runtime".bp: "tokio::task::spawn_blocking()"

"Tokio Runtime".ts -> "Tokio Runtime".ts.wt: "dispatches tasks"
"Tokio Runtime".ts -> "Tokio Runtime".drivers: "polls for events"
"Tokio Runtime".drivers.io <-> os: "I/O Events"
```

## Scheduler Types

Tokio offers different scheduling strategies, allowing you to choose the best fit for your application's workload.

### Multi-Thread Scheduler

This is the default and most commonly used scheduler. It utilizes a pool of worker threads, typically one for each CPU core, and employs a work-stealing strategy to keep all threads busy. When a thread runs out of tasks in its local queue, it will "steal" tasks from other, busier threads. This approach is ideal for most server-side applications and workloads that can benefit from parallelism.

It is enabled by default with `Runtime::new()` or `Builder::new_multi_thread()`.

```rust
use tokio::runtime;

// Creates a multi-threaded runtime with default settings.
let rt = runtime::Runtime::new().unwrap();

rt.block_on(async {
    println!("Running on the multi-thread scheduler!");
});
```

### Current-Thread Scheduler

The current-thread scheduler executes all tasks on the thread that creates the runtime. It's a single-threaded executor. This scheduler is useful for scenarios where you need to run async code but don't require multi-threading, such as in resource-constrained environments or when embedding an async runtime into a larger, existing application.

To use it, you must construct it with the `Builder`.

```rust
use tokio::runtime;

// Creates a single-threaded runtime.
let rt = runtime::Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();

// Runs the runtime on the current thread.
rt.block_on(async {
    println!("Running on the current-thread scheduler!");
});
```

## Creating and Configuring a Runtime

You can create a runtime with default settings or customize it extensively using the `Builder`.

### The Simple Way: `#[tokio::main]`

For most applications, the `#[tokio::main]` attribute macro is the simplest way to start a runtime. It transforms an `async fn main()` into a synchronous `fn main()` that initializes a `Runtime` and executes the future.

```rust,no_run
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}
```

### The Manual Way: `Runtime::new()` and `Builder`

For more control, you can build a runtime manually. This is necessary when you need to configure thread counts, enable specific drivers, or set up lifecycle hooks.

The `Builder` provides a fluent API for configuration. Remember that when using the `Builder`, resource drivers for I/O and time are disabled by default and must be explicitly enabled with methods like `enable_io()`, `enable_time()`, or the convenient `enable_all()`.

**Common Configuration Options**

| Method                   | Description                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| `worker_threads(usize)`  | Sets the number of worker threads for the multi-thread scheduler.                        |
| `max_blocking_threads(usize)` | Sets the maximum number of threads in the pool for blocking operations.                  |
| `thread_name(String)`    | Sets a custom name for spawned worker threads, useful for debugging.                     |
| `enable_all()`           | Enables both the I/O and time drivers.                                                   |
| `enable_io()`            | Enables the I/O driver for networking, filesystem, etc.                                  |
| `enable_time()`          | Enables the time driver for sleeps, intervals, and timeouts.                             |
| `thread_stack_size(usize)` | Sets the stack size for worker threads.                                                  |
| `thread_keep_alive(Duration)` | Sets a custom timeout for idle threads in the blocking pool.                             |

**Example: Building a Custom Runtime**

```rust
use tokio::runtime::Builder;
use std::time::Duration;

fn main() {
    // Build a custom runtime
    let runtime = Builder::new_multi_thread()
        .worker_threads(4) // Use 4 worker threads
        .thread_name("my-tokio-worker")
        .thread_keep_alive(Duration::from_millis(100))
        .enable_all() // Enable I/O and time drivers
        .build()
        .unwrap();

    // Use the runtime to block on the main future
    runtime.block_on(async {
        println!("Running on a custom-configured runtime!");
    });
}
```

## Runtime Shutdown

The runtime shuts down when the `Runtime` instance is dropped. During shutdown, the runtime attempts to gracefully stop all spawned work. The thread that drops the `Runtime` will block until the shutdown process is complete.

-   **For async tasks:** Tasks run until their next yield point (`.await`), at which point they are dropped.
-   **For blocking tasks:** Tasks spawned with `spawn_blocking` run to completion.

Because waiting for all work to complete can take an indefinite amount of time, Tokio provides alternative shutdown methods:

-   `shutdown_timeout(duration)`: Waits for a specified duration for work to stop. If the timeout is reached, remaining work and threads are leaked, and the function returns.
-   `shutdown_background()`: Initiates shutdown without waiting. This is equivalent to `shutdown_timeout(Duration::from_nanos(0))` and is useful for dropping a runtime from within an async context.

## Further Reading

This page provides a conceptual overview of the Tokio runtime. For detailed configuration options and methods, refer to the [Runtime API Reference](./api-runtime.md).
