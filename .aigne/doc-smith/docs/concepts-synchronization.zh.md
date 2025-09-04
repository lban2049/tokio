# 同步

异步程序通常涉及多个独立运行的任务。为了协调这些任务并管理共享资源，Tokio 提供了一套同步原语。这些工具对于构建正确且高效的并发应用程序至关重要。

Tokio 的同步原语大致可分为两类：

1.  **消息传递**：任务通过通道相互发送消息进行通信。这种方法避免了显式的共享状态，并且通常更易于理解。
2.  **状态同步**：传统的同步原语（如互斥锁和信号量）被适配到了异步世界。它们用于控制对共享、可变数据的访问。

本模块中的所有同步原语都与运行时无关，可以与任何 Tokio 运行时甚至非 Tokio 运行时一起使用。在 Tokio 运行时中使用时，它们会参与协作式调度，以防止任务饥饿。

## 消息传递

消息传递是 Tokio 中一种常见的同步模式。它涉及独立任务通过通道发送消息。Tokio 提供了几种通道类型，每种类型都适用于不同的通信模式。

```d2
direction: down

Channels: {
  grid-columns: 2
  grid-gap: 50

  oneshot: "oneshot: 单一值" {
    Producer -> Consumer: "send(value)"
  }

  mpsc: "mpsc: 多生产者，单消费者" {
    Producer-1: {label: "生产者 1"}
    Producer-2: {label: "生产者 2"}
    Producer-N: {label: "..."}

    Producer-1 -> Consumer
    Producer-2 -> Consumer
    Producer-N -> Consumer
  }

  broadcast: "broadcast: 多生产者，多消费者" {
    Producer-1: {label: "生产者 1"}
    Producer-2: {label: "生产者 2"}
    Consumer-1: {label: "消费者 1"}
    Consumer-2: {label: "消费者 2"}

    Producer-1 -> Consumer-1
    Producer-1 -> Consumer-2
    Producer-2 -> Consumer-1
    Producer-2 -> Consumer-2
  }

  watch: "watch: 状态分发" {
    Producer -> Consumer-1: {label: "notify(latest_value)"}
    Producer -> Consumer-2: {label: "notify(latest_value)"}
    Consumer-1: {label: "消费者 1"}
    Consumer-2: {label: "消费者 2"}
  }
}
```

### Oneshot 通道

`oneshot` 通道允许从一个生产者向一个消费者发送单个值。它通常用于将计算结果发送给等待中的任务。

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
如果一个任务在终止前产生一个结果，你可以直接使用它的 `JoinHandle` 而不是 `oneshot` 通道。

### MPSC 通道

`mpsc`（多生产者，单消费者）通道允许多个生产者向单个消费者发送多个值。它通常用于向任务分发工作或从多个计算中收集结果。

创建 `mpsc` 通道时，必须指定其容量。该容量是可缓冲的最大消息数，这对于处理背压至关重要。

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

### Broadcast 通道

`broadcast` 通道支持多生产者、多消费者的模式，其中每个消费者都会收到每个值。这对于“扇出”式模式（如发布/订阅系统）非常有用。

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

### Watch 通道

`watch` 通道是一种多生产者、多消费者的通道，它只存储最近的一个值。当新值发送时，消费者会收到通知，但不能保证它们能看到每一个值。这非常适合广播配置更改或发出状态转换信号，例如关闭信号。

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

## 状态同步

为了直接管理共享状态，Tokio 提供了标准库中同步原语的异步版本。这些类型会异步等待，而不是阻塞线程。

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    提供互斥功能，确保一次只有一个任务可以访问数据。它保证了锁获取的先进先出（FIFO）顺序。
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open">
    在任何时候允许多个读取者或一个写入者。它偏向写入者以防止写入者饥饿。
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    限制可以访问资源的并发任务数量。它持有一组任务必须获取的许可。
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    使多个任务能够相互等待，直到它们都达到某个点后才能一起继续执行。
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    一个基础工具，用于通知单个等待中的任务恢复工作，而不发送任何数据。
  </x-card>
</x-cards>

### Mutex

异步 `Mutex` 提供对数据的独占访问。与 `std::sync::Mutex` 不同，它的锁守卫可以跨 `.await` 点持有。它最适合用于保护 I/O 资源。对于内存中的数据，通常首选 `std::sync::Mutex`。

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

`RwLock`（读写锁）允许在同一时间存在多个读取者或一个写入者。对于读取频繁但写入不频繁的数据，这可能比 `Mutex` 更高效。

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

`Semaphore` 用于限制对资源的并发访问量。它维护一组许可；任务必须先获取一个许可才能继续进行。

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

`Barrier` 允许多个任务等待，直到所有任务都达到某个执行点后，才允许它们继续进行。

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

有了这些工具，你就可以安全高效地管理异步任务之间复杂的交互。有关详细的 API 信息，请参阅[同步原语 API 参考](./api-sync.md)。接下来，我们将在[计时器](./concepts-timers.md)部分探讨 Tokio 如何处理基于时间的操作。