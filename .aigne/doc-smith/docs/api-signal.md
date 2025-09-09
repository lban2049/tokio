# Signals

Asynchronous signal handling for Tokio. This module provides tools to receive and handle OS signals in an asynchronous manner.

Note that signal handling is a complex topic and should be used with care. This implementation follows best practices, but you should evaluate its suitability for your application's specific needs. There are also fundamental limitations documented on the OS-specific structures.

### Cross-Platform Ctrl-C

Tokio provides a convenient, cross-platform future that resolves when the process receives a `Ctrl-C` signal.

```rust Handling Ctrl-C icon=logos:rust
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

## Unix

On Unix platforms, you can create listeners for arbitrary signals. These listeners are represented as streams that yield a unit type `()` for each signal received.

### `signal()`

The `signal` function creates a new `Signal` stream that listens for a specific `SignalKind`.

```rust Waiting for SIGHUP icon=logos:rust
# #[cfg(unix)] {
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a stream of hangup signals.
    let mut stream = signal(SignalKind::hangup())?;

    // Print whenever a HUP signal is received.
    println!("Waiting for SIGHUP...");
    loop {
        stream.recv().await;
        println!("Received SIGHUP");
    }
}
# }
```

### Caveats

There are some important limitations to keep in mind when using Unix signals with Tokio:

*   **Signal Coalescing**: If multiple signals of the same kind are received before the stream is polled, they may be coalesced into a single event. The stream guarantees that at least one signal was received for each item yielded.
*   **Permanent Handler**: The first time a listener is created for a specific signal, Tokio installs a global OS signal handler for that signal. This handler replaces the default system behavior for the entire lifetime of the process and is not uninstalled, even if the `Signal` stream is dropped.

### `SignalKind`

The `SignalKind` struct represents a specific Unix signal. It provides constructor methods for common signals.

<x-cards data-columns="3">
  <x-card data-title="alarm()" data-icon="lucide:alarm-clock">Represents the `SIGALRM` signal, typically sent when a real-time timer expires.</x-card>
  <x-card data-title="child()" data-icon="lucide:baby">Represents the `SIGCHLD` signal, sent when the status of a child process has changed.</x-card>
  <x-card data-title="hangup()" data-icon="lucide:phone-off">Represents the `SIGHUP` signal, sent when a terminal is disconnected.</x-card>
  <x-card data-title="interrupt()" data-icon="lucide:hand">Represents the `SIGINT` signal, sent to interrupt a program (e.g., Ctrl-C).</x-card>
  <x-card data-title="io()" data-icon="lucide:arrow-left-right">Represents the `SIGIO` signal, sent when I/O operations are possible on a file descriptor.</x-card>
  <x-card data-title="pipe()" data-icon="lucide:pipeline">Represents the `SIGPIPE` signal, sent when writing to a pipe with no readers.</x-card>
  <x-card data-title="quit()" data-icon="lucide:log-out">Represents the `SIGQUIT` signal, sent to request a process shutdown and core dump.</x-card>
  <x-card data-title="terminate()" data-icon="lucide:power-off">Represents the `SIGTERM` signal, sent to request a graceful process shutdown.</x-card>
  <x-card data-title="user_defined1()" data-icon="lucide:user">Represents the `SIGUSR1` signal, for user-defined purposes.</x-card>
  <x-card data-title="user_defined2()" data-icon="lucide:user-cog">Represents the `SIGUSR2` signal, for user-defined purposes.</x-card>
  <x-card data-title="window_change()" data-icon="lucide:maximize">Represents the `SIGWINCH` signal, sent when the terminal window is resized.</x-card>
  <x-card data-title="from_raw()" data-icon="lucide:hash">Allows listening for any valid OS signal by providing its raw integer value.</x-card>
</x-cards>

---

## Windows

On Windows, signal handling is based on receiving console control events. Tokio provides separate functions to create listeners for each type of event.

### Console Event Listeners

Each function returns a listener struct (e.g., `CtrlC`, `CtrlBreak`) that can be used to asynchronously wait for the corresponding event.

<x-cards data-columns="2">
  <x-card data-title="ctrl_c()" data-icon="lucide:keyboard" data-href="#">
    Creates a listener that receives "ctrl-c" notifications.
  </x-card>
  <x-card data-title="ctrl_break()" data-icon="lucide:keyboard" data-href="#">
    Creates a listener that receives "ctrl-break" notifications.
  </x-card>
  <x-card data-title="ctrl_close()" data-icon="lucide:x-square" data-href="#">
    Creates a listener that receives "ctrl-close" notifications when the console is closed.
  </x-card>
  <x-card data-title="ctrl_logoff()" data-icon="lucide:log-out" data-href="#">
    Creates a listener that receives "ctrl-logoff" notifications when the user logs off.
  </x-card>
  <x-card data-title="ctrl_shutdown()" data-icon="lucide:power" data-href="#">
    Creates a listener that receives "ctrl-shutdown" notifications when the system is shutting down.
  </x-card>
</x-cards>

Like Unix signals, these notifications are coalesced. If multiple events of the same type occur rapidly, the listener may only yield a single notification.

### Example

The following example demonstrates how to listen for `CTRL-BREAK` events.

```rust Handling CTRL-BREAK on Windows icon=logos:rust
use tokio::signal::windows::ctrl_break;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = ctrl_break()?;

    println!("Waiting for CTRL-BREAK...");
    stream.recv().await;
    println!("CTRL-BREAK received!");

    Ok(())
}
```