# I/O

用于异步 I/O 功能的 Trait、辅助函数和类型定义。此模块是 `std::io` 的异步版本。

Tokio 的 I/O 模型围绕几个核心 Trait 构建：`AsyncRead` 和 `AsyncWrite`。这些是标准库中 `Read` 和 `Write` Trait 的异步版本。关键区别在于它们的方法不会阻塞当前线程。它们不会阻塞，而是返回 `Poll::Pending`，并安排在 I/O 操作可以继续时唤醒当前任务。

## 核心 Trait

这些 Trait 构成了 Tokio 中所有异步 I/O 操作的基础。

<x-cards>
  <x-card data-title="AsyncRead" data-icon="lucide:book-open">
    提供核心 `poll_read` 方法，用于从源异步读取字节。大多数用户将通过 `AsyncReadExt` 提供的便捷方法与此 Trait 交互。
  </x-card>
  <x-card data-title="AsyncWrite" data-icon="lucide:edit-3">
    提供向目标异步写入字节的核心方法，包括 `poll_write`、`poll_flush` 和 `poll_shutdown`。`AsyncWriteExt` Trait 提供了更易用的写入方法。
  </x-card>
  <x-card data-title="AsyncBufRead" data-icon="lucide:file-text">
    `std::io::BufRead` 的异步版本，用于从缓冲源读取字节。它提供 `poll_fill_buf` 等方法，并由 `AsyncBufReadExt` Trait 补充，以提供 `read_line` 等辅助函数。
  </x-card>
  <x-card data-title="AsyncSeek" data-icon="lucide:move-horizontal">
    `std::io::Seek` 的异步版本，用于在字节流中移动光标。它使用一个两步过程：`start_seek` 和 `poll_complete`。
  </x-card>
</x-cards>

### 读写数据

虽然核心 Trait 提供了低级别的轮询方法，但您通常会使用 `AsyncReadExt` 和 `AsyncWriteExt` 扩展 Trait 上的辅助方法。对于任何实现了 `AsyncRead` 或 `AsyncWrite` 的类型，这些 Trait 都会自动可用。

例如，您可以使用 `AsyncReadExt` 中的 `read` 方法从文件中读取数据：

```rust Reading from a file icon=logos:rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // 最多读取 10 个字节
    let n = f.read(&mut buffer).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

## 缓冲读取器和写入器

为了提高效率和便利性，Tokio 提供了类似于 `std::io` 的缓冲 I/O 类型。这些包装器可以减少系统调用的次数，并为常见任务提供了有用的方法。

`BufReader` 为任何异步读取器添加缓冲功能，并与 `AsyncBufReadExt` 结合使用，可以启用 `read_line` 等方法。

```rust Reading a line with BufReader icon=logos:rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let reader = BufReader::new(f);
    let mut lines = reader.lines();
    
    while let Some(line) = lines.next_line().await? {
        println!("{}", line);
    }

    Ok(())
}
```

`BufWriter` 缓冲对任何异步写入器的写入操作。调用 `flush` 至关重要，以确保所有缓冲数据都已写入底层的写入器。

```rust Using BufWriter icon=logos:rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // 向缓冲区写入一些字节。
        writer.write_all(b"some bytes").await?;

        // 刷新缓冲区以确保数据被写入。
        writer.flush().await?;

    } // 缓冲区在 drop 时会再次刷新，但最好显式调用。

    Ok(())
}
```

## 工具

此模块还包括几个用于处理 I/O 流的工具。

<x-cards data-columns="3">
  <x-card data-title="split" data-icon="lucide:split">
    将一个同时实现了 `AsyncRead` 和 `AsyncWrite` 的值拆分为一个独立的读取器部分（`ReadHalf`）和一个写入器部分（`WriteHalf`）。
  </x-card>
  <x-card data-title="join" data-icon="lucide:merge">
    `split` 的逆操作。将一个 `AsyncRead` 和一个 `AsyncWrite` 值合并为一个同时实现这两个 Trait 的句柄。
  </x-card>
  <x-card data-title="Standard I/O" data-icon="lucide:terminal">
    `stdin`、`stdout` 和 `stderr` 函数提供对进程标准 I/O 流的异步句柄。这些函数必须在 Tokio 运行时内调用。
  </x-card>
</x-cards>

## `std` 重导出

为方便起见，以下常见类型从 `std::io` 重导出：

*   `Error`
*   `ErrorKind`
*   `Result`
*   `SeekFrom`

---

在介绍了基本的 I/O Trait 之后，可以探索不同用例的具体实现：

<x-cards>
  <x-card data-title="Networking" data-icon="lucide:network" data-href="/api/net">
    用于异步 TCP、UDP 和 Unix 套接字。
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder" data-href="/api/fs">
    用于异步文件和文件系统操作。
  </x-card>
  <x-card data-title="Processes" data-icon="lucide:cpu" data-href="/api/process">
    用于与子进程 I/O 流交互。
  </x-card>
</x-cards>