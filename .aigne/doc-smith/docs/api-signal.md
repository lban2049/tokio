# Signals

This module provides support for asynchronous signal handling. Signals are a form of inter-process communication, and handling them correctly can be complex. This implementation aims to follow best practices, but you should evaluate its suitability for your specific application needs.

Note that signal handling is platform-specific. Tokio provides different APIs for Unix and Windows to accommodate their distinct models.

A common use case is to gracefully shut down a server on `CTRL+C`.

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

## Unix Signals

On Unix platforms, you can listen for arbitrary signals. The primary types for this are `Signal`, which represents the listener, and `SignalKind`, which specifies the signal.

### `signal()`

Creates a new listener for a specified signal kind. This returns a `Signal` instance that can be used to receive notifications.

**Important Caveats:**

*   **Signal Coalescing**: If multiple signals are received before the listener is polled, they will be coalesced into a single notification. After polling, the next signal is guaranteed to generate a new notification.
*   **Persistent Handler**: The first time a listener is created for a particular signal, an OS-level signal handler is installed for the entire duration of the process. This behavior is not reset even if the `Signal` instance is dropped. For example, listening for `SIGINT` will prevent the default behavior of process termination for subsequent `SIGINT` signals.

**Example: Waiting for `SIGHUP`**

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

### `Signal` Struct

A listener for receiving a specific OS signal. It provides two main methods for receiving notifications.

*   `recv()`: An `async` method that resolves when the next signal is received.
*   `poll_recv()`: A method for use in manual `Future` implementations that polls for the next signal.

### `SignalKind` Struct

Represents the specific kind of signal to listen for. It provides constructor methods for common signals.

<x-cards data-columns="3">
  <x-card data-title="alarm()" data-icon="lucide:alarm-clock">SIGALRM: Sent when a real-time timer has expired.</x-card>
  <x-card data-title="child()" data-icon="lucide:baby">SIGCHLD: Sent when the status of a child process has changed.</x-card>
  <x-card data-title="hangup()" data-icon="lucide:phone-off">SIGHUP: Sent when the terminal is disconnected.</x-card>
  <x-card data-title="interrupt()" data-icon="lucide:keyboard">SIGINT: Sent to interrupt a program (e.g., Ctrl+C).</x-card>
  <x-card data-title="io()" data-icon="lucide:arrow-left-right">SIGIO: Sent when I/O operations are possible on a file descriptor.</x-card>
  <x-card data-title="pipe()" data-icon="lucide:pipeline">SIGPIPE: Sent when writing to a pipe with no reader.</x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">SIGQUIT: Sent to terminate the process and dump core.</x-card>
  <x-card data-title="terminate()" data-icon="lucide:shield-x">SIGTERM: Sent to request a graceful process shutdown.</x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user-cog">SIGUSR1: A user-defined signal.</x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:user-cog">SIGUSR2: A user-defined signal.</x-card>
  <x-card data-title="window_change()" data-icon="lucide:rectangle-horizontal">SIGWINCH: Sent when the terminal window is resized.</x-card>
  <x-card data-title="from_raw()" data-icon="lucide:hash">Allows listening for any valid OS signal by its raw integer value.</x-card>
</x-cards>

---

## Windows Signals

On Windows, signal handling is based on console control events. Tokio provides separate functions to create listeners for each specific event type.

Like the Unix implementation, notifications are coalesced. If multiple events of the same type occur before the listener is polled, they will be delivered as a single notification.

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard">Creates a listener that receives notifications when the user presses `Ctrl+C`.</x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:keyboard">Creates a listener for `Ctrl+Break` events.</x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-square">Creates a listener for `Ctrl+Close` events, sent when the console window is closed.</x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:power-off">Creates a listener for `Ctrl+Shutdown` events, sent when the system is shutting down.</x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out">Creates a listener for `Ctrl+Logoff` events, sent when the user logs off.</x-card>
</x-cards>

Each of these functions returns a dedicated struct (e.g., `CtrlC`, `CtrlBreak`) with `recv()` and `poll_recv()` methods, which behave identically to their Unix counterparts.

**Example: Handling `Ctrl+Break`**

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

Now that you've learned about signal handling, you might be interested in managing child processes asynchronously. See the [Processes](./api-process.md) documentation for more details.