# 任务

用于管理并发操作的异步绿色线程。

## 概述

`_任务_` 是一个轻量级的非阻塞执行单元。任务类似于操作系统线程，但任务不是由操作系统调度器管理，而是由 Tokio 运行时管理。这种模式也被称为[绿色线程](https://en.wikipedia.org/wiki/Green_threads)。如果你熟悉 Go 的 goroutine、Kotlin 的协程或 Erlang 的进程，你可以将 Tokio 的任务视为类似的东西。

关于任务的要点包括：

*   **轻量级**：创建和切换任务的开销很低，因为它不需要操作系统上下文切换。
*   **协作式调度**：任务会一直运行直到它们让出（yield）（例如，在 `.await` 点），从而允许 Tokio 调度器运行另一个任务。
*   **非阻塞**：任务不应执行标准 I/O 等阻塞操作，因为这会阻塞整个工作线程。相反，Tokio 提供了用于处理阻塞代码的特定 API。

```d2
direction: down

"你的应用程序代码" {
  shape: cloud
}

"Tokio 运行时" {
  shape: package

  调度器 {
    shape: hexagon
    grid-columns: 2

    "工作线程" {
      label: "工作线程\n（用于异步任务）"
      shape: package
      
      Task-1 {
        label: "异步任务 1"
        shape: rectangle
      }
      Task-2 {
        label: "异步任务 2"
        shape: rectangle
      }
      Task-N {
        label: "..."
        shape: rectangle
      }
    }
    
    "阻塞线程池" {
      label: "阻塞线程池\n（用于同步代码）"
      shape: package
      
      Blocking-Task-1 {
        label: "阻塞操作 1"
        shape: rectangle
      }
      Blocking-Task-M {
        label: "..."
        shape: rectangle
      }
    }
  }
}

"你的应用程序代码" -> "Tokio 运行时".调度器."工作线程".Task-1: "tokio::spawn(async {...})"
"你的应用程序代码" -> "Tokio 运行时".调度器."阻塞线程池".Blocking-Task-1: "tokio::spawn_blocking(|| {...})"

"Tokio 运行时".调度器."工作线程".Task-1 -> "Tokio 运行时".调度器."工作线程".Task-2: "让出"
```

该模块提供了用于生成、取消和协调任务的 API。

---

## 生成任务

处理任务最常见的方式是生成它们以进行并发执行。

<x-cards data-columns="2">
  <x-card data-title="spawn" data-icon="lucide:play-circle">
    生成一个新的异步任务以并发运行。
  </x-card>
  <x-card data-title="spawn_local" data-icon="lucide:anchor">
    在当前线程的 `LocalSet` 上生成一个 `!Send` future。
  </x-card>
</x-cards>

### `spawn`

生成一个新的异步任务，并为其返回一个 `JoinHandle`。提供的 future 将立即在后台开始运行。

此函数必须在 Tokio 运行时的上下文中调用。生成的任务可能在当前线程上执行，也可能被发送到另一个线程。

```rust
use tokio::net::{TcpListener, TcpStream};
use std::io;

async fn process(socket: TcpStream) {
    // ...
#   drop(socket);
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            // 并发处理每个套接字。
            process(socket).await
        });
    }
}
```

返回的 `JoinHandle` 是一个 future，可以对其进行 `await` 以获取任务的输出。如果任务发生 panic，`await` `JoinHandle` 将返回一个 `JoinError`。

```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let join_handle = task::spawn(async {
        // ...
        "hello world!"
    });

    // 等待生成任务的结果。
    let result = join_handle.await?;
    assert_eq!(result, "hello world!");
    Ok(())
}
```

**Panics**：如果在 Tokio 运行时之外调用此函数，则会发生 panic。

### `spawn_local`

在当前的 `LocalSet` 上生成一个 `!Send` future。

这对于持有不能安全地跨线程发送的数据（例如 `Rc<T>`）的 future 是必需的。生成的 future 将始终在调用 `spawn_local` 的同一线程上运行。

```rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("my nonsend data...");

    let local = task::LocalSet::new();

    // 运行本地任务集。
    local.run_until(async move {
        let nonsend_data_clone = nonsend_data.clone();
        task::spawn_local(async move {
            println!("{}", nonsend_data_clone);
            // ...
        }).await.unwrap();
    }).await;
}
```

**Panics**：如果在 `LocalSet` 之外调用此函数，则会发生 panic。

---

## 管理任务集合

### `JoinSet`

`JoinSet<T>` 是一个任务集合，其中所有任务都具有相同的返回类型 `T`。它允许你生成多个任务，并按照它们完成的顺序等待它们完成。

当 `JoinSet` 被丢弃时，集合中所有仍在运行的任务都将被中止。

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

`JoinSet` 提供了 `spawn`、`spawn_blocking`、`join_next`、`abort_all` 和 `shutdown` 等方法，用于管理集合内任务的生命周期。

---

## 处理阻塞代码

异步任务不应执行阻塞操作，因为这会阻止同一线程上的其他任务运行。Tokio 提供了两个 API，用于在异步上下文中安全地运行阻塞代码。

### `spawn_blocking`

在专用于阻塞操作的线程池上运行阻塞闭包。这是运行 CPU 密集型代码或同步 I/O 的首选方法。

```rust
use tokio::task;

async fn docs() -> Result<(), Box<dyn std::error::Error>>{
    let mut v = "Hello, ".to_string();
    let res = task::spawn_blocking(move || {
        // 这是计算密集型工作的替代品
        v.push_str("world");
        v
    }).await?;

    assert_eq!(res.as_str(), "Hello, world");
    Ok(())
}
```

### `block_in_place`

此函数在多线程运行时上可用，它将当前工作线程转换为阻塞线程。它会将当前线程上的其他任务移动到另一个工作线程，这可以通过避免上下文切换来提高性能。

```rust
use tokio::task;

async fn docs() {
    let result = task::block_in_place(|| {
        // 执行一些计算密集型工作或调用同步代码
        "blocking completed"
    });

    assert_eq!(result, "blocking completed");
}
```

---

## 协作式让出

### `yield_now`

将执行权交还给 Tokio 调度器，允许其他任务运行。当前任务被放置在待处理队列的末尾，并将在稍后再次被轮询。

```rust
use tokio::task;

async fn example() {
    task::spawn(async {
        println!("spawned task done!")
    });

    // 让出，允许新生成的任务先执行。
    task::yield_now().await;
    println!("main task done!");
}
```

---

## 任务取消

生成的任务可以使用其 `JoinHandle` 或 `AbortHandle` 上的 `abort` 方法来取消。取消是一个信号，请求任务在其下一个 `.await` 点关闭。

-   `JoinHandle::abort()`：安排任务以进行取消。
-   中止后等待 `JoinHandle` 将会失败，并返回一个 `JoinError::is_cancelled` 错误。
-   中止并不能保证立即终止。任务会一直运行到其下一个让出点。
-   使用 `spawn_blocking` 生成的任务一旦开始运行就无法中止。

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        // 一个长时间运行的任务
        tokio::time::sleep(std::time::Duration::from_secs(10)).await;
    });

    // 取消任务
    handle.abort();

    // 现在等待句柄将返回一个已取消的错误。
    let result = handle.await;
    assert!(result.is_err());
    assert!(result.unwrap_err().is_cancelled());
}
```

现在你已经了解了如何管理任务，你可能想学习如何协调它们。有关更多信息，请参阅[同步原语](./api-sync.md)文档。