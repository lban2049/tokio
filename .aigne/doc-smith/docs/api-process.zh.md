# 进程

该模块为 Tokio 提供了异步进程管理的实现。它提供了一个 `Command` 结构体，其 API 与标准库中的 `std::process::Command` 非常相似，但包含了用于创建和管理进程的异步方法。

这些异步函数（如 `spawn`、`status` 和 `output`）返回与 future 兼容的类型，可与 Tokio 运行时无缝集成。这使你可以在不阻塞线程的情况下管理子进程，像处理任何其他异步任务一样处理它们。

## 快速示例

### 生成一个进程并等待其完成

最基本的用例是运行一个命令并等待其完成状态。

```rust,no_run
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
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

### 生成一个进程并捕获其输出

对于许多应用程序，你需要捕获子进程的输出（stdout 和 stderr）。

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

## `Command` 结构体

`Command` 结构体是用于配置和生成子进程的主要构建器。它提供了一个链式接口，用于在执行前设置参数、环境变量、工作目录和 I/O 句柄。

### 配置

在生成进程之前，你可以通过多种方式对其进行配置：

| Method | Description |
|---|---|
| `new(program)` | 构造一个新的 `Command` 以启动指定的程序。 |
| `arg(arg)` | 添加一个要传递给程序的参数。 |
| `args(args)` | 从迭代器中添加多个参数。 |
| `env(key, val)` | 为子进程设置一个环境变量。 |
| `envs(vars)` | 添加或更新多个环境变量。 |
| `env_remove(key)` | 移除一个环境变量。 |
| `env_clear()` | 清除所有继承的环境变量。 |
| `current_dir(dir)` | 为子进程设置工作目录。 |
| `stdin(cfg)` | 配置标准输入 (stdin) 句柄。 |
| `stdout(cfg)` | 配置标准输出 (stdout) 句柄。 |
| `stderr(cfg)` | 配置标准错误 (stderr) 句柄。 |
| `kill_on_drop(bool)` | 如果 `Child` 句柄被丢弃，则终止子进程。 |

### 执行

配置完成后，你可以通过以下三种主要方式之一执行命令：

*   **`spawn()`**：执行命令并返回一个 `Child` 句柄，从而可以与正在运行的进程进行详细交互。
*   **`status()`**：一个便捷方法，用于运行命令并等待其 `ExitStatus`。
*   **`output()`**：一个便捷方法，用于运行命令，等待其完成，并收集其所有的标准输出和标准错误。

## `Child` 结构体

`Child` 句柄由 `spawn` 方法返回，代表一个正在运行的子进程。它提供了等待进程退出、终止进程以及访问其 I/O 流的方法。

### 管理进程

| Method | Description |
|---|---|
| `wait()` | 异步等待子进程完全退出，并返回其 `ExitStatus`。 |
| `kill()` | 强制终止子进程并等待其被回收。 |
| `start_kill()` | 发送终止信号，但不等待进程退出。 |
| `try_wait()` | 在不阻塞的情况下检查子进程是否已退出。 |
| `id()` | 返回操作系统分配的进程标识符。 |

### 与 I/O 交互

如果 `Command` 配置了管道 I/O (`Stdio::piped()`)，则 `Child` 句柄将包含 `stdin`、`stdout` 和 `stderr` 的句柄。它们分别实现了 `AsyncWrite` 和 `AsyncRead`。

以下是一个向子进程的 stdin 写入数据并从其 stdout 读取数据的示例：

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

    // 在一个单独的任务中向子进程的 stdin 写入数据。
    tokio::spawn(async move {
        stdin.write_all(b"dog\nbird\nfrog\ncat\nfish\n").await.unwrap();
    });

    let output = child.wait_with_output().await?;

    assert_eq!(output.stdout, b"bird\ncat\ndog\nfish\nfrog\n");

    Ok(())
}
```

## 注意事项

在使用 Tokio 进程时，需要注意一些重要的行为。

### 丢弃与取消

与大多数 future 不同，丢弃 `Child` 句柄不会自动终止进程。该进程将继续在后台运行。如果你希望在句柄被丢弃时终止进程，必须在生成进程前使用 `.kill_on_drop(true)` 配置命令。

### Unix 上的僵尸进程

在类 Unix 系统上，一个已退出但其父进程未对其进行等待的进程会变成“僵尸进程”。这些进程会消耗系统资源。Tokio 运行时会尽力回收生成的进程，但为了保证清理，建议始终使用 `.wait().await` 或类似方法显式等待 `Child` 完成。