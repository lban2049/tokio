# Synchronization

Tokio programs are typically structured as a set of independent tasks that may execute on separate physical threads. The synchronization primitives provided by Tokio allow these independent tasks to communicate and coordinate safely.

There are two main categories of synchronization primitives:

1.  **Message Passing**: Tasks operate independently and send messages to each other through channels to synchronize. This approach is often preferred as it avoids the complexities of shared state.
2.  **State Synchronization**: Primitives like mutexes and semaphores that control access to shared state. These are asynchronous equivalents to the primitives found in the standard library.

## Message Passing with Channels

Message passing is a common and effective way to handle synchronization in Tokio. It's implemented using channels, which allow one or more producer tasks to send messages to one or more consumer tasks. Tokio offers several types of channels, each suited for different communication patterns.

```d2
direction: right

Producers: { 
  shape: person
  style.multiple: true 
}

Consumers: { 
  shape: person
  style.multiple: true 
}

Producer: { 
  shape: person
}

Consumer: { 
  shape: person
}

subgraph "oneshot": {
  label: "Single value, single use"
  Producer -> Consumer
}

subgraph "mpsc": {
  label: "Multiple values, single consumer"
  Producers -> Consumer
}

subgraph "broadcast": {
  label: "Multiple values, multiple consumers (fan-out)"
  Producers -> Consumers
}

subgraph "watch": {
  label: "Latest value, multiple consumers"
  Producers -> Consumers
}
```

### `oneshot` Channel

The `oneshot` channel is designed for sending a **single** value from one producer to one consumer. It's commonly used to return the result of a computation from a spawned task to a waiting task.

**Example: Receiving a computation result**

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

    // Do other work while the computation is happening

    // Wait for the computation result
    let res = rx.await.unwrap();
    println!("Got result: {}", res);
}
```
Note: If the task's final action is returning the result, you can often use the task's `JoinHandle` directly instead of a `oneshot` channel.

### `mpsc` Channel

The `mpsc` (multi-producer, single-consumer) channel allows sending **many** values from **many** producers to a **single** consumer. It's ideal for distributing work to a worker task or aggregating results from multiple sources.

The channel has a fixed capacity, which provides backpressure. If the channel is full, senders will wait asynchronously until there is space for a new message.

**Example: Streaming computation results**

```rust
use tokio::sync::mpsc;

async fn perform_computation(input: u32) -> String {
    format!("the result of computation {}", input)
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    tokio::spawn(async move {
        for i in 0..10 {
            let res = perform_computation(i).await;
            tx.send(res).await.unwrap();
        }
    });

    while let Some(res) = rx.recv().await {
        println!("got = {}", res);
    }
}
```

### `broadcast` Channel

The `broadcast` channel enables sending **many** values from **many** producers to **many** consumers. Every consumer receives **every** value sent. This is useful for "fan-out" patterns like pub/sub systems or chat applications.

If a receiver is too slow and falls behind, it will receive a `Lagged` error, and its internal cursor will be updated to the oldest message still in the channel.

**Example: Basic broadcast usage**

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

The `watch` channel is a multi-producer, multi-consumer channel that only stores the **most recent** value. Consumers are notified when a new value is sent, but they are not guaranteed to see every intermediate value. It's particularly useful for broadcasting configuration changes or signaling state updates, like a shutdown signal.

**Example: Notifying tasks of configuration changes**

```rust
use tokio::sync::watch;
use tokio::time::{self, Duration};

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config {
    timeout: Duration,
}

#[tokio::main]
async fn main() {
    let config = Config { timeout: Duration::from_secs(1) };
    let (tx, mut rx) = watch::channel(config);

    tokio::spawn(async move {
        // In a real application, you might monitor a file or another source.
        time::sleep(Duration::from_millis(500)).await;
        let new_config = Config { timeout: Duration::from_secs(2) };
        tx.send(new_config).unwrap();
    });

    loop {
        tokio::select! {
            _ = rx.changed() => {
                let new_timeout = rx.borrow().timeout;
                println!("Configuration changed: timeout is now {:?}", new_timeout);
                if new_timeout.as_secs() == 2 {
                    break;
                }
            }
        }
    }
}
```

## State Synchronization

For managing direct access to shared data, Tokio provides asynchronous versions of the standard library's synchronization primitives. These types wait asynchronously instead of blocking a thread.

| Primitive | Description |
|---|---|
| [`Mutex`](./api-sync.md) | A mutual exclusion lock that ensures only one task can access data at a time. It guarantees FIFO ordering for tasks waiting for the lock. |
| [`RwLock`](./api-sync.md) | A reader-writer lock that allows for either multiple readers or a single writer at any given time. It is write-preferring to prevent writer starvation. |
| [`Semaphore`](./api-sync.md) | Limits concurrent access to a resource by maintaining a set of permits. Tasks must acquire a permit before proceeding. |
| [`Barrier`](./api-sync.md) | Enables multiple tasks to wait for each other to reach a certain point in the program before all continuing together. |
| [`Notify`](./api-sync.md) | A basic primitive for notifying a single waiting task to resume its work, without sending any data. |

## Next Steps

Choosing the right synchronization primitive is crucial for writing correct and performant asynchronous code. For detailed usage and API information, please refer to the API reference.

<x-cards>
  <x-card data-title="API Reference: Synchronization" data-icon="lucide:book-open" data-href="/api/sync">
    Dive into the detailed API documentation for all of Tokio's synchronization primitives.
  </x-card>
  <x-card data-title="Next Concept: Timers" data-icon="lucide:arrow-right" data-href="/concepts/timers">
    Learn about Tokio's utilities for scheduling work based on time, such as sleeps, intervals, and timeouts.
  </x-card>
</x-cards>