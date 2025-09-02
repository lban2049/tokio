# 同步原语

同步原语是协调独立异步任务行为的基本工具。在 Tokio 中，任务可以在不同线程上并发运行，这些原语可以实现共享状态的安全通信和管理。

Tokio 提供了一系列同步工具，可大致分为两类：用于无共享状态通信的消息传递通道，以及用于同步访问共享状态的原语。

### 概览

以下是可用原语及其典型用例的简要概述：

| 原语 | 用例 | 生产者 | 消费者 |
|---|---|---|---|
| `mpsc` | 从多个任务向单个任务发送多个值。 | 多个 | 单个 |
| `oneshot` | 在两个任务之间发送单个值，通常用于一次性结果。 | 单个 | 单个 |
| `broadcast` | 向多个任务广播多个值（扇出）。 | 多个 | 多个 |
| `watch` | 分发状态变更；只保留最新值。 | 多个 | 多个 |
| `Mutex` | 用于保护共享数据的异步互斥锁。 | 不适用 | 不适用 |
| `RwLock` | 允许多个读取者或单个写入者访问共享数据。 | 不适用 | 不适用 |
| `Semaphore` | 限制访问资源的并发任务数量。 | 不适用 | 不适用 |
| `Barrier` | 使多个任务能够相互等待，直到所有任务都到达某个点。 | 不适用 | 不适用 |
| `Notify` | 向单个等待中的任务发送信号的基础机制。 | 不适用 | 不适用 |

## 消息传递通道

消息传递是 Tokio 中最常见的同步形式。它通过让任务通过通道进行通信来避免共享状态的复杂性。不同的通道类型支持各种通信模式。

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

一个多生产者、单消费者通道。它用于从多个任务向单个接收任务发送多个值。这也是单生产者、单消费者场景的标准选择。

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

一个单生产者、单消费者通道，用于发送单个值。它通常用于将派生任务的计算结果发送给其创建者。

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

一个多生产者、多消费者广播通道。发送的每个值都会被每个活跃的消费者看到。这对于像发布/订阅系统这样的“扇出”模式非常理想。

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

一个多生产者、多消费者通道，只保留最近发送的值。消费者会收到变更通知，但可能会错过中间值。它非常适合分发配置或状态更新。

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

## 状态同步

对于任务需要直接管理共享状态的情况，Tokio 提供了标准库同步原语的异步版本。这些原语与 Tokio 运行时集成，会异步等待而不是阻塞线程。

### Mutex

一种异步互斥原语。它确保在任何给定时间只有一个任务可以访问其中包含的数据。锁通过异步的 `lock()` 方法获取，并且在 `.await` 点之间保持。

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

一种读写锁，允许任意数量的读取者或最多一个写入者同时存在。对于频繁读取但很少写入的数据，这比 `Mutex` 更高效。

```rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // 可以同时持有多个读锁
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    }

    // 一次只能持有一个写锁
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    }
}
```

### Semaphore

一种用于限制并发量的计数信号量。信号量持有一定数量的许可证，任务可以获取这些许可证以进入临界区。

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

使多个任务能够在同一个点上同步。栅栏用一个计数进行初始化，所有调用 `wait()` 的任务都将阻塞，直到指定数量的任务到达。

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

一种基本的任务通知原语。`Notify` 允许一个任务向另一个任务发送信号，以唤醒它并恢复处理，而无需发送任何数据。它就像一个二进制信号量。

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

## 运行时兼容性

本模块中提供的所有同步原语都与运行时无关。您可以自由地在 Tokio 运行时的不同实例之间移动它们，甚至可以在非 Tokio 运行时中使用它们。在 Tokio 运行时中使用时，它们会参与协作式调度以防止任务饿死。