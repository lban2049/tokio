# API Reference

This section provides a comprehensive, searchable reference for all public APIs provided by the Tokio crate. The APIs are organized by module to help you quickly find the functionality you need. Select a module below to explore its detailed documentation.

<x-cards data-columns="3">
  <x-card data-title="I/O" data-href="/api/io" data-icon="lucide:arrow-right-left">
    Tokio's asynchronous core I/O primitives, including the AsyncRead, AsyncWrite, and AsyncBufRead traits.
  </x-card>
  <x-card data-title="Networking" data-href="/api/net" data-icon="lucide:globe">
    Non-blocking versions of TCP, UDP, and Unix Domain Sockets for network communication.
  </x-card>
  <x-card data-title="Synchronization" data-href="/api/sync" data-icon="lucide:lock">
    Primitives for communicating and sharing data between tasks, such as channels and mutexes.
  </x-card>
  <x-card data-title="Tasks" data-href="/api/task" data-icon="lucide:list-checks">
    Tools for working with asynchronous tasks, including spawning new tasks and awaiting their output.
  </x-card>
  <x-card data-title="Time" data-href="/api/time" data-icon="lucide:timer">
    Utilities for tracking time and scheduling work, such as sleeps, intervals, and timeouts.
  </x-card>
  <x-card data-title="Filesystem" data-href="/api/fs" data-icon="lucide:folder">
    APIs for performing filesystem I/O asynchronously, similar to the standard library's `std::fs`.
  </x-card>
  <x-card data-title="Processes" data-href="/api/process" data-icon="lucide:terminal-square">
    Tools for spawning and managing child processes asynchronously.
  </x-card>
  <x-card data-title="Signals" data-href="/api/signal" data-icon="lucide:siren">
    Utilities for asynchronously handling Unix and Windows OS signals.
  </x-card>
  <x-card data-title="Runtime" data-href="/api/runtime" data-icon="lucide:settings-2">
    Powerful APIs for configuring and managing runtimes, for when the `#[tokio::main]` macro is not enough.
  </x-card>
</x-cards>