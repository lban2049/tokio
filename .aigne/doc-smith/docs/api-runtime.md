# Runtime

The Tokio runtime is the engine that drives asynchronous applications. It provides essential services like an I/O event loop, a task scheduler, and a timer. While many applications can simply use the `#[tokio::main]` macro to get a default runtime, this module provides the tools for manual configuration and interaction when more control is needed.

For a deeper conceptual overview of how these components work together, please see our guide on [The Runtime](./concepts-runtime.md).

This page covers the primary types for manually managing the runtime:

<x-cards>
  <x-card data-title="Runtime" data-icon="lucide:cpu">
    The main Tokio runtime instance, bundling the scheduler, I/O driver, and timer. You create one to execute your asynchronous code.
  </x-card>
  <x-card data-title="Builder" data-icon="lucide:settings-2">
    A tool for constructing a `Runtime` with custom configuration for threads, drivers, scheduler behavior, and more.
  </x-card>
  <x-card data-title="Handle" data-icon="lucide:grip">
    A lightweight, cloneable handle to the runtime. It allows you to spawn tasks or enter the runtime context from other threads or synchronous code.
  </x-card>
</x-cards>

## Runtime

The `Runtime` struct is the main entry point for executing asynchronous code. It bundles all necessary services and manages their lifecycle.

### Creating a Runtime

You can create a runtime with a default multi-threaded configuration or use the `Builder` for customization.

#### `new()`

Creates a new `Runtime` instance with default values. This initializes a multi-threaded scheduler and enables both I/O and time drivers.

```rust icon=logos:rust Creating a default Runtime
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt = Runtime::new()?;

    // Use the runtime...
    rt.block_on(async {
        println!("Hello from the Tokio runtime!");
    });

    Ok(())
}
```

### Running Futures

#### `block_on<F: Future>(&self, future: F) -> F::Output`

Runs a future to completion on the Tokio runtime, blocking the current thread until it finishes. This is the primary way to start executing asynchronous code from a synchronous context.

This method must not be called from within an asynchronous context.

```rust icon=logos:rust Using block_on
use tokio::runtime::Runtime;

// Create the runtime
let rt  = Runtime::new().unwrap();

// Execute the future, blocking the current thread until completion
rt.block_on(async {
    println!("hello");
});
```

### Spawning Tasks

Tasks are lightweight, non-blocking units of execution. Spawning a task allows it to run concurrently on the runtime's thread pool.

#### `spawn<F>(&self, future: F) -> JoinHandle<F::Output>`

Spawns a new asynchronous task, returning a `JoinHandle` for it. The spawned future must be `Send` and `'static`.

```rust icon=logos:rust Spawning a task
use tokio::runtime::Runtime;
use std::time::Duration;

let rt = Runtime::new().unwrap();

rt.block_on(async {
    let handle = rt.spawn(async {
        tokio::time::sleep(Duration::from_secs(1)).await;
        "done"
    });

    // Do other work while the task runs...
    println!("Spawned task in the background");

    let result = handle.await.unwrap();
    assert_eq!(result, "done");
});
```

#### `spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>`

Runs a blocking function on a separate thread pool dedicated to blocking operations. This is crucial for preventing long-running synchronous code from blocking the asynchronous task scheduler.

```rust icon=logos:rust Spawning a blocking task
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();

rt.block_on(async {
    let handle = rt.spawn_blocking(|| {
        // This is a blocking operation
        std::thread::sleep(std::time::Duration::from_secs(1));
        "blocking operation complete"
    });

    let result = handle.await.unwrap();
    println!("{}", result);
});
```

### Managing the Runtime

#### `handle(&self) -> &Handle`

Returns a reference to a `Handle` for this runtime. Handles can be cloned and sent to other threads.

```rust icon=logos:rust Getting a handle
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
let handle = rt.handle();

// Use the handle to spawn tasks
handle.spawn(async {
    println!("Task spawned via handle!");
});
```

#### `enter(&self) -> EnterGuard<'_>`

Enters the runtime context. This allows code in a synchronous context to use context-aware functions like `tokio::spawn`.

```rust icon=logos:rust Entering the runtime context
use tokio::runtime::Runtime;
use tokio::task::JoinHandle;

fn function_that_spawns(msg: String) -> JoinHandle<()> {
    // Had we not used `rt.enter` below, this would panic.
    tokio::spawn(async move {
        println!("{}", msg);
    })
}

let rt = Runtime::new().unwrap();
let s = "Hello World!".to_string();

// By entering the context, we tie `tokio::spawn` to this executor.
let _guard = rt.enter();
let handle = function_that_spawns(s);

// Wait for the task before we end.
rt.block_on(handle).unwrap();
```

### Shutdown

Dropping the `Runtime` instance will initiate a graceful shutdown, waiting for spawned tasks to complete. For more control, you can use explicit shutdown methods.

