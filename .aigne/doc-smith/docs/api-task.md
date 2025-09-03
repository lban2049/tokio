# Tasks

Asynchronous green-threads for managing concurrent operations in Tokio. For a conceptual overview, please see the [Tasks & Scheduling](./concepts-tasks.md) guide.

## What are Tasks?

A _task_ is a lightweight, non-blocking unit of execution. A task is similar to an OS thread, but instead of being managed by the OS scheduler, it is managed by the Tokio runtime. This pattern is also known as [green threads](https://en.wikipedia.org/wiki/Green_threads).

Key points about tasks include:

*   **Lightweight**: Creating, switching, and destroying tasks has very low overhead compared to OS threads.
*   **Cooperative**: Tasks run until they yield (e.g., at an `.await` point), allowing the Tokio scheduler to run another task.
*   **Non-blocking**: Tasks should not perform blocking operations like standard I/O, as this would block the entire worker thread. Instead, Tokio provides APIs for running blocking operations asynchronously.

### Task Lifecycle

The following diagram illustrates the basic lifecycle of a Tokio task.

```d2
direction: down

"spawn()": { shape: oval }
"Running": { shape: rectangle }
"Yielded": { shape: rectangle; style.stroke-dash: 2 }
"Completed": { shape: oval; style.fill: "#d4edda" }
"Panicked": { shape: oval; style.fill: "#f8d7da" }
"Cancelled": { shape: oval; style.fill: "#fff3cd" }

"spawn()" -> "Running": "Scheduled for execution"
"Running" -> "Yielded": ".await on incomplete Future"
"Yielded" -> "Running": "Woken by event source"
"Running" -> "Completed": "Future returns Ready(val)"
"Running" -> "Panicked": "panic!() called"

"Running" -> "Cancelled": "JoinHandle.abort()"
"Yielded" -> "Cancelled": "JoinHandle.abort()"

```

## Spawning and Execution

Tokio provides several functions for spawning tasks, each suited for different scenarios.

<x-cards data-columns="2">
  <x-card data-title="spawn" data-icon="lucide:play-circle">
    Spawns a new asynchronous task to run concurrently. This is the primary function for creating new tasks.
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:cpu">
    Runs a blocking function on a dedicated thread pool, preventing it from blocking the async runtime.
  </x-card>
  <x-card data-title="block_in_place" data-icon="lucide:pause-circle">
    Transitions the current worker thread to a blocking thread, useful for short, infrequent blocking operations.
  </x-card>
  <x-card data-title="yield_now" data-icon="lucide:fast-forward">
    Yields execution back to the Tokio scheduler, allowing other tasks to run.
  </x-card>
</x-cards>

### spawn

Spawns a new asynchronous task, returning a `JoinHandle` for it. The task begins running immediately in the background.

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    let s = format!("Starting background task {}.", id);
    println!("{}", s);
    s
}

#[tokio::main]
async fn main() {
    let ops = vec![1, 2, 3];
    let mut tasks = Vec::with_capacity(ops.len());
    for op in ops {
        tasks.push(tokio::spawn(my_background_op(op)));
    }

    let mut outputs = Vec::with_capacity(tasks.len());
    for task in tasks {
        outputs.push(task.await.unwrap());
    }
    println!("{:?}", outputs);
}
```

The `JoinHandle` returned by `spawn` is a future that resolves to the output of the task. Awaiting it allows you to get the task's result. If the task panics, awaiting the handle will return a `JoinError`.

### spawn_blocking

Use `spawn_blocking` to execute CPU-bound or blocking I/O code without interfering with other asynchronous tasks.

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut v = "Hello, ".to_string();
    let res = task::spawn_blocking(move || {
        // This is a stand-in for compute-heavy work
        v.push_str("world");
        v
    }).await?;

    assert_eq!(res.as_str(), "Hello, world");
    Ok(())
}
```

### block_in_place

This function allows running a blocking operation from within an asynchronous context on the multi-threaded runtime. It transitions the current worker thread into a blocking thread and moves other tasks to a different worker.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let result = task::block_in_place(|| {
        // do some compute-heavy work or call synchronous code
        "blocking completed"
    });

    assert_eq!(result, "blocking completed");
}
```

### yield_now

Calling `yield_now().await` yields control back to the scheduler, allowing it to poll other tasks. The current task is placed at the back of the queue and will be resumed later.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("spawned task done!")
    });

    // Yield, allowing the newly-spawned task to execute first.
    task::yield_now().await;
    println!("main task done!");
}
```

## Managing Task Collections: JoinSet

A `JoinSet` is a collection that allows you to spawn and manage multiple tasks, awaiting them as they complete in any order.

- **`spawn()`**: Adds a new task to the set.
- **`join_next()`**: Awaits the completion of the next task in the set.
- **`abort_all()`**: Aborts all tasks in the set.
- **`shutdown()`**: Aborts all tasks and waits for them to complete.

When a `JoinSet` is dropped, all tasks within it are aborted.

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

## Local Tasks for `!Send` Futures

Some types, like `std::rc::Rc`, are not `Send` and cannot be safely sent across threads. To work with futures that use such types, Tokio provides a mechanism to ensure they run on a single thread.

### LocalSet

A `LocalSet` is a task executor that runs all its tasks on the current thread. This allows you to spawn `!Send` futures.

### spawn_local

Within a `LocalSet`, you must use `spawn_local` to spawn `!Send` futures. This function guarantees that the spawned task will remain on the thread that created the `LocalSet`.

```rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("my nonsend data...");

    // Construct a local task set.
    let local = task::LocalSet::new();

    // Run the local task set until the future completes.
    local.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();
        // `spawn_local` ensures the future runs on the current thread.
        task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
        }).await.unwrap();
    }).await;
}
```

## Cancellation

Tasks can be cancelled using the `abort` method on their `JoinHandle` or an associated `AbortHandle`.

- When a task is aborted, it is signaled to shut down the next time it yields at an `.await` point.
- All local variables are dropped, running their destructors.
- Awaiting the `JoinHandle` of an aborted task will result in a `JoinError` where `is_cancelled()` returns `true`.
- Note that `spawn_blocking` tasks cannot be aborted once they have started running.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        panic!("something bad happened!")
    });

    // The returned result indicates that the task failed.
    assert!(handle.await.is_err());
}
```

## Related Data Structures

| Struct | Description |
|---|---|
| `JoinHandle<T>` | A handle returned by spawning functions. It can be awaited to get the task's output `T` or a `JoinError`. |
| `JoinError` | An error returned when a task fails, either by panicking or by being cancelled. |
| `AbortHandle` | A handle that can be used to abort a task without needing to await its completion. |
| `Id` | A unique identifier for a spawned task. |

---

This page covers the primary APIs for task management. For details on how tasks are scheduled and executed, see the documentation for the [Tokio Runtime](./api-runtime.md).
