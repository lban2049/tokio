# Runtime

The Tokio runtime provides an I/O driver, task scheduler, timer, and blocking pool necessary for running asynchronous tasks. It bundles these services into a single type, allowing them to be started, shut down, and configured together.

For most applications, the `#[tokio::main]` attribute macro is the easiest way to get started, as it creates and manages a `Runtime` automatically. However, for more advanced configuration, you can build a `Runtime` instance manually using the `Builder`.

For a deeper understanding of the runtime's role and architecture, see the [runtime concepts page](./concepts-runtime.md).

## Runtime

The `Runtime` struct is the main entry point for the Tokio runtime. It encapsulates all the necessary components for executing asynchronous code.

```rust
use tokio::runtime::Runtime;

// Create a new runtime with default configuration
let rt = Runtime::new().unwrap();

// Use the runtime to block on a future
rt.block_on(async {
    println!("Hello from the runtime!");
});
```

### Shutdown

Shutting down the runtime is done by dropping the `Runtime` value or by calling `shutdown_background` or `shutdown_timeout`. When a runtime is dropped, the thread initiating the shutdown blocks until all spawned work has been stopped. This can take an indefinite amount of time. The `shutdown_timeout` method allows specifying a maximum duration to wait.

### Sharing

Access to a `Runtime` can be shared across threads in several ways:
- **`Arc<Runtime>`**: A shared pointer that prevents the runtime from shutting down as long as a reference exists.
- **`Handle`**: A lightweight, cloneable handle that allows spawning tasks and entering the runtime context without preventing shutdown. See the `Handle` section below for more details.
- **Entering the context**: The `enter` method provides a context guard, allowing Tokio functions like `tokio::spawn` to work within its scope.

### Methods

#### new() -> Result<Runtime, io::Error>
Creates a new `Runtime` instance with a default multi-threaded scheduler and all drivers enabled.

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
```

#### handle(&self) -> &Handle
Returns a handle to the runtime, which can be cloned and sent to other threads to interact with the runtime.

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
let handle = rt.handle();
```

#### spawn<F>(&self, future: F) -> JoinHandle<F::Output>
Spawns a future onto the runtime. The future must be `Send`.

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
rt.spawn(async {
    println!("Task running on the runtime.");
});
```

#### spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>
Runs a blocking function on a dedicated thread pool for blocking operations.

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
rt.spawn_blocking(|| {
    // This is a blocking operation
    std::thread::sleep(std::time::Duration::from_secs(1));
    println!("Blocking task complete.");
});
```

#### block_on<F: Future>(&self, future: F) -> F::Output
Runs a future to completion on the current thread, blocking until it finishes. This is the main entry point for a runtime.

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
let result = rt.block_on(async {
    42
});
assert_eq!(result, 42);
```

#### enter(&self) -> EnterGuard<'_>
Enters the runtime context. While the returned `EnterGuard` is in scope, functions like `tokio::spawn` will use this runtime.

```rust
use tokio::runtime::Runtime;

fn some_function() {
    // This would panic if not inside a runtime context
    tokio::spawn(async { println!("Spawned!"); });
}

let rt = Runtime::new().unwrap();
let _guard = rt.enter(); // Enter the context
some_function();
// Guard is dropped, context is exited
```

#### shutdown_timeout(self, duration: Duration)
Shuts down the runtime, waiting for at most `duration` for all spawned work to stop. Any work that doesn't stop in time is leaked.

```rust
use tokio::runtime::Runtime;
use std::time::Duration;

let rt = Runtime::new().unwrap();
// ... spawn tasks ...
rt.shutdown_timeout(Duration::from_millis(100));
```

#### shutdown_background(self)
Shuts down the runtime without waiting for any spawned work to stop. This is useful for dropping a runtime from within another asynchronous context.

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
rt.block_on(async {
    let inner_rt = Runtime::new().unwrap();
    // ...
    inner_rt.shutdown_background();
});
```

## Builder

The `Builder` provides a way to configure a `Runtime` before creating it. You can select the scheduler type, configure worker threads, enable or disable drivers, and set up lifecycle hooks.

```rust
use tokio::runtime::Builder;

let runtime = Builder::new_multi_thread()
    .worker_threads(4)
    .thread_name("my-runtime-worker")
    .enable_all()
    .build()
    .unwrap();

runtime.block_on(async {
    println!("Hello from a custom-built runtime!");
});
```

### Methods

#### Creating a Builder

| Method | Description |
|---|---|
| `new_multi_thread()` | Creates a builder for the multi-threaded, work-stealing scheduler. |
| `new_current_thread()` | Creates a builder for the single-threaded scheduler. |

#### Configuring Threads

