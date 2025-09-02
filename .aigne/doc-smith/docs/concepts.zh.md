# 核心概念

Tokio 是一个用于使用 Rust 编程语言编写异步应用程序的事件驱动、非阻塞 I/O 平台。要构建快速且可靠的应用程序，理解构成 Tokio 运行时的基本组件会很有帮助。本节将探讨这些核心概念，为高级用法奠定坚实的基础。

从宏观上看，Tokio 应用程序围绕一个执行异步任务的运行时构建。该运行时包括一个任务调度器、一个与操作系统事件队列（如 epoll、kqueue 或 IOCP）交互的 I/O 驱动程序，以及一个高性能的计时器。

```d2
direction: down

"Tokio 应用程序": {
  "运行时": {
    "调度器": {
      "任务 1": "async fn"
      "任务 2": "async fn"
      "任务 N": "..."
    }

    "I/O 驱动程序": {
      style.fill: "#B6DDF6"
      "操作系统事件 (epoll, kqueue, IOCP)"
    }

    "计时器": {
      style.fill: "#B6DDF6"
    }
  }
}

"运行时.调度器.任务 1" -> "运行时.I/O 驱动程序": "异步 I/O (例如 net, fs)" { style.animated: true }
"运行时.I/O 驱动程序" -> "运行时.调度器.任务 1": "在 I/O 就绪时唤醒任务" { style.animated: true }

"运行时.调度器.任务 2" -> "运行时.计时器": "请求休眠/超时" { style.animated: true }
"运行时.计时器" -> "运行时.调度器.任务 2": "当时间到达时唤醒任务" { style.animated: true }

"运行时.调度器.任务 1" <-> "运行时.调度器.任务 2": "同步 (例如 channel, mutex)" { style.stroke-dash: 4 }
```

要更好地理解这些组件如何协同工作，请详细探索以下核心概念。

<x-cards data-columns="2">
  <x-card data-title="任务与调度" data-icon="lucide:workflow" data-href="/concepts/tasks">
    Rust 中的异步程序是围绕称为任务的轻量级、非阻塞执行单元构建的。学习如何派生、等待和管理任务，并了解 Tokio 的调度器如何高效地执行它们。
  </x-card>
  <x-card data-title="异步 I/O" data-icon="lucide:arrow-right-left" data-href="/concepts/io">
    Tokio 提供了一套用于 I/O 操作的非阻塞 API，包括网络（TCP、UDP、UDS）、文件系统访问和进程间通信，所有这些都建立在 `AsyncRead` 和 `AsyncWrite` 这两个 trait 之上。
  </x-card>
  <x-card data-title="同步" data-icon="lucide:lock" data-href="/concepts/synchronization">
    当任务需要通信或共享数据时，可以使用 Tokio 的同步原语。这些原语包括通道（oneshot、mpsc、watch、broadcast）和像 `Mutex` 这样的并发工具。
  </x-card>
  <x-card data-title="计时器" data-icon="lucide:timer" data-href="/concepts/timers">
    管理异步代码中基于时间的操作。Tokio 提供了用于创建延迟 (`sleep`)、按固定间隔执行代码 (`interval`) 以及对操作强制执行时间限制 (`timeout`) 的实用工具。
  </x-card>
  <x-card data-title="运行时" data-icon="lucide:server" data-href="/concepts/runtime">
    运行时是驱动异步应用程序的引擎。深入了解其组件，包括多线程和当前线程调度器，并学习如何根据特定需求进行配置。
  </x-card>
</x-cards>

掌握了这些基础知识，你就能更好地编写更复杂的异步应用程序。最好的起点是深入了解 Tokio 如何管理异步工作。

[下一篇：任务与调度](./concepts-tasks.md)
