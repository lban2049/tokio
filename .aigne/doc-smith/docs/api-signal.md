# Signals

This module provides support for asynchronous signal handling. Signal handling is a complex topic and should be approached with care. This implementation aims to follow best practices, but you should evaluate its suitability for your specific needs.

There are fundamental limitations documented on the OS-specific structures.

## Cross-Platform: Handling Ctrl-C

Tokio provides a convenient, cross-platform function to listen for the `ctrl-c` signal (or `SIGINT` on Unix). This is often the easiest way to handle graceful shutdown in an application.

```rust,no_run
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Waiting for ctrl-c...");
    signal::ctrl_c().await?;
    println!("ctrl-c received!");
    Ok(())
}
```

## Platform-Specific Signals

For more advanced or platform-specific signal handling, Tokio provides modules for both Unix and Windows.

### Unix Signals

The `tokio::signal::unix` module provides the primary `Signal` type for receiving notifications of various Unix signals.

#### `signal()`

To create a listener for a specific signal, use the `signal` function with a given `SignalKind`.

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

#### The `Signal` Struct

The `signal` function returns a `Signal` struct, which acts as a listener for a specific signal type. It can be used to asynchronously wait for signals.

**Methods:**

*   `recv(&mut self) -> Option<()>`: Asynchronously waits for the next signal notification.
*   `poll_recv(&mut self, cx: &mut Context<'_>) -> Poll<Option<()>>`: Polls for the next signal notification in a non-async context.

#### Important Caveats

When using Unix signal handlers, keep these points in mind:

1.  **Handler Lifetime**: The first time a `Signal` is created for a particular signal kind, an OS signal handler is installed for the *entire duration of the process*. It replaces the default platform behavior. Even if the `Signal` instance is dropped, the handler is **not** reset. For example, after listening for `SIGINT` once, the process will no longer terminate by default on `SIGINT`.

2.  **Signal Coalescing**: Signals can be coalesced. If multiple signals are received before the listener is polled, they may result in a single event. The listener guarantees that each item corresponds to *at least one* signal.

#### `SignalKind`

This struct represents the specific kind of signal to listen for. Common signals are available as constants.

| Method | Signal | Description |
|---|---|---|
| `alarm()` | `SIGALRM` | Sent when a real-time timer has expired. |
| `child()` | `SIGCHLD` | Sent when the status of a child process has changed. |
| `hangup()` | `SIGHUP` | Sent when the controlling terminal is disconnected. |
| `interrupt()` | `SIGINT` | Sent to interrupt a program (e.g., ctrl-c). |
| `io()` | `SIGIO` / `SIGPOLL` | Sent when I/O operations are possible on a file descriptor. |
| `pipe()` | `SIGPIPE` | Sent when writing to a pipe with no reader. |
| `quit()` | `SIGQUIT` | Sent to request a process shutdown and core dump. |
| `terminate()` | `SIGTERM` | Sent to request a graceful process shutdown. |
| `user_defined1()` | `SIGUSR1` | User-defined signal 1. |
| `user_defined2()` | `SIGUSR2` | User-defined signal 2. |
| `window_change()` | `SIGWINCH` | Sent when the terminal window is resized. |

For platform-specific or less common signals, you can create a `SignalKind` from a raw integer value:

```rust,no_run
# use tokio::signal::unix::SignalKind;
# let signum = -1;
// let signum = libc::OS_SPECIFIC_SIGNAL;
let kind = SignalKind::from_raw(signum);
```

### Windows Events

The `tokio::signal::windows` module allows receiving console control events like "ctrl-c" and "ctrl-break" via the `SetConsoleCtrlHandler` function.

Like their Unix counterparts, these notifications are coalesced if not processed quickly enough.

#### `ctrl_c`

Creates a listener for "ctrl-c" events.

```rust,no_run
use tokio::signal::windows::ctrl_c;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_c()?;
    stream.recv().await;
    println!("got ctrl-c");
    Ok(())
}
```

#### `ctrl_break`

Creates a listener for "ctrl-break" events.

```rust,no_run
use tokio::signal::windows::ctrl_break;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_break()?;
    stream.recv().await;
    println!("got ctrl-break");
    Ok(())
}
```

#### `ctrl_close`

Creates a listener for console close events.

```rust,no_run
use tokio::signal::windows::ctrl_close;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_close()?;
    stream.recv().await;
    println!("got ctrl-close");
    Ok(())
}
```

#### `ctrl_shutdown`

Creates a listener for system shutdown events.

```rust,no_run
use tokio::signal::windows::ctrl_shutdown;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_shutdown()?;
    stream.recv().await;
    println!("got ctrl-shutdown");
    Ok(())
}
```

#### `ctrl_logoff`

Creates a listener for user logoff events.

```rust,no_run
use tokio::signal::windows::ctrl_logoff;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = ctrl_logoff()?;
    stream.recv().await;
    println!("got ctrl-logoff");
    Ok(())
}
```

Each of these functions returns a struct (e.g., `CtrlC`, `CtrlBreak`) with `recv()` and `poll_recv()` methods, which behave similarly to the Unix `Signal` struct.