# 进程

此模块为 Tokio 提供了异步进程管理的实现。它提供了一个 `Command` 结构体，该结构体与标准库的 `std::process::Command` 接口类似，但提供了用于创建进程的异步版本函数。

这些函数（`spawn`、`status`、`output` 及其变体）返回与 Tokio 运行时集成的、支持 future 的类型。这种异步支持在 Unix 上通过信号处理实现，在 Windows 上则通过系统 API 实现。

## 快速入门

### 生成一个进程并等待其完成

基本用法与标准库非常相似。主要区别在于，等待子进程退出等操作是 `async` 的。

```rust,no_run
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 用法与标准库的 `Command` 类型类似
    let mut child = Command::new("echo")
        .arg("hello")
        .arg("world")
        .spawn()
        .expect("failed to spawn");

    // 等待命令完成
    let status = child.wait().await?;
    println!("the command exited with: {}", status);
    Ok(())
}
```

### 捕获输出

要生成一个进程并捕获其所有输出，可以使用 `output` 方法，该方法返回一个 future，该 future 会解析为进程的 `Output`。

```rust,no_run
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 使用 `output`，它返回一个 future，而不是
    // 立即返回 `Child`。
    let output = Command::new("echo").arg("hello").arg("world")
                        .output()
                        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");
    Ok(())
}
```

## 关键结构体

| Struct | Description |
|---|---|
| `Command` | 一个用于异步配置和生成新子进程的构建器。 |
| `Child` | 表示一个正在运行的子进程，提供对其 I/O 流的访问，并允许你等待其完成。 |
| `ChildStdin` | 一个子进程标准输入（stdin）的句柄，实现了 `AsyncWrite`。 |
| `ChildStdout` | 一个子进程标准输出（stdout）的句柄，实现了 `AsyncRead`。 |
| `ChildStderr` | 一个子进程标准错误（stderr）的句柄，实现了 `AsyncRead`。 |

## 高级用法

### 从 Stdout 读取

你可以通过管道传输子进程的标准输出，并异步地从中读取，例如，逐行读取。

```rust,no_run
use tokio::io::{BufReader, AsyncBufReadExt};
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("cat");

    // 指定我们希望将命令的标准输出通过管道回传给我们。
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn()
        .expect("failed to spawn command");

    let stdout = child.stdout.take()
        .expect("child did not have a handle to stdout");

    let mut reader = BufReader::new(stdout).lines();

    // 确保子进程在运行时中生成，以便它可以在
    // 我们等待任何输出时自行取得进展。
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

### 写入 Stdin

同样，你也可以通过管道连接到子进程的标准输入，并异步地向其写入。

```rust,no_run
use tokio::io::AsyncWriteExt;
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("sort");

    // 将 stdout 和 stdin 都通过管道连接。
    cmd.stdout(Stdio::piped());
    cmd.stdin(Stdio::piped());

    let mut child = cmd.spawn().expect("failed to spawn command");

    let mut stdin = child.stdin.take()
        .expect("child did not have a handle to stdin");

    // 在一个单独的任务中向子进程写入数据，以避免死锁。
    tokio::spawn(async move {
        stdin.write_all(b"dog\nbird\nfrog\ncat\nfish\n").await.unwrap();
        // 丢弃 stdin 表示文件结束（EOF）。
    });

    let output = child.wait_with_output().await?;

    // 结果应按排序顺序返回
    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```
注意：某些程序（如 `sort`）的行为是在写入任何输出之前缓冲所有输入。通常，建议在等待子进程退出或输出的任务之外，另开一个单独的任务来向子进程写入数据，以避免死锁。

### 在进程间使用管道

你可以将一个命令的输出通过管道连接到另一个命令的输入。

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

## 注意事项

### 丢弃与取消

默认情况下，一个已生成的进程在其 `Child` 句柄被丢弃后仍会继续执行。此行为与标准库类似，但与常见的 future 范式不同，在 future 范式中，丢弃意味着取消。

要改变此行为，可以使用 `Command::kill_on_drop(true)` 方法。这会导致在进程退出前如果 `Child` 句柄被丢弃，子进程将被终止。

### Unix 进程与僵尸进程

在 Unix 平台上，父进程必须在子进程退出后“回收”它，以释放所有操作系统资源。一个已退出但未被回收的子进程是“僵尸”进程。僵尸进程的累积会阻止新进程的生成。

Tokio 运行时会尽力回收它生成的任何进程。然而，为了获得更严格的清理保证，建议避免丢弃 `Child` 句柄，而是使用 `.wait().await` 显式等待其完成。
