# 任务

用于管理并发操作的异步绿色线程。该模块提供了在 Tokio 运行时中生成、管理和同步异步任务的核心工具。

任务是 Tokio 中执行的基本单元。它们是轻量级、非阻塞和协作调度的，使你能够高效地运行大量的并发操作。要深入了解任务背后的理论，请参阅[任务与调度概念指南](./concepts-tasks.md)。

本页为最常见的任务相关功能提供了 API 参考。

### 核心函数

<x-cards data-columns="2">
  <x-card data-title="spawn" data-icon="lucide:play-circle">
    生成一个新的异步任务以并发运行。
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:loader-2">
    在专用的线程池上运行阻塞函数，防止其阻塞异步运行时。
  </x-card>
  <x-card data-title="yield_now" data-icon="lucide:rotate-cw">
    将执行权交还给调度器，允许其他任务运行。
  </x-card>
  <x-card data-title="JoinSet" data-icon="lucide:box-select">
    用于管理一组动态生成的任务的集合。
  </x-card>
</x-cards>

## 生成任务

创建新任务的主要方式是使用 `tokio::spawn` 函数。

### `spawn`

生成一个新的异步任务，并为其返回一个 `JoinHandle`。这相当于 `std::thread::spawn` 的异步版本。

即使没有等待 `JoinHandle`，所提供的 future 也会立即在后台开始运行。根据运行时的配置，任务可能在当前线程上执行，也可能被移动到不同的工作线程。

```rust Spawning a task icon=logos:rust
use tokio::net::TcpListener;
use std::io;

async fn process_socket(socket: tokio::net::TcpStream) {
    // ... 处理连接
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        // 生成一个新任务以并发处理每个连接。
        tokio::spawn(async move {
            process_socket(socket).await;
        });
    }
}
```

要运行多个任务并等待它们的结果，你可以存储它们的 `JoinHandle`。

```rust Waiting for multiple tasks icon=logos:rust
# #[tokio::main(flavor = "current_thread")] async fn main() {
async fn my_background_op(id: i32) -> String {
    format!("Finished background task {}.", id)
}

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
# }
```

**Panic**

如果在 Tokio 运行时上下文之外调用此函数，将会引发 panic。

## 处理阻塞操作

异步任务不应执行阻塞操作，因为这会暂停整个工作线程，阻止其他任务取得进展。Tokio 提供了两个函数来安全地集成阻塞代码。

### `spawn_blocking`

在专用于阻塞任务的独立线程池上运行一个闭包。这是运行 CPU 密集型代码或同步 I/O 操作的首选方式。

```rust Using spawn_blocking icon=logos:rust
use tokio::task;

async fn compute_and_save() -> Result<(), Box<dyn std::error::Error>> {
    let data_to_compute = "some complex data".to_string();

    let result = task::spawn_blocking(move || {
        // 这在阻塞线程上运行。
        // 执行计算密集型工作或同步 I/O。
        let processed = data_to_compute.to_uppercase();
        std::fs::write("output.txt", processed)
    }).await?;

    result?;
    println!("Blocking operation complete.");
    Ok(())
}
```

使用 `spawn_blocking` 生成的任务不能被直接中止。如果运行时关闭，它将无限期地等待这些任务完成，除非配置了关闭超时。

### `block_in_place`

该函数将*当前*工作线程转换为阻塞线程，允许执行阻塞操作而不会拖慢运行时。它通过将当前线程上的其他任务移交给一个新的工作线程来实现这一点。这可能比 `spawn_blocking` 更高效，因为它避免了上下文切换，但它仅在多线程运行时中可用。

```rust Using block_in_place icon=logos:rust
use tokio::task;

# async fn docs() {
let result = task::block_in_place(|| {
    // 执行一些计算密集型工作或调用同步代码
    "blocking completed"
});

assert_eq!(result, "blocking completed");
# }
```

**Panic**

如果在 `current_thread` 运行时中调用此函数，将会引发 panic，因为没有其他工作线程可以分流任务。

