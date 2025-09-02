# 进程

该模块为 Tokio 提供了异步进程管理的实现。它提供了一个 `Command` 结构体，其 API 与 `std::process::Command` 非常相似，但为其进程创建函数提供了异步版本。

像 `spawn`、`status` 和 `output` 这样的关键函数都是异步的，它们返回与 future 兼容的类型，可以与 Tokio 运行时无缝集成。这使你可以在 Unix 上使用信号处理，在 Windows 上使用系统 API 来管理子进程，而不会阻塞线程。

## 快速入门：生成一个进程

使用该模块最直接的方法是生成一个命令并等待其完成。该接口的设计旨在让任何使用过标准库 `Command` 类型的用户都感到熟悉。

```rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 用法与标准库的 `Command` 类型相似
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

## 核心组件

进程管理 API 围绕几个关键类型构建。

<x-cards>
  <x-card data-title="Command" data-icon="lucide:terminal">
    一个命令构建器，用于配置和生成新的子进程。你可以用它来设置程序、参数、环境变量和 I/O 句柄。
  </x-card>
  <x-card data-title="Child" data-icon="lucide:cpu">
    一个指向正在运行的子进程的句柄。它允许你等待进程退出并管理其 I/O 流。
  </x-card>
  <x-card data-title="ChildStdin" data-icon="lucide:arrow-right-from-line">
    一个指向子进程标准输入（stdin）的句柄，实现了 `AsyncWrite`。
  </x-card>
  <x-card data-title="ChildStdout / ChildStderr" data-icon="lucide:arrow-left-from-line">
    指向子进程标准输出（stdout）和标准错误（stderr）的句柄，实现了 `AsyncRead`。
  </x-card>
</x-cards>

## `Command` 结构体

`Command` 结构体是创建和配置子进程的入口点。它提供了一个构建器模式，用于在生成进程之前设置其环境。

### 创建一个 Command

你可以通过指定要执行的程序来创建一个新的 `Command`。

```rust
use tokio::process::Command;

let mut command = Command::new("sh");
```

### 配置参数和环境

可以使用多种方法来配置参数、环境变量和工作目录。

| Method | Description |
|---|---|
| `.arg(arg)` | 添加一个要传递给程序的参数。 |
| `.args(args)` | 从一个迭代器中添加多个参数。 |
| `.env(key, val)` | 设置一个环境变量。 |
| `.envs(vars)` | 从一个迭代器中添加多个环境变量。 |
| `.env_remove(key)` | 移除一个环境变量。 |
| `.env_clear()` | 清除所有继承的环境变量。 |
| `.current_dir(dir)` | 为子进程设置工作目录。 |

配置命令的示例：

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

### 配置 I/O

你可以使用 `stdin`、`stdout` 和 `stderr` 方法来控制如何处理子进程的标准 I/O 流。这些方法接受一个 `std::process::Stdio` 值。

- `Stdio::inherit()`:（`spawn` 和 `status` 的默认值）子进程从父进程继承句柄。
- `Stdio::piped()`:（`output` 的默认值）创建一个管道与子进程通信。
- `Stdio::null()`: 创建一个新的空句柄。

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

### 生成进程

有三种主要方式来执行已配置的命令：

| Method | Return Type | Description |
|---|---|---|
| `.spawn()` | `io::Result<Child>` | 执行命令并立即返回一个 `Child` 句柄。之后你可以与该进程交互并等待其完成。 |
| `.status()` | `impl Future<Output = io::Result<ExitStatus>>` | 执行命令，等待其完成，并返回其退出状态。标准 I/O 句柄会被关闭。 |
| `.output()` | `impl Future<Output = io::Result<Output>>` | 执行命令，等待其完成，并收集其所有的 stdout 和 stderr。自动通过管道传输 stdout 和 stderr。 |

捕获输出是一个常见的用例：

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

## `Child` 句柄

`Command::spawn()` 会返回一个 `Child` 实例，它代表一个正在运行的子进程。该实例提供了等待进程、检查其状态以及访问其 I/O 流的方法。

### 等待子进程

等待进程完成的主要方式是等待 `Child` 句柄本身，或调用 `.wait()` 方法。

- `.wait()`: 异步等待子进程完全退出，并返回其 `ExitStatus`。
- `.try_wait()`: 尝试在不阻塞的情况下收集退出状态。如果已退出，则返回 `Ok(Some(status))`；如果仍在运行，则返回 `Ok(None)`。
- `.wait_with_output()`: 等待子进程退出，并从其 stdout 和 stderr 流中读取所有数据。

### 终止子进程

你可以强制终止一个子进程。

- `.start_kill()`: 向子进程发送终止信号，但不等待其退出。
- `.kill()`: 发送终止信号，然后异步等待进程被完全终止。

```rust
# use tokio::process::Command;
# use tokio::time::{sleep, Duration};
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut child = Command::new("sleep").arg("5").spawn()?;

    // 做一些其他工作
    sleep(Duration::from_millis(100)).await;

    // 终止进程
    child.kill().await?;
    println!("Killed the child process");

    Ok(())
}
```

### 访问 I/O 流

如果 I/O 被配置为管道化，`Child` 结构体将包含类型为 `Option<ChildStdin>`、`Option<ChildStdout>` 和 `Option<ChildStderr>` 的 `stdin`、`stdout` 和 `stderr` 字段。

通常会使用 `.take()` 方法来获取这些句柄的所有权，以避免在 `Child` 结构体上出现借用问题。

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

// 现在你可以在处理 stdin 和 stdout 的同时，仍然能够调用 child 上的方法
let status = child.wait().await;
# }
```

