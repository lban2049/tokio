# Synchronization

Asynchronous programs, by their nature, involve multiple tasks running independently. To build robust applications, these tasks often need to communicate and synchronize their actions. Tokio provides a comprehensive suite of synchronization primitives to manage shared state and coordinate work between tasks safely and efficiently.

These primitives can be broadly categorized into two groups:

1.  **Message Passing (Channels):** Tasks communicate by sending messages to each other through channels. This approach avoids shared mutable state, which can simplify concurrent code and prevent entire classes of bugs.
2.  **State Synchronization:** For situations where shared state is necessary, Tokio provides asynchronous versions of standard synchronization primitives like mutexes and semaphores. These tools control access to shared data, ensuring that only one task (or a limited number of tasks) can access it at a time.

## Message Passing with Channels

Tokio offers several types of channels, each tailored for a different communication pattern. Choosing the right channel is key to building efficient and correct applications.

<x-cards data-columns="2">
  <x-card data-title="oneshot" data-icon="lucide:arrow-right-from-line">
    A single-producer, single-consumer channel for sending a single value, typically for returning the result of a computation.
  </x-card>
  <x-card data-title="mpsc" data-icon="lucide:git-merge">
    A multi-producer, single-consumer channel for sending many values from multiple tasks to a single worker task.
  </x-card>
  <x-card data-title="broadcast" data-icon="lucide:rss">
    A multi-producer, multi-consumer channel where every message is seen by every receiver. Ideal for fan-out patterns.
  </x-card>
  <x-card data-title="watch" data-icon="lucide:eye">
    A multi-producer, multi-consumer channel that only stores the most recent value. Perfect for broadcasting configuration changes.
  </x-card>
</x-cards>

### Oneshot Channel

The `oneshot` channel is designed for sending a single value from one task to another. It's most commonly used to send the result of a computation back to a waiting task.

```rust icon=logos:rust
use tokio::sync::oneshot;

async fn some_computation() -> String {
    // ... perform some work ...
    "result of the computation".to_string()
}

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let res = some_computation().await;
        tx.send(res).unwrap();
    });

    // Do other work while the computation runs in the background

    // Wait for the result
    let result = rx.await.unwrap();
    println!("Got result: {}", result);
}
```

If a task's final action is to produce a value, you can often use its `JoinHandle` directly instead of a `oneshot` channel, which can be more efficient.

### MPSC Channel

The `mpsc` (multi-producer, single-consumer) channel allows many tasks to send messages to a single receiving task. This is a common pattern for distributing work to a dedicated worker task or for aggregating results from multiple sources.

When creating an `mpsc` channel, you must specify a capacity, which is the maximum number of messages that can be buffered. This capacity is crucial for handling backpressure: if the channel is full, senders will wait asynchronously until there is space, preventing the consumer from being overwhelmed.

```rust icon=logos:rust
use tokio::sync::mpsc;

async fn process_data(data: u32) {
    println!("Processing {}", data);
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    // Spawn a worker task to process data
    tokio::spawn(async move {
        while let Some(data) = rx.recv().await {
            process_data(data).await;
        }
    });

    // Send data from multiple producer tasks
    for i in 0..10 {
        let tx_clone = tx.clone();
        tokio::spawn(async move {
            tx_clone.send(i).await.unwrap();
        });
    }

    // Drop the original sender to allow the receiver to terminate
    drop(tx);

    // The program will exit once all data is processed.
}
```

### Broadcast Channel

A `broadcast` channel allows multiple senders to broadcast messages to multiple receivers. Every receiver sees every message sent *after* it subscribes. This is useful for pub/sub systems, live updates, or chat applications where a single event needs to be fanned out to many listeners.

If a receiver is too slow and messages build up, it will start receiving `Lagged` errors, indicating that it has missed some messages. This prevents a single slow receiver from blocking the entire system.

```rust icon=logos:rust
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

### Watch Channel

The `watch` channel is similar to a broadcast channel but with a key difference: it only retains the single most recent value. Receivers are notified when a new value is sent, but they are not guaranteed to see every intermediate value. This makes it highly efficient for distributing state updates, such as configuration changes, where only the latest version matters.

```rust icon=logos:rust
use tokio::sync::watch;
use tokio::time::{self, Duration};

#[derive(Debug, Clone, PartialEq)]
struct Config {
    api_key: String,
}

