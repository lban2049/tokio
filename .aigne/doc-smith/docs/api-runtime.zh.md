# 运行时

Tokio 运行时是驱动异步应用程序的引擎。它提供 I/O 事件循环、任务调度器和计时器等基本服务。虽然许多应用程序可以直接使用 `#[tokio::main]` 宏来获取默认运行时，但当需要更多控制时，此模块提供了手动配置和交互的工具。

有关这些组件如何协同工作的更深入概念性概述，请参阅我们的指南 [运行时](./concepts-runtime.md)。

本页介绍了用于手动管理运行时的主要类型：

<x-cards>
  <x-card data-title="Runtime" data-icon="lucide:cpu">
    主要的 Tokio 运行时实例，捆绑了调度器、I/O 驱动程序和计时器。您可以创建一个来执行异步代码。
  </x-card>
  <x-card data-title="Builder" data-icon="lucide:settings-2">
    一种用于构建具有自定义配置的 `Runtime` 的工具，可配置线程、驱动程序、调度器行为等。
  </x-card>
  <x-card data-title="Handle" data-icon="lucide:grip">
    运行时的轻量级、可克隆的句柄。它允许您从其他线程或同步代码中派生任务或进入运行时上下文。
  </x-card>
</x-cards>

## Runtime

`Runtime` 结构体是执行异步代码的主要入口点。它捆绑了所有必要的服务并管理其生命周期。

### 创建 Runtime

您可以使用默认的多线程配置创建一个运行时，或使用 `Builder` 进行自定义。

#### `new()`

使用默认值创建一个新的 `Runtime` 实例。这将初始化一个多线程调度器，并启用 I/O 和时间驱动程序。

```rust icon=logos:rust 创建一个默认的 Runtime
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建运行时
    let rt = Runtime::new()?;

    // 使用运行时...
    rt.block_on(async {
        println!("Hello from the Tokio runtime!");
    });

    Ok(())
}
```

### 运行 Future

#### `block_on<F: Future>(&self, future: F) -> F::Output`

在 Tokio 运行时上运行一个 future 直至完成，并阻塞当前线程直至其结束。这是从同步上下文开始执行异步代码的主要方式。

不得在异步上下文中调用此方法。

```rust icon=logos:rust 使用 block_on
use tokio::runtime::Runtime;

// 创建运行时
let rt  = Runtime::new().unwrap();

// 执行 future，阻塞当前线程直至完成
rt.block_on(async {
    println!("hello");
});
```

### 派生任务

任务是轻量级的非阻塞执行单元。派生任务使其可以在运行时的线程池上并发运行。

#### `spawn<F>(&self, future: F) -> JoinHandle<F::Output>`

派生一个新的异步任务，并为其返回一个 `JoinHandle`。派生的 future 必须是 `Send` 和 `'static`。

```rust icon=logos:rust 派生一个任务
use tokio::runtime::Runtime;
use std::time::Duration;

let rt = Runtime::new().unwrap();

rt.block_on(async {
    let handle = rt.spawn(async {
        tokio::time::sleep(Duration::from_secs(1)).await;
        "done"
    });

    // 任务运行时执行其他工作...
    println!("Spawned task in the background");

    let result = handle.await.unwrap();
    assert_eq!(result, "done");
});
```

#### `spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>`

在专用于阻塞操作的独立线程池上运行一个阻塞函数。这对于防止长时间运行的同步代码阻塞异步任务调度器至关重要。

```rust icon=logos:rust 派生一个阻塞任务
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();

rt.block_on(async {
    let handle = rt.spawn_blocking(|| {
        // 这是一个阻塞操作
        std::thread::sleep(std::time::Duration::from_secs(1));
        "blocking operation complete"
    });

    let result = handle.await.unwrap();
    println!("{}", result);
});
```

### 管理运行时

#### `handle(&self) -> &Handle`

返回此运行时的 `Handle` 引用。Handle 可以被克隆并发送到其他线程。

```rust icon=logos:rust 获取一个 handle
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
let handle = rt.handle();

// 使用 handle 派生任务
handle.spawn(async {
    println!("Task spawned via handle!");
});
```

#### `enter(&self) -> EnterGuard<'_>`

进入运行时上下文。这允许同步上下文中的代码使用上下文感知函数，如 `tokio::spawn`。

```rust icon=logos:rust 进入运行时上下文
use tokio::runtime::Runtime;
use tokio::task::JoinHandle;

fn function_that_spawns(msg: String) -> JoinHandle<()> {
    // 如果我们没有在下面使用 rt.enter，这里会发生 panic。
    tokio::spawn(async move {
        println!("{}", msg);
    })
}

let rt = Runtime::new().unwrap();
let s = "Hello World!".to_string();

// 通过进入上下文，我们将 tokio::spawn 与此执行器绑定。
let _guard = rt.enter();
let handle = function_that_spawns(s);

// 在结束前等待任务完成。
rt.block_on(handle).unwrap();
```

### 关闭

丢弃 `Runtime` 实例将启动一个平滑关闭过程，等待已派生的任务完成。要获得更多控制，您可以使用显式的关闭方法。

