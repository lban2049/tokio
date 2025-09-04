# 异步 I/O

Tokio 提供了一整套全面的非阻塞 I/O 原语，用于构建高性能网络应用程序、处理文件系统以及管理进程间通信。这些实用工具被设计为异步的，这意味着它们与 Tokio 运行时集成以避免线程阻塞，从而允许少量线程处理大量并发操作。

本节将介绍 Tokio I/O 操作背后的基本概念。有关详细的 API 信息，请参阅 [API 参考](./api.md)。

## `tokio::io` 模块：核心原语

Tokio I/O 的基础是 `tokio::io` 模块，它是 `std::io` 的异步等价物。它定义了两个基本的 trait：

- **`AsyncRead`**：`std::io::Read` 的异步版本，用于从源读取字节。
- **`AsyncWrite`**：`std::io::Write` 的异步版本，用于向目标写入字节。

当对 `AsyncRead` 或 `AsyncWrite` 类型的操作需要等待数据时，它会把控制权交还给 Tokio 调度器，而不是阻塞线程。这使得其他任务可以在该 I/O 操作等待期间运行。

这些 trait 的实用方法通过 `AsyncReadExt` 和 `AsyncWriteExt` 扩展 trait 提供，任何实现了 `AsyncRead` 或 `AsyncWrite` 的类型都可以自动使用它们。

以下示例从文件中读取最多 10 个字节：

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

### 缓冲读取器和写入器

为了提高效率并减少系统调用次数，Tokio 提供了类似于标准库的缓冲 I/O 类型。`BufReader` 和 `BufWriter` 结构体可以包装任何异步读取器或写入器，以提供内存缓冲。

`BufReader` 添加了 `read_line` 等便捷方法，用于分块读取数据：

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

使用 `BufWriter` 时，调用 `flush()` 方法非常重要，以确保所有缓冲数据都已写入底层的写入器。

```rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // 向缓冲区写入一个字节。
        writer.write(&[42u8]).await?;

        // 在缓冲区离开作用域前刷新它。
        writer.flush().await?;

    } // 除非刷新，否则缓冲区在 drop 时会被丢弃。

    Ok(())
}
```

## 使用 `tokio::net` 进行网络编程

`tokio::net` 模块提供了异步的 TCP、UDP 和 Unix 域套接字 API。这些类型与 Tokio 运行时集成，可以在不阻塞的情况下处理网络 I/O。

主要组件包括：
- **`TcpListener` 和 `TcpStream`**：用于构建 TCP 客户端和服务器。
- **`UdpSocket`**：用于 UDP 通信。
- **`UnixListener` 和 `UnixStream`**：用于在类 Unix 系统上通过 Unix 套接字进行基于流的通信。

下面是一个简单的 TCP 回显服务器示例，它接受连接并将接收到的任何数据写回。

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // 在循环中，从套接字读取数据并将其写回。
            loop {
                let n = match socket.read(&mut buf).await {
                    Ok(0) => return, // 套接字已关闭
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("从套接字读取失败；err = {:?}", e);
                        return;
                    }
                };

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("写入套接字失败；err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

## 使用 `tokio::fs` 进行文件系统操作

`tokio::fs` 模块提供了用于文件和目录操作的异步 API。需要注意的是，大多数操作系统并不提供真正的异步文件系统 API。为了解决这个问题，Tokio 使用其阻塞线程池 (`spawn_blocking`) 在后台执行文件系统操作，从而避免阻塞主运行时线程。

该模块适用于普通文件。对于命名管道等特殊文件，最好使用 `tokio::net::unix::pipe` 等专用类型。

以下是如何将文件的全部内容读取到一个字符串中：

```rust
async fn read_file_contents() -> std::io::Result<()> {
    let contents = tokio::fs::read_to_string("my_file.txt").await?;
    println!("File has {} lines.", contents.lines().count());
    Ok(())
}
```

## 使用 `tokio::process` 管理进程

Tokio 允许通过 `tokio::process` 模块异步管理子进程。`Command` 结构体模仿了 `std::process::Command` 的 API，但提供了 `spawn`、`status` 和 `output` 等异步方法。

此示例生成 `echo` 命令并捕获其输出：

```rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let output = Command::new("echo")
        .arg("hello")
        .arg("world")
        .output()
        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");

    Ok(())
}
```

## 后续步骤

你已经了解了 Tokio 中异步 I/O 的核心概念。要更深入地研究特定 API，请浏览以下部分：

<x-cards data-columns="2">
  <x-card data-title="I/O API 参考" data-icon="lucide:file-text" data-href="/api/io">
    关于异步 I/O trait、辅助函数和类型定义的详细文档。
  </x-card>
  <x-card data-title="网络 API 参考" data-icon="lucide:globe" data-href="/api/net">
    用于网络通信的 TCP、UDP 和 Unix 套接字类型的 API 文档。
  </x-card>
  <x-card data-title="文件系统 API 参考" data-icon="lucide:folder" data-href="/api/fs">
    关于异步文件和文件系统操作的 API 文档。
  </x-card>
  <x-card data-title="进程 API 参考" data-icon="lucide:terminal-square" data-href="/api/process">
    关于异步进程管理的 API 文档。
  </x-card>
</x-cards>