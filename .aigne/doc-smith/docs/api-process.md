# Processes

This module provides an implementation of asynchronous process management for Tokio. It features a `Command` struct that closely mirrors the API of `std::process::Command` but offers asynchronous versions of its process-creation functions.

Key functions like `spawn`, `status`, and `output` are asynchronous, returning future-aware types that integrate seamlessly with the Tokio runtime. This allows you to manage child processes without blocking threads, using signal handling on Unix and system APIs on Windows.

## Quick Start: Spawning a Process

The most straightforward way to use this module is to spawn a command and wait for it to complete. The interface is designed to be familiar to anyone who has used the standard library's `Command` type.

```rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // The usage is similar to the standard library's `Command` type
    let mut child = Command::new("echo")
        .arg("hello")
        .arg("world")
        .spawn()
        .expect("failed to spawn");

    // Await the command's completion
    let status = child.wait().await?;

    println!("the command exited with: {}", status);

    Ok(())
}
```

## Core Components

The process management API revolves around a few key types.

<x-cards>
  <x-card data-title="Command" data-icon="lucide:terminal">
    A command builder used to configure and spawn new child processes. You use it to set the program, arguments, environment variables, and I/O handles.
  </x-card>
  <x-card data-title="Child" data-icon="lucide:cpu">
    A handle to a running child process. It allows you to wait for the process to exit and manage its I/O streams.
  </x-card>
  <x-card data-title="ChildStdin" data-icon="lucide:arrow-right-from-line">
    A handle to the standard input (stdin) of a child process, implementing `AsyncWrite`.
  </x-card>
  <x-card data-title="ChildStdout / ChildStderr" data-icon="lucide:arrow-left-from-line">
    Handles to the standard output (stdout) and standard error (stderr) of a child process, implementing `AsyncRead`.
  </x-card>
</x-cards>

## The `Command` Struct

The `Command` struct is the entry point for creating and configuring child processes. It provides a builder pattern for setting up the process environment before spawning it.

### Creating a Command

You can create a new `Command` by specifying the program to be executed.

```rust
use tokio::process::Command;

let mut command = Command::new("sh");
```

### Configuring Arguments and Environment

Methods are available to configure arguments, environment variables, and the working directory.

| Method | Description |
|---|---|
| `.arg(arg)` | Adds a single argument to pass to the program. |
| `.args(args)` | Adds multiple arguments from an iterator. |
| `.env(key, val)` | Sets an environment variable. |
| `.envs(vars)` | Adds multiple environment variables from an iterator. |
| `.env_remove(key)` | Removes an environment variable. |
| `.env_clear()` | Clears all inherited environment variables. |
| `.current_dir(dir)` | Sets the working directory for the child process. |

Example of configuring a command:

```rust
# use tokio::process::Command;
# async fn run() {
let output = Command::new("git")
        .arg("log")
        .arg("--oneline")
        .current_dir("/path/to/repo")
        .env("PAGER", "cat")
        .output()
        .await
        .unwrap();
# }
```

### Configuring I/O

You can control how the child process's standard I/O streams are handled using `stdin`, `stdout`, and `stderr` methods. These accept a `std::process::Stdio` value.

- `Stdio::inherit()`: (Default for `spawn` and `status`) The child inherits the handle from the parent.
- `Stdio::piped()`: (Default for `output`) Creates a pipe to communicate with the child process.
- `Stdio::null()`: Creates a new null handle.

```rust
use std::process::Stdio;
use tokio::process::Command;

async fn run_command_with_piped_stdout() {
    let mut cmd = Command::new("ls");
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn()
        .expect("failed to spawn command");
}
```

### Spawning the Process

There are three primary ways to execute the configured command:

| Method | Return Type | Description |
|---|---|---|
| `.spawn()` | `io::Result<Child>` | Executes the command and returns a `Child` handle immediately. You can then interact with the process and wait for it to finish. |
| `.status()` | `impl Future<Output = io::Result<ExitStatus>>` | Executes the command, waits for it to complete, and returns its exit status. Standard I/O handles are closed. |
| `.output()` | `impl Future<Output = io::Result<Output>>` | Executes the command, waits for it to complete, and collects all of its stdout and stderr. Automatically pipes stdout and stderr. |

