# 任务

该模块提供了用于管理异步任务的 API，这些任务是轻量级的非阻塞执行单元，也称为绿色线程。与操作系统线程不同，任务由 Tokio 运行时管理，因此创建和切换的成本很低。

要更深入地从概念上理解任务，请参阅[任务与调度指南](./concepts-tasks.md)。

本页涵盖了以下主要 API：
- 并发生成新任务。
- 在异步应用程序中处理阻塞代码。
- 管理任务集合。
- 处理 `!Send` 数据。

## 生成任务

创建新任务的主要方法是使用 `tokio::spawn` 函数。

### `tokio::spawn`

生成一个新的异步任务，并为其返回一个 `JoinHandle`。生成的任务可能在当前线程上执行，也可能被发送到不同的线程，具体取决于运行时的配置。

即使不等待 `JoinHandle`，提供的 future 也会立即开始运行。

**何时使用：** 对于任何希望与其他任务并发运行的 `async` 操作，都应使用 `spawn`。

**签名**
```rust
fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
```

**示例：处理网络连接**

在服务器中，每个传入的连接都可以在其自己的任务中处理。

```rust
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

        tokio::spawn(async move {
            // 并发处理每个套接字。
            process(socket).await
        });
    }
}
```

**示例：等待结果**

`spawn` 返回的 `JoinHandle` 是一个 future，它会解析为所生成任务的输出。你可以对其使用 `.await` 来获取结果。

```rust
use tokio::task;

async fn my_background_op(id: i32) -> String {
    format!("Finished background task {}.", id)
}

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(my_background_op(1));

    // 在此处执行其他工作...

    let result = handle.await.unwrap();
    println!("{}", result);
}
```
如果生成的任务发生 panic，等待其 `JoinHandle` 将返回一个 `JoinError`。


## 处理阻塞操作

异步任务不应执行阻塞操作，因为这会阻止工作线程执行其他任务。Tokio 提供了专门的 API 来安全地运行阻塞代码。

下图说明了不同的生成函数如何与运行时的线程池交互。

```d2
direction: down

"异步上下文（例如，在 `async fn` 内）": {
  shape: cloud
  style.fill: "#E6F7FF"

  "tokio::spawn": {
    "在运行时的调度器上生成一个新的 `Send` future。": {
      "立即返回一个 `JoinHandle`。": "任务并发运行。"
    }
  }

  "tokio::spawn_blocking": {
    "在一个独立的专用线程池上生成一个阻塞函数。": {
      "立即返回一个 `JoinHandle`。": "不会阻塞异步工作线程。"
    }
  }

  "tokio::block_in_place": {
    "暂停当前异步任务并将工作线程转换为阻塞状态。": {
      "运行时可能会生成一个新的工作线程来运行其他任务。": "函数在*同一个*线程上运行。"
    }
  }
}

"Tokio 运行时": {
  "调度器线程（工作线程池）": {
    style.fill: "#D1F0E0"
  }
  "阻塞线程池": {
    style.fill: "#FFF0E6"
  }
}

"异步上下文（例如，在 `async fn` 内）"."tokio::spawn" -> "Tokio 运行时"."调度器线程（工作线程池）": "调度异步任务"
"异步上下文（例如，在 `async fn` 内）"."tokio::spawn_blocking" -> "Tokio 运行时"."阻塞线程池": "调度阻塞任务"
"异步上下文（例如，在 `async fn` 内）"."tokio::block_in_place" -> "Tokio 运行时"."调度器线程（工作线程池）": "向运行时发送信号"
```

| 函数 | 任务类型 | 执行上下文 | 使用场景 |
|---|---|---|---|
| `tokio::spawn` | 异步 (`Future`) | Tokio 工作线程池 | 标准的并发异步操作。 |
| `tokio::spawn_blocking` | 同步 (`FnOnce`) | 专用的阻塞线程池 | 长时间运行的 CPU 密集型工作或同步 I/O。 |
| `tokio::block_in_place` | 同步 (`FnOnce`) | *当前*的 Tokio 工作线程 | 在多线程运行时中，异步代码内短暂且不可避免的阻塞调用。 |

### `tokio::spawn_blocking`

在一个允许阻塞的线程上运行一个闭包。这是处理 CPU 密集型工作或同步 I/O 的推荐方法。

