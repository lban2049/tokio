# I/O

该模块为异步 I/O 功能提供 trait、辅助函数和类型定义。它可作为 `std::io` 的异步对应部分。

Tokio I/O 的基石是一对 trait：`AsyncRead` 和 `AsyncWrite`，它们是标准库中 `Read` 和 `Write` trait 的异步版本。

## `AsyncRead` 和 `AsyncWrite`

与标准库的 `Read` 和 `Write` trait 类似，`AsyncRead` 和 `AsyncWrite` 提供了用于读取和写入字节流的通用接口。主要区别在于它们的异步性质。当 I/O 操作无法立即完成时，它不会阻塞线程，而是交由 Tokio 调度器处理。这使得在 I/O 操作挂起时，其他任务可以继续运行。

这些 trait 的实用方法通过扩展 trait `AsyncReadExt` 和 `AsyncWriteExt` 提供，它们会自动为任何实现 `AsyncRead` 和 `AsyncWrite` 的类型实现。

例如，从 `tokio::fs::File` 读取数据与从 `std::fs::File` 读取非常相似：

```rust,no_run
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

### 缓冲 I/O

为了提高效率并减少系统调用，Tokio 提供了缓冲 I/O 类型，类似于 `std::io` 中的类型。这些包括 `AsyncBufRead` trait 以及 `BufReader` 和 `BufWriter` 结构体。这些包装器使用内部缓冲区来批量处理 I/O 操作。

`BufReader` 为任何异步读取器增强了 `read_line` 等方法：

```rust,no_run
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

`BufWriter` 会缓冲写入操作。必须调用 `flush` 以确保所有缓冲数据都已写入底层的写入器。

```rust,no_run
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write a byte to the buffer.
        writer.write(&[42u8]).await?;

        // Flush the buffer before it goes out of scope.
        writer.flush().await?;

    } // Unless flushed, the contents of the buffer is discarded on drop.

    Ok(())
}
```

## 核心 I/O Trait

Tokio 的 I/O 功能是围绕一组定义了读取、写入和定位的异步行为的核心 trait 构建的。

<x-cards data-columns="2">
  <x-card data-title="AsyncRead" data-icon="lucide:arrow-down-circle">
    从源异步读取字节。类似于 `std::io::Read`。
  </x-card>
  <x-card data-title="AsyncWrite" data-icon="lucide:arrow-up-circle">
    将字节异步写入目标。类似于 `std::io::Write`。
  </x-card>
  <x-card data-title="AsyncBufRead" data-icon="lucide:layers">
    用于缓冲异步读取的 trait，提供 `read_line` 等方法。
  </x-card>
  <x-card data-title="AsyncSeek" data-icon="lucide:move-horizontal">
    用于在异步 I/O 流中定位到不同位置的 trait。
  </x-card>
</x-cards>

### trait AsyncRead

该 trait 允许从源异步读取字节。其核心方法 `poll_read` 尝试将字节读入缓冲区。如果数据不能立即可用，它会返回 `Poll::Pending`，并安排在源再次可读时通知当前任务。

**核心方法**

| Method | Description |
|---|---|
| `poll_read(self: Pin<&mut Self>, cx: &mut Context<'_>, buf: &mut ReadBuf<'_>) -> Poll<io::Result<()>>` | 尝试将数据读入 `buf`。成功时返回 `Poll::Ready(Ok(()))`，无可用数据时返回 `Poll::Pending`，出错时返回 `Poll::Ready(Err(e))`。 |

### trait AsyncWrite

该 trait 允许将字节异步写入目标。如果目标未准备好接收更多数据，其方法将返回 `Poll::Pending`，并调度当前任务在写入器变为可写时收到通知。

**核心方法**

| Method | Description |
|---|---|
| `poll_write(self: Pin<&mut Self>, cx: &mut Context<'_>, buf: &[u8]) -> Poll<Result<usize, io::Error>>` | 尝试将缓冲区写入此写入器，返回写入的字节数。 |
| `poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), io::Error>>` | 尝试刷新对象，确保所有缓冲数据到达其目的地。 |
| `poll_shutdown(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), io::Error>>` | 启动或尝试关闭此写入器。 |

### trait AsyncBufRead

`AsyncRead` 的扩展，增加了缓冲读取的方法。这对于更复杂的解析（如逐行读取）很有用。

**核心方法**

| Method | Description |
|---|---|
| `poll_fill_buf(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<&[u8]>>` | 尝试用更多数据填充内部缓冲区，返回可用字节的切片。 |
| `consume(self: Pin<&mut Self>, amt: usize)` | 通知缓冲区已从缓冲区中消费了 `amt` 个字节，这些字节不应再次返回。 |

### trait AsyncSeek

该 trait 提供异步定位功能，类似于 `std::io::Seek`。

**核心方法**

| Method | Description |
|---|---|
| `start_seek(self: Pin<&mut Self>, position: SeekFrom) -> io::Result<()>` | 提交一个定位到指定偏移量的操作。 |
| `poll_complete(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<u64>>` | 等待挂起的定位操作完成，返回新位置。 |

## 实用工具

### 拆分和合并 I/O

Tokio 提供了将单个 I/O 资源拆分为独立的读取和写入句柄，或将独立的读取和写入句柄合并为单个资源的实用工具。

<x-cards>
  <x-card data-title="fn split()" data-icon="lucide:git-pull-request-arrow">
    将一个同时实现 `AsyncRead` 和 `AsyncWrite` 的值拆分为一个 `ReadHalf` 和一个 `WriteHalf`。这对于将这两个部分传递给不同的任务很有用。
  </x-card>
  <x-card data-title="fn join()" data-icon="lucide:git-merge">
    将一个独立的 `AsyncRead` 值和一个 `AsyncWrite` 值合并成一个同时实现这两个 trait 的 `Join` 句柄。
  </x-card>
</x-cards>

### 标准 I/O

Tokio 为标准输入、输出和错误流提供了异步 API。

- `stdin()`：返回标准输入流的句柄。
- `stdout()`：返回标准输出流的句柄。
- `stderr()`：返回标准错误流的句柄。

> **注意：** 这些函数必须在 Tokio 运行时的上下文中调用。

### 与 Stream 和 Sink 的互操作性

对于更高级的用例，将 `AsyncRead` 或 `AsyncWrite` 适配为 `Stream` 或 `Sink` 会很方便。[`tokio-util`](https://docs.rs/tokio-util) crate 为此提供了适配器：

- **`ReaderStream`**：将 `AsyncRead` 转换为字节块的 `Stream`。
- **`StreamReader`**：将字节块的 `Stream` 转换为 `AsyncRead`。
- **`Decoder` 和 `Encoder`**：用于构建帧协议的 trait，将字节流转换为结构化消息流，反之亦然。

### 从 `std::io` 重新导出

为方便起见，本模块从 `std::io` 中重新导出了以下常用类型：

- `Error`
- `ErrorKind`
- `Result`
- `SeekFrom`

---

在对 Tokio 的 I/O 原语有了扎实的理解之后，你现在可以探索用于特定任务的相关模块：

<x-cards>
  <x-card data-title="网络" data-icon="lucide:network" data-href="/api/net">
    探索用于网络通信的异步 TCP、UDP 和 Unix 套接字。
  </x-card>
  <x-card data-title="文件系统" data-icon="lucide:folder-git-2" data-href="/api/fs">
    了解异步文件和文件系统操作。
  </x-card>
</x-cards>