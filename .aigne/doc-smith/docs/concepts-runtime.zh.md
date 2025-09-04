# 运行时

Rust 中的异步应用程序需要一个运行时来执行。Tokio 运行时提供了实现这一点所需的服务，包括：

- 一个 **I/O 事件循环**，通常称为驱动程序，它管理 I/O 资源并在任务准备好继续执行时通知它们。
- 一个**任务调度器**，用于协调许多并发任务的执行。
- 一个**计时器**，用于调度将来某个时间运行的工作。

Tokio 中的 `Runtime` 类型捆绑了所有这些服务，允许它们被一起配置、启动和关闭。虽然你可以手动配置 `Runtime`，但大多数应用程序会从 `#[tokio::main]` 属性宏开始，它能方便地设置一个默认的运行时。

```d2
direction: down

"你的应用程序代码": {
  shape: rectangle

  "main()": {
    label: "async fn main()"
    shape: code
  }

  "tokio::spawn()": {
    label: "tokio::spawn(async { ... })"
    shape: code
  }
}

"Tokio 运行时": {
  shape: package
  style.stroke-dash: 2

  调度器: {
    shape: hexagon
    grid-columns: 2

    "任务队列": {
      shape: queue
    }
    "工作线程": {
      shape: class
    }
  }

  "I/O 驱动 (epoll, kqueue, IOCP)": {
    shape: hexagon
  }

  计时器: {
    shape: hexagon
  }

  "阻塞池": {
    label: "阻塞线程池"
    shape: class
  }
}

"你的应用程序代码"."main()" -> "Tokio 运行时": "在其上执行"
"你的应用程序代码"."tokio::spawn()" -> "Tokio 运行时".调度器."任务队列": "提交任务"
"Tokio 运行时".调度器."工作线程" -> "Tokio 运行时".调度器."任务队列": "拉取任务"
"Tokio 运行时".调度器 <-> "Tokio 运行时"."I/O 驱动 (epoll, kqueue, IOCP)": "轮询事件"
"Tokio 运行时".调度器 <-> "Tokio 运行时".计时器: "调度超时"
"Tokio 运行时".调度器 -> "Tokio 运行时"."阻塞池": "卸载阻塞工作"

```

## 用法

使用 Tokio 运行时主要有两种方式：为简单起见使用 `#[tokio::main]` 宏，或者为了更多控制而手动创建和管理一个 `Runtime` 实例。

### `#[tokio::main]` 宏

对于大多数应用程序，`#[tokio::main]` 属性是入门最简单的方式。它会创建一个默认的多线程运行时，并在此之上运行被修饰的 `async fn main` 函数。

```rust,no_run
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            loop {
                let n = match socket.read(&mut buf).await {
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

### 手动管理运行时

如果你需要自定义运行时的配置，可以直接创建一个 `Runtime` 实例。`block_on` 方法是在运行时上运行一个 future 直至其完成的入口点。

```rust,no_run
use tokio::runtime::Runtime;
use tokio::net::TcpListener;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt  = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
        println!("Listening on: {}", listener.local_addr().unwrap());
        // ... application logic ...
    });
    Ok(())
}
```

## 运行时配置

Tokio 提供不同的调度策略以适应各种应用程序的需求。你可以使用 `Builder` 来选择一个调度器。

<x-cards data-columns="2">
  <x-card data-title="多线程调度器" data-icon="lucide:users">
    在一个工作窃取线程池上执行 future，通常每个 CPU 核心对应一个工作线程。这是默认选项，也是大多数服务器端应用程序的最佳选择。
  </x-card>
  <x-card data-title="当前线程调度器" data-icon="lucide:user">
    一个单线程执行器，在当前线程上运行所有任务。适用于只需要单个线程的场景，或在 `LocalSet` 中派生 `!Send` future 的情况。
  </x-card>
</x-cards>

### 使用 Builder 自定义

`Builder` 提供了对运行时配置的细粒度控制。你可以链式调用方法来自定义线程行为、启用/禁用驱动程序以及设置调度器参数。

以下是一些最常见的配置选项：

| Method | Description |
|---|---|
| `worker_threads(n)` | 为多线程调度器设置工作线程的数量。 |
| `max_blocking_threads(n)` | 为阻塞操作派生的线程设置上限。 |
| `thread_name("name")` | 为运行时派生的线程设置名称。 |
| `thread_stack_size(bytes)` | 为工作线程设置栈大小。 |
| `enable_all()` | 同时启用 I/O 和时间驱动程序。 |
| `enable_io()` | 为网络、进程和信号启用 I/O 驱动程序。 |
| `enable_time()` | 为 `tokio::time` 工具启用时间驱动程序。 |

**示例：构建一个自定义运行时**

```rust
use tokio::runtime::Builder;

fn main() {
    // 构建一个拥有 4 个工作线程、自定义线程名
    // 和更大栈大小的运行时。
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("my-tokio-worker")
        .thread_stack_size(3 * 1024 * 1024)
        .enable_all()
        .build()
        .unwrap();

    runtime.block_on(async {
        println!("Hello from the custom runtime!");
    });
}
```

## 运行时关闭

通过丢弃 `Runtime` 实例来关闭运行时。理解关闭时的行为很重要：

- 发起关闭的线程会阻塞，直到所有已派生的工作都已停止。
- **异步任务** (`spawn`) 会一直运行直到它们让出（yield），此时它们将被丢弃。不保证它们会运行到完成。
- **阻塞任务** (`spawn_blocking`) 被允许一直运行直到它们返回。

因为默认的丢弃行为可能会无限期阻塞，Tokio 提供了替代的关闭方法：

- `shutdown_timeout(duration)`：等待指定的一段时间让工作完成。如果达到超时，任何剩余的工作和运行它们的线程都会被泄漏，并且关闭调用会解除阻塞。
- `shutdown_background()`：`shutdown_timeout(Duration::from_nanos(0))` 的简写。它会启动关闭并立即返回，不会等待任何工作停止。这对于从另一个异步上下文中丢弃运行时很有用。

---

在对运行时有了扎实的理解之后，你现在可以探索可用的详细配置选项和指标了。

要获取运行时功能和构建器选项的完整列表，请参阅 [运行时 API 参考](./api-runtime.md)。