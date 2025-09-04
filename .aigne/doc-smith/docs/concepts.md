# Core Concepts

Tokio is an event-driven, non-blocking I/O platform for writing asynchronous applications with Rust. To use Tokio effectively, it's important to understand its fundamental components. This section provides a high-level overview of the key concepts that form the foundation of the Tokio runtime, giving you a solid base for building advanced applications.

At a high level, Tokio is composed of several key parts that work in concert:

```d2
direction: down

"Tokio-Runtime": {
  label: "Tokio Runtime"
  shape: package
  grid-columns: 1
  grid-gap: 50

  "Core-Components": {
    label: "Core Components"
    shape: rectangle
    grid-columns: 3

    Scheduler: {
      label: "Task Scheduler"
    }
    "I-O-Driver": {
      label: "I/O Driver\n(epoll, kqueue, IOCP)"
    }
    Timer: {
      label: "High-Performance Timer"
    }
  }

  "User-Code": {
    label: "User Code"
    shape: rectangle
    grid-columns: 2

    "Async-Task-1": {
      label: "Async Task 1"
    }
    "Async-Task-2": {
      label: "Async Task 2"
    }
    "...": {}
    "Async-Task-N": {
      label: "Async Task N"
    }
  }

  "OS-Resources": {
    label: "OS Resources"
    shape: rectangle
    grid-columns: 2

    "TCP-Socket": { 
      label: "TCP Socket"
      shape: cylinder 
    }
    File: { 
      shape: cylinder 
    }
    "UDP-Socket": { 
      label: "UDP Socket"
      shape: cylinder 
    }
    Process: { 
      shape: cylinder 
    }
  }

  "Core-Components".Scheduler -> "User-Code": "Executes"
  "User-Code"."Async-Task-1" -> "OS-Resources"."TCP-Socket": "Performs I/O"
  "OS-Resources" -> "Core-Components"."I-O-Driver": "Registers with"
  "Core-Components"."I-O-Driver" -> "Core-Components".Scheduler: "Notifies readiness"
}
```

Explore the fundamental concepts in more detail below:

<x-cards data-columns="2">
  <x-card data-title="Tasks & Scheduling" data-icon="lucide:box" data-href="/concepts/tasks">
    Learn about the basic unit of execution in Tokio: the asynchronous task. This includes how to spawn new tasks, await their results, and manage blocking or CPU-intensive operations without halting the entire runtime.
  </x-card>
  <x-card data-title="Asynchronous I/O" data-icon="lucide:arrow-right-left" data-href="/concepts/io">
    Explore Tokio's non-blocking primitives for I/O operations. This covers networking with TCP and UDP, filesystem access, and interacting with OS signals and child processes asynchronously.
  </x-card>
  <x-card data-title="Synchronization" data-icon="lucide:lock" data-href="/concepts/synchronization">
    Discover tools for managing shared state and communication between tasks. This includes various channel types (mpsc, oneshot, watch), mutexes for exclusive access, and other synchronization primitives.
  </x-card>
  <x-card data-title="Timers" data-icon="lucide:timer" data-href="/concepts/timers">
    Understand how to schedule work based on time. Learn to create delays (sleeps), set timeouts for operations, and execute code at regular intervals.
  </x-card>
  <x-card data-title="The Runtime" data-icon="lucide:server" data-href="/concepts/runtime">
    Dive into the engine that powers it all. Learn about the multi-threaded and current-thread schedulers, how the runtime is configured, and how it drives asynchronous code to completion.
  </x-card>
</x-cards>

These core components work together to provide a powerful and efficient platform for building reliable network applications. To get a deeper understanding, we recommend starting with the fundamental building block of any Tokio application.

Next, let's dive into [Tasks & Scheduling](./concepts-tasks.md).