Capturing output is a common use case:

```rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
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

## The `Child` Handle

A `Child` instance is returned by `Command::spawn()` and represents a running child process. It provides methods to wait for the process, check its status, and access its I/O streams.

### Waiting for a Child

The primary way to wait for a process to finish is by awaiting the `Child` handle itself, or by calling the `.wait()` method.

- `.wait()`: Asynchronously waits for the child to exit completely, returning its `ExitStatus`.
- `.try_wait()`: Attempts to collect the exit status without blocking. Returns `Ok(Some(status))` if exited, `Ok(None)` if still running.
- `.wait_with_output()`: Waits for the child to exit and reads all data from its stdout and stderr streams.

### Terminating a Child

You can terminate a child process forcefully.

- `.start_kill()`: Sends a kill signal to the child but does not wait for it to exit.
- `.kill()`: Sends a kill signal and then asynchronously waits for the process to be fully terminated.

```rust
# use tokio::process::Command;
# use tokio::time::{sleep, Duration};
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut child = Command::new("sleep").arg("5").spawn()?;

    // Do some other work
    sleep(Duration::from_millis(100)).await;

    // Terminate the process
    child.kill().await?;
    println!("Killed the child process");

    Ok(())
}
```

### Accessing I/O Streams

If I/O was configured to be piped, the `Child` struct will contain `stdin`, `stdout`, and `stderr` fields of type `Option<ChildStdin>`, `Option<ChildStdout>`, and `Option<ChildStderr>`.

It is common to `.take()` these handles to gain ownership without borrowing issues on the `Child` struct.

```rust
# use tokio::process::Command;
# use std::process::Stdio;
# #[tokio::main]
# async fn main() {
let mut child = Command::new("cat")
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .spawn()
    .unwrap();

let mut stdin = child.stdin.take().expect("child did not have a handle to stdin");
let mut stdout = child.stdout.take().expect("child did not have a handle to stdout");

// Now you can work with stdin and stdout while still being able to call methods on child
let status = child.wait().await;
# }
```

## Advanced Usage: Piping

You can pipe the standard output of one command into the standard input of another. This requires coordinating the handles between two `Child` processes.

```d2
shape: sequence_diagram

App: "Tokio Application"
Echo: "Process: echo"
Tr: "Process: tr"

App -> Echo: "spawn() with stdout piped"
Echo -> App: "Child handle (with stdout)"
App -> Tr: "spawn() with stdin from echo's stdout"
Tr -> App: "Child handle"

note over Echo, Tr: "OS pipes stdout of echo to stdin of tr"

App -> Echo: "wait()"
App -> Tr: "wait_with_output()"
```

Here is the implementation of the diagram above:

```rust
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

    // Create a Stdio object from the handle
    let tr_stdin: Stdio = echo_stdout.try_into().expect("failed to convert to Stdio");

    // Spawn the `tr` command, using echo's stdout as its stdin
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

There are some important behaviors to be aware of when working with asynchronous processes.

### Dropping and Cancellation

By default, dropping a `Child` handle does **not** terminate the underlying process. The process will continue to run in the background. If you need the process to be killed when the handle is dropped, you must configure it on the `Command` before spawning.

```rust
# use tokio::process::Command;
let mut child = Command::new("sleep")
    .arg("100")
    .kill_on_drop(true) // Set the process to be killed on drop
    .spawn()
    .unwrap();

// When `child` goes out of scope here, a kill signal will be sent.
drop(child);
```

### Unix Zombie Processes

On Unix-like systems, a process that has exited but has not been waited on by its parent becomes a "zombie" process. These processes consume system resources. The Tokio runtime attempts to reap spawned processes on a best-effort basis, but for stronger guarantees, you should always explicitly wait for the `Child` to complete.

> It is recommended to avoid dropping a `Child` handle before it has been fully awaited if stricter cleanup guarantees are required.