# API 参考

本部分为 Tokio crate 提供的所有公共 API 提供了一个全面、可搜索的参考。这些 API 按模块进行组织，以帮助您快速找到所需的功能。请在下方选择一个模块，以浏览其详细文档。

<x-cards data-columns="3">
  <x-card data-title="I/O" data-href="/api/io" data-icon="lucide:arrow-right-left">
    Tokio 的异步核心 I/O 原语，包括 AsyncRead、AsyncWrite 和 AsyncBufRead trait。
  </x-card>
  <x-card data-title="网络" data-href="/api/net" data-icon="lucide:globe">
    用于网络通信的 TCP、UDP 和 Unix 域套接字的非阻塞版本。
  </x-card>
  <x-card data-title="同步" data-href="/api/sync" data-icon="lucide:lock">
    用于在任务之间通信和共享数据的原语，例如通道和互斥锁。
  </x-card>
  <x-card data-title="任务" data-href="/api/task" data-icon="lucide:list-checks">
    用于处理异步任务的工具，包括派生新任务和等待其输出。
  </x-card>
  <x-card data-title="时间" data-href="/api/time" data-icon="lucide:timer">
    用于跟踪时间和调度工作的实用工具，例如休眠、间隔和超时。
  </x-card>
  <x-card data-title="文件系统" data-href="/api/fs" data-icon="lucide:folder">
    用于异步执行文件系统 I/O 的 API，类似于标准库的 `std::fs`。
  </x-card>
  <x-card data-title="进程" data-href="/api/process" data-icon="lucide:terminal-square">
    用于异步派生和管理子进程的工具。
  </x-card>
  <x-card data-title="信号" data-href="/api/signal" data-icon="lucide:siren">
    用于异步处理 Unix 和 Windows 操作系统信号的实用工具。
  </x-card>
  <x-card data-title="运行时" data-href="/api/runtime" data-icon="lucide:settings-2">
    用于配置和管理运行时的强大 API，适用于 `#[tokio::main]` 宏不够用的情况。
  </x-card>
</x-cards>