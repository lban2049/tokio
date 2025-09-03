# 异步 I/O

Tokio 提供了一套非阻塞 I/O 原语，用于构建高性能网络应用程序、管理文件系统操作以及与子进程交互。与标准库的 `std::io` 不同，`std::io` 会在等待 I/O 完成时阻塞当前线程，而 Tokio 的 I/O 操作是异步的。当一个操作无法立即完成时，它会将控制权交还给 Tokio 调度器，从而允许其他任务运行。这是用少量线程实现高并发的关键。

Tokio 中所有的异步 I/O 都建立在 `tokio::io` 模块的两个核心 trait 之上：`AsyncRead` 和 `AsyncWrite`。它们是标准库中 `Read` 和 `Write` trait 的异步等价物。

## 核心 I/O 原语

`AsyncRead` 和 `AsyncWrite` trait 为异步字节流提供了基本的构建块。然而，你很少会直接实现或使用这些 trait 的核心方法。相反，你会使用由 `AsyncReadExt` 和 `AsyncWriteExt` 扩展 trait 提供的便捷实用方法，这些方法对于任何实现 `AsyncRead` 或 `AsyncWrite` 的类型都是自动可用的。

例如，要从异步源读取数据，你可以使用 `AsyncReadExt` 中的 `.read()` 方法：

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

async fn read_from_file() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

### 缓冲 I/O

为了通过减少系统调用次数来提高效率，Tokio 提供了类似于标准库的缓冲读取器和写入器。`BufReader` 和 `BufWriter` 结构体分别包装了任何 `AsyncRead` 或 `AsyncWrite` 类型。`BufReader` 还支持更便捷的方法，例如逐行读取数据。

```rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

async fn read_lines_from_file() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let mut reader = BufReader::new(f);
    let mut buffer = String::new();

    // read a line into buffer
    reader.read_line(&mut buffer).await?;

    println!("{}", buffer);
    Ok(())
}
```

## Tokio 的 I/O 模型

Tokio 运行时协调所有异步操作。对于基于网络的 I/O，它使用操作系统最高效的事件通知系统（如 Linux 上的 epoll、macOS 上的 kqueue 或 Windows 上的 IOCP）。对于文件系统操作，由于其在操作系统层面通常是阻塞的，Tokio 使用一个专用的线程池来确保主调度器永远不会被阻塞。

```d2
direction: down

"应用程序任务" {
  shape: package
}

"Tokio 运行时" {
  grid-columns: 2
  
  "核心调度器": {
    shape: rectangle
    "根据事件唤醒任务"
  }

  "I/O 驱动 (epoll, kqueue, IOCP)": {
    shape: rectangle
    "处理非阻塞资源"
  }
  
  "计时器": {
    shape: rectangle
    "管理休眠和超时"
  }

  "阻塞线程池": {
    shape: rectangle
    "用于 CPU 密集型或阻塞的系统调用"
  }
}

"操作系统资源" {
    shape: package
    grid-columns: 2

    "网络 (TCP, UDP, UDS)": { shape: cylinder }
    "文件系统": { shape: cylinder }
    "子进程": { shape: cylinder }
    "操作系统信号": { shape: cylinder }
}

"应用程序任务" <-> "Tokio 运行时"."核心调度器": "派生和让出"
"Tokio 运行时"."核心调度器" -> "Tokio 运行时"."I/O 驱动 (epoll, kqueue, IOCP)": "注册 I/O 关注点"
"Tokio 运行时"."I/O 驱动 (epoll, kqueue, IOCP)" -> "Tokio 运行时"."核心调度器": "就绪时通知"
"Tokio 运行时"."核心调度器" -> "Tokio 运行时"."阻塞线程池": "分派阻塞工作"
"Tokio 运行时"."阻塞线程池" -> "Tokio 运行时"."核心调度器": "返回结果"

"Tokio 运行时"."I/O 驱动 (epoll, kqueue, IOCP)" <-> "操作系统资源"."网络 (TCP, UDP, UDS)": "非阻塞"
"Tokio 运行时"."I/O 驱动 (epoll, kqueue, IOCP)" <-> "操作系统资源"."子进程": "非阻塞"
"Tokio 运行时"."I/O 驱动 (epoll, kqueue, IOCP)" <-> "操作系统资源"."操作系统信号": "非阻塞"
"Tokio 运行时"."阻塞线程池" <-> "操作系统资源"."文件系统": "阻塞"
```

## I/O 资源类型

Tokio 提供了一套丰富的 I/O 类型，涵盖了大多数常见用例。每种类型都旨在无缝集成到异步运行时中。

<x-cards data-columns="2">
  <x-card data-title="网络" data-icon="lucide:globe">
    用于 TCP、UDP 和 Unix 套接字。这些类型直接与操作系统的事件队列集成，以实现高效的非阻塞网络通信。
  </x-card>
  <x-card data-title="文件系统" data-icon="lucide:folder">
    用于异步文件操作。由于大多数操作系统缺乏真正的异步文件 API，这些操作在专用的阻塞线程池上运行，以避免阻塞运行时。
  </x-card>
  <x-card data-title="进程" data-icon="lucide:terminal">
    用于派生和管理子进程。子进程的标准输入、输出和错误流都作为异步 I/O 句柄公开。
  </x-card>
  <x-card data-title="信号" data-icon="lucide:siren">
    用于异步处理 Unix 和 Windows 操作系统信号。这使你的应用程序能够优雅地响应系统事件，如终止请求。
  </x-card>
</x-cards>

## 总结

Tokio 的 I/O 模型为构建可靠的应用程序提供了一个一致且强大的基础。通过在 `AsyncRead` 和 `AsyncWrite` trait 背后抽象不同类型的 I/O，你可以编写通用的代码，使其能够跨网络套接字、文件和进程流工作。

既然你已经了解了 I/O 的基础知识，下一步最好是到[同步](./concepts-synchronization.md)部分探索如何在执行 I/O 的任务之间管理共享数据。