*   **`shutdown_timeout(duration: Duration)`**: Shuts down the runtime, waiting at most `duration` for all spawned work to stop. Any work that doesn't stop in time is leaked.
*   **`shutdown_background()`**: Shuts down the runtime without waiting for any spawned work to stop. This is useful for dropping a runtime from within another async context.

## Builder

The `Builder` provides a flexible way to configure and create a `Runtime` instance.

### Creating a Builder

You can start with a builder for either a multi-threaded or a current-thread scheduler.

*   **`new_multi_thread()`**: Creates a builder for the multi-thread, work-stealing scheduler. This is the default and suitable for most applications.
*   **`new_current_thread()`**: Creates a builder for a single-threaded scheduler that runs all tasks on the current thread.

```rust icon=logos:rust Creating builders
use tokio::runtime::Builder;

// A builder for a multi-threaded runtime
let multi_thread_builder = Builder::new_multi_thread();

// A builder for a single-threaded runtime
let current_thread_builder = Builder::new_current_thread();
```

### Configuration Methods

The builder uses a chaining pattern to configure the runtime. Here are some of the most common options:

| Method | Description |
|---|---|
| `enable_all()` | Enables both I/O and time drivers. A convenient shorthand. |
| `enable_io()` | Enables the I/O driver (for networking, filesystems, etc.). |
| `enable_time()` | Enables the time driver (for `tokio::time`). |
| `worker_threads(val: usize)` | Sets the number of worker threads for the multi-thread scheduler. |
| `max_blocking_threads(val: usize)` | Sets the maximum number of threads for the blocking task pool. |
| `thread_name(val: impl Into<String>)` | Sets a static name for all threads spawned by the runtime. |
| `thread_name_fn<F>(f: F)` | Sets a function to generate names for spawned threads dynamically. |
| `thread_stack_size(val: usize)` | Sets the stack size for spawned threads. |
| `on_thread_start<F>(f: F)` | Executes a function after each worker thread starts. |
| `on_thread_stop<F>(f: F)` | Executes a function before each worker thread stops. |

### Building the Runtime

After configuring the builder, call `build()` to create the `Runtime` instance.

```rust icon=logos:rust Building a custom runtime
use tokio::runtime::Builder;
use std::time::Duration;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("my-tokio-worker")
        .thread_stack_size(3 * 1024 * 1024)
        .enable_all()
        .build()?;

    runtime.block_on(async {
        println!("Running on a custom-configured runtime!");
    });

    Ok(())
}
```

## Handle

A `Handle` is a lightweight, cloneable, reference-counted pointer to a `Runtime`. It allows you to interact with the runtime (e.g., spawn tasks) from any code that can access the handle, including other threads.

### Obtaining a Handle

There are two main ways to get a handle:

*   **`Runtime::handle()`**: Get a handle from an existing `Runtime` instance.
*   **`Handle::current()`**: Get a handle to the runtime of the current execution context. **This will panic if called outside of a Tokio runtime context.**
*   **`Handle::try_current()`**: A non-panicking version of `Handle::current()` that returns a `Result`.

```rust icon=logos:rust Obtaining a handle
use tokio::runtime::Handle;

#[tokio::main]
async fn main () {
    // This is okay because we are inside a #[tokio::main] context.
    let handle = Handle::current();
    
    std::thread::spawn(move || {
        // We can use the moved handle in another thread.
        let _guard = handle.enter();
        
        // Now this is also okay.
        let handle2 = Handle::current();
        println!("Got handle in another thread.");
    }).join().unwrap();
}
```

### Using a Handle

A `Handle` provides many of the same methods as a `Runtime`, such as `spawn`, `spawn_blocking`, `block_on`, and `enter`.

One important distinction exists for `block_on`: when used on a handle to a `current_thread` runtime, it cannot drive the I/O or timer drivers. Only the original `Runtime::block_on` call can do that. This means timers or I/O may not function as expected unless another thread is actively driving the runtime.

### Introspection

`Handle` provides methods to inspect the runtime it belongs to.

*   **`runtime_flavor()`**: Returns a `RuntimeFlavor` enum (`CurrentThread` or `MultiThread`).
*   **`metrics()`**: Returns a `RuntimeMetrics` view to get performance information.

## Related Types

<x-cards data-columns="2">
  <x-card data-title="RuntimeFlavor" data-icon="lucide:git-branch">
    An enum indicating whether the runtime uses a `CurrentThread` or `MultiThread` scheduler.
  </x-card>
  <x-card data-title="UnhandledPanic (unstable)" data-icon="lucide:shield-alert">
    Configures how the runtime responds when a spawned task panics. The default is `Ignore`, but can be set to `ShutdownRuntime`.
  </x-card>
</x-cards>