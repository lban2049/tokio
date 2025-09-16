# OS Signals

Tokio provides tools for handling operating system signals asynchronously, allowing your application to gracefully interact with events like process termination requests or user interruptions. Signal handling is a complex topic, and Tokio's implementation aims to follow best practices, but it's important to understand its behavior and limitations on different platforms.

This guide covers how to listen for signals on both Unix-like systems and Windows.

A common use case is listening for a "ctrl-c" event to gracefully shut down an application.

```rust Graceful Shutdown with Ctrl-C icon=logos:rust
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Press Ctrl+C to exit.");
    signal::ctrl_c().await?;
    println!("Ctrl-C received, shutting down.");
    Ok(())
}
```

## Unix Signals

On Unix-based platforms, you can listen for a wide range of signals using the `tokio::signal::unix` module. This is handled by creating a `Signal` listener for a specific `SignalKind`.

### Creating a Signal Listener

The `signal()` function takes a `SignalKind` and returns a `Signal` struct, which can be used to receive notifications.

```rust Listening for SIGHUP icon=logos:rust
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a stream of hangup (SIGHUP) signals.
    let mut stream = signal(SignalKind::hangup())?;

    // Loop and print a message whenever a SIGHUP signal is received.
    loop {
        stream.recv().await;
        println!("Received SIGHUP signal");
    }
}
```

### Common Signal Kinds

The `SignalKind` struct provides constants for common signals. You can also create a kind from a raw integer value for platform-specific signals using `SignalKind::from_raw()`.

<x-cards data-columns="2">
  <x-card data-title="hangup()" data-icon="lucide:phone-off">
    SIGHUP: Sent when a terminal is disconnected. Often used to signal a configuration reload.
  </x-card>
  <x-card data-title="interrupt()" data-icon="lucide:keyboard">
    SIGINT: Sent to interrupt a program, typically by pressing Ctrl+C.
  </x-card>
  <x-card data-title="terminate()" data-icon="lucide:power-off">
    SIGTERM: Sent to request a graceful process termination.
  </x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">
    SIGQUIT: Sent to request termination and a core dump.
  </x-card>
  <x-card data-title="child()" data-icon="lucide:baby">
    SIGCHLD: Sent when a child process changes state (e.g., terminates).
  </x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user">
    SIGUSR1: A user-defined signal for custom application behavior.
  </x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:users">
    SIGUSR2: A second user-defined signal.
  </x-card>
  <x-card data-title="window_change()" data-icon="lucide:move-horizontal">
    SIGWINCH: Sent when the terminal window is resized.
  </x-card>
</x-cards>

### Important Caveats

- **Permanent Handler**: The first time you create a `Signal` for a specific kind, Tokio installs an OS-level signal handler for that signal. This handler **is never uninstalled** for the lifetime of the process. This means the default system behavior (like terminating on `SIGINT`) is permanently replaced.
- **Signal Coalescing**: If multiple signals of the same kind are received before your code has a chance to process them, they may be coalesced into a single event. Your listener is guaranteed to receive notification of *at least one* signal, but possibly more.

## Windows Signals

On Windows, Tokio handles console control events rather than traditional Unix-style signals. The `tokio::signal::windows` module provides functions to listen for these specific events.

Each function returns a dedicated listener struct that you can await.

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard">
    Creates a listener for `CTRL_C_EVENT` notifications. Returns a `CtrlC` struct.
  </x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:pause-circle">
    Creates a listener for `CTRL_BREAK_EVENT` notifications. Returns a `CtrlBreak` struct.
  </x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-circle">
    Creates a listener for `CTRL_CLOSE_EVENT` when the console window is closed. Returns a `CtrlClose` struct.
  </x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:server-off">
    Creates a listener for `CTRL_SHUTDOWN_EVENT` when the system is shutting down. Returns a `CtrlShutdown` struct.
  </x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out">
    Creates a listener for `CTRL_LOGOFF_EVENT` when the user logs off. Returns a `CtrlLogoff` struct.
  </x-card>
</x-cards>

### Example: Handling Multiple Ctrl-C Events

This example shows how to create a listener that waits for three `CTRL-C` events before exiting.

```rust Windows Ctrl-C Handling icon=logos:rust
use tokio::signal::windows::ctrl_c;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut signal = ctrl_c()?;

    println!("Waiting for CTRL-C events...");
    for i in (0..3).rev() {
        signal.recv().await;
        println!("Got CTRL-C. {} more to exit.", i);
    }

    println!("Exiting.");
    Ok(())
}
```

Like with Unix signals, notifications on Windows are coalesced if they arrive faster than they are processed.

---

After learning how to handle OS signals, you might be interested in managing child processes. For more information, see the [Child Processes](./io-process.md) documentation.