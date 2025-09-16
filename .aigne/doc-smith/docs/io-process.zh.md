# 子进程

Tokio 提供了异步管理子进程的工具。其 `tokio::process` 模块提供了一个 `Command` 结构体，该结构体模仿了标准库 `std::process::Command` 的接口，但提供了用于生成和管理进程的异步方法。

这使得你可以在不阻塞 Tokio 运行时的情况下，执行外部命令、捕获其输出，并与其标准 I/O 流进行交互。

## `Command` 结构体

进程管理的主要入口点是 `tokio::process::Command` 结构体。你可以使用它在生成进程前进行配置，例如设置参数、环境变量和 I/O 管道。

### 生成一个简单的进程

让我们从一个基本示例开始。我们将生成一个 `echo` 命令，等待其完成，并检查其退出状态。

```rust Spawning a Command icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 用法与标准库的 `Command` 类型类似
    let mut child = Command::new("echo")
        .arg("hello")
        .arg("world")
        .spawn()
        .expect("failed to spawn
        ");

    // 等待子进程完成
    let status = child.wait().await?;

    println!("the command exited with: {}", status);
    Ok(())
}
```

这段代码创建了一个新的 `Command`，将其配置为运行 `echo hello world`，然后生成该进程并异步等待其退出。

### 捕获输出

通常，你需要捕获命令的输出。`output()` 方法会生成进程并返回一个 future，该 future 会解析为一个包含 `stdout`、`stderr` 和退出 `status` 的 `Output` 结构体。

```rust Capturing Command Output icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 使用 `output()`，它返回一个 future，该 future 会解析为命令的输出
    let output = Command::new("echo").arg("hello").arg("world")
                        .output()
                        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");

    println!("Command executed successfully");
    Ok(())
}
```

## 与 I/O 交互（管道）

对于更复杂的交互，你可以通过管道连接子进程的标准输入、输出和错误流。这可以通过将 `stdin`、`stdout` 和 `stderr` 句柄配置为使用 `std::process::Stdio::piped()` 来实现。

### 从 `stdout` 读取

你可以在子进程生成标准输出时捕获并读取它。

```rust Reading from stdout icon=logos:rust
use tokio::io::{BufReader, AsyncBufReadExt};
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("cat");

    // 将命令的标准输出通过管道传给我们
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn()
        .expect("failed to spawn command");

    let stdout = child.stdout.take()
        .expect("child did not have a handle to stdout");

    let mut reader = BufReader::new(stdout).lines();

    // 生成一个任务来等待子进程完成。
    // 如果子进程产生大量输出，这可以防止死锁。
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
在此示例中，我们获取 `child.stdout` 的所有权，并将其包装在 `BufReader` 中以逐行读取其输出。`child.wait()` future 被移至一个单独的任务中，以确保在处理其输出的同时，子进程能够运行至完成。

### 写入 `stdin`

同样，你也可以向子进程的标准输入写入数据。

```rust Writing to stdin icon=logos:rust
use tokio::io::AsyncWriteExt;
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("sort");

    // 将 stdout 和 stdin 都通过管道连接
    cmd.stdout(Stdio::piped());
    cmd.stdin(Stdio::piped());

    let mut child = cmd.spawn().expect("failed to spawn command");

    let animals = b"dog\nbird\nfrog\ncat\nfish";

    let mut stdin = child.stdin.take().expect("child did not have a handle to stdin");

    // 将我们的 animals 写入子进程的 stdin
    stdin.write_all(animals).await.expect("could not write to stdin");

    // 丢弃 stdin 句柄以向子进程发送 EOF 信号。
    drop(stdin);

    let output = child.wait_with_output().await?;

    // 期望结果是排序好的
    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```
在这里，我们向 `sort` 命令写入一个动物列表。写入后调用 `drop(stdin)`至关重要。这会关闭管道，向子进程发送文件结束（EOF）信号，子进程从而知道应结束处理并退出。

### 在命令之间使用管道

你可以将一个命令的输出直接通过管道传送给另一个命令的输入。

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

    // 获取 echo 进程的 stdout
    let echo_stdout = echo.stdout.take().unwrap();

    // 将 ChildStdout 转换为下一个命令的 Stdio 句柄
    let tr_stdin: Stdio = echo_stdout.try_into().expect("failed to convert to Stdio");

    let tr = Command::new("tr")
        .arg("a-z")
        .arg("A-Z")
        .stdin(tr_stdin)
        .stdout(Stdio::piped())
        .spawn()
        .expect("failed to spawn tr");

    // 等待两个进程都完成
    let (echo_result, tr_output) = join!(echo.wait(), tr.wait_with_output());

    assert!(echo_result.unwrap().success());
    let tr_output = tr_output.expect("failed to await tr");
    assert!(tr_output.status.success());
    assert_eq!(tr_output.stdout, b"HELLO WORLD!\n");

    Ok(())
}
```

## 管理 `Child` 句柄

当你生成一个进程时，会得到一个 `Child` 句柄。此句柄可用于管理正在运行的进程。

- **`wait()`**: 异步等待子进程退出。
- **`id()`**: 获取由操作系统分配的进程 ID。
- **`start_kill()`**: 发起一个终止信号（在 Unix 上为 `SIGKILL`），但不会等待进程退出。
- **`kill()`**: 一个便捷方法，它会发送一个终止信号，然后等待进程退出。

以下示例展示了如何在某个特定事件发生时终止一个子进程：

```rust Killing a Child Process icon=logos:rust
use tokio::process::Command;
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<()>();
    let mut child = Command::new("sleep").arg("5").spawn().unwrap();

    // 发送一个信号来终止该进程
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

## 重要注意事项

### 丢弃与取消

默认情况下，丢弃 `Child` 句柄**不会**终止进程。该进程将继续在后台运行。这与标准库的行为一致。

如果你希望在 `Child` 句柄被丢弃时终止子进程，则必须在生成进程前在 `Command` 上进行配置：

```rust
let mut child = Command::new("long-running-process")
    .kill_on_drop(true)
    .spawn()?;
```

### Unix 上的僵尸进程

在类 Unix 系统上，一个已退出但其父进程尚未等待（wait on）它的进程会成为“僵尸进程”。这些僵尸进程仍会消耗系统资源（如进程 ID）。

Tokio 运行时会尽力自动回收其生成的僵尸进程。然而，对于需要严格保证资源清理的应用程序，强烈建议使用 `child.wait().await` 显式等待每个子进程完成，而不是依赖后台的回收机制。

## 后续步骤

管理子进程通常与处理系统信号相辅相成。要了解如何响应 `SIGINT` (Ctrl-C) 等事件，请继续阅读下一节。

<x-card data-title="操作系统信号" data-href="/io/signals" data-icon="lucide:radio-tower">
  学习如何在 Tokio 应用程序中异步处理 Unix 和 Windows 操作系统信号。
</x-card>