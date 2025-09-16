# 操作系统信号

Tokio 提供了异步处理操作系统信号的工具，允许你的应用程序优雅地与进程终止请求或用户中断等事件进行交互。信号处理是一个复杂的主题，Tokio 的实现旨在遵循最佳实践，但理解其在不同平台上的行为和局限性非常重要。

本指南介绍了如何在类 Unix 系统和 Windows 上监听信号。

一个常见的用例是监听“ctrl-c”事件以实现应用程序的优雅关闭。

```rust 使用 Ctrl-C 优雅关闭 icon=logos:rust
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("按 Ctrl+C 退出。");
    signal::ctrl_c().await?;
    println!("收到 Ctrl-C，正在关闭。");
    Ok(())
}
```

## Unix 信号

在基于 Unix 的平台上，你可以使用 `tokio::signal::unix` 模块监听各种信号。这是通过为特定的 `SignalKind` 创建一个 `Signal` 监听器来处理的。

### 创建信号监听器

`signal()` 函数接收一个 `SignalKind` 并返回一个 `Signal` 结构体，该结构体可用于接收通知。

```rust 监听 SIGHUP icon=logos:rust
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建一个挂起 (SIGHUP) 信号流。
    let mut stream = signal(SignalKind::hangup())?;

    // 每当接收到 SIGHUP 信号时，循环并打印一条消息。
    loop {
        stream.recv().await;
        println!("收到 SIGHUP 信号");
    }
}
```

### 常见的信号类型

`SignalKind` 结构体为常见信号提供了常量。你也可以使用 `SignalKind::from_raw()` 从原始整数值为特定于平台的信号创建一个类型。

<x-cards data-columns="2">
  <x-card data-title="hangup()" data-icon="lucide:phone-off">
    SIGHUP：当终端断开连接时发送。通常用于通知重新加载配置。
  </x-card>
  <x-card data-title="interrupt()" data-icon="lucide:keyboard">
    SIGINT：用于中断程序，通常通过按 Ctrl+C 发送。
  </x-card>
  <x-card data-title="terminate()" data-icon="lucide:power-off">
    SIGTERM：用于请求进程优雅终止。
  </x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">
    SIGQUIT：用于请求终止并生成核心转储 (core dump)。
  </x-card>
  <x-card data-title="child()" data-icon="lucide:baby">
    SIGCHLD：当子进程状态改变（例如，终止）时发送。
  </x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user">
    SIGUSR1：用户定义的信号，用于自定义应用程序行为。
  </x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:users">
    SIGUSR2：第二个用户定义的信号。
  </x-card>
  <x-card data-title="window_change()" data-icon="lucide:move-horizontal">
    SIGWINCH：当终端窗口大小调整时发送。
  </x-card>
</x-cards>

### 重要注意事项

- **永久处理程序**：当你首次为特定类型的信号创建 `Signal` 时，Tokio 会为该信号安装一个操作系统级别的信号处理程序。该处理程序在进程的整个生命周期内**永远不会被卸载**。这意味着默认的系统行为（例如在接收到 `SIGINT` 时终止进程）将被永久替换。
- **信号合并**：如果在你的代码有机会处理它们之前收到了多个相同类型的信号，它们可能会被合并为单个事件。你的监听器保证会收到*至少一个*信号的通知，但可能不止一个。

## Windows 信号

在 Windows 上，Tokio 处理的是控制台控制事件，而非传统的类 Unix 风格信号。`tokio::signal::windows` 模块提供了用于监听这些特定事件的函数。

每个函数都会返回一个专用的监听器结构体，你可以对其进行 await 操作。

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard">
    为 `CTRL_C_EVENT` 通知创建一个监听器。返回一个 `CtrlC` 结构体。
  </x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:pause-circle">
    为 `CTRL_BREAK_EVENT` 通知创建一个监听器。返回一个 `CtrlBreak` 结构体。
  </x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-circle">
    当控制台窗口关闭时，为 `CTRL_CLOSE_EVENT` 创建一个监听器。返回一个 `CtrlClose` 结构体。
  </x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:server-off">
    当系统关闭时，为 `CTRL_SHUTDOWN_EVENT` 创建一个监听器。返回一个 `CtrlShutdown` 结构体。
  </x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out">
    当用户注销时，为 `CTRL_LOGOFF_EVENT` 创建一个监听器。返回一个 `CtrlLogoff` 结构体。
  </x-card>
</x-cards>

### 示例：处理多个 Ctrl-C 事件

此示例展示了如何创建一个在退出前等待三个 `CTRL-C` 事件的监听器。

```rust Windows Ctrl-C 处理 icon=logos:rust
use tokio::signal::windows::ctrl_c;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut signal = ctrl_c()?;

    println!("正在等待 CTRL-C 事件...");
    for i in (0..3).rev() {
        signal.recv().await;
        println!("收到 CTRL-C。再按 {} 次退出。", i);
    }

    println!("正在退出。");
    Ok(())
}
```

与 Unix 信号类似，如果 Windows 上的通知到达速度快于其处理速度，它们也会被合并。

---

学习了如何处理操作系统信号后，你可能对管理子进程感兴趣。更多信息，请参阅 [子进程](./io-process.md) 文档。