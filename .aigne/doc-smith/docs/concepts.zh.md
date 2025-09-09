# 核心概念

Tokio 是一个事件驱动、非阻塞的 I/O 平台，用于使用 Rust 编写异步应用程序。要高效地使用它，理解其基本组件至关重要。本节将宏观地概述构成 Tokio 基础的核心概念，为构建高级、高性能的应用程序奠定坚实的基础。

每个概念都有专门的页面进行更深入的探讨。我们建议按顺序阅读，以循序渐进地加深理解。

<x-cards data-columns="2">
  <x-card data-title="任务与调度" data-icon="lucide:box" data-href="/concepts/tasks">
    Rust 中的异步程序是围绕称为任务的轻量级、非阻塞执行单元构建的。了解 Tokio 如何生成、运行和管理这些任务。
  </x-card>
  <x-card data-title="异步 I/O" data-icon="lucide:arrow-right-left" data-href="/concepts/io">
    了解 Tokio 用于网络（TCP、UDP）、文件系统操作以及与操作系统异步交互的非阻塞 I/O 原语。
  </x-card>
  <x-card data-title="同步" data-icon="lucide:link" data-href="/concepts/synchronization">
    探索用于在任务之间通信和共享数据的原语，包括通道（oneshot、mpsc、watch）、互斥锁和屏障。
  </x-card>
  <x-card data-title="计时器" data-icon="lucide:timer" data-href="/concepts/timers">
    了解用于跟踪时间和调度工作的实用工具，例如设置超时、休眠或按间隔重复操作。
  </x-card>
  <x-card data-title="运行时" data-icon="lucide:cpu" data-href="/concepts/runtime">
    深入了解执行异步代码的引擎，它包含任务调度器、I/O 驱动程序和高性能计时器。
  </x-card>
</x-cards>

## CPU 密集型任务和阻塞代码

Tokio 擅长处理 I/O 密集型任务，它能在少量线程上并发运行大量此类任务。这是因为 I/O 密集型任务在等待 I/O 时会将控制权交还给调度器，从而允许其他任务运行。然而，执行耗时较长的 CPU 密集型计算且不进行 await 的代码会阻塞线程，导致其他任务无法运行。

为应对这种情况，Tokio 为阻塞操作提供了一个专用的线程池。你可以使用 `spawn_blocking` 函数来运行阻塞代码，例如 CPU 密集型计算或与阻塞文件 I/O 的交互。这会将工作移至一个独立的线程，从而确保主运行时不会被阻塞。

```rust main.rs icon=logos:rust
#[tokio::main]
async fn main() {
    // 这在核心线程上运行。

    let blocking_task = tokio::task::spawn_blocking(|| {
        // 这在阻塞线程上运行。
        // 在这里阻塞是可以的。
        // 例如，一个 CPU 密集型计算：
        let mut sum = 0;
        for i in 0..1_000_000_000 {
            sum += i;
        }
        sum
    });

    // 我们可以等待阻塞任务完成。
    let result = blocking_task.await.unwrap();
    println!("Blocking task finished with result: {}", result);
}
```

对于管理 CPU 密集型任务池，可以考虑使用像 [Rayon](https://docs.rs/rayon) 这样的专用库。你可以通过 `oneshot` 通道将 Rayon 任务的结果发送回异步的 Tokio 任务，从而实现与 Tokio 的集成。

---

现在你已经对核心组件有了大致了解，下一步的重点是深入探究 Tokio 如何管理并发操作。

**下一步**：[任务与调度](./concepts-tasks.md)