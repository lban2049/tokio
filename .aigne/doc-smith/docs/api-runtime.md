# Runtime

The Tokio runtime is the engine that powers asynchronous applications. It provides essential services like an I/O event loop, a task scheduler, a timer, and a thread pool for blocking operations. This section provides detailed API documentation for manually configuring and interacting with the runtime.

For a higher-level conceptual overview, please see [The Runtime](./concepts-runtime.md) in our Core Concepts guide.

## Core Components

A Tokio runtime bundles several key components that work together to execute asynchronous tasks.

```d2
direction: down

"Tokio Runtime": {
  shape: package
  grid-columns: 2
  grid-gap: 50

  "Task Scheduler": {
    shape: rectangle
    "Manages and executes asynchronous tasks."
  }

  "I/O Driver (Reactor)": {
    shape: rectangle
    "Interfaces with the OS for non-blocking I/O."
  }

  "Timer": {
    shape: rectangle
    "Provides utilities like `sleep` and `interval`."
  }

  "Blocking Pool": {
    shape: rectangle
    "A dedicated thread pool for blocking operations."
  }
}

"Your Async Code": {
  shape: rectangle
}

"Your Async Code" -> "Tokio Runtime": "Spawns tasks & uses resources"

"Tokio Runtime"."Task Scheduler" <-> "Tokio Runtime"."I/O Driver (Reactor)": "Wakes tasks on I/O events"
"Tokio Runtime"."Task Scheduler" <-> "Tokio Runtime"."Timer": "Wakes tasks on timeout"
"Tokio Runtime"."Task Scheduler" -> "Tokio Runtime"."Blocking Pool": "Offloads blocking calls"

```

## Runtime

The `Runtime` struct is the main entry point. It encapsulates all the runtime services. While many applications can rely on the `#[tokio::main]` macro, creating a `Runtime` instance directly offers fine-grained control.

### Creating a Runtime

The simplest way to create a multi-threaded runtime with default settings is with `Runtime::new()`.

```rust
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a new runtime with default configurations.
    let rt = Runtime::new()?;

    // Use the runtime to block on a future.
    rt.block_on(async {
        println!("Hello from the Tokio runtime!");
    });

    Ok(())
}
```

### Core Methods

<x-cards>
  <x-card data-title="block_on" data-icon="lucide:arrow-down-square">
    Runs a future to completion on the runtime, blocking the current thread until the future resolves.
  </x-card>
  <x-card data-title="spawn" data-icon="lucide:send">
    Spawns a new asynchronous task to be executed on the runtime.
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:box">
    Runs a blocking function on a dedicated thread pool, preventing it from blocking the async scheduler.
  </x-card>
  <x-card data-title="handle" data-icon="lucide:grip">
    Returns a `Handle` to the runtime, which can be cloned and sent to other threads.
  </x-card>
</x-cards>

### Shutdown

Shutting down the runtime is accomplished by dropping the `Runtime` value. The `drop` implementation will block the current thread until all spawned work has been stopped. For non-blocking shutdown or shutdown with a timeout, you can use `shutdown_background()` or `shutdown_timeout()`.

```rust
use tokio::runtime::Runtime;
use tokio::task;
use std::thread;
use std::time::Duration;

fn main() {
   let runtime = Runtime::new().unwrap();

   runtime.block_on(async move {
       task::spawn_blocking(move || {
           // Simulate a long-running blocking task
           thread::sleep(Duration::from_secs(10));
       });
   });

   // Shutdown with a 100ms timeout. The blocking task will be leaked.
   runtime.shutdown_timeout(Duration::from_millis(100));
}
```

## Builder

The `Builder` provides a flexible way to configure a new `Runtime` instance. You can select the scheduler type, configure worker threads, enable/disable drivers, and set up lifecycle hooks.

### Creating a Builder

You can start building a runtime for either a multi-threaded or a current-thread scheduler.

-   `Builder::new_multi_thread()`: Creates a builder for the work-stealing multi-threaded scheduler. This is suitable for most applications.
-   `Builder::new_current_thread()`: Creates a builder for a single-threaded scheduler that runs all tasks on the current thread.

