# Child Processes

Tokio provides tools for managing child processes asynchronously. This module, `tokio::process`, offers a `Command` struct that mirrors the interface of the standard library's `std::process::Command` but with asynchronous methods for spawning and managing processes.

This allows you to execute external commands, capture their output, and interact with their standard I/O streams without blocking the Tokio runtime.

## The `Command` Struct

The primary entry point for process management is the `tokio::process::Command` struct. You use it to configure a process before spawning it, setting arguments, environment variables, and I/O pipes.

### Spawning a Simple Process

Let's start with a basic example. We'll spawn an `echo` command and wait for it to complete, checking its exit status.

```rust Spawning a Command icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // The usage is similar to the standard library's `Command` type
    let mut child = Command::new("echo")
        .arg("hello")
        .arg("world")
        .spawn()
        .expect("failed to spawn
        ");

    // Await the child process to complete
    let status = child.wait().await?;

    println!("the command exited with: {}", status);
    Ok(())
}
```

This code creates a new `Command`, configures it to run `echo hello world`, spawns it, and then asynchronously waits for the process to exit.

### Capturing Output

Often, you need to capture the output of a command. The `output()` method spawns the process and returns a future that resolves to an `Output` struct containing `stdout`, `stderr`, and the exit `status`.

```rust Capturing Command Output icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Use `output()` which returns a future resolving to the command's output
    let output = Command::new("echo").arg("hello").arg("world")
                        .output()
                        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");

    println!("Command executed successfully");
    Ok(())
}
```

## Interacting with I/O (Piping)

For more complex interactions, you can pipe the standard input, output, and error streams of the child process. This is done by configuring the `stdin`, `stdout`, and `stderr` handles to use `std::process::Stdio::piped()`.

### Reading from `stdout`

You can capture and read the standard output of a child process as it's generated.

```rust Reading from stdout icon=logos:rust
use tokio::io::{BufReader, AsyncBufReadExt};
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("cat");

    // Pipe the command's standard output to us
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn()
        .expect("failed to spawn command");

    let stdout = child.stdout.take()
        .expect("child did not have a handle to stdout");

    let mut reader = BufReader::new(stdout).lines();

    // Spawn a task to wait for the child process to complete.
    // This prevents deadlocks if the child produces a lot of output.
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
In this example, we take ownership of `child.stdout` and wrap it in a `BufReader` to read its output line by line. The `child.wait()` future is moved to a separate task to ensure the child process can run to completion while we process its output.

### Writing to `stdin`

Similarly, you can write data to the standard input of a child process.

```rust Writing to stdin icon=logos:rust
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

    let animals = b"dog\nbird\nfrog\ncat\nfish";

    let mut stdin = child.stdin.take().expect("child did not have a handle to stdin");

    // Write our animals to the child process's stdin
    stdin.write_all(animals).await.expect("could not write to stdin");

    // Drop the stdin handle to signal EOF to the child process.
    drop(stdin);

    let output = child.wait_with_output().await?;

    // Expect the results to be sorted
    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```
Here, we write a list of animals to the `sort` command. It's crucial to `drop(stdin)` after writing. This closes the pipe, signaling End-Of-File (EOF) to the child process, which then knows to finish its processing and exit.

### Piping Between Commands

You can pipe the output of one command directly into the input of another.

```rust Piping Between Commands icon=logos:rust
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

    // Take the stdout of the echo process
    let echo_stdout = echo.stdout.take().unwrap();

    // Convert the ChildStdout into a Stdio handle for the next command
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

## Managing the `Child` Handle

When you spawn a process, you get a `Child` handle. This handle allows you to manage the running process.

- **`wait()`**: Asynchronously waits for the child to exit.
- **`id()`**: Gets the OS-assigned process ID.
- **`start_kill()`**: Initiates a kill signal (`SIGKILL` on Unix) but doesn't wait for the process to exit.
- **`kill()`**: A convenience method that sends a kill signal and then waits for the process to exit.

Here's an example of how you might kill a child process if a certain event occurs:

```rust Killing a Child Process icon=logos:rust
use tokio::process::Command;
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<()>();
    let mut child = Command::new("sleep").arg("5").spawn().unwrap();

    // Send a signal to kill the process
    tokio::spawn(async move { tx.send(()) });

    tokio::select! {
        _ = child.wait() => {
            println!("Child completed on its own.");
        }
        _ = rx => {
            println!("Kill signal received. Terminating child.");
            child.kill().await.expect("kill failed");
        }
    }
}
```

## Important Caveats

### Dropping and Cancellation

By default, dropping a `Child` handle does **not** terminate the process. The process will continue to run in the background. This mirrors the behavior of the standard library.

If you want the child process to be killed when the `Child` handle is dropped, you must configure it on the `Command` before spawning:

```rust
let mut child = Command::new("long-running-process")
    .kill_on_drop(true)
    .spawn()?;
```

### Zombie Processes on Unix

On Unix-like systems, a process that has exited but has not been waited on by its parent becomes a "zombie" process. These zombies still consume system resources (like a process ID).

Tokio's runtime will, on a best-effort basis, automatically reap zombie processes it has spawned. However, for applications that require strict guarantees about resource cleanup, it is highly recommended to explicitly wait for every child process to complete using `child.wait().await` rather than relying on the background reaping mechanism.

## Next Steps

Managing child processes often goes hand-in-hand with handling system signals. To learn how to respond to events like `SIGINT` (Ctrl-C), continue to the next section.

<x-card data-title="OS Signals" data-href="/io/signals" data-icon="lucide:radio-tower">
  Learn how to handle Unix and Windows OS signals asynchronously within a Tokio application.
</x-card>
