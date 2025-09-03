# Filesystem

This module provides asynchronous utilities for working with the file system. This includes reading from and writing to files, as well as manipulating directories.

Be aware that most operating systems do not provide asynchronous file system APIs. Because of this, Tokio will use ordinary blocking file operations behind the scenes. This is done using a dedicated thread pool to run them in the background. For more details on this, see the [Tasks & Scheduling](./concepts-tasks.md) documentation.

The `tokio::fs` module should only be used for ordinary files. Trying to use it with special files, such as a named pipe on Linux, can result in surprising behavior. For special files, you should use a dedicated type like `tokio::net::unix::pipe` or `AsyncFd` instead.

## Quick Start: Reading and Writing Files

The easiest way to use this module is with the utility functions that operate on entire files at once.

| Function | Description |
|---|---|
| `tokio::fs::read` | Asynchronously reads the entire contents of a file into a `Vec<u8>`. |
| `tokio::fs::read_to_string` | Asynchronously reads the entire contents of a file into a string. |
| `tokio::fs::write` | Asynchronously writes a slice of bytes to a file, overwriting existing content. |

### Reading a file to a string

This example reads the entire contents of `my_file.txt` and prints the number of lines.

```rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

### Writing a file

This example writes a string to `my_file.txt`, overwriting the file if it already exists.

```rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

## The `File` Type

For more complex operations than reading or writing an entire file, the [`File`](#file-struct) struct is the main tool. It implements the `AsyncRead` and `AsyncWrite` traits, allowing for buffered or chunk-based I/O.

**Note:** It is important to use `flush` when writing to a Tokio `File`. Calls to `write` will return before the write has finished. The `flush` method will wait for the write to complete. This is different from `std::fs::File` and is a consequence of using a background thread pool for I/O operations.

### Reading a file in chunks

This example counts the number of lines in a file without loading the entire file into memory.

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

Because Tokio's file I/O uses a thread pool, each operation can have some overhead. To get good performance, it is recommended to batch your operations into as few calls as possible.

Here are some strategies:

1.  **Buffer in Memory:** When creating a file, write the data to a `String` or `Vec<u8>` in memory first, then write the entire buffer to the file in a single call with `tokio::fs::write`.

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

2.  **Use `BufWriter`:** Use `tokio::io::BufWriter` to buffer many small writes into fewer, larger ones. The actual write to the file happens when the buffer is full or when you explicitly call `flush()`.

    ```rust,no_run
    use tokio::fs::File;
    use tokio::io::{AsyncWriteExt, BufWriter};

    # async fn dox() -> std::io::Result<()> {
    let mut file = BufWriter::new(File::create("my_file.txt").await?);

    file.write_all(b"First line.\n").await?;
    file.write_all(b"Second line.\n").await?;
    file.write_all(b"Third line.\n").await?;

    // The actual write and blocking call happens when you flush.
    file.flush().await?;
    # Ok(())
    # }
    ```

3.  **Manual `spawn_blocking`:** For full control, use the standard library's `std::fs` types inside a `tokio::task::spawn_blocking` call.

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

## Key Types and Functions

### Structs

| Struct | Description |
|---|---|
| `File` | A reference to an open file on the filesystem. Implements `AsyncRead`, `AsyncWrite`, and `AsyncSeek`. |
| `OpenOptions` | A builder for configuring how a file is opened. |
| `DirBuilder` | A builder for creating directories with specific options. |
| `ReadDir` | A stream over the entries within a directory. |
| `DirEntry` | An entry in a directory, returned by `ReadDir`. |

### Functions

| Function | Description |
|---|---|
| `canonicalize` | Resolves a path to its canonical, absolute form. |
| `copy` | Copies the contents of one file to another. |
| `create_dir` | Creates a new, empty directory at the provided path. |
| `create_dir_all` | Recursively creates a directory and all of its parent components if they are missing. |
| `hard_link` | Creates a hard link on the filesystem. |
| `metadata` | Queries the file system for metadata about a path. |
| `read` | Reads the entire contents of a file into a byte vector. |
| `read_dir` | Returns a stream over the entries within a directory. |
| `read_link` | Reads a symbolic link, returning the path that it points to. |
| `read_to_string` | Reads the entire contents of a file into a string. |
| `remove_dir` | Removes an empty directory. |
| `remove_dir_all` | Removes a directory at this path, after removing all its contents. |
| `remove_file` | Removes a file. |
| `rename` | Renames a file or directory. |
| `set_permissions` | Changes the permissions found on a file or a directory. |
| `symlink` | Creates a new symbolic link on the filesystem (Unix-only). |
| `symlink_metadata` | Queries metadata about a path without following symbolic links. |
| `try_exists` | Checks if a path exists on the filesystem. |
| `write` | Writes a slice of bytes to a file. |
