# 任务与调度

任何 Tokio 应用的核心都是异步任务。任务是一种轻量级、非阻塞的执行单元，类似于 Go 的 goroutine 或 Erlang 的进程。与传统线程由操作系统管理不同，任务由 Tokio 运行时管理，这使得它们的创建和管理成本极低。

本节涵盖了在 Tokio 中使用任务的基本概念，从创建任务到管理其生命周期以及处理不同类型的工作负载。

## 什么是任务？

Tokio 任务的主要特点：

*   **轻量级：** 与操作系统线程相比，创建、切换和销毁任务的开销非常低，因为它不需要系统上下文切换。
*   **协作式调度：** 任务会一直运行，直到它们将控制权交还给调度器，这通常发生在 `.await` 点。这与操作系统线程中的抢占式多任务处理不同，在抢占式多任务处理中，操作系统可以随时中断线程。
*   **非阻塞：** 任务不应执行阻塞操作，例如传统的 文件 I/O 或繁重的、长时间运行的计算。这样做会阻止同一线程上的其他任务取得进展。Tokio 提供了处理此类情况的特定工具。

## 生成任务

启动并发操作最常见的方法是使用 `tokio::spawn`。该函数接受一个异步块（一个 `Future`），并立即在 Tokio 运行时上开始执行它，同时返回一个 `JoinHandle`。

```rust Spawning a Task icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        // 这在一个新的并发任务中运行。
        "hello world!"
    });

    // 原始任务继续独立运行。
    println!("Spawned a task.");

    // 我们可以等待生成的任务完成。
    let result = handle.await.unwrap();
    assert_eq!(result, "hello world!");
}
```

`JoinHandle` 是一个 future，它会解析为所生成任务的输出。等待该句柄可以让你取回结果。如果生成的任务发生 panic，等待其 `JoinHandle` 将返回一个错误。

## 任务取消

任务可以被取消，这会向它们发出信号，在下一个可用的 `.await` 点停止执行。这是一种平滑的关闭机制。取消操作通常通过任务的 `JoinHandle` 上的 `abort` 方法来完成。

```rust Cancelling a Task icon=logos:rust
use tokio::time::{self, Duration};

#[tokio::main]
async fn main() {
    let task = tokio::spawn(async {
        // 这个任务将运行一段时间...
        time::sleep(Duration::from_secs(10)).await;
        println!("Task finished normally.");
    });

    // 让它运行片刻。
    time::sleep(Duration::from_millis(100)).await;

    // 现在，中止该任务。
    task.abort();

    // 等待一个被取消的任务会导致错误。
    let result = task.await;
    assert!(result.is_err());
    println!("Task was aborted.");
}
```

当任务被中止时，它会在其被挂起的 `.await` 点停止，并且其局部变量会被丢弃。需要注意的是，使用 `spawn_blocking` 生成的任务无法被中止，因为它们不是异步的，也没有 `.await` 点。

## 处理阻塞代码

因为任务不能阻塞它们正在运行的线程，所以 Tokio 提供了特定的函数来处理同步、阻塞或 CPU 密集型代码。

<x-cards data-columns="2">
  <x-card data-title="spawn_blocking" data-icon="lucide:cpu">
    在专用于阻塞任务的线程池上运行阻塞函数，而不会干扰异步运行时。它返回一个 `JoinHandle` 以等待结果。
  </x-card>
  <x-card data-title="block_in_place" data-icon="lucide:pause-circle">
    将当前工作线程转换为阻塞线程，并将其他异步任务移动到不同的工作线程。这通过避免上下文切换可以更高效，但仅在多线程运行时中可用。
  </x-card>
</x-cards>

### 使用 `spawn_blocking`

这是运行阻塞代码的推荐方法。它将工作卸载到一个单独的线程池，从而保持异步核心线程的空闲。

```rust Using spawn_blocking icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let handle = task::spawn_blocking(|| {
        // 这是计算密集型或阻塞 I/O 操作的替代。
        std::thread::sleep(std::time::Duration::from_millis(500));
        "done"
    });

    let result = handle.await?;
    assert_eq!(result, "done");
    Ok(())
}
```

### 使用 `block_in_place`

当您需要在多线程运行时中从异步任务内部执行较短的阻塞操作时，`block_in_place` 可能是一个不错的选择。

```rust Using block_in_place icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() {
    let result = task::block_in_place(|| {
        // 这在当前工作线程上运行，该线程被临时
        // 标记为阻塞线程。
        std::thread::sleep(std::time::Duration::from_millis(500));
        "done"
    });

    assert_eq!(result, "done");
}
```

## 让步

你可以通过调用 `tokio::task::yield_now()` 主动将控制权交还给 Tokio 调度器。这允许调度器在恢复当前任务之前运行其他待处理的任务。这对于确保长时间运行的任务不独占 CPU 时间非常有用。

```rust Yielding a Task icon=logos:rust
use tokio::task;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        println!("spawned task done!");
    });

    println!("main task yielding.");
    // 让步，允许新生成的任务可能首先执行。
    task::yield_now().await;
    println!("main task done!");
}
```

## 使用 `JoinSet` 管理多个任务

当你需要生成和管理动态数量的任务时，使用 `JoinSet` 比在 `Vec` 中收集 `JoinHandle` 更方便。`JoinSet` 允许你按照任务完成的顺序（而非生成的顺序）等待它们。

当 `JoinSet` 被丢弃时，集合中所有剩余的任务都会被自动中止。

```rust Managing Tasks with JoinSet icon=logos:rust
use tokio::task::JoinSet;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..5 {
        set.spawn(async move {
            sleep(Duration::from_millis((5 - i) * 100)).await;
            i
        });
    }

    while let Some(res) = set.join_next().await {
        let completed_task_index = res.unwrap();
        println!("Task {} completed.", completed_task_index);
    }
    
    println!("All tasks finished.");
}
```

这个例子演示了具有不同睡眠时间的任务是如何乱序完成的，以及 `join_next` 如何在它们可用时高效地检索它们的结果。

---

现在你已经了解了如何使用任务创建和管理并发操作，可以继续探索这些任务如何执行有用的工作。请继续阅读 [异步 I/O](./concepts-io.md) 以了解网络和文件操作，或阅读 [同步](./concepts-synchronization.md) 以了解任务如何安全地通信和共享数据。