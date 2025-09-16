# Synchronization Primitives

Tokio applications are typically structured as a collection of independent tasks that run concurrently. To ensure these tasks can communicate and coordinate effectively, Tokio provides a rich set of synchronization primitives. These tools are essential for managing shared state and orchestrating complex asynchronous workflows.

This guide covers two primary categories of synchronization:

1.  **Message Passing:** Using channels to send data between tasks, which helps avoid the complexities of shared state.
2.  **State Synchronization:** Using primitives like `Mutex` and `RwLock` to control access to shared data, similar to their counterparts in the standard library but designed for an asynchronous context.

For a foundational understanding of how Tokio manages concurrent operations, you may want to review our guide on [Spawning & Managing Tasks](./tasks-scheduling-spawning.md).

## Message Passing with Channels

Message passing is a common and effective pattern for synchronization in concurrent systems. By sending messages through channels, tasks can operate independently without needing direct access to shared memory, reducing the risk of race conditions and simplifying logic.

Tokio offers several types of channels, each tailored for different communication patterns.

### oneshot channel

The `oneshot` channel is designed for sending a single value from one task to another. It's ideal for scenarios where a task needs to return a single result to a waiter, such as the outcome of a computation.

```rust Example: Using a oneshot channel icon=logos:rust
use tokio::sync::oneshot;

async fn perform_computation() -> String {
    // Simulate some work
    "computation result".to_string()
}

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let result = perform_computation().await;
        // The send operation can fail if the receiver is dropped.
        let _ = tx.send(result);
    });

    // Do other work while the computation is running...

    // Wait for the result
    match rx.await {
        Ok(value) => println!("Got value: {}", value),
        Err(_) => println!("The sender was dropped"),
    }
}
```

Note that if a task's final action is to produce a result, you can often use its `JoinHandle` directly instead of a `oneshot` channel. Awaiting the `JoinHandle` returns a `Result`, which will be an `Err` if the task panics.

### mpsc channel

The `mpsc` (multi-producer, single-consumer) channel allows many tasks to send messages to a single receiving task. This is useful for distributing work to a worker task or aggregating results from multiple computations.

When creating an `mpsc` channel, you must specify a capacity, which is the maximum number of messages that can be buffered. This capacity is crucial for managing backpressure; if the channel is full, senders will wait asynchronously until there is space.

```rust Example: Using an mpsc channel icon=logos:rust
use tokio::sync::mpsc;

async fn compute(input: u32) -> String {
    format!("Result for input {}", input)
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    // Spawn a producer task
    tokio::spawn(async move {
        for i in 0..10 {
            let result = compute(i).await;
            if tx.send(result).await.is_err() {
                eprintln!("receiver dropped");
                return;
            }
        }
    });

    // The receiver will get `None` when all senders have been dropped.
    while let Some(message) = rx.recv().await {
        println!("Received: {}", message);
    }
}
```

### broadcast channel

The `broadcast` channel supports a multi-producer, multi-consumer pattern. Every message sent is delivered to every active receiver. This is often used for "fan-out" scenarios like pub/sub systems or chat applications.

Like `mpsc` channels, broadcast channels have a fixed capacity. If a sender adds a message to a full channel, the oldest message is dropped to make room. A receiver that falls behind and misses messages will receive a `RecvError::Lagged` error.

```rust Example: Using a broadcast channel icon=logos:rust
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

### watch channel

The `watch` channel is a specialized single-producer, multi-consumer channel that only stores the most recent value sent. Receivers are notified when a new value is sent, but they are not guaranteed to see every intermediate value.

This makes it ideal for broadcasting state changes, such as updates to application configuration, where consumers only care about the latest version.

```rust Example: Using a watch channel for configuration icon=logos:rust
use tokio::sync::watch;
use tokio::time::{self, Duration};

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config { timeout: Duration }

#[tokio::main]
async fn main() {
    let initial_config = Config { timeout: Duration::from_secs(1) };
    let (tx, mut rx) = watch::channel(initial_config.clone());

    // Spawn a task to monitor for configuration changes
    tokio::spawn(async move {
        // In a real app, this would load from a file or service
        time::sleep(Duration::from_millis(50)).await;
        let new_config = Config { timeout: Duration::from_secs(5) };
        tx.send(new_config).unwrap();
    });

    println!("Initial timeout: {:?}", rx.borrow().timeout);

    // Wait for the configuration to change
    if rx.changed().await.is_ok() {
        println!("New timeout: {:?}", rx.borrow().timeout);
    }
}
```

## State Synchronization

For situations where tasks need to directly share and modify state, Tokio provides asynchronous versions of the synchronization primitives found in the standard library. These types integrate with the Tokio runtime, allowing tasks to wait asynchronously instead of blocking threads.

<x-cards>
  <x-card data-title="Mutex" data-icon="lucide:lock">
    A mutual exclusion primitive that ensures only one task can access some data at any given time. It provides asynchronous `lock` methods.
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open-check">
    A reader-writer lock that allows any number of readers or at most one writer at a time. This can be more efficient than a `Mutex` for read-heavy workloads.
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    A counting semaphore that limits the number of concurrent tasks that can access a resource. Useful for rate limiting or managing resource pools.
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    Enables multiple tasks to wait for each other to reach a certain point in the program before all continuing together.
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    A basic task notification primitive. `Notify` can be used to signal a single waiting task to wake up and resume processing, without sending any data.
  </x-card>
  <x-card data-title="OnceCell" data-icon="lucide:box-select">
    A thread-safe cell that can be written to only once, allowing for asynchronous initialization of shared data on first use.
  </x-card>
</x-cards>

## Next Steps

Now that you understand how to synchronize tasks, the next step is to learn how to manage time-based operations.

<x-card data-title="Time, Delays, and Timeouts" data-href="/tasks-scheduling/time" data-icon="lucide:timer">
  Learn how to use `sleep`, `interval`, and `timeout` to control the timing of your asynchronous operations.
</x-card>
