# 同步

Tokio 程序通常被组织为一组并发执行的独立任务。为了管理这些任务间的通信和共享状态，Tokio 提供了一套同步原语。

这些原语可分为两大类：

1.  **消息传递**：任务通过通道相互发送消息进行通信。这种方法通常是首选，因为它避免了共享状态的复杂性。
2.  **状态同步**：当共享状态不可或缺时，Tokio 提供了标准库原语（如 `Mutex` 和 `RwLock`）的异步版本，允许任务安全地访问和修改共享数据，而不会阻塞线程。

## 使用通道进行消息传递

通道是 Tokio 中进行消息传递的主要工具。它允许一个或多个任务向一个或多个接收任务发送消息。Tokio 提供了多种类型的通道，每种都适用于不同的通信模式。

### `oneshot` 通道

`oneshot` 通道用于从单个生产者向单个消费者发送单个值。它非常适合用于将计算结果从一个派生的任务返回给其父任务。

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

    // Do other work while the computation is happening...

    // Wait for the result
    match rx.await {
        Ok(v) => println!("got = {:?}", v),
        Err(_) => println!("the sender dropped"),
    }
}
```
如果任务的最终操作是产生一个结果，通常可以直接使用其 `JoinHandle`，而无需使用 `oneshot` 通道。

### `mpsc` 通道

多生产者、单消费者（`mpsc`）通道允许多个任务向单个接收任务发送消息。这对于向工作任务分发工作或从多个来源聚合结果非常有用。

`mpsc` 通道是有界的，意味着它们有固定的容量。如果通道已满，发送方将异步等待，直到有可用空间，从而提供反压。

下面是一个使用 `mpsc` 通道结合 `oneshot` 通道来管理共享计数器的示例，它演示了请求-响应模式：

```rust
use tokio::sync::{oneshot, mpsc};

// Define the command for our counter task
enum Command {
    Increment,
}

#[tokio::main]
async fn main() {
    // Create a channel for commands. The sender sends the command and a oneshot sender
    // for the response. The receiver gets the value before the increment.
    let (cmd_tx, mut cmd_rx) = mpsc::channel::<(Command, oneshot::Sender<u64>)>(100);

    // Spawn a task to manage the counter state
    tokio::spawn(async move {
        let mut counter: u64 = 0;

        while let Some((cmd, response_tx)) = cmd_rx.recv().await {
            match cmd {
                Command::Increment => {
                    let prev = counter;
                    counter += 1;
                    response_tx.send(prev).unwrap();
                }
            }
        }
    });

    let mut join_handles = vec![];

    // Spawn 10 tasks to increment the counter
    for _ in 0..10 {
        let cmd_tx = cmd_tx.clone();

        join_handles.push(tokio::spawn(async move {
            let (resp_tx, resp_rx) = oneshot::channel();

            // Send the increment command and the response channel
            cmd_tx.send((Command::Increment, resp_tx)).await.ok().unwrap();
            
            // Wait for the response
            let res = resp_rx.await.unwrap();
            println!("previous value = {}", res);
        }));
    }

    // Wait for all tasks to complete
    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

### `broadcast` 通道

多生产者、多消费者的 `broadcast` 通道允许多个发送方向多个接收方广播消息。每个接收方都能看到每条消息。这对于聊天系统或发布/订阅模型等“扇出”模式非常有用。

如果接收方速度过慢而落后，它将收到一个 `Lagged` 错误，表明它错过了消息。然后，它可以从最旧的可用消息开始恢复接收。

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

`watch` 通道是一个单生产者、多消费者的通道，它只存储最新发送的值。当新值发送时，接收方会收到通知，但不能保证看到每一个中间值。

这使其非常适合广播状态变更（例如配置更新），因为在这种场景下，消费者只关心最新版本。

```rust
use tokio::sync::watch;
use tokio::time::{self, Duration};

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config { timeout: Duration }

#[tokio::main]
async fn main() {
    let initial_config = Config { timeout: Duration::from_secs(1) };
    let (tx, mut rx) = watch::channel(initial_config.clone());

    // Spawn a task to listen for config changes
    tokio::spawn(async move {
        loop {
            // Wait until the sender sends a new value
            if rx.changed().await.is_err() {
                // Sender was dropped
                break;
            }
            let new_config = rx.borrow().clone();
            println!("Config changed to: {:?}", new_config);
        }
    });

    // Simulate updating the config
    time::sleep(Duration::from_millis(100)).await;
    tx.send(Config { timeout: Duration::from_secs(5) }).unwrap();

    time::sleep(Duration::from_millis(100)).await;
}
```

## 状态同步

对于需要任务直接共享和修改状态的场景，Tokio 提供了标准库同步原语的异步版本。这些原语会进行异步等待，而不会阻塞线程。

<x-cards data-columns="2">
  <x-card data-title="Mutex" data-icon="lucide:lock">
    提供互斥锁，确保一次只有一个任务可以访问其中包含的数据。它保证了公平的、先进先出的访问。
  </x-card>
  <x-card data-title="RwLock" data-icon="lucide:book-open">
    一种读写锁，允许多个读者或最多一个写者同时访问。它倾向于写者，以防止写者饥饿。
  </x-card>
  <x-card data-title="Semaphore" data-icon="lucide:traffic-cone">
    限制可以访问资源的并发任务数量。任务可以获取许可，如果达到限制，则会等待。
  </x-card>
  <x-card data-title="Barrier" data-icon="lucide:git-commit-horizontal">
    使多个任务能够等待，直到所有任务都达到某个执行点，然后才能继续执行。
  </x-card>
  <x-card data-title="Notify" data-icon="lucide:bell-ring">
    一个基本原语，允许一个或多个任务等待来自另一个任务的通知，而无需发送任何数据。
  </x-card>
</x-cards>

### 何时使用 Tokio 的 `Mutex` 与 `std::sync::Mutex`

在异步代码中，使用标准库中的阻塞 `Mutex` 通常是可以接受的，甚至更可取。`tokio::sync::Mutex` 的关键特性是它能够在 `.await` 点上保持锁定状态。这使其更为复杂，且潜在成本更高。

-   **使用 `std::sync::Mutex`**：当受保护的数据不涉及 I/O，且锁的持有时间很短，不会跨越 `.await` 点时。
-   **使用 `tokio::sync::Mutex`**：当需要跨越 `.await` 点持有锁时，例如在保护对数据库连接池等 I/O 资源的共享访问时。

### 示例：使用 `RwLock`

```rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // many reader locks can be held at once
    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // read locks are dropped here

    // only one write lock may be held
    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // write lock is dropped here
}
```

## 运行时兼容性

Tokio 中的所有同步原语都与运行时无关。你可以在不同的 Tokio 运行时之间移动它们，甚至可以在非 Tokio 环境中使用它们。然而，像协作调度这样的功能只有在 Tokio 运行时内部使用时才会被激活。

---

在了解了如何管理状态和通信之后，你现在可以探索如何处理基于时间的操作。请继续阅读 [计时器](./concepts-timers.md) 部分，以了解有关休眠、间隔和超时的内容。
