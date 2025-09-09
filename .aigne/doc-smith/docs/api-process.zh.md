# 进程

该模块为 Tokio 提供了一个异步进程管理的实现。它提供了一个 `Command` 结构体，该结构体模仿了标准库 [`std::process::Command`](https://doc.rust-lang.org/std/process/Command.html) 的接口，但提供了创建和管理子进程的函数的异步版本。

这些函数（`spawn`、`status`、`output` 及其变体）返回能与 Tokio 运行时无缝集成的、感知 future 的类型。这种异步支持在 Unix 上通过信号处理实现，在 Windows 上通过专门的系统 API 实现。

## 快速入门：生成一个进程

最常见的用例是生成一个命令，执行它，然后等待其完成。其接口设计与标准库非常相似。

```rust Spawning a simple command icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 用法与标准库的 `Command` 类型相似
    let mut child = Command::new("echo")
        .arg("hello")
        .arg("world")
        .spawn()
        .expect("failed to spawn");

    // 等待子进程完成
    let status = child.wait().await?;
    println!("the command exited with: {}", status);
    Ok(())
}
```

## `Command` 结构体

`Command` 结构体是用于异步配置和执行新子进程的主要构建器。

### 创建和配置命令

首先创建一个新的 `Command`，然后使用其构建器方法在执行前配置进程。

| Method | Description |
| --- | --- |
| `new(program)` | 构造一个新的 `Command` 以启动指定的程序。 |
| `arg(arg)` | 添加一个要传递给程序的参数。 |
| `args(args)` | 添加多个要传递给程序的参数。 |
| `env(key, val)` | 为子进程设置一个环境变量。 |
| `envs(vars)` | 添加或更新多个环境变量。 |
| `env_remove(key)` | 移除一个环境变量。 |
| `env_clear()` | 清除子进程的所有环境变量。 |
| `current_dir(dir)` | 设置子进程的工作目录。 |
| `stdin(cfg)` | 配置标准输入 (stdin) 句柄。 |
| `stdout(cfg)` | 配置标准输出 (stdout) 句柄。 |
| `stderr(cfg)` | 配置标准错误 (stderr) 句柄。 |

### 执行命令

配置完成后，可以通过几种方式执行命令：

*   **`spawn()`**：执行命令作为子进程，并立即返回其句柄。这是最灵活的方法，它提供一个 `Child` 对象供您管理。

*   **`status()`**：执行命令并等待其完成，仅返回其 `ExitStatus`。当您不需要与子进程的 I/O 交互时，这是一种方便的简写方式。

*   **`output()`**：执行命令，等待其完成，并收集其所有的标准输出和标准错误。这对于捕获命令的完整输出很有用。

```rust Capturing output icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 使用 `output()`，它返回一个 future，该 future 会解析为进程的输出
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

## `Child` 结构体

当您 `spawn` 一个进程时，会得到一个 `Child` 结构体。该结构体代表正在运行的子进程，并提供与其交互的方法。

### 等待子进程

最基本的操作是等待进程退出。`Child` 结构体本身可以被 `.await`，这等同于调用 `wait()` 方法。

*   **`wait()`**：异步等待子进程完全退出，返回其 `ExitStatus`。
*   **`try_wait()`**：尝试在不阻塞的情况下收集退出状态。如果子进程已退出，则返回 `Ok(Some(status))`；如果仍在运行，则返回 `Ok(None)`；如果出错，则返回 `Err`。
*   **`wait_with_output()`**：一个 future，它会等待子进程退出，并收集其 stdout 和 stderr 句柄上的所有剩余输出。

### 终止子进程

如果需要，可以强制终止子进程。

*   **`start_kill()`**：向子进程发送终止信号，但不等待其退出。
*   **`kill()`**：发送终止信号，然后异步等待进程完全终止。

```rust Killing a child process icon=logos:rust
use tokio::process::Command;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut child = Command::new("sleep").arg("5").spawn()?;

    // 给它一点时间启动
    sleep(Duration::from_millis(100)).await;

    println!("Killing the child process");
    child.kill().await?;

    let status = child.wait().await?;
    println!("Child exited with: {}", status);

    Ok(())
}
```

## 处理 I/O

当需要与子进程的标准输入、输出和错误流进行交互时，Tokio 的进程管理功能就显得尤为突出。为此，必须将相应的句柄配置为 `Stdio::piped()`。

### 从子进程的 `stdout` 读取

一旦生成了带有管道化 stdout 的子进程，就可以从 `child.stdout` 获取一个异步读取器。

```rust Reading stdout line by line icon=logos:rust
use tokio::io::{AsyncBufReadExt, BufReader};
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("cat");

    // 将命令的标准输出通过管道传回给我们。
    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn()
        .expect("failed to spawn command");

    let stdout = child.stdout.take()
        .expect("child did not have a handle to stdout");

    let mut reader = BufReader::new(stdout).lines();

    // 确保子进程在运行时中生成以取得进展。
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

### 向子进程的 `stdin` 写入

同样，如果子进程的标准输入被配置为管道，也可以异步地向其写入数据。

```rust Writing to stdin and reading from stdout icon=logos:rust
use tokio::io::AsyncWriteExt;
use tokio::process::Command;
use std::process::Stdio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("sort");

    // 将 stdout 和 stdin 都设置为管道
    cmd.stdout(Stdio::piped());
    cmd.stdin(Stdio::piped());

    let mut child = cmd.spawn().expect("failed to spawn command");

    let mut stdin = child.stdin.take().expect("child did not have a handle to stdin");

    // 在一个单独的任务中向子进程的 stdin 写入数据
    tokio::spawn(async move {
        stdin.write_all(b"dog\nbird\nfrog\ncat\nfish\n").await.unwrap();
        // 丢弃 stdin 会向子进程发送 EOF 信号
    });

    let output = child.wait_with_output().await?;

    // 输出应为排序后的结果
    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```

### 在进程间使用管道

您还可以将一个命令的输出直接通过管道传递给另一个命令的输入。

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

    // 获取第一个命令的 stdout
    let echo_stdout = echo.stdout.take().unwrap();

    // 从第一个命令的 stdout 创建第二个命令的 stdin
    let tr_stdin: Stdio = echo_stdout.try_into().expect("failed to convert to Stdio");

    let tr = Command::new("tr")
        .arg("a-z")
        .arg("A-Z")
        .stdin(tr_stdin)
        .stdout(Stdio::piped())
        .spawn()
        .expect("failed to spawn tr");

    // 等待两个进程完成
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

与大多数 future 不同，丢弃 `Child` 句柄并**不会**自动终止进程。子进程将继续在后台运行。如果希望在句柄被丢弃时终止子进程，必须使用 `Command` 构建器上的 `kill_on_drop` 方法来配置此行为。

```rust
let mut child = Command::new("sleep")
    .arg("1000")
    .kill_on_drop(true) // 如果 Child 句柄被丢弃，则终止该进程
    .spawn()
    .unwrap();

// 当 `child` 在此处超出作用域时，将发送一个终止信号。
```

### Unix 上的僵尸进程

在类 Unix 系统上，一个已经退出但其父进程尚未对其进行等待（wait）的进程会变成“僵尸进程”。这些进程会消耗系统资源。Tokio 运行时会尽力回收它所生成的僵尸进程。然而，为确保清理，强烈建议始终使用 `.await` 或 `wait()` 显式等待 `Child` 完成。