## 高级用法：管道

你可以将一个命令的标准输出通过管道连接到另一个命令的标准输入。这需要协调两个 `Child` 进程之间的句柄。

```d2
shape: sequence_diagram

App: "Tokio 应用程序"
Echo: "进程：echo"
Tr: "进程：tr"

App -> Echo: "使用管道化的 stdout 执行 spawn()"
Echo -> App: "Child 句柄（带 stdout）"
App -> Tr: "使用 echo 的 stdout 作为 stdin 执行 spawn()"
Tr -> App: "Child 句柄"

note over Echo, Tr: "操作系统将 echo 的 stdout 通过管道连接到 tr 的 stdin"

App -> Echo: "wait()"
App -> Tr: "wait_with_output()"
```

下面是上述图表的实现：

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

    // 获取 echo 进程的 stdout
    let echo_stdout = echo.stdout.take().unwrap();

    // 从句柄创建一个 Stdio 对象
    let tr_stdin: Stdio = echo_stdout.try_into().expect("failed to convert to Stdio");

    // 生成 `tr` 命令，使用 echo 的 stdout 作为其 stdin
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

## 注意事项

在使用异步进程时，需要注意一些重要的行为。

### 丢弃与取消

默认情况下，丢弃 `Child` 句柄**不会**终止底层进程。该进程将继续在后台运行。如果希望在丢弃句柄时终止进程，必须在生成进程前在 `Command` 上进行配置。

```rust
# use tokio::process::Command;
let mut child = Command::new("sleep")
    .arg("100")
    .kill_on_drop(true) // 设置进程在句柄被丢弃时终止
    .spawn()
    .unwrap();

// 当 `child` 在此处超出作用域时，将发送一个终止信号。
drop(child);
```

### Unix 僵尸进程

在类 Unix 系统上，一个已经退出但其父进程没有对其进行等待（wait）的进程会变成“僵尸”进程。这些进程会消耗系统资源。Tokio 运行时会尽力回收生成的进程，但为了获得更强的保证，你应该总是显式地等待 `Child` 完成。

> 如果需要更严格的清理保证，建议在 `Child` 句柄被完全等待之前，避免丢弃它。
