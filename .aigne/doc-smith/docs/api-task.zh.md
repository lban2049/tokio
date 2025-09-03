# 任务

Tokio 中用于管理并发操作的异步绿色线程。有关概念性概述，请参阅 [任务与调度](./concepts-tasks.md) 指南。

## 什么是任务？

_任务_ 是一个轻量级、非阻塞的执行单元。任务与操作系统线程类似，但它不由操作系统调度器管理，而是由 Tokio 运行时管理。这种模式也被称为 [绿色线程](https://en.wikipedia.org/wiki/Green_threads)。

关于任务的关键点包括：

*   **轻量级**：与操作系统线程相比，创建、切换和销毁任务的开销非常低。
*   **协作式**：任务会一直运行直到让出（例如，在 `.await` 点），从而允许 Tokio 调度器运行另一个任务。
*   **非阻塞**：任务不应执行标准 I/O 等阻塞操作，因为这会阻塞整个工作线程。作为替代方案，Tokio 提供了用于异步运行阻塞操作的 API。

### 任务生命周期

下图说明了 Tokio 任务的基本生命周期。

```d2
direction: down

"spawn()": { shape: oval }
"运行中": { shape: rectangle }
"已让出": { shape: rectangle; style.stroke-dash: 2 }
"已完成": { shape: oval; style.fill: "#d4edda" }
"已 Panic": { shape: oval; style.fill: "#f8d7da" }
"已取消": { shape: oval; style.fill: "#fff3cd" }

"spawn()" -> "运行中": "已调度执行"
"运行中" -> "已让出": ".await 等待未完成的 Future"
"已让出" -> "运行中": "被事件源唤醒"
"运行中" -> "已完成": "Future 返回 Ready(val)"
"运行中" -> "已 Panic": "调用了 panic!()"

"运行中" -> "已取消": "JoinHandle.abort()"
"已让出" -> "已取消": "JoinHandle.abort()"

```

## 生成与执行

Tokio 提供了几个用于生成任务的函数，每个函数都适用于不同的场景。

<x-cards data-columns="2">
  <x-card data-title="spawn" data-icon="lucide:play-circle">
    生成一个新的异步任务以并发运行。这是创建新任务的主要函数。
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:cpu">
    在专用线程池上运行阻塞函数，防止其阻塞异步运行时。
  </x-card>
  <x-card data-title="block_in_place" data-icon="lucide:pause-circle">
    将当前工作线程转换为阻塞线程，适用于短暂、不频繁的阻塞操作。
  </x-card>
  <x-card data-title="yield_now" data-icon="lucide:fast-forward">
    将执行权交还给 Tokio 调度器，允许其他任务运行。
  </x-card>
</x-cards>

### spawn

生成一个新的异步任务，并为其返回一个 `JoinHandle`。该任务会立即在后台开始运行。

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    let s = format!("Starting background task {}.", id);
    println!("{}", s);
    s
}

#[tokio::main]
async fn main() {
    let ops = vec![1, 2, 3];
    let mut tasks = Vec::with_capacity(ops.len());
    for op in ops {
        tasks.push(tokio::spawn(my_background_op(op)));
    }

    let mut outputs = Vec::with_capacity(tasks.len());
    for task in tasks {
        outputs.push(task.await.unwrap());
    }
    println!("{:?}", outputs);
}
```

`spawn` 返回的 `JoinHandle` 是一个 future，它会解析为任务的输出。对其进行 await 可以获取任务的结果。如果任务发生 panic，await 该 handle 将返回一个 `JoinError`。

### spawn_blocking

使用 `spawn_blocking` 来执行 CPU 密集型或阻塞 I/O 代码，而不会干扰其他异步任务。

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut v = "Hello, ".to_string();
    let res = task::spawn_blocking(move || {
        // 这代表计算密集型工作
        v.push_str("world");
        v
    }).await?;

    assert_eq!(res.as_str(), "Hello, world");
    Ok(())
}
```

### block_in_place

该函数允许在多线程运行时中从异步上下文内部运行阻塞操作。它会将当前工作线程转换为阻塞线程，并将其他任务移至不同的工作线程。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let result = task::block_in_place(|| {
        // 执行一些计算密集型工作或调用同步代码
        "blocking completed"
    });

    assert_eq!(result, "blocking completed");
}
```

### yield_now

调用 `yield_now().await` 会将控制权交还给调度器，使其能够轮询其他任务。当前任务被放置在队列的末尾，并将在稍后恢复。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("spawned task done!")
    });

    // 让出，允许新生成的任务先执行。
    task::yield_now().await;
    println!("main task done!");
}
```

## 管理任务集合：JoinSet

`JoinSet` 是一个集合，允许你生成和管理多个任务，并以任意顺序等待它们完成。

- **`spawn()`**：向集合中添加一个新任务。
- **`join_next()`**：等待集合中下一个任务的完成。
- **`abort_all()`**：中止集合中的所有任务。
- **`shutdown()`**：中止所有任务并等待它们完成。

当 `JoinSet` 被丢弃时，其中的所有任务都会被中止。

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..10 {
        set.spawn(async move { i });
    }

    let mut seen = [false; 10];
    while let Some(res) = set.join_next().await {
        let idx = res.unwrap();
        seen[idx] = true;
    }

    for i in 0..10 {
        assert!(seen[i]);
    }
}
```

## 用于 `!Send` Future 的本地任务

某些类型（如 `std::rc::Rc`）不是 `Send` 类型，不能安全地在线程间发送。为了处理使用这类类型的 future，Tokio 提供了一种机制来确保它们在单个线程上运行。

### LocalSet

`LocalSet` 是一个任务执行器，它在当前线程上运行其所有任务。这允许你生成 `!Send` future。

### spawn_local

在 `LocalSet` 中，你必须使用 `spawn_local` 来生成 `!Send` future。该函数保证生成的任务将保留在创建 `LocalSet` 的线程上。

```rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("my nonsend data...");

    // 构建一个本地任务集。
    let local = task::LocalSet::new();

    // 运行本地任务集，直到 future 完成。
    local.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();
        // `spawn_local` 确保 future 在当前线程上运行。
        task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
        }).await.unwrap();
    }).await;
}
```

## 取消

可以使用任务的 `JoinHandle` 或关联的 `AbortHandle` 上的 `abort` 方法来取消任务。

- 当任务被中止时，它会在下一次在 `.await` 点让出时收到关闭信号。
- 所有局部变量都会被丢弃，并运行其析构函数。
- 等待一个已中止任务的 `JoinHandle` 将导致一个 `JoinError`，其中 `is_cancelled()` 返回 `true`。
- 请注意，`spawn_blocking` 任务一旦开始运行就无法中止。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        panic!("something bad happened!")
    });

    // 返回的结果表明任务失败。
    assert!(handle.await.is_err());
}
```

## 相关数据结构

| Struct | Description |
|---|---|
| `JoinHandle<T>` | 由生成函数返回的 handle。可以对其进行 await 以获取任务的输出 `T` 或 `JoinError`。 |
| `JoinError` | 当任务因 panic 或被取消而失败时返回的错误。 |
| `AbortHandle` | 一个可用于中止任务而无需等待其完成的 handle。 |
| `Id` | 已生成任务的唯一标识符。 |

---

本页涵盖了任务管理的主要 API。有关任务如何调度和执行的详细信息，请参阅 [Tokio 运行时](./api-runtime.md) 的文档。
