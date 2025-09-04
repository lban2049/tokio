# Signals

Asynchronous signal handling for Tokio. This module provides tools to handle OS signals in an asynchronous manner, integrating them into the Tokio runtime.

Signal handling is a complex topic and should be approached with care. This implementation follows best practices but should be evaluated for your application's specific needs. Note that there are fundamental limitations documented on the OS-specific structures.

### Cross-Platform `ctrl_c`

A common requirement is to gracefully shut down on `Ctrl-C`. Tokio provides a convenient, cross-platform function for this.

```rust,no_run
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Waiting for Ctrl-C...");
    signal::ctrl_c().await?;
    println!("Ctrl-C received, shutting down.");
    Ok(())
}
```

---

## Unix-Specific Signals

On Unix platforms, `tokio::signal::unix` provides the primary `Signal` type for receiving notifications for a wide range of signals.

### Caveats

There are important limitations to keep in mind when using Unix signals:

*   **Signal Coalescing**: If multiple signals are received before being processed, they may be coalesced into a single notification. An event from the stream corresponds to *at least one* signal.
*   **Persistent Signal Handlers**: The first time a listener is registered for a particular signal, an OS-level signal handler is installed for the entire duration of the process. This handler is **not** unregistered when the `Signal` instance is dropped. This means the default process behavior (like terminating on `SIGINT`) is permanently replaced.

### Creating a Signal Listener

The `signal` function creates a new listener for a specific `SignalKind`.

```rust,no_run
# #[cfg(unix)] {
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a stream of SIGHUP signals.
    let mut stream = signal(SignalKind::hangup())?;

    // Print whenever a HUP signal is received.
    loop {
        stream.recv().await;
        println!("got signal HUP");
    }
}
# }
```

### `Signal` Methods

The `Signal` struct is the listener that receives notifications.

| Method | Description |
|---|---|
| `recv(&mut self)` | Asynchronously waits for the next signal notification. Returns `None` if the stream is closed. This method is cancel-safe. |
| `poll_recv(&mut self, cx: &mut Context<'_'>)` | Polls for the next signal notification in a non-async context. Returns `Poll::Ready(Some(()))` if a signal is available. |


### Signal Kinds

The `SignalKind` struct represents a specific Unix signal. You can use predefined kinds or create one from a raw integer value for platform-specific signals.

<x-cards data-columns="3">
  <x-card data-title="alarm()" data-icon="lucide:alarm-clock">Represents the `SIGALRM` signal, sent when a real-time timer expires.</x-card>
  <x-card data-title="child()" data-icon="lucide:baby">Represents the `SIGCHLD` signal, sent when a child process changes status.</x-card>
  <x-card data-title="hangup()" data-icon="lucide:phone-off">Represents the `SIGHUP` signal, sent when a terminal is disconnected.</x-card>
  <x-card data-title="interrupt()" data-icon="lucide:keyboard">Represents the `SIGINT` signal, sent to interrupt a program (e.g., Ctrl-C).</x-card>
  <x-card data-title="io()" data-icon="lucide:arrow-right-left">Represents the `SIGIO` signal, sent when I/O is possible on a file descriptor.</x-card>
  <x-card data-title="pipe()" data-icon="lucide:pipe">Represents the `SIGPIPE` signal, sent on write to a pipe with no readers.</x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">Represents the `SIGQUIT` signal, sent to request a process shutdown and core dump.</x-card>
  <x-card data-title="terminate()" data-icon="lucide:siren">Represents the `SIGTERM` signal, sent to request a graceful process shutdown.</x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user">Represents the `SIGUSR1` signal, a user-defined signal.</x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:users">Represents the `SIGUSR2` signal, another user-defined signal.</x-card>
  <x-card data-title="window_change()" data-icon="lucide:rectangle-horizontal">Represents the `SIGWINCH` signal, sent when the terminal window is resized.</x-card>
  <x-card data-title="from_raw(signum)" data-icon="lucide:hash">Creates a `SignalKind` from a raw integer signal number for OS-specific signals.</x-card>
</x-cards>

---

## Windows-Specific Signals

On Windows, `tokio::signal::windows` allows receiving console control events like `CTRL_C_EVENT`, `CTRL_BREAK_EVENT`, and shutdown events via `SetConsoleCtrlHandler`.

Like the Unix implementation, notifications are coalesced. If multiple events occur in rapid succession, the listener may only receive a single notification.

### Available Listeners

Functions are available to create listeners for specific console control events.

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard">Creates a listener for `CTRL_C_EVENT` notifications.</x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:keyboard">Creates a listener for `CTRL_BREAK_EVENT` notifications.</x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-square">Creates a listener for `CTRL_CLOSE_EVENT` notifications, sent when the console window is closed.</x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:power-off">Creates a listener for `CTRL_SHUTDOWN_EVENT` notifications, sent when the system is shutting down.</x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out">Creates a listener for `CTRL_LOGOFF_EVENT` notifications, sent when the user logs off.</x-card>
</x-cards>

### Usage

Each function returns a corresponding struct (e.g., `ctrl_c()` returns `CtrlC`). All of these structs provide the same `recv()` and `poll_recv()` methods for consuming events.

Here is an example of handling `CTRL-BREAK` events:

```rust,no_run
# #[cfg(windows)] {
use tokio::signal::windows::ctrl_break;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a listener for CTRL-BREAK events.
    let mut stream = ctrl_break()?;

    // Print whenever a CTRL-BREAK event is received.
    loop {
        stream.recv().await;
        println!("got signal CTRL-BREAK");
    }
}
# }
```

### Listener Methods

All Windows signal listener structs (`CtrlC`, `CtrlBreak`, etc.) have the following methods:

| Method | Description |
|---|---|
| `recv(&mut self)` | Asynchronously waits for the next notification. Returns `None` if the listener is closed. |
| `poll_recv(&mut self, cx: &mut Context<'_'>)` | Polls for the next notification in a non-async context. |
