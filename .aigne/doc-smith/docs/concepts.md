# Core Concepts

Tokio is an event-driven, non-blocking I/O platform for writing asynchronous applications with the Rust programming language. To build fast and reliable applications, it is helpful to understand the fundamental components that make up the Tokio runtime. This section explores these core concepts, providing a solid foundation for advanced usage.

At a high level, a Tokio application is structured around a runtime that executes asynchronous tasks. The runtime includes a task scheduler, an I/O driver that interacts with the operating system's event queue (like epoll, kqueue, or IOCP), and a high-performance timer.

```d2
direction: down

"Tokio Application": {
  "Runtime": {
    "Scheduler": {
      "Task 1": "async fn"
      "Task 2": "async fn"
      "Task N": "..."
    }

    "I/O Driver": {
      style.fill: "#B6DDF6"
      "OS Events (epoll, kqueue, IOCP)"
    }

    "Timer": {
      style.fill: "#B6DDF6"
    }
  }
}

"Runtime.Scheduler.Task 1" -> "Runtime.I/O Driver": "Async I/O (e.g., net, fs)" { style.animated: true }
"Runtime.I/O Driver" -> "Runtime.Scheduler.Task 1": "Wakes task on I/O readiness" { style.animated: true }

"Runtime.Scheduler.Task 2" -> "Runtime.Timer": "Request sleep/timeout" { style.animated: true }
"Runtime.Timer" -> "Runtime.Scheduler.Task 2": "Wakes task when time elapses" { style.animated: true }

"Runtime.Scheduler.Task 1" <-> "Runtime.Scheduler.Task 2": "Synchronization (e.g., channels, mutex)" { style.stroke-dash: 4 }
```

To better understand how these components work together, explore the following core concepts in detail.

<x-cards data-columns="2">
  <x-card data-title="Tasks & Scheduling" data-icon="lucide:workflow" data-href="/concepts/tasks">
    Asynchronous programs in Rust are built around lightweight, non-blocking units of execution called tasks. Learn how to spawn, await, and manage tasks, and understand how Tokio's scheduler executes them efficiently.
  </x-card>
  <x-card data-title="Asynchronous I/O" data-icon="lucide:arrow-right-left" data-href="/concepts/io">
    Tokio provides a suite of non-blocking APIs for I/O operations, including networking (TCP, UDP, UDS), filesystem access, and inter-process communication, all built on the `AsyncRead` and `AsyncWrite` traits.
  </x-card>
  <x-card data-title="Synchronization" data-icon="lucide:lock" data-href="/concepts/synchronization">
    When tasks need to communicate or share data, you can use Tokio's synchronization primitives. These include channels (oneshot, mpsc, watch, broadcast) and concurrency tools like `Mutex`.
  </x-card>
  <x-card data-title="Timers" data-icon="lucide:timer" data-href="/concepts/timers">
    Manage time-based operations in your asynchronous code. Tokio offers utilities for creating delays (`sleep`), executing code at regular intervals (`interval`), and enforcing time limits on operations (`timeout`).
  </x-card>
  <x-card data-title="The Runtime" data-icon="lucide:server" data-href="/concepts/runtime">
    The runtime is the engine that powers your asynchronous application. Delve into its components, including the multi-threaded and current-thread schedulers, and learn how to configure it for your specific needs.
  </x-card>
</x-cards>

With a grasp of these foundational pieces, you are well-equipped to write more sophisticated asynchronous applications. The best place to start is by taking a deeper look at how Tokio manages asynchronous work.

[Next: Tasks & Scheduling](./concepts-tasks.md)
