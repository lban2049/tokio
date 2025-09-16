# Filesystem

Learn how to perform asynchronous file operations with `tokio::fs`, including reading, writing, and manipulating files and directories. This module provides asynchronous versions of standard library functions for filesystem I/O.

### How It Works

Be aware that most operating systems do not provide native asynchronous file system APIs. Because of this, `tokio::fs` uses a thread pool managed by the Tokio runtime to execute standard, blocking file operations without blocking the main asynchronous tasks. This is achieved via the [`spawn_blocking`](./tasks-scheduling-spawning.md) mechanism.

While this approach provides an asynchronous API, it carries performance implications. Each operation may involve sending the task to a blocking thread, which has some overhead. For optimal performance, it's recommended to batch operations into as few `spawn_blocking` calls as possible.

> **Note:** The `tokio::fs` module is designed for ordinary files. Using it with special files like named pipes on Linux can lead to unexpected behavior, such as hangs. For special files, consider dedicated types like [`tokio::net::unix::pipe`](./io-networking.md) or `AsyncFd`.

## Quick Start: Reading and Writing Entire Files

The most straightforward way to work with files is to use the utility functions that operate on the entire file at once.

### Read a File to a String

To read the entire contents of a file into a string, use `tokio::fs::read_to_string`.

```rust Read a file icon=logos:rust
async fn read_file() -> std::io::Result<()> {
    let contents = tokio::fs::read_to_string("my_file.txt").await?;
    println!("File has {} lines.", contents.lines().count());
    Ok(())
}
```

### Write a Slice to a File

To write the entire contents of a byte slice to a file, use `tokio::fs::write`. If the file already exists, its contents are overwritten.

```rust Write to a file icon=logos:rust
async fn write_file() -> std::io::Result<()> {
    let contents = "First line.\nSecond line.\nThird line.\n";
    tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
    Ok(())
}
```

## Advanced Usage with `File`

For more granular control, such as reading a file in chunks or streaming writes, use the `tokio::fs::File` struct. It implements the `AsyncRead` and `AsyncWrite` traits, allowing you to use it with the rich set of combinators from `tokio::io`.

### Creating and Writing to a `File`

You can create a new file or open an existing one using `File::create` or `File::open`.

**Important:** When writing to a `File`, you should almost always call `flush()` when you are done. Tokio's `File` buffers writes internally. The `write_all` call will return before the data is passed to the operating system. `flush()` waits for the underlying write operation to complete.

```rust Line-by-line writing icon=logos:rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

async fn write_line_by_line() -> std::io::Result<()> {
    let mut file = File::create("my_file.txt").await?;

    file.write_all(b"First line.\n").await?;
    file.write_all(b"Second line.\n").await?;
    file.write_all(b"Third line.\n").await?;

    // Remember to call `flush` after writing!
    file.flush().await?;
    Ok(())
}
```

### Reading from a `File`

To read from a file, open it with `File::open` and use methods from `AsyncReadExt`, like `read`.

This example counts the number of lines in a file without loading the entire file into memory, which is useful for very large files.

```rust Reading in chunks icon=logos:rust
use tokio::fs::File;
use tokio::io::AsyncReadExt;

async fn count_lines() -> std::io::Result<()> {
    let mut file = File::open("my_file.txt").await?;

    let mut chunk = vec![0; 4096];
    let mut number_of_lines = 0;
    loop {
        let len = file.read(&mut chunk).await?;
        if len == 0 {
            // A length of zero means the end of the file has been reached.
            break;
        }
        for &b in &chunk[..len] {
            if b == b'\n' {
                number_of_lines += 1;
            }
        }
    }

    println!("File has {} lines.", number_of_lines);
    Ok(())
}
```

## Performance Tuning Strategies

Because `tokio::fs` uses a blocking thread pool, minimizing the number of distinct I/O operations is key to good performance. Here are some strategies to achieve that.

<x-cards data-columns="1">
  <x-card data-title="In-Memory Buffering" data-icon="lucide:memory-stick" data-horizontal="true">
    For writes, build the complete file content in a `String` or `Vec<u8>` in memory first, then write it all in a single call to `tokio::fs::write`. This ensures only one `spawn_blocking` call is made for the entire operation.
  </x-card>
  <x-card data-title="Use BufReader and BufWriter" data-icon="lucide:file-cog" data-horizontal="true">
    Wrap your `File` in a `tokio::io::BufReader` or `tokio::io::BufWriter`. These wrappers aggregate many small read/write calls into fewer, larger operations on the underlying file, reducing the overhead of context switching to blocking threads.
  </x-card>
  <x-card data-title="Manual spawn_blocking" data-icon="lucide:cpu" data-horizontal="true">
    For complex sequences of file operations, you can perform them all within a single `tokio::task::spawn_blocking` call using the standard `std::fs` library. This gives you maximum control over batching.
  </x-card>
</x-cards>

### Example: Using `BufWriter`

`BufWriter` will buffer the writes in memory and only perform the actual `spawn_blocking` call when its internal buffer is full or when `flush()` is called explicitly.

```rust icon=logos:rust
use tokio::fs::File;
use tokio::io::{AsyncWriteExt, BufWriter};

async fn buffered_write() -> std::io::Result<()> {
    let file = File::create("my_file.txt").await?;
    let mut file = BufWriter::new(file);

    file.write_all(b"First line.\n").await?;
    file.write_all(b"Second line.\n").await?;

    // The actual write and spawn_blocking call happens here.
    file.flush().await?;
    Ok(())
}
```

## Opening Files with `OpenOptions`

For more control over how files are opened, you can use the `OpenOptions` builder. This allows you to configure read, write, append, create, and truncate modes.

```rust OpenOptions example icon=logos:rust
use tokio::fs::OpenOptions;
use tokio::io::AsyncWriteExt;

async fn append_to_file() -> std::io::Result<()> {
    let mut file = OpenOptions::new()
        .write(true)
        .append(true)
        .open("my_log.txt")
        .await?;

    file.write_all(b"Appending a new log entry.\n").await?;
    file.flush().await?;

    Ok(())
}
```

## Other Filesystem Operations

Beyond reading and writing files, the `tokio::fs` module provides a comprehensive set of functions for interacting with the filesystem, including:

<x-cards data-columns="2">
  <x-card data-title="create_dir / create_dir_all" data-icon="lucide:folder-plus">
    Create a new directory or a directory and all its parents.
  </x-card>
  <x-card data-title="remove_dir / remove_file" data-icon="lucide:trash-2">
    Remove an empty directory or a file.
  </x-card>
  <x-card data-title="read_dir" data-icon="lucide:folder-search">
    Read the contents of a directory.
  </x-card>
  <x-card data-title="rename / copy" data-icon="lucide:copy-check">
    Rename or copy a file.
  </x-card>
  <x-card data-title="symlink / read_link" data-icon="lucide:link-2">
    Create or read a symbolic link.
  </x-card>
  <x-card data-title="metadata / set_permissions" data-icon="lucide:file-lock">
    Read or modify file metadata and permissions.
  </x-card>
</x-cards>

---

Now that you understand how to perform asynchronous file I/O, you might be interested in managing other system resources. Continue to the [Child Processes](./io-process.md) section to learn how to spawn and manage subprocesses asynchronously.