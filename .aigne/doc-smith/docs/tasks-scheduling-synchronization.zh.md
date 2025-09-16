# 同步原语

Tokio 应用程序通常被构建为一组并发运行的独立任务。为确保这些任务能够有效通信和协作，Tokio 提供了一套丰富的同步原语。这些工具对于管理共享状态和编排复杂的异步工作流至关重要。

本指南涵盖了两种主要的同步类别：

1.  **消息传递：** 使用通道在任务之间发送数据，这有助于避免共享状态的复杂性。
2.  **状态同步：** 使用像 `Mutex` 和 `RwLock` 这样的原语来控制对共享数据的访问，它们类似于标准库中的对应部分，但专为异步上下文设计。

要对 Tokio 如何管理并发操作有一个基础的了解，您可以查阅我们关于[生成和管理任务](./tasks-scheduling-spawning.md)的指南。

## 使用通道进行消息传递

消息传递是并发系统中一种常见且有效的同步模式。通过通道发送消息，任务可以独立运行，无需直接访问共享内存，从而降低了竞争条件的风险并简化了逻辑。

Tokio 提供了多种类型的通道，每种都针对不同的通信模式进行了优化。

### oneshot 通道

`oneshot` 通道设计用于从一个任务向另一个任务发送单个值。它非常适合任务需要向等待者返回单个结果的场景，例如计算的结果。

```rust Example: Using a oneshot channel icon=logos:rust
use tokio::sync::oneshot;

async fn perform_computation() -> String {
    // 模拟一些工作
    "computation result".to_string()
}

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let result = perform_computation().await;
        // 如果接收端被丢弃，发送操作可能会失败。
        let _ = tx.send(result);
    });

    // 在计算运行时执行其他工作...

    // 等待结果
    match rx.await {
        Ok(value) => println!("Got value: {}", value),
        Err(_) => println!("The sender was dropped"),
    }
}
```

请注意，如果一个任务的最终操作是产生一个结果，通常可以直接使用其 `JoinHandle` 而不是 `oneshot` 通道。等待 `JoinHandle` 会返回一个 `Result`，如果任务发生恐慌，它将是 `Err`。

### mpsc 通道

`mpsc`（多生产者，单消费者）通道允许多个任务向单个接收任务发送消息。这对于向工作任务分发工作或聚合来自多个计算的结果非常有用。

创建 `mpsc` 通道时，必须指定一个容量，即可以缓冲的最大消息数。这个容量对于管理背压至关重要；如果通道已满，发送者将异步等待，直到有可用空间。

```rust Example: Using an mpsc channel icon=logos:rust
use tokio::sync::mpsc;

async fn compute(input: u32) -> String {
    format!("Result for input {}", input)
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    // 生成一个生产者任务
    tokio::spawn(async move {
        for i in 0..10 {
            let result = compute(i).await;
            if tx.send(result).await.is_err() {
                eprintln!("receiver dropped");
                return;
            }
        }
    });

    // 当所有发送者都被丢弃后，接收者将收到 `None`。
    while let Some(message) = rx.recv().await {
        println!("Received: {}", message);
    }
}
```

### broadcast 通道

`broadcast` 通道支持多生产者、多消费者的模式。发送的每条消息都会传递给每个活跃的接收者。这通常用于“扇出”场景，如发布/订阅系统或聊天应用。

与 `mpsc` 通道类似，`broadcast` 通道也有固定的容量。如果发送者向已满的通道添加消息，最旧的消息将被丢弃以腾出空间。一个落后并错过消息的接收者将收到一个 `RecvError::Lagged` 错误。

```rust Example: Using a broadcast channel icon=logos:rust
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

### watch 通道

`watch` 通道是一种特殊的单生产者、多消费者通道，它只存储最近发送的值。当新值发送时，接收者会收到通知，但不能保证它们能看到每个中间值。

这使得它非常适合广播状态变化，例如应用程序配置的更新，因为消费者只关心最新版本。

```rust Example: Using a watch channel for configuration icon=logos:rust
use tokio::sync::watch;
use tokio::time::{self, Duration};

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config { timeout: Duration }

#[tokio::main]
async fn main() {
    let initial_config = Config { timeout: Duration::from_secs(1) };
    let (tx, mut rx) = watch::channel(initial_config.clone());

    // 生成一个任务来监控配置变化
    tokio::spawn(async move {
        // 在实际应用中，这将从文件或服务加载
        time::sleep(Duration::from_millis(50)).await;
        let new_config = Config { timeout: Duration::from_secs(5) };
        tx.send(new_config).unwrap();
    });

    println!("Initial timeout: {:?}", rx.borrow().timeout);

    // 等待配置发生变化
    if rx.changed().await.is_ok() {
        println!("New timeout: {:?}", rx.borrow().timeout);
    }
}
```

## 状态同步

对于任务需要直接共享和修改状态的情况，Tokio 提供了标准库中同步原语的异步版本。这些类型与 Tokio 运行时集成，允许任务异步等待而不是阻塞线程。

<x-cards>
  <x-card data-title="Mutex" data-icon="lucide:lock">
    互斥原语，确保在任何给定时间只有一个任务可以访问某些数据。它提供异步的 `lock` 方法。
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open-check">
    读写锁，允许任意数量的读取者或最多一个写入者。对于读取密集型工作负载，这可能比 `Mutex` 更高效。
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    计数信号量，限制可以访问资源的并发任务数量。可用于速率限制或管理资源池。
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    使多个任务能够相互等待，直到它们都到达程序中的某个点，然后一起继续执行。
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    一个基本的任务通知原语。`Notify` 可用于通知单个等待中的任务唤醒并继续处理，而无需发送任何数据。
  </x-card>
  <x-card data-title="OnceCell" data-icon="lucide:box-select">
    一个线程安全的单元格，只能写入一次，允许在首次使用时异步初始化共享数据。
  </x-card>
</x-cards>

## 后续步骤

既然您已经了解了如何同步任务，下一步是学习如何管理基于时间的操作。

<x-card data-title="时间、延迟和超时" data-href="/tasks-scheduling/time" data-icon="lucide:timer">
  学习如何使用 `sleep`、`interval` 和 `timeout` 来控制异步操作的时间。
</x-card>