# API Reference

Welcome to the Tokio API reference. This section provides detailed, comprehensive documentation for all public APIs exposed by the Tokio crate. The APIs are organized by module, allowing you to quickly find the functionality you need.

If you are looking for a higher-level understanding of Tokio's components, you may want to start with the [Core Concepts](./concepts.md) documentation first.

Below is a high-level overview of the main API modules provided by Tokio.

```d2
direction: down

Tokio-Runtime: {
  shape: package
  label: "Tokio Runtime"
  grid-columns: 1
  style.fill: "#f0f4f8"

  Core: {
    shape: rectangle
    label: "Scheduler & I/O Driver"
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

  Core -> Modules: "Executes & Manages"
}
```

<x-cards data-columns="3">
  <x-card data-title="I/O" data-icon="lucide:arrow-left-right" data-href="/api/io">
    Core asynchronous I/O primitives, including the AsyncRead and AsyncWrite traits and their utilities.
  </x-card>
  <x-card data-title="Networking" data-icon="lucide:globe" data-href="/api/net">
    Asynchronous TCP, UDP, and Unix Domain Sockets for building network applications.
  </x-card>
  <x-card data-title="Synchronization" data-icon="lucide:lock" data-href="/api/sync">
    Primitives for managing shared state and coordinating tasks, such as channels, mutexes, and barriers.
  </x-card>
  <x-card data-title="Tasks" data-icon="lucide:cpu" data-href="/api/task">
    Tools for spawning, managing, and interacting with asynchronous tasks, including handling blocking operations.
  </x-card>
  <x-card data-title="Time" data-icon="lucide:timer" data-href="/api/time">
    Utilities for working with time, including sleeps, intervals, and timeouts.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder-open" data-href="/api/fs">
    Asynchronous APIs for interacting with the filesystem, mirroring `std::fs`.
  </x-card>
  <x-card data-title="Processes" data-icon="lucide:terminal-square" data-href="/api/process">
    APIs for spawning and managing child processes asynchronously.
  </x-card>
  <x-card data-title="Signals" data-icon="lucide:radio-tower" data-href="/api/signal">
    Cross-platform support for handling OS signals like SIGINT asynchronously.
  </x-card>
  <x-card data-title="Runtime" data-icon="lucide:settings-2" data-href="/api/runtime">
    Advanced APIs for configuring and managing the Tokio runtime itself.
  </x-card>
</x-cards>

This reference is designed for quick lookups. To see how these APIs are used in practice, check out the [Examples](./examples.md) section for complete, runnable code snippets.
