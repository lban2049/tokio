# Synchronization Primitives

Synchronization primitives are essential tools for coordinating and managing communication between independent asynchronous tasks in Tokio. As Tokio applications often consist of multiple tasks running concurrently, potentially on different threads, these primitives ensure safe and predictable interactions.

This module provides two main categories of synchronization tools: message-passing channels for avoiding shared state, and state synchronization primitives that are asynchronous versions of their counterparts in the standard library.

## Message Passing

Message passing is the most common form of synchronization in Tokio. It involves tasks sending messages to each other through channels, which helps avoid the complexities of shared state. Tokio offers several types of channels, each tailored for different communication patterns.

```d2
direction: down

"Message Passing Channels": {
  shape: package
  grid-columns: 2

  "oneshot": {
    label: "oneshot"
    tooltip: "Single value, single producer, single consumer"
    shape: class
  }

  "mpsc": {
    label: "mpsc"
    tooltip: "Multiple values, multiple producers, single consumer"
    shape: class
  }

  "broadcast": {
    label: "broadcast"
    tooltip: "Multiple values, multiple producers, multiple consumers (fan-out)"
    shape: class
  }

  "watch": {
    label: "watch"
    tooltip: "Latest value, multiple producers, multiple consumers"
    shape: class
  }
}
```

### oneshot

The `oneshot` channel facilitates sending a single value from one producer to one consumer. It is typically used for returning the result of a computation from a spawned task to a waiting task.

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

    // Do other work while the computation happens

    // Wait for the result
    let res = rx.await.unwrap();
    println!("Got = {:?}", res);
}
```

Note: If a task's final action is producing a result, using its `JoinHandle` is often a more direct way to retrieve the value without needing a `oneshot` channel.

### mpsc

The `mpsc` (multi-producer, single-consumer) channel allows many producers to send multiple values to a single consumer. It's commonly used for distributing work to a dedicated task or aggregating results from multiple computations.

**Example: Streaming computation results**

```rust
use tokio::sync::mpsc;

async fn some_computation(input: u32) -> String {
    format!("the result of computation {}", input)
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    tokio::spawn(async move {
        for i in 0..10 {
            let res = some_computation(i).await;
            tx.send(res).await.unwrap();
        }
    });

    while let Some(res) = rx.recv().await {
        println!("got = {}", res);
    }
}
```

The channel's capacity, set during creation (e.g., `mpsc::channel(100)`), is crucial for managing backpressure. If the channel is full, senders will wait asynchronously until there is space.

### broadcast

The `broadcast` channel enables many producers to send messages to many consumers, where every consumer receives every message. This is ideal for "fan-out" patterns like pub/sub systems.

**Example: Basic broadcast**

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

If a receiver is too slow and misses messages, it will receive a `RecvError::Lagged` error.

### watch

The `watch` channel is similar to a broadcast channel but with a crucial difference: it only stores the most recent value. Consumers are notified of new values, but they are not guaranteed to see every intermediate value. This makes it perfect for broadcasting state changes, such as configuration updates.

**Example: Notifying tasks of configuration changes**

```rust
use tokio::sync::watch;
use tokio::time::{self, Duration};
use std::io;

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config {
    timeout: Duration,
}

impl Config {
    fn load() -> io::Result<Config> {
        // In a real app, this would load from a file
        Ok(Config { timeout: Duration::from_secs(1) })
    }
}

#[tokio::main]
async fn main() {
    let config = Config::load().unwrap();
    let (tx, mut rx) = watch::channel(config);

    tokio::spawn(async move {
        loop {
            time::sleep(Duration::from_secs(5)).await;
            // Periodically send a new config
            let new_config = Config::load().unwrap();
            tx.send(new_config.clone()).unwrap();
        }
    });

    // Do some work, and check for config changes
    loop {
        tokio::select! {
            _ = rx.changed() => {
                println!("Config changed to: {:?}", *rx.borrow());
            }
            _ = time::sleep(Duration::from_secs(1)) => {
                println!("Doing work...");
            }
        }
    }
}
```

## State Synchronization

These primitives are asynchronous equivalents of those found in `std::sync`, designed to manage shared state across tasks without blocking threads.

| Primitive | Description |
| :--- | :--- |
| `Mutex` | Provides mutual exclusion, ensuring only one task can access data at a time. |
| `RwLock` | A reader-writer lock that allows multiple concurrent readers or a single writer. |
| `Semaphore` | Limits the number of concurrent tasks that can access a resource. |
| `Barrier` | Ensures multiple tasks wait for each other to reach a certain point before proceeding. |
| `Notify` | A basic primitive to signal a single waiting task to wake up. |

### Mutex

An asynchronous `Mutex` that protects shared data. A task must acquire the lock before accessing the data, and the lock is released when the returned guard is dropped. Tokio's `Mutex` is fair, granting locks in a FIFO (First-In, First-Out) order.

While this `Mutex` can be held across `.await` points, it's often better to use a standard library `Mutex` for data that is only accessed in short, non-async critical sections.

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(0));

    let mut tasks = vec![];
    for _ in 0..10 {
        let data_clone = Arc::clone(&data);
        tasks.push(tokio::spawn(async move {
            let mut lock = data_clone.lock().await;
            *lock += 1;
        }));
    }

    for task in tasks {
        task.await.unwrap();
    }

    assert_eq!(*data.lock().await, 10);
}
```

### RwLock

An `RwLock` provides either multiple read accesses or a single write access at any given time. It is useful when you have a resource that is read frequently but written to infrequently. The lock has a write-preferring policy to prevent writers from being starved by readers.

```rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // Multiple readers can acquire the lock simultaneously
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // Read locks are dropped here

    // Only one writer can acquire the lock
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // Write lock is dropped here
}
```

### Semaphore

A `Semaphore` maintains a set of permits. It is used to control access to a shared resource by limiting the number of concurrent users. A task must acquire a permit before proceeding, and the permit is returned when the guard is dropped.

```rust
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let semaphore = Semaphore::new(3);

    let _a_permit = semaphore.acquire().await.unwrap();
    let _two_permits = semaphore.acquire_many(2).await.unwrap();

    assert_eq!(semaphore.available_permits(), 0);
}
```

### Barrier

A `Barrier` allows multiple tasks to wait until all of them have reached a certain point of execution before any of them are allowed to continue. One task is designated as the "leader" upon completion.

```rust
use tokio::sync::Barrier;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let mut handles = Vec::with_capacity(10);
    let barrier = Arc::new(Barrier::new(10));
    for _ in 0..10 {
        let c = barrier.clone();
        handles.push(tokio::spawn(async move {
            println!("before wait");
            c.wait().await;
            println!("after wait");
        }));
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

### Notify

A `Notify` instance provides a way to signal a single waiting task. It acts like a semaphore with zero initial permits. A call to `notify_one()` makes a permit available, and a task waiting on `notified().await` will consume the permit and wake up.

```rust
use tokio::sync::Notify;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    let handle = tokio::spawn(async move {
        notify2.notified().await;
        println!("received notification");
    });

    println!("sending notification");
    notify.notify_one();

    handle.await.unwrap();
}
```

## Runtime Compatibility

All synchronization primitives in this module are runtime-agnostic and can be used with different Tokio runtimes or even non-Tokio runtimes. When used within a Tokio runtime, they participate in cooperative scheduling to prevent task starvation. Note that methods with a `_timeout` suffix require access to the Tokio timer and are not runtime-agnostic.