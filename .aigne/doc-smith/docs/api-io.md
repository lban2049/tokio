# I/O

Traits, helpers, and type definitions for asynchronous I/O functionality. This module is the asynchronous version of `std::io`.

This module provides asynchronous equivalents of the standard library's `Read`, `Write`, `BufRead`, and `Seek` traits. It also includes utilities for working with these traits, handling standard I/O streams, and manipulating I/O objects.

## Core Concepts

The fundamental components of Tokio's I/O system are the `AsyncRead` and `AsyncWrite` traits. These traits provide the most general interface for reading and writing bytes asynchronously.

Unlike their counterparts in the standard library, methods on these traits will yield to the Tokio scheduler when I/O is not ready, rather than blocking the current thread. This allows other tasks to run while the application waits for I/O operations to complete.

Most of the convenient utility methods for I/O operations are not on the core traits themselves. Instead, they are provided by the extension traits `AsyncReadExt`, `AsyncWriteExt`, `AsyncBufReadExt`, and `AsyncSeekExt`, which are automatically implemented for all types that implement the corresponding base trait.

For example, reading from a file asynchronously looks very similar to the synchronous version:

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

## Core I/O Traits

Tokio's I/O system is built around a few core traits that define the behavior of asynchronous byte streams.

<x-cards>
  <x-card data-title="AsyncRead" data-icon="lucide:arrow-down-circle">
    Provides the `poll_read` method for asynchronously reading bytes from a source. When data is not available, it registers the current task to be woken up when the source becomes readable again.
  </x-card>
  <x-card data-title="AsyncWrite" data-icon="lucide:arrow-up-circle">
    Provides `poll_write`, `poll_flush`, and `poll_shutdown` methods for asynchronously writing bytes to a destination, flushing internal buffers, and gracefully shutting down the connection.
  </x-card>
  <x-card data-title="AsyncBufRead" data-icon="lucide:book-open">
    An asynchronous version of `std::io::BufRead`. It allows reading from an internal buffer, which can reduce the number of system calls and improve performance.
  </x-card>
  <x-card data-title="AsyncSeek" data-icon="lucide:move-horizontal">
    An asynchronous version of `std::io::Seek`. It provides methods to change the current position within a stream of bytes.
  </x-card>
</x-cards>

## Buffered Readers and Writers

Directly using byte-based interfaces can be inefficient due to frequent system calls. To mitigate this, Tokio provides buffered I/O types, similar to `std::io`.

-   **`BufReader`**: Wraps an `AsyncRead` to provide buffered reading. It introduces helpful methods like `read_line` via the `AsyncBufReadExt` trait.
-   **`BufWriter`**: Wraps an `AsyncWrite` to buffer write operations. It's important to call `flush()` on a `BufWriter` to ensure all buffered data is written to the underlying writer before it is dropped.

### Reading Lines from a File

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

### Buffering Writes

```rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write a byte to the buffer.
        writer.write_all(&[42u8]).await?;

        // Flush the buffer to ensure data is written to the file.
        writer.flush().await?;

    } // The buffer is discarded on drop unless flushed.

    Ok(())
}
```

## Standard I/O

Tokio provides asynchronous APIs for standard input, output, and error streams. These functions return handles that implement `AsyncRead` and `AsyncWrite`.

-   `stdin()`: Returns a handle to the standard input of the current process.
-   `stdout()`: Returns a handle to the standard output of the current process.
-   `stderr()`: Returns a handle to the standard error of the current process.

**Note:** These APIs must be called from within the context of a Tokio runtime.

## Utilities

This module includes several utility functions and structs for common I/O tasks.

<x-cards>
  <x-card data-title="split()" data-icon="lucide:git-pull-request-arrow">
    Splits a single value that implements both `AsyncRead` and `AsyncWrite` into separate readable (`ReadHalf`) and writable (`WriteHalf`) handles.
  </x-card>
  <x-card data-title="join()" data-icon="lucide:git-merge">
    Joins a reader and a writer into a single handle that implements both `AsyncRead` and `AsyncWrite`.
  </x-card>
  <x-card data-title="copy()" data-icon="lucide:copy">
    Asynchronously copies the entire contents of a reader into a writer.
  </x-card>
  <x-card data-title="empty()", "sink()", "repeat()" data-icon="lucide:box">
    Provides specialized I/O objects: `empty()` is a reader that is always at EOF, `sink()` is a writer that endlessly accepts and discards data, and `repeat()` is a reader that endlessly yields a specific byte.
  </x-card>
</x-cards>

## `std` Re-exports

For convenience, several common types from `std::io` are re-exported. This allows you to use `tokio::io` without needing to import `std::io` separately for these types.

-   `Error`
-   `ErrorKind`
-   `Result`
-   `SeekFrom`