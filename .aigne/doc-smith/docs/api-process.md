# Processes

This module provides an implementation of asynchronous process management for Tokio. It offers a `Command` struct that closely mirrors the API of `std::process::Command` from the standard library, but with asynchronous methods for creating and managing processes.

These asynchronous functions, such as `spawn`, `status`, and `output`, return future-aware types that integrate seamlessly with the Tokio runtime. This allows you to manage child processes without blocking threads, handling them as you would any other asynchronous task.

## Quick Examples

### Spawning a process and waiting for it to complete

The most basic use case is to run a command and wait for its completion status.

```rust,no_run
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut child = Command::new("echo")
        .arg("hello")
        .arg("world")
        .spawn()
        .expect("failed to spawn");

    // Await the child process to complete
    let status = child.wait().await?;

    println!("the command exited with: {}", status);
    Ok(())
}
```

### Spawning a process and capturing its output

For many applications, you'll need to capture the output (stdout and stderr) of the child process.

```rust,no_run
use tokio::process::Command;
use std::process::Output;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let output: Output = Command::new("echo")
        .arg("hello")
        .arg("world")
        .output()
        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");

    println!("Command executed successfully");
    Ok(())
}
```

## The `Command` Struct

The `Command` struct is the primary builder for configuring and spawning child processes. It provides a fluent interface for setting arguments, environment variables, the working directory, and I/O handles before execution.

### Configuration

You can configure the process in several ways before spawning it:

| Method | Description |
|---|---|
| `new(program)` | Constructs a new `Command` to launch the specified program. |
| `arg(arg)` | Adds a single argument to pass to the program. |
| `args(args)` | Adds multiple arguments from an iterator. |
| `env(key, val)` | Sets an environment variable for the child process. |
| `envs(vars)` | Adds or updates multiple environment variables. |
| `env_remove(key)` | Removes an environment variable. |
| `env_clear()` | Clears all inherited environment variables. |
| `current_dir(dir)` | Sets the working directory for the child process. |
| `stdin(cfg)` | Configures the standard input (stdin) handle. |
| `stdout(cfg)` | Configures the standard output (stdout) handle. |
| `stderr(cfg)` | Configures the standard error (stderr) handle. |
| `kill_on_drop(bool)` | Kills the child process if the `Child` handle is dropped. |

### Execution

Once configured, you can execute the command in one of three main ways:

*   **`spawn()`**: Executes the command and returns a `Child` handle, allowing for detailed interaction with the running process.
*   **`status()`**: A convenience method that runs the command and waits for its `ExitStatus`.
*   **`output()`**: A convenience method that runs the command, waits for it to finish, and collects all of its standard output and standard error.

## The `Child` Struct

A `Child` handle is returned by the `spawn` method and represents a running child process. It provides methods to wait for the process to exit, kill it, and access its I/O streams.

### Managing the Process

| Method | Description |
|---|---|
| `wait()` | Asynchronously waits for the child to exit completely, returning its `ExitStatus`. |
| `kill()` | Forcefully terminates the child process and waits for it to be reaped. |
| `start_kill()` | Sends a kill signal but does not wait for the process to exit. |
| `try_wait()` | Checks if the child has exited without blocking. |
| `id()` | Returns the OS-assigned process identifier. |

### Interacting with I/O

If the `Command` was configured with piped I/O (`Stdio::piped()`), the `Child` handle will contain handles for `stdin`, `stdout`, and `stderr`. These implement `AsyncWrite` and `AsyncRead` respectively.

Here is an example of writing data to a child's stdin and reading from its stdout:

```rust,no_run
use tokio::io::{AsyncWriteExt, AsyncReadExt};
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("sort");

    cmd.stdin(Stdio::piped());
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn().expect("failed to spawn command");

    let mut stdin = child.stdin.take().expect("child did not have a handle to stdin");

    // Write data to the child process's stdin in a separate task.
    tokio::spawn(async move {
        stdin.write_all(b"dog\nbird\nfrog\ncat\nfish\n").await.unwrap();
    });

    let output = child.wait_with_output().await?;

    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```

## Caveats

There are a few important behaviors to be aware of when working with Tokio processes.

### Dropping and Cancellation

Unlike most futures, dropping a `Child` handle does **not** automatically terminate the process. The process will continue to run in the background. If you want the process to be killed when the handle is dropped, you must configure the command with `.kill_on_drop(true)` before spawning it.

### Zombie Processes on Unix

On Unix-like systems, a process that has exited but has not been waited on by its parent becomes a "zombie". These processes consume system resources. The Tokio runtime attempts to reap spawned processes on a best-effort basis, but for guaranteed cleanup, it is recommended to always explicitly wait for the `Child` to complete using `.wait().await` or a similar method.
