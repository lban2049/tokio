# 文件系统

用于异步文件和文件系统操作的 API 文档。

该模块提供了与文件系统交互的异步实用工具。它包含用于读取和写入文件、管理目录以及检查文件元数据的函数。

请注意，大多数操作系统不提供异步文件系统 API。因此，Tokio 使用一个阻塞线程池 (`spawn_blocking`) 在后台执行文件操作。这种设计允许你的异步任务在 I/O 操作于一组独立的线程上并发执行时保持非阻塞状态。

**注意：** `tokio::fs` 模块专为普通文件设计。将其用于特殊文件（如 Linux 上的命名管道）可能会导致意外行为，例如挂起。对于特殊文件，请考虑使用专用类型，如 `tokio::net::unix::pipe` 或 `AsyncFd`。

## 用法

使用该模块主要有两种方式：用于简单任务的高级实用函数和用于更精细控制的 `File` 结构体。

### 简单文件操作

对于一次性读取或写入整个文件等常见任务，使用实用函数是最简单的方法。

- [`tokio::fs::read`](#functions)：将整个文件读取到 `Vec<u8>` 中。
- [`tokio::fs::read_to_string`](#functions)：将整个文件读取到 `String` 中。
- [`tokio::fs::write`](#functions)：将字节切片写入文件，并覆盖现有内容。

**示例：将文件读取为字符串**

```rust,no_run 读取文件 icon=logos:rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

**示例：将字符串写入文件**

```rust,no_run 写入文件 icon=logos:rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

### 使用 `File` 进行精细控制

对于更复杂的场景，例如流式传输数据或定位到特定位置，[`File`](#file) 结构体提供了更多控制。它实现了 `AsyncRead`、`AsyncWrite` 和 `AsyncSeek` trait。

**重要提示：** 向 `File` 写入数据时，调用 `flush()` 以确保所有缓冲数据都已写入至关重要。与 `std::fs::File` 不同，Tokio 的 `File` 会缓冲写入并在后台执行它们。`flush()` 方法会等待这些后台操作完成。

**示例：在不加载整个文件的情况下计算行数**

```rust,no_run 分块读取文件 icon=logos:rust
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

## 性能调优

由于 Tokio 的文件操作使用 `spawn_blocking`，每次调用都有一些开销。为实现最佳性能，建议将操作分批处理，以尽可能减少 `spawn_blocking` 的调用次数。

以下是一些策略：

1.  **在内存中缓冲：** 对于创建文件，先在 `String` 或 `Vec<u8>` 中构建内容，然后使用 `tokio::fs::write` 一次性将其全部写入。

    ```rust,no_run 在内存中缓冲 icon=logos:rust
    # async fn dox() -> std::io::Result<()> {
    let mut contents = String::new();

    contents.push_str("First line.\n");
    contents.push_str("Second line.\n");
    contents.push_str("Third line.\n");

    tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
    # Ok(())
    # }
    ```

2.  **使用 `BufWriter`：** 将 `File` 包装在 `tokio::io::BufWriter` 中，以将多个小写入缓冲为对底层文件的少数几次大写入。

    ```rust,no_run 使用 BufWriter icon=logos:rust
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

3.  **手动 `spawn_blocking`：** 对于复杂的操作序列，你可以使用标准库的 `std::fs` 类型，在单个 `spawn_blocking` 调用中执行所有操作。

    ```rust,no_run 手动 spawn_blocking icon=logos:rust
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

## API 参考

### 结构体

<x-cards data-columns="2">
  <x-card data-title="File" data-icon="lucide:file">
    一个可异步访问的文件。为 I/O 操作实现了 `AsyncRead`、`AsyncWrite` 和 `AsyncSeek`。
  </x-card>
  <x-card data-title="OpenOptions" data-icon="lucide:settings-2">
    用于自定义文件打开方式的构建器，允许对读取、写入、创建和追加模式进行精细控制。
  </x-card>
  <x-card data-title="ReadDir" data-icon="lucide:folder-open">
    一个迭代目录中条目的流，会产生 `DirEntry` 实例。
  </x-card>
  <x-card data-title="DirEntry" data-icon="lucide:file-text">
    表示目录中的单个条目，提供对其路径、名称和元数据的访问。
  </x-card>
  <x-card data-title="DirBuilder" data-icon="lucide:folder-plus">
    用于创建具有特定选项（例如在 Unix 平台上设置模式）的目录的构建器。
  </x-card>
</x-cards>

### 函数

该模块为常见的文件系统操作提供了许多顶级函数。

| 函数 | 描述 |
|---|---|
| `canonicalize(path)` | 将路径解析为其规范的绝对形式。 |
| `copy(from, to)` | 异步地将一个文件的内容复制到另一个文件。 |
| `create_dir(path)` | 在指定路径创建一个新的空目录。 |
| `create_dir_all(path)` | 递归地创建一个目录及其所有缺失的父组件。 |
| `hard_link(src, dst)` | 在文件系统上创建一个硬链接。 |
| `metadata(path)` | 读取路径的元数据，会跟随符号链接。 |
| `read(path)` | 将文件的全部内容读取到一个字节向量中。 |
| `read_dir(path)` | 返回一个遍历目录中条目的流。 |
| `read_link(path)` | 读取一个符号链接，返回该链接指向的路径。 |
| `read_to_string(path)` | 将文件的全部内容读取到一个字符串中。 |
| `remove_dir(path)` | 移除一个空目录。 |
| `remove_dir_all(path)` | 递归地移除一个目录及其所有内容。 |
| `remove_file(path)` | 移除一个文件。 |
| `rename(from, to)` | 重命名或移动一个文件或目录。 |
| `set_permissions(path, perm)` | 更改文件或目录的权限。 |
| `symlink(src, dst)` | 在文件系统上创建一个新的符号链接（仅限 Unix）。 |
| `symlink_metadata(path)` | 读取路径的元数据，不跟随符号链接。 |
| `try_exists(path)` | 异步检查路径是否存在。 |
| `write(path, contents)` | 将字节切片写入文件，如果文件不存在则创建，如果存在则覆盖。 |
