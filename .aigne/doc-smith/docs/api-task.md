# Tasks

Asynchronous green-threads for managing concurrent operations.

## Overview

A _task_ is a lightweight, non-blocking unit of execution. A task is similar to an OS thread, but rather than being managed by the OS scheduler, tasks are managed by the Tokio runtime. This pattern is also known as [green threads](https://en.wikipedia.org/wiki/Green_threads). If you're familiar with Go's goroutines, Kotlin's coroutines, or Erlang's processes, you can think of Tokio's tasks as something similar.

Key points about tasks include:

*   **Lightweight**: Creating and switching between tasks has low overhead as it doesn't require an OS context switch.
*   **Cooperatively Scheduled**: Tasks run until they yield (e.g., at an `.await` point), allowing the Tokio scheduler to run another task.
*   **Non-blocking**: Tasks should not perform blocking operations like standard I/O, as this would block the entire worker thread. Instead, Tokio provides specific APIs for handling blocking code.

```d2
direction: down

"Your Application Code" {
  shape: cloud
}

"Tokio Runtime" {
  shape: package

  Scheduler {
    shape: hexagon
    grid-columns: 2

    "Worker Threads" {
      label: "Worker Threads\n(for async tasks)"
      shape: package
      
      Task-1 {
        label: "Async Task 1"
        shape: rectangle
      }
      Task-2 {
        label: "Async Task 2"
        shape: rectangle
      }
      Task-N {
        label: "..."
        shape: rectangle
      }
    }
    
    "Blocking Pool" {
      label: "Blocking Thread Pool\n(for sync code)"
      shape: package
      
      Blocking-Task-1 {
        label: "Blocking Op 1"
        shape: rectangle
      }
      Blocking-Task-M {
        label: "..."
        shape: rectangle
      }
    }
  }
}

"Your Application Code" -> "Tokio Runtime".Scheduler."Worker Threads".Task-1: "tokio::spawn(async {...})"
"Your Application Code" -> "Tokio Runtime".Scheduler."Blocking Pool".Blocking-Task-1: "tokio::spawn_blocking(|| {...})"

"Tokio Runtime".Scheduler."Worker Threads".Task-1 -> "Tokio Runtime".Scheduler."Worker Threads".Task-2: "yields"
```

This module provides APIs for spawning, canceling, and coordinating tasks.

---

## Spawning Tasks

The most common way to work with tasks is to spawn them for concurrent execution.

<x-cards data-columns="2">
  <x-card data-title="spawn" data-icon="lucide:play-circle">
    Spawns a new asynchronous task to run concurrently.
  </x-card>
  <x-card data-title="spawn_local" data-icon="lucide:anchor">
    Spawns a `!Send` future on the current thread's `LocalSet`.
  </x-card>
</x-cards>

### `spawn`

Spawns a new asynchronous task, returning a `JoinHandle` for it. The provided future will start running in the background immediately.

This function must be called from the context of a Tokio runtime. The spawned task may execute on the current thread or be sent to a different one.

```rust
use tokio::net::{TcpListener, TcpStream};
use std::io;

async fn process(socket: TcpStream) {
    // ...
#   drop(socket);
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            // Process each socket concurrently.
            process(socket).await
        });
    }
}
```

The returned `JoinHandle` is a future that can be awaited to get the task's output. If the task panics, awaiting the `JoinHandle` will return a `JoinError`.

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let join_handle = task::spawn(async {
        // ...
        "hello world!"
    });

    // Await the result of the spawned task.
    let result = join_handle.await?;
    assert_eq!(result, "hello world!");
    Ok(())
}
```

**Panics**: This function panics if called from outside of the Tokio runtime.

### `spawn_local`

Spawns a `!Send` future on the current `LocalSet`.

This is necessary for futures that hold data that is not safe to send across threads, such as `Rc<T>`. The spawned future will always run on the same thread that called `spawn_local`.

```rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("my nonsend data...");

    let local = task::LocalSet::new();

    // Run the local task set.
    local.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();
        task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
            // ...
        }).await.unwrap();
    }).await;
}
```

**Panics**: This function panics if called outside of a `LocalSet`.

---

## Managing Collections of Tasks

### `JoinSet`

A `JoinSet<T>` is a collection of tasks that all have the same return type `T`. It allows you to spawn multiple tasks and await their completion in the order they finish.

When the `JoinSet` is dropped, all tasks still in the set are aborted.

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

`JoinSet` provides methods like `spawn`, `spawn_blocking`, `join_next`, `abort_all`, and `shutdown` for managing the lifecycle of tasks within the set.

---

## Handling Blocking Code

Asynchronous tasks should not perform blocking operations, as this prevents other tasks on the same thread from running. Tokio provides two APIs to safely run blocking code from an asynchronous context.

### `spawn_blocking`

Runs a blocking closure on a thread pool dedicated to blocking operations. This is the preferred way to run CPU-bound code or synchronous I/O.

```rust
use tokio::task;

async fn docs() -> Result<(), Box<dyn std::error::Error>>{
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

### `block_in_place`

Available on the multi-threaded runtime, this function transitions the current worker thread into a blocking thread. It moves other tasks on the current thread to a different worker, which can improve performance by avoiding context switches.

```rust
use tokio::task;

async fn docs() {
    let result = task::block_in_place(|| {
        // do some compute-heavy work or call synchronous code
        "blocking completed"
    });

    assert_eq!(result, "blocking completed");
}
```

---

## Cooperative Yielding

### `yield_now`

Yields execution back to the Tokio scheduler, allowing other tasks to run. The current task is placed at the back of the pending queue and will be polled again later.

```rust
use tokio::task;

async fn example() {
    task::spawn(async {
        println!("spawned task done!")
    });

    // Yield, allowing the newly-spawned task to execute first.
    task::yield_now().await;
    println!("main task done!");
}
```

---

## Task Cancellation

Spawned tasks can be cancelled using the `abort` method on their `JoinHandle` or `AbortHandle`. Cancellation is a signal that requests the task to shut down at its next `.await` point.

-   `JoinHandle::abort()`: Schedules the task for cancellation.
-   Awaiting the `JoinHandle` after abortion will fail with a `JoinError::is_cancelled` error.
-   Aborting does not guarantee immediate termination. The task runs until its next yield point.
-   Tasks spawned with `spawn_blocking` cannot be aborted once they start running.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        // A long-running task
        tokio::time::sleep(std::time::Duration::from_secs(10)).await;
    });

    // Cancel the task
    handle.abort();

    // Awaiting the handle will now return a cancelled error.
    let result = handle.await;
    assert!(result.is_err());
    assert!(result.unwrap_err().is_cancelled());
}
```

Now that you understand how to manage tasks, you may want to learn how to coordinate them. For more information, see the [Synchronization Primitives](./api-sync.md) documentation.