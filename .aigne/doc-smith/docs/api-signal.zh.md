# 信号

Tokio 的异步信号处理。此模块提供了以异步方式处理操作系统信号的工具，并将其集成到 Tokio 运行时中。

信号处理是一个复杂的主题，应谨慎对待。此实现遵循了最佳实践，但仍应根据您应用程序的特定需求进行评估。请注意，特定于操作系统的结构上存在一些基本限制，相关文档已有说明。

### 跨平台的 `ctrl_c`

一个常见的需求是在按下 `Ctrl-C` 时优雅地关闭程序。Tokio 为此提供了一个便捷的跨平台函数。

```rust,no_run
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("等待 Ctrl-C...");
    signal::ctrl_c().await?;
    println!("收到 Ctrl-C，正在关闭。");
    Ok(())
}
```

---

## Unix 特定信号

在 Unix 平台上，`tokio::signal::unix` 提供了主要的 `Signal` 类型，用于接收各种信号的通知。

### 注意事项

使用 Unix 信号时，需要注意一些重要的限制：

*   **信号合并**：如果在处理前收到了多个信号，它们可能会被合并为单个通知。来自流的一个事件对应*至少一个*信号。
*   **持久化信号处理器**：首次为特定信号注册监听器时，会为整个进程的生命周期安装一个操作系统级别的信号处理器。当 `Signal` 实例被丢弃时，该处理器**不会**被注销。这意味着进程的默认行为（如在收到 `SIGINT` 时终止）将被永久替换。

### 创建信号监听器

`signal` 函数为特定的 `SignalKind` 创建一个新的监听器。

```rust,no_run
# #[cfg(unix)] {
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建一个 SIGHUP 信号流。
    let mut stream = signal(SignalKind::hangup())?;

    // 每当收到 HUP 信号时就打印信息。
    loop {
        stream.recv().await;
        println!("got signal HUP");
    }
}
# }
```

### `Signal` 方法

`Signal` 结构体是接收通知的监听器。

| 方法 | 描述 |
|---|---|
| `recv(&mut self)` | 异步等待下一个信号通知。如果流已关闭，则返回 `None`。此方法是取消安全的。 |
| `poll_recv(&mut self, cx: &mut Context<'_'>)` | 在非异步上下文中轮询下一个信号通知。如果信号可用，则返回 `Poll::Ready(Some(()))`。 |


### 信号类型

`SignalKind` 结构体代表一个特定的 Unix 信号。您可以使用预定义的类型，也可以根据原始整数值为特定于平台的信号创建一个类型。

<x-cards data-columns="3">
  <x-card data-title="alarm()" data-icon="lucide:alarm-clock">代表 `SIGALRM` 信号，当实时定时器到期时发送。</x-card>
  <x-card data-title="child()" data-icon="lucide:baby">代表 `SIGCHLD` 信号，当子进程状态改变时发送。</x-card>
  <x-card data-title="hangup()" data-icon="lucide:phone-off">代表 `SIGHUP` 信号，当终端断开连接时发送。</x-card>
  <x-card data-title="interrupt()" data-icon="lucide:keyboard">代表 `SIGINT` 信号，用于中断程序（例如 Ctrl-C）。</x-card>
  <x-card data-title="io()" data-icon="lucide:arrow-right-left">代表 `SIGIO` 信号，当文件描述符上可以进行 I/O 操作时发送。</x-card>
  <x-card data-title="pipe()" data-icon="lucide:pipe">代表 `SIGPIPE` 信号，当向没有读取者的管道写入时发送。</x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">代表 `SIGQUIT` 信号，用于请求进程关闭并生成核心转储。</x-card>
  <x-card data-title="terminate()" data-icon="lucide:siren">代表 `SIGTERM` 信号，用于请求进程优雅地关闭。</x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user">代表 `SIGUSR1` 信号，一个用户定义的信号。</x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:users">代表 `SIGUSR2` 信号，另一个用户定义的信号。</x-card>
  <x-card data-title="window_change()" data-icon="lucide:rectangle-horizontal">代表 `SIGWINCH` 信号，当终端窗口大小调整时发送。</x-card>
  <x-card data-title="from_raw(signum)" data-icon="lucide:hash">根据原始整数信号编号为特定于操作系统的信号创建一个 `SignalKind`。</x-card>
</x-cards>

---

## Windows 特定信号

在 Windows 上，`tokio::signal::windows` 允许通过 `SetConsoleCtrlHandler` 接收控制台控制事件，如 `CTRL_C_EVENT`、`CTRL_BREAK_EVENT` 和关闭事件。

与 Unix 实现类似，通知也会被合并。如果多个事件快速连续发生，监听器可能只会收到单个通知。

### 可用的监听器

有一些函数可用于为特定的控制台控制事件创建监听器。

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard">为 `CTRL_C_EVENT` 通知创建一个监听器。</x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:keyboard">为 `CTRL_BREAK_EVENT` 通知创建一个监听器。</x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-square">为 `CTRL_CLOSE_EVENT` 通知创建一个监听器，当控制台窗口关闭时发送。</x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:power-off">为 `CTRL_SHUTDOWN_EVENT` 通知创建一个监听器，当系统关闭时发送。</x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out">为 `CTRL_LOGOFF_EVENT` 通知创建一个监听器，当用户注销时发送。</x-card>
</x-cards>

### 用法

每个函数都会返回一个相应的结构体（例如，`ctrl_c()` 返回 `CtrlC`）。所有这些结构体都提供了相同的 `recv()` 和 `poll_recv()` 方法来消费事件。

以下是处理 `CTRL-BREAK` 事件的示例：

```rust,no_run
# #[cfg(windows)] {
use tokio::signal::windows::ctrl_break;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 为 CTRL-BREAK 事件创建一个监听器。
    let mut stream = ctrl_break()?;

    // 每当收到 CTRL-BREAK 事件时就打印信息。
    loop {
        stream.recv().await;
        println!("got signal CTRL-BREAK");
    }
}
# }
```

### 监听器方法

所有 Windows 信号监听器结构体（`CtrlC`、`CtrlBreak` 等）都具有以下方法：

| 方法 | 描述 |
|---|---|
| `recv(&mut self)` | 异步等待下一个通知。如果监听器已关闭，则返回 `None`。 |
| `poll_recv(&mut self, cx: &mut Context<'_'>)` | 在非异步上下文中轮询下一个通知。 |
