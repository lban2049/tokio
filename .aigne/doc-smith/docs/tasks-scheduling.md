# Tasks & Scheduling

Asynchronous programs in Tokio are built around tasks. A task is a lightweight, non-blocking unit of execution, similar to goroutines in Go or coroutines in Kotlin. Instead of being managed by the OS scheduler like traditional threads, tasks are managed by the Tokio runtime, which makes creating and switching between them extremely efficient.

This cooperative, non-blocking model allows a small number of threads to handle a massive number of concurrent operations. To help you build sophisticated applications, Tokio provides a comprehensive suite of tools for spawning, managing, and coordinating these tasks.

Explore the core concepts of Tokio's concurrency model in the following sections:

<x-cards data-columns="2">
  <x-card data-title="Spawning & Managing Tasks" data-icon="lucide:play-circle" data-href="/tasks-scheduling/spawning">
    Learn how to create and manage concurrent tasks. This section covers spawning tasks with `tokio::spawn`, awaiting their results using `JoinHandle`, and safely running blocking code without stalling the runtime.
  </x-card>
  <x-card data-title="Synchronization Primitives" data-icon="lucide:git-merge" data-href="/tasks-scheduling/synchronization">
    Coordinate independent tasks and share data safely. Explore message-passing with channels (mpsc, oneshot, broadcast, watch) and state synchronization with primitives like Mutex, Semaphore, and Barrier.
  </x-card>
  <x-card data-title="Time, Delays, and Timeouts" data-icon="lucide:timer" data-href="/tasks-scheduling/time">
    Integrate time-based logic into your asynchronous applications. This section covers how to create delays with `sleep`, run code at fixed periods with `interval`, and enforce execution deadlines with `timeout`.
  </x-card>
  <x-card data-title="The Runtime" data-icon="lucide:cpu" data-href="/tasks-scheduling/runtime">
    Understand the engine that powers your Tokio applications. The runtime includes a task scheduler, an I/O driver, and a timer. Learn how to configure different schedulers and manage the runtime's lifecycle.
  </x-card>
</x-cards>

Mastering these components is key to building fast, reliable, and scalable network applications. Once you are comfortable with how tasks are managed, you can move on to leveraging them for real-world operations.

Next, let's explore how to perform [Asynchronous I/O](./io.md).
