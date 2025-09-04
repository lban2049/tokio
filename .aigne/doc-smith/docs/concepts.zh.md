# 核心概念

Tokio 是一个用于使用 Rust 编写异步应用程序的事件驱动、非阻塞 I/O 平台。为了有效地使用 Tokio，理解其基本组件至关重要。本节对构成 Tokio 运行时基础的关键概念进行了高层次的概述，为您构建高级应用程序奠定了坚实的基础。

从高层次来看，Tokio 由几个协同工作的关键部分组成：

```d2
direction: down

"Tokio-Runtime": {
  label: "Tokio 运行时"
  shape: package
  grid-columns: 1
  grid-gap: 50

  "Core-Components": {
    label: "核心组件"
    shape: rectangle
    grid-columns: 3

    Scheduler: {
      label: "任务调度器"
    }
    "I-O-Driver": {
      label: "I/O 驱动\n(epoll, kqueue, IOCP)"
    }
    Timer: {
      label: "高性能计时器"
    }
  }

  "User-Code": {
    label: "用户代码"
    shape: rectangle
    grid-columns: 2

    "Async-Task-1": {
      label: "异步任务 1"
    }
    "Async-Task-2": {
      label: "异步任务 2"
    }
    "...": {}
    "Async-Task-N": {
      label: "异步任务 N"
    }
  }

  "OS-Resources": {
    label: "操作系统资源"
    shape: rectangle
    grid-columns: 2

    "TCP-Socket": { 
      label: "TCP 套接字"
      shape: cylinder 
    }
    File: { 
      shape: cylinder 
    }
    "UDP-Socket": { 
      label: "UDP 套接字"
      shape: cylinder 
    }
    Process: { 
      shape: cylinder 
    }
  }

  "Core-Components".Scheduler -> "User-Code": "执行"
  "User-Code"."Async-Task-1" -> "OS-Resources"."TCP-Socket": "执行 I/O"
  "OS-Resources" -> "Core-Components"."I-O-Driver": "注册到"
  "Core-Components"."I-O-Driver" -> "Core-Components".Scheduler: "通知就绪"
}
```

详细了解以下基本概念：

<x-cards data-columns="2">
  <x-card data-title="任务与调度" data-icon="lucide:box" data-href="/concepts/tasks">
    了解 Tokio 中的基本执行单元：异步任务。这包括如何派生新任务、等待其结果，以及在不暂停整个运行时的情况下管理阻塞或 CPU 密集型操作。
  </x-card>
  <x-card data-title="异步 I/O" data-icon="lucide:arrow-right-left" data-href="/concepts/io">
    探索 Tokio 用于 I/O 操作的非阻塞原语。这涵盖了使用 TCP 和 UDP 进行网络编程、文件系统访问，以及与操作系统信号和子进程进行异步交互。
  </x-card>
  <x-card data-title="同步" data-icon="lucide:lock" data-href="/concepts/synchronization">
    探索用于管理共享状态和任务间通信的工具。这包括各种通道类型（mpsc、oneshot、watch）、用于独占访问的互斥锁以及其他同步原语。
  </x-card>
  <x-card data-title="计时器" data-icon="lucide:timer" data-href="/concepts/timers">
    了解如何根据时间调度工作。学习创建延迟（休眠）、为操作设置超时，以及按固定间隔执行代码。
  </x-card>
  <x-card data-title="运行时" data-icon="lucide:server" data-href="/concepts/runtime">
    深入了解驱动这一切的引擎。了解多线程和当前线程调度器、运行时的配置方式，以及它如何驱动异步代码直至完成。
  </x-card>
</x-cards>

这些核心组件协同工作，为构建可靠的网络应用程序提供了一个强大而高效的平台。为了更深入地理解，我们建议从任何 Tokio 应用程序的基本构建块开始。

接下来，让我们深入了解[任务与调度](./concepts-tasks.md)。