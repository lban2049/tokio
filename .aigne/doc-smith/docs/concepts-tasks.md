# Tasks & Scheduling

At the heart of any Tokio application are asynchronous tasks. A task is a lightweight, non-blocking unit of execution, similar to Go's goroutines or Erlang's processes. Instead of being managed by the operating system like traditional threads, tasks are managed by the Tokio runtime, which makes them incredibly cheap to create and manage.

This section covers the fundamental concepts of working with tasks in Tokio, from creating them to managing their lifecycle and handling different types of workloads.

## What are Tasks?

Key characteristics of Tokio tasks:

*   **Lightweight:** Creating, switching, and destroying tasks has very low overhead compared to OS threads because it doesn't require a system context switch.
*   **Cooperatively Scheduled:** Tasks run until they yield control back to the scheduler, typically at an `.await` point. This is different from preemptive multitasking in OS threads, where the OS can interrupt a thread at any time.
*   **Non-blocking:** Tasks should never perform blocking operations like traditional file I/O or heavy, long-running computations. Doing so would prevent other tasks on the same thread from making progress. Tokio provides specific tools for handling such cases.

## Spawning Tasks

The most common way to start a concurrent operation is with `tokio::spawn`. This function takes an asynchronous block (a `Future`) and immediately starts executing it on the Tokio runtime, returning a `JoinHandle`.

```rust Spawning a Task icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        // This is running in a new, concurrent task.
        "hello world!"
    });

    // The original task continues running independently.
    println!("Spawned a task.");

    // We can wait for the spawned task to finish.
    let result = handle.await.unwrap();
    assert_eq!(result, "hello world!");
}
```

The `JoinHandle` is a future that resolves to the output of the spawned task. Awaiting the handle allows you to get the result back. If the spawned task panics, awaiting its `JoinHandle` will return an error.

## Task Cancellation

Tasks can be cancelled, which signals them to stop execution at the next available `.await` point. This is a graceful shutdown mechanism. Cancellation is typically done using the `abort` method on the task's `JoinHandle`.

```rust Cancelling a Task icon=logos:rust
use tokio::time::{self, Duration};

#[tokio::main]
async fn main() {
    let task = tokio::spawn(async {
        // This task will run for a while...
        time::sleep(Duration::from_secs(10)).await;
        println!("Task finished normally.");
    });

    // Let it run for a moment.
    time::sleep(Duration::from_millis(100)).await;

    // Now, abort the task.
    task.abort();

    // Awaiting a cancelled task results in an error.
    let result = task.await;
    assert!(result.is_err());
    println!("Task was aborted.");
}
```

When a task is aborted, it stops at the `.await` it was suspended at, and its local variables are dropped. It's important to note that tasks spawned with `spawn_blocking` cannot be aborted because they are not asynchronous and do not have `.await` points.

## Handling Blocking Code

Because tasks must not block the thread they are running on, Tokio provides specific functions to handle synchronous, blocking, or CPU-intensive code.

<x-cards data-columns="2">
  <x-card data-title="spawn_blocking" data-icon="lucide:cpu">
    Runs a blocking function on a dedicated thread pool for blocking tasks, without interfering with the async runtime. It returns a `JoinHandle` to await the result.
  </x-card>
  <x-card data-title="block_in_place" data-icon="lucide:pause-circle">
    Transitions the current worker thread into a blocking thread, moving other async tasks to a different worker. This can be more efficient by avoiding a context switch, but is only available on the multi-threaded runtime.
  </x-card>
</x-cards>

### Using `spawn_blocking`

This is the recommended way to run blocking code. It offloads the work to a separate thread pool, keeping the async core threads free.

```rust Using spawn_blocking icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let handle = task::spawn_blocking(|| {
        // This is a stand-in for a compute-heavy or blocking I/O operation.
        std::thread::sleep(std::time::Duration::from_millis(500));
        "done"
    });

    let result = handle.await?;
    assert_eq!(result, "done");
    Ok(())
}
```

### Using `block_in_place`

When you need to execute a shorter blocking operation from within an async task on the multi-threaded runtime, `block_in_place` can be a good choice.

```rust Using block_in_place icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() {
    let result = task::block_in_place(|| {
        // This runs on the current worker thread, which is temporarily
        // marked as a blocking thread.
        std::thread::sleep(std::time::Duration::from_millis(500));
        "done"
    });

    assert_eq!(result, "done");
}
```

## Yielding

You can voluntarily yield control back to the Tokio scheduler by calling `tokio::task::yield_now()`. This allows the scheduler to run other pending tasks before resuming the current one. This can be useful for ensuring long-running tasks don't monopolize CPU time.

```rust Yielding a Task icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        println!("spawned task done!");
    });

    println!("main task yielding.");
    // Yield, allowing the newly-spawned task to potentially execute first.
    task::yield_now().await;
    println!("main task done!");
}
```

## Managing Many Tasks with `JoinSet`

When you need to spawn and manage a dynamic number of tasks, using a `JoinSet` is more convenient than collecting `JoinHandle`s in a `Vec`. A `JoinSet` allows you to await tasks as they complete, in completion order, not spawn order.

When the `JoinSet` is dropped, all tasks still remaining in the set are automatically aborted.

```rust Managing Tasks with JoinSet icon=logos:rust
use tokio::task::JoinSet;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..5 {
        set.spawn(async move {
            sleep(Duration::from_millis((5 - i) * 100)).await;
            i
        });
    }

    while let Some(res) = set.join_next().await {
        let completed_task_index = res.unwrap();
        println!("Task {} completed.", completed_task_index);
    }
    
    println!("All tasks finished.");
}
```

This example demonstrates how tasks with different sleep durations complete out of order, and `join_next` efficiently retrieves their results as they become available.

---

Now that you understand how to create and manage concurrent operations with tasks, you're ready to explore how these tasks can perform useful work. Continue to [Asynchronous I/O](./concepts-io.md) to learn about networking and file operations, or to [Synchronization](./concepts-synchronization.md) to see how tasks can communicate and share data safely.