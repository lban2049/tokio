# 运行时

Tokio 运行时是驱动异步应用程序的引擎。它提供了一些基本服务，如 I/O 事件循环、任务调度器、计时器以及用于阻塞操作的线程池。本节提供了用于手动配置运行时并与之交互的详细 API 文档。

关于更高级别的概念性概述，请参阅我们的核心概念指南中的[运行时](./concepts-runtime.md)。

## 核心组件

一个 Tokio 运行时捆绑了几个协同工作的关键组件来执行异步任务。

```d2
direction: down

"Tokio 运行时": {
  shape: package
  grid-columns: 2
  grid-gap: 50

  "任务调度器": {
    shape: rectangle
    "管理并执行异步任务。"
  }

  "I/O 驱动 (Reactor)": {
    shape: rectangle
    "与操作系统交互以实现非阻塞 I/O。"
  }

  "计时器": {
    shape: rectangle
    "提供 `sleep` 和 `interval` 等实用工具。"
  }

  "阻塞池": {
    shape: rectangle
    "用于阻塞操作的专用线程池。"
  }
}

"你的异步代码": {
  shape: rectangle
}

"你的异步代码" -> "Tokio 运行时": "派生任务并使用资源"

"Tokio 运行时"."任务调度器" <-> "Tokio 运行时"."I/O 驱动 (Reactor)": "在 I/O 事件上唤醒任务"
"Tokio 运行时"."任务调度器" <-> "Tokio 运行时"."计时器": "在超时时唤醒任务"
"Tokio 运行时"."任务调度器" -> "Tokio 运行时"."阻塞池": "分流阻塞调用"

```

## 运行时

`Runtime` 结构体是主要入口点。它封装了所有运行时服务。虽然许多应用程序可以依赖 `#[tokio::main]` 宏，但直接创建 `Runtime` 实例可以实现更精细的控制。

### 创建运行时

使用默认设置创建多线程运行时的最简单方法是使用 `Runtime::new()`。

```rust
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 使用默认配置创建一个新的运行时。
    let rt = Runtime::new()?;

    // 使用运行时阻塞一个 future。
    rt.block_on(async {
        println!("你好，来自 Tokio 运行时！");
    });

    Ok(())
}
```

### 核心方法

<x-cards>
  <x-card data-title="block_on" data-icon="lucide:arrow-down-square">
    在运行时上运行一个 future 直至完成，阻塞当前线程直到 future 解析。
  </x-card>
  <x-card data-title="spawn" data-icon="lucide:send">
    派生一个在运行时上执行的新的异步任务。
  </x-card>
  <x-card data-title="spawn_blocking" data-icon="lucide:box">
    在一个专用线程池上运行一个阻塞函数，防止它阻塞异步调度器。
  </x-card>
  <x-card data-title="handle" data-icon="lucide:grip">
    返回一个运行时的 `Handle`，它可以被克隆并发送到其他线程。
  </x-card>
</x-cards>

### 关闭

通过丢弃 `Runtime` 值来关闭运行时。`drop` 的实现将阻塞当前线程，直到所有已派生的工作都已停止。对于非阻塞关闭或带超时的关闭，你可以使用 `shutdown_background()` 或 `shutdown_timeout()`。

```rust
use tokio::runtime::Runtime;
use tokio::task;
use std::thread;
use std::time::Duration;

fn main() {
   let runtime = Runtime::new().unwrap();

   runtime.block_on(async move {
       task::spawn_blocking(move || {
           // 模拟一个长时间运行的阻塞任务
           thread::sleep(Duration::from_secs(10));
       });
   });

   // 使用 100 毫秒超时关闭。阻塞任务将被泄漏。
   runtime.shutdown_timeout(Duration::from_millis(100));
}
```

## 构建器

`Builder` 提供了一种灵活的方式来配置新的 `Runtime` 实例。你可以选择调度器类型、配置工作线程、启用/禁用驱动程序，以及设置生命周期钩子。

### 创建构建器

你可以开始为多线程调度器或当前线程调度器构建运行时。

-   `Builder::new_multi_thread()`: 为工作窃取多线程调度器创建一个构建器。这适用于大多数应用程序。
-   `Builder::new_current_thread()`: 为单线程调度器创建一个构建器，该调度器在当前线程上运行所有任务。

### 配置示例

这是一个创建自定义多线程运行时的示例。

