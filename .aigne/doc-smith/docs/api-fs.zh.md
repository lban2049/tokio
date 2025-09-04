# 文件系统

用于文件和文件系统操作的异步工具。

该模块提供了用于执行文件系统异步 I/O 的实用方法。这包括读写文件以及操作目录。

需要注意的是，大多数操作系统不提供原生的异步文件系统 API。因此，Tokio 使用 `spawn_blocking` 在专用线程池上执行标准的阻塞式文件操作。虽然 Tokio 将来可能会采用像 `io_uring` 这样更新的异步 API，但目前的实现依赖于这种基于线程的方法。

**注意：** `tokio::fs` 模块专为普通文件设计。将其用于特殊文件（如 Linux 上的命名管道）可能会导致挂起等意外行为。对于这些情况，请使用 `tokio::net::unix::pipe` 或 `AsyncFd` 等专用类型。

## 快速入门：读取和写入整个文件

与文件交互最直接的方法是使用一次性处理整个文件的实用函数。

*   `tokio::fs::read`：将整个文件读取到 `Vec<u8>` 中。
*   `tokio::fs::read_to_string`：将整个文件读取到 `String` 中。
*   `tokio::fs::write`：将字节切片写入文件，覆盖现有内容。

### 将文件读取为字符串

```rust
# async fn dox() -> std::io::Result<()> {
let contents = tokio::fs::read_to_string("my_file.txt").await?;

println!("File has {} lines.", contents.lines().count());
# Ok(())
# }
```

### 写入文件

```rust
# async fn dox() -> std::io::Result<()> {
let contents = "First line.\nSecond line.\nThird line.\n";

tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
# Ok(())
# }
```

## 使用 `File` 的高级用法

对于更复杂的场景，例如流式传输数据或避免将整个文件加载到内存中，请使用 `File` 结构体。它实现了 `AsyncRead` 和 `AsyncWrite` 特征，用于进行细粒度的 I/O 操作。

**重要提示：** 写入 Tokio `File` 时，必须调用 `flush()` 以确保写入操作完成。因为 `File` 内部使用 `spawn_blocking`，`write` 调用可能在后台线程实际写入数据之前返回。`flush()` 会等待这个后台操作完成。

### 分块读取文件

此示例在不将整个文件加载到内存的情况下计算行数。

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

// 写入后记得调用 `flush`！
file.flush().await?;
# Ok(())
# }
```

## 性能调优

由于 Tokio 的文件 I/O 依赖于 `spawn_blocking`，每次操作都可能引入开销。为获得良好性能，应将操作分批处理，以尽可能减少 `spawn_blocking` 的调用次数。

以下是一些有效的策略：

1.  **在内存中缓冲，然后一次性写入：** 将数据收集到 `String` 或 `Vec<u8>` 中，然后通过单次调用 `tokio::fs::write` 写入整个缓冲区。

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

2.  **使用 `BufWriter`：** `BufWriter` 会缓冲小的写入操作，并将它们作为一次较大的写入操作进行刷新，从而减少底层系统调用的次数。

    ```rust,no_run
    use tokio::fs::File;
    use tokio::io::{AsyncWriteExt, BufWriter};

    # async fn dox() -> std::io::Result<()> {
    let mut file = BufWriter::new(File::create("my_file.txt").await?);

    file.write_all(b"First line.\n").await?;
    file.write_all(b"Second line.\n").await?;
    file.write_all(b"Third line.\n").await?;

    // 实际的写入和 spawn_blocking 调用发生在刷新时。
    file.flush().await?;
    # Ok(())
    # }
    ```

3.  **手动使用 `spawn_blocking`：** 为实现最大程度的控制，可以在 `spawn_blocking` 调用中自行执行标准库的文件 I/O 操作。

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

        // 与 Tokio 的文件不同，std::fs 文件不需要刷新。

        io::Result::Ok(())
    }).await.unwrap()?;
    # Ok(())
    # }
    ```

你还可以使用 `File::set_max_buf_size` 来调整 Tokio 在单次 `spawn_blocking` 调用中处理的数据量。

## 关键结构体

<x-cards data-columns="2">
  <x-card data-title="File" data-icon="lucide:file-text">
    文件系统上已打开文件的异步句柄。
  </x-card>
  <x-card data-title="OpenOptions" data-icon="lucide:settings-2">
    用于配置如何使用特定选项打开文件的构建器。
  </x-card>
  <x-card data-title="ReadDir" data-icon="lucide:folder-open">
    一个流，用于产生目录中的条目。
  </x-card>
  <x-card data-title="DirEntry" data-icon="lucide:file">
    从目录中读取的单个条目，由 `ReadDir` 返回。
  </x-card>
  <x-card data-title="DirBuilder" data-icon="lucide:folder-plus">
    用于创建具有特定选项（如在 Unix 上设置模式）的目录的构建器。
  </x-card>
</x-cards>

## 函数

### 文件操作

| Function | Description |
|---|---|
| `copy` | 异步地将一个文件的内容复制到另一个文件。 |
| `read` | 将文件的全部内容读取到字节向量中。 |
| `read_to_string` | 将文件的全部内容读取到字符串中。 |
| `remove_file` | 删除一个文件。 |
| `write` | 将字节切片作为文件的全部内容写入。 |

### 目录操作

| Function | Description |
|---|---|
| `create_dir` | 在指定路径创建一个新的空目录。 |
| `create_dir_all` | 递归地创建一个目录及其所有缺失的父组件。 |
| `read_dir` | 返回一个遍历目录中条目的流。 |
| `remove_dir` | 删除一个空目录。 |
| `remove_dir_all` | 删除此路径下的目录及其所有内容。 |

### 文件系统操作

| Function | Description |
|---|---|
| `rename` | 重命名文件或目录。 |
| `hard_link` | 在文件系统上创建一个新的硬链接。 |
| `symlink` | 在文件系统上创建一个新的符号链接。（Unix 特定） |
| `symlink_dir` | 在文件系统上创建一个新的目录符号链接。（Windows 特定） |
| `symlink_file` | 在文件系统上创建一个新的文件符号链接。（Windows 特定） |

### 元数据和路径

| Function | Description |
|---|---|
| `canonicalize` | 返回路径的规范化、绝对形式，所有中间组件都已标准化。 |
| `metadata` | 查询路径的文件系统元数据，会跟随符号链接。 |
| `symlink_metadata` | 查询文件的元数据，不会跟随符号链接。 |
| `read_link` | 读取符号链接，返回链接指向的路径。 |
| `set_permissions` | 更改文件或目录的权限。 |
| `try_exists` | 检查路径是否存在于文件系统上。 |
