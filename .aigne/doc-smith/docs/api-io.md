# I/O

Traits, helpers, and type definitions for asynchronous I/O functionality. This module is the asynchronous version of `std::io`.

Tokio's I/O model is built around a few core traits: `AsyncRead` and `AsyncWrite`. These are asynchronous versions of the `Read` and `Write` traits in the standard library. The key difference is that their methods do not block the current thread. Instead of blocking, they return `Poll::Pending` and schedule the current task to be woken up when the I/O operation can continue.

## Core Traits

These traits form the foundation for all asynchronous I/O operations in Tokio.

<x-cards>
  <x-card data-title="AsyncRead" data-icon="lucide:book-open">
    Provides the core `poll_read` method for asynchronously reading bytes from a source. Most users will interact with this trait via the convenient methods provided by `AsyncReadExt`.
  </x-card>
  <x-card data-title="AsyncWrite" data-icon="lucide:edit-3">
    Provides the core methods for asynchronously writing bytes to a destination, including `poll_write`, `poll_flush`, and `poll_shutdown`. The `AsyncWriteExt` trait provides more ergonomic methods for writing.
  </x-card>
  <x-card data-title="AsyncBufRead" data-icon="lucide:file-text">
    An asynchronous version of `std::io::BufRead` for reading bytes from a buffered source. It provides methods like `poll_fill_buf` and is complemented by the `AsyncBufReadExt` trait for helpers like `read_line`.
  </x-card>
  <x-card data-title="AsyncSeek" data-icon="lucide:move-horizontal">
    An asynchronous version of `std::io::Seek` for moving the cursor within a byte stream. It uses a two-step process: `start_seek` and `poll_complete`.
  </x-card>
</x-cards>

### Reading and Writing Data

While the core traits provide the low-level polling methods, you'll typically use the helper methods on the `AsyncReadExt` and `AsyncWriteExt` extension traits. These traits are automatically available for any type that implements `AsyncRead` or `AsyncWrite`.

For example, you can use the `read` method from `AsyncReadExt` to read data from a file:

```rust Reading from a file icon=logos:rust
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

## Buffered Readers and Writers

For efficiency and convenience, Tokio provides buffered I/O types, similar to `std::io`. These wrappers reduce the number of system calls and provide helpful methods for common tasks.

`BufReader` adds buffering to any async reader and, along with `AsyncBufReadExt`, enables methods like `read_line`.

```rust Reading a line with BufReader icon=logos:rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let reader = BufReader::new(f);
    let mut lines = reader.lines();
    
    while let Some(line) = lines.next_line().await? {
        println!("{}", line);
    }

    Ok(())
}
```

`BufWriter` buffers writes to any async writer. It's crucial to call `flush` to ensure that all buffered data is written to the underlying writer.

```rust Using BufWriter icon=logos:rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write some bytes to the buffer.
        writer.write_all(b"some bytes").await?;

        // Flush the buffer to ensure the data is written.
        writer.flush().await?;

    } // The buffer is flushed again on drop, but it's best to be explicit.

    Ok(())
}
```

## Utilities

This module also includes several utilities for working with I/O streams.

<x-cards data-columns="3">
  <x-card data-title="split" data-icon="lucide:split">
    Splits a single value that is both `AsyncRead` and `AsyncWrite` into a separate reader half (`ReadHalf`) and writer half (`WriteHalf`).
  </x-card>
  <x-card data-title="join" data-icon="lucide:merge">
    The inverse of `split`. Joins an `AsyncRead` and an `AsyncWrite` value into a single handle that implements both traits.
  </x-card>
  <x-card data-title="Standard I/O" data-icon="lucide:terminal">
    The `stdin`, `stdout`, and `stderr` functions provide asynchronous handles to the standard I/O streams of the process. These must be called from within a Tokio runtime.
  </x-card>
</x-cards>

## `std` Re-exports

For convenience, the following common types are re-exported from `std::io`:

*   `Error`
*   `ErrorKind`
*   `Result`
*   `SeekFrom`

---

With the fundamental I/O traits covered, explore concrete implementations for different use cases:

<x-cards>
  <x-card data-title="Networking" data-icon="lucide:network" data-href="/api/net">
    For asynchronous TCP, UDP, and Unix sockets.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder" data-href="/api/fs">
    For asynchronous file and filesystem operations.
  </x-card>
  <x-card data-title="Processes" data-icon="lucide:cpu" data-href="/api/process">
    For interacting with child process I/O streams.
  </x-card>
</x-cards>
