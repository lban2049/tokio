# 文件系统

学习如何使用 `tokio::fs` 执行异步文件操作，包括读取、写入以及操作文件和目录。该模块为文件系统 I/O 提供了标准库函数的异步版本。

### 工作原理

需要注意的是，大多数操作系统并未提供原生的异步文件系统 API。因此，`tokio::fs` 使用由 Tokio 运行时管理的线程池来执行标准的阻塞式文件操作，而不会阻塞主异步任务。这是通过 [`spawn_blocking`](./tasks-scheduling-spawning.md) 机制实现的。

虽然这种方法提供了异步 API，但它也带来了性能影响。每个操作都可能涉及将任务发送到阻塞线程，这会产生一些开销。为了获得最佳性能，建议将操作分批处理，尽可能减少 `spawn_blocking` 的调用次数。

> **注意：** `tokio::fs` 模块专为普通文件设计。将其用于特殊文件（如 Linux 上的命名管道）可能会导致意外行为，例如挂起。对于特殊文件，请考虑使用专用类型，如 [`tokio::net::unix::pipe`](./io-networking.md) 或 `AsyncFd`。

## 快速入门：读取和写入整个文件

处理文件最直接的方法是使用一次性操作整个文件的实用函数。

### 将文件读取为字符串

要将文件的全部内容读取到字符串中，请使用 `tokio::fs::read_to_string`。

```rust Read a file icon=logos:rust
async fn read_file() -> std::io::Result<()> {
    let contents = tokio::fs::read_to_string("my_file.txt").await?;
    println!("File has {} lines.", contents.lines().count());
    Ok(())
}
```

### 将切片写入文件

要将字节切片的全部内容写入文件，请使用 `tokio::fs::write`。如果文件已存在，其内容将被覆盖。

```rust Write to a file icon=logos:rust
async fn write_file() -> std::io::Result<()> {
    let contents = "First line.\nSecond line.\nThird line.\n";
    tokio::fs::write("my_file.txt", contents.as_bytes()).await?;
    Ok(())
}
```

## `File` 的高级用法

如需更精细的控制，例如分块读取文件或流式写入，请使用 `tokio::fs::File` 结构体。它实现了 `AsyncRead` 和 `AsyncWrite` trait，允许你将其与 `tokio::io` 提供的丰富组合器结合使用。

### 创建 `File` 并向其写入

你可以使用 `File::create` 或 `File::open` 创建新文件或打开现有文件。

**重要提示：** 向 `File` 写入数据后，几乎总是应该调用 `flush()`。Tokio 的 `File` 会在内部缓冲写入操作。`write_all` 调用在数据传递给操作系统之前就会返回。`flush()` 会等待底层的写入操作完成。

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

### 从 `File` 读取

要从文件读取数据，请使用 `File::open` 打开文件，并使用 `AsyncReadExt` 中的方法，如 `read`。

此示例计算文件中的行数，而无需将整个文件加载到内存中，这对于非常大的文件非常有用。

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

## 性能调优策略

由于 `tokio::fs` 使用阻塞线程池，因此最大限度地减少独立 I/O 操作的数量是获得良好性能的关键。以下是一些实现该目标的策略。

<x-cards data-columns="1">
  <x-card data-title="内存中缓冲" data-icon="lucide:memory-stick" data-horizontal="true">
    对于写入操作，首先在内存中的 `String` 或 `Vec<u8>` 中构建完整的文件内容，然后通过单次调用 `tokio::fs::write` 将其全部写入。这确保了整个操作只进行一次 `spawn_blocking` 调用。
  </x-card>
  <x-card data-title="使用 BufReader 和 BufWriter" data-icon="lucide:file-cog" data-horizontal="true">
    将你的 `File` 包装在 `tokio::io::BufReader` 或 `tokio::io::BufWriter` 中。这些包装器会将许多小的读/写调用聚合成对底层文件的少数几次较大操作，从而减少上下文切换到阻塞线程的开销。
  </x-card>
  <x-card data-title="手动 spawn_blocking" data-icon="lucide:cpu" data-horizontal="true">
    对于复杂的文件操作序列，你可以使用标准的 `std::fs` 库，在单个 `tokio::task::spawn_blocking` 调用中执行所有操作。这使你能够最大限度地控制批处理。
  </x-card>
</x-cards>

### 示例：使用 `BufWriter`

`BufWriter` 会将写入操作缓冲在内存中，仅在内部缓冲区已满或显式调用 `flush()` 时才执行实际的 `spawn_blocking` 调用。

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

## 使用 `OpenOptions` 打开文件

为了更精细地控制文件的打开方式，你可以使用 `OpenOptions` 构建器。它允许你配置读取、写入、追加、创建和截断模式。

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

## 其他文件系统操作

除了读写文件，`tokio::fs` 模块还提供了一套全面的函数，用于与文件系统交互，包括：

<x-cards data-columns="2">
  <x-card data-title="create_dir / create_dir_all" data-icon="lucide:folder-plus">
    创建一个新目录，或创建一个目录及其所有父目录。
  </x-card>
  <x-card data-title="remove_dir / remove_file" data-icon="lucide:trash-2">
    删除一个空目录或一个文件。
  </x-card>
  <x-card data-title="read_dir" data-icon="lucide:folder-search">
    读取目录的内容。
  </x-card>
  <x-card data-title="rename / copy" data-icon="lucide:copy-check">
    重命名或复制文件。
  </x-card>
  <x-card data-title="symlink / read_link" data-icon="lucide:link-2">
    创建或读取符号链接。
  </x-card>
  <x-card data-title="metadata / set_permissions" data-icon="lucide:file-lock">
    读取或修改文件元数据和权限。
  </x-card>
</x-cards>

---

现在你已经了解了如何执行异步文件 I/O，可能对管理其他系统资源感兴趣。请继续阅读 [子进程](./io-process.md) 部分，学习如何异步地生成和管理子进程。
