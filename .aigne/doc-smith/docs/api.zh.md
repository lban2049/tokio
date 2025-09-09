# API 参考

欢迎查阅 Tokio 的综合 API 参考。本节按模块组织，提供了所有公共 API 的详细文档。每个模块涵盖一个特定的功能领域，从异步 I/O 和网络到任务管理和同步。请浏览以下模块，找到您应用程序所需的工具。

如需了解 Tokio 组件更具叙事性的介绍，建议从 [核心概念](./concepts.md) 指南开始。

<x-cards data-columns="3">
  <x-card data-title="I/O" data-href="/api/io" data-icon="lucide:arrow-left-right">
    核心异步 I/O 原语，包括 AsyncRead 和 AsyncWrite 特质，以及用于处理它们的各种实用工具。
  </x-card>
  <x-card data-title="网络" data-href="/api/net" data-icon="lucide:globe">
    用于构建网络应用的非阻塞、异步 API，支持 TCP、UDP 和 Unix 域套接字 (UDS)。
  </x-card>
  <x-card data-title="同步" data-href="/api/sync" data-icon="lucide:lock">
    用于管理共享状态和任务间通信的原语，例如通道 (mpsc、oneshot 等)、Mutex 和 Barrier。
  </x-card>
  <x-card data-title="任务" data-href="/api/task" data-icon="lucide:cpu">
    用于创建、管理异步任务并与之交互的工具，包括任务局部存储和阻塞操作处理。
  </x-card>
  <x-card data-title="时间" data-href="/api/time" data-icon="lucide:timer">
    用于跟踪时间和调度工作的实用工具，包括超时、休眠和间隔函数。
  </x-card>
  <x-card data-title="文件系统" data-href="/api/fs" data-icon="lucide:folder">
    用于与文件系统交互的异步 API，为标准库的 `std::fs` 模块提供了非阻塞的替代方案。
  </x-card>
  <x-card data-title="进程" data-href="/api/process" data-icon="lucide:terminal-square">
    用于异步创建和管理子进程、捕获其输出和退出状态的 API。
  </x-card>
  <x-card data-title="信号" data-href="/api/signal" data-icon="lucide:radio-tower">
    用于异步处理操作系统信号的功能，支持平滑关闭及其他基于信号的交互。
  </x-card>
  <x-card data-title="运行时" data-href="/api/runtime" data-icon="lucide:settings-2">
    用于手动配置和管理 Tokio 运行时的 API，包括多线程和当前线程调度器。
  </x-card>
</x-cards>

现在您已经对可用模块有了大致了解，可以深入研究您感兴趣的特定领域。关于这些 API 的实际应用，请务必查看 [示例](./examples.md) 部分。