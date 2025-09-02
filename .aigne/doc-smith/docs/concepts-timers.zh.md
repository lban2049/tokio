# 计时器

Tokio 提供了用于跟踪时间和调度未来要运行的任务的实用工具。这些工具对于处理异步应用程序中的延迟、周期性任务和最后期限至关重要。所有计时器实用工具都需要一个活动的 Tokio `Runtime`。

Tokio 与时间相关的功能主要由三种类型组成：

<x-cards data-columns="3">
  <x-card data-title="Sleep" data-icon="lucide:timer-off">将当前任务暂停指定的持续时间或直到某个特定的时间点。</x-card>
  <x-card data-title="Interval" data-icon="lucide:repeat">创建一个以固定周期产生值的流，非常适合周期性任务。</x-card>
  <x-card data-title="Timeout" data-icon="lucide:alarm-clock">包装一个 future，对其执行强制施加时间限制。</x-card>
</x-cards>

## Sleep：暂停执行

最基本的计时器是 `tokio::time::sleep`。此函数会创建一个在指定持续时间过后完成的 future。它是 `std::thread::sleep` 的异步等效项。

你可以使用 `sleep` 在任务中引入延迟，而不会阻塞整个线程，从而允许其他任务并发运行。

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    println!("Waiting...");
    sleep(Duration::from_millis(500)).await;
    println!("500 ms have elapsed");
}
```

同样，`sleep_until(deadline)` 可用于暂停任务，直到达到某个特定的 `Instant`。

> **关于 Panic 的说明**：使用 `sleep` 等计时器函数需要一个已启用计时器的 Tokio runtime。如果你在运行的 Tokio runtime 之外调用计时器函数，或者在构建时未启用计时器的 runtime（`Builder::enable_time` 或 `Builder::enable_all`）上调用，你的程序将会 panic。

## Interval：按计划重复

对于需要以固定频率重复运行的任务，你可以在循环中使用 `sleep`。然而，这种方法可能会随着时间的推移导致偏差，因为任务本身所花费的时间没有被计算在内。总周期时间会变为 `sleep_duration + task_duration`。

Tokio 的 `tokio::time::interval` 提供了一种更精确的解决方案。`Interval` 会按指定周期生成一个“滴答” (tick) 流。当你 `await` 下一个滴答时，`interval` 会计算正确的延迟以维持期望的频率，同时会考虑自上一个滴答以来你的任务所花费的时间。

下面的图表演示了其中的差异：

```d2
direction: down

subsystem_A: "不精确：使用 sleep 的循环" {
  timeline_A: {
    direction: right
    shape: sequence_diagram

    start_1: "周期 1 开始 (0s)"
    sleep_1: "sleep(2s).await"
    work_1: "work(1s).await"
    end_1: "周期 1 结束 (~3s)"

    start_1 -> sleep_1
    sleep_1 -> work_1
    work_1 -> end_1
  }
  style.stroke: "#D83B01"
  "每个周期总时间漂移 +1s"
}

subsystem_B: "精确：使用 interval 的循环" {
  timeline_B: {
    direction: right
    shape: sequence_diagram

    start_2: "周期 1 开始 (0s)"
    tick_1: "interval.tick().await"
    work_2: "work(1s).await"
    end_2: "周期 1 结束 (2s)"
    tick_2: "interval.tick().await"

    start_2 -> tick_1: "立即"
    tick_1 -> work_2
    work_2 -> end_2
    end_2 -> tick_2: "等待 ~1s"
  }
  style.stroke: "#107C10"
  "每个周期的总时间保持在 2s"
}
```

以下是使用 `interval` 的一个实际示例：

```rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("Executing task...");
    time::sleep(time::Duration::from_secs(1)).await
}

#[tokio::main]
async fn main() {
    let mut interval = time::interval(time::Duration::from_secs(2));
    for _i in 0..5 {
        interval.tick().await;
        task_that_takes_a_second().await;
    }
}
```

### 处理错过的滴答

如果你的任务耗时超过了 `interval` 的周期，那么一个滴答 (tick) 就被认为是“错过了”。`Interval` 提供了一个 `MissedTickBehavior` 枚举来控制其追赶方式。你可以使用 `set_missed_tick_behavior()` 来进行设置。

*   **`Burst` (默认)：** `interval` 会尽快触发滴答，直到赶上进度。这些滴答的时间戳将与没有发生延迟时应有的时间戳保持一致。
*   **`Delay`：** 下一个滴答的调度是相对于上一次 `tick()` 调用完成的时间，这实际上重置了 `interval` 的计划。这确保了滴答之间总是经过一个完整的周期。
*   **`Skip`：** `interval` 会跳过所有错过的滴答，并在原始周期的下一个倍数点上调度下一个滴答。这可能导致为了回到正轨而出现比平时更短的延迟。

## Timeout：强制执行最后期限

为防止 future 无限期运行，你可以使用 `tokio::time::timeout` 对其进行包装。如果该 future 未在指定时间内完成，它将被取消，并且 `timeout` future 会解析为一个错误。

`timeout` 函数返回一个 `Result`。如果内部的 future 及时完成，你会得到 `Ok(value)`；否则，你会得到 `Err(Elapsed)`。

```rust
use tokio::time::{timeout, Duration};
use tokio::sync::oneshot;

async fn long_running_task() {
    // Simulate a task that might or might not complete in time.
    tokio::time::sleep(Duration::from_millis(150)).await;
}

#[tokio::main]
async fn main() {
    let duration = Duration::from_millis(100);

    if let Err(_) = timeout(duration, long_running_task()).await {
        println!("The task timed out after {:?}", duration);
    } else {
        println!("The task completed within {:?}", duration);
    }
}
```

与 `sleep` 类似，`timeout` 也有一个 `timeout_at(deadline)` 变体，它可用于处理特定的 `Instant`。

这些计时器实用工具为控制异步应用程序中的时间流提供了基础构建模块。有关更多详细信息，请参阅 [`tokio::time` API 参考](./api-time.md)。