# Synchronization

Tokio programs are often organized as a set of independent tasks that are executed concurrently. To manage communication and shared state between these tasks, Tokio provides a suite of synchronization primitives.

These primitives can be categorized into two main groups:

1.  **Message Passing**: Tasks communicate by sending messages to each other through channels. This approach is often preferred as it avoids the complexities of shared state.
2.  **State Synchronization**: When shared state is necessary, Tokio provides asynchronous versions of standard library primitives like `Mutex` and `RwLock` that allow tasks to safely access and modify shared data without blocking threads.

## Message Passing with Channels

Channels are the primary tool for message passing in Tokio. They allow one or more tasks to send messages to one or more receiving tasks. Tokio offers several types of channels, each suited for different communication patterns.

### `oneshot` Channel

A `oneshot` channel is used for sending a single value from a single producer to a single consumer. It's ideal for returning the result of a computation from a spawned task to its parent.

```rust
use tokio::sync::oneshot;

async fn some_computation() -> String {
    "represents the result of the computation".to_string()
}

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let res = some_computation().await;
        tx.send(res).unwrap();
    });

    // Do other work while the computation is happening...

    // Wait for the result
    match rx.await {
        Ok(v) => println!("got = {:?}", v),
        Err(_) => println!("the sender dropped"),
    }
}
```
If a task's final action is to produce a result, you can often use its `JoinHandle` directly instead of a `oneshot` channel.

### `mpsc` Channel

The multi-producer, single-consumer (`mpsc`) channel allows many tasks to send messages to a single receiving task. This is useful for distributing work to a worker task or aggregating results from multiple sources.

`mpsc` channels are bounded, meaning they have a fixed capacity. If the channel is full, senders will wait asynchronously until there is space available, providing backpressure.

Here's an example of using an `mpsc` channel combined with `oneshot` channels to manage a shared counter, demonstrating a request-response pattern:

```rust
use tokio::sync::{oneshot, mpsc};

// Define the command for our counter task
enum Command {
    Increment,
}

#[tokio::main]
async fn main() {
    // Create a channel for commands. The sender sends the command and a oneshot sender
    // for the response. The receiver gets the value before the increment.
    let (cmd_tx, mut cmd_rx) = mpsc::channel::<(Command, oneshot::Sender<u64>)>(100);

    // Spawn a task to manage the counter state
    tokio::spawn(async move {
        let mut counter: u64 = 0;

        while let Some((cmd, response_tx)) = cmd_rx.recv().await {
            match cmd {
                Command::Increment => {
                    let prev = counter;
                    counter += 1;
                    response_tx.send(prev).unwrap();
                }
            }
        }
    });

    let mut join_handles = vec![];

    // Spawn 10 tasks to increment the counter
    for _ in 0..10 {
        let cmd_tx = cmd_tx.clone();

        join_handles.push(tokio::spawn(async move {
            let (resp_tx, resp_rx) = oneshot::channel();

            // Send the increment command and the response channel
            cmd_tx.send((Command::Increment, resp_tx)).await.ok().unwrap();
            
            // Wait for the response
            let res = resp_rx.await.unwrap();
            println!("previous value = {}", res);
        }));
    }

    // Wait for all tasks to complete
    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

### `broadcast` Channel

A multi-producer, multi-consumer `broadcast` channel allows multiple senders to broadcast messages to multiple receivers. Every receiver sees every message. This is useful for "fan-out" patterns like chat systems or pub/sub models.

If a receiver is too slow and falls behind, it will receive a `Lagged` error, indicating that it has missed messages. It can then resume receiving from the oldest available message.

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, mut rx1) = broadcast::channel(16);
    let mut rx2 = tx.subscribe();

    tokio::spawn(async move {
        assert_eq!(rx1.recv().await.unwrap(), 10);
        assert_eq!(rx1.recv().await.unwrap(), 20);
    });

    tokio::spawn(async move {
        assert_eq!(rx2.recv().await.unwrap(), 10);
        assert_eq!(rx2.recv().await.unwrap(), 20);
    });

    tx.send(10).unwrap();
    tx.send(20).unwrap();
}
```

### `watch` Channel

The `watch` channel is a single-producer, multi-consumer channel that only stores the most recent value sent. Receivers are notified when a new value is sent, but they are not guaranteed to see every intermediate value.

This makes it ideal for broadcasting state changes, such as configuration updates, where consumers only care about the latest version.

```rust
use tokio::sync::watch;
use tokio::time::{self, Duration};

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config { timeout: Duration }

#[tokio::main]
async fn main() {
    let initial_config = Config { timeout: Duration::from_secs(1) };
    let (tx, mut rx) = watch::channel(initial_config.clone());

    // Spawn a task to listen for config changes
    tokio::spawn(async move {
        loop {
            // Wait until the sender sends a new value
            if rx.changed().await.is_err() {
                // Sender was dropped
                break;
            }
            let new_config = rx.borrow().clone();
            println!("Config changed to: {:?}", new_config);
        }
    });

    // Simulate updating the config
    time::sleep(Duration::from_millis(100)).await;
    tx.send(Config { timeout: Duration::from_secs(5) }).unwrap();

    time::sleep(Duration::from_millis(100)).await;
}
```

## State Synchronization

For situations where tasks need to directly share and modify state, Tokio provides asynchronous versions of the standard library's synchronization primitives. These primitives wait asynchronously instead of blocking the thread.

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    Provides mutual exclusion, ensuring that only one task can access the contained data at a time. It guarantees fair, first-in, first-out access.
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open">
    A reader-writer lock that allows any number of readers or at most one writer at a time. It is write-preferring to prevent writer starvation.
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    Limits the number of concurrent tasks that can access a resource. Tasks can acquire permits, and will wait if the limit is reached.
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    Enables multiple tasks to wait until all of them have reached a certain point of execution before any of them continue.
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    A basic primitive that allows one or more tasks to wait for a notification from another task without sending any data.
  </x-card>
</x-cards>

### When to use Tokio's `Mutex` vs. `std::sync::Mutex`

It is often acceptable and even preferable to use the blocking `Mutex` from the standard library in asynchronous code. The key feature of `tokio::sync::Mutex` is its ability to remain locked across an `.await` point. This makes it more complex and potentially more expensive.

-   **Use `std::sync::Mutex`** when the data being protected does not involve I/O and the lock is held for a short duration without crossing an `.await` point.
-   **Use `tokio::sync::Mutex`** when you need to hold a lock across an `.await` point, such as when protecting shared access to an I/O resource like a database connection pool.

### Example: Using `RwLock`

```rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // many reader locks can be held at once
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // read locks are dropped here

    // only one write lock may be held
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // write lock is dropped here
}
```

## Runtime Compatibility

All synchronization primitives in Tokio are runtime-agnostic. You can move them between different Tokio runtimes or even use them in non-Tokio environments. However, features like cooperative scheduling are only active when used within a Tokio runtime.

---

With an understanding of how to manage state and communication, you can now explore how to handle time-based operations. Continue to the [Timers](./concepts-timers.md) section to learn about sleeps, intervals, and timeouts.