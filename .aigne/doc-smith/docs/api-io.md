# I/O

Traits, helpers, and type definitions for asynchronous I/O functionality. This module is the asynchronous version of `std::io`.

This module provides the fundamental building blocks for working with asynchronous input and output in Tokio. The primary components are the `AsyncRead` and `AsyncWrite` traits, which are non-blocking versions of the standard library's `Read` and `Write` traits.

```d2
direction: down

"Core I/O Traits" : {
  shape: package
  "AsyncRead": "Asynchronously reads bytes from a source."
  "AsyncWrite": "Asynchronously writes bytes to a destination."
  "AsyncBufRead": "Trait for buffered asynchronous reading."
  "AsyncSeek": "Trait for asynchronous seeking in streams."
}

"I/O Utilities & Structs" : {
   shape: package
   "BufReader": "Adds buffering to any AsyncRead."
   "BufWriter": "Adds buffering to any AsyncWrite."
   "split()": "Splits a stream into separate readable and writable halves."
   "join()": "Joins a reader and a writer into a single stream."
   "stdin(), stdout(), stderr()": "Standard I/O streams."
}

"Core I/O Traits" -> "I/O Utilities & Structs": "Are used and implemented by"
```

## Core Traits

Tokio's I/O functionality is built around a set of core traits that define the behavior of asynchronous byte streams.

### `AsyncRead`

The `AsyncRead` trait is for sources from which bytes can be asynchronously read. It is the asynchronous equivalent of `std::io::Read`.

Unlike its synchronous counterpart, methods on `AsyncRead` will yield to the Tokio scheduler when I/O is not ready, rather than blocking the thread. This allows other tasks to run while waiting for I/O operations to complete.

The core method of this trait is `poll_read`:

```rust
fn poll_read(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>,
    buf: &mut ReadBuf<'_>,
) -> Poll<io::Result<()>>;
```

- **`Poll::Ready(Ok(()))`**: Indicates that data was successfully read into the buffer. The number of bytes read can be determined by checking the change in `buf.filled().len()`. If the length is unchanged, it implies that the end-of-file (EOF) has been reached.
- **`Poll::Pending`**: Means no data is currently available. The current task is scheduled to be woken up when the I/O resource becomes readable again.
- **`Poll::Ready(Err(e))`**: An I/O error occurred.

End users will typically use the methods provided by [`AsyncReadExt`](#asyncreadext) (like `.read()`) rather than calling `poll_read` directly.

### `AsyncWrite`

The `AsyncWrite` trait is for destinations to which bytes can be asynchronously written. It is the asynchronous equivalent of `std::io::Write`.

It provides three core methods, each returning a `Poll` that will schedule the current task for wakeup if the operation cannot be completed immediately.

| Method | Description |
|---|---|
| `poll_write` | Attempts to write bytes from a buffer into the object. Returns the number of bytes written. |
| `poll_flush` | Attempts to flush any buffered data, ensuring it reaches its destination. |
| `poll_shutdown` | Initiates a graceful shutdown of the writer. This may involve flushing data and performing a shutdown handshake. |

### `AsyncBufRead`

This trait is analogous to `std::io::BufRead` and is implemented by asynchronous readers that have an internal buffer. It provides methods for more efficient, buffered reading, such as reading until a delimiter.

Its core methods are:

| Method | Description |
|---|---|
| `poll_fill_buf` | Attempts to fill the internal buffer with more data from the underlying reader. Returns a slice of the available data. |
| `consume` | Informs the buffer that a certain number of bytes have been consumed from the buffer and should not be returned again. |

### `AsyncSeek`

This trait provides asynchronous seeking capabilities, similar to `std::io::Seek`. It allows the position within a stream to be changed.

| Method | Description |
|---|---|
| `start_seek` | Submits a seek operation to a specified position. Does not block. |
| `poll_complete` | Waits for a pending seek operation to complete, returning the new position from the start of the stream. |

## Extension Traits

Utility methods for `AsyncRead` and `AsyncWrite` are provided via extension traits, which are automatically available for any type that implements the core traits.

### `AsyncReadExt`

Provides convenient, future-based methods like `read`, `read_exact`, and `read_to_end`. These are the primary methods you will use when reading from an I/O resource.

Example using `AsyncReadExt::read`:

```no_run
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

### `AsyncWriteExt`

Provides convenient, future-based methods like `write`, `write_all`, and `flush`.

## Buffered Readers and Writers

To reduce the number of system calls and improve performance, Tokio provides buffered I/O types, similar to the standard library.

- **`BufReader`**: Wraps any `AsyncRead` to provide buffering. It implements `AsyncBufRead`, offering methods like `read_line`.
- **`BufWriter`**: Wraps any `AsyncWrite` to buffer write operations. Data is written to the underlying writer only when the buffer is full or when `flush` is called.

Example with `BufReader`:
```no_run
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

**Important**: When using `BufWriter`, you must call `.flush()` to ensure that any data remaining in the buffer is written to the underlying stream before the writer is dropped.

```no_run
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    let mut writer = BufWriter::new(f);

    writer.write_all(b"some bytes").await?;

    // Flush the buffer to ensure data is written to the file.
    writer.flush().await?;

    Ok(())
}
```

## Utilities

### `split()`

The `split()` function takes a single value that implements `AsyncRead + AsyncWrite` (like a `TcpStream`) and splits it into two separate handles: a `ReadHalf` and a `WriteHalf`. This is useful for moving the readable and writable parts of a stream to different tasks.

- `ReadHalf<T>` implements `AsyncRead`.
- `WriteHalf<T>` implements `AsyncWrite`.

The original stream can be reconstituted by calling `unsplit()` on the `ReadHalf` with its corresponding `WriteHalf`.

### `join()`

The `join()` function is the inverse of `split()`. It takes two separate values, one implementing `AsyncRead` and one implementing `AsyncWrite`, and combines them into a single value that implements both traits.

### Standard I/O

Tokio provides asynchronous handles for standard input, output, and error streams:

- **`stdin()`**: Returns a handle to the standard input of the current process.
- **`stdout()`**: Returns a handle to the standard output.
- **`stderr()`**: Returns a handle to the standard error.

These functions must be called from within the context of a Tokio runtime.

### Re-exports

For convenience, this module re-exports common types from `std::io`, including `Error`, `ErrorKind`, `Result`, and `SeekFrom`.