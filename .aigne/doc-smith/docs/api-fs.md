# Filesystem

This module provides asynchronous utilities for file and filesystem manipulation. It allows you to work with the file system without blocking the Tokio runtime.

Be aware that most operating systems do not provide asynchronous file system APIs. Because of this, Tokio uses a thread pool (`spawn_blocking`) to run standard blocking file operations in the background. This approach ensures that your asynchronous tasks remain non-blocking, but it's important to understand the performance implications.

**Note:** The `tokio::fs` module is designed for ordinary files. Using it with special files like named pipes on Linux can lead to unexpected behavior, such as hangs. For these cases, use dedicated types like [`tokio::net::unix::pipe`](./api-net.md) or `AsyncFd`.

## Key Concepts

### Basic Usage

For simple tasks like reading or writing an entire file, Tokio provides convenient top-level functions.

**Reading a file to a string:**

```rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

**Writing a byte slice to a file:**

```rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

### The `File` Type

For more advanced operations like reading or writing in chunks, use the [`File`](#file) struct. It implements the `AsyncRead` and `AsyncWrite` traits.

When using `File` for writing, it's important to call `flush()` to ensure that all buffered data is written to the underlying file. This is because writes are performed in the background and may not be complete when the `write` method returns.

**Writing to a `File` line-by-line:**

```rust
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

### Performance Tuning

Since file operations run on a separate thread pool, each call has some overhead. To optimize performance, it's best to batch operations into as few calls as possible.

1.  **Buffer in Memory**: For writes, build the content in a `String` or `Vec<u8>` first, then write it all at once with `tokio::fs::write`.

    ```rust
    # async fn dox() -> std::io::Result<()> {
    let mut contents = String::new();

    contents.push_str("First line.\n");
    contents.push_str("Second line.\n");
    contents.push_str("Third line.\n");

    tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
    # Ok(())
    # }
    ```

2.  **Use `BufWriter`**: Wrap your `File` in a `BufWriter` to buffer small writes into larger ones. The actual write operation is deferred until the buffer is full or `flush()` is called.

    ```rust
    use tokio::fs::File;
    use tokio::io::{AsyncWriteExt, BufWriter};

    # async fn dox() -> std::io::Result<()> {
    let mut file = BufWriter::new(File::create("my_file.txt").await?);

    file.write_all(b"First line.\n").await?;
    file.write_all(b"Second line.\n").await?;
    file.write_all(b"Third line.\n").await?;

    // The actual write and spawn_blocking call happens here.
    file.flush().await?;
    # Ok(())
    # }
    ```

3.  **Use `spawn_blocking` Manually**: For maximum control, perform standard library file I/O within a `spawn_blocking` call.

    ```rust
    use std::fs::File;
    use std::io::{self, Write};
    use tokio::task::spawn_blocking;

    # async fn dox() -> std::io::Result<()> {
    spawn_blocking(move || {
        let mut file = File::create("my_file.txt")?;

        file.write_all(b"First line.\n")?;
        file.write_all(b"Second line.\n")?;
        file.write_all(b"Third line.\n")?;

        // std::fs::File does not need flush.
        io::Result::Ok(())
    }).await.unwrap()?;
    # Ok(())
    # }
    ```

## Structs

<x-cards data-columns="2">
  <x-card data-title="File" data-icon="lucide:file-text">
    A reference to an open file on the filesystem, implementing `AsyncRead`, `AsyncWrite`, and `AsyncSeek`.
  </x-card>
  <x-card data-title="OpenOptions" data-icon="lucide:settings-2">
    A builder for configuring how a file is opened with specific permissions and modes.
  </x-card>
  <x-card data-title="ReadDir" data-icon="lucide:folder-open">
    A stream over the entries within a directory.
  </x-card>
  <x-card data-title="DirEntry" data-icon="lucide:file">
    An entry found in a directory, returned by `ReadDir`.
  </x-card>
  <x-card data-title="DirBuilder" data-icon="lucide:folder-plus">
    A builder for creating directories with specific options.
  </x-card>
</x-cards>

## Functions

| Function | Description |
| --- | --- |
| `canonicalize(path)` | Resolves a path to its canonical, absolute form. |
| `copy(from, to)` | Asynchronously copies the contents of one file to another. |
| `create_dir(path)` | Creates a new, empty directory at the specified path. |
| `create_dir_all(path)` | Recursively creates a directory and all of its parent components if they are missing. |
| `hard_link(src, dst)` | Creates a hard link on the filesystem. |
| `metadata(path)` | Reads the metadata for a path without following symlinks. |
| `read(path)` | Reads the entire contents of a file into a `Vec<u8>`. |
| `read_dir(path)` | Returns a stream over the entries within a directory. |
| `read_link(path)` | Reads a symbolic link, returning the path that it points to. |
| `read_to_string(path)` | Reads the entire contents of a file into a string. |
| `remove_dir(path)` | Removes an empty directory. |
| `remove_dir_all(path)` | Removes a directory at this path, after removing all its contents. |
| `remove_file(path)` | Removes a file. |
| `rename(from, to)` | Renames a file or directory. |
| `set_permissions(path, perm)` | Changes the permissions of a file or directory. |
| `symlink(src, dst)` | Creates a new symbolic link on the filesystem. (Unix-only) |
| `symlink_metadata(path)` | Reads the metadata of a file, but does not follow symbolic links. |
| `try_exists(path)` | Checks if a path exists in the filesystem. |
| `write(path, contents)` | Writes a slice of bytes to a file, overwriting the existing file. |

---

## `File`

A reference to an open file on the filesystem. It provides methods for reading, writing, and seeking.

### Creating and Opening Files

**`File::open(path)`**: Opens a file in read-only mode.

```rust
use tokio::fs::File;
use tokio::io::AsyncReadExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::open("foo.txt").await?;
let mut contents = vec![];
file.read_to_end(&mut contents).await?;
println!("Read {} bytes", contents.len());
# Ok(())
# }
```

**`File::create(path)`**: Opens a file in write-only mode. Creates the file if it doesn't exist, and truncates it if it does.

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::create("foo.txt").await?;
file.write_all(b"hello, world!").await?;
# Ok(())
# }
```

**`File::options()`**: Returns an `OpenOptions` builder for more advanced configuration.

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut f = File::options().append(true).open("example.log").await?;
f.write_all(b"new line\n").await?;
# Ok(())
# }
```

### File Operations

**`metadata()`**: Queries metadata about the underlying file.

```rust
use tokio::fs::File;

# async fn dox() -> std::io::Result<()> {
let file = File::open("foo.txt").await?;
let metadata = file.metadata().await?;
println!("Is read-only: {}", metadata.permissions().readonly());
# Ok(())
# }
```

**`sync_all()`**: Attempts to sync all OS-internal metadata and data to disk.

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::create("foo.txt").await?;
file.write_all(b"hello, world!").await?;
file.sync_all().await?;
# Ok(())
# }
```

**`set_len(size)`**: Truncates or extends the underlying file to the specified size.

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::create("foo.txt").await?;
file.write_all(b"hello, world!").await?;
file.set_len(10).await?;
# Ok(())
# }
```

---

## `OpenOptions`

Provides a builder for configuring how a file is opened. This allows for fine-grained control over read, write, append, create, and truncate modes.

### Example: Opening a file for both reading and writing

This example opens a file for both reading and writing. If the file doesn't exist, it will be created.

```rust
use tokio::fs::OpenOptions;
use std::io;

# async fn main() -> io::Result<()> {
let file = OpenOptions::new()
    .read(true)
    .write(true)
    .create(true)
    .open("foo.txt")
    .await?;

Ok(())
}
```

### Configuration Methods

| Method | Description |
|---|---|
| `read(bool)` | Sets the option for read access. |
| `write(bool)` | Sets the option for write access. |
| `append(bool)` | Sets the option for append mode. |
| `truncate(bool)` | Sets the option for truncating the file. Requires write access. |
| `create(bool)` | Sets the option to create a new file if it does not exist. Requires write or append access. |
| `create_new(bool)` | Sets the option to create a new file, failing if it already exists. Atomic operation. |

---

## `ReadDir` and `DirEntry`

These structs are used to iterate over the entries in a directory.

### Example: Listing directory contents

The `read_dir` function returns a `ReadDir` stream. You can call `next_entry()` in a loop to process each `DirEntry`.

```rust
use tokio::fs;
use std::io;

# async fn dox() -> io::Result<()> {
let mut entries = fs::read_dir(".").await?;

while let Some(entry) = entries.next_entry().await? {
    println!("Found: {:?}", entry.path());
}
# Ok(())
# }
```

### `DirEntry` Methods

A `DirEntry` represents a single entry in a directory and provides methods to get more information.

| Method | Description |
|---|---|
| `path()` | Returns the full path to the file. |
| `file_name()` | Returns the bare file name of the entry. |
| `metadata()` | Returns the metadata for the file this entry points at. Does not follow symlinks. |
| `file_type()` | Returns the `FileType` for the file this entry points at. Does not follow symlinks. |

Here is an example that inspects the file type of each entry:

```rust
use tokio::fs;

# async fn dox() -> std::io::Result<()> {
let mut entries = fs::read_dir(".").await?;

while let Some(entry) = entries.next_entry().await? {
    if let Ok(file_type) = entry.file_type().await {
        // Now let's show our entry's file type!
        println!("{:?}: {:?}", entry.path(), file_type);
    } else {
        println!("Couldn't get file type for {:?}", entry.path());
    }
}
# Ok(())
# }
```

This provides a convenient way to inspect directory contents without blocking the async runtime.