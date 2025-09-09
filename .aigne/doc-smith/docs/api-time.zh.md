# 时间

用于跟踪时间和调度工作的实用工具。该模块提供了几种类型，用于在设定的时间段后执行代码。

这些类型需要 Tokio 运行时。如果在运行时上下文之外使用它们，将会引发 panic。

<x-cards data-columns="2">
  <x-card data-title="Sleep" data-icon="lucide:timer-off">
    `std::thread::sleep` 的异步版本。它会将当前任务暂停指定的时长或直到特定的时刻。
  </x-card>
  <x-card data-title="Interval" data-icon="lucide:timer">
    一个以固定周期产生值的流。可用于按计划运行任务。
  </x-card>
  <x-card data-title="Timeout" data-icon="lucide:alarm-clock-off">
    包装一个 future，为其执行时间设置上限。如果 future 未在超时时间内完成，它将被取消。
  </x-card>
  <x-card data-title="Instant" data-icon="lucide:clock">
    一个单调非递减时钟的测量值，用于计时操作。
  </x-card>
</x-cards>

## Sleep

Tokio 提供了两个暂停任务的函数：`sleep` 和 `sleep_until`。

### `sleep(duration: Duration)`

等待直到 `duration` 指定的时间过去。这是 `std::thread::sleep` 的异步模拟。

```rust icon=logos:rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### `sleep_until(deadline: Instant)`

等待直到达到时间上的某个特定 `Instant`。

```rust icon=logos:rust
use tokio::time::{sleep_until, Instant, Duration};

#[tokio::main]
async fn main() {
    sleep_until(Instant::now() + Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

这两个函数都返回一个 `Sleep` future。`Sleep` future 可以被操作，例如，重置其截止时间。

```rust Reseting a Sleep future icon=logos:rust
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

## Interval

`Interval` 是一个以固定周期产生值的流。这对于按常规计划运行任务非常有用。

`interval` 和在循环中使用 `sleep` 的关键区别在于，`Interval` 会计算两次 tick 之间任务花费的时间。如果一个任务花费的时间超过了间隔周期，下一个 tick 将立即触发。

### `interval(period: Duration)`

创建一个新的 `Interval`，它会立即开始计时，并每隔一个 `period` 持续触发。

```rust Execute a task every 2 seconds icon=logos:rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("hello");
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

### Missed Tick Behavior

如果 `Interval` 的消费者速度太慢，可能会错过 tick。`MissedTickBehavior` 枚举允许您配置 interval 在此场景下的行为。

| 行为 | 描述 |
|---|---|
| `Burst`  | (默认) 尽可能快地触发 tick，直到赶上进度。 |
| `Delay`  | 重置 interval，从当前时刻开始调度下一个 tick。 |
| `Skip`   | 跳过所有错过的 tick，并等待下一个计划的 tick。 |

您可以使用 `set_missed_tick_behavior` 来更改行为。

```rust icon=logos:rust
use tokio::time::{interval, Duration, MissedTickBehavior};

let mut interval = interval(Duration::from_millis(50));
interval.set_missed_tick_behavior(MissedTickBehavior::Skip);
```

## Timeout

超时用于限制 future 允许执行的时间。`timeout` 函数包装一个 future 并返回一个 `Result`。

- `Ok(value)`：future 在时间限制内成功完成。
- `Err(Elapsed)`：已达到时间限制，future 已被取消。

### `timeout(duration: Duration, future: F)`

要求 future 在指定的持续时间内完成。

```rust Require an operation to complete within 1 second icon=logos:rust
use tokio::time::{timeout, Duration};

async fn long_future() {
    // some long-running work
    tokio::time::sleep(Duration::from_secs(5)).await;
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_future()).await;

    if res.is_err() {
        println!("operation timed out");
    }
}
```

还有一个 `timeout_at(deadline: Instant, future: F)` 变体，它接受一个特定的截止时间而不是持续时间。

## Instant

Tokio 的 `time::Instant` 是对标准库 `std::time::Instant` 的封装。所有 Tokio 的时间相关 API 都使用它，并且它对于正确使用 Tokio 的测试工具至关重要，这些工具允许进行时间操作。

它提供了相同的核心功能来测量经过的时间。

```rust Measuring elapsed time icon=logos:rust
use tokio::time::{Duration, Instant, sleep};

#[tokio::main]
async fn main() {
    let instant = Instant::now();
    let three_secs = Duration::from_secs(3);
    sleep(three_secs).await;
    assert!(instant.elapsed() >= three_secs);
    println!("Successfully waited for at least 3 seconds.");
}
```