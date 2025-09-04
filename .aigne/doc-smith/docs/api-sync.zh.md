# 同步原语

同步原语是异步编程中协调多个任务活动的基本工具。使用 Tokio 构建应用程序时，通常会将程序构建为一组通过通信来完成工作的独立任务。此模块中的原语为这些任务之间提供了安全高效的通信和状态共享机制。

这些原语大致可分为两类：

1.  **消息传递 (Channels):** 用于独立运行并通过相互发送消息进行同步的任务。这种方法通过避免共享状态，通常能编写出更简单、更健壮的并发代码。
2.  **状态同步:** 用于任务需要直接共享和修改状态的场景。这些是标准库中原语的异步版本，例如 `Mutex` 和 `RwLock`。

## 消息传递：通道

Tokio 提供了多种通道类型，每种都为不同的消息传递模式量身定制。选择正确的通道是构建健壮高效应用程序的关键。

<x-cards data-columns="2">
  <x-card data-title="oneshot" data-icon="lucide:send-to-back">
    一种一次性通道，用于将单个值从一个任务发送到另一个任务。非常适合返回计算结果。
  </x-card>
  <x-card data-title="mpsc" data-icon="lucide:arrow-right-left">
    一个多生产者、单消费者通道。许多任务可以发送消息，但只有一个任务可以接收它们。
  </x-card>
  <x-card data-title="broadcast" data-icon="lucide:wifi">
    一个多生产者、多消费者通道，其中每条消息都会被每个消费者接收。适用于“扇出”模式。
  </x-card>
  <x-card data-title="watch" data-icon="lucide:eye">
    一个单生产者、多消费者通道，只存储最新的值。消费者会收到变更通知。
  </x-card>
</x-cards>

### oneshot

`oneshot` 通道设计用于将单个值从生产者发送到消费者。它通常用于将异步计算的结果发送给等待中的任务。

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

如果 `Sender` 在未发送值的情况下被丢弃，`Receiver` 将返回一个错误。`Receiver` 本身是一个 `Future`，因此可以直接对其进行 `.await` 以获取结果。

### mpsc

`mpsc` (多生产者，单消费者) 通道允许多个任务向单个接收任务发送消息。它是向工作线程分发任务或从多个计算中收集结果的主力。

该通道是有界的，意味着它有固定的容量。如果通道已满，发送者将异步等待直到有空间可用，从而提供背压。

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

`broadcast` 通道允许多个发送者向多个接收者广播消息。每个接收者都能看到每条消息。这对于发布/订阅系统、聊天应用程序或任何需要“扇出”消息分发的场景都很有用。

如果接收者速度太慢而落后，它可能会错过消息。发生这种情况时，其下一次调用 `recv()` 将返回一个 `RecvError::Lagged` 错误，此时它可以决定是追赶还是处理错过的消息。

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

`watch` 通道与 `broadcast` 类似，但有一个关键区别：它只存储最新的单个值。当新值发送时，接收者会收到通知，但不能保证看到每个中间值。

这使其在广播状态更新（例如配置更改）时非常高效，因为在这种情况下，只有最新版本的状态才重要。

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

## 状态同步

这些原语是 `std::sync` 中原语的异步版本，设计用于异步上下文中，并可在 `.await` 点之间持有。

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    提供互斥功能，确保一次只有一个任务可以访问数据。锁是异步获取的。
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open">
    一种读写锁，允许多个并发读取者或一个写入者，从而提高读取密集型工作负载的并发性。
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    限制可并发访问资源或代码段的任务数量。
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    使多个任务能够相互等待，直到所有任务都达到某个执行点后，才能继续执行。
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell">
    一个基本工具，用于向单个等待中的任务发送唤醒信号以恢复执行，不发送任何数据。
  </x-card>
</x-cards>

### Mutex

异步 `Mutex` 提供对数据的互斥访问。与 `std::sync::Mutex` 不同，它的 `lock()` 方法是 `async` 的，并且返回的 guard 可以在 `.await` 点之间持有。

Tokio 的 `Mutex` 是公平的，意味着它以先进先出 (FIFO) 的顺序授予锁。

**注意：** 如果锁不会跨越 `.await` 点持有，通常最好使用 `std::sync::Mutex`。标准库的 mutex 速度更快。`tokio::sync::Mutex` 的主要用例是保护对 I/O 资源的共享访问。

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

`RwLock` (读写锁) 允许在任何给定时间存在多个读取者或一个写入者。对于读取频繁但写入不频繁的数据，这可能比 `Mutex` 更高效。

Tokio 的 `RwLock` 是写优先的，以防止写入者因连续的读取者流而饿死。

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

`Semaphore` 维护一组许可。它用于控制对容量有限的共享资源的访问，例如连接池或速率限制器。

任务可以异步地 `acquire` 许可，如果没有可用许可则会等待。当许可被丢弃时，它会返回到信号量中。

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

`Barrier` 使多个任务能够在特定点同步。在所有 `n` 个任务都调用 `wait()` 方法之前，任何任务都不能越过屏障。

屏障是可重用的。一旦所有任务都通过了屏障，它们就可以再次用于另一个同步点。

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

`Notify` 是一个向单个任务发送唤醒信号的基本工具。它不携带任何数据。一个任务可以通过调用 `notified().await` 来等待通知，另一个任务可以通过 `notify_one()` 发送通知。

如果在 `notified().await` 之前调用 `notify_one()`，许可会被存储，下一次调用 `notified().await` 将立即完成。

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

此模块中提供的所有同步原语都与运行时无关。你可以自由地在 Tokio 运行时的不同实例之间移动它们，甚至可以在非 Tokio 运行时中使用它们。

在 Tokio 运行时中使用时，这些原语会参与[协作调度](https://docs.rs/tokio/latest/tokio/task/index.html#cooperative-scheduling)以避免饥饿。此功能在非 Tokio 运行时中使用时不适用。

一个例外是以后缀 `_timeout` 结尾的方法，它们不是运行时无关的，因为它们需要访问 Tokio 计时器。
