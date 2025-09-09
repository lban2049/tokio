# API Reference

Welcome to the comprehensive API reference for Tokio. This section provides detailed documentation for all public APIs, organized by module. Each module covers a specific area of functionality, from asynchronous I/O and networking to task management and synchronization. Browse the modules below to find the tools you need for your application.

For a more narrative introduction to Tokio's components, you might want to start with the [Core Concepts](./concepts.md) guide.

<x-cards data-columns="3">
  <x-card data-title="I/O" data-href="/api/io" data-icon="lucide:arrow-left-right">
    Core asynchronous I/O primitives, including the AsyncRead and AsyncWrite traits, and various utilities for working with them.
  </x-card>
  <x-card data-title="Networking" data-href="/api/net" data-icon="lucide:globe">
    Non-blocking, asynchronous APIs for TCP, UDP, and Unix Domain Sockets (UDS) for building network applications.
  </x-card>
  <x-card data-title="Synchronization" data-href="/api/sync" data-icon="lucide:lock">
    Primitives for managing shared state and communication between tasks, such as channels (mpsc, oneshot, etc.), Mutex, and Barrier.
  </x-card>
  <x-card data-title="Tasks" data-href="/api/task" data-icon="lucide:cpu">
    Tools for spawning, managing, and interacting with asynchronous tasks, including task-local storage and handling blocking operations.
  </x-card>
  <x-card data-title="Time" data-href="/api/time" data-icon="lucide:timer">
    Utilities for tracking time and scheduling work, including functions for timeouts, sleeps, and intervals.
  </x-card>
  <x-card data-title="Filesystem" data-href="/api/fs" data-icon="lucide:folder">
    Asynchronous APIs for interacting with the filesystem, providing non-blocking alternatives to the standard library's `std::fs` module.
  </x-card>
  <x-card data-title="Processes" data-href="/api/process" data-icon="lucide:terminal-square">
    APIs for asynchronously spawning and managing child processes, capturing their output and exit status.
  </x-card>
  <x-card data-title="Signals" data-href="/api/signal" data-icon="lucide:radio-tower">
    Functionality for handling OS signals asynchronously, allowing for graceful shutdown and other signal-based interactions.
  </x-card>
  <x-card data-title="Runtime" data-href="/api/runtime" data-icon="lucide:settings-2">
    APIs for manually configuring and managing the Tokio runtime, including the multi-threaded and current-thread schedulers.
  </x-card>
</x-cards>

Now that you have an overview of the available modules, you can dive into the specific area you're interested in. For practical applications of these APIs, be sure to check out the [Examples](./examples.md) section.