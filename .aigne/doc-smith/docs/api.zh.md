# API 参考

本节为 Tokio crate 提供的所有公共 API 提供了全面、可搜索的参考。这些 API 按模块组织，让您可以快速找到应用程序所需的函数、结构和 trait。

有关 Tokio 架构和理念的更高级别介绍，请参阅 [核心概念](./concepts.md) 指南。

### 模块概述

Tokio 的功能分为几个模块，每个模块都针对异步编程的特定领域。下图说明了主要组件之间的关系。

```d2
direction: down

API: "Tokio API" {
  Runtime: {
    href: "/api/runtime"
    tooltip: "执行异步任务的引擎。"
  }

  Tasks: {
    href: "/api/task"
    tooltip: "并发执行的基本单元。"
  }

  Synchronization: {
    href: "/api/sync"
    tooltip: "用于任务间安全通信的原语。"
  }

  Time: {
    href: "/api/time"
    tooltip: "用于处理基于时间的事件的实用工具。"
  }

  IO: "I/O" {
    href: "/api/io"
    tooltip: "用于异步 I/O 的核心 trait 和实用工具。"

    Networking: {
      href: "/api/net"
    }
    Filesystem: {
      href: "/api/fs"
    }
  }

  OS: "操作系统集成" {
    Processes: {
      href: "/api/process"
    }
    Signals: {
      href: "/api/signal"
    }
  }
}

API.Runtime -> API.Tasks: "管理"
API.Tasks -> API.IO: "执行"
API.Tasks -> API.Time: "等待"
API.Tasks -> API.Synchronization: "使用"
API.IO -> API.OS: "构建于"
```

在下方选择一个模块以查看其详细的 API 文档。

<x-cards data-columns="2">
  <x-card data-title="运行时" data-icon="lucide:cpu" data-href="/api/runtime">
    用于配置和管理 Tokio 运行时的 API，包括多线程和当前线程调度器。
  </x-card>
  <x-card data-title="任务" data-icon="lucide:cog" data-href="/api/task">
    用于处理异步任务的工具，包括生成、等待完成和任务局部存储。
  </x-card>
  <x-card data-title="I/O" data-icon="lucide:arrow-right-left" data-href="/api/io">
    核心异步 I/O 原语，如 `AsyncRead` 和 `AsyncWrite`，以及用于处理它们的实用函数。
  </x-card>
  <x-card data-title="网络" data-icon="lucide:globe" data-href="/api/net">
    用于构建高性能网络应用程序的异步 TCP、UDP 和 Unix 套接字。
  </x-card>
  <x-card data-title="同步" data-icon="lucide:lock" data-href="/api/sync">
    用于管理共享状态和任务间通信的原语，例如通道、互斥锁和屏障。
  </x-card>
  <x-card data-title="时间" data-icon="lucide:timer" data-href="/api/time">
    用于处理时间的实用工具，包括用于休眠、设置间隔和强制超时的函数。
  </x-card>
  <x-card data-title="文件系统" data-icon="lucide:folder" data-href="/api/fs">
    用于与文件系统交互的异步 API，提供 `std::fs` 的非阻塞替代方案。
  </x-card>
  <x-card data-title="进程" data-icon="lucide:terminal" data-href="/api/process">
    用于异步生成和管理子进程的 API。
  </x-card>
  <x-card data-title="信号" data-icon="lucide:siren" data-href="/api/signal">
    用于以异步方式处理操作系统信号（如 SIGINT 或 SIGHUP）的实用工具。
  </x-card>
</x-cards>

---

浏览以上模块以找到您需要的特定 API。如果您正在开始构建网络服务，[网络](./api-net.md) 和 [I/O](./api-io.md) 模块是绝佳的起点。
