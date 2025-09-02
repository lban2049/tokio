# Synchronization Primitives

Synchronization primitives are essential tools for coordinating the behavior of independent asynchronous tasks. In Tokio, where tasks can run concurrently on different threads, these primitives allow for safe communication and management of shared state.

Tokio provides a range of synchronization tools that can be broadly categorized into two groups: message passing channels for communication without shared state, and primitives for synchronizing access to shared state.

### At a Glance

Here's a quick overview of the available primitives and their typical use cases:

| Primitive | Use Case | Producers | Consumers |
|---|---|---|---|
| `mpsc` | Sending many values from multiple tasks to a single task. | Many | Single |
| `oneshot` | Sending a single value between two tasks, often for a one-time result. | Single | Single |
| `broadcast` | Broadcasting many values to many tasks (fan-out). | Many | Many |
| `watch` | Distributing state changes; only the latest value is kept. | Many | Many |
| `Mutex` | Asynchronous mutual exclusion for protecting shared data. | N/A | N/A |
| `RwLock` | Allows multiple readers or a single writer to access shared data. | N/A | N/A |
| `Semaphore` | Limits the number of concurrent tasks accessing a resource. | N/A | N/A |
| `Barrier` | Enables multiple tasks to wait for each other to reach a certain point. | N/A | N/A |
| `Notify` | A basic mechanism to signal a single waiting task. | N/A | N/A |

## Message Passing Channels

Message passing is the most common form of synchronization in Tokio. It avoids the complexities of shared state by having tasks communicate through channels. Different channel types support various communication patterns.

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


subgraph_channels: "Channels" {
  "oneshot\n(single value)": channel_oneshot
  "mpsc\n(many values)": channel_mpsc
  "broadcast\n(fan-out)": channel_broadcast
  "watch\n(latest value)": channel_watch
}

Producer -> channel_oneshot -> Consumer
Producers -> channel_mpsc -> Consumer
Producers -> channel_broadcast -> Consumers
Producers -> channel_watch -> Consumers
```

### mpsc

A multi-producer, single-consumer channel. It's used for sending many values from multiple tasks to a single receiving task. This is also the standard choice for single-producer, single-consumer scenarios.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
        }
    });

    while let Some(i) = rx.recv().await {
        println!("got = {}", i);
    }
}
```

### oneshot

A single-producer, single-consumer channel for sending a single value. It's typically used to send the result of a computation from a spawned task to its creator.

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        if let Err(_) = tx.send(3) {
            println!("the receiver dropped");
        }
    });

    match rx.await {
        Ok(v) => println!("got = {:?}", v),
        Err(_) => println!("the sender dropped"),
    }
}
```

### broadcast

A multi-producer, multi-consumer broadcast channel. Each value sent is seen by every active consumer. This is ideal for "fan-out" patterns like pub/sub systems.

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

### watch

A multi-producer, multi-consumer channel that only retains the most recently sent value. Consumers are notified of changes, but may miss intermediate values. It's perfect for distributing configuration or state updates.

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel("hello");

    tokio::spawn(async move {
        loop {
            if rx.changed().await.is_err() {
                break;
            }
            println!("new value = {}", *rx.borrow());
        }
    });

    tx.send("world").unwrap();
}
```

## State Synchronization

For situations where tasks need to directly manage shared state, Tokio provides asynchronous versions of the standard library's synchronization primitives. These primitives integrate with the Tokio runtime and will wait asynchronously instead of blocking a thread.

### Mutex

An asynchronous mutual exclusion primitive. It ensures that only one task can access the contained data at any given time. Locks are acquired via the async `lock()` method and are held across `.await` points.

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(0));

    let mut handles = vec![];
    for _ in 0..10 {
        let data_clone = Arc::clone(&data);
        handles.push(tokio::spawn(async move {
            let mut lock = data_clone.lock().await;
            *lock += 1;
        }));
    }

    for handle in handles {
        handle.await.unwrap();
    }

    assert_eq!(*data.lock().await, 10);
}
```

### RwLock

A reader-writer lock that allows any number of readers or at most one writer at a time. This can be more efficient than a `Mutex` for data that is read frequently but written to infrequently.

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
    }

    // only one write lock may be held
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    }
}
```

### Semaphore

A counting semaphore used to limit the amount of concurrency. A semaphore holds a number of permits, which tasks can acquire to enter a critical section.

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

Enables multiple tasks to synchronize at a common point. A barrier is initialized with a count, and all tasks calling `wait()` will block until the specified number of tasks have arrived.

```rust
use tokio::sync::Barrier;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let barrier = Arc::new(Barrier::new(10));
    let mut handles = Vec::with_capacity(10);

    for _ in 0..10 {
        let c = barrier.clone();
        handles.push(tokio::spawn(async move {
            c.wait().await;
        }));
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

### Notify

A basic task notification primitive. `Notify` allows one task to signal another to wake up and resume processing, without sending any data. It's like a binary semaphore.

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

    notify.notify_one();
    handle.await.unwrap();
}
```

## Runtime Compatibility

All synchronization primitives provided in this module are runtime agnostic. You can freely move them between different instances of the Tokio runtime or even use them from non-Tokio runtimes. When used within a Tokio runtime, they participate in cooperative scheduling to prevent task starvation.