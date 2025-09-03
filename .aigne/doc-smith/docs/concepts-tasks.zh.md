# 任务与调度

Tokio 中的异步程序是围绕称为任务的轻量级、非阻塞执行单元构建的。本指南涵盖了如何创建、管理和协调这些任务以构建并发应用程序的核心概念。

## 什么是任务？

任务类似于操作系统线程，但它由 Tokio 运行时而不是操作系统调度器管理。这使得它们明显更轻量。你可以将它们视为类似于 Go 的 goroutine 或 Erlang 的进程。

Tokio 任务的主要特点包括：

*   **轻量级**：与操作系统线程相比，创建、切换和销毁任务的开销非常低。一个 Tokio 应用程序管理成千上万甚至数百万个任务是很常见的。
*   **协作式调度**：任务会一直运行，直到它们在 `.await` 点将控制权交还给调度器。然后调度器会运行另一个任务。这与操作系统线程使用的抢占式多任务处理形成对比，在抢占式多任务处理中，操作系统可以随时中断线程。
*   **非阻塞**：任务不应执行阻塞线程的操作，例如同步 I/O 或繁重的、长时间运行的计算。这样做会阻止同一线程上的其他任务取得进展。对于这种情况，Tokio 提供了特定的 API。

让我们来探讨如何在实践中使用任务。

```d2
direction: down

"Tokio 运行时": {
  shape: package
  grid-columns: 2
  grid-gap: 80

  "核心工作线程": {
    label: "核心工作线程（用于异步任务）"
    shape: package
    grid-columns: 2

    "工作线程 1": {
      shape: rectangle
      "任务 A (运行)" -> "在 .await 处让出" -> "任务 B (运行)" -> "在 .await 处让出" -> "任务 A (恢复)"
    }

    "工作线程 2": {
      shape: rectangle
      "任务 C (运行)" -> "在 .await 处让出" -> "任务 D (运行)"
    }
  }

  "阻塞线程池": {
    label: "阻塞线程池（用于同步代码）"
    shape: package
    grid-columns: 2

    "阻塞线程 1": {
      shape: rectangle
      "阻塞操作 1"
    }
    "阻塞线程 2": {
      shape: rectangle
      "阻塞操作 2"
    }
  }
}

"应用程序代码": {
  shape: rectangle
  grid-columns: 1
  "spawn(async_fn)": {
    label: "tokio::spawn(async { ... })"
  }
  "spawn_blocking(sync_fn)": {
    label: "tokio::spawn_blocking(|| { ... })"
  }
}

"应用程序代码"."spawn(async_fn)" -> "Tokio 运行时"."核心工作线程": "在工作线程上调度"
"应用程序代码"."spawn_blocking(sync_fn)" -> "Tokio 运行时"."阻塞线程池": "在专用线程上运行"

```

## 生成任务

创建任务最常见的方法是使用 `tokio::spawn` 函数。它接受一个异步块或 future，并立即调度它在运行时上运行。它返回一个 `JoinHandle`，你可以用它与生成的任务进行交互。

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    let s = format!("Background task {} complete.", id);
    println!("{}", s);
    s
}

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(my_background_op(1));

    // 在任务运行时执行其他工作……
    println!("Main function continues...");

    // 等待生成任务的结果。
    let result = handle.await.unwrap();
    println!("Result from task: {}", result);
}
```

如果一个生成的任务发生 panic，等待其 `JoinHandle` 将返回一个 `JoinError`。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        panic!("something went wrong!")
    });

    // 因为任务发生 panic，await 返回一个 Err 结果。
    assert!(handle.await.is_err());
}
```

## 任务取消

你可以通过调用其 `JoinHandle` 上的 `abort()` 方法来取消一个正在运行的任务。这会向任务发出信号，在下一次于 `.await` 点让出时关闭。等待一个被中止任务的句柄将导致一个 `JoinError`，表明它已被取消。

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // 这个任务将长时间运行
        sleep(Duration::from_secs(10)).await;
    });

    // 中止任务
    handle.abort();

    // 现在等待句柄将返回一个已取消的错误
    let err = handle.await.unwrap_err();
    assert!(err.is_cancelled());
}
```

请注意，`abort()` 仅调度取消操作。要等待任务完全关闭，你仍然必须 `.await` `JoinHandle`。

## 使用 `JoinSet` 管理多个任务

当你需要生成几个相关的任务并在它们完成时处理它们的结果时，`JoinSet` 是一个极好的工具。它允许你添加任务，然后等待下一个完成的任务，而无需手动管理一个 `JoinHandle` 的集合。

当一个 `JoinSet` 被丢弃时，集合中所有仍然存在的任务都会被自动中止。

```rust
use tokio::task::JoinSet;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..5 {
        set.spawn(async move {
            sleep(Duration::from_millis(100 * i)).await;
            i * 2
        });
    }

    while let Some(res) = set.join_next().await {
        let output = res.unwrap();
        println!("Task completed with result: {}", output);
    }
}
```

## 处理阻塞操作

因为任务是协作式调度的，直接在异步任务中运行阻塞操作将阻塞整个工作线程，从而阻止该线程上的任何其他任务运行。Tokio 提供了两种主要机制来处理这个问题。

### `spawn_blocking`

`spawn_blocking` 函数接受一个同步闭包，并在专为阻塞操作设计的专用线程池上运行它。这可以防止主异步工作线程被阻塞。

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let blocking_task = task::spawn_blocking(|| {
        // 这在阻塞线程上运行。
        // 在这里可以进行同步文件 I/O 或繁重的计算。
        std::thread::sleep(std::time::Duration::from_secs(1));
        "done"
    });

    // 来自 spawn_blocking 的 JoinHandle 可以像其他任何句柄一样被等待。
    let result = blocking_task.await?;
    assert_eq!(result, "done");
    Ok(())
}
```

### `block_in_place`

当使用多线程运行时，`block_in_place` 提供了另一种选择。它向运行时发出信号，表明当前线程即将阻塞。然后，运行时可以将此线程上的其他任务移动到不同的工作线程以保持它们运行，而当前线程则专用于阻塞操作。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
     task::block_in_place(|| {
        // 这在*同一个*线程上运行，但运行时已经
        // 将其他任务移走了。
        std::thread::sleep(std::time::Duration::from_secs(1));
    });
}
```

## 让出

协作式调度意味着任务会一直运行，直到遇到 `.await`。如果你的任务在没有等待的情况下进行了大量计算，它可能会独占调度器。你可以通过调用 `tokio::task::yield_now().await` 主动将控制权交还给调度器。这给了其他待处理的任务一个运行的机会。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("Spawned task running");
    });

    println!("Main task running");
    // 让出，允许新生成的任务在我们继续之前有机会运行。
    task::yield_now().await;
    println!("Main task resumed");
}
```

---

理解如何有效管理任务是构建健壮的 Tokio 应用程序的基础。既然你已经知道如何运行和协调并发操作，你可以探索这些任务是如何执行工作的。

接下来，让我们看看 Tokio 如何处理 [异步 I/O](./concepts-io.md)。
