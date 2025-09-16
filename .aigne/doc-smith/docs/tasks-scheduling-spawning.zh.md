# 生成和管理任务

在 Tokio 中，任务是一个轻量级的、非阻塞的执行单元。为了实现并发，你需要将异步代码作为任务来运行。本节将介绍生成和管理这些任务的主要方法，从基本的“即发即忘”操作到处理大型、动态的并发工作集。

## 使用 `tokio::spawn` 生成异步任务

创建新任务最常用的方法是使用 `tokio::spawn` 函数。它接收一个 `async` 块或任何 future，并将其提交给 Tokio 运行时以并发执行。

生成的任务必须是 `'static` 的，并且其 future 必须是 `Send` 的，这意味着它可以安全地在线程之间移动。这使得 Tokio 调度器可以在任何可用的工作线程上高效地执行该任务。

```rust icon=logos:rust Spawning a connection handler
use tokio::net::{TcpListener, TcpStream};
use std::io;

async fn process(socket: TcpStream) {
    // ... 处理连接 ...
    # drop(socket);
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        // 为每个入站套接字生成一个新任务。
        // `move` 关键字用于将套接字的所有权转移
        // 到新任务中。
        tokio::spawn(async move {
            // 并发处理每个套接字。
            process(socket).await
        });
    }
}
```

### 使用 `JoinHandle` 等待任务完成

当你生成一个任务时，`tokio::spawn` 会返回一个 `JoinHandle`。这个句柄本身就是一个 future，你可以对其进行 `.await` 操作以获取生成任务的输出。通过这种方式，你可以等待一个并发操作完成并获取其结果。

```rust icon=logos:rust Awaiting a JoinHandle
# #[tokio::main(flavor = "current_thread")] async fn main() {
async fn my_background_op(id: i32) -> String {
    let s = format!("正在启动后台任务 {}.\n", id);
    println!("{}", s);
    s
}

let handle = tokio::spawn(my_background_op(1));

// 在任务运行时执行其他工作...

let output = handle.await.unwrap();
println!("{:?}", output);
# }
```

如果生成的任务发生 panic，对其 `JoinHandle` 进行 await 操作将返回一个包含 `JoinError` 的 `Result`。

### 任务取消

可以使用 `JoinHandle` 上的 `abort` 方法来取消任务。这会向任务发出信号，使其在下一次到达 `.await` 点时关闭。要等待取消操作完成，你仍然需要 await 该 `JoinHandle`。

```rust icon=logos:rust Aborting a task
use std::time::Duration;
use tokio::time::sleep;

# #[tokio::main] async fn main() {
let handle = tokio::spawn(async {
    println!("任务正在运行...");
    sleep(Duration::from_secs(10)).await;
    println!("任务完成！"); // 这行不会被打印
});

sleep(Duration::from_millis(100)).await;

// 中止任务
handle.abort();

// 等待一个被取消的任务会返回一个 `JoinError`
let err = handle.await.unwrap_err();
assert!(err.is_cancelled());
# }
```

## 使用 `spawn_blocking` 处理阻塞操作

异步任务永远不应执行阻塞操作，因为这会暂停整个工作线程，从而阻止其他任务运行。对于同步 I/O 或 CPU 密集型计算，请使用 `tokio::task::spawn_blocking`。

该函数在一个专用于阻塞任务的线程池上运行所提供的闭包，确保它不会干扰异步调度器。

```rust icon=logos:rust Running a blocking operation
use tokio::task;

# async fn docs() -> Result<(), Box<dyn std::error::Error>>{
let v = "Hello, ".to_string();
let join_handle = task::spawn_blocking(move || {
    // 这段代码在一个阻塞线程上运行。
    // 在这里阻塞是可以的。
    let result = v + "world";
    result
});

let result = join_handle.await?;
assert_eq!(result, "Hello, world");
# Ok(())
# }
```

请注意，使用 `spawn_blocking` 生成的任务一旦开始运行就无法中止。运行时在关闭期间会等待它们完成。

## 使用 `spawn_local` 处理 `!Send` 的 future

某些类型（如 `std::rc::Rc`）不是 `Send` 的，无法安全地在线程之间移动。如果你的 future 在 `.await` 点之间持有了这种类型的变量，你就不能使用 `tokio::spawn`。在这种情况下，你可以使用 `LocalSet` 在单个线程上运行任务。

在 `LocalSet` 中，你可以使用 `tokio::task::spawn_local` 来生成 `!Send` 的 future。生成的 future 将始终在其创建的同一线程上运行。

```rust icon=logos:rust Spawning a !Send future
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    // Rc 不是 `Send` 的
    let nonsend_data = Rc::new("my nonsend data...");

    // LocalSet 允许我们运行 `!Send` 的 future。
    let local = task::LocalSet::new();

    // 运行 LocalSet
    local.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();
        
        // spawn_local 用于处理 `!Send` 的 future
        task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
            // ...
        }).await.unwrap();
    }).await;
}
```

## 使用 `JoinSet` 管理任务组

当你需要管理一个动态的任务集合时，`JoinSet` 是理想的工具。它允许你生成多个任务，并按照它们完成的顺序等待其完成，而无需手动管理一个 `Vec<JoinHandle>`。

当 `JoinSet` 被丢弃时，其中的所有任务都会被自动中止。

```rust icon=logos:rust Using JoinSet to manage multiple tasks
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

`JoinSet` 为管理并发操作提供了一个强大且符合人体工程学的 API，包括以下方法：

| Method           | Description                                                              |
| ---------------- | ------------------------------------------------------------------------ |
| `spawn`          | 向集合中生成一个 `Send` 任务。                                       |
| `spawn_local`    | 向集合中生成一个 `!Send` 任务（需要 `LocalSet`）。              |
| `spawn_blocking` | 向集合中生成一个阻塞任务。                                     |
| `join_next`      | 等待集合中的下一个任务完成。                             |
| `abort_all`      | 中止集合中当前的所有任务。                                   |
| `shutdown`       | 中止所有任务并等待它们完成。                           |

## 使用 `yield_now` 让出执行权

有时，你可能想让 Tokio 调度器有机会运行其他待处理的任务。你可以通过调用 `tokio::task::yield_now().await` 来实现。这会将当前任务放到队列的末尾，从而允许其他任务被轮询。

```rust icon=logos:rust Yielding to other tasks
use tokio::task;

# #[tokio::main] async fn main() {
async {
    task::spawn(async {
        println!("生成的任务已完成！")
    });

    // 让出执行权，允许新生成的任务先执行。
    task::yield_now().await;
    println!("主任务已完成！");
}
# .await;
# }
```

但是，请谨慎使用此功能。通常无法保证调度器接下来会运行哪个任务，而且结构良好的异步代码很少需要显式地让出执行权。

---

借助这些工具，你可以在 Tokio 应用程序中有效地创建和管理并发操作。选择 `spawn`、`spawn_blocking` 还是 `spawn_local` 完全取决于你的任务需要执行的工作性质。

接下来，学习如何使用[同步原语](./tasks-scheduling-synchronization.md)在这些任务之间进行协调和通信。