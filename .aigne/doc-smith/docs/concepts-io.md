# Asynchronous I/O

Tokio provides a suite of non-blocking I/O primitives for building high-performance network applications. This includes tools for networking, filesystem operations, and inter-process communication. At its core, Tokio's I/O model is based on a few key traits that are asynchronous versions of their counterparts in the standard library.

## The `AsyncRead` and `AsyncWrite` Traits

The foundation of Tokio I/O is two fundamental traits: [`AsyncRead`](./api-io.md) and [`AsyncWrite`](./api-io.md). These are asynchronous versions of `std::io::Read` and `std::io::Write`. The key difference is that their methods do not block the current thread. When an operation cannot be completed immediately (for example, waiting for data from a network socket), the task will yield to the Tokio scheduler. The scheduler will then run another task, and resume the original task once the I/O resource is ready again.

This non-blocking approach allows a small number of threads to handle a large number of concurrent I/O operations.

```d2
shape: sequence_diagram
direction: down

"User Task"
"Tokio Runtime"
"Operating System (OS)"

"User Task" -> "Tokio Runtime": "socket.read().await"
"Tokio Runtime" -> "Operating System (OS)": "Check if socket is readable"
"Operating System (OS)" -> "Tokio Runtime": "Not ready"
"Tokio Runtime" -> "User Task": "Park the task"
"User Task": "Suspended (Yields CPU)"

"Operating System (OS)" -> "Tokio Runtime": "Event: Socket is now readable" {
  style.animated: true
}
"Tokio Runtime" -> "User Task": "Wake up the task"
"User Task" -> "Tokio Runtime": "Retry socket.read()"
"Tokio Runtime" -> "Operating System (OS)": "Read data from socket"
"Operating System (OS)" -> "Tokio Runtime": "Return data"
"Tokio Runtime" -> "User Task": ".await completes"
```

Most of the time, you won't use the core trait methods directly. Instead, you'll use convenient utility methods provided by the [`AsyncReadExt`](./api-io.md) and [`AsyncWriteExt`](./api-io.md) traits, which are automatically available for any type that implements `AsyncRead` or `AsyncWrite`.

Here's an example of reading up to 10 bytes from a file:

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

## Buffered I/O

Directly making system calls for every small read or write can be inefficient. To mitigate this, Tokio provides buffered readers and writers, similar to the standard library. The `BufReader` and `BufWriter` structs wrap any async reader or writer and use an in-memory buffer to reduce the number of system calls.

`BufReader` adds convenient methods like `read_line`:

```rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let mut reader = BufReader::new(f);
    let mut buffer = String::new();

    // read a line into buffer
    reader.read_line(&mut buffer).await?;

    println!("{}", buffer);
    Ok(())
}
```

`BufWriter` buffers write operations. It's crucial to call `flush()` to ensure that any data remaining in the buffer is written to the underlying writer.

```rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    let mut writer = BufWriter::new(f);

    // Write bytes to the buffer.
    writer.write_all(b"some bytes").await?;

    // Flush the buffer to ensure data is written to the file.
    writer.flush().await?;

    Ok(())
}
```

## Tokio's I/O Toolkit

Tokio provides a comprehensive set of APIs for various kinds of I/O operations.

<x-cards data-columns="3">
  <x-card data-title="Networking" data-icon="lucide:globe" data-href="/api/net">
    Non-blocking TCP, UDP, and Unix Sockets for building network clients and servers.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder" data-href="/api/fs">
    Asynchronous APIs for file and filesystem manipulation, like reading, writing, and creating directories.
  </x-card>
  <x-card data-title="Processes" data-icon="lucide:terminal-square" data-href="/api/process">
    Spawn and manage child processes asynchronously, interacting with their stdin, stdout, and stderr streams.
  </x-card>
</x-cards>

### A Note on Filesystem I/O

Most operating systems do not provide true asynchronous APIs for filesystem access. To work around this, Tokio's `fs` module uses a dedicated thread pool for blocking file operations via `spawn_blocking`. While this provides an asynchronous API to your application, be aware that it has performance implications. For high-throughput file I/O, consider strategies like batching operations or using `spawn_blocking` manually to perform multiple synchronous operations in one go.

### Standard I/O and OS Signals

Tokio also provides asynchronous versions of standard input, output, and error streams ([`stdin`](./api-io.md), [`stdout`](./api-io.md), [`stderr`](./api-io.md)). Additionally, the [`tokio::signal`](./api-signal.md) module allows you to handle OS signals like `SIGINT` (Ctrl-C) asynchronously.

---

Now that you understand Tokio's I/O model, the next step is to learn how to manage shared state between tasks that are performing I/O. For this, you should explore Tokio's [Synchronization Primitives](./concepts-synchronization.md).
