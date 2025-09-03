# Tasks & Scheduling

Asynchronous programs in Tokio are built around lightweight, non-blocking units of execution called tasks. This guide covers the core concepts of how to create, manage, and coordinate these tasks for building concurrent applications.

## What are Tasks?

A task is similar to an OS thread, but it's managed by the Tokio runtime instead of the OS scheduler. This makes them significantly more lightweight. You can think of them as being similar to Go's goroutines or Erlang's processes.

Key characteristics of Tokio tasks include:

*   **Lightweight**: Creating, switching, and destroying tasks has very low overhead compared to OS threads. It's common for a Tokio application to manage thousands or even millions of tasks.
*   **Cooperatively Scheduled**: Tasks run until they yield control back to the scheduler, typically at an `.await` point. The scheduler then runs another task. This is in contrast to preemptive multitasking used by OS threads, where the OS can interrupt a thread at any time.
*   **Non-blocking**: Tasks should not perform operations that block the thread, such as synchronous I/O or heavy, long-running computations. Doing so would prevent other tasks on the same thread from making progress. For such cases, Tokio provides specific APIs.

Let's explore how to work with tasks in practice.

```d2
direction: down

"Tokio Runtime": {
  shape: package
  grid-columns: 2
  grid-gap: 80

  "Core Worker Threads": {
    label: "Core Worker Threads (for async tasks)"
    shape: package
    grid-columns: 2

    "Worker 1": {
      shape: rectangle
      "Task A (runs)" -> "yields at .await" -> "Task B (runs)" -> "yields at .await" -> "Task A (resumes)"
    }

    "Worker 2": {
      shape: rectangle
      "Task C (runs)" -> "yields at .await" -> "Task D (runs)"
    }
  }

  "Blocking Thread Pool": {
    label: "Blocking Thread Pool (for sync code)"
    shape: package
    grid-columns: 2

    "Blocking Thread 1": {
      shape: rectangle
      "Blocking Operation 1"
    }
    "Blocking Thread 2": {
      shape: rectangle
      "Blocking Operation 2"
    }
  }
}

"Application Code": {
  shape: rectangle
  grid-columns: 1
  "spawn(async_fn)": {
    label: "tokio::spawn(async { ... })"
  }
  "spawn_blocking(sync_fn)": {
    label: "tokio::spawn_blocking(|| { ... })"
  }
}

"Application Code"."spawn(async_fn)" -> "Tokio Runtime"."Core Worker Threads": "Schedules on a worker"
"Application Code"."spawn_blocking(sync_fn)" -> "Tokio Runtime"."Blocking Thread Pool": "Runs on a dedicated thread"

```

## Spawning Tasks

The most common way to create a task is with the `tokio::spawn` function. It takes an asynchronous block or future and immediately schedules it to run on the runtime. It returns a `JoinHandle`, which you can use to interact with the spawned task.

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    let s = format!("Background task {} complete.", id);
    println!("{}", s);
    s
}

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(my_background_op(1));

    // Do other work while the task runs...
    println!("Main function continues...");

    // Await the result of the spawned task.
    let result = handle.await.unwrap();
    println!("Result from task: {}", result);
}
```

If a spawned task panics, awaiting its `JoinHandle` will return a `JoinError`.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        panic!("something went wrong!")
    });

    // The await returns an Err result because the task panicked.
    assert!(handle.await.is_err());
}
```

## Task Cancellation

You can cancel a running task by calling the `abort()` method on its `JoinHandle`. This signals the task to shut down the next time it yields at an `.await` point. Awaiting a handle for an aborted task will result in a `JoinError` indicating it was cancelled.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // This task will run for a long time
        sleep(Duration::from_secs(10)).await;
    });

    // Abort the task
    handle.abort();

    // Awaiting the handle will now return a cancelled error
    let err = handle.await.unwrap_err();
    assert!(err.is_cancelled());
}
```

Note that `abort()` only schedules the cancellation. To wait for the task to fully shut down, you must still `.await` the `JoinHandle`.

## Managing Multiple Tasks with `JoinSet`

When you need to spawn several related tasks and process their results as they complete, a `JoinSet` is an excellent tool. It allows you to add tasks and then await the next one that finishes, without needing to manage a collection of `JoinHandle`s manually.

When a `JoinSet` is dropped, all tasks still remaining in the set are automatically aborted.

```rust
use tokio::task::JoinSet;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..5 {
        set.spawn(async move {
            sleep(Duration::from_millis(100 * i)).await;
            i * 2
        });
    }

    while let Some(res) = set.join_next().await {
        let output = res.unwrap();
        println!("Task completed with result: {}", output);
    }
}
```

## Handling Blocking Operations

Because tasks are scheduled cooperatively, running a blocking operation directly within an async task will block the entire worker thread, preventing any other tasks on that thread from running. Tokio provides two primary mechanisms to handle this.

### `spawn_blocking`

The `spawn_blocking` function takes a synchronous closure and runs it on a dedicated thread pool designed for blocking operations. This prevents the main async worker threads from being blocked.

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let blocking_task = task::spawn_blocking(|| {
        // This is running on a blocking thread.
        // Synchronous file I/O or heavy computation is OK here.
        std::thread::sleep(std::time::Duration::from_secs(1));
        "done"
    });

    // The JoinHandle from spawn_blocking can be awaited like any other.
    let result = blocking_task.await?;
    assert_eq!(result, "done");
    Ok(())
}
```

### `block_in_place`

When using the multi-threaded runtime, `block_in_place` provides an alternative. It signals to the runtime that the current thread is about to block. The runtime can then move other tasks from this thread to a different worker to keep them running, while the current thread is dedicated to the blocking operation.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
     task::block_in_place(|| {
        // This is running on the *same* thread, but the runtime has
        // moved other tasks away.
        std::thread::sleep(std::time::Duration::from_secs(1));
    });
}
```

## Yielding

Cooperative scheduling means a task runs until it hits an `.await`. If you have a task that does a lot of computation without awaiting, it can hog the scheduler. You can voluntarily yield control back to the scheduler by calling `tokio::task::yield_now().await`. This gives other pending tasks a chance to run.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("Spawned task running");
    });

    println!("Main task running");
    // Yield, allowing the newly-spawned task to potentially run before we continue.
    task::yield_now().await;
    println!("Main task resumed");
}
```

---

Understanding how to effectively manage tasks is fundamental to building robust Tokio applications. Now that you know how to run and coordinate concurrent operations, you can explore how these tasks perform work.

Next, let's look at how Tokio handles [Asynchronous I/O](./concepts-io.md).
