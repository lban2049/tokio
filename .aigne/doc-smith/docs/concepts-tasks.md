# Tasks & Scheduling

Asynchronous programs in Tokio are built around tasks. A task is a lightweight, non-blocking unit of execution that runs concurrently with other tasks. Instead of being managed by the operating system like traditional threads, tasks are managed by the Tokio runtime, which makes them significantly cheaper to create and manage.

Key characteristics of Tokio tasks include:

*   **Lightweight:** Creating and switching between tasks has very low overhead compared to OS threads.
*   **Cooperative Scheduling:** Tasks run until they voluntarily yield control to the scheduler (usually at an `.await` point), allowing other tasks to run. This is different from the preemptive multitasking used for OS threads.
*   **Non-blocking:** Tasks should not perform operations that block the thread, such as synchronous I/O or heavy, long-running computations. Doing so would prevent other tasks on the same thread from making progress.

This model allows a small number of OS threads to handle a massive number of concurrent tasks efficiently.

```d2
direction: down

OS: {
  shape: package
  label: "Operating System"
  grid-columns: 2

  Thread-1: {
    label: "OS Thread 1"
    shape: rectangle
  }
  Thread-2: {
    label: "OS Thread 2"
    shape: rectangle
  }
}

Tokio-Runtime: {
  label: "Tokio Runtime"
  shape: package

  Scheduler: {
    shape: hexagon
  }

  Worker-Thread-1: {
    label: "Worker Thread 1"
    shape: rectangle

    Task-A: { label: "Task A"; shape: circle }
    Task-B: { label: "Task B"; shape: circle }
    Task-C: { label: "Task C"; shape: circle }
  }
  Worker-Thread-2: {
    label: "Worker Thread 2"
    shape: rectangle

    Task-D: { label: "Task D"; shape: circle }
    Task-E: { label: "Task E"; shape: circle }
  }

  OS.Thread-1 -> Worker-Thread-1: "Runs"
  OS.Thread-2 -> Worker-Thread-2: "Runs"
  Scheduler -> Worker-Thread-1: "Manages"
  Scheduler -> Worker-Thread-2: "Manages"
  Worker-Thread-1.Task-A <-> Worker-Thread-1.Task-B: "Cooperatively\nYields"
  Worker-Thread-1.Task-B <-> Worker-Thread-1.Task-C: "Cooperatively\nYields"
}
```

## Spawning Tasks

The most common way to create a task is with the `tokio::spawn` function. It takes an asynchronous block or future and immediately begins running it in the background, returning a `JoinHandle` that you can use to interact with the task.

```rust,no_run
use tokio::net::{TcpListener, TcpStream};
use std::io;

async fn process(socket: TcpStream) {
    // ... handle the connection
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        // Spawn a new task to process each connection concurrently.
        tokio::spawn(async move {
            process(socket).await
        });
    }
}
```

The `JoinHandle` allows you to await the task's completion and get its return value.

```rust
# #[tokio::main] async fn main() -> Result<(), Box<dyn std::error::Error>> {
let join_handle = tokio::spawn(async {
    // ... perform some work
    "hello world!"
});

// Await the result of the spawned task.
let result = join_handle.await?;
assert_eq!(result, "hello world!");
# Ok(())
# }
```

If a task panics, awaiting its `JoinHandle` will return a `JoinError`.

## Managing Multiple Tasks with `JoinSet`

When you need to manage a dynamic collection of tasks, `JoinSet` is a powerful utility. It allows you to spawn multiple tasks and await their results as they complete, in completion order.

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..10 {
        set.spawn(async move { i });
    }

    let mut seen = [false; 10];
    while let Some(res) = set.join_next().await {
        let idx = res.unwrap();
        seen[idx] = true;
    }

    for i in 0..10 {
        assert!(seen[i]);
    }
}
```

When the `JoinSet` is dropped, all tasks still contained within it are automatically aborted.

## Task Cancellation

Tasks can be cancelled using the `abort` method on their `JoinHandle` or an `AbortHandle`. Cancellation is a signal that requests the task to shut down at its next `.await` point. When a task is cancelled, awaiting its `JoinHandle` will result in a `JoinError` indicating it was cancelled.

Note that calling `abort` only schedules the cancellation. To wait for the task to fully shut down, you must still await its `JoinHandle`.

## Handling Blocking Operations

Because tasks are scheduled cooperatively, performing a blocking operation directly within an async task will stall the entire worker thread, preventing other tasks from running. Tokio provides two primary mechanisms to handle this.

### `spawn_blocking`

For I/O-bound or CPU-bound work that is synchronous, use `spawn_blocking`. This function runs the provided closure on a separate thread pool dedicated to blocking operations, without interfering with the async runtime's main scheduler.

```rust
# use tokio::task;
# async fn docs() -> Result<(), Box<dyn std::error::Error>>{
let join_handle = task::spawn_blocking(|| {
    // Perform some compute-heavy work or call synchronous I/O code.
    "blocking operation completed"
});

let result = join_handle.await?;
assert_eq!(result, "blocking operation completed");
# Ok(())
# }
```

Tasks spawned with `spawn_blocking` cannot be aborted once they have started running.

### `block_in_place`

When using the multi-threaded runtime, `block_in_place` offers an alternative. It informs the scheduler that the current thread is about to block. The runtime can then hand off other tasks scheduled on this thread to a different worker, preventing them from being stalled. This can be more efficient than `spawn_blocking` as it may avoid a thread context switch.

```rust
use tokio::task;

# #[tokio::main] async fn main() {
let result = task::block_in_place(|| {
    // do some compute-heavy work or call synchronous code
    "blocking completed"
});

assert_eq!(result, "blocking completed");
# }
```

This function will panic if called from a single-threaded runtime.

## Yielding Control

You can voluntarily yield control back to the Tokio scheduler by calling `tokio::task::yield_now()`. This allows the scheduler to run other pending tasks before resuming the current one.

```rust
use tokio::task;

# #[tokio::main] async fn main() {
async {
    task::spawn(async {
        println!("spawned task done!")
    });

    // Yield, allowing the newly-spawned task to potentially execute first.
    task::yield_now().await;
    println!("main task done!");
}.await;
}
```

It's important to note that the exact scheduling order is not guaranteed. The runtime might choose to poll the yielding task again immediately without running other tasks first.

---

Now that you understand how to create and manage tasks, the next step is to learn how these tasks can perform work. Continue to the [Asynchronous I/O](./concepts-io.md) section to see how Tokio handles non-blocking network and file operations.