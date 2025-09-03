# 信号

该模块为异步信号处理提供支持。信号是一种进程间通信的形式，正确处理它们可能很复杂。此实现旨在遵循最佳实践，但您应评估其是否适合您的特定应用需求。

请注意，信号处理是平台特定的。Tokio 为 Unix 和 Windows 提供了不同的 API，以适应它们各自不同的模型。

一个常见的用例是在按下 `CTRL+C` 时正常关闭服务器。

```rust,no_run
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Waiting for CTRL+C...");
    signal::ctrl_c().await?;
    println!("CTRL+C received, shutting down.");
    Ok(())
}
```

---

## Unix 信号

在 Unix 平台上，您可以监听任意信号。用于此目的的主要类型是 `Signal`（表示监听器）和 `SignalKind`（指定信号）。

### `signal()`

为指定的信号类型创建一个新的监听器。该函数返回一个 `Signal` 实例，可用于接收通知。

**重要注意事项：**

*   **信号合并**：如果在轮询监听器之前接收到多个信号，它们将被合并为一个通知。轮询后，下一个信号保证会生成一个新的通知。
*   **持久化处理器**：当首次为特定信号创建监听器时，会为整个进程的生命周期安装一个操作系统级别的信号处理器。即使 `Signal` 实例被丢弃，此行为也不会重置。例如，监听 `SIGINT` 将阻止后续 `SIGINT` 信号的默认进程终止行为。

**示例：等待 `SIGHUP`**

```rust,no_run
# #[cfg(unix)] {
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建一个 SIGHUP 信号流。
    let mut stream = signal(SignalKind::hangup())?;

    // 每当接收到 HUP 信号时，打印信息。
    loop {
        stream.recv().await;
        println!("got signal HUP");
    }
}
# }
```

### `Signal` 结构体

用于接收特定操作系统信号的监听器。它提供两种主要方法来接收通知。

*   `recv()`：一个 `async` 方法，在接收到下一个信号时完成。
*   `poll_recv()`：一个用于手动实现 `Future` 的方法，用于轮询下一个信号。

### `SignalKind` 结构体

表示要监听的特定信号类型。它为常见信号提供了构造方法。

<x-cards data-columns="3">
  <x-card data-title="alarm()" data-icon="lucide:alarm-clock">SIGALRM：当实时定时器到期时发送。</x-card>
  <x-card data-title="child()" data-icon="lucide:baby">SIGCHLD：当子进程的状态发生变化时发送。</x-card>
  <x-card data-title="hangup()" data-icon="lucide:phone-off">SIGHUP：当终端断开连接时发送。</x-card>
  <x-card data-title="interrupt()" data-icon="lucide:keyboard">SIGINT：用于中断程序（例如，Ctrl+C）。</x-card>
  <x-card data-title="io()" data-icon="lucide:arrow-left-right">SIGIO：当文件描述符上可以进行 I/O 操作时发送。</x-card>
  <x-card data-title="pipe()" data-icon="lucide:pipeline">SIGPIPE：当向没有读取者的管道写入时发送。</x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">SIGQUIT：用于终止进程并转储核心。</x-card>
  <x-card data-title="terminate()" data-icon="lucide:shield-x">SIGTERM：用于请求进程正常关闭。</x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user-cog">SIGUSR1：用户定义的信号。</x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:user-cog">SIGUSR2：用户定义的信号。</x-card>
  <x-card data-title="window_change()" data-icon="lucide:rectangle-horizontal">SIGWINCH：当终端窗口大小调整时发送。</x-card>
  <x-card data-title="from_raw()" data-icon="lucide:hash">允许通过其原始整数值监听任何有效的操作系统信号。</x-card>
</x-cards>

---

## Windows 信号

在 Windows 上，信号处理基于控制台控制事件。Tokio 提供了独立的函数来为每种特定事件类型创建监听器。

与 Unix 实现类似，通知会被合并。如果在轮询监听器之前发生多个相同类型的事件，它们将作为单个通知被传递。

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard">创建一个监听器，在用户按下 `Ctrl+C` 时接收通知。</x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:keyboard">为 `Ctrl+Break` 事件创建一个监听器。</x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-square">为 `Ctrl+Close` 事件创建一个监听器，当控制台窗口关闭时发送。</x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:power-off">为 `Ctrl+Shutdown` 事件创建一个监听器，当系统关闭时发送。</x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out">为 `Ctrl+Logoff` 事件创建一个监听器，当用户注销时发送。</x-card>
</x-cards>

这些函数各自返回一个专用的结构体（例如 `CtrlC`、`CtrlBreak`），带有 `recv()` 和 `poll_recv()` 方法，其行为与它们的 Unix 对应部分完全相同。

**示例：处理 `Ctrl+Break`**

```rust,no_run
use tokio::signal::windows::ctrl_break;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut stream = ctrl_break()?;

    loop {
        stream.recv().await;
        println!("got CTRL-BREAK signal");
    }
}
```

现在您已经了解了信号处理，您可能对异步管理子进程感兴趣。有关更多详细信息，请参阅 [进程](./api-process.md) 文档。