| Method | Description |
|---|---|
| `worker_threads(val: usize)` | Sets the number of worker threads for the multi-thread scheduler. Panics if `val` is 0. |
| `max_blocking_threads(val: usize)` | Sets the maximum number of threads for the blocking thread pool. Defaults to 512. |
| `thread_name(val: impl Into<String>)` | Sets a static name for all threads spawned by the runtime. |
| `thread_name_fn<F>(f: F)` | Sets a function that generates a name for each new thread. |
| `thread_stack_size(val: usize)` | Sets the stack size (in bytes) for worker threads. |
| `thread_keep_alive(duration: Duration)` | Sets a custom timeout for idle threads in the blocking pool. |

#### Configuring Drivers and Schedulers

| Method | Description |
|---|---|
| `enable_all()` | Enables both I/O and time drivers. |
| `enable_io()` | Enables the I/O driver (for networking, processes, signals). |
| `enable_time()` | Enables the time driver (for `tokio::time`). |
| `event_interval(val: u32)` | Sets how often the scheduler checks for external events (I/O, timers). Default is 61 ticks. |
| `global_queue_interval(val: u32)` | Sets how often the scheduler polls the global task queue. |

#### Lifecycle and Task Hooks
These methods allow you to execute custom code at different points in the runtime and task lifecycle. They are primarily used for monitoring and bookkeeping.

| Method | Description |
|---|---|
| `on_thread_start<F>(f: F)` | Executes a function after each worker thread is started. |
| `on_thread_stop<F>(f: F)` | Executes a function before each worker thread stops. |
| `on_thread_park<F>(f: F)` | Executes a function just before a worker thread goes idle. |
| `on_thread_unpark<F>(f: F)` | Executes a function just after a worker thread becomes active. |

#### Building the Runtime

| Method | Description |
|---|---|
| `build() -> io::Result<Runtime>` | Creates the configured `Runtime` instance. |

## Handle

A `Handle` is a lightweight, cloneable reference to a `Runtime`. It allows you to interact with the runtime (e.g., spawn tasks) from any thread that has a handle, without needing ownership of the `Runtime` object itself.

### Methods

#### current() -> Handle
Returns a handle to the currently running runtime. Panics if called outside of a Tokio runtime context.

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::runtime::Handle::current();
    handle.spawn(async { /* ... */ });
}
```

#### try_current() -> Result<Handle, TryCurrentError>
Returns a handle to the currently running runtime, or an error if not in a runtime context. This method does not panic.

```rust
use tokio::runtime::Handle;

if let Ok(handle) = Handle::try_current() {
    println!("Running inside a Tokio runtime.");
} else {
    println!("Not running inside a Tokio runtime.");
}
```

#### enter(&self) -> EnterGuard<'_'>
Enters the runtime context associated with this handle. See `Runtime::enter` for more details.

#### spawn<F>(&self, future: F) -> JoinHandle<F::Output>
Spawns a future onto the runtime associated with this handle.

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::runtime::Handle::current();
    let join_handle = handle.spawn(async {
        "Hello from a spawned task!"
    });
    let result = join_handle.await.unwrap();
    println!("{}", result);
}
```

#### spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>
Runs a blocking function on the runtime's blocking thread pool.

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::runtime::Handle::current();
    let join_handle = handle.spawn_blocking(|| {
        // Blocking I/O or CPU-intensive work
        "done"
    });
    let result = join_handle.await.unwrap();
    assert_eq!(result, "done");
}
```

#### block_on<F: Future>(&self, future: F) -> F::Output
Blocks the current thread until the provided future completes. This is useful for running async code from a synchronous context when you have a `Handle`.

```rust
use tokio::runtime::Handle;

#[tokio::main]
async fn main() {
    let handle = Handle::current();
    std::thread::spawn(move || {
        // Use the handle to block on an async task in a new synchronous thread.
        handle.block_on(async {
            println!("Running async code in another thread");
        });
    }).join().unwrap();
}
```

#### runtime_flavor(&self) -> RuntimeFlavor
Returns the flavor of the runtime, indicating whether it is a `CurrentThread` or `MultiThread` scheduler.

## Other Types

### RuntimeFlavor
An enum that indicates the scheduling strategy of a `Runtime`.
- `CurrentThread`: A single-threaded scheduler that runs all tasks on the current thread.
- `MultiThread`: A multi-threaded, work-stealing scheduler.

### EnterGuard
An RAII guard returned by `Runtime::enter` and `Handle::enter`. The runtime context is active as long as this guard is in scope. It is important to drop guards in the reverse order they were created to avoid panics.

### TryCurrentError
An error type returned by `Handle::try_current` when a handle to the current runtime cannot be obtained.