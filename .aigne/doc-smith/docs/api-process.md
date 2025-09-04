# Processes

This module provides an implementation of asynchronous process management for Tokio. It features a `Command` struct that mirrors the interface of the standard library's `std::process::Command`, but provides asynchronous versions of functions that create processes.

These functions (`spawn`, `status`, `output`, and their variants) return future-aware types that integrate with the Tokio runtime. This asynchronous support is handled through signal handling on Unix and system APIs on Windows.

## Quick Start

### Spawning a process and waiting for it to complete

The basic usage is very similar to the standard library. The main difference is that operations like waiting for the child to exit are `async`.

```rust,no_run
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // The usage is similar to the standard library's `Command` type
    let mut child = Command::new("echo")
        .arg("hello")
        .arg("world")
        .spawn()
        .expect("failed to spawn");

    // Await until the command completes
    let status = child.wait().await?;
    println!("the command exited with: {}", status);
    Ok(())
}
```

### Capturing output

To spawn a process and capture all of its output, you can use the `output` method, which returns a future resolving to the process's `Output`.

```rust,no_run
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Use `output` which returns a future instead of
    // immediately returning the `Child`.
    let output = Command::new("echo").arg("hello").arg("world")
                        .output()
                        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");
    Ok(())
}
```

## Key Structs

| Struct | Description |
|---|---|
| `Command` | A builder for configuring and spawning new child processes asynchronously. |
| `Child` | Represents a running child process, providing access to its I/O streams and allowing you to wait for it to complete. |
| `ChildStdin` | A handle to a child process's standard input (stdin), implementing `AsyncWrite`. |
| `ChildStdout` | A handle to a child process's standard output (stdout), implementing `AsyncRead`. |
| `ChildStderr` | A handle to a child process's standard error (stderr), implementing `AsyncRead`. |

## Advanced Usage

### Reading from Stdout

You can pipe the child's standard output and read from it asynchronously, for example, line-by-line.

```rust,no_run
use tokio::io::{BufReader, AsyncBufReadExt};
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("cat");

    // Specify that we want the command's standard output piped back to us.
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn()
        .expect("failed to spawn command");

    let stdout = child.stdout.take()
        .expect("child did not have a handle to stdout");

    let mut reader = BufReader::new(stdout).lines();

    // Ensure the child process is spawned in the runtime so it can
    // make progress on its own while we await for any output.
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

### Writing to Stdin

Similarly, you can pipe to a child's standard input and write to it asynchronously.

```rust,no_run
use tokio::io::AsyncWriteExt;
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("sort");

    // Pipe both stdout and stdin.
    cmd.stdout(Stdio::piped());
    cmd.stdin(Stdio::piped());

    let mut child = cmd.spawn().expect("failed to spawn command");

    let mut stdin = child.stdin.take()
        .expect("child did not have a handle to stdin");

    // Write data to the child process in a separate task to avoid deadlocks.
    tokio::spawn(async move {
        stdin.write_all(b"dog\nbird\nfrog\ncat\nfish\n").await.unwrap();
        // Dropping stdin signals EOF.
    });

    let output = child.wait_with_output().await?;

    // Results should come back in sorted order
    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```
Note: The behavior of some programs, like `sort`, is to buffer all input before writing any output. In general, it is recommended to write to the child in a separate task from awaiting its exit or output to avoid deadlocks.

### Piping Between Processes

You can pipe the output of one command into the input of another.

```rust,no_run
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

    let tr_stdin: Stdio = echo
        .stdout
        .take()
        .unwrap()
        .try_into()
        .expect("failed to convert to Stdio");

    let tr = Command::new("tr")
        .arg("a-z")
        .arg("A-Z")
        .stdin(tr_stdin)
        .stdout(Stdio::piped())
        .spawn()
        .expect("failed to spawn tr");

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

A spawned process will, by default, continue to execute even after its `Child` handle has been dropped. This behavior is similar to the standard library and differs from the common futures paradigm where dropping implies cancellation.

To change this, you can use the `Command::kill_on_drop(true)` method. This will cause the child process to be killed if the `Child` handle is dropped before the process has exited.

### Unix Processes and Zombies

On Unix platforms, a parent process must "reap" its child after it has exited to release all OS resources. A child process that has exited but has not been reaped is a "zombie" process. An accumulation of zombie processes can prevent new processes from being spawned.

The Tokio runtime attempts to reap any process it spawns on a best-effort basis. However, for stricter cleanup guarantees, it is recommended to avoid dropping a `Child` handle and instead explicitly await its completion with `.wait().await`.
