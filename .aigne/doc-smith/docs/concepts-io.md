# Asynchronous I/O

Tokio provides a suite of non-blocking I/O primitives for building high-performance network applications, managing filesystem operations, and interacting with child processes. Unlike the standard library's `std::io`, which blocks the current thread while waiting for I/O to complete, Tokio's I/O operations are asynchronous. When an operation cannot complete immediately, it yields control back to the Tokio scheduler, allowing other tasks to run. This is the key to achieving high concurrency with a small number of threads.

All asynchronous I/O in Tokio is built upon a foundation of two core traits from the `tokio::io` module: `AsyncRead` and `AsyncWrite`. These are the asynchronous equivalents of the standard library's `Read` and `Write` traits.

## Core I/O Primitives

The `AsyncRead` and `AsyncWrite` traits provide the basic building blocks for asynchronous byte streams. However, you will rarely implement or use these traits' core methods directly. Instead, you'll use the convenient utility methods provided by the `AsyncReadExt` and `AsyncWriteExt` extension traits, which are automatically available for any type that implements `AsyncRead` or `AsyncWrite`.

For example, to read data from an asynchronous source, you can use the `.read()` method from `AsyncReadExt`:

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

async fn read_from_file() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

### Buffered I/O

To improve efficiency by reducing the number of system calls, Tokio provides buffered readers and writers, similar to the standard library. The `BufReader` and `BufWriter` structs wrap any `AsyncRead` or `AsyncWrite` type, respectively. `BufReader` also enables more convenient methods, such as reading data line-by-line.

```rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

async fn read_lines_from_file() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let mut reader = BufReader::new(f);
    let mut buffer = String::new();

    // read a line into buffer
    reader.read_line(&mut buffer).await?;

    println!("{}", buffer);
    Ok(())
}
```

## Tokio's I/O Model

The Tokio runtime orchestrates all asynchronous operations. For network-based I/O, it uses the operating system's most efficient event notification system (like epoll on Linux, kqueue on macOS, or IOCP on Windows). For filesystem operations, which are typically blocking at the OS level, Tokio uses a dedicated thread pool to ensure the main scheduler is never blocked.

```d2
direction: down

"Application Tasks" {
  shape: package
}

"Tokio Runtime" {
  grid-columns: 2
  
  "Core Scheduler": {
    shape: rectangle
    "Wakes tasks based on events"
  }

  "I/O Driver (epoll, kqueue, IOCP)": {
    shape: rectangle
    "Handles non-blocking resources"
  }
  
  "Timer": {
    shape: rectangle
    "Manages sleeps and timeouts"
  }

  "Blocking Thread Pool": {
    shape: rectangle
    "For CPU-bound or blocking OS calls"
  }
}

"OS Resources" {
    shape: package
    grid-columns: 2

    "Network (TCP, UDP, UDS)": { shape: cylinder }
    "Filesystem": { shape: cylinder }
    "Child Processes": { shape: cylinder }
    "OS Signals": { shape: cylinder }
}

"Application Tasks" <-> "Tokio Runtime"."Core Scheduler": "Spawns & Yields"
"Tokio Runtime"."Core Scheduler" -> "Tokio Runtime"."I/O Driver (epoll, kqueue, IOCP)": "Registers I/O interests"
"Tokio Runtime"."I/O Driver (epoll, kqueue, IOCP)" -> "Tokio Runtime"."Core Scheduler": "Notifies on readiness"
"Tokio Runtime"."Core Scheduler" -> "Tokio Runtime"."Blocking Thread Pool": "Dispatches blocking work"
"Tokio Runtime"."Blocking Thread Pool" -> "Tokio Runtime"."Core Scheduler": "Returns result"

"Tokio Runtime"."I/O Driver (epoll, kqueue, IOCP)" <-> "OS Resources"."Network (TCP, UDP, UDS)": "Non-blocking"
"Tokio Runtime"."I/O Driver (epoll, kqueue, IOCP)" <-> "OS Resources"."Child Processes": "Non-blocking"
"Tokio Runtime"."I/O Driver (epoll, kqueue, IOCP)" <-> "OS Resources"."OS Signals": "Non-blocking"
"Tokio Runtime"."Blocking Thread Pool" <-> "OS Resources"."Filesystem": "Blocking"
```

## I/O Resource Types

Tokio provides a rich set of I/O types that cover most common use cases. Each is designed to integrate seamlessly into the asynchronous runtime.

<x-cards data-columns="2">
  <x-card data-title="Networking" data-icon="lucide:globe">
    For TCP, UDP, and Unix Sockets. These types integrate directly with the operating system's event queue for efficient, non-blocking network communication.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder">
    For asynchronous file operations. Since most operating systems lack true async file APIs, these operations are run on a dedicated blocking thread pool to avoid stalling the runtime.
  </x-card>
  <x-card data-title="Processes" data-icon="lucide:terminal">
    For spawning and managing child processes. The standard input, output, and error streams of a child process are exposed as asynchronous I/O handles.
  </x-card>
  <x-card data-title="Signals" data-icon="lucide:siren">
    For handling Unix and Windows OS signals asynchronously. This allows your application to gracefully respond to system events like termination requests.
  </x-card>
</x-cards>

## Summary

Tokio's I/O model provides a consistent and powerful foundation for building reliable applications. By abstracting different kinds of I/O behind the `AsyncRead` and `AsyncWrite` traits, you can write generic code that works across network sockets, files, and process streams.

Now that you understand the I/O fundamentals, a good next step is to explore how to manage shared data between tasks that perform I/O in the [Synchronization](./concepts-synchronization.md) section.