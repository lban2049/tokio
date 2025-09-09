# 计时器

Tokio 提供了用于跟踪时间和调度工作在特定时间段后执行的实用工具。这些工具对于处理异步应用程序中的超时、延迟和周期性任务至关重要。主要的时间相关组件有：

*   **Sleep**: 在特定时间点完成的 future。
*   **Interval**: 以固定周期产生值的流。
*   **Timeout**: 限制 future 最大执行时间的包装器。

所有计时器实用工具都需要在 Tokio [Runtime](./concepts-runtime.md) 的上下文中使用，该运行时提供了必要的计时器实现。

## 休眠：暂停任务

引入延迟的最简单方法是使用 `tokio::time::sleep`。此函数返回一个在指定持续时间过后完成的 future。它是 `std::thread::sleep` 的异步等效版本。

在等待 `Sleep` future 期间，不会执行任何工作。这使得其他任务可以并发运行。

```rust main.rs icon=logos:rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    println!("Waiting...");
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

如果需要等待到某个特定时刻，可以使用 `sleep_until(deadline)`，其中 `deadline` 是一个 `Instant`。

**取消**：取消休眠操作很简单，只需丢弃 `Sleep` future 即可。无需额外的清理工作。

**Panics**: 在 Tokio 运行时之外使用 `sleep` 会导致 panic。这是因为该函数需要访问运行时的计时器驱动程序。请确保它在由 Tokio 管理的 `async` 块内调用，例如通过 `#[tokio::main]` 或 `runtime.block_on()`。

## 间隔：重复操作

对于需要按计划重复运行的任务，Tokio 提供了 `tokio::time::interval`。一个 interval 以指定周期产生 tick。与在循环中调用 `sleep` 的关键区别在于，`Interval` 会将任务本身所花费的时间计算在内，从而确保 tick 以更规则的频率发生。

如果在循环中使用 `sleep`，每次迭代的总时间将是休眠时长*加上*任务执行时间。而使用 `interval`，即使任务需要一些时间来执行，tick 之间的时间也保持一致。

```rust main.rs icon=logos:rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("Executing task...");
    time::sleep(time::Duration::from_secs(1)).await;
}

#[tokio::main]
async fn main() {
    let mut interval = time::interval(time::Duration::from_secs(2));
    // The first tick completes immediately.
    for _i in 0..5 {
        interval.tick().await;
        task_that_takes_a_second().await;
    }
}
```
在上面的示例中，循环大约每两秒运行一次。如果使用 `sleep` 而不是 `interval.tick()`，它将每三秒（2 秒休眠 + 1 秒任务）运行一次。

### 错过 Tick 的行为

如果 `tick().await` 调用之间的代码执行时间超过了 interval 周期，可能会“错过”一个或多个 tick。你可以使用 `set_missed_tick_behavior()` 并通过以下策略之一来配置 interval 如何处理这种情况：

*   `Burst` (默认): interval 将尽快触发 tick，以追赶上它本应达到的进度。
*   `Delay`: 从当前时刻开始调度下一个 tick，这实际上重置了 interval 的相位。
*   `Skip`: interval 会跳过错过的 tick，并等待下一个常规调度的 tick。

## 超时：设置截止时间

为防止 future 无限期运行，可以将其包装在 `tokio::time::timeout` 中。该函数接受一个持续时间和 一个 future，并返回一个解析为 `Result` 的新 future。

*   如果内部 future 在持续时间结束前完成，新的 future 将解析为 `Ok(value)`。
*   如果在内部 future 完成前持续时间已过，新的 future 将解析为 `Err(Elapsed)`，并且内部 future 会被取消。

这对于构建健壮的网络应用程序至关重要，因为远程服务可能会变得无响应。

```rust main.rs icon=logos:rust
use tokio::time::{timeout, Duration};

async fn long_running_operation() {
    // This operation might take a long time.
    tokio::time::sleep(Duration::from_secs(5)).await;
    println!("Operation complete!");
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_running_operation()).await;

    if res.is_err() {
        println!("Operation timed out!");
    }
}
```

---

现在你对如何在 Tokio 中管理基于时间的操作有了概念性的理解。有关完整的函数列表和详细的配置选项，请参阅 [Time API 参考](./api-time.md)。