**何时使用：** 用于那些否则会长时间阻塞异步工作线程的计算。

**签名**
```rust
fn spawn_blocking<F, R>(f: F) -> JoinHandle<R>
where
    F: FnOnce() -> R + Send + 'static,
    R: Send + 'static,
```

**示例**
```rust
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut v = "Hello, ".to_string();
    let res = task::spawn_blocking(move || {
        // 这是计算密集型工作的替代
        v.push_str("world");
        v
    }).await?;

    assert_eq!(res.as_str(), "Hello, world");
    Ok(())
}
```

### `tokio::block_in_place`

将当前工作线程转换为阻塞线程，允许运行时将其他任务移交给新的工作线程。这比 `spawn_blocking` 更高效，因为它避免了上下文切换，但它仅在多线程运行时上可用。

**何时使用：** 用于必须在当前线程上运行的短暂阻塞操作。

**签名**
```rust
fn block_in_place<F, R>(f: F) -> R
where
    F: FnOnce() -> R,
```

**示例**
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

**注意：** 如果从 `current_thread` 运行时调用此函数，将会发生 panic。

## 让出执行权

### `tokio::yield_now`

将执行权交还给 Tokio 调度器，允许其他任务运行。当前任务会被重新排队，并在稍后再次被轮询。

**何时使用：** 为了给其他任务运行的机会，尤其是在没有自然 `.await` 点的长时间运行任务中。

**签名**
```rust
async fn yield_now()
```

**示例**
```rust
use tokio::task;

#[tokio::main]
async fn main() {
    task::spawn(async {
        println!("spawned task done!")
    });

    // 让出执行权，允许新生成的任务先执行。
    task::yield_now().await;
    println!("main task done!");
}
```

## 管理任务集合

### `JoinSet<T>`

`JoinSet` 是一个任务集合，可以在任务完成时对其进行等待。它对于管理动态数量的子任务非常有用。

**何时使用：** 当你需要生成多个任务并按其完成顺序（而非生成顺序）处理结果时。

**示例**

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

关键方法包括：
- `spawn()`: 向集合中添加一个新任务。
- `join_next()`: 等待集合中的下一个任务完成。
- `abort_all()`: 中止集合中的所有任务。
- `shutdown()`: 中止所有任务并等待它们完成。


## 处理 `!Send` Future

标准的 `tokio::spawn` 要求 future 是 `Send` 的，这意味着它们可以安全地在线程之间移动。对于 `!Send` 的 future（例如，持有 `Rc<T>` 的 future），你必须使用 `LocalSet`。

### `LocalSet` 和 `spawn_local`

`LocalSet` 在当前线程上运行一组任务，允许使用 `spawn_local` 生成 `!Send` 的 future。

**何时使用：** 当你需要处理无法跨线程发送的数据类型时，例如 `Rc` 或来自单线程库的某些类型。

**示例**

```rust
use std::rc::Rc;
use tokio::task;

#[tokio::main]
async fn main() {
    let nonsend_data = Rc::new("my nonsend data...");

    // 构建一个可以运行 `!Send` future 的本地任务集。
    let local = task::LocalSet::new();

    // 运行本地任务集，直到 future 完成。
    local.run_until(async move {
        let nonsend_data = nonsend_data.clone();
        // `spawn_local` 确保 future 在当前线程的 LocalSet 上运行。
        task::spawn_local(async move {
            println!("{}", nonsend_data);
        }).await.unwrap();
    }).await;
}
```

**注意：** `LocalSet::run_until` 和等待 `LocalSet` 只能在运行时的 `block_on` 调用中直接使用，例如在 `#[tokio::main]` 中。

## 后续步骤

现在你已经了解了如何创建和管理任务，你可能希望协调它们的执行或根据时间调度工作。

<x-cards data-columns="2">
  <x-card data-title="同步原语" data-icon="lucide:git-merge" data-href="/api/sync">
    学习如何使用通道、互斥锁和信号量来管理任务间的共享状态。
  </x-card>
  <x-card data-title="时间" data-icon="lucide:timer" data-href="/api/time">
    探索用于跟踪时间的实用工具，包括休眠、间隔和超时。
  </x-card>
</x-cards>
