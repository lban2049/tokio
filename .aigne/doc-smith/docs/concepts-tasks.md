# Tasks & Scheduling

In Tokio, a **task** is a lightweight, non-blocking unit of execution. Think of tasks as asynchronous green threads: they are similar to OS threads, but instead of being managed by the operating system, they are managed by the Tokio runtime. This makes creating and switching between tasks extremely cheap compared to OS threads.

Key characteristics of Tokio tasks include:

*   **Lightweight**: Creating, running, and destroying large numbers of tasks has very low overhead.
*   **Cooperative**: Tasks run until they voluntarily yield control to the scheduler (usually at an `.await` point), allowing other tasks to run. This is in contrast to the preemptive multitasking used by OS threads.
*   **Non-blocking**: Tasks should not perform operations that block the thread, such as synchronous I/O or heavy CPU computations. Doing so would prevent other tasks on the same thread from making progress. Tokio provides specific tools for handling such cases.

```d2
direction: down

"Tokio Runtime" {
  shape: cloud

  "Worker Threads (Core)" {
    style.stroke-dash: 2
    "Worker Thread 1" {
      "Async Task A"
      "Async Task B"
    }
    "Worker Thread 2" {
      "Async Task C"
    }
  }

  "Blocking Thread Pool" {
    shape: package
    "Blocking Task X"
    "Blocking Task Y"
  }

  "Worker Thread 1" -> "Async Task A": polls
  "Worker Thread 1" -> "Async Task B": polls
  "Worker Thread 2" -> "Async Task C": polls
}
```

This section covers the essential patterns for working with tasks, from creating them to managing their execution and handling special cases like blocking operations.

## Spawning Tasks

The most fundamental operation is creating, or *spawning*, a new asynchronous task. This is done using the `tokio::spawn` function, which accepts a future and immediately begins executing it concurrently on the runtime.

`tokio::spawn` returns a `JoinHandle`, which is itself a future. You can `.await` the `JoinHandle` to get the output of the spawned task. This is how you can wait for a task to complete and retrieve its result.

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    let s = format!("Processing background task {}.", id);
    println!("{}", s);
    // Simulate some work
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    s
}

#[tokio::main]
async fn main() {
    // Spawn a new task.
    let join_handle = tokio::spawn(my_background_op(1));

    // Do other work in the main task.
    println!("Doing other work in main task.");

    // Await the result of the spawned task.
    match join_handle.await {
        Ok(result) => println!("Spawned task completed with result: '{}'", result),
        Err(e) => println!("Spawned task failed: {:?}", e),
    }
}
```

If a spawned task panics, awaiting its `JoinHandle` will return a `JoinError` indicating the failure.

## Managing Multiple Tasks with `JoinSet`

When you need to manage a group of tasks, a `JoinSet` is a powerful tool. It allows you to spawn multiple tasks and await their results as they complete, without needing to know which one will finish first.

When the `JoinSet` is dropped, all tasks still remaining in the set are automatically aborted.

```rust
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

    println!("Waiting for tasks to complete...");
    while let Some(res) = set.join_next().await {
        match res {
            Ok(val) => println!("Task {} completed.", val),
            Err(e) => println!("A task failed: {:?}", e),
        }
    }
    println!("All tasks finished.");
}
```

In this example, tasks are processed in the order they finish (4, 3, 2, 1, 0), not the order they were spawned.

## Task Cancellation

Tasks can be cancelled using the `abort` method on their `JoinHandle` or an `AbortHandle`. Cancellation is a signal that requests the task to shut down at its next `.await` point. Once a task is cancelled and finishes shutting down, awaiting its `JoinHandle` will result in a `JoinError` where `is_cancelled()` returns `true`.

It's important to note that `abort()` schedules the cancellation and returns immediately; it does not wait for the task to stop running. To ensure a task is fully stopped, you should `abort()` it and then `.await` its `JoinHandle`.

Tasks spawned with `spawn_blocking` cannot be aborted once they begin execution.

## Handling Blocking Operations

Because Tokio's scheduler is cooperative, a task must not perform blocking operations on a worker thread, as this would stall all other tasks on that same thread. To handle code that must block (e.g., synchronous file I/O, CPU-intensive computations), Tokio provides two main solutions.

<x-cards data-columns="2">
  <x-card data-title="spawn_blocking" data-icon="lucide:cpu">
    Runs a blocking function on a separate thread pool dedicated to blocking tasks. This is the most common way to integrate blocking code into an async application.
  </x-card>
  <x-card data-title="block_in_place" data-icon="lucide:pause-circle">
    Transitions the current worker thread into a blocking thread, allowing the runtime to migrate other tasks to a new worker. This can be more efficient by avoiding a context switch but is only available on the multi-threaded runtime.
  </x-card>
</x-cards>

### Using `spawn_blocking`

This function moves the blocking operation off the main async worker threads, preventing it from interfering with other asynchronous tasks.

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut data = "start-".to_string();

    let result = task::spawn_blocking(move || {
        // This is running on a blocking thread.
        // Synchronous, CPU-heavy work is acceptable here.
        std::thread::sleep(std::time::Duration::from_secs(1));
        data.push_str("end");
        data
    }).await?;

    assert_eq!(result, "start-end");
    Ok(())
}
```

### Using `block_in_place`

This function should be used when you are already inside an async task on a multi-threaded runtime and need to perform a short-lived blocking operation.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let result = task::block_in_place(|| {
        // This code is now running on a thread that is allowed to block.
        std::thread::sleep(std::time::Duration::from_secs(1));
        "blocking completed"
    });

    assert_eq!(result, "blocking completed");
}
```

## Yielding Control

Occasionally, you may want a task to voluntarily give up its execution time and allow other tasks to run. The `tokio::task::yield_now().await` function does exactly this. It yields control back to the Tokio scheduler, which will place the current task at the back of the queue and schedule another ready task.

This can be useful for ensuring long-running computations don't starve other tasks, even if the computation itself is fully asynchronous.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("[spawned] Task starting");
        // ... work ...
        println!("[spawned] Task finished");
    });

    println!("[main] Before yield");
    // Yield, allowing the newly-spawned task to potentially execute.
    task::yield_now().await;
    println!("[main] After yield");
}
```

You have now learned the fundamentals of creating and managing tasks in Tokio. With these concepts, you can build concurrent applications that are both efficient and scalable.

Next, learn how to perform non-blocking I/O operations, a common activity for most tasks.

<x-card data-title="Next: Asynchronous I/O" data-icon="lucide:arrow-right" data-href="/concepts/io" data-cta="Read More" >
  Explore Tokio's non-blocking primitives for networking, filesystem operations, and more.
</x-card>