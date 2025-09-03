# 计时器

Tokio 提供了用于跟踪时间和调度任务在设定时间后执行的实用工具。这些工具对于处理延迟、周期性任务和有截止日期的操作至关重要。

所有计时器实用工具都必须在 Tokio [Runtime](./concepts-runtime.md) 的上下文中使用，因为它们依赖其内部计时器进行调度。

Tokio 的主要时间相关组件是：

- **`Sleep`**：一个在特定时刻完成的 future。
- **`Interval`**：一个按固定周期产生值的流。
- **`Timeout`**：一个限制 future 最大执行时间的包装器。

我们来逐一探讨这些概念。

## Sleep：等待一段时间

最基本的计时器原语是 `sleep`。它创建一个在指定时长过去后完成的 future。这是 `std::thread::sleep` 的异步等价物。

当您需要暂停任务一段固定时间而又不阻塞整个线程时，`sleep` 非常有用。

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 毫秒已过去");
}
```

如果要等到一个特定的时间点，可以使用 `sleep_until(instant)`。如果在 `Sleep` future 完成前丢弃它，计时器将被取消，无需额外的清理工作。

## Interval：按计划重复

虽然可以在循环中使用 `sleep` 来周期性地执行任务，但这种方法可能会导致时间漂移。任务执行本身所花费的时间没有被计算在内，因此实际周期将是 `sleep_duration + task_duration`。

对于更精确的周期性任务，Tokio 提供了 `interval`。`interval` 从*上一次* tick 完成时开始计时，确保 tick 以更一致的频率发生。

下图说明了两者之间的差异：

```d2
direction: down

"使用 sleep(2s) 的循环": {
  shape: package
  grid-columns: 1

  "时间线": {
    shape: sequence_diagram
    "任务": {shape: step}
    "休眠": {shape: step}

    "0s" -> "任务": "Tick 1"
    "任务" -> "休眠": "工作 (1s)"
    "休眠" -> "3s": "休眠 (2s)"
    "3s": "Tick 2"
  }
  "总周期变为约 3 秒。"
}

"Interval(2s)": {
  shape: package
  grid-columns: 1

  "时间线": {
    shape: sequence_diagram
    "任务": {shape: step}
    "等待": {shape: step}

    "0s" -> "任务": "Tick 1"
    "任务" -> "等待": "工作 (1s)"
    "等待" -> "2s": "等待 (约 1s)"
    "2s": "Tick 2"
  }
  "总周期保持为 2 秒。"
}
```

下面是一个使用 `interval` 每两秒运行一次任务的实际示例，任务本身需要一秒钟：

```rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("正在执行任务...");
    time::sleep(time::Duration::from_secs(1)).await
}

#[tokio::main]
async fn main() {
    let mut interval = time::interval(time::Duration::from_secs(2));
    for _i in 0..3 {
        interval.tick().await;
        task_that_takes_a_second().await;
    }
}
```
在此示例中，“Executing task...” 消息将大约在 t=0s、t=2s 和 t=4s 时打印。

### 处理错过的 Tick

如果任务的执行时间超过了 interval 的周期，则称 interval“错过了一个 tick”。您可以使用 `MissedTickBehavior` 配置 `Interval` 在此场景下的行为：

- **`Burst` (默认)：** interval 将尽快触发 tick，直到赶上进度。
- **`Delay`：** 下一个 tick 相对于当前时间进行调度，从而有效重置 interval 的相位。
- **`Skip`：** interval 会跳过错过的 tick，并在下一个常规 interval 点调度下一个 tick。

## Timeout：限制执行时间

通常，您需要确保一个操作不会花费太长时间完成。`timeout` 函数包装一个 future，并对其执行施加时间限制。

如果内部 future 在指定时间内完成，`timeout` 返回 `Ok(value)`。如果时间先耗尽，内部 future 将被取消，`timeout` 返回 `Err(Elapsed)`。

```rust
use tokio::time::{timeout, Duration};

async fn long_running_task() {
    // 此任务耗时超过超时时间。
    sleep(Duration::from_secs(5)).await;
    println!("任务完成！");
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_running_task()).await;

    if res.is_err() {
        println!("操作超时！");
    }
}
```

这对于为 I/O 操作（如网络请求或数据库查询）添加截止日期非常有用，可以防止应用程序无限期挂起。

---

Tokio 的时间实用工具——`sleep`、`interval` 和 `timeout`——为在异步应用程序中管理基于时间的逻辑提供了必要的工具。有关函数的完整列表和详细选项，请参阅[时间 API 参考](./api-time.md)。