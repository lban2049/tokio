# 运行时

Tokio 运行时是驱动异步应用程序的引擎。它为执行任务、处理 I/O 和管理基于时间的事件提供了必要的服务。本节提供了用于手动配置运行时并与之交互的详细 API 参考。

若想在更高层次上理解运行时的角色及其组件，请参阅 [核心概念：运行时](./concepts-runtime.md) 指南。

## Runtime

`Runtime` 结构体是主入口点。它捆绑了 I/O 驱动、任务调度器、计时器以及用于阻塞操作的线程池。大多数应用程序可以使用 `#[tokio::main]` 属性，但你也可以为了更强的控制力而直接创建和管理 `Runtime` 实例。

### 创建运行时

你可以使用 `Runtime::new()` 创建一个具有默认多线程配置的运行时。

```rust
use tokio::runtime::Runtime;

// 使用默认设置创建一个新的运行时
let rt = Runtime::new().unwrap();

// 使用该运行时...
```

对于更高级的配置，例如选择调度器或启用特定驱动，请使用 [`Builder`](#builder)。

### 执行 Future

在运行时上运行 future 的主要方法是 `block_on` 方法。该方法会接管当前线程，将给定的 future 运行至完成，并返回其结果。

```rust
use tokio::runtime::Runtime;

// 创建运行时
let rt = Runtime::new().unwrap();

// 执行一个 future，阻塞当前线程直到其完成
rt.block_on(async {
    println!("Hello from the runtime!");
});
```

### 派生任务

在运行时的上下文中，你可以使用 `spawn` 来派生额外的异步任务以并发运行。这些任务将在运行时的线程池上执行。

```rust
use tokio::runtime::Runtime;

// 创建运行时
let rt = Runtime::new().unwrap();

// 将一个 future 派生到运行时上
rt.spawn(async {
    println!("Now running on a worker thread.");
});

// 若要等待派生的任务，可以阻塞其 JoinHandle
let handle = rt.spawn(async {
    "Task finished"
});

let result = rt.block_on(handle).unwrap();
println!("{}", result);
```

对于阻塞、CPU 密集型或其他长时间运行的同步操作，请使用 `spawn_blocking` 以避免调度器阻塞。

```rust
use tokio::runtime::Runtime;
use std::thread;
use std::time::Duration;

let rt = Runtime::new().unwrap();

rt.spawn_blocking(|| {
    println!("Running a blocking operation...");
    thread::sleep(Duration::from_secs(1));
    println!("Blocking operation complete.");
});
```

### 关闭

丢弃 `Runtime` 实例会启动优雅关闭。它将等待所有已派生的任务完成。如果你需要控制关闭行为，可以使用 `shutdown_timeout` 或 `shutdown_background`。

- **`shutdown_timeout(duration)`**: 最多等待 `duration` 时间让任务停止。超时后，任何剩余的任务及其线程都将被泄露。
- **`shutdown_background()`**: 启动关闭而不阻塞，允许从异步上下文中丢弃运行时。如果阻塞任务仍在运行，这可能会导致资源泄露。

```rust
use tokio::runtime::Runtime;
use std::time::Duration;

let runtime = Runtime::new().unwrap();

runtime.spawn(async {
    // 一些长时间运行的任务
    tokio::time::sleep(Duration::from_secs(10)).await;
});

// 关闭，最多等待 100 毫秒让任务完成。
runtime.shutdown_timeout(Duration::from_millis(100));
```

## Builder

`Builder` 提供了一种在创建 `Runtime` 之前对其进行配置的方法。你可以选择调度器类型、设置工作线程数、启用 I/O 和时间驱动等。

### 创建 Builder

根据所需的调度器，创建 `Builder` 有两个主要入口点：

- **`Builder::new_multi_thread()`**: 为工作窃取、多线程调度器创建一个构建器。这适用于大多数应用程序。
- **`Builder::new_current_thread()`**: 为单线程调度器创建一个构建器，该调度器在当前线程上运行所有任务。

### 配置

以下是创建自定义多线程运行时的示例：

```rust
use tokio::runtime::Builder;
use std::time::Duration;

let runtime = Builder::new_multi_thread()
    .worker_threads(4) // 设置工作线程的数量
    .thread_name("my-tokio-worker") // 为线程设置名称
    .thread_stack_size(3 * 1024 * 1024) // 设置栈大小
    .enable_all() // 同时启用 I/O 和时间驱动
    .build()
    .unwrap();

runtime.block_on(async {
    println!("Hello from a custom runtime!");
});
```

**常用配置方法：**

| 方法 | 描述 |
|---|---|
| `enable_all()` | 同时启用 I/O 和时间驱动。一个方便的简写。 |
| `enable_io()` | 为网络、进程和信号启用 I/O 驱动。 |
| `enable_time()` | 为 `tokio::time` 工具启用时间驱动。 |
| `worker_threads(usize)` | 为多线程调度器设置工作线程的数量。 |
| `max_blocking_threads(usize)` | 为阻塞池设置最大线程数。 |
| `thread_name(String)` | 为派生的工作线程的名称设置前缀。 |
| `thread_keep_alive(Duration)` | 为阻塞池线程设置空闲超时时间。 |

## Handle

`Handle` 是对一个活动 `Runtime` 的轻量级、可克隆引用。它允许你从任何上下文（包括其他线程）与运行时进行交互，例如派生任务。

### 获取 Handle

- **`Runtime::handle()`**: 从现有的 `Runtime` 实例获取一个 Handle。
- **`Handle::current()`**: 获取当前执行上下文的运行时的 Handle。如果在 Tokio 运行时上下文之外调用，此函数会 panic。
- **`Handle::try_current()`**: `current()` 的一个非 panic 版本，它返回一个 `Result`。

```rust
use tokio::runtime::{Handle, Runtime};

let rt = Runtime::new().unwrap();

// 从运行时实例获取一个 Handle
let handle_from_rt = rt.handle();

rt.block_on(async {
    // 从当前上下文获取一个 Handle
    let handle_from_ctx = Handle::current();
    
    handle_from_ctx.spawn(async {
        println!("Task spawned from a handle!");
    });
});
```

### 使用 Handle

`Handle` 可用于派生任务、运行阻塞 future 以及进入运行时上下文，即使从标准的 `std::thread` 中也可以。

```rust
use tokio::runtime::{Handle, Runtime};
use std::thread;

#[tokio::main]
async fn main() {
    let handle = Handle::current();

    let std_thread = thread::spawn(move || {
        // 使用 handle 从另一个线程在运行时上运行一个异步块
        handle.block_on(async {
            println!("Hello from another thread!");
        });
    });

    std_thread.join().unwrap();
}
```

### 进入运行时上下文

The `enter()` method on both `Runtime` and `Handle` returns an `EnterGuard`. While the guard is in scope, the current thread is considered to be within that runtime's context. This allows functions like `tokio::spawn` to work without needing an explicit `Handle`.

```rust
use tokio::runtime::Runtime;

fn function_that_spawns() {
    // 如果不在运行时上下文中，这会 panic
    tokio::spawn(async {
        println!("Spawned without an explicit handle.");
    });
}

let rt = Runtime::new().unwrap();

// 进入运行时上下文
let _guard = rt.enter();

// 现在我们可以调用隐式依赖于运行时上下文的函数了
function_that_spawns();
```

## RuntimeFlavor

Tokio 支持两种调度器类型。你可以使用 `runtime_flavor()` 方法来确定一个 `Handle` 关联的调度器类型，该方法会返回一个 `RuntimeFlavor` 枚举。

- `RuntimeFlavor::CurrentThread`: 单线程调度器。
- `RuntimeFlavor::MultiThread`: 多线程、工作窃取调度器。

```rust
use tokio::runtime::{Handle, RuntimeFlavor};

#[tokio::main(flavor = "multi_thread")]
async fn main() {
  assert_eq!(RuntimeFlavor::MultiThread, Handle::current().runtime_flavor());
}
```