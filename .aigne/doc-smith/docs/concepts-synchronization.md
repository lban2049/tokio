# Synchronization

Asynchronous programs often involve multiple tasks running independently. To coordinate these tasks and manage shared resources, Tokio provides a suite of synchronization primitives. These tools are essential for building correct and efficient concurrent applications.

Tokio's synchronization primitives can be broadly divided into two categories:

1.  **Message Passing**: Tasks communicate by sending messages to each other through channels. This approach avoids explicit shared state and is often easier to reason about.
2.  **State Synchronization**: Traditional synchronization primitives like mutexes and semaphores, adapted for the asynchronous world. These are used to control access to shared, mutable data.

All synchronization primitives in this module are runtime-agnostic and can be used with any Tokio runtime or even non-Tokio runtimes. When used within a Tokio runtime, they participate in cooperative scheduling to prevent task starvation.

## Message Passing

Message passing is a common pattern for synchronization in Tokio. It involves independent tasks sending messages through channels. Tokio provides several channel types, each suited for different communication patterns.

```d2
direction: down

Channels: {
  grid-columns: 2
  grid-gap: 50

  oneshot: "oneshot: Single Value" {
    Producer -> Consumer: "send(value)"
  }

  mpsc: "mpsc: Multi-Producer, Single-Consumer" {
    Producer-1: {label: "Producer 1"}
    Producer-2: {label: "Producer 2"}
    Producer-N: {label: "..."}

    Producer-1 -> Consumer
    Producer-2 -> Consumer
    Producer-N -> Consumer
  }

  broadcast: "broadcast: Multi-Producer, Multi-Consumer" {
    Producer-1: {label: "Producer 1"}
    Producer-2: {label: "Producer 2"}
    Consumer-1: {label: "Consumer 1"}
    Consumer-2: {label: "Consumer 2"}

    Producer-1 -> Consumer-1
    Producer-1 -> Consumer-2
    Producer-2 -> Consumer-1
    Producer-2 -> Consumer-2
  }

  watch: "watch: State Distribution" {
    Producer -> Consumer-1: {label: "notify(latest_value)"}
    Producer -> Consumer-2: {label: "notify(latest_value)"}
    Consumer-1: {label: "Consumer 1"}
    Consumer-2: {label: "Consumer 2"}
  }
}
```

### Oneshot Channel

The `oneshot` channel allows sending a single value from one producer to one consumer. It's typically used to send the result of a computation to a waiting task.

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

    // Do other work while the computation is happening in the background

    // Wait for the computation result
    let res = rx.await.unwrap();
    println!("Got = {}", res);
}
```
If a task produces a result just before terminating, you can use its `JoinHandle` directly instead of a `oneshot` channel.

### MPSC Channel

The `mpsc` (multi-producer, single-consumer) channel allows sending many values from multiple producers to a single consumer. It's often used for distributing work to a task or collecting results from many computations.

When creating an `mpsc` channel, you must specify its capacity. This capacity is the maximum number of messages that can be buffered, which is crucial for handling backpressure.

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

### Broadcast Channel

The `broadcast` channel supports a multi-producer, multi-consumer pattern where every consumer receives every value. This is useful for "fan-out" style patterns like pub/sub systems.

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

### Watch Channel

The `watch` channel is a multi-producer, multi-consumer channel that only stores the most recent value. Consumers are notified when a new value is sent, but they are not guaranteed to see every value. This is ideal for broadcasting configuration changes or signaling state transitions, such as a shutdown signal.

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
    let (tx, mut rx) = watch::channel(config.clone());

    tokio::spawn(async move {
        time::sleep(Duration::from_secs(2)).await;
        let new_config = Config { timeout: Duration::from_secs(5) };
        tx.send(new_config).unwrap();
    });

    loop {
        tokio::select! {
            _ = rx.changed() => {
                let new_timeout = rx.borrow().timeout;
                println!("Configuration changed: new timeout is {:?}", new_timeout);
                if new_timeout == Duration::from_secs(5) { break; }
            }
            _ = time::sleep(Duration::from_secs(1)) => {
                println!("Still waiting for config change...");
            }
        }
    }
}
```

## State Synchronization

For managing shared state directly, Tokio provides asynchronous versions of the synchronization primitives found in the standard library. These types wait asynchronously instead of blocking threads.

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    Provides mutual exclusion, ensuring only one task can access data at a time. It guarantees FIFO ordering for lock acquisition.
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open">
    Allows multiple readers or a single writer at any time. It is write-preferring to prevent writer starvation.
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    Limits the number of concurrent tasks that can access a resource. It holds a number of permits that tasks must acquire.
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    Enables multiple tasks to wait for each other to reach a certain point before continuing execution together.
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    A basic tool for notifying a single waiting task to resume its work, without sending any data.
  </x-card>
</x-cards>

### Mutex

An asynchronous `Mutex` provides exclusive access to data. Unlike `std::sync::Mutex`, its lock guard can be held across `.await` points. It's best suited for protecting I/O resources. For in-memory data, `std::sync::Mutex` is often preferred.

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

An `RwLock` (Reader-Writer Lock) allows for either multiple readers or one writer at a time. This can be more efficient than a `Mutex` for data that is read frequently but written to infrequently.

```rust
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
    } // read locks are dropped here

    // Only one writer lock can be held
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // write lock is dropped here
}
```

### Semaphore

A `Semaphore` is used to limit the amount of concurrent access to a resource. It maintains a set of permits; a task must acquire a permit before proceeding.

```rust
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let semaphore = Semaphore::new(3);

    let _a_permit = semaphore.acquire().await.unwrap();
    let _two_permits = semaphore.acquire_many(2).await.unwrap();

    assert_eq!(semaphore.available_permits(), 0);

    // This would wait until a permit is released
    // let _ = semaphore.acquire().await;
}
```

### Barrier

A `Barrier` allows multiple tasks to wait until all of them have reached a certain point of execution before any of them are allowed to proceed.

```rust
use tokio::sync::Barrier;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let mut handles = Vec::with_capacity(10);
    let barrier = Arc::new(Barrier::new(10));

    for i in 0..10 {
        let c = barrier.clone();
        handles.push(tokio::spawn(async move {
            println!("Task {} waiting at barrier", i);
            let wait_result = c.wait().await;
            println!("Task {} passed barrier", i);
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

---

With these tools, you can manage complex interactions between asynchronous tasks safely and efficiently. For detailed API information, please refer to the [Synchronization Primitives API Reference](./api-sync.md). Next, we will explore how Tokio handles time-based operations in the [Timers](./concepts-timers.md) section.