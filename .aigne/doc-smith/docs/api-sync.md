# Synchronization Primitives

Tokio provides a suite of synchronization primitives for use in asynchronous contexts. When building multi-task applications, you often need ways for tasks to communicate or coordinate. These primitives are the tools for that job.

For a deeper dive into the concepts behind synchronization in Tokio, check out the [Synchronization guide](./concepts-synchronization.md).

Tokio's synchronization tools can be broadly categorized into two groups: message passing and state synchronization.

### Message Passing (Channels)

Message passing is the most common form of synchronization in Tokio. Tasks operate independently and send messages to each other to coordinate. This approach is often preferred as it avoids the complexities of shared state.

<x-cards data-columns="2">
  <x-card data-title="mpsc" data-icon="lucide:arrow-right-from-line">
    A multi-producer, single-consumer channel. Many values can be sent from many tasks to a single receiving task.
  </x-card>
  <x-card data-title="oneshot" data-icon="lucide:send-to-back">
    A single-producer, single-consumer channel that transmits a single value. Ideal for sending the result of a computation to a waiting task.
  </x-card>
  <x-card data-title="broadcast" data-icon="lucide:rss">
    A multi-producer, multi-consumer channel where every message is received by every consumer. Useful for "fan-out" or pub/sub patterns.
  </x-card>
  <x-card data-title="watch" data-icon="lucide:eye">
    A multi-producer, multi-consumer channel that only retains the most recent value sent. Perfect for broadcasting configuration changes.
  </x-card>
</x-cards>

### State Synchronization

These primitives are asynchronous equivalents of those found in `std::sync`. They are used to manage shared state between tasks.

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    Provides mutual exclusion, ensuring only one task can access some data at any given time.
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-lock">
    Allows for multiple concurrent readers or a single exclusive writer, which can be more efficient than a Mutex for read-heavy workloads.
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    Limits the number of concurrent tasks that can access a resource or critical section.
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    Enables multiple tasks to wait for each other to reach a certain point in the program before continuing together.
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    A basic tool for one task to signal another to wake up and resume processing, without sending any data.
  </x-card>
</x-cards>

---

## Channels

### mpsc

A multi-producer, single-consumer channel for sending many values between asynchronous tasks. This is the go-to channel for sending work to a task or streaming results.

To create one, use the `mpsc::channel` function, specifying the channel's capacity. This capacity is crucial for managing backpressure.

```rust MPSC Channel Example icon=logos:rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    tokio::spawn(async move {
        for i in 0..10 {
            let msg = format!("message {}", i);
            if tx.send(msg).await.is_err() {
                println!("receiver dropped");
                return;
            }
        }
    });

    while let Some(res) = rx.recv().await {
        println!("got = {}", res);
    }
}
```

### oneshot

A one-shot channel is used for sending a single message between two asynchronous tasks. The `oneshot::channel` function creates a `Sender` and `Receiver` pair. It's typically used to send the result of a computation to a waiting task.

```rust Oneshot Channel Example icon=logos:rust
use tokio::sync::oneshot;

async fn some_computation() -> String {
    "result of the computation".to_string()
}

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let res = some_computation().await;
        tx.send(res).unwrap();
    });

    // Wait for the computation result
    match rx.await {
        Ok(v) => println!("got = {:?}", v),
        Err(_) => println!("the sender dropped"),
    }
}
```

### broadcast

A multi-producer, multi-consumer broadcast channel. Each value sent is seen by all active consumers. This is useful for implementing "fan out" patterns, like in a chat system.

New `Receiver` handles are created by calling `Sender::subscribe()`.

If a receiver is too slow, it might miss messages. This is called "lagging". The `recv` method will return a `RecvError::Lagged` to indicate that messages have been dropped.

```rust Broadcast Channel Example icon=logos:rust
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

A multi-producer, multi-consumer channel that only retains the *last* sent value. This is ideal for watching for changes to a value, such as configuration updates.

Receivers are notified when a new value is sent, but there's no guarantee they will see every intermediate value. The `Receiver::changed()` method allows a task to wait for a new, unseen value.

```rust Watch Channel Example icon=logos:rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel("initial configuration");

    tokio::spawn(async move {
        // Use a loop to watch for changes.
        loop {
            if rx.changed().await.is_err() {
                // Sender was dropped
                break;
            }
            println!("Config changed to: {}", *rx.borrow());
        }
    });

    tx.send("new configuration").unwrap();
    // The spawned task will print the new value.
}
```

---

## State Synchronization Primitives

### Mutex

An asynchronous `Mutex`-like type. It provides mutual exclusion, ensuring that at most one task can access some data at a time. The key feature is that its lock guard can be held across `.await` points.

Tokio's `Mutex` is fair, operating on a FIFO basis.

```rust Mutex Example icon=logos:rust
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

An asynchronous reader-writer lock. It allows any number of concurrent readers or at most one writer at any point in time. This is beneficial for data that is read frequently but written to infrequently.

Tokio's `RwLock` is write-preferring to prevent writers from being starved by a constant stream of readers.

```rust RwLock Example icon=logos:rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // Multiple readers can acquire the lock simultaneously.
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // Read locks are dropped here.

    // Only one writer can acquire the lock.
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // Write lock is dropped here.
}
```

### Semaphore

A counting semaphore used to limit the amount of concurrency. It holds a number of permits, and tasks must acquire a permit to enter a critical section. This is useful for implementing rate limiting or bounding resource usage.

```rust Semaphore Example icon=logos:rust
use tokio::sync::Semaphore;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    // Limit concurrency to 2 tasks.
    let semaphore = Arc::new(Semaphore::new(2));
    let mut join_handles = Vec::new();

    for i in 0..5 {
        let semaphore_clone = Arc::clone(&semaphore);
        join_handles.push(tokio::spawn(async move {
            // The permit is returned to the semaphore when `_permit` goes out of scope.
            let _permit = semaphore_clone.acquire().await.unwrap();
            println!("Task {} is running", i);
            // Simulate work
            tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

### Barrier

A barrier enables multiple tasks to synchronize, waiting until all of them have reached a certain point before any of them are allowed to proceed.

```rust Barrier Example icon=logos:rust
use tokio::sync::Barrier;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let num_tasks = 10;
    let barrier = Arc::new(Barrier::new(num_tasks));
    let mut handles = Vec::with_capacity(num_tasks);

    for i in 0..num_tasks {
        let c = barrier.clone();
        handles.push(tokio::spawn(async move {
            println!("Task {} before wait", i);
            c.wait().await;
            println!("Task {} after wait", i);
        }));
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

### Notify

A basic primitive to notify a single task to wake up. It does not carry any data. A `Notify` can be thought of as a `Semaphore` starting with 0 permits. `notified().await` waits for a permit, and `notify_one()` makes a permit available.

```rust Notify Example icon=logos:rust
use tokio::sync::Notify;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    let handle = tokio::spawn(async move {
        println!("Task waiting for notification...");
        notify2.notified().await;
        println!("Task received notification");
    });

    // Simulate some work
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;

    println!("Sending notification...");
    notify.notify_one();

    handle.await.unwrap();
}
```

---

## Runtime Compatibility

All synchronization primitives provided in this module are runtime agnostic. You can freely move them between different instances of the Tokio runtime or even use them from non-Tokio runtimes.

When used in a Tokio runtime, they participate in cooperative scheduling to avoid starvation. This feature does not apply when used from non-Tokio runtimes.

As an exception, methods ending in `_timeout` are not runtime agnostic because they require access to the Tokio timer.
