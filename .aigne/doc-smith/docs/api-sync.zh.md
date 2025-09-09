# 同步原语

Tokio 提供了一套用于异步上下文的同步原语。在构建多任务应用程序时，通常需要任务之间进行通信或协调。这些原语就是用于完成这项工作的工具。

要深入了解 Tokio 中同步背后的概念，请查看[同步指南](./concepts-synchronization.md)。

Tokio 的同步工具大致可分为两类：消息传递和状态同步。

### 消息传递（通道）

消息传递是 Tokio 中最常见的同步形式。任务独立运行，并通过相互发送消息进行协调。这种方法通常是首选，因为它避免了共享状态的复杂性。

<x-cards data-columns="2">
  <x-card data-title="mpsc" data-icon="lucide:arrow-right-from-line">
    一个多生产者、单消费者通道。多个任务可以将多个值发送给一个接收任务。
  </x-card>
  <x-card data-title="oneshot" data-icon="lucide:send-to-back">
    一个单生产者、单消费者通道，用于传输单个值。非常适合将计算结果发送给等待中的任务。
  </x-card>
  <x-card data-title="broadcast" data-icon="lucide:rss">
    一个多生产者、多消费者通道，其中每条消息都会被每个消费者接收。适用于“扇出”或发布/订阅模式。
  </x-card>
  <x-card data-title="watch" data-icon="lucide:eye">
    一个多生产者、多消费者通道，仅保留最近发送的值。非常适合广播配置更改。
  </x-card>
</x-cards>

### 状态同步

这些原语是 `std::sync` 中同步原语的异步等价物。它们用于管理任务之间的共享状态。

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    提供互斥功能，确保在任何给定时间只有一个任务可以访问某些数据。
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-lock">
    允许多个并发读取者或一个独占写入者，对于读取密集型工作负载，这可能比互斥锁更高效。
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    限制可以访问资源或临界区的并发任务数量。
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    使多个任务能够相互等待，直到它们都到达程序中的某个点，然后一起继续执行。
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    一个基本工具，用于一个任务向另一个任务发送信号以唤醒并恢复处理，而不发送任何数据。
  </x-card>
</x-cards>

---

## 通道

### mpsc

一个用于在异步任务之间发送多个值的多生产者、单消费者通道。这是向任务发送工作或流式传输结果的首选通道。

要创建一个通道，请使用 `mpsc::channel` 函数，并指定通道的容量。此容量对于管理背压至关重要。

```rust MPSC Channel Example icon=logos:rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    tokio::spawn(async move {
        for i in 0..10 {
            let msg = format!("message {}", i);
            if tx.send(msg).await.is_err() {
                println!("接收者已丢弃");
                return;
            }
        }
    });

    while let Some(res) = rx.recv().await {
        println!("收到 = {}", res);
    }
}
```

### oneshot

一次性通道用于在两个异步任务之间发送单条消息。`oneshot::channel` 函数会创建一个 `Sender` 和 `Receiver` 对。它通常用于将计算结果发送给等待中的任务。

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

    // 等待计算结果
    match rx.await {
        Ok(v) => println!("收到 = {:?}", v),
        Err(_) => println!("发送者已丢弃"),
    }
}
```

### broadcast

一个多生产者、多消费者广播通道。发送的每个值都会被所有活动的消费者看到。这对于实现“扇出”模式很有用，例如在聊天系统中。

新的 `Receiver` 句柄通过调用 `Sender::subscribe()` 创建。

如果接收者速度太慢，可能会错过消息。这被称为“滞后”。`recv` 方法将返回一个 `RecvError::Lagged` 来指示消息已被丢弃。

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

一个多生产者、多消费者通道，只保留*最后*发送的值。这非常适合监视值的变化，例如配置更新。

当新值发送时，接收者会收到通知，但不能保证它们会看到每个中间值。`Receiver::changed()` 方法允许任务等待一个新的、未见过的值。

```rust Watch Channel Example icon=logos:rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = watch::channel("initial configuration");

    tokio::spawn(async move {
        // 使用循环监视变化。
        loop {
            if rx.changed().await.is_err() {
                // 发送者已丢弃
                break;
            }
            println!("配置已更改为：{}", *rx.borrow());
        }
    });

    tx.send("new configuration").unwrap();
    // 生成的任务将打印新值。
}
```

---

## 状态同步原语

### Mutex

一个异步的类 `Mutex` 类型。它提供互斥功能，确保最多只有一个任务可以同时访问某些数据。其关键特性是它的锁守卫可以跨 `.await` 点持有。

Tokio 的 `Mutex` 是公平的，按先进先出（FIFO）的原则运行。

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

一个异步读写锁。它允许任意数量的并发读取者或在任何时间点最多一个写入者。这对于频繁读取但很少写入的数据很有利。

Tokio 的 `RwLock` 偏向写入者，以防止写入者因持续的读取者流而饿死。

```rust RwLock Example icon=logos:rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // 多个读取者可以同时获取锁。
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // 读取锁在此处被丢弃。

    // 只有一个写入者可以获取锁。
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // 写入锁在此处被丢弃。
}
```

### Semaphore

一种计数信号量，用于限制并发量。它持有一组许可，任务必须获取一个许可才能进入临界区。这对于实现速率限制或限制资源使用很有用。

```rust Semaphore Example icon=logos:rust
use tokio::sync::Semaphore;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    // 将并发限制为 2 个任务。
    let semaphore = Arc::new(Semaphore::new(2));
    let mut join_handles = Vec::new();

    for i in 0..5 {
        let semaphore_clone = Arc::clone(&semaphore);
        join_handles.push(tokio::spawn(async move {
            // 当 `_permit` 离开作用域时，许可将返回给信号量。
            let _permit = semaphore_clone.acquire().await.unwrap();
            println!("任务 {} 正在运行", i);
            // 模拟工作
            tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

### Barrier

屏障使多个任务能够同步，等待所有任务都到达某个点后，才允许它们继续执行。

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
            println!("任务 {} 等待前", i);
            c.wait().await;
            println!("任务 {} 等待后", i);
        }));
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

### Notify

一个用于通知单个任务唤醒的基础原语。它不携带任何数据。可以将 `Notify` 视为一个初始许可为 0 的 `Semaphore`。`notified().await` 等待一个许可，而 `notify_one()` 使一个许可可用。

```rust Notify Example icon=logos:rust
use tokio::sync::Notify;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    let handle = tokio::spawn(async move {
        println!("任务正在等待通知...");
        notify2.notified().await;
        println!("任务收到通知");
    });

    // 模拟一些工作
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;

    println!("正在发送通知...");
    notify.notify_one();

    handle.await.unwrap();
}
```

---

## 运行时兼容性

本模块中提供的所有同步原语都与运行时无关。你可以在 Tokio 运行时的不同实例之间自由移动它们，甚至可以在非 Tokio 运行时中使用它们。

在 Tokio 运行时中使用时，它们会参与协作式调度以避免饿死。此功能在非 Tokio 运行时中使用时不适用。

但有一个例外，以 `_timeout` 结尾的方法与运行时相关，因为它们需要访问 Tokio 计时器。
