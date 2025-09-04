# Runtime

The Tokio runtime is the engine that powers asynchronous applications. It provides the necessary services for executing tasks, handling I/O, and managing time-based events. This section provides a detailed API reference for manually configuring and interacting with the runtime.

For a higher-level understanding of the runtime's role and its components, please see the [Core Concepts: The Runtime](./concepts-runtime.md) guide.

## Runtime

The `Runtime` struct is the main entry point. It bundles an I/O driver, a task scheduler, a timer, and a thread pool for blocking operations. Most applications can use the `#[tokio::main]` attribute, but you can create and manage a `Runtime` instance directly for more control.

### Creating a Runtime

You can create a runtime with a default multi-threaded configuration using `Runtime::new()`.

```rust
use tokio::runtime::Runtime;

// Create a new runtime with default settings
let rt = Runtime::new().unwrap();

// Use the runtime...
```

For more advanced configurations, such as selecting a scheduler or enabling specific drivers, use the [`Builder`](#builder).

### Executing Futures

The primary way to run a future on the runtime is with the `block_on` method. This method takes ownership of the current thread, runs the given future to completion, and returns its result.

```rust
use tokio::runtime::Runtime;

// Create the runtime
let rt = Runtime::new().unwrap();

// Execute a future, blocking the current thread until completion
rt.block_on(async {
    println!("Hello from the runtime!");
});
```

### Spawning Tasks

From within the context of a runtime, you can spawn additional asynchronous tasks to run concurrently using `spawn`. These tasks will be executed on the runtime's thread pool.

```rust
use tokio::runtime::Runtime;

// Create the runtime
let rt = Runtime::new().unwrap();

// Spawn a future onto the runtime
rt.spawn(async {
    println!("Now running on a worker thread.");
});

// To wait for the spawned task, you can block on its JoinHandle
let handle = rt.spawn(async {
    "Task finished"
});

let result = rt.block_on(handle).unwrap();
println!("{}", result);
```

For blocking, CPU-bound, or otherwise long-running synchronous operations, use `spawn_blocking` to avoid stalling the scheduler.

```rust
use tokio::runtime::Runtime;
use std::thread;
use std::time::Duration;

let rt = Runtime::new().unwrap();

rt.spawn_blocking(|| {
    println!("Running a blocking operation...");
    thread::sleep(Duration::from_secs(1));
    println!("Blocking operation complete.");
});
```

### Shutdown

Dropping the `Runtime` instance initiates a graceful shutdown. It will wait for all spawned tasks to complete. If you need to control the shutdown behavior, you can use `shutdown_timeout` or `shutdown_background`.

- **`shutdown_timeout(duration)`**: Waits for at most `duration` for tasks to stop. After the timeout, any remaining tasks and their threads are leaked.
- **`shutdown_background()`**: Initiates shutdown without blocking, allowing the runtime to be dropped from within an asynchronous context. This may lead to resource leaks if blocking tasks are still running.

```rust
use tokio::runtime::Runtime;
use std::time::Duration;

let runtime = Runtime::new().unwrap();

runtime.spawn(async {
    // some long-running task
    tokio::time::sleep(Duration::from_secs(10)).await;
});

// Shut down, waiting a maximum of 100ms for tasks to finish.
runtime.shutdown_timeout(Duration::from_millis(100));
```

## Builder

The `Builder` provides a way to configure a `Runtime` before it is created. You can choose the scheduler type, set the number of worker threads, enable I/O and time drivers, and more.

### Creating a Builder

There are two main entry points for creating a `Builder`, depending on the desired scheduler:

- **`Builder::new_multi_thread()`**: Creates a builder for the work-stealing, multi-threaded scheduler. This is suitable for most applications.
- **`Builder::new_current_thread()`**: Creates a builder for the single-threaded scheduler, which runs all tasks on the current thread.

### Configuration

Here's an example of creating a custom multi-threaded runtime:

```rust
use tokio::runtime::Builder;
use std::time::Duration;

let runtime = Builder::new_multi_thread()
    .worker_threads(4) // Set the number of worker threads
    .thread_name("my-tokio-worker") // Set a name for the threads
    .thread_stack_size(3 * 1024 * 1024) // Set stack size
    .enable_all() // Enable both I/O and time drivers
    .build()
    .unwrap();

runtime.block_on(async {
    println!("Hello from a custom runtime!");
});
```

**Common Configuration Methods:**

| Method | Description |
|---|---|
| `enable_all()` | Enables both I/O and time drivers. A convenient shorthand. |
| `enable_io()` | Enables the I/O driver for networking, processes, and signals. |
| `enable_time()` | Enables the time driver for `tokio::time` utilities. |
| `worker_threads(usize)` | Sets the number of worker threads for the multi-thread scheduler. |
| `max_blocking_threads(usize)` | Sets the maximum number of threads for the blocking pool. |
| `thread_name(String)` | Sets the prefix for the name of spawned worker threads. |
| `thread_keep_alive(Duration)` | Sets the idle timeout for blocking pool threads. |

## Handle

A `Handle` is a lightweight, cloneable reference to an active `Runtime`. It allows you to interact with the runtime, such as spawning tasks, from any context, including other threads.

### Obtaining a Handle

- **`Runtime::handle()`**: Get a handle from an existing `Runtime` instance.
- **`Handle::current()`**: Get a handle to the runtime of the current execution context. This will panic if called outside of a Tokio runtime context.
- **`Handle::try_current()`**: A non-panicking version of `current()` that returns a `Result`.

```rust
use tokio::runtime::{Handle, Runtime};

let rt = Runtime::new().unwrap();

// Get a handle from the runtime instance
let handle_from_rt = rt.handle();

rt.block_on(async {
    // Get a handle from the current context
    let handle_from_ctx = Handle::current();
    
    handle_from_ctx.spawn(async {
        println!("Task spawned from a handle!");
    });
});
```

### Using a Handle

A `Handle` can be used to spawn tasks, run blocking futures, and enter the runtime context, even from a standard `std::thread`.

```rust
use tokio::runtime::{Handle, Runtime};
use std::thread;

#[tokio::main]
async fn main() {
    let handle = Handle::current();

    let std_thread = thread::spawn(move || {
        // Use the handle to run an async block on the runtime from another thread
        handle.block_on(async {
            println!("Hello from another thread!");
        });
    });

    std_thread.join().unwrap();
}
```

### Entering the Runtime Context

The `enter()` method on both `Runtime` and `Handle` returns an `EnterGuard`. While the guard is in scope, the current thread is considered to be within that runtime's context. This allows functions like `tokio::spawn` to work without needing an explicit `Handle`.

```rust
use tokio::runtime::Runtime;

fn function_that_spawns() {
    // This would panic if not inside a runtime context
    tokio::spawn(async {
        println!("Spawned without an explicit handle.");
    });
}

let rt = Runtime::new().unwrap();

// Enter the runtime context
let _guard = rt.enter();

// Now we can call functions that implicitly rely on the runtime context
function_that_spawns();
```

## RuntimeFlavor

Tokio supports two scheduler flavors. You can determine which flavor a `Handle` is associated with using the `runtime_flavor()` method, which returns a `RuntimeFlavor` enum.

- `RuntimeFlavor::CurrentThread`: A single-threaded scheduler.
- `RuntimeFlavor::MultiThread`: A multi-threaded, work-stealing scheduler.

```rust
use tokio::runtime::{Handle, RuntimeFlavor};

#[tokio::main(flavor = "multi_thread")]
async fn main() {
  assert_eq!(RuntimeFlavor::MultiThread, Handle::current().runtime_flavor());
}
```
