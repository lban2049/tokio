# 计时器

Tokio 提供了用于跟踪时间和调度未来工作的实用工具。这些工具对于在异步应用程序中处理超时、延迟和周期性任务至关重要。

总体而言，`tokio::time` 模块提供了三种主要类型的实用工具：

- **`Sleep`**：一个在特定时间点完成的 future。
- **`Interval`**：一个按固定周期产生值的流。
- **`Timeout`**：一个限制 future 最大执行时间的包装器。

所有计时器实用工具都必须在 Tokio [运行时](./concepts-runtime.md) 的上下文中使用，因为它们依赖于其内部的计时器机制。

## Sleep：暂停任务

最基本的计时器实用工具是 `sleep`，它会创建一个在指定持续时间过后完成的 future。它相当于 `std::thread::sleep` 的异步版本。

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

在等待 `sleep` future 期间，不会执行任何工作。它只是暂停当前任务，并允许调度器运行其他任务，直到达到指定的时间。

如果要等待到某个特定时刻，可以使用 `sleep_until(instant)`。这些函数返回的 `Sleep` future 也可以被修改。例如，可以将其 `reset` 到一个新的截止时间，这在 `select!` 循环中非常有用。

```rust,no_run
use tokio::time::{self, Duration, Instant};

#[tokio::main]
async fn main() {
    let sleep = time::sleep(Duration::from_millis(10));
    tokio::pin!(sleep);

    loop {
        tokio::select! {
            () = &mut sleep => {
                println!("timer elapsed");
                sleep.as_mut().reset(Instant::now() + Duration::from_millis(50));
            },
        }
    }
}
```

## Interval：重复操作

`Interval` 用于按固定计划执行任务。它会创建一个按指定周期产生值的流。

在循环中使用 `interval` 和 `sleep` 的关键区别在于它们处理其他工作所花费时间的方式。`Interval` 会考虑自*上一个* tick 以来经过的时间。如果两个 tick 之间的工作耗时超出预期，下一个 `.tick().await` 将会更快（或立即）完成以追赶进度。

下图说明了这种差异：

```d2
direction: down

"使用 sleep(2s) 的循环": {
  shape: sequence_diagram

  "任务": {
    "工作 (1s)": {lifespan: 1}
    "sleep(2s)": {lifespan: 2}
    "工作 (1s) ": {lifespan: 1}
    "sleep(2s) ": {lifespan: 2}
  }
  
  note over "任务": "总周期时间：3s"
}

"使用 interval(2s) 的循环": {
  shape: sequence_diagram

  "任务": {
    "工作 (1s)": {lifespan: 1}
    "tick() await (1s)": {lifespan: 1}
    "工作 (1s) ": {lifespan: 1}
    "tick() await (1s) ": {lifespan: 1}
  }

  note over "任务": "总周期时间：2s（按计划）"
}
```

下面是一个实际的例子：

```rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("Performing a task...");
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

在此示例中，即使任务本身需要一秒钟才能运行，它也大约每两秒执行一次。

### Missed Tick 行为

如果在两次调用 `.tick().await` 之间经过了大量时间，`Interval` 就被认为“错过”了一个或多个 tick。默认行为 `MissedTickBehavior::Burst` 会尽快触发 tick 直到赶上进度。可以配置其他行为，如 `Delay` 和 `Skip`，以便更好地控制这种情况。

## Timeout：设置截止时间

通常，需要限制一个异步操作的允许运行时长。`timeout` 函数会包装一个 future，如果它未在指定时间内完成，则会取消该 future。

该函数返回一个 `Result`。如果 future 在超时前完成，它会返回包含 future 输出的 `Ok`。如果先超时，它会返回 `Err(Elapsed)`。

```rust
use tokio::time::{timeout, Duration};

async fn long_running_operation() {
    // some work that might take too long
    sleep(Duration::from_secs(5)).await;
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_running_operation()).await;

    if res.is_err() {
        println!("Operation timed out!");
    }
}
```

值得注意的是，超时检查发生在轮询被包装的 future *之前*。如果 future 运行一个耗时较长的、受 CPU 限制的计算而没有让出（即没有到达 `.await`），它可能会超过截止时间而不会被立即取消。

---

这些计时器实用工具是创建可靠、时间敏感应用程序的基础构建块。有关其 API 的更多详细信息，请参阅 [`tokio::time` API 参考文档](./api-time.md)。下一节将更详细地探讨 [Tokio 运行时](./concepts-runtime.md)。