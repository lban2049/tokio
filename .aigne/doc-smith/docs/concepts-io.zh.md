# 异步 I/O

Tokio 的核心是其非阻塞的异步 I/O 模型。这是 `std::io` 的异步版本，旨在防止应用程序在等待 I/O 操作完成时阻塞线程。任务无需等待，而是可以将控制权交还给 Tokio 调度器，从而允许它运行其他任务。这是仅用少量操作系统线程就能构建可处理海量并发连接的应用程序的关键。

Tokio 提供了一套全面的工具来满足各种 I/O 需求，从网络到文件系统操作以及进程间通信。

## 核心 I/O Trait

与标准库一样，Tokio 的 I/O 功能也围绕着一对核心 Trait 构建：`AsyncRead` 和 `AsyncWrite`。它们是 `std::io::Read` 和 `std::io::Write` 的异步对应版本。

- **`AsyncRead`**：一个用于可以异步读取的类型的 Trait。
- **`AsyncWrite`**：一个用于可以异步写入的类型的 Trait。

与它们的同步对应版本不同，这些 Trait 只包含异步操作所必需的方法。`AsyncReadExt` 和 `AsyncWriteExt` 这两个扩展 Trait 提供了一组丰富的实用方法（如 `read_to_string`、`write_all` 等），任何实现了 `AsyncRead` 或 `AsyncWrite` 的类型都可以自动使用它们。

例如，以下是如何从文件中读取最多 10 个字节：

```rust Rust code for reading from a file icon=logos:rust
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

为了提高效率并减少系统调用次数，Tokio 提供了缓冲读取器和写入器，类似于标准库。`BufReader` 和 `BufWriter` 结构分别包装任何 `AsyncRead` 或 `AsyncWrite` 类型，以缓冲操作。`BufReader` 还支持更便捷的方法，例如逐行读取。

```rust Reading a line from a file icon=logos:rust
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

使用 `BufWriter` 时，请记得调用 `.flush().await` 以确保所有缓冲数据都已写入底层的写入器。

## 异步 I/O 的类型

Tokio 为不同类型的 I/O 和与操作系统的异步交互提供了一组模块。

<x-cards data-columns="2">
  <x-card data-title="网络" data-icon="lucide:network" data-href="/api/net">
    用于构建高性能网络服务的非阻塞 TCP、UDP 和 Unix 域套接字。
  </x-card>
  <x-card data-title="文件系统" data-icon="lucide:folder" data-href="/api/fs">
    用于文件和文件系统操作的异步 API，例如读取、写入和创建目录。
  </x-card>
  <x-card data-title="进程" data-icon="lucide:terminal" data-href="/api/process">
    用于异步生成和管理子进程的工具，包括捕获其标准 I/O 流。
  </x-card>
  <x-card data-title="信号" data-icon="lucide:radio-tower" data-href="/api/signal">
    异步处理 Unix 和 Windows 操作系统信号，以实现优雅关闭和其他基于信号的逻辑。
  </x-card>
</x-cards>

### 网络示例：TCP 回显服务器

这是一个简单的 TCP 回显服务器的完整示例，它侦听传入的连接并将接收到的任何数据写回客户端。

```rust A simple TCP echo server icon=logos:rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write the data back.
            loop {
                let n = match socket.read(&mut buf).await {
                    // socket closed
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // Write the data back
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

### 文件系统：阻塞的现实

需要注意的是，大多数操作系统并不提供真正的异步文件系统 API。为了解决这个问题，Tokio 的 `fs` 模块在内部使用了 `spawn_blocking` 函数。这意味着文件操作是在一个专门用于阻塞任务的线程池上执行的，从而防止它们阻塞运行时核心线程上的主要异步任务。

虽然这提供了一个异步 API，但它会带来性能方面的影响。为了获得最佳性能，建议将文件操作分批合并为尽可能少的调用，例如一次性使用 `tokio::fs::write` 写入整个文件，或者将 `File` 包装在 `BufWriter` 中。

### 进程管理

Tokio 允许你使用 `tokio::process::Command` 异步管理子进程。它提供了一个熟悉的构建器 API，类似于 `std::process::Command`，但其执行方法是 `async` 的。

以下是生成 `echo` 命令并捕获其输出的示例：

```rust Spawning a command and capturing output icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Use `output` which returns a future instead of
    // a `Child` immediately.
    let output = Command::new("echo").arg("hello").arg("world")
                        .output()
                        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");
    Ok(())
}
```

### 标准 I/O

Tokio 还通过 `tokio::io::stdin`、`stdout` 和 `stderr` 函数为标准输入、输出和错误提供了异步 API。这些是标准库句柄的异步版本，并实现了 `AsyncRead` 和 `AsyncWrite`。请注意，这些函数**必须**在 Tokio 运行时的上下文中调用。

## 后续步骤

在对异步 I/O 有了扎实的理解之后，你就可以探索如何管理任务之间的状态和通信了。

<x-card data-title="同步" data-icon="lucide:link" data-href="/concepts/synchronization" data-cta="了解同步">
  探索 Tokio 的同步原语，如通道和互斥锁，用于协调异步任务。
</x-card>