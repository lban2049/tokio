# The Runtime

Asynchronous applications in Rust require a runtime to execute. The Tokio runtime provides the necessary services to make this happen, including:

- An **I/O event loop**, often called the driver, which manages I/O resources and notifies tasks when they are ready to proceed.
- A **task scheduler** to coordinate the execution of many concurrent tasks.
- A **timer** for scheduling work to run at a future time.

The `Runtime` type in Tokio bundles all of these services, allowing them to be configured, started, and shut down together. While you can configure a `Runtime` manually, most applications will start with the `#[tokio::main]` attribute macro, which conveniently sets up a default runtime.

```d2
direction: down

"Your Application Code": {
  shape: rectangle

  "main()": {
    label: "async fn main()"
    shape: code
  }

  "tokio::spawn()": {
    label: "tokio::spawn(async { ... })"
    shape: code
  }
}

"Tokio Runtime": {
  shape: package
  style.stroke-dash: 2

  Scheduler: {
    shape: hexagon
    grid-columns: 2

    "Task Queue": {
      shape: queue
    }
    "Worker Threads": {
      shape: class
    }
  }

  "I/O Driver (epoll, kqueue, IOCP)": {
    shape: hexagon
  }

  Timer: {
    shape: hexagon
  }

  "Blocking Pool": {
    label: "Blocking Thread Pool"
    shape: class
  }
}

"Your Application Code"."main()" -> "Tokio Runtime": "Executes on"
"Your Application Code"."tokio::spawn()" -> "Tokio Runtime".Scheduler."Task Queue": "Submits Task"
"Tokio Runtime".Scheduler."Worker Threads" -> "Tokio Runtime".Scheduler."Task Queue": "Pulls Task"
"Tokio Runtime".Scheduler <-> "Tokio Runtime"."I/O Driver (epoll, kqueue, IOCP)": "Polls for events"
"Tokio Runtime".Scheduler <-> "Tokio Runtime".Timer: "Schedules timeouts"
"Tokio Runtime".Scheduler -> "Tokio Runtime"."Blocking Pool": "Offloads blocking work"

```

## Usage

There are two primary ways to use the Tokio runtime: with the `#[tokio::main]` macro for simplicity, or by manually creating and managing a `Runtime` instance for more control.

### The `#[tokio::main]` Macro

For most applications, the `#[tokio::main]` attribute is the easiest way to get started. It creates a default multi-threaded runtime and runs the decorated `async fn main` on it.

```rust,no_run
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

### Manual Runtime Management

If you need to customize the runtime's configuration, you can create an instance of `Runtime` directly. The `block_on` method is the entry point for running a future to completion on the runtime.

```rust,no_run
use tokio::runtime::Runtime;
use tokio::net::TcpListener;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt  = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
        println!("Listening on: {}", listener.local_addr().unwrap());
        // ... application logic ...
    });
    Ok(())
}
```

## Runtime Configurations

Tokio offers different scheduling strategies to suit various application needs. You can select a scheduler using the `Builder`.

<x-cards data-columns="2">
  <x-card data-title="Multi-Thread Scheduler" data-icon="lucide:users">
    Executes futures on a work-stealing thread pool, typically with one worker thread per CPU core. This is the default and best choice for most server-side applications.
  </x-card>
  <x-card data-title="Current-Thread Scheduler" data-icon="lucide:user">
    A single-threaded executor that runs all tasks on the current thread. Useful for scenarios where only a single thread is required, or for spawning `!Send` futures within a `LocalSet`.
  </x-card>
</x-cards>

### Customizing with the `Builder`

The `Builder` provides fine-grained control over the runtime's configuration. You can chain methods to customize thread behavior, enable/disable drivers, and set scheduler parameters.

Here are some of the most common configuration options:

| Method | Description |
|---|---|
| `worker_threads(n)` | Sets the number of worker threads for the multi-thread scheduler. |
| `max_blocking_threads(n)` | Sets the upper limit for threads spawned for blocking operations. |
| `thread_name("name")` | Sets the name for threads spawned by the runtime. |
| `thread_stack_size(bytes)` | Sets the stack size for worker threads. |
| `enable_all()` | Enables both I/O and time drivers. |
| `enable_io()` | Enables the I/O driver for networking, processes, and signals. |
| `enable_time()` | Enables the time driver for `tokio::time` utilities. |

**Example: Building a custom runtime**

```rust
use tokio::runtime::Builder;

fn main() {
    // Build a runtime with 4 worker threads, a custom thread name,
    // and a larger stack size.
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

Shutting down the runtime is accomplished by dropping the `Runtime` instance. The behavior upon shutdown is important to understand:

- The thread initiating the shutdown blocks until all spawned work has been stopped.
- **Asynchronous tasks** (`spawn`) run until they yield, at which point they are dropped. They are not guaranteed to run to completion.
- **Blocking tasks** (`spawn_blocking`) are allowed to run until they return.

Because the default drop behavior can block indefinitely, Tokio provides alternative shutdown methods:

- `shutdown_timeout(duration)`: Waits for a specified duration for work to complete. If the timeout is reached, any remaining work and the threads running it are leaked, and the shutdown call unblocks.
- `shutdown_background()`: A shorthand for `shutdown_timeout(Duration::from_nanos(0))`. It initiates the shutdown and returns immediately, without waiting for any work to stop. This is useful for dropping a runtime from within another async context.

---

With a solid understanding of the runtime, you can now explore the detailed configuration options and metrics available.

For a complete list of runtime features and builder options, please see the [Runtime API Reference](./api-runtime.md).