# Tasks

Asynchronous green-threads for managing concurrent operations. This module provides the core tools for spawning, managing, and synchronizing asynchronous tasks within the Tokio runtime.

Tasks are the fundamental unit of execution in Tokio. They are lightweight, non-blocking, and cooperatively scheduled, allowing you to run a massive number of concurrent operations efficiently. For a deeper dive into the theory behind tasks, please see the [Tasks & Scheduling concepts guide](./concepts-tasks.md).

This page provides an API reference for the most common task-related functionalities.

### Core Functions

<x-cards data-columns="2">
  <x-card data-title="spawn" data-icon="lucide:play-circle">
    Spawns a new asynchronous task to run concurrently.
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:loader-2">
    Runs a blocking function on a dedicated thread pool, preventing it from blocking the async runtime.
  </x-card>
  <x-card data-title="yield_now" data-icon="lucide:rotate-cw">
    Yields execution back to the scheduler, allowing other tasks to run.
  </x-card>
  <x-card data-title="JoinSet" data-icon="lucide:box-select">
    A collection for managing a dynamic set of spawned tasks.
  </x-card>
</x-cards>

## Spawning Tasks

The primary way to create a new task is with the `tokio::spawn` function.

### `spawn`

Spawns a new asynchronous task, returning a `JoinHandle` for it. This is the asynchronous equivalent of `std::thread::spawn`.

The provided future starts running immediately in the background, even if the `JoinHandle` is not awaited. The task may be executed on the current thread or moved to a different worker thread, depending on the runtime's configuration.

```rust Spawning a task icon=logos:rust
use tokio::net::TcpListener;
use std::io;

async fn process_socket(socket: tokio::net::TcpStream) {
    // ... handle the connection
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        // Spawn a new task to process each connection concurrently.
        tokio::spawn(async move {
            process_socket(socket).await;
        });
    }
}
```

To run multiple tasks and await their results, you can store their `JoinHandle`s.

```rust Waiting for multiple tasks icon=logos:rust
# #[tokio::main(flavor = "current_thread")] async fn main() {
async fn my_background_op(id: i32) -> String {
    format!("Finished background task {}.", id)
}

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
# }
```

**Panics**

This function will panic if called from outside of a Tokio runtime context.

## Handling Blocking Operations

Asynchronous tasks should never perform blocking operations, as this would halt the entire worker thread, preventing other tasks from making progress. Tokio provides two functions to safely integrate blocking code.

### `spawn_blocking`

Runs a closure on a separate thread pool dedicated to blocking tasks. This is the preferred way to run CPU-bound code or synchronous I/O operations.

```rust Using spawn_blocking icon=logos:rust
use tokio::task;

async fn compute_and_save() -> Result<(), Box<dyn std::error::Error>> {
    let data_to_compute = "some complex data".to_string();

    let result = task::spawn_blocking(move || {
        // This runs on a blocking thread.
        // Perform compute-heavy work or synchronous I/O.
        let processed = data_to_compute.to_uppercase();
        std::fs::write("output.txt", processed)
    }).await?;

    result?;
    println!("Blocking operation complete.");
    Ok(())
}
```

Tasks spawned with `spawn_blocking` cannot be aborted directly. If the runtime is shut down, it will wait indefinitely for these tasks to complete unless a shutdown timeout is configured.

### `block_in_place`

This function transitions the *current* worker thread into a blocking thread, allowing a blocking operation to be executed without stalling the runtime. It does this by handing off other tasks on the current thread to a new worker thread. This can be more efficient than `spawn_blocking` as it avoids a context switch, but it is only available on the multi-threaded runtime.

```rust Using block_in_place icon=logos:rust
use tokio::task;

# async fn docs() {
let result = task::block_in_place(|| {
    // do some compute-heavy work or call synchronous code
    "blocking completed"
});

assert_eq!(result, "blocking completed");
# }
```

**Panics**

This function will panic if called from a `current_thread` runtime, as there are no other worker threads to offload tasks to.

## Yielding

### `yield_now`

Yields execution back to the Tokio scheduler, allowing other pending tasks to run. The current task is placed at the back of the queue and will be resumed later.

```rust Yielding execution icon=logos:rust
use tokio::task;

# #[tokio::main] async fn main() {
async {
    task::spawn(async {
        println!("spawned task done!")
    });

    // Yield, allowing the newly-spawned task to execute first.
    task::yield_now().await;
    println!("main task done!");
}
# .await;
# }
```

## Task Collections

### `JoinSet<T>`

A collection for managing a set of spawned tasks. It allows you to await tasks as they complete, in completion order, which is useful when you don't need to await all tasks at once or when tasks have different durations.

When a `JoinSet` is dropped, all tasks remaining in the set are aborted.

```rust Managing tasks with JoinSet icon=logos:rust
use tokio::task::JoinSet;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..5 {
        set.spawn(async move {
            tokio::time::sleep(Duration::from_millis(100 * i)).await;
            i
        });
    }

    while let Some(res) = set.join_next().await {
        let completed_task_index = res.unwrap();
        println!("Task {} completed!", completed_task_index);
    }
}
```

## `!Send` Futures

Standard `tokio::spawn` requires futures to be `Send`, meaning they can be safely moved between threads. For futures that are `!Send` (e.g., those holding an `Rc<T>`), you must use a `LocalSet`.

### `LocalSet` and `spawn_local`

A `LocalSet` executes tasks on the current thread. Any task spawned within the context of a `LocalSet` using `spawn_local` is guaranteed to remain on that thread, making it safe to use `!Send` types.

```rust Spawning a !Send future icon=logos:rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    // Rc is !Send
    let nonsend_data = Rc::new("my local data");

    let local_set = task::LocalSet::new();

    // Run the LocalSet
    local_set.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();

        // spawn_local can accept !Send futures.
        let handle = task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
            42
        });

        let result = handle.await.unwrap();
        assert_eq!(result, 42);
    }).await;
}
```

`spawn_local` will panic if called outside the context of a `LocalSet`.

## Task Handles & Cancellation

### `JoinHandle<T>`

Returned by `spawn` and `spawn_local`, a `JoinHandle` is a future that resolves to the output of the associated task. Awaiting the handle will wait for the task to complete.

If the task panics, awaiting its `JoinHandle` will return a `JoinError`.

```rust Handling a panicked task icon=logos:rust
use tokio::task;

# #[tokio::main] async fn main() {
let join = task::spawn(async {
    panic!("something bad happened!")
});

// The returned result indicates that the task failed.
assert!(join.await.is_err());
# }
```

#### Cancellation

You can cancel a task by calling the `abort()` method on its `JoinHandle`. This signals the task to shut down the next time it reaches an `.await` point. To wait for the cancellation to complete, you must still `.await` the handle.

```rust Aborting a task icon=logos:rust
use tokio::task;
use std::time::Duration;

# #[tokio::main] async fn main() {
let handle = task::spawn(async {
    // This task will run forever unless aborted.
    loop {
        tokio::time::sleep(Duration::from_secs(1)).await;
        println!("task is running...");
    }
});

tokio::time::sleep(Duration::from_millis(50)).await;
handle.abort();

let join_result = handle.await;
assert!(join_result.is_err());
assert!(join_result.unwrap_err().is_cancelled());
# }
```

### `AbortHandle`

An `AbortHandle` provides the ability to abort a task without being able to await its result. A task can have multiple `AbortHandle`s but only one `JoinHandle`. This is useful for separating the concern of task management from task completion.

---

This covers the core APIs for task management in Tokio. For details on how the runtime schedules and executes these tasks, see the [Runtime API reference](./api-runtime.md).