# I/O

This module provides traits, helpers, and type definitions for asynchronous I/O functionality. It serves as the asynchronous counterpart to `std::io`.

The cornerstone of Tokio's I/O is a pair of traits: `AsyncRead` and `AsyncWrite`, which are asynchronous versions of the `Read` and `Write` traits in the standard library.

## `AsyncRead` and `AsyncWrite`

Similar to the standard library's `Read` and `Write` traits, `AsyncRead` and `AsyncWrite` offer a general-purpose interface for reading and writing byte streams. The primary difference is their asynchronous nature. When an I/O operation cannot be completed immediately, instead of blocking the thread, it yields to the Tokio scheduler. This allows other tasks to run while the I/O operation is pending.

Utility methods for these traits are provided through extension traits, `AsyncReadExt` and `AsyncWriteExt`, which are automatically implemented for any type that implements `AsyncRead` and `AsyncWrite`.

For instance, reading from a `tokio::fs::File` is very similar to reading from a `std::fs::File`:

```rust,no_run
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

### Buffered I/O

To improve efficiency and reduce system calls, Tokio provides buffered I/O types, analogous to those in `std::io`. These include the `AsyncBufRead` trait and the `BufReader` and `BufWriter` structs. These wrappers use an internal buffer to batch I/O operations.

`BufReader` enhances any async reader with methods like `read_line`:

```rust,no_run
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

`BufWriter` buffers write operations. It is essential to call `flush` to ensure that all buffered data is written to the underlying writer.

```rust,no_run
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write a byte to the buffer.
        writer.write(&[42u8]).await?;

        // Flush the buffer before it goes out of scope.
        writer.flush().await?;

    } // Unless flushed, the contents of the buffer is discarded on drop.

    Ok(())
}
```

## Core I/O Traits

Tokio's I/O functionality is built around a set of core traits that define asynchronous behavior for reading, writing, and seeking.

<x-cards data-columns="2">
  <x-card data-title="AsyncRead" data-icon="lucide:arrow-down-circle">
    Asynchronously reads bytes from a source. Analogous to `std::io::Read`.
  </x-card>
  <x-card data-title="AsyncWrite" data-icon="lucide:arrow-up-circle">
    Asynchronously writes bytes to a destination. Analogous to `std::io::Write`.
  </x-card>
  <x-card data-title="AsyncBufRead" data-icon="lucide:layers">
    A trait for buffered asynchronous reading, providing methods like `read_line`.
  </x-card>
  <x-card data-title="AsyncSeek" data-icon="lucide:move-horizontal">
    A trait for seeking to different positions in an asynchronous I/O stream.
  </x-card>
</x-cards>

### trait AsyncRead

This trait allows for reading bytes from a source asynchronously. The core method, `poll_read`, attempts to read bytes into a buffer. If data is not immediately available, it returns `Poll::Pending` and arranges for the current task to be notified when the source becomes readable again.

**Key Method**

| Method | Description |
|---|---|
| `poll_read(self: Pin<&mut Self>, cx: &mut Context<'_>, buf: &mut ReadBuf<'_>) -> Poll<io::Result<()>>` | Attempts to read data into `buf`. Returns `Poll::Ready(Ok(()))` on success, `Poll::Pending` if no data is available, and `Poll::Ready(Err(e))` on error. |

### trait AsyncWrite

This trait allows for writing bytes to a destination asynchronously. Its methods will return `Poll::Pending` if the destination is not ready to accept more data, scheduling the current task for notification when it becomes writable.

**Key Methods**

| Method | Description |
|---|---|
| `poll_write(self: Pin<&mut Self>, cx: &mut Context<'_>, buf: &[u8]) -> Poll<Result<usize, io::Error>>` | Attempts to write a buffer into this writer, returning how many bytes were written. |
| `poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), io::Error>>` | Attempts to flush the object, ensuring that any buffered data reach their destination. |
| `poll_shutdown(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), io::Error>>` | Initiates or attempts to shut down this writer. |

### trait AsyncBufRead

Extends `AsyncRead` with methods for buffered reading. This is useful for more complex parsing, such as reading line by line.

**Key Methods**

| Method | Description |
|---|---|
| `poll_fill_buf(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<&[u8]>>` | Attempts to fill the internal buffer with more data, returning a slice of the available bytes. |
| `consume(self: Pin<&mut Self>, amt: usize)` | Informs the buffer that `amt` bytes have been consumed from the buffer and should not be returned again. |

### trait AsyncSeek

This trait provides asynchronous seeking capabilities, analogous to `std::io::Seek`.

**Key Methods**

| Method | Description |
|---|---|
| `start_seek(self: Pin<&mut Self>, position: SeekFrom) -> io::Result<()>` | Submits a seek operation to a specified offset. |
| `poll_complete(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<u64>>` | Waits for a pending seek operation to complete, returning the new position. |

## Utilities

### Splitting and Joining I/O

Tokio provides utilities to split a single I/O resource into separate read and write handles, or to join separate read and write handles into a single resource.

<x-cards>
  <x-card data-title="fn split()" data-icon="lucide:git-pull-request-arrow">
    Splits a single value that implements both `AsyncRead` and `AsyncWrite` into a `ReadHalf` and a `WriteHalf`. This is useful for passing the two halves to different tasks.
  </x-card>
  <x-card data-title="fn join()" data-icon="lucide:git-merge">
    Joins a separate `AsyncRead` value and an `AsyncWrite` value into a single `Join` handle that implements both traits.
  </x-card>
</x-cards>

### Standard I/O

Tokio provides asynchronous APIs for standard input, output, and error streams.

- `stdin()`: Returns a handle to the standard input stream.
- `stdout()`: Returns a handle to the standard output stream.
- `stderr()`: Returns a handle to the standard error stream.

> **Note:** These functions must be called from within the context of a Tokio runtime.

### Interoperability with Streams and Sinks

For more advanced use cases, it can be convenient to adapt an `AsyncRead` or `AsyncWrite` into a `Stream` or `Sink`. The [tokio-util](https://docs.rs/tokio-util) crate provides adapters for this purpose:

- **`ReaderStream`**: Converts an `AsyncRead` into a `Stream` of byte chunks.
- **`StreamReader`**: Converts a `Stream` of byte chunks into an `AsyncRead`.
- **`Decoder` and `Encoder`**: Traits for building framed protocols, transforming a byte stream into a stream of structured messages and vice versa.

### Re-exports from `std::io`

For convenience, this module re-exports the following common types from `std::io`:

- `Error`
- `ErrorKind`
- `Result`
- `SeekFrom`

---

With a solid understanding of Tokio's I/O primitives, you can now explore related modules for specific tasks:

<x-cards>
  <x-card data-title="Networking" data-icon="lucide:network" data-href="/api/net">
    Explore asynchronous TCP, UDP, and Unix sockets for network communication.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder-git-2" data-href="/api/fs">
    Learn about asynchronous file and filesystem manipulation operations.
  </x-card>
</x-cards>