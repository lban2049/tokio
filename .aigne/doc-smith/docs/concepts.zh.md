# 核心概念

Tokio 是一个事件驱动、非阻塞的 I/O 平台，用于使用 Rust 编写异步应用程序。要构建可靠且高性能的网络应用程序，了解构成 Tokio 运行时的基本组件会很有帮助。本节对这些核心概念进行了高层次的概述，为更高级的使用奠定了坚实的基础。

从高层次来看，Tokio 的架构是围绕几个主要组件构建的，这些组件协同工作以高效地执行你的异步代码。

```d2
direction: down

"Tokio 运行时" {
  shape: package
  grid-columns: 1
  grid-gap: 50

  "核心组件" {
    grid-columns: 2

    "调度器": {
      shape: rectangle
      "管理并执行任务"
    }

    "I/O 驱动": {
      shape: rectangle
      "与操作系统事件（epoll、kqueue、IOCP）交互"
    }
  }

  "面向用户的 API" {
    grid-columns: 2
    grid-gap: 20

    "任务": {
      shape: package
      "spawn()"
      "JoinHandle"
      "spawn_blocking()"
    }

    "异步 I/O": {
      shape: package
      "TCP / UDP"
      "文件系统"
      "信号"
      "进程"
    }

    "同步原语": {
      shape: package
      "通道 (mpsc, oneshot)"
      "Mutex"
      "Barrier"
    }

    "计时器": {
      shape: package
      "sleep()"
      "interval()"
      "timeout()"
    }
  }

  "核心组件".调度器 -> "面向用户的 API".任务: "执行"
  "核心组件"."I/O 驱动" -> "面向用户的 API"."异步 I/O": "驱动"
}

"你的应用程序代码" {
  shape: rectangle
}

"你的应用程序代码" -> "Tokio 运行时"."面向用户的 API": "使用"

```

下面，我们将探讨这些基本构建块。每个卡片都链接到关于特定主题的更详细的指南。

<x-cards data-columns="2">
  <x-card data-title="任务与调度" data-href="/concepts/tasks" data-icon="lucide:workflow">
    Rust 中的异步程序是围绕称为任务的轻量级、非阻塞执行单元构建的。了解如何在 Tokio 运行时上生成、管理和协调这些任务。
  </x-card>
  <x-card data-title="异步 I/O" data-href="/concepts/io" data-icon="lucide:arrow-right-left">
    Tokio 提供了一套用于 I/O 操作的非阻塞 API，包括网络（TCP、UDP）、文件系统访问和进程间通信，所有这些都不会阻塞线程。
  </x-card>
  <x-card data-title="同步" data-href="/concepts/synchronization" data-icon="lucide:lock">
    当任务需要通信或共享数据时，Tokio 提供了一套专为异步世界设计的同步原语，如通道、互斥锁和屏障。
  </x-card>
  <x-card data-title="计时器" data-href="/concepts/timers" data-icon="lucide:timer">
    探索用于跟踪时间和调度未来工作的实用工具。这包括设置超时、休眠一段时间或按特定间隔重复操作。
  </x-card>
  <x-card data-title="运行时" data-href="/concepts/runtime" data-icon="lucide:server">
    运行时是执行异步任务的引擎。深入了解其组件，包括任务调度器、I/O 驱动和计时器，并学习如何根据你的需求进行配置。
  </x-card>
</x-cards>

## 处理阻塞代码

Tokio 通过在少量线程上运行许多任务来实现高并发。该模型依赖于任务在 `.await` 点出让控制权，以便其他任务可以运行。然而，执行长时间运行、CPU 密集型计算或阻塞 I/O 的代码会阻止同一线程上的其他任务运行。

为了处理这种情况，Tokio 提供了一个专用于阻塞操作的线程池。你可以使用 `spawn_blocking` 函数将阻塞代码卸载到该线程池，确保它不会干扰主异步任务调度器。

```rust
#[tokio::main]
async fn main() {
    // 这在核心调度器线程上运行。

    let blocking_task = tokio::task::spawn_blocking(|| {
        // 这在专用的阻塞线程上运行。
        // 在这里执行阻塞操作是可以的。
        std::thread::sleep(std::time::Duration::from_secs(1));
        "done"
    });

    // 我们可以等待阻塞任务完成而无需阻塞调度器。
    let result = blocking_task.await.unwrap();
    assert_eq!(result, "done");
}
```
这种分离对于构建混合了异步和同步代码的响应式应用程序至关重要。

---

有了这个概述，你就了解了 Tokio 的核心架构图。深入研究的最佳起点是任务，因为它们是任何 Tokio 应用程序中的基本执行单元。

接下来，我们建议阅读有关[任务与调度](./concepts-tasks.md)的内容，以了解你的异步代码是如何执行的。