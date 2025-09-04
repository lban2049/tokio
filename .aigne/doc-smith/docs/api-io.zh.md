# I/O

用于异步 I/O 功能的 trait、辅助函数和类型定义。该模块是 `std::io` 的异步版本。

该模块提供了标准库中 `Read`、`Write`、`BufRead` 和 `Seek` trait 的异步等效项。它还包含了用于处理这些 trait、处理标准 I/O 流和操作 I/O 对象的实用工具。

## 核心概念

Tokio I/O 系统的基本组件是 `AsyncRead` 和 `AsyncWrite` trait。这些 trait 为异步读写字节提供了最通用的接口。

与标准库中的对应项不同，当 I/O 未就绪时，这些 trait 上的方法将让出控制权给 Tokio 调度器，而不是阻塞当前线程。这使得应用程序在等待 I/O 操作完成时，可以运行其他任务。

大多数方便的 I/O 操作实用方法并不在核心 trait 本身上。相反，它们由扩展 trait `AsyncReadExt`、`AsyncWriteExt`、`AsyncBufReadExt` 和 `AsyncSeekExt` 提供，这些扩展 trait 会为所有实现相应基础 trait 的类型自动实现。

例如，异步读取文件与同步版本非常相似：

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

## 核心 I/O Trait

Tokio 的 I/O 系统围绕几个核心 trait 构建，这些 trait 定义了异步字节流的行为。

<x-cards>
  <x-card data-title="AsyncRead" data-icon="lucide:arrow-down-circle">
    提供了 `poll_read` 方法，用于从源异步读取字节。当数据不可用时，它会注册当前任务，以便在源再次变为可读时被唤醒。
  </x-card>
  <x-card data-title="AsyncWrite" data-icon="lucide:arrow-up-circle">
    提供了 `poll_write`、`poll_flush` 和 `poll_shutdown` 方法，用于向目标异步写入字节、刷新内部缓冲区以及优雅地关闭连接。
  </x-card>
  <x-card data-title="AsyncBufRead" data-icon="lucide:book-open">
    `std::io::BufRead` 的异步版本。它允许从内部缓冲区读取，这可以减少系统调用的次数并提高性能。
  </x-card>
  <x-card data-title="AsyncSeek" data-icon="lucide:move-horizontal">
    `std::io::Seek` 的异步版本。它提供了在字节流中更改当前位置的方法。
  </x-card>
</x-cards>

## 缓冲读取器和写入器

由于频繁的系统调用，直接使用基于字节的接口可能效率低下。为了解决这个问题，Tokio 提供了类似于 `std::io` 的缓冲 I/O 类型。

-   **`BufReader`**：包装一个 `AsyncRead` 以提供缓冲读取。它通过 `AsyncBufReadExt` trait 引入了诸如 `read_line` 之类的有用方法。
-   **`BufWriter`**：包装一个 `AsyncWrite` 以缓冲写入操作。在 `BufWriter` 被丢弃之前，对其调用 `flush()` 很重要，以确保所有缓冲数据都已写入底层写入器。

### 从文件读取行

```rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let mut reader = BufReader::new(f);
    let mut buffer = String::new();

    // read a line into buffer
    reader.read_line(&mut buffer).await?;

    println!("{}", buffer);
    Ok(())
}
```

### 缓冲写入

```rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write a byte to the buffer.
        writer.write_all(&[42u8]).await?;

        // Flush the buffer to ensure data is written to the file.
        writer.flush().await?;

    } // The buffer is discarded on drop unless flushed.

    Ok(())
}
```

## 标准 I/O

Tokio 为标准输入、输出和错误流提供了异步 API。这些函数返回实现了 `AsyncRead` 和 `AsyncWrite` 的句柄。

-   `stdin()`：返回当前进程标准输入的句柄。
-   `stdout()`：返回当前进程标准输出的句柄。
-   `stderr()`：返回当前进程标准错误的句柄。

**注意：** 这些 API 必须在 Tokio 运行时上下文中调用。

## 实用工具

该模块包含几个用于常见 I/O 任务的实用函数和结构体。

<x-cards>
  <x-card data-title="split()" data-icon="lucide:git-pull-request-arrow">
    将一个同时实现了 `AsyncRead` 和 `AsyncWrite` 的值拆分为独立的可读 (`ReadHalf`) 和可写 (`WriteHalf`) 句柄。
  </x-card>
  <x-card data-title="join()" data-icon="lucide:git-merge">
    将一个读取器和一个写入器合并为一个同时实现了 `AsyncRead` 和 `AsyncWrite` 的句柄。
  </x-card>
  <x-card data-title="copy()" data-icon="lucide:copy">
    异步地将读取器的全部内容复制到写入器中。
  </x-card>
  <x-card data-title="empty()", "sink()", "repeat()" data-icon="lucide:box">
    提供专用的 I/O 对象：`empty()` 是一个始终处于 EOF 的读取器，`sink()` 是一个无限接受并丢弃数据的写入器，而 `repeat()` 是一个无限产生特定字节的读取器。
  </x-card>
</x-cards>

## `std` 重导出

为方便起见，`std::io` 中的几个常用类型被重导出。这使你可以使用 `tokio::io` 而无需为这些类型单独导入 `std::io`。

-   `Error`
-   `ErrorKind`
-   `Result`
-   `SeekFrom`