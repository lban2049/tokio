# Core Concepts

Tokio is an event-driven, non-blocking I/O platform for writing asynchronous applications with Rust. To build reliable and high-performance network applications, it's helpful to understand the fundamental components that make up the Tokio runtime. This section provides a high-level overview of these core concepts, forming a solid foundation for more advanced usage.

At a high level, Tokio's architecture is built around a few major components that work together to execute your asynchronous code efficiently.

```d2
direction: down

"Tokio Runtime" {
  shape: package
  grid-columns: 1
  grid-gap: 50

  "Core Components" {
    grid-columns: 2

    "Scheduler": {
      shape: rectangle
      "Manages and executes tasks"
    }

    "I/O Driver": {
      shape: rectangle
      "Interfaces with OS events (epoll, kqueue, IOCP)"
    }
  }

  "User-Facing APIs" {
    grid-columns: 2
    grid-gap: 20

    "Tasks": {
      shape: package
      "spawn()"
      "JoinHandle"
      "spawn_blocking()"
    }

    "Asynchronous I/O": {
      shape: package
      "TCP / UDP"
      "Filesystem"
      "Signals"
      "Processes"
    }

    "Synchronization Primitives": {
      shape: package
      "Channels (mpsc, oneshot)"
      "Mutex"
      "Barrier"
    }

    "Timers": {
      shape: package
      "sleep()"
      "interval()"
      "timeout()"
    }
  }

  "Core Components".Scheduler -> "User-Facing APIs".Tasks: "Executes"
  "Core Components"."I/O Driver" -> "User-Facing APIs"."Asynchronous I/O": "Drives"
}

"Your Application Code" {
  shape: rectangle
}

"Your Application Code" -> "Tokio Runtime"."User-Facing APIs": "Uses"

```

Below, we explore each of these fundamental building blocks. Each card links to a more detailed guide on the specific topic.

<x-cards data-columns="2">
  <x-card data-title="Tasks & Scheduling" data-href="/concepts/tasks" data-icon="lucide:workflow">
    Asynchronous programs in Rust are built around lightweight, non-blocking units of execution called tasks. Learn how to spawn, manage, and coordinate these tasks on the Tokio runtime.
  </x-card>
  <x-card data-title="Asynchronous I/O" data-href="/concepts/io" data-icon="lucide:arrow-right-left">
    Tokio provides a suite of non-blocking APIs for I/O operations, including networking (TCP, UDP), filesystem access, and inter-process communication, all without blocking threads.
  </x-card>
  <x-card data-title="Synchronization" data-href="/concepts/synchronization" data-icon="lucide:lock">
    When tasks need to communicate or share data, Tokio offers a set of synchronization primitives like channels, mutexes, and barriers, all designed for the asynchronous world.
  </x-card>
  <x-card data-title="Timers" data-href="/concepts/timers" data-icon="lucide:timer">
    Explore utilities for tracking time and scheduling future work. This includes setting timeouts, sleeping for a duration, or repeating an operation at a specific interval.
  </x-card>
  <x-card data-title="The Runtime" data-href="/concepts/runtime" data-icon="lucide:server">
    The runtime is the engine that executes asynchronous tasks. Delve into its components, including the task scheduler, I/O driver, and timer, and learn how to configure it for your needs.
  </x-card>
</x-cards>

## Handling Blocking Code

Tokio achieves high concurrency by running many tasks on a small number of threads. This model relies on tasks yielding control at `.await` points so other tasks can run. However, code that performs long-running, CPU-intensive computations or blocking I/O will prevent other tasks from running on the same thread.

To handle this, Tokio provides a dedicated thread pool for blocking operations. You can offload blocking code to this pool using the `spawn_blocking` function, ensuring it doesn't interfere with the main asynchronous task scheduler.

```rust
#[tokio::main]
async fn main() {
    // This is running on a core scheduler thread.

    let blocking_task = tokio::task::spawn_blocking(|| {
        // This is running on a dedicated blocking thread.
        // Performing a blocking operation here is okay.
        std::thread::sleep(std::time::Duration::from_secs(1));
        "done"
    });

    // We can wait for the blocking task to complete without blocking the scheduler.
    let result = blocking_task.await.unwrap();
    assert_eq!(result, "done");
}
```
This separation is crucial for building responsive applications that mix asynchronous and synchronous code.

---

With this overview, you have a map of Tokio's core architecture. The best place to start a deeper dive is with tasks, as they are the fundamental unit of execution in any Tokio application.

Next, we recommend reading about [Tasks & Scheduling](./concepts-tasks.md) to understand how your asynchronous code is executed.