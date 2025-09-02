# I/O

用于异步 I/O 功能的 Trait、辅助函数和类型定义。该模块是 `std::io` 的异步版本。

该模块为在 Tokio 中处理异步输入和输出提供了基础构建块。其主要组件是 `AsyncRead` 和 `AsyncWrite` 这两个 Trait，它们是标准库中 `Read` 和 `Write` Trait 的非阻塞版本。

```d2
direction: down

"核心 I/O Trait" : {
  shape: package
  "AsyncRead": "从源异步读取字节。"
  "AsyncWrite": "向目标异步写入字节。"
  "AsyncBufRead": "用于带缓冲的异步读取的 Trait。"
  "AsyncSeek": "用于在流中进行异步寻址的 Trait。"
}

"I/O 工具与结构体" : {
   shape: package
   "BufReader": "为任何 AsyncRead 添加缓冲。"
   "BufWriter": "为任何 AsyncWrite 添加缓冲。"
   "split()": "将一个流拆分为独立的可读和可写两部分。"
   "join()": "将一个读取器和一个写入器合并成单个流。"
   "stdin(), stdout(), stderr()": "标准 I/O 流。"
}

"核心 I/O Trait" -> "I/O 工具与结构体": "由其使用和实现"
```

## 核心 Trait

Tokio 的 I/O 功能是围绕一组核心 Trait 构建的，这些 Trait 定义了异步字节流的行为。

### `AsyncRead`

`AsyncRead` Trait 用于可以异步读取字节的源。它是 `std::io::Read` 的异步等价物。

与其同步版本不同，当 I/O 未就绪时，`AsyncRead` 上的方法会把控制权交给 Tokio 调度器，而不是阻塞线程。这允许其他任务在等待 I/O 操作完成时运行。

该 Trait 的核心方法是 `poll_read`：

```rust
fn poll_read(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>,
    buf: &mut ReadBuf<'_>,
) -> Poll<io::Result<()>>;
```

- **`Poll::Ready(Ok(()))`**：表示数据已成功读入缓冲区。读取的字节数可以通过检查 `buf.filled().len()` 的变化来确定。如果长度没有变化，则意味着已到达文件末尾 (EOF)。
- **`Poll::Pending`**：表示当前没有可用数据。当前任务被调度为在 I/O 资源再次变为可读时被唤醒。
- **`Poll::Ready(Err(e))`**：发生 I/O 错误。

最终用户通常会使用 [`AsyncReadExt`](#asyncreadext) 提供的方法（如 `.read()`），而不是直接调用 `poll_read`。

### `AsyncWrite`

`AsyncWrite` Trait 用于可以异步写入字节的目标。它是 `std::io::Write` 的异步等价物。

它提供了三个核心方法，如果操作无法立即完成，每个方法都会返回一个 `Poll`，用于调度当前任务以便后续唤醒。

| 方法 | 描述 |
|---|---|
| `poll_write` | 尝试将缓冲区中的字节写入对象。返回写入的字节数。 |
| `poll_flush` | 尝试刷新所有缓冲数据，确保其到达目的地。 |
| `poll_shutdown` | 启动写入器的平滑关闭。这可能涉及刷新数据和执行关闭握手。 |

### `AsyncBufRead`

该 Trait 类似于 `std::io::BufRead`，由具有内部缓冲区的异步读取器实现。它提供了更高效的缓冲读取方法，例如读取直到遇到分隔符。

其核心方法是：

| 方法 | 描述 |
|---|---|
| `poll_fill_buf` | 尝试从底层读取器获取更多数据以填充内部缓冲区。返回可用数据的切片。 |
| `consume` | 通知缓冲区，已有特定数量的字节被消耗，不应再次返回。 |

### `AsyncSeek`

该 Trait 提供了类似于 `std::io::Seek` 的异步寻址功能。它允许更改流中的位置。

| 方法 | 描述 |
|---|---|
| `start_seek` | 提交一个到指定位置的寻址操作。该操作不阻塞。 |
| `poll_complete` | 等待一个待处理的寻址操作完成，返回从流开始处的新位置。 |

## 扩展 Trait

`AsyncRead` 和 `AsyncWrite` 的实用方法通过扩展 Trait 提供，这些 Trait 对任何实现核心 Trait 的类型都自动可用。

### `AsyncReadExt`

提供方便的、基于 Future 的方法，如 `read`、`read_exact` 和 `read_to_end`。这些是您从 I/O 资源读取时将使用的主要方法。

使用 `AsyncReadExt::read` 的示例：

```no_run
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

### `AsyncWriteExt`

提供方便的、基于 Future 的方法，如 `write`、`write_all` 和 `flush`。

## 带缓冲的读取器和写入器

为了减少系统调用次数并提高性能，Tokio 提供了与标准库类似的带缓冲的 I/O 类型。

- **`BufReader`**：包装任何 `AsyncRead` 以提供缓冲。它实现了 `AsyncBufRead`，并提供了如 `read_line` 等方法。
- **`BufWriter`**：包装任何 `AsyncWrite` 以缓冲写入操作。只有当缓冲区已满或调用 `flush` 时，数据才会写入底层的写入器。

`BufReader` 示例：
```no_run
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let mut reader = BufReader::new(f);
    let mut buffer = String::new();

    // 将一行读入 buffer
    reader.read_line(&mut buffer).await?;

    println!("{}", buffer);
    Ok(())
}
```

**重要提示**：使用 `BufWriter` 时，必须调用 `.flush()` 以确保在写入器被丢弃前，缓冲区中剩余的任何数据都已写入底层流。

```no_run
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    let mut writer = BufWriter::new(f);

    writer.write_all(b"some bytes").await?;

    // 刷新缓冲区以确保数据写入文件。
    writer.flush().await?;

    Ok(())
}
```

## 工具

### `split()`

`split()` 函数接收一个实现了 `AsyncRead + AsyncWrite` 的值（如 `TcpStream`），并将其拆分为两个独立的句柄：一个 `ReadHalf` 和一个 `WriteHalf`。这对于将流的可读和可写部分移动到不同的任务中非常有用。

- `ReadHalf<T>` 实现了 `AsyncRead`。
- `WriteHalf<T>` 实现了 `AsyncWrite`。

通过在 `ReadHalf` 上调用 `unsplit()` 并传入其对应的 `WriteHalf`，可以重构原始流。

### `join()`

`join()` 函数是 `split()` 的逆操作。它接收两个独立的值，一个实现 `AsyncRead`，另一个实现 `AsyncWrite`，并将它们组合成一个同时实现这两个 Trait 的单一值。

### 标准 I/O

Tokio 为标准输入、输出和错误流提供了异步句柄：

- **`stdin()`**：返回当前进程标准输入的句柄。
- **`stdout()`**：返回标准输出的句柄。
- **`stderr()`**：返回标准错误的句柄。

这些函数必须在 Tokio 运行时的上下文中调用。

### 重新导出

为方便起见，该模块从 `std::io` 重新导出了常用类型，包括 `Error`、`ErrorKind`、`Result` 和 `SeekFrom`。