# 时间、延迟和超时

Tokio 的时间模块提供了管理基于时间的异步操作所必需的实用工具。无论您需要暂停任务、周期性地运行某个操作，还是确保操作不会无限期地运行，`tokio::time` 模块都能提供您所需的工具。这些类型必须在 Tokio [Runtime](./tasks-scheduling-runtime.md) 的上下文中才能使用。

本指南涵盖三个主要组件：

- **`Sleep`**：一个在特定时间点完成的 future，可用于创建延迟。
- **`Interval`**：一个按固定周期产生值的流，非常适合周期性任务。
- **`Timeout`**：一个限制 future 最大执行时间的包装器。

## 使用 sleep 实现延迟

在异步任务中引入延迟最直接的方法是使用 `sleep` 函数。它会创建一个在指定时长过去后完成的 future。这是 `std::thread::sleep` 的异步等价版本。

```rust 等待一段时间 icon=logos:rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 毫秒已经过去");
}
```

如果您需要等待到某个特定的时间点，可以使用 `sleep_until(deadline)`，该函数接收一个 `Instant` 作为参数。

### Sleep Future

`sleep` 和 `sleep_until` 都会返回一个 `Sleep` future。这个 future 可以被操作，最常见的操作是重置其截止时间。这在循环中重用计时器而无需每次都分配新的计时器时非常有用，通常与 `tokio::select!` 结合使用。

在下面的示例中，`sleep` future 被固定 (pinned)，并在其到期后于循环内重置其截止时间。

```rust 复用 Sleep future icon=logos:rust
use tokio::time::{self, Duration, Instant};

#[tokio::main]
async fn main() {
    let sleep = time::sleep(Duration::from_millis(10));
    tokio::pin!(sleep);

    loop {
        tokio::select! {
            () = &mut sleep => {
                println!("计时器已到期");
                // 将截止时间重置为从现在开始的 50 毫秒后
                sleep.as_mut().reset(Instant::now() + Duration::from_millis(50));
            },
        }
    }
}
```

### 重要注意事项

如果没有活动的 Tokio 运行时，或者运行时未配置启用计时器，创建像 `Sleep` 这样基于计时器的 future 将会引发 panic。请确保您的 `Builder` 包含 `.enable_time()` 或 `.enable_all()`，或者您正在使用 `#[tokio::main]` 宏（该宏默认启用计时器）。

```rust
// 这会引发 panic，因为 sleep() 是在运行时上下文之外调用的。
// let rt = tokio::runtime::Builder::new_current_thread().build().unwrap();
// rt.block_on(sleep(Duration::from_secs(1)));

// 这是正确的。异步块在运行时内部执行。
let rt = tokio::runtime::Builder::new_current_thread().enable_all().build().unwrap();
rt.block_on(async {
    sleep(Duration::from_secs(1)).await;
});
```

## 使用 interval 执行周期性任务

对于需要按固定计划重复运行的任务，`tokio::time::interval` 比带 `sleep` 的循环更合适。`Interval` 会自动调整两次 tick 之间因执行其他工作而花费的时间，确保 tick 的平均间隔符合指定的周期。

如果在循环中使用 `sleep`，每次迭代的总时间将是 `sleep_duration + task_duration`。而使用 `interval`，两次 tick 之间的时间则保持一致。

```rust 每两秒运行一次任务 icon=logos:rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("正在执行任务...");
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
在此示例中，尽管任务需要一秒钟时间，但新的 tick 仍会大约每两秒发生一次。

### 处理错过的 Tick

如果两次 `interval.tick().await` 调用之间的代码执行时间超过了 interval 的周期，可能会“错过”一个或多个 tick。您可以使用 `MissedTickBehavior` 来配置 `Interval` 在这种情况下的行为。

| 行为 | 描述 |
| :--- | :--- |
| `Burst` (默认) | interval 将尽快触发 tick，以追赶上它本应达到的时间点。 |
| `Delay` | 下一个 tick 的调度是相对于上一次调用 `tick()` 的时间，这实际上会延迟整个计划。 |
| `Skip` | interval 会跳过错过的 tick，并在原始周期的下一个倍数时间点调度下一个 tick。 |

```rust 设置 MissedTickBehavior icon=logos:rust
use tokio::time::{interval, Duration, MissedTickBehavior};

let mut interval = interval(Duration::from_millis(50));
interval.set_missed_tick_behavior(MissedTickBehavior::Delay);
```

## 使用 timeout 限制执行时间

为防止 future 无限期运行，您可以用 `tokio::time::timeout` 将其包装。如果被包装的 future 未在指定时长内完成，`timeout` future 将以 `Err` 状态完成。

`timeout` 函数返回一个 `Result`。如果 future 在时间限制内成功完成，则返回 `Ok(value)`。否则，返回 `Err(Elapsed)`。

```rust 使用 timeout icon=logos:rust
use tokio::time::{timeout, Duration};
use tokio::sync::oneshot;

async fn some_long_operation() {
    // 模拟工作
    tokio::time::sleep(Duration::from_millis(500)).await;
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_millis(100), some_long_operation()).await {
        Ok(_) => println!("操作成功完成。"),
        Err(_) => println!("操作超时！"),
    };
}
```

如果您需要在超时后取回原始的 future，可以在 `Timeout` future 上使用 `into_inner()` 方法。

---

您现在已经掌握了控制异步操作计时的工具。理解如何管理延迟、周期性任务和超时，对于构建健壮、可靠的应用程序至关重要。要了解有关驱动这些任务的引擎的更多信息，请继续阅读下一部分。

<x-card data-title="下一步：运行时" data-href="/tasks-scheduling/runtime" data-icon="lucide:cpu">
  了解 Tokio 运行时、其调度器以及如何配置它。
</x-card>