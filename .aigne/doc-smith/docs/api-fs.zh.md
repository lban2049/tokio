# 文件系统

该模块提供了用于处理文件系统的异步实用工具。这包括读写文件以及操作目录。

请注意，大多数操作系统不提供异步文件系统 API。因此，Tokio 会在后台使用普通的阻塞式文件操作。这是通过一个专用线程池在后台运行这些操作来实现的。有关更多详细信息，请参阅 [Tasks & Scheduling](./concepts-tasks.md) 文档。

`tokio::fs` 模块只应用于普通文件。尝试将其用于特殊文件（例如 Linux 上的命名管道）可能会导致意外行为。对于特殊文件，应改用 `tokio::net::unix::pipe` 或 `AsyncFd` 等专用类型。

## 快速入门：读写文件

使用该模块最简单的方法是使用那些能一次性操作整个文件的实用函数。

| 函数 | 描述 |
|---|---|
| `tokio::fs::read` | 异步地将文件的全部内容读入一个 `Vec<u8>`。 |
| `tokio::fs::read_to_string` | 异步地将文件的全部内容读入一个字符串。 |
| `tokio::fs::write` | 异步地将一个字节切片写入文件，并覆盖现有内容。 |

### 将文件读取为字符串

此示例读取 `my_file.txt` 的全部内容并打印行数。

```rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

### 写入文件

此示例将一个字符串写入 `my_file.txt`，如果文件已存在，则会覆盖该文件。

```rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

## File 类型

对于比读写整个文件更复杂的操作，[`File`](#file-struct) 结构体是主要工具。它实现了 `AsyncRead` 和 `AsyncWrite` trait，允许进行缓冲或基于块的 I/O。

**注意：** 向 Tokio `File` 写入时，使用 `flush` 非常重要。对 `write` 的调用会在写入完成前返回。`flush` 方法会等待写入操作完成。这与 `std::fs::File` 不同，是由于使用后台线程池进行 I/O 操作而导致的。

### 分块读取文件

此示例在不将整个文件加载到内存的情况下，计算文件中的行数。

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
        // 长度为零表示文件结束。
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

### 逐行写入文件

```rust,no_run
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

# async fn dox() -> std::io::Result<()> {
let mut file = File::create("my_file.txt").await?;

file.write_all(b"First line.\n").await?;
file.write_all(b"Second line.\n").await?;
file.write_all(b"Third line.\n").await?;

// 写入后记得调用 flush！
file.flush().await?;
# Ok(())
# }
```

## 性能调优

由于 Tokio 的文件 I/O 使用线程池，每次操作都可能产生一些开销。为了获得良好性能，建议将操作分批处理，以尽可能减少调用次数。

以下是一些策略：

1.  **在内存中缓冲：** 创建文件时，先将数据写入内存中的 `String` 或 `Vec<u8>`，然后通过单次调用 `tokio::fs::write` 将整个缓冲区写入文件。

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

2.  **使用 `BufWriter`：** 使用 `tokio::io::BufWriter` 将多次小写入缓冲为少量大写入。当缓冲区已满或您显式调用 `flush()` 时，才会真正写入文件。

    ```rust,no_run
    use tokio::fs::File;
    use tokio::io::{AsyncWriteExt, BufWriter};

    # async fn dox() -> std::io::Result<()> {
    let mut file = BufWriter::new(File::create("my_file.txt").await?);

    file.write_all(b"First line.\n").await?;
    file.write_all(b"Second line.\n").await?;
    file.write_all(b"Third line.\n").await?;

    // 真正的写入和阻塞调用发生在 flush 时。
    file.flush().await?;
    # Ok(())
    # }
    ```

3.  **手动 `spawn_blocking`：** 为了获得完全控制，可以在 `tokio::task::spawn_blocking` 调用中使用标准库的 `std::fs` 类型。

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

        // 与 Tokio 的文件不同，std::fs 文件
        // 不需要 flush。

        io::Result::Ok(())
    }).await.unwrap()?;
    # Ok(())
    # }
    ```

## 关键类型和函数

### 结构体

| 结构体 | 描述 |
|---|---|
| `File` | 对文件系统上已打开文件的引用。实现了 `AsyncRead`、`AsyncWrite` 和 `AsyncSeek`。 |
| `OpenOptions` | 用于配置如何打开文件的构建器。 |
| `DirBuilder` | 用于创建具有特定选项的目录的构建器。 |
| `ReadDir` | 目录中条目的流。 |
| `DirEntry` | 目录中的一个条目，由 `ReadDir` 返回。 |

### 函数

| 函数 | 描述 |
|---|---|
| `canonicalize` | 将路径解析为其规范的绝对形式。 |
| `copy` | 将一个文件的内容复制到另一个文件。 |
| `create_dir` | 在指定路径创建一个新的空目录。 |
| `create_dir_all` | 递归地创建一个目录及其所有缺失的父组件。 |
| `hard_link` | 在文件系统上创建一个硬链接。 |
| `metadata` | 查询有关路径的文件系统元数据。 |
| `read` | 将文件的全部内容读入一个字节向量。 |
| `read_dir` | 返回目录中条目的流。 |
| `read_link` | 读取一个符号链接，返回其指向的路径。 |
| `read_to_string` | 将文件的全部内容读入一个字符串。 |
| `remove_dir` | 删除一个空目录。 |
| `remove_dir_all` | 删除指定路径的目录及其所有内容。 |
| `remove_file` | 删除一个文件。 |
| `rename` | 重命名文件或目录。 |
| `set_permissions` | 更改文件或目录的权限。 |
| `symlink` | 在文件系统上创建一个新的符号链接（仅限 Unix）。 |
| `symlink_metadata` | 查询路径的元数据，但不跟随符号链接。 |
| `try_exists` | 检查路径是否存在于文件系统上。 |
| `write` | 将一个字节切片写入文件。 |
