# Tasks

This module provides APIs for managing asynchronous tasks, which are lightweight, non-blocking units of execution, also known as green threads. Unlike OS threads, tasks are managed by the Tokio runtime, making them cheap to create and switch between.

For a more in-depth conceptual understanding of tasks, please see the [Tasks & Scheduling guide](./concepts-tasks.md).

This page covers the primary APIs for:
- Spawning new tasks concurrently.
- Handling blocking code within an asynchronous application.
- Managing collections of tasks.
- Working with `!Send` data.

## Spawning Tasks

The primary way to create a new task is with the `tokio::spawn` function.

### `tokio::spawn`

Spawns a new asynchronous task, returning a `JoinHandle` for it. The spawned task may execute on the current thread, or it may be sent to a different thread, depending on the runtime's configuration.

The provided future starts running immediately, even if the `JoinHandle` is not awaited.

**When to use:** Use `spawn` for any `async` operation that you want to run concurrently with other tasks.

**Signature**
```rust
fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
```

**Example: Processing network connections**

In a server, each incoming connection can be processed in its own task.

```rust
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

        tokio::spawn(async move {
            // Process each socket concurrently.
            process(socket).await
        });
    }
}
```

**Example: Awaiting a result**

The `JoinHandle` returned by `spawn` is a future that resolves to the output of the spawned task. You can `.await` it to get the result.

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    format!("Finished background task {}.", id)
}

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(my_background_op(1));

    // Do other work here...

    let result = handle.await.unwrap();
    println!("{}", result);
}
```
If the spawned task panics, awaiting its `JoinHandle` will return a `JoinError`.


## Handling Blocking Operations

Asynchronous tasks should not perform blocking operations, as this prevents the worker thread from executing other tasks. Tokio provides dedicated APIs for running blocking code safely.

This diagram illustrates how different spawning functions interact with the runtime's thread pools.

```d2
direction: down

"Async Context (e.g., within `async fn`)": {
  shape: cloud
  style.fill: "#E6F7FF"

  "tokio::spawn": {
    "Spawns a new `Send` future on the runtime's scheduler.": {
      "Returns a `JoinHandle` immediately.": "Task runs concurrently."
    }
  }

  "tokio::spawn_blocking": {
    "Spawns a blocking function on a separate, dedicated thread pool.": {
      "Returns a `JoinHandle` immediately.": "Does not block the async worker thread."
    }
  }

  "tokio::block_in_place": {
    "Pauses the current async task and transitions the worker thread to a blocking state.": {
      "Runtime may spawn a new worker to run other tasks.": "Function runs on the *same* thread."
    }
  }
}

"Tokio Runtime": {
  "Scheduler Threads (Worker Pool)": {
    style.fill: "#D1F0E0"
  }
  "Blocking Thread Pool": {
    style.fill: "#FFF0E6"
  }
}

"Async Context (e.g., within `async fn`)"."tokio::spawn" -> "Tokio Runtime"."Scheduler Threads (Worker Pool)": "Schedules async task"
"Async Context (e.g., within `async fn`)"."tokio::spawn_blocking" -> "Tokio Runtime"."Blocking Thread Pool": "Schedules blocking task"
"Async Context (e.g., within `async fn`)"."tokio::block_in_place" -> "Tokio Runtime"."Scheduler Threads (Worker Pool)": "Signals to runtime"
```

| Function | Task Type | Execution Context | Use Case |
|---|---|---|---|
| `tokio::spawn` | Asynchronous (`Future`) | Tokio worker thread pool | Standard concurrent async operations. |
| `tokio::spawn_blocking` | Synchronous (`FnOnce`) | Dedicated blocking thread pool | Long-running, CPU-intensive work or synchronous I/O. |
| `tokio::block_in_place` | Synchronous (`FnOnce`) | The *current* Tokio worker thread | Short, unavoidable blocking calls within async code on the multi-threaded runtime. |

### `tokio::spawn_blocking`

Runs a closure on a thread where blocking is acceptable. This is the recommended way to handle CPU-bound work or synchronous I/O.

**When to use:** For computations that would otherwise block an async worker thread for a significant amount of time.

**Signature**
```rust
fn spawn_blocking<F, R>(f: F) -> JoinHandle<R>
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static,
```

**Example**
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

### `tokio::block_in_place`

Transitions the current worker thread to a blocking thread, allowing the runtime to hand off other tasks to a new worker. This can be more efficient than `spawn_blocking` as it avoids a context switch, but it is only available on the multi-threaded runtime.

**When to use:** For short-lived blocking operations that must run on the current thread.

**Signature**
```rust
fn block_in_place<F, R>(f: F) -> R
where
    F: FnOnce() -> R,
```

**Example**
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

**Note:** This function will panic if called from a `current_thread` runtime.

## Yielding Execution

### `tokio::yield_now`

Yields execution back to the Tokio scheduler, allowing other tasks to run. The current task is re-queued and will be polled again later.

**When to use:** To give other tasks a chance to run, especially in long-running tasks that do not have natural `.await` points.

**Signature**
```rust
async fn yield_now()
```

**Example**
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

## Managing Task Collections

### `JoinSet<T>`

A `JoinSet` is a collection of tasks that can be awaited as they complete. It is useful for managing a dynamic number of child tasks.

**When to use:** When you need to spawn multiple tasks and process their results in the order of completion, rather than the order of spawning.

**Example**

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

Key methods include:
- `spawn()`: Adds a new task to the set.
- `join_next()`: Awaits the next task in the set to complete.
- `abort_all()`: Aborts all tasks in the set.
- `shutdown()`: Aborts all tasks and waits for them to finish.


## Working with `!Send` Futures

Standard `tokio::spawn` requires futures to be `Send`, meaning they can be safely moved between threads. For futures that are `!Send` (e.g., they hold an `Rc<T>`), you must use a `LocalSet`.

### `LocalSet` and `spawn_local`

A `LocalSet` runs a group of tasks on the current thread, allowing `!Send` futures to be spawned using `spawn_local`.

**When to use:** When you need to work with data types that cannot be sent across threads, such as `Rc` or certain types from single-threaded libraries.

**Example**

```rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("my nonsend data...");

    // Construct a local task set that can run `!Send` futures.
    let local = task::LocalSet::new();

    // Run the local task set until the future completes.
    local.run_until(async move {
        let nonsend_data = nonsend_data.clone();
        // `spawn_local` ensures the future runs on the current thread's LocalSet.
        task::spawn_local(async move {
            println!("{}", nonsend_data);
        }).await.unwrap();
    }).await;
}
```

**Note:** `LocalSet::run_until` and awaiting a `LocalSet` can only be used directly within a runtime's `block_on` call, such as in `#[tokio::main]`.

## Next Steps

Now that you understand how to create and manage tasks, you may want to coordinate their execution or schedule work based on time.

<x-cards data-columns="2">
  <x-card data-title="Synchronization Primitives" data-icon="lucide:git-merge" data-href="/api/sync">
    Learn how to use channels, mutexes, and semaphores to manage shared state between tasks.
  </x-card>
  <x-card data-title="Time" data-icon="lucide:timer" data-href="/api/time">
    Explore utilities for tracking time, including sleeps, intervals, and timeouts.
  </x-card>
</x-cards>
