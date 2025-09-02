# 任务和调度

在 Tokio 中，**任务**是一个轻量级的、非阻塞的执行单元。可以把任务看作是异步的绿色线程：它们与操作系统线程类似，但不是由操作系统管理，而是由 Tokio 运行时管理。与操作系统线程相比，这使得创建和切换任务的成本极低。

Tokio 任务的主要特点包括：

*   **轻量级**：创建、运行和销毁大量任务的开销非常低。
*   **协作式**：任务会一直运行，直到它们自愿将控制权交还给调度器（通常在 `.await` 点），从而允许其他任务运行。这与操作系统线程使用的抢占式多任务处理形成对比。
*   **非阻塞**：任务不应执行阻塞线程的操作，例如同步 I/O 或繁重的 CPU 计算。这样做会阻止同一线程上的其他任务取得进展。Tokio 提供了处理此类情况的特定工具。

```d2
direction: down

"Tokio Runtime" {
  shape: cloud

  "Worker Threads (Core)" {
    style.stroke-dash: 2
    "Worker Thread 1" {
      "Async Task A"
      "Async Task B"
    }
    "Worker Thread 2" {
      "Async Task C"
    }
  }

  "Blocking Thread Pool" {
    shape: package
    "Blocking Task X"
    "Blocking Task Y"
  }

  "Worker Thread 1" -> "Async Task A": polls
  "Worker Thread 1" -> "Async Task B": polls
  "Worker Thread 2" -> "Async Task C": polls
}
```

本节涵盖了使用任务的基本模式，从创建任务到管理其执行，再到处理阻塞操作等特殊情况。

## 生成任务

最基本的操作是创建或*生成*一个新的异步任务。这通过使用 `tokio::spawn` 函数来完成，该函数接受一个 future 并立即在运行时上并发执行它。

`tokio::spawn` 返回一个 `JoinHandle`，它本身也是一个 future。你可以 `.await` 这个 `JoinHandle` 来获取生成任务的输出。通过这种方式，你可以等待任务完成并检索其结果。

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    let s = format!("Processing background task {}.", id);
    println!("{}", s);
    // 模拟一些工作
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    s
}

#[tokio::main]
async fn main() {
    // 生成一个新任务。
    let join_handle = tokio::spawn(my_background_op(1));

    // 在主任务中执行其他工作。
    println!("Doing other work in main task.");

    // 等待生成任务的结果。
    match join_handle.await {
        Ok(result) => println!("Spawned task completed with result: '{}'", result),
        Err(e) => println!("Spawned task failed: {:?}", e),
    }
}
```

如果一个生成的任务发生 panic，对其 `JoinHandle` 进行 `.await` 将返回一个 `JoinError`，表示失败。

## 使用 JoinSet 管理多个任务

当你需要管理一组任务时，`JoinSet` 是一个强大的工具。它允许你生成多个任务，并在它们完成时等待其结果，而无需知道哪一个会先完成。

当 `JoinSet` 被 drop 时，集合中所有剩余的任务都会被自动中止。

```rust
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

    println!("Waiting for tasks to complete...");
    while let Some(res) = set.join_next().await {
        match res {
            Ok(val) => println!("Task {} completed.", val),
            Err(e) => println!("A task failed: {:?}", e),
        }
    }
    println!("All tasks finished.");
}
```

在此示例中，任务是按照它们完成的顺序（4、3、2、1、0）处理的，而不是它们被生成的顺序。

## 任务取消

任务可以通过其 `JoinHandle` 或 `AbortHandle` 上的 `abort` 方法来取消。取消是一个信号，请求任务在下一个 `.await` 点关闭。一旦任务被取消并完成关闭，对其 `JoinHandle` 进行 `.await` 将导致一个 `JoinError`，其中 `is_cancelled()` 返回 `true`。

需要注意的是，`abort()` 会安排取消操作并立即返回；它不会等待任务停止运行。为了确保任务完全停止，你应该先 `abort()` 它，然后 `.await` 它的 `JoinHandle`。

使用 `spawn_blocking` 生成的任务一旦开始执行就无法中止。

## 处理阻塞操作

因为 Tokio 的调度器是协作式的，所以任务不能在工作线程上执行阻塞操作，因为这会阻塞同一线程上的所有其他任务。为了处理必须阻塞的代码（例如，同步文件 I/O、CPU 密集型计算），Tokio 提供了两种主要的解决方案。

<x-cards data-columns="2">
  <x-card data-title="spawn_blocking" data-icon="lucide:cpu">
    在专门用于阻塞任务的独立线程池上运行阻塞函数。这是将阻塞代码集成到异步应用程序中的最常用方法。
  </x-card>
  <x-card data-title="block_in_place" data-icon="lucide:pause-circle">
    将当前工作线程转换为阻塞线程，允许运行时将其他任务迁移到新的工作线程。这通过避免上下文切换可以更高效，但仅在多线程运行时中可用。
  </x-card>
</x-cards>

### 使用 `spawn_blocking`

此函数将阻塞操作从主异步工作线程中移出，防止其干扰其他异步任务。

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut data = "start-".to_string();

    let result = task::spawn_blocking(move || {
        // 这正在一个阻塞线程上运行。
        // 在这里可以执行同步的、CPU 密集型的工作。
        std::thread::sleep(std::time::Duration::from_secs(1));
        data.push_str("end");
        data
    }).await?;

    assert_eq!(result, "start-end");
    Ok(())
}
```

### 使用 `block_in_place`

当你已经在一个多线程运行时的异步任务中，并且需要执行一个短暂的阻塞操作时，应该使用此函数。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let result = task::block_in_place(|| {
        // 这段代码现在运行在一个允许阻塞的线程上。
        std::thread::sleep(std::time::Duration::from_secs(1));
        "blocking completed"
    });

    assert_eq!(result, "blocking completed");
}
```

## 交出控制权

有时，你可能希望一个任务自愿放弃其执行时间，让其他任务运行。`tokio::task::yield_now().await` 函数正是为此而设计的。它将控制权交还给 Tokio 调度器，调度器会将当前任务放到队列的末尾，并调度另一个就绪的任务。

这对于确保长时间运行的计算不会饿死其他任务非常有用，即使计算本身是完全异步的。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("[spawned] Task starting");
        // ... work ...
        println!("[spawned] Task finished");
    });

    println!("[main] Before yield");
    // 交出控制权，允许新生成的任务有机会执行。
    task::yield_now().await;
    println!("[main] After yield");
}
```

现在你已经学习了在 Tokio 中创建和管理任务的基础知识。利用这些概念，你可以构建既高效又可扩展的并发应用程序。

接下来，学习如何执行非阻塞 I/O 操作，这是大多数任务的常见活动。

<x-card data-title="下一步：异步 I/O" data-icon="lucide:arrow-right" data-href="/concepts/io" data-cta="阅读更多" >
  探索 Tokio 用于网络、文件系统操作等的非阻塞原语。
</x-card>