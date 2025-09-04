# 任务与调度

Tokio 中的异步程序围绕任务构建。任务是一个轻量级的、非阻塞的执行单元，与其他任务并发运行。与传统线程由操作系统管理不同，任务由 Tokio 运行时管理，这使得它们的创建和管理成本大大降低。

Tokio 任务的主要特点包括：

*   **轻量级：** 与操作系统线程相比，创建和在任务之间切换的开销非常低。
*   **协作式调度：** 任务会一直运行，直到它们自愿将控制权交还给调度器（通常在 `.await` 处），从而允许其他任务运行。这与操作系统线程使用的抢占式多任务处理不同。
*   **非阻塞：** 任务不应执行阻塞线程的操作，例如同步 I/O 或繁重的、长时间运行的计算。这样做会阻止同一线程上的其他任务继续执行。

这种模型使得少量操作系统线程能够高效地处理大量并发任务。

```d2
direction: down

OS: {
  shape: package
  label: "Operating System"
  grid-columns: 2

  Thread-1: {
    label: "OS Thread 1"
    shape: rectangle
  }
  Thread-2: {
    label: "OS Thread 2"
    shape: rectangle
  }
}

Tokio-Runtime: {
  label: "Tokio Runtime"
  shape: package

  Scheduler: {
    shape: hexagon
  }

  Worker-Thread-1: {
    label: "Worker Thread 1"
    shape: rectangle

    Task-A: { label: "Task A"; shape: circle }
    Task-B: { label: "Task B"; shape: circle }
    Task-C: { label: "Task C"; shape: circle }
  }
  Worker-Thread-2: {
    label: "Worker Thread 2"
    shape: rectangle

    Task-D: { label: "Task D"; shape: circle }
    Task-E: { label: "Task E"; shape: circle }
  }

  OS.Thread-1 -> Worker-Thread-1: "Runs"
  OS.Thread-2 -> Worker-Thread-2: "Runs"
  Scheduler -> Worker-Thread-1: "Manages"
  Scheduler -> Worker-Thread-2: "Manages"
  Worker-Thread-1.Task-A <-> Worker-Thread-1.Task-B: "Cooperatively\nYields"
  Worker-Thread-1.Task-B <-> Worker-Thread-1.Task-C: "Cooperatively\nYields"
}
```

## 生成任务

创建任务最常用的方法是使用 `tokio::spawn` 函数。它接收一个异步代码块或 future，并立即在后台开始运行，同时返回一个可用于与该任务交互的 `JoinHandle`。

```rust,no_run
use tokio::net::{TcpListener, TcpStream};
use std::io;

async fn process(socket: TcpStream) {
    // ... 处理连接
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        // 为每个连接生成一个新任务以并发处理。
        tokio::spawn(async move {
            process(socket).await
        });
    }
}
```

`JoinHandle` 允许你等待任务完成并获取其返回值。

```rust
# #[tokio::main] async fn main() -> Result<(), Box<dyn std::error::Error>> {
let join_handle = tokio::spawn(async {
    // ... 执行一些工作
    "hello world!"
});

// 等待生成任务的结果。
let result = join_handle.await?;
assert_eq!(result, "hello world!");
# Ok(())
# }
```

如果任务发生 panic，等待其 `JoinHandle` 将返回一个 `JoinError`。

## 使用 `JoinSet` 管理多个任务

当你需要管理一个动态的任务集合时，`JoinSet` 是一个强大的实用工具。它允许你生成多个任务，并按完成顺序等待它们的结果。

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

当 `JoinSet` 被丢弃时，其中仍包含的所有任务都会被自动中止。

## 任务取消

任务可以通过其 `JoinHandle` 或 `AbortHandle` 上的 `abort` 方法来取消。取消是一个请求任务在下一个 `.await` 点关闭的信号。当一个任务被取消时，等待其 `JoinHandle` 将返回一个表示任务已被取消的 `JoinError`。

请注意，调用 `abort` 仅会调度取消操作。要等待任务完全关闭，你仍必须等待其 `JoinHandle`。

## 处理阻塞操作

由于任务是协作调度的，在异步任务中直接执行阻塞操作会阻塞整个工作线程，从而阻止其他任务运行。Tokio 提供了两种主要机制来处理这个问题。

### `spawn_blocking`

对于同步的 I/O 密集型或 CPU 密集型工作，请使用 `spawn_blocking`。此函数会在一个专用于阻塞操作的独立线程池上运行提供的闭包，而不会干扰异步运行时的主要调度器。

```rust
# use tokio::task;
# async fn docs() -> Result<(), Box<dyn std::error::Error>>{
let join_handle = task::spawn_blocking(|| {
    // 执行一些计算密集型工作或调用同步 I/O 代码。
    "blocking operation completed"
});

let result = join_handle.await?;
assert_eq!(result, "blocking operation completed");
# Ok(())
# }
```

使用 `spawn_blocking` 生成的任务一旦开始运行便无法中止。

### `block_in_place`

在使用多线程运行时，`block_in_place` 提供了另一种选择。它会通知调度器当前线程即将阻塞。然后，运行时可以将此线程上调度的其他任务移交给另一个工作线程，以防止它们被阻塞。这种方式可能比 `spawn_blocking` 更高效，因为它可能会避免线程上下文切换。

```rust
use tokio::task;

# #[tokio::main] async fn main() {
let result = task::block_in_place(|| {
    // 执行一些计算密集型工作或调用同步代码
    "blocking completed"
});

assert_eq!(result, "blocking completed");
# }
```

如果从单线程运行时调用此函数，将会发生 panic。

## 让出控制权

你可以通过调用 `tokio::task::yield_now()` 将控制权主动让给 Tokio 调度器。这使得调度器可以在恢复当前任务之前，先运行其他待处理的任务。

```rust
use tokio::task;

# #[tokio::main] async fn main() {
async {
    task::spawn(async {
        println!("spawned task done!")
    });

    // 让出控制权，允许新生成的任务有可能先执行。
    task::yield_now().await;
    println!("main task done!");
}.await;
}
```

值得注意的是，确切的调度顺序无法保证。运行时可能会选择在运行其他任务之前，立即再次轮询让出控制权的任务。

---

现在你已经了解了如何创建和管理任务，下一步是学习这些任务如何执行工作。请继续阅读 [异步 I/O](./concepts-io.md) 部分，了解 Tokio 如何处理非阻塞的网络和文件操作。
