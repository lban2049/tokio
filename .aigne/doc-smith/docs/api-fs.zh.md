# 文件系统

此模块提供用于文件和文件系统操作的异步实用工具。它允许你在不阻塞 Tokio 运行时的情况下处理文件系统。

请注意，大多数操作系统不提供异步文件系统 API。因此，Tokio 使用线程池 (`spawn_blocking`) 在后台运行标准的阻塞式文件操作。这种方法可确保你的异步任务保持非阻塞状态，但了解其性能影响非常重要。

**注意：** `tokio::fs` 模块专为普通文件设计。将其用于特殊文件（如 Linux 上的命名管道）可能会导致意外行为，例如挂起。对于这些情况，请使用专用类型，如 [`tokio::net::unix::pipe`](./api-net.md) 或 `AsyncFd`。

## 核心概念

### 基本用法

对于读取或写入整个文件等简单任务，Tokio 提供了便捷的顶层函数。

**将文件读取为字符串：**

```rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

**将字节切片写入文件：**

```rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

### `File` 类型

对于更高级的操作，如分块读取或写入，请使用 [`File`](#file) 结构体。它实现了 `AsyncRead` 和 `AsyncWrite` 特征。

使用 `File` 进行写入时，调用 `flush()` 很重要，以确保所有缓冲数据都已写入底层文件。这是因为写入操作在后台执行，当 `write` 方法返回时可能尚未完成。

**逐行写入 `File`：**

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

### 性能调优

由于文件操作在单独的线程池上运行，每次调用都有一些开销。为优化性能，最好将操作分批处理，以尽可能减少调用次数。

1.  **在内存中缓冲**：对于写入操作，首先在 `String` 或 `Vec<u8>` 中构建内容，然后使用 `tokio::fs::write` 一次性全部写入。

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

2.  **使用 `BufWriter`**：将 `File` 包装在 `BufWriter` 中，以将小的写入操作缓冲为较大的写入操作。实际的写入操作会延迟到缓冲区满或调用 `flush()` 时才执行。

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

3.  **手动使用 `spawn_blocking`**：为了获得最大程度的控制，请在 `spawn_blocking` 调用中执行标准库的文件 I/O 操作。

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

## 结构体

<x-cards data-columns="2">
  <x-card data-title="File" data-icon="lucide:file-text">
    对文件系统上已打开文件的引用，实现了 `AsyncRead`、`AsyncWrite` 和 `AsyncSeek`。
  </x-card>
  <x-card data-title="OpenOptions" data-icon="lucide:settings-2">
    用于配置如何以特定权限和模式打开文件的构建器。
  </x-card>
  <x-card data-title="ReadDir" data-icon="lucide:folder-open">
    目录中条目的流。
  </x-card>
  <x-card data-title="DirEntry" data-icon="lucide:file">
    在目录中找到的条目，由 `ReadDir` 返回。
  </x-card>
  <x-card data-title="DirBuilder" data-icon="lucide:folder-plus">
    用于创建具有特定选项的目录的构建器。
  </x-card>
</x-cards>

## 函数

| 函数 | 描述 |
| --- | --- |
| `canonicalize(path)` | 将路径解析为其规范的绝对形式。 |
| `copy(from, to)` | 异步地将一个文件的内容复制到另一个文件。 |
| `create_dir(path)` | 在指定路径创建一个新的空目录。 |
| `create_dir_all(path)` | 递归地创建一个目录及其所有缺失的父组件。 |
| `hard_link(src, dst)` | 在文件系统上创建一个硬链接。 |
| `metadata(path)` | 读取路径的元数据，不跟随符号链接。 |
| `read(path)` | 将文件的全部内容读取到 `Vec<u8>` 中。 |
| `read_dir(path)` | 返回目录中条目的流。 |
| `read_link(path)` | 读取一个符号链接，返回它指向的路径。 |
| `read_to_string(path)` | 将文件的全部内容读取为字符串。 |
| `remove_dir(path)` | 移除一个空目录。 |
| `remove_dir_all(path)` | 移除此路径下的目录及其所有内容。 |
| `remove_file(path)` | 移除一个文件。 |
| `rename(from, to)` | 重命名文件或目录。 |
| `set_permissions(path, perm)` | 更改文件或目录的权限。 |
| `symlink(src, dst)` | 在文件系统上创建一个新的符号链接。（仅限 Unix） |
| `symlink_metadata(path)` | 读取文件的元数据，但不跟随符号链接。 |
| `try_exists(path)` | 检查路径是否存在于文件系统中。 |
| `write(path, contents)` | 将字节切片写入文件，覆盖现有文件。 |

---

## `File`

对文件系统上已打开文件的引用。它提供了读取、写入和寻址的方法。

### 创建和打开文件

**`File::open(path)`**：以只读模式打开文件。

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

**`File::create(path)`**：以只写模式打开文件。如果文件不存在则创建，如果存在则截断。

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::create("foo.txt").await?;
file.write_all(b"hello, world!").await?;
# Ok(())
# }
```

**`File::options()`**：返回一个 `OpenOptions` 构建器，用于更高级的配置。

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut f = File::options().append(true).open("example.log").await?;
f.write_all(b"new line\n").await?;
# Ok(())
# }
```

### 文件操作

**`metadata()`**：查询有关底层文件的元数据。

```rust
use tokio::fs::File;

# async fn dox() -> std::io::Result<()> {
let file = File::open("foo.txt").await?;
let metadata = file.metadata().await?;
println!("Is read-only: {}", metadata.permissions().readonly());
# Ok(())
# }
```

**`sync_all()`**：尝试将所有操作系统内部的元数据和数据同步到磁盘。

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

**`set_len(size)`**：将底层文件截断或扩展到指定大小。

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

提供一个用于配置如何打开文件的构建器。这允许对读取、写入、追加、创建和截断模式进行细粒度控制。

### 示例：以读写模式打开文件

此示例以读写模式打开文件。如果文件不存在，则会创建该文件。

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

### 配置方法

| 方法 | 描述 |
|---|---|
| `read(bool)` | 设置读取权限选项。 |
| `write(bool)` | 设置写入权限选项。 |
| `append(bool)` | 设置追加模式选项。 |
| `truncate(bool)` | 设置截断文件选项。需要写入权限。 |
| `create(bool)` | 设置在文件不存在时创建新文件的选项。需要写入或追加权限。 |
| `create_new(bool)` | 设置创建新文件的选项，如果文件已存在则失败。此为原子操作。 |

---

## `ReadDir` 和 `DirEntry`

这些结构体用于迭代目录中的条目。

### 示例：列出目录内容

`read_dir` 函数返回一个 `ReadDir` 流。你可以在循环中调用 `next_entry()` 来处理每个 `DirEntry`。

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

### `DirEntry` 方法

`DirEntry` 代表目录中的单个条目，并提供获取更多信息的方法。

| 方法 | 描述 |
|---|---|
| `path()` | 返回文件的完整路径。 |
| `file_name()` | 返回条目的基本文件名。 |
| `metadata()` | 返回此条目指向的文件的元数据。不跟随符号链接。 |
| `file_type()` | 返回此条目指向的文件的 `FileType`。不跟随符号链接。 |

以下是一个检查每个条目文件类型的示例：

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

这提供了一种在不阻塞异步运行时的情况下检查目录内容的便捷方法。