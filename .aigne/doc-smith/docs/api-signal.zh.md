# 信号

本模块提供对异步信号处理的支持。信号处理是一个复杂的主题，应谨慎对待。此实现旨在遵循最佳实践，但您应评估其是否适合您的特定需求。

特定于操作系统的结构中记录了一些基本限制。

## 跨平台：处理 Ctrl-C

Tokio 提供了一个方便的跨平台函数来监听 `ctrl-c` 信号（在 Unix 上为 `SIGINT`）。这通常是在应用程序中处理优雅关闭的最简单方法。

```rust,no_run
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("等待 ctrl-c...");
    signal::ctrl_c().await?;
    println!("已收到 ctrl-c！");
    Ok(())
}
```

## 平台特定信号

对于更高级或平台特定的信号处理，Tokio 为 Unix 和 Windows 均提供了模块。

### Unix 信号

`tokio::signal::unix` 模块提供了主要的 `Signal` 类型，用于接收各种 Unix 信号的通知。

#### `signal()`

要为特定信号创建监听器，请使用 `signal` 函数并提供一个 `SignalKind`。

```rust,no_run
# #[cfg(unix)] {
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建一个 SIGHUP 信号流。
    let mut stream = signal(SignalKind::hangup())?;

    // 每当收到 HUP 信号时打印。
    loop {
        stream.recv().await;
        println!("got signal HUP");
    }
}
# }
```

#### Signal 结构体

`signal` 函数返回一个 `Signal` 结构体，它充当特定信号类型的监听器，可用于异步等待信号。

**方法：**

*   `recv(&mut self) -> Option<()>`: 异步等待下一个信号通知。
*   `poll_recv(&mut self, cx: &mut Context<'_>) -> Poll<Option<()>>`: 在非异步上下文中轮询下一个信号通知。

#### 重要注意事项

使用 Unix 信号处理程序时，请注意以下几点：

1.  **处理程序生命周期**：首次为特定信号类型创建 `Signal` 时，会为*进程的整个生命周期*安装一个操作系统信号处理程序，并替换平台的默认行为。即使 `Signal` 实例被丢弃，该处理程序也**不会**被重置。例如，在监听一次 `SIGINT` 后，进程将不再默认因 `SIGINT` 而终止。

2.  **信号合并**：信号可能会被合并。如果在轮询监听器之前收到多个信号，它们可能会合并成一个事件。监听器保证每个事件对应*至少一个*信号。

#### `SignalKind`

此结构体表示要监听的特定信号类型。常用信号可作为常量使用。

| Method | Signal | Description |
|---|---|---|
| `alarm()` | `SIGALRM` | 当实时计时器到期时发送。 |
| `child()` | `SIGCHLD` | 当子进程的状态发生变化时发送。 |
| `hangup()` | `SIGHUP` | 当控制终端断开连接时发送。 |
| `interrupt()` | `SIGINT` | 用于中断程序（例如，ctrl-c）。 |
| `io()` | `SIGIO` / `SIGPOLL` | 当可以在文件描述符上执行 I/O 操作时发送。 |
| `pipe()` | `SIGPIPE` | 当向没有读取者的管道写入时发送。 |
| `quit()` | `SIGQUIT` | 用于请求进程关闭并生成核心转储。 |
| `terminate()` | `SIGTERM` | 用于请求进程优雅关闭。 |
| `user_defined1()` | `SIGUSR1` | 用户自定义信号 1。 |
| `user_defined2()` | `SIGUSR2` | 用户自定义信号 2。 |
| `window_change()` | `SIGWINCH` | 当终端窗口大小调整时发送。 |

对于平台特定或不太常见的信号，您可以从原始整数值创建 `SignalKind`：

```rust,no_run
# use tokio::signal::unix::SignalKind;
# let signum = -1;
// let signum = libc::OS_SPECIFIC_SIGNAL;
let kind = SignalKind::from_raw(signum);
```

### Windows 事件

`tokio::signal::windows` 模块允许通过 `SetConsoleCtrlHandler` 函数接收“ctrl-c”和“ctrl-break”等控制台控制事件。

与 Unix 信号类似，如果处理不够迅速，这些通知也会被合并。

#### `ctrl_c`

为“ctrl-c”事件创建一个监听器。

```rust,no_run
use tokio::signal::windows::ctrl_c;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_c()?;
    stream.recv().await;
    println!("收到 ctrl-c");
    Ok(())
}
```

#### `ctrl_break`

为“ctrl-break”事件创建一个监听器。

```rust,no_run
use tokio::signal::windows::ctrl_break;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_break()?;
    stream.recv().await;
    println!("收到 ctrl-break");
    Ok(())
}
```

#### `ctrl_close`

为控制台关闭事件创建一个监听器。

```rust,no_run
use tokio::signal::windows::ctrl_close;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_close()?;
    stream.recv().await;
    println!("收到 ctrl-close");
    Ok(())
}
```

#### `ctrl_shutdown`

为系统关闭事件创建一个监听器。

```rust,no_run
use tokio::signal::windows::ctrl_shutdown;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_shutdown()?;
    stream.recv().await;
    println!("收到 ctrl-shutdown");
    Ok(())
}
```

#### `ctrl_logoff`

为用户注销事件创建一个监听器。

```rust,no_run
use tokio::signal::windows::ctrl_logoff;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_logoff()?;
    stream.recv().await;
    println!("收到 ctrl-logoff");
    Ok(())
}
```

这些函数各自返回一个结构体（例如 `CtrlC`、`CtrlBreak`），其中包含 `recv()` 和 `poll_recv()` 方法，其行为与 Unix 的 `Signal` 结构体类似。