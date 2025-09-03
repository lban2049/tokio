# 同步原语

在 Tokio 中，同步原语是用于协调和管理独立异步任务之间通信的重要工具。由于 Tokio 应用程序通常包含多个并发运行的任务，这些任务可能位于不同线程上，因此这些原语可以确保交互的安全性和可预测性。

该模块提供了两大类同步工具：用于避免共享状态的消息传递通道，以及标准库中对应原语的异步版本——状态同步原语。

## 消息传递

消息传递是 Tokio 中最常见的同步形式。它通过任务间经由通道发送消息的方式，帮助避免了共享状态的复杂性。Tokio 提供了多种类型的通道，每种都为不同的通信模式量身定制。

```d2
direction: down

"消息传递通道": {
  shape: package
  grid-columns: 2

  "oneshot": {
    label: "oneshot"
    tooltip: "单值，单生产者，单消费者"
    shape: class
  }

  "mpsc": {
    label: "mpsc"
    tooltip: "多值，多生产者，单消费者"
    shape: class
  }

  "broadcast": {
    label: "broadcast"
    tooltip: "多值，多生产者，多消费者（扇出）"
    shape: class
  }

  "watch": {
    label: "watch"
    tooltip: "最新值，多生产者，多消费者"
    shape: class
  }
}
```

### oneshot

`oneshot` 通道用于实现从一个生产者向一个消费者发送单个值。它通常用于将计算结果从一个衍生的任务返回给一个等待中的任务。

**示例：接收计算结果**

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

注意：如果一个任务的最终操作是生成一个结果，那么使用其 `JoinHandle` 通常是获取该值更直接的方法，无需 `oneshot` 通道。

### mpsc

`mpsc`（多生产者，单消费者）通道允许多个生产者向单个消费者发送多个值。它通常用于将工作分发给专用任务或聚合多个计算的结果。

**示例：流式传输计算结果**

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

在创建时设置的通道容量（例如 `mpsc::channel(100)`）对于管理背压至关重要。如果通道已满，发送方将异步等待，直到有可用空间。

### broadcast

`broadcast` 通道允许多个生产者向多个消费者发送消息，每个消费者都会收到每一条消息。这对于“扇出”（fan-out）模式（如发布/订阅系统）非常理想。

**示例：基本广播**

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

如果接收方速度过慢导致消息丢失，它将收到一个 `RecvError::Lagged` 错误。

### watch

`watch` 通道与广播通道类似，但有一个关键区别：它只存储最新的值。消费者会收到新值的通知，但不保证能看到每一个中间值。这使其非常适合广播状态变更，例如配置更新。

**示例：通知任务配置变更**

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

## 状态同步

这些原语是 `std::sync` 中相应原语的异步版本，旨在跨任务管理共享状态而无需阻塞线程。

| 原语 | 描述 |
| :--- | :--- |
| `Mutex` | 提供互斥，确保一次只有一个任务可以访问数据。 |
| `RwLock` | 一种读写锁，允许多个并发读取者或单个写入者。 |
| `Semaphore` | 限制可以访问资源的并发任务数量。 |
| `Barrier` | 确保多个任务在继续执行前，等待彼此到达某个特定点。 |
| `Notify` | 一种用于通知单个等待任务唤醒的基本原语。 |

### Mutex

一种用于保护共享数据的异步 `Mutex`。任务在访问数据前必须获取锁，当返回的 guard 被销毁时，锁会自动释放。Tokio 的 `Mutex` 是公平的，以先进先出（FIFO, First-In, First-Out）的顺序授予锁。

虽然这个 `Mutex` 可以在 `.await` 点之间保持锁定状态，但对于仅在简短的、非异步临界区内访问的数据，使用标准库的 `Mutex` 通常是更好的选择。

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

`RwLock` 在任意时刻提供多个读访问或单个写访问。当某个资源被频繁读取但很少写入时，它非常有用。该锁采用写优先策略，以防止写入者因读取者过多而“饿死”。

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

`Semaphore` 维护一组许可。它通过限制并发用户的数量来控制对共享资源的访问。任务必须先获取一个许可才能继续执行，当 guard 被销毁时，许可会自动归还。

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

`Barrier` 允许多个任务相互等待，直到所有任务都达到某个执行点后，才允许它们继续执行。完成后，其中一个任务会被指定为“领导者”。

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

`Notify` 实例提供了一种通知单个等待任务的方式。它的行为类似于一个初始许可为零的信号量。调用 `notify_one()` 会释放一个许可，而等待 `notified().await` 的任务将消耗该许可并被唤醒。

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

## 运行时兼容性

本模块中的所有同步原语都与运行时无关，可以与不同的 Tokio 运行时甚至非 Tokio 运行时一起使用。在 Tokio 运行时中使用时，它们会参与协作式调度，以防止任务饿死。请注意，带有 `_timeout` 后缀的方法需要访问 Tokio 计时器，因此并非与运行时无关。