*   **`shutdown_timeout(duration: Duration)`**：关闭运行时，最多等待 `duration` 时间让所有已派生的工作停止。任何未及时停止的工作都会被泄露。
*   **`shutdown_background()`**：关闭运行时，不等待任何已派生的工作停止。这对于从另一个异步上下文中丢弃运行时非常有用。

## Builder

`Builder` 提供了一种灵活的方式来配置和创建 `Runtime` 实例。

### 创建 Builder

您可以从用于多线程或当前线程调度器的构建器开始。

*   **`new_multi_thread()`**：为多线程、工作窃取调度器创建一个构建器。这是默认选项，适用于大多数应用程序。
*   **`new_current_thread()`**：为单线程调度器创建一个构建器，该调度器在当前线程上运行所有任务。

```rust icon=logos:rust 创建构建器
use tokio::runtime::Builder;

// 一个用于多线程运行时的构建器
let multi_thread_builder = Builder::new_multi_thread();

// 一个用于单线程运行时的构建器
let current_thread_builder = Builder::new_current_thread();
```

### 配置方法

构建器使用链式调用模式来配置运行时。以下是一些最常用的选项：

| 方法 | 描述 |
|---|---|
| `enable_all()` | 启用 I/O 和时间驱动程序。一个方便的简写。 |
| `enable_io()` | 启用 I/O 驱动程序（用于网络、文件系统等）。 |
| `enable_time()` | 启用时间驱动程序（用于 `tokio::time`）。 |
| `worker_threads(val: usize)` | 设置多线程调度器的工作线程数。 |
| `max_blocking_threads(val: usize)` | 设置阻塞任务池的最大线程数。 |
| `thread_name(val: impl Into<String>)` | 为运行时派生的所有线程设置一个静态名称。 |
| `thread_name_fn<F>(f: F)` | 设置一个函数来动态生成派生线程的名称。 |
| `thread_stack_size(val: usize)` | 设置派生线程的堆栈大小。 |
| `on_thread_start<F>(f: F)` | 在每个工作线程启动后执行一个函数。 |
| `on_thread_stop<F>(f: F)` | 在每个工作线程停止前执行一个函数。 |

### 构建 Runtime

配置构建器后，调用 `build()` 来创建 `Runtime` 实例。

```rust icon=logos:rust 构建一个自定义运行时
use tokio::runtime::Builder;
use std::time::Duration;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("my-tokio-worker")
        .thread_stack_size(3 * 1024 * 1024)
        .enable_all()
        .build()?;

    runtime.block_on(async {
        println!("Running on a custom-configured runtime!");
    });

    Ok(())
}
```

## Handle

A `Handle` 是一个指向 `Runtime` 的轻量级、可克隆、引用计数的指针。它允许您从任何可以访问该句柄的代码（包括其他线程）与运行时进行交互（例如，派生任务）。

### 获取 Handle

获取句柄主要有两种方式：

*   **`Runtime::handle()`**：从现有的 `Runtime` 实例中获取句柄。
*   **`Handle::current()`**：获取当前执行上下文的运行时句柄。**如果在 Tokio 运行时上下文之外调用，将会发生 panic。**
*   **`Handle::try_current()`**：`Handle::current()` 的非 panic 版本，返回一个 `Result`。

```rust icon=logos:rust 获取一个 handle
use tokio::runtime::Handle;

#[tokio::main]
async fn main () {
    // 这是可以的，因为我们在 #[tokio::main] 上下文中。
    let handle = Handle::current();
    
    std::thread::spawn(move || {
        // 我们可以在另一个线程中使用移动后的 handle。
        let _guard = handle.enter();
        
        // 现在这也是可以的。
        let handle2 = Handle::current();
        println!("Got handle in another thread.");
    }).join().unwrap();
}
```

### 使用 Handle

A `Handle` 提供了许多与 `Runtime` 相同的方法，例如 `spawn`、`spawn_blocking`、`block_on` 和 `enter`。

对于 `block_on` 存在一个重要的区别：当在 `current_thread` 运行时的句柄上使用时，它无法驱动 I/O 或计时器驱动程序。只有原始的 `Runtime::block_on` 调用可以做到这一点。这意味着除非另一个线程正在主动驱动运行时，否则计时器或 I/O 可能无法按预期工作。

### 内省

`Handle` 提供了检查其所属运行时的方法。

*   **`runtime_flavor()`**：返回一个 `RuntimeFlavor` 枚举（`CurrentThread` 或 `MultiThread`）。
*   **`metrics()`**：返回一个 `RuntimeMetrics` 视图以获取性能信息。

## 相关类型

<x-cards data-columns="2">
  <x-card data-title="RuntimeFlavor" data-icon="lucide:git-branch">
    一个枚举，指示运行时是使用 `CurrentThread` 还是 `MultiThread` 调度器。
  </x-card>
  <x-card data-title="UnhandledPanic (unstable)" data-icon="lucide:shield-alert">
    配置当派生的任务发生 panic 时运行时的响应方式。默认为 `Ignore`，但可以设置为 `ShutdownRuntime`。
  </x-card>
</x-cards>