## 让步

### `yield_now`

将执行权交还给 Tokio 调度器，允许其他待处理的任务运行。当前任务被放置在队列的末尾，并将在稍后恢复。

```rust Yielding execution icon=logos:rust
use tokio::task;

# #[tokio::main] async fn main() {
async {
    task::spawn(async {
        println!("spawned task done!")
    });

    // 让步，允许新生成的任务先执行。
    task::yield_now().await;
    println!("main task done!");
}
# .await;
# }
```

## 任务集合

### `JoinSet<T>`

一个用于管理一组已生成任务的集合。它允许你按完成顺序等待任务，这在不需要一次性等待所有任务或任务持续时间不同的情况下非常有用。

当 `JoinSet` 被丢弃时，集合中所有剩余的任务都将被中止。

```rust Managing tasks with JoinSet icon=logos:rust
use tokio::task::JoinSet;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    for i in 0..5 {
        set.spawn(async move {
            tokio::time::sleep(Duration::from_millis(100 * i)).await;
            i
        });
    }

    while let Some(res) = set.join_next().await {
        let completed_task_index = res.unwrap();
        println!("Task {} completed!", completed_task_index);
    }
}
```

## `!Send` Future

标准的 `tokio::spawn` 要求 future 是 `Send` 的，这意味着它们可以安全地在线程之间移动。对于 `!Send` 的 future（例如，持有 `Rc<T>` 的 future），你必须使用 `LocalSet`。

### `LocalSet` 和 `spawn_local`

`LocalSet` 在当前线程上执行任务。在 `LocalSet` 上下文中使用 `spawn_local` 生成的任何任务都保证会保留在该线程上，从而可以安全地使用 `!Send` 类型。

```rust Spawning a !Send future icon=logos:rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    // Rc 是 !Send
    let nonsend_data = Rc::new("my local data");

    let local_set = task::LocalSet::new();

    // 运行 LocalSet
    local_set.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();

        // spawn_local 可以接受 !Send future。
        let handle = task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
            42
        });

        let result = handle.await.unwrap();
        assert_eq!(result, 42);
    }).await;
}
```

如果在 `LocalSet` 上下文之外调用 `spawn_local`，将会引发 panic。

## 任务句柄与取消

### `JoinHandle<T>`

由 `spawn` 和 `spawn_local` 返回，`JoinHandle` 是一个 future，它会解析为关联任务的输出。等待该句柄将会等待任务完成。

如果任务发生 panic，等待其 `JoinHandle` 将返回一个 `JoinError`。

```rust Handling a panicked task icon=logos:rust
use tokio::task;

# #[tokio::main] async fn main() {
let join = task::spawn(async {
    panic!("something bad happened!")
});

// 返回的结果表明任务失败了。
assert!(join.await.is_err());
# }
```

#### 取消

你可以通过在其 `JoinHandle` 上调用 `abort()` 方法来取消任务。这会向任务发出信号，使其在下一次到达 `.await` 点时关闭。要等待取消完成，你仍然必须 `.await` 该句柄。

```rust Aborting a task icon=logos:rust
use tokio::task;
use std::time::Duration;

# #[tokio::main] async fn main() {
let handle = task::spawn(async {
    // 除非被中止，否则此任务将永远运行。
    loop {
        tokio::time::sleep(Duration::from_secs(1)).await;
        println!("task is running...");
    }
});

tokio::time::sleep(Duration::from_millis(50)).await;
handle.abort();

let join_result = handle.await;
assert!(join_result.is_err());
assert!(join_result.unwrap_err().is_cancelled());
# }
```

### `AbortHandle`

`AbortHandle` 提供了中止任务的能力，而无需等待其结果。一个任务可以有多个 `AbortHandle`，但只能有一个 `JoinHandle`。这对于将任务管理与任务完成的关注点分离非常有用。

---

以上涵盖了 Tokio 中任务管理的核心 API。有关运行时如何调度和执行这些任务的详细信息，请参阅[运行时 API 参考](./api-runtime.md)。