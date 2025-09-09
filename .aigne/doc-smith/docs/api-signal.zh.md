# 信号

Tokio 的异步信号处理。该模块提供了以异步方式接收和处理操作系统信号的工具。

请注意，信号处理是一个复杂的主题，应谨慎使用。此实现遵循了最佳实践，但您仍需评估其是否适合您应用程序的特定需求。此外，针对特定操作系统的结构，也存在一些已记录的基本限制。

### 跨平台 Ctrl-C

Tokio 提供了一个方便的跨平台 future，当进程接收到 `Ctrl-C` 信号时，该 future 会被解析。

```rust 处理 Ctrl-C icon=logos:rust
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("正在等待 Ctrl-C...");
    signal::ctrl_c().await?;
    println!("已接收到 Ctrl-C，正在关闭。");
    Ok(())
}
```

---

## Unix

在 Unix 平台上，您可以为任意信号创建监听器。这些监听器以流的形式表示，每接收到一个信号，就会产生一个单元类型 `()`。

### `signal()`

`signal` 函数会创建一个新的 `Signal` 流，用于监听特定的 `SignalKind`。

```rust 等待 SIGHUP icon=logos:rust
# #[cfg(unix)] {
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建一个挂断信号流。
    let mut stream = signal(SignalKind::hangup())?;

    // 每当接收到 HUP 信号时打印信息。
    println!("正在等待 SIGHUP...");
    loop {
        stream.recv().await;
        println!("已接收到 SIGHUP");
    }
}
# }
```

### 注意事项

在 Tokio 中使用 Unix 信号时，需要注意以下几个重要限制：

*   **信号合并**：如果在轮询流之前接收到多个相同类型的信号，它们可能会被合并成一个事件。该流保证每个产生的项都至少对应一个接收到的信号。
*   **永久处理器**：当首次为特定信号创建监听器时，Tokio 会为该信号安装一个全局的操作系统信号处理器。此处理器将替换整个进程生命周期内的默认系统行为，并且即使 `Signal` 流被丢弃，该处理器也不会被卸载。

### `SignalKind`

`SignalKind` 结构体代表一个特定的 Unix 信号，并为常见信号提供了构造方法。

<x-cards data-columns="3">
  <x-card data-title="alarm()" data-icon="lucide:alarm-clock">代表 `SIGALRM` 信号，通常在实时定时器到期时发送。</x-card>
  <x-card data-title="child()" data-icon="lucide:baby">代表 `SIGCHLD` 信号，在子进程状态改变时发送。</x-card>
  <x-card data-title="hangup()" data-icon="lucide:phone-off">代表 `SIGHUP` 信号，在终端断开连接时发送。</x-card>
  <x-card data-title="interrupt()" data-icon="lucide:hand">代表 `SIGINT` 信号，用于中断程序（例如 Ctrl-C）。</x-card>
  <x-card data-title="io()" data-icon="lucide:arrow-left-right">代表 `SIGIO` 信号，在文件描述符上可以进行 I/O 操作时发送。</x-card>
  <x-card data-title="pipe()" data-icon="lucide:pipeline">代表 `SIGPIPE` 信号，在向没有读取者的管道写入时发送。</x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">代表 `SIGQUIT` 信号，用于请求进程关闭并生成核心转储。</x-card>
  <x-card data-title="terminate()" data-icon="lucide:power-off">代表 `SIGTERM` 信号，用于请求进程优雅地关闭。</x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user">代表 `SIGUSR1` 信号，用于用户自定义目的。</x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:user-cog">代表 `SIGUSR2` 信号，用于用户自定义目的。</x-card>
  <x-card data-title="window_change()" data-icon="lucide:maximize">代表 `SIGWINCH` 信号，在终端窗口大小调整时发送。</x-card>
  <x-card data-title="from_raw()" data-icon="lucide:hash">允许通过提供原始整数值来监听任何有效的操作系统信号。</x-card>
</x-cards>

---

## Windows

在 Windows 上，信号处理基于接收控制台控制事件。Tokio 为每种事件类型都提供了独立的函数来创建监听器。

### 控制台事件监听器

每个函数都会返回一个监听器结构体（例如 `CtrlC`、`CtrlBreak`），可用于异步等待相应的事件。

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard" data-href="#">
    创建一个接收 “ctrl-c” 通知的监听器。
  </x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:keyboard" data-href="#">
    创建一个接收 “ctrl-break” 通知的监听器。
  </x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-square" data-href="#">
    创建一个在控制台关闭时接收 “ctrl-close” 通知的监听器。
  </x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out" data-href="#">
    创建一个在用户注销时接收 “ctrl-logoff” 通知的监听器。
  </x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:power" data-href="#">
    创建一个在系统关闭时接收 “ctrl-shutdown” 通知的监听器。
  </x-card>
</x-cards>

与 Unix 信号类似，这些通知也会被合并。如果多个相同类型的事件快速发生，监听器可能只会产生一个通知。

### 示例

以下示例演示了如何监听 `CTRL-BREAK` 事件。

```rust 在 Windows 上处理 CTRL-BREAK icon=logos:rust
use tokio::signal::windows::ctrl_break;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = ctrl_break()?;

    println!("正在等待 CTRL-BREAK...");
    stream.recv().await;
    println!("已接收到 CTRL-BREAK！");

    Ok(())
}
```