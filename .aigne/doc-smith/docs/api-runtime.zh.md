# Runtime

Tokio 运行时提供了运行异步任务所需的 I/O 驱动、任务调度器、计时器和阻塞池。它将这些服务捆绑到单一类型中，从而可以统一地启动、关闭和配置它们。

对于大多数应用程序而言，`#[tokio::main]` 属性宏是最简单的入门方式，因为它会自动创建和管理一个 `Runtime`。
然而，若需进行更高级的配置，你可以使用 `Builder` 手动构建 `Runtime` 实例。

若想深入了解运行时的角色和架构，请参阅 [运行时概念页面](./concepts-runtime.md)。

## Runtime

`Runtime` 结构体是 Tokio 运行时的主要入口点。它封装了执行异步代码所需的所有组件。

```rust
use tokio::runtime::Runtime;

// 使用默认配置创建一个新的运行时
let rt = Runtime::new().unwrap();

// 使用运行时在一个 future 上进行阻塞
rt.block_on(async {
    println!("Hello from the runtime!");
});
```

### Shutdown

通过丢弃 `Runtime` 值，或调用 `shutdown_background` 或 `shutdown_timeout` 方法，可以关闭运行时。当一个运行时被丢弃时，发起关闭的线程会阻塞，直到所有已衍生的任务全部停止。这个过程可能会花费不确定的时间。`shutdown_timeout` 方法允许指定一个最长等待时间。

### Sharing

`Runtime` 的访问权限可以通过以下几种方式在线程间共享：
- **`Arc<Runtime>`**：一个共享指针，只要引用存在，它就会阻止运行时关闭。
- **`Handle`**：一个轻量级、可克隆的句柄，它允许衍生任务和进入运行时上下文，而不会阻止运行时关闭。更多详情请参阅下面的 `Handle` 部分。
- **进入上下文**：`enter` 方法提供一个上下文守卫，允许像 `tokio::spawn` 这样的 Tokio 函数在其作用域内工作。

### Methods

#### new() -> Result<Runtime, io::Error>
创建一个新的 `Runtime` 实例，该实例带有默认的多线程调度器并启用了所有驱动程序。

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
```

#### handle(&self) -> &Handle
返回运行时的句柄，该句柄可以被克隆并发送到其他线程以与运行时交互。

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
let handle = rt.handle();
```

#### spawn<F>(&self, future: F) -> JoinHandle<F::Output>
将一个 future 衍生到运行时上。该 future 必须是 `Send`。

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
rt.spawn(async {
    println!("Task running on the runtime.");
});
```

#### spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>
在专用于阻塞操作的线程池上运行一个阻塞函数。

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
rt.spawn_blocking(|| {
    // 这是一个阻塞操作
    std::thread::sleep(std::time::Duration::from_secs(1));
    println!("Blocking task complete.");
});
```

#### block_on<F: Future>(&self, future: F) -> F::Output>
在当前线程上运行一个 future 直至其完成，并阻塞直到它结束。这是运行时的主要入口点。

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
let result = rt.block_on(async {
    42
});
assert_eq!(result, 42);
```

#### enter(&self) -> EnterGuard<'_>
进入运行时上下文。当返回的 `EnterGuard` 在作用域内时，像 `tokio::spawn` 这样的函数将使用此运行时。

```rust
use tokio::runtime::Runtime;

fn some_function() {
    // 如果不在运行时上下文中，这会引发 panic
    tokio::spawn(async { println!("Spawned!"); });
}

let rt = Runtime::new().unwrap();
let _guard = rt.enter(); // 进入上下文
some_function();
// Guard 被丢弃，上下文退出
```

#### shutdown_timeout(self, duration: Duration)
关闭运行时，最多等待 `duration` 时间让所有衍生的任务停止。任何未及时停止的任务都将被泄露。

```rust
use tokio::runtime::Runtime;
use std::time::Duration;

let rt = Runtime::new().unwrap();
// ... 衍生任务 ...
rt.shutdown_timeout(Duration::from_millis(100));
```

#### shutdown_background(self)
关闭运行时，不等待任何衍生的任务停止。这对于在另一个异步上下文中丢弃运行时很有用。

```rust
use tokio::runtime::Runtime;

let rt = Runtime::new().unwrap();
rt.block_on(async {
    let inner_rt = Runtime::new().unwrap();
    // ...
    inner_rt.shutdown_background();
});
```

## Builder

`Builder` 提供了一种在创建 `Runtime` 之前对其进行配置的方法。你可以选择调度器类型、配置工作线程、启用或禁用驱动程序以及设置生命周期钩子。

```rust
use tokio::runtime::Builder;

let runtime = Builder::new_multi_thread()
    .worker_threads(4)
    .thread_name("my-runtime-worker")
    .enable_all()
    .build()
    .unwrap();