### Configuration Example

Here is an example of creating a custom multi-threaded runtime.

```rust
use tokio::runtime::Builder;
use std::time::Duration;

fn main() {
    let runtime = Builder::new_multi_thread()
        .worker_threads(4) // Set the number of worker threads
        .thread_name("my-tokio-worker") // Set a name for the worker threads
        .thread_stack_size(3 * 1024 * 1024) // Set the stack size for worker threads
        .enable_all() // Enable both I/O and time drivers
        .build() // Build the runtime
        .unwrap();

    runtime.block_on(async {
        println!("Running on a custom-configured runtime!");
    });
}
```

### Common Configuration Methods

| Method | Description |
|---|---|
| `enable_all()` | Enables both I/O and time drivers. A convenient shorthand. |
| `enable_io()` | Enables the I/O driver for networking, processes, and signals. |
| `enable_time()` | Enables the time driver for utilities like `sleep`, `interval`, and `timeout`. |
| `worker_threads(usize)` | Sets the number of worker threads for the multi-thread scheduler. |
| `max_blocking_threads(usize)` | Sets the maximum number of threads in the blocking pool. |
| `thread_name(impl Into<String>)` | Sets a static name for threads spawned by the runtime. |
| `thread_keep_alive(Duration)` | Sets a custom keep-alive timeout for threads in the blocking pool. |
| `on_thread_start(F)` | Executes a function after each worker thread starts. |

## Handle

A `Handle` is a cloneable, reference-counted handle to a `Runtime`. It allows you to interact with the runtime (e.g., spawn tasks) from any thread that has a handle, without needing a reference to the `Runtime` instance itself.

### Obtaining a Handle

There are two primary ways to get a `Handle`:

1.  **From an existing `Runtime`**: `runtime.handle()`
2.  **From within a runtime context**: `Handle::current()`

```rust
use tokio::runtime::{Handle, Runtime};

fn main() {
    let rt = Runtime::new().unwrap();

    // 1. Get a handle from the runtime instance
    let handle_from_rt = rt.handle().clone();

    rt.block_on(async {
        // 2. Get a handle from the current runtime context
        let handle_from_context = Handle::current();

        handle_from_context.spawn(async {
            println!("Task spawned from a handle!");
        });
    });
}
```

`Handle::current()` will panic if called outside of a Tokio runtime context. For cases where a runtime may not be active, `Handle::try_current()` returns a `Result` instead.

### Using a Handle

A `Handle` provides similar methods to `Runtime` for spawning tasks and blocking on futures.

-   `handle.spawn(future)`: Spawns a task on the associated runtime.
-   `handle.spawn_blocking(f)`: Spawns a blocking task on the runtime's blocking pool.
-   `handle.block_on(future)`: Blocks the current thread and runs a future to completion. Note that on a `current_thread` runtime, this method cannot drive I/O or timers; only `Runtime::block_on` can.
-   `handle.enter()`: Enters the runtime context, returning an `EnterGuard`. This is necessary when you need to create I/O or timer-based types outside of an `async` block.

```rust
use tokio::runtime::{Handle, Runtime};
use tokio::task::JoinHandle;
use tokio::time::{sleep, Duration};

// This function requires a runtime context to spawn a task.
fn function_that_spawns(msg: String) -> JoinHandle<()> {
    tokio::spawn(async move {
        println!("{}", msg);
        sleep(Duration::from_millis(10)).await;
    })
}

fn main() {
    let rt = Runtime::new().unwrap();

    let s = "Hello from outside the runtime context!".to_string();

    // Enter the runtime context to call `tokio::spawn`.
    let _guard = rt.enter();
    let handle = function_that_spawns(s);

    // Block on the handle to wait for the task to complete.
    rt.block_on(handle).unwrap();
}
```

### RuntimeFlavor

You can determine the type of scheduler the runtime is using via `handle.runtime_flavor()`.

```rust
use tokio::runtime::{Handle, RuntimeFlavor};

#[tokio::main(flavor = "current_thread")]
async fn main() {
  assert_eq!(RuntimeFlavor::CurrentThread, Handle::current().runtime_flavor());
}
```
