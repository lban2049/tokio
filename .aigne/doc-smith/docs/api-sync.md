# Synchronization Primitives

Synchronization primitives are essential tools in asynchronous programming for coordinating the activities of multiple tasks. When building applications with Tokio, you often structure your program as a set of independent tasks that communicate to get work done. The primitives in this module provide the mechanisms for safe and efficient communication and state sharing between these tasks.

These primitives can be broadly categorized into two groups:

1.  **Message Passing (Channels):** For tasks that operate independently and send messages to each other to synchronize. This approach often leads to simpler and more robust concurrent code by avoiding shared state.
2.  **State Synchronization:** For situations where tasks need to share and mutate state directly. These are asynchronous versions of primitives found in the standard library, like `Mutex` and `RwLock`.

## Message Passing: Channels

Tokio provides several channel types, each tailored for a different message-passing pattern. Choosing the right channel is key to building robust and efficient applications.

<x-cards data-columns="2">
  <x-card data-title="oneshot" data-icon="lucide:send-to-back">
    A single-use channel for sending a single value from one task to another. Ideal for returning the result of a computation.
  </x-card>
  <x-card data-title="mpsc" data-icon="lucide:arrow-right-left">
    A multi-producer, single-consumer channel. Many tasks can send messages, but only one task can receive them.
  </x-card>
  <x-card data-title="broadcast" data-icon="lucide:wifi">
    A multi-producer, multi-consumer channel where every message is received by every consumer. Useful for "fan-out" patterns.
  </x-card>
  <x-card data-title="watch" data-icon="lucide:eye">
    A single-producer, multi-consumer channel that only stores the most recent value. Consumers are notified of changes.
  </x-card>
</x-cards>

### oneshot

The `oneshot` channel is designed for sending a single value from a producer to a consumer. It's commonly used to send the result of an asynchronous computation to a waiting task.

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

    // Do other work while the computation happens...

    // Wait for the result
    match rx.await {
        Ok(v) => println!("got = {:?}", v),
        Err(_) => println!("the sender dropped"),
    }
}
```

If the `Sender` is dropped without sending a value, the `Receiver` will return an error. The `Receiver` itself is a `Future`, so you can `.await` it directly to get the result.

### mpsc

The `mpsc` (multi-producer, single-consumer) channel allows many tasks to send messages to a single receiving task. It's a workhorse for distributing tasks to a worker or collecting results from multiple computations.

This channel is bounded, meaning it has a fixed capacity. If the channel is full, senders will wait asynchronously until there is space, providing backpressure.

```rust
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

### broadcast

The `broadcast` channel allows multiple senders to broadcast messages to multiple receivers. Every receiver sees every message. This is useful for pub/sub systems, chat applications, or any scenario requiring a "fan-out" message distribution.

If a receiver is too slow and falls behind, it might miss messages. When this happens, its next call to `recv()` will return a `RecvError::Lagged` error, at which point it can decide to catch up or handle the missed data.

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

The `watch` channel is similar to `broadcast` but with a key difference: it only stores the single most recent value. Receivers are notified when a new value is sent, but they are not guaranteed to see every intermediate value.

This makes it highly efficient for broadcasting state updates, such as configuration changes, where only the latest version of the state matters.

```rust
use tokio::sync::watch;
use tokio::time::{self, Duration};

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config { timeout: Duration }

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel(Config { timeout: Duration::from_secs(1) });

    tokio::spawn(async move {
        // Periodically send config updates
        time::sleep(Duration::from_millis(50)).await;
        tx.send(Config { timeout: Duration::from_secs(5) }).unwrap();
    });

    loop {
        tokio::select! {
            _ = rx.changed() => {
                println!("Config changed to: {:?}", *rx.borrow());
            }
            _ = time::sleep(Duration::from_secs(10)) => {
                break;
            }
        }
    }
}
```

## State Synchronization

These primitives are asynchronous versions of those found in `std::sync`, designed to be used in async contexts and held across `.await` points.

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    Provides mutual exclusion, ensuring only one task can access data at a time. Locks are acquired asynchronously.
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open">
    A reader-writer lock that allows many concurrent readers or a single writer, improving concurrency for read-heavy workloads.
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    Limits the number of tasks that can access a resource or section of code concurrently.
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    Enables multiple tasks to wait for each other to reach a certain point of execution before any of them continue.
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell">
    A basic tool to signal a single waiting task to wake up and resume execution, without sending any data.
  </x-card>
</x-cards>

### Mutex

An asynchronous `Mutex` provides mutually exclusive access to data. Unlike `std::sync::Mutex`, its `lock()` method is `async` and the returned guard can be held across `.await` points.

Tokio's `Mutex` is fair, meaning it grants locks in a FIFO (First-In, First-Out) order.

**Note:** It is often preferable to use `std::sync::Mutex` if the lock is not held across an `.await` point. The standard library's mutex is faster. The primary use case for `tokio::sync::Mutex` is to protect shared access to I/O resources.

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

An `RwLock` (Reader-Writer Lock) allows for either multiple readers or one writer at any given time. This can be more efficient than a `Mutex` for data that is read frequently but written to infrequently.

Tokio's `RwLock` is write-preferring to prevent writers from being starved by a continuous stream of readers.

```rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // Many readers can acquire the lock at once.
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // read locks are dropped here

    // Only one writer can acquire the lock.
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // write lock is dropped here
}
```

### Semaphore

A `Semaphore` maintains a set of permits. It is used to control access to a shared resource that has a limited capacity, such as a connection pool or a rate limiter.

Tasks can `acquire` permits asynchronously, waiting if none are available. When a permit is dropped, it is returned to the semaphore.

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3));
    let mut join_handles = Vec::new();

    for _ in 0..5 {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        join_handles.push(tokio::spawn(async move {
            // Perform some work that is limited by the semaphore
            // ...
            // The permit is dropped when the task finishes
            drop(permit);
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

### Barrier

A `Barrier` enables multiple tasks to synchronize at a specific point. No task can proceed past the barrier until all `n` tasks have called the `wait()` method.

Barriers are reusable. Once all tasks have passed the barrier, they can be used again for another synchronization point.

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
            let wait_result = c.wait().await;
            println!("after wait");
            wait_result
        }));
    }

    let mut num_leaders = 0;
    for handle in handles {
        let wait_result = handle.await.unwrap();
        if wait_result.is_leader() {
            num_leaders += 1;
        }
    }

    assert_eq!(num_leaders, 1);
}
```

### Notify

A `Notify` is a basic tool for sending a wake-up signal to a single task. It doesn't carry any data. A task can wait for a notification by calling `notified().await`, and another task can send a notification with `notify_one()`.

If `notify_one()` is called before `notified().await`, the permit is stored, and the next call to `notified().await` will complete immediately.

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

All synchronization primitives provided in this module are runtime agnostic. You can freely move them between different instances of the Tokio runtime or even use them from non-Tokio runtimes.

When used in a Tokio runtime, these primitives participate in [cooperative scheduling](https://docs.rs/tokio/latest/tokio/task/index.html#cooperative-scheduling) to avoid starvation. This feature does not apply when used from non-Tokio runtimes.

An exception is methods ending in `_timeout`, which are not runtime agnostic because they require access to the Tokio timer.