runtime.block_on(async {
    println!("Hello from a custom-built runtime!");
});
```

### Methods

#### 创建 Builder

| 方法 | 描述 |
|---|---|
| `new_multi_thread()` | 为多线程、工作窃取调度器创建一个构建器。 |
| `new_current_thread()` | 为单线程调度器创建一个构建器。 |

#### 配置线程

| 方法 | 描述 |
|---|---|
| `worker_threads(val: usize)` | 为多线程调度器设置工作线程的数量。如果 `val` 为 0，则会引发 panic。 |
| `max_blocking_threads(val: usize)` | 设置阻塞线程池的最大线程数。默认为 512。 |
| `thread_name(val: impl Into<String>)` | 为运行时衍生的所有线程设置一个静态名称。 |
| `thread_name_fn<F>(f: F)` | 设置一个为每个新线程生成名称的函数。 |
| `thread_stack_size(val: usize)` | 设置工作线程的堆栈大小（以字节为单位）。 |
| `thread_keep_alive(duration: Duration)` | 为阻塞池中的空闲线程设置自定义超时时间。 |

#### 配置驱动和调度器

| 方法 | 描述 |
|---|---|
| `enable_all()` | 同时启用 I/O 和时间驱动程序。 |
| `enable_io()` | 启用 I/O 驱动程序（用于网络、进程、信号）。 |
| `enable_time()` | 启用时间驱动程序（用于 `tokio::time`）。 |
| `event_interval(val: u32)` | 设置调度器检查外部事件（I/O、计时器）的频率。默认为 61 个 tick。 |
| `global_queue_interval(val: u32)` | 设置调度器轮询全局任务队列的频率。 |

#### 生命周期和任务钩子
这些方法允许你在运行时和任务生命周期的不同点执行自定义代码。它们主要用于监控和记账。

| 方法 | 描述 |
|---|---|
| `on_thread_start<F>(f: F)` | 在每个工作线程启动后执行一个函数。 |
| `on_thread_stop<F>(f: F)` | 在每个工作线程停止前执行一个函数。 |
| `on_thread_park<F>(f: F)` | 在工作线程进入空闲状态之前执行一个函数。 |
| `on_thread_unpark<F>(f: F)` | 在工作线程变为活动状态之后执行一个函数。 |

#### 构建 Runtime

| 方法 | 描述 |
|---|---|
| `build() -> io::Result<Runtime>` | 创建已配置的 `Runtime` 实例。 |

## Handle

`Handle` 是对 `Runtime` 的一个轻量级、可克隆的引用。它允许你从任何拥有句柄的线程与运行时进行交互（例如，衍生任务），而无需拥有 `Runtime` 对象本身。

### Methods

#### current() -> Handle
返回当前正在运行的运行时的句柄。如果在 Tokio 运行时上下文之外调用，则会引发 panic。

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::runtime::Handle::current();
    handle.spawn(async { /* ... */ });
}
```

#### try_current() -> Result<Handle, TryCurrentError>
返回当前正在运行的运行时的句柄，如果不在运行时上下文中，则返回错误。此方法不会引发 panic。

```rust
use tokio::runtime::Handle;

if let Ok(handle) = Handle::try_current() {
    println!("正在 Tokio 运行时内部运行。");
} else {
    println!("未在 Tokio 运行时内部运行。");
}
```

#### enter(&self) -> EnterGuard<'_'>
进入与此句柄关联的运行时上下文。更多详情请参阅 `Runtime::enter`。

#### spawn<F>(&self, future: F) -> JoinHandle<F::Output>
将一个 future 衍生到与此句柄关联的运行时上。

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::runtime::Handle::current();
    let join_handle = handle.spawn(async {
        "Hello from a spawned task!"
    });
    let result = join_handle.await.unwrap();
    println!("{}", result);
}
```

#### spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>
在运行时的阻塞线程池上运行一个阻塞函数。

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::runtime::Handle::current();
    let join_handle = handle.spawn_blocking(|| {
        // 阻塞的 I/O 或 CPU 密集型工作
        "done"
    });
    let result = join_handle.await.unwrap();
    assert_eq!(result, "done");
}
```

#### block_on<F: Future>(&self, future: F) -> F::Output>
阻塞当前线程，直到提供的 future 完成。当拥有 `Handle` 时，这对于从同步上下文运行异步代码非常有用。

```rust
use tokio::runtime::Handle;

#[tokio::main]
async fn main() {
    let handle = Handle::current();
    std::thread::spawn(move || {
        // 在一个新的同步线程中使用句柄来阻塞一个异步任务。
        handle.block_on(async {
            println!("在另一个线程中运行异步代码");
        });
    }).join().unwrap();
}
```

#### runtime_flavor(&self) -> RuntimeFlavor
返回运行时的类型，指示它是 `CurrentThread` 还是 `MultiThread` 调度器。

## 其他类型

### RuntimeFlavor
一个指示 `Runtime` 调度策略的枚举。
- `CurrentThread`：一个单线程调度器，在当前线程上运行所有任务。
- `MultiThread`：一个多线程、工作窃取的调度器。

### EnterGuard
由 `Runtime::enter` 和 `Handle::enter` 返回的 RAII 守卫。只要此守卫在作用域内，运行时上下文就处于活动状态。以与创建相反的顺序丢弃守卫非常重要，以避免 panic。

### TryCurrentError
当无法获取当前运行时的句柄时，由 `Handle::try_current` 返回的错误类型。