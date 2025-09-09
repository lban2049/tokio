# Processes

This module provides an implementation of asynchronous process management for Tokio. It offers a `Command` struct that mirrors the interface of the standard library's [`std::process::Command`](https://doc.rust-lang.org/std/process/Command.html), but provides asynchronous versions of functions that create and manage child processes.

These functions (`spawn`, `status`, `output`, and their variants) return future-aware types that integrate seamlessly with the Tokio runtime. This asynchronous support is handled through signal handling on Unix and specialized system APIs on Windows.

## Quick Start: Spawning a Process

The most common use case is to spawn a command, execute it, and wait for it to complete. The interface is designed to be very similar to the standard library.

```rust Spawning a simple command icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // The usage is similar to the standard library's `Command` type
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

## The `Command` Struct

The `Command` struct is the primary builder for configuring and executing new child processes asynchronously.

### Creating and Configuring a Command

You start by creating a new `Command` and then use its builder methods to configure the process before execution.

| Method | Description |
| --- | --- |
| `new(program)` | Constructs a new `Command` to launch the specified program. |
| `arg(arg)` | Adds a single argument to pass to the program. |
| `args(args)` | Adds multiple arguments to pass to the program. |
| `env(key, val)` | Sets an environment variable for the child process. |
| `envs(vars)` | Adds or updates multiple environment variables. |
| `env_remove(key)` | Removes an environment variable. |
| `env_clear()` | Clears all environment variables for the child process. |
| `current_dir(dir)` | Sets the working directory for the child process. |
| `stdin(cfg)` | Configures the standard input (stdin) handle. |
| `stdout(cfg)` | Configures the standard output (stdout) handle. |
| `stderr(cfg)` | Configures the standard error (stderr) handle. |

### Executing a Command

Once configured, you can execute the command in several ways:

*   **`spawn()`**: Executes the command as a child process, returning a handle to it immediately. This is the most flexible method, giving you a `Child` object to manage.

*   **`status()`**: Executes the command and waits for it to finish, returning only its `ExitStatus`. This is a convenient shorthand for when you don't need to interact with the child's I/O.

*   **`output()`**: Executes the command, waits for it to complete, and collects all of its standard output and standard error. This is useful for capturing the full output of a command.

```rust Capturing output icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Use `output()` which returns a future that resolves to the process's output
    let output = Command::new("echo")
        .arg("hello")
        .arg("world")
        .output()
        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");
    Ok(())
}
```

## The `Child` Struct

When you `spawn` a process, you get a `Child` struct. This struct represents the running child process and provides methods to interact with it.

### Waiting for a Child

The most fundamental operation is waiting for the process to exit. The `Child` struct itself can be `.await`ed, which is equivalent to calling the `wait()` method.

*   **`wait()`**: Asynchronously waits for the child to exit completely, returning its `ExitStatus`.
*   **`try_wait()`**: Attempts to collect the exit status without blocking. It returns `Ok(Some(status))` if the child has exited, `Ok(None)` if it's still running, and `Err` on error.
*   **`wait_with_output()`**: A future that waits for the child to exit and collects all remaining output on its stdout and stderr handles.

### Terminating a Child

You can forcefully terminate a child process if needed.

*   **`start_kill()`**: Sends a kill signal to the child but does not wait for it to exit.
*   **`kill()`**: Sends a kill signal and then asynchronously waits for the process to be fully terminated.

```rust Killing a child process icon=logos:rust
use tokio::process::Command;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut child = Command::new("sleep").arg("5").spawn()?;

    // Give it a moment to start
    sleep(Duration::from_millis(100)).await;

    println!("Killing the child process");
    child.kill().await?;

    let status = child.wait().await?;
    println!("Child exited with: {}", status);

    Ok(())
}
```

## Working with I/O

Tokio's process management shines when you need to interact with the child's standard input, output, and error streams. To do this, you must configure the corresponding handle to be `Stdio::piped()`.

### Reading from a Child's `stdout`

Once a child is spawned with a piped stdout, you can get an asynchronous reader from `child.stdout`.

```rust Reading stdout line by line icon=logos:rust
use tokio::io::{AsyncBufReadExt, BufReader};
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("cat");

    // Pipe the command's standard output back to us.
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn()
        .expect("failed to spawn command");

    let stdout = child.stdout.take()
        .expect("child did not have a handle to stdout");

    let mut reader = BufReader::new(stdout).lines();

    // Ensure the child process is spawned in the runtime to make progress.
    tokio::spawn(async move {
        let status = child.wait().await
            .expect("child process encountered an error");
        println!("child status was: {}", status);
    });

    while let Some(line) = reader.next_line().await? {
        println!("Line: {}", line);
    }

    Ok(())
}
```

### Writing to a Child's `stdin`

Similarly, you can write to a child's standard input asynchronously if it's configured as a pipe.

```rust Writing to stdin and reading from stdout icon=logos:rust
use tokio::io::AsyncWriteExt;
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("sort");

    // Pipe both stdout and stdin
    cmd.stdout(Stdio::piped());
    cmd.stdin(Stdio::piped());

    let mut child = cmd.spawn().expect("failed to spawn command");

    let mut stdin = child.stdin.take().expect("child did not have a handle to stdin");

    // Write data to the child process's stdin in a separate task
    tokio::spawn(async move {
        stdin.write_all(b"dog\nbird\nfrog\ncat\nfish\n").await.unwrap();
        // Dropping stdin signals EOF to the child
    });

    let output = child.wait_with_output().await?;

    // The output should be sorted
    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```

### Piping Between Processes

You can also pipe the output of one command directly into the input of another.

```rust Piping between two commands icon=logos:rust
use tokio::join;
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut echo = Command::new("echo")
        .arg("hello world!")
        .stdout(Stdio::piped())
        .spawn()
        .expect("failed to spawn echo");

    // Take the stdout of the first command
    let echo_stdout = echo.stdout.take().unwrap();

    // Create the stdin for the second command from the first's stdout
    let tr_stdin: Stdio = echo_stdout.try_into().expect("failed to convert to Stdio");

    let tr = Command::new("tr")
        .arg("a-z")
        .arg("A-Z")
        .stdin(tr_stdin)
        .stdout(Stdio::piped())
        .spawn()
        .expect("failed to spawn tr");

    // Wait for both processes to complete
    let (echo_result, tr_output) = join!(echo.wait(), tr.wait_with_output());

    assert!(echo_result.unwrap().success());

    let tr_output = tr_output.expect("failed to await tr");
    assert!(tr_output.status.success());
    assert_eq!(tr_output.stdout, b"HELLO WORLD!\n");

    Ok(())
}
```

## Caveats

### Dropping and Cancellation

Unlike most futures, dropping a `Child` handle does **not** automatically terminate the process. The child process will continue to run in the background. If you want the child to be killed when the handle is dropped, you must configure this behavior using the `kill_on_drop` method on the `Command` builder.

```rust
let mut child = Command::new("sleep")
    .arg("1000")
    .kill_on_drop(true) // Kill the process if the Child handle is dropped
    .spawn()
    .unwrap();

// When `child` goes out of scope here, a kill signal will be sent.
```

### Zombie Processes on Unix

On Unix-like systems, a process that has exited but has not yet been waited on by its parent becomes a "zombie". These processes consume system resources. The Tokio runtime will attempt to reap zombie processes it has spawned on a best-effort basis. However, for guaranteed cleanup, it is highly recommended to always explicitly wait for the `Child` to complete using `.await` or `wait()`.