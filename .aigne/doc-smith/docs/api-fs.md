# Filesystem

API documentation for asynchronous file and filesystem manipulation operations.

This module provides asynchronous utilities for interacting with the file system. It includes functions for reading and writing files, managing directories, and inspecting file metadata.

Be aware that most operating systems do not provide asynchronous file system APIs. Because of this, Tokio uses a blocking thread pool (`spawn_blocking`) to execute file operations behind the scenes. This design allows your asynchronous tasks to remain non-blocking while I/O operations are performed concurrently on a separate set of threads.

**Note:** The `tokio::fs` module is designed for ordinary files. Using it with special files like named pipes on Linux can lead to unexpected behavior, such as hangs. For special files, consider using dedicated types like `tokio::net::unix::pipe` or `AsyncFd`.

## Usage

There are two main ways to use this module: high-level utility functions for simple tasks and the `File` struct for more granular control.

### Simple File Operations

For common tasks like reading or writing an entire file at once, the utility functions are the easiest approach.

- [`tokio::fs::read`](#functions): Reads the entire file into a `Vec<u8>`.
- [`tokio::fs::read_to_string`](#functions): Reads the entire file into a `String`.
- [`tokio::fs::write`](#functions): Writes a slice of bytes to a file, overwriting existing content.

**Example: Reading a file to a string**

```rust,no_run Reading a file icon=logos:rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

**Example: Writing a string to a file**

```rust,no_run Writing a file icon=logos:rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

### Using `File` for Granular Control

For more complex scenarios, such as streaming data or seeking to specific positions, the [`File`](#file) struct provides more control. It implements the `AsyncRead`, `AsyncWrite`, and `AsyncSeek` traits.

**Important:** When writing to a `File`, it is crucial to call `flush()` to ensure all buffered data is written. Unlike `std::fs::File`, Tokio's `File` buffers writes and performs them in the background. The `flush()` method waits for these background operations to complete.

**Example: Counting lines without loading the whole file**

```rust,no_run Reading a file in chunks icon=logos:rust
use tokio::fs::File;
use tokio::io::AsyncReadExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::open("my_file.txt").await?;

let mut chunk = vec![0; 4096];
let mut number_of_lines = 0;
loop {
    let len = file.read(&mut chunk).await?;
    if len == 0 {
        // Length of zero means end of file.
        break;
    }
    for &b in &chunk[..len] {
        if b == b'\n' {
            number_of_lines += 1;
        }
    }
}

println!("File has {} lines.", number_of_lines);
# Ok(())
# }
```

## Performance Tuning

Since Tokio's file operations use `spawn_blocking`, each call has some overhead. To achieve the best performance, it's recommended to batch operations into as few `spawn_blocking` calls as possible.

Here are some strategies:

1.  **Buffer in memory:** For creating files, build the content in a `String` or `Vec<u8>` first, then write it all at once with `tokio::fs::write`.

    ```rust,no_run Buffering in memory icon=logos:rust
    # async fn dox() -> std::io::Result<()> {
    let mut contents = String::new();

    contents.push_str("First line.\n");
    contents.push_str("Second line.\n");
    contents.push_str("Third line.\n");

    tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
    # Ok(())
    # }
    ```

2.  **Use `BufWriter`:** Wrap your `File` in a `tokio::io::BufWriter` to buffer many small writes into fewer, larger writes to the underlying file.

    ```rust,no_run Using BufWriter icon=logos:rust
    use tokio::fs::File;
    use tokio::io::{AsyncWriteExt, BufWriter};

    # async fn dox() -> std::io::Result<()> {
    let mut file = BufWriter::new(File::create("my_file.txt").await?);

    file.write_all(b"First line.\n").await?;
    file.write_all(b"Second line.\n").await?;
    file.write_all(b"Third line.\n").await?;

    // The actual write and spawn_blocking call happens when you flush.
    file.flush().await?;
    # Ok(())
    # }
    ```

3.  **Manual `spawn_blocking`:** For complex sequences of operations, you can perform them all within a single `spawn_blocking` call using the standard library's `std::fs` types.

    ```rust,no_run Manual spawn_blocking icon=logos:rust
    use std::fs::File;
    use std::io::{self, Write};
    use tokio::task::spawn_blocking;

    # async fn dox() -> std::io::Result<()> {
    spawn_blocking(move || {
        let mut file = File::create("my_file.txt")?;

        file.write_all(b"First line.\n")?;
        file.write_all(b"Second line.\n")?;
        file.write_all(b"Third line.\n")?;

        io::Result::Ok(())
    }).await.unwrap()?;
    # Ok(())
    # }
    ```

## API Reference

### Structs

<x-cards data-columns="2">
  <x-card data-title="File" data-icon="lucide:file">
    An asynchronously accessible file. Implements `AsyncRead`, `AsyncWrite`, and `AsyncSeek` for I/O operations.
  </x-card>
  <x-card data-title="OpenOptions" data-icon="lucide:settings-2">
    A builder for customizing how a file is opened, allowing fine-grained control over read, write, create, and append modes.
  </x-card>
  <x-card data-title="ReadDir" data-icon="lucide:folder-open">
    A stream that iterates over the entries in a directory, yielding `DirEntry` instances.
  </x-card>
  <x-card data-title="DirEntry" data-icon="lucide:file-text">
    Represents a single entry within a directory, providing access to its path, name, and metadata.
  </x-card>
  <x-card data-title="DirBuilder" data-icon="lucide:folder-plus">
    A builder for creating directories with specific options, such as setting the mode on Unix platforms.
  </x-card>
</x-cards>

### Functions

This module provides a number of top-level functions for common filesystem operations.

| Function | Description |
|---|---|
| `canonicalize(path)` | Resolves a path to its canonical, absolute form. |
| `copy(from, to)` | Copies the contents of one file to another asynchronously. |
| `create_dir(path)` | Creates a new, empty directory at the specified path. |
| `create_dir_all(path)` | Recursively creates a directory and all of its parent components if they are missing. |
| `hard_link(src, dst)` | Creates a hard link on the filesystem. |
| `metadata(path)` | Reads the metadata for a path, following symbolic links. |
| `read(path)` | Reads the entire contents of a file into a bytes vector. |
| `read_dir(path)` | Returns a stream over the entries within a directory. |
| `read_link(path)` | Reads a symbolic link, returning the path that the link points to. |
| `read_to_string(path)` | Reads the entire contents of a file into a string. |
| `remove_dir(path)` | Removes an empty directory. |
| `remove_dir_all(path)` | Removes a directory and all its contents recursively. |
| `remove_file(path)` | Removes a file. |
| `rename(from, to)` | Renames or moves a file or directory. |
| `set_permissions(path, perm)` | Changes the permissions of a file or directory. |
| `symlink(src, dst)` | Creates a new symbolic link on the filesystem (Unix-only). |
| `symlink_metadata(path)` | Reads the metadata for a path without following symbolic links. |
| `try_exists(path)` | Asynchronously checks if a path exists. |
| `write(path, contents)` | Writes a slice of bytes to a file, creating it if it doesn't exist and overwriting it if it does. |