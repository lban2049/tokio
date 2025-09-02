# 异步 I/O

Tokio 提供了一套非阻塞 I/O 原语，用于构建高性能网络应用程序。这包括用于网络、文件系统操作和进程间通信的工具。Tokio 的 I/O 模型核心基于几个关键的 trait，它们是标准库中对应部分的异步版本。

## `AsyncRead` 和 `AsyncWrite` Trait

Tokio I/O 的基础是两个基本 trait：[`AsyncRead`](./api-io.md) 和 [`AsyncWrite`](./api-io.md)。它们是 `std::io::Read` 和 `std::io::Write` 的异步版本。主要区别在于它们的方法不会阻塞当前线程。当一个操作无法立即完成时（例如，等待来自网络套接字的数据），任务将让出给 Tokio 调度器。调度器将运行另一个任务，并在 I/O 资源再次就绪时恢复原始任务。

这种非阻塞方法允许少量线程处理大量并发 I/O 操作。

```d2
shape: sequence_diagram
direction: down

"用户任务"
"Tokio 运行时"
"操作系统 (OS)"

"用户任务" -> "Tokio 运行时": "socket.read().await"
"Tokio 运行时" -> "操作系统 (OS)": "检查套接字是否可读"
"操作系统 (OS)" -> "Tokio 运行时": "未就绪"
"Tokio 运行时" -> "用户任务": "暂停任务"
"用户任务": "已挂起 (让出 CPU)"

"操作系统 (OS)" -> "Tokio 运行时": "事件：套接字现在可读" {
  style.animated: true
}
"Tokio 运行时" -> "用户任务": "唤醒任务"
"用户任务" -> "Tokio 运行时": "重试 socket.read()"
"Tokio 运行时" -> "操作系统 (OS)": "从套接字读取数据"
"操作系统 (OS)" -> "Tokio 运行时": "返回数据"
"Tokio 运行时" -> "用户任务": ".await 完成"
```

大多数情况下，您不会直接使用核心 trait 方法。而是会使用 [`AsyncReadExt`](./api-io.md) 和 [`AsyncWriteExt`](./api-io.md) trait 提供的便捷实用方法，这些方法对于任何实现 `AsyncRead` 或 `AsyncWrite` 的类型都自动可用。

这是一个从文件中读取最多 10 个字节的示例：

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // 读取最多 10 个字节
    let n = f.read(&mut buffer).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

## 缓冲 I/O

对每个小的读写操作都直接进行系统调用可能效率低下。为了缓解这个问题，Tokio 提供了缓冲读取器和写入器，类似于标准库。`BufReader` 和 `BufWriter` 结构体包装了任何异步读取器或写入器，并使用内存中的缓冲区来减少系统调用的数量。

`BufReader` 添加了像 `read_line` 这样的便捷方法：

```rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let mut reader = BufReader::new(f);
    let mut buffer = String::new();

    // 读取一行到 buffer 中
    reader.read_line(&mut buffer).await?;

    println!("{}", buffer);
    Ok(())
}
```

`BufWriter` 缓冲写入操作。调用 `flush()` 至关重要，以确保缓冲区中剩余的任何数据都被写入到底层的写入器中。

```rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    let mut writer = BufWriter::new(f);

    // 将字节写入缓冲区。
    writer.write_all(b"some bytes").await?;

    // 刷新缓冲区以确保数据写入文件。
    writer.flush().await?;

    Ok(())
}
```

## Tokio 的 I/O 工具包

Tokio 为各种 I/O 操作提供了一套全面的 API。

<x-cards data-columns="3">
  <x-card data-title="网络" data-icon="lucide:globe" data-href="/api/net">
    用于构建网络客户端和服务器的非阻塞 TCP、UDP 和 Unix 套接字。
  </x-card>
  <x-card data-title="文件系统" data-icon="lucide:folder" data-href="/api/fs">
    用于文件和文件系统操作的异步 API，如读取、写入和创建目录。
  </x-card>
  <x-card data-title="进程" data-icon="lucide:terminal-square" data-href="/api/process">
    异步生成和管理子进程，并与其 stdin、stdout 和 stderr 流进行交互。
  </x-card>
</x-cards>

### 关于文件系统 I/O 的说明

大多数操作系统没有为文件系统访问提供真正的异步 API。为了解决这个问题，Tokio 的 `fs` 模块通过 `spawn_blocking` 为阻塞文件操作使用一个专门的线程池。虽然这为您的应用程序提供了一个异步 API，但请注意它会带来性能影响。对于高吞吐量的文件 I/O，请考虑批量操作或手动使用 `spawn_blocking` 一次性执行多个同步操作等策略。

### 标准 I/O 和 OS 信号

Tokio 还提供了标准输入、输出和错误流 ([`stdin`](./api-io.md)、[`stdout`](./api-io.md)、[`stderr`](./api-io.md)) 的异步版本。此外，[`tokio::signal`](./api-signal.md) 模块允许您异步处理像 `SIGINT` (Ctrl-C) 这样的操作系统信号。

---

现在您已经了解了 Tokio 的 I/O 模型，下一步是学习如何在执行 I/O 的任务之间管理共享状态。为此，您应该探索 Tokio 的[同步原语](./concepts-synchronization.md)。
