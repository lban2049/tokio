# Spawning & Managing Tasks

In Tokio, a task is a lightweight, non-blocking unit of execution. To achieve concurrency, you run your asynchronous code as tasks. This section covers the primary ways to spawn and manage these tasks, from basic fire-and-forget operations to handling large, dynamic sets of concurrent work.

## Spawning Asynchronous Tasks with `tokio::spawn`

The most common way to create a new task is with the `tokio::spawn` function. It takes an `async` block or any future and submits it to the Tokio runtime to run concurrently.

The spawned task must be `'static` and the future must be `Send`, meaning it's safe to move between threads. This allows the Tokio scheduler to efficiently execute the task on any available worker thread.

```rust icon=logos:rust Spawning a connection handler
use tokio::net::{TcpListener, TcpStream};
use std::io;

async fn process(socket: TcpStream) {
    // ... handle the connection ...
    # drop(socket);
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        // A new task is spawned for each inbound socket.
        // The `move` keyword is used to transfer ownership of the socket
        // into the new task.
        tokio::spawn(async move {
            // Process each socket concurrently.
            process(socket).await
        });
    }
}
```

### Awaiting Task Completion with `JoinHandle`

When you spawn a task, `tokio::spawn` returns a `JoinHandle`. This handle is itself a future that you can `.await` to get the output of the spawned task. This is how you can wait for a concurrent operation to complete and retrieve its result.

```rust icon=logos:rust Awaiting a JoinHandle
# #[tokio::main(flavor = "current_thread")] async fn main() {
async fn my_background_op(id: i32) -> String {
    let s = format!("Starting background task {}.
", id);
    println!("{}", s);
    s
}

let handle = tokio::spawn(my_background_op(1));

// Do other work while the task runs...

let output = handle.await.unwrap();
println!("{:?}", output);
# }
```

If the spawned task panics, awaiting its `JoinHandle` will return a `Result` containing a `JoinError`.

### Task Cancellation

Tasks can be cancelled using the `abort` method on their `JoinHandle`. This signals the task to shut down the next time it yields at an `.await` point. To wait for the cancellation to complete, you must still await the `JoinHandle`.

```rust icon=logos:rust Aborting a task
use std::time::Duration;
use tokio::time::sleep;

# #[tokio::main] async fn main() {
let handle = tokio::spawn(async {
    println!("Task running...");
    sleep(Duration::from_secs(10)).await;
    println!("Task finished!"); // This will not be printed
});

sleep(Duration::from_millis(100)).await;

// Abort the task
handle.abort();

// Awaiting a cancelled task returns a `JoinError`
let err = handle.await.unwrap_err();
assert!(err.is_cancelled());
# }
```

## Handling Blocking Operations with `spawn_blocking`

Asynchronous tasks should never perform blocking operations, as this can halt the entire worker thread, preventing other tasks from running. For synchronous I/O or CPU-intensive computations, use `tokio::task::spawn_blocking`.

This function runs the provided closure on a dedicated thread pool for blocking tasks, ensuring it doesn't interfere with the asynchronous scheduler.

```rust icon=logos:rust Running a blocking operation
use tokio::task;

# async fn docs() -> Result<(), Box<dyn std::error::Error>>{
let v = "Hello, ".to_string();
let join_handle = task::spawn_blocking(move || {
    // This is running on a blocking thread.
    // Blocking here is okay.
    let result = v + "world";
    result
});

let result = join_handle.await?;
assert_eq!(result, "Hello, world");
# Ok(())
# }
```

Note that tasks spawned with `spawn_blocking` cannot be aborted once they have started running. The runtime will wait for them to complete during shutdown.

## Working with `!Send` Futures using `spawn_local`

Some types, like `std::rc::Rc`, are not `Send` and cannot be safely moved between threads. If your future holds such a type across an `.await` point, you cannot use `tokio::spawn`. In these cases, you can use a `LocalSet` to run tasks on a single thread.

Within a `LocalSet`, you use `tokio::task::spawn_local` to spawn `!Send` futures. The spawned future will always run on the same thread that created it.

```rust icon=logos:rust Spawning a !Send future
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    // Rc is not `Send`
    let nonsend_data = Rc::new("my nonsend data...");

    // A LocalSet allows us to run `!Send` futures.
    let local = task::LocalSet::new();

    // Run the LocalSet
    local.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();
        
        // spawn_local is used for `!Send` futures
        task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
            // ...
        }).await.unwrap();
    }).await;
}
```

## Managing Groups of Tasks with `JoinSet`

When you need to manage a dynamic collection of tasks, `JoinSet` is the ideal tool. It allows you to spawn multiple tasks and await their completion in the order they finish, without needing to manually manage a `Vec<JoinHandle>`.

When a `JoinSet` is dropped, all tasks within it are automatically aborted.

```rust icon=logos:rust Using JoinSet to manage multiple tasks
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

`JoinSet` provides a powerful and ergonomic API for managing concurrent operations, including methods like:

| Method           | Description                                                              |
| ---------------- | ------------------------------------------------------------------------ |
| `spawn`          | Spawns a `Send` task into the set.                                       |
| `spawn_local`    | Spawns a `!Send` task into the set (requires a `LocalSet`).              |
| `spawn_blocking` | Spawns a blocking task into the set.                                     |
| `join_next`      | Awaits the next task in the set to complete.                             |
| `abort_all`      | Aborts all tasks currently in the set.                                   |
| `shutdown`       | Aborts all tasks and waits for them to finish.                           |

## Yielding Execution with `yield_now`

Occasionally, you might want to give the Tokio scheduler a chance to run other pending tasks. You can do this by calling `tokio::task::yield_now().await`. This will place the current task at the back of the queue, allowing other tasks to be polled.

```rust icon=logos:rust Yielding to other tasks
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

However, use this sparingly. It's generally not guaranteed which task the scheduler will run next, and well-structured async code rarely needs explicit yielding.

---

With these tools, you can effectively create and manage concurrent operations in your Tokio application. The choice between `spawn`, `spawn_blocking`, and `spawn_local` depends entirely on the nature of the work your task needs to perform.

Next, learn how to coordinate and communicate between these tasks using [Synchronization Primitives](./tasks-scheduling-synchronization.md).
