# API 参考

欢迎查阅 Tokio API 参考。本部分为 Tokio crate 提供的所有公共 API 提供了详细、全面的文档。这些 API 按模块进行组织，方便您快速查找所需的功能。

如果您希望对 Tokio 的组件有一个更高层次的理解，建议先阅读 [核心概念](./concepts.md) 文档。

以下是 Tokio 提供的主要 API 模块的概览。

```d2
direction: down

Tokio-Runtime: {
  shape: package
  label: "Tokio 运行时"
  grid-columns: 1
  style.fill: "#f0f4f8"

  Core: {
    shape: rectangle
    label: "调度器与 I/O 驱动"
  }

  Modules: {
    grid-columns: 3
    grid-gap: 50

    Tasks: {
      shape: class
      label: "tokio::task"
    }
    Sync: {
      shape: class
      label: "tokio::sync"
    }
    Time: {
      shape: class
      label: "tokio::time"
    }
    IO: {
      shape: class
      label: "tokio::io"
    }
    Net: {
      shape: class
      label: "tokio::net"
    }
    FS: {
      shape: class
      label: "tokio::fs"
    }
    Process: {
      shape: class
      label: "tokio::process"
    }
    Signal: {
      shape: class
      label: "tokio::signal"
    }
  }

  Core -> Modules: "执行与管理"
}
```

<x-cards data-columns="3">
  <x-card data-title="I/O" data-icon="lucide:arrow-left-right" data-href="/api/io">
    核心异步 I/O 原语，包括 AsyncRead 和 AsyncWrite trait 及其相关工具。
  </x-card>
  <x-card data-title="网络" data-icon="lucide:globe" data-href="/api/net">
    用于构建网络应用的异步 TCP、UDP 和 Unix 域套接字。
  </x-card>
  <x-card data-title="同步" data-icon="lucide:lock" data-href="/api/sync">
    用于管理共享状态和协调任务的原语，例如通道、互斥锁和屏障。
  </x-card>
  <x-card data-title="任务" data-icon="lucide:cpu" data-href="/api/task">
    用于创建、管理异步任务以及与之交互的工具，包括处理阻塞操作。
  </x-card>
  <x-card data-title="时间" data-icon="lucide:timer" data-href="/api/time">
    用于处理时间的实用工具，包括休眠、间隔和超时。
  </x-card>
  <x-card data-title="文件系统" data-icon="lucide:folder-open" data-href="/api/fs">
    用于与文件系统交互的异步 API，与 `std::fs` 类似。
  </x-card>
  <x-card data-title="进程" data-icon="lucide:terminal-square" data-href="/api/process">
    用于异步创建和管理子进程的 API。
  </x-card>
  <x-card data-title="信号" data-icon="lucide:radio-tower" data-href="/api/signal">
    提供跨平台支持，用于异步处理 SIGINT 等操作系统信号。
  </x-card>
  <x-card data-title="运行时" data-icon="lucide:settings-2" data-href="/api/runtime">
    用于配置和管理 Tokio 运行时本身的高级 API。
  </x-card>
</x-cards>

本参考旨在方便快速查阅。要了解这些 API 在实践中的应用，请查看 [示例](./examples.md) 部分，其中包含完整、可运行的代码片段。