```rust
use tokio::runtime::Builder;
use std::time::Duration;

fn main() {
    let runtime = Builder::new_multi_thread()
        .worker_threads(4) // 设置工作线程的数量
        .thread_name("my-tokio-worker") // 为工作线程设置一个名称
        .thread_stack_size(3 * 1024 * 1024) // 设置工作线程的堆栈大小
        .enable_all() // 同时启用 I/O 和时间驱动程序
        .build() // 构建运行时
        .unwrap();

    runtime.block_on(async {
        println!("正在一个自定义配置的运行时上运行！");
    });
}
```

### 常用配置方法

| Method | Description |
|---|---|
| `enable_all()` | 同时启用 I/O 和时间驱动程序。这是一个方便的简写。 |
| `enable_io()` | 为网络、进程和信号启用 I/O 驱动程序。 |
| `enable_time()` | 为 `sleep`、`interval` 和 `timeout` 等实用工具启用时间驱动程序。 |
| `worker_threads(usize)` | 为多线程调度器设置工作线程的数量。 |
| `max_blocking_threads(usize)` | 设置阻塞池中的最大线程数。 |
| `thread_name(impl Into<String>)` | 为运行时派生的线程设置一个静态名称。 |
| `thread_keep_alive(Duration)` | 为阻塞池中的线程设置自定义的保活超时时间。 |
| `on_thread_start(F)` | 在每个工作线程启动后执行一个函数。 |

## 句柄

`Handle` 是一个可克隆、引用计数的 `Runtime` 句柄。它允许你从任何拥有句柄的线程与运行时进行交互（例如，派生任务），而无需引用 `Runtime` 实例本身。

### 获取句柄

获取 `Handle` 主要有两种方式：

1.  **从现有的 `Runtime` 中获取**：`runtime.handle()`
2.  **从运行时上下文中获取**：`Handle::current()`

```rust
use tokio::runtime::{Handle, Runtime};

fn main() {
    let rt = Runtime::new().unwrap();

    // 1. 从运行时实例获取一个句柄
    let handle_from_rt = rt.handle().clone();

    rt.block_on(async {
        // 2. 从当前运行时上下文获取一个句柄
        let handle_from_context = Handle::current();

        handle_from_context.spawn(async {
            println!("从句柄派生的任务！");
        });
    });
}
```

如果在 Tokio 运行时上下文之外调用 `Handle::current()`，它将会 panic。对于运行时可能不处于活动状态的情况，`Handle::try_current()` 会返回一个 `Result`。

### 使用句柄

`Handle` 提供了与 `Runtime` 类似的方法，用于派生任务和阻塞 future。

-   `handle.spawn(future)`: 在关联的运行时上派生一个任务。
-   `handle.spawn_blocking(f)`: 在运行时的阻塞池上派生一个阻塞任务。
-   `handle.block_on(future)`: 阻塞当前线程并运行一个 future 直至完成。请注意，在 `current_thread` 运行时上，此方法无法驱动 I/O 或计时器；只有 `Runtime::block_on` 可以。
-   `handle.enter()`: 进入运行时上下文，返回一个 `EnterGuard`。当需要在 `async` 块之外创建基于 I/O 或计时器的类型时，这是必需的。

```rust
use tokio::runtime::{Handle, Runtime};
use tokio::task::JoinHandle;
use tokio::time::{sleep, Duration};

// 此函数需要一个运行时上下文来派生任务。
fn function_that_spawns(msg: String) -> JoinHandle<()> {
    tokio::spawn(async move {
        println!("{}", msg);
        sleep(Duration::from_millis(10)).await;
    })
}

fn main() {
    let rt = Runtime::new().unwrap();

    let s = "来自运行时上下文之外的问候！".to_string();

    // 进入运行时上下文以调用 `tokio::spawn`。
    let _guard = rt.enter();
    let handle = function_that_spawns(s);

    // 在句柄上阻塞以等待任务完成。
    rt.block_on(handle).unwrap();
}
```

### RuntimeFlavor

你可以通过 `handle.runtime_flavor()` 来确定运行时正在使用的调度器类型。

```rust
use tokio::runtime::{Handle, RuntimeFlavor};

#[tokio::main(flavor = "current_thread")]
async fn main() {
  assert_eq!(RuntimeFlavor::CurrentThread, Handle::current().runtime_flavor());
}
```
