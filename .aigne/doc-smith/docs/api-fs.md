# Filesystem

Asynchronous utilities for file and filesystem manipulation.

This module provides utility methods for performing asynchronous I/O with the filesystem. This includes reading and writing to files, as well as manipulating directories.

It's important to understand that most operating systems do not provide native asynchronous file system APIs. Consequently, Tokio executes standard blocking file operations on a dedicated thread pool using `spawn_blocking`. While Tokio may adopt newer asynchronous APIs like `io_uring` in the future, the current implementation relies on this thread-based approach.

**Note:** The `tokio::fs` module is designed for ordinary files. Using it with special files, such as named pipes on Linux, can lead to unexpected behavior like hangs. For these cases, use dedicated types like `tokio::net::unix::pipe` or `AsyncFd`.

## Quick Start: Reading and Writing Entire Files

The most straightforward way to interact with files is through the utility functions that handle the entire file at once.

*   `tokio::fs::read`: Reads the entire file into a `Vec<u8>`.
*   `tokio::fs::read_to_string`: Reads the entire file into a `String`.
*   `tokio::fs::write`: Writes a slice of bytes to a file, overwriting existing content.

### Read a file to a string

```rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

### Write to a file

```rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

## Advanced Usage with `File`

For more complex scenarios, such as streaming data or avoiding loading an entire file into memory, use the `File` struct. It implements the `AsyncRead` and `AsyncWrite` traits for fine-grained I/O operations.

**Important:** When writing to a Tokio `File`, you must call `flush()` to ensure the write operation completes. Because `File` uses `spawn_blocking` internally, `write` calls can return before the data is actually written by the background thread. `flush()` waits for this background operation to finish.

### Reading a file in chunks

This example counts lines without loading the entire file into memory.

```rust,no_run
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

### Writing to a file line-by-line

```rust,no_run
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::create("my_file.txt").await?;

file.write_all(b"First line.\n").await?;
file.write_all(b"Second line.\n").await?;
file.write_all(b"Third line.\n").await?;

// Remember to call `flush` after writing!
file.flush().await?;
# Ok(())
# }
```

## Performance Tuning

Since Tokio's file I/O relies on `spawn_blocking`, each operation can introduce overhead. To achieve good performance, batch your operations into as few `spawn_blocking` calls as possible.

Here are some effective strategies:

1.  **Buffer in memory, then write once:** Collect data in a `String` or `Vec<u8>` and write the entire buffer with a single call to `tokio::fs::write`.

    ```rust,no_run
    # async fn dox() -> std::io::Result<()> {
    let mut contents = String::new();

    contents.push_str("First line.\n");
    contents.push_str("Second line.\n");
    contents.push_str("Third line.\n");

    tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
    # Ok(())
    # }
    ```

2.  **Use `BufWriter`:** `BufWriter` buffers small writes and flushes them as a single larger write, reducing the number of underlying system calls.

    ```rust,no_run
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

3.  **Manual `spawn_blocking`:** For maximum control, perform standard library file I/O inside a `spawn_blocking` call yourself.

    ```rust,no_run
    use std::fs::File;
    use std::io::{self, Write};
    use tokio::task::spawn_blocking;

    # async fn dox() -> std::io::Result<()> {
    spawn_blocking(move || {
        let mut file = File::create("my_file.txt")?;

        file.write_all(b"First line.\n")?;
        file.write_all(b"Second line.\n")?;
        file.write_all(b"Third line.\n")?;

        // Unlike Tokio's file, the std::fs file does
        // not need flush.

        io::Result::Ok(())
    }).await.unwrap()?;
    # Ok(())
    # }
    ```

You can also adjust the amount of data Tokio processes in a single `spawn_blocking` call using `File::set_max_buf_size`.

## Key Structs

<x-cards data-columns="2">
  <x-card data-title="File" data-icon="lucide:file-text">
    An asynchronous handle to an open file on the filesystem.
  </x-card>
  <x-card data-title="OpenOptions" data-icon="lucide:settings-2">
    A builder for configuring how a file is opened with specific options.
  </x-card>
  <x-card data-title="ReadDir" data-icon="lucide:folder-open">
    A stream that yields the entries within a directory.
  </x-card>
  <x-card data-title="DirEntry" data-icon="lucide:file">
    A single entry read from a directory, returned by `ReadDir`.
  </x-card>
  <x-card data-title="DirBuilder" data-icon="lucide:folder-plus">
    A builder for creating directories with specific options, like setting the mode on Unix.
  </x-card>
</x-cards>

## Functions

### File Operations

| Function | Description |
|---|---|
| `copy` | Copies the contents of one file to another asynchronously. |
| `read` | Reads the entire contents of a file into a bytes vector. |
| `read_to_string` | Reads the entire contents of a file into a string. |
| `remove_file` | Removes a file. |
| `write` | Writes a slice of bytes as the entire contents of a file. |

### Directory Operations

| Function | Description |
|---|---|
| `create_dir` | Creates a new, empty directory at the provided path. |
| `create_dir_all` | Recursively creates a directory and all of its parent components if they are missing. |
| `read_dir` | Returns a stream over the entries within a directory. |
| `remove_dir` | Removes an empty directory. |
| `remove_dir_all` | Removes a directory at this path, after removing all its contents. |

### Filesystem Manipulation

| Function | Description |
|---|---|
| `rename` | Renames a file or directory. |
| `hard_link` | Creates a new hard link on the filesystem. |
| `symlink` | Creates a new symbolic link on the filesystem. (Unix-specific) |
| `symlink_dir` | Creates a new directory symbolic link on the filesystem. (Windows-specific) |
| `symlink_file` | Creates a new file symbolic link on the filesystem. (Windows-specific) |

### Metadata and Paths

| Function | Description |
|---|---|
| `canonicalize` | Returns the canonical, absolute form of a path with all intermediate components normalized. |
| `metadata` | Queries the file system metadata for a path, following symlinks. |
| `symlink_metadata` | Queries the metadata of a file without following symbolic links. |
| `read_link` | Reads a symbolic link, returning the path that the link points to. |
| `set_permissions` | Changes the permissions of a file or directory. |
| `try_exists` | Checks if a path exists on the filesystem. |
