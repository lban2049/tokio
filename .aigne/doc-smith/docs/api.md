# API Reference

This section provides a comprehensive, searchable reference for all public APIs provided by the Tokio crate. The APIs are organized by module, allowing you to quickly find the functions, structs, and traits you need for your application.

For a higher-level introduction to Tokio's architecture and philosophy, please refer to the [Core Concepts](./concepts.md) guide.

### Module Overview

Tokio's functionality is divided into several modules, each targeting a specific area of asynchronous programming. The diagram below illustrates the relationship between the major components.

```d2
direction: down

API: "Tokio API" {
  Runtime: {
    href: "/api/runtime"
    tooltip: "The engine that executes asynchronous tasks."
  }

  Tasks: {
    href: "/api/task"
    tooltip: "The basic units of concurrent execution."
  }

  Synchronization: {
    href: "/api/sync"
    tooltip: "Primitives for safe communication between tasks."
  }

  Time: {
    href: "/api/time"
    tooltip: "Utilities for handling time-based events."
  }

  IO: "I/O" {
    href: "/api/io"
    tooltip: "Core traits and utilities for async I/O."

    Networking: {
      href: "/api/net"
    }
    Filesystem: {
      href: "/api/fs"
    }
  }

  OS: "OS Integration" {
    Processes: {
      href: "/api/process"
    }
    Signals: {
      href: "/api/signal"
    }
  }
}

API.Runtime -> API.Tasks: "Manages"
API.Tasks -> API.IO: "Performs"
API.Tasks -> API.Time: "Awaits"
API.Tasks -> API.Synchronization: "Uses"
API.IO -> API.OS: "Builds on"
```

Select a module below to view its detailed API documentation.

<x-cards data-columns="2">
  <x-card data-title="Runtime" data-icon="lucide:cpu" data-href="/api/runtime">
    APIs for configuring and managing the Tokio runtime, including the multi-threaded and current-thread schedulers.
  </x-card>
  <x-card data-title="Tasks" data-icon="lucide:cog" data-href="/api/task">
    Tools for working with asynchronous tasks, including spawning, awaiting completion, and task-local storage.
  </x-card>
  <x-card data-title="I/O" data-icon="lucide:arrow-right-left" data-href="/api/io">
    Core asynchronous I/O primitives like `AsyncRead` and `AsyncWrite`, along with utility functions for working with them.
  </x-card>
  <x-card data-title="Networking" data-icon="lucide:globe" data-href="/api/net">
    Asynchronous TCP, UDP, and Unix sockets for building high-performance network applications.
  </x-card>
  <x-card data-title="Synchronization" data-icon="lucide:lock" data-href="/api/sync">
    Primitives for managing shared state and communication between tasks, such as channels, mutexes, and barriers.
  </x-card>
  <x-card data-title="Time" data-icon="lucide:timer" data-href="/api/time">
    Utilities for working with time, including functions for sleeping, setting intervals, and enforcing timeouts.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder" data-href="/api/fs">
    Asynchronous APIs for interacting with the filesystem, providing non-blocking alternatives to `std::fs`.
  </x-card>
  <x-card data-title="Processes" data-icon="lucide:terminal" data-href="/api/process">
    APIs for spawning and managing child processes asynchronously.
  </x-card>
  <x-card data-title="Signals" data-icon="lucide:siren" data-href="/api/signal">
    Utilities for handling OS signals, such as SIGINT or SIGHUP, in an asynchronous manner.
  </x-card>
</x-cards>

---

Explore the modules above to find the specific APIs you need. If you are getting started with a network service, the [Networking](./api-net.md) and [I/O](./api-io.md) modules are excellent places to begin.