#[tokio::main]
async fn main() {
    let initial_config = Config { api_key: "initial_key".to_string() };
    let (tx, mut rx) = watch::channel(initial_config);

    // A worker task that uses the configuration
    tokio::spawn(async move {
        loop {
            // Wait for a change in configuration
            if rx.changed().await.is_err() {
                // Sender was dropped
                break;
            }
            let config = rx.borrow().clone();
            println!("Worker sees new config: {:?}", config);
        }
    });

    // Simulate updating the configuration
    time::sleep(Duration::from_millis(100)).await;
    let new_config = Config { api_key: "updated_key".to_string() };
    tx.send(new_config).unwrap();

    time::sleep(Duration::from_millis(100)).await;
}
```

## State Synchronization

When tasks need to directly access and modify shared data, message passing might not be the best fit. For these scenarios, Tokio provides asynchronous versions of traditional synchronization primitives.

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    Provides mutual exclusion, ensuring only one task can access data at a time. It's fair (FIFO).
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open-check">
    A reader-writer lock that allows multiple concurrent readers or a single exclusive writer.
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:ticket">
    Limits the number of concurrent tasks that can access a resource by managing a set of permits.
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    Allows multiple tasks to wait until all of them have reached a certain point of execution before proceeding.
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    A basic primitive to notify a single waiting task to wake up and resume its work, without sending any data.
  </x-card>
</x-cards>

### Mutex

A `tokio::sync::Mutex` provides mutually exclusive access to data. Unlike its counterpart in the standard library, locking a Tokio `Mutex` is an asynchronous operation. The lock guard can be held across `.await` points, which is essential for managing shared I/O resources like a database connection pool.

Tokio's `Mutex` is fair, meaning it grants locks in the order they are requested (First-In, First-Out).

```rust icon=logos:rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter_clone = Arc::clone(&counter);
        let handle = tokio::spawn(async move {
            let mut num = counter_clone.lock().await;
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }

    assert_eq!(*counter.lock().await, 10);
}
```

### RwLock

An `RwLock` (Reader-Writer Lock) allows for either multiple readers or a single writer to access data at any given time. This is beneficial for data that is read frequently but written to infrequently, as it allows for higher concurrency than a `Mutex`.

Tokio's `RwLock` is write-preferring to prevent writers from being starved by a continuous stream of readers.

```rust icon=logos:rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // Multiple reader locks can be held at once
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // Read locks are dropped here

    // Only one writer lock can be held
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // Write lock is dropped here
}
```

### Semaphore

A `Semaphore` maintains a count of permits. Tasks can `acquire` a permit to access a resource, and they release it when done. If no permits are available, a task will wait until one is freed. This is a powerful tool for controlling access to a limited pool of resources, such as limiting the number of concurrent network connections or file handles.

```rust icon=logos:rust
use tokio::sync::Semaphore;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    // Limit concurrency to 3 tasks
    let semaphore = Arc::new(Semaphore::new(3));
    let mut handles = vec![];

    for i in 0..5 {
        let semaphore_clone = Arc::clone(&semaphore);
        let handle = tokio::spawn(async move {
            let _permit = semaphore_clone.acquire().await.unwrap();
            println!("Task {} is running", i);
            // Simulating work
            tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

### Barrier

A `Barrier` is used to synchronize a group of tasks at a certain point in their execution. When a task reaches the barrier, it calls `.wait()` and blocks. Once the specified number of tasks have reached the barrier, all of them are unblocked and can proceed. One task is designated as the "leader" each time the barrier is passed.

```rust icon=logos:rust
use tokio::sync::Barrier;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let barrier = Arc::new(Barrier::new(10));
    let mut handles = vec![];

    for i in 0..10 {
        let barrier_clone = Arc::clone(&barrier);
        let handle = tokio::spawn(async move {
            println!("Task {} waiting at barrier...", i);
            let wait_result = barrier_clone.wait().await;
            println!("Task {} passed barrier!", i);
            if wait_result.is_leader() {
                println!("Task {} was the leader.", i);
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

With these powerful synchronization tools, you can build complex, multi-task applications in Tokio that are both safe and performant. For a deeper dive into their APIs, please see the [API Reference](./api-sync.md).

Next, let's explore how to handle time-based operations with [Timers](./concepts-timers.md).
