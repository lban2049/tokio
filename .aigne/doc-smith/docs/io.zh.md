# 异步 I/O

此模块提供了 `std::io` 的异步版本。它包含用于处理非阻塞输入和输出操作的 trait、辅助函数和类型定义。

Tokio 的 I/O 功能核心围绕两个基本 trait：[`AsyncRead`](#asyncread) 和 [`AsyncWrite`](#asyncwrite)。它们是标准库中 `Read` 和 `Write` trait 的异步版本。当一个 I/O 操作无法立即完成时（例如，等待网络数据），它不会阻塞当前线程，而是将控制权交还给 Tokio 调度器。这使得其他任务可以运行，从而仅用少量操作系统线程即可实现大规模并发。

## `AsyncRead` 和 `AsyncWrite` Trait

虽然库的作者会为其类型实现 `AsyncRead` 和 `AsyncWrite`，但应用程序开发者通常会使用扩展 trait `AsyncReadExt` 和 `AsyncWriteExt` 提供的便捷方法。任何实现了 `AsyncRead` 或 `AsyncWrite` 的类型都可以自动使用这些 trait，它们提供了诸如 `read()`、`read_to_string()`、`write()` 和 `write_all()` 等熟悉的方法。

让我们看一个从文件读取的示例，这与使用 `std::fs::File` 的方式类似。

```rust File Read Example icon=logos:rust
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

这段代码异步地打开一个文件，并将最多 10 个字节读入缓冲区。如果文件未准备好立即读取，`.await` 调用将暂停该任务，并允许其他任务运行，直到数据可用。

### 核心 Trait 方法（供实现者参考）

对于构建 I/O 类型的人来说，理解这些 trait 的核心方法非常重要。

#### AsyncRead

`AsyncRead` trait 有一个必需方法：`poll_read`。

<x-field data-name="poll_read" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>, buf: &mut ReadBuf<'_'>) -> Poll<io::Result<()>>" data-required="true" data-desc="尝试将数据读入 `buf`。如果数据未就绪，它会返回 `Poll::Pending` 并注册当前任务的 waker，以便在数据可用时收到通知。成功时，它返回 `Poll::Ready(Ok(()))`。"></x-field>

#### AsyncWrite

`AsyncWrite` trait 需要三个方法，分别用于写入、刷新和关闭 I/O 资源。

<x-field data-name="poll_write" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>, buf: &[u8]) -> Poll<Result<usize, io::Error>>" data-required="true" data-desc="尝试从 `buf` 写入数据。返回 `Poll::Ready(Ok(n))`，其中 n 是写入的字节数；如果写入器未就绪，则返回 `Poll::Pending`。"></x-field>
<x-field data-name="poll_flush" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>) -> Poll<Result<(), io::Error>>" data-required="true" data-desc="尝试刷新所有缓冲数据。成功时返回 `Poll::Ready(Ok(()))`；如果无法立即完成刷新，则返回 `Poll::Pending`。"></x-field>
<x-field data-name="poll_shutdown" data-type="fn(self: Pin<&mut Self>, cx: &mut Context<'_'>) -> Poll<Result<(), io::Error>>" data-required="true" data-desc="启动写入器的平滑关闭。这用于需要在关闭连接前进行握手的协议。"></x-field>

## 缓冲读取器和写入器

对每次小的读写操作都直接进行系统调用可能效率低下。为了解决这个问题，`tokio::io` 提供了缓冲包装器 `BufReader` 和 `BufWriter`，它们与 `AsyncBufRead` trait 配合使用。这些包装器维护一个内存缓冲区，以减少系统调用的次数。

`BufReader` 添加了有用的读取方法，例如 `read_line`。

```rust BufReader Example icon=logos:rust
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

`BufWriter` 缓冲写入操作。在它被丢弃之前，调用 `flush()` 以确保所有缓冲数据都已写入底层的写入器至关重要。

```rust BufWriter Example icon=logos:rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write a byte to the buffer.
        writer.write_all(b"hello").await?;

        // Flush the buffer before it goes out of scope.
        writer.flush().await?;

    } // Unless flushed, the contents of the buffer is discarded on drop.

    Ok(())
}
```

## I/O 资源

Tokio 提供了一套丰富的异步 I/O 资源，这些资源构建于这些核心 trait 之上。

<x-cards data-columns="2">
  <x-card data-title="网络" data-icon="lucide:globe" data-href="/io/networking">
    用于异步 TCP、UDP 和 Unix 域套接字的基元。
  </x-card>
  <x-card data-title="文件系统" data-icon="lucide:folder-open" data-href="/io/fs">
    执行非阻塞的文件和目录操作。
  </x-card>
  <x-card data-title="子进程" data-icon="lucide:terminal-square" data-href="/io/process">
    异步地生成和管理子进程。
  </x-card>
  <x-card data-title="操作系统信号" data-icon="lucide:siren" data-href="/io/signals">
    处理 Unix 和 Windows 操作系统的信号。
  </x-card>
</x-cards>

此外，Tokio 还为标准输入 (`stdin`)、输出 (`stdout`) 和错误 (`stderr`) 提供了异步 API，它们实现了 `AsyncRead` 和 `AsyncWrite`。请注意，这些标准 I/O API 必须在 Tokio 运行时上下文中使用。

## 后续步骤

现在你已经对 Tokio 的异步 I/O 模型有了大致了解，可以开始探索具体的 I/O 资源了。一个很好的起点是网络，这是 Tokio 的主要用例之一。

<x-card data-title="下一步：网络" data-icon="lucide:arrow-right-circle" data-href="/io/networking" data-cta="阅读更多">
  学习如何使用 TCP 和 UDP 构建异步网络应用程序。
</x-card>