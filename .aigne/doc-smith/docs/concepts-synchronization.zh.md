# 同步

异步程序本质上涉及多个独立运行的任务。为了构建健壮的应用程序，这些任务通常需要通信并同步它们的行为。Tokio 提供了一套全面的同步原语，用于安全高效地管理共享状态和协调任务间的工作。

这些原语大致可分为两类：

1.  **消息传递（通道）：** 任务通过通道相互发送消息进行通信。这种方法避免了共享可变状态，从而可以简化并发代码并防止整类错误的发生。
2.  **状态同步：** 对于需要共享状态的情况，Tokio 提供了标准同步原语（如互斥锁和信号量）的异步版本。这些工具控制对共享数据的访问，确保同一时间只有一个任务（或有限数量的任务）可以访问它。

## 使用通道进行消息传递

Tokio 提供了多种类型的通道，每种都为不同的通信模式量身定制。选择合适的通道是构建高效且正确的应用程序的关键。

<x-cards data-columns="2">
  <x-card data-title="oneshot" data-icon="lucide:arrow-right-from-line">
    一个单生产者、单消费者的通道，用于发送单个值，通常用于返回计算结果。
  </x-card>
  <x-card data-title="mpsc" data-icon="lucide:git-merge">
    一个多生产者、单消费者的通道，用于从多个任务向单个工作任务发送多个值。
  </x-card>
  <x-card data-title="broadcast" data-icon="lucide:rss">
    一个多生产者、多消费者的通道，其中每条消息都会被每个接收者看到。非常适合扇出模式。
  </x-card>
  <x-card data-title="watch" data-icon="lucide:eye">
    一个多生产者、多消费者的通道，只存储最新的值。非常适合广播配置变更。
  </x-card>
</x-cards>

### Oneshot 通道

`oneshot` 通道专为从一个任务向另一个任务发送单个值而设计。它最常用于将计算结果发送回等待中的任务。

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

如果一个任务的最终操作是产生一个值，通常可以直接使用其 `JoinHandle` 而不是 `oneshot` 通道，这样效率更高。

### MPSC 通道

`mpsc`（多生产者，单消费者）通道允许多个任务向单个接收任务发送消息。这是将工作分发给专用工作任务或聚合来自多个来源结果的常见模式。

创建 `mpsc` 通道时，必须指定一个容量，即可以缓冲的最大消息数。这个容量对于处理背压至关重要：如果通道已满，发送者将异步等待直到有空间，从而防止消费者不堪重负。

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

### Broadcast 通道

`broadcast` 通道允许多个发送者向多个接收者广播消息。每个接收者都能看到在其订阅*之后*发送的每条消息。这对于发布/订阅系统、实时更新或聊天应用等需要将单个事件扇出给多个监听者的场景非常有用。

如果一个接收者速度太慢导致消息堆积，它将开始收到 `Lagged` 错误，表明它错过了一些消息。这可以防止单个慢速接收者阻塞整个系统。

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

### Watch 通道

`watch` 通道与 broadcast 通道类似，但有一个关键区别：它只保留最新的单个值。当新值发送时，接收者会收到通知，但不保证能看到每个中间值。这使得它在分发状态更新（例如配置变更）时非常高效，因为在这些场景中，只有最新版本才重要。

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

## 状态同步

当任务需要直接访问和修改共享数据时，消息传递可能不是最佳选择。对于这些场景，Tokio 提供了传统同步原语的异步版本。

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    提供互斥访问，确保一次只有一个任务可以访问数据。它是公平的（FIFO）。
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open-check">
    一种读写锁，允许多个并发读取者或单个独占写入者。
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:ticket">
    通过管理一组许可来限制可以访问资源的并发任务数量。
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    允许多个任务等待，直到所有任务都达到某个执行点后再继续执行。
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    一个基础原语，用于通知单个等待中的任务唤醒并继续其工作，不发送任何数据。
  </x-card>
</x-cards>

### Mutex

`tokio::sync::Mutex` 提供对数据的互斥访问。与标准库中的对应物不同，锁定 Tokio `Mutex` 是一个异步操作。锁守卫可以在 `.await` 点之间持有，这对于管理共享的 I/O 资源（如数据库连接池）至关重要。

Tokio 的 `Mutex` 是公平的，意味着它会按照请求的顺序（先进先出）授予锁。

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

`RwLock`（读写锁）允许在任何给定时间有多个读取者或单个写入者访问数据。这对于读取频繁但写入不频繁的数据很有利，因为它比 `Mutex` 允许更高的并发性。

Tokio 的 `RwLock` 是写优先的，以防止写入者因连续的读取者流而饿死。

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

`Semaphore` 维护一个许可计数。任务可以 `acquire` 一个许可来访问资源，并在完成后释放它。如果没有可用的许可，任务将等待直到有许可被释放。这是一个强大的工具，用于控制对有限资源池的访问，例如限制并发网络连接或文件句柄的数量。

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

`Barrier` 用于在一组任务执行到某个特定点时进行同步。当一个任务到达屏障时，它会调用 `.wait()` 并阻塞。一旦指定数量的任务都到达了屏障，它们将全部被解除阻塞并继续执行。每次通过屏障时，会有一个任务被指定为“领导者”。

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

借助这些强大的同步工具，您可以在 Tokio 中构建复杂、安全且高性能的多任务应用程序。要深入了解它们的 API，请参阅 [API 参考](./api-sync.md)。

接下来，让我们探讨如何使用 [计时器](./concepts-timers.md) 处理基于时间的操作。
