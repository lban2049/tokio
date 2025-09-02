# 同步

Tokio 程序通常被构建为一组独立的任务，这些任务可能在不同的物理线程上执行。Tokio 提供的同步原语允许这些独立任务安全地进行通信和协调。

同步原语主要有两大类：

1.  **消息传递**：任务独立运行，通过通道相互发送消息以进行同步。这种方法通常是首选，因为它避免了共享状态的复杂性。
2.  **状态同步**：像互斥锁和信号量这样的原语，用于控制对共享状态的访问。它们是标准库中原语的异步等价物。

## 使用通道进行消息传递

消息传递是 Tokio 中处理同步的一种常见且有效的方式。它通过通道实现，允许一个或多个生产者任务向一个或多个消费者任务发送消息。Tokio 提供了几种类型的通道，每种都适用于不同的通信模式。

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

subgraph "oneshot": {
  label: "Single value, single use"
  Producer -> Consumer
}

subgraph "mpsc": {
  label: "Multiple values, single consumer"
  Producers -> Consumer
}

subgraph "broadcast": {
  label: "Multiple values, multiple consumers (fan-out)"
  Producers -> Consumers
}

subgraph "watch": {
  label: "Latest value, multiple consumers"
  Producers -> Consumers
}
```

### `oneshot` 通道

`oneshot` 通道设计用于从一个生产者向一个消费者发送**单个**值。它通常用于将一个派生任务的计算结果返回给一个等待任务。

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

    // Do other work while the computation is happening

    // Wait for the computation result
    let res = rx.await.unwrap();
    println!("Got result: {}", res);
}
```
注意：如果任务的最终操作是返回结果，通常可以直接使用任务的 `JoinHandle`，而无需使用 `oneshot` 通道。

### `mpsc` 通道

`mpsc`（多生产者，单消费者）通道允许从**多个**生产者向**单个**消费者发送**多个**值。它非常适合将工作分发给一个工作任务或从多个来源聚合结果。

该通道具有固定容量，可提供背压。如果通道已满，发送者将异步等待，直到有空间容纳新消息。

**示例：流式传输计算结果**

```rust
use tokio::sync::mpsc;

async fn perform_computation(input: u32) -> String {
    format!("the result of computation {}", input)
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    tokio::spawn(async move {
        for i in 0..10 {
            let res = perform_computation(i).await;
            tx.send(res).await.unwrap();
        }
    });

    while let Some(res) = rx.recv().await {
        println!("got = {}", res);
    }
}
```

### `broadcast` 通道

`broadcast` 通道能够从**多个**生产者向**多个**消费者发送**多个**值。每个消费者都会收到发送的**每一个**值。这对于“扇出”模式（如发布/订阅系统或聊天应用）非常有用。

如果接收者速度太慢而落后，它将收到一个 `Lagged` 错误，其内部游标将更新为通道中仍然存在的最旧消息。

**示例：广播的基本用法**

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

### `watch` 通道

`watch` 通道是一个多生产者、多消费者的通道，它只存储**最新**的值。当新值发送时，消费者会收到通知，但它们不保证能看到每一个中间值。它特别适用于广播配置更改或发出状态更新信号，如关闭信号。

**示例：通知任务配置更改**

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
    let (tx, mut rx) = watch::channel(config);

    tokio::spawn(async move {
        // 在实际应用中，你可能会监控一个文件或其他来源。
        time::sleep(Duration::from_millis(500)).await;
        let new_config = Config { timeout: Duration::from_secs(2) };
        tx.send(new_config).unwrap();
    });

    loop {
        tokio::select! {
            _ = rx.changed() => {
                let new_timeout = rx.borrow().timeout;
                println!("Configuration changed: timeout is now {:?}", new_timeout);
                if new_timeout.as_secs() == 2 {
                    break;
                }
            }
        }
    }
}
```

## 状态同步

为了管理对共享数据的直接访问，Tokio 提供了标准库同步原语的异步版本。这些类型会异步等待，而不会阻塞线程。

| 原语 | 描述 |
|---|---|
| [`Mutex`](./api-sync.md) | 一种互斥锁，确保一次只有一个任务可以访问数据。它保证了等待锁的任务的 FIFO 顺序。 |
| [`RwLock`](./api-sync.md) | 一种读写锁，允许在任何给定时间有多个读取者或单个写入者。它采用写者优先策略以防止写者饥饿。 |
| [`Semaphore`](./api-sync.md) | 通过维护一组许可证来限制对资源的并发访问。任务必须在获取许可证后才能继续执行。 |
| [`Barrier`](./api-sync.md) | 使多个任务能够相互等待，直到它们都到达程序中的某个点，然后再一起继续执行。 |
| [`Notify`](./api-sync.md) | 一种基本原语，用于通知单个等待中的任务恢复工作，而不发送任何数据。 |

## 后续步骤

选择正确的同步原语对于编写正确且高性能的异步代码至关重要。有关详细用法和 API 信息，请参阅 API 参考。

<x-cards>
  <x-card data-title="API 参考：同步" data-icon="lucide:book-open" data-href="/api/sync">
    深入了解 Tokio 所有同步原语的详细 API 文档。
  </x-card>
  <x-card data-title="下一个概念：计时器" data-icon="lucide:arrow-right" data-href="/concepts/timers">
    了解 Tokio 用于基于时间调度工作的实用工具，例如睡眠、间隔和超时。
  </x-card>
</x-cards>