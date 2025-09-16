# Asynchronous I/O

This module provides the asynchronous equivalent of `std::io`. It includes traits, helpers, and type definitions for working with non-blocking input and output operations.

At its core, Tokio's I/O functionality revolves around two fundamental traits: [`AsyncRead`](#asyncread) and [`AsyncWrite`](#asyncwrite). These are the asynchronous versions of the standard library's `Read` and `Write` traits. When an I/O operation cannot be completed immediately (e.g., waiting for data from the network), instead of blocking the current thread, it yields control back to the Tokio scheduler. This allows other tasks to run, enabling massive concurrency with only a few OS threads.

## The `AsyncRead` and `AsyncWrite` Traits

While library authors implement `AsyncRead` and `AsyncWrite` for their types, application developers will typically use the convenient methods provided by the extension traits `AsyncReadExt` and `AsyncWriteExt`. These traits are automatically available for any type that implements `AsyncRead` or `AsyncWrite` and provide familiar methods like `read()`, `read_to_string()`, `write()`, and `write_all()`.

Let's look at an example of reading from a file, similar to how you would with `std::fs::File`.

```rust File Read Example icon=logos:rust
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

This code asynchronously opens a file and reads up to 10 bytes into a buffer. If the file is not immediately ready for reading, the `.await` call will pause the task and allow other tasks to run until the data is available.

### Core Trait Methods (for Implementers)

For those building I/O types, it's important to understand the core methods of the traits.

#### AsyncRead

The `AsyncRead` trait has one required method, `poll_read`.

<x-field data-name="poll_read" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>, buf: &mut ReadBuf<'_'>) -> Poll<io::Result<()>>" data-required="true" data-desc="Attempts to read data into `buf`. If data is not ready, it returns `Poll::Pending` and registers the current task's waker to be notified when data becomes available. On success, it returns `Poll::Ready(Ok(()))`."></x-field>

#### AsyncWrite

The `AsyncWrite` trait requires three methods for writing, flushing, and shutting down the I/O resource.

<x-field data-name="poll_write" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>, buf: &[u8]) -> Poll<Result<usize, io::Error>>" data-required="true" data-desc="Attempts to write data from `buf`. Returns `Poll::Ready(Ok(n))` with the number of bytes written, or `Poll::Pending` if the writer is not ready."></x-field>
<x-field data-name="poll_flush" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>) -> Poll<Result<(), io::Error>>" data-required="true" data-desc="Attempts to flush any buffered data. Returns `Poll::Ready(Ok(()))` on success or `Poll::Pending` if the flush cannot complete immediately."></x-field>
<x-field data-name="poll_shutdown" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>) -> Poll<Result<(), io::Error>>" data-required="true" data-desc="Initiates a graceful shutdown of the writer. This is used for protocols that require a handshake before closing a connection."></x-field>

## Buffered Readers and Writers

Directly making system calls for every small read or write can be inefficient. To mitigate this, `tokio::io` provides buffered wrappers, `BufReader` and `BufWriter`, which work with the `AsyncBufRead` trait. These wrappers maintain an in-memory buffer to reduce the number of system calls.

`BufReader` adds helpful methods for reading, such as `read_line`.

```rust BufReader Example icon=logos:rust
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

`BufWriter` buffers write operations. It's crucial to call `flush()` to ensure all buffered data is written to the underlying writer before it is dropped.

```rust BufWriter Example icon=logos:rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write a byte to the buffer.
        writer.write_all(b"hello").await?;

        // Flush the buffer before it goes out of scope.
        writer.flush().await?;

    } // Unless flushed, the contents of the buffer is discarded on drop.

    Ok(())
}
```

## I/O Resources

Tokio provides a rich set of asynchronous I/O resources that build upon these core traits.

<x-cards data-columns="2">
  <x-card data-title="Networking" data-icon="lucide:globe" data-href="/io/networking">
    Primitives for asynchronous TCP, UDP, and Unix Domain Sockets.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder-open" data-href="/io/fs">
    Perform non-blocking file and directory operations.
  </x-card>
  <x-card data-title="Child Processes" data-icon="lucide:terminal-square" data-href="/io/process">
    Asynchronously spawn and manage child processes.
  </x-card>
  <x-card data-title="OS Signals" data-icon="lucide:siren" data-href="/io/signals">
    Handle Unix and Windows operating system signals.
  </x-card>
</x-cards>

Additionally, Tokio provides asynchronous APIs for standard input (`stdin`), output (`stdout`), and error (`stderr`), which implement `AsyncRead` and `AsyncWrite`. Note that these standard I/O APIs must be used from within a Tokio runtime context.

## Next Steps

Now that you have an overview of Tokio's asynchronous I/O model, you can explore specific I/O resources. A great place to start is with networking, which is one of Tokio's primary use cases.

<x-card data-title="Next: Networking" data-icon="lucide:arrow-right-circle" data-href="/io/networking" data-cta="Read More">
  Learn how to build asynchronous network applications using TCP and UDP.
</x-card>