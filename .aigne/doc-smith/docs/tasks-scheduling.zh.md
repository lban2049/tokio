# 任务与调度

Tokio 中的异步程序是围绕任务构建的。任务是一个轻量级、非阻塞的执行单元，类似于 Go 中的 goroutine 或 Kotlin 中的协程。与由操作系统调度器管理的传统线程不同，任务由 Tokio 运行时管理，这使得创建和切换任务极为高效。

这种协作式、非阻塞模型允许少量线程处理海量的并发操作。为了帮助你构建复杂的应用程序，Tokio 提供了一套全面的工具集，用于派生、管理和协调这些任务。

在以下各节中探索 Tokio 并发模型的核心概念：

<x-cards data-columns="2">
  <x-card data-title="派生和管理任务" data-icon="lucide:play-circle" data-href="/tasks-scheduling/spawning">
    学习如何创建和管理并发任务。本节内容涵盖使用 `tokio::spawn` 派生任务、使用 `JoinHandle` 等待其结果，以及如何在不阻塞运行时的情况下安全地运行阻塞代码。
  </x-card>
  <x-card data-title="同步原语" data-icon="lucide:git-merge" data-href="/tasks-scheduling/synchronization">
    协调独立任务并安全地共享数据。探索通过通道（mpsc、oneshot、broadcast、watch）进行消息传递，以及使用 Mutex、Semaphore 和 Barrier 等原语进行状态同步。
  </x-card>
  <x-card data-title="时间、延迟和超时" data-icon="lucide:timer" data-href="/tasks-scheduling/time">
    将基于时间的逻辑集成到你的异步应用程序中。本节内容涵盖如何使用 `sleep` 创建延迟、使用 `interval` 按固定周期运行代码，以及使用 `timeout` 强制执行截止时间。
  </x-card>
  <x-card data-title="运行时" data-icon="lucide:cpu" data-href="/tasks-scheduling/runtime">
    了解驱动 Tokio 应用程序的引擎。运行时包含任务调度器、I/O 驱动和计时器。学习如何配置不同的调度器以及管理运行时的生命周期。
  </x-card>
</x-cards>

掌握这些组件是构建快速、可靠且可扩展的网络应用程序的关键。一旦你熟悉了任务的管理方式，就可以继续在实际操作中运用它们。

接下来，让我们探讨如何执行[异步 I/O